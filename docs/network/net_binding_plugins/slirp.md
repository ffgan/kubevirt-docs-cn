# Slirp {#slirp}

## 概述 {#overview}

[SLIRP](https://en.wikipedia.org/wiki/Slirp) 提供用户空间网络连通性。

> **注意：** 在 `slirp` 模式下，唯一受支持的协议是 TCP 和 UDP。ICMP *不*受支持。

## Slirp 网络绑定插件 {#slirp-network-binding-plugin}

> **重要**：核心 SLIRP 绑定已在 v1.3.0 弃用并移除。本文档介绍的是 SLIRP 的**网络绑定插件**实现。

该绑定插件取代了[核心 `slirp` 绑定](../../network/interfaces_and_networks.md#slirp) API。

slirp 绑定插件由以下组件组成：

- 边车镜像。

如[定义与流程](../../network/network_binding_plugins.md#definition--flow)一节所述，slirp 插件需要：

- 确保能够访问边车镜像。
- 在 KubeVirt CR 上注册该绑定插件。
- 从 VM spec 的接口中按名称引用该网络绑定。

> **注意**：此绑定的注册名称应当为 `slirp`，以便与引用 slirp 绑定的现有 VM 规约保持兼容。

### 功能门控 {#feature-gate}

自 v1.5.0 起，网络绑定插件功能默认启用，并且没有功能门控。

slirp 插件同样没有自己的功能门控，但需要通过[注册](./#slirp-registration)使其在集群中可用。

### Slirp 注册 {#slirp-registration}

如[注册一节](../../network/network_binding_plugins.md#register)所述，需要把 slirp 绑定插件的配置添加到 kubevirt CR 中。

要注册 slirp 绑定，请按如下方式修补 kubevirt CR：
```yaml
kubectl patch kubevirts -n kubevirt kubevirt --type=json -p='[{"op": "add", "path": "/spec/configuration/network",   "value": {
            "binding": {
                "slirp": {
                    "sidecarImage": "quay.io/kubevirt/network-slirp-binding:v1.1.0"
                }
            }
        }}]'
```

### VM 的 Slirp 网络接口 {#vm-slirp-network-interface}

将 VM 网络接口的绑定名称设置为 kubevirt CR 中所定义的名称。

```yaml
---
apiVersion: kubevirt.io/v1
kind: VirtualMachine
metadata:
  labels:
    kubevirt.io/vm: vm-net-binding-slirp
  name: vm-net-binding-slirp
spec:
  runStrategy: Always
  template:
    metadata:
      labels:
        kubevirt.io/vm: vm-net-binding-slirp
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
          - name: slirpnet
            binding:
              name: slirp
          rng: {}
        resources:
          requests:
            memory: 1024M
      networks:
      - name: slirpnet
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
