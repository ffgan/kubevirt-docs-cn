# 节点维护 {#node-maintenance}

在从集群中移除 Kubernetes 节点之前，用户会希望
确保 VirtualMachineInstance 已优雅终止，
然后再关闭该节点的电源。由于所有 VirtualMachineInstance
都由 Pod 支撑，驱逐
VirtualMachineInstance 的推荐方法是使用 **kubectl drain** 命令，在
OKD 的情况下则使用 **oc adm drain** 命令。

## 从节点上驱逐所有 VM {#evict-all-vms-from-a-node}

通过在集群节点列表中确认节点，
选择你希望从中驱逐 VirtualMachineInstance 的节点。

`kubectl get nodes`

以下命令将优雅终止特定
节点上的所有 VM。将 `<node-name>` 替换为应当发生驱逐的节点名称。

`kubectl drain <node-name> --delete-local-data --ignore-daemonsets=true --force --pod-selector=kubevirt.io=virt-launcher`

下面说明传给 drain 命令的每个参数
为何是必需的。

-   `kubectl drain <node-name>` 会选择特定节点作为
    驱逐目标

-   `--delete-local-data` 是必需标志，用于
    移除任何使用 emptyDir 卷的 Pod。
    VirtualMachineInstance Pod 确实使用 emptyDir 卷，不过
    这些卷中的数据是临时的，这意味着终止后
    可以安全删除。

-   `--ignore-daemonsets=true` 是必需标志，因为每个
    运行 VirtualMachineInstance 的节点也会运行名为 virt-handler 的辅助
    DaemonSet。不允许使用 **kubectl drain** 驱逐 DaemonSet。默认情况下，如果此命令
    在目标节点上遇到 DaemonSet，命令将会失败。
    此标志告诉命令可以安全地继续驱逐，
    并忽略 DaemonSet。

-   `--force` 是必需标志，因为 VirtualMachineInstance Pod
    并不由 ReplicaSet 或 DaemonSet 控制器拥有。这意味着
    kubectl 无法保证目标节点上被终止的 Pod
    在被驱逐后，会在集群的其他位置
    得到重新调度的替代 Pod。KubeVirt 有自己的控制器
    来管理底层的 VirtualMachineInstance Pod。每个
    控制器对被驱逐的 VirtualMachineInstance
    的行为各不相同。该行为在本文档后面进一步说明。

-   `--pod-selector=kubevirt.io=virt-launcher` 表示只会从
    该节点移除由 KubeVirt 管理的
    VirtualMachineInstance Pod。

## 从节点上驱逐所有 VM 和 Pod {#evict-all-vms-and-pods-from-a-node}

从前一条命令中去掉 `-pod-selector` 参数，
就可以对该节点上的所有 Pod 发起驱逐。此命令确保与 VM
关联的 Pod 以及所有其他 Pod 都会从
目标节点上被驱逐。

`kubectl drain <node name> --delete-local-data --ignore-daemonsets=true --force`

## 通过实时迁移从节点上疏散 VMI {#evacuate-vmis-via-live-migration-from-a-node}

如果启用了 `LiveMigration`
[功能门控](../cluster_admin/activating_feature_gates.md#how-to-activate-a-feature-gate)，
就可以在 VMI 上指定 `evictionStrategy`，使其在节点出现
特定污点时以实时迁移作为响应。VMI 上或 VM 中 VMI 模板上的
以下片段可确保在节点驱逐期间迁移该 VMI：

    spec:
      evictionStrategy: LiveMigrate


下面是一个完整的 VMI：

```yaml
apiVersion: kubevirt.io/v1
kind: VirtualMachineInstance
metadata:
  name: testvmi-nocloud
spec:
  terminationGracePeriodSeconds: 30
  evictionStrategy: LiveMigrate
  domain:
    resources:
      requests:
        memory: 1024M
    devices:
      disks:
      - name: containerdisk
        disk:
          bus: virtio
      - disk:
          bus: virtio
        name: cloudinitdisk
  volumes:
  - name: containerdisk
    containerDisk:
      image: kubevirt/fedora-cloud-container-disk-demo:latest
  - name: cloudinitdisk
    cloudInitNoCloud:
      userData: |-
        #cloud-config
        password: fedora
        chpasswd: { expire: False }
```

在幕后，会为每个定义了 **evictionStrategy** 的 VMI
创建一个 **PodDisruptionBudget**。这确保
这些 VMI 上的驱逐会被阻止，从而我们可以保证 VMI
将被迁移，而不是被关闭。

**注意** 在 v0.34 之前，带实时迁移的排空过程与
`kubectl drain` 本身是分离的，并且还需要在节点上额外指定一个特殊污点：
`kubectl taint nodes foo kubevirt.io/drain=draining:NoSchedule`。
现在不再需要这样做。如果提供了该污点，它仍会被遵守，但已经
**过时**。

## 驱逐后重新启用节点 {#re-enabling-a-node-after-eviction}

**kubectl drain** 会使目标节点被标记为
不可调度。这意味着该节点将不能运行新的
VirtualMachineInstance 或 Pod。

如果决定目标节点应当重新变为可调度，
则必须运行以下命令。

`kubectl uncordon <node name>`

或者在 OKD 的情况下。

`oc adm uncordon <node name>`

## 驱逐后关闭节点 {#shutting-down-a-node-after-eviction}

从 KubeVirt 的角度来看，一旦所有
VirtualMachineInstance 都已从该节点上被驱逐，就可以安全关闭该节点。在
VirtualMachineInstance 与
其他容器化工作负载一起调度的多用途
集群中，集群管理员需要负责确保
在关闭节点电源之前，所有其他 Pod 都已安全驱逐。

## VirtualMachine 驱逐 {#virtualmachine-evictions}

驱逐任何由设置为 **running=true** 的
VirtualMachine 所拥有的 VirtualMachineInstance，都会导致
该 VirtualMachineInstance 被重新调度到另一个节点。

在这种情况下，VirtualMachineInstance 将被强制关机，并
在另一个节点上重新启动。将来一旦 KubeVirt 引入实时
迁移支持，VM 就能在驱逐期间无缝迁移到另一个
节点。

### VirtualMachineInstanceReplicaSet 的驱逐行为 {#virtualmachineinstancereplicaset-eviction-behavior}

驱逐由
VirtualMachineInstanceReplicaSet 拥有的 VirtualMachineInstance，会导致
VirtualMachineInstanceReplicaSet 在集群中的其他节点上为被驱逐的
VirtualMachineInstance 调度替代实例。

### VirtualMachineInstance 的驱逐行为 {#virtualmachineinstance-eviction-behavior}

既没有
VirtualMachineInstanceReplicaSet、也没有 VirtualMachine 对象支撑的
VirtualMachineInstance，在驱逐后不会被
重新调度。

## 为分组 VM 工作负载自定义 PodDisruptionBudget {#custom-poddisruptionbudgets-for-grouped-vm-workloads}

当你定义自己的 `PodDisruptionBudget`（PDB）来保护一组 VM 时——
例如 `virt-launcher` 这类支撑 `VirtualMachinePool` 的 Pod——请使用
`spec.minAvailable`，而不要使用 `spec.maxUnavailable`。

### 为什么 `maxUnavailable` 不起作用 {#why-maxunavailable-does-not-work}

`virt-launcher` Pod 由 `VirtualMachineInstance`（VMI）直接拥有：

```
virt-launcher Pod  →  VirtualMachineInstance   (direct owner)
                   →  VirtualMachine
                   →  VirtualMachinePool
```


为了遵守 `maxUnavailable`，Kubernetes 干扰控制器必须知道
预期的 Pod 数量，它通过调用 `/scale` 子资源来获取该数量，调用对象是
Pod 的直接所有者——`VirtualMachineInstance`。VMI 是单个、
非副本实例，并且不实现 `/scale`，因此控制器无法
计算预期数量：

```
Warning  CalculateExpectedPodCountFailed
  Failed to calculate the number of expected pods:
  virtualmachineinstances.kubevirt.io does not implement the scale subresource
```


随后 PDB 会报告 `status.expectedPods: 0`、`status.disruptionsAllowed: 0` 以及
`DisruptionAllowed: False`——这会阻止**每一次**排空或自愿驱逐，无论
有多少 VM 处于健康状态。

相反，`spec.minAvailable` 不使用 `/scale` 子资源；它直接统计
当前健康的匹配 Pod，并且可以与 KubeVirt
工作负载正确配合。

!!! note
    此限制跟踪于
    [kubevirt/kubevirt#18063](https://github.com/kubevirt/kubevirt/issues/18063)。上游正在推进一项
    长期的平台级修复，见
    [kubernetes/kubernetes#139582](https://github.com/kubernetes/kubernetes/issues/139582)
    （让干扰控制器沿所有者链遍历到可扩缩的
    祖先对象）。在该修复落地之前，请优先使用下文所述的 `minAvailable`。

### 推荐配置 {#recommended-configuration}

将 `minAvailable` 设置为工作负载在干扰期间必须保持
运行的最小实例数——即应用程序所需的可用性下限——
并以**绝对整数**表示：

```yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: vmpool-pdb
spec:
  minAvailable: 3            # keep at least 3 VMs of the pool running at all times
  selector:
    matchLabels:
      kubevirt.io/vmpool: my-pool   # match the virt-launcher pods of the pool
```


该值应来自应用程序真实的可用性要求，而不是
机械地取 `replicas - 1`。固定的 `N-1` 不会随池的扩缩而变化：
扩容之后它允许的并发干扰会远超预期，
缩容之后它可能完全阻止驱逐。请选取你需要保留的实际最小
容量，并在显著调整池大小时重新审视该值。

!!! warning
    请使用**绝对整数**，不要使用百分比。百分比形式的 `minAvailable`
    （例如 `minAvailable: 75%`）会使干扰控制器通过 `/scale` 子资源解析总
    副本数——而 `VirtualMachineInstance`
    并未实现该子资源——因此它会以与 `maxUnavailable` 相同的方式失败。只有
    整数形式的 `minAvailable` 才会直接统计健康 Pod，并避免 scale 查找。

避免对由 VirtualMachineInstance 支撑的 Pod 使用 `maxUnavailable`：

```yaml
spec:
  maxUnavailable: 1          # results in disruptionsAllowed: 0 (see above)
```


!!! note
    可以使用 `spec.maxUnavailable`，把它配置在 `VirtualMachinePool` 对象本身上，因为
    它由 VMPool 控制器直接读取。上述限制仅适用于
    `PodDisruptionBudget` 的 `maxUnavailable` 字段。

### 验证 {#verifying}

```console
$ kubectl get pdb vmpool-pdb -o wide
NAME         MIN AVAILABLE   MAX UNAVAILABLE   ALLOWED DISRUPTIONS   AGE
vmpool-pdb   1               N/A               1                     2m
```


`ALLOWED DISRUPTIONS` 应当为 `>= 1`。如果你看到 `0`，并且伴随
`CalculateExpectedPodCountFailed` 事件，则说明该 PDB 使用了 `maxUnavailable`——请将其
改为 `minAvailable`。
