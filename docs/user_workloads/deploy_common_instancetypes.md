# 部署 common-instancetypes {#deploy-common-instancetypes}

[`kubevirt/common-instancetypes`](https://github.com/kubevirt/common-instancetypes) 提供一组[实例类型和偏好](../user_workloads/instancetypes.md)，用以帮助创建 KubeVirt [`VirtualMachines`](http://kubevirt.io/api-reference/main/definitions.html#_v1_virtualmachine)。

从 KubeVirt 1.1 发行版开始，可以不借助另一个 operator，直接通过 KubeVirt 部署集群范围的资源。这样就可以随 KubeVirt 一并部署一组默认实例类型和偏好。

从 KubeVirt [`v1.4.0`](https://github.com/kubevirt/kubevirt/releases/tag/v1.4.0) 发行版起，common-instancetypes 现在默认会部署。

## **功能状态：** {#feature-state}

* Alpha - [`v1.1.0`](https://github.com/kubevirt/kubevirt/releases/tag/v1.1.0)
* Beta - [`v1.2.0`](https://github.com/kubevirt/kubevirt/releases/tag/v1.2.0)
* GA - [`v1.4.0`](https://github.com/kubevirt/kubevirt/releases/tag/v1.4.0)（默认启用）

## 控制 common-instancetypes 的部署 {#control-deployment-of-common-instancetypes}

要通过 KubeVirt `virt-operator` 显式启用或禁用集群范围 common-instancetypes 的部署，请使用可配置项 `spec.configuration.commonInstancetypesDeployment.enabled`。

```shell
kubectl patch -n kubevirt kv/kubevirt --type merge -p '{"spec":{"configuration":{"commonInstancetypesDeployment":{"enabled": false}}}}'
```

## 手动部署 common-instancetypes {#deploy-common-instancetypes-manually}

出于定制目的，或为了安装命名空间范围的资源，也可以手动部署 common-instancetypes。

要安装 [`kubevirt/common-instancetypes`](https://github.com/kubevirt/common-instancetypes) 项目提供的全部资源且不做进一步定制，只需在启用 [`kustomize`](https://kustomize.io/) 的情况下 apply（-k 标志）：

```shell
kubectl apply -k https://github.com/kubevirt/common-instancetypes.git
```

另外，每种可用的自定义资源类型（例如命名空间范围的实例类型）都有对应的目标。

例如，要部署 `VirtualMachineInstancetypes`，请运行以下命令：

```shell
kubectl apply -k https://github.com/kubevirt/common-instancetypes.git/VirtualMachineInstancetypes
```
