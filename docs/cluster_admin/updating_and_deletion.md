# 更新与删除 {#updating-and-deletion}

## 更新 KubeVirt 控制平面 {#updating-kubevirt-control-plane}

从 `v0.17.0` 版本起支持零停机滚动更新。不支持从
KubeVirt `v0.17.0` 之前的任何版本进行更新。

!!! note "支持的升级路径"
    仅支持从 N-1 版本更新到 N 版本：

    **在次要版本之间升级：**

    - 从一个次要版本的最新补丁版本升级到下一个次要版本的最新补丁版本
    - 示例：如果 1.7 的最新版本是 v1.7.6，并且 1.6 的最新版本是 v1.6.4
        - ✓ 支持从 v1.6.4 升级到 v1.7.6
        - ✗ 不支持从 v1.6.2 升级到 v1.7.6
        - ✗ 不支持从 v1.5.z 升级到 v1.7.6

    **在同一个次要版本内升级：**

    - 示例：如果 1.6 的最新版本是 v1.6.4
        - ✓ 支持从 v1.6.3 升级到 v1.6.4
        - ✗ 不支持从 v1.6.2 升级到 v1.6.4

更新可通过以下两种方式之一触发。

1.  通过更改 KubeVirt CR spec 中的 imageTag 值。

    例如，从 `v0.17.0-alpha.1` 更新到 `v0.17.0`，只需用 `imageTag: v0.17.0` 这一值修补 KubeVirt CR 即可。此后
    KubeVirt operator 将开始滚动更新
    新版本的 KubeVirt。现有的 VM/VMI 在更新过程中以及更新成功之后
    都不会中断。

    ```
    $ kubectl patch kv kubevirt -n kubevirt --type=json -p '[{ "op": "add", "path": "/spec/imageTag", "value": "v0.17.0" }]'
    ```


2.  或者，如果没有设置 imageTag 值，则通过更新 kubevirt operator。

    当 kubevirt CR 中没有设置 imageTag 值时，系统假定
    KubeVirt 的版本锁定为 operator 的版本。
    这意味着更新 operator 也会一并更新底层的
    KubeVirt 安装。

    ```
    $ export RELEASE=v0.26.0
    $ kubectl apply --server-side -f https://github.com/kubevirt/kubevirt/releases/download/${RELEASE}/kubevirt-operator.yaml
    ```


第一种方式提供更细粒度的做法，你可以完全
控制所安装的 KubeVirt 版本，而与你可能正在运行的
KubeVirt operator 版本无关。第二种
做法允许你把 operator 和操作对象锁定到同一
版本。

较新的 KubeVirt 可能需要额外或扩展的 RBAC 规则。在这种
情况下，第 1 种更新方法可能会失败，因为集群中现有的 virt-operator
本身并不具备这些 RBAC 规则。在这种情况下，你
需要先更新 `virt-operator`，然后再继续更新
kubevirt。更多细节请参阅[此问题](https://github.com/kubevirt/kubevirt/issues/2533)。

## 更新 KubeVirt 工作负载 {#updating-kubevirt-workloads}

从 `v0.39.0` 起，工作负载更新作为可选启用的功能受到支持。

默认情况下，更新 KubeVirt 只涉及控制平面
组件。更新发生之前正在运行的任何现有 VirtualMachineInstance（VMI）工作负载
都保持 100% 不被改动。这些工作负载
会继续运行，不会作为默认更新过程的一部分被中断。

需要注意的是，这些 VMI 工作负载确实涉及 libvirt、qemu 和 virt-launcher 等组件，
在 KubeVirt 更新过程中也可以选择更新这些组件。不过，这需要选择启用，让
virt-operator 对工作负载执行自动化操作。

选择启用 VMI 更新，需要在 KubeVirt CR 上配置 `workloadUpdateStrategy`
字段。该字段控制 virt-operator 在更新 VMI 工作负载 Pod 时
将使用的方法。

支持两种方法。

**LiveMigrate：** 通过把虚拟机客户机实时迁移到已启用全部更新组件的新 Pod 中来更新 VMI。

**Evict：** 会关闭该 VMI 的 Pod。如果该 VMI 由设置了 `runStrategy: always` 的更高层级 VirtualMachine 对象控制，
则会在带有已更新组件的新 Pod 中拉起一个新的 VMI。

更新 VMI 工作负载时干扰最小的方式是使用 LiveMigrate。任何
不可实时迁移的 VMI 工作负载都会保持不动。如果集群中未启用实时
迁移，那么由 virt-operator 管理的 VMI 更新唯一可用的选项
就是 Evict 方法。


**示例：通过 LiveMigration 启用 VMI 工作负载更新**

```yaml
apiVersion: kubevirt.io/v1
kind: KubeVirt
metadata:
  name: kubevirt
  namespace: kubevirt
spec:
  imagePullPolicy: IfNotPresent
  workloadUpdateStrategy:
    workloadUpdateMethods:
      - LiveMigrate
```


**示例：通过 Evict 并配合批量调优启用 VMI 工作负载更新**

批量调优允许配置驱逐 VMI 的速度。在大型
集群中，最好确保分批驱逐 VMI，以便
分散负载。

```yaml
apiVersion: kubevirt.io/v1
kind: KubeVirt
metadata:
  name: kubevirt
  namespace: kubevirt
spec:
  imagePullPolicy: IfNotPresent
  workloadUpdateStrategy:
    workloadUpdateMethods:
      - Evict
    batchEvictionSize: 10
    batchEvictionInterval: "1m"
```



**示例：同时使用 LiveMigrate 和 Evict 启用 VMI 工作负载更新**

当同时指定 LiveMigrate 和 Evict 时，任何可实时迁移的工作负载
都保证会被实时迁移。只有不可
实时迁移的工作负载才会被驱逐。


```yaml
apiVersion: kubevirt.io/v1
kind: KubeVirt
metadata:
  name: kubevirt
  namespace: kubevirt
spec:
  imagePullPolicy: IfNotPresent
  workloadUpdateStrategy:
    workloadUpdateMethods:
      - LiveMigrate
      - Evict
    batchEvictionSize: 10
    batchEvictionInterval: "1m"
```


## 删除 KubeVirt {#deleting-kubevirt}

要删除 KubeVirt，应当先删除 `KubeVirt` 自定义
资源，然后再删除 KubeVirt operator。

    $ export RELEASE=v0.17.0
    $ kubectl delete -n kubevirt kubevirt kubevirt --wait=true # --wait=true should anyway be default
    $ kubectl delete apiservices v1.subresources.kubevirt.io # this needs to be deleted to avoid stuck terminating namespaces
    $ kubectl delete mutatingwebhookconfigurations virt-api-mutator # not blocking but would be left over
    $ kubectl delete validatingwebhookconfigurations virt-operator-validator # not blocking but would be left over
    $ kubectl delete validatingwebhookconfigurations virt-api-validator # not blocking but would be left over
    $ kubectl delete -f https://github.com/kubevirt/kubevirt/releases/download/${RELEASE}/kubevirt-operator.yaml --wait=false


> 注意：如果误先删除了 operator，KV 自定义
> 资源会卡在 `Terminating` 状态。要修复它，请
> 手动从该资源中删除 finalizer。
>
> 注意：由于存在一个缺陷，需要手动
> 删除 `apiservice` 和 `webhookconfigurations`。
>
>     $ kubectl -n kubevirt patch kv kubevirt --type=json -p '[{ "op": "remove", "path": "/metadata/finalizers" }]'

