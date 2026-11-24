# 实时迁移 {#live-migration}

实时迁移是这样一个过程：正在运行的虚拟机实例移动到另一个计算节点，与此同时客户机工作负载继续运行并保持可访问。

## 启用实时迁移支持 {#enabling-the-live-migration-support}

在较新版本的 KubeVirt 中，实时迁移默认启用。在 v0.56 之前的版本中，必须在功能门控里启用它。KubeVirt CR 中的[功能门控](../cluster_admin/activating_feature_gates.md#how-to-activate-a-feature-gate)字段必须通过添加 `LiveMigration` 来扩展。

## 限制 {#limitations}

- 使用 PersistentVolumeClaim（PVC）的虚拟机必须具备共享的 ReadWriteMany（RWX）访问模式，才能进行实时迁移，除非该卷也正在被迁移（参见[卷迁移](../storage/volume_migration.md)和 `status.migratedVolumes`）。

- 当 Pod 网络绑定为 bridge 接口类型时，不允许实时迁移（[接口和网络](../network/interfaces_and_networks.md#bridge)）。

- 实时迁移要求 virt-launcher Pod 中的端口 `49152, 49153` 可用。如果在[masquerade 接口](../network/interfaces_and_networks.md#masquerade)中显式指定了这些端口，实时迁移将无法工作。

- 实时迁移要求 virt-launcher Pod 的主网络接口在源 Pod 和目标 Pod 上的名称相同。

- 后拷贝实时迁移在大多数环境中需要额外的节点级权限。详情请参阅[后拷贝的节点配置](#node-configuration-for-post-copy)。

- 带有 GPU 的 VM 的实时迁移存在限制。请参阅[使用 vGPU 的实时迁移](#live-migration-with-nvidia-vgpus)。

## 发起实时迁移 {#initiate-live-migration}

实时迁移通过向集群提交一个 VirtualMachineInstanceMigration（VMIM）对象来发起。下面的示例会为虚拟机实例 `vmi-fedora` 启动迁移过程。

```yaml
apiVersion: kubevirt.io/v1
kind: VirtualMachineInstanceMigration
metadata:
  name: migration-job
spec:
  vmiName: vmi-fedora
```

### 使用 virtctl 发起实时迁移 {#using-virtctl-to-initiate-live-migration}

也可以使用 virtctl 发起实时迁移。

```console
    virtctl migrate vmi-fedora
```

## 迁移状态报告 {#migration-status-reporting}

### 状况与迁移方法 {#condition-and-migration-method}

启动虚拟机实例时，还会计算这台机器是否可以实时迁移。结果存放在 VMI 的 `VMI.status.conditions` 中。该计算基于 VMI 的多个参数。迁移不可 LiveMigratable 的 VMI 的请求会被拒绝。

报告的 `Migration Method` 也会在 VMI 启动期间计算出来。`BlockMigration` 表示 VMI 的某些磁盘需要从源复制到目标。`LiveMigration` 表示只会复制实例内存。

```yaml
Status:
  Conditions:
    Status: True
    Type: LiveMigratable
  Migration Method: BlockMigration
```

### 迁移状态 {#migration-status}

迁移进度状态报告在 VMI 的 `VMI.status` 中。最重要的是，它表明迁移是否已经 `Completed`，或者是否 `Failed`。

下面是一次成功迁移的示例。

```
Migration State:
    Completed:                               true
    End Timestamp:                           2019-03-29T03:37:52Z
    Migration Config:
      Completion Timeout Per GiB:            150
      Progress Timeout:                      150
    Migration UID:                           c64d4898-51d3-11e9-b370-525500d15501
    Mode:                                    PreCopy
    Source Node:                             node02
    Source Pod:                              virt-launcher-testvmi-source
    Start Timestamp:                         2019-03-29T03:37:16Z
    Target Direct Migration Node Ports:
      35001:                                 0
      41068:                                 49152
      38284:                                 49153
    Target Node:                             node01
    Target Node Address:                     10.128.0.46
    Target Node Domain Detected:             true
    Target Node Domain Ready Timestamp:      2019-03-29T03:37:51Z
    Target Pod:                              virt-launcher-testvmi-target
```

## 取消实时迁移 {#canceling-a-live-migration}

也可以通过简单地删除迁移对象来取消实时迁移。成功中止的迁移会表明已经请求中止，即 `Abort Requested`，并且中止已成功：`Abort Status: Succeeded`。在这种情况下，迁移会是 `Completed` 并且 `Failed`。

```
Migration State:
    Abort Requested:                         true
    Abort Status:                            Succeeded
    Completed:                               true
    End Timestamp:                           2019-03-29T04:02:49Z
    Failed:                                  true
    Migration Config:
      Completion Timeout Per GiB:            150
      Progress Timeout:                      150
    Migration UID:                           57a693d6-51d7-11e9-b370-525500d15501
    Mode:                                    PreCopy
    Source Node:                             node02
    Source Pod:                              virt-launcher-testvmi-source
    Start Timestamp:                         2019-03-29T04:02:47Z
    Target Direct Migration Node Ports:
      39445:                                 0
      43345:                                 49152
      44222:                                 49153
    Target Node:                             node01
    Target Node Address:                     10.128.0.46
    Target Node Domain Detected:             true
    Target Pod:                              virt-launcher-testvmi-target
```

### 使用 virtctl 取消实时迁移 {#using-virtctl-to-cancel-a-live-migration}

也可以使用 virtctl 取消实时迁移，方法是指定当前正在被迁移的 VMI 的名称。

```console
    virtctl migrate-cancel vmi-fedora
```

## 更改集群范围的迁移配置 {#changing-cluster-wide-migration-configuration}

KubeVirt 设置了一些限制，以免迁移压垮集群。默认情况下，它被配置为只并行运行 `5` 个迁移，另外还限制每个节点最多 `2` 个出站迁移。每次迁移的带宽默认不受限制（`0Mi`）。

你可以在 `kubevirt` CR 中覆盖这些设置以及其他迁移设置。下面的示例展示了当前的默认值：

```yaml
apiVersion: kubevirt.io/v1
kind: KubeVirt
metadata:
  name: kubevirt
  namespace: kubevirt
spec:
  configuration:
    migrations:
      parallelMigrationsPerCluster: 5
      parallelOutboundMigrationsPerNode: 2
      bandwidthPerMigration: 0Mi
      completionTimeoutPerGiB: 150
      progressTimeout: 150
      disableTLS: false
      nodeDrainTaintKey: "kubevirt.io/drain"
      allowAutoConverge: false
      allowPostCopy: false
      unsafeMigrationOverride: false
```

### 配置可接受的工作负载中断级别 {#configuring-acceptable-levels-of-workload-disruption}

**AllowWorkloadDisruption** 参数让管理员能够定义迁移期间可接受的工作负载中断级别，特别是当迁移未能在指定的 `AcceptableCompletionTime` 内收敛时。该时间由 `completionTimeoutPerGiB` 配置以及 VM 的“大小”决定。此选项在确保迁移完成与降低不同迁移策略所带来的风险之间取得平衡。

#### 理解 AllowWorkloadDisruption {#understanding-the-allowworkloaddisruption}

`AllowWorkloadDisruption` 所规定的行为取决于它的配置：

- **禁用时**：
  - 迁移期间工作负载不会被中断。
  - 如果迁移未能在 `AcceptableCompletionTime` 内收敛，它会被取消，并且可能需要重新开始。
  - 这种方式把工作负载中断降到最低，不过，“繁忙”的工作负载可能无法及时完成迁移，从而增加重复迁移的风险。

- **启用时**：
  - 允许迁移控制器中断与长时间运行的迁移相关联的工作负载，以促使迁移完成。
  - 当达到 `AcceptableCompletionTime` 阈值时，迁移控制器会把迁移切换到后拷贝模式。如果不允许后拷贝，迁移控制器则会暂停相应的虚拟机实例，以确保迁移完成。

#### 配置 {#configuration}

要配置 `AllowWorkloadDisruption`：

1. **禁用工作负载中断**：
   - 不设置 `AllowWorkloadDisruption`，或将其显式设置为 `false`。
   - 示例：

    ```yaml
    spec:
      configuration:
        migrations:
         allowWorkloadDisruption: false
     ```

2. **启用工作负载中断**：
   - 将 `AllowWorkloadDisruption` 设置为 `true`，以允许迁移控制器在达到 `AcceptableCompletionTime` 阈值之后采取行动。
   - 要启用后拷贝，请将 `AllowPostCopy` 设置为 `true`。否则，如果更倾向于暂停模式，请不要设置 `AllowPostCopy`，或将其显式设置为 `false`：

    ```yaml
    spec:
      configuration:
        migrations:
         allowWorkloadDisruption: true
         allowPostCopy: true
     ```

#### 关键考虑 {#key-considerations}

- `AllowWorkloadDisruption` 决定迁移控制器能否把完成迁移放在避免工作负载中断之前。
- 启用后拷贝迁移后，如果在后拷贝阶段发生故障，会带来一定的数据丢失风险。
- 在大多数环境中，后拷贝需要额外的节点级权限。详情请参阅[后拷贝的节点配置](#node-configuration-for-post-copy)。
- 暂停工作负载有助于在没有数据丢失风险的情况下完成迁移，但可能导致工作负载暂时处于不活动状态。

请记住，这些配置中的大部分都可以被覆盖，并针对指定的一组 VM 进行微调。更多信息请参阅[迁移策略](../cluster_admin/migration_policies.md)。

## 理解不同的迁移策略 {#understanding-different-migration-strategies}

实时迁移是一个复杂的过程。迁移期间，源 VM 需要把它的全部状态（主要是 RAM）传输到目标 VM。如果有足够的可用资源，例如网络带宽和 CPU 算力，迁移应当能够顺利收敛。如果情况并非如此，迁移可能会卡住，无法继续推进。

从客户机角度看，影响迁移的主要因素是它的 `dirty rate`，也就是 VM 把内存写脏的速率。脏页率高的客户机会在迁移期间导致一场竞态。一方面，内存会持续传输到目标；另一方面，同一块内存又会被客户机写脏。在这类场景下，可以考虑使用更高级的迁移策略。

下面说明目前支持的 3 种迁移策略。

### 预拷贝 {#pre-copy}

预拷贝是默认策略。大多数情况都应当使用它。

它的工作方式如下：

1. 创建目标 VM，但客户机继续在源 VM 上运行。
2. 源开始向目标发送 VM 状态的数据块（主要是内存）。这一过程持续进行，直到全部状态都已传输到目标。
3. 客户机开始在目标 VM 上执行。
4. 源 VM 被移除。

对大多数情况而言，预拷贝是最安全也最快的策略。此外，它可以轻松取消，可以利用多线程，还有更多优点。如果没有真正的理由去使用另一种策略，这绝对是应当采用的策略。

不过，在某些情况下迁移可能不容易收敛。也就是说，当源 VM 状态的一块数据被目标 VM 接收到时，它可能已经被源 VM（即客户机正在其上执行的那台 VM）改写。迁移无法收敛有许多原因，例如脏页率很高，或者网络带宽和 CPU 等资源不足。在这类场景下，请参阅下面的替代策略。

对于显式设置了 CPU 资源限制的虚拟机，多线程迁移会被禁用，改为使用单线程迁移。

### 后拷贝 {#post-copy}

在 KubeVirt 中，迁移总是以预拷贝模式开始，之后可能会切换到后拷贝。一旦进入后拷贝：

1. 客户机在**目标 VM**上运行。
2. 源继续向目标发送 VM 状态的数据块（主要是内存）。
3. 当正在目标 VM 上运行的客户机要访问内存时：
    1. 如果这块内存已经存在于目标 VM 上，客户机就可以访问它。
    2. 否则，目标 VM 会向源 VM 请求一块内存。
4. 一旦全部内存状态都已在目标 VM 上更新完毕，就会移除源 VM。

后拷贝的主要思想是：客户机在目标 VM 上运行，同时按需获取剩余内存。这种方法既有优点也有缺点：

<u>优点</u>：

* 同一内存块永远不会被传输两次。这之所以可能，是因为使用后拷贝时，某一页曾经被写脏并不重要，因为客户机已经在目标 VM 上运行。
  * 这意味着高脏页率的影响要小得多。
* 消耗的网络带宽更少。

<u>缺点</u>：

* 使用后拷贝时，VM 状态没有单一的事实来源。当客户机（在目标 VM 上运行）写入内存时，这块内存是客户机状态的一部分，但状态的另一些部分可能仍然只在源 VM 上得到更新。这种情况通常很危险，例如，如果目标 VM 或客户机 VM 崩溃，状态就无法恢复。
* 预热缓慢：客户机开始执行时，目标 VM 上还不存在任何内存。因此，客户机不得不在很短的时间内等待大量内存。
* 在大多数情况下比预拷贝更慢。
* 更难取消迁移。

#### 后拷贝的节点配置 {#node-configuration-for-post-copy}

后拷贝迁移在目标节点上使用 `userfaultfd` 系统调用，以便按需从源获取内存页。由于 QEMU 进程以非特权方式运行，根据你的环境，可能需要额外的权限。

##### 内核 sysctl {#kernel-sysctl}

每一台可能接收后拷贝迁移的节点都必须启用以下 sysctl：

```
vm.unprivileged_userfaultfd=1
```

受支持版本的 OpenShift/OKD 已经通过 Machine Config Operator 设置了此 sysctl，因此在那里无需任何操作。在其他 Kubernetes 发行版上，请把这一行写入诸如 `/etc/sysctl.d/99-postcopy.conf` 的文件来持久化该设置，可以手动进行，也可以使用集群的节点配置工具（例如 cloud-init、Ansible 或 DaemonSet）。

!!! warning
    启用 `vm.unprivileged_userfaultfd` 会允许 VM 进程使用 `userfaultfd` 系统调用。一些经过安全加固的内核默认会禁用它。

##### Seccomp {#seccomp}

在强制执行 seccomp 的集群上，CRI-O 等容器运行时默认可能会阻止 `userfaultfd`。`KubevirtSeccompProfile` [功能门控](../cluster_admin/activating_feature_gates.md#how-to-activate-a-feature-gate)会安装一个允许此系统调用的 seccomp 配置文件。此功能门控在 KubeVirt v1.7 达到 Beta，但 Beta 功能门控从 v1.9 起才默认启用——在更早的版本上必须显式启用它。

除功能门控之外，还必须配置 KubeVirt CR 以使用该自定义配置文件：

```yaml
apiVersion: kubevirt.io/v1
kind: KubeVirt
metadata:
  name: kubevirt
  namespace: kubevirt
spec:
  configuration:
    seccompConfiguration:
      virtualMachineInstanceProfile:
        customProfile:
          localhostProfile: kubevirt/kubevirt.json
```

##### SELinux {#selinux}

在 SELinux 处于 enforcing 模式的节点上，virt-launcher 进程可能会被拒绝 `userfaultfd`，具体取决于正在使用的 `container_t` 策略。运行 `container-selinux` v2.248 或更高版本的节点已经包含所需权限（`kernel_userfaultfd_use(container_domain)`）。在更早的版本上，管理员可能需要创建一个自定义 SELinux 策略模块，以便为相关上下文允许此系统调用。

### 自动收敛 {#auto-converge}

自动收敛是一种帮助预拷贝迁移更快收敛的技术，同时不改变迁移工作方式的核心算法。

由于高脏页率通常是迁移无法收敛的最重要因素，自动收敛只是对客户机的 CPU 进行节流。如果迁移能够足够快地收敛，客户机的 CPU 不会被节流，或者只会被忽略不计地节流。但是，如果迁移无法足够快地收敛，随着时间推移，CPU 会被越来越多地节流。

这项技术会大幅提高迁移最终得以收敛的概率。

## 使用不同的网络进行迁移 {#using-a-different-network-for-migrations}

可以把实时迁移配置为发生在 Kubernetes 所配置使用的网络之外的另一个网络上。根据使用场景，这有可能带来更强的确定性、控制力以及/或者带宽。

### 在集群上创建迁移网络 {#creating-a-migration-network-on-a-cluster}

需要一张独立的物理网络。这意味着集群上的每个节点至少要有 2 块 NIC，并且将用于迁移的 NIC 必须互连，也就是全部接到同一台交换机上。下面的示例假定将使用 `eth1` 进行迁移。

Kubernetes 集群还必须安装 [multus](https://github.com/k8snetworkplumbingwg/multus-cni.git)。

如果期望的网络不包含 DHCP 服务器，那么还需要 [whereabouts](https://github.com/k8snetworkplumbingwg/whereabouts)。

最后，需要在安装 KubeVirt 的命名空间中创建一个 NetworkAttachmentDefinition。下面是一个示例：

```yaml
apiVersion: "k8s.cni.cncf.io/v1"
kind: NetworkAttachmentDefinition
metadata:
  name: migration-network
  namespace: kubevirt
spec:
  config: '{
      "cniVersion": "0.3.1",
      "name": "migration-bridge",
      "type": "macvlan",
      "master": "eth1",
      "mode": "bridge",
      "ipam": {
        "type": "whereabouts",
        "range": "10.1.1.0/24"
      }
    }'
```

### 配置 KubeVirt 通过该网络迁移 VMI {#configuring-kubevirt-to-migrate-vmis-over-that-network}

只需把 NetworkAttachmentDefinition 的名称添加到 KubeVirt CR 中，如下所示：

```yaml
apiVersion: kubevirt.io/v1
kind: KubeVirt
metadata:
  name: kubevirt
  namespace: kubevirt
spec:
  configuration:
    migrations:
      network: migration-network
```

这一更改会触发 virt-handler Pod 重启，因为它们会连接到这个新网络。

从此以后，迁移将通过该网络进行。

### 配置 KubeVirtCI 以测试迁移网络 {#configuring-kubevirtci-for-testing-migration-networks}

想在部署到真实集群之前测试此功能的开发者和其他人，可能希望在 KubeVirtCI 中配置一张专用的迁移网络。

KubeVirtCI 可以简单地被配置为包含一个虚拟的辅助网络，并自动安装 multus 和 whereabouts。只需在运行 `make cluster-up` 之前声明以下环境变量：

```
export KUBEVIRT_NUM_NODES=2;
export KUBEVIRT_NUM_SECONDARY_NICS=1;
export KUBEVIRT_DEPLOY_ISTIO=true;
export KUBEVIRT_WITH_CNAO=true
```

## 迁移超时 {#migration-timeouts}

根据类型，实时迁移过程会把虚拟机内存页和磁盘块复制到目标。在此过程中，未锁定的页和块会被复制，并重新变为可供该实例再次使用的空闲资源。要成功完成迁移，这里假定实例写入这些空闲页和块（污染这些页）的速率低于它们被复制的速率。

### 完成时间 {#completion-time}

在某些情况下，虚拟机的脏页率可能很高，这意味着它写入不同内存页 / 磁盘块的速率高于这些页和块能够被复制过去的速率。这种情况会阻止迁移过程在合理的时间内完成。

在这种情况下，可以定义一个超时，使得实时迁移如果运行了很长一段时间，就会被中止，或者切换到后拷贝模式（如果已启用后拷贝）。

超时根据 VMI 的大小、它的内存以及需要被复制的临时磁盘来计算。可配置参数 `completionTimeoutPerGiB` 默认为 150s，表示每 GiB 数据在迁移被中止 / 切换到后拷贝模式之前所允许的最长时间。例如，使用默认值时，一台内存为 8GiB 的 VMI 会在 1200 秒后超时。

### 进度超时 {#progress-timeout}

当实时迁移发现迁移中的“剩余字节”没有任何进展时，也会中止。等待实时迁移在传输数据上取得进展的时间可以通过 `progressTimeout` 参数配置，默认值为 150s。由于只要产生了*任何*进展，此超时就会重置，因此该机制不会让剩余字节来回波动的迁移超时，它只对完全卡住的迁移有用。

### 停滞检测 {#stall-detection}

**功能状态：** KubeVirt v1.9（Alpha）

繁忙的客户机更常常因为剩余字节进入平台期或来回振荡而无法收敛，而不是因为传输完全卡住。停滞检测覆盖的就是这种情况：一旦剩余字节不再产生*净进展*，它就会在接近局部最小值的位置把迁移切换过去，而不是一直等待 `completionTimeoutPerGiB`。此功能处于 Alpha 阶段，必须通过 `MigrationStallDetection` 功能门控启用。

```yaml
apiVersion: kubevirt.io/v1
kind: KubeVirt
metadata:
  name: kubevirt
  namespace: kubevirt
spec:
  configuration:
    developerConfiguration:
      featureGates:
        - MigrationStallDetection
```

与停滞检测相关的重要配置选项有两个：

- `maxDowntimeMs`（默认 900）：你的工作负载可接受的最大停机时间，单位为毫秒。只有当 `allowWorkloadDisruption` 为 false（默认值）时才有意义。停滞检测器会尝试获得比这个值更好的停机时间；如果连这个预算也无法满足，迁移就会中止。可以在 KubeVirt CR（[MigrationConfiguration](https://kubevirt.io/api-reference/main/definitions.html#_v1_migrationconfiguration)）以及 [MigrationPolicy](../cluster_admin/migration_policies.md)（[MigrationPolicySpec](https://kubevirt.io/api-reference/main/definitions.html#_v1alpha1_migrationpolicyspec)）上配置。
- `stallProgressTimeout`（默认 40）：剩余字节可以停留在平台期多少秒，之后迁移才被视为停滞。较大的值会花更多时间寻找更好的切换点（迁移更长，停机时间通常更好）；较小的值则相反。在 Alpha 阶段，它位于 `MigrationPolicy.spec.experimental.stallDetector`（[ExperimentalMigrationOptions](https://kubevirt.io/api-reference/main/definitions.html#_v1_experimentalmigrationoptions)）之下。待该功能毕业时，预计会改用现有的 `progressTimeout` 字段。

其他停滞检测器选项在 Alpha 阶段也暴露在 `MigrationPolicy.spec.experimental.stallDetector` 之下；完整列表请参阅 [StallDetectorOptions](https://kubevirt.io/api-reference/main/definitions.html#_v1_stalldetectoroptions)。目前预计只有 `stallProgressTimeout` 会毕业。其余选项被暴露出来，是为了让社区在默认值不适合自己时尝试不同的值，并反馈是否真的有必要继续暴露其中任何一个选项。其余大多数字段技术上很密集，并且与具体算法相关；关于它们各自的作用，请参阅 [StallDetectorOptions](https://kubevirt.io/api-reference/main/definitions.html#_v1_stalldetectoroptions) API 参考或 [VEP 248](https://github.com/kubevirt/enhancements/tree/main/veps/sig-compute/248-migration-convergence)。

`experimental.stallDetector` 和 `experimental.downtimeTuning` 不能在同一条策略上使用。

### 停机时间调优 {#downtime-tuning}

**功能状态：** KubeVirt v1.10（Alpha）

脏页率适中的繁忙客户机常常能让迁移取得进展，但永远无法收敛：QEMU 估计的切换时间一直徘徊在当前 `max_downtime` 刚刚之上，于是迁移不断迭代，直到 `completionTimeoutPerGiB` 触发。停机时间调优会随着内存复制迭代的推进逐步提高 `max_downtime`，在现有超时逻辑触发后拷贝、暂停或中止之前，给 QEMU 更多的切换空间。

此功能处于 Alpha 阶段，按工作负载通过 [MigrationPolicy](../cluster_admin/migration_policies.md) 选择启用，并且必须通过 `MigrationDowntimeTuning` 功能门控启用。

```yaml
apiVersion: kubevirt.io/v1
kind: KubeVirt
metadata:
  name: kubevirt
  namespace: kubevirt
spec:
  configuration:
    developerConfiguration:
      featureGates:
        - MigrationDowntimeTuning
```

通过设置 `spec.experimental.downtimeTuning`，在匹配的 `MigrationPolicy` 上激活调优。空对象会以内置默认值启用该算法。也可以在同一条策略上设置 `maxDowntimeMs`，作为爬升所瞄准的目标切换预算（默认 900）。`experimental.stallDetector` 和 `experimental.downtimeTuning` 不能在同一条策略上使用。

```yaml
apiVersion: migrations.kubevirt.io/v1alpha1
kind: MigrationPolicy
metadata:
  name: tune-downtime
spec:
  selectors:
    namespaceSelector:
      workload-type: memory-intensive
  maxDowntimeMs: 5000
  experimental:
    downtimeTuning: {}
```

有两个重要的配置选项：

- `maxDowntimeMs`（默认 900）：目标最大客户机切换暂停时间，单位为毫秒。算法以相等的步长把 QEMU 的 `max_downtime` 向这个值爬升；它是调优目标，并不是对观测到的停机时间的硬性保证。可以在 KubeVirt CR（[MigrationConfiguration](https://kubevirt.io/api-reference/main/definitions.html#_v1_migrationconfiguration)）以及 [MigrationPolicy](../cluster_admin/migration_policies.md)（[MigrationPolicySpec](https://kubevirt.io/api-reference/main/definitions.html#_v1alpha1_migrationpolicyspec)）上配置。
- `downtimeTuning`（通过字段存在来激活；默认值见下文）：控制迁移期间提升 `max_downtime` 的激进程度。在 Alpha 阶段，它位于 `MigrationPolicy.spec.experimental`（[DowntimeTuningOptions](https://kubevirt.io/api-reference/main/definitions.html#_v1_downtimetuningoptions)）之下。

当使用 `downtimeTuning: {}` 时，默认值为 `initialMs: 150`、`steps: 7`、`startAfterIteration: 3`、`cooldownSeconds: 10`；各选项的作用请参阅 [DowntimeTuningOptions](https://kubevirt.io/api-reference/main/definitions.html#_v1_downtimetuningoptions)。

更多细节请参阅 [VEP 291](https://github.com/kubevirt/enhancements/tree/main/veps/sig-compute/291-dynamic-downtime-tuning)。

### 压缩 {#compression}

**功能状态：** KubeVirt v1.9（Alpha）

大内存 VM 或带宽受限的环境常常会碰到迁移超时，因为客户机脏页率超过了可用链路容量。压缩通过对迁移流上的内存页进行压缩来减少经网络发送的数据量。这是用源节点上适度的 CPU 开销换取更低的带宽占用，并且可以帮助那些否则会超时的迁移收敛。此功能处于 Alpha 阶段，并按工作负载通过 [MigrationPolicy](../cluster_admin/migration_policies.md) 选择启用。

把 `spec.experimental.compression` 设置为 `zstd`，用在匹配目标 VM 的 `MigrationPolicy` 上。省略该字段，或将其设置为 `none`，即可让压缩保持禁用（这是默认行为）。

```yaml
apiVersion: migrations.kubevirt.io/v1alpha1
kind: MigrationPolicy
metadata:
  name: compress-heavy-vms
spec:
  selectors:
    namespaceSelector:
      workload-type: memory-intensive
  experimental:
    compression: zstd
```

只有一个配置选项：

- `compression`（默认值：`none`）：设置为 `zstd` 时，virt-launcher 会在实时迁移数据流上启用 QEMU/libvirt zstd 压缩。在 Alpha 阶段，它位于 `MigrationPolicy.spec.experimental`（[ExperimentalMigrationOptions](https://kubevirt.io/api-reference/main/definitions.html#_v1_experimentalmigrationoptions)）之下。目前只支持 `zstd`；`none` 会为匹配到的 VM 显式禁用压缩。

压缩处于活动状态时，virt-launcher 的迁移日志会包含 `Migration compression enabled: method=zstd`，并在迁移进度行中包含 `CompressionRatio` 值。压缩与 multifd 并行通道（未设置每次迁移 CPU 限制时的默认方式）、TLS、带宽限制、自动收敛、后拷贝以及停滞检测兼容。设计细节请参阅 [VEP 246](https://github.com/kubevirt/enhancements/tree/main/veps/sig-compute/246-migration-compression)。

## 禁用安全迁移 {#disabling-secure-migrations}

**功能状态：** KubeVirt v0.43

有时为了提高性能，可能希望禁用迁移的 TLS 加密。使用 `disableTLS` 来做到这一点：

```yaml
apiVersion: kubevirt.io/v1
kind: KubeVirt
metadata:
  name: kubevirt
  namespace: kubevirt
spec:
  configuration:
    migrations:
      disableTLS: true
```

**注意：** 这样做虽然能提高性能，但可能允许 MITM 攻击。请务必小心。

## 使用（NVIDIA）vGPU 进行实时迁移 {#live-migration-with-nvidia-vgpus}

**功能状态：** KubeVirt v1.9（Alpha）

KubeVirt 可以对拥有单个 NVIDIA vGPU、并以中介设备（mdev）形式暴露的 VirtualMachineInstance 进行实时迁移。

### 前提条件 {#prerequisites}

在由 vGPU 支持的 VMI 能够实时迁移之前，请完成以下全部事项：

1. 在每一台可能运行或接收该 VM 的节点上安装 NVIDIA Virtual GPU Manager（宿主机驱动）。NVIDIA 要求在节点上存在任何中介设备之前先安装此驱动。你可以自行安装，也可以使用 NVIDIA GPU Operator 安装。
2. 在这些节点上创建中介设备实例。厂商驱动已经安装之后，只选择**一个**创建者，以免它们互相冲突：
   - KubeVirt 的[中介设备配置](../compute/mediated_devices_configuration.md)，它会根据驱动所暴露的类型创建 mdev。
   - NVIDIA GPU Operator 的 [vGPU Device Manager](https://docs.nvidia.com/datacenter/cloud-native/gpu-operator/latest/gpu-operator-kubevirt.html)，或其他会创建 mdev 的厂商工具。如果由外部插件负责创建，请在这些节点上禁用 KubeVirt 的 mdev 管理（`spec.configuration.mediatedDevicesConfiguration.enabled: false`）。
3. 按照[主机设备分配](../compute/host-devices.md#listing-permitted-devices)中的说明，在 `permittedHostDevices.mediatedDevices` 下将该 mdev 类型加入允许列表，并把它分配给 VM。如果 NVIDIA GPU Operator（或其他插件）通告了该资源，请设置 `externalResourceProvider: true`。PCI GPU 直通不可迁移；该设备必须以中介设备的形式出现。
4. 启用 `VGPULiveMigration` 功能门控（见下文）。

源宿主机和目标宿主机还必须满足 [NVIDIA 的 vGPU 迁移要求](https://docs.nvidia.com/vgpu/latest/grid-vgpu-release-notes-red-hat-el-kvm/validated-platforms.html#red-hat-el-vgpu-migration-support)。在 Alpha 阶段，KubeVirt 在选择目标时不会强制这些约束。

### 启用 vGPU 实时迁移 {#enabling-vgpu-live-migration}

在 KubeVirt CR 中启用 `VGPULiveMigration` 功能门控：

```yaml
apiVersion: kubevirt.io/v1
kind: KubeVirt
metadata:
  name: kubevirt
  namespace: kubevirt
spec:
  configuration:
    developerConfiguration:
      featureGates:
        - VGPULiveMigration
```

启用之后，请像迁移任何其他可实时迁移的 VM 一样迁移该 VMI（例如使用 `VirtualMachineInstanceMigration` 对象或 `virtctl migrate`）。

### 限制与要求 {#limitations-and-requirements}

Alpha 支持有意保持狭窄。以下限制适用：

- **仅限单个 NVIDIA mdev vGPU。** VMI 必须恰好请求一个作为中介设备的 GPU。以下情况不支持实时迁移：
  - PCI GPU 直通
  - SR-IOV vGPU
  - 拥有多个 vGPU 的 VM（计划在 Beta 中支持）
- **兼容的客户机操作系统。** 客户机 VM 上运行的操作系统必须运行[受支持的 Windows 版本](https://docs.nvidia.com/vgpu/latest/grid-vgpu-release-notes-red-hat-el-kvm/validated-platforms.html#red-hat-el-windows-support)或[受支持的 Linux 版本](https://docs.nvidia.com/vgpu/latest/grid-vgpu-release-notes-red-hat-el-kvm/validated-platforms.html#red-hat-el-linux-support)。
- **兼容的客户机驱动。** 客户机上运行的 NVIDIA 驱动与宿主机 Virtual GPU Manager [必须兼容](https://docs.nvidia.com/vgpu/latest/grid-vgpu-release-notes-red-hat-el-kvm/release-notes.html)。
- **NVIDIA 客户机功能限制。** 如果客户机启用了 CUDA 统一内存、调试器或性能分析器，迁移就会失败。完整的厂商要求请参阅[前提条件](#prerequisites)中的 NVIDIA 链接。
- **不支持后拷贝。** 带有 vGPU 的 VM 不能使用后拷贝实时迁移。
- **收敛与停机时间。** 即使是空闲的、带有 vGPU 的客户机也有基线内存脏页率，因此预拷贝通常无法达到 QEMU 默认约 300 ms 的停机时间目标。实践中，大多数 vGPU 迁移因此需要 [`allowWorkloadDisruption`](#configuring-acceptable-levels-of-workload-disruption)（例如通过[迁移策略](../cluster_admin/migration_policies.md)）。由于 vGPU 无法使用后拷贝，这里的中断意味着暂停客户机，直到迁移完成。

  在这条默认路径上，切换仍然要等待迁移完成超时（`completionTimeoutPerGiB` 以及 VM 大小），这可能使迁移在暂停之前运行很长时间。

  启用[停滞检测](#stall-detection)（同样处于 Alpha）会改变这一行为：一旦剩余字节进入平台期，KubeVirt 就可以更早切换，而不必把完成超时等完。停滞检测还会暴露 `maxDowntimeMs`，让你可以让 `allowWorkloadDisruption` 保持为 false，同时仍然通过把停机时间预算提高到你所选择的最大可接受停机时间来实现收敛；如果连这个上限也无法满足，迁移就会中止。
