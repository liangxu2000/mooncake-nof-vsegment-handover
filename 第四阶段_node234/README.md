# 第四阶段：node2/3/4 双 target vsegment 性能收口

- 开始日期：2026-11-26
- 目录：`D:\mooncake nof vsegment\第四阶段_node234`
- 任务通道：`../HANDOVER.md`
- 历史依据：`../第三阶段 node2&4/`（单 target 定位、双 target 打通）、`../接手以来重要结果.md`
- 拓扑与产物只使用 **node2/node3/node4** 指代，不写内网 IP。

---

## 1. 为什么进入第四阶段

第三阶段完成了两件事：

1. **单 target 下的 2× 被压缩到 target 单 IO 服务段**：vseg P50 555µs vs plain 274µs（2.03×），target W9 248 vs 120（2.07×）；客户端计划/提交、child 并行、worker、inflight、qpair、数据 LBA 全部排除；2× 是**按字节**的服务成本，不是每 IO 固定开销。
2. **双 target vsegment 打通**（NOF-115）：node3 A + node4 B 两个 segment 注册成功，`member_count=2` quota 发布成功，每个 key 2 个 child、两 target IO 均分。

第四阶段不再回答“单 target 为什么 2×”，而是回答**真实部署形态下 vseg 是否达到 parity**：

```text
plain：400 keys，只打 node4 一个 target
vseg ：400 keys，member_count=2，跨 node3+node4 两个 target
验收：vseg P50 ≤ plain P50 × 1.05
```

---

## 2. 当前拓扑（用户 2026-11-26 确认）

```text
node2（reader + writer）
┌────────────────────────┐
│ stress_cluster_bench   │──元数据 RPC──► node4（master + etcd）
│ （supercache 构建）    │
│                        │──数据 URMA──► node3（target A）
│                        │──数据 URMA──► node4（target B）
└────────────────────────┘

node3：target A（11 盘 RAID0，strip 256KiB，nsid1，4457）
node4：master + etcd + target B（11 盘 RAID0，strip 256KiB，nsid1，4457）
```

- reader/writer：**node2**
- master + etcd：**node4**
- 两个远端 target：**node3 A + node4 B**，共同构成一个 vsegment
- vsegment：`member_count=2`，4MiB key → 2×2MiB child，分别落 node3/node4
- plain：只用 **node4 一个 target**
- 对 reader 来说，元数据（node4）和数据（node3/node4）**都跨节点**

### 端口

| 组件 | 节点 | 端口 |
|---|---|---|
| SPDK nvmf target A | node3 | **4457** |
| SPDK nvmf target B | node4 | **4457** |
| Mooncake master RPC | node4 | **50056** |
| metrics | node4 | **9006** |
| HTTP metadata | node4 | **8086** |
| etcd（本地） | node4 | **2379** |
| client TE RPC（如需） | node2 | **50156** |

---

## 3. 当前环境状态（2026-11-26 NOF-129 完成后）

| 对象 | 状态 |
|---|---|
| target A（node3） | **正常**：PID 3329243；NQN `...urma-gpu-test-node3`、SN `URMAGPU0003`；RAID0 UUID `2c5bf1f2…`；11/11 online、strip 256KiB、nsid1、4457、0 ERROR |
| target B（node4） | **已重建**：PID 2458315；NQN `...urma-gpu-test`、SN `URMAGPU0001`；**新 UUID 50d6259e-51a6-4599-adf3-68defc0c5a53**；RAID0 strip 256K、online 11/11、4457、ERROR=0 |
| master | **`plain_nof135p` 2509048/2509051 运行中**：Keys=60000（当前 plain 基线，先保留） |
| etcd | 2455814；`--max-request-bytes=67108864`；dbSize 1931.57MiB、raftIndex 2,261,561 |
| node2 | 无 bench/register 残留；客户端 md5 bb4a07a3…（未重编） |
| 当前任务 | **NOF-159（512K/128K 复测 · 60k）**：E0/E1 各 3 个正常遍（SLOW 不计、补跑）+ 4M 前后锚点；查清 512K E1 是否仍 49µs、128K E1 是否仍快 42–49µs；结果另写 `NOF159_512K_128K_复测_2026-11-28.md`；NOF-160 放大 chunk 未下发 |
| 风险 | node4 非独占：外部用户 `computin` 16:20–16:50 多次登录；本次重建后未被清；若再次清场/资源争用，停手回报 |

---

## 4. 已确认、必须继承的口径

### 4.1 双 target vseg

1. 两个 segment 注册成功且不覆盖：node4 B 与 node3 A 的 traddr/NQN 均不同。
2. 每个 key `child_count=2`、`blocks=4096`、`sub_ios=16`；两个 target 各收到约一半 IO（NOF-115 为 66 vs 66）。
3. `member_count=2` 是**条带，不是副本**：每个 target 存 key 的一半（2MiB），不是各存一份 4MiB。

### 4.2 工程口径（容易踩坑）

1. **segment 注册不依赖常驻进程**：master 通过 `nof_heartbeat_*` 主动探测 target 维持；`nof_register` 退出 40+ 分钟后 segment 数仍为 2、心跳计数继续增长。**只有 master 重启才需要按 node4 B → node3 A 重注册**；此前“注册进程必须整臂存活”的说法作废。
2. **`segment_write` 与 `segment_read` 的 `--num_keys=N` 都是每 segment N**：vseg 两个 segment 要写/读 400 keys，用 `--num_keys=200`（总 400 queries）；用 400 会得到 800 queries/400 rpc_fail。plain 单 segment 用 400。
3. **`vsegments=1` 是 quota 发布时预建**，正常；写数据不会新增。
4. **`logical_used` gauge 1000s 才刷新**；验收以 master `Keys` + writer 日志为准。
5. child→target 的 endpoint 不在 `nof_slow_task` 里；落点均分用**两个 target 的 IO 计数相等**判断。
6. `extent` 是**每个 member 在 NoF segment 上的物理 extent**，不是节点 DRAM；NOF-124 基线为 50GiB/member → 每 vsegment 逻辑 100GiB；NOF-125 改为 **256GiB/member → 逻辑 512GiB**（每 target 注册 512GiB，无需扩大）。

### 4.3 单 target 下的结论（第三阶段，仍有效）

- vseg P50 555µs vs plain 274µs（2.03×）；target W9 248 vs 120（2.07×）。
- 差值 95% 在 `completion_wait`；客户端 resolve/prepare/submit 合计 <10µs。
- 两个 child 100% 并行，op span = 1.10× 单 child；不是串行。
- worker、inflight、qpair、数据 LBA 均排除。
- **2× 是按字节的服务成本，不是每 IO 固定开销**；chunk 加大不降时延，W9 反而变差；target 单 IO 在 ≥256KiB 时被夹到 256KiB。
- 剩余解释：target 对 vseg 请求的每字节服务路径（尚未拆 W9）。

### 4.4 正式测试口径（2026-11-26 用户定，后续默认）

1. key = 4MiB，**60,000 keys**；400 keys 不再使用。
2. vseg = **一整块**：`extent=256GiB/member`（274877906944），逻辑 512GiB；`member_count=2 / stripe=2MiB`；50G/3vseg 仅历史对照。
3. plain = 单 target B。
4. 读 = T1（threads=1/batch=1）、warmup=128、qpairs=4、verify=false、diag 关、sequential；vsegment `--num_keys=30000`/segment；**每轮只读一遍 60k keys（约 20s），禁止 `--duration` 连续循环读**。
5. 稳定性：每臂 ≥6 轮读，至少 2 个交替周期（V3→P3→V3→P3）；**正式结果按 `测试结果聚合方法_20261112.md` 聚合**（Avg=各轮均值、P50/P90/P99=各轮同分位中位）；逐轮原始值落盘，正文只给聚合值 + spread；spread>3% 标注原因、>10% 不进表；不报吞吐。
6. 第四阶段第一个正式结果 = NOF-127。

---

## 5. 三层参数速查

| 层 | 参数 | 当前值 | 作用 |
|---|---|---|---|
| vsegment 逻辑条带 | `stripe_size` | **2MiB** | 4MiB key → 2 个 2MiB child，落两个 member |
| NoF 提交切片 | `MC_NOF_SUBMIT_CHUNK_BYTES` | **128KiB** | 每个 2MiB child → 16 个 128KiB 子 IO |
| target RAID0 条带 | `strip_size` | **256KiB** | 每个 128KiB 子 IO 按起始地址落到某块盘 |

详细公式、LBA 例子和常见误解见：  
`../第三阶段 node2&4/条带与分块三层参数说明.md`

---

## 6. NOF-116 结果与当前 NOF-117

### NOF-116（已完成，未达 parity）

| 臂 | 每遍 P50 | W9（每 target） |
|---|---|---|
| plain（node4 单 target） | 267.74 / 264.86 / 266.49 | **120.33µs** |
| vseg（node3+node4） | 300.09 / 461.35 / 284.93 / 290.85 / 324.15 / 280.17 | **157.3µs** |

- plain 三遍 264.86–267.74；vseg 除离群 461.35 外 280.17–324.15 ⇒ 未达 parity（≤1.05×）。
- 双 target 把单 target 的 **2.02×** 压到约 **1.05–1.22×**，但**未达 parity**。
- 两个 target 写 n=6400/6402、读 n≈8450，均分；`type` 全 vsegment。

### NOF-117（已完成，未达 parity）

| 项 | vseg@128K | vseg@64K |
|---|---|---|
| P50（逐遍） | 303.75 / 291.37 / 286.59 | 272.27 / 294.54 / 293.91 |
| 逐遍比 64K/128K | — | 0.896 / 1.011 / 1.026 |
| target W9 | 157.58 | **130.50（−17.2%）** |
| 每 target n | ~8,449 | ~16,897 |

- P50 逐遍互有高低、没有整体下降；target W9 明显改善 ⇒ 队列深度/每 target IO 数不是 P50 剩余差距的主因，停止调 chunk。
- 剩余 ~25µs 不在 target 单 IO 执行段；更可能在客户端两 child 聚合/等待、两跳网络差异、或 64K 带来的客户端开销翻倍。
- 当前待裁决：A 接受近 parity；B 零写查剩余 ~25µs；C 改码或第三 target（均不轻）。

### 当前 NOF-118（零写，查剩余 ~25µs）

- 现有 `vseg_nof116` 400 keys 只读；`--num_keys=200`（每 segment，共 400 queries）。
- 6 遍交替：64K diag / 128K diag / 64K / 128K / 64K / 128K；`MC_STORE_TRANSFER_DIAG=1`。
- 解析：
  - `nof_vsegment_child` / `nof_slow_task`：两 child 的 submit/complete、latency、completion_gap、last_child、endpoint（node3/node4）；
  - `get_into_breakdown`：query/read/resolve/prepare/submit/completion_wait/unattributed；
  - 计算 `read_us - key_span`、按 endpoint 分组的 child latency。
- 判决：A 慢 child/两跳不对称；B 客户端聚合/观察；C 64K 客户端开销。
- 零写、零重启、零改码；diag 绝对 P50 不作性能。
- **状态：暂缓**（用户要求先纠正 key 规模，NOF-119 优先）。

### 当前 NOF-119（单遍 ~20s，60k keys/240GiB）

- 用户要求：400 keys 太少，要很多 keys，最好一遍连续读 20s。
- 按当前 Avg：20s ≈ 50–60k keys；取 **60,000 keys = 240GiB**。
- **先过两道门**：
  1. NoF segment 容量从 100GiB 扩到 **512GiB**（若注册/目标支持，不重启 target）；
  2. **本单选择** `enable_ha=true + enable_oplog=false`，先验证能发布 quota。  
     关 oplog 是配置选择（不把 HA 日志变成大测试变量），不是断言当前环境 60k 会爆；旧 node1/3 的 942MB 不外推。
- A 段只做容量/oplog 探针 + 2-key 高 LBA 验证；不写 60k。失败即停。
- 通过后：plain 60k 单遍（node4，240G），vseg 60k 单遍（node3+node4，每 target 120G，`--num_keys=30000`/segment）；T1、warmup=128、qpairs=4、diag 关；各 1 遍约 20s。
- 验收 parity（vseg P50 ≤ plain×1.05）；不达则记录，不调 key 数重跑。

### NOF-119 A 段结果（2026-11-26）

- **512GiB segment 注册通过**：两 segment 各 549,755,813,888B，总 1TiB；无需重启 target。
- **oplog off 时 vseg quota 不发布**：2 分钟 enabled=0；同机 oplog on 对照 40s 发布成功 ⇒ quota 发布依赖 HA/OpLog 提交路径。
- 手工 quota 回退不可用：planner 工具未构建、HTTP 404。
- 按判据停手，未写 60k。
- target B 已重建（PID 2391418、新 UUID `c19b91e1…`、11/11、0 ERROR）；旧 vseg/plain 数据失效，需新 cluster。

### 当前 NOF-120（oplog-on 5k 试水）

- 新 vseg master `vseg_nof120`：正常 HA + **oplog on**、512GiB segments、member_count=2/extent=50G/stripe=2M。
- 写 **5,000 keys**（20GiB），实测当前环境 etcd dbSize / data-dir / `/oplog/vseg_nof120/` 增长。
- 判决：外推 60k oplog + 当前 dbSize <4GiB 且 etcd 盘剩余 >50% ⇒ “oplog on 可支撑 60k”；否则停手，建议编译 quota planner 走 oplog off 或缩小规模。
- 只写 5k；不写 60k；不重启 target。

### NOF-120 结果（2026-11-26，通过）

| 项 | 实测 |
|---|---|
| Keys | 5,000 |
| etcd dbSize | 13,619,200 B（12.988 MiB） |
| dbSize 增量 | +8,257,536 B（≈1,651.5 B/key） |
| `/oplog/vseg_nof120/` | 9,870 keys / 4.68 MiB |
| 外推 60k 增量 | **≈0.105 GiB** |
| 文件系统 | 2.7T / used 1.5T / **avail 1.1T**（59%） |
| vsegments | 1 / 100GiB；create_success=1、failure=0 |

- 判据②“剩余>50%”字面不满足（41%），但 59% 是历史占用，与本测试无关；绝对 1.1TiB，60k 仅 ~0.1GiB ⇒ **按绝对空间判通过**。
- oplog on 可支撑 60k；不切 oplog off。

### 当前 NOF-121（60k 单遍）

- 复用 `vseg_nof120`（Keys=5000）补写到 **60,000 keys**；`--num_keys=30000`/segment；`VSEGMENT_CREATING` 最多 5 轮；验收 Keys=60000、logical_used=251,658,240,000。
- vseg 读 **1 遍**（60k queries，T1/warmup128/qpairs4/diag off），约 20–24s；target A/B n≈962,048。
- 再停 vseg、起 plain `plain_nof121`（512GiB 注册 B），写 60k、读 1 遍，约 20s；target B n≈1,924,096。
- parity：vseg P50 ≤ plain×1.05；记录实际读时长、W9、etcd 增长。
- 不重启 target、不改码；master 重启需重注册 512GiB。

### NOF-121 A 失败（2026-11-26）

- vseg 补写 60k 在 **25,647 keys** 处失败：34,353 failed，`ret=-1010`。
- 根因：**etcd 单事务 13,295,444B > 默认 max-request-bytes 2,097,152B**，`TxnCompareAndPut` ResourceExhausted；master 对同一超大批判无限重试 3784+ 次。
- **不是总空间**：etcd dbSize 54.5MiB、oplog 23.89MiB、/home 余 1.1TiB。
- 25,647×4MiB=100.184GiB，刚好越过单 vsegment 100GiB quota；vsegments=2 但 capacity 仍 100GiB、create_success=1，扩容路径可疑但未分离。
- 决策：`--oplog_batch_max_entries=64`（不重启 etcd/不改码）；停旧 master 无限重试，新 cluster 重试 60k；暂不退 20k。

### 当前 NOF-122（修批大小后重试 60k）

- 停 `vseg_nof120` master（授权，仅 master，不动 target）。
- 起 `vseg_nof122`：normal HA+oplog on、**batch=64**、512GiB segments、member_count=2/extent=50G/stripe=2M。
- 写 60k（`--num_keys=30000`/segment）；监控 etcd 是否再 ResourceExhausted；验收 Keys=60000、logical_used=251,658,240,000、vsegments/capacity 是否增长。
- 若仍在 ~25,647 失败且非批大小，停手；下一步再裁决 20k/扩容。
- 通过后 vseg 读 1 遍；再 plain `plain_nof122`（batch=64）写/读 1 遍；parity。

### NOF-122 失败（2026-11-26）

- batch 1024→64，失败点仍 **25,632 keys**，事务仍 **13,322,832B**（差 0.2%）⇒ 批条数不是杠杆。
- 源码确认：`VSegmentOpLogRecord.full_state=true` 的 payload 是 `PartitionVSegmentSnapshot`，含每 key allocation（约 520B/key），是**一条 OpLogEntry**；batch 拆不开。
- 25.6k keys ≈100.12GiB，刚好越过单 vsegment 100GiB；第 2 个 vsegment 的 full-state 事务卡住（[推算]）。
- 不是总空间：etcd 92MiB、/home 1.1TiB、/cvm 0.012MiB。

### 当前 NOF-123（etcd 64MiB + 重试 60k）

- **E1**：重启 etcd，加 `--max-request-bytes=67108864`（64MiB）；数据复用；不改 data-dir。
- 停 wedged `vseg_nof122` master（2404042/2404045，CPU 86%、无限重试）；只停 master，不动 target。
- 新 `vseg_nof123`：HA+oplog on、batch=64、512GiB segments、member_count=2/extent=50G/stripe=2M；注册 B→A。
- 写 60k；观察 full-state（25k≈13.3MB、60k≈31MB）能否落盘、vsegments/capacity 是否增长；若 64MiB 仍失败，停手。
- 通过后 vseg 读 1 遍；plain `plain_nof123` 写/读 1 遍；parity。
- E2（缩小 extent）暂不做（full-state 是 per partition，不是 per vsegment）；E3 并入最终 plain 臂。

### NOF-123 成功（2026-11-26，写入墙解除）

| 项 | 实测 |
|---|---|
| etcd `--max-request-bytes` | 67,108,864（64MiB） |
| ResourceExhausted | **0**（NOF-122 12,588） |
| full-state 写入 | 成功，`value_max=13,284,282 B` |
| vsegments / create_success | **3 / 3** |
| Keys | **59,931 / 60,000** |
| 失败码 | **-1401 TASK_PENDING_LIMIT_EXCEEDED ×67**、-105×2、CREATING×4 |
| target A/B 写 n | 958,911 / 958,912 |
| 写吞吐 / P50 | 10,450 MiB/s / 364µs |

- `-1401` 是客户端 pending 任务上限（`types.h:417`），瞬态，不是数据错误。
- `logical_capacity` gauge 仍 100GiB 而 vsegments=3：展示/quota 统计问题；Keys/落盘正常，不阻塞。
- 决策：69/60,000=0.115%，接受 59,931 直接读，不重跑全量覆盖写。

### 当前 NOF-124（读臂 + plain）

- 复用 `vseg_nof123`，vseg 读 1 遍（60k queries，预期 failed≈69，接受）；记录 P50/Avg/P90/P99、target W9/n、实际时长。
- 停 vseg，起 `plain_nof124`（etcd 64MiB、batch=64、注册 B 512GiB），写 60k、读 1 遍；记录同指标。
- parity：vseg P50 ≤ plain×1.05；不补写、不重跑、不改码、不重启 target。

### NOF-124 结果（2026-11-26，parity 达成）

| 指标 | vseg（59,931） | plain（60,000） | 比值 |
|---|---|---|---|
| **P50** | **246.32µs** | **255.06µs** | **0.966×（vseg 快 3.4%）** |
| Avg | 252.35 | 256.74 | 0.983× |
| P90 | 255.10 | 263.26 | 0.969× |
| P99 | 361.82 | 274.62 | 1.317× |
| Max | 57,788.40 | 2,443.73 | 23.6× |
| target W9avg | 40.4 / 40.3µs | 44.45µs | 0.909 / 0.907 |
| 读耗时 | 19s | 20s | — |
| failed | 69（0.115%） | 0 | — |

- **P50/Avg/P90/W9 达到 parity；P99/Max 未达到。**
- vseg 69 个缺失 key（type unknown、1× -704 OBJECT_NOT_FOUND）可能污染尾部；未补齐/未剔除前不能把尾部差距归因 vseg 路径。
- `logical_capacity` 刷新后 300GiB、`logical_used=251,368,833,024=59,931×4MiB` ⇒ NOF-123 的 100GiB 是 gauge 延迟。
- etcd 64MiB 两阶段 ResourceExhausted 0 ⇒ 写入墙解决。
- 决策：本阶段收口；不追 P99、不推进一整块 128GiB；未来若要求尾部/部署优化另立单。

### NOF-125 结果（2026-11-26，写侧全胜、读侧未达 parity）

| 项 | 实测 |
|---|---|
| extent | 274,877,906,944（256GiB/member）；logical_capacity=549,755,813,888（512GiB） |
| 写入 | **60,000 / 0 failed**；vsegments 全程 **1**；`VSegment created` 条数=1；value_max 9,087B（无大 full-state）；ResourceExhausted=0；CREATING=0；145s 墙钟；写 P50/P99 326.09/417.56 |
| 写入 target | A/B n=960,017/960,016；W9avg 22,890/22,610 ns |
| 读取 | 60,000 / 0 failed；20s 墙钟；Min 250.54 / Avg 278.06 / **P50 272.37** / P90 285.03 / P99 386.92 / Max 60,862.44 |
| 读 target | A/B n=822,083/841,329；W9avg 38,890/39,150 ns |
| vs plain（NOF-124） | P50 **1.068×（>1.05，未通过）**；Avg 1.083× / P90 1.083× / P99 1.409× / Max 24.9×；W9 **0.881/0.875** |
| vs vseg（NOF-124，50G） | P50 **+10.6%**；Avg +10.2% / P90 +11.7%；target 读 n 535,330/515,570 → 841,329/822,083（+57%/+59%）；W9 −3.1%/−3.6% |

- 写侧：一整块目标完全达成，NOF-122 的 H2（full-state 来自跨 vsegment 扩容）得到验证；写 P50 326.09 vs NOF-123 364.43（−10.5%，后者含扩容停顿，非干净对照）。
- 读侧：1.068× 未达 parity；但**单遍 + 跨会话**，10.6% 差异落在历史会话漂移（~9%）量级内，**不能归因 extent**。
- target W9 反而低 3%、端到端却高 10.6% ⇒ 劣化不在 target 单 IO。
- “读 IO +58%”不可采信为事实：理论 60k×16=960,000/target；两单实测分别为 841k/822k（86%/87%）与 535k/515k（54%/56%），计数本身不完整、窗口口径不同。
- 决策：50G/3vseg 暂为 parity 配置；NOF-126 同会话三臂复核后再定。**（已复核：见 NOF-126；extent 无显著影响，50G 定为读默认）**

### NOF-126 结果（2026-11-26，同会话三臂；extent 无显著影响）

**每臂区间（µs；逐轮原始值在 `run/nof126_summary.txt`）**

| 臂 | 轮数 | Avg 范围 | P50 范围 | P90 范围 | P99 范围 | failed |
|---|---|---|---|---|---|---|
| plain（C） | 3 | 241.90–265.05 | 242.86–262.50 | 251.13–271.74 | 264.46–296.04 | 0 |
| 50G 历史（B） | 3 | 259.87–277.69 | 254.51–272.75 | 262.90–285.33 | 369.29–400.94 | 33 |
| vsegment（A） | 3 | 268.98–275.73 | 260.74–266.47 | 274.06–280.61 | 369.98–386.94 | 0 |

- **P50 对 plain 的逐轮比**：50G 历史 **1.023–1.048（3/3 ≤1.05）**；vsegment **1.012–1.074（2/3 >1.05）**。
- **extent 影响**：vsegment / 50G 历史的逐轮 P50 比在 **−2.6%～+3.8%**，小于各自臂内跨度 ⇒ **无显著影响**。
- NOF-125 的 272.37 高于 vsegment 臂 P50 上限 266.47 ⇒ 会话偏高；+10.6% 主要是漂移。
- “+58% IO”查实为 target dump 的**块内计数伪影**（非累计）：plain 第 3 遍同一负载只有 1,034,594（前两遍 1,923,395/1,924,099）；同会话 vsegment 的 n 反而低于 50G 历史。
- **注**：本单在用户定“一整块 vsegment”正式条件之前；当时 50G 逐轮全过、整块两遍超；**后续正式条件 = 一整块 vsegment，50G 仅历史对照（见 NOF-127）**。
- 偏差：B 臂第 2 轮补写被人工中止（bench 从 key 0 重写，Keys 恒 59,967）；最终 59,967/60,000，33 失败（0.055%）未补写。A 臂 target B 的 n 未取到（脚本缺口）。

### NOF-127 结果（2026-11-26，第四阶段第一正式结果）

**聚合结果（µs；按 `测试结果聚合方法_20261112.md`：Avg=各轮均值、分位=各轮同分位中位）**

| 臂 | 轮数 | Avg | P50 | P90 | P99 | Avg spread |
|---|---|---|---|---|---|---|
| vsegment | 6 | **264.39** | **257.01** | **268.55** | **376.69** | 6.2% |
| plain | 6 | **254.14** | **248.51** | **256.80** | **274.85** | 6.4% |
| vsegment/plain | — | **1.040×** | **1.034×** | **1.046×** | **1.371×** | — |

- **P50/Avg/P90 均 ≤1.05 ⇒ 聚合口径 parity 达成；P99 1.371× 未达。**
- **spread 6.2% / 6.4% >3%**：按方法“先查原因”；两臂在 p4/p5 窗口同步偏高 ⇒ 共同时间漂移；6% <10%，仍进表。
- 时间就近配对（辅助）：6 对 1.076 / 1.014 / 1.029 / 0.977 / 1.020 / 1.059（5/6 vsegment 慢，4/6 ≤1.05）。
- target W9：vsegment 39,190/39,220ns；plain 44,170–44,220ns（vsegment 每 IO 低约 11–12%）。
- 结论：**P50/Avg/P90 达到 parity；P99 仍落后；无 2×。** 不追 P99/Max；不再用单遍/单轮代表阶段结果。
- 四阶段写入均 60,000/0；etcd +≈630MiB、ResourceExhausted 0；逐轮原始值在 `run/nof127_summary.txt`。

### NOF-129/130 结果（2026-11-26，同环境 vsegment/plain 10 轮诊断；不进正式表）

**条件**：node4 重建后（etcd 64MiB、target B 新 UUID）；vsegment 一整块、plain 单 target B；各写 60k、各读 10 轮；每轮一遍 60k≈20s；T1/warmup128/qpairs4/verify=false/diag 关；每轮取 target W9。

| 指标 | vsegment（N-129） | plain（N-130） | vseg/plain |
|---|---|---|---|
| Avg | 266.93（12.7%） | 255.71（8.1%） | **1.044×** |
| P50 | **260.59**（11.1%） | **252.08**（8.2%） | **1.034×** |
| P90 | 269.12（10.2%） | 261.21（6.2%） | **1.030×** |
| P99 | 383.08（30.8%） | 274.91（7.5%） | **1.393×** |

- target W9：vsegment 39,622ns（1.9%）vs plain 47,629ns（3.7%）⇒ vsegment 每 IO 快约 20%，端到端慢 3.3% ⇒ 差距在客户端聚合/路径。
- 两臂 spread >3%；**vsegment >10% ⇒ 按方法不进正式表**；同环境比值 1.034/1.044/1.030 **与 NOF-127 一致**，作为佐证，不替换正式表。
- 不盲目补测到 spread ≤10%；若要正式同环境格，先做 node2 稳定化再重跑。
- 不单独立项查“plain target W9 更慢”：vsegment 每 IO 已更快，target 不能解释 vsegment 端到端更慢。

### NOF-133 矩阵结果（2026-11-26，member=1，task/lane 四组）

| 组 | 配置 | Avg | P50 | P90 | P99 | P50 spread | W9 |
|---|---|---|---|---|---|---|---|
| A | plain/q4 | 258.30 | 255.99 | 263.90 | 276.31 | 8.1% | 46,920ns |
| B | vseg m=1 / 4M / q4 | 264.69 | 261.63 | 270.31 | 282.40 | 4.1% | 47,845ns |
| C | vseg m=1 / 2M / q4 | 346.87 | 270.01 | 499.33 | 635.78 | 2.9% | 60,202ns |
| D | vseg m=1 / 2M / q1 | 263.26 | 259.88 | 268.69 | 284.62 | 3.5% | 44,922ns |

- **B→C**：P50 1.032×、P90 1.847×、P99 2.251× ⇒ 拆 2 child 的成本主要在尾部。
- **C→D**：P50 0.962×、P90 0.538×、P99 0.448× ⇒ q1 全面更快，**“同 lane 争用”不成立**。
- **A→B**：P50 1.022×、P99 1.022×；**A→D**：P50 1.015×、P99 1.030× ⇒ 单 task 和双 task+q1 都在 plain 1.05 内。
- **A→C**：P50 1.055×、P99 2.301× ⇒ 双 task+q4 不达。
- child 数 probe [未获取]；按配置/代码 B=1 child、C/D=2 child（仅语义）。
- 结论：正式 2-member 下值得试 **qpairs=1**；下发 NOF-134。

### NOF-134 结果（2026-11-26，正式 2-member q4 vs q1；q1 不采用）

| 组 | 配置 | Avg | P50 | P90 | P99 | P50 spread |
|---|---|---|---|---|---|---|
| P | plain/q4 | 260.87 | 254.13 | 266.57 | 285.03 | 6.7% |
| V4 | vseg 2-member/q4 | 268.69 | 264.48 | 274.29 | 386.49 | 4.2% |
| V1 | vseg 2-member/q1 | 270.17 | 261.30 | 272.39 | 383.47 | 6.9% |

- **V4→V1**：P50 0.988×、P90 0.993×、P99 0.992× ⇒ 仅快 0.7–1.2%，小于 spread ⇒ 不显著。
- **plain→V4**：1.041 / 1.029 / **1.356**；**plain→V1**：**1.028 / 1.022 / 1.345**。
- 判据要求三项全 ≤1.05，V1 的 P99 **未过** ⇒ **q1 不采用**；正式配置保持 q4，正式结果仍为 NOF-127。
- target W9：plain 47.4µs；vseg 39–40µs（vseg 单 IO 更快 15–17%）⇒ 差额在客户端聚合/路径；剩余 2.8–4.1%（P50/P90）与 1.345（P99）判结构性，本线收口。
- V4 P99 spread 12.4% >10% ⇒ 该格仅诊断。

### NOF-135 结果（2026-11-26，m=1 只读前 400 keys；旧 2× 未复现）

| 组 | 范围 | 轮数 | Avg | P50 | P90 | P99 | Avg spread |
|---|---|---|---|---|---|---|---|
| V400 | vseg m=1 前 400 keys | **3（p2–p4；p1 剔除）** | **340.95** | **261.00** | **566.02** | **674.64** | 4.3% |
| P400 | plain 前 400 keys | 4 | 340.04 | 266.30 | 589.44 | 661.49 | 3.9% |
| V60k | vseg m=1 全 60k | 1 | 295.60 | 269.30 | 322.22 | 572.78 | n/a |

- **V400/P400（有效轮）Avg 1.003×、P50 0.980×、P90 0.960×、P99 1.020×**，四项全在 0.960–1.020 ⇒ **旧 400-key 2× 未复现**；P90/P99 为 p2–p4 重算（NOF-136），不沿用含 p1 的 4 轮值。
- p1（Avg 450.06/P50 466.63/P90 595.43/P99 671.49）整轮剔除，疑似 m=1 写后首读冷启动，原因未验证；原始值保留。
- 旧 NOF-113R 551.93/270.52 是旧 target 2142228（W9 286.6–288.7）+ 64MiB×~25 vseg 的账，**不是 400-key 规模**。
- V60k 269.30 vs NOF-133 C 270.01 = 0.997×；400 与 60k P50 几乎相同。
- 400-key W9 高值多数来自 n=1/2 空闲块，两臂都有，不作归因。
- 正式结果仍为 NOF-127；不单独立项复测旧环境/p1 异常。

---

## 7. 验收与决策规则

1. **parity 即可**：vseg P50 与 W9 都 ≤ plain × 1.05；2× 绝不可接受。
2. 若 NOF-116 达到 parity ⇒ 用双 target 配置收口，记录最佳参数。
3. 若仍差很多 ⇒ 不再调客户端旋钮；两条路：
   - 拆 target 侧 W9（`URMA_STAGE_BUFFER` 24 字段 trace，需 target 重启，待用户批准）；
   - 改 vseg 读提交逻辑（例如合并相邻 child），需用户单独批准。
4. **暂不接受修改 vsegment 逻辑代码**；任何改码必须单独拍板。

---

## 8. 风险与边界

- **target 不重启**：node3 A / node4 B 都是当前基线；重启会改 RAID0 UUID，需重新 register/mount。
- **master 可重启，但 segment 注册不持久**：每次都要重注册并保持心跳。
- **数据保留不是目标**：plain 与 vseg 会覆盖同一 target 的 LBA 区域；顺序执行，先测 plain 再写 vseg。
- **etcd/oplog**：400 keys 已知安全；1000/20k 需要先关 oplog 或预检增长（旧 2000-key 曾产生 942MB oplog）。
- **node3 独占**：royi 进程已清理；不要杀系统守护/sshd。
- **文档只用 node 标签**，不写内网 IP。

---

## 9. 产物索引

- `../HANDOVER.md`：任务书与回执（NOF-104 起）。
- `../工作汇总.md`：结论登记表与当前状态。
- `../第三阶段 node2&4/400keys-首次端到端测试结果.md`：单 target 400-key 结果与客户端分段分析。
- `../第三阶段 node2&4/条带与分块三层参数说明.md`：stripe/chunk/strip 三层说明。
- `../第三阶段 node2&4/README.md`：第三阶段环境、源码政策、NOF-104~115 里程碑。
- `oplog写入墙跟踪.md`：大数据写入的 oplog/full-state 墙时间线、根因、实验与最终解决记录（持续更新，直到 60k 写入解决）。
- `第四阶段关键结果_测试条件.md`：**只收读时延测试（vseg vs plain）**——NOF-113R/116/117/124/125，每单先条件、后结果，附禁用比法；对外汇报优先引用此文件。写入性能与 oplog 墙见 `oplog写入墙跟踪.md`。
- 本目录：第四阶段及之后的结果、回执摘要、图表、脚本摘要。

---

## 10. 工作原则

- **事实三态**：`[事实]` / `[推算]` / `[未获取]`；不猜、不用默认值填充。
- **一单一阶段**：测试 agent 只执行不决策；失败停手回报。
- **改动可回退**：env 关 / 反向补丁 / 文件归档+md5。
- **每个任务写清**：做什么、为什么、回答什么问题、边界、预计耗时。
- **不写内网 IP**，只用 node2/node3/node4。
- **连续两单没有决定性证据就停下复盘**，不钻进牛角尖。