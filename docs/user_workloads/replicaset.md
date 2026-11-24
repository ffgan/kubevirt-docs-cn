# VirtualMachineInstanceReplicaSet {#virtualmachineinstancereplicaset}

*VirtualMachineInstanceReplicaSet* 会设法确保在任何时候都有指定数量的
VirtualMachineInstance 副本正在运行。换言之，*VirtualMachineInstanceReplicaSet*
确保一个 VirtualMachineInstance 或一组同构的 VirtualMachineInstance
始终处于运行并就绪的状态。它与 [Kubernetes
ReplicaSet](https://kubernetes.io/docs/concepts/workloads/controllers/replicaset/) 非常相似。

系统不保存状态，也不对处于运行状态的
VirtualMachineInstance 副本数量上限给出保证。例如，如果可能仍在运行的 VM 进入
未知状态，*VirtualMachineInstanceReplicaSet* 可能会决定创建新的副本。


## 使用 VirtualMachineInstanceReplicaSet {#using-virtualmachineinstancereplicaset}

*VirtualMachineInstanceReplicaSet* 允许我们在 `spec.template` 中指定
*VirtualMachineInstanceTemplate*。它由 `ObjectMetadata`（位于
`spec.template.metadata` 中）以及 `VirtualMachineInstanceSpec`（位于
`spec.template.spec` 中）组成。虚拟机的规格与
`VirtualMachineInstance` 工作负载中的虚拟机规格相同。

`spec.replicas` 可用于指定所需的副本数量。如果
未指定，默认值为 1。该值可以随时更新。
控制器会对这些变更作出反应。

`spec.selector` 由控制器用来跟踪所管理的
虚拟机。此处指定的选择器必须能够匹配
`spec.template.metadata.labels` 中指定的虚拟机标签。
如果选择器与这些标签不匹配，或者这些标签为空，
控制器除了记录一条错误日志之外不会做任何事情。用户
有责任不要创建与该选择器及
模板标签相冲突的其他虚拟机或
*VirtualMachineInstanceReplicaSet*。


## 将 VirtualMachineInstanceReplicaSet 暴露为 Service {#exposing-a-virtualmachineinstancereplicaset-as-a-service}

VirtualMachineInstanceReplicaSet 可以被暴露为服务。完成此操作后，
会从 VirtualMachineInstance 副本中挑选一个，
用于实际提供该服务。

例如，使用 virtctl 在 VirtualMachineInstanceReplicaSet 上将 SSH 端口（22）暴露为 ClusterIP 服务：

    $ virtctl expose vmirs vmi-ephemeral --name vmiservice --port 27017 --target-port 22

适用于 VirtualMachineInstance 的所有服务暴露选项
同样适用于 VirtualMachineInstanceReplicaSet。更多细节请参阅[暴露
VirtualMachineInstance](../network/service_objects.md)。


## 何时使用 VirtualMachineInstanceReplicaSet {#when-to-use-a-virtualmachineinstancereplicaset}

> **注意：** 基本假设是所引用的磁盘为只读，
> 或者 VMI 在内部写入 tmpfs。KubeVirt
> 所支持的、适合 VirtualMachineInstanceReplicaSet 的最常见
> 卷源如下所示。如果使用其他类型，则可能发生**数据
> 损坏**。

当需要许多相同的 VM，并且不关心 VM 终止之后
是否保留任何磁盘状态时，使用 VirtualMachineInstanceReplicaSet 是正确的选择。

与 VirtualMachineInstanceReplicaSet 配合良好的[卷类型](../storage/disks_and_volumes.md)包括：

-   **cloudInitNoCloud**
-   **ephemeral**
-   **containerDisk**
-   **emptyDisk**
-   **configMap**
-   **secret**
-   任何其他类型，前提是 VMI 在内部写入 tmpfs

### 快速启动的临时虚拟机 {#fast-starting-ephemeral-virtual-machines}

此用例涉及小型、启动迅速的 VM，初始化期间
只进行很少的置备。

在此场景中，迁移并不重要。只需删除运行在过载节点上、
受管理的 VirtualMachineInstance，即可在节点之间重新分布 VM
工作负载。对此类 VirtualMachineInstance 的 `eviction`
可以通过直接删除该 VirtualMachineInstance 实例来实现（感知 KubeVirt 的工作负载
重新分布），也可以通过删除虚拟机所运行于其中的对应 Pod 来实现（仅感知 Kubernetes 的工作负载重新分布）。

### 启动缓慢的临时虚拟机 {#slow-starting-ephemeral-virtual-machines}

在此用例中，VM 体积大、启动慢，并且在启动期间会进行复杂或
资源密集的置备。更具体地说，
从创建新 VM 到其进入就绪
状态之间的时间跨度很长。

在此场景中，用户仍然不关心状态，但由于
重新置备 VM 的代价很高，迁移就很重要。节点之间的工作负载
重新分布可以通过将
VirtualMachineInstance 迁移到不同节点来实现。工作负载再分布程序
需要感知 KubeVirt 并创建迁移，而不是
通过删除来 `evicting` VirtualMachineInstance。

> **注意：** 拥有可迁移的临时
> VirtualMachineInstance 的最简单形式，是结合基于文件的后端存储，使用基于
> `ContainerDisks` 的本地存储。
> 不过，可迁移后端存储的支持尚未正式进入
> KubeVirt，并且未经测试。

#### 示例 {#example}

```yaml
apiVersion: kubevirt.io/v1
kind: VirtualMachineInstanceReplicaSet
metadata:
  name: testreplicaset
spec:
  replicas: 3
  selector:
    matchLabels:
      myvmi: myvmi
  template:
    metadata:
      name: test
      labels:
        myvmi: myvmi
    spec:
      domain:
        devices:
          disks:
          - disk:
            name: containerdisk
        resources:
          requests:
            memory: 64M
      volumes:
      - name: containerdisk
        containerDisk:
          image: kubevirt/cirros-container-disk-demo:latest
```
将此清单保存为 `testreplicaset.yaml` 并提交到
Kubernetes，将会基于该模板创建三台虚拟机。

    $ kubectl create -f testreplicaset.yaml
    virtualmachineinstancereplicaset "testreplicaset" created
    $ kubectl describe vmirs testreplicaset
    Name:         testreplicaset
    Namespace:    default
    Labels:       <none>
    Annotations:  <none>
    API Version:  kubevirt.io/v1
    Kind:         VirtualMachineInstanceReplicaSet
    Metadata:
      Cluster Name:
      Creation Timestamp:  2018-01-03T12:42:30Z
      Generation:          0
      Resource Version:    6380
      Self Link:           /apis/kubevirt.io/v1/namespaces/default/virtualmachineinstancereplicasets/testreplicaset
      UID:                 903a9ea0-f083-11e7-9094-525400ee45b0
    Spec:
      Replicas:  3
      Selector:
        Match Labels:
          Myvmi:  myvmi
      Template:
        Metadata:
          Creation Timestamp:  <nil>
          Labels:
            Myvmi:  myvmi
          Name:    test
        Spec:
          Domain:
            Devices:
              Disks:
                Disk:
                Name:         containerdisk
                Volume Name:  containerdisk
            Resources:
              Requests:
                Memory:  64M
          Volumes:
            Name:  containerdisk
            Container Disk:
              Image:  kubevirt/cirros-container-disk-demo:latest
    Status:
      Conditions:      <nil>
      Ready Replicas:  2
      Replicas:        3
    Events:
      Type    Reason            Age   From                                 Message
      ----    ------            ----  ----                                 -------
      Normal  SuccessfulCreate  13s   virtualmachineinstancereplicaset-controller  Created virtual machine: testh8998
      Normal  SuccessfulCreate  13s   virtualmachineinstancereplicaset-controller  Created virtual machine: testf474w
      Normal  SuccessfulCreate  13s   virtualmachineinstancereplicaset-controller  Created virtual machine: test5lvkd

`Replicas` 为 `3`，`Ready Replicas` 为 `2`。这意味着在
显示该状态的那一刻，三台虚拟机已经
创建，但只有两台正在运行并就绪。

### 通过 Scale 子资源进行扩缩 {#scaling-via-the-scale-subresource}

> **注意：** 对于 1.11 之前的集群，这需要启用 `CustomResourceSubresources`
> [功能门控](../cluster_admin/activating_feature_gates.md#how-to-activate-a-feature-gate)。

`VirtualMachineInstanceReplicaSet` 支持 `scale` 子资源。
因此可以通过 `kubectl` 对其进行扩缩：

    $ kubectl scale vmirs myvmirs --replicas 5


### 使用 Horizontal Pod Autoscaler {#using-the-horizontal-pod-autoscaler}

> **注意：** 这要求集群版本不低于 1.11。

[HorizontalPodAutoscaler](https://kubernetes.io/docs/tasks/run-application/horizontal-pod-autoscale/)
（HPA）可以与 `VirtualMachineInstanceReplicaSet` 一起使用。只需
在自动扩缩器的 spec 中引用它：

```yaml
apiVersion: autoscaling/v1
kind: HorizontalPodAutoscaler
metadata:
  name: myhpa
spec:
  scaleTargetRef:
    kind: VirtualMachineInstanceReplicaSet
    name: vmi-replicaset-cirros
    apiVersion: kubevirt.io/v1
  minReplicas: 3
  maxReplicas: 10
  targetCPUUtilizationPercentage: 50
```

或者使用 `kubectl autoscale` 通过命令行定义 HPA：

    $ kubectl autoscale vmirs vmi-replicaset-cirros --min=3 --max=10
