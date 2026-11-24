# Passt 绑定 {#passt-binding}

!!! Warning 
    passt 网络绑定插件已在 v1.8.0 中被[核心绑定](../../interfaces_and_networks#passtbinding)取代。

## 概述 {#overview}

[Plug A Simple Socket Transport](https://passt.top/passt/about/) 是 [SLIRP](https://en.wikipedia.org/wiki/Slirp) 的增强替代方案，提供用户空间网络连通性。

`passt` 是一个通用工具，它在二层网络接口与宿主机上的原生四层套接字（TCP、UDP、ICMP/ICMPv6 echo）之间实现转换层。

其主要优点如下：

- 不需要 CAP_NET_RAW 和 CAP_NET_ADMIN 这类额外网络权能。
- 开箱即用地支持与服务网格集成（服务网格期望应用在本地运行）。
- 开箱即用支持 IPv6（现有绑定需要手动配置 IPv6）。

### 功能支持 {#functionality-support}

| 功能 | 支持情况 |
|------------------------------------------------|---------|
| 迁移支持 | 是 |
| 服务网格支持 | 是 |
| 客户机中的 Pod IP | 是 |
| 客户机中的自定义 CIDR | 否 |
| 运行需要额外权能（在 Pod 上） | 否 |
| 主网络（Pod 网络） | 是 |
| 辅助网络 | 否 |


> **注意**：要在使用 Passt 绑定时获得最优的内存消耗，请指定工作负载所需的端口。未显式指定任何端口时，所有端口都会被转发，从而带来最高达 250 Mi 的内存开销。

## Passt 网络绑定插件 {#passt-network-binding-plugin}

[v1.1.0-v1.8.0]

该绑定插件取代了实验性的核心 passt 绑定实现（包括其 API）。

passt 绑定插件由以下组件组成：

- Passt CNI 插件。
- 边车镜像。

如[定义与流程](../../network/network_binding_plugins.md#definition--flow)一节所述，passt 插件需要：

- 在节点上部署 CNI 插件二进制文件。
- 定义一个指向该 CNI 插件的 NetworkAttachmentDefinition。
- 确保能够访问边车镜像。
- 在 KubeVirt CR 上注册该绑定插件。
- 从 VM spec 的接口中按名称引用该网络绑定。

详细说明如下：

### 在节点上部署 Passt CNI {#passt-cni-deployment-on-nodes}

#### v1.6.0 及更高版本 {#v160-and-above}

KubeVirt 发布了容器镜像 `quay.io/kubevirt/network-passt-binding-cni:v1.6.0`，可以通过 DaemonSet 用它把 CNI 二进制文件复制到节点上。

创建如下 DaemonSet：

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: passt-binding-cni
  namespace: kubevirt
  labels:
    tier: node
    app: passt-binding-cni
spec:
  selector:
    matchLabels:
      name: passt-binding-cni
  updateStrategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 10%
  template:
    metadata:
      labels:
        name: passt-binding-cni
        tier: node
        app: passt-binding-cni
      annotations:
        description: passt-binding-cni installs 'passt binding' CNI on cluster nodes
    spec:
      priorityClassName: system-cluster-critical
      containers:
      - name: installer
        image: quay.io/kubevirt/network-passt-binding-cni:v1.6.0
        command: [ "/bin/sh", "-ce" ]
        args:
        - |
          ls -la "/cni/kubevirt-passt-binding"
          cp -f "/cni/kubevirt-passt-binding" "/opt/cni/bin"
          echo "passt binding CNI plugin installation complete..sleep infinity"
          sleep 2147483647
        resources:
          requests:
            cpu: "10m"
            memory: "15Mi"
        securityContext:
          privileged: true
        volumeMounts:
        - name: cnibin
          mountPath: /opt/cni/bin
      volumes:
      - name: cnibin
        hostPath:
          path: /opt/cni/bin
```

#### v1.6.0 之前 {#prior-to-v160}

可以直接从 kubevirt 的发布资产（位于 GitHub）获取该 CNI 二进制文件，也可以从其[源码](https://github.com/kubevirt/kubevirt/tree/release-1.1/cmd/cniplugins/passt-binding)构建。

> **注意**：kubevirt 项目使用 Bazel 构建二进制文件和容器镜像。关于如何构建整个项目的更多信息，请访问开发者[入门指南](https://github.com/kubevirt/kubevirt/blob/release-1.1/docs/getting-started.md)。

二进制文件准备好之后，可以把它重命名为一个有意义的名称（例如 `kubevirt-passt-binding`）。这个名称会用在 NetworkAttachmentDefinition 的配置中。

把该二进制文件复制到集群中的每个节点上。CNI 插件的位置可能因平台和版本而异。一个常见路径是 `/opt/cni/bin/`。

### Passt NetworkAttachmentDefinition {#passt-networkattachmentdefinition}

passt 所需的配置非常精简：

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
                "type": "kubevirt-passt-binding"
              }
            ]
  }'
```

该对象应当创建在其他所有命名空间都能访问的 “default” 命名空间中，或者创建在 VM 所在的同一命名空间中。

### Passt 边车镜像 {#passt-sidecar-image}

Passt 边车镜像已构建并推送到 [kubevirt quay 仓库](https://quay.io/repository/kubevirt/network-passt-binding)。

边车源码见[此处](https://github.com/kubevirt/kubevirt/tree/release-1.1/cmd/sidecars/network-passt-binding)。

集群需要能够访问相应的边车镜像，并在注册网络绑定插件时，在 KubeVirt CR 中指定它。

### 功能门控 {#feature-gate}

自 v1.5.0 起，网络绑定插件功能默认启用，并且没有功能门控。passt 插件同样没有自己的功能门控，但需要通过[注册](./#passt-registration)使其在集群中可用。

### Passt 注册 {#passt-registration}

如[注册一节](../../network/network_binding_plugins.md#register)所述，需要把 passt 绑定插件的配置添加到 kubevirt CR 中。

要注册 passt 绑定，请按如下方式修补 kubevirt CR：
```yaml
kubectl patch kubevirts -n kubevirt kubevirt --type=json -p='[{"op": "add", "path": "/spec/configuration/network",   "value": {
            "binding": {
                "passt": {
                    "networkAttachmentDefinition": "default/netbindingpasst",
                    "sidecarImage": "quay.io/kubevirt/network-passt-binding:v1.6.0",
                    "migration": {},
                    "computeResourceOverhead": {
                      "requests": {
                        "memory": "250Mi",
                      }
                    }
                }
            }
        }}]'
```

NetworkAttachmentDefinition 和 sidecarImage 的值应当与前面几节中使用的名称相对应，见[此处](#passt-networkattachmentdefinition)和[此处](#passt-sidecar-image)。

使用该插件时，会为 virt-launcher Pod 中的 compute 容器额外请求 `250Mi` 内存开销。

VM 首次启动时，Passt 的内部 DHCP 服务器会以“无限”租约时长为客户机分配一个 IP 地址。

**注意**：
> 在实时迁移期间：
> - 目标 Pod 会从 Pod 网络分配到一个新的 IP 地址，这会反映在 VMI 接口状态中。
> - 客户机操作系统会保留原来的 IP 地址，直到它向 Passt 的内部 DHCP 服务器显式请求地址。
>
> 连通性方面的考虑
> Passt 在传递客户机出入流量时会自动应用网络地址转换（NAT）。
> 这会实现以下行为：
> - 连通时使用 Pod 的新 IP 地址
> - 尽管 Pod IP 与客户机 IP 不一致，客户机的出入连通性不受影响。
>
> 可以从客户机操作系统发起 DHCP 请求，使客户机与 Pod 的 IP 地址保持同步。

### VM 的 Passt 网络接口 {#vm-passt-network-interface}

将 VM 网络接口的绑定名称设置为 kubevirt CR 中所定义的名称。

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
            ports:
            - name: http
              port: 80
              protocol: TCP
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
          image: quay.io/kubevirt/fedora-with-test-tooling-container-disk:v1.6.0
        name: containerdisk
      - cloudInitNoCloud:
          networkData: |
            version: 2
            ethernets:
              eth0:
                dhcp4: true
        name: cloudinitdisk
```
