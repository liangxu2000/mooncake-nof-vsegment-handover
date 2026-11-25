# 第三阶段：node2/node4 重建环境与 vseg 回归定位

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

用户决定：**迁移到 node2/node4 独占环境，目标不变**（vseg 读不慢于 plain，最终更快；300µs 目标保留）。

第三阶段不是推倒重来：**保留第二阶段已经证实的测量事实，换一套干净、可运行的环境重新基线化**，然后继续定位 vseg 2.07×。

---

## 2. 当前拓扑（用户 2026-11-25 指定）

```
node2（reader + writer）                     node4（master + etcd + target）
┌────────────────────────┐                  ┌──────────────────────────┐
│ stress_cluster_bench   │──元数据 RPC─────►│ mooncake_master :50056   │
│ （supercache 构建）    │                  │ metrics :9006            │
│                        │                  │ HTTP metadata :8086      │
│                        │──数据 URMA──────►│ nvmf_tgt :4457           │
│                        │                  │ URMA_RAID0（11 块 7T）   │
│                        │                  │ nsid1 → URMA_RAID0       │
└────────────────────────┘                  └──────────────────────────┘
```

- **reader/writer = node2**
- **master + etcd + target = node4**
- 对 reader 来说，**元数据（master）和数据（target）都跨节点**，满足“两者都远端”的要求。
- master 与 target 同机是本拓扑的代价：master/etcd 会占用 node4 部分 CPU/内存；解析性能时必须记录这一混叠。

### 端口

| 组件 | 节点 | 端口 |
|---|---|---|
| SPDK nvmf target | node4 | **4457** |
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
| **NOF-107（当前）** | 新环境 plain vs vseg 400-key 基线 | **master+target 都迁到 node4，reader 放 node2**；plain 400 keys 3 遍 → vseg_nof107 写 400 读 3 遍；target 不重启、不修代码 |

---

## 5. 当前要回答的问题

1. 在新 node2/node4 环境、supercache-snapshot 源码下，**vseg 是否仍约 2× 慢于 plain**？
2. 如果仍慢，差距是否仍在 `read_us`/target 执行段？
3. 如果不再慢，是旧环境（node1 host/旧 RAID 顺序）还是旧源码（be10be29）造成的？——这将直接决定后续是否还需要修代码。

**注意**：NOF-106 的 10-key P50 555.36µs 是 node2 远端读、10 个 key 的冒烟，含冷启动/远端路径，不能当性能基线。正式数字只看 NOF-107 的 400-key 3 遍。

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
6. **旧结论不可直接复用**：be10be29/node1 的数字与 supercache/node4 不可跨环境相减；第三阶段必须重新基线化。

---

## 8. 产物与索引

- `../HANDOVER.md`：NOF-104 起的所有任务书与回执。
- `../接手以来重要结果.md`：接手至 NOF-106 的有效/撤回结论汇总。
- `../第二阶段/README.md`：第二阶段历史（node1/node3），仅作参考，不再作为当前状态。
- node4：`/home/l00955908/mooncake-nof-vsegment-2/`（target 树、启动脚本、日志）。
- node2：`/home/l00955908/mooncake-nof-vsegment-2/`（supercache 源码/build、etcd、master、bench、日志）。