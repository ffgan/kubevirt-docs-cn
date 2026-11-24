# 将 VM 连接到由 DRA 支持的网络 {#connecting-vms-to-dra-backed-networks}

**功能状态：** v1.10.0: Beta

KubeVirt 可以把虚拟机连接到由 Kubernetes [动态资源分配（DRA）](https://kubernetes.io/docs/concepts/scheduling-eviction/dynamic-resource-allocation/) 支持的网络。设备通过 Kubernetes `ResourceClaim` API 请求，并由 DRA 驱动分配。VM 的网络声明一个新的 `resourceClaim` 源，该源通过 `claimName` 和 `requestName` 引用声明和请求，VM 再通过由匹配的[网络绑定插件](network_binding_plugins.md)配置的接口连接到该网络。

DRA 驱动与网络绑定插件通过其作者定义的契约进行互操作。请选择相互兼容的驱动和绑定插件，并参阅它们的文档，了解二者共同支持的能力。

!!! Note
    DRA 驱动和网络绑定插件都是外部的第三方组件。它们不在 KubeVirt 以及本指南的范围内。

!!! Note
    截至 KubeVirt v1.10，由 DRA 支持的网络只能与[网络绑定插件](network_binding_plugins.md)一起使用。任何核心绑定都不支持 DRA。

## 管理员与用户的职责 {#admin-vs-user-responsibility}

DRA 网络流程分属两个角色，在使用由 DRA 支持的网络之前，二者都必须就绪：

- **集群管理员**准备集群：启用 `NetworkDevicesWithDRA` 功能门控，并部署第三方组件。具体检查清单见[前提条件](#prerequisites)。
- **VM 所有者（用户）**创建 `ResourceClaim`/`ResourceClaimTemplate`，以及引用它的 VM 清单；其中的接口使用已注册的绑定插件，网络使用 `resourceClaim` 源。

## 功能门控 {#feature-gate}

`resourceClaim` 网络源由 `NetworkDevicesWithDRA` 功能门控控制。截至 KubeVirt v1.10，该功能门控处于 Beta 阶段，并且默认启用。

某些 KubeVirt 发行版可能会禁用它。如果在你的集群中它处于禁用状态，请按[激活功能门控](../cluster_admin/activating_feature_gates.md)中的说明，在 KubeVirt CR 上启用它。

## 前提条件 {#prerequisites}

- **已启用 `NetworkDevicesWithDRA` 功能门控。**

- **网络设备 DRA 驱动。** 必须在集群中部署面向目标网络设备类型的外部 DRA 驱动。

- **一个 [DeviceClass](https://kubernetes.io/docs/reference/kubernetes-api/resource/device-class-v1/)。** `DeviceClass` 是管理员定义的一类设备。`ResourceClaim`/`ResourceClaimTemplate` 对象要引用它，它必须已经存在。

- **已注册的网络绑定插件。** 用于配置由 DRA 支持的接口的绑定插件，必须在 KubeVirt CR 中注册。集群管理员如何注册，请参阅[网络绑定插件](network_binding_plugins.md)。

## 示例：将 VM 连接到由 DRA 支持的网络 {#example-connecting-a-vm-to-a-dra-backed-network}

下面的示例通过 `ResourceClaimTemplate` 请求一个设备，然后把 VM 连接到由 DRA 支持的网络。它假定集群管理员已经在 KubeVirt CR 中注册了绑定插件 `my-binding-plugin`。

首先，创建一个向 DRA 驱动请求设备的 `ResourceClaimTemplate`：

```yaml
apiVersion: resource.k8s.io/v1
kind: ResourceClaimTemplate
metadata:
  name: network-claim-template
spec:
  spec:
    devices:
      requests:
      - name: dev
        exactly:
          deviceClassName: my-device-class
```

!!! Note
    设备请求字段取决于你的 DRA 驱动。关于如何正确定义 `ResourceClaimTemplate`，请参阅该驱动的文档。

然后创建一个 VM：在 `spec.template.spec.resourceClaims` 中引用该声明模板，声明一个使用该绑定插件的接口，并添加一个带有 `resourceClaim` 源、指向该声明和请求的网络：

```yaml
apiVersion: kubevirt.io/v1
kind: VirtualMachine
metadata:
  name: vm-dra-binding-plugin
spec:
  template:
    spec:
      domain:
        devices:
          interfaces:
          - name: dra-net
            binding:
              name: my-binding-plugin
      networks:
      - name: dra-net
        resourceClaim:
          claimName: network-claim
          requestName: dev
      resourceClaims:
      - name: network-claim
        resourceClaimTemplateName: network-claim-template
```

!!! warning "重要"
    **这些字段必须相互引用：**

    - `spec.template.spec.resourceClaims[].resourceClaimTemplateName` 必须与 `metadata.name` 匹配，该字段属于 `ResourceClaimTemplate`（本例中为 `network-claim-template`）。
    - `spec.template.spec.networks[].resourceClaim.claimName` 必须与为该设备提供支持的声明所对应的 `spec.template.spec.resourceClaims[].name` 匹配（本例中为 `network-claim`）。
    - `spec.template.spec.networks[].resourceClaim.requestName` 必须与请求 `name` 匹配，该请求定义在 `ResourceClaimTemplate` 中（本例中为 `dev`）。

    接口与网络通过它们共享的 `name` 相关联（本例中为 `dra-net`），与其他任何 KubeVirt 接口/网络对完全一样。

## 校验规则 {#validation-rules}

KubeVirt 在创建 VM/VMI 时校验 DRA 网络配置。你的配置必须满足以下全部条件：

- 映射到 `resourceClaim` 网络的接口**必须**使用网络绑定插件。
- `claimName` 和 `requestName` 都必须设置（非空）。
- `claimName` + `requestName` 元组在所有网络、`hostDevices` 和 `gpus` 中必须**唯一**，任意两个条目都不能共用同一个声明和请求。
- 同一个 VMI 中不能混用 `multus` 和 `resourceClaim` 网络源。

## 已知限制 {#known-limitations}

- **实时迁移**仅支持由 `resourceClaimTemplate` 支持的声明，不支持直接的 `resourceClaim` 引用。
- **每个请求一个设备。** 如果一个请求分配了多个设备（`count > 1`），则只会使用第一个已分配的设备。
