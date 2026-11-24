# 运行策略 {#run-strategies}

## 概述 {#overview}

VirtualMachine 具有 `spec.runStrategy` 字段，它决定是否应当有客户机在运行，以及当客户机停止时 KubeVirt 应当做什么。

较旧的清单则使用布尔字段 `spec.running`。因为对于带有 `spec.running: true` 的 VirtualMachine，KubeVirt 总会立即重启 VirtualMachineInstance，所以一个简单的布尔值并不总是足以完整描述期望的行为。例如，有时用户希望能够从虚拟机内部关闭客户机。在 `spec.running: true` 的情况下，KubeVirt 会立即重启 VirtualMachineInstance。

!!! warning
    自 KubeVirt v1.4.0 起，`spec.running` 已被弃用。请改用
    `spec.runStrategy`：`spec.running: true` 对应于
    `runStrategy: Always`，而 `spec.running: false` 对应于
    `runStrategy: Halted`。API 服务器仍然接受 `spec.running`，
    但在创建或更新设置了该字段的 VirtualMachine 时会返回警告
    `spec.running is deprecated, please use spec.runStrategy instead.`。

## RunStrategy {#runstrategy}

为了允许用户状态有更大的变化，引入了 `RunStrategy` 字段。它与 `Running` 互斥，因为二者的条件有所重叠。目前定义了五种 RunStrategy：

-   Always：系统的任务是使 VM 保持运行状态。
    每当当前 VirtualMachineInstance 以受控方式（例如从客户机内部关机）或不受控方式（例如崩溃）终止时，系统会重新创建一个 VirtualMachineInstance，从而实现这一点。
    此行为等同于 `spec.running: true`。

-   RerunOnFailure：与 `Always` 类似，不同之处在于，只有当 VM 以不受控方式（例如崩溃）终止，并且是由于基础设施原因（即节点崩溃、与 KVM 相关的进程发生了 OOM）时，才会重启 VM。
    这允许用户通过在客户机内部发起关机来决定 VM 应当何时关闭。
    注意：客户机侧崩溃（即 BSOD）不在此范围内。
    在这种情况下，存活检查或使用看门狗会有所帮助。

-   Once：VM 将运行一次，完成后不会重启，无论完成阶段是 Failure 还是 Success。

-   Manual：系统不会自动开启或关闭 VM，而是由用户通过在 VirtualMachine 子资源端点上发出 start、stop 和 restart 命令来手动控制 VM 状态。

-   Halted：系统被要求确保没有 VM 在运行。
    这是通过停止与该 VM 关联的任何 VirtualMachineInstance 来实现的。如果客户机已经在运行，它将被停止。
    此行为等同于 `spec.running: false`。

*注意*：`RunStrategy` 和 `running` 互斥，因为它们可能相互矛盾。API 服务器将拒绝同时定义这两者的 VirtualMachine 资源。

### virtctl {#virtctl}

virtctl 的 `start`、`stop` 和 `restart` 方法将调用 VirtualMachine 各自对应的子资源。这可能会对 VirtualMachine 的 runStrategy 产生如下影响：

| RunStrategy        | start              | stop               | restart            |
|--------------------|--------------------|--------------------|--------------------|
| **Always**         | `-`                | `Halted`           | `Always`           |
| **RerunOnFailure** | `RerunOnFailure`   | `RerunOnFailure`   | `RerunOnFailure`   |
| **Manual**         | `Manual`           | `Manual`           | `Manual`           |
| **Halted**         | `Always`           | `-`                | `-`                |

标有 `-` 的表项没有意义，因此不会对 RunStrategy 产生影响。

## RunStrategy 示例 {#runstrategy-examples}

### Always {#always}

Always RunStrategy 的一个使用示例。

```yaml
apiVersion: kubevirt.io/v1
kind: VirtualMachine
metadata:
  labels:
    kubevirt.io/vm: vm-cirros
  name: vm-cirros
spec:
  runStrategy: Always
  template:
    metadata:
      labels:
        kubevirt.io/vm: vm-cirros
    spec:
      domain:
        devices:
          disks:
          - disk:
              bus: virtio
            name: containerdisk
      terminationGracePeriodSeconds: 0
      volumes:
      - containerDisk:
          image: kubevirt/cirros-container-disk-demo:latest
        name: containerdisk
```
