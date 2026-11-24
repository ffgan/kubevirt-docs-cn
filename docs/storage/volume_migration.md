# 更新卷策略与卷迁移 {#update-volume-strategy-and-volume-migration}

在 VM 运行期间，可以通过更新卷策略进行存储迁移。当用户需要更换底层存储时，存储迁移会很有用，例如存储类已被弃用，或者有新的、性能更好的驱动可用。

此功能不处理卷的创建，也不涵盖存储类之间的迁移，而是实现一个基本 API，上层工具可以用它来执行更高级的迁移规划。

如果把 `Migration` 指定为 `updateVolumesStrategy`，当 VirtualMachine spec 被更新时，KubeVirt 会尝试把存储从旧卷集合迁移到新卷集合。迁移会考虑单次更新中出现的已变更卷。单次更新可以包含对多个卷的修改，但对卷集合的连续变更将作为各自独立的迁移来处理。

更新是声明式的，并且与 [GitOps](https://kubevirt.io/user-guide/cluster_admin/gitops/) 兼容。例如，带有新卷集合和迁移卷更新策略的新版本 VM 规范，可以直接使用 `kubectl apply` 应用，或者使用 `kubectl edit` 交互式地编辑该 VM

示例：
带有 datavolume 和 datavolume 模板的原始 VM：
```yaml
apiVersion: kubevirt.io/v1
kind: VirtualMachine
metadata:
  labels:
    kubevirt.io/vm: vm-dv
  name: vm-dv
spec:
  dataVolumeTemplates:
  - metadata:
      creationTimestamp: null
      name: src-dv
    spec:
      storage:
        accessModes:
        - ReadWriteOnce
        volumeMode: Filesystem
        resources:
          requests:
            storage: 2Gi
        storageClassName: local
      source:
        registry:
          url: docker://registry:5000/kubevirt/alpine-container-disk-demo:devel
  runStrategy: "Always"
  template:
    metadata:
      labels:
        kubevirt.io/vm: vm-dv
    spec:
      domain:
        devices:
          disks:
          - disk:
              bus: virtio
            name: datavolumedisk1
          interfaces:
          - masquerade: {}
            name: default
        resources:
          requests:
            memory: 128Mi
      terminationGracePeriodSeconds: 0
      volumes:
      - dataVolume:
          name: src-dv
        name: datavolumedisk1
      networks:
      - name: default
        pod: {}
```

datavolume `src-dv` 被迁移到 pvc `dest-pvc`：
```yaml
apiVersion: cdi.kubevirt.io/v1beta1
kind: DataVolume
metadata:
  name: dst-dv
spec:
  source:
      blank: {}
  storage:
    accessModes:
      - ReadWriteMany
    volumeMode: Block
    resources:
      requests:
        storage: 5Gi
    storageClassName: rook-ceph
```

通过更新该 VM：
```diff
 apiVersion: kubevirt.io/v1
 kind: VirtualMachine
     kubevirt.io/vm: vm-dv
   name: vm-dv
 spec:
+  updateVolumesStrategy: Migration
   dataVolumeTemplates:
   - metadata:
-      name: src-pvc
+      name: dst-dv

       volumes:
       - dataVolume:
-          name: src-pvc
+          name: dst-dv
         name: datavolumedisk1
```

### 使用 PVC 作为目标卷 {#using-a-pvc-as-the-target-volume}

未通过 DataVolume 创建的普通 PVC 完全支持卷迁移。在这种情况下，volumes 部分使用 `persistentVolumeClaim` 而不是 `dataVolume` 来引用该 PVC。

首先，创建目标 PVC：
```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: dst-pvc
spec:
  accessModes:
    - ReadWriteMany
  volumeMode: Block
  resources:
    requests:
      storage: 5Gi
  storageClassName: rook-ceph
```

然后更新该 VM，以把源 DataVolume 迁移到目标 PVC：
```diff
 apiVersion: kubevirt.io/v1
 kind: VirtualMachine
     kubevirt.io/vm: vm-dv
   name: vm-dv
 spec:
+  updateVolumesStrategy: Migration
-  dataVolumeTemplates:
-  - metadata:
-      name: src-dv
-    spec:
-      ...

       volumes:
-      - dataVolume:
-          name: src-dv
+      - persistentVolumeClaim:
+          claimName: dst-pvc
         name: datavolumedisk1
```

目标卷的类型或大小可以与源不同。既可以迁移到块卷，也可以迁移到文件系统卷，并且可以从块卷或文件系统卷迁出。
目标卷应当等于或大于源卷。然而，目标卷多出来的大小差异不会立即在 VM 内部可见，并且必须手动调整大小，因为客户机并不知道发生了迁移。

卷迁移依赖于 `LiveMigrate` workloadUpdateStrategy。要完全启用此功能，请把以下内容添加到 KubeVirt CR：
```yaml
apiVersion: kubevirt.io/v1
kind: KubeVirt
spec:
  configuration:
    vmRolloutStrategy: LiveUpdate
  workloadUpdateStrategy:
    workloadUpdateMethods:
    - LiveMigrate
```

可以通过观察带有标签 `kubevirt.io/volume-update-in-progress: <vm-name>` 的相应 VirtualMachineInstanceMigration 对象来监控卷迁移进度。示例：
```bash
$ kubectl get virtualmachineinstancemigrations -l kubevirt.io/volume-update-in-progress: vmi` --watch=true
NAME                           PHASE       VMI
kubevirt-workload-update-abcd  Running     vmi
```

## Datavolume 模板 {#datavolume-template}

更新被 datavolume 模板引用的 datavolume 需要特别小心。volumes 部分必须包含对 datavolume 模板名称的引用。这意味着 datavolume 模板必须被整体删除，或者也一并更新。

第一个示例中原始 VM 的 datavolume 更新示例：
```yaml
apiVersion: cdi.kubevirt.io/v1beta1
kind: DataVolume
metadata:
  name: dst-dv
spec:
  source:
      blank: {}
  storage:
    accessModes:
      - ReadWriteMany
    volumeMode: Block
    resources:
      requests:
        storage: 10Gi
    storageClassName: rook-ceph-block
---
apiVersion: kubevirt.io/v1
kind: VirtualMachine
metadata:
  labels:
    kubevirt.io/vm: vm-dv
  name: vm-dv
spec:
  updateVolumesStrategy: Migration
  dataVolumeTemplates:
  - metadata:
      name: dst-dv
    spec:
      storage:
        accessModes:
        - ReadWriteOnce
        volumeMode: Filesystem
        resources:
          requests:
            storage: 2Gi
        storageClassName: local
      source:
        registry:
          url: docker://registry:5000/kubevirt/alpine-container-disk-demo:devel
  runStrategy: "Always"
  template:
    metadata:
      labels:
        kubevirt.io/vm: vm-dv
    spec:
      domain:
        devices:
          disks:
          - disk:
              bus: virtio
            name: datavolumedisk1
          interfaces:
          - masquerade: {}
            name: default
        resources:
          requests:
            memory: 128Mi
      terminationGracePeriodSeconds: 0
      volumes:
      - dataVolume:
          name: dst-dv
        name: datavolumedisk1
      networks:
      - name: default
        pod: {}
```

## 卷迁移取消 {#volume-migration-cancellation}

用户可能出于各种原因希望停止并取消正在进行的卷迁移。
迁移取消也是以声明方式处理的，因此用户必须恢复先前的卷集合，这将被读取为对卷更新和迁移的取消。

一种可能的取消迁移方式是应用 VM 声明的旧版本。

## 卷迁移失败与恢复 {#volume-migration-failure-and-recovery}

如果卷迁移失败，KubeVirt 会继续尝试迁移，直到卷更新被取消。因此，如果最初的失败是暂时的，后续的迁移可能会成功；否则，KubeVirt 会继续以指数退避时间生成新的迁移。

要从持续的失败中恢复，用户必须把卷集合恢复到原来的状态，以表示取消卷迁移。

### 需要手动恢复 {#manual-recovery-required}

如果 VMI 因任何原因消失，则不再重试卷迁移。如果用户无意中关闭了 VM，或者 VMI 被意外删除，就可能发生这种情况。

然而，在这些情况下，VM spec 处于不一致状态，因为卷集合包含目标卷，但复制并未成功，用户可能无法正确启动该 VM。因此，该 VM 会被标记为条件 `ManualRecoveryRequired`，KubeVirt 将拒绝启动具有此条件的 VM。

为了恢复 VM spec，用户需要恢复 VM spec 中的卷集合，就像卷迁移取消时那样。

卷迁移信息也存储在 VM status 中，用户可以看到已迁移卷的完整列表，其中包含源名称和目标名称以及对应的卷名称。

用户可以在 VM status 中找到已迁移卷的完整列表，其中包括源名称和目标名称以及关联的卷名称。

## 限制 {#limitations}

只有某些类型的磁盘和卷支持被迁移。对于无效的卷类型，会设置 RestartRequired 条件，卷将在 VM 重启时被替换。
目前，卷迁移支持在 PersistentVolumeClaim 与 Datavolume 之间进行。
此外，如果磁盘属于以下情况，则禁止卷迁移：

* shareable，因为它无法保证多个写入者情况下的数据一致性
* filesystem，因为 virtiofs 目前不支持实时迁移
* lun，原先磁盘可能支持 SCSI 协议，但目标 PVC 类不支持。这种情况目前不受支持。

目前，KubeVirt 只支持在不同节点之间进行实时迁移。卷迁移依赖于实时迁移；因此，在同一节点上对存储进行实时迁移同样不可能。卷迁移可以在本地存储之间进行，例如在两个访问模式为 RWO 的 PVC 之间，但它们需要位于两台不同的宿主机上。

## 使用迁移策略取消卷更新 {#cancellation-of-the-volume-update-with-the-migration-strategy}

在卷迁移期间，用户可以通过恢复旧的卷集合来取消更新。例如，使用原始的 VM 定义。

如果卷集合与原始集合并不完全匹配，该更新将被视为一次新的修改，当前迁移不会被取消。

## 失败 {#failures}

卷迁移可能因各种原因失败。迁移会以递增的超时时间执行多次，以便有可能解决暂时性错误，例如连接问题。

如果失败持续存在，并且用户希望停止重试，他们必须取消卷更新。

### 手动恢复 {#manual-recovery}

如果迁移期间出现问题并且 VM 不再运行，会对该 VM 应用 'ManualRecoveryRequired' 条件。例如，如果 VM 在迁移期间被错误地关闭，或者 VMI 消失了。

```yaml
status:
  conditions:
  [..]
  - lastProbeTime: null
    lastTransitionTime: null
    reason: VMI was removed during the volume migration
    status: "True"
    type: ManualRecoveryRequired
```


当此条件存在时，它会禁止启动该 VM，因为更新期间复制未成功，导致 VM 卷不一致。

修改后的卷仍保留在 VM 定义中，用户必须在启动 VM 之前通过取消更新来手动恢复原始卷，并最终重试该更新。

#### 目标卷的恢复 {#destination-volumes-restoration}

迁移期间可能出现一种特殊的竞态条件，在取消更新之前需要仔细验证。

在某些情况下，复制可能已经完成，但迁移被标记为失败。这种情况发生在：

  1. VirtualMachineInstanceMigration 处于 `Failed` 状态：
```sh
$ kubectl get vmims -l kubevirt.io/volume-update-migration=vm
NAME                             PHASE    VMI
kubevirt-workload-update-qzfb5   Failed   vm
```
  2. Libvirt 报告目标 virt-launcher Pod 中的域从暂停变为运行，并且 KubeVirt 设置了域在目标上启动的时间：
```bash
$ kubectl get vmi vmi-migratable -ojsonpath='{.status.migrationState.targetNodeDomainReadyTimestamp}'
2025-01-14T08:29:07Z
```
如果满足上述条件，则用户必须保留目标卷，并从 VM 中移除 `ManualRecoveryCondition`，方法是修补该 VM 并移除 `updateVolumeStrategy` 字段。这样一来，该变更将被解释为卷的替换。
```bash
$ kubectl patch vm vm-dv --type='json' -p='[{"op": "remove", "path": "/spec/updateVolumesStrategy"}]'
```
否则，用户需要恢复源卷，并最终重试迁移。
