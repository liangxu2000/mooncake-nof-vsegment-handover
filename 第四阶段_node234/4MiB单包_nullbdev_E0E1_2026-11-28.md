# 4MiB 单包 + null bdev：E0 / E1 对照（2026-11-28 修正版）

> **重要更正**：NOF-152 最初报的“target 33µs”是 **E1 单侧值**；NOF-153 按 EID 分开后，**E0 target = 88.6µs、E1 = 33.4µs**。E1 省的 ~55µs **全在 target 侧 URMA completion push（push JFC wait）**，不是客户端。

## 结论

- **4MiB 单包成立**：400 keys 写 `write_ops=400`、`write_bytes=1,677,721,600=400×4MiB`；读每遍 `read_ops≈528`（400 keys + 128 warmup），**没有 12,800（32×400）**；客户端 `sub_ios=1 / blocks=8192`，target capsule splits=0。
- **E0/E1 端到端**：E0 P50 中位 **249.4µs**、E1 **203.5µs**（diag-on；NOF-152 diag-off 为 219.4/179.3）。
- **target 侧按 EID 分开**：E0 **88.6µs/IO**、E1 **33.4µs/IO**，差 **55.2µs**；其中 **push JFC wait 差 55.2µs = 100%**。
- **客户端**：query 两侧均 **109µs**；`wait − drain` 残差两侧均 **22–23µs**；客户端代码路径无差。
- 因此 E1 的收益在 **target/URMA completion push**，不是 Mooncake 客户端；是否可由 Mooncake 优化 ⇒ 否；E1 是部署/URMA 侧旋钮。

## 1. 目的

- 验证单 reader/单 target、单个 4MiB 包下 E0/E1 的差异层。
- 用 null bdev 去掉 SSD，用 `MC_NOF_SUBMIT_CHUNK_BYTES=4MiB` 让 4MiB key 成为一个 4MiB NoF IO。

## 2. 条件

| 项 | 值 |
|---|---|
| reader | node2（`stress_cluster_bench`，md5 bb4a07a3…） |
| master | node4 备用端口 plain master：50057/9007/8087，非 HA/非 oplog，cluster `plain_nof153`，**不写 etcd** |
| target | node4 target B（原 `nvmf_tgt` 不重启）；运行时 `Null153` bdev + null153 subsystem/listener |
| vseg 环境 | 原 vseg master/Keys=60000/enabled=1 全程未动 |
| 数据 | 400 keys × 4MiB（null bdev，无 SSD 写入） |
| 读取 | T1/batch1、warmup128、qpairs4、verify=false、**diag-on** |
| EID | bench 客户端 `SPDK_URMA_EID_INDEX=0/1`；target 保持 EID0 |
| chunk | `MC_NOF_SUBMIT_CHUNK_BYTES=4194304`、`SPDK_URMA_MAX_IO_SIZE=8388608` |
| 顺序 | E0→E1→E0→E1→E0→E1，各 3 遍 |

## 3. 4MiB 单包验证

| 判据 | 期望 | 实测 | 结论 |
|---|---|---|---|
| 写 `write_ops` | ~400 | **400** | ✅ |
| 写 `write_bytes` | 1,677,721,600 | **1,677,721,600** | ✅ 400×4MiB |
| 读每遍 `read_ops` 增量 | ~528 | **528/528/529/528/529/528** | ✅ 未切分 |
| 若未合并（128KiB） | ~12,800 | **未出现** | ✅ chunk 生效 |
| 客户端 `sub_ios / blocks` | 1 / 8192 | **1 / 8192** | ✅ 4MiB/512B |
| target capsule splits | 0 | **n=0** | ✅ |

## 4. E0 / E1 端到端（diag-on，400 keys）

| 指标 | E0（3 遍） | E1（3 遍） | E1/E0 |
|---|---|---|---|
| P50（中位） | **249.4**（249.39 / 248.93 / 266.55） | **203.5**（207.72 / 175.59 / 203.45） | **0.816（−18.4%）** |
| Avg | 255.9 | 196.7 | 0.769 |
| P90 | 264.2 | 206.5 | 0.782 |
| P99 | 275.1 | 215.9 | 0.785 |
| failed | 0 | 0 | — |
| read_ops/遍 | 528 / 528 / 529 | 528 / 528 / 529 | — |

- NOF-152 diag-off 为 E0 219.4 / E1 179.3（0.817）；两批方向一致。
- 400 keys 样本小，P99 仅供参考。

## 5. target 侧按 EID 分开（关键）

| 段（µs/IO） | E0（中位） | E1（中位） | E0−E1 |
|---|---|---|---|
| **TOTAL（parse→rsp）** | **88.6** | **33.4** | **+55.2** |
| **push JFC wait (C2H)** | **86.9** | **31.7** | **+55.2（100%）** |
| W9 exec→cpl（null） | 0.91 | 0.78 | +0.13 |
| W4a parse | 0.20 | 0.20 | ~0 |
| W7 post WR | 0.21 | 0.20 | ~0 |
| W10 send rsp | 0.32 | 0.33 | −0.01 |
| push 占比 | **97.9%** | **94.7%** | — |

- **E0/E1 的差 100% 在 push JFC wait**；其余段差 ≤0.14µs。
- 该表来自 target 的 `---- 新连接进入 ----` 后 final dump，按每遍连接对齐；target 块本身无时间戳/EID 标签，属**推断对齐**，已用 NOF-152 同窗口 5 块（89.1/33.9/89.3/33.9/89.2）交叉验证。
- W9 对 null bdev 无意义（空介质）。

## 6. 客户端按 EID 分开（µs）

| 项 | E0 | E1 | 差 |
|---|---|---|---|
| query | 109 | 109 | 0 |
| prepare | 3 | 2 | +1 |
| submit | 3 | 2 | +1 |
| completion_wait | 121 | 66 | +55 |
| read_unattributed | 8 | 6 | +2 |
| read_us | 133 | 77 | +56 |
| total_us | 244 | 189 | +55 |
| task total | 110 | 55 | +55 |
| task drain | 99 | 43 | +56 |
| task max_subio | 96 | 40 | +56 |
| poll_calls（全 empty） | 187 | 74 | +113 |
| wait − drain 残差 | 22 | 23 | −1 |

- **客户端 query 两侧一样；wait − drain 残差也一样**；客户端逻辑没有解释 55µs。
- 客户端 `completion_wait` 少 55µs，是因为 target push 快；`poll_calls` 187→74 是结果，不是原因。

## 7. 判读

- **E1 的 18% 差额在 target 侧 URMA completion push**，不是 Mooncake 客户端、不是 SSD、不是 W9。
- 机制未验证：E0 vs E1 的 URMA jetty/JFC/物理口映射、`num_jetty_per_ep=1`、`jfc_depth=4096` 等需要 URMA 层取证。
- 若确认是 UB 侧 EID/口差异，则 **“E1 更快”不是 Mooncake 能优化的**；E1 是部署/URMA 侧旋钮。
- 本案是 Mooncake NoF 端到端，不是 raw UB 带宽；单包有效带宽 E0 17.8 / E1 21.8 GiB/s（NOF-152 diag-off），高并发才是 46.8/69.4 GiB/s。

## 8. 偏差/边界

- target 统计块无时间戳、无 EID 标签；按 final dump + 连接对齐，属推断；已交叉验证。
- 400 keys 样本小；P99 波动；diag-on 绝对值比 diag-off 高约 12–14%，不混用。
- `qos_fifo_wait_us` 字段为 nan（plain 路径无 QoS 字段），不影响结论。
- node4 有他人 ql client（PID 2928050，00:13 起），未清理，影响未验证。
- 未写 SSD；vseg master/etcd/target 4457 未动；Null153/sub-ss/master 测试后已清理。

## 9. 下一步

- **NOF-154**：URMA completion push E0/E1 机制取证（只读）：`urma_admin show -d udmac0d1e2` 的 EID/端口、client/target EID 选择、jetty/JFC 映射、poll/JFC 行为；确认 55µs 是否 UB 侧 EID/口差异。
- 不改码、不重启 target/etcd；若必须改 target/URMA 配置或重启，停手回报。
- 后续 E0/E1 对比必须**target final dump 按 EID + 客户端按 EID**两表齐全后才出结论。