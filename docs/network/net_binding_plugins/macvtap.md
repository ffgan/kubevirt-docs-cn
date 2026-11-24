# Macvtap 绑定 {#macvtap-binding}

## 概述 {#overview}

借助 `macvtap` 绑定插件，虚拟机会被直接暴露到 Kubernetes 节点的 L2 网络。其实现方式是“扩展”一个现有网络接口，为其附加一个拥有自身 MAC 地址的虚拟设备。

其主要优点如下：

- 直接连接到节点网卡，中间没有网桥。

### 功能支持 {#functionality-support}

| 功能 | 支持情况 |
|------------------------------------------|---------|
| 无需额外权能即可运行（在 Pod 上） | 是 |
| 迁移支持 | 否 |
| IPAM 支持（在 Pod 上） | 否 |
| 主网络（Pod 网络） | 否 |
| 辅助网络 | 是 |

### 已知问题 {#known-issues}

- 实时迁移尚未得到完全支持，请参阅[问题 #5912](https://github.com/kubevirt/kubevirt/issues/5912#issuecomment-888938920)

> **警告**：在 [KinD](https://github.com/kubernetes-sigs/kind) 集群上，
> 用户需要[调整集群配置](https://github.com/kubevirt/macvtap-cni/issues/39#issuecomment-1242765996)，
把正在运行的宿主机的 `dev` 挂载到 KinD 节点上，原因是一个
[已知问题](https://github.com/kubevirt/macvtap-cni/issues/39)。

### 部署 {#deployment}

`macvtap` 方案由一个 CNI 和一个 DP 组成。

要使用 `macvtap`，需要完成以下几点：

- 在节点上部署 CNI 插件二进制文件。
- 在节点上部署设备插件守护进程。
- 配置暴露哪些节点接口。
- 定义一个指向该 CNI 插件的 NetworkAttachmentDefinition。

#### 在节点上部署 Macvtap CNI 和 DP {#macvtap-cni-and-dp-deployment-on-nodes}

为简化该过程，请使用 [Cluster Network Addons Operator](https://github.com/kubevirt/cluster-network-addons-operator#macvtap) 在集群中部署并配置 macvtap 组件。

上述 Operator 实际部署的是 [macvtap](https://github.com/kubevirt/macvtap-cni) CNI 和设备插件。

#### 向 macvtap 设备插件暴露节点接口 {#expose-node-interface-to-the-macvtap-device-plugin}

有两种不同的方式，用来配置向用户暴露哪些宿主机接口，从而使用户能够在这些接口之上创建 macvtap 接口：

- 选择宿主机接口：指明暴露哪些宿主机接口。
- 暴露全部接口：暴露所有宿主机的全部接口。

这两个选项都通过 `macvtap-deviceplugin-config` ConfigMap 进行配置，关于如何配置的更多信息见 [macvtap-cni](https://github.com/kubevirt/macvtap-cni#deployment) 仓库。

下面是一个最小示例，其中暴露了 Kubernetes 节点的 `eth0` 接口，经由 `lowerDevice` 属性。

```yaml
kind: ConfigMap
apiVersion: v1
metadata:
  name: macvtap-deviceplugin-config
data:
  DP_MACVTAP_CONF: |
    [
        {
            "name"        : "dataplane",
            "lowerDevice" : "eth0",
            "mode"        : "bridge",
            "capacity"    : 50
        },
    ]
```

可以省略这一步，因为上述 `ConfigMap` 的默认配置就是暴露所有宿主机接口（由以下配置表示）：

```yaml
kind: ConfigMap
apiVersion: v1
metadata:
  name: macvtap-deviceplugin-config
data:
  DP_MACVTAP_CONF: '[]'
```

#### Macvtap NetworkAttachmentDefinition {#macvtap-networkattachmentdefinition}

macvtap 网络附加所需的配置可以非常精简：

```yaml
kind: NetworkAttachmentDefinition
apiVersion: k8s.cni.cncf.io/v1
metadata:
  name: macvtapnetwork
  annotations:
    k8s.v1.cni.cncf.io/resourceName: macvtap.network.kubevirt.io/eth0
spec:
  config: '{
      "cniVersion": "0.3.1",
      "name": "macvtapnetwork",
      "type": "macvtap",
      "mtu": 1500
    }'
```

该对象应当创建在其他所有命名空间都能访问的 “default” 命名空间中，或者创建在 VM 所在的同一命名空间中。

所请求的 `k8s.v1.cni.cncf.io/resourceName` 注解必须指向一个已暴露的宿主机接口（通过 `lowerDevice` 属性，位于 `macvtap-deviceplugin-config` `ConfigMap` 上）。

## Macvtap 网络绑定插件 {#macvtap-network-binding-plugin}

> **注意**：实验性的核心 macvtap 绑定已在 v1.2.0 弃用，并在 v1.3.0 移除。本文档介绍的是 macvtap 的**网络绑定插件**实现。

macvtap 绑定插件由以下组件组成：

- [Macvtap CNI 插件](https://github.com/kubevirt/macvtap-cni)。

该插件需要：

- 在 KubeVirt CR 上注册该绑定插件。
- 从 VM spec 的接口中按名称引用该网络绑定。

详细说明如下：

### 功能门控 {#feature-gate}

自 v1.5.0 起，网络绑定插件功能默认启用，并且没有功能门控。macvtap 插件同样没有自己的功能门控，但需要通过[注册](./#macvtap-registration)使其在集群中可用。

### Macvtap 注册 {#macvtap-registration}

要供 VM 使用，需要把 macvtap 绑定插件配置添加到 kubevirt CR 中。

要注册 macvtap 绑定，请按如下方式修补 kubevirt CR：

```yaml
kubectl patch kubevirts -n kubevirt kubevirt --type=json -p='[{"op": "add", "path": "/spec/configuration/network",   "value": {
            "binding": {
                "macvtap": {
                    "domainAttachmentType": "tap"
                }
            }
        }}]'
```

### VM 的 Macvtap 网络接口 {#vm-macvtap-network-interface}

将 VM 网络接口的绑定名称设置为 kubevirt CR 中所定义的名称。

```yaml
---
apiVersion: kubevirt.io/v1
kind: VirtualMachine
metadata:
  labels:
    kubevirt.io/vm: vm-net-binding-macvtap
  name: vm-net-binding-macvtap
spec:
  runStrategy: Always
  template:
    metadata:
      labels:
        kubevirt.io/vm: vm-net-binding-macvtap
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
          interfaces:
          - name: podnet
            masquerade: {}
          - name: hostnetwork
            binding:
              name: macvtap
          rng: {}
        resources:
          requests:
            memory: 1024M
      networks:
      - name: podnet
        pod: {}
      - name: hostnetwork
        multus:
          networkName: macvtapnetwork
      terminationGracePeriodSeconds: 0
      volumes:
      - containerDisk:
          image: quay.io/kubevirt/fedora-with-test-tooling-container-disk:v1.1.0
        name: containerdisk
      - cloudInitNoCloud:
          networkData: |
            version: 2
            ethernets:
              eth0:
                dhcp4: true
        name: cloudinitdisk
```

multus 的 `networkName` 值应当与[网络附加定义](#macvtap-networkattachmentdefinition)一节中使用的名称相对应。

`binding` 值应当与[注册](#macvtap-registration)中使用的名称相对应。
