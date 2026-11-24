# KSM 管理 {#ksm-management}

内核同页合并（[KSM](http://www.linux-kvm.org/page/KSM)）
允许对内存去重。KSM 尝试查找相同的内存页并将其合并，以释放内存。
>更多信息：  
>- [KSM（内核同页合并）功能](https://www.thomas-krenn.com/en/wiki/KSM_(Kernel_Samepage_Merging)_feature)  
>- [内核同页合并（KSM）](https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux/7/html/virtualization_tuning_and_optimization_guide/chap-ksm)

## 通过 KubeVirt CR 启用 KSM {#enabling-ksm-through-kubevirt-cr}
可以通过 KubeVirt CR 中的 `spec.configuration.ksmConfiguration` 在节点上启用 KSM。  
`ksmConfiguration` 指示在哪些节点上启用 KSM，并公开一个 `nodeLabelSelector`。  
`nodeLabelSelector` 是一个 [LabelSelector](https://github.com/kubernetes/apimachinery/blob/60180f072f73eafec72ef9f2c418a6bb1357d434/pkg/apis/meta/v1/types.go#L1195)，
并根据节点标签定义过滤器。如果节点的标签与标签选择器条件匹配，
则会在该节点上启用 KSM。  
>**注意**  
>如果 `nodeLabelSelector` 为 nil，则不会在任何节点上启用 KSM。  
>空的 `nodeLabelSelector` 将在每个节点上启用 KSM。  

#### 示例： {#examples}

- 在主机名为 `node01` 或 `node03` 的节点上启用 KSM：
```yaml
spec:
  configuration:
    ksmConfiguration:
      nodeLabelSelector:
        matchExpressions:
          - key: kubernetes.io/hostname
            operator: In
            values:
              - node01
              - node03
```

- 在带有标签 `kubevirt.io/first-label: true`、`kubevirt.io/second-label: true` 的节点上启用 KSM：
```yaml
spec:
  configuration:
    ksmConfiguration:
      nodeLabelSelector:
        matchLabels:
          kubevirt.io/first-label: "true"
          kubevirt.io/second-label: "true"
```

- 在每个节点上启用 KSM：
```yaml
spec:
  configuration:
    ksmConfiguration:
      nodeLabelSelector: {}
```


## 注解与恢复机制 {#annotation-and-restore-mechanism}
 
在 KubeVirt 通过配置启用 KSM 的那些节点上，会添加一个注解
（`kubevirt.io/ksm-handler-managed`）。  
此注解是一条内部记录，用于跟踪当前由 virt-handler 管理的节点，以便在将来 ksmConfiguration 发生变化时，能够区分哪些节点应当被恢复。

设想如下场景：

1. 集群中有 3 个节点，其中一个（`node01`）已从外部启用了 KSM。
2. 管理员修补 KubeVirt CR，添加 ksmConfiguration，为 `node02` 和 `node03` 启用 ksm。
3. 过一段时间后，管理员再次修补 KubeVirt CR，删除 ksmConfiguration。

借助该注解，virt-handler 能够仅在它自己曾经启用过 ksm 的那些节点上停用 ksm（`node02` `node03`），而保持其他节点不变（`node01`）。

## 节点标签 {#node-labelling}

KubeVirt 可以发现哪些节点启用了 KSM，并用特殊标签（`kubevirt.io/ksm-enabled`）标记它们，其值为 `true`。
此标签可用于将 VM 调度到已启用或未启用 KSM 的节点上。
```yaml
apiVersion: kubevirt.io/v1
kind: VirtualMachine
metadata:
  name: testvm
spec:
  runStrategy: Always
  template:
    metadata:
      labels:
        kubevirt.io/vm: testvm
    spec:
      nodeSelector:
        kubevirt.io/ksm-enabled: "true"
      [...]
```
