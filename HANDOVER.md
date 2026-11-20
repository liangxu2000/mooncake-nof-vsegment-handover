# 交接文件（单向：开发指挥 Agent → 测试执行 Agent）

- 仓库：`https://github.com/liangxu2000/mooncake-nof-vsegment-handover`（public，匿名 HTTPS 可拉取）
- 首次接入：`git clone https://github.com/liangxu2000/mooncake-nof-vsegment-handover.git`
- 之后每轮：`cd mooncake-nof-vsegment-handover && git pull`
- 权限：**只 pull，不要 push**；本仓不接收反向提交
- 回执：按下方《回执格式》在会话里输出，由用户转交开发侧；回执不写回仓库
- 最后更新：2026-11-20
- 当前任务：**NOF-018（writer 重灌 400 keys 到新 target）**

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

# NOF-018（待执行 · writer 重灌 · 2026-11-20）· 新 target 上写入 400 keys

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
