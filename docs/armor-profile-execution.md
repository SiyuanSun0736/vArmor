# vArmor: ArmorProfile 执行流程（从规则到生效）

本文档说明 vArmor 中一条像 `audit deny mount,` 这样的 AppArmor 规则，如何从生成被封装到 ArmorProfile CRD 并最终由节点 Agent 加载生效。适合排查或二次开发参考。

## 目标
- 说明规则如何生成
- 说明规则如何插入 AppArmor 模板并渲染为完整 profile
- 说明如何封装到 `ArmorProfile` CRD 的 `spec.profile.appArmor`
- 说明 Agent 如何写入并用 `apparmor_parser` 加载

## 关联源码（快速定位）
- 规则生成：[internal/profile/apparmor/rules.go](internal/profile/apparmor/rules.go#L36-L90)
- 模板拼接：[internal/profile/apparmor/apparmor.go](internal/profile/apparmor/apparmor.go#L1-L120)
- 模板内容：[internal/profile/apparmor/template.go](internal/profile/apparmor/template.go#L1-L200)
- Profile 构建与封装：[internal/profile/profile.go](internal/profile/profile.go#L56-L120)
- Controller 创建/更新 CRD：[internal/policy/policy_controller.go](internal/policy/policy_controller.go#L208-L246)
- Agent 写文件并加载：[internal/agent/agent.go](internal/agent/agent.go#L440-L500)
- AppArmor helper：[pkg/lsm/apparmor/apparmor.go](pkg/lsm/apparmor/apparmor.go#L1-L160)

## 步骤详解

1. 规则行生成
   - 函数：`generateHardeningRules(rule, qualifier)` 会根据规则名生成一行 AppArmor 条目。
   - 示例：`case "disallow-mount": rules += qualifier + "mount,\n"`（见源码）。
   - `qualifier` 由 `GenerateEnhanceProtectProfile` 构造，初始为两个空格（用于缩进），并根据配置追加 `audit ` 和/或 `deny `。
     - 例如：`"  audit deny "` → 生成 `  audit deny mount,`

2. 插入模板并拼接完整 profile 文本
   - `GenerateEnhanceProtectProfile` 会把所有硬化规则、攻击防护规则、自定义规则等累加到 `baseRules`，并选择合适模板：
     - 特权容器使用 `alwaysAllowTemplate`；非特权（enhance protect）使用 `runtimeDefaultTemplateForEnhanceProtectMode`。
   - 对增强保护模板，模板中使用 `QUALIFIER` 占位，函数会用 `strings.ReplaceAll(..., "  QUALIFIER ", qualifier)` 将其替换为实际前缀（含 `audit`/`deny` 等）。
   - 最终通过 `fmt.Sprintf` 把 `profileName`、peer 名称和 `baseRules` 填入模板，得到完整的 AppArmor profile 文本（即 `spec.profile.appArmor` 的值）。

3. 封装到 `ArmorProfile` CRD
   - `GenerateProfile` 将生成的 profile 文本赋给 `Profile.AppArmor`（string）。见：[internal/profile/profile.go](internal/profile/profile.go#L56-L100)。
   - `NewArmorProfile` 将该 `Profile` 放入 `ArmorProfile.Spec.Profile` 并返回 `ArmorProfile` 对象。
   - Policy controller 调用 vArmor API 创建或更新该 CR：
     ```go
     ap, err = c.varmorInterface.ArmorProfiles(vp.Namespace).Create(context.Background(), ap, metav1.CreateOptions{})
     ```
   - CR 保存到 Kubernetes（etcd），供节点 Agent 通过 informer 读取。

4. 节点端保存并加载
   - Agent 监听到 `ArmorProfile` 的 create/update 事件后，执行：
     1. `varmorapparmor.SaveAppArmorProfile(profilePath, ap.Spec.Profile.AppArmor)`
        - 该函数会使用 Go 的 `text/template` 渲染模板数据（比如 `DiskDevices`），并把最终文本写入节点文件（例如 `/etc/apparmor.d/varmor-...`）。见：[pkg/lsm/apparmor/apparmor.go](pkg/lsm/apparmor/apparmor.go#L1-L80)。
     2. 如果 profile 未加载，执行加载：
        - 强制模式（enforce）：`apparmor_parser -Ka /path/to/profile`
        - Complain 模式：`apparmor_parser -KaC /path/to/profile`
       （`LoadAppArmorProfile` 内部调用 `apparmor_parser`）
     3. 若为更新事件，Agent 会调用 `UpdateAppArmorProfile`（`apparmor_parser -Kr` / `-KrC`）进行重载。

5. 运行时生效
   - 当内核中的 AppArmor profile 以 `enforce` 模式加载，并且规则行为为 `audit deny mount,`：
     - 内核会拒绝受管进程的 mount 系统调用，并生成审计事件；
     - 如果处于 `complain` 模式则不会阻止，但会记录违规信息。
   - vArmor 的 `auditor` / `monitor` 可捕获审计/违规并上报/记录。

6. 并行/补充机制
   - 若策略同时启用了 BPF 或 Seccomp，相应的生成逻辑会在生成 profile 时一并产生（见 `internal/profile/bpf` 与 `internal/profile/seccomp`），并由 Agent 下发：BPF 使用 `SaveAndApplyBpfProfile`。

## 示例：最终 profile 片段（取核心部分）

```text
## == Managed by vArmor == ##

abi <abi/3.0>,
#include <tunables/global>

profile varmor-example flags=(attach_disconnected,mediate_deleted) {

  #include <abstractions/base>

  network,
  capability,
  file,
  umount,

  # ... 省略若干规则 ...

  audit deny mount,

  # ... 其余规则 ...
}
```

## 常用调试命令（在节点上）

- 查看是否已加载 profile：

```bash
cat /sys/kernel/security/apparmor/profiles | grep varmor-example
```

- 加载 profile（Agent 通常会执行）：

```bash
apparmor_parser -Ka /etc/apparmor.d/varmor-example
# 或者 complain 模式
apparmor_parser -KaC /etc/apparmor.d/varmor-example
```

- 卸载 profile：

```bash
apparmor_parser -R /etc/apparmor.d/varmor-example
```

## 参考源码位置
- [internal/profile/apparmor/rules.go](internal/profile/apparmor/rules.go#L36-L90)
- [internal/profile/apparmor/apparmor.go](internal/profile/apparmor/apparmor.go#L1-L120)
- [internal/profile/apparmor/template.go](internal/profile/apparmor/template.go#L1-L200)
- [internal/profile/profile.go](internal/profile/profile.go#L56-L120)
- [internal/policy/policy_controller.go](internal/policy/policy_controller.go#L208-L246)
- [internal/agent/agent.go](internal/agent/agent.go#L440-L500)
- [pkg/lsm/apparmor/apparmor.go](pkg/lsm/apparmor/apparmor.go#L1-L160)
