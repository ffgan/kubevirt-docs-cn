# 热插拔卷 {#hotplug-volumes}

KubeVirt 支持向正在运行的虚拟机热插拔持久卷。这些卷可以表示为磁盘、LUN 或 CD-ROM。

## 启用热插拔卷支持 {#enabling-hotplug-volume-support}

必须在功能门控中启用热插拔卷支持，才能使用该功能。必须扩展 KubeVirt CR 中的
[功能门控](../cluster_admin/activating_feature_gates.md#how-to-activate-a-feature-gate)
字段，向其中添加 `DeclarativeHotplugVolumes`。

> **_NOTE:_**  `DeclarativeHotplugVolumes` 与已弃用的 `HotplugVolumes` 功能门控不兼容。如果两者都被声明，在 `HotplugVolumes` 被停用之前，`HotplugVolumes` 将具有优先权。

## 支持的磁盘总线 {#supported-disk-busses}

|       |  sata  | virtio |  scsi  |
|-------|--------|--------|--------|
| cdrom |    X   |        |        |
| disk  |        |    X   |    X   |
| lun   |        |        |    X   |

如果要并发热插拔大量磁盘，应使用 scsi 总线。可用的 VirtIO 端口数量有限。scsi 总线在[性能上非常接近](https://mpolednik.github.io/2017/01/23/virtio-blk-vs-virtio-scsi/) virtio。

## 可用的 VirtIO 端口 {#available-virtio-ports}

下表列出了可用于热插拔磁盘的 VirtIO 端口的最小数量。

| 内存 | 端口 |
|:------:|:-----:|
|  <= 2G |   3   |
|  > 2G  |   6   |

> **_NOTE:_**  每增加一个[热插拔网络接口](../network/hotplug/interfaces.md)，可用的 VirtIO 端口就会减少

## 声明式 API {#declarative-api}

可以以声明式、与 GitOps 兼容的方式，为 VirtualMachine 定义热插拔 [DataVolume](./disks_and_volumes.md/#persistentvolumeclaim) 和 [PersistentVolumeClaim](./disks_and_volumes.md/#persistentvolumeclaim) 卷。

### 虚拟机定义 {#virtual-machine-definition}

下面的 VirtualMachine 没有持久磁盘，并有一个空的 CD-ROM。

```yaml
apiVersion: kubevirt.io/v1
kind: VirtualMachine
metadata:
  name: testvm
spec:
  runStrategy: Always
  template:
    spec:
      domain:
        devices:
          disks:
          - cdrom:
              bus: sata
            name: cdrom
          - disk:
              bus: virtio
            name: disk0
        resources:
          requests:
            memory: 128Mi
      volumes:
      - containerDisk:
          image: quay.io/kubevirt/alpine-container-disk-demo:v1.6.0
        name: disk0
```

### 测试卷 {#test-volume}

下面的 yaml 将创建一个卷，其中包含 VirtualMachine 根磁盘的副本

```yaml
apiVersion: cdi.kubevirt.io/v1beta1
kind: DataVolume
metadata:
  name: hotplug-disk
spec:
  source:
    registry:
      url: "docker://quay.io/kubevirt/alpine-container-disk-demo:v1.6.0"
  storage:
    resources:
      requests:
        storage: 300Mi
```

### 插入 CD-ROM {#inject-cd-rom}

要把 CD-ROM 插入正在运行的 VirtualMachine，必须在 VM 启动时声明一个 `cdrom` 类型的磁盘。此后任何时候，都可以把热插拔卷添加到 VirtualMachine 的 `spec.template.spec.volumes` 部分。

```yaml
apiVersion: kubevirt.io/v1
kind: VirtualMachine
metadata:
  name: testvm
spec:
  runStrategy: Always
  template:
    spec:
      domain:
        devices:
          disks:
          - cdrom:
              bus: sata
            name: cdrom
          - disk:
              bus: virtio
            name: disk0
        resources:
          requests:
            memory: 128Mi
      volumes:
      - containerDisk:
          image: quay.io/kubevirt/alpine-container-disk-demo:v1.6.0
        name: disk0
      - dataVolume:
          name: hotplug-disk
          hotpluggable: true
        name: cdrom
```

### 弹出 CD-ROM {#eject-cd-rom}

移除 `cdrom` 卷，将其从 VirtualMachine 的 `spec.template.spec.volumes` 部分删除。

### 热插拔磁盘/LUN {#hotplug-disklun}

要把磁盘/LUN 热插拔到正在运行的 VirtualMachine，必须在 VM 的 `spec.template.spec.domain.devices.disks` 部分添加新的磁盘/LUN，并在 `spec.template.spec.volumes` 中添加新卷。

```yaml
apiVersion: kubevirt.io/v1
kind: VirtualMachine
metadata:
  name: testvm
spec:
  runStrategy: Always
  template:
    spec:
      domain:
        devices:
          disks:
          - cdrom:
              bus: sata
            name: cdrom
          - disk:
              bus: virtio
            name: disk0
          - disk:
              bus: virtio
            name: disk1
        resources:
          requests:
            memory: 128Mi
      volumes:
      - containerDisk:
          image: quay.io/kubevirt/alpine-container-disk-demo:v1.6.0
        name: disk0
      - dataVolume:
          name: hotplug-disk
          hotpluggable: true
        name: disk1
```

### 拔出磁盘/LUN {#unplug-disklun}

把 `disk1` 从 VirtualMachine 的 `spec.template.spec.domain.devices.disks` 部分移除，并移除 `disk1`（来自 `spec.template.spec.volumes`）

## virtctl 支持 {#virtctl-support}

也可以使用 virtctl 向 VirtualMachine 添加卷。virtctl 也是把卷直接热插拔到 VirtualMachineInstance 的唯一方式。virtctl 不支持 CD-ROM 的插入/弹出。

> **_NOTE:_** `--persist` 标志用于 `virtctl addvolume` 和 `virtctl removevolume`，已被弃用，并将在未来的发行版中移除。这些命令的默认行为将是始终把热插拔变更同时持久化到 VirtualMachineInstance 和所属的 VirtualMachine（如果存在）

### 向 VirtualMachine 添加磁盘 {#add-a-disk-to-a-virtualmachine}

```bash
$ virtctl addvolume testvm --volume-name=hotplug-disk
```

### 向 VirtualMachineInstance 添加磁盘 {#add-a-disk-to-a-virtualmachineinstance}

只有当 VirtualMachineInstance（VMI）独立于更高层资源创建时，才能向其热插拔磁盘。例如，如果该 VMI 由 VirtualMachine 拥有，此操作将失败。

```bash
$ virtctl addvolume testvmi --volume-name=hotplug-disk
```

### 向 VirtualMachine 添加 LUN {#add-a-lun-to-a-virtualmachine}

```bash
$ virtctl addvolume testvm --volume-name=hotplug-disk --disk-type=lun
```

### 其他 `addvolume` 参数 {#additional-addvolume-args}

`--bus <bus type>` 用于指定磁盘/LUN 总线

`--cache <cache type>` 用于指定磁盘缓存类型（default|none|writethrough|writeback|directsync）

`--serial <serial>` 用于指定磁盘序列号（在客户机中识别特定磁盘的规范方式）

### 从 VirtualMachine 拔出磁盘/LUN {#unplug-a-disklun-from-a-virtualmachine}

```bash
$ virtctl removevolume testvm --volume-name=hotplug-disk
```

### VolumeStatus {#volumestatus}

VMI 对象有一个 `status.VolumeStatus` 字段。这是一个数组，包含每个磁盘，无论是否为热插拔。例如：

```yaml
volumeStatus:
- containerDiskVolume:
    checksum: 538764798
  name: disk0
  target: vda
- hotplugVolume:
    attachPodName: hp-volume-phzmq
    attachPodUID: 3eea974f-85ad-4a58-97a2-46c463f9b639
  message: Successfully attach hotplugged volume blank-disk to VM
  name: blank-disk
  persistentVolumeClaimInfo:
    accessModes:
    - ReadWriteMany
    capacity:
      storage: 300Mi
    claimName: blank-disk
    filesystemOverhead: "0"
    requests:
      storage: "314572800"
    volumeMode: Block
  phase: Ready
  reason: VolumeReady
  target: sda
```

在此示例中，`vda` 是包含操作系统的容器磁盘。可以看到，它只包含在把它们分配给 VM 时使用的名称和 target。target 是在指定磁盘时传递给 QEMU 的值。target 值对 VM 是唯一的，并且 *并不* 代表客户机内部的命名。例如，对于 Windows 客户机操作系统，target 没有意义。热插拔卷也是如此。target 只是一个供 QEMU 使用的唯一标识符，在客户机内部，磁盘可以被分配不同的名称。

`hotplugVolume` 具有常规卷状态所没有的一些额外信息。`attachPodName` 是用于把卷挂接到 VMI 正在运行的节点上的 Pod 的名称。如果此 Pod 被删除，它也会停止该 VMI，因为我们无法保证该卷会保持挂接到该节点。其他字段类似于条件，并指示热插拔过程的状态。一旦卷就绪，VM 就可以使用它。
