# 实时更新 NAD 引用 {#live-update-nad-reference}

版本：

- v1.8.0: Beta
- v1.9.0: GA

KubeVirt 支持在正在运行的虚拟机（VM）上更新 NetworkAttachmentDefinition（NAD）引用，而无需重启。用户可以通过修改 `networkName` 字段并等待 VMI 的实时迁移，来更改 VM 所连接的网络。

当需要在保持相同客户机接口属性（例如名称或 MAC 地址）的同时，把 VM 在不同网络之间移动（例如不同的 VLAN）时，此功能非常有用。

## 要求 {#requirements}

要使用此功能，必须满足以下要求：

- VM 必须能够实时迁移
- 必须启用 `LiveUpdateNADRef` [功能门控](https://kubevirt.io/user-guide/operations/activating_feature_gates/#how-to-activate-a-feature-gate)（仅 v1.8 需要）
- 目标 NetworkAttachmentDefinition 必须存在

## 在正在运行的 VM 上更新 NAD 引用的过程 {#process-of-updating-nad-reference-on-a-running-vm}

当前 NAD
```yaml
apiVersion: k8s.cni.cncf.io/v1
kind: NetworkAttachmentDefinition
metadata:
  name: nad-with-vlan10
spec:
  config: '{
    "cniVersion": "0.3.1",
    "name": "nad-with-vlan10",
    "type": "bridge",
    "bridge": "br1",
    "vlan": 10
  }'
```

目标 NAD
```yaml
apiVersion: k8s.cni.cncf.io/v1
kind: NetworkAttachmentDefinition
metadata:
  name: nad-with-vlan20
spec:
  config: '{
    "cniVersion": "0.3.1",
    "name": "nad-with-vlan20",
    "type": "bridge",
    "bridge": "br2",
    "vlan": 20
  }'
```

从一台正在运行、并连接到附加于当前 NAD 的辅助网络的 VM 开始：

```yaml
---
apiVersion: kubevirt.io/v1
kind: VirtualMachine
...
  template:
    spec:
      domain:
        devices:
          interfaces:
          - bridge: {}
            name: bridge-net
...
      networks:
      - name: bridge-net
        multus:
          networkName: nad-with-vlan10
...
```

要更改网络，请编辑 VM spec 模板并更新 `networkName` 字段：

```yaml
apiVersion: kubevirt.io/v1
kind: VirtualMachine
...
  template:
    spec:
      domain:
        devices:
          interfaces:
          - bridge: {}
            name: bridge-net
      networks:
      - name: bridge-net
        multus:
          networkName: nad-with-vlan20  # Updated from nad-with-vlan10
```

此更改会触发一次实时迁移，进而创建一个连接到新 NAD 的新 Pod。更多信息请参阅[实时迁移](../../compute/live_migration.md)文档。

## 限制 {#limitations}

- 此功能不支持与 [Multus Dynamic networks Controller](https://github.com/k8snetworkplumbingwg/multus-dynamic-networks-controller) 一起使用
- 迁移到不同的 CNI 类型可能会失败，尤其是当新的 CNI 类型需要额外配置时。
- 不支持切换到新的绑定类型/绑定插件。
- 只有可迁移的 VM 才能更新，因为无法在不可迁移的 VMI 上触发迁移；这类 VMI 的状态中具有 `LiveMigratable`: `False` 状况。
- 实时迁移过程中，网络连通性可能会中断。
- 更新多个 VM 时，并行运行的迁移数量有限，因此更新可能会排队。更多信息请参阅[更改集群范围的迁移配置](../../compute/live_migration.md#changing-cluster-wide-migration-configuration)。
