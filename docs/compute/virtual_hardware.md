# 虚拟硬件 {#virtual-hardware}

微调与设备无关的硬件的不同方面（BIOS、主板等）有时是必要的，这样才能让客户机操作系统正常启动和重启。

## 机器类型 {#machine-type}

QEMU 能够使用两类 x86\_64 芯片组，即所谓的机器类型。x86\_64 芯片组是 i440fx（也称为 pc）和 q35。它们按照 qemu-system-${ARCH} 进行版本划分，格式为 `pc-${machine_type}-${qemu_version}`，例如 `pc-i440fx-2.10` 和 `pc-q35-2.10`。

KubeVirt 默认使用 QEMU 最新的 q35 机器类型。如果需要自定义机器类型，可以通过以下结构进行配置：

```yaml
metadata:
  name: myvmi
spec:
  domain:
    machine:
      # This value indicates QEMU machine type.
      type: pc-q35-2.10
    resources:
      requests:
        memory: 512M
    devices:
      disks:
      - name: myimage
        disk: {}
  volumes:
    - name: myimage
      persistentVolumeClaim:
        claimName: myclaim
```

这两种机器类型内部结构的比较可以在 [QEMU wiki](https://wiki.qemu.org/Features/Q35) 找到。

### BIOS/UEFI {#biosuefi}

所有虚拟机默认都使用 BIOS 启动。

可以通过 `spec.firmware.bootloader` 设置一个值来使用 UEFI/OVMF：

```yaml
apiVersion: kubevirt.io/v1
kind: VirtualMachineInstance
metadata:
  labels:
    special: vmi-alpine-efi
  name: vmi-alpine-efi
spec:
  domain:
    devices:
      disks:
      - disk:
          bus: virtio
        name: containerdisk
    features:
      smm:
        enabled: true
    firmware:
      # this sets the bootloader type
      bootloader:
        efi: {}
```

启用 EFI 会自动启用[安全启动](https://docs.microsoft.com/en-us/windows-hardware/design/device-experiences/oem-secure-boot)，除非把 `secureBoot` 字段（位于 `efi` 之下）设置为 `false`。

安全启动本身要求像上面那样启用 [SMM](https://en.wikipedia.org/wiki/System_Management_Mode) CPU 特性。出于安全原因，这不会自动发生。

### SMBIOS 固件 {#smbios-firmware}

为了向客户机操作系统提供一致的虚拟化硬件视图，可以通过 `spec.firmware.uuid` 把 SMBIOS UUID 设置为一个常量：

```yaml
metadata:
  name: myvmi
spec:
  domain:
    firmware:
      # this sets the UUID
      uuid: 5d307ca9-b3ef-428c-8861-06e72d69f223
      serial: e4686d2c-6e8d-4335-b8fd-81bee22f4815
    resources:
      requests:
        memory: 512M
    devices:
      disks:
      - name: myimage
        disk: {}
  volumes:
    - name: myimage
      persistentVolumeClaim:
        claimName: myclaim
```

此外，可以像上面演示的那样，通过 `spec.firmware.serial` 把 SMBIOS 序列号设置为一个常量。

### CPU {#cpu}

**注意**：这与调度决策或资源分配无关。

#### 拓扑 {#topology}

可以通过 `spec.domain.cpu.cores` 设置 CPU 核心数。下面的 VM 将拥有一个具有 `3` 个核心的 CPU：

```yaml
metadata:
  name: myvmi
spec:
  domain:
    cpu:
      # this sets the cores
      cores: 3
    resources:
      requests:
        memory: 512M
    devices:
      disks:
      - name: myimage
        disk: {}
  volumes:
    - name: myimage
      persistentVolumeClaim:
        claimName: myclaim
```

#### 使用 CPU 模型、CPU 特性和机器类型为节点打标签 {#labeling-nodes-with-cpu-models-cpu-features-and-machine-types}

KubeVirt 可以根据 VM 的 CPU 模型、特性和机器类型创建节点选择器。借助这些节点选择器，VM 会被调度到支持相匹配的 VM CPU 模型、特性和机器类型的节点上。

要正确地为节点打标签，用户可以使用 KubeVirt Node-labeller，它会创建全部必要的标签，用户也可以自己创建节点标签。

KubeVirt node-labeller 会创建 4 类标签：CPU 模型、CPU 特性、KVM 信息和机器类型。它使用 libvirt 获取宿主机上所有受支持的 CPU 模型和 CPU 特性，然后 Node-labeller 根据 CPU 模型和机器类型创建标签。

Node-labeller 支持过时 CPU 模型列表，以及用于特性的最小基线 CPU 模型。这两项都可以通过 KubeVirt CR 设置：

```yaml
apiVersion: kubevirt.io/v1
kind: Kubevirt
metadata:
  name: kubevirt
  namespace: kubevirt
spec:
  ...
  configuration:
    obsoleteCPUModels:
      486: true
      pentium: true
...
```

过时的 CPU 不会被写入标签。如果 KubeVirt CR 不包含 `obsoleteCPUModels` 变量，Labeller 会设置默认值（“pentium、pentium2、pentium3、pentiumpro、coreduo、n270、core2duo、Conroe、athlon、phenom、kvm32、kvm64、qemu32 和 qemu64”）。

用户可以通过在配置映射中添加 / 移除 CPU 模型来更改 obsoleteCPUModels。随后 KubeVirt 会用新标签更新节点。

对于同构集群 / 没有启用实时迁移的集群，可以通过向节点添加以下注解，来禁用节点标签器，并避免向节点添加标签：

`node-labeller.kubevirt.io/skip-node`。

#### 模型 {#model}

**注意**：如果没有定义 CPU 模型，VM 将拥有最接近 VM 运行所在节点上所用模型的 CPU 模型。

**注意**：CPU 模型区分大小写。

可以通过 `spec.domain.cpu.model` 设置 CPU 模型。下面的 VM 将拥有 `Conroe` 模型的 CPU：

```yaml
apiVersion: kubevirt.io/v1
kind: VirtualMachineInstance
metadata:
  name: myvmi
spec:
  domain:
    cpu:
      # this sets the CPU model
      model: Conroe
...
```

你可以在[此处](https://github.com/libvirt/libvirt/blob/master/src/cpu_map/index.xml)查看可用模型列表。

当启用 CPUNodeDiscovery 功能门控并且 VM 具有 CPU 模型时，KubeVirt 会创建格式为 `cpu-model.node.kubevirt.io/<cpuModel>` 的节点选择器，例如 `cpu-model.node.kubevirt.io/Conroe`。当 VM 没有 CPU 模型时，就不会创建节点选择器。

#### 启用默认的集群 CPU 模型 {#enabling-default-cluster-cpu-model}

要启用默认 CPU 模型，用户可以在 KubeVirt CR 中添加 `cpuModel` 字段。

```yaml
apiVersion: kubevirt.io/v1
kind: KubeVirt
metadata:
  name: kubevirt
  namespace: kubevirt
spec:
  ...
  configuration:
    cpuModel: "EPYC"
...
```

当 vmi 没有任何 CPU 模型时，会设置默认 CPU 模型。当 vmi 设置了 CPU 模型时，优先使用 vmi 的 CPU 模型。当默认 CPU 模型没有设置，并且 vmi 的 CPU 模型也没有设置时，会设置 `host-model`。可以在 KubeVirt 运行期间更改默认 CPU 模型。当启用 CPUNodeDiscovery 功能门控时，KubeVirt 会用默认 CPU 模型创建节点选择器。

#### CPU 模型的特殊情况 {#cpu-model-special-cases}

作为特殊情况，可以把 `spec.domain.cpu.model` 设置为：-
`host-passthrough`，以便把节点上的 CPU 直通到 VM

    metadata:
      name: myvmi
    spec:
      domain:
        cpu:
          # this passthrough the node CPU to the VM
          model: host-passthrough
    ...

-   `host-model`，以便让 VM 上的 CPU 接近节点上的 CPU

<!-- -->

    metadata:
      name: myvmi
    spec:
      domain:
        cpu:
          # this set the VM CPU close to the node one
          model: host-model
    ...

更多细节请参阅 [CPU API 参考](https://libvirt.org/formatdomain.html#elementsCPU)。

#### CPU 模型与实时迁移 {#cpu-model-and-live-migration}

CPU 模型决定了与潜在实时迁移目标的 CPU 兼容性。KubeVirt 会根据该模型，向迁移目标 Pod 添加节点选择器：

| CPU 模型 | 迁移目标 Pod 上的节点选择器 |
|---|---|
| 具名模型，例如 `Skylake-Client-v4` | `cpu-model.node.kubevirt.io/<model>` 和 `cpu-vendor.node.kubevirt.io/<vendor>` |
| `host-model` | `cpu-model-migration.node.kubevirt.io/<host model of the source node>`，外加 `cpu-feature.node.kubevirt.io/<feature>`，对应源节点的每一个 `host-model-required-features.node.kubevirt.io/<feature>` 标签，再加上厂商 |
| `host-passthrough` | 仅 `cpu-vendor.node.kubevirt.io/<vendor>` |

这些标签来自 node-labeller（参见[使用 CPU 模型、CPU 特性和机器类型为节点打标签](#labeling-nodes-with-cpu-models-cpu-features-and-machine-types)）。对于已经迁移过的 `host-model` VMI，KubeVirt 会复用当前 Pod 的 CPU 选择器，而不是再次读取节点标签，因此该 VMI 仍然可以移回它启动时所在的节点。

请记住以下几点：

- `LiveMigratable` 状况并不会验证是否有另一个节点满足该 VMI 的 CPU 模型和特性要求。即使没有任何其他节点满足这些要求，VMI 也可以报告 `LiveMigratable: True`。在这种情况下，目标 Pod 会保持 `Pending`，并带有一条 `didn't match Pod's node affinity/selector` 调度消息。对于 `host-model` VMI，当没有其他节点拥有所需标签时，KubeVirt 还会在该 VMI 上记录一个 `NoSuitableNodesForHostModelMigration` 警告事件。默认情况下，如果目标 Pod 在创建之后五分钟仍然无法调度，迁移就会失败。
- 当 VMI 和集群都没有设置模型时，会使用 `host-model`（集群侧对应 `spec.configuration.cpuModel`）。两个节点可以报告相同的宿主机模型名称，所需特性仍然可以不同。例如，如果源节点带有标签 `host-model-required-features.node.kubevirt.io/rfds-no`，而另一个节点没有 `cpu-feature.node.kubevirt.io/rfds-no` 标签，那么在源节点上启动的 `host-model` VMI 就不能迁移到那个节点。要识别兼容的目标，请检查每个候选节点是否拥有与迁移目标 Pod 的 `cpu-model-migration.node.kubevirt.io/<model>`、`cpu-feature.node.kubevirt.io/<feature>` 和 `cpu-vendor.node.kubevirt.io/<vendor>` 选择器相匹配的标签。对于已经迁移过的 VMI，请使用它当前 Pod 的 CPU 选择器，而不是根据它当前节点的 host-model 标签推导出新的要求。
- `host-passthrough` 只检查 CPU 厂商，因此迁移到一块不同的 CPU 所在的节点可能成功，而客户机仍然会看到新宿主机并不具备的 CPU 特性。只有当源节点和目标节点拥有完全相同的 CPU 时，才使用 `host-passthrough`。
- 要在 CPU 不同的节点之间迁移，请设置一个这些节点全都支持的具名模型（它们全都带有 `cpu-model.node.kubevirt.io/<model>` 标签），可以按 VMI 设置，也可以作为集群默认值（参见[启用默认的集群 CPU 模型](#enabling-default-cluster-cpu-model)）。KubeVirt 不会校验模型名称，因此如果没有任何节点支持某个模型，带有该模型的 VMI 会停留在 `Scheduling`（[kubevirt/kubevirt#16854](https://github.com/kubevirt/kubevirt/issues/16854)）。

### 特性 {#features}

可以通过 `spec.domain.cpu.features` 设置 CPU 特性，其中可以包含零个或多个 CPU 特性：

    metadata:
      name: myvmi
    spec:
      domain:
        cpu:
          # this sets the CPU features
          features:
          # this is the feature's name
          - name: "apic"
          # this is the feature's policy
           policy: "require"
    ...

**注意**：policy 属性可以省略，也可以包含以下策略之一：force、require、optional、disable、forbid。

**注意**：如果某个特性省略了 policy，则默认为 **require**。

根据策略的行为：

-   在创建虚拟机期间，所有策略都会传递给 libvirt。
-   如果启用了功能门控 “CPUNodeDiscovery”，并且省略了 policy，或者 policy 的值为 “require”，那么虚拟机只能被调度到支持该特性的节点上。
-   如果启用了功能门控 “CPUNodeDiscovery”，并且 policy 的值为 “forbid”，那么虚拟机**不会**被调度到支持该特性的节点上。

关于特性和策略的完整说明可以在[此处](https://libvirt.org/formatdomain.html#elementsCPU)找到。

当启用 CPUNodeDiscovery 功能门控时，KubeVirt 会根据 CPU 特性创建节点选择器，格式为 `cpu-feature.node.kubevirt.io/<cpuFeature>`，例如 `cpu-feature.node.kubevirt.io/apic`。当 VM 没有 CPU 特性时，就不会创建节点选择器。

### 时钟 {#clock}

#### 客户机时间 {#guest-time}

把 VM 内部的虚拟化硬件时钟设置为某个特定时间。可用选项有

-   **utc**

-   **timezone**

全部可能的配置选项请参阅[时钟 API 参考](https://kubevirt.github.io/api-reference/master/definitions.html#_v1_clock)。

#### utc {#utc}

如果指定了 `utc`，VM 的时钟将被设置为 UTC。

    metadata:
      name: myvmi
    spec:
      domain:
        clock:
          utc: {}
        resources:
          requests:
            memory: 512M
        devices:
          disks:
          - name: myimage
            disk: {}
      volumes:
        - name: myimage
          persistentVolumeClaim:
            claimName: myclaim

#### timezone {#timezone}

如果指定了 `timezone`，VM 的时钟将被设置为所指定的本地时间。

    metadata:
      name: myvmi
    spec:
      domain:
        clock:
          timezone: "America/New York"
        resources:
          requests:
            memory: 512M
        devices:
          disks:
          - name: myimage
            disk: {}
      volumes:
        - name: myimage
          persistentVolumeClaim:
            claimName: myclaim

#### 定时器 {#timers}

-   **pit**

-   **rtc**

-   **kvm**

-   **hyperv**

VM 的一种很常见的定时器配置如下所示：

    metadata:
      name: myvmi
    spec:
      domain:
        clock:
          utc: {}
          # here are the timer
          timer:
            hpet:
              present: false
            pit:
              tickPolicy: delay
            rtc:
              tickPolicy: catchup
            hyperv: {}
        resources:
          requests:
            memory: 512M
        devices:
          disks:
          - name: myimage
            disk: {}
      volumes:
        - name: myimage
          persistentVolumeClaim:
            claimName: myclaim

`hpet` 被禁用，`pit` 和 `rtc` 被配置为使用特定的 `tickPolicy`。最后，`hyperv` 也被提供出来。

全部可能的配置选项请参阅[定时器 API 参考](https://kubevirt.github.io/api-reference/master/definitions.html#_v1_timer)。

**注意**：定时器可以是机器类型的一部分。因此，可能有必要显式禁用它们。如果它们属于某个 QEMU 机器定义，我们将来可能会决定通过集群级默认值把它们添加进来。

### 随机数生成器（RNG） {#random-number-generator-rng}

你可能希望在客户机内部使用集群节点所收集的熵。KubeVirt 允许按如下方式向虚拟机添加一个 `virtio` RNG 设备。

    metadata:
      name: vmi-with-rng
    spec:
      domain:
        devices:
          rng: {}

对于 Linux 客户机，应当在启动过程的早期加载 `virtio-rng` 内核模块，以便获得对熵源的访问。其他系统可能需要进行类似的调整，才能与 `virtio` RNG 设备一起工作。

**注意**：某些客户机操作系统或用户负载可能需要具备足够熵的 RNG 设备，没有它就可能无法启动。例如，内核较新（4.16.4 及以上）的全新 Fedora 镜像可能需要存在 `virtio` RNG 设备，才能启动到登录界面。

### 视频和图形设备 {#video-and-graphics-device}

默认情况下，会向 VirtualMachineInstance 应用一套最小化的视频和图形设备配置。视频设备与 `vga` 兼容，并带有 16 MB 的内存大小。此设备允许通过 `vnc` 连接到操作系统。

可以通过把 `spec.domain.devices.autoattachGraphicsDevice` 设置为 `false` 来不挂接视频设备：

    metadata:
      name: myvmi
    spec:
      domain:
        devices:
          autoattachGraphicsDevice: false
          disks:
          - name: myimage
            disk: {}
      volumes:
        - name: myimage
          persistentVolumeClaim:
            claimName: myclaim

没有图形设备和视频设备的 VMI 常常被称为 `headless` VMI。

如果使用大量小型 VM，这有助于提高每个节点的 VMI 密度，因为不需要为视频预留内存。

#### 自定义视频设备类型 {#custom-video-device-types}

可以配置自定义视频设备类型，以覆盖默认的视频配置。这使你能够根据客户机操作系统的要求和性能需要，指定诸如 `virtio`、`vga` 或 `bochs` 之类的视频设备类型。

> **注意**：在 1.9 之前的 KubeVirt 版本上，必须手动启用 `VideoConfig` 功能门控。说明请参阅[功能门控激活指南](../cluster_admin/activating_feature_gates.md#how-to-activate-a-feature-gate)。

##### 配置视频设备类型 {#configuring-video-device-types}

你可以在 VM 模板规约中的 `spec.template.spec.domain.devices.video` 下指定视频设备类型。例如：

```yaml
apiVersion: kubevirt.io/v1
kind: VirtualMachine
metadata:
  name: my-vm-with-virtio-gpu
spec:
  template:
    spec:
      domain:
        devices:
          video:
            type: virtio
        resources:
          requests:
            memory: 2Gi
```

**支持的视频设备类型：**

- **`virtio`**：性能更好，推荐用于已提供 virtio 驱动的现代客户机操作系统
- **`vga`**：标准的 VGA 兼容设备（在使用 BIOS 的 AMD/x86_64 上为默认值）
- **`bochs`**：与 `vga` 类似，但效率更高（在使用 EFI 的 AMD/x86_64 上为默认值）
- **`cirrus`**：传统的 Cirrus Logic 设备
- **`ramfb`**：简单的帧缓冲设备，适用于 ARM 架构

##### 特定架构的行为 {#architecture-specific-behavior}

默认视频设备类型因架构而异，并且可以被覆盖：

| 架构 | 启动模式 | 默认类型 | 支持的类型                             |
|--------------|-----------|--------------|---------------------------------------------|
| AMD/x86_64   | BIOS      | `vga`        | `virtio`, `vga`, `bochs`, `cirrus`, `ramfb` |
| AMD/x86_64   | EFI       | `bochs`      | `virtio`, `vga`, `bochs`, `cirrus`, `ramfb` |
| ARM64        | BIOS/EFI  | `virtio`     | `virtio`, `ramfb`                           |
| s390x        | BIOS/EFI  | `virtio`     | `virtio`                                    |

##### 自定义视频设备配置的好处 {#benefits-of-custom-video-device-configuration}

使用自定义视频设备类型有若干优点：

- **性能**：与传统的 VGA 模拟相比，`virtio` 视频设备能提供更好的性能
- **分辨率灵活性**：借助 `virtio`，你可以设置不受 VGA 限制的自定义显示分辨率
- **内存效率**：`ramfb` 这类视频类型对于无头或仅控制台的操作更节省内存

显式使用 virtio 配置的 ARM64 示例：

```yaml
apiVersion: kubevirt.io/v1
kind: VirtualMachine
metadata:
  name: arm64-vm-custom-video
spec:
  template:
    spec:
      domain:
        devices:
          video:
            type: virtio
        resources:
          requests:
            memory: 4Gi
```

把 AMD/x86_64 的默认 VGA 视频类型覆盖为 virtio 的示例：

```yaml
apiVersion: kubevirt.io/v1
kind: VirtualMachine
metadata:
  name: amd64-vm-virtio-video
spec:
  template:
    spec:
      domain:
        devices:
          video:
            type: virtio
        resources:
          requests:
            memory: 2Gi
```

同样，你可以覆盖为 `bochs`、`cirrus` 或 `ramfb` 等其他受支持的类型：

```yaml
apiVersion: kubevirt.io/v1
kind: VirtualMachine
metadata:
  name: amd64-vm-bochs-video
spec:
  template:
    spec:
      domain:
        devices:
          video:
            type: bochs
        resources:
          requests:
            memory: 2Gi
```

**注意**：使用自定义视频设备类型时，请确保客户机操作系统安装了相应的驱动。对于 virtio-gpu 设备，需要 virtio 图形驱动才能获得最佳性能。Bochs 也需要驱动，但只针对 BIOS 客户机，因为我们的 EFI 固件（OVMF）自带 bochs 驱动。

### 特性 {#features_1}

KubeVirt 支持一系列虚拟化特性，可以进行调整，以便让非基于 Linux 的操作系统能够正常启动。最值得注意的是

-   **acpi**

-   **apic**

-   **hyperv**

下面的示例展示了一种常见的特性配置：

```yaml
apiVersion: kubevirt.io/v1
kind: VirtualMachineInstance
metadata:
  name: myvmi
spec:
  domain:
    # typical features
    features:
      acpi: {}
      apic: {}
      hyperv:
        relaxed: {}
        vapic: {}
        spinlocks:
          spinlocks: 8191
    resources:
      requests:
        memory: 512M
    devices:
      disks:
      - name: myimage
        disk: {}
  volumes:
    - name: myimage
      persistentVolumeClaim:
        claimname: myclaim
```

全部可用的特性以及配置选项请参阅[特性 API 参考](https://kubevirt.github.io/api-reference/master/definitions.html#_v1_features)。

## 资源请求与限制 {#resources-requests-and-limits}

用户可以指定可选的资源请求，让调度器更好地决定把 VM 放置到最合适的节点上。

```yaml
apiVersion: kubevirt.io/v1
kind: VirtualMachineInstance
metadata:
  name: myvmi
spec:
  domain:
    resources:
      requests:
        memory: "1Gi"
        cpu: "1"
      limits:
        memory: "2Gi"
        cpu: "2"
      disks:
      - name: myimage
        disk: {}
  volumes:
    - name: myimage
      persistentVolumeClaim:
        claimname: myclaim
```

### CPU {#cpu_1}

指定 CPU 限制将决定 VM 正在其中运行的控制组上所设置的 *cpu* *shares* 数量，换句话说，就是当存在 CPU 资源竞争时，VM 的 CPU 可以在已分配资源上执行的时间量。

更多信息请参阅[带有资源限制的 Pod 如何运行](https://kubernetes.io/docs/concepts/configuration/manage-compute-resources-container/#how-pods-with-resource-limits-are-run)。

### 大页 {#hugepages}

KubeVirt 让你可以使用大页作为 VM 的后备内存。你需要提供期望的内存量 `resources.requests.memory`，以及要使用的大页大小 `memory.hugepages.pageSize`，例如对于 x86\_64 架构，它可以是 `2Mi`。

```yaml
apiVersion: kubevirt.io/v1alpha1
kind: VirtualMachine
metadata:
  name: myvm
spec:
  domain:
    resources:
      requests:
        memory: "64Mi"
    memory:
      hugepages:
        pageSize: "2Mi"
    disks:
    - name: myimage
      disk: {}
  volumes:
    - name: myimage
      persistentVolumeClaim:
        claimname: myclaim
```

在上面的示例中，VM 将拥有 `64Mi` 内存，但它使用的不是常规内存，而是大小为 `2Mi` 的节点大页。

### 限制 {#limitations}

-   节点必须预先分配好大页

-   大页大小不能大于所请求的内存

-   所请求的内存必须能被大页大小整除

-   大页默认使用 memfd。内核版本大于等于 4.14 时才支持 memfd。如果你在更旧的宿主机上运行（例如 centos 7.9），必须在 VMI metadata 注解中加入 `kubevirt.io/memfd: "false"` 来禁用 memfd。

## 输入设备 {#input-devices}

### 数位板 {#tablet}

KubeVirt 支持输入设备。唯一受支持的类型是 `tablet`。数位板输入设备只支持 `virtio` 和 `usb` 总线。总线可以为空。在这种情况下，会选择 `usb`。

```yaml
apiVersion: kubevirt.io/v1
kind: VirtualMachine
metadata:
  name: myvm
spec:
  domain:
    devices:
      inputs:
      - type: tablet
        bus: virtio
        name: tablet1
      disks:
      - name: myimage
        disk: {}
  volumes:
    - name: myimage
      persistentVolumeClaim:
        claimname: myclaim
```
