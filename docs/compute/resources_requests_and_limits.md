# 资源请求与限制 {#resources-requests-and-limits}

在本文档中，我们讨论的是设置在 virt-launcher 计算容器上的资源值，为简单起见，下文称之为“容器”。

## CPU {#cpu}

注意：此处忽略专用 CPU（以及隔离的模拟器线程），因为它们有[专门的页面](../compute/dedicated_cpu_resources.md)。

### 容器上的 CPU 请求 {#cpu-requests-on-the-container}
- 默认情况下，容器为每个 vCPU 请求 (1/cpuAllocationRatio) 的 CPU。vCPU 的数量为 sockets * cores * threads，默认值为 1。
- cpuAllocationRatio 的默认值为 10，但可以在 CR 中更改。
- 如果在 VM(I) 上手动设置了 CPU 限制，但没有设置 CPU 请求，则容器上的 CPU 请求将与 CPU 限制一致
- 在 VM(I) 上手动设置 CPU 请求将覆盖上述所有设置，并成为该容器的 CPU 请求

### 容器上的 CPU 限制 {#cpu-limits-on-the-container}
- 默认情况下，不为该容器设置 CPU 限制
- 如果启用了自动 CPU 限制（见下一节），则该容器将具有每个 vCPU 为 1 的 CPU 限制
- 在 VM(I) 上手动设置 CPU 限制将覆盖上述所有设置，并成为该容器的 CPU 限制

### 自动 CPU 限制 {#auto-cpu-limits}
如果以下所有条件均为真，KubeVirt 会自动为 VM(I) 设置 CPU 限制：

- 将要创建 VMI 的命名空间拥有一份包含 cpu 限制的 ResourceQuota。
- 该 VMI 没有手动设置的 cpu 限制。
- 该 VMI 没有请求专用 CPU。

此外，你可以在 KubeVirt CR 中添加 namespaceLabelSelector，以强制设置 CPU 限制。

在这两种情况下，所创建的 VM(I) 都将具有每个 vCPU 为 1 的 CPU 限制。

#### autoCPULimitNamespaceLabelSelector 配置 {#autocpulimitnamespacelabelselector-configuration}
集群管理员可以在 KubeVirt CR 中定义标签选择器。
一旦定义了该标签选择器，如果创建时所在的命名空间与该选择器匹配，则在其中创建的所有 VM(I) 都会设置 CPU 限制。

示例：

- CR：
```yaml
apiVersion: kubevirt.io/v1
kind: KubeVirt
metadata:
  name: kubevirt
  namespace: kubevirt
spec:
  configuration:
    autoCPULimitNamespaceLabelSelector:
      matchLabels:
        autoCpuLimit: "true"
```

- 命名空间：
```yaml
apiVersion: v1
kind: Namespace
metadata:
  labels:
    autoCpuLimit: "true"
    kubernetes.io/metadata.name: default
  name: default
```

## 内存 {#memory}
### 容器上的内存请求 {#memory-requests-on-the-container}
- VM(I) 必须指定期望的内存量，可以写在 spec.domain.memory.guest 或 spec.domain.resources.requests.memory 中（忽略大页，参见[专门的页面](../compute/hugepages.md)）。如果两者都已设置，则内存请求优先。会在其上加上计算得出的开销量，从而构成该容器的内存请求值。关于哪些因素构成该开销，请参见下方的[内存开销](#memory-overhead)。

### 内存开销 {#memory-overhead}

各种 VM 资源，例如视频适配器、IOThreads 以及辅助系统软件，会从节点消耗额外内存，超出准备供客户机操作系统消耗的请求内存。为了给调度器提供更好的估计值，将计算此内存开销并加到请求的内存上。

默认内存开销已设置为 32Mi，用以计入这些额外的系统资源。

有关资源请求与限制的更多信息，请参见[具有资源请求的 Pod 如何被调度](https://kubernetes.io/docs/concepts/configuration/manage-compute-resources-container/#how-pods-with-resource-requests-are-scheduled)。

#### 查看内存开销 {#viewing-memory-overhead}

> 注意：此功能要求启用 `VmiMemoryOverheadReport` 功能门控。

启用该功能门控后，计算得出的开销会公开在 VMI 的 status 中：

```yaml
apiVersion: kubevirt.io/v1
kind: VirtualMachineInstance
metadata:
  name: my-vmi
status:
  memory:
    memoryOverhead: 512Mi
```

`status.memory.memoryOverhead` 字段在 VMI 启动时填充，并反映为当前 virt-launcher Pod 计算得出的开销。

#### 实时迁移后的内存开销 {#memory-overhead-after-live-migration}

在实时迁移期间，目标节点可能会计算出不同的开销值（例如，由于硬件或配置不同）。迁移进行期间，目标开销报告在 `status.migrationState.targetMemoryOverhead` 中。迁移完成后，`status.memory.memoryOverhead` 会更新以反映新值：

```yaml
status:
  memory:
    memoryOverhead: 520Mi
  migrationState:
    targetMemoryOverhead: 520Mi
```

### 容器上的内存限制 {#memory-limits-on-the-container}
- 默认情况下，不为该容器设置内存限制
- 如果启用了自动内存限制（见下一节），则该容器将具有所请求内存 2 倍的限制。
- 在 VM(I) 上手动设置内存限制，会在该容器上设置相同的值

#### 警告 {#warnings}
- 内存限制必须大于内存请求与开销之和，否则容器会出现内存请求大于限制的情况，并被 Kubernetes 拒绝。
- 设置了内存限制时，内存用量突发可能导致 VM 崩溃


### 自动内存限制 {#auto-memory-limits}
如果以下所有条件均为真，KubeVirt 会自动为 VM(I) 设置内存限制：

- 将要创建 VMI 的命名空间拥有一份包含内存限制的 ResourceQuota。
- 该 VMI 没有手动设置的内存限制。
- 该 VMI 没有请求专用 CPU。

如果前述所有条件均为真，内存限制将被设置为内存请求的（`2x`）。
可以按命名空间添加标签 `alpha.kubevirt.io/auto-memory-limits-ratio`，并带上期望的自定义值，来调整这一比例。
例如，使用 `alpha.kubevirt.io/auto-memory-limits-ratio: "1.2"` 时，所设置的内存限制将等于内存请求的（`1.2x`）。
