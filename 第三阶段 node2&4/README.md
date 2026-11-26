# 第三阶段：node2/3/4 双 target 环境与 vseg 回归定位

> **2026-11-26：已进入第四阶段。** 后续任务、结果与回执摘要统一放在 `../第四阶段_node234/`；本目录保留第三阶段的历史定位、单 target 结果和三层参数说明，作为第四阶段的依据。

- 开始日期：2026-11-25
- 目录：`D:\mooncake nof vsegment\第三阶段 node2&4`
- 任务通道：`../HANDOVER.md`（NOF-104 起）
- 历史依据：`../接手以来重要结果.md`、`../第二阶段/README.md`
- 拓扑与产物只使用 **node1/node2/node3/node4** 指代，不写内网 IP。

---

## 1. 为什么进入第三阶段

第二阶段（node1/node3）已完成：

- 126µs 的公平差距 100% 落在 target 的 SSD 执行段；
- 数据、命令、描述符、child 数、NUMA、客户端提交代码均排除；
- vseg 仍比 plain 慢约 **2.07×**（L2），差距在 `read_us`；
- node1 的 host URMA 从 2026-11-24 21:55 起整体失效（capsule `err=-5`），target 重启也无法恢复；
- node1/node3 需交还他人。

用户决定：**迁移到 node2/node4 独占环境，目标不变**（vseg 读不慢于 plain，最终更快；300µs 目标保留）。**2026-11-26 用户补充真实部署形态：node2/3/4 可用；reader 在 node2；元数据 master 在 node4；两个远端 target（node3+node4）构成一个 vsegment（member_count=2）；plain 只用一个 target。验收改为 parity（vseg ≤ plain×1.05），2× 绝不可接受。**

第三阶段不是推倒重来：**保留第二阶段已经证实的测量事实，换一套干净、可运行的环境重新基线化**，然后继续定位 vseg 2.07×。

---

## 2. 当前拓扑（用户 2026-11-26 确认）

```
node2（reader + writer）                node4（master + etcd + target B）
┌────────────────────────┐             ┌──────────────────────────┐
│ stress_cluster_bench   │─元数据 RPC─►│ mooncake_master :50056   │
│ （supercache 构建）    │             │ metrics :9006            │
│                        │             │ HTTP metadata :8086      │
│                        │─数据 URMA─►│ nvmf_tgt :4457           │
│                        │             │ URMA_RAID0（11 块 7T）   │
│                        │             │ nsid1 → URMA_RAID0       │
│                        │             └──────────────────────────┘
│                        │
│                        │─数据 URMA─►  node3（target A）
│                        │             ┌──────────────────────────┐
│                        │             │ nvmf_tgt :4457           │
│                        │             │ URMA_RAID0（node3 盘）   │
│                        │             │ nsid1 → URMA_RAID0       │
└────────────────────────┘             └──────────────────────────┘
```

- **reader/writer = node2**
- **master + etcd = node4**
- **两个远端 target：node3（新，target A）+ node4（现有，target B）**
- **vsegment `member_count = 2`**：一个 4MiB key 切成 2×2MiB，两个 child 分别落到 node3/node4 两个 target，并行读。
- **plain 基线只用一个 target（node4）**，不分片；这是“单 target plain vs 双 target vseg”的架构对比。
- 对 reader 来说，**元数据（node4）和两个数据 target（node3/node4）都跨节点**，满足“两者都远端”。
- 验收：**parity（vseg P50/W9 ≤ plain × 1.05）**；2× 绝不可接受。

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

## 3. 版本与源码政策（硬规则）

| 组件 | 选择 | 说明 |
|---|---|---|
| Mooncake | `wangyuqi0429/Mooncake @ supercache-snapshot`，HEAD `aadab9573fe13ae6e355fcfce4520dcf67b1f385` | master 与 bench **必须从该分支重编**；不混 be10be29 |
| SPDK | **`e12b7cb` / `bd5ee124d` 谱系，wire v4** | 不升级 v6；不重编 SPDK |
| target 二进制 | 我们自编的 `nvmf_tgt`，md5 `3129641f…` | 从 node3 复制；static 链接；带 target 侧日志打点 |
| 客户端静态库 | `spdk-e12b7cb-src/build/lib`，`libspdk_bdev.a=4056b6bd…` | **禁止 `/usr/local/lib/libspdk_*.a`**（NOF-096 禁用集） |
| node3 旧 bench | `bench-nof097`（be10be29，md5 `6757c001…`） | 只允许 10-key 临时冒烟，**不得用于性能结论** |

- node2/node4 上现成的 GAP/nds 工作目录和构建**不使用**；只用我们自己的 `mooncake-nof-vsegment-2`。
- Mooncake 在 node2 的 `build/`（/home 余 595G）编译；node2 根分区仅 2.6G，不往根盘写大文件。

---

## 4. 第三阶段关键里程碑

| 任务 | 内容 | 结果 |
|---|---|---|
| **NOF-104** | node2/node4 只读盘点 | node4 11 块 7T 全空闲、BDF 与 node3 一致；node2 仅 6 块空闲盘；两台原非独占（用户确认旧进程可 kill）；node4 缺 meson/ninja；现有 SPDK 库均不可直接用 |
| **NOF-105** | 清理我方旧进程 + node4 target 部署 | target **PID 2142228**、md5 `3129641f…`、`:4457 LISTEN`、RAID0 online/11 base/BDF 升序/strip256K、nsid1、URMA/udmac0d1e2/RM、新 UUID `0470fe3a…`；hugepages 启动后 free 4476 页；未重编/未装包/未写数据 |
| **NOF-106** | supercache 编译 + node2 plain 环境 | supercache@aadab957 在 node2 编译 master/bench/nof_register，只链接自有 e12b7cb 库；etcd 3.4.14；plain master PID 34172（50056/9006/8086、Keys=400、owned_slots=1）；register node4 成功（0→100GB、心跳 0 失败）；400 keys/1.56GiB；10-key **10/10、P50 555.36µs**（node2 远端读，仅冒烟） |
| **NOF-107** | 新环境 plain vs vseg 400-key 基线 | **master+target 都在 node4、reader 在 node2**（元数据与数据均跨节点）；target 2142228 未重启。plain **P50 274.00µs / W9 120.07µs**；vseg **P50 555.00µs / W9 248.80µs** ⇒ **2.03× / 2.07×**；两臂每遍 target IO 数相同（n≈16.9k，32×128KiB/key）。P90/P99 比值 1.07–1.11× 是 plain 自身长尾（P90/P50=2.11）所致，不是追平 |
| **NOF-108** | vseg 2× 归因第一步 | `type` 补证 **vseg 528/528 / plain 528/528**；**95% 差值在 `completion_wait`（+206.5µs）**，客户端逻辑排除；W9 **248.03 vs 120.08µs（2.07×）**；plain 后跑不变 ⇒ 臂序排除。**plain 实测 key 0–95 三遍一致前缀慢档（~452µs vs 96+ ~250µs），vseg 全区平坦 ~470µs**；diag 开销不对称，真实比值仍用 diag-off 2.03× |
| **NOF-109** | vsegment 子任务时序（纯离线） | **1200/1200 op overlap 100%**；submit_skew 6.8µs、completion_gap 22.9µs；child0/child1 各 ~375/394µs；op span 420µs = **1.10× 单 child**、span/(lat0+lat1)=0.52 ⇒ **“串行两波”被否，“并行但每 child 自身慢”成立**。源码确认 `member_count=1` 时两 child 物理相邻 |
| **NOF-110** | 单变量 `MC_NOF_WORKERS=1` | **无改善**：vseg P50 555.00→558.50、W9 248.80→249.36；plain 274.00→266.27、W9 120.07→120.36。4 条 lane 全绑 worker0，两配置 `lane_inflight=16384`/`segment_inflight=65536` 相同 ⇒ **worker 分布、inflight 额度排除**。但 qpair 仍 4 条，连续 task 轮转 ⇒ vseg 每 key 2 条 qpair、plain 1 条，**qpair 扇出未排除** |
| **NOF-111** | 单变量 qpair 数 | **无改善**：vseg W9 248.80→**248.87**、P50 555.00→558.74；plain W9 120.29。探针确认 1 条 lane/qpair、单 lane 额度仍 16384 ⇒ **qpair 扇出排除**。客户端组织方式累计全部排除；W9 比值三次独立实验均为 **2.07×** |
| **NOF-112** | LBA 布局对照 | **布局假说被否**：两臂起始 LBA 逐值相同 400/400；非慢区 301 个 key 上 vseg **301/301 全慢**（p50 478.5 vs 179.0µs），慢区 99 个 key 两臂相等（447.5 vs 448.5）⇒ 位置不能解释 2× |
| **NOF-113R** | 纯配置：vseg 大 IO vs plain | **配置路径到顶**：vseg@2M **551.93** vs plain@128K **270.52µs（2.04×）**、vs plain@2M 277.73（1.99×）；chunk 对 vseg 无影响，W9 反而 250→287（plain 120→143）⇒ **2× 按字节计，不是每 IO 固定开销**；target 单 IO 在 ≥256K 时被夹到 256KiB（n 16898→8450）。parity 未达成 |
| **NOF-114** | node3 第二个 target 部署与就绪 | **已就绪、无需部署**：PID 2810360、md5 同 node4、11 盘 RAID0 strip256K、nsid1、4457、0 ERROR、空转 n=0；但 NQN/SN 与 node4 相同 |
| **NOF-114R** | node3 独占清理 + target A 换唯一 NQN | **完成**：royi master/client/构建循环已清；target A 重启为 PID 3329243、`urma-gpu-test-node3`/`URMAGPU0003`、UUID `2c5bf1f2…`、11/11、4457、0 ERROR、零 IO；node4 未动 |
| **NOF-115** | 双 target 就绪 + 2-key 冒烟 | **完成**：2 segment 注册成功；quota member_count=2/extent=50G/stripe=2M、logical_capacity=100G、reconcile_fail=0；写读通过，每 key 2 child（4096 blocks/16 sub-io），两 target 各约 66 IO 均分。注册不持久，需保持心跳 |
| **NOF-116（当前）** | 双 target 400-key 性能 | plain 400 keys 单 target 先测；vseg 400 keys 双 target（`--num_keys=200` 每 segment，总 400）；T1×3 遍、diag 关；验收 parity（vseg P50 ≤ plain×1.05）。每次 bench/重启后重注册 B→A |
| **NOF-115（待下发）** | 双 target vsegment 测量 | 参数：`member_count=2`、**extent=50GiB**、**keys=400（先跑通）**、oplog/HA 保持现状。register node3+node4 两个 NoF segment；写 400 keys（4MiB→2×2MiB 跨两个 target）；plain 单 target 400 keys 对照；验收 parity（≤ plain×1.05）。1000/20k 等跑通后再试 |

---

## 5. 当前要回答的问题

1. **已回答（NOF-107）**：在新环境 + supercache-snapshot 下，**vseg 仍慢约 2×**（P50 2.03×、target W9 2.07×）⇒ 旧环境/旧源码不是必要条件。
2. **已回答（NOF-108）**：`type` 确认走 vsegment；客户端 95% 差值在 `completion_wait`，resolve/submit 可忽略；臂序排除。**plain 有 key 0–95 前缀慢档，vseg 全区平坦 ~470µs ⇒ 2× 来自 plain 快区**。
3. **已回答（NOF-109）**：两个 child **100% 重叠**，span = 1.10× 单 child ⇒ 不是串行两波；**每个 child 自身慢**。源码确认两 child 物理相邻。
4. **已回答（NOF-110）**：`MC_NOF_WORKERS=1` 无改善 ⇒ worker 线程分布、inflight 额度排除；但 qpair 仍 4 条，**每 key 两个 qpair 扇出未排除**。
5. **已回答（NOF-111）**：qpairs=1 无改善（W9 248.87）⇒ qpair 扇出排除；客户端组织方式累计全部排除。
6. **已回答（NOF-112）**：两臂起始 LBA 逐值相同 400/400；非慢区 301 个 key 上 vseg 301/301 全慢（478.5 vs 179.0µs）⇒ **数据落点排除**。
7. **待裁决（NOF-113）**：只剩“请求切分边界（2MiB×2 vs 4MiB×1）”与“vseg 会话/服务路径”。A=改码合并相邻 child + env 开关；B=stripe=4MiB 新写数据。
8. **仍未隔离**：两臂是不同 cid/client 进程/NoF 会话；即使 LBA 相同，会话身份仍未控 ⇒ 路径归因仍为 L2+。
9. **要达成用户目标（vseg 必须快于 plain）**：若 A/B 显示切分边界是原因，合并 child 即可；若否，转 target 侧 trace（bdev/URMA 层）。

**注意**：NOF-106 的 10-key P50 555.36µs 是 node2 远端读、10 个 key 的冒烟，含冷启动/远端路径，不能当性能基线。正式数字只看 NOF-107/108 的 400-key 3 遍。

---

## 6. 第三阶段工作原则

- **一单一阶段**：先跑通环境，再做基线，再定位，再修复；不把构建、起服务、实测、分析塞进一单。
- **每 2–3 单强制复盘**：目标 → 当前 gap → 已排除 → 最小下一步 → 是否换层。连续两单没有决定性证据就停止当前方向。
- **事实三态**：`[事实]` / `[推算]` / `[未获取]`；不猜、不用默认值填充。
- **源码一致性**：Mooncake 只用 supercache-snapshot；SPDK 只用 wire v4；target/bench 不混版本。
- **数据保留不是目标**：允许覆写/新建 cid；但不碰他人进程、不改系统配置、不装包。
- **高危操作**：target 重启、节点重启、改内核参数、kill 他人进程，必须先经用户确认。
- **只用 node 标签**：不写内网 IP、账号、密码。

---

## 7. 已知风险

1. **master/target 同机**：node4 的 master/etcd 会与 target 抢 CPU/网络；任何 target W9 变化需记录这一混叠，不能直接与 node1/node3 数字比较。
2. **hugepages 17.1G**：target 启动后 free 4476 页；若后续并发/缓冲增大，可能不足，需提前核算。
3. **node2 根分区仅 2.6G**：大文件只放 /home；不要把日志/构建写根盘。
4. **源码/依赖离线**：三台集群机无外网，源码/子模块/gtest 需 Windows 中转；etcd Go wrapper 复用 node3 预构建库，版本差异需记录。
5. **RAID0 superblock=false**：11 盘顺序敏感；node4 已按 BDF 升序接回，后续任何重启都要重新核对 BDF。
6. **旧结论不可直接复用**：be10be29/node1 的数字与 supercache/node4 不可跨环境相减；第三阶段必须重新基线化。旧数字亦未经本侧复核，只作参照。
7. **盘首慢窗口未在 node2/node4 验证**：36MB/盘、RAID0 前 ~396MB、keys 0–96 都是 node1/node3 的观测，新环境的盘序/RAID/数据布局不同，不能预设。NOF-107 主表用全量 400 keys；只有新数据里出现多遍可重复的按 key index 前缀慢档时，才按实测边界追加排除列，禁止套用 0–96。

---

## 8. 产物与索引

- `../HANDOVER.md`：NOF-104 起的所有任务书与回执。
- `400keys-首次端到端测试结果.md`：第三阶段第一份 400-key 端到端结果（NOF-107/108）；含 plain/vseg 对照、客户端分段结构图、逐 key 快慢档分析、结论等级与边界。
- `条带与分块三层参数说明.md`：vsegment `stripe_size=2MiB`、NoF `chunk=128KiB`、target RAID0 `strip=256KiB` 三层的区别、公式、LBA 例子和常见误解。
- `../接手以来重要结果.md`：接手至 NOF-106 的有效/撤回结论汇总。
- `../第二阶段/README.md`：第二阶段历史（node1/node3），仅作参考，不再作为当前状态。
- node4：`/home/l00955908/mooncake-nof-vsegment-2/`（target 树、启动脚本、日志）。
- node2：`/home/l00955908/mooncake-nof-vsegment-2/`（supercache 源码/build、etcd、master、bench、日志）。