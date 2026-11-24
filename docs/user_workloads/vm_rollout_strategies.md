# VM 滚动更新策略 {#vm-rollout-strategies}

在 KubeVirt 中，VM 滚动更新策略定义对 VM 对象的变更如何影响正在运行的客户机。  
换句话说，它定义对 VM 对象的变更何时以及如何传播到对应的 VMI 对象。

目前有 2 种滚动更新策略：`LiveUpdate` 和 `Stage`。
只能指定 1 种，默认是 `Stage`。

## LiveUpdate {#liveupdate}

`LiveUpdate` VM 滚动更新策略会尽快把 VM 对象的变更传播到正在运行的 VMI。  
例如，更改 CPU socket 数量会触发 [CPU 热插拔](../compute/cpu_hotplug.md)。

在 KubeVirt CR 中启用 `LiveUpdate` VM 滚动更新策略：

```yaml
apiVersion: kubevirt.io/v1
kind: KubeVirt
spec:
  configuration:
    vmRolloutStrategy: "LiveUpdate"
```

## Stage {#stage}

`Stage` VM 滚动更新策略会暂存对 VM 对象所做的每一项变更，直到它下一次重启。  

```yaml
apiVersion: kubevirt.io/v1
kind: KubeVirt
spec:
  configuration:
    vmRolloutStrategy: "Stage"
```

## RestartRequired 状况 {#restartrequired-condition}

当滚动更新策略为 `Stage` 时，对 VM 对象所做的任何变更都会触发 `RestartRequired` VM 状况。  
当滚动更新策略为 `LiveUpdate` 时，只有无法传播的变更才会触发该状况。

一旦在 VM 对象上设置了 `RestartRequired` 状况，就无法再传播后续变更，即使策略设置为 `LiveUpdate` 也是如此。  
变更将在下一次重启时生效，该状况也会被移除。

## 限制 {#limitations}

当前实现有以下限制：

- 一旦设置了 `RestartRequired` 状况，消除它的唯一办法是重启 VM。未来我们计划实现一种办法：把 VM 模板 spec 还原到最近一次非 RestartRequired 状态，从而消除该状况。
- 集群默认值不在此逻辑范围内。这意味着，更改会影响 VM spec 的集群范围设置时，无论滚动更新策略如何，都不会实时更新。
- `RestartRequired` 状况附带一条消息，说明是哪类变更触发了该状况（CPU/内存/其他）。该消息只针对触发该状况的第一次变更。通常会触发该状况的后续变更只会暂存，不会再添加额外的 `RestartRequired` 状况。
