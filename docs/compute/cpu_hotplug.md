# CPU 热插拔 {#cpu-hotplug}

CPU 热插拔功能在 KubeVirt v1.0 中引入，使得可以配置 VM 工作负载，以便在 VM 运行期间添加或移除虚拟 CPU。

### 摘要 {#abstract}
**虚拟 CPU**（vCPU）是客户机 VM 操作系统所看到的 CPU。VM 所有者可以使用 CPU 拓扑字段（`spec.template.spec.domain.cpu`），从 VM spec 模板管理 vCPU 的数量。`cpu` 对象包含整数 `cores,sockets,threads`，因此虚拟 CPU 按以下公式计算：`cores * sockets * threads`。

在引入 CPU 热插拔之前，VM 所有者可以在 VM 运行期间更改 VM 模板中的这些整数，但它们要到下一个启动周期才会在正在运行的 VM 中生效。有了 CPU 热插拔，就可以修补 VM 模板中的 `sockets` 整数，并且更改会立即生效。

每热插拔一个新插槽，客户机将看到的新增 vCPU 数量为 `cores * threads`，因为 vCPU 的总体计算公式是 `cores * sockets * threads`。

## 配置 {#configuration}

### 配置工作负载更新策略 {#configure-the-workload-update-strategy}
当前热插拔流程的实现要求 VM 进行实时迁移。
迁移将由工作负载更新器自动触发。KubeVirt CR 中的工作负载更新策略必须配置为 `LiveMigrate`，如下所示：

```yaml
apiVersion: kubevirt.io/v1
kind: KubeVirt
spec:
  workloadUpdateStrategy:
    workloadUpdateMethods:
    - LiveMigrate
```

### 配置 VM 滚动更新策略 {#configure-the-vm-rollout-strategy}
热插拔要求 VM 滚动更新策略为 `LiveUpdate`，这样对 VM 对象所做的更改才会传播到 VMI 而无需重启。
这也在 KubeVirt CR 配置中完成：

```yaml
apiVersion: kubevirt.io/v1
kind: KubeVirt
spec:
  configuration:
    vmRolloutStrategy: "LiveUpdate"
```

更多信息可以在 [VM 滚动更新策略](../user_workloads/vm_rollout_strategies.md) 页面找到。

### [可选] 设置最大插槽数或热插拔比例 {#optional-set-maximum-sockets-or-hotplug-ratio}
你可以用三种方式显式设置插槽的最大数量：

1. 在 VM 级别使用一个值
2. 在集群级别使用一个值
3. 在集群级别使用一个比例（`maxSockets = ratio * sockets`）。

注意：第三种方式（集群级别的比例）也会影响其他定量的热插拔资源，例如内存。


<table style="width: 100% ; display: inline-table">
<tr>
<th>
<p>
VM 级别
</p>
</th>
<th>
<p>
集群级别的值
</p>
</th>
<th>
<p>
集群级别的比例
</p>
</th>
</tr>
<tr>
<td>

```yaml
apiVersion: kubevirt.io/v1
kind: VirtualMachine
spec:
  template:
    spec:
      domain:
        cpu:
          maxSockets: 8
```
</td>
<td>

```yaml
apiVersion: kubevirt.io/v1
kind: KubeVirt
spec:
  configuration:
    liveUpdateConfiguration:
      maxCpuSockets: 8
```

</td>

<td>

```yaml
apiVersion: kubevirt.io/v1
kind: KubeVirt
spec:
  configuration:
    liveUpdateConfiguration:
      maxHotplugRatio: 4
```

</td>
</tr>
</table>

VM 级别的配置将优先于集群范围的配置。



## 热插拔流程 {#hotplug-process}
假设我们有一台正在运行的 VM，它具有 4 个 vCPU，配置为 `sockets:4 cores:1 threads:1`。
在 VMI 的 status 中，我们可以观察到该 VM 正在使用的当前 CPU 拓扑：

```yaml
apiVersion: kubevirt.io/v1
kind: VirtualMachineInstance
...
status:
  currentCPUTopology:
    cores: 1
    sockets: 4
    threads: 1
```
现在，我们要通过修补 VM 对象来热插拔另一个插槽：

```
kubectl patch vm vm-cirros --type='json' \
-p='[{"op": "replace", "path": "/spec/template/spec/domain/cpu/sockets", "value": 5}]'
```
我们可以在 VMI 的 status 中观察到 CPU 热插拔流程：

```yaml
status:
  conditions:
  - lastProbeTime: null
    lastTransitionTime: null
    status: "True"
    type: LiveMigratable
  - lastProbeTime: null
    lastTransitionTime: null
    status: "True"
    type: HotVCPUChange
  currentCPUTopology:
    cores: 1
    sockets: 4
    threads: 1
```

请注意条件 `HotVCPUChange`，它表示热插拔流程正在进行。
你还可以看到为该 VM 创建的 VirtualMachineInstanceMigration 对象：

```
NAME                             PHASE     VMI
kubevirt-workload-update-kflnl   Running   vm-cirros
```
热插拔流程完成后，`currentCPUTopology` 将更新为新的插槽数量，并且迁移被标记为成功。

```yaml
#kubectl get vmi vm-cirros -oyaml

apiVersion: kubevirt.io/v1
kind: VirtualMachineInstance
metadata:
  name: vm-cirros
spec:
  domain:
    cpu:
      cores: 1
      sockets: 5
      threads: 1
...
...
status:
  currentCPUTopology:
    cores: 1
    sockets: 5
    threads: 1


#kubectl get vmim -l kubevirt.io/vmi-name=vm-cirros
NAME                             PHASE       VMI
kubevirt-workload-update-cgdgd   Succeeded   vm-cirros
```
  
## 限制 {#limitations}
* ARM64 架构目前不支持 vCPU 热插拔。
* 当前的热插拔实现涉及 VM 工作负载的实时迁移。

### 使用 NetworkInterfaceMultiQueue {#with-networkinterfacemultiqueue}

当 VM 将 VM.spec.template.spec.domain.devices.networkInterfaceMultiQueue 设置为 `true` 时：

- 在 CPU 热插拔场景中，VM 所有者肯定会从额外的 vCPU 中获益。
  但是，在 CPU 热插拔之前已经存在的任何网络接口都将保留其初始队列数。
  这是底层虚拟化技术的特性。
- 如果 VM 所有者希望网络接口的队列数与新的 vCPU 配置对齐以最大化性能，则需要重启 VM。当然，这一决定完全由 VM 所有者做出。
- 在 CPU 热插拔之后热插拔的任何新 virtio 网络接口，其队列数都会自动配置为与更新后的 vCPU 配置相匹配。

> **注意：** 此场景在 release-1.3 和 release-1.2 中被阻止。
