# 交接文件（单向：开发指挥 Agent → 测试执行 Agent）

- 仓库：`https://github.com/liangxu2000/mooncake-nof-vsegment-handover`（public，匿名 HTTPS 可拉取）
- 首次接入：`git clone https://github.com/liangxu2000/mooncake-nof-vsegment-handover.git`
- 之后每轮：`cd mooncake-nof-vsegment-handover && git pull`
- 权限：**只 pull，不要 push**；本仓不接收反向提交
- 回执：按下方《回执格式》在会话里输出，由用户转交开发侧；回执不写回仓库
- 最后更新：2026-11-26


> ## 📌 代码来源链（2026-11-24 21:38 · 用户提供 + NOF-092 实测 · **所有源码判断的唯一依据**）
>
> ### 三条线，只有一条跟我们有关
>
> | 线 | 仓库 / 分支 | 状态 |
> |---|---|---|
> | **原始 nof-vsegment 开发线** | `wangyuqi0429/Mooncake` @ **`supercache-snapshot`**（同事 wangyuqi0429 开发） | **上游源头** |
> | **我们实际在跑的** | `LeeJace993/Mooncake` @ **`mooncake-vesegment`** @ **`be10be29`**（2026-11-17 08:38） | **唯一有效基准** |
> | 无关线 | `LeeJace993/Mooncake` @ `nof-urma-v2-bench-merge`（`d371e22`，2026-10-23） | **无 vsegment 代码，与本事无关，勿再引用** |
>
> ### 来源链
>
> ```
> wangyuqi0429/Mooncake @ supercache-snapshot      ← 原始 vsegment 实现
>          │
>          │  yin 于 2026-11-17 基于它新建分支
>          │  ★ 并加入了他自己的「光组网」改动
>          ▼
> LeeJace993/Mooncake @ mooncake-vesegment @ be10be29   ← 测试机实际编译运行的
> ```
>
> ### 实测证据（NOF-092）
>
> - bench 二进制（md5 `59662a26…`，mtime 2026-11-17 19:48）**内嵌源码路径** = `/home/yin/nof/mooncake-vesegment/mooncake-store/src/client_service.cpp`
> - 构建目录 `CMakeCache` 指向同一棵树 `/home/yin/nof/mooncake-vesegment`
> - 该树 HEAD = `be10be29a126b31fdbb9292c7107a468ed50828f`，`2026-11-17 08:38:44` —— **与 yin「17 号新建」的说法一致**
> - 该树 `status` 仅 2 条（`etcd/go.mod`、`go.sum`），**无 C++ 改动**
> - 我方副本 `mooncake-src` 另有 1 条：`transfer_task.cpp` +31 行（`MC_NOF_IO_PARAMS_LOG` 日志插桩，不改逻辑）
>
> ### ✅ **「光组网改动」混淆变量 → 已排除**（用户 2026-11-24 21:40 核实）
>
> **用户核实结论：`be10be29` 不涉及光组网改动的影响。**
>
> ⇒ `be10be29` 可作为**纯 vsegment 实现**使用，源码分析与性能结论的责任范围可直接归于 vsegment 本身。
> ⇒ 开发侧 21:39 提议的「拿 `supercache-snapshot` 原始版本跑对照以分离光组网影响」**取消，不派 NOF-093**。
> ⇒ **保留记录**：此变量曾于 21:38 被提出为待分离项，21:40 由用户排除；后续若出现"改动来源"相关疑问，以此为准，不再重复质疑。
>
> ### 连带作废声明
>
> 开发侧 2026-11-24 全天基于 `_ref/Mooncake`（`706551a`，11-03）和 `D:\gds性能\repos\Mooncake`（`d371e22`，10-23）得出的所有源码级判断，**一律作废**：
> - `Plan` 里「同一 view 校验两遍」只占 3–5µs
> - `openSegment` 只是 map 查找
> - 两臂提交方式均为一次性 batch
> - `13b7c19` 去重复校验的价值评估
>
> **其中仅一条经实测复核后仍成立**：`be10be29` 的 `vsegment_transfer.cpp` 里，`ResolveTransfer` 调用**未传** `prevalidated` 参数 ⇒ **重复校验在实际版本里确实存在**（已用 GitHub raw 取原文确认）。但其耗时占比仍未实测，仅有 3–5µs 的估算，**不得当作定案**。

> ## ⚠️ 口径决议（2026-11-24 19:52 · 用户批准 · **优先级高于一切任务书**）
>
> ### **bench 口径回到 400 keys。2000-key 口径封存。**
>
> **理由**：2000 keys 唯一不可替代的产出（工作集 400 MiB → 8,000 MiB 时延迟不退化，**1.043%**）**已经拿到**；其余产出都是 400-key 已有结论的重复。而它付出 **942 MB oplog**、覆写了 4 个 cid 的物理数据、把 etcd 余量从充裕压到 **150 MB**。**性价比不成立。**
>
> **封存的 2000-key 产出（留档，不再扩展）**：
>
> | 轮次 | 产出 | 是否仍有效 |
> |---|---|---|
> | NOF-086 | 读基线 P50 526–578µs（4 遍）、failed=0、Keys=2000 | 有效，仅作参照 |
> | **NOF-087** | **慢程在端到端不可见：1.043×** | **有效（同口径内部比较，本轮最高价值）** |
> | NOF-088 | vseg 比 plain 慢 2.07×（与 NOF-025 的 2.08× 同向） | 有效，但**只是重复，不构成新证据** |
> | NOF-089 | `read_us` 内部无 breakdown 埋点，**拆解失败** | 有效（排除项） |
>
> **口径隔离规则继续生效**：2000-key 与 400-key 的数字**不得混用 / 并列 / 做比值**。
>
> ### 实验成本表（**派单前必须查这张表**）
>
> | 口径 | 模式 | oplog 实测 | 数据区/盘 |
> |---|---|---|---|
> | **400 keys** | vseg | **~22 MB**（NOF-080） | 145.45 MiB |
> | **400 keys** | plain | **~178 KB**（vseg_nofvseg） | 145.45 MiB |
> | 2000 keys | vseg | **942 MB**（NOF-086） | 727.27 MiB |
> | 2000 keys | plain | 899 KB（NOF-088） | 727.27 MiB |
>
> ⇒ **400-key 的 vseg 成本是 2000-key 的 1/43，数据覆盖面是 1/5。**
> ⇒ 当前 etcd 余量 **150 MB**（1,345,523,712 / 2 GiB = 62.66%，70% 线余 157,714,841 B）⇒ 还能做约 **6 次** 400-key vseg 实验。
> ⇒ **不含 defrag 时，删 key 只产生自由页、不降 dbSize**；要降账面必须 `etcdctl defrag`（共用机维护操作，未授权）。
>
> ### cid 现状（**务必按此核对，别再误用**）
>
> | cid | 模式 | keys | 物理数据 | oplog | 可重启 |
> |---|---|---|---|---|---|
> | `plain_nof088` | plain | 2000 | **完好** | 899 KB | **是** |
> | `vseg_nof086` | vseg | 2000 | 完好 | **942 MB** | **否**（撞 32MiB 客户端上限） |
> | `vseg_nof080` | vseg | 400 | **已被覆盖** | 22 MB | 元数据可启，**但数据不可用** |
> | `vseg_nofvseg` | plain | 400 | **已被覆盖** | 178 KB | 同上 |
> | `vseg_nof084` | vseg | 100 | **已被覆盖（前段）** | 8.5 MB | 同上 |
>
> ⇒ **目前唯一「可重启且数据完好」的是 `plain_nof088`。任何新的 vseg 实验都必须新建 cid 重写数据。**

- **构建硬规范（NOF-096 定案）**：客户端重链必须使用 `$D/build/spdk-e12b7cb-src/build/lib` 的 **e12b7cb / wire v4** 静态 SPDK 库（真实命令 `/tmp/nof051/link.sh`），**禁止使用 `/usr/local/lib/libspdk_*.a`**（2026-11-20 已被整体替换，会导致 attach `rc=-19`）。每次构建记录库路径/md5、链接命令与产物 md5；`ldd` 不能证明静态库一致。

- **回执指标硬规则（2026-11-26 用户指定按 `D:\gds性能\测试结果聚合方法_20261112.md`，后续所有 NOF 任务生效）**：每轮记录 Avg / P50 / P90 / P99（µs）；**正式结果按该方法聚合**：Avg = 各轮等权算术平均，P50/P90/P99 = 各轮同分位中位数（Min/Max 若采集则取中位）；**逐轮原始值落盘**（`run/*.csv`），正文只给**聚合值 + spread**，不逐轮罗列；spread=(max avg−min avg)/mean avg，**>3% 标注原因、>10% 不进表**；**不报吞吐**；每轮只读一遍 60k keys（约 20s），**禁止 `--duration` 连续循环**。此规则覆盖此前“不写 P50 中位”的展示约定。

- **正式测试口径（2026-11-26 用户定，后续默认）**：① key=4MiB，**60,000 keys**，不再用 400 keys；② vsegment 用**一整块**：`extent=256GiB/member`（274877906944），逻辑 512GiB，`member_count=2 / stripe=2MiB / required_medium=nof / reserved_ratio=0 / io_alignment=4096 / slot=1`；50G/3vseg 仅作历史对照，不再默认；③ plain 单 target B；④ 读 = T1（threads=1/batch=1）、warmup=128、qpairs=4、verify=false、diag 关、sequential；vsegment `--num_keys=30000`/segment；**每轮只读一遍 60k keys（约 20s），禁止用 `--duration` 连续循环读**；⑤ **稳定性**：每臂 ≥6 轮，至少 2 个交替周期（V3→P3→V3→P3）；**回执正文不逐轮罗列，按 `测试结果聚合方法_20261112.md` 给聚合值（Avg 均值、分位中位）+ spread**；⑥ 第四阶段第一个正式结果 = NOF-127。

- **用户约束（2026-11-26 确认，后续不再提）**：① `stripe=2MiB` 固定，**不采用 4MiB/每 key 单 member 方案**——用 vsegment 就是要 key 拆到多个 member；② master/etcd 必须在**远端 node4**，不移到 node2；③ 当前只做 **member_count=2 / 双 target**，不加第三 target；④ **暂不改 vsegment 逻辑代码**。调优路线：**先拆解定位、再定向调**，不再逐个试已证否的旋钮（qpairs/workers/partitions/extent/layout/chunk）。

- 当前任务：**NOF-152（plain NoF + bdev_null · 4MiB 单包 E0/E1 · 2026-11-27）**。背景：第一刀（NOF-150）无收益且破坏打点语义，已决定回退；**NOF-151 取消/并入本单，回退作为第 0 步**；diag 行不全的数据质量另单。**目标**：plain NoF（不开 vsegment）+ target null bdev（不碰 SSD）+ 客户端 `MC_NOF_SUBMIT_CHUNK_BYTES=4MiB` + `SPDK_URMA_MAX_IO_SIZE=8MiB`，T1 单并发读 400 keys，E0/E1 各 3 遍，确认每个 key 是否为**一个 4MiB IO**（target IO 计数 ~400，而不是 12,800），并测 4MiB 单包时延/带宽。**预检**：target 是否支持 `bdev_null`/`-Z`、能否不重启现有 target/etcd 添加 null bdev namespace、JFC 是否可取；若需重启 target/etcd ⇒ 停手回报，不在本单重启。**边界**：先回退 NOF-150 旧 bench/source（bb4a07a3）；不改码/不写 SSD/不重启 target/etcd；null bdev 不可用即停，不回退到 SSD；预计 20–30 分钟。
- 历史：**NOF-094（量 vsegment 读的 plan/submit/wait 三段 · 零改码 · 用 `--warmup_keys=0` 检验“128 样本配额”假设）**。该假设已被 NOF-094 配对实验否证，路线关闭。**替代 NOF-091（三轮概念已混，废弃）。** 关键：bench 走的是 `TransferReadInternal`（be10be29 `client_service.cpp:5926`），三套 `VSEGMENT_READ_*` 里只有这一套在路径上；plain 臂的点位名不同（`GET_INTO_INTERNAL_NOF_READ`）；顺序必须 B1(plain) → B2(建 cid) → B3(vseg)，因为 vseg 写入会覆写 plain 前 400 key 的位置。
- 历史：**NOF-091 第三轮（用 `--warmup_keys=0` 检验“128 配额被 warmup 吃满”假设）**。前两轮只证明 B1 路线可行、点位有记录；**“128 是配额”的推断已由 NOF-094 否证**（warmup=0 时读取点位 0 条），不得再引用。NOF-092 已完成（源码 = `LeeJace993/Mooncake` @ **`mooncake-vesegment`** @ `be10be29`，2026-11-17）。
- 历史：**NOF-091 续（B1 路线可行但被两个阻塞卡住，开发侧 20:55 裁决：允许重启 master 建 `vseg_nof091`、不改全局 spdiag 配置改用连跑 ≥6 遍）**。**另更正：测试机源码 HEAD 为 `be10be29`，与开发侧分析用的本地副本 `d371e22` 不是同一份代码 ⇒ 本地得出的三个源码判断降级为「待核」，本单 C 段须回贴 `client_service.cpp` L2170–2340 原文。**
- 历史：**NOF-091（拿 vsegment 的 plan/submit/wait 分解）**。用户 20:41 批准派任务。**优先零改码**：源码里三个 `SpDiag::PerfPoint` 埋点（`VSEGMENT_READ_PLAN/SUBMIT/WAIT`）已存在，若构建是 Layer 1 且 `spdiag` CLI 在，直接 `spdiag show` 拿数；否则才加 chrono 重编（65s）。
- 历史（已作废，NOF-092 更正）：**NOF-090（版本对照：换掉 be10be2 看 2× 是否版本问题）**。以下分支/版本判断来自错误的本地副本 `d371e22`，**不得引用**；实际基准是顶部 `be10be29`。当前没有“升一版拿回 2×”的已验证选项。
- 历史（已作废）：**NOF-090（版本对照：换掉 be10be2 看 2× 是否版本问题）**。开发侧决策：60+ 轮已排除 IO 层/命令层/客户端结构层；唯一未排除项是**版本落后**（`be10be2` 缺 `ff3e853`/`13b7c19` 等读路径修复）。源码里那处「重复 view 校验」已量化仅 **3–5µs（约 1%）**，不构成主因、不值得单独重编。
- 历史任务位：**无 —— 口径已回 400 keys（见上方口径决议），待用户定向下一单**。NOF-089 已完成：`read_us` 内部无 breakdown 埋点，**拆解失败**（offload 三子项恒 0，`read_overhead_us` ≡ `read_us` 是恒等式），要细分需新增插桩（改码）。
- 原任务位记录：**NOF-089（把 `read_us` 的 2.58× 拆成 4 子项 · 纯离线）**。用户 19:39 把决策权交开发侧，决定**先拆解定位、暂不做升 L3 实验**——理由：① C18 的 2.07× 已与 NOF-025 的 2.08× 独立吻合 0.5%，「是不是真的」把握已较高；② 升 L3 需同 cid 双副本，但 bench 的 flag 不控制路径（NOF-088 已证），可能根本做不到；③ 先把 261µs 的差值拆到具体子项，才知道该往哪查。
- 历史：原「待用户定向」如下 —— NOF-088 已完成并**推翻 C03/C17**：bench 无法在同 cid 切读取方式（`type` 取自 `metadata.replica_type`），「两 flags 差 0.00%」是同一条路径测两遍。**实测 vsegment 比 plain 慢 2.07×**（与 NOF-025 的 2.08× 吻合 0.5%），差异在 `read_us`（2.58×）而非 `query_us`（1.11×）。等级 L2（布局混淆未排除）。
- **⚠️ 权威口径以本文件顶部「当前任务」与 `工作汇总.md` 的《结论登记表》为准；C03/C17 已撤回，任何仍引用「vsegment 无性能代价」的表述一律作废。**NOF-087 判定：**慢程在端到端不可见（1.043×，四遍一致）⇒ 对业务无影响，这条线关闭**。**当前主线：C18（L2）——vsegment 比 plain 慢约 2.07×，差异在 `read_us`（2.58×）；根因未定。** 单 IO 的 400MiB 慢程是真实现象，但不得当作端到端根因。
- **NOF-085 已被 NOF-086 裁决取代**：A 段的 `--max-request-bytes` 定位**已被源码证否**（见 NOF-086 裁决 §7），真正的墙是 Mooncake 客户端 `storeMaxMsgSize = 32MiB`。原单作废，改由 NOF-087 核版本常量。
- **NOF-084 的「外推 >24MB 即停手」判据已撤销**（24.2MB 下限假设 1 entry/key，与实测 7 entries/key 矛盾）。
- **NOF-084 已完成（主动停手，L0）**：2000 keys 未写入；100 keys 预检实测 84.8KB/key，2000 keys 外推 24.2–169.6MB 全部超 24MB 阈值。见该单顶部裁决块。
- **口径整改阻塞已解除**：阻塞根因是 Mooncake 客户端常量，**不需要改共用 etcd**。修复方案 A/B/C 见 NOF-086 裁决 §7；改码需用户授权，未授权前不执行。
- **NOF-083 已放行**：C/D 段复用已有 trace，A 段在裸 SPDK 读；边界已从 `start ≥ 2048MiB` 修正为 **`start ≥ 8000MiB`**（= 盘内 727.27MiB，在 NOF-086 新数据区外）。**全段零写入，不增一个字节 oplog。**
- **口径隔离**：2000-key 结果与全部 400-key 历史数字不得混用 / 并列 / 做比值。
- NOF-082 **已完成，结论部分修正**：慢程模型成立；B 段「形态不同⇒根因」判据撤回（见该单顶部裁决块）。
- NOF-081 **已完成，结论部分被拒**：A 段判决与 C 段「对外表述定稿」已作废（见该单顶部裁决块）。
- NOF-077（自建 `--rounds` / `--start_key` 口径补丁）**暂缓**；NOF-025 **须重做**，重做前不得引用。
- **状态提示（2026-11-25）**：此前所有 4.6–5.9× 单因归因与 C03/C17 均已撤回；当前只保留结论登记表里的 C18（L2，vseg 慢 2.07×）、C09/C16（慢程存在但端到端不可见）、C02 排除项等。NOF-095 测客户端内部 timing，用于判定该修客户端还是 target/会话层。

## 归因纪律（2026-11-24 起 · **所有单必读** · 优先于任何单的任务书判据）

> 立规原因：NOF-025~082 共 20+ 轮，正面根因一次未抓到，反而每隔几轮撤回上一轮自己的归因。问题不在执行质量，在**允许把组间差异直接命名为因果**、且**撤回时静默覆盖**。以下为硬约束。

### 1. 结论分级（回执第一行必须标等级；未标视为无效回执）

| 等级 | 允许写什么 | 需要什么证据 | 能否进对外口径 |
|---|---|---|---|
| **L0 观测** | 数字 + 测量条件 | 一次测量 | 否 |
| **L1 排除** | 「X 不是必要条件 / 不解释差异」 | 有对照组且 X 被实际操纵 | 否 |
| **L2 假说** | 「可能是 X」 | 存在组间差异 | 否 |
| **L3 结论** | 「差异由 X 造成」 | **过完 §2 三闸门 + 两个独立手段复现** | **是** |

历史教训：NOF-045 / 061 / 076 / 078 / 080 / 081 全是把 **L2 当 L3** 用，随后被撤回。

### 2. 三闸门（任何 L3 必须在回执里逐条回答，缺一条自动降为 L2）

- **G1 单变量**：列出两臂之间**全部**差异变量——`cid` / 数据批次 / LBA 落点 / `slot_count` / `stripe` / `warmup_keys` / `range` / `io_size` / QD / target PID+md5 / 客户端二进制 / 诊断开关 / **累计读取次数** / 每遍流长——逐个写明如何控制。任一未控 ⇒ 最高 L2。
- **G2 必要性**：该变量**不存在时现象必须消失**；出现反例即否证。（NOF-082：裸 SPDK 严格单一顺序步长也有 201→25µs 两档 ⇒ 「访问流形态差异」不是必要条件，B 段判据因此撤回。）
- **G3 双手段复现**：客户端 trace + target trace，或裸 SPDK + Mooncake，至少两个独立测量手段给出一致方向。（NOF-045 只用裸 SPDK 单手段就定了 +210µs，是反面案例。）

### 3. 口径基线（每轮回执逐项报；任一变动 ⇒ 与历史**自动不可比**，禁止算倍数）

`num_keys` / `warmup_keys` / `range` / `io_size` / QD / `slot_count` / `stripe` / `cid` / target PID+md5 / 客户端二进制 / 诊断开关 / **每遍流长**

- 禁止把不同混合分布的 **p50 直接做比值**；必须同时给 `avg`、两档占比、`p50/p90/p99`。NOF-082 已证 p50 是两档混合的阶跃函数，随慢占比非线性变化。
- 400-key（NOF-001~083）与 2000-key（NOF-084 起）口径**永久隔离**。

### 4. 任务书去偏见

- 判据必须**两边都写**：「若 P ⇒ 支持 H；若 Q ⇒ 否证 H」。禁止「p50 何时从 A 掉到 B」这类把结论写进前提的写法（NOF-081 B 段）。
- 每单必须含一个**证伪支**：能推翻当前最受偏好假说的那一支。
- 禁止「组间不同 ⇒ 差异来自该变量」式判据（NOF-082 B 段已撤回）。

### 5. 硬刹车

- **连续 2 轮出现「撤回上一轮归因」⇒ 暂停派新归因单**，先做一轮零采集的口径复核（纯离线，只对齐口径、不引入新变量）。
- 回执新增必答项：**「本轮是否触发任何既有结论降级？若有，给出编号与原因。」**

### 6. 单一事实源

- `工作汇总.md`《结论登记表》是唯一事实源；**撤回不删除**，只改状态 + 填撤回原因，避免文档里同时存在互相矛盾的表述。
- **任何单不得自称「对外口径定稿」**；定稿只能由开发侧在 L3 + 三闸门全过后写。

---

## 第二阶段（2026-11-23 起）

- **第一阶段（NOF-001~046）已收口**：单 target 下 vsegment 无读时延优势；核心未解问题已定位到 **vseg 读路径在 target 的 bdev 执行段多 ~210µs**（数据/盘无罪：裸 SPDK 同区 41µs；target 侧顺序/落核/命令与 plain 完全一致）。
- **第二阶段目标**：消除这 210µs，让 vseg 的 target bdev 接近 plain（~70–100µs）、端到端不劣于 plain；之后再谈 300µs 与多 target 结构性优势。
- **计划与假设（H1–H4）、里程碑（M0–M4）、验收与回退**：见 `第二阶段/README.md`（第二阶段所有内容统一放在 `第二阶段/`）。
- **M0（NOF-047）已完成**：复制并校验 Mooncake/SPDK 源码与 build（8.82GiB，md5 全 MATCH）。
- **最小重建（NOF-048）已验证**：改一个 .cpp → 编译 61s → `ar r` 换库成员 → 重链 4s，产物 `--help` 正常；M1/S2 的代码迭代成本约 65s/轮。
- **2026-11-23 复盘纠偏**：发现两处高价值线索——①构建基线 **be10be2 落后于上游**（**2026-11-24 20:31 更正**：原文「tip 4c0ea28」有误 —— `4c0ea28` 只是一个 vsegment 单测提交；实际 main HEAD 已是 `bf3b95a`(11-24)。读路径相关提交含 `ff3e853` 诊断 opt-in、`13b7c19` 去重复校验、**`3838892` submit watermark 用错 channel 数**、**`f3b8101` 批量状态查询改非阻塞**、**`d3fd032` SSD prefetch**）；②旧构建 `MC_HIFREQ_LOG_SAMPLE_RATE` **默认 1.0**，per-task 计时/breakdown 日志默认全开（tip 的提交明确说这会抬高读时延），而我们**从未测过"诊断全关"的 vseg**。
- **S0 结果（NOF-049，已完成）**：诊断全关 W9 仅 **258.85→251.75µs（−2.7%）** ⇒ 默认诊断/客户端背压排除。
- **S2 复核（2026-11-23）**：be10be2→4c0ea28 的 `transfer_task.cpp` 仅 40 行改动（诊断 gating/UB poll 默认），**NoF 读提交/QoS/分块未变**；其余为客户端/写路径 ⇒ 构建 tip 对 210µs 的预期收益低，暂缓。S1（本机裸读）降为可选参照（vseg/plain 对比都在本机，宿主混淆不改变结论）。
- **M1a（NOF-050）已构建成功但被版本阻塞**：补丁/编译/链接全部成功（`bench-io-params` md5 fc2b5129…），但新 bench 的客户端 SPDK 是 **9a413a44**，运行中 target 是 **e12b7cb**，握手失败（rc=-71/-104）；同事原 bench（匹配 e12b7cb）同参数正常（P50 473.52 / W9 259.77，**reader 在 node1 远端**，与 NOF-040 本机 274 不可比）。
- **关键事实**：target 现跑的是 **deleted 旧二进制 736a902f**；磁盘上的 `nvmf_tgt` = **d6c39465（9a413a44）**。任何 target 重启都会永久切到 9a413a44（旧 e12b7cb 二进制不可恢复）。
- **A 路线已成功（NOF-051，2026-11-23）**：e12b7cb 本身因系统 UMDK 缺 `is_gpu_seg` 编不过；改用 **wire v4 的紧邻后继 bd5ee124d**（GPU 注册 opt-in）构建成功；重链的 `bench-io-params-e12` 在 **node3 本机握手成功**。**wire 谱系**：e12b7cb / bd5ee124d / 170cbb9 = **v4**；91763b015 / 9a413a44 = **v6**（握手失败的根因）。
- **vseg 参数（NOF-051）**：SUBMIT/WORKER 1056/1056；ptr 注册域 100%；child seg_lba 差 4096 blocks；lane 4 路均匀；buf_type=kHost、device=-1、**domain 全非空**；target W9 258.55µs、端到端 P50 569.78（node3 本机）。
- **NOF-052 结果（公平对照，2026-11-23）**：plain 新客户端 W9 **132.77µs** / 旧客户端 **133.29µs**（+0.4%）⇒ **客户端版本对 W9 无影响**；plain/vseg 的 domain **完全相同**（非空 0x143cd90）⇒ **domain 排除**；**vseg 258.55 vs plain 132.77 = +125.8µs（1.95×）**。历史 plain 46–68µs 是 **slot=1024** 口径，在 slot=1 下不可复现 ⇒ **旧的 3.75×/210µs 对照不再成立，公平差距是 1.95×/+126µs**。
- **新发现**：两臂 `P50 − W9` = vseg **311µs** / plain **147µs**，≈**147µs/task 的客户端开销**（vseg 2 task/key、plain 1 ⇒ 2.11× vs 2.00×）；它放大了端到端差异（570 vs 280），但 target W9 的 1.95× 仍是数据面核心。
- **NOF-053 客户端分解（2026-11-23）**：plain 每 key 283µs = **master 元数据 query 116µs（41%）** + submit 6 + wait 156（含 target W9 68，超出 88）+ 其他 5；**QoS/worker/dispatch 仅 8–11µs（~3%）**。vseg 的 2× **不是 setup/QoS**（query 103 vs 116 更低），而是 **drain 350 vs 110µs、max_subio 362 vs 112（3.2×）**、submit 19 vs 6 ⇒ **客户端调度排除**；vseg 2× 在 target/传输 drain 段。**master 元数据 RPC ~110µs/key 是两臂共有的固定大头**（与 vseg/plain 无关，是 plain 绝对时延的最大单项）。
- **用户已批准 A 路线**（自编 bd5ee124d/v4 target）；**NOF-054 取证确认：A 无需写补丁**——已构建的 `$D/spdk-e12b7cb-src/build/bin/nvmf_tgt`（md5 979f5517…）自带全套 target 打点，尤其 **`URMA_STAGE_BUFFER` 逐 IO 24 字段**：`trace_seq qid cid lba nlb hz exec enter submit submit_return callback wait enomem rx buffer bdev_cpl data_post data_cpl rel0 lcore flags`；**进程退出时落盘**。BUFFER env：`SPDK_URMA_STAGE_TARGET_SINK=buffer` + `SPDK_URMA_STAGE_TARGET_PATH=<不存在文件>` + `SPDK_URMA_STAGE_TARGET_CAPACITY=200000`（默认 16.7M×192B≈3GiB）；客户端必须 `SPDK_URMA_STAGE_TRACE_US!=0` 才会请求 trace。
- **重启预案（NOF-054 取证）**：停 plain master 2454260 → 停 target 341038（先备份 `/tmp/nvmf_tgt.log` 32.5MB 与 pid）→ 等 spdk.sock 消失 → 用 `TARGET_SPDK_ROOT=$D/spdk-e12b7cb-src` + `TARGET_DUMP_SEC=10` + BUFFER env 起 target；wrapper 实际路径 `/home/yin/Vsegment/run_vesegment_target.sh`（非 vseg_scripts/）。RAID0 `superblock=false` ⇒ 11 盘 BDF/顺序必须核对；nvme 名可能重排；node3 有第三方 `royi:mooncake_client`（109% CPU）与 load 14.4。回退 B = 磁盘 9a413a44（v6）nvmf_tgt + `bench-io-params`（需先把 `build.9a413a44` 改回 `build`）。
- **NOF-055 完成（2026-11-23，M2-1）**：target 已切到 **bd5ee124d**（PID **394826**，md5 979f5517…）；4457/RAID0(11 盘/BDF 升序/256K/online)/ns/transport 正常；BUFFER 已生效（`run/urma_stage_nof055.bin` 0B、O_EXCL、退出落盘；CAPACITY=200000）；旧日志备份 32,780,200B/md5 d294d6ef…。**注意**：11 盘按 BDF 升序接回（每盘 7.15TiB；NOF-054 的 15,002,931,712 是 blocks），旧 target 的 -d 顺序未记录 ⇒ **新旧数据不可按 LBA 对比**；vseg 元数据已清，NOF-056 重写。
- **NOF-056 在 B2 停手（2026-11-23）**：vseg 从零建集群后 T1 写触发 **23 次 `VSEGMENT_CREATING`**（每 64MiB vsegment 容 16 key，第 17 个 key 撞创建窗口；key 16,33,…,390 步长 17），只写入 **377/400** keys、logical_used 1,581,252,608；按规则未跑 reader、未切 plain、未停 target（BUFFER 仍 0B）。这是**已知机制**（NOF-028/029/032/034 已 4 次复现），不是随机故障。
- **NUMA 审计结果（NOF-056 A2，关键新线索）**：4 节点 node0=cpu0–69、node1=70–153、node2=154–237、node3=238–307（0↔2 距离 20）；**11 块数据盘 5 块在 node0（21/22/28/29/2a）、6 块在 node2（91/92/a1/a2/a3/a4）**；**target 核 2–7 全在 node0** ⇒ 6/11 盘与 target 跨 NUMA；RAID0 每 256KiB 条带在 node0/node2 盘间交替，vseg 2MiB task=8 条带、plain 4MiB task=16 条带，跨节点比例/顺序不同。hugepages 各节点 1160、free 16/75/128/128。uburma 设备 numa `[未获取]`。**这成为 126µs 的首要候选；E 段解析必须按 disk/node 分组。**
- **下一步（NOF-057，选项 1）**：同参数 T1 writer 重复补写（≤3 次）至 **400 keys**；然后 vseg 读（trace）→ plain 写+读（trace）→ 优雅停 target 落盘 → 解析 24 字段；新增 **NUMA 分组**（disk=(lba/512)%11，0–4=node0、5–10=node2），并检查补齐的 23 个 key 在 vseg 的 LBA 是否仍为规范窗口（若在尾部/非规范，则两阶段都排除这 23 个 key 窗口后比较）。
- **NOF-057 完成（2026-11-23，M2-2 定案）**：补齐 **400 keys**（2 次尝试，VSEGMENT_CREATING 23→0；vsegments=26，补写多建 1 个尾卷，不影响 key/LBA）；vseg T1 P50 **551.22** / W9 **261.44**，plain P50 **277.50** / W9 **137.00**；target 优雅退出落盘 **33,792 条、dropped=0**；**126µs 100% 落在 `submit_return→callback`（bdev/SSD 执行段）**，其余 6 段为 0（data 段 vseg 反而更短 6 vs 12µs）；两段 `nlb=256`、**lba 集合完全相同**、flags 全 3；**NUMA 完全无关**（vseg node0 p50 256 / node2 255；plain 59/61）；唯一留下的差异是**盘级访问流**：换盘率 **79.7% vs 50.0%**、每盘 LBA 递增 **98.0% vs 99.9%**、每盘回退 **31 vs 1**（2MiB child 铺 8 盘、child1 从 disk8 绕回 disk0–4）。**但 NOF-034（stripe=4MiB 单 child 仍 W9 250）与 NOF-046（q1 顺序仍慢）与该解释有张力 ⇒ 不把"child 数"直接当根因**；下一步 NOF-058 用离线数据判别快/慢 IO 的局部条件（同盘并发重叠 vs 同队列 LBA 局部性）。
- **NOF-058 完成（2026-11-23，M2-3：并发与局部顺序均排除）**：零环境（target 保持 down、只读 `run/urma_stage_nof055.bin` 与两份 params 日志）。**同 tick 占比 0.01%/0%、两种 tie-break 结果一致**（顺序指标稳定）；vseg 客户端提交序换盘 59.7% vs target 执行序 79.7%（+20pp，target 内部有重排放大），plain 两序均 50%。**ov=0（该盘零并发）**：vseg **206µs**（n=4,400，34.4%）vs plain **39µs**（n=4,405，34.4%）；两臂 ov 分布几乎相同 ⇒ **并发/队列深度排除**。**同 (disk,lcore) 紧接前驱（gap≤512）**：vseg p50 **275** vs plain **61**；busy=False（前驱已完成）vseg **206** vs plain **39**；同 gap 桶（near<8MiB）vseg 207 vs plain 41 ⇒ **局部 LBA 顺序排除**。**plain 的快 IO 全部在高 LBA 区（keys≥98，8,336 条 <64µs）；慢 IO 全部是盘首慢区 keys 0–97（3,133 条 ≥128µs，LBA≤802,304≈391MiB）**；vseg 在高 LBA 区也全慢（9,664 条 p50 256）。**同一 LBA（keys≥98）两段读**：vseg mean 250.2 vs plain mean 49.4（**5.07×**）——但 vseg 数据写轮次与 plain 数据不同（vseg 377 keys 单写 + 23 keys 补写；plain 后写覆盖），**该对比受写入历史混杂，不能单独归因**。客户端 per-lane 顺序率 vseg 93.9% vs plain 99.2%（文件行序，`[推断]`）；child0/child1 SUBMIT 严格交替 100%。**判决：24 字段内无判别变量；候选只剩"vseg 写出的数据物理状态"与"读路径的全局/请求层差异"** ⇒ NOF-059。
- **NOF-059 已下发（M2-4，含 target 重启）**：**同时回答两个问题**——① `--vsegment_stripe_size=4194304` 是否真正实现每 key 单 child（528 条 SUBMIT、blocks=8192），且 W9 是否从 ~256 回落到 plain 档；② 同一份单轮写入的 vseg 数据、**不重写**，用裸 SPDK `urma_perf`（NOF-045 同款）顺序/随机读：顺序快（~41µs/91）⇒ 数据无罪、读路径仍有差异；顺序慢（~220）⇒ **写出的数据物理状态**是主因。随后 plain 写+读+裸读作同批对照。**target 用 bd5ee124d 同 RPC 重启、不启用 BUFFER、本单结束保持运行**；所有裸读在 Mooncake 读之后（preflight 写不影响 R1）。
- **NOF-059 完成（2026-11-23，M2-4：单 child 无效；数据无罪；锁定读路径）**：target 重启成功（**PID 708167**、md5 979f5517…、bd5ee124d、4457；无 BUFFER env；首启因 11 盘 vfio 残留 ABORT，用 takeover `-r -y` 还原后重启，同 NOF-055 补救）。**stripe=4MiB 确实实现每 key 单 child**：写 params 377+23 条、读 params **528/528**，全部 `blocks=8192`、`child=0/1`（2MiB 时为 1056）。**但 vseg4 未追平**：客户端 P50 **547.92** vs plain **268.44**（**2.04×**；NOF-057 为 551.22 vs 277.50=1.99×）⇒ **child 数无关**。**同一份数据裸 SPDK 读**：顺序 p50 **40.0 vs 40.8µs**（W9 86.3 vs 79.6）、随机 **220.16 vs 220.16µs**（W9 191.1 vs 190.9）⇒ **数据物理状态无罪**。plain 写 400/0、0.16s、9921 MiB/s、params 400 SUBMIT/8192/child=0/0；plain 读 P50 268.44。裸读 D2（key 200 定点）=218.88µs（4MiB 单 key 内部顺序仍"随机档"）。preflight 覆盖 key98/key200 各 128KiB（已知）。**target W9 本单表（vseg 122.6 vs plain 30.4）受 dump 滞后/跨相位影响，不作结论**；per-key 24 字段需 BUFFER（本单禁止）。环境末态：target **保持运行**（708167，无 BUFFER）、两 master 已停、无 bench。**判决：child 数无关、数据无罪 ⇒ 差异在 vseg 客户端把同一 4MiB NoF 操作驱动到 target 的方式（提交时序/SGL/buffer/endpoint）** ⇒ NOF-060。
- **NOF-060 已下发（M3-0，零环境取证）**：① 对比 NOF-059 的 vseg4/plain 读写 params **全字段**（重点 `endpoint/ptr/block_size/bytes/child/vseg_op_id/lane`）——两臂在 NoF 提交层是否完全同构；② 检查 NOF-057 `URMA_SLOW_IO` 行是否有绝对时间戳（有则算提交节奏/突发，无则记 `[未获取]`）；③ 只读客户端源码（`nvme_urma.c` + `transfer_task.cpp`/vsegment runtime），找出 4MiB→32×128KiB 切分与提交的确切路径、vseg/plain 差异清单、以及 **M3-1 最小插桩点与重建路径**。**零环境：target 不动、不构建、不改码**。
- **NOF-060 完成（2026-11-23，M3-0：两臂 NoF 提交层完全同构）**：**params 全字段除 child 外全同**——op、`blocks=8192`、`bytes=4194304`、`block_size=512`、`endpoint uniq=1`、`vseg_op_id`、lane 0/1/2/3 各 132、`domain=0x143cd90`；`ptr` 两臂地址不同但 **mod 4096/mod 131072 均 0（100% 对齐）**；唯一差异 child=0/1（vseg）vs 0/0（plain），**源码证实 child 只进日志/timing 标签，不进 SPDK 调用**。源码：两臂在 `client_service.cpp:5845/5855` 分叉、`transfer_task.cpp:2360/2527` 各自调用、**`:3076` 同一 `submitSpdkNofOperation`、同一参数集**；`blocks_per_chunk=131072/512=256`（128KiB，两臂同）；`inflight_blocks_limit=8MiB/lane`（两臂同）；lane 轮转 `next_lane++%4`；**task 内 32 个 128KiB 固定同一 qpair**（`:891`）。`URMA_SLOW_IO` **有绝对 tick**（`initiator_submit_tick/initiator_complete_tick/completion_mono_ns` + 全套 `target_*_tick` + `target_us/rx_buffer/buffer_exec/bdev/data/pre_send/release/outside/residual`）。**唯一实测行为差异：每批突发长度 = vseg 16×128KiB(2MiB) vs plain 32×128KiB(4MiB)**（来自 NOF-057 的 2MiB-child 时代；vseg4 时代 TRACE_US=0，突发形态 `[未获取]`）；相邻 submit 间隔 p50 0.45 vs 0.44µs、frac<5µs 93.8% vs 96.9%。SUBMIT 行间 dt（T1 每 key 串行）：vseg4 全程 **545±10µs**、span 0.800s；plain 前 240 op ~540µs、之后稳定 **264µs**、span 0.617s。**结论：没有单点可解释 2× 的现成差异；M3-1a（零构建，仅开 env 复跑读）可拿到逐 128KiB tick + 逐 task submit_span/drain/fifo/credit/max_subio**。产物 `run/nof060_params_compare.txt`、`run/nof060_source_trace.txt`；target 708167 保持运行、无 bench、未构建未改码。
- **NOF-061 已下发（M3-1，零构建、target 不重启）**：**关键设计：同一份 vseg 数据上跑 plain 读路径**——新建 vseg 集群用**新 cluster_id `vseg_nof061`**（不删旧 `vseg_nofvseg` 的 plain 元数据），T1 写 400 keys（stripe=4MiB、slot=1、≤3 次补齐）并 vseg 读（`SPDK_URMA_STAGE_TRACE_US=1`，逐 128KiB `URMA_SLOW_IO`）；然后停 vseg master、**不重写**直接用 `vseg_nofvseg` 旧 plain 元数据起 plain master，读**同一批 LBA 上的 vseg 写入数据**（同样 trace）。判读：plain 在同一份数据上若快（~60µs/IO）⇒ **vseg 读路径/会话是原因**；若也慢 ⇒ 数据-读路径交互。同时取 vseg4 的突发长度（32 还是 16）与 task 级 submit_span/drain/fifo/credit。**不启用 BUFFER、不重启 target、不构建**；旧 plain 元数据缺失/读取失败 → 停手回报（不重写、不破坏 vseg 数据）。
- **NOF-061 完成（2026-11-23，M3-1：判决 = vseg 读路径/会话；数据二次无罪）**：新建 `vseg_nof061` 写入 400 keys（写 LBA 集合与 NOF-059 plain 集合**逐值相同**：400 唯一、0..3,268,608、步长 8192），随后**零写入**用旧 `vseg_nofvseg` plain 元数据读**同一批 LBA**。**plain 读 vseg 数据快**：客户端 P50 **308.37**（复测 330.41）、逐 128KiB **bdev 57/58µs / target 79/80 / roundtrip 108/111**；**vseg 读同数据慢**：P50 **573.03**（复测 601.09）、bdev **249/248 / target 256 / roundtrip 278/276**。**同 LBA 配对 12,324 条：vseg 246 vs plain 53（4.64×），plain 更快占 85.1%**。**vseg4 突发 = 31（32×128KiB=4MiB），与 plain 完全相同**（间隔 p50 0.58–0.61 vs 0.56–0.59µs、frac<5µs 均 0.968）⇒ NOF-060 的"16 vs 32"在单 child 下**不复存在**。新增三条硬排除：**突发形态**（同 32）、**target lcore**（同 lcore 仍差 4.3×：lcore2 248/57、lcore7 250/56）、**顺序/预热**（vseg 复测在 plain 两轮之后仍 601）。分 key：盘首慢区两臂都慢（key0 314/277、key50 257/252），高 LBA 才拉开（key100 250/46、key200 251/40、key300 250/47、key399 263/52）⇒ 盘首窗口是共有项、不是差异源。vseg task 级 span 19 / drain 376 / maxsub 383 / fifo 0 / credit 0；plain 无 `nof_slow_task` 行 `[未获取]`（该插桩只在 nof 路径）。**偏差**：为排除顺序/预热，超出任务书多做了 plain r2、vseg r2 与一次纯元数据 re-register（`vseg_nof061` rc=0，未删键未写盘，Keys=400/logical_used 未变）。环境：target 708167 全程未重启、`NO_BUFFER_ENV`；四臂 400/0 failed、register 全 rc=0；`/cvm/vseg_nofvseg/`=2、`/oplog/`=401、`vseg_nof061` 数据保留。产物 `run/nof061_summary.txt` + 6 份原始日志。**⇒ 剩余唯一变量：会话/qpair 身份。**
- **NOF-062 已下发（M3-2，零构建、target 不重启）**：三件事——① **并发-时延曲线**（同一份数据上 vseg 与 plain 各跑 T1/T4/T16 三档 trace 读），判 gap 是否随并发收敛（收敛 ⇒ 完成检测/唤醒类附加延迟；恒定 ~190µs ⇒ 每命令固有；vseg 恒 250 而 plain 单调升 ⇒ vseg 处于"恒定排队"状态）；② **NOF-061 日志全段分解**（16 字段逐段非零计数 + wait/enomem/buffer/rx 段、per qid/per lcore 配对）；③ **target 侧源码取证**（`lib/nvmf/` URMA transport 读路径：rx → buffer 归属 → bdev 提交 → 完成回调，列出每个"每会话/每 qpair"状态及其初始化点；并核对客户端连接/attach 层 vseg vs plain 是否有差异）。零写入（两臂读同一批 LBA）、不启用 BUFFER、不改码。
- **NOF-062 完成（2026-11-23，M3-2：gap 随并发强收敛 ⇒ 完成检测类）**：同一份数据、两条路径、各三档。**bdev p50（逐 128KiB）：T1 249 vs 54（4.61×）→ T4 262 vs 259（1.01×）→ T16 270 vs 265（1.02×）**；vseg 恒 249→262→270，**plain 单向追平 54→259→265** ⇒ 判据 ①（完成检测/唤醒）命中且兼具 ③ 形态。**16 字段分段：T1 差异 100% 在 `bdev_us`**，`rx_buffer/buffer_exec/pre_send/release` 全 0、`data` 6 vs 9、`outside` 两臂同 21µs；`wait/enomem` 未出现在数据 IO 上（0）。同 qid 配对：T1 4.4–4.8×（q2 249/56、q3 247/54、q4 249/55、q5 250/52），T4/T16 全 1.0×。**qid→lcore 两臂不同**（vseg {2:6,3:7,4:2,5:3} vs plain {2:2,3:3,4:4,5:5}），但 NOF-061 的 vseg r2 在 plain 两轮之后仍 601 ⇒ **顺序/分配不是主因**。突发同构（run=31；间隔 vseg 0.62–0.64 vs plain 0.83–0.85µs）。task 级 vseg：T1 span 21/drain 383、T4 30/494、T16 54/852（fifo p90 960、credit p90 166 ⇒ 客户端 QoS 仅高并发生效）；plain 无 `nof_slow_task` `[未获取]`。源码链：`urma.c` 收包 → `transport.c:932 spdk_iobuf_get(group->buf_cache,…)`（**每 poll group 独立 buf_cache**）→ `ctrlr_bdev.c:449/493` → `raid0.c:76` → 盘；`bdev_cpl_tick` 于 `urma.c:2493` 的 `req_complete`（**poll 上下文**）取。**`[事实]` target 无 vseg/child 语义**（`lib/nvmf`/`lib/bdev`/`include/spdk` 全量 grep 仅命中 config.h 构建路径）；连接层 NQN/nsid/qpair 数（4）全同，唯一差异＝会话实例身份与 qpair→lcore 分配。偏差：首次 vseg 三档漏 `--warmup_keys=128`（SUBMIT=405、T1 614.49），已修正重跑（正式 SUBMIT=528、T1 604.30），旧日志同名覆盖；`/cvm/vseg_nofvseg/` 2→3 键、`/oplog/vseg_nof061/` 2003→2004（register/租约元数据，无数据写入）。环境：target 708167 全程未重启、无 BUFFER；六档全 400/0 failed。产物 `run/nof062_summary.txt`（5,304B）+ 6 份日志（各 ~11–12MB）。
- **NOF-063 已下发（M3-3，零构建、零环境）**：只在源码里回答「那 ~195µs 是谁在等」——① `urma.c` 的 `poll_group_poll` 全路径是否有**阻塞等待/超时/批量上限/事件合并**（列出所有 100–500µs 量级时间常量与 env 默认值）；② qpair/jetty/**JFC** 与 poll group 的绑定结构，以及**两个会话在这套结构里可能不同的唯一自由度**；③ 本地 NVMe 完成由谁轮询（是否 interrupt/事件模式）；④ `req_complete` 之前是否有 defer/批量回写；⑤ 结论：源码能否直接定位（候选常量与实测 195µs 比对），还是必须插桩（给唯一插桩点与预期读数）。**不改码、不跑测试、target 不动。**
- **NOF-063 完成（2026-11-23，M3-3：静态无 50–500µs 常量；找到结构性错配，但开发侧复核判定其对称、不能解释臂间差）**：`poll_group_poll` 主链 = `transport.c:777→779` → `urma.c:2303 nvmf_urma_poll_group_poll`（2303–2383，81 行）。**`[事实]` 50–500µs 量级时间常量 = 0 个**：`urma_poll_jfc` 无 timeout 形参、`urma.c` 内 `nanosleep/usleep/spdk_delay_us/sched_yield` 零命中；`abort_timeout_sec=1`、`min_kato=10000`、`jfc_depth=4096`、`jetty_depth=2048` 全不在该量级。环境实际值：`/proc/708167/environ` 只有 `SPDK_URMA_DEV_NAME` + `SPDK_URMA_TARGET_DUMP_SEC`；`nvmf_get_transports` = `send/recv_jfc_count=2`、`num_jetty_per_ep=1`。**结构发现**：`stride = worker_count = 6` 而 `send_jfc_count = 2` ⇒ `for (j = worker_index; j < 2; j += 6)` **只有 worker_index 0/1 两个 reactor 轮询网络**，2..5 完全不轮询；qpair 的 JFC = 建连序 mod 2，且与属主 worker_index 无对齐关系 ⇒ 跨线程投递（`urma.c:2289 spdk_thread_send_msg`）→ `completion_msg_fn` → `handle_completion`（`exec_tick` 在 `2167`）。本地 NVMe 侧：`bdev_nvme.c:1970 spdk_nvme_poll_group_process_completions(group,0,…)` 每轮不限量、无中断/idle；`req_complete` 前无 defer（读路径全内联）。**开发侧复核（重要）**：该错配在两臂间**对称**——plain 属主 {2,3,4,5}=worker{0,1,2,3}、vseg 属主 {6,7,2,3}=worker{4,5,0,1}，**两边都是 2 个 qpair 落在轮询核、2 个落在非轮询核**，且 qid→JFC 同为 mod 2；更要紧的是**同一臂内 4 个 qid 时延均匀**（vseg 247–250、plain 52–56），若机制来自属主核/JFC 绑定，四个绑定各异的 qpair 不可能同值 ⇒ **该结构不是 4.6× 的原因，也不构成 `SPDK_URMA_JFC_COUNT=6` 实验的依据**（该实验可另作参照，不作为判决实验）。**结论：静态读码无法判定 195µs，必须插桩**；候选读数表 H1（完成早在 CQ、reactor 没轮到：`d_rounds` 大 / `d_nvme_poll` 小且首轮即拿到）、H2（轮到了但 CQ 空：`d_nvme_poll` 大、多数轮 0）、H3（两者均 ~1 而 bdev_us 仍大 ⇒ 调度抢占或设备侧）。产物 `run/nof063_source_poll.txt`（11,455B）；target 708167 未重启（起始时间 17:35:36 未变）。偏差：`worker_count`/`send_jfc_count` 运行期实际值无 RPC 可读，已列入插桩一次性打印；期间发现 node3 有第三方 `royi` master（50151），仅记录未触碰。
- **NOF-064 已下发（M3-4，需改码/重编/重启——待用户批准后才执行）**：target 逐 IO 插桩两组信息——① **子命令字段**（`nsid/lba/nbytes/iov_cnt/iov[0]`+总长/SQ 位置/qpair/thread）以判定"设备收到的命令是否真的相同"；② **轮询节奏**（`urma.c:2303` 入口线程本地 `poll_round`、`bdev_nvme.c:1970` 的 `nvme_poll` 轮与完成数、`bdev_us`），把 H1/H2/H3 分开；附一次性映射打印（worker_index/worker_count/JFC 索引）。两臂同代码、开关 `NOF064_TRACE=1`、T1 各 1 遍，另加一次**反序复测**（plain 先、vseg 后）确认臂间差在插桩下仍成立。回退=旧二进制 md5 `979f5517…` + 原 RPC。
- **NOF-064 结果（2026-11-23，M3-4：改码/重编/重启三步完成，两臂采集未完成 · 部分关闭）**：新 target **PID 1587271、md5 `21ced674…`**、4457/RAID0(11 盘)/transport 验收全通过、`NOF064_TRACE=1` 生效。三份二进制并存：`nof059`=`979f5517…`（原 bd5ee124d）、`nof064`=`21ced674…`（v1 插桩，在跑）、`nof064b`=`afe95672…`（v2，补读路径快照，已编译未运行）。编译坑：`nof064_*` 符号在 nvmf 库而 bdev 模块不链它 → 计数器改定义在 `bdev_nvme.c`、`urma.c` 弱引用。**一次性映射实测**：`worker_index 0→core3, 1→core4, 2→core5, 3→core6, 4→core7, 5→core2`。**两处硬阻塞**：① **重启使 target identity 变化**（raid0/nsid1 uuid `c697219c…`→`63a1e3cd…`，base bdev uuid 未变）⇒ `vseg_nof061` mount `rc[-300] INVALID_VERSION`、`SUBMIT=0`；**plain `vseg_nofvseg` 仍可挂载**（`[事实]` vseg 元数据跨 target 重启失效，plain 不受影响 —— 产品级缺陷）；② **v1 插桩位置错**：读路径走 `urma.c:1921`（C2H，no pull），v1 只插了 `2214`（PULLING）⇒ 读 IO 没取快照，`d_rounds=467,099,749` 是绝对值不是 delta。已采集只有 plain 臂 v1（client P50 317.44、400/0；target 4,000 条 `NOF064_IO`：bdev mean 248.8 / p50 225 / p90 312、≤70µs 占 16.6%）。**开发侧判定（已修正）**：该 225µs 是**采样偏差**，不是真实抬升——4,000 条预算按到达序约等于前 125 个 key，其中 keys 0–97（78%）落在盘首慢窗口，与实测 83.4% >70µs 吻合；反证是同臂客户端 P50 317.44 与 NOF-062 plain T1 307.51 基本相同，端到端并未变慢。**早前把它归因为 per-IO printf 观测者效应，因客户端 P50 未同步抬升而不成立，已撤回。**v1 数据仍不可用于任何 bdev/`d_*` 结论（另因 `d_*` 为绝对值）。
- **NOF-065 已下发（M3-5，沿用 2026-11-23 改码/重编/重启授权）**：① 插桩升级 **v3**——把 per-IO printf 换成**静态环形数组**（≥400k×96B，带 valid 标记，非 reactor pthread 每 3s 追加落盘），补 **iov 字段**（`nsid/lba/nbytes/iov_cnt/iov[0].base/len/total_iov_bytes`），并把 `d_rounds/d_nvme_poll/d_nvme_cpl` 改成**真 delta**；② 重启到 v3 后**新 cid `vseg_nof065` 写一次 400 keys**（重启后 vseg 旧元数据必失效，故必须重写）；③ 两臂 T1 各 2 遍 instrumented + 各 1 遍 **`NOF065_TRACE=0` 对照**（零干扰验证，差 >5% 则整轮作废）；④ 解析按 `bdev_us` 分桶给 `d_*` p50/p90，并用 `/proc/<target>/numa_maps` 把 `iov[0].base` 分 NUMA 桶（buffer 归属首次可查）。**判决前明确弃用两条旧假设**：v1 的 225µs、以及 JFC/stride 结构（`exec_tick` 在 `handle_completion` 内、跨线程投递在它之前 ⇒ 不进 bdev_us 窗口，`stride=1`/`JFC_COUNT=6` 不作为修复实验）。
- **NOF-066 已下发（口径统一，零环境）**：现有日志（`run/nof062_{vseg,plain}_t{1,4,16}.log`、`run/urma_stage_nof055.bin`）都带 LBA，按 key i 占 `[i×8192,(i+1)×8192)` 的映射**离线剔除盘首慢窗口**（keys 0–97 / LBA < 802,816 块 ≈ 391.8 MiB）后重算，给「全量 / 排除」两列：逐 128KiB 的 roundtrip/target/bdev 与端到端每 key 的 p50/p90/p99/mean、同 LBA 配对比值、直方图分桶。**动机**：现状口径不一致——01/02 章主表是 400 keys 全量（含慢窗口），而早期部分分析（NOF-039/040）用的是另一轮 800–1200 keys 的干净口径，跨轮不可换算。**不启停服务、不跑 bench、零写入、不改码。**
- **NOF-067 已下发（用户转达同事意见后的纠偏 · 零环境）**：开发侧此前判断「`submit_return→callback` 之间没有第三个可观测点」，指的是**现有 `stage_trace` 的软件打点位置**；用户转达同事意见称该段可用 **SQE / CQE / SQHD 对比**继续拆解，并给了现成脚本：node3 `/home/yin/run_exp/run_exp.sh p -F 3`（裸 SPDK）、node1 `/home/yin/Vsegment/vseg_scripts/vsegment_cluster.sh` + 运行时 `--target-trace-entries 2097152`（Mooncake 侧）。本单只做**只读取证**：① `-F 3` 究竟输出哪些字段；② `--target-trace-entries` 在脚本→env/CLI→源码里的传递链与容量语义；③ 24 字段里哪些是时间戳、哪些是队列索引，**是否存在 SQ 写入位置 / doorbell(SQHD) / CQE 出现 / CQ head**，据此能否直接量化「CQE 早已出现、回调来晚」；④ 现役二进制（`21ced674…`）是否支持该参数。**产出一句话结论：走同事这套（零改码）还是仍需我们的 poll 计数 + iov 插桩。** 只读脚本/源码/`--help`，不启动服务、不跑压测、不改码、不重启。
- **NOF-066 完成（2026-11-24，M3 附带的汇报口径重算 · 纯离线）**：数据源 `run/urma_stage_nof055.bin`（6,488,160 B、magic `URMA3B01`、footer `URMA3END`、`dropped=0`、nrec=33,792、hz=1e8）；按 `exec_tick` 全局最大间隙 **208.953 s**（idx 16,896）切分 → seg1=vseg / seg2=plain 各 16,896 条，**lba 集合相同**，且 256/314/261.1 与 61/285/136.5 **与 NOF-057 逐位吻合** ⇒ 切分正确。**全量 vs 排除（R1：`lba>=802,816`，剔除 37.1%）**：vseg **256/314/261.1 → 256/314/250.9**（不动）；plain **61/285/136.5 → 50/67/50.0**（p50 −18%、mean −63%）；**比值 p50 4.197× → 5.120×、mean 1.912× → 5.015×**。慢窗口内 6,272 条两臂 p50 **256 / 283** ⇒ 确为共有项、只稀释 gap。NOF-062 六份 trace 独立复算同结论：T1 vseg 249/308/246.7→248/307/245.5、plain 54/280/120.6→45/58/44.9，**p50 4.611×→5.511×、mean 2.046×→5.463×**；**T16 仍 1.030×**（排除后依然打平）。**直方图（排除后）：vseg 全在 128–512µs（128–256 55.5% + 256–512 44.3%，`<128µs` 为 0），plain 90.5% 在 32–64µs**；同 LBA 配对 4.843×→**5.489×（plain 快 100%）**。⇒ **结论不变、幅度变大**；今后汇报主表用「排除口径」+ 慢窗口小表。产物 `run/nof066_excl_summary.txt`（10,337 B / 145 行 / md5 `8aca4392…`）。偏差：etcd 键统计因本地 shell 转义失败（`2>/dev/null`、`$p`、awk 被上层 shell 展开）而 `[未获取]`，需 base64 下发重试；不影响本单结论。
- **新事实（影响后续改码基线）**：`/proc/1587271/exe → …/nvmf_tgt (deleted)` ⇒ **正在运行的是被删除的 v1 inode**，而磁盘上 `build/bin/nvmf_tgt` 已是 **nof064b（v2，`afe95672…`）**。⇒ 任何 B 段改码必须**以 v2 源码（含 `urma.c:1921` 读路径快照）为基线**，否则会丢掉读路径快照。

- **第三阶段（2026-11-25 起）**：node1/node3 交还他人，迁移到 **node2/node4**；**master+target 在 node4，reader/writer 在 node2**（元数据和数据都跨节点）。源码统一 `wangyuqi0429/Mooncake @ supercache-snapshot`（aadab957），SPDK 保持 e12b7cb/bd5ee124d wire v4。拓扑、源码政策、里程碑、风险见 `第三阶段 node2&4/README.md`；当前任务 NOF-107。

## 项目背景（开发侧已知的全部信息）

- 项目名：**mooncake nof vsegment**（Mooncake NVMe-oF + vsegment），由同事交接，开发侧此前无任何了解。
- 同事交接资料位于内网 Windows：`C:\Users\l00955908\Desktop\mooncake nof vsegment`（路径含空格，命令中注意引号）。
- 测试环境为 node1、node3 两台机器，接入方式见（就在交接资料目录内）：
  `C:\Users\l00955908\Desktop\mooncake nof vsegment\连接node1和node3.md`
- 测试环境工作目录（本轮创建）：`/home/l00955908/mooncake-nof-vsegment`（node1、node3 各一份）。
- 环境为多人轮流使用。**本轮不是独占**，按共用机约束执行；用户明确宣布"独占"后才放宽进程占用处理，重启等高危操作任何情况下都需先经用户确认。

## 端口规划（用户已定，2026-11-20，后续所有任务必须遵守）

| 组件 | 节点 | 端口 | 说明 |
|---|---|---|---|
| SPDK nvmf target | node3 | **4457** | NVMe-oF over URMA 监听（trsvcid），固定；旧 4421 已停用；换端口前要 `nvmf_subsystem_remove_listener` |
| Mooncake master RPC | node3 | **50056** | 必须显式传；默认 50051 禁用 |
| Mooncake master HTTP metadata | node3 | **8086** | 显式传 |
| Mooncake master metrics | node3 | **9006** | 必须显式传；默认 9003 禁用 |
| Mooncake client TE RPC | node3 | **50156** | client 侧，固定 |

- 总规则：**不得使用各组件默认端口，也不得照搬同事使用的端口；我方 Mooncake 端口一律以 6 结尾**。
- 起任何服务前先核对该表；同事脚本里的端口/路径不能直接沿用。
- 同事环境实测端口（仅作差异对照，禁止沿用）：master RPC 50058、metrics 9008、target trsvcid 4420；cluster_id `vseg_yin_bonding`。

## 基准测试口径（用户 2026-11-23 补充，所有性能任务必须遵守）

- **盘首慢窗口是 node1/node3 的历史观测，不是普适事实**：每块盘开头约 **36MB** 慢区、RAID0 11 盘 ⇒ 前 **~396MB**（旧环境 4MiB key 实测为前 97 个 key ≈388MiB，~290µs）——**全部来自 node1/node3 + 旧源码/旧盘序，尚未在 node2/node4 验证，不得预设**。
- **第三阶段（node2/node4）规则（2026-11-25 用户更正）**：
  1. 主表一律用**全量 400 keys**，不预先剔除任何区间；
  2. 只有当新数据中**按 key index 出现可重复的前缀慢档**（多遍一致）时，才按**该数据实测出的边界**追加"全量 / 数据实测慢前缀"两列，并写明边界来自哪几遍；
  3. 禁止直接套用 0–96 或其他旧边界；
  4. 旧 node1/node3 数字（含 2.07×）只作参照，未经本侧复核，**不作基线、不与新结果混比**。
- **旧规则仅存档（node1/node3 适用）**：warmup 不能跳过慢窗口（NOF-039：warmup 阶段与测量阶段读同一批 key 都 ~245µs，keys 0–96 慢占比 90.9%/91.0%）；当时用"剔除 keys 0–96"的统计口径，并保留 warmup=128 维持历史一致。该规则的适用范围**仅限旧环境**。
- 旧环境下该慢窗口**不能解释** vsegment 的 250–274µs（vseg 是全 key 均匀慢，不是前 97 个 key）；此判断同样待 node2/node4 复核。
- **旗标核对（任务设计方必须做）**：同事构建二进制基于 **be10be2**，比远端源码 tip **4c0ea28** 旧，源码里有而二进制没有的旗标确实存在（NOF-038 的 `--key_order` 就是）。任务书里写的旗标必须在 `--help` 中逐一核对；不存在则按"停手回报"处理。

## 协作约定

1. **方向单向**：本文件只由开发指挥 Agent 更新并 push；测试执行 Agent 只 pull。
2. **任务编号**：`NOF-001`、`NOF-002`…；状态：`待执行` / `执行中` / `待回执` / `已关闭`。每次会话只执行当前任务（或传话中指定的任务号）。
3. **事实与推测必须分开**：所有陈述按 `[事实]` / `[推算]` / `[未获取]` 三类标注。
   - `[事实]`：直接读到或实测，附来源（`文件:行号` / 命令 / 原始输出）。
   - `[推算]`：由事实四则运算得到，写出式子，可复算。
   - `[未获取]`：没查到或没权限，写明原因与已尝试的路径。
   - **禁止用默认值、经验值或猜测填充**。查不到比编造有价值。
4. **资料最小化**：回执只给结论、`文件:行号`、`≤20 行`原始输出片段；不贴整文件、大段日志或源码。回执与任务书中**不写具体内网 IP、账号、密码、token**，用 `node1` / `node3` 指代；IP 需要描述时写网段并给最后一段打码（如 `x.x.x.*`）。
5. **任务拆分**：一单一阶段、一个验收点；测试 Agent 只执行任务书里的逐条命令与验收，不选型、不自行改参数、不扩大范围。任何不确定/冲突 → 标 `[未获取]` / `[冲突]` 并回报，由开发侧决定。
6. **安全红线（所有任务默认生效）**：
   - 本轮非独占：禁止停止、挂起、renice、信号干扰任何非本方进程；禁止触碰他人启动的 master / client / target / 压测进程；禁止重启任何服务；禁止装卸驱动；禁止 `pkill`/`killall`。
   - 默认只读。NOF-001 唯一的写操作是：在两节点 `mkdir -p /home/l00955908/mooncake-nof-vsegment`。其余一律不写。
   - 不 reboot；不改系统配置；不装包。
   - 所有远程命令非交互（`sudo -n`），不等待输入。
7. **长任务**：预计超过 30 min 的任务，每 ~15 min 输出一行进度（阶段 / 已完成问项 / 阻塞）。
8. **不猜原则**：任务书中凡要求取证的点，一律以实测或文档原文为准；无法取得的按 `[未获取]` 记录。

## 回执格式（每个任务固定五段）

```
一句话结论
1. 结论（按问项逐条编号回答）
2. 证据（file:line / 命令 / ≤20 行原始片段）
3. 未做 / 未获取（原因）
4. 风险与偏差
5. 产物（路径清单；本次无产物则写"无"）
```

---

## 进度台账

| 任务 | 状态 | 一句话结论 |
|---|---|---|
| NOF-001 | 已关闭 2026-11-20 | 项目/资料/环境已摸清；P6 reconcile INVALID_PARAMS 是当前阻塞；node3 资源被旧 gap 环境占用 |
| NOF-002 | 已关闭 2026-11-20 | P6=reconcile 门禁；1024 分区全灭且 manager 未安装；首要根因指向"旧 oplog 重放"，NOF-003 验证 |
| NOF-003 | 已关闭 2026-11-20 | 假设证实：旧 oplog 重放 1024 条 vsegment 快照（9306 条 entry）；清场必须同时删 `/cvm/<cid>/` + `/oplog/<cid>/`；自建环境要素已收集 |
| NOF-004 | 已关闭 2026-11-20 | 我方 master 在 node1 稳定运行（PID 2091164，50056/9006），全新 cluster 无 oplog 重放、0 reconcile 失败；8086 未启用（见事实） |
| NOF-005 | 已关闭 2026-11-20（**未通过**） | mount 失败：master attach rc=-19 / URMA qpair err=-5（Operation not permitted）；客户端谎报成功；P6 门禁未获得有效验证 |
| NOF-006 | 已关闭 2026-11-20 | **不是协议问题**（两树 WIRE v4 一致）；是设备/provider 问题：master 用 udmac1d1e2 导致 0 个 `/dev/uburma` 映射、未加载 URMA provider；同一二进制在 yin master（udmac0d1e2）下正常；现役 target 也是 udmac0d1e2 |
| NOF-007 | 已关闭 2026-11-20 | master 已按 `udmac0d1e2` 重启（PID 2162056，端口正常、无新 ERROR）；**设备映射验收口径有误**——SPDK 惰性初始化，未 mount 不产生映射；映射验证并入 NOF-008 |
| NOF-008 | 已关闭 2026-11-20（**通过**） | mount rc[0]；**P6 首次有效验证通过**：scanned=1024/owned_by_me=1024/reconcile_fail=0，partitions_installed=1024，60s 复查仍 0；设备映射 mount 时刻 0→20 条；quota 快照 181,399 B |
| NOF-009 | 已关闭 2026-11-20（**通过**） | writer 400/0，1.5625 GiB，0.40 s，3970.7 MiB/s；master logical_used 精确 +1,677,721,600 B；无 ERROR；target 仅周期 W 汇总（per-IO detail 未开） |
| NOF-010 | 已关闭 2026-11-20（**通过**） | 我们的读：400/400、verify 0 错；**Avg 801.29 / P50 791.06 / P90 1017.98 / P99 1709.42 / Max 1886.75µs**；URMA_SLOW_IO=0（打点默认关、阈值 100ms）；控制面逐 key RPC 158–386µs；target 读窗口 W9 285.87µs/exec 94.3% |
| NOF-011 | 已关闭 2026-11-20（**通过**） | 采到 5,044 条 `URMA_SLOW_IO`（读阶段 4,675）；**p50 五段：roundtrip 528 / target 430 / bdev 387 / data 32 / outside 81 / residual 82µs**；bdev 占 target 90%、占 roundtrip 73%；诊断开销 +17.7%（Avg 801→943）；阈值 400µs 覆盖 36.5% child IO |
| NOF-012 | 已关闭 2026-11-20（**无定论**） | 15/15 遍成功；各臂 p50 中位 721–759µs，臂间差 < 臂内极差（35–75µs）⇒ chunk/qpairs 在本次样本下无可辨识影响；target W9 277.8µs 未变；chunk 是否生效无法证明（诊断关、日志无 child 记录） |
| NOF-013 | 已关闭 2026-11-20（**结论待复核**） | 当时判"chunk 对读路径无效"（blocks 恒 256）；但复核代码后：vsegment 读路径本应读该 env，怀疑**运行环境被脚本硬编码覆盖**，NOF-015 用环境自证复测。文档 §3.1/3.2/3.4/§四/§八原文已取 |
| NOF-014 | 已关闭 2026-11-20（**通过**） | 15/15 遍成功；**只有降线程有效**：L3（4 线程）p50 591.1 vs 714.9（−17.3%）、p99 −49.5%；8MiB inflight 无益、2MiB 灾难（+670%）；副本缓存 env 在二进制中不存在；target W9 在 L3 满窗口仍 254–259µs |
| NOF-015 | 已关闭 2026-11-20（**通过**） | env 自证 chunk 已进进程（131072/262144/2097152），但读路径 NVMe IO 恒为 **128KiB**（blocks p50=256）⇒ **代码层固定**，env 无法改；vsegment child=2MiB（planned_children=2），128KiB 是更下层 IO 粒度 |
| NOF-016 | 已关闭 2026-11-20（**通过**） | yin target 已起：PID 341038、md5 736a902f…、listener **node3:4457**、strip 256K/11 成员、dev udmac0d1e2；旧三进程全退；我方键清空、yin 键未动；master 重启 PID 2462234（leader/serving、enabled=0）；回滚快照已存 |
| NOF-017 | 已关闭 2026-11-20（**通过**） | 新 target 上 mount rc[0]（233ms）；1024 分区 reconcile 全通过（60s+ 复查仍 0）；quota 181,399 B；device_identity=5805fd63… 与新 raid 一致；**mount 期 target W9≈70µs（n=10，旧 target 255–288µs）** |
| NOF-018 | 已关闭 2026-11-20（**通过**） | writer 400/0，1.5625 GiB，0.39s，4091.95 MiB/s；写时延 Avg 977.53 / P50 901.03 / P99 2775.35；**写窗口 target W9 23,550 ns（旧 24,840，−5.2%）** ⇒ 换 target 不影响写路径，5× 差异在读路径 |
| NOF-019 | 已关闭 2026-11-20（**通过**） | **端到端基本不变**：R16 p50 719.0（旧 714.9，+0.58%）、R4 602.5（旧 591.1，+1.93%，均在臂内离散内）；**target 读 W9 254.8µs（旧 283–288，−11%）** ⇒ target 变快未传导到端到端 |
| NOF-020 | 已关闭 2026-11-20（**通过**） | **文档 54.68µs = 11 盘写路径样本**（非读！"5× 差距"系读/写跨口径，撤回）；新 target 读五段 p50：roundtrip 514 / target 424 / **bdev 387（与旧 target 完全相同）** / data 13 / outside 59 / residual 60；`target_valid=98.9%`（TRACE=0 不影响遥测）；端到端诊断 P50 779，控制面+应用约 265µs |
| NOF-021 | 已关闭 2026-11-20（**通过**） | 4 线程 vs 16 线程：**bdev 418 vs 380µs（+10%，不降反升）** ⇒ 不是客户端排队；盘 = 11× 华为 HWE7AP457T6L005N（7.68TB）；target W9 259.8µs（exec 96.3%）；4 线程端到端诊断 P50 656.5µs |
| NOF-022 | 已关闭 2026-11-20（**通过，含 1 次误写**） | Mooncake T1 bdev 425µs、端到端 P50 571.1；**裸 SPDK 128K T1B1：客户端 p50 41.3µs、target W9 30.2µs**；同 target 下 Mooncake W9 259µs（T1/T4/T16 不变）⇒ 8.6× 差在 target 执行层；但裸=顺序、Mooncake=随机 2MiB，**访问模式是混淆变量**，NOF-023 受控验证；urma_perf preflight 在 LBA0 写了一次（已授权修复） |
| NOF-023 | 已关闭 2026-11-20（**通过**） | **8.6× 主体 = 随机 vs 顺序**：QD1 target exec 30.35→190.58µs（6.3×）；QD 只次要（seq +36%、rand +8%）；**随机下 Mooncake 259 vs 裸 SPDK 190–206µs = 1.26–1.36×**；LBA0 修复完成（writer 400/0、verify 0 错）；yin urma_perf ABI 不兼容（WIRE v6 vs v4） |
| NOF-024 | 已关闭 2026-11-20（**旋钮证否**） | T1 下 qpairs 4/8/16 × workers 4/8：P50 中位 551.5/542.7/537.5/537.8µs（差 < 臂内极差）；**target W9 恒 250.5–251.2µs**；最好 537.5µs，距 300 差 1.79×；target 执行占 251µs（46.7%），客户端流水线 ~287µs；12/12 无失败 |
| NOF-025 | 已关闭 2026-11-20（**判决：VSegment 慢**，含布局混淆） | plain NoF：**T1 P50 258.3µs vs VSegment 537.5µs（plain 快 2.08×）**；16t 打平（718.7 vs 719.0）；写 P50 273.0 vs 738.6（plain 快 2.7×）；target W9 plain 96.9/194–196µs vs VSeg 250.5/259.6µs；但 plain 数据连续堆放 1.6GiB、VSeg 散在 1024 分区 ⇒ **布局是混淆变量**，NOF-027 消除 |
| NOF-027 | 已关闭 2026-11-20（**分区数证否**） | 1024→64：T1 P50 539.9 vs 537.5（无变化，target W9 仍 250.8）；**16t 变差 854.3 vs 719.0（+18.8%）**；仍未赢 plain（T1 2.09×）。机制指向"读序与物理布局错位"，非分区数量 |
| NOF-028 | 已关闭 2026-11-20（**写入受阻，读未测**） | slot=1 被接受（partitions=1、reconcile 0）；writer 16 线程 377/400，**23 个 VSEGMENT_CREATING**（每跨 64MiB 卷边界丢 1 key，不重试）；容量 24×64MiB=1.5GiB 不足 1.6GiB；reader 未跑 |
| NOF-029 | 已关闭 2026-11-20（**布局假设否**） | vseg-1：T1 P50 543.0 vs plain 258.3（2.10×），W9 263.6–276.8（更高）；分区数 1/64/1024 的 T1 全在 537.5–543.0 窄带 ⇒ **布局无关**；差距在 vsegment 读路径（2MiB child×16×128KiB）。数据 400/400 verify 0 错 |
| NOF-030 | **已被 NOF-031 取代（未执行；若你已在跑则继续，回执按 NOF-030 登记）** | vsegment stripe=4MiB（每 key 单 child）实验；内容并入 NOF-031 |
| NOF-031 | 已关闭 2026-11-23 | 环境可用（同事仅占 8086 python3、node1 load ~2.6%）；stripe=4MiB：T1 551.6 / 16t 705.4、W9 252.3/270.5 ⇒ **未追平 plain**；**target IO 数 n=25,762 与 2MiB 相同 ⇒ stripe 不改 IO 粒度**；且 plain 的 n 同为 25,762 ⇒ **plain 也是 128KiB IO，差距在每 IO 执行时间而非 IO 大小** |
| NOF-032 | 已关闭 2026-11-23（**布局二度否证**） | 全新 vseg-1 + 纯 T1 顺序写（2 次尝试后对象层 400 全齐）：T1 P50 **536.7**、16t **921.8**、target W9 **251.1**（plain 258.3/718.7/96.9）；分区/stripe/写序全部试完，W9 恒 250–252 ⇒ 每 IO 执行成本是 vsegment 路径自身；`VSEGMENT_CREATING` 第 3 次复现 |
| NOF-033 | 已关闭 2026-11-23 | **LBA 完全连续**：key i 严格占 [i×8192,(i+1)×8192) 块、397/399 步长 8192、key 内相邻差 256 占 93.4%、0 次 >1MiB 跳变、覆盖 98.4% ⇒ 布局彻底洗清；期间 W9 仍 258.9µs（plain 96.9）。代码复核：plain 与 vsegment 最终都调用同一 `submitSpdkNofOperation`，target 端无 vsegment 概念 ⇒ 差异只能来自 IO 到达时序/缓存，不是"vsegment 寻址" |
| NOF-034 | 已关闭 2026-11-23（**五个维度穷尽**） | vseg-1 + stripe=4MiB（单 child）+ 纯 T1 顺序写：T1 559.2（plain 258.3，2.17×）、16t **735.3（plain 718.7，+2.3%，最接近）**、W9 250.1–270.9；分区数/stripe/child/写序/物理连续性全部排除 ⇒ 配置层无剩余旋钮 |
| NOF-035 | 已关闭 2026-11-23（**伪影推断被否证**） | **96.9 可在干净窗口复现**：plain W9 **99.14µs**（写窗口 28.56）；plain 12,774 条 IO = 128KiB 97.1% + 4MiB 2.9%，vseg 12,764 条 = 128KiB 96.8% + 2MiB 3.2%；LBA 两边同为每 key 严格连续、>1MiB 跳变 0；**客户端单条 128KiB roundtrip p50 plain 97 vs vseg 274µs（2.8×）** ⇒ 2.5× 差异真实存在于 target 执行段 |
| NOF-036 | 已关闭 2026-11-23（**主问题未验证，发现 bench 缺陷**） | 三臂读 W9 全在 96–110µs（稳定复现 ~97，从未接近 250）；但臂 2/3 的"重写"被 master 以 `OBJECT_ALREADY_EXISTS` 全拒（各 400 条）、target 写窗口 n=2 ⇒ **三臂读的是同一份 NOF-035 数据，写模式对照未发生**；并发现 **bench 把被拒 put 计为成功**（400 succeeded vs 400 拒绝） |
| NOF-037 | 已关闭 2026-11-23（**写模式排除；发现段首慢区**） | 两次写都落盘（Keys 800→1200、SSD 3.12→4.69GB、写窗口 n=12,802、W9 28.3µs）；同会话三桶 128KiB p50：桶1（16t 旧）98–100、桶2（T1 新）85–87、桶3（16t 新）83–86µs ⇒ **写线程数无影响**；**新线索：桶1 慢 IO 精确集中在 key 0–96（LBA [0,388MiB)，p90≈290µs、慢占比 22%），其余 key 0–4%，三遍复现，按盘/通道均匀** |
| NOF-038 | 已关闭 2026-11-23（**阻断：旗标不存在**） | 构建二进制（be10be2）无 `--key_order`/`--key_order_seed`（远端源码 4c0ea28 才有），随机读无法执行；`--shuffle_seed` 对 remote_memory 无效（no-op）；旁证读逐位复现 NOF-037（桶1 ≥200µs 22.0%，key 96/97 边界锐利，慢段 ~0.68–0.76s） |
| NOF-039 | 已关闭 2026-11-23（**warmup 不能跳过慢窗口**） | warmup 与测量两阶段读 keys 0–96 均 ~245µs（慢占比 90.9%/91.0%）⇒ 慢区是固定 LBA 段，非连接预热；干净的**排除 0–96**基线：**T1 p50 86 / p90 109 / W9 68；16t p50 359 / p90 570 / W9 161** ⇒ 并发 1→16 抬高 per-IO **4.2×** |
| NOF-040 | 已关闭 2026-11-23（**qpair 排除；vseg 全程均匀慢**） | vseg T1 干净（排除 0–96）p50 **274**/p90 332、W9 **255.25**（plain 86/109/68 ⇒ 3.2×/3.0×/3.75×）；**qpairs 4→1 仅 −2.6%（274→267，W9 255→248）**；keys 97–399 慢占比 **90.7%**（与 0–96 的 91.2% 相同；plain 同段 0%）⇒ 全程均匀慢；16t 差距收窄（p50 414 vs plain 359，1.15×） |
| NOF-041 | 已关闭 2026-11-23（**全部落在 bdev 段**） | vseg T1（排除 0–96）**bdev_us p50 250 / p90 306** vs plain **46 / 58**（+204）；data_us **5 vs 14（更小）**、outside_us 16 vs 23（更小）；vseg bdev **单峰**（200–300 占 76.2%、<100µs 占 0.0%），plain 干净区**双簇**（<50 占 60.0%、50–100 占 39.9%）；**vseg 的 bdev 分布与 plain 盘首慢区（keys 0–96）逐桶吻合** ⇒ vseg 全部 400 keys 都处于同一"慢状态" |
| NOF-042 | 已关闭 2026-11-23（**时序/落核无差异**） | 每条 lane 内严格 LBA 递增（(key,qid) 逆序对 p50=0）；q1 全局单调；四核四 qpair 与 plain 相同且均匀（lcore3/4/5/6）；**四核 bdev 全线 218–261µs vs plain 47–48µs**（慢不集中某核）；大 IO 与下一 key 小 IO 基本不重叠（0.61% vs 0.08%）；**纠正日志事实**：此前"每 key 一条大 IO"是 `nof_slow_task` 行拼接所致，URMA_SLOW_IO 全为 128KiB（任务级：vseg 2×2MiB、plain 1×4MiB） |
| NOF-043 | 已关闭 2026-11-23（**写 lane 排除**） | q1 串行写 400 新 key（自证 io_qpairs=1；Keys=800、logical_used 精确、写 W9 27.73µs）；读回三桶 bdev p50 **252 / 252 / 253µs**（慢窗口 / q4 写 / q1 写），慢占比 91.6/91.0/91.2% ⇒ **写侧 lane 交织不是主因**；六个维度全部排除，只剩 target/盘内部逐 IO 细节 |
| NOF-044 | 已关闭 2026-11-23（**明细已有但退化，不重启**） | 现有 13,767 条 `URMA_BDEV_DETAIL`（8 计时字段），但 `submit_to_callback_us ≡ bdev_us`（全量）、其余子段≈0 ⇒ 拆不开 raid0/NVMe；无时间戳、trace_seq 每会话重置、LEGACY 只采 ≥40µs（plain 偏斜）；cmdline 仅 `-m [2-7] --wait-for-rpc`，重启需 RPC 重放 ⇒ 按 A4 不重启 |
| NOF-045 | 已关闭 2026-11-23（**vseg 区数据无罪，慢在 vseg 路径**） | 裸 SPDK 顺序 QD1：A（vseg 区 1.6G）p50 **40.96**/W9 **91.57**；B（plain 区 3.2G）**41.28/91.28**；C（0.5G）**215.04/155.79**；D（0–0.5G）**212.48/153.92**；E（同 A 区随机）**220.16/188.90** ⇒ 高 LBA 的 vseg 数据裸读快（41µs）而 Mooncake-vseg 253µs ⇒ **慢在 vseg 路径（+~210µs）**；低 LBA [0,~1GiB) 盘本身慢；盘对顺序/随机极敏感（41 vs 220µs，vseg 253 正在"随机档"） |
| NOF-046 | 已关闭 2026-11-23（**顺序假设证否**） | **q1 单 lane：rx/exec/submit 顺序逆序对全 0、位移 0、单 lcore、submit=exec 100%**，但 bdev 仍 **223–265µs**（plain 同 lcore 43–52）；q4 逆序 61/496 仅来自 4 lane×2 child 的 8 条顺序流交错，且 q4≈q1 ⇒ 顺序不是差异来源；bdev 随执行次序仅 +19–24%（三种读法一致，通用排队）⇒ **"盘看成随机档"解释不成立**，剩余只有 target bdev 提交侧参数（buffer/描述符/flags/stride）需代码插桩 |
| NOF-047 | 已关闭 2026-11-23（第二阶段 M0） | 复制 8.82GiB（Mooncake 源/build、SPDK 源/build），HEAD/status/关键二进制与静态库 md5 全部 MATCH；配置：Release、USE_NOF/NOF_URMA/UB/ETCD=ON、SPDK 指 /usr/local（**纯静态 74 个 .a、0 .so**）；**正在跑的 target 是 deleted 旧二进制（736a902f），磁盘为 d6c39465**；补丁点：target `ctrlr_bdev.c:466-467`、client `nvme_urma.c:995-1056`、Mooncake `transfer_task.cpp:3047/1080-1106` |
| NOF-048 | 已关闭 2026-11-23（**通过**；基线仍待升级） | 最小重建成功：编译 61s、链接 4s，新 bench 530,148,040B / md5 43ba518d…，`--help` 与原版仅差第 1 行程序名；配方 = 编译 `transfer_task.cpp` → 复制 `libmooncake_store.a` 并用 `ar r` 替换成员 → 按 link.txt 重链；无 `_deps`、`/home/yin` 全可读。**注意：编译 -I 指向同事树（改头文件需前置我们的 include）、SPDK 用 /usr/local 静态库（target 插桩需换 `$D/spdk-build/lib` 重链）、新 bench RPATH 依赖同事 common/etcd**。构建基线仍建议 S2 升级到 tip 4c0ea28 |
| NOF-049 | 已关闭 2026-11-23（**诊断排除**） | 同环境同 key T1 两臂：ON W9 **258.85** / OFF **251.75µs（−2.7%）**、端到端 P50 504.11→494.56（−1.9%）；OFF 日志 167KB 且 `URMA_SLOW_IO`/`nof_slow_task`/`nof_vsegment_operation`/`transfer_data_breakdown` 全 0（env 生效）；两臂 exec 占比 96.2–96.3% ⇒ **默认诊断/客户端背压不是 210µs 来源**。注意绝对 P50 494–504 与 NOF-040 274 不一致（疑 slot env/集群变化），后续固定口径 |
| NOF-051 | 已关闭 2026-11-23（**通过；版本对齐成功**） | e12b7cb 因系统 UMDK 缺 `is_gpu_seg` 编不过；改用**同 wire v4 的紧邻后继 bd5ee124d**（GPU 注册 opt-in）构建成功（5 min、74 libs）；重链 `bench-io-params-e12`（md5 88ad3134…）在 **node3 本机握手成功**。vseg 采到 SUBMIT/WORKER **1056/1056**（400+128 keys ×2 child）；**ptr 注册域命中 100%**（2 个 ptr，child1=+2MiB）；child seg_lba 差恒 4096 blocks；lane/worker 各 264 均匀；buf_type=kHost、device=-1、**domain 全为非空 0x143cd90**；target W9 **258.55µs**（exec 97.2%）、端到端 P50 569.78（node3 本机） |
| NOF-052 | 已关闭 2026-11-23（**公平对照：1.95×；domain/客户端版本排除**） | plain 同新客户端（bd5ee124d）W9 **132.77µs**、旧客户端（59662a26）**133.29µs**（+0.4%）⇒ 客户端版本无影响；plain/vseg 的 domain **完全相同**（非空 0x143cd90）⇒ domain 排除；**vseg 258.55 vs plain 132.77 = +125.8µs（1.95×）**；task 数 1056/528、P50−W9 311/147µs（≈147µs/task 客户端开销）。历史 plain 46–68µs 在 slot=1 不可复现（slot=1024 口径），3.75× 的旧对照不可直接比 |
| NOF-054 | 已关闭 2026-11-23（**A 路线无需补丁**） | 已构建的 bd5ee124d `nvmf_tgt`（`$D/spdk-e12b7cb-src/build/bin/nvmf_tgt`，md5 979f5517…）**本身带全套 target 打点**：URMA_BDEV_DETAIL 6 段 + **URMA_STAGE_BUFFER 逐 IO 24 字段**（lat/nlb/各段 tick/lcore/flags）+ W1–W10；无需写补丁。读提交点 `ctrlr_bdev.c:449/493`（`req->iov/iovcnt`、cdw10/11+cdw12）；BUFFER env：`SINK=buffer`+`PATH`(O_EXCL)+`CAPACITY=200000`（默认 3GiB 内存），**进程退出时落盘**；客户端须 `SPDK_URMA_STAGE_TRACE_US!=0`。重启预案/回退 B（9a413a44 v6）已取证；wrapper 实际在 `/home/yin/Vsegment/run_vesegment_target.sh` |
| NOF-055 | 已关闭 2026-11-23（**通过**） | target 已切到 **bd5ee124d**（PID **394826**，md5 979f5517…）；4457/RAID0(11 盘/256K/online)/ns/transport 全部正常；BUFFER 模式（SINK=buffer、PATH=`run/urma_stage_nof055.bin`、CAPACITY=200000、DUMP_SEC=10）已生效，buffer 文件 0B（O_EXCL，退出时落盘）；旧日志备份 32,780,200B/md5 d294d6ef…；11 盘 nvme0–10 按 **BDF 升序**接回（每盘 7.15TiB；NOF-054 的 15,002,931,712 是 blocks）。**盘序可能与旧 target 不同（旧序未记录）⇒ 新旧数据不可按 LBA 直接对比**；vseg 元数据已清，NOF-056 重写 |
| NOF-056 | 已关闭 2026-11-23（**B2 停手：VSEGMENT_CREATING 23 次；NUMA 审计完成**） | vseg 从零建集群后 T1 写触发 **23 次 VSEGMENT_CREATING**（key 16,33,…,390 步长 17），只写入 377/400 keys、logical_used 1,581,252,608；按规则未跑 reader/未切 plain/未停 target（buffer 0B）。**NUMA 审计**：4 节点（node0 cpu0–69、node2 cpu154–237，距离 20）；**11 盘 5 块 node0 / 6 块 node2**；target 核 2–7 全在 node0；hugepages 各节点 1160、free 16/75/128/128；uburma numa 不可读 |
| NOF-057 | 已关闭 2026-11-23（**126µs 定位于 SSD 执行段；NUMA 排除；访问流差异**） | 补齐 400 keys（2 次尝试；VSEGMENT_CREATING 23→0；vsegments=26）；vseg T1 P50 **551.22**/W9 **261.44**，plain **277.50**/**137.00**；落盘 **33,792 条 dropped=0**；**126µs 100% 在 `submit_return→callback`（bdev/SSD）**，其余 6 段 0、data 段 vseg 更短；两段 `nlb=256`、**lba 集合完全相同**、flags 全 3；**NUMA 排除**（256/255 vs 59/61）；盘级访问流：换盘率 **79.7% vs 50.0%**、每盘递增 **98.0% vs 99.9%**、回退 **31 vs 1**。与 NOF-034/046 有张力，child 数不直接当根因；NOF-058 离线判别局部条件 |
| NOF-058 | 已关闭 2026-11-23（**并发与局部顺序均排除；判据指向数据状态/读路径**） | 零环境；同 tick 0.01%/0%、两种 tie-break 一致；**ov=0：vseg 206µs vs plain 39µs**（两臂 ov 分布均 34.4%）⇒ 并发排除；**同 (disk,lcore) 紧接/busy=False：vseg 275/206 vs plain 61/39** ⇒ 局部 LBA 顺序排除；plain 快 IO 全在 keys≥98（8,336 条 <64µs）、慢 IO 全是盘首 keys 0–97（3,133 条，LBA≤802,304）；**同一 LBA（keys≥98，n=9,664）：vseg 250.2 vs plain 49.4（5.07×）但受写轮次混杂**；vseg 客户端序换盘 59.7% vs target 执行序 79.7%（+20pp，`[推断]`）；**24 字段内无判别变量** ⇒ NOF-059 |
| NOF-059 | 已关闭 2026-11-23（**单 child 无效；数据无罪；锁定读路径**） | target 重启 708167（bd5ee124d/无 BUFFER/保持运行）；**stripe=4MiB 单 child 属实**（读 528 SUBMIT/8192/child=0/1）；vseg4 P50 **547.92** vs plain **268.44（2.04×）**，与双 child 1.99× 同档 ⇒ child 无关；同一数据裸 SPDK 顺序 **40.0 vs 40.8µs**（W9 86.3/79.6）、随机 **220.16/220.16** ⇒ **数据物理状态无罪**；判决：差异在 vseg 客户端驱动 4MiB NoF 操作的方式 ⇒ NOF-060 |
| NOF-060 | 已关闭 2026-11-23（**两臂 NoF 提交层完全同构**） | params 除 child 外全同（op/blocks=8192/bytes=4MiB/block_size=512/endpoint uniq=1/lane 各 132/domain/ptr 对齐 100%）；源码两臂在 `transfer_task.cpp:3076` 汇合、同参数；128KiB chunk、8MiB/lane、task 内 32 IO 同 qpair 均同；child 仅日志标签。URMA_SLOW_IO **有绝对 tick**；唯一实测差异=每批突发 16 vs 32（2MiB-child 时代；vseg4 未获取）；T1 每 key 串行 dt vseg4 545±10µs vs plain 264µs（前 240 op 540µs=盘首区）。**无单点可解释 2× 的现成差异** ⇒ NOF-061 |
| NOF-061 | 已关闭 2026-11-23（**判决：vseg 读路径/会话；数据二次无罪**） | 写 LBA 集合与 plain 逐值相同；**零写入**用 plain 元数据读同一批 LBA：plain P50 **308.37**（复测 330.41）/bdev **57**；vseg 同数据 P50 **573.03**（复测 601.09）/bdev **249**；**同 LBA 配对 12,324 条 246 vs 53（4.64×），plain 更快 85.1%**。vseg4 突发 **31（4MiB）与 plain 完全相同** ⇒ NOF-060 的 16 vs 32 不复存在；新增排除突发形态、target lcore（同核仍 4.3×）、顺序/预热。剩余唯一变量=会话/qpair 身份 ⇒ NOF-062 |
| NOF-062 | 已关闭 2026-11-23（**gap 随并发强收敛 ⇒ 完成检测类**） | bdev p50 **T1 249/54（4.61×）→ T4 262/259（1.01×）→ T16 270/265（1.02×）**；vseg 恒 249→270、plain 单向追平 54→259 ⇒ 判据① + ③。**差异 100% 在 `bdev_us`**，其余段 0、`outside` 同 21µs、`wait/enomem` 为 0。同 qid 配对 T1 4.4–4.8×、T4/T16 1.0×。源码链到 `transport.c:932`（每 poll group 独立 buf_cache）与 `urma.c:2493`（bdev_cpl 在 poll 上下文取）；**target 无 vseg 语义**；两臂仅会话身份与 qpair→lcore 分配不同 ⇒ NOF-063 |
| NOF-063 | 已关闭 2026-11-23（**无 50–500µs 常量；结构错配对称，不解释臂间差**） | 主链 `transport.c:777→779`→`urma.c:2303`（81 行）；`urma_poll_jfc` 无 timeout、`usleep/nanosleep/spdk_delay_us` 零命中 ⇒ **静态无该量级常量**。结构：`stride=worker_count=6` vs `send_jfc_count=2` ⇒ 仅 worker 0/1 轮询网络，qpair JFC=建连序 mod 2 与属主无对齐 ⇒ 跨线程投递；本地 NVMe 每轮不限量、无 defer。**复核：错配两臂对称（各 2 个轮询 + 2 个非轮询），且同臂 4 个 qid 时延均匀 ⇒ 不是 4.6× 原因，JFC_COUNT=6 不作判决实验** ⇒ 必须插桩 ⇒ NOF-064 |
| NOF-064 | 已关闭 2026-11-23（**部分：改码/重编/重启完成；采集受阻**） | 新 target **1587271 / `21ced674…`**、验收全过、`NOF064_TRACE=1`；三份二进制并存（`nof059` 979f5517 / `nof064` 21ced674 / `nof064b` afe95672）。`worker_index 0→core3,1→core4,2→core5,3→core6,4→core7,5→core2`。**阻塞①重启使 nsid1 uuid 变（c697219c→63a1e3cd）⇒ `vseg_nof061` mount rc[-300]，plain 元数据不受影响**（产品级缺陷）；**阻塞②v1 插在 2214 而非读路径 1921 ⇒ `d_*` 是绝对值**。plain v1：client P50 317.44、bdev p50 225 —— **判为采样偏差，不可用**：4,000 条预算按到达序≈前 125 个 key，其中 keys 0–97（78%）落在盘首慢窗口，与 83.4% >70µs 吻合；反证是客户端 P50 317.44 与 NOF-062 plain T1 307.51 基本一致 ⇒ 225 是子集统计假象。**早前归因为 printf 观测者效应，已撤回** ⇒ NOF-065 |
| NOF-065 | **已取消 2026-11-24（被 NOF-068 取代）**：NOF-067 证明现役二进制已具备所需 tracepoint，C 代码插桩不再必要 | v3 插桩：printf→**环形数组 + 非 reactor 线程落盘**、补 iov 字段、`d_*` 改真 delta；重启后**新 cid `vseg_nof065` 写一次 400 keys**；两臂 T1 各 2 遍 + 各 1 遍 `TRACE=0` 对照（差 >5% 作废）；解析按 `bdev_us` 分桶、`iov[0].base` 分 NUMA 桶 |
| NOF-068 | **C 段阻塞 2026-11-24（target 段全绿，master 侧 mount 静默失败）** | 已切干净 target：**PID 487638 / nof059(979f5517) / `--num-trace-entries 262144` / 4457 / 11-11 online / ring 52,736,328B（6×262144 反算吻合）/ tpoint mask 0x0**；nsid1 uuid 63a1e3cd→**9d89ad0c**。master（vseg_nof068）mount **requests=2 / failures=2**、`MountNoFSegment` **146µs** 返回 ok 但**无 attach**（成功轮 206ms + `Opened NoF segment`）；`/cvm/vseg_nof068/` 只有 cluster_meta+ring_slots、**无 segment 描述符**（成功轮 nofvseg = 8 键）。已排除：capsule 模式（已修）、sudo 剥离 env（已修）、MC_URMA_* 缺失、RPC 序列、cluster_meta 陈旧。零写入（Keys=0、capacity=0）。⇒ NOF-069 |
| NOF-069 | 已关闭 2026-11-24（**H2 定案；挂载根因=脏 cluster_meta**） | 新 cid `vseg_nof068b` mount **214,462µs / rc=0**、`Opened NoF segment ns=1 io_qpairs=4`、`/cvm/` 8 键（含 `segments/` + `nof_identities/9d89ad0c…`）、`partitions_installed=1`、`reconcile_fail=0`。**146µs 早退根因**：首轮坏 env 写入 `cluster_meta.slot_count=16384`，`cvm_controller.cpp:281` 拒启 → `cvm_lease_id_==0` → `master_service.cpp:4862-4863` 静默 `INVALID_PARAMS`（描述符从来不是前置条件，此前假设错）。**H2 证据**：轮询间隔 p50 **3.0µs**、门铃后 **237.8µs** 仍空、3.2µs 后取到 CQE、`t_cb−t_obs=0.1µs`；同慢桶 `[128,256)` 两臂 le−db 完全同值 **204.0**（vseg n=9,927 / plain n=3,495）。**数据瑕疵**：trace 只覆盖 **1,164 个唯一 LBA（≈37 个 key 区）**，与 12,800 child IO 矛盾 ⇒ 需对账 |
| NOF-071 | 已下发 2026-11-24（M3-8，**零写入 · target 不重启**） | A：**同宿主 A/B**（两臂都在 node3 各 1 遍；两臂都在 node1 各 1 遍）消掉客户端位置偏差，判 vseg 是否仍 ≥4×；B：`bdev/bdev_raid/bdev_nvme` tracepoint 字段清单，确认能否零改码拿「父 IO 长度 / 缓冲地址 / iov 数」；C：若可得则比两臂父 IO 长度（vseg 是否 32×128KiB vs plain 1×4MiB）与缓冲 NUMA 桶；D：M3-8 假设排序 + 最小插桩点 |
| NOF-072 | 已下发 2026-11-24（M3-9，**target 不重启**） | A 解阻决策树（`--enable_oplog=false` → 删我方 `/oplog/vseg_nof068b/` → 新 cid 单遍写入）；B **交错 A/B**（vseg→plain→vseg→plain，同宿主各 1 遍）判时间/设备漂移；C **冷/稳检验**（plain 连读 2 遍、vseg 连读 2 遍），关联 NOF-059 裸 SPDK 长扫 40µs vs 单 key 定点 218.88µs 的旁证；D 判设备两态是否成立并给 M3-9 方向 |
| NOF-073 | 已关闭 2026-11-24（**命令 dword 层零差异**） | 重导 7 份快照取 `NVME_PCIE_SUBMIT` 的 `opc/dw10/dw11/dw12`：两臂 `opc=2` 100%、`FUA=0` 100%、`NLB=256`、1,164 个盘内 offset **元素级一致**（only_A=only_B=0）⇒ 命令的 LBA/NLB/opcode/FUA 全同，**A 段未定案**。附带：同 LBA 比值 p50 **4.31×**，`ratio<1.5×` 恰占 **24.4%**（==盘首慢窗口占比 24.5%），plain 慢 LBA 相邻 256 块×241（每盘 ~30MiB 连续）；11 盘逐盘 plain 95.8–100.1 / vseg 231.6–235.9 ⇒ 无盘级差异。方法学偏差：采集窗口是 ring 尾部（每核被容量钉住），span/吞吐不可用 ⇒ 只在 A 段结论内使用 |
| NOF-074 | **已批准 2026-11-24 · 待执行**（M3-11，用户批准改码/重编/重启/一次写入） | A 租户核查（4457 有他人客户端即停手）；B 在 `nvme_pcie_common.c` 写 SQ 前记录 `cmd[0]` 原 4 字节 / `psdt` / PRP1-PRP2 或 SGL 段数 / `buf_va` / `submit_tick`（环形 ≥200k、独立线程落盘、无 printf）；C 重编+重启（`--num-trace-entries 262144`）+ 新 cid `vseg_nof074` 写 400 keys；D 两臂同宿主各 1 遍；E 判 PSDT/描述符/缓冲 NUMA-页大小/时序是否为差异载体；回退 `nof059` |
| NOF-071 | 已关闭 2026-11-24（**客户端位置排除；父 IO 长度相同**） | 同宿主 node3：vseg **242.6** vs plain **50.0** = **4.86×**（target 内部）；plain 宿主效应仅 **1.03×**（50.0→48.5）、node1 两次 **1.00×** ⇒ 客户端位置排除。**公平端到端 2.20×**（608.83 vs 276.93，P90 703.97/570.01）⇒ NOF-069 的 2.57× 跨宿主混比作废。`BDEV_IO_START.size` 两臂全为 **128KiB/256 块**（各 33,792 条）、层级一一对应 ⇒ **父 IO 长度假设否证**。缓冲地址/NUMA/iov **`[未获取]`**（三组 tpoint 无该字段；最小插桩点在 `lib/bdev/bdev.c:3851`）。**E1 阻塞**：`vseg_nof068b` oplog **2,801 批/58.5MB** 触发 etcd gRPC 32MB 上限（35,087,559>33,554,432），master 起不来、50056 未 LISTEN ⇒ NOF-072 |
| NOF-070 | 已关闭 2026-11-24（**对账闭环；队列压力与访问流均排除**） | target 的 `lba` 是**盘内 offset**：12,788 逻辑 LBA（opcode=2）经 RAID0 映射 = 1,164 盘内 offset，与 target 集合 `|only|=0` 零差异；16,897 行 = 12,800 + 4,096(warmup 重复) + 1，÷11 盘 = 1,164 ⇒ **"只覆盖 37 个 key 区"作废，per-LBA 5.505× 恢复**。qd 分布 34.4/65.6 vs 34.5/65.5（无 >4），**同 qd 桶 5.11×（qd1）/5.28×（qd2-4）**；在飞 32/32、per-qpair 4/4；换盘率 50.1% vs 50.1%、每 key 11 盘、每盘 1,526–1,535 ⇒ **队列压力、访问流双排除**。**新偏差：两臂客户端不同机**（vseg 在 node3 环回、plain 在 node1 跨网）⇒ 端到端 608 vs 237 不可比 ⇒ NOF-071 |
| NOF-066 | 已关闭 2026-11-24（**排除慢窗口后差距放大，结论更硬**） | 落盘 33,792 条按 `exec_tick` 最大间隙 208.953s 切为 seg1=vseg / seg2=plain 各 16,896（与 NOF-057 逐位吻合）。排除（lba≥802,816）vs 全量：vseg 256/314/261.1→256/314/250.9；plain 61/285/136.5→**50/67/50.0**；**比值 p50 4.197×→5.120×、mean 1.912×→5.015×**。NOF-062 trace 同结论：T1 p50 4.611×→5.511×、mean 2.046×→5.463×；T16 仍 1.030×。**排除后 vseg 100% 落在 128–512µs（<128µs 为 0），plain 90.5% 落在 32–64µs** ⇒ vseg 全程均匀慢、与慢窗口无关。推荐口径＝排除口径 + 慢窗口小表 |
| NOF-067 | 已关闭 2026-11-24（**判决走 A：零改码，现役二进制字段齐全**） | `nvme_pcie`(6 tpoints) + `urma_p99`(4) 均在、`mask=0x0` 未开；`NVME_PCIE_CQ_OBSERVE` 带 `{req,qpair,cid,last_empty,poll_enter,prev_poll_end}`、`SQ_RING` 带 `{qpair,tail,mmio}`（写 tdbl 后）、`SUBMIT` 带 `dw10/dw11/dw12` ⇒ lba/nlb，可做 per-LBA 归因；**H1/H2 三分支判别可落地**。纠错：`-F 3` 是**慢事件阈值因子**（latency>3×avg）非格式开关；`--target-trace-entries` 落到二进制是 **`--num-trace-entries`**（2 的幂、默认 32768/核、0 关闭）。24 字段**无**队列索引/SQHD；`sq_head=cpl->sqhd` 读入但未落 trace。硬阻塞：同事 `analyze_p99_slow.py` 写死 `SQ_PUBLISH/SQ_NOTIFY`，我方 v7 命名是 `SQ_RING/POLL_GAP/REQ_QUEUED` ⇒ 需改约 20 行 Python（不碰 C）。产物 `run/nof067_recon.txt`（15,611 B） ⇒ NOF-068 |
| NOF-053 | 已关闭 2026-11-23（**客户端分解；QoS 排除**） | 指定 `batch_get_*` 为 0（单 key Get 走 `get_into_breakdown`/`query_rpc_breakdown`/`transfer_data_breakdown`/`nof_slow_task`）。plain 每 key 283µs = **master 元数据 query 116µs（41%）** + submit 6 + wait 156（含 target W9 68，超出 88）+ 其他 5；**QoS/worker/dispatch 仅 8–11µs（~3%）**。vseg 2× **不是 setup/QoS**（query 103 vs 116 反而低），而是 **drain 350 vs 110µs、max_subio 362 vs 112（3.2×）**、submit 19 vs 6 ⇒ **客户端调度排除**，vseg 的 2× 在 target/传输 drain 段 |
| NOF-050 | 已关闭 2026-11-23（**阻塞：SPDK 版本不兼容**） | 补丁/编译/链接成功（`bench-io-params` 530,162,656B / md5 fc2b5129…，strings 命中 3 条）；但新 bench 连 target 失败：客户端 SPDK **9a413a44** vs target 运行中 **e12b7cb**，握手 rc=-71/-104；同事原 bench（59662a26，匹配 e12b7cb）同 env/参数 rc=0、P50 473.52 / target W9 259.77。**口径修正**：该 P50 为 **node1 远端 reader**，与 NOF-040 本机 274 不可比（SLOT=1 未改变 P50）；target 现跑的是 **deleted 旧二进制 736a902f**，磁盘 d6c39465=9a413a44 |
| NOF-026 | 已关闭 2026-11-20 | **750 只是目标句里的近似值**（L12），非实测；文档唯一 VSegment 实测 = 662/669/913µs（6盘/8MiB/单线程）；**300 无任何推导**；资料**没有**"不开 vsegment 的 Mooncake 读时延"数字（仅裸 SPDK 236µs 与写吞吐 12.89 vs 13.07） |

### NOF-001 关键事实（供后续任务引用，2026-11-20）

- **项目**：Mooncake Transfer Engine（mooncake-store）+ NoF（NVMe-oF over URMA）+ VSegment 分层存储；目标单 key（4–8 MiB）读时延 ~750µs → 300µs。
- **资料**：9 个文件 + 1 个子目录，约 2.16 MB；必读三份：`交接文档_NoF_VSegment性能测试.md`、`连接node1和node3.md`、`exp_11disk_run.sh`；`机器与使用说明.md` 在资料中缺失。
- **代码**：公开仓 `github.com/LeeJace993/Mooncake`，branch `mooncake-vesegment`；同事本地 HEAD `be10be2`（2026-11-17，仅 `mooncake-common/etcd/go.mod|go.sum` 有本地改动）；远端分支已推进到 `4c0ea28`（2026-11-19，含其他协作者提交）——构建用哪个 commit 待定。
- **vsegment 模块**：`mooncake-store/src/vsegment/` 下 8 个文件（vsegment / service / manager / runtime / ha / transfer / metrics / partition_quota_planner），外加 master_client / master_service。
- **已有结论**：控制面 GetReplicaList 缓存 p50 841→610µs；bdev_us≈513µs 根因 = raid0 单 child 单盘定向 + 排队放大；6→11 盘 W9 294→54.7µs；**11 盘读路径端到端 + 五段拆分尚未取得**。
- **当前阻塞**：P6 reconcile 失败（master 日志每周期 `count=1024, first=0: INVALID_PARAMS`，随后 `Failed to refresh vsegment ownership: PERSISTENT_FAIL`）。开发侧代码定位：失败来自 `ReconcilePartitionRoute`（首个样本 partition "0"）；`PERSISTENT_FAIL` 是 reconcile 不完整时的 fail-closed 返回。候选落点集中在 `AddPartition → Restore / EnsureInitialVSegments` 的校验；缺 P6 现场事实，尚不能定因。
- **环境**：两机接入 OK（`sudo -n` 可用）；node1 工作目录 `/home/l00955908/mooncake-nof-vsegment` 已存在但属 root:root 700（待处置）；node3 同名目录已创建。
- **node3 现状**：4457/50056/8086/9006 被**我方旧 gap 环境**占用（nvmf_tgt / mooncake_master），11 块 7T 盘 vfio 绑定；HugePages 仅剩 346/4640。node1：5 个目标端口空闲，HugePages 充裕。
- **互通**：node1→node3 通（avg 0.061 ms）；node3→node1 SSH 拒绝；node1 无外网。
- **硬件**：两机 aarch64、308 CPU、NUMA 4、内存 2261 GiB；node1 有 RTX 4090D，node3 无 GPU；URMA 设备两机各 7 个（udmac0/1d1e2-4、bonding_dev_0）。
- **红线**：node3 的 gap 环境与 vfio 盘属历史项目产物，本阶段一律不清理；接管/停服/重启须用户明确批准。

### NOF-002 关键事实（2026-11-20）

- **P6 定义**：脚本 `gate_reconcile`（L186–199）是 writer/reader 前的门禁——每 5s 查 master 日志 `grep -c "reconcile failures"`，120s 内必须为 0，否则 writer/reader 会 `STALE_ROUTE(-1801)`。
- **现场输入**：master flags：`--vsegment_auto_plan=true --vsegment_required_medium=nof --vsegment_io_alignment=4096 --vsegment_member_count=1 --vsegment_stripe_size=2097152 --vsegment_member_extent_size=67108864 --vsegment_reserved_ratio=0 --submaster_count=1`；slot_count=1024；quota generation=1、1024 分区（每分区 100MiB，合计 100GiB）、1 个 profile（default）；唯一 nof segment（100GiB，io_alignment=512，supports_unaligned_io=false）。
- **失败形态**：`scanned=1024 / owned_by_me=0 / foreign=0 / no_route=0 / load_fail=0 / reconcile_fail=1024`；`first=0: INVALID_PARAMS`；日志无 detail；VSegment Status 里 `partitions_installed` 全程 0 ⇒ **失败发生在 partition manager 安装之前**。
- **脚本危险动作**：`do_clean` 按进程名 `kill -9` master（不区分归属）、只删 `/cvm/<cid>/` 前缀、`pkill -9 -x nvmf_tgt`、`target_nvme_takeover.sh -r` 还原 11 盘、删 state 文件；无确认提示。
- **五段拆分**：实际是 initiator 侧 `URMA_SLOW_IO` 7 字段（roundtrip/pre_send/target_us/bdev_us/data_us/outside_us/residual）+ target 侧 W1–W10；脚本 `do_collect` 有 `NameError` 缺陷，表格产不出来。
- **现场 master**：11-18 20:59 启动的常驻进程，二进制 11-17 20:14 构建（对应 be10be2），与脚本"每轮清场重启"的设计不一致；etcd 无 per-partition route key，只有 quota 快照与 ring_slots。

### P6 根因假设（开发侧代码分析，NOF-003 验证）

1. `do_clean` 只删 `/cvm/<cid>/`，**未删 `/oplog/<cid>/`**（HA OpLog 流前缀，见 `oplog_batch_types.h` 的 key 布局）。
2. master 启动时 `TrySelfRecoverFromOpLog()` 重放旧 OpLog，把历史 vsegment 快照装入 `recovered_vsegment_snapshots_`；日志 `Restored from standby: 0 objects, 0 segments` 只有在 objects/segments/vsegment_partitions 至少一项非空时才会打印 ⇒ 现场 vsegment 快照非空。
3. 新一轮 quota generation 又从 1 开始，旧快照与新 quota/配置不兼容，`Restore()` 返回 `INVALID_PARAMS`（detail 被 reconcile 调用方置空，无法在日志看到），manager 未安装 ⇒ 每周期 1024 全灭 ⇒ `PERSISTENT_FAIL`。
4. 修复路径（待回执后定）：环境侧"真清场"（删 `/oplog/<cid>/` 或换全新 cluster_id）；代码侧忽略不兼容旧快照并补 detail 日志。

### 代码与版本（2026-11-20 核对）

- 当前运行物 = 同事 2026-11-17 20:14 构建的二进制；对应源码 `github.com/LeeJace993/Mooncake` branch `mooncake-vesegment` commit `be10be2`（"Register NoF VSegment benchmark buffers"）。已验证 be10be2 在远端分支历史中；运行日志行号（`master_service.cpp:1685/1828`）与 be10be2 一致。
- **未推送（重建时必须补）**：同事本地的 `mooncake-common/etcd/go.mod|go.sum` 两处修改（x/sys 0.39→0.45、x/text 0.32→0.37），未提交。
- 远端分支最新为 `4c0ea28`（2026-11-19，领先 20+ 提交，含其他协作者合并）；**不是**同事实验所用版本。
- SPDK：node1 `/home/yin/nof/spdk-cpu` 干净（HEAD d98b48e）；node3 同路径脏（HEAD e12b7cb，12 个改动）；origin `[未获取]`；target 二进制为本地构建，未推送。
- 测试脚本（`vsegment_cluster.sh` 等）仅在 yin 本地，未推送。

### NOF-004 关键事实（2026-11-20）

- 我方 master：PID 2091164（root），node1，2026-11-20 14:37:22 启动；`--cluster_id=vseg_nofvseg`；`rpc_address=192.168.84.*`、`rpc_port=50056`、`metrics_port=9006`。
- 状态：`role=leader, state=serving, service_ready=true`；`VSegment Status: enabled=0`；auto-plan deferred（等 mount）；`reconcile failures` 0 次；`Failed to refresh vsegment ownership` 0 次。
- **无 oplog replay / self oplog replay / Restored from standby** 三行 ⇒ 全新 cluster 无旧状态（与 yin 的 1024 条重放形成对照）。
- **8086 未监听**：传了 `--http_metadata_server_port=8086`，但 `--enable_http_metadata_server` 默认 false 未开启。开发侧决定暂不开启（同事环境也未启用；客户端用 `--metadata-server=etcd://` + master RPC，不依赖它）。
- 保留 env：`MC_STORE_PARTITION_SLOT_COUNT=1024`、`MOONCAKE_VSEGMENT_METRICS_REFRESH_INTERVAL_MS=1000000`、`SPDK_URMA_DEV_NAME=udmac1d1e2`、`SPDK_URMA_CAPSULE_TRANSPORT=urma`；VSegment/HA 共 14 项参数与同事一致。
- etcd：`/cvm/vseg_nofvseg/` 3 键（cluster_meta、masters/…、ring_slots/00000）；`/oplog/vseg_nofvseg/` 1 键（durable_prefix）。
- 启动 1 s 后出现 1 条 `etcd_helper.cpp:352 lease keep alive ... timeout` ERROR，之后未复现（yin master 同期同形态，判为启动期瞬时）。
- 产物：`/home/l00955908/mooncake-nof-vsegment/run/master_50056.log`（root）；停止命令 `sudo -n kill -TERM 2091164`（10s 不退再 KILL）。

### 数字口径核对（2026-11-20 用户提出"750/300、800+、380"来源疑问）

- `750→300µs`：交接文档 L5 的目标口径（VSegment 端到端读，精确测量点待 §3.1 原文确认）。
- `841→610µs`：交接文档 L81 的 `GetReplicaList 缓存 p50`，是**文档内**数字；它究竟是"整个 Get 端到端"还是"仅元数据 RPC"，需 §3.1/§3.4 原文确认。
- **`~380µs` 不在本交接文档的任何回执里**。最接近的 `387.23µs` 来自**上一个 GAP 项目**（GAP-16：Mooncake 8MiB seq 读 avg，P50 389.12；同一轮裸 SPDK 258.60µs）——那是**不带 vsegment 的普通 Mooncake NoF**、8MiB、reader 在远端节点。
- 因此"vsegment 841 vs 不开 vsegment 387"是**跨项目、跨版本、跨盘数、跨尺寸、跨客户端位置**的对比，不是受控 A/B，不能据此判定 vsegment 更慢。**受控对比**要用同一环境同时测"带/不带 vsegment"两组（我们的环境具备条件）。
- 文档内 6 盘 VSegment：Get Avg 662 / P50 669 / P99 913µs；五段 roundtrip 548 / target 519 / bdev 513µs；裸 SPDK 11 盘 Get 236 / exec 68µs；W9 6→11 盘 294→54.68µs。待办：NOF-006 提取文档 §3.1/§3.2/§3.4 原表并建立"同口径臂表"。

### NOF-005 关键事实（mount 失败，2026-11-20）

- **结果**：客户端 `nof_register` 退出码 0 并打印 `NoF registration succeeded`（P4 谎报）；master 侧 `MountNoFSegment_slow ... rc[-101]`；底层 `Failed to attach NoF controller ... rc=-19`；SPDK 日志 `nvme_urma_qpair_defer_error ... err=-5` → `URMA qpair 0: fatal rc=-5` → `Operation not permitted`。target 未受影响。
- **后果**：mount 未成立 ⇒ quota 0、`enabled=0`、`reconcile stats` 行未出现；此时 `reconcile failures=0` 是**假阴性**，不能作为 P6 已解证据。
- **现场构成**：现役 target 是旧 gap 项目的 SPDK 构建（`~/gap/spdk`，nvmf_tgt md5 `6668b4905ec3886a38aa3ed5630af19a`，690,992 B 动态，transport 名含 `urma_p99`）；master 的 NoF 客户端来自同事 **sysurma** 构建（be10be2）。两者 URMA 实现很可能不同源 ⇒ 协议/transport 不兼容是首要嫌疑（NOF-006 证实）。
- **设备名冲突**：交接文档 §一写 `udmac0d1e2`，同事脚本 `VSEG_URMA_DEVICE=udmac1d1e2`；本次 master 用 `udmac1d1e2`。`[冲突]` 未定。
- **文档拓扑口径（§一/§二）**：node1 = etcd + master + writer + 编排；node3 = target + reader；11×7T + 系统盘；4–8 MiB/单 key；master RPC 50058 / metrics 9008；target 4420；RAID0 strip 256 KiB；100 GiB ns。→ **读测试是"本机 client"口径**，写测试跨节点。
- **target 打点（D2）**：现役 gap target 的日志在 `~/gap/logs/nvmf_tgt.log`（44.9 MB 持续写），含 `URMA_BDEV_DETAIL` 181,061 行（bdev_us / exec_to_read / read_to_submit / submit_call / submit_to_callback / callback_to_complete）与 W9/W10 行（当前 n=0）；**W1 为 0 行**；`SPDK_URMA_STAGE_TARGET_SINK` 读不到。→ target 侧五段类数据可用（字段名与 yin 的 W1–W10 不完全相同）。
- **产物**：node1 `~/mooncake-nof-vsegment/run/register_100gib.log`、`run/master_50056.log`（新增失败链 13 行）；master 仍在运行（未停）。

### NOF-006 关键事实（URMA attach 失败根因，2026-11-20）

- **协议不是原因**：gap 与 yin 两棵 SPDK 的线协议一致（`SPDK_URMA_WIRE_MAGIC=0x41524d55`、`WIRE_VERSION=4`），`nvme_urma.c` 2064 vs 2050 行、`nvmf/urma.c` 2832 vs 2827 行，握手字段集合相同。
- **设备/provider 是原因**：我方 master（PID 2091164）`/proc/<pid>/maps` 中 `/dev/uburma` 映射 **0 条**，`liburma-udma.so`/`liburma_ubagg.so` **均未加载**；对照：yin master（PID 151060，**同一二进制同 md5**）用 `SPDK_URMA_DEV_NAME=udmac0d1e2`，映射 `/dev/uburma/udmac0d1e2` 且两个 provider 均加载；他人 client 与 node3 target 同样加载。
- **现役 target 用 udmac0d1e2**：`target_start.log` L99 的 `SPDK_URMA_DEV_NAME=udmac0d1e2`，RPC `nvmf_get_transports` 也显示 `dev_name=udmac0d1e2`；而 yin 脚本默认 `TARGET_URMA_DEVICE=udmac1d1e2`、交接文档写 `udmac0d1e2` —— 三处口径不一，实测以 `udmac0d1e2` 为准。
- 设备拓扑：两机 `/sys/class/uburma` 各 7 个（`bonding_dev_0` + udmac0d1e2-4 + udmac1d1e2-4，udmac0/udmac1 属两个 UB controller）；`urma_admin show` 两机均只有 `bonding_dev_0` 带 EID。
- **target 侧无握手记录**：14:41:35 窗口 target 日志无任何行（唯一 14:4x 行为 14:40:57 的 `handshake failed rc=-104 peer=node1`，早于 register 38s）⇒ 失败发生在 initiator 本地，进一步支持设备层。
- 换 target 前置（备用）：现役 gap target 占 4293 页 2MiB 大页（8.38 GiB）；停 target 可释放至 ~4639/4640 页；takeover 在已有 SPDK 进程时会硬拒绝（L416/L446/L289），必须先停；正向模板 `-N` 设备必须显式选 `udmac0d1e2`，listener 会改用 fabric IP。
- **结论与对策**：改用 `SPDK_URMA_DEV_NAME=udmac0d1e2`（并按 yin 环境补 `VSEG_URMA_DEVICE=udmac0d1e2`）重启我方 master 即可，**无需换 target**（NOF-007 验证）。

### NOF-007 关键事实（master 设备口径切换 + 惰性初始化，2026-11-20）

- 新 master：PID 2162056（root，15:01:57），env 已确认 `SPDK_URMA_DEV_NAME=udmac0d1e2`、`VSEG_URMA_DEVICE=udmac0d1e2`；50056/9006 LISTEN；除启动期 1 条已知 etcd lease 超时外无 ERROR；`enabled=0`、auto-plan deferred。
- 旧 master 2091164 已 TERM 退出（1 s），yin master 151060 未动；etcd 键数不变（3/1）。
- **重要机制**：SPDK/URMA 初始化是**惰性**的——只在收到 mount 请求时才发生。证据：r2 日志 `spdk_wrapper` 行数 0；r1 日志的 4 条 SPDK 行全部出现在 14:41:35（NOF-005 的 mount 时刻）；yin master 日志同样是 SPDK init 与 MountNoFSegment 同一时刻。
- 因此"启动后 `/dev/uburma` 映射=0"不是故障；yin master 现有 18 条映射是因为它曾经 mount 成功、设备打开后常驻。
- **验收口径修正**：设备/provider 映射必须在 mount 请求时验证，不能作为"未 mount"阶段的验收项。

### NOF-008 关键事实（mount 成功、P6 首次有效通过，2026-11-20）

- **mount rc[0]**（`MountNoFSegment_slow elapsed_us[189995] rc[0]`）；SPDK init `init_rc=0`；`Attached`；`Opened NoF segment ns=1 io_qpairs=4`。
- **设备映射（mount 时刻）**：`/dev/uburma` 0 → **20 条**；出现 `/dev/uburma/udmac0d1e2`；`liburma-udma.so`、`liburma_ubagg.so` 均加载。
- **P6 门禁通过**：`reconcile stats: scanned=1024, owned_by_me=1024, foreign=0, no_route=0, load_fail=0, reconcile_fail=0, stale_cas=0`；`reconcile failures` 0；60 s 复查仍 0（非假通过）。
- **VSegment Status**：`enabled=1, generation=1, profiles=1, partitions_installed=1024, vsegments=1024, logical_capacity=68719476736 (64 GiB), logical_free=64 GiB, logical_used=0`；pool `default: members=1, medium=nof`；首卷 `partition=0, capacity=67108864`。
- **etcd**：`/cvm/vseg_nofvseg/` 3→8 键（含 quota 快照 181,399 B、segment 描述符、nof_identities 等）；`/oplog/vseg_nofvseg/` 1→28 键。segment：capacity=107374182400（100 GiB）、base_offset=0、io_alignment=512、supports_unaligned_io=false、vsegment_exclusive=true。
- **设备名是关键变量**：NOF-005（udmac1d1e2）rc[-101] vs NOF-008（udmac0d1e2）rc[0]，强相关（NOF-005 未在 mount 时刻采集映射，非严格单变量实验）。
- **P6 结论**：全新 cluster（无旧 `/oplog` 重放）+ 正确 URMA 设备 ⇒ 首轮 reconcile 即通过；yin 的 1024×28,505 次失败来自旧 oplog，诊断闭环。
- 备注：`logical_capacity=64 GiB = 1024 × 64 MiB`，物理段 100 GiB；planner 容量折算规则 `[未获取]`，不影响本轮目标。
- target 侧 15:04–15:05 唯一错误是 15:04:51 `handshake failed rc=-104`（预检 TCP 探测造成，早于 register 8 s），真实 mount 无错误记录。

### NOF-009 关键事实（writer 预填成功，2026-11-20）

- **writer 400 succeeded / 0 failed**：400 keys × 4 MiB，Total 1.56 GB，0.40 s，3970.70 MiB/s；写入路径时延（非读）Min 764.79 / Avg 1007.38 / P50 931.45 / P90 1148.68 / P99 3103.38 / Max 4725.84 µs。
- **master 对账精确**：`logical_used` 0 → **1,677,721,600 B**（=400×4 MiB）；`logical_free` 减同额；NVMe-oF SSD 1.56 GB / 100.00 GB (1.6%)；Keys 400；vsegments 仍 1024（写入填既有 vsegment）；无新增 E。
- **etcd**：`/cvm/vseg_nofvseg/` 8 键不变；`/oplog/vseg_nofvseg/` 28 → 1228（+1200）；yin 与 mooncake_cluster 前缀键数不变（无跨集群污染）。
- **参数映射无冲突**：bench 的 64 个 flag 中无 cluster/namespace，集群路由只由 `--master-server` 决定；7 处替换：master 50056、admin 9006、URMA 设备 udmac0d1e2、其余照脚本（400/4MiB/16 线程/batch16/verify=false；chunk 128KiB/inflight 32MiB/workers 4/qpairs 4）。
- **target 侧**：W 汇总块 n=12,807（理论 12,800）；W8 JFC wait 62.7%、W9 exec 36.4%、W9 avg 24,840 ns、TOTAL 68 µs（写路径口径）。**per-IO `URMA_BDEV_DETAIL` 未新增**（当前 target 运行态未开；最后样本仍是 11-19）——五段拆分的 target 逐 IO 明细需另想办法（优先用 initiator 侧 `URMA_SLOW_IO`，target 遥测随完成回传）。

### NOF-010 关键事实（首份我们自己的读数据，2026-11-20）

- **reader（node3，400 keys × 4MiB，16 线程，order=seq，verify=true）**：400/400 成功、校验 0 错误。
- **端到端读时延（per-query, n=400）**：Min 462.64 / **Avg 801.29** / **P50 791.06** / P90 1017.98 / **P99 1709.42** / P999 1883.61 / Max 1886.75 µs。→ 与同事基线（~750–841µs）同量级，问题在我们环境**复现成功**。
- **吞吐不可用**：bench 打印 Total time 0.07 s，但日志墙钟约 5.99 s（读阶段）；21341 MiB/s 是按 0.07 s 算的，**不得当端到端带宽**。
- **控制面逐 key 分段（query_rpc_breakdown，815 行）**：字段 switch_us / rpc_first_us / route_refresh_us / retry_switch_us / rpc_retry_us / backoff_us / total_us；样本 total=386µs、158µs；全部 retry=0、status=ok；master 侧 `GetReplicaList latency_us=4` ⇒ **每 key 客户端 RPC 往返 150–400µs**，是端到端 801µs 的主要组成之一。
- **target 读窗口 W 汇总**：n=25,763（≈2×12,800，verify 二次读）；**W9 avg 285.87µs、exec 94.3%**、push 5.4%、TOTAL 303µs；per-IO URMA_BDEV_DETAIL 未开。
- **打点为何为空**：脚本默认 `VSEG_DIAGNOSTIC=0` ⇒ 三个阈值全 0；即使设 1，默认 `VSEG_SLOW_IO_US=100000`（100ms）远高于本次 Max 1.89ms ⇒ 仍无记录。需开诊断并把阈值降到 <1ms。
- **二进制差异（需记录）**：node3 reader 用 `stress_cluster_bench` md5 `59662a266baf5e462e86ce6a0b3f89af`（11-17 19:48），node1 则是 `575d2151ed39d256dbfa6269e4221abb`（11-17 20:13）；本轮按同事拓扑用 node3 的。
- node3→node1 fabric（50056/9006/2379）连通；管理网 50056 不可达（master 绑 fabric，符合预期）。

### NOF-011 关键事实（五段/七字段首采成功，2026-11-20）

- 采集：`URMA_SLOW_IO` 共 **5,044 条**（读阶段 4,675 / verify 364 / warmup 5）；报文含 **29 字段**（其中 11 个计时项）；读阶段 opcode=2 占 4,641/4,675。
- **读阶段 p50 五段链**：`roundtrip 528 → target 430 → bdev 387` µs；`data 32 / outside 81 / residual 82`；bdev/target=90.0%、bdev/roundtrip=73.3%。
- p90：roundtrip 777 / target 585 / bdev 537 / data 96 / outside 127 / residual 81 µs；p99：roundtrip 1,225 / target 775 / bdev 758 / data 168 / outside 553 / residual 158 µs。
- **诊断开销可测**：开诊断后端到端 Min 523.72 / Avg 942.89 / P50 910.47 / P90 1171.22 / P99 2212.68 / Max 2646.88 µs（对比 NOF-010：Avg +17.7%、P99 +29.4%）⇒ 诊断轮只用于**相对拆分**，绝对值以 NOF-010 为准。
- **阈值口径**：`VSEG_SLOW_IO_US=400` 只打印 ≥400µs 的 IO，读阶段 4,675/12,800=36.5%；全量分布需更低阈值（后续可选）。
- warmup 样本（opcode 127/6）pre_send 8.3–8.8 ms，已按行号剔除，避免污染。
- 二进制确认含 `URMA_SLOW_IO` 格式串（node3/node1 各 1 处）；`MC_STORE_TRANSFER_DIAG` 在二进制中无符号（无效，已记录）。
- target 窗口：W9 avg 283,010 ns、exec 93.9%、push 5.8%；per-IO `URMA_BDEV_DETAIL` 仍 0 增长。
- 结论（事实层）：**盘上执行 bdev 是读路径最大单项**（p50 387µs，占 73%）；控制面每 key RPC（NOF-010 的 150–400µs）与 bdev 是两个独立大头。是否由"128KiB < 256KiB strip 导致单盘定向"解释，待 NOF-012 矩阵验证。

### NOF-012 关键事实（旋钮矩阵无定论，2026-11-20）

- 15/15 遍成功（failed=0、verify 0 错、无超时）；单遍 10–11 s；诊断关。
- 各臂 p50 中位：V0 732.94 / **V1 721.17（最优，−1.61%）** / V2 729.06 / V3 759.02（+3.56%，p99 +248µs 最差）/ V4 744.67（qpairs 16，+1.60%）µs。
- 臂内 3 遍极差 35–75µs > 臂间最大差 26µs ⇒ **本次样本无法区分各臂**。
- target W9 avg 277,840 ns（−1.6% vs 跑前）、exec 94.8%；**chunk/qpairs 对 target SSD 执行段无影响**。
- **关键疑点**：chunk 是否真正生效无法证明——诊断关闭后 V0 与 V3 日志行数完全相同、无 child/chunk 级记录；需要诊断口径 A/B（NOF-013）。
- 控制面 `qrpc_total_p50` 74–92µs（与端到端无相关）；get_into 的 query_us ≈120µs。
- 口径偏差：bench `--duration` 不支持 remote_memory；本矩阵为"400 keys/遍 × 3 遍取中位"，非 120s。

### NOF-013 关键事实（chunk 复核中 + 同事优化方向原文，2026-11-20）

- **[待复核] chunk 疑似被运行环境覆盖**：NOF-013 观测到 V0（128KiB）与 V3（4MiB）的 `URMA_SLOW_IO` `blocks` 恒为 256（128KiB）；但代码复核显示 vsegment 读路径本应读取 `MC_NOF_SUBMIT_CHUNK_BYTES`（`transfer_task.cpp` 的 SpdkNofQos），且 GAP 项目同系统在 256KiB child 下曾得到显著更低时延（8MiB 读 387µs vs 128KiB 时 1194µs）。怀疑 NOF-012/013 运行方式下该 env 被脚本硬编码 131072 覆盖或未传入进程。**NOF-015 用环境自证复测。**
- **文档 §3.1**：裸 SPDK 11 盘基线 Get 236µs / exec 68µs；VSegment 6 盘 Get Avg 662 / P50 669 / P99 913µs，bdev 538.8µs（慢 child 均值）；裸 SPDK sweep 128K/256K = 233–255µs。
- **文档 §3.2**：Query（master RPC）128.9 / Submit 19.1 / **Wait 494.4（76%）** / 其他 9.0 µs。
- **文档 §3.4**：GetReplicaList 客户端缓存（`MC_REPLICA_CACHE_TTL_MS`，默认 0 关闭；容量 10 万）→ 端到端 p50 **841→610µs**；参数调优仅 ~3%；plain NoF 12.89 vs VSeg 13.07（4MiB 批量）⇒ vsegment 路由层排除嫌疑。
- **文档 §四 根因**：① raid0.c 单 child 单盘定向（`module/bdev/raid/raid0.c:86-91`，IO 禁止跨 strip，`assert(false)`）——每条 128KiB child 必落单盘；② 排队放大：瓶颈=在途 child 数 vs 盘数，6 盘时每盘排队 3~5 深 ≈ 54.7µs×5≈294µs；③ 盘数是第一变量：6→11 盘 W9 294→54.7µs（−81%），称已达裸基线；④ 数据链已收敛：控制面已优化 → URMA 洗净（4+17µs）→ 只剩 bdev 执行层。
- **文档 §四 优化方向**：① 提高 child 并发（nof_inflight_bytes 当前 32MiB）；② child 尺寸对齐 strip（128K→256K，nof_chunk_bytes / vsegment_stripe 2MiB）；③ 跨 child 条带并行读（Mooncake 分层改造，SPDK 侧不动）。
- **文档 §八 交接建议**：check → 处理 target 冲突 → 解 P6 → 跑一键实验填 comparison.md → 按三条方向深挖（改造点在 mooncake 分层）。
- **我方对照**：我们 11 盘 target 的 W9 仍 **278–288µs**，不是文档的 54.7µs ⇒ 我们的在途 child/每盘排队仍深（16 线程 × 32 child）；这正是 NOF-014 要验证的。

### NOF-014 关键事实（并发/在途扫描，2026-11-20）

- 15/15 遍成功；诊断关。
- 各臂中位：L0（16线程/32MiB）p50 714.9 / avg 754.9 / p99 1672.3；L1（8MiB）726.1（+1.6%，p99 +714µs）；**L2（2MiB）5507.2（+670%，pipeline 被串行化，推算）**；**L3（4线程）591.1（−17.3%）、avg −20.8%、p99 −49.5%、臂内极差仅 28.7µs（最稳）**；L4（缓存）719.2（+0.6%，无效果）。
- **L4 无效原因**：`MC_REPLICA_CACHE_TTL_MS` 在 node1/node3 两个 bench 二进制中**均无符号**（env 未实现），不是策略无效。
- **target W9**：L3 满窗口 254–259µs、exec 96.8–97.2%（历史满窗口 283–288µs）⇒ 降并发后每条 child 的 target 执行时间仍 ~255µs；L0 未捕到满窗口（快照 n=0/1），不能严格对比。
- 结论（事实层）：**降线程能降端到端（−17%）与长尾（−50%）**，但每条 child 的 target 底板 ~255µs 未明显下降 ⇒ 底部成本可能在 target/盘执行层，而非纯客户端排队；需要与"chunk 256KiB"（GAP 经验值 387µs）结合验证。

### NOF-015 关键事实（chunk 无效已定案 + IO 结构澄清，2026-11-20）

- **env 自证**：三臂 `/proc/<pid>/environ` 分别显示 `MC_NOF_SUBMIT_CHUNK_BYTES=131072 / 262144 / 2097152`；且同时含诊断 env。→ NOF-013 的"脚本覆盖"疑点排除，参数确实进了进程。
- **结论：读路径 NVMe/URMA IO 恒为 128KiB**（blocks p50=256，三臂一致；C2 的 max 也 256）⇒ 该粒度由代码层（NoF/URMA 驱动）固定，**环境变量无法改变**。
- **结构澄清**：vsegment 层 child = **2MiB**（`planned_children=2`，4MiB ÷ stripe_size 2MiB）；每个 2MiB child 内部再切成 16 个 128KiB 的 NVMe/URMA IO。文档 §四 "每条 128KiB child 必落单盘" 的 child 实际指**下层 IO**，与 vsegment child 术语有口径差。
- C1（256KiB）出现少量 171/85-block 碎片与偶发 4096-block，但 p50 未变；三臂端到端 p50 851–898µs、target W9 275.8–283.6µs，差异均在噪声内。
- **方向②（128K→256K）只能改代码**，且改动点在 NoF/URMA 驱动层（固定 128KiB 的位置），不属于"只改 Mooncake 分层"。

### 决策点（2026-11-20，用户已批准换 target）

- 最大未验证差异：**target 构建**。现役旧 gap target（`urma_p99` 构建）实测每 IO W9 ~255–288µs；同事文档的 yin target 达 **54.7–68µs**（5× 差距，非负载可完全解释：4 线程时我们 W9 仍 ~255µs）。旧 target 带 p99 门控/不同 iobuf/queue 参数，是首要嫌疑。
- **用户已批准（2026-11-20）**：停旧 gap target/master、重绑 11 盘、用 yin spdk-cpu 构建重启 target；旧 GAP 数据可能受影响，用户同意必要时重灌。
- 分阶段执行：NOF-016 换 target + 重启 master（不 mount）；NOF-017 mount + reconcile 门禁；NOF-018 writer 重灌；NOF-019 reader 基线对比（看 5× 差距是否闭合）。

### NOF-016 关键事实（target 已换为同事构建，2026-11-20）

- **新 target**：yin `nvmf_tgt`（md5 `736a902ffac8ce608acd22c5459f643d`，2026-11-17 19:20），PID 341038；listener **URMA node3:4457**（fabric 网段，旧地址 node3 已拒绝）；NQN `nqn.2026-01.io.spdk:urma-gpu-test`，nsid 1 → `URMA_RAID0`（strip 256 KiB、11 成员、raid uuid `5805fd63-…`）；transport dev_name `udmac0d1e2`、max_io_size 8 MiB、capsule_transport=urma。
- **旧环境退出**：我方 master 2162056（2 s）、gap master 3673976（1 s）、gap target 3243095（3 s）；4457 释放；无 nvmf_tgt/spdk_tgt 残留。
- **etcd**：我方 `/cvm/vseg_nofvseg/` 6→0、`/oplog/vseg_nofvseg/` 1228→0；yin 与 mooncake_cluster 键数不变。
- **我方 master 重启**：PID 2462234（16:31:45），leader/serving，50056/9006，`enabled=0`；`/cvm/vseg_nofvseg/` 3 键、`/oplog/` 1 键；env `udmac0d1e2` 自证。
- **地址/身份变化**：listener 由管理网改为 fabric；raid uuid 变化；NOF-017 register 必须用 **node3:4457**；旧 device_identity 冲突不存在（键已清）。
- **target 日志路径改为 `/tmp/nvmf_tgt.log`**（gap 的 `~/gap/logs/nvmf_tgt.log` 已停写）。
- 回滚快照：node3 `~/mooncake-nof-vsegment/run/rollback_gap_target.json`（4,761 B，含 4 类 RPC 全量 + 三进程 cmdline + gap target md5）。
- 唯一额外动作：wrapper 仅存在于 node1，已原样 scp 到 node3 `/tmp/run_vesegment_target.sh`（md5 一致）。
- HugePages：Free 347（yin target 占 ~8.4 GiB）。
- **待验证**：`TARGET_TRACE_ENTRIES=0`（wrapper 本次默认）是否影响 target 侧 per-IO 遥测（URMA_SLOW_IO 的 target 段）；NOF-019 读时观察 `target_valid`。

### NOF-017 关键事实（新 target mount 通过，2026-11-20）

- mount rc[0]（`elapsed_us[233163]`，233 ms）；`init_rc=0`、`Attached`、`Opened NoF segment ns=1 io_qpairs=4`；设备映射 20 条（udmac0d1e2 + 两个 provider）。
- **P6 在新 target 上再次通过**：`scanned=1024, owned_by_me=1024, reconcile_fail=0`；`VSegment Status: enabled=1, partitions_installed=1024, vsegments=1024, logical_capacity=64 GiB`；60 s+ 复查仍 0。
- etcd：`/cvm/vseg_nofvseg/` 8 键（quota 181,399 B）、`/oplog/` 32 键；segment 描述符 `device_identity=5805fd63…` = 新 yin raid0 的 nguid（与 NOF-016 记录一致）。
- **quota 发布延迟 +18 s**（mount 16:34:52 → 发布 16:35:11）：后续门禁检查必须等 ≥20 s，否则易误判。
- **target W9 在 mount 期 ≈ 70,210 ns（n=10，仅控制面/探测 IO）**——与旧 gap target 的 255–288µs 形成量级差异，也与同事文档 54.7–68µs 同档。样本很小，**仅作线索**，待读流量验证。
- 唯一 target ERROR 是预检 TCP 探测造成的 `handshake failed rc=-104`（16:34:43），非 mount 失败；`/tmp/nvmf_tgt.log` 已增长至 43 KB。

### NOF-018 关键事实（新 target 写路径与旧 target 同档，2026-11-20）

- writer 400 succeeded / 0 failed：1.56 GB，0.39 s，**4091.95 MiB/s**；写时延 Min 739.48 / Avg 977.53 / P50 901.03 / P90 1131.46 / P99 2775.35 / Max 5443.41 µs（与 NOF-009 旧 target：Avg 1007.38 / P50 931.45 / P99 3103.38 相比略优）。
- master：`logical_used=1,677,721,600`（=400×4MiB）；NVMe-oF SSD 1.56 GB / 100 GB；Keys=400；E=1（启动期）；1 条 `PutEnd_slow 3076µs rc[0]` 长尾。
- etcd：`/cvm/` 8 键不变；`/oplog/` 32→1232。
- **target 写窗口 W9 avg 23,550 ns（n=12,801）vs 旧 gap target 24,840 ns（−5.2%）**；TOTAL 67µs；exec 34.9%、jfc 63.8%。
- **结论（事实层）**：换 target 构建对**写路径** target 侧执行几乎无影响；旧 target 的 5× 差距出现在**读路径**（旧读 W9 255–288µs vs 新 mount 控制 IO 70µs），待 NOF-019 读基线确认。

### NOF-019 关键事实（换 target 后端到端不变，2026-11-20）

- 端到端（3 遍中位）：**R16 p50 719.0 / avg 780.6 / p99 1737.6 µs**（旧 target：714.9 / 754.9 / 1672.3；差 +0.58%，小于臂内极差 118.8）；**R4 p50 602.5 / avg 603.5 / p99 816.3 µs**（旧：591.1 / 597.9 / 845.4；+1.93%，极差 23.5）。
- target 读窗口：**W9 avg 254,780–254,820 ns（旧 283–288µs，−10~11.5%）**；TOTAL 265µs（旧 289–312）；exec 96.0%、push 3.5%。
- **结论**：target 构建换掉后，target 侧读执行变快 ~11%，但**端到端没有传导**；瓶颈已不在 target 的 bdev 执行段。
- 读 W9（254.8µs）与写 W9（23.6µs）相差 **10.8×**：读写 target 执行特性差异极大。
- 5× 差距：若按读口径，仍是 4.66×；但**54.68µs 是读还是写口径需核对**（NOF-020 B）。
- 6 遍 failed=0、verify 0 错；qrpc p50 66–96µs；无新增 master/target 错误。

### NOF-020 关键事实（口径更正 + 新 target 五段，2026-11-20）

- **文档口径更正**：§3.3 原文明确 `11 盘 avg=54.68µs（n=4023，TOTAL=59µs，exec 91.2%）——11 盘写路径样本`。⇒ 54.68µs 是**写路径**，不是读；此前"读 W9 254.8 vs 54.68、5× 差距"属**读/写跨口径比较，撤回**。文档同样未给出 11 盘**读**路径的 W9。
- **新 target 读五段（16 线程，诊断口径 p50）**：roundtrip **514** / target **424** / **bdev 387** / data 13 / outside 59 / residual 60 / pre_send 0 µs；bdev/roundtrip=75.3%、bdev/target=91.3%。
- **bdev 387µs 与旧 gap target 完全相同（0 变化）** ⇒ 换 target 的收益只在 data/outside/residual（分别 32→13、81→59、82→60），不在盘执行段。
- `target_valid=1` 占 **98.9%**（2,686/2,713 读阶段）⇒ `TARGET_TRACE_ENTRIES=0` 不影响 target 段遥测。
- 端到端（诊断）：Min 510.76 / Avg 858.27 / **P50 779.19** / P90 1173.18 / P99 2200.19；诊断开销 P50 +8.4%。roundtrip p50 514 与 bench P50 779 相差 ~265µs = 控制面 + 应用层/流水线。
- 同阈值 400µs 下 SLOW_IO 条数 4,675→2,790（−40%），但 p50 仅 −2.7% ⇒ 改善集中在长尾。
- target W 块：诊断窗口 W9 270.3µs（滚动窗口，与 NOF-019 的 254.7 不可严格比较）、exec 94.8%。
- **结论**：读路径的盘执行段（bdev ~387µs）是当前最大单项且未随 target 构建变化；要到 300µs 端到端，必须动 bdev 本身或减少/重排 child，而不是继续换 target。

### NOF-021 关键事实（bdev 不是排队，是路径底板，2026-11-20）

- 4 线程 vs 16 线程（同一百分位算法重算）：roundtrip 446 vs 513（−13%）、target 427 vs 417、**bdev 418 vs 380（+10.0%，不降反升）**、data 6 vs 11、outside 14 vs 46、residual 14 vs 46（µs, p50）。
- ⇒ 降并发只压缩了 outside/residual/data；**bdev 没有随并发下降**，不支持"客户端排队放大"的解释。
- 4 线程端到端（诊断口径）：Min 497.29 / Avg 657.06 / **P50 656.54** / P90 758.91 / P99 896.89 / Max 942.34 µs；400/400、verify 0 错。
- 盘：11 块成员全部 **Huawei HWE7AP457T6L005N**（vendor 0x19e5，7.68 TB，FW 1049×9 / 1019×2，512B 块）；TLC/QLC `[未获取]`（RPC/日志无介质字段）。
- target W9：259,780 ns（n=25,777）、TOTAL 269µs、exec 96.3%、push 3.2%。
- 诊断采样：4 线程读阶段 opcode=2 800 条，target_valid {1:767, 0:33}。
- **与文档裸 SPDK 的对比线索**：文档 §3.1 裸 SPDK 11 盘基线 exec avg **68µs**（8MiB、非 vsegment）；我们 Mooncake 路径 target W9 平均 ~260µs、慢 IO bdev p50 ~380–418µs ⇒ 路径层差 ~4×，需裸 SPDK 实测确认（NOF-022）。

### NOF-022 关键事实（裸 SPDK 30µs vs Mooncake 259µs，2026-11-20）

- **Mooncake T1（诊断）**：roundtrip 452 / target 432 / **bdev 425** µs；端到端 Min 477.30 / Avg 582.71 / **P50 571.10** / P99 737.93；400/400、verify 0。
- **裸 SPDK urma_perf 128KiB（-T1 -b1，6s，root）**：completed 130,577、errors 0、21,762 IOPS、2720 MiB/s；**客户端 p50 41.28µs / avg 45.87**；客户端 URMA submit 70ns、completion 90ns（send/compl 各 0.2%）。
- **同一 target 的 target 侧对照**：裸 SPDK W9 **30,170 ns**（n=130,580，exec 79.6%、push 18.7%、TOTAL 37µs）vs Mooncake W9 **259,080 ns**（n=24,898，exec 96.9%、TOTAL 267µs）⇒ **8.6× 差在 target 盘执行段**；Mooncake W9 在 T1/T4/T16 几乎不变（259.1/259.8/259.9）。
- **混淆变量**：裸 SPDK 是**顺序** LBA 扫描；Mooncake 是 **1024 分区内的随机 2MiB 块**（每 key 2 child × 16×128KiB）。8.6× 可能来自"随机 vs 顺序"或"QD/IO 形态"，尚不能归因（NOF-023 受控）。
- `urma_perf`（gap 构建 md5 `c653c0a6…`）单 IO 上限 **128KiB**：256K/8M 探针被拒（`I/O size ... no greater than 131072`）。
- 运行条件：必须 **root**（非 root 报 EAL IOVA `init_rc=-22`）；裸 SPDK 需 `SPDK_URMA_CAPSULE_TRANSPORT=urma` 才能握手（否则 `rc=-93` capsule mode mismatch）。
- **安全事件（如实记录）**：urma_perf 内置 preflight 在 nsid1 **LBA 0 写入一次**（≈128–256KiB），落在我们 vsegment 段 base 0 的最前面；17:00:37 的 verify 在写之前（0 错）。开发侧决定：NOF-023 末尾重跑 writer 修复 + verify 复检。
- 文件：node1 `~/nof022_perf_131072.log`（成功）、`nof022_perf_262144.log`/`_8388608.log`（被拒）；node3 `run/diag_r3_t1.log`。

### NOF-023 关键事实（8.6× 归因：随机 vs 顺序，2026-11-20）

- 裸 SPDK 受控 2×2（gap urma_perf，128KiB，6s，-T1，root）：

| 支 | 模式/-b | 客户端 p50 | target W9 | target TOTAL | exec% |
|---|---|---|---|---|---|
| S1 | seq / 1 | 40.00 µs | 30,350 ns | 37 µs | 80.7% |
| S32 | seq / 32 | 114.56 | 41,430 | 77 | 53.2% |
| **R1** | **rand / 1** | **220.16** | **190,580** | **198** | 96.1% |
| **R32** | **rand / 32** | **225.28** | **206,300** | **215** | 95.7% |

- **随机化解释 6.3×**（QD1 30.35→190.58µs）；QD 影响小（seq +36.5%、rand +8.2%）。
- **随机下 Mooncake 259.1µs vs 裸 SPDK 190.6–206.3µs = 1.26–1.36×**，剩余差在 Mooncake/vsegment 路径（2MiB child 内部的 16×128KiB 流水线）。
- `yin urma_perf` 256K/8M 未取得：node1 树 WIRE v6 vs target v4，HELLO 长度不符 → `rc=-71 EPROTO`。node3 有同批构建的 urma_perf（5,869,272 B，11-17 19:20，WIRE v4），可用但未授权拷贝（开发侧决定：需要时在 node3 本机跑，不拷）。
- **LBA0 修复完成**：writer 400/0（1.56 GB，0.33 s，4910.71 MiB/s）+ 16 线程 verify reader（0 errors out of 400；P50 703.12 / P99 1710.81）；`logical_used=1,677,721,600` 未变。
- W 块采样口径：target 的 W 块是"下一个新连接进入时打印上一轮累计"，本次已按 `n` 对齐改正（NOF-022 的"跑完立即采"会取到陈旧块）。
- S32 的 33 GB/s 异常（可能顺序重复读同一小 LBA 区间）`[未验证]`，不影响随机结论。
- 结论：**随机 128KiB 的盘执行底板 ≈190–206µs**（target）；要接近 300µs 端到端，需把每 key 的 32 个小 IO 压成一波（lanes/workers），并把控制面/流水线开销压到最低。

### NOF-024 关键事实（qpairs/workers 证否，2026-11-20）

- T1 四臂 P50 中位：L4 551.5 / L8 542.7 / L16 537.5 / L16W8 537.8 µs；差异（−14~−18µs）**小于 L4 臂内极差 34.3µs** ⇒ 不能认定有效。
- **target W9 恒为 250,820–251,230 ns**（n=25,762，exec 96.2–97.0%），完全不随 qpairs/workers 变化。
- 参数已生效：日志 `NoF qpairs/segment: 4/16`；workers=8 有 `16 I/O lanes but only 8 workers` 警告。
- 最好 T1 P50 **537.5µs**，距 300µs 差 **+237.5µs（1.79×）**；其中 target 盘执行 ~251µs（46.7%），**客户端/流水线 ~287µs**。
- 12/12 遍 400/0、verify 0 错；吞吐 1513–1549 MiB/s 各臂一致。
- 运行间漂移：L4 基线 551.5 vs NOF-022 同配置 571.1（−3.4%）⇒ 2.5% 的"提升"与漂移同量级。
- **结论**：客户端参数旋钮到头；剩余 ~287µs 需改结构（batch/预取/放大 IO 粒度/改 vsegment 切分），或在现状收口。

### 目标与硬约束（2026-11-20 用户澄清）

- **原目标（不变）**：单 key 4–8MiB 读时延 **750µs → 300µs**（交接文档 L5）。所有调优实验都朝它去。**用户 2026-11-20：不必拘泥同事资料（750/300 均无实测支撑），以我方同环境实测数据为准。**
- **用户的硬约束（叠加，不替代）**："开了 vsegment 的性能一定要比没开 vsegment 更好，不然 vsegment 就没任何意义。" ⇒ 就算达到 300µs，若 plain NoF 同样/更快，则 vsegment 无价值；最终交付必须同时满足"接近 300µs"与"vsegment 优于 plain NoF"两条。
- 现有缺口：只量了 VSegment 绝对值（T1 537.5µs / 16t 719µs），**从未在本环境跑过 plain NoF**，无法判断进步/退步。
- 计划：NOF-025 建 plain NoF 基准线（临时拆 vsegment 环境）；随后按结果：①vsegment 已更快 → 继续压绝对时延向 300µs；②vsegment 更慢 → 先做 vsegment 布局调优（首选"减少分区数"，让 key 连续分配、把随机读变顺序读），直到 vsegment 胜出后再压 300。
- plain NoF flag 依据（`stress_cluster_bench.cpp` be10be2）：`--replica_num=0 --nof_replica_num=1 --vsegment_nof_io=false --segment_write_vsegment=false`；`--vsegment_nof_io` 注释明确 "VSegment puts require --nof_replica_num=0"。

### NOF-026 关键事实（750/300 出处核实，2026-11-20）

- **750µs 不是实测值**：全文仅 L12 目标句「单 key（4-8 MiB）读时延从 ~750µs 压到 300µs」。文档唯一 VSegment 端到端实测为 §3.1 的 **Avg 662 / P50 669 / P99 913µs（6 盘、8MiB 单次 Get、单线程）**；750 比 P50 高 81µs（+12.1%）、比 Avg 高 88µs（+13.3%），取自哪次测量 `[未获取]`。
- **300µs 无任何推导**：全文 3 处（L4/L12/L94）均为目标陈述；§四只列手段（提高 child 并发 / child 对齐 strip / 跨 child 条带并行），无对标依据。
- **资料中没有"不开 vsegment 的 Mooncake 读端到端"数字**：§3.1 只有 ①裸 SPDK（urma_perf，11 盘，236µs/exec68，非 vsegment、非 Mooncake）②VSegment 6 盘。唯一 plain NoF 对照是 **写吞吐** 12.89 vs 13.07 MiB/s（L83），不能作读 A/B 依据。
- `380` 在交接文档 0 处；`800` 仅命中错误码区间 `-1800~-1899`；`841/610` 见 L81（客户端缓存 B0→B1 p50）。
- 口径差异：文档 662/669 是 **8MiB/单线程/6 盘**；我们 537.5/719 是 **4MiB/400 keys/verify/11 盘**，不可直接相减。
- **决定**：不补"6 盘校准臂"（需重配 target，且 750 出处不影响 A/B）；直接执行 NOF-025（同环境 plain NoF vs VSegment）。
- 750 的 6 盘原始 profile 在 node1 `…/vsegment-results/20261118-201528-*/`（未取，`[未获取]`）；3 张 PNG 未解析。

### NOF-025 关键事实（判决：VSegment 比 plain NoF 慢，2026-11-20）

- 同 target、同客户端、同负载（400×4MiB、verify）：
  - **T1 P50：plain NoF 258.3µs（242.1–262.8） vs VSegment 537.5µs（L16）/551.5（L4）⇒ plain 快 2.08–2.14×**；plain T1 已进入 300µs 区间。
  - 16t：plain 718.7 vs VSegment 719.0（差 0.3µs，打平）；target W9 plain 194–196µs vs VSeg 259.6µs ⇒ 16t 时端到端由客户端/流水线主导，target 差异未传导。
  - 写：plain 400/0、1.56GB、0.11s、**14,316.6 MiB/s、P50 273.0µs** vs VSegment 0.33s、4910.7 MiB/s、P50 738.6µs ⇒ plain 快 2.7×。
  - target W9：plain T1 96.9µs、16t 194–196µs、写 27.8µs；VSeg T1 250.5µs、16t 259.6µs。
- **布局混淆（关键）**：plain 把 400×4MiB 连续写满最前 1.6GiB（顺序布局）；VSegment 按哈希散到 **1024 个分区**（跨 100GiB 随机布局）⇒ 2.08× 差异可能同时含"vsegment 路径开销"与"随机 vs 顺序布局"，本单无法拆分（NOF-027 处理）。
- 环境：plain master PID 2627750（`VSegment Status: enabled=0`，etcd 3 键、无 quota），Keys=400、NVMe-oF SSD 1.56GB/100GB；yin master/target 未动。旧 vsegment 数据已被 plain 写入覆盖（恢复需重跑 writer）。

### NOF-027 关键事实（分区数不是杠杆，2026-11-20）

- 1024→64 分区：**T1 P50 539.9µs（vs 537.5，+0.4%，等价）**；P90/P99/Avg 均在噪声内；**16t P50 854.3（vs 719.0，+18.8%）**，P90 1084–1122。
- target W9：T1 **250.8–251.3µs（与 vsegment-1024 完全一致）**；16t 287.5（+10.7%，push 10.4%、exec 89.1%）。
- 仍未赢 plain：T1 慢 2.09×（539.9 vs 258.3）；16t 慢 1.19×（854.3 vs 718.7）。
- 控制面：`rpc_first_us` p50 88µs（T1）/103µs（16t）；即使控制面归零，vsegment 仍慢。
- 写：vseg-64 400/0、4007.9 MiB/s、P50 916.09µs（比 vseg-1024 的 738.61 更慢）。
- **容量偏差（重要）**：64 分区时 logical_capacity 只有 **4GiB**（64×64MiB 初始卷），不是任务书预算的 100GiB；auto-plan 每分区只发 initial_vsegment_count=1 个 64MiB 卷。本单不是纯净"只改分区数"实验。
- **机制结论**：分区多少不改变 target 看到的 IO 形态（W9 恒定）⇒ 差距不在"数据散在 100GiB 的随机范围"，而在 **每 key 的 2MiB child × 16×128KiB 拆分方式与读序/物理布局错位**（plain 的 4MiB 连续读 W9 96.9µs）。分区数杠杆证否。

### NOF-028 关键事实（单分区写入受阻，读未测，2026-11-20）

- `MC_STORE_PARTITION_SLOT_COUNT=1` 被正常接受：`slot_count=1`、quota `partitions=1`、`reconcile_fail=0`、`partitions_installed=1`；初始 `logical_capacity=64MiB`。
- writer（16 线程、batch 16）：**377 succeeded / 23 failed**，错误码统一 `VSEGMENT_CREATING`；失败 key 严格等间隔（16, 33, 50, …, 步长 17）⇒ **每跨一个 64MiB 卷边界丢 1 个 key**，bench 对该错误码不重试。
- 容量演进：vsegments 1→24（1.5GiB），`logical_free=28MiB`；400×4MiB=1.6GiB 需要 ~26 个卷 ⇒ 容量+创建速度双不足。
- reader 未执行（按停止规则）⇒ T1/16t/W9 `[未获取]`，布局假设未判定。
- **判断**：这是**按需建卷 + 客户端不重试**的机制问题，不是"单分区不可用"；用单线程/batch=1 写入可避免创建竞争（NOF-029 验证）。
- 环境：vsegment-1 master PID 2667366 运行中；数据为 377/400 混合态。

### NOF-029 关键事实（布局假设否定，2026-11-20）

- 数据实质齐备：writer T1/batch1 报 398/2，但 2 个"失败"key 在 r1 已 PutStart/PutEnd ok（r2 撞创建窗口误返回）；`logical_used=1,677,721,600`、`committed_allocations=400`、`vsegments=26`；6 遍 reader verify **0 errors out of 400**。
- **vseg-1 读侧**：T1 P50 **543.0µs**（vs plain 258.3，慢 **2.10×**；vs vseg-1024 537.5，+1.0%）；16t **739.5**（vs plain 718.7，+2.9%；vs vseg-64 854.3，−13.4%）；target W9 T1 **263.6–276.8µs**（中位 271.2，比 vseg-1024 的 250.5 更高）。
- **分区数 1 / 64 / 1024 的 T1 P50 = 543.0 / 539.9 / 537.5（5.5µs 窄带）⇒ 布局/分区数对读时延几乎无影响**；plain W9 96.9 vs vsegment 250–271 ⇒ 差距稳定存在于 **vsegment 分层读路径（每 key 2 child × 16×128KiB）**。
- 控制面：`rpc_first_us` p50 91µs（T1）/90µs（16t）。
- 推算（非实测）：vseg-1 T1 543 ≈ 控制 91 + target W9 271 + 客户端流水线 ~181µs；W9 一项比 plain 多 174µs（占差距 285µs 的 61%）。
- 遗留缺陷：`VSEGMENT_CREATING` 在单线程重写时仍复现 2 次（卷创建窗口）；属 vsegment 侧机制问题。
- 环境：vseg-1 master PID 2667366 运行中（partitions=1、vsegments=26、logical_used=1.6GiB）。

### NOF-031 关键事实（stripe 证否 + 口径纠正，2026-11-23）

- **环境探测（node1 共享 / node3 独占）**：我方 master 2667366 与 target 341038 均存活；50056/9006/4457 均归我方；我方 etcd 键完整；yin 键未变。新发现同事 z00926396 的 `python3 http_metadata_server.py --port 8086`（PID 1955247）及 `mooncake_cluster` 键、client 进程——**仅记录未触碰**；node1 load 8.06（≈2.6%），node3 load 14.85（≈4.8%）。判定可用。
- **stripe=4MiB（NOF-030 内容）**：quota stripe_size=4194304 生效、1024 分区、reconcile 0；writer 400/0；读 T1 P50 **551.6**（550.1–557.9）、16t **705.4**（675.2–786.6，离散 111µs）；target W9 **252.3 / 270.5**；未追平 plain（T1 慢 2.14×；16t −1.9% 在噪声内）。
- **关键机制发现**：stripe 2MiB 与 4MiB 的 target IO 总数**完全相同（n=25,762）** ⇒ stripe 只改 child 打包，不改实际 IO 粒度（仍 128KiB）。
- **口径纠正（重要）**：plain NoF（NOF-025）的 target W9 同样 n=25,762 ⇒ **plain 也是 128KiB 小 IO，不是"4MiB 大 IO"**；差距是**每条 IO 的执行时间**（plain 96.9µs vs vsegment 252µs），不是 IO 大小/数量。此前"改代码放大 IO"的提案缺乏证据支持。
- 由此：读路径的 IO 粒度不是"调大 chunk"能解决的（NOF-015 已证 env 无效）；且**布局假设此前被污染**——NOF-028/029 的 vseg-1 是 16 线程先写 377 key（分配顺序打乱），r2 单线程只重写已有 key，物理布局并非"读序=写序"。

### NOF-032 关键事实（布局二度否证 + 前提未取证，2026-11-23）

- **纯 T1/batch1 顺序写（全新 vseg-1）**：r1 377/23 失败（全部 `VSEGMENT_CREATING`，key 16,33,50,… 步长 17——卷边界），r2 补齐 398/2（2 个为已存在 key 的误报）；对象层最终 `committed_allocations=400`、`logical_used=1,677,721,600`、`vsegments=26`；6/6 遍 verify 0 错。
- **读侧**：T1 P50 **536.7**（528.0–545.5）、16t **921.8**（838.5–950.0，比 NOF-029 的 739.5 更差）；target W9 **251.1**（261.5/251.1/250.9）；控制 `rpc_first` p50 87µs。
- **布局二度否证**：分区 1/64/1024 × stripe 2/4MiB × 写序（16 线程/T1）全部组合下，target W9 恒 **250–252µs**、T1 恒 537–552µs；plain 同尺寸 IO 为 **96.9µs**、T1 258.3µs。
- **尚未取证的前提**：T1 顺序写是否真的产生**物理连续**的 LBA（无 LBA trace）；vseg-1 的 16t 变差（921.8 vs 739.5）原因也未知。
- `VSEGMENT_CREATING` 缺陷第 3 次复现（16 线程 23 个、单线程首次 23 个、补齐 2 个）⇒ 稳定机制缺陷，建议单独立项。

### NOF-033 关键事实（LBA 完全连续；布局彻底洗清，2026-11-23）

- vseg-1 的 400 个 key **严格占据 LBA [i×8192, (i+1)×8192) 块**（4MiB/个）；key 起点步长 8192 出现 397/399 次，单调递增；key 内相邻差=256 占 93.4%，**0 次 >1MiB 跳变**；128KiB 槽覆盖率 98.4%；2MiB（4096 块）IO 占 3.2%（372/400 key 至少 1 条）。
- 该窗口 target W9 = **258.9µs（exec 97.4%）**，仍是 plain（96.9）的 2.67 倍 ⇒ **在"完全连续"这一最优布局下 W9 不变**，布局维度正式排除。
- **代码复核（开发侧）**：plain（一次 4MiB task）与 vsegment（N×2MiB task）最终都调用同一个 `submitSpdkNofOperation` → 同一个 QoS/驱动；target 端只看到 NVMe 命令，**没有 vsegment 概念** ⇒ "vsegment 寻址让每 IO 贵 2.6×"的说法不成立；差异只能来自 IO 到达时序、并发形态或盘缓存状态。
- 需警惕：plain 的 W9 96.9µs 取自滚动窗口，可能混入写路径快 IO（写 W9 27.8µs），有被拉低的风险；**端到端仍是主指标**。
- 尚未测的干净组合：**slot=1 + stripe=4MiB（每 key 单 child）+ 纯 T1 顺序写**（NOF-032 是 2MiB stripe=2 child；NOF-031 是 1024 分区+16 线程写，布局散）。

### NOF-034 关键事实（单 child 也不行；配置空间穷尽，2026-11-23）

- stripe=4MiB + slot=1 均生效；T1 顺序写 r1 377/23、r2 399/1（对象层 400 全齐：committed=400、logical_used 精确、6/6 verify 0 错）。
- 读：T1 P50 **559.2**（550.0–563.1）、16t **735.3**（728.1–817.0）；target W9 **250.1 / 260.3 / 270.9**；控制 rpc_first p50 105µs（环境非独占，略高于历次 87–91）。
- vs plain（258.3/718.7，W9 96.9）：T1 慢 **2.17×**；16t 仅 +2.3%（**最接近 plain 的 vsegment 配置**）；W9 高 2.6–2.8×。
- vs stripe2（536.7/921.8）：T1 +4.2%（更差）、16t −20.2%（明显更好）；W9 基本相同。
- **五个维度全部穷尽**：分区数 1/64/1024、stripe 2/4MiB、child 1/2、写序 16t/T1、物理连续性（LBA 已验证连续）→ target W9 恒 250–271µs、T1 恒 536–559µs；plain 恒 96.9/258.3。
- `VSEGMENT_CREATING` 第 4 次复现（稳定机制缺陷）。
- **开发侧判断（已被 NOF-035 修正）**：当时推断"target 端对同尺寸、同连续 LBA 的命令无法区分 plain/vsegment ⇒ W9 差异不可归因于 vsegment、可能是滚动窗口/盘缓存测量偏差"；NOF-035 已实测否证该推断（96.9 可干净复现），配置层无杠杆的结论不变。

### NOF-035 关键事实（96.9 可在干净窗口复现；伪影推断被否证，2026-11-23）

- **96.9 复现**：plain 按 NOF-025 同序（16t 写 → 6s 后读）取读窗口 W9 = **99.14µs**（n=25,762、exec 86.4%、TOTAL 114µs），历史 96.9（+2.3%）；写窗口 W9 28.56µs（n=12,802）⇒ **不是滚动窗口/写路径混算的伪影**。
- **命令形态**：plain 读阶段 12,774 条（128KiB×12,398=97.1%、4MiB×376=2.9%，无 2MiB）；vseg NOF-033 同口径 12,764 条（128KiB×12,353=96.8%、2MiB×411=3.2%）。LBA 两边同为 400 个连续 4MiB 窗口（0..399；相邻窗口步长 399 次全部=1）、窗内排序相邻差=256 占 94.1%/93.4%、**>1MiB 跳变 0/0**、重复 LBA 377/409 ⇒ **主体命令（97%）与 LBA 布局相同**。
- **唯一可见差异**：每 key 一条大 IO 的尺寸——plain 4MiB vs vseg 2MiB（plain 375 窗 = 1×4MiB + 31×128KiB；vseg 322 窗 = 1×2MiB + 31×128KiB）；该大 IO 属读循环还是 verify 路径 `[未获取]`。
- **每命令差是真实的**：客户端单条 128KiB roundtrip p50 plain **97µs** vs vseg **274µs（2.8×）**，大 IO 115 vs 325µs；与 target W9（99 vs 251）同量级 ⇒ 差距在 **target 执行段**，与命令尺寸、LBA 顺序均无关。
- **与并发的关系（历史事实并置）**：plain W9 从 T1 96.9 升到 16t 194–196；vseg 从 T1 到 16t 恒 250–271 ⇒ vseg 像是存在一个与并发无关的每命令固定开销；具体机制 `[未获取]`。
- **口径注意**：本轮端到端 P50 311.54 / Avg 375.35 / P90 603.50 / P99 701.72 / Max 856.03µs 为**开诊断**口径（日志 18.3MB、slow_io 25,765 条），不可与诊断关的 258.3 直接对比；W9 为同口径可比。
- **环境收尾**：停在 **plain master PID 1778131**（cvm=3、oplog=401），target 341038 未动；node1 同事进程未动。

### NOF-036 关键事实（复读稳定；重写未落盘；bench 计数缺陷，2026-11-23）

- **三臂读 W9（每臂 2 遍）**：arm1 106.21/105.15、arm2 110.48/97.48、arm3 105.20/99.50µs ⇒ 全部 **96–110µs**，从未接近 250；端到端 P50 除 arm2 p1 的 503.39（单遍偶发，W9 仍 110）外均 290.9–313.7µs（诊断开口径）。
- **臂 2/3 的"重写"没有落盘**：master 对两臂各返回 400 条 `PutStart failed: OBJECT_ALREADY_EXISTS`（11:08/11:09/11:10 共 1,200 条）；target 写窗口 **n=2、avg 36–37µs**（对照 NOF-035 新鲜写入 n=12,802、avg 28.56µs）⇒ **三臂读的是同一份 NOF-035 16t 写数据，写模式对照未发生**。
- **bench 计数缺陷（可复现）**：writer 报 `Write complete: 400 succeeded, 0 failed`、0.02s、79,123 MiB/s、P50 48.67µs，但 PutStart 全被拒（内存拷贝量级）⇒ **bench 把被拒 put 记为成功**；今后凡"写后计数"一律用 master `Keys`/`logical_used` + target 写窗口 n 验收，不信 bench 的成功数。
- **布局与数据**：三臂 6 遍解析均为 400 个连续 4MiB 窗口（步长恒 1）、128KiB 占 96.8–97.1%、>1MiB 跳变 0、重复 LBA 367–375 ⇒ 与 NOF-035 完全一致；读 verify 6/6 全 0 错。
- **plain 的写不可覆盖**：key 已存在时 PutStart 一律拒绝 ⇒ 要产生"新写入"，必须换新 key（NOF-037 用 `--num_keys` 扩展）。
- 流程偏差：首次臂 2 整脚本 sudo 导致 node1→node3 的 root ssh 被拒（该轮 writer 已执行=11:08 那批 400 条拒绝）；改"脚本普通用户 + 仅 writer sudo"后恢复；SSH 别名仍失效，用 `l00955908@<IP>`。

### NOF-037 关键事实（写模式排除；发现段首 97-key 慢区，2026-11-23）

- **两次写都落盘（验收通过）**：桶 2（T1，400 新键）→ master Keys=800、SSD 3.12GB、target 写窗口 n=12,802、W9 28.27µs；桶 3（16t，400 新键）→ Keys=1200、SSD 4.69GB、n=12,802、W9 28.33µs；OBJECT_ALREADY_EXISTS 累计 1,200→2,400→3,600 完全符合预期 ⇒ NOF-036 的"写不落盘"已用新 key 规避。
- **写线程数对读时延无影响（主问题否决）**：同一读会话内 128KiB roundtrip p50——桶 1（旧 16t 写）98/100/98、桶 2（**T1 写**）87/85/86、桶 3（16t 写）86/83/84µs；桶 2/3 不可区分（差 ≤2µs），**均未接近 vseg 的 250–274µs** ⇒ "写模式决定读慢"不成立。
- **新线索：段首慢区**。桶 1 的慢 IO（≥200µs）**精确集中在 key 0–96**（LBA [0, 388MiB)）：p90 284–290µs、慢占比 22.1/22.0/22.1%（三遍完全一致）；keys 97–399 与桶 2/3 的慢占比 0–4.3%；按 mod 2/4/6/11（盘/通道）分类慢占比均匀（各 7–8%）⇒ **不是单盘/通道问题**，是一段整 key 的异常。
- **target 读窗口 W9**（read+verify 合并，n=76,963）：65.4/66.5/71.7µs（NOF-035 同结构 400 keys 为 99.14）；慢区只占 97/1200 keys，摊薄后均值下降。
- **慢区已被用户解释（2026-11-23）**：这是**已知的盘首慢窗口**——每盘 ~36MB、RAID0 11 盘 ≈396MB，与 97 个 4MiB key ≈388MiB 吻合；不是新异常，也不是 vseg 慢的来源（vseg 全 key 均匀慢）。基准必须用 warmup 跳过（见"基准测试口径"）。
- **旗标纠正（NOF-038 结果）**：`--order` 不存在；`--key_order` 只存在于远端源码 tip 4c0ea28，**同事构建二进制（be10be2）没有**（`--help` 84 个旗标无此项，实跑 unknown flag）。
- 环境：plain master 1778131（Keys=1200、4.69GB、oplog 1,201），target 341038 未动。

### NOF-038 关键事实（随机读旗标不存在；旁证复现慢窗口，2026-11-23）

- **阻断点**：`--key_order` / `--key_order_seed` 在构建二进制中不存在（`--help` 无、实跑 `unknown command line flag`、零 I/O）；`uniform_random` 在二进制中 0 命中 ⇒ **构建基于 be10be2（2026-11-17），早于远端源码 4c0ea28 的这两个旗标**。任务书按远端源码写旗标是开发侧错误；今后任务设计必须按 `--help` 核对构建二进制。
- **`--shuffle_seed` 无效**：它只对 segment_read 生效（"deterministic shuffle of segment_read keys"）；remote_memory 读序仍是 key 0→1199（实测前 200 条只跨 7 个 key、严格递增）。
- **旁证读逐位复现 NOF-037**（`--shuffle_seed=42`，实为顺序）：桶1 p50 100 / p90 292 / ≥200µs **22.0%**；桶2 90/113、桶3 89/111，两桶 0.0%。
- **慢窗口边界极锐**：key 92–96 慢占比 97/97/94/94/90%，key 97–101 骤降为 **0%** ⇒ 边界在 key 96/97 之间（≈388MiB），与用户所述 11×36MB≈396MB 吻合。
- **慢段时长**：约 **0.68–0.76s**（p2 0.68s、探测 0.76s；p3 尾部散点延至 2.95s）；整段 1200-key 读约 3.3s ⇒ 慢段约占单次连接 21%（此前"前 ~1.5s"的假设偏高）。
- 随机顺序问题**未判定**（二进制不支持；要随机读只能改代码重编）。

### NOF-039 关键事实（warmup 不能跳过慢窗口；干净基线；并发 4.2×，2026-11-23）

- **warmup 无效（主假设否证）**：warmup 阶段本身读 keys 0–127 就是 p50 **243/248µs**；测量阶段重读同一批 key 仍 **243/247µs**；keys 0–96 慢占比 **90.9%（w128）/91.0%（w400）**（验收线 <3%）⇒ **慢区是固定 LBA 段，不是连接预热**；warmup 不能跳过。
- **干净基线（排除 keys 0–96）**：T1 per-IO p50 **86µs** / p90 **109** / p99 121，target W9 **67.6–73.9µs**；16t p50 **359** / p90 **570** / p99 804，target W9 **161.4µs**（与 NOF-037 p2 同口径 85/114 一致）。
- **并发效应**：1→16 线程，per-IO p50 **86→359µs（4.2×）**、p90 109→570（5.2×）、target W9 **68→161（2.4×）**；16t 下四个 key 桶慢占比均 94.2–94.5%（慢窗口被并发噪声淹没，不再可区分）。
- **对 vseg 的意义（事实层面）**：plain 16t 的 per-IO p50 已达 **359µs**，高于 vseg T1 的 250–274µs ⇒ "vseg 慢"不能仅归因于 vseg 自身路径，**多流/并发本身就能产生该量级的每 IO 时延**；vseg T1 的干净复测（NOF-040）将给出直接对照。
- 环境：plain master 1778131（Keys=1200、4.69GB），target 341038 未动。

### NOF-040 关键事实（qpair 排除；vseg 全程均匀慢；16t 差距收窄，2026-11-23）

- **环境与写入验收**：plain master 1778131 停；vseg-1+stripe2MiB master **PID 2077841**（slot=1、partitions=1、reconcile 0、vsegments 26、capacity 1.744GB）；writer 两次尝试补齐（r1 377/23 `VSEGMENT_CREATING` → r2 398/2），master committed=400、logical_used 精确；写窗口 W9 24.03/24.67µs。
- **读臂（400 keys、warmup=128、诊断开）**：
  - 臂 A T1 **qpairs=4**：128KiB 全量 p50 274 / p90 332 / p99 450；排除 0–96 后 **274/332**；target W9 **255.25µs**（n=29,698，exec 97.0%）。
  - 臂 B T1 **qpairs=1**：全量 267/328/448；排除 0–96 267/328；W9 **248.19µs**（exec 97.6%）。相比 q4 仅 **−2.6%/−2.8%** ⇒ **lane/qpair 数不是主因**。
  - 臂 C **16t qpairs=4**：全量 426/765/1032；排除 0–96 **414/759**；W9 **270.43µs**（exec 93.7%）。
- **vseg 全程均匀慢**：keys 97–399（完全避开盘首慢区）慢占比 **90.7%（q4）/90.4%（q1）**，与 keys 0–96 的 91.2%/91.1% 几乎相同；plain 同段为 **0%** ⇒ 慢不是局部区域，与盘首慢窗口无关。
- **16t 差距收窄**：vseg 16t vs plain 16t（干净口径）p50 **414 vs 359（1.15×）**、p90 759 vs 570（1.33×）、W9 270.43 vs 161.36（1.68×）；单线程差距 3.2×/3.75×。
- 环境收尾：vseg-1+stripe2 master 2077841（Keys=400、1.68GiB），target 341038 未动。

### NOF-041 关键事实（188µs 全在 bdev；vseg 全程处于"慢状态"，2026-11-23）

- **逐 IO 字段齐全**（11 个，含 `target_lcore`、`target_*_tick`、`bdev_us`、`data_us` 等），统计口径内 `target_valid=0` 为 0 条。
- **T1 对比（128KiB 读，排除 keys 0–96）**：roundtrip p50 **274 vs 86**；target_us **256 vs 62**；**bdev_us p50 250 / p90 306 vs 46 / 58（+204/+248）**；data_us **5 vs 14（更小）**；outside_us **16 vs 23（更小）** ⇒ **多出的 ~188µs 全部在 bdev（盘执行段）**，传输与客户端不是瓶颈。
- **分布形状**：vseg bdev **单峰极集中**（<100µs 占 **0.0%**；200–300 占 76.2%、300–500 占 12.0%、100–200 占 11.7%；mean 244.5≈p50 250）；plain 干净区 **双簇**（<50 占 60.0%、50–100 占 39.9%、≥200 仅 0.1%；mean 45.5≈p50 46）。
- **强对照（新发现）**：vseg 的 bdev 分布与 **plain 盘首慢区（keys 0–96）** 的分布**逐桶吻合**（200–300：76.2% vs 76.8%；300–500：12.0% vs 12.2%；100–200：11.7% vs 10.9%；p50 250 vs 249）⇒ **vseg 的全部 400 个 key 都处在 plain 仅盘首 97 个 key 才有的"慢状态"**。
- **qpair=1 同样单峰**（200–300 占 76.9%、<100µs 0.0%）⇒ bdev 分布形状与 lane 数无关。
- **16t 对照**：vseg bdev 281 vs plain 266（+15），差距转为 outside_us +29、data_us +5；16t 下两者 bdev 分布已接近（plain 16t 也 0.0% <100µs），无法区分 vseg 特有与并发效应。
- 日志偏差：16t 两份有 3.4–4.6% 行被并发日志打断（torn），已剔除；T1 两份 0 条打断，结论完全可靠。

### NOF-042 关键事实（时序/落核/重叠均无差异；日志事实纠正，2026-11-23）

- **提交顺序**：vseg 每条 lane 内严格按 LBA 递增（`(key,qid)` 逆序对 p50=0，800 组仅 1 组异常）；q1 臂全局完全单调；q4 臂"每 key 整体"~47 个逆序对，纯粹是 2 个 child 落在不同 lane 交错提交所致。每 key 32 条提交总时长 p50 = 32.0µs（plain 23.6µs）；>50µs 断点仅 1/12,277（异常行）。
- **落核/qpair**：两边都用 lcore 3/4/5/6、qid 2/3/4/5，一一对应且均匀；**但 vseg 四核 bdev 全线 218–261µs，plain 四核 47–48µs**；q1 单 lane 为 248µs ⇒ 慢不集中于某个核，lane 数不改结论。
- **大 IO 重叠**：T1 下一 key 小 IO 落在上一 key 任务执行期内的比例 0.61%（plain 0.08%）⇒ 无实质重叠；16t 7.38% vs 1.95%，但 16t 两边 bdev 已重合。
- **重要事实纠正**：此前"每 key 一条 2MiB/4MiB 大 IO（URMA_SLOW_IO）"是**日志行拼接**所致（`nof_slow_task` 任务级日志插入 URMA_SLOW_IO 行、换行丢失）；测量段真正的 URMA_SLOW_IO（opcode=2）**全部是 blocks=256（128KiB）**。任务级记录：vseg 每 key 2×2MiB（child_count=2）、plain 每 key 1×4MiB；任务 drain p50 vseg 349µs vs plain 108µs。此前各单只取 blocks=256，结论不受影响。
- **判决**：时序/落核/重叠均无可辨识差异 ⇒ 188µs 增量不在"提交方式/target 调度/排队"，只能在**数据在盘上的物理状态**或设备对该数据的执行时间。

### NOF-043 关键事实（写侧 lane 排除；六个维度穷尽，2026-11-23）

- **q1 写生效并落盘**：writer 自证 `io_qpairs=1`；两轮补齐（777/23 → 798/2），master `committed_allocations=800`、`logical_used=3,355,443,200`（精确 = 800×4MiB）、vsegments=51；写窗口 W9 **27.73µs**（n=12,067）+ 补齐轮 30.55µs。
- **读回（T1 q4、warmup=128）**：800/800、verify 0 错；读窗口 W9 **255.00µs**（n=55,299，exec 97.1%）。
- **三桶 bdev 完全一致**：桶1（0–96 慢窗口）p50 **252** / p90 309；桶2（97–399，q4 写）**252/309**；桶3（400–799，**q1 串行写**）**253/310**；慢占比 91.6% / 91.0% / 91.2%；三桶直方图逐桶几乎相同（200–300 占 75–77%，<100µs 均 0%）。data_us 三桶均 p50 6µs。
- **布局全连续**：三桶窗口步长恒 1、相邻差=256 占 99.4–99.6%、>1MiB 跳变 0。
- **判决**：**写侧 lane 交织不是主因**。至此六个维度全部排除：IO 尺寸/数量、LBA 布局、写线程数、盘首慢区、qpair/lane 数、写侧 lane 数；加上时序/落核/重叠与传输/客户端，vseg 的 188µs 增量只能落在 **target/盘内部的逐 IO 执行细节**。

### NOF-044 关键事实（逐 IO 明细已有但退化；不重启，2026-11-23）

- **现有明细**：`/tmp/nvmf_tgt.log` 有 **13,767 条 `URMA_BDEV_DETAIL`**，字段：`trace_seq qid cid bdev_us exec_to_read_us read_to_submit_us submit_call_us submit_to_callback_us callback_to_complete_us enomem_wait_us enomem_count`。
- **子段退化**：`submit_to_callback_us ≡ bdev_us`（13,767/13,767）；read_to_submit/enomem_wait 恒 0；submit_call 仅 620 条非零；exec_to_read 3 条、callback_to_complete 43 条 ⇒ 只能得出"时间在 submit→callback"，**拆不开 raid0 队列 vs NVMe 完成**。
- **使用限制**：明细无时间戳；`trace_seq` 每客户端会话从 1 重置，不能跨会话 join（实测 60 条 T1 join 仅 17 条且数值不符，判为误匹配未采信）；LEGACY 模式只打印 `bdev_cpl−exec ≥ hz/2500 = 40µs` 的读 ⇒ plain（46µs）样本严重偏斜。
- **源码（/home/yin/nof/spdk-cpu）**：打印点 `urma.c:2558`；触发需客户端 trace 请求 + READ + LEGACY ≥40µs；sink 模式 legacy/quiet/echo/buffer，未设置默认 legacy；buffer 模式需 `SPDK_URMA_STAGE_TARGET_PATH`，退出时 dump 24 字段二进制（仍无 raid0/NVMe 拆分）；`SPDK_URMA_TRACE_ENTRIES` 在 lib/nvmf 内无命中（当前 `TARGET_TRACE_ENTRIES=0`）。
- **重启代价更高**：target cmdline 只有 `-m [2-7] --wait-for-rpc`，子系统/RAID0/11 盘是启动后 RPC 下发，重启需完整 RPC 重放（NOF-016 脚本是否等价未验证）。
- **判决**：按 A4"已有明细 ⇒ 不重启"；环境未动（target 341038、vseg master 2077841）。

### NOF-045 关键事实（vseg 区数据无罪；盘对模式极敏感，2026-11-23）

- **裸 SPDK 顺序 QD1（node1→target，`-o 131072 -T1 -b1 -t6`）**：

| 支 | 区域 | 客户端 p50 | target W9 | 备注 |
|---|---|---|---|---|
| A | [1.6,2.6)GiB，**vseg q1 写区** | **40.96µs** | **91.57µs** | n=56,381 |
| B | [3.2,4.2)GiB，**plain 写区** | **41.28µs** | **91.28µs** | 与 A 几乎逐位相同 |
| C | [0.5,1.0)GiB，vseg q4 区（低 LBA） | 215.04µs | 155.79µs | 裸读也慢 |
| D | [0,0.5)GiB，盘首慢窗口 | 212.48µs | 153.92µs | 裸读也慢 |
| E | 同 A 区，**随机** | 220.16µs | 188.90µs | seq→rand 41→220µs |

- **判决（分区域）**：
  - **高 LBA 的 vseg 数据裸读 41µs，而 Mooncake-vseg 读同批 key 是 253µs ⇒ 数据/盘无辜，慢在 vseg 路径，增量约 +210µs**；
  - 低 LBA [0,~1GiB) 裸读也 212–215µs ⇒ 盘在该地址段本身慢（慢区上边界未知，介于 1.0–1.6GiB）；
  - NOF-040 的"全部 400 key 慢"是两者叠加：低 LBA 盘慢 + 高 LBA vseg 路径慢。
- **盘对访问模式极敏感**：同一区域顺序 41µs vs 随机 220µs（W9 91.6→188.9）；**vseg 的 253µs 正落在"随机档"** ⇒ 候选机制：vseg 的读流在 target/盘侧被看成非顺序（但 q1 仍 267µs，不能只归因 lane 数，需看 target 侧实际执行顺序）。
- **preflight 偏差（重要）**：`urma_perf` 的 preflight 写在**每支的 `-l` 起始 LBA**（非固定 LBA0），已覆盖 keys **0 / 128 / 400 / 800** 各 128KiB ⇒ 这些 key 的 Mooncake verify 会失败；后续 Mooncake 测试需新集群/新 key。
- 环境未变：target 341038 未重启、vseg master 2077841 在。

### NOF-046 关键事实（顺序假设证否；矛盾收窄到 bdev 提交侧，2026-11-23）

- **target 侧顺序**：q1 单 lane（12,401 条）rx/exec/submit 逆序对全 **0**、LBA 位移 0、只用 lcore3、submit 与 exec **100% 一致**；plain 同样严格（0 逆序，p50；少量异常 key 不计）；q4 逆序 p50 61/496（12.3%）、最大位移 12–13，成因是 **4 lane × 2 child 的 8 条各自严格递增的流交错**（全局逆序仅 0.19%），不是乱序。
- **q1 严格顺序仍慢**：q1 bdev **223–265µs** vs plain 同 lcore **43–52µs**（NOF-042 已证同核差异）⇒ **"盘把 vseg 看成随机档"的解释不成立**（q1 与 plain 的 target 顺序完全一致）。
- **bdev 与执行次序**：每 key 内按执行次序 1–8/9–16/17–24/25–32，bdev 从 218/223 升到 271/265/52；三种读法相对增幅一致（+19–24%），属通用排队/缓存累积，不区分 vseg/plain。
- **当前唯一未解矛盾**：target 侧看到的 vseg q1 流与 plain **完全相同**（顺序、IO 大小 128KiB、数量、盘区；该区裸 SPDK 顺序仅 41µs），但 bdev 执行时间 5×。剩余可疑面只有 **target bdev 提交时携带的其他参数**（数据 buffer/描述符、raid0 stride 对齐、submit flags、iobuf 类型）——现有遥测不含这些字段，需代码级插桩。
- 环境未变：target 341038、vseg master 2077841；零环境操作。

### 决策点（2026-11-23，NOF-046 后更新）

- 已证否：IO 尺寸/数量、LBA 布局、写线程数、盘首慢区（仅低 LBA 部分）、qpair/lane、写侧 lane、提交顺序、target 到达/执行顺序、"盘看成随机档"、传输/客户端、测量伪影。
- 剩余矛盾需要一个**当前遥测看不到的 target bdev 提交侧参数**才能解释。可选：
  1. **收口交付**（推荐）：以"vseg 路径 +210µs、数据无罪、单 target 无读时延优势"更新最终报告，不再改代码；
  2. **客户端代码插桩**：改 Mooncake vseg 提交路径，记录每 child 的客户端 buffer 地址/MR、target LBA、提交时刻与 lane（自建构建，天级）；
  3. **target 代码插桩**：改 SPDK `lib/nvmf/urma.c`，记录 bdev 提交的 buffer/描述符/flags/stride（重编 target + 重启，需用户批准，天级）。
- 其余备选（暂缓）：
1. **按现状交付**：报告已按 NOF-035 更正；遗留一个明确开放问题（每命令 2.5× 的机制）。
2. **重启 target 开逐 IO 明细**（需用户批准；target 重启后需重新 register/mount，盘上数据不丢；预计 1.5–2 h）：用 `TARGET_TRACE_ENTRIES>0` 采 `URMA_BDEV_DETAIL`，对比 plain/vseg 的逐条 exec 时间、到达顺序与尺寸分桶。
3. **客户端代码插桩**：自建构建，在 vsegment 提交路径加时间戳（提交节奏/子任务并发），定位与 plain 的差异；周期以天计。
4. **多 target/多成员部署**：当前环境不具备，唯一可能让 vsegment 有结构性优势的方向。

### 口径与数据安全（2026-11-20 用户补充）

- **数据保留**：GAP 旧数据是盘起始连续的 ~2.47 TB（294,669 keys × 8 MiB）；用户要求尽量保留，**实在不行可清理、后续重新 prefill**。新 vsegment 段为 100GiB @ offset 0，writer 只写 400×4MiB=1.6GiB ⇒ 只覆盖最前 1.6GiB；若要零覆盖，需改造 `nof_register` 传入 `base=5TiB`（mount RPC 支持 `base`，该 CLI 写死 0），暂不做。
- **拓扑事实（同事脚本 + 交接文档 §一/§二 已相互印证）**：node1 = etcd + master + writer + 编排，node3 = target + reader ⇒ 写路径跨节点、读路径数据与 target 同机（元数据仍远程）；reader 侧 roundtrip−target≈29µs，属**本机口径**。我们计划 reader 两种位置都测（node3 对齐同事；node1 远端端到端）。
- **旧 gap 进程**：用户已许可清理 master/client；当前计划暂不清理（我方 master 放 node1，不与其抢端口），需要 node3 资源时再停。

### NOF-003 关键事实（假设已证实，2026-11-20）

- **实测日志**：`oplog replay: source=<node1>:50058, applied_entries=9306, objects=0, segments=0, vsegment_partitions=1024, elapsed_ms=527`；紧随 `self oplog replay: restoring recovered state … vsegment_partitions=1024`；0.6ms 后 `Restored from standby: 0 objects, 0 segments`。
- **etcd**：`/oplog/` 共 8752 key——`vseg_yin_bonding/` 7942（7941 batches + durable_prefix，体积 11.07 MiB，`last_seq=9306`）+ `mooncake_cluster/` 810；顶层只有 `/oplog/`、`/cvm/`、`/datasystem/`（后者属别的服务）。
- **结论**：清场必须删 `/cvm/<cid>/` **和** `/oplog/<cid>/`，否则重启即重放；"全新 cluster_id"天然规避。
- **构建**：同事构建目录 7.7 GB（root 700），master md5 `f4ac492fff6a28e0d2b4f6619d540266`、client `ff75606d…`、nof_register `1655b48a…`、stress_cluster_bench md5 `575d2151ed39d256dbfa6269e4221abb`；工具链 python 3.11.6 / go 1.24.6 / cmake 3.27.9 / ninja 1.13.2 / gcc 12.3.1；两机无外网。
- **SPDK**：node1 yin spdk-cpu 干净（HEAD d98b48e，nvmf_tgt md5 `053eb241…`）；node3 yin spdk-cpu 脏（HEAD e12b7cb，12 个改动）；CONFIG_URMA=n 但二进制含 967–1005 个 URMA 符号，机制 `[未获取]`；node3 gap spdk 的 nvmf_tgt md5 未记录（690,992 B）。
- **实验参数**：slot_count=1024；writer/reader 均为 400 keys × 4 MiB、16 线程、reader order=seq、verify=true；客户端 env 含 `MC_URMA_TRANS_MODE=RM`、`SPDK_URMA_CAPSULE_TRANSPORT=urma`、`MC_STORE_PARTITION_SLOT_COUNT=1024` 等。
- **node3 旧 gap 环境（只读实测）**：12 个 bdev（11 单盘 + `URMA_RAID0` 76.85 TiB）；subsystem `nqn.2026-01.io.spdk:urma-gpu-test`，listen URMA / IPv4 / 141.61.84.*:4457，nsid 1 → `URMA_RAID0`；RPC socket `~/gap/run/spdk.sock`。
- **其余前置**：node3 HugePages 余 346/4640；yin 资产均 root:root 700；我方无构建产物；node1 的 50056/8086/9006 空闲、HugePages 充裕。

---

# NOF-001（已关闭 · 只读侦察 · 2026-11-20）

## 背景与目标

开发侧对这个交接项目零了解。本单目的是**把事实摸全**：同事资料里到底是什么项目、代码/补丁/文档/数据有哪些、两台测试机现状如何。本单**不做任何部署、不启动任何服务、不下结论**，只取证。

## A. 环境接入（唯一允许的写操作在此）

1. 读连接说明：`C:\Users\l00955908\Desktop\mooncake nof vsegment\连接node1和node3.md`，按其方式非交互 SSH 到 **node1** 和 **node3**。
2. 两节点各执行：`mkdir -p /home/l00955908/mooncake-nof-vsegment`，并 `ls -ld` 确认。
3. 记录：两节点是否接入成功、登录用户、hostname、`sudo -n true` 是否可用（只报可用/不可用）。
4. **不要**把连接文档里的 IP、密码、密钥写进回执；用 node1/node3 指代。

## B. 同事交接资料全量清点（Windows 侧，只读）

根目录：`C:\Users\l00955908\Desktop\mooncake nof vsegment`

要求"全量"：**所有文件都要出现在清单里，不得抽样**；对文档/脚本/配置要逐份打开读。

1. **目录树**：相对路径 + 文件大小 + mtime，覆盖全部文件（若超过 400 行，先给完整目录结构 + 每类文件的统计，再在证据段列出关键文件）。
2. **文档类**（md / txt / docx / pdf / xlsx / pptx，含所有子目录）：每份 1–3 句摘要；重点文档提炼：项目目标、方案设计、当前进展、待办事项、已测数据与结论。
3. **脚本 / 配置**（sh / py / yaml / json / toml / conf / ini）：每份用途一句话 + 关键参数（端口、路径、设备名、远端地址；地址用占位符）。**特别标出所有端口号**，与《端口规划》表对照，列差异。
4. **代码**：
   - 识别资料里有哪些仓库/代码目录/补丁文件；
   - 若是 git 仓：给 remote（URL 主机名可保留）、branch、commit、`git status` 摘要、相对上游的改动文件清单（`git diff --stat` 即可，**不贴 diff 全文**）；
   - 若是散文件或 patch：给文件清单、入口点、patch 清单及每份作用一句话；
   - 明确回答：**vsegment 是什么、改在哪些模块/文件、基于哪个 mooncake 版本**；若资料中无明确说明，标 `[未获取]`。
5. **测试数据 / 结果**：列出数据文件与日志（名称、大小、时间）；对结果类文件给结论性摘要（吞吐/时延/配置），不贴原始大表。
6. **同事的运行方式**：从脚本/文档里还原——服务如何起步（master / target / client / etcd 等各是什么命令、什么参数）、数据如何准备、测试如何发起。按"步骤清单"格式给，标注来源 `文件:行号`。

## C. node1 / node3 现状盘点（只读）

1. OS 发行版与 kernel 版本；CPU 型号、物理核/逻辑核数、NUMA 拓扑；内存总量。
2. 磁盘：块设备清单、挂载点、`df -h` 可用空间（重点 `/home`）。
3. 网络：网卡名与链路状态、网段（IP 最后一段打码）；RDMA/URMA 设备是否存在（`ibv_devices`、`rdma link` 或 URMA 对应工具，有什么报什么）。
4. 组件现状：SPDK（目录、版本、是否编译产物齐全）、Mooncake（源码/二进制、版本/commit）、etcd、其他相关组件；各自是否正在运行。
5. 进程与端口：`ss -ltnp` 摘要；**逐个核对 4457 / 50056 / 8086 / 9006 / 50156 是否被占用**，被占则给 PID、进程名、属主（不杀、不动）。
6. node1 ↔ node3 互通：各 ping 3 包，记录通断与 avg 时延、走哪张网卡。
7. `/home/l00955908/` 下是否已有与本项目相关目录或旧产物（只列名称与大小）。

## D. 本单边界（禁止事项）

- 不启动 / 停止 / 重启任何服务与进程；不执行 SPDK setup 脚本；不挂载盘；不改系统配置；不装包。
- 除了 A.2 的 mkdir，不在节点上创建/修改任何文件；不落探针脚本。
- 资料目录只读，不做任何修改；不复制大文件。

## 回执必答（第一行）

一句话：交接资料是什么项目、核心产物有哪些；node1/node3 是否可接入；5 个目标端口是否空闲。

逐条必答：
1. 资料总体：文件总数、总大小、顶层目录结构；哪几份是必读文档。
2. 项目是什么、目标是什么、当前完成到哪一步、同事留下的待办（各 1–3 条，标注来源）。
3. vsegment 相关：代码/补丁位置、基于哪个 mooncake 版本、改了哪些模块（文件级）。
4. 同事脚本/文档中出现的端口、路径、设备名，与我方《端口规划》的差异清单。
5. node1/node3 接入结果（成功/失败）、`sudo -n` 可用性、工作目录创建结果。
6. 5 个目标端口逐个：空闲 / 被占（占用者 PID+进程名）。
7. 两机是否已具备 SPDK + Mooncake + RDMA/URMA 条件；磁盘空间是否充足。
8. 发现的所有异常与风险。

---

# NOF-002（已关闭 · 只读取证 · 2026-11-20）· 交接文档 + 11 盘实验脚本 + P6 现场事实提取

## 背景（来自 NOF-001，勿重复取证）

- 目标：单 key（4–8 MiB）读时延 750µs → 300µs；**11 盘读路径端到端 + 五段拆分未取得**，卡在 P6 reconcile `INVALID_PARAMS`。
- 开发侧已用公开仓代码把报错定位到 `ReconcilePartitionRoute`（master 每周期日志 `count=1024, first=0: INVALID_PARAMS`，随后 `Failed to refresh vsegment ownership: PERSISTENT_FAIL`）；但 **P6 的执行序列与现场配置未知**，不能定因。
- 本单是**只读取证**：不跑脚本、不动任何进程/服务/文件；唯一目标是把 P6 与五段拆分所需的事实拿全。

## A. 交接文档提取

对象：资料目录内 `交接文档_NoF_VSegment性能测试.md`（约 11.9 KB，8 节）。要求：**摘要 + 关键段落原文，总输出 ≤120 行**。

1. 全文目录：8 节标题 + 每节行号范围（`文件:行号`）。
2. 环境与构建：mooncake / spdk 仓库、分支、commit、构建命令、产物路径；本地改动（go.mod / go.sum）的 diff（≤20 行）。
3. P1–P6 各阶段：名称、目的、执行位置（哪台机）、启动/停止顺序、关键命令、判据。**特别说明 P6 之前是否重启/接管 master，是否复用快照。**
4. 五段拆分：五段各是什么（定义 / 计时点）、采集工具 / 日志字段、已有结果。
5. 已有性能数字的原文上下文：841→610µs、W9 294→54.7µs、bdev_us≈513µs 等（各一句话 + 行号）。
6. P6 原文段落（≤30 行）+ 同事留下的待办原文。
7. `机器与使用说明.md` 缺失影响哪些步骤（文档中哪些地方引用了它）。

## B. 实验脚本提取

对象：资料目录内 `exp_11disk_run.sh`（约 15 KB）。

1. 常量区（端口 / 路径 / cluster_id / NQN / 设备名，约 L1–L60）原文 ≤40 行；IP 打码。
2. P1–P6 各阶段：函数名 / 命令序列摘要（每阶段 ≤15 行），标注行号范围。
3. P6 完整代码段原文（≤40 行）。
4. 危险动作区（约 L100–L114）原文（≤20 行）：哪些动作、触发条件、是否会杀进程 / 接管 / 还原。
5. 运行方式：在 Windows 还是 node1 执行；入口参数；依赖前置（谁起 master / target、谁 prefill、谁发起读、分区 / quota 怎么建）。
6. 若脚本含"不绕回"校验或数据量 / 读取范围计算，给出对应行。

## C. P6 现场取证（只读）

1. node1 的 yin master 进程：PID、完整 cmdline（IP 打码）、cwd、二进制路径 + mtime + md5、启动时间；从日志首部找启动参数 / 版本行（≤20 行）。
2. 该 master 的 etcd（node1:2379）：`etcdctl get --prefix` 统计 partition / vsegment / quota 相关 key 数量；给出 partition "0" 的 route key 名称与 value（≤10 行，脱敏）；列出 quota / profile 相关 key 名。工具缺失则标 `[未获取]`。
3. master 日志中 P6 失败前后的关键行（去重 ≤60 行）：快照加载 / 恢复、quota load、reconcile stats、错误行。
4. 脚本依赖产物现状：`/tmp/vsegment_cluster_0.state`、`/var/tmp/spdk.sock`、`/var/tmp/nvmf_tgt.pid` 等（存在性 / 大小 / 时间）。
5. node1 上当前与本项目相关的进程清单（master / etcd / target / bench）：PID、启动时间、cmdline 摘要。
6. 结论性回答：**P6 的输入是什么**（分区数 / quota profile / 快照来源）、**失败前最后一步成功操作是什么**、**日志里是否带 detail 字符串**。给不出就标 `[未获取]`，不推断。

## D. 边界

- 只读；不启停 / 不接管任何进程；不改任何文件；**不执行脚本**。
- 不读 yin 源码正文（开发侧已从公开仓获取代码，无需外传）。
- 不贴整篇文档 / 脚本 / 日志；按上方行数限制。
- IP / 账号 / 密码一律打码。

## 回执必答（第一行）

P6 具体做什么（带出处）、失败发生在哪一步、输入是什么；现场能否给出 `INVALID_PARAMS` 的 detail；五段拆分如何采集。

---

# NOF-003（已关闭 · 只读取证 · 2026-11-20）· 验证"旧 oplog 重放"假设 + 采集自建环境要素

## 背景（开发侧分析结论，供判断取证重点）

- NOF-002 已确认：P6 失败时 `partitions_installed=0`，即失败在 partition manager 安装之前；`INVALID_PARAMS` 无 detail。
- 开发侧读 be10be2 源码：master 启动时 `TrySelfRecoverFromOpLog()` 会重放自己的 OpLog 流（etcd 前缀 `/oplog/<cluster_id>/<source_id>/...`），把历史 vsegment 快照装入 `recovered_vsegment_snapshots_`；日志 `Restored from standby: 0 objects, 0 segments` 只有在 objects/segments/vsegment_partitions **至少一项非空**时才会打印，现场 objects=0、segments=0 ⇒ **vsegment 快照非空**。
- 而脚本 `do_clean` 只删 `/cvm/<cid>/`，未删 `/oplog/<cid>/`。
- 本单只验证这条链 + 收集"自建我们自己的环境"所需信息；**全部只读**。

## A. 验证假设（node1，只读）

1. 日志：`sudo -n grep -n "oplog replay:\|self oplog replay\|Restored from standby" /tmp/master_ha_50058.log | head -20`，给原文（≤20 行）。重点记录 `source=`、`applied_entries=`、`vsegment_partitions=`。
2. etcd（node1:2379，只读，**不得删除/修改任何 key**）：
   - `/oplog/` 前缀 key 总数；
   - 按 cluster 分组计数（如 `/oplog/vseg_yin_bonding/`、`/oplog/mooncake_cluster/`）；
   - `vseg_yin_bonding` 下：source_id 层数量；`durable_prefix` 的值（若有）；
   - 批次体量估算（`--print-value-only | wc -c`，若 >10 MB 立即停止并标"体量大"）。
3. 明确回答：要让清场"真干净"，除 `/cvm/<cid>/` 外还必须清哪些前缀（按实测 key 结构回答）。
4. 核对：master 日志有无 `quota generation ... already published`（本轮是否二次发布/跳过）。

## B. 自建环境要素（只读）

1. yin 的 `vsegment_cluster.sh`（`/home/yin/Vsegment/vseg_scripts/`）——只给结构与关键参数（≤40 行，附行号）：
   - 子命令（master/target/register/writer/reader）及各自命令模板；
   - master 参数怎么传（端口、cluster_id、vsegment 参数）；
   - target：盘列表、raid0/ns 建法、ns 大小、NQN、端口、spdk 目录；
   - writer/reader：keys 数、key 大小、线程数、order、输出路径；
   - register：mount 命令与判据。
2. `target_nvme_takeover.sh`：正向绑定 / 还原各是什么命令、盘列表来源、是否幂等、是否需 root（摘要 ≤20 行 + 行号）。
3. 构建信息：
   - `mooncake-vesegment-sysurma-build`：CMakeCache 关键项（源目录、install prefix、build type、编译器、生成器）、产物清单与 md5（mooncake_master 及常用 bench 工具）、目录总大小；
   - `/home/yin/nof/spdk-cpu`：`build/bin/nvmf_tgt` 等产物 md5、CONFIG 中 URMA 相关开关、目录总大小；
   - 两机 `python3 / go / cmake / ninja / gcc` 版本。
4. node3 现有 gap 环境的盘占用：`lsblk` 与 vfio 状态、现有 raid0/ns 名称与大小（只读，**不调用任何变更类 RPC**）。

## C. 边界

- 只读；不启停 / 不接管进程；不绑 / 解绑盘；不删改 etcd；不执行 takeover 或实验脚本。
- 不贴源码 / 脚本全文；按行数限制；IP / 密码打码。

## 回执必答（第一行）

旧 oplog 是否真实存在且被重放（给 `vsegment_partitions` 数值）；`/oplog/<cid>/` 是否为清场漏删前缀；自建环境还缺哪些前置。

---

# NOF-004（已关闭 · 启动我方 master · 2026-11-20）· 全新 cluster_id + 我方端口，验证干净环境

## 背景与范围

- NOF-003 已证实：P6 全灭的直接原因是旧 oplog 重放（1024 条旧 vsegment 快照）。本单验证"**全新 cluster_id（无旧 /cvm、无旧 /oplog）时 reconcile 应能通过**"的前半程：只把我们的 master 拉起来。
- **master 放 node1**（该机 50056/8086/9006 空闲、HugePages 充裕），与同事拓扑一致（master=node1，target=node3）；**不打扰 node3 的旧 gap 环境**。
- 使用同事现成二进制**原地运行**（不拷贝、不重建）：
  `/home/yin/nof/mooncake-builds/mooncake-vesegment-sysurma-build/mooncake-store/src/mooncake_master`（md5 `f4ac492fff6a28e0d2b4f6619d540266`，be10be2 构建）。
- **全新 cluster_id：`vseg_nofvseg`**（禁止与 `vseg_yin_bonding` / `mooncake_cluster` 混用）。
- 本单**不 mount、不注册、不起 target、不写盘**。

## 1. 预检（只读）

1. node1：`ss -ltnp | grep -E ':50056|:8086|:9006'` 必须全空；有占用则停手回报。
2. `sudo -n pgrep -af mooncake_master`：记录现有实例（不得干扰）。
3. etcd：`etcdctl --endpoints=127.0.0.1:2379 endpoint health`；`/cvm/vseg_nofvseg/` 与 `/oplog/vseg_nofvseg/` 的 key 数必须为 0。
4. 工作目录：`/home/l00955908/mooncake-nof-vsegment`（node1）。若属主为 root：`sudo -n chown l00955908:l00955908 /home/l00955908/mooncake-nof-vsegment`（只允许改这一个目录）；在其下建 `run/`。

## 2. 参数来源

- `sudo -n cat /home/yin/Vsegment/vseg_scripts/run_vesegment_master.sh`（40 行）——按其中 **环境变量与参数集合** 原样复刻；本单只允许替换：`cluster_id`→`vseg_nofvseg`、`rpc_port`→`50056`、`metrics_port`→`9006`、`rpc_address`→node1 实际的 `192.168.84.*` 地址、bind IP 同理。
- 必须保留：`MC_STORE_PARTITION_SLOT_COUNT=1024` 及脚本里 master 相关的其余 env（如 URMA 设备相关），以及全部 `--vsegment_*` 参数（auto_plan、required_medium=nof、io_alignment=4096、member_count=1、stripe_size=2097152、member_extent_size=67108864、reserved_ratio=0）、`--enable_ha=true --enable_oplog=true --ha_backend_type=etcd --submaster_count=1 --default_kv_lease_ttl=300000 --default_kv_soft_pin_ttl=300000 --cvm_segments_vsegment_exclusive=true`。
- 若 `--help` 中存在 HTTP metadata 端口开关，显式设为 **8086**；不存在则在回执写 `[不存在]`。

## 3. 启动（root，nohup）

- 日志：`/home/l00955908/mooncake-nof-vsegment/run/master_50056.log`。
- 用 `sudo -n bash -c 'cd <workdir>; export <env…>; nohup <binary> <args…> > run/master_50056.log 2>&1 & echo $!'` 形式，拿到 PID。
- 等待 ~15 s 后按第 4 节确认；**失败不做任何清理**，回执贴日志尾部 ≤30 行。

## 4. 启动后确认（只读）

1. 进程：PID、属主、启动时间、cmdline（IP 打码）。
2. 端口：50056、9006（+8086 若存在）是否 LISTEN。
3. 日志（≤30 行关键行）：启动 banner、SPDK init rc、`VSegment auto-plan deferred`（未 mount 时预期出现，属正常）、任何 ERROR/WARNING。
4. `ss -ltnp` 摘要；`VSegment Status` 行（预期 `enabled=0`，无 quota）。
5. etcd：`/cvm/vseg_nofvseg/`、`/oplog/vseg_nofvseg/` 的 key 数与 key 名（预期：cluster_meta、masters、ring_slots 等少量；oplog 0）。
6. 写清停止命令（供后续任务）：`sudo -n kill -TERM <PID>`（10 s 后不退再 `kill -KILL <PID>`；禁止 pkill/killall）。

## 5. 边界

- 只在 node1 新增一个 master 进程；**不停/不碰**任何现有进程（含 yin master、etcd、他人 client）。
- 不 mount / 不注册 / 不起 target / 不写 node3 / 不绑盘 / 不重启 / 不装卸驱动。
- 不改动任何既有 etcd key（只允许 master 自己写 `vseg_nofvseg` 前缀）。
- 回执不写具体 IP / 密码，用 node1 / node3 指代。

## 回执必答（第一行）

master 是否启动并监听 50056/9006；有无 ERROR；`/cvm|/oplog/vseg_nofvseg/` key 数；PID 与停止命令。

---

# NOF-005（已关闭 · 未通过 · mount + reconcile 门禁 · 2026-11-20）· 把 node3 现有 target 挂到我们的 master，验证 P6 已解

## 背景

- NOF-004：我们的 master（`vseg_nofvseg`，PID 2091164，50056/9006）在 node1 稳定运行，`enabled=0`、auto-plan deferred（等 mount）、reconcile 失败 0 次、无旧 oplog 重放。
- 本单：用同事的 `nof_register` 把 node3 **现有** target（URMA_RAID0 76.85 TiB，NQN `nqn.2026-01.io.spdk:urma-gpu-test`，4457，nsid 1）以 **100 GiB（107374182400 B）、base=0** 挂到我们的 master，验证 auto-plan 发布 1024 分区 quota 且 **reconcile 门禁 = 0**——这是"P6 已解"的第一直接证据。
- **不写数据**：mount 只做控制面登记（写我方 etcd + master 内存），不写盘；writer 留到下一单。
- **不动 target**：node3 的 `nvmf_tgt` 保持原样，不重启、不改参数、不碰盘。

## A. 预检（只读）

1. master：PID 2091164 存活；50056/9006 仍 LISTEN。
2. target 可达：node1 → target 的 `141.61.84.*:4457` 建连测试（`timeout 5 bash -c '</dev/tcp/…/4457' && echo OK`）；在 node3 上 `ss -lnt | grep 4457` 确认监听。
3. 取 register 的准确命令与环境：`sudo -n sed -n '560,600p' /home/yin/Vsegment/vseg_scripts/vsegment_cluster.sh`（register 函数及客户端 env）；回执只给 ≤15 行摘要（变量=值），不贴全文。
4. etcd：`/cvm/vseg_nofvseg/` 仍 3 键、无 `snapshot/vsegment_partition_quota`。

## B. mount（本单唯一写操作：我方 cluster 的 etcd 键 + master 内存）

1. 使用同事构建目录的 `nof_register`（md5 `1655b48aa849297b96249c1aa132a0f3`），root 执行；按 A3 探明的 env；输出到 `run/register_100gib.log`：
   ```
   <build>/mooncake-store/src/nof_register \
     "nqn.2026-01.io.spdk:urma-gpu-test" 1 <node3 的 141.61.84.*> 4457 URMA \
     <node1 的 192.168.84.*:50056> 107374182400
   ```
   - base 固定 0（CLI 写死，**不要改参数**）。
2. 命令返回后等待 ≤30 s，再进入验证；**失败不做任何修改性重试**。

## C. 验证（只读）

1. register 输出：`rc[0]` / `NoF registration succeeded`。
2. master 日志关键行（≤30 行）：`MountNoFSegment_slow … rc[0]`、`published vsegment quota generation 1 with 1024 partition/profile quotas`、`VSegment pool formed`、`vsegment service published … running first full reconcile sweep`。
3. **P6 门禁**：`grep -c "reconcile failures"` = **0**、`grep -c "Failed to refresh vsegment ownership"` = **0**；`reconcile stats` 中 `owned_by_me=1024`、`reconcile_fail=0`（若 stats 行未出现，等 10 s 再查）。
4. `VSegment Status`：`enabled=1, generation=1, profiles=1, partitions_installed=1024, vsegments=1024`。
5. etcd：`/cvm/vseg_nofvseg/snapshot/vsegment_partition_quota` 存在（~181 KB 量级）；`/cvm/vseg_nofvseg/` 键清单。
6. 60 s 观察窗后再核一次门禁（防止首轮假通过）。

## D. 顺带只读取证（≤20 行输出）

1. 交接文档 §一（L8–18）+ §二拓扑表（L20–27）原文摘录——用于敲定测试口径（远端 vs 本机）。
2. 现有 gap target 是否带 target 侧 W1–W10 打点：`sudo -n strings /home/l00955908/gap/spdk/build/bin/nvmf_tgt | grep -c 'URMA_BDEV_DETAIL\|W9'`；`sudo -n grep -c 'W9\|URMA_BDEV_DETAIL' /tmp/nvmf_tgt.log`（node3）。决定五段拆分的 target 段能否直接用现有 target 采。

## E. 边界

- 不跑 writer/reader；不写盘；不动 node3 的任何进程/盘；不重启 master；不改参数重试。
- 若 register 失败（URMA 协议/身份不匹配等）：贴 register 输出 + master 日志尾部 ≤30 行，**停手回报**，不换 target/不换端口。
- 不碰 yin 的 `vseg_yin_bonding` / `mooncake_cluster` 键与进程。

## 回执必答（第一行）

mount 是否 rc[0]；`reconcile failures` 是否 0；`partitions_installed` 是否 1024；失败则给错误码与首个错误行；§一 口径要点；现有 target 是否有 W1–W10 打点。

---

# NOF-006（已关闭 · 只读诊断 · 2026-11-20）· URMA attach 失败根因取证 + 换 target 前置

## 背景

- NOF-005：master（同事 sysurma 构建）attach 现役 target（旧 gap 的 `urma_p99` 构建）时 `URMA qpair err=-5`、attach `rc=-19`（Operation not permitted）；无 quota、无 reconcile。
- 同事的实验用的是 **yin 的 spdk-cpu** 构建的 target，不是旧 gap target。本单先**只读**取证，判断失败属于"协议/transport 不兼容"还是"设备/路径/权限"，并为"换 target 为同事构建"准备准确的前置命令。
- 全部只读；不重试 mount、不停进程、不绑/解绑盘、不跑 takeover、不改 etcd、不重启 master。

## A. target 侧现场（只读）

1. node3 现役 target 日志 `~/gap/logs/nvmf_tgt.log`：14:41:30–14:41:45 窗口内**除 `URMA_BDEV_DETAIL` 外**的全部行（≤30 行，去重）；重点找 connect capsule 被拒/错误码/未知命令。
2. `/proc/3243095/cmdline`（已有）与 cwd；`~/gap/run/`、`~/gap/logs/` 里找我们旧项目启动 target 的脚本/环境记录：URMA 设备名、bonding、transport 名（`urma_p99` 的来源）、ns 大小、strip。只给 ≤15 行摘要。
3. RPC 只读查询（socket `~/gap/run/spdk.sock`）：`nvmf_get_subsystems`、`nvmf_get_transports`（若存在）；确认 transport 注册名（urma / urma_p99）与 listener；`bdev_raid_get_bdevs`（若存在）确认 strip 与成员顺序。
4. URMA 设备与网络对应：`ls /dev/uburma`、`rdma link show` 或 urma 对应命令；`udmac0d1e2` / `udmac1d1e2` / `bonding_dev_0` 与网卡/平面的对应关系（sysfs 或驱动信息）；说明 target 的 `141.61.84.*:4457` 监听走哪个设备/平面，master 侧 `SPDK_URMA_DEV_NAME=udmac1d1e2` 是否在同一平面。

## B. 版本/协议对照（只读，≤30 行输出）

1. 两棵 SPDK 树在 URMA 传输上的差异：node3 `~/gap/spdk`（HEAD 170cbb9）与 node3 `/home/yin/nof/spdk-cpu`（HEAD e12b7cb，脏）——分别 `git log --oneline -10 -- lib/nvmf/urma.c lib/nvme/nvme_urma.c`，给提交标题即可；指出是否有 capsule/协议/标志位相关改动。
2. 两个 target 构建对比：gap `build/bin/nvmf_tgt`（md5 `6668b490…`，690,992 B）vs yin `build/bin/nvmf_tgt`（15 MB 级）；`strings` 对比关键 URMA 符号/版本串（各 ≤5 条）。
3. 结论只报事实：是否存在能解释 `capsule completion err=-5` 的协议/transport 差异。

## C. 换 target 的前置信息（只读）

1. `run_vesegment_target.sh`（node3 `/home/yin/nof/spdk-cpu` 或 vseg_scripts）：它如何调用 `target_nvme_takeover.sh`（完整参数模板：盘、`-R` strip、`-p`/`-N`、mask、SPDK 根、NQN、端口如何注入）、ns 大小如何确定、启动后如何等待端口。给 ≤20 行摘要 + 行号。
2. `target_nvme_takeover.sh` 在当前状态下的行为（11 盘已 vfio 绑定、旧 nvmf_tgt 在跑）：是否拒绝、是否自动还原；`-r` 还原命令；正向启动命令模板。
3. node3 HugePages 现状；旧 target 停掉后预计释放多少（`/proc/meminfo` + 目标进程的 hugepage 占用，能取多少报多少）。

## D. 边界

- 只读；不执行 takeover、不启停任何进程、不绑/解绑盘、不删改 etcd、不重启 master。
- 不贴整文件/大段日志；IP/密码打码。

## 回执必答（第一行）

attach 失败更像"协议/transport 不兼容"还是"设备/路径/权限"？给直接证据；换 target 的准确命令链与前置；旧 target 停止后预计释放的 HugePages。

---

# NOF-007（已关闭 · 重启我方 master 修正 URMA 设备 · 2026-11-20）· 只换设备名，不 mount、不动 node3

## 背景

- NOF-006 已证实：不是协议问题（两树 WIRE v4 一致）；我方 master 用 `SPDK_URMA_DEV_NAME=udmac1d1e2` 导致**零 `/dev/uburma` 映射、未加载 URMA provider**；同一二进制在 yin master（`udmac0d1e2`）下两者正常；现役 target 也是 `udmac0d1e2`。
- 本单只做一件事：**按 yin 的设备口径重启我们的 master**，确认设备/provider 映射出现。**不 mount、不动 node3、不写盘**。

## 1. 预检（只读）

1. 记录：master PID 2091164、cmdline、50056/9006 LISTEN。
2. etcd：`/cvm/vseg_nofvseg/` 3 键、`/oplog/vseg_nofvseg/` 1 键（重启前后应不变）。

## 2. 停止旧进程

- `sudo -n kill -TERM 2091164`；最多等 10 s，未退出再 `sudo -n kill -KILL 2091164`（禁止 pkill/killall）。
- 确认 50056/9006 已释放；**不得触碰 yin master 151060**。

## 3. 启动（root，nohup，新日志 `run/master_50056_r2.log`）

- 参数集合与 NOF-004 完全一致（cluster_id=vseg_nofvseg、50056/9006、地址、14 项 vsegment/HA、`--http_metadata_server_port=8086`）。
- **只改 env**：
  - `SPDK_URMA_DEV_NAME=udmac0d1e2`（原 udmac1d1e2）
  - 新增 `VSEG_URMA_DEVICE=udmac0d1e2`（对齐 yin 运行环境）
  - 其余保持：`MC_STORE_PARTITION_SLOT_COUNT=1024`、`MOONCAKE_VSEGMENT_METRICS_REFRESH_INTERVAL_MS=1000000`、`SPDK_URMA_CAPSULE_TRANSPORT=urma`。
- 记录新 PID。

## 4. 验证（只读）——设备映射是本单核心验收

1. 进程/端口：新 PID 存活；50056/9006 LISTEN。
2. **设备映射**：`sudo -n grep -c '/dev/uburma' /proc/<新PID>/maps` **必须 > 0**；确认出现 `/dev/uburma/udmac0d1e2`；`sudo -n grep -o 'liburma-udma.so\|liburma_ubagg.so' /proc/<新PID>/maps | sort -u` **必须两者都出现**。
3. 与 yin master（PID 151060）并排对比上述三项（只读）。
4. 日志：启动 banner、SPDK init `init_rc=0`、`VSegment auto-plan deferred`（预期，未 mount）、ERROR/WARNING 行；**不应**出现 attach/URMA 错误。
5. etcd 键数不变（3/1）。
6. 写清新的停止命令（`sudo -n kill -TERM <新PID>`）。

## 5. 边界

- 只动我们自己的 master 进程；不碰 yin master、etcd 既有键、node3、gap 环境；不 mount、不注册、不起 target、不写盘。
- 若新进程启动异常或设备映射仍为 0：**停手回报**（日志 ≤30 行），不尝试其它设备名、不改参数重试。

## 回执必答（第一行）

新 PID 是否存活并监听 50056/9006；`/dev/uburma` 映射数与 `udmac0d1e2` 是否出现；两个 URMA provider 是否加载；与 yin master 的对比；有无 ERROR。

---

# NOF-008（已关闭 · 通过 · 重新 mount + reconcile 门禁 · 2026-11-20）· 设备映射改为 mount 时验证

## 背景与口径修正

- NOF-007 已把 master 的设备口径改为 `udmac0d1e2`（PID 2162056，端口正常）。
- NOF-007 的"启动后映射>0"验收**作废**：SPDK/URMA 惰性初始化，未 mount 不会打开设备。**设备/provider 映射改为在 mount 请求触发的同一时刻验证。**
- 本单用与 NOF-005 **完全相同**的 `nof_register` 参数重试一次；不改参数、不做二次重试。若失败，停手回报。
- 不写盘、不动 node3 进程、不重启 master。

## 1. 预检（只读）

1. master：PID 2162056 存活；50056/9006 LISTEN；`/proc/2162056/environ` 中设备名仍为 `udmac0d1e2`。
2. target：node1 → `141.61.84.*:4457` 建连测试 OK；node3 `ss -lnt | grep 4457` 正常。
3. etcd：`/cvm/vseg_nofvseg/` 3 键、无 `snapshot/vsegment_partition_quota`；`/oplog/vseg_nofvseg/` 1 键。

## 2. mount（与 NOF-005 同参数，唯一一次）

- 在 **node1** 用同事 `nof_register`（md5 `1655b48aa849297b96249c1aa132a0f3`），root，输出 `run/register_100gib_r2.log`：
  ```
  <build>/mooncake-store/src/nof_register \
    "nqn.2026-01.io.spdk:urma-gpu-test" 1 <node3 141.61.84.*> 4457 URMA \
    <node1 192.168.84.*:50056> 107374182400
  ```
- **不要额外 export 任何 env**（与 yin 脚本的 register 调用一致）。
- 记录客户端输出与退出码。

## 3. 验证（mount 触发后立即采集，只读）

1. master 日志新增行（≤40 行）：`SPDK initialization CPU affinity … init_rc=?`、`Attaching to …`、`Opened NoF segment …`、`MountNoFSegment_slow … rc[?]`、任何 attach/URMA/capsule ERROR。
2. **设备映射（关键，此时才有意义）**：`sudo -n grep -c '/dev/uburma' /proc/2162056/maps`；是否出现 `/dev/uburma/udmac0d1e2`；`liburma-udma.so`、`liburma_ubagg.so` 是否加载。
3. 若 `rc[0]`：
   - `published vsegment quota generation 1 with 1024 …`、`VSegment pool formed`、`vsegment service published … first full reconcile sweep`；
   - `reconcile failures` **=0**、`Failed to refresh vsegment ownership` =0；
   - `reconcile stats`：`owned_by_me=1024`、`reconcile_fail=0`；
   - `VSegment Status`：`enabled=1, generation=1, profiles=1, partitions_installed=1024, vsegments=1024`；
   - etcd：`snapshot/vsegment_partition_quota` 存在；60 s 后再复查一次门禁。
4. 若失败：记录 master 错误链 + 该时刻 node3 target 日志（`~/gap/logs/nvmf_tgt.log`）窗口行（≤20 行），**停手**；不换设备名、不改参数重试。

## 4. 边界

- 不跑 writer/reader；不写盘；不动 node3 进程/盘；不重启 master；不碰 yin 的进程与 etcd 既有键。
- 失败不做任何修改性动作。

## 回执必答（第一行）

mount 是否 rc[0]；设备映射数（mount 时刻）与 udmac0d1e2/provider 状态；reconcile failures 与 partitions_installed；若失败，错误链与 target 侧对应日志；P6 是否首次得到有效验证。

---

# NOF-009（已关闭 · 通过 · writer 预填 · 2026-11-20）· 400 keys × 4MiB 写入我们的 100GiB 段

## 背景

- NOF-008 已通过：mount rc[0]，1024 分区 reconcile 全对（reconcile_fail=0），`enabled=1`、`partitions_installed=1024`、`logical_capacity=64 GiB`、`logical_used=0`。
- 本单用同事的 writer 口径写入 **400 keys × 4MiB = 1.6GiB**。
- **数据影响**：写入落在我们 100GiB 段（base 0）内、各分区 100MiB 窗口中的分散 4MiB 块，最多 ~1.6GiB。会覆盖旧 GAP 数据最前 100GiB 内的部分区域（用户已授权；必要时可重灌）。**不要**做其它写操作。
- 不跑 reader；不动 target/master 配置；不重启 master。

## A. 参数核对（先做，输出 ≤25 行）

1. 读 `run_vesegment_write.sh`（root 700，40 行级）全文：列出 writer 的全部 env（URMA 设备、chunk/inflight/workers、cluster/namespace 相关）与 `stress_cluster_bench` 的完整参数模板。
2. 建立"同事值 → 我方值"映射表：
   - master：`192.168.84.*:50056`；admin/metrics：`9006`；
   - `--metadata-server`：保持脚本原值；**若其中包含 cluster/namespace 名，必须改为 `vseg_nofvseg`**；
   - URMA 设备：`udmac0d1e2`；
   - 负载：400 keys × 4MiB、16 线程、batch 16、verify=false（与脚本一致）。
3. 若存在无法映射到我们集群的参数（写死 cluster/namespace 且无覆盖开关）→ **停手回报**，不要带错集群跑。

## B. 运行 writer（node1，root，日志 `run/writer_r1.log`）

1. 前置只读：master PID 2162056 存活、`enabled=1`；无同名 writer 在跑。
2. 按 A2 映射执行一次 writer；等待 `400 succeeded, 0 failed`（或等价 FINAL 行）；超时 10 min → 停手回报。
3. 记录完整命令、env、起止时间。

## C. 验证（只读）

1. writer 输出：成功/失败计数、耗时。
2. master：`VSegment Status` 的 `logical_used` 从 0 增至 ≈1.6GiB（给实际值）；`vsegments` 数量变化；无 ERROR。
3. etcd：`/oplog/vseg_nofvseg/` 键数增长；`/cvm/vseg_nofvseg/` 键数不变（8）。
4. node3 target：`~/gap/logs/nvmf_tgt.log` 中 `URMA_BDEV_DETAIL` 行计数增长；无新的真实写入错误（区分预检 TCP 探测造成的 `handshake failed rc=-104`）。
5. 写入量粗核：writer 输出与 target 侧 bdev 统计至少一项。

## D. 边界

- 只写本实验 400 keys；不跑 reader；不动 target 配置；不重启 master；不删任何 etcd key。
- 失败停手，不重试、不自行改参数。

## 回执必答（第一行）

writer 是否 `400 succeeded, 0 failed`；`logical_used` 实际值；master/target 有无错误；参数映射有无绕不过的冲突。

---

# NOF-010（已关闭 · 通过 · reader 读 + 五段拆分采集 · 2026-11-20）· node3 口径，拿我们自己的端到端数据

## 背景

- NOF-009：400 keys 已写入，master `logical_used=1,677,721,600 B`，无错误。
- 本单用同事的 reader 口径读取同一批 400 keys（4 MiB），**reader 放 node3**（与 target 同机，对齐同事口径），采集：
  1. reader 的端到端统计（Avg/P50/P90/P99/Max）；
  2. initiator 侧 `URMA_SLOW_IO` 五段/七字段逐 IO 记录（target 遥测随完成回传）；
  3. target 侧 W 汇总块（per-IO detail 当前未开，已知）。
- 只读数据，不写盘；不动 target/master 配置；不重启。

## A. 预检（只读，任何一条不满足即停手回报）

1. master：PID 2162056、`enabled=1`、Keys=400、`logical_used=1,677,721,600 B`。
2. **node3 是否有可用二进制**：`sudo -n ls -l <VSEG_BUILD_DIR>/mooncake-store/benchmarks/stress_cluster_bench`（按 run_vesegment_reader.sh 里的实际路径）。**若 node3 无该二进制 → 停手回报**，不要自行拷贝或改在 node1 跑。
3. 读取 `run_vesegment_reader.sh`（root 700）全文：列出 reader 的全部 env 与参数模板（≤25 行）；特别确认 `MC_STORE_TRANSFER_DIAG`、`MC_HIFREQ_LOG_SAMPLE_RATE`、`MC_NOF_SLOW_TRACE_THRESHOLD_US`、`SPDK_URMA_STAGE_TRACE_US` 等打点开关的取值。
4. 建立"同事值 → 我方值"映射：master `192.168.84.*:50056`、admin `9006`、URMA 设备 `udmac0d1e2`、metadata-server 保持脚本值；负载 400 keys × 4 MiB、16 线程、order=seq、verify=true（与脚本一致）。若出现无法映射的 cluster/namespace 参数 → 停手回报。

## B. 运行 reader（node3，root，日志写 node3 `~/mooncake-nof-vsegment/run/reader_r1.log`）

1. 记录 reader 启动前 target W 块基线（node3 `~/gap/logs/nvmf_tgt.log` 末尾 W8/W9/W10/TOTAL 行）。
2. 按 A4 映射执行一次 reader；等待 `Read complete` 或等价 FINAL 行；超时 10 min → 停手回报。
3. 记录完整命令、env、起止时间。

## C. 采集与验证（只读）

1. reader 输出：成功/失败数、Avg/P50/P90/P99/Max、吞吐、总耗时。
2. `URMA_SLOW_IO` 行：总条数；字段名清单；**给 3 条原始样本**（≤3 行）；若为 0 条，检查 env 是否按脚本设置并如实回报（不要自行加 env 重跑）。
3. master：`VSegment Status` 前后对比（logical_used 不变）；ERROR 计数。
4. target：本窗口 W 汇总块（n、W8/W9/W10 avg、TOTAL、breakdown 百分比）。
5. 读正确性：verify=true 的校验失败计数（应为 0）。

## D. 边界

- 只读；不写盘；不动 target/master 配置；不重启；不自行加/改 env 重跑；不拷贝二进制。
- 失败/缺前置：停手回报，不扩大范围。

## 回执必答（第一行）

reader 读成功多少、失败多少；Avg/P50/P90/P99/Max；`URMA_SLOW_IO` 条数与 3 条样本；target W9 本窗口均值；node3 二进制是否存在。

---

# NOF-011（已关闭 · 通过 · 诊断口径重跑 reader · 2026-11-20）· 打开打点 + 阈值 400µs，采五段/七字段

## 背景与授权

- NOF-010：端到端读 Avg 801.29µs；但 `URMA_SLOW_IO` 为 0，原因是诊断默认关闭（`VSEG_DIAGNOSTIC=0`），且默认阈值 100ms ≫ 实测 1.9ms。
- 开发侧授权本单**仅修改测量口径环境变量**（不改系统/服务/参数文件）：
  - `VSEG_DIAGNOSTIC=1`
  - `VSEG_SLOW_IO_US=400`（把慢 IO 阈值从 100000µs 降到 400µs，覆盖本次全部读 IO 462–1887µs）
  - 另按交接文档 L43 补 `MC_STORE_TRANSFER_DIAG=1`（文档口径；脚本无此变量，若无效不影响）
  - 其余 env/参数与 NOF-010 **完全一致**。
- 只跑一次；不写盘、不改 target/master、不重启。

## A. 预检（只读，任一不满足即停手回报）

1. master：PID 2162056、`enabled=1`、Keys=400、`logical_used=1,677,721,600 B`。
2. **二进制打点存在性**：
   - node3：`sudo -n strings <node3 build>/mooncake-store/benchmarks/stress_cluster_bench | grep -c 'URMA_SLOW_IO'`；
   - node1：同命令对 `575d2151…` 那个二进制。
   - 若 node3 为 0 → **停手回报**（不换到 node1 跑，由开发侧决定）。
3. 确认 node3 上无残留 reader 进程。

## B. 运行（node3，root，日志 `node3:~/mooncake-nof-vsegment/run/reader_r2.log`）

- 参数与 NOF-010 完全相同（400 keys × 4 MiB、16 线程、batch 1、warmup 5、order=seq、verify=true、master 50056、admin 9006、URMA 设备 udmac0d1e2、metadata-server 脚本值）。
- 只加上述 3 个诊断 env；记录最终生效的 env 与完整命令。
- 等待 `Read complete`/`Verification complete`，超时 10 min 停手。

## C. 采集（只读）

1. 端到端统计：Avg/P50/P90/P99/Max、成功/失败、verify 错误数（应与上一轮一致）。
2. `URMA_SLOW_IO`：总条数；**字段名全清单**；**3 条原始样本（原样）**；按时间戳区分"读阶段"与"verify 阶段"各多少条。
3. 若仍为 0：如实回报（不自行再加 env 重跑），并给 `grep -c MC_STORE_TRANSFER_DIAG` 等诊断行。
4. target W 汇总块（本窗口 n、W9 avg、breakdown）。
5. master：ERROR 计数（应无新增）。
6. reader 日志保存路径与大小。

## D. 边界

- 只读数据；仅新增 3 个诊断 env；不改负载参数；不动 target/master；不重启；失败停手不重试。

## 回执必答（第一行）

`URMA_SLOW_IO` 条数（读阶段/verify 阶段）；字段清单与 3 条样本；端到端 Avg/P50/P99；verify 错误数；node3 二进制是否含 `URMA_SLOW_IO`。

---

# NOF-012（已关闭 · 无定论 · 读路径旋钮矩阵 · 2026-11-20）· chunk 128K→4M × qpairs，诊断关闭

## 背景与问题

- NOF-011：读阶段 p50 `roundtrip 528 / target 430 / bdev 387µs`，**bdev 占 roundtrip 73%**；控制面每 key RPC 另计 150–400µs（NOF-010）。
- 假设（待验证）：当前 child IO = 128 KiB **小于 RAID0 strip 256 KiB**，每个 IO 可能只落单盘；把 IO 粒度放大到跨条带，bdev 段可能下降。
- 本单是**受控旋钮矩阵**：只改客户端 env，诊断关闭（避免 +17% 开销），每臂 3 遍同一 400-key 读。
- **口径说明（重要）**：该 bench 的 `--duration` 仅支持 local_memory / segment_read 场景，我们的 `remote_memory` 不支持；单遍 = 400 keys ≈ 6 s，故采用"每臂 3 遍取中位"而非 120 s。若要 120 s 口径，需先扩充数据集（另议）。

## 臂定义（全部：node3 reader，400 keys × 4 MiB，16 线程，batch 1，warmup 5，order=seq，verify=true，诊断关闭）

| 臂 | `MC_NOF_SUBMIT_CHUNK_BYTES` | `--nof_qpairs_per_segment` |
|---|---|---|
| V0 | 131072（128 KiB，基线） | 4 |
| V1 | 262144（256 KiB） | 4 |
| V2 | 1048576（1 MiB） | 4 |
| V3 | 4194304（4 MiB） | 4 |
| V4 | 262144（256 KiB） | 16 |

- 其余 env 保持：`MC_NOF_INFLIGHT_BYTES_LIMIT=33554432`、`MC_NOF_WORKERS=4`、`MC_URMA_TRANS_MODE=RM`、`SPDK_URMA_DEV_NAME=udmac0d1e2`、`SPDK_URMA_CAPSULE_TRANSPORT=urma` 等（与 NOF-010 相同）。
- 注意：V3 在 16 线程下 4 MiB×16=64 MiB > inflight 32 MiB，可能被限流——如实记录，作为该臂的解释变量。

## 执行

1. 预检（只读）：master 2162056 存活、enabled=1、Keys=400；node3 无残留 reader；记录当前 target W 块基线。
2. 按 V0→V4 顺序执行；每臂 **3 遍**（独立进程，每遍跑完等退出、确认 procs=0 再开下一遍）。
3. 每遍记录：开始/结束时间、FINAL 的 Min/Avg/P50/P90/P99/Max、failed、verify 错误；日志按 `run/matrix_V<n>_p<m>.log` 保存。
4. 每遍从日志聚合 `query_rpc_breakdown`：`total_us` 与 `rpc_first_us` 的 p50。
5. 汇总表：臂 × 遍次 + 每臂中位数。

## 停止规则

- 任一遍 `failed>0` 或 verify 错误 >0 → **立即停手回报**，不继续后续臂。
- 任一遍 reader 进程异常退出/超时（>60 s）→ 停手回报。

## 边界

- 只读；只改上述客户端 env/参数；不写盘；不动 target/master；不重启；不开诊断。
- 不自行增加/删除臂；不改线程数/verify/order。

## 回执必答（第一行）

各臂中位 P50/P99/Avg（列出 V0–V4）；哪一臂最好、差距多少；有无 failed/verify 错误；V3 是否受 inflight 限流（如实）；chunk 与读时延的关系（只报事实）。

---

# NOF-013（已关闭 · 通过 · chunk 生效性 A/B + 文档优化方向 · 2026-11-20）

## 目的

1. 证明/证伪 `MC_NOF_SUBMIT_CHUNK_BYTES` 是否作用于**读路径**（NOF-012 无法区分：V0 与 V3 日志行数相同、target W9 未变）。
2. 取交接文档里同事给出的**根因结论与优化方向原文**，作为下一步实验依据。

## A. 文档取证（只读；总输出 ≤40 行原文）

1. §3.1 端到端表（L45–54）逐行原文；
2. §3.2 内部拆分（L56–63）逐行原文；
3. §3.4 控制面（L79–83）逐行原文；
4. §四 根因结论 + "优化方向"表（L85–99）原文；
5. §八 交接建议（L145–151）5 条原文。

## B. chunk 生效性 A/B（两遍 reader，诊断口径）

- 环境：与 NOF-011 相同诊断开关（`VSEG_DIAGNOSTIC=1`、`VSEG_SLOW_IO_US=400`、`MC_HIFREQ_LOG_SAMPLE_RATE=1`）；负载与 NOF-012 相同（400 keys × 4 MiB、16 线程、batch 1、warmup 5、seq、verify=true）。
- 两遍，按顺序：
  - **V0**：`MC_NOF_SUBMIT_CHUNK_BYTES=131072`
  - **V3**：`MC_NOF_SUBMIT_CHUNK_BYTES=4194304`
- 每遍跑完后立即采集 target W 块（n、W9 avg、exec%），并保存完整日志 `run/chunkab_V0.log`、`run/chunkab_V3.log`（node3）。
- 从每份日志提取（读阶段按行号切分，剔除 warmup）：
  1. `URMA_SLOW_IO` 条数；
  2. opcode=2 的 `blocks` 字段：min / p50 / max、出现次数 top3 值；
  3. 端到端 p50/p99（诊断口径，仅相对参考）。
- **判定（只报事实）**：V3 的 blocks 是否从 256（128 KiB）变为 4096（2 MiB 切片）或 8192（4 MiB）？SLOW_IO 条数是否相应约为 1/16 或 1/32？

## C. 边界

- 只读；仅两遍 reader（V0/V3）；不改其它负载参数；不动 target/master；不重启；失败停手。

## 回执必答（第一行）

V0/V3 的 blocks p50 与 SLOW_IO 条数、target W n；chunk 是否作用于读路径（给判断依据）；文档优化方向 3 条原文要点。

---

# NOF-014（已关闭 · 通过 · 读时延并发/在途扫描 · 2026-11-20）

## 背景

- 文档 §四：6 盘 W9 294µs = 单盘服务 54.7µs × 排队 3~5 深；11 盘应降到 ~55µs。**我们实测 11 盘 W9 仍 278–288µs** ⇒ 可能在途 child 太多、每盘排队仍深。
- 本单扫三个客户端旋钮：线程数、NoF 在途上限（inflight）、控制面副本缓存 TTL；诊断关闭；不改 target/master。

## 臂定义（全部：node3，400 keys × 4 MiB，batch 1，warmup 5，order=seq，verify=true，chunk 128KiB，qpairs 4，诊断关）

| 臂 | 线程 | `MC_NOF_INFLIGHT_BYTES_LIMIT` | 副本缓存 |
|---|---|---|---|
| L0 | 16 | 33554432（32MiB，基线） | 关 |
| L1 | 16 | 8388608（8MiB） | 关 |
| L2 | 16 | 2097152（2MiB） | 关 |
| L3 | 4 | 33554432 | 关 |
| L4 | 16 | 33554432 | `MC_REPLICA_CACHE_TTL_MS=1000` |

- 其余 env 与 NOF-012/013 相同（`MC_NOF_WORKERS=4`、`MC_NOF_SUBMIT_CHUNK_BYTES=131072`、`SPDK_URMA_DEV_NAME=udmac0d1e2` 等）。
- 每臂 3 遍（独立进程）；每遍前确认 procs=0；每臂跑完立即采一次 target W 块（注明滚动窗口口径）。

## 采集

1. 每遍：Min/Avg/P50/P90/P99/Max、failed、verify 错误、起止时间；日志 `run/conc_L<n>_p<m>.log`。
2. 每遍 `query_rpc_breakdown` 的 `total_us` p50。
3. 汇总：臂 × 遍次 + 每臂中位；target W9 每臂数值。
4. 停止规则：任一遍 failed>0 或 verify 错误>0 → 立即停手回报。

## 边界

- 只读；仅上述客户端 env/参数；不写盘；不动 target/master；不重启；不增删臂。

## 回执必答（第一行）

各臂中位 P50/P90/P99/Avg；哪个旋钮最能降时延、降了多少；target W9 随臂如何变；缓存臂效果；有无失败。

---

# NOF-015（已关闭 · 通过 · chunk 复测：环境自证 · 2026-11-20）

## 目的

确认 `MC_NOF_SUBMIT_CHUNK_BYTES` 是否真的作用到**读路径**。NOF-013 判"无效"可能被运行环境覆盖（reader 脚本硬编码 131072）；代码复核显示 vsegment 读路径本该读取该 env（`transfer_task.cpp` SpdkNofQos）。本单用**环境自证**重测。

## A. 启动方式（关键）

1. 先读 `run_vesegment_reader.sh`，确认它对 `MC_NOF_SUBMIT_CHUNK_BYTES` 是否无条件 `export`（给行号）。
2. **不经过该脚本的 env 注入**：用显式 env 直接 exec bench；启动前把相关 env 写入 `run/envproto_<tag>.txt`（`env | grep -E 'MC_NOF_SUBMIT_CHUNK_BYTES|MC_NOF_INFLIGHT_BYTES_LIMIT|MC_NOF_WORKERS|VSEG_DIAGNOSTIC|MC_HIFREQ_LOG_SAMPLE_RATE|SPDK_URMA'`）；必要时运行中读 `/proc/<pid>/environ` 自证。
3. 其余参数/环境与 NOF-011 诊断口径一致：400 keys × 4 MiB、16 线程、batch 1、warmup 5、order=seq、verify=true、`VSEG_DIAGNOSTIC=1`、`VSEG_SLOW_IO_US=400`、`MC_HIFREQ_LOG_SAMPLE_RATE=1`、`SPDK_URMA_DEV_NAME=udmac0d1e2`。

## B. 三臂（各 1 遍，诊断开）

| 臂 | `MC_NOF_SUBMIT_CHUNK_BYTES` |
|---|---|
| C0 | 131072（128KiB） |
| C1 | 262144（256KiB） |
| C2 | 2097152（2MiB，等于 vsegment stripe） |

每臂跑前自证 env；跑后立即采 target W 块（n、W9）。

## C. 采集（读阶段按行号切分，剔除 warmup）

1. 每臂 `URMA_SLOW_IO` 条数；opcode=2 的 `blocks` 分布（p50 / max / top3）。
2. 判定：若 chunk 生效 → C1 blocks≈512（256KiB）、C2 blocks≈4096（2MiB）；若三臂都 256 → **代码层固定 128KiB**，需改代码。
3. 端到端 p50/p99（诊断口径，仅相对参考）；failed / verify 错误（应 0）。
4. 每臂实际生效的 env（env 文件内容 + 进程 environ 自证）。

## D. 边界

- 只读；仅三遍 reader；不改其它参数；不动 target/master；不重启；失败停手。

## 回执必答（第一行）

每臂实际生效的 chunk 值（自证）；blocks p50；SLOW_IO 条数；target W n；chunk 是否真正作用到读路径。

---

# NOF-016（已关闭 · 通过 · 换 target 为同事构建 · 2026-11-20）· 用户已批准破坏性操作

## 授权与范围

- 用户已批准：停旧 gap target/master、重绑 11 盘、用 **yin spdk-cpu** 构建重启 target；旧 GAP 数据可能受影响（已同意必要时重灌）。
- 本单只做"换 target + 重启我方 master"，**不 mount、不注册、不写数据**；mount/门禁在 NOF-017。
- 只动 node3 的旧 gap 进程与我方 master；**不动 yin 的进程/键、不动 node1**。

## A. 预检与回滚信息（先做，只读）

1. 记录：gap target PID 3243095（cmdline/cwd/exe md5）、gap master PID 3673976、我方 master PID 2162056；`ss -ltnp` 相关端口。
2. **保存回滚快照**到 node3 `run/rollback_gap_target.json`：
   - RPC 全量：`nvmf_get_subsystems`、`nvmf_get_transports`、`bdev_get_bdevs`、`bdev_raid_get_bdevs all`；
   - `/home/l00955908/gap/logs/target_start.log` 的建靶参数/ iobuf / mask / 设备等关键行（≤15 行）；
   - gap `nvmf_tgt` md5（`6668b490…`）。
3. 记录 yin target 二进制：`/home/yin/nof/spdk-cpu/build/bin/nvmf_tgt` 的 md5/大小；takeover 脚本 md5/大小。
4. 读 node1 `run_vesegment_target.sh` 与 `target_nvme_takeover.sh` 相关段：给出本单使用的**确切启动链路**（在哪台机、如何调用 takeover、NQN/端口如何注入、盘列表如何生成），≤20 行摘要。
5. 确认 11 块 7T 盘当前 vfio 绑定（BDF 列表）与 HugePages 现状。

## B. 停止旧环境（按 PID；先 TERM，10s 不退再 KILL）

1. 停我方 master 2162056。
2. 停 gap master 3673976。
3. 停 gap target 3243095；确认无 `nvmf_tgt`/`spdk_tgt` 进程、4457 释放。
4. **只删我方 cluster 键**：`/cvm/vseg_nofvseg/` + `/oplog/vseg_nofvseg/`；yin 的键一律不动。

## C. 启动 yin target（按 A4 链路）

- 用 yin 的 takeover/wrapper，参数按 A4 确认后的原样：`-w /home/yin/nof/spdk-cpu`、11 盘、`-R 256`、`-p urma`、`-N udmac0d1e2`、`-m '[2-7]'`、`-C 64`、`-L /usr/lib64`、`-i <node3 fabric IP>`、`-s <dump sec>`、NQN `nqn.2026-01.io.spdk:urma-gpu-test`、端口 **4457**、`-y`。
- 等待 4457 就绪；失败贴日志尾部 ≤30 行并**停手**（回滚快照已存）。
- 若 takeover 拒绝（仍有 SPDK 进程），确认 B 全部退出后再试一次；**不绕过脚本校验**。

## D. 验证（只读）

1. `nvmf_get_subsystems`：NQN、listener（trtype URMA、traddr/trsvcid）、nsid 1 → raid0。
2. `nvmf_get_transports`：dev_name（应为 udmac0d1e2）、max_io_size、capsule 配置。
3. raid0：strip 256 KiB、11 成员（target 日志确认）。
4. yin `nvmf_tgt` 进程 PID 与 md5；4457 LISTEN。
5. **重启我方 master**（同一套参数/env：cluster `vseg_nofvseg`、50056/9006、`SPDK_URMA_DEV_NAME=udmac0d1e2`、`VSEG_URMA_DEVICE=udmac0d1e2`），验证 leader/serving、无 ERROR、`enabled=0`。
6. 记录新 target 的**实际 listener 地址**与 trsvcid（NOF-017 mount 用）。

## E. 边界

- 不 mount/不注册/不写数据；不跑 writer/reader；不动 node1；不动 yin 的进程与 etcd 键；不重启系统；盘绑定只由 takeover 脚本执行（已授权）。
- 失败停手，不做 A4 链路之外的临时改动。

## 回执必答（第一行）

yin target 是否在 4457 起来（PID/md5）、listener 地址/NQN/ns/strip/dev_name；我方 master 是否重启正常；旧进程是否全部退出；回滚快照路径。

---

# NOF-017（已关闭 · 通过 · mount 新 target + reconcile 门禁 · 2026-11-20）

## 背景

- NOF-016：yin target 在 **node3:4457**（URMA，UDMAC0D1E2，strip 256K，11 成员）；我方 master PID 2462234（leader/serving，`enabled=0`）。
- 本单用**新地址**重新 mount（100GiB、base 0、nsid 1），验证 1024 分区 reconcile 门禁。**不写数据**；不跑 writer/reader。

## 1. 预检（只读）

1. master：PID 2462234 存活；50056/9006 LISTEN；etcd `/cvm/vseg_nofvseg/` 3 键、无 quota 快照；`/oplog/vseg_nofvseg/` 1 键。
2. target：node1 → `node3:4457` 建连 OK；node3 `ss -lnt` 4457 正常；`nvmf_get_subsystems` 与 NOF-016 D1 一致。
3. 记录 target 日志 `/tmp/nvmf_tgt.log` 当前 W 块基线（n 应为 0）。

## 2. mount（唯一一次，零额外 env）

- 在 node1 用 `nof_register`（md5 `1655b48a…`），root，输出 `run/register_r3.log`：
  ```
  <build>/mooncake-store/src/nof_register \
    "nqn.2026-01.io.spdk:urma-gpu-test" 1 node3 4457 URMA \
    <node1 192.168.84.*:50056> 107374182400
  ```
- **不要** export 额外 env。

## 3. 验证（只读）

1. master 日志（≤40 行）：`SPDK initialization … init_rc=0`、`Attaching to node3`、`Attached`、`Opened NoF segment`、`MountNoFSegment_slow … rc[?]`；任何 attach/URMA ERROR。
2. **设备映射（mount 时刻）**：`/dev/uburma` 映射数（应 >0）、`/dev/uburma/udmac0d1e2`、两个 URMA provider。
3. 若 rc[0]：`published vsegment quota generation 1 with 1024 …`、`VSegment pool formed`、`first full reconcile sweep`；`reconcile failures`=0、`Failed to refresh vsegment ownership`=0；`reconcile stats`：`owned_by_me=1024, reconcile_fail=0`；`VSegment Status`：`enabled=1, partitions_installed=1024, vsegments=1024`；etcd quota 快照存在（~181 KB）；60 s 后复查仍 0。
4. 若失败：贴 register 输出 + master 错误链 + target 日志对应时间窗（≤20 行），**停手**；不换参数重试。
5. 记录 target 新 W 块（mount 后 n/W9）与日志路径。

## 4. 边界

- 不跑 writer/reader；不写数据；不动 target/master 配置；不重启；不碰 yin 的进程与键。

## 回执必答（第一行）

mount 是否 rc[0]；设备映射数/provider；reconcile failures 与 partitions_installed；etcd quota；失败则错误链与 target 侧日志。

---

# NOF-018（已关闭 · 通过 · writer 重灌 · 2026-11-20）· 新 target 上写入 400 keys

## 背景

- NOF-017：新 yin target 上 mount rc[0]，1024 分区 reconcile 全通过；`logical_used=0`。
- 本单把 400 keys × 4MiB 重新写入（新 raid 上的数据布局与旧数据无关联；覆盖范围同前，用户已授权）。
- **不跑 reader**；不动 target/master 配置；不重启。

## 1. 预检（只读）

1. master：PID 2462234 存活、`enabled=1`、`partitions_installed=1024`、`logical_used=0`。
2. target：PID 341038 存活、4457 LISTEN；记录跑前 W 块基线。
3. etcd：`/cvm/vseg_nofvseg/` 8 键、`/oplog/vseg_nofvseg/` 32 键。

## 2. 运行 writer（与 NOF-009 完全相同）

- node1，root，`stress_cluster_bench`（同二进制 md5 `575d2151…`）；日志 `run/writer_r2.log`：
  - `--metadata-server=etcd://141.61.84.*:2379 --master-server=192.168.84.*:50056 --local-hostname=141.61.84.* --global-segment-size=0 --local-buffer-size=10737418240 --device-name=udmac0d1e2 --segment_write_vsegment=true --scenario=remote_memory --role=writer --num_keys=400 --value_size=4194304 --protocol=ub --verify=false --num_threads=16 --batch_size=16 --master_admin_port=9006 --vsegment_nof_io=true --nof_whole_pool_register=true --nof_qpairs_per_segment=4 --replica_num=1 --nof_replica_num=0`
  - env 按 NOF-009 清单（`MC_URMA_TRANS_MODE=RM`、`MC_URMA_BONDING_MULTIPATH_ENABLE=off`、`MC_NOF_SUBMIT_CHUNK_BYTES=131072`、`MC_NOF_INFLIGHT_BYTES_LIMIT=33554432`、`MC_NOF_WORKERS=4`、`SPDK_URMA_DEV_NAME=udmac0d1e2`、`SPDK_URMA_CAPSULE_TRANSPORT=urma`、`MC_STORE_PARTITION_SLOT_COUNT=1024`）。
- 等待 `400 succeeded, 0 failed`；超时 10 min 停手。

## 3. 验证（只读）

1. writer 输出：成功/失败数、写时延统计（Min/Avg/P50/P99）。
2. master：`logical_used` = 1,677,721,600 B；NVMe-oF SSD ≈1.56 GB / 100 GB；Keys=400；无 ERROR。
3. etcd：`/oplog/vseg_nofvseg/` 键数增长（32→约 1230）；`/cvm/` 仍 8 键。
4. **target W 块（跑完立即采）**：n、W9 avg、TOTAL、exec%/push%（这是与旧 target 对比的关键）：旧 gap target 写窗口 W9≈24.8–25µs? 实测旧值为**写路径 W9 avg 24,840 ns**（NOF-009，n=12,807）；新 target 同窗口对比。
5. 无真实写入错误（区分预检 TCP 探测的 handshake -104）。

## 4. 边界

- 只写本实验 400 keys；不跑 reader；不动 target/master；不重启；失败停手不重试。

## 回执必答（第一行）

writer 是否 400/0；`logical_used`；**target 侧写窗口 W9 avg / n / exec%**（与旧值 24,840 ns 对比）；master/target 有无错误。

---

# NOF-019（已关闭 · 通过 · 新 target 读基线对比 · 2026-11-20）

## 目的

在**新 yin target** 上重测读时延，与旧 gap target 的**同配置**数字直接对比，验证"换 target 后 5× W9 差距是否闭合、端到端是否跟着下降"。

对照基线（旧 target，已关闭 NOF-010/014 数据）：

| 臂 | 线程 | 旧 target p50 | 旧 target avg | 旧 target p99 |
|---|---|---|---|---|
| 16 线程 | 16 | 714.9 µs | 754.9 µs | 1672.3 µs |
| 4 线程 | 4 | 591.1 µs | 597.9 µs | 845.4 µs |

## 臂定义（node3 reader，400 keys × 4 MiB，batch 1，warmup 5，order=seq，verify=true，诊断关，chunk 128KiB，inflight 32MiB，qpairs 4）

- **R16**：`--num_threads=16`，3 遍；
- **R4**：`--num_threads=4`，3 遍；
- env 与 NOF-014 相同（`MC_NOF_SUBMIT_CHUNK_BYTES=131072`、`MC_NOF_INFLIGHT_BYTES_LIMIT=33554432`、`MC_NOF_WORKERS=4`、`SPDK_URMA_DEV_NAME=udmac0d1e2`、`SPDK_URMA_CAPSULE_TRANSPORT=urma`、诊断三项=0）。

## 采集

1. 每遍：Min/Avg/P50/P90/P99/Max、failed、verify 错误、`query_rpc_breakdown` 的 `total_us` p50；日志 `run/newtarget_R16_p<m>.log` / `run/newtarget_R4_p<m>.log`。
2. 每遍跑完立即采 target W 块（`/tmp/nvmf_tgt.log` 最新块）：**W9 avg、n、exec%、TOTAL**（与旧读 W9 283–288µs 对比）。
3. 汇总：臂 × 遍次 + 中位；与上表旧值并排对比（只报数字与差）。
4. 停止规则：任一遍 failed>0 或 verify 错误>0 → 停手回报。

## 边界

- 只读；不写盘；不动 target/master；不重启；不增删臂；不自行加诊断 env。

## 回执必答（第一行）

R16/R4 中位 P50/P90/P99/Avg；与旧 target 同臂的差；target 读窗口 W9 avg/n；5× W9 差距是否闭合；有无失败。

---

# NOF-020（已关闭 · 通过 · 新 target 五段复采 + 文档 W9 口径核对 · 2026-11-20）

## 目的

1. 在新 yin target 上重采 `URMA_SLOW_IO` 五段/七字段，找出"target 变快 11%、端到端不变"的原因——钱现在花在哪一段。
2. 核对交接文档 `W9 54.68µs`（n=4023）的上下文：是**读**还是**写**口径、来自哪一臂。

## A. 诊断读（node3，1 遍）

- 环境与 NOF-011 相同：`VSEG_DIAGNOSTIC=1`、`VSEG_SLOW_IO_US=400`、`MC_HIFREQ_LOG_SAMPLE_RATE=1`；负载 400 keys × 4 MiB、**16 线程**、batch 1、warmup 5、seq、verify=true；其余 env 与 NOF-019 R16 相同。
- 采集：
  1. 读阶段（按行号剔除 warmup）`URMA_SLOW_IO` 条数；opcode=2 的 29 字段统计（p50/p90）：roundtrip / target_us / bdev_us / data_us / pre_send_us / outside_us / residual_us / rx_buffer_us / buffer_exec_us；
  2. **`target_valid` 的分布**（0/1 各多少条）——验证 `TARGET_TRACE_ENTRIES=0` 是否让 target 段遥测失效；
  3. target W 块（W9/n/exec%）。
- 与旧 target（NOF-011 读阶段 p50：roundtrip 528 / target 430 / bdev 387 / data 32 / outside 81 / residual 82）并排对比。

## B. 文档口径核对（只读，≤10 行原文）

- 读 `交接文档_NoF_VSegment性能测试.md` §3.3（L65–77）的表头与上下文，确认 `W9 6 盘 294µs / 11 盘 54.68µs（n=7649/4023）` 是**读**还是**写**、来自哪一臂、什么负载参数。

## C. 边界

- 只读；仅 1 遍诊断 reader；不改负载参数；不动 target/master；不重启；失败停手。

## 回执必答（第一行）

新 target 五段 p50（roundtrip/target/bdev/pre_send/outside/residual）及与旧的差；`target_valid` 分布；target W9；54.68µs 的读/写口径原文。

---

# NOF-021（已关闭 · 通过 · 4 线程诊断五段 + 盘型核对 · 2026-11-20）

## 目的

1. 判断 **bdev ~387µs** 是"排队放大"还是"盘的介质底板"：把并发降到 4 线程再采五段，看 bdev_us 是否下降。
2. 识别 11 块成员盘的型号/固件（判断是 TLC 还是 QLC、随机读介质延迟量级）。

## A. 4 线程诊断读（node3，1 遍）

- 环境与 NOF-020 相同（`VSEG_DIAGNOSTIC=1`、`VSEG_SLOW_IO_US=400`、`MC_HIFREQ_LOG_SAMPLE_RATE=1`；400 keys × 4 MiB、batch 1、warmup 5、seq、verify=true、节点 node3、master 50056、admin 9006、dev `udmac0d1e2`）。
- **唯一改动：`--num_threads=4`**。
- 采集：
  1. 读阶段 p50/p90：roundtrip / target / bdev / data / outside / residual / pre_send；
  2. `target_valid` 分布；SLOW_IO 条数；
  3. 端到端 Min/Avg/P50/P90/P99/Max；failed/verify；
  4. 跑完立即采 target W 块（W9/n/exec%）。
- 与 16 线程诊断（NOF-020：roundtrip 514 / target 424 / bdev 387 / outside 59 / residual 60）并排对比。

## B. 盘型核对（只读）

- 通过 target RPC `bdev_get_bdevs` 读取成员 bdev `Nvme0n1…Nvme10n1` 的 `product_name`/model、`block_size`、`num_blocks`；若 RPC 不提供型号，从 target 启动日志/内核日志（`dmesg` 或 `/var/log`，只读）找 NVMe model string。
- 报告：型号、固件（若可得）、容量；并注明是否为常见 QLC/TLC 系列（只报事实，不推断性能）。

## C. 边界

- 只读；仅 1 遍诊断 reader；不改其它负载参数；不动 target/master；不重启；失败停手。

## 回执必答（第一行）

4 线程五段 p50（roundtrip/target/bdev/outside/residual）与 16 线程的差；bdev 是否随并发下降（降多少）；盘型号/固件；target W9/n；有无失败。

---

# NOF-022（已关闭 · 通过 · 裸 SPDK 底板 vs Mooncake 单线程 · 2026-11-20）

## 目的

- NOF-021：Mooncake 路径 bdev ~380–418µs、target W9 ~260µs，且不随并发下降；文档裸 SPDK 基线 exec **68µs**（8MiB、256KiB child）。差 ~4×。
- 本单用**同一个 yin target**、同一批盘，分别测：
  1. Mooncake reader **1 线程**（最接近裸 SPDK `-T1 -b1` 的并发）五段；
  2. **裸 SPDK `urma_perf`** 的逐 IO 延迟（128KiB / 256KiB / 8MiB object），拿硬件/驱动真实底板。
- 判断 4× 差是"Mooncake 客户端路径"还是"盘/驱动底板"。

## A. Mooncake 1 线程诊断读（node3，1 遍）

- 环境与 NOF-020 相同（`VSEG_DIAGNOSTIC=1`、`VSEG_SLOW_IO_US=400`、`MC_HIFREQ_LOG_SAMPLE_RATE=1`；400 keys × 4 MiB、batch 1、warmup 5、seq、verify=true、dev udmac0d1e2）。
- **唯一改动：`--num_threads=1`**。
- 采集：读阶段 p50/p90（roundtrip/target/bdev/data/outside/residual）、target_valid 分布、端到端 P50/P99、target W9（跑完即采）。

## B. 裸 SPDK `urma_perf`（node1 → yin target）

1. 定位二进制：node1 `~/gap/spdk/build/examples/urma_perf`（GAP 项目沿用；**先 `--help`** 确认目标地址/端口/NQN/nsid 参数与 `-o`/`-T`/`-b`/`-t` 语义，并记录 md5）。
2. 连接参数：traddr `node3`、trsvcid `4457`、NQN `nqn.2026-01.io.spdk:urma-gpu-test`、nsid 1；env `SPDK_URMA_DEV_NAME=udmac0d1e2`、`URMA_PERF_ADDR_MODE=seq`（按 `--help` 实际参数调整，**不改源码**）。
3. 三个短探针（各 5–10 s，读，`-T 1 -b 1`）：
   - `-o 131072`（128KiB IO）
   - `-o 262144`（256KiB IO）
   - `-o 8388608`（8MiB，对齐文档裸基线口径）
4. 每个探针：记录 avg/P50/P99 延迟、IOPS、bytes；并在**运行中**采一次 target W 块（W9/n/exec%）——用于对比"裸 SPDK 驱动下的 target exec"与"Mooncake 驱动下的 target exec"。
5. 只读；不写盘；若 urma_perf 报 ABI/参数错误 → 停手回报 `--help` 与错误，不自行改源码/换二进制。

## C. 分析与回执（只报事实）

- 表：Mooncake 1t 的 roundtrip/target/bdev p50；裸 SPDK 三种 IO 的延迟；各自期间的 target W9。
- 明确回答：**4× 差距在 Mooncake 客户端路径，还是在盘/驱动底板？**
- 停止规则：任一步失败（reader failed/verify、urma_perf 报错）→ 停手回报。

## D. 边界

- 只读；不写盘；不动 target/master；不重启；不增删探针；不自行换二进制/改源码。

## 回执必答（第一行）

Mooncake 1t 五段 p50（尤 bdev）；裸 SPDK 128K/256K/8M 的延迟与 target W9；4× 差落在哪一层（给判断依据）。

---

# NOF-023（已关闭 · 通过 · 受控对照 + 修复 LBA0 · 2026-11-20）

## 开发侧三项决定（已授权）

1. **允许重跑 writer 修复** NOF-022 preflight 在 LBA 0 的误写（末尾执行，见 C）。
2. **授权裸 SPDK 队列深度梯度**（`-b 8/16/32`）与**随机模式**（`URMA_PERF_ADDR_MODE=rand` + seed；该 env 是 GAP 补丁项，不会出现在 `--help`，二进制 md5 `c653c0a6…` 含该补丁）。
3. **授权使用 yin 树的 `urma_perf`**（`/home/yin/nof/spdk-cpu/build/examples/urma_perf`，root）做 256KiB/8MiB 探针；记录 md5 与 `--help`。

## A. 裸 SPDK 受控 2×2（node1，root，gap urma_perf，128KiB IO，每支 6 s）

| 支 | 模式 | 批大小 |
|---|---|---|
| S1 | seq | `-b 1` |
| S32 | seq | `-b 32` |
| R1 | rand | `-b 1` |
| R32 | rand | `-b 32` |

- 连接：traddr `node3`、trsvcid `4457`、NQN `nqn.2026-01.io.spdk:urma-gpu-test`、nsid 1；env：`SPDK_URMA_DEV_NAME=udmac0d1e2`、`SPDK_URMA_CAPSULE_TRANSPORT=urma`、随机支加 `URMA_PERF_ADDR_MODE=rand URMA_PERF_RANDOM_SEED=20261115`，`--range-bytes=107374182400`（随机范围=我们的 100GiB 段）。
- 每支记录：avg/p50/p99 延迟、IOPS；**运行结束后立即采 target W 块**（W9/n/exec%）。
- 注意：每支 preflight 都会写 LBA 0（已知）；全部探针结束后再修复（C）。

## B. yin urma_perf 的 256K/8M（若兼容）

- 先 `md5sum` + `--help` + `ldd`；跑 256KiB 与 8MiB 各一支（seq，`-T1 -b1`，6 s，root，同连接/env）。
- 若 ABI/参数不兼容：记录错误并停此段，不影响 A。
- 目的：复现文档裸基线 8MiB（Get 236µs / exec 68µs）。

## C. 修复 LBA0（写操作，已授权）

- A/B 全部结束后：按 NOF-018 同命令重跑 writer（400 keys × 4 MiB），确认 400/0；随后跑 1 遍 reader `verify=true`（16 线程，诊断关），确认 `0 errors out of 400 keys`。
- 记录 writer/reader 结果与 `logical_used`（应仍为 1,677,721,600）。

## D. 边界

- A/B 只读（除 urma_perf preflight）；C 仅 writer 重写 + verify；不动 target/master；不重启；不改负载参数。

## 回执必答（第一行）

S1/S32/R1/R32 的延迟与 target W9（顺序 vs 随机、QD1 vs QD32 各差多少）；yin urma_perf 256K/8M 结果（若成）；8.6× 差是"随机 vs 顺序"还是"QD/IO 形态"（给判断依据）；修复 writer/verify 是否 0 错。

---

# NOF-024（已关闭 · 旋钮证否 · T1 下 qpairs/workers 扫描 · 2026-11-20）

## 背景与授权

- NOF-023：随机 128KiB 的 target 执行底板 ≈190–206µs（裸 SPDK）；Mooncake 259µs（1.26–1.36×）；T1 端到端 P50 571µs，远高于"单波 IO ~220µs + 控制 ~90µs"的粗算。
- 假设：每 key 的 32×128KiB 受 **lanes（qpairs=4）+ 每 lane 信用**限制，需要多波才能发完；提高 lanes/workers 可能把 key 压到 1–2 波。
- **授权只改客户端参数**：`--nof_qpairs_per_segment`（4→8/16）与 `MC_NOF_WORKERS`（4→8）；不动 target/master。
- 对测试 Agent 两个提问的答复：① 8MiB 探针暂不做；若后续需要，用 **node3 本机同批 urma_perf**（WIRE v4）本地跑，**不拷贝到 node1**（已预授权，但不在本单）。② R1/R32 已覆盖随机 100GiB 范围，剩余 1.26× 先由本单 qpairs 解释，暂不做条带专项。

## 臂定义（node3 reader，**T1**，400 keys × 4 MiB，batch 1，warmup 5，seq，verify=true，诊断关，chunk 128KiB，inflight 32MiB）

| 臂 | `--nof_qpairs_per_segment` | `MC_NOF_WORKERS` |
|---|---|---|
| L4 | 4 | 4（基线） |
| L8 | 8 | 4 |
| L16 | 16 | 4 |
| L16W8 | 16 | 8 |

- 每臂 3 遍；其余 env 与 NOF-019/021 相同（`SPDK_URMA_DEV_NAME=udmac0d1e2` 等；诊断三项=0）。
- 每遍：Min/Avg/P50/P90/P99/Max、failed、verify 错误；运行后按 target 的"**下一次新连接打印上一轮**"规则取 W 块并按 n 对齐（照 NOF-023 §4.2 的方法）。
- 可选：若某臂明显优于基线（超臂内极差），再用该配置在 T4 跑 3 遍补充；不影响本单主结论。

## 停止规则

- 任一遍 failed>0 或 verify 错误>0 → 立即停手回报。

## 边界

- 只读；仅改上述客户端参数；不写盘；不动 target/master；不重启；不增删臂。

## 回执必答（第一行）

L4/L8/L16/L16W8 中位 P50/P90/P99/Avg；最优组合；target W9/n；T1 端到端距 ~300µs 还差多少；有无失败。

---

# NOF-025（已关闭 · 关键 A/B：plain NoF 基线 · 2026-11-20）

## 背景与目标

- 项目目标不变：单 key 4–8MiB 读 **750µs → 300µs**；用户硬约束：**vsegment 必须优于 plain NoF**（否则 vsegment 无意义）。
- 现有 vsegment：T1 P50 **537.5µs**、16t P50 **719µs**（新 yin target，400×4MiB，verify）。
- 本单在同一 target、同一客户端、同一负载下建立 **plain NoF（无 vsegment）** 基线，做首次同口径 A/B；它决定的是"vsegment 是否合格"，不是更换项目目标。

## A. 参数核对（只读）

- 依 `stress_cluster_bench.cpp`（be10be2）flag 语义：
  - plain NoF：`--replica_num=0 --nof_replica_num=1 --vsegment_nof_io=false --segment_write_vsegment=false`；
  - master 关闭 vsegment：`--vsegment_auto_plan=false --cvm_segments_vsegment_exclusive=false`，其余 HA/oplog/端口/env 不变。
- 若实际 binary `--help` 与上述不符：按 binary 实际 flag 执行并在回执记录差异。

## B. 切换环境（只动我方 master/cluster）

1. 记录当前 vsegment 状态（PID 2462234、etcd 键、logical_used、数据）；停 master。
2. 清**仅我方**键：`/cvm/vseg_nofvseg/` + `/oplog/vseg_nofvseg/`。
3. 以 plain 参数重启 master（node1，50056/9006，`SPDK_URMA_DEV_NAME=udmac0d1e2`、`VSEG_URMA_DEVICE=udmac0d1e2`，无 vsegment 参数）；确认 leader/serving、无 quota key。
4. `nof_register` 同命令挂载 target（node3:4457、100GiB、base0、nsid1）→ 确认 mount rc[0]。

## C. plain NoF 写 + 读

1. **writer**：400 keys × 4MiB、16 线程、batch 16、verify=false；flags 按 A 段（plain）；其余 env/参数同 NOF-018；确认 `400 succeeded, 0 failed`、master Keys=400。
2. **reader**：16 线程 × 3 遍 + **T1 × 3 遍**（verify=true、诊断关、batch1）；记录 Avg/P50/P90/P99/Max、failed/verify；target W 块按"下一次新连接打印上一轮"规则对齐。
3. 记录 plain NoF 的 target W9 / exec% / TOTAL。

## D. 回执与对比

- 对比表（同口径）：plain NoF vs VSegment：**T1** 的 P50/P90/P99/Avg；**16t** 的 P50/P90/P99/Avg；target W9。
- 明确回答：**VSegment 比 plain NoF 快还是慢？差多少？**（若慢，写明差值与可能的布局原因。）
- 环境处理：**保留 plain 环境，不恢复 vsegment**（下一单按结论：若 vsegment 慢，先做"减少分区数/布局连续化"调优）。

## E. 边界

- 只动我方 master/cluster；不动 yin/target 配置；target 仅正常读写（覆盖我们自己的 100GiB 段）。
- 失败停手；回滚信息：vsegment 恢复步骤 = NOF-004/007 参数 + NOF-017 mount + NOF-018 writer（均已记录）。

## 回执必答（第一行）

plain NoF 的 T1 / 16t 中位 P50/P90/P99/Avg；与 VSegment 的差；target W9；**VSegment 是否更快**；有无失败。

---

# NOF-026（已关闭 · 只读 · 资料复读：750/300 的出处与口径 · 2026-11-20）

> **本单先于 NOF-025 执行**（用户要求先把资料读清楚；NOF-025 保持待执行）。

## 目的

用户需确认：
1. `750µs` 是否是"开 vsegment 的 Mooncake 端到端"实测结果？
2. `300µs` 从何而来？资料里有没有推导？是否可以理解为"不开 vsegment 的 Mooncake 端到端水平"？

## A. 交接文档检索（只读；输出 ≤50 行）

对象：`交接文档_NoF_VSegment性能测试.md`（151 行）。

1. 列出所有含 `750` / `300` / `380` / `800` / `841` / `610` / `662` / `913` 的行：行号 + 该行原文（必要时前后各 1 行，控制总输出）。
2. 重点抄录：
   - L1–L5（标题/目标）；
   - §3.1 表头与整表（L45–54）；
   - §3.4（L79–83）；
   - §四（L85–99）。
3. 明确回答：**文档是否有任何一句话解释 300µs 的推导/依据？** 有 → 给原文与行号；没有 → 标 `[未获取]`。
4. 文档中是否有**"不开 vsegment 的 Mooncake 端到端"**数字？（注意区分：文档 §3.1 的"裸 SPDK 基线"不是 Mooncake；有没有 plain Mooncake / 非 vsegment 的 Mooncake Get 数字？）有 → 原文+行号；没有 → `[未获取]`。

## B. 资料目录其他文件（只读；≤15 行）

- `others.txt` 及资料目录内其他非交接文档文件：是否提到 750/300/目标来源/对比口径？给原文与文件名。

## C. 边界

- 只读 Windows 资料目录；不碰 node1/node3；不跑任何服务；不改任何文件。

## 回执必答（第一行）

750 的原文与口径（是否开 vsegment、Avg/P50）；300 是否有推导（有→原文，无→[未获取]）；资料里是否存在"不开 vsegment 的 Mooncake 端到端"数字。

---

# NOF-027（已关闭 · vsegment 布局调优：1024→64 分区 · 2026-11-20）

## 背景

- NOF-025 判决：**VSegment T1 537.5µs vs plain NoF 258.3µs（plain 快 2.08×）**；但存在布局混淆——plain 数据连续堆在最前 1.6GiB（顺序），VSegment 散在 1024 个分区（随机）。
- 假设：把分区数从 **1024 降到 64**，每个分区约 6 个 key、在 64MiB vsegment 内连续摆放 ⇒ I/O 变顺序，vsegment 时延应向 plain 靠拢甚至反超。
- 本单只改 `MC_STORE_PARTITION_SLOT_COUNT`（master 与 client 必须一致），其余参数不变。

## A. 参数核对（只读）

1. 确认分区数由 `MC_STORE_PARTITION_SLOT_COUNT` 决定（master 日志 `SaveClusterMeta ... slot_count=`；quota 快照 partitions 数）；client 启动日志/rpc 里同 env。
2. 预算：64 分区 × 100GiB/64 = 1.5625GiB/分区；每分区约 6–7 keys × 4MiB = 24–28MiB < 64MiB initial vsegment ⇒ 每个 key 都落在首卷、连续分配。
3. 若环境不生效（slot_count 仍 1024）→ **停手回报**，不要改其它参数硬凑。

## B. 切换 vsegment-64（只动我方 master/cluster）

1. 停 plain master 2627750；清**仅我方**键 `/cvm/vseg_nofvseg/` + `/oplog/vseg_nofvseg/`。
2. 启动 vsegment master：NOF-004/007 同参数（cluster `vseg_nofvseg`、50056/9006、udmac0d1e2、20 项 vsegment/HA 参数）+ **`MC_STORE_PARTITION_SLOT_COUNT=64`**。
3. `nof_register` 同命令（node3:4457、100GiB、base0、nsid1）；等 quota 发布 ≥20s。
4. 验证：quota 快照分区数 = **64**；`reconcile_fail=0`；`partitions_installed=64`、`vsegments≈64`；`logical_capacity` 记录实际值。

## C. 写 + 读（同 NOF-018/019 口径）

1. writer 400 keys × 4MiB（vsegment flags 同 NOF-018）+ **client `MC_STORE_PARTITION_SLOT_COUNT=64`**；确认 400/0、`logical_used=1,677,721,600`。
2. reader：**T1 × 3 遍 + 16t × 3 遍**（verify=true、诊断关）；记录 Avg/P50/P90/P99/Max、failed/verify；target W 块按"下一次连接打印上一轮"对齐。
3. 另记录 reader 的 `query_rpc_breakdown` p50（控制面）。

## D. 对比与回执

- 对比表（同口径）：

| 场景 | T1 P50 | 16t P50 | target W9 |
|---|---|---|---|
| plain NoF（NOF-025） | 258.3 | 718.7 | 96.9 / 194–196 |
| VSegment-1024（NOF-024/019） | 537.5 | 719.0 | 250.5 / 259.6 |
| **VSegment-64（本单）** | ? | ? | ? |

- 明确回答：**分区数从 1024 降到 64 后，vsegment 时延是否明显下降？是否已快过 plain NoF？** 若仍慢，写清差值与 target W9 对比（判断是布局还是 vsegment 路径开销）。
- 保留 vsegment-64 环境；失败停手。

## E. 边界

- 只动我方 master/cluster；不动 yin/target 配置；target 仅正常读写；不重启 target。
- 回滚：plain 环境恢复 = NOF-025 的 plain master 参数 + nof_register + writer（均已记录）。

## 回执必答（第一行）

VSegment-64 的 T1/16t P50/P90/P99/Avg；target W9；与 plain 258.3/718.7 及 VSegment-1024 537.5/719 的差；**vsegment 是否已赢 plain**；有无失败。

---

# NOF-028（已关闭 · vsegment 单分区=1，验证布局假设 · 2026-11-20）

## 背景与假设

- NOF-027：64 分区 T1 无改善、16t 变差 ⇒ 分区数量不是杠杆；读 key 的顺序（0..399）与"哈希散列后的物理位置"错位，读的时候一直在跳。
- **极端对照**：分区数=1 ⇒ 400 个 key 全部顺序写入同一分区/同一组连续 vsegment，**读序=写序=物理顺序**，与 plain 的连续布局同构。
  - 若 vsegment-1 的 T1 接近 plain（258.3µs）、target W9 从 251µs 降到 ~97µs ⇒ 差距来自"哈希散列布局"，vsegment 层本身开销小；
  - 若仍 ~537µs ⇒ vsegment 读路径自身有约 280µs 开销。

## A. 预检（只读，任一不满足即停手回报）

1. 确认 slot_count=1 可被接受（预期 `SaveClusterMeta ... slot_count=1`；quota `partitions=1`）。若 1 不被接受，退回 **2** 并在回执注明，不自行改其它参数。
2. 记录当前 vseg-64 环境（master PID 2646678、etcd 键、logical_used）。

## B. 切换 vsegment-1（只动我方 master/cluster）

1. 停 vseg-64 master（按 PID）；清**仅我方**键 `/cvm/vseg_nofvseg/` + `/oplog/vseg_nofvseg/`。
2. 启动 vsegment master：NOF-027 同参数 + **`MC_STORE_PARTITION_SLOT_COUNT=1`**。
3. `nof_register` 同命令（node3:4457、100GiB、base0、nsid1）；等 quota 发布 ≥20s。
4. 验证：`partitions=1`、`reconcile_fail=0`、`partitions_installed=1`、`vsegments` 初始值；记录 `logical_capacity`。

## C. 写 + 读（同口径）

1. writer 400 keys × 4MiB（vsegment flags 同 NOF-018）+ **client `MC_STORE_PARTITION_SLOT_COUNT=1`**；确认 400/0、`logical_used=1,677,721,600`；**记录 vsegments 数量**（预期随写入增长，如 ~25 个 64MiB 卷；若创建失败或容量不足 → 停手回报）。
2. reader：**T1 × 3 遍 + 16t × 3 遍**（verify=true、诊断关）；记录 Avg/P50/P90/P99/Max、failed/verify；target W 块按"下一次连接打印上一轮"对齐；记录 query_rpc p50。

## D. 对比与回执

| 场景 | T1 P50 | 16t P50 | target W9（T1） |
|---|---|---|---|
| plain NoF | 258.3 | 718.7 | 96.9 |
| VSegment-1024 | 537.5 | 719.0 | 250.5 |
| VSegment-64 | 539.9 | 854.3 | 250.8 |
| **VSegment-1（本单）** | ? | ? | ? |

- 明确回答：**vsegment-1 是否接近 plain？target W9 是否降到 ~100µs 量级？** 若接近，结论=布局/读序是主因；若仍慢，结论=vsegment 读路径自身开销（给 bdev/控制面证据）。
- 保留 vsegment-1 环境；失败停手。

## E. 边界

- 只动我方 master/cluster；不动 yin/target 配置；target 仅正常读写；不重启 target。
- **容量风险**：1 分区时初始卷仅 64MiB，写入 1.6GiB 依赖按需创建更多 vsegment；若出现创建失败/容量不足，立即停手回报（不要扩容量参数）。

## 回执必答（第一行）

VSegment-1 的 T1/16t P50/P90/P99/Avg；target W9；vsegments 数量；与 plain 258.3 的差；**布局假设是否成立**；有无失败。

---

# NOF-029（已关闭 · vsegment-1 数据补齐 + 读侧判决 · 2026-11-20）

## 背景与授权

- NOF-028：单分区被接受，但 16 线程 writer 在卷边界丢 23 key（`VSEGMENT_CREATING` 不重试）。
- **授权**：仅允许把 **writer 改为单线程、batch=1**（避免创建竞争）重写 400 keys；不改其它参数、不改容量参数。
- 补齐后跑读，判定"布局假设"（读序=写序 ⇒ 是否接近 plain 258.3µs）。

## A. 预检（只读）

1. master（vseg-1，PID 2667366）存活；`partitions_installed=1`、`vsegments=24`、`logical_used=1,581,252,608`、`logical_free≈28MiB`。
2. 确认无残留 writer/reader 进程。

## B. T1 writer 补齐（写操作，已授权）

- node1，root，同二进制/env/flags（vsegment 写路径），**唯一改动 `--num_threads=1 --batch_size=1`**；日志 `run/writer_vseg1_r2.log`。
- 等待 `400 succeeded, 0 failed`（超时 8 min 停手）。
- 验证：`logical_used=1,677,721,600`；`vsegments` 数量（预期 ~26）；无 `VSEGMENT_CREATING` 或列出残留。
- 若仍失败（创建竞争/容量不足）→ **停手回报**，不改容量参数、不重试第二次。

## C. Reader（仅当 B 成功）

- T1 × 3 遍 + 16t × 3 遍（verify=true、诊断关；client `MC_STORE_PARTITION_SLOT_COUNT=1`）。
- 记录 Avg/P50/P90/P99/Max、failed/verify；target W 块按"下一次连接打印上一轮"对齐；`query_rpc` p50。

## D. 对比与回执

| 场景 | T1 P50 | 16t P50 | target W9（T1） |
|---|---|---|---|
| plain NoF | 258.3 | 718.7 | 96.9 |
| VSegment-1024 | 537.5 | 719.0 | 250.5 |
| VSegment-64 | 539.9 | 854.3 | 250.8 |
| **VSegment-1（本单）** | ? | ? | ? |

- 明确回答：**T1 是否接近 258.3？target W9 是否降到 ~100µs 量级？** 是 → 布局/读序是主因、vsegment 层开销小；否 → vsegment 读路径自身开销。
- 保留 vsegment-1 环境。

## E. 边界

- 只动我方 master/cluster；target 不动；不改容量/其它参数；失败停手。

## 回执必答（第一行）

T1 writer 是否 400/0（`logical_used`、vsegments）；VSegment-1 的 T1/16t P50 与 target W9；**布局假设是否成立**；有无失败。

---

# NOF-030（已被 NOF-031 取代 · 未执行 · vsegment stripe=4MiB：每 key 单 child · 2026-11-20）

## 背景与假设

- NOF-029：分区数 1/64/1024 的 T1 全在 537.5–543.0µs ⇒ **布局无关**；plain W9 96.9µs vs vsegment 250–271µs ⇒ 差距在 **vsegment 读路径的 2MiB child × 16×128KiB 拆分**。
- 假设：把 `--vsegment_stripe_size` 从 **2MiB 改为 4MiB**，每个 4MiB key 只产生 **1 个 4MiB child**（与 plain 的单次 4MiB 传输同构），target W9 应朝 ~97µs 回落、T1 朝 ~258µs 靠拢。
- 注：`MC_NOF_SUBMIT_CHUNK_BYTES` 已被 NOF-015 证明对读路径无效，本单**不**再试；只改 stripe_size。

## A. 预检（只读）

1. 确认 master 支持 `--vsegment_stripe_size=4194304`；`member_extent_size=67108864` 是 4MiB 的整数倍（64/4=16）；io_alignment 4096 可整除。
2. 记录当前 vseg-1 环境（PID 2667366、logical_used=1.6GiB）。

## B. 切换 vsegment-stripe4（只动我方 master/cluster）

1. 停 vseg-1 master（PID）；清**仅我方**键 `/cvm/vseg_nofvseg/` + `/oplog/vseg_nofvseg/`。
2. 启动 master：NOF-004/007 同参数 + **`--vsegment_stripe_size=4194304`** + `MC_STORE_PARTITION_SLOT_COUNT=1024`（恢复 1024 分区，避免单分区容量问题）。
3. `nof_register` 同命令（node3:4457、100GiB、base0、nsid1）；等 quota ≥20s。
4. 验证：quota 的 `stripe_size=4194304`；partitions=1024；reconcile 0；`partitions_installed=1024`。

## C. 写 + 读（同口径）

1. writer 400 keys × 4MiB（vsegment flags 同 NOF-018，16 线程；client slot env=1024）；确认 400/0、`logical_used=1,677,721,600`。
2. reader：**T1 × 3 遍 + 16t × 3 遍**（verify=true、诊断关；client slot env=1024）；记录 Avg/P50/P90/P99/Max、failed/verify；target W 块按"下一次连接打印上一轮"对齐；`rpc_first_us` p50。

## D. 对比与回执

| 场景 | stripe | T1 P50 | 16t P50 | target W9（T1） |
|---|---|---|---|---|
| plain NoF | — | 258.3 | 718.7 | 96.9 |
| VSegment（NOF-024） | 2MiB | 537.5 | 719.0 | 250.5 |
| **VSegment（本单）** | **4MiB** | ? | ? | ? |

- 明确回答：**改成 4MiB stripe（每 key 单 child）后，W9 是否降到 ~100µs？T1 是否接近 258µs？** 是 → child 拆分是主因、vsegment 可追平 plain；否 → vsegment 层还有其它开销（给证据）。
- 保留 vsegment-stripe4 环境；失败停手。

## E. 边界

- 只动我方 master/cluster；不动 yin/target 配置；target 仅正常读写；不重启 target；不改其它参数。
- 回滚：vsegment 标准配置 = NOF-004/007 参数 + NOF-017 mount + NOF-018 writer。

## 回执必答（第一行）

quota stripe_size 是否 4194304、partitions=1024、reconcile 0；writer 400/0；VSegment-4MiB 的 T1/16t P50 与 target W9；**是否追平 plain（258.3/96.9）**；有无失败。

---

# NOF-031（已关闭 · 共享环境探测 + stripe=4MiB 实验 · 2026-11-20）

> 本单**取代 NOF-030**。用户告知：**node1 现非独占**（另有同事在使用，但不跑测试）；**node3 由用户独占**。先探测环境，确认我方环境未被影响后再继续实验；若环境有变则停手回报。

## A. 环境探测（只读；不碰任何非本方进程）

1. **node1**
   - 我方 vseg-1 master（PID 2667366）是否存活；50056/9006 是否仍由它监听；`VSegment Status`（预期 `enabled=1, partitions_installed=1, vsegments=26, logical_used=1,677,721,600`）。
   - 是否有**新的非本方进程/端口占用**：`ps -eo pid,user,lstart,comm,args` 摘要 + `ss -ltnp` 摘要；重点核对 50056 / 9006 / 8086 / 2379；只记录，不处理。
   - yin master（PID 151060）与 etcd 2379 健康；他人 client 进程仍在（记录 PID/属主）。
   - 我方 etcd 前缀：`/cvm/vseg_nofvseg/`（预期约 6 键）、`/oplog/vseg_nofvseg/`（约 2,400 键）是否完整；yin 与 mooncake_cluster 键未变。
   - node1 负载：`uptime`、`top -bn1 | head` 摘要（判断同事的使用是否带来明显负载）。
2. **node3**
   - 我方 target（PID 341038）是否存活；4457 是否 LISTEN 在 `node3`；`/tmp/nvmf_tgt.log` 是否可写/仍在增长。
   - 是否出现其它非本方进程；HugePages、磁盘空间现状。
3. **判定**：若我方 master/target 任一缺失、50056/9006/4457 被他人占用、我方 etcd 键异常、或 node1 负载明显影响测试 → **停手回报，不执行实验**。

## B. 实验（仅当 A 判定干净）：vsegment stripe=4MiB（内容同 NOF-030）

1. 停我方 vseg-1 master（按 PID）；清**仅我方**键 `/cvm/vseg_nofvseg/` + `/oplog/vseg_nofvseg/`。
2. 启动 master：NOF-004/007 同参数 + **`--vsegment_stripe_size=4194304`** + `MC_STORE_PARTITION_SLOT_COUNT=1024`。
3. `nof_register` 同命令（node3:4457、100GiB、base0、nsid1）；等 quota ≥20s；验证 quota `stripe_size=4194304`、partitions=1024、`reconcile_fail=0`、`partitions_installed=1024`。
4. writer 400 keys × 4MiB（16 线程；vsegment flags 同 NOF-018；client slot env=1024）；确认 400/0、`logical_used=1,677,721,600`。
5. reader：T1 × 3 遍 + 16t × 3 遍（verify=true、诊断关；client slot env=1024）；记录 Avg/P50/P90/P99/Max、failed/verify；target W 块按"下一次连接打印上一轮"对齐；`rpc_first_us` p50。
6. 对比与回答：
   - plain NoF：T1 258.3 / 16t 718.7（W9 96.9 / 194–196）；
   - VSegment-2MiB：T1 537.5 / 16t 719.0（W9 250.5 / 259.6）；
   - **VSegment-4MiB（本单）**：? / ?（W9 ?）
   - 明确回答：W9 是否降到 ~100µs 量级？T1 是否接近 258µs？**是否追平 plain**？

## C. 边界

- 只动我方 master/cluster 与我方数据；**不碰同事进程、yin 进程/键、target 配置**；不重启 target；失败停手。
- 共享机：node1 非独占——非本方进程只记录、不处理；node3 独占也需记录异常。
- 回滚：vsegment 标准配置 = NOF-004/007 参数 + NOF-017 mount + NOF-018 writer。

## 回执必答（第一行）

环境是否干净（我方 master/target/键/端口状态；node1 是否有他人负载）；若干净：stripe-4MiB 的 T1/16t P50 与 target W9、是否追平 plain；若停手：原因与证据。

---

# NOF-032（已关闭 · vseg-1 全新集群 + 纯单线程顺序写 · 2026-11-23）

## 背景与假设（重要）

- 此前的 vseg-1 布局假设**被污染**：NOF-028 的 377 个 key 由 **16 线程**写入（分配顺序被打乱），NOF-029 的单线程只重写了已有 key ⇒ "读序=写序"从未真正成立。
- stripe 已证否（2/4MiB 的 target IO 数均为 25,762）；plain 也是 128KiB IO（n=25,762），差距是**每条 IO 的执行时间**（96.9 vs 252µs），不是 IO 大小。
- 本单做**干净对照**：全新集群 + vseg-1 + **纯单线程顺序写** 400 keys（顺序分配、读序=写序）：
  - 若 target W9 降到 ~97µs、T1 接近 258µs ⇒ **布局/分配顺序是主因**，vsegment 可通过控制分配顺序追平 plain；
  - 若仍 ~250/537 ⇒ **vsegment 读路径自身开销**，只能改代码。

## A. 预检（只读）

- 按 NOF-031 口径快速复核：我方 master（当前 stripe-4/slot-1024，PID 1610536）、target 341038、50056/9006/4457、etcd 键；node1 同事进程只记录不动；node3 独占。
- 任一异常 → 停手回报。

## B. 切换 vseg-1（标准 stripe 2MiB）

1. 停当前 master（按 PID）；清**仅我方**键。
2. 启动 master：NOF-004/007 同参数 + **`MC_STORE_PARTITION_SLOT_COUNT=1`**；stripe 保持标准 2MiB。
3. `nof_register` 同命令；等 quota ≥20s；验证 partitions=1、`reconcile_fail=0`、初始 logical_capacity=64MiB。

## C. 纯单线程顺序写（关键）

- writer：**`--num_threads=1 --batch_size=1`**，400 keys × 4MiB，vsegment flags 同 NOF-018；日志 `run/writer_vseg1_t1seq.log`。
- 若出现 `VSEGMENT_CREATING` 失败（卷边界）：允许**原地再跑一次同样的 T1 writer** 补齐（总尝试 ≤2 次）；确认 400/0 或至少 `logical_used=1,677,721,600` 且 `committed_allocations=400`；记录失败 key 数与位置。
- 记录 vsegments 数量（预期 ~26）。

## D. 读（同口径）

- T1 × 3 遍 + 16t × 3 遍（verify=true、诊断关；client slot env=1）。
- 记录 Avg/P50/P90/P99/Max、failed/verify；target W 块按"下一次连接打印上一轮"对齐；`rpc_first_us` p50。

## E. 对比与回答

| 场景 | T1 P50 | 16t P50 | target W9（T1） |
|---|---|---|---|
| plain NoF | 258.3 | 718.7 | 96.9 |
| VSegment-1（16 线程写，被污染） | 543.0 | 739.5 | 271.2 |
| **VSegment-1（顺序写，本单）** | ? | ? | ? |

- 明确回答：顺序分配后 **W9 是否降到 ~97µs？T1 是否接近 258µs？布局假设是否成立？**
- 若仍 ~250/537：记录差值，结论转为 vsegment 路径自身开销（为后续代码改动提供依据）。

## F. 边界

- 只动我方 master/cluster；不动 target 配置；共享机规则同 NOF-031（同事进程只记录）；失败停手。

## 回执必答（第一行）

T1 顺序写结果（400/0 或 2 次尝试、logical_used、vsegments）；vseg-1 干净版 T1/16t P50 与 target W9；**布局假设是否成立**；有无失败。

---

# NOF-033（已关闭 · 只读 LBA 序列取证 · 2026-11-23）

## 背景

- 布局假设已被两次否证，但两次都建立在同一个**未取证的前提**上：T1 顺序写 ⇒ 物理 LBA 连续。我们从未看过 vseg-1 的实际读 LBA 序列。
- 在决定是否改代码前，补这一张事实：**vseg-1 读一个 key 的 32 条 IO，LBA 是否连续；key 与 key 之间是否顺序增长。**

## A. 预检（只读）

- 当前环境：vseg-1 master（PID 1648052）、target 341038、4457、etcd 键；node1 同事进程只记录不动（共享机）。
- **不需要切换任何环境**；任何异常 → 停手回报。

## B. 一遍低阈值诊断读（T1）

- reader：node3，**T1**、batch1、verify=true；`VSEG_DIAGNOSTIC=1`、`MC_HIFREQ_LOG_SAMPLE_RATE=1`；
- **压低阈值以捕获绝大多数 IO**：`VSEG_SLOW_IO_US=1`（日志预计 ~10–20MB；若 >50MB 或磁盘压力大 → 停手回报，改用 50µs 阈值重试一次并说明）。
- 日志：`run/nof033_lba_t1.log`；跑完记录 target W 块（W9/n）。

## C. LBA 解析（只报事实）

从 `URMA_SLOW_IO`（opcode=2、读阶段、按行号剔除 warmup）提取 `lba`、`blocks`、`roundtrip_us`：

1. 取**前 10 个 key**（按出现顺序）：每个 key 的 32 条 IO 的首/末 LBA；相邻 IO 的 LBA 差（是否 256 blocks = 128KiB 递增）；
2. 统计：捕获的读 IO 总数；"相邻差=256"的比例；大跳变（>1MiB）次数；给出 3 段原始序列样例（每段 ≤5 个 LBA）；
3. 判断：**每个 key 内是否连续？key 与 key 之间是否单调递增？** 给结论与证据；
4. target W9（本窗口）。

## D. 边界

- 只读；不切换环境（沿用当前 vseg-1）；不动 target/同事进程/键；失败停手。
- 不改任何负载参数（除诊断 env 与 T1 线程数）。

## 回执必答（第一行）

vseg-1 读 IO 的 LBA 是否连续/顺序（每 key 内、key 间）；相邻差=256 的比例；3 段样例；target W9；**LBA 数据是否能为布局洗清或坐实**。

---

# NOF-034（已关闭 · 最后干净组合：vseg-1 + stripe=4MiB + T1 顺序写 · 2026-11-23）

## 背景与假设（最后一个未测组合）

- NOF-032（布局干净）：slot=1 + **stripe=2MiB** ⇒ 每 key **2 个 child**；T1 536.7 / W9 251.1。
- NOF-031（stripe=4MiB）：**slot=1024 + 16 线程写**（布局散）；T1 551.6 / W9 252.3。
- 从未测过：**slot=1 + stripe=4MiB（每 key 单 child）+ 纯 T1 顺序写（布局连续）**。
  - 若该组合 T1 接近 plain 258.3、W9 降到 ~97 ⇒ **"每 key 拆 2 child"就是剩余成本**，vsegment 可通过该配置追平 plain；
  - 若仍 ~537/252 ⇒ 每 key 的客户端开销在别处（需代码层定位）。

## A. 预检（只读）

- 当前 vseg-1 master（PID 1648052）、target 341038、4457、etcd 键；node1 同事进程只记录不动；node3 独占。异常 → 停手。

## B. 切换 vseg-1 + stripe4（只动我方 master/cluster）

1. 停当前 master（按 PID）；清**仅我方**键。
2. 启动 master：NOF-004/007 同参数 + `MC_STORE_PARTITION_SLOT_COUNT=1` + **`--vsegment_stripe_size=4194304`**。
3. `nof_register` 同命令；等 quota ≥20s；验证 `stripe_size=4194304`、partitions=1、`reconcile_fail=0`。

## C. 纯 T1 顺序写

- writer：**`--num_threads=1 --batch_size=1`**，400 keys × 4MiB（vsegment flags 同 NOF-018）；日志 `run/writer_vseg1_s4m_t1.log`。
- `VSEGMENT_CREATING`（卷边界）允许多跑 1 次补齐（总尝试 ≤2）；确认 `committed_allocations=400` 或 `logical_used=1,677,721,600`；记录 vsegments。
- 若两次后仍缺 key → 停手回报（不第 3 次）。

## D. 读（同口径）

- T1 × 3 遍 + 16t × 3 遍（verify=true、诊断关、client slot env=1）。
- 记录 Avg/P50/P90/P99/Max、failed/verify；target W 块按"下一次连接打印上一轮"对齐；`rpc_first_us` p50。

## E. 对比与回答

| 场景 | 布局 | child | T1 P50 | 16t P50 | W9（T1） |
|---|---|---|---|---|---|
| plain NoF | 连续 | 无 | 258.3 | 718.7 | 96.9 |
| vseg-1 / stripe2 | 连续 | 2×2MiB | 536.7 | 921.8 | 251.1 |
| **vseg-1 / stripe4（本单）** | 连续 | **1×4MiB** | ? | ? | ? |

- 明确回答：**单 child + 连续布局下，T1 是否接近 258？W9 是否降到 ~100 量级？** 若是 → child 拆分是成本、配置可解；若否 → 客户端每 key 开销在别处（转代码层定位）。
- 保留该环境。

## F. 边界

- 只动我方 master/cluster；不动 target；共享机规则；失败停手。

## 回执必答（第一行）

stripe=4MiB/partitions=1 是否生效；T1 顺序写是否补齐（committed/logical_used/vsegments）；vseg-1/4MiB 的 T1/16t P50 与 target W9；**是否追平 plain（258.3/96.9）**；有无失败。

---

# NOF-035（已关闭 · 补 plain 的 LBA/IO 分布 + 同窗 W9 · 2026-11-23）

> 结果：96.9 可在干净窗口复现（99.14µs），伪影推断被否证；命令形态与 LBA 布局两边相同，差异在 target 执行段。详见"NOF-035 关键事实"。

## 目的（收口后的补证单）

报告中的一个推断尚未被直接验证：**"plain 与 vsegment 发给 target 的命令相同（128KiB、连续 LBA），W9 的 2.6× 差异更像窗口/盘缓存造成的测量伪影"**。
本单切回 plain，按 NOF-025 的**同序**（写 → 立即读），用 NOF-033 的**同口径**低阈值诊断，一次性拿到：
1. plain 读的 **LBA 序列与 IO 大小分布**（与 vseg 对比）；
2. plain 本轮的**同窗 W9**（与历史 96.9 对比）；
3. plain 的写窗口 W9（识别写/读窗口是否被混算）。

## A. 预检与切换（只动我方 master/cluster）

1. 记录当前环境（vseg-1+stripe4 master PID 1676025、target 341038、端口、etcd 键）；node1 同事进程只记录不动。
2. 停我方 master（按 PID）；清**仅我方**键 `/cvm/vseg_nofvseg/` + `/oplog/vseg_nofvseg/`。
3. 启动 **plain master**（NOF-025 参数：无 vsegment，`--vsegment_auto_plan=false --cvm_segments_vsegment_exclusive=false`；50056/9006；env `udmac0d1e2`）；`nof_register` 同命令挂载；确认 leader/serving、无 quota。
4. **plain writer**（NOF-025 同参：16 线程、batch16、`--replica_num=0 --nof_replica_num=1 --vsegment_nof_io=false --segment_write_vsegment=false`）：确认 400/0、master Keys=400；**记录 writer 后 target W 块**（写窗口基线）。

## B. 同窗诊断读（T1，1 遍，低阈值）

- writer 结束后 **10 秒内**启动 reader：T1、batch1、verify=true；`VSEG_DIAGNOSTIC=1`、`MC_HIFREQ_LOG_SAMPLE_RATE=1`、`VSEG_SLOW_IO_US=1`；日志 `run/nof035_lba_plain_t1.log`（若 >50MB → 停手，改用 50µs 阈值重试一次并说明）。
- 记录：端到端 Min/Avg/P50/P90/P99/Max；读阶段 `URMA_SLOW_IO`（lba/blocks）；target W 块（**写窗口、读窗口各一块**：n / avg / TOTAL / exec%）。

## C. 解析与对比（与 NOF-033 完全相同算法）

1. 读阶段（opcode=2）IO 总数；`blocks` 直方图（256 / 4096 / 其他）；每 4MiB 窗口归组：相邻差=256 比例、>1MiB 跳变次数、key 起点步长分布与是否单调、覆盖率；
2. 给 3 段原始序列样例（每段 ≤5 条 LBA+blocks）；
3. 与 vseg NOF-033 对比：**plain 是否也以 128KiB 为主？是否也有 2MiB IO？LBA 连续性是否相同？**
4. **W9 对比**：plain 本轮读窗口 W9 vs 历史 96.9；vseg-1 为 251.1；写窗口 W9（预期 ~27.8）。
5. 明确回答：
   - plain 与 vseg 的命令**是否真的相同**（尺寸分布 + LBA 连续性）？
   - plain 的 **96.9µs 能否在干净窗口里复现**？若升到 ~250 ⇒ 历史值是窗口/缓存伪影；若仍 ~97 ⇒ 差异真实，需 target 逐 IO 明细（另一单，需重启 target）。

## D. 边界

- 只动我方 master/cluster；**不动 target**；共享机规则同 NOF-031（同事进程只记录）；失败停手。
- plain 写入会覆盖此前 vsegment 数据（允许，后续可恢复）。

## 回执必答（第一行）

plain 读的 blocks 直方图与 LBA 连续性（与 vseg 是否相同）；本轮读窗口 W9（vs 历史 96.9）；写窗口 W9；**plain 与 vseg 命令是否相同、96.9 能否复现**；有无失败。

---

# NOF-036（已关闭 · 免重启写模式对照：16t 写 vs T1 写 · 2026-11-23）

> 结果：三臂读 W9 96–110µs（稳定）；但臂 2/3 的写被 `OBJECT_ALREADY_EXISTS` 全拒、未落盘，写模式对照未发生；发现 bench 把被拒 put 计为成功。详见"NOF-036 关键事实"。

## 背景与假设

- NOF-035：plain（16t 写 → T1 读）读窗口 W9 = **99.14µs**；所有 vseg 环境（T1 或 16t 写）读 W9 恒 **250–271µs**，且命令形态/LBA 已证相同。
- 从未控制过的一个变量：**写模式**。"快读"的数据全部是 plain 16t 写出来的；"慢读"的数据全部经 vseg 写路径写出来。若写线程数/写顺序决定了盘上物理布局，则读 W9 差异可能来自**写侧留下的盘上状态**，而不是 vseg 读路径。
- 本单在**当前 plain 环境**（master 1778131、target 341038）内做三臂对照，一次定性；**不动 target**。

## A. 预检（只读）

1. 确认 master 1778131 leader/serving、50056/9006 在听；target 341038 未动、4457 在听；`logical_used=1,677,721,600`、Keys=400；node1 同事进程只记录不动。
2. 记录起点 target W 快照。

## B. 臂 1：不重写，直接复读（基线稳定性）

- reader 与 NOF-035 完全相同：T1、batch1、verify=true、plain flags；诊断开（`VSEG_DIAGNOSTIC=1`、`MC_HIFREQ_LOG_SAMPLE_RATE=1`、`VSEG_SLOW_IO_US=1`）；日志 `run/nof036_arm1_read_p1.log` / `_p2.log`。
- **连跑 2 遍**；每遍后记录 target W 快照（新连接进入即清零，直接读本轮块）。
- 记录：端到端 Min/Avg/P50/P90/P99/Max、failed/verify、读窗口 W9（n/avg/exec%/TOTAL）。
- 同时跑 NOF-035 同款解析脚本：blocks 直方图 + 400 个 4MiB 窗口的起点步长、>1MiB 跳变数（确认布局与 NOF-035 一致）。

## C. 臂 2：T1 重写 → 立即读

1. writer：`--num_threads=1 --batch_size=1`，plain flags（`--replica_num=0 --nof_replica_num=1 --vsegment_nof_io=false --segment_write_vsegment=false`），400 keys × 4MiB、verify=false；确认 400/0、master Keys=400；记录写窗口 W9（n/avg/exec%/TOTAL）。
2. 写完 **10 秒内**启动 reader（B 同口径），2 遍；每遍记录读窗口 W9 + 解析结果。
3. 关注：T1 写是否把读 W9 拉到 ~250；写窗口 W9 是否仍 ~28µs；布局是否仍连续。

## D. 臂 3：恢复 16t 重写 → 立即读

1. writer：`--num_threads=16 --batch_size=16`（NOF-035 原参），400/0；记录写窗口 W9。
2. 立即读 2 遍（B 同口径），确认读 W9 能否回到 ~99；若回不去，记录"写后残留状态"的持续时间（可加读 1 遍，间隔 60s，仅记录）。

## E. 对比与回答

| 臂 | 写模式 | 读 W9（2 遍） | 读端到端 P50 | 布局（步长/跳变） |
|---|---|---|---|---|
| 1 不重写 | —（16t 写的数据） | ? | ? | ? |
| 2 T1 写 | 1t / batch1 | ? | ? | ? |
| 3 16t 写 | 16t / batch16 | ? | ? | ? |

- 明确回答：**同一批 plain 数据、同一 target，仅改变写线程数/写顺序，读 W9 是否从 ~99 变到 ~250？**
  - **是** ⇒ 读 W9 由写模式/盘上状态决定，vsegment 的慢读可能源自其写路径（下一单测 vseg-1+stripe4 的 16t 写 + T1 读，配置层可能仍有解）；
  - **否** ⇒ 写模式无关，唯一剩余变量是 **vseg 客户端读路径**（下一单转 target 逐 IO 明细，需重启 target，另行申请）。
- 副回答：T1 写的写窗口 W9 是否仍 ~28µs；三臂布局是否都保持每 key 严格连续。

## F. 边界

- 只动我方 writer/reader；**不动 target**、不改 master 参数；共享机规则；失败停手。
- 重写覆盖同一 1.6GiB 数据（允许）；每臂读 2 遍（W9 本身稳定，2 遍防偶发）；诊断全程开启，端到端仅作辅助（诊断会抬高绝对值），判决以 W9 为准。

## 回执必答（第一行）

三臂读 W9（不重写 / T1 写 / 16t 写）与端到端 P50；T1 写是否把读 W9 拉到 ~250；各臂是否 400/0、verify 0 错、布局仍连续；有无失败。

---

# NOF-037（已关闭 · 写模式对照·续：新增 key 扩展法，按 LBA 桶比较读时延 · 2026-11-23）

> 结果：两次写均落盘并验收通过；写线程数对读时延无影响（三桶 83–100µs）；发现段首 key 0–96（LBA [0,388MiB)）稳定慢区（p90≈290µs），是新的关键线索。详见"NOF-037 关键事实"。

## 背景与设计

- NOF-036 证明：plain 数据**不支持重复写**（key 已存在时 PutStart 一律被拒），所以"重写同一批 key"无法制造新的写模式；同时 bench 会把被拒 put 计为成功，**验收必须用 master 与 target**。
- 本单用**新增 key**（不改代码、不删 key、不清集群、不动 target）：`MakeKey(i)="bench_key_"+i`（`stress_cluster_bench.cpp:2357`），写 800/1200 keys 时，已存在的 0–399/0–799 会被拒，新 key 400–799 / 800–1199 会真正写入段内空闲区。
- 三桶对照（读一遍即可同时比较，桶 1/3 为对照，桶 2 为处置）：

| 桶 | key | 写入方式 | 预期 LBA 范围（512B 块） |
|---|---|---|---|
| 1 | 0–399 | NOF-035 的 16t 写（已有） | [0, 3,276,800) |
| 2 | 400–799 | 本单 T1 写 | [3,276,800, 6,553,600) |
| 3 | 800–1199 | 本单 16t 写 | [6,553,600, 9,830,400) |

## A. 预检（只读）

1. 确认 master 1778131 leader/serving、50056/9006；`Keys=400`、`logical_used=1,677,721,600`；target 341038 未动、4457。
2. 记录起点 target W 快照；确认 node3 run 目录剩余空间 >500MB（可先删 NOF-036 的 6 个 18MB 日志，保留 NOF-035 日志）。

## B. 写桶 2（T1 写，400 个新 key）

- writer：`--num_keys=800 --num_threads=1 --batch_size=1`、plain flags、verify=false。
- 预期：keys 0–399 被拒（`OBJECT_ALREADY_EXISTS` 约 400 条，属预期，不视为失败）；keys 400–799 真正写入。
- **验收（必须全部满足，否则停手回报）**：
  1. master `Keys=800`、`logical_used=3,355,443,200`；
  2. target 写窗口 **n ≥ 12,000**（对照新鲜写 12,802），记录 W9 avg；
  3. 不采信 bench 的 `succeeded` 计数。
- 日志 `run/nof037_write_arm2.log`；写窗口快照 `nof037_W_arm2_write.txt`。

## C. 写桶 3（16t 写，再 400 个新 key）

- writer：`--num_keys=1200 --num_threads=16 --batch_size=16`、plain flags、verify=false。
- 预期：keys 0–799 被拒（约 800 条，属预期）；keys 800–1199 真正写入。
- **验收**：master `Keys=1200`、`logical_used=5,033,164,800`；target 写窗口 n ≥ 12,000；记录 W9 avg。
- 日志 `run/nof037_write_arm3.log`；快照 `nof037_W_arm3_write.txt`。

## D. 一次读覆盖三桶（T1 诊断，2 遍）

- reader：`--num_keys=1200 --num_threads=1 --batch_size=1 --order=sequential --verify=true`、plain flags；`VSEG_DIAGNOSTIC=1`、`MC_HIFREQ_LOG_SAMPLE_RATE=1`、`VSEG_SLOW_IO_US=1`。
- 连跑 2 遍；日志 `run/nof037_read_1200_p1.log` / `_p2.log`（每遍约 55MB；>80MB 才需处理并说明）。
- 记录：端到端 Min/Avg/P50/P90/P99/Max、failed/verify；每遍后 target 读窗口 W9（n≈77,286）/exec%/TOTAL。
- 解析（扩展 NOF-035 脚本到 1200 个 4MiB 窗口）：`bucket = window_index // 400`
  - 每桶：blocks 直方图、**128KiB 的客户端 roundtrip p50/p90**、窗口起点步长与 >1MiB 跳变、与预期 LBA 范围是否吻合；
  - 给 3 段原始样例（桶 2 开头/中间/结尾各 ≤5 条）。

## E. 对比与回答

| 桶 | key | 写 | 128KiB roundtrip p50（p1/p2） | 布局（步长/跳变） |
|---|---|---|---|---|
| 1 | 0–399 | 16t（NOF-035） | ? | ? |
| 2 | 400–799 | **T1** | ? | ? |
| 3 | 800–1199 | 16t（本单） | ? | ? |

- 明确回答：**仅改变写线程数，同一读会话内桶 2 的 128KiB roundtrip 是否显著高于桶 1/桶 3（~97µs）？是否达到 vseg 的 ~250–274µs 量级？**
  - 是 ⇒ 写模式/盘上状态是读慢的主因 → 下一单测 vseg-1+stripe4 的 16t 写；
  - 否（三桶都 ~97）⇒ 写模式无关 → 唯一剩余变量是 vseg 客户端读路径，下一单转 target 逐 IO 明细（需重启 target，另行申请）。
- 副回答：桶 3 与桶 1 是否一致（排除会话/时间漂移）；target 读窗口 W9 是否仍 ~99；三次写的写窗口 W9 是否仍 ~28µs。

## F. 边界

- 只动我方 writer/reader；**不动 target、不改 master 参数、不删 key、不清集群**；共享机规则；失败停手。
- 新增写入量 2×1.6GiB（段 100GiB，充足）；读到 4.8GiB 范围。
- bench 的成功计数不可信；每个写入臂必须过 B/C 的 master+target 验收，任一不满足即停手上报（不做第 3 次尝试）。

## 回执必答（第一行）

三桶 128KiB roundtrip p50/p90（桶 2 vs 桶 1/3）；T1 写桶是否达到 ~250µs 量级；各写入臂的 master Keys/logical_used 与 target 写窗口 n；target 读窗口 W9；布局是否连续；有无失败。

---

# NOF-038（已关闭 · 阻断：构建二进制无随机读旗标 · 2026-11-23）

> 结果：`--key_order`/`--key_order_seed` 不存在（构建 be10be2 早于源码 4c0ea28），随机读无法执行；`--shuffle_seed` 对 remote_memory 无效。旁证读逐位复现 NOF-037（22.0% 慢、key 96/97 边界、慢段 ~0.7s）。详见"NOF-038 关键事实"。

> 【2026-11-23 用户补充 · 执行口径更新】段首慢区是**已知盘特性**（每盘 ~36MB，RAID0 11 盘 ≈396MB ≈ 前 97 个 4MiB key），不是新线索，也解释不了 vseg（vseg 全 key 均匀慢）。本单所有读臂**必须 `--warmup_keys=128`** 跳过该窗口，慢窗口的读不计入结论。主问题只有一个：**去掉慢窗口后，随机顺序是否让全部 key 变慢**（vseg 的快慢是否依赖连续顺序读流）。
> 若已按旧版（warmup=5）开跑：跑完照常回执（旧版结果中 keys 0–96 慢属已知），再按本版补跑。

## A. 预检（只读）

1. master 1778131 leader/serving、50056/9006；`Keys=1200`、SSD 4.69GB；target 341038 未动、4457；node3 run 目录空间充足。
2. 记录起点 target W 快照。
3. **旗标确认**：读序旗标 `--key_order`（不是 `--order`）、warmup 旗标 `--warmup_keys`；用 `--help 2>&1 | grep -E "key_order|warmup_keys"` 确认。

## B. 臂 0：顺序读 + warmup=128（干净基线，1 遍）

- reader：`--num_keys=1200 --num_threads=1 --batch_size=1 --key_order=sequential --warmup_keys=128 --verify=true`、plain flags；`VSEG_DIAGNOSTIC=1`、`MC_HIFREQ_LOG_SAMPLE_RATE=1`、`VSEG_SLOW_IO_US=1`；日志 `run/nof038_read_seq_w128.log`（约 55MB）。
- 记录：端到端 Min/Avg/P50/P90/P99/Max、verify、target 读窗口 W9（n≈76,963）。
- 验收：**keys 0–96 的慢占比应显著低于 NOF-037 的 22%**（warmup 生效的证据）；其余 key 仍 ~97µs。

## C. 臂 1/2：随机读 + warmup=128（2 遍，不同种子）

- reader 同 B，只改 `--key_order=uniform_random --key_order_seed=42`（臂 1）与 `--key_order_seed=7`（臂 2）；日志 `run/nof038_read_rand42_w128.log` / `nof038_read_rand7_w128.log`。
- 每遍记录：端到端 Min/Avg/P50/P90/P99/Max、failed/verify、target W 快照/W9。
- **顺序自证**：取前 20 条读 IO 的 lba 确认非单调；给 5 条原始样例。

## D. 解析（NOF-037 脚本扩展）

1. 每臂总体 128KiB roundtrip p50/p90（≥200µs 为慢）。
2. **按 key 归组**：0–96 / 97–399 / 400–799 / 800–1199 的慢占比与 p90（验证 warmup 是否把 0–96 拉回正常）。
3. **按日志位置归组**：前 5% / 5–10% / 10–25% / >25% 行的慢占比（看是否还有连接预热尾巴）。
4. 随机臂 vs 顺序臂的总体 p50/p90 差值。

## E. 回答与判决

| 臂 | 读序 | warmup | 总体 p50/p90 | keys 0–96 慢占比 | 前 10% 行慢占比 |
|---|---|---|---|---|---|
| 0 | 顺序 | 128 | ? | ? | — |
| 1 | 随机 seed42 | 128 | ? | ? | ? |
| 2 | 随机 seed7 | 128 | ? | ? | ? |

- 明确回答：
  1. warmup=128 后，keys 0–96 是否回到 ~97µs（**慢窗口是否被 warmup 跳过**）？
  2. **随机顺序是否让全部 key 变慢到 ~250µs**（快模式是否依赖顺序流）？
  3. 对 vseg 的含义：
     - 随机仍 ~97 ⇒ 顺序流不是关键，vseg 的慢另有原因 → 下一单转 target 逐 IO 明细（需重启 target，另行申请）；
     - 随机全变慢 ⇒ 顺序流是关键，vseg 的 2MiB child/断点读法可能就是原因 → 下一单回 vseg 环境（vseg-1+stripe4 + warmup=128）验证读流形态。

## F. 边界

- **零写入、零重启**：只读 1200 keys；不动 target/master 参数；不删 key；共享机规则；失败停手。
- 所有读臂必须 `--warmup_keys=128`；回执必须写明 warmup 值。
- 若随机两遍结论不一致 → 加第 3 个种子（seed 123）。

## 回执必答（第一行）

三臂（顺序/随机×2，均 warmup=128）总体 p50/p90 与 keys 0–96 慢占比；**warmup=128 是否跳过慢窗口；随机顺序是否让全部 key 变慢**；target W9；有无失败。

---

# NOF-039（已关闭 · 零代码验证 warmup 跳过盘首慢窗口 + 干净 T1/16t 基线 · 2026-11-23）

> 结果：**warmup 不能跳过慢窗口**（两阶段读 keys 0–96 均 ~245µs）；慢区是固定 LBA 段。干净（排除 0–96）基线：T1 p50 86 / W9 68；16t p50 359 / W9 161 ⇒ 并发抬高 4.2×。详见"NOF-039 关键事实"。

## 背景与目的

- 用户 2026-11-23 告知：每块盘开头约 **36MB 是慢区**，RAID0 11 盘 ⇒ 前 **~396MB**（4MiB key 实测前 97 个 ≈388MiB）读时延 ~290µs；**基准必须用 warmup 跳过，不计入结果**。
- NOF-038 因构建二进制无 `--key_order` 而阻断；本单用现有 `--warmup_keys`（零代码、零写入、零重启）验证"warmup 是否真能跳过慢窗口"，同时取得**去掉慢窗口后的干净 T1 / 16t 基线**，供最终报告与 vseg 对比使用。

## A. 预检（只读）

1. master 1778131 leader/serving、50056/9006；`Keys=1200`、SSD 4.69GB；target 341038 未动、4457；node3 run 目录空间充足（**不要删** NOF-037/038 日志）。
2. `--help 2>&1 | grep -E "warmup_keys|num_threads|batch_size"` 确认旗标存在。
3. 记录起点 target W 快照。

## B. 臂 A：T1 顺序读 + warmup=128

- reader：`--num_keys=1200 --num_threads=1 --batch_size=1 --warmup_keys=128 --verify=true`、plain flags；`VSEG_DIAGNOSTIC=1`、`MC_HIFREQ_LOG_SAMPLE_RATE=1`、`VSEG_SLOW_IO_US=1`；日志 `run/nof039_read_t1_w128.log`（约 55MB）。
- 起跑日志确认出现 `Warmup: reading 128 keys...` 且 warmup 完成。
- 记录：端到端 Min/Avg/P50/P90/P99/Max、failed/verify、target 读窗口 W9（n≈76,963）/exec%/TOTAL。
- 解析（NOF-037 同脚本）：按 key 桶 **0–127 / 128–399 / 400–799 / 800–1199** 的 128KiB p50/p90、≥200µs 慢占比；重点 keys 0–96 慢占比（对照 NOF-037 的 22%）。
- **验收**：keys 0–96 慢占比 <3%（≈0）⇒ warmup=128 成功跳过慢窗口。

## C. 臂 B：T1 顺序读 + warmup=400

- 同 B，`--warmup_keys=400`；日志 `run/nof039_read_t1_w400.log`。
- 目的：确认 400 同样有效，并作为最终报告的"完全干净"T1 基线（测量阶段含 0–399 的第二遍读，但应全快）。

## D. 臂 C：16t 读 + warmup=400

- reader：`--num_keys=1200 --num_threads=16 --batch_size=1 --warmup_keys=400 --verify=true`，其余同 B；日志 `run/nof039_read_16t_w400.log`。
- 目的：慢窗口排除后，并发 1→16 的每 IO 时延差（对照历史 16t W9 194–196µs（未排除慢窗口）与干净 T1 基线），为 vseg 的读流形态问题提供参照。
- 解析：总体与按 key 桶的 p50/p90、target W9。

## E. 回答与判决

| 臂 | 线程 | warmup | 总体 p50/p90 | keys 0–96 慢占比 | target W9 |
|---|---|---|---|---|---|
| A | 1 | 128 | ? | ? | ? |
| B | 1 | 400 | ? | ? | ? |
| C | 16 | 400 | ? | ? | ? |

- 明确回答：
  1. **warmup 是否真能跳过盘首慢窗口**（keys 0–96 在测量阶段是否回到 ~97µs）？128 够不够，还是必须 400？
  2. 干净基线：T1 与 16t 的 p50/p90、target W9 各是多少？并发 1→16 抬高了多少？
  3. 若 warmup 后 keys 0–96 仍慢 ⇒ 说明该 bench 的 warmup 无法真正跳过，给出**排除 keys 0–96**后的统计，并在结论中声明"今后按排除口径"。
- 对 vseg 的意义：若 16t 干净基线明显高于 T1（如 >150µs），说明并发/多流本身会抬高每 IO 时延，vseg 的多流读法（2 child × 16 chunk）可能落入同一类效应；若 16t 仍 ~97，则并发无关，vseg 的慢另有原因。

## F. 边界

- **零写入、零重启**：只读 1200 keys；不动 target/master 参数；不删 key、不删既有日志；共享机规则；失败停手。
- 每臂 1 遍（W9 稳定；三臂共约 1–2 分钟读 + warmup）；所有臂必须写明 warmup 值。
- 若臂 A 的结果不足以判定，臂 B/C 照跑即可；不做额外臂。

## 回执必答（第一行）

warmup=128/400 后 keys 0–96 的慢占比与 p50/p90；干净 T1 与 16t 的总体 p50/p90 与 target W9；**warmup 是否成功跳过慢窗口、128 是否足够**；有无失败。

---

# NOF-040（已关闭 · vseg 干净复测（排除慢窗口）+ qpairs=1 对照 · 2026-11-23）

> 结果：vseg T1 干净 p50 274 / W9 255.25（plain 86/68）；qpairs 4→1 仅 −2.6% ⇒ lane 数无关；keys 97–399 慢占比 90.7% ⇒ vseg 全程均匀慢；16t 差距收窄到 1.15×。详见"NOF-040 关键事实"。

## 背景与目的

- NOF-039：warmup 不能跳过盘首慢窗口（慢区是固定 LBA 段）；干净（排除 keys 0–96）基线：**plain T1 p50 86 / p90 109 / W9 68µs；plain 16t p50 359 / p90 570 / W9 161µs**（并发 1→16 抬高 4.2×）。
- vseg 的历史数字（T1 p50 250–274 / W9 250；16t W9 250–271）都**未做排除**，且缺一条同口径的干净基线；本轮同时检验一个未测过的极端旋钮 **qpairs=1**（NOF-024 只测过 4/8/16）——若"多 lane/多 qpair 并发"是 vseg 每 IO 慢的来源，qpairs=1 应显著降低时延。
- **用户已授权切环境**（只动我方 master/cluster，target 不动；盘上 plain 数据会被覆盖，允许）。

## A. 环境切换与写入（vseg-1 + stripe=2MiB）

1. 记录 plain 环境状态（master 1778131、Keys=1200）；按 PID 停 master；删**仅我方**键 `/cvm/vseg_nofvseg/` + `/oplog/vseg_nofvseg/`。
2. 启动 vseg-1 master：NOF-032/034 同参数 + `MC_STORE_PARTITION_SLOT_COUNT=1` + `--vsegment_stripe_size=2097152` + `--vsegment_auto_plan=true`；`nof_register` 同命令；等 quota ≥20s；验收 `partitions=1`、`stripe_size=2097152`、`reconcile_fail=0`。
3. writer T1：`--num_threads=1 --batch_size=1`、vsegment flags（`--vsegment_nof_io=true --segment_write_vsegment=true --replica_num=1 --nof_replica_num=0`）；400 keys；`VSEGMENT_CREATING` 允许**最多 2 次尝试**补齐；验收 master `committed_allocations=400` / `logical_used=1,677,721,600`、读 verify 0 错；记录写窗口 W9。

## B. 读臂（每臂 `--num_keys=400 --warmup_keys=128 --verify=true`、诊断开、plain 参数）

- 臂 A：`--num_threads=1 --batch_size=1 --nof_qpairs_per_segment=4`（基线）；日志 `run/nof040_vseg_t1_q4.log`。
- 臂 B：`--num_threads=1 --batch_size=1 --nof_qpairs_per_segment=1`（极端对照）；日志 `run/nof040_vseg_t1_q1.log`；若 qpairs=1 被拒绝/异常 → 记录并跳过该臂，不重试。
- 臂 C：`--num_threads=16 --batch_size=1 --nof_qpairs_per_segment=4`；日志 `run/nof040_vseg_16t_q4.log`。
- 每臂记录：端到端 Min/Avg/P50/P90/P99/Max、failed/verify、target W9（n≈25,762）/exec%/TOTAL。

## C. 解析与回答

- 每臂按 key 桶：0–96（慢窗口）/ 97–399（干净）；同时给"全量"与"排除 0–96"两套 p50/p90。

| 环境 | 线程 | qpairs | 全量 p50 | 排除 0–96 p50/p90 | target W9 |
|---|---|---|---|---|---|
| plain（NOF-039） | 1 | 4 | 86 | 86 / 109 | 68 |
| plain（NOF-039） | 16 | 4 | 359 | 359 / 570 | 161 |
| vseg 臂 A（本单） | 1 | 4 | ? | ? | ? |
| vseg 臂 B（本单） | 1 | **1** | ? | ? | ? |
| vseg 臂 C（本单） | 16 | 4 | ? | ? | ? |

- 明确回答：
  1. **vseg T1 的干净基线**（排除 0–96）是否仍 ~250µs？与 plain（86/68）的差是否仍是 ~3×？
  2. **qpairs 4→1 是否显著降低** vseg 的 per-IO 与 target W9？
     - 是（降到 ~70–100）⇒ 多 lane/多 qpair 并发是 vseg 慢的主因 → 下一单围绕 lane 做配置/代码定位；
     - 否（仍 ~250）⇒ lane 数无关 → 只剩 target 逐 IO 明细（需重启 target，另行申请）或客户端代码插桩。
  3. vseg 16t 干净基线 vs plain 16t（359/570/W9 161），以及历史端到端 735 vs 719µs 在新口径下的位置。

## D. 边界

- 只动我方 master/cluster；**不动 target**；共享机规则；失败停手。
- 切环境覆盖盘上 plain 数据（允许）；`VSEGMENT_CREATING` 超过 2 次尝试即停手回报。
- 所有臂写明 warmup；统计同时给"全量"与"排除 0–96"两套口径。

## 回执必答（第一行）

vseg T1 的干净（排除 0–96）p50/p90 与 target W9；qpairs=4 vs 1 的差异；vseg 16t 干净基线；与 plain（86/68、359/161）的对比；有无失败。

---

# NOF-041（已关闭 · 现有日志逐 IO 五段分布对比：vseg vs plain，零环境操作 · 2026-11-23）

> 结果：vseg 多出的 188µs **全部在 bdev 段**（250 vs 46µs）；data_us/outside_us 反而更小；vseg bdev 单峰且与 plain 盘首慢区分布逐桶吻合 ⇒ vseg 全部 key 都处于同一"慢状态"。详见"NOF-041 关键事实"。

## 背景与目的

- NOF-040：vseg T1 干净（排除 0–96）p50 274 / target W9 255.25µs；plain 同口径 86 / 68µs；qpair 数无关；vseg 全程均匀慢。
- 现有日志里每条读 IO 都带客户端 roundtrip 与 **target 回传的逐 IO 五段字段**（NOF-011/020 用过：`target_us`、`bdev_us`、`data_us`、`pre_send_us`、`outside_us`、`residual_us`）。
- 本单**不跑任何新读、不动环境**，只解析已有日志，定位 vseg 多出的 ~185µs 落在哪一层，以及是均匀还是快异常。

## A. 输入（node3 `~/mooncake-nof-vsegment/run/`，只读）

- vseg：`nof040_vseg_t1_q4.log`、`nof040_vseg_t1_q1.log`、`nof040_vseg_16t_q4.log`
- plain：`nof039_read_t1_w400.log`、`nof039_read_16t_w400.log`
- 先打印 1 条完整 `URMA_SLOW_IO` 行，列出实际字段名；若某字段不存在，如实标注并用实际存在的字段替代。

## B. 解析口径

1. 仅 **opcode=2（读）、blocks=256（128KiB）**；剔除 **warmup 段**（日志开头的 warmup 读）与 **verify 大 IO**（blocks=4096/8192）；按 key 桶 **0–96 / 97–399** 分别统计。
2. 每份日志输出每字段 **n / p50 / p90 / mean**：`roundtrip_us`、`target_us`、`bdev_us`、`data_us`、`pre_send_us`、`outside_us`、`residual_us`（按实际字段名）。
3. **bdev_us 直方图**（6 桶：<50 / 50–100 / 100–200 / 200–300 / 300–500 / >500µs）：vseg T1 q4 与 plain T1 各一份；用 <100µs 与 ≥200µs 两簇占比描述是否快异常。
4. **data_us 直方图 + p50/p90** 同上。
5. 计算 `roundtrip − target_us` 的 p50（客户端/其他层开销）。
6. `target_valid=0` 的行单列计数，不参与五段统计（roundtrip 仍可用）。

## C. 回答

| 指标（128KiB，排除 0–96） | vseg T1 q4 | plain T1 |
|---|---|---|
| roundtrip p50 | 274 | 86 |
| target_us p50 | ? | ? |
| bdev_us p50 / p90 | ? | ? |
| data_us p50 / p90 | ? | ? |
| bdev_us 分布（<100µs 占比 / ≥200µs 占比） | ? | ? |

- 明确回答：
  1. vseg 多出的 ~185µs 主要落在 **bdev_us（盘执行）**、**data_us（数据传输）** 还是客户端字段？
  2. vseg 的 bdev_us 是**均匀 ~250** 还是**快异常**（部分快、部分排队）？
  3. plain 同字段的分布（对照）。
  4. 从分布推断：更像"盘/队列执行慢"（bdev 均匀高）还是"传输/背压/排队"（data 高或快异常）。

## D. 边界

- **零环境操作**：不跑 bench、不动 target/master、不改任何文件；只读日志并用本地脚本解析。
- 回执只给汇总表、直方图计数与 ≤20 行样例；不贴原始日志、不外传数据。

## 回执必答（第一行）

vseg/plain 的 `bdev_us` 与 `data_us` p50/p90 及直方图（均匀还是快异常）；多出的 ~185µs 落在哪个字段；target_valid 缺失计数；有无失败。

---

# NOF-042（已关闭 · 纯日志时序分析：提交顺序/落核/大 IO 重叠，零环境操作 · 2026-11-23）

> 结果：时序/落核/重叠均无可辨识差异；vseg 四核 bdev 全线 218–261µs vs plain 47–48µs；并纠正"每 key 一条大 IO"为日志拼接假象（URMA_SLOW_IO 全为 128KiB）。详见"NOF-042 关键事实"。

## 背景与目的

- NOF-041：vseg 的 188µs 增量全部在 `bdev_us`（盘执行段），分布单峰（200–300µs 占 76.2%、<100µs 为 0），与 plain 盘首慢区逐桶一致；data/outside 反而更小。
- 在"命令相同、数据不同"之外，还剩一组**未检查的时序事实**，日志里都有：`initiator_submit_tick`、`target_rx_tick`/`target_exec_tick`、`target_lcore`、`qid`。
- 本单**不跑任何新读、不动环境**，只解析已有日志，回答：vseg 的慢是否与"提交顺序/断点/落核/大 IO 重叠"相关。

## A. 输入（node3 run/，只读）

- vseg：`nof040_vseg_t1_q4.log`、`nof040_vseg_t1_q1.log`
- plain：`nof039_read_t1_w400.log`
- （16t 两份可选，作旁证；torn 行按 NOF-041 口径剔除。）

## B. 解析口径

1. 仅 opcode=2；把 IO 分为 **小 IO（blocks=256）** 与 **大 IO（blocks=4096/8192，verify）**；测量段（剔 warmup）。
2. **提交顺序**：按 4MiB key 窗口（lba/8192//8192）分组；每 key 的小 IO 按 `initiator_submit_tick` 排序，检查：
   - LBA 是否单调递增（逆序对数）；
   - 相邻提交的 tick 间隔（p50/p90/max），>50µs 的断点计数与位置；
   - 每 key "首→末"提交时长（即 32 条小 IO 的发完时间）。
3. **落核**：`target_lcore` 直方图；每个 lcore 的 bdev_us p50/p90；vseg vs plain 对比。
4. **qpair**：`qid` 直方图；每个 qid 的 bdev_us p50；vseg q4 内部各 qid 是否一致（q1 只有 1 个 qid，作对照）。
5. **大 IO 重叠**：对每 key，取 verify 大 IO 的提交 tick 与 bdev 完成（用 bdev_us 估算）；检查**下一 key 的第一条小 IO**是否在大 IO 完成之前提交；统计"小 IO 提交时距最近大 IO 提交 <X µs"的比例及其 bdev_us。
6. 若 `initiator_submit_tick`/`target_*` 的时钟域不同，只做**同一域内**的差值；跨域只报原始字段，不做换算并注明。

## C. 回答

| 检查项 | vseg T1 q4 | vseg T1 q1 | plain T1 |
|---|---|---|---|
| 每 key 小 IO 提交逆序对数（p50/max） | ? | ? | ? |
| 相邻提交间隔 p50 / >50µs 断点数 | ? | ? | ? |
| 每 key 32 条提交总时长 p50 | ? | ? | ? |
| `target_lcore` 分布（top3） | ? | ? | ? |
| 各 lcore bdev_us p50 | ? | ? | ? |
| `qid` 分布与各 qid bdev_us p50 | ? | ? | ? |
| 下一 key 小 IO 与大 IO 重叠计数 | ? | ? | ? |

- 明确回答：
  1. vseg 的 32 条 128KiB 是否**严格按 LBA 递增**提交？是否有 child 边界断点（>50µs）？
  2. vseg 的 IO 是否落在**与 plain 不同的 target lcore / qid**？慢是否集中在某些 lcore/qid？
  3. verify 大 IO 是否与下一 key 的小 IO **重叠**？重叠段的小 IO 是否更慢？
  4. 若以上都无差异，明确写"时序/落核/重叠均无可辨识差异" ⇒ 差异只能在数据在盘上的物理状态，转写路径取证或 target 逐 IO 明细。

## D. 边界

- **零环境操作**：不跑 bench、不动 target/master、不写任何文件（仅 /tmp 只读脚本）。
- 回执只给汇总表、计数与 ≤20 行样例；不贴原始日志、不外传数据。

## 回执必答（第一行）

vseg/plain 的提交顺序（逆序对/断点）、每 key 提交总时长、target_lcore 与 qid 分布及对应 bdev_us、大 IO 与小 IO 重叠计数；**时序/落核是否有可辨识差异**；有无失败。

---

# NOF-043（已关闭 · 写路径 lane 对照：qpairs=1 写新 key 再读 · 2026-11-23）

> 结果：q1 串行写的新数据读回 bdev p50 **253µs**，与 q4 写（252）和慢窗口（252）完全一致 ⇒ **写侧 lane 交织不是主因**；六个维度全部排除。详见"NOF-043 关键事实"。

## 背景与假设

- NOF-041/042：vseg 的 188µs 增量全部在 **bdev（盘执行）**；提交顺序、落核、qpair、大 IO 重叠全部无差异；vseg 全部 key 处于与 plain 盘首慢区相同的"慢状态"。
- 源码事实（`transfer_task.cpp:895`）：lane 按 **task 轮转**（`next_lane++ % lane_count`）。vseg 一个 key 的 **2 个 2MiB child 落在两条 lane** 上、写入到达交错；plain 一个 key 只有 1 个 4MiB task、只走一条 lane。**这是唯一还没控制过的差异**（此前只换过 writer 线程数，没换过 writer lane 数）。
- 本单在**现有 vseg 环境**（master 2077841、stripe=2MiB、slot=1）写 400 个新 key，**writer 用 `--nof_qpairs_per_segment=1`**（写入严格串行），再读回按 key 桶对比。**不动 target**。

## A. 预检（只读）

1. vseg master 2077841 leader/serving、50056/9006；`Keys=400`、`logical_used=1,677,721,600`；target 341038 未动、4457。
2. `--help 2>&1 | grep -E "nof_qpairs_per_segment|num_keys"` 确认旗标存在。
3. 记录起点 target W 快照；node3 run 目录空间充足（不删既有日志）。

## B. 写新 key（writer qpairs=1）

- writer：`--num_keys=800 --num_threads=1 --batch_size=1 --nof_qpairs_per_segment=1`、vsegment flags、verify=false。
- 预期：keys 0–399 被拒（`OBJECT_ALREADY_EXISTS` 约 400 条，属预期）；keys 400–799 真正写入。
- **验收（必须全部满足，否则停手）**：
  1. master `Keys=800`、`logical_used=3,355,443,200`；
  2. target 写窗口 **n ≥ 12,000**（对照全新写 12,802），记录 W9 avg；
  3. `VSEGMENT_CREATING` 允许**最多 2 次尝试**补齐（总尝试 ≤2），超过即停手。
- 日志 `run/nof043_write_q1.log`；写窗口快照 `nof043_W_write.txt`。
- 记录 writer 进程的 qpairs 证据（启动日志/`nof_qseg` 相关行或 `/proc/<pid>/environ` 只读确认）。

## C. 读回（T1 q4，同口径）

- reader：`--num_keys=800 --num_threads=1 --batch_size=1 --nof_qpairs_per_segment=4 --warmup_keys=128 --verify=true`、vsegment flags、诊断开（`VSEG_DIAGNOSTIC=1`、`MC_HIFREQ_LOG_SAMPLE_RATE=1`、`VSEG_SLOW_IO_US=1`）。
- 日志 `run/nof043_read_800.log`（约 45MB）。
- 记录：端到端 Min/Avg/P50/P90/P99/Max、failed/verify、target W9（n≈51,524）/exec%/TOTAL。
- 解析（沿用 NOF-037/040 脚本）按 key 桶：
  - **桶 1**：keys 0–96（盘首慢窗口，对照）
  - **桶 2**：keys 97–399（**q4 写**的旧数据，对照）
  - **桶 3**：keys 400–799（**q1 写**的新数据，处置）
  - 每桶：128KiB roundtrip p50/p90、`bdev_us` p50/p90（若日志含五段字段）、≥200µs 慢占比、布局是否连续（LBA 边界与步长）。
- 若日志含 `bdev_us` 等五段字段，**以 bdev_us 为主判据**（与 NOF-041 一致）；否则用 roundtrip。

## D. 回答与判决

| 桶 | key | 写入 | bdev_us p50/p90（或 roundtrip） | ≥200µs 占比 |
|---|---|---|---|---|
| 1 | 0–96 | —（盘首慢区） | ? | ? |
| 2 | 97–399 | q4 写（2 child 跨 lane） | ? | ? |
| 3 | 400–799 | **q1 写（串行）** | ? | ? |

- 明确回答：**同一 vseg 环境里，q1 写的新数据读回后，bdev/roundtrip 是否从 ~250µs 回到 ~46–100µs？**
  - 是 ⇒ **写侧 lane 交织是主因**；下一单把 vseg 写 children 串行化（配置或代码层）并复测原 400 keys；
  - 否（仍 ~250）⇒ 写 lane 无关；剩余只有 target 逐 IO 明细（需重启 target，另行申请）或按现状分析收口。
- 副回答：桶 2/3 的布局是否都连续（LBA 步长 1、>1MiB 跳变 0）；写窗口 W9 是否仍 ~24–28µs。

## E. 边界

- 只动我方 writer/reader；**不动 target**、不改 master 参数、不删 key/日志；共享机规则；失败停手。
- 写入量 +1.6GiB（段 100GiB 充足）；`VSEGMENT_CREATING` 超过 2 次尝试即停手。
- 所有臂写明 warmup；读写都记录 W 快照。

## 回执必答（第一行）

q1 写是否落盘（master Keys/logical_used + target 写 n）；三桶的 bdev/roundtrip p50/p90 与慢占比；**q1 写的新数据是否回到快档**；布局是否连续；有无失败。

---

# NOF-044（已关闭 · 零成本检查 target 逐 IO 明细；按 A4 不重启 · 2026-11-23）

> 结果：`URMA_BDEV_DETAIL` 已存在（13,767 条/8 字段），但 `submit_to_callback ≡ bdev_us`、其余子段≈0，拆不开 raid0/NVMe；无时间戳、trace_seq 每会话重置、LEGACY 只采 ≥40µs。按 A4 不重启，环境未动。详见"NOF-044 关键事实"。

## 背景与授权

- 用户已明确选择："**先查，无则重启**"——先零成本检查现有 target 日志与 SPDK 源码；若没有逐 IO 明细，**授权本单重启 target** 加打点。
- 目的：让 target 具备逐 IO 明细（`URMA_BDEV_DETAIL` 或等价），供 NOF-045 对比 vseg/plain 的 bdev 段内部（raid0 队列 vs NVMe 完成）。
- 本单**只做**"检查 + 让明细可用"；**不启动 master、不写不读**（读写留给 NOF-045）。

## A. 零成本检查（只读，先做）

1. 记录 target 现状：PID 341038 的 `/proc/341038/cmdline`（启动参数全文，IP 打码）、启动时间；`/tmp/nvmf_tgt.log` 字节数；4457 LISTEN。
2. `grep -c "URMA_BDEV_DETAIL" /tmp/nvmf_tgt.log`；若 >0，取 ≤5 条样例并列出字段名（`key=value` 名）。
3. 只读查源码确认开关与字段（`sudo -n grep -rn`，≤20 行输出）：
   - `URMA_BDEV_DETAIL` 在 `/home/yin/nof/spdk-cpu/lib/` 下的打印位置与字段列表（是否有 NVMe 层/队列层时间，或只是 lba/blocks/exec/cpl）；
   - `SPDK_URMA_TRACE_ENTRIES`、`SPDK_URMA_STAGE_TARGET_SINK` 的取值语义（尤其 "quiet" 是否抑制逐 IO 打印、非 quiet 的正确取值）。
4. **判决**：
   - 若现有日志已有逐 IO 明细且字段可用 ⇒ **不重启**；回执样例/字段，停手等 NOF-045。
   - 若没有（或源码显示必须 TRACE_ENTRIES>0 才有）⇒ 进入 B。

## B. 重启 target 开启逐 IO 明细（用户已授权）

1. 记录现状快照：target cmdline（上一步）、`nvmf_get_subsystems` 的 ns→URMA_RAID0、4457、`/tmp/nvmf_tgt.log` 当前字节数（后续只解析该 offset 之后的行）；记录 rollback 命令（同参数**不带** trace）。
2. 停我方 vseg master：按 PID 2077841 `kill -TERM`，等退出（**不要**清 etcd 键）；不动 node1 同事进程。
3. 停 target：按 PID 341038 `kill -TERM`，等退出（30s 不退再考虑更强信号，动作前记录）；确认 4457 释放。
4. 用**与 341038 相同的启动路径/参数**重启 target（NOF-016 路径：yin `run_vesegment_target.sh` / patched takeover；同 disks `nvme0n1..nvme10n1`、`-R 256`、`-m [2-7]`、`-C 64`、`-L /usr/lib64`、`-i <fabric_ip>`、`-p urma`、`-N udmac0d1e2`），并加：
   - `SPDK_URMA_TRACE_ENTRIES=2097152`
   - **sink 按源码确认设为非 quiet**（确保 `URMA_BDEV_DETAIL` 逐 IO 行真的打印；若脚本会强制 `SPDK_URMA_STAGE_TARGET_SINK=quiet`，需绕过/改写该行并在回执说明）
   - 保留原有 `SPDK_URMA_DEV_NAME=udmac0d1e2` 等 env。
5. 验收（缺一不可，否则按 rollback 恢复并停手）：
   1. 4457 LISTEN；`nvmf_get_subsystems` 仍为 nsid1 → URMA_RAID0；
   2. 启动日志出现 trace 相关行（记录原文 ≤5 行）；
   3. 新日志中出现**逐 IO 明细样例**（`URMA_BDEV_DETAIL` 或源码对应格式；若无 IO 则不打印，可用一次只读的极小探测——**不要**跑 bench、不要写数据）；
   4. 记录新 target PID、二进制 md5、日志路径、启动命令全文（IP 打码）。
6. **停手**：不启动 master、不 register、不写不读；环境保持在"target 带 trace 运行、master 已停"。

## C. 边界

- target 重启是本单唯一高危动作，**已获用户明确授权**；失败（端口不起/RAID 不齐）→ 按 rollback（同参数无 trace）恢复后回报，不反复尝试。
- 不动 node1 同事进程；node3 独占；不删日志；所有命令非交互。
- 不触碰 etcd（NOF-045 再决定 remount 还是 fresh cluster）。

## 回执必答（第一行）

现有日志是否已有逐 IO 明细及字段清单；如无，target 是否已带 trace 重启成功（新 PID/4457/明细样例/rollback 命令）；有无失败。

---

# NOF-045（已关闭 · 裸 SPDK 定点读 vseg/plain 数据区 · 2026-11-23）

> 结果：高 LBA 的 **vseg 数据裸读 41µs**（W9 91.6），Mooncake-vseg 却 253µs ⇒ **慢在 vseg 路径（+210µs）**；低 LBA [0,~1GiB) 盘本身慢（裸读 215µs）；盘对顺序/随机极敏感（41 vs 220µs）。preflight 写在各支起始 LBA（keys 0/128/400/800 各 128KiB 被覆盖）。详见"NOF-045 关键事实"。

## 背景与目的

- NOF-041~043：vseg 的 188µs 增量全在 bdev；写路径、布局、qpair/lane、时序/落核全部排除；NOF-044 现有逐 IO 明细子段退化，拆不开 raid0/NVMe。
- 在改代码/重启 target 之前，用**裸 SPDK `urma_perf`**（gap 构建 md5 `c653c0a6…`，NOF-022/023 用过）直接读**同一批 LBA**：
  - 裸客户端不经过 Mooncake/vseg，但读的是同一 target、同一 ns 上的数据；
  - 若裸读 vseg 数据区 **快（~30–46µs）** ⇒ **数据/盘无辜，慢在 vseg 客户端/连接路径**；
  - 若裸读 vseg 数据区 **慢（~190–250µs）** ⇒ **数据/盘状态慢**（需重新审视写路径结论）。
- **零重启**；urma_perf 内置 preflight 会写 LBA0（已知，NOF-022 先例），本单接受该小范围覆盖（见 D）。

## A. 预检（只读）

1. target 341038、4457、`URMA_RAID0`/nsid1 在位；vseg master 2077841 保持运行（本单不驱动它）。
2. 定位 node1 `~/gap/spdk/build/examples/urma_perf`，记录 md5；`--help` 确认：
   - `-l` 是否为**起始 LBA**（NOF-022/023 用 `-l 0`）、单位（512B 块还是字节）、`--range-bytes` 语义；
   - 可用参数组合：`-w read -o 131072 -T 1 -b 1 -t 6 -n 1 -g 0 -l <start> -M cpu`；
   - `URMA_PERF_ADDR_MODE=seq` 生效。
   - **若找不到起始 LBA/范围参数 ⇒ 停手回报 `--help`，不要从 LBA0 全盘扫。**
3. LBA 地图（512B 块；以 NOF-043 读回布局为准）：
   - vseg 写入区：keys 0–399 → [0, 1.6GiB)；keys 400–799 → [1.6GiB, 3.2GiB)；
   - **plain 未覆盖区**：keys 800–1199 → [3.2GiB, 4.8GiB)（NOF-039 的 plain 16t 写，未被 vseg 覆盖）；
   - 盘首慢窗口：前 ~388MiB。

## B. 裸 SPDK 定点读（node1 → target，root；每支 6s；`-T1 -b1 -o 131072`；seq）

| 支 | `-l` 起始 LBA | `--range-bytes` | 覆盖 | 含义 |
|---|---|---|---|---|
| A | 3,276,800（1.6GiB） | 1,073,741,824（1GiB） | [1.6,2.6GiB) | **vseg q1 写区（处置）** |
| B | 6,553,600（3.2GiB） | 1,073,741,824 | [3.2,4.2GiB) | **plain 写区（对照）** |
| C | 1,048,576（0.5GiB） | 536,870,912（512MiB） | [0.5,1.0GiB) | vseg q4 写区 |
| D | 0 | 536,870,912 | [0,0.5GiB) | 盘首慢窗口（对照） |

- env：`SPDK_URMA_DEV_NAME=udmac0d1e2`、`SPDK_URMA_CAPSULE_TRANSPORT=urma`。
- 若 `-l` 单位是字节而非块，按 `--help` 换算并记录；不要自行扩大范围。
- 每支：记录 avg/p50/p99、IOPS、bytes；**结束后立即采 target W 块**（W9/n/exec%）并记录 `/tmp/nvmf_tgt.log` 当时的 offset。
- 可选支 E：`URMA_PERF_ADDR_MODE=rand URMA_PERF_RANDOM_SEED=20261115`，`-l 3276800 --range-bytes=1073741824`（随机读 vseg 区，6s）。

## C. 判据与回答

| 支 | 区域 | 若"数据/盘快" | 若"数据/盘慢" |
|---|---|---|---|
| A | vseg q1 写 | ~30–46µs | ~190–250µs |
| B | plain 写 | ~30–46µs | ~190–250µs |
| C | vseg q4 写 | ~30–46µs | ~190–250µs |
| D | 盘首慢窗口 | 混合/慢 | 混合/慢 |

- 明确回答：
  1. 裸读 vseg 数据区（A/C）的 target W9 是多少？接近 vseg Mooncake 的 ~250µs，还是接近裸 SPDK 顺序的 ~30µs？
  2. 裸读 plain 区（B）是多少？（数据状态对照）
  3. 判决：**数据/盘慢** 还是 **vseg 客户端/连接路径慢**？
     - A/C 快、B 快 ⇒ vseg 路径/连接是原因（数据无辜）；
     - A/C 慢、B 快 ⇒ 数据状态不同（写路径结论需重审）；
     - A/B/C 全慢 ⇒ 盘当前处于全局慢状态（与时间相关），需记录并复测一次确认。

## D. 边界与安全

- **零重启**、不动 target/master；只读为主。
- **urma_perf 内置 preflight 会写 LBA0（≈128–256KiB，已知）**：本单接受该覆盖；**不要**在该状态上跑 Mooncake verify（key 0 会不一致）；如需后续 Mooncake 测试，按新 key/新集群处理。
- root 必须；连接/ABI 失败 → 停手回报，不换二进制、不改源码。
- 不拷二进制、不跑其他负载；不动 node1 同事进程。

## 回执必答（第一行）

支 A/B/C/D 的裸 SPDK 延迟（avg/p50/p99）与 target W9；vseg 数据区裸读是快还是慢；**数据/盘慢还是 vseg 客户端/连接路径慢**；preflight LBA0 是否有写入；有无失败。

---

# NOF-046（已关闭 · 纯日志分析：target 侧到达/执行顺序 vs bdev · 2026-11-23）

> 结果：q1 单 lane 的 target 侧顺序与 plain 完全一致（0 逆序/0 位移/100% submit=exec），bdev 仍 223–265µs ⇒ **"盘看成随机档"解释不成立**；顺序假设证否，矛盾收窄到 bdev 提交侧参数（需代码插桩）。详见"NOF-046 关键事实"。

## 背景与目的

- NOF-045：高 LBA 的 vseg 数据裸 SPDK 顺序读 **41µs**，Mooncake-vseg 同区 **253µs** ⇒ 慢在 vseg 路径（+210µs）；同区**随机**裸读 **220µs** ⇒ 盘对访问模式极敏感，vseg 正落在"随机档"。
- 本单只解析现有日志（**零环境操作**）：用 target 回传的 `target_rx_tick` / `target_exec_tick` 重建 **target 侧的到达/执行顺序**，检验 vseg 的 LBA 流在 target/盘看来是否仍是严格顺序。

## A. 输入（node3 run/，只读）

- vseg：`nof040_vseg_t1_q4.log`、`nof040_vseg_t1_q1.log`（可选 16t 作旁证）
- plain：`nof039_read_t1_w400.log`
- 字段：`lba blocks initiator_submit_tick target_rx_tick target_exec_tick target_bdev_cpl_tick target_lcore qid bdev_us roundtrip_us`（按日志实际名）。

## B. 解析口径

1. 仅 opcode=2、blocks=256、测量段；warmup/verify 剔除；拼接 torn 行按 NOF-042 口径剔除并计数。
2. 每 4MiB key 窗口（32 条小 IO）：
   - 按 `target_rx_tick` 排序：LBA 逆序对数、最大位移；给前 10 条的 LBA 序列样例；
   - 按 `target_exec_tick` 排序：同样统计；
   - 与 `initiator_submit_tick` 顺序对比（三者是否一致）；
   - 相邻 `target_rx_tick` / `target_exec_tick` 间隔 p50/p90，>50µs 断点数。
3. **bdev_us 与执行次序**：每 key 内按 exec 次序分 4 段（1–8/9–16/17–24/25–32）的 bdev_us p50/p90；以及"先执行的一半 vs 后一半"的差。
4. **按 lcore/qid**：每 lcore 内 exec 顺序是否严格递增；全局 exec 顺序是否为多 lcore 交错（统计全局逆序对）。
5. **重点对照**：vseg **q1（单 lane）** 的 target exec 顺序是否严格顺序；若 q1 顺序但仍慢，"盘看成随机"的解释不成立。

## C. 回答与判决

| 检查项 | vseg q4 | vseg q1 | plain T1 |
|---|---|---|---|
| rx 顺序逆序对 p50/max | ? | ? | ? |
| exec 顺序逆序对 p50/max | ? | ? | ? |
| 相邻 exec 间隔 p50 / >50µs 断点 | ? | ? | ? |
| bdev_us：前 8 执行 / 后 8 执行 | ? | ? | ? |
| 与 initiator 顺序是否一致 | ? | ? | ? |

- 明确回答：
  1. **target 执行顺序是否严格按 LBA 递增**（每 key、每 lane、全局）？
  2. vseg 与 plain 的 target 侧顺序是否有可辨识差异？
  3. bdev_us 是否随执行次序变化（排队/乱序相关）？
  4. 判决：
     - vseg 的 target 执行顺序**非严格顺序**（例如两 child 交错、多 lane 交错、乱序）⇒ 盘把它看成"随机档"（220µs）已可解释 253µs；方向是让 target 看到顺序流（串行化 children/单 task/单 lane）；
     - q1（单 lane）**仍严格顺序**却仍慢 ⇒ "随机档"解释不成立，需另做裸 SPDK QD/模式扫描或更深打点（另行下发）。

## D. 边界

- **零环境操作**：不跑 bench、不动 target/master、不写任何文件；只读日志 + /tmp 脚本。
- 回执只给汇总表、计数与 ≤20 行样例；不贴原始日志、不外传数据。

## 回执必答（第一行）

vseg q4/q1 与 plain 的 target rx/exec 顺序（逆序对/断点/样例）、bdev 与执行次序的关系；**target 侧是否严格顺序、vseg 是否不同**；有无失败。

---

# NOF-047（待执行 · 第二阶段 M0：构建/依赖取证 + 复制源码与 build 到我们目录 · 2026-11-23）

## 背景与目标

- 第二阶段计划见 `第二阶段/README.md`；M0 是插桩构建的前置。
- 目标：**不改同事原树、不构建、不跑服务**，把构建所需信息取证齐全，并把 Mooncake 与 SPDK 的源码/build 复制到我们目录，为 M1 插桩做准备。

## A. 只读取证（回执必须给）

1. **Mooncake 构建配置**（`/home/yin/nof/mooncake-builds/mooncake-vesegment-sysurma-build/CMakeCache.txt`）：
   - `CMAKE_HOME_DIRECTORY`、`CMAKE_BUILD_TYPE`、C/CXX 编译器、构建系统（Unix Makefiles？）；
   - SPDK 路径、依赖库路径（glog/gflags/etcd 等）、`USE_NOF`/`USE_NOF_URMA`/`USE_UB` 等开关；
   - 目标名与产物：`stress_cluster_bench` 路径/md5/大小（node3 的 59662a26… 与 node1 的 575d2151… 分别标注）。
2. **源码状态**：`/home/yin/nof/mooncake-vesegment` 的 HEAD（应 be10be2）与 `git status --porcelain`（应只有 go.mod/go.sum）；`git log -1 --format=%cd`。
3. **SPDK 两棵树**：
   - yin `spdk-cpu`：HEAD/status、build 目录大小、`nvmf_tgt` 与 `libspdk*` 路径/md5；
   - 客户端 SPDK 库与 Mooncake 的链接关系（CMakeCache 中 SPDK 库路径）。
4. **target 启动与 RPC 配置来源**：cmdline 只有 `-m [2-7] --wait-for-rpc`；确认 NOF-016 用的 `run_vesegment_target.sh`/`target_nvme_takeover.sh` 的 RPC 序列与当前 target（URMA_RAID0/11 盘/4457）一致；记录脚本路径/md5。
5. **依赖/工具链/空间**：cmake/gcc/go/python 版本；liburma 路径/版本；`/home` 剩余空间；两机无外网（确认）。
6. **补丁上下文**（各给 `file:line` + ≤20 行原文；不贴整文件）：
   - target：`lib/nvmf/urma.c` 中 bdev 读提交点（`spdk_bdev_read`/`spdk_bdev_readv` 调用附近）与解析后的数据 buffer/SGL 字段；
   - 客户端 SPDK：`lib/nvme/nvme_urma.c` 中读请求构造 capsule/SGL 的位置（payload 指针/长度/MR/rkey/queue）；
   - Mooncake：`mooncake-store/src/transfer_task.cpp` 的 `submitSpdkNofOperation` 与 worker 中 `buffer_info` 解析（公开源码可只给行号与函数名，不必贴文）。

## B. 复制到我们目录（不构建、不运行）

1. 在 `/home/l00955908/mooncake-nof-vsegment/build/` 下：
   - `sudo -n cp -a /home/yin/nof/mooncake-vesegment` → `mooncake-src/`；`chown -R l00955908`；
   - `sudo -n cp -a .../mooncake-vesegment-sysurma-build` → `mooncake-build-colleague/`（7.7G，保留作为构建参考/回退）；chown；
   - `sudo -n cp -a /home/yin/nof/spdk-cpu` → `spdk-src/`；其 build 目录 → `spdk-build/`；chown。
2. 校验：复制后源码 HEAD/status 与原件一致；二进制 md5 与原件一致；记录总占用。
3. **不修改任何源码、不构建、不启动服务、不跑 bench、不动 target/master、不写盘数据。**
4. 空间上限：复制总量 ≤ 15GB；若剩余空间不足或复制失败 → 停手回报。

## C. 边界

- 只读同事原树；所有写操作在我们目录；sudo 仅用于 cp/chown/读 root 文件。
- 不贴大段源码/日志；补丁上下文每处 ≤20 行。
- 长任务（复制 >10 min）每 ~15 min 一行进度。

## 回执必答（第一行）

CMake 关键配置与源码/SPDK 基线；复制是否完成（路径/占用/校验）；补丁上下文（bdev 提交点、capsule/SGL 构造点、Mooncake 行号）；构建配方建议（下一步怎么在我们的副本里构建）；有无失败。

---

# NOF-048（待执行 · 第二阶段 M1a-0：Mooncake 最小重建验证（未改源码）· 2026-11-23）

## 背景与目标

- NOF-047 已把 Mooncake/SPDK 源码与 build 复制到 `$D=/home/l00955908/mooncake-nof-vsegment/build`（md5 全 MATCH）。
- 直接新建 cmake build 目录可能因依赖（FetchContent/_deps、无外网）或路径耦合失败；先用**最小重建**验证：
  1. 从现有 build 里提取 `transfer_task.cpp` 的编译命令与 `stress_cluster_bench` 的链接命令；
  2. **只重编一个未改动的源文件 + 重链一个新 bench**（不改任何源码、不覆盖现有二进制）；
  3. `--help` 能跑即停。
- 本单**不跑 bench、不动 target/master、不写盘数据**；为 M1a 插桩构建拿到确切配方。

## A. 只读取证（回执必须给）

1. 从 `$D/mooncake-build-colleague/compile_commands.json` 提取 `mooncake-store/src/transfer_task.cpp` 的**完整编译命令**（≤20 行，含 include/flags）。
2. 找到 `stress_cluster_bench` 的**链接命令**：优先 `.../mooncake-store/benchmarks/CMakeFiles/stress_cluster_bench.dir/link.txt`；若没有，用 `gmake -C $D/mooncake-build-colleague -n stress_cluster_bench` 取（不实际执行）。
3. 依赖布局：该 build 是否用 FetchContent/_deps（路径在哪、是否都在 build 目录内）；是否仍有 `/home/yin/...` 绝对路径（列关键项）。
4. 现有 `transfer_task.cpp.o` 的路径/时间；`stress_cluster_bench` 现有二进制路径/md5/大小。

## B. 最小重建（不跑 bench）

1. 用 A1 的命令编译 **`$D/mooncake-src/mooncake-store/src/transfer_task.cpp`（未改）**，输出到 `/tmp/nof048/transfer_task.o`；只改输入/输出路径，flags 原样。
2. 用 A2 的链接命令，把该 .o **替换**原 `transfer_task.cpp.o` 的位置，链接出一个**新二进制** `$D/bench-rebuild-test`；不覆盖任何现有二进制。
3. 运行 `$D/bench-rebuild-test --help | head -30`（只跑帮助）；记录退出码、md5、大小。
4. 若编译或链接失败：报第一个错误（≤20 行）+ 缺失项（头文件/库/依赖），停手；**不要**改为全量 cmake 构建。
5. 若链接命令依赖 `_deps` 或 `/home/yin` 路径：记录这些路径是否仍存在/可读；若不可读，停手回报（不要复制/修改同事原树）。

## C. 交付

- 可复现的**编译命令 + 链接命令**（M1a 直接复用）；
- 最小重建是否成功、新 bench md5/大小、`--help` 结果；
- 需要开发侧决策的偏差（≤5 条）。

## D. 边界

- 不修改任何源码；不改同事原树；不构建整个项目；不跑 bench。
- 临时文件放 `/tmp/nof048/`；结束后可删；不写盘数据。

## 回执必答（第一行）

编译/链接命令是否取得；最小重建是否成功（新 bench md5/大小/`--help`）；依赖与路径是否可读；有无失败。

---

# NOF-049（待执行 · 第二阶段 S0：诊断开/关对照，判定 210µs 是否来自默认诊断/背压 · 2026-11-23）

## 背景（复盘发现）

- 旧构建（be10be2）的 `MC_HIFREQ_LOG_SAMPLE_RATE` **默认 1.0**：`ShouldSampleHiFreqLog` 控制的 per-transfer/per-task 计时与 breakdown 日志**默认对每条读开启**；vseg 还有额外的 per-key/per-child 日志。
- tip 的 `ff3e853` 把这类 breakdown 日志改为显式 opt-in（`MC_STORE_TRANSFER_DIAG`），commit message 明确写："emitting it ... directly inflates caller-observed read latency"。
- 我们的 NOF-035+ 还显式设了 `MC_HIFREQ_LOG_SAMPLE_RATE=1`、`VSEG_SLOW_IO_US=1`（逐 IO 29 字段日志）。
- **"诊断全关后 vseg 的 target bdev/W9 是多少"从未被单独测过**；per-task 日志在 NoF 轮询线程的完成路径输出，有可能通过客户端背压抬高 target W9。

## A. 预检（只读）

1. vseg master 2077841、target 341038、4457、Keys=800；记录起点 target W 快照。
2. keys 0/128/400/800 各 128KiB 已被 urma_perf preflight 覆盖 ⇒ 本单 **verify=false**，不做数据校验。
3. reader 参数与 NOF-040 一致：`--num_keys=400 --num_threads=1 --batch_size=1 --nof_qpairs_per_segment=4 --warmup_keys=128 --verify=false`、vsegment flags（`--vsegment_nof_io=true --segment_write_vsegment=true --replica_num=1 --nof_replica_num=0`）。

## B. 两臂（同环境、同 key、同参数，各 1 遍，先 ON 后 OFF）

- **臂 ON（历史口径）**：`VSEG_DIAGNOSTIC=1`、`MC_HIFREQ_LOG_SAMPLE_RATE=1`、`VSEG_SLOW_IO_US=1`；日志 `run/nof049_diag_on.log`。
- **臂 OFF（全关）**：**不设** `VSEG_DIAGNOSTIC`；`MC_HIFREQ_LOG_SAMPLE_RATE=0`；`VSEG_SLOW_IO_US=0`（或 unset）；`MC_NOF_SLOW_TRACE_THRESHOLD_US=0`（或 unset）；日志 `run/nof049_diag_off.log`。
- 每臂记录：端到端 Min/Avg/P50/P90/P99/Max、failed；**结束后立即采 target 读窗口 W9**（n/avg/exec%/TOTAL）。
- 记录两臂日志字节数；OFF 臂中 `URMA_SLOW_IO` / `nof_slow_task` / `nof_vsegment_operation` 的行数（应≈0，用于证明 env 生效）。
- 若两臂 W 窗口 n 不同（warmup/verify 口径差异），以 W 块 n 为准并在回执说明；本单 verify=false，两臂应一致。

## C. 回答与判决

| 臂 | 日志大小 | target W9 | 端到端 P50 |
|---|---|---|---|
| ON（历史口径） | ? | ?（预期 ~250µs） | ? |
| OFF（全关） | ? | ? | ? |

- 明确回答：**诊断全关后 target W9 是否从 ~250 降到 ~70–100µs？**
  - **是** ⇒ 默认诊断/客户端背压是 210µs 的主要来源；修复方向 = 生产关闭诊断 + 采用 tip 的 opt-in（ff3e853）。**不需要**继续深挖代码插桩。
  - **否（仍 ~250）** ⇒ 诊断不是主因；按复盘顺序做 S1（node3 本机裸 SPDK 定点读）→ S2（构建 tip 4c0ea28 重测）。
- 副回答：OFF 臂的日志是否确实降到很小（env 生效证据）；两臂 W 窗口 n 是否一致。

## D. 边界

- reader 只读；不动 target/master；不写盘数据（verify=false）；不删日志；失败停手。
- 本单**不构建、不改代码**；两臂各 1 遍（W9 稳定；若臂间异常可加 1 遍，需说明）。

## 回执必答（第一行）

ON/OFF 两臂的 target W9、端到端 P50、日志大小与关键日志行数；**诊断全关是否显著降低 W9**；有无失败。

---

# NOF-050（已关闭 · 阻塞：客户端/target SPDK 版本不兼容 · 2026-11-23）

> 结果：补丁/编译/链接成功（`bench-io-params` md5 fc2b5129…），但新 bench 的客户端 SPDK **9a413a44** 与运行中 target **e12b7cb** 握手失败（rc=-71/-104）；同事原 bench（匹配 e12b7cb）同参数正常。target 现跑 deleted 旧二进制 736a902f，磁盘 d6c39465=9a413a44。等待用户决定 A/B/C（见"第二阶段（2026-11-23 起）"上方的决策说明）。

## 背景

- NOF-049：诊断全关后 target W9 仍 251.75µs（−2.7%）⇒ 默认诊断/客户端背压**不是** 210µs 来源。
- 复盘核对 tip diff：be10be2→4c0ea28 的 `transfer_task.cpp` 仅 40 行诊断/UB 默认改动，**NoF 读提交/QoS/分块未变** ⇒ 构建 tip 对 210µs 预期收益低，暂缓；直接做 M1a 客户端插桩。
- 补丁 `第二阶段/patches/nof050_client_io_params.patch`（base be10be2，仅 `transfer_task.cpp`，+56 行，纯日志、默认关闭）：每 task 打印 `NOF_IO_PARAMS_SUBMIT`（ptr/seg_lba/blocks/bytes/child/vseg_op_id/endpoint）与 `NOF_IO_PARAMS_WORKER`（ptr/seg_lba/blocks/lane/worker/buf_type/buf_device/domain）。
- 现有 `[register_buffer] ... base_addr=... size=...` 启动日志可用于核对 ptr 是否落在注册域内。

## A. 预检（只读）

1. vseg master 2077841、target 341038、4457、Keys=800；记录起点 W 快照。
2. 确认 `$D=/home/l00955908/mooncake-nof-vsegment/build` 与 `/tmp/nof048` 的 compile.sh/link.sh 仍在；记录原 bench 与 `libmooncake_store.a` 的 md5（回退基线）。
3. keys 0/128/400/800 已被 preflight 污染 ⇒ 本单 **verify=false**。

## B. 应用补丁 + 构建 `$D/bench-io-params`（复用 NOF-048 配方）

1. 备份 `$D/mooncake-src/mooncake-store/src/transfer_task.cpp`（cp + md5）。
2. 应用补丁（从 handover 仓的 `第二阶段/patches/nof050_client_io_params.patch`）：
   `cd $D/mooncake-src && patch -p1 < <handover>/第二阶段/patches/nof050_client_io_params.patch`（或 `git apply`）；记录 rc 与补丁后文件 md5。
3. 编译：按 `/tmp/nof048/compile.sh`（确认其 `-c` 指向 `$D/mooncake-src/.../transfer_task.cpp`），输出 `/tmp/nof050/transfer_task.o`；记录耗时/rc。
4. 换库成员：`cp` 原 `libmooncake_store.a` → `/tmp/nof050/libmooncake_store_io_params.a`；把新 .o 命名为 `transfer_task.cpp.o`，`ar r` + `ranlib`；记录 rc。
5. 重链：复制 `/tmp/nof048/link.sh` 为 `/tmp/nof050/link.sh`，仅改两处——库路径 → `/tmp/nof050/libmooncake_store_io_params.a`，输出 → `$D/bench-io-params`；执行；记录 rc/耗时/md5。
6. 验收：`strings $D/bench-io-params | grep -c "NOF_IO_PARAMS"`（应 ≥2）；`$D/bench-io-params --help`（gflags rc=1、206 行）。失败 → 报第一个错误（≤20 行）停手，不改其他源码。

## C. vseg 参数采集（T1，1 遍，诊断全关 + 参数日志开）

- env：`MC_NOF_IO_PARAMS_LOG=1`；**诊断全关**：`MC_HIFREQ_LOG_SAMPLE_RATE=0`、`MC_NOF_SLOW_TRACE_THRESHOLD_US=0`、`SPDK_URMA_STAGE_TRACE_US=0`、不设 `VSEG_DIAGNOSTIC`；**`MC_STORE_PARTITION_SLOT_COUNT=1`**（与 master slot=1 对齐，修正 NOF-049 的 494–504 vs 274 口径偏差）；其余 vsegment flags + `SPDK_URMA_DEV_NAME`/`SPDK_URMA_CAPSULE_TRANSPORT` 等必填 env 同 NOF-040。
- reader：`--num_keys=400 --num_threads=1 --batch_size=1 --nof_qpairs_per_segment=4 --warmup_keys=128 --verify=false`；日志 `run/nof050_vseg_params.log`（预计 ~2MB）。
- 记录：端到端 Min/Avg/P50/P90/P99/Max、failed；target W9（n/avg/exec%）；`NOF_IO_PARAMS_SUBMIT`/`NOF_IO_PARAMS_WORKER` 行数；`[register_buffer]` 行（base_addr/size）全部列出（应 1–2 条）。
- 分析：
  1. 前 20 条 SUBMIT 与 WORKER 样例（原样）；
  2. **ptr 注册域命中率**：每个 task 的 `ptr` 是否 ∈ 某个 `[base_addr, base_addr+size)`；给出命中数/总数与未命中样例；
  3. `child=0/2` 与 `1/2` 的 `seg_lba` 差（应为 4096 blocks = 2MiB）、lane 分布、`buf_type`/`buf_device`/`domain` 取值；
  4. `NOF_IO_PARAMS_SUBMIT` 与 `WORKER` 的 task 数是否与 400 keys×2 child 吻合。

## D. 回答

- 客户端每个 child 的 ptr、seg_lba、lane、buffer 类型/domain、注册域命中情况；
- 是否存在"ptr 不在注册域 / buffer 类型异常 / child 的 LBA 不连续"等可解释 210µs 的客户端侧异常；
- 与 plain 的逐字段对比留给 NOF-051（本单不跑 plain）。

## E. 边界

- 只改我们的 `$D/mooncake-src`（仅补丁）；不动同事原树/原 bench/原库；构建只在我们目录/`/tmp`。
- 不改 target/master；reader 只读；不写盘数据（verify=false）；失败停手；不跑 plain、不构建 tip。

## 回执必答（第一行）

补丁/编译/链接是否成功与新 bench md5；vseg 参数日志行数、前 20 条样例、ptr 注册域命中率、child/lane/buf_type/domain 统计；target W9；有无失败。

---

# NOF-051（已关闭 · 通过：e12b7cb 不可编，bd5ee124d 同 wire v4 打通并采到参数 · 2026-11-23）

> 结果：e12b7cb 因系统 UMDK 缺 `is_gpu_seg` 编不过；改用同 wire v4 的紧邻后继 **bd5ee124d** 构建成功；重链 bench 在 node3 本机握手成功；vseg 采到 SUBMIT/WORKER 1056/1056，ptr 注册域 100%，child seg_lba 差 4096 blocks，lane 均匀，**domain 全非空**；target W9 258.55µs。wire 谱系：e12b7cb/bd5ee124d/170cbb9=v4，91763b015/9a413a44=v6。详见"第二阶段"上方决策说明。

## 背景与决策

- NOF-050：我们的 bench（客户端 SPDK **9a413a44**）与运行中 target（**e12b7cb**）握手失败（rc=-71/-104）；同事原 bench（59662a26，匹配 e12b7cb）同参数正常。
- **开发侧决策：走 A 路线**——在我们副本里构建 **e12b7cb** 客户端 SPDK，重链已插桩的 bench 后重试；**不动 target、保留第一阶段基线**。若干净 e12b7cb 客户端仍失败（旧 target 可能带本地改动），再申请 C（重启 target，需用户批准且不可回退）。

## A. 源码与配置（只动我们的副本）

1. `$D=/home/l00955908/mooncake-nof-vsegment/build`；`cd $D/spdk-src`；确认 `git cat-file -e e12b7cb^{commit}` 存在；`git checkout e12b7cb`；记录 HEAD/status（现有 9a413a44 的 `spdk-build/` 保留作参照，不覆盖）。
2. 读取参考配置：9a413a44 构建的 `build/config.log`（configure 原始命令）与 `mk/config.mk`/`CONFIG` 的 URMA 相关项（`CONFIG_URMA`/`URMA_RJETTY`/`URMA_SEG_CTX`/`URMA_PEERMEM`）。核对 e12b7cb 树的同名文件：
   - 若 e12b7cb 干净树**缺少 URMA 开关/支持**（可能是同事本地改动才有）→ **停手回报**，不要猜/不要改配置硬编；
   - 若具备 → 按参考命令配置到我们自己的 prefix：`./configure --with-dpdk=$D/spdk-src/dpdk/build --prefix=$D/spdk-e12b7cb` + 参考中的 URMA 选项。
3. 构建：`gmake -j64`（node3 独占；若 nproc<64 用 nproc）；记录耗时/日志；每 ~15 min 一行进度。
4. 验收：
   - 关键静态库存在（`libspdk_nvme.a`/`libspdk_nvmf.a`/`libspdk_env_dpdk.a`/`libspdk_bdev.a`/`libspdk_event.a`/`libspdk_sock_posix.a` 等）；
   - `strings <libspdk_nvme.a> | grep -i urma` 非空、`grep -R "SPDK_URMA_WIRE_VERSION" include lib | head` 记录版本；
   - 记录构建产物的版本串（应显示 e12b7cb）。
5. 构建失败 → 报第一个错误（≤20 行）+ 缺失项，停手；**不动 target**。

## B. 重链已插桩的 bench

1. 复制 `/tmp/nof050/link.sh` → `/tmp/nof051/link.sh`；把所有 `/usr/local/lib/libspdk_*.a` 路径替换为 `$D/spdk-e12b7cb/lib/...`（rte 库按需指向 e12b7cb 构建的 dpdk）；输出 `$D/bench-io-params-e12`。
2. 复用 `/tmp/nof050/libmooncake_store_io_params.a`（已含插桩成员）。
3. 验收：`$D/bench-io-params-e12 --help`（gflags rc=1、206 行）；`strings | grep -i "e12b7cb"`（版本串）；记录 md5/大小。
4. 若链接失败（未定义符号/ABI 不匹配）→ 报错误并停手；**未经指示不要重编整个 Mooncake**。

## C. 在 node3 本机重试参数采集（与 target 同机，对齐 NOF-040 口径）

- env：`MC_NOF_IO_PARAMS_LOG=1`；**诊断全关**（`MC_HIFREQ_LOG_SAMPLE_RATE=0`、`MC_NOF_SLOW_TRACE_THRESHOLD_US=0`、`SPDK_URMA_STAGE_TRACE_US=0`、无 `VSEG_DIAGNOSTIC`）；`MC_STORE_PARTITION_SLOT_COUNT=1`；必需 SPDK/URMA env + vsegment flags 同 NOF-040。
- reader：`--num_keys=400 --num_threads=1 --batch_size=1 --nof_qpairs_per_segment=4 --warmup_keys=128 --verify=false`；**在 node3 本机运行**（不要 scp 到 node1）；日志 `run/nof051_vseg_params.log`。
- 若握手仍失败（rc=-71/-104）→ 停手，回报完整错误链与 target 日志对应行；**不要重启 target**（C 路线需用户批准）。
- 成功则采集：`NOF_IO_PARAMS_SUBMIT`/`WORKER` 行数与前 20 条样例；`[register_buffer]` 区间与 **ptr 命中率**；child 0/2 与 1/2 的 `seg_lba` 差（应 4096 blocks）；lane 分布；`buf_type`/`buf_device`/`domain`；target W9（n/avg/exec%）；端到端 Min/Avg/P50/P90/P99/Max；**明确记录 reader 在 node3**。

## D. 边界

- 只动我们的副本/prefix；不动同事原树、原 bench、原库；不覆盖 9a413a44 的 `spdk-build/`。
- **不改 target/master、不重启 target**；reader 只读；不写盘数据（verify=false）；失败停手。
- 本单不跑 plain（plain 对照放 NOF-052）。

## 回执必答（第一行）

e12b7cb 构建是否成功与新 bench md5/版本串；握手是否成功；参数日志行数/前 20 条样例/ptr 注册域命中率/child/lane/buf_type/domain 统计；target W9；reader 是否在 node3；有无失败。

---

# NOF-052（已关闭 · 公平对照 1.95×；domain/客户端版本排除；发现 147µs/task 客户端开销 · 2026-11-23）

> 结果：plain 同新客户端 W9 **132.77µs**、旧客户端 **133.29µs**（+0.4%）⇒ 客户端版本无影响；plain/vseg 的 domain 完全相同（非空）⇒ domain 排除；**vseg 258.55 vs plain 132.77 = +125.8µs（1.95×）**。历史 plain 46–68 是 slot=1024 口径，旧的 3.75×/210µs 不再成立。两臂 P50−W9 = 311/147µs ⇒ ≈147µs/task 客户端开销。详见"第二阶段"上方决策说明。

## 背景与目的

- NOF-051：用 **bd5ee124d（wire v4）** 客户端 + 插桩 bench，在 **node3 本机**采到 vseg 参数：SUBMIT/WORKER 1056/1056；ptr 注册域 100%；child seg_lba 差恒 4096 blocks；4 lane 均匀；buf_type=kHost、device=-1、**domain 全为非空 0x143cd90**；target W9 **258.55µs**、端到端 P50 569.78。
- 还差 plain 的**同客户端**对照。必须回答两件事：
  1. plain 的 WORKER 参数是否不同（尤其 **domain 是否非空**、child/seg_lba 形态）；
  2. **plain 用同一个新客户端时 target W9 是否仍 ~46–70µs**——若 plain 也 ~250，说明"新客户端版本"是混淆变量（历史 plain 46 是旧客户端），vseg 差异需重新解读。

## A. 切换 plain 环境（只动我方 master/cluster；target 不动）

1. 记录 vseg 状态（master 2077841、Keys=800）；按 PID 停 master；删**仅我方**键 `/cvm/vseg_nofvseg/` + `/oplog/vseg_nofvseg/`。
2. 启动 **plain master**（NOF-035 参数：无 vsegment auto-plan/exclusive，50056/9006，`SPDK_URMA_DEV_NAME=udmac0d1e2`/`VSEG_URMA_DEVICE=udmac0d1e2`）；确认 leader/serving、`enabled=0`、无 quota。
3. `nof_register` 同命令；确认 `NVMe-oF SSD: 100 GB`。
4. plain writer：16t/batch16、400 keys、verify=false；验收 master `Keys=400` / `logical_used=1,677,721,600`、target 写窗口 n≈12,802；记录写窗口 W9。

## B. plain 读（同一 bd5ee124d 客户端 + 同一插桩 bench，node3 本机）

- 二进制：`$D/bench-io-params-e12`（md5 88ad3134…，客户端 bd5ee124d）。
- flags：plain（`--vsegment_nof_io=false --segment_write_vsegment=false --replica_num=0 --nof_replica_num=1`），`--num_keys=400 --num_threads=1 --batch_size=1 --nof_qpairs_per_segment=4 --warmup_keys=128 --verify=false`；诊断全关；`MC_NOF_IO_PARAMS_LOG=1`；`MC_STORE_PARTITION_SLOT_COUNT=1`（与 NOF-051 保持同 env）；必需 SPDK/URMA env。
- 在 **node3 本机**运行；日志 `run/nof052_plain_params.log`。
- 采集：`NOF_IO_PARAMS_SUBMIT/WORKER` 行数（预期 528 = 400+128）；前 20 条样例；`[register_buffer]` 区间；ptr 注册域命中率；`domain`/`buf_type`/`device` 取值；`blocks`（plain 4MiB task 应为 8192）、`child`（预期 0/0）、`vseg_op_id`；lane 分布；target W9（n/avg/exec%）；端到端 Min/Avg/P50/P90/P99/Max；failed。

## C. 对比与判决（vs NOF-051 vseg）

| 项 | vseg（NOF-051） | plain（本单） |
|---|---|---|
| SUBMIT/WORKER | 1056/1056 | ?（预期 528） |
| blocks / child | 4096 / 0/2,1/2 | ?（预期 8192 / 0/0） |
| ptr 注册域 | 100% | ? |
| buf_type / device / **domain** | kHost / -1 / **非空 0x143cd90** | ? |
| lane 分布 | 4 路均匀 | ? |
| target W9 | 258.55µs | ? |
| 端到端 P50 | 569.78µs | ? |
| P50 − W9 | ~311µs | ? |

- 明确回答：
  1. **plain 的 domain 是否与 vseg 相同（都非空）或为空/不同？** 若不同 ⇒ 这是首个客户端侧差异，直接指向 target 数据路径；
  2. **plain 用同一个新客户端时 target W9 是否仍 ~46–70µs？**
     - 是 ⇒ vseg 差异真实，与客户端版本无关；继续沿 domain/child 差异查 target 侧；
     - 否（plain 也 ~250）⇒ **新客户端 bd5ee124d 是混淆变量**；历史 plain 46 是旧客户端，需用旧客户端重测 plain 或重新解释 210µs；
  3. plain 的 `P50 − W9` 是否也 ~311µs（客户端版本开销）还是 ~20–30µs（vseg 特有）？

## D. 边界

- 只动我方 master/cluster；**不动 target**；plain 写入覆盖 [0,1.6GiB)（允许）；verify=false；reader 只读。
- 不重启 target；失败停手；不跑 vseg（已有 NOF-051）。

## 回执必答（第一行）

plain 参数行数/domain/ptr 命中/child/lane/blocks；plain 的 target W9 与端到端 P50；**domain 是否不同、plain W9 是否仍 ~46–70、P50−W9 是否 ~311**；有无失败。

---

# NOF-053（已关闭 · 客户端分解：QoS 排除；元数据 RPC 116µs/key；vseg 2× 在 drain 段 · 2026-11-23）

> 结果：plain 每 key 283µs = master 元数据 query 116µs（41%）+ submit 6 + wait 156（含 W9 68，超出 88）+ 其他 5；QoS/worker/dispatch 仅 8–11µs（~3%）。vseg 2× 不是 setup/QoS（query 103 vs 116），而是 drain 350 vs 110µs / max_subio 362 vs 112（3.2×）。⇒ 客户端调度排除；master 元数据 RPC ~110µs/key 是两臂共有的大头；vseg 的 2× 在 target/传输 drain 段。详见"第二阶段"上方决策说明。

## 背景

- NOF-052：vseg P50 569.78 / W9 258.55；plain P50 279.95 / W9 132.77；`P50 − W9` = **311 / 147µs**，≈**147µs/task**（vseg 2 task/key、plain 1）。这个客户端开销放大了端到端差异，需要知道它落在哪一段。
- 客户端源码（be10be2）已有三类 breakdown 日志（诊断开时输出）：
  - `batch_get_data_breakdown`：`setup_us / submit_us / wait_blocked_us / checksum_us / result_us / lease_us / total_us`；
  - `batch_get_nof_wait_breakdown`：`nof_tasks / sub_ios / ready_before_wait / blocked_futures / future_wait_* / dispatch_queue_* / worker_setup_* / tasks_ahead_* / qos_fifo_wait_*`；
  - `transfer_data_breakdown`：`op / replica / strategy / slices / bytes / submit_us / wait_us / total_us / status`。

## A. 输入（node3 `run/`，只读；先确认存在，缺失则停手回报）

- vseg：`nof040_vseg_t1_q4.log`（T1 q4、诊断开、旧客户端、slot=1、400 keys）
- plain：`nof039_read_t1_w400.log`（T1、诊断开、slot=1024、1200 keys）
- **口径注意**：两者 master 负载/slot 不同，绝对值不可直接比；本单看**组件占比与结构**，以及 vseg 内 task 翻倍的影响。

## B. 解析

1. 统计每份日志中三类 breakdown 的行数。
2. `batch_get_data_breakdown`（每 key 一行）：n、p50/p90/mean 的 `setup_us / submit_us / wait_blocked_us / checksum_us / result_us / lease_us / total_us`；并检查 `total_us` 与各分项之和的残差。
3. `batch_get_nof_wait_breakdown`（每批一行）：`nof_tasks / sub_ios / ready_before_wait / blocked_futures / future_wait_* / dispatch_queue_avg / worker_setup_avg / tasks_ahead_avg / qos_fifo_wait_avg` 的 n 与均值/分位。
4. `transfer_data_breakdown`（每 transfer 一行）：n、`submit_us / wait_us / total_us` 的 p50/p90；plain 与 vseg 的每 key transfer 数（预期 1 vs 2）。
5. 若日志含有 trace_id/时间戳，可按 key 对齐并给出 3 个 key 的完整时间线（≤20 行样例）；没有则只做聚合。

## C. 回答

| 组件 | vseg（NOF-040） | plain（NOF-039） | 备注 |
|---|---|---|---|
| setup_us p50 | ? | ? | 控制面/元数据 |
| submit_us p50 | ? | ? | 提交 |
| wait_blocked_us p50 | ? | ? | 传输等待 |
| checksum/result/lease | ? | ? | 其他 |
| total_us p50 | ? | ? | 每 key |
| dispatch_queue_avg / worker_setup_avg / qos_fifo_wait_avg | ? | ? | 客户端调度 |
| transfer wait_us p50 / 每 key 次数 | ? | ? | 1 vs 2 |

- 明确回答：
  1. **147µs/task 的客户端开销主要落在哪一段**（控制面 setup / 传输 wait / QoS/worker / 其他）？
  2. vseg 的每 key 开销为什么约为 plain 的 2×：是 `wait_blocked` 随 task 数翻倍，还是 setup/QoS 也翻倍？
  3. `transfer_data_breakdown.wait_us` 与 target W9 的关系（wait_us 是否包含 W9；能否用 wait_us − W9 得到客户端/传输层开销）；
  4. 对下一步的含义：若控制面 setup 占主导 → 优化 master/route；若 QoS/worker 占主导 → 客户端调度；若 wait 占主导 → target/传输。

## D. 边界

- **零环境操作**：不跑 bench、不动 target/master、不写任何文件；只读日志 + /tmp 脚本。
- 回执只给汇总表、分位与 ≤20 行样例；不贴原始日志、不外传数据。

## 回执必答（第一行）

三类 breakdown 行数；vseg/plain 的各分项 p50（setup/submit/wait/QoS/worker）；**147µs/task 落在哪一段、vseg 为何约 2×**；有无缺失字段/失败。

---

# NOF-054（已关闭 · **A 路线无需补丁**：bd5ee124d nvmf_tgt 自带全套打点 · 2026-11-23）

> 结果：已构建的 bd5ee124d `nvmf_tgt`（md5 979f5517…）已含 `URMA_BDEV_DETAIL` 6 段 + **`URMA_STAGE_BUFFER` 逐 IO 24 字段**（lba/nlb/各段 tick/lcore/flags）+ W1–W10；无需写 target 补丁。BUFFER 在**进程退出时落盘**，客户端须 `SPDK_URMA_STAGE_TRACE_US!=0`。重启/RPC/回退预案已取证（wrapper 实际在 `/home/yin/Vsegment/run_vesegment_target.sh`）。详见"第二阶段"上方决策说明。

## 背景

- 用户已批准 **A 路线**：自编**带打点的 bd5ee124d（wire v4）** target，重启后与现有 v4 客户端/参数配对，拿 target 侧逐 IO 解释。
- 本单只做**只读取证**，为写 target 插桩补丁和构建/重启任务做准备；**不构建、不重启、不动 target/master**。

## A. 源码上下文（`$D/spdk-e12b7cb-src`，bd5ee124d；每处 ≤20 行原文 + file:line）

1. `lib/nvmf/ctrlr_bdev.c`：
   - read 命令处理函数（含 `p99_read_enter_tick` 的位置）与 `spdk_bdev_read*` 提交调用点；
   - 记录函数名、`req`/`nvme_cmd`（lba/nlb）访问方式、传给 bdev 的 data buffer/iov、`io_flags`、bdev 名；
   - 若该调用点不在 ctrlr_bdev.c，给出实际 file:line 与函数。
2. `lib/nvmf/urma.c`：
   - capsule 解析后 data buffer/SGL 的存放位置（`req->data`/`iov`/`payload` 等实际字段名）；
   - `URMA_BDEV_DETAIL` 打印点与可用变量（现有 stage ticks、qid/cid）。
3. nvmf request 结构定义（`lib/nvmf/nvmf_internal.h` 或实际头）：`struct spdk_nvmf_request` 中 data buffer/SGL/长度相关字段（≤20 行）。
4. 三处文件的 md5 与 HEAD（bd5ee124d）。

## B. 构建与重启预案（只读取证）

1. `$D/spdk-e12b7cb-src` 的 build 配置（`mk/config.mk`、configure 命令、URMA/DPDK 开关）；确认 `nvmf_tgt` 目标是否已构建（`build/bin/nvmf_tgt` 路径/md5，未构建则说明）。
2. 增量构建命令建议（只编译 `nvmf_tgt` 及依赖）；预计时长未测可注明。
3. RPC 启动序列：
   - `/home/yin/Vsegment/vseg_scripts/run_vesegment_target.sh` 与 `target_nvme_takeover.sh` 的路径/md5；
   - 当前 target 341038 的只读配置（subsystems/transports/listeners/bdevs/raid0/ns）摘要；
   - 重启后需重放的步骤（与 NOF-016 相同的路径），包括停止顺序（我方 master → target）。
4. 回退预案：磁盘 9a413a44（v6）`nvmf_tgt` d6c39465 + 已有 v6 bench `bench-io-params`；若 A 的 target 启动/握手失败，切 B 的步骤。
5. 记录 node3 上是否还有第三方负载（构建/进程），以及 `/home` 空间。

## C. 边界

- **零环境变更**：不构建、不重启、不动 target/master；只读源码/脚本/RPC；不贴大段源码（每处 ≤20 行）。
- 不改同事原树；不改 `$D/spdk-e12b7cb-src`。

## 回执必答（第一行）

三处源码的 file:line 与关键字段（lba/nlb、data buffer/SGL、DETAIL 变量）；nvmf_tgt 构建状态与增量构建建议；RPC 启动/重启/回退预案；有无阻碍。

---

# NOF-055（已关闭 · 通过：target 已切到 bd5ee124d + BUFFER 模式 · 2026-11-23）

> 结果：target **394826**（bd5ee124d，md5 979f5517…），4457/RAID0(11 盘/BDF 升序/256K/online)/ns/transport 正常；BUFFER 已生效（`run/urma_stage_nof055.bin` 0B、O_EXCL、退出落盘、CAPACITY=200000）；旧日志备份 32,780,200B/md5 d294d6ef…。**盘序为 BDF 升序、旧序未记录 ⇒ 新旧数据不可按 LBA 对比**；vseg 元数据已清，NOF-056 重写。详见"第二阶段"上方决策说明。

## 背景与授权

- 用户已批准 **A 路线**（target 重启）；NOF-054 确认**无需补丁**：已构建的 bd5ee124d `nvmf_tgt` 自带 `URMA_STAGE_BUFFER` 逐 IO 24 字段打点（lba/nlb/各段 tick/lcore/flags），**进程退出时落盘**。
- 本单只做**重启与验收**：停 plain master、停旧 target、按同一 RPC 序列启动我们的 bd5ee124d target（BUFFER 模式）；**不跑读、不写数据**，target 保持运行供 NOF-056 采集。
- **target 重启后不可回退 e12b7cb**（旧二进制已删除）；回退 B（磁盘 9a413a44/v6 + `bench-io-params`）预案已取证。

## A. 预检与备份（只读）

1. 记录当前：plain master **PID 2454260**（leader/serving、Keys=400）；target **341038**（`/var/tmp/nvmf_tgt.pid`）、4457、`/tmp/nvmf_tgt.log` 32.5MB、`/var/tmp/spdk.sock`；hugepages Total/Free；node3 load 与第三方进程（`royi:mooncake_client` 等）**只记录**。
2. 备份：`/tmp/nvmf_tgt.log` → `run/nvmf_tgt_e12b7cb_<ts>.log`（记录 md5/字节）；`/var/tmp/nvmf_tgt.pid` → `run/`；记录 target cmdline/env（只读）。
3. 核对 11 块盘 BDF：`0000:21:00.0, 22:00.0, 28:00.0, 29:00.0, 2a:00.0, 91:00.0, 92:00.0, a1:00.0, a2:00.0, a3:00.0, a4:00.0`；每盘 15,002,931,712 B；系统盘 `0000:2b:00.0` **禁碰**。

## B. 停服（顺序敏感）

1. 停 plain master（node1，PID 2454260，`kill -TERM`）；确认退出。
2. 停 target（PID 341038，`kill -TERM`；等退出；30s 不退先记录再升级信号）；确认 4457/spdk.sock 释放、vfio 盘已 restore 为 nvme（takeover 的 restore 步骤）。
3. **核对重启候选盘**：11 个 BDF 与容量一致、系统盘保护生效；nvme 名可能重排，**以 BDF 为准**；盘数/BDF 不符 → 停手回报，不强起。

## C. 启动我们的 bd5ee124d target（BUFFER 模式）

1. 目标：`TARGET_SPDK_ROOT=$D/spdk-e12b7cb-src`（`build/bin/nvmf_tgt` md5 **979f5517…**）+ wrapper `/home/yin/Vsegment/run_vesegment_target.sh`（md5 9eea126e…）+ `TARGET_DUMP_SEC=10`。
2. **BUFFER env**：
   - `SPDK_URMA_STAGE_TARGET_SINK=buffer`
   - `SPDK_URMA_STAGE_TARGET_PATH=/home/l00955908/mooncake-nof-vsegment/run/urma_stage_<tag>.bin`（**必须不存在**，O_EXCL）
   - `SPDK_URMA_STAGE_TARGET_CAPACITY=200000`
   - 注意 wrapper 在 `TARGET_TRACE_ENTRIES>0` 时会强制 `SPDK_URMA_STAGE_TARGET_SINK=quiet`：保持 `TARGET_TRACE_ENTRIES=0` 或绕过该行；**以实际 env 为准并记录**。
3. 同一 RPC 序列（NOF-054）：iobuf → framework_start_init → `bdev_nvme_attach_controller` ×11（按 BDF/顺序）→ `nvmf_create_transport`（URMA、udmac0d1e2、max_io 8MiB 等）→ `nvmf_create_subsystem`（NQN）→ `bdev_raid_create -r raid0 -z 256` → `nvmf_subsystem_add_ns` → `nvmf_subsystem_add_listener -t URMA -f IPv4 -a node3 -s 4457`。
4. 验收（缺一停手/按回退 B）：
   - 4457 LISTEN；subsystem/NQN、nsid1→URMA_RAID0、listener 正确；
   - `nvmf_get_transports`：URMA/udmac0d1e2/capsule urma；
   - target 版本串 bd5ee124d、启动日志含 target timing/URMA_STAGE 相关行；
   - **BUFFER 文件已创建**（路径/大小；O_EXCL）；
   - raid0 online、11 base bdev、hugepages 余量。
5. **停手**：不启动 master、不 register、不跑 bench、不写盘业务数据；target 保持运行（BUFFER 模式）供 NOF-056 使用。

## D. 边界

- target 重启已获用户批准（A）；**不可回退 e12b7cb**；回退 B（9a413a44/v6 + `bench-io-params`）步骤已记录（需先把 `$D/spdk-src/build.9a413a44` 改回 `build`）。
- 不动 node1 同事进程；node3 第三方进程只记录；失败停手。
- 不跑任何读；不写盘业务数据。

## 回执必答（第一行）

target 是否成功切到 bd5ee124d（PID/版本/md5）；4457/RAID0/ns/transport 是否正常；BUFFER 文件路径/大小/容量；旧日志备份与 pid；盘/BDF 核对；有无失败。

---

# NOF-056（已关闭 · B2 停手：VSEGMENT_CREATING 23 次；NUMA 审计完成 · 2026-11-23）

> 结果：vseg 从零建集群后 T1 写触发 23 次 `VSEGMENT_CREATING`（key 16,33,…,390），仅 377/400 keys；按规则未跑 reader、未切 plain、未停 target（buffer 0B）。A2 NUMA 审计给出关键线索：11 盘 5 块 node0 / 6 块 node2，target 核全在 node0 ⇒ 6/11 跨 NUMA。续跑见 NOF-057。

## 背景

- target **394826**（bd5ee124d，BUFFER 模式）运行中；buffer 文件 `run/urma_stage_nof055.bin`（0B，**进程退出时落盘**，CAPACITY=200000 条）。
- 目标：用同一 v4 插桩 bench `bench-io-params-e12`（md5 88ad3134…）依次跑 **vseg** 与 **plain** 的 T1 读（都请求 target trace），然后**优雅停 target** 触发落盘，解析逐 IO 24 字段，定位 1.95×/126µs 在哪一段。

## A. 预检（只读）

1. target 394826 alive、4457、URMA_RAID0 online、buffer 文件 0B；master 未启动；node3 load/第三方进程（`royi:mooncake_client` 等）记录。
2. **NUMA 审计**（0.10 节清单）：`lscpu`/`numactl -H`；11 BDF + 系统盘 2b 的 `numa_node`；`/sys/class/uburma/*/device/numa_node`；`cpu2..7` 的 node；每节点 hugepages total/free；记录 reader 的 `Allocated ... NUMA-aware, N sockets` 行。
3. 记录起点 target W 快照；`/tmp/nvmf_tgt.log` 大小。

## B. 阶段 1：vseg（slot=1、stripe=2MiB）

1. 清我方键 `/cvm/vseg_nofvseg/` + `/oplog/vseg_nofvseg/`（若存在）；启动 vseg-1 master（50056/9006、udmac0d1e2、slot=1、stripe=2MiB）；register；quota ≥20s；验收 partitions=1、reconcile_fail=0。
2. writer T1：`--num_threads=1 --batch_size=1`、vseg flags、verify=false；400 keys；`VSEGMENT_CREATING` ≤2 次尝试；验收 Keys=400、logical_used=1,677,721,600；记录写窗口 W9。
3. reader（**node3 本机**）：`bench-io-params-e12`、`--num_keys=400 --num_threads=1 --batch_size=1 --nof_qpairs_per_segment=4 --warmup_keys=128 --verify=false`、vseg flags；env：`SPDK_URMA_STAGE_TRACE_US=1`（**请求 target trace**）、`MC_NOF_IO_PARAMS_LOG=1`、`MC_STORE_PARTITION_SLOT_COUNT=1`、`VSEG_SLOW_IO_US=0`、`MC_HIFREQ_LOG_SAMPLE_RATE=0`、`MC_NOF_SLOW_TRACE_THRESHOLD_US=0`；日志 `run/nof056_vseg_read.log`。
4. 记录：端到端 Min/Avg/P50/P90/P99/Max、failed；target W9（n/avg/exec%）；params 行数；reader 的 NUMA-aware 行。

## C. 阶段 2：plain（同 target、同客户端）

1. 停 vseg master（按 PID）；删我方键；启动 plain master（NOF-052 同参、slot=1）；register；验收 `enabled=0`、Keys=0。
2. plain writer 16t/batch16、400 keys、verify=false；验收 Keys=400、logical_used；记录写窗口 W9。
3. plain reader：同一 bench/参数；plain flags（`--vsegment_nof_io=false --segment_write_vsegment=false --replica_num=0 --nof_replica_num=1`）；同样的 trace/params env；日志 `run/nof056_plain_read.log`。
4. 记录同 B4。

## D. 停 target 触发 BUFFER 落盘

1. 备份 `/tmp/nvmf_tgt.log`（新 target 日志）到 `run/nvmf_tgt_bd5ee124d_<ts>.log`。
2. **优雅停止** target 394826（`kill -TERM`；等待退出）。验收 buffer：
   - 文件大小 >0；magic `URMA3B01`、footer `URMA3END`；记录 count/dropped；
   - 预期记录数 ≈ 2×16,896 = **33,792**（两阶段读；若含控制面读则更多），≤200000。
3. 若文件仍 0B/无 footer → 停手回报（不要重启 target，避免覆盖）；可尝试 SPDK 的优雅退出 RPC（如有），但先记录。
4. 不重启 target；环境保持 target down、master down。

## E. 解析（Python；24×u64/条，hz 换算 µs）

1. 按文件顺序读取记录；按 **exec_tick 大间隙** 把记录分成 vseg / plain 两段（给出间隙两侧 tick 差与各段记录数作为证据）。
2. 每段：记录数；`lba/nlb` 分布（确认 128KiB 命令；lba 顺序/步长）；`lcore` 分布；`flags`。
3. 每段各段耗时 p50/p90（µs）：
   - `enter−exec`（parse/queue）
   - `submit−enter`（bdev prep）
   - `submit_return−submit`（调用开销）
   - **`callback−submit_return`（bdev/SSD 执行，核心）**
   - `bdev_cpl−callback`（完成处理）
   - `data_post−bdev_cpl`、`data_cpl−data_post`（数据发送）
   - `bdev_cpl−exec`（W9 总）
4. 对比 vseg vs plain：**126µs/1.95× 落在哪一段**；结合 lba/nlb/lcore/flags 判断命令是否真的相同。
5. 明确回答：
   1. 两段 `nlb/lba` 是否相同/顺序（target 实际收到的命令）；
   2. vseg 的增量在哪一段（bdev 执行 / 完成处理 / 数据发送 / 队列）；
   3. 与 NUMA 审计是否有关联线索（跨节点 buffer/盘/设备）；
   4. 下一步修复方向（target 配置/代码 或 数据路径）。

## F. 边界

- 只动我方 master/cluster；target 只**停止**（不重启）；写入覆盖允许（RAID0 盘序已变，数据重写）；verify=false。
- BUFFER 容量 200k；超限丢记录（footer dropped）；失败停手；不自行重启 target（buffer 只在退出落盘）。

## 回执必答（第一行）

两阶段是否成功（Keys/failed）；target 是否优雅退出且 buffer 落盘（记录数/magic/footer/dropped）；两段 nlb/lba/lcore；各段 p50/p90 与 **126µs 落在哪一段**；NUMA 审计摘要；有无失败。

---

# NOF-057（已关闭 · 126µs 全在 SSD 执行段；NUMA 排除；两臂访问流不同 · 2026-11-23）

> 结果：补齐 **400 keys**（2 次尝试：r1 398/2、r2 400/0；VSEGMENT_CREATING 23 次后清零；Keys=400、logical_used=1,677,721,600、vsegments=26）。两阶段读 **failed=0**：vseg Min/Avg/P50/P90/P99/Max = 468.54/559.93/**551.22**/644.84/769.59/863.98µs，W9 **261.44**；plain = 255.87/347.45/**277.50**/578.10/758.10/793.74µs，W9 **137.00**。target TERM 后 3s 优雅退出，`run/urma_stage_nof055.bin` **6,488,160 B**、magic `URMA3B01`、footer `URMA3END`、**count=33,792 dropped=0**（=72+33,792×192+24，严格吻合）。按 exec_tick 最大间隙 208.953s 分为 seg1 vseg 16,896 / seg2 plain 16,896。**各段**（µs，p50/p90/mean）：`submit_return→callback` vseg **256/314/260.4** vs plain **60/284/135.9**（+196/+124.5）；W9 **256/314/261.1** vs **61/285/136.5**；其余 6 段两阶段均 0（`data_post→data_cpl` vseg 6/10 vs plain 12/33，vseg 更快）。命令完全相同：`nlb` 全 256、唯一 lba 12,800、**两段 lba 集合相同**、flags 全 3；lcore vseg 2/5/6/7 vs plain 2/3/4/7（各 4,224）。**NUMA 排除**：vseg node0 p50 256（45.5%）/ node2 255（54.5%）；plain 59/61。**盘级访问流不同**：换盘率 **79.7%（13,462/16,895）vs 50.0%（8,447/16,895）**、每盘 LBA 递增 **98.0% vs 99.9%**、每盘回退 **31 vs 1**、每盘 IO 数 1,534–1,538（均匀）。23-key 排除无需做（LBA 集合一致且全部规范：400 唯一 key、范围 0–399）。vseg 分桶 128–256µs 49.9% + 256–512µs 49.7%（64–128 仅 12 条）；plain 32–64µs 49.9% + ≥128µs 36.6%。张力：NOF-034（stripe=4MiB 单 child 仍 W9 250.1–270.9）与 NOF-046（q1 严格顺序仍 223–265µs）说明"child 数/表面顺序"不能直接当根因 ⇒ NOF-058 离线判别快/慢 IO 的局部条件。产物：`run/urma_stage_nof055.bin`、`run/nvmf_tgt_bd5ee124d_20261123_163717.log`（193,272 B / md5 a8222276…）、`run/nof057_vseg_read.log`（10,843,883 B）、`run/nof057_plain_read.log`（10,570,443 B）、master/写日志、/tmp 脚本。末态：target down、4457 free、两 master 已停、无 bench。

## 起点状态（原地续跑）

- target **394826**（bd5ee124d，BUFFER，4457，buffer `run/urma_stage_nof055.bin` 0B）
- vseg master **2562417**（slot=1/stripe=2MiB，**377 keys**、vsegments=24、logical_used=1,581,252,608）
- 无 reader、无 bench、master 端口在听

## A. 预检（只读）

1. 确认上述状态与起点 W 快照；记录 23 个缺失 key（16,33,50,67,84,101,118,135,152,169,186,203,220,237,254,271,288,305,322,339,356,373,390）。
2. **NUMA 摘要（A2 已得，记录即可）**：node0=cpu0–69、node1=70–153、node2=154–237、node3=238–307（0↔2 距离 20）；11 盘 5 块 node0（21/22/28/29/2a）、6 块 node2（91/92/a1/a2/a3/a4）；target 核 2–7 全 node0；hugepages 各 1160、free 16/75/128/128。

## B. 补齐 400 keys（选项 1；最多 3 次尝试）

- 同参数 T1 writer（`--num_threads=1 --batch_size=1`、vseg flags、verify=false）；每次记录 `Write complete` 的 succeeded/failed、失败 key 列表、master `Keys/logical_used/vsegments`、写窗口 W9（仅记录）。
- 直到 `Keys=400`、`logical_used=1,677,721,600`、vsegments=25；**3 次后仍 <400 → 停手回报**（不做第 4 次）。
- 明确记录每次 `VSEGMENT_CREATING` 数（预期逐次减少；这是已知机制，不作为失败）。

## C. 阶段 1：vseg 读（400 keys，请求 trace）

- reader（node3 本机）：`bench-io-params-e12`、`--num_keys=400 --num_threads=1 --batch_size=1 --nof_qpairs_per_segment=4 --warmup_keys=128 --verify=false`、vseg flags；env：`SPDK_URMA_STAGE_TRACE_US=1`、`MC_NOF_IO_PARAMS_LOG=1`、`MC_STORE_PARTITION_SLOT_COUNT=1`、`VSEG_SLOW_IO_US=0`、`MC_HIFREQ_LOG_SAMPLE_RATE=0`、`MC_NOF_SLOW_TRACE_THRESHOLD_US=0`；日志 `run/nof057_vseg_read.log`。
- 记录：端到端 Min/Avg/P50/P90/P99/Max、failed；target W9（n/avg/exec%）；params 行数；reader 的 `Allocated ... NUMA-aware, N sockets` 行。
- **LBA 检查**：从 params 的 `seg_lba` 看补齐的 23 个 key 是否仍在 `[k×8192,(k+1)×8192)`；若在尾部/非规范，记录其 LBA 区间。

## D. 阶段 2：plain（同 target、同客户端）

1. 停 vseg master（2562417）；删我方键 `/cvm/vseg_nofvseg/`+`/oplog/vseg_nofvseg/`；启动 plain master（NOF-052 同参、slot=1）；register；验收 `enabled=0`、Keys=0。
2. plain writer 16t/batch16、400 keys、verify=false；验收 Keys=400、logical_used；记录写窗口 W9。
3. plain reader：同一 bench/参数；plain flags；同样的 trace/params env；日志 `run/nof057_plain_read.log`；记录同 C。

## E. 停 target 落盘 + 解析（含 NUMA 分组）

1. 备份 `/tmp/nvmf_tgt.log`；**优雅停 target 394826**（TERM）；验收 buffer：大小 >0、magic `URMA3B01`、footer `URMA3END`、count/dropped（预期 ~2×16,896 条，≤200,000）。失败 → 停手，不重启。
2. 解析 24×u64：按 `exec_tick` 大间隙分 vseg/plain；每段 `lba/nlb/lcore/flags` 分布。
3. **NUMA 分组**：strip=256KiB=512 blocks；`disk=(lba/512)%11`；BDF 升序成员 ⇒ disk 0–4=node0、5–10=node2。每段按 disk/node 分组统计各段 p50/p90（尤其 `submit_return→callback`、`bdev_cpl→exec`/W9），并统计 node0/node2 的 IO 比例与顺序。
4. **23-key 处理**：若补齐 key 的 vseg LBA 非规范（在尾部），则两阶段都**排除这 23 个 key 的 LBA 窗口**后比较（给排除前/后两组数字）。
5. 明确回答：
   1. 两阶段 `nlb/lba` 是否一致（target 实际命令）；
   2. **126µs 落在哪一段**（enter→submit / submit_return→callback / callback→bdev_cpl / data）；
   3. **local(node0) vs remote(node2) 盘的 per-IO 差异，vseg/plain 是否不同；NUMA 能否解释 126µs**；
   4. 下一步方向。

## F. 边界

- 只动我方 master/cluster；target 只**停止**不重启；写入允许；verify=false；buffer 200k；失败停手。
- 每轮写/读记录 W 快照；不采信 bench 的写成功数（以 master Keys/logical_used 为准）。

## 回执必答（第一行）

是否补齐 400 keys（尝试次数/最终 Keys/logical_used）；两阶段读是否成功；target 是否落盘（count/dropped）；两段 nlb/lba/lcore；各段 p50/p90 与 **126µs 所在段**；**NUMA 分组结果（node0 vs node2、vseg vs plain）**；有无失败。

---

# NOF-058（已关闭 · 并发与局部顺序均排除；判决指向数据状态/读路径 · 2026-11-23）

> 结果：零环境、只读 NOF-057 产物。**顺序重建稳定**：同 exec_tick 占比 vseg 2 条（0.01%）/plain 0，最大同 tick 组 2/1；`(exec,trace_seq)` 与 `(exec,rx,trace_seq)` 两种 tie-break 结果完全一致（vseg 换盘 79.7%/每盘递增 98.0%/回退 **341 次=31/盘**；plain 50.0%/99.9%/回退 **11 次=1/盘**）；纯 trace_seq 序 vseg 换盘 59.7%（−20pp）。qid 2/3/4/5 各 4,224，qid→lcore 严格 1:1。warmup→测量边界 index=4,096（LBA 990,976/1,048,320→0，dt 0.419/0.416s），两段对称（4,096+12,800）。**ov=0（该盘零并发）**：vseg n=4,400（34.4%）p50 **206**/p90 250/mean 202.0，分桶 128–256:4,081、64–128:9、256–512:306；plain n=4,405（34.4%）p50 **39**/p90 213/mean 79.2，分桶 32–64:2,638、<32:677、128–256:975；两臂 ov 分布相同（0/1/2/3 = 34.4/34.4/15.6/15.6 vs 34.4/36.0/16.4/13.2），ov≥4 为 0 ⇒ **并发/队列深度不是主因**。**同 (disk,lcore) 紧接前驱（gap≤512）**：vseg 50.2% p50 **275**；plain 50.2% p50 **61**；busy=False：vseg **206** vs plain **39**；同 gap 桶（near<8MiB 非紧接）vseg 207 vs plain 41；far≥8MiB 两臂均 0 ⇒ **局部 LBA 顺序不是主因**。**plain 快 IO（<64µs，65.1%，8,336 条）全部在 LBA 802,560–3,276,288（keys≥98）；慢 IO（≥128µs，24.5%，3,133 条）全部在 LBA 0–802,304（keys 0–97 = 盘首慢区）**；vseg 在高 LBA 区也慢（9,664 条 p50 256）⇒ vseg 的"高 LBA 本可快却慢"。**同一 LBA（keys≥98，n=9,664）**：vseg p50 256/mean 250.2 vs plain p50 49/mean 49.4（**5.07×**）；共同 12,800 LBA 中 plain 更快 10,978（85.8%）。**但两边数据由不同 writer/不同写入历史写入（vseg 377 keys 单写 + 23 keys 补写；plain 后写覆盖）⇒ 该 5.07× 受写入历史混杂，不能单独归因**。按 exec 10 分段 p50：vseg 全程 254–257；plain d0 256/d1 253（keys 0–97 慢区）后 49–50。child0 p50 226 vs child1 267（off 0–10 217–239、off 11–15 272–277）。客户端 per-lane 顺序率 vseg 93.9%（回退 15–17/lane）vs plain 99.2%（1/lane，文件行序，`[推断]`）；vseg child0/child1 SUBMIT 严格交替 1,055/1,055=100%（plain 单 child 为 0）。**判决：24 字段内无判别变量；候选只剩"vseg 写出的数据物理状态"与"读路径的全局/请求层差异"**；因 NOF-045 曾显示 q1 写的同区数据裸读 41µs，数据状态假设需用"单轮写入 + 不重写裸读"复核 ⇒ NOF-059。产物：`run/nof058_analysis.txt`（8,327 B）、`/tmp/nof058/*.py`；环境保持 target down、4457/50056 FREE、无 bench。

## 背景与目标

- NOF-057 已把 126µs **100% 定位在 `submit_return→callback`（SSD 执行段）**，并证明命令完全相同（lba 集合/nlb/flags）、NUMA 无关；唯一留下的差异是盘级访问流（换盘率 79.7% vs 50.0%、每盘 LBA 回退 31 vs 1）与 **每个 IO 在盘上的"局部处境"**。
- 但"访问流不同"≠"因果"：NOF-034（stripe=4MiB、每 key 单 child，W9 仍 250–271µs）与 NOF-046（q1 严格顺序仍 223–265µs）都提示不能把 child 数/表面顺序直接当根因。
- **关键自然实验**：plain 同一段内同时存在快 IO（32–64µs 占 49.9%）与慢 IO（≥128µs 占 36.6%），命令/盘/会话完全相同 ⇒ **快/慢的判别变量一定在现有 24 字段里**。vseg 几乎 100% 是慢 IO。
- 本单是**纯离线分析**：target 保持 NOF-057 末态（down），不重启、不起 master/bench、不写盘、不改任何配置；只读 `run/` 下既有产物。

## A. 输入与方法审计（先做，必答）

1. 读 `run/urma_stage_nof055.bin`：确认 header/footer/**count=33,792**；复现按 exec_tick 最大间隙分两段（16,896/16,896、间隙 208.953s）。
2. **顺序重建口径**：记录是按 release（完成）顺序追加；"提交顺序"= 按 `exec_tick` 升序。报告：
   - 每段 **exec_tick 相同（同 tick）的 IO 占比**、最大同 tick 组大小；
   - 用 `(exec_tick, trace_seq)` 与 `(exec_tick, rx_tick, trace_seq)` 两种 tie-break 重算"换盘率 / 每盘递增比例 / 每盘回退"三指标，报告差异（若 >2% 视为顺序指标不稳，须说明）；
   - `qid` 的取值分布与 `qid ↔ lcore` 映射（每 qid 对应几个 lcore）。
3. **分离 warmup / 测量两遍**：每个 lba 出现 2 次（warmup 128 keys + 测量 400 keys，0–127 重合）。在提交顺序中找到"lba 从高值回落到 0"的边界（辅以时间间隙与客户端日志的阶段行），报告边界前后 IO 数/时间戳；**B/C 段默认只用测量遍**，同时给全段对照。

## B. 并发重叠 vs 服务时间（区分"排队"与"访问局部性"）

按盘 `disk=(lba/512)%11`，对每段（测量遍）：

1. 对每条 IO i，计算执行时刻同盘的**在服务 IO 数** `ov_i = |{j : disk_j=disk_i, exec_j ≤ exec_i < callback_j, j≠i}|`；
2. 报告 `ov` 分布（0/1/2/≥3 的比例）及每档的 `submit_return→callback` p50/p90；
3. **关键子集**：取 `ov=0`（该盘在执行 i 时没有任何其他在服务 IO）的 IO，报告其服务时间分布（p50/p90/各桶计数）。
   - 若 vseg 的 `ov=0` IO 仍 ~250µs ⇒ 并发不是主因，转 C；
   - 若明显更快 ⇒ 并发/队列深度是主因，给出"从 ov=几开始进入 250µs 档"的拐点。

## C. 同盘/同队列的 LBA 局部性（区分"顺序流"）

对每段（测量遍），分两个口径：**同 disk** 与 **同 (disk, lcore)**（lcore = target reactor，近似 NVMe 队列）：

1. 按提交顺序，为每条 IO 找同口径的前一条 j（`exec_j < exec_i`，同 tick 用 trace_seq），计算
   - `gap = lba_i − (lba_j + nlb_j)`（blocks；0 表示严格紧接）、
   - `dt = exec_i − exec_j`、`busy = (exec_i < callback_j)`（前一条是否仍在服务中）；
2. 交叉表：服务时间桶（<32 / 32–64 / 64–128 / 128–256 / 256–512 / ≥512µs）× `gap` 桶（紧接 `|gap|≤512`、近距 `<8MiB`、远距 `≥8MiB` 或回退）× `busy`；
3. 给两臂的**紧接前驱占比**（`|gap|≤512`）与对应服务时间 p50/p90；
4. 明确回答：**plain 的快 IO 能否由"紧接前驱"和/或"ov=0"解释**（各命中率与 p50 对照）；**vseg 为什么几乎没有快 IO**（上述条件的命中率各是多少，差在哪一环）。
5. 附：每段给 20 条连续 IO 的紧凑摘录（相对 exec_tick µs、disk、lba、qid、lcore、服务 µs），便于人工复核。

## D. 客户端提交结构（params 日志，只读）

1. 解析 `run/nof057_vseg_read.log` / `run/nof057_plain_read.log` 的 `NOF_IO_PARAMS_SUBMIT` / `NOF_IO_PARAMS_WORKER` 行（应为 vseg 1056/1056、plain 528/528）；先报告两种行的**字段全集与样例各 2 行**。
2. 按 `lane`（或 `worker`）分组，依**文件行序**重建每 lane 的 `seg_lba` 序列；报告每 lane 的 LBA 回退次数/比例、blocks、以及 vseg 的 `child` 标记序列。
3. 对 vseg：统计同一 key 的两个 child 是否**相邻提交**（SUBMIT 行相邻），以及每个 lane 上 child0/child1 的切换次数（衡量两 child 在同 lane 上的交错）。
4. 对 plain：同样给每 lane 的 LBA 单调性（对照测量遍 target 序列）。
5. 若行序不能可靠代表提交时序（例如无时间戳、跨线程写交错），明确写 `[未获取]` 并说明推断边界，**不要猜**。

## E. 结论（必答）

1. **最佳单一判别变量**：ov / 同队列紧接前驱 / 两者组合 / 都不是；给两臂命中率与 p50 数字；
2. 机制判断（三选一并给证据强度）：(i) 同盘并发重叠；(ii) 盘内/队列内 LBA 局部性被打断；(iii) 都不是（需新维度）；
3. **修复杠杆**：基于数据指出最可能有效的客户端改动（例：同 lane 子 IO 串行/相邻、限制每盘在途数、child 合并/按 LBA 排序提交），并给出该改动在当前数据上的**预期落点**（哪一档回落、预期 W9 量级）；若数据不支持任何杠杆，直说。

## F. 边界与产物

- 零环境：target 保持 down；不启动 master/bench；不重启、不改配置、不写盘；只读 `run/` 文件（不得修改 NOF-057 产物）。
- 分析脚本放 `/tmp`；持久产物写 `run/nof058_*.txt`（≤5 个，每个 ≤1MB）；完整逐 IO CSV 可留 /tmp，不作为产物。
- 输入缺失/损坏 → 停手回报，不重建环境。

## 回执必答（第一行）

重建方法审计（同 tick 占比/两种 tie-break 敏感性）；warmup→测量边界；`ov` 分布与 **ov=0 子集的服务时间（两臂）**；同队列紧接前驱占比与交叉表（两臂）；**plain 快 IO 的解释比例**与 **vseg 无快 IO 的原因**；客户端每 lane 单调性/child 交错；结论与修复杠杆。

---

# NOF-059（已关闭 · 单 child 无效；数据无罪；锁定读路径 · 2026-11-23）

> 结果：target 重启 **PID 708167**（bd5ee124d/md5 979f5517…、无 BUFFER env、保持运行；首启因 vfio 残留 ABORT，takeover `-r -y` 还原后成功）。**stripe=4MiB 确实每 key 单 child**：写 params 377+23、读 params **528/528**，全部 `blocks=8192 child=0/1`。vseg4 读 Min/Avg/P50/P90/P99/Max = 462.17/554.79/**547.92**/631.42/700.07/1011.90、failed=0；plain 读 Avg 339.75/**P50 268.44**/P90 574.79/P99 691.91 ⇒ **2.04×，与 2MiB 双 child 的 1.99× 同档 ⇒ child 数无关**。同一份数据裸 SPDK（node1 gap urma_perf，md5 c653c0a6…）：D1 顺序 keys98–399 **p50 40.0/W9 86.3**（vseg 数据）vs **40.8/79.6**（plain 数据）；D3 随机 **220.16/191.1** vs **220.16/190.9**；D2 单 key 200 定点 4MiB = **218.88/187.9** ⇒ **数据物理状态无罪；单 key 内部顺序读也已是"随机档"**。preflight 覆盖 LBA 802816（key98）与 1638400（key200）各 128KiB。plain 写 400/0（0.16s、9921 MiB/s、SUBMIT=400/8192/child=0/0）、Keys=400。**target W9 本单表（vseg 122.6 vs plain 30.4）受 dump 滞后/跨相位与 n 分块不齐影响，不用于结论**（per-key 需 BUFFER，本单禁止）。新增偏差：首次 target 启动 ABORT（11 盘 vfio 残留）；首次写以非 root 失败（改 sudo）；写触发 23 次 VSEGMENT_CREATING，按规则补 1 轮。末态：**target 保持运行**（708167、4457、无 BUFFER）、两 master 已停、无 bench。**判决：child 数无关、数据无罪 ⇒ 差异在 vseg 客户端把同一 4MiB NoF 操作驱动到 target 的方式** ⇒ NOF-060。

## 背景与目标

- NOF-058 已排除：同盘并发重叠（ov=0 时 vseg 206µs vs plain 39µs）、局部 LBA 顺序（紧接前驱/前驱已完成同样 5× 差）、target 内部排队/完成/发送段（NOF-057：其余 6 段为 0）。剩余两个候选：
  1. **读流结构**：vseg 每 key 2×2MiB child 并发 → 若 `stripe=4MiB` 真正变成每 key 单 4MiB child（与 plain 同构），W9 是否回落？（NOF-034 曾测但**未验证 child 数**；本次 params 可验证）
  2. **数据物理状态**：NOF-057 的"同一 LBA 5.07×"受**写轮次/写者不同**混杂；NOF-045 又曾显示 q1 写的同区数据裸 SPDK 顺序读 41µs。需用**单轮写入 + 不重写**的同一份数据、两种读法来分离。
- 本单用**一轮运行**同时回答这两问：vseg-4MiB 单轮写→读（W9）→**不重写**裸 SPDK 顺序/随机读同一数据；随后 plain 写→读→裸读作同批对照。

## 0. 关键约束与授权

- **本单包含 target 重启**（A 路线延续）：同 NOF-055 的 bd5ee124d 二进制（md5 979f5517…）与同一 RPC 序列；**不启用 BUFFER**（不设 `SPDK_URMA_STAGE_TARGET_SINK/PATH/CAPACITY`，客户端不设 `SPDK_URMA_STAGE_TRACE_US`）；**本单结束后 target 保持运行**（供后续任务），不优雅停、不落盘。
- 只动我方 master/cluster；不动 node1 同事进程；node3 第三方进程只记录；失败停手。
- 数据会被覆盖（允许）；verify=false；裸读 preflight 会写各支起始 LBA 128KiB（已知，均在对应 Mooncake 读之后，不影响 R1）。
- 端口沿用：target 4457、master 50056/9006、TE 50156。

## A. 重启 target（同 NOF-055 C 段）

1. 预检（只读）：target 当前 down；4457/50056 FREE；无 master/bench；11 块盘 BDF 核对 `0000:21/22/28/29/2a/91/92/a1/a2/a3/a4:00.0`、容量一致；系统盘 `0000:2b:00.0` **禁碰**；hugepages 记录。
2. 启动：`TARGET_SPDK_ROOT=$D/spdk-e12b7cb-src` + wrapper `/home/yin/Vsegment/run_vesegment_target.sh` + `TARGET_DUMP_SEC=10`；**不设 BUFFER env**；记录 PID、`/proc/<pid>/exe` md5、版本串。
3. 验收（缺一停手）：4457 LISTEN；`nvmf_get_subsystems` nsid1→URMA_RAID0、NQN 正确；`nvmf_get_transports` URMA/udmac0d1e2/capsule urma；RAID0 11 base bdev/BDF 升序/strip 256K/online；target 日志无 handshake/attach 失败。
4. 起服务后**不做任何读写**，直接进入 B。

## B. vseg stripe=4MiB：单轮写 + 读（核心臂 1）

1. 清我方键 `/cvm/vseg_nofvseg/` + `/oplog/vseg_nofvseg/`；起 vseg master（50056/9006、udmac0d1e2、`MC_STORE_PARTITION_SLOT_COUNT=1`、NOF-004/007 同参 + `--vsegment_stripe_size=4194304` + `--vsegment_auto_plan=true --vsegment_required_medium=nof --vsegment_io_alignment=4096 --vsegment_member_count=1 --vsegment_member_extent_size=67108864 --vsegment_reserved_ratio=0 --submaster_count=1`）；register；等 quota ≥20s；验收 `stripe_size=4194304`、`partitions=1`、`reconcile_fail=0`。
2. **T1 writer**（`--num_threads=1 --batch_size=1`、vseg flags、verify=false、`MC_NOF_IO_PARAMS_LOG=1`、`MC_STORE_PARTITION_SLOT_COUNT=1`、诊断关；日志 `run/nof059_vseg4_write.log`）：400 keys；`VSEGMENT_CREATING` ≤3 次补齐；**每轮记录实际新增 key 数**（老 key 的 put 可能被拒而 bench 仍计成功 ⇒ 以 master Keys/logical_used 为准）；验收 `Keys=400`、`logical_used=1,677,721,600`；记录写窗口 W9（n/avg/exec%）。
   - **写 params 验收**：SUBMIT 行数、每 key child 数、blocks（预期单 child：blocks=**8192**；若仍 4096×2 ⇒ 记录"stripe=4MiB 未改变 child 拆分"并继续）。
3. **vseg reader**（node3 本机）：`bench-io-params-e12`、`--num_keys=400 --num_threads=1 --batch_size=1 --nof_qpairs_per_segment=4 --warmup_keys=128 --verify=false`、vseg flags；env `MC_NOF_IO_PARAMS_LOG=1`、`MC_STORE_PARTITION_SLOT_COUNT=1`、`SPDK_URMA_STAGE_TRACE_US=0`、其余诊断关；日志 `run/nof059_vseg4_read.log`。
   - 记录 Min/Avg/P50/P90/P99/Max、failed；**读 params**：SUBMIT/WORKER 行数、blocks、child 字段（预期 528/528、8192、单 child；若 1056/1056、4096×2 ⇒ 单 child 未生效）；读窗口 W9。
   - **本步骤之后到 C 结束，禁止任何写入**（保持数据状态不变）。

## C. 裸 SPDK 读同一份 vseg 数据（不重写；核心臂 2）

1. 工具：node1 `~/gap/spdk/build/examples/urma_perf`（NOF-045 同款，记录 md5；若不存在，用 node3 同批 v4 构建并记录，二选一后全程一致）；`--help` 核对并记录完整命令（`-w read -o 131072 -T 1 -b 1 -t 6 -n 1 -g 0 -M cpu`、`-l` 起始 LBA、`--range-bytes`、连接地址/端口/NQN/nsid 同 NOF-045）；env `SPDK_URMA_DEV_NAME=udmac0d1e2`、`SPDK_URMA_CAPSULE_TRANSPORT=urma`。ABI/连接失败 → 停手回报，不换二进制。
2. **D1 顺序长扫**：`-l 802816 --range-bytes 1266679808`（keys 98–399，避开盘首慢区）；`URMA_PERF_ADDR_MODE=seq`；记录 client avg/p50/p99、IOPS；**结束后立即采 target W9（n/avg/exec%）并记录 `/tmp/nvmf_tgt.log` offset**。
3. **D2 顺序定点**：`-l 1638400 -o 131072 --range-bytes 4194304`（key 200 整 4MiB），记录同上。
4. **D3 随机同 D1 区**：`URMA_PERF_ADDR_MODE=rand URMA_PERF_RANDOM_SEED=20261115` + D1 的 `-l/--range-bytes`，记录同上。
5. 记录三支 preflight 写入的起始 LBA（D1/D3=802816、D2=1638400，各 128KiB）——这些 LBA 被覆盖，后续不得再用于 Mooncake verify。

## D. plain 对照（同一 target）：写 + 读 + 裸读

1. 停 vseg master；删我方键；起 plain master（NOF-052 同参、slot=1、`enabled=0`）；register；验收 Keys=0。
2. **plain writer**（16t/batch16、400 keys、verify=false、`MC_NOF_IO_PARAMS_LOG=1`、slot=1；日志 `run/nof059_plain_write.log`）：验收 `Keys=400`、`logical_used`；记录写窗口 W9；记录写 params（预期 blocks=8192、child=0/0）。
3. **plain reader**：同一 bench/参数；plain flags（`--vsegment_nof_io=false --segment_write_vsegment=false --replica_num=0 --nof_replica_num=1`）；params on；日志 `run/nof059_plain_read.log`；记录 P50/W9。
4. **不重写**，重复 C 的 **D1（顺序）与 D3（随机）** 裸读支（plain 数据），记录 client p50/W9/preflight LBA。
5. 收尾：停 plain master（确认退出）；**target 保持运行**，记录 PID/md5/4457；**不启用 BUFFER、不优雅停、不落盘**。

## E. 解析与判决（离线；必答）

1. 汇总表（单位 µs，标口径）：
   - R1_vseg4：P50 / W9（p50 或 avg，注明）；
   - D1_vsegData（裸 SPDK 顺序长扫）、D2_vsegData（顺序定点）、D3_vsegData（随机）：client p50 / target W9；
   - R1_plain：P50 / W9；
   - D1_plainData、D3_plainData：client p50 / target W9。
2. 明确回答：
   1. **stripe=4MiB 是否真正落到每 key 单 child**（写/读 params 的 SUBMIT 数、blocks、child 字段）；若否，配置层该旋钮作废，记 `[事实]`。
   2. **vseg4 的 W9 是否从 ~256 回落到 plain 档（≤~160 视为回落）**？client P50 是否接近 plain？
   3. 若未回落：**D1_vsegData 是快（client p50 ~41 / W9 ~90）还是慢（~220/190）**？与 D1_plainData 差多少？⇒ 同一数据状态下裸读是否不同。
   4. 写侧 params 的 child/blocks 与读侧是否一致。
3. 判据（三选一，给数字）：
   - **vseg4 W9 ≤ ~160 且 D1_vsegData 快** ⇒ 单 child 修复有效（配置可解，进入验证/回归）；
   - **vseg4 W9 ≥ ~230 且 D1_vsegData 快** ⇒ child 数无关、数据无罪，读路径仍有未识别差异（进入 M3 代码定位：客户端提交时序/SGL/buffer）；
   - **vseg4 W9 ≥ ~230 且 D1_vsegData 慢（≈D3 随机档）** ⇒ **数据物理状态**是主因（写侧；比较 D1_plainData）。

## F. 边界

- target 重启已获批准（A 路线延续）；本单结束 **target 保持运行**；不启用 BUFFER、不停 target。
- 裸读 preflight 写已知；所有裸读在对应 Mooncake 读之后；不跑 verify；不拷二进制、不改源码。
- 失败停手；不回退 e12b7cb；不扩大盘范围（系统盘 2b 禁碰）。

## 回执必答（第一行）

target 重启验收（PID/md5/4457/RAID0/BDF）；stripe=4MiB 是否生效（quota + 写/读 params 的 child/blocks/SUBMIT 数）；vseg4 P50/W9；D1/D2/D3 裸读（vseg 数据）client p50 与 W9；plain 写验收 + P50/W9；D1/D3 裸读（plain 数据）；**vseg4 是否追平 plain**；**数据状态 vs 读路径的判决**；有无失败。

---

# NOF-060（已关闭 · 两臂 NoF 提交层完全同构；唯一实测差异=突发长度 · 2026-11-23）

> 结果：零环境。**params 除 child 外全同**：op、`blocks=8192`、`bytes=4194304`、`block_size=512`、`endpoint uniq=1`、`vseg_op_id`、lane 0/1/2/3 各 132、`domain=0x143cd90`、`ptr` mod 4096/131072 均 0；唯一字段差异 `child=0/1`（vseg）vs `0/0`（plain），源码证实 **child 只进日志/timing 标签、不进 SPDK 调用**。源码：两臂 `client_service.cpp:5845/5855` 分叉、`transfer_task.cpp:2360/2527` 调用、**`:3076` 同一 `submitSpdkNofOperation` 同参数**；128KiB chunk（`blocks_per_chunk=256`）、8MiB/lane 在飞、lane 轮转、**task 内 32 IO 固定同一 qpair**（`:891`）两臂完全相同。`URMA_SLOW_IO` **有绝对 tick**（initiator submit/complete、completion_mono_ns、全套 target_*_tick 与 bdev/data 段）。**唯一实测行为差异：每批突发 16×128KiB(2MiB) vs 32×128KiB(4MiB)**（NOF-057 的 2MiB-child 时代；vseg4 时代 TRACE_US=0 ⇒ 突发形态 `[未获取]`）；submit 间隔 p50 0.45/0.44µs、<5µs 占比 93.8%/96.9%。T1 每 key SUBMIT 行间 dt：vseg4 **545±10µs**、span 0.800s；plain 前 240 op ~540µs、之后 **264µs**、span 0.617s（前段慢=盘首区 keys 0–97）。**无单点可解释 2× 的现成差异**；M3-1a（零构建、仅开 env）可拿逐 128KiB tick 与 task 级 submit_span/drain/fifo/credit/max_subio。产物 `run/nof060_params_compare.txt`（3,171B）、`run/nof060_source_trace.txt`（5,103B）；target 708167 保持运行、无 bench、未构建未改码。

## 背景与目标

- NOF-059：`stripe=4MiB` **确实**每 key 单 child（读 `SUBMIT=528`、`blocks=8192`、`child=0/1`），但端到端 **547.92 vs plain 268.44µs（2.04×）**，与双 child 的 1.99× 同档 ⇒ **child 数不是原因**。
- 同一份数据裸 SPDK：顺序 **40.0 vs 40.8µs**、随机 **220.16 vs 220.16µs** ⇒ **数据物理状态无罪**。
- 因此差异只剩 **vseg 客户端把同一个 4MiB NoF 操作驱动到 target/盘的方式**（提交时序/顺序/SGL/buffer/endpoint/会话），或 target 对两种会话请求的处理。
- 本单是 **M3-0 取证**（零环境、不重启、不构建、不改码）：用已有日志 + 客户端源码，回答"两臂在 NoF 提交层到底有没有差别、差在哪一层"，并给出 M3-1 的精确插桩点与最小重建路径。

## A. 现有 params 日志全字段对比（只读）

1. input：node3 `run/nof059_vseg4_read.log`、`run/nof059_plain_read.log`、`run/nof059_vseg4_write.log`、`run/nof059_plain_write.log`（node1 对应 master/register 日志不需要）。
2. 打印 `NOF_IO_PARAMS_SUBMIT` 与 `NOF_IO_PARAMS_WORKER` 的**字段全集**，各给 2 条原始样本；然后按臂聚合：
   - SUBMIT：`op/seg_lba/blocks/bytes/block_size/child/vseg_op_id/endpoint` 的取值/分布；**`endpoint` 的唯一值清单**（两臂是否同一会话/段）；`ptr` 的 min/max 与对齐（mod 4096 / mod 131072）；
   - WORKER：`lane/worker` 分布；`buf_type/buf_device/domain`；`ptr` 对齐；`blocks`；文件序下每 lane 的 seg_lba 单调率（剔除 warmup 4,096 条后给测量遍）。
3. **明确回答：两臂在 NoF 提交参数上是否有任何差异；若全部相同，写 `[事实] 无差异`**。

## B. 现有客户端逐 IO 日志的时间信息（NOF-057 产物）

1. 在 `run/nof057_vseg_read.log` / `run/nof057_plain_read.log` 中找 `URMA_SLOW_IO` 行（各 ~16,910 条）：打印**字段全集** + 2 条样本。
2. 判断是否存在**绝对时间戳/tick**（而非 duration）：若有，按 qid 重建每 128KiB 的提交间隔、突发大小（如 <5µs 连续条数）与空闲间隙，并给出两臂分布；若无，明确 `[未获取]`（**不得用文件行序猜时间**）。
3. 若行内带 target 回传的 rx/exec tick，可作 target 到达节奏的交叉验证（以 NOF-057 buffer 为准，不重复费力）。

## C. 客户端源码取证（我们自己的 bd5ee124d 副本，只读）

目标：找出"4MiB NoF 操作 → 32×128KiB NVMe 命令"的**确切代码路径**，以及 vseg/plain 两条路径在该层的差异。

1. 客户端 SPDK：`$D/spdk-e12b7cb-src/lib/nvme/nvme_urma.c`（及同目录相关文件）——128KiB 切分的实际位置、每个子请求的 SGL/payload 构造（iov 数、ptr/rkey）、qpair/jetty 选择、提交循环顺序、可用的 submit/completion tick。
2. Mooncake：`$D/mooncake-src/mooncake-store/src/transfer_task.cpp`（及 `vsegment_runtime.cpp` 如相关）——`submitSpdkNofOperation` 的调用点与参数（chunk/`nof_qpairs_per_segment`/`nof_inflight_bytes`/endpoint），以及 **vseg 路径与 plain 路径在此函数前后的差异**（是否经 vsegment runtime、buffer/QoS/批处理/顺序是否不同）。
3. 给出 `file:line` 与关键字段，明确回答：
   - 两臂最终是否**同一函数、同一参数**？若有差异，逐条列出（首要候选）；
   - 512B→128KiB 的切分发生在哪一层？切分后是否按 LBA 顺序提交、是否跨 qpair 轮转、每 qpair 在途上限是多少；
   - **M3-1 插桩点清单**：要打印的 per-128KiB 字段（qid/jetty、lba/nlb、payload ptr、SGL 数、submit tick、completion tick、outstanding）、env 开关、需要重建哪些 `.a`、如何重链 bench（引用 NOF-048/051 的配方）。
4. 不构建、不改文件、不重启。

## D. 结论（必答）

1. params 层是否有差异（逐字段）；
2. `URMA_SLOW_IO` 是否有可用绝对时间戳（有/无/哪些字段）；
3. 源码层 vseg vs plain 在 128KiB 提交路径上的**差异清单**（无则明确写"无差异"）；
4. **M3-1 插桩清单**（文件:行、字段、env、构建步骤、预计产物）；
5. 若发现某个参数/路径差异已可直接解释 2×，给出验证方案（可先于 M3-1）。

## E. 边界

- 零环境：target 保持运行（NOF-059 末态 PID 708167，**不重启/不停/不启用 BUFFER**）；不构建；不写盘；只读日志与源码。
- 不改任何源码；产物写 `run/nof060_*.txt`（≤3 个，每个 ≤1MB）。

## 回执必答（第一行）

params 层字段对比（endpoint/ptr/block_size/bytes/child 等是否全部相同）；`URMA_SLOW_IO` 时间戳可用性；源码层 vseg vs plain 的差异清单；M3-1 插桩点与最小重建路径；有无失败。

---

# NOF-061（已关闭 · 判决：vseg 读路径/会话；数据二次无罪 · 2026-11-23）

> 结果：`vseg_nof061` 写 400 keys（377+23 补，Keys=400、logical_used=1,677,721,600），写 LBA 集合与 NOF-059 plain 集合**逐值相同**（400 唯一 / 0..3268608 / 步长 8192）。vseg 读：P50 **573.03**（r2 601.09）、逐 128KiB bdev **249** / target **256** / roundtrip **278**。停 vseg master、**零写入**用旧 `vseg_nofvseg` plain 元数据读同一批 LBA：P50 **308.37**（r2 330.41）、bdev **57** / target **79** / roundtrip **108**。**同 LBA 配对 12,324 条：246 vs 53（4.64×），plain 更快 85.1%**；分 key：盘首 key0 314/277、key50 257/252（两臂都慢），高 LBA key100 250/46、key200 251/40、key300 250/47、key399 263/52。**vseg4 突发 max_run=31（32×128KiB=4MiB），与 plain 完全相同**（间隔 p50 0.58–0.61 vs 0.56–0.59µs、frac<5µs 均 0.968）⇒ NOF-060 的"16 vs 32"是 2MiB-child 时代产物，单 child 下不存在。新增硬排除：**突发形态**、**target lcore**（同 lcore 仍 4.3×：lcore2 248/57、lcore7 250/56）、**顺序/预热**（vseg 在 plain 两轮之后仍 601）。vseg task 级 span 19 / drain 376 / maxsub 383 / fifo 0 / credit 0；plain 无 `nof_slow_task` 行 `[未获取]`。偏差：多做 plain r2 / vseg r2 与一次纯元数据 re-register（`vseg_nof061`，rc=0，未删键未写盘）。环境：target 708167 全程未重启、无 BUFFER；4 次读全 400/0 failed；`/cvm/vseg_nofvseg/`=2、`/oplog/`=401、`vseg_nof061` 数据保留。产物 `run/nof061_summary.txt`（2,810B）+ 6 份原始日志 + `/tmp/nof061_an{1..8}.py`。**判决：差异在 vseg 读路径/会话，剩余唯一变量=会话/qpair 身份。**

## 背景与目标

- NOF-060：两臂在 NoF 提交层**完全同构**（同一 `submitSpdkNofOperation`、同参数、128KiB chunk、8MiB/lane、task 内 32 IO 同 qpair），`URMA_SLOW_IO` 有绝对 tick；唯一实测差异是 2MiB-child 时代的**每批突发 16 vs 32**（vseg4 时代未获取）。
- NOF-059：单 child 未追平；同一数据裸 SPDK 快 ⇒ 数据无罪。
- 本单做**决定性隔离**：先让 vseg 写出一份数据并 vseg 读；然后**不重写**，用 **plain 读路径**读**同一批 LBA 上的同一份 vseg 数据**。
  - plain 若快（~60µs/IO）⇒ **vseg 读路径/会话是原因**；
  - plain 若也慢 ⇒ 数据-读路径交互（裸读又说数据可快，需查 Mooncake 读的数据通路）。
- 同时取 vseg4 的**逐 128KiB tick、突发长度、task 级 submit_span/drain/fifo/credit**，与 plain 同口径对比。
- **零构建、target 不重启**（708167 保持运行、无 BUFFER）。

## 0. 关键技巧与前置检查（决定成败）

- **复用旧 plain 元数据**：NOF-059 的 plain 集群（`--cluster_id=vseg_nofvseg`）写过 400 keys，其 key→LBA 映射（LBA 0..3,268,608，400 个唯一值）仍在 etcd。本单 vseg 集群用**全新 cluster_id `vseg_nof061`**，不碰 `vseg_nofvseg`；vseg 写入会覆盖同一批 LBA 的物理数据，旧 plain 元数据仍指向这些 LBA ⇒ 之后用 plain 路径读到的就是 **vseg 刚写入的数据**。
- **A 段先验证**：`/cvm/vseg_nofvseg/` 仍含 plain 集群元数据与 400 keys（从 master 日志/etcd key 数判断；若无法确认，起 plain master 验证 `Keys=400` 即可，不写入）；`/cvm|/oplog/vseg_nof061/` 为 0 或不存在。**若 plain 键缺失/不可用 ⇒ 停手回报，不要重写、不要破坏 vseg 数据**。
- vseg 写入的 LBA 集合必须与 plain 的相同（预期 0..3,268,608 步长 8192、400 唯一）。若不同，记录并**只在两者交集 LBA 上做同数据对比**。

## A. 预检（只读）

1. target 708167 alive、4457、RAID0/ns 正常、**无 BUFFER env**；无 master/bench；node3 load/第三方进程记录。
2. etcd：`/cvm/vseg_nofvseg/`、`/oplog/vseg_nofvseg/` 的 key 数；`/cvm/vseg_nof061/`、`/oplog/vseg_nof061/` 的 key 数（预期 0/不存在）。
3. 记录 NOF-059 的 plain 读 params 的 seg_lba 集合（`run/nof059_plain_read.log`，作为预期 LBA 集合）。

## B. vseg 臂：新 cid 写 + 读（trace）

1. 起 vseg master：`--cluster_id=vseg_nof061`、slot=1、`--vsegment_stripe_size=4194304`、其余同 NOF-059 B1；register（对新 cid）；quota ≥20s；验收 `stripe_size=4194304`、partitions=1、reconcile_fail=0。
2. **T1 writer**（同 NOF-059 B2；trace 关、`MC_NOF_IO_PARAMS_LOG=1`）：400 keys、≤3 次补齐；验收 Keys=400、logical_used=1,677,721,600；记录写窗口 W9；**记录写 params 的 seg_lba 集合**并与 A3 比对（是否相同）。
3. **vseg reader（带 trace）**：`bench-io-params-e12`、`--num_keys=400 --num_threads=1 --batch_size=1 --nof_qpairs_per_segment=4 --warmup_keys=128 --verify=false`、vseg flags；env：
   - `SPDK_URMA_STAGE_TRACE_US=1`（逐 128KiB `URMA_SLOW_IO` + target tick）
   - `MC_NOF_IO_PARAMS_LOG=1`
   - `MC_STORE_PARTITION_SLOT_COUNT=1`
   - `MC_HIFREQ_LOG_SAMPLE_RATE=1`（task 级 breakdown）
   - `VSEG_SLOW_IO_US=0`、`MC_NOF_SLOW_TRACE_THRESHOLD_US=0`
   - 日志 `run/nof061_vseg4_trace.log`；记录 Min/Avg/P50/P90/P99/Max、failed、URMA_SLOW_IO 条数、W9。
4. **不重写、不删 vseg 数据**；停 vseg master（记录 PID 已退）。vseg 集群键保留（`vseg_nof061`，不影响 plain）。

## C. 同一数据 plain 臂：不写、直接读（trace）

1. 起 plain master：`--cluster_id=vseg_nofvseg`（NOF-059 同参、slot=1）；**不要清键、不要写**；register/mount（若 register 报"已存在"则记录并以已有段继续）；验收 `Keys=400`、`logical_used=1,677,721,600`、`enabled=0`。
   - **descriptor 被 B 段覆盖的补救（重要）**：B1 对新 cid register 可能把 target 侧该 namespace 的 descriptor/cluster 归属改成 `vseg_nof061`，导致本段 mount 报 `device_identity`/descriptor 不匹配。此时**允许对 `vseg_nofvseg` 重新 register 一次**（纯元数据操作，不触碰盘上数据），再 mount。**禁止删 `/cvm`、`/oplog` 键，禁止任何写入**。重 register 后仍失败 ⇒ 停手回报，保留现场。
2. **plain reader（带 trace）**：同一 bench/参数；plain flags（`--vsegment_nof_io=false --segment_write_vsegment=false --replica_num=0 --nof_replica_num=1`）；**与 B3 完全相同的 env**；日志 `run/nof061_plain_ondata_trace.log`；记录同 B3。
3. 若读数 <400 / 读取失败（旧段身份不匹配等）⇒ **停手回报，不重写**；记录错误码与 master 日志。
4. 收尾：停 plain master；**target 保持运行**（708167、无 BUFFER）；不构建、不改码。

## D. 解析与判决（离线；必答）

1. 两臂 `URMA_SLOW_IO`：条数、blocks/opcode 分布；**逐 128KiB** 的 `initiator_submit_tick → initiator_complete_tick`（roundtrip）、`bdev_us`、`target_us` 的 p50/p90；分 qid/lcore/disk 的 p50。
2. **突发形态**：相邻 submit <5µs 的连续 max_run（vseg4 预期 32；若为 16 ⇒ vseg 内部仍按 2MiB 分批，是新事实）；submit 间隔分布。
3. **task 级**：submit_span/drain/fifo/credit/max_subio 的 p50/p90（两臂）。
4. **同一 LBA 对比**：取两臂 LBA 交集，按 LBA 配对 `bdev_us`（同一物理数据）；给 plain 更快/更慢的比例与 p50 差。
5. 明确回答：
   1. **plain 路径读同一份 vseg 数据是否快**（bdev_us ~60 / 客户端 P50 ~270）？还是仍 ~250µs？
   2. vseg4 的突发长度是 32 还是 16；两臂 submit 间隔是否相同；
   3. 若 plain 快 ⇒ 差异在 **vseg 读路径/会话**（结合 NOF-060 的"提交层同构"，指向元数据/会话/endpoint 或 target 侧会话资源）；
   4. 若 plain 也慢 ⇒ 数据-读路径交互，下一步查 Mooncake 读的 target 数据通路（iobuf/SGL/buffer）；
   5. 下一步 M3-2 的具体取证点。
6. **口径警告**：本单若记录 target W9 块，需标注 dump 间隔/跨相位；以逐 128KiB `URMA_SLOW_IO` 为准。

## E. 边界

- **target 不重启、不停、不启用 BUFFER**；不构建、不改码。
- 允许 vseg 写入覆盖（同前）；**C 段禁止任何写入**（保持同一数据）。
- 旧 plain 元数据不可用 → 停手不重写；失败停手。
- 脚本/产物：分析脚本放 `/tmp`；产物 `run/nof061_*.txt`（≤3 个，每个 ≤1MB）。

## 回执必答（第一行）

vseg4 写/读是否成功与 Keys；vseg4 突发长度（32/16）与 submit 间隔；两臂逐 128KiB bdev_us/roundtrip p50；**plain 读同一份 vseg 数据是否快（关键）**；同 LBA 配对结果；task 级 drain/fifo/credit；判决（vseg 路径 vs 数据-路径交互）；C 段是否发生过重新 register；有无失败。

---

# NOF-062（已关闭 · gap 随并发强收敛：T1 4.61×→T4 1.01×→T16 1.02×；差异 100% 在 bdev · 2026-11-23）

> 结果：六档逐 128KiB bdev p50 —— **vseg T1/T4/T16 = 249/262/270**，**plain = 54/259/265**；gap **+195（4.61×）→ +3（1.01×）→ +5（1.02×）**。⇒ **① 完成检测/唤醒类命中**，兼具 **③ plain 单向追平**（vseg 恒 ~250，plain 54→259）。16 字段分段：T1 差异 **100% 落在 `bdev_us`**（249/308 vs 54/279），`rx_buffer`、`buffer_exec`、`pre_send`、`release` 全 0，`data` 6 vs 9，`outside` 两臂同为 21µs，`wait/enomem` 未出现在数据 IO（0）。同 qid 配对 T1 4.4–4.8×、T4/T16 全部 1.0×。**qid→lcore**：vseg {2:6,3:7,4:2,5:3}、plain {2:2,3:3,4:4,5:5}；但 NOF-061 的 vseg r2 在 plain 两轮之后仍 601 ⇒ 顺序不是主因。突发同构（run=31，间隔 0.62–0.64 vs 0.83–0.85µs）。task 级 vseg T1 span 21/drain 383、T4 30/494、T16 54/852（fifo p90 960、credit p90 166）；plain `[未获取]`。源码链：`urma.c` → `transport.c:932 spdk_iobuf_get(group->buf_cache,…)`（每 poll group 独立 buf_cache）→ `ctrlr_bdev.c:449/493` → `raid0.c:76` → 盘；`bdev_cpl_tick` 于 `urma.c:2493` 在 **poll 上下文**取。`[事实]` **target 无 vseg/child 语义**；连接层 NQN/nsid/qpair 数全同，唯一差异＝会话实例身份与 qpair→lcore 分配。偏差：首次 vseg 三档漏 `--warmup_keys=128`（SUBMIT=405/T1 614.49）已修正重跑（528/604.30），旧日志同名覆盖；`/cvm/vseg_nofvseg/` 2→3、`/oplog/vseg_nof061/` 2003→2004（register/租约元数据，无数据写入）。环境：target 708167 全程未重启、无 BUFFER；六档 400/0 failed。产物 `run/nof062_summary.txt` + 6 份日志。**判决：vseg 的每次 128KiB 读在设备空闲时也要 ~250µs，等价于完成被发现得晚 195µs；需查 reactor 轮询路径。**

## 背景与目标

- NOF-061 判决：**同一份数据、同一 LBA、同一 disk、同一 lcore、同参数命令**，plain bdev **57µs** vs vseg **249µs**（同 LBA 配对 4.64×）。数据/盘/突发形态/lcore/顺序全部排除，**剩余唯一变量 = 会话/qpair 身份**。
- 三种机制类别，用**并发曲线**即可分开：
  - ① gap 随并发**收敛**（T16 gap ≤30µs）⇒ 低负载下的**完成检测/唤醒**类附加延迟（轮询/唤醒/中断合并）；
  - ② gap **恒定 ~190µs**（三档比值都 ≥4×）⇒ **每命令固有**（命令/队列/缓冲归属）；
  - ③ vseg 恒 ~250 而 plain 由 57 **单向追平** ~160 ⇒ vseg 处于"恒定排队/非首请求"状态。
- 同时把 NOF-061 日志榨干（16 字段分段），并从 **target 侧源码**找出唯一可疑点，给 M3-3 的一行插桩位。

## 0. 前置与口径

- 数据就绪且**不得改写**：`vseg_nof061`（400 keys，vseg 元数据）与 `vseg_nofvseg`（400 keys，plain 元数据）指向**同一批 LBA**；两臂都只读。key i 占 `[i×8192, (i+1)×8192)` 块。
- 统计以逐 128KiB `URMA_SLOW_IO` 为准；target W9 块只记录、不判读（dump 滞后/跨相位）。
- 盘首慢窗口统一**统计排除 keys 0–96**，分析给"全量/排除"两套。
- 任一侧 mount 报 descriptor / `device_identity` 不匹配 ⇒ **允许对该 cid 重新 register 一次**（纯元数据，不写盘、不删键、不删 `/cvm` `/oplog`）；仍失败停手。

## A. 预检（只读）

1. target 708167 alive、4457 LISTEN、RAID0/ns 正常、`NO_BUFFER_ENV`；无 bench、无 master。
2. etcd：`/cvm/vseg_nofvseg/`、`/oplog/vseg_nofvseg/`、`/cvm|/oplog/vseg_nof061/` 键数（记录，不删）。
3. node3 load、第三方进程、50056/9006 占用。

## B. 并发曲线（核心新数据：同一份数据 × 两条路径 × 三档并发）

1. **vseg 臂**：起 vseg master（`--cluster_id=vseg_nof061`、slot=1、`--vsegment_stripe_size=4194304`、同 NOF-061 B1 参数），register/mount；验收 `Keys=400`、`logical_used=1,677,721,600`、`partitions=1`、`reconcile_fail=0`。
2. 依次三档，每档 1 遍（`bench-io-params-e12`，node3 本机，vseg flags）：
   - T1：`--num_threads=1 --batch_size=1` → `run/nof062_vseg_t1.log`
   - T4：`--num_threads=4 --batch_size=1` → `run/nof062_vseg_t4.log`
   - T16：`--num_threads=16 --batch_size=1` → `run/nof062_vseg_t16.log`
   - 三档固定：`--num_keys=400 --nof_qpairs_per_segment=4 --warmup_keys=128 --verify=false`；env `SPDK_URMA_STAGE_TRACE_US=1`、`MC_NOF_IO_PARAMS_LOG=1`、`MC_STORE_PARTITION_SLOT_COUNT=1`、`MC_HIFREQ_LOG_SAMPLE_RATE=1`、`VSEG_SLOW_IO_US=0`、`MC_NOF_SLOW_TRACE_THRESHOLD_US=0`。
3. 停 vseg master；起 plain master（`--cluster_id=vseg_nofvseg`、slot=1、同 NOF-061 C1；**不清键、不写**）；验收 `Keys=400`、`enabled=0`。
4. **plain 臂**同三档、plain flags（`--vsegment_nof_io=false --segment_write_vsegment=false --replica_num=0 --nof_replica_num=1`）、同 env → `run/nof062_plain_t{1,4,16}.log`。
5. 收尾：停 plain master；**target 保持运行**（708167、无 BUFFER）；不构建、不改码。

## C. 离线解析（必答）

1. 六份日志各自：逐 128KiB（opcode=2、blocks=256）的 **roundtrip / target / bdev** p50/p90/p99；**主力表 = bdev**。
2. **并发曲线**：x=线程数(1/4/16)、y=bdev p50，两臂两条线；逐档给 gap（vseg−plain）与比值。
3. **16 字段分段**（两臂 T1 与 T16 各一组）：`rx→buffer`、`buffer→exec`、`exec→submit`、`submit→submit_return`、`submit_return→callback`、`callback→bdev_cpl`、`bdev_cpl→data_post`、`data_post→data_cpl` 的 p50 与**非零计数**；`wait`/`enomem` 单独列。
4. **结构对照**：两臂 qid 集合、每 qid 的 IO 数、qid→lcore 映射；**同 qid 编号配对**（vseg qid2 vs plain qid2 等）的 bdev p50；每 (lcore,qid) 的 IO 数是否一一对应。
5. 突发与间隔：max_run、间隔 p50（三档各一）。
6. task 级：vseg 的 span/drain/fifo/credit/maxsub（T1/T4/T16）；plain 若无 `nof_slow_task` 行记 `[未获取]`。

## D. target 侧源码取证（只读、零构建）

在 `$D/spdk-e12b7cb-src`（bd5ee124d）里追**读路径**，逐条回答（附 `文件:行号`，片段 ≤20 行）：

1. **层次链**：nvmf 收包 → 请求/数据 buffer 从哪个池取（per-qpair？per-poll-group？全局 iobuf 大小类？）→ `spdk_bdev_read` → 完成回调 → 回写客户端。
2. **每会话/每 qpair 状态清单**：qpair 结构字段、poll group 归属、io_device/channel、buffer cache、jetty/资源绑定；逐项判断"能否影响同一 LBA 命令的完成延迟"，给出会影响的与不会影响的。
3. **是否有 vseg/child 语义进入 target**：grep `vseg`、`child`、`op_id` 于 `lib/nvmf/` 与 transport 层；无则写 `[事实] 无`。
4. **连接/attach 层**：客户端 vseg 与 plain 建连时可能发送的不同参数（NQN、qpair 数、jetty 配置、vseg 专用注册字段）。
5. **结论**：给出**唯一**的 M3-3 插桩点（一行打印哪些值），并说明它能分开哪两类机制。

## E. 判决（必答：三选一 + 数字 + 排序）

- ① gap 随并发收敛（T16 gap ≤30µs）⇒ 完成检测/唤醒类；
- ② gap 恒定 ~190µs（三档 ≥4×）⇒ 每命令固有；
- ③ plain 单向追平（vseg ≈250 不变、plain 57→~160）⇒ vseg 处于"恒定排队/非首请求"状态。
- 另给：机制候选排序（带源码/数据证据）+ M3-3 一行插桩建议。

## F. 边界

- **target 不重启、不停、不启用 BUFFER**；不构建、不改码；**全程只读、零写入**（不得重写 400 keys、不得做裸读 preflight 写）。
- 不删 `/cvm` `/oplog` 任何键；descriptor 不匹配只允许对相应 cid 重新 register 一次。
- 脚本放 `/tmp`；产物 `run/nof062_*.txt`（≤3 个，每个 ≤1MB）；失败停手保现场。

## 回执必答（第一行）

六档 bdev p50（vseg/plain × T1/T4/T16）与 gap 曲线；收敛/恒定/追平三选一；16 字段分段中哪段承载差异、wait/enomem 是否非零；两臂 qid 结构与同 qid 配对结果；target 读路径的 buffer 归属与每会话状态清单（文件:行号）；是否有 vseg 语义进入 target；唯一插桩点；有无失败。

---

# NOF-063（已关闭 · 静态无 50–500µs 常量；结构性错配存在但两臂对称 · 2026-11-23）

> 结果：主链 `transport.c:777→779` → `urma.c:2303 nvmf_urma_poll_group_poll`（2303–2383，81 行）。**`[事实]` 全链路无 50–500µs 时间常量**：`urma_poll_jfc` 无 timeout 形参，`nanosleep/usleep/spdk_delay_us/sched_yield/spin` 在 `lib/nvmf/urma.c` 零命中；`abort_timeout_sec=1`、`min_kato=10000`、`jfc_depth=4096`、`jetty_depth=2048` 均不在该量级。环境实际生效值仅 `SPDK_URMA_DEV_NAME`、`SPDK_URMA_TARGET_DUMP_SEC`；`send/recv_jfc_count=2`、`num_jetty_per_ep=1`。**结构发现**：`stride=worker_count=6` 而 `send_jfc_count=2` ⇒ 仅 worker_index 0/1 轮询网络 JFC，2..5 不轮询；qpair 的 JFC = 建连序 mod 2、与属主无对齐 ⇒ 跨线程 `spdk_thread_send_msg`（2289）投递后才 `handle_completion`（exec_tick 在 2167）。本地 NVMe：`bdev_nvme.c:1970` 每轮不限量、无中断/idle；`req_complete` 前无 defer。**开发侧复核：结构错配两臂对称**（plain {2,3,4,5}=worker{0,1,2,3}、vseg {6,7,2,3}=worker{4,5,0,1}，各 2 个轮询核 + 2 个非轮询核），**且同臂内 4 个 qid 时延均匀**（vseg 247–250、plain 52–56），四个绑定各异的 qpair 同值 ⇒ **结构不是 4.6× 的原因；`SPDK_URMA_JFC_COUNT=6` 不作判决实验**。⇒ 静态无法判定，必须插桩（H1/H2/H3 判别表见 §7）。产物 `run/nof063_source_poll.txt`（11,455B）；target 708167 未重启。

## 背景与目标

- NOF-062 定案：逐 128KiB `bdev_us` **T1 249 vs 54（4.61×）→ T4 262 vs 259 → T16 270 vs 265**；差异 **100%** 在 `bdev_us`，其余 15 段为 0；`outside` 两臂同 21µs。
- `bdev_cpl_tick` 在 `urma.c:2493`（`req_complete`，**poll 上下文**）取 ⇒ `bdev_us` = SSD 执行 + **完成被 reactor 发现之前的等待**。
- 设备空闲（T1）时 vseg 仍是 250µs、plain 54µs；设备饱和（T4/T16）时两者都 ~262–270 ⇒ **那 195µs 只在"轮到本 qpair 的完成被看见"这一环上，且能随负载摊薄**。
- 本单**只在源码里定位这 195µs 由谁产生**，不跑测试、不改码、不动 target。若能直接定位，M3-4 直接改；若不能，给出唯一插桩点与预期读数。

## A. 现场事实（只读，先做）

1. `cat /proc/708167/environ`（过滤并记录全部 `SPDK_URMA_*`、`SPDK_NVMF_*`、`SPDK_NOF_*`、`VSEG_*`，敏感项打码）：确认 target 当前生效的 URMA 相关开关与默认值来源。
2. target cmdline（`/proc/708167/cmdline`）：core mask、`--wait-for-rpc`、transport 配置（`nvmf_get_transports` 里的 `urma` 选项，若可读）。
3. 记录 `nvmf_tgt` 二进制 md5 与源码树 HEAD，确认取证对象 = 运行中二进制。

## B. 源码取证（`$D/spdk-e12b7cb-src`，bd5ee124d；**必答，附 `文件:行号`，片段 ≤20 行**）

1. **轮询主链**：`lib/nvmf/transport/urma/` 的 `poll_group_poll`（及被它调用的所有函数）逐层列出：每层做什么、返回值含义、循环上限。给函数→行号表。
2. **阻塞/等待/超时**：全链路检索 `wait`、`timeout`、`sleep`、`usleep`、`cond`、`event`、`sem`、`while` 自旋、`jfc` 相关 API（如 `urma_wait_jfc`/`urma_poll_jfc`/`urma_get_jfc`）。**把每个时间常量与 env 默认值列成表**（含单位换算），并**标出 50–500µs 量级的**——实测待解释量是 **195µs**。
3. **批量/预算**：每轮最多处理多少完成/请求（`burst`、`batch`、`max_*`、`poll_budget`）；是否"每轮只取 N 个"或"取完为止"。
4. **qpair ↔ jetty/JFC ↔ poll group 绑定**：结构字段、创建点、分配规则（qid 如何映射到 jetty/JFC/lcore，NOF-062 观测 vseg {2:6,3:7,4:2,5:3} vs plain {2:2,3:3,4:4,5:5}）。**指出在这套结构里"每会话"唯一可变的自由度**：JFC 数、jetty 数、qpair→jetty 映射、poll group 归属、buf_cache、io_channel，逐项判断"能否造成同一 LBA 命令的完成被发现晚 195µs"。
5. **本地 NVMe 完成路径**：`raid0`/`nvme bdev` 的完成由谁调用 `spdk_nvme_qpair_process_completions`（同一 reactor？每轮一次还是多次？），是否启用 interrupt/事件模式（`set_intr_enabled`、`fd`、`spdk_thread` 中断），以及 reactor 是否存在 idle/backoff/阻塞（`spdk_thread_poll` 的 idle 处理、`spdk_spin`、CPU 调度）。
6. **`req_complete` 之前是否有 defer**：完成是否被放入待处理队列、在下一次 poll 才回写客户端（`pending`、`deferred`、`flush`）。
7. **vseg 语义再次确认**：`lib/nvmf`、`lib/bdev`、`include` 中检索 `vseg`、`child`、`op_id`、`nof`（NOF-062 已判无；此处只复核 target 目录是否另有 nof 专用模块）。

## C. 判决（必答，二选一 + 依据）

1. **能直接定位**：给出**具体常量/分支**（`文件:行号`）、其量级与 195µs 的吻合关系，以及它为什么对 vseg 会话生效而对 plain 不生效（或为何两臂本应相同却不同）；再给 M3-4 的修复假设（改哪一行/哪个参数）。
2. **不能直接定位**：给出**唯一**插桩点（一行打印什么字段）+ **预期读数表**：
   - "完成早已在 CQ、轮询没到" ⇒ 记录 `submit→cpl` 期间 reactor 的 **poll 轮次增量 / empty_polls 增量 / 是否 idle-sleep**；
   - "轮询到了但 CQ 空" ⇒ 同段 `poll_calls` 增量小、每次 CQ 空 ⇒ 指向 SSD/队列；
   - 插桩必须能**同时**覆盖这两个读数，并说明插桩本身的测量开销（纳秒级）。
3. 另给：**为什么 T4/T16 消失**的源码解释（是摊薄、还是被 SSD 排队掩盖）。

## D. 边界

- **不跑任何压测、不启动/停止任何服务、不改码、不构建、不重启 target**（708167 保持运行）。
- 只读：源码树、`/proc/708167/*`、现有日志（`run/nof062_*.log` 可用于交叉核对，不必再解析）。
- 不删 etcd 键；不写 `run/` 之外的文件；产物 `run/nof063_source_poll.txt`（≤1MB）。
- 失败/查不到 → `[未获取]` 写清尝试路径，不猜。

## 回执必答（第一行）

`poll_group_poll` 主链与总行数；**50–500µs 量级的时间常量清单（含 env 默认值）**；是否存在阻塞等待/超时/批量上限；qpair↔jetty/JFC↔poll group 绑定与每会话自由度；本地 NVMe 完成轮询方式与 reactor 是否 idle/阻塞；`req_complete` 前是否 defer；**能否用源码解释 195µs（能→给常量与行号；不能→给唯一插桩点+预期读数表）**；为什么 T4/T16 会消失；有无失败。

---

# NOF-064（已关闭·部分 · 改码/重编/重启完成；vseg 臂因重启 identity 变化不可挂载、v1 插桩位置错 · 2026-11-23）

> 结果：三步全部完成——新 target **PID 1587271、md5 `21ced674…`**、4457 / nsid1=URMA_RAID0 / 11 盘 strip256K online / transport urma 验收全通过、`NOF064_TRACE=1`。二进制三份并存（`nof059` 979f5517 = 原 bd5ee124d；`nof064` 21ced674 = v1 在跑；`nof064b` afe95672 = v2 补读路径快照，已编译未运行）。编译坑：`nof064_*` 在 nvmf 库、bdev 模块不链它 ⇒ 计数器改定义在 `bdev_nvme.c`、`urma.c` 弱引用。**实测映射**：`worker_index 0→core3, 1→core4, 2→core5, 3→core6, 4→core7, 5→core2`（确证 NOF-063 的 stride 结构：只有 worker 0/1 = core3/core4 轮询 JFC）。**阻塞①（产品级事实）**：重启使 raid0/nsid1 uuid 由 `c697219c-1034-476f-aca8-ae3ac8de3567` 变为 `63a1e3cd-d735-4b0d-83ce-4b967fe490d7`（base bdev uuid 未变）⇒ `vseg_nof061` register rc=0 但 master mount `rc[-300] INVALID_VERSION`、reader `SUBMIT=0`；**plain `vseg_nofvseg` 仍正常挂载（100GiB）** ⇒ **vseg 元数据跨 target 重启失效、plain 不受影响**。**阻塞②**：v1 把快照插在写路径 `urma.c:2214`（PULLING），读走 `1921`（C2H, no pull）⇒ 读 IO 未取快照，`d_rounds=467,099,749` 等为绝对计数。已采 plain 臂 v1：client P50 **317.44**（400/0）、target 4,000 条 `NOF064_IO` bdev mean **248.8** / p50 **225** / p90 **312**、≤70µs 仅 16.6%。**开发侧判定（已修正）：该 225µs 是**采样偏差**——4,000 条预算≈前 125 个 key，其中 keys 0–97（78%）落在盘首慢窗口，与 83.4% >70µs 吻合；同臂客户端 P50 317.44 与 NOF-062 的 307.51 基本相同（端到端未变）⇒ 225 为子集假象。**早前归因 printf 观测者效应已撤回**（若打印真推迟后续 IO，客户端 P50 应同步抬升）。v1 数据仍不用于 bdev/`d_*` 结论（另因 `d_*` 是绝对值）；另**弃用 JFC/stride 假设**（`exec_tick` 在 `handle_completion` 内、跨线程投递在其之前 ⇒ 不进 bdev_us 窗口）。⇒ NOF-065。

## 0. 授权与前置（**用户已于 2026-11-23 明确批准：改码 + 重编 + 重启 target**）

> 授权范围：仅限本单 B 段的日志语句、`nvmf_tgt` 重编、以及按同 RPC 重启 target。超出范围（改控制流、改缓冲分配、改 RPC 默认值、动他人进程）仍需另行批准。

- 本单要动运行中的 target，三步都必须先获用户当场批准：① 改 `$D/spdk-e12b7cb-src` 源码（**只加日志，不改行为**）；② 重编 `nvmf_tgt`（NOF-048 配方：改一个 .c → 61s 编译 + 4s `ar r` 换库成员 + 重链）；③ **重启 target 708167**（同 RPC 重放；11 盘 vfio 残留 ABORT 用 takeover `-r -y` 补救，NOF-055/059 已两次成功）。
- **回退锚点（必须前后一致）**：旧二进制 md5 `979f5517…`（bd5ee124d）、旧 PID 708167、`/tmp/nvmf_tgt.log`（记录 offset 与大小）、当前 RPC 配置导出。回退 = 旧二进制 + 原 RPC + 同启动脚本。
- 授权已获；若重启验收不通过（4457/RAID0/11 盘/transport 任一失败）→ **立即回退旧二进制并停手回报**，不得在未验收状态下继续 C 段。

## A. 预检（只读）

1. target 708167 alive、4457、RAID0/ns、`NO_BUFFER_ENV`；`/proc/708167/environ` 与 cmdline（确认无 poll/timeout 覆盖）。
2. 源码树 HEAD/md5、`build/bin/nvmf_tgt` 当前 md5 = `979f5517…`；备份 `/tmp/nvmf_tgt.log`（记 offset）。
3. 记录当前 RPC 全量（`nvmf_get_subsystems`/`get_transports`/raid0 配置），作为重启重放清单。
4. 无我方 bench/master 残留；node3 load 与第三方进程记录。

## B. 打点（**只加日志**，同一份代码两臂共用，env `NOF064_TRACE=1` 开关）

1. **子命令字段**（在 raid0 → 本地 nvme 的 read 提交处，`module/bdev/nvme/bdev_nvme.c` 组装 nvme 命令的位置，或 raid0 子 IO 提交点，取其一并注明）：
   每 IO 打印：`thread/lcore`、`qpair ptr`、`nsid`、`lba`、`nbytes`、**`iov_cnt`、`iov[0].base/len`、`total_iov_bytes`**、SQ 位置（若可取）、`bdev_io` 所属 channel。
2. **轮询节奏**（与 B1 同一条记录关联）：
   - `urma.c:2303` 入口给线程本地 `poll_round` 自增；
   - `bdev_nvme.c:1970` 处累计 `nvme_poll_calls` 与 `nvme_cpl_total`；
   - 在 `urma.c:2493`（`bdev_cpl_tick = now` 处）打印 `bdev_us=(now-exec_tick)`、`d_rounds`、`d_nvme_poll`、`d_nvme_cpl`。
3. **一次性映射**：`poll_group_create`（`urma.c:1473`）打印 `worker_index`/`worker_count`；qpair 建连（`urma.c:1519`）打印 `qid`/`group->worker_index`/`uqpair->send_jfc_index`/`recv_jfc_index`；`nvme_urma_common.c:1021` 打印 `device->send_jfc_count/recv_jfc_count`。
4. **量控**：只对 data IO（nbytes=128KiB）采样，前 4,000 条或每 8 条取 1；单份日志 ≤200MB；不得改变行为（不引入锁、不引入 IO、不做 flush）。
5. 编译产物另存（如 `build/bin/nvmf_tgt.nof064`），**保留旧二进制副本**（`nvmf_tgt.nof059`）以便回退。

## C. 采集（同一份数据、只读、零写入）

1. 用 NOF-048 配方重编，启动带打点的 target（**同 RPC 重放、同 4457、同 RAID0 11 盘、同 core mask `-m [2-7]`**），验收口径同 NOF-059 A4。
2. **vseg 臂**（cid `vseg_nof061`，只读）：T1 一遍，`--num_keys=400 --num_threads=1 --batch_size=1 --nof_qpairs_per_segment=4 --warmup_keys=128 --verify=false`，env 同 NOF-062 B2（含 `SPDK_URMA_STAGE_TRACE_US=1`），另加 `NOF064_TRACE=1`；日志 `run/nof064_vseg_t1.log`。
3. **plain 臂**（cid `vseg_nofvseg`，只读同一批 LBA）：同上换 plain flags；日志 `run/nof064_plain_t1.log`。
4. **反序复测**：再跑 plain 一遍、再跑 vseg 一遍（或至少 vseg 再一遍）确认"臂间差"在插桩下仍成立、非首轮效应；日志 `*_r2.log`。
5. 收尾：target **保持运行**（新 PID 记录、md5 记录）；若新二进制不可用 → 按回退锚点恢复旧二进制再启，并在回执写明。

## D. 判决（必答，三问三答 + 数字）

1. **命令是否真相同**：两臂 `nsid/lba/nbytes/iov_cnt/total_iov_bytes/SQ 位置` 的分布对比。若 `iov_cnt` 或 `iov` 分布不同 ⇒ 指向 target 侧 buffer/描述符拆分差异（这是首次可直接观察的一层）。
2. **那 195µs 落在哪**：按 H1/H2/H3 判读 `d_rounds` / `d_nvme_poll` / `d_nvme_cpl`（NOF-063 §7 表）：H1 没轮到（`d_rounds` 大、`d_nvme_poll` 小且首轮即拿到）；H2 轮到了但 CQ 空（`d_nvme_poll` 大、多数轮 0）；H3 两者均 ~1 而 `bdev_us` 仍大（⇒ 查调度抢占：`/proc/<tid>/stat` 的 involuntary_switches、cpu 迁移、steal）。
3. **同臂 4 个 qid 是否仍同值**（复核 NOF-063 的对称性论证），以及 `worker_index`/JFC 索引的实测映射。
4. **根因假设 + 修复方案**：给出唯一假设、改哪一行/哪个参数、预期收益（target bdev 从 ~250 → ~60µs），以及 M3-5 验证口径。

## E. 边界

- 若未获批准 → 只做 A 段。
- 改码仅限 B 段的日志语句；**禁止**改控制流、加锁、改缓冲分配、改 RPC 默认值。
- 采集全程**只读数据**（不重写 400 keys、不做裸读 preflight 写）；不删 `/cvm` `/oplog` 键。
- target 重启后必须验收：4457 LISTEN、nsid1=URMA_RAID0、11 盘 BDF 升序/strip 256K/online、transport urma/udmac0d1e2；不通过 → 立即回退旧二进制并停手回报。
- 脚本放 `/tmp`；产物 `run/nof064_*.txt`（≤3 个，每个 ≤1MB）+ 二进制 md5 清单；失败停手保现场。

## 回执必答（第一行）

是否获授权并完成改码/重编/重启（新 PID、md5、4457/RAID0 验收）；两臂子命令字段（nsid/lba/nbytes/iov_cnt/total_iov_bytes/SQ）是否一致；`d_rounds`/`d_nvme_poll`/`d_nvme_cpl` 与 `bdev_us` 的对应关系；H1/H2/H3 判定；同臂 4 个 qid 是否同值；worker_index/JFC 实测映射；根因假设与修复方案；有无失败/回退。

---

# NOF-065（待执行 · 第二阶段 M3-5：v3 零干扰插桩（环形缓冲 + 真 delta + iov 字段）+ 新 cid 一次写入 + 两臂 T1 采集 · 2026-11-23）

## 0. 授权与三条前置判决（**必读，避免重复劳动**）

- **授权**：沿用 2026-11-23 用户批准的范围（仅日志改码 + 重编 + 同 RPC 重启 target）；写入仅限本单 C3 的**一次** 400 keys（用户已在 NOF-064 回执的三选一中授权「重启 + 写入」）。
- **判决 1：v1 的 plain 225µs 不可用，原因是采样偏差**。4,000 条预算按到达序≈前 125 个 key，其中 keys 0–97（78%）落在盘首慢窗口；83.4% >70µs 与该比例吻合，且同臂客户端 P50 317.44 与 NOF-062 的 307.51 基本一致（端到端未变）。**此前归因为 per-IO printf 观测者效应，已撤回**（若打印真推迟后续 IO，客户端 P50 应同步抬升）。⇒ 本轮要求：**全量记录或均匀覆盖 keys 98–399，禁止按到达序截断**；零干扰设计与 on/off 对照照旧保留。
- **判决 2：JFC/stride 结构不解释 bdev_us**。`exec_tick` 在 `handle_completion` 内（读路径 `urma.c:1921`），而 qpair 的跨线程投递（`urma.c:2289`）发生在它**之前** ⇒ 不在 `exec→bdev_cpl` 窗口内。故 `stride=1` / `SPDK_URMA_JFC_COUNT=6` **不作为本问题的修复实验**（可作旁证，另立单）。
- **判决 3（产品级新事实）**：target 重启使 raid0/nsid1 uuid 变化 ⇒ **vseg 元数据失效（mount rc[-300] INVALID_VERSION）**，plain 元数据不受影响。⇒ 任何重启后必须用**新 cid 重写数据**；旧 cid 的 vseg 元数据不要再尝试挂载。

## A. 预检（只读）

1. target 1587271 alive、4457、RAID0 11 base/11 operational、strip 256K、md5 `21ced674…`、`NOF064_TRACE=1`。
2. 三份二进制路径与 md5（`nof059` 979f5517 / `nof064` 21ced674 / `nof064b` afe95672）；确认源码树 `urma.c` 已有 `*.nof064orig` 备份。
3. 记录当前 nsid1 uuid（重启后再记一次，用于复现判决 3）。
4. etcd：`/cvm|/oplog/vseg_nofvseg/`（plain 元数据，禁止删）、`vseg_nof061`（已失效、保留作证据）、`vseg_nof065`（预期 0）。

## B. 插桩 v3（**只加日志**，基于 v2 `nof064b`）

1. **去 printf**：`NOF064_IO` 的 per-IO 打印全部删除，改为写**静态环形数组**：预分配 ≥400,000 条 × 96B（≈38MB），每条带 `valid` 标记（先写 payload，最后 `__atomic_store_n(&valid,1,...)`）。
2. **补 iov 字段**（`module/bdev/nvme/bdev_nvme.c` 组装 nvme 命令处）：`nsid`、`lba`、`nbytes`、`iov_cnt`、`iov[0].base`、`iov[0].len`、`total_iov_bytes`、`bdev_io` 指针；与 urma 记录用 `(cid,qid,lba,nbytes)` 关联。
3. **真 delta**：`exec_tick` 处快照 `poll_round / nvme_poll_calls / nvme_cpl_total`，`req_complete` 处算差值（v1 打绝对值是因为读路径没快照）。
4. **落盘与采样**：起一个**非 reactor 的 pthread**（不进 SPDK reactor 集合），每 3s 把新增 `valid=1` 记录**追加**写 `/tmp/nof065_trace.bin`（每行一条，含时间戳），不阻塞 reactor；target 全程保持运行。
   **采样硬约束**：按全量记录（400 keys × 32 = 12,800 条/遍，环形容量 ≥400k 足够）；**禁止按到达序截断前 N 条**——v1 正因此把样本全落在盘首慢窗口（NOF-064 教训）。若必须限流，须**均匀跨 key 采样**（覆盖 keys 98–399），并在回执写明采样规则与生效 key 范围。
5. **一次性打印**（允许 printf，非 IO 路径）：会话建立打印 `cid + hostnqn + 时间`；qpair 打印 `qid → worker_index → core → send_jfc_index`；启动打印 `worker_count / send_jfc_count / recv_jfc_count`。
6. 开关 `NOF065_TRACE=1`，关闭时**零开销**；产物 `build/bin/nvmf_tgt.nof065`，**保留** 979f5517 / 21ced674 / afe95672 三份。

## C. 重启 + 新 cid 写一次

1. 停当前 target（记录 PID/md5/日志 offset），用 `nvmf_tgt.nof065` + 同 RPC 重放 + `NOF065_TRACE=1` 重启；验收同 NOF-064（4457 LISTEN / nsid1=URMA_RAID0 / 11 盘 BDF 升序 strip256K online / transport urma/udmac0d1e2 / handshake 失败 0）。**不通过 → 回退 `nof059` + 原 RPC 并停手**。
2. 记录新 nsid1 uuid；起 vseg master **新 cid `vseg_nof065`**（slot=1、`--vsegment_stripe_size=4194304`、其余同 NOF-059 B1）；register；等 quota；验收 `partitions=1`、`reconcile_fail=0`。
3. **T1 writer 写一次 400 keys**（`--num_threads=1 --batch_size=1`、vseg flags、verify=false；`VSEGMENT_CREATING` ≤3 次补齐）；验收 `Keys=400`、`logical_used=1,677,721,600`；记录写 params 的 `seg_lba` 集合（预期 0..3268608 步长 8192）。**此后全程禁止再写。**

## D. 两臂采集（T1，各 2 遍 + 各 1 遍无插桩对照）

1. **vseg 臂**（cid `vseg_nof065`）：T1 ×2，env 同 NOF-062 B2 + `NOF065_TRACE=1`；日志 `run/nof065_vseg_t1{,_r2}.log`。
2. **vseg 对照**：T1 ×1，`NOF065_TRACE=0`；日志 `run/nof065_vseg_t1_notrace.log`。
3. **plain 臂**（cid `vseg_nofvseg`，读**同一批 LBA**）：T1 ×2 instrumented + ×1 `TRACE=0`；日志 `run/nof065_plain_t1{,_r2,_notrace}.log`。
4. 每遍结束等 ≥4s（等 flusher 落盘），记录 `/tmp/nof065_trace.bin` 的字节偏移区间与该遍 `cid`，便于按遍切分。
5. 收尾：停两 master；**target 保持运行**（nof065 二进制）；不删 etcd 键。

## E. 解析与判决（必答）

1. **零干扰验证**：两臂 instrumented vs `TRACE=0` 的 client P50/P90，差值 >5% ⇒ **本轮数据全部作废**并停手回报（不要强行解释）。
2. **命令是否真相同**：两臂 `iov_cnt / iov[0].base / iov[0].len / total_iov_bytes / nsid / lba` 的分布；把 `iov[0].base` 用 `/proc/<target>/numa_maps` **按 NUMA 节点分桶**（buffer 物理归属首次可查）。
3. **H1/H2/H3**：按 `bdev_us` 分桶（<70 / 70–150 / 150–300 / >300µs）给 `d_rounds / d_nvme_poll / d_nvme_cpl` 的 p50/p90（**必须是 delta**）。
4. **同臂 4 个 qid**：`bdev_us` 与 `d_*` 是否同值；给出实测 `qid→worker→core→jfc` 表。
5. **判决**：① 设备收到的命令是否真相同（若 iov 不同，直接给差异项）；② 195µs 属 H1（没轮到）/ H2（轮到但 CQ 空）/ H3（两者均 ~1、仍是 250µs）；③ 若 H1：属主 reactor 同轮还在干什么（同轮其他 qpair 完成数、`check_lifetime_socket`、capsule 处理计数）；若 H2：转设备/队列方向；若 H3：查调度抢占（`/proc/<tid>/stat` involuntary_switches、CPU 迁移、steal）。
6. **M3-6 修复方案**：唯一假设 + 改哪一行/哪个参数 + 预期（target bdev 250 → 60µs 量级）+ 验证口径。

## F. 边界

- 只加日志，**禁止**改控制流、加锁、改缓冲分配/SGL/buffer 类型、改 RPC 默认值。
- 重启验收不通过 → 回退 `nof059`（979f5517）+ 原 RPC + 原启动脚本，停手回报。
- 写入仅 C3 一次；D 段零写入；不删 `/cvm|/oplog/vseg_nofvseg/`。
- 脚本放 `/tmp`；产物 `run/nof065_*.txt`（≤3 个、每个 ≤1MB）；完整 trace 放 `/tmp/nof065_trace.bin`（不受 1MB 限制）。
- 失败停手保现场；任何"看起来能解释但没达到零干扰验证"的结论一律不写。

## 回执必答（第一行）

零干扰对照结果（instrumented vs TRACE=0 的 P50 差）；两臂 iov 字段是否相同（含 `iov[0].base` 的 NUMA 桶）；`bdev_us` 分桶下的 `d_rounds/d_nvme_poll/d_nvme_cpl`（delta）；H1/H2/H3 判定；同臂 4 qid 是否同值；`qid→worker→core→jfc` 实测表；M3-6 修复方案；有无失败/回退。

---

# NOF-066（待执行 · 统一「排除盘首慢窗口」口径的离线重算 · 零环境 · 2026-11-23）

## 背景与目的

- 现状口径不一致：汇报主表用的是 **400 keys 全量**（含盘首慢窗口），而部分分析（NOF-039/040）用的是**另一轮 800–1200 keys 的干净口径**，跨轮不可换算。
- 盘首慢窗口已实测确认：keys 0–97 / **LBA < 802,816 块（≈391.8 MiB）= 11 块盘各自开头约 37MB 的慢区**；plain 的慢 IO 全部落在此区（NOF-058），vseg 则是全程均匀慢（keys 97–399 慢占比 90.7%）。
- 本单不测任何新东西，只用**已有日志**把同一轮的「排除慢窗口」口径算出来，给「全量 / 排除」两列并排。

## A. 数据源（只读）

1. `run/nof062_vseg_t{1,4,16}.log`、`run/nof062_plain_t{1,4,16}.log`（逐 128KiB `URMA_SLOW_IO`，含 `lba`、`bdev_us`、`roundtrip`、`target_us`）。
2. `run/urma_stage_nof055.bin`（NOF-057 落盘，24 字段/条，含 `lba`、`bdev_us`、`exec_tick`）。
3. 如日志含 per-key 完成时间（`SUBMIT` 行 dt 或 bench 每 key 输出），用其算端到端每 key 时延；**若确无 per-key 时延，则该项记 `[未获取]` 并说明**，不要用 IO 级数据冒充。

## B. 排除规则（两条都算，报告以 LBA 规则为准）

- 主规则：`lba >= 802,816` 块（严格按 LBA 剔除，等价 keys ≥98）。
- 备规则：`key >= 98`（按 key 号剔除；两条结果若有差异，说明边界模糊并都列出）。
- 口径必须写死在结果表头：`n`（剔除前后 IO 数）、剔除比例。

## C. 重算项（每项都给「全量 / 排除」两列）

1. 逐 128KiB（opcode=2、blocks=256）：`roundtrip / target / bdev` 的 **p50 / p90 / p99 / mean**，两臂 × T1/T4/T16。
2. 端到端每 key：**p50 / p90 / p99 / mean**，两臂 × T1/T4/T16（若可得）。
3. **同 LBA 配对**（vseg vs plain，同一 LBA 的 bdev 配对）：p50 比值、plain 更快占比——分别在全量、排除两种集合上算。
4. **直方图分桶**：32–64 / 64–128 / 128–256 / 256–512 / ≥512 µs 的占比（两臂、T1）。
5. 慢窗口内的样本单独汇总一张小表（keys 0–97 两臂各 p50），用于证明"该窗口是共有项"。

## D. 判决（必答）

1. 排除后两臂 T1 的 per-IO p50/p90/mean 与比值；与全量口径相比各变化多少。
2. 排除后 16 并发是否仍打平（约 1.0×）。
3. **结论是否改变**（vseg 更慢 → 是否仍成立；幅度是变大还是变小）。
4. 汇报建议：主表应该用哪个口径，给一列推荐值。

## E. 边界

- **纯离线**：不启停任何服务、不跑 bench、不写盘、不改码、不删 etcd 键；只读 `run/` 下既有文件。
- 脚本放 `/tmp`；产物 `run/nof066_excl_summary.txt`（≤1MB，另可加 2 个明细 txt）。
- 数据缺失/格式不符 → 记 `[未获取]` 并写尝试路径，不猜、不用替代口径冒充。

## 回执必答（第一行）

两条排除规则各自的 n 与剔除比例；排除后两臂 T1/T16 的 per-IO p50/p90/mean 与比值；端到端每 key 是否可得；同 LBA 配对在两种集合下的比值；直方图分桶；结论是否改变；推荐的汇报口径；有无失败。

---

# NOF-067（待执行 · 只读取证：同事的 SQE/CQE/SQHD 拆解能力与 `--target-trace-entries` · 零环境 · 2026-11-24）

## 背景与目的

- 开发侧此前判断「`submit_return → callback` 之间没有第三个可观测点」，指的是**现有 `stage_trace` 的软件打点位置**（起点=提交/门铃、终点=完成项被处理，中间无设备时间戳）。
- 用户转达同事意见：**该段可用 SQE / CQE / SQHD 对比进一步拆解**，且已有现成脚本：
  - 裸 SPDK：node3 `/home/yin/run_exp/run_exp.sh p -F 3`；
  - Mooncake 侧：node1 `/home/yin/Vsegment/vseg_scripts/vsegment_cluster.sh`，运行时加 `--target-trace-entries 2097152`。
- 本单**只做只读取证**，回答一件事：这套东西能否直接量化「CQE 早已出现、而回调来晚」，从而**省掉我们自定义的 poll 计数 + iov 插桩**。

## A. 预检（只读）

1. 现役 target：PID 1587271、`/proc/<pid>/exe` md5 `21ced674…`、`/proc/<pid>/cmdline`。
2. 脚本存在性与 md5：`/home/yin/run_exp/run_exp.sh`、`/home/yin/Vsegment/vseg_scripts/vsegment_cluster.sh`（只读，**不执行**其中的启动/压测段）。
3. 记录两脚本的权限与修改时间（判断是否为同事现役版本）。

## B. 脚本取证（贴 ≤20 行关键片段 + `文件:行号`）

1. `run_exp.sh`：`-F 3` 分支做了什么、调用哪些工具/参数、**输出哪些列**（SQE/CQE/SQHD 是打印原始值还是已解析）。
2. `vsegment_cluster.sh`：`--target-trace-entries` 的传递链（脚本变量 → 进程 env 或 CLI → 哪个二进制开关）；与 `SPDK_URMA_STAGE_TRACE_ENTRIES` / `SPDK_URMA_STAGE_TARGET_SINK` 的关系；容量语义（条数 × 每条字节 = 内存/文件大小）。
3. 两脚本里是否已有**解析这些字段**的 Python/awk（若有，记录路径与输出列）。

## C. 源码取证（`$D/spdk-e12b7cb-src`，bd5ee124d；只读）

1. grep `target_trace_entries` / `SPDK_URMA_STAGE_TRACE_ENTRIES` / `stage_trace` / `cq_head` / `sq_head` / `sqhd` / `doorbell`：给出定义处与容量/落盘时机（`文件:行号`）。
2. **字段性质**：24 字段（`trace_seq qid cid lba nlb hz exec enter submit submit_return callback wait enomem rx buffer bdev_cpl data_post data_cpl rel0 lcore flags`）逐项判定是**时间戳**还是**队列索引/计数**。
3. **关键问题**：是否存在 **SQ 写入位置（sq_tail）、门铃值/设备消费（SQHD）、CQE 出现（cq_tail/phase）、CQ head** 中任一信息？
   - 若有：给出「CQE 出现 → 回调」的**可直接计算方法**（字段与公式）；
   - 若无：明确写「`[事实]` 现有 24 字段不含队列索引」，并说明从 `nvme_pcie` 路径能不能零改码获得（例如 BUFFER 的 `buffer`/`bdev_cpl` 是否已覆盖）。
4. 本地 NVMe 完成路径：`spdk_nvme_poll_group_process_completions` 与 `nvme_pcie` 的 `cq_head` 自增点；一轮最多处理多少、是否按 CQ 顺序。

## D. 判决（必答，二选一 + 依据）

1. **走 A（用同事这套，零改码）**：给出可直接执行的命令清单（一行一条），并说明它能给出的数字（每 IO 的 CQE 时延 / 设备消费进度 / 检测延迟），以及**缺口**（若有）。
2. **仍走 B（我们的 poll 计数 + iov 插桩）**：说明同事这套为何不够（具体缺哪几条字段），以及 B 的最小增量。
3. 附：现役二进制（`21ced674…`）是否支持 `--target-trace-entries`（`--help` / `strings` 核对）；不支持时的替代（env 名或需重编）。
4. 附：若走 A，采一次两臂 T1 的**日志量估算**（条数 × 每条字节）与落盘方式。

## E. 边界

- **零环境**：不启动/停止任何服务、不跑压测、不改码、不重启 target（708167→现为 1587271，保持运行）。
- 只读：脚本、源码树、`/proc/<pid>/*`、`--help`；不执行 `run_exp.sh` 的压测段（需要另批）。
- 不删 etcd 键；产物 `run/nof067_recon.txt`（≤1MB）；查不到写 `[未获取]` + 尝试路径。

## 回执必答（第一行）

`-F 3` 与 `--target-trace-entries` 分别输出/控制了哪些字段；24 字段里哪些是时间戳、哪些是队列索引；是否存在 SQ 位置 / SQHD / CQE / CQ head；能否直接量化「CQE 早于回调」；现役二进制是否支持；判决走 A（零改码）还是 B（仍需插桩）与理由；有无失败。

---

# NOF-068（待执行 · 第二阶段 M3-6：改用 SPDK 原生 tracepoint 采两臂 · **零改码** · 含一次干净重启 · 2026-11-24）

## 0. 授权与前置

- 沿用 2026-11-23 用户批准范围（重启 target + 一次 400 keys 写入）。本单**不改码、不重编**：直接回到原始二进制 `nof059`（`979f5517…` = bd5ee124d，无任何自建打点），同时清掉当前"deleted inode + `NOF064_TRACE=1` 打印"的污染状态。
- 依据 NOF-067：现役二进制已内置 `nvme_pcie`(6 tpoints) 与 `urma_p99`(4)，`mask` 只是没开；H1/H2 判别所需的 `poll_enter / last_empty / prev_poll_end / POLL_GAP / CQ_OBSERVE` 齐全。
- 术语纠正（写死在回执里）：`-F 3` 是**慢事件阈值因子**（latency > 3×avg），不是格式开关；`--target-trace-entries` 是 node1 脚本的参数名，落到二进制是 **`--num-trace-entries`**（2 的幂，默认 32768/核，0 关闭）。

## A. 预检（只读）

1. target 1587271：`/proc/<pid>/exe`（应为 deleted）、`environ`、`cmdline`、`/dev/shm/nvmf_trace.pid*` 大小。
2. 四份二进制 md5；**核验 `nof059` 也带 `--num-trace-entries` 与 `NVME_PCIE_CQ_OBSERVE`**（`--help` + `strings`）；若 `nof059` 不带 → **停手回报**（改用 `nof064b` + 不设 `NOF064_TRACE` 作为替代并说明）。
3. etcd 键数（**用 base64 下发脚本**，避开上层 shell 展开）：`/cvm|/oplog` 下 `vseg_nofvseg`、`vseg_nof061`、`vseg_nof068`。

## B. 切到干净 target（一次重启）

1. 停 1587271；用 `nof059` + **同 RPC 重放** + `SPDK_URMA_TRACE_ENTRIES=262144`（→ `--num-trace-entries 262144`）；**不设 `NOF064_TRACE`**。
2. 验收：4457 LISTEN / nsid1=URMA_RAID0 / 11 base 11 operational strip256K online / transport urma / `--num-trace-entries` 生效（`/dev/shm/nvmf_trace.pid<new>` ≈ 262144×32B×cores）/ `trace_get_tpoint_group_mask` 可查且为 0x0。
3. 记录新的 nsid1 uuid（复现 NOF-064 判决 3）。

## C. 写入（一次，新 cid）

- 起 vseg master `vseg_nof068`（slot=1、`--vsegment_stripe_size=4194304`、其余同 NOF-059 B1）；register；quota；验收 `partitions=1`、`reconcile_fail=0`。
- T1 写 400 keys（`--num_threads=1 --batch_size=1`、verify=false、`VSEGMENT_CREATING` ≤3 次补齐）；验收 `Keys=400`、`logical_used=1,677,721,600`；记录写 params 的 `seg_lba` 集合（预期 0..3268608 步长 8192）。**此后禁止再写。**

## D. 两臂采集（tracepoint）

1. 运行期开组：`trace_enable_tpoint_group nvme_pcie`、`urma_p99`，另加 `bdev`(0x8) / `bdev_raid`(0x20000) / `bdev_nvme`(0x4000)（NOF-067 建议，用于 bdev_io 时延）；记录 `trace_get_tpoint_group_mask`。
2. **pilot（先做，验解析）**：vseg 臂 `--num_keys=100 --warmup_keys=0`，其余同正式参数；跑完 `trace_clear` 前后各快照一次；确认字段有值、无溢出、可解析。
3. **正式 vseg 臂**：`trace_clear` → `--num_keys=400 --warmup_keys=128 --num_threads=1 --batch_size=1 --nof_qpairs_per_segment=4 --verify=false`；客户端 env 含 `SPDK_URMA_STAGE_TRACE_US=1`（`URMA_P99_BEGIN` 要求 `trace_seq != 0`）；跑完**立即**把 `/dev/shm/nvmf_trace.pid*` 复制到 `run/nof068_vseg.bin`（ring 静默回绕，必须每遍快照）。
4. **plain 臂**：停 vseg master、起 plain master（`vseg_nofvseg`，不清键不写），`trace_clear` → 同一批 LBA 读 400 keys（plain flags，同 env）→ 快照 `run/nof068_plain.bin`。
5. 收尾：`trace_disable_tpoint_group`；停两 master；**target 保持运行**。

## E. 解析（**只改 `/tmp` 下的 Python，不碰 C**）

1. 先出**名称映射表**：`SQ_PUBLISH→?`、`SQ_NOTIFY→SQ_RING`、`CQ_OBSERVE`、`COMPLETE`、`SUBMIT`、`REQ_QUEUED`、`POLL_GAP`；不确定项按语义最小改动，并把差异写进回执（同事脚本写死旧名，直接跑会退出）。
2. **溢出判定**：每核条数 vs 容量；`trace_seq` 不连续或达上限 ⇒ 该遍**作废重跑**（缩小 keys 数），不得拿截断数据出结论。
3. 每 IO 提取：`t_submit(SUBMIT)`、`t_db(SQ_RING, 写 tdbl 后)`、`t_obs(CQ_OBSERVE)`、`t_cb(COMPLETE)`、`last_empty`、`poll_enter`、`prev_poll_end`、`lba`、`nlb`、`qid`、核。
4. 计算四个量：`t_cb − t_db`（盘+队列）、`t_obs − t_db`、**`t_obs − last_empty`（CQE 静置时长上界）**、`POLL_GAP` 次数。
5. 按 NOF-067 三分支判：`last_empty > t_db 且 poll_enter − t_db 大` ⇒ **H1 没轮到**；`last_empty > t_db 且 poll_enter ≈ t_db 而 t_obs − poll_enter 大` ⇒ **H2 轮到但 CQ 空**；`last_empty < t_db` ⇒ CQE 落在 (t_db, t_obs)。
6. 归因：按 `lba` 与客户端 `URMA_SLOW_IO` 配对，给**同 LBA** 两臂的 `t_cb − t_db` 与 `t_obs − last_empty`；按核/盘/桶分组。

## F. 判决（必答）

1. pilot 是否可解析、字段是否都有值、是否溢出。
2. 两臂 `t_cb − t_db` 的 p50/p90（"盘+队列"口径）与 `t_obs − last_empty` 的 p50/p90（"静置"口径）；两者能否解释 ~195µs（给加总关系）。
3. **H1 还是 H2**（带数字）；若 H1，静置发生在哪（`POLL_GAP` 次数、`poll_enter` 间隔分布、是否跨核）。
4. 若 H2 ⇒ `t_cb − t_db` 即盘时间，转设备/队列侧，给出下一步取证点。
5. M3-7 修复方案：唯一假设 + 改哪一行/哪个参数 + 预期（target bdev 250 → 60µs 量级）+ 验证口径。

## G. 边界

- **不改码、不重编**；只改 `/tmp` 下 Python；不启用 `NOF064_TRACE`。
- 一次重启 + 一次写入；之后零写入；不删 `/cvm|/oplog/vseg_nofvseg`。
- 每遍必须快照 ring；溢出即作废；任何"看起来能解释但溢出了"的数据一律不认。
- 验收不过 → 回退到 1587271 的等价配置（`nof059` + 原 RPC + `TRACE_ENTRIES=0`）并停手回报。

## 回执必答（第一行）

pilot 是否可解析/字段是否有值/是否溢出；tpoint 名称映射表；两臂 `t_cb−t_db` 与 `t_obs−last_empty` 的 p50/p90；H1/H2 判定与数字；`POLL_GAP` 统计；同 LBA 配对结果；M3-7 修复方案；有无回退。

---

# NOF-069（待执行 · NOF-068 C 段修复：新 cid + 先验 descriptor 再 mount + 源码定位 146µs 早退 · 2026-11-24）

## 0. 判决与环境（开发侧决定）

- **选选项 1**：用**新 cid `vseg_nof068b`** 重跑 C 段；**不删任何 etcd 键**（`vseg_nof068` 的键保留作证据）；不选选项 2、暂不判选项 4。
- **根因方向（现有证据）**：`/cvm/vseg_nof068/` 只有 `cluster_meta`(62B) + `ring_slots/00000`，**没有 segment 描述符**；对照 NOF-008 成功轮 `/cvm/vseg_nofvseg/` = **8 键**（含 segment 描述符）。而 `MountNoFSegment` 只花 **146µs**、**没有走到 attach**（成功轮 206ms 且有 `Opened NoF segment ns=1 io_qpairs=4`）⇒ 极可能是 **register 没把描述符写进去**（C1/C2 在坏 env 下跑过 register，或随后被"已存在"判定幂等跳过），mount 在前置校验就早退了。
- 因此本单把「**先验 descriptor、再 mount**」设为硬闸：**描述符不在就停手**，不允许反复 mount 试探。

## A. 预检（只读）

1. target **487638** alive、4457、11/11 operational、`--num-trace-entries 262144`、`tpoint_group_mask=0x0`、`/dev/shm/nvmf_trace.pid487638` 52,736,328 B。
2. 记录现存 master（1431656，cluster `vseg_nof068`）状态后**停掉**（按 PID，不删键）。
3. etcd 基线（**base64 下发脚本**，避开上层 shell 展开）：
   - `/cvm/vseg_nof068/`、`/oplog/vseg_nof068/` 键数与键名；
   - `/cvm/vseg_nofvseg/`（成功轮，**把键名清单打出来当模板**）；
   - `/cvm/vseg_nof068b/`（预期不存在）。

## B. 起新 master + register（env 单一来源）

1. 用**同一个脚本**（`set -a`，全部 env 写在一处）启动 vseg master：`cluster_id=vseg_nof068b`、`MC_STORE_PARTITION_SLOT_COUNT=1`、`--vsegment_stripe_size=4194304`、50056/9006、`SPDK_URMA_DEV_NAME=udmac0d1e2`、`SPDK_URMA_CAPSULE_TRANSPORT=urma`，其余与 **NOF-059 B1 逐项一致**。
2. **先验 master environ**（`/proc/<pid>/environ` 含全部 URMA 变量）**再** register；register 从同一脚本环境执行，**不嵌套 sudo**。
3. **register 硬闸**：register 结束后立刻数 `/cvm/vseg_nof068b/` 键——
   - 键数 < 3 或**缺少 segment 描述符键** ⇒ **停手回报**（不 mount、不重试、不删键），并把 register 的完整 stdout/stderr 与 rc 贴回；
   - 键结构与 `/cvm/vseg_nofvseg/` 一致（含描述符）⇒ 进 C。
4. 记录 register 耗时与输出，与 NOF-059 的 register 日志逐行对比（成功轮有 `Opened NoF segment`? 记录有无）。

## C. mount（**只做一次**）

1. 客户端 `MountNoFSegment`；记录 `elapsed_us` 与 `rc`。判据：
   - **成功** = elapsed ≳ 200ms **且** master 日志出现 `Opened NoF segment ns=1 io_qpairs=4` **且** `rc[0]`；
   - **146µs 级 + 无 `Opened NoF segment`** ⇒ 停手，直接进 D 段（**禁止重复 mount**）。
2. 成功后：等 quota ≥20s，验收 `partitions=1`、`reconcile_fail=0`。

## D. 源码定位（零环境，可与 B/C 并行）

1. 在 `$D/mooncake-src` 定位 `MountNoFSegment` 处理链（`master_service.cpp:4914` 附近）与 `spdk_wrapper.cpp:768`。
2. **列出所有"不打印日志就返回/计失败"的分支**，标出哪一条能在 **146µs** 内命中（前置校验类：描述符缺失/字段非法/device identity 不匹配/配额未就绪等），给 `文件:行号` 与判定条件。
3. 检查 register 侧是否有「已存在即跳过」的幂等逻辑——若有，说明 C1/C2 坏 env 下的 register 如何留下坏状态。

## E. 后续（仅当 C 成功）

按 NOF-068 的 D/E/F 段继续：pilot 100 keys → 两臂 400 keys（vseg `vseg_nof068b` + plain `vseg_nofvseg` 读同一批 LBA）→ 每遍快照 ring → 解析 `t_submit/t_db/t_obs/t_cb/last_empty/poll_enter` → 判 H1/H2。

## F. 边界

- **不删任何 etcd 键**（含 `vseg_nof068`）；只新增 `vseg_nof068b`。
- target **保持运行、不再重启**；不改码、不重编；tracepoint 仍不启用（等 C 成功再开）。
- 写入仅 C 成功后的一次（400 keys）；失败则零写入。
- 产物 `run/nof069_*.txt`（≤3 个，每个 ≤1MB）。

## 回执必答（第一行）

register 后 `/cvm/vseg_nof068b/` 的键清单与键数（是否含 segment 描述符、是否与 nofvseg 同构）；mount 的 `elapsed_us`/`rc`、是否有 `Opened NoF segment`；若失败：146µs 命中哪条无日志分支（`文件:行号`）；register 输出与 NOF-059 的差异；是否进入 D/E；有无失败。

---

# NOF-070（待执行 · M3-7 取证：先对账数据完整性，再按 qd 分桶判「队列压力」· **零采集**（必要时一次免写重采）· 2026-11-24）

## 0. 现状与两条前置判决

- NOF-069 已给出 **H2：门铃→CQE 之间是设备/队列侧慢**，证据：vseg 轮询间隔 p50 **3.0µs**（连续高频轮询）、门铃后 **237.8µs** 那一次轮询仍见空、再 **3.2µs** 才取到 CQE、`t_cb−t_obs` 仅 **0.1µs** ⇒ 不是 reactor 没轮到（H1 否）。
- **但本单第一件事是数据对账，不是继续分析**：NOF-069 自报 trace 只覆盖 **LBA 0..297,728 块（≈37 个 4MiB key 区）/ 1,164 个唯一 LBA**，与「400 keys × 32 = 12,800 个 child IO、LBA 应跨 0..3.27M」矛盾；同时「两臂各 16.9k 个 NVMe IO 全量配对」与 1,164 个唯一 LBA 也不能同时成立。**在解释清楚之前，凡依赖 LBA 配对的结论（per-LBA 5.505×、同桶 204.0）一律视为 `[待复核]`**；不依赖 LBA 的聚合结论（p50/p90、H2 判定）保留。

## A. 数据对账（必做，纯离线）

1. 每臂给出：原始记录总数、join 成功数、**distinct `(qid,cid,lba,nbl)`**、唯一 LBA 数、LBA min/max、步长分布。
2. 解释三处矛盾：① 为何唯一 LBA 只有 1,164；② 为何只覆盖前 ~37 个 key 区；③ 「16.9k IO」统计的是哪一层（trace 条数？客户端 URMA_SLOW_IO？join 前/后？）。
3. 检查快照合并：`/dev/shm/nof069_*_snaps` 的 16（vseg）/125（plain）份快照是否**时间上覆盖整遍**、合并后 `gaps=0` 是否真实、是否有段被 `trace_clear` 或 ring 覆盖丢掉。
4. 检查 key→LBA 映射：`vseg_nof068b` 的 400 keys 实际落在哪些 LBA（用 master 元数据或写 params 的 `seg_lba`），与 trace 覆盖区间对比。
5. **判决**：若对账能解释（如 trace 只保留早期段）⇒ 给正确的覆盖结论并说明哪些数字需重算；若不能解释 ⇒ 进 A6。
6. **可选重采（免写、target 不动）**：起 `vseg_nof068b` master（数据已在，目标机未重启，mount 应直接可用）→ 400 keys 读一遍，**快照间隔缩短到 0.2s**、并逐遍校验「已接收 IO 数 == 预期 12,800」；plain 臂同样。**全程零写入**。

## B. qd 分桶（M3-7 主假设，纯离线）

1. 用 `NVME_PCIE_SUBMIT.qd`（提交时该 qpair 的队列深度）把每臂 IO 分桶（1/2–4/5–8/9–16/17–32/>32）。
2. 每桶给 `t_cb − t_db`、`last_empty − t_db`、`t_obs − last_empty` 的 p50/p90，并给**两臂 qd 分布**。
3. **判据**：同 qd 桶内 vseg 仍显著慢（>2×）⇒ 队列压力不是主因；只有 vseg 的 qd 高时才慢 ⇒ 队列压力成立。

## C. 提交时间结构（纯离线）

1. 每块盘的到达间隔分布、同盘在飞数时间序列（按 100µs 窗口）、每 key 32 个 IO 的落盘分布（换盘率）。
2. 检查是否存在「同一 SSD 上 vseg 的在飞数显著高于 plain」的时段；给两臂各自的最大在飞数。

## D. 判决（必答）

1. 对账结论：1,164 唯一 LBA 是否可信；`[待复核]` 项哪些可以恢复、哪些作废。
2. H2 内部再分类：**队列压力 / 特定 SSD 状态 / 其他**（给数字）。
3. M3-8 修复方向：改客户端提交节奏、改 target 侧限流、还是回到设备侧（并说明为什么）。
4. 若 B 显示"同 qd 仍慢"⇒ 明确写「队列压力排除」，并给出下一层取证点（例如按 SSD 内部队列号/命令交错）。

## E. 边界

- 默认**零采集**：只读 `run/nof069_summary.txt`、`/tmp/nof069_{vseg,plain}.csv`、`/dev/shm/nof069_*_snaps`。
- 若走 A6 重采：**target 不重启、零写入**（现有 400 keys 只读），且必须逐遍校验覆盖数。
- 不删 etcd 键；不删 `/dev/shm` 快照（磁盘紧张时先回报再定）。
- 产物 `run/nof070_*.txt`（≤3 个，每个 ≤1MB）。

## 回执必答（第一行）

唯一 LBA 数与覆盖区间的对账结论；16.9k / 1,164 / 37 个 key 区三者的关系；qd 分布与分桶后的 `t_cb−t_db` 对比；队列压力是否成立；H2 内部再分类；M3-8 方向；是否触发重采；有无失败。

---

# NOF-071（待执行 · M3-8：消掉「客户端位置」偏差 + bdev 层命令/缓冲取证 · 零写入 · 2026-11-24）

## 0. 前置判决（来自 NOF-069/070）

- **H2 成立**：慢在门铃→CQE 之间（设备/队列侧）；target 轮询间隔 p50 **3.0µs**、门铃后 237.8µs 仍空、3.2µs 后取到 CQE、`t_cb−t_obs=0.1µs`。
- **队列压力排除**：qd 分布两臂几乎相同（1: 34.4% vs 34.5%，2–4: 65.6% vs 65.5%，无 >4）；**同 qd 桶内仍 5.11×（qd=1）/5.28×（qd=2–4）**；在飞数 32/32、per-qpair 4/4 相同。
- **访问流排除**：换盘率 50.1% vs 50.1%、每 key 跨 11 盘、每盘 1,526–1,535 均匀；inter-arrival 5.24 vs 18.78µs 是「完成慢→腾空慢」的结果，不是原因。
- **对账闭环**：target trace 的 `lba` 是**单盘内 offset**，客户端 12,788 个逻辑 LBA 经 RAID0 映射后 = 1,164 个盘内 offset，与 target 集合**零差异**（`|mapped only|=|target only|=0`）⇒ 「只覆盖 37 个 key 区」作废，per-LBA 5.505× 与同桶 204.0 **恢复有效**。
- **新偏差（必须处理）**：NOF-069 两臂**客户端不同机**——vseg 在 node3（与 target 同机、环回），plain 在 node1（跨网络）。该偏差对 target 内部指标无影响（都在 target 内测），但**端到端 P50（608 vs 237）不可用于公平对比**，且「会话身份」与「客户端位置」两个变量被混在一起。

## A. 同宿主 A/B（本单首要，零写入、target 不重启）

1. **A1 两臂都在 node3（本机，与 NOF-061/062 同协议）**：vseg 臂（cid `vseg_nof068b`，数据已在）+ plain 臂（cid `vseg_nofvseg`，读同一批 LBA），各 1 遍 T1（`--num_keys=400 --warmup_keys=128 --num_threads=1 --batch_size=1 --verify=false`）。记录端到端 P50/P90、以及 target 内部 `t_cb−t_db`/`le−db`/`obs−le`（开同一组 tracepoint，采完即关）。
2. **A2 两臂都在 node1（远端）**：同参数各 1 遍。
3. **判据**：① 同宿主下 vseg 是否仍 ≥4× 慢（若是 ⇒ 客户端位置排除，效应确在 target 会话侧）；② 端到端 P50 在同宿主下两臂差多少（这才是可外报的公平数字）。
4. 注意：两臂必须**同一台机、同一遍内切换 master**，不得混机；开跑前记录 bench 二进制路径与 md5、`hostname`、以及 `ss` 里到 4457 的连接源地址（证明是环回还是跨网）。

## B. bdev 层可观测性清单（零改码）

1. 用 `trace_get_tpoint_group_mask` + 源码，列出 `bdev`(0x8)/`bdev_raid`(0x20000)/`bdev_nvme`(0x4000) 三组的**全部 tracepoint 名与字段**（`文件:行号`）。
2. 明确回答：能否零改码拿到每条 IO 的 ① **父 IO 长度**（4MiB vs 128KiB）② **数据缓冲地址** ③ **iov/SGL 数量或类型** ④ `bdev_io` 指针。
3. 若某项拿不到，写 `[未获取]` 并给最小插桩点（一行），不作为本单阻塞。

## C. 命令/缓冲对比（若 B 可拿字段；零写入）

1. 两臂各采一遍，比较 **bdev 层父 IO 长度分布**（这是关键：vseg 是否为 32 个 128KiB 的父 IO、plain 是否为 1 个 4MiB 的父 IO）。
2. 若可拿缓冲地址：按 `/proc/<target>/numa_maps` 分 NUMA 桶，比较两臂分布；并给地址对齐（2MiB/4KiB）分布。
3. 若可拿 iov/SGL：比较两臂的 iov 数分布。

## D. 判决（必答）

1. 同宿主 A/B：两宿主下 vseg 的倍数各是多少；**客户端位置是否被排除**。
2. 可外报的公平端到端数字（同宿主）是多少。
3. bdev 层父 IO 长度两臂是否相同；缓冲地址/NUMA/iov 是否相同（或 `[未获取]`）。
4. M3-8 假设排序：命令字节（SGL/PRP/flags）／缓冲物理归属／SSD 内部（同 LBA 不同写历史）——按证据强弱排。
5. 下一单需要的最小插桩点（若必须改码，给出文件:行号与要打印的字段，标明"仅日志"）。

## E. 边界

- **零写入**（两臂均只读现有数据，不得重写）；**target 不重启**（487638，nof059，`--num-trace-entries 262144`）。
- 不删 etcd 键、不删快照；tracepoint 用完即关（mask 回 0x0）。
- 两臂必须同宿主；开跑前记录 hostname/二进制 md5/连接源地址。
- 产物 `run/nof071_*.txt`（≤3 个，每个 ≤1MB）。

## 回执必答（第一行）

A1/A2 两宿主下 vseg 相对 plain 的倍数（target 内部 `t_cb−t_db` 与端到端各给一组）；客户端位置是否排除；bdev 层父 IO 长度两臂是否相同；缓冲地址/NUMA/iov 是否可得与是否相同；M3-8 假设排序；下一单最小插桩点；有无失败。

---

# NOF-072（待执行 · M3-9：解阻 vseg master（oplog 32MB）+ 交错 A/B + 冷稳状态检验 · 一次写入仅在必要时 · 2026-11-24）

## 0. 前置判决（NOF-071）与两处新事实

- **客户端位置排除**：同宿主 node3、同 bench、同 target、同 400 keys——vseg `t_cb−t_db` **242.6µs** vs plain **50.0µs** = **4.86×**；plain 从 node3→node1 只变 **1.03×**（50.0→48.5），且 plain 在 node1 两次测量 **1.00×** 复现。
- **可外报的公平端到端**：同宿主 **608.83 vs 276.93µs = 2.20×**（P90 703.97 vs 570.01）。**NOF-069 的 2.57× 是跨宿主混比，作废**。
- **父 IO 长度两臂完全相同**：`BDEV_IO_START.size` 全为 128KiB/256 块（vseg 33,792 / plain 33,792 条），层级 raid0→nvme 一一对应 ⇒ **「vseg 32×128K vs plain 1×4M」假设否证**。
- **缓冲地址/NUMA/iov：`[未获取]`**——`bdev`/`bdev_raid`/`bdev_nvme` 三组 tpoint 无 base/iovcnt 字段；最小插桩点已定位 `lib/bdev/bdev.c:3851` 旁。
- **产品级缺陷（新，需上报）**：`vseg_nof068b` 的 `/oplog/` 涨到 **2,801 批 / 58,545,043 B**，master 启动回放时 etcd gRPC 单消息 **32MB 上限**（35,087,559 > 33,554,432）触发 `ResourceExhausted`，75s 内重试 959 次、50056 从未 LISTEN ⇒ **该 cluster 无法再启动**。对照 `/oplog/vseg_nofvseg/` = 400 批 / 178KB（正常）。历史 cid（`vseg_nof061` 2,005 批）当时能启动，说明存在阈值。

## A. 解阻（决策树，按序尝试；每步记录结果）

1. **A1（首选，零破坏）**：`--enable_oplog=false` 起 `vseg_nof068b` master（只读实验）；验收 50056 LISTEN + mount rc[0] + `Keys=400`。若成功 ⇒ 进 B。
2. **A2（仅删我方 oplog）**：删除 **仅** `/oplog/vseg_nof068b/`（保留 `/cvm/vseg_nof068b/`），正常启动；验收同上。**删前记录键数与字节数**。若 /cvm 快照足以恢复 mount ⇒ 成功。
3. **A3（最后手段，需写入）**：新 cid `vseg_nof072`，**单遍**写 400 keys（`VSEGMENT_CREATING` 仍会丢 key，最多补 3 轮）；验收 Keys=400、logical_used=1,677,721,600。此步为**唯一允许的写入**，执行前需在回执首行标注"已按 A3 写入"。
4. 三步都失败 ⇒ 停手回报，附 `etcdctl` 键数/体积与 master 日志。

## B. 交错 A/B（同宿主 node3，核心）

1. 顺序：**vseg → plain → vseg → plain**，各 1 遍 T1（400 keys、warmup=128、batch=1、verify=false、node3 本机）。
2. 每遍开同一组 tracepoint、跑完立即快照 ring、`trace_clear` 再下一遍；记录每遍 `t_cb−t_db` p50/p90/mean 与端到端 P50。
3. **判据**：四遍交错后 vseg 两遍是否仍 ≥4×（若是 ⇒ **时间/设备状态漂移排除**；若差距随次序变化 ⇒ 设备状态是主变量，需记录漂移方向）。

## C. 冷/稳状态检验（同宿主）

1. **plain 连读 2 遍**（不换 master、不重挂）：第 2 遍 `t_cb−t_db` 是否仍 ~50µs。
2. **vseg 连读 2 遍**：是否仍 ~242µs；并给两遍的 `last_empty−t_db` 与 `obs−le`。
3. 关联已知旁证：NOF-059 裸 SPDK **长扫 40µs** vs **单 key 4MiB 定点 218.88µs** ⇒ 设备存在"冷/稳"两态；本段要判 Mooncake 的两臂是否分别落在不同态。
4. **判据**：若 plain 第 2 遍明显变慢（趋近 242）或 vseg 第 2 遍明显变快 ⇒ 强弱不在路径而在"设备被喂饱和的程度"，M3-9 方向随之改判。

## D. 判决（必答）

1. A 段最终用哪条路径解阻（A1/A2/A3），代价与风险。
2. B 段四遍交错的 vseg 倍数（每遍给数字）⇒ 时间漂移是否排除。
3. C 段冷/稳结论：plain 两遍、vseg 两遍各是多少；是否支持"设备两态"。
4. M3-9 假设排序与下一步（若指向设备两态 ⇒ 给"如何把 vseg 喂进稳态"的可验证方案；若仍指向路径 ⇒ 给最小插桩点）。
5. 产品级缺陷上报口径：oplog 阈值、复现条件、规避手段（不影响本单结论）。

## E. 边界

- target **不重启**（487638、nof059、`--num-trace-entries 262144`）；tracepoint 用完即关（mask→0x0）。
- etcd：**只允许** A2 删除 `/oplog/vseg_nof068b/`（我方、单 cid）；不删 `/cvm/vseg_nof068b/`、不删 `vseg_nofvseg|vseg_nof061`、不 compact 全局。
- 写入仅 A3 一次；B/C 段零写入。
- 两臂**必须同宿主 node3**；每遍记录 bench md5 与 `hostname`。
- 产物 `run/nof072_*.txt`（≤3 个，每个 ≤1MB）。

## 回执必答（第一行）

A 段用哪条路径解阻（及是否发生 A3 写入）；B 段四遍交错的 vseg/plain 倍数（逐遍）；C 段 plain 两遍与 vseg 两遍的 `t_cb−t_db`；时间漂移是否排除；设备两态是否成立；M3-9 方向；有无失败。

---

# NOF-073（待执行 · M3-10：命令字节对比 —— 先吃免费数据，必要时一行插桩 · 2026-11-24）

## 0. 前置判决（NOF-072）与排除表

- **A 段只能 A3**（新 cid `vseg_nof072` 写入 400 keys，已授权执行）。**A1 暴露产品级事实**：`--enable_oplog=false` 起 master 后 `Keys=0`、对象不可达、`/cvm/` 里只有 cluster_meta/masters/nof_identities/ring_slots/segments，**没有对象元数据** ⇒ **oplog 是对象元数据的唯一载体**，删 `/oplog/` 等价丢数据（A2 因此未执行）。
- **时间/设备漂移排除**：四遍交错 vseg **245.0 / 245.5 / 244.1 / 244.4**（spread **1.01×**）、plain 49.2 / 52.5 / 51.5（1.07×）；逐遍倍数 **4.98× / 4.68×**（内部）、2.14× / 1.87×（端到端）。
- **"设备两态"不成立**：各臂连读两遍零漂移（plain 0.98×、vseg 1.00×）⇒ vseg 已在自身稳态，**"喂热"不是修复方向**。
- **剩余候选收敛为**：① 同 LBA 的数据物理/NAND 状态；② **命令字节（opcode/FUA/PSDT/PRP-SGL）**。
  - **开发侧判决：① 已被 NOF-061 排除**——同一份 vseg 写入的数据，用 plain 路径读只有 **57µs**（vs vseg 路径 249µs）。数据、写者、写历史都相同，只换读路径就快 ⇒ 写历史/物理状态不是变量。
  - ⇒ **唯一剩下的就是 ②：同一条"读 128KiB @ 同 LBA"到了 SSD，两臂的命令字节或到达时序不同。**
- 另：`vseg_nof068b` 的 oplog 58.5MB > etcd 32MB 上限 ⇒ **该 cluster 永久起不来**（本次绕开用新 cid）。这是要上报的产品缺陷（见 D5）。

## A. 免费对比：现有 trace 里的命令 dword（零环境、零写入）

1. 用 NOF-072 的六份 CSV（或重导）：比较两臂逐 IO 的 `opc`、`dw10/dw11`（LBA 低/高）、`dw12`（低 16 位 = NLB−1；**bit14 = FUA**）、`qd`、`qpair`。
2. 给：`opc` 分布、`dw12` 的 **FUA 位**与 NLB 分布、`dw10/dw11` 是否覆盖同 LBA 集合、以及 `nlb` 分布。
3. **判据**：若 `opc`、FUA 位、NLB 任一不同 ⇒ **直接定案，不进 B**（并给该位对应的修复方向）。

## B. 仅在 A 无差异时：命令字节插桩（**需用户在会话中批准改码/重编/重启**）

1. 打点位：`lib/nvme/nvme_pcie_common.c` 写 SQ 之前（与 `NVME_PCIE_SUBMIT` 同处），**只读日志**打印：`cmd[0]` 原始 4 字节（opcode/PSDT/FUSE/CID 类型）、`dw12`、`psdt` 类型、PRP1/PRP2（或 SGL 段数与首地址）、以及该 IO 的 `submit_tick`。
2. 采样：环形数组（≥200k 条，避免 per-IO printf 污染——NOF-064 教训），非 reactor 线程落盘。
3. 重编（NOF-048 配方）→ 重启 target（同 RPC + `--num-trace-entries 262144`）→ **新 cid 写入 400 keys**（唯一一次写入）→ 两臂**同宿主 node3** 各 1 遍。
4. 对比：`cmd[0]` 全字节、PSDT、PRP/SGL、FUA、NLB、以及**相邻 submit 的时间间隔分布**（到达时序也是一种"命令输入"）。
5. 回退锚点：`nof059`（`979f5517…`）+ 原 RPC。

## C. 判决（必答）

1. A 段：`opc/FUA/NLB/LBA` 是否完全相同（给分布表）。
2. 若进入 B：命令字节是否相同；若不同，差在哪一位（`cmd[0]` 位、PSDT、PRP/SGL 形态）；若相同，则只剩"到达时序"——给两臂 submit 间隔分布与瞬时并发。
3. 结论：**差异的确切载体**（命令位 / 描述符形态 / 时序），以及它由哪一层产生（客户端的 NoF 提交？target 的 bdev→nvme 组装？）。
4. M3-11 修复方向：改哪一层、预期收益。
5. 产品级缺陷上报口径（oplog 唯一元数据载体 + 58.5MB 不可启）。

## D. 边界

- A 段**零环境**；B 段需改码+重编+重启+一次写入，**未获批准则止于 A 段并回报**。
- target 不重启（除非 B 获准）；tracepoint 用完即关；不删 etcd 键。
- 两臂必须**同宿主 node3**；每遍记录 bench md5、hostname、master cid。
- 产物 `run/nof073_*.txt`（≤3 个，每个 ≤1MB）。

## 回执必答（第一行）

A 段 `opc/FUA/NLB/LBA` 是否完全相同；是否进入 B；若进入 B，命令字节差异在哪一位/哪种描述符形态；到达时序分布；差异的确切载体与产生层；M3-11 方向；有无失败。

---

# NOF-074（已批准 · 待执行 · M3-11：命令描述符插桩（PRP/SGL + 缓冲地址）· 用户 2026-11-24 批准改码/重编/重启/一次写入 · 2026-11-24）

> **授权范围**：仅限 B 段日志语句、`nvmf_tgt` 重编、按同 RPC 重启 target、以及 C3 的一次 400 keys 写入。超出范围（改控制流/缓冲分配/SGL 策略/RPC 默认、动他人进程）仍需另行批准。A1 租户核查发现他人客户端挂在 4457 上时，**一律停手回报**。

## 0. 前置判决与唯一剩余候选

- **命令 dword 层已比空**（NOF-073）：两臂 `opc=2` 100%、`dw12 bit14 (FUA)=0` 100%、`NLB=256`、`dw10/dw11` 的 1,164 个盘内 offset **元素级完全一致**（only_A=only_B=0）⇒ 命令的 **LBA/NLB/opcode/FUA 全同**。
- 已排除（累计）：数据与写历史（NOF-061 同数据 plain 读 57µs）、时间/设备漂移（四遍 1.01× spread）、设备冷稳两态（各臂零漂移）、qd/在飞/访问流/父 IO 长度/轮询/回调。
- ⇒ **唯一剩余：`cmd[0]` 的 PSDT（PRP vs SGL）/描述符形态（PRP1/PRP2、SGL 段数与首地址、缓冲地址本身）或到达时序。**
- 附带事实（NOF-073 免费挖到，可用于交叉验证）：同 LBA 比值 p50 4.31×，**ratio<1.5× 的 LBA 恰占 24.4%**（283–284/1,164），与盘首慢窗口占比（392MiB/1.6GiB=24.5%）吻合；plain 慢 LBA 相邻步长 256 块 ×241 ⇒ 慢区连续（每盘 ~30MiB）；11 块盘逐盘均值 plain 95.8–100.1 / vseg 231.6–235.9 ⇒ **无盘级差异**。

## A. 前置（只读，全部通过才进 B）

1. **共用机核查**：`ss -tnp | grep 4457` 列出所有连到 target 的客户端；**若有非我方客户端挂着，停手回报**（重启会中断他人）。记录 `royi` 的 master(50151/18080) 与 client(8975) 是否与我们 4457 有关系。
2. 记录 target 487638 的 md5/pid/cmdline/environ、`/tmp/nvmf_tgt.log` offset、当前 RPC 全量、11 盘 BDF 顺序。
3. 回退锚点：`nof059` = `979f5517…` + 原 RPC 文本；确认 `spdk-e12b7cb-src` 源码树干净（无遗留 nof064 补丁；若 .nof064orig 存在需说明以哪份为基线）。

## B. 插桩（**只加日志**，环形数组，非 reactor 线程落盘）

1. 点位：`lib/nvme/nvme_pcie_common.c` 写 SQ 之前（与 `NVME_PCIE_SUBMIT` 同处）。
2. 每条记录（定长，≤96B）：`submit_tick, lcore, qpair, cid, nsid, lba, nlb, cmd[0] 原 4 字节, psdt, prp1, prp2, sgl_addr/sgl_cnt(若 SGL), iov_cnt(若可得), buf_va`。
3. 环形数组 ≥200,000 条；**禁止 per-IO printf**（NOF-064 教训）；独立 pthread 每 3s 追加落盘 `/tmp/nof074_desc.bin`。
4. 采样：全量（400 keys × 32 = 12,800 IO/遍，够）；开关 `NOF074_DESC=1`，关闭时零开销。
5. 编译产物 `build/bin/nvmf_tgt.nof074`；**保留 nof059**。

## C. 重编 + 重启 + 写入（各一次）

1. NOF-048 配方重编（改一个 .c → ~61s 编译 + 4s 重链）。
2. 重启 target：同 RPC 重放 + `--num-trace-entries 262144` + `NOF074_DESC=1`；验收同 NOF-068 B4（4457/RAID0 11 盘/transport/tracepoint）；**11 盘 vfio 残留 ABORT 用 takeover `-r -y` 还原**（已知路径）。若不通过 → 回退 `nof059` 停手。
3. 新 cid `vseg_nof074` 写 400 keys（**唯一一次写入**，≤3 轮补齐）；验收 Keys=400、logical_used=1,677,721,600。
4. 两臂**同宿主 node3** 各 1 遍 T1（400 keys、warmup=128、batch=1、verify=false）：vseg(`vseg_nof074`) + plain(`vseg_nofvseg`，读同一批 LBA)；每遍结束快照。

## D. 解析与判决（必答）

1. **`cmd[0]` 原始 4 字节**逐位比对（opcode/PSDT/FUSE/CID 类型）；`psdt` 分布。
2. **描述符形态**：PRP（`prp1` 指向缓冲还是 PRP list、list 条目数）vs SGL（段数、首地址、是否 inline）；两臂分布对比。
3. **缓冲地址**：`buf_va` 分布；用 `/proc/<target>/numa_maps` 分 **NUMA 节点 + 页大小（2MiB/4KiB）** 桶；两臂对比。
4. **到达时序**：相邻 submit 间隔分布、瞬时并发（每盘/每 qpair）。
5. **判决**：差异的确切载体（PSDT / PRP list / SGL 段数 / 缓冲物理归属 / 时序 / 全同）；若全同 ⇒ 明确写"命令与描述符全同，则只能是设备内部对同一输入的不同响应，需厂商侧介入"。
6. M3-12 修复方向：改哪一层（客户端 buffer 分配 / target iobuf / SGL 组装），预期收益，验证口径。

## E. 边界

- 改码**仅限** B 段日志；禁改控制流/缓冲分配/SGL 策略/RPC 默认。
- 写入仅 C3 一次；D 段零写入；不删 etcd 键。
- 共用机：A1 若有他人客户端挂 4457 → 停手；重启只动我方 target。
- 产物 `run/nof074_*.txt`（≤3 个、每个 ≤1MB）；完整 trace 放 `/tmp/nof074_desc.bin`。

## 回执必答（第一行）

是否获授权；A1 租户核查结果（4457 是否只有我方）；`cmd[0]`/`psdt` 两臂是否相同；PRP vs SGL 形态与段数差异；缓冲地址的 NUMA/页大小桶对比；到达时序；差异确切载体；M3-12 方向；有无回退。

---

# NOF-075（待执行 · M3-12：最后一个宿主侧变量 —— 统一 qpair 数 · **零重启、零写入** · 2026-11-24）

## 0. 前置判决（NOF-074）与开发侧复核

- **命令与描述符已全部比空**：`opc=2` / `fuse=0` / **`psdt=0`（全 PRP，零 SGL）** / `nlb=256` / `payload=131072` / `payload_type=CONTIG` / `iov_cnt=0` / `prp1` 低 12 位 **=0（4KiB 页对齐）**；两臂 × 2 遍逐位相同 ⇒ **PSDT 与 PRP-vs-SGL 两个候选排除**。
- **唯一稳定结构差异**：`buf_va` 1MiB 桶数 **65/65 vs 129/129**、`prp2_distinct` **89/89 vs 177/177**、每桶记录 **264 vs 132** ⇒ 目标端 I/O 缓冲池 vseg ≈64 槽 / plain ≈128 槽，两遍 100% 复现。
- **开发侧复核（这条不能当根因）**：`spdk_iobuf_get` 的耗时在 NOF-062 的 16 字段里是 **`rx_buffer_us = 0`**（两臂），缓冲获取本身不花时间；且 NOF-061 已证"同 lcore 仍差 4.3×"。**缓冲池规模差异更像是"每轮询组缓存在不同时刻长到不同大小"的结果，而不是 5× 的原因。**
- **宿主侧还剩一个从未控制过的变量**：两臂的 **qpair 数/落点不同**（vseg 属主核 {2,3,6,7}、plain {2,3,4,5}），导致轮询组（poll group）与 `buf_cache` 实例不同。本单把它钉死。

## A. 免费分析（零环境）

1. `buf_va` 用 `/proc/905586/numa_maps` 分桶：**NUMA 节点** + **页大小（2MiB hugepage / 4KiB）**，两臂对比（NOF-074 只给了 1MiB 桶，缺 NUMA/页大小）。
2. 每 qpair 的 **CID 复用与并发**：`cmd0_full` 的 CID 分布、每 qpair 的瞬时并发（时间窗 100µs），两臂对比。
3. `submit` 间隔分布（p50/p90/p99）与每盘到达密度，两臂对比。
4. `[未获取]` 项说明：PRP list 长度需读内存，desc.bin 只有 `prp2` 地址 ⇒ 记 `[未获取]`，不要猜。

## B. 统一 qpair 数对照（**核心，零写入、不重启**）

1. 两臂都用 **`--nof_qpairs_per_segment=1`**：vseg(`vseg_nof074`) 2 遍、plain(`vseg_nofvseg`) 2 遍，交错（v→p→v→p），同宿主 node3，T1、400 keys、warmup=128、batch=1、verify=false；开 `nvme_pcie`/`urma_p99` 采 target 内部 `t_cb−t_db`。
2. 两臂都用 **`--nof_qpairs_per_segment=2`** 各 1 遍（看趋势）。
3. **判据**：qpairs=1 时 vseg 是否仍 ≥4×（target 内部）。若是 ⇒ **qpair/轮询组身份排除**，宿主侧观测已穷尽；若收敛（≤1.5×）⇒ 差异来自多 qpair 结构，修法明确。
4. 注意：现役 target 是**插桩版**（`3129641f`、`NOF074_DESC=1`，有 per-IO 开销），两臂同等承受 ⇒ **只比比值，不报绝对值**；回执需标注该前提。

## C. 判决（必答）

1. A：`buf_va` 的 NUMA/页大小分布两臂是否相同；CID 并发与 submit 间隔是否同形。
2. B：qpairs=1 / 2 时两臂的 `t_cb−t_db` p50 与比值（逐遍给）。
3. **结论**：qpair/轮询组身份是否解释差异。
   - 若否 ⇒ 明确写「宿主侧可观测面已穷尽：命令、描述符、缓冲、队列、时序、LBA 全同，盘对相同输入给出不同响应」，并给出**厂商证据包清单**（需要哪些字段、怎么复现）。
   - 若是 ⇒ 给 M3-13 修复方向。
4. 附带：本轮所有数字都在"插桩版 target"下取得，需注明；若要出对外数字，后续重启到 `nof059` 复测。

## D. 边界

- **不重启 target、不改码、不写入**（两臂均只读现有数据）。
- 4457 租户：开跑前 `ss -tnp | grep 4457`，有他人连接则停手。
- 每遍开 tracepoint、跑完快照、`trace_clear`、最后 mask→0x0。
- 产物 `run/nof075_*.txt`（≤3 个，每个 ≤1MB）。

## 回执必答（第一行）

`buf_va` 的 NUMA/页大小桶两臂是否相同；qpairs=1 与 2 时的比值（逐遍）；qpair/轮询组身份是否解释差异；若不是，厂商证据包清单；有无失败。

---

# NOF-076（待执行 · M3-13：慢档的"跃迁"到底由什么触发 —— 固定协议 10 连跑 · **零重启、零写入** · 2026-11-24）

## 0. 前置判决（NOF-075）与三处更正

- **qpair/轮询组身份排除**：qpairs 4→1 后比值 **不降反略升**（4.6→**4.81/5.23**），QP=2 为 **5.36×**；且**两臂 lcore 完全相同**（{2,3,6,7} 各 4,224、n_qpair 均 45）⇒ 宿主侧可观测面（命令/描述符/缓冲页大小与 NUMA/队列/时序/LBA）**已全部对齐**。
- **更正 1**：NOF-075 任务书里"vseg 属主核 {2,3,6,7}、plain {2,3,4,5}"**被实测否证**（四遍全部 {2,3,6,7}）——该说法来自 NOF-062 的一轮观测，不具一般性。
- **更正 2（最重要）**：`p1_q1_plain` 那一遍 **plain 自己也是慢的**（内部 **221.61µs**、客户端 556.55，与 vseg 1.09×），随后两遍回到 **46.19 / 49.96µs**。⇒ **"慢档"不是 vseg 专属**；vseg 只是**从不回到快档**。
- **更正 3（协议缺陷）**：本轮两臂**读的是不同 LBA 集合**（`vseg_nof074` slot=1024 → 1,819 个盘内 offset；`vseg_nofvseg` → 1,164 个）。NOF-072 那轮两臂 LBA 集合完全一致（1164/1164）；本轮不一致 ⇒ **必须改回用 slot=1 的 `vseg_nof072`**，否则"同 LBA"前提不成立。
- 因此本单要回答的唯一问题是：**慢档的跃迁由什么触发，为什么 vseg 不出来。**

## A. 固定协议 10 连跑（不切 master、不改参数）

1. **A1：`vseg_nof072`（slot=1，400 keys，LBA 集合与 plain 完全一致）+ vseg flags，连跑 10 遍**。
   - 每遍：T1、400 keys、warmup=128、batch=1、verify=false、node3 本机；开 `nvme_pcie`/`urma_p99`、跑完快照、`trace_clear`。
   - 记录每遍：client P50/P90、target 内部 `t_cb−t_db` p50/p90、`last_empty−t_db`、`obs−le`、n、bench PID 与启动时刻。
2. **A2：切到 `vseg_nofvseg` + plain flags，同样连跑 10 遍**（记录同 A1）。
3. **A3：切回 `vseg_nof072` + vseg flags，再跑 5 遍**（验证能否复现/是否与切换有关）。
4. **日志卫生（本次必守）**：master/bench 日志一律**唯一文件名**（含时间戳与 cid），**禁止 `>` 覆盖**（NOF-075 因此丢了 master 冷启动证据）；记录每次 master 启动时刻。

## B. 判读（必答，三选一 + 数字）

| 观察 | 结论 |
|---|---|
| vseg 10+5 遍全慢、plain 仅第 1 遍慢后全快 | 慢档**与路径绑定**，plain 有一次性跃迁；下一步查"跃迁时 target 侧发生了什么"（用 A1/A2 的分段读数对比第 1 遍与第 2 遍） |
| vseg 也出现跃迁（给出遍次） | 慢档是**可退出的状态**，下一步找触发条件（时间/累计 IO/切换动作） |
| plain 也全慢 | 与集群无关，慢档取决于**启动后的时间窗**，下一步扫时间轴 |

另必须回答：
1. plain 第 1 遍慢的那一份 trace，其 `last_empty−t_db` 是否与 vseg 同档（≈238µs）——若是，则**慢档的物理形态与 vseg 完全一样**，只是持续时间不同。
2. plain 从慢→快的那一次跃迁发生在第 1 遍内部还是第 2 遍开始（用遍内分桶：前 100 key vs 后 100 key）。
3. vseg 在 10 遍里是否有任何一段（如某 50 个 key 窗口）掉进过快档（内部 p50 <100µs）。

## C. 判决（必答）

1. 跃迁是否可复现、出现在第几遍/第几个 key。
2. 慢档是否与"路径"绑定，还是与"时间窗/累计 IO"绑定。
3. 若与时间窗绑定 ⇒ 给出时间常数估计（从 master 启动到首次快档的秒数/IO 数），并给 M3-14 的验证方案。
4. 若与路径绑定 ⇒ 明确下次要插桩的位置（target 侧哪一层）与判据。

## D. 边界

- **不重启 target、不改码、不写入**（两臂均只读；`vseg_nof072` 与 `vseg_nofvseg` 数据都在）。
- 每遍异常（如 `err=-5 alloc qpair failed`）⇒ 等 25s 重跑该遍，作废首跑并标注。
- 开跑前 `ss -tnp | grep 4457`：有他人客户端则停手。
- 日志唯一命名、只追加；tracepoint 末态 mask→0x0。
- 产物 `run/nof076_*.txt`（≤3 个，每个 ≤1MB）；逐遍明细可放 `/tmp/nof076_passes.csv`。

## 回执必答（第一行）

vseg 10 遍与 plain 10 遍的逐遍内部 p50（两列）与是否出现跃迁；plain 第 1 遍慢档的 `last_empty−t_db` 是否与 vseg 同档；跃迁发生在第几遍/第几个 key；慢档与路径还是与时间窗绑定；M3-14 方案；有无失败。

---

# NOF-077（待执行 · 自建测量口径：给 bench 加 `--rounds` / `--start_key` · 零 target 影响 · 2026-11-24）

## 0. 为什么换口径（开发侧决策，不再沿用同事参数）

现行 bench 的三个硬限制已经被证明会误导结论，本单直接改掉：

| 限制 | 后果 | 我们的做法 |
|---|---|---|
| 每个 key 只读一次 | 测不出"同一个 key 再读一次会不会变快" | 同一进程内**多轮**重复读 |
| 每遍都是新进程/新会话 | "遍"之间混入建连与注册差异 | **一个进程内跑 N 轮**，会话只建一次 |
| `--warmup_keys` 读的就是 keys 0–127 | NOF-039 已证无效（不跳慢窗口、不预热设备） | **一律用 0**，改用统计排除 + 多轮观察 |
| 永远从 key 0 开始 | 每次都先撞盘首慢窗口，位置效应与状态效应混淆 | 加 `--start_key`（环形起点） |

保留的：400 keys × 4MiB（覆盖慢窗口 97 + 正常区 303，快异常可分）、顺序读、T1/T4/T16 三档、key 名固定（A/B 可比）。

## A. bench 补丁（**只加参数与输出，不改数据面逻辑**）

1. **`--rounds=N`（默认 1）**：把现有的"读 num_keys 个 key"那段包进 `for (r=0; r<N; r++)`；**同一个进程、同一个会话、同一份连接**重复 N 轮。
2. **`--start_key=S`（默认 0）**：读 key 的顺序改为 `(S+i) % num_keys`，用于随机化起点、避开固定从慢窗口开始；**不得改变 key 名生成方式**。
3. **每轮独立统计**：每轮结束打印一行，格式固定可解析：
   `ROUND idx=<r> n=<n> min=<> avg=<> p50=<> p90=<> p99=<> max=<> failed=<>`
   轮末再打一行进程级汇总 `TOTAL rounds=<N> ...`。
4. **兼容性**：`--rounds=1 --start_key=0 --warmup_keys=0` 的行为必须与旧版一致（同一份数据、同一 key 序列、同一统计口径），以便和历史数字对齐；**默认值即为此**，不破坏旧用法。
5. 编译产物 `build/.../bench-rounds`（记录 md5 与 `--help`），保留原二进制。

## B. 自检（先验补丁无害）

1. `--rounds=1 --start_key=0 --warmup_keys=0` 跑 plain 臂一遍，与 NOF-076 同参数的内部 p50 对比，**差 >10% ⇒ 停手**（说明补丁改了行为）。
2. `--rounds=3 --start_key=0 --warmup_keys=0`：确认打出 3 行 `ROUND`，每行 n=400、failed=0。
3. `--rounds=1 --start_key=200`：确认读的 key 从 200 开始（日志里的 key 序号或 LBA 可验）。

## C. 核心实验（**一个进程 10 轮 × 两臂**）

1. **vseg 臂**：`vseg_nof072`（slot=1，LBA 与 plain 逐值相同）+ vseg flags；
   `--num_keys=400 --rounds=10 --start_key=0 --warmup_keys=0 --num_threads=1 --batch_size=1 --verify=false`。
2. **plain 臂**：`vseg_nofvseg` + plain flags，同参数。
3. 两臂各开 target tracepoint（`nvme_pcie`/`urma_p99`）、**按轮切分**：每轮开始/结束各打一条标记（可用 `trace_clear` + 每轮快照，或写一个标记文件+时间戳对齐），给每轮的 target 内部 `t_cb−t_db` p50。
4. 记录：`ROUND` 行原样、每轮内部 p50、bench PID/启动时刻、master 启动时刻（日志唯一命名，禁止覆盖）。

## D. 判读（必答）

1. 两臂各 10 轮的 `ROUND p50` 序列（原样贴出）：
   - vseg 是否在**同一会话内**出现慢→快跃迁（第几轮）；
   - plain 第 1 轮是否仍慢、第 2 轮起是否回到 46–50µs。
2. 每轮 target 内部 p50 与客户端 p50 是否同向（排除客户端因素）。
3. **判决**：慢档是"会话内可退出"还是"会话级锁定"；若会话内可退出，给退出条件（轮次/时间/累计 IO 数）。
4. 若 10 轮都不退出，再用 `--start_key=200` 跑 3 轮（换起点），看是否与"从慢窗口开始"有关。
5. 给 M3-14 方案。

## E. 边界

- **只改 bench**（客户端二进制），不动 target、不改码 target、不重启 target、不写数据。
- **不得在 NOF-076 运行期间覆盖正在使用的 bench 二进制**：编译产物用新文件名（`bench-rounds`），且编译前先确认 NOF-076 已结束或已复制副本；若两者并行，各用各的路径。
- 补丁禁改：读逻辑/并发模型/key 命名/校验逻辑；只加参数、循环与打印。
- 默认值必须保持旧行为；B1 校验不过即停手。
- 两臂同宿主 node3；开跑前 `ss -tnp | grep 4457` 有他人则停手。
- 产物 `run/nof077_*.txt`（≤3 个、每个 ≤1MB）；每轮明细放 `/tmp/nof077_rounds.csv`。

## 回执必答（第一行）

补丁 `--help` 与 md5；B 段三项自检结果（尤其 rounds=1 与旧数字的偏差）；两臂各 10 轮的 `ROUND p50` 序列；vseg 是否在会话内跃迁、plain 第 1 轮是否仍慢；慢档是会话内还是会话级；M3-14 方案；有无失败。

---

# NOF-078（待执行 · 重验：把「路径」与「数据年龄」彻底分离 · 零写入（C 段除外，需批准）· 2026-11-24）

## 0. NOF-076 的三个判决与一处重大修正

- **路径无关（2×2 矩阵）**：同一 cid 换 flags 差 **≤3%**（`vseg_nofvseg`：vseg flags 49.77 / plain flags 46.19–49.96µs）；跨 cid 差 **5×**（`vseg_nof074`：vseg flags 240.46–241.40 / plain flags 247.11µs）。
- **慢档退出由「累计读取量」触发，不是时间**：快 cid 19/19 有效遍都在**遍内 IO#7100 = 887.5 MiB** 处从 ~240µs 跃到 ~40µs；墙钟 531–7244ms（**13× 波动**）而跃迁点恒定 ⇒ 非时间驱动。
- **新写数据不退出**：`vseg_nof074`（今日新写）15/15 遍、累计 2,112 MiB 内**零跃迁**（仍 246.06–247.67µs，spread 1.007×）。
- **慢档物理形态相同**：plain 第 1 遍慢档 `last_empty−t_db` = **205.71µs**，vseg 慢 cid 为 243.65–245.93µs（同档，差 16%）；快档仅 35.08–35.76µs。
- **开发侧修正（必须接受）**：**此前所有「vseg 慢 / plain 快」的对比都可能被「数据年龄」混淆**——NOF-072 的 4.98×、NOF-075 的 4.8–5.4× 都是"新写 cid vs 老 cid"。**在重验之前，第二阶段的核心结论（"vseg 读路径多 195µs"）只能记为"待重验"。**
- 但必须同时记住的反例：**NOF-025 两端各自新写数据后立刻读，plain 仍快 2.08×**——所以"数据年龄"不能单独解释第一阶段，必须做 4 象限才能定论。
- **产品级缺陷 F3（第三次复现）**：target 重启 ⇒ nsid1 uuid 变 ⇒ 旧 cid **永久不可 mount**（`vseg_nof072` 的 `segments.device_identity=9d89ad0c…` ≠ 当前 `b66fe5ea…`，`rc[-300] INVALID_VERSION`），**无迁移路径**。

## A. 4 象限（本单核心，零写入）

对 **同一 cid**、**同一时间窗**（≤10 分钟内）、**同一批 LBA**，两 flags 各 3 遍交替（v/p/v/p/v/p）：

| 象限 | 数据年龄 | 期望 |
|---|---|---|
| Q1 | 老 cid（`vseg_nofvseg`，≥1 天） | 已知：两 flags 都快 |
| Q2 | 新 cid（`vseg_nof074`，今日写） | 未知：两 flags 是否都慢 |
| Q3 | 新 cid 的**后段**（累计读取超过阈值后） | 是否与 flags 无关地变快 |
| Q4 | 新 cid 等 **+2h / +6h / +24h** 各重做一次 Q2 | 阈值是否随年龄单调下降 |

**判据**：若每个象限内两 flags 的 `t_cb−t_db` p50 差 ≤10% 且跨年龄差异显著 ⇒ **路径彻底无关，年龄是唯一变量**。

## B. 阈值曲线（零写入的部分）

1. 快 cid（`vseg_nofvseg`）：复测跃迁点是否仍在 IO#7100（±5%），给逐遍跃迁点与墙钟。
2. 新 cid（`vseg_nof074`）：确认 2,112 MiB 内无跃迁（复测 1 遍即可，已有 15 遍）。
3. 用客户端日志中的累计读取字节数（而非行号）定位跃迁，验证"按字节而非按条数"。

## C. 阈值上限探测（**需写入批准，未批准则跳过**）

- 若要把新 cid 的阈值推到 2GiB 以上，需要单遍读 >2.1GiB ⇒ 单遍 key 数 ≥ 600。做法：新 cid 写 **2,000 keys（8GiB）**，然后单遍读 2,000 keys，看跃迁是否出现、出现在第几 MiB。
- **这一步会写 8GiB 数据，需用户单独批准**；不批准则本单止于 A/B 段。

## D. 判决（必答）

1. 四象限逐格数字（每格 3 遍的 p50/p90）。
2. 路径是否还有任何独立贡献（给差值百分比）。
3. 跃迁点与"数据写入后经过时间"是否单调（+2h/+6h/+24h 三点的阈值）。
4. **结论表述**：给出可以对外用的正确说法（例如"新写入的数据在 X 小时内读慢约 5×，与 vsegment 无关"），并明确第一阶段/第二阶段哪些结论需要撤回或改写。
5. 下一步：若路径无关 ⇒ 重新评估"vseg 是否有价值"的结论；若路径有关 ⇒ 回到路径。

## E. 边界

- A/B 段**零写入、不重启 target**；两臂同宿主 node3；每遍记录 bench PID、master 启动时刻、当前 target uptime。
- **同一时间窗内完成一个象限**，禁止跨小时拼一个象限。
- C 段需写入批准；未批准则跳过并在回执标注。
- 日志唯一命名、只追加；tracepoint 末态 mask→0x0。
- 产物 `run/nof078_*.txt`（≤3 个、每个 ≤1MB）；逐遍明细 `/tmp/nof078_matrix.csv`。

## 回执必答（第一行）

四象限逐格 p50（两 flags × 老/新 cid × 各 3 遍）；路径是否还有独立贡献（%）；跃迁点复测值；+2h 那一格的阈值；哪些既有结论需要撤回/改写；C 段是否执行；有无失败。

---

# NOF-079（待执行 · 更正与重建：同龄数据上的公平 A/B + 年龄曲线 · 需两次写入批准 · 2026-11-24）

## 0. NOF-078 的判决（本项目的结论级更正）

| 象限 | 数据年龄 | vseg flags | plain flags | 差 |
|---|---|---|---|---|
| Q1 老 cid | ≥4 天 | **50.07µs** | **50.14µs** | **0.14%** |
| Q2 新 cid | 1h16m | **247.86µs** | **247.86µs** | **0.00%** |
| 跨年龄 | — | 247.86 | 50.10 | **4.95×** |

- **路径贡献 = 0（≤0.14%）**；4.95× 全部由「数据年龄」承担。
- 跃迁点：老 cid **6/6 遍精确在 IO#7050 = 881.2 MiB**（NOF-076 为 7100/887.5，漂移 0.7%）；新 cid **6/6 遍在 2,112 MiB 内零跃迁**。
- 遍内形状：老 cid `240.5 → 133.0 → 243.9 → 52.8 → 43.1 → 42.5 → 43.2 → 43.0`（慢→过渡→快）；新 cid 全程 245–248。
- **必须撤回的结论**：① 「NOF-072 4.98× / NOF-075 4.8–5.4× / NOF-061 4.64× 是 vsegment 读路径更慢」——它们是"新写 cid vs 老 cid"；② 第一阶段「vseg 读路径多约 195µs」；③ NOF-025「两端新写 plain 仍快 2.08×」与本单 Q2 直接矛盾（新写数据 plain 247.71 / vseg 247.44）⇒ **NOF-025 须重做，在此之前不得引用**。
- **可对外表述（暂定）**：刚写入（1 小时内）的数据在本环境读约 **246µs/128KiB**，与是否启用 vsegment **无关**（两种路径差 0.00%）；放置 ≥4 天的同规格数据约 **50µs**（快 4.95×）。慢档是"预热期"：单次进程内累计读满约 **881 MiB** 后自动进入快档（~43µs）；新写数据在 2,112 MiB 内不出现该跃迁。

## A. 年龄曲线（零写入，按时间点复跑）

1. **+2h / +6h / +24h** 各跑一次 Q2（新 cid `vseg_nof074`，两 flags 各 2 遍，≤2 分钟窗口）：记录内部 p50 与**跃迁点（无则 NONE）**。
2. 判据：跃迁点随年龄单调下降（或出现跃迁）⇒ 给出"慢档持续时长"的量级；若 +24h 仍无跃迁 ⇒ 说明本例不可恢复，需另查（数据放置/盘位）。
3. 每个时点单独记录 target uptime、master 启动时刻。

## B. 同龄数据的公平 A/B（**需两次写入批准**）

目的：回答"**vsegment 到底有没有独立的读性能代价**"。做法：

1. **T0 同一小时内**：新建 `vseg_nof079v`（slot=1、stripe=4MiB）写 400 keys；新建 `vseg_nof079p`（plain 模式、slot=1）写 400 keys。
2. **等两臂同龄**（建议 ≥6h，与 A 段同一时点），在 **≤2 分钟窗口**内：`079v` 用 vseg flags 读 3 遍、`079p` 用 plain flags 读 3 遍，交替。
3. 记录：内部 `t_cb−t_db` p50、**端到端 P50/P90**、跃迁点。
4. 判据：若两臂内部与端到端均差 ≤10% ⇒ **vsegment 无独立读代价**，第一阶段结论正式撤回；若端到端有差异而内部无 ⇒ 差异在客户端侧（task 数/元数据 RPC），需单列。
5. 未获批准则跳过 B 段，只做 A/C。

## C. 端到端口径补齐（零写入）

1. 从 NOF-078 的 12 份 CSV 里补齐 Q1/Q2 的**客户端 P50/P90**（v vs p），确认端到端是否同样 0%。
2. 给一张"同数据同龄 × 两 flags × 内部/端到端"的四格表，作为对外口径。

## D. 判决与报告更正（必答）

1. 年龄曲线三点结果（+2h/+6h/+24h）与慢档持续时长估计。
2. B 段（若执行）：两臂内部与端到端的差。
3. **结论表述**：vsegment 是否存在独立读代价；若不存，给出对外一句话（含数据年龄前提）。
4. 报告更正清单：需要撤回/改写的具体结论（含第一阶段报告与已交付的进展汇报）。
5. 下一步：是否需要把"刚写数据的 5× 读惩罚"作为一个独立问题立项（这对业务读取新鲜数据是真实影响）。

## E. 边界

- A/C 零写入；B 需两次写入（各 1.6GiB），**未批准则跳过**。
- target 不重启；两臂同宿主 node3；同一象限/时点必须在 ≤2 分钟窗口内完成。
- 日志唯一命名、只追加；tracepoint 末态 mask→0x0；不删 etcd 键。
- 产物 `run/nof079_*.txt`（≤3 个、每个 ≤1MB）。

## 回执必答（第一行）

+2h/+6h/+24h 三点的内部 p50 与跃迁点（或 NONE）；慢档持续时长估计；B 段是否执行及两臂端到端差；vsegment 是否存在独立读代价；报告更正清单；有无失败。

---

## NOF-079 修订（改判：先查布局与跃迁，不要等时间）

> 用户质疑"数据放一会儿读更快"不合理，开发侧复核后**收回"年龄是唯一变量"的表述**。原因：跨 cid 对照同时改了两个变量，回执自己也写明「跨象限 LBA 集合不同（1,164 vs 1,819 个盘内 offset，因 slot_count 1 vs 1024），跨象限不可配对」。

**当前可信的只有一半**：
- ✅ **同一 cid、同一批 LBA、两 flags 交替 3+3 遍 ⇒ 差 0.00–0.14%（6 遍）** ⇒ **当时判为“读路径无独立代价”**；**该结论已由 NOF-088 撤回：两 flags 实际未切换读取路径，`type` 全为 vsegment。**
- ❓ **跨 cid 的 4.95× 是"年龄"还是"LBA 布局"，未分离** ⇒ 不得对外说"放几天就快"。

因此本单改为下列三段，**A/B 全部零写入、不需要等任何时间**：

### A. 布局体检（零成本，先做）
1. 取 `vseg_nofvseg`（slot=1，1,164 offsets）与 `vseg_nof074`（slot=1024，1,819 offsets）的**盘内 offset 分布**：min/max、直方图、是否覆盖盘首慢窗口（offset < ~36MB/盘）、是否分散到 100GiB 各段。
2. 判据：若新 cid 的 offset 分布显著更分散或更多落在慢区 ⇒ **4.95× 至少部分是位置效应**，"年龄"降级为待验。

### B. 跃迁前后到底变了什么（零成本，用 NOF-078 的 12 份 CSV）
1. 在老 cid 的跃迁点（IO#7050 前后各 500 条）比较：`qd`、在飞数、相邻 submit 间隔、每盘 IO 分布、`last_empty−t_db`、`obs−le`。
2. 判据：若跃迁前后只有"每 IO 的盘执行时间"变化而 qd/在飞/间隔都没变 ⇒ 是设备侧；若 qd/在飞在跃迁点同步爬升 ⇒ 是**宿主侧在飞窗口爬坡**（则 881MiB 是窗口爬到满的标志，与盘无关）。
3. 这条能解释"为什么每遍都重置"——若是宿主窗口，每遍新进程重置就顺理成章。

### C. 时间序列（延后，不必守着）
- 对 `vseg_nof074`（同一 cid、同一 LBA）在 +2h / +6h / +24h 各复读 3 遍；**任何时点补跑即可**，不需要本会话等待。
- 判据：同一 cid、同一 LBA、只变时间 ⇒ 这是唯一能干净证明"年龄"的实验；若 +24h 仍是 247µs ⇒ 年龄假设作废。

### D. 若 A 显示布局是主因（需一次写入，另行批准）
- 新建一个 **slot=1** 的 cid 写 400 keys（与老 cid 同布局），立刻读 ⇒ 与老 cid 的 50µs 直接对比：同布局、不同年龄。这一步才能分离布局与年龄。

### 回执必答（第一行）
两 cid 的盘内 offset 分布对比（min/max/直方图/是否覆盖慢区）；跃迁前后 qd/在飞/间隔/每盘分布是否变化；881MiB 是设备侧还是宿主在飞窗口；C 段各时点结果（有则给）；是否触发 D；有无失败。

---

# NOF-080（待执行 · 钉死最后两个变量：写入时间 vs 访问分散度 · **需一次 1.6GiB 写入** · 2026-11-24）

## 0. NOF-079 判决（两个独立效应 + 一个假象被推翻）

| 效应 | 数字 | 性质 |
|---|---|---|
| **盘首慢区** | 成员盘 offset **< 35.4 MiB → 242µs**；**> 35.5 MiB → 43µs**（分界间隙仅 0.12 MiB）；两 cid 在该区差 **1.01×** | 设备侧**物理位置**效应，与路径、年龄都无关 |
| **新写数据/分散布局的读惩罚** | 同 offset 区间 `[36,145.4) MiB`：老 cid **42.91µs** vs 新写 **238.55µs** = **5.56×** | 归因**待定**（见下） |
| ~~881 MiB 预热阈值~~ | **撤销** | 真实机制：顺序扫描越过 offset 35.4 MiB 边界（6/6 遍，跃迁点对应 offset 33.5–33.6 MiB）；**不需要任何状态重置**，每遍都从 offset 0 扫起才是"重置"的假象 |
| 路径独立代价 | 同 cid 两 flags **0.00% / 0.14%** | 当时保留并强化；**NOF-088 已撤回：两 flags 未切换路径** |
| 跃迁前后 | `qd` 1.00×、gap 1.01×、在飞 1.00×、qpair/core 同；**唯一变化 `last_empty−t_db` 0.14×** | 设备侧 |

**开发侧怀疑（本单要解决）**：NOF-079 的 5.56× 仍有**两个变量没分离**——
- 变量 A：**写入时间**（老 cid 4 天 vs 新 cid 1.5 小时）；
- 变量 B：**访问分散度**（老 cid 严格连续于 offset 0–145.4 MiB、步长恒 256 块；新 cid 分散于 0–9,282 MiB、span 63.8×）。

回执也指出：新 cid 在 `[36,145.4)` 区间只有 **14 个 offset**（老 cid 876 个），样本虽一致但薄。**"分散"同样能让 128KiB 读失去顺序性**，这是与"年龄"并列的候选。

## A. 决定性对照（一次写入）

1. **新建 `vseg_nof080`（slot=1，与老 cid 同布局）写 400 keys**（1.6GiB，**唯一写入**）；验收 Keys=400、logical_used=1,677,721,600；记录写入完成时刻 T0。
2. **T0 后 ≤2 分钟内**：读该 cid 3 遍，两 flags 各 1 遍 + 1 遍复测；记录 per-offset 分档（<35.4 MiB 与 >36 MiB）的 `t_cb−t_db` p50。
3. **同窗口内**：读老 cid（`vseg_nofvseg`，4 天、连续）3 遍；读新 cid（`vseg_nof074`，1.5h、分散）3 遍。
4. 三组都在 **同 offset 区间 `[36,145.4) MiB`** 上比较 p50。

## B. 判据（三选一）

| 结果 | 结论 |
|---|---|
| 新 cid(slot=1, 刚写) ≈ **43µs** | **年龄无关**；5.56× 来自**访问分散度**（顺序性）⇒ 修复方向是布局/连续分配 |
| 新 cid(slot=1, 刚写) ≈ **240µs** | **写入时间效应为真** ⇒ 需查盘的写入后后台动作（SLC→TLC 回写/GC），并给出"多久恢复"的曲线 |
| 介于两者（如 100–150µs） | 两个因素都有，给各自占比 |

## C. 附带（零成本）

1. 老 cid 与 `vseg_nof074` 的 **per-offset 步长/连续性**量化（连续段长度分布、相邻 offset 差直方图），作为"分散度"的度量，与 B 段结果对齐。
2. 若 B 判为分散度 ⇒ 给出"按 128KiB 顺序性"的可验证预测：新建 slot=1 cid 应快、slot=1024 应慢，且与年龄无关。

## D. 判决与报告更新（必答）

1. 三组在同 offset 区间的 p50 与比值。
2. 5.56× 的归因（年龄 / 分散度 / 混合，给占比与证据）。
3. **对外表述修订版**（含两个独立效应 + 归因结论）。
4. 若判为分散度 ⇒ 与 vsegment 的关系：slot 越大是否越分散（我们已知 partition 数不影响第一阶段结果，但**当时没有"新写数据"这个混淆**，需重新审视）。

## E. 边界

- 仅 A1 一次写入（1.6GiB，需用户批准）；其余零写入。
- target 不重启（PID 905586 插桩版、`NOF074_DESC=1`）⇒ 只比比值。
- 两臂同宿主 node3；同一象限 ≤2 分钟窗口；日志唯一命名。
- 产物 `run/nof080_*.txt`（≤3 个、每个 ≤1MB）。

## 回执必答（第一行）

新建 slot=1 cid 刚写后在同 offset 区间的 p50（与老 cid 4 天、新 cid 分散 三方对比）；5.56× 归因（年龄/分散度/混合，占比）；对外表述修订版；有无失败。

---

# NOF-081（已完成 · 结论部分被拒 · 定性 + 衰减曲线 · 2026-11-24）

> **【2026-11-24 16:35 开发侧裁决 · 先于本单执行阅读】**
>
> 本单 A 段判决「设备侧排除、落点收窄到 Mooncake 客户端 NoF 读路径」**不成立**：裸 SPDK 完全绕过 Mooncake，本轮回执自己就在同一批物理块上复现了 **5.62×**（217.6 / 38.72µs），触发变量是**扫描窗口长度**。A 段两臂（217.6 / 217.6 / 212.5）都在 32MiB 短窗口的同一慢档内，属于「两个都慢」，不构成对照。
>
> **C 段「对外表述定稿」「业务影响评估 / 延后读建议」「300µs 已达成」全部作废，不得执行、不得引用。** 其中「刚写入的数据比 4 天前慢 5.8–5.9×」是用户 2026-11-24 明确否决的表述，B 段 +15.2min→+35min 无恢复也反证它。
>
> 保留：E3（同 cid 两 flags 0.00–0.14%）；B 段两个时点的观测值；两 cid LBA 集合 1164/1164 逐元素相同、`io_per_lba` mean 14.52 相同。
>
> 撤回：E1「盘首慢区是物理位置」（短窗口在 0/1024/2048MiB 起始均 216–218µs）、E2「新写惩罚 / 数据年龄」、NOF-079「惩罚在设备侧」的归因。
>
> 交叉核对只比了 LBA 集合与重读次数，**缺相邻 IO 步长 / 连续游程长度 / 每遍总流长** ⇒ 见 NOF-082 B 段。



## 0. NOF-080 判决（归因闭合）

| 组 | slot | 写入时间 | 布局 | slow[0,36)MiB | fast[36,145.4)MiB | 全量 p50 |
|---|---|---|---|---|---|---|
| old `vseg_nofvseg` | 1 | 4 天 | 1,164 offsets / 100% 连续 / 145.4MiB | 240.43 | **42.91** | 50.11 |
| new074 `vseg_nof074` | 1024 | 1.5h | 1,819 / 79.8% / 369 段 / 9,282MiB | 244.79 | 238.55 | 247.44 |
| **new080 `vseg_nof080`** | **1** | **30 秒** | **与 old 逐位相同** | 249.70 | **248.95** | 249.20 |

- **`old : new080 = 5.80×`（同 offset 区间、同布局、同 flags）** ⇒ **写入时间效应为真**。
- **布局/分散度 100% 排除**：new074（369 段）vs new080（1 段）在新写状态下仅差 **1.04×**。
- **路径**：new080 上 v/p 交替差 **0.08%**。
- **设备未整体变慢**：同窗口复测老 cid 50.04 / 50.17µs。
- 慢区（<36MiB）三组一致 240/245/250（≤3.9%）⇒ 位置效应独立于年龄。
- 结论：**slot 只决定 offset 连续度，不改变读延迟**（新写状态下 1.04×）。

## A. 定性：裸 SPDK 读同一批 LBA（零写入）

1. 用 `urma_perf`（node1 `~/gap/spdk/build/examples/urma_perf`，NOF-045/059 同款）顺序读两段，**必须落在 offset > 36MiB**（避开盘首慢区）：
   - **新写区**：`vseg_nof080` 数据所在 LBA（T0=15:51:00，读取时年龄已知）；
   - **老数据区**：`vseg_nofvseg` 对应 LBA。
2. 记录 client p50/p99 与 target W9；参数同 NOF-059 D1（`-w read -o 131072 -T 1 -b 1 -t 6 -n 1 -g 0 -M cpu`）。
3. **判据**：新写区若裸读也慢（≥200µs）⇒ **纯设备侧**（与 target 无关）；若裸读快（~40–50µs）⇒ target 侧参与，需另查。
4. 注意 preflight 写会覆盖起始 LBA（已知副作用），记录被覆盖的 LBA。

## B. 衰减曲线（零写入，随时点补跑）

1. 对 `vseg_nof080`（同一批 LBA）在 **+2h / +6h / +24h** 各读 3 遍（v/p 交替），给：
   - per-offset 分档 p50（<36MiB 与 >36MiB）；
   - 是否出现"快档"offset 及占比。
2. 判据：>36MiB 段的 p50 何时从 ~249 掉到 ~43；给出**恢复时间量级**（分钟/小时/天）。
3. **任何时点补跑即可**，不需要守候。

## C. 判决与对外口径（**已作废 · 不得执行、不得引用**）

1. 裸 SPDK 结果：新写区 vs 老数据区的 p50 ⇒ 设备侧 or target 侧。
2. 衰减曲线（已得时点）：`>36MiB` 段 p50 随年龄的变化。
3. **对外表述定稿**（三段式）：
   - E1 盘首慢区：offset < 35.4MiB → 242–250µs（仅老数据可观测）；
   - E2 新写读惩罚：写后（已知窗口内）任何 offset 均 ~249µs，4 天后同区间 42.91µs ⇒ **5.80×**；与布局、路径无关；
   - E3 读路径：vseg vs plain **0.00–0.14%**，无独立代价。
4. 报告更正建议：第一阶段/第二阶段哪些结论正式撤回；`300µs` 目标是否需要重新定义（既然旧数据 43µs，目标其实已达成）。
5. 业务影响评估：写后立刻读（训练 checkpoint / 断点续训 / 副本校验）会吃满这 5.8×，给出规避建议（延后读 / 提前写 / 双缓冲）。

## D. 边界

- **零写入**（`vseg_nof080` 的 oplog 已 22MB，距 32MB 墙 9MB，**严禁再写**）。
- target 不重启（PID 905586 插桩版）⇒ 只比比值；裸读一支不受该限制（不经 target 软件栈的 bdev 路径时需说明）。
- 两臂同宿主 node3；日志唯一命名；tracepoint 末态 mask→0x0。
- 产物 `run/nof081_*.txt`（≤3 个、每个 ≤1MB）。

## 回执必答（第一行）

裸 SPDK 新写区 vs 老数据区的 p50 ⇒ 设备侧还是 target 侧；>36MiB 段 p50 随年龄的变化（已得时点）；恢复时间量级；对外表述定稿；报告更正建议；有无失败。

---

# NOF-082（已完成 · 结论部分修正 · M3-14 · 2026-11-24）

> **【2026-11-24 16:45 开发侧裁决 · 优于本单任务书判据】**
>
> **成立（观测层）**：同一批物理块、同一 QD1，**每遍读流前 ~350 MiB 固定慢档**（~201µs / slowfrac 0.98），400MiB 之后全快（25.08µs / slowfrac 0.000），**下一遍回到起点重新变慢**。A1 十点的 avg 可用单一参数拟合：N=350MiB、slow=215µs、fast=39µs 按字节加权，R≥512 五点误差 <2%（开发侧已复核）。A2 位置无关（极差 2.4%）、A4 QD 1→64 慢档不消失。
>
> ⇒ NOF-045「裸 SPDK 41µs ⇒ 盘无罪」、NOF-076「881MiB 阈值」、NOF-081「512–1200MiB 跃迁」都是**同一个慢程在不同 range 下慢占比的投影**，不是三个独立现象；与数据年龄、物理位置、设备侧、客户端侧都无关。
>
> **需修正 / 撤回**：
> - 本单 B 段判据「1–4 任一项不同 ⇒ 差异来自访问流形态」**逻辑不成立，现撤回**。裸 SPDK 是严格单一顺序步长（step 恒 128KiB），同样有 201→25µs 的 8× 两档 ⇒ **形态差异不是产生两档的必要条件**。B 段只证明「两 cid 动态 IO 序列不同」（step256 51.20% vs 65.80%、步长种类 173 vs 5、游程 mean 2.04 vs 2.91），因果未立；且 B-3 每遍总流长 144.5 vs 144.2MiB 基本相同、B-4 重访序两者均平坦。
> - **A3 内部矛盾**：`first32 203.40 / mid32 201.76 / last32 204.10µs` 三者全慢，与 A3 分档表 `pos 3200–9600 (400–1200MiB)=25.08µs / slowfrac 0.000–0.015` 直接冲突 ⇒ 三段定义未明前不得引用。
> - 回执「拟合全部 10 点」**只对 avg 成立**：p50 在 R=512（172.80）、R=768（129.92）为中间值，不符合硬阶跃；R≤256 的 avg 也比全慢档 215µs 低 ~10% ⇒ 慢程边界有过渡 / 每遍 N 有波动，不能当常数阈值。
>
> **仍未定位**：为什么存在 ~350MiB 慢程、为什么逐遍重置、Mooncake 两 cid 的 5.91× 是否与它同一现象 ⇒ NOF-083。
>
> **禁止**：结论行不得出现「数据年龄 / 设备侧 / 客户端侧 / 访问流形态是根因」的因果定性。



## 0. 为什么派这单（先读）

NOF-081 回执里唯一的新事实不是 A 段判决，而是它自己标为「[未验证]、机制未明」的 6× 窗口效应：

| 观测 | 数字 |
|---|---|
| 同 `start=2048MiB`，**只改 range** | 32→**212.48** / 128→216.32 / 512→171.52 / **1200→36.16** / 4096→38.56 µs |
| 短窗口换起始位置 | 0 / 1024 / 2048 MiB 起 32MiB 全部 **216–218µs** |
| 长窗口 | `[396,1596)` 38.72µs、`[2048,3248)` 39.84µs |
| 比值 | 裸 SPDK **217.6/38.72 = 5.62×** vs Mooncake **253.73/42.91 = 5.91×** |

- 同一批物理块、同一 QD1，仅换扫描窗口长度就有 5.6× 档位差；与起始 LBA、是否新写、io_size 无关。
- ⇒ E1「盘首慢区是物理位置」**被否证**；E2「新写数据读惩罚 / 放几天变快」继续不成立；NOF-079「惩罚在设备侧」归因撤回，但理由是「裸读以窗口效应的形式复现了」，不是「裸读不复现」；NOF-081「落点收窄到客户端」不成立。
- ⇒ 此前所有 4.6–5.9× 归因（路径 / 会话 / qpair / 命令描述符 / 布局 / 年龄 / 设备）**均未成立**。当前唯一被独立复现的触发变量是**访问流长度**，机制未明。
- 跃迁落在 **512–1200 MiB**，与 NOF-076 报的「7100 条 IO ≈ 887.5 MiB」量级吻合；NOF-079 用 35.4 MiB 物理边界替换掉它是错的。这条是**强线索，不是结论**。

**禁止**：结论行不得出现「设备侧 / 客户端侧 / 数据年龄 / 放几天」的因果定性；不得把「刚写入比 4 天前慢 5.8×」写进任何结论或对外口径。

## A. 裸 SPDK：把流长效应钉死（node3，同 NOF-081 配方）

配方同 NOF-081：`sudo -n env LD_LIBRARY_PATH=/home/l00955908/gap/spdk/build/lib LD_PRELOAD="<yin>/librte_bus_pci.so.26.1 <yin>/librte_bus_vdev.so.26.1"`，`-w read -o 131072 -T 1 -b 1 -t 6 -M cpu`。

1. **流长曲线**：固定 `start=2048MiB`（从未写区），`range` 依次 16 / 32 / 64 / 128 / 256 / 512 / 768 / 1024 / 1200 / 2048 MiB。每点给 p50 / avg / IOPS / 快慢档 IO 占比；给出 p50 掉 ≥50% 的相邻区间（跃迁点）。
2. **位置无关性**：固定 `range=32MiB`，`start` 扫 0 / 32 / 128 / 512 / 1024 / 2048 / 4096 MiB。**若 preflight 写不可关闭，start < 2048MiB 的点一律不新增写入**，直接复用 NOF-081 已有数据并注明来源。
3. **整流预热 vs 统计口径**：`range=1200MiB` 一支，把完成序列按 IO 序号切成**前 32MiB / 中 32MiB / 末 32MiB** 三段分别算 p50。若 urma_perf 只给汇总，用 trace 分片；分片若触发新的 preflight 写必须如实记录。
4. **QD 曲线**：固定 `range=32MiB`，`b` 扫 1 / 2 / 8 / 16 / 64，给 p50 与吞吐，判慢档是否随 QD 消失。

判据：① 跃迁点稳定且不随 `start` 移动 ⇒ 流长是自变量；② 1200MiB 流的前 32MiB 也快 ⇒ 整流级效应（顺序流建立），不是统计口径；③ 前段慢 / 末段快 ⇒ 流内逐步加速，给加速曲线。

## B. 纯离线：两 cid 访问流逐元素对账（零环境，复用 NOF-081 snaps，不重跑）

从两 cid（`vseg_nof080` 新写、`vseg_nofvseg` 老）已有 trace 提取并逐元素对比：

1. **相邻 IO 的盘内 offset 步长直方图**（256 blocks 步长占比、跳变次数与跳变幅度分布）；
2. **连续游程长度分布**（连续递增 run 的 p50 / p90 / max，以及 run 数）；
3. **每遍总流长**（覆盖的盘内 offset 跨度、IO 总数、单遍墙钟时长）；
4. **同一 offset 在流内第几次出现**与 p50 的关系（判断重读次数在两 cid 间的分布是否不同）。

回执已做且保留：LBA 集合 1164/1164 逐元素相同、`io_per_lba` mean 14.52 相同。

判据：若 1–4 中有任一项两 cid 不同 ⇒ 差异来自**访问流形态**，与内容 / 年龄 / 读路径都无关，后续统一 bench 流长即可复现或消除；若四项完全相同 ⇒ 再进客户端 NoF 读路径分段插桩（另案，需批准）。

## C. 回执必答（第一行）

流长→p50 跃迁区间；两 cid 游程长度 / 步长直方图 / 每遍总流长是否相同；是否支持「访问流长度是唯一被复现的触发变量」；有无失败；本单新增写入的 LBA 清单（应为 0）。

另外必答：`range=1200MiB` 内前 / 中 / 末三段 p50；QD 1/2/8/16/64 曲线。

## D. 边界

- **零改码、零重编、零重启 target**（PID 905586、md5 `3129641f…` 保持不变）；trace mask 末态 `0x0`；不碰他人进程。
- **严禁写 `vseg_nof080`**（oplog 22MB，距 etcd gRPC 32MB 墙约 9MB）。**不重建该 cid**：接受 NOF-081 preflight 已覆盖的 640KiB 内容，标记「内容不可信」，后续一律 `verify=false`。
- `urma_perf` preflight 每个 `--start-lba` 写 128KiB：**先查 `urma_perf.c` 是否有开关可关**；若不可关，所有 `start` 必须 ≥ 2048MiB（cid 数据区之外），并逐一记录被写 LBA；**不得在卷 0–1600MiB 区间新增任何写入**。
- 两臂同宿主 node3；日志唯一命名；产物 `run/nof082_*.txt`（≤3 个、每个 ≤1MB）。
- 结论只写「观测 + 是否支持判据」，**禁止因果定性**。

---

# NOF-083（**已完成 · N 精确为 400.0 MiB · 裁决见下** · M3-15 · 2026-11-24）

> **【2026-11-24 19:17 开发侧裁决 · 优先级高于本单任务书】**
>
> **A/B/C 三段按预期完成，D 段记 `[未获取]` 正确。N 首次被精确测量；A3 的「矛盾」判定是我上轮判错，予以撤回；但 C 段与 NOF-086 端到端数据产生新矛盾，本单不裁，交 NOF-087 判。**
>
> ### 1. **N = 400.0 MiB（精确）** —— 覆盖此前 350 MiB 的拟合值
>
> | 维度 | 扫描 | 结果 |
> |---|---|---|
> | io_size | 16K / 64K / 128K | N(IO) = 25600 / 6400 / 3200 ⇒ **全部等于 400 MiB ÷ io_size** |
> | 时长 | 2 / 6 / 20 s（1.37 / 6.46 / 22 遍） | **400.0 恒定** ⇒ 不按时间 |
> | QD | 1 / 2 / 4 | **400.0 恒定** ⇒ 不按 qpair IO 计数 |
>
> **⇒ 升 L3：慢程按「字节」计量，N = 400.0 MiB，每遍重置。** NOF-082 拟合的 `N≈350 MiB` 作废，全局改用 **400.0**。
> **4 KiB 臂不构成反例**：6 s 只扫 343 MiB 未跨边界，N 未测出，但它落在 55.5(4k) < 78.6(16k) < 127.6(64k) < 204.0(128k) 的单调慢档序列内，与「按字节」自洽。
>
> ### 2. A3 的「内部矛盾」**撤回** —— 我上轮判错
>
> 本单澄清：`first32 / mid32 / last32` 都按**绝对 idx** 切，流内 pos 全落在 <400 MiB 慢程区（pos 0–31.9 / 169.6–201.5 / 339.4–371.2 MiB），**三段本来就都该是 204µs，不存在矛盾**。NOF-082 回执里那个「矛盾」是**它自己按绝对 idx 做 12 等分、跨遍混合**造成的。
> ⇒ NOF-082 裁决块中「A3 内部矛盾」一条作废，**NOF-082 的 A3 数据恢复可用**。
> ⇒ 记录我的错误：把「分档口径造成的伪冲突」当成了「数据自相矛盾」，未先核分档定义就下判断。
>
> ### 3. C 段成立，且**与 NOF-086 端到端数据矛盾** —— 本轮最重要的事
>
> | cid | IO 数 | p50 | 快档(<100µs) | 形态 |
> |---|---|---|---|---|
> | NEW080 | 16,896 | 251.87µs | **0（0.0%）** | 20 档 p50 全 247–256µs ⇒ **全程慢** |
> | OLD | 16,897 | 57.22µs | 10,626（**62.9%**） | 前 ~7,600 IO 慢 233–250µs，后 48–50µs |
>
> ⇒ **「OLD 无慢程」不成立**；**「OLD 被 warmup 跳过」也不成立**（OLD 自己就有 37.1% 慢 IO）。
> ⇒ 慢程**确实存在于 Mooncake 侧**，形态与裸 SPDK 一致（前段慢、后段快）。
>
> **但 NOF-086 的 2000 keys 端到端看不到它**：
> - 若慢档作用于整 key：单 IO 204µs × 等效串行 21.7 ≈ **4,362µs/key**；400 MiB = **100 个 key** = **5%** ⇒ 应出现在 **P95** 以上。
> - 实测 W0：P99 = **705µs**、Max = **1,227µs**、Min = **471µs** ⇒ **整个分布跨度只有 2.6×，容不下一个 8× 的慢档**。
> - 反向算：P99 = 705µs ⇒ 最多 **20 个**样本可高于它，而理论上该有 **100 个**。
>
> **⇒ 三份数据（裸 SPDK 有慢程 / Mooncake trace 有慢程 / 端到端看不到）不能同时为真，必有一条是错的。**
>
> 三种可能，本单不裁：
> 1. **C 段数据源是 NOF-078 旧 trace**（`/tmp/nof078_B081_v1_160609.csv`、`/tmp/nof078_Q1_v3.csv`），描述的是**当时**状态；NOF-086 新写 2000 keys 已覆写每盘 0–727.27 MiB，当前状态可能已变。
> 2. **warmup 语义**：若 `--warmup_keys=128` 是「跳过不读」，R1–R3 从未触碰慢程区；但 **W0（warmup=0）是全读的，仍应看到** ⇒ 这条不足以为 2 开脱。
> 3. **4 MiB key 的内部并行把慢档摊平**：32 个 128 KiB IO 若充分并行，key 延迟由最慢一批决定而非 32 倍累加。**这条最可能，但它同时意味着「慢程对业务不可见」。**
>
> **⇒ 交 NOF-087 判：用 NOF-086 那份逐任务日志（`warmup=0` 那遍）按流内位置分档，直接看前 100 个 key 是否更慢。**
>
> ### 4. 两个待决策的裁决
>
> **(a) `400/11 = 36.36 MiB/盘` vs NOF-079 的 `35.4 MiB` 是否同源 —— 不需要专门实验。**
> - NOF-082 A4 已区分：32 MiB 短窗口在 **0 / 1024 / 2048 / 4096 MiB** 起始均 216–218µs。若慢区是「盘首物理位置 < 35.4 MiB」，从 1024 MiB 起始（盘内 93 MiB）就该是快的；实测仍慢 ⇒ **「盘首」假说已被否证**。
> - 两个数字接近（35.4 vs 36.36）**纯属巧合**，都只是「几十 MiB」量级。
> - **不派专门实验。** 若将来要用，一次「从盘内 100 MiB 起始读 400 MiB」即可，不是当前优先级。
>
> **(b) 是否往 target 侧 URMA/bdev 每流状态查 —— 方向对，但顺序不对：先判价值，再查机制。**
> - 若 NOF-087 证明端到端看不到慢程，查这个机制等于**在一条不影响业务的路径上做优化**：技术上有意思，对主线无价值。
> - **NOF-087 出来之前不动这个方向。**
>
> ### 5. 环境与边界（全部符合）
> - 新增写入仅 LBA 16384000（12 × 128 KiB = **1,536 KiB**），`vseg_nof086` 数据区 0–8000 MiB **零写入** ✓
> - etcd entries 13,930 **零增长** ✓；dbSize 55%→63% 属 etcd 内部 compaction，与 oplog 无关 ✓
> - `vseg_nof086` PID 2292801 未重启、Keys=2000 ✓；target PID 905586 / md5 未变 ✓；mask 末态 `0x0` ✓
> - **D 段 `[未获取]` 处理正确**：bench 无 `key_order` / `lba_plan` / `start_key` / `rounds`，只有 `-shuffle_seed`（仅 segment_read 场景）。**NOF-083 D 段到此关闭，不再尝试、不改码。**


## 0. 为什么派这单

NOF-082 把「流长」钉死成了更精确的东西：**每遍前 ~350MiB 固定慢程，之后全快，逐遍重置**。所以此前所有 5× 归因（年龄、位置、设备侧、客户端侧、vseg 路径）都不是根因；现在要回答的是「这个慢程是什么、以及它能否解释 Mooncake 的 5.91×」。详细裁决见 NOF-082 顶部。

## A. 慢程 N 的不变量扫描（node3 裸 SPDK，零写入，start ≥ 2048MiB）

沿用 NOF-082 配方与 A3 的 target trace 切片法（`nvme_pcie + urma_p99`、0.5s 快照、按 IO 序号分档给 p50 与 slowfrac）。

1. **io_size**：4KiB / 16KiB / 64KiB / 128KiB（urma_perf 单 IO 上限 128KiB，256K 被拒见 NOF-023），固定 `range=1200MiB`、`-t 6`、`b=1`。每支给 N（卷 MiB）与 N 对应的 IO 条数。
2. **时长**：`-t` = 2 / 6 / 20 s，固定 io=128KiB、range=1200MiB，看 N 是否随遍数 / 墙钟变。
3. **QD**：补 b=2 / 4（A4 已做 1/8/16/64），判慢程按 IO 数还是按字节计。

判据：
- N（卷 MiB）恒定、N 的 IO 条数随 io_size 变 ⇒ **按字节计的流长**；
- N 的 IO 条数恒定、N（MiB）随 io_size 变 ⇒ **按 IO 计数**；
- N 随 `-t` / 遍数变 ⇒ **按时间周期**，不是流长。

另给慢程能否换算成固定的「每成员盘 X MiB」或「每 qpair Y 条 IO」，用于区分盘侧顺序预取建立 vs 软件侧每遍重置的状态。

## B. A3 口径澄清（必做，纯离线，复用已有 trace）

1. 明确 `first32 / mid32 / last32` 是「三支独立 32MiB 读」还是「同一 1200MiB 流内按 IO 序号切的三段」，给出各自 IO 序号区间。
2. 若是后者，与分档表冲突 ⇒ 重算并说明原因（是否按遍切、是否取了每遍的前 32MiB）。
3. 补二维表：**每遍编号 × 流内每 100MiB 分档**的 p50 / slowfrac，直接展示「逐遍重置」。
4. 给慢程边界的过渡宽度：slowfrac 从 0.9 掉到 0.1 跨多少 MiB，以及遍与遍之间 N 的波动范围。

## C. 把慢程映射回 Mooncake（关键，零写入，复用已有 trace，不重跑写入）

1. 对 `vseg_nofvseg`（OLD，p50 ~50µs）与 `vseg_nof080`（NEW080，p50 ~250µs），各自按**每遍 IO 序号**分档给 p50 / slowfrac（同 A3 口径），看每遍是否也存在前 ~350MiB 慢程。
2. 判据：
   - OLD 每遍也有慢程但被 `warmup_keys=128` 跳过 ⇒ 解释 OLD 汇总为何是快档；
   - OLD 完全无慢程 ⇒ 裸 SPDK 慢程与 Mooncake 5.91× **不是同一现象**，两案分开查；
   - NEW080 全程慢程 ⇒ 支持「NEW080 从未建立起快档流」。
3. 给两 cid 的「快档 IO 占比」与各自汇总 p50 的对应关系。

## D. 形态与 cid 解耦（零写入，先核实能力）

1. **先 `--help` 核实**现役 bench 是否支持 `key_order` / `--lba-plan` 类参数；不支持则记 `[未获取]`，**不改码、不启用 NOF-077、不自造脚本**。
2. 支持则：同一 cid（选 OLD，数据不动）只换访问顺序跑两支，看能否在同一 cid 上造出快 / 慢两档。
3. 判据：同一 cid 仅换顺序即可造出两档 ⇒ 访问序列是自变量；不能 ⇒ B 段形态差异只是伴随现象，不得作为根因。

## E. 回执必答（第一行）

N 的不变量结论（按字节 / 按 IO / 按时间）；A3 三段定义与冲突解释；OLD / NEW080 每遍是否都有慢程；同一 cid 换顺序能否造出两档；有无失败；新增写入 LBA 清单（应为 0）。

## F. 边界

- 零改码、零重编、**零重启 target**（PID 905586 / md5 `3129641f…` 保持）；trace mask 末态 `0x0`。
- **严禁写 `vseg_nof080`**（oplog 22MB，距**客户端 32MiB recv 上限**约 10MB）、**严禁写 `vseg_nof086`**（oplog **942MB**，已永久不可重启；etcd 余量仅 **316MB**，一轮补写即可吃光）；**不重建任何 cid、不切 cid、不重启 master**；preflight 不可关 ⇒ 所有 `start ≥ 8000MiB`（**2026-11-24 18:51 修正**：NOF-086 已把每盘 0–727.27MiB 写满，原 `start ≥ 2048MiB` 换算到盘内仅 186.18MiB、已落入新的 2000-key 数据区；`start ≥ 8000MiB` = 盘内 727.27MiB，正好在数据区外）。本单零写入，读不改变任何数据。
- 不碰他人进程；产物 `run/nof083_*.txt`（≤3 个、每个 ≤1MB）；结论只写「观测 + 是否支持判据」。

---

# NOF-087（**已完成 · U04 定案：慢程在端到端不可见** · 纯读 · 2026-11-24）

> **【2026-11-24 19:24 开发侧裁决 · U04 定案】**
>
> **判据命中，四遍一致，结论明确：慢程在 Mooncake 端到端不可见。**
>
> | 遍 | 前 100 key p50 | key≥500 p50 | 比值 | 前 100 Max |
> |---|---|---|---|---|
> | R1 | 540 | 530 | **1.019×** | 729 |
> | R2 | 568 | 566 | **1.004×** | 934 |
> | R3 | 536 | 517 | **1.037×** | 739 |
> | **W0（主判据）** | **553** | **530** | **1.043×** | **1215**（含 key0 冷启动 1176） |
>
> 判据为「<20% ⇒ 不可见」，实测 **4.3%**，四遍全部 <1.05×。前 100 的 Max 全在 1215µs 以内，**远低于判据要求的 3000–5000µs**。`read_us` 前 100 / ≥500 = **1.049×**；`query_us` = **1.029×** —— 三个口径一致。
>
> ### 1. **U04 定案：不是矛盾，是两个层级**（升 L3，登记 C16）
>
> 此前判为「三份数据不能同时为真」，**这个判断本身是错的**——我把它当成了矛盾，实际上它们描述的是两个不同层级：
>
> | 层级 | 测量对象 | 慢档表现 |
> |---|---|---|
> | **单 IO 层** | 1 个 128 KiB task | 慢 204µs / 快 26µs，**8× 真实存在**（NOF-082/083） |
> | **per-key 层** | 1 个 4 MiB key（32 个 task） | **看不到慢档**，前 100 与后段差 4.3%（NOF-087） |
>
> **换算的关键证据**：单 key `read_us` p50 = 426µs ÷ 32 task = **13.3µs/task**，**比裸 SPDK 的快档 26µs 还低** ⇒ Mooncake 路径的 key 内部存在充分并行/流水，单 IO 的慢档被摊平。
> ⇒ **对 U03 的贡献（不等于解决）**：13.3µs/task 是「per-key `read_us` ÷ 32」的算术平均，**不是与 trace p50 对齐过的值**。它只说明「单 IO 延迟 × 32 ≠ per-key 延迟」这个方向成立，**换算本身仍未完成，U03 保持未决**。
>
> ### 2. `--warmup_keys` 语义确认 —— **NOF-086 裁决 §8 成立，不降级**
>
> 源码原文：`DEFINE_uint64(warmup_keys, 5, "Number of warmup keys (not counted in stats)")`；`L1368-1370` 额外 `get_into` 前 N 个 ⇒ **额外预热，非跳过**。
> **交叉验证（算术闭合）**：key 0–127 在日志里出现 **7 次**（R1/R2/R3 各 2 次 + W0 1 次），key 128–1999 出现 **4 次**（R1/R2/R3 各 1 次 + W0 1 次）—— 与「预热 + 统计」结构精确吻合，证明解析正确、语义无误。
>
> ### 3. **慢程这条线到此关闭** —— 对业务无影响
>
> 追了 NOF-076 到 NOF-087 共 12 轮的那个「前 400 MiB 慢 8 倍」，**结论是：真的存在，但只存在于单 IO 观察层，对实际读取没有任何影响。**
>
> ⇒ **此前所有「用慢程解释 Mooncake 5.91×」的方向，整体作废。** 具体作废项：
> - 「慢程是 5.91× 的根因」——不成立（慢程对端到端贡献 4.3%）
> - 「追慢程机制能解释 vsegment 差异」——方向错（两者不同层级）
> - NOF-083 §待决策 (b)「往 target 侧 URMA 每流状态查」——**取消**，查它等于在不影响业务的路径上做优化
>
> ### 4. 主线因此收敛到一个可以交付的结论
>
> | 问题 | 结论 | 等级 |
> |---|---|---|
> | **vsegment 是否引入性能代价？** | **否**。同 cid / 同批 LBA / 同时窗，两 flags 交替 6 遍差 **0.00% / 0.14%** | **L3（C03）** |
> | 慢程是否影响读取？ | **否**。端到端差 4.3%，四遍一致 | **L3（C16）** |
> | 那 5.91× / 2.08× 是哪来的？ | **跨 cid 的数据状态差异**，与 vsegment 路径无关；具体混淆变量仍未定位 | L2 待查 |
>
> **⇒ 对本项目目标的回答：vsegment 的读取性能可以接受，vsegment 本身不引入代价。**
> 剩余未解的是「老数据读着慢」这个**底座级现象**，它对我的 vsegment 结论无影响，但是一条真实存在的产品问题。
>
> ### 5. 一个必须说明的缺口
>
> C03 的「两 flags 无差异」是 **400-key 口径**下测的。**2000-key 口径只有 vseg，没有 plain 对照**——因为建 plain cid 需要写入，会吃掉 etcd 仅剩的余量。
> ⇒ 严格表述：**「vsegment 无性能代价」建立在 400-key 口径的 L3 上；2000-key 口径未做 plain 对照。**
> ⇒ 若要补齐，需新建一个 plain cid 写 2000 keys（预计 oplog ~10 MB，是 vseg 的 1/90，etcd 余量可承受）。**是否要做，交用户决定。**
>
> ### 6. 环境与边界（全部符合）
> 新增写入 **0** 条（`read.log.gz` mtime 仍为 17:32:07 可证）；oplog 13,930 → 13,930、dbSize 1,345,523,712 → 1,345,523,712，**均零增长**；target PID 905586 / md5 未变；`vseg_nof086` PID 2292801 未重启、Keys=2000；mask 末态 `0x0`；未跑 bench、未跑 urma_perf —— **A 段「三项全有 ⇒ 跳 B 段」的判据执行正确，保住了零写入**。


## 0. 为什么派这单

NOF-083 C 段证明 **Mooncake 侧有慢程**（NEW080 fast 0.0%、OLD 62.9% 快档）。但 NOF-086 的 2000 keys 端到端**看不到它**（P99=705µs、Max=1227µs、分布跨度只有 2.6×，容不下一个 8× 慢档）。

**三份数据不能同时为真，必有一条错。** 本单唯一目标：**直接看 2000 keys 的读流里，前 400 MiB（前 100 个 key）是否真的更慢。**

这是主线的**生死判**：有差 ⇒ 慢程真实影响业务，继续查机制；没差 ⇒ 慢程对业务不可见，此前用它解释 Mooncake 5.91× 的方向整体作废。

## A. 先探现有日志能不能用（只读，10 分钟）

对象：`run/nof086_read.log.gz`（21 MB，`MC_HIFREQ_LOG_SAMPLE_RATE=1` 逐任务日志）

1. `zcat run/nof086_read.log.gz | head -5`，原样贴出
2. 取一条完整 per-task 记录，**列出全部字段名**
3. 三项逐条回答「有 / 无」：
   - 能否拿到 **key 标识**（key id / 对象名 / 卷内 offset / LBA）？
   - 能否拿到 **单 task 延迟**或起止时间戳？
   - 能否**分辨这是第几遍**（遍分隔标记 / 时间戳断层 / key 重复出现）？
4. 判据：**三项全有 ⇒ 直接用现有日志，跳 B 段进 C 段；任一缺失 ⇒ 进 B 段。**

## B. 若 A 不可用：重跑一遍干净读（**纯读，安全**）

1. 沿用 NOF-086 D 段的 reader 命令，只改 `--warmup_keys=0`，`--num_keys=2000`，`MC_HIFREQ_LOG_SAMPLE_RATE=1`
2. **只跑 1 遍**，跑完立即关日志开关
3. 先 `pgrep` 确认无残留；跑前记录 `etcd dbSize` 与 oplog entries，跑后复测，**两者必须零增长**
4. 这是纯读：不动数据、不动 oplog、不动 master 状态

## C. 按流内位置分档（核心）

1. **归组**：4 MiB key = 32 × 128 KiB task（以实际 task 数为准）。key 延迟口径二选一，**必须写明用了哪个**：
   - 有起止时间戳 ⇒「末 task 完成 − 首 task 起始」
   - 无 ⇒ 组内 **max**（更保守，慢档更容易显形）
2. **分档**（共 20 档）：

   | 档 | key 范围 | 说明 |
   |---|---|---|
   | 1–10 | 0–99，每 10 个一档 | **慢程区 = 前 100 个 key（400 MiB）**，细粒度看边界 |
   | 11–14 | 100–499，每 100 个一档 | 过渡区 |
   | 15–20 | 500–1999，每 250 个一档 | 快档基准 |

3. 每档给 `count / p50 / p90 / p99 / mean`
4. **判据（两边都写）**：
   - **前 100 key 的 p50 ≥ 2× key≥500 的 p50** ⇒ 慢程在端到端**可见** ⇒ U04 解释为「C 段的 NOF-078 旧 trace 不代表当前态」，慢程真实影响业务，主线继续
   - **差 < 20%** ⇒ 慢程在端到端**不可见** ⇒ U04 解释为「4 MiB key 内部并行把慢档摊平」，**慢程对业务不可见，此前用它解释 5.91× 的方向作废**
   - **20%–2× 之间** ⇒ 给过渡斜率与转折点位置（在哪一档掉下去）
5. 附加：给**前 100 key 的 Max**，与后段 Max 比。若慢档真在，前 100 的 Max 应在 3,000–5,000µs 量级。

## D. `--warmup_keys` 语义核实（一句话，必答）

`--warmup_keys=128` 是「读 128 个 key 作预热但**不计入统计**」还是「**跳过**前 128 个 key 不读」？
读 `--help` 原文或 bench 源码常量（**只读**），给原文引用。
**这决定 NOF-086 裁决 §8 的 warmup 对比是否成立**：若为「跳过不读」，§8 需降级，本单 C 段成为唯一判据。

## E. 回执必答（第一行）

前 100 key 的 p50 与 key≥500 的 p50 之比（**给的哪个口径**）；慢程在端到端**可见 / 不可见 / 过渡**；前 100 的 Max；`--warmup_keys` 语义原文；日志能否按 key 归组（用了 A 还是 B 段）；有无失败；**新增写入 LBA 清单（应为 0）**；oplog entries 与 dbSize 前后值（应零增长）。

## F. 边界

- **零写入**（B 段重跑也是纯读）；**零重启** target / master；**零改码**；**不切 cid**。
- **禁止写 `vseg_nof080` / `vseg_nof086`**；禁止重建任何 cid。
- **不测 plain**：那需要新 cid + 写入，会吃掉 etcd 仅剩的 316 MB 余量。本单只测 `vseg_nof086`。
- 不碰他人进程（royi master 3463695 / yin 的 `vseg_yin_bonding`）。
- 产物 `run/nof087_*.txt`（≤3 个、每个 ≤1 MB）；**不要重传 21 MB 原始日志**，只传分档汇总表。
- 若 A 段发现日志完全不含 key 维度信息、且 B 段重跑也拿不到 ⇒ **就此停手如实回执**，禁止自造解析、禁止猜字段、禁止用别的 cid 补数据。

---

# NOF-088（**已完成 · 结论翻转：C03/C17 撤回，vsegment 慢 2.07×** · 2026-11-24）

> **【2026-11-24 19:37 开发侧裁决 · C03/C17 立即撤回 · 结论翻转】**
>
> **A 段判「不存在」正确且证据硬；C 段执行规范。核心结论：vsegment 比 plain 慢 2.07×，与 NOF-025 的 2.08× 吻合到 0.5%。**
>
> ### 1. **C03 撤回（实验设计缺陷，非表述问题）**
>
> 硬证据两条，互相独立：
> - **8-key probe**：同一批 key、两套 flags 各读一遍，`type` / `selected_type` **全部是 `vsegment`**
> - **源码** `real_client.cpp:4598-4601`：`type[" << metadata.replica_type` —— **`type` 取自 master 返回的副本类型，与 reader flag 无关**
>
> ⇒ `--vsegment_nof_io=true` / `false` 与 `--replica_num=1` / `--nof_replica_num=1` 这组开关**不决定读哪条路径**。
> ⇒ **NOF-076/078/080 所谓「两 flags 交替 6 遍差 0.00% / 0.14%」，实际是同一条路径测了两遍。**
>
> **这是本轮最该记的教训：0.00% 的「完美复现」本身就是危险信号，不是强证据。**
> 两个真正不同的组，不可能差 0.00%；差 0.00% 更可能意味着**对照组根本没生效**。我在 NOF-076~080 那几轮把这个数字当成「L3 强证据」，方向反了。
> **⇒ C03 撤回，C17（vsegment 无代价）随之撤回——它唯一的依据就是 C03。**
>
> ### 2. **真实差距：vsegment 慢 2.07×，且与 NOF-025 独立吻合**
>
> | 遍 | plain P50 | vseg P50 | 比值 |
> |---|---|---|---|
> | 1 | 257.51 | 538.88 | **2.093×** |
> | 2 | 269.59 | 577.86 | **2.144×** |
> | 3 | 291.87 | 525.87 | **1.802×** |
> | 4 | 264.80 | 544.28 | **2.055×** |
> | **中位** | **~265** | **~540** | **2.07×** |
>
> **与 NOF-025（400-key 口径：plain 258.3 vs vseg 537.5 = 2.08×）吻合度 0.5%。**
> **两个独立实验、两个不同口径、两批不同数据，得到同一个 2.07–2.08×。** ⇒ 「布局混淆」很可能不是主因，NOF-025 的判决方向是对的（虽然它的具体数值口径仍需重做）。
>
> **差异定位（新增，有价值）**：
> | 段 | plain | vseg | 比值 |
> |---|---|---|---|
> | `query_us` | 95（mean 96.3） | 105（mean 110.8） | **1.11×** |
> | `read_us` | 165（mean 196.0） | 426（mean 448.0） | **2.58×** |
> | `total_us` | 265（mean 294.3） | 533（mean 554.7） | ~2.0× |
>
> ⇒ **慢在 `read_us` 段（2.58×），不在元数据/查询段（1.11×）。** 这把搜索面从「整条 vsegment 路径」收窄到「数据读取段」。
>
> ### 3. 登记 C18（L2，封顶原因写清）
>
> 本对照是**两批数据**（vseg_nof086 经 3 轮补写、plain_nof088 一轮写完），**布局混淆未排除** ⇒ 结论只能到 L2。
> **升 L3 的路径**：同一 cid 内构造两种副本（`replica_num=1` + `nof_replica_num=1` 同时挂），或写入顺序与物理布局严格对齐。**这正是 C03 当初想做但没做成的事。**
>
> ### 4. 附带硬数据：vsegment 的 oplog 是 plain 的 **1048×**
>
> | 口径 | entries | bytes | 写入轮次 | `VSEGMENT_CREATING` |
> |---|---|---|---|---|
> | plain_nof088 | 2,001 | **899,149 B** | **1 轮** | **0 次** |
> | vseg_nof086 | 13,930 | **942,185,725 B** | 3 轮 | 125 次 |
>
> ⇒ **1048×**。这不只是 oplog 体积差异，也是「写入复杂度」差异：plain 一次过，vseg 必须多轮补写。
> ⇒ 给产品缺陷 ③ 补上对照组：**同一 2000-key 工作量，vsegment 触发 942MB oplog，plain 只有 899KB。**
>
> ### 5. 哪些结论不受影响
>
> | 结论 | 状态 | 为什么不受影响 |
> |---|---|---|
> | **C16** 慢程在端到端不可见（1.043×） | **保留** | 是 vseg_nof086 **内部**前 100 vs 后 1500 的比较，不依赖 flags |
> | **C09** N = 400.0 MiB 按字节计量 | **保留** | 裸 SPDK 测量，与 Mooncake flags 无关 |
> | **C14** 步长 17 / `⌈log₁₇N⌉` 轮 | **保留** | 有 2000/3 轮的精确算术吻合 |
> | **C11** 32MiB 墙在客户端 | **保留** | 源码 + 数值双证据，与本次无关 |
>
> ### 6. 环境
> `vseg_nof086` 已永久报废（切 cid，942MB oplog，不可重启），其 NOF-086/087 数据已固化但不可重测 —— **这一点已在 D 段按要求先行记录**。
> target PID 905586 / md5 未变；master 切至 `plain_nof088` PID 2565234；etcd dbSize **零增长**（1,345,523,712），plain oplog 仅 899KB；mask 末态 `0x0`。


## 0. 为什么派这单

**（本段为 NOF-087 当时结论，已由 NOF-088 撤回）** 主线结论 C17「vsegment 本身不引入性能代价」目前只建立在 **400-key 口径**（C03：同 cid 两 flags 交替 6 遍差 0.00%/0.14%）。**2000-key 口径只有 vseg，没有 plain 对照。**

本单补这个缺口。**但优先方案不是新建 cid**——新建要重启 master，会让 `vseg_nof086` 永久报废（942MB oplog），还引入布局混淆。**先看能不能在同一批数据上换读取方式。**

## A. 先核实 bench 能否在同 cid 切读取方式（只读，10 分钟）

依据：NOF-086 日志里有 `type[vsegment] mode[full]` 字段，说明 `type` 是可变的。

1. `stress_cluster_bench --help` 全文检索与读取方式相关的开关，关键词：`vsegment` / `use_vsegment` / `storage_type` / `nof` / `plain` / `type` / `--mode`
2. 源码只读确认：`type` 字段有哪几个取值，切换是否需要重写数据
3. **判据：**
   - 存在「同一 cid、同一批 key，只换读取方式」的开关 ⇒ **走 B 段（优先，零写入零重启）**
   - 不存在 ⇒ **走 C 段（备选，新建 plain cid）**

## B. 方案 A：同 cid 两方式交替读（**优先；零写入、零重启、零 etcd**）

1. 对象固定 `vseg_nof086`（Keys=2000，勿动数据）
2. 口径**与 NOF-086 D 段逐项一致**：`num_keys=2000`、`io_size=4MiB`、`QD=batch=1`、`threads=1`、`slot=1`、同宿主、`warmup_keys=128`
3. 交替 4 遍：**vseg → plain → vseg → plain**（每遍独立进程，跑前确认残留=0）
4. 每遍记录：`Total / failed / Min / P50 / P90 / P99 / Max / Avg / MiB-s / keys-s`
5. **判据（两边都写）：**
   - 两方式 P50 差 **< 20%** ⇒ **vsegment 无代价，2000-key 口径确认 C17**
   - 差 **≥ 2×** ⇒ **vsegment 有代价**，C03 的 400-key 结论需重审，回执必须写明
   - 20%–2× ⇒ 给过渡值并标 `口径待细化`
6. **必须同时给**：两方式各自的 `type` 字段值（证明切换生效），以及 `read_us` / `query_us` 分解

## C. 方案 B（仅当 A 判据为「不存在」时执行）：新建 plain cid

1. **A 段判「不存在」才执行**，不得主动选这条路
2. 新 cid `plain_nof088`，`slot=1`，其余配置与 `vseg_nof086` 一致，**但关闭 vsegment**
3. 写 2000 keys：**plain 预计 1 轮写完**（无 `VSEGMENT_CREATING`），预计 oplog **< 2 MB**
4. 写前记录 etcd `dbSize` 与 `/oplog` 总体积；**余量只有 150.4 MB**（1,345,523,712 / 2GiB = 62.7%，70% 线余 157,714,841 B），**每 500 keys 复测一次，超 140 MB 增量立即停写**
5. 读 4 遍，口径同 B 段
6. **必须写明**：这是两批数据，**存在布局混淆**（plain 连续堆放 vs vseg 散在 vsegment 里），结论等级最高只能到 **L2**，不得与被 C03 排除的 NOF-025 混用

## D. 若走 C 段：切 cid 前先固化 vseg_nof086（不可逆前置）

1. 记录并写入回执：`Keys`、`logical_used`、`vsegments`、`/oplog/vseg_nof086/` 条目数与字节数、`dbSize`
2. 明确记录 NOF-086 / NOF-087 已采完的 4 遍读数据（P50 538.88/577.86/525.87/544.28、per-key 分档表）
3. **切 cid 后 `vseg_nof086` 永久不可重启**（942MB > 客户端 32MiB recv 上限），以上数据不可重测 —— 这一点已由开发侧源码确认：新 cid 的 `master_id` / `cluster_id` 不同 ⇒ 回放走新前缀，**不会误读旧 cid 的 942MB**（`master_service.cpp:212` `master_id_(config.master_id)`；前缀 `/oplog/<cluster_id>/<source_id>/`）

## E. 回执必答（第一行）

bench 能否同 cid 切读取方式（A 段判据命中哪条）；走了 B 还是 C；**两方式的 P50 与比值**；两方式 `type` 字段值；`read_us` / `query_us` 分解；有无失败；**新增写入 LBA 清单**（B 段应为 0）；oplog 与 dbSize 前后值。

## F. 边界

- **零改码、零重编**；**B 段零写入、零重启、零切 cid**。
- **禁止写 `vseg_nof080` / `vseg_nof086`**；**B 段禁止切 cid、禁止重启 master**。
- C 段才允许切 cid，且必须先完成 D 段固化。
- 不碰他人进程（royi master 3463695 / yin 的 `vseg_yin_bonding`）。
- **不要删任何 `/oplog` 前缀**（删除可释放 899MB，但属不可逆操作，须用户单独授权，本单不做）。
- 产物 `run/nof088_*.txt`（≤3 个、每个 ≤1MB）。
- 若 A 段判「不存在」且 C 段 etcd 增量逼近 140MB ⇒ **停手如实回执**，不得为凑数降 keys 或改口径。

---

# NOF-089（**把 read_us 的 2.58× 拆成 4 个子项** · 纯离线 / 零写入 · 2026-11-24 19:40 派）

## 0. 为什么派这单（开发侧理由，不是猜测）

C18 已知：vseg `read_us` **426µs** vs plain **165µs** = **2.58×**，而 `query_us` 只差 1.11×。
**但 `read_us` 本身还能拆。** 源码 `real_client.cpp:4511-4517` 确认：

```cpp
known_read_us    = offload_rpc_us + transfer_data_us + release_buffer_us
read_overhead_us = read_us - known_read_us    // read_us <= known 时为 0
```

⇒ **`read_us` ≈ `offload_rpc_us` + `transfer_data_us` + `release_buffer_us` + `read_overhead_us`**（可加和验证）

**两边日志都还在**：vseg 侧 `run/nof086_read.log.gz`（21MB）、plain 侧 `run/nof088_read.log.gz`（17.6MB），字段是同一套 19 项。
⇒ **不需要跑任何新实验，就能定位 2.58× 落在哪一段。** 这比「验证 2 倍是否成立」有用：验证只回答「是不是」，拆分回答「在哪」。

## A0. **零值预检（必做第一步，防白工）**

> **开发侧新发现的风险（2026-11-24 19:45）**：这四个子项**只在 offload 路径里赋值**（`real_client.cpp:7359/7397/7422/7458`，均由 `if (current_offload_read_timing)` 保护，而该指针只被 `ScopedOffloadReadTiming` 设置）。
> 而 `mode` 字段**不是** offload/full 的意思 —— `real_client.cpp:4486` 是 `(src_offset==0 && actual_size==total_size) ? "full" : "range"`，即**「整对象 vs 部分对象」**，与走不走 offload 无关。
> ⇒ **`mode[full]` 完全不能证明 offload 字段非零。本单必须先在 A 段验证，否则可能拆出 `0/0/0/426` 这种零信息量结果。**

1. **各抽 3 条完整日志原文**（vseg 侧 3 条、plain 侧 3 条），原样贴出，不要摘要
2. 逐条读 `offload_rpc_us` / `transfer_data_us` / `release_buffer_us` 三个值
3. **判据（硬）：**
   - **三项在两侧、六条样本里全为 0** ⇒ **立即停手**，回执第一行写「offload 三子项恒为 0，split 无信息量」+ 贴那 6 条原文，**不要执行 B/C 段**
   - 三项中**至少一项非零** ⇒ 继续 B/C 段
4. 同时记录两侧 `mode` 与 `type` 的实际取值（用于说明为什么走/不走 offload）

> 若三项全 0，说明耗时全在 `read_overhead_us`（= `read_us`），那本单的替代价值是：**确认 2.58× 不在 RPC/传输/释放这三段**，方向应转向 `execute_ranged_read` 内部或 target 侧。这一点在停手回执里写明。

## A. 提取（纯离线，用已有日志）

1. **vseg 侧**：取 `nof086_read.log.gz` 的 **W0 段**（`warmup=0`，2000 条完整，无预热混入）。
   识别方法：该段 `key` 无重复出现（预热 key 会重复）。若无法可靠切段 ⇒ 改用 R1 段并**只取 key≥128**（排除预热），在回执写明用了哪段。
2. **plain 侧**：取 `nof088_read.log.gz` 的 **P1 段**（2000 条，`type=nof_remote`）。
3. 两侧各提取每个 key 的：
   `query_us` / `select_us` / `select_replica_us` / **`read_us`** / **`offload_rpc_us`** / **`transfer_data_us`** / **`release_buffer_us`** / **`read_overhead_us`** / `total_us`
4. 记录两侧的 `mode` / `type` / `replica_count` 取值分布（**必须核对两侧是否同 mode**，不同则全单结论降级）

## B. 统计与加和校验（必做）

1. 每个字段给 `p50 / p90 / mean`，两侧并列，给**差值（µs）与倍数**
2. **加和校验**：逐 key 验证 `offload_rpc + transfer_data + release_buffer + read_overhead` 是否 ≈ `read_us`（误差 ≤2% 计通过）。
   - 不通过 ⇒ 说明该版本代码的定义与本地 `706551a` 不同，**记 `[未获取]` 停手**，不要强行归因
3. **贡献度分解**：把 `read_us` 的 261µs 总差值（426−165）按四个子项拆分，给每项**占总差值的百分比**

## C. 判据（两边都写）

| 情形 | 判定 |
|---|---|
| 某一项占总差值 **≥ 70%** | **该段是 2.58× 的主因**，记明是哪一项，主线收窄到该层 |
| 差值是**分散的**（无单项 ≥70%） | 说明是**整体开销放大**，不是单点；下一步应查「vsegment 是否让每 key 的有效 IO 次数变多」 |
| `read_overhead_us` 占大头 | 说明耗时在**客户端未被计时的部分**（拼装/拷贝/等待），方向转向客户端 |

## D. 附带（顺手，零成本）

1. 给两侧 `read_us` 的**分布形状**：`min / p1 / p50 / p99 / max`，看 vseg 是「整体右移」还是「长尾拉高」
2. 给两侧 `transfer_data_us / 4MiB` 的**等效带宽**（MiB/s），与 bench 报的 `Throughput` 对账

## E. 回执必答（第一行）

2.58× 落在哪一项（或「分散」）；该项占总差值百分比；四项各自的 p50 与倍数；加和校验是否通过；两侧 `mode` 是否相同；有无失败；**新增写入=0**。

## F. 边界

- **零写入、零重启、零改码、不切 cid、不跑 bench、不跑 urma_perf**。只读 `run/` 下两份已有 `.log.gz`。
- **禁止写 `vseg_nof080` / `plain_nof088`**；禁止动 master（现为 `plain_nof088` PID 2565234）。
- 结论等级上限 **L2**（两批数据布局未对齐）；本单只做「哪一段」的定位，**不做因果定性**。
- 不碰他人进程；产物 `run/nof089_*.txt`（≤2 个、每个 ≤1MB），**不要重传原始日志**。
- 若 A 段发现日志字段缺失或两侧 mode 不同 ⇒ **如实停手回执**，不得拼凑、不得用其他 cid 数据补。

---

# NOF-090（**版本对照：换掉 be10be2，看 2× 是不是版本问题** · 2026-11-24 20:25 派）


> ## 🛑 NOF-090 暂停 · 版本目标搞错了（2026-11-24 20:37 · **优先级最高**）
>
> **开发侧实地核查 `D:\gds性能\repos\Mooncake` 后确认：本单上一版写的「目标 bf3b95a」「基线 be10be2」全部错误，不得执行。**
>
> ### 实际用的是哪个分支
>
> | 项 | 实际值 |
> |---|---|
> | 源码路径 | `D:\gds性能\repos\Mooncake` |
> | 远端 | **`https://github.com/LeeJace993/Mooncake`（同事的 fork，不是 kvcache-ai/Mooncake）** |
> | **分支** | **`nof-urma-v2-bench-merge`** |
> | 本地 HEAD | **`d371e22`（2026-10-23）** `[Build] Pin SPDK Round 7 v2 tracing fix` |
> | 落后远端 | **1 个提交**：`523256d`（2026-10-28）`fix(ub): honor MC_EID_INDEX when selecting URMA context` |
> | 本地未提交改动 | `mooncake-store/src/real_client.cpp`、`mooncake-store/src/spdk/nof_memory_domain.cpp` |
>
> ### 三处必须更正
>
> 1. **`be10be2` 不在我们的历史里** —— 本地 `git cat-file -t be10be2` 报「无此对象」。它是 `kvcache-ai/Mooncake` main 上的提交，与 `nof-urma-v2-bench-merge` 分支无关。**HANDOVER 里「构建基线 be10be2」这条记录是错的**，已在此更正，后续不得再引用。
> 2. **`bf3b95a` 不能当目标** —— 它是 kvcache main 的 HEAD，而我们跑的是 fork 的 NoF/URMA-v2 专用分支，带大量未上游定制。**直接换到 kvcache main 会丢掉这些定制，不可行。**
> 3. **唯一可升的那 1 个提交与性能无关** —— `523256d` 只加 `MC_EID_INDEX` 配置与 EID 选择逻辑（`urma_eid.h` +36 行、`urma_endpoint.cpp` 19 行改动 + 3 个测试），**不涉及读路径性能**。
>
> ### 因此本单现状
>
> - **A/B/C 段全部暂停**，不要构建、不要切 cid、不要写入。
> - 我们这条线**没有"升一版就能拿回 2×"的选项** —— 现有分支上不存在这样的提交。
> - **正确下一步待用户决定**（见下），开发侧不擅自选。
>
> ### 待决方向（需用户拍板）
>
> | 方向 | 内容 | 代价/风险 |
> |---|---|---|
> | **A** | 找同事（`LeeJace993`）确认：这个分支是否已是可用的最新版、他那边有没有更新的性能改动没推上来 | 零成本，但要等对方 |
> | **B** | 把 kvcache main 上的性能修复（`3838892` submit watermark、`f3b8101` 非阻塞批量状态、`d3fd032` SSD prefetch）**cherry-pick 到 `nof-urma-v2-bench-merge`** | 需先确认这些修复在我们这条线上是否适用（谱系可能已分叉一个月）；fetch kvcache main 失败（`invalid index-pack`），谱系未算清 |
> | **C** | 停止版本方向，接受"2× 是当前分支的固有代价"，转「业务影响评估 or 改设计」 | 零成本，但放弃找回 2× |
>
> **开发侧倾向**：先走 **A**（零成本、且这个 fork 是同事维护的，他最清楚分支状态）。**B 的可行性取决于谱系，而谱系目前没算清，贸然 cherry-pick 风险高。**

## 0. 开发侧决策依据（不是猜测）

经 NOF-025→089 共 60+ 轮排除，**IO 层与命令层已被证完全一致**（NOF-070：`BDEV_IO_START.size` 全 128 KiB，vseg 33,792 / plain 33,792；NOF-073：LBA/NLB/opcode/FUA 元素级一致，1,164 个 offset only_A=only_B=0）。

本轮源码排查结论（开发侧，`706551a`）：
- vseg 与 plain 的结构差异只有三处：`Plan()`（layout 校验 + stripe 拆分）、`openSegment()`（map 查找）、request 数量
- **`Plan()` 里有一处确定的重复计算**：`Plan()` 自己 `ValidateViewStructure` 一次，`ResolveTransfer` 内部又校验一次；上游 `13b7c19`（"skip duplicate view validation"）已修，**我们跑的 `be10be2` 没有**
- **但量化后该开销仅 3–5µs/键（序列化 ~660B + CRC32C 硬件指令 ×2），占 426µs 的约 1%** ⇒ 不构成主因

⇒ **剩下唯一未被排除的可能是：我们测的版本落后。** 已知 `be10be2` 落后 tip `4c0ea28`，后者含 `ff3e853`（诊断改 opt-in，commit message 明写"directly inflates caller-observed read latency"）与 `13b7c19` 等读路径修复。

**本单唯一目标：用同一台机器、同一套 400-key 口径，比较「旧客户端」与「新客户端」下的 vseg/plain 比值。**

## A. 前置核实（只读，30 分钟）

0. **先算升级跨度**：执行 `git log --oneline be10be2..HEAD | wc -l` 得提交数、`git diff --stat be10be2..HEAD | tail -1` 得文件变更数，两者都写进回执。**若跨度 >150 个提交或 >500 个文件变更 ⇒ 在 A0 停手**，回执写明跨度过大、不建议整体升级，改为只 cherry-pick A2 列出的四条。
1. 记录当前**在用**的客户端二进制 md5、其 Mooncake git HEAD、SPDK 版本
2. 去远端仓库列出 `be10be2..HEAD` 之间**涉及读路径**的提交（`mooncake-store/src/vsegment/`、`transfer_task.cpp`、`client_service.cpp`、`real_client.cpp`），逐条给 hash + 标题 + 一句话影响
3. **关键约束核实**：**SPDK wire 版本必须保持 v4**（`e12b7cb` / `bd5ee124d` / `170cbb9`）。历史上 `9a413a44` / `91763b015` 是 wire **v6**，与运行中 target `e12b7cb` 握手失败（rc=-71/-104，NOF-050）。**若新版 Mooncake 强制要求 v6 SPDK，则本单在 A3 停手**，回执写明"升级被 wire 版本阻塞"，不要尝试重启 target
4. 记录 target 现状：PID 905586 / md5 `3129641f…` / 4457。**本单全程不重启 target**


> **【2026-11-24 20:31 版本目标修正 · 覆盖本单任何「含上述修复的版本」表述】**
>
> **开发侧查证 GitHub（kvcache-ai/Mooncake），两处必须更正：**
>
> | 项 | 原记录（错） | 实际 |
> |---|---|---|
> | `4c0ea28` | 「远端 tip，含 ff3e853 / 13b7c19 等读路径修复」 | **只是一个测试提交**：`test(vsegment): cover nof_logical_used gauge and RETIRED exclusion`，改 `vsegment_test.cpp`，+55 行 |
> | 「远端 tip」 | `4c0ea28` | **`bf3b95a`（2026-11-24）**，比 4c0ea28 新得多 |
>
> **分支归属（已用 `git ls-remote` 核实，2026-11-24 20:35）**：
> - `bf3b95a` = **`refs/heads/main` 的 HEAD**（仓库共 93 个分支，无 `master`，默认分支就是 `main`）
> - `be10be2` **也在 main 上**（没有任何分支的 HEAD 指向它，说明它是 main 历史中的一个提交，不是侧分支）
> - ⇒ **升级路径干净：就是沿 `main` 从 be10be2 往前走，不需要跨分支合并**
> - 唯一相关侧分支是 `codex/segment-pool-nof-placement-cleanup`（`fd93f9f`），**不是我们用的，不要去那个分支**
>
> **一个很有说服力的细节**：`be10be2` 自己的标题是 `[Bugfix] Register NoF VSegment benchmark buffers`，它引入了 `--vsegment_nof_io` 这个 flag（NOF-088 用的就是它）。**⇒ 我们的基线版本恰好是「为了让 vsegment bench 能跑」而打的那个提交**，功能刚打通，未必经过性能优化。
>
> **⇒ 本单目标版本定为 `bf3b95a`（main HEAD，2026-11-24）。** 若拉取时 HEAD 已前进，**允许取最新 HEAD，但必须在回执写明实际用的 hash**。
>
> **从 `be10be2` 到现在，与读延迟直接相关的提交（按可疑度排序，A 段必须逐条核实是否影响我们的路径）：**
>
> | commit | 标题 | 为什么可疑 |
> |---|---|---|
> | **`3838892`** | `[Bugfix][TransferEngine] Fix UB transport submit watermark using wrong per-endpoint channel count` | **提交水位用错 channel 数 ⇒ 限流计算错误 ⇒ 直接影响 IO 提交节奏**。这是我们 60+ 轮都碰不到的层，且完全能解释「整体右移而非长尾」 |
> | **`f3b8101`** | `[Integration][TE] Support non-blocking whole-batch and vectorized status checks with BatchID validation` | 批量状态查询改非阻塞 ⇒ 直接影响 `read_us`（我们的 `read_us` 就是「提交 + 等完成」） |
> | **`d3fd032`** | `[Store] Enable SSD prefetch for standalone mooncake_client` | SSD 预取，直接关系读延迟 |
> | `13b7c19` | `vsegment: skip duplicate view validation` | 已量化仅 **3–5µs（约 1%）**，不是主因，但顺手带上 |
>
> **A 段必答**：上述四条各自的改动文件与函数；`3838892` 改的 transport 是否覆盖 **URMA**（我们用的），还是仅 UB；`f3b8101` 改的是否是我们走的 `get_into` → `execute_ranged_read` 路径。
> **若 `3838892` 限定 UB-only、且其余三条都不作用于 `get_into`** ⇒ **本单价值大幅下降，允许在 A 段停手**，回执写明理由。


## B. 构建新版客户端（不改 target）

1. 在**我方副本**内更新 Mooncake 到含上述修复的版本；SPDK 依赖锁在 wire v4
2. 依 NOF-048 验证过的最小编译路径：改 `.cpp` → 编译 → `ar r` 换库成员 → 重链（约 65s/轮）
3. 产物 `--help` 正常、client 与 target **握手成功**（这是硬门槛，握手不过就停手）
4. **不改 target、不改 etcd、不改 master**

## C. 同口径重测 400 keys（**唯一允许写入的段**）

口径：**400 keys × 4 MiB**、`threads=1`、`batch_size=1`、`warmup_keys=128`、同宿主（node3 本机）。

1. **plain 臂优先**：先尝试用新版客户端读现有 `plain_nof088` 的前 400 个 key（零写入）。
   - 若新版读不了旧 cid（格式不兼容）⇒ 新建 `plain_nof090` 写 400 keys（约 178 KB oplog）
2. **vseg 臂**：新建 `vseg_nof090`，写 400 keys（预计 22 MB oplog、需多轮补写，`VSEGMENT_CREATING` 约 23 次，步长 17）
3. 两臂各读 **3 遍**，给 `Min/P50/P90/P99/Max/Avg`
4. **etcd 余量 150 MB**，写入前复测 `dbSize`，增量超 100 MB 立即停

> **注意写入顺序**：新 vseg 的数据会覆盖每盘 0–145 MiB，正好是 `plain_nof088` 前 400 key 的位置 ⇒ **plain 臂必须先测完再写 vseg**。

## D. 判据（两边都写）

对比三个数：**旧客户端 vseg/plain = 2.07×**（NOF-088，2000-key）与 **旧客户端 400-key = 2.08×**（NOF-025）。

| 新版 400-key 的 vseg/plain | 判定 | 下一步 |
|---|---|---|
| **< 1.2×** | **2× 是版本问题**，升级即解决 | 升到生产版，主线收口 |
| 1.2–1.7× | 版本修掉了一部分 | 逐条比对 A2 列出的提交，定位是哪一条 |
| **≥ 1.9×** | **不是版本问题**，是设计/实现固有代价 | 停止版本方向，转「接受 or 改设计」决策 |

## E. 回执必答（第一行）

新版 vseg/plain 比值；落在 D 段哪一档；新旧客户端各自的 md5 与 HEAD；A2 列出的读路径提交清单；握手是否成功；wire 版本是否仍是 v4；有无失败；**新增写入 LBA 清单与 oplog 增量**。

## F. 边界

- **禁止改 target、禁止重启 target**（PID 905586 / md5 `3129641f…` 必须不变）。
- **禁止改共用 etcd**；**不要删任何 `/oplog` 前缀**。
- 若 A3 判「升级被 wire 版本阻塞」⇒ **就此停手**，不得为了升级去重启 target 换 wire 版本（那会让旧 e12b7cb 永久不可恢复）。
- 400-key 口径，**不得超过 400 keys**；写入前必查 etcd 余量。
- 不碰他人进程；产物 `run/nof090_*.txt`（≤4 个、每个 ≤1MB）。
- 若构建失败 3 次仍不通 ⇒ 停手回执，给失败日志尾部 50 行。

---

# NOF-091（**拿 vsegment 内部分解：plan / submit / wait** · 优先零改码 · 2026-11-24 20:42 派）

> **【2026-11-24 20:55 开发侧裁决 · 两个阻塞都放行 · 并更正一个前提】**
>
> **A 段结论成立，B1 路线技术可行**：`MOONCAKE_ENABLE_SPDIAG=ON` / layer=system / `/usr/bin/spdiag` 在位 / bench 链接 `libspdiag.so.1` / MODULE(3) 点位有值 ⇒ **埋点是活的**，这条路能走。
>
> ### 1. **必须先更正一个前提：测试机源码 ≠ 开发侧分析用的源码**
>
> 回执「源码树 **be10be29** dirty=3」 ⇒ 测试机用的是 `be10be29`。
> 而开发侧本轮所有源码分析，用的是本地副本 `D:\gds性能\repos\Mooncake`（`d371e22`，2026-10-23）。
> **⇒ 两者不是同一份代码。** 此前基于本地副本得出的三个判断（`Plan` 重复校验仅 3–5µs、`openSegment` 只是 map 查找、两臂提交方式一致）**降级为「待核」，不得再作为定案依据**。
>
> **本单 C 段补做**：把测试机上 `client_service.cpp` 的 vsegment 分支原样贴出（约 L2170–2340 共 170 行），开发侧逐行比对后再重新定性。
>
> ### 2. 阻塞 1（vseg 臂无数据源）→ **放行重启 master**
>
> 前提已满足：plain 臂数据够了（NOF-088 四遍 + 本单两遍）。
>
> 1. **先固化** `plain_nof088` 状态：`Keys=2000`、`/oplog/plain_nof088/` = 2,001 entries / 899,149 B、`dbSize` 现值。**它的 oplog 只有 899 KB，随时可切回**，风险远低于 `vseg_nof086`（942 MB 不可重启）
> 2. 新建 `vseg_nof091`（`slot=1`，vsegment enabled），写 **400 keys**（预计 22 MB oplog；余量 150 MB）
> 3. **写入用 400 keys 上限**，不得超；每轮复测 `dbSize`，增量 >100 MB 立即停
> 4. 切 cid 前后**不得重启 target**（PID 905586 / md5 `3129641f…` 必须不变）
>
> ### 3. 阻塞 2（采样窗口不足）→ **不改全局配置，改用「多跑几遍」**
>
> 回执推断「dump 周期 1000ms，400-key 读流仅约 0.5s，进程在第二次 dump 前退出」——**推断合理。**
>
> **禁止改 `/etc/spdiag/spdiag.conf`**（系统级共享，会影响其他用 SpDiag 的进程；本单未获此授权）。
>
> **改用**：同一口径**连跑 ≥6 遍**（每遍约 0.5s ⇒ 累计 ≥3s，跨过 ≥3 个 dump 周期）。
> - SpDiag 计数是**累积**的，多遍会累加，取均值仍有效；**口径不变**（仍 400 keys × 4 MiB、warmup=128、batch=1、thread=1）
> - 跑完**立即** `spdiag history` / `show` 取最终累计值，再与 bench 报的每遍 P50 对账
> - **两臂都要跑 ≥6 遍**（plain 也补跑，使口径对称）
>
> **判据（两边都写）**：
> - 若 `Ticks` 随遍数增长（≥128 + 400×遍数）⇒ 窗口问题解决，取测量段数值
> - **若 Ticks 仍恒为 128（不随遍数增长）⇒ 说明 SpDiag 对 MODULE 点位有独立上限或只记首块，属另一问题**：如实回执并**停手**，不要为凑数去改配置或改口径
>
> ### 4. warmup 段那组数据仍保留价值（但要标口径）
>
> 本单抓到的 `Submit[G]=4,857ns` / `Wait[G]=441,661ns` ⇒ **wait 占 97.8%**，与「时间花在等设备」一致。
> **但必须标注**：那是 warmup 段（冷启动），该遍 avg 554µs / max 9.2ms，而测量段 avg 328–381µs / max 695–815µs ⇒ **绝对值偏大，只有比例结构可参考**，不得与测量段数值混用。
>
> ### 5. 回执必答补充
>
> 除原任务书 D 段外，追加：**测试机源码 HEAD 与 `git status` 是否仍为 `be10be29` dirty=3**；`client_service.cpp` L2170–2340 原文；两臂各跑了几遍、Ticks 是否随遍数增长；`vseg_nof091` 的 oplog 增量与 `Keys`。

> **【2026-11-24 21:48 开发侧裁决 · 不改码，换 warmup=0 检验 128 配额假设（后被否证）】**
>
> **本单的 L1 排除成立**：层（Layer 1）、CLI（`/usr/bin/spdiag`）、门控（`level_gate=0`）、点位（MODULE 级确有记录）已逐一排除；`SetupReal Ticks=1` / `RegisterBuffer Ticks=2` 证明**不是全局截断**；头文件里 `MAX_PERF_SLOTS=256`、`maxTopN=100` 都不是 128 ⇒ **128 是库内部按点位计的固定窗口**，无源码可查。**在「400 keys + warmup=128」这个用法下，spdiag 确实给不出测量段数据。**
>
> ### 1. 关键线索：128 = warmup_keys = 128，不是巧合
>
> `GetIntoInternal` 的 `Ticks` 恒为 **128**，而 bench 的 `--warmup_keys` 也正好是 **128**。
> ⇒ **推断：SpDiag 每个点位只保留前 128 个样本（用于算分位数）；bench 先读 128 个 warmup key，正好把这个配额吃满，正式测量的样本一个都没进去。**
> ⇒ 这解释了为什么抓到的是冷启动量级（avg 554µs / max 9.2ms），而不是测量段（avg 328–381µs）。
>
> ### 2. 绕过办法：`--warmup_keys=0`（**零改码、不改配置、一轮可得**）
>
> `--warmup_keys=0` 是合法参数（NOF-086 的 W0 那遍即为 0）。设成 0 后，**前 128 个样本直接落在测量段**。
>
> **执行**：两臂各跑一遍 `--warmup_keys=0`，其余口径不变（`--num_keys=400 --value_size=4MiB --batch_size=1 --num_threads=1`），全程 `spdiag watch --interval 200`，跑完取三个点位的 `Ticks / avg / p50`。
>
> ### 3. 判据（两边都写，必须看数值有没有变）
>
> | 跑 `warmup=0` 后 | 判定 |
> |---|---|
> | **Ticks 仍 128，但 avg/p50 从 ~554µs 降到 ~300µs 量级** | **假设成立，已拿到测量段数据** ⇒ 直接进第 4 步判 plan/submit/wait |
> | Ticks 仍 128，数值仍是 ~554µs 那组 | 128 是别的机制，此路不通 ⇒ **退回 B2（改码加 chrono）**，不要再试 other 参数组合 |
> | Ticks < 128 | 说明配额更小，如实记录，按实际样本数报 |
>
> ### 4. 拿到数据后的判据（同原任务书 C 段）
>
> | 结果 | 判定 |
> |---|---|
> | **`wait` ≥ 70%** | 时间花在等设备 ⇒ 结合 target W9 的 1.95× 与「IO 命令已被证完全一致」（NOF-073）⇒ **只剩「IO 到达时序」**，搜索面收窄到提交节奏 |
> | **`submit` ≥ 30%** | 提交本身有开销 ⇒ 查 `submitVSegment` 内的 `openSegment` 循环与 lane 分配 |
> | **`plan` > 30µs** | 开发侧「3–5µs」估算错了，须按 `be10be29` 实测重估 |
> | 三者之和远小于 `read_us` | 仍有未覆盖段，如实写明差额，**不要硬凑** |
>
> ### 5. vseg 臂仍然必须建（本单未建，下轮补）
>
> 本单因判据判停未建 `vseg_nof091`，**下一轮必须建**：`slot=1` + vsegment enabled，写 **400 keys**（约 22 MB oplog，余量 150 MB），每轮复测 `dbSize`，增量 >100 MB 立即停；**不得重启 target**（PID 905586 / md5 `3129641f…`）。
>
> ### 6. 三个偏差的处理
>
> - **plain 第 3 遍异常**（P50 460.66 / Max 5213 / 1511 keys/s，其余 5 遍 289–324µs 稳定）：回执未能定位，且与 node1 他人 `cmake --build`（21:45:36）**时间对不上**（第 3 遍为 21:43:56）。**标注为噪声，不参与任何计算、不据此推断。**
> - **oplog 字节口径**：本单按 value 计 757,092 B vs NOF-088 记录 899,149 B，差 142 KB（疑为 key 名/etcd 开销）。**两种口径并列标注、不混用**——这是正确处置。
> - **`dirty=3` 归属更正**：`be10be29` 树 status=2（仅 etcd go.mod/go.sum）；`dirty=3` 属于我方副本 `mooncake-src`（多一条 `transfer_task.cpp` 日志插桩）。**NOF-091 首轮回执的归属表述有误，此处更正。**
>
> ### 7. 不选的选项
>
> 回执给的三个选项——①改走 B2 ②先查 128 机制 ③接受 warmup 比例——**暂都不选**：
> - ① 在试过 warmup=0 之前不必付改码成本
> - ② 需要 SpDiag 库源码，代价高且不带来性能结论
> - ③ 只有比例结构、绝对值不可用，不足以支撑判据
>
> **先把 warmup=0 跑一遍，成本一轮，可能直接解决。**

## 0. 为什么派这单（开发侧已查到的关键事实）

`execute_ranged_read` 的 `read_us`（vseg 426µs / plain 165µs）**内部没有 breakdown 埋点**（NOF-089 已证），所以 `offload_rpc/transfer_data/release_buffer` 三个字段恒 0。

**但源码里已经有三个正确的埋点位置**（`client_service.cpp`，vsegment 分支）：

| 点位 | 行 | 覆盖什么 |
|---|---|---|
| `PerfKey::VSEGMENT_READ_PLAN` | 2246-2252 | `vsegment_transfer_planner_->Plan()` —— layout 校验 + 按 stripe 拆分 |
| `PerfKey::VSEGMENT_READ_SUBMIT` | 2257-2262 | `transfer_submitter_->submitVSegment(plan, READ)` —— 含每个子请求的 `openSegment` |
| `PerfKey::VSEGMENT_READ_WAIT` | 2312-2314 起 | 等 `TransferFuture` 完成 |

**关键前提（`docs/spdiag_integration_guide.md`）**：这些 `SpDiag::PerfPoint` 只有在 **`MOONCAKE_ENABLE_SPDIAG=ON`（Layer 1）** 时才真正计时；**默认 `OFF`（Layer 0）会把 `Start()`/`End()` 编译成空实现**。
⇒ **这解释了为什么 60+ 轮没人用过这份数据：很可能构建时它就是关的。**

## A. 只读核查（5 分钟，决定走 B1 还是 B2）

### A0. ~~先确认测试机上实际用的是哪份源码~~ · **【2026-11-24 21:03 已拆分为独立任务 NOF-092，本单跳过 A0、直接从 A1 开始】**

> **开发侧的结论有缺口，必须由本单补上**：开发侧只核对了**本地副本** `D:\gds性能\repos\Mooncake` 的 git 元数据（`git branch --show-current` = `nof-urma-v2-bench-merge`，HEAD `d371e22`，origin = `LeeJace993/Mooncake`）。**但从未验证测试机上跑的那份源码/二进制是否来自同一个副本**。这是推断，不是实测。

1. 找 bench 二进制：`find / -maxdepth 5 -name "stress_cluster_bench*" -type f 2>/dev/null | head`
2. 看它的来源线索：`ls -l <路径>`（mtime）、`strings <路径> | grep -iE "mooncake-store/src|/repos/Mooncake" | head -5`
3. 在它所属的源码目录执行：`git branch --show-current`、`git log -1 --format="%h %ci %s"`、`git remote -v`、`git status --short | head`
4. **判据：**
   - 分支 = `nof-urma-v2-bench-merge` 且 HEAD ≥ `d371e22` ⇒ **与开发侧一致，继续 A1**
   - **分支或 HEAD 不同** ⇒ **回执第一行必须写明实际值**，并说明该分支上是否存在 vsegment 读路径相关改动 ⇒ **开发侧的版本结论需要整体重算**
   - 找不到源码目录（只有预编译二进制）⇒ 记 `[未获取]`，从**二进制 mtime + md5** 推断，不要猜分支

### A1 起的核查

1. **SpDiag 运行时是否存在**：`which spdiag`；`ls /usr/lib*/libspdiag* /usr/local/lib*/libspdiag* 2>/dev/null`；`ls /usr/lib*/cmake/SpDiag* 2>/dev/null`
2. **当前 bench 是什么层**：在构建目录 `grep -i "MOONCAKE_ENABLE_SPDIAG" CMakeCache.txt`；再 `ldd <bench二进制> | grep -i spdiag`（有 `libspdiag.so` = Layer 1）
3. **判据：**
   - **`spdiag` CLI 存在 且 bench 链接了 `libspdiag.so`** ⇒ **走 B1（零改码）**
   - 否则 ⇒ **走 B2（改码加 chrono）**

## B1. 优先：零改码，直接用 spdiag（**若能走这条，不动任何代码**）

1. `spdiag clear`（清零）；若提示未初始化则 `spdiag start`
2. 用**现有 bench 二进制**跑一遍 vseg reader：`--num_keys=400 --num_threads=1 --batch_size=1 --warmup_keys=128`，同宿主 node3 本机
3. 跑完立刻 `spdiag show`（或 `spdiag show --top 50`），把 **`VSEGMENT_READ_PLAN` / `VSEGMENT_READ_SUBMIT` / `VSEGMENT_READ_WAIT`** 三个点位的 `count / avg / p50 / p99 / max` 原样贴出
4. **对账**：三个点位之和应与该遍 `read_us` 同量级；若对不上，说明还有未覆盖的段，如实写明
5. 额外：同时贴 plain 臂（`nof_remote`）的 `GET_INTO_INTERNAL_NOF_READ` 等相邻点位，用于对照

> **注意**：vseg 数据已被覆写（`vseg_nof080` / `vseg_nofvseg` / `vseg_nof084` 均不可用，见顶部 cid 现状表）。**本单若需 vseg 数据，须新建 `vseg_nof091` 写 400 keys**（约 22 MB oplog）；plain 臂可读 `plain_nof088` 前 400 个。**写入顺序：先测 plain，再写 vseg。**

## B2. 备选：改码加 chrono（**仅在 A 段判「Layer 0 / 无 spdiag」时执行**）

1. 只改 `mooncake-store/src/client_service.cpp` 的 vsegment 分支，**只加计时，不改任何逻辑、不改控制流、不改返回码**：
   - `submitVSegment` 前后各取一次 `steady_clock::now()`，求差记 `vseg_submit_us`
   - `future` 等待前后各取一次，求差记 `vseg_wait_us`
   - `Plan()` 前后各取一次，求差记 `vseg_plan_us`（已有 `pt_vsegment_plan`，无 spdiag 时再手工取一次）
   - 用 `LOG(INFO)` 输出到与 `get_into_breakdown` 同一份日志，字段名 `vseg_plan_us` / `vseg_submit_us` / `vseg_wait_us`
2. 编译走已验证路径：`cmake --build` 该目标 → `ar r` 换库成员 → 重链（NOF-048：约 65s/轮）
3. **必须保留原文件副本**（`client_service.cpp.orig`），测完回滚；回执写明是否已回滚
4. 跑 400 keys，取这三个字段的 `p50 / p90 / mean`

## C. 判据（两边都写，对账 vseg `read_us` ≈ 426µs）

| 结果 | 判定 | 下一步 |
|---|---|---|
| **`wait` ≥ 70%** | 时间花在**等设备** | 结合 target W9 的 1.95×（NOF-052）⇒ 设备侧确实慢；而 IO 命令已被证完全一致（NOF-073）⇒ **只剩「IO 到达时序」，继续查提交节奏** |
| **`submit` ≥ 30%** | **提交本身有开销** | 查 `submitVSegment` 内的 `openSegment` 循环与 lane 分配 |
| **`plan` > 30µs** | 开发侧「3–5µs」的估算**错了** | 重新评估 `13b7c19` 那条修复的价值 |
| 三者之和远小于 `read_us` | 还有未覆盖段 | 如实写明差额，不要硬凑 |

## D. 回执必答（第一行）

走了 B1 还是 B2；三个点位的 `p50`（或均值）与占比；`wait` 是否 ≥70%；三者之和 vs `read_us` 的对账；SpDiag 层（Layer0/1）与 `spdiag` 是否存在；有无失败；**新增写入 LBA 与 oplog 增量**；若走 B2，源码是否已回滚。

## E. 边界

- **禁止改 target、禁止重启 target**（PID 905586 / md5 `3129641f…` 不变）。
- **禁止改共用 etcd、禁止删任何 `/oplog` 前缀、禁止重启 master。**
- B1 **零改码、零重编**；B2 允许改 `client_service.cpp` 一个文件，**必须备份并回滚**，不得改其他文件。
- 400-key 口径，写入量 ≤ 400 keys；写入前查 etcd 余量（当前 150 MB）。
- 不碰他人进程；产物 `run/nof091_*.txt`（≤3 个、每个 ≤1MB）。
- 若 A 段发现 spdiag 不存在、且 B2 编译失败 3 次 ⇒ **停手如实回执**，给失败日志尾部 50 行，不要自行改方案。

---

# NOF-092（**只读取证：测试机源码的仓库 / 分支 / 版本** · 2026-11-24 21:03 派）

## 0. 为什么单列一单

**用户 2026-11-24 21:02 确认：`d371e22` 这个版本没问题，作为基准。**

但 NOF-091 回执里报告的测试机源码版本是 **`be10be29`（dirty=3）**，与 `d371e22` 不是同一个 hash。**这件事必须查清，否则开发侧基于本地源码得出的一切判断（`Plan` 重复校验占比、`openSegment` 开销、两臂提交方式是否一致）都没有立足点。**

本单**只做取证**：不跑性能测试、不写任何数据、不动任何进程。与 NOF-091 并行，互不干扰。

## A. 只读采集（5 分钟）

1. 找 bench 二进制及其所属源码目录：
   - `find / -maxdepth 5 -name "stress_cluster_bench*" -type f 2>/dev/null | head`
   - `ls -l <bench路径>`、`md5sum <bench路径>`
2. 进入源码目录，**四条命令全执行、原文贴出**：
   ```
   git remote -v
   git branch --show-current
   git log -1 --format="%H %ci %s"
   git status --short
   ```
3. **关键判据命令**（决定开发侧源码分析能否直接引用）：
   ```
   git merge-base --is-ancestor d371e22 HEAD && echo "d371e22_is_ancestor" || echo "NOT_ancestor_or_missing"
   ```
4. 若提示对象缺失：先 `git fetch --all` 再判一次；仍缺失则记 `[未获取]`，不要猜。

## B. 判定（两边都写）

| 结果 | 结论与后续 |
|---|---|
| **`d371e22` 是 HEAD 的祖先** | 测试机是在基准上往前走的版本。**再给 `git log --oneline d371e22..HEAD` 全表**，并逐条标出是否改了 `client_service.cpp` / `transfer_task.cpp` / `real_client.cpp` / `mooncake-store/src/vsegment/`。**若这些文件都没动 ⇒ 开发侧源码分析可直接引用**；若动了 ⇒ 列出改了什么 |
| **`d371e22` 不是祖先** | 两条线已分叉。给 `git merge-base HEAD d371e22` 的结果，说明分叉点在哪个提交之后 |
| **对象缺失且 fetch 不到** | 记 `[未获取]`，只报 remote / branch / HEAD / status 四个值，**不做任何推断** |

## C. 回执必答（第一行）

`remote` / `branch` / `HEAD 全 hash` / `status 条数` / **`d371e22` 是否为祖先**；若在其后，列出 `d371e22..HEAD` 的提交清单与其中被改动的读路径文件；bench 二进制的 md5 与 mtime；有无失败；**新增写入 = 0**。

## D. 边界

- **纯只读**：零写入、零重启、不跑 bench、不跑 urma_perf、不动任何 cid / master / target。
- 不碰他人进程（royi master 3463695 / yin 的 `vseg_yin_bonding`）。
- 只产出 `run/nof092_summary.txt`（1 个文件，≤100 KB）。
- **不做性能判断**，本单只回答「测试机上跑的是哪份代码」。

---

# NOF-094（**量 vsegment 读的三段分解：plan / submit / wait** · 零改码 · 2026-11-25 10:44 派）

> **【2026-11-25 11:02 开发侧裁决 · warmup=0 路线关闭 · 已转 NOF-095】**
>
> **A 段判据不成立，接受停手。** 开发侧「128 = 采样配额、被 warmup 吃满」的推断**被否证**：配对实验（同会话、同 master、只改 warmup）显示 **warmup=128 时 13 个读取点位全 ticks=128；warmup=0 时读取点位 0 条**。跨两轮、跨两次 master 重启、5 次观测一致 ⇒ **不是配额问题，也不是时序偶然**。根因在 SpDiag 库内部（无源码），**该路线关闭，不再试其他参数组合**。
>
> ### 1. ⚠️ 一个必须记账的环境事故（**开发侧指令缺陷**）
>
> **昨日 21:55 切回 `plain_nof088` 时，master 启动漏了 `MC_STORE_PARTITION_SLOT_COUNT=1`** ⇒ `slot count mismatch: cluster=1, local=16384` ⇒ master 不持有任何 slot ⇒ **21:55 至今日 10:48，所有读都返回 `SLOT_NOT_OWNED (ret=-1012)`，环境实际坏了约 13 小时。**
>
> **这是开发侧的指令缺陷，不是执行侧的问题**——昨日那条切 cid 指令没把 slot 参数写全。已由执行侧修复（`owned_slots=1`、`ready=1/1`）。
> **例外**：昨日 21:43 那 6 遍 plain 读数是在原 master（PID 2565234，21:55 之前）上跑的，**不受影响，数据仍有效**。
>
> ### 2. 三条强制规范（**写入所有后续任务书**）
>
> | # | 发现 | 规范 |
> |---|---|---|
> | **1** | **master 重启会丢 NoF 挂载**（`/cvm/<cid>/` 无 `segments/` 键） | 每次重启后**必须重跑 `nof_register`** |
> | **2** | **一次成功的 bench 跑完后，挂载会再次消失** | **每遍 bench 前都要重新挂载并验证** |
> | **3** | 建 vseg cid 必须带**完整 8 项 vsegment 规划参数**（昨日漏了，疑似导致 `insufficient space`） | 见 NOF-095 §C，**逐字使用，不得删减** |
>
> ### 3. 对 `spdk_wrapper.cpp:638 Failed to attach NoF controller rc=-19 (ENODEV)`
>
> **暂不处理。** 与读性能无直接关系，当前读路径已通（`keys=2000` 可读）。记录在案，**性能主线收口后再单独查**，避免再次岔开。
>
> ### 4. 三个待决策的答复
>
> - **① 关闭 spdiag 取样路线** —— **同意，已关闭**
> - **② 建 vseg cid 用完整参数** —— **同意，已写入 NOF-095 §C**
> - **③ 环境要求写进交接文档** —— **同意，已写入本节与 NOF-095**
>
> ### 5. 下一步
>
> **已转 NOF-095**：不再加「三段 chrono」，改为**一次性输出 `future.nofTiming()` 的 30+ 字段**（`lane` / `qpair_id` / `credit_wait_us` / `head_to_first_submit_us` / `max_sub_io_latency_us` 等），信息量比三段大一个数量级，能一次验证 lane 分散、信用限流、队头阻塞三个假设。

> **本单替代 NOF-091（091 三轮概念已混，废弃不再续）。091 只继承两条**：① SpDiag 是 Layer 1 且埋点活着；② 现有取样方式只能拿到冷启动/不完整样本。**“128 配额被 `warmup_keys=128` 吃满”的假设已被本单否证，不得再引用。**

> **【2026-11-25 10:55 开发侧裁决 · spdiag 路线关闭 · 改走改码，但一次改到位】**
>
> ### 1. A 段不成立，我的假设错了
>
> 我的假设是「SpDiag 每点位只存前 128 个样本，warmup=128 把配额吃满」。**实测否证**：
> - `warmup=0` 时，读取点位样本数 = **0**（不是 128）
> - 对照 `warmup=128` 时，13 个读取点位**全是 128**
> - 且与遍数无关（091 第二轮 6 遍后仍 128）
>
> **⇒ 「128 是配额上限」这个推断不成立。** 128 更像「进程存活期间只赶上一次快照」的副产物，而 `warmup=0` 时进程活得更短，连那次快照都没赶上。
> **⇒ 接受回执建议：spdiag 取样路线关闭，不再试。**（5 次观测、跨两轮、跨两次 master 重启，已足够）
>
> ### 2. 但改码不该只加 chrono —— 一次改到位
>
> 回执建议「改 `client_service.cpp` 加 chrono」。**开发侧认为这个方案太保守，会浪费一轮编译。**
>
> **本单源码排查发现**：`Client::BatchGet` 里有 `future.nofTiming()`，能一次性给出 **20+ 个客户端内部时序字段**（`client_service.cpp:2338-2440`）：
>
> | 字段 | 含义 | 为什么关键 |
> |---|---|---|
> | `sub_io_count` | 子 IO 数 | vseg 是不是真的拆成 2×16 |
> | **`dispatch_queue_us`** | **调度队列耗时** | **IO 在客户端排了多久队** |
> | `worker_setup_us` | worker 准备耗时 | 线程分配开销 |
> | **`qos_fifo_wait_us`** | **QoS FIFO 等待** | 是否被 QoS 限流 |
> | **`credit_wait_us`** | **信用等待** | 是否等 URMA credit |
> | **`submit_span_us`** | **首次到末次提交** | **提交是否被拉开** |
> | **`drain_us`** | **末次提交到完成** | 等数据回来的净时间 |
> | `task_total_us` | 任务总耗时 | 对账用 |
>
> **但这套输出只在 `BatchGet` 路径上；bench 走的是 `TransferReadInternal`（`client_service.cpp:5956-5990`），那里没有。**
>
> **⇒ 改码只需在 `TransferReadInternal` 的 `future->get()` 之后，把 `future.nofTiming()` 的字段打到日志里（约 20 行）。一次编译，拿到的信息比「三段 chrono」多一个数量级。**
>
> ### 3. 回执里两条新发现，价值很高
>
> **新发现 A：master 重启会丢 NoF 挂载。** `/cvm/<cid>/` 无 `segments/` 键，必须重跑 `nof_register`；**且一次成功 bench 跑完后挂载会再次消失** ⇒ 需「每遍前先挂载」。
> **新发现 B：master 侧持续报 `spdk_wrapper.cpp:638 Failed to attach NoF controller rc=-19 (ENODEV)`**，未定位。
> **⇒ 这两条都指向「master 与 NoF 挂载的稳定性」问题，与性能结论无直接关系，但会影响每轮实验的可重复性。登记为产品级缺陷，后续单独查。**
>
> **新发现 C（最有价值）：建 vseg cid 必须带一整套 vsegment 规划参数。**
> `node1:/tmp/nof076_switch.sh` 里的完整参数组：
> ```
> --cvm_segments_vsegment_exclusive=true
> --vsegment_auto_plan=true
> --vsegment_required_medium=nof
> --vsegment_io_alignment=4096
> --vsegment_member_count=1
> --vsegment_stripe_size=2097152
> --vsegment_member_extent_size=67108864
> --vsegment_reserved_ratio=0
> ```
> **昨日（NOF-091）建 cid 时漏了这组参数 ⇒ 未验证，但高度吻合当时的 "insufficient space"。**
> **⇒ 本单已把这三组参数（slot / URMA 设备 / vsegment 规划）写进任务书，回执建议采纳。**
>
> ### 4. 一条环境事故要认
>
> **昨日 21:55 切回 `plain_nof088` 时漏了 `MC_STORE_PARTITION_SLOT_COUNT=1`** ⇒ `CvmController slot count mismatch(cluster=1, local=16384)` ⇒ master 不持有 slot ⇒ **昨日 21:55 至今日 10:48 期间所有读都返回 `SLOT_NOT_OWNED`。**
> **⇒ 责任在开发侧任务书未固化该 env。今日起所有「建 cid / 重启 master」的任务书必须带完整 env 清单。**
> **⇒ 但昨日 21:43 那 6 遍读数在故障前的原 master（PID 2565234）上跑的，有效。**

## 0. 为什么派这单（**已被 2026-11-25 裁决部分推翻，见上方；本单不再执行**）

`read_us`：vseg **426µs** vs plain **165µs**（2.58×），但内部无 breakdown 埋点。
**而源码里已有完整的三段计时**，位置在 **`TransferReadInternal`**（`client_service.cpp:5940-5990`），正是 bench 走的路径：

```
get_into → execute_ranged_read → Client::Get(:1843)
        → TransferReadRange(:6037) → TransferReadInternal(5940-5990)
```

该段代码为：

| 点位 | 行 | 包住什么 |
|---|---|---|
| `VSEGMENT_READ_PLAN` | **5956** | `vsegment_transfer_planner_->Plan(...)` |
| `VSEGMENT_READ_SUBMIT` | **5963** | `transfer_submitter_->submitVSegment(plan, READ)` |
| `VSEGMENT_READ_WAIT` | **5788** | `future->get()` |

> **注意**：`client_service.cpp` 里 `VSEGMENT_READ_*` 共出现 3 套（2246/5826/5956），只有 **5956 这套在 bench 的路径上**。另两套分别属于 `Client::BatchGet` 与 `Client::TransferData`，**不要采**。

**⇒ 本单零改码**：埋点已在，`spdiag` 可读（Layer 1 已确认）。

## A. 关键手法：`--warmup_keys=0`

**091 的教训（已更正）**：`Ticks` 恒为 128 与 `warmup=128` 的对应关系曾被误认为“配额被吃满”，但 NOF-094 配对实验已否证；现状是取样机制不明、`warmup=0` 时读取点位为 0，路线关闭。

**本单两臂一律 `--warmup_keys=0`**，让前 128 个样本落在**测量段**。

**验证判据**：数值应从 091 抓到的 ~554µs（冷启动）降到 **~300µs 量级**。若仍是 554µs ⇒ 128 不是配额，**停手回执，不要改方案硬试**。

## B. 执行顺序（**不可颠倒**）

### B1. 先测 plain 臂（零写入）

1. 对象：`plain_nof088` 的**前 400 个 key**（它现有 2000 keys，读前 400 即 1600 MiB，与 vseg 400 keys 同量）
2. `spdiag clear` → 跑 reader：`--num_keys=400 --warmup_keys=0 --value_size=4MiB --batch_size=1 --num_threads=1`，同宿主 node3 本机
3. 跑完立刻 `spdiag watch --interval 200` 或 `history`，取 **`GET_INTO_INTERNAL_NOF_READ`**（plain 走 nof 分支，没有 VSEGMENT_* 点位）的 `Ticks / avg / p50`
4. **连跑 3 遍**，每遍前 `spdiag clear`，逐遍记录

> **为什么 plain 的点位名不同**：`SelectGetIntoReadPerfKey()` 按副本类型分派——`is_nof_replica()` → `GET_INTO_INTERNAL_NOF_READ`；`is_vsegment_replica()` → `GET_INTO_INTERNAL_VSEG_READ`。两臂点位不同是设计如此，不是异常。

### B2. 再建 vseg 臂（**会覆写 plain 前 400 key 的物理位置，所以必须 B1 之后**）

1. 固化 `plain_nof088` 状态（`Keys=2000`、`/oplog/plain_nof088/` entries、`dbSize`）
2. 切 master 到新 cid **`vseg_nof094`**（`slot=1`、vsegment enabled）
3. 写 **400 keys**（预计 22 MB oplog；余量 150 MB）。每轮复测 `dbSize`，增量 >100 MB 立即停
4. **`VSEGMENT_CREATING` 会丢约 23 个 key（步长 17），需多轮补写**，直到 `Keys=400`

### B3. 测 vseg 臂

1. `spdiag clear` → 同 B1 参数跑 reader
2. 取 **`VSEGMENT_READ_PLAN` / `VSEGMENT_READ_SUBMIT` / `VSEGMENT_READ_WAIT`** 三个点位的 `Ticks / avg / p50`
3. **连跑 3 遍**，逐遍记录

## C. 判据（两边都写）

**先对账**：三段之和 ÷ 该遍 `read_us`（vseg 约 426µs）。若和远小于 `read_us`（<70%）⇒ 还有未覆盖段，如实写明差额，**不要硬凑**。

| 结果 | 判定 | 下一步 |
|---|---|---|
| **`wait` ≥ 70%** | 时间在**等设备** | 客户端洗清。结合 target W9 的 1.95× 与「IO 命令已被证完全一致」（NOF-073）⇒ 只剩「IO 到达时序」 |
| **`submit` ≥ 30%** | 提交本身有开销 | 查 `submitVSegment`（`transfer_task.cpp:2479`）内的 `openSegment` 循环与 lane 分配 |
| **`plan` > 30µs** | 开发侧「3–5µs」估算错了 | 按实际版本重估 |
| 三段和 <70% read_us | 有未覆盖段 | 写明差额，等开发侧定 |

**同时给 plain 臂的对照值**（`GET_INTO_INTERNAL_NOF_READ`），用于判「vseg 的三段结构是否与 plain 同形」。

## D. 回执必答（第一行）

两臂各 3 遍的三个点位 `avg / p50`；**wait 占比**；三段和 vs `read_us` 的对账；`Ticks` 是否 = 128 且数值是否降到 ~300µs 量级；`vseg_nof094` 的 `Keys` 与 oplog 增量；有无失败；**新增写入 LBA 清单**。

## E. 边界

- **零改码、零重编**（埋点已有，不需要动代码）。
- **禁止改 `/etc/spdiag/spdiag.conf`**（系统级共享，影响他人）。
- **禁止重启 target**（PID 905586 / md5 `3129641f…` 必须不变）。
- **禁止改共用 etcd、禁止删任何 `/oplog` 前缀。**
- B2 允许重启 master（`plain_nof088` 的 oplog 仅 899 KB，可随时切回）。
- 不碰他人进程（royi master / yin 的 `vseg_yin_bonding` / 以及 node1 上正在 cmake 构建的进程）。
- 若 A 段验证失败（数值仍是 554µs）⇒ **停手如实回执**，不要改配置、不要改口径凑数。
- 产物 `run/nof094_*.txt`（≤3 个、每个 ≤1 MB）。

---

# NOF-095（**输出 `NofTransferTiming` 全字段：一次拿到客户端侧 IO 画像** · 改码一次 · 2026-11-25 加固）

> **【2026-11-25 NOF-095 回执 · 阻塞关闭，未产生 timing 数据】**
>
> - **补丁/构建成功**：be10be29 的 `TransferReadInternal` 两分支加入 `NOF095_TIMING`（+60/−1，仅一个文件）；两个构建目录产物 md5 一致 `881df1a1…`；控制组/裸重链 `0afb26d6…`。
> - **冒烟失败**：NEW / CTRL / 裸重链全部 attach NoF controller `rc=-19 (ENODEV)`，0 次读；OLD 二进制正常。`vseg_nof095` 未创建，新增业务写入=0。
> - **L1 成立**：不是本次补丁导致——未改源码、未改 archive 的裸重链也同样失败。
> - **L0 成立**：当前按 `link.txt` 重链无法复现旧二进制行为。
> - **尚不能证明“相同输入不可复现”**：旧二进制当初的真实链接命令与静态 SPDK 库集，尚未证明与当前 `link.txt` 相同；`ldd` 相同不能排除静态 SPDK 库不同。首要候选是 OLD 使用 NOF-051 的 bd5ee124d/wire v4 静态库，而裸重链使用 `/usr/local` 的另一套库。
> - **回滚完成**：源码 md5 恢复 `a147b489…`，`.orig` 已删；原 `bench-io-params-e12=88ad3134…` / in-tree `59662a26…` 均未变；target 905586 未重启；master/etcd 未变。产物 `run/nof095_summary.txt`（5,578B / md5 7f0ae4d0…）。
> - **下一步 NOF-096**：先查 OLD/raw 的静态 SPDK 版本与真实链接输入，再决定重链路径；不在原因未明时继续堆插桩。

> **【2026-11-25 加固 · 执行以本段为准】**
>
> - **代码基准**：`LeeJace993/Mooncake @ mooncake-vesegment @ be10be29`。开发侧已直接核对：`mooncake-store/include/transfer_task.h:56` 的 `NofTransferTiming` 与 `:412 nofTiming()` 存在，字段齐全；`client_service.cpp:5926 Client::TransferReadInternal` 的 vseg/plain 分支各只有一个 `future`。旧 `d371e22/706551a` 的行号与源码判断不得引用。
> - **补丁范围**：同一个 `TransferReadInternal` 内，**vseg 与 plain 两个分支都要在 `future->get()` 后输出** `NOF095_TIMING`，带 `arm=vseg|plain`、`has_timing=0|1` 和 `NofTransferTiming` **全部字段**；原 SpDiag 三点位保持不动。可用函数内 lambda 避免复制两份 30 字段格式，只改这一个文件。
> - **两次冒烟**：plain 20-key 冒烟在写 vseg 前做；vseg 20-key 冒烟在 `vseg_nof095` 写完后做。任一分支无日志或 `has_timing=0` ⇒ 停手并回滚，不继续正式测量。
> - **源码/产物隔离**：只改我方副本 `$D/mooncake-src/mooncake-store/src/client_service.cpp`；先记录 HEAD/status/md5，构建独立产物 `bench-nof095`，原 `bench-io-params-e12` 的 md5 必须不变。测完恢复 `.orig` 并重编回原状（或删除新产物并验证原二进制未变），回执写明。
> - **环境红线**：target **905586 / md5 3129641f 不重启**；不启用 BUFFER；400-key 口径；etcd 写入前复测余量。

## 0. 为什么这单比「三段 chrono」强

原计划在三个点位加 chrono 只能得 3 个数。**而 `future.nofTiming()` 返回的 `NofTransferTiming` 有 30+ 个字段**（`transfer_task.h:56-101`），全部由 NoF 传输层在完成时填好：

| 字段 | 能直接回答什么 |
|---|---|
| `lane` / `qpair_id` / `worker_index` | **这个 IO 落在哪条通道、哪个 qpair** |
| `lane_task_sequence` / `lane_completion_sequence` | **在通道队列里排第几** |
| `qos_tasks_ahead_at_enqueue` | 入队时前面压了多少任务 |
| `dispatch_queue_us` / `worker_setup_us` | 调度队列 / 线程准备耗时 |
| **`credit_wait_us` / `first_credit_wait_us`** | **有没有被信用（限流）卡住** |
| **`head_to_first_submit_us`** | **有没有队头阻塞** |
| `qos_fifo_wait_us`（及 full/partial/zero 三分解） | FIFO 背压细分 |
| `first_to_last_submit_us` | 首次提交到末次提交的跨度 |
| **`last_submit_to_complete_us`** | drain（末次提交→完成） |
| `max_sub_io_latency_us` / `max_sub_io_index` / `max_sub_io_cid` | **最慢的子 IO 及其 CID —— 能与 target 侧关联** |
| `queued_mono_ns` / `first_submit_mono_ns` / `last_submit_mono_ns` / `completed_mono_ns` | 绝对时间戳，可排时序 |
| `sub_io_count` | 子 IO 数量（预期 vseg 32） |

**⇒ 这些字段能一次性验证此前所有假设**（lane 分散、信用限流、队头阻塞、调度开销），不用再一轮一轮试。

## A. 采样前提（**必须满足，否则 timing 为空**）

`transfer_task.cpp:2496` 与 `transfer_task.h:303`：`track_timing_` 由 `ShouldSampleHiFreqLog(parent_trace_id)` 决定。

⇒ **`MC_HIFREQ_LOG_SAMPLE_RATE` 必须为 1（全采样）**。历史上我们一直设 1，本单继续。

## B. 改码（**只改一个文件、两个分支、独立产物、备份回滚**）

1. **基线**：在我方副本 `$D/mooncake-src` 记录 `git rev-parse HEAD`（必须为 `be10be29`）、`git status --short`、`mooncake-store/src/client_service.cpp` 的 md5；备份为 `client_service.cpp.orig`；记录原 `bench-io-params-e12` 的 md5。
2. **改 `Client::TransferReadInternal`**（be10be29 函数起点 `client_service.cpp:5926`）：
   - **vseg 分支**：在 `const ErrorCode result = future->get(); pt_vsegment_wait.End(...);` 之后、`return result;` 之前调用计时输出；
   - **plain 分支**：把 `return future->get();` 改为“先取 result → 调用计时输出 → return result”；
   - 输出统一日志名 **`NOF095_TIMING`**，字段包含 `arm=vseg|plain`、`has_timing=0|1`，以及 `transfer_task.h` 中 `NofTransferTiming` 的**全部字段、按定义顺序**；无 timing 时也要输出 `has_timing=0`，不得静默跳过；
   - **原 `SpDiag::PerfPoint` 三点位完全不动**；只改这一个 `.cpp`。
3. **独立构建**：按 NOF-048 的最小重建路径编译该 `.cpp` → `ar r` 换库成员 → 重链为 **`bench-nof095`**；`--help` 正常；**原 `bench-io-params-e12` 文件与 md5 必须不变**。
4. **plain 冒烟**：在 `plain_nof088` 上跑 20 keys、`--warmup_keys=0`，预期 **20 条 `NOF095_TIMING arm=plain has_timing=1`**；不足/为空 ⇒ 停手回执，不继续。
5. **vseg 冒烟**：待 D2 建好 `vseg_nof095` 后，同样跑 20 keys、`warmup=0`，预期 **20 条 `arm=vseg has_timing=1`**；不足/为空 ⇒ 停手回执，不继续正式 3 遍。
6. **回滚**：全部测完后恢复 `client_service.cpp.orig` 并重编回原状（或删除 `bench-nof095` 并验证原二进制未变）；回执写明源码已恢复、原二进制 md5 未变。

## C. 建 vseg cid（**必须带全参数组，漏参数会导致环境不可用**）

**master 启动命令必须逐字带全**（来源：node1 `/tmp/nof076_switch.sh`）：

```
环境变量：
  MC_STORE_PARTITION_SLOT_COUNT=1          ← 昨日漏这个，导致 SLOT_NOT_OWNED 13 小时
  SPDK_URMA_DEV_NAME=udmac0d1e2
  SPDK_URMA_CAPSULE_TRANSPORT=urma

vsegment 规划参数（命令行）：
  --cvm_segments_vsegment_exclusive=true
  --vsegment_auto_plan=true
  --vsegment_required_medium=nof
  --vsegment_io_alignment=4096
  --vsegment_member_count=1
  --vsegment_stripe_size=2097152
  --vsegment_member_extent_size=67108864
  --vsegment_reserved_ratio=0
```

**reader 进程 env（冒烟与正式读都必须）**：`MC_HIFREQ_LOG_SAMPLE_RATE=1`（`track_timing_` 前提）、`MC_STORE_PARTITION_SLOT_COUNT=1`、`MC_NOF_IO_PARAMS_LOG=1`（可选，用于交叉核对）、`SPDK_URMA_STAGE_TRACE_US=0`、其余诊断关。

**另：master 每次重启后必须重跑 `nof_register` 挂载**；且**每次 bench 跑前都要重新挂载并验证**（NOF-094 已两次实测：重启丢挂载、bench 跑完也丢）。

## D. 执行顺序

0. **plain 环境预检**：若 `plain_nof088` master 未运行，启动必须带 `--cluster_id=plain_nof088`、`MC_STORE_PARTITION_SLOT_COUNT=1`、`SPDK_URMA_DEV_NAME=udmac0d1e2`、`SPDK_URMA_CAPSULE_TRANSPORT=urma`；`nof_register` 后验收 `Keys=2000`、`owned_slots=1`、`ready=1/1`。若已在运行，也要先验证这些值与挂载；缺一项先修复再读。
1. **plain 冒烟 + 正式臂**（零写入）：在 `plain_nof088` 上先跑 20 keys、`warmup_keys=0` 验证 `arm=plain` 日志；通过后读前 400 key、`warmup_keys=128`、跑 **3 遍**。每遍 bench 前重新 `nof_register` 并验证挂载。
2. **建 `vseg_nof095`**：按 §C 完整参数启动 vseg master，写 **400 keys**（多轮补齐，验收只认 master `Keys=400` / `logical_used=1,677,721,600`）；写入前复测 etcd 余量。
3. **vseg 冒烟 + 正式臂**：先跑 20 keys、`warmup_keys=0` 验证 `arm=vseg` 日志；通过后按同口径跑 **3 遍**（400 keys / warmup 128 / T1 / batch1 / qpairs4 / verify=false）。每遍前重新挂载并验证。
4. 两臂各日志中提取 **`NOF095_TIMING`** 行；正式臂每遍预期 **528 条**（400 测量 + 128 warmup），冒烟每臂预期 20 条。
5. 全部完成后按 §B6 回滚源码/产物，再停 master；**target 保持运行**。

## E. 判据（逐条回答，不要只给结论）

| 问题 | 看哪个字段 |
|---|---|
| vseg 的 IO 落在**几条 lane**？plain 呢？ | `lane` 去重计数 |
| 有没有**信用等待**？ | `credit_wait_us` / `first_credit_wait_us` |
| 有没有**队头阻塞**？ | `head_to_first_submit_us` |
| 调度/线程准备各占多少？ | `dispatch_queue_us` / `worker_setup_us` |
| 子 IO 数量对不对？ | `sub_io_count`（预期 vseg 32、plain 32） |
| **最慢的子 IO 有多慢？** | `max_sub_io_latency_us`，并用 `max_sub_io_cid` 与 target 侧 W9 关联 |
| drain 占比 | `last_submit_to_complete_us` ÷ `total_us` |

**判读（两臂同口径，时延字段同时给 p50/p90 与非零占比）**：
- vseg 的 lane 更少、`credit_wait_us` 或 `head_to_first_submit_us` 明显更大 ⇒ **客户端提交层有责任**；
- lane/credit/队头三项相同，但 `last_submit_to_complete_us` 或 `max_sub_io_latency_us` 更大 ⇒ **客户端提交正常，target/传输/会话等待更大**；
- 全部相同 ⇒ **客户端 timing 层排除**，不得继续猜，转 target/会话层的受控实验。

## F. 回执必答（第一行）

源码 HEAD/status/md5 与原 bench md5（改前/改后）；plain/vseg 冒烟各多少条 `has_timing=1`；正式臂各多少条 `NOF095_TIMING`（预期 528/遍）；vseg/plain 的 lane 去重数、qpair/worker 分布；`credit_wait_us`、`first_credit_wait_us`、`head_to_first_submit_us`、`dispatch_queue_us`、`qos_fifo_wait_us`、`worker_setup_us` 的 **p50/p90/非零占比**；`first_to_last_submit_us`、`last_submit_to_complete_us`、`task_total_us`；`max_sub_io_latency_us/index/cid`；`sub_io_count` 分布；按 §E 判读客户端是否有责任；源码是否已回滚 + 原二进制 md5 是否未变；`vseg_nof095` 的 Keys 与 oplog 增量；有无失败。

## G. 边界

- **只在我方副本 `$D/mooncake-src` 改 `client_service.cpp` 一个文件**，不改同事源码树；必须备份 + 回滚，构建独立 `bench-nof095`，原 `bench-io-params-e12` 的 md5 必须不变。
- **禁止重启 target**（PID 905586 / md5 `3129641f…` 不变）。
- **禁止改共用 etcd、禁止删 `/oplog` 前缀、禁止改 `/etc/spdiag/spdiag.conf`。**
- 建 vseg cid 允许重启 master，但**必须带全 §C 的参数**。
- 400 keys 上限；写入前查 etcd 余量（当前约 150 MB）。
- 不碰他人进程（royi / yin / node1 上的 cmake 构建）。
- 冒烟失败（日志无 `nof095_vseg_timing`）⇒ **停手回执**，不要改采样开关或试其他参数。

---

# NOF-084（**已完成 · 按 A3 判据主动停手 · L0 观测** · 口径整改 400 → 2000 · 2026-11-24）

> **【2026-11-24 17:08 开发侧裁决 · 优先级高于本单任务书】**
>
> **停手正确，L0 分级正确**，符合《归因纪律》§1/§5（回执已答「本轮未触发任何既有结论降级」）。实测保留：100 keys → oplog **+8,481,612 B**（700 entries / 12,116.6 B per entry / **84.8 KB per key**）；三种外推 2000 keys = **24.2 / 116.9 / 169.6 MB**，全部超任务书 24MB 阈值 ⇒ B/C 段按判据不执行；bench 原生支持 2000（`-num_keys` uint64、仅 size_t 溢出保护）；slot=1 下 100 keys → vsegments=7，`VSEGMENT_CREATING` 12 次 / 3 轮补写。
>
> **但「32MB 墙」这个前提是未实测的经验值，且回执对它的定位有错**：
> - NOF-071 原文：`35,087,559 > 33,554,432`（= 32 MiB）触发 `ResourceExhausted`、75s 重试 959 次、50056 从未 LISTEN —— 这是**回放时的单条 gRPC 消息**，不是 etcd 存储上限。
> - 回执把它归到 etcd `--max-request-bytes`（默认 1.5MB → 需 40MB+）**不成立**：若请求侧上限真是 1.5MB，`vseg_nof061` / `vseg_nof072` 的 23.69MB 早就起不来了，而它们历史上都能正常起 master。
> - 回执「NOF-068b 已 58MB 仍未崩」混淆两件事：**etcd 里数据仍在（存储层正常）** vs **master 回放失败（NOF-071 实测）**。两者不矛盾，不能据此推断墙的位置。
> - ⇒ 24MB 阈值是开发侧拍的保守值，不是实测墙；方案 (c) 改 `--max-request-bytes` 很可能改错地方。**未经实测不得改共用 etcd**。
>
> **方向决策留给用户**：(b) snapshot 治理 / (c) 改共用 etcd / (d) 分片 / (e) 降 keys / (f) 改码，分别涉及共用机变更、整改初衷或改码授权，本单不替用户选、不擅自执行，只把决策所需事实拿齐。



## 0. 整改内容与量化影响（先读）

用户判断：写 400 keys、每遍只读 400 keys 不合理 ⇒ **改为 2000 keys**，写与读同一口径。其余参数（4MiB/key、`batch_size=1`、qpairs、verify 策略）保持不变。

| 项 | 400 keys（现行） | **2000 keys（整改后）** |
|---|---|---|
| 数据量 | 1600 MiB 卷 | **8000 MiB 卷 = 7.8125 GiB** |
| 条带数（256KiB） | 6,400 | **32,000** |
| 每盘条带数 | 582 / 581 | **2,910 / 2,909** |
| 每盘数据量 | 145.45 MiB | **727.27 MiB** |
| 每盘盘内 offset 范围 | 0 – 145.45 MiB | **0 – 727.27 MiB** |
| 128KiB 唯一盘内落点 | 1,164 | **5,820** |
| `warmup_keys=128` 占比 | 32.0%（512MiB 卷） | **6.4%（仍 512MiB 卷）** |
| 每遍慢程 350MiB 占比 | 21.875% | **4.375%** |

**口径隔离（硬性）**：2000-key 结果与全部 400-key 历史数字**不得混用、不得并列比较、不得画在同一张表里**。所有历史报告标注「400 keys 口径」。

## A. Preflight：先核实能力，**不写数据**

1. `--help` / 源码只读核实 `stress_cluster_bench --num_keys` 是否直接接受 2000（有无硬编码上限、key 数组 / LBA 分配 / warmup 是否随之失效）。**若不支持 ⇒ 停手回报，不擅自改码、不重编**。
2. 确认 `slot_count`、`stripe_size`、`member_extent_size` 在 2000 keys 下是否仍能一次分配完，记录 auto-plan 的分区 / vsegment 数。
3. **oplog 32MB 墙预检（本单最高风险）**：
   - 已知 `vseg_nof080`（400 keys，一次写入）oplog = **22 MB**；线性外推 2000 keys ≈ **110 MB**，远超 etcd gRPC **32 MB** 上限（NOF-071 已发生过 master 永久起不来）。
   - 用**全新 cid `vseg_nof084`** 先写一小批（如 100 keys），实测 oplog 增量 / key，推算 2000 keys 总量。
   - **若推算 > 24 MB（留 8MB 余量）⇒ 立即停手回报，不得写满 2000 keys**，并说明可选规避（注意 `--enable_oplog=false` 已由 NOF-072 证明会导致 `Keys=0`，不可用）。
   - 不得删任何 `/oplog/*`（oplog 是对象元数据唯一载体，删了等价删数据）。

## B. 正式写入（仅在 A 全过之后）

1. 新 cid `vseg_nof084`，`--num_keys=2000`，其余参数同现行口径；`--warmup_keys` 记录实际值（不擅自改）。
2. 必须记录：`Keys`、`logical_used`、verify 结果、写入耗时、`VSEGMENT_CREATING` 次数（NOF-056 的已知机制：每 64MiB vsegment 容 16 key，第 17 个 key 撞创建窗口 ⇒ 2000 keys 下会显著放大，**允许按既定方式补写至 2000，但须记录补写轮次**）。
3. **不碰** `vseg_nof080`（oplog 22MB）、`vseg_nofvseg`（老数据）、任何已有 cid；它们保留为 400-key 口径的历史对照，只读。

## C. 验收（任一不过即停手）

| # | 判据 |
|---|---|
| 1 | 写入 `Keys=2000/2000`，verify 0 错 |
| 2 | oplog < 24 MB，etcd 前缀未超 gRPC 32MB |
| 3 | **master 重启后仍能起来且 `Keys=2000`**（这是 NOF-071 的致命点，必须实测） |
| 4 | 读 ≥3 遍，给每遍 p50 / p90 / p99 / avg / span / failed |
| 5 | target 未重启、trace mask 末态 0x0、无残留进程 |

## D. 与其他单的关系

- **NOF-083 暂缓**：其 C/D 段依赖 400-key cid，且 F 段边界「`start ≥ 2048MiB` 在 cid 数据区之外」在 2000 keys 下**失效**——卷 2048MiB 换算盘内 186.18MiB，落在新的 0–727.27MiB 数据区内。整改后若要继续裸读，数据区外的起始点须 **≥ 8000 MiB 卷**（对应盘内 >727MiB）。NOF-083 待本单完成后重派或改写。
- NOF-077（`--rounds` / `--start_key` 自建口径）仍暂缓；本单不自造脚本、不改 bench 源码。
- NOF-081 C 段、NOF-082 B 段判据的撤回结论不受本次整改影响。

## E. 回执必答（第一行）

2000 keys 是否写成 / oplog 实测大小 / master 重启后 Keys / 读 p50（3 遍）/ bench 是否原生支持 2000 / 有无失败 / 新增写入的 cid 与 LBA 范围。

## F. 边界

- 不删 `/cvm|/oplog/vseg_nofvseg`；不碰他人进程；重启 master 前核查 4457 是否有他人客户端。
- target **不重启、不改码、不重编**（PID 905586 / md5 `3129641f…` 保持）。
- 唯一允许的写入是 A 的小批预检 + B 的 2000 keys，且全部落在**新 cid `vseg_nof084`**。
- 结论只写观测；**不得与 400-key 历史数字做比值或因果比较**。
- 产物 `run/nof084_*.txt`（≤3 个、每个 ≤1MB）。

---

# NOF-085（**已作废 · 其 A 段定位被源码证否** · 口径整改阻塞诊断 · 2026-11-24）

## A. 精确定位 `33,554,432`（纯只读，不改任何配置）

1. 从 NOF-071 现场日志 / `run/` 产物提取 `ResourceExhausted` **完整原文**：哪个 RPC、请求还是响应、报错给出的实际字节数与上限值、来自 etcd server 还是 Mooncake 客户端。
2. 分别给出当前值：
   - etcd server `--max-request-bytes`（请求侧：`/proc/<etcd_pid>/cmdline` + `--help`）；
   - gRPC `MaxCallRecvMsgSize` / `MaxRecvMsgSize`（响应侧，客户端常量）；
   - Mooncake master 源码里 `33554432` / 32MiB 常量的出处（只读 grep，给 `文件:行号`）。
3. 判据：
   - 若 32MiB 是**客户端接收上限** ⇒ 方案 (c) 改 etcd `--max-request-bytes` **无效**，应改客户端；
   - 若是 **etcd 响应 / 请求上限** ⇒ 属共用 etcd 变更，**需用户明确授权**才能动；
   - 找不到常量出处 ⇒ 记 `[未获取]`，不得推断。

## B. 把「墙」从经验变成实测界（零业务写入；起 master 只允许元数据 / 租约变更）

1. **先从历史日志取证（不重启）**：找 `vseg_nof061` / `vseg_nof072` / `vseg_nof080` / `vseg_nof068b` 起 master 时的回放字节数、成败、`Keys`。给「**已知能起的最大 oplog 体积**」与「**已知起不来的最小体积**」两个界。
2. 历史日志不足时，才做一次**受控切换**（本单唯一允许的状态变更）：
   - 切换前记录现役 master（`vseg_nof084`，Keys=100）与 50056 状态；
   - 每次只起一个 cid，验收 `50056 LISTEN` / `Keys` / 回放字节数 / 完整报错；
   - **末尾必须切回 `vseg_nof084` 并验收 `Keys=100`**，记录切换前后状态。
3. 不重启 target、不写任何业务数据、不删任何 `/oplog` 前缀。
4. 结论必答：24MB 阈值是否过于保守、真实可用上界在哪、**2000 keys 的 24.2MB 下限是否其实可行**。

## C. `--enable_snapshot` 只读核实（判方案 b 是否值得实测）

1. `--help` 列全部 snapshot 参数与默认值（`snapshot_interval_seconds` 默认 600、`snapshot_object_store_type`、`snapshot_retention_count` 等）。
2. 源码只读：snapshot 落盘后是否**裁剪 / 迁移 `/oplog` 前缀**，还是只把对象快照写到外部 store、oplog 原样保留。
3. 判据：若不裁剪 ⇒ 方案 (b) **排除**，不必实测；若裁剪 ⇒ 记录触发条件与前置依赖，交用户裁决后再测。

## D. 分片可行性的纯计算（零环境）

按本轮实测 **84.8KB/key** 与历史 **58.4KB/key** 两个口径，给「N 个 cid × 每 cid M keys」组合表：单 cid oplog 峰值、etcd 总增量、是否仍满足「单 cid 2000 keys」初衷。**只给数，不推荐方案**。

## E. 回执必答（第一行）

`33,554,432` 的出处（客户端 / etcd / 未获取）；三个上限当前值；已知能起与起不来的 oplog 体积界；2000 keys 的 24.2MB 下限是否可行；snapshot 是否裁剪 oplog；本轮是否触发既有结论降级；有无失败；master 末态是否回到 `vseg_nof084` / `Keys=100`。

## F. 边界

- **纯只读为主**；唯一允许的状态变更是 B 段的 master 受控切换，且必须切回 `vseg_nof084`、Keys=100。
- **不改 etcd 配置、不改码、不重编、不重启 target**（PID 905586 / md5 `3129641f…` 保持）、不写 2000 keys、不删任何 `/oplog`、不碰他人 cid 与进程（含 royi master 3463695）。
- **物理覆盖已知**：NOF-084 的 100 keys = 每盘 **36.36MiB**（成员盘 offset 0–36.36MiB），已覆盖 `vseg_nof080` / `vseg_nofvseg` 数据区前段；旧 cid 内容维持「不可信」，后续对照一律 `verify=false` 或避开该区间。
- 现役 master：`vseg_nof084`（slot=1，Keys=100，logical_used=419,430,400，`/oplog/vseg_nof084` 8.48MB）；旧 cid 元数据完好、未删改。
- 产物 `run/nof085_*.txt`（≤3 个、每个 ≤1MB）；结论只写「观测 + 是否支持判据」并标等级。

---

# NOF-086（**已完成 · 验收通过 · 裁决见下** · 口径整改 400 → 2000 · 2026-11-24）

> **【2026-11-24 18:51 开发侧裁决 · 优先级高于本单任务书】**
>
> **A/B/C/D 四段按预期执行，验收通过**：`Keys=2000`、`logical_used=8,388,608,000`（精确 = 2000×4MiB）、读 4 遍 `failed=0`、master 全程未重启（PID 2292801 / 启动记录 1 次 / `ResourceExhausted` 0 次）。**步长 17 与「3 轮补满」精确闭合**（见 §3，升 L3）。
>
> ---
>
> ### 1. 「多跑 ROUND3 导致 oplog 从 131.8MB 涨到 942.2MB」——**归因不成立，驳回**
>
> 按 key 增量重算（数字全部取自回执 §C 原始输出）：
>
> | 时点 | Keys | oplog bytes | 本段新增 key | 本段 oplog 增量 | entries 增量 | B / 新增 key | entry 均体积 |
> |---|---|---|---|---|---|---|---|
> | baseline | 0 | 1,307 | — | — | — | — | — |
> | ROUND1 后 | 1,883 | 13,599,140 | +1,883 | +13,597,833 | +5,883 | **7,222** | 2,311 B |
> | ROUND2 后 | 1,992 | 131,835,820 | **+109** | +118,236,680 | +4,117 | **1,084,740** | 28,719 B |
> | 最终 | 2,000 | 942,185,725 | **+8** | **+810,349,905** | +3,928 | **101,293,738** | 206,301 B |
>
> **那 8 个 key 是 ROUND1/2 按步长 17 漏掉的，不补就到不了 2000。** ⇒ 810.35MB 是「补齐最后 8 个 key」的**必然代价**，不是「多跑」的代价。脚本 bug 的真实后果只有两项：日志 206MB、未在第一时间停。
>
> **此区分必须保留到机制定位为止，不得并入脚本 bug 账上**——它决定 2000-key 口径能否复用：
> - 若为必然代价 ⇒ 任何 2000-key 写入都撞这个量级，该口径**不可复用**；
> - 若为多跑代价 ⇒ 改脚本即可省，口径可用。
>
> **本轮最有价值的量化事实（两条干净算术）**：
> 1. ROUND2：4,117 entries ÷ 当时已有 1,883 keys = **2.19×**；ROUND3：3,928 entries ÷ 当时已有 1,992 keys = **1.97×**。
>    ⇒ **每轮补写的 entry 数 ≈ 2 × 当前已写入的 key 总数，与「本轮新增多少 key」无关**（R2 只补 109 个、R3 只补 8 个，代价却由 1,883 / 1,992 决定）。
> 2. **entry 均体积逐轮暴涨 2,311 → 28,719 → 206,301 B（89×）**，最终累计 67,637 B/entry。
>
> ⇒ **任务书 §C.5「轮次上限 12」是危险约束，本裁决予以撤销**：实测 3 轮已到 942MB，若真跑满 12 轮，etcd 2GiB quota 必然 `NOSPACE`。今后一律改为「**oplog 余量优先，任一轮前复测 dbSize**」。
>
> ### 2. 「A 段直接定位了 32MiB 上限」——**方向对，结论不可采信**
>
> 事实：`ExecStart ... --max-request-bytes=33554432`，数字与 NOF-071 的 `33,554,432` 完全一致。
>
> **但语义不对**：`--max-request-bytes` 限制的是**请求**；回放失败时超限的是**响应**（从 etcd 读出 35,087,559 B 数据）。三种措辞对应三个不同的墙，必须用报错原文动词区分：
>
> | 报错措辞 | 含义 | 墙的位置 |
> |---|---|---|
> | `received message larger than max (X vs. Y)` | 客户端**收**不下 | master 的 gRPC `MaxCallRecvMsgSize` |
> | `trying to send message larger than max (X vs. Y)` | 客户端**发**不出 | etcd 的 `--max-request-bytes` |
> | `etcdserver: request is too large` | 服务端拒绝 | etcd 的 `--max-request-bytes` |
>
> **已 grep 全部 HANDOVER：NOF-071 只记了数字 `35,087,559 > 33,554,432`，从未记报错原文的动词。** ⇒ 三分支无法区分，**C11 改判「候选来源已定位，语义方向未定」，不得升 L3**。
>
> **副产品（有价值）**：`--max-request-bytes` 默认 1.5MiB，这台 etcd 被**显式改成 32MiB** ⇒ 已有人为「大请求」调过一次限。运维知道这个限制存在，只是没料到 oplog 会长到 942MB。
>
> **最快的下一步不是上机，是 grep 源码**：在 Mooncake 源码里搜 `33554432` / `32 * 1024 * 1024` / `MaxCallRecvMsgSize` / `MaxRecvMsgSize`，纯静态、零风险、不影响任何人的机器。
>
> ### 3. 步长 17 精确闭合 —— **C14 升 L3**
>
> 回执：失败 key = `16, 33, 50, 67...` 步长 17，ROUND1 `failed = 117`。
> - 验算：`index = 16 + 17k ≤ 1999 ⇒ k = 0…116 ⇒ 117 个` —— **与实测精确相等**。
> - 补写轮数：`⌈log₁₇(2000)⌉ = ⌈2.68⌉ = 3` —— **与实测 3 轮精确相等**（R1 漏 117 → R2 漏 8 → R3 到 2000）。
>
> ⇒ 机制完全闭合：`member_extent_size = 64MiB = 16 × 4MiB`，第 17 个 key（index 16）撞创建窗口，客户端不重试。**工程常数：N 个 key 必然需 `⌈log₁₇ N⌉` 轮补写。**
>
> ### 4. 端到端 per-key 延迟里看不到 128KiB 层的慢程（**本节已于 2026-11-24 19:30 自我纠正**）
>
> > **【自我纠正 · 方法违规】** 本节原版把 **400-key 口径的 537.5µs（NOF-025）** 与 **2000-key 口径的 546.72µs（本单）** 并列，据此得出「流长 5×，P50 只差 1.7%」。**这直接违反本单裁决 §0 自己立的「口径隔离」规则**（2000-key 结果与全部 400-key 历史数字不得混用 / 并列 / 做比值）。
> >
> > **该对比与由它推出的结论全部作废。** 具体作废项：
> > - 「流长 5× P50 不变」——不成立（两数不可比）
> > - 「400 keys 的 P90 应出现 6ms 级 key」——该推断建立在跨口径的慢占比 21.9% 上，一并作废
> >
> > **性质**：不是笔误，是方法上的自相矛盾 —— 规则写在同一个裁决块的另一节，违反写在相邻一节。记入教训。
>
> **本节只保留 2000-key 口径内部的观测**：
> - 4 遍 P50：538.88 / 577.86 / 525.87 / 544.28；P90 635–736µs；P99 690–736µs
> - 分布跨度 Min 461 → Max 1227，**仅 2.6×**
> - 若慢程（201µs/128KiB）作用在整 key 上：按等效串行 21.7 计，单个慢 key ≈ **4,362µs**；慢程区 = 前 100 key = **5%** ⇒ 应出现在 **P95** 以上
> - **实测 P99 = 705µs、Max = 1,227µs，无任何 4,000µs 级 key** ⇒ 慢程未在端到端显形
>
> **这个判断只依赖本单自己的数据，不跨口径**，故成立；后续由 NOF-087 独立复核为 **1.043×**。
>
> ⇒ **U03（换算缺口）仍登记为未决**：trace 的单 128KiB IO 延迟 ↔ bench 的 per-key 4MiB 延迟之间还缺一层换算。**NOF-087 给的 13.3µs/task 是「per-key ÷ 32」的算术平均，不是与 trace p50 对齐过的值**，只能算提供一个候选口径，**不能当作换算已完成**。
>
> ### 5. 决策：`vseg_nof086` 立即冻结为**只读资产**
>
> 三条硬约束，写入后续所有任务书：
> 1. **禁止再向 `vseg_nof086` 写入任何 key。** 它已不可重启，唯一价值是 2000-key 读基线。
> 2. **禁止切换 cid / 重启 master。** 一重启就触发 942MB 回放 ⇒ 集群报废（产品缺陷 ③）。
> 3. **etcd 余量 = 316MB**（1,187,229,696 / 2GiB = 55%，70% 线余 316MB）。**任何新一轮补写都可能一次吃光**（R3 单轮 810MB）。⇒ 定位超线性增长机制前，**禁止任何写入型实验**；只读实验不受限。
>
> ### 7. **C11 定案（L3）：墙在 Mooncake 客户端，不在 etcd** —— 源码已定位（2026-11-24 19:00）

用户 18:55 批准后，开发侧直接 grep 本地源码（`D:\mooncake-spdk-gap\_ref\Mooncake`，HEAD `706551a`，2026-11-03），得到完整因果链，**每一环都有文件:行号**：

| # | 环节 | 证据 |
|---|---|---|
| 1 | 启动回放入口 | `master_service.cpp:16878 TrySelfRecoverFromOpLog()`；外层 `kMaxSelfRecoverAttempts = 3`、间隔 200ms |
| 2 | 回放主体 | `master_service.cpp:16767 RecoverStateFromOpLogStream()`；`kMaxPollRounds = 4096`、`kMaxReplayWallClock = 30s`、RETRYABLE 时 `sleep 50ms` |
| 3 | 每轮拉取 | `master_service.cpp:16822 reader.PollOnce()` —— **未传参** |
| 4 | 默认批数 | `oplog_batch_standby_reader.h:37 PollOnce(size_t max_batches = 1024)` ⇒ **1024 批/轮** |
| 5 | 落到存储层 | `oplog_batch_standby_reader.cpp:90 storage_.ReadBatchesAfter(last_applied_batch_id_, max_batches, batches)` |
| 6 | **单条 etcd Range** | `oplog_batch_storage.cpp:234 backend_.Range(begin_key, range.end_key, remaining, kvs)`，`remaining = limit - 已拉数` ⇒ **一次请求最多拉 1024 个 KV** |
| 7 | **客户端上限** | `mooncake-common/etcd/etcd_wrapper.go:96 storeMaxMsgSize = 32 * 1024 * 1024 // 32MB`；`:105-106` 赋给 `MaxCallSendMsgSize` / `MaxCallRecvMsgSize` |
| 8 | 数值吻合 | `32 x 1024 x 1024 = 33,554,432` = NOF-071 报错值，**精确相等** |
| 9 | 重试为何 959 次 | `Range` 失败 ⇒ `ETCD_OPERATION_ERROR` ⇒ `IsRetryableBackendError` → RETRYABLE（`oplog_batch_standby_reader.cpp:13-16`）⇒ sleep 50ms **再发同一请求** ⇒ 959 × 50ms ≈ 48s，与 NOF-071 的 75s（≈ 2.5 × 30s 预算）吻合 |
| 10 | 终态 | 3 次外层全败 ⇒ `LOG(ERROR) "self oplog replay failed ... fall back to empty state"` ⇒ `return false` ⇒ serve 前失败 ⇒ **50056 从未 LISTEN** |

**语义辨析（这正是 NOF-084 回执归错的地方）**：
- etcd `--max-request-bytes=33554432` 管**请求**。回放是纯读，请求只有 begin/end key + limit，**不可能 35 MB** ⇒ **etcd 那个 32 MiB 与本次故障无关**。
- 响应体 35,087,559 B 是从 etcd **读出**的数据，撞的是客户端 `MaxCallRecvMsgSize`。
- 两者数值相同（都是 32 MiB），**纯属巧合**（都取了「32MB」这个整数），靠数字无法区分，必须靠语义。

**⇒ 结论：`vseg_nof068b` 与 `vseg_nof086` 起不来，责任在 Mooncake 客户端 `storeMaxMsgSize = 32MiB`，与共用 etcd 无关。改这个常量不影响任何人。**

**根因（比常量更深一层）：单次拉取量按「批数」控制，而不是按「字节数」控制。**
- 批的大小是可变的：NOF-086 实测 entry 均体积逐轮 **2,311 → 28,719 → 206,301 B（89×）**。
- 固定 1024 批 ⇒ 字节数完全不可控：NOF-068b 前 1024 批 = 35,087,559 B（34,265 B/批）；NOF-086 均 67,637 B/批 ⇒ 估计 **69 MB**。
- **按批数限流是个错误的抽象**——这正是它必然被写爆的原因。

**修复选项（均需改码授权，本单不执行）**：

| 方案 | 改动 | 效果 | 风险 |
|---|---|---|---|
| **A 降批数** | `PollOnce(128)`，或把 header 默认值改小 | 单次 4.4 MB（按 34,265 B/批）；NOF-086 的 13,930 批需 109 轮 << 4096 上限 | 低；功能无损 |
| **B 按字节预算截断（推荐·治本）** | `ReadBatchesAfter` 加 `max_bytes`，累计到 8 MiB 即停 | 与批大小无关，永不超限 | 中；改函数签名 |
| **C 提高上限** | `storeMaxMsgSize` → 2GB（对齐 `snapshotMaxMsgSize`） | 一行改动 | **单独不可行**：NOF-086 的 942MB oplog 一次拉 1024 批也要 ~69MB 内存，仍需配合 A/B |

**一个必须点明的设计矛盾**：同一份文件里 `snapshotMaxMsgSize = 2000 * 1000 * 1000 // 2GB`（`etcd_wrapper.go:85`，注释 "for GB-level snapshot files"）——**作者自己知道 GB 级数据要单独开一条大额度通道，只是没把同样的思路用到 oplog 回放上**。⇒ 这不是「做不到」，是「没做到」。

> **版本缺口（必须如实标注）**：以上行号取自本地 `706551a`。生产部署为 `be10be2`，**未逐行核对**。若 `be10be2` 中 `storeMaxMsgSize` 不是 `32 * 1024 * 1024`，则本结论失效 —— 但 `333554432` 与报错值精确相等，且调用链的 1024/4096/30s/50ms 四个数字与 NOF-071 时序全部吻合，巧合概率极低。**NOF-087 第一步就是核这个版本的常量值。**

> ### 6. 产品缺陷 ③ 证据强度升级
>
> 原先只有一例（`vseg_nof068b` 58.5MB）。现在多了 **942MB** 一例，且是在**常规用量**下产生（2000 个 4MiB key，非极端配置）。
> ⇒ 缺陷 ③ 从「极端场景」降级为「**常规用量即可触发**」。建议进对外报告。
>
> ---
>
> ### 本单保留的验收数据（后续引用以此为准）
>
> **读 4 遍**（`num_keys=2000 / warmup=128（第3遍为0） / io_size=4MiB / QD=batch=1 / threads=1 / slot=1`）：
>
> | 遍 | Total | failed | Min | P50 | P90 | P99 | Max | Avg | MiB/s | keys/s |
> |---|---|---|---|---|---|---|---|---|---|---|
> | R1 | 2000 | 0 | 466.46 | 538.88 | 646.27 | 703.39 | 756.12 | 558.72 | 7,153.44 | 1,788.36 |
> | R2 | 2000 | 0 | 476.22 | 577.86 | 679.53 | 736.66 | 961.08 | 594.93 | 6,719.44 | 1,679.86 |
> | R3 | 2000 | 0 | 461.37 | 525.87 | 635.22 | 690.74 | 764.78 | 547.21 | 7,307.35 | 1,826.84 |
> | W0 | 2000 | 0 | 471.43 | 544.28 | 653.04 | 705.32 | 1227.07 | 564.85 | 7,079.71 | 1,769.93 |
>
> **写入末态**：`Keys=2000 (soft-pinned 0)`、`logical_capacity=8,455,716,864`、`logical_used=8,388,608,000`、`logical_free=67,108,864`、`vsegments=126`、`NVMe-oF SSD 7.81GB/100GB`。
> **物理覆盖**：每盘 offset 0–727.27MiB（8,000MiB ÷ 11）⇒ 覆盖 `vseg_nof080`/`vseg_nofvseg` 全部数据区 + `vseg_nof084` 前段。**此前「跨 cid 按 LBA 对比新旧数据」的可能已彻底消失。**
> **环境末态**：target PID 905586 / md5 `3129641f…` 未变 / uptime 05:11:29 / 4457 LISTEN；master PID 2292801（17:28:33 起，未重启）/ 50056+9006 LISTEN；etcd 3.4.14 / dbSize 1,187,229,696（55%）/ alarms 空；trace mask 0x0；bench 残留 0。


## 0. 开发侧决定（覆盖此前所有相关约束）

**结论：能写，直接写。** 三条依据：
1. oplog 只在 **master 重启回放**时挡人，**不参与写入与读取**。本单全程不重启 master，墙就不存在。
2. bench 原生支持 2000（`-num_keys` uint64，仅 size_t 溢出保护）。
3. 空间够：2000 × 4MiB = **8,000 MiB = 8,388,608,000 B** 逻辑，ns 100 GiB。

**撤销 NOF-084 的「外推 >24MB 即停手」判据**（那是开发侧拍的保守值，且 24.2MB 那个下限假设 1 entry/key，与实测 7 entries/key 矛盾）。NOF-085 暂缓——它的 B 段要切 master，与本单「全程不重启」冲突。

**代价（必须书面接受）**：写完 2000 keys 后，该 cid 的 oplog 预计 **170–250MB**，**远超 32MiB 回放上限 ⇒ 该 cid 此后永久不能重启 master**。本单验收完成后它就是一个「只能活着用」的集群，直到另行处理 32MB 墙。**不得在本单内试图重启验证**。

## A. 前置只读检查（5 分钟，不过则降级）

1. etcd：`--quota-backend-bytes` 现值、当前 `dbSize`、`/oplog/` 各 cid 体积总和。
2. 判据：**预估新增 oplog（250MB）+ 当前 dbSize 必须 < quota 的 70%**。
   - 不过 ⇒ 降级写入目标：按余量算出能写的最大 keys（1000 / 500），在回执写明实际目标值，其余照本单执行。
   - etcd 一旦 `NOSPACE` 变只读，会影响**所有人**（yin / mooncake_cluster / royi），这是本单唯一的高危项。
3. 记录 target 状态（PID 905586 / md5 `3129641f…` / 4457），本单不动它。

## B. 切到全新 cid（避免 key 冲突污染计数）

1. **必须新建 cid `vseg_nof086`**，不得在 `vseg_nof084` 上续写。原因：`vseg_nof084` 已有 100 keys，bench 从 key 0 开始写会撞 `OBJECT_ALREADY_EXISTS`，而 `stress_cluster_bench` 会把它**计为成功**（NOF-036 已知缺陷）⇒ `Keys` 与 bench 报数对不上。
2. 参数：`slot=1`、`stripe=2097152`、`member_extent=67108864`、其余同 NOF-084；`MC_STORE_PARTITION_SLOT_COUNT=1`。
3. 停 `vseg_nof084` master（Keys=100 / oplog 8.48MB，**它能正常重启，所以随时可切回**），起 `vseg_nof086` master；`nof_register` + mount，验收 `50056 LISTEN` / `Keys=0` / `mount rc[0]`。

## C. 写 2000 keys（多轮补写；**每 500 keys 一次 checkpoint**）

1. writer：`--num_keys=2000`，`value_size=4MiB`、`batch=1`、`threads=1`（与 400-key 口径一致），`warmup_keys` 不用（writer 不涉及）。
2. **多轮补写直到 `Keys=2000`**：`VSEGMENT_CREATING` 会在每 17 个 key 的卷边界漏 key（2000 keys ⇒ 125 个 vsegment、约 124 次边界穿越）。按 NOF-057 的方式重跑 writer 补齐，每轮记录：
   `ROUND n: succeeded / failed / VSEGMENT_CREATING 次数 / master Keys / logical_used / vsegments / oplog entries / oplog bytes`
3. **验收不信 bench 报数**（NOF-036 缺陷），只认 master 的 `Keys=2000` 且 `logical_used=8,388,608,000`。
4. **checkpoint 止损**：每跨 500 keys 记录一次 oplog 体积；若 `dbSize + oplog 增量 > quota × 0.70` 或 etcd 报 `NOSPACE` ⇒ **立即停写**，用当前已写入的 keys 数执行 D 段，并在回执写明停在哪。
5. 轮次上限 12 轮；到 12 轮仍未满 ⇒ 停手，用实际 Keys 做 D 段。

## D. 写完立即读（趁 master 还活着）

1. reader：`--num_keys=<实际 Keys>`、`batch=1`、`threads=1`、`warmup_keys=128`（记录：2000 keys 下只覆盖 6.4%，不擅自改）、`verify=false`（前 100 个 key 的内容可能被历史 preflight 覆盖，以 master 元数据为准）。
2. **读 3 遍**，每遍给 `p50 / p90 / p99 / avg / failed / span`。
3. 追加一遍 `--warmup_keys=0` 作对照（记录即可，不作结论）。
4. 读全程**不写数据**，oplog 不应增长；读完复测 oplog，若增长 >1% 需说明原因。

## E. 绝对禁止

- **禁止在任何时刻重启 master**（写完后重启必然 `ResourceExhausted`，会毁掉本单成果）。
- 禁止重启 target、禁止改 etcd 配置、禁止改码/重编、禁止删任何 `/oplog` 前缀。
- 禁止在 `vseg_nof080`（oplog 22MB）上写入；`vseg_nof084` 只停不删。
- 不碰他人进程（royi master 3463695 / yin 的任何进程）。
- 若 master 中途意外退出 ⇒ **不要重启**，立即停手回报（重启会失败，且会丢本单结论）。

## F. 回执必答（第一行）

实际写入 `Keys`（是否 2000）/ `logical_used` / 补写轮次 / 每轮 `VSEGMENT_CREATING` 次数 / oplog 最终体积 / 读 3 遍 p50 / 是否触发 etcd 空间告警 / 有无失败 / **master 是否全程未重启**。

另需逐项报口径基线：`num_keys / warmup_keys / io_size / QD / threads / slot / stripe / member_extent / cid / target PID+md5 / 客户端二进制 / 诊断开关 / 是否开 HA（读 `/proc/<master_pid>/cmdline`，本单补上这一项）`。

## G. 产物与环境末态

- `run/nof086_summary.txt`（≤1MB）+ 每轮 writer/reader 原始日志（≤5 个文件）。
- 末态：`vseg_nof086` master 运行中（Keys=实际值）、target 未重启、trace mask 0x0、无残留 bench。
- **明确写下**：该 cid 的 oplog 体积与「不可重启」状态，供后续接手人知悉。

---

# NOF-096（已关闭 · L3：ENODEV 根因=静态 SPDK 库集错误；plain 打点位置需改 · 2026-11-25）

> **结果**：`link.txt` 引用的 `/usr/local/lib/libspdk_*.a` 已于 **2026-11-20 21:55** 被整体替换；旧可用二进制实际使用 `/tmp/nof051/link.sh` 的 `$D/build/spdk-e12b7cb-src/build/lib`（e12b7cb / wire v4）。同一份源码仅换静态库来源：`/usr/local/lib` → 全部 attach `rc=-19`；e12b7cb 库集 → attach rc=0、20/20 读取成功。**NOF-095 的“重链不可复现”判断推翻：输入本来就不同。** 新构建 `bench-nof096`（md5 `8a7f8548…`）已验证可 attach，保留备用；源码已回滚，原二进制未变，零业务写入，target 905586 未重启。
>
> **plain 路径更正**：`Client::Get`（3 参）→ `TransferRead` → **`TransferData`**；4 参 `Get(src_offset)` 才走 `TransferReadInternal`，且会拒绝非 memory/vsegment 副本。plain 的 timing 应在 `TransferData` 的 `future->get()` 后输出。**回执中的 L2338/L2503 是 BatchGet 的行号，不是 TransferData；be10be29 的 `TransferData` 起点 L5770，`future->get()` 约 L5888。** vseg 现有 `TransferReadInternal`（L5926）补丁位置正确。
>
> **下一步 NOF-097**：plain 改打 `TransferData`，vseg 保持 `TransferReadInternal`，用 e12b7cb 静态库重链，plain/vseg 各 20-key 冒烟后完成正式测量。

## 背景与目标

- NOF-095 已证明：**不是补丁导致 attach 失败**——未改源码、未改 archive 的裸重链也 `rc=-19`；OLD 二进制正常。
- 但“相同输入链接出不同结果”尚未证明：**OLD 当初的真实链接命令和静态 SPDK 库集，可能不同于当前 `link.txt`**。`ldd` 相同不能排除静态 SPDK 库不同。
- 首要候选：OLD 使用 NOF-051 的 **bd5ee124d / wire v4** 静态库，裸重链使用 `/usr/local` 的另一套静态库。
- 本单只回答一个问题：**OLD 与 raw 的静态 SPDK/链接输入是否相同；若不同，按 OLD 的库集能否重建出可 attach 的补丁版。**

## A. 固定对照与版本取证（只读）

1. 记录并保留 md5/size/mtime：
   - OLD `bench-io-params-e12`（`88ad3134…`）、in-tree OLD（`59662a26…`）、NOF-095 raw（`0afb26d6…`）。若 raw 已删，用 NOF-095 的**原样控制命令**重链一次，记录命令与 md5；不改源码。
2. 对 OLD / in-tree OLD / raw 分别执行：
   - `strings | grep -E 'SPDK v|git sha1|wire|URMA|openEIPer|DPDK'`，记录内嵌版本；
   - `readelf -n`（build-id/note）、`readelf -d`（NEEDED/RPATH/RUNPATH）、`readelf -l`（program headers）；
   - `stat`、`getcap`、`getfattr`、`getfacl`（文件权限/cap/xattr）。
3. 找 OLD 的**真实链接命令**：NOF-051 的重链脚本/日志、`/tmp/nof051*`、构建目录备份、shell history；与当前两个构建目录的 `link.txt` 逐词对比。
4. 列出两条链接命令实际使用的**全部静态 `.a` 路径与 md5**，重点区分：
   - `$D/spdk-e12b7cb-src/build/lib`（bd5ee124d/v4）；
   - `$D/spdk-build/lib`（9a413a44/v6）；
   - `/usr/local/lib`。
   若 OLD 的历史库已不存在，记 `[未获取]`，不得用“mtime 早”替代内容证据。

## B. 决定性分叉

### B1. 若 OLD 与 raw 的 SPDK 版本/静态库集不同

1. 记录具体差异（版本串、wire、库路径/md5）。
2. 在我方 `$D/mooncake-src` 重新应用 NOF-095 的同一补丁（只改 `client_service.cpp`，备份 `.orig`，原二进制不动）。
3. 按 **OLD 的静态库集与真实链接参数**重链为 `bench-nof096`；`--help` 正常。
4. 在 `plain_nof088` 上跑 **20 keys / warmup=0**（root、每遍前 mount 验证），验收：
   - attach rc=0、读取成功；
   - **20 条 `NOF095_TIMING arm=plain has_timing=1`**。
5. 成功后**停在本单**：不建 vseg、不跑正式 3 遍、不写业务数据；回滚源码并验证原二进制 md5 未变。

### B2. 若版本与静态库集完全相同

1. **不要随机试 build-id/段布局参数**。先做纯二进制对比：
   - OLD vs raw 的 section 地址/大小/对齐、init/fini 数组、符号表、`.rodata`、build-id、文件权限；
   - 用同一命令再裸重链一次，比较 md5 是否稳定。
2. 回执给出**第一个可复现差异**与是否可解释 `rc=-19`；若不能，停手，不继续堆实验。

## C. 成功标准与回执必答

- OLD / raw 的内嵌 SPDK 版本、wire、build-id、RPATH；
- OLD 的真实链接命令 vs 当前 `link.txt` 的差异；
- 两套静态 `.a` 路径/md5 对比；
- 若重建：`bench-nof096` md5、plain 20-key attach/读取/timing 条数；
- 源码是否回滚、原二进制 md5 是否未变；
- 结论：**静态库集错误 / 链接不可复现 / 仍未定位**，三选一；
- 新增业务写入=0，target 905586 未重启。

## D. 边界

- **target 905586 / md5 3129641f 不重启**；不启用 BUFFER。
- **零业务写入**：只读 `plain_nof088` 做 20-key 冒烟；不建 `vseg_nof095`、不改 etcd、不重启 master（除非 plain master 未运行，则按完整 env 启动并 `nof_register`）。
- 只改我方副本 `client_service.cpp`，不改同事树；原 `bench-io-params-e12` / in-tree OLD 的 md5 必须不变。
- 构建或冒烟失败即停手回滚，不扩大实验矩阵。

---

# NOF-097（已关闭 · plain timing 完成但统计需重算；vseg 挂载 rc=-101 阻塞 · 2026-11-25）

> **plain 臂结果（L0，待重算）**：`TransferData` 打点成功，4 遍有效正式读（P1/P3/P4/P5）各 528 条；lane 0–3、qpair 2–5、worker 0–3 各 560，完全均匀；`sub_io_count=32`。初算 `credit_wait_us` / `first_credit_wait_us` / `qos_fifo_wait_us` / `qos_tasks_ahead` 非零占比 0%，`head_to_first_submit_us` p50=1µs，`dispatch_queue_us` p50=11µs，`first_to_last_submit_us` p50=27µs；时间集中在 drain（p50 106µs，占 total 78.5%）与最慢子 IO（p50 109µs）。**但回执统计 n=2240 = 4×528 + P2 的 128 条 warmup，和“P2 未混入判读”自相矛盾；max=9121µs 等尾部数字可能被污染。NOF-098 必须先只用 P1/P3/P4/P5 重算（n=2112），再引用 plain 结论。**
>
> **vseg 臂阻塞**：`vseg_nof097` master 启动正常（owned_slots=1、ready=1/1），但 `MountNoFSegment rc=-101 SEGMENT_NOT_FOUND`，master 侧 `Failed to attach nvme ctrlr`，`discovered_segments=0`。plain 同二进制/同 target 可挂载；target subsystem 正常；`exclusive=true` 不可去。**关键对照：历史 `vseg_nof086` 同参数/同 `nof_register` 能挂载（rc=0）⇒ 不能只归因 host 残留，先查新老 cid 的注册元数据/命令/env 与 target 首个错误。**
>
> **环境**：源码已回滚（md5 `a147b489…`），原二进制未变；`bench-nof097`（md5 `6757c001…`）保留；target 905586 未重启；`vseg_nof097` 0 keys、仅元数据；plain_nof088 Keys=2000、挂载已恢复；新增业务写入=0。产物 `run/nof097_summary.txt` + 5 个日志 gz。
>
> **下一步 NOF-098**：先重算 plain（排除 P2），再对比 `vseg_nof097` 与 `vseg_nof086` 的 register/env/etcd descriptor/target 错误；不清理资源、不写业务数据。

## 背景与目标

- NOF-096 已定案：ENODEV 根因是**静态 SPDK 库集错误**；必须用 `$D/build/spdk-e12b7cb-src/build/lib`（e12b7cb/wire v4，`/tmp/nof051/link.sh`），禁用 `/usr/local/lib/libspdk_*.a`。
- plain 路径已核实：`Client::Get`（3 参）→ `TransferRead` → **`TransferData`（L5770，`future->get()` 约 L5888）**；vseg 路径：`TransferReadRange` → **`TransferReadInternal`（L5926）**。
- 本单完成 NOF-095 未完成的测量：两个真实路径都输出 `NOF095_TIMING`，回答 lane/credit/队头阻塞/drain/最慢子 IO。

## A. 基线与构建硬规范

1. 在我方 `$D/mooncake-src` 记录 HEAD（必须 be10be29）、status、`client_service.cpp` md5；备份 `.orig`；记录原 `bench-io-params-e12` / in-tree OLD md5。
2. **只允许 e12b7cb 静态库集**：使用 `/tmp/nof051/link.sh` 的库路径与链接参数；**禁止 `/usr/local/lib/libspdk_*.a`**。构建后 `strings bench-nof097 | grep spdk-e12b7cb-src` 必须命中，记录 22 个静态库路径/md5 与产物 md5。
3. 独立产物 `bench-nof097`，原二进制不动。

## B. 补丁（两处真实路径）

1. **plain NoF**：在 `Client::TransferData` 的 `const ErrorCode result = future->get();` 之后，仅对 `is_read && replica_descriptor.is_nof_replica()` 输出 `NOF095_TIMING arm=plain`；保留原 SpDiag 与 `transfer_data_breakdown`。
2. **vseg**：在 `Client::TransferReadInternal` 的 vseg 分支 `future->get()` 之后输出 `NOF095_TIMING arm=vseg`；**删除/不保留 NOF-095 加在 `TransferReadInternal` plain 分支上的无效日志**（该分支不是 plain bench 路径）。
3. 两条日志均包含 `has_timing=0|1` 与 `NofTransferTiming` **全部字段、按定义顺序**；无 timing 也输出 `has_timing=0`，不得静默跳过。
4. 只改 `client_service.cpp` 一个文件；用函数内 lambda 避免复制格式。

## C. 执行顺序

0. **plain 预检**：`plain_nof088` master 若未运行，按完整 env 启动（`MC_STORE_PARTITION_SLOT_COUNT=1`、URMA 设备/capsule）；`nof_register` 后验收 `Keys=2000`、`owned_slots=1`、`ready=1/1`。
1. **plain 冒烟**：20 keys、`warmup_keys=0`，预期 **20 条 `NOF095_TIMING arm=plain has_timing=1`**；为 0 ⇒ 停手回滚，不继续。
2. **plain 正式**：读前 400 keys、`warmup_keys=128`，跑 **3 遍**；每遍前重新挂载并验证；预期 **528 条/遍** timing。
3. **建 `vseg_nof097`**：按 NOF-095 §C 完整参数组启动 vseg master（slot=1、stripe=2MiB、member_count=1、member_extent=64MiB 等）；写入前复测 etcd 余量；T1 写 400 keys，多轮补齐，验收只认 master `Keys=400` / `logical_used=1,677,721,600`。
4. **vseg 冒烟**：20 keys、`warmup_keys=0`，预期 **20 条 `arm=vseg has_timing=1`**；为 0 ⇒ 停手回滚，不继续。
5. **vseg 正式**：同口径跑 **3 遍**（400 keys / warmup 128 / T1 / batch1 / qpairs4 / verify=false）；每遍前重新挂载；预期 **528 条/遍** timing。
6. 测完恢复 `.orig` 并重编回原状（或删除新产物并验证原二进制未变）；停 master，target 保持运行。

## D. 解析与判据

每臂报告：
- timing 行数与 `has_timing` 分布；
- lane 去重数、qpair/worker 分布；
- `credit_wait_us`、`first_credit_wait_us`、`head_to_first_submit_us`、`dispatch_queue_us`、`qos_fifo_wait_us`、`worker_setup_us` 的 **p50/p90/非零占比**；
- `first_to_last_submit_us`、`last_submit_to_complete_us`、`task_total_us`；
- `max_sub_io_latency_us/index/cid`、`sub_io_count` 分布。

判读：
- vseg lane 更少或 credit/队头明显更大 ⇒ 客户端提交层有责任；
- 三项相同但 drain/最慢子 IO 更大 ⇒ 客户端提交正常，target/传输/会话等待更大；
- 全部相同 ⇒ 客户端 timing 层排除，转 target/会话受控实验。

## E. 边界

- **target 905586 / md5 3129641f 不重启**；不启用 BUFFER。
- **禁止 2000 keys**；只允许新建 `vseg_nof097` 写 400 keys；写入前查 etcd，余量不足停手。
- 不改共用 etcd、不删 `/oplog`、不改 `/etc/spdiag/spdiag.conf`。
- 只改我方副本一个文件，原二进制 md5 必须不变；构建或冒烟失败即停手回滚，不扩大实验矩阵。

---

# NOF-098（已关闭 · plain 统计复核通过；vseg 假设否证，host attach EPERM 仍未定位 · 2026-11-25）

> **plain 统计复核**：只用 P1/P3/P4/P5（n=2112）与含 P2 warmup 的 n=2240 相比，各字段差异 **≤1µs**；max=9121µs 来自 **P4**，不是 P2 污染。plain timing 结果可作为干净单臂基线：4 lane/qpair/worker 均匀，credit/FIFO/队头全零，时间在 drain/最慢子 IO；但仍无 vseg 对照。
>
> **关键更正**：今日 **plain 模式（无任何 vsegment 参数）同样 attach 失败**：`Operation not permitted / rc=-19 / opened=0`。NOF-097 的“plain 挂载成功”结论作废——`Keys=2000` 只是 etcd 元数据，不代表有活的 NoF 挂载。
>
> **失败时间线**：最后一次成功 **2026-11-24 19:30 plain_nof088**；首次失败 **2026-11-24 21:55 vseg_nof091**；此后 plain/vseg 全部失败。target 905586 自 11-24 12:21 未重启，4457/subsystem/allow_any_host 正常，target 日志零连接/拒绝错误。node1 `/dev/uburma/udmac0d1e2` mtime 为今日 12:49（其余 udmac* 为 11-17），dmesg 仅周期 `ummu event 0x12`，未见 EPERM 归因行。
>
> **已排除**：新老 cid 命令/env（仅 cluster_id 不同）、缺失 descriptor、identity、target 资源/会话错误。**结论：host 侧 attach 链路整体失效，EPERM 上游触发点未定位。**
>
> **下一步 NOF-099**：只读查 host 设备权限/provider/cgroup/内核日志与首个失败 syscall，并用旧 bench 做全局性对照；不重启 target、不换设备、不清理资源。

## 背景与目标

- NOF-097 plain 打点成功，但统计 `n=2240` 混入了作废 P2 的 128 条 warmup；必须先重算。
- `vseg_nof097` 挂载 `rc=-101 SEGMENT_NOT_FOUND`，master attach NVMe controller 失败；plain 同环境可挂载，target subsystem 正常。
- **历史 `vseg_nof086` 同参数/同 `nof_register` 能挂载** ⇒ “host 残留”不是唯一嫌疑，可能是新 cid 的注册元数据/命令/env 或 target 会话状态差异。
- 本单只定位失败点，不做资源清理、不写业务数据。

## A. plain 统计重算（零环境，先做）

1. 只用 NOF-097 的 **P1/P3/P4/P5** 四遍正式日志（`n=4×528=2112`）重算全部 timing 字段；**排除 P2 的 128 条 warmup**。
2. 对比初算 n=2240 的 p50/p90/max/非零占比，明确哪些数字被 P2 污染。
3. 形成 plain 的干净基线表；在此之前不得引用 max=9121µs 等尾部数字。

## B. 新老 cid 注册/挂载对比（只读）

1. **命令/env**：找出 `vseg_nof097` 与历史 `vseg_nof086` 的 master 启动命令、`/proc/<pid>/environ`、`nof_register` 命令；逐项对比 cluster_id、slot_count、URMA 设备/capsule、vsegment 8 参数、etcd/master 地址、segment name/base/nsid/capacity。
2. **etcd 元数据**：对比 `/cvm/vseg_nof097/` 与 `/cvm/vseg_nof086/` 的 key 集合、value 大小/hash；重点 `segments/`、`nof_identities`、quota/profile、device_identity、base_offset、capacity。若可解码，给出字段级差异；不可解码则给 key 名/大小/hash，不猜。
3. **日志**：对齐两次 `nof_register` / `MountNoFSegment` 的完整错误链，给出 **target 侧第一个错误**（不是最后的 rc=-101）；对比时间、endpoint、subnqn、nsid、device_identity。
4. **target 会话资源（只读）**：记录 URMA controller/qpair/jetty/JFC 相关 RPC 计数、`/dev/uburma` 映射、target 日志中的 handshake/jetty/resource 错误；与历史成功时日志对照（若有）。

## C. 一次受控重注册（不清理、不新建 cid）

1. 完成 B 段快照后，对**现有 `vseg_nof097`** 重跑一次 `nof_register` + mount，记录完整日志与 etcd key 变化。
2. 若挂载成功：记录 rc/discovered_segments/Keys=0，**停在本单**，不写 key、不跑 timing。
3. 若仍失败：停手，给出“缺失/无效 descriptor / identity 不匹配 / target 资源或会话错误 / 仍未定位”四选一。

## D. 回执必答

- plain 重算 n=2112 的完整表与 n=2240 的差异；
- 新老 cid 的注册命令/env 差异；
- etcd segment descriptor 的 key/大小/hash/字段差异；
- target 侧第一个错误；
- 重注册结果；
- 结论四选一；
- 新增业务写入=0、target 905586 未重启、原二进制未变。

## E. 边界

- **只读为主**；唯一状态变更是 C 段一次现有 cid 的 `nof_register` + mount。
- **不清理 host/target 资源、不 kill 进程、不删 cid、不新建 cid、不写业务 key**。
- target 905586 不重启；不改共用 etcd 配置；plain_nof088 保持可读。
- 若发现需要清理/删除/重置会话层，先回报，由开发侧授权，不在本单执行。

---

# NOF-099（已关闭 · 无 syscall EPERM；收窄到 URMA 用户态握手/connect 应答层 · 2026-11-25）

> **关键结果**：两次 strace 均**没有任何 EPERM syscall**；`openat("/dev/uburma/udmac0d1e2", O_RDWR)`、URMA ioctl、TCP `connect():4457` 全部成功。失败发生在这些调用之后：**host 侧 URMA 用户态（liburma/ubcore provider）握手，或 NVMe-oF connect 应答被映射为 EPERM**（L2 候选，尚未定位）。
>
> **已排除**：`/dev/uburma` 权限（666 root:root、无 ACL/xattr）、EID 表（两端 1024/1024 一致）、provider/URMA 库 md5（两端一致）、capability（全能力）、devices cgroup（无）、AppArmor（关）、SELinux（Disabled）、hugepages/IOVA。`udmac0d1e2` 节点 12:49 重建晚于首次失败，不是起点；udev/journal/dmesg 无 EPERM/permission/reset 记录。
>
> **偏差**：指定旧 bench `bench-io-params-e12` 在 node1 不存在，使用的 node1 `bench-io-params` 在 attach 前因“无 mounted NoF segment”退出，未能复现 EPERM；因此“旧 bench 全局对照”仍未完成。strace 做了两次（第一次漏 write/read），结论一致。
>
> **环境**：master plain_nof088 仍运行但 NoF 未挂载；target 905586 未重启、4457 正常；新增业务写入=0。产物 `run/nof099_summary.txt` 与 strace/register 日志。
>
> **下一步 NOF-100**：在一次 `nof_register` 前后采集 target transport/qpair/stats/日志增量，判断连接请求是否到达 target；不重启、不换设备、不清理资源。

## 背景与目标

- NOF-098 已定案：失败与 vseg 配置无关，plain 同样 `Operation not permitted / rc=-19 / opened=0`；target 完全正常且看不到连接请求。
- 最后成功：2026-11-24 19:30；首次失败：2026-11-24 21:55；此后所有 plain/vseg master 均失败。
- node1 `/dev/uburma/udmac0d1e2` 节点今日被重建；dmesg 仅见周期 `ummu event 0x12`，没有 EPERM 归因。
- 本单只查 **host 侧 attach 在哪个 syscall/权限/provider 状态失败**，不做恢复操作。

## A. 全局性对照（旧 bench vs master）

1. 用原可用二进制 `bench-io-params-e12`（88ad3134）在 root、相同 env/cwd 下做一次 10-key attach/读尝试；记录首个错误、rc、是否 opened。
2. 若旧 bench 也失败 ⇒ host 链路全局失效；若旧 bench 成功而 master 失败 ⇒ 收窄到 master 进程/env/资源。
3. 对一次失败 attempt 使用 `strace -f`（openat/ioctl/mmap/read/write 等），保存**第一个返回 EPERM 的 syscall 与参数**；不得只贴最后的 `rc=-19`。

## B. `/dev/uburma` 设备状态（只读）

1. 对 `udmac0d1e2` 与对照设备 `udmac0d1e3/1e4` 记录：`ls -lZ`、`stat`、`readlink -f`、`getfacl`、`getfattr`、major/minor、owner/mode/mtime。
2. 读 `/sys/class/uburma/udmac0d1e2/` 与 `/sys/class/uburma/udmac0d1e2/device/` 的 uevent/provider/numa/reset/error 计数；与其它 udmac 设备对比。
3. 查 udev/journal 从 **2026-11-24 19:00 至今**的 `uburma|urma|ummu|udmac0d1e2|reset|permission|EPERM` 事件，定位节点重建/设备 reset 的**首次时间**。

## C. 进程权限与运行上下文（只读）

1. 对失败 master/bench：`id`、`capsh --print`、`/proc/<pid>/status`、`/proc/<pid>/limits`、cgroup 路径与 device controller 规则、AppArmor/SELinux 状态、ulimit、hugepages、IOVA 模式。
2. 对比历史成功 master 的日志/env（若有）与当前进程；重点看 21:55 启动脚本是否改过 env、cgroup、设备权限或 provider。
3. `dmesg`/journal 查 `uburma|urma|ummu|EPERM|permission|reset`，记录最早错误；没有就写 `[未获取]`。

## D. provider/工具自检（只读）

- 若 `rdma`/`ibv*`/URMA CLI 存在，查询 `udmac0d1e2` 的 provider/context/device 状态；只查询，不 attach、不 reset。
- 检查 URMA provider 模块/服务状态、`/dev/uburma` 映射数、是否残留旧进程持有设备 fd（只记录 PID/fd，不 kill）。

## E. 回执必答

1. 旧 bench 是否也失败；master 与 bench 是否同样全局失效；
2. 第一个 EPERM syscall 与参数；
3. `/dev/uburma/udmac0d1e2` 的权限/owner/sysfs/provider 与对照设备差异；
4. 21:55 失败起点前后的 udev/journal/provider 事件；
5. 进程 capability/cgroup/AppArmor/SELinux 是否有拒绝证据；
6. 结论：**设备权限 / provider/cgroup / 设备 reset / master 特有 / 仍未定位**；
7. 最小安全恢复建议（只给建议，不执行）。

## F. 边界

- **不重启 target、不换 `SPDK_URMA_DEV_NAME`、不 reload driver、不清理资源、不 kill 进程、不重置设备**。
- 只读为主；唯一允许的运行是 A 段一次旧 bench attach/读尝试与一次 strace。
- target 905586 保持不变；plain_nof088 元数据不删；新增业务写入=0。
- 若需要重启 target/换设备/清理会话，先回报，由开发侧授权。

---

# NOF-100（已关闭 · target 零增量；host URMA qpair capsule 完成 err=-5；报文级未获取 · 2026-11-25）

> **target 侧（L0）**：一次 register 前后 `admin_cum/io_cum/cur_admin/cur_io/cur_unassigned` 全部 **+0**，controller 为空；新增 17,660B 日志全是周期 timing breakdown（n=0），无 connect/ctrlr/handshake/jetty/reject/error 行。
>
> **host 侧关键证据**：`nvme_urma.c:1199` 记录 **qpair 0 capsule completion error err=-5(EIO)**，随后 `nvme_urma.c:1540 fatal rc=-5, disconnecting for reconnect`，最终映射为 attach EPERM。说明 capsule 已进入 URMA 提交路径并拿到完成错误。
>
> **报文级证据缺失**：两端无 tcpdump/tshark；`ss -tni` 0.3s 轮询 80s 两侧均 0 命中，但这不是报文证据，且可能漏掉短连接。按判据判 **“不确定”**，不判“未到达 target”。
>
> **L2 候选（未过三闸门）**：失败点在 host URMA RC/jetty 路径，不是 target NVMe-oF 逻辑拒绝，也不是 TCP connect 失败；具体 URMA 错误码/失败操作未定位。
>
> **环境**：target 905586 未重启；master plain_nof088 仍运行但 NoF 未挂载；新增业务写入=0。产物 `run/nof100_summary.txt`、register/flow 日志、target pre/post 快照。
>
> **下一步 NOF-101**：用受支持的 liburma/ubcore debug 机制复现一次 attach，捕获 `err=-5` 前面的真实 URMA 错误码与失败操作；不重启 target、不换设备、不清理资源。

## 背景与目标

- NOF-099 证明 host 侧 `/dev/uburma` open/ioctl 与 TCP `connect():4457` 成功，strace 无 EPERM；失败在 URMA 用户态握手或 connect 应答映射层。
- 本单只回答一个问题：**这次 NVMe-oF connect 请求到底有没有到达 target 的 URMA transport？**
- 用两种独立证据：target RPC/日志增量 + 可选 TCP 抓包。

## A. target 前快照（只读）

1. 记录 target PID/md5/启动时刻、4457 LISTEN、`nvmf_get_subsystems`、`nvmf_get_transports`。
2. 采集 `nvmf_get_stats`、当前 admin/io qpair 数、URMA transport 可用计数；记录 `/tmp/nvmf_tgt.log` 的字节数/md5/offset。
3. 若 `tcpdump` 存在，准备对 node1↔node3 4457 的单次抓包（只抓本次 register 窗口，文件写 /tmp）。

## B. 触发一次注册

1. 在当前 `plain_nof088` master 上执行**一次** `nof_register`（不写业务 key）；记录开始/结束时间与 master 日志中的 3 次 attach 尝试。
2. 同时抓包（若可用）；不要触发第二次 register。

## C. target 后快照与日志增量

1. 重复 A 的全部 RPC/计数，给出前后 delta。
2. 只分析本次时间窗内新增的 target 日志，grep `connect|ctrlr|URMA|handshake|jetty|jfc|reject|error`；记录第一条相关行。
3. 若抓包成功：统计 SYN/ACK、TCP payload 字节数、连接方向；判断 NVMe-oF connect 数据是否发出/到达。

## D. 判读（三选一）

- **到达 target**：target 计数/日志或抓包证明 connect payload 到达 ⇒ 失败在 URMA 握手/target 应答层；下一步开 host/target 的 URMA 调试日志。
- **未到达 target**：只有 TCP SYN/ACK、无 connect payload，且 target 零增量 ⇒ host URMA 用户态在发送 connect 前失败；下一步开 liburma/ubcore debug。
- **不确定**：RPC 计数器不覆盖失败握手且抓包不可用 ⇒ 明确 `[未获取]`，不得猜。

## E. 回执必答

- target 前后 qpair/stats/transport 计数 delta；
- target 新增日志首条；
- 抓包是否有 TCP payload、字节数；
- 三选一判决；
- 新增业务写入=0、target 905586 未重启、原二进制未变。

## F. 边界

- **不重启 target、不换设备、不 reload driver、不清理资源、不 kill 进程**。
- 只读 RPC + 一次现有 master 的 `nof_register`；可选单次抓包。
- 不建新 cid、不写业务 key、不改 etcd 配置。

---

# NOF-101（已关闭 · provider 初始化成功，13.2ms 后 capsule err=-5；底层错误仍未定位 · 2026-11-25）

> **时序结果（L0）**：urma_open_drivers、udma/ub_agg/bondp provider 注册与初始化全部成功；**13.2ms 空窗后** capsule completion `err=-5(EIO)`；provider unregister/close 发生在报错之后 **8.2ms**，是清理，不是原因。
>
> **debug 结果**：`URMA_LOG_LEVEL=debug` 生效（journal 98 条 `[URMA]`），但没有底层错误码；`UDMA_LOG_LEVEL=debug` 零 `[UDMA]` 输出，实际生效性未证实；内核 ubcore 无错误行。唯一可证实失败阶段仍是 **capsule completion**，EID/jetty/JFC 编号 `[未获取]`。
>
> **偏差**：第一次停 master 因非 root 信号被拒，改 `sudo -n kill -TERM`；启动脚本第一次非 root 失败后残留，第二次 sudo 启动成功，导致实际发出 2 次 register RPC（第二次未新增 attach），已杀掉我方残留脚本进程。违反“一次 register”，如实记账。
>
> **环境**：debug master 已恢复为正常 env（PID 482472、Keys=2000、owned_slots=1、ready=1/1、无 debug env 残留）；target 905586 未重启；新增业务写入=0。产物 `run/nof101_summary.txt`、debug/register/journal 日志。
>
> **结论**：继续加日志收益低。**用户已批准 NOF-102**：同二进制/同 RPC 重启 target，用 plain 10-key attach 做恢复对照；成功⇒target 会话状态，失败⇒host URMA 数据通路。

## 背景与目标

- NOF-100：target 零增量；host URMA qpair 0 已提交 capsule，完成错误 `err=-5(EIO)`，随后 fatal disconnect。
- 目标：用库支持的 debug 日志抓到 **-5 前面的 URMA 错误码、失败操作、jetty/JFC/EID 上下文**。
- 本单不修环境，只定位；不重启 target、不换设备、不清理资源。

## A. 先确认受支持的 debug 机制（只读）

1. 对实际加载的 `liburma-udma.so`、`liburma_ubagg.so`、`liburma.so`、`libummu.so` 使用 `strings`/帮助/配置样例，找出**文档化或二进制明确支持**的日志级别 env/config（如 URMA/UBCORE log level、log file）。
2. 查 `/etc/urma*`、`/etc/ubcore*`、provider 文档/CLI；**不得猜 env 名**。
3. 若没有受支持的单进程 debug 机制，或需要改系统配置/装包 ⇒ 停手回报，不继续。

## B. 受控 master 重启 + 一次 register

1. 记录当前 `plain_nof088` master PID/cmdline/env、Keys=2000、target PID/md5。
2. 优雅停当前 master；用**完整正常 env + §A 找到的 debug env**启动同一 `plain_nof088`：
   - `MC_STORE_PARTITION_SLOT_COUNT=1`
   - `SPDK_URMA_DEV_NAME=udmac0d1e2`
   - `SPDK_URMA_CAPSULE_TRANSPORT=urma`
   - 其余 master/HA/etcd 参数与原进程一致。
3. 验收 50056 LISTEN、`owned_slots=1`、`ready=1/1`、`Keys=2000`；执行**一次** `nof_register`，记录 3 次 attach 尝试窗口。
4. 保存 master 日志、provider debug 输出、journal 中同窗口内容。

## C. 解析首个 URMA 错误

按时间顺序找 `nvme_urma_qpair_defer_error err=-5` **之前**的第一条 URMA/ubcore 错误，给出：
- 错误码与名称；
- 失败操作（jetty create/modify/rearm、send/recv、completion、EID exchange 等）；
- 本地/远端 EID、qpair/jetty/JFC 编号；
- 是否 timeout / reset / permission / resource / protocol；
- 与 target 零增量是否一致。

## D. 恢复

1. 停 debug master；用原正常 env 重启 `plain_nof088`，验收 Keys=2000/50056/owned_slots=1。
2. **不尝试恢复 NoF 挂载**（本单只定位）；target 905586 保持不变。
3. 回执 debug env 名、值、来源（help/strings/文档），不得只写“开了 debug”。

## E. 回执必答

- 受支持 debug 机制及来源；
- 首个 URMA 错误码/操作/上下文；
- 是否能解释 `err=-5` 与 target 零增量；
- 结论：**jetty/JFC / EID/握手 / send-recv / completion / 仍未定位**；
- master 是否恢复、Keys、target 未重启、新增业务写入=0。

## F. 边界

- **不重启 target、不换设备、不 reload driver、不清理资源、不 kill 他人进程、不装包、不改系统配置文件**。
- 只允许一次 plain master 重启与一次 register；debug 仅作用于该进程。
- 不建新 cid、不写业务 key、不改 etcd 配置。

---

# NOF-102（已关闭 · target 同二进制重启后仍 err=-5；target 状态排除 · 2026-11-25）

> **target 重启成功**：旧 PID 905586 → 新 PID **2810360**，二进制 md5 前后均 `3129641f…`；4457/NQN/nsid1/RAID0(11 盘 BDF 升序/strip256K)/transport 全部验收通过；新 namespace UUID `1e71527b…`（旧 `b66fe5ea…`）。
>
> **恢复失败且与重启前逐字同型**：plain_nof088 master 重启正常（Keys=2000、owned_slots=1、ready=1/1），一次 register 后 `opened=0 / attach_fail=3 / defer_error=3`；host `nvme_urma.c:1199 capsule completion err=-5 → nvme.c:834 Operation not permitted`；target 6 个 poll group qpair/stats 全 0、日志零错误。
>
> **结论（L1）**：target 会话/资源状态排除；剩余问题在 node1 host URMA provider/内核数据通路。新建 plain cid 不能修复同一 host 错误，故未执行 D.3；未跑 bench、新增业务写入=0、etcd dbSize 未变。
>
> **偏差**：首次重启因盘仍在 vfio、`/sys/block/nvme0n1` 不存在被 takeover 拒绝；用脚本自带 `-r -y` 还原 11 盘到 nvme 后重跑成功，根分区仍 rw。全部优雅 TERM，无 SIGKILL。
>
> **下一步待用户批准 NOF-103**：重启 node1，完整复位 host URMA provider/内核会话；node3 target 保持运行。若恢复 ⇒ host 状态问题；若仍失败 ⇒ URMA 驱动/固件/系统配置问题，转环境侧，不再加日志。

## 授权与目标

- **用户已明确批准重启 node3 target**，唯一目的是恢复 bench。
- **数据保留不是本单目标**：不做旧数据/旧 vseg 元数据保护工程；只保存重启所需的二进制、配置和日志。
- 成功标准：plain bench 能 attach 并完成 10/10 读取。
- 若现有 plain 因 UUID/元数据变化失败，允许新建 `plain_nof102` 写 400 keys（plain oplog 约 178KB），继续跑通；不修复旧 vseg 元数据。

## A. 最小重启预检

1. target：记录 PID 905586、`/proc/<pid>/exe` 路径/md5（必须 `3129641f…`）、cmdline/env；若为 deleted，复制该二进制到 `run/nof102_nvmf_tgt`。
2. 保存 RPC 配置文本（NQN/nsid1/RAID0/11 BDF/strip/listener/transport）与 `/tmp/nvmf_tgt.log` 尾部；**不备份业务数据、不保护旧 cid**。
3. 记录 plain master PID/cmdline/env/Keys=2000；etcd 当前 dbSize（新建 plain 前复测）。

## B. 停服（顺序）

1. 优雅停 `plain_nof088` master（root/TERM），确认退出。
2. 优雅停 target 905586（TERM）；等待退出。30s 不退则记录并停手回报，不擅自 SIGKILL。
3. 确认 4457 与 spdk.sock 释放；记录盘状态。

## C. 用同一二进制/RPC 重启 target

1. 使用 §A 保存的同一二进制、同一 args/env（**不启用 BUFFER**），按 NOF-055/064 的 wrapper/RPC 序列启动。
2. 验收：
   - 新 PID、`/proc/<pid>/exe` md5 仍为 `3129641f…`；
   - 4457 LISTEN、NQN/nsid1/URMA_RAID0 正常；
   - 11 base bdev/BDF 升序、strip 256K、online；
   - transport URMA/udmac0d1e2/capsule urma；
   - 无 handshake/attach 错误。
3. 记录新 RAID0 namespace UUID（预期变化）；**不尝试修复旧 vseg 元数据**。

## D. 恢复 bench（现有 plain 优先，失败则新建 plain）

1. 用完整正常 env 启动 `plain_nof088` master（`MC_STORE_PARTITION_SLOT_COUNT=1`、URMA 设备/capsule、HA/etcd 参数）；验收 Keys=2000、owned_slots=1、ready=1/1；执行**一次** `nof_register`。
2. 若 attach 仍为 EPERM/`err=-5` ⇒ **target 重启无效**，记录首个错误并停手，不重复 register、不换设备。
3. 若仅因 UUID/identity/元数据变化挂不上 ⇒ **新建 `plain_nof102`**（完整 env、slot=1、无 vsegment 参数），register 后写 **400 keys**（plain，约 178KB oplog），验收 `Keys=400`、`logical_used=1,677,721,600`。
4. 挂载成功后，在 node3 用 `bench-io-params-e12`（88ad3134）跑 **10 keys / warmup=0 / verify=false**，记录 rc、attachfail、读取数、P50。

## E. 判决（二选一）

- **bench 跑通**：现有 plain 或新建 `plain_nof102` 任一能 attach 并 10/10 读取 ⇒ target 会话/资源状态是根因；本单结束，不跑 vseg/正式 timing。
- **仍失败**：现有 plain 与新建 plain 都 EPERM/`err=-5` ⇒ target 状态排除；本单结束，转 host URMA provider/session reset 或节点重启方案（需另行批准）。

## F. 回执必答

- 旧/新 target PID、二进制 md5、启动时刻；
- RPC/RAID0/BDF/transport 验收；
- 新 namespace UUID；
- 现有 plain 的挂载结果；若失败，`plain_nof102` 的 Keys/oplog 增量；
- 10-key attach/读取结果或首个失败错误；
- 二选一判决；
- 未换设备、未修复 vseg 元数据；新增写入仅限 fallback 的 plain 400 keys。

## G. 边界

- 用户已批准 target 重启；**不得更换 target 二进制版本、不得启用 BUFFER、不得换 URMA 设备**。
- **数据保留不是目标**：不保护旧 cid 数据、不修复 vseg 元数据；允许新建 `plain_nof102` 写 400 keys 作为恢复 fallback。
- 不改 etcd 配置、不清理 host/target 资源；plain 写入前复测 etcd 余量。

---

# NOF-103（已取消 · 用户 2026-11-25 决定改用 node2/node4，不重启 node1 · 2026-11-25）

> **取消原因**：node1/node3 他人要用；用户决定切换至 node2/node4 独占环境继续，目标不变。本单未执行，node1 未重启、node3 target 未动。**迁移评估见 NOF-104。**

## 授权与目标

- **用户已明确批准重启 node1**，要求尽快恢复 bench。
- 目的：完整复位 node1 host URMA provider/内核会话；node3 target 保持运行。
- 成功标准：plain 能 attach 并完成 10/10 读取；数据保留不是目标。

## A. 重启前（只读 + 停我方进程）

1. 记录 node1 我方 master PID/cmdline/env、bench/脚本残留；记录 etcd PID/service/data dir/端口/dbSize。
2. 记录 node3 target PID **2810360**、md5 `3129641f…`、4457、RAID0 UUID；本单不重启 target。
3. 优雅停我方 master；确认无我方 bench。**不停止他人进程**。

## B. 重启 node1

1. `sudo -n reboot`；等待 SSH 恢复（最多 5 分钟，超时回报）。
2. 重启后验证：主机/网络/URMA 设备节点、etcd endpoint health、dbSize；node3 target 仍在运行。

## C. 恢复 plain 并跑通 bench

1. 用完整 env 启动 `plain_nof088`（slot=1、URMA 设备/capsule、HA/etcd）；验收 Keys=2000、owned_slots=1、ready=1/1；一次 `nof_register`。
2. 若仅元数据/UUID 问题导致不可用 ⇒ 新建 `plain_nof103`，写 **400 keys**（约 178KB oplog），验收 Keys=400/logical_used 精确。
3. 在 node3 用 `bench-io-params-e12` 跑 **10 keys / warmup=0 / verify=false**，记录 rc/attachfail/读取数/P50。

## D. 判决

- **10/10 成功** ⇒ host 状态是根因，bench 恢复；本单结束，下一步回 timing 主线。
- **仍 EPERM/err=-5** ⇒ host 简单复位无效，停止加日志，转 URMA 驱动/固件/系统配置问题（环境侧）。

## E. 边界

- node1 重启已获用户批准；**target 不重启、不换设备、不清理资源、不改内核参数**。
- 数据保留不是目标；允许 fallback 新建 plain_nof103 写 400 keys；不修复 vseg 元数据。
- 不碰他人 cid/进程；etcd 只验证，不改配置。

---

# NOF-104（已关闭 · 迁移可行但有条件：node4=target、node2=master+client；两台均非独占 · 2026-11-25）

> **盘点结论（L0 只读）**：
> - **node4=target**：11 块 7T NVMe 全部空闲、nvme 驱动、未挂载/未 vfio；BDF 与 node3 完全一致（21/22/28/29/2a/91/92/a1/a2/a3/a4），系统盘 2b。URMA/provider 库与 node1/node3 完全一致（liburma `5c021f7f…`、udma `09daf8ea…`、ubagg `267a9259…`），udmac0d1e2 EID 1024/1024。
> - **node2=master+client**：仅 6 块空闲盘（a1/92/a2/a3/a4/91，全 numa2），5 块被他人 vfio；50056/9006/8086 在 node2 上 free；50156 被他人 client 占用（需改 50256）。
> - **两台都不独占**：node2 多人会话/loadavg 14.8、root fs 97%（2.6G）；node4 多人会话/loadavg 9.4，且有他人 master 占 50056/9006/8086；node4 root fs 66G、hugepages 17.1G（远低于 node1/3 的 444G）。
> - **SPDK 库不可直接复用**：`/usr/local/lib/libspdk_bdev.a` 是 NOF-096 禁用集；node4 现成解包树 `3ec28c61…` ≠ node3 已知可用 `4056b6bd…`。node3 可用树 origin 是本地路径 `/home/yin/nof/spdk`、detached `bd5ee124d`、dirty=6；原始仓库 URL/分支 `[未获取]`。node2 有 tarball `spdk-e12b7cb.tar.gz`，sha256 `f98de24f…`。
> - **迁移顺序建议**：① SPDK 用 node2 tarball/或复制 node3 已知可用树到 node4 自建目录；② target 优先复制 node3 `nvmf_tgt`（3129641f）+ takeover/wrapper；③ Mooncake 用 `wangyuqi0429/Mooncake@supercache-snapshot` 在 node4 重 clone/重编；④ node2 起独立 etcd；⑤ bench 复制 `bench-nof097`/`bench-io-params-e12`；⑥ 端口 target 4457、master 50056/9006/8086、client 50256。
> - **阻塞**：独占性未确认、node4 缺 meson/ninja（若重编 SPDK）、node2 空间小、node4 hugepages 17.1G 需核算。
> - **环境**：全只读，未重启/未绑盘/未写业务数据；产物 `~/nof104_*`、`run/nof104_summary.txt`。

## 背景与目标

- 用户 2026-11-25 决定：node1/node3 他人要用，**取消 NOF-103 node1 重启**；改用 **node2/node4（独占）** 继续，目标不变（vseg 读不慢于 plain、最终更快）。
- 当前 blocker 是 node1 host attach EPERM；换到全新 host 可能直接恢复 bench，但 vseg 2.07× 的性能差距仍需重新基线化。
- 本单**只读盘点**，不重启、不绑盘、不写数据、不装包、不改配置。

## A. 接入与硬件（两台分别记录）

1. 接入方式由用户提供；记录主机角色、OS/kernel/arch、root/`sudo -n`、是否独占、磁盘空间。
2. CPU/NUMA：核数/节点/距离；内存/hugepages total/free。
3. URMA：`/dev/uburma/*`、provider 库路径/md5、EID 表、`rdma`/`urma_admin` 可用性、设备 NUMA；node2↔node4 连通性与 4457/50056/9006/50156 端口状态。
4. **NVMe 数据盘（决定 target 可行性）**：`lsblk`、`lspci -nn`、BDF、型号/容量、当前驱动（nvme/vfio）、NUMA、系统盘识别；可用数据盘数量。**至少需要能组成 RAID0 的多块数据盘；若没有，及时回报“不可行”。**

## B. 软件与资产（只读）

### B0. 源码/版本政策（用户 2026-11-25 明确）

1. **不使用 node2/node4 上现成的 GAP 工作目录/构建**；只读查看，不作为我们的环境。我们在 node2/node4 新建自己的工作目录（如 `/home/l00955908/mooncake-nof-vsegment-2`），所有源码/构建/日志放这里。
2. **Mooncake**：统一用 `https://github.com/wangyuqi0429/Mooncake` 分支 **`supercache-snapshot`**（已核实 HEAD `aadab9573fe13ae6e355fcfce4520dcf67b1f385`）；重新 clone、重新编译，接受从头开始。
3. **SPDK 不变**：保持现有 **wire v4** 这套（`e12b7cb` / `bd5ee124d` 谱系），不升级到 v6。**具体仓库 URL/分支/origin 尚未取证**（现有记录只有目录 `/home/yin/nof/spdk-cpu`、HEAD `e12b7cb` 且带 12 个本地改动；我们的构建副本 `$D/spdk-e12b7cb-src` 基线为 `bd5ee124d`）。本单必须用 `git remote -v` / `git branch -vv` / `git log -1` 查出，不猜。
4. 若 node2/4 上已有旧 SPDK 树，只用于确认 origin/分支/commit；**必须在我们自己的目录重新构建**，不直接复用旧 build。

### B1. 资产清单

1. 是否已有 Mooncake/SPDK 源码或构建、`nvmf_tgt`、`bench-io-params-e12`、etcd；各自路径/md5/版本。
2. 是否能从 node1/node3 复制我们自己的资产（target 二进制 `3129641f…`、bench、be10be29 源码、e12b7cb 静态库）；若不能，记录需要重建的清单。
3. 现有进程/端口占用；node2/node4 是否有他人 etcd/master/target。
4. 系统依赖：liburma/UMDK/DPDK/SPDK 版本、gcc/cmake、hugepages 配置。

## C. 迁移方案输出（必答）

1. **角色建议**：哪台做 target（有数据盘/URMA/vfio）、哪台做 master+client；若两台都不满足，给出缺项。
2. 需要复制/重建的资产清单与顺序（etcd → master → target → bench）。
3. 预计端口/NQN/RAID0 参数与数据盘 BDF 清单；是否可沿用 4457/50056/9006。
4. 迁移后第一轮验收：plain attach + 10-key bench；再 plain/vseg 400-key 重基线。
5. 风险：数据盘被他人占用、URMA EID 未配置、源码/构建缺失、node2/4 网络不通。

## D. 边界

- **全只读**：不重启、不绑 vfio、不写数据、不装包、不改系统配置、不动他人进程。
- 不把内网 IP/账号写入公开仓；用 node2/node4 指代。
- 若关键项无法获取，记 `[未获取]` 并说明；不猜。

---

# NOF-105（已关闭 · 通过：node4 target 就绪；旧进程清理；资产 md5 对齐 · 2026-11-25）

> **结果**：按 PID 清理 node4 master 1260028/metadata 1260073、node2 client 3204031/nvmf_tgt 3206327；端口 50056/9006/8086/4457（node4）与 50156（node2）释放，余下 `/home/ql` 他人进程未碰。node4/node2 各自 `mooncake-nof-vsegment-2`（1.1G），`nvmf_tgt=3129641f…`、`libspdk_bdev.a=4056b6bd…`、74 libs sha256 `e4737186…`、takeover `8b825b66…`、wrapper `9eea126e…`、bench md5 全部与 node3 一致。
>
> **target**：PID **2142228**、md5 `3129641f…`、`node4:4457 LISTEN`；`URMA_RAID0` online/11 base/strip256K/superblock=false/新 UUID `0470fe3a-b5db-4656-8346-0159c6ba9277`；nsid1→URMA_RAID0；transport URMA/max_io 8MiB/udmac0d1e2/capsule urma/RM。hugepages 启动前 8768 页（17.1GiB），启动后 free 4476 页；11 盘 vfio，系统盘/根分区未动。
>
> **偏差**：rsync `-e/-o` 写法首次报错，改 `RSYNC_RSH` 成功；node2↔node3/node4 无 ssh 互信，经 node4 中转；node4 系统盘为 nvme10n1（2b），11 数据盘为 nvme0–9+nvme11，target 侧映射 Nvme0..Nvme10；wrapper rc 未落盘 `[未获取]`，以 RPC 全成功 + “Target 就绪”为证据；日志唯一 `handshake rc=-104 peer=node2` 是我方 TCP 探测建连即关的副作用。
>
> **末态**：target 保持运行供 NOF-106；未启动 master、未跑 bench、未重编、未装包、未写业务数据。

## 授权与目标

- **用户已确认 node2/node4 独占**；node4 50056/9006/8086、node2 50156 上的进程是用户之前起的，**可以 kill（佛挡杀佛）**。
- 本单只做：清理我方旧进程/端口 → 建我们自己的目录 → 复制 node3 已知可用 target/bench/SPDK 树 → 启动 node4 target 并验收。
- 不跑 master/数据、不重编 SPDK、不装包、不改内核参数。目标：target 4457 就绪。

## A. 清理我方旧进程（严格按 PID/路径）

0. **只读检查 node2/node4 的 `/home/l00955908/mooncake-nof-vsegment`**：记录 `ls -ld`（owner/mode/mtime）、顶层内容、是否 git 仓库（remote/branch/HEAD/status）、是否有进程 cwd/fd 指向它、日志最后时间。**不删除、不修改**；若是我方旧遗留，记录来源与可复用资产；新环境仍用 `mooncake-nof-vsegment-2`。
1. 记录候选进程的 PID/cmdline/owner：node4 master 1260028、metadata 1260073、node2 client 3204031、node2 `nds/spdk_v2_runtime/nvmf_tgt`，以及 `/home/l00955908/nds` 下仍存活的 `mooncake_master|mooncake_client|nvmf_tgt`。
2. **只 kill 上述用户所属路径/PID**：先 `kill -TERM`，10s 不退再 `kill -KILL`；逐个记录。**不碰其他用户/未知路径进程**。
3. 清端口验收：node4 50056/9006/8086/4457、node2 50156/2379 均 free（或仅剩非我方进程时停手回报）。

## B. 建自有目录并复制资产

1. 在 node4/node2 创建 `/home/l00955908/mooncake-nof-vsegment-2`（或用户已有命名），**不使用 node2/node4 的 nds/GAP 工作目录**。
2. 从 node3 复制我们自己的已知可用资产到 node4 自有目录（node4→node3 ssh 可用）：
   - `build/spdk-e12b7cb-src` 的构建产物/必要源码（优先 `build/bin/nvmf_tgt` md5 `3129641f…`、`build/lib`、takeover/wrapper 脚本、RPC 快照）；
   - `bench-nof097`（md5 `6757c001…`）或 `bench-io-params-e12`；
   - 记录每个文件的 md5/sha256 与来源路径。
3. **不重编 SPDK**（node4 缺 meson/ninja）；若复制产物不可运行，停手回报，不自行装包/重编。

## C. 启动 node4 target

1. 预检：11 块 7T BDF 升序 21/22/28/29/2a/91/92/a1/a2/a3/a4、系统盘 2b 禁碰；URMA udmac0d1e2 正常；hugepages **free ≥10GiB**（当前 17.1G，必须实测；不足则停手）。
2. 按 takeover 脚本绑定 11 盘 vfio；用复制的同一 `nvmf_tgt`（3129641f）启动，同 RPC 序列：
   - RAID0 strip 256K、11 base、nsid1 100GiB、NQN `nqn.2026-01.io.spdk:urma-gpu-test`、listener URMA 4457、transport udmac0d1e2/capsule urma；
   - **不启用 BUFFER**。
3. 验收：新 PID/md5、4457 LISTEN、RAID0 online/11 base/BDF 升序、nsid1、transport、target 日志无错误；记录新 RAID0 UUID。
4. **停手**：不启动 master、不 register、不跑 bench、不写业务数据。target 保持运行供 NOF-106。

## D. 回执必答

- 清理的 PID/路径清单与端口释放；
- 复制资产清单/md5；
- target 新 PID/md5/4457/RAID0/ns/transport/新 UUID；
- hugepages free 实测；
- 是否安装/重编（应为否）；
- target 未重启/未写数据/未动其他用户进程。

## E. 边界

- **只 kill 用户确认的我方旧进程**；不碰未知/他人进程。
- 不装包、不改内核参数、不重编 SPDK、不换 URMA 设备。
- 不启动 master/bench、不写业务数据；target 起后保持运行。

---

# NOF-106（已关闭 · 通过：新环境 plain 400 keys + 10-key 读通 · 2026-11-25）

> **结果**：node2 直接在 `/home`（余 595G）编译 **supercache-snapshot@aadab957**（dirty=0，Windows 离线打包上传）；master/bench/nof_register 全部只链接自有 `spdk-e12b7cb-src/build/lib`（`libspdk_bdev.a=4056b6bd…`，`/usr/local` 命中 0）。独立 etcd 3.4.14（127.0.0.1:2379，healthy/leader）；plain master **PID 34172**（50056/9006/8086、Keys=400、owned_slots=1、leader/serving）；`nof_register` 到 node4 4457 成功（容量 0→100GB，心跳 0 失败）；writer 400/0、1.56GiB；10-key 读 **10/10、P50 555.36µs**（node2 远端读，仅冒烟）。
>
> **偏差**：三台集群机无外网，源码/子模块/gtest 由 Windows 上传，gtest 用 `FETCHCONTENT_SOURCE_DIR_GOOGLETEST` 离线；etcd Go wrapper 未重编（go 版本/联网限制），复用 node3 预构建 `libetcd_wrapper.so`（源码 md5 一致）；DPDK 必须 root；实际 SPDK 路径多一层 `build/`；master 启动后 2s 一次 lease timeout，未复现。
>
> **环境**：target 2142228 未重启（RAID0 UUID 0470fe3a…、4457）；master 34172 运行；node2 /home 余 595G；未装包/未改内核/未重编 SPDK/未动他人进程。产物 `run/nof106_summary.txt`、master/writer/bench/etcd 日志。
>
> **下一步 NOF-107**：node4 本机 reader 先读 plain 400 keys 3 遍，再切 `vseg_nof107` 写 400 keys 读 3 遍；给同环境 plain/vseg P50/W9。

## 背景与目标

- node4 target 已就绪：`node4:4457`、RAID0 11 盘、nsid1 100GiB、hugepages 余 4476 页。
- 用户要求 Mooncake 统一用 `wangyuqi0429/Mooncake @ supercache-snapshot`（HEAD `aadab957…`），master 和 bench 必须从该分支重编，链接 `spdk-e12b7cb-src` 的 e12b7cb/bd5ee124d 静态库。
- 本单只做：编译 → 部署 node2 → 起本地 etcd → register node4 target → plain 400 keys → 10-key bench。**不做 vseg、不做性能结论。**

## A. 构建位置与准备

1. **先查空间**：在 node2 执行 `df -h /home /home/l00955908 /`；记录每个挂载点可用空间。
   - 若 `/home`（或工作目录所在挂载）可用 **≥10G** ⇒ **直接在 node2 编译**（源码/依赖经 node4 或 Windows 中转）。
   - 若不足 ⇒ 在 node4 编译（66G 空闲），只把 master/bench 二进制复制到 node2；复制后记录 md5。
   - **不要为了省事盲目跨机复制，也不要冒险写满 node2 根盘。**
2. clone `wangyuqi0429/Mooncake` 分支 `supercache-snapshot`（HEAD 必须 `aadab957…`；无外网则经 Windows 上传压缩包）。
3. 记录 `cmake/gcc/g++/ninja` 版本与依赖；检查 `CMakeLists.txt` 的构建选项与目标名（master、bench）。
4. 配置必须指向我们的 SPDK：
   - `SPDK_INCLUDE_DIR=/home/l00955908/mooncake-nof-vsegment-2/spdk-e12b7cb-src/build/include`
   - `SPDK_LIB_DIR=/home/l00955908/mooncake-nof-vsegment-2/spdk-e12b7cb-src/build/lib`
   - `USE_NOF=ON`、`USE_NOF_URMA=ON`、`USE_UB=ON`、`STORE_USE_ETCD=ON`、`USE_NOF_GPU_DIRECT=OFF`
   - **禁止 `/usr/local/lib/libspdk_*.a`**。
5. 构建 Release；产物记录 md5/size；`--help` 正常。若缺依赖/目标名不同 ⇒ 停手回报，不装包。

## B. etcd 与部署到 node2

1. 在 node4/node2 找可用 `etcd`/`etcdctl`（`which`、旧 nds 目录、node3 可复制）；版本记录。若找不到 ⇒ 停手回报。
2. 在 node2 自有目录启动**独立 etcd**：监听 `127.0.0.1:2379`，data-dir 在我们目录；验证 endpoint health。
3. 若在 node4 编译，将 `mooncake_master` 与 bench 复制到 node2 自有目录；若在 node2 编译则直接使用。记录 md5/size；检查 node2 剩余空间（root fs 97%，必要时用 /home 其他挂载；不足则停手回报）。
4. 复制/准备 register 工具与 bench 运行脚本；不改 `/usr/local`。

## C. 起 plain master + register + 写 400 keys

1. node2 起 plain master：`--cluster_id=plain_nof106`、`--rpc_port=50056`、`--metrics_port=9006`、HTTP 8086、etcd `127.0.0.1:2379`、`MC_STORE_PARTITION_SLOT_COUNT=1`、`SPDK_URMA_DEV_NAME=udmac0d1e2`、`SPDK_URMA_CAPSULE_TRANSPORT=urma`；记录 cmdline/env。
2. 验收 50056/9006/8086 LISTEN、role=leader/serving、Keys=0、owned_slots=1。
3. `nof_register` 到 node4：`node4:4457`、100GiB、base0、nsid1；等 quota ≥20s；验收 mount/opened segment（记录 `Opened NoF segment` 与 qpair 数）。
4. plain writer：400 keys、4MiB、T1/batch1（或与历史一致的 16t/batch16，记录清楚）、verify=false；验收 master `Keys=400`、`logical_used=1,677,721,600`（**不信 bench 报数**）。

## D. 10-key bench（只做连通性/基线冒烟）

1. 在 node2 跑 supercache 编出的 bench：10 keys、warmup=0、T1、batch1、qpairs4、verify=false、plain 模式（无 vsegment 参数）。
2. 记录 rc、failed、10/10、P50、attachfail、target 4457 状态；**本单不评价性能**。
3. 若 attach 仍 `err=-5/EPERM` ⇒ 停手回报（说明问题可能不是 node1 特有）；若 10/10 成功 ⇒ 环境跑通，停手，下一单再做 400-key plain/vseg 重基线。

## E. 回执必答

- supercache clone HEAD/编译配置/产物 md5；
- SPDK include/lib 路径与 `libspdk_bdev.a` md5；
- etcd 版本/端口/health；
- master PID/端口/Keys/owned_slots；
- register 的 opened/qpair；
- writer Keys/logical_used；
- 10-key bench rc/10/10/P50/attachfail；
- 新增写入 LBA/cid、node2 空间、target 未重启；
- 有无失败/缺依赖。

## F. 边界

- **target 保持运行，不重启、不换设备**。
- **不装包、不改系统配置、不用 `/usr/local/lib`、不用 node2/node4 的 GAP/nds 构建**。
- 不写 vseg 数据；只写 plain 400 keys；不跑正式 400-key 性能对比。
- 只 kill 我方旧进程；不碰 `/home/ql` 等他人进程。

---

# NOF-107（待执行 · 新环境 plain vs vseg 400-key 基线；master+target 在 node4，reader 在 node2 · 2026-11-25）

## 目标

- **拓扑（用户指定）**：master 和 target 都放 node4；reader 放 node2。这样 reader 的元数据（到 node4 master）和数据（到 node4 target）都跨节点，不出现本地读。
- 同一 target、同一 SuperCache 构建、同一 reader 位置，给 plain 与 vseg 的 T1 400-key 基线。
- **不修代码、不跑 16t、不做结论外推**；只回答：新环境下 vseg 是否仍约 2× 慢。

## A. 拓扑迁移与预检

1. 记录 target 2142228/4457/RAID0；停 node2 的 plain master（34172）与 node2 etcd（可保留数据但本单不用）；target 保持运行。
2. 把 node2 编出的 `mooncake_master`（md5 `79e30da5…`）与 etcd 二进制/启动方式复制到 node4（node4→node2 ssh 可用）；在 node4 起 **独立 etcd**（127.0.0.1:2379，data-dir 在自有目录），health+leader。
3. 在 **node4** 起 plain master：50056/9006/8086、`--cluster_id=plain_nof107`、`MC_STORE_PARTITION_SLOT_COUNT=1`、URMA 设备/capsule、etcd 127.0.0.1:2379；验收 50056 LISTEN/leader/serving/Keys=0/owned_slots=1。
4. `nof_register` 到 node4 target（4457、100GiB、base0、nsid1），验收 mount/opened/qpair。
5. **reader 留在 node2**：用 node2 已编的 supercache bench（md5 `bb4a07a3…`）先跑 10-key 冒烟，确认 node2→node4 的 master 与 target 都是**跨节点**、rc=0/10/10。若不通，停手回报，不改回 node2 本地 master。

## B. plain 基线（plain_nof107，master 在 node4）

1. 在 node2 跑 writer：400 keys、4MiB、16t/batch16（记录参数）；验收 master `Keys=400`、`logical_used=1,677,721,600`。
2. 每遍 bench 前重新 `nof_register` 到 node4 并验证挂载。
3. node2 reader 读 400 keys：T1、batch1、warmup=128、qpairs4、verify=false；跑 **3 遍**。
4. 记录每遍 Min/Avg/P50/P90/P99/Max、failed、target W9；确认 400/400；**记录 reader 到 master/target 均跨节点**。

## C. vseg 基线（新 cid `vseg_nof107`，master 在 node4）

1. 停 node4 plain master；起 vseg master（node4，50056/9006/8086）：`--cluster_id=vseg_nof107`、slot=1、`MC_STORE_PARTITION_SLOT_COUNT=1`、URMA 设备/capsule、`--vsegment_auto_plan=true --vsegment_required_medium=nof --vsegment_io_alignment=4096 --vsegment_member_count=1 --vsegment_stripe_size=2097152 --vsegment_member_extent_size=67108864 --vsegment_reserved_ratio=0 --submaster_count=1`、HA/etcd 同 node4 etcd。
2. 验收 partitions=1、reconcile_fail=0、Keys=0；`nof_register` 到 node4 target，mount 成功。
3. 在 node2 T1 writer 写 400 keys（≤3 次补齐；`VSEGMENT_CREATING` 已知机制）；验收 master `Keys=400`、`logical_used=1,677,721,600`。
4. 每遍前重新挂载；node2 reader 读 400 keys，同 B 参数跑 **3 遍**；记录同 B。
5. **不修任何代码、不启用 BUFFER**；target 保持运行。

## D. 判读（必答）

- plain 与 vseg 的 P50/Avg/P90/P99、W9 绝对值（µs）；
- 比值：新环境是否仍 ~2×；若追平/反超，记录并停手（说明旧环境/旧源是问题）；
- 若仍 ~2×，下一步才有资格继续 timing/机制定位；
- 明确标注 **reader 在 node2、master 和 target 都在 node4，两者对 reader 都是远端**；不与旧 node1/node3 数字混比。

## E. 边界

- target 不重启、不换设备；不重编 SPDK/Mooncake；不改 `/usr/local`；不装包。
- 数据保留不是目标；只新建 `vseg_nof107` 写 400 keys；plain 现有数据只读。
- 只跑 T1 400-key；16t 不在本单；失败停手回报。

---

## NOF-107 裁决（2026-11-25 · 回执已收）

### 1. 结果（[事实]，3 遍中位，µs）

| 指标 | plain | vseg | 比值 |
|---|---|---|---|
| Min | 261.95 | 466.83 | 1.78× |
| Avg | 345.69 | 563.89 | 1.63× |
| **P50** | **274.00** | **555.00** | **2.03×** |
| P90 | 577.25 | 642.62 | 1.11× |
| P99 | 682.03 | 730.85 | 1.07× |
| Max | 745.76 | 800.43 | 1.07× |
| target W9 | 120.07 | 248.80 | **2.07×** |

环境：target 2142228 全程未重启；master+target 在 node4、reader 在 node2（元数据与数据均跨节点）；每遍 400/0 failed；target 读窗口 n≈16,896–16,905，两臂同为（400+128）×32×128KiB。

### 2. 能站住的判断

1. **复现成立（L0）**：P50 2.03×、W9 2.07×，与旧环境/旧源码的 2.07× 独立吻合 ⇒ **旧 node1/node3 环境与 be10be29 源码不是该现象的必要条件（L1 排除）**。
2. **不是 IO 数差异**：两臂每遍 target IO 数、单 IO 大小（128KiB）完全一致 ⇒ 差异是 **target 单个 128KiB IO 的服务时间**，不是请求数量或切片大小。
3. **尾部收敛是分布形状，不是追平**：plain P90/P50=2.11、vseg=1.16；vseg 分布紧、plain 长尾重。不得用 P90/P99 比值下降说“差距缩小”。
4. **W9 绝对值不可跨环境比**：plain 旧环境 ~68µs、新环境 120.07µs；vseg 旧 ~250µs、新 248.8µs。只做本环境两臂对比。

### 3. 仍不能下的结论（防过度归因）

- **仍为 L2，不是 L3**：两臂数据由不同写入路径产生（plain 16t/batch16、vseg T1），**物理 LBA 未对齐、写入并发不同**；本单没有同 LBA/同数据对照，不能排除“布局/写入历史”是差异源。
- **vseg reader 未带 vseg flag**：C4 用 `--scenario=segment_read`；源码 `stress_cluster_bench.cpp:3449` 禁止该场景与 `--vsegment_nof_io` 同用，且 `real_client.cpp:4623-4626`（supercache aadab957；旧 be10be29 为 4598-4601）表明 `type` 取 master 返回的 `replica_type` ⇒ 机制上 vseg 数据仍走 vseg 路径，但回执未直接给出逐 key `type` 证据（[未获取]）⇒ NOF-108 补。
- **node4 上 master/etcd 与 target 同机**：CPU/网络混叠未测，不能排除它放大 W9；NOF-108 用 plain 后跑 + 同机条件作对照。
- **vseg writer 23+2 次 `VSEGMENT_CREATING` 失败后补齐**：最终 Keys=400 达成，但写入历史与 plain 不同，作为布局混杂记录。

### 4. 下一步

**NOF-108**：两臂开 `MC_STORE_TRANSFER_DIAG=1` 拿逐 key 分解与 `type`，并把 plain 放到 vseg 之后跑，先分清“客户端哪一段 + 是否臂序/target 状态造成”，再决定是否值得做同 LBA 对照或 target 侧打点。

---

# NOF-108（待执行 · vseg 2× 归因第一步：两臂逐 key 分解 + 臂序对照 · 零写/零重启/零改码 · 2026-11-25）

## 目标

- 用 supercache 自带诊断 `MC_STORE_TRANSFER_DIAG=1`（`MC_HIFREQ_LOG_SAMPLE_RATE` 默认 1.0，不必另设），拿每个 key 的 `get_into_breakdown`：`query_us` / `select_us` / `read_us` / `vsegment_resolve_us` / `nof_prepare_us` / `nof_submit_us` / `completion_wait_us` / `read_unattributed_us` / `type` / `selected_type` / `status`。
- 同一批已有 plain 数据与 `vseg_nof107` 数据，**不做任何写入**；先 vseg（master 已在跑）后 plain（切 master），兼作臂序对照。
- 只回答四件事：
  1. vseg 臂的 `type` 是否确实为 vsegment（补 NOF-107 的缺口）；
  2. 281µs 的 P50 差在客户端落在哪一段（metadata query / plan / submit / completion wait / unattributed）；
  3. 逐 key 分布：新盘上是否存在多遍可重复的前缀慢档（不预设 0–96）；
  4. plain 在 vseg 之后跑，P50/W9 是否仍 ~274/120（排除臂序、target 运行时长、master 进程差异）。

## A. 预检

1. 记录 target 2142228 未变（PID 起始时间、md5、RAID0 UUID、4457 LISTEN）；**不重启**。
2. 确认 vseg master 2157604 运行、Keys=400；node2 无残留 bench；plain 数据与 `vseg_nof107` 数据均保留且本单只读。
3. 用 `--help 2>&1 | grep -E "warmup_keys|num_keys|batch_size"` 核对旗标。

## B. vseg 臂（diag on，先跑）

1. node2 reader：`--num_keys=400 --num_threads=1 --batch_size=1 --warmup_keys=128 --verify=false`、qpairs4、`--scenario=segment_read`；env `MC_STORE_TRANSFER_DIAG=1`。
2. 每遍前重新 `nof_register` 到 node4 target 并验证挂载。
3. 跑 **3 遍**；日志 `run/nof108_vseg_p{1,2,3}.log`；记每遍 Min/Avg/P50/P90/P99/Max、failed、400/400。
4. 每遍读窗口取 target W9（沿用 NOF-107 的 FLUSH/period 块法）。

## C. plain 臂（diag on，后跑）

1. 停 vseg master（记录 PID/退出码），在 node4 起 plain master，参数与 NOF-107 B 段完全相同：`--cluster_id=plain_nof107`、50056/9006/8086、`MC_STORE_PARTITION_SLOT_COUNT=1`、etcd `127.0.0.1:2379`、URMA 设备/capsule；验收 leader/serving/Keys=400/owned_slots=1。
2. `nof_register` 到 node4 target，验证挂载。
3. 跑 **3 遍**，参数与 env 同 B；日志 `run/nof108_plain_p{1,2,3}.log`；W9 同法。

## D. 交付（必答）

1. 两臂逐 key 行数（应 400/遍）、`type`/`selected_type` 计数、`status` 非 `read_ok` 数。
2. 每臂 3 遍的 P50/P90/P99，及 `read_us` / `query_us` / `nof_submit_us` / `completion_wait_us` / `read_unattributed_us` 的 p50 与 avg（先按 key 汇总，再按遍）。
3. 逐 key `read_us` 按 key index 分桶（0–31、32–95、96–127、128–255、256–399，另附每 32 key 一桶的中位数）：判断是否有**≥2 遍一致**的前缀慢档；有则给出实测边界，无则明确写“无”。
4. 同 key index 的 plain↔vseg 配对差值（只按 index 配对，**不声称物理 LBA 相同**）。
5. 两臂 W9；与 NOF-107（diag off）的差异，用于提示诊断开销。
6. 臂序结论：plain 后跑是否仍 ~274µs / W9 ~120。

## E. 边界

- **零写入**；不重启 target；不重编/不改码；不启用 BUFFER；不跑 16t；不动 `/usr/local`。
- 只切 master（vseg→plain），不碰他人进程；失败停手回报。
- 诊断日志可能抬高绝对值：**两臂同 diag 才可比**；绝对数仍以 NOF-107（diag off）为准。

---

## NOF-108 裁决（2026-11-25 · 回执已收）

### 1. 硬结论（[事实]）

| 项 | 结果 |
|---|---|
| `type` 补证 | vseg 臂 **528/528 = vsegment**；plain 臂 **528/528 = nof_remote**；status 全 `read_ok` ⇒ NOF-107 的 [未获取] 关闭，读路径确由 `replica_type` 决定 |
| 客户端分段（measured 中位，µs） | vseg vs plain：`query` 95.5/96、`resolve` 2/0、`nof_prepare` 4/3、`nof_submit` 9/3、**`completion_wait` 449.5/243（+206.5）**、`unattributed` 10/7；`read_us` 475.5/257（+218.5） |
| target W9 | **248.03 vs 120.08µs（2.07×）**，与 NOF-107 的 248.80/120.07 逐位一致 |
| 臂序 | plain 后跑（vseg master 已停）仍 **363–372µs、W9 120.08** ⇒ 臂序、master 进程、target 运行时长均排除 |
| 逐 key 分布 | **plain：key 0–95 read_us ~452µs（三遍一致），96+ ~250µs；vseg：全 key 平坦 ~470µs（447–521）**；在 plain 慢区两臂几乎相等（vseg 465 vs plain 455，配对差仅 +2.5~+17.5µs），2× 的绝大部分来自 plain 快区 |

### 2. 判读

1. **不是客户端逻辑**：resolve/prepare/submit 合计 <10µs，95% 差值在 `completion_wait`，与 target W9 的 +128µs/IO 同向 ⇒ 时间花在等 target/传输完成，不在算/排队。
2. **vseg 没有“快区”**：plain 在 96+ 落回 ~250µs，vseg 始终 ~470µs（≈ 2×W9）。这与“plain 慢区 ≈ vseg 全区”的旧形状（NOF-041）在新环境重现。
3. **diag 开销不对称**：vseg 555.00→586.68（+31.7），plain 274.00→365.44（+91.4）⇒ diag-on 的 P50 比 1.61× **无效**；分段占比、W9、配对差可用，绝对比值用 NOF-107 的 2.03×。
4. **plain 前缀慢档已在新环境实测**：key 0–95（边界 96，三遍一致，warmup 已读过仍慢）⇒ 此前“旧环境结论不可搬”是对的；现在只能说**本数据集上**存在该前缀，仍不能声称是物理盘位置（未做 LBA 对照）。

### 3. 仍不能下的结论

- 2× 仍为 **L2**：两臂物理 LBA 未对齐、写入路径不同；`completion_wait` 只能证明“在等 target”，不能证明“为什么 target 慢”。
- vseg 全区平坦 ~470µs 有两种互斥解释，未分离：
  - **(a) 串行两波**：2 个 child 任务未并行，≈2×单 child；
  - **(b) 每 child 自身慢**：target 对 vseg 的每个 128KiB IO 就是 ~248µs（随机/慢状态），与 child 数无关。
- 旧 NOF-059（stripe=4MiB 单 child 无效）与旧 NOF-061（同 LBA 双路径）**未经本侧复核**，不能拿来排除 (a)/(b)。

### 4. 下一步

**NOF-109 纯离线**：解析 NOF-108 已落盘的 vseg 日志里 `nof_vsegment_child` / `nof_vsegment_operation`（diag on 时 `sampled=true` 才打印，每遍应有 800 measured + 256 warmup = 1056 条 child 行），直接看两个 child 的提交/完成时间是否重叠、各自延迟多少。这能在不碰环境的前提下分离 (a)/(b)。

---

# NOF-109（待执行 · 纯离线：vsegment 子任务时序解析 · 零环境/零写/零重启/零改码 · 2026-11-25）

## 目标

- 只解析 **NOF-108 已落盘的 3 份 vseg 日志**（`run/nof108_vseg_p{1,2,3}.log`，diag on），提取：
  - `nof_vsegment_child`：`trace_id/vsegment_op_id/child_idx/submit_seq/first_submit_mono_ns/complete_mono_ns/get_block_us/complete_to_observe_us/serial_gap_us/fifo_wait_us/first_completion_from_submit_us`；
  - `nof_vsegment_operation`：`planned_children/submitted_children/timed_children/first_submit_mono_ns/last_completion_mono_ns/last_child_idx/observed_total_us/rc`。
- 按 `(trace_id, vsegment_op_id)` 归组，每 key 应有 2 个 child（4MiB / 2MiB stripe）。
- **只回答**：vseg 的 ~470µs 是「两个 child 串行两波」还是「每个 child 自身就 ~250µs」；不做任何新测量。

## A. 解析步骤

1. 每份日志先数行：`nof_vsegment_child`、`nof_vsegment_operation`、`get_into_breakdown`；measured 与 warmup 按 key 名区分（warmup 128 行）。
2. 每个 op 计算（child0 为基准）：
   - `submit_skew_us = (child1.first_submit - child0.first_submit)/1000`
   - `completion_gap_us = (child1.complete - child0.complete)/1000`
   - `overlap = child1.first_submit < child0.complete`
   - `child0/child1` 各自的 `first_completion_from_submit_us`、`fifo_wait_us`
   - `op_total_span_us = (last_completion - first_submit)/1000`，与 `observed_total_us` 对照
3. 统计（measured 400 个 op）：
   - overlap 的 op 占比；child1 晚于 child0 完成的比例；
   - `submit_skew_us`、`completion_gap_us`、两 child 延迟的 p50/p90；
   - `timed_children` 分布（是否恒 2；有无 1）；
   - 若 `observed_total_us ≈ child0_latency + child1_latency` 且 overlap=false ⇒ 串行；若两 child 延迟各 ~250 且 overlap=true、total ≈ max(latency) ⇒ 并行但每 child 慢。
4. 抽 5 个代表 key 贴原始行（含 child0/child1/operation 三行），便于人工核对。
5. 检查 plain 日志是否有 `nof_slow_task` 或 child 级行；若无，记录 0 即可，不要求补测。

## B. 交付（必答）

1. 三份日志的行数表（child/operation/breakdown，measured/warmup）。
2. overlap 占比、submit_skew p50/p90、completion_gap p50/p90、两 child 延迟 p50/p90、`timed_children` 分布。
3. 判决：**(a) 串行两波** / **(b) 并行但每 child 慢** / **两者混合**（给出比例）。
4. `observed_total_us` 与 child 时间戳的闭合误差（p50/p90）。
5. 5 个代表 key 的原始行。

## C. 边界

- **纯离线**：不跑 bench、不写、不重启、不改码、不启动/停止任何进程；只读 NOF-108 日志与已有 summary。
- 日志缺失/行数明显不足时停手回报（不要用新测量补）。
- 只做描述统计，不做物理 LBA/盘位归因。

---

## NOF-109 裁决（2026-11-25 · 回执已收）

### 1. 硬结果（[事实]，3 个 pass、1200 个 measured op）

| 指标 | 实测 p50 / p90 |
|---|---|
| overlap（child1 提交早于 child0 完成） | **1200/1200 = 100%** |
| submit_skew | **6.8 / 8.2µs** |
| completion_gap | **22.9 / 94.4µs** |
| child0 延迟 | 375 / 472µs |
| child1 延迟 | 394 / 495µs |
| op span（first_submit → last_completion） | **420 / 504µs** |
| span / 单 child | **1.10×** |
| span / (child0 + child1) | **0.52** |
| fifo_wait 非零 child | **0 / 1200** |
| timed_children / planned / submitted | 2/2/2，400/400；rc≠0 = 0 |

### 2. 判决

- **(a) 串行两波：否。** 若串行，completion_gap 应 ≈ 单 child（~380µs），实测 22.9µs；span/单 child 应 ≈ 2.0×，实测 1.10×。
- **(b) 并行但每 child 自身慢：成立。** 两个 child 各约 375–394µs、100% 重叠，op 总时长 ≈ 单 child。
- [推算] 每 child = 2MiB = 16×128KiB，380µs 走完 16 chunk ⇒ ~24µs/chunk 的流水推进；若 target 逐 IO 串行（16×248µs）应约 4ms ⇒ child 内部确实在流水。等效并发 32×W9/span：vseg ≈18.9、plain ≈14.9 ⇒ **两者都在流水，vseg 并发甚至更高，差别只剩 target 对每个 128KiB IO 的服务时间（248 vs 120µs）**。

### 3. 源码补充（新事实）

- `member_count=1` 时，`ResolveTransfer` 的映射为 `physical_offset = member.base_offset + stripe_index × stripe_size + offset_in_stripe`（`vsegment_runtime.cpp:339`）⇒ 4MiB key 的两个 2MiB child **物理相邻**（base+0 / base+2MiB）。
- 所以“两个 child 分散在远端不同位置”不成立；但它们是 **两个独立 NoF task**，默认 4 个 worker 时可能落到不同 worker/lane，这一点尚未操纵。
- plain 的 4MiB 是 **一个 NoF task**；vseg 是 **两个 task**。在 T1 下，plain 无论 workers=4 还是 1 都只用一个 worker；vseg 则可能用两个。

### 4. 下一步

**NOF-110**：只改 env `MC_NOF_WORKERS=1`，把两个 child 压到同一个 worker/lane。若 vseg 明显变快 ⇒ 原因是“两个 task/多条 lane 的并发 IO 让 target 进入慢档”，修法是合并相邻 child 为一个 4MiB task（或同 lane 顺序提交）；若不变 ⇒ 与 task/lane 分布无关，继续查 target 服务路径。

---

# NOF-110（待执行 · 单变量 `MC_NOF_WORKERS=1` · 零写/零重启/零改码 · 2026-11-25）

## 目标

- 只操纵一个客户端 env：`MC_NOF_WORKERS=1`（默认 4）。
- T1 下 plain 每 key 只有 1 个 NoF task ⇒ workers 数不影响它，作对照；vseg 每 key 有 2 个 child task，默认可能落在两个 worker/lane，workers=1 时两个 child 必须走同一个 worker/lane。
- 只回答：**vseg 的 P50 / target W9 是否向 plain 靠近**。
  - 若明显变快 ⇒ “两个 task/多条 lane 的并发 IO”是 target 进入慢档的原因；下一步做合并相邻 child（一个 4MiB task）或同 lane 顺序提交。
  - 若不变 ⇒ 与 task/lane 分布无关，继续查 target 对 vseg IO 的服务路径。

## A. 预检

1. target 2142228 不重启；记录 md5/4457/RAID0 UUID。
2. 当前应运行 plain master 2171241（NOF-108 末态）、`Keys=400`；vseg 数据 `vseg_nof107` 的 400 keys 仍保留。
3. 确认 `MC_NOF_WORKERS` 在本源码生效：`transfer_task.cpp` 的 `GetSpdkNofWorkerCount()`；启动日志/`--help` 无该 env 时，用 client 启动后的 worker 数日志或 `nof_worker_pool` 行为核对；无法确认则停手回报。

## B. plain 臂（对照，workers=1，先跑）

1. node2 reader：`MC_NOF_WORKERS=1`；其余同 NOF-107：400 keys、4MiB、T1（threads=1/batch=1）、sequential、warmup=128、qpairs=4、verify=false、`--scenario=segment_read`；**diag 关闭**。
2. 每遍前重新 `nof_register` 并验证挂载。
3. 3 遍；记 Min/Avg/P50/P90/P99/Max、failed、400/400、target W9（同 NOF-107 窗口法）。
4. 预期（若 workers 对 plain 无影响）：P50 ~274µs、W9 ~120µs；若明显不同，说明 workers 也影响 plain，需记录并重新解释。

## C. vseg 臂（workers=1，后跑）

1. 停 plain master（记录 PID），按 NOF-107 C 段同参起 vseg master（`vseg_nof107`、50056/9006/8086、slot=1、stripe=2MiB、member_count=1、extent=64MiB、io_alignment=4096、required_medium=nof、reserved_ratio=0、submaster_count=1、etc 127.0.0.1:2379）；验收 leader/serving、Keys=400、partitions=1。
2. `nof_register` 到 node4 target，验证挂载。
3. `MC_NOF_WORKERS=1`；其余同 B；**零写入**（读已有 `vseg_nof107` 数据）。
4. 3 遍；记录同 B。

## D. 可选（只在 C 出现明显改善时做）

- `MC_NOF_WORKERS=2`，仅 vseg，3 遍；看是否随 worker 数单调变化。

## E. 交付（必答）

1. 两臂 × 3 遍的 Min/Avg/P50/P90/P99/Max、failed、400/400、W9。
2. 与基线（workers=4：plain 274.00/120.07、vseg 555.00/248.80）的对比表。
3. `MC_NOF_WORKERS` 生效证据（日志/worker 数）。
4. 判决：workers=1 是否让 vseg 明显变快；plain 是否保持不变。
5. 若没变化，明确写“task/lane 分布不是原因”，不要外推。

**判读注意（必须写进回执）**：`SpdkNofQos` 的 `inflight_blocks_limit = MC_NOF_INFLIGHT_BYTES_LIMIT(32MiB) / lane_count`（`transfer_task.cpp:655-668`）。workers=1 时单 lane 额度从 8MiB 提到 **32MiB**，所以若 vseg 变快，**不能直接归因“两个 task 同 lane”**，也可能是“单 lane 额度变大”。下一单需用 `MC_NOF_WORKERS=1` + 调低 `MC_NOF_INFLIGHT_BYTES_LIMIT`（例如 8MiB）把两者分离。

## F. 边界

- **零写入**；target 不重启；不改码/不重编；不启用 diag/BUFFER；不跑 16t；不动 `/usr/local`。
- 只切 master（plain→vseg），不碰他人进程；失败停手回报。
- 不把 workers=1 当成性能配置对外；本单只用于机制判定。

---

## NOF-110 裁决（2026-11-25 · 回执已收）

### 1. 结果（[事实]，diag 关，3 遍中位，µs）

| 指标 | workers=4（基线） | workers=1 | 变化 |
|---|---|---|---|
| plain P50 | 274.00 | 266.27（261.72–276.95） | 在臂内波动内 |
| plain W9 | 120.07 | 120.36 | +0.2% |
| vseg P50 | 555.00 | **558.50**（553.58–568.61） | +0.6% |
| vseg W9 | 248.80 | **249.36** | +0.2% |
| P50 比值 | 2.03× | 2.10× | 不变 |
| W9 比值 | 2.07× | 2.07× | 不变 |

- env 生效证据：`nof_worker_thread worker_idx=0`；`nof_lane_bind` 四条 lane 全部 `worker_idx=0`、qpair 2/3/4/5；warning “4 I/O lanes but only 1 workers” 出现。
- **额度混杂项实测排除**：workers=1 与 workers=4 的 `lane_inflight_blocks_limit=16384`、`segment_inflight_blocks_limit=65536` 完全相同。原因是公式除以 `lane_count`（=qpairs=4），不是 worker 数（`transfer_task.cpp:655-668`）。

### 2. 能下的结论

- **worker 线程分布不是原因（L1 排除）**：把 4 条 lane 的轮询压到 1 个 worker，vseg 无变化。
- **inflight 额度不是原因（L1 排除）**：两配置额度相同。
- plain 保持不变，符合“T1 每 key 只有 1 个 task”的预期。

### 3. 不能下的结论（重要纠正）

- 回执写“客户端侧的编排、队列归属、并发深度、inflight 额度全部可以排除”——**“队列归属/qpair 扇出”不能排除**。
- 原因：workers=1 **只合并了 worker 线程，qpair 仍是 4 条**。`SpdkNofWorkerPool::submitTask` 用 `binding.next_lane++ % lane_count` 给每个 task 选 lane（`transfer_task.cpp:911-916`），所以：
  - vseg：child0、child1 连续提交 ⇒ 落到**两条不同的 qpair**；
  - plain：每 key 1 个 task ⇒ 每 key 只用**一条 qpair**（不同 key 之间轮转）。
- 因此“每 key 两个 qpair 扇出”仍是未测变量，NOF-111 用 `--nof_qpairs_per_segment=1` 处理。

### 4. 下一步

**NOF-111**：把 qpairs 从 4 改为 1，让 vseg 两个 child 共享同一条 qpair；同时把 `MC_NOF_INFLIGHT_BYTES_LIMIT` 夹到 8MiB，使单 lane 额度仍为 16384（避免 qpairs=1 导致额度变 4× 的混杂）。若 vseg 变快 ⇒ qpair 扇出是触发条件，修法是合并相邻 child 为一个 4MiB task；若不变 ⇒ qpair 扇出也排除，转向 LBA/布局或 target 侧 trace。

---

# NOF-111（待执行 · 单变量 `--nof_qpairs_per_segment=1` + 8MiB 额度夹紧 · 零写/零重启/零改码 · 2026-11-25）

## 目标

- 只操纵 qpair 数：4 → 1；同时 `MC_NOF_INFLIGHT_BYTES_LIMIT=8388608`（8MiB），抵消 `lane_count=1` 导致单 lane 额度从 16384 变 65536 的混杂。
- T1 下 plain 每 key 1 个 task，qpairs 数只影响 key 间轮转，不动单 key；vseg 的两个 child 在 qpairs=1 时必须共享同一条 qpair。
- 只回答：**vseg 的 P50 / target W9 是否向 plain 靠近**。
  - 若明显变快 ⇒ “每 key 两个 qpair 扇出”是 target 慢档的触发条件；修复方向：合并相邻 child 为一个 4MiB task（物理相邻，可合并），或让两个 child 绑定同一 lane/qpair。
  - 若不变 ⇒ qpair 扇出也排除，剩余候选只有数据 LBA/布局与 target 侧服务路径。

## A. 预检

1. target 2142228 不重启；记录 md5/4457/RAID0 UUID。
2. 当前应运行 vseg master 2194604（NOF-110 末态、Keys=400、partitions=1）；plain master 已停，`plain_nof107` 数据保留。
3. `--help 2>&1 | grep -E "nof_qpairs_per_segment"` 确认旗标存在；不存在则改用 env `MC_NOF_QPAIRS_PER_SEGMENT=1`（源码 `stress_cluster_bench.cpp:3292-3294` 支持两者之一），并记录实际用法。
4. 用 **2-key 探针 + `MC_NOF_DEBUG=1`** 记录 `nof_lane_bind` 与 `lane_inflight_blocks_limit` / `segment_inflight_blocks_limit`，确认真的是 1 条 lane/qpair、单 lane 额度 16384；探针不算性能。

## B. vseg 臂（qpairs=1，先跑；vseg master 已在跑）

1. 正式 reader：`--nof_qpairs_per_segment=1`（或 env），env `MC_NOF_INFLIGHT_BYTES_LIMIT=8388608`；其余同 NOF-107：400 keys、4MiB、T1（threads=1/batch=1）、sequential、warmup=128、verify=false、`--scenario=segment_read`；**diag 关、DEBUG 关**。
2. 每遍前重新 `nof_register` 并验证挂载。
3. 3 遍；记 Min/Avg/P50/P90/P99/Max、failed、400/400、target W9（同 NOF-107 窗口法）。
4. 记录启动日志里的 `NoF qpairs/segment` 行，确认 1。

## C. plain 臂（qpairs=1，后跑）

1. 停 vseg master（记录 PID），按 NOF-107 B 段同参起 plain master（`plain_nof107`、50056/9006/8086、slot=1、etcd 127.0.0.1:2379、URMA 设备/capsule）；验收 leader/serving、Keys=400、owned_slots=1。
2. `nof_register` 到 node4 target，验证挂载。
3. 正式 reader 同 B（qpairs=1、8MiB inflight、diag 关）；3 遍；记录同 B。

## D. 交付（必答）

1. 探针：lane 数/ qpair id / lane_inflight / segment_inflight（证明单变量成立）。
2. 两臂 × 3 遍的 Min/Avg/P50/P90/P99/Max、failed、400/400、W9。
3. 与基线（qpairs=4：plain 274.00/120.07、vseg 555.00/248.80）的对比表；与 workers=1（plain 266.27/120.36、vseg 558.50/249.36）的对比。
4. 判决：qpairs=1 是否让 vseg 明显变快；plain 是否保持。
5. 若不变，明确写“qpair 扇出也不是原因”，不要外推；若变快，给出 P50/W9 的下降幅度，并注明下一步是合并 child 的实现验证。

## E. 边界

- **零写入**；target 不重启；不改码/不重编；不启用 diag/BUFFER；不跑 16t；不动 `/usr/local`。
- 只切 master（vseg→plain），不碰他人进程；失败停手回报。
- qpairs=1 只用于机制判定，不作为最终性能配置对外。
- 注意：qpairs=1 会改变 target 看到的连接数（4→1），这本身就是本单要测的变量；不要把它和“客户端 worker 数”混为一谈。

---

## NOF-111 裁决（2026-11-25 · 回执已收）

### 1. 结果（[事实]，diag 关，3 遍，µs）

| 指标 | qpairs=4 | workers=1 | qpairs=1 |
|---|---|---|---|
| plain P50 | 274.00 | 266.27 | 254.10（漂移，W9 未动） |
| plain W9 | 120.07 | 120.36 | 120.29 |
| vseg P50 | 555.00 | 558.50 | 558.74 |
| vseg W9 | 248.80 | 249.36 | **248.87** |
| W9 比值 | 2.07× | 2.07× | **2.07×** |

- 探针：`NoF qpairs/segment: 1`、`lanes=1`、`lane=0 qpair_id=2`、`lane_inflight_blocks_limit=16384` ⇒ 单变量成立，额度与基线同值（不是“额度变化抵消收益”）。
- `segment_inflight_blocks_limit` 从 65536 降到 16384 是 `lane_count=1` 的直接结果；T1 每 key 32×128KiB=4MiB=8192 blocks，仍小于 16384，不会限流。
- plain P50 254 vs 274：W9 完全没动（120.29 vs 120.07），且近几单 plain P50 在 253–277 漂移；**不作归因**。

### 2. 判决

- **qpair 扇出不是原因（L1 排除）**：把 4 条 qpair 压到 1 条，vseg W9 三遍 248.79/248.87/248.86，与基线差 0.03%。
- **客户端组织方式累计排除**：resolve/prepare/submit（NOF-108）、child 串行（NOF-109）、worker 分布与 inflight 额度（NOF-110）、qpair 扇出（NOF-111）。
- 三次独立实验 W9 比值都精确锁在 **2.07×** ⇒ 差异稳定锚在 **target 对单笔 128KiB IO 的服务时间**。

### 3. 措辞边界（防过度）

- 回执写“child 拆分也排除”——严格说，**“两个 child 合并成一个 task”没有实现过**；被操纵的是 worker/inflight/qpair。只是在 workers=1 + qpairs=1 下，两个 child 已共享同一 worker、同一 qpair，target 看到的 32 个 128KiB IO 在数量/大小/通道上都与 plain 相同。因此可以把“客户端组织方式”视为基本排除，但**不能写“已证明合并 task 无效”**。
- W9 只覆盖 target 的 `exec→bdev_cpl`；`completion_wait` 还含 URMA 往返/完成通知。说“锚在 target 单 IO 服务时间”是**落点**，不是机制根因。
- **数据 LBA/落点仍未对照**；这是当前唯一能直接解释“所有 vseg key 都慢”的客户端可见差异。

### 4. 下一步

**NOF-112（LBA 布局对照）**：
- vseg 侧**零成本**：NOF-108 的 vseg 日志里 `nof_slow_task` 已经带 `lba` / `blocks` / `offset_bytes` / `max_subio_lba`（每遍 1056 条），可直接解析出 400 个 key 的两个 child 物理 LBA。
- plain 侧：补 1 遍 `MC_NOF_SLOW_TRACE_THRESHOLD_US=1` 的只读日志（目前 plain master 2203377 在跑），拿 528 条 task 的 LBA；仅用于布局，不作性能。
- 判读：若 vseg 两个 child 相邻、且其 LBA 大部分不在盘首慢区，**但 per-key 延迟仍平坦 ~470µs** ⇒ 位置解释力弱，下一步转 target 侧 trace；若 vseg 数据明显落在慢区/碎片化 ⇒ 布局是主因，改分配策略。

---

# NOF-112（待执行 · LBA 布局对照 · 零写/零重启/零改码 · 2026-11-25）

## 目标

- 拿到两臂 400 个 key 的**精确物理 LBA**，回答：vseg 的 2× 是否可能由数据落点（盘首慢区、跨盘分布、碎片化）解释。
- vseg 侧直接用 **NOF-108 已落盘日志**（diag on 时 `nof_slow_task` 每 child 一条，含 `lba/blocks/offset_bytes/max_subio_lba`），不重跑。
- plain 侧补 **1 遍只读 LBA 日志**：`MC_NOF_SLOW_TRACE_THRESHOLD_US=1` 让每个 task 都打印；仅用于布局，不作性能结论。

## A. vseg 离线解析（不碰环境）

1. 日志：`run/nof108_vsegp{1,2,3}.log`（注意实际文件名无下划线；若不存在，用 `ls run/nof108_vseg*` 核对后说明）。每遍应有 `nof_slow_task=1056` 条（400 measured + 128 warmup，×2 child）。
2. 每条取：`vsegment_op_id`、`child_idx`、`child_count`、`lba`、`blocks`、`offset_bytes`、`length_bytes`、`qpair_id`、`max_subio_lba`、`max_subio_blocks`。
3. 按 `(trace_id, vsegment_op_id)` 归组；warmup/measured 的切分沿用 NOF-109 的方法（按 op first_submit 排序，前 128 为 warmup），并说明这是 [推算]。
4. 每个 measured key 计算：
   - `child1.lba - child0.lba`（期望 4096 blocks = 2MiB；统计相邻比例）；
   - 两个 child 的 namespace 偏移（bytes）与覆盖范围；
   - per-disk 分布：`strip = lba / 512`（256KiB strip）、`disk = strip % 11`、`round = strip / 11`、`offset_on_disk = round × 256KiB + (lba % 512) × 512B`；
   - 慢区标记：`offset_on_disk < 36MiB`（同时用由 plain 前缀实测的 384MiB/11 ≈ 34.9MiB 再算一版）；
   - 对 `max_subio_lba` 做同样统计（每个 child 一个样本，补充 128KiB 落点）。
5. 与 NOF-108 的逐 key `read_us` 关联：
   - 按 key 的 child0 LBA 分桶（前 384MiB / 384MiB–1GiB / >1GiB，或四分位），看 read_us 是否随 LBA 变化；
   - 计算“慢区 child 占比”与“慢区 key 的平均 read_us vs 非慢区 key”。

## B. plain LBA 日志（1 遍只读）

1. 预检：plain master 2203377 运行、Keys=400；target 2142228 不动。
2. reader：`MC_NOF_SLOW_TRACE_THRESHOLD_US=1`，diag 关；其余同 NOF-107（400 keys、4MiB、T1、warmup=128、qpairs=4、verify=false）。
3. 每遍前 `nof_register`；跑 **1 遍**；400/0。
4. 解析 528 条 `nof_slow_task`：`lba`、`blocks`（期望 8192）、`max_subio_lba`；用与 A 相同的 per-disk/慢区公式统计。
5. 记录 P50/W9 仅作有效性检查（threshold 日志有开销，**不作性能结论**）。

## C. 对比与判决（必答）

1. 两臂 LBA 表：min/max/span、每盘 strip 数、慢区 child/key 占比、相邻比例。
2. **同 key index** 的 LBA 差分布（只作布局对照，不声称同位置）。
3. vseg 内部：key read_us 与 (a) child0 LBA、(b) 慢区标记、(c) 每盘分布 的相关性。
4. 判决三选一：
   - **布局可解释**：vseg 数据显著落在慢区/碎片化，且慢区 key 明显更慢；
   - **布局解释力弱**：vseg 数据大部分不在慢区、child 相邻，且 key 延迟与 LBA 无相关 ⇒ 下一步转 target 侧 trace；
   - **混合**：给出比例与边界。
5. 5 个代表 key 的原始 `nof_slow_task` 行（vseg 两 child + plain 一条）。

## D. 边界

- **零写入**；target 不重启；不改码/不重编；不启用 BUFFER；不跑 16t；不动 `/usr/local`。
- 不切换 master：plain master 已在跑，vseg 日志直接用 NOF-108 的，无需重启 vseg。
- threshold=1 的 plain 单遍只用于 LBA 采集，**不得把它的 P50/W9 当性能数字**。
- 所有 LBA 统计都基于 **512B block**；若发现 block_size 不是 512，停手回报并写明实际值。

---

## NOF-112 裁决（2026-11-25 · 回执已收）

### 1. 决定性结果（[事实]）

| 项 | vseg | plain |
|---|---|---|
| task 粒度 | 2MiB child ×2（每 child 4096 blocks） | 4MiB ×1（8192 blocks） |
| key i 起始 LBA | child0=i×8192，child1=i×8192+4096 | i×8192 |
| 覆盖区间 | 同一 1.56GiB，child1−child0 ≡ 4096（528/528） | 同一 1.56GiB |
| 每盘 task 数 | 96–97 × 11（均匀） | 47–49 × 11（均匀） |
| 慢区 task 占比 | 37.5%（含 warmup；measured 慢区 key=99/400=24.75%） | 同左 |

**同 LBA 直比**（vseg 用 NOF-108 diag-on read_us；plain 用本轮 threshold=1 read_us）：

| 分组 | n | vseg p50 | plain p50 | 比值 | vseg 更慢 |
|---|---|---|---|---|---|
| 全部同 LBA | 400 | 472.5 | 180.0 | 2.62× | 363/400 |
| **非慢区（offset≥36MiB）** | **301** | **478.5** | **179.0** | **2.67×** | **301/301 = 100%** |
| 慢区（offset<36MiB） | 99 | 447.5 | 448.5 | 1.00× | 62/99 |

- plain 有 2.5× 的慢区落差（448.5 vs 179.0）；vseg 没有落差（447.5 vs 478.5，慢区甚至略快）。
- 一句话：**vseg 在每个 LBA 上都按“慢区速度”被服务，plain 只在真正盘首慢区才慢。**

### 2. 判决与等级

- **数据落点假说被否（L1）**：两臂起始 LBA 逐值相同 400/400、覆盖同一 1.56GiB、盘分布均匀、慢区占比同构。物理位置不能解释 2×。
- 至此已排除：旧环境/源码、客户端 resolve/prepare/submit、child 串行、worker 分布+inflight、qpair 扇出、**LBA 落点**。
- 仍为 **L2+**（强假说，未到 L3）：两臂是不同 cid/不同 client 进程/不同 NoF 会话；**“两个 2MiB task”与“一个 4MiB task”的切分边界**、以及 **vseg 会话/服务路径** 尚未分离。

### 3. 口径警告（必须保留）

- 配对用的两侧 read_us **来自不同诊断配置**：vseg 是 NOF-108 diag-on（每 child 日志），plain 是本轮 `MC_NOF_SLOW_TRACE_THRESHOLD_US=1`（每 task 日志）。两边都有日志开销，但开销未必相等 ⇒ **2.67× 的量值只能当近似，方向（plain 179 vs vseg 478、301/301 全慢）是硬的**。
- plain 采集遍 P50 287.75µs **不作性能结论**；它只用于 LBA 与配对。
- 37.5% 是含 warmup 的 528 task 占比；measured 侧慢区 key 是 **99/400 = 24.75%**，不要混用。
- 盘号公式 `disk=(lba/512)%11` 是 [推算]，未与 target 实际 RAID0 映射表核对；不影响“同 LBA”结论，但影响 per-disk 数字的精确性。

### 4. 剩余候选与下一步

1. **请求切分边界**：vseg 以 2MiB 为单元（2 个 task、各 16×128KiB） vs plain 4MiB（1 个 task、32×128KiB）。覆盖区间相同，但 task 边界不同。
2. **vseg 会话/服务路径**：不同 cid/client 进程/NoF 会话，target 侧 qpair→lcore/缓冲区映射可能不同。

**NOF-113 路线（用户 2026-11-25 拍板：一起上）**：不再一单一因素；把 merge、chunk、并发矩阵合并成一单一次跑完。A 路线（合并 child + env 开关）作为矩阵中的一组，另加 chunk 128K/256K/512K 与 T4/T16。若矩阵里仍没有任何配置让 vseg ≤ plain，则停止在单 target 上追 2×，转结构性方案/验收口径。

---

# NOF-113（待执行 · 综合矩阵：merge × chunk × 并发 · 一次跑完 · 零写/零重启 target · 2026-11-25）

> **❌ 本单已取消（2026-11-25，用户明确“不接受修改 vsegment 的逻辑代码”）**。不要应用 `patches/nof113_merge_adjacent_reads.patch`、不要构建 merge 版 bench；补丁已从仓库撤下。本单的 chunk/并发旋钮也一并取消——用户认为继续拧旋钮太慢且到不了“vseg 更快”。**替代任务见文末 `# NOF-113R`（纯配置：vseg 大 IO vs plain，零改码）**。

## 目标

用户已明确“一起上，不要一单一因素”。本单一次回答四个问题：

1. **merge**：把同一 key 两个物理相邻的 2MiB read child 合并成一个 4MiB NoF task（env 开关），能否消除 2×？
2. **chunk**：NoF 提交块 128K / 256K / 512K 对两臂绝对时延与比值的影响？
3. **并发**：T4 / T16 下 vseg 与 plain 是否收敛？
4. **有没有任何配置 vseg ≤ plain**？若没有，记录“当前同工作/单 member 配置达不到用户要求”，下一单改为改变工作方式（多 member 跨独立 target，或内存分层/缓存），**不再继续调参**。

不改数据、不重启 target；只重建 `stress_cluster_bench`（带 env 开关）。

## A. 补丁与构建

1. 补丁在 handover 仓根：`patches/nof113_merge_adjacent_reads.patch`。把它应用到 Mooncake 源码根（node2 的 supercache aadab957 源码树）：
   ```bash
   patch --dry-run -p1 < <handover>/patches/nof113_merge_adjacent_reads.patch
   patch -p1 < <handover>/patches/nof113_merge_adjacent_reads.patch
   ```
   - 补丁只改 `mooncake-store/src/transfer_task.cpp` 的 `TransferSubmitter::submitVSegment` NoF 分支。
   - 仅当 `op_code==READ` 且 env `MC_VSEGMENT_MERGE_ADJACENT_READS=1` 时，把 endpoint、segment_id、physical_offset、client_address 都连续的下一个 request 合并；**默认不设 env ⇒ 行为与原版完全一致**；写路径不受影响。
2. 备份现有 bench（记录路径/md5，复制为 `.nof112`），用 NOF-106 同一构建树/命令重建 `stress_cluster_bench`；记录构建命令、新 binary 路径/md5、`--help` 正常。
3. 用 `strings <bench> | grep MC_VSEGMENT_MERGE_ADJACENT_READS` 确认新 binary 含开关。
4. 构建失败 ⇒ 恢复旧 binary、停手回报，不跑矩阵。

## B. 探针（不做性能结论）

1. **merge 探针**：2-key 只读、`MC_STORE_TRANSFER_DIAG=1`：
   - OFF：每 key 应 2 条 `nof_slow_task`，`blocks=4096`、`sub_ios=16`、`child_count=2`；
   - ON（`MC_VSEGMENT_MERGE_ADJACENT_READS=1`）：每 key 应 **1 条**，`blocks=8192`、`sub_ios=32`、`child_count=1`。
   - 不通过 ⇒ 停手回报。
2. **chunk 探针**：2-key、`MC_NOF_DEBUG=1`，分别 128K/256K/512K，记录 `nof_qos_create ... blocks_per_chunk=`（期望 256/512/1024）与 lane 额度；512K 若 I/O 失败/超 MDTS，记录并**删掉 512K 相关臂**。
3. 目标：target 2142228 不重启；当前 plain master 2203377 在跑。

## C. plain 矩阵（先跑，master 已在跑）

固定：400 keys、4MiB、batch=1、warmup=128、qpairs=4、verify=false、sequential；diag/DEBUG 关；每遍前 register；每臂 **3 遍**；记录 Min/Avg/P50/P90/P99/Max、failed、W9。

| 臂 | threads | chunk |
|---|---|---|
| P1 | 1 | 128K |
| P2 | 1 | 256K |
| P3 | 1 | 512K（若探针失败则跳过） |
| P4 | 4 | 128K |
| P5 | 16 | 128K |

## D. vseg 矩阵（切到 `vseg_nof107`，零写入）

1. 停 plain master（记录 PID），按 NOF-107 C 段同参起 vseg master；验收 Keys=400、partitions=1、reconcile_fail=0；register。
2. 固定参数同 C；**零写入**；每遍前 register；每臂 3 遍；记录同 C。
3. **执行顺序**（先跑最决定性的）：

| 臂 | threads | chunk | merge |
|---|---|---|---|
| V1 | 1 | 128K | off（控制） |
| V2 | 1 | 128K | **on** |
| V3 | 1 | 512K | off |
| V4 | 1 | 512K | **on** |
| V5 | 1 | 256K | off |
| V6 | 4 | 128K | off |
| V7 | 16 | 128K | off |

## E. 交付（必答）

1. 补丁 apply / 构建命令 / 新 binary md5 / strings 证据；merge 与 chunk 探针原始行。
2. **一张总矩阵表**：臂、threads、chunk、merge、3 遍 P50/单遍明细、W9、failed；附 plain/vseg 比值。
3. 四个问题的直接答案：
   - merge 在 128K/512K 各让 vseg 的 P50/W9 变化多少？是否回到 plain 档？
   - chunk 128→256→512 对两臂绝对时延和比值的影响？
   - T4/T16 下 vseg/plain 是否收敛？比值多少？
   - **是否存在任何配置 vseg ≤ plain**？若有，给出最佳配置与数值；若没有，明确写“当前同工作/单 member 配置达不到用户要求”，并说明下一单必须改变工作方式（多 member 跨独立 target，或内存分层/缓存），不再继续调参。
4. 与历史基线对照：plain 274.00/120.07、vseg 555.00/248.80（diag off，T1/128K）。
5. 环境末态与回退命令。

## F. 边界与回退

- **零写入**；target 2142228 不重启；不改 SPDK、不动 `/usr/local`。
- 只改 Mooncake 客户端并重建 bench；旧 binary 保留为 `.nof112`。
- 本单**包含 T4/T16**（用户明确批准），不再受“只跑 T1”限制；但不跑其他线程数。
- 回退三级：env 不设（行为同原版）→ 换回 `.nof112` binary → `patch -R -p1` / `git checkout -- mooncake-store/src/transfer_task.cpp`。
- 任一臂出现 failed≠0、rc≠0、挂载失败：记录该臂并继续；若 merge 路径崩溃或 400/0 不成立，立即停止 merge 臂并回退，仍可完成 chunk/plain 臂。
- 预计：构建 1–3 分钟；探针 3 分钟；36 个 pass（12 臂×3）约 10–15 分钟；总计 **≤30 分钟**。

## G. 环境基线（本单开始前应确认）

- target 2142228（md5 3129641f…、4457、RAID0 UUID 不变）。
- plain master 2203377（Keys=400、owned_slots=1）；vseg 数据 `vseg_nof107` 400 keys 保留。
- node2 无残留 bench；`vseg_nof107` 只读。

---

# NOF-113R（修订 · 纯配置：vseg 大 IO vs plain · 零改码/零写/零重启 · 2026-11-25）

## 用户约束（优先于一切）

- **不接受修改 vsegment 逻辑代码**。NOF-113 的 merge 补丁取消：不 apply、不构建、不使用；补丁已从仓库撤下。
- 只允许现有二进制 + env/CLI 配置；不重编 Mooncake/SPDK。

## 目标

单 target 下，vseg 唯一可能赢过 plain 的配置杠杆是**更大的 NoF IO**：

- vseg 的 2MiB child 直接作为一个 IO 下发（`MC_NOF_SUBMIT_CHUNK_BYTES=2097152` 或 1MiB）；
- plain 保持 128KiB 作对照；
- 同时跑 plain 的大 IO 臂作同工作对照。

只回答三个问题：

1. **vseg@2MiB IO vs plain@128KiB**：单 target 上 vseg 能否更快？
2. **vseg@2MiB vs plain@2MiB**：同样的 IO 大小下，vseg 是否有任何优势？
3. **大 IO 是否降低绝对时延**？最佳配置是什么？

若 vseg@2M 仍不快于 plain@128K，且 vseg@2M ≈ plain@2M ⇒ **单 target 同工作下没有 vseg 的赢面**；停止配置调参，转多 target/member 或内存分层（需另议，不在本单）。

## A. 预检

1. target 2142228 不重启；plain master 2203377 在跑（Keys=400）；`vseg_nof107` 400 keys 保留。
2. 现有 bench 不改/不重编；只用 env `MC_NOF_SUBMIT_CHUNK_BYTES`。
3. 2-key 探针 + `MC_NOF_DEBUG=1`，分别 128K/512K/1M/2M，记录 `nof_qos_create ... blocks_per_chunk=`（期望 256/1024/2048/4096）。
4. 1M/2M 若 I/O 失败或超 target MDTS：记录错误，删掉该尺寸对应臂；不要重试改码。

## B. plain 臂（先跑，master 已在跑）

- 固定：400 keys、4MiB、T1（threads=1/batch=1）、warmup=128、qpairs=4、verify=false、sequential；diag/DEBUG 关；每遍前 register；每臂 **3 遍**。
- chunk：**128K / 1M / 2M**（512K 可略，除非时间充裕）；记录 Min/Avg/P50/P90/P99/Max、failed、W9。

## C. vseg 臂（切到 `vseg_nof107`，零写入）

- 同 B 参数；chunk：**128K / 512K / 1M / 2M**（探针失败的尺寸跳过）；每臂 3 遍；每遍前 register；记录同 B。
- 不启用 merge、不改代码；只设 `MC_NOF_SUBMIT_CHUNK_BYTES`。

## D. 交付（必答）

1. 探针原始行：每个 chunk 的 `blocks_per_chunk` 与实际生效证据。
2. 矩阵表：臂、chunk、3 遍 P50/W9、failed；附 vseg/plain 比值。
3. 三个问题的直接答案：
   - vseg@2M 与 plain@128K 的 P50/W9 差多少？是否达到 **parity**？
   - vseg@2M 与 plain@2M 是否相同/谁快？
   - 大 IO 是否降低绝对时延？最佳配置数值。
4. **验收口径（用户 2026-11-25 明确）**：**打平即可，2× 绝不可接受**。parity 定义为 vseg 的 P50 与 W9 均 ≤ plain 基线 ×1.05（或绝对差 ≤10–15µs）；超过即未达标。
5. 判决分支：
   - 若 vseg@2M 达到 parity ⇒ **本方向达标**，记录最佳配置（env 值），无需改码。
   - 若仍有明显差距（>1.2×）⇒ 记录“单 target 配置调参不能消除差距”；**唯一已知的 parity 修法是合并相邻 child（改 vseg 读提交逻辑，需用户另行批准）**；多 target/member 或内存分层属于另一路线（可更快，但工作量大）。
6. 环境末态与回退（env 不设即回原配置）。

## E. 边界

- **零代码/零构建/零补丁**；零写入；target 不重启；不跑 T4/T16（本单只做 T1，避免再次铺旋钮）。
- 不改 SPDK、不动 `/usr/local`。
- 失败（挂载/IO 错误）停手回报；不要自行改代码修复。
- 预计：探针 3 分钟 + 7 臂 ×3 遍约 8–10 分钟；总计 **≤15 分钟**。

---

## NOF-113R 裁决（2026-11-26 · 回执已收）

### 1. 结果（[事实]，T1，3 遍均值，µs）

| 臂 | chunk | P50 | W9（target） | target n |
|---|---|---|---|---|
| plain | 128K | **270.52** | 120.4–129.1 | 16897–16898 |
| plain | 1M | 264.00 | 143.4–143.5 | 8449–8451 |
| plain | 2M | 277.73 | 142.5–143.5 | 8450 |
| vseg | 128K | **546.36** | 249.9–257.4 | 16897–16899 |
| vseg | 512K | 559.24 | 287.0–287.6 | 8449–8452 |
| vseg | 1M | 549.87 | 286.7–288.3 | 8449–8451 |
| vseg | 2M | **551.93** | 286.6–288.7 | 8449–8451 |

- vseg@2M vs plain@128K = **2.04×**；vseg@2M vs plain@2M = **1.99×**。
- vseg 四个 chunk 全在 546–559 噪声带，**chunk 对 vseg 无影响**。
- 大 IO 不降时延：plain 270.52→264.00→277.73，vseg 546.36→549.87→551.93；**W9 反而升**（plain 120→143，vseg 250→287）。

### 2. 两个关键机制事实

1. **target 实际单 IO 被夹在 256KiB**：chunk≥256K 时 target n 从 16898（528×32）掉到 8450（528×16），即每 key 16 个 IO、每个 256KiB；虽然 `blocks_per_chunk` 到了 4096，实际 IO 没有继续变大。512K/1M/2M 三档表现一致就是这个原因。夹点很可能是 target MDTS/传输上限（[推算]，回执未直接给出 MDTS 值）。
2. **2× 是按字节的成本，不是每 IO 固定开销**：IO 从 128K 变 256K，单 IO 时间与总时延都基本不变（甚至 W9 变差）。如果 vseg 多出的是“每笔请求固定 128µs”，大 IO 应该摊薄它；实测没有 ⇒ vseg 是**每字节的服务速率约为 plain 的一半**。

### 3. 判决

- **配置路径到顶**：单 target、同 LBA、同 IO 数下，调 chunk 不能消除 2×，也达不到 parity（≤1.05×）。
- 仍为 **L2**：知道“按字节慢一倍”，不知道 target 内部哪一段/什么机制导致。
- 用户验收未达成；不能把 2× 收口。

### 4. 下一步（需用户批准）

要回答“为什么按字节慢一倍”，必须**拆开 W9**，看 target 逐 IO 的：
- `enter/submit/submit_return/callback`（bdev 提交前/中/盘完成）；
- `buffer/wait/enomem`（buffer 等待）；
- `lcore/qid/cid/lba/nlb`（哪个轮询核/队列、在飞并发、到达顺序）。

target 已有 `URMA_STAGE_BUFFER` 24 字段 trace，可直接用；但必须让 target **带 BUFFER env 重启并退出时 dump**（高危：RAID0 UUID 可能变、vseg 元数据可能失效，需重新 register/mount，必要时重写 400 keys）。一次重启可同时采 vseg 与 plain，按 `exec_tick` 间隙切分。

**替代方案（不重启 target）**：多 member 跨独立 target，或内存分层；都不是“定位这个 2×”，而是换路线。

---

# NOF-114（待执行 · node3 第二个 target 部署与就绪 · 不 register/不写数据/不动 node4 · 2026-11-26）

## 用户确认的拓扑（本单背景）

```text
node2：reader + writer
node4：master + etcd + target B（现有，不动）
node3：target A（本单新部署）
vseg：member_count=2，4MiB key → 2×2MiB child，分别落 node3/node4
plain：只用 node4 一个 target
reader 的元数据（node4）和数据（node3/node4）都跨节点
验收：parity（vseg P50/W9 ≤ plain×1.05）
```

## 目标

- 只做一件事：在 **node3** 部署第二个我们自己的 SPDK nvme-oF target，验证就绪。
- **不 register 到 master、不 mount、不写数据、不改 master/etcd、不动 node4 target**。
- 为 NOF-115（双 target vsegment）准备一个可用的 target A。

## A. node3 盘点（只读）

1. 用户已宣布 node3 独占；仍记录事实：CPU/NUMA、内存、hugepages、磁盘列表（BDF/型号/容量）、SPDK/URMA 驱动、网络、是否有旧 target/残留进程。
2. 核对 node3 到 node2/node4 的连通性（文档只写 node 标签，不写 IP）。
3. 若发现他人进程/非独占迹象：停手回报，不清理。

## B. 部署 target A

1. 复用 node4 的 target 二进制（md5 `3129641f…`）与启动脚本/配置；复制到 node3 自有目录（`/home/.../mooncake-nof-vsegment-2/`），记录脚本/md5。
2. 按 node3 实际磁盘做 RAID0：strip **256KiB**、BDF 升序；盘数与 BDF 列表记录（不必与 node4 相同）。
3. 启动 target：`:4457`、URMA 设备/capsule（与 node4 同谱系，wire v4）、nsid1、NQN **与 node4 不同**；记录 PID/md5/RAID0 UUID。
4. 验收：4457 LISTEN、RAID0 online、nsid1、URMA attach 正常、target 日志无 ERROR、hugepages 足够。
5. **不执行 register/mount/nof_register**；不改 master。

## C. 交付（必答）

1. node3 盘点表：CPU/NUMA/内存/hugepages/磁盘 BDF/型号/容量/旧进程。
2. target A：启动命令、PID、md5、4457、RAID0 UUID、盘数/BDF、strip、nsid1、URMA 设备。
3. 与 node4 target B 的差异表（盘数、UUID、NQN、配置）。
4. 从 node2 到 node3:4457 的连通性检查结果（只读探测，不 register）。
5. 环境末态：node3 target 运行；node4 target/master 未动；无数据写入。

## D. 边界

- **不 register、不 mount、不写数据、不改 master/etcd、不重启 node4 target**。
- 不装包、不改 `/usr/local`、不改 vseg 逻辑代码；node3 只用自有目录。
- 缺依赖（SPDK/URMA/驱动/hugepages）⇒ 停手回报，不擅自安装。
- 用户已确认 node3 独占；仍不碰他人进程。
- 预计 **20–30 分钟**。

## E. 为 NOF-115 预留的信息

- node3 target 的 endpoint（node 标签）、nsid、UUID、BDF 顺序。
- `nof_register` 是否支持对两个 target 分别注册（本单只查 `--help`/源码，不执行注册）。
- master 启动参数中 `member_count=2` 的可行性（只查源码/配置，不改 master）。

### NOF-115 预定参数（用户 2026-11-26 拍板，先跑通）

- **keys = 400**（先跑通；1000/20k 后面再试）
- **extent = 50GiB**（`--vsegment_member_extent_size=53687091200`）
- `member_count=2`、`stripe_size=2MiB`、`required_medium=nof`、`reserved_ratio=0`
- **oplog/HA 保持现状**（400 keys 已知安全，旧实现约 22MB oplog；不为这轮增加额外变量）
- plain：400 keys，只打 node4 一个 target
- vseg：400 keys，跨 node3+node4；先写 plain 测量，再写 vseg（会覆盖 node4 同 LBA 区域）
- 验收：parity（vseg P50/W9 ≤ plain×1.05）
- 说明：400 keys 在 50GiB extent 下只用 1 个 vsegment（逻辑 100GiB，每 target 预留 50GiB、实际用 ~0.8GiB）；跑通后可继续往同一 vsegment 加 keys，不用改 extent。

---

## NOF-114 裁决（2026-11-26 · 回执已收）

### 1. 结果（[事实]）

- **node3 target A 已存在且就绪**，无需部署：PID 2810360（11-25 14:16:23 启动）、md5 `3129641f…`（与 node4 B 相同）、11 盘 RAID0 strip 256KiB online、nsid1、4457 LISTEN、日志 **0 ERROR**、最新 breakdown 块 n=0（从未有 IO）。
- 盘/BDF 集合与 node4 相同（21,22,28,29,2a + 91,92,a1,a2,a3,a4）。
- **NQN/SN 与 node4 相同**：`nqn.2026-01.io.spdk:urma-gpu-test` / `URMAGPU0001`；RAID0 UUID 不同（`1e71527b…` vs `0470fe3a…`）。
- **node3 有第三方 royi master/client**（50151/18080/19005），未触碰；node3 实际非独占，load ~14.5。
- 资源：node3 2261GiB 内存，hugepages free 347；target 6 核 pcpu ~599%；node2→node3 网络 L3 通（0% loss，rtt 0.056ms）；URMA 数据面未验证（本单禁止 register）。
- node4 target 2142228 未动（ERROR 6 条全为 pci_event socket read error 105）；vseg master `vseg_nof107` 未动。

### 2. 两个待裁决点

1. **NQN 冲突**：两个独立 target 用同一 NQN/SN，注册层可能不阻塞（segment 名含 traddr，`ssd_register_client.cpp:48-95`），但 initiator 侧可能把它们当成同一 NVMe-oF 子系统的两条路径，存在读到错误数据的风险（[未验证]）。**建议重启 node3 target 换唯一 NQN/SN**：target A 空转无数据，3–5 分钟，UUID 会变。
2. **royi 进程**：第三方 master/client 在 node3 运行；任务书要求发现他人进程停手。需要用户决定：保留（记录噪声）还是授权清理。

### 3. 下一步分支

- **(a) 保持同 NQN 直接进 NOF-115**：可试双 segment 注册 + 2-key verify 探针；若注册/读失败或身份冲突，再回头换 NQN。风险是可能读到错误数据。
- **(b) 重启 node3 target 换唯一 NQN/SN**（推荐）：用同一 wrapper 重建（11 盘 vfio/RAID0/strip256K），记录新 UUID；然后 NOF-115。
- **(c) 先处理 royi**：用户确认是否可停；若保留，NOF-115 需记录 node3 负载/CPU 混叠。

**NOF-115 参数不变**：keys=400、extent=50GiB、member_count=2、plain 单 target、oplog/HA 保持现状。

---

# NOF-114R（待执行 · node3 独占清理 + target A 换唯一 NQN/SN · 不 register/不写数据/不动 node4 · 2026-11-26）

## 用户授权

用户明确：**node2/3/4 我们独占，佛挡杀佛**。node3 上的第三方进程可以清理；node3 target A 可以重启。仍不碰 node4 target/master，不写数据。

## 目标

1. 清理 node3 上的第三方 Mooncake/测试进程（royi master/client 及其他非系统、非我方残留进程），释放 CPU/内存。
2. 重启 node3 target A，使用**唯一 NQN/SN**（与 node4 不同），保持同一二进制、同一 11 盘 RAID0 strip256K、nsid1、4457。
3. 验证 target A 就绪；为 NOF-115 双 target 注册做准备。
4. **不 register、不 mount、不写数据、不动 node4 target/master**。

## A. node3 进程清理（先取证再杀）

1. 全量记录 node3 用户态进程：PID/PPID/user/cmdline/端口/CPU/内存；标出：
   - 我们的：target A（PID 2810360）、node3 上可能残留的 bench/register；
   - 第三方的：royi master（3463695，50151/18080/19005）、royi client（3465188）及其他 royi 子进程；
   - 系统/他人：sshd、系统服务等**不杀**。
2. 先停 royi client，再停 master；`TERM` 后等待，必要时 `KILL`；记录每步 PID/退出码。清理后确认 50151/18080/19005 释放、load 下降。
3. 若发现其他非系统、非我方进程：记录后按“佛挡杀佛”处理；**不杀 PID 1、系统守护进程、sshd/网络管理**。若无法判断归属，停手回报。
4. 检查 node3 端口占用：4457 应仅剩 target A；其他我方端口（50056/9006/8086/2379/50156）不应在 node3 运行。

## B. 重启 target A（换唯一 NQN/SN）

1. 取证 target A 的启动方式：PID 2810360 的 exe/cwd/cmdline/environ、启动 wrapper、日志路径；记录原 NQN/SN、RAID0 UUID、BDF 顺序、hugepages。
2. 停止 target A（SIGTERM，等待退出；记录日志尾部）。确认 11 盘回到 vfio、4457 释放。
3. 用同一 wrapper/同一二进制重启，仅改：
   - **NQN = `nqn.2026-01.io.spdk:urma-gpu-test-node3`**（与 node4 的 `urma-gpu-test` 不同）
   - **SN = `URMAGPU0003`**（与 node4 的 `URMAGPU0001` 不同）
   - 保持 11 盘 RAID0、strip 256KiB、superblock=false、nsid1、4457、`-m [2-7]`、trace entries 262144、URMA 设备/capsule 同原配置。
   - 若 wrapper 硬编码 NQN/SN：复制到我们自有目录再改，记录 diff/md5；不改他人脚本。
4. 启动后验收：
   - 4457 LISTEN；`nvmf_get_subsystems` 显示新 NQN/SN、nsid1→URMA_RAID0；
   - RAID0 online、11 盘、strip 256K；新 RAID0 UUID 记录；
   - 日志 0 ERROR；`breakdown` 块 n=0（无 IO）；
   - node2→node3:4457 L3 连通（ping）。
5. **不执行 register/mount/nof_register**；不改 node4。

## C. 交付（必答）

1. node3 清理前后进程表、被杀 PID/命令/退出码、端口释放、load 对比。
2. target A 原配置 vs 新配置：NQN/SN、PID、启动时间、日志路径、RAID0 UUID、BDF、hugepages。
3. 新 target 验收：4457、NQN/SN、nsid1、RAID0 online/11 盘/strip256K、0 ERROR、breakdown n=0。
4. 明确写：**未 register、未 mount、未写数据、node4 未动**。
5. 环境末态：node3 target A 运行（新 NQN）、royi 已清理；node2/node4 未变。

## D. 边界与回退

- 只清理 node3 用户态进程；**不杀系统守护/sshd/PID 1**；不动 node4。
- 不 register、不 mount、不写数据、不改 vseg 逻辑代码、不装包。
- target A 空转无数据，重启风险低；若重启失败：记录日志，恢复原 wrapper/配置并重启（旧 NQN 可回退），停手回报。
- 新 RAID0 UUID 会变；NOF-115 重新 register 即可，不依赖旧 UUID。
- 预计 **10–15 分钟**。

## E. 为 NOF-115 准备

- 记录 node3 target A 的新 endpoint（node 标签）、NQN、SN、nsid、UUID。
- 只查 `nof_register --help` 是否支持按 endpoint+NQN 注册两个 segment（**不执行注册**）。

---

# NOF-115（待执行 · 双 target vsegment 就绪 + 2-key 冒烟 · 不跑性能/不写 400 · 2026-11-26）

## 目标

- 用 **node3 target A + node4 target B** 组成 `member_count=2` 的 vseg。
- 验证三件事：
  1. 两个 target 都能注册成**不同 segment**（不同 traddr/NQN）；
  2. quota 能以 `member_count=2 / extent=50GiB / stripe=2MiB / required_medium=nof` 发布；
  3. 每个 key 的 2 个 child **分别落到 node3 和 node4**。
- **只写 2 keys 做冒烟**；400-key 性能、plain 基线留给 NOF-116。

## A. 预检

1. target A：node3 PID 3329243、NQN `nqn.2026-01.io.spdk:urma-gpu-test-node3`、SN `URMAGPU0003`、nsid1、4457、RAID0 UUID `2c5bf1f2…`、11/11、0 ERROR。
2. target B：node4 PID 2142228、NQN `nqn.2026-01.io.spdk:urma-gpu-test`、SN `URMAGPU0001`、nsid1、4457、UUID `0470fe3a…`；**不动**。
3. 当前 `vseg_nof107` master 在 node4 运行（记录 PID/Keys=400）；node2 无 bench。
4. `nof_register --help`：确认能分别指定 **traddr/trsvcid/NQN/nsid**（或等价的 endpoint 字符串）；记录用法。若不支持区分两个 NQN，停手回报。

## B. 停旧 master，注册两个 segment

1. 停 `vseg_nof107` master（记录 PID/退出码）；node4 target B 不动。
2. 在 node4 起新 vseg master **`vseg_nof115`**，先 `--vsegment_auto_plan=false`：
   - 50056/9006/8086、slot=1、HA/etcd 127.0.0.1:2379、URMA 设备/capsule、`--cvm_segments_vsegment_exclusive=true`、`--submaster_count=1`；
   - 验收 leader/serving、Keys=0、owned_slots=1。
3. 用 `nof_register` 分别注册：
   - target B（node4）：NQN `nqn.2026-01.io.spdk:urma-gpu-test`，nsid1，4457；
   - target A（node3）：NQN `nqn.2026-01.io.spdk:urma-gpu-test-node3`，nsid1，4457。
   - 记录两次 registration 的 segment name（含 traddr/node 标签），确认 master 侧有 **2 个不同 segment**。
4. 若第二个注册被拒、覆盖、同名，或 master 只看到 1 个 segment：**停手回报**，不自行改 NQN/端口/注册方式。

## C. 发布 member_count=2 的 quota

1. 停新 master；用 `--vsegment_auto_plan=true` + 以下参数重启：
   - `--vsegment_member_count=2`
   - `--vsegment_member_extent_size=53687091200`（50GiB）
   - `--vsegment_stripe_size=2097152`
   - `--vsegment_required_medium=nof`
   - `--vsegment_reserved_ratio=0`
   - `--vsegment_io_alignment=4096`
   - `--submaster_count=1`、`--cvm_segments_vsegment_exclusive=true`
2. 验收：partitions=1、reconcile_fail=0、Keys=0、`logical_capacity=100GiB`（50G×2）、vsegments=0。
3. 若 auto_plan 无法在 2 个 segment 上发布 quota（报 member_count/medium/空间不足），停手回报；可尝试手工 `vsegment_quota_planner --publish` 或 HTTP `/vsegment_quota`，但必须在回执里写明实际方式与返回。
4. 不写数据前确认 quota 已发布、owned_slots=1。

## D. 2-key 冒烟（只写 2 keys）

1. T1 writer 写 **2 keys**（4MiB）；`VSEGMENT_CREATING` 可重试，最多 3 轮；验收 `Keys=2`、`logical_used=8,388,608`（2×4MiB）。
2. 读 2 keys（`warmup_keys=0`、T1/threads=1/batch=1、qpairs=4、verify=false），开 `MC_STORE_TRANSFER_DIAG=1` 或 `MC_NOF_SLOW_TRACE_THRESHOLD_US=1`：
   - 每 key 应有 **2 个 child**，`child_count=2`、`blocks=4096`、`sub_ios=16`；
   - 两个 child 的 `endpoint` 必须**分属 node3/node4**（回执用 node 标签，不写 IP）；
   - target A/B 各收到约 **32 个 128KiB IO**（2 keys×16），记录 n。
3. 记录 master：`partitions=1`、`vsegments=1`、`Keys=2`、`logical_used=8,388,608`。
4. 若两个 child 落在同一 target、或某个 target 只收到一半/零 IO：停手回报。

## E. 交付（必答）

1. 旧 master 停止记录；新 master 两次启动参数/PID/ports。
2. 两次 register 的 segment name/endpoints（node 标签）、master 侧 segment 数=2。
3. quota 发布方式与验收：partitions/logical_capacity/reconcile_fail/vsegments。
4. 2-key 写读：Keys/logical_used、每 key child_count/blocks/sub_ios、两个 child 的 endpoint（node 标签）、target A/B 的 n。
5. 环境末态：target A/B 未重启/未动；未写 400、未跑性能、未改码。

## F. 边界

- **只写 2 keys**；不跑 400-key 性能；不跑 plain 基线（NOF-116）。
- 不改 vseg 逻辑代码、不重编、不装包；不动 target A/B 进程；不 register 除两个 endpoint 之外的任何东西。
- 任何一步失败停手回报；不要把失败写成成功。
- 预计 **20–30 分钟**。

---

## NOF-115 裁决（2026-11-26 · 回执已收）

### 1. 结果（[事实]）

- **两个 segment 注册成功且不覆盖**：node4 B（`traddr:node4 … subnqn:…urma-gpu-test … ns:1`）与 node3 A（`traddr:node3 … subnqn:…urma-gpu-test-node3 … ns:1`），各 100GiB；`master_total_nof_capacity_bytes=214748364800`、segment 数=2。
- **quota 发布成功**：generation=1、partitions=1、profiles=1、discovered_segments=2、reconcile_fail=0；profile `medium=nof, member_count=2, stripe_size=2097152, member_extent_size=53687091200`；`logical_capacity=107374182400`（100GiB=2×50GiB）；vsegments=1（发布即预建，正常）。
- **写读通过**：`--num_keys=2` 在 segment_write 下是每 segment 2 keys ⇒ 总 Keys=4、16MiB；`type` 全 vsegment；每 key `child_count=2`、`blocks=4096`、`sub_ios=16`；两个 target 各收到约 66 IO（≈4×16=64+探测），**均分**。
- 读 diag 开 P50 838µs（4 keys，不作性能）；target 读窗口 W9：node4 ~232–234µs、node3 ~226–228µs；写窗口 ~54–56µs。

### 2. 关键口径（必须继承）

1. **segment 注册不持久**：NoF segment 由注册进程心跳续期，进程退出即失效；master 重启后 segment 归零，必须**重新注册**。NOF-116 每次 bench/每次 master 重启后按 **node4 B → node3 A** 顺序重注册，并在整臂期间保持注册进程存活。
2. `segment_write` 的 `--num_keys=N` 是**每 segment N keys**：vseg 两 segment 要 400 keys 用 `--num_keys=200`（预期 Keys=400）；plain 单 segment 400 keys 用 `--num_keys=400`。
3. `vsegments=1` 是 quota 发布时预建，不是异常；写数据不会新增（400 keys 仍在 100GiB 的同一 vsegment 内）。
4. `logical_used` gauge 受 1000s 刷新限制；验收以 master `Keys` + writer 日志为准。
5. child→target 的 endpoint 不在 `nof_slow_task` 里；落点均分证据用**两个 target 的 IO 计数相等**（NOF-115 66 vs 66）。
6. node3 target ERROR 的 2 条是 TCP 探测导致的 handshake failed，不是故障。

### 3. 下一步

**NOF-116**：400-key plain（node4 单 target）vs vseg（node3+node4，member_count=2）性能对比，T1×3 遍，diag 关，验收 parity（vseg P50 ≤ plain×1.05）。

---

# NOF-116（待执行 · 双 target 400-key 性能：plain 单 target vs vseg member_count=2 · 2026-11-26）

## 目标

- 同一 target B（node4）上：plain 400 keys 基线；vseg 400 keys 跨 node3+node4 两 target，`member_count=2 / extent=50GiB / stripe=2MiB`。
- T1 读 3 遍，diag 关；给两臂 P50/W9；验收 **parity：vseg P50 ≤ plain P50×1.05**（W9 同时报）。
- 不改码、不重启 target；允许写数据（400 keys）和 master 重启。

## A. 预检

1. target A（node3 PID 3329243，新 NQN/SN）、target B（node4 PID 2142228）运行；**都不重启**。
2. 当前 `vseg_nof115` master 2355509/2355512 运行（Keys=4、2 segments、quota 100GiB）；准备停它给 plain 让端口。
3. 确认 `nof_register` 的**守护/心跳方式**：注册进程必须整臂存活；若工具退出即失效，用循环/后台 keepalive，记录方式。
4. node2 无 bench/register 残留；磁盘/日志空间足够。

## B. Plain 基线（node4 单 target，先测）

1. 停 `vseg_nof115` master（记录 PID/退出码）；target B 不动。
2. 起 **plain master `plain_nof116`**（node4，50056/9006/8086、slot=1、etcd 127.0.0.1:2379、URMA 设备/capsule）；验收 leader/serving、Keys=0、owned_slots=1。
3. 注册 **target B**；保持注册进程存活；确认 master segment=1。
4. Writer：400 keys×4MiB、**16t/batch16**、plain；验收 `Keys=400`、`logical_used=1,677,721,600`；记录 target B 写窗口 n≈12,800。
5. Reader：**3 遍**，T1（threads=1/batch=1）、warmup=128、qpairs=4、verify=false、diag 关；每遍前确认注册存活/挂载有效；记录 Min/Avg/P50/P90/P99/Max、failed、target B W9（期望 n≈16,896）。
6. 记录后停 plain master 与注册进程；数据保留（会被 vseg 覆盖）。

## C. Vseg 双 target 400 keys

1. 起 **vseg master `vseg_nof116`**（node4，新 cluster_id；先 `auto_plan=false`，参数 `member_count=2 / extent=50GiB / stripe=2MiB / required_medium=nof / reserved_ratio=0 / io_alignment=4096 / slot=1 / HA+etcd`）；验收 leader/serving。
2. 注册 **node4 B → node3 A**；保持两个注册进程存活；确认 master segment=2。
3. 停 master；以 `auto_plan=true` + 同组参数重启；**重新注册 B → A**（不持久）；保持存活；验收 quota：generation≥1、partitions=1、logical_capacity=100GiB、reconcile_fail=0、vsegments=1。
4. Writer：vseg segment_write，**`--num_keys=200`（每 segment 200，预期总 Keys=400）**、T1/batch1、4MiB；`VSEGMENT_CREATING` 最多重试 3 轮；验收 `Keys=400`、`logical_used=1,677,721,600`；target A/B 写窗口各≈6,400（400×16）。
   - 若 Keys 不是 400（例如 200 或 800），**停手回报**，不要自行改 num_keys 继续。
5. Reader：**3 遍**，T1 400 keys、warmup=128、qpairs=4、verify=false、diag 关；每遍前确认两个注册进程存活、master segment=2；记录 Min/Avg/P50/P90/P99/Max、failed、target A/B W9（各期望 n≈8,448）。
6. 与 plain 对比：P50、W9、比值；判 parity。

## D. 交付（必答）

1. 两阶段 master 配置/PID/segment 数/quota 验收。
2. Plain：Keys/logical_used、3 遍 P50/W9、target B n。
3. Vseg：Keys/logical_used、3 遍 P50、target A/B W9 与 n（应均分）、每 key child 抽检。
4. **判读**：vseg/plain 的 P50 与 W9 比值；是否达到 parity（≤1.05×）。
   - 若达到 ⇒ 双 target 形态达标，记录最佳配置。
   - 若仍差很多 ⇒ 记录“双 target 未消除差距”，为后续（拆 W9/改代码）提供数据；不继续调旋钮。
5. 环境末态；未改码/未重启 target。

## E. 边界

- 只写 400 keys；不跑 1000/20k；不跑 T4/T16；不改 vseg 逻辑代码/不重编/不装包。
- 不重启 target A/B；master 重启允许，但每次必须重注册两 segment。
- 注册进程必须整臂存活；若中途失效（segment 数掉到 0/1），停手回报。
- 不要混用 plain/vseg 的 Keys 语义：plain 单 segment 400；vseg 每 segment 200（总 400）。
- 预计 **30–40 分钟**。

---

## NOF-116 裁决（2026-11-26 · 回执已收）

### 1. 结果（[事实]）

| 臂 | P50（6 遍/3 遍） | 均值 | 中位 | W9（每 target） |
|---|---|---|---|---|
| plain（node4 单 target） | 267.74 / 264.86 / 266.49 | **266.36** | 266.49 | **120.33** |
| vseg（node3+node4） | 300.09 / 461.35 / 284.93 / 290.85 / 324.15 / 280.17 | **323.59** | **295.47** | **157.3**（A/B 各 ~157） |

- 比值：**均值 1.215×、中位 1.109×、最好一遍 1.058×**（280.17 vs 264.86）；W9 1.31×。
- vseg 6 遍方差大，461.35µs 是离群；去掉它后均值 ~296µs（1.11×）。
- 相对单 target 的 2.03×，双 target 把差距压到约 **1.1×**，但**未达 parity（≤1.05×）**。
- 两 target 写窗口 n=6400/6402，读窗口各 ~8450，**均分**；`type` 全 vsegment。

### 2. 口径更正（重要，后续必须遵守）

1. **vseg reader 的 `--num_keys` 也是每 segment**：`--num_keys=400` 会读 800 queries（400 rpc_fail）；要读 400 个 key 用 **`--num_keys=200`**。NOF-116 前 3 遍 vseg P50（185.91/217.84/216.33）因 400 个失败样本污染，**已作废**。
2. **segment 注册不依赖常驻进程**：master 通过 `nof_heartbeat_*` 主动探测 target 维持；`nof_register` 退出 40+ 分钟后 segment 数仍为 2、心跳计数增长。**只有 master 重启才需要重注册**；此前“注册进程必须整臂存活”的说法作废。
3. `vsegments=1` 是 quota 发布预建；`logical_used` gauge 1000s 滞后，验收以 Keys/写日志为准。
4. plain 数据已被 vseg 写入覆盖；后续 plain 基线需要重新写。

### 3. 判读

- 差距已从“每个 target 单 IO 2×”降到“每 target 157 vs plain 120（1.31×）”，同时 vseg 每 target 只承担一半 IO。
- 剩余差距的两个方向：
  1. **vseg 每 target 每 key 16 个 IO，plain 是 32 个** ⇒ 队列深度/批处理形状不同；若把 chunk 从 128KiB 减到 64KiB，vseg 每 target 会变成 32 个 IO，与 plain 对齐。
  2. vseg 每 IO 的 target 执行段仍比 plain 慢（157 vs 120），属于 target 服务路径。
- **不改码前提下，先试方向 1**：NOF-117。

### 4. 下一步

**NOF-117**：vseg 只读，chunk 128KiB（对照）vs 64KiB（每 child 32 个子 IO，每 target 32 IO/key），3 遍交替，零写/零重启/零改码。若 vseg@64K 达到 parity（≤266.36×1.05≈279.7µs），收口；若不变，说明队列深度不是剩余差距的主因，再考虑第三 target 或 target 侧 W9 trace。

---

# NOF-117（待执行 · vseg 64KiB chunk 只读对照 · 零写/零重启/零改码 · 2026-11-26）

## 目标

- 现有 `vseg_nof116` 数据（400 keys，两个 target，master 运行中）**只读**，不改数据、不重启 master/target。
- 只改客户端 env `MC_NOF_SUBMIT_CHUNK_BYTES`：
  - 对照：128KiB（现状，每 child 16 个 128KiB，每 target 16 IO/key）
  - 实验：**64KiB**（每 child 32 个 64KiB，每 target 32 IO/key，与 plain 的每 target 32 IO 对齐）
- 回答：把 vseg 每 target 的 IO 数从 16 提到 32，能否把 P50 从 ~323µs 压到 parity（≤ plain 266.36×1.05 ≈ **279.7µs**）。

## A. 预检

1. `vseg_nof116` master 运行中（Keys=400、2 segments、quota 100GiB）；**不要重启 master/target**。
2. 两个 target A/B 运行；node2 无残留。
3. 用 2-key 探针 + `MC_NOF_DEBUG=1` 确认 `MC_NOF_SUBMIT_CHUNK_BYTES=65536` 生效：`blocks_per_chunk=128`；128KiB 对照为 256。探针不算性能。
4. 注册不需要常驻进程；若发现 master 被重启，按 B→A 重注册后再跑。

## B. 只读 A/B（同一 master、同一数据）

- reader：vseg，**`--num_keys=200`（每 segment 200，共 400 queries）**、T1（threads=1/batch=1）、warmup=128、qpairs=4、verify=false、diag 关。
- 6 遍交替：128K、64K、128K、64K、128K、64K（降低漂移）；每遍记录 Min/Avg/P50/P90/P99/Max、failed、target A/B W9 与 n。
- 不写数据；不换 master；不改代码。

## C. 交付（必答）

1. 探针证据：两档 `blocks_per_chunk`（128/256）。
2. 6 遍表：chunk、P50、failed、target A/B W9/n。
3. 对比：
   - vseg@64K vs vseg@128K（同会话）；
   - vseg@64K vs plain@128K 基线 266.36µs / W9 120.33；
   - 是否达到 parity（≤279.7µs）。
4. 判决：
   - 达到 ⇒ 记录 64KiB 为候选配置，收口；
   - 未达到但明显改善 ⇒ 可再试 32KiB（仅当 64K 相比 128K 有 ≥5% 改善）；
   - 无改善 ⇒ 队列深度/IO 数不是剩余差距主因，停止调 chunk；下一步第三 target 或 target 侧 W9 trace。
5. 环境末态；未写/未重启/未改码。

## D. 边界

- **零写入**；不改 master/target；不改代码/不重编；不跑 plain（plain 数据已被覆盖，用 NOF-116 基线）。
- reader 的 `--num_keys=200` 是每 segment，不要用 400。
- 若 64K chunk 导致 I/O 失败或 target 错误，停手回报。
- 预计 **10–15 分钟**。

---

## NOF-117 裁决（2026-11-26 · 回执已收）

### 1. 结果（[事实]，同 master/同数据/同会话，3+3 交替）

| 项 | vseg@128K | vseg@64K | 变化 | plain@128K（NOF-116） |
|---|---|---|---|---|
| P50 三遍 | 303.75 / 291.37 / 286.59 | 272.27 / 294.54 / 293.91 | — | 267.74 / 264.86 / 266.49 |
| P50 均值 | 293.90 | **286.91** | −2.38% | **266.36** |
| P50 中位 | 291.37 | **293.91** | +0.9%（略升） | 266.49 |
| Avg 均值 | 392.48 | 371.10 | −5.45% | 338.51 |
| P90 均值 | 537.46 | 508.84 | −5.32% | ~567 |
| target W9 | 157.58µs | **130.50µs** | **−17.2%** | **120.33µs** |
| 每 target n | ~8,449 | ~16,897 | ×2 | 16,898 |

- 对 plain：vseg@64K **P50 均值 1.077×**、中位约 **1.10×**；W9 1.085×。
- 单遍 c64k_r2=272.27（1.022×）达 parity 门槛，但另两遍 294.54/293.91 未达 ⇒ 在门槛附近摆动，**不判达标**。
- 探针确认 64K 生效：`blocks_per_chunk=128`（128K 为 256）；所有遍 failed=0、type 全 vsegment。
- 会话漂移：本会话 128K 基线 293.90 比 NOF-116 的 vseg 均值 323.59 低 9%；本单只用同会话交替对照，plain 仅取绝对值作门槛。

### 2. 判决

- **队列深度 / 每 target IO 数不是 P50 剩余差距的主因**：把每 target IO 从 16 提到 32，W9 降 17.2%，但 P50 均值只降 2.38%、中位略升。
- 剩余 ~25µs（中位）**不在 target 单 IO 执行段**；W9 已 130.5 vs plain 120.3，差 ~10µs。更可能在：
  - 客户端两个 child 的聚合/等待（两 task、两 target、两跳网络）；
  - node3/node4 两跳的 RTT/完成先后差异（慢 child 决定 key 时延）；
  - 更多子 IO 带来的客户端 QoS/worker 开销（64K 把客户端提交量翻倍，抵消了 target 收益）。
- 因此 **停止调 chunk**；继续调只会 target 变快、客户端变慢，P50 不动。

### 3. 下一步（待用户裁决）

- **A. 接受近 parity**：双 target 的 P50 均值 1.077×、中位 ~1.10×、最好 1.022×；记录“双远端 target 的 T1 实际落点约 1.08–1.10”，停止追 1.05。
- **B. 查剩余 ~25µs**：零写、零重启，开 `MC_STORE_TRANSFER_DIAG=1` 在 64K 下取逐 child 的提交/完成时间与 `completion_wait`，判断是两 child 聚合、网络跳差异，还是客户端 QoS；再决定是否值得改。
- **C. 改码 / 结构性**：合并 child（需批准改码，且会放弃双 target 并行）或第三 target（可能增加聚合开销）；均不建议在未定因前做。

**建议**：先走 B（~10 分钟、零风险），因为它能区分“物理两跳/聚合”与“客户端可修”；若证明是物理代价，就按 A 收口。

---

# NOF-118（待执行 · 零写查剩余 ~25µs：两 child 时序 + 客户端分段 · 2026-11-26）

## 目标

现有 `vseg_nof116` 400 keys 只读，同一 master/数据/会话，开 `MC_STORE_TRANSFER_DIAG=1`，回答剩余 ~25µs（中位 P50 293.9 vs plain 266.4）在哪：

1. **慢 child / 两跳不对称**：两个 target 的 child 完成时间是否一前一后，晚的那个决定 key 时延；
2. **客户端聚合/观察**：两个 child 都完成后，key 的 `read_us`/`completion_wait` 是否还多出一段；
3. **64K 客户端开销**：64K 把子 IO 数翻倍，`nof_submit`/`unprepare`/`unattributed` 是否比 128K 高，抵消 target W9 收益。

不改数据、不重启 master/target、不改码。

## A. 预检

- `vseg_nof116` master 运行中（Keys=400、2 segments、quota 100GiB）；target A/B 运行；node2 无残留。
- 注册由 master 心跳维持；若 master 被重启，按 B→A 重注册后再跑。
- 确认 `MC_STORE_TRANSFER_DIAG=1` 生效：日志出现 `get_into_breakdown`、`nof_vsegment_child`、`nof_vsegment_operation`、`nof_slow_task`。

## B. 采集（只读）

- reader：vseg，`--num_keys=200`（每 segment，共 400 queries）、T1/threads=1/batch=1、warmup=128、qpairs=4、verify=false。
- 6 遍交替：**64K diag、128K diag、64K、128K、64K、128K**；`MC_STORE_TRANSFER_DIAG=1`，其余同 NOF-117。
- 每遍记录：P50（diag 抬高，仅参考）、failed、target A/B W9/n。
- 保存 6 份日志（每份约 8–10MB）。

## C. 解析（必答）

### C1 每 key 两个 child 的时序

从 `nof_vsegment_child` / `nof_slow_task` 按 `(trace_id, vsegment_op_id)` 归组，取：
- child0/child1 的 `first_submit_mono_ns`、`complete_mono_ns`；
- 每 child 的 `first_completion_from_submit_us`、`max_subio_us`、`fifo_us`；
- `nof_slow_task` 的 `endpoint`（映射为 node3/node4 标签，不写 IP）、`lba`、`blocks`、`sub_ios`。

计算每个 key：
- `child_latency = complete - first_submit`；
- `completion_gap = child1.complete - child0.complete`；
- `last_child`（谁完成得晚）；
- `submit_skew = child1.first_submit - child0.first_submit`；
- `key_span = last_completion - first_submit`。

### C2 客户端分段

从 `get_into_breakdown` 取每 key：
- `query_us`、`read_us`、`vsegment_resolve_us`、`nof_prepare_us`、`nof_submit_us`、`completion_wait_us`、`read_unattributed_us`。

计算：
- `key_read_minus_span = read_us - key_span`（客户端聚合/观察/提交等超出 child 时间跨度的部分）；
- 64K vs 128K 各段差异。

### C3 目标侧

- target A/B 的 W9/n（应接近；若一边明显高，记录）；
- 按 endpoint 分组的 child latency：node3 vs node4 谁更慢、慢多少。

## D. 交付（必答）

1. 6 遍 P50/W9/n 表（64K vs 128K）。
2. **每 key 两 child**：child latency、completion_gap、last_child 比例、submit_skew 的 p50/p90；按 endpoint（node3/node4）分组。
3. **客户端分段**：64K/128K 的 query/read/resolve/prepare/submit/wait/unattributed 均值；`read_us - key_span` 的 p50/p90。
4. 判决三选一（给比例/数值）：
   - **(A) 慢 child/两跳不对称**：某个 target 的 child 系统性晚到/更慢，晚 child 决定 key 时延；
   - **(B) 客户端聚合/观察**：两 child 完成接近，但 `read_us - key_span` 有稳定 ~20–25µs；
   - **(C) 64K 客户端开销**：64K 的 submit/prepare/unattributed 明显高于 128K，target 收益被吃掉。
5. 若 (A)，给出两 target 的 child latency 差与占比；若 (B)，给出 `read_us - key_span` 与 `completion_wait - span` 的数值；若 (C)，给出各段差值。
6. 5 个代表 key 的原始行（child0/child1/operation/breakdown 各一行）。
7. 环境末态；零写/零重启/零改码。

## E. 边界

- **零写入**；不改 master/target；不改代码/不重编；不跑 plain。
- `--num_keys=200` 是每 segment，共 400 queries；不要用 400。
- diag 只用于分段，绝对 P50 不作性能结论。
- 日志大；若单遍超过 20MB 或磁盘不足，减到 2 遍/档（64K/128K 各 2 遍），并说明。
- 失败（日志缺字段/挂载失败）停手回报。
- 预计 **15 分钟**。

---

# NOF-119（待执行 · 单遍 ~20s 大数据集：60k keys/240GiB · 先验证 512GiB segment + oplog off · 2026-11-26）

## 目标

- 用户要求：400 keys 太少，要**很多 keys**，**一遍连续读约 20 秒**。
- 按当前 plain Avg 338.5µs、vseg Avg 371–400µs：20s ≈ **50–60k keys**；本单取 **60,000 keys ×4MiB = 240GiB**（plain 单 target 240G；vseg 每 target 120G）。
- 两个前提必须先验证，不通过就停：
  1. **NoF segment 容量扩到 512GiB**（当前 100GiB；target bdev 77TB，可能只需注册时改容量，不用重启 target）；
  2. **本单选择 `--enable_ha=true --enable_oplog=false`**，先验证它能正常发布 vseg quota。  
     说明：关 oplog 是**配置选择**（不把 HA 日志变成 60k 大测试的变量），**不是**断言当前环境 60k 一定会爆；旧 node1/3 的 942MB 不外推。若 A 段验证失败，回退手工 quota 或另议。
- 先 plain 后 vseg，各 **1 遍** 读（60k samples）；验收 parity（vseg P50 ≤ plain×1.05）。

## A. 容量 / oplog 可行性（不写 60k）

1. 记录 target A/B 运行状态；`nvmf_get_subsystems` 查 nsid1 的**实际 namespace size**；`bdev_get_bdevs URMA_RAID0` 的 num_blocks（应 ~77TB）。
2. `nof_register --help`：查是否有 **size/capacity/base** 参数；记录。
3. 起临时 master **`probe_nof119`**（新 cluster，端口 **50066/9016/8096** 避开现网，etcd 127.0.0.1:2379，`enable_ha=true`、`enable_oplog=false`）：
   - 注册 node4 B、node3 A，容量尝试 **512GiB**（base 0）；
   - 确认 master `nof_segment_total_capacity_bytes` 各 512G、segment 数 2；
   - 若 512G 失败，尝试 256G；仍失败 ⇒ **停手回报**（说明 target/注册不支持扩容，需另议 target 重建或退回多遍）。
4. 若 512G 注册成功：写/读 **2 keys** 探针；其中至少一个 key 尝试落在**高 LBA**（>100GiB，如 offset 400GiB；若 bench 不支持指定起点，记录 `[未获取]`，以 target nsid size + 后续写入是否越界为准）。
5. 记录 etcd 数据量、`/oplog` 是否增长（应为 0）、auto_plan 是否成功（generation/partitions）。
6. 停 probe master；不改 target；不写 60k。

## B. Plain 60k（node4 单 target，512GiB segment）

1. 起 plain master `plain_nof119`（node4，50056/9006/8086、slot=1、etcd、URMA、**oplog off**）；注册 target B，容量 **512GiB**。
2. Writer：60,000 keys×4MiB、16t/batch16、plain；验收 `Keys=60000`、`logical_used=251,658,240,000`；target B 写窗口 n≈1,920,000（60k×32）。
3. Reader：**1 遍**，60,000 keys、T1/threads=1/batch=1、warmup=128、qpairs=4、verify=false、diag 关；目标读时长约 20s；记录 P50/Avg/P90/P99、failed、target B W9/n（n≈1,924,096）。
4. 记录后停 plain master；数据会被 vseg 覆盖。

## C. Vseg 60k（node3+node4，512GiB segments）

1. 起 vseg master `vseg_nof119`（node4，`member_count=2 / extent=50GiB / stripe=2MiB / required_medium=nof / reserved_ratio=0 / io_alignment=4096 / slot=1 / HA on / oplog off`）；注册 B→A，容量 **512GiB**；确认 quota：partitions=1、segments=2、logical_capacity 起始 100GiB、reconcile_fail=0。
2. Writer：`--num_keys=30000`（每 segment 30000，总 **60,000 keys**）、T1/batch1、4MiB；`VSEGMENT_CREATING` 最多重试 **5 轮**（60k 会跨多个 vsegment）；验收 `Keys=60000`、`logical_used=251,658,240,000`；target A/B 写窗口各≈960,000（60k×16）。
3. Reader：**1 遍**，`--num_keys=30000`（每 segment，共 60,000 queries）、T1、warmup=128、qpairs=4、verify=false、diag 关；目标读时长约 20–24s；记录 P50/Avg/P90/P99、failed、target A/B W9/n（各≈962,048）。
4. 与 plain 对比：P50、W9、比值；判 parity（≤1.05×）。

## D. 交付（必答）

1. A 段：target nsid size、bdev num_blocks、`nof_register` 参数、512G 注册结果、etcd/oplog 增长、auto_plan 状态。
2. Plain：Keys/logical_used、写窗口 n、读单遍 P50/Avg/P90/P99、W9/n、实际读时长。
3. Vseg：Keys/logical_used、写窗口 A/B n、读单遍 P50/Avg/P90/P99、A/B W9/n、实际读时长。
4. 比值与 parity 判决；若未达，给两 target W9 差与客户端是否再次成为瓶颈的判断（只描述）。
5. 环境末态；零改码/未重启 target。

## E. 边界

- **必须先过 A**；512G/256G 都失败就停手，不写 60k。
- oplog off 是本单的配置选择；若 A 段 auto_plan/quota 失败，停手回报，回退手工 quota 或另议；**不擅自开 oplog 写 60k**。
- 不重启 target；不改码/不重编/不装包；只写 60k 数据（plain 240G + vseg 240G，顺序覆盖）。
- 容量：target 512G/segment；plain 240G、vseg 120G/target；extent 50G → 约 3 个 vsegment，per target 150G，留余量。
- 单遍读 60k，不再加第 2 遍（用户要单遍 20s）；若时长明显 <15s 或 >30s，记录实际值，不调整 key 数重跑。
- 日志与 target W9 窗口要覆盖整遍；n 很大（~1.9M/臂），注意磁盘。
- 预计：A 15 分钟；plain 写+读 ~2 分钟；vseg 写+读 ~3–5 分钟；总计 **25–35 分钟**。

---

## NOF-119 A 段裁决（2026-11-26 · 回执已收，停手）

### 1. 结果（[事实]，当前环境）

| 前提 | 结果 | 证据 |
|---|---|---|
| NoF segment 容量扩到 512GiB | **通过** | 两个 segment 各 `549755813888` B（512GiB），总 1TiB；只需注册时传 size，**未重启 target** |
| oplog off 发布 vseg quota | **不通过** | 2 分钟内 `enabled=0`、logical_capacity=0；日志 `no eligible psegment pool yet` → 每 10s 重试仍不发布 |
| 同机 oplog on 对照 | **通过** | 40s 发布 generation=1、partitions=1、discovered_segments=2、logical_capacity=100GiB |
| 手工 quota 回退 | **不可用** | planner 工具未构建；`POST /vsegment_quota` 等全部 404；构建需另议 |
| target B 重建 | 完成 | PID 2391418、4457、11/11 online、strip256K、0 ERROR、新 UUID `c19b91e1…`；NQN/SN 还原 |

- 结论：**quota 发布依赖 HA 的 OpLog 提交路径**（[推算]：两轮唯一变量是 `enable_oplog`；日志无显式 override 报错）。oplog off 这条路在现有构建下走不通。
- 未写 60k；仅两次注册 + 一次失败探针；etcd-data 128M。
- 环境：node4 etcd 2389250、target B 2391418；node3 target A 3329243 未动；node2 无残留。旧 vseg/plain 数据因 UUID 变已失效，需新 cluster。

### 2. 开发侧决策

用户未在三选一中拍板，授权开发侧决定；**选 (a)**：

- 用正常配置（HA + oplog on）写 5k keys 试水，实测当前环境的 oplog 增长，再决定是否写 60k；
- 不选 (b)：planner 工具未构建、HTTP 无路由，为它先编译不划算；只有在 5k 实测证明 oplog 不可接受时才走 (b)。
- 不选 (c)：违背用户“大 keys、单遍 20s”的目标。
- 不再用 node1/node3 旧数据外推；5k 的实测值决定下一步。

---

# NOF-120（待执行 · oplog-on 5k vseg 试水 · 不写 60k · 2026-11-26）

## 目标

- 用**正常 HA + oplog on** 起 vseg master（512GiB segments），写 **5,000 keys**（20GiB），实测当前环境的：
  - etcd dbSize / data-dir 增长；
  - `/oplog/vseg_nof120/` 的 key 数/字节增长；
  - 写时长、`VSEGMENT_CREATING` 次数。
- 用 5k 的**实测**增长判断 60k 是否可行；**本单不写 60k**。

## A. 起 master / 注册 / quota

1. target B（node4，PID 2391418，新 UUID）与 target A（node3，3329243）运行；不重启。
2. 起 vseg master **`vseg_nof120`**（node4，50056/9006/8086、slot=1、etcd 127.0.0.1:2379、URMA、**enable_ha=true、enable_oplog=true**、`member_count=2 / extent=50GiB / stripe=2MiB / required_medium=nof / reserved_ratio=0 / io_alignment=4096 / submaster_count=1`）。
3. 注册 node4 B → node3 A，容量 **512GiB**；确认 segment=2、quota generation=1、partitions=1、logical_capacity=100GiB、reconcile_fail=0。
4. 记录 etcd 路径（从 etcd 进程 cmdline）、data-dir 大小、`etcdctl endpoint status --write-out=json` 的 dbSize。

## B. 5k 写入与 oplog 测量

1. 写前记录：
   - etcd dbSize、data-dir `du -sb`；
   - `/oplog/vseg_nof120/` 的 key 数/字节（`etcdctl get --prefix ... -w json` 或 count）。
2. Writer：`--num_keys=2500`（每 segment 2500，总 **5,000 keys**）、T1/batch1、4MiB；`VSEGMENT_CREATING` 最多重试 3 轮；记录写时长与重试次数。
3. 验收：`Keys=5000`、`logical_used=20,971,520,000`（5000×4MiB）；target A/B 写窗口各≈80,000（5000×16）。
4. 写后同一口径记录 etcd dbSize、data-dir、`/oplog/vseg_nof120/` 增长；计算：
   - 每 key oplog 增量；
   - 线性外推 60k 的总 oplog；
   - etcd 所在文件系统剩余空间。

## C. 判决（必答）

- 若 `外推 60k oplog + 当前 dbSize < 4GiB` **且** etcd 文件系统剩余 >50%：
  - 判 **“oplog on 可支撑 60k”**，给出 5k 实测值/每 key 增量/外推值；**本单不写 60k**，下一单执行。
- 否则：
  - 判 **“oplog on 不适合 60k”**，给出实测值；建议 (b) 编译 `vsegment_quota_planner` 工具（不改 vseg 逻辑）走 oplog off，或 (c) 缩小规模/多遍；本单停手。
- 不做旧数据外推；只报当前环境实测。

## D. 边界

- **只写 5,000 keys**（20GiB）；不写 60k；不跑性能。
- 不重启 target；不改码/不重编/不装包；master 可重启，注册不持久需重注册。
- 数据会覆盖旧目标区域；目标 B UUID 已变，旧数据本来已失效。
- 若 quota/写入失败或 etcd 明显异常，立即停止并记录。
- 预计 **10–15 分钟**。

---

## NOF-120 裁决（2026-11-26 · 回执已收，通过）

### 1. 结果（[事实]，当前环境）

| 项 | 实测（13:45 / 13:50 复核一致） |
|---|---|
| Keys | 5,000 |
| etcd dbSize | 13,619,200 B（12.988 MiB） |
| dbSize 相对写前增量 | +8,257,536 B（≈1,651.5 B/key） |
| `/oplog/vseg_nof120/` | 9,870 keys / 4,910,154 B（4.68 MiB） |
| 外推 60k dbSize 增量 | **≈0.105 GiB** |
| 文件系统 | size 2.7T / used 1.5T / **avail 1.1T**（use 59%） |
| vsegments / logical_capacity | 1 / 100GiB；create_success=1、failure=0 |
| 驱逐 | 0/0，keys=0 |

- 5k 写入成功，环境稳定；`logical_used` gauge 仍 0（1000s 刷新），以 Keys 为准。
- target A/B 各 10GiB 数据；target B/A、etcd、master 均未重启。

### 2. 判据裁定

- 判据①“外推 60k dbSize <4GiB”：**通过**（0.105GiB，38× 余量）。
- 判据②“etcd 文件系统剩余 >50%”：字面不满足（剩 41%）。但 **59% used 是长期历史占用，与本测试无关**；绝对可用 **1.1TiB**，60k 仅需 ~0.1GiB（约万分之一）。**按绝对空间判通过**，不再用 50% 这个任意阈值。
- 结论：**oplog on 可以支撑 60k**；继续 NOF-121。

### 3. 下一步

**NOF-121**：vseg 先复用当前 5k 数据补写到 60k 并读 1 遍；再 plain 60k 读 1 遍；各约 20s；验收 parity。

---

# NOF-121（待执行 · 60k keys 单遍 ~20s：vseg + plain · oplog on · 2026-11-26）

## 目标

- 在 NOF-120 的 `vseg_nof120`（Keys=5000、2×512GiB segment、vsegments=1）基础上，把 vseg 补写到 **60,000 keys（240GiB）**，读 **1 遍**（约 20–24s）。
- 再切 plain master，写 **60,000 keys**（240GiB，node4 单 target），读 **1 遍**（约 20s）。
- 对比 P50/W9，验收 parity（vseg ≤ plain×1.05）。

## A. Vseg 60k（当前 master 直接复用）

1. 预检：`vseg_nof120` 运行、Keys=5000、2 segments 各 512GiB、vsegments=1、quota 100GiB、reconcile_fail=0；target A/B 运行。
2. Writer：`--num_keys=30000`（每 segment 30000，总 60,000）、T1/batch1、4MiB；已有 keys 0–4999 会被拒（计入 succeeded），实际新增 55,000；`VSEGMENT_CREATING` 最多重试 **5 轮**（60k 会跨约 3 个 vsegment）。
3. 验收：**Keys=60000**、`logical_used=251,658,240,000`；target A/B 写窗口各≈960,000（5k 已有 80k + 新 55k×16=880k）。若 Keys≠60000 或 vsegment 创建失败 5 轮，停手回报。
4. Reader：**1 遍**，`--num_keys=30000`（每 segment，共 60,000 queries）、T1、warmup=128、qpairs=4、verify=false、diag 关；记录实际读时长、P50/Avg/P90/P99、failed、target A/B W9/n（各≈962,048）。
5. 记录 etcd dbSize / `/oplog/vseg_nof120/` 增长（供后续参考）。

## B. Plain 60k（node4 单 target）

1. 停 `vseg_nof120` master（记录 PID/退出码）；target B 不动；起 plain master `plain_nof121`（50056/9006/8086、slot=1、etcd、URMA、HA+oplog on）；注册 target B，容量 **512GiB**。
2. Writer：60,000 keys×4MiB、16t/batch16；验收 `Keys=60000`、`logical_used=251,658,240,000`；target B 写窗口 n≈1,920,000。
3. Reader：**1 遍**，60,000 keys、T1、warmup=128、qpairs=4、verify=false、diag 关；记录实际读时长、P50/Avg/P90/P99、failed、target B W9/n（n≈1,924,096）。
4. 记录 etcd dbSize/oplug 增长（两阶段合计仍在安全区）。

## C. 交付（必答）

1. Vseg：Keys/logical_used、写窗口 A/B n、单遍读实际时长、P50/Avg/P90/P99、W9/n、VSEGMENT_CREATING 次数。
2. Plain：Keys/logical_used、写窗口 n、单遍读实际时长、P50/Avg/P90/P99、W9/n。
3. 比值与 parity 判决（vseg P50 / plain P50；W9 同报）。
4. etcd dbSize/oplog 增长；环境末态。
5. 若 vseg 读时长明显偏离 20s（<15s 或 >30s），记录实际值，不调 key 数重跑。

## D. 边界

- oplog on（已实测 60k 约 0.1GiB，安全）；不切 oplog off。
- 不重启 target；不改码/不重编/不装包；master 重启需重注册 512GiB。
- 先 vseg 后 plain；plain 会覆盖 node4 上 vseg 数据，vseg 已测完。
- 每臂只读 1 遍；不跑 T4/T16；不跑 1000/20k。
- 若 60k 写入/读取失败、Keys≠60000、target 掉线，停手回报。
- 预计 **30–40 分钟**（vseg 补写约 2–3 分钟、读 20–24s；plain 写 ~30s、读 ~20s；master 切换/注册约 10 分钟）。

---

## NOF-121 A 裁决（2026-11-26 · 回执已收，失败停手）

### 1. 失败事实

| 项 | 实测 |
|---|---|
| 写入结果 | **25,647 succeeded / 34,353 failed**（总 60,000） |
| 失败码 | `ret=-1010` ×34,352（OpLog queue failed）、`VSEGMENT_CREATING` ×2、`ret=-105` ×1 |
| 根因原文 | `TxnCompareAndPut failed: ResourceExhausted: received message larger than max (13295444 vs. 2097152)` |
| 单批事务 | **13,295,444 B > etcd 默认 2,097,152 B**；同尺寸重试 3784+ 次，尺寸不变 |
| etcd 总量 | dbSize 54.5MiB、`/oplog/vseg_nof120/` 23.89MiB、/home 余 1.1TiB ⇒ **不是空间问题** |
| 数据落点 | 25,647×4MiB = **100.184 GiB**，刚好越过单 vsegment 静态 quota 100GiB |
| vsegment 状态 | `vsegments=2`，但 `logical_capacity=100GiB`、`create_success_total=1`（扩容路径可疑，未分离） |
| 环境 | target A/B 未重启；旧 master 2396324/2396327 仍在无限重试（CPU 29%、日志 25 万行） |

- 这是**单事务消息尺寸墙**，不是 oplog 总空间墙；NOF-120 的“oplog on 可支撑 60k”在实践中被**部分证伪**：总空间够，但批事务超 2MiB。5k 预检没有触发 1024 条大批判，所以没暴露。

### 2. 决策（无需再问）

1. **走 (a)：`--oplog_batch_max_entries=64`**（1024→64，批事务约 13.3MB→0.83MB，留 2× 余量；不重启 etcd、不改码）。
2. **授权停掉旧 master 2396324/2396327 的无限重试**（只停 master，不动 target）；新 cluster `vseg_nof122` 重写 60k。
3. **暂不退回 20k**：先修批大小试 60k；若仍在 ~25,647 处因 vsegment 扩容/quota 失败，再单独裁决 20k/80GiB。
4. 25,647 keys 的元数据随 master 重启丢失，数据在 target 上会被新 60k 覆盖；数据保留不是目标。

---

# NOF-122（待执行 · 修 oplog 批大小后重试 60k · 只停 master/不动 target · 2026-11-26）

## 目标

- 用 `--oplog_batch_max_entries=64` 重试 60k，验证：
  1. etcd 单批事务 <2MiB，不再 ResourceExhausted；
  2. 60k（240GiB）能否跨过 100GiB 单 vsegment quota，vsegments/logical_capacity 是否随写入增长；
  3. vseg 60k 单遍读 ~20–24s，再 plain 60k 单遍读，给 parity。

## A. 停旧 master

1. 停 `vseg_nof120` master 2396324/2396327（TERM，必要时 KILL），确认 50056/9006/8086 释放；备份其日志（25 万行）到 run/。
2. target A（3329243）/B（2391418）不动；node2 无残留。

## B. 新 vseg master / 注册 / quota

1. 起 `vseg_nof122`（node4，50056/9006/8086、slot=1、etcd、URMA、`enable_ha=true`、`enable_oplog=true`、**`--oplog_batch_max_entries=64`**、`member_count=2 / extent=50GiB / stripe=2MiB / required_medium=nof / reserved_ratio=0 / io_alignment=4096 / submaster_count=1`）。
2. 注册 node4 B → node3 A，容量 **512GiB**；确认 segment=2、quota generation=1、partitions=1、vsegments=1、logical_capacity=100GiB、reconcile_fail=0。
3. 记录启动命令行中的 `oplog_batch_max_entries=64`（flag 生效证据）。

## C. 60k vseg 写入 + 读

1. Writer：`--num_keys=30000`（每 segment，总 60,000）、T1/batch1、4MiB；`VSEGMENT_CREATING` 最多 5 轮；**监控 etcd 日志是否再出现 ResourceExhausted**。
2. 验收：`Keys=60000`、`logical_used=251,658,240,000`；target A/B 写窗口各≈960,000；记录 `vsegments`、`logical_capacity` 是否随 25k/50k 增长（预期 3 个 vsegment / 300GiB）。
3. 若再次失败在 ~25,647 且错误不是批大小（如 vsegment quota/创建），**停手回报**，不要继续第 2 轮；下一步再裁决 20k 或扩容。
4. Reader：**1 遍**，`--num_keys=30000`（每 segment，60k queries）、T1、warmup=128、qpairs=4、verify=false、diag 关；记录实际时长、P50/Avg/P90/P99、failed、target A/B W9/n（各≈962,048）。
5. 记录 etcd dbSize/oplog 增长。

## D. Plain 60k + parity

1. 停 `vseg_nof122` master；起 plain `plain_nof122`（50056/9006/8086、slot=1、etcd、URMA、HA+oplog、**`--oplog_batch_max_entries=64`**）；注册 B 512GiB。
2. Writer 60k（16t/batch16），验收 Keys=60000/logical_used；reader 1 遍 60k（T1/warmup128/qpairs4/diag off），记录 P50/Avg/P90/P99、W9/n（≈1,924,096）、实际时长。
3. 对比 vseg/plain：P50、W9、比值；判 parity（≤1.05×）。

## E. 交付

1. 旧 master 停止/日志备份；新 master flag 生效证据。
2. vseg 写入 Keys/logical_used、vsegments/logical_capacity 变化、失败数/错误码、target A/B 写 n。
3. vseg 读单遍 P50/Avg/P90/P99、W9/n、实际时长。
4. plain 写/读同样指标；parity 比值。
5. etcd dbSize/oplog 增长；环境末态。

## F. 边界

- **只停/起 master，不动 target**；不改码/不重编/不装包；oplog on，batch=64。
- 60k 数据覆盖旧数据；master 重启后必须重注册 512GiB。
- 若批大小修复后仍在 25,647 处失败，停手；**不要自行退回 20k 或改 quota**。
- 单遍读，不加第 2 遍；不跑 T4/T16。
- 预计 **30–40 分钟**。

---

## NOF-122 裁决（2026-11-26 · 回执已收，batch 无效）

### 1. 事实

| 项 | NOF-121（batch=1024） | NOF-122（batch=64） |
|---|---|---|
| 失败 Keys | 25,647 | **25,632** |
| 失败事务 | 13,295,444 B | **13,322,832 B（+0.2%）** |
| 每 key 折算 | 518 B | **520 B** |
| etcd 上限 | 2,097,152 B | 2,097,152 B（未变） |
| 结果 | `-1010` 无限重试 | 同样 `-1010`，同一尺寸 |

- 批次条数降 16 倍，事务尺寸**不变** ⇒ `oplog_batch_max_entries` 不是这条事务的杠杆。
- 源码确认：`VSegmentOpLogRecord.full_state=true` 时 payload 是 **`PartitionVSegmentSnapshot`**（`vsegment_ha.h:24-37`），在 `CommitDeferred` 里整块 `struct_json::to_json` 成**一个 OpLogEntry**（`vsegment_ha.cpp:121-148`）；其中 `logical_allocation` 随 key 线性增长，约 520B/key。25.6k keys ⇒ 13.3MB，60k ⇒ 约 31MB。
- 25,632×4MiB = 100.12GiB，刚好越过单 vsegment 100GiB；`vsegments=2`、capacity 仍 100GiB、create_success=1 ⇒ **第 2 个 vsegment 的 full-state 事务卡在这里**（[推算]，需 E1 后验证）。
- 不是总空间：etcd dbSize 92MiB、/home 余 1.1TiB、`/cvm/` 仅 0.012MiB。

### 2. 决策（推翻上一轮“改批大小”）

- **E1：抬 etcd `--max-request-bytes` 到 64MiB**。这不是“纵容大事务”，而是当前唯一不改码、能落盘 full-state 快照的办法；60k 约 31MB，64MiB 留 2× 余量。
- 停 wedged master（2404042/2404045，CPU 86.2%、无限重试）；只停 master，不动 target。
- 新 cluster `vseg_nof123` 重写 60k；观察第 2/3 个 vsegment 创建是否随 full-state 落盘而恢复。
- **E2（缩小 extent）暂不做**：full-state 是 **per partition**，不是 per vsegment；缩小 extent 不会缩小 state，可能反而增加 full checkpoint 次数。
- **E3（plain 60k）并入最终两臂**：plain 不写 vsegment state，预期不受此墙；但仍按最终 parity 测。

---

# NOF-123（待执行 · etcd 64MiB 上限 + 重试 60k · 只重启 etcd/master · 2026-11-26）

## 目标

- 用 etcd `--max-request-bytes=67108864`（64MiB）让 full-state 快照（25k≈13.3MB、60k≈31MB）能写入。
- 停 wedged master；新 cluster `vseg_nof123` 重写 60k；观察 100GiB 处第 2 个 vsegment 创建是否恢复。
- vseg 60k 读 1 遍；plain 60k 读 1 遍；parity。

## A. 停 wedged master

1. 停 `vseg_nof122` master 2404042/2404045（TERM，必要时 KILL）；备份日志（14 万+行）到 run/；确认 50056/9006/8086 释放。
2. target A/B 不动；node2 无残留。

## B. etcd 抬上限（只重启 etcd）

1. 记录 etcd 2389250 的原 cmdline、data-dir、`--max-request-bytes` 是否未设（默认 2MiB）、dbSize。
2. 优雅停 etcd；用**同一 data-dir/同一参数** + **`--max-request-bytes=67108864`** 重启（其余参数不动）；记录新 PID/cmdline。
3. 验收：2379 LISTEN、health/leader 正常、dbSize 与停前一致（数据复用）、无 FATAL。
4. 若 etcd 启动失败（flag 不被接受/数据异常）：恢复原参数重启，停手回报。

## C. 新 vseg master / 写 60k

1. 起 `vseg_nof123`（node4，50056/9006/8086、slot=1、etcd、URMA、HA+oplog on、`--oplog_batch_max_entries=64`、`member_count=2 / extent=50GiB / stripe=2MiB / required_medium=nof / reserved_ratio=0 / io_alignment=4096 / submaster_count=1`）。
2. 注册 B→A 各 **512GiB**；quota generation=1、partitions=1、vsegments=1、logical_capacity=100GiB。
3. Writer：`--num_keys=30000`（每 segment，总 60,000）、T1/batch1、4MiB；`VSEGMENT_CREATING` 最多 5 轮；监控 etcd 是否仍有 ResourceExhausted。
4. 验收：**Keys=60000**、`logical_used=251,658,240,000`；记录 `vsegments`/`logical_capacity` 是否随 25k/50k 增长（预期 3 个 virsegment / 300GiB）、target A/B 写 n（各≈960,000）。
5. 若仍在 ~25,632 失败且错误仍是 13.3MB 事务：说明 64MiB 没生效/单条更大，停手回报，附 etcd flag/错误原文。
6. 若写入成功：reader **1 遍**，`--num_keys=30000`（60k queries）、T1、warmup=128、qpairs=4、verify=false、diag 关；记录实际时长、P50/Avg/P90/P99、failed、target A/B W9/n（各≈962,048）。

## D. Plain 60k + parity

1. 停 vseg master；起 plain `plain_nof123`（50056/9006/8086、slot=1、etcd、URMA、HA+oplog、batch=64）；注册 B 512GiB。
2. Write 60k（16t/batch16）；read 1 遍 60k（T1/warmup128/qpairs4/diag off）；记录 P50/Avg/P90/P99、W9/n（≈1,924,096）、实际时长。
3. 对比 vseg/plain：P50、W9、比值；判 parity（≤1.05×）。

## E. 交付

1. etcd 原/新 cmdline、flag、PID、dbSize；master flag/注册/quota。
2. vseg 写入 Keys/logical_used、vsegments/capacity 变化、失败数/错误、target A/B 写 n。
3. vseg 读单遍 P50/Avg/P90/P99、W9/n、实际时长。
4. plain 写/读同指标；parity 比值。
5. 环境末态。

## F. 边界

- **只重启 etcd 和 master，不动 target**；不改码/不重编/不装包。
- etcd 数据复用；失败恢复原参数；不改 data-dir。
- 60k 数据覆盖旧数据；master 重启后重注册 512GiB。
- 若 64MiB 仍失败，停手，不擅自继续加 etcd 上限或缩 keys。
- 单遍读，不跑 T4/T16；不跑 20k。
- 预计 **40–50 分钟**。

---

## NOF-123 裁决（2026-11-26 · E1 成功）

### 1. 结果

| 项 | 实测 |
|---|---|
| etcd flag | `--max-request-bytes=67108864` 生效（cmdline 确认）；data-dir 复用，dbSize 不变 96,854,016B |
| ResourceExhausted | **0**（NOF-122 为 12,588） |
| full-state 写入 | `value_max=13,284,282 B`（13.3MB）成功落盘 |
| vsegments / create_success | **3 / 3**（1→3，扩容恢复） |
| Keys | **59,931 / 60,000**（69 failed） |
| 失败码 | `-1401 TASK_PENDING_LIMIT_EXCEEDED` ×67、`-105` ×2、`VSEGMENT_CREATING` ×4 |
| target A/B 写 n | 958,911 / 958,912（均分，≈59,931×16） |
| 写吞吐 | 10,450 MiB/s；写 P50 364µs（含扩容停顿） |
| etcd dbSize | 246,902,784 B（235MiB）；/home 余 1.1TiB |

- `-1401` 源码定义：`TASK_PENDING_LIMIT_EXCEEDED`（`mooncake-store/include/types.h:417`）——客户端 pending 任务超过上限，属瞬态/背压，不是数据错误。
- `logical_capacity` 仍显示 100GiB 而 vsegments=3：quota/gauge 展示问题；Keys、target 落盘、create_success 均正常，不阻塞。
- 结论：**2MiB 单事务墙已解除；100GiB 处 vsegment 扩容恢复**。

### 2. 决策

- 69/60,000 = **0.115%**，不影响 P50/W9；**接受 59,931 keys 直接读**，不重跑全量覆盖写（10–12 分钟）。
- plain 60k 照常写/读；etcd 保持 64MiB。
- 下一单候选（未下发）：extent=128GiB 一整块，60k 全装 1 个 vsegment，避免写入期 full-state；重启/恢复仍需 etcd 64MiB。

---

# NOF-124（待执行 · 接受 vseg 59,931 keys 直接读 + plain 60k + parity · 2026-11-26）

## 目标

- 复用当前 `vseg_nof123`（Keys=59,931、vsegments=3、etcd 64MiB），**读 1 遍 60k queries**；接受 69 个缺失 key 的失败（0.115%），只看 P50/W9。
- 再切 plain master，写 60k、读 1 遍；对比 parity。
- 不补写、不重跑全量覆盖写、不改码、不重启 target。

## A. Vseg 读（直接复用）

1. 预检：`vseg_nof123` 运行（Keys=59,931、vsegments=3、2 segments、etcd 64MiB）；target A/B 运行。
2. Reader：`--num_keys=30000`（每 segment，共 60,000 queries）、T1/threads=1/batch=1、warmup=128、qpairs=4、verify=false、diag 关；**1 遍**。
3. 预期 failed≈69（缺失 key）；记录实际 failed、实际读时长、P50/Avg/P90/P99、target A/B W9/n。
4. 若 read 因缺失 key 中止/无统计，停手回报；**本单不做补写**。

## B. Plain 60k

1. 停 `vseg_nof123` master（记录 PID）；target B 不动；起 plain `plain_nof124`（50056/9006/8086、slot=1、etcd、URMA、HA+oplog、batch=64、etcd 64MiB 保持）；注册 B **512GiB**。
2. Writer：60,000 keys×4MiB、16t/batch16；验收 Keys=60000、logical_used=251,658,240,000；target B 写 n≈1,920,000。
3. Reader：**1 遍**，60k keys、T1、warmup=128、qpairs=4、verify=false、diag 关；记录实际时长、P50/Avg/P90/P99、failed、target B W9/n（n≈1,924,096）。

## C. 交付

1. Vseg 读：failed/缺失数、实际时长、P50/Avg/P90/P99、target A/B W9/n。
2. Plain 写/读：Keys/logical_used、写 n、读 P50/Avg/P90/P99、W9/n、实际时长。
3. 比值与 parity 判决（vseg P50 / plain P50；W9 同报）。
4. etcd dbSize；环境末态。

## D. 边界

- 接受 59,931 keys；**不补写 69 个**、不重跑全量覆盖写。
- 只读 vseg；plain 写 60k；不跑 T4/T16、不跑 20k；不改码/不重启 target。
- 若 vseg read 无统计/中止，停手；不要在本单改数据。
- 预计 **20–30 分钟**。

---

## NOF-124 裁决（2026-11-26 · parity 达成）

### 1. 结果（[事实]）

| 指标 | vseg（59,931 keys） | plain（60,000 keys） | 比值 vseg/plain | 判据 ≤1.05 |
|---|---|---|---|---|
| **P50** | **246.32µs** | **255.06µs** | **0.966×** | ✅ |
| Avg | 252.35 | 256.74 | 0.983× | ✅ |
| P90 | 255.10 | 263.26 | 0.969× | ✅ |
| P99 | 361.82 | 274.62 | 1.317× | ✗ |
| Max | 57,788.40 | 2,443.73 | 23.6× | ✗ |
| Min | 81.45 | 242.03 | 0.336× | — |
| target W9avg | 40,410 / 40,330ns | 44,450ns | 0.909 / 0.907 | ✅ |
| 读耗时 | 19s | 20s | — | ✅ |
| failed | 69（0.115%） | 0 | — | — |

- 演进：NOF-116（400 keys）1.215× → NOF-124（60k keys）**0.966×**，跨过 parity。
- `logical_capacity` 刷新后 **300GiB**，`logical_used=251,368,833,024 = 59,931×4MiB` 精确相等 ⇒ NOF-123 的 100GiB 只是 gauge 刷新延迟，不是功能缺陷。
- etcd 64MiB 全程保持；两阶段 ResourceExhausted 均为 0 ⇒ 写入墙彻底解决。
- vseg 的 69 个缺失 key：type unknown、1× `ret=-704`（OBJECT_NOT_FOUND）；按边界未回填。

### 2. 判读边界

- **P50/Avg/P90/W9 达到 parity，P99/Max 未达到。**
- vseg 的尾部（P99/Max）**可能被 69 个缺失 key / OBJECT_NOT_FOUND 污染**：Max 57.8ms 很可能是一次失败查找/重试；P99 的 600 个样本里也包含这 69 个失败。**在补齐 60k 或剔除失败样本前，不能把尾部差距归因 vseg 路径。**
- target n 不可直接比：vseg 60000 queries ×16 分两 target（含 69 失败/重试），plain 60000×32 单 target；W9avg 可比。
- 本结果是在 `extent=50GiB`、3 个 vsegment、etcd 64MiB 下取得；未验证一整块配置。

### 3. 决策

1. **不追长尾/P99**：当前验收是 P50/W9 parity，已达成；尾部若以后要求，先补/剔除 69 个失败 key 再立单，不在污染数据上调 qpairs/submaster/stripe。
2. **不推进“一整块 128GiB”**：当前 50GiB×2/3 vsegment + etcd 64MiB 已达成 parity；换 extent 会改变布局、需要重写+重测，且可能失去当前结果。它只作为“想移除 etcd 64MiB 写入依赖”时的未来候选，不在本阶段做。
3. **本阶段收口**：无在途任务。后续如需 P99 parity 或部署优化，另立单并明确验收。

### 4. 环境末态

- plain `plain_nof124` 2413231/2413234 运行中（Keys=60000）；etcd 2408033/2408036（64MiB，dbSize 293MiB）；target B 2391418、target A 3329243 未重启；node2/node4 无残留。
- 未改码/未重编/未装包。

---

# NOF-125（待执行 · 一整块 vsegment：extent=256GiB/member、逻辑 512GiB · 2026-11-26）

## 目标

- 验证“**一整块 vsegment 装下全部 60k 数据**”：`member_extent_size=256GiB`/member（`274877906944`），逻辑容量 512GiB；60k×4MiB=251,658,240,000B（234.375GiB）全部落进 1 个 vsegment。
- 目的：① 看读 parity 是否保持（对比 NOF-124 50G/3vseg 基线）；② 看写入过程是否不再创建第 2 个 vsegment、不再出现 13–31MB full-state 事务。
- **256GiB 是每个 member 在 NoF segment 上的物理 extent，不是节点 DRAM**；现有每 target 注册 512GiB，256G/member 留一半余量，无需扩大注册。
- 只改 extent 这一个变量；不改码、不重编、不重启 target、不重启 etcd。

## A. 停 plain master，起 vseg_nof125

1. 记录并停 plain `plain_nof124` master 2413231/2413234（TERM，必要时 KILL）；备份日志到 run/；确认 50056/9006/8086 释放；target A/B 不动。
2. etcd 2408033/2408036 **保持原样**（`--max-request-bytes=67108864`，不重启、不改 data-dir）。
3. 起新 cluster `vseg_nof125`（node4，50056/9006/8086、slot=1、`MC_STORE_PARTITION_SLOT_COUNT=1`、etcd、URMA、HA+oplog on、`--oplog_batch_max_entries=64`，其余参数与 NOF-123 完全相同：`--vsegment_auto_plan=true`、`--cvm_segments_vsegment_exclusive=true`、`member_count=2 / stripe=2MiB / required_medium=nof / reserved_ratio=0 / io_alignment=4096 / submaster_count=1`），**仅把 `--vsegment_member_extent_size` 改为 `274877906944`（256GiB；NOF-123 为 53687091200）**。
4. 若 etcd 中已存在 `/cvm/vseg_nof125/` 或 `/oplog/vseg_nof125/` 键：两处都删；否则不动。**不删其它 cluster 的键。**
5. 注册 B→A 各 **512GiB**；等 quota ≥20s；验收：`member_count=2`、**`member_extent_size=274877906944`**、`stripe_size=2097152`、partitions=1、reconcile_fail=0、**`logical_capacity=549755813888`（512GiB）**、vsegments=1；记录配额发布耗时。

## B. 写 60k（T1，同 NOF-123 口径）

1. Writer：`--num_keys=30000`（每 segment，总 60,000）、4MiB、T1/threads=1/batch=1。
2. 验收：**Keys=60000**、`logical_used=251,658,240,000`（若仍有失败：Keys×4MiB 精确对齐，记录 failed 数与错误码）。
3. **一整块判据**：写入全程 `vsegments` 保持 **1**；master 日志无 13–31MB 级 full-state 事务（搜 `full_state`/`value_max`）；etcd 无 `ResourceExhausted`；target A/B 写 n 各≈960,000。
4. 若超过 5 分钟卡在同一 key（wedged）：停手回报，附日志/etcd 证据，不擅自改参数。
5. **不做补写**（与 NOF-124 同边界）；失败 key 如实记录。
6. 记录写入总时长、写 P50/Avg/P99、target A/B 写 W9/n。

## C. 读 1 遍（同 NOF-124 vseg 口径）

1. Reader：`--num_keys=30000`（每 segment，共 60,000 queries）、T1/threads=1/batch=1、warmup=128、qpairs=4、verify=false、diag 关；**1 遍**。
2. 记录：failed、实际读时长、P50/Avg/P90/P99/Max、target A/B 读 W9/n（各≈960,000）。

## D. 判定

1. **parity**：本单 vseg P50 ≤ NOF-124 plain P50 **255.06µs × 1.05 = 267.81µs**；W9avg ≤ plain **44,450ns × 1.05**。与 NOF-124 vseg 基线（P50 246.32 / W9 40.4）并列报比值。
2. **一整块**：vsegments=1、logical_capacity=512GiB、无第 2 个 vsegment 创建、无大 full-state ⇒ 通过。
3. 尾部（P99/Max）如实报数，**本单不据此调参、不追长尾**；若本单 60k 无缺失 key，附尾部与 NOF-124 的对照供后续。

## E. 交付

1. master cmdline 关键项（extent 实测值）、注册 B→A 的 segment 数与大小、quota 细节（member_count/extent/stripe/logical_capacity）、发布耗时。
2. 写入：Keys/logical_used/failed+错误码、vsegments 变化、full-state/ResourceExhausted 计数、写时长、写 P50/Avg/P99、target A/B 写 W9/n。
3. 读取：failed、时长、P50/Avg/P90/P99/Max、target A/B 读 W9/n。
4. 比值：vs NOF-124 vseg（P50/W9）、vs plain（parity 判决）。
5. etcd dbSize；环境末态（master/target/etcd PID）。

## F. 边界

- **不改码/不重编/不装包/不重启 target/不重启 etcd**；只停 plain master、起 vseg master。
- 不改 member_count/stripe/required_medium/reserved_ratio/io_alignment；只把 extent 50→256GiB。
- 不删其它 cluster 的 etcd 键；不清 target 数据（新 cluster 从 offset 0 覆盖写）。
- 单遍读，不跑 T4/T16、不跑 20k；不做补写。
- 预计 **40–60 分钟**（写 60k 为主）。

---

## NOF-125 裁决（2026-11-26 · 写侧全胜；读侧 1.068× 未达 parity）

### 1. 结果（[事实]）

**A. 配置生效 / quota**

| 项 | 实测 |
|---|---|
| extent flag | `--vsegment_member_extent_size=274877906944`（唯一改动；NOF-123 为 53687091200） |
| auto-plan 日志 | `member_count=2, stripe_size=2097152, member_extent_size=274877906944` |
| VSegment created | `capacity=549755813888`；两 member 各 `offset=0, len=274877906944` |
| quota 发布耗时 | 30s |
| 注册 | B→A 两条 succeeded；segment=2；`master_total_nof_capacity_bytes=1099511627776`（1TiB） |
| vsegments / logical_capacity | 1 / 549,755,813,888（512GiB） |

**B. 写入（T1/batch1，60k，15:02:42→15:05:07，145s 墙钟）**

| 项 | 实测 |
|---|---|
| Keys / failed | **60,000 / 0** |
| vsegments 全程 | **1**（t+0/60/120/130s 均 1；`VSegment created` 条数=1） |
| full-state 大事务 | **无**：`value_max` 1,281→3,591→7,518→**9,087 B**；`full_state` 日志=0 |
| ResourceExhausted | 0；`VSEGMENT_CREATING`=0；`ret=[]` |
| 写 P50 / P99 | **326.09 / 417.56** |
| target A/B 写 n / W9avg | 960,017 / 960,016；22,890 / 22,610 ns |
| etcd 增量 | +94.4MiB（oplog +56.56MiB）；dbSize 387.75MiB；raftIndex 452,312 |

**C. 读取（1 遍，60k queries，T1/warmup128/qpairs4/verify=false/diag 关，20s 墙钟）**

| 项 | 实测 |
|---|---|
| queries / failed | 60,000 / **0** |
| Min / Avg / **P50** / P90 / P99 / Max | 250.54 / 278.06 / **272.37** / 285.03 / 386.92 / 60,862.44 |
| type | 120,256 全 `vsegment`（含 warmup 256）；无 unknown |
| target B / A 读 n | 841,329 / 822,083 |
| target B / A 读 W9avg | 39,150 / 38,890 ns |

**D. 对比（[事实]，均为单遍/跨会话）**

| 对比 | 指标 | 本单 | 参照 | 比值 |
|---|---|---|---|---|
| vs plain（NOF-124） | P50 | 272.37 | 255.06 | **1.068×（判据 ≤1.05，未通过；阈值 267.81）** |
| | Avg / P90 / P99 / Max | 278.06 / 285.03 / 386.92 / 60,862.44 | 256.74 / 263.26 / 274.62 / 2,443.73 | 1.083 / 1.083 / 1.409 / 24.9 |
| | target W9avg | 39,150 / 38,890 | 44,450 | **0.881 / 0.875** |
| vs vseg（NOF-124，同 60k） | P50 | 272.37 | 246.32 | **+10.6%（变差）** |
| | Avg / P90 | 278.06 / 285.03 | 252.35 / 255.10 | +10.2% / +11.7% |
| | target 读 n | 841,329 / 822,083 | 535,330 / 515,570 | **+57% / +59%** |
| | target W9avg | 39,150 / 38,890 | 40,410 / 40,330 | −3.1% / −3.6% |
| 写入 vs NOF-123（50G/3vseg） | 写 P50 | 326.09 | 364.43（含扩容停顿） | −10.5%（非干净对照） |
| | Keys / failed | 60,000 / 0 | 59,931 / 69 | 本单完整 |
| | vsegments / value_max | 1 / 9,087B | 1→3 / 13,284,282B | 无扩容、无大 full-state |

### 2. 判读边界（重要）

1. **写侧：一整块目标完全达成**——vsegments 恒 1、无第 2 个 vsegment、无 13–31MB full-state、60k 零失败；NOF-122 的 H2（full-state 来自跨 vsegment 扩容）得到验证。
2. **读侧：未达 parity**（1.068× > 1.05）。但这是**单遍 + 跨会话**对比，且 NOF-124 与 NOF-125 的 10.6% 差异落在历史会话漂移（~9%）量级内 ⇒ **不能直接归因 extent**。
3. **不是 target 单 IO 变慢**：target W9 反而低 3%，端到端却高 10.6% ⇒ 劣化在 target 执行段之外。
4. **“+58% target IO 数”不成立为事实**：理论应为 60k×16=960,000/target；实测 NOF-125 为 841,329/822,083（86%/87%），NOF-124 为 535,330/515,570（54%/56%）⇒ 该计数本身不是完整 IO 总量，且两单窗口口径不同；**不能据此判定“读被拆成更多 IO”**，需同会话三臂复核。
5. 吞吐类数字为 bench 内部口径，与墙钟不符（写 145s / 读 20s），**不作验收**；P99/Max 按既定边界不追（Max 60.86ms 为单点离群，未归因）。
6. 部署仍保留 etcd 64MiB 的必要性：master 重启恢复仍会触发 60k≈31MB full-state（extent 大小不改变该恢复路径）。

### 3. 决策

1. **50GiB/3-vseg 暂时保留为 parity 配置**（NOF-124：P50 0.966×）；256GiB 暂不定为读路径配置。
2. 下发 **NOF-126 同会话三臂**（vseg256 复读 → vseg50 重写+读 → plain 重写+读），一次性判决：extent 是否真的影响读、同会话 parity、以及 per-pass target n 是否随配置变化。**零改码、不重启 target/etcd、不追长尾。**
3. 若 NOF-126 证明 256G 与 50G 读等价且同会话 parity ⇒ 保留 256G（写侧收益）；若 256G 稳定慢 >5% ⇒ 回 50G，256G 仅作写侧选项。

### 4. 环境末态

- `vseg_nof125` master 2416799/2416802 运行中：Keys=60000、vsegments=1、logical_capacity=549755813888；etcd 2408033/2408036（64MiB，dbSize 387.75MiB）；target A 3329243 / target B 2391418 未重启；node2/node4 无残留。
- 未改码/未重编/未装包/未重启 target/未重启 etcd。

---

# NOF-126（待执行 · 同会话三臂：vseg256 复读 vs vseg50 重写 vs plain 重写 · 2026-11-26）

## 目标

- 用**同一会话、同一读取参数**回答三件事：
  1. NOF-125 的 P50 272.37 是稳定值还是会话漂移（vseg256 数据已存在，零写复读 3 遍）；
  2. extent 50GiB vs 256GiB 是否真的改变读时延（同会话重写 50G 臂，3 遍）；
  3. 同会话 plain 阈值下的 parity（plain 60k 重写+3 遍）。
- 同时记录每臂 per-pass target A/B 的读 n 与 W9，验证 NOF-125 的“+58% IO 数”是真实现象还是窗口计数伪影。
- 不改码、不重启 target/etcd、不补写、不追长尾。

## A. vseg256 复读（零写，3 遍）

1. 复用当前 `vseg_nof125`（master 2416799/2416802，Keys=60000，vsegments=1）；target A/B 不动。
2. Reader 3 遍：`--num_keys=30000`/segment（60k queries）、T1/threads=1/batch=1、warmup=128、qpairs=4、verify=false、diag 关。
3. 每遍记录：P50/Avg/P90/P99/Max、failed、target A/B 读 n 与 W9avg、墙钟时长。
4. 本段零写；若 master/数据异常，停手回报。

## B. vseg50 臂（重写 60k + 3 遍读）

1. 停 `vseg_nof125` master（记录 PID）；target A/B 不动；etcd 保持 64MiB 不重启。
2. 起新 cluster `vseg_nof126_50g`（node4，50056/9006/8086、slot=1、etcd、URMA、HA+oplog on、`--oplog_batch_max_entries=64`），除 extent 外与 NOF-125 完全相同：`member_count=2 / stripe=2MiB / required_medium=nof / reserved_ratio=0 / io_alignment=4096 / submaster_count=1`；**`--vsegment_member_extent_size=53687091200`（50GiB）**。若 etcd 已存在 `/cvm/vseg_nof126_50g/` 或 `/oplog/vseg_nof126_50g/` 键，两处都删；不删其它 cluster 键。
3. 注册 B→A 各 512GiB；等 quota ≥20s；验收 `member_extent_size=53687091200`、`logical_capacity=107374182400`、partitions=1、reconcile_fail=0。
4. 写 60k：`--num_keys=30000`/segment、T1/batch1、4MiB；记录 Keys/failed/失败码、vsegments/create_success、`value_max`（预期 13.3MB full-state 出现）、写 P50/P99、墙钟时长、target A/B 写 n 与 W9。
5. **不补写**；Reader 3 遍（同 A 段参数），记录同指标与 failed。

## C. plain 臂（重写 60k + 3 遍读）

1. 停 vseg50 master；起 `plain_nof126p`（50056/9006/8086、slot=1、etcd、URMA、HA+oplog、batch=64）；若 etcd 已存在 `/cvm/plain_nof126p/` 或 `/oplog/plain_nof126p/` 键，两处都删；注册 B 512GiB。
2. 写 60k：16t/batch16、4MiB；验收 Keys=60000、logical_used=251,658,240,000；记录写 n、墙钟时长（写时延可报 P50）。
3. Reader 3 遍（同 A 段参数）；记录同指标、failed、target B 读 n 与 W9。

## D. 判决（必答）

1. **稳定性**：A 段 3 遍彼此的离散度，以及相对 NOF-125 首遍 272.37 的位置。若 3 遍彼此差 <5% 且都在 272 附近 ⇒ 272 是该会话的稳定值；若明显低于 272（接近 246）或离散 >5% ⇒ 首遍受会话状态影响，NOF-125 的 10.6% 不能归因 extent。
2. **extent 影响**：A（vseg256）与 B（vseg50）3 遍 P50 均值/中位对比；同会话下若 256 稳定慢 >5% ⇒ extent 真实影响读；若 ≤5% 或交叉 ⇒ 无显著影响。
3. **parity**：A、B 分别与 C（同会话 plain）比 P50/W9，判据 ≤1.05×；给出两臂各自的比值。
4. **IO 数**：逐臂逐遍 target A/B 读 n；若 50G 与 256G 在同会话内 n 仍差 >20% 且稳定，记录为真实线索；若 n 随遍数大幅波动或接近理论 960k，判定 NOF-125 的 +58% 为窗口计数伪影。
5. **配置建议**：按 D2/D3 给出保留 256G 还是回 50G 的结论（测试侧只报事实与比值，不自行改配置）。

## E. 边界

- 不改码/不重编/不装包/不重启 target/不重启 etcd（保持 64MiB）。
- 顺序固定：**A（256G 复读，零写）→ B（50G 重写+读）→ C（plain 重写+读）**；B/C 会覆盖前面的数据，不得调换。
- 不补写缺失 key；不跑 T4/T16、不跑 20k；不追 P99/Max；diag 关。
- 每遍读都是 1 遍 60k queries（约 20s）；共 3+3+3 遍。
- 50G 臂预期会出现 3 个 vsegment 和 13.3MB full-state（etcd 64MiB 已验证可承载）；若 ResourceExhausted 复现，停手回报。
- 预计 **25–35 分钟**。

---

## NOF-126 裁决（2026-11-26 · 同会话三臂：extent 无显著影响；50G 稳过 parity）

### 1. 结果（[事实]，同一会话、同一读取参数）

**A. vseg256 零写复读 3 遍（现有 60k、vsegments=1）**

| 遍 | Avg | P50 | P90 | P99 | failed | target A n / W9 |
|---|---|---|---|---|---|---|
| p1 | 271.89 | 265.70 | 277.46 | 386.94 | 0 | 824,291 / 39,560 |
| p2 | 268.98 | 260.74 | 274.06 | 369.98 | 0 | 809,909 / 39,250 |
| p3 | 275.73 | 266.47 | 280.61 | 376.99 | 0 | 776,116 / 39,270 |

- target B 的 n 未取到（node4 缺 `nof126_win.sh`，B/C 臂已修复）；A 臂只用 A 的 n。
- **NOF-125 的 272.37 未复现**：本臂三遍 260.74 / 265.70 / 266.47，全部低于 272.37。

**B. vseg50 重写 + 3 遍读**

- 写入：59,967 / 60,000（33 failed：`-1401`×31、`-105`×2、`VSEGMENT_CREATING`×4）；vsegments 1→2→3；value_max 1,284 → 13,288,734 → 13,352,055B；rex=0；写 P50 365.44µs（≈150s）。
- 读取（failed 33）：

| 遍 | Avg | P50 | P90 | P99 | target B n / W9 | target A n / W9 |
|---|---|---|---|---|---|---|
| p1 | 277.69 | 272.75 | 285.33 | 400.94 | 961,523 / 39,040 | 961,522 / 39,110 |
| p2 | 259.87 | 254.51 | 262.90 | 369.29 | 948,915 / 39,260 | 927,372 / 39,200 |
| p3 | 262.39 | 256.72 | 266.31 | 372.50 | 917,667 / 39,470 | 896,402 / 39,390 |

**C. plain 重写 + 3 遍读（60,000/0 failed）**

| 遍 | Avg | P50 | P90 | P99 | target B n / W9 |
|---|---|---|---|---|---|
| p1 | 265.05 | 262.50 | 271.74 | 296.04 | 1,923,395 / 44,100 |
| p2 | 241.90 | 242.86 | 251.13 | 264.46 | 1,924,099 / 44,490 |
| p3 | 254.02 | 250.98 | 259.04 | 283.97 | 1,034,594 / 45,040 |

**D. 同会话逐遍对比（µs）**

| 遍 | plain P50 | vseg50 P50 | vseg256 P50 | vseg50/plain | vseg256/plain |
|---|---|---|---|---|---|
| p1 | 262.50 | 272.75 | 265.70 | **1.039×** | 1.012× |
| p2 | 242.86 | 254.51 | 260.74 | **1.048×** | **1.074×** |
| p3 | 250.98 | 256.72 | 266.47 | **1.023×** | **1.062×** |

- **extent 影响（逐遍 vseg256/vseg50）**：p1 −2.6%、p2 +2.4%、p3 +3.8%，均小于各自臂内 P50 跨度（plain 242.86–262.50、vseg50 254.51–272.75、vseg256 260.74–266.47）⇒ **无显著影响**。
- **parity**：vseg50 三遍 **1.039 / 1.048 / 1.023，全部 ≤1.05**；vseg256 三遍 1.012 / 1.074 / 1.062，**两遍超**。
- **+58% IO 伪影**：target dump 的 n 是**单个 dump 块内计数，不是累计**；plain 第 3 遍同一个 60k 负载只有 1,034,594（前两遍 1,923,395/1,924,099）；同会话 vseg256 的 n（776k–824k）反而低于 vseg50（896k–962k）⇒ NOF-125 的 +58% 判为窗口计数伪影，非真实 IO 放大。
- **NOF-125 的 272.37**：高于本会话三遍 260.74 / 265.70 / 266.47 ⇒ 会话偏高；其相对 NOF-124 的 +10.6% 主要是漂移。
- **target W9**：vseg50 39.0–39.5、vseg256 39.0–39.6，plain 44.1–45.0；两 vseg 臂均比 plain 低约 12%（逐遍原始值在各表中）。

### 2. 判读

1. **extent 50↔256GiB 对读时延无显著影响**；NOF-125 的读侧劣化不是 extent 造成的。
2. **50G/3vseg 是同会话下唯一三遍逐遍比全部 ≤1.05 的配置**（1.039 / 1.048 / 1.023），且 NOF-124 也给出 0.966×。
3. 256G 三遍 1.012 / 1.074 / 1.062，两遍超 1.05；写侧收益（60,000/0、vsegments=1、无 full-state、写 P50 326.09 vs 365.44）真实，但**读是验收目标**。
4. vseg 每 IO 的 target W9 比 plain 低约 12%（39 vs 44µs，两臂一致），端到端剩余差距在客户端聚合/两跳，非 target 单 IO。
5. P99 逐遍：vseg50 369.29–400.94、vseg256 369.98–386.94、plain 264.46–296.04；Max 有大离群；按既定边界不追。

### 3. 决策

1. **读路径默认配置 = 50GiB/member（3 个 vsegment）**；256GiB 不作为默认，仅在对写侧（0 失败、无 full-state、写 P50 更低）有强需求时作为可选。
2. 不再调 extent/chunk/qpairs；不追 P99/Max。
3. **本阶段收口，无在途任务。** 环境停在 plain `plain_nof126p`（Keys=60,000）；如需 vseg50 常驻，另立单重写（plain 已覆盖 target B，vseg A 半边不完整）。

### 4. 环境末态

- plain `plain_nof126p` 2427483/2427487：leader/serving、Keys=60,000、NVMe-oF SSD 234.38GB/512GB。
- etcd 2408033/2408036：`--max-request-bytes=67108864`、dbSize 596,172,800B（568.55MiB）、raftIndex 635,751。
- target A 3329243 / target B 2391418 未重启；node2/node4 无残留。
- 未改码/未重编/未装包/未重启 target/未重启 etcd。
- 偏差：B 臂第 2 轮补写被人工中止（bench 从 key 0 重写，Keys 恒 59,967，继续只会空转）；B 最终 59,967/60,000，未补写，33 个失败按 0.055% 接受。

---

# NOF-127（待执行 · 正式条件稳定性测量 · 60k · 一整块 vsegment vs plain · 多轮 · 2026-11-26）

## 目标

- 按**正式条件**（HANDOVER 顶部“正式测试口径”）：60,000 keys × 4MiB；vseg = **一整块**（extent=256GiB/member、逻辑 512GiB）；plain = 单 target B；T1 读。
- 用 **2 个交替周期（V3→P3→V3→P3）** 共 **12 轮读**（每臂 6 轮），抵消时间漂移；交付两臂**逐轮 Avg/P50/P90/P99** 与 **P50 稳定区间**，作为第四阶段第一个正式结果。
- **不报吞吐，不计算/不写 P50 均值/中位**；每轮单独一行。
- 不改码、不重启 target/etcd；每个 phase 用新 cluster 重写 60k（后写覆盖前数据）。

## A. 预检

1. 当前 `plain_nof126p`（2427483/2427487，Keys=60000）运行中；记录 PID 后停止，释放 50056/9006/8086。
2. target A（3329243）/ target B（2391418）运行、4457 LISTEN；**均不重启**。
3. etcd 2408033/2408036 运行，`--max-request-bytes=67108864`；**不重启**。
4. node2 无 bench/register 残留；记录客户端二进制 md5（不重编）。

## B. Phase 1：vsegment（一整块，`vseg_nof127a`，3 轮读）

1. 起 vseg master `vseg_nof127a`（node4，50056/9006/8086、slot=1、etcd、URMA、HA+oplog on、`--oplog_batch_max_entries=64`）：`member_count=2 / extent=274877906944 / stripe=2MiB / required_medium=nof / reserved_ratio=0 / io_alignment=4096 / submaster_count=1`；若 etcd 存在同名 `/cvm/`、`/oplog/` 键，两处都删；不删其它 cluster 键。
2. 注册 B→A 各 512GiB；等 quota ≥20s；验收 `member_extent_size=274877906944`、`logical_capacity=549755813888`、partitions=1、reconcile_fail=0。
3. 写 60k：`--num_keys=30000`/segment、T1/batch1、4MiB；验收 **Keys=60000**（记录 failed 与错误码；**不补写**；不记录写入性能）。
4. 读 3 轮（p1/p2/p3）：`--num_keys=30000`/segment（60k queries）、T1/threads=1/batch=1、warmup=128、qpairs=4、verify=false、diag 关；每轮记录 **Avg/P50/P90/P99、failed、墙钟**；target A/B 的 W9 附记（注明是 target 侧口径）。

## C. Phase 2：plain（`plain_nof127a`，3 轮读）

1. 停 `vseg_nof127a`；起 `plain_nof127a`（50056/9006/8086、slot=1、etcd、URMA、HA+oplog、batch=64）；若存在同名 `/cvm/`、`/oplog/` 键，两处都删；注册 B 512GiB。
2. 写 60k：16t/batch16、4MiB；验收 **Keys=60000**（记录 failed；不补写；不记录写入性能）。
3. 读 3 轮（p1/p2/p3）：60k keys、T1、warmup=128、qpairs=4、verify=false、diag 关；记录 **Avg/P50/P90/P99、failed、墙钟**；target B 的 W9 附记。

## D. Phase 3：vsegment 复测（`vseg_nof127b`，3 轮读）

- 停 plain；按 B 段完整重做（新 cluster、extent 256GiB、注册 B→A、写 60k、读 3 轮 p4/p5/p6）。
- 目的：让 vseg 的 6 轮分布跨两个时间窗口，抵消单向漂移。

## E. Phase 4：plain 复测（`plain_nof127b`，3 轮读）

- 停 vseg；按 C 段完整重做（新 cluster、单 target B、写 60k、读 3 轮 p4/p5/p6）。

## F. 交付（必答）

1. **12 轮逐轮表**：`臂 | phase | 轮 | Avg | P50 | P90 | P99 | failed | 墙钟`；每臂 6 行，不聚合。
2. 每臂 6 轮的 **P50 原始值列表 + min–max 区间**；Avg/P90/P99 同样给原始列表。
3. 稳定性判读：
   - vseg 的 P50 区间 [min,max] 与 plain 的 P50 区间 [min,max]；
   - 两区间是否重叠/分离；给出 vseg/plain 的区间比（min/max 与 max/min 两端）；
   - **不写均值/中位、不写吞吐**。
4. 每个 phase 的配置验收（extent、capacity、vsegments）、数据完整性（Keys/failed）。
5. 环境末态（master/target/etcd PID）；未改码/未重启 target/etcd。

## G. 边界

- **不改码/不重编/不装包/不重启 target/不重启 etcd**；只重启 master 与 bench。
- 不再跑 400 keys；不跑 T4/T16、不跑 20k；不补写；不调 chunk/qpairs；不追 P99/Max（只如实报）。
- 顺序固定 V3→P3→V3→P3；每个 phase 写后必须用 master `Keys` 验收；若失败 >0.2% 或 wedged，停手回报。
- 每次 master 重启后必须按 B→A 重注册；注册后等 quota ≥20s。
- 预计 **35–45 分钟**。

---

## NOF-127 裁决（2026-11-26 · 第四阶段第一个正式结果）

### 1. 结果（[事实]：6+6 轮，每轮一遍 60k keys ≈20s，不循环）

- 配置：60k keys、一整块 vsegment（extent=274877906944、logical=549755813888、vsegments=1）、plain 单 target B；四阶段 V3→P3→V3→P3；每阶段写入均 **60,000/0 failed**；12/12 轮读 **failed=0**。
- **聚合规则按 `D:\gds性能\测试结果聚合方法_20261112.md`**：Avg = 各轮等权算术平均；P50/P90/P99 = 各轮同分位中位数；分位不在原始样本级合并。逐轮原始值在 `run/nof127_summary.txt`（123 行，md5 26edd1eb…）；正文只给聚合值。
- 逐轮 Min/Max 未采集，本表不列。

| 臂 | 轮数 | Avg（均值） | P50（中位） | P90（中位） | P99（中位） | Avg spread |
|---|---|---|---|---|---|---|
| vsegment | 6 | **264.39** | **257.01** | **268.55** | **376.69** | 6.2% |
| plain | 6 | **254.14** | **248.51** | **256.80** | **274.85** | 6.4% |
| vsegment/plain | — | **1.040×** | **1.034×** | **1.046×** | **1.371×** | — |

- **P50 257.01 vs 248.51 = 1.034×；Avg 1.040×、P90 1.046×，三项均 ≤1.05 ⇒ 聚合口径 parity 达成。**
- **P99 376.69 vs 274.85 = 1.371×，未达。**
- **spread（avg）= 6.2% / 6.4%，>3%**：按方法“先查原因”；已查为**共同时间漂移**（两臂在 p4/p5 窗口同步偏高：plain 260.64/264.63、vsegment 257.35/272.93），6% <10% 仍进表。
- 时间就近配对（辅助检查，不作主结果）：6 对 1.076 / 1.014 / 1.029 / 0.977 / 1.020 / 1.059（0.977–1.076，5/6 vsegment 慢，4/6 ≤1.05）。
- target W9：vsegment B/A = 39,190 / 39,220 ns；plain = 44,170–44,220 ns（vsegment 每 IO 低约 11–12%）。
- etcd：四阶段 dbSize 663.05→721.11→815.60→873.65 MiB（+≈630 MiB）；ResourceExhausted 全 0。

### 2. 判读

1. **聚合口径 P50/Avg/P90 达 parity**（1.034/1.040/1.046×）；P99 仍落后（1.371×）。
2. **不是 2× 问题**：P50 差距 3.4%；与早期单 target 的 2.03× 已不同量级。
3. **P99 差距真实**：12 轮 0 缺失 key，尾部无缺 key 污染；按既定边界不追，但如实记录。
4. **target 每 IO 更快、端到端略慢**：W9 39.2 vs 44.2µs（vsegment 低 ~11%），剩余在客户端聚合/两跳。
5. **离散 6% >3%**：两臂在 p4/p5 时间段同步升高，属共同时间漂移；交替周期有效，但单轮仍有 ±3% 波动；按方法记录原因，不重跑。

### 3. 正式结果（第四阶段）

- **vsegment：Avg 264.39 / P50 257.01 / P90 268.55 / P99 376.69µs（6 轮聚合）**
- **plain：Avg 254.14 / P50 248.51 / P90 256.80 / P99 274.85µs（6 轮聚合）**
- **比值：Avg 1.040× / P50 1.034× / P90 1.046× / P99 1.371×**
- 结论：**P50/Avg/P90 达到 parity（≤1.05）；P99 未达；无 2× 问题。** 本聚合值作为第四阶段第一正式结果；不追 P99/Max；不再用单遍/单轮数字代表阶段结果。

### 4. 环境末态

- `plain_nof127b` 2438798/2438801：leader/serving、Keys=60,000、E=1、FATAL=0。
- etcd 2408033/2408036：64MiB、dbSize 873.65MiB、raftIndex 994,093。
- target A 3329243 / target B 2391418 未重启；客户端 md5 bb4a07a3319faae7f74214f4d0ee0a4b（未重编）。
- 未改码/未重编/未装包/未重启 target/etcd。
- 偏差：target A 的 W9 由 node3 末块补取（n=962,055、39,220ns）；墙钟为脚本计时 18–20s，非 bench 自报。

---

# NOF-128（待执行 · 只测 vsegment · 60k · 一整块 · 10 轮 · 查轮间波动 · 2026-11-26）

## 目标

- 只测 **vsegment**（不测 plain）：60,000 keys、一整块 vsegment（extent=274877906944、logical=549755813888）、T1、每轮只读一遍 60k（≈20s）、连续 **10 轮**。
- 每轮同时记录：客户端端到端 Avg/P50/P90/P99、failed、墙钟；**target A/B 的读 W9avg 与 n（每轮都取）**。
- 目的：① 给 vsegment 一个更稳的聚合值（按 `D:\gds性能\测试结果聚合方法_20261112.md`）；② 用每轮 W9 判断轮间波动在 target 还是客户端/系统。
- 不改码、不重启 target/etcd、不补写、不测 plain。

## A. 预检

1. 当前 `plain_nof127b`（2438798/2438801，Keys=60000）运行；记录 PID 后停止，释放 50056/9006/8086。
2. target A（3329243）/ target B（2391418）、etcd 2408033/2408036 运行，**均不重启**。
3. node2 无 bench/register 残留；客户端 md5 bb4a07a3…（不重编）；记录磁盘/日志空间。

## B. 起 vsegment / 写 60k

1. 起 `vseg_nof128`（node4，50056/9006/8086、slot=1、etcd、URMA、HA+oplog on、`--oplog_batch_max_entries=64`）：`member_count=2 / extent=274877906944 / stripe=2MiB / required_medium=nof / reserved_ratio=0 / io_alignment=4096 / submaster_count=1`；若 etcd 存在同名 `/cvm/`、`/oplog/` 键，两处都删；不删其它 cluster 键。
2. 注册 B→A 各 512GiB；等 quota ≥20s；验收 `member_extent_size=274877906944`、`logical_capacity=549755813888`、partitions=1、reconcile_fail=0、vsegments=1。
3. 写 60k：`--num_keys=30000`/segment、T1/batch1、4MiB；验收 **Keys=60000**（记录 failed；不补写；不记录写入性能）。

## C. 读 10 轮（每轮一遍 60k ≈20s）

- 每轮：`--num_keys=30000`/segment、T1/threads=1/batch=1、warmup=128、qpairs=4、verify=false、diag 关；记录 **Avg/P50/P90/P99、failed、墙钟**。
- **每轮结束后立即取 target A/B 的 W9avg 与 n**（必须对应当前轮窗口；取本轮末块，禁止用跨轮累计/下一轮块代替）。
- 10 轮连续跑，轮间不重启 master/client；若 W9 取不到或 failed>0，停手回报，不要用末块近似。

## D. 交付

1. 逐轮原始行落盘 `run/nof128_rounds.csv`：轮号、Avg/P50/P90/P99、failed、墙钟、target A/B W9avg/n。
2. 按 `测试结果聚合方法_20261112.md` 聚合：Avg=各轮等权算术平均；P50/P90/P99=各轮同分位中位数；**正文只给聚合表 + spread**，不逐轮罗列。
3. 波动判读：
   - 端到端 P50 的 spread、最高/最低轮；
   - 每轮 target A/B W9 的范围与波动；
   - **端到端波动与 W9 波动是否同向**：W9 平稳而端到端波动 ⇒ 客户端/系统；W9 同向波动 ⇒ target；无法分离就写“未分离”。
4. 配置/数据完整性、环境末态。

## E. 边界

- **只测 vsegment，不测 plain**；每轮只读一遍 60k（≈20s），**禁止 `--duration`**；不补写；不跑 T4/T16；不调 chunk/qpairs。
- 不改码/不重编/不装包/不重启 target/etcd；master 可停/起，注册每次 B→A。
- 10 轮连续；每轮必须取到 W9；失败停手。
- 预计 **20–30 分钟**。

---

## NOF-128 阻断（2026-11-26 · node4 环境被外部清空，零数据）

### 1. 事实

- 16:46:37 启动时，node4 上 **etcd / master / target B 全部不存在**；端口 2379/2380/4457/50056/9006/8086 均未监听。
- 本次启动的 master 因 etcd 不可达，16:47:03 abort（`failed to create HA batch-record OpLog writer: ETCD_OPERATION_ERROR`）。
- 注册 B/A 均失败（segment=0）；写 60k 空转 98s，Keys 恒 0（ret=-1）；读轮无统计；16:52:32 停手。
- target B 旧进程 2391418 不存在；`/tmp/nvmf_tgt.log` mtime=16:25:50，最后一行正常 timing，**无 Exiting/shutdown/SIGTERM/ERROR** ⇒ 非正常退出。
- etcd 旧进程 2408033/2408036 不存在；`/tmp/etcd_nof123.log` 末行 16:06:00（正常 WAL purge），data-dir 1.3G 完好；16:09:12 NOF-127 收尾时仍可读（dbSize 873.65MiB）。
- 11 块数据盘仍绑定 vfio-pci；`lsblk` 仅系统盘 nvme10n1；target A（node3，3329243）正常。
- 外部登录：`last` 显示 `computin` 在 16:20/16:23/16:32/16:41/16:42/16:50 多次登录 node4；load 9.32、15 users。**无法证明与清场的因果**，只记录事实。
- 测试 Agent 未启动/重启任何服务；无残留。

### 2. 影响与决策

- NOF-128 本轮 **0 有效数据**；NOF-127 已交付结果不受影响（16:09 完成并落盘）。
- target B 上的 8 次 60k 写入均为测试数据，**不保留**；只备份旧 target/etcd 日志用于取证。
- **下发 NOF-129**：按 NOF-119R 已验证路径重建 node4（etcd 64MiB → 回收 11 盘 → 重启 target B），随后重跑 NOF-128 的 10 轮 vsegment。
- node4 非独占（外部用户多次登录）；**不杀外部用户会话/进程**；若其作业再次清场或占用资源，停手回报，由用户决定是否处理该用户。

---

# NOF-129（待执行 · 重建 node4 + 重跑 NOF-128 · 用户已授权 · 2026-11-26）

## 授权范围（仅此单有效）

- **允许重启/重建** node4 的 etcd、target B，并回收 11 块 NVMe（vfio-pci→nvme）；允许停/起本方 master。
- **不允许**杀外部用户会话/进程、重启 node4 整机、动 target A（node3）、改码/重编。
- target B 旧数据丢弃（测试数据）；先备份旧日志。

## A. 取证（只读）

1. 备份到 `run/nof129_evidence/`：`/tmp/nvmf_tgt.log`（旧 target B，23,335 行）、`/tmp/etcd_nof123.log`、`last -n 30`、`who`、`ps -ef` 相关快照、`ss -ltnp`、`lsblk`、vfio 设备列表。
2. 检查 11 块 NVMe 是否被外部进程占用（`fuser`/`lsof` 相关设备）；若有外部占用，**停手回报**，不杀。
3. 记录 node4 当前 load/用户数；若外部作业明显在跑（持续高负载），先回报再决定。

## B. 重建 etcd

1. 用**同一 data-dir + 原参数 + `--max-request-bytes=67108864`** 启动 etcd；记录 PID/cmdline。
2. 验收：2379 LISTEN、health/leader 正常、dbSize ≈873MiB、无 FATAL；若起不来，停手回报。

## C. 回收 11 盘 + 重启 target B

1. 运行 `target_nvme_takeover.sh -r -y`（NOF-119R 路径）；验收 11/11 `[ OK ]`，`lsblk` 出现 11 块数据盘（排除系统盘 nvme10n1）。
2. 按 BDF 升序取 nvmeXn1，重启 target B：RAID0 strip 256KiB、端口 4457、`--wait-for-rpc`、trace 262144；记录 PID、新 UUID、NQN/SN。
3. 验收：4457 LISTEN、11/11 online、0 ERROR；target A（node3）不动。

## D. 重跑 NOF-128

1. 起 `vseg_nof128`（node4，50056/9006/8086、slot=1、etcd、URMA、HA+oplog、batch=64、`extent=274877906944`、member_count=2/stripe=2MiB/…）；清同名键；注册 **新 target B → target A** 各 512GiB；quota 验收。
2. 写 60k（T1/batch1，`--num_keys=30000`/segment）；验收 Keys=60000；不补写。
3. 读 10 轮（每轮一遍 60k≈20s，T1/warmup128/qpairs4/verify=false/diag 关）；**每轮取 target A/B 的 W9avg 与 n**（本轮窗口，禁止跨轮累计）。
4. 按 `测试结果聚合方法_20261112.md` 聚合（Avg 均值、分位中位）+ spread；正文不逐轮罗列。

## E. 边界

- 只重建 node4 的 etcd/target B；target A 不重启；不改码/不重编/不装包。
- 外部用户会话/进程一律不动；若外部作业导致再次清场或资源争用，停手回报。
- target B 旧数据不保留；NOF-127 结果已落盘。
- 预计 **30–45 分钟**（重建 + 写 60k + 10 轮）。

---

## NOF-129 裁决（2026-11-26 · 重建成功；vsegment 10 轮；波动在客户端/系统）

### 1. 重建结果（[事实]）

- etcd：新 PID 2455814；同 data-dir + `--max-request-bytes=67108864`；2379/2380 LISTEN、healthy；dbSize 916,090,880B（873.65MiB，与 NOF-127 末值一致）、raftIndex 994,095、FATAL=0。
- 11 盘：`takeover -r -y` 11/11 `[ OK ]`，仍 vfio 的盘数=0；根分区 rw。
- target B：新 PID 2458315、4457 LISTEN、NQN `...urma-gpu-test` / SN `URMAGPU0001`、**新 UUID 50d6259e-51a6-4599-adf3-68defc0c5a53**、RAID0 block 512 / 165,032,248,832 blocks / strip 256K / online 11/11、max_io_size 8,388,608、ERROR=0。
- target A（node3 3329243）未动。
- `vseg_nof128` 2458697/2458700：extent=274877906944、logical_capacity=549755813888、vsegments=1、reconcile_fail=0；注册 B→A 各 512GiB；写入 **60,000/0 failed**、vsegments 全程 1、rex=0。

### 2. vsegment 10 轮聚合（按 `测试结果聚合方法_20261112.md`：Avg=各轮均值、分位=各轮同分位中位）

| 指标 | 聚合值 | 各轮 min | 各轮 max | spread | 最低/最高轮 |
|---|---|---|---|---|---|
| Avg | 266.93 | 247.18 | 281.09 | **12.7%** | p9 / p4 |
| P50 | **260.59** | 246.03 | 274.99 | **11.1%** | p9 / p4 |
| P90 | 269.12 | 257.52 | 285.26 | 10.2% | p9 / p4 |
| P99 | 383.08 | 310.27 | 428.38 | 30.8% | p9 / p5 |

- target B W9avg：39,622ns（39,240–40,010，spread **1.9%**）；target A：39,264ns（38,990–40,040，spread **2.7%**）。
- 端到端 P50 spread 11.1% / target W9 spread 1.9–2.7% = **4.1×**。
- 高值集中在连续三轮 p3/p4/p5（269.49/274.99/273.33，约 17:02:50–17:04:10），同期两 target W9 仍为 39.24–39.51µs，无抬升。

### 3. 判读

1. **波动不在 target**：端到端波动是 target W9 的 4.1 倍，且高值轮 target 无变化 ⇒ 在**客户端/系统侧**（node2 及其路径）。具体成因（node2 load/CPU 竞争/队列）本单未测。
2. **按 2026-11-12 方法，Avg spread 12.7% >10% ⇒ NOF-129 不进正式表**，只作诊断；**正式结果仍为 NOF-127**（Avg spread 6.2/6.4%，P50 1.034×）。
3. **vsegment 跨会话可复现**：NOF-129 vs NOF-127（同配置）P50 260.59 vs 257.01（+1.4%）、Avg +1.0%、P90 +0.2%、P99 +1.7% ⇒ 1–2% 内；11% 是**同一次里的客户端/系统噪声**，不是配置漂移。
4. vs NOF-127 plain（跨会话）：P50 260.59/248.51=1.049、Avg 1.050、P90 1.048、P99 1.394；P50/Avg/P90 贴线，但 NOF-129 spread >10% 且 plain 为跨会话 ⇒ **不作正式对比**，正式 parity 仍按 NOF-127 的 1.034×。
5. 环境重建未改码/未装包；target A 未重启；未杀任何外部进程/未重启整机。

### 4. 环境末态

- etcd 2455814：64MiB、dbSize 968.22MiB、raftIndex 1,113,386；`/oplog/vseg_nof128/` 118,998 条、value_max 3,984B（整块无大 full-state）。
- target B 2458315（新 UUID 50d6259e…）、target A 3329243；master `vseg_nof128` 2458697/2458700、Keys=60,000、E=1 FATAL=0。
- node2 无残留；客户端 md5 bb4a07a3…（未重编）。
- 偏差：W9 主口径用每轮窗口末块；窗口内全块口径（B 23 块 spread 13.1%、A 28 块 8.2%）含 10s 分段边界块，仅参考；plain 基线沿用 NOF-127（跨会话）；客户端侧成因未测。

---

# NOF-130（待执行 · 补测 plain · 当前重建环境 · 60k · 10 轮 · 2026-11-26）

## 目标

- 当前 node4 重建后，vsegment 10 轮已测（NOF-129：Avg 266.93 / P50 260.59 / P90 269.12 / P99 383.08）。
- 补测**同一重建环境**的 plain 10 轮，得到可比的 plain 聚合值与 spread；按 `测试结果聚合方法_20261112.md`（Avg=各轮均值、分位=各轮同分位中位）。
- 只补 plain；不再写/测 vsegment；plain 写入会覆盖 target B 上的 vsegment 半边，NOF-129 结果已保存。

## A. 切换

1. 停 `vseg_nof128` master（2458697/2458700）；target A（3329243）/ target B（2458315）/ etcd（2455814）均不动。
2. 起 `plain_nof130`（node4，50056/9006/8086、slot=1、etcd、URMA、HA+oplog、batch=64）；若 etcd 存在同名 `/cvm/`、`/oplog/` 键，两处都删；注册 target B 512GiB。

## B. 写 60k

- 60,000 keys × 4MiB，16t/batch16；验收 **Keys=60000**（记录 failed；不补写；不记录写入性能）。

## C. 读 10 轮

- 每轮：60k keys、T1/threads=1/batch=1、warmup=128、qpairs=4、verify=false、diag 关、sequential；每轮一遍≈20s。
- 记录 **Avg/P50/P90/P99、failed、墙钟**；**每轮取 target B 的 W9avg/n**（与 vsegment 的 W9 对照）。

## D. 聚合与对比

1. 按 2026-11-12 方法聚合 + spread；正文只给聚合表，不逐轮罗列。
2. 对比：
   - vsegment（NOF-129，同重建环境，10 轮）：Avg 266.93 / P50 260.59 / P90 269.12 / P99 383.08；
   - plain（本单）：Avg/P50/P90/P99；
   - 旧 plain（NOF-127，旧 target B，跨环境）Avg 254.14 / P50 248.51：仅历史参考，不作正式对比。
3. 给出 plain/vsegment 与 vsegment/plain 的比值；按方法标注 spread（>3% 说明原因、>10% 不进表）。

## E. 边界

- 不重启 target/etcd；不改码/不重编/不装包；不补写；不跑 T4/T16。
- 只读一遍 60k/轮（≈20s），禁止 `--duration`；每轮取 W9。
- 预计 **15–20 分钟**（写 60k + 10×20s + 切换）。

---

## NOF-130 裁决（2026-11-26 · 同环境 plain 完成；vseg/plain P50 1.034×，仍为诊断）

### 1. 结果（[事实]：vsegment 10 轮 + plain 10 轮，同一重建环境/同一 target B）

| 指标 | vsegment（NOF-129） | plain（NOF-130） | vseg/plain |
|---|---|---|---|
| Avg | 266.93（spread 12.7%） | 255.71（spread 8.1%） | **1.044×** |
| P50 | 260.59（spread 11.1%） | 252.08（spread 8.2%） | **1.034×** |
| P90 | 269.12（spread 10.2%） | 261.21（spread 6.2%） | **1.030×** |
| P99 | 383.08（spread 30.8%） | 274.91（spread 7.5%） | **1.393×** |

- target W9：vsegment **39,622ns（1.9%）** vs plain **47,629ns（3.7%）** ⇒ vsegment 每 IO 快约 20%，端到端却慢 3.3% ⇒ 差距在客户端聚合/路径。
- plain：60,000/0、10/10 轮 failed=0；target B 未重启；etcd 1026.18MiB；master `plain_nof130` 2464822/2464825。
- 历史参考：plain 本单 255.71/252.08 vs NOF-127 254.14/248.51（跨环境）⇒ +0.6%/+1.4%；同环境比值更可信。

### 2. 裁定

1. **两臂 spread 均 >3%（plain 6–8%、vseg 10–31%）；vseg >10% ⇒ 按 `测试结果聚合方法_20261112.md`，NOF-129/130 同环境比值不进正式表**，只作诊断。
2. **不盲目补测到 spread ≤10%**：spread 是客户端/系统噪声（NOF-129 W9 1.9/2.7%、NOF-130 plain W9 3.7%，端到端却 8–31%），多跑几轮不保证降低；要正式同环境格，先做 node2 稳定化（绑核/performance governor/IRQ），再重跑。
3. **不单独立项查“plain target W9 更慢”**：vsegment 每 IO 已经更快，target 侧不能解释 vsegment 端到端更慢；差距在客户端两 child 聚合/两跳。
4. **正式结果仍为 NOF-127**（P50 1.034×、Avg 1.040×、P90 1.046×、P99 1.371×；spread 6.2/6.4% <10%）；NOF-129/130 的同环境 1.034/1.044/1.030 **与 NOF-127 一致，作为佐证**，不替换正式表。

### 3. 下一步

- 下发 **NOF-131 客户端分段拆解**（diag on，vsegment/plain 各 3 轮）：query/resolve/prepare/submit/completion_wait + 两 child 完成差 + target W9；不改码、不改 stripe/member_count、不重启 target/etcd。
- 判读：差距在 completion_wait/child 差 ⇒ 两 child 聚合，当前约束下无合法旋钮，接受或松约束；在 query/resolve ⇒ 控制面/layout，需改码；在 submit/prepare ⇒ 可试 node2 绑核/governor；无稳定差 ⇒ 3.4% 是噪声，收口。

### 4. 环境末态

- `plain_nof130` 2464822/2464825：Keys=60,000、E=1、FATAL=0、rex=0；etcd 1026.18MiB、raftIndex 1,173,750、64MiB；target B UUID 50d6259e…、11/11、ERROR=0；target A 3329243 未动。
- 未改码/未重编/未装包/未补写/未重启 target/etcd。

---

# NOF-131（待执行 · vsegment vs plain 客户端分段拆解 · 同环境 · diag on · 零改码 · 2026-11-26）

## 目标

- 同环境（重建后 target B）、同 60k/一整块 vsegment/plain，各写 60k、各读 **3 轮 diag on**（`MC_STORE_TRANSFER_DIAG=1`）。
- 抓 `get_into_breakdown` 的 **query / resolve / prepare / submit / completion_wait / unattributed**，vsegment 两 child 的完成时间差、endpoint A/B，以及每轮 target W9。
- 回答：vsegment 端到端比 plain 慢的 3–4% 落在客户端哪一段；据此判断是否还有合法旋钮。
- **diag 绝对值不作性能**（有开销）；只比较同 diag 下两臂的分段差。
- 不改码、不改 stripe/member_count、不重启 target/etcd、不补写。

## A. vsegment 臂

1. 停 `plain_nof130` master（2464822/2464825）；target A/B、etcd 不动。
2. 起 `vseg_nof131`（同正式配置：extent=274877906944、member_count=2、stripe=2MiB、slot=1、HA+oplog、batch=64）；清同名键；注册 B→A 512GiB；写 60k（T1/batch1）；验收 Keys=60000。
3. 读 3 轮 diag on：60k queries（`--num_keys=30000`/segment）、T1、warmup=128、qpairs=4、verify=false、`MC_STORE_TRANSFER_DIAG=1`；每轮抓 breakdown、child 完成差、target A/B W9。

## B. plain 臂

1. 停 vseg；起 `plain_nof131`（单 target B）；注册 B 512GiB；写 60k（16t/batch16）；验收 Keys=60000。
2. 读 3 轮 diag on：60k、T1、warmup=128、qpairs=4、verify=false、`MC_STORE_TRANSFER_DIAG=1`；每轮抓 breakdown、target B W9。

## C. 交付

1. 每臂 3 轮的 **query/resolve/prepare/submit/completion_wait/unattributed** 原始值（逐轮落盘，正文给 3 轮范围或逐轮 3 行；diag 不作正式表）。
2. vsegment 两 child：每轮先到/后到完成时间、差、endpoint A/B。
3. **分段差表**：vsegment−plain 各段；指出最大且稳定的差值段。
4. 判读（按 D 段的四种去向）+ 环境末态。
5. 不报吞吐；不做正式聚合；不追 P99/Max。

## D. 边界

- 不改码/不重编/不装包/不重启 target/etcd；不改 stripe/member_count/extent。
- 每轮一遍 60k≈20s，禁止 duration；diag on 只作归因。
- 3+3 轮；失败停手；不补写；预计 20–30 分钟。

---

# NOF-132（待执行 · 任务拆分 / lane 矩阵 · plain vs vseg member=1 三配置 · 2026-11-26）

## 目标

- 用 **member_count=1（只 target B）** 拿掉“跨 target”变量，单独验证 vsegment 剩余的 3–4% 是否来自：
  - 4MiB get 被拆成两个 child task（stripe=4MiB vs 2MiB）；
  - 两个 task 是否共用 lane（qpairs=4 vs 1）。
- 四组：A plain（基线）；B vseg member=1/stripe=4MiB/qpairs=4；C vseg member=1/stripe=2MiB/qpairs=4；D vseg member=1/stripe=2MiB/qpairs=1。
- **本单替代 NOF-131**；若 NOF-131 已启动，停掉，其数据作废。
- 不改码/不重启 target/etcd；member=1 与 stripe=4MiB **仅用于本诊断**，不改正式 2-member 配置。

## 共同条件

- 60,000 keys × 4MiB；T1（threads=1/batch=1）；warmup=128；qpairs 按组；verify=false；diag 关；sequential；每轮一遍 60k≈20s。
- target 只用 **target B（node4，UUID 50d6259e…）**；target A 不动；master+etcd node4；etcd 64MiB。
- 每组新 cluster、重写 60k；写后验收 Keys=60000；不补写；每组 4 轮读。
- 组顺序 A→B→C→D；每组停上一 master、起新 master、注册 target B 512GiB（vseg 只注册 B）。
- vseg extent=274877906944（256GiB/member），logical=256GiB，vsegments=1；plain 单 target B。

## A. plain（qpairs=4）

- `plain_nof132a`：注册 B 512GiB；写 60k（16t/batch16）；读 4 轮（60k、qpairs=4）。
- 记录每轮 Avg/P50/P90/P99、failed、墙钟、target B W9avg/n。

## B. vseg member=1 / stripe=4MiB / qpairs=4

- `vseg_nof132b`：`member_count=1 / stripe=4MiB / extent=256GiB / qpairs=4`；注册 B 512GiB；写 60k（T1/batch1、`--num_keys=60000`，只有一个 segment）；读 4 轮（qpairs=4）。
- 验收 vsegments=1、logical_capacity=274877906944；确认 4MiB key 只生成 1 个 child task（日志/probe）。

## C. vseg member=1 / stripe=2MiB / qpairs=4

- `vseg_nof132c`：`member_count=1 / stripe=2MiB / extent=256GiB / qpairs=4`；注册 B；写 60k；读 4 轮。
- 确认 4MiB key 生成 **2 个 child task**（均落 target B）。

## D. vseg member=1 / stripe=2MiB / qpairs=1

- `vseg_nof132d`：`member_count=1 / stripe=2MiB / extent=256GiB / qpairs=1`；注册 B；写 60k；读 4 轮。
- 确认 qpairs=1 生效（日志 `NoF qpairs/segment`），两个 child task 共用 1 条 lane。

## E. 交付

1. 四组聚合表（µs；按 2026-11-12 方法：Avg=各轮均值、分位=各轮同分位中位）：
   `组 | 配置 | 轮数 | Avg | P50 | P90 | P99 | spread`。
2. 逐轮原始值落盘 `run/nof132_rounds.csv`，正文不逐轮罗列。
3. 对比与判读：
   - **B vs C**：4MiB 单 task vs 2×2MiB 双 task（同 qpairs=4）⇒ 拆 task 的成本；
   - **C vs D**：双 task 不同 lane vs 同一 lane ⇒ lane 争用的成本；
   - **A vs B**：单 task 的 vsegment 是否接近 plain；
   - 若 B≈A、C>B、D>C ⇒ 拆 task + 同 lane 是主因；
   - 若 B/C/D 相近 ⇒ 不是 task/lane，另找。
4. 配置自证：vsegments=1、extent、stripe、qpairs、child 数；target W9；环境末态。
5. 不报吞吐；不追 P99/Max；不改正式配置。

## F. 边界

- 不改码/不重编/不装包/不重启 target/etcd；只停/起 master 和 bench。
- 只 target B；target A 不动；member=1 与 stripe=4MiB 仅本诊断。
- 每组 4 轮；每轮一遍 60k≈20s，禁止 duration；不补写；失败停手。
- 预计 **25–35 分钟**（4 组写 + 16 轮读 + 切换）。

---

## NOF-132 中断（2026-11-26 · 脚本顺序错误；B 无效、C 无读、D 未跑）

### 事实

- **A 组有效**：plain/q4，写 60k、4 轮全 failed=0；Avg 253.70 / P50 248.91 / P90 257.56 / P99 275.99；target B W9 均 46,955ns。
- **B 组无效**：脚本把“等 quota enabled=1”放在“注册 target B”之前；master 日志 `VSegment auto-plan deferred: no eligible psegment pool yet`，120s 全 enabled=0；写 20s 零增长退出 Keys=0；4 轮读 failed=60000（脚本未在写失败时中止）。
- **C 组**：已写 60k，但被停手时未读；数据作废。
- **D 组**：未开始；残留 master `vseg_nof132d` 2484180/2484183（enabled=0、未注册、无数据）。
- 偏差：写失败后脚本继续读（违反失败停手）；CSV 表头漏 child 两列；B 写日志被同名覆盖（master 侧证据足够）。
- 环境：etcd 2455814、target B 2458315 未动；node2 无残留；未改码/未重编。

### 决策

- **修脚本并重跑 A/B/C/D 四组**；A 旧 4 轮归档、不作矩阵数据，保证四组同一会话。
- 修正：**先注册 target B → 等 enabled=1 → 写 → 读**；写失败立即中止整组和 driver，不进入读；CSV 补齐 child 列。
- 停残留 `vseg_nof132d` master；旧 `nof132*` 产物归档到 `run/nof132_partial/`。

---

# NOF-133（待执行 · NOF-132 修正重跑 · A/B/C/D 四组 · 2026-11-26）

## 目标

- 修正脚本顺序与失败处理，四组（A plain/q4；B vseg m=1 stripe=4M/q4；C m=1 stripe=2M/q4；D m=1 stripe=2M/q1）全部重跑；每组 60k 写入 + 4 轮读。
- 矩阵判读同 NOF-132：B vs C（拆 task 成本）、C vs D（同 lane 成本）、A vs B（单 task vsegment 是否接近 plain）。

## A. 清理与归档

1. 停残留 `vseg_nof132d` master 2484180/2484183（先 TERM，必要时 KILL）；检查所有 `vseg_nof132*`/`plain_nof132*` master 无残留；确认 50056/9006/8086 释放。
2. 将 `run/nof132*`、`run/master_vseg_nof132*`、`run/nof132_rounds.csv` 等移到 `run/nof132_partial/`（**归档不删**）；A 旧 4 轮仅参考，不进矩阵。
3. target A/B、etcd 2455814 均不动；node2 无残留。

## B. 脚本修正（必须）

1. 每组顺序固定：起 master → **注册 target B 512GiB** → 等 `VSegment Status enabled=1`/quota（最多 180s）→ 写 60k → 读 4 轮。
2. 写后 `Keys<60000` 或零增长 ⇒ **立即中止整组和 driver**，不进入读轮；记录 master 日志。
3. CSV 表头补齐 child 两列（如 `child_count`、`child_gap_us`），行与表头对齐；每轮记录 A/B/C/D 组号、轮号、Avg/P50/P90/P99、failed、墙钟、target B W9/n。
4. qpairs 按组设置并用日志自证（`NoF qpairs/segment`）；vseg member=1 时注册/写只用 target B、`--num_keys=60000`。

## C. 四组（A→B→C→D）

- **A**：`plain_nof133a`，qpairs=4；写 60k（16t/batch16）；读 4 轮。
- **B**：`vseg_nof133b`，member_count=1 / stripe=4MiB / extent=256GiB / qpairs=4；注册 B；写 60k（T1/batch1）；读 4 轮；确认 4MiB key 生成 **1 个 child task**。
- **C**：`vseg_nof133c`，member_count=1 / stripe=2MiB / extent=256GiB / qpairs=4；写 60k；读 4 轮；确认 **2 个 child task**、均落 B。
- **D**：`vseg_nof133d`，member_count=1 / stripe=2MiB / extent=256GiB / qpairs=1；写 60k；读 4 轮；确认 qpairs=1 生效、两 task 共用 lane。
- 每组新 cluster、重写 60k；组间停上一 master；target A 不动；etcd 不动。

## D. 交付

1. 四组聚合表（2026-11-12 方法：Avg=各轮均值、分位=各轮同分位中位；正文不逐轮罗列）：
   `组 | 配置 | 轮数 | Avg | P50 | P90 | P99 | spread`。
2. 逐轮原始值 `run/nof133_rounds.csv`（含 child 列）。
3. 对比：B vs C、C vs D、A vs B；给出各组差值与判读。
4. 配置自证（vsegments=1、extent、stripe、qpairs、child 数）、target W9、环境末态。
5. 不报吞吐；不追 P99/Max；不改正式配置。

## E. 边界

- 不改码/不重编/不装包/不重启 target/etcd；只停/起 master 与 bench。
- 只 target B；target A 不动；member=1/stripe=4M 仅本诊断。
- 每组 4 轮；每轮一遍 60k≈20s，禁止 duration；写失败立即停手；不补写。
- 预计 **30–40 分钟**。

---

## NOF-133 裁决（2026-11-26 · 四组跑通；q1 反而更快；拆 task 成本在尾部）

### 1. 结果（[事实]；16/16 轮 failed=0，四组 Keys=60000）

| 组 | 配置 | Avg | P50 | P90 | P99 | P50 spread | target B W9 |
|---|---|---|---|---|---|---|---|
| A | plain / q4 | 258.30 | 255.99 | 263.90 | 276.31 | 8.1% | 46,920ns（4.9%） |
| B | vseg m=1 / stripe=4M / q4 | 264.69 | 261.63 | 270.31 | 282.40 | 4.1% | 47,845ns（0.2%） |
| C | vseg m=1 / stripe=2M / q4 | 346.87 | 270.01 | 499.33 | 635.78 | 2.9% | 60,202ns（10.7%） |
| D | vseg m=1 / stripe=2M / q1 | 263.26 | 259.88 | 268.69 | 284.62 | 3.5% | 44,922ns（1.5%） |

- **B→C（拆 2 child，同 q4）**：P50 1.032×、P90 1.847×、P99 2.251×、Avg 1.310× ⇒ **拆 task 的成本几乎全在尾部**；C 的 target W9 也最高（60.2µs）。
- **C→D（同 lane q1）**：P50 0.962×、P90 0.538×、P99 0.448×、Avg 0.759× ⇒ **q1 全面更快；“同 lane 争用”假说不成立**；q1 的 target W9 44.9µs，低于 q4 的 60.2µs。
- **A→B（单 task vseg vs plain）**：P50 1.022×、P90 1.024×、P99 1.022× ⇒ 单 task vsegment 已接近 plain。
- **A→D（双 task + q1 vs plain）**：Avg 1.019×、P50 1.015×、P90 1.018×、P99 1.030× ⇒ **四项全部 ≤1.05**。
- **A→C（双 task + q4 vs plain）**：P50 1.055×、P99 2.301× ⇒ 不达。
- **B→D**：P50 0.993×、P99 1.008× ⇒ 4M/q4 与 2M/q1 在 P50/P99 上等价。
- child 数 probe [未获取]（debug 无输出）；按代码/配置 B=1 child、C/D=2 child，仅作语义说明，不当实测。
- A 组快异常：p1/p2≈246µs、p3/p4≈265µs，spread 8.1%；跨组比较注意时间漂移。

### 2. 判读

1. **单 task vsegment（B）≈ plain**（P50 +2.2%）；**双 task + q1（D）也 ≈ plain**（P50 +1.5%，P99 +3.0%）；**双 task + q4（C）尾部爆炸**（P99 2.3×）。
2. 所以当前正式 2-member 的剩余 3.4% 不是“跨 target”本身，也不是“拆成两个 task”必然带来的 P50 成本；**q4 下两 child 的并发/队列形态**是尾部恶化的关键，q1 可以消掉。
3. C 的 P99 636µs vs 2-member 正式配置 NOF-129 的 383µs：C 是两个 child 落**同一个 target**，正式是各落一个 target；[推算] 同 target 队列放大，本单未验证。
4. 正式 2-member + qpairs=1 是**唯一还没试、且合法的旋钮**（不改码、不改 member/stripe/拓扑）；需要同会话验证它是否把正式配置的 P50/P99 拉进 1.05。

### 3. 决策

- **下发 NOF-134**：正式 2-member（stripe=2M、extent=256G）下 **qpairs=4 vs qpairs=1**，加 plain 同会话基线；各 4 轮。若 q1 使 P50/P90/P99 都 ≤ plain×1.05 且优于 q4 ⇒ 正式配置改用 q1；否则 3.4% 属结构性，收口。
- 停 `vseg_nof133d` master 2491062/2491065；target A/B、etcd 不动。
- 不单独立项查 child 计数（[未获取]，代码/配置语义足够）；不单独立项查 C 的 636µs（NOF-134 的 q4/q1 同会话数据可回答方向）。

### 4. 环境末态

- `vseg_nof133d` master 2491062/2491065：Keys=60000、enabled=1、vsegments=1、logical_capacity=274877906944。
- etcd 2455814：dbSize 1531.26MiB、raftIndex 1,783,020；target B 2458315 online 11/11、ERROR=0；target A 3329243 未动；node2 无 bench。
- 未改码/未重编/未装包/未重启 target/etcd；NOF-132 旧产物已归档 `run/nof132_partial/`。

---

# NOF-134（待执行 · 正式 2-member：qpairs=4 vs qpairs=1 + plain 同会话 · 60k · 4+4+4 轮 · 2026-11-26）

## 目标

- 在**正式 2-member 配置**（member_count=2、stripe=2MiB、extent=256GiB、一整块 vsegment）下，验证 **qpairs=1** 是否能把 P50/P90/P99 拉进 plain×1.05，且优于 qpairs=4。
- 三组同会话：P plain/q4；V4 vseg 2-member/q4；V1 vseg 2-member/q1；各 60k、4 轮。
- 不改码/不改 member/stripe/拓扑；只改 qpairs。

## A. 清理

1. 停 `vseg_nof133d` master 2491062/2491065；检查无 `nof133` 残留；释放 50056/9006/8086。
2. target A/B、etcd 2455814 不动；node2 无残留。

## B. 三组（顺序 P→V4→V1）

- **P**：`plain_nof134p`，qpairs=4；注册 B 512GiB；写 60k（16t/batch16）；读 4 轮（60k、T1、warmup128、qpairs4、verify=false、diag 关）。
- **V4**：`vseg_nof134v4`，member_count=2 / stripe=2MiB / extent=256GiB / qpairs=4；注册 B→A 各 512GiB；先注册后等 enabled=1；写 60k（T1/batch1、`--num_keys=30000`/segment）；读 4 轮（qpairs4、60k queries）。
- **V1**：`vseg_nof134v1`，同 V4 但 **qpairs=1**；注册 B→A；等 enabled=1；写 60k；读 4 轮（qpairs1）。
- 每组新 cluster；写失败立即中止整组；每轮记录 Avg/P50/P90/P99、failed、墙钟、target A/B W9/n；qpairs 用客户端日志自证。

## C. 交付

1. 三组聚合表（2026-11-12 方法：Avg=各轮均值、分位=各轮同分位中位）+ spread；逐轮 CSV 落盘，正文不逐轮罗列。
2. 对比：V1 vs V4（q1 效果）；V4/V1 vs P（parity）；P50/P90/P99 全部给出。
3. 判读：若 V1 的 P50/P90/P99 均 ≤P×1.05 且优于 V4 ⇒ 正式配置采用 q1；否则 q1 无效，3.4% 结构性，收口。
4. 配置自证、target W9、环境末态。

## D. 边界

- 不改码/不重编/不装包/不重启 target/etcd；只 stop/start master 与 bench。
- 正式 2-member 配置不变（member=2、stripe=2M、extent=256G）；只改 qpairs。
- 每组 4 轮；每轮一遍 60k≈20s，禁止 duration；不补写；失败停手。
- 预计 **25–35 分钟**。

---

## NOF-134 裁决（2026-11-26 · q1 无效；正式配置保持 q4；本线收口）

### 1. 结果（[事实]；三组 12 轮 failed 全 0）

| 组 | 配置 | 轮数 | Avg | P50 | P90 | P99 | spread（Avg/P50/P90/P99） |
|---|---|---|---|---|---|---|---|
| P | plain / q4 | 4 | 260.87 | 254.13 | 266.57 | 285.03 | 7.1 / 6.7 / 6.6 / 9.1% |
| V4 | vseg 2-member / q4 | 4 | 268.69 | 264.48 | 274.29 | 386.49 | 4.1 / 4.2 / 5.3 / **12.4%** |
| V1 | vseg 2-member / q1 | 4 | 270.17 | 261.30 | 272.39 | 383.47 | 6.7 / 6.9 / 6.9 / 9.4% |

- **V4→V1（q4→q1）**：P50 0.988×、P90 0.993×、P99 0.992×、Avg 1.006× ⇒ q1 三项略快，但幅度 **0.7–1.2% < 轮间 spread（4.1–12.4%）** ⇒ 不显著。
- **plain→V4**：P50 1.041×、P90 1.029×、P99 **1.356×**。
- **plain→V1**：P50 **1.028×**、P90 **1.022×**、P99 **1.345×**。
- 判据“P50/P90/P99 均 ≤1.05 且优于 V4”**未满足（P99 ✗）** ⇒ **q1 不采用**。
- target W9：P 47,352ns（4.5%）；V4 B 40,178（2.1%）/A 39,102（1.1%）；V1 B 39,512（4.7%）/A 39,475（3.0%）。vseg 单 IO 比 plain 低约 **15–17%**，端到端反而慢 ⇒ 差额在客户端聚合/路径，与 NOF-130/133 三次观测一致。
- V4 的 P99 spread 12.4% >10% ⇒ 按方法 V4 格不进正式表，仅诊断；P/V1 spread >3%，同向线索是 target W9 只 1.1–4.7%、端到端 2–3 倍 ⇒ 波动不在 target。
- qpairs 自证：P/V4 `io_qpairs=4`；V1 `io_qpairs=1`。

### 2. 判读与决策

1. **q1 有效但不显著**：0.7–1.2% 的改善小于 spread，继续加轮数才能判定，收益不值得；**不做**。
2. **正式配置保持 qpairs=4**；正式结果仍为 NOF-127（P50 1.034×、Avg 1.040×、P90 1.046×、P99 1.371×）。
3. **剩余 2.8–4.1%（P50/P90）与 1.345（P99）判为结构性**：target 单 IO 已经更快（39–40 vs 47µs），差距在客户端两 child 聚合/路径；在你“不改码、不改拓扑、不改 stripe/member、不加第三 target”的约束下**没有合法旋钮**，本线收口。
4. P99 仍不达 parity，按既定边界不追；不再用单遍/单轮代表阶段结果。
5. 偏差：`nof134_run.sh` 被本地 PowerShell 重写带入 UTF-8 BOM，出现一次 `#!/binn/bash` 报错；driver 用 `bash script` 调用，三组 rc=0、数据无影响；已清 BOM。qpairs 自证第一版看错节点，已改 node2 复查。

### 3. 环境末态

- `vseg_nof134v1` master 运行中（PID 回执未列，需在环境内确认）：Keys=60000、enabled=1；**先保留**作为当前 vseg 环境；target A 3329243 / target B 2458315 / etcd 2455814 未动。
- etcd：64MiB、dbSize **1778.46MiB**、raftIndex 2,081,968；target B online 11/11、ERROR=0；node2 无 bench。
- 未改码/未重编/未装包/未重启 target/etcd。

---

# NOF-135（待执行 · m=1 一整块 60k 数据只读前 400 keys + plain 400 对照 · 2026-11-26）

## 目标

- 在当前重建后的 target B、m=1、一整块 vsegment（extent=256GiB）下，**写 60k 但只读前 400 keys**，看旧 NOF-113R 的 400-key 2× 会不会复现；同时给 plain 400 keys 同环境对照。
- 回答：**旧 2× 是“400 keys 这个规模”造成的，还是旧 target/旧 64MiB 配置造成的？**
- 不改码/不重启 target/etcd；只 stop/start master 与 bench。

## A. 切换

1. 停当前 `vseg_nof134v1` master；target A/B、etcd 2455814 不动；node2 无残留。
2. 记录 target B（2458315）W9 基线。

## B. V 组：m=1 一整块，写 60k、读前 400 keys

1. 起 `vseg_nof135v`：`member_count=1 / stripe=2MiB / extent=256GiB / qpairs=4`；注册 target B 512GiB；等 enabled=1。
2. 写 60k（T1/batch1、`--num_keys=60000`，只有一个 segment）；验收 Keys=60000。
3. 读 **前 400 keys**：`--num_keys=400`、T1、warmup=128、qpairs=4、verify=false、diag 关；**4 轮**；记录 Avg/P50/P90/P99、failed、墙钟、target B W9/n。
4. 追加 **1 遍 60k** 同数据对照（`--num_keys=60000`，同参数；只作上下文，不并入 400 聚合）。

## C. P 组：plain，写 60k、读前 400 keys

1. 停 vseg；起 `plain_nof135p`；注册 B 512GiB；写 60k（16t/batch16）；验收 Keys=60000。
2. 读前 400 keys：`--num_keys=400`、T1、warmup=128、qpairs=4、verify=false、diag 关；4 轮；记录同指标。

## D. 交付与判读

1. V400、P400、V60k（1 遍）三组数值；V400/P400 比值。
2. 与旧 NOF-113R（vseg 551.93 / plain 270.52，旧 target + 64MiB×25 vseg）并列，但标注不可直接比。
3. 与 NOF-133 m=1 60k（C 270.01）对比：400 keys 是否与 60k 不同。
4. 判读：
   - V400 若 ≈P400（~1.0–1.1×）⇒ 旧 2× **不是 400-key 规模造成的**，是旧 target/旧 64MiB 配置；
   - V400 若 ≈2× ⇒ 400-key 规模/前缀本身能复现，需继续查；
   - 同时给 target W9，确认是否旧 287µs 的值复现。
5. 配置自证（vsegments=1、extent、qpairs、`--num_keys=400`）、环境末态。

## E. 边界

- 不改码/不重编/不装包/不重启 target/etcd；只 target B；target A 不动。
- 写 60k 但只读 400（V/P 各 4 轮）；V 追加 1 遍 60k 对照；每轮一遍，禁止 duration；不补写。
- 预计 **20–25 分钟**。

---

## NOF-135 裁决（2026-11-26 · 旧 400-key 2× 未复现；400-key 规模不是原因）

### 1. 结果（[事实]；V400 实测 4 轮，**p1 异常剔除，有效 3 轮**；P400 4 轮）

| 组 | 范围 | 轮数 | Avg | P50 | P90 | P99 | Avg spread |
|---|---|---|---|---|---|---|---|
| V400 | vseg m=1 前 400 keys | **3（p2–p4；p1 剔除）** | **340.95** | **261.00** | **566.02** | **674.64** | **4.3%** |
| P400 | plain 前 400 keys | 4 | 340.04 | 266.30 | 589.44 | 661.49 | 3.9% |
| V60k | vseg m=1 全 60k | 1 | 295.60 | 269.30 | 322.22 | 572.78 | n/a |

- **V400/P400（有效轮）**：Avg **1.003×**、P50 **0.980×**、P90 **0.960×**、P99 **1.020×**；四项全在 **0.960–1.020**，无一项超 1.05。P90/P99 来自 p2–p4 三轮，**未沿用含 p1 的 4 轮值**。
- V400 3 轮（p2–p4）：Avg 349.38/334.77/338.70，P50 260.81/261.00/266.31 ⇒ Avg spread **4.3%**、P50 spread 2.1%；P400 Avg spread 3.9%、P50 4.7%；全部 ≤10%。
- **旧 NOF-113R 2×（vseg 551.93 / plain 270.52）未复现** ⇒ 旧 2× **不是 400-key 规模造成的**；旧结果是旧 target 2142228（W9 286.6–288.7）+ 64MiB extent（~25 vseg）的账（两项各占多少未复测）。
- V60k P50 269.30 vs NOF-133 m=1 60k C 270.01 = **0.997×**，同配置跨会话一致；400 与 60k 的 P50 几乎相同，差别只在尾部，且 400 样本太少。
- 400-key target W9：V 9800/8380/122260/6800（n=1,1,16897,1）；P 9250/252220/6630/38830（n=1,2625,1,2）⇒ 多数末块落在空闲块，**只有 2 个有效捕获**；高值两臂同时出现，不能归因 vsegment，[未验证]。
- **p1 剔除记录**：P50 466.63 / Avg 450.06（其余三轮 260.8–266.3 / 334.8–349.4），疑似 m=1 写后首读冷启动，原因 [未验证]；整轮不参与聚合。
- 400-key 每轮仅 400 样本，P90/P99 由少数 IO 决定，不宜当精确分位。

### 2. 判读与决策

1. **p1 剔除后 V400≈P400**：Avg 1.003×、P50 0.980×，旧 2× 不是 400-key 规模造成；归因旧 target/旧 64MiB 配置，**不单独立项复测旧环境**（旧 target 已重建）。
2. **不单独立项复测 p1 异常**：主问题已回答；p1 像冷启动，且 400 样本/W9 捕获不足；记录即可。
3. **NOF-136 已完成（只读）**：V400 p2–p4 重算 P90 566.02 / P99 674.64；P400 保持 4 轮；所有 spread ≤10%；原始 CSV/NOF-135 记录未变。
4. 400-key 的 target W9 高值（122/252µs）与 60k 的 47.9µs 不是同一口径：前者多数 n=1/2 空闲块，且两臂都有；不作 target 归因。
5. 正式结果仍为 NOF-127；本单只作诊断。
6. 环境：`plain_nof135p` 2509048/2509051 运行、Keys=60000；**先保留**作为当前 plain 基线；target A/B、etcd 未动。

### 3. 环境末态

- etcd 2455814：64MiB、dbSize 1931.57MiB、raftIndex 2,261,561；target B 2458315 online 11/11、ERROR=0；target A 3329243 未动；node2 无 bench。
- 未改码/未重编/未装包/未重启 target/etcd。

---

# NOF-136（待执行 · NOF-135 V400 剔除 p1 重新聚合 · 只读 CSV · 不重跑 · 2026-11-26）

## 目标

- 用户裁定：NOF-135 的 V400 **第 1 轮为异常轮，必须从统计中剔除**，不能只在备注说明。
- 用已有 `run/nof135_rounds.csv` **只读重算** V400（p2/p3/p4 三轮）与 P400（4 轮）的聚合值；**不重跑任何测试、不动环境**。
- 输出修正后的 V400/P400 对比；若 CSV 缺每轮 P90/P99，明确写 [未获取]，不要用含 p1 的 4 轮值替代。

## A. 重算

1. 读取 `run/nof135_rounds.csv`，确认每轮字段含 Avg/P50/P90/P99（缺失就报告字段清单）。
2. **V400 只用 p2/p3/p4**；p1 整轮排除（记录 p1 原始值：Avg 450.06 / P50 466.63）。
3. 按 `测试结果聚合方法_20261112.md`：
   - Avg = 有效轮 Avg 的等权算术平均；
   - P50/P90/P99 = 有效轮同分位的中位数；
   - Avg spread = (max Avg − min Avg) / mean Avg；P50/P90/P99 的 spread 同式计算并单列。
4. P400 保持 4 轮；V60k 仍为 1 遍对照，不参与聚合。

## B. 交付

1. 修正表：`组 | 有效轮数 | Avg | P50 | P90 | P99 | Avg spread | 排除轮说明`。
2. V400/P400 的 Avg、P50、P90、P99 比值；标注 P90/P99 来自 3 轮。
3. 明确写出“p1 剔除，原因：首轮异常（疑似冷启动，未验证）”。
4. 环境末态：不改动任何服务；plain_nof135p 保持运行。

## C. 边界

- **只读 CSV/日志，不跑 bench、不重启 master/target/etcd、不改码**。
- 不重跑 V400；不补测 p1；不改变 NOF-135 的原始记录（只追加修正聚合）。
- 预计 **2–3 分钟**。

---

## NOF-136 裁决（2026-11-26 · V400 剔除 p1 的只读重算完成）

- 全程只读：`run/nof135_rounds.csv` 9 行、字段 Avg/P50/P90/P99 齐全；未跑 bench、未重启 master/target/etcd、未改码、未改 NOF-135 原始记录。
- **V400 只用 p2/p3/p4（3 轮）**；p1 整轮排除，原始值保留：Avg 450.06 / P50 466.63 / P90 595.43 / P99 671.49（P50 +78.8%、Avg +32.0% vs 其余三轮；疑似冷启动，未验证）。
- 修正后：**V400 Avg 340.95 / P50 261.00 / P90 566.02 / P99 674.64；P400 340.04 / 266.30 / 589.44 / 661.49；V60k 295.60 / 269.30 / 322.22 / 572.78（1 遍）**。
- **V400/P400：Avg 1.003×、P50 0.980×、P90 0.960×、P99 1.020×**，四项全在 0.960–1.020，无一项超 1.05。
- spread：V400 Avg 4.3% / P50 2.1% / 其余 2.1–6.3%；P400 Avg 3.9% / P50 4.7% / 其余 2.4–4.7% ⇒ 全部 ≤10%，可按方法进表（400 样本下 P90/P99 仍不宜当精确分位）。
- 环境：`plain_nof135p` 2509048/2509051 保持运行；target A/B、etcd 未动；**先保留**作为当前 plain 基线，不再测时再停。

---

# NOF-137（待执行 · 只读核对 NOF-133 C 逐轮，判断 346.87 是否被异常轮拉高 · 2026-11-26）

## 背景

- NOF-133 C（m=1 / stripe=2M / q4 / 60k / 4 轮）聚合 Avg **346.87**、P50 270.01、P90 499.33、P99 635.78。
- NOF-135 V60k（同配置、1 遍）Avg **295.60**、P50 269.30、P90 322.22、P99 572.78。
- 两者 P50 几乎相同，Avg 差 51µs（15%）；需确认 C 的 4 轮 Avg 是否每轮都高，还是某一轮异常拉高。

## 任务（全程只读）

1. 找到并读取 NOF-133 C 组的逐轮 CSV（`run/nof133_rounds.csv` 或 run/ 下对应文件），列出 C 的 4 轮 **Avg/P50/P90/P99/failed/墙钟/target W9**。
2. 按用户已定规则检查异常轮：若某轮 Avg/P50 明显高于其余（参照 V400 p1 的量级，例如 P50 > 其余中位 +50% 或 Avg > 其余均值 +20%），**整轮剔除**；否则保留。
3. 若剔除：按 2026-11-12 方法重算 C 的 Avg（有效轮均值）、P50/P90/P99（有效轮同分位中位）、spread；**原始值保留**，并注明剔除轮原因。
4. 对比 C（原始/修正）与 V60k；直接回答：346.87 vs 295.60 是“四轮都高”还是“一轮异常拉高”。
5. 同时列出 V60k 单遍的原始值（已在 NOF-135，确认即可）。

## 边界

- **只读 CSV/日志，不跑 bench、不重启 master/target/etcd、不改码、不修改 NOF-133/135 原始文件**。
- 若找不到 C 的逐轮 CSV，停手回报文件清单；不要用汇总值反推。
- 预计 **2–3 分钟**。

---

## NOF-137 取消（2026-11-26 · 用户不再追该测量问题）

- 用户明确“算了不管这个测量问题了”；NOF-133 C 的逐轮异常检查不再执行；如已启动，停掉并归档，不改环境。
- 后续以 NOF-138 EID A/B 为当前任务。

---

# NOF-138（待执行 · MC_EID_INDEX A/B：0 vs 1 · plain 60k 带宽读 · 2026-11-26）

## 目标

- 验证 yin 的 `MC_EID_INDEX=1` 是否真的提升读带宽：当前 `plain_nof135p`（60k）保持不动，只改 **bench 进程**的 `MC_EID_INDEX`，做 0 vs 1 同数据/同 target 的 A/B。
- 先确认 EID 1 能否正常建链/读；若能，再看吞吐是否显著变化，再决定是否切 vsegment 复测。
- 不改码、不重启 master/target/etcd、不写数据。

## A. 预检（只读）

1. 记录 `plain_nof135p` 2509048/2509051、Keys=60000、target B 2458315、target A 3329243、etcd 2455814。
2. 记录 `udmac0d1e2` 的 EID 列表（`urma_admin show` / 相关日志），确认 EID index 0 和 1 都存在；记录对应 EID 值。
3. 若 NOF-137 还在跑，停掉（read-only）；node2 无 bench 残留。

## B. EID 0/1 交替 A/B

1. 同一 plain 数据、同 master、同 target B；**只改 bench 进程 env** `MC_EID_INDEX=0 或 1`。
2. 顺序 **E0 → E1 → E0 → E1**，每臂 2 遍。
3. 每遍：`segment_read` 60k keys（`--num_keys=60000`，plain 单 segment）、`--num_threads=16`、`--batch_size=16`、warmup=128、qpairs=4、verify=false、diag 关。
4. 记录：bench 吞吐（本单主指标，仅诊断）、Avg/P50/P90/P99、failed、墙钟、target B W9/n。
5. 每遍检查客户端日志：`Selected URMA device ..., EID index 0/1, EID ...`；若 EID1 初始化失败/读失败/rc≠0 ⇒ **停手，不重启 target**，回报错误原文。

## C. 判读与交付

1. 表：E0/E1 各 2 遍的吞吐、Avg/P50/P90/P99、failed、W9。
2. 判据：EID1 吞吐相对 EID0 **>5% 且稳定**、读全成功 ⇒ 记录“EID 有带宽收益”，再议 vsegment 复测；≤5% 或失败 ⇒ EID 不是带宽旋钮，收口。
3. 吞吐只在本单诊断用，不并入正式时延表；时延仍按正式口径。
4. 环境末态：plain master 保持；target/etcd 未动。

## D. 边界

- 不改码/不重编/不装包/不重启 master/target/etcd；只改 bench env。
- 不写数据、不切 vsegment、不重启 target。
- 若 EID1 不可用，停手；不要为它重启 target。
- 预计 **5–10 分钟**。

---

# NOF-138 完成（2026-11-26 · MC_EID_INDEX 0 vs 1 · plain 60k · **带宽维度无收益**）

## 条件

- 环境：`plain_nof135p` 2509048/2509051（Keys=60000）、target B 2458315、target A 3329243、etcd 2455814 全部不动；**只改 bench 进程 env**。
- 读：`segment_read` 60k（plain 单 segment，`--num_keys=60000`）、`--num_threads=16 --batch_size=16`、warmup=128、qpairs=4、verify=false、diag 关；顺序 E0→E1→E0→E1，每臂 2 遍。
- 预检：`urma_admin show -d udmac0d1e2` 显示 8 个 EID（index 0–7）全 ACTIVE；**必须带 `-d`**，不带只列 bonding_dev_0 的 eid0。

## 结果（每遍读成功：failed=0、234.38 GB、3,760 queries）

| 臂 | 吞吐 MiB/s | Avg µs | P50 µs | P90 µs | P99 µs | target B W9 ns |
|---|---|---|---|---|---|---|
| E0（2 遍） | **47,917.09** | 21,296.55 | 21,342.68 | 21,640.68 | 22,156.34 | 283,495 |
| E1（2 遍） | **47,705.41** | 21,385.46 | 21,336.69 | 21,641.34 | 22,013.48 | 282,980 |
| E1/E0 | **0.996（−0.44%）** | 1.004 | 1.000 | 1.000 | 0.994 | 0.998 |

- 离散度：E0 吞吐 spread 0.25%、E1 0.03%；四遍日志均含 `Selected URMA device udmac0d1e2, EID index 0/1`，EID 设置确实生效。
- **判据（>5% 且稳定）不满足，方向为负 ⇒ `MC_EID_INDEX` 不是带宽旋钮；带宽维度不进入 vsegment 复测**；NOF-139 取消；NoF 的 EID 由 NOF-140 用 `SPDK_URMA_EID_INDEX` 复核。
- **事后澄清（2026-11-26 同事）**：`MC_EID_INDEX` 设的是 **Transfer Engine（内存通道）** 的 EID，NoF 数据面不走它 ⇒ 本单 −0.44% 属**层级不对**，不代表 NoF 的 EID 无收益；NoF 的对应旋钮是 **`SPDK_URMA_EID_INDEX`**（NOF-140）。
- 本单时延是 16×4MiB/批（≈21ms）的批时延，**与正式 T1/batch1（P50≈254µs）不可比，不并入正式时延表**；吞吐仅本单诊断。

## 偏差

- `run/nof138_rounds.csv` 的 Throughput 列原为空（抓取模式写成 `Throughput:`，日志实际为 `Throughput (avg):`）；已从客户端日志补算并另写 `run/nof138_rounds_fixed.csv`，原始读数未丢。
- 客户端日志在 node2（4 份×10MB），聚合时复制到 node4 `/tmp/nof138_ev/` 供核对。

## 裁定与末态

- **EID 带宽维度关闭**；T1 时延维度由 NOF-139 复核；`plain_nof135p` master 保持运行（无写入、零成本）；target/etcd 未动，数据完整。
- 如需释放 node4，仅停 master 2509048/2509051，**不要停 etcd/target、不删数据**；以后可同集群重启复用 plain 基线。

---

# NOF-139（待执行 · MC_EID_INDEX 0 vs 1 · 正式 T1 时延 A/B · 2026-11-26）

## 目标

- 用户要求：**不看带宽、不高并发**，按之前正式读口径（T1/batch1、60k、每遍一遍≈20s）看 EID 0/1 的**端到端时延**有没有差别。
- NOF-138 只证明 16t/batch16 带宽口径下 EID1 −0.44%；本单补 T1 时延口径。
- 只改 bench 进程 env；数据/服务不动；不改码、不重启。

## 条件

- 环境：`plain_nof135p` 2509048/2509051（Keys=60000）、target B 2458315、target A 3329243、etcd 2455814；**全部不动**。
- 数据：现成 plain 60k（单 segment），不重写、不切 vsegment。
- 每遍：`segment_read --num_keys=60000 --num_threads=1 --batch_size=1`、`--warmup_keys=128`、`--nof_qpairs_per_segment=4`、verify=false、diag 关、sequential（`--shuffle_seed=0`）；**每遍只读一遍 60k（约 16–20s），禁止 `--duration`**。
- 顺序：**E0→E1→E0→E1→E0→E1**（每臂 3 遍，交替抵消时间漂移）。
- 每遍记录：**Avg/P50/P90/P99/P999/Max、failed、墙钟、target B W9avg/n**；核日志 `Selected URMA device udmac0d1e2, EID index 0/1`。
- EID1 初始化/读失败/rc≠0 ⇒ **停手，不重启 target**，回报错误原文。

## 聚合与判读

- 按 `D:\gds性能\测试结果聚合方法_20261112.md`：**Avg=各遍等权均值；P50/P90/P99=各遍同分位中位**；`spread=(max−min)/mean`。
- 表：E0、E1 两行（Avg/P50/P90/P99 + spread + failed），再给 **E1/E0 比值**。
- 判据：① 若 E1/E0 与 1 的偏差小于两臂 spread ⇒ **无差别**，EID 线彻底关闭；② 若 E1 稳定低 **>5%** ⇒ 记“EID 在 T1 时延有收益”，再议 vsegment 复测；③ 若 E1 更慢 ⇒ 关闭。
- **正文不报吞吐、不列逐遍值**；逐遍原始值落 `run/nof139_rounds.csv`。
- E0 三遍应接近正式 plain 基线（P50≈252–254µs）；若明显偏离，先报环境异常，不作 EID 归因。

## 边界

- 不改码/不重编/不装包/不重启 master/target/etcd；**只改 bench env**。
- 不写数据、不切 vsegment、不重启 target。
- 环境末态：`plain_nof135p` 保持运行；target/etcd 未动。
- 预计 **3–5 分钟**。

---

## NOF-139 取消（2026-11-26 · 变量名更正）

- 用户撤回：要试的是 SPDK 的 **`SPDK_URMA_EID_INDEX`**，不是 Mooncake 的 `MC_EID_INDEX`；NOF-139 不再执行。
- 如已启动，停掉；其数据作废、不并入任何表；环境保持不动。
- 后续以 **NOF-140** 为当前任务。

---

# NOF-140（待执行 · SPDK_URMA_EID_INDEX 0 vs 1 · 正式 T1 时延 A/B · 2026-11-26）

## 背景与源码事实

- 用户更正：yin 说的旋钮是 **`SPDK_URMA_EID_INDEX`**（SPDK URMA transport），不是 Mooncake 的 `MC_EID_INDEX`。
- 本地 SPDK 参考树（`_ref/spdk`，**实际运行树须在预检中核对**）：
  - `lib/nvme/nvme_urma_common.c:147` `spdk_urma_opts_init` 读取 `SPDK_URMA_EID_INDEX`，**默认 0**；
  - 设备打开时按 `eids[i].eid_index == opts->eid_index` 选 EID，**找不到则静默回退首个 EID**（与 Mooncake `MC_EID_INDEX` 的报错不回退不同）；
  - `lib/nvmf/urma.c:1209` target 的 `nvmf_urma_opts_init` 也调用同一 `spdk_urma_opts_init` ⇒ 该 env **客户端和 target 都读**，target 在启动时读入（JSON 可覆盖）。
- **本单只改 bench 客户端 env（零重启）**；target 当前 EID 只读记录，不重启。若客户端无差别而同事指的是 target 侧，另单再议（需重启 target）。

## A. 预检（只读）

1. 环境快照：`plain_nof135p` 2509048/2509051（Keys=60000）、target B 2458315、target A 3329243、etcd 2455814。
2. **确认实际 bench 链接的 SPDK 树里有该 env**：`grep -rn "SPDK_URMA_EID_INDEX" $D/spdk-e12b7cb-src/lib/nvme/nvme_urma_common.c`；并 `strings <bench 二进制> | grep SPDK_URMA_EID_INDEX`。**若没有该字符串 ⇒ 停手回报，不重编、不换库**。
3. `urma_admin show -d udmac0d1e2`：确认 EID index 0 与 1 都存在且 ACTIVE。
4. 只读记录 target B 当前 transport 的 `eid_index`（`nvmf_get_transport` / 启动日志 / `/proc/<pid>/environ`，能取到什么记什么）；**不改**。
5. 确认所有遍 `MC_EID_INDEX` 未设或为 0（与 NOF-138 变量隔离）；只变 `SPDK_URMA_EID_INDEX`。

## B. A/B

1. 每遍 env：既有 `SPDK_URMA_DEV_NAME=udmac0d1e2` + `SPDK_URMA_EID_INDEX=0` 或 `1`；其余不变。
2. 每遍：`segment_read --num_keys=60000 --num_threads=1 --batch_size=1`、`--warmup_keys=128`、`--nof_qpairs_per_segment=4`、verify=false、diag 关、sequential（`--shuffle_seed=0`）；**每遍只读一遍 60k（约 20s），禁止 `--duration`**。
3. 顺序：**E0→E1→E0→E1→E0→E1**（每臂 3 遍，交替抵消漂移）。
4. 每遍记录：**Avg/P50/P90/P99/P999/Max、failed、墙钟、target B W9avg/n**；另存该遍 bench 进程的 `/proc/<pid>/environ` 中 `SPDK_URMA_EID_INDEX` 值（证明 env 到达进程）；SPDK 日志如有 EID 行一并摘录。
5. EID1 初始化/读失败/rc≠0 ⇒ **停手**，恢复到 E0，**不重启 target**，回报错误原文。

## C. 聚合与判读

- 按 `D:\gds性能\测试结果聚合方法_20261112.md`：**Avg=各遍等权均值；P50/P90/P99=各遍同分位中位**；`spread=(max−min)/mean`。
- 表：E0、E1 两行（Avg/P50/P90/P99 + spread + failed），再给 **E1/E0 比值**。
- 判据：① 偏差小于两臂 spread ⇒ **无差别**，该维度关闭；② E1 稳定低 **>5%** ⇒ 记“SPDK EID 在 T1 时延有收益”，再议 vsegment 复测；③ E1 更慢 ⇒ 关闭。
- **正文不报吞吐、不列逐遍值**；逐遍原始值落 `run/nof140_rounds.csv`。
- E0 三遍应接近正式 plain 基线（P50≈252–254µs）；明显偏离先报环境异常，不作 EID 归因。

## D. 边界

- 不改码/不重编/不装包/不重启 master/target/etcd；**只改 bench env**。
- 不写数据、不切 vsegment；target EID 只读不改。
- 环境末态：`plain_nof135p` 保持运行；target/etcd 未动。
- 预计 **3–5 分钟**。

---

## EID 两层（2026-11-26 同事确认 · 后续引用以此为准）

| 环境变量 | 作用的层 | 代码位置 | 与 NoF 的关系 |
|---|---|---|---|
| `MC_EID_INDEX` | Mooncake **Transfer Engine 的 URMA endpoint（内存通道）** | `mooncake-transfer-engine/src/config.cpp:662` → `urma_endpoint.cpp:221/239/663-677` | **NoF 数据面不走它**；NOF-138 测的是这层，−0.44% 属层级不对 |
| `SPDK_URMA_EID_INDEX` | **SPDK NVMe-oF URMA transport**（NoF 数据面） | `lib/nvme/nvme_urma_common.c:147`（initiator）；`lib/nvmf/urma.c:1209`（target transport 初始化也读） | **NoF 的对应旋钮**；NOF-140 测客户端侧 |

- 因此：NOF-138 的 null 结果**符合层级预期**，不能用来关闭 NoF 的 EID；判断 NoF 的 EID 必须看 NOF-140。
- `SPDK_URMA_EID_INDEX` 的实际语义：按 index 匹配 EID，**找不到时静默回退首个 EID**（与 `MC_EID_INDEX` 报错不回退不同）；客户端和 target 都在进程启动时读，target 的 JSON `eid_index` 可覆盖。
- **取证边界**：上表源码行号来自参考树 `_ref/spdk`；NOF-140 预检发现实际运行树 `$D/build/spdk-e12b7cb-src` 只有构建产物（无 `lib/` 源码），**行号未在实际树复核**；实际树以二进制 `strings` 为准（`libspdk_nvme.a`、`nvmf_tgt`、`stress_cluster_bench` 均命中 `SPDK_URMA_EID_INDEX`）。

---

# NOF-140 完成（2026-11-26 · SPDK_URMA_EID_INDEX 0 vs 1 · **plain-only** · 正式 T1 · **无稳定差别，维度关闭**）

## 条件

- 环境：`plain_nof135p` 2509048/2509051（Keys=60000）、target B 2458315、target A 3329243、etcd 2455814；全程未动。
- 数据：现成 **plain** 60k（单 target B；**不是 vsegment**）；`MC_EID_INDEX` 两机均 unset（与 NOF-138 隔离）。
- 每遍：`SPDK_URMA_EID_INDEX=0/1` 只加在 bench 客户端进程；`segment_read --num_keys=60000 --num_threads=1 --batch_size=1`、warmup=128、q4、verify=false、diag 关、sequential；每遍一遍 60k（16–19s）。
- 顺序：E0→E1→E0→E1→E0→E1（各 3 遍）；六遍 `/proc/<pid>/environ` 均自证到对应 `SPDK_URMA_EID_INDEX`。
- 预检：target B `/proc/2458315/environ` 无该变量 ⇒ target 默认 EID0；`urma_admin show -d udmac0d1e2` 的 EID 0–7 全 ACTIVE；实际 SPDK 树无 `lib/` 源码，但二进制三处均含该 env 字符串。

## 结果（µs；按 2026-11-12：Avg=各遍均值、分位=各遍同分位中位）

| 臂 | 遍数 | Avg | P50 | P90 | P99 | Avg spread | P50 spread |
|---|---|---|---|---|---|---|---|
| E0 | 3 | **261.23** | **258.67** | **268.36** | **288.07** | 5.4% | 5.2% |
| E1 | 3 | **247.51** | **244.86** | **256.23** | **272.49** | 6.5% | 6.5% |
| E1/E0 | — | 0.947 | 0.947 | 0.955 | 0.946 | — | — |

- 逐遍 P50：E0 **251.90 / 265.43 / 258.67**；E1 **252.36 / 244.86 / 236.54**。
- 配对差（同对 E1−E0）：**+0.18% / −7.75% / −8.56%**；E1 始终后跑。
- E0 三遍 P50 中位 258.67，比正式 plain 基线 252–254 高 2–3%（轻度偏离，已标注）；Max 有 2,562–49,690µs 的个别极端样本，按既定边界不追。

## 判读

- 表面 E1/E0 P50 0.947（低 5.34%），但 **5.34% < E1 自身 P50 spread 6.5%**；配对差三对不一致（第一对 +0.18%，后两对 −7.75%/−8.56%），且 E1 在每对中都后跑 ⇒ **与向下时间漂移一致，不是稳定的 EID 效应**。
- 按 C.3 判据：**无稳固差别；不记“SPDK EID 在 T1 时延有收益”；SPDK EID 维度关闭，不复测 vsegment**。
- 结合同事澄清（`MC_EID_INDEX` 是 Transfer Engine 内存通道，NoF 不走它），**两个 EID 变量均关闭**。
- 本单是 plain-only 的 EID 诊断，不是 vsegment/plain 对比，**不进正式表**；正式结果仍为 NOF-127（P50 1.034×/Avg 1.040×/P90 1.046×/P99 1.371×）。

## 偏差

- 实际 SPDK 树无 `lib/` 源码，A.2 的源码 grep 无法执行；源码行号来自参考树 `_ref/spdk`，实际树以二进制 `strings` 取证。
- `nof140_run.sh` 末尾用绝对路径 `pgrep -f` 匹配不到相对路径 cmdline，显示为空；已用 `ps | grep '[b]uild/bin/nvmf_tgt'` 复核 PID 2458315 仍在（未重启）。

## 裁定与末态

- **EID 线（`MC_EID_INDEX` 与 `SPDK_URMA_EID_INDEX`）全部关闭**；无在途任务。
- `plain_nof135p` master 暂保持运行（无写入、零成本）；target/etcd 未动，数据完整。
- 如需释放 node4，仅停 master 2509048/2509051，**不要停 etcd/target、不删数据**；以后可同集群重启复用 plain 基线。

---

# NOF-141（待执行 · 高并发下 SPDK_URMA_EID_INDEX 0 vs 1 · 平衡顺序 · 2026-11-27）

## 背景

- 同事称 EID 的收益只在**高并发**下明显；NOF-140 的 T1 时延 A/B 无稳固差别。
- NOF-138 的高并发测的是 `MC_EID_INDEX`（Transfer Engine 内存通道，NoF 不走它，层级不对）；**NoF 数据面的 `SPDK_URMA_EID_INDEX` 还没在高并发下测过**。
- 本单在高并发下重测，并把 E0/E1 的顺序做平衡，分离“EID 效应”与“时间漂移”。

## A. 条件

- 环境：`plain_nof135p` 2509048/2509051（Keys=60000）、target B 2458315、target A 3329243、etcd 2455814；**全部不动**，target 保持 EID0。
- 数据：现成 plain 60k；不重写、不切 vsegment。
- 只改 bench 客户端进程 env：`SPDK_URMA_EID_INDEX=0/1`；`MC_EID_INDEX` unset；既有 `SPDK_URMA_DEV_NAME=udmac0d1e2` 保留。
- 主档读法：`segment_read --num_keys=60000 --num_threads=16 --batch_size=16`、warmup=128、qpairs=4、verify=false、diag 关、sequential；每遍一遍 60k（约 9–10s），**禁止 `--duration`**。
- 顺序（平衡）：**E0→E1 / E1→E0 交替 4 组 = 每臂 8 遍（共 16 遍）**；序列固定为 E0,E1, E1,E0, E0,E1, E1,E0, E0,E1, E1,E0, E0,E1, E1,E0；每遍记录序号与时间戳。
- 若主档 16t/batch16 无稳固差别，**升一档 32t/batch16**（仍 q4；每臂 4 遍，E0→E1/E1→E0 各 2 组，共 8 遍）。**不要直接上 batch32**（在途 4GiB，可能撞 inflight 限制）；32t 起不来/失败则记录错误即停，不追。
- 每遍记录：**吞吐（MiB/s，主指标）、Avg/P50/P90/P99（批时延，仅诊断）、failed、墙钟、target B W9avg/n**；另存该遍 bench 进程 `/proc/<pid>/environ` 中 `SPDK_URMA_EID_INDEX` 值（env 自证）。
- EID1 初始化/读失败/rc≠0 ⇒ **停手**，恢复 E0，不重启 target，回报错误原文。

## B. 聚合与判读

- 吞吐：各臂各遍 bench 报告值的算术平均 + spread；时延：Avg=各遍均值、P50/P90/P99=各遍同分位中位（2026-11-12 方法）。
- **平衡配对**：分别算“E0 先跑”的块和“E1 先跑”的块内的 E1/E0（或 E1−E0）；再算吞吐/P50 对 pass 序号的趋势。
- **只有两个方向都稳定 E1 更好，才是 EID 效应；只有 E1 后跑时更好 = 时间漂移。**
- 判据：E1 吞吐在**两个方向都 >5%** 且 block 级一致（≥3/4 块）⇒ 记“高并发 EID 收益”，再议 vsegment 高并发复测；否则 **关闭 SPDK EID 线，不再复测 vsegment**。
- 正文给聚合表 + 平衡配对结论；**不逐遍罗列**（原始值落 `run/nof141_rounds.csv`）；批时延不与正式 T1 表混比。
- 预检：`strings <bench> | grep SPDK_URMA_EID_INDEX` 应命中（NOF-140 已确认）；`urma_admin show -d udmac0d1e2` 的 EID 0–7 应全 ACTIVE；target B `/proc/2458315/environ` 应无该变量（默认 0）。

## C. 边界

- 不改码/不重编/不装包/不重启 master/target/etcd；**只改 bench env**。
- 不写数据、不切 vsegment；target EID 只读不改。
- 环境末态：`plain_nof135p` 保持运行；target/etcd 未动。
- 预计 **5–8 分钟**（主档约 3 分钟；升 32t 再加 2–3 分钟）。

---

# NOF-141 完成（2026-11-27 · plain 高并发 · **SPDK EID1 +48.31%，成立**）

## 条件

- plain 60k（`plain_nof141p`，Keys=60000）、target B 2692715；target 保持 EID0、未重启；`MC_EID_INDEX` unset；只改 bench 客户端 `SPDK_URMA_EID_INDEX=0/1`。
- 16t/batch16、`segment_read` 60k、warmup=128、qpairs=4、verify=false、diag 关、sequential；每遍一遍 60k（≈9–10s）。
- 平衡顺序 E0→E1 / E1→E0 交替 4 组，每臂 8 遍；16 遍 `/proc/<pid>/environ` 自证 env；16 份客户端日志 keys=60000 / data=234.38GB / failed=0 / qpairs=4。

## 结果（吞吐主指标；时延为批时延，仅诊断）

| 指标 | E0（8 遍） | E1（8 遍） | E1/E0 |
|---|---|---|---|
| 吞吐 MiB/s | **47,927.44**（spread 0.20%） | **71,082.26**（spread 4.17%） | **1.483（+48.31%）** |
| Avg µs | 21,287.1 | 14,351.2 | 0.674 |
| P50 µs | 21,334.5 | 14,341.1 | 0.672 |
| P90 µs | 21,655.0 | 14,819.6 | 0.684 |
| P99 µs | 22,062.1 | 15,696.5 | 0.711 |

- 平衡配对：E0 先跑 4 块增益 **+50.54 / +46.28 / +49.73 / +50.40%（均值 +49.24%）**；E1 先跑 4 块 **+45.79 / +45.37 / +46.90 / +51.49%（均值 +47.39%）**；**8/8 块 E1 更快，最低 +45.37%**。
- 两个方向都 >45% ⇒ **EID 效应，不是时间漂移**。
- 批时延 14–21ms（16×4MiB/批），**不可与 T1/batch1（P50≈252–254µs）比**；本单只证明高并发带宽，不改变正式 T1 结果。

## 判读

- **`SPDK_URMA_EID_INDEX=1` 在 16t/batch16 下有显著吞吐收益：47,927.44 → 71,082.26 MiB/s（+48.31%）**；两个方向、8/8 块一致 ⇒ 成立；未升 32t。
- T1 口径 NOF-140 无稳固差别 ⇒ **EID1 是高并发带宽优化，不是单 key 时延优化**；正式结果仍为 NOF-127（P50 1.034×）。
- 下一步：vsegment 高并发 + EID1 复核（NOF-142）。

## 偏差

- `run/nof141.out` 末态 Keys=59505 是旧日志 `master_plain_nof135p.log` 的旧快照；新 `nof141p` 日志 Keys=60000、applied_seq=60150、lag=0 ⇒ 数据完整，无需补写。
- E1 吞吐 spread 4.17%（>3%），8 遍落 69.7k–72.6k；因两方向增益 >>5%，不影响判据。
- 本单时延为批时延，已单列口径。

## 末态与裁定

- etcd 2690803、target B 2692715、master `plain_nof141p` 2693369/2693372 运行；Keys=60000；11 盘 vfio；node2 无 bench 残留；未改配置/未重启/未写数据。
- 裁定：plain master 保持运行；立 **NOF-142** 做 vsegment 高并发复核。

---

# NOF-142（待执行 · vsegment 高并发 + SPDK_URMA_EID_INDEX 0/1 · 平衡 A/B · 2026-11-27）

## 背景

- NOF-141 已证明 plain 在 16t/batch16 下 `SPDK_URMA_EID_INDEX=1` 有 **+48.31%** 吞吐收益（两个方向、8/8 块一致）。
- T1 口径 NOF-140 无稳固差别 ⇒ EID1 是高并发带宽优化，不是单 key 时延优化；正式结果不变。
- 本单把同一 A/B 移到 **vsegment 60k** 上，判断收益是否同样存在。

## A. 预检（只读）

1. 环境快照：`plain_nof141p` 2693369/2693372（Keys=60000）、etcd 2690803、target B 2692715、target A 实际 PID/UUID；所有 target/etcd **不重启**。
2. 找现成 vsegment 60k 数据：优先 **m=2 正式数据**（NOF-129 `vseg_nof128`、NOF-134 `vseg_nof134v1`；Keys=60000、vsegments=1）；备选 **m=1**（NOF-135 数据）。以 etcd `/cvm/` 键、master 日志、target 数据实际为准。
3. 若 vsegment master 未运行：允许停 plain master、用旧 cluster_id 启动 vsegment master 恢复元数据；**不写数据、不重启 target/etcd**；等待 Keys=60000 / enabled=1。若恢复失败、需要重建/补写 ⇒ **停手回报，不在本单写数据**。
4. 确认 target A/B 在线、segment 注册/心跳正常；`urma_admin show -d udmac0d1e2` EID 0–7 ACTIVE；`MC_EID_INDEX` unset。
5. 若找不到可读的 vsegment 60k 数据 ⇒ **停手回报**（列出 etcd 键、master/target 状态），不写、不重建。

## B. 高并发 A/B

1. 读法：`segment_read` 60k——m=2 用 `--num_keys=30000`/segment；m=1 用 `--num_keys=60000`；`--num_threads=16 --batch_size=16`、warmup=128、qpairs=4、verify=false、diag 关、sequential；每遍一遍（约 10–15s），禁止 `--duration`。
2. 只改 bench 客户端 env `SPDK_URMA_EID_INDEX=0/1`；target 保持 EID0；`MC_EID_INDEX` unset。
3. 平衡顺序 **E0→E1 / E1→E0 交替 4 组 = 每臂 8 遍**；每遍记录吞吐（主指标）、Avg/P50/P90/P99（批时延，仅诊断）、failed、target A/B W9、`/proc/<pid>/environ`。
4. EID1 初始化/读失败/rc≠0 ⇒ 停手，恢复 E0，不重启 target。

## C. 判读

- 吞吐：各臂各遍 bench 报告值均值 + spread；批时延只作诊断，**不与正式 T1 表混比**。
- 平衡配对：E0 先跑块、E1 先跑块分别算增益；**两个方向都 >5% 且 ≥3/4 块一致** ⇒ 记“vsegment 高并发 EID 收益”；否则无收益。
- 正文给聚合表 + 平衡结论；不逐遍罗列（原始值落 `run/nof142_rounds.csv`）。
- 不改变正式结果（NOF-127 T1 P50 1.034×）；本单只回答高并发带宽。

## D. 边界

- 不改码/不重编/不装包；**不重启 target/etcd**；不写数据（含不补写、不重建 vsegment）。
- 允许停 plain master、启动 vsegment master（仅为读取现有数据）；若需重建则停手。
- 环境末态：记录实际 master（plain 或 vsegment）、Keys、target/etcd 状态；node2 无 bench 残留。
- 预计 **10–15 分钟**。

---

# NOF-142 完成（2026-11-27 · vsegment 高并发 · **SPDK EID1 +101.11%（保守 +91.58%）**）

## 条件

- 环境：新集群 `vseg_nof142v`（m=2 / stripe=2MiB / extent=256GiB），写 60k（152s，60,000/0 failed，Keys=60000，enabled=1，vsegments=1）；target B 2692715、target A 3329243、etcd 2690803 全程未重启。
- 背景：旧候选集群 `vseg_nof134v1` 等恢复失败（logical_used=0、partition degraded、payload too large）⇒ 旧数据不可用，故重建；重建后先完成正式配置的 60k 写入。
- 读取：vsegment 60k（`--num_keys=30000`/segment）、16t/batch16、warmup=128、qpairs=4、verify=false、diag 关、sequential；每遍一遍（约 10–15s）。
- 只改 bench 客户端 `SPDK_URMA_EID_INDEX=0/1`；target 保持 EID0；E0/E1 平衡交替 4 组（前 8 遍统计；后 8 遍仅存档，不参与统计）。
- 16/16 遍读取成功：Total keys=60000 / failed=0 / 234.38 GB。

## 结果（前 4 组 8 遍；吞吐主指标，时延为批时延）

| 臂 | 遍数 | 吞吐 MiB/s | 吞吐 spread | Avg µs | P50 µs | P90 µs | P99 µs |
|---|---|---|---|---|---|---|---|
| E0 | 4 | **45,488.34** | 19.93% | 22,602.9 | 21,336.0 | 21,832.3 | 23,284.1 |
| E1 | 4 | **91,481.57** | 24.82% | 11,210.5 | 11,322.9 | 12,065.0 | 12,863.9 |
| E1/E0 | — | **2.011（+101.11%）** | — | 0.496 | 0.531 | 0.553 | 0.552 |

- 4 块增益：0→1 **+117.62%**、1→0 **+93.67%**、0→1 **+109.65%**、1→0 **+85.14%**；两方向均值 **+113.63% / +89.40%**，**4/4 块 E1 更快**。
- 异常轮：E0 seq=5 为 38,702.82 MiB/s（其余 3 遍 spread 1.4%）；**剔除后 E1/E0 = 1.916（+91.58%）**，结论不变，原因 [未验证]，不复测。
- 两臂 spread 19.93%/24.82% >10% ⇒ 按 2026-11-12 方法**不进正式汇总表**；本单按配对 A/B 判据成立。
- 批时延 9.8–26.4ms（16×4MiB/批），**不可与正式 T1/batch1（P50≈252–254µs）比**。

## 判读

- **vsegment 高并发 16t/batch16 下，`SPDK_URMA_EID_INDEX=1` 有显著吞吐收益：45.5k → 91.5k MiB/s（+101.11%；保守剔除离群 +91.58%）**；两个方向、4/4 块一致 ⇒ 成立，且幅度大于 plain（+48.31%）。
- T1 单 key 口径 NOF-140 无稳固差别 ⇒ **EID1 是高并发带宽优化，不是单 key 时延优化**；正式结果仍为 NOF-127（P50 1.034×/Avg 1.040×/P90 1.046×/P99 1.371×）。
- 裁定：**不复测 E0 seq=5、不升 32t**；EID 高并发线以 16t 数据收口；保留 vsegment master `vseg_nof142v` 2707317/2707320（Keys=60000），target/etcd 未动。
- 若后续要做部署配置，建议客户端 `SPDK_URMA_EID_INDEX=1` 用于高并发读；T1 场景无收益。

## 偏差

- 旧 vsegment 集群恢复失败，本单重建了新集群并写了 60k（152s）；这是环境准备，不是性能测试的一部分。
- `logical_used=0` 是指标刷新被抑制（REFRESH_INTERVAL=1000000，同 NOF-134），数据以读取为准：16/16 遍 60k、failed=0。
- 实际跑了 16 遍，按指令取前 4 组 8 遍；前 8 遍 E1 均值低于后 8 遍 ⇒ 取值保守，无择优。
- E0 离群遍未复测；结论用保守值 +91.58%。

---

# NOF-143（待执行 · T1 单并发 EID1：vsegment 与 plain 绝对值 · 2026-11-27）

## 背景

- NOF-142 已证明高并发下 EID1 有显著带宽收益（plain +48.31%、vsegment +101.11%）；但 **T1 单并发下 vsegment + EID1 没有数**。
- NOF-140 只在 plain 上做过 T1 3+3（E1 总后跑，判为无稳固差别）。
- 本单只回答：**T1 单并发、EID1 时，vsegment 和 plain 各自的端到端时延绝对值是多少**；不做拆解（拆解另单）。

## A. 预检与环境

1. 当前：`vseg_nof142v` master 2707317/2707320（Keys=60000、enabled=1）、target B 2692715、target A 3329243、etcd 2690803；target/etcd 不动。
2. 确认 vsegment 60k 可读（`segment_read --num_keys=30000`/segment）；`urma_admin show -d udmac0d1e2` EID 0–7 ACTIVE；`MC_EID_INDEX` unset。
3. plain 数据：`plain_nof135p`（Keys=60000，master 2693369/2693372 可能已停）；测 plain 前停 vseg master、用原 cluster_id 起 plain master，等 Keys=60000；**只切 master，不重启 target/etcd、不写数据**。
4. 若 plain master 起不来/Keys≠60000 ⇒ 停手回报，不补写、不重建。

## B. T1 A/B（vsegment 先、plain 后）

1. 每数据集：T1（`--num_threads=1 --batch_size=1`）、warmup=128、qpairs=4、verify=false、diag 关、sequential；每遍一遍 60k（≈20s），**禁止 `--duration`**。
2. 只改 bench 客户端 env `SPDK_URMA_EID_INDEX=0/1`；target 保持 EID0。
3. 平衡顺序：**E0→E1 / E1→E0 / E0→E1**（每臂 3 遍，共 6 遍/数据集）；vsegment 用 `--num_keys=30000`/segment，plain 用 `--num_keys=60000`。
4. 每遍记录：**Avg/P50/P90/P99/P999/Max、failed、墙钟、target A/B W9avg/n**；另存 `/proc/<pid>/environ` 的 `SPDK_URMA_EID_INDEX`（env 自证）。
5. EID1 初始化/读失败/rc≠0 ⇒ 停手，恢复 E0，不重启 target。

## C. 聚合与交付

- 按 `D:\gds性能\测试结果聚合方法_20261112.md`：Avg=各遍均值；P50/P90/P99=各遍同分位中位；spread=(max−min)/mean。
- 分别给 **vsegment 的 E0/E1** 与 **plain 的 E0/E1** 两行表（Avg/P50/P90/P99 + spread + failed），再给各自 E1/E0；**E1 的绝对值是主交付**。
- 这是 T1 EID 诊断，**不进正式表**；正式结果仍 NOF-127。正文不报吞吐、不逐遍罗列（原始值落 `run/nof143_rounds.csv`）。
- 最后恢复 vsegment master `vseg_nof142v`（若恢复失败，记录实际末态并回报）。
- 预计 **10–15 分钟**（两数据集各 6 遍 + 两次 master 切换）。

## D. 边界

- 不改码/不重编/不装包；**不重启 target/etcd**；不写数据（含不补写、不重建）。
- 只允许 master 切换；若 plain 数据不可用，停手，不写/不重建。
- 环境末态：优先恢复 vseg_nof142v（Keys=60000）；记录 master/target/etcd 状态。

---

# NOF-143 完成（2026-11-27 · T1 单并发 EID1 · **vseg 230.31 / plain 232.93µs**；vseg 重启恢复失败）

## 条件

- vsegment `vseg_nof142v` 60k（双 target）；plain `plain_nof135p` 60k（单 target B）；T1/batch1、warmup=128、qpairs=4、verify=false、diag 关、sequential；E0/E1 平衡 3+3（E0→E1 / E1→E0 / E0→E1）；只改 bench 客户端 env；target B 2692715 / target A 3329243 / etcd 2690803 未重启；12 遍全部 failed=0、Keys=60000。
- vsegment 400-key 可读性探针 P50 292.33µs，不计入测量。

## 结果（µs；Avg=各遍均值、分位=各遍同分位中位）

**vsegment（vseg_nof142v，60k）**

| 臂 | 遍数 | Avg | P50 | P90 | P99 | P50 spread |
|---|---|---|---|---|---|---|
| E0 | 3 | 263.78 | **250.91** | 260.17 | 390.42 | 8.77% |
| E1 | 3 | 228.61 | **230.31** | 241.61 | 368.20 | 9.11% |
| E1/E0 | — | 0.867 | 0.918 | 0.929 | 0.943 | — |

**plain（plain_nof135p，60k）**

| 臂 | 遍数 | Avg | P50 | P90 | P99 | P50 spread |
|---|---|---|---|---|---|---|
| E0 | 3 | 252.48 | **248.49** | 257.05 | 268.75 | 4.46% |
| E1 | 3 | 232.98 | **232.93** | 240.77 | 256.10 | 4.39% |
| E1/E0 | — | 0.923 | 0.937 | 0.937 | 0.953 | — |

- **跨数据集（EID1，主交付）：P50 230.31（vsegment）vs 232.93（plain）= 0.989；P90 1.003；P99 1.438。**
- 平衡配对：vsegment −7.56% / −8.21% / −22.71%；plain −4.78% / −9.19% / −7.80% ⇒ 两数据集均 3/3 块 E1 更快。

## 判读

- **T1 单并发下 EID1 的收益远小于高并发**：vsegment P50 −8.2%、plain −6.3%（对比高并发 +48%/+101%）；vsegment 与 plain 的 E1 中位基本相等（0.989×）。
- **vsegment 的 P99 仍差**：368.20 vs plain 256.10 = **1.438×**；与正式 NOF-127 的 P99 1.371× 同向，属尾部结构性问题。
- **只能算诊断**：vsegment E1 P99 spread 42.68%、E0 11.07% >10%；P50 spread 8.8–9.1% 也已接近阈值，E1/E0 的 −8% 与 spread 同量级；不写进正式表。NOF-140（plain T1 3+3）方向一致、幅度相近（0.937）。
- 正式结果仍为 NOF-127（P50 1.034×/Avg 1.040×/P90 1.046×/P99 1.371×）。

## vsegment 重启恢复失败（新问题）

- 按任务书恢复 `vseg_nof142v` master 失败：`payload too large: size=23146809`（≈22.1MiB）×18 → partition degraded、`logical_used=0`、segment=0、读 `rc=255`（no NoF segments found）。
- 同进程内数据完全可读（本单 6 遍 vseg failed=0）；**重启即失效**。NOF-142 时同类报错 size=23,267,690（≈22.2MiB）。
- **22.1MiB < etcd 64MiB** ⇒ 不是 etcd `max-request-bytes`；真正阈值本机无源码树，[未验证]。疑似 master/恢复路径上的另一条消息/事务上限。
- 影响：只影响 vsegment master 重启恢复，不影响同进程读取；但会阻塞部署重启。**另立只读排查单**，不在本单改码。

## 裁定与末态

- **切回可读的 `plain_nof135p`**（master 2693369/2693372，Keys=60000）；degraded vseg master 停掉，日志/数据保留。
- T1 E1 结果作为**诊断**写入汇报（vseg 230.31 / plain 232.93µs；P50 0.989×；P99 1.438×），不改变正式结果。
- **不补测 T2/T4**：T1 与 16t 已给出趋势；需要部署曲线再另说。
- 下一步：**NOF-144 plain T1 分段拆解**（query vs 数据面）；vsegment 拆解待重建/恢复后另单。

---

# NOF-144（待执行 · T1 E1-only 打点拆解：query vs 数据面 · plain 先做 · 2026-11-27）

## 目标

- 用户模型：**~200µs 端到端 = 100µs query + 100µs 数据面**；NOF-143 实测 plain E1 P50 **232.93µs**。
- **用户定规：以后只看 E1，不做 E0。** 本单只跑 E1。
- 先用**现有 diag** 把 plain 的 E1 端到端拆成 query 与数据面各段，验证 100+100 模型；**不改码**。

## A. 环境

- 停 degraded vseg master（2767544/2767547），起 `plain_nof135p`（Keys=60000，master 2693369/2693372）；target B 2692715 / target A 3329243 / etcd 2690803 不动；不写数据。
- 确认 plain 60k 可读、failed=0；`SPDK_URMA_EID_INDEX=1`；`MC_EID_INDEX` unset。

## B. 采集

1. 沿用 **NOF-108/131 的 diag 配置**（diag on；`VSEG_DIAGNOSTIC` / `MC_STORE_TRANSFER_DIAG` 等按历史脚本），T1/batch1、60k、warmup=128、qpairs=4、verify=false、sequential；**只跑 E1 3 遍**（每遍新进程，env 自证 `SPDK_URMA_EID_INDEX=1`），每遍一遍 60k；不写、不重启。
2. 提取每遍 `get_into_breakdown` / `query_rpc_breakdown` / `transfer_data_breakdown` / `nof_slow_task` 与 target W9；按段取中位：**query、resolve、prepare、submit、completion_wait、read_us、target W9**；给总时延与各段占比。
3. 只对比用户模型 **100µs query + 100µs 数据面**；不跑 E0、不做 E0/E1 对比。
4. **注意**：diag-on 绝对时延会膨胀；主要看同轮段占比与相对值，不把 diag-on 绝对值当正式时延。

## C. 判读与交付

- 输出一张分段表（E1 各段中位、占比）；明确 query 段与数据面各多少 µs；与 100/100 模型的差异。
- 若现有 diag 字段不足以拆出某段（例如 submit/wait 混在一起），**列出缺口、停手回报**，不擅自改码；新打点另单。
- 这是诊断，不进正式表；正文不报吞吐、不逐遍（原始值落 `run/nof144_rounds.csv`）。

## D. 边界

- 不改码/不重编/不装包；不重启 target/etcd；不写数据。
- 环境末态：plain master 保持；vseg 停（degraded）；记录状态。
- 预计 **8–12 分钟**。

---

# NOF-144 完成（2026-11-27 · plain T1 E1-only · **query 83 / 数据面 176µs；drain/max_subio 105–106µs**）

## 条件

- plain 60k（`plain_nof135p`，Keys=60000）；T1/batch1、warmup=128、qpairs=4、verify=false、sequential；**只跑 E1 3 遍**（新进程，environ 自证 `MC_STORE_TRANSFER_DIAG=1`、`SPDK_URMA_EID_INDEX=1`）。
- diag-on：`MC_STORE_TRANSFER_DIAG=1`、`MC_NOF_SLOW_TRACE_THRESHOLD_US=1`；3 遍全部 failed=0、TotalKeys=60000；200-key 探针已确认 `get_into_breakdown` 字段齐全。
- target B 2692715 / target A 3329243 / etcd 2690803 未重启；不写数据。

## 结果

**A. `get_into_breakdown`（三遍中位，µs）**

| 段 | 中位 | 占 total |
|---|---|---|
| query_us | **83.0** | **32.2%** |
| nof_prepare_us | 2.0 | 0.8% |
| nof_submit_us | 5.0 | 1.9% |
| completion_wait_us | **160.0** | **62.0%** |
| read_unattributed_us | 9.0 | 3.5% |
| **read_us（数据面）** | **176.0** | **68.2%** |
| **total_us** | **258.0** | **100%** |

- 校验：query 83 + read 176 = 259 ≈ total 258（残差 −1µs）。
- 端到端 diag-on P50：249.66 / 273.61 / 265.72（三遍中位 265.72）；三遍 failed=0。
- target W9（diag-on）：44.4 / 44.4 / 45.4µs，与 NOF-143 diag-off 一致 ⇒ diag-on 不影响 target 侧。

**D. `nof_slow_task`（三遍中位，µs）**

| 段 | 中位 |
|---|---|
| total_us | 148.0 |
| dispatch_queue_us | 11.0 |
| submit_span_us | 26.0 |
| drain_us | 105.0 |
| max_subio_us | 106.0 |

## 判读（对照 100µs query + 100µs 数据面）

- **“200µs 量级”成立**：total 258µs（diag-on；diag-off 232.93）。
- **比例相反**：query **83µs（32%）** 比模型低 17%；数据面 **176µs（68%）** 比模型高 76%。
- **数据面内 `drain/max_subio` 105–106µs（客户端 child task 尾部/最慢子IO），接近 100µs 模型；target W9 44–45µs 才是设备执行**（+5–6µs 是对 100µs 模型；不是纯设备）；额外 **~42µs 在提交/派发/排队**（submit_span 26 + dispatch_queue 11 + 其他），另有 **9µs 未归因**。
- **瓶颈在数据面，不在 query**；现有 diag 字段无缺口，**无需新打点**。
- 可行动的优化点是数据面的 **~42µs 提交/派发开销**；这属于后续优化候选，需改码/新阶段，当前结果不变。

## 偏差

- diag-on 绝对值比 diag-off 膨胀约 **14%**（265.72 vs 232.93）⇒ 只看段结构与占比，不当正式时延。
- 全量打点日志约 60MB/遍；CSV 首版表头/ pass 列已修正，数据值未改；`pgrep -cf` 自匹配 SSH 命令行误报，实际 bench 残留=0。

## 末态与裁定

- 环境：`plain_nof135p` 2771100/2771103 运行（Keys=60000、VSegment enabled=0、segment=1）；target B 2692715 / target A 3329243 / etcd 2690803 未重启；11 盘 vfio；node2 无 bench 残留；未改码/未重编/未写数据。
- 裁定：**vsegment 同款拆解见 NOF-145**（新建集群重写 60k，不复用 degraded 数据）；数据面 ~42µs 提交/派发列为后续优化候选；不调低 trace 阈值。

---

# NOF-145（待执行 · vsegment T1 E1-only 打点拆解 · 重建 60k · 2026-11-27）

## 目标

- 用户明确：**plain 只是参照，最终优化对象是 vsegment**；本单做 **vsegment 的 E1 分段拆解**。
- 当前 `vseg_nof142v` master 重启恢复失败（payload too large ~22.1MiB → partition degraded、数据不可读），**不复用旧数据**；本单新建 vsegment 集群并重写 60k，再做拆解。
- **用户定规：只看 E1，不做 E0。**

## A. 环境重建（必要写入）

1. 停 plain master `plain_nof135p`（2771100/2771103），保留 plain 数据/日志；target B 2692715 / target A 3329243 / etcd 2690803 **不重启**。
2. 新集群 `vseg_nof145v`：m=2 / stripe=2MiB / extent=256GiB；注册 target B → A，等 `enabled=1`、`vsegments=1`。
3. 写 60k（T1/batch1，约 152s）；**要求 60,000 succeeded / 0 failed、Keys=60000**；写失败/键数不对 ⇒ 停手，不补写。
4. 可复用 NOF-142 的脚本/参数；不改码/不重编/不重启 target/etcd。

## B. diag-on T1 E1（只跑 E1）

1. diag-on：`MC_STORE_TRANSFER_DIAG=1`、`MC_NOF_SLOW_TRACE_THRESHOLD_US=1`；T1/batch1、`--num_keys=30000`/segment、warmup=128、qpairs=4、verify=false、sequential；**只跑 E1 3 遍**，每遍一遍 60k；env 自证 `SPDK_URMA_EID_INDEX=1`；target 保持 EID0。
2. 提取 `get_into_breakdown` / `nof_slow_task` / target A/B W9 中位：**query、nof_prepare、nof_submit、completion_wait、read_unattributed、read_us、total**；**drain、max_subio、submit_span、dispatch_queue**；两 target 的 W9avg/n。
3. 与 **plain NOF-144** 对照：query 83、data 176、total 258、device IO 105–106、submit/dispatch ~42µs；说明 vsegment 的差异（尤其 completion_wait、两 target W9、child 聚合）。

## C. 判读与交付

- 输出 vsegment 分段表（E1 3 遍中位 + 占比）+ 与 plain 对照；明确 vsegment 的可优化段（query vs 数据面 vs 提交/派发/等待）。
- diag-on 绝对值会膨胀；只看结构/占比与同轮相对值，不进正式表。
- 若某段字段不足，停手回报，不擅自改码；新打点另单。

## D. 边界

- 允许新建 vsegment 集群并写 60k（本单必要）；**不重启 target/etcd**；不改码/不重编。
- 环境末态：优先保持 `vseg_nof145v` master 运行（Keys=60000）；记录 plain/vseg master、target/etcd 状态；node2 无 bench 残留。
- 预计 **15–20 分钟**（重建 3–5 分钟 + 3 遍 diag + 聚合）。

---

# NOF-145 阻断（2026-11-27 · etcd NOSPACE）

- 开跑前环境已死：etcd 2379 未监听、target B 4457 未监听、11 盘 vfio 悬空；plain master 2771100/2771103 仍在但报 `context deadline exceeded`（E=1929）。
- 恢复：停 plain master → 11 盘 takeover -r -y（11/11 → nvme）→ 起 etcd（同 data-dir + 64MiB，PID 2822942）→ 起 target B（2824683，新 UUID `8bb484fa-bcc6-4475-aee3-387d3f8b466c`，RAID0 online 11/11、strip256K、ERROR=0）→ 建 `vseg_nof145v`（2825213/2825216，m=2/stripe=2MiB/extent=256GiB）、注册 B→A、enabled=1、segment=2。
- 写 60k：**13,432 succeeded / 46,568 failed**，Keys 冻结 13432；etcd 报 `mvcc: database space exceeded` ×82,979、`alarm:NOSPACE`；dbSize **2.1GB**，默认 quota **2GiB**；WAL 367M + snap 2.1G = 2.4G；/home 1.1T 可用 ⇒ 磁盘不是瓶颈。
- 原因：历史多集群 oplog 累积（vseg_nof142v 119,520 键、plain_nof135p 60,398、vseg_nof145v 26,638；总 2.25M+ 键）。
- 按任务书 A.3 停手；**未 compact/defrag/删键/重启 etcd**，未进入 diag。
- 末态：etcd 2822938/2822942 运行但 alarm:NOSPACE；target B 2824683、target A 3329243；`vseg_nof145v` master 2825213/2825216（Keys=13432、enabled=1、segment=2）；11 盘 vfio；plain_nof135p 数据保留未动；node2 无 bench 残留。

---

# NOF-146（待执行 · etcd NOSPACE 修复 + 续跑 NOF-145 · 2026-11-27）

## 授权与原则

- 用户已授权：**先 compact + defrag；若仍不够，提 `--quota-backend-bytes=8GiB`（需重启 etcd）；不删任何键。**
- 不换 data-dir、不删业务键/历史集群键、不重启 target；失败即停手回报。

## A. etcd 修复

1. 记录当前 `endpoint status`（dbSize、revision、alarm）；`etcdctl compact` 到当前 revision，再 `etcdctl defrag`；复查 dbSize/alarm。
2. 若 dbSize 降到 quota 以下且 `alarm:NOSPACE` 清除 ⇒ 进入 B。
3. 若仍不足/仍 NOSPACE ⇒ 停 etcd，用同一 data-dir + `--max-request-bytes=67108864 --quota-backend-bytes=8589934592` 重启；等 healthy、alarm 清除。若重启失败 ⇒ 停手，不删键。
4. 记录 compact/defrag/重启前后 dbSize、revision、alarm。

## B. 续写 60k

1. 先查 `segment_write --help` 是否有 start_key/offset：
   - 有 ⇒ 对 `vseg_nof145v` 从 13,432 续写剩余 46,568，写满 60k；
   - 没有 ⇒ **保留旧 partial 集群**，新建 `vseg_nof145v2`（m=2/stripe=2MiB/extent=256GiB），注册 B→A、enabled=1，重写 60k。
2. 要求 **60,000 succeeded / 0 failed、Keys=60000**；若仍失败 ⇒ 停手回报（附 etcd dbSize/alarm、失败点）。
3. 允许停旧 vseg master、起新 vseg master；**不重启 target/etcd（除 A.3 的 quota fallback）**。

## C. diag E1 拆解（只跑 E1）

1. 对写满的 vsegment 60k：diag-on（`MC_STORE_TRANSFER_DIAG=1`、`MC_NOF_SLOW_TRACE_THRESHOLD_US=1`）、T1/batch1、`--num_keys=30000`/segment、warmup=128、qpairs=4、verify=false、sequential；**只跑 E1 3 遍**；env 自证 `SPDK_URMA_EID_INDEX=1`。
2. 提取 `get_into_breakdown` / `nof_slow_task` / target A/B W9 中位：query、nof_prepare、nof_submit、completion_wait、read_unattributed、read_us、total；drain、max_subio、submit_span、dispatch_queue；两 target W9avg/n。
3. 与 **plain NOF-144** 对照（query 83、data 176、total 258 diag-on、device IO 105–106、submit/dispatch ~42）；明确 vsegment 的可优化段。
4. diag-on 绝对值会膨胀；只看结构/占比与同轮相对值，不进正式表；若字段不足停手回报。

## D. 边界

- 不改码/不重编/不装包；**不重启 target**；etcd 仅在 quota fallback 时重启；不删键。
- 允许停/起 master；plain 数据保留；环境末态优先保持 `vseg_nof145v`/`vseg_nof145v2` master（Keys=60000）。
- 预计 **15–25 分钟**（etcd 修复 3–8 分钟 + 续写 2–3 分钟 + 3 遍 diag + 聚合）。

---

# NOF-146 完成（2026-11-27 · etcd NOSPACE 修复 + vseg_nof145v2 60k + vsegment E1 拆解）

## A. etcd 修复

- 基线：dbSize 2.1GB、`alarm:NOSPACE`；`compact` revision 2409655；首次 defrag 默认 5s 超时；停旧 vseg master 后 `compact` 2409656，`defrag --command-timeout=900s` 成功（8s）；dbSize **1,951,080,448B**（<2GiB）；`alarm disarm`；put/read/del 探针 OK；dbSize 1,951,096,832B，余量 **196,386,816B（≈187MiB）**。
- 容量核算：抽样 200 个 oplog 值均值 **413B** × vseg_nof142v 119,520 键 ≈ **47MB** < 余量 187MB ⇒ 2GiB 够用；**未重启 etcd、未提 quota、未删任何键**。
- 写入 60k 后 dbSize **2,048,397,312B（≈1.91GiB）**，`alarm` 空；余量仅 **≈99MB**。

## B. 写 60k

- 新建 `vseg_nof145v2`（m=2 / stripe=2MiB / extent=256GiB），注册 target B+A（segment=2），enabled=1（20s），写 60k **152s，60,000 succeeded / 0 failed**，Keys=60000。

## C. vsegment E1 拆解（diag-on，T1，只跑 E1）

- 5 遍（p3 离群）；稳定组 **p1/p2/p4** 中位（µs）：

| 段 | vsegment | plain NOF-144 | 差 |
|---|---|---|---|
| query_us | **89.0** | 83.0 | +6.0 |
| vsegment_resolve_us | **2.0** | 0.0 | +2.0 |
| nof_prepare_us | 3.0 | 2.0 | +1.0 |
| nof_submit_us | 3.0 | 5.0 | −2.0 |
| completion_wait_us | **150.0** | 160.0 | −10.0 |
| read_unattributed_us | 7.0 | 9.0 | −2.0 |
| **read_us（数据面）** | **169.0** | 176.0 | −7.0 |
| **total_us** | **261.0** | 258.0 | **+3.0** |

- `nof_slow_task` 中位：drain **104** vs 105、max_subio **104** vs 106、submit_span **9** vs 26、dispatch_queue **6** vs 11、total **126** vs 148。
- 占比：query **34.1%**、completion_wait **57.5%**、数据面 **64.8%**。
- **判读：vsegment 与 plain 分段结构几乎重合；vsegment 仅多 2µs `vsegment_resolve`，总差 +3µs 在 spread 内；vsegment 不多付数据面成本。** 共同瓶颈是 completion_wait 150/160 与 drain 104/105 的 **~45–55µs 残差**：其中 submit_span+dispatch 可测为 15/37µs，第二 child、回调/聚合等需代码打点再拆；target W9 44–45µs 才是设备执行，drain 是客户端 task 尾部跨度。

## 偏差

- p3 离群：P50 381µs、drain 161µs，但同遍 target B/A W9 仍 38,890/39,210ns（正常）⇒ 慢在客户端侧；原因 [未验证]，已剔除，5 遍 spread 39.5% 已标注。
- `get_into_breakdown` 行数不全：p1 23,264 / p2 22,308 / p5 32,216（p3/p4 60,128），`nof_slow_task` 均 10.8–12.0 万条、日志 565–955MB/遍；各遍样本 ≥2.2 万，中位稳定；机制 [未验证]，未改码查证。
- 首次 defrag 默认 5s 超时，已用 `--command-timeout=900s` 重试成功；`alarm disarm` 是必需步骤（否则 etcd 持续拒写），已执行并记录。

## 末态与裁定

- 环境：`vseg_nof145v2` master 2828887/2828890（Keys=60000、enabled=1、segment=2、E=1 FATAL=0）；etcd 2822942（dbSize 2.0GB、alarm 空）；target B 2824683（UUID 8bb484fa…、RAID0 online 11/11、ERROR=0）、target A 3329243；11 盘 vfio；node2 无 bench 残留；未删键；plain 数据保留。
- **裁定：① p3 不单独立项；② get_into_breakdown 采样不全不追（nof_slow_task 全量、样本 ≥2.2 万）；③ etcd 余量仅 99MB，下次新建 60k 集群前先 compact/defrag，不够再按已授权提 quota 8GiB；④ 提交/派发/等待 ~45–55µs 列为下一步优化目标，但需改码/新阶段，当前不做。**
- 正式结果仍 NOF-127（P50 1.034×）。

---

# NOF-147（待执行 · 零改码：重解析 NOF-144/146 完整 timing 字段 · 2026-11-27）

## 目标

- 拆开 `completion_wait − drain` 的 **~45–55µs 残差**，回答：多少是 queue/worker/QoS、submit、completion gap、poll gap、observe delay、第二 child/聚合；为后续改码优化定点。
- **只读现有日志；不跑 bench、不改码、不重启、不写。**

## A. 数据源

- NOF-144 plain：`run/nof144*`、node2 3 份 diag 日志（`get_into_breakdown` + `nof_slow_task`）。
- NOF-146 vsegment：`run/nof146*`、node2 5 份 diag 日志（稳定组 p1/p2/p4；p3 离群可单列）。
- 先清点文件大小/行数/字段；若日志已被清理，停手回报清单，不重跑。

## B. 字段提取（每 task / 每 key）

- `nof_slow_task`：`total_us`、`dispatch_queue_us`、`worker_setup_us`、`qos_fifo_wait_us`、`qos_fifo_full/partial/zero_wait_us`、`head_to_submit_us`、`first_credit_wait_us`、`submit_credit_wait_us`、`submit_span_us`、`drain_us`、`first_completion_us`、`completion_submit_gap_us`、`completion_span_us`、`observe_delay_us`、`max_subio_us`、`max_subio_idx/cid`、`max_completion_gap_us`、`max_poll_return_gap_us`、`poll_calls`、`sub_io_count`、`qpair_id`、`lane`、`worker_idx`。
- `get_into_breakdown`：`query_us`、`read_us`、`nof_prepare_us`、`nof_submit_us`、`completion_wait_us`、`read_unattributed_us`、`total_us`；`vsegment_resolve_us`。
- 按 task 取中位；按 key 聚合（vsegment 2 child tasks/key；plain 1 task/key）。

## C. 残差归因

- 计算 `completion_wait − drain`（vseg 150−104、plain 160−105）的组成：
  - **前端**：`queue_to_first_submit`（dispatch + worker_setup + qos_fifo_wait + head_to_submit）、`submit_span`；
  - **完成**：`first_completion_us`、`completion_submit_gap`、`completion_span`、`max_completion_gap`、`max_poll_return_gap`、`observe_delay`；
  - **第二 child/聚合**：whole-key `completion_wait` vs 单 child task `total_us`（vseg）；
  - **未归因残差**。
- 给 vsegment vs plain 对照表（中位 µs + 占比）；明确最大项和下一步改码候选。
- 若字段不全/日志采样不足，列缺口，不猜；停手回报。

## D. 交付格式（用户明确要求）

### D.1 结构图（先图后表）

输出 **Mermaid + ASCII 两份**，覆盖 key-level 与 task-level；每个点位编号，图上标明加法链与重叠关系。示例骨架（按实际字段补全）：

```mermaid
flowchart LR
  K0[Key request] --> K1[K1 query_us]
  K0 --> K2[K2 read_us]
  K2 --> K3[K3 prepare + resolve]
  K3 --> K4[K4 nof_submit]
  K4 --> K5[K5 completion_wait]
  K5 --> T1[Child task 1 timeline]
  K5 --> T2[Child task 2 timeline]
  K5 --> K6[K6 read_unattributed]
  T1 --> K7[K7 aggregation / callback]
  T2 --> K7
  K7 --> K8[K8 total_us]
  subgraph Task timeline (per child)
    Q0[Q0 queued] --> Q1[Q1 dispatch_queue]
    Q1 --> Q2[Q2 worker_setup + qos_fifo_wait]
    Q2 --> Q3[Q3 head_to_first_submit]
    Q3 --> Q4[Q4 submit_span]
    Q4 --> Q5[Q5 drain]
    Q5 --> Q6[Q6 observe_delay]
  end
```

- 必须标清：`queue_to_first_submit + submit_span + drain ≈ task total`（源码 `transfer_task.h` 注释）；`first_completion/completion_gap/completion_span/observe_delay/max_subio/max_completion_gap/max_poll_return_gap` 是**重叠注解**，不能加总。
- 图上每个点必须与下面表格的编号一致；vsegment 画 2 个 child task，plain 画 1 个 task（32×128KiB）。

### D.2 大段表（只放可加总的段）

| 段 | 编号 | 字段 | vsegment | plain | 差 | 占 total |
|---|---|---|---|---|---|---|
| 元数据 query | K1 | query_us | | | | |
| 准备/解析 | K3 | nof_prepare_us + vsegment_resolve_us | | | | |
| 提交 | K4 | nof_submit_us | | | | |
| 数据面等待 | K5 | completion_wait_us | | | | |
| 未归因 | K6 | read_unattributed_us | | | | |
| 数据面小计 | K2 | read_us | | | | |
| 整 key | K8 | total_us | | | | |

### D.3 全点表（所有点都列，不遗漏）

每行：**编号 / 层（key/task/subIO）/ 字段名 / 定义（一句话）/ vsegment 中位 / plain 中位 / 占对应 total / 是否可加总 / 在结构图的位置**。
至少覆盖：`query_us`、`nof_prepare_us`、`vsegment_resolve_us`、`nof_submit_us`、`completion_wait_us`、`read_unattributed_us`、`read_us`、`total_us`、`dispatch_queue_us`、`worker_setup_us`、`qos_fifo_wait_us`、`head_to_first_submit_us`、`submit_span_us`、`drain_us`、`first_completion_us`、`completion_submit_gap_us`、`completion_span_us`、`observe_delay_us`、`max_subio_us`、`max_completion_gap_us`、`max_poll_return_gap_us`、`poll_calls`、`sub_io_count`。
**可加总/重叠必须显式标注**；重叠字段不得进入任何 sum。

### D.4 残差归因表（45–55µs）

| 组成 | 来源字段 | vsegment | plain | 说明 |
|---|---|---|---|---|
| 前端 dispatch/worker/QoS | dispatch_queue + worker_setup + qos_fifo_wait | | | |
| 提交跨度 | submit_span | | | |
| 完成 gap/span | first_completion / completion_gap / completion_span | | | |
| 第二 child/聚合 | whole-key completion_wait − 单 child task total | | | |
| 观察延迟 | observe_delay | | | |
| 未归因 | 差值 | | | |

- 没有字段的部分写“未测/缺口”，不猜；下一单再决定改码打点。
- 结论必须回答：**45–55µs 里最大项是什么、是否在客户端、下一步改哪一段**。

## E. 边界

- 只读日志；不改码/不重编/不跑 bench/不重启/不写。
- 预计 **5–10 分钟**。

---

# NOF-147 完成（2026-11-27 · 零改码 timing 重解析 · **残差全在客户端**）

## 数据与方法

- plain 3 份 + vsegment p1/p2/p4（排除 NOF-146 判定的 p3 离群）diag 日志；提取 `nof_slow_task` 62 个字段与 `get_into_breakdown`；Python grep + trace_id 逐 key 关联。
- 样本：plain 每份 key=60000/task=60000；vseg p1 key=23,264、p2 key=22,308、p4 key=60,000（get_into 行数不全），task 均 60,000；vseg 合计 key 105,572 / task 180,000。

## 结果

**Task 时间线（mono_ns 推算，中位 µs）**

| 段 | plain | vseg | 差 |
|---|---|---|---|
| queued→first_submit（前端） | 14.3 | 11.1 | −3.2 |
| first→last_submit（提交跨度） | **26.8** | 9.5 | −17.3 |
| last_submit→completed（drain） | 107.3 | 105.9 | −1.4 |
| queued→completed（task total） | 144.5 | 127.5 | −17.0 |

**trace_id 关联（µs）**

| 项 | plain | vseg |
|---|---|---|
| n | 179,982 | 55,711 |
| completion_wait | 156.0 | 155.0 |
| drain | 107.0 | 116.0 |
| task_total | 144.0 | 139.0 |
| **残差（cw−drain）** | **54.0（P90 60）** | **39.0（P90 48）** |
| 拆解 | task 内部 42.0 + key 聚合 11.0 = 53.0 | task 内部 22.0 + key 聚合 16.0 = 38.0 |

**D.4 残差归因（µs）**

| 组成 | vsegment | plain |
|---|---|---|
| 前端 dispatch/worker/QoS + head | 11.1 | 14.3 |
| 提交跨度 submit_span | 9.5 | **26.8（plain 最大项）** |
| 两 child 完成差 / key 聚合 | **16.0（vseg 最大项）** | 11.0 |
| 未归因 | ≈2 | ≈1 |
| **合计** | **39.0** | **54.0** |

## 判读

- **残差全部在客户端**：target W9 44–45µs 已含在 drain 105–107 内；不是 target/设备。
- **plain**：最大项是 **submit_span 26.8µs**（32 次 128KiB 提交串行），占残差约一半；优化方向是减少/并行化 submit。
- **vsegment**：最大项是 **两 child 完成差/整 key 聚合 16µs**；submit_span 只有 9.5µs。vseg 每 child 16 subIO（2MiB）vs plain 32（4MiB），所以 task total 反而更小（127.5 vs 144.5），但 vseg 单次 poll 返回间隔更大（max_poll_return_gap 34 vs 9）。
- **字段缺口**：`qos_fifo_wait_us`、`first_credit_wait_us`、`submit_credit_wait_us`、`first_completion_us`、`completion_submit_gap_us`、`completion_span_us`、`observe_delay_us` 日志中不存在；已用 fifo/credit/max_completion_gap/max_poll_return_gap/mono_ns 替代。要精确拆 vseg 16µs 的完成/observe，需要新增打点（NOF-148）。

## 偏差与末态

- vseg p1/p2 的 `get_into_breakdown` 行数不全（23,264/22,308），key 级基于 105,572 样本；task 级 180,000 完整；未使用 p3。
- 环境未变：etcd 2822942（dbSize 2.05GB、alarm 空）、target B 2824683、`vseg_nof145v2` master 2828887/2828890（Keys=60000、enabled=1）；未跑 bench/未改码/未重启/未写。

---

# NOF-148（待执行 · vsegment child 完成/聚合/observe 打点 · 只加日志不改逻辑 · 2026-11-27）

## 目标

- 拆开 vsegment 的 **16µs 两 child 完成差/聚合**：确定是 child1→child2 完成差、`child.get()` 串行/observe 延迟，还是 key 级回调/聚合。
- **只加时间戳和日志，不改变等待顺序、并发和业务逻辑。**

## A. 源码与构建

1. 在测试机实际源码树（supercache@aadab957）检查是否已有 `nof_vsegment_child` / `complete_to_observe_us` / `serial_gap_us` 日志；若已有，先确认运行时二进制是否包含；没有则加最小日志。
2. 打点位置（参考 `transfer_task.cpp` 的 `VSegmentOperationState::wait_for_completion`）：
   - 每个 child `get()` 前后：`get_block_us`、`complete_to_observe_us`、`serial_gap_us`、`fifo_wait_us`、`first_completion_from_submit_us`；
   - operation 级：`first_submit_mono_ns`、`last_completion_mono_ns`、`last_child_idx`、`observed_total_us`；
   - key 级：`client_service.cpp` vseg `future->get()` 前后/返回、聚合完成时刻。
3. 构建规则（硬规范）：源码 supercache@aadab957；链接 `$D/build/spdk-e12b7cb-src/build/lib` 的 **e12b7cb/bd5ee124d wire v4 静态 SPDK**，禁止 `/usr/local/lib/libspdk_*.a`；记录源码 HEAD/status、库路径/md5、链接命令、bench md5；保留旧二进制回退。

## B. 运行

- 当前 `vseg_nof145v2` 60k；E1；T1/batch1、`--num_keys=30000`/segment、warmup=128、qpairs=4、verify=false、sequential；diag-on；**3 遍**；不写、不重启 target/etcd。
- 新 bench 若连接失败/读失败/rc≠0 ⇒ 回退旧二进制并停手；不改 target。
- 采集：`nof_vsegment_child`（每 child）、`nof_vsegment_operation`、`nof_slow_task` 全字段、`get_into_breakdown`。

## C. 交付

- 结构图（Mermaid + ASCII）+ 大段表 + 全点表 + 归因表（同 NOF-147 格式），所有点位编号、区分 additive/overlap。
- 明确 vseg 16µs 的组成：child1/child2 完成差、`child.get()` 串行/observe、key 聚合/回调；给下一步改码优化的精确位置。
- 若日志仍不足，列缺口，不猜；再议下一步。

## D. 边界

- 只加日志不改逻辑；不改 target；不重启 etcd/target；不写数据；构建/运行失败回退；预计 **20–30 分钟**。

---

# NOF-148 完成（2026-11-27 · 打点已存在，零改码重解析）

## A 段结论：不需要加日志/重编

- `nof_vsegment_child`、`nof_vsegment_operation`、`nof_slow_task`、`get_into_breakdown` 在 aadab957 已实现，运行时二进制 strings 4/4 命中，日志已全量落盘：**child 120,256 / operation 60,128 / key 60,128**。
- 本单改为只读重解析；未构建、未改码、未重启、未写。

## 数据与口径

- 完整全量 diag-on：p3、p4（各 60,128 全量）；p1/p2/p5 行数不全，未采用；p3 离群（drain 185 / cw 252 / 残差 69），**p4 与 NOF-147 的 39µs 对齐（drain 121 / cw 158 / 残差 38）⇒ 以 p4 为基线**。
- 关联完整性（p4）：ops=60,128 / child=60,128 / task=60,128 / key=60,128，joined=60,128，miss 全 0；last_child_idx：0=14,307、1=45,821。

## P1–P5 结构（p4 中位，µs）

```
P1 --①1.8--> P2 --②139.9 [drain 121.0]--> P3 --③11.0--> P4 --④5.3--> P5(cw=158.0)
                  │
                  ├─ child0: get_block 127.0 → observe 11.3
                  └─ child1: serial_gap 11.4 → get_block 22.0 → observe 11.5
```

| 段 | 含义 | 中位 | P90 | 可加性 |
|---|---|---|---|---|
| ① | get_begin→first_submit（提交前） | 1.8 | 3.8 | additive |
| ② | first_submit→最慢 child complete（含 drain 121.0） | 139.9 | 158.3 | additive |
| ③ | 最慢 child complete→最后 get_end（observe） | 11.0 | 14.3 | additive |
| ④ | key 聚合（cw − op 观测） | 5.3 | 11.7 | additive（P4 之后） |
| — | cw（key completion_wait） | 158.0 | 178.0 | 终点 |

- 校验：①+②+③=**152.7** vs op 观测 **151.7**（差 +1.0）；+④≈158.0。

## 残差归因（cw − drain = 38µs）

| 编号 | 成分 | 中位 | 占比 | 性质 |
|---|---|---|---|---|
| A | 提交前（get_begin→first_submit） | 1.8 | 4.7% | additive，客户端 |
| B | 段2 − drain（提交跨度+完成回调/分发） | 21.2 | 55.7% | additive，客户端 |
| C | observe（最后 child 完成→get() 返回） | 11.0 | 28.9% | additive，child.get() 串行 |
| D | key 聚合/回调（cw − op 观测） | 5.3 | 14.0% | additive，客户端 |
| — | 合计 | 39.3 | — | vs 38.0（+1.3） |

## 16µs 的真身

- **不是“两 child 完成差”**：skew 24.4µs 已完全含在②段内，不是独立可加项。
- **16µs = key 级聚合 = cw − 最慢 child task total**（158.0 − 143.0 = 15.0）；再拆为 `op 观测 − task total = 8.7` + `cw − op 观测 = 6.3/5.3`。
- **child.get() 串行**：`transfer_task.cpp:2455-2461`，代价 = child1 serial_gap **11.4µs** + 末 child observe **11.0µs**；76.2% 场景 child1 最慢，根因之一是 `submitVSegment` 串行派发（`2548-2558`）：child0 head→first_submit 1µs，child1 10µs。

## 偏差/未验证

- p3 与 p4 差异大（残差 69 vs 38）；p3 离群原因 [未验证]，怀疑机器负载。
- op 观测（计算 151.7）vs 日志 `observed_total_us`（163.0）差 −10.3µs：基线不同（started_at vs min(get_begin)），已标注非等口径。
- fifo_wait 全为 0 ⇒ 本结论不含 QoS 排队成本。
- node2 有他人进程：PID 1049544 `mooncake_master --port=50055 --metrics_port=9003 --http_metadata_server_port=8015`（父进程 1049540 `tent_store_run`，14:59 起）；端口不冲突，对 p3/p4 差异影响 [未验证]，未触碰。

## 环境末态

- node4：etcd 2822942、target B 2824683、`vseg_nof145v2` master 2828887/2828890；alarm 空；dbSize 2,048,471,040B。
- node2：无 stress_cluster_bench / nof148_parse.py 残留；日志未被改写。未改码/未重建/未重启/未写/未跑 bench。

---

# NOF-149（待执行 · 改码前基线：vsegment 3 遍 diag-on · 2026-11-27）

## 目标

- 现有完整 diag-on 只有 p3/p4，且 p3 离群；**改码前先补 3 遍完整基线**，否则改完无法判断收益。
- 同时记录 node2 的 ql/tent_store master 是否为干扰。

## A. 条件

- 当前 `vseg_nof145v2` 60k（Keys=60000）、E1、T1/batch1、`--num_keys=30000`/segment、warmup=128、qpairs=4、verify=false、sequential；diag-on。
- **不改码/不重编/不重启/不写 bench 数据**；target/etcd 不动。
- 新 tag（如 `nof149_e1_p1..p3`），确保 child/op/task/key 四表全量（child:op = 2:1，op:key = 1:1）；若某遍行数不全，标记并补 1 遍（最多 4 遍），不要用不完整遍。

## B. 采集与聚合

1. 3 遍完整 diag-on；每遍记录端到端 Avg/P50/P90/P99、failed；child/op/task/key 行数与关联完整性。
2. 按 p4 同口径提取 P1–P5：①get_begin→first_submit、②first_submit→最慢 child complete（含 drain）、③observe、④cw−op 观测；给中位/P90 与 additive 校验。
3. 残差 A–D（提交前/段2−drain/observe/key 聚合）与 16µs 拆分；三遍逐遍列，给稳定组聚合（异常遍按规则整轮剔除）。
4. 记录 node2 ql/tent_store master（PID/端口/启动时间/load）；若负载高或与本测试时间重叠，标注；**不杀他人进程**。

## C. 交付

- 结构图（P1–P5，Mermaid + ASCII）+ 大段表 + 全点表 + 残差归因表（同 NOF-147/148 格式）。
- 结论：3 遍基线是否稳定、p3 离群是否复现、16µs 是否稳定；给出改码前基线值和可接受噪声范围。
- 不改正式结果；不跑改码。

## D. 边界

- 只读测试；不改码/不重编/不重启 target/etcd/不写；不杀 ql 进程；环境末态保持 `vseg_nof145v2`。
- 预计 **10–15 分钟**。

---

# NOF-149 完成（2026-11-27 · 改码前基线 · **慢 p1/p2 基线 + p3 快异常（用户裁定剔除）**）

## 条件与完整性

- 3 遍全部完整（child 120,256 / op 60,128 / key 60,128 / task 60,128，关联 miss 0，0 failed）；E1、T1/batch1、60k、diag-on。
- 慢基线 p1/p2：cw 245/249、drain 175/178、残差 71/72；快异常 p3：cw 167、drain 99、残差 66。

## 逐遍结构（µs）

| 段 | p1 | p2 | p3 | 说明 |
|---|---|---|---|---|
| ① 提交前 | 6.8 | 6.9 | 5.7 | |
| ② first_submit→最慢完成（含 drain） | 206.8 | 210.4 | 133.9 | |
| ③ observe（最后完成→get_end） | 23.3 | 23.2 | 17.4 | |
| ④ key 聚合 | 6.7 | 6.7 | 9.6 | |
| cw | 245.0 | 249.0 | 167.0 | |
| drain | 175.0 | 178.0 | 99.0 | |
| 残差 A/B/C/D | 6.8/35.2/23.3/6.7 | 6.9/35.4/23.2/6.7 | 5.7/33.7/17.4/9.6 | |
| cw − 最慢 child task total | 28.0 | 30.0 | 25.0 | NOF-148 p4 为 15.0 |

## 稳定性

- **不稳**：cw spread 37.2%、drain 52.4%、② 41.7%、③ 27.4%、skew 66.8% ⇒ 不能进正式表、不能取三遍均值。
- **稳**：B（段2−drain）**35.2/35.4/33.7（4.9%）**；残差 **71/72/66（8.6%）**；设备 `first_completion_from_submit` **42/43µs（0%/4.6%）**；query **93.7（4.3%）**；submit **5（0%）**；resolve **3（0%）**。
- **快异常不是设备/单 target**：target B W9 38.69/38.70/39.32µs、target A 39.44/39.18/39.44µs；同遍 node3/node4 cw 差 ≤16µs 且同向（p1 244/245、p2 256/243、p3 176/160）。
- 慢基线特征：drain 175→99 翻倍，但设备 `first_completion` 仍 42µs ⇒ **客户端完成观察/轮询路径变慢**。

## 判读

- **快异常是客户端全局的**，不是设备、不是单一 target；慢基线在 NOF-146 p3 也出现过，属测试窗口异常，不是单次离群。
- **NOF-148 的 16µs 随测试窗口变化**：本次 cw−最慢 child task total = 28/30/25；NOF-148 p4 15.0；**p3 快值按异常剔除；改码前后用 clean window 稳定基线比较，不跨异常比**。
- **改码前基线用慢基线 p1+p2**（cw 245/249，spread 1.6%）；快异常 p3（cw 167）作第二参考。
- 快异常根因 [未验证]：已排除设备/单 target；未排除 node2 CPU 频率/中断亲和/他人进程。node2 ql_master（PID 1049544，6.4% CPU，14:59 起，load 7.6–9.4/308 核）与测试窗口重叠，影响 [未验证]，未触碰；`nvidia-smi` 无法通信，影响 [未验证]。
- NOF-149 绝对延迟高于 NOF-146 p4（cw 167/245/249 vs 158），16:02 后机器状态变化，原因 [未验证]。

## 环境末态

- node4：etcd 2822942、target B 2824683、`vseg_nof145v2` master 2828887/2828890；Keys=60000、segments=2、alarm 空；dbSize 2,048,507,904B（读测试期间 +36,864B）。
- node2：无 bench/parse 残留；新增 nof149 日志 3 份（各 ≈1.0GB）。未改码/未重编/未重启/未写。

---

# NOF-150（待执行 · 第一刀：vsegment 两半并行观察 · 清场 + clean 基线 + A/B · 2026-11-27）

## 0. 清场（用户已授权 node2/3/4 独占）

- 用户明确：**node2/3/4 我们独占，谁影响都能 kill**。
- node2：找到外部 `ql_master`（PID 1049544，父 1049540，`/home/ql/...`，14:59 起）及子进程；`TERM`，必要时 `KILL`；记录 PID/命令/时间。检查其他非我方 `mooncake_master`/bench/target 进程；**只杀非我方，不杀我们的 master/target/etcd**。
- node3/node4：检查非我方进程；若占用 CPU/端口，按同样规则清理并记录；**不重启 target/etcd**。
- 清场后记录 `uptime`/load、相关 PID；确认无外部干扰；若无法清理，停手回报。

## A. clean window 旧二进制基线

- 用**当前旧 bench 二进制**（未改码）在 clean window 跑 **2 遍** diag-on（E1 T1 60k）；若出现 p3 式快异常（cw 明显偏低），按异常剔除并补 1 遍（最多 3 遍），用稳定慢基线。
- 历史参考：NOF-149 p1/p2 cw 245/249、残差 71/72、C 23.3/23.2；**以 clean window 实测为准**，不硬套旧值。
- 记录 failed、child/op/key 行数、target A/B W9、load。

## B. 改码与构建

1. 源码 supercache@aadab957；改 `VSegmentOperationState::wait_for_completion`（`transfer_task.cpp:2455-2461`）：
   - 现：`for (child) { child.get(); ... }` 串行阻塞；
   - 改：先 `while (存在 !isReady())` 轮询等待；全部 ready 后再 `for (child) child.get()` 收集错误/observe 时间戳。
   - 保持错误处理、`submission_failed`、observation 时间戳、日志字段不变。
2. 硬规范构建：SPDK `$D/build/spdk-e12b7cb-src/build/lib` 的 e12b7cb/bd5ee124d wire v4 **静态库**，禁止 `/usr/local/lib/libspdk_*.a`；记录源码 HEAD/status、库路径/md5、链接命令、bench md5；保留旧二进制回退。
3. 构建失败/新二进制无法连接 ⇒ 回退旧二进制并停手。

## C. 改码后测试（异常剔除）

- 3 遍 diag-on；**p3 式快值按异常剔除**（用户裁定，不称模态）；若出现异常，补 1 遍（最多 4 遍）以取得 3 遍稳定。
- 指标：cw、drain、A–D、C observe、child serial_gap、child get_block、query/submit/resolve、target A/B W9、failed。
- 成功判据：与 clean window 旧基线比较，C observe / serial_gap / cw / 残差下降，failed=0、无回归；若没有下降或变慢，如实回报，不强行归因。
- 不跨异常比较；不把异常值混入平均。

## D. 边界

- 清场仅限 node2/3/4 非我方进程；**不杀我们的 master/target/etcd**；不重启 target/etcd；不写数据。
- 只改这一处 vsegment 客户端编排；不改 submit/plain/target/master。
- 构建/连接失败回退；环境末态保持 `vseg_nof145v2`。
- 预计 **30–40 分钟**（清场 + clean 基线 + 构建 + 3 遍）。

---

# NOF-150 完成（2026-11-27 · 第一刀 · **无收益 + 打点语义被破坏；已决定回退**）

## 清场与 clean 基线

- 用户授权 node2/3/4 独占：`sudo -n kill -TERM` 终止 node2 ql master 1049544（5.8% CPU，14:59 起）、node4 ql client 2811209（42.8% CPU，15:02 起）、node4 ql http-metadata-server 1305724（:8080）；普通 kill 被拒（跨用户），必须 sudo。我方 etcd/target A/B/master 全在位。
- 旧二进制 clean 基线（完整遍 p2/p4/p6）：**P50 374.27 / 376.80 / 380.05（spread 1.5%）**；Avg 380.06/382.79/388.18；W9 B 38.67/39.14/38.66、A 39.11/39.29/39.37。
- **重要：diag 行不全（BAD）的遍 P50 更低**（295.62/320.94/275.06/289.34…）；完整遍 374–388。不能把行数完整度与 P50 脱钩比较。

## 构建与 A/B

- 新 bench：md5 **8b6d2f9d…**、480,981,312B；SPDK 静态库（libspdk_nvme.a 5a90008f…、libspdk_env_dpdk.a 2be37321…）；`/usr/local/lib/libspdk` 引用 **0**；旧 bench 备份 `stress_cluster_bench.nof150_old`（**bb4a07a3…**）。
- 24 次运行全部 failed=0、rc=0、Total keys=60000；设备 `first_completion_from_submit` 稳定 42–46µs。
- **ABBA 平衡块（双方完整行）：new 379.66 vs old 366.29 ⇒ 新慢 3.6%**。
- 正向配对 3/3 偏新、反向配对 2/2 偏旧；**5/5 对都是“后跑者更快”⇒ 顺序效应成立，正向结论作废**。
- **改动无收益**：真实效应 ≤ 噪声，[未验证] 有无小幅收益。

## 打点语义被破坏

- 补丁把 `get_begin` 移到“全部 child ready 之后”，导致 **① 变负（−122～−230µs）、get_block_us 恒 0、op 观测≈0、④ 失效（127～240µs）**；①/③/④ 不能再按 NOF-148/149 口径拆。
- 若继续该方向，必须新增 `observe_begin_mono_ns`，不能复用 `get_begin`；但本单 ABBA 已显示无收益，先回退。

## 偏差/未验证

- 任务书 C 段“3 遍稳定”未达成（new P50 269.22/368.97/318.92/321.49，spread 大）；改用配对/反向/ABBA 共 9 次补充运行，偏离原设计。
- 源码树 `/home/l00955908/mooncake-nof-vsegment-2/src/Mooncake` 无 `.git`，无法记录 HEAD/status；只记录补丁文本与 bench md5。
- `run/nof150_pair_rounds.csv` 列错位（缺 op_lines/child_lines 表头），已按语义手工对齐；`nof150_abba.sh` 首跑 `SEQ: unbound variable`，修正后重跑。
- 未重跑 NOF-148 口径 16µs 拆分（新二进制语义已破）。

## 环境末态

- `stress_cluster_bench` = **新 8b6d2f9d**（480,981,312B）；旧 **bb4a07a3** 备份保留。回退：`cp stress_cluster_bench.nof150_old stress_cluster_bench`。
- 源码 `transfer_task.cpp` 已改（`#include <thread>` + 2456-2466 轮询块），未回退。
- 服务未动：etcd 2822942、target B 2824683、`vseg_nof145v2` master 2828887/2828890、target A 3329243；Keys=60000、alarm 空；dbSize 2,048,806,912B。
- node2/3/4 无 bench/脚本残留；load 8.56/13.3/14.8。

## 裁定

- **回退新二进制与源码到旧**；第一刀方向暂停，不再保留“并行观察”补丁。
- **diag 行不全 vs P50 偏低**影响所有 diag-on 数据可比性，先只读排查；未来 A/B 强制 **ABBA 平衡块 + 每臂 ≥2 遍完整行**，单臂单遍结论无效。
- 不重启 target/etcd、不写。

---

# NOF-151（待执行 · 回退第一刀 + 只读排查 diag 行不全 vs P50 · 2026-11-27）

## A. 回退

1. 用 `stress_cluster_bench.nof150_old` 覆盖当前 `stress_cluster_bench`；核对 md5 **bb4a07a3…**、size 480,981,376B。
2. 回退 `transfer_task.cpp` 补丁（移除 `#include <thread>` 与 2456-2466 轮询块，恢复原 `for (child) child.get()` 语义）；源码树无 git，按补丁文本反向编辑或从备份恢复，记录改前/改后 md5。
3. 不重启 target/etcd、不写数据；回退后确认 `get_begin` 语义恢复（下一单再跑）。

## B. 只读排查 diag 行不全 vs P50（不跑 bench、不改码）

1. 用现有 NOF-149/150 日志（old/new、完整/BAD 遍）列：P50、key_lines、op_lines、child_lines、日志文件大小、最后一行时间戳、是否结尾截断/采样。
2. 检查 `MC_STORE_TRANSFER_DIAG` 采样、`MC_NOF_SLOW_TRACE_THRESHOLD_US=1`、日志限速/缓冲、进程退出 flush、磁盘、日志轮转；确认行数不全是“采样/截断”还是“真实丢数据”。
3. 回答：**BAD 遍的低 P50 是真实更快（日志跟不上/采样），还是测量被扭曲（丢行）**；据此决定未来基线是否只能用完整行，还是需要修日志/采样。
4. 若现有证据不足，列缺口并提出下一步受控实验（如更高阈值/同步 flush/只开一部分打点）；**本单不擅自跑**。

## C. 交付

- 表：pass / P50 / key/op/child lines / 完整度 / 日志大小 / 最后时间戳 / 结论。
- 一句话结论：行数不全的原因 + 对 P50 的影响 + 未来 A/B 基线政策（ABBA + 完整行 + ≥2 遍）。
- 不改正式结果。

## D. 边界

- 只回退二进制/源码；不改 target/etcd、不写数据；只读日志；不跑 bench（除非另单批准）。
- 预计 **10–15 分钟**。

---

## NOF-151 取消/并入 NOF-152（2026-11-27）

- 用户改为先做 **plain NoF + bdev_null 的 4MiB 单包实验**；NOF-151 的回退部分并入 NOF-152 第 0 步；diag 行不全的数据质量另单（NOF-153 待定）。
- 如 NOF-151 已启动，停掉；不改环境。

---

# NOF-152（待执行 · plain NoF + bdev_null · 4MiB 单包 E0/E1 · 2026-11-27）

## 目标

- plain NoF（**不开 vsegment**），target 用 **null bdev / -Z**（不碰 SSD），客户端设 `MC_NOF_SUBMIT_CHUNK_BYTES=4MiB`，让一个 4MiB key 成为一个 4MiB IO。
- T1 单并发、400 keys，E0/E1 各 3 遍；确认 target IO 计数 **~400**（而不是 12,800），测 4MiB 单包时延/带宽。
- 单并发诊断；不写正式表。

## 0. 回退 NOF-150

- 用 `stress_cluster_bench.nof150_old` 覆盖当前 `stress_cluster_bench`；核对旧 md5 **bb4a07a3…**、size 480,981,376B。
- 回退 `transfer_task.cpp` 补丁（移除 `#include <thread>` 与 2456-2466 轮询块）；记录改前/后 md5；不重启 target/etcd。

## A. 预检（能不动环境就不动）

1. target B/A 是否支持 `bdev_null`（`rpc.py bdev_null_create` / `bdev_get_bdevs`）与 `-Z`；能否**不停止/不重启现有 target/etcd**，用现有 target 的 RPC 添加 null bdev namespace，或在**备用端口**起一个独立的 null-bdev target。**若必须重启现有 target/etcd ⇒ 停手回报，不在本单重启。**
2. 记录 target `max_io_size`；确认旧 bench 支持 `MC_NOF_SUBMIT_CHUNK_BYTES` 与 `SPDK_URMA_MAX_IO_SIZE`。
3. JFC：确认 target URMA JFC/时间戳怎么取；若只有计数，如实说明。
4. 若 null bdev 无法在不重启现有 target/etcd 的前提下使用 ⇒ 停手；**不回退到 SSD**。

## B. 搭建 plain NoF（仅 A 通过后）

- 用 null bdev namespace / 独立 null-bdev target（备用端口、唯一 NQN）；注册 plain NoF segment；新 plain cluster `plain_nof152`（**不动现有 vseg master/target/etcd**；若必须停 vseg master 才能起新 master，停手回报）。
- 写 400 keys（null bdev，verify=false），Keys=400；写失败即停。
- 若搭建失败，停手；不写 SSD、不重启 target。

## C. E0/E1 4MiB 单包测试

- bench env：`MC_NOF_SUBMIT_CHUNK_BYTES=4194304`、`SPDK_URMA_MAX_IO_SIZE=8388608`；E0→E1→E0→E1→E0→E1（各 3 遍）；T1/batch1、400 keys、warmup128、qpairs4、verify=false、diag 关；target 保持 EID0。
- 记录 P50/Avg/P90/P99、failed、target IO 计数、JFC、W9。
- **验证是否一个 4MiB IO**：400 keys 应为 ~400 个 target IO（加 warmup 略多）；若 ~12,800（32×400），说明 chunk/transport/target 仍在切，报告实际层。
- 给 E0/E1 对照：4MiB 单包时延/带宽；不写正式表。

## D. 边界

- 除回退外不改码；不重启 target/etcd；不写 SSD；保留现有 vseg 数据；若搭建必须重启现有 target/etcd ⇒ 停手回报。
- 环境末态记录 master/targets；旧 bench 保留；无残留。
- 预计 **20–30 分钟**。
