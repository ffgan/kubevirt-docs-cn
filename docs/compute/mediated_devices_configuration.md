# 中介设备与虚拟 GPU {#mediated-devices-and-virtual-gpus}
## 配置中介设备与虚拟 GPU {#configuring-mediated-devices-and-virtual-gpus}

KubeVirt 旨在便于在大型集群上配置中介设备。
管理员可以使用 KubeVirt CR 中的 `mediatedDevicesConfiguration` API，通过提供他们期望在集群中配置的中介设备类型列表，以声明式方式创建或移除中介设备。

你还可以加入 `nodeMediatedDeviceTypes` 选项，借助节点选择器直接针对特定节点或一组节点提供更具体的配置。
`nodeMediatedDeviceTypes` 选项必须与 `mediatedDeviceTypes` 结合使用，才能覆盖在 `mediatedDeviceTypes` 部分中设置的全局配置。

KubeVirt 将使用所提供的配置，在能够支持这些设备的节点上自动创建相应的 mdev/vGPU 设备。

目前，每张卡只会配置一种 mdev 类型。
每张卡将配置所选 mdev 类型的最大实例数。

> 注意：某些厂商（例如 NVIDIA）要求在节点上安装驱动程序，才能提供中介设备，包括 vGPU。

同时包含 `nodeMediatedDeviceTypes` 和 `mediatedDeviceTypes` 的 KubeVirt CR 配置示例片段：
```yaml
spec:
  configuration:
    mediatedDevicesConfiguration:
      mediatedDeviceTypes:
      - nvidia-222
      - nvidia-228
      nodeMediatedDeviceTypes:
      - nodeSelector:
          kubernetes.io/hostname: nodeName
        mediatedDeviceTypes:
        - nvidia-234
```

## 配置场景 {#configuration-scenarios}
### 示例：每个节点上有多张卡的大型集群 {#example-large-cluster-with-multiple-cards-on-each-node}

在拥有多张卡、且这些卡可以支持相似 vGPU 类型的节点上，相关的期望类型将以轮询方式创建。

例如，考虑下面的 KubeVirt CR 配置：

```yaml
spec:
  configuration:
    mediatedDevicesConfiguration:
      mediatedDeviceTypes:
      - nvidia-222
      - nvidia-228
      - nvidia-105
      - nvidia-108
```

此集群中的节点拥有两种不同的 PCIe 卡：

1. 拥有 3 张 Tesla T4 卡的节点，每张卡可以支持多种设备类型：
    * nvidia-222
    * nvidia-223
    * nvidia-228
    * ...

2. 拥有 2 张 Tesla V100 卡的节点，每张卡可以支持多种设备类型：
    * nvidia-105
    * nvidia-108
    * nvidia-217
    * nvidia-299
    * ...

KubeVirt 随后将创建以下设备：

1. 拥有 3 张 Tesla T4 卡的节点将配置为：
    * 卡 1 上 16 个类型为 nvidia-222 的 vGPU
    * 卡 2 上 2 个类型为 nvidia-228 的 vGPU
    * 卡 3 上 16 个类型为 nvidia-222 的 vGPU
2. 拥有 2 张 Tesla V100 卡的节点将配置为：
    * 卡 1 上 16 个类型为 nvidia-105 的 vGPU
    * 卡 2 上 2 个类型为 nvidia-108 的 vGPU


### 示例：节点上只有一张卡，且支持多种期望的 vGPU 类型 {#example-single-card-on-a-node-multiple-desired-vgpu-types-are-supported}

当节点只有一张卡时，将配置列表中第一个受支持的类型。

例如，考虑下面的期望类型列表，其中 nvidia-223 和 nvidia-224 受支持：

```yaml
spec:
  configuration:
    mediatedDevicesConfiguration:
      mediatedDeviceTypes:
      - nvidia-223
      - nvidia-224
```
在这种情况下，将在该节点上配置 nvidia-223，因为它是列表中第一个受支持的类型。

## 覆盖特定节点上的配置 {#overriding-configuration-on-a-specific-node}

要覆盖由 `mediatedDeviceTypes` 设置的全局配置，请加入 `nodeMediatedDeviceTypes` 选项，指定节点选择器以及你要为该节点覆盖的 `mediatedDeviceTypes`。

### 示例：在每个节点有多张卡的大型集群中，覆盖特定节点的配置 {#example-overriding-the-configuration-for-a-specific-node-in-a-large-cluster-with-multiple-cards-on-each-node}

在此示例中，KubeVirt CR 包含 `nodeMediatedDeviceTypes` 选项，专门针对节点 2 覆盖全局配置，该节点将只使用 nvidia-234 类型。

```yaml
spec:
  configuration:
    mediatedDevicesConfiguration:
      mediatedDeviceTypes:
      - nvidia-230
      - nvidia-223
      - nvidia-224
    nodeMediatedDeviceTypes:
    - nodeSelector:
        kubernetes.io/hostname: node2  
      mediatedDeviceTypes:
      - nvidia-234
```

该集群有两个节点，两者都拥有 3 张 Tesla T4 卡。

* 每张卡都可以支持一长串类型，其中包括：
    * nvidia-222
    * nvidia-223
    * nvidia-224
    * nvidia-230
    * ...

KubeVirt 随后将创建以下设备：

1. 节点 1
    * 卡 1 上的类型为 nvidia-230
    * 卡 2 上的类型为 nvidia-223
2. 节点 2
    * 卡 1 和卡 2 上的类型为 nvidia-234

节点 1 已根据全局配置以轮询方式完成配置，但节点 2 只使用为它指定的 nvidia-234。

## 更新和移除 vGPU 类型 {#updating-and-removing-vgpu-types}

对 KubeVirt CR 的 `mediatedDeviceTypes` 部分所做的更改，将触发对集群节点上已配置 mdev/vGPU 类型的重新评估。

对与 KubeVirt CR 中 `nodeMediatedDeviceTypes` 的 nodeSelector 相匹配的节点标签所做的任何更改，都会触发类似的重新评估。

因此，中介设备将根据更新后的配置被重新配置，或被完全移除。

## 将 vGPU/MDEV 分配给虚拟机 {#assigning-vgpumdev-to-a-virtual-machine}
请参阅[主机设备分配](../compute/host-devices.md)，了解如何使用新创建的中介设备/vGPU。
