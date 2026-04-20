# Ubuntu 22.04 / 5.15.0-174-generic 内核兼容说明与构建指南

本文适用于以下场景：

* 节点操作系统为 Ubuntu 22.04
* 目标内核为 `5.15.0-174-generic`，或同系列 Ubuntu 22.04 `5.15` generic 内核
* vArmor 需要启用 BPF enforcer
* 当前主仓库仍使用 Ubuntu 24 修复版 `vArmor-ebpf` 子模块作为默认基线

## 1. 问题是什么

当前主仓库跟踪的 `vArmor-ebpf` 子模块已经包含 Ubuntu 24 内核兼容修复。这套修复对 Ubuntu 24 是必须的，但其中有一项变更不能直接原样套用到 Ubuntu 22.04 的 `5.15.0-174-generic` 内核上。

核心差异在于 `path_rename` LSM hook 原型：

* Ubuntu 24 当前基线的 `headers/vmlinux.h` 中，`path_rename` 带有额外的 `unsigned int flags` 参数
* 当前 `5.15.0-174-generic` 的内核 BTF 中，`path_rename` 仍然是旧原型，不带 `flags`

如果直接使用仓库默认的 `headers/vmlinux.h` 执行 `make build-ebpf`，最终生成的 BPF 对象可能会在 Ubuntu 22.04 / 5.15 节点上加载 `path_rename` hook 时失败，典型现象是：

* `field VarmorPathRename: program varmor_path_rename: load program: invalid argument`

需要特别说明的是，Ubuntu 24 兼容修复里的另外两类调整仍然是有价值的，不应该回退：

* `path_link` 和 `path_rename` 分离 tail-call prog array
* 将 `old_path_check()` 调整为 `__noinline`，降低 `move_mount` 相关 verifier 压力

因此，Ubuntu 22.04 / 5.15 的正确处理方式不是“把 Ubuntu 24 修复整包回退”，而是“保留通用修复，只用当前目标内核的 BTF 重新生成 `headers/vmlinux.h` 和后续 BPF 产物”。

## 2. 这次补了什么

从这次开始，`vArmor-ebpf/Makefile` 支持通过 `VMLINUX_BTF` 指定目标内核 BTF 文件：

```bash
make VMLINUX_BTF=/sys/kernel/btf/vmlinux build-ebpf
```

当传入这个变量时，构建流程会先执行：

1. 用 `bpftool btf dump file ... format c` 按目标节点 BTF 重建 `vArmor-ebpf/headers/vmlinux.h`
2. 再基于这份新的 `vmlinux.h` 重新编译 `bpfenforcer` 和 `processtracer`

这样做的结果是：

* `path_rename` 的本地编译期原型会与当前 `5.15.0-174-generic` 内核保持一致
* Ubuntu 24 中已经修好的 tail-call 拆分和 verifier 降压逻辑仍然保留
* 本地构建、方案三现场替换、以及后续 Docker 构建都可以复用同一套已重建的 BPF 产物

需要注意：如果你后续执行 Docker 构建，也要确保 build context 中的 `vArmor-ebpf/headers/vmlinux.h` 以及生成后的 `bpf_bpfel.o` / `bpf_bpfel.go` 已经是按目标内核重新生成过的版本。

## 3. 部署前检查

### 3.1 确认节点满足基础条件

在任意 Ubuntu 22.04 节点上执行：

```bash
uname -r
cat /sys/kernel/security/lsm
test -r /sys/kernel/btf/vmlinux && echo btf-ok
containerd --version
bpftool version
```

检查要点如下：

* 当前目标环境为 `5.15.0-174-generic`
* `/sys/kernel/security/lsm` 的输出中必须包含 `bpf`
* 节点必须存在可读取的 `/sys/kernel/btf/vmlinux`
* 容器运行时需要为 containerd `1.6.0` 及以上版本
* 重新生成 `headers/vmlinux.h` 时需要 `bpftool`

如果 `/sys/kernel/security/lsm` 中没有 `bpf`，需要先在节点启动参数中把 `bpf` 加入 `lsm=` 列表，再重启节点。修改时应保留当前节点已有的 LSM 顺序。

### 3.2 同步主仓库和子仓库

建议执行以下命令：

```bash
git submodule sync --recursive
git submodule update --init --recursive
git submodule status
git -C vArmor-ebpf rev-parse --short HEAD
```

这一步的目标不是切回旧版子模块，而是确认当前确实使用了包含 Ubuntu 24 通用修复的基线，再在此基础上按 5.15 的 BTF 重新生成本地产物。

### 3.3 安装本地构建依赖

如果你准备在宿主机上执行源码构建，建议先安装：

```bash
sudo apt install make golang libseccomp-dev libapparmor-dev clang llvm bpftool
```

其中：

* `bpftool` 用于从目标内核 BTF 生成 `headers/vmlinux.h`
* `clang` 和 `llvm` 用于重新编译 eBPF 产物

## 4. 构建方式

### 4.1 先按当前内核 BTF 重建 `headers/vmlinux.h`

在主仓库根目录执行：

```bash
make -C vArmor-ebpf generate-vmlinux VMLINUX_BTF=/sys/kernel/btf/vmlinux
grep -n '(*path_rename)' vArmor-ebpf/headers/vmlinux.h | head -n 1
```

在当前 `5.15.0-174-generic` 上，检查输出时应重点确认：

* `path_rename` 不再带 `unsigned int flags`

如果这一步仍然看到带 `flags` 的原型，说明你没有真正使用当前节点的 BTF 文件重新生成头文件。

### 4.2 基于当前 5.15 BTF 重新生成 eBPF 产物和本地二进制

继续执行：

```bash
make VMLINUX_BTF=/sys/kernel/btf/vmlinux build-ebpf copy-ebpf local
sha256sum bin/vArmor
```

这里的关键点是：

* `build-ebpf` 会自动把 `VMLINUX_BTF` 透传到 `vArmor-ebpf` 子模块
* 当该变量存在时，构建流程会先重建 `headers/vmlinux.h`，再重新生成 BPF 产物
* `copy-ebpf` 会把最新生成的 `bpf_bpfel.go` 和 `bpf_bpfel.o` 同步回主仓库
* `local` 最终产出适合当前 Ubuntu 22.04 / 5.15 节点验证的 `bin/vArmor`

### 4.3 如果你只想做现场验证

如果当前集群已经安装了 vArmor，而你只想先验证这版 `bin/vArmor` 能否在 Ubuntu 22.04 / 5.15 节点上成功加载 BPF LSM 程序，可以继续参考配套示例：

* [docs/guides/ubuntu_22_5_15_kernel_support_scheme3_example.zh_CN.md](ubuntu_22_5_15_kernel_support_scheme3_example.zh_CN.md)

这份示例采用与 Ubuntu 24 方案三相同的思路：

* 用新生成的 `bin/vArmor` 覆盖节点上的 `vArmor-patched`
* 重启 `varmor-agent`
* 用一个最小 Deployment 和一个最小 `VarmorPolicy` 验证 BPF profile 已经真正下发

## 5. 部署后验证

建议至少执行：

```bash
kubectl -n varmor rollout status ds/varmor-agent --timeout=150s
kubectl -n varmor get ds varmor-agent
kubectl -n varmor get pods -l app.kubernetes.io/component=varmor-agent -o wide
kubectl -n varmor logs -l app.kubernetes.io/component=varmor-agent --tail=200
```

通过标准如下：

1. `varmor-agent` DaemonSet 全部 Ready
2. 日志中可以看到 `attach VarmorPathRename to the LSM hook point`
3. 日志中可以看到 `attach VarmorMoveMount to the LSM hook point`
4. 日志中可以看到 `vArmor agent is online`
5. 不再出现 `path_rename` 的 `invalid argument` 错误

## 6. 常见坑位

### 6.1 只同步了子模块，没有按 5.15 BTF 重建 `vmlinux.h`

这是最常见的问题。当前子模块基线是为 Ubuntu 24 修复过的版本，但默认 `headers/vmlinux.h` 不等于当前 Ubuntu 22.04 / 5.15 节点的 BTF。

### 6.2 本地构建是对的，但 Docker 构建又把旧头文件重新带回来了

`cmd/varmor/Dockerfile` 会在镜像构建阶段执行 `make build-ebpf`。因此，如果 build context 中带进去的还是旧的 `headers/vmlinux.h`，镜像阶段仍然会重新编出错误的对象。

### 6.3 误以为要回退 Ubuntu 24 的全部修复

不需要。对 5.15 来说，真正需要按目标内核重建的是 `vmlinux.h` 所描述的 hook 原型；tail-call 拆分和 `old_path_check()` 的 `__noinline` 仍然建议保留。

## 7. 建议的升级顺序

建议按以下顺序推进：

1. 在目标节点上确认 `bpf` 已出现在 `/sys/kernel/security/lsm`
2. 确认 `/sys/kernel/btf/vmlinux` 可读
3. 同步主仓库和子模块到当前基线
4. 执行 `make VMLINUX_BTF=/sys/kernel/btf/vmlinux build-ebpf copy-ebpf local`
5. 用方案三或其它方式先做一次现场验证
6. 验证通过后，再决定是否将同一套已重建产物带入正式镜像流程