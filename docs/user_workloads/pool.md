# VirtualMachinePool {#virtualmachinepool}

*VirtualMachinePool* 会设法确保在任何时候都有指定数量的
VirtualMachine 副本及其各自的
VirtualMachineInstance 处于就绪状态。
换言之，*VirtualMachinePool* 确保一个
VirtualMachine 或一组 VirtualMachine 始终处于运行
并就绪的状态。

系统不保存状态，也不对任何时刻正在运行的
VirtualMachineInstance 副本数量上限作出保证。
例如，如果可能仍在运行的 VM 进入
未知状态，*VirtualMachinePool* 可能会决定创建
新的副本。


## 使用 VirtualMachinePool {#using-virtualmachinepool}

*VirtualMachinePool* 允许我们在 `spec.virtualMachineTemplate` 中指定 *VirtualMachineTemplate*。
它由 `ObjectMetadata`（位于 `spec.virtualMachineTemplate.metadata` 中）
以及 `VirtualMachineSpec`（位于 `spec.virtualMachineTemplate.spec` 中）组成。虚拟机的规格
与 `VirtualMachine` 工作负载中的虚拟机规格相同。

`spec.replicas` 可用于指定所需的副本数量。如果
未指定，默认值为 1。该值可以随时更新。
控制器会对这些变更作出反应。

`spec.selector` 由控制器用来跟踪所管理的
虚拟机。此处指定的选择器必须能够匹配
`spec.virtualMachineTemplate.metadata.labels` 中指定的虚拟机标签。
如果选择器与这些标签不匹配，或者这些标签为空，
控制器除了记录一条错误日志之外不会做任何事情。用户
有责任避免创建可能与该选择器及
模板标签相冲突的其他虚拟机或
*VirtualMachinePool*。

### 创建 VirtualMachinePool {#creating-a-virtualmachinepool}

VirtualMachinePool 属于 KubeVirt API `pool.kubevirt.io/v1alpha1`。

下面的示例展示如何创建一个简单的 `VirtualMachinePool`：

#### 示例 {#example}

```yaml
apiVersion: pool.kubevirt.io/v1alpha1
kind: VirtualMachinePool
metadata:
  name: vm-pool-cirros
spec:
  replicas: 3
  selector:
    matchLabels:
      kubevirt.io/vmpool: vm-pool-cirros
  virtualMachineTemplate:
    metadata:
      creationTimestamp: null
      labels:
        kubevirt.io/vmpool: vm-pool-cirros
    spec:
      runStrategy: Always
      template:
        metadata:
          creationTimestamp: null
          labels:
            kubevirt.io/vmpool: vm-pool-cirros
        spec:
          domain:
            devices:
              disks:
              - disk:
                  bus: virtio
                name: containerdisk
            resources:
              requests:
                memory: 128Mi
          terminationGracePeriodSeconds: 0
          volumes:
          - containerDisk:
              image: kubevirt/cirros-container-disk-demo:latest
            name: containerdisk 
```

将此清单保存为 `vm-pool-cirros.yaml` 并提交到
Kubernetes，将会基于该模板创建三台虚拟机。

    $ kubectl create -f vm-pool-cirros.yaml
    virtualmachinepool.pool.kubevirt.io/vm-pool-cirros created
    $ kubectl describe vmpool vm-pool-cirros
    Name:         vm-pool-cirros
    Namespace:    default
    Labels:       <none>
    Annotations:  <none>
    API Version:  pool.kubevirt.io/v1alpha1
    Kind:         VirtualMachinePool
    Metadata:
      Creation Timestamp:  2023-02-09T18:30:08Z
      Generation:          1
        Manager:      kubectl-create
        Operation:    Update
        Time:         2023-02-09T18:30:08Z
        API Version:  pool.kubevirt.io/v1alpha1
        Fields Type:  FieldsV1
        fieldsV1:
          f:status:
            .:
            f:labelSelector:
            f:readyReplicas:
            f:replicas:
        Manager:         virt-controller
        Operation:       Update
        Subresource:     status
        Time:            2023-02-09T18:30:44Z
      Resource Version:  6606
      UID:               ba51daf4-f99f-433c-89e5-93f39bc9989d
    Spec:
      Replicas:  3
      Selector:
        Match Labels:
          kubevirt.io/vmpool:  vm-pool-cirros
      Virtual Machine Template:
        Metadata:
          Creation Timestamp:  <nil>
          Labels:
            kubevirt.io/vmpool:  vm-pool-cirros
        Spec:
          Running:  true
          Template:
            Metadata:
              Creation Timestamp:  <nil>
              Labels:
                kubevirt.io/vmpool:  vm-pool-cirros
            Spec:
              Domain:
                Devices:
                  Disks:
                    Disk:
                      Bus:  virtio
                    Name:   containerdisk
                Resources:
                  Requests:
                    Memory:                      128Mi
              Termination Grace Period Seconds:  0
              Volumes:
                Container Disk:
                  Image:  kubevirt/cirros-container-disk-demo:latest
                Name:     containerdisk
    Status:
      Label Selector:  kubevirt.io/vmpool=vm-pool-cirros
      Ready Replicas:  2
      Replicas:        3
    Events:
      Type    Reason            Age   From                           Message
      ----    ------            ----  ----                           -------
      Normal  SuccessfulCreate  17s   virtualmachinepool-controller  Created VM default/vm-pool-cirros-0
      Normal  SuccessfulCreate  17s   virtualmachinepool-controller  Created VM default/vm-pool-cirros-2
      Normal  SuccessfulCreate  17s   virtualmachinepool-controller  Created VM default/vm-pool-cirros-1

`Replicas` 为 `3`，`Ready Replicas` 为 `2`。这意味着在
显示该状态的那一刻，三台虚拟机已经
创建，但只有两台正在运行并就绪。

### 通过 Scale 子资源进行扩缩 {#scaling-via-the-scale-subresource}

> **注意：** 这要求 KubeVirt 0.59 或更新版本。

`VirtualMachinePool` 支持 `scale` 子资源。
因此可以通过 `kubectl` 对其进行扩缩：

    $ kubectl scale vmpool vm-pool-cirros --replicas 5


### 从 VirtualMachinePool 中移除 VirtualMachine {#removing-a-virtualmachine-from-virtualmachinepool}

也可以把一个 `VirtualMachine` 从它所属的 `VirtualMachinePool` 中移除。

在此场景中，`ownerReferences` 需要从该 `VirtualMachine` 上移除。
这可以通过 `kubectl edit` 或 `kubectl patch` 来完成。
使用 `kubectl patch` 时如下所示：

    kubectl patch vm vm-pool-cirros-0 --type merge --patch '{"metadata":{"ownerReferences":null}}' 

> **注意：** 你可能还需要更新 VirtualMachine 的标签，以避免对选择器造成影响。


### 使用 Horizontal Pod Autoscaler {#using-the-horizontal-pod-autoscaler}

> **注意：** 这要求 KubeVirt 0.59 或更新版本。

[HorizontalPodAutoscaler](https://kubernetes.io/docs/tasks/run-application/horizontal-pod-autoscale/)
（HPA）可以与 `VirtualMachinePool` 一起使用。只需
在自动扩缩器的 spec 中引用它：

```yaml
apiVersion: autoscaling/v1
kind: HorizontalPodAutoscaler
metadata:
  creationTimestamp: null
  name: vm-pool-cirros
spec:
  maxReplicas: 10
  minReplicas: 3
  scaleTargetRef:
    apiVersion: pool.kubevirt.io/v1alpha1
    kind: VirtualMachinePool
    name: vm-pool-cirros
  targetCPUUtilizationPercentage: 50
```


或者使用 `kubectl autoscale` 通过命令行定义 HPA：

    $ kubectl autoscale vmpool vm-pool-cirros --min=3 --max=10 --cpu-percent=50

## 将 VirtualMachinePool 暴露为 Service {#exposing-a-virtualmachinepool-as-a-service}

VirtualMachinePool 可以被暴露为服务。完成此操作后，
会从 VirtualMachine 副本中挑选一个，
用于实际提供该服务。

例如，将 SSH 端口（22）暴露为 ClusterIP 服务：

```yaml
apiVersion: v1
kind: Service
metadata:
  name: vm-pool-cirros-ssh
spec:
  type: ClusterIP
  selector:
    kubevirt.io/vmpool: vm-pool-cirros
  ports:
    - protocol: TCP
      port: 2222
      targetPort: 22
```
将此清单保存为 `vm-pool-cirros-ssh.yaml` 并提交到
Kubernetes，将会创建在端口 2222 上监听、
并转发到端口 22 的 `ClusterIP` 服务。

更多细节请参阅[服务
对象](http://kubevirt.io/user-guide/virtual_machines/service_objects/)。


## 使用持久存储 {#using-persistent-storage}

> **注意：** DataVolume 是 [CDI](https://kubevirt.io/user-guide/operations/containerized_data_importer/) 的一部分

`DataVolumeTemplates` 在 `spec.virtualMachineTemplate.spec` 中的使用，会为 VMPool 中的每个 VM 创建
独有的持久存储。`DataVolumeTemplate`
的名称会在该 VM 从 `spec.virtualMachineTemplate.spec.dataVolumeTemplates` 创建时追加该 VM 的顺序后缀。这使得每个 VM 都成为完全独有的有状态工作负载。


## 在 VirtualMachinePool 中使用互不相同的 CloudInit 与 ConfigMap 卷 {#using-unique-cloudinit-and-configmap-volumes-with-virtualmachinepools}

默认情况下，`spec.virtualMachineTemplate.spec.template`
卷部分中的任何 secret 或 configMap 引用都会按原样直接使用，不会对名称做任何修改。这
意味着，如果你在 `CloudInitNoCloud` 卷中指定了一个 secret，那么从带有该卷的 VirtualMachinePool
生成的每一个 VM 实例都会把完全相同的 secret 用于它们的 cloud-init
用户数据。

可以通过把 `AppendIndexToSecretRefs` 和
`AppendIndexToConfigMapRefs` 这两个布尔值都设置为 `true`（位于 VMPool 定义的 `spec.NameGeneration` 中）来改变这一默认行为。
启用这两个布尔值后，对 secret 和 configMap 名称的引用会把该 VM 的顺序索引
追加到 secret 和 configMap 名称上。这样就可以预先为 VirtualMachinePool 中的每个 VM
生成互不相同的 `secret` 和 `configMap` 数据，并以可预期的方式
分配给该 VirtualMachinePool 中的各个 VM。
