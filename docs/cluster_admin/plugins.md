# 插件 {#plugins}

## 概述 {#overview}

插件允许用树外功能扩展 KubeVirt。Plugin 是一种
集群作用域的自定义资源，用于定义 launcher 钩子、节点钩子和
准入引用。

插件功能目前处于 Alpha 阶段（在 v1.9 中引入），并且需要 `Plugins` 功能门控。

## 启用功能门控 {#enabling-the-feature-gate}

按照
[激活功能门控](activating_feature_gates.md)
中的步骤启用 `Plugins` 功能门控。

## Launcher 钩子 {#launcher-hooks}

Launcher 钩子在 virt-launcher Pod 内、于 VM 生命周期中定义明确的时间点运行。每个钩子要么是 CEL 表达式，要么是 sidecar 容器。

| 钩子点 | 运行时机 | CEL | Sidecar |
|------------|--------------|-----|---------|
| `GuestDefinition` | 域生成之后、交给 libvirt 之前 | 是 | 是 |
| `PreBoot` | 启动产物生成之后、客户机启动之前 | 否 | 是 |
| `PreMigrationSource` | 在迁移源端，libvirt 开始迁移之前 | 否 | 是 |

钩子在每个插件内按声明顺序应用。跨插件时，
钩子按插件名称的字母顺序应用。

### CEL 表达式 {#cel-expressions}

CEL 表达式针对单个钩子点，由 `hookPoint` 设置。CEL
仅支持 `GuestDefinition` 钩子点。

该表达式就地转换域。它使用 `vmi` 和 `domainSpec`
变量，其字段通过 Go 名称访问（例如
`vmi.Spec.Domain`），并且必须使用
[libvirtxml](https://pkg.go.dev/libvirt.org/go/libvirtxml) 类型名返回 `Domain{...}` 结构体。结果会与生成的域深度合并——
表达式中提到的字段才会被覆盖，未提到的字段
保持不变。切片会被整体替换，而不是逐元素合并。

```yaml
apiVersion: plugin.kubevirt.io/v1alpha1
kind: Plugin
metadata:
  name: cel-domain-example
spec:
  launcherHooks:
  - cel:
      hookPoint: GuestDefinition
      expression: 'Domain{CPU: DomainCPU{Mode: "host-passthrough"}}'
```


### Sidecar {#sidecar}

virt-launcher 中的 sidecar 容器通过 Unix 套接字提供 gRPC。当需要
超出 CEL 表达能力的逻辑时，请使用
sidecar 钩子，例如读取
外部状态、发起 API 调用或修改文件。

Plugin CR 引用套接字路径，并在 `permittedHooks` 中列出该 sidecar
处理的钩子点，因此单个 sidecar 可以服务多个钩子点。
要将 sidecar 容器注入 virt-launcher Pod，请设置
MutatingAdmissionPolicy（或 webhook）。这样你可以完全控制
sidecar 容器规约（镜像、资源、安全上下文、卷）。

```yaml
apiVersion: plugin.kubevirt.io/v1alpha1
kind: Plugin
metadata:
  name: my-plugin
spec:
  launcherHooks:
  - sidecar:
      socketPath: "/var/run/kubevirt-plugin/my-plugin/hook.sock"
      permittedHooks:
      - GuestDefinition
      - PreBoot
      - PreMigrationSource
```


**套接字路径约束：**

- 套接字必须位于 `/var/run/kubevirt-plugin/<plugin-name>/` 之下。KubeVirt 将 `/var/run/kubevirt-plugin` 作为 `kubevirt-plugin-sockets` 卷添加到 virt-launcher Pod 中，并与计算容器共享。
- 路径必须以 `.sock` 结尾，长度至多 108 字节，且不得包含 `..`，也不得包含重复或末尾的 `/`。
- 套接字路径中不允许符号链接。
- Sidecar 必须挂载 `kubevirt-plugin-sockets` 卷并使用 `subPath`，以便每个插件只能隔离访问自己的目录。违反此要求的 Pod 会被拒绝。

在每个钩子点，virt-launcher 总共最多等待 30 秒，
以等待处理该钩子点的 sidecar 套接字出现。此等待与
每个钩子的 `timeout` 相互独立。

### GuestDefinition {#guestdefinition}

`GuestDefinition` 钩子在域被交给 libvirt 之前变更域。
它在 VM 启动时运行，并且在单次
启动期间可能被调用不止一次，因此实现必须是幂等的。

在迁移目标端，该钩子还会带着 `MigrationTarget`
调用上下文被调用，以便 sidecar 可以准备目标 Pod。其返回的
域不会被使用：目标端运行的是从源端迁移来的域，其中
已经包含启动时应用的变更。该钩子不会在
迁移源端运行。

基于注解的旧式[钩子 sidecar](../user_workloads/hook-sidecar.md)在
插件钩子之前运行，因此插件看到的是已被它们修改过的域。

### PreBoot {#preboot}

`PreBoot` 钩子在启动产物（例如 cloud-init ISO）
生成之后、客户机启动之前运行。用它来修改依赖于运行时信息的客户机
初始化数据，例如
根据 Pod 的 network-status
注解构建的 cloud-init 网络配置。

请求中包含 VMI 以及计算容器内已生成产物的路径：

- `cloudInitISO`：生成的 cloud-init ISO（如果该 VM 具有 cloud-init 卷）。
- `ephemeralDiskDir`：存放临时磁盘的目录，
  `/var/run/kubevirt-ephemeral-disks`。

要访问这些产物，sidecar 必须把 `ephemeral-disks` 卷挂载到
`/var/run/kubevirt-ephemeral-disks`，与计算容器中的路径相同。
当多个插件处理 `PreBoot` 时，每个 sidecar 看到的是上一个
插件留下的文件。

需要注意的事项：

- `PreBoot` 仅在 VM 启动时运行，不会在迁移目标端运行。热插拔到正在运行的 VM 中的卷
  永远不会经过它。
- `PreBoot` 在启动失败后可能会被再次调用，并且文件变更不会
  回滚。实现必须是幂等的。
- 设置了 `failureStrategy: Ignore` 的钩子在超过其
  超时后不会被停止：VM 会启动，而钩子可能仍在修改文件。会修改文件的钩子
  应当使用 `failureStrategy: Fail`，以原子方式写入
  （先写临时文件再重命名），并遵守请求上下文。
- `GuestDefinition` 钩子在 `PreBoot` 钩子之前运行。

### PreMigrationSource {#premigrationsource}

`PreMigrationSource` launcher 钩子在迁移源端运行，就在
virt-launcher 把迁移交给 libvirt 之前。它允许插件
按 VM 调整迁移，例如启用并行（multifd）迁移
或选择压缩方法。

请求中包含 VMI 以及 KubeVirt 即将
传递给 libvirt 的迁移配置：`VIR_MIGRATE_*` 标志位掩码和
`VIR_MIGRATE_PARAM_*` 参数。响应中只包含变更：

- `set_flags` 添加标志位，`clear_flags` 移除标志位。同一位同时出现在
  二者中则是错误。
- 对于每个参数，未设置的字段保持当前值，已设置的字段
  会覆盖当前值，空包装或空列表会移除该参数。
- 空响应不改变任何内容。

当多个钩子处理 `PreMigrationSource` 时，每个钩子收到的是
已被前面的钩子修改过的配置。结果会按原样传递给
libvirt，因此插件对迁移配置（包括迁移 URI）拥有完全的控制，也承担全部
责任。

需要注意的事项：

- 如果设置了 `failureStrategy: Fail` 的钩子失败，迁移即失败。若为
  `Ignore`，迁移将使用此前成功的钩子所返回的配置继续进行。
- 如果在钩子运行期间中止迁移，请求上下文会被
  取消。钩子应当遵守该上下文。
- 迁移监控在钩子返回之后才启动，因此花在
  钩子上的时间不计入迁移超时。
- 迁移监控遵循结果中的 `VIR_MIGRATE_POSTCOPY` 标志：
  清除它会禁用后拷贝，设置它则允许在与 KubeVirt 设置该标志时
  相同的条件下使用后拷贝。

此钩子不同于 `PreMigrationSource` 节点钩子，后者在
迁移开始之前于节点上运行，并且不能更改迁移
配置。

## 节点钩子 {#node-hooks}

节点钩子在 VM 生命周期事件期间执行，由 virt-handler 通过 gRPC 调用
插件服务器。插件作者以 DaemonSet 形式部署并管理此服务器，
使集群管理员可以通过
标准 Kubernetes 机制控制其能力与资源。

可用的钩子点：

- `PreVMStart`：VM 启动之前。
- `PostVMStart`：VM 进入运行状态之后。
- `OnVMStop`：当 KubeVirt 停止正在运行的 VM 时，例如 VMI 被
  删除时。客户机自行关机或崩溃时不会触发。
- `PostVMStop`：VM 已经停止之后。
- `PreMigrationSource`：在源节点上，迁移开始之前。
- `PreMigrationTarget`：在目标节点上，迁移开始之前。
- `PostMigrationTarget`：在目标节点上，迁移完成之后。

由于协调重试，节点钩子可能针对同一生命周期事件触发多次。实现必须是幂等的。

套接字必须位于 virt-handler 所挂载的宿主机路径上，例如
`/var/run/kubevirt/` 之下。

```yaml
apiVersion: plugin.kubevirt.io/v1alpha1
kind: Plugin
metadata:
  name: node-hook-example
spec:
  nodeHooks:
  - socket: /var/run/kubevirt/plugins/my-plugin.sock
    permittedHooks:
    - PreVMStart
    - OnVMStop
```


## 准入引用 {#admission-references}

插件可以引用 Kubernetes 准入策略和 webhook，以便修改 Pod/VMI：

```yaml
apiVersion: plugin.kubevirt.io/v1alpha1
kind: Plugin
metadata:
  name: admission-example
spec:
  mutatingAdmissionPolicies:
    - name: "my-mutating-policy"
  validatingAdmissionPolicies:
    - name: "my-validating-policy"
  mutatingAdmissionWebhooks:
    - name: "my-mutating-webhook"
  validatingAdmissionWebhooks:
    - name: "my-validating-webhook"
```


将这些内容列在 Plugin CR 中，是为了提供一个集中位置，以便了解所有插件组件。

## 失败处理与条件 {#failure-handling-and-conditions}

- **`failureStrategy`**：`Fail`（默认）在出错时阻止该操作。`Ignore` 记录错误并继续。对于 launcher 钩子，可以在插件级别将其设为默认值，并按钩子覆盖。节点钩子只使用自身的 `failureStrategy`。
- **`condition`**：一个 CEL 表达式，用于筛选插件应用于哪些 VMI。它只能引用 `vmi`，使用 Go 字段名，并且必须求值为布尔值。可以在插件级别和每个钩子上设置。每个钩子的条件会进一步收窄插件级别的筛选。求值条件时出错会使操作失败，与 `failureStrategy` 无关。
- **`timeout`**：sidecar launcher 钩子和节点钩子的每个钩子最大等待时长。默认为 30 秒。它不适用于 CEL 钩子。

此示例仅适用于带有 `example.com/tuning` 标签的 VMI。
CEL 钩子进一步收窄到标签值为 `cpu` 的 VMI。
sidecar 钩子调整所有带该标签的 VMI 的迁移；如果它失败，或在
10 秒内没有响应，迁移会在不采用其变更的情况下继续。

```yaml
apiVersion: plugin.kubevirt.io/v1alpha1
kind: Plugin
metadata:
  name: full-example
spec:
  failureStrategy: Fail
  condition: '"example.com/tuning" in vmi.Labels'
  launcherHooks:
  - cel:
      hookPoint: GuestDefinition
      expression: 'Domain{CPU: DomainCPU{Mode: "host-passthrough"}}'
    condition: 'vmi.Labels["example.com/tuning"] == "cpu"'
  - sidecar:
      socketPath: "/var/run/kubevirt-plugin/full-example/hook.sock"
      permittedHooks:
      - PreMigrationSource
    failureStrategy: Ignore
    timeout: 10s
```


## 开发插件 {#developing-plugins}

Sidecar launcher 钩子和节点钩子通过 gRPC 通信。proto 定义
位于 KubeVirt 仓库中的
[`pkg/hooks/plugins/v1alpha1/api.proto`](https://github.com/kubevirt/kubevirt/blob/main/pkg/hooks/plugins/v1alpha1/api.proto)。

- **Launcher 钩子**实现 `LauncherHookService`，每个钩子点对应一个 RPC：
    - `GuestDefinition`：请求包含域 XML、VMI 对象（JSON 形式），以及带有调用上下文（`Boot` 或 `MigrationTarget`）的 sidecar 上下文。响应包含修改后的域 XML。
    - `PreBoot`：请求包含 VMI 对象（JSON 形式）和产物路径。
    - `PreMigrationSource`：请求包含 VMI 对象（JSON 形式）以及 libvirt 迁移标志和参数。响应包含要应用的变更。
- **节点钩子**实现 `NodeHookService.ExecuteNodeHook` RPC。请求包含钩子点名称、VMI 对象（JSON 形式）以及节点上下文。

sidecar 只需要实现其
`permittedHooks` 中列出的钩子点所对应的 RPC。

## 相关内容 {#related}

- [钩子 Sidecar 容器](../user_workloads/hook-sidecar.md) - 旧机制（已弃用）。
- [VEP 190](https://github.com/kubevirt/enhancements/blob/main/veps/sig-compute/190-kubevirt-structured-plugins/vep.md) - 设计细节。
