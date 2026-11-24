# KubeVirt 迁移控制器 {#kubevirt-migration-controller}

[KubeVirt 迁移控制器](https://github.com/kubevirt/kubevirt-migration-controller)项目提供了一组 API，用于协助批量存储实时迁移操作。这使你能够把多个虚拟机磁盘实时迁移到一个共同的目标存储类。

## 先决条件 {#prerequisites}
要使用持久磁盘，KubeVirt 环境必须已安装[容器化数据导入器（CDI）](../storage/containerized_data_importer.md)。

## 安装 {#installation}

使用 operator 安装最新发行版

```shell
for YAML in $(curl -s https://api.github.com/repos/kubevirt/kubevirt-migration-operator/releases/latest | grep browser_download_url | grep -oE 'https://[^"]+\.yaml'); do kubectl create -f $YAML; done
```


## 在存储 A 上创建带磁盘的 VirtualMachine {#create-a-virtualmachine-with-disk-on-storage-a}

执行以下命令，创建一对 VirtualMachine 和 DataVolume：

```shell
cat <<EOF | kubectl apply -f -
apiVersion: cdi.kubevirt.io/v1beta1
kind: DataVolume
metadata:
  name: simple-dv
spec:
  source:
      registry:
        url: docker://quay.io/kubevirt/fedora-with-test-tooling-container-disk:v1.7.0
  storage:
    storageClassName: storageclass-a
    resources:
      requests:
        storage: "10Gi"
---
apiVersion: kubevirt.io/v1
kind: VirtualMachine
metadata:
  name: simple-vm
spec:
  runStrategy: Always
  template:
    metadata:
      labels: {kubevirt.io/domain: simple-vm,
        kubevirt.io/vm: simple-vm}
    spec:
      domain:
        devices:
          disks:
          - disk: {bus: virtio}
            name: dv-disk
          - disk: {bus: virtio}
            name: cloudinitdisk
        resources:
          requests: {memory: 2048M}
      volumes:
      - dataVolume: {name: simple-dv}
        name: dv-disk
      - cloudInitNoCloud:
          userData: |
            #cloud-config
            password: fedora
            chpasswd: { expire: False }
        name: cloudinitdisk
EOF
```

## 创建迁移计划 {#create-a-migration-plan}

```shell
cat <<EOF | kubectl apply -f -
apiVersion: migrations.kubevirt.io/v1alpha1
kind: VirtualMachineStorageMigrationPlan
metadata:
  name: test-plan
spec:
  virtualMachines:
  - name: simple-vm
    targetMigrationPVCs:
    - volumeName: dv-disk
      destinationPVC:
        name: test-pvc
        storageClassName: storageclass-b
EOF
```

## 启动迁移 {#start-the-migration}

```shell
cat <<EOF | kubectl apply -f -
apiVersion: migrations.kubevirt.io/v1alpha1
kind: VirtualMachineStorageMigration
metadata:
  name: test-migration
spec:
  virtualMachineStorageMigrationPlanRef:
    name: test-plan
EOF
```

观察完成情况
```shell
$ kubectl get virtualmachinestoragemigration
NAME             PLAN        PHASE       AGE
test-migration   test-plan   Completed   50s
```

## 多命名空间 {#multi-namespace}

也可以使用 `MultiNamespaceVirtualMachineStorageMigrationPlan/MultiNamespaceVirtualMachineStorageMigration` 针对多个命名空间发起单个计划/迁移。  
前者嵌入单命名空间的 spec，并在指定的命名空间中创建它。  
示例见此处：

```yaml
apiVersion: migrations.kubevirt.io/v1alpha1
kind: MultiNamespaceVirtualMachineStorageMigrationPlan
metadata:
  name: multi-ns-storage-migration-plan
spec:
  namespaces:
    - name: production
      virtualMachines:
        - name: prod-vm-1
          targetMigrationPVCs:
            - volumeName: rootdisk
              destinationPVC:
                storageClassName: fast-ssd
    - name: development
      virtualMachines:
        - name: dev-vm-1
          targetMigrationPVCs:
            - volumeName: rootdisk
              destinationPVC:
                storageClassName: standard
```

同样，要启动它

```yaml
apiVersion: migrations.kubevirt.io/v1alpha1
kind: MultiNamespaceVirtualMachineStorageMigration
metadata:
  name: multi-ns-storage-migration
spec:
  multiNamespaceVirtualMachineStorageMigrationPlanRef:
    name: multi-ns-storage-migration-plan
```

## 取消迁移 {#canceling-a-migration}

要取消正在进行的迁移，请删除 `VirtualMachineStorageMigration` 资源：

```shell
kubectl delete virtualmachinestoragemigration test-migration
```

控制器会使迁移经过 `Canceling` 阶段，自动把 VM 卷恢复为原来的源 PVC，并清理迁移期间创建的任何目标 DataVolume。完成后，迁移进入 `Canceled` 阶段，该资源会被删除。

```shell
$ kubectl get virtualmachinestoragemigration
NAME             PLAN        PHASE      AGE
test-migration   test-plan   Canceled   30s
```

对于多命名空间迁移，请删除 `MultiNamespaceVirtualMachineStorageMigration` 资源 — 每个子迁移都会以相同方式被取消。

> **Note:** 这与 KubeVirt 内置的[卷迁移](volume_migration.md#volume-migration-cancellation)不同。在那里，取消需要应用旧的 VM spec 以恢复卷集合。使用迁移控制器时，删除迁移 CR 就足够了 — 控制器会自动处理回退。
