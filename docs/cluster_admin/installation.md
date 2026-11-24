# 安装 {#installation}

KubeVirt 是 Kubernetes 的虚拟化附加组件，本指南假定
Kubernetes 集群已经安装完成。

如果安装在 OKD 上，Web 控制台会被扩展，用于管理
虚拟机。

## 要求 {#requirements}

开始之前需要满足以下几项要求：

-   [Kubernetes](https://kubernetes.io) 集群或衍生发行版
    （例如 [OpenShift](https://github.com/openshift/origin)），
    基于 KubeVirt 发布时已经推出的最近三个 Kubernetes 版本之一。
-   Kubernetes apiserver 必须设置 `--allow-privileged=true`，以便运行 KubeVirt 的特权 DaemonSet。
-   `kubectl` 客户端工具

### 容器运行时支持 {#container-runtime-support}

KubeVirt 目前支持以下容器运行时：

-   containerd
-   crio（配合 runv）

其他不使用虚拟化特性的容器运行时
应当也能工作。不过，上面提到的那些是主要目标。

### 与 AppArmor 集成 {#integration-with-apparmor}

在大多数场景中，KubeVirt 可以在启用
AppArmor 的系统上正常运行。不过，有几种已知用例可能需要
用户进行额外操作。

-   在启用 AppArmor 的系统上，本地安装的配置文件
    可能会阻止 KubeVirt 特权容器执行。这
    通常会导致 `virt-handler`
    Pod 初始化失败：

        $ kubectl get pods -n kubevirt
        NAME                               READY   STATUS       RESTARTS         AGE
        virt-api-77df5c4f87-7mqv4          1/1     Running      1 (17m ago)      27m
        virt-api-77df5c4f87-wcq44          1/1     Running      1 (17m ago)      27m
        virt-controller-749d8d99d4-56gb7   1/1     Running      1 (17m ago)      27m
        virt-controller-749d8d99d4-78j6x   1/1     Running      1 (17m ago)      27m
        virt-handler-4w99d                 0/1     Init:Error   14 (5m18s ago)   27m
        virt-operator-564f568975-g9wh4     1/1     Running      1 (17m ago)      31m
        virt-operator-564f568975-wnpz8     1/1     Running      1 (17m ago)      31m

        $ kubectl logs -n kubevirt virt-handler-4w99d virt-launcher
        error: failed to get emulator capabilities

        error: internal error: Failed to start QEMU binary /usr/libexec/qemu-kvm for probing: libvirt: error : cannot execute binary /usr/libexec/qemu-kvm: Permission denied

        $ journalctl -b | grep DEN
        ...
        May 18 16:44:20 debian audit[6316]: AVC apparmor="DENIED" operation="exec" profile="libvirtd" name="/usr/libexec/qemu-kvm" pid=6316 comm="rpc-worker" requested_mask="x" denied_mask="x" fsuid=107 ouid=0
        May 18 16:44:20 debian kernel: audit: type=1400 audit(1652888660.539:39): apparmor="DENIED" operation="exec" profile="libvirtd" name="/usr/libexec/qemu-kvm" pid=6316 comm="rpc-worker" requested_mask="x" denied_mask="x" fsuid=107 ouid=0
        ...


    这里，宿主机上 `libvirtd` 的 AppArmor 配置文件不允许
    执行 `/usr/libexec/qemu-kvm` 二进制文件。将来
    这有望开箱即用
    （[跟踪问题](https://github.com/kubevirt/kubevirt/issues/8744)），
    但在此之前有几种可行的变通办法。

    第一种（也是最简单的）是从宿主机上移除 libvirt 软件包：
    假定该宿主机是专用的 Kubernetes 节点，
    你很可能本来就不需要它。

    如果你确实需要宿主机上存在 libvirt，则可以向 libvirtd 的
    AppArmor 配置文件
    （通常为 `/etc/apparmor.d/usr.sbin.libvirtd`）添加以下规则：

        # vim /etc/apparmor.d/usr.sbin.libvirtd
        ...
        /usr/libexec/qemu-kvm PUx,
        ...
        # apparmor_parser -r /etc/apparmor.d/usr.sbin.libvirtd # or systemctl reload apparmor.service


-   容器运行时使用的默认 AppArmor 配置文件通常
    会拒绝工作负载的 `mount` 调用。这可能会阻止
    运行带有 [VirtIO-FS](../storage/disks_and_volumes.md#virtio-fs) 的 VM。
    这是一个[已知问题](https://github.com/kubevirt/kubevirt/issues/4290)。
    当前的变通办法是，通过向 VM 或 VMI 对象添加
    以下注解，以 `unconfined` 方式运行此类 VM：

        annotations:
          container.apparmor.security.beta.kubernetes.io/compute: unconfined


### 验证硬件虚拟化支持 {#validate-hardware-virtualization-support}

建议使用支持虚拟化的硬件。你可以使用
virt-host-validate 来确认宿主机能够运行
虚拟化工作负载：

    $ virt-host-validate qemu
      QEMU: Checking for hardware virtualization                                 : PASS
      QEMU: Checking if device /dev/kvm exists                                   : PASS
      QEMU: Checking if device /dev/kvm is accessible                            : PASS
      QEMU: Checking if device /dev/vhost-net exists                             : PASS
      QEMU: Checking if device /dev/net/tun exists                               : PASS
    ...


### 宿主机内核与用户态兼容性 {#host-kernel-and-userland-compatibility}

`virt-launcher` 容器附带基于企业级 Linux（CentOS Stream / RHEL）
的 libvirt 和 QEMU 用户态。不建议在与构建该用户态时所针对的内核
差异很大的宿主机内核上运行此用户态——尤其是
更新的、非企业级 Linux 内核。

宿主机内核暴露的 KVM 和 vhost 接口用于协商
虚拟设备特性（例如 virtio-net 卸载）。如果宿主机内核
提供的特性集合与捆绑用户态所期望的不同，可能导致
设备特性协商不匹配。这些问题可能稍后才显现，例如
表现为实时迁移失败：目标端 QEMU 拒绝在源端
协商得到的设备状态
（参见 [kubevirt/kubevirt#16386](https://github.com/kubevirt/kubevirt/issues/16386)）。

为获得最可预期的行为，请在与 `virt-launcher` 中捆绑的
企业级 Linux 用户态相匹配的宿主机内核上运行节点（例如当前的
CentOS Stream 或 RHEL 版本）。

### SELinux 支持 {#selinux-support}

启用 SELinux 的节点需要安装 [Container-selinux](https://github.com/containers/container-selinux)。最低版本记录在 kubevirt/kubevirt 仓库内的 docs/getting-started.md 中，“SELinux support”一节。

对于未指定 container-selinux 版本的（较旧）发行分支，建议使用 2.170.0 或更新版本。

## 在 Kubernetes 上安装 KubeVirt {#installing-kubevirt-on-kubernetes}

可以使用 KubeVirt operator 安装 KubeVirt，它管理
所有 KubeVirt 核心组件的生命周期。下面是安装
KubeVirt 最新正式版本的示例。它支持在
x86_64 和 Arm64 平台上部署 KubeVirt。

    # Point at latest release
    $ export RELEASE=$(curl https://storage.googleapis.com/kubevirt-prow/release/kubevirt/kubevirt/stable.txt)
    # Deploy the KubeVirt operator
    $ kubectl apply --server-side -f https://github.com/kubevirt/kubevirt/releases/download/${RELEASE}/kubevirt-operator.yaml
    # Create the KubeVirt CR (instance deployment request) which triggers the actual installation
    $ kubectl apply --server-side -f https://github.com/kubevirt/kubevirt/releases/download/${RELEASE}/kubevirt-cr.yaml
    # wait until all KubeVirt components are up
    $ kubectl -n kubevirt wait kv kubevirt --for condition=Available


如果没有硬件虚拟化，则可以按如下方式启用
[软件模拟回退](https://github.com/kubevirt/kubevirt/blob/main/docs/software-emulation.md)，
即在 KubeVirt CR 中将 `spec.configuration.developerConfiguration.useEmulation` 设置为 `true`：

    $ kubectl edit -n kubevirt kubevirt kubevirt


将以下内容添加到 `kubevirt.yaml` 文件中

```yaml
spec:
    ...
    configuration:
        developerConfiguration:
            useEmulation: true
```


> 注意：在 v0.20.0 版本之前，`kubectl wait`
> 命令所用的状况名为 "Ready"，而不是 "Available"

> 注意：在 KubeVirt 0.34.2 之前，安装命名空间中名为 `kubevirt-config` 的 ConfigMap
> 用于配置 KubeVirt。自 0.34.2 起，此方法已
> 弃用。该 ConfigMap 仍然优先于 CR 上存在的 `configuration`，
> 但它不会再接收后续更新，你应当把任何
> 自定义配置迁移到 KubeVirt CR 的 `spec.configuration`。

所有新组件都将部署在 `kubevirt` 命名空间下：

    kubectl get pods -n kubevirt
    NAME                                           READY     STATUS        RESTARTS   AGE
    virt-api-6d4fc3cf8a-b2ere                      1/1       Running       0          1m
    virt-controller-5d9fc8cf8b-n5trt               1/1       Running       0          1m
    virt-handler-vwdjx                             1/1       Running       0          1m
    ...


## 在 OKD 上安装 KubeVirt {#installing-kubevirt-on-okd}

部署 KubeVirt 之前需要添加以下
[SCC](https://docs.openshift.com/container-platform/4.10/authentication/managing-security-context-constraints.html)：

    $ oc adm policy add-scc-to-user privileged -n kubevirt -z kubevirt-operator


授予权限后，即可按上述方式部署 KubeVirt。

### OKD 上的 Web 用户界面 {#web-user-interface-on-okd}

无需额外步骤即可为 KubeVirt 扩展 OKD 的 Web 控制台。

检测到 KubeVirt 部署后，虚拟化扩展会自动启用。

### 通过服务目录以 APB 形式安装 {#from-service-catalog-as-an-apb}

你可以在 OKD 服务目录中找到 KubeVirt，并从
那里安装。为此，请遵循
[KubeVirt APB
仓库](https://github.com/ansibleplaybookbundle/kubevirt-apb)中的文档。

## 在 k3OS 上安装 KubeVirt {#installing-kubevirt-on-k3os}

部署 KubeVirt 之前，需要向所有节点添加以下
[配置](https://github.com/rancher/k3os#configuration)：

```yaml
k3os:
  modules:
  - kvm
  - vhost_net
```


节点使用此配置重启后，即可按上述方式部署 KubeVirt。

## 安装每日开发者构建 {#installing-the-daily-developer-builds}

<!-- markdown-link-check-disable -->
KubeVirt 每天从当前 main 分支发布一次开发者构建。可以通过查看我们的
[nightly-build-jobs](https://prow.ci.kubevirt.io/?job=periodic-kubevirt-push-nightly-build-*)
了解最近一次发布发生的时间。
<!-- markdown-link-check-enable -->

要安装最新的开发者构建，请运行以下命令：

    $ LATEST=$(curl -L https://storage.googleapis.com/kubevirt-prow/devel/nightly/release/kubevirt/kubevirt/latest)
    $ kubectl apply --server-side -f https://storage.googleapis.com/kubevirt-prow/devel/nightly/release/kubevirt/kubevirt/${LATEST}/kubevirt-operator.yaml
    $ kubectl apply --server-side -f https://storage.googleapis.com/kubevirt-prow/devel/nightly/release/kubevirt/kubevirt/${LATEST}/kubevirt-cr.yaml


要查明此构建基于哪一次提交，请运行：

    $ LATEST=$(curl -L https://storage.googleapis.com/kubevirt-prow/devel/nightly/release/kubevirt/kubevirt/latest)
    $ curl https://storage.googleapis.com/kubevirt-prow/devel/nightly/release/kubevirt/kubevirt/${LATEST}/commit
    d358cf085b5a86cc4fa516215f8b757a4e61def2


## 从源码部署 {#deploying-from-source}

参阅[开发者入门
指南](https://github.com/kubevirt/kubevirt/blob/main/docs/getting-started.md)，
了解如何从源码构建并部署 KubeVirt。

## 安装网络插件（可选） {#installing-network-plugins-optional}

KubeVirt 本身不附带任何额外的网络插件，它只是
允许用户使用这些插件。如果要把 VM 连接到多个
网络（Multus CNI），或完全控制 L2（OVS CNI），则需要
部署相应的网络插件。更多信息请参阅
[OVS CNI 安装
指南](https://github.com/kubevirt/ovs-cni/blob/main/docs/deployment-on-arbitrary-cluster.md)。

> 注意：KubeVirt Ansible [网络
> playbook](https://github.com/kubevirt/kubevirt-ansible/tree/master/playbooks#network)
> 默认会安装这些插件。

## 限制 KubeVirt 组件的节点放置 {#restricting-kubevirt-components-node-placement}

你可以通过编辑 KubeVirt CR，限制 KubeVirt 组件在
集群节点上的放置：

- KubeVirt 控制平面组件（virt-controller、virt-api）的放置
  由 KubeVirt CR 中的 `.spec.infra.nodePlacement` 字段控制。
- virt-handler DaemonSet Pod 的放置（因而也包括调度到该集群的
  VM 工作负载的放置）由 KubeVirt CR 中的 `.spec.workloads.nodePlacement`
  字段控制。

对于每个此类 `.nodePlacement` 对象，可以配置 `.affinity`、`.nodeSelector` 和 `.tolerations` 子字段。
有关使用这些字段的更多信息，请参阅
[API 参考](http://kubevirt.io/api-reference/master/definitions.html#_v1_componentconfig)
中的说明。

例如，将 virt-controller 和 virt-api Pod 限制为只在控制平面节点上运行：

    kubectl patch -n kubevirt kubevirt kubevirt --type merge --patch '{"spec": {"infra": {"nodePlacement": {"nodeSelector": {"node-role.kubernetes.io/control-plane": ""}}}}}'


将 virt-handler Pod 限制为只在带有 "region=primary" 标签的节点上运行：

    kubectl patch -n kubevirt kubevirt kubevirt --type merge --patch '{"spec": {"workloads": {"nodePlacement": {"nodeSelector": {"region": "primary"}}}}}'

