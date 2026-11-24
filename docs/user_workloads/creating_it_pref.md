# 使用 virtctl 创建实例类型和偏好 {#creating-instance-types-and-preferences-by-using-virtctl}

从 KubeVirt v1.0 起，你可以使用 virtctl 子命令创建实例类型和偏好。

## 创建实例类型 {#creating-instance-types}

virtctl 子命令 `create instancetype` 让你能够从命令行轻松创建实例类型清单。该命令还提供若干标志，可用于创建所需的清单。

有两个必须指定的标志：

1. `--cpu`：要请求的 vCPU 数量
2. `--memory`：要请求的内存量

此外还有若干可选标志，例如指定用于直通的 GPU 列表、选择所需的 IOThreadsPolicy，或直接提供实例类型的名称。

默认情况下，该命令创建集群范围的实例类型。如果用户想创建命名空间范围的版本，需要提供 namespaced 标志。可以使用 `--namespace` 标志指定命名空间名称。

要查看标志的完整列表及其说明，请使用以下命令：

```shell
virtctl create instancetype -h
```

### 示例 {#examples}

使用必需的 `--cpu` 和 `--memory` 标志，为 VirtualMachineClusterInstancetype 创建清单：

```shell
virtctl create instancetype --cpu 2 --memory 256Mi
```

为指定了命名空间的 VirtualMachineInstancetype 创建清单：

```shell
virtctl create instancetype --cpu 2 --memory 256Mi --namespace my-namespace
```

为未指定命名空间名称的 VirtualMachineInstancetype 创建清单：

```shell
virtctl create instancetype --cpu 2 --memory 256Mi --namespaced
```

## 创建偏好 {#creating-preferences}

virtctl 子命令 `create preference` 让你能够从命令行轻松创建偏好清单。该命令是创建偏好清单基本结构的起点，因为它不允许指定偏好所支持的全部选项。

当前这组标志允许我们指定例如首选 CPU 拓扑、机器类型或存储类。

默认情况下，该命令创建集群范围的偏好。如果用户想创建命名空间范围的版本，需要提供 namespaced 标志。可以使用 `--namespace` 标志指定命名空间名称。

要查看标志的完整列表及其说明，请使用以下命令：

```shell
virtctl create preference -h
```

### 示例 {#examples_1}

为具有首选 CPU 拓扑的 VirtualMachineClusterPreference 创建清单：

```shell
virtctl create preference --cpu-topology preferSockets
```

为指定了命名空间的 VirtualMachinePreference 创建清单：

```shell
virtctl create preference --namespace my-namespace
```

为具有首选存储类的 VirtualMachinePreference 创建清单：

```shell
virtctl create preference --namespaced --volume-storage-class my-storage
```
