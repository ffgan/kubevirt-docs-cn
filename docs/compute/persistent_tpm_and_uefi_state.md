# 持久化 TPM 和 UEFI 状态 {#persistent-tpm-and-uefi-state}

**功能状态：** KubeVirt v1.0.0

对于 TPM 和 UEFI，libvirt 支持将虚拟机创建的数据以文件形式持久保存在虚拟化宿主机上。
在 KubeVirt 中，虚拟化宿主机就是 virt-launcher Pod，它是临时的（在 VM 启动时创建，在 VM 停止时销毁）。
从 v1.0.0 起，KubeVirt 支持使用 PVC 来持久保存这些文件。KubeVirt 通常将该存储区域称为“后端存储”。

## 后端存储 {#backend-storage}

KubeVirt 会自动为需要它的 VM 创建后端存储 PVC。但是，要持久保存 TPM 和 UEFI 状态，管理员必须首先启用 `VMPersistentState` 功能门控。可以使用 KubeVirt CR 配置选项 `vmStateStorageClass` 手动指定存储类，否则将使用默认存储类。

以下是同时设置这两者的 KubeVirt CR 示例：
```yaml
apiVersion: kubevirt.io/v1
kind: KubeVirt
spec:
  configuration:
    vmStateStorageClass: "nfs-csi"
    developerConfiguration:
      featureGates:
      - VMPersistentState
```

### 注意： {#notes}

#### 快照与恢复兼容性 {#snapshot-and-restore-compatibility}

从 KubeVirt v1.4 起，使用后端状态存储的 VirtualMachine 同时支持在线快照和离线快照。只要恢复到同一个 VirtualMachine，这些快照就可以成功恢复。

目前不支持将快照恢复到另一个 VirtualMachine（VM 克隆）。

计划在未来的版本中重新审视对带有后端存储的、基于快照的 VM 克隆的支持。

## 带持久状态的 TPM {#tpm-with-persistent-state}

从 KubeVirt v0.53.0 起，可以向 VM 添加 TPM 设备（只需 `tpm: {}`）。但是，存储在其中的数据不会在多次重启之间保持。
持久化支持在 v1.0.0 中加入，使用一个简单的 `persistent` 布尔参数，该参数默认为 false，以保留先前的行为。
当然，在向 VM 添加持久 TPM 之前，必须先配置后端存储。请参见上文。
以下是包含持久 TPM 的 VM 定义片段：
```yaml
apiVersion: kubevirt.io/v1
kind: VirtualMachine
metadata:
  name: vm
spec:
  template:
    spec:
      domain:
        devices:
          tpm:
            persistent: true
```


为了成功创建持久 tpm 卷，你必须确保存储类和[存储配置文件](https://github.com/kubevirt/containerized-data-importer/blob/main/doc/storageprofile.md)配置正确。
如果某一访问模式的约束条件之一成立，持久 tpm 卷将按下面的访问模式创建。

* RWX（ReadWriteMany）：
    * 相应的存储配置文件拥有任意 `claimPropertySet`，它位于 `claimPropertySets` 中，并且具有 Filesystem 卷模式和 RWX 访问模式。
    * kubevirt 集群配置已设置 `VMStateStorageClass`，且存储配置文件不存在。
    * kubevirt 集群配置已设置 `VMStateStorageClass`，且存储配置文件存在，但 `claimPropertySets` 为空列表。

* RWO（ReadWriteOnce）：
    * 相应的存储配置文件具有 `claimPropertySets`，其中所有 `claimPropertySet` 都位于 `claimPropertySets` 之中，并且具有 Filesystem 卷模式和 RWO 访问模式，但没有 RWX。
    * kubevirt 集群配置**未设置** `VMStateStorageClass`，且存储配置文件不存在。
    * kubevirt 集群配置**未设置** `VMStateStorageClass`，且存储配置文件存在，但 `claimPropertySets` 为空列表。

### 用途 {#uses}
- Microsoft Windows 11 安装程序要求存在 TPM 设备，即使它并不使用该设备。不过，这种情况下并不需要持久化。
- 某些磁盘加密软件具有可选或强制的 TPM 支持。例如，Bitlocker 需要持久的 TPM 设备。

### 注意 {#notes_1}
- 暴露给虚拟机的 TPM 设备是完全模拟的（vTPM）。工作节点不需要具备 TPM 设备。
- 启用 TPM 持久化后，使用的是 `tpm-crb` 模型（非持久 vTPM 则使用 `tpm-tis`）
- 虚拟 TPM 无法提供与物理 TPM 相同的安全保证。

## 带持久 VARS 的 EFI {#efi-with-persistent-vars}

EFI 支持由 libvirt 使用 OVMF 来处理。OVMF 数据通常由 2 个文件组成，即 CODE 和 VARS。VARS 是可以存储来自客户机的持久数据的位置。
在 VM 上启用 EFI 持久化后，VARS 文件将持久保存在后端存储中。
当然，在 VM 上启用 EFI 持久化之前，必须先配置后端存储。请参见上文。
以下是包含持久 EFI 的 VM 定义片段：
```yaml
apiVersion: kubevirt.io/v1
kind: VirtualMachine
metadata:
  name: vm
spec:
  template:
    spec:
      domain:
        firmware:
          bootloader:
            efi:
              persistent: true
```

### 用途 {#uses_1}
- 保留用户创建的安全启动证书。
- 保留 EFI 固件设置，例如语言或显示分辨率。

### 注意 {#notes_2}
- 启动项和启动顺序可能会，而且很可能会被 libvirt 覆盖。这是为了满足 VM 规格。不要期望手动更改的启动设置会持久保留。
