# VirtualMachine 模板 {#virtualmachine-templates}

**功能状态：**

* `template.kubevirt.io/v1alpha1`（Alpha），自 [`v1.8.0`](https://github.com/kubevirt/kubevirt/releases/tag/v1.8.0) KubeVirt 发行版起
* `template.kubevirt.io/v1beta1`（Beta），自 `v1.9.0` KubeVirt 发行版起

## 简介 {#introduction}

`VirtualMachineTemplates` 为 KubeVirt 提供原生的、集群内的 VM 模板能力。它们允许你定义带有参数化取值的可复用 VM 蓝图，这些蓝图经过处理后可以创建 `VirtualMachine` 对象。

与 Helm 或 Kustomize 等外部模板工具不同，`VirtualMachineTemplates` 可以捕获外部工具无法表示的存储状态（例如 `DataVolume` 快照和克隆的磁盘）。它们也可以在任何 Kubernetes 集群上工作，而不需要 OpenShift。

!!! Note
    基于 OpenShift Templates 的旧式模板机制可以在 OpenShift 集群上找到。详情参见[模板](templates.md)。新的部署应使用本页所述的原生 `VirtualMachineTemplate` 功能。

### 何时使用模板，何时使用实例类型和偏好 {#when-to-use-templates-vs-instancetypes-and-preferences}

对于只需要标准化 CPU、内存和设备偏好的简单 VM，请使用 [VirtualMachineInstancetype 和 VirtualMachinePreference](instancetypes.md)。当你还需要对网络、卷以及 `DataVolume` 源这类集群特定资源做模板化时，模板就变得有价值，而这些内容是实例类型和偏好无法覆盖的。

模板与实例类型和偏好也可以[一起使用](#example)，以获得最大的灵活性。

### CRD {#crds}

该功能引入两个自定义资源定义：

- **`VirtualMachineTemplate`**：定义带有参数的可复用 VM 蓝图
- **`VirtualMachineTemplateRequest`**：从现有 VM 创建模板（黄金镜像工作流）

## 启用该功能 {#enabling-the-feature}

从 KubeVirt v1.9.0 开始，Template 功能门控处于 Beta，并且默认启用。无需任何操作即可启用它。

在 KubeVirt v1.8.x 上，Template 功能门控处于 Alpha，必须与 `Snapshot` 依赖项一起显式启用：

```shell
$ kubectl patch kubevirt kubevirt -n kubevirt --type merge -p '{"spec":{"configuration":{"developerConfiguration":{"featureGates":["Snapshot","Template"]}}}}'
```

一旦启用，KubeVirt 就会自动部署原生 `VirtualMachineTemplates` 所需的组件。

关于功能门控的更多细节，参见[激活功能门控](../cluster_admin/activating_feature_gates.md)。

### 禁用 virt-template 部署 {#disabling-virt-template-deployment}

要在保持 Template 功能门控启用的同时禁用 virt-template 组件的部署，请把 `virtTemplateDeployment.enabled` 设置为 `false`，位置在 `KubeVirt` CR 中：

```yaml
apiVersion: kubevirt.io/v1
kind: KubeVirt
metadata:
  name: kubevirt
  namespace: kubevirt
spec:
  configuration:
    virtTemplateDeployment:
      enabled: false
```

!!! Note
    `virtctl template` 子命令要求 virtctl v1.8.0 或更高版本。

## VirtualMachineTemplate {#virtualmachinetemplate}

### 概述 {#overview}

`VirtualMachineTemplate` 是 `template.kubevirt.io/v1beta1` API 组中的命名空间作用域资源。

它由下列部分组成：

- **`spec.parameters`**：处理模板时可以替换的参数列表
- **`spec.virtualMachine`**：带有参数占位符的 `VirtualMachine` spec 蓝图
- **`spec.message`**：处理后返回的可选消息

### 参数 {#parameters}

`spec.parameters` 中的每个参数支持下列字段：

| 字段 | 说明 |
|---------------|----------------------------------------------------------------------------|
| `name` | 参数名称，在 VM spec 中用作 `${NAME}` 占位符 |
| `value` | 如果处理期间未提供该参数，则使用的默认值 |
| `generate` | 设置为 `expression`，以便使用 `from` 模式自动生成值 |
| `from` | 用于生成值的正则表达式模式（需要 `generate`） |
| `required` | 若为 `true`，则必须提供该参数，并且没有默认值 |
| `displayName` | 供 UI 显示的人类可读名称 |
| `description` | 对该参数用途的说明 |

**静态参数**具有固定的默认值：

```yaml
- name: INSTANCETYPE
  value: u1.medium
  description: Instance type for the VM
```

**必需参数**必须在处理期间提供：

```yaml
- name: PASSWORD
  required: true
  description: Password for the cloud-init user
```

**生成参数**会基于模式产生随机值：

```yaml
- name: NAME
  generate: expression
  from: "fedora-[a-z0-9]{16}"
  description: Unique VM name
```

`from` 字段支持下列表达式元素：

| 元素 | 说明 |
|---------|----------------------------------------|
| `[a-z]` | 小写 ASCII 字母 |
| `[A-Z]` | 大写 ASCII 字母 |
| `[0-9]` | 十进制数字 |
| `\w` | 字母数字字符（`[a-zA-Z0-9]`） |
| `\d` | 十进制数字（`[0-9]`） |
| `\a` | 小写 ASCII 字母（`[a-z]`） |
| `\A` | 大写 ASCII 字母（`[A-Z]`） |
| `{n}` | 将前一个元素重复 `n` 次 |

**可选参数**在没有默认值或生成器时不是必需的。如果在处理期间未提供，它们的占位符会被替换为空字符串。

当创建或更新 `VirtualMachineTemplate`，并且所有必需参数都已有值（无论是默认值还是生成的值）时，会通过执行一次试运行处理来校验该模板。会产生无效 `VirtualMachine` spec 的模板会被拒绝。

### 示例 {#example}

模板可以在 VM spec 中引用实例类型和偏好。实例类型定义计算资源，偏好定义设备和操作系统的默认值，而模板则用集群特定的网络、卷以及 `DataVolume` 源把这一切组合在一起。更多细节参见[实例类型和偏好](instancetypes.md)。

下面的示例定义一个 Fedora VM 模板，它带有生成的名称、默认的实例类型和偏好，以及一个必需的密码参数：

```yaml
apiVersion: template.kubevirt.io/v1beta1
kind: VirtualMachineTemplate
metadata:
  name: fedora-template
spec:
  parameters:
    - name: NAME
      generate: expression
      from: "fedora-[a-z0-9]{16}"
      description: Unique VM name
    - name: INSTANCETYPE
      value: u1.medium
      description: Instance type for the VM
    - name: PASSWORD
      required: true
      description: Password for the cloud-init user
  virtualMachine:
    metadata:
      name: ${NAME}
    spec:
      instancetype:
        kind: VirtualMachineClusterInstancetype
        name: ${INSTANCETYPE}
      preference:
        kind: VirtualMachineClusterPreference
        name: fedora
      runStrategy: Halted
      template:
        spec:
          domain:
            devices: { }
          volumes:
            - containerDisk:
                image: quay.io/containerdisks/fedora:latest
              name: rootdisk
            - cloudInitNoCloud:
                userData: |-
                  #cloud-config
                  user: fedora
                  password: ${PASSWORD}
                  chpasswd: { expire: False }
              name: cloudinitdisk
```

## 处理 `VirtualMachineTemplates` {#processing-virtualmachinetemplates}

### 查看模板参数 {#viewing-template-parameters}

要列出模板中定义的参数：

```shell
$ virtctl template process fedora-template --print-params
NAME                DESCRIPTION                        GENERATOR    VALUE
NAME                Unique VM name                     expression   fedora-[a-z0-9]{16}
INSTANCETYPE        Instance type for the VM                        u1.medium
PASSWORD            Password for the cloud-init user
```

### 服务端处理 {#server-side-processing}

默认情况下，在使用 `virtctl template process` 时，模板在服务端进行处理：

```shell
$ virtctl template process fedora-template -p PASSWORD=secretpass
```

这会输出结果 `VirtualMachine` YAML，其中所有参数都已替换。

### 客户端处理 {#client-side-processing}

对于以本地文件形式存储的模板，使用 `--local` 在客户端进行处理，而无需联系集群：

```shell
$ virtctl template process --local -f fedora-template.yaml -p PASSWORD=secretpass
```

### 输出格式 {#output-formats}

使用 `--output` 来控制输出格式：

```shell
$ virtctl template process fedora-template -p PASSWORD=secretpass --output yaml
$ virtctl template process fedora-template -p PASSWORD=secretpass --output json
```

默认的输出格式是 YAML。

### 直接创建 VM {#direct-vm-creation}

使用 `--create` 处理模板，并在同一步骤中创建结果 `VirtualMachine`：

```shell
$ virtctl template process fedora-template -p PASSWORD=secretpass --create
```

!!! Note
    直接创建 VM 使用子资源 API（API 组 `subresources.template.kubevirt.io`），并带有单独的 RBAC 控制。集群管理员可以只授予 `process/create` 子资源权限，而不授予直接的 `VirtualMachine` 创建权限，从而把用户限制为只能从预定义模板创建 VM。详情参见 [RBAC](#rbac) 一节。

## 从现有 VM 创建模板 {#creating-templates-from-existing-vms}

### VirtualMachineTemplateRequest {#virtualmachinetemplaterequest}

`VirtualMachineTemplateRequest` 创建 `VirtualMachineTemplate`，其来源是现有的 `VirtualMachine`。这样就可以实现黄金镜像工作流：你按自己的需要配置一台 VM，然后把它转换成可复用的模板。

### 跨命名空间创建模板 {#cross-namespace-template-creation}

`VirtualMachineTemplateRequest` 的主要用途之一，是根据用户私有命名空间中正在运行的 VM，在共享的或“目录”命名空间中创建模板。

```shell
$ cat <<EOF | kubectl apply -f -
apiVersion: template.kubevirt.io/v1beta1
kind: VirtualMachineTemplateRequest
metadata:
  name: promote-to-golden
  namespace: shared-templates  # The template will be created here
spec:
  virtualMachineRef:
    name: my-configured-vm
    namespace: my-namespace  # Source VM in a different namespace
  templateName: my-golden-vm
EOF
```

**RBAC 要求：**
要从来自另一个命名空间的 VM 创建模板，用户（或控制器的服务账号）必须在**源** VM 的命名空间中拥有 `virtualmachinetemplaterequest/create` 权限，并且在**目标**命名空间中拥有创建 `VirtualMachineTemplates` 和 `DataVolumes` 的权限。

!!! Warning
    模板创建过程并**不会**执行 sysprep。来自源 VM 的全部 secret、凭据以及机器特定标识符都会保留在结果模板中。在分发该模板之前，你应当审查敏感值并对其参数化。

### 示例：使用 `DataVolumeTemplates` 的持久存储 {#example-persistent-storage-with-datavolumetemplates}

虽然 `containerDisk` 对于测试很有用，生产模板常常使用 `dataVolumeTemplates`，以便从黄金镜像 PVC 克隆持久存储。

```yaml
apiVersion: template.kubevirt.io/v1beta1
kind: VirtualMachineTemplate
metadata:
  name: fedora-pvc-template
spec:
  parameters:
    - name: NAME
      generate: expression
      from: "fedora-[a-z0-9]{16}"
      description: Unique VM name
  virtualMachine:
    metadata:
      name: ${NAME}
    spec:
      runStrategy: Halted
      template:
        spec:
          domain:
            devices:
              disks:
                - disk:
                    bus: virtio
                  name: rootdisk
          volumes:
            - dataVolume:
                name: ${NAME}-rootdisk
              name: rootdisk
      dataVolumeTemplates:
        - metadata:
            name: ${NAME}-rootdisk
          spec:
            source:
              pvc:
                name: fedora-golden
                namespace: shared-templates
            storage: {}
```

### 使用 virtctl template create {#using-virtctl-template-create}

`virtctl template create` 命令提供了从现有 VM 创建模板的快捷方式：

```shell
$ virtctl template create my-configured-vm
virtualmachinetemplate.template.kubevirt.io/my-configured-vm created
```

## 转换 OpenShift `Templates` {#converting-openshift-templates}

### virtctl template convert {#virtctl-template-convert}

如果你已有现有的 OpenShift Templates（`template.openshift.io/v1`），其中包含单个 `VirtualMachine` 对象，可以把它们转换成原生的 `VirtualMachineTemplates`：

```shell
$ virtctl template convert -f openshift-fedora-template.yaml
```

这会输出转换后的 `VirtualMachineTemplate` YAML。该转换会把 OpenShift 模板参数映射为 `VirtualMachineTemplate` 参数。

!!! Note
    该转换要求 OpenShift 模板恰好包含一个 `VirtualMachine` 对象。不支持包含多个对象的模板。

## RBAC {#rbac}

### 用户角色 {#user-personas}

| 角色 | 说明 | 权限 |
|-------------------|-----------------------------------------------|---------------------------------------------------------------------------------------|
| VM 所有者 | 从现有模板创建 VM | 对模板的 `process` 和 `create` 子资源（`subresources.template.kubevirt.io`） |
| VM 模板所有者 | 创建并管理 `VirtualMachineTemplates` | 对两个 CRD 的完整 CRUD |
| 集群管理员 | 对 VM 的创建方式强制执行策略 | 完整 CRUD 外加 RBAC 管理 |

### 聚合的 `ClusterRoles` {#aggregated-clusterroles}

Template 功能定义了 `ClusterRoles`，它们会聚合进标准的 Kubernetes `admin`、`editor` 和 `view` 角色。

- **admin / editor**：对 `VirtualMachineTemplates` 和 `VirtualMachineTemplateRequests` 的完整 CRUD 访问
- **view**：对 `VirtualMachineTemplates` 和 `VirtualMachineTemplateRequests` 的只读访问

### 把 VM 的创建限制为通过模板 {#restricting-vm-creation-to-templates}

一个关键的 RBAC 模式，是限制用户只能通过模板创建 VM，而不能任意自由创建。要做到这一点：

1. 授予用户 `process/create` 子资源权限，作用对象是 `VirtualMachineTemplates`（API 组 `subresources.template.kubevirt.io`）
2. **不要**授予直接的 `create` 权限，其对象是 `VirtualMachine` 资源（API 组 `kubevirt.io`）

这确保所有 VM 都从已批准的模板创建，从而使管理员能够控制集群中所允许的 VM 配置。

## OCI 导出 {#oci-export}

从 KubeVirt v1.9.0 开始，可以使用[导出 API](../storage/export_api.md)把 `VirtualMachineTemplates` 导出为 OCI 制品。这允许你通过标准镜像仓库把模板作为容器镜像来分发。需要 `OCIExport` 和 `Template` 功能门控，并且必须启用 [virt-template 部署](#disabling-virt-template-deployment)。`Template` 以及 virt-template 部署默认是启用的。

关于 OCI 导出、功能门控设置以及推送到镜像仓库的完整细节，参见[导出 API - OCI 格式](../storage/export_api.md#oci-format)文档。

### 导出模板 {#exporting-a-template}

为 `VirtualMachineTemplate` 创建一份 VMExport，并以 OCI 格式下载它：

```shell
$ virtctl vmexport create my-export --vmtemplate=fedora-template
$ virtctl vmexport download my-export --format=oci --output=fedora-template.oci.tar
```

该 OCI 制品包含模板 spec 以及所有关联的卷数据。

### 导入模板 {#importing-a-template}

原生 OCI 导入计划在未来的发行版（>= v1.10.0）中提供。在此期间，可以使用 [`hack/oci-import.sh`](https://github.com/kubevirt/kubevirt/blob/main/hack/oci-import.sh) 脚本来导入已导出的模板。
