# 虚拟机实例 {#virtual-machines-instances}

`VirtualMachineInstance` 类型在概念上包含两部分：

-   用于做出调度决策的信息

-   关于虚拟机 API 的信息

每个 `VirtualMachineInstance` 对象代表一个正在运行的虚拟机实例。

## API 概览 {#api-overview}

安装 KubeVirt 后，Kubernetes API 中会新增用于管理虚拟机的类型。

你可以像操作其他任何 API 资源一样（通过 `kubectl`）与这些新资源交互。

## VirtualMachineInstance API {#virtualmachineinstance-api}

> 注意：完整的 API 参考位于
> <https://kubevirt.io/api-reference/>。

以下是 VirtualMachineInstance 对象的示例：

```yaml
apiVersion: kubevirt.io/v1
kind: VirtualMachineInstance
metadata:
  name: testvmi-nocloud
spec:
  terminationGracePeriodSeconds: 30
  domain:
    resources:
      requests:
        memory: 1024M
    devices:
      disks:
      - name: containerdisk
        disk:
          bus: virtio
      - name: emptydisk
        disk:
          bus: virtio
      - disk:
          bus: virtio
        name: cloudinitdisk
  volumes:
  - name: containerdisk
    containerDisk:
      image: kubevirt/fedora-cloud-container-disk-demo:latest
  - name: emptydisk
    emptyDisk:
      capacity: "2Gi"
  - name: cloudinitdisk
    cloudInitNoCloud:
      userData: |-
        #cloud-config
        password: fedora
        chpasswd: { expire: False }
```

此示例结合使用 fedora cloud 镜像与 cloud-init，以及一块容量为 `2Gi` 的临时空磁盘。为简单起见，此示例中的卷源是临时的，不需要集群中的制备器。

## 更多信息 {#additional-information}

-   在 VirtualMachine 中使用实例类型和偏好：
    [实例类型和偏好](../user_workloads/instancetypes.md)

-   关于持久卷和临时卷的更多信息：
    [磁盘和卷](../storage/disks_and_volumes.md)

-   如何通过 `console` 或 `vnc` 访问 VirtualMachineInstance：
    [控制台访问](../user_workloads/accessing_virtual_machines.md)

-   如何使用 `cloud-init` 定制 VirtualMachineInstance：
    [Cloud-init](../user_workloads/startup_scripts.md#cloud-init)
