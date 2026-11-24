# 快照恢复 API {#snapshot-restore-api}

`snapshot.kubevirt.io` API 组定义了用于为 KubeVirt `VirtualMachines` 创建快照和进行恢复的资源

## 先决条件 {#prerequisites}

### VolumeSnapshotClass {#volumesnapshotclass}

KubeVirt 利用 Kubernetes [CSI 驱动](https://kubernetes-csi.github.io/docs/drivers.html)的 `VolumeSnapshot` 功能来捕获持久的 `VirtualMachine` 状态。因此，应当确保你的 `VirtualMachine` 使用 `DataVolumes` 或 `PersistentVolumeClaims`，它们由 `StorageClass` 支撑，该存储类支持 `VolumeSnapshots`，并且已正确配置 `VolumeSnapshotClass` 以用于该 `StorageClass`。

KubeVirt 在 `v1` 版本中查找与 Kubernetes 卷快照相关的 API/资源。为确保 KubeVirt 的快照控制器能够按预期为 VirtualMachine 及所引用的卷创建快照，Kubernetes 卷快照 API 必须由 `v1` 版本提供。

要列出 `VolumeSnapshotClasses`：

```bash
kubectl get volumesnapshotclass
```

请确保 `provisioner` 属性（属于你的 `StorageClass`）与 `driver` 属性（属于 `VolumeSnapshotClass`）相匹配

即使集群中没有 `VolumeSnapshotClasses`，`VirtualMachineSnapshots` 也并非完全无用。它们仍会备份你的 `VirtualMachine` 配置。

### 快照功能门控 {#snapshot-feature-gate}

必须在功能门控中启用快照/恢复支持，该功能才受支持。必须扩展 KubeVirt CR 中的
[功能门控](../cluster_admin/activating_feature_gates.md#how-to-activate-a-feature-gate)
字段，向其中添加 `Snapshot`。


## 为 VirtualMachine 创建快照 {#snapshot-a-virtualmachine}

支持为在线和离线 VM 创建快照。

为正在运行的 VM 创建快照时，控制器会检查该 VM 中是否有 qemu 客户机代理。如果代理存在，它会在创建快照之前冻结 VM 文件系统，并在快照之后解冻。建议在有客户机代理的情况下创建在线快照，以获得更好的快照；如果不存在，则会创建尽力而为的快照。

> *Note* 要检查你的 VM 是否有 qemu-guest-agent，请在 VM status 中查找 'AgentConnected'。

vmSnapshot status 中会有一项指示，说明快照是在线创建的，以及是否有客户机代理参与。

> *Note* 支持对带有热插拔磁盘的 VM 进行在线快照，只有持久的热插拔磁盘会被包含在快照中。


要对 `VirtualMachine` `larry` 创建快照，请应用以下 yaml。

```yaml
apiVersion: snapshot.kubevirt.io/v1beta1
kind: VirtualMachineSnapshot
metadata:
  name: snap-larry
spec:
  source:
    apiGroup: kubevirt.io
    kind: VirtualMachine
    name: larry
```

可以在 vmSnapshot status 中查看 vmSnapshot 所处的阶段。它可以是以下之一：

- InProgress
- Succeeded
- Failed。

vmSnapshot 的默认截止时间是 5 分钟。如果 vmSnapshot 在截止时间之前没有成功完成，它将被标记为 Failed。如有必要，VM 将被解冻，已创建的快照内容将被清理。vmSnapshot 对象将保持 Failed 状态，直到用户删除它。要更改默认截止时间，请向 VirtualMachineSnapshot spec 添加 'FailureDeadline' 并赋予新值。允许的格式是 [时长](https://pkg.go.dev/time#ParseDuration) 字符串，即可能带符号的十进制数字序列，每个数字可以带有可选的小数部分和单位后缀，例如 "300ms"、"-1.5h" 或 "2h45m"

```yaml
apiVersion: snapshot.kubevirt.io/v1beta1
kind: VirtualMachineSnapshot
metadata:
  name: snap-larry
spec:
  source:
    apiGroup: kubevirt.io
    kind: VirtualMachine
    name: larry
  failureDeadline: 1m
```

要把截止时间设为无限，可以把它设置为 0（不建议）。

要等待快照完成，请执行：

```bash
kubectl wait vmsnapshot snap-larry --for condition=Ready
```

可以通过检查 `snapshot.status` 中的以下字段来验证快照的状态和组成：

- `readyToUse` - 表示是否可以从该快照恢复。有可能在快照完成后，volumeSnapshot 发生了某些情况（例如有人不小心删除了它），使该快照无法恢复。
- `includedVolumes/excludedVolumes` - 快照中被包含或被排除的卷的列表，用户应当验证所有预期的卷都在其中。
- `indications` - 表示快照是如何创建的指示数组：<br>
    - `Online` 表示快照是在 VM 运行时创建的。
    - `GuestAgent` 表示 QEMU 客户机代理处于活动状态，并成功冻结（静默）了客户机文件系统以进行在线快照。这会提供应用一致性快照，确保较高的数据完整性，如同应用在快照之前已优雅关闭。
    - `NoGuestAgent` 表示未安装 QEMU 客户机代理，或者在在线快照期间它尚未准备好静默文件系统。这会产生崩溃一致性快照，所捕获的 VM 状态如同突然断电。无法保证应用一致性，关键应用存在出现数据问题的风险。为了获得更好的可靠性，强烈建议安装并运行客户机代理，或重试快照。
    - `QuiesceFailed` 表示在在线快照过程中尝试冻结文件系统失败。尽管快照已完成，它并不必然具有应用一致性。通常建议重试快照，以获得适当的一致性。

## 恢复 VirtualMachine {#restoring-a-virtualmachine}

要恢复 `VirtualMachine` `larry`，来源是 `VirtualMachineSnapshot` `snap-larry`，请停止该 VM，等待它停止，然后应用以下 yaml。

```yaml
apiVersion: snapshot.kubevirt.io/v1beta1
kind: VirtualMachineRestore
metadata:
  name: restore-larry
spec:
  target:
    apiGroup: kubevirt.io
    kind: VirtualMachine
    name: larry
  virtualMachineSnapshotName: snap-larry
```

要等待恢复完成，请执行：

```bash
kubectl wait vmrestore restore-larry --for condition=Ready
```

## 目标就绪策略 {#target-readiness-policies}

在已经存在的目标（例如快照的父 VM）上恢复 `VirtualMachineSnapshot` 时，可以指定就绪策略，以调整目标 VM 未就绪时恢复如何进行。当目标 VM 完全停止时，它被视为就绪。通过把 `targetReadinessPolicy` 设置为可用类型之一来控制该策略。

可用策略如下：

- **WaitGracePeriod**（默认策略）：等待 5 分钟，让目标 VM 就绪。如果未能及时就绪，恢复将失败。
- **StopTarget**：停止目标 VM，以便恢复可以立即继续。
- **FailImmediate**：在尝试恢复之前不等待目标就绪。如果它未就绪，恢复会立即失败。
- **WaitEventually**：KubeVirt 会保留 `VirtualMachineRestore`，直到目标就绪。一旦目标就绪，就会启动恢复。

## 卷恢复策略 {#volume-restore-policies}

恢复 `VirtualMachineSnapshot` 时，其中一个步骤是恢复该快照的卷。`volumeRestorePolicy` 设置为用户提供了对该恢复过程如何进行的一定控制。

可用策略如下：

- **RandomizeNames**（默认策略）：已恢复 PVC 的名称使用恢复作业的 UID 随机生成。此策略保证不会发生冲突，但在使用 GitOps 时可能会成为问题，因为这些 ID 不可预测。
- **InPlace**：此策略用已恢复的卷覆盖原始卷。如果源 PVC 仍然存在，它们会在**创建已恢复的 PVC 之前被删除**。当在原始 VM 之上进行恢复、同时又希望保持相同的卷名称时，这很有用。
- **PrefixTargetName**：已恢复 PVC 的名称通过在卷名前加上目标 VM 名称来生成：`{targetVMName}-{volumeName}`。这样可以提供可预测的确定性命名，同时在把同一快照恢复到不同目标 VM 时避免冲突。名称会被截断为 63 个字符，以符合 DNS 标签长度限制。

## 卷所有权策略 {#volume-ownership-policies}

恢复 `VirtualMachineSnapshot` 时，KubeVirt 会尽其所能为所有已恢复的卷找到所有者。

默认策略是：如果找不到其他所有者（例如父 DataVolume），就把已恢复卷的所有权交给该 VM。
用户可能不希望这种行为，因为删除已恢复的 VM 也会把删除传播到已恢复的卷。

`.volumeOwnershipPolicy` 字段让用户能够控制已恢复卷的所有权。

策略 **VM**（默认策略）具有以下行为：

- 作为 VM 的 `.dataVolumeTemplate` 的一部分恢复的卷，由 `DataVolumes`（属于 `dataVolumeTemplate`）拥有
- 作为 `DataVolumes` 的卷会被恢复为 PVC，并由该 VM 拥有（因为原来的 DataVolume 不会被保留）
- 以 `VolumeRestorePolicy` **InPlace** 作为 `DataVolumes` 的一部分恢复的卷会被恢复为 PVC，并由该 DataVolume 拥有（因为原来的 DataVolume 会重新绑定到该 PVC）
- 作为 PVC 的卷会被恢复为 PVC，并由该 VM 拥有

**注意**：此行为会被一个 [缺陷](https://github.com/kubevirt/kubevirt/issues/12785) 改变，该策略仅在 VM 已经存在、但卷已被删除时才能正确应用。

策略 **None** 具有以下行为：

- 作为 VM 的 `.dataVolumeTemplate` 的一部分恢复的卷，由 `DataVolumes`（属于 `dataVolumeTemplate`）拥有
- 作为 `DataVolumes` 的卷会被恢复为 PVC，不被任何对象拥有
- 以 `VolumeRestorePolicy` **InPlace** 作为 `DataVolumes` 的一部分恢复的卷会被恢复为 PVC，并由该 DataVolume 拥有（因为原来的 DataVolume 会重新绑定到该 PVC）
- 作为 PVC 的卷会被恢复为 PVC，不被任何对象拥有

## 卷恢复覆盖 {#volume-restore-overrides}

恢复 `VirtualMachineSnapshot` 时，用户可能希望在卷被恢复时修改它们。可以使用 `volumeRestoreOverrides` 做到这一点。覆盖以列表形式提供，每个元素针对快照中的一个卷并对其应用变更。

卷恢复覆盖的语法如下：

```yaml
volumeRestoreOverrides:
  - volumeName: "original-volume" # Required - the original name of the volume within the snapshot
    # Optional - the new name of the restored volume
    # If combined with volumeRestorePolicy "InPlace", the PVC that has that name will be deleted before the volume is restored
    restoreName:
    # Optional - used to add or modify labels 
    labels:
      newLabel: "value"
      oldLabel: "newValue"
    # Optional - used to add or modify annotations
    annotations:
      newAnnotation: "value"
      oldAnnotation: "newValue"
```

## 以对 GitOps 友好的方式恢复 {#restoring-in-a-gitops-friendly-way}

除非做出一些调整，否则在 KubeVirt 中恢复 VM 可能无法完全兼容 GitOps 工作流。

“原地”恢复 VM（即用快照的内容覆盖它）时的问题是，原始 VM 会被新信息修改。例如，任何以 `DataVolume` 挂载的卷都会被 PVC 替换，并且所有卷的名称都会被改为随机生成的 UID。

这意味着 **ArgoCD** 之类的 GitOps 工具会把原始 VM 报告为*不同步*，并提示用户重新同步它。如果该 VM 被重新同步，已恢复的卷最终会处于未挂载状态，VM 会回到原来的状态，这通常不是预期效果。

要在恢复 VM 的同时不造成 Git 中的内容与集群之间失步，建议对 `VirtualMachineRestores` 应用以下变更：

- 使用 `volumeRestorePolicy` **InPlace**，使卷具有与原始卷相同的可预测名称，或者使用 **PrefixTargetName**，以基于目标 VM 名称获得可预测的名称
- 如果希望更改已恢复卷的名称，请使用 `volumeRestoreOverrides`
- 避免在 VM 中使用 `dataVolumeTemplates`，因为在创建已恢复的 DataVolume 时它们会被编辑（如果未使用 `volumeRestorePolicy` InPlace）
- 如果使用 `DataVolumes`，请把它们作为 `PersistentVolumeClaims` 挂载（DataVolume 会创建与自身同名的 PVC，使两种挂载类型相互兼容）
 
```yaml
# This DataVolume can be mounted on the VM as a persistentVolumeClaim or as a dataVolume.
# It doesn't really change anything, as both mount types are compatible, but one is GitOps safe and the other is not.

---
apiVersion: cdi.kubevirt.io/v1beta1
kind: DataVolume
metadata:
  name: volume
spec:
  ...

---
apiVersion: kubevirt.io/v1
kind: VirtualMachine
metadata:
  name: example
spec:
  template:
    spec:
      ...
      volumes:
      # This is not going to be overwritten by the VMRestore if volumeRestorePolicy is InPlace. It is GitOps-safe.
      - name: volume 
        persistentVolumeClaim:
          claimName: datavol
      # This will be transformed into the same volume mount as the one above, using a persistentVolumeName.
      # It will cause a desynchronization between the Git and the cluster. Avoid using this syntax.
      - name: volume
        dataVolume:
          name: datavol
```

## 清理 {#cleanup}

只要你可能希望再次从它们恢复，就保留 `VirtualMachineSnapshots`（以及与之对应的 `VirtualMachineSnapshotContents`）。

恢复完成后就不再需要 `restore-larry`，可以随意删除它。
