# VSOCK {#vsock}

VM 套接字（vsock）是一种快速且高效的客户机与宿主机通信机制。

## 背景 {#background}

目前 KubeVirt 使用 virtio-serial 进行本地的客户机与宿主机通信。当前在 KubeVirt 中，libvirt 和 qemu 用它与 qemu-guest-agent 通信。virtio-serial 也可以被其他代理使用，但由于以下几点，它有些繁琐：

- virtio-serial 设备上只有少量端口
- 带宽低
- 无法进行基于套接字的通信，这就要求每个代理建立自己的协议，或者配合 SLIP 之类的转换层，才能使用 gRPC 之类的协议进行可靠通信。
- 没有简单且可支持的方法来获得已分配的 virtio-serial 套接字，并在不进入 virt-launcher Pod 的情况下访问它。
- 由于上面这一点，服务需要特权。

借助 [virtio-vsock](https://man7.org/linux/man-pages/man7/vsock.7.html)，我们可以获得对简便的客户机与宿主机通信的支持，并从用户/管理员的角度解决上述问题。

## 用法 {#usage}

### 功能门控 {#feature-gate}

要在 KubeVirt 集群中启用 VSOCK，用户可以扩展 KubeVirt CR 中的 `featureGates` 字段，方法是向其中添加 `VSOCK`。

```yaml
apiVersion: kubevirt.io/v1
kind: Kubevirt
metadata:
  name: kubevirt
  namespace: kubevirt
spec:
  ...
  configuration:
    developerConfiguration:
      featureGates:
        - "VSOCK"
```

或者，用户可以编辑现有的 kubevirt CR：

`kubectl edit kubevirt kubevirt -n kubevirt`

```yaml
...
spec:
  configuration:
    developerConfiguration:
      featureGates:
        - "VSOCK"
```

### 虚拟机实例 {#virtual-machine-instance}

要将 VSOCK 设备附加到虚拟机，用户必须添加 `autoattachVSOCK: true`，位置是在虚拟机实例规格的 `devices` 部分：

```yaml
apiVersion: kubevirt.io/v1
kind: VirtualMachineInstance
metadata:
  name: testvmi-vsock
spec:
  domain:
    resources:
      requests:
        memory: 64M
    devices:
      autoattachVSOCK: true
```

这将向 VM 公开 VSOCK 设备。`CID` 将由 `virt-controller` 随机分配，并公开到虚拟机实例的 status 中：

```yaml
status:
  VSOCKCID: 123
```

## 安全性 {#security}

> **_注意：_**  `/dev/vhost-vsock` 设备对于连接或绑定 VSOCK 套接字*并不需要*。

为使 VSOCK 功能保持安全，采取了以下措施：

- 全部 VSOCK 功能都将位于功能门控之后。
- 默认情况下，vsock 设备的前 1024 个端口是特权端口。试图绑定这些端口的服务需要 `CAP_NET_BIND_SERVICE` 能力。
-  `AF_VSOCK` 套接字系统调用在 containerd 1.7+ 中会被阻止（containerd/containerd#7442）。目前由厂商负责确保所使用的 CRI 选择一项默认 seccomp 策略，以与 containerd 类似的方式阻止 VSOCK 套接字调用。
- CID 由 `virt-controller` 分配，并且对每个虚拟机实例都是唯一的，以确保 `virt-handler` 能够方便地跟踪身份而不会发生竞争。虽然这仍然允许 `virt-launcher` 伪造使用已分配的 CID，但它消除了攻击者可能利用来重定向 VSOCK 调用的分配竞争。
