# 网络绑定插件 {#network-binding-plugins}

一种与 KubeVirt 集成以实现网络绑定的模块化插件。

> **有限支持**：KubeVirt 为插件作者提供网络绑定插件基础设施的常规支持。不过，各个网络插件绑定只能获得 KubeVirt 社区有限的、尽力而为的支持。
>
> 部署了 KubeVirt 且使用网络绑定插件的集群，如果遇到任何问题（无论是网络问题还是其他问题），应联系插件供应商寻求支持。
>
> 若要向 KubeVirt 核心项目及其社区请求支持，请使用不包含任何网络绑定插件的环境。下面列出的[插件示例](#plugins)是此规则的例外，因为它们**确实**由 KubeVirt 网络核心维护者维护。

## 概述 {#overview}

### 网络连通性 {#network-connectivity}

为了让 VM 能够访问外部网络，需要按连通性方面的要求，定义并配置若干层次。

这些层次包括：

- 宿主机连通性：网络提供商。
- 宿主机到 Pod 的连通性：CNI。
- Pod 到域的连通性：网络绑定。

本指南重点介绍网络绑定这一部分。

> **注意**：绑定插件也可以配置由 Kubernetes 动态资源分配（DRA）支持的网络接口。请参阅[将 VM 连接到由 DRA 支持的网络](dra_networking.md)。

### 网络绑定 {#network-binding}

网络绑定定义域（VM）网络接口在 VM Pod 中如何经由域连接到客户机。

网络绑定包括：

- 域 vNIC 配置。
- Pod 网络配置（可选）。
- 向客户机传递网络详情的服务（可选）。例如，用 DHCP 服务器把 IP 配置传递给客户机。

### 插件 {#plugins}

网络绑定一直是 KubeVirt 核心 API 和代码库的一部分。随着加入的网络绑定越来越多，并且频繁出现调整和更改现有网络绑定的请求，因此决定创建一套网络绑定插件基础设施。

该插件基础设施提供了组装网络绑定插件、并以模块化方式将其集成到 KubeVirt 中的手段。

KubeVirt 提供了若干网络绑定插件作为参考。可用的插件如下：

- [passt](../network/net_binding_plugins/passt.md)
- [macvtap](../network/net_binding_plugins/macvtap.md)
- [slirp](../network/net_binding_plugins/slirp.md)

## 定义与流程 {#definition-flow}

网络绑定插件的配置包含以下步骤：

- 部署网络绑定的可选组件：

  - 绑定 CNI 插件。
  - 绑定 NetworkAttachmentDefinition 清单。
  - 能够访问边车镜像。

- 注册网络绑定。
- 指定 VM 网络接口绑定。

### 部署 {#deployment}

根据插件的不同，集群中可能需要部署一些组件。并非所有网络绑定插件都需要全部这些组件，因此这些步骤是可选的。

- 绑定 CNI 插件：当需要更改 Pod 网络栈（且核心域附加方式不适用）时，会组装一个自定义 CNI 插件来服务于该网络绑定插件。

  该二进制文件需要部署到集群的每个节点上，与其他任何 CNI 插件一样。

  该二进制文件可以从源码构建，也可以直接使用现有制品。

> **注意**：CNI 插件二进制文件的位置取决于所使用的平台及其配置。这类二进制文件的常用路径是 `/opt/cni/bin/`。

- 绑定 NetworkAttachmentDefinition：它引用绑定 CNI 插件，并可带有可选的配置设置。该清单需要部署到集群中 VM 及其 Pod 可以访问的命名空间。

示例：
```yaml
apiVersion: "k8s.cni.cncf.io/v1"
kind: NetworkAttachmentDefinition
metadata:
  name: netbindingpasst
spec:
  config: '{
            "cniVersion": "1.0.0",
            "name": "netbindingpasst",
            "plugins": [
              {
                "type": "cni-passt-binding-plugin"
              }
            ]
  }'
```

> **注意**：可以把 NetworkAttachmentDefinition 部署在 `default` 命名空间中，其他所有命名空间都可以访问它。不过，出于安全原因，建议在 VM 所在的同一命名空间中定义 NetworkAttachmentDefinition。

- [Multus](https://github.com/k8snetworkplumbingwg/multus-cni)：为了让网络绑定 CNI 和 NetworkAttachmentDefinition 能够工作，集群中需要部署 Multus。更多信息请查看[快速入门安装指南](https://github.com/k8snetworkplumbingwg/multus-cni#quickstart-installation-guide)。

- 边车镜像：当核心域附加方式不适用时，使用边车来配置 vNIC 的域配置。在更复杂的场景中，边车还会运行 DHCP 之类的服务，以便向客户机传递 IP 信息。

  边车镜像构建完成后，通常会被推送到镜像仓库供使用。因此，集群需要能够访问该镜像。

  可以从源码构建该镜像并推送到可访问的仓库，也可以使用已经包含该镜像的指定仓库。

### 注册 {#register}

要使用网络绑定插件，集群管理员需要注册该绑定。注册包括把绑定名称及其全部参数添加到 KubeVirt CR 中。

当前支持以下（可选）参数：

- [networkAttachmentDefinition](#networkattachmentdefinition)
- [sidecarImage](#sidecarimage)
- [domainAttachmentType](#domainattachmenttype)
- [migration](#migration)

#### networkAttachmentDefinition {#networkattachmentdefinition}

自：v1.1.0

使用 <namespace/object-name> 格式，指定定义了绑定插件所使用的 CNI 插件及配置的 [NetworkAttachementDefinition](https://github.com/k8snetworkplumbingwg/multi-net-spec/blob/master/v1.0/%5Bv1%5D%20Kubernetes%20Network%20Custom%20Resource%20Definition%20De-facto%20Standard.md)。当绑定插件需要更改 Pod 网络命名空间时使用。

#### sidecarImage {#sidecarimage}

自：v1.1.0

指定仓库中的一个容器镜像。当绑定插件需要修改域 vNIC 配置，或需要执行某个服务（例如 DHCP 服务器）时使用。

#### domainAttachmentType {#domainattachmenttype}

自：v1.1.1

域附加类型是 KubeVirt 预定义的、用于把接口附加到域的核心方法。

指定一种核心域附加类型的名称。它可以作为边车的一种替代方案，用来配置域 vNIC。

支持的类型：

- `tap`（自 v1.1.1 起）：域配置被设置为使用已有的 tap 设备。它也支持已有的 `macvtap` 设备。
- `managedTap`：（自 1.4 起）域配置被设置为使用 tap 设备。该 tap 设备通过网桥连接到 Pod 接口（类似于核心 bridge 绑定），并且全部按需创建。此类型不提供 IPAM 支持。

当同时指定 `domainAttachmentType` 和 `sidecarImage` 时，会先按照 `domainAttachmentType` 配置域，然后 `sidecarImage` 可以再修改它。

#### migration {#migration}

自：v1.2.0

指定网络绑定插件是否支持迁移。可以指定一种迁移方法。支持的迁移方法类型：

- `link-refresh`（自 v1.2.0 起）：迁移之后，客户机网卡会先被停用，然后再被激活。这可以用于续订 DHCP 租约。

> **注意**：在某些部署中，KubeVirt CR 由外部控制器控制（例如 [HCO](https://github.com/kubevirt/hyperconverged-cluster-operator)）。在这种情况下，请配置外层 Operator/控制器，以便这些更改得以保留。

示例（`passt` 绑定）：
```
kubectl patch kubevirts -n kubevirt kubevirt --type=json -p='[{"op": "add", "path": "/spec/configuration/network",   "value": {
            "binding": {
                "passt": {
                    "networkAttachmentDefinition": "default/netbindingpasst",
                    "sidecarImage": "quay.io/kubevirt/network-passt-binding:20231205_29a16d5c9"
                    "migration": {
                        "method": "link-refresh"
                    }
                }
            }
        }}]'
```

#### 计算资源开销 {#compute-resource-overhead}

自：v1.3.0

某些插件可能需要向 virt-launcher Pod 的 compute 容器追加额外资源。

可以指定计算资源开销，这些开销会被加到由使用该插件的虚拟机派生出的 virt-launcher Pod 的 `compute` 容器上。

> **注意**：目前仅支持内存开销请求。

> **注意**：在某些部署中，KubeVirt CR 由外部控制器控制（例如 [HCO](https://github.com/kubevirt/hyperconverged-cluster-operator)）。在这种情况下，请配置外层 Operator/控制器，以便这些更改得以保留。

示例（`passt` 绑定）：
```shell
kubectl patch kubevirts -n kubevirt kubevirt --type=json -p='[{"op": "add", "path": "/spec/configuration/network",   "value": {
            "binding": {
                "passt": {
                    "networkAttachmentDefinition": "default/netbindingpasst",
                    "sidecarImage": "quay.io/kubevirt/network-passt-binding:20231205_29a16d5c9",
                    "migration": {
                        "method": "link-refresh"
                    },
                    "computeResourceOverhead": {
                      "requests": {
                        "memory": "500Mi",
                      }
                    }
                }
            }
        }}]'
```

由使用 passt 网络绑定插件的 VM 派生出的 virt-launcher Pod 中的每个 compute 容器，都会额外增加 `500Mi` 内存开销。

### VM 网络接口 {#vm-network-interface}

配置 VM/VMI 网络接口时，可以指定绑定插件名称。如果该名称存在于 KubeVirt CR 中，就会用它来设置网络接口。

示例（`passt` 绑定）：
```yaml
---
apiVersion: kubevirt.io/v1
kind: VirtualMachine
metadata:
  labels:
    kubevirt.io/vm: vm-net-binding-passt
  name: vm-net-binding-passt
spec:
  runStrategy: Always
  template:
    metadata:
      labels:
        kubevirt.io/vm: vm-net-binding-passt
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
          - name: passtnet
            binding:
              name: passt
          rng: {}
        resources:
          requests:
            memory: 1024M
      networks:
      - name: passtnet
        pod: {}
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
