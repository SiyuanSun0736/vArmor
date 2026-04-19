# Ubuntu 24.04 内核兼容说明与部署指南

本文基于以下两个提交整理：

* 主仓库：`f7dd53e Support Ubuntu 24 kernel`
* 子仓库：`00860c5 Support Ubuntu 24 kernel`

本文适用于以下场景：

* 节点操作系统为 Ubuntu 24.04
* 目标内核为 `6.8.0-110-generic`，或需要对同系列 Ubuntu 24 内核做兼容性验证
* vArmor 需要启用 BPF enforcer

## 1. 这次改了什么

### 主仓库改动

主仓库提交的职责主要是“把兼容性修复带进 vArmor 的构建和发布链路”。

| 位置 | 改动 | 作用 |
| --- | --- | --- |
| `.gitmodules` | 将 `vArmor-ebpf` 子模块地址从 `bytedance/vArmor-ebpf` 切换到 `SiyuanSun0736/vArmor-ebpf` | 让主仓库能够拉到包含 Ubuntu 24 修复的 eBPF 源码 |
| `vArmor-ebpf` gitlink | 子模块指针从 `9562760` 升级到 `00860c5` | 锁定到真正包含 Ubuntu 24 修复的子模块版本 |
| `pkg/lsm/bpfenforcer/bpf_bpfel.go` | 同步更新生成后的 Go 绑定代码 | 让主仓库中的 BPF map/program 定义与子模块保持一致 |
| `pkg/lsm/bpfenforcer/bpf_bpfel.o` | 同步更新预编译 BPF 对象 | 让主仓库直接携带可在 Ubuntu 24 上工作的 BPF enforcer 对象 |
| `pkg/processtracer/bpf_bpfel.o` | 同步刷新生成产物 | 保持主仓库与子模块生成结果一致 |
| `README.md` / `README.zh_CN.md` | 新增 Ubuntu 24 内核兼容说明 | 明确兼容范围和修复点 |

### 子仓库改动

子仓库提交真正解决的是“Ubuntu 24 内核下 BPF LSM 程序加载失败”的根因。

| 位置 | 改动 | 作用 |
| --- | --- | --- |
| `headers/vmlinux.h` | 更新 `path_rename` LSM hook 原型，补上 `unsigned int flags` 参数 | 对齐 Ubuntu 24 内核中的 hook 签名 |
| `pkg/bpfenforcer/bpf/enforcer.c` | 将 `path_link` 和 `path_rename` 分别使用独立的 tail-call prog array | 避免不同 LSM hook 的 tail program 混用，保证程序可加载 |
| `pkg/bpfenforcer/bpf/enforcer.h` | 将 `old_path_check()` 从 `__always_inline` 改为 `__noinline` | 降低 `move_mount` 相关路径检查的 verifier 压力 |
| `pkg/bpfenforcer/bpf_bpfel.go` / `pkg/bpfenforcer/bpf_bpfel.o` | 重新生成 BPF 绑定代码与对象文件 | 把上述修复反映到最终产物 |
| `pkg/processtracer/bpf_bpfel.o` | 同步刷新生成产物 | 保持整体生成结果一致 |

## 2. 问题是什么

在 Ubuntu 24.04 的 `6.8.0-110-generic` 内核上，旧版本的 BPF enforcer 会在加载阶段遇到三类兼容性问题：

### 2.1 `path_rename` hook 签名不匹配

Ubuntu 24 内核中的 `path_rename` LSM hook 带有额外的 `flags` 参数。旧版本 eBPF 程序按照旧原型编译，导致程序原型与目标内核不一致。

结果是：BPF 程序在加载或 attach `path_rename` hook 时可能直接失败。

### 2.2 `path_link` 与 `path_rename` 共用一个 tail-call prog array

旧实现把 `path_link_tail` 和 `path_rename_tail` 放在同一个 prog array 里。Ubuntu 24 对这类 LSM tail call 的约束更严格，要求同一个 program array 只承载同一 hook 语义下的 tail program。

结果是：即使主程序能通过 verifier，tail-call 相关 map 也可能让对象整体加载失败。

### 2.3 `move_mount` 路径检查的 verifier 压力过高

`move_mount` 相关路径检查会带来更高的 verifier 压力。旧实现把 `old_path_check()` 强制内联，在 Ubuntu 24 上更容易触发 verifier 复杂度问题。

结果是：对象在 `LoadAndAssign` 阶段可能因为 verifier 拒绝而失败。

### 2.4 为什么主仓库也必须同时升级

这次兼容性修复不是“只更新一下子模块指针”就结束。主仓库还必须把新的生成产物一并带上，原因有两个：

1. `make build` 会执行 `build-ebpf` 和 `copy-ebpf`，直接从子模块重新生成并覆盖主仓库中的 BPF 产物。
2. `cmd/varmor/Dockerfile` 在镜像构建阶段也会执行 `make build-ebpf`，同样直接依赖子模块当前 checkout 的源码。

如果主仓库已经更新，但本地 `vArmor-ebpf` 仍停留在旧提交，那么重新执行 `make build` 或 Docker 构建时，仍然会重新生成旧的、不兼容 Ubuntu 24 的 BPF 对象。

## 3. 部署前检查

### 3.1 确认节点满足基础条件

在任意 Ubuntu 24 节点上执行：

```bash
uname -r
cat /sys/kernel/security/lsm
containerd --version
```

检查要点如下：

* 内核版本至少为 5.10，本次目标环境为 `6.8.0-110-generic`
* `/sys/kernel/security/lsm` 的输出中必须包含 `bpf`
* 容器运行时需要为 containerd 1.6.0 及以上版本

如果 `/sys/kernel/security/lsm` 中没有 `bpf`，需要先在节点启动参数中把 `bpf` 加入 `lsm=` 列表，再重启节点。修改时应保留当前节点已有的 LSM 顺序，不要用只包含 `bpf` 的 `lsm=` 覆盖原有配置。

### 3.2 同步主仓库和子仓库到正确版本

无论是全新 clone，还是已有 checkout 升级到这个提交，都建议执行以下命令：

```bash
git submodule sync --recursive
git submodule update --init --recursive
git submodule status
git -C vArmor-ebpf rev-parse --short HEAD
```

检查结果中应看到 `vArmor-ebpf` 停留在 `00860c5` 或更新的兼容提交上。

这一步非常关键，因为 `.gitmodules` 已经切换了子模块 URL；如果本地没有执行 `git submodule sync --recursive`，Git 仍可能继续使用旧的子模块远端地址。

### 3.3 安装本地构建依赖

如果你准备按源码方式执行 `make build`、本地运行 agent，或者在宿主机上直接完成构建，建议先安装以下依赖：

```bash
sudo apt install make golang libseccomp-dev libapparmor-dev clang llvm
```

其中：

* `make` 和 `golang` 用于执行主仓库的构建流程
* `libseccomp-dev` 和 `libapparmor-dev` 用于编译 vArmor 依赖的安全组件
* `clang` 和 `llvm` 用于生成 eBPF 相关产物

## 4. 部署方式

## 4.3 方式三：不重建正式镜像，直接替换 agent 二进制做现场验证

如果集群里已经安装了 vArmor，但你当前不方便重建并推送正式镜像，也可以先用“节点 hostPath 覆盖二进制”的方式做一次现场验证。

这个方式适合以下场景：

* 需要尽快确认 Ubuntu 24 内核兼容修复是否真的能让 `varmor-agent` 正常加载 BPF LSM 程序
* 本地没有可用的镜像构建器，或者暂时不想修改正式镜像 tag
* 集群已经存在 `varmor-agent` DaemonSet

需要注意：这种方式更适合验证，不建议长期作为正式发布手段。验证通过后，仍建议按 4.2 节中的标准镜像流程完成正式升级。

### 步骤 1：在本地重新生成 agent 二进制

```bash
make build-ebpf copy-ebpf local
sha256sum bin/vArmor
```

建议记录 `sha256sum` 的输出，后面可以用它和节点上的目标文件做比对，确认每个节点上实际生效的二进制就是刚刚编出来的这一版。

### 步骤 2：准备一个临时 DaemonSet，把节点上的目标目录挂出来

如果当前 `varmor-agent` 已经通过 `hostPath` 挂载 `/var/run/varmor/bin/vArmor-patched` 到容器内 `/varmor/vArmor`，那么可以直接复用这个路径。

如果没有现成的挂载，可以先创建一个临时 DaemonSet，在每个节点上把 `/var/run/varmor/bin` 暴露出来：

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: varmor-bin-loader
  namespace: varmor
spec:
  selector:
    matchLabels:
      app: varmor-bin-loader
  template:
    metadata:
      labels:
        app: varmor-bin-loader
    spec:
      tolerations:
        - operator: Exists
      containers:
        - name: loader
          image: <当前集群正在使用的 varmor 镜像>
          imagePullPolicy: IfNotPresent
          command: ["/bin/sh", "-c", "mkdir -p /hostbin && sleep 36000"]
          securityContext:
            runAsUser: 0
          volumeMounts:
            - name: hostbin
              mountPath: /hostbin
      volumes:
        - name: hostbin
          hostPath:
            path: /var/run/varmor/bin
            type: DirectoryOrCreate
```

应用后，可以通过下面的命令确认每个节点上都已经有一个 `loader` Pod：

```bash
kubectl -n varmor get pods -l app=varmor-bin-loader -o wide
```

### 步骤 3：把新二进制分别复制到每个节点

假设上一步得到两个 Pod，分别位于 `ubuntu24` 和 `worker` 节点，可以直接执行：

```bash
kubectl -n varmor cp ./bin/vArmor varmor-bin-loader-<pod-on-ubuntu24>:/hostbin/vArmor-patched -c loader
kubectl -n varmor cp ./bin/vArmor varmor-bin-loader-<pod-on-worker>:/hostbin/vArmor-patched -c loader

kubectl -n varmor exec varmor-bin-loader-<pod-on-ubuntu24> -c loader -- sha256sum /hostbin/vArmor-patched
kubectl -n varmor exec varmor-bin-loader-<pod-on-worker> -c loader -- sha256sum /hostbin/vArmor-patched
```

检查要点如下：

* 节点上的 `sha256sum` 应与本地 `bin/vArmor` 完全一致
* 如果某个节点校验和不同，不要继续重启 agent，而是先重新复制

### 步骤 4：重启 agent，让新二进制生效

```bash
kubectl -n varmor delete pod -l app.kubernetes.io/component=varmor-agent
kubectl -n varmor rollout status ds/varmor-agent --timeout=150s
```

这一步的关键点是：只复制文件还不够，必须让 `varmor-agent` Pod 重建，新的容器进程才会重新打开并执行新的 `/varmor/vArmor` 文件。

### 步骤 5：验证 agent 已经恢复正常

```bash
kubectl -n varmor get ds varmor-agent
kubectl -n varmor get pods -l app.kubernetes.io/component=varmor-agent -o wide
kubectl -n varmor logs -l app.kubernetes.io/component=varmor-agent --tail=200
```

如果这次 Ubuntu 24 修复已经真正生效，应当看到以下现象：

* `varmor-agent` DaemonSet 为全部 Ready，例如 `2/2`
* 每个节点上的 agent Pod 都处于 `Running` 且 `READY 1/1`
* 日志中可以看到 `attach VarmorPathRename to the LSM hook point`
* 日志中可以看到 `attach VarmorMoveMount to the LSM hook point`
* 日志中可以看到 `vArmor agent is online`
* 日志中可以看到已有策略被重新下发，例如 `saving and applying the BPF profile`

同时，不应再看到以下错误：

* `field VarmorPathRename: program varmor_path_rename: load program: invalid argument`
* `field VarmorMoveMount: program varmor_move_mount: load program: argument list too long: BPF program is too large`
* 因临时 verifier 调试日志过重而导致的 `OOMKilled`

### 步骤 6：清理临时分发 DaemonSet

如果你创建了 `varmor-bin-loader`，验证完成后建议删除：

```bash
kubectl -n varmor delete ds varmor-bin-loader
```

这样可以避免集群中长期保留一个只用于复制二进制的临时常驻组件。

## 5. 部署后验证

### 5.1 检查组件状态

```bash
kubectl -n varmor get pods -o wide
kubectl -n varmor get daemonset varmor-agent
kubectl -n varmor get deployment varmor-manager
```

重点确认：

* `varmor-agent` 在 Ubuntu 24 节点上全部 Ready
* `varmor-manager` 已正常启动
* 如果启用了 `behaviorModeling`，再额外确认 `varmor-classifier` 状态

### 5.2 检查 agent 日志

```bash
kubectl -n varmor logs daemonset/varmor-agent -c agent --tail=200
```

重点观察以下几类信息：

* 不应再出现 “the BPF LSM is not supported”
* 不应再出现与 `LoadAndAssign`、`AttachLSM`、`path_rename`、`move_mount` 相关的初始化失败
* agent 应能完成 BPF 初始化，并持续保持 Ready

### 5.3 做一次最小策略验证

如果条件允许，建议再下发一个最小的 BPF 策略到测试工作负载上，确认 Ubuntu 24 节点上的 BPF enforcer 不只是“启动成功”，而是能够真正对目标容器施加规则。

### 5.4 本次 Ubuntu 24 现场验证的通过标准

如果你采用了 4.3 节中的“直接替换 agent 二进制”方式，可以按下面这组最小检查项判断本次部署是否通过：

```bash
kubectl -n varmor rollout status ds/varmor-agent --timeout=150s
kubectl -n varmor get ds varmor-agent
kubectl -n varmor get pods -l app.kubernetes.io/component=varmor-agent -o wide
kubectl -n varmor logs -l app.kubernetes.io/component=varmor-agent --tail=200
```

通过标准如下：

1. `kubectl rollout status` 返回成功，没有超时。
2. `varmor-agent` DaemonSet 显示 `DESIRED`、`CURRENT`、`READY`、`AVAILABLE` 全部一致。
3. 日志中不再出现 `path_rename` 的 `invalid argument` 错误。
4. 日志中不再出现 `move_mount` 的 “BPF program is too large” 错误。
5. 日志中能够看到 `attach VarmorPathRename`、`attach VarmorMoveMount`、`vArmor agent is online`。
6. 如果集群中已经存在测试策略，日志中应能看到 profile 下发和目标容器匹配信息，例如 `saving and applying the BPF profile`、`target container was created`。

## 6. 常见坑位

### 6.1 只拉了主仓库，没有同步子模块

这是最常见的问题。主仓库已经指向新提交，但本地 `vArmor-ebpf` 仍然停留在旧版本。随后执行 `make build` 或 Docker 构建时，会重新编出旧的 BPF 对象。

解决方法：重新执行 `git submodule sync --recursive` 和 `git submodule update --init --recursive`。

### 6.2 节点内核满足要求，但没有启用 BPF LSM

vArmor agent 在判断 BPF 支持时，不只检查最低内核版本，还会读取 `/sys/kernel/security/lsm`，确认其中包含 `bpf`。内核版本正确并不代表 BPF LSM 已启用。

### 6.3 用目录直接安装 chart，结果镜像 tag 不对

如果直接对 `manifests/varmor` 执行 `helm upgrade --install`，默认 `appVersion` 仍是占位值。这样很容易把旧镜像或不存在的 tag 部署进集群。

### 6.4 只更新了 README，没有更新运行时产物

这次兼容性修复最终是否生效，取决于运行中的 BPF 对象是否来自新子模块。README 里的说明、`.gitmodules` 的 URL 更新和真正的运行时产物，三者缺一不可。

## 7. 建议的升级顺序

建议按以下顺序推进：

1. 在 Ubuntu 24 节点上确认 `bpf` 已出现在 `/sys/kernel/security/lsm`
2. 同步主仓库和子模块到包含 `Support Ubuntu 24 kernel` 的版本
3. 先执行源码级本地验证，确认 agent 能成功加载 BPF LSM 程序
4. 再构建镜像并使用 Helm 升级集群
5. 升级后检查 `varmor-agent` 日志，并做一次最小策略验证