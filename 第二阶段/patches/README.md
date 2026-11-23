# 第二阶段代码补丁

所有补丁只用于**我们自己的构建副本**（`$D=/home/l00955908/mooncake-nof-vsegment/build/mooncake-src`），不改同事原树、不改原 bench/原库。

## nof050_client_io_params.patch（M1a，2026-11-23）

- **基线**：Mooncake `be10be2`（同事构建基线）
- **范围**：仅 `mooncake-store/src/transfer_task.cpp`，+56 行，**纯日志、默认关闭**
- **触发**：`MC_NOF_IO_PARAMS_LOG=1`
- **日志**：
  - `NOF_IO_PARAMS_SUBMIT ...`：每 task 提交时打印 `op / ptr / seg_lba / blocks / bytes / block_size / child / vseg_op_id / endpoint`
  - `NOF_IO_PARAMS_WORKER ...`：每 task 入队后打印 `op / ptr / seg_lba / blocks / lane / worker / buf_type / buf_device / domain`
- **用途**：核对 vseg 与 plain 的客户端参数是否一致（ptr 是否在 `[register_buffer]` 注册域内、child 的 seg_lba 是否差 2MiB、lane/buffer type/domain 是否相同），并给 target 侧插桩提供对照。

### 应用 / 构建 / 回退

```bash
D=/home/l00955908/mooncake-nof-vsegment/build
H=<handover 仓路径>/第二阶段/patches/nof050_client_io_params.patch
B=$D/mooncake-build-colleague
W=/tmp/nof050

# 0) 备份
cp $D/mooncake-src/mooncake-store/src/transfer_task.cpp /tmp/nof050/transfer_task.cpp.orig
md5sum $D/mooncake-src/mooncake-store/src/transfer_task.cpp

# 1) 应用（在我们副本内）
cd $D/mooncake-src && patch -p1 < $H

# 2) 编译（复用 NOF-048 的 compile.sh，确认 -c 指向 $D/mooncake-src）
bash /tmp/nof048/compile.sh        # 输出改为 /tmp/nof050/transfer_task.o

# 3) 换库成员
cp $B/mooncake-store/src/libmooncake_store.a $W/libmooncake_store_io_params.a
cp $W/transfer_task.o $W/obj/transfer_task.cpp.o
cd $W/obj && ar r $W/libmooncake_store_io_params.a transfer_task.cpp.o && ranlib $W/libmooncake_store_io_params.a

# 4) 重链（link.sh 改：库路径 → $W/libmooncake_store_io_params.a；输出 → $D/bench-io-params）
cd $B/mooncake-store/benchmarks && bash $W/link.sh

# 5) 校验
strings $D/bench-io-params | grep NOF_IO_PARAMS

# 回退
cd $D/mooncake-src && patch -R -p1 < $H
# 或恢复备份：cp /tmp/nof050/transfer_task.cpp.orig $D/mooncake-src/mooncake-store/src/transfer_task.cpp
```

### 注意

- 编译命令的 `-I` 指向同事头文件（与我们的源码同 md5）；本补丁只改 `.cpp`，不受影响。
- 链接仍用 `/usr/local/lib` 的静态 SPDK 库；本补丁不涉及 SPDK。
- 原 bench（`stress_cluster_bench`）与原 `libmooncake_store.a` 不被覆盖。