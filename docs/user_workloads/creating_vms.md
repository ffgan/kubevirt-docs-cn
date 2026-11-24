# 使用 virtctl 创建 VirtualMachine {#creating-virtualmachines-by-using-virtctl}

virtctl 子命令 `create vm` 允许从命令行方便地创建 VirtualMachine 清单。它默认使用[实例类型和偏好](../user_workloads/instancetypes.md)并进行推断（参见[使用实例类型和偏好](#using-instance-types-and-preferences)），并且提供若干标志来控制所创建虚拟机的细节。

例如，有些标志用来指定虚拟机的名称或运行策略，也有些标志用来向虚拟机添加卷。实例类型和偏好既可以显式指定，也可以让 KubeVirt 从用于启动该虚拟机的卷中推断它们。

要查看完整的标志集合及其说明，请使用下面的命令：

```shell
virtctl create vm -h
```

## 在集群上创建 VirtualMachine {#creating-virtualmachines-on-a-cluster}

`virtctl create vm` 的输出可以直接通过管道传给 `kubectl`，从而在集群上创建 VirtualMachine，例如：

```shell
# Create a VM with name my-vm on the cluster
virtctl create vm --name my-vm | kubectl create -f -
virtualmachine.kubevirt.io/my-vm created
```

## 使用实例类型和偏好 {#using-instance-types-and-preferences}

实例类型和偏好可以通过相应的标志来使用。如果没有另行指定，默认会从虚拟机的启动卷推断实例类型和偏好。关于推断的更多信息，见[下文](#inference-of-instance-type-andor-preference)。

下面的示例通过相应标志创建一台指定了实例类型和偏好的 VM：

```shell
virtctl create vm --instancetype my-instancetype --preference my-preference
```

实例类型或偏好的类型（命名空间作用域或集群作用域）可以通过在实例类型或偏好名称前加上相应的 CRD 名称作为前缀来控制，例如：

```shell
# Using a cluster scoped instance type and a namespaced preference
virtctl create vm \
  --instancetype virtualmachineclusterinstancetype/my-instancetype \
  --preference virtualmachinepreference/my-preference
```

如果未提供前缀，则默认使用集群作用域的资源。

### 推断实例类型和/或偏好 {#inference-of-instance-type-andor-preference}

要从用于启动虚拟机的卷显式推断[实例类型和/或偏好](../user_workloads/instancetypes.md#inferFromVolume)，请添加下面的标志：

```shell
virtctl create vm --infer-instancetype --infer-preference
```

隐式的默认行为是始终尝试从启动卷推断实例类型和偏好。此功能使用 `IgnoreInferFromVolumeFailure` 策略，该策略会抑制实例类型和偏好推断时的失败。如果上面的某个开关被显式指定，则会改用 `RejectInferFromVolumeFailure` 策略。这样，用户就会在虚拟机创建期间意识到潜在问题。

要从用于启动虚拟机的那个卷以外的另一个卷推断实例类型或偏好，请使用 `--infer-instancetype-from` 和 `--infer-preference-from` 标志来指定该虚拟机的任意卷。

```shell
# This virtual machine will boot from volume-a, but the instance type and
# preference are inferred from volume-b.
virtctl create vm \
  --volume-import=type:pvc,src:my-ns/my-pvc-a,name:volume-a \
  --volume-import=type:pvc,src:my-ns/my-pvc-b,name:volume-b \
  --infer-instancetype-from volume-b \
  --infer-preference-from volume-b
```

## 所添加卷的启动顺序 {#boot-order-of-added-volumes}

请注意，不同种类的卷目前具有下面这一固定的启动顺序，与这些标志在命令行上的指定顺序无关：

1. Containerdisk
2. 直接使用的 PVC
3. DataSource
4. 克隆的 PVC
5. 空白卷
6. 导入的卷（通过 `--volume-import` 标志）

如果指定了多个相同种类的卷，它们的顺序由这些标志被指定的先后顺序决定。

## 生成 cloud-init 用户数据 {#generating-cloud-init-user-data}

要使用 `virtctl create vm` 生成 cloud-init 用户数据，可以使用下面的标志。

!!! Note
    生成 cloud-init 用户数据与[指定自定义 cloud-init 用户数据](#specifying-custom-cloud-init-user-data)互斥，如下所述。

### `--user` 标志 {#-user-flag}

指定由 cloud-init 创建的虚拟机主用户。它会在生成的 cloud-init 用户数据中设置 `user` 参数。

### `--password-file` 标志 {#-password-file-flag}

指定一个文件，从中读取虚拟机主用户的密码。在生成的 cloud-init 用户数据中，它把 `password` 参数的值设置为所读入的值，并把 `chpasswd` 参数的值设置为 `{ expire: False }`。

### `--ssh-key` 标志 {#-ssh-key-flag}

为虚拟机的主用户指定一个或多个 SSH 授权密钥。它会在生成的 cloud-init 用户数据中设置 `ssh_authorized_keys` 参数。

### `--ga-manage-ssh` 标志 {#-ga-manage-ssh-flag}

设置此标志时，会向生成的 cloud-init 用户数据中添加一条命令，使 `qemu-guest-agent` 能够管理 SSH 授权密钥。该命令被添加到 `runcmd` 参数中；这在启用了 SELinux 的发行版上是必需的，否则这些发行版不会允许 `qemu-guest-agent` 管理用户主目录中的 SSH 授权密钥。

### 示例 {#example}

```shell
$ virtctl create vm --user myuser --access-cred=src:my-keys --ga-manage-ssh
```

此命令会生成下面的 cloud-init 用户数据：

```yaml
volumes:
  - cloudInitNoCloud:
      userData: |-
        #cloud-config
        user: myuser
        runcmd:
          - [ setsebool, -P, 'virt_qemu_ga_manage_ssh', 'on' ]
    name: cloudinitdisk
```

通过显式传入 `--ga-manage-ssh` 标志，`qemu-guest-agent` 能够管理从 Secret `my-keys` 读取的凭据，该 Secret 被指定为 `--access-cred` 标志的 source 参数。请注意，如果没有把 `--ga-manage-ssh` 显式设置为 `false`，这也是默认行为。

## 指定自定义 cloud-init 用户数据 {#specifying-custom-cloud-init-user-data}

要把自定义 cloud-init 用户数据传给 virtctl，需要把它编码为 base64 字符串。

!!! Note
    指定自定义 cloud-init 用户数据与[生成 cloud-init 用户数据](#generating-cloud-init-user-data)互斥，如上所述。

下面是一个做法示例：

```shell
# Put your cloud-init user data into a file.
# This will add an authorized key to the default user.
# To get the default username read the documentation for the cloud image
$ cat cloud-init.txt
#cloud-config
ssh_authorized_keys:
  - ssh-rsa AAAA...

# Base64 encode the contents of the file without line wraps and store it in a variable
$ CLOUD_INIT_USERDATA=$(base64 -w 0 cloud-init.txt)

# Show the contents of the variable
$ echo $CLOUD_INIT_USERDATA
I2Nsb3VkLWNvbmZpZwpzc2hfYXV0aG9yaXplZF9rZXlzOgogIC0gc3NoLXJzYSBBQUFBLi4uCg==
```

现在可以把这个变量作为 `--cloud-init-user-data` 标志的参数：

```shell
virtctl create vm --cloud-init-user-data $CLOUD_INIT_USERDATA
```

## 向虚拟机添加访问凭据 {#adding-access-credentials-to-a-virtual-machine}

通过使用 `--access-cred` 标志，`virtctl create vm` 命令可以在所创建的虚拟机中配置[访问凭据](./accessing_virtual_machines.md#dynamic-ssh-public-key-injection-via-qemu-guest-agent)。它支持 SSH 授权密钥和密码这两种访问凭据，并且可以把它们配置为通过 `qemu-guest-agent` 注入，或者通过 cloud-init 元数据注入。该标志所支持的参数取决于所选的 `type` 和 `method`。该标志可以多次传入，以配置不止一个访问凭据。

此标志会与用于[生成 cloud-init 用户数据](#generating-cloud-init-user-data)的那些标志相互作用：它为 SSH 密钥注入沿用相同的 `--user`，并且会让 `qemu-guest-agent` 管理 SSH 授权密钥（`--ga-manage-ssh`），前提是用户没有显式禁用它。

### 示例 {#example_1}

```shell
$ virtctl create vm --user myuser --access-cred=src:my-keys
```

此命令会生成下面的访问凭据和 cloud-init 用户数据：

```
[...]
accessCredentials:
  - sshPublicKey:
      propagationMethod:
        qemuGuestAgent:
          users:
            - myuser
      source:
        secret:
          secretName: my-keys
volumes:
  - cloudInitNoCloud:
      userData: |-
        #cloud-config
        user: myuser
        runcmd:
          - [ setsebool, -P, 'virt_qemu_ga_manage_ssh', 'on' ]
    name: cloudinitdisk
```

## 添加 sysprep 卷 {#adding-a-sysprep-volume}

可以通过传入 `--volume-sysprep` 标志，向所创建的虚拟机添加 [sysprep 卷](startup_scripts.md#sysprep-examples)，该标志传给 `virtctl
create vm` 命令。

该标志支持从 `ConfigMap` 或 `Secret` 添加 sysprep 卷。

请参阅示例了解具体做法。

## 简短示例 {#short-examples}

为具有随机名称的 VirtualMachine 创建清单：

```shell
virtctl create vm
```

为具有指定名称且 RunStrategy 为 Always 的 VirtualMachine 创建清单：

```shell
virtctl create vm --name=my-vm --run-strategy=Always
```

为指定了 VirtualMachineClusterInstancetype 的 VirtualMachine 创建清单：

```shell
virtctl create vm --instancetype=my-instancetype
```

为指定了 VirtualMachineInstancetype（命名空间作用域）的 VirtualMachine 创建清单：

```shell
virtctl create vm --instancetype=virtualmachineinstancetype/my-instancetype
```

为指定了 VirtualMachineClusterPreference 的 VirtualMachine 创建清单：

```shell
virtctl create vm --preference=my-preference
```

为指定了 VirtualMachinePreference（命名空间作用域）的 VirtualMachine 创建清单：

```shell
virtctl create vm --preference=virtualmachinepreference/my-preference
```

为指定了内存以及一个临时 containerdisk 卷的 VirtualMachine 创建清单：

```shell
virtctl create vm --memory=1Gi \
  --volume-containerdisk=src:my.registry/my-image:my-tag
```

为在命名空间中克隆了 DataSource 并指定了大小的 VirtualMachine 创建清单：

```shell
virtctl create vm --volume-import=type:ds,src:my-ns/my-ds,size:50Gi
```

为克隆了 DataSource 并推断实例类型和偏好的 VirtualMachine 创建清单：

```shell
virtctl create vm --volume-import=type:ds,src:my-annotated-ds \
  --infer-instancetype --infer-preference
```

为具有多个卷并指定了启动顺序的 VirtualMachine 创建清单：

```shell
virtctl create vm --volume-containerdisk=src:my.registry/my-image:my-tag \
  --volume-import=type:ds,src:my-ds,bootorder:1
```

为具有多个卷、并从指定卷推断实例类型和偏好的 VirtualMachine 创建清单：

```shell
virtctl create vm --volume-import=type:ds,src:my-annotated-ds \
  --volume-pvc=my-annotated-pvc --infer-instancetype=my-annotated-ds \
  --infer-preference=my-annotated-pvc
```

为克隆了 PVC 的 VirtualMachine 创建清单：

```shell
virtctl create vm --volume-import=type:pvc,src:my-ns/my-pvc
```

为使用 PVC 但不克隆它的 VirtualMachine 创建清单：

```shell
virtctl create vm --volume-pvc=src:my-pvc
```

为带有克隆 DataSource 和空白卷的 VirtualMachine 创建清单：

```shell
virtctl create vm --volume-import=type:ds,src:my-ns/my-ds \
  --volume-import=type:blank,size:50Gi
```

为指定了 VirtualMachineCluster{Instancetype,Preference} 并克隆了 DataSource 的 VirtualMachine 创建清单：

```shell
virtctl create vm --instancetype=my-instancetype --preference=my-preference \
  --volume-import=type:ds,src:my-ds
```

为指定了 VirtualMachineCluster{Instancetype,Preference} 以及两个克隆 DataSource 的 VirtualMachine 创建清单（该标志可以提供多次）：

```shell
virtctl create vm --instancetype=my-instancetype --preference=my-preference \
  --volume-import=type:ds,src:my-ds1 --volume-import=type:ds,src:my-ds2
```

为指定了 VirtualMachineCluster{Instancetype,Preference} 并直接使用 PVC 的 VirtualMachine 创建清单：

```shell
virtctl create vm --instancetype=my-instancetype --preference=my-preference \
  --volume-pvc=my-pvc
```

为指定了 DataVolumeTemplate 的 VirtualMachine 创建清单：

```shell
virtctl create vm \
  --volume-import=type:pvc,name:my-pvc,namespace:default,size:256Mi
```

为生成了 cloud-init 配置、用于设置用户并添加一条 ssh 授权密钥的 VirtualMachine 创建清单：

```shell
virtctl create vm --user=cloud-user --ssh-key="ssh-ed25519 AAAA...."
```

为生成了 cloud-init 配置、用于设置用户并从文件设置密码的 VirtualMachine 创建清单：

```shell
virtctl create vm --user=cloud-user --password-file=/path/to/file
```

为这样的 VirtualMachine 创建清单：把 SSH 公钥从名为 my-keys 的 secret 注入 VM，并注入到同样在 cloud-init 配置中指定的用户：

```shell
virtctl create vm --user=cloud-user --access-cred=type:ssh,src:my-keys
```

为这样的 VirtualMachine 创建清单：把 SSH 公钥从名为 my-keys 的 secret 注入 VM，并注入到作为参数指定的用户：

```shell
virtctl create vm --access-cred=type:ssh,src:my-keys,user:myuser
```

为把密码从名为 my-pws 的 secret 注入 VM 的 VirtualMachine 创建清单：

```shell
virtctl create vm --access-cred=type:password,src:my-pws
```

为带有 Containerdisk 和 Sysprep 卷的 VirtualMachine 创建清单（源 ConfigMap 需要已经存在）：

```shell
virtctl create vm --memory=1Gi \
  --volume-containerdisk=src:my.registry/my-image:my-tag \
  --volume-sysprep=src:my-cm
```

## 复杂示例 {#complex-examples}

这些示例展示 `virtctl create vm` 如何用于更复杂的场景。

### 第一个示例 {#first-example}

创建具有下列设置的 VirtualMachine：

- 运行策略：`Manual`
- 终止宽限期：`123` 秒
- 实例类型：`u1.small`
- 偏好：`fedora`
- 使用 `quay.io/containerdisks/fedora` containerdisk 作为第一个卷
- 添加大小为 `1Gi` 的第二个空白卷
- 主用户名为 `myuser`
- 可以使用所指定的授权密钥以主用户登录

```shell
virtctl create vm --run-strategy=Manual --termination-grace-period=123 \
  --instancetype=u1.small --preference=fedora \
  --volume-containerdisk=src:quay.io/containerdisks/fedora \
  --volume-import=type:blank,size:1Gi \
  --user=myuser --ssh-key='ssh-ed25519 AAAA...'
```

### 第二个示例 {#second-example}

创建具有下列设置、并使用 secret 来配置访问凭据的 VirtualMachine：

- 实例类型：`u1.small`
- 偏好：`fedora`
- 使用 `quay.io/containerdisks/fedora` containerdisk 作为第一个卷
- 添加大小为 `1Gi` 的第二个空白卷
- 主用户名为 `myuser`
- 可以使用访问凭据中所指定的授权密钥以主用户登录

```shell
# First create the secret with the public key:
kubectl create secret generic my-keys --from-file=$HOME/.ssh/id_ed25519.pub

# Then create the VM on the cluster
virtctl create vm --name my-vm --instancetype=u1.small --preference=fedora \
  --volume-containerdisk=src:quay.io/containerdisks/fedora \
  --volume-import=type:blank,size:1Gi --user=myuser \
  --access-cred=src:my-keys | kubectl create -f -

# Login via SSH once the VM is ready
virtctl ssh -i $HOME/.ssh/id_ed25519 myuser@my-vm
```
