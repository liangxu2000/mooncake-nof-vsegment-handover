# 第二阶段：消除 vseg 读路径的 +210µs

- 日期：2026-11-23
- 阶段：第一阶段（NOF-001~046）已收口；本文档为第二阶段核心目标的开工依据
- 目录约定：**第二阶段所有内容（文档、代码补丁、测量数据）统一放在 `第二阶段/` 目录下**；本文件为总文档/索引
- 数据来源：`../HANDOVER.md` 各单回执 + 第一阶段最终报告 `../工作汇总.md`
- 相关文件：`../HANDOVER.md`（任务通道）、`../工作汇总.md`（第一阶段最终报告）
- **当前状态（2026-11-23）**：M0 已完成、NOF-048 最小重建已验证（65s/轮）；**S0 已完成（诊断排除：关诊断 W9 仅 −2.7%）**；核对 tip diff 后 **S2（构建 tip）暂缓**（NoF 读提交未变）；当前进入 **M1a 客户端插桩（NOF-050，补丁在 `patches/`）**。详见下方第 0 节、「最小重建配方」与 `patches/README.md`。
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
| **M1a** | 客户端 NoF 参数插桩（`patches/nof050_client_io_params.patch`）+ vseg 采集 | **进行中（NOF-050）** | ptr/seg_lba/child/lane/buf_type/domain 是否与 plain 一致、ptr 是否在注册域内 |
| **M1b** | 切换 plain，同口径采集与对比 | 下一单（NOF-051） | 找出客户端侧差异；若无差异 ⇒ target 插桩 |
| **M1c** | target 侧插桩（SGL/bdev 提交参数/分段 tick） | 需重编 target + 重启（用户批准） | 定位 210µs 在 raid0 队列/NVMe/回调哪一段 |

- NOF-048（最小重建验证）保留为 S2 的构建工具准备，但**构建基线改为 tip**，不再以 be10be2 为最终目标。

---

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

### 2.3 当前唯一未解矛盾

> **target 看到的顺序、IO 大小（128KiB）、数量、盘区（裸 SPDK 该区 41µs）全部相同，bdev 执行时间却差 5 倍。**

现有遥测（客户端 `URMA_SLOW_IO`、target `URMA_BDEV_DETAIL`、W 块）能看的字段已经穷尽；差异只能落在**现有遥测看不到的、target 在提交 bdev 时携带/使用的参数**上。

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
| target 到达/执行顺序 | NOF-046 | q1 与 plain 完全一致，0 逆序 |
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