# 生命周期 {#lifecycle}

每个 `VirtualMachineInstance` 代表一个虚拟机*实例*。一般来说，VirtualMachineInstance 的管理方式与 `Pods` 的管理方式保持相似：集群中定义的每个 VM 都应当处于运行状态，就像 Pod 一样。删除 VirtualMachineInstance 等同于将其关闭，这也与 Pod 的行为一致。


## 启动虚拟机 {#launching-a-virtual-machine}

要启动 VirtualMachineInstance，只需创建一个 `VirtualMachineInstance` 对象，并使用 `kubectl`：

    $ kubectl create -f vmi.yaml


## 列出虚拟机 {#listing-virtual-machines}

可以通过查询 VirtualMachineInstance 对象来列出 VirtualMachineInstance：

    $ kubectl get vmis


## 获取虚拟机实例定义 {#retrieving-a-virtual-machine-instance-definition}

可以通过获取特定的 VirtualMachineInstance 对象来取得单个 VirtualMachineInstance 定义：

    $ kubectl get vmis testvmi


## 停止虚拟机实例 {#stopping-a-virtual-machine-instance}

要停止 VirtualMachineInstance，只需删除对应的 `VirtualMachineInstance` 对象，并使用 `kubectl`。

    $ kubectl delete -f vmi.yaml
    # OR
    $ kubectl delete vmis testvmi

> **注意：** 停止 VirtualMachineInstance 意味着它会从集群中删除。你将无法再次启动该 VirtualMachineInstance 对象。

## 启动和停止虚拟机 {#starting-and-stopping-a-virtual-machine}

虚拟机与 VirtualMachineInstance 不同，具有运行状态。因此你可以在 VM 上定义它是否应当运行。只要 VirtualMachineInstance 在集群中被定义，就会一直运行并消耗资源。

使用 `virtctl` 来启动和停止 VirtualMachine：

    $ virtctl start my-vm
    $ virtctl stop my-vm
    
> **注意：** 你可以强制停止 VM（这如同拔掉电源线，并带来相应后果，例如数据不一致，或在最坏情况下的数据丢失），方法是

    $ virtctl stop my-vm --grace-period 0 --force

## 暂停和恢复虚拟机 {#pausing-and-unpausing-a-virtual-machine}

> **注意：** 此处的暂停指的是 libvirt 的 `virDomainSuspend` 命令：  
> “进程被冻结，不再继续访问 CPU 资源和 I/O，但该 domain 在 hypervisor 层所使用的内存仍会保持分配”

要暂停虚拟机，需要 `virtctl` 命令行工具。它的 `pause` 命令可用于 `VirtualMachine` 或 `VirtualMachinesInstance`：

    $ virtctl pause vm testvm
    # OR
    $ virtctl pause vmi testvm

已暂停的 VMI 在其状态中带有 `Paused` 状况：

    $ kubectl get vmi testvm -o=jsonpath='{.status.conditions[?(@.type=="Paused")].message}'
    VMI was paused by user

恢复暂停的方式与暂停类似：

    $ virtctl unpause vm testvm
    # OR
    $ virtctl unpause vmi testvm
