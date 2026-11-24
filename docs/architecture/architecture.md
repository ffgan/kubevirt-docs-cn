# 架构 {#architecture}

KubeVirt 采用面向服务的架构和协同模式构建。

## 技术栈 {#stack}


      +---------------------+
      | KubeVirt            |
    ~~+---------------------+~~
      | 编排 (K8s)          |
      +---------------------+
      | 调度 (K8s)          |
      +---------------------+
      | 容器运行时          |
    ~~+---------------------+~~
      | 操作系统            |
      +---------------------+
      | 虚拟化(kvm)         |
    ~~+---------------------+~~
      | 物理                |
      +---------------------+

需要虚拟化服务的用户会与虚拟化 API（见下文）通信，而该 API 再与 Kubernetes 集群通信，以调度所请求的虚拟机实例（VMI）。调度、网络和存储都委托给 Kubernetes，而 KubeVirt 提供虚拟化功能。


## 附加服务 {#additional-services}

KubeVirt 为你的 Kubernetes 集群提供额外功能，用以执行虚拟机管理。

如果我们回想 Kubernetes 是如何处理 Pod 的，就会记得 Pod 是通过向 Kubernetes API Server 提交一份 Pod 规约来创建的。然后，该规约会在 API Server 内部被转换成为一个对象。这个对象具有特定的类型，也就是 _kind_——规约中就是这样称呼它的。
类型为 `Pod` 的对象就是 Pod。Kubernetes 中的控制器知道如何处理这些 Pod 对象。因此，一旦出现新的 Pod 对象，这些控制器就会执行必要的操作，使 Pod 运行起来，并匹配所需状态。

KubeVirt 使用同样的机制。因此，KubeVirt 交付三样东西来提供这项新功能：

1. 附加类型——即所谓的[自定义资源定义](https://kubernetes.io/docs/concepts/extend-kubernetes/api-extension/custom-resources/)（CRD）——被添加到 Kubernetes API
2. 与这些新类型关联的、用于集群范围逻辑的附加控制器
3. 与新类型关联的、用于节点特定逻辑的附加守护进程

这三步全部完成后，你就可以

- 在 Kubernetes 中创建这些新类型的新对象（在我们的场景中是 VMI）
- 并且新的控制器负责把 VMI 调度到某台宿主机上，
- 并且有一个守护进程——即 `virt-handler`——与 `kubelet` 一起照管一台宿主机，启动 VMI 并对其进行配置，直到它匹配所需状态。

最后说明一点：控制器和守护进程都作为 Pod（或类似对象）运行在 Kubernetes 集群 _之上_，而不是与集群并列安装。如前所述，类型甚至是在 Kubernetes API server 内部定义的。这使得用户可以与 Kubernetes 对话，但修改的是 VMI。

下图说明附加的控制器和守护进程如何与 Kubernetes 通信，以及附加类型存储在何处：

![架构图](../assets/architecture.png "架构")

以及一个简化版本：

![简化架构图](../assets/architecture-simple.png "简化架构")

## 应用布局 {#application-layout}

* 集群
  * KubeVirt 组件
    * virt-controller
    * virt-handler
    * libvirtd
    * …
  * KubeVirt 管理的 Pod
    * VMI Foo
    * VMI Bar
    * …
  * KubeVirt 自定义资源
    * VirtualMachine (VM) Foo
        -> VirtualMachineInstance (VMI) Foo
    * VirtualMachineInstanceReplicaSet (VMIRS) Bar
        -> VirtualMachineInstance (VMI) Bar

VirtualMachineInstance（VMI）是表示一个实例的基本临时构建块的自定义资源。
在很多情况下，这个对象不会由用户直接创建，而是由更高层的资源创建。
VMI 的高层资源可以是：

* VirtualMachine（VM）——有状态 VM，可以停止和启动，并保留 VM 数据和状态。
* VirtualMachineInstanceReplicaSet（VMIRS）——类似于 Pod 的 ReplicaSet，一组配置相似的临时 VMI，其配置定义在模板中。

## 原生工作负载 {#native-workloads}

KubeVirt 部署在 Kubernetes 集群之上。
这意味着你可以继续在通过 KubeVirt 管理的 VMI 旁边运行 Kubernetes 原生工作负载。

此外：如果你可以运行原生工作负载，并且已经安装了 KubeVirt，那么你也应该能够运行基于 VM 的工作负载。
例如，与把某项集群功能用于普通 Pod 相比，应用 Operator 在把该功能用于 VM 时不应需要额外权限。

从安全角度看，安装和使用 KubeVirt 不得授予用户任何他们在原生工作负载方面尚不具备的权限。例如，非特权的应用 Operator 绝不能通过使用某项 KubeVirt 功能而获得对特权 Pod 的访问。

## 剃刀 {#the-razor}

我们热爱虚拟机，认为它们非常重要，并努力让它们在 Kubernetes 中易于使用。但比起 VM，我们更热爱良好的设计，以及模块化、可复用的组件。
我们经常面临一个两难：应当以对 VM 最优的方式在 KubeVirt 中解决问题，还是应当走一条更长的路，把解决方案也引入基于 Pod 的工作负载？

为了决定这些两难，我们提出了 **KubeVirt Razor**：
“如果某项能力对 Pod 有用，我们就不应当只为 VM 实现它”。

例如，我们讨论过应当如何把 VM 连接到外部网络资源。最快的办法似乎是引入 KubeVirt 专用代码，把 VM 挂接到宿主机网桥。
然而，我们选择了与 [Multus](https://github.com/intel/multus-cni) 和 [CNI](https://github.com/containernetworking) 集成并改进它们这条更长的路。

## VirtualMachine {#virtualmachine}

`VirtualMachine` 为集群内的 VirtualMachineInstance 提供额外的管理能力。这包括：

-   API 稳定性

-   控制器层面的启动/停止/重启能力

-   离线配置变更，并在重新创建 VirtualMachineInstance 时进行传播

-   确保 VirtualMachineInstance 在应当运行时处于运行状态

它侧重于控制器实例与虚拟机实例之间的 1:1 关系。在许多方面，它非常类似于将 `spec.replica` 设置为 `1` 的 [StatefulSet](https://kubernetes.io/docs/concepts/workloads/controllers/statefulset/)。

## 如何使用 VirtualMachine {#how-to-use-a-virtualmachine}

当 VirtualMachine 处于 Running 状态时，它会确保集群中存在一个名称相同的 VirtualMachineInstance 对象。该状态通过 `spec.runStrategy` 字段控制。
有关运行策略的更多信息，请参阅[运行策略](../compute/run_strategies.md)

### 启动与停止 {#starting-and-stopping}

虚拟机可以用命令式或声明式方式开启或关闭。
将 `spec.runStrategy` 设置为 `Always` 或 `Halted` 之类的值，意味着系统会持续尝试确保虚拟机处于开启或关闭状态：

    # Start the virtual machine:
    kubectl patch virtualmachine vm --type merge -p \
        '{"spec":{"runStrategy": "Always"}}'

    # Stop the virtual machine:
    kubectl patch virtualmachine vm --type merge -p \
        '{"spec":{"runStrategy": "Halted"}}'

不过，在 `Manual` `runStrategy` 下，用户会以命令式方式自行选择何时开启或关闭 VM，系统不会执行任何自动操作：

    # Start the virtual machine:
    virtctl start vm

    # Stop the virtual machine:
    virtctl stop vm

更多细节请参见[相关章节中的 VM 生命周期](../user_workloads/lifecycle.md)

### 控制器状态 {#controller-status}

一旦创建了 VirtualMachineInstance，其状态将通过 VirtualMachine 的 `status.created` 和 `status.ready` 字段进行跟踪。如果集群中存在 VirtualMachineInstance，`status.created` 将等于 `true`。如果该 VirtualMachineInstance 也已就绪，`status.ready` 同样等于 `true`。

如果 VirtualMachineInstance 到达终态，但 `spec.runStrategy` 等于 `Always`，VirtualMachine 控制器会把 `status.ready` 设为 `false`，并重新创建 VirtualMachineInstance。

此外，`status.printableStatus` 字段提供关于 VirtualMachine 状态的高层摘要信息。使用 CLI 列出 VirtualMachine 时也会显示该信息：

```
$ kubectl get virtualmachines
NAME     AGE   STATUS    VOLUME
vm1      4m    Running
vm2      11s   Stopped
```

以下是当前支持的状态及其含义。
请注意，未来的发行版可能会增加或移除状态，因此自动化程序在使用这些状态时应谨慎。

 - **Stopped**：虚拟机当前已停止，并且预期不会启动。
 - **Provisioning**：与虚拟机关联的集群资源（例如 DataVolumes）正在被制备和准备。
 - **Starting**：虚拟机正在为运行做准备。
 - **Running**：虚拟机正在运行。
 - **Paused**：虚拟机已暂停。
 - **Migrating**：虚拟机正在被迁移到另一台宿主机。
 - **Stopping**：虚拟机正在停止。
 - **Terminating**：虚拟机正在被删除，其关联资源（VirtualMachineInstance、DataVolumes 等）也在被删除。
 - **Unknown**：无法获取虚拟机的状态，通常是因为与其运行所在的宿主机通信时出错。


### 重启 {#restarting}

可以通过删除 VirtualMachineInstance 来触发其重启。这也会传播 VirtualMachine 中模板的配置变更：

    # Restart the virtual machine (you delete the instance!):
    kubectl delete virtualmachineinstance vm

要使用 virtctl 重启名为 vm 的 VirtualMachine：

    $ virtctl restart vm

这会对 VirtualMachineInstance 执行正常重启，并在一个新的 virt-launcher Pod 上重新调度该 VirtualMachineInstance。

要使用 virtctl 强制重启名为 vm 的 VirtualMachine：

    $ virtctl restart vm --force --grace-period=0

这会尝试执行正常重启，并且还会删除该 VirtualMachineInstance 的 virt-launcher Pod，同时把 GracePeriodSeconds 设置为命令中传入的秒数。

目前仅支持设置 grace-period=0。

!!! warning "注意"
    强制重启可能导致数据损坏，应当在发生内核 panic，或 VirtualMachine 对正常重启无响应时使用。

### 隔离注意事项 {#fencing-considerations}

在当前的 VirtualMachineInstance 实例从集群中删除之前，VirtualMachine 绝不会重启或重新创建 VirtualMachineInstance。

### 暴露为 Service {#exposing-as-a-service}

VirtualMachine 可以暴露为 Service。一旦 VirtualMachineInstance 启动，实际的 Service 即可用，无需额外交互。

例如，在 VirtualMachine 已创建但尚未启动之后，将 SSH 端口（22）暴露为 `ClusterIP` Service，并使用 `virtctl`：

    $ virtctl expose virtualmachine vmi-ephemeral --name vmiservice --port 27017 --target-port 22

适用于 VirtualMachineInstance 的所有 Service 暴露选项同样适用于 VirtualMachine。

更多细节请参见 [Service 对象](../network/service_objects.md)。

## 何时使用 VirtualMachine {#when-to-use-a-virtualmachine}

### 当重启之间需要 API 稳定性时 {#when-api-stability-is-required-between-restarts}

`VirtualMachine` 确保 VirtualMachineInstance 的 API 配置在重启之间保持一致。一个典型的例子是绑定到虚拟机固件 UUID 的许可证。`VirtualMachine` 确保 UUID 始终保持不变，而用户无需自行处理。

主要好处之一是，即使用户需要稳定的 API，仍然可以使用默认值逻辑。

### 当配置更新应在下次重启时生效时 {#when-config-updates-should-be-picked-up-on-the-next-restart}

如果 VirtualMachineInstance 的配置应当可以在集群内修改，并且这些变更应当在下次 VirtualMachineInstance 重启时被采纳。这意味着不涉及热插拔。

### 当你希望让集群管理你的单个 VirtualMachineInstance 时 {#when-you-want-to-let-the-cluster-manage-your-individual-virtualmachineinstance}

Kubernetes 作为一个声明式系统，可以帮助你管理 VirtualMachineInstance。你告诉它希望这个运行着你的应用的 VirtualMachineInstance 存在，VirtualMachine 会尽量确保它保持运行。

!!! info "注意"
    当前的看法是：如果定义了 VirtualMachineInstance 应当运行，它就应当处于运行状态。这与许多传统虚拟化平台不同；在那些平台上，如果 VM 被关闭，它们会保持关闭。如有需要，可以增加重启策略。如果你需要这一点，请提供你的使用场景！

### 示例 {#example}

```yaml
apiVersion: kubevirt.io/v1
kind: VirtualMachine
metadata:
  labels:
    kubevirt.io/vm: vm-cirros
  name: vm-cirros
spec:
  runStrategy: Halted
  template:
    metadata:
      labels:
        kubevirt.io/vm: vm-cirros
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
        machine:
          type: ""
        resources:
          requests:
            memory: 64M
      terminationGracePeriodSeconds: 0
      volumes:
      - name: containerdisk
        containerDisk:
          image: kubevirt/cirros-container-disk-demo:latest
      - cloudInitNoCloud:
          userDataBase64: IyEvYmluL3NoCgplY2hvICdwcmludGVkIGZyb20gY2xvdWQtaW5pdCB1c2VyZGF0YScK
        name: cloudinitdisk
```

将此清单保存为 `vm.yaml` 并提交到 Kubernetes，将会创建控制器实例：

```
$ kubectl create -f vm.yaml
virtualmachine "vm-cirros" created
```

由于 `spec.runStrategy` 被设置为 `Halted`，不会创建 vmi：

```
$ kubectl get vmis
No resources found.
```

让我们启动这个 VirtualMachine：

```
$ virtctl start vm-cirros
```

正如预期，一个名为 `vm-cirros` 的 VirtualMachineInstance 已被创建：

```
$ kubectl describe vm vm-cirros
Name:         vm-cirros
Namespace:    default
Labels:       kubevirt.io/vm=vm-cirros
Annotations:  <none>
API Version:  kubevirt.io/v1
Kind:         VirtualMachine
Metadata:
  Cluster Name:
  Creation Timestamp:  2018-04-30T09:25:08Z
  Generation:          0
  Resource Version:    6418
  Self Link:           /apis/kubevirt.io/v1/namespaces/default/virtualmachines/vm-cirros
  UID:                 60043358-4c58-11e8-8653-525500d15501
Spec:
  Run Strategy:  Always
  Template:
    Metadata:
      Creation Timestamp:  <nil>
      Labels:
        Kubevirt . Io / Ovmi:  vm-cirros
    Spec:
      Domain:
        Devices:
          Disks:
            Disk:
              Bus:        virtio
            Name:         containerdisk
            Volume Name:  containerdisk
            Disk:
              Bus:        virtio
            Name:         cloudinitdisk
            Volume Name:  cloudinitdisk
        Machine:
          Type:
        Resources:
          Requests:
            Memory:                      64M
      Termination Grace Period Seconds:  0
      Volumes:
        Name:  containerdisk
        Registry Disk:
          Image:  kubevirt/cirros-registry-disk-demo:latest
        Cloud Init No Cloud:
          User Data Base 64:  IyEvYmluL3NoCgplY2hvICdwcmludGVkIGZyb20gY2xvdWQtaW5pdCB1c2VyZGF0YScK
        Name:                 cloudinitdisk
Status:
  Created:  true
  Ready:    true
Events:
  Type    Reason            Age   From                              Message
  ----    ------            ----  ----                              -------
  Normal  SuccessfulCreate  15s   virtualmachine-controller  Created virtual machine: vm-cirros
```

### Kubectl 命令行交互 {#kubectl-commandline-interactions}

每当你想通过命令行操作 VirtualMachine 时，都可以使用 kubectl 命令。以下示例演示了具体做法。

```
    # Define a virtual machine:
    kubectl create -f vm.yaml

    # Start the virtual machine:
    kubectl patch virtualmachine vm --type merge -p \
        '{"spec":{"runStrategy":"Always"}}'

    # Look at virtual machine status and associated events:
    kubectl describe virtualmachine vm

    # Look at the now created virtual machine instance status and associated events:
    kubectl describe virtualmachineinstance vm

    # Stop the virtual machine instance:
    kubectl patch virtualmachine vm --type merge -p \
        '{"spec":{"runStrategy":"Halted"}}'

    # Restart the virtual machine (you delete the instance!):
    kubectl delete virtualmachineinstance vm

    # Implicit cascade delete (first deletes the virtual machine and then the virtual machine instance)
    kubectl delete virtualmachine vm

    # Explicit cascade delete (first deletes the virtual machine and then the virtual machine instance)
    kubectl delete virtualmachine vm --cascade=true

    # Orphan delete (The running virtual machine is only detached, not deleted)
    # Recreating the virtual machine would lead to the adoption of the virtual machine instance
    kubectl delete virtualmachine vm --cascade=false
```
