# 去中心化实时迁移 {#decentralized-live-migration}

去中心化实时迁移是[存储实时迁移](../storage/volume_migration.md)的一种变体，允许你在命名空间之间，甚至在不同的 Kubernetes 集群之间迁移虚拟机。这在多种场景下都很有用，包括：

1. **均衡工作负载：** 在集群之间分布虚拟机有助于优化资源利用率。如果一个集群负载很高，而另一个集群处于空闲，重新均衡可以显著提高运维效率。
2. **便于维护：** 对于拥有多个集群的环境，虚拟机迁移可以让维护无缝进行。你可以把虚拟机移出计划升级或关闭的集群，从而保证服务零停机。
3. **加快恢复：** 备份厂商提供的即时恢复能力，尤其是与命名空间迁移结合时，可以大幅缩短恢复时间。虚拟机可以快速恢复到临时位置，然后再迁移到原来的命名空间和存储。

本文假定存在一个编排器，用于管理资源的创建和删除。虽然未来的 KubeVirt 版本会为去中心化实时迁移纳入编排器，但这里给出的示例描述的是编排器会执行的底层步骤，而不是直接使用某个编排器。

### 去中心化实时迁移功能门控 {#decentralized-live-migration-feature-gate}

必须在功能门控中启用去中心化实时迁移支持，该能力才可用。KubeVirt CR 中的[功能门控](../cluster_admin/activating_feature_gates.md#how-to-activate-a-feature-gate)字段必须通过添加 `DecentralizedLiveMigration` 来扩展。

## 架构 {#architecture}

**计算实时迁移**

计算实时迁移的特征是只有一个 `VirtualMachineInstance` 资源和一个 `VirtualMachineInstanceMigrations` 资源。迁移通过创建 `VirtualMachineInstanceMigration` 资源来发起，协调则经由 VirtualMachineInstance 的状态完成，如图 1 所示。

图 1：
![图 1](live-migration.png)

**存储实时迁移**

存储实时迁移是计算实时迁移的一种变体，会通过迁移网络复制虚拟机的状态和磁盘内容。有关如何发起存储实时迁移的更多信息，请参阅这份[文档](https://kubevirt.io/user-guide/storage/volume_migration/)。

**去中心化实时迁移**

去中心化实时迁移是存储实时迁移的一种变体。它们涉及两个 `VirtualMachineInstances` 和两个 `VirtualMachineInstanceMigrations`。与存储实时迁移类似，磁盘内容会通过网络复制到接收方虚拟机。不过，关键区别在于接收方虚拟机拥有一个完全独立的 `VirtualMachineInstance`。为了协调迁移，源和目标 'VirtualMachineInstances` 的状态必须保持同步，如图 2 所示。

图 2：
![图 2](decentralized-live-migration.png)

专用的同步控制器促进源和目标 `VirtualMachineInstances` 之间的通信，如图 2 所示。这种设计使得为 `compute` 迁移开发的大部分现有实时迁移机制都可以被复用。

## 所需资源 {#required-resources}

要在命名空间或集群之间迁移虚拟机，必须同时创建目标 `VirtualMachine` 资源和目标 `VirtualMachineInstanceMigration` 资源。仅使用 `VirtualMachineInstance` 资源无法完成去中心化实时迁移。

目标集群/命名空间必须具备虚拟机运行所需的全部资源。有些资源在目标上可能并不完全一致。例如，目标上可用的存储类可能与源不同。由于去中心化实时迁移属于存储实时迁移，因此可以在目标上更改存储类并成功完成迁移。网络也同样如此，不过连通性可能会因网络配置而中断。

此外，辅助资源，例如包含 SSH 密钥的 `Secret`，或带有配置数据的 `ConfigMap`，也必须被复制并在目标集群上可用。

## 目标虚拟机 {#target-virtual-machine}

为了构造目标 `VirtualMachine` 资源，合理的做法是沿用源 `VirtualMachine` 资源的大部分字段，但替换网络和磁盘，以匹配目标集群/命名空间上可用的资源。`runStrategy` 必须设置为 `WaitAsReceiver`，以此向 KubeVirt 表明这是一台特殊的目标虚拟机，并且在迁移完成之前，标准虚拟机命令（启动/停止）不会起作用。请考虑下面这台运行在集群 1 上的源虚拟机。

源虚拟机：

```yaml
apiVersion: kubevirt.io/v1
kind: VirtualMachine
metadata:
  name: example-decentralized-lm-vm
spec:
  runStrategy: Always
  template:
    metadata:
  	  creationTimestamp: null
    spec:
  	  architecture: amd64
  	  domain:
    	    devices:
    	    autoattachGraphicsDevice: false
    	  disks:
    	  - disk:
             bus: virtio
      	    name: disk0
    	  interfaces:
    	  - masquerade: {}
      	    name: default
    	  machine:
      	    type: q35
    	  resources:
      	    requests:
             memory: 128Mi
    networks:
    - name: default
  	  pod: {}
    volumes:
  	  - dataVolume:
      	    name: example-decentralized-lm-vm-dv
    	  name: disk0
```

名为 'example-decentralized-lm-vm' 的 `VirtualMachine` 已创建，它使用名为 'example-decentralized-lm-vm-dv' 的 `DataVolume`。此 `DataVolume` 可以包含任意种类的磁盘镜像，例如 fedora。由于 `runStrategy` 被设置为 `Always`，一旦将该 YAML 应用到集群，虚拟机就会始终启动。

在目标集群内，部署目标虚拟机所需的全部资源都必须可用。为了存放源磁盘的内容，会创建一个空白的 `DataVolume`。虽然这不是强制要求，但建议保留原来的名称。

目标 DataVolume：

```yaml
apiVersion: cdi.kubevirt.io/v1beta1
kind: DataVolume
metadata:
  annotations:
	cdi.kubevirt.io/storage.usePopulator: "true"
  name: example-decentralized-lm-vm-dv
spec:
  source:
	blank: {}
  storage:
    storageClassName: <other storage class>
    resources:
      requests:
        storage: 10Gi
```

目标虚拟机：

```yaml
apiVersion: kubevirt.io/v1
kind: VirtualMachine
metadata:
  annotations:
	kubevirt.io/restore-run-strategy: Always
  name: example-decentralized-lm-vm
spec:
  runStrategy: WaitAsReceiver
  template:
	metadata:
  	  creationTimestamp: null
	spec:
  	  architecture: amd64
  	  domain:
    	    devices:
    	    autoattachGraphicsDevice: false
    	  disks:
    	  - disk:
             bus: virtio
      	    name: disk0
    	  interfaces:
    	  - masquerade: {}
      	    name: default
    	  machine:
      	    type: q35
    	  resources:
      	    requests:
             memory: 128Mi
	networks:
	- name: default
  	  pod: {}
	volumes:
  	  - dataVolume:
      	    name: example-decentralized-lm-vm-dv
    	  name: disk0
```

源虚拟机定义与目标虚拟机定义有两个关键差异：

* 目标的 `runStrategy` 被设置为 `WaitAsReceiver`。
* 目标虚拟机包含一个注解，用于指定迁移之后的 `runStrategy`，它应当与源相同，在本例中为 `Always`。

除虚拟机磁盘之外的任何额外资源，例如 `Secret` 或 `Configmap`，也必须在目标集群上创建。创建 `VirtualMachine` 资源之后，KubeVirt 会生成对应的接收方 `VirtualMachineInstace`，随后可以在目标迁移资源中引用它。

## 迁移资源 {#migration-resources}

目标虚拟机创建之后，要启动迁移，你需要创建“源”和“目标” `VirtualMachineInstanceMigration`。源迁移资源和目标迁移资源都创建之后，迁移就会开始。

### 目标迁移资源 {#target-migration-resource}

目标迁移资源与标准的 `VirtualMachineInstanceMigration` 资源非常相似，另外增加了若干字段，用来表明它是接收方的去中心化实时迁移资源。

```yaml
apiVersion: kubevirt.io/v1
kind: VirtualMachineInstanceMigration
metadata:
  name: example-target-migration
  namespace: <same namespace as target vm>
spec:
  receive:
	migrationID: <unique identifier>
  vmiName: <name of the vmi that is created from target vm>
```

`receive` 字段表明这是去中心化迁移的目标资源。`migrationID` 字符串唯一标识“源”和“目标” `VirtualMachineInstanceMigrations`。然后 `vmiName` 把 `migrationID` 映射到相应的 `VirtualMachineInstance`。

### 源迁移资源 {#source-migration-resource}

源迁移资源现在包含一个额外字段，该字段带有两个子字段：

1. `migrationID`（与目标迁移资源相同）
2. `connectURL`。此连接 URL 用于连接到目标集群上的同步控制器。

获取同步控制器地址有两种方法：

1. **访问目标集群上的 KubeVirt CR：** 如果你可以访问 KubeVirt CR，一个名为 `synchronizationAddresses` 的新字段会提供可用于 `connectURL` 的地址/端口组合列表。
2. **创建目标迁移资源之后：** 如果你无法访问 KubeVirt CR，那么一旦创建了目标迁移资源，其状态中就会包含 `synchronizationAddresses` 字段，其中是同一份地址/端口组合列表。

这两种方式的关键区别在于，方法 1 允许你提前获得连接 URL，从而可以与目标迁移资源同时创建源迁移资源，或者在目标迁移资源之前创建。

获得连接 URL 之后，应把它的值放入源迁移资源的 `connectURL` 字段。

```yaml
apiVersion: kubevirt.io/v1
kind: VirtualMachineInstanceMigration
metadata:
  name: example-source-migration
spec:
  sendTo:
	connectURL: "<synchronization address>"
	migrationID: <unique identifier, same as target migration resource migrationID>
  vmiName: <name of the source vmi>
```

## 配置 KubeVirt CA 以进行跨集群实时迁移 {#configuring-kubevirt-ca-with-cross-cluster-live-migration}

在两个集群之间迁移时，每个集群都有自己的 KubeVirt 安装。每个 KubeVirt 安装都会创建并使用自己的证书颁发机构（CA）。此 CA 用于在迁移期间保护同步控制器与 virt-handler 之间的通信。为了让每个集群信任另一个集群，必须在集群之间交换 CA 公钥。

CA 公钥存储在 KubeVirt 安装所在命名空间中的 `ConfigMap` 里（默认为 `kubevirt`）。此 `ConfigMap` 名为 `kubevirt-ca`。下面是此 `ConfigMap` 的一个示例：

```yaml
apiVersion: v1
data:
  ca-bundle: |
    -----BEGIN CERTIFICATE-----
...
    -----END CERTIFICATE-----
    -----BEGIN CERTIFICATE-----
...
    -----END CERTIFICATE-----
kind: ConfigMap
metadata:
  name: kubevirt-ca
  namespace: kubevirt
```

在同一命名空间中，还会有另一个 `ConfigMap`，名为 `kubevirt-external-ca`，其 data 中包含 `ca-bundle` 字段。为了让 Cluster1 信任来自 Cluster2 的 CA，请从 Cluster2 的 `kubevirt-ca` `ConfigMap` 中取出 ca-bundle，并把 `ca-bundle` 复制到 Cluster1 的 `kubevirt-external-ca` 中，反之亦然。更新 `kubevirt-external-ca` 之后，它会自动清空 `kubevirt-external-ca` 的内容，但会把这些内容复制进 `kubevirt-ca` `ConfigMap`。在复制之前，它会移除无效、重复和过期的密钥。

根据密钥有效期的长短，密钥过期后你必须再次执行此过程。默认情况下，KubeVirt CA 的有效期为 2 天。

## 取消去中心化实时迁移 {#cancelling-decentralized-live-migration}

要取消正在进行的去中心化实时迁移，请从源和目标集群/命名空间中删除相应的迁移资源。此操作会通知 KubeVirt 取消迁移。去中心化实时迁移的取消过程遵循与标准迁移取消相同的原则。可能会有一段短暂的延迟，在此期间迁移有可能已经完成，此时取消不会产生任何效果。

## 迁移失败 {#migration-failure}

与会自动重试直到完成的标准实时迁移不同，失败的去中心化实时迁移不会自动重试。去中心化实时迁移失败时，源虚拟机继续在源上运行，目标虚拟机进入失败状态。要重新尝试迁移，你必须手动删除源和目标 `VirtualMachineInstanceMigration` 资源，以及目标虚拟机。

## 限制 {#limitations}

由于去中心化实时迁移是存储实时迁移的一种变体，它具有相同的限制：

* 不支持可共享磁盘，因为在存在多个写入者时，我们无法保证数据一致性。
* 不支持 [`filesystem` 设备](../storage/disks_and_volumes.md/#filesystems)，因为 virtio-fs 目前不支持实时迁移。
* 不支持 [LUN 磁盘](../storage/disks_and_volumes.md/#lun)，因为在迁移时我们无法保证目标同样是 LUN。
* 源和目标上的 VM 状态 PVC 都是 RWX。由于一个[缺陷](https://issues.redhat.com/browse/RHEL-108915)，它不会复制 vTPM 设备数据，因为系统假定源和目标上的共享存储就是同一存储。
