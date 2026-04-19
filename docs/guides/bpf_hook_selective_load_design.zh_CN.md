# BPF Hook 细粒度选择与 Selective Load 设计

## 背景

当前 vArmor 的 BPF LSM 加载路径采用单一 eBPF 对象全量加载的方式：用户态读取一份预编译对象，初始化共享 maps 和 variables，然后把所有 LSM programs 一次性加载并依次 attach 到内核 hook 上。

这种模型实现简单，但在内核兼容性上有两个明显问题：

1. 某一个 hook 的函数原型、上下文约束或 verifier 行为发生变化时，可能导致整套对象加载失败。
2. 当前 profile 实际只使用了部分规则类型时，仍然会加载和 attach 全部 hook，扩大了兼容面和故障面。

vArmor 的 eBPF 源码已经通过 CO-RE 解决了大量结构体字段偏移差异问题，但 CO-RE 不能自动解决 LSM hook 原型变化、特定 hook verifier 压力变化、以及不同内核对程序 attach 行为的差异。因此，需要在加载模型上从“全量静态加载”升级为“按 hook 细粒度选择 + selective load”。

## 目标

本设计的目标如下：

1. 将 BPF LSM 的加载粒度从“整套对象”下沉到“单个 hook”。
2. 只为当前实际需要的规则类型加载对应 hook，避免无关 hook 参与 verifier 和 attach。
3. 为高风险 hook 提供多 variant 试探机制，在运行时选择当前内核可用的实现。
4. 使某个可选 hook 的失败不会拖垮整个 BPF enforcer。
5. 保持现有 profile 数据面和规则下发逻辑尽量稳定，降低改造成本。

## 非目标

本设计不试图解决以下问题：

1. 不保证一个 eBPF 程序可以无条件兼容所有内核版本。
2. 不在第一阶段重构 BPF 规则结构、CRD 结构或 profile 语义。
3. 不在第一阶段实现所有 hook 的多 variant，只优先覆盖最容易随内核变化的 hook。
4. 不在第一阶段引入复杂的自动卸载和动态收缩机制。

## 现状问题

### 1. 共享数据面和程序面耦合过深

当前 BPF enforcer 使用一个静态的 `bpfObjects` 承载全部 maps、variables 和 programs。这样做的直接后果是：

1. 一次 `LoadAndAssign` 默认尝试处理整套对象。
2. 某个 program 无法通过 verifier 或无法 attach 时，很难只剔除单个 hook。
3. Close 逻辑假设所有 link 都存在，不适合部分加载。

### 2. 加载前缺乏 hook 级能力判断

当前 agent 侧更多是做最低内核版本和 BPF LSM 开关判断。这可以挡住明显不支持的内核，但无法判断：

1. 某个特定 hook 是否可 attach。
2. 某个 hook 的当前函数原型是否与编译期假设一致。
3. 某个 hook 在当前内核下是否会触发 verifier 压力问题。

### 3. 规则使用面和 hook 使用面没有对齐

当前 profile 规则是按 Capabilities、Files、Processes、Networks、Ptrace、Mounts 分类下发，但 hook 仍然全部加载。实际上很多 profile 只会触发其中一部分 hook。

### 4. 高风险 hook 需要专门兼容策略

`path_rename`、`move_mount` 等 hook 在不同内核版本上的兼容风险明显更高。这类 hook 不适合继续沿用“单实现 + 整套失败”的模式。

## 总体方案

总体方案是把 BPF LSM 加载模型拆成三层：

1. 共享数据面：常驻加载，只初始化 maps 和 variables。
2. Hook 运行时：按需加载单个 hook 的 program，并按需 attach。
3. Variant 选择层：对高风险 hook 维护多个实现，运行时试探选择可用 variant。

核心原则如下：

1. 共享 maps 不随 hook variant 切换而变化。
2. program 的选择和 attach 结果按 hook 独立管理。
3. hook 是否启用由 profile 当前实际需要的规则类型决定。
4. 高风险 hook 可以先基于内核版本、发行版 flavor 和已知 changelog 做候选排序，再通过试探加载最终确认实现。

## 架构设计

### 1. 共享数据面

共享数据面负责承载所有 profile 下发所需的 maps 和 variables。第一阶段保留现有规则数据结构，不改变下发逻辑。

建议常驻加载的对象包括：

1. `v_profile_mode`
2. `v_audit_rb`
3. `v_buffer`
4. `v_pod_ip`
5. `v_capable`
6. `v_file_outer`
7. `v_bprm_outer`
8. `v_net_outer`
9. `v_ptrace`
10. `v_mount_outer`
11. `path_link_progs`
12. `path_rename_progs`
13. `init_mnt_ns`

这样做的好处是 profile 数据面仍然是一致的，`applyProfile()`、`deleteProfile()` 等现有逻辑可以最大程度复用。

### 2. Hook 运行时注册表

程序面不再使用静态字段保存所有 programs 和 links，而是改成运行时注册表。

建议定义如下概念：

```go
type HookID string

const (
    HookCapable           HookID = "capable"
    HookFileOpen          HookID = "file_open"
    HookPathSymlink       HookID = "path_symlink"
    HookPathLink          HookID = "path_link"
    HookPathRename        HookID = "path_rename"
    HookBprmCheckSecurity HookID = "bprm_check_security"
    HookSocketCreate      HookID = "socket_create"
    HookSocketConnect     HookID = "socket_connect"
    HookPtraceAccessCheck HookID = "ptrace_access_check"
    HookMount             HookID = "sb_mount"
    HookMoveMount         HookID = "move_mount"
    HookUmount            HookID = "sb_umount"
)

type HookVariant struct {
    Name            string
    EntryProgram    string
    TailPrograms    []string
    RequiredMaps    []string
    Optional        bool
}

type HookStatus struct {
    Enabled        bool
    Selected       string
    LastError      error
    LastErrorStage string
}

type HookRuntime struct {
    Program  *ebpf.Program
    Tails    map[string]*ebpf.Program
    Link     link.Link
    Variant  string
}
```

其中：

1. `HookVariant` 描述某个 hook 的候选实现。
2. `HookStatus` 负责记录当前 hook 是否可用，以及最后一次失败原因。
3. `HookRuntime` 负责保存已经成功加载的程序和 link。

### 3. Hook 家族与规则映射

应将 profile 规则类型和 hook 家族做显式绑定。建议映射如下：

| 规则类型 | 需要的 hook |
| --- | --- |
| Capabilities | `capable` |
| Files | `file_open`、`path_symlink`、`path_link`、`path_rename` |
| Processes | `bprm_check_security` |
| Networks.Address | `socket_connect` |
| Networks.Socket | `socket_create` |
| Ptrace | `ptrace_access_check` |
| Mounts | `sb_mount`、`move_mount`、`sb_umount` |

第一阶段建议对 Files 和 Mounts 使用“家族整体启用”策略，减少对业务语义的分析成本。

第二阶段可以进一步细化：

1. 文件规则只涉及读写时，不强制启用 `path_link` 和 `path_rename`。
2. mount 规则不涉及移动挂载时，不启用 `move_mount`。

## Selective Load 流程

### 阶段一：初始化共享对象

`initBPF()` 不再直接加载全部 programs，而是只完成以下动作：

1. `RemoveMemlock()`。
2. 读取一份基础 `CollectionSpec`。
3. 初始化 outer map 的 inner map 模板。
4. 设置 `init_mnt_ns`。
5. 只加载共享 maps 和 variables。

此阶段结束后，BPF enforcer 已具备 profile 数据面能力，但尚未 attach 任何 LSM hook。

### 阶段二：根据 profile 计算所需 hook

在 `SaveAndApplyBpfProfile()` 调用 `applyProfile()` 之前，先根据 `BpfContent` 计算本 profile 需要的 hook 集合。

建议引入：

```go
func ResolveRequiredHooks(content varmor.BpfContent) map[HookID]struct{}
```

这个集合应代表“为了正确执行当前 profile，必须存在的 hook”。

### 阶段三：确保所需 hook 已可用

在真正下发 profile 之前，调用：

```go
func (enforcer *BpfEnforcer) EnsureHooks(required map[HookID]struct{}) error
```

其流程如下：

1. 遍历 required hooks。
2. 已启用的 hook 直接跳过。
3. 未启用的 hook 先根据当前节点内核信息生成候选 variant 顺序，再逐个尝试。
4. 每次尝试时，只加载该 hook 对应的 programs。
5. 使用共享 maps 做 map replacement，避免重复创建数据面。
6. 如果存在 tail programs，则在主程序和 tail 程序都加载成功后回填 prog array。
7. attach 成功后记录 `HookRuntime` 和 `HookStatus`。
8. 如果所有 variant 都失败，返回错误。

### 阶段四：再执行 profile 下发

只有在 required hooks 已全部 ready 后，才执行现有 `applyCapabilityRule()`、`applyFileRules()`、`applyProcessRules()` 等规则下发逻辑。

这样可以确保 profile 的语义前提已经满足，不会出现规则写入了 map，但对应 hook 根本没生效的情况。

## Variant 选择策略

### 1. 适用对象

并不是所有 hook 都需要 variant。建议优先对以下 hook 使用多 variant：

1. `path_rename`
2. `move_mount`
3. 后续视情况扩展到 `path_link`

### 2. 选择原则

variant 选择可以利用内核版本信息做预判，但不应只依赖 `uname -r` 的硬编码判断，而应采用“版本预判 + 试探加载确认”的策略：

1. 为同一个 hook 维护一个有序 variant 列表。
2. 先根据当前节点的内核版本、发行版 flavor 和兼容矩阵，生成一个更符合当前环境的候选顺序。
3. 再按该顺序逐个尝试加载和 attach。
4. 第一个成功的 variant 即为当前节点选中实现。
5. 失败原因要记录到 `HookStatus.LastError` 中，便于排查。

### 3. 是否可以通过内核版本和 changelog 来判断

可以，但建议把它定位成“预判机制”，而不是“最终判定机制”。

更准确地说，应该利用内核版本和 changelog 产出的兼容知识来缩小候选集、调整 variant 尝试顺序，而不是在运行时仅凭一个版本字符串直接决定加载哪个实现。

原因如下：

1. 相同的大版本或小版本内核，在不同发行版上可能带有不同的 backport 补丁集。
2. 云厂商内核、发行版定制内核和自编译内核，可能携带额外 patch，但 `uname -r` 不能完整表达这些差异。
3. 节点上未必能稳定拿到完整、标准化的内核 package changelog，因此不适合依赖“运行时在线解析 changelog 文件”来做最终决策。

因此，更推荐的做法是：

1. 在离线研发阶段，根据 upstream 和发行版 changelog 维护一份兼容矩阵。
2. 在运行时读取节点内核标识，命中兼容矩阵后生成候选 variant 顺序。
3. 最后仍用试探加载来确认当前节点真正可用的实现。

运行时可采集的信号建议包括：

1. `uname -r` 的 release 字符串。
2. `/proc/version_signature` 或发行版提供的类似签名信息。
3. `/etc/os-release` 中的发行版与版本信息。
4. BTF、LSM 开关等能力信息。

推荐的判定流程如下：

1. 读取节点内核标识，构造 `KernelIdentity`。
2. 根据预置兼容矩阵，为某个 hook 生成优先级更高的 variant 候选顺序。
3. 先尝试最可能匹配当前内核的 variant。
4. 如果失败，再依次回退到其他 variant。
5. 只有全部 variant 都失败时，才认为该 hook 在当前节点不可用。

这种方式的优点是：

1. 比完全盲试探更快，日志也更容易解释。
2. 比纯版本字符串判断更稳健，能够容忍一部分发行版 backport 差异。
3. 能把 changelog 的价值真正转化为工程上的兼容知识，而不是运行时的脆弱依赖。

### 4. Wrapper + 共用核心逻辑

对于原型可能变化的 hook，不建议复制整份业务逻辑。更合理的方式是：

1. 每个 variant 只保留一个薄 wrapper，负责适配不同 hook 原型。
2. 业务逻辑下沉为共用 helper。
3. variant 间只允许在参数提取层不同，尽量不分叉业务语义。

例如：

1. `varmor_path_rename_v1()` 对应旧内核原型。
2. `varmor_path_rename_v2()` 对应新内核原型。
3. 二者都调用 `handle_path_rename(...)` 完成规则检查和审计逻辑。

## Tail Call 设计

当前 `path_link` 和 `path_rename` 使用静态 prog array 初始化。为了适应按 hook 选择和多 variant，需要改为“空 prog array + 用户态回填”。

建议规则如下：

1. prog array map 仍作为共享数据面的一部分常驻加载。
2. eBPF C 代码中只保留 map 定义，不在 `.values` 中写死 tail 程序引用。
3. 当某个 hook variant 被选中并加载成功后，由用户态把对应 tail program FD 填入共享 prog array map。
4. hook 被卸载时，用户态负责清空相应 prog array slot。

这样做的收益是：

1. tail 程序不再与单一静态符号强绑定。
2. variant 切换时无需重新组织整套对象。
3. selective load 对 tail-call hook 也成立。

## 错误处理与降级语义

### 1. 必选 hook

如果某个 profile 依赖的 hook 无法加载，应视为 profile 应用失败，而不是静默降级。

理由如下：

1. 否则 profile 表面上已经生效，实际上只执行了部分规则。
2. 对安全产品来说，静默丢失约束的风险高于显式失败。

### 2. 可选 hook

对于未来明确标记为可选的 hook，可以允许失败后继续运行，但必须满足两个前提：

1. 该 hook 对应的规则语义确实可以接受退化。
2. 退化结果会明确暴露到日志和状态中。

第一阶段不建议引入太多“可选 hook”，先保持行为确定性。

### 3. 审计与状态反馈

建议为每个 hook 输出以下信息：

1. 是否启用成功。
2. 选中的 variant 名称。
3. 失败阶段是 load、map replace、tail setup 还是 attach。
4. 最后一次 verifier 或 attach 错误。

后续可进一步考虑把这些信息纳入 Agent 状态上报，以便管理面感知节点兼容差异。

## 对现有代码的改造建议

### 1. 先拆数据面和程序面

建议把现有 `bpfObjects` 的使用方式拆成两类：

1. `sharedObjects`：只持有 maps 和 variables。
2. `hookRuntimes`：按 hook 保存运行时加载的 programs 和 links。

这样可以保留现有 profile 下发逻辑，同时为 selective load 留出空间。

### 2. 用运行时注册表替代静态 link 字段

当前 `BpfEnforcer` 中存在大量静态 link 字段，这不适合部分加载。建议改成：

```go
type BpfEnforcer struct {
    TaskStartCh      chan varmortypes.ContainerInfo
    TaskDeleteCh     chan varmortypes.ContainerInfo
    TaskDeleteSyncCh chan bool

    shared      sharedObjects
    hooks       map[HookID]*HookRuntime
    hookStatus  map[HookID]*HookStatus

    bpfProfileCache map[string]bpfProfile
    containerCache  map[string]enforceID
    log             logr.Logger
}
```

### 3. 封装单 hook 加载器

建议增加一个统一入口：

```go
func (enforcer *BpfEnforcer) loadHook(id HookID, variant HookVariant) error
```

它负责：

1. 获取该 variant 的专用 `CollectionSpec`。
2. 剪裁不相关 programs。
3. 替换共享 maps。
4. 加载主程序和 tail 程序。
5. 配置 prog array。
6. 执行 attach。
7. 保存运行时状态。

### 4. Close 逻辑改成遍历运行时状态

Close 阶段不应假设全部 link 都存在，而应改成：

1. 遍历 `hooks`，逐个关闭 link 和 program。
2. 清理共享 prog array 中的 tail entry。
3. 最后再关闭共享 maps 和 variables。

## 分阶段落地计划

### 第一阶段：只做单 hook selective load

目标：

1. 共享 maps 常驻加载。
2. 单 hook 程序按需加载。
3. 不引入 variant。
4. `Files`、`Mounts` 暂时仍按家族整体启用。

收益：

1. 即使某个高风险 hook 有问题，也不会影响完全无关的 hook。
2. 能验证共享数据面与程序面解耦是否成立。

### 第二阶段：为高风险 hook 引入 variant

目标：

1. 给 `path_rename` 增加多个 wrapper variant。
2. 给 `move_mount` 增加多个 variant。
3. 以“版本预判 + 试探加载确认”取代纯版本字符串硬编码判断。

收益：

1. 内核兼容能力明显增强。
2. 兼容差异被限制在单 hook 维度。

### 第三阶段：细化 hook 需求解析

目标：

1. Files 规则按权限位细化所需 hook。
2. Mounts 规则按操作类型细化所需 hook。
3. 只加载 profile 真正会用到的 hook。

收益：

1. 更少的 verifier 压力。
2. 更小的兼容面。

### 第四阶段：引入引用计数和自动卸载

目标：

1. 统计当前所有 profile 对 hook 的需求并集。
2. 在 hook 无引用时自动 detach。
3. 允许在长期运行过程中按需收缩启用集。

第一阶段可以暂时不做，以降低复杂度。

## 测试建议

建议按以下维度构建测试矩阵：

1. 内核版本：5.10、5.15、6.1、6.8。
2. 规则类型：仅 capability、仅 file、仅 network、仅 mount、组合规则。
3. 行为模式：enforce、complain。
4. 兼容场景：某个高风险 hook 故意失败，验证其他无关 hook 不受影响。
5. variant 场景：同一 hook 在不同内核上选中不同 variant。

至少应覆盖以下验证点：

1. profile 应用前会先保证所需 hook ready。
2. 单个非相关 hook 失败不会导致无关 profile 无法工作。
3. 必选 hook 失败时 profile 会显式失败。
4. path_link、path_rename 的 tail call 在 variant 模式下仍然正常工作。

## 风险与注意事项

### 1. map replacement 复杂度上升

Selective load 的本质是让多个单 hook 的程序共享同一组 maps。这要求用户态能够稳定完成 map replacement，且所有 hook variant 对共享 map 的定义必须严格一致。

### 2. tail-call hook 的维护成本更高

带 tail program 的 hook 需要额外管理 prog array 生命周期和 slot 写入逻辑，测试覆盖要比普通 hook 更严格。

### 3. 部分加载后日志和状态必须更清晰

一旦从“全量成功/失败”变成“按 hook 独立状态”，日志和状态信息就变得非常关键。否则定位兼容问题会更难。

### 4. profile 失败语义要保持一致

必须避免出现“规则已经写入 map，但 hook 实际不可用”的中间态。安全语义上，宁可显式失败，也不要静默半生效。

## 结论

对 vArmor 而言，“按 hook 细粒度选择 + selective load”是比“继续维护单一全量对象”更可持续的内核兼容路线。

它的核心不是依赖更多内核版本判断，而是把兼容问题收敛到单 hook 维度，并通过共享数据面、按需程序加载和 variant 试探，把失败影响范围从“整个 BPF enforcer”缩小到“单个 hook 家族”。

建议的实施顺序是：

1. 先完成共享数据面与程序面的解耦。
2. 再完成单 hook selective load。
3. 然后为高风险 hook 增加 variant。
4. 最后再逐步引入更细的按规则类型裁剪和自动卸载能力。

这一路线能在不重写现有 profile 数据面的前提下，显著提升 vArmor 对多内核环境的适应能力。