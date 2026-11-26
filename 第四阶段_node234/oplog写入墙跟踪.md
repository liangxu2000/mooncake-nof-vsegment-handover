# 大数据写入的 oplog 墙：跟踪记录

- 开始日期：2026-11-26
- 状态：**写入墙已解除；NOF-124 parity 达成（P50/Avg/P90）**（etcd 64MiB + extent=50GiB；vseg 59,931/60,000；P50 0.966×、W9 0.909×；P99/Max 未平，尾部待清洁数据）
- 一句话：目标 **60,000 keys（240GiB）**，在 **25.6k keys（约 100GiB）** 撞上 etcd 单事务上限；不是总空间，不是批条数，而是 **vsegment full-state 快照作为一条 oplog entry** 太大。
- 关联任务：NOF-119 A / NOF-120 / NOF-121 / NOF-122 / **NOF-123（当前）**

---

## 1. 背景

- 真实拓扑：reader node2；master+etcd node4；target A node3 + target B node4；vsegment `member_count=2`；plain 单 target。
- 目标：60k keys ×4MiB = 240GiB，单遍读约 20s，vseg 与 plain 对比 parity。
- 正常 HA 配置：`--enable_ha=true --enable_oplog=true`；etcd 默认 `--max-request-bytes=2MiB`。
- 旧 node1/node3 的“2000-key 942MB oplog”**不外推**；本记录只写当前环境实测。

---

## 2. 时间线与实验

### 2.1 NOF-119 A：容量可扩，oplog off 不发 quota

| 项 | 结果 |
|---|---|
| 512GiB segment 注册 | **通过**：两 segment 各 549,755,813,888B，总 1TiB；无需重启 target |
| oplog off 发布 vseg quota | **不通过**：2 分钟 `enabled=0`；日志 `no eligible psegment pool yet` |
| 同机 oplog on 对照 | **通过**：40s 发布 generation=1、partitions=1、discovered_segments=2 |
| 手工 quota 回退 | 不可用：planner 工具未构建、HTTP 404 |

结论：oplog off 这条路在当前构建不可用；quota 发布依赖 HA/OpLog 提交路径。

### 2.2 NOF-120：5k 试水，总空间安全

| 项 | 实测 |
|---|---|
| Keys | 5,000 |
| etcd dbSize | 13,619,200 B（12.99MiB） |
| `/oplog/vseg_nof120/` | 9,870 keys / 4,910,154 B（4.68MiB） |
| dbSize 增量 | +8,257,536 B（≈1,651.5 B/key） |
| 外推 60k | **≈0.105GiB** |
| 文件系统 | 可用 1.1TiB（used 59% 为历史占用） |

结论：**总空间不是问题**；oplog on 可以支撑 60k 的空间需求。  
注意：5k 没有触发 1024 条大批判，因此没有暴露后面的“单条 full-state”问题。

### 2.3 NOF-121：第一次撞墙（batch=1024）

| 项 | 实测 |
|---|---|
| 写入结果 | **25,647 succeeded / 34,353 failed** |
| 失败码 | `ret=-1010` ×34,352、`VSEGMENT_CREATING` ×2、`ret=-105` ×1 |
| 错误原文 | `TxnCompareAndPut failed: ResourceExhausted: received message larger than max (13295444 vs. 2097152)` |
| 单事务 | **13,295,444 B > etcd 默认 2,097,152 B**；同尺寸重试 3784+ 次 |
| 数据量 | 25,647×4MiB = **100.184 GiB** |
| vsegment 状态 | `vsegments=2`，但 `logical_capacity=100GiB`、`create_success=1` |
| etcd | dbSize 54.5MiB、oplog 23.89MiB、`/cvm` 0.012MiB；总空间安全 |

### 2.4 NOF-122：把 batch 从 1024 降到 64，仍然无效

| 项 | NOF-121（1024） | NOF-122（64） |
|---|---|---|
| 失败 Keys | 25,647 | **25,632** |
| 失败事务 | 13,295,444 B | **13,322,832 B（+0.2%）** |
| 每 key 折算 | 518 B | **520 B** |
| etcd 上限 | 2,097,152 B | 2,097,152 B（未变） |
| 结果 | `-1010` 无限重试 | 同样 `-1010`，同一尺寸 |

结论：**`oplog_batch_max_entries` 不是这条事务的尺寸杠杆**。

### 2.5 NOF-123：etcd 64MiB 解除写入墙

| 项 | 实测 |
|---|---|
| etcd `--max-request-bytes` | 67,108,864（64MiB） |
| ResourceExhausted | **0**（NOF-122 12,588） |
| full-state 写入 | 成功，`value_max=13,284,282 B` |
| vsegments / create_success | **3 / 3**（1→3） |
| Keys | **59,931 / 60,000** |
| 失败码 | **-1401 `TASK_PENDING_LIMIT_EXCEEDED` ×67**、-105×2、CREATING×4 |
| target A/B 写 n | 958,911 / 958,912 |
| 写吞吐 / P50 | 10,450 MiB/s / 364µs |

结论：**2MiB 单事务墙解除；100GiB 处 vsegment 扩容恢复**。69 个失败是客户端 pending 上限，不是数据/空间问题。

---

## 3. 根因（源码级）

- `VSegmentOpLogRecord`（`mooncake-store/include/vsegment/vsegment_ha.h:24-37`）有一个 `full_state` 标志；
- `full_state=true` 时，payload 是整块 **`PartitionVSegmentSnapshot`**；
- `OrderedOpLogVSegmentCommitter::CommitDeferred`（`vsegment_ha.cpp:121-148`）把它 `struct_json::to_json` 成 **一个 `OpLogEntry`**；
- 该快照包含 `logical_allocation`（每个 key 的分配记录），约 **520B/key**；
- 25.6k keys ⇒ **13.3MB**；60k keys ⇒ **约 31MB**；
- 它是一条 entry，`oplog_batch_max_entries=64` 只能限制“批里几条”，**不能拆开这一条**；
- 于是 `TxnCompareAndPut` 把整条 13.3MB 放进一个 etcd 事务，超过 `--max-request-bytes=2MiB`，被 `ResourceExhausted` 拒收；
- 批一旦封好，master 用同一冻结事务无限重试，不会自愈。

**为什么卡在 ~100GiB**：25,632 keys 刚好越过单 vsegment 100GiB；第 2 个 vsegment 创建触发了 full-state checkpoint，而这条 checkpoint 超过 etcd 上限。  
（[推算]，NOF-123 抬 etcd 上限后应能验证；若第 2 个 vsegment 仍不创建，则 100GiB 静态 quota 是第二道墙。）

---

## 4. 已排除 vs 未排除

### 4.1 已排除（当前环境实测）

| 假说 | 结论 |
|---|---|
| oplog 总空间不够 | **排除**：etcd 92MiB、/home 1.1TiB、/cvm 0.012MiB |
| oplog 批条数太大 | **排除**：1024→64，事务尺寸不变 |
| vseg quota 空间不足 | **排除**：512GiB segment 注册成功、总 1TiB |
| oplog off 可绕开 | **排除**：quota 不发布，手工 quota 不可用 |
| 数据/磁盘满 | **排除**：target 写入 n 与容量均正常 |

### 4.2 未排除

| 假说 | 状态 |
|---|---|
| **etcd `max-request-bytes=2MiB` 是唯一卡点** | NOF-123 验证中（64MiB） |
| 100GiB 单 vsegment 静态 quota 是第二道墙 | 未分离；NOF-123 观察 vsegments/capacity |
| full-state checkpoint 的触发频率/大小是否还有更大值 | 60k 预计 31MB；需实测 |
| plain 是否不受此墙 | 预期不受（不写 vsegment state），NOF-123 最终 plain 臂验证 |

---

## 5. 解决决策

### 5.1 选择 E1：抬 etcd 单请求上限

- 重启 etcd，增加 **`--max-request-bytes=67108864`（64MiB）**；
- 数据复用，不改 data-dir；
- 60k full-state 约 31MB，64MiB 留 2× 余量；
- 只重启 etcd + master，**不动 target**；
- 为什么不是 E2/E3：
  - E2（缩小 extent）不解决：full-state 是 **per partition**，不是 per vsegment，缩小 extent 不会缩小 state；
  - E3（plain 60k）不解决 vseg，只作为最终对照臂。

### 5.2 回退

- etcd 启动失败/数据异常：恢复原参数重启，停手回报；
- 64MiB 仍被拒：停手，附 etcd flag/错误原文；下一步才考虑改码（拆分/省略 full-state）或退回 20k。

### 5.3 下一单候选（未下发，等 NOF-123 回执）

**一整块 vsegment**：`member_extent_size=128GiB`（逻辑 256GiB），60k/240GiB 全部装进 1 个 vsegment。

- 依据（L2 假说）：当前 50GiB×2=100GiB/vsegment，25,600 keys 填满，第 25,601 个 key 触发第 2 个 vsegment 创建；full-state checkpoint 恰在此时出现（25,632 keys/13.3MB）。
- 一整块 = 不再创建第 2 个 vsegment ⇒ 写入过程中不触发这次 full-state；初始 checkpoint（0 keys）很小，后续都是 delta。
- **只解决写入过程**：master 重启/恢复仍会做一次 full-state（60k≈31MB），etcd 默认 2MiB 仍不够 ⇒ 恢复/HA 仍需 etcd 64MiB 或改码。
- 变量隔离：如果先在一整块上测试，可保持 etcd 默认 2MiB，避免“extent + etcd”两个变量混在一起；NOF-123 先验证 etcd 64MiB + 50G。

---

## 6. NOF-123 执行项（当前）

1. 停 wedged `vseg_nof122` master（2404042/2404045；CPU 86%、无限重试），备份日志；target 不动。
2. 优雅停 etcd 2389250，用同 data-dir + `--max-request-bytes=67108864` 重启；验证 health/leader/dbSize。
3. 起 `vseg_nof123`（HA+oplog、batch=64、512GiB segments、member_count=2/extent=50G/stripe=2M），注册 B→A。
4. 写 60k；监控 ResourceExhausted；验收 Keys=60000、logical_used=251,658,240,000；观察 vsegments/capacity 是否增长。
5. 通过后 vseg 读 1 遍（60k queries，T1/warmup128/qpairs4/diag off），约 20–24s。
6. plain `plain_nof123` 写/读 1 遍，约 20s；parity。

**成功判据**：etcd 无 `ResourceExhausted`；Keys=60000；vsegments 按需增长；vseg 读通；plain 读通；parity 可计算。

---

## 7. 最终结果（NOF-123 已更新，读臂/plain 待 NOF-124）

### 7.1 NOF-123 E1（写入墙解除）

| 项 | 实测 |
|---|---|
| etcd `--max-request-bytes` | **67,108,864（64MiB）**；data-dir 复用，dbSize 96,854,016B 不变 |
| ResourceExhausted | **0**（NOF-122 12,588） |
| full-state 快照 | **写入成功**，etcd `value_max=13,284,282 B`（13.3MB） |
| vsegments / create_success | **3 / 3**（1→3，扩容恢复） |
| Keys | **59,931 / 60,000**（99.885%） |
| 失败码 | **-1401 `TASK_PENDING_LIMIT_EXCEEDED` ×67**、-105×2、CREATING×4 |
| target A/B 写 n | 958,911 / 958,912（均分，≈59,931×16） |
| 写吞吐 / P50 | 10,450 MiB/s / 364µs（含扩容停顿） |
| etcd dbSize | 246,902,784B（235MiB）；/home 余 1.1TiB |

- `-1401` 是客户端 pending 任务上限（`types.h:417`），瞬态/背压，不是数据错误。
- `logical_capacity` gauge 仍 100GiB 而 vsegments=3：展示/quota 统计问题；Keys、target 落盘、create_success 均正常，不阻塞。
- **结论：2MiB 单事务墙已解除；100GiB 处 vsegment 扩容恢复；写入过程解决。**

### 7.2 NOF-124（读臂 + plain parity）

| 指标 | vseg（59,931 keys） | plain（60,000 keys） | 比值 |
|---|---|---|---|
| **P50** | **246.32µs** | **255.06µs** | **0.966×（vseg 快 3.4%）** |
| Avg | 252.35 | 256.74 | 0.983× |
| P90 | 255.10 | 263.26 | 0.969× |
| P99 | 361.82 | 274.62 | 1.317× |
| Max | 57,788.40 | 2,443.73 | 23.6× |
| target W9avg | 40,410 / 40,330ns | 44,450ns | 0.909 / 0.907 |
| 读耗时 | 19s | 20s | — |
| failed | 69（0.115%） | 0 | — |

- **P50/Avg/P90/W9 达到 parity；P99/Max 未达到。**
- vseg 69 个缺失 key（type unknown、1× `-704 OBJECT_NOT_FOUND`）可能污染尾部（Max 57.8ms 很可能是一次失败查找）；**在补齐或剔除失败样本前，不能把尾部差距归因 vseg 路径**。
- `logical_capacity` 刷新后 300GiB，`logical_used=251,368,833,024=59,931×4MiB` 精确相等 ⇒ NOF-123 的 100GiB 是 gauge 刷新延迟。
- etcd 64MiB 两阶段 ResourceExhausted 0 ⇒ **写入墙解决**。
- 当前配置 `extent=50GiB`、3 个 vsegment；未验证“一整块 128GiB”。

---

## 8. 环境与口径（2026-11-26）

| 对象 | 状态 |
|---|---|
| target A（node3） | PID 3329243；NQN `...urma-gpu-test-node3`；RAID0 UUID `2c5bf1f2…`；11/11、strip256K、nsid1、4457 |
| target B（node4） | PID 2391418（重建后）；RAID0 UUID `c19b91e1…`；11/11、strip256K、nsid1、4457 |
| etcd | PID 2389250；data-dir 复用；默认 max-request-bytes 2MiB |
| wedged master | `vseg_nof122` 2404042/2404045；Keys=25,632；无限重试 13.3MB 事务 |
| 数据 | vseg 25,632 keys ≈100.12GiB；target A/B 各 410,127 IO |
| key/并发 | 4MiB、T1（threads=1/batch=1）、warmup=128、qpairs=4、verify=false、diag 关 |
| vseg | member_count=2、stripe=2MiB、extent=50GiB、required_medium=nof |
| 文档原则 | 只用 node 标签，不写内网 IP |

---

## 9. 相关源码

| 内容 | 位置 |
|---|---|
| `VSegmentOpLogRecord` / `full_state` | `mooncake-store/include/vsegment/vsegment_ha.h:24-37` |
| `CommitDeferred` 整块序列化为一条 entry | `mooncake-store/src/vsegment/vsegment_ha.cpp:121-148` |
| OrderedOpLog writer 批封口 | `mooncake-store/src/ha/oplog/ordered_oplog_writer.cpp` |
| 批事务 `TxnCompareAndPut`（单 value） | `mooncake-store/src/ha/oplog/oplog_batch_storage.cpp:134-190` |
| etcd 错误 `ResourceExhausted` | `mooncake-store/src/etcd_helper.cpp:263` |
| `--oplog_batch_max_entries` 定义 | `mooncake-store/src/master.cpp:385` |

---

## 10. 变更记录

- 2026-11-26：建立本文档；记录 NOF-119 A / 120 / 121 / 122；确定根因为 full-state 单条 oplog entry；下发 NOF-123（etcd 64MiB）。
- 2026-11-26：NOF-123 E1 成功，写入墙解除；NOF-124 P50/Avg/P90 parity 达成，P99/Max 待清洁数据。
- **遗留**：vseg 69 个缺失 key（0.115%）未回填；P99/Max 未平，暂不归因 vseg 路径；一整块 128GiB 未验证，仅作未来候选。