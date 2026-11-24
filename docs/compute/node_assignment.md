# 节点分配 {#node-assignment}

你可以将 VM 约束为仅在特定节点上运行，或者优先在特定节点上运行：

-   **nodeSelector**
-   **亲和性与反亲和性**
-   **污点与容忍**


## nodeSelector {#nodeselector}

设置 `spec.nodeSelector` 要求会约束调度器，使其仅将 VM 调度到包含指定标签的节点上。在以下示例中，该 VMI 包含标签 `cpu: slow` 和 `storage: fast`：

```yaml
metadata:
  name: testvmi-ephemeral
apiVersion: kubevirt.io/v1
kind: VirtualMachineInstance
spec:
  nodeSelector:
    cpu: slow
    storage: fast
  domain:
    resources:
      requests:
        memory: 64M
    devices:
      disks:
      - name: mypvcdisk
        lun: {}
  volumes:
    - name: mypvcdisk
      persistentVolumeClaim:
        claimName: mypvc
```

因此，调度器只会将该 VMI 调度到元数据中包含这些标签的节点。它的工作方式与 Pod 的 `nodeSelector` 完全相同。更多示例请参见 [Pod nodeSelector 文档](https://kubernetes.io/docs/concepts/configuration/assign-pod-node/#nodeselector)。


## 亲和性与反亲和性 {#affinity-and-anti-affinity}

`spec.affinity` 字段允许为 VM 指定硬亲和性与软亲和性。可以针对工作负载（VM 和 Pod）以及节点编写匹配规则。由于 VM 是基于 Pod 的一种工作负载类型，Pod 亲和性同样会影响 VM。

`podAffinity` 和 `podAntiAffinity` 的示例可能如下所示：

```yaml
metadata:
  name: testvmi-ephemeral
apiVersion: kubevirt.io/v1
kind: VirtualMachineInstance
spec:
  nodeSelector:
    cpu: slow
    storage: fast
  domain:
    resources:
      requests:
        memory: 64M
    devices:
      disks:
      - name: mypvcdisk
        lun: {}
  affinity:
    podAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
      - labelSelector:
          matchExpressions:
          - key: security
            operator: In
            values:
            - S1
        topologyKey: failure-domain.beta.kubernetes.io/zone
    podAntiAffinity:
      preferredDuringSchedulingIgnoredDuringExecution:
      - weight: 100
        podAffinityTerm:
          labelSelector:
            matchExpressions:
            - key: security
              operator: In
              values:
              - S2
          topologyKey: kubernetes.io/hostname
  volumes:
    - name: mypvcdisk
      persistentVolumeClaim:
        claimName: mypvc
```

亲和性与反亲和性的工作方式与 Pod 的 `affinity` 完全相同。这包括 `podAffinity`、`podAntiAffinity`、`nodeAffinity` 和 `nodeAntiAffinity`。更多示例和细节请参见 [Pod 亲和性与反亲和性文档](https://kubernetes.io/docs/concepts/configuration/assign-pod-node/#affinity-and-anti-affinity)。


## 污点与容忍 {#taints-and-tolerations}

如上所述，亲和性是 VM 的一种属性，它将 VM 吸引到一组节点（作为偏好或硬性要求）。污点则相反——它们允许节点排斥一组 VM。

污点与容忍协同工作，以确保 VM 不会被调度到不合适的节点上。一个或多个污点会应用到节点上；这表示该节点不应接受任何不容忍这些污点的 VM。容忍应用到 VM 上，允许（但不要求）VM 调度到具有匹配污点的节点上。

你可以使用 kubectl taint 为节点添加污点。例如，

    kubectl taint nodes node1 key=value:NoSchedule

`tolerations` 的示例可能如下所示：

```yaml
metadata:
  name: testvmi-ephemeral
apiVersion: kubevirt.io/v1
kind: VirtualMachineInstance
spec:
  nodeSelector:
    cpu: slow
    storage: fast
  domain:
    resources:
      requests:
        memory: 64M
    devices:
      disks:
      - name: mypvcdisk
        lun: {}
  tolerations:
  - key: "key"
    operator: "Equal"
    value: "value"
    effect: "NoSchedule"
```
## 使用 Descheduler 进行节点均衡 {#node-balancing-with-descheduler}

在某些情况下，我们可能需要根据当前的调度策略和负载状况重新均衡集群。[Descheduler](https://github.com/kubernetes-sigs/descheduler) 可以找到违反调度决策等情况的 Pod，并根据 descheduler 策略驱逐它们。KubeVirt VM 被当作带有本地存储的 Pod 处理，因此默认情况下，descheduler 不会驱逐它们。但可以通过向 VM 中的 VMI 模板添加特殊注解来轻松覆盖这一行为：

```console
spec:
  template:
    metadata:
      annotations:
        descheduler.alpha.kubernetes.io/evict: true
```

此注解会使 descheduler 能够驱逐该 VM 的 Pod，随后调度器可以将它调度到不同的节点上。在当前 VirtualMachineInstance 实例从集群中删除之前，VirtualMachine 绝不会重启或重新创建 VirtualMachineInstance。

## 实时更新 {#live-update}

当 [VM 滚动更新策略](../user_workloads/vm_rollout_strategies.md) 设置为 `LiveUpdate` 时，对 VM 的节点选择器或亲和性的更改将动态传播到 VMI（除非设置了 `RestartRequired` 条件）。对容忍的更改不会动态传播，如果在正在运行的 VM 上更改，则会触发 `RestartRequired` 条件。

对节点选择器/亲和性的修改只会在下一次[迁移](live_migration.md)时生效，仅此更改本身不会触发迁移。
