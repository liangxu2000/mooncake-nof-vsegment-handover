# 第二阶段：消除 vseg 读路径的 +210µs

> **⚠️ 2026-11-24 16:35 状态更正（覆盖下方 2026-11-23 的「当前状态」与「一句话」）**
>
> 第二阶段原目标「消除 vseg 读路径 +210µs」所依据的归因链已整体撤回：同 cid 两 flags 差 **0.00% / 0.14%**，读路径无独立代价；所有 4.6–5.9× 归因（路径 / 会话 / qpair / 命令描述符 / 布局 / 年龄 / 设备侧）均未通过复核。
>
> 当前唯一被独立复现的触发变量是**访问流连续长度**（NOF-081 裸 SPDK 窗口矩阵 5.62×，跃迁 512–1200MiB）。**当前任务 NOF-082**：钉死流长曲线 + 两 cid 访问流逐元素对账，零改码 / 零重启 / 数据区零写入。对外口径暂不定稿。
>
> 详见 `../工作汇总.md` 顶部状态更正、`../HANDOVER.md` NOF-081 裁决块与 NOF-082 任务书。

---

- 日期：2026-11-23
- 阶段：第一阶段（NOF-001~046）已收口；本文档为第二阶段核心目标的开工依据
- 目录约定：**第二阶段所有内容（文档、代码补丁、测量数据）统一放在 `第二阶段/` 目录下**；本文件为总文档/索引
- 数据来源：`../HANDOVER.md` 各单回执 + 第一阶段最终报告 `../工作汇总.md`
- 相关文件：`../HANDOVER.md`（任务通道）、`../工作汇总.md`（第一阶段最终报告）
- **当前状态（2026-11-23）**：M3-0（NOF-060）提交层同构；M3-1（NOF-061）同数据 plain 快 4.64×；M3-2（NOF-062）**bdev p50 T1 249/54 → T4 262/259 → T16 270/265，差异 100% 在 `bdev_us`**；M3-3（NOF-063）静态无 50–500µs 常量，JFC/stride 结构错配经复核**不进 bdev_us 窗口**（`exec_tick` 在 `handle_completion` 内、跨线程投递在其前）；**M3-4（NOF-064）改码/重编/重启三步完成（target 1587271 / `21ced674…`），但两处受阻：① 重启使 raid0/nsid1 uuid 变化 ⇒ vseg 元数据 mount rc[-300]（plain 不受影响，产品级缺陷）；② v1 插桩插在写路径 2214 而非读路径 1921，`d_*` 为绝对值**。**v1 的 plain 225µs 判为 per-IO printf 观测者效应（无插桩口径 54µs），不用于任何结论**。**NOF-065 已下发（M3-5，沿用改码/重编/重启授权）**：v3 零干扰插桩（printf→环形数组 + 独立线程落盘、补 iov 字段、`d_*` 改真 delta）→ 重启后新 cid `vseg_nof065` 写一次 400 keys → 两臂 T1 各 2 遍 + 各 1 遍 `TRACE=0` 对照（差 >5% 整轮作废）→ 按 `bdev_us` 分桶判 H1/H2/H3、`iov[0].base` 分 NUMA 桶。**重启不可回退 e12b7cb**。
- 一句话：**数据/盘无罪、target 看到的命令与顺序也与 plain 完全相同，但 vseg 每条 128KiB 读在 target 的 bdev 执行段多花 ~210µs。第二阶段的目标就是把这 210µs 找出来并消掉。**

---

## 0. 2026-11-23 复盘与纠偏（写于 M0 完成、M1 未开始时）

方向判断：**"定位到 vseg 路径 → 再修"这个大方向是对的**，但执行顺序需要纠正——在投入代码插桩前，有三个更便宜、且有源码证据的假设没有排除。

### 0.1 发现一：我们一直在用"旧构建"，远端 tip 已包含读路径/时延修复

- 同事构建基线 **be10be2（11-17）**；远端分支 tip **4c0ea28（11-19）**，两者差 **75 文件 / +6057 / −473 行**。
- tip 中与读时延直接相关的提交：
  - `ff3e853 fix(store): restore low-latency UB read defaults` —— 把 NoF per-transfer breakdown 日志改为**显式 opt-in**（`MC_STORE_TRANSFER_DIAG`），并把 UB poll sleep 默认从 50µs 改为 **0（busy poll）**、增加 idle spin。**commit message 原文：该 breakdown 日志"directly inflates caller-observed read latency"**。
  - `13b7c19 vsegment: skip duplicate view validation`（去掉每次读的重复校验扫描）
  - `1712b68 vsegment: guard resolve loop...`
  - `c0a3252 feat(bench): add key_order selection`（我们缺的随机读旗标）
  - `9382888 fix(urma): 发送请求启用驱动扩展标志`
  - `f3d39a2 / 343ba6f / e15da9d`：vsegment 写路径的持久化/分配器快照性能修复
- **be10be2 不含以上任何一条**。我们在旧构建上研究 210µs，可能是在研究一个**已经在新代码里被修掉或缓解**的问题。

### 0.2 发现二：诊断默认全开，且我们从未测过"诊断完全关闭"的 vseg

- 旧构建 `MC_HIFREQ_LOG_SAMPLE_RATE` **默认 1.0（全采样）**；`ShouldSampleHiFreqLog` 控制的 per-transfer / per-task breakdown 与计时**默认对每条读都开启**。
- vseg 路径额外有 `VSegmentOperationState`、逐 child 计时、`nof_vsegment_operation` per-key 日志；其 task 数还是 plain 的 2 倍（2 child/key）。
- 我们的 NOF-035+ 还显式设了 `MC_HIFREQ_LOG_SAMPLE_RATE=1`、`VSEG_SLOW_IO_US=1`（逐 IO 29 字段日志）。
- **per-task 日志在 NoF 轮询线程的任务完成路径上输出**；轮询线程停顿会推迟 target 完成回调，**有可能把 target 的 bdev/W9 抬高**（客户端背压）。
- 结论：**"210µs 有多少是默认诊断造成的"从未被单独测过**。这条比代码插桩便宜得多。

### 0.3 发现三：宿主位置是未控变量

- 裸 SPDK 快基线（41µs）跑在 **node1（远端）**；所有 Mooncake plain/vseg 对比跑在 **node3（本机）**。
- 本地/远端数据路径不同；plain 本机 bdev 46µs 甚至低于裸 SPDK 远端 W9 91µs，说明口径不能直接混用。
- 需要补：node3 本机裸 SPDK 定点读（同区）和/或 node1 远端 Mooncake-vseg 读，才能把"vseg 路径慢"与"宿主/数据路径"彻底分开。

### 0.4 目标再校准

- 即使 210µs 全部消除，单 target 下 vseg 的上限是**追平 plain**；"vseg 必须更快"需要多 target/多成员或分层缓存（结构性方向）。
- 第二阶段验收应定为：**机制定案 + 消除回归（追平）**；"反超"单列为后续结构性问题，不纳入本阶段验收。

### 0.5 修正后的执行顺序（替代原 M1 直接插桩）

| 步骤 | 内容 | 状态/成本 | 结果与判据 |
|---|---|---|---|
| **S0** | 同环境、同 key、T1、verify=false，诊断开 vs 诊断全关，比较 target W9 | **已完成 2026-11-23**（NOF-049） | ON **258.85** / OFF **251.75µs（−2.7%）**，exec 96.2–96.3% 不变 ⇒ **诊断/客户端背压排除** |
| **S1** | node3 本机裸 urma_perf 定点读同一区 | 可选参照，~10 min | vseg/plain 对比都在本机，宿主混淆不改变结论；暂缓 |
| **S2** | 构建 tip 4c0ea28 重测 | **暂缓** | diff 复核：be10be2→4c0ea28 的 `transfer_task.cpp` 仅 40 行诊断/UB 默认改动，**NoF 读提交/QoS/分块未变** ⇒ 对 210µs 预期收益低 |
| **M1a** | 客户端 NoF 参数插桩（`patches/nof050_client_io_params.patch`）+ vseg 采集 | 构建成功，**被版本阻塞**（NOF-050） | 新 bench（SPDK 9a413a44）与 target（e12b7cb）握手失败；同事原 bench 正常 |
| **M1a-1** | 构建 e12b7cb 客户端 SPDK，重链插桩 bench，node3 本机重试 | **已完成 2026-11-23（NOF-051）** | e12b7cb 不可编（UMDK 缺 is_gpu_seg）；改用 **bd5ee124d（wire v4）** 成功；vseg 采到 1056 task，**domain 非空**，W9 258.55 |
| **M1b** | 切换 plain，同客户端同插桩 bench，node3 本机采集对比 | **已完成 2026-11-23（NOF-052）** | domain 与 vseg 完全相同；plain W9 **132.77**（旧客户端 133.29，版本无关）；**公平差距 vseg 258.55 vs plain 132.77 = 1.95×/+125.8µs**；P50−W9 311/147µs ⇒ ≈147µs/task 客户端开销 |
| **M1c** | 零环境解析客户端 breakdown 日志（NOF-053） | **已完成 2026-11-23** | 客户端 QoS/worker/dispatch 仅 3%；master 元数据 RPC 116µs/key（41%）两臂共有；vseg 2× 在 drain 350 vs 110µs ⇒ **客户端调度排除，转 target/传输** |
| **M2-target** | 换 target 进程 + target 侧逐 IO 打点 | **A 已选；M2-0 完成** | **无需补丁**：bd5ee124d nvmf_tgt 自带 `URMA_STAGE_BUFFER`（24 字段：lba/nlb/hz/exec/enter/submit/submit_return/callback/wait/enomem/rx/buffer/bdev_cpl/data_post/data_cpl/rel0/lcore/flags），进程退出落盘 |
| **M2-1** | 重启 target 到 bd5ee124d + BUFFER | **已完成 2026-11-23（NOF-055）** | target **394826**（bd5ee124d/md5 979f5517…），4457/RAID0(11 盘/BDF 升序/256K/online)/ns 正常；BUFFER 生效（`run/urma_stage_nof055.bin` 0B、O_EXCL、退出落盘）；旧日志备份 32,780,200B/d294d6ef…；**盘序 BDF 升序、旧序不可恢复 ⇒ 新旧数据不可按 LBA 对比** |
| **M2-2** | vseg/plain 两阶段采集 + target 退出落盘 + 解析 | **已完成 2026-11-23（NOF-056 停手 → NOF-057）** | 补齐 400 keys；**126µs 100% 在 `submit_return→callback`（SSD 执行段）**；其余 6 段 0（data 段 vseg 更短）；**NUMA 排除**（256/255 vs 59/61）；命令相同（lba 集合/nlb/flags）；盘级访问流：换盘率 **79.7% vs 50.0%**、每盘递增 98.0% vs 99.9%、回退 31 vs 1；与 NOF-034/046 有张力 |
| **M2-3** | 离线判别"快/慢 IO 的局部条件"（零环境） | **已完成 2026-11-23（NOF-058）** | **并发排除**（ov=0：vseg 206 vs plain 39，ov 分布同为 34.4%）；**局部 LBA 顺序排除**（紧接/busy=False 仍 5×）；plain 慢 IO 全部是盘首 keys 0–97，快 IO 全在 keys≥98；same-LBA 5.07× 但受写轮次混杂 ⇒ 剩"数据状态 vs 读路径" |
| **M2-4** | 单轮写入下：stripe=4MiB 单 child 验证 + 同数据裸 SPDK 判别 | **已完成 2026-11-23（NOF-059）** | **单 child 属实**（读 528 SUBMIT/8192/child=0/1）但 **未追平**（P50 547.92 vs 268.44，2.04×）⇒ child 数无关；**同一数据裸读顺序 40.0 vs 40.8µs、随机 220.16 vs 220.16** ⇒ **数据物理状态无罪**；差异锁定在 vseg 客户端驱动 4MiB NoF 操作的方式 |
| **M3-0** | 客户端 NoF 提交层取证（零环境） | **已完成 2026-11-23（NOF-060）** | **两臂完全同构**：params 除 child 外全同（op/blocks=8192/bytes=4MiB/block_size=512/endpoint/lane/domain/ptr 对齐）；源码 `transfer_task.cpp:3076` 同一函数同参数、128KiB chunk、8MiB/lane、task 内 32 IO 同 qpair；child 仅日志标签。URMA_SLOW_IO 有绝对 tick；唯一实测差异=突发 16 vs 32（2MiB-child 时代），vseg4 未获取 |
| **M3-1** | 同一数据两读路径 + 逐 128KiB 时钟（零构建、target 不重启） | **已完成 2026-11-23（NOF-061）** | 写 LBA 集合与 plain 逐值相同；零写入用 plain 元数据读同一批 LBA：**plain P50 308.37 / bdev 57µs** vs **vseg P50 573.03 / bdev 249µs**；同 LBA 配对 12,324 条 **246 vs 53（4.64×）**，plain 更快 85.1% ⇒ **差异在 vseg 读路径/会话**。vseg4 突发 31（4MiB）与 plain 相同；新增排除：突发形态、target lcore（同核仍 4.3×）、顺序/预热 |
| **M3-3** | URMA transport 轮询/阻塞路径源码取证（零环境） | **已完成 2026-11-23（NOF-063）** | 主链 `transport.c:777→779`→`urma.c:2303`（81 行）；**无 50–500µs 常量**（`urma_poll_jfc` 无 timeout、无 sleep/spin）；结构：`stride=worker_count=6` vs `send_jfc_count=2` ⇒ 仅 worker 0/1 轮询网络，qpair JFC=建连序 mod 2 与属主无对齐；**复核：两臂对称 + 同臂 4 qid 同值 ⇒ 不解释 4.6×**；本地 NVMe 每轮不限量、无 defer ⇒ 静态无法定位，转插桩 |
| **M3-4** | target 逐 IO 插桩（子命令字段 + 轮询节奏） | **部分完成 2026-11-23（NOF-064）** | 改码/重编/重启全过（PID 1587271 / md5 `21ced674…`，`NOF064_TRACE=1`；三份二进制保留）；`worker_index 0→core3,1→core4,2→core5,3→core6,4→core7,5→core2`。**阻塞①重启使 nsid1 uuid 变化 ⇒ `vseg_nof061` mount rc[-300]（plain 元数据正常）**；**阻塞②v1 插在写路径 2214、读走 1921 ⇒ `d_*` 为绝对值**。plain v1 bdev p50 225µs **判为采样偏差（4,000 条≈前 125 key，78% 落在慢窗口；客户端 P50 未变）⇒ 不可用** |
| **M3-5** | v3 零干扰插桩 + 新 cid 写入 + 两臂对照 | **NOF-065 已下发** | 环形数组（≥400k 条）+ 非 reactor 线程 3s 落盘；补 `iov_cnt/iov[0]/total_iov_bytes`；`d_*` 改真 delta；重启后新 cid `vseg_nof065` 写一次；两臂各 2 遍 + 各 1 遍 `TRACE=0`（差 >5% 作废）；`bdev_us` 分桶 + `iov[0].base` 分 NUMA 桶 |
| **M1c** | target 侧插桩（SGL/bdev 提交参数/分段 tick） | 需重编 target + 重启（用户批准） | 定位 210µs 在 raid0 队列/NVMe/回调哪一段 |

- NOF-048（最小重建验证）已完成，配方已用于 M1a；构建基线是否升级 tip 视后续需要（S2 暂缓）。

### 0.6 NOF-050 阻塞（2026-11-23）：客户端/target SPDK 版本不兼容

- **现象**：插桩 bench（客户端 SPDK **9a413a44**）连 target 失败，客户端 `Ctrlr operation failed -104` → `Connection reset by peer` → `rc=-19`，target 侧 `accept: handshake failed rc=-71`；同事原 bench（md5 59662a26，客户端匹配 **e12b7cb**）同 env/参数 rc=0。
- **版本事实**：target 日志首行 `SPDK v26.09-pre git sha1 e12b7cb21`；target 进程是 **deleted 旧二进制 736a902f**，磁盘 `nvmf_tgt`=d6c39465（=9a413a44）；两者之间 **10 个 commit**（含 `exchange full bonding contexts`、`patch exported rjetty copy`、`correct wire version`）。
- **口径修正**：NOF-050 的对照 P50 473.52 / W9 259.77 是 **node1 远端 reader**；NOF-040 的 274µs 是 **node3 本机 reader**，两者不可直接比。`MC_STORE_PARTITION_SLOT_COUNT=1` 未把 P50 拉回 274 ⇒ 之前的"SLOT 解释"不成立，宿主位置更可能是原因。后续 plain/vseg 对比必须在**同一台机器**上跑。
- **三条路线**：
  - **A（已选，NOF-051）**：在我们副本里构建 e12b7cb 客户端 SPDK（与 target 同源、wire 兼容），重链插桩 bench，node3 本机重试。不动 target、保留第一阶段基线；风险：旧 target 可能带本地改动（干净 e12b7cb 仍可能不匹配）、SPDK 全量构建耗时。
  - **B（暂不）**：试 node1 `/usr/local`（6fae46095，第三个版本），成功概率低、relink 也麻烦。
  - **C（备选，需用户批准）**：重启 target 到磁盘 d6c39465（9a413a44），一步对齐、现有 bench 可用；但 **旧 e12b7cb 二进制已删除、不可回退**，会改变第一阶段基线，需要重新 register/mount 并重测。
- **环境提醒**：target 当前跑 deleted 旧二进制，任何意外重启都会切到 9a413a44（e12b7cb 不可恢复）；node1 在 15:02 有第三方 root `cmake --build ... -j308`，做时延测量时需记录/避开。

---

### 0.7 NOF-051 结果（M1a-1 打通）与 M1b 待答（2026-11-23）

- **wire 谱系（决定握手）**：`e12b7cb` / `bd5ee124d` / `170cbb9` = **WIRE v4**；`91763b015` / `9a413a44` = **WIRE v6**。运行中 target 是 v4；我们的第一个 bench 用了 v6 ⇒ 握手失败。NOF-050 根因坐实。
- **构建**：`e12b7cb` 本身**不可编译**（系统 UMDK `umdk-urma-devel-26.06.0-B020` 缺 `is_gpu_seg`/`urma_register_seg_dmabuf`，属新 API）；改用其**紧邻后继 `bd5ee124d`**（"make GPU registration mechanisms opt-in"，wire 仍 v4）→ `./configure --with-urma` + `gmake -j64` 成功（约 5 min、74 个 `.a`）。
- **握手**：重链后的 `bench-io-params-e12`（md5 `88ad3134…`）在 **node3 本机**成功连接 target，target 日志零新增 handshake 失败。
- **vseg 参数（400 keys + 128 warmup，T1）**：SUBMIT/WORKER **1056/1056**（= 528 keys × 2 child）；`ptr` 注册域命中 **100%**（仅 2 个 ptr，child1 = +2 MiB）；child `seg_lba` 差恒 **4096 blocks = 2 MiB**；lane/worker 各 264，4 路均匀；`blocks` 全 4096（2 MiB）；`buf_type=kHost`、`buf_device=-1`、**`domain` 全为非空 `0x143cd90`**；`vseg_op_id` 全 0（该 build 未递增）。
- **target W9**：258.55µs（n=16,902，exec 97.2%）；端到端 P50 569.78µs（node3 本机）⇒ 客户端侧开销约 311µs（**远高于历史 NOF-040 的 ~19µs**；与 bd5ee124d 客户端或同机竞争有关，未验证）。
- **关键待答（NOF-052）**：
  1. plain 用**同一个 bd5ee124d 客户端**时，WORKER 的 `domain` 是否与 vseg 相同（都非空）或为空/不同？
  2. plain 的 target W9 是否仍 ~46–70µs？若也 ~250 ⇒ **新客户端版本是混淆变量**（历史 plain 46 来自旧客户端），210µs 的归因需重做；若 ~46–70 ⇒ vseg 差异真实，继续沿 domain/child 查 target 侧。

### 0.8 NOF-052 公平对照结果（2026-11-23，**修正旧口径**）

- **domain 排除**：plain 与 vseg 的 WORKER 参数完全相同——`buf_type=kHost`、`buf_device=-1`、`domain=0x143cd90`（非空）、ptr 注册域命中 100%、4 lane 均匀。memory domain 不是差异来源。
- **客户端版本排除**：同一 plain 数据/环境/node3，新客户端（bd5ee124d）W9 **132.77µs**，旧客户端（59662a26，e12b7cb 时代）**133.29µs**（+0.4%，target n 完全一致）⇒ 客户端 SPDK 版本对 W9 无影响。
- **公平差距 = 1.95×/+125.8µs**：vseg 258.55 vs plain 132.77。**历史 plain 46–68µs 是 slot=1024 口径**，而 vseg 一直是 slot=1；旧的 3.75×/+210µs 对照不可直接比。slot=1 下 plain 的 46–68 无法复现（133µs），原因未验证（slot/写入新鲜度/盘状态）。
- **新发现：≈147µs/task 的客户端开销**：`P50 − W9` 在 vseg 为 **311µs**、plain 为 **147µs**，比值 2.11× ≈ task 数比 2.00× ⇒ 每个 transfer task 约 147µs 的客户端成本。它放大了端到端差异（P50 570 vs 280），但对"vseg 必须比 plain 快"而言，**target W9 的 1.95× 才是数据面核心**。
- **下一步（NOF-053）**：零环境解析现有 breakdown 日志（setup/submit/wait/QoS/worker），分解这 147µs/task 的构成；再决定客户端优化 vs target 侧插桩（需重启批准）。

### 0.9 NOF-053 客户端分解（2026-11-23，零环境）

- **日志口径**：任务书点名的 `batch_get_data_breakdown`/`batch_get_nof_wait_breakdown` 只属于 BatchGet；本 bench 走单 key Get，实际字段在 `get_into_breakdown`、`query_rpc_breakdown`、`transfer_data_breakdown` 与 `nof_slow_task`（90+ 字段，覆盖 QoS/worker 全部指标）。
- **plain 每 key 283µs 的构成**（keys 97+，历史日志）：

| 段 | µs | 占比 |
|---|---|---|
| master 元数据 query（GetReplicaList） | **116** | **41%** |
| transfer submit | 6 | 2% |
| transfer wait（含 target W9 68，超出 88） | 156 | 55% |
| 其他/残差 | ~5 | 2% |
| QoS/worker/dispatch（nof_slow_task） | 8–11 | ~3% |

- **vseg 的 2× 不是 setup/QoS**：query 103 vs 116（反而更低）；submit 19 vs 6；wait 439 vs 156；每 task total 391 vs 145。直接原因是 **drain 350 vs 110µs（3.2×）**、**max_subio 362 vs 112（3.2×）**；`qos_tasks_ahead` 全 0、fifo/credit 全 0、dispatch 11/8µs、worker_setup ≤1µs ⇒ **客户端调度排除**。
- **口径提醒**：`wait_us` 包含 W9，`wait_us − W9` 只是客户端/传输开销的上界（随 W9 升高而缩小）；更干净的指标是 `max_subio_us − W9` = **44µs plain / 107µs vseg**（单子 IO 的客户端+传输往返）。
- **结论**：
  1. **客户端 QoS/调度不是杠杆**（~3%）；
  2. **master 元数据 RPC ≈110µs/key 是两臂共有的固定大头**——它是 plain 绝对时延（P50 280µs）的最大单项，但与 vseg/plain 的相对差异无关；
  3. **vseg 的 2× 全部落在子 IO 的 drain/目标执行段** ⇒ 下一步必须查 target/传输，而不是客户端。
- **下一步（需用户批准 target 重启）**：当前 target 是 deleted e12b7cb，旧二进制不可恢复。二选一：
  - **A**：自编**带打点的 bd5ee124d（v4）** target，保持现有 v4 客户端/参数（行为最接近当前 target；需 target 插桩+构建+重启）；
  - **B**：直接用磁盘上的 **9a413a44（v6）** target + 已有 **v6 打点 bench**（`bench-io-params`）；无需 target 构建，但版本跳两代（wire v6、nvmf/urma.c +239 行），行为可能变化、需重测基线；重启前可先只读确认新版逐 IO 字段是否更丰富。
  - 两者都需重放 RPC 配置、重新 register/mount、重测 vseg/plain 基线，且**不可回退到 e12b7cb**。

### 0.10 NUMA 审计（用户 2026-11-23 提问；**放到 NOF-056/后续任务，不改已开跑的 NOF-055**）

- **已确定的固定项**：URMA 设备一直是 **udmac0d1e2**（UB controller0；target 与 client 都是；NOF-005 误用 udmac1d1e2 失败后已改回）；target reactors 固定 `-m [2-7]`；客户端 bench 的 `NR_SOCKETS=numa_num_configured_nodes()`、`bindToSocket(tid%NR_SOCKETS)`、`numa_alloc_onnode(..., t%NR_SOCKETS)` ⇒ **T1 的读线程绑 socket 0、buffer 在 node 0**；16t 时线程/buffer 轮转到 4 个节点。
- **未审计/未控制**：①11 块 NVMe 的 BDF `numa_node`；②URMA 设备 `udmac0d1e2` 的 numa_node；③target 的 4640 个 hugepages（旧记录 4×1160）及 8GiB iobuf 实际落在哪些节点；④每条 IO 的 iobuf/数据 buffer 节点（2MiB vs 4MiB task 可能选中不同池/节点）；⑤客户端 buffer 与设备/盘的节点距离。
- **为什么重要**：target 的 iobuf 需求（~8GiB）大于单节点 hugepages（~2.3GiB），**必然跨节点**；若 vseg（2MiB task）与 plain（4MiB task）的 iobuf 分配落在不同 NUMA 节点，或盘/设备跨节点，可能贡献 target W9 的 1.95×。这是**尚未排除的混淆变量**。
- **审计清单（加入 NOF-056 的预检，只读）**：`lscpu`/`numactl -H`；11 BDF + 系统盘 2b 的 `numa_node`；`/sys/class/uburma/*/device/numa_node`；`/sys/devices/system/cpu/cpu{2..7}/node*`；每节点 hugepages total/free；reader 的 `Allocated ... NUMA-aware, N sockets` 行。
- **2026-11-23 实测（NOF-056 A2）**：
  - 4 节点：node0=cpu0–69、node1=70–153、node2=154–237、node3=238–307；距离 0↔1=15、0↔2=**20**、2↔3=15。
  - **11 块数据盘：5 块在 node0**（21/22/28/29/2a）、**6 块在 node2**（91/92/a1/a2/a3/a4）；系统盘 2b 在 node0。
  - **target 核 2–7 全部在 node0** ⇒ **6/11 盘与 target 跨 NUMA**；RAID0 每 256KiB 条带在 node0/node2 盘间交替；vseg 2MiB task=8 条带、plain 4MiB task=16 条带，跨节点比例/顺序不同。
  - hugepages 各节点 1160 页，free **16/75/128/128**（总 4640/free 347）；uburma 设备的 numa_node 读不到 `[未获取]`。
- 若解析显示 vseg/plain 的盘节点比例/顺序或 per-node 时延不同，再设计受控 NUMA 实验（pin 核/内存/设备）；在此之前不把 1.95× 归因于 NUMA。

### 0.11 NOF-056 在 B2 停手（VSEGMENT_CREATING；续跑 NOF-057，2026-11-23）

- **停手点**：vseg 从零建集群后 T1 写 400 keys，触发 **23 次 `VSEGMENT_CREATING`**（每 64MiB vsegment 容 16 个 4MiB key，第 17 个 key 撞创建窗口且客户端不重试）；key 16,33,…,390（步长 17）；只写入 **377/400**、logical_used 1,581,252,608、vsegments=24；按规则未跑 reader、未切 plain、未停 target（buffer 0B）。
- **定性**：这是**已知机制**（NOF-028/029/032/034 已 4 次复现），不是随机故障；之前"≤2"是因为读的是已提前长好的 800-key 集群。
- **续跑选择（选项 1）**：同参数 T1 writer 重复补写（≤3 次）至 400 keys；然后完成 vseg 读（trace）→ plain 写+读（trace）→ 优雅停 target 落盘 → 解析。补齐的 23 个 key 若非规范 LBA（在尾部），两阶段都排除这 23 个 key 窗口后比较。
- **环境**：target 394826（bd5ee124d/BUFFER）、vseg master 2562417（377 keys）、buffer 0B；原地续跑，不重置集群。

### 0.12 NOF-057 定案（2026-11-23）：126µs 全在 SSD 执行段，NUMA 排除，剩"盘级访问流"

**采集**：vseg（slot=1、stripe=2MiB）与 plain 两阶段 T1 读（400 keys、warmup 128），target TERM 优雅退出落盘 **33,792 条、dropped=0**（`run/urma_stage_nof055.bin`，6,488,160 B = 72 + 33,792×192 + 24）。按 exec_tick 最大间隙 208.953 s 分段：vseg 16,896 / plain 16,896。

| 项 | vseg | plain | 差 |
|---|---|---|---|
| 端到端 P50 / Avg | 551.22 / 559.93 | 277.50 / 347.45 | +273.7 / +212.5 |
| W9（exec→bdev_cpl）p50 / mean | **256 / 261.1** | **61 / 136.5** | +195 / +124.6 |
| `submit_return→callback` p50/p90/mean | **256 / 314 / 260.4** | **60 / 284 / 135.9** | **+196 / +124.5** |
| 其余 6 段 | 0 | 0 | 0（data 段 vseg 6/10 vs plain 12/33，更快） |
| 服务时间形状 | 128–256µs 49.9% + 256–512µs 49.7%（64–128 仅 12 条） | 32–64µs 49.9%，≥128µs 36.6% | — |

**命令与 NUMA**：两段 `nlb` 全 256、唯一 lba 12,800、**lba 集合完全相同**、flags 全 3；lcore vseg 2/5/6/7 vs plain 2/3/4/7（各 4,224）。按 `disk=(lba/512)%11`（0–4=node0、5–10=node2）：vseg node0 p50 **256** / node2 **255**；plain **59/61** ⇒ **跨 NUMA 不是原因，彻底排除**。

**唯一剩余差异（相关性，未证因果）**：盘级访问流——换盘率 **79.7%（13,462/16,895）vs 50.0%（8,447/16,895）**、每盘 LBA 递增 **98.0% vs 99.9%**、每盘回退 **31 vs 1**、每盘 IO 数 1,534–1,538（均匀）。与 NOF-045 的盘特性（同区顺序 41µs vs 随机 220µs）吻合：vseg 被打碎的盘内访问流落在"随机档"。

**为什么还不能收口（张力）**：
1. **NOF-034**：stripe=4MiB、每 key 单 4MiB child（与 plain 同构）时 W9 仍 **250.1–270.9**——若"child 数/两段流"是根因，单 child 应已恢复；
2. **NOF-046**：q1 单 lane 的 rx/exec/submit 顺序与 plain 完全一致（0 逆序）仍 223–265µs——但该结论基于旧 target 的粗粒度 tick（同 tick 批量可能掩盖组内乱序），需以 NOF-057 的逐 IO 数据为准；
3. 因此"表面顺序"或"换盘率"可能只是**代理变量**，真正的局部条件可能是"同盘/同队列的在途重叠（`ov`）"或"前一条命令是否紧接（gap/busy）"。

**NOF-058（已下发，零环境）**：用同一份 buffer 做判别——(a) 每条 IO 执行时同盘在服务 IO 数 `ov` 与服务时间的关系、`ov=0` 子集是否仍慢；(b) 同 `(disk,lcore)` 前驱的 `gap`/`busy` 与服务时间的交叉表；(c) plain 段内快/慢 IO 的自然实验（命令/盘相同，判别变量必在 24 字段内）；(d) params 日志的每 lane 结构与 child 交错。输出"并发 vs 局部性"的判决与修复杠杆。

**产物**：`run/urma_stage_nof055.bin`、`run/nvmf_tgt_bd5ee124d_20261123_163717.log`（193,272 B / md5 a8222276…）、`run/nof057_vseg_read.log`（10,843,883 B）、`run/nof057_plain_read.log`（10,570,443 B）、master/写日志。**末态：target down（4457 free）、两 master 已停、无 bench**；buffer 已完整落盘（后续再采须换新文件名，O_EXCL）。

### 0.13 NOF-058 离线判别（2026-11-23）：并发与局部顺序均排除，剩"数据状态 vs 读路径"

零环境（target 保持 down），只读 NOF-057 的 33,792 条 buffer 与两份 params 日志。

**顺序重建是稳的**：同 exec_tick 的 IO 占比 vseg 0.01%（2 条）/plain 0（最大同 tick 组 2/1）；`(exec,trace_seq)` 与 `(exec,rx_trace,trace_seq)` 两种 tie-break 结果完全一致；仅纯 trace_seq 序会把 vseg 换盘率从 79.7% 降到 59.7%（说明记录按完成序追加）。warmup→测量边界 index=4,096（LBA 990,976/1,048,320→0），两段对称（4,096 + 12,800）。

| 测试 | vseg | plain | 结论 |
|---|---|---|---|
| **ov=0**（该盘零并发）服务时间 | n=4,400（34.4%）p50 **206** / p90 250 / mean 202.0 | n=4,405（34.4%）p50 **39** / p90 213 / mean 79.2 | **并发/队列深度排除** |
| ov 分布（0/1/2/3） | 34.4/34.4/15.6/15.6% | 34.4/36.0/16.4/13.2% | 两臂几乎相同 |
| 同 (disk,lcore) 紧接前驱 gap≤512 | 50.2% p50 **275** | 50.2% p50 **61** | **局部 LBA 顺序排除** |
| busy=False（前驱已完成） | **206** | **39** | 同上 |
| 同 gap 桶 near<8MiB | 207 | 41 | 同上 |
| 全体快/慢 IO 的 LBA 分布 | 慢 ≥128µs 99.9%（400 keys 全覆盖） | 慢 24.5% 全部 keys 0–97（LBA≤802,304≈391MiB 盘首慢区）；快 <64µs 65.1% 全部 keys≥98 | plain 只有在盘首慢区才慢 |
| 同一 LBA（keys≥98，n=9,664） | p50 256 / mean 250.2 | p50 49 / mean 49.4 | **5.07×，但受写轮次混杂** |

**为什么还不能收口**：NOF-057 的同一 LBA 对比中，vseg 数据与 plain 数据由**不同 writer/不同写入历史**产生（vseg 377 keys 单写 + 23 keys 补写；plain 随后覆盖写入），所以 5.07× 不能单独归因于读路径；而 NOF-045 又曾显示 q1 写的同区数据裸 SPDK 顺序读只要 41µs（数据无罪）。**两个方向都还站得住**，必须用"单轮写入 + 不重写 + 两种读法"分离。

补充事实：客户端 per-lane 顺序率 vseg 93.9%（回退 15–17/lane）vs plain 99.2%（1/lane，文件行序 `[推断]`）；vseg 的 child0/child1 SUBMIT 严格交替 1,055/1,055；target 执行序比客户端提交序多 +20pp 换盘（79.7% vs 59.7%），target 内部有重排放大（但客户端序为文件行序推断）。

**NOF-059（已下发，含 target 重启）**：一次运行同时测两问——
1. `stripe=4MiB` 写/读 params **验证每 key 单 child**（528 SUBMIT、blocks=8192），看 W9 是否从 ~256 回落到 plain 档；
2. 同一份**单轮写入、不重写**的 vseg 数据，用裸 SPDK `urma_perf` 顺序（keys 98–399、key 200）与随机读；随后 plain 写+读+裸读作同批对照。判据：vseg4 回落 ⇒ 单 child 修复有效；未回落但裸读顺序快 ⇒ 数据无罪、读路径仍有差异；未回落且裸读顺序也慢 ⇒ **数据物理状态**（写侧）为主因。

### 0.14 NOF-059 结果（2026-11-23）：单 child 无效、数据无罪，差异锁定在读路径

target 重启 **PID 708167**（bd5ee124d / md5 979f5517…，**无 BUFFER**，保持运行）；`stripe=4MiB` 的 vseg master（slot=1）与 plain 各写 400 keys 后测 T1 读 + 同数据裸读。

| 项 | vseg4 | plain | 判读 |
|---|---|---|---|
| 端到端 P50 | **547.92** | **268.44** | **2.04×**（双 child 为 1.99×）⇒ **child 数无关** |
| 写/读 params | 写 377+23、读 **528/528**，`blocks=8192`, `child=0/1` | 写 400、读 528，`blocks=8192`, `child=0/0` | **单 child 确实生效** |
| 裸 SPDK 顺序（同数据，keys 98–399） | p50 **40.0** / W9 86.3 | p50 **40.8** / W9 79.6 | **数据物理状态无罪** |
| 裸 SPDK 随机（同区） | 220.16 / 191.1 | 220.16 / 190.9 | 随机档两臂相同 |
| 裸 SPDK 单 key 200 定点（4MiB 内顺序） | 218.88 / 187.9 | — | 单 key 内部顺序读也已是"随机档" |

- **结论**：child 数与数据状态都不是原因；差异只能在 **vseg 客户端把同一个 4MiB NoF 操作驱动到 target/盘的方式**（提交时序/顺序/SGL/buffer/endpoint/会话）。
- **口径警告**：本单 target W9 表（vseg 122.6 vs plain 30.4）受 dump 滞后、跨相位与 n 分块不齐影响，**不用于结论**；per-key 24 字段需 BUFFER，本单被禁止。
- **偏差**：首启 ABORT（11 盘 vfio 残留，`takeover -r -y` 还原后重启）；首次写非 root 失败（改 sudo）；23 次 VSEGMENT_CREATING（补 1 轮）；urma_perf preflight 覆盖 key98/key200 各 128KiB。
- **环境**：target 保持运行（708167、4457、无 BUFFER），两 master 已停，无 bench。

**NOF-060（M3-0，已下发，零环境）**：① 对比 vseg4/plain 的 params **全字段**（endpoint/ptr/block_size/bytes/child/vseg_op_id/lane）——NoF 提交层是否完全同构；② 检查 NOF-057 `URMA_SLOW_IO` 是否有绝对时间戳（提交节奏/突发；无则 `[未获取]`）；③ 只读客户端源码（`nvme_urma.c` + `transfer_task.cpp`/vsegment runtime）定位 4MiB→32×128KiB 切分与提交路径、两臂差异清单、M3-1 插桩点与最小重建路径。target 不动、不构建。

### 0.15 NOF-060 取证（2026-11-23）：两臂 NoF 提交层完全同构

| 项 | vseg4 | plain | 判读 |
|---|---|---|---|
| params 全字段 | op、blocks=8192、bytes=4MiB、block_size=512、endpoint uniq=1、vseg_op_id、lane 0–3 各 132、domain 0x143cd90、ptr mod 4096/131072=0 | 同（ptr 地址不同但同对齐） | **除 child 外全同** |
| child | 0/1 | 0/0 | 源码证实**只进日志/timing 标签，不进 SPDK 调用** |
| 源码汇合点 | `transfer_task.cpp:2360/2527` → **`:3076` 同一 `submitSpdkNofOperation`、同参数** | 同 | 提交层同构 |
| chunk/在飞/队列 | 128KiB（`blocks_per_chunk=256`）、8MiB/lane、lane 轮转 `%4`、**task 内 32 IO 固定同一 qpair**（`:891`） | 同 | 同构 |
| URMA_SLOW_IO | 有**绝对 tick**（initiator submit/complete、completion_mono_ns、target_*_tick、bdev/data 段） | 同 | 可用于逐 128KiB 时钟 |
| 唯一实测行为差异 | 每批突发 **16×128KiB（2MiB）** | **32×128KiB（4MiB）** | 来自 NOF-057 的 2MiB-child 时代；vseg4 时代 TRACE_US=0，突发形态 `[未获取]` |
| T1 每 key 串行 dt | 545±10µs（span 0.800s） | 264µs（前 240 op ~540µs=盘首区） | vseg 的每 key 周期稳慢 |

**结论**：没有单点可解释 2× 的现成差异；需要 M3-1a 的逐 128KiB 时钟 + task 级 breakdown，并在**同一份数据**上跑 plain 读路径做隔离。

**NOF-061（M3-1，已下发，零构建、target 不重启）**：
1. 新建 vseg 集群用**新 cluster_id `vseg_nof061`**（旧 `vseg_nofvseg` 的 plain 元数据不动），stripe=4MiB/slot=1 写 400 keys，vseg 读（`SPDK_URMA_STAGE_TRACE_US=1` + task breakdown）；
2. 停 vseg master，**不重写**，用旧 plain 元数据起 plain master，读**同一批 LBA 上的 vseg 数据**（同样 trace）；
3. 判读：plain 在同一数据上若快（~60µs/IO）⇒ **vseg 读路径/会话是原因**；若也慢 ⇒ 数据-读路径交互。同时取 vseg4 突发长度（32 还是 16）与 task 级 drain/fifo/credit。
4. 旧 plain 元数据缺失/读取失败 → 停手，不重写、不破坏 vseg 数据。

## 1. 与总目标的关系

### 1.1 原始目标（交接文档）

- 单 key（4–8 MiB）读时延 **750µs → 300µs**（交接文档 L5）。
- 用户的硬约束：**开了 vsegment 的性能一定要比没开 vsegment 更好**，否则 vsegment 没有意义。

### 1.2 第一阶段已经确认的事实

| 口径（修复慢窗口后） | plain NoF | vsegment（最好配置） |
|---|---|---|
| 单线程 per-IO p50 | **86µs** | **274µs**（3.2×） |
| 单线程 target W9（盘执行段） | **68µs** | **255µs**（3.75×） |
| 16 线程 per-IO p50 | **359µs** | **414µs**（1.15×） |
| 16 线程端到端 P50 | **718.7µs** | **735.3µs**（+2.3%） |
| 300µs 目标 | 16 并发下不可达 | 更不可达 |

结论：**单 target / 单 RAID 拓扑下，vsegment 没有结构性读时延优势**（跨成员条带在 member_count=1 时不生效）；当前实现下它比 plain 慢，16 并发基本持平。

### 1.3 第二阶段目标

1. **先解决"vseg 路径为什么每条 IO 多 210µs"**——这是当前实现的核心缺陷；
2. 目标是让 vseg 的 target 侧 bdev 执行接近 plain（~70–100µs），端到端单线程接近 plain，16 并发不劣于 plain；
3. 只有先把这条路径拉平，才有资格谈 300µs 与"vseg 必须更快"；结构性反超仍需多 target/多成员或分层缓存（不在本阶段范围，但保留为后续方向）。

---

## 2. 当前问题（矛盾点）

### 2.1 现象

- vseg 的每条 128KiB 读在 **target 的 bdev 执行段（exec→bdev completion）** 是 ~250µs；plain 是 ~46–68µs。
- 该差异**不是**：测量伪影、布局、分区数、stripe、child 数、写线程数、写侧 lane 数、qpair/lane 数、提交顺序、target 落核、target 到达/执行顺序、传输段、客户端段、盘首慢窗口（高 LBA 区）。
- 该差异**全部发生在盘执行段**：客户端 `data_us` 5µs（plain 14µs）、`outside_us` 16µs（plain 23µs）——vseg 的传输与客户端反而更快。

### 2.2 决定性证据（NOF-045/046）

- **同一 target、同一批 LBA、同一份数据**，裸 SPDK `urma_perf` 顺序 QD1 读：
  - vseg 写区 [1.6, 2.6) GiB：**p50 40.96µs / target W9 91.57µs**
  - plain 写区 [3.2, 4.2) GiB：**41.28 / 91.28µs**
  - 同区随机读：**220.16 / 188.90µs**（盘对访问模式极敏感）
- **Mooncake-vseg 读同一 vseg 区：274µs / W9 255µs**。
  ⇒ 数据/盘无罪；慢在 vseg 这条路径。
- **target 侧顺序证否（NOF-046）**：vseg q1（单 lane）在 target 的接收/执行/提交顺序与 plain **完全一致**（0 逆序、0 位移、单 lcore、submit=exec 100%），bdev 仍 223–265µs；plain 同 lcore 43–52µs。
  ⇒ 连"盘把 vseg 看成随机访问"也不成立。
- **NOF-057 逐 IO 更新（2026-11-23，新 target/bd5ee124d + BUFFER，见 §0.12）**：126µs **100% 在 `submit_return→callback`（SSD 执行段）**，其余 6 段为 0；**NUMA 排除**；两臂 lba 集合/nlb/flags 完全相同。唯一剩余差异是**盘级访问流**（换盘率 79.7% vs 50.0%、每盘回退 31 vs 1）。NOF-046 的"q1 严格顺序"结论基于旧 target 的粗粒度 tick（同 tick 批量可能掩盖组内乱序），以 NOF-057 的逐 IO 数据为准；但 NOF-034（单 child 仍慢）说明**表面顺序/child 数不能直接当根因**。

### 2.3 当前唯一未解矛盾（NOF-060 后更新）

> **命令相同、NUMA 无关、126µs 在 SSD 服务时间；同盘并发（ov）与局部 LBA 顺序（gap/busy）都解释不了（NOF-058）；每 key 单 4MiB child 与 plain 同构后仍慢 2.04×（NOF-059）；同一份数据裸 SPDK 顺序读 40µs、随机 220µs 两臂相同（NOF-059）；两臂的 NoF 提交层完全同构（同一函数/参数/128KiB chunk/8MiB 在飞/32 IO 同 qpair，NOF-060）。**

结论：**child 数、数据物理状态、局部并发/顺序、客户端提交代码全部排除**。差异只剩：
1. **vseg 读路径的元数据/会话层**（vseg master 解析、endpoint/segment 会话、target 侧为该会话分配的资源），或
2. **数据与 Mooncake 读路径的交互**（裸读快，但 Mooncake 读的 target 数据通路可能不同）。

**NOF-061（M3-1，已下发，零构建、target 不重启）** 用**同一份 vseg 数据 + plain 读路径**做隔离：plain 快 ⇒ 差异在 vseg 路径/会话；plain 也慢 ⇒ 数据-读路径交互。同时取 vseg4 的突发长度（32/16）与逐 128KiB/逐 task 时钟。

### 2.4 已排除项总表

| 维度 | 证据单 | 结论 |
|---|---|---|
| IO 尺寸/数量 | NOF-031/035 | 两边都是 128KiB × 32/key |
| LBA 布局 | NOF-033/035 | 两边都严格连续，>1MiB 跳变 0 |
| 写线程数 | NOF-037 | T1 vs 16t 写，读 p50 86 vs 84µs |
| 盘首慢区 | NOF-040 / 045 | 只影响低 LBA（[0,~1GiB)），高 LBA 数据无罪 |
| qpair/lane（读） | NOF-024/040 | 4→1 仅 −2.6% |
| 写侧 lane | NOF-043 | q1 串行写的新数据仍 253µs |
| 提交顺序 | NOF-042 | 每 lane 严格递增，q1 全局单调 |
| target 落核 | NOF-042 | 四核与 plain 相同，vseg 四核全线慢 |
| target 到达/执行顺序 | NOF-046 | q1 与 plain 完全一致，0 逆序（**粗粒度 tick，NOF-057 逐 IO 复测下存疑**） |
| 命令内容（lba/nlb/flags） | NOF-057 | 两臂 lba 集合完全相同、nlb 全 256、flags 全 3 |
| NUMA（盘/核/设备节点） | NOF-057 | vseg node0 256 / node2 255，plain 59/61 ⇒ 无关 |
| 126µs 所在段 | NOF-057 | **100% 在 `submit_return→callback`（SSD 执行段）**，其余 6 段为 0 |
| 盘级访问流 | NOF-057 | 两臂不同（换盘 79.7% vs 50.0%、回退 31 vs 1） |
| 同盘并发重叠（ov） | NOF-058 | **排除**：ov=0 时 vseg p50 206 vs plain 39；两臂 ov 分布同（34.4%） |
| 局部 LBA 顺序（gap/busy） | NOF-058 | **排除**：紧接前驱 275 vs 61、busy=False 206 vs 39、同 gap 桶 207 vs 41 |
| 数据物理状态（写者/写轮次） | NOF-059 | **排除**：同一数据裸读顺序 40.0 vs 40.8µs、随机 220.16 vs 220.16、W9 86.3 vs 79.6 |
| child 数 | NOF-059 | **排除**：stripe=4MiB 单 child 已验证（528/8192/child=0/1），P50 仍 2.04× |
| 客户端 NoF 提交层（函数/参数/chunk/lane/qpair） | NOF-060 | **排除（完全同构）**：同一 `submitSpdkNofOperation`、同参数、128KiB、8MiB/lane、32 IO 同 qpair；child 仅日志标签 |
| vseg 元数据/会话层（vseg master 解析、endpoint/segment、target 会话资源） | NOF-061 | **已定案为本行**：同一份 vseg 数据用 plain 路径读 bdev 57µs vs vseg 249µs；同 LBA 配对 4.64× |
| 会话/qpair 身份的**作用机制**（完成检测/唤醒 vs 每命令固有 vs 恒定排队） | NOF-062 | **已定案：完成检测/唤醒类**——gap T1 195µs → T4 3µs → T16 5µs，差异 100% 在 bdev_us |
| 那 195µs 在源码里的来源（阻塞等待/超时/批量/事件合并/轮询预算） | NOF-063 | **已排除常量/阻塞**：无 50–500µs 常量、无 sleep；结构错配两臂对称 ⇒ 不解释 |
| 设备实际收到的子命令与 target 轮询节奏（插桩） | NOF-064 | **受阻**：重启使 vseg 元数据失效 + v1 插桩错位/打印污染 |
| 零干扰插桩下的 iov 字段与轮询 delta | NOF-065 | **待查**：环形缓冲 + 真 delta + `iov[0].base` NUMA 桶 |
| 测量伪影 | NOF-035 | plain 96.9→99.14 可复现 |
| 传输/客户端 | NOF-041 | data/outside 反而更小 |
| 连接预热 | NOF-039 | warmup 不能跳过；慢区是固定段 |
| "盘看成随机档" | NOF-046 | q1 严格顺序仍慢，解释不成立 |

---

## 3. 待验证假设（按优先级）

### H1（最高优先）：target 的 bdev 提交参数不同

vseg 与 plain 发给 target 的 NVMe 命令看似相同，但 target 在**提交 bdev 读**时使用的参数可能不同：

- 数据 buffer 的**类型/来源**：本地 iobuf vs 直接指向 initiator 内存（零拷贝/暂存）；
- **SGL/描述符**：单 SGE vs 多 SGE、rkey/内存域不同；
- **io_flags**、raid0 **stride 对齐**、iobuf 池类型（small/large）；
- bdev channel/线程绑定。

若其中任一不同，会改变 SSD 的实际读路径（例如走暂存拷贝或不同的 NVMe 队列），表现为 bdev 段变慢。

**验证方式**：在 target 的 `lib/nvmf/urma.c` 里对每条读命令打印"解析后的 SGL（地址/rkey/长度）、数据 buffer 类型、bdev 名、io_flags、iobuf 类型/地址"，以及 `rx → bdev_submit → nvme_cpl → bdev_cpl → data_post → data_cpl` 的时间戳；对比 plain vs vseg 同一 LBA 区。

### H2：客户端的 capsule/SGL 与 plain 不同

vseg 的 `submitVSegment` 为每个 child 调用 `submitSpdkNofOperation`，虽然 payload 指针与长度在 Mooncake 层连续，但 SPDK 客户端构造 capsule 时可能因指针/长度/注册域不同而产生不同的 SGL 或 rkey。

**验证方式**：在客户端 SPDK `lib/nvme/nvme_urma.c` 的读提交处打印 payload 指针、长度、MR/rkey、SGL 数量、qpair/jetty；对比 plain vs vseg。

### H3：target 的完成回调被数据发送/流控延迟

bdev 可能很快完成，但 target 的完成回调要等 URMA 数据发送/credit，导致 `submit→callback` 变长。现有 `URMA_BDEV_DETAIL` 的子段退化（`submit_to_callback ≡ bdev_us`），无法区分"NVMe 慢"与"callback 被延后"。

**验证方式**：同 H1 的时间戳拆解，并在发送路径上打印 credit/发送完成时刻。

### H4：客户端 buffer 不在注册域内，触发慢路径

bench 的线程 buffer 由 `numa_alloc_onnode` 分配后 `client_->register_buffer` 注册；vseg 的 child 指针为 buffer+2MiB，理论上在同一注册域内，但需实测确认。

**验证方式**：在 Mooncake 客户端打印每个 task 的 `ptr`、注册域范围、`NofMemoryDomainManager::ResolveBuffer` 的 type/domain；确认 vseg 与 plain 一致。

---

## 4. 代码改造方案（分里程碑）

### M0：建立我们自己的可重建构建树（前置）——已完成（NOF-047，2026-11-23）

**已复制并校验（全部 md5 MATCH，共 8.82 GiB）**，位于 `/home/l00955908/mooncake-nof-vsegment/build/`：

| 副本 | 大小 | 校验 |
|---|---|---|
| `mooncake-src/` | 197M | HEAD be10be2、status 2 行（go.mod/go.sum）MATCH |
| `mooncake-build-colleague/` | 7.7G | `stress_cluster_bench` md5 59662a26… MATCH |
| `spdk-src/` | 763M | HEAD 9a413a44…、status 0 行 MATCH；含自带 build/ |
| `spdk-build/` | 298M | `nvmf_tgt` d6c39465…、`libspdk_nvmf.a` 107a761b… MATCH |

**构建配置（来自同事 CMakeCache）**：Release；USE_NOF=ON、USE_NOF_URMA=ON、USE_UB=ON、USE_NOF_GPU_DIRECT=OFF、USE_UBSHMEM=OFF、YLT_ENABLE_URMA=ON、STORE_USE_ETCD=ON；SPDK_INCLUDE_DIR=/usr/local/include、SPDK_LIB_DIR=/usr/local/lib；NOF_URMA/URMA lib=/usr/lib64/liburma.so。

**重要约束（M1 必须遵守）**：

1. **SPDK 纯静态**：`spdk-build/lib` 74 个 `.a`、0 个 `.so`，`/usr/local/lib` 同样纯静态 ⇒ 插桩后不能靠 `LD_LIBRARY_PATH` 换库，必须**用我们的 `.a` 重新链接** bench/nvmf_tgt。
2. **路径耦合**：`mooncake-build-colleague/CMakeCache.txt` 与 `spdk-src/mk/config.mk` 内仍写死 `/home/yin/nof/...`；新建 build 目录重配置或做最小重链，不能直接依赖同事原树。
3. **正在跑的 target 是 deleted 旧二进制**：PID 341038 的 `/proc/exe` md5 **736a902f…**，磁盘 `spdk-src/build/bin/nvmf_tgt` 是 **d6c39465…** ⇒ target 侧插桩生效必须换进程启动（需用户批准，并先完整抓取/复现 RPC 配置）。
4. 冗余 298MB（`spdk-src/build` 与 `spdk-build` 相同）：暂保留作回退参考。

**补丁定位（已确认）**：

- target 读提交点：`spdk-src/lib/nvmf/ctrlr_bdev.c:466-467`（p99_read_enter_tick）；`URMA_BDEV_DETAIL` 在 `spdk-src/lib/nvmf/urma.c`；
- 客户端 capsule/SGL 构造：`spdk-src/lib/nvme/nvme_urma.c:995-1056`；
- Mooncake：`submitSpdkNofOperation` 在 `mooncake-src/mooncake-store/src/transfer_task.cpp:3047`，`ResolveBuffer` 在 `1080-1106`。

**最小重建已验证（NOF-048，2026-11-23）**：

- 编译 `transfer_task.cpp` **61s**（rc=0）、链接 **4s**（rc=0）；新 bench 530,148,040B / md5 `43ba518d…`；`--help` 与原版仅差第 1 行程序名。
- 配方（`D=/home/l00955908/mooncake-nof-vsegment/build`，`B=$D/mooncake-build-colleague`，`W=/tmp/nof048`）：
  1. `cd $B/mooncake-store/src && bash $W/compile.sh`（取自 `compile_commands.json`，仅改输入/输出路径）；
  2. `cp` 原 `libmooncake_store.a` 到副本 → `ar r` 用新 `transfer_task.cpp.o` 替换同名成员 → `ranlib`；
  3. `cd $B/mooncake-store/benchmarks && bash $W/link.sh`（取自 `link.txt`，把 `../src/libmooncake_store.a` 换成副本、输出改名）。
- **三个坑**：
  1. 编译命令的 `-I` 全部指向**同事树** ⇒ 改 `transfer_task.cpp` 生效，但若改**头文件**，必须前置 `-I$D/mooncake-src/.../include`；
  2. 链接仍用 `/usr/local/lib/libspdk_*.a`（静态）⇒ 客户端/`target` 插桩若需换 SPDK 实现，必须改用 `$D/spdk-build/lib` 的 `.a` 重链；
  3. 新 bench 的 RPATH 指向同事 `mooncake-common/etcd` 构建目录 ⇒ 依赖同事树存在（如需独立，M1 一并改指副本）。

**M0 之后的当前步骤 = 复盘后的 S0（NOF-049）**：诊断开/关对照（零构建），判定 210µs 是否来自默认诊断/客户端背压；NOF-048 的配方留给 S2/M1 的代码迭代。

### M1：插桩（只加日志，不改行为）

**M1a 客户端（Mooncake + 客户端 SPDK）**

- `mooncake-store/src/transfer_task.cpp`（提交路径）：每个 task 打印 `vseg_op_id/child_index/child_count`、`ptr`、`target_lba`、`length`、`lane`、`qpair_id`、`buffer_info.type/domain`；
- `mooncake-store/src/spdk/spdk_wrapper.cpp`（whole-pool registration）：打印注册的 buffer 基址/长度/覆盖的 qpair；
- 客户端 SPDK `lib/nvme/nvme_urma.c`：每条读打印 payload 指针、长度、MR/rkey、SGL 数量、qpair/jetty、提交/完成 tick。

**M1b target（SPDK）**

- `lib/nvmf/urma.c`：每条读打印 `trace_seq/qid/cid`、NVMe `lba/nlb`、解析后的 SGL（地址/rkey/长度）、数据 buffer 类型（本地 iobuf/主机内存）、bdev 名、`io_flags`、iobuf 类型/地址；
- 增加 `rx → bdev_submit → nvme_cpl（若可得）→ bdev_cpl → data_post → data_cpl` 的分段 tick；
- 只在**受控窗口**打印（按 trace 请求/采样），避免日志爆炸。

### M2：采集与定位

- 在同一 target、同一 LBA 区（[1.6, 2.6)GiB）、同样 QD1 条件下，各跑一遍 plain 与 vseg 的受控读；
- 对比 M1a/M1b 的逐 IO 字段，找出 plain 与 vseg 在 **bdev 提交参数/SGL/数据 buffer/时间戳** 上的差异；
- 输出"210µs 落在哪一段"的结论（若仍拆不开，进入 M1c：在 raid0/子 NVMe 上加更细打点）。

### M3：修复

按 M2 的结论选择修复方向（示例，视证据而定）：

- 若数据 buffer/SGL 不同 ⇒ 统一 vseg 的 buffer 描述（对齐/注册域/单 SGE）；
- 若 target bdev flags/stride/iobuf 不同 ⇒ 修正 target 提交路径；
- 若 callback 被流控延迟 ⇒ 调整 target 发送/credit 或客户端接收缓冲。

### M4：验证与回归

- vseg T1：target bdev ≤ ~100µs、per-IO p50 ≤ ~110µs、端到端 ≤ plain+10%；
- vseg 16t：不劣于 plain 16t；
- plain 路径无回归；verify 0 错；数据一致性。

---

## 5. 验收标准（第二阶段）

1. **机制定案**：用插桩数据指出 210µs 的确切来源（某一段/某参数），可复现；
2. **修复有效**：vseg T1 的 target W9 从 ~255µs 降到 ≤ ~100µs，per-IO p50 从 274µs 降到 ≤ ~110µs；
3. **不劣化**：16t 下 vseg ≤ plain；plain 不回归；
4. **可回退**：所有改动以构建副本 + 备份 + md5 管理，任何一步可回到第一阶段末尾状态。

---

## 6. 风险与回退

| 风险 | 处置 |
|---|---|
| 构建不可复现/依赖缺失 | M0 先原样重编验证；失败则记录缺口（工具链/依赖），不盲改 |
| 改同事源码树 | **只在我们的副本上改**；原树只读；保留 md5 |
| target 重启/重编 | 每次 target 重启前征得用户明确批准；保留原二进制与启动/RPC 配置；失败按原参数无插桩恢复 |
| 日志爆炸 | 插桩按采样/受控窗口打印；单次运行日志上限 200MB，超出停手 |
| 代码外传 | 只回传 diff（base64 + md5）与文字规格，不回传整文件/大段日志 |
| 数据损坏 | urma_perf preflight 已污染 keys 0/128/400/800；第二阶段用新集群/新 key；所有实验前记录 etcd/主数据状态 |

---

## 7. 环境与协作约束（沿用第一阶段）

- 端口：target 4457、master 50056、metrics 9006、client TE 50156；不用默认端口。
- 工作目录：`/home/l00955908/mooncake-nof-vsegment`（node1/node3）。
- 共享机规则：不动同事进程/文件；node3 用户独占；重启/高危操作先经用户批准。
- 任务机制：HANDOVER + NOF 编号，一单一阶段；回执走会话。
- 基准口径：排除盘首慢区（keys 0–96 / LBA <388MiB）；warmup 写入 128（不能跳过，仅保持口径一致）；统计同时给全量与排除两套。

---

## 8. 关键环境快照（第二阶段起点）

- target：node3 PID 341038（yin `nvmf_tgt`，md5 `d6c39465…`），4457，URMA_RAID0（11 盘，strip 256KiB）；cmdline `-m [2-7] --wait-for-rpc`，子系统经 RPC 下发。
- vseg master：node1 PID 2077841（vseg-1 + stripe=2MiB、slot=1，Keys=800、logical_used 3,355,443,200）。
- 已知污染：keys 0/128/400/800 各 128KiB 被 urma_perf preflight 覆盖。
- 二进制/源码：Mooncake 公开仓 `github.com/LeeJace993/Mooncake` branch `mooncake-vesegment`，**构建基线 be10be2**（+ go.mod/go.sum 两处本地改动）；SPDK 为 yin `spdk-cpu`（HEAD e12b7cb + 工作区改动）。

---

## 9. 附录：关键数字（第一阶段定案）

- plain 干净 T1：per-IO p50 86µs、target W9 68µs；16t：359µs / 161µs。
- vseg 干净 T1：per-IO p50 274µs、target W9 255µs；16t：414µs / 270µs。
- 裸 SPDK 同区顺序 QD1：p50 41µs、target W9 91µs；随机 220µs / 189µs。
- 188µs 增量全部在 `bdev_us`；`data_us` 5 vs 14、`outside_us` 16 vs 23。
- 低 LBA 慢区：裸读 [0,~1GiB) 也是 212–215µs（上边界 1.0–1.6GiB 之间）。