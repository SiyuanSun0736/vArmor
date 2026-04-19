# Ubuntu 24.04 方案三实操示例：用一个 Deployment 和一个 VarmorPolicy 验证 BPF Enforcer

本文是 [docs/guides/ubuntu_24_kernel_support.zh_CN.md](docs/guides/ubuntu_24_kernel_support.zh_CN.md) 中“4.3 方式三：不重建正式镜像，直接替换 agent 二进制做现场验证”的配套实例。

本文不再重复解释 Ubuntu 24 内核兼容问题本身，而是直接给出一套可以照着执行的示例流程。目标是：

* 用新的 `bin/vArmor` 覆盖每个节点上的 `vArmor-patched`
* 重启 `varmor-agent`，确认 Ubuntu 24 节点上的 BPF LSM 程序已经可以正常加载
* 部署一个测试工作负载
* 下发一个最小的 BPF 策略
* 验证策略已经被 agent 正常应用到目标容器

## 1. 示例场景

假设当前环境满足以下条件：

* Kubernetes 集群中已经安装了 vArmor，命名空间为 `varmor`
* 节点操作系统为 Ubuntu 24.04，内核为 `6.8.0-110-generic`
* 节点已经启用 BPF LSM
* `varmor-agent` 已经支持通过 `hostPath` 从 `/var/run/varmor/bin/vArmor-patched` 启动 `/varmor/vArmor`

如果你的环境还不满足这些前提，请先参考 [docs/guides/ubuntu_24_kernel_support.zh_CN.md](docs/guides/ubuntu_24_kernel_support.zh_CN.md) 完成准备工作。

## 2. 本文使用的两个 YAML

本文使用下面这两个对象做验证：

* 一个 Deployment：部署两个副本，每个 Pod 里有两个 `ubuntu:22.04` 容器
* 一个 VarmorPolicy：对这个 Deployment 开启 BPF EnhanceProtect，并启用 `disallow-write-core-pattern`

### 2.1 Deployment 示例

保存为 `demo-deploy-sleep.yaml`：

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: varmor-demo-sleep
  namespace: test1
  labels:
    app: varmor-demo-sleep
    sandbox.varmor.org/enable: "true"
spec:
  replicas: 2
  selector:
    matchLabels:
      app: varmor-demo-sleep
  template:
    metadata:
      labels:
        app: varmor-demo-sleep
        sandbox.varmor.org/enable: "true"
    spec:
      containers:
      - name: sleeper-1
        image: ubuntu:22.04
        command: ["bash", "-c", "sleep infinity"]
      - name: sleeper-2
        image: ubuntu:22.04
        command: ["bash", "-c", "sleep infinity"]
```

### 2.2 VarmorPolicy 示例

保存为 `test.yaml`：

```yaml
apiVersion: crd.varmor.org/v1beta1
kind: VarmorPolicy
metadata:
  name: demo-policy
  namespace: test1
spec:
  updateExistingWorkloads: true
  target:
    kind: Deployment
    name: varmor-demo-sleep
  policy:
    enforcer: BPF
    mode: EnhanceProtect
    enhanceProtect:
      auditViolations: true
      allowViolations: false
      privileged: false
      hardeningRules:
        - disallow-write-core-pattern
```

这个策略的含义是：

* 目标工作负载是 `test1` 命名空间下的 `Deployment/varmor-demo-sleep`
* 使用 `BPF` enforcer
* 运行模式是 `EnhanceProtect`
* 对命中的容器启用 `disallow-write-core-pattern`
* 发现违规时记录审计，不放行违规操作

## 3. 第一步：重新生成本地 agent 二进制

在 vArmor 主仓库目录执行：

```bash
cd /home/ssy/vArmor

make build-ebpf copy-ebpf local
sha256sum bin/vArmor
```

建议把输出的哈希保存到一个变量里，后面用来校验节点上的目标文件：

```bash
NEW_SHA=$(sha256sum bin/vArmor | awk '{print $1}')
echo "$NEW_SHA"
```

如果这里没有生成新的 `bin/vArmor`，后面的步骤都不应继续。

## 4. 第二步：创建一个临时 loader DaemonSet

这个 DaemonSet 的唯一目的，是在每个节点上把 `/var/run/varmor/bin` 挂出来，方便把新二进制复制进去。

```bash
cat <<'EOF' | kubectl apply -f -
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
          image: elkeid-ap-southeast-1.cr.volces.com/varmor/varmor:v0.10.0
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
EOF
```

等待 Pod 起好：

```bash
kubectl -n varmor get pods -l app=varmor-bin-loader -o wide
```

如果你有两个节点，例如 `ubuntu24` 和 `worker`，输出里应当各有一个 `loader` Pod。

为了后续命令更容易执行，可以把两个 Pod 名记录下来：

```bash
UBUNTU24_LOADER=$(kubectl -n varmor get pods -l app=varmor-bin-loader -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.spec.nodeName}{"\n"}{end}' | awk '$2=="ubuntu24" {print $1}')
WORKER_LOADER=$(kubectl -n varmor get pods -l app=varmor-bin-loader -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.spec.nodeName}{"\n"}{end}' | awk '$2=="worker" {print $1}')

echo "$UBUNTU24_LOADER"
echo "$WORKER_LOADER"
```

## 5. 第三步：把新的 vArmor 二进制复制到每个节点

把刚刚生成的 `bin/vArmor` 覆盖到节点上的 `/var/run/varmor/bin/vArmor-patched`：

```bash
kubectl -n varmor cp ./bin/vArmor ${UBUNTU24_LOADER}:/hostbin/vArmor-patched -c loader
kubectl -n varmor cp ./bin/vArmor ${WORKER_LOADER}:/hostbin/vArmor-patched -c loader
```

复制完成后，立刻校验节点上的目标文件哈希：

```bash
kubectl -n varmor exec ${UBUNTU24_LOADER} -c loader -- sha256sum /hostbin/vArmor-patched
kubectl -n varmor exec ${WORKER_LOADER} -c loader -- sha256sum /hostbin/vArmor-patched
```

检查要点：

* 两个节点上的哈希都必须和 `NEW_SHA` 一致
* 如果某个节点哈希不一致，先重新执行 `kubectl cp`，不要继续重启 agent

## 6. 第四步：重启 varmor-agent，让新二进制真正生效

只替换文件并不会让已经运行中的 agent 自动切到新版本，必须重建 Pod：

```bash
kubectl -n varmor delete pod -l app.kubernetes.io/component=varmor-agent
kubectl -n varmor rollout status ds/varmor-agent --timeout=150s
```

然后检查 agent 是否已经恢复：

```bash
kubectl -n varmor get ds varmor-agent
kubectl -n varmor get pods -l app.kubernetes.io/component=varmor-agent -o wide
kubectl -n varmor logs -l app.kubernetes.io/component=varmor-agent --tail=200
```

通过标准：

* `kubectl rollout status` 成功返回，没有超时
* `varmor-agent` DaemonSet 为全部 Ready
* 日志中能够看到：
  * `attach VarmorPathRename to the LSM hook point`
  * `attach VarmorMoveMount to the LSM hook point`
  * `vArmor agent is online`
* 日志中不再出现：
  * `field VarmorPathRename: program varmor_path_rename: load program: invalid argument`
  * `field VarmorMoveMount: program varmor_move_mount: load program: argument list too long: BPF program is too large`

只有 agent 启动成功以后，才有必要继续做工作负载和策略验证。

## 7. 第五步：部署测试工作负载

先准备测试命名空间：

```bash
kubectl get ns test1 >/dev/null 2>&1 || kubectl create ns test1
```

然后应用 Deployment：

```bash
kubectl apply -f demo-deploy-sleep.yaml
kubectl -n test1 rollout status deploy/varmor-demo-sleep --timeout=120s
kubectl -n test1 get pods -o wide
```

这个 Deployment 会创建两个副本，每个副本都有两个容器：

* `sleeper-1`
* `sleeper-2`

Pod 模板上已经带有 `sandbox.varmor.org/enable: "true"`，能够进入 vArmor 相关链路。

## 8. 第六步：下发 BPF 策略

应用策略对象：

```bash
kubectl apply -f test.yaml
kubectl -n test1 get varmorpolicy demo-policy
kubectl -n test1 get armorprofile
```

如果策略已经被 manager 正常处理，通常可以看到一个名字类似下面的 `ArmorProfile`：

```text
varmor-test1-demo-policy
```

建议继续查看该对象：

```bash
kubectl -n test1 get armorprofile varmor-test1-demo-policy -o yaml
```

## 9. 第七步：验证策略已经真正下发到目标容器

最直接的检查方式是看 agent 日志：

```bash
kubectl -n varmor logs -l app.kubernetes.io/component=varmor-agent --tail=300 | grep -E 'demo-policy|saving and applying the BPF profile|target container was created'
```

如果一切正常，通常可以看到类似下面的关键信息：

* `saving and applying the BPF profile 'varmor-test1-demo-policy (enforce)'`
* `target container was created`

这表示：

* 策略已经被转换为 BPF profile
* agent 已经把这个 profile 下发到对应节点
* 目标 Deployment 里的容器已经被 agent 命中并纳入管控

还可以从资源侧再检查一次：

```bash
kubectl -n test1 get varmorpolicy demo-policy -o yaml
kubectl -n test1 get armorprofile varmor-test1-demo-policy -o yaml
kubectl -n test1 get pods -l app=varmor-demo-sleep -o wide
```

## 10. 第八步：可选地做一次规则触发验证

由于这个示例启用的是 `disallow-write-core-pattern`，可以尝试进入容器执行一次对 `/proc/sys/kernel/core_pattern` 的写操作：

```bash
POD=$(kubectl -n test1 get pod -l app=varmor-demo-sleep -o jsonpath='{.items[0].metadata.name}')

kubectl -n test1 exec ${POD} -c sleeper-1 -- sh -c 'echo test > /proc/sys/kernel/core_pattern'
```

需要注意：

* 这个操作是否失败，不仅受 vArmor 影响，也受容器本身权限影响
* 因此，更可靠的验证信号仍然是 `ArmorProfile` 状态和 agent 日志中的 profile 下发记录

如果你还开启了审计链路，也可以进一步查看审计日志或 BPF ringbuf 相关输出。

## 11. 清理示例资源

验证完成后，可以按需删除示例对象：

```bash
kubectl delete -f test.yaml
kubectl delete -f demo-deploy-sleep.yaml
kubectl -n varmor delete ds varmor-bin-loader
```

如果你只是想移除测试工作负载，但保留策略对象或保留 loader DaemonSet，可以只执行其中一部分命令。

## 12. 这套示例最适合验证什么

这份例子最适合验证以下三件事是否已经同时成立：

1. Ubuntu 24 上的 `varmor-agent` 已经能够成功加载 `path_rename`、`move_mount` 等 BPF LSM 程序。
2. 新生成的 `bin/vArmor` 已经真正进入每个节点上的运行时路径，而不是只停留在本地构建目录。
3. 一个真实的 `VarmorPolicy` 已经能够被转换、下发，并命中目标 Deployment 下的容器。

如果这三件事都通过，说明“方案三”已经把本次 Ubuntu 24 内核兼容修复验证到了运行时层面。