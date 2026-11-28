# 4MiB 单包 + null bdev：E0 / E1 对照（2026-11-28）

## 结论

- **4MiB 单包成立**：400 keys 写 `write_ops=400`、`write_bytes=1,677,721,600=400×4MiB`；读每遍 `read_ops≈528`（400 keys + 128 warmup），**没有出现 12,800（32×400）**。
- T1 单并发：**E0 P50 中位 219.4µs，E1 179.3µs，E1/E0 = 0.817（−18.27%）**；三遍同向。
- **target 每 IO 仅 33µs**（push JFC wait 31.5µs / 92.8%，W9 1.3µs）；扣掉 target 后，客户端/传输仍有 **~146–186µs**。
- 本案是 **Mooncake NoF 端到端**，**不是 raw UB 裸带宽**；E1 的 18% 差额主要在客户端/传输路径。
- 按 4MiB ÷ P50 折算的端到端有效带宽：**E0 17.8 GiB/s、E1 21.8 GiB/s**；高并发 Mooncake 吞吐才是 plain E0 46.8 / E1 69.4 GiB/s。

## 1. 目的

- 验证“单 reader、单 target、单个 4MiB 包”下，E0/E1 的端到端时延是否有差别。
- 用 **null bdev** 去掉 SSD 介质，用 `MC_NOF_SUBMIT_CHUNK_BYTES=4MiB` 让 4MiB key 成为一个 4MiB NoF IO，从而把 UB/客户端路径单独暴露出来。

## 2. 条件

| 项 | 值 |
|---|---|
| reader | node2（`stress_cluster_bench`，md5 bb4a07a3…） |
| master | node4 备用端口独立 plain master：50057 / 9007 / 8087，非 HA / 非 oplog，cluster `plain_nof152`，**不写 etcd** |
| target | node4 target B（原 `nvmf_tgt` 不重启）；运行时创建 `Null152` bdev + null152 subsystem/listener |
| vseg 环境 | 原 vseg master 2828887/2828890、Keys=60000、enabled=1 全程未动 |
| 数据 | 400 keys × 4MiB（null bdev，无 SSD 写入），Keys=400 |
| 读取 | T1/batch1、warmup128、qpairs4、verify=false、diag 关 |
| EID | 只改 bench 客户端 `SPDK_URMA_EID_INDEX=0/1`；target 保持 EID0 |
| chunk | `MC_NOF_SUBMIT_CHUNK_BYTES=4194304`、`SPDK_URMA_MAX_IO_SIZE=8388608` |
| 顺序 | E0→E1→E0→E1→E0→E1，各 3 遍 |

> 备用端口独立 master 是 NOF-152 的变通：不碰 vseg master/etcd，仅用于 plain 诊断；已确认 vseg 数据与 etcd 未变。

## 3. 4MiB 单包验证

| 判据 | 期望 | 实测 | 结论 |
|---|---|---|---|
| 写：target `write_ops` | ~400 | **400** | ✅ 单包 |
| 写：`write_bytes` | 1,677,721,600 | **1,677,721,600** | ✅ 400×4MiB |
| 读：每遍 `read_ops` 增量 | ~528（400+128 warmup） | **528/528/529/528/529/528** | ✅ 未切分 |
| 读：`read_bytes` 增量 | 2,214,592,512 | **2,214,592,512**（两遍 529 ops 为 2,214,593,024，+512B） | ✅ |
| 若未合并（默认 128KiB） | ~12,800 | **未出现** | ✅ chunk 生效 |
| target capsule splits | 0 | **n=0** | ✅ |

- `Null152`：block=512、3,276,800 blocks；运行时创建，**未重启 target**。
- iostat 2 遍 +1 op / +512B 属探测/扫描噪声，不影响“未切分”结论。

## 4. E0 / E1 结果（µs，T1/batch1，400 keys）

| 指标 | E0（3 遍） | E1（3 遍） | E1/E0 |
|---|---|---|---|
| P50（中位） | **219.4**（236.82 / 218.50 / 219.41） | **179.3**（179.32 / 167.40 / 184.75） | **0.817（−18.27%）** |
| Avg | 222.2 | 180.6 | 0.813 |
| P90 | 227.1 | 184.9 | 0.814 |
| P99 | 275.1 | 204.9 | 0.745 |
| failed | 0 | 0 | — |
| read_ops/遍 | 529 / 528 / 529 | 528 / 528 / 528 | — |

- 三遍各自同向，无交叉。
- 400 keys 样本小，P99 波动明显（E0 P99 280.72 / 275.06 / 238.06），**P50 可用，P99 仅供参考**。
- 4MiB ÷ P50 的端到端有效带宽：**E0 17.8 GiB/s、E1 21.8 GiB/s**（含客户端/NoF/传输，不是 UB 裸带宽）。

## 5. target 侧分解（null bdev，单遍，**未标 EID**）

> **注意**：这张表来自 NOF-152 的某一遍 target 统计，回执没有记录这遍是 E0 还是 E1；**不能归因到任一臂**。E0/E1 分开的 target JFC/W9/TOTAL 由 NOF-153 采集。

| 段 | 数值 | 占比 |
|---|---|---|
| TOTAL（parse→rsp） | **33µs** | 100% |
| push JFC wait (C2H) | **31.5µs** | **92.8–93.1%** |
| W9 exec→cpl（null bdev） | 1.3–1.4µs | 4.2% |
| W10 send rsp | 0.34µs | 1.0% |
| W7 post WR | 0.23µs | 0.7% |
| W4a parse / W4b iobuf / W8 pull | 0.18 / 0.02 / 5.85（n=1）µs | 0.6 / 0.1 / ~0 |

- **该单遍 target 只占 33µs**；即使全扣掉，客户端仍有 ~146–186µs。但 E0/E1 的 target 是否相同尚未分采，NOF-153 确认。
- **W9 对 null bdev 无意义**（1.3µs 是空介质），不能当 SSD 时延。
- JFC 数值来自 target 周期统计块（n≈535–537），是块内平均，不是逐 IO。

## 6. 判读

- **target 不是瓶颈**：每 IO 33µs，其中 92.8% 是 push JFC wait，真正 bdev 执行 1.3µs。
- **E1 的 18% 差额在客户端/传输**，不是 SSD、也不是 target 执行。
- 本案是 **Mooncake NoF 端到端**；不能直接把它当作“UB 带宽 E1 高 18%”。要测 raw UB 带宽，需要 raw URMA/SPDK 4MiB 单包实验。
- 同事说的 47–49 GiB/s 是**高并发打满**的口径；单包有效带宽 17.8/21.8 GiB/s 受单包时延限制，不是链路容量。

## 7. 偏差/边界

- 400 keys，样本小；P99 仅参考。
- W9 对 null bdev 无意义；以 target TOTAL 33µs / push JFC 31.5µs 为准。
- iostat 的 +1 op / +512B 是探测噪声；JFC 是周期块平均；非逐 IO。
- node4 有他人 ql client（PID 2928050，00:13 起，监听 18076/8976），本单未清理，影响 [未验证]。
- 临时 plain master（50057/9007/8087）非 HA/非 oplog、不写 etcd；vseg master/etcd/target 4457 全程未动；测试后 Null152/sub-ss/master 已清理。
- 未写 SSD；原 URMA_RAID0 通道完好。

## 8. 下一步

- **NOF-153**：继续用 null bdev（无 SSD），diag-on 跑 E0/E1 各 3 遍，拆 query / prepare / submit / completion_wait / drain / max_subio + target JFC/W9/TOTAL，定位 **146–186µs** 的客户端构成和 E1 省下的 **~40µs**。
- push JFC wait 31.5µs（92.8%）先并入 NOF-153 的 E0/E1 target 对照；若 E0/E1 无差别，不单独立项。