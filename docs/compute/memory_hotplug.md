# 内存热插拔 {#memory-hotplug}

内存热插拔在 KubeVirt 1.1 版本中引入，使正在运行的 VM 可用的内存量能够动态调整大小。

## 限制 {#limitations}
* 内存热插拔目前仅在 x86_64 架构上受支持。
* 运行至少 Linux v5.8 的 Linux 客户机得到完全支持。
* Windows 客户机支持已添加到 virtio-win，但应视为不稳定。
* 当前的热插拔实现涉及 VM 工作负载的实时迁移。
* VirtualMachine 必须至少拥有 1GiB 内存才能支持内存热插拔。


# 配置 {#configuration}

### 配置工作负载更新策略 {#configure-the-workload-update-strategy}

在 KubeVirt CR 中配置 `LiveMigrate` 作为 `workloadUpdateStrategy`，因为当前热插拔流程的实现要求 VM 进行实时迁移。

```yaml
apiVersion: kubevirt.io/v1
kind: KubeVirt
spec:
  workloadUpdateStrategy:
    workloadUpdateMethods:
    - LiveMigrate
```

### 配置 VM 滚动更新策略 {#configure-the-vm-rollout-strategy}

最后，将 VM 滚动更新策略设置为 `LiveUpdate`，这样对 VM 对象所做的更改才会传播到 VMI 而无需重启。
这也在 KubeVirt CR 配置中完成：

```yaml
apiVersion: kubevirt.io/v1
kind: KubeVirt
spec:
  configuration:
    vmRolloutStrategy: "LiveUpdate"
```

**注意：** 如果在已经运行的 VM 上启用或禁用内存热插拔，则必须重启才能使更改生效。

更多信息可以在 [VM 滚动更新策略](../user_workloads/vm_rollout_strategies.md) 页面找到。

### [可选] 设置集群范围的最大内存量 {#optional-set-a-cluster-wide-maximum-amount-of-memory}

你可以使用 KubeVirt CR 中的集群级别设置，来设置客户机的最大内存量。

```yaml
apiVersion: kubevirt.io/v1
kind: KubeVirt
spec:
  configuration:
    liveUpdateConfiguration:
      maxGuest: 8Gi
```

VM 级别的配置将优先于集群范围的配置。

## 内存热插拔实践 {#memory-hotplug-in-action}

首先，我们在 KubeVirt CR 中将滚动更新策略设置为 `LiveUpdate`，并将 `LiveMigrate` 作为 `workloadUpdateStrategy`。

```sh
$ kubectl --namespace kubevirt patch kv kubevirt -p='[{"op": "add", "path": "/spec/configuration/vmRolloutStrategy", "value": "LiveUpdate"}]' --type='json'
$ kubectl --namespace kubevirt patch kv kubevirt -p='[{"op": "add", "path": "/spec/workloadUpdateStrategy/workloadUpdateMethods", "value": ["LiveMigrate"]}]' --type='json'
```

现在我们创建一个启用了内存热插拔的 VM。

```yaml
apiVersion: kubevirt.io/v1
kind: VirtualMachine
metadata:
  name: vm-alpine
spec:
  runStrategy: Always
  template:
    spec:
      domain:
        memory:
          guest: 1Gi
        devices:
          interfaces:
            - masquerade: {}
              model: virtio
              name: default
          disks:
          - disk:
              bus: virtio
            name: containerdisk
      networks:
        - name: default
          pod: {}
      volumes:
      - containerDisk:
          image: registry:5000/kubevirt/alpine-container-disk-demo:devel
        name: containerdisk
```

虚拟机将自动启动，启动完成后，它会在 VMI 内的 `status.memory` 字段中报告当前可供客户机使用的内存。

```sh
$ kubectl get vmi vm-cirros -o json | jq .status.memory
```
```json
{
  "guestAtBoot": "1Gi",
  "guestCurrent": "1Gi",
  "guestRequested": "1Gi"
}
```

由于虚拟机现在正在运行，我们可以修补 VM 对象，将可用的客户机内存加倍，从而从 1Gi 增加到 2Gi。

```sh
$ kubectl patch vm vm-cirros -p='[{"op": "replace", "path": "/spec/template/spec/domain/memory/guest", "value": "2Gi"}]' --type='json'
```

热插拔请求处理完成且虚拟机完成实时迁移后，新的内存量应可供客户机使用，并在 VMI 对象中可见。

```sh
$ kubectl get vmi vm-cirros -o json | jq .status.memory
```
```json
{
  "guestAtBoot": "1Gi",
  "guestCurrent": "2Gi",
  "guestRequested": "2Gi"
}
```
