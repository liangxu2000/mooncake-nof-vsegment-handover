# 交接文件（单向：开发指挥 Agent → 测试执行 Agent）

- 仓库：`https://github.com/liangxu2000/mooncake-nof-vsegment-handover`（public，匿名 HTTPS 可拉取）
- 首次接入：`git clone https://github.com/liangxu2000/mooncake-nof-vsegment-handover.git`
- 之后每轮：`cd mooncake-nof-vsegment-handover && git pull`
- 权限：**只 pull，不要 push**；本仓不接收反向提交
- 回执：按下方《回执格式》在会话里输出，由用户转交开发侧；回执不写回仓库
- 最后更新：2026-11-20
- 当前任务：**NOF-004（启动我方 master：node1、50056/9006、全新 cluster_id）**

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

### 口径与数据安全（2026-11-20 用户补充）

- **数据保留**：GAP 旧数据是盘起始连续的 ~2.47 TB（294,669 keys × 8 MiB）；用户要求尽量保留，**实在不行可清理、后续重新 prefill**。新 vsegment 段为 100GiB @ offset 0，writer 只写 400×4MiB=1.6GiB ⇒ 只覆盖最前 1.6GiB；若要零覆盖，需改造 `nof_register` 传入 `base=5TiB`（mount RPC 支持 `base`，该 CLI 写死 0），暂不做。
- **拓扑事实（同事脚本）**：master/etcd/writer 在 node1，target/reader 在 node3 ⇒ 写路径跨节点、读路径数据与 target 同机；reader 侧 roundtrip−target≈29µs，属**本机口径**。我们计划 reader 两种位置都测（node3 对齐同事；node1 远端端到端），最终口径以交接文档 §一 为准（NOF-005 附带取证）。
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

# NOF-004（待执行 · 启动我方 master · 2026-11-20）· 全新 cluster_id + 我方端口，验证干净环境

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
