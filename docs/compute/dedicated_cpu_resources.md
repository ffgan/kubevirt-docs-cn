# 专用 CPU 资源 {#dedicated-cpu-resources}

某些工作负载在执行期间需要可预测的延迟和更高的性能，获得专用 CPU 资源会对它们有帮助。KubeVirt 依赖 Kubernetes CPU 管理器，能够将客户机的 vCPU 绑定到宿主机的 pCPU。

## [Kubernetes CPU 管理器](https://kubernetes.io/docs/tasks/administer-cluster/cpu-management-policies/) {#kubernetes-cpu-manager}

Kubernetes CPU 管理器是一种影响工作负载调度的机制。它会把工作负载放置到能够分配 `Guaranteed` 资源的宿主机上，并在满足以下要求时，将某些 Pod 的容器绑定到宿主机 pCPU：

* [Pod 的 QoS](https://kubernetes.io/docs/tasks/configure-pod-container/quality-service-pod/#create-a-pod-that-gets-assigned-a-qos-class-of-guaranteed) 为 Guaranteed
	* resources 的 requests 与 limits 相等
	* Pod 中的所有容器都声明了 CPU 和内存需求
* 请求的 CPU 数量为整数

更多信息：

* [在 Kubernetes 上启用 CPU 管理器](https://kubernetes.io/docs/tasks/administer-cluster/cpu-management-policies/)
* [在 OKD 上启用 CPU 管理器](https://docs.openshift.com/container-platform/4.10/scalability_and_performance/using-cpu-manager.html)
* [介绍该功能的 Kubernetes 博客](https://kubernetes.io/blog/2018/07/24/feature-highlight-cpu-manager/)

## 请求专用 CPU 资源 {#requesting-dedicated-cpu-resources}

在 VMI spec 中将 `spec.domain.cpu.dedicatedCpuPlacement` 设置为 `true`，即表示希望为该 VMI 分配专用 CPU 资源。

KubeVirt 会验证是否满足全部必要条件，以便 Kubernetes CPU 管理器将 virt-launcher 容器绑定到专用的宿主机 CPU。一旦 virt-launcher 运行，VMI 的 vCPU 就会被绑定到已为 virt-launcher 容器专用的 pCPU。

可以通过在 `spec.domain.cpu` 中设置客户机拓扑（`sockets`、`cores`、`threads`），或者将 `spec.domain.resources.[requests/limits].cpu` 设置为表示所请求 vCPU 数量的整数（[1-9]+），来表达期望的 VMI vCPU 数量。vCPU 数量按 `sockets * cores * threads` 计算；如果 `spec.domain.cpu` 为空，则取 `spec.domain.resources.requests.cpu` 或 `spec.domain.resources.limits.cpu` 的值。

> **注意：** 用户不应同时指定 `spec.domain.cpu` 和 `spec.domain.resources.[requests/limits].cpu`
>
> **注意：** `spec.domain.resources.requests.cpu` 必须等于 `spec.domain.resources.limits.cpu`
>
> **注意：** 多项 CPU 密集型微基准测试表明，使用 `spec.domain.cpu.sockets` 时比使用 `spec.domain.cpu.cores` 具有显著的性能优势。

所有不一致的要求都会被拒绝。

```yaml
apiVersion: kubevirt.io/v1
kind: VirtualMachineInstance
spec:
  domain:
    cpu:
      sockets: 2
      cores: 1
      threads: 1
      dedicatedCpuPlacement: true
    resources:
      limits:
        memory: 2Gi
[...]
```

或者

```yaml
apiVersion: kubevirt.io/v1
kind: VirtualMachineInstance
spec:
  domain:
    cpu:
      dedicatedCpuPlacement: true
    resources:
      limits:
        cpu: 2
        memory: 2Gi
[...]
```

## 为 QEMU 模拟器请求专用 CPU {#requesting-dedicated-cpu-for-qemu-emulator}

若干 QEMU 线程（例如 QEMU 主事件循环、异步 I/O 操作完成等）也会在与 VMI 的 vCPU 相同的物理 CPU 上执行。这可能影响 vCPU 的预期延迟。为了增强 KubeVirt 中的实时支持并改善延迟，KubeVirt 会额外分配一个专用 CPU，专供模拟器线程使用，并将该线程绑定到此 CPU。这会有效地将模拟器线程与 VMI 的 vCPU“隔离”。如果将 `ioThreadsPolicy` 设置为 `auto`，IOThreads 也会被“隔离”，并放置在与 QEMU 模拟器线程相同的物理 CPU 上。

可以通过指定 `isolateEmulatorThread: true` 来启用此功能，该字段位于 VMI spec 的 `Spec.Domain.CPU` 部分。自然，该设置必须与 `dedicatedCpuPlacement: true` 组合指定。

示例：

```yaml
apiVersion: kubevirt.io/v1
kind: VirtualMachineInstance
spec:
  domain:
    cpu:
      dedicatedCpuPlacement: true
      isolateEmulatorThread: true
    resources:
      limits:
        cpu: 2
        memory: 2Gi
```

### 启用了 SMT 的计算节点 {#compute-nodes-with-smt-enabled}

当满足以下条件时：

- 计算节点已启用 [SMT](https://en.wikipedia.org/wiki/Simultaneous_multithreading)
- kubelet 的 CPUManager 策略设置为 static - [full-pcpus-only](https://kubernetes.io/docs/tasks/administer-cluster/cpu-management-policies/#static-policy-options)
- VM 配置为具有偶数个 CPU
- 已启用 `dedicatedCpuPlacement` 和 `isolateEmulatorThread`

该 VM 会被调度，但会被 kubelet 拒绝，并出现以下事件：
```
SMT Alignment Error: requested 3 cpus not multiple cpus per core = 2
```

为了解决此问题：

1. 在 KubeVirt CR 中启用 `AlignCPUs` 功能门控。
2. 向 KubeVirt CR 添加以下注解：

```yaml
alpha.kubevirt.io/EmulatorThreadCompleteToEvenParity:
```

随后，KubeVirt 会为模拟器线程添加一个或两个专用 CPU，使 CPU 总数补齐为偶数。

## 识别正在运行 CPU 管理器的节点 {#identifying-nodes-with-a-running-cpu-manager}

目前，[Kubernetes 不会为节点打标签](https://github.com/kubernetes/kubernetes/issues/66525)以标明正在运行 CPU 管理器的节点。

KubeVirt 有一种机制，可以识别哪些节点正在运行 CPU 管理器，并手动添加 `cpumanager=true` 标签。当 KubeVirt 识别到该节点上不再运行 CPU 管理器时，会移除此标签。在 Kubernetes 提供所需功能之前，这种自动识别应视为临时变通办法。因此，应通过在 KubeVirt CR 上激活 `CPUManager` [功能门控](../cluster_admin/activating_feature_gates.md#how-to-activate-a-feature-gate) 来手动启用此功能。

当自动识别被禁用时，集群管理员可以在 CPU Manager 运行时，手动为所有节点添加上述标签。

-   可以查看节点标签：`kubectl describe nodes`

-   管理员可以手动为遗漏的节点打标签：
    `kubectl label node [node_name] cpumanager=true`


## 边车容器与 CPU 分配开销 {#sidecar-containers-and-cpu-allocation-overhead}

**注意：** 要运行边车容器，KubeVirt 要求在 KubeVirt 的 CR 中启用 `Sidecar` [功能门控](../cluster_admin/activating_feature_gates.md#how-to-activate-a-feature-gate)。

根据 Kubernetes CPU 管理器模型，为了使 Pod 达到所需的 QOS 级别 `Guaranteed`，Pod 中的所有容器都必须声明 CPU 和内存需求。目前，KubeVirt 经常使用边车容器来挂载 VMI 的镜像仓库磁盘。它还会使用其钩子机制的边车容器。这些额外资源可以视为开销，在计算节点容量时应加以考虑。

**注意：** 边车资源的当前默认值为：`CPU: 200m` `Memory: 64M`。由于 CPU 资源不是以整数表示，CPU 管理器不会尝试将边车容器绑定到宿主机 CPU。
