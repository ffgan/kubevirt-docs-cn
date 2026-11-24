# 无响应节点 {#unresponsive-nodes}

KubeVirt 有自己的节点守护进程，称为 virt-handler。除了检测节点问题的常规 k8s 方法之外，virt-handler 守护进程还有自己的心跳机制。这允许对 VirtualMachineInstance 进行精细的错误处理。

## virt-handler 心跳 {#virt-handler-heartbeat}

`virt-handler` 会定期尝试更新它所在节点上的
`kubevirt.io/schedulable` 标签和 `kubevirt.io/heartbeat`
注解：

    $ kubectl get nodes -o yaml
    apiVersion: v1
    items:
    - apiVersion: v1
      kind: Node
      metadata:
        annotations:
          kubevirt.io/heartbeat: 2018-11-05T09:42:25Z
        creationTimestamp: 2018-11-05T08:55:53Z
        labels:
          beta.kubernetes.io/arch: amd64
          beta.kubernetes.io/os: linux
          cpumanager: "false"
          kubernetes.io/hostname: node01
          kubevirt.io/schedulable: "true"
          node-role.kubernetes.io/control-plane: ""

如果某个 `VirtualMachineInstance` 被调度，调度器只会
考虑 `kubevirt.io/schedulable` 为 `true` 的节点。查看
`VirtualMachineInstance` 对应的 Pod 即可看到这一点：

    $ kubectl get pods  virt-launcher-vmi-nocloud-ct6mr -o yaml
    apiVersion: v1
    kind: Pod
    metadata:
      [...]
    spec:
      [...]
      nodeName: node01
      nodeSelector:
        kubevirt.io/schedulable: "true"
      [...]

如果出现通信问题或宿主机宕机，
`virt-handler` 将无法再更新其标签和注解。一旦
最近的 `kubevirt.io/heartbeat` 时间戳早于五分钟，
KubeVirt node-controller 就会介入，并将
`kubevirt.io/schedulable` 标签设置为 `false`。因此，在 virt-handler 重新连接之前，
不会再有 VMI 被调度到此节点。

## 当 virt-handler 无响应时删除卡住的 VMI {#deleting-stuck-vmis-when-virt-handler-is-unresponsive}

如果 `virt-handler` 出现一些问题但节点总体
正常，可以照常删除 `VirtualMachineInstance`，方法是
`kubectl delete vmi <myvm>`。`VirtualMachineInstance` 的 Pod 会
被 cluster-controllers 告知应当关闭。一旦
Pod 消失，`VirtualMachineInstance` 将被移至 `Failed`
状态，前提是 `virt-handler` 在此期间未能更新其心跳。
如果 `virt-handler` 在此期间能够恢复，
则由 `virt-handler` 而不是 cluster-controllers 将 `VirtualMachineInstance` 移至失败状态。

## 当整个节点无响应时删除卡住的 VMI {#deleting-stuck-vmis-when-the-whole-node-is-unresponsive}

如果整个节点无响应，删除 `VirtualMachineInstance` 时若仅使用
`kubectl delete vmi <myvmi>`，则永远无法移除该
`VirtualMachineInstance`。在这种情况下，需要强制删除无响应节点上的所有 Pod：首先确认该节点确实已经宕机。
然后通过强制删除来删除该节点上的所有 Pod：
`kubectl delete pod --force --grace-period=0 <mypod>`。

一旦 Pod 消失且 virt-handler 的心跳
超时，VMI 将被移至 `Failed` 状态。如果它们已被
标记为删除，则会直接消失。如果没有，可以将其
删除，并且几乎会立即消失。

## 时间方面的考虑 {#timing-considerations}

KubeVirt 集群组件最多需要五分钟才能
检测到 virt-handler 不健康。在此时间段内，
新的 VMI 有可能被调度到受影响的节点。如果
virt-handler 无法连接到该节点上的这些 Pod，这些
Pod 迟早会进入失败状态。一旦集群
最终检测到该问题，VMI 将被集群设置为失败。
