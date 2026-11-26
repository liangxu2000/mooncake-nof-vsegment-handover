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

## 3. 当前环境状态（2026-11-26 NOF-115 后）

| 对象 | 状态 |
|---|---|
| target A（node3） | PID 3329243；NQN `...urma-gpu-test-node3`、SN `URMAGPU0003`；RAID0 UUID `2c5bf1f2…`；11/11 online、strip 256KiB、nsid1、4457、0 ERROR、零 IO |
| target B（node4） | **已重建**：PID 2391418；NQN `...urma-gpu-test`、SN `URMAGPU0001`；RAID0 UUID `c19b91e1…`；11/11、strip 256KiB、nsid1、4457；0 ERROR |
| master | NOF-119 A 后 probe master 已停；旧 `vseg_nof116` 因 target B UUID 变已失效；当前无业务 master |
| node2 | 无 bench/register 残留 |
| 当前任务 | **NOF-122：`oplog_batch_max_entries=64` 修 2MiB 事务墙；重启 master（不动 target）；重试 60k vseg/plain** |

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
6. `extent` 是**每个 member 在 NoF segment 上的物理 extent**，不是节点 DRAM；当前 50GiB/member → 每个 vsegment 逻辑 100GiB。

### 4.3 单 target 下的结论（第三阶段，仍有效）

- vseg P50 555µs vs plain 274µs（2.03×）；target W9 248 vs 120（2.07×）。
- 差值 95% 在 `completion_wait`；客户端 resolve/prepare/submit 合计 <10µs。
- 两个 child 100% 并行，op span = 1.10× 单 child；不是串行。
- worker、inflight、qpair、数据 LBA 均排除。
- **2× 是按字节的服务成本，不是每 IO 固定开销**；chunk 加大不降时延，W9 反而变差；target 单 IO 在 ≥256KiB 时被夹到 256KiB。
- 剩余解释：target 对 vseg 请求的每字节服务路径（尚未拆 W9）。

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

| 臂 | P50 均值 | P50 中位 | 最好一遍 | W9（每 target） |
|---|---|---|---|---|
| plain（node4 单 target） | **266.36µs** | 266.49 | 264.86 | **120.33µs** |
| vseg（node3+node4） | 323.59µs | 295.47 | 280.17 | **157.3µs** |

- 比值：均值 **1.215×**、中位 **1.109×**、最好 **1.058×**；W9 1.31×。
- vseg 6 遍方差大（280–461µs，461 为离群）；去掉离群后均值约 296µs（1.11×）。
- 双 target 把单 target 的 **2.03×** 压到约 **1.1×**，但**未达 parity（≤1.05×）**。
- 两个 target 写 n=6400/6402、读 n≈8450，均分；`type` 全 vsegment。

### NOF-117（已完成，未达 parity）

| 项 | vseg@128K | vseg@64K | plain@128K |
|---|---|---|---|
| P50 均值 | 293.90 | **286.91（−2.38%）** | 266.36 |
| P50 中位 | 291.37 | 293.91（+0.9%） | 266.49 |
| Avg / P90 | 392.48 / 537.46 | 371.10 / 508.84（−5.4% / −5.3%） | 338.51 / ~567 |
| target W9 | 157.58 | **130.50（−17.2%）** | 120.33 |
| 每 target n | ~8,449 | ~16,897 | 16,898 |

- vseg@64K / plain：P50 均值 **1.077×**、中位约 **1.10×**；单遍 272.27（1.022×）达线，另两遍 294 未达。
- **target W9 明显改善，但 P50 不动** ⇒ 队列深度/每 target IO 数不是 P50 剩余差距的主因，停止调 chunk。
- 剩余 ~25µs 不在 target 单 IO 执行段；更可能在客户端两 child 聚合/等待、两跳网络差异、或 64K 带来的客户端开销翻倍。
- 当前待裁决：A 接受 1.08–1.10 近 parity；B 零写查剩余 ~25µs（diag/child timing）；C 改码或第三 target（均不轻）。

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
- 本目录：第四阶段及之后的结果、回执摘要、图表、脚本摘要。

---

## 10. 工作原则

- **事实三态**：`[事实]` / `[推算]` / `[未获取]`；不猜、不用默认值填充。
- **一单一阶段**：测试 agent 只执行不决策；失败停手回报。
- **改动可回退**：env 关 / 反向补丁 / 文件归档+md5。
- **每个任务写清**：做什么、为什么、回答什么问题、边界、预计耗时。
- **不写内网 IP**，只用 node2/node3/node4。
- **连续两单没有决定性证据就停下复盘**，不钻进牛角尖。