# 交接文件（单向：开发指挥 Agent → 测试执行 Agent）

- 仓库：`https://github.com/liangxu2000/mooncake-nof-vsegment-handover`（public，匿名 HTTPS 可拉取）
- 首次接入：`git clone https://github.com/liangxu2000/mooncake-nof-vsegment-handover.git`
- 之后每轮：`cd mooncake-nof-vsegment-handover && git pull`
- 权限：**只 pull，不要 push**；本仓不接收反向提交
- 回执：按下方《回执格式》在会话里输出，由用户转交开发侧；回执不写回仓库
- 最后更新：2026-11-23
- 当前任务：**NOF-037（写模式对照·续：用新增 key 扩展法让 T1/16t 写真正落盘，按 LBA 桶比较读时延）**

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
| NOF-037 | 已下发（2026-11-23） | 写模式对照·续：用 `--num_keys` 扩展法写新 key（不删不清集群），三桶（旧 16t 写 / 新 T1 写 / 新 16t 写）按 LBA 桶比较客户端 128KiB roundtrip，判定写模式是否决定读慢 |
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

### 决策点（2026-11-23，NOF-036 后更新）

- 最终报告已交付（`工作汇总.md`）；NOF-035 否证"测量伪影"；NOF-036 的写模式对照因无法覆盖写而未完成，但复读稳定性再次确认（96–110µs）。
- **开发侧决定（2026-11-23，用户授权"你来决策"）**：用**不删 key、不清集群**的 `--num_keys` 扩展法补做写模式对照（NOF-037）：旧键 0–399（NOF-035 的 16t 写）/ 新键 400–799（T1 写）/ 新键 800–1199（16t 写），一次 1200-key 读按 LBA 桶比较客户端 128KiB roundtrip（桶 1/3 是对照，桶 2 是处置）。
  - T1 写桶变慢 ⇒ 写路径决定读慢 → 下一单测 vseg-1+stripe4 的 16t 写；
  - 三桶都 ~97 ⇒ 写模式无关 → 再申请重启 target 开逐 IO 明细（下面第 2 条）。
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

# NOF-037（待执行 · 写模式对照·续：新增 key 扩展法，按 LBA 桶比较读时延 · 2026-11-23）

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
