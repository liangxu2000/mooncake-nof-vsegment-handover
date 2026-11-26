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
| target B（node4） | PID 2142228（11-25 15:20:22 起）；NQN `...urma-gpu-test`、SN `URMAGPU0001`；RAID0 UUID `0470fe3a…`；11/11、strip 256KiB、nsid1、4457；ERROR 6 条（历史 pci_event socket read error 105） |
| master | `vseg_nof116` 运行中（2360608/2360611）；Keys=400；2 个 segment；quota generation=1、member_count=2、extent=50GiB、logical_capacity=100GiB |
| node2 | 无 bench/register 残留 |
| 当前任务 | **NOF-118：零写查剩余 ~25µs（vseg@64K/128K diag，两 child 时序 + 客户端分段）** |

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