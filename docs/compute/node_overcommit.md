# 节点超分 {#node-overcommit}

KubeVirt 尚不支持经典的内存超分管理或内存气球。换句话说，VirtualMachineInstance 无法归还已经分配到的内存。不过，仍可以调整其他一些方面，以减小内存占用，并对每个 VMI 的内存开销进行超分。

## 移除图形设备 {#remove-the-graphical-devices}

首先，减小内存占用最安全的做法，是通过将 `spec.domain.devices.autottachGraphicsDevice` 设置为 `false`，从 VMI 中移除图形设备。更多细节和示例请参阅视频与图形设备[文档](../compute/virtual_hardware.md#video-and-graphics-device)。

这将为每个 VirtualMachineInstance 节省固定的 `16MB`，同时也会禁用 VNC 访问。

## 超分客户机开销 {#overcommit-the-guest-overhead}

在继续之前，请先熟悉 Kubernetes 的[资源不足管理](https://kubernetes.io/docs/tasks/administer-cluster/out-of-resource/)。

每个 VirtualMachineInstance 向 Kubernetes 请求的内存，都略多于用户为操作系统所请求的内存。额外的内存用于每个 VMI 的开销，这些开销来自包裹实际 VirtualMachineInstance 进程的基础设施。

为了提高节点上的 VMI 密度，可以将 `spec.domain.resources.overcommitGuestOverhead` 设置为 `true`，从而不再请求这部分额外开销：

```yaml
apiVersion: kubevirt.io/v1
kind: VirtualMachineInstance
metadata:
  name: testvmi-nocloud
spec:
  terminationGracePeriodSeconds: 30
  domain:
    resources:
      overcommitGuestOverhead: true
      requests:
        memory: 1024M
[...]
```

只要大多数 VirtualMachineInstance 不会请求全部内存，这种方式就能正常工作。短生命周期的 VMI 尤其如此。但如果你有长期运行的 VirtualMachineInstance，或者在 VirtualMachineInstance 内执行极端的内存密集型任务，你的 VMI 迟早会用完所获得的全部内存。

## 超分客户机内存 {#overcommit-guest-memory}

第三种做法是对 VMI 进行真正的内存超分。在这种场景中，通过将 `spec.domain.memory.guest` 设置为高于 `spec.domain.resources.requests.memory` 的值，明确告知 VMI：它可用的内存多于向集群请求的内存。

下面的定义向集群请求 `1024MB`，但告知 VMI 它有 `2048MB` 可用内存：

```yaml
apiVersion: kubevirt.io/v1
kind: VirtualMachineInstance
metadata:
  name: testvmi-nocloud
spec:
  terminationGracePeriodSeconds: 30
  domain:
    resources:
      overcommitGuestOverhead: true
      requests:
        memory: 1024M
    memory:
      guest: 2048M
[...]
```

只要节点上有足够的空闲内存，VMI 就可以顺利使用最多 `2048MB`。Kubernetes 会为该 VMI 分配 `Burstable` 资源类别（更多细节请参阅 [Kubernetes 中的 QoS 类别](https://kubernetes.io/docs/tasks/configure-pod-container/quality-service-pod/#create-a-pod-that-gets-assigned-a-qos-class-of-burstable)）。当节点处于内存压力之下时，适用于 Pod 的相同驱逐规则也适用于该 VMI。

隐式内存超分默认禁用。这意味着当未指定内存 request 时，它会被设置为与 `spec.domain.memory.guest` 匹配。不过，可以使用如下配置启用

```yaml
...
spec:
  configuration:
    developerConfiguration:
      memoryOvercommit: 150
```

在 `kubevirt` CR 中。例如，通过设置 `memoryOvercommit: "150"`，我们规定：当未显式设置内存 request 时，会隐式设置它，以实现 150% 的内存超分。例如，当 `spec.domain.memory.guest: 3072M` 且省略内存 request 时，内存 request 会被设置为 2048M。请注意，实际的内存 request 还取决于 OvercommitGuestOverhead 等其他配置选项。

集群范围的 `memoryOvercommit` 设置不会应用于启用了专用 CPU 放置的 VMI（`spec.domain.cpu.dedicatedCpuPlacement: true`）。KubeVirt 会跳过这些 VMI 的隐式内存超分计算。

## 配置节点的内存压力行为 {#configuring-the-memory-pressure-behavior-of-nodes}

如果节点处于内存压力之下，根据 `kubelet` 的配置，虚拟机可能会被 OOM 处理程序或 `kubelet` 本身杀死。可以根据 VirtualMachineInstance 的需求，通过以下方式调整该行为：

* 配置[软驱逐阈值](https://kubernetes.io/docs/tasks/administer-cluster/out-of-resource/#soft-eviction-thresholds)
* 配置[硬驱逐阈值](https://kubernetes.io/docs/tasks/administer-cluster/out-of-resource/#hard-eviction-thresholds)
* 为 VirtualMachineInstance 请求合适的 QoS 类别
* 设置 `--system-reserved` 和 `--kubelet-reserved`
* 启用 KSM
* 启用交换空间

### 配置软驱逐阈值 {#configuring-soft-eviction-thresholds}

> 注意：软驱逐实际上会关闭 VirtualMachineInstance。
> 它们不会被暂停、休眠或迁移。此外，软驱逐
> 默认处于禁用状态。

如果已配置，一旦可用内存低于通过 `--eviction-soft` 指定的阈值，VirtualMachineInstance 就会被驱逐，并且该 VirtualMachineInstance 有机会在通过 `--eviction-max-pod-grace-period` 指定的时间范围内执行 VMI 的关闭。标志 `--eviction-soft-grace-period` 指定软驱逐条件必须持续多久，才会触发软驱逐。

如果按照 VMI 的需求正确设置，超分应当只在少数情况下导致某些 VMI 发生软驱逐。它们甚至可能以更低的初始内存需求被重新调度到同一节点。对于某些工作负载类型，这完全可以接受，并能带来更好的整体内存利用率。

### 配置硬驱逐阈值 {#configuring-hard-eviction-thresholds}

> 注意：如果未指定，一旦 `memory.available` 低于 `100Mi`，kubelet 就会对 Pod 执行硬驱逐。

通过 `--eviction-hard` 设置的限制会导致 VirtualMachineInstance 或 Pod 被立即驱逐。这会在没有宽限期的情况下停止 VMI，相当于真实计算机上的断电。

如果触及硬限制，VMI 有时会直接被杀死。由于它们再次启动时内存消耗更低，可能会立刻被重新调度回同一节点。如果内存阈值极少被触及，并且 VMI 执行的工作可以复现，或者可以从某些检查点恢复，那么这可以是一种简单的选择。

## 为 VirtualMachineInstance 请求合适的 QoS 类别 {#requesting-the-right-qos-class-for-virtualmachineinstances}

不同的 QoS 类别会根据 `requests.memory` 和 `limits.memory` [分配给 Pod 和 VirtualMachineInstance](https://kubernetes.io/docs/tasks/administer-cluster/cpu-management-policies/#static-policy)。KubeVirt 目前支持的 QoS 类别为 `Burstable` 和 `Guaranteed`。`Burstable` VMI 会先于 `Guaranteed` VMI 被驱逐。

这样就可以创建两类 VMI：

* 一类可以设置相等的 `requests.memory` 和 `limits.memory`，因此会被分配 `Guaranteed` 类别。这类不会被驱逐，也不应遇到内存问题，但资源要求更高。
* 另一类可以不设置 `limits.memory`，或者其 `limits.memory` 大于 `requests.memory`，因此会被分配 `Burstable` 类别。这些 VMI 会首先被驱逐。

## 设置 `--system-reserved` 和 `--kubelet-reserved` {#setting-system-reserved-and-kubelet-reserved}

为同一节点上运行的其他守护进程（不是 DaemonSet）保留一些内存可能很重要（例如 ssh、dhcp 服务器等）。可以使用 `--system reserved` 开关完成此预留。另外，针对 kubelet 和 Docker，还存在一个名为 `--kubelet-reserved` 的特殊标志。

## 启用 KSM {#enabling-ksm}

可以在节点上启动 [KSM](https://www.linux-kvm.org/page/KSM)（内核同页合并）守护进程。根据其调优参数，它可以或多或少激进地尝试在应用程序与 VirtualMachineInstance 之间合并相同的页面。配置得越激进，它自身占用的 CPU 就越多，因此内存超分带来的好处会伴随轻微的 CPU 性能损失。

配置文件调优可以更改扫描频率（KSM 多久激活一次）和激进程度（它每秒扫描多少个页面）。

## 启用交换空间 {#enabling-swap}

> 注意：这确实可以确保你的虚拟机不会崩溃，也不会被从节点上驱逐，但一旦节点内存耗尽，性能会变得相当不可预测，而且 kubelet 可能检测不到应当驱逐 Pod 以再次提升性能。

目前，在 Kubernetes 上启用交换空间通常[不推荐](https://github.com/kubernetes/kubernetes/issues/53533)。不过，它与 KSM 结合时会有用，因为 KSM 会随着时间合并相同的页面。交换空间让 VMI 能够成功分配内存，而这些内存随后实际上不会被使用，因为 KSM 之后会进行去重。

## 节点 CPU 分配比 {#node-cpu-allocation-ratio}

KubeVirt 在 Kubernetes Pod 中运行虚拟机。该 Pod 向宿主机请求一定量的 CPU 时间。另一方面，虚拟机在创建时带有一定数量的 vCPU。vCPU 的数量不一定与 Pod 所请求的 CPU 数量对应。根据 Pod 的 [QOS](https://kubernetes.io/docs/tasks/configure-pod-container/quality-service-pod/)，vCPU 可以被调度到数量可变的物理 CPU 上；这取决于节点上的可用 CPU 资源。当节点上可用的 CPU 少于所请求的 vCPU 时，vCPU 就会被超分。

默认情况下，每个 Pod 请求 100 毫核的 CPU 时间。Pod 上请求的 CPU 会设置 cgroups 的 cpu.shares，调度器以此作为优先级，为该 Pod 中的 vCPU 提供 CPU 时间。
随着 vCPU 数量增加，在与节点上的其他进程，或与 vCPU 数量更少的其他虚拟机实例竞争时，每个 vCPU 可能获得的 CPU 时间会减少。

`cpuAllocationRatio` 用来根据 vCPU 数量，规范化 Pod 将要请求的 CPU 时间。
例如，Pod 的 CPU request = vCPU 数量 * 1/cpuAllocationRatio。
当 cpuAllocationRatio 设置为 1 时，会为该 Pod 请求全部数量的 vCPU。

> 注意：在 Kubernetes 中，一个完整的核是 1000 单位的 CPU 时间
> [更多信息](https://kubernetes.io/docs/tasks/configure-pod-container/assign-cpu-resource/)

管理员可以通过更新 KubeVirt CR 来更改此比率

```yaml
...
spec:
  configuration:
    developerConfiguration:
      cpuAllocationRatio: 10
```
