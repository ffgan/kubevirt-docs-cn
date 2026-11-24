# NUMA {#numa}

**功能状态：** KubeVirt v0.43

KubeVirt 中的 NUMA 支持在现阶段仅限于一小部分特殊用例，并将随着 Kubernetes 的改进而逐步完善。

总体而言，目标是尽可能高效地把宿主机 NUMA 拓扑映射到虚拟机拓扑，以提升性能。

可以使用以下 NUMA 映射策略：

- [**GuestMappingPassthrough**](#guestmappingpassthrough)

## 前提条件 {#preconditions}

要使用当前的 NUMA 支持，必须满足以下前提条件：

* 必须配置[专用 CPU 资源](../compute/dedicated_cpu_resources.md)。
* 目标节点上需要能够分配[大页](../compute/virtual_hardware.md#hugepages)。
* 必须启用 `NUMA` [功能门控](../cluster_admin/activating_feature_gates.md#how-to-activate-a-feature-gate)。

## GuestMappingPassthrough {#guestmappingpassthrough}

GuestMappingPassthrough 会把节点的 NUMA 拓扑直通给客户机。该拓扑基于 VMI 通过 CPU 管理器从 kubelet 分配到的专用 CPU。可以在 VMI 上设置 `spec.domain.cpu.numa.guestMappingPassthrough` 来请求它。

由于 KubeVirt 无法预先知道 VMI 将从 kubelet 获得哪些独占 CPU，因此存在一些限制：

* 客户机在被重新调度时可能会看到不同的 NUMA 拓扑。
* 得到的 NUMA 拓扑可能是不对称的。
* 如果所分配的 NUMA 节点上没有足够的大页，VMI 可能无法在该节点上启动。

尽管这种 NUMA 建模策略有其局限，但使客户机的 NUMA 架构与节点对齐，对于高性能应用可能至关重要。

VMI 示例可能如下所示：

```yaml
apiVersion: kubevirt.io/v1
kind: VirtualMachineInstance
metadata:
  name: numavm
spec:
  domain:
    cpu:
      cores: 4
      dedicatedCpuPlacement: true
      numa:
        guestMappingPassthrough: { }
    devices:
      disks:
        - disk:
            bus: virtio
          name: containerdisk
        - disk:
            bus: virtio
          name: cloudinitdisk
    resources:
      requests:
        memory: 64Mi
    memory:
      hugepages:
        pageSize: 2Mi
  volumes:
    - containerDisk:
        image: quay.io/kubevirt/cirros-container-disk-demo
      name: containerdisk
    - cloudInitNoCloud:
        userData: |
          #!/bin/sh
          echo 'printed from cloud-init userdata'
      name: cloudinitdisk
```

## 运行实时工作负载 {#running-real-time-workloads}

### 概述 {#overview}

可以部署运行实时内核的虚拟机，并利用 [libvirtd 的客户机 CPU 与内存优化](https://www.libvirt.org/kbase/kvm-realtime.html) 来改善整体延迟。这些改动主要利用 KubeVirt 中已经提供的设置，下文会很快说明；同时，VMI 清单现在公开了两个新设置，用于指示 KubeVirt 按照运行实时工作负载的推荐调优设置来配置生成的 libvirt XML。

要使用这些优化设置，VMI 规范中新增了两个设置：

- `spec.domain.cpu.realtime`：定义后，它指示 KubeVirt 为 VCPU 配置 Linux 调度器，使进程以 FIFO 调度策略（SCHED_FIFO）和优先级 1 运行。该设置保证宿主机上运行的所有进程都以实时优先级执行。

- `spec.domain.cpu.realtime.mask`：它定义分配给 VM 的哪些 VCPU 用于实时。如果未定义，libvirt 会把分配到的所有 VCPU 定义为以 FIFO 调度和最高优先级（1）运行进程。

### 前提条件 {#preconditions_1}

运行实时工作负载的前提包括锁定集群中的资源，以便实时 VM 独占使用。这对应于已经配置了[一组专用 CPU](dedicated_cpu_resources.md)、并且提供 [NUMA](numa.md) 支持的节点（或某一个节点），同时有若干空闲的 2Mi 或 1Gi 大页（取决于 VMI 中的配置）。此外，节点必须配置为允许调度器以实时策略运行进程。

### 能够运行实时工作负载的节点 {#nodes-capable-of-running-real-time-workloads}

当 KubeVirt Pod 部署到某个节点时，它会检查该节点是否能够以实时调度策略运行进程，并将该节点标记为具备实时能力（kubevirt.io/realtime）。另一方面，如果节点无法提供这种能力，则不会打上该标签。要检查哪些节点能够承载实时 VM 工作负载，请运行以下命令：

```bash
$>kubectl get nodes -l kubevirt.io/realtime
NAME         STATUS   ROLES              AGE    VERSION
worker-0-0   Ready    worker             12d    v1.20.0+df9c838
```

在内部，每个节点上运行的 KubeVirt Pod 会检查内核设置 `kernel.sched_rt_runtime_us` 是否等于 -1。该值允许进程以实时调度策略运行，且时间不受限制。

### 配置 VM 清单 {#configuring-a-vm-manifest}

下面是一个 VM 清单示例，它运行配置为使用实时内核的自定义 Fedora 容器磁盘。这些设置已针对最佳效率进行配置。

```yaml
---
apiVersion: kubevirt.io/v1
kind: VirtualMachine
metadata:
  labels:
    kubevirt.io/vm: fedora-realtime
  name: fedora-realtime
spec:
  runStrategy: Always
  template:
    metadata:
      labels:
        kubevirt.io/vm: fedora-realtime
    spec:
      domain:
        devices:
          autoattachSerialConsole: true
          autoattachMemBalloon: false
          autoattachGraphicsDevice: false
          disks:
          - disk:
              bus: virtio
            name: containerdisk 
          - disk:
              bus: virtio
            name: cloudinitdisk
        machine:
          type: ""
        resources:
          requests:
            memory: 1Gi
            cpu: 2
          limits:
            memory: 1Gi
            cpu: 2
        cpu:
          model: host-passthrough
          dedicatedCpuPlacement: true
          isolateEmulatorThread: true
          ioThreadsPolicy: auto
          features:
            - name: tsc-deadline
              policy: require
          numa:
            guestMappingPassthrough: {}
          realtime: {}
        memory:
          hugepages:
            pageSize: 1Gi
      terminationGracePeriodSeconds: 0
      volumes:
      - containerDisk:
          image: quay.io/kubevirt/fedora-realtime-container-disk:v20211008-22109a3
        name: containerdisk
      - cloudInitNoCloud:
          userData: |-
            #cloud-config
            password: fedora
            chpasswd: { expire: False }
            bootcmd:
              - tuned-adm profile realtime
        name: cloudinitdisk
```

拆解这些经过调优的部分，配置如下：

[设备](https://libvirt.org/kbase/kvm-realtime.html#device-configuration)：
- 禁用客户机的内存气球能力
- 避免挂接图形设备，以减少内核收到的中断次数。

```yaml
spec:
  domain:
    devices:
      autoattachSerialConsole: true
      autoattachMemBalloon: false
      autoattachGraphicsDevice: false
```

[CPU](https://libvirt.org/kbase/kvm-realtime.html#cpu-configuration)：
- model：`host-passthrough`，以便客户机看到宿主机 CPU，且不屏蔽任何能力。
- 专用 CPU 放置：VM 需要分配到专用 CPU。Kubernetes CPU 管理器负责这一方面。
- isolatedEmulatorThread：请求一个额外的 CPU 来运行模拟器，从而避免占用工作负载 CPU 的 CPU 周期。
- ioThreadsPolicy：设置为 auto，让专用 IO 线程与模拟器线程运行在同一 CPU 上。
- NUMA：定义 `guestMappingPassthrough` 即可为此 VM 启用 NUMA 支持。
- realtime：指示 virt-handler 将此 VM 配置为实时工作负载，例如将 VCPU 配置为使用 FIFO 调度器策略，并把优先级设置为 1。
cpu:
```yaml
cpu:
  model: host-passthrough
  dedicatedCpuPlacement: true
  isolateEmulatorThread: true
  ioThreadsPolicy: auto
  features:
    - name: tsc-deadline
      policy: require
  numa:
    guestMappingPassthrough: {}
  realtime: {}
```

[内存](https://libvirt.org/kbase/kvm-realtime.html#memory-configuration)
- pageSize：以给定大小的大页分配该 Pod 的内存，本例中为 1Gi。
```yaml
memory:
  hugepages:
    pageSize: 1Gi
```

### 如何只把 VCPU 专用给实时用途 {#how-to-dedicate-vcpus-for-real-time-only}

可以通过 `realtime.mask` 设置，传入要隔离出来以使用实时调度策略的 VCPU 正则表达式。

```yaml
cpu:
  numa:
    guestMappingPassthrough: {}
  realtime:
    mask: "0"
```

应用此配置后，KubeVirt 只会把第一个 VCPU 设置为实时调度器策略，其余 VCPU 继续使用默认调度器策略。其他有效掩码的示例如下：
- `0-3`：将核 0 到 3 用于实时调度，假定该 VM 至少请求了 3 个核。
- `0-3,^1`：仅将核 0、2 和 3 用于实时调度，假定该 VM 至少请求了 3 个核。

## 延伸阅读 {#additional-reading}

Kubernetes 还提供了其他可能与你的用例相关的 NUMA 组件，但通常默认不启用。有关这些组件的配置细节，请查阅 Kubernetes 文档。

### 拓扑管理器 {#topology-manager}

拓扑管理器提供与 CPU 隔离、内存和设备局部性相关的优化。例如，当 SR-IOV 网络适配器的 VF 分配需要与某个 NUMA 节点对齐时，它就很有用。

[https://kubernetes.io/docs/tasks/administer-cluster/topology-manager/](https://kubernetes.io/docs/tasks/administer-cluster/topology-manager/)

### 内存管理器 {#memory-manager}

内存管理器类似于 [CPU 管理器](dedicated_cpu_resources.md)。例如，当你希望将大页分配与某个 NUMA 节点对齐时，它就很有用。它与 [拓扑管理器](numa.md#topology-manager) 配合工作。

> 内存管理器采用提示生成协议，为 Pod 产生最合适的 NUMA 亲和性。内存管理器把这些亲和性提示提供给中央管理器（拓扑管理器）。Pod 会根据这些提示以及拓扑管理器策略，被拒绝或被准入到该节点。

[https://kubernetes.io/docs/tasks/administer-cluster/memory-manager/](https://kubernetes.io/docs/tasks/administer-cluster/memory-manager/)
