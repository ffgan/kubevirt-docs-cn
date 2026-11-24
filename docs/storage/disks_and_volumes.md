# 文件系统、磁盘和卷 {#filesystems-disks-and-volumes}

让集群中的持久化存储（**卷**）可供 VM 访问，这一过程由三个部分组成。首先，在 `spec.volumes` 中指定卷。其次，通过在 `spec.domain.devices.disks` 中指定磁盘，把磁盘添加到 VM。最后，按名称把对所指定卷的引用添加到磁盘规格中。

## 磁盘 {#disks}

与所有其他 VMI 设备一样，`spec.domain.devices.disks` 元素具有必填的 `name`，并且磁盘的 `name` 必须引用某个卷的 `name`，该卷位于 `spec.volumes` 中。

磁盘可以通过四种不同类型供访问：

-   [**lun**](#lun)

-   [**disk**](#disk)

-   [**cdrom**](#cdrom)

-   [**fileystems**](#filesystems)

所有可用的配置选项见[磁盘 API 参考](https://kubevirt.github.io/api-reference/master/definitions.html#_v1_disk)。

所有类型都允许指定 `bus` 属性。`bus` 属性决定磁盘如何呈现给客户机操作系统。

### lun {#lun}

`lun` 磁盘会把卷作为 LUN 设备暴露给 VM。这允许 VM 执行任意 SCSI 命令透传。

将 `PersistentVolumeClaim`（名为 `mypvc`）作为 `lun` 设备挂接到 VM 的最小示例如下：

```yaml
metadata:
  name: testvmi-lun
apiVersion: kubevirt.io/v1
kind: VirtualMachineInstance
spec:
  domain:
    resources:
      requests:
        memory: 64M
    devices:
      disks:
      - name: mypvcdisk
        # This makes it a lun device
        lun: {}
  volumes:
    - name: mypvcdisk
      persistentVolumeClaim:
        claimName: mypvc
```

#### 持久预留 {#persistent-reservation}

可以通过 SCSI Persistent Reserve 命令预留 LUN。为了发出特权 SCSI ioctl，VM 需要在磁盘上启用持久预留：

```yaml
devices:
  disks:
  - name: mypvcdisk
    lun:
      reservation: true
```

此功能需要在 KubeVirt 配置中启用持久预留：

```yaml
configuration:
  persistentReservationConfiguration:
    enabled: true
```

> **注意：** 持久预留功能会额外部署一个与 virt-handler 一同运行的特权组件。由于此功能涉及敏感的安全操作，它默认禁用，并且需要集群管理员进行配置。

### disk {#disk}

`disk` 磁盘会把卷作为普通磁盘暴露给 VM。

将 `PersistentVolumeClaim`（名为 `mypvc`）作为 `disk` 设备挂接到 VM 的最小示例如下：

```yaml
metadata:
  name: testvmi-disk
apiVersion: kubevirt.io/v1
kind: VirtualMachineInstance
spec:
  domain:
    resources:
      requests:
        memory: 64M
    devices:
      disks:
      - name: mypvcdisk
        # This makes it a disk
        disk: {}
  volumes:
    - name: mypvcdisk
      persistentVolumeClaim:
        claimName: mypvc
```

可以设置磁盘的 `bus` 类型以覆盖默认值，而默认值又取决于 VM 所配置使用的芯片组：

```yaml
metadata:
  name: testvmi-disk
apiVersion: kubevirt.io/v1
kind: VirtualMachineInstance
spec:
  domain:
    resources:
      requests:
        memory: 64M
    devices:
      disks:
      - name: mypvcdisk
        # This makes it a disk
        disk:
          # This makes it exposed as /dev/vda, being the only and thus first
          # disk attached to the VM
          bus: virtio
  volumes:
    - name: mypvcdisk
      persistentVolumeClaim:
        claimName: mypvc
```

### cdrom {#cdrom}

`cdrom` 磁盘会把卷作为光驱暴露给 VM。默认情况下它是只读的。

将 `PersistentVolumeClaim`（名为 `mypvc`）作为 `cdrom` 设备挂接到 VM 的最小示例如下：

```yaml
metadata:
  name: testvmi-cdrom
apiVersion: kubevirt.io/v1
kind: VirtualMachineInstance
spec:
  domain:
    resources:
      requests:
        memory: 64M
    devices:
      disks:
      - name: mypvcdisk
        # This makes it a cdrom
        cdrom:
          # This makes the cdrom writeable
          readonly: false
          # This makes the cdrom be exposed as SATA device
          bus: sata
  volumes:
    - name: mypvcdisk
      persistentVolumeClaim:
        claimName: mypvc
```

### filesystems {#filesystems}

`filesystem` 设备会把卷作为文件系统暴露给 VM。`filesystems` 依赖 `virtiofs`，使外部文件系统对 `KubeVirt` VM 可见。有关 `virtiofs` 的更多信息见 [Virtiofs 官方网站](https://virtio-fs.gitlab.io/)。

与 `disk` 相比，`filesystems` 允许源端的变更动态反映到 VM 内部的卷中。例如，如果某个 `configMap` 通过 `filesystems` 共享，对其所做的任何更改都会反映到 VM 中。

此外，必须在 VM 内部挂载 `filesystem` 设备。可以通过 [cloudInitNoCloud](#cloudinitnocloud) 完成，也可以手动连接到 VM shell 并执行相同的命令。主要难点在于理解用于标识新文件系统的设备标签，以及如何用 `mount -t virtiofs [device tag] [path]` 命令挂载它。为此，该标签在 VM 规格的 `spec.domain.devices.filesystems.name` 中分配给文件系统。例如，如果某个 VM 规格中是 `spec.domain.devices.filesystems.name: foo`，那么在 VM 内部把文件系统挂载到 `/tmp/foo` 路径所需的命令是 `mount -t virtiofs foo /tmp/foo`：

```yaml
apiVersion: kubevirt.io/v1
kind: VirtualMachineInstance
metadata:
  name: testvmi-filesystems
spec:
  domain:
    devices:
      filesystems:
        - name: foo
          virtiofs: {}
      disks:
        - name: containerdisk
          disk:
            bus: virtio
        - name: cloudinitdisk
          disk:
            bus: virtio
    volumes:
      - containerDisk:
          image: quay.io/containerdisks/fedora:latest
        name: containerdisk
      - cloudInitNoCloud:
            userData: |-
              #cloud-config
              chpasswd:
                expire: false
              password: fedora
              user: fedora
              bootcmd:
                - "sudo mkdir /tmp/foo"
                - "sudo mount -t virtiofs foo /tmp/foo"
      - persistentVolumeClaim:
          claimName: mypvc
        name: foo
```

> **注意：** 如前所述，`filesystems` 依赖 `virtiofs`。此外，`virtiofs` 需要 Linux 内核支持才能在 VM 中工作。要检查 VM 的 Linux 镜像是否具备所需支持，可以执行以下命令：`modprobe virtiofs`。如果命令输出为 `modprobe: FATAL: Module virtiofs not found`，**则该 VM 的 Linux 镜像不支持 virtiofs**。此外，还可以检查内核版本在任意 Linux 发行版上是否达到 5.4，或在 CentOS/RHEL 上是否达到 4.18。要检查这一点，可以执行以下命令：`uname -r`。

有关 `filesystems` 的用法示例，请参阅[与 VM 共享目录](#sharing-directories-with-vms)一节。

### 错误策略 {#error-policy}

错误策略控制当磁盘读写发生 IO 错误时，虚拟化管理程序应如何表现。默认行为是停止客户机，并生成一个 Kubernetes 事件。不过，可以把该值更改为以下之一：

- `report`：错误会在客户机中报告
- `ignore`：忽略错误，但读写失败不会被察觉
- `enospace`：磁盘空间不足时出错

可以为每个 disk 或 lun 指定错误策略。

示例：

```yaml
spec:
  domain:
    devices:
      disks:
      - disk:
          bus: virtio
        name: containerdisk
        errorPolicy: "report"
      - lun:
          bus: scsi
        name: scsi-disk
        errorPolicy: "report"
```

## 卷 {#volumes}

支持的卷源有

-   [**cloudInitNoCloud**](#cloudinitnocloud)

-   [**cloudInitConfigDrive**](#cloudinitconfigdrive)

-   [**persistentVolumeClaim**](#persistentvolumeclaim)

-   [**dataVolume**](#datavolume)

-   [**ephemeral**](#ephemeral)

-   [**containerDisk**](#containerdisk)

-   [**emptyDisk**](#emptydisk)

-   [**hostDisk**](#hostdisk)

-   [**configMap**](#configmap)

-   [**secret**](#secret)

-   [**serviceAccount**](#serviceaccount)

-   [**downwardMetrics**](#downwardmetrics)

所有可用的配置选项见[卷 API 参考](https://kubevirt.github.io/api-reference/master/definitions.html#_v1_volume)。

### cloudInitNoCloud {#cloudinitnocloud}

允许把 `cloudInitNoCloud` 数据源挂接到 VM。如果 VM 包含正确的 cloud-init 配置，它会把该磁盘用作 user-data 源。

将 `Secret` 作为 cloud-init `disk` 数据源挂接的简单示例如下：

```yaml
metadata:
  name: testvmi-cloudinitnocloud
apiVersion: kubevirt.io/v1
kind: VirtualMachineInstance
spec:
  domain:
    resources:
      requests:
        memory: 64M
    devices:
      disks:
      - name: mybootdisk
        lun: {}
      - name: mynoclouddisk
        disk: {}
  volumes:
    - name: mybootdisk
      persistentVolumeClaim:
        claimName: mypvc
    - name: mynoclouddisk
      cloudInitNoCloud:
        secretRef:
          name: testsecret
```

### cloudInitConfigDrive {#cloudinitconfigdrive}

允许把 `cloudInitConfigDrive` 数据源挂接到 VM。如果 VM 包含正确的 cloud-init 配置，它会把该磁盘用作 user-data 源。

将 `Secret` 作为 cloud-init `disk` 数据源挂接的简单示例如下：

```yaml
metadata:
  name: testvmi-cloudinitconfigdrive
apiVersion: kubevirt.io/v1
kind: VirtualMachineInstance
spec:
  domain:
    resources:
      requests:
        memory: 64M
    devices:
      disks:
      - name: mybootdisk
        lun: {}
      - name: myconfigdrivedisk
        disk: {}
  volumes:
    - name: mybootdisk
      persistentVolumeClaim:
        claimName: mypvc
    - name: myconfigdrivedisk
      cloudInitConfigDrive:
        secretRef:
          name: testsecret
```

`cloudInitConfigDrive` 也可用于通过 Ignition 配置 VM。只需用 Ignition 数据替换 cloud-init 数据即可。

### persistentVolumeClaim {#persistentvolumeclaim}

允许把 `PersistentVolumeClaim` 连接到 VM 磁盘。

当 VirtualMachineInstance 的磁盘需要在 VM 终止后仍然保留时，使用 PersistentVolumeClaim。这样，VM 的数据可以在重启之间保持持久。

`PersistentVolume` 可以处于 "filesystem" 或 "block" 模式：

-   Filesystem：为了让 KubeVirt 能够使用 PersistentVolume 文件系统上已有的磁盘，该磁盘必须命名为 `disk.img`，并放置在文件系统的根路径下。目前还要求磁盘为 raw 格式。**> 重要：** `disk.img` 镜像文件需要由用户 ID `107` 拥有，以避免权限问题。

    > **注意：** 如果在启动 VM 之前尚未手动创建 `disk.img` 镜像文件，则会按 `PersistentVolumeClaim` 的大小自动创建。由于并非每个存储置备器都会提供与所请求完全一致的可用空间（例如由于文件系统开销），KubeVirt 最多容忍可用空间少 10%。这可以通过 KubeVirt CR 中的 `developerConfiguration.pvcTolerateLessSpaceUpToPercent` 值进行配置（`kubectl edit kubevirt kubevirt -n kubevirt`）。

-   Block：使用块卷来消费原始块设备。注意：需要启用 `BlockVolume` [功能门控](../cluster_admin/activating_feature_gates.md#how-to-activate-a-feature-gate)。

将 `PersistentVolumeClaim` 作为 `disk` 挂接的简单示例如下：

```yaml
metadata:
  name: testvmi-pvc
apiVersion: kubevirt.io/v1
kind: VirtualMachineInstance
spec:
  domain:
    resources:
      requests:
        memory: 64M
    devices:
      disks:
      - name: mypvcdisk
        lun: {}
  volumes:
    - name: mypvcdisk
      persistentVolumeClaim:
        claimName: mypvc
```

#### 厚置备与瘦置备 {#thick-and-thin-volume-provisioning}

稀疏化可以使磁盘变为瘦置备，也就是说，它可以把磁盘镜像内部释放的空间转换回宿主机上的空闲空间。[fstrim](https://man7.org/linux/man-pages/man8/fstrim.8.html#:~:text=fstrim%20is%20used%20on%20a,unused%20blocks%20in%20the%20filesystem) 工具可在已挂载的文件系统上使用，以丢弃文件系统未使用的块。为了能够稀疏化客户机内部的磁盘，需要在 [libvirt XML](https://libvirt.org/formatdomain.html) 中用选项 `discard=unmap` 配置该磁盘。在 KubeVirt 中，每块磁盘在传递时默认都启用了此选项。可以在客户机中运行 `lsblk -D`，检查是否支持 trim 配置，并查看每块磁盘支持的 discard 选项。

示例：

```bash
$ lsblk -D
NAME   DISC-ALN DISC-GRAN DISC-MAX DISC-ZERO
loop0         0        4K       4G         0
loop1         0       64K       4M         0
sr0           0        0B       0B         0
rbd0          0       64K       4M         0
vda         512      512B       2G         0
└─vda1        0      512B       2G         0
```

不过，在某些情况下，例如预分配，或者磁盘是厚置备时，需要禁用该选项。磁盘的 PVC 必须带有包含 `/storage.preallocation` 或 `/storage.thick-provisioned` 的注解，并设置为 true。如果卷是使用 [CDI](https://github.com/kubevirt/containerized-data-importer) 预先置备的，并且启用了[预分配](https://github.com/kubevirt/containerized-data-importer/blob/main/doc/preallocation.md)，那么 PVC 会自动加上注解 `cdi.kubevirt.io/storage.preallocation: true`，并且 discard 透传选项会被禁用。

带有用于禁用 discard 透传的注解的 PVC 定义示例：

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: pvc
  annotations:
    user.custom.annotation/storage.thick-provisioned: "true"
spec:
  storageClassName: local
  accessModes:
    - ReadWriteOnce
  volumeMode: Filesystem
  resources:
    requests:
      storage: 1Gi
```

#### 磁盘扩容 {#disk-expansion}

对于某些存储方式，Kubernetes 可能支持对正在使用的存储进行扩容（allowVolumeExpansion 特性）。KubeVirt 可以对此作出响应，使增加的存储可供虚拟机使用。

此特性会做两件事：

- 通知虚拟机大小已发生变化
- 如果磁盘是 Filesystem PVC，则把对应的文件扩展到剩余大小（同时为文件系统开销预留一些空间）。

#### 静态置备的块 PVC {#statically-provisioned-block-pvcs}

要在 VM 中直接使用来自宿主机、由外部管理的本地块设备（例如 /dev/sdb、zvol、LVM 等），需要一个支持块设备的置备器，例如 OpenEBS LocalPV。

也可以手动置备本地卷。也就是说，下面这个 PVC：

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: myblock
spec:
  storageClassName: local-device
  volumeMode: Block
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 100Gi
```

可以申领由集群管理员预先创建的 PersistentVolume，如下所示：

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: local-device
provisioner: kubernetes.io/no-provisioner
---
apiVersion: v1
kind: PersistentVolume
metadata:
  name: myblock
spec:
  volumeMode: Block
  storageClassName: local-device
  nodeAffinity:
    required:
      nodeSelectorTerms:
      - matchExpressions:
        - key: kubernetes.io/hostname
          operator: In
          values:
          - my-node
  accessModes:
    - ReadWriteOnce
  capacity:
    storage: 100Gi
  local:
    path: /dev/sdb
```

#### dataVolume {#datavolume}

DataVolume 是一种在虚拟机启动流程期间把虚拟机磁盘自动导入到 PVC 上的方式。如果不使用 DataVolume，用户必须先准备好带有磁盘镜像的 PVC，然后才能把它分配给 VM 或 VMI 清单。使用 DataVolume 时，PVC 的创建和数据导入都会代用户自动完成。

#### DataVolume 的 VM 行为 {#datavolume-vm-behavior}

可以在 VM 规格中直接定义 DataVolume，方法是把 DataVolume 添加到 `dataVolumeTemplates` 列表。下面是一个示例。

```yaml
apiVersion: kubevirt.io/v1
kind: VirtualMachine
metadata:
  labels:
    kubevirt.io/vm: vm-alpine-datavolume
  name: vm-alpine-datavolume
spec:
  runStrategy: Halted
  template:
    metadata:
      labels:
        kubevirt.io/vm: vm-alpine-datavolume
    spec:
      domain:
        devices:
          disks:
          - disk:
              bus: virtio
            name: datavolumedisk1
        resources:
          requests:
            memory: 64M
      volumes:
      - dataVolume:
          name: alpine-dv
        name: datavolumedisk1
  dataVolumeTemplates:
  - metadata:
      name: alpine-dv
    spec:
      storage:
        resources:
          requests:
            storage: 2Gi
      source:
        http:
          url: http://cdi-http-import-server.kubevirt/images/alpine.iso
```

可以看到，dataVolumeTemplates 部分中定义的 DataVolume 包含两个部分：**source** 和 **pvc**。

**source** 部分声明，有一个位于 http 服务器上的磁盘镜像，我们希望把它用作此 VM 的卷。**pvc** 部分声明用于创建承载 **source** 数据的 PVC 的规格。

当此 VM 清单被提交到集群时，作为启动流程的一部分，会使用所提供的规格创建 PVC，并在 VM 启动之前把源数据自动导入该 PVC。当 VM 被删除时，由 DataVolume 置备的存储也会被自动删除。

#### DataVolume 的 VMI 行为 {#datavolume-vmi-behavior}

对于 VMI 对象，可以把 DataVolume 作为该 VMI 的卷源来引用。这样做时，期望所引用的 DataVolume 已经存在于集群中。VMI 会使用该 DataVolume，但 DataVolume 的生命周期不会与该 VMI 绑定。

下面是 VMI 引用 DataVolume 的示例。期望在把 VMI 清单提交到集群之前，已经创建了 DataVolume *alpine-datavolume*。在 DataVolume 仍在导入数据时把 VMI 清单提交到集群也没有问题。KubeVirt 会等到所有被引用的 DataVolume 都完成其克隆和导入阶段之后，才启动 VMI。

```yaml
apiVersion: kubevirt.io/v1
kind: VirtualMachineInstance
metadata:
  labels:
    special: vmi-alpine-datavolume
  name: vmi-alpine-datavolume
spec:
  domain:
    devices:
      disks:
      - disk:
          bus: virtio
        name: disk1
    machine:
      type: ""
    resources:
      requests:
        memory: 64M
  terminationGracePeriodSeconds: 0
  volumes:
  - name: disk1
    dataVolume:
      name: alpine-datavolume
```

#### 启用 DataVolume 支持 {#enabling-datavolume-support}

DataVolume 是由 Containerized Data Importer（CDI）项目提供的自定义资源。KubeVirt 与 CDI 集成，以便为用户提供动态创建 PVC 并把数据导入这些 PVC 的工作流。

要在 VM 或 VMI 上使用 DataVolume 卷源，必须安装 CDI。

**安装 CDI**

前往 [CDI 发行页面](https://github.com/kubevirt/containerized-data-importer/releases)。

选择最新的稳定发行版，并把对应的 cdi-controller-deployment.yaml 清单提交到你的集群。

### ephemeral {#ephemeral}

临时卷是一个本地 COW（写时复制）镜像，它使用网络卷作为只读后备存储。使用临时卷时，网络后备存储从不会被改写。所有写入都存储在位于本地存储上的临时镜像中。KubeVirt 会在 VM 启动时动态生成与该 VM 关联的临时镜像，并在 VM 停止时丢弃这些临时镜像。

临时卷适用于任何不需要磁盘持久化的场景。当 VM 到达终态（例如 succeeded、failed）时，COW 镜像会被丢弃。

目前，只有 `PersistentVolumeClaim` 可以用作临时卷的后备存储。

有关受支持后备存储的最新信息见 [KubeVirt API](http://kubevirt.io/api-reference/master/definitions.html#_v1_ephemeralvolumesource)。

```yaml
metadata:
  name: testvmi-ephemeral-pvc
apiVersion: kubevirt.io/v1
kind: VirtualMachineInstance
spec:
  domain:
    resources:
      requests:
        memory: 64M
    devices:
      disks:
      - name: mypvcdisk
        lun: {}
  volumes:
    - name: mypvcdisk
      ephemeral:
        persistentVolumeClaim:
          claimName: mypvc
```

### containerDisk {#containerdisk}

**containerDisk 最初是 registryDisk，请在需要时更新你的代码。**

`containerDisk` 特性提供在容器镜像仓库中存储和分发 VM 磁盘的能力。可以在 VirtualMachineInstance 规格的 disks 部分把 `containerDisks` 分配给 VM。

`containerDisks` 不使用任何网络共享存储设备。磁盘从容器仓库拉取，并驻留在承载使用这些磁盘的 VM 的本地节点上。

#### 何时使用 containerDisk {#when-to-use-a-containerdisk}

`containerDisks` 是可以分配给任意数量活动 VirtualMachineInstance 的临时存储设备。对于希望复制大量不需要持久数据的 VM 工作负载的用户来说，它们是理想的工具。`containerDisks` 通常与 VirtualMachineInstanceReplicaSet 一起使用。

#### 何时不使用 containerDisk {#when-not-to-use-a-containerdisk}

对于任何需要根磁盘在 VM 重启后仍然持久的工作负载，`containerDisks` 都不是好的方案。

如果磁盘必须在 VM 重启或停止后仍然保留，请改用 PVC 或 [dataVolume](#datavolume)。

#### containerDisk 工作流示例 {#containerdisk-workflow-example}

用户可以把 VirtualMachineInstance 磁盘注入容器镜像，使其能够被 KubeVirt 运行时使用。磁盘必须放在容器内的 `/disk` 目录中。支持 raw 和 qcow2 格式。建议使用 qcow2，以减小容器镜像的大小。`containerdisks` 可以且应当基于 `scratch`。除镜像之外不需要其他内容。

> **注意：** 在 kubevirt 0.20 之前，containerDisk 镜像需要以 **kubevirt/container-disk-v1alpha** 作为基础镜像。

> **注意：** containerDisk 需要对 UID 为 107（qemu）的用户可读。

示例：把本地 VirtualMachineInstance 磁盘注入容器镜像。

    cat << END > Dockerfile
    FROM scratch
    ADD --chown=107:107 fedora25.qcow2 /disk/
    END

    docker build -t vmidisks/fedora25:latest .

示例：把远程 VirtualMachineInstance 磁盘注入容器镜像。

    cat << END > Dockerfile
    FROM scratch
    ADD --chown=107:107 https://cloud.centos.org/centos/7/images/CentOS-7-x86_64-GenericCloud.qcow2 /disk/
    END

示例：把 ContainerDisk 容器镜像上传到仓库。

    docker push vmidisks/fedora25:latest

示例：把 ContainerDisk 作为临时磁盘挂接到 VM。

```yaml
metadata:
  name: testvmi-containerdisk
apiVersion: kubevirt.io/v1
kind: VirtualMachineInstance
spec:
  domain:
    resources:
      requests:
        memory: 64M
    devices:
      disks:
      - name: containerdisk
        disk: {}
  volumes:
    - name: containerdisk
      containerDisk:
        image: vmidisks/fedora25:latest
```

请注意，`containerDisk` 是基于文件的，因此不能作为 `lun` 设备挂接到 VM。

#### 自定义磁盘镜像路径 {#custom-disk-image-path}

ContainerDisk 也允许在需要时把磁盘镜像存储在任意文件夹中。流程与前面相同。主要区别在于，对于自定义位置，kubevirt 不会扫描任何镜像。你需要自行提供磁盘镜像的完整路径。提供镜像 `path` 是可选的。未提供 `path` 时，kubevirt 会在默认位置 `/disk` 中搜索磁盘镜像。

示例：构建容器磁盘镜像：

    cat << END > Dockerfile
    FROM scratch
    ADD fedora25.qcow2 /custom-disk-path/fedora25.qcow2
    END

    docker build -t vmidisks/fedora25:latest .
    docker push vmidisks/fedora25:latest

创建指向自定义位置的容器磁盘的 VMI：

```yaml
metadata:
  name: testvmi-containerdisk
apiVersion: kubevirt.io/v1
kind: VirtualMachineInstance
spec:
  domain:
    resources:
      requests:
        memory: 64M
    devices:
      disks:
      - name: containerdisk
        disk: {}
  volumes:
    - name: containerdisk
      containerDisk:
        image: vmidisks/fedora25:latest
        path: /custom-disk-path/fedora25.qcow2
```

### emptyDisk {#emptydisk}

`emptyDisk` 的工作方式类似于 Kubernetes 中的 `emptyDir`。系统会分配一块额外的稀疏 `qcow2` 磁盘，其寿命与 VM 相同。因此，它能在客户机侧 VM 重启后仍然保留，但在重新创建 VM 后不会保留。需要指定磁盘的 `capacity`。

示例：使用一块额外的 `emptyDisk` 启动 cirros，其大小为 `2GiB`：

```yaml
apiVersion: kubevirt.io/v1
kind: VirtualMachineInstance
metadata:
  name: testvmi-nocloud
spec:
  terminationGracePeriodSeconds: 5
  domain:
    resources:
      requests:
        memory: 64M
    devices:
      disks:
      - name: containerdisk
        disk:
          bus: virtio
      - name: emptydisk
        disk:
          bus: virtio
  volumes:
    - name: containerdisk
      containerDisk:
        image: kubevirt/cirros-registry-disk-demo:latest
    - name: emptydisk
      emptyDisk:
        capacity: 2Gi
```

#### 何时使用 emptyDisk {#when-to-use-an-emptydisk}

临时 VM 常常带有只读根镜像和有限的 tmpfs 空间。在许多情况下，这不足以安装应用依赖，也无法为应用数据提供足够的磁盘空间。这些数据并不关键，因此可以丢失，但在应用的生命周期内，仍然需要它们，应用才能正常运行。这时 `emptyDisk` 就能派上用场。emptyDisk 经常被使用，并挂载在 `/var/lib` 或 `/var/run` 中的某处。

### hostDisk {#hostdisk}

`hostDisk` 卷类型提供在节点上某处创建或使用磁盘镜像的能力。它的工作方式类似于 Kubernetes 中的 `hostPath`，并提供两种用法：

-   `DiskOrCreate`：如果给定位置不存在磁盘镜像，则创建一个

-   `Disk`：给定位置必须已存在磁盘镜像

注意：需要启用 HostDisk 功能门控。

示例：创建位于 /data/disk.img 的 1Gi 磁盘镜像，并将其挂接到 VM。

```yaml
apiVersion: kubevirt.io/v1
kind: VirtualMachineInstance
metadata:
  labels:
    special: vmi-host-disk
  name: vmi-host-disk
spec:
  domain:
    devices:
      disks:
      - disk:
          bus: virtio
        name: host-disk
    machine:
      type: ""
    resources:
      requests:
        memory: 64M
  terminationGracePeriodSeconds: 0
  volumes:
  - hostDisk:
      capacity: 1Gi
      path: /data/disk.img
      type: DiskOrCreate
    name: host-disk
status: {}
```

> 注意：这并不总是按预期工作。你可能需要考虑改为创建 [PersistentVolume](#persistentvolumeclaim)。

### configMap {#configmap}

`configMap` 是对 Kubernetes 中 [ConfigMap](https://kubernetes.io/docs/tasks/configure-pod-container/configure-pod-configmap/) 的引用。`configMap` 可以作为 `disks` 或 `filesystem` 呈现给 VM。下面各节分别描述每种方法。

#### 作为磁盘 {#as-a-disk}

使用 disk 时，会分配一块额外的 `iso` 磁盘，必须把它挂载到 VM 上。要挂载 `configMap`，用户可以使用 `cloudInit` 和磁盘的序列号。需要设置 `name`，以引用所创建的 Kubernetes `ConfigMap`。

> **注意：** 目前，ConfigMap 的更新不会传播到 VMI。如果 ConfigMap 被更新，只有 Pod 会感知到变更，正在运行的 VMI 不会。

> **注意：** 由于 Kubernetes CRD [问题](https://github.com/kubernetes/kubernetes/issues/68466)，你无法控制 ConfigMap 键被投射到卷内的路径。

示例：把 `configMap` 挂接到 VM，并使用 `cloudInit` 挂载 `iso` 磁盘：

```yaml
apiVersion: kubevirt.io/v1
kind: VirtualMachineInstance
metadata:
  labels:
    special: vmi-fedora
  name: vmi-fedora
spec:
  domain:
    devices:
      disks:
      - disk:
          bus: virtio
        name: containerdisk
      - disk:
          bus: virtio
        name: cloudinitdisk
      - disk:
        name: app-config-disk
        # set serial
        serial: CVLY623300HK240D
    machine:
      type: ""
    resources:
      requests:
        memory: 1024M
  terminationGracePeriodSeconds: 0
  volumes:
  - name: containerdisk
    containerDisk:
      image: kubevirt/fedora-cloud-container-disk-demo:latest
  - cloudInitNoCloud:
      userData: |-
        #cloud-config
        password: fedora
        chpasswd: { expire: False }
        bootcmd:
          # mount the ConfigMap
          - "sudo mkdir /mnt/app-config"
          - "sudo mount /dev/$(lsblk --nodeps -no name,serial | grep CVLY623300HK240D | cut -f1 -d' ') /mnt/app-config"
    name: cloudinitdisk
  - configMap:
      name: app-config
    name: app-config-disk
status: {}
```

#### 作为文件系统 {#as-a-filesystem}

使用 filesystem 时，`configMaps` 通过 `virtiofs` 共享。与使用 disk 共享 `configMaps` 相比，`filesystem` 允许把 `configMaps` 上的变更动态传播到 VMI（也就是说，VM 无需重启）。

要共享给定的 `configMap`，可以使用以下 VM 定义：

```yaml
apiVersion: kubevirt.io/v1
kind: VirtualMachineInstance
metadata:
  labels:
    special: vmi-fedora
  name: vmi-fedora
spec:
  domain:
    devices:
      filesystems:
        - name: config-fs
          virtiofs: {}
      disks:
      - disk:
          bus: virtio
        name: containerdisk
    machine:
      type: ""
    resources:
      requests:
        memory: 1024M
  terminationGracePeriodSeconds: 0
  volumes:
  - name: containerdisk
    containerDisk:
      image: quay.io/containerdisks/fedora:latest
  - cloudInitNoCloud:
      userData: |-
        #cloud-config
        chpasswd:
          expire: false
        password: fedora
        user: fedora
        bootcmd:
          # mount the ConfigMap
          - "sudo mkdir /mnt/app-config"
          - "sudo mount -t virtiofs config-fs /mnt/app-config"
    name: cloudinitdisk
  - configMap:
      name: app-config
    name: config-fs
```

### secret {#secret}

`secret` 是对 Kubernetes 中 [Secret](https://kubernetes.io/docs/concepts/configuration/secret/) 的引用。`secret` 可以作为磁盘或文件系统呈现给 VM。下面各节分别描述每种方法。

#### 作为磁盘 {#as-a-disk_1}

使用 disk 时，会分配一块额外的 `iso` 磁盘，必须把它挂载到 VM 上。要挂载 `secret`，用户可以使用 `cloudInit` 和磁盘序列号。需要设置 `secretName`，以引用所创建的 Kubernetes `Secret`。

> **注意：** 目前不支持 Secret 更新传播。如果 Secret 被更新，只有 Pod 会感知到变更，正在运行的 VMI 不会。

> **注意：** 由于 Kubernetes CRD [问题](https://github.com/kubernetes/kubernetes/issues/68466)，你无法控制 Secret 键被投射到卷内的路径。

示例：把 `secret` 挂接到 VM，并使用 `cloudInit` 挂载 `iso` 磁盘：

```yaml
apiVersion: kubevirt.io/v1
kind: VirtualMachineInstance
metadata:
  labels:
    special: vmi-fedora
  name: vmi-fedora
spec:
  domain:
    devices:
      disks:
      - disk:
          bus: virtio
        name: containerdisk
      - disk:
          bus: virtio
        name: cloudinitdisk
      - disk:
        name: app-secret-disk
        # set serial
        serial: D23YZ9W6WA5DJ487
    machine:
      type: ""
    resources:
      requests:
        memory: 1024M
  terminationGracePeriodSeconds: 0
  volumes:
  - name: containerdisk
    containerDisk:
      image: kubevirt/fedora-cloud-container-disk-demo:latest
  - cloudInitNoCloud:
      userData: |-
        #cloud-config
        password: fedora
        chpasswd: { expire: False }
        bootcmd:
          # mount the Secret
          - "sudo mkdir /mnt/app-secret"
          - "sudo mount /dev/$(lsblk --nodeps -no name,serial | grep D23YZ9W6WA5DJ487 | cut -f1 -d' ') /mnt/app-secret"
    name: cloudinitdisk
  - secret:
      secretName: app-secret
    name: app-secret-disk
status: {}
```

#### 作为文件系统 {#as-a-filesystem_1}

使用 filesystem 时，`secrets` 通过 `virtiofs` 共享。与使用 disk 共享 `secrets` 相比，`filesystem` 允许把 `secrets` 上的变更动态传播到 VMI（也就是说，VM 无需重启）。

要共享给定的 `secret`，可以使用以下 VM 定义：

```yaml
apiVersion: kubevirt.io/v1
kind: VirtualMachineInstance
metadata:
  labels:
    special: vmi-fedora
  name: vmi-fedora
spec:
  domain:
    devices:
      filesystems:
        - name: app-secret-fs
          virtiofs: {}
      disks:
        - disk:
            bus: virtio
          name: containerdisk
    machine:
      type: ""
    resources:
      requests:
        memory: 1024M
  terminationGracePeriodSeconds: 0
  volumes:
    - name: containerdisk
      containerDisk:
        image: quay.io/containerdisks/fedora:latest
    - cloudInitNoCloud:
        userData: |-
          #cloud-config
          chpasswd:
            expire: false
          password: fedora
          user: fedora
          bootcmd:
            # mount the Secret
            - "sudo mkdir /mnt/app-secret"
            - "sudo mount -t virtiofs app-secret-fs /mnt/app-secret"
      name: cloudinitdisk
    - secret:
        secretName: app-secret
      name: app-secret-fs
```

### serviceAccount {#serviceaccount}

`serviceAccount` 卷引用 Kubernetes [`ServiceAccount`](https://kubernetes.io/docs/tasks/configure-pod-container/configure-service-account/)。`serviceAccount` 可以作为磁盘或文件系统呈现给 VM。下面各节分别描述每种方法。

#### 作为磁盘 {#as-a-disk_2}

使用 disk 时，会分配一块新的 `iso` 磁盘，其中包含服务账号的内容（`namespace`、`token` 和 `ca.crt`），需要在 VM 中挂载它。有关自动挂载，请参见上面的 `configMap` 和 `secret` 示例。

> **注意：** 目前不支持 ServiceAccount 更新传播。如果 ServiceAccount 被更新，只有 Pod 会感知到变更，正在运行的 VMI 不会。

示例：

```yaml
apiVersion: kubevirt.io/v1
kind: VirtualMachineInstance
metadata:
  labels:
    special: vmi-fedora
  name: vmi-fedora
spec:
  domain:
    devices:
      disks:
      - disk:
        name: containerdisk
      - disk:
        name: serviceaccountdisk
    machine:
      type: ""
    resources:
      requests:
        memory: 1024M
  terminationGracePeriodSeconds: 0
  volumes:
  - name: containerdisk
    containerDisk:
      image: kubevirt/fedora-cloud-container-disk-demo:latest
  - name: serviceaccountdisk
    serviceAccount:
      serviceAccountName: default
```

#### 作为文件系统 {#as-a-filesystem_2}

使用 filesystem 时，`serviceAccounts` 通过 `virtiofs` 共享。与使用 disk 共享 `serviceAccounts` 相比，`filesystem` 允许把 `serviceAccounts` 上的变更动态传播到 VMI（也就是说，VM 无需重启）。

要共享给定的 `serviceAccount`，可以使用以下 VM 定义：

```yaml
apiVersion: kubevirt.io/v1
kind: VirtualMachineInstance
metadata:
  labels:
    special: vmi-fedora
  name: vmi-fedora
spec:
  domain:
    devices:
      filesystems:
        - name: serviceaccount-fs
          virtiofs: {}
      disks:
        - disk:
            bus: virtio
          name: containerdisk
    machine:
      type: ""
    resources:
      requests:
        memory: 1024M
  terminationGracePeriodSeconds: 0
  volumes:
    - name: containerdisk
      containerDisk:
        image: quay.io/containerdisks/fedora:latest
    - cloudInitNoCloud:
        userData: |-
          #cloud-config
          chpasswd:
            expire: false
          password: fedora
          user: fedora
          bootcmd:
            # mount the ConfigMap
            - "sudo mkdir /mnt/serviceaccount"
            - "sudo mount -t virtiofs serviceaccount-fs /mnt/serviceaccount"
      name: cloudinitdisk
    - name: serviceaccount-fs
      serviceAccount:
        serviceAccountName: default
```

### downwardMetrics {#downwardmetrics}

`downwardMetrics` 向客户机暴露有限的一组 VM 和宿主机指标。其格式与 [vhostmd](https://github.com/vhostmd/vhostmd) 兼容。

在某些情况下，需要获取有限的一组宿主机和 VM 指标，以便第三方诊断其设备上的性能问题。一个突出的例子是 SAP HANA。

为了向 VM 暴露 `downwardMetrics`，支持 `disk` 和 `virtio-serial port` 这两种方法。

> **注意：** 要使用这些指标，[必须启用](../cluster_admin/activating_feature_gates.md#how-to-activate-a-feature-gate) **DownwardMetrics** 功能门控。从 KubeVirt v0.42.0 开始可用。

#### 磁盘 {#disk_1}

会创建一个卷，并把它作为原始块卷暴露给客户机。KubeVirt 会定期更新它（默认每 5 秒一次）。

示例：

```yaml
apiVersion: kubevirt.io/v1
kind: VirtualMachineInstance
metadata:
  labels:
    special: vmi-fedora
  name: vmi-fedora
spec:
  domain:
    devices:
      disks:
      - disk:
          bus: virtio
        name: containerdisk
      - disk:
          bus: virtio
        name: metrics
    machine:
      type: ""
    resources:
      requests:
        memory: 1024M
  terminationGracePeriodSeconds: 0
  volumes:
  - name: containerdisk
    containerDisk:
      image: quay.io/containerdisks/fedora:latest
  - name: metrics
    downwardMetrics: {}
```

#### Virtio 串口 {#virtio-serial-port}

此方法使用 virtio 串口向 VM 暴露指标数据。KubeVirt 会在 VM 内部创建一个名为 `/dev/virtio-ports/org.github.vhostmd.1` 的端口，该端口支持 Virtio Transport 协议。可以从该端口获取 `downwardMetrics`。更多信息见 [vhostmd 文档](https://github.com/vhostmd/vhostmd/blob/master/README) 中的 `Virtio Transport`。

要使用 virtio 串口暴露指标，必须添加 `downwardMetrics` 设备（即 `spec.domain.devices.downwardMetrics: {}`）。

示例：

```yaml
apiVersion: kubevirt.io/v1
kind: VirtualMachineInstance
metadata:
  labels:
    special: vmi-fedora
  name: vmi-fedora
spec:
  domain:
    devices:
      downwardMetrics: {}
      disks:
      - disk:
          bus: virtio
        name: containerdisk
    machine:
      type: ""
    resources:
      requests:
        memory: 1024M
  terminationGracePeriodSeconds: 0
  volumes:
  - name: containerdisk
    containerDisk:
      image: quay.io/containerdisks/fedora:latest
```

#### 访问指标数据 {#accessing-metrics-data}

要访问通过磁盘或 virtio 串口共享的 DownwardMetrics，可以使用 `vm-dump-metrics` 工具：

```xml
$ sudo dnf install -y vm-dump-metrics
$ sudo vm-dump-metrics
<metrics>
  <metric type="string" context="host">
    <name>HostName</name>
    <value>node01</value>
[...]
  <metric type="int64" context="host" unit="s">
    <name>Time</name>
    <value>1619008605</value>
  </metric>
  <metric type="string" context="host">
    <name>VirtualizationVendor</name>
    <value>kubevirt.io</value>
  </metric>
</metrics>
```

`vm-dump-metrics` 作为独立工具，可用于验证串口是否正常工作并检查指标。不过，消费指标的应用通常会自己连接到 virtio 串口。

> **注意：** 在使用 virtio 串口的情况下，工具 `vm-dump-metrics` 提供 `--virtio` 选项。更多信息请参阅 `vm-dump-metrics --help`。

## 高性能特性 {#high-performance-features}

### IOThreads {#iothreads}

Libvirt 能够使用 IOThreads 进行专用磁盘访问（适用于受支持的设备）。这些是执行块 I/O 请求的专用事件循环线程，可以提高 SMP 系统上的可扩展性。KubeVirt 通过 `ioThreadsPolicy` 设置暴露这一 libvirt 特性。此外，每个 `Disk` 设备都暴露 `dedicatedIOThread` 设置。这是一个布尔值，表示应为指定磁盘分配一个永不与其他磁盘共享的独占 IOThread。

当前有效的策略有 `shared`、`auto` 和 `supplementalPool`。如果完全省略 `ioThreadsPolicy`，则会禁用 IOThreads。不过，如果任何磁盘请求专用 IOThread，则会启用 `ioThreadsPolicy`，并默认为 `shared`。

#### 共享 {#shared}

`ioThreadsPolicy` 为 `shared` 表示 KubeVirt 应使用一个由所有磁盘设备共享的线程。这一策略源于这样一个事实：大量 IOThreads 通常并无益处，因为每个线程都会带来额外的上下文切换。

`dedicatedIOThread` 设置为 `true` 的磁盘不会使用共享线程，而是会被分配一个独占线程。如果预期某块特定磁盘会有繁重的 I/O 流量，例如数据库磁盘，这通常很有用。

#### 自动 {#auto}

`auto` IOThreads 表示 KubeVirt 应使用一个 IOThread 池，并以轮询方式把磁盘分配给 IOThread。池大小通常限制为分配给 VM 的 VCPU 数量的两倍。这本质上是尝试把磁盘专用到各自的 IOThread，但只到一个合理的上限。例如，对于磁盘数量很多而 CPU 数量较少的系统，这一点就会发挥作用。

关于 IOThread 池大小有一个限制：带有 `dedicatedIOThread` 的磁盘始终保证拥有自己的线程。这实际上会降低分配给其余磁盘的线程数量上限。例如，一台有 2 个 CPU 的 VM 通常会为所有磁盘使用 4 个 IOThread。但如果有一块磁盘把 `dedicatedIOThread` 设置为 true，那么 KubeVirt 只会为共享池使用 3 个 IOThread。

对于将使用共享 IOThread 池的磁盘，始终保证至少有一个线程。因此，如果有足够多的磁盘被分配了专用 IOThread，`auto` 和 `shared` 策略实际上会产生相同的布局。

#### SupplementalPool {#supplementalpool}

`supplementalPool` IOThreads 允许用户设置 IOThread 的数量，并把相应的额外 CPU 添加到 Pod。

IOThread 的数量可以在 `ioThreads` 字段下指定：

```yaml
spec:
  domain:
    ioThreadsPolicy: supplementalPool
    ioThreads:
      supplementalPoolThreadCount: 4
```

如果 dedicatedCpuPlacement 设置为 `true`，则 Pod 使用专用 CPU，并且每个 iothread 都绑定到每个额外的 CPU。这样，vCPU 和 IOThread 各自调度到独立的 pCPU 上，彼此不会互相干扰。如果 dedicatedCpuPlacement 为 false，则 iothread 不会被绑定，vcpu 线程和 iothread 会在整组 CPU 上浮动。

此策略会消耗额外的 CPU 资源，因此当工作负载需要高度并行的 IO 时，需要谨慎设置。在底层，它使用来自 QEMU 的 virtqueue 映射；根据经验，理想的 IOThread 数量在 4 到 8 之间，具体取决于工作负载类型。这个数字取决于多个因素，需要通过运行一些性能测量来确定。有关此特性的更多细节见[这篇博客](https://developers.redhat.com/articles/2024/09/05/scaling-virtio-blk-disk-io-iothread-virtqueue-mapping#how_iothread_virtqueue_mapping_works)。

#### 使用专用（绑核）CPU 的 IOThreads {#iothreads-with-dedicated-pinned-cpus}

当客户机的 vCPU 绑定到宿主机的物理 CPU 时，最好也把 IOThread 绑定到特定 CPU，以防止它们在各 CPU 之间浮动。KubeVirt 会根据二者之间的比例自动计算，并把每个 IOThread 绑定到一个 CPU 或一组 CPU。如果 IOThread 多于 CPU，则每个 IOThread 会以轮询方式绑定到一个 CPU。否则，当 IOThread 少于 CPU 时，每个 IOThread 会绑定到一组 CPU。

#### 使用 QEMU 模拟器线程和专用（绑核）CPU 的 IOThreads {#iothreads-with-qemu-emulator-thread-and-dedicated-pinned-cpus}

为了进一步改善 vCPU 延迟，KubeVirt 可以额外分配一个专用物理 CPU<sup>[1](../compute/virtual_hardware.md#cpu)</sup>，专供模拟器线程使用，并把该线程绑定到这个 CPU。这会有效地把模拟器线程与 VMI 的 vCPU“隔离”开来。当 `ioThreadsPolicy` 设置为 `auto` 时，IOThread 也会与 vCPU“隔离”，并放置在与 QEMU 模拟器线程相同的物理 CPU 上。

### 示例 {#examples}

#### 共享 IOThreads {#shared-iothreads}

```yaml
apiVersion: kubevirt.io/v1
kind: VirtualMachineInstance
metadata:
  labels:
    special: vmi-shared
  name: vmi-shared
spec:
  domain:
    ioThreadsPolicy: shared
    cpu:
      cores: 2
    devices:
      disks:
      - disk:
          bus: virtio
        name: vmi-shared_disk
      - disk:
          bus: virtio
        name: emptydisk
        dedicatedIOThread: true
      - disk:
          bus: virtio
        name: emptydisk2
        dedicatedIOThread: true
      - disk:
          bus: virtio
        name: emptydisk3
      - disk:
          bus: virtio
        name: emptydisk4
      - disk:
          bus: virtio
        name: emptydisk5
      - disk:
          bus: virtio
        name: emptydisk6
    machine:
      type: ""
    resources:
      requests:
        memory: 64M
  volumes:
  - name: vmi-shared_disk
    persistentVolumeClaim:
      claimName: vmi-shared_pvc
  - emptyDisk:
      capacity: 1Gi
    name: emptydisk
  - emptyDisk:
      capacity: 1Gi
    name: emptydisk2
  - emptyDisk:
      capacity: 1Gi
    name: emptydisk3
  - emptyDisk:
      capacity: 1Gi
    name: emptydisk4
  - emptyDisk:
      capacity: 1Gi
    name: emptydisk5
  - emptyDisk:
      capacity: 1Gi
    name: emptydisk6
```

在此示例中，emptydisk 和 emptydisk2 都请求专用 IOThread。vmi-shared_disk 以及 emptydisk 3 到 6 将全部共享一个 IOThread。

    mypvc:        1
    emptydisk:    2
    emptydisk2:   3
    emptydisk3:   1
    emptydisk4:   1
    emptydisk5:   1
    emptydisk6:   1

#### 自动 IOThreads {#auto-iothreads}

```yaml
apiVersion: kubevirt.io/v1
kind: VirtualMachineInstance
metadata:
  labels:
    special: vmi-shared
  name: vmi-shared
spec:
  domain:
    ioThreadsPolicy: auto
    cpu:
      cores: 2
    devices:
      disks:
      - disk:
          bus: virtio
        name: mydisk
      - disk:
          bus: virtio
        name: emptydisk
        dedicatedIOThread: true
      - disk:
          bus: virtio
        name: emptydisk2
        dedicatedIOThread: true
      - disk:
          bus: virtio
        name: emptydisk3
      - disk:
          bus: virtio
        name: emptydisk4
      - disk:
          bus: virtio
        name: emptydisk5
      - disk:
          bus: virtio
        name: emptydisk6
    machine:
      type: ""
    resources:
      requests:
        memory: 64M
  volumes:
  - name: mydisk
    persistentVolumeClaim:
      claimName: mypvc
  - emptyDisk:
      capacity: 1Gi
    name: emptydisk
  - emptyDisk:
      capacity: 1Gi
    name: emptydisk2
  - emptyDisk:
      capacity: 1Gi
    name: emptydisk3
  - emptyDisk:
      capacity: 1Gi
    name: emptydisk4
  - emptyDisk:
      capacity: 1Gi
    name: emptydisk5
  - emptyDisk:
      capacity: 1Gi
    name: emptydisk6
```

此 VM 与第一个相同，区别是它请求 auto IOThreads。`emptydisk` 和 `emptydisk2` 仍会各自分配独立的 IOThread，但其余磁盘会分摊到 2 个独立的 iothread 上（CPU 核心数的两倍是 4）。

磁盘将按如下方式分配给 IOThread：

    mypvc:        1
    emptydisk:    3
    emptydisk2:   4
    emptydisk3:   2
    emptydisk4:   1
    emptydisk5:   2
    emptydisk6:   1

**注意：** 目前不支持 `dedicatedIOThreads`。以 `scsi` 作为总线类型的磁盘如果设置了该字段，webhook 会报出此错误消息：`IOThreads are not supported for disks on a %s bus`。这一限制的原因在于，KubeVirt 对所有使用 `scsi` 总线的磁盘只支持一个 scsi 控制器，而 QEMU 不支持把各个 IOThread 分配给各个 scsi 磁盘，它们只能在控制器级别分配。

### Virtio 块多队列 {#virtio-block-multi-queue}

块多队列是 Linux 块层的一个框架，它把设备 I/O 请求映射到多个队列。这把 I/O 处理拆分到多个线程，因而也拆分到多个 CPU。libvirt 建议所使用的队列数应与所分配的 CPU 数匹配，以获得最佳性能。

此特性由 `BlockMultiQueue` 设置启用，该设置位于 `Devices` 之下：

```yaml
spec:
  domain:
    devices:
      blockMultiQueue: true
      disks:
      - disk:
          bus: virtio
        name: mydisk
```

**注意：** 由于 KubeVirt 实现 CPU 分配的方式，只有在请求了特定的 CPU 分配时才能使用 blockMultiQueue。如果没有为 VirtualMachine 分配特定数量的 CPU，KubeVirt 会尽力使用节点上的全部 CPU。在这种情况下，在宿主机层面分配给 VM 的 CPU 数量可能会随时间变化。如果 blockMultiQueue 请求与节点上全部 CPU 数量相匹配的队列数，就可能导致过度分配。为避免这种情况，KubeVirt 强制要求请求特定的一份 CPU 资源，才能使用此特性。

#### 示例 {#example}

```yaml
metadata:
  name: testvmi-disk
apiVersion: kubevirt.io/v1
kind: VirtualMachineInstance
spec:
  domain:
    resources:
      requests:
        memory: 64M
        cpu: 4
    devices:
      blockMultiQueue: true
      disks:
      - name: mypvcdisk
        disk:
          bus: virtio
  volumes:
    - name: mypvcdisk
      persistentVolumeClaim:
        claimName: mypvc
```

此示例将为磁盘 `mypvcdisk` 启用块多队列，并分配 4 个队列（以匹配所请求的 CPU 数量）。

### 磁盘设备缓存 {#disk-device-cache}

KubeVirt 支持 `none`、`writeback` 和 `writethrough` 这几种 KVM/QEMU 缓存模式。

-   `none`：来自客户机的 I/O 不会在宿主机上缓存。对 I/O 需求较大的客户机使用此选项。此选项通常是最佳选择。

-   `writeback`：来自客户机的 I/O 在宿主机上缓存，并在客户机操作系统发出 flush 时写穿到物理介质。

-   `writethrough`：来自客户机的 I/O 在宿主机上缓存，但必须在写操作完成之前写穿到物理介质。

> **重要：** 如果文件系统支持直接 I/O，则默认设置 `none` 缓存模式，否则使用 `writethrough`。

> **注意：** 可以强制指定特定的缓存模式，不过如果选择了 `none` 模式而文件系统不支持直接 I/O，则启动的 VMI 会返回错误。

示例：强制使用 `writethrough` 缓存模式

```yaml
apiVersion: kubevirt.io/v1
kind: VirtualMachineInstance
metadata:
  labels:
    special: vmi-pvc
  name: vmi-pvc
spec:
  domain:
    devices:
      disks:
      - disk:
          bus: virtio
        name: pvcdisk
        cache: writethrough
    machine:
      type: ""
    resources:
      requests:
        memory: 64M
  terminationGracePeriodSeconds: 0
  volumes:
  - name: pvcdisk
    persistentVolumeClaim:
      claimName: disk-alpine
status: {}
```

### 磁盘共享 {#disk-sharing}

可共享磁盘允许多个 VM 共享同一底层存储。使用此特性时需要特别小心，因为这可能导致数据损坏和重要数据丢失。可共享磁盘要求在应用层进行数据同步，或者使用集群文件系统。这些高级配置不在本文档范围内，并且因用例而异。

如果设置了 `shareable` 选项，它会告知 libvirt/QEMU 该磁盘将由多个 VM 访问，并且不要为写入创建锁。

在此示例中，我们使用 Rook Ceph 来动态置备 PVC。

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: block-pvc
spec:
  accessModes:
    - ReadWriteMany
  volumeMode: Block
  resources:
    requests:
      storage: 1Gi
  storageClassName: rook-ceph-block
```

```bash
$ kubectl get pvc
NAME        STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS      AGE
block-pvc   Bound    pvc-0a161bb2-57c7-4d97-be96-0a20ff0222e2   1Gi        RWO            rook-ceph-block   51s
```

然后，我们可以声明 2 个 VM，并把共享磁盘的 `shareable` 选项设置为 true。

```yaml
apiVersion: kubevirt.io/v1
kind: VirtualMachine
metadata:
  labels:
    kubevirt.io/vm: vm-block-1
  name: vm-block-1
spec:
  runStrategy: Always
  template:
    metadata:
      labels:
        kubevirt.io/vm: vm-block-1
    spec:
      domain:
        devices:
          disks:
          - disk:
              bus: virtio
            name: containerdisk
          - disk:
              bus: virtio
            name: cloudinitdisk
          - disk:
              bus: virtio
            shareable: true
            name: block-disk
        machine:
          type: ""
        resources:
          requests:
            memory: 2G
      terminationGracePeriodSeconds: 0
      volumes:
      - containerDisk:
          image: registry:5000/kubevirt/fedora-with-test-tooling-container-disk:devel
        name: containerdisk
      - cloudInitNoCloud:
          userData: |-
            #cloud-config
            password: fedora
            chpasswd: { expire: False }
        name: cloudinitdisk
      - name: block-disk
        persistentVolumeClaim:
          claimName: block-pvc
---
apiVersion: kubevirt.io/v1
kind: VirtualMachine
metadata:
  labels:
    kubevirt.io/vm: vm-block-2
  name: vm-block-2
spec:
  runStrategy: Always
  template:
    metadata:
      labels:
        kubevirt.io/vm: vm-block-2
    spec:
      affinity:
        podAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
          - labelSelector:
              matchExpressions:
              - key: kubevirt.io/vm
                operator: In
                values:
                - vm-block-1
            topologyKey: "kubernetes.io/hostname"
      domain:
        devices:
          disks:
          - disk:
              bus: virtio
            name: containerdisk
          - disk:
              bus: virtio
            name: cloudinitdisk
          - disk:
              bus: virtio
            shareable: true
            name: block-disk
        machine:
          type: ""
        resources:
          requests:
            memory: 2G
      terminationGracePeriodSeconds: 0
      volumes:
      - containerDisk:
          image: registry:5000/kubevirt/fedora-with-test-tooling-container-disk:devel
        name: containerdisk
      - cloudInitNoCloud:
          userData: |-
            #cloud-config
            password: fedora
            chpasswd: { expire: False }
        name: cloudinitdisk
      - name: block-disk
        persistentVolumeClaim:
          claimName: block-pvc
```

现在我们可以尝试从第一个客户机写入一个字符串，然后从第二个客户机读取该字符串，以测试共享是否正常工作。

```bash
$ virtctl console vm-block-1
$ printf "Test awesome shareable disks" | sudo dd  of=/dev/vdc bs=1 count=150 conv=notrunc
28+0 records in
28+0 records out
28 bytes copied, 0.0264182 s, 1.1 kB/s
# Log into the second guest
$ virtctl console vm-block-2
$ sudo dd  if=/dev/vdc bs=1 count=150 conv=notrunc
Test awesome shareable disks150+0 records in
150+0 records out
150 bytes copied, 0.136753 s, 1.1 kB/s
```

如果使用的是本地设备或 RWO PVC，在共享该存储的 VM 上设置亲和性可以保证它们被调度到同一节点。在示例中，我们使用第一个 VM 上所用的标签，在第二个 VM 上设置亲和性。如果使用的是带 RWX PVC 的共享存储，则不需要亲和性规则，因为该存储可以同时挂接到多个节点。

## 与 VM 共享目录 {#sharing-directories-with-vms}

`Virtiofs` 可以使外部文件系统对 `KubeVirt` VM 可见。`Virtiofs` 是一种共享文件系统，让 VM 能够访问宿主机上的目录树。更多细节见 [Virtiofs 官方网站](https://virtio-fs.gitlab.io/)。

KubeVirt 只支持使用非特权 virtiofs 守护进程来共享 PVC。由于缺少特权，它有一些限制：

- virtiofsd 守护进程（负责与 VM 共享 PVC 的守护进程）将以 QEMU 的 UID/GID（107）运行，并且不能在不同的 UID/GID 之间切换。因此，它只能访问 UID/GID 107 有权限的目录和文件。此外，创建新文件时，无论客户机内进程的 UID/GID 是什么，这些文件始终以 QEMU 的 UID/GID 创建。
- 不支持扩展属性。

> **注意：** 需要启用功能门控 `EnableVirtioFsStorageVolumes`，才能把 `PVCs` 交由 `virtiofs` 共享。

### 共享持久卷声明 {#sharing-persistent-volume-claims}

#### 集群配置 {#cluster-configuration}

我们需要创建一个新的 VM 定义，其中包含 `spec.devices.disk.filesystems.virtiofs` 和一个 PVC。

示例：

```yaml
apiVersion: kubevirt.io/v1
kind: VirtualMachineInstance
metadata:
  name: testvmi-fs
spec:
  domain:
    devices:
      disks:
        - disk:
            bus: virtio
          name: containerdisk
        - disk:
            bus: virtio
          name: cloudinitdisk
      filesystems:
        - name: virtiofs-disk
          virtiofs: {}
    resources:
      requests:
        memory: 1024Mi
  volumes:
    - name: containerdisk
      containerDisk:
        image: quay.io/containerdisks/fedora:latest
    - cloudInitNoCloud:
        userData: |-
          #cloud-config
          password: fedora
          chpasswd: { expire: False }
      name: cloudinitdisk
    - name: virtiofs-disk
      persistentVolumeClaim:
        claimName: mypvc
```

#### VM 内部配置 {#configuration-inside-the-vm}

以下配置可以使用启动脚本完成。更多细节见 [cloudInitNoCloud](#cloudinitnocloud) 一节。不过，我们也可以通过登录到 VM 并挂载来手动完成。下面是在 Linux 和 Windows VM 中如何挂载的示例：

- Linux 示例

```bash
$ sudo mkdir -p /mnt/disks/virtio
$ sudo mount -t virtiofs virtiofs-disk /mnt/disks/virtio
```

- Windows 示例

  有关 Windows VM 所需启动步骤的细节，请参阅[此](https://virtio-fs.gitlab.io/howto-windows.html)指南。

### 共享节点目录 {#sharing-node-directories}

允许使用 [hostPath](https://kubernetes.io/docs/concepts/storage/volumes/#hostpath)。下面的配置示例仅供说明。不过，更推荐 [PVCs](#sharing-persistent-volume-claims) 方法，因为出于安全原因，通常不鼓励使用 hostPath。

#### 节点内部配置 {#configuration-inside-the-node}

要与 VM 共享该目录，我们需要登录到节点，创建共享目录（如果它尚不存在），并为共享目录设置正确的 SELinux 上下文标签 `container_file_t`。在此示例中，我们将共享一个新目录 `/mnt/data`（如果所需目录已经存在，可以跳过 `mkdir` 命令）：

```shell
$ mkdir /tmp/data
$ sudo chcon -t container_file_t /tmp/data
```

> **注意：** 如果要共享一个已存在的目录，必须先用命令 `ls -Z <directory>` 检查 SELinux 上下文标签。如果标签不存在或不是 `container_file_t`，则需要用 `chcon` 命令为它打标签。

#### 集群配置 {#cluster-configuration_1}

我们需要一个 `StorageClass`，它使用提供者 `no-provisioner`：

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
   name: no-provisioner-storage-class
provisioner: kubernetes.io/no-provisioner
reclaimPolicy: Delete
volumeBindingMode: WaitForFirstConsumer
```

为了让共享目录可供 VM 使用，我们需要创建一个可供 VM 使用的 PV 和一个 PVC：

```yaml
kind: PersistentVolume
apiVersion: v1
metadata:
  name: hostpath
spec:
  capacity:
    storage: 10Gi
  accessModes:
    - ReadWriteMany
  hostPath:
    path: "/tmp/data"
  storageClassName: "no-provisioner-storage-class"
  nodeAffinity:
    required:
      nodeSelectorTerms:
      - matchExpressions:
        - key: kubernetes.io/hostname
          operator: In
          values:
          - node01
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: hostpath-claim
spec:
  accessModes:
    - ReadWriteMany
  storageClassName: "no-provisioner-storage-class"
  resources:
    requests:
      storage: 10Gi
```

> **注意：** 把 `node01` 的值改为你希望共享目录所在节点的名称。

VM 定义必须请求 PVC `hostpath-claim`，并把它作为 virtiofs 文件系统挂接：

```yaml
apiVersion: kubevirt.io/v1
kind: VirtualMachine
metadata:
  labels:
    kubevirt.io/vm: hostpath-vm
  name: hostpath
spec:
  runStrategy: Always
  template:
    metadata:
      labels:
        kubevirt.io/domain: hostpath
        kubevirt.io/vm: hostpath
    spec:
      domain:
        cpu:
          cores: 1
          sockets: 1
          threads: 1
        devices:
          filesystems:
            - name: vm-hostpath
              virtiofs: {}
          disks:
            - name: containerdisk
              disk:
                bus: virtio
            - name: cloudinitdisk
              disk:
                bus: virtio
          interfaces:
            - name: default
              masquerade: {}
          rng: {}
        resources:
          requests:
            memory: 1Gi
      networks:
        - name: default
          pod: {}
      terminationGracePeriodSeconds: 180
      volumes:
        - containerDisk:
            image: quay.io/containerdisks/fedora:latest
          name: containerdisk
        - cloudInitNoCloud:
            userData: |-
              #cloud-config
              chpasswd:
                expire: false
              password: password
              user: fedora
          name: cloudinitdisk
        - name: vm-hostpath
          persistentVolumeClaim:
            claimName: hostpath-claim
```

#### VM 内部配置 {#configuration-inside-the-vm_1}

我们需要登录到 VM 并挂载共享目录：

```shell
$ sudo mount -t virtiofs vm-hostpath /mnt
```

#### 更新卷策略 {#update-volumes-strategy}

`updateVolumesStrategy` 字段用于指定更新正在运行的 VM 的卷时所采用的策略。支持以下策略：

  * `Replacement`：更新后的卷将在 VM 重启时被替换。
  * `Migration`：卷的更新会触发从旧卷到新卷的存储迁移。有关卷迁移的更多细节见[卷迁移文档](../storage/volume_migration.md)。

更新卷迁移依赖于功能门控 `VolumesUpdateStrategy`，而该功能门控依赖于 workloadUpdateStrategy 配置。

KubeVirt CR：

```yaml
apiVersion: kubevirt.io/v1
kind: KubeVirt
spec:
  configuration:
    developerConfiguration:
      featureGates:
        - VolumesUpdateStrategy
  workloadUpdateStrategy:
    workloadUpdateMethods:
    - LiveMigrate
```
