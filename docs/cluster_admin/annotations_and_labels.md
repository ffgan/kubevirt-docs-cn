# 注解与标签 {#annotations-and-labels}

KubeVirt 构建并公开了若干标签和注解，它们要么用于内部实现需求，要么向 API 用户公开有用信息。本页记录对常规 API 使用者可能有用的标签和注解。本页有意*不*列出仅属于内部实现的标签和注解。

**注意：**并非 KubeVirt 专有的注解和标签也记录在
[此处](https://kubernetes.io/docs/reference/kubernetes-api/labels-annotations-taints/)。

## kubevirt.io {#kubevirtio}

示例：`kubevirt.io=virt-launcher`

用于：Pod

此标签标记属于 KubeVirt 的资源。可选值可以表明资源属于哪个具体的 KubeVirt 组件。此标签可用于列出属于 KubeVirt 的全部资源，例如从集群中卸载它。

### kubevirt.io/schedulable {#kubevirtioschedulable}

示例：`kubevirt.io/schedulable=true`

用于：节点

此标签声明某个节点是否可用于在其上调度虚拟机实例。

### kubevirt.io/heartbeat {#kubevirtioheartbeat}

示例：`kubevirt.io/heartbeat=2018-07-03T20:07:25Z`

用于：节点

此注解由 virt-handler 定期更新，用于帮助判断某个节点是否存活，从而决定它是否可用于新的虚拟机实例调度。
