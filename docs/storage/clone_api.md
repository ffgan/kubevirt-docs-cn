# 克隆 API {#clone-api}

`clone.kubevirt.io` API 组定义了用于克隆 KubeVirt 对象的资源。

此 API 有两个主要用例：
* 克隆虚拟机。
* 创建将用作新建 VM 模板的“黄金快照”
（参见下方[将克隆用作“黄金 VM 镜像”](#using-clones-as-a-golden-vm-image)）。

请注意，克隆 API 的版本为 `v1beta1`。这意味着此 API 尚未完全稳定，
未来 API 可能会发生变化。

## 先决条件 {#prerequisites}

### 快照 / 恢复 {#snapshot-restore}

在底层，克隆 API 依赖快照与恢复 API。因此，要能够使用克隆 API，
请参阅[快照与恢复的先决条件](../storage/snapshot_restore_api.md#prerequisites)。

### 快照功能门控 {#snapshot-feature-gate}

目前，克隆 API 受 Snapshot 功能门控保护。必须扩展 KubeVirt CR 中的
[功能门控](../cluster_admin/activating_feature_gates.md#how-to-activate-a-feature-gate)
字段，向其中添加 `Snapshot`。

## 克隆对象 {#the-clone-object}

首先，如上所述，克隆 API 在底层依赖快照与恢复 API。因此，
查阅[快照与恢复](../storage/snapshot_restore_api.md)用户指南页面以了解更多信息可能会有所帮助。

### VirtualMachineClone 对象概述 {#virtualmachineclone-object-overview}

要启动克隆，需要在集群上创建 `VirtualMachineClone` 对象（CRD）。此类对象的示例如下：
```yaml
kind: VirtualMachineClone
apiVersion: "clone.kubevirt.io/v1beta1"
metadata:
  name: testclone

spec:
  # source & target definitions
  source:
    apiGroup: kubevirt.io
    kind: VirtualMachine
    name: vm-cirros
  target:
    apiGroup: kubevirt.io
    kind: VirtualMachine
    name: vm-clone-target

  # labels & annotations definitions
  labelFilters:
    - "*"
    - "!someKey/*"
  annotationFilters:
    - "anotherKey/*"

  # template labels & annotations definitions
  template:
    labelFilters:
      - "*"
      - "!someKey/*"
    annotationFilters:
      - "anotherKey/*"

  # other identity stripping specs:
  newMacAddresses:
    interfaceName: "00-11-22"
  newSMBiosSerial: "new-serial"
```

下一节将逐一说明这些不同的设置。

#### 源与目标 {#source-target}

源和目标指明源/目标的 API 组、kind 和名称。几点重要说明：

* 目前，唯一支持的 kind 是 `VirtualMachine`（属于 `kubevirt.io` API 组）和 `VirtualMachineSnapshot`（
属于 `snapshot.kubevirt.io` API 组），但预计未来会支持更多类型。

* 目标名称是**可选的**。如果未指定，克隆控制器会自动为目标生成名称。

* 目标与源必须位于同一命名空间。

#### 标签与注解过滤器 {#label-annotation-filters}

这些 spec 字段用于确定哪些标签 / 注解会被复制到目标，或被剥离。

过滤器是一个字符串列表。每个字符串表示可能存在于源上的一个键。每一个与这些值之一匹配的源键
都会被复制到克隆目标。此外，可以使用特殊的、类似正则表达式的字符：

* **通配符（\*）** 可用于匹配任意内容。通配符只能用在过滤器的**末尾**。
  * 以下过滤器有效：
    * "*"
    * "some/key*"
  * 以下过滤器无效：
    * "some/*/key"
    * "*/key"
* **否定字符（!）** 可用于避免匹配某些键。否定字符只能用在过滤器的**开头**。
  请注意，否定字符和通配符可以一起使用。
  * 以下过滤器有效：
    * "!some/key"
    * "!some/*"
  * 以下过滤器无效：
    * "key!"
    * "some/!key"

设置标签 / 注解过滤器是**可选的**。如果未设置，默认会复制所有标签 / 注解。

#### 模板标签与模板注解过滤器 {#template-label-template-annotation-filters}

某些网络 CNI（例如 Kube-OVN 或 OVN-Kubernetes）会把网络信息注入 VM 的注解。从目标 VM 克隆 VM 时，克隆得到的 VM 将使用同一网络。要避免这种情况，可以使用模板标签和注解过滤器。

#### newMacAddresses {#newmacaddresses}

此字段用于显式替换某些接口的 MAC 地址。该字段是字符串到字符串的映射；
键表示接口名称，值表示克隆目标的新 MAC 地址。

此字段是可选的。默认情况下，所有 MAC 地址都会被剥离。这适用于集群中部署了 kube-mac-pool 的情况，
它会自动为目标分配一个新的有效 MAC 地址。

#### newSMBiosSerial {#newsmbiosserial}

此字段用于显式设置目标的 SMBios 序列号。

此字段是可选的。默认情况下，目标会拥有一个基于 VM 名称自动生成的序列号。

### JSON 补丁 {#json-patches}

有时需要在克隆之前对 VM 的 spec 做比过滤更进一步的修改（例如为 CNI 修改或添加注解）。

可以对克隆 VM 的 `specs` 和 `labels/annotations` 应用 JSON 补丁，方法是使用 `patches` 参数：

```yaml
apiVersion: snapshot.kubevirt.io/v1beta1
kind: VirtualMachineClone
metadata:
  name: cone
spec:
  source:
    apiGroup: kubevirt.io
    kind: VirtualMachine
    name: vm-cirros
  target:
    apiGroup: kubevirt.io
    kind: VirtualMachine
    name: vm-clone-target
  patches:
    - '{"op": "add", "path": "/metadata/labels/test", "value": "something"}'
    - '{"op": "replace", "path": "/spec/template/metadata/labels/example", "value": "replaced-value"}'
```

**请牢记，补丁必须谨慎应用。** 某些字段可能用于引用其他资源，也可能被其他资源用作目标。

补丁与通过模板标签/注解过滤器以及 MAC 地址/SMBIOS 字段所做的修改是累积的（并且具有更高优先级）。

### 卷名称策略 {#volume-name-policy}

`volumeNamePolicy` 字段控制克隆操作期间卷的命名方式。在底层，这会被转换为 `volumeRestorePolicy`，位于克隆控制器所创建的 `VirtualMachineRestore` 上。

可用策略如下：

- **RandomizeNames**（默认策略）：克隆 PVC 的名称随机生成。这可以保证不会发生冲突，但名称不可预测。
- **PrefixTargetName**：克隆 PVC 的名称通过在卷名前加上目标 VM 名称来生成：`{targetVMName}-{volumeName}`。这样可以提供可预测的确定性命名，同时在克隆到不同目标时避免冲突。名称会被截断为 63 个字符，以符合 DNS 标签长度限制。

```yaml
kind: VirtualMachineClone
apiVersion: "clone.kubevirt.io/v1beta1"
metadata:
  name: testclone
spec:
  source:
    apiGroup: kubevirt.io
    kind: VirtualMachine
    name: vm-cirros
  target:
    apiGroup: kubevirt.io
    kind: VirtualMachine
    name: vm-clone-target
  volumeNamePolicy: PrefixTargetName
```

在此示例中，源 VM 上名为 `disk0` 的卷会产生名为 `vm-clone-target-disk0` 的克隆 PVC。

### 创建 VirtualMachineClone 对象 {#creating-a-virtualmachineclone-object}

克隆清单准备好之后，就可以创建它：
```bash
kubectl create -f clone.yaml
```

要等待克隆完成，请执行：
```bash
kubectl wait vmclone testclone --for condition=Ready
```

可以在克隆的 status 中查看克隆所处的阶段。它可以是以下之一：

* SnapshotInProgress

* CreatingTargetVM

* RestoreInProgress

* Succeeded

* Failed

* Unknown

克隆完成后，可以检查目标：
```bash
kubectl get vm vm-clone-target -o yaml
```

### 将克隆用作“黄金 VM 镜像” {#using-clones-as-a-golden-vm-image}

当源的 kind 为 `VirtualMachineSnapshot` 时，使用克隆 API 可以完成的一件很有用的事情
是创建“黄金 VM 镜像”（亦称模板 / 书签 VM 等）。换句话说，可以使用以下
工作流：

**创建黄金镜像**

* 创建 VM

* 准备“黄金 VM”环境。这在不同上下文中可以有不同含义。
例如，添加某些用户、安装数据库、以某种方式配置它，等等。

* 为该 VM 创建快照

* 可选：删除该 VM

然后，可以根据需要多次复制此“黄金镜像”。要从快照实例化 VM：

* 创建一个克隆对象，其源指向先前创建的快照
* 创建所需数量的 VM
