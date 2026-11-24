# 接口与网络 {#interfaces-and-networks}

将虚拟机连接到网络由两部分组成。首先，在 `spec.networks` 中指定网络。然后，通过在 `spec.domain.devices.interfaces` 中指定以这些网络为后端的接口，把这些接口添加到 VM。

每个接口都必须有一个名称相同的对应网络。

`interface` 定义虚拟机的虚拟网络接口。`network` 指定 `interface` 的后端，并声明它所连接的逻辑或物理设备。

配置 `interface` 和 `network` 都有多种方式。

所有可用的配置选项见[接口 API 参考](https://kubevirt.io/api-reference/master/definitions.html#_v1_interface)和[网络 API 参考](https://kubevirt.io/api-reference/master/definitions.html#_v1_network)。

## 网络 {#networks}

网络在 VM 的 `spec.template.spec.networks` 中配置。网络必须具有唯一名称。

每个网络应通过定义以下字段之一来声明其类型：

| 类型     | 描述                                                                                  |
|----------|---------------------------------------------------------------------------------------|
| `pod`    | 默认 Kubernetes 网络                                                                  |
| `multus` | 使用 Multus 提供的辅助网络，或在将 Multus 定义为默认时的主网络                        |

### pod {#pod}

表示由集群网络方案配置、存在于每个 Pod 中的默认（即主）Pod 接口（通常为 `eth0`）。
此网络类型的主要优势在于它原生于 Kubernetes，使 VM 能够受益于 Kubernetes 提供的所有网络服务。

```yaml
# partial example - kept short for brevity 
apiVersion: kubevirt.io/v1
kind: VirtualMachine
spec:
  template:
    spec:
      domain:
        devices:
          interfaces:
            - name: default
              masquerade: {}
      networks:
      - name: default
        pod: {} # Stock pod network
```

### multus {#multus}

Kubernetes 中的辅助网络允许 Pod 连接到默认网络之外的其他网络，从而实现更复杂的网络拓扑。这些辅助网络由 [Multus](https://github.com/k8snetworkplumbingwg/multus-cni) 等元插件支持，使每个 Pod 能够挂接多个网络接口。
KubeVirt 支持使用 Multus 将 VM 连接到辅助网络。
这假定 multus 已在整个集群中安装，并且已创建相应的 `NetworkAttachmentDefinition` CRD。

以下示例定义了一个使用 [bridge CNI 插件](https://www.cni.dev/plugins/current/main/bridge/) 的辅助网络，该网络会将 VM 连接到 Linux 网桥 `br10`。也可以使用 ptp、bridge-cni 或 sriov-cni 等其他 CNI 插件。有关其安装与用法，请参阅相应项目的文档。

首先需要创建 `NetworkAttachmentDefinition`。这通常由管理员完成。用户随后可以引用该定义。

```yaml
apiVersion: "k8s.cni.cncf.io/v1"
kind: NetworkAttachmentDefinition
metadata:
  name: linux-bridge-net-ipam
spec:
  config: '{
      "cniVersion": "0.3.1",
      "name": "mynet",
      "plugins": [
        {
          "type": "bridge",
          "bridge": "br10",
          "disableContainerInterface": true,
          "macspoofchk": true
        }
      ]
    }'
```

使用以下定义，VM 将连接到默认 Pod 网络和辅助网桥网络，并引用上文所示的 `NetworkAttachmentDefinition`（位于同一命名空间）。

```yaml
# partial example - kept short for brevity 
apiVersion: kubevirt.io/v1
kind: VirtualMachine
spec:
  template:
    spec:
       domain:
         devices:
           interfaces:
           - name: default
             masquerade: {}
           - name: bridge-net
             bridge: {}
       networks:
       - name: default
         pod: {} # Stock pod network
       - name: bridge-net
         multus: # Secondary multus network
           networkName: linux-bridge-net-ipam #ref to NAD name
```

#### 网络资源注入 {#network-resource-injection}

版本：
- v1.8：Beta
- v1.9：Beta（默认启用）
- v1.10：GA

某些辅助网络配置使用 `k8s.v1.cni.cncf.io/resourceName` `NetworkAttachmentDefinition`（NAD）注解，表明节点上需要网络设备资源。常见用例包括：

- **SR-IOV** — NAD 引用 SR-IOV VF 资源（例如 `intel.com/sriov_net`）。
- **使用 [bridge-marker](https://github.com/kubevirt/bridge-marker) 的网桥网络** — NAD 引用网桥设备资源。
- **[Macvtap](../network/net_binding_plugins/macvtap.md)** — NAD 引用 macvtap 设备资源（例如 `macvtap.network.kubevirt.io/eth0`）。

对于这些用例，必须在集群上部署 [network-resources-injector](https://github.com/k8snetworkplumbingwg/network-resources-injector) 组件。它是一个变更准入 webhook，从 NAD 注解读取资源名称，并将相应的资源请求与限制注入 VM Pod，从而确保 Kubernetes 调度器将 VM 放置到具备所需网络资源的节点上。

有关安装说明，请参阅 [network-resources-injector 文档](https://github.com/k8snetworkplumbingwg/network-resources-injector#readme)。

!!! warning "弃用通知"
    在 v1.8 之前，KubeVirt 的 virt-controller 会直接查询 NAD 对象，以将网络资源需求注入 virt-launcher Pod。此旧行为已弃用，现在由 `ExternalNetResourceInjection` 功能门控控制。

    - **v1.8**：引入了 `ExternalNetResourceInjection` 功能门控，默认禁用。启用后，KubeVirt 不再查询 NAD，也不再部署相关 RBAC 规则——改由 `network-resources-injector` 负责处理资源注入。
    - **v1.9**：该功能门控默认启用。用户仍可禁用它，以回退到旧行为。
    - **v1.10**：`ExternalNetResourceInjection` 功能门控已升级为正式可用（GA），并且旧的 NAD 查询代码已被完全移除。

    使用受影响用例的集群管理员必须部署 `network-resources-injector`，然后才能启用 `ExternalNetResourceInjection` 功能门控或升级到默认启用该功能门控的 v1.9 或 v1.10。在 v1.10 上没有旧行为回退。

#### 将 Multus 作为主网络提供方 {#multus-as-primary-network-provider}

也可以通过指定 VM 的 `spec.template.spec.networks.multus.default=true`，将 multus 网络定义为默认 Pod 网络。
更多信息请参阅 [Multus](https://github.com/k8snetworkplumbingwg/multus-cni) 文档。
>**注意：** multus 的 `default` 网络与 `pod` 网络类型互斥

>被选为默认的 multus 委托**必须**至少返回一个 IP 地址。


示例：带有 IPAM 的 `NetworkAttachmentDefinition`。

```yaml
apiVersion: "k8s.cni.cncf.io/v1"
kind: NetworkAttachmentDefinition
metadata:
  name: bridge-test
spec:
  config: '{
      "cniVersion": "0.3.1",
      "name": "bridge-test",
      "type": "bridge",
      "bridge": "br1",
      "ipam": {
        "type": "host-local",
        "subnet": "10.250.250.0/24"
      }
    }'
```

定义一个以 `multus` 网络作为默认网络的 VM。

```yaml
# partial example - kept short for brevity 
apiVersion: kubevirt.io/v1
kind: VirtualMachine
spec:
  template:
    spec:
      domain:
        devices:
          interfaces:
            - name: test1
              bridge: {}
      networks:
      - name: test1
        multus: # Multus network as default
          default: true
          networkName: bridge-test
```

## 接口 {#interfaces}

网络接口在 `spec.domain.devices.interfaces` 中配置。
它们描述客户机实例内部所“看到”的虚拟接口属性。同一个 `network` 可以用多种不同方式连接到虚拟机，每种方式都有各自的连通性保证和特性。
> **注意** 网络与接口必须是一对一关系

必需的接口配置包括：
- `name`，用于引用网络名称
- 下表中受支持的网络核心绑定的名称，或对[网络绑定插件](https://kubevirt.io/user-guide/network/network_binding_plugins/)的引用。

| 类型           | 描述                                                                      |
|----------------|---------------------------------------------------------------------------|
| `bridge`       | 使用 Linux 网桥连接                                                       |
| `sriov`        | 通过 vfio 使用直通 SR-IOV VF 连接                                         |
| `masquerade`   | 使用 `nftables` 规则对出站和入站流量进行 NAT                              |
| `passtBinding` | 使用 `passt` 用户态绑定连接                                               |

每个接口还可以有额外的配置字段，用于修改客户机实例内部所“看到”的属性，如下表所示：

| 名称       | 格式                                                                    | 默认值           | 描述                                                                                       |
|------------|-------------------------------------------------------------------------|------------------|--------------------------------------------------------------------------------------------|
| model      | 以下之一：`e1000`、`e1000e`、`ne2k_pci`、`pcnet`、`rtl8139`、`virtio`   | `virtio`         | 网卡类型。**注意：** 如果客户机镜像未附带 virtio 驱动，请使用 `e1000` 型号                 |
| macAddress | `ff:ff:ff:ff:ff:ff` 或 `FF-FF-FF-FF-FF-FF`                              |                  | 客户机系统内部看到的 MAC 地址，例如：`de:ad:00:00:be:af`                                   |
| ports      |                                                                         | 空（即所有端口） | 要转发到 VM 客户机的端口列表。                                                             |
| pciAddress | `0000:81:00.1`                                                          |                  | 设置网络接口 PCI 地址，例如：`0000:81:00.1`                                                |

```yaml
# partial example - kept short for brevity 
apiVersion: kubevirt.io/v1
kind: VirtualMachine
spec:
  template:
    spec:
      domain:
        devices:
          interfaces:
            - name: default
              model: e1000 # expose e1000 NIC to the guest
              masquerade: {} # connect through a masquerade
              ports:
               - name: http
                 port: 80 # forward only http traffic
      networks:
      - name: default
        pod: {}
```

> **注意：** 对于辅助接口，当为虚拟机接口指定 MAC 地址时，该地址会传递给底层 CNI 插件，而该插件进而应当配置网络提供方，以允许这一特定 MAC。
> 并非每个插件都原生支持自定义 MAC 地址。

> **注意：** 对于一些不原生支持自定义 MAC 地址的 CNI 插件，有一种变通办法，即使用 `tuning` CNI 插件调整 Pod 接口的 MAC 地址。用法如下：

```yaml
apiVersion: "k8s.cni.cncf.io/v1"
kind: NetworkAttachmentDefinition
metadata:
  name: ptp-mac
spec:
  config: '{
      "cniVersion": "0.3.1",
      "name": "ptp-mac",
      "plugins": [
        {
          "type": "ptp",
          "ipam": {
            "type": "host-local",
            "subnet": "10.1.1.0/24"
          }
        },
        {
          "type": "tuning"
        }
      ]
    }'
```

> 此方法未必适用于所有插件。例如，OKD SDN 与 `tuning` 插件不兼容。
>
> -   原生处理自定义 MAC 地址的插件：`ovs`、`bridge`。
>
> -   与 `tuning` 插件兼容的插件：`flannel`、`ptp`。
>
> -   不需要对 MAC 地址做特殊处理的插件：`sriov`（`vfio` 模式）。
>
### 端口 {#ports}

声明要转发到虚拟机客户机的端口。

| 名称       | 格式      | 是否必需 | 描述         |
|------------|-----------|----------|--------------|
| `name`     |           | 否       | 名称         |
| `port`     | 1 - 65535 | 是       | 要公开的端口 |
| `protocol` | TCP,UDP   | 否       | 连接协议     |

> **注意：**  
> `ports` 属性仅由特定网络绑定处理，即 masquerade 和 passt 核心网络绑定。  
> 在 bridge 网络绑定中，它不起作用。对于网络绑定插件，其行为取决于插件实现。  
> 详情请参阅网络绑定插件[文档](https://kubevirt.io/user-guide/network/network_binding_plugins/)。

如果省略 `spec.domain.devices.interfaces`，虚拟机将使用 `bridge` 类型的默认 Pod 网络接口进行连接。如果希望虚拟机实例没有任何网络连通性，可以按如下方式使用 `autoattachPodInterface` 字段：

```yaml
# partial example - kept short for brevity 
apiVersion: kubevirt.io/v1
kind: VirtualMachine
spec:
  template:
    spec:
      domain:
        devices:
          autoattachPodInterface: false
```

### bridge {#bridge}

在 `bridge` 模式下，虚拟机通过 Linux“网桥”连接到网络后端。Pod 网络的 IPv4 地址（如果存在）会通过 DHCPv4 委派给虚拟机。虚拟机应配置为使用 DHCP 获取 IPv4 地址。

> **注意：** 如果虚拟机接口规约中未配置特定 MAC 地址，相关 Pod 接口的 MAC 地址会委派给虚拟机。

```yaml
# partial example - kept short for brevity 
apiVersion: kubevirt.io/v1
kind: VirtualMachine
spec:
  template:
    spec:
      domain:
        devices:
          interfaces:
            - name: red
              bridge: {} # connect through a bridge
      networks:
      - name: red
        multus:
          networkName: red
```

目前，`bridge` 模式不支持额外的配置字段。

> **注意：** 由于 IPv4 地址委派，在 `bridge` 模式下 Pod 没有配置 IP 地址，这可能会给依赖该地址的第三方方案带来问题。例如，Istio 在此模式下可能无法工作。

> **注意：** 管理员可以通过指定的配置标志，禁止对 Pod 网络使用 `bridge` 接口类型。为此，管理员应将以下选项设置为 `false`：

```yaml
apiVersion: kubevirt.io/v1
kind: Kubevirt
metadata:
  name: kubevirt
  namespace: kubevirt
spec:
  configuration:
    network:
      permitBridgeInterfaceOnPodNetwork: false
```

> **注意：** 使用 `bridge` 接口类型绑定 Pod 网络可能会引起问题。除上述说明中提到的第三方问题外，`bridge` 接口类型的 Pod 网络绑定不允许实时迁移，并且某些 CNI 插件可能不允许为 VM 实例使用自定义 MAC 地址。如果认为可能受到上述任一问题的影响，请考虑将默认接口类型改为 `masquerade`，并按上文示例禁用 Pod 网络的 `bridge` 类型。


### masquerade {#masquerade}

在 `masquerade` 模式下，KubeVirt 为虚拟机分配内部 IP 地址，并将其隐藏在 NAT 之后。离开虚拟机的所有流量都使用 Pod IP 地址进行“源 NAT”；因此，集群工作负载应使用 Pod 的 IP 地址通过此接口联系 VM。该 IP 地址报告在 VMI 的 `status.interfaces` 中。客户机操作系统应配置为使用 DHCP 获取 IPv4 地址。

为使 VM 在实时迁移或硬重启（二者都会使 VM 改在另一个 Pod 上运行，且 IP 地址不同）之后仍然可达，应通过 Kubernetes [服务](../network/service_objects.md#service-objects) 将其公开。

要允许特定端口的流量进入虚拟机，应按如下方式配置接口模板的 `ports` 部分。如果缺少 `ports` 部分，所有端口都会转发到 VM。

```yaml
# partial example - kept short for brevity 
apiVersion: kubevirt.io/v1
kind: VirtualMachine
spec:
  template:
    spec:
      domain:
        devices:
          interfaces:
            - name: red
              masquerade: {} # connect using masquerade mode
              ports:
                - port: 80 # forward traffic on port 80 to the virtual machine guest
      networks:
      - name: red
        pod: {}
```

> **注意：** Masquerade 只允许连接到 Pod 网络。

> **注意：** 可以使用 `vmNetworkCIDR` 属性在 Pod 网络部分配置网络 CIDR。

#### masquerade - IPv4 与 IPv6 双栈支持 {#masquerade-ipv4-and-ipv6-dual-stack-support}

`masquerade` 模式可用于 IPv4 与 IPv6 双栈集群，为 VM 提供两种协议上的 IP 连通性。

与 IPv4 `masquerade` 模式一样，可以使用 Pod 的 IP 地址联系 VM——此时将是两个 IP 地址，一个 IPv4 和一个 IPv6。出站流量也会被“NAT”到 Pod 上对应地址族的 IP 地址。

与 IPv4 不同，IPv6 地址和默认路由不会自动配置；应通过 cloud init 配置，如下所示：

```yaml
# partial example - kept short for brevity 
apiVersion: kubevirt.io/v1
kind: VirtualMachine
spec:
  template:
    spec:
      domain:
        devices:
          interfaces:
            - name: red
              masquerade: {} 
      networks:
      - name: red
        pod: {}
    volumes:
      - cloudInitNoCloud:
          networkData: |
            version: 2
            ethernets:
              eth0:
                dhcp4: true
                addresses: [ fd10:0:2::2/120 ]
                gateway6: fd10:0:2::1
```

> **注意：** VM 的 IPv6 地址和默认网关**必须**是上文所示的地址。

#### masquerade - IPv6 单栈支持 {#masquerade-ipv6-single-stack-support}

`masquerade` 模式可用于 IPv6 单栈集群，为 VM 提供仅 IPv6 的连通性。

与 IPv4 `masquerade` 模式一样，可以使用 Pod 的 IP 地址联系 VM——此时为 IPv6 地址。
出站流量也会被“NAT”到 Pod 相应的 IPv6 地址。

与双栈集群一样，IPv6 地址和默认路由不会自动配置；应通过 cloud init 配置，如[双栈部分](#masquerade-ipv4-and-ipv6-dual-stack-support)所示。

与为 IPv4 提供 DHCP 服务器的双栈集群不同，IPv6 单栈集群完全没有 DHCP 服务器。因此，VM 不会获得搜索域信息，也无法通过 FQDN 访问目标。
跟踪问题 - https://github.com/kubevirt/kubevirt/issues/7184

### sriov {#sriov}

在 `sriov` 核心网络绑定中，SR-IOV 虚拟功能的 PCI 设备被直接暴露给虚拟机。
可以使用 [SR-IOV 设备插件](https://github.com/k8snetworkplumbingwg/sriov-network-device-plugin)和 [CNI](https://github.com/k8snetworkplumbingwg/sriov-cni) 在 Kubernetes 中管理 SR-IOV 设备，供 KubeVirt 使用。
该设备通过 [vfio](https://www.kernel.org/doc/Documentation/vfio.txt) 用户态接口作为[主机设备](https://libvirt.org/drvnodedev.html)直通到客户机操作系统，以保持较高的网络性能。

#### 如何向 KubeVirt 暴露 SR-IOV VF {#how-to-expose-sr-iov-vfs-to-kubevirt}

为简化步骤，请使用 [SR-IOV 网络 Operator](https://github.com/k8snetworkplumbingwg/sriov-network-operator/blob/v1.4.0/doc/quickstart.md) 在集群中部署并配置 SR-IOV 组件。有关如何使用该 operator，请参阅[相应文档](https://github.com/k8snetworkplumbingwg/sriov-network-operator/blob/master/doc/quickstart.md)。

> **注意：** KubeVirt 依赖 VFIO 用户态驱动将 PCI 设备传入 VM 客户机。因此，在配置 SR-IOV operator 策略时，请确保定义一个使用 `deviceType: vfio-pci` 的 VF 资源池。


#### 启动 SR-IOV VM {#start-an-sr-iov-vm}

假定集群节点上已部署 `sriov-device-plugin` 和 `sriov-cni`，请[按此处所示](https://github.com/k8snetworkplumbingwg/sriov-cni?tab=readme-ov-file#usage)创建 network-attachment-definition CR。
CR 的名称应与 VM 网络规约中的引用相对应（见下方示例）。

最后，要创建将挂接到上述网络的 VM，请参考以下 VM 规约：

```yaml
---
# partial example - kept short for brevity 
apiVersion: kubevirt.io/v1
kind: VirtualMachine
spec:
  template:
    spec:
      domain:
        devices:
          interfaces:
          - name: default
            masquerade: {}
          - name: sriov-net
            sriov: {}
      networks:
      - name: default
        pod: {}
      - multus:
          networkName: default/sriov-net
        name: sriov-net
```

> **注意：** 对于某些网卡（例如 Mellanox），需要在客户机 VM 中安装内核模块。

> **注意：** 只有当 Kubernetes CPU 管理器在具备 SR-IOV 能力的工作节点上运行时，才能实现专用 CPU 放置。
> 更多详情请参阅[专用 CPU 资源文档](../compute/dedicated-cpu_resources.md/)。

#### 使用 SR-IOV 进行实时迁移 {#live-migration-with-sr-iov}

使用 SR-IOV 网卡的 VM 可以迁移。在实时迁移期间，SR-IOV 设备会在迁移开始前自动从源 VM 热拔出，然后在迁移完成后在目标节点上重新热插入。此过程自动处理，无需人工干预。

### passtBinding {#passtbinding}

[v1.8.0] - Beta

[Plug A Simple Socket Transport](https://passt.top/passt/about/) 是 [SLIRP](https://en.wikipedia.org/wiki/Slirp) 的增强替代方案，提供用户态网络连通性。

`passt` 是一个通用工具，在二层网络接口与宿主机上的原生四层套接字（TCP、UDP、ICMP/ICMPv6 echo）之间实现转换层。

其主要优点如下：

- 不需要 CAP_NET_RAW 和 CAP_NET_ADMIN 等额外网络能力。
- 开箱即可与服务网格集成（服务网格期望应用在本地运行）。
- 开箱即支持 IPv6（现有绑定则需要手动配置 IPv6）。

#### 功能支持 {#functionality-support}

| 功能                          | 支持 |
|-------------------------------|------|
| 迁移支持                      | 是   |
| 服务网格支持                  | 是   |
| 客户机中的 Pod IP             | 是   |
| 客户机中的自定义 CIDR         | 否   |
| 运行需要额外能力（在 Pod 上） | 否   |
| 主网络（Pod 网络）            | 是   |
| 辅助网络                      | 否   |

> **注意**：使用 `passtBinding` 会为 VM 的资源需求增加 `250Mi` 内存。

VM 首次启动时，Passt 的内部 DHCP 服务器会为客户机分配一个租期为“无限”的 IP 地址。

#### 实时迁移支持 {#live-migration-support}

在实时迁移期间：
 - 目标 Pod 会从 Pod 网络获得一个新 IP 地址，并反映在 VMI 接口状态中。
 - 客户机操作系统会保留原始 IP 地址，直到它显式向 Passt 的内部 DHCP 服务器请求地址。

#### 连通性注意事项 {#connectivity-considerations}

Passt 在向客户机传入或从客户机传出流量时，会自动应用网络地址转换（NAT）。
这会实现以下行为：
 - 使用 Pod 的新 IP 地址进行连通
 - 尽管 Pod IP 与客户机 IP 不一致，进出客户机的连通性不受影响。

可以从客户机操作系统发起 DHCP 请求，以同步客户机与 Pod 的 IP 地址。

##### TCP 通信的无缝迁移 {#seamless-migration-for-tcp-communication}

当使用的 CNI/网络提供方支持在迁移过程中保留 virt-launcher Pod 的 IP 地址时，`passt` 可确保 TCP 连接持续不断，用户不会察觉到中断。

#### 前提条件 {#preconditions}

必须启用 `PasstBinding` [功能门控](../cluster_admin/activating_feature_gates.md#how-to-activate-a-feature-gate)。

#### VM passt 网络接口 {#vm-passt-network-interface}

```yaml
---
apiVersion: kubevirt.io/v1
kind: VirtualMachine
metadata:
  name: vm-net-binding-passt
spec:
  template:
    spec:
      domain:
        devices:
          interfaces:
          - name: passtnet
            passtBinding: {}
            ports:
            - name: http
              port: 80
              protocol: TCP
      networks:
      - name: passtnet
        pod: {}
```

### 链路状态管理 {#link-state-management}

从 KubeVirt v1.5.0 起，可以使用接口的 `state` 字段设置期望的接口链路状态。
允许的取值如下：

- `up` - 等同于活动的网络连接。未指定值时，此为默认值。
- `down` - 等同于网线已从 vNIC 断开。
- `absent` - 仅与网卡热拔出相关。

> **注意：** 除 `sriov` 之外，所有绑定都可以为网络接口指定期望的链路状态，因为 KubeVirt 使用的虚拟化栈不允许为 SR-IOV 设备设置该状态。

将接口指定为 `down` 状态的示例：

```yaml
apiVersion: kubevirt.io/v1
kind: VirtualMachine
metadata:
  name: my-vm
spec:
  template:
    spec:
      domain:
        devices:
          interfaces:
            - name: default
              state: down
              masquerade: { }
      networks:
        - name: default
          pod: { }
```

可以在以下时机指定期望的链路状态：

1. 创建 VM 时。
2. VM 正在运行时。
3. VM 未运行时。
4. 热插拔新的网络接口时。

> **警告：** 当 VM 上指定了 [HTTP / TCP 就绪探针和/或存活探针](../user_workloads/liveness_and_readiness_probes.md) 时，将主接口的链路状态设置为 `down` 会导致该 VM：
>
> 1. 被标记为未就绪（就绪探针）
> 2. 被重启，因为 kubelet 会终止 virt-launcher Pod（存活探针）

### 当前链路状态 {#current-link-state-status}

网络接口的当前链路状态报告在 VirtualMachineInstance 状态部分：

```yaml
apiVersion: kubevirt.io/v1
kind: VirtualMachineInstance
metadata:
  name: my-vm
spec:
  domain:
    devices:
      interfaces:
      - name: default
        state: down
        masquerade: { }
  networks:
  - name: default
    pod: { }
status:
  interfaces:
    - name: default
      linkState: down
```

> **注意：** KubeVirt 不报告 SR-IOV 设备的当前链路状态，因为它无法跟踪对这些设备所做的外部更改。

> **注意：** `passtBinding` 不支持链路状态管理功能。

## 安全性 {#security}

### MAC 欺骗检查 {#mac-spoof-check}

MAC 欺骗是指能够使用任意源 MAC 地址生成流量。
攻击者可能利用此能力对网络发起攻击。

为防范此类情形，可以在支持该功能的 CNI 插件中启用 mac-spoof-check。

由集群网络提供方提供的 Pod 主网络不在本文档范围内。请参阅相应提供方，了解如何启用欺骗检查。
以下内容针对使用 multus 提供的辅助网络。

已知有两个支持 mac-spoof-check 的 CNI 插件：

- [sriov-cni](https://github.com/openshift/sriov-cni)：
  通过 `spoofchk` 参数。
- [bridge-cni](https://www.cni.dev/plugins/current/main/bridge/)：
  通过 `macspoofchk` 参数。

此配置由运维人员在 NetworkAttachmentDefinition 上完成，任何引用它的接口都将启用此功能。

以下是使用 `bridge` CNI 并启用 `macspoofchk` 的示例：

```yaml
apiVersion: "k8s.cni.cncf.io/v1"
kind: NetworkAttachmentDefinition
metadata:
  name: br-spoof-check
spec:
  config: '{
            "cniVersion": "0.3.1",
            "name": "br-spoof-check",
            "type": "bridge",
            "bridge": "br10",
            "disableContainerInterface": true,
            "macspoofchk": true
        }'
```

在 VM 上，网络部分应按名称指向此 NetworkAttachmentDefinition：

```yaml
networks:
- name: default
  pod: {}
- multus:
    networkName: br-spoof-check
  name: br10
```

## 限制与已知问题 {#limitations-and-known-issues}

### 对辅助网络无效的 CNI {#invalid-cnis-for-secondary-networks}

以下 CNI 已知**不能**用于网桥接口——网桥接口是辅助接口中最常见的类型。

- [macvlan](https://www.cni.dev/plugins/current/main/macvlan/)

- [ipvlan](https://www.cni.dev/plugins/current/main/ipvlan/)

原因类似：bridge 接口类型会把 Pod 接口的 MAC 地址移到 VM，使 Pod 接口使用不同的地址。上述 CNI 要求 Pod 接口保留原始 MAC 地址。

这些问题分别在以下问题中跟踪：

- [macvlan](https://github.com/kubevirt/kubevirt/issues/5483)

- [ipvlan](https://github.com/kubevirt/kubevirt/issues/7001)

欢迎就这些问题进行讨论和/或提出修复；我们希望这些插件能成为我们生态中的有效选项。

- `bridge` CNI 通过 nftables 支持 mac-spoof-check，因此节点必须支持 nftables，并部署 `nft` 二进制文件。


## 补充说明 {#additional-notes}

### MTU {#mtu}

有两种方法可以将 MTU 传播到客户机接口。

* libvirt - 为此，客户机需要足够新的 virtio 网络驱动，以理解通过模拟设备中的 PCI 配置寄存器传入客户机的数据。
* DHCP - 为此，客户机 DHCP 客户端应能从 DHCP 服务器的响应中读取 MTU。

在 **Windows** 客户机的非 virtio 接口上，必须使用 `netsh` 或其他工具手动设置 MTU，因为 Windows DHCP 客户端不会请求/读取 MTU。

下表汇总了 MTU 向客户机的传播方式。

|            | masquerade      | 带 CNI IP 的 bridge | 不带 CNI IP 的 bridge | Windows |
|------------|-----------------|---------------------|-----------------------|---------|
| virtio     | DHCP 与 libvirt | DHCP 与 libvirt     | libvirt               | libvirt |
| non-virtio | DHCP            | DHCP                | X                     | X       |

* 带 CNI IP 的 bridge - 指 CNI 向 Pod 接口提供 IP，并使用 bridge 绑定将 Pod 接口绑定到客户机。

### virtio-net 多队列 {#virtio-net-multiqueue}

将 `networkInterfaceMultiqueue` 设置为 `true` 会启用多队列功能，从而增加以 `virtio` 型号配置的接口的 vhost 队列数量。

```yaml
# partial example - kept short for brevity 
apiVersion: kubevirt.io/v1
kind: VirtualMachine
spec:
  template:
    spec:
      domain:
        devices:
          networkInterfaceMultiqueue: true
```

拥有多个 vCPU 的虚拟机用户可以从更高的网络吞吐量和性能中获益。

目前，队列数量由 VM 的 vCPU 数量决定。这是因为多队列支持会优化 RX 中断亲和性与 TX 队列选择，使特定队列专属于特定 vCPU。

如果不启用此功能，网络性能不会随 vCPU 数量增加而提升。客户机无法并行发送或接收数据包，因为 virtio-net 只有一个 TX 队列和一个 RX 队列。

Virtio 接口会在其 status.interfaces.interface 条目中公布名为 queueCount 的字段。
queueCount 字段表示分配给该接口的队列数量。
队列数量的值来自 domain XML。
如果无法确定队列数量（即仅由 quest-agent 报告的接口），则省略该字段。


>*注意*：尽管 virtio-net 多队列功能能带来性能收益，但它也有一些限制，因此不应无条件启用。

#### 一些已知限制 {#some-known-limitations}

-   客户机操作系统的 MSI 向量上限约为 200。每个网卡队列都需要一个 MSI 向量，任何 virtio 设备或已分配的 PCI 设备也是如此。定义具有多个 virtio 网卡和多个 vCPU 的实例，有可能触及客户机 MSI 上限。

-   virtio-net 多队列对入站流量效果良好，但偶尔会导致出站流量性能下降。具体来说，通过传输控制协议（TCP）流发送小于 1,500 字节的数据包时可能出现这种情况。

-   启用 virtio-net 多队列会提高网络总吞吐量，但同时也会增加 CPU 消耗。

-   在宿主机 QEMU 配置中启用 virtio-net 多队列，并不会在客户机操作系统中启用该功能。客户机操作系统管理员需要使用 ethtool，为每个需要此功能的客户机网卡手动开启。

-   如果在宿主机上启用了多队列，但管理员尚未在客户机操作系统中启用，MSI 向量仍会被消耗（浪费）。

-   如果客户机实例中的 vNIC 数量与 vCPU 数量成比例，启用多队列功能的重要性就会降低。

-   每个 virtio-net 队列会为 vhost 驱动消耗 64 KiB 内核内存。

>*注意*：应使用 ethtool 在客户机操作系统中手动启用 Virtio-net 多队列。例如：
`ethtool -L <NIC> combined #num_of_queues`

更多信息请参阅 [KVM/QEMU 多队列](http://www.linux-kvm.org/page/Multiqueue)。
