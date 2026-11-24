# 虚拟机内存转储 {#virtual-machine-memory-dump}

KubeVirt 现在支持获取 VM 内存转储，以供分析之用。
内存转储可用于诊断、识别并解决 VM 中的问题。它通常提供程序、应用程序和系统在被终止或崩溃之前最后状态的信息。

> *注意* 此内存转储不用于保存 VM 状态并在以后恢复。

## 前提条件 {#prerequisites}

### 热插拔功能门控 {#hot-plug-feature-gate}

内存转储流程会将 PVC 挂载到 virt-launcher，以便在该 PVC 中获取输出，因此必须启用热插拔卷功能门控。必须通过添加 `HotplugVolumes` 来扩展 KubeVirt CR 中的[功能门控](../cluster_admin/activating_feature_gates.md#how-to-activate-a-feature-gate)字段。

## virtctl 支持 {#virtctl-support}

### 获取内存转储 {#get-memory-dump}

现在假设我们有一台正在运行的 VM，该 VM 的名称是 'my-vm'。
我们可以转储到现有的 PVC，也可以请求创建一个 PVC。

#### 现有 PVC {#existing-pvc}
PVC 的大小必须足够容纳内存转储。计算公式为 (VMMemorySize + 100Mi) * FileSystemOverhead，其中 `VMMemorySize` 是内存大小，
100Mi 是为内存转储开销预留的空间，`FileSystemOverhead` 是用于按文件系统开销调整所请求 PVC 大小的值。
此外，PVC 必须具有 `FileSystem` 卷模式。

此类 PVC 的示例：

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: my-pvc
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 2Gi
  storageClassName: rook-ceph-block
  volumeMode: Filesystem
```

我们可以使用 virtctl 提供的 'memory-dump get' 命令，将 VM 的内存转储获取到该 PVC。

```bash
$ virtctl memory-dump get my-vm --claim-name=my-pvc
```

#### 按需 PVC {#on-demand-pvc}
对于按需 PVC，我们需要向 virtctl 请求添加 `--create-claim` 标志：

```bash
$ virtctl memory-dump get my-vm --claim-name=new-pvc --create-claim
```

将创建一个大小足以容纳转储的 PVC。我们还可以使用相应标志请求特定的存储类和访问模式。

#### 下载内存转储 {#download-memory-dump}
添加 `--output` 标志后，内存将被转储到 PVC，然后下载到给定的输出路径。

```bash
$ virtctl memory-dump get myvm --claim-name=memoryvolume --create-claim --output=memoryDump.dump.gz
```

要从与 VM 关联的 PVC 下载最近一次内存转储，而不触发另一次内存转储，请使用内存转储下载命令。

```bash
$ virtctl memory-dump download myvm --output=memoryDump.dump.gz
```

要从已经与 VM 解除关联的 PVC 下载内存转储，可以使用 [virtctl vmexport 命令](https://github.com/kubevirt/user-guide/blob/main/docs/operations/export_api.md)

### 监控内存转储 {#monitoring-the-memory-dump}
有关内存转储流程的信息将出现在 VM 的 status 部分。
```yaml
memoryDumpRequest:
  claimName: memory-dump
  phase: Completed
  startTimestamp: "2022-03-29T11:00:04Z"
  endTimestamp: "2022-03-29T11:00:09Z"
  fileName: my-vm-my-pvc-20220329-110004
```

在此过程中，VMI 上的 volumeStatus 会更新流程信息，例如挂载 Pod 的信息和消息；如果一切顺利，一旦流程完成，PVC 会从 virt-launcher Pod 卸载，并且 volumeStatus 会被删除。
内存转储注解将被添加到 PVC 上，其中包含内存转储文件名。

### 重新触发内存转储 {#retriggering-the-memory-dump}
无需使用任何标志，即可将新的内存转储获取到同一个 PVC：
```bash
$ virtctl memory-dump get my-vm
```

> *注意* 每条 memory-dump 命令都会删除该 PVC 中先前的转储。

要将内存转储获取到不同的 PVC，你需要 'remove' 当前的 memory-dump PVC，然后使用新的 PVC 名称执行新的 get。

### 移除内存转储 {#remove-memory-dump}

如前所述，要移除关联的内存转储 PVC，你需要运行 'memory-dump remove' 命令。这样你就可以替换当前 PVC，并将内存转储获取到新的 PVC。

```bash
$ virtctl memory-dump remove my-vm
```

## 处理内存转储 {#handle-the-memory-dump}
内存转储流程完成后，PVC 将保存输出。
你可以通过以下方式之一管理该转储：
- 下载内存转储
- 创建一个带有故障排查工具的 Pod，由它挂载 PVC 并在 Pod 内进行检查。
- 将内存转储包含在 VM 快照中（将同时包含内存转储和磁盘），以保存该时间点的 VM 快照，并在需要时进行检查。（VM 快照可以导出和下载）。

可以使用内存分析工具检查内存转储的输出，例如 [Volatility3](https://github.com/volatilityfoundation/volatility3)。
