# 第三阶段 · 400 keys 首次端到端测试结果

- 日期：2026-11-25
- 数据来源：NOF-107（诊断关，主口径）+ NOF-108（`MC_STORE_TRANSFER_DIAG=1`，用于逐 key 分解）
- 一句话：新拓扑、新源码、元数据与数据全远端的情况下，**vseg P50 555.00µs vs plain 274.00µs（2.03×）**；差距不在客户端的计划/准备/提交，而在**等完成**；plain 的 key 0–95 是慢档、96+ 是快档，**vseg 全区停在慢档，2× 全部来自 plain 的快区**。

---

## 0. 先给结论

1. **2× 复现**：P50 2.03×、target W9 2.07×；与旧 node1/node3 的 2.07× 独立吻合 ⇒ 旧环境、旧源码不是必要条件。
2. **不是客户端逻辑**：vseg 的 `resolve + prepare + submit` 合计约 15µs，与 plain 的**差值只有约 9µs**；`completion_wait` 的差值 **+206.5µs 占 read_us 总差值 218.5µs 的 94.5%**。
3. **不是 IO 数量**：两臂每遍 target IO 数约 16.9k、单 IO 128KiB，完全相同 ⇒ 差在 **target 单个 128KiB IO 的服务时间**。
4. **关键新现象**：plain 有 key 0–95 前缀慢档（~452µs，三遍一致），96+ 回落到 ~250µs；vseg 无前缀、全 key 平坦 ~470µs。在 plain 的慢区两臂几乎相等（465 vs 455，配对差仅 +2.5~+17.5µs），**2× 的绝大部分来自 plain 的快区**。
5. **仍为 L2**：两臂物理 LBA 未对齐、写入路径不同，不能把 2× 归因到“路径”以外的具体机制；`completion_wait` 只能证明“在等”，不能证明“为什么等得久”。

---

## 1. 环境与拓扑

| 角色 | 节点 | 说明 |
|---|---|---|
| master + etcd | node4 | plain：`plain_nof107`；vseg：`vseg_nof107`；端口 50056/9006/8086，etcd 127.0.0.1:2379 |
| target | node4 | SPDK nvmf_tgt，PID 2142228，md5 `3129641f…`，`:4457`，RAID0 / 11 盘 / strip **256KiB** / nsid1 |
| writer / reader | node2 | 元数据（到 node4 master）与数据（到 node4 target）**都跨节点** |

- target 全程未重启；两臂打的是同一个 target、同一份 RAID0。
- Mooncake 源码：`wangyuqi0429/Mooncake @ supercache-snapshot`，HEAD `aadab957`；SPDK 保持 wire v4。master/bench 均从该源码重编。
- 本文件不写内网 IP，只用 node 标签。

---

## 2. 测试参数

| 项 | plain 臂 | vseg 臂 |
|---|---|---|
| 数据量 | 400 keys × 4MiB = 1.56GiB | 同左 |
| 写入 | 16 threads / batch16 | T1；`VSEGMENT_CREATING` 重试补齐（23+2 次失败） |
| reader 并发 | **T1**：threads=1、batch_size=1（单并发） | 同左 |
| 顺序 | sequential | sequential |
| warmup | `--warmup_keys=128`（预热读，不计入延迟统计） | 同左 |
| qpairs | 4 | 4 |
| verify | false | false |
| 遍数 | 3 遍，报每遍与中位 | 3 遍 |
| 关键 vseg 参数 | — | `slot=1`、`stripe_size=2MiB`、`member_count=1`、`member_extent_size=64MiB`、`io_alignment=4096`、`required_medium=nof`、`reserved_ratio=0`、`submaster_count=1` |
| NoF 提交切片 | **128KiB**（`kDefaultSpdkNofSubmitChunkBytes`，两臂相同） | 同左 |
| 每 key 的 target IO | 4MiB ÷ 128KiB = **32 个** | 2 个 2MiB child × 16 = **32 个** |

两个 “stripe” 不要混：

| 名称 | 值 | 作用 |
|---|---|---|
| vsegment `stripe_size` | 2MiB | 把 4MiB key 切成 2 个子段 |
| target RAID0 strip | 256KiB | 决定每个 128KiB IO 落到哪块盘 |

---

## 3. 端到端结果（诊断关，主口径）

3 遍中位，单位 µs：

| 指标 | plain | vseg | 比值 |
|---|---|---|---|
| Min | 261.95 | 466.83 | 1.78× |
| Avg | 345.69 | 563.89 | 1.63× |
| **P50** | **274.00** | **555.00** | **2.03×** |
| P90 | 577.25 | 642.62 | 1.11× |
| P99 | 682.03 | 730.85 | 1.07× |
| Max | 745.76 | 800.43 | 1.07× |
| target W9 | 120.07 | 248.80 | **2.07×** |

- 每遍 400/0 failed；两臂 target 读窗口 n≈16,896–16,905，均等于（400+128）×32。
- **尾部收敛是假象**：plain 自身 P90/P50=2.11（长尾重），vseg=1.16（分布紧）。P90/P99 比值下降是 plain 被拉高，不是 vseg 追平。
- W9 绝对值不可跨环境比：plain 在旧环境约 68µs、本环境 120.07µs；vseg 旧约 250µs、本环境 248.80µs。只做本环境两臂对比。

---

## 4. 客户端路径结构与时间去向

### 4.1 结构图

```text
T1 get_into(key)                                        （RealClient::get_into）
│
├─ query_us ─────────────── 元数据查询：master RPC + 选副本
│                           plain 96.0 / vseg 95.5   ← 两臂几乎相同，差值 -0.5
│
└─ read_us ──────────────── 实际读（execute_ranged_read）
   │
   ├─ vsegment_resolve_us ─ vseg 计划解析（plain 没有这一段）
   │                        plain 0 / vseg 2         ← 差值 +2
   │
   ├─ nof_prepare_us ────── OpenNofSegment、buffer/对齐检查
   │                        plain 3 / vseg 4         ← 差值 +1
   │
   ├─ nof_submit_us ─────── 把 NoF task 入队到 SPDK worker pool
   │                        plain 3 / vseg 9         ← 差值 +6
   │
   ├─ completion_wait_us ── future.get()：等所有 128KiB 子 IO 完成
   │                        plain 243 / vseg 449.5   ← 差值 +206.5（94.5%）
   │                        └─ 内部包含：target 执行（W9）+ URMA 往返
   │                           + 完成回调 + 轮询/唤醒
   │
   └─ read_unattributed_us ─ 未归因
                            plain 7 / vseg 10        ← 差值 +3
```

两种路径在 target 侧的落点：

```text
plain：1 个 4MiB NoF task  → 32 × 128KiB sub-IO
vseg ：2 个 2MiB NoF child → 每 child 16 × 128KiB sub-IO（共 32）
        └─ 两个 child 是并行还是串行，本轮未测（NOF-109）
```

### 4.2 分段表（NOF-108，measured 中位，µs）

| 段 | vseg | plain | 差值 |
|---|---|---|---|
| query | 95.5 | 96.0 | -0.5 |
| vsegment_resolve | 2.0 | 0 | +2.0 |
| nof_prepare | 4.0 | 3.0 | +1.0 |
| nof_submit | 9.0 | 3.0 | +6.0 |
| **completion_wait** | **449.5** | **243.0** | **+206.5** |
| read_unattributed | 10.0 | 7.0 | +3.0 |
| **read_us** | **475.5** | **257.0** | **+218.5** |

闭合检查：`2 + 1 + 6 + 206.5 + 3 = 218.5`，与 `read_us` 差值一致。

### 4.3 为什么“resolve/submit 合计不到 10µs”就能说客户端逻辑不是差异源

这段推理要分三层，缺一层都不成立：

1. **这些段是客户端唯一要“干活”的地方。**
   - `resolve`：vsegment 把 4MiB 拆成 2 个 child 的计划；
   - `prepare`：打开 NoF segment、检查 buffer/对齐；
   - `submit`：把 task 放进 SPDK worker pool 的队列。
   三者之外，`query` 是等 master RPC，`completion_wait` 是阻塞等待，`unattributed` 是残余；都不是“客户端计算/调度”。

2. **差值算术：客户端可解释的部分太小。**
   - 非等待段差值 = resolve 2 + prepare 1 + submit 6 + unattributed 3 = **12µs**；
   - 即使把 vseg 这几段优化到和 plain 完全一样，也只能拿回约 12µs；
   - `completion_wait` 差值 **206.5µs**，占 read_us 总差值 **218.5µs 的 94.5%**。
   - `query` 两臂差 −0.5µs：它是两臂共同的 ~96µs 固定成本，**不能解释 vseg 多出来的时间**。

3. **`completion_wait` 是“空等”，不是“客户端慢”。**
   - 这段是 `future.get()` 的阻塞时间：客户端已经提交完，线程在等 32 个 128KiB 子 IO 全部完成；
   - 等待期间包含 **target 执行（W9）+ URMA 往返 + 完成回调 + 轮询/唤醒**；
   - 所以它能排除“客户端计划/准备/提交逻辑”，但**不能单独把责任判给 target**；要结合 W9 的 2.07× 才能说“重点在 target 执行/传输完成段”。

> 结论的准确措辞：**客户端逻辑不是差异源（L1 排除）**；差异发生在等待完成段，且 target 单 IO 服务时间同步翻倍（W9 2.07×）。不能写成“已证明是 target 根因”。

---

## 5. 逐 key 分布：plain 两档、vseg 一档

NOF-108 的 `read_us` p50，按 key index 分桶（µs）：

| key 桶 | plain | vseg |
|---|---|---|
| 0–31 | 455.5 | 464 |
| 32–95 | 448.5 | 469 |
| **0–95 小计** | **~452（慢档）** | **~465** |
| 96–127 | 258 | 456 |
| 128–255 | 249.5 | 475.5 |
| 256–399 | 248 | 484.5 |
| **96+ 小计** | **~250（快档）** | **~470** |

形状：

```text
key:   0────────────────95│96──────────────────────399
plain:      ~452µs 慢档     │        ~250µs 快档
vseg:       ~465µs          │        ~470–485µs（平坦，没有快档）
                            ↑
             两臂在慢区几乎相等；2× 的绝大部分来自右侧快区
```

- plain 的 key 0–95 慢档**三遍一致**，且 warmup 已经读过这批 key，仍慢 ⇒ 不是冷启动/连接预热。
- vseg 没有前缀：13 个桶全部落在 447–521µs，按 key index 平坦。
- 400 个 key 全配对：**vseg 更慢 359/400 = 89.8%**；0–95 桶差仅 +2.5~+17.5µs，96+ 桶差 +198~+252µs。
- 这不表示“vseg 变慢了”，而是 **plain 从 key 96 起进入了快档，vseg 一直没有进入**；vseg 的 ~470µs 恰好接近 plain 慢档的 ~452µs。

**注意口径**：这是 `read_us`（端点）按 key 的分布；W9 是 target 全窗口平均，**没有按区拆分**。所以“2× 来自 plain 快区”是端点层面的结论，不是 target 层面的分区结论。

### 5.1 由此产生的两个互斥假设

- **(a) 串行两波**：两个 2MiB child 没有并行，一个完成后再跑另一个 ⇒ 一个 key ≈ 2 × 单 child。
- **(b) 每 child 自身慢**：两个 child 并行，但 target 对 vseg 的每个 128KiB IO 就是 ~248µs（慢档水平），与 child 数无关。

两者对“怎么修”完全不同：  
(a) 改 child 的提交/分配或合并相邻 child；  
(b) 要查 target 对 vseg IO 的执行路径。  
**NOF-109 用已有日志里的 `nof_vsegment_child` 时间戳来分离，不需要新测量。**

---

## 6. 本轮顺带补掉的证据与混杂

### 6.1 读路径确认（补 NOF-107 缺口）

- NOF-108 逐 key 日志：vseg 臂 **528/528 `type=vsegment`**；plain 臂 **528/528 `type=nof_remote`**；status 全 `read_ok`。
- vseg reader 用的是 `--scenario=segment_read`、没有带 vseg flag；源码 `stress_cluster_bench.cpp:3449` 本来就禁止该场景与 `--vsegment_nof_io` 同用，而 `real_client.cpp:4623-4626`（supercache aadab957）表明 `type` 取自 master 返回的 `replica_type`。逐 key 日志把这条机制证据补齐了。

### 6.2 臂序排除

- NOF-107：plain 先跑、vseg 后跑。
- NOF-108：vseg 先跑、plain 后跑（vseg master 已停）。
- plain 后跑仍 **363.88–372.40µs、W9 120.08µs** ⇒ 臂序、master 进程差异、target 运行时长都不是原因。

### 6.3 诊断开销不对称（重要）

| 臂 | diag off P50 | diag on P50 | 增量 |
|---|---|---|---|
| vseg | 555.00 | 586.68 | +31.7 |
| plain | 274.00 | 365.44 | +91.4 |

⇒ **diag-on 的比值 1.61× 不能当真实比值**；分段占比、W9、配对差可用，**绝对比值一律用 diag-off 的 2.03×**。

### 6.4 其他观察（不影响本结论）

- NoF 提交切片 128KiB 是客户端常量 `kDefaultSpdkNofSubmitChunkBytes = 1<<17`（`transfer_task.h:625`），可被 `MC_NOF_SUBMIT_CHUNK_BYTES` 覆盖；两臂相同，不是 vseg 特有。
- plain master 重启后 `used` 聚合显示 0B（Keys=400 正常、读 400/400 成功）；读取不受影响，属展示层未重算。
- vseg writer 有 23+2 次 `VSEGMENT_CREATING` 失败后补齐；写入历史与 plain 不同，是布局混杂之一。

---

## 7. 结论等级与“不能说的话”

### 7.1 已成立

| 等级 | 内容 |
|---|---|
| L0 观测 | P50 555.00 vs 274.00µs；W9 248.80 vs 120.07µs；两臂每遍 32×128KiB、n≈16.9k |
| L0 观测 | plain key 0–95 前缀慢档 ~452µs（三遍一致），96+ ~250µs；vseg 全区 ~470µs |
| L1 排除 | 旧 node1/node3 环境与 be10be29 源码不是必要条件（新环境/新源码复现 2×） |
| L1 排除 | 客户端 resolve/prepare/submit 不是差异源（差值 ~9µs，等待占 94.5%） |
| L1 排除 | 臂序、master 进程、target 运行时长（plain 后跑不变） |
| L1 排除 | IO 数量/切片大小（两臂同为 32×128KiB） |
| L1 排除 | 读路径混淆（vseg 528/528 = vsegment） |

### 7.2 仍是假设（L2）

- vseg 为什么“没有快档”：**(a) 两个 child 串行两波** 或 **(b) 每 child 自身慢**；未分离。
- plain 的 key 0–95 慢档是否由**物理盘位置**造成：本环境实测存在该前缀，但**未做物理 LBA 对照**，不能归因。
- 两臂数据写入路径不同（plain 16t/batch16、vseg T1 且有重试），物理 LBA 未对齐 ⇒ “路径 vs 布局”仍未分离。

### 7.3 不能说的话

- ❌ “已证明 target 是根因”：`completion_wait` 还包含 URMA 往返与完成通知；W9 只覆盖 target 的 exec→bdev_cpl。
- ❌ “vseg 比 plain 慢 1.61×”：diag 开销不对称，1.61× 无效。
- ❌ “vseg 的尾部已经追平”：P90/P99 收敛是 plain 长尾造成的。
- ❌ “盘首慢区是物理位置导致”：只证明本数据集上 key 0–95 有前缀慢档。
- ❌ 与旧 node1/node3 的数字混比：环境/源码/拓扑全变。

---

## 8. 产物与下一步

### 8.1 产物

| 文件 | 说明 |
|---|---|
| `run/nof107_summary.txt` | NOF-107 汇总（md5 `777b09d8…`） |
| `run/nof108_vseg_p{1,2,3}.log` | vseg diag 日志（各 6.5–8.9MB，528 行 breakdown/遍） |
| `run/nof108_plain_p{1,2,3}.log` | plain diag 日志（同上） |
| `run/nof108_summary.txt` | NOF-108 汇总（md5 `9f0b86ff…`） |

### 8.2 复现关键 env

```bash
# 逐 key 分解（两臂同开；采样率默认 1.0）
MC_STORE_TRANSFER_DIAG=1
# 注意：diag 会抬高绝对值且两臂不对称，绝对比值用 diag-off
```

### 8.3 下一步

**NOF-109（纯离线）**：解析 `nof_vsegment_child` / `nof_vsegment_operation`，看两个 child 的提交/完成时间是否重叠、各自延迟多少，判定 (a) 还是 (b)。判决后再决定是改 child 提交/合并，还是查 target 对 vseg IO 的执行路径。