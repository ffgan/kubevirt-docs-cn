# 访问虚拟机 {#accessing-virtual-machines}

如下所示，有多种方法可以访问 `VirtualMachines`。

说明一下：可以把 `VirtualMachineInstance` 视为正在运行的 `VirtualMachine`。

## 图形控制台和串行控制台访问 {#graphical-and-serial-console-access}

虚拟机启动后，你可以连接到它暴露的控制台。通常有两种控制台：

- 串行控制台
- 图形控制台（VNC）

> 注意：你需要已经安装 `virtctl`
> （见[安装](../user_workloads/virtctl_client_tool.md)），才能
> 访问 VirtualMachineInstance。

### 访问串行控制台 {#accessing-the-serial-console}

可以使用 `console` 命令访问虚拟机的串行控制台：

```shell
virtctl console testvm
```

### 访问图形控制台（VNC） {#accessing-the-graphical-console-vnc}

要访问虚拟机的图形控制台，通常使用 VNC 协议。这需要安装 `remote-viewer`。安装该工具后，可以使用以下命令访问图形控制台：

```shell
virtctl vnc testvm
```

如果只想打开 vnc-proxy 而不执行 `remote-viewer` 命令，可以用以下方式完成：

```shell
virtctl vnc --proxy-only testvm
```

这会在你的机器上打印端口号，你可以用任何 VNC 查看器手动连接。

### 调试控制台访问 {#debugging-console-access}

如果连接失败，可以使用 `-v` 标志，从 `virtctl` 和 `remote-viewer` 工具获得更详细的输出，以便排查问题。

```shell
virtctl vnc testvm -v 4
```

> **注意：** 如果你通过 SSH 在远程机器上使用 virtctl，需要把 X 会话转发到你的机器。如果不熟悉，请查阅 `ssh` 的 -X 和 -Y 标志。作为替代，你可以用 SSH 把 API 服务器端口代理到你的机器（直接代理，或与 `kubectl proxy` 结合使用）。

## SSH 访问 {#ssh-access}

管理虚拟机时一种常见的运维模式是在启动时把 SSH 公钥注入虚拟机。这样自动化工具（如 Ansible）就可以置备虚拟机。这也为运维人员提供了一种安全且免密码访问虚拟机的方式。

KubeVirt 提供多种把 SSH 公钥注入虚拟机的方式。

一般来说，这些方法分为两类：
 - [静态密钥注入](../user_workloads/accessing_virtual_machines.md#static-ssh-key-injection-via-cloud-init)，在虚拟机首次启动时把密钥放到虚拟机上。
 - [动态密钥注入](../user_workloads/accessing_virtual_machines.md#dynamic-ssh-key-injection-via-qemu-user-agent)，允许在启动时和运行期间动态更新密钥。

一旦 SSH 公钥注入虚拟机，就可以通过 `virtctl` 访问。

### 通过 cloud-init 静态注入 SSH 公钥 {#static-ssh-key-injection-via-cloud-init}

创建虚拟机的用户可以向虚拟机提供启动脚本，从而进行多种定制操作。

把公钥注入 VM 的一种办法是通过 [cloud-init 启动脚本](../user_workloads/startup_scripts.md#cloud-init)。不过，还有更灵活的办法。

虚拟机的访问凭据 API 允许在启动时静态注入 SSH 公钥，这独立于 cloud-init 用户数据，做法是把 SSH 公钥放入 Kubernetes `Secret`。这样就可以把 cloud-init 用户数据中的应用数据，与用于访问虚拟机的凭据分开。

可以这样从 SSH 公钥创建 Kubernetes `Secret`：

```shell
# Place SSH public key into a Secret
kubectl create secret generic my-pub-key --from-file=key1=id_rsa.pub
```

然后把包含公钥的 `Secret` 通过访问凭据 API、以 `noCloud` 传播方法分配给虚拟机。

KubeVirt 使用生成的 cloud-init 元数据而不是用户数据，把 SSH 公钥注入虚拟机。这样就把应用用户数据和用于访问虚拟机的凭据分开了。

> 注意：不会改动 cloud-init 的 `userData`。

```shell
# Create a VM referencing the Secret using propagation method noCloud
kubectl create -f - <<EOF
apiVersion: kubevirt.io/v1
kind: VirtualMachine
metadata:
  name: testvm
spec:
  runStrategy: Always
  template:
    spec:
      domain:
        devices:
          disks:
          - disk:
              bus: virtio
            name: containerdisk
          - disk:
              bus: virtio
            name: cloudinitdisk
          rng: {}
        resources:
          requests:
            memory: 1024M
      terminationGracePeriodSeconds: 0
      accessCredentials:
      - sshPublicKey:
          source:
            secret:
              secretName: my-pub-key
          propagationMethod:
            noCloud: {}
      volumes:
      - containerDisk:
          image: quay.io/containerdisks/fedora:latest
        name: containerdisk
      - cloudInitNoCloud:
          userData: |-
            #cloud-config
            password: fedora
            chpasswd: { expire: False }
        name: cloudinitdisk
EOF
```

### 通过 qemu-guest-agent 动态注入 SSH 公钥 {#dynamic-ssh-key-injection-via-qemu-user-agent}

KubeVirt 允许借助访问凭据 API，向 VirtualMachine 动态注入 SSH 公钥。

利用 `qemuGuestAgent` 传播方法，配置好的 Secret 会在 VM 启动时挂接到 VirtualMachine。这样就可以在运行时通过更新已挂接的 Secret 来动态注入 SSH 公钥。

请注意，不能把新的 Secret 挂接到正在运行的 VM：必须重启 VM 才能挂接新的 Secret。

> 注意：这要求在客户机内安装 qemu-guest-agent。
>
> 注意：使用 qemuGuestAgent 传播时，
> `/home/$USER/.ssh/authorized_keys` 文件将归客户机代理所有。
> 不是由客户机代理做出的文件更改将会丢失。
>
> 注意：关于访问凭据 API 背后动机的更多信息，
> 见引入该 API 的
> [拉取请求描述](https://github.com/kubevirt/kubevirt/pull/4195)。

在下面的示例中，包含 SSH 公钥的 `Secret` 通过访问凭据 API、以 `qemuGuestAgent` 传播方法挂接到虚拟机。这允许随时更新 `Secret` 的内容，变更会立即应用到正在运行的虚拟机。`Secret` 也可以包含多个 SSH 公钥。

```shell
# Place SSH public key into a secret
kubectl create secret generic my-pub-key --from-file=key1=id_rsa.pub
```

现在，在 `VirtualMachine` spec 中用访问凭据 API 引用此 secret，并使用 `qemuGuestAgent` 传播方法。

```shell
# Create a VM referencing the Secret using propagation method qemuGuestAgent
kubectl create -f - <<EOF
apiVersion: kubevirt.io/v1
kind: VirtualMachine
metadata:
  name: testvm
spec:
  runStrategy: Always
  template:
    spec:
      domain:
        devices:
          disks:
          - disk:
              bus: virtio
            name: containerdisk
          - disk:
              bus: virtio
            name: cloudinitdisk
          rng: {}
        resources:
          requests:
            memory: 1024M
      terminationGracePeriodSeconds: 0
      accessCredentials:
      - sshPublicKey:
          source:
            secret:
              secretName: my-pub-key
          propagationMethod:
            qemuGuestAgent:
              users:
              - fedora
      volumes:
      - containerDisk:
          image: quay.io/containerdisks/fedora:latest
        name: containerdisk
      - cloudInitNoCloud:
          userData: |-
            #cloud-config
            password: fedora
            chpasswd: { expire: False }
            # Disable SELinux for now, so qemu-guest-agent can write the authorized_keys file
            # The selinux-policy is too restrictive currently, see open bugs:
            #   - https://bugzilla.redhat.com/show_bug.cgi?id=1917024
            #   - https://bugzilla.redhat.com/show_bug.cgi?id=2028762
            #   - https://bugzilla.redhat.com/show_bug.cgi?id=2057310
            bootcmd:
              - setenforce 0
        name: cloudinitdisk
EOF
```

### 使用 virtctl 访问 VM 或 VMI {#accessing-a-vm-or-vmi-using-virtctl}

用户可以使用 `virtualmachineinstances/portforward` 子资源（属于 `VirtualMachineInstance`），或 `virtualmachines/portfoward` 子资源（属于 `VirtualMachine`），创建一条到 VMI 内部某个端口的、由 websocket 支撑的网络隧道。

这些子资源的一个使用场景，是把 SSH 流量转发进 `VirtualMachineInstance`，无论是从 CLI 还是从 Web UI。`VirtualMachine` 子资源会自动查找对应的 `VirtualMachineInstance`。

要从本地机器连接到 `VirtualMachineInstance`，`virtctl` 用 `ssh` 命令包装本地 SSH 客户端，并透明地使用上面描述的端口转发。更多细节请参阅该命令的帮助。

```shell
virtctl ssh
```

要从 `VirtualMachineInstance` 传入或传出文件，`virtctl` 还通过 `scp` 命令提供 SCP 客户端。其用法与 `ssh` 命令类似。更多细节请参阅该命令的帮助。

```shell
virtctl scp
```

#### 把 virtctl 用作代理 {#using-virtctl-as-proxy}

连接 VM 和 VMI 的首选方式是使用 `virtctl` 包装本地 OpenSSH 客户端。如果你更愿意直接使用本地 OpenSSH 客户端，也可以结合 `virtctl` 这样做。

> 注意：其中大部分内容也适用于 `virtctl scp` 命令。

1. 默认情况下，`virtctl` 会以对用户透明的方式包装本地 OpenSSH 客户端。可以通过提高详细程度（`-v 3`）查看实际执行的 SSH 命令。

```shell
virtctl ssh --local-ssh -v 3 vm/testvm/mynamespace
```

2. `virtctl port-forward` 命令提供一个选项，把单个端口隧道到本地的 stdout/stdin。这样该命令就可以与 OpenSSH 客户端的 `ProxyCommand` 选项结合使用。

```shell
ssh -o 'ProxyCommand=virtctl port-forward --stdio=true vm/testvm/mynamespace 22' fedora@vm/testvm/mynamespace
```

为了更方便地访问任意虚拟机，可以把以下各行加入你的 SSH `config`：

```
Host vmi/*
   ProxyCommand virtctl port-forward --stdio=true %h %p
Host vm/*
   ProxyCommand virtctl port-forward --stdio=true %h %p
```

这样你只需调用 `ssh user@vm/testvm/mynamespace`，你的 SSH `config` 和 `virtctl` 会完成其余工作。使用这种方法，很容易在 SSH `config` 里为不同命名空间设置不同的身份。

此功能也可以与 Ansible 一起使用，以自动化配置在 KubeVirt 上运行的虚拟机。你可以把上面的片段放到单独的文件中（例如 `~/.ssh/virtctl-proxy-config`），并在 `.ansible.cfg` 中加入以下各行：

```
[ssh_connection]
ssh_args = -F ~/.ssh/virtctl-proxy-config
```

请注意，所有端口转发流量都会经过 Kubernetes 控制平面。大量连接和流量会增加 API 服务器的压力。如果你经常需要大量连接和流量，请考虑改用专用的 Kubernetes `Service`。

#### 示例 {#example}

1. 按上面的说明创建虚拟机并注入 SSH 公钥

2. SSH 进入虚拟机

```shell
virtctl ssh -i id_rsa fedora@vm/testvm/mynamespace
```

或

```shell
ssh -o 'ProxyCommand=virtctl port-forward --stdio=true vm/testvm/mynamespace 22' -i id_rsa fedora@vm/testvm/mynamespace
```

3. 用 SCP 把文件传到虚拟机

```shell
virtctl scp -i id_rsa testfile fedora@vm/testvm/mynamespace:/tmp
```

或

```shell
scp -o 'ProxyCommand=virtctl port-forward --stdio=true vmi/testvm/mynamespace 22' -i id_rsa testfile fedora@vm.testvm.mynamespace:/tmp
```

!!! Note
    本地 `scp` 不支持主机名中的斜杠。因此该示例
    像 `virtctl` 包装本地 SCP 客户端时那样，用点来分隔
    类型、名称和命名空间。

#### 控制台/VNC/SSH 访问的 RBAC 权限 {#rbac-permissions-for-consolevncssh-access}

##### 使用默认 RBAC 集群角色 {#using-default-rbac-cluster-roles}

从 v0.5.1 版本开始，每次 KubeVirt 安装都会附带一组默认的 RBAC 集群角色，可用于授予用户访问 VirtualMachineInstance 的权限。

`kubevirt.io:admin` 和 `kubevirt.io:edit` 集群角色内置了控制台、VNC 以及 SSH 端口转发的访问权限。把其中任一角色绑定给用户后，该用户就可以使用 virtctl 访问控制台、VNC 和 SSH。

##### 使用自定义 RBAC 集群角色 {#using-custom-rbac-cluster-role}

默认的 KubeVirt 集群角色授予的权限不止控制台、VNC 和端口转发。下面的 `ClusterRole` 演示如何编写一个自定义角色，只允许访问控制台、VNC 和端口转发。

```yaml
apiVersion: rbac.authorization.k8s.io/v1beta1
kind: ClusterRole
metadata:
  name: allow-console-vnc-port-forward-access
rules:
  - apiGroups:
      - subresources.kubevirt.io
    resources:
      - virtualmachineinstances/console
      - virtualmachineinstances/vnc
    verbs:
      - get
  - apiGroups:
      - subresources.kubevirt.io
    resources:
      - virtualmachineinstances/portforward
    verbs:
      - update
```

用 `ClusterRoleBinding` 绑定后，上面的 `ClusterRole` 会授予对所有命名空间中虚拟机的访问权限。

要把范围缩小到单个命名空间，请绑定此 `ClusterRole`，并使用针对单个命名空间的 `RoleBinding`。
