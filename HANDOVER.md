# 交接文件（单向：开发指挥 Agent → 测试执行 Agent）

- 仓库：`https://github.com/liangxu2000/mooncake-nof-vsegment-handover`（public，匿名 HTTPS 可拉取）
- 首次接入：`git clone https://github.com/liangxu2000/mooncake-nof-vsegment-handover.git`
- 之后每轮：`cd mooncake-nof-vsegment-handover && git pull`
- 权限：**只 pull，不要 push**；本仓不接收反向提交
- 回执：按下方《回执格式》在会话里输出，由用户转交开发侧；回执不写回仓库
- 最后更新：2026-11-23
- 当前任务：**NOF-068（M3-6：改用 SPDK 原生 tracepoint 采两臂 · 零改码 · 含一次干净重启 · 2026-11-24）**

## 第二阶段（2026-11-23 起）

- **第一阶段（NOF-001~046）已收口**：单 target 下 vsegment 无读时延优势；核心未解问题已定位到 **vseg 读路径在 target 的 bdev 执行段多 ~210µs**（数据/盘无罪：裸 SPDK 同区 41µs；target 侧顺序/落核/命令与 plain 完全一致）。
- **第二阶段目标**：消除这 210µs，让 vseg 的 target bdev 接近 plain（~70–100µs）、端到端不劣于 plain；之后再谈 300µs 与多 target 结构性优势。
- **计划与假设（H1–H4）、里程碑（M0–M4）、验收与回退**：见 `第二阶段/README.md`（第二阶段所有内容统一放在 `第二阶段/`）。
- **M0（NOF-047）已完成**：复制并校验 Mooncake/SPDK 源码与 build（8.82GiB，md5 全 MATCH）。
- **最小重建（NOF-048）已验证**：改一个 .cpp → 编译 61s → `ar r` 换库成员 → 重链 4s，产物 `--help` 正常；M1/S2 的代码迭代成本约 65s/轮。
- **2026-11-23 复盘纠偏**：发现两处高价值线索——①构建基线 **be10be2 已落后远端 tip 4c0ea28**（含 `ff3e853` 诊断 opt-in/低时延默认、`13b7c19` 去重复校验等读路径修复）；②旧构建 `MC_HIFREQ_LOG_SAMPLE_RATE` **默认 1.0**，per-task 计时/breakdown 日志默认全开（tip 的提交明确说这会抬高读时延），而我们**从未测过"诊断全关"的 vseg**。
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

- **盘首慢窗口（已知盘特性）**：每块盘开头约 **36MB** 是慢区；RAID0 11 盘 ⇒ 命名空间前 **~396MB**（我方 4MiB key 实测为前 **97 个 key ≈ 388MiB**）读时延 ~290µs，之后恢复 ~97µs（NOF-037 实测）。
- **规则（2026-11-23 NOF-039 后修订）**：本 bench 的 `--warmup_keys` **不能跳过**该窗口——warmup 阶段与测量阶段读同一批 key 都是 ~245µs，且测量循环仍会重读这些 key（NOF-039 实证：warmup=128/400 均无效，keys 0–96 慢占比 90.9%/91.0%）。**今后统一用统计排除口径**：分析时剔除 keys 0–96（LBA < 388MiB）及 >396MB 的等价区间，报告同时给出"全量"与"排除慢窗口"两套数字，并在表头标注。warmup 仍写 128（保持口径一致），但不把它当作跳过手段。
- 该慢窗口**不能解释** vsegment 的 250–274µs（vseg 是全 key 均匀慢，不是前 97 个 key）。
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
| NOF-016 | 已关闭 2026-11-20（**通过**） | yin target 已起：PID 341038、md5 736a902f…、listener **192.168.84.149:4457**、strip 256K/11 成员、dev udmac0d1e2；旧三进程全退；我方键清空、yin 键未动；master 重启 PID 2462234（leader/serving、enabled=0）；回滚快照已存 |
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

- **新 target**：yin `nvmf_tgt`（md5 `736a902ffac8ce608acd22c5459f643d`，2026-11-17 19:20），PID 341038；listener **URMA 192.168.84.149:4457**（fabric 网段，旧地址 141.61.84.149 已拒绝）；NQN `nqn.2026-01.io.spdk:urma-gpu-test`，nsid 1 → `URMA_RAID0`（strip 256 KiB、11 成员、raid uuid `5805fd63-…`）；transport dev_name `udmac0d1e2`、max_io_size 8 MiB、capsule_transport=urma。
- **旧环境退出**：我方 master 2162056（2 s）、gap master 3673976（1 s）、gap target 3243095（3 s）；4457 释放；无 nvmf_tgt/spdk_tgt 残留。
- **etcd**：我方 `/cvm/vseg_nofvseg/` 6→0、`/oplog/vseg_nofvseg/` 1228→0；yin 与 mooncake_cluster 键数不变。
- **我方 master 重启**：PID 2462234（16:31:45），leader/serving，50056/9006，`enabled=0`；`/cvm/vseg_nofvseg/` 3 键、`/oplog/` 1 键；env `udmac0d1e2` 自证。
- **地址/身份变化**：listener 由管理网改为 fabric；raid uuid 变化；NOF-017 register 必须用 **192.168.84.149:4457**；旧 device_identity 冲突不存在（键已清）。
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

- NOF-016：yin target 在 **192.168.84.149:4457**（URMA，UDMAC0D1E2，strip 256K，11 成员）；我方 master PID 2462234（leader/serving，`enabled=0`）。
- 本单用**新地址**重新 mount（100GiB、base 0、nsid 1），验证 1024 分区 reconcile 门禁。**不写数据**；不跑 writer/reader。

## 1. 预检（只读）

1. master：PID 2462234 存活；50056/9006 LISTEN；etcd `/cvm/vseg_nofvseg/` 3 键、无 quota 快照；`/oplog/vseg_nofvseg/` 1 键。
2. target：node1 → `192.168.84.149:4457` 建连 OK；node3 `ss -lnt` 4457 正常；`nvmf_get_subsystems` 与 NOF-016 D1 一致。
3. 记录 target 日志 `/tmp/nvmf_tgt.log` 当前 W 块基线（n 应为 0）。

## 2. mount（唯一一次，零额外 env）

- 在 node1 用 `nof_register`（md5 `1655b48a…`），root，输出 `run/register_r3.log`：
  ```
  <build>/mooncake-store/src/nof_register \
    "nqn.2026-01.io.spdk:urma-gpu-test" 1 192.168.84.149 4457 URMA \
    <node1 192.168.84.*:50056> 107374182400
  ```
- **不要** export 额外 env。

## 3. 验证（只读）

1. master 日志（≤40 行）：`SPDK initialization … init_rc=0`、`Attaching to 192.168.84.149`、`Attached`、`Opened NoF segment`、`MountNoFSegment_slow … rc[?]`；任何 attach/URMA ERROR。
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
2. 连接参数：traddr `192.168.84.149`、trsvcid `4457`、NQN `nqn.2026-01.io.spdk:urma-gpu-test`、nsid 1；env `SPDK_URMA_DEV_NAME=udmac0d1e2`、`URMA_PERF_ADDR_MODE=seq`（按 `--help` 实际参数调整，**不改源码**）。
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

- 连接：traddr `192.168.84.149`、trsvcid `4457`、NQN `nqn.2026-01.io.spdk:urma-gpu-test`、nsid 1；env：`SPDK_URMA_DEV_NAME=udmac0d1e2`、`SPDK_URMA_CAPSULE_TRANSPORT=urma`、随机支加 `URMA_PERF_ADDR_MODE=rand URMA_PERF_RANDOM_SEED=20261115`，`--range-bytes=107374182400`（随机范围=我们的 100GiB 段）。
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
4. `nof_register` 同命令挂载 target（192.168.84.149:4457、100GiB、base0、nsid1）→ 确认 mount rc[0]。

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
3. `nof_register` 同命令（192.168.84.149:4457、100GiB、base0、nsid1）；等 quota 发布 ≥20s。
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
3. `nof_register` 同命令（192.168.84.149:4457、100GiB、base0、nsid1）；等 quota 发布 ≥20s。
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
3. `nof_register` 同命令（192.168.84.149:4457、100GiB、base0、nsid1）；等 quota ≥20s。
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
   - 我方 target（PID 341038）是否存活；4457 是否 LISTEN 在 `192.168.84.149`；`/tmp/nvmf_tgt.log` 是否可写/仍在增长。
   - 是否出现其它非本方进程；HugePages、磁盘空间现状。
3. **判定**：若我方 master/target 任一缺失、50056/9006/4457 被他人占用、我方 etcd 键异常、或 node1 负载明显影响测试 → **停手回报，不执行实验**。

## B. 实验（仅当 A 判定干净）：vsegment stripe=4MiB（内容同 NOF-030）

1. 停我方 vseg-1 master（按 PID）；清**仅我方**键 `/cvm/vseg_nofvseg/` + `/oplog/vseg_nofvseg/`。
2. 启动 master：NOF-004/007 同参数 + **`--vsegment_stripe_size=4194304`** + `MC_STORE_PARTITION_SLOT_COUNT=1024`。
3. `nof_register` 同命令（192.168.84.149:4457、100GiB、base0、nsid1）；等 quota ≥20s；验证 quota `stripe_size=4194304`、partitions=1024、`reconcile_fail=0`、`partitions_installed=1024`。
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
- 本单**不跑任何新读、不动环境**，只解析已有日志，定位 vseg 多出的 ~185µs 落在哪一层，以及是均匀还是双峰。

## A. 输入（node3 `~/mooncake-nof-vsegment/run/`，只读）

- vseg：`nof040_vseg_t1_q4.log`、`nof040_vseg_t1_q1.log`、`nof040_vseg_16t_q4.log`
- plain：`nof039_read_t1_w400.log`、`nof039_read_16t_w400.log`
- 先打印 1 条完整 `URMA_SLOW_IO` 行，列出实际字段名；若某字段不存在，如实标注并用实际存在的字段替代。

## B. 解析口径

1. 仅 **opcode=2（读）、blocks=256（128KiB）**；剔除 **warmup 段**（日志开头的 warmup 读）与 **verify 大 IO**（blocks=4096/8192）；按 key 桶 **0–96 / 97–399** 分别统计。
2. 每份日志输出每字段 **n / p50 / p90 / mean**：`roundtrip_us`、`target_us`、`bdev_us`、`data_us`、`pre_send_us`、`outside_us`、`residual_us`（按实际字段名）。
3. **bdev_us 直方图**（6 桶：<50 / 50–100 / 100–200 / 200–300 / 300–500 / >500µs）：vseg T1 q4 与 plain T1 各一份；用 <100µs 与 ≥200µs 两簇占比描述是否双峰。
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
  2. vseg 的 bdev_us 是**均匀 ~250** 还是**双峰**（部分快、部分排队）？
  3. plain 同字段的分布（对照）。
  4. 从分布推断：更像"盘/队列执行慢"（bdev 均匀高）还是"传输/背压/排队"（data 高或双峰）。

## D. 边界

- **零环境操作**：不跑 bench、不动 target/master、不改任何文件；只读日志并用本地脚本解析。
- 回执只给汇总表、直方图计数与 ≤20 行样例；不贴原始日志、不外传数据。

## 回执必答（第一行）

vseg/plain 的 `bdev_us` 与 `data_us` p50/p90 及直方图（均匀还是双峰）；多出的 ~185µs 落在哪个字段；target_valid 缺失计数；有无失败。

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
3. 同一 RPC 序列（NOF-054）：iobuf → framework_start_init → `bdev_nvme_attach_controller` ×11（按 BDF/顺序）→ `nvmf_create_transport`（URMA、udmac0d1e2、max_io 8MiB 等）→ `nvmf_create_subsystem`（NQN）→ `bdev_raid_create -r raid0 -z 256` → `nvmf_subsystem_add_ns` → `nvmf_subsystem_add_listener -t URMA -f IPv4 -a 192.168.84.149 -s 4457`。
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
