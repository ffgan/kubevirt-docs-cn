---
hide:
  - navigation
---

# KubeVirt 发行说明 {#kubevirt-release-notes}

## v1.10.0 {#v1100}

发布日期：2026 年 11 月 18 日星期三

KubeVirt v1.10 面向 Kubernetes v1.37 构建，并额外支持此前两个版本。更多信息请参见 [KubeVirt 支持矩阵](https://github.com/kubevirt/sig-release/blob/main/releases/k8s-support-matrix.md)。

要查看发行产物，以及使这个版本得以发布的 100 多位（！）优秀的人，请参见 [KubeVirt v1.10.0 发行标签](https://github.com/kubevirt/kubevirt/releases/tag/v1.10.0)。

!!! Important
    安装 `kubevirt-operator.yaml` 时如果使用客户端 `kubectl apply -f kubevirt-operator.yaml`
    会失败，因为 kubevirts.kubevirt.io CRD 现已超过 256 KiB 的注解限制。请改用
    `kubectl apply --server-side`（或 `kubectl create`）安装 `kubevirt-operator.yaml`。
    参见 [#19980](https://github.com/kubevirt/kubevirt/issues/19980)。

### API 变更 {#api-change}
- [[PR #19732]](https://github.com/kubevirt/kubevirt/pull/19732)[stu-gott] 实时迁移不再从运行已更新 virt-handler 的节点调度到尚未更新的节点，从而避免滚动升级期间迁移卡住。
- [[PR #19690]](https://github.com/kubevirt/kubevirt/pull/19690)[kubevirt-bot] Beta：IncrementalBackup，并引入 backup.kubevirt.io/v1beta1
- [[PR #19294]](https://github.com/kubevirt/kubevirt/pull/19294)[alromeros] 离线增量备份
- [[PR #19574]](https://github.com/kubevirt/kubevirt/pull/19574)[noamasu] 在 External 模式快照期间 virt-launcher 崩溃后，VM 启动时会恢复客户机写入。
- [[PR #19360]](https://github.com/kubevirt/kubevirt/pull/19360)[noamasu] VEP 321：新增 External 模式的 VirtualMachineSnapshot
- [[PR #19550]](https://github.com/kubevirt/kubevirt/pull/19550)[Acedus] 内存转储以工具卷的形式挂接，并需要 UtilityVolumes 功能门控；该功能门控已提升为 Beta 并默认启用。VM 快照和导出不再包含内存转储 PVC。
- [[PR #19056]](https://github.com/kubevirt/kubevirt/pull/19056)[Acedus] 当磁盘丢失基础检查点的位图时，VirtualMachineBackup 不再失败；该磁盘会做全量备份，并且类型现在按卷在 `status.includedVolumes[].type` 中报告，取代已移除的 `status.type`。
- [[PR #19511]](https://github.com/kubevirt/kubevirt/pull/19511)[nirdothan] NetworkDevicesWithDRA 功能门控现已进入 Beta 阶段
- [[PR #19228]](https://github.com/kubevirt/kubevirt/pull/19228)[Sreeja1725] Alpha：新增 CPUsWithDRA，通过 DRA CPU 驱动为 VMI 置备专用 CPU，以代替 kubelet 的 CPU 管理器。
- [[PR #19572]](https://github.com/kubevirt/kubevirt/pull/19572)[orelmisan] 启用 NetworkDevicesWithDRA 功能门控后，对于设备已在 Pod 中就绪的接口（由 DRA 支撑的接口，以及由 Multus 支撑的辅助接口），现在会报告 pod-status 这一 VMI 网络接口信息来源。
- [[PR #19173]](https://github.com/kubevirt/kubevirt/pull/19173)[alromeros] 声明式 VirtualMachineState
- [[PR #18249]](https://github.com/kubevirt/kubevirt/pull/18249)[dsanatar] virtio-scsi 的 IOThread Virtqueue 映射
- [[PR #19403]](https://github.com/kubevirt/kubevirt/pull/19403)[iholder101] 为插件新增 PreBoot 和 PreMigrationSource 边车 launcher 钩子点。
- [[PR #19215]](https://github.com/kubevirt/kubevirt/pull/19215)[Barakmor1] 新增 TLSGroupPreferences 功能门控，并向 TLSConfiguration 添加 Groups 字段，使集群管理员能够在所有 virt Pod 端点上配置 TLS 支持的组（椭圆曲线），为后量子密码学做好准备
- [[PR #19244]](https://github.com/kubevirt/kubevirt/pull/19244)[jschintag] 新增集群范围以及按 VM 的仿真配置选项（Alpha 阶段）。
- [[PR #19351]](https://github.com/kubevirt/kubevirt/pull/19351)[awels] 使用代理 dataPath 的去中心化实时迁移支持 Route/Ingress/Service 端点
- [[PR #19160]](https://github.com/kubevirt/kubevirt/pull/19160)[iholder101] 插件的 spec.domainHooks 已重命名为 spec.launcherHooks。Launcher 钩子现在需要显式的钩子点。PreVMStop 已重命名为 OnVMStop。
- [[PR #19006]](https://github.com/kubevirt/kubevirt/pull/19006)[lyarwood] 将所引用 instancetype 与偏好对象上的 `instancetype.kubevirt.io/common-instancetypes-version` 标签传播到它们所捕获的 `ControllerRevision` 元数据标签。
- [[PR #18537]](https://github.com/kubevirt/kubevirt/pull/18537)[davmarro] 新增结构化 `links` 字段到 `VirtualMachineBackup` 的 status 中，用于暴露拉取模式备份的内部和外部导出端点。现有的端点字段已被移除。
- [[PR #19033]](https://github.com/kubevirt/kubevirt/pull/19033)[michalskrivanek] 新增 kubevirt.io/mergeable-memory 注解，用于让客户机内存退出内核内存合并
- [[PR #17922]](https://github.com/kubevirt/kubevirt/pull/17922)[awels] 去中心化实时迁移现在可以使用代理，以减少所需的 IP 地址数量
- [[PR #19100]](https://github.com/kubevirt/kubevirt/pull/19100)[xpivarc] 不再使用标签 "cpumanager"
- [[PR #17718]](https://github.com/kubevirt/kubevirt/pull/17718)[michalskrivanek] 新增对 "directsync" 磁盘缓存模式的支持。Directsync 类似于 "none" 与 "writethrough" 的组合：绕过宿主机页缓存，客户机侧也不做缓存，所有写入都直接进入存储。由于绕过了宿主机页缓存，它比 "writethrough" 性能更好，宿主机内存开销也更低。
- [[PR #18315]](https://github.com/kubevirt/kubevirt/pull/18315)[alancaldelas] Kubernetes 依赖已更新到 v0.36.4（Kubernetes 1.36）
- [[PR #18847]](https://github.com/kubevirt/kubevirt/pull/18847)[akalenyu] 清理：移除遗留的 containerdisk CRC32 校验和计算
- [[PR #18303]](https://github.com/kubevirt/kubevirt/pull/18303)[Aseeef] 破坏性变更：修改了停顿检测功能的内部线上 API 字段；集群升级要成功，必须禁用 MigrationStallDetection FG。
- [[PR #17480]](https://github.com/kubevirt/kubevirt/pull/17480)[michalskrivanek] 现在可以通过 MigrationPolicy 中的 .spec.experimental.downtimeTuning 字段动态调节实时迁移停机时间。启用后，QEMU 的 max_downtime 会从初始值（默认 150ms）逐渐提高到可配置上限（默认 1050ms），帮助迁移在有上界的亚秒级切换停机时间内收敛。
- [[PR #18491]](https://github.com/kubevirt/kubevirt/pull/18491)[vladikr] 新增将 OCI 制品（例如使用 ORAS 推送的原始磁盘镜像）作为 containerDisk 源的支持，经由 ImageVolume 路径提供。在 VMI 上设置注解 `kubevirt.io/image-volume-skip-digest-resolution: "true"`，以跳过与非容器 OCI 制品不兼容的摘要解析 init 容器。磁盘文件发现现在也支持磁盘位于根目录的镜像，而不仅限于 /disk 之下。
- [[PR #18222]](https://github.com/kubevirt/kubevirt/pull/18222)[lukashes] 修复了 virt-launcher Pod 已经消失时删除附加 Pod 所导致的 VMI 所有者解析问题
- [[PR #17509]](https://github.com/kubevirt/kubevirt/pull/17509)[michalskrivanek] 现在可以通过应用于一组 VM 的 MigrationPolicy 中的 .spec.experimental.compression 字段压缩实时迁移数据流。目前仅支持 "zstd" 算法。

### 缺陷修复 {#bug-fix}
- [[PR #19862]](https://github.com/kubevirt/kubevirt/pull/19862)[kubevirt-bot] 将 libvirtxml 升级到 v1.12007.0，并修复 smmuv3 IOMMU 的 ssidsize 属性，使 Grace IOVirtualization VMI 在客户机定义插件钩子运行时获得其 SSID 大小，并保留其 IOMMU 驱动属性。
- [[PR #19771]](https://github.com/kubevirt/kubevirt/pull/19771)[kubevirt-bot] 修复启用 Plugins 功能门控且集群中存在任意 Plugin 时，带有 SR-IOV、GPU 或 PCI 主机设备的 VMI 无法启动的问题。
- [[PR #19863]](https://github.com/kubevirt/kubevirt/pull/19863)[kubevirt-bot] 节点钩子在未设置自己的 failureStrategy 时，现在会像 launcher 钩子一样回退到 Plugin 级别的 failureStrategy。spec.failureStrategy 为 Ignore 的 Plugin，在其某个未单独设置 failureStrategy 的节点钩子失败时，不再阻止 VM 启动、停止或迁移。
- [[PR #19743]](https://github.com/kubevirt/kubevirt/pull/19743)[kubevirt-bot] 修复了设备插件重启时 virt-handler 发生 panic 的问题；在带有主机设备的节点上，每次 kubelet 重启都会触发该问题。
- [[PR #19492]](https://github.com/kubevirt/kubevirt/pull/19492)[caxu-rh] 修复了 virt-handler 客户端初始化失败时导致 gRPC 连接泄漏的缺陷
- [[PR #19485]](https://github.com/kubevirt/kubevirt/pull/19485)[0xFelix] 导出的 OCI 归档现在包含作为 blob 的镜像索引，并由 index.json 以 ref.name "latest" 引用，因此推送这些归档会在镜像仓库中创建镜像索引
- [[PR #19590]](https://github.com/kubevirt/kubevirt/pull/19590)[lyarwood] 使用 go 1.26.10 构建 KubeVirt
- [[PR #19066]](https://github.com/kubevirt/kubevirt/pull/19066)[0xFelix] 导出的 OCI 制品的 io.kubevirt.disk.name 注解现在携带 VM 中的卷名，而不再是 PersistentVolumeClaim 名称
- [[PR #19064]](https://github.com/kubevirt/kubevirt/pull/19064)[0xFelix] 修复了如下缺陷：当某个 VM 的两个 claim 名称仅在点号上不同时，其 VirtualMachineExport 永远无法完成
- [[PR #18618]](https://github.com/kubevirt/kubevirt/pull/18618)[imantaba] 修复了 VirtualMachineRestore 准入器拒绝有效补丁的问题；这些补丁的值包含 ":" 或 ","（例如替换接口 MAC 地址），或者其值为对象或数组。
- [[PR #19019]](https://github.com/kubevirt/kubevirt/pull/19019)[lllamnyp] virt-api：port-forward 子资源（由 `virtctl port-forward` 和 `virtctl ssh` 使用）现在会拨号连接到 Pod 网络的 VMI 所对应的 virt-launcher Pod IP，而不再使用 VMI 接口状态中报告的第一个 IP。virt-api 集群角色获得对 Pod 的 `watch` 动词。
- [[PR #19315]](https://github.com/kubevirt/kubevirt/pull/19315)[Acedus] 修复：增量拉取模式备份不再在碎片化磁盘上把已变更块从备份映射中静默遗漏。
- [[PR #19166]](https://github.com/kubevirt/kubevirt/pull/19166)[alromeros] 缺陷修复：延长 Velero 备份后钩子的超时时间
- [[PR #19120]](https://github.com/kubevirt/kubevirt/pull/19120)[akalenyu] 缺陷修复：对文件系统卷执行 I/O 时 blockresize 刷屏
- [[PR #19085]](https://github.com/kubevirt/kubevirt/pull/19085)[0xFelix] 修复了带有持久化 TPM 或 EFI 的 VM 的 OCI VirtualMachineExport 一直未能就绪的缺陷
- [[PR #18274]](https://github.com/kubevirt/kubevirt/pull/18274)[ksimon1] virtctl：新增 vsock ssh proxy 命令
- [[PR #18843]](https://github.com/kubevirt/kubevirt/pull/18843)[awels] 缺陷修复：同一个 VM 同时使用工具卷和热插拔卷时，避免产生重复的附加 Pod
- [[PR #18965]](https://github.com/kubevirt/kubevirt/pull/18965)[lukashes] 修复了 CA 轮换后重新创建 exporter Pod 时 VirtualMachineExport 清单仍保留过期 CA 证书的问题，该问题可能导致 CDI 磁盘导入的 TLS 校验失败
- [[PR #19002]](https://github.com/kubevirt/kubevirt/pull/19002)[machadovilaca] 向 GetVMStats 添加 guest-get-fsinfo
- [[PR #18604]](https://github.com/kubevirt/kubevirt/pull/18604)[awels] 缺陷修复：修复了目标节点字段与源字段发生竞争、可能导致迁移失败的竞态条件。
- [[PR #16346]](https://github.com/kubevirt/kubevirt/pull/16346)[ram2valar] 修复了 virt-handler 在检测 CPU 管理器是否启用时忽略 --kubelet-root 标志的缺陷，该问题会在 kubelet 根目录不是默认值的集群（例如 k3s、k0s、MicroK8s）上导致错误的节点标签。```
- [[PR #18988]](https://github.com/kubevirt/kubevirt/pull/18988)[davmarro] 修复了去中心化实时迁移中源端幽灵记录泄漏的问题，该泄漏可能阻止后续同名 VirtualMachineInstance 在同一节点上启动。
- [[PR #18970]](https://github.com/kubevirt/kubevirt/pull/18970)[xpivarc] 缺陷修复：无响应套接字没有被正确计入
- [[PR #18905]](https://github.com/kubevirt/kubevirt/pull/18905)[fossedihelm] 修复在不支持 CONFIG_VFIO_DEVICE_CDEV 的内核上 PCI 直通的回归。当 IOMMUFD cdev 不可用时，带有 GPU/PCI 主机设备的 VM 现在会正确回退到旧版 VFIO group 接口。
- [[PR #18716]](https://github.com/kubevirt/kubevirt/pull/18716)[xpivarc] 缺陷修复：确保记录（幽灵记录、挂载记录）以原子方式写入磁盘
- [[PR #18787]](https://github.com/kubevirt/kubevirt/pull/18787)[davmarro] 修复了如下缺陷：RunStrategy 为 RunStrategyRerunOnFailure 的 VM 在其 PVC 未绑定的情况下被手动停止后，错误地显示 WaitingForVolumeBinding 状态。
- [[PR #18822]](https://github.com/kubevirt/kubevirt/pull/18822)[sradco] 通过恢复带有静态 install-namespace 标签的集群范围评估，修复 OrphanedVirtualMachineInstances 误报以及 VirtLauncherPodsStuckFailed 的按命名空间阈值
- [[PR #17008]](https://github.com/kubevirt/kubevirt/pull/17008)[xpivarc] 缺陷修复："virtctl evacuate-cancel" 现在可以正常工作
- [[PR #17605]](https://github.com/kubevirt/kubevirt/pull/17605)[dasionov] 缺陷修复：防止客户机操作系统通过热插拔机制（例如 Windows 的“安全删除硬件”托盘菜单）弹出关键 PCI 设备（磁盘、内存气球、RNG、watchdog）。承载非 NIC 设备的 PCI 根端口现在被标记为不可热插拔，而 NIC 端口以及为将来热插拔预留的空端口仍然可用。
- [[PR #18362]](https://github.com/kubevirt/kubevirt/pull/18362)[iholder101] 缺陷修复：实时迁移期间，域统计信息收集不再在 libvirt 作业锁上阻塞长达 30 秒。现使用 NOWAIT 标志跳过已锁定的域，而不是等待，从而避免锁竞争和干扰性错误日志。
- [[PR #18311]](https://github.com/kubevirt/kubevirt/pull/18311)[akalenyu] 缺陷修复：在线扩容在卷数量很多时扩展性较差
- [[PR #18460]](https://github.com/kubevirt/kubevirt/pull/18460)[dasionov] 修复边车钩子在 XML 往返过程中静默丢弃 QEMU 命令行参数（qemu:commandline）的问题
- [[PR #18542]](https://github.com/kubevirt/kubevirt/pull/18542)[thc1006] RamFB 显示校验现在会把存在 ramFB 但其 enabled 字段未设置的 vGPU 视为已启用的显示设备，与 API 默认值一致。包含多个此类 vGPU 的 VirtualMachine 或 VirtualMachineInstanceReplicaSet 模板现在会在创建时被拒绝，而不再是先被接受、等到创建其 VMI 时才失败。
- [[PR #18400]](https://github.com/kubevirt/kubevirt/pull/18400)[awels] 缺陷修复：vmexport 代理中的文件描述符泄漏
- [[PR #18410]](https://github.com/kubevirt/kubevirt/pull/18410)[vikin91] 修复 client-go 的 AsyncSubresourceHelper 在其 StreamInterface 通过 AsConn() 而不是 Stream() 消费时的 goroutine（以及连接资源）泄漏（影响 VSOCK、PortForward TCP/UDP 以及类似的消费者）。
- [[PR #18414]](https://github.com/kubevirt/kubevirt/pull/18414)[iholder101] 修复 CallNodeHooks，使其评估插件级别的 Spec.Condition，确保域钩子与节点钩子的行为一致。
- [[PR #18453]](https://github.com/kubevirt/kubevirt/pull/18453)[iholder101] 缺陷修复：当 virt-handler 重启并暂时失去 launcher 套接字连接时，处于 Running 状态的 VMI 不再被立即标记为 Failed。现在提供 3 分钟宽限期（与现有 Scheduled VMI 的行为一致），允许 launcher 客户端在 VMI 转为 Failed 之前重新连接。
- [[PR #18374]](https://github.com/kubevirt/kubevirt/pull/18374)[orelmisan] 修复了网络接口绑定缺少校验的问题：没有绑定或存在多个绑定的接口现在会在准入时被拒绝，而不再是在运行时静默失败。
- [[PR #18300]](https://github.com/kubevirt/kubevirt/pull/18300)[lyarwood] 缺陷修复：向 virt-launcher 容器暴露 VFIO cdev 设备（/dev/vfio/devices/vfioN），修复带有 SMMUv3 的 ARM64 系统上基于 IOMMUFD 的 GPU 直通。
- [[PR #18044]](https://github.com/kubevirt/kubevirt/pull/18044)[maxwmsft] 缺陷修复：已经移除的卷在分离热插拔卷时死锁。
- [[PR #18045]](https://github.com/kubevirt/kubevirt/pull/18045)[maxwmsft] 缺陷修复：VMI 在附加 Pod 之间交换 RWO PVC 时，热插拔卷分离死锁。
- [[PR #18435]](https://github.com/kubevirt/kubevirt/pull/18435)[orelmisan] 网络绑定插件边车现在可以使用 `NETWORK_BINDING_PLUGIN_NAME` 环境变量推断其注册名称。
- [[PR #17752]](https://github.com/kubevirt/kubevirt/pull/17752)[iholder101] 缺陷修复：使用 NFS 支撑存储的 VM 在 NFS 暂时不可用期间（例如 NFS 服务器故障转移）不再被错误地重启。
- [[PR #17286]](https://github.com/kubevirt/kubevirt/pull/17286)[alromeros] 缺陷修复：修复后端存储不可快照时的 VM 恢复
- [[PR #18392]](https://github.com/kubevirt/kubevirt/pull/18392)[akalenyu] 缺陷修复：导入器 Pod 被 ValidatingAdmissionPolicy 'kubevirt-plugin-sidecar-subpath-policy' 拒绝
- [[PR #18370]](https://github.com/kubevirt/kubevirt/pull/18370)[michalskrivanek] 现在会发出 `kubevirt_vmi_guest_os_panic_total` 指标
- [[PR #18272]](https://github.com/kubevirt/kubevirt/pull/18272)[laxmi-333] 缺陷修复：修复了 hugetblfs-dir EmptyDir 卷遮盖 /dev/hugepages 处父级 hugetlbfs 挂载的问题，该问题使 QEMU 无法通过 -mem-path 分配大页。该 EmptyDir 现在显式由大页支撑。
- [[PR #18294]](https://github.com/kubevirt/kubevirt/pull/18294)[guzalv] client-go/log：`-v` 详细程度标志不再注册到 `flag.CommandLine`（发生在 `init()` 期间）。此前使用 `flag.CommandLine.Lookup("v")` 获取该详细程度标志的外部使用者必须改用 `log.VerbosityFlag()`。这样在导入 `kubevirt.io/client-go` 的同时，不会再因为其他同样注册 `-v` 标志的包（例如 glog、klog）而 panic。
- [[PR #18206]](https://github.com/kubevirt/kubevirt/pull/18206)[lukashes] 修复了 VMI 控制器错过 Pod 事件时可能出现的 VM/VMI 删除延迟。
- [[PR #18314]](https://github.com/kubevirt/kubevirt/pull/18314)[lyarwood] 缺陷修复：当 VMI 上不存在 panic 设备时，PreferredPanicDeviceModel 现在会创建一个，而不再静默忽略该偏好。
- [[PR #17775]](https://github.com/kubevirt/kubevirt/pull/17775)[akalenyu] 缺陷修复：管理员可以从其 VM 中移除内存转储

### 弃用 {#deprecation}
- [[PR #19362]](https://github.com/kubevirt/kubevirt/pull/19362)[NoamBerkoLaor] 移除了已弃用的 virtctl create vm 标志 --volume-blank、--volume-datasource 和 --volume-clone-pvc。请改用带 type:blank、type:ds 或 type:pvc 的 --volume-import。
- [[PR #13881]](https://github.com/kubevirt/kubevirt/pull/13881)[akrejcir] 已弃用在 VM 中设置 SSH 密钥的旧方法。

### SIG-compute {#sig-compute}
- [[PR #19688]](https://github.com/kubevirt/kubevirt/pull/19688)[kubevirt-bot] CBT 叠加层现在以卷和 PVC 名称为键。把卷指向另一个 PVC 会触发 Full 备份，而不是错误的增量备份。
- [[PR #17135]](https://github.com/kubevirt/kubevirt/pull/17135)[lyarwood] 通过 ARM64SecureBoot 和 FirmwareAutoSelection 功能门控，为 ARM64 客户机新增 Alpha 阶段的 UEFI 安全启动支持。需要带有合适 QEMU、libvirt 和 edk2 固件的 CentOS Stream 10 virt-launcher 镜像。支持通过现有后端存储持久化 EFI 状态。
- [[PR #19410]](https://github.com/kubevirt/kubevirt/pull/19410)[Acedus] HostDisk 卷不再适用于变更块跟踪。
- [[PR #19435]](https://github.com/kubevirt/kubevirt/pull/19435)[awels] 禁用 CBT 时清理 CBT 叠加层
- [[PR #19382]](https://github.com/kubevirt/kubevirt/pull/19382)[mhenriks] ContainerPath 卷升级为 Beta
- [[PR #19240]](https://github.com/kubevirt/kubevirt/pull/19240)[vladikr] virt-launcher-monitor 已用 C 重写，将内存占用降至约 2Mi
- [[PR #19546]](https://github.com/kubevirt/kubevirt/pull/19546)[dasionov] 将基于 mdev 的 vGPU 实时迁移的 Alpha 功能门控从 VGPULiveMigration 重命名为 MDevVGPULiveMigration。
- [[PR #19534]](https://github.com/kubevirt/kubevirt/pull/19534)[caxu-rh] 修复了在设置套接字文件所有权时如果出错，Unix 套接字监听器可能保持打开的缺陷
- [[PR #19409]](https://github.com/kubevirt/kubevirt/pull/19409)[Acedus] 对启用 CBT 的 VM 进行实时迁移时，会根据源端 domain 创建目标端的 CBT 叠加层；拔出卷时，无论 VMI 的 CBT 状态如何，都会移除其 CBT 叠加层。
- [[PR #19234]](https://github.com/kubevirt/kubevirt/pull/19234)[vladikr] 新增 pkg/render，用于在没有正在运行的集群的情况下，根据 VM/VMI 定义渲染 Pod spec
- [[PR #19213]](https://github.com/kubevirt/kubevirt/pull/19213)[Acedus] 同一 VMI 上另一个已热插拔的卷仍在填充、引用了缺失的 claim，或无法开始填充时，已热插拔的卷不再等待挂接。
- [[PR #19521]](https://github.com/kubevirt/kubevirt/pull/19521)[kubevirt-bot] 将 common-instancetypes 捆绑包更新到 v1.8.0
- [[PR #19468]](https://github.com/kubevirt/kubevirt/pull/19468)[orelmisan] DRA 网络由 ResourceClaim（而不是 ResourceClaimTemplate）支撑的 VMI 现在被报告为不可迁移。
- [[PR #19192]](https://github.com/kubevirt/kubevirt/pull/19192)[machadovilaca] 子命令失败时保留部分 VMStats
- [[PR #17399]](https://github.com/kubevirt/kubevirt/pull/17399)[dasionov] 移除了基于名称的遗留固件 UUID 计算以及固件同步器。
- [[PR #19099]](https://github.com/kubevirt/kubevirt/pull/19099)[HarshithaMS005] virt-handler 不再删除并非由它创建的中介设备，因此 DRA 拥有的 vfio-ap 设备不会从正在运行的 VM 底下被拆除
- [[PR #19239]](https://github.com/kubevirt/kubevirt/pull/19239)[akalenyu] Istio 边车代理的终止不再由 virt-launcher-monitor 处理；与 KubeVirt 一起使用时需要 Istio >= 1.27（或者使用原生边车模式）
- [[PR #19388]](https://github.com/kubevirt/kubevirt/pull/19388)[0xFelix] 将 virt-template 更新到 v0.3.0
- [[PR #18941]](https://github.com/kubevirt/kubevirt/pull/18941)[0xFelix] kubevirt.io/disablePCIHole64 注解现在也会阻止 SeaBIOS 配置 64 位 PCI 孔洞，从而使 Windows Server 2003 等旧版客户机能够以 4Gi 或更大内存运行。
- [[PR #18606]](https://github.com/kubevirt/kubevirt/pull/18606)[OOsemka] 将新的 virt-export Service 设为端口 8443 上的无头 Service，以便 virt-exportproxy 能够跨隔离网络访问 exporter。现有的 ClusterIP/443 导出 Service 保持不变。内部 VMExport 链接包含 Service 拨号端口（:8443 或 :443）。
- [[PR #19011]](https://github.com/kubevirt/kubevirt/pull/19011)[michalskrivanek] 当 Windows 客户机崩溃且 VM 配置了 `hyperv` panic 设备时，KubeVirt 不再立即终止该 VM，而是允许 Windows 完成其默认内存转储过程并重启。实际行为可在客户机侧配置。
- [[PR #17974]](https://github.com/kubevirt/kubevirt/pull/17974)[bxuan1999] virt-handler 现在为每个 VMI 暴露一个 gauge，其中包含最近一次成功实时迁移所报告的停机时间。
- [[PR #18938]](https://github.com/kubevirt/kubevirt/pull/18938)[machadovilaca] virt-handler 的 GetVMStats 接受按 VMI 的 VMStatsRequest 请求体
- [[PR #19155]](https://github.com/kubevirt/kubevirt/pull/19155)[0xFelix] VSOCK 功能门控已升级为 Beta，并且现在默认启用。
- [[PR #19169]](https://github.com/kubevirt/kubevirt/pull/19169)[0xFelix] 客户机操作系统重启后会刷新 VMStats 的客户机代理数据，而不再一直从缓存提供，直到其 TTL 过期
- [[PR #19131]](https://github.com/kubevirt/kubevirt/pull/19131)[vishnubijukumar] 修复了在 s390x 上热插拔块卷或触发实时迁移时，由于 /dev/urandom 被移出 cgroup 设备允许列表而导致 virtqemud 崩溃的缺陷。/dev/urandom（字符设备 1:9）现在无条件包含在默认 cgroup 设备规则中，与 OCI 运行时规范一致。这也修复了 x86_64 上的一个潜伏缺陷：/dev/urandom 被错误撤销，但被 RDRAND 硬件熵回退所掩盖。
- [[PR #18872]](https://github.com/kubevirt/kubevirt/pull/18872)[ameya-keskar] 在 VMI 上报告 InvalidImageName containerDisk 错误，并在 VM 上呈现 ErrImagePull，而不再让 VM 卡在 Starting。针对该错误条件，VM 池自动修复现在可能会生效，而此前不会。
- [[PR #19089]](https://github.com/kubevirt/kubevirt/pull/19089)[jean-edouard] 客户机时钟现在会在多次暂停/取消暂停之后正确同步
- [[PR #18943]](https://github.com/kubevirt/kubevirt/pull/18943)[HoustonBoston] 当使用 NFS 存储类且两个 VM 共享同一磁盘镜像时，用户现在会在故障 VM/VMI 的 'Conditions' 字段中收到比 QEMU 所报告内容更简化的错误消息。复杂的 QEMU 错误仍可在 'virt-handler' Pod 中找到。
- [[PR #18951]](https://github.com/kubevirt/kubevirt/pull/18951)[dsanatar] 为兼容 Centos Stream 10 所做的镜像变更，主要是在 virt-handler 镜像层中添加 /var/run -> ../run 符号链接，因为 CS10 在 /run/ 中创建套接字，而 KubeVirt 期望它们存在于 /var/run/
- [[PR #19070]](https://github.com/kubevirt/kubevirt/pull/19070)[lyarwood] Vera Rubin GPU（10de:3041）现在被识别为 Grace GPU，从而为直通 Vera Rubin GPU 的 VM 启用 GraceIOVirtualization。
- [[PR #18287]](https://github.com/kubevirt/kubevirt/pull/18287)[dasionov] 确保实时迁移之后，已终止的 launcher Pod 会从 VMI 的 status.activePods 中清理掉
- [[PR #18416]](https://github.com/kubevirt/kubevirt/pull/18416)[Acedus] 启用 CBT 的 VM 现在会正确计入每个所含磁盘所需的内存开销。
- [[PR #19007]](https://github.com/kubevirt/kubevirt/pull/19007)[codingben] 通过 VMStatsCollector 暴露客户机设备信息
- [[PR #18827]](https://github.com/kubevirt/kubevirt/pull/18827)[germanovm] 修复 PVC 扩容时在已满的精简置备磁盘上预分配新空间的问题
- [[PR #18888]](https://github.com/kubevirt/kubevirt/pull/18888)[dankenigsberg] 将 KubevirtSeccompProfile 功能门控升级为 GA。自定义 seccomp 配置文件现在始终会被安装。将配置文件权限收紧为 0600，并以原子方式写入。
- [[PR #18741]](https://github.com/kubevirt/kubevirt/pull/18741)[frenzyfriday] 网络绑定插件边车会接收 DRA 资源声明，以便访问设备元数据
- [[PR #18908]](https://github.com/kubevirt/kubevirt/pull/18908)[kubevirt-bot] 将 virt-template 更新到 v0.2.3
- [[PR #18451]](https://github.com/kubevirt/kubevirt/pull/18451)[orelmisan] DRA 网络不再支持 SR-IOV 接口。
- [[PR #18317]](https://github.com/kubevirt/kubevirt/pull/18317)[orelmisan] 将 PodSecondaryInterfaceNamingUpgrade 功能门控升级为 GA。把序号式网络接口名转换为哈希等价名的迁移前钩子现在会无条件运行。
- [[PR #18867]](https://github.com/kubevirt/kubevirt/pull/18867)[cwedgwood] 修复准备 VFIO 设备所有权失败时可能发生的 virt-handler 崩溃。
- [[PR #18624]](https://github.com/kubevirt/kubevirt/pull/18624)[dsanatar] 插件：整合 CEL 求值器
- [[PR #18829]](https://github.com/kubevirt/kubevirt/pull/18829)[omastama-spec] 新增从宿主机拓扑到客户机 VM 的 NUMA 节点距离直通，使对性能敏感的应用能够在多 NUMA 系统上做出最优调度决策。
- [[PR #18388]](https://github.com/kubevirt/kubevirt/pull/18388)[dasionov] 将 QEMU 进程内存开销从 30Mi 提高到 50Mi，从而使 VMI Pod 的内存请求增加 20Mi，以反映当前 QEMU 版本的实际 QEMU RSS 需求。
- [[PR #18687]](https://github.com/kubevirt/kubevirt/pull/18687)[Barakmor1] 将 LibvirtHooksServerAndClient 功能门控升级为 GA
- [[PR #18545]](https://github.com/kubevirt/kubevirt/pull/18545)[amasolov] 修复 QEMU 启动缓慢期间过早 undefine domain 的问题，该问题会使 VM 变为临时状态并破坏热插拔
- [[PR #17598]](https://github.com/kubevirt/kubevirt/pull/17598)[vamsikrishna-siddu] 将容器镜像构建从 rules_oci 迁移到 rules_img，从而能够进行具备原生 s390x 平台支持的 hermetic 多架构镜像构建。
- [[PR #17707]](https://github.com/kubevirt/kubevirt/pull/17707)[jean-edouard] virt-launcher Pod 会随迁移对象一起被清理
- [[PR #17117]](https://github.com/kubevirt/kubevirt/pull/17117)[suPer8Hu] N/A
- [[PR #17857]](https://github.com/kubevirt/kubevirt/pull/17857)[lyarwood] 为需要内存锁定的 VM（VFIO、realtime、SEV、显式 memlock）设置无限制的 memlock prlimit，确保 libvirt 的 memlock 计算在 session 模式下始终成功。
- [[PR #17936]](https://github.com/kubevirt/kubevirt/pull/17936)[dsanatar] virtctl：addvolume 的可选 --disk-name 标志
- [[PR #18337]](https://github.com/kubevirt/kubevirt/pull/18337)[0xFelix] 为基于 VolumeSnapshot 的 DataVolume、DataVolumeTemplate 和 DataSource 支持 inferFromVolume
- [[PR #18316]](https://github.com/kubevirt/kubevirt/pull/18316)[fanzhangio] 为 Grace I/O Virtualization 设置显式的 SMMUv3 地址能力默认值。
- [[PR #17955]](https://github.com/kubevirt/kubevirt/pull/17955)[akrejcir] 在 virt-handler 中新增可感知命名空间的 VSOCK 拨号支持（VEP 222）

### SIG-storage {#sig-storage}
- [[PR #18267]](https://github.com/kubevirt/kubevirt/pull/18267)[akalenyu] 启用持久预留时，将 LUN 总线默认设为 SCSI
- [[PR #19484]](https://github.com/kubevirt/kubevirt/pull/19484)[Acedus] VMI 在备份过程中被删除的备份现在会立即失败，并且拉取模式备份会移除其导出。
- [[PR #19057]](https://github.com/kubevirt/kubevirt/pull/19057)[0xFelix] 修复了源从不止一个卷引用同一个 PVC 时，VirtualMachineExport 永远无法完成的缺陷
- [[PR #18458]](https://github.com/kubevirt/kubevirt/pull/18458)[Ahmad-Hafe] 为存储的 PVC 扩容测试添加 Conformance 装饰器
- [[PR #18470]](https://github.com/kubevirt/kubevirt/pull/18470)[Acedus] 为启用 CBT 的 VM 将后端存储 PVC 大小增加 1Gi。
- [[PR #18200]](https://github.com/kubevirt/kubevirt/pull/18200)[naruto-lgtm] 通过改为常量时间比较，修复了导出服务器令牌检查中的计时侧信道。

### SIG-network {#sig-network}
- [[PR #18753]](https://github.com/kubevirt/kubevirt/pull/18753)[orelmisan] 修复 virt-api port-forward 中的 SSRF 漏洞（CVE-2026-13318）。该漏洞允许恶意 QEMU 客户机代理把 virtctl port-forward 和 ssh 流量重定向到任意网络端点。virt-api 现在只拨号 launcher Pod 的 IP，绝不会拨号客户机代理所报告的接口 IP。
- [[PR #18497]](https://github.com/kubevirt/kubevirt/pull/18497)[orelmisan] ExternalNetResourceInjection 已升级为 GA。virt-controller 不再查询 NetworkAttachmentDefinition 对象，也不再向 launcher Pod 注入网络资源请求。k8s.cni.cncf.io network-attachment-definitions 的 get 规则是
- [[PR #18613]](https://github.com/kubevirt/kubevirt/pull/18613)[orelmisan] 移除了 slirp 网络绑定插件边车镜像。该插件没有已知的使用者。
- [[PR #18194]](https://github.com/kubevirt/kubevirt/pull/18194)[frenzyfriday] 不可迁移的 VM 在交换 NAD 时需要重启
- [[PR #18021]](https://github.com/kubevirt/kubevirt/pull/18021)[orelmisan] 现在可以通过 `--primary-network-binding-plugin` 测试标志，使用自定义网络绑定插件运行网络一致性测试。

### SIG-scale {#sig-scale}
- [[PR #17985]](https://github.com/kubevirt/kubevirt/pull/17985)[sradco] 当组件完全不可用时，Virt*Down 告警会包含容器等待原因

### SIG-observability {#sig-observability}
- [[PR #19470]](https://github.com/kubevirt/kubevirt/pull/19470)[avlitman] 为 kubevirt_vmi_info 指标新增 has_panic_device 标签（true/false）
- [[PR #19149]](https://github.com/kubevirt/kubevirt/pull/19149)[sradco] 为 kubevirt_vm_info 和 kubevirt_vmi_info 添加 vm 标签（仅当 VMI 由 VirtualMachine 拥有时）
- [[PR #19144]](https://github.com/kubevirt/kubevirt/pull/19144)[machadovilaca] 新增 GPU 利用时间和活跃引擎时间的记录规则
- [[PR #18962]](https://github.com/kubevirt/kubevirt/pull/18962)[machadovilaca] 新增关联 DCGM 指标的 VMI GPU 使用记录规则
- [[PR #19041]](https://github.com/kubevirt/kubevirt/pull/19041)[machadovilaca] 允许 Prometheus 在 KubeVirt 命名空间中 list EndpointSlice
- [[PR #18994]](https://github.com/kubevirt/kubevirt/pull/18994)[avlitman] 修复挂载 UDF CDROM/ISO 时 GuestFilesystemAlmostOutOfSpace 在 Windows VM 上触发的问题。
- [[PR #18986]](https://github.com/kubevirt/kubevirt/pull/18986)[Ronilerr] 更改记录规则表达式
- [[PR #18942]](https://github.com/kubevirt/kubevirt/pull/18942)[Ronilerr] 修复无就绪告警
- [[PR #18902]](https://github.com/kubevirt/kubevirt/pull/18902)[machadovilaca] 为 kubevirt_vmi_gpu_info 添加 name 标签
- [[PR #18854]](https://github.com/kubevirt/kubevirt/pull/18854)[sradco] 为 kubevirt_vm_info 和 kubevirt_vmi_info 添加 uid 标签（Kubernetes 对象 UID）
- [[PR #18521]](https://github.com/kubevirt/kubevirt/pull/18521)[Ronilerr] 新增用于报告疑似孤立 PVC 的记录规则
- [[PR #17995]](https://github.com/kubevirt/kubevirt/pull/17995)[sradco] 为所有 KubeVirt 告警添加 namespace 标签，以便正确适配 OpenShift
- [[PR #18380]](https://github.com/kubevirt/kubevirt/pull/18380)[Ronilerr] 低 virt 计数告警使用的是 running 规则，而不是 up 规则
- [[PR #18154]](https://github.com/kubevirt/kubevirt/pull/18154)[Ronilerr] 新增 NoLeadingVirtController 告警

### 其他 {#other}
- [[PR #19520]](https://github.com/kubevirt/kubevirt/pull/19520)[noamasu] 使用内存型 emptyDir 作为热插拔挂载根目录，以支持挂载传播。
- [[PR #19490]](https://github.com/kubevirt/kubevirt/pull/19490)[awels] 仅当启用代理时，才把 virt-handler 迁移身份加入同步控制器。
- [[PR #19251]](https://github.com/kubevirt/kubevirt/pull/19251)[mhenriks] 在准入时检查 ContainerPathVolume 功能门控。拒绝 /run/kubevirt* 和 /run/libvirt* 路径。
- [[PR #19499]](https://github.com/kubevirt/kubevirt/pull/19499)[fjvbn2003] 当功能门控被禁用时，virt-operator 不再在每次调谐时向 API 服务器查询 ContainerPathVolumes mutating webhook。
- [[PR #19481]](https://github.com/kubevirt/kubevirt/pull/19481)[lyarwood] 修复由不稳定的缓存顺序引起的、重复出现的同步端点错误配置 Warning 事件。
- [[PR #18853]](https://github.com/kubevirt/kubevirt/pull/18853)[davmarro] 为所有 KubeVirt 基础设施容器启用 readOnlyRootFilesystem
- [[PR #19164]](https://github.com/kubevirt/kubevirt/pull/19164)[0xFelix] 在 amd64 架构上禁用 acpi 时添加警告
- [[PR #19048]](https://github.com/kubevirt/kubevirt/pull/19048)[michalskrivanek] libvirt 已更新到 12.5.0-4.el10
- [[PR #18874]](https://github.com/kubevirt/kubevirt/pull/18874)[xpivarc] client-go：GetKubevirtSubresourceClient 已被移除
- [[PR #18880]](https://github.com/kubevirt/kubevirt/pull/18880)[jean-edouard] 现在对使用大页的 VMI 忽略内存超分
- [[PR #18761]](https://github.com/kubevirt/kubevirt/pull/18761)[lentzi90] 使用 go 1.26.6 构建 KubeVirt
- [[PR #18708]](https://github.com/kubevirt/kubevirt/pull/18708)[Barakmor1] 将 VmiMemoryOverheadReport 功能门控升级为 GA
- [[PR #18692]](https://github.com/kubevirt/kubevirt/pull/18692)[vamsikrishna-siddu] 将 hermetic_launcher 升级到 v0.0.15，以修复 s390x 容器镜像构建。
- [[PR #18061]](https://github.com/kubevirt/kubevirt/pull/18061)[Sreeja1725] 为 DRA 功能新增 E2E 测试支持。
- [[PR #17880]](https://github.com/kubevirt/kubevirt/pull/17880)[Shreesha001] 修复 Informer 投递的是墓碑对象而不是已删除的 CRD 时，ClusterConfig CRD 删除处理程序中的 panic。
- [[PR #18247]](https://github.com/kubevirt/kubevirt/pull/18247)[alaypatel07] vep-10：将 GPUsWithDRA 和 HostDevicesWithDRA 移至 Beta

## v1.9.0 {#v190}

发布日期：2026 年 7 月 22 日星期三

KubeVirt v1.9 面向 Kubernetes v1.36 构建，并额外支持此前两个版本。更多信息请参见 [KubeVirt 支持矩阵](https://github.com/kubevirt/sig-release/blob/main/releases/k8s-support-matrix.md)。

要查看使这个版本得以发布的众多出色的人，请参见 [KubeVirt v1.9.0 发行标签](https://github.com/kubevirt/kubevirt/releases/tag/v1.9.0)。

### API 变更 {#api-change_1}
- [[PR #18331]](https://github.com/kubevirt/kubevirt/pull/18331)[kubevirt-bot] 现在可以通过应用于一组 VM 的 MigrationPolicy 中的 .spec.experimental.compression 字段压缩实时迁移数据流。目前仅支持 "zstd" 算法。
- [[PR #18095]](https://github.com/kubevirt/kubevirt/pull/18095)[fra2404] 为 masquerade 接口新增 PortRanges 支持（Alpha，由 PortRangesSpec 功能门控保护）
- [[PR #18048]](https://github.com/kubevirt/kubevirt/pull/18048)[alancaldelas] WorkloadEncryptionSEV 已从 Alpha 升级到 Beta
- [[PR #18101]](https://github.com/kubevirt/kubevirt/pull/18101)[dsanatar] 移除临时热插拔卷的指标和告警
- [[PR #18082]](https://github.com/kubevirt/kubevirt/pull/18082)[alromeros] 在 VirtualMachineBackup 中提前暴露文件系统冻结状态
- [[PR #18198]](https://github.com/kubevirt/kubevirt/pull/18198)[orenc1] VEP-160：将 OptOutRoleAggregation 功能门控升级为 Beta
- [[PR #17199]](https://github.com/kubevirt/kubevirt/pull/17199)[lyarwood] 新增 CrossArchitectureVirtualization Alpha 功能门控，以支持跨架构 VM 执行。启用后，VM 可以通过 QEMU TCG 软件仿真在架构不同的节点上运行（例如在 AMD64 宿主机上运行 ARM64 客户机）。调度器优先选择原生架构节点，不可用时回退到跨架构仿真。该功能门控的设计允许在同一门控下容纳未来的硬件加速后端（例如 SAE）。
- [[PR #18102]](https://github.com/kubevirt/kubevirt/pull/18102)[iholder101] 插件边车现在可以通过 gRPC 域钩子修改 libvirt domain XML，从而借助注入到 virt-launcher Pod 旁边的边车容器实现任意 domain 定制。
- [[PR #18032]](https://github.com/kubevirt/kubevirt/pull/18032)[0xFelix] 新增 VirtualMachineTemplate，作为通过 VMExport 进行 OCI 导出的源 kind。
- [[PR #18001]](https://github.com/kubevirt/kubevirt/pull/18001)[iholder101] 为插件框架新增节点生命周期钩子，使插件能够在 VM 启动/停止以及迁移阶段运行回调（Alpha，受 Plugins 功能门控保护）。
- [[PR #17544]](https://github.com/kubevirt/kubevirt/pull/17544)[Aseeef] 新增稳健的迁移停顿检测机制，在局部极小值处伺机触发 post-copy 或 stop-and-copy，从而同时缩短总迁移时长和停机时间。
- [[PR #17664]](https://github.com/kubevirt/kubevirt/pull/17664)[ksimon1] 功能：新增基于注解的 GuestAgentPing 探针暂停
- [[PR #17910]](https://github.com/kubevirt/kubevirt/pull/17910)[jcanocan] virtctl：将 virt-template 子命令更新到 v1beta1
- [[PR #18090]](https://github.com/kubevirt/kubevirt/pull/18090)[Acedus] VirtualMachineBackup 现在只包含 3 个状况：Progressing、Complete 和 Failed，并带有匹配的 reason，以取代现已移除的 Initializing、ExportInitiated、ExportReady、Aborting 和 Done。
- [[PR #17861]](https://github.com/kubevirt/kubevirt/pull/17861)[mhenriks] VEP 250：向 VirtualMachineInstance Spec 添加 serviceAccountName
- [[PR #17897]](https://github.com/kubevirt/kubevirt/pull/17897)[iholder101] 为结构化插件系统新增基于 CEL 的域钩子求值（Alpha，受 Plugins 功能门控保护）。
- [[PR #17551]](https://github.com/kubevirt/kubevirt/pull/17551)[jschintag] 允许通过偏好配置启动安全。
- [[PR #17923]](https://github.com/kubevirt/kubevirt/pull/17923)[alaypatel07] vep-10：将 dra 包加入 lint 配置并修复 linter 错误
- [[PR #17797]](https://github.com/kubevirt/kubevirt/pull/17797)[alaypatel07] vep-10：从 k8sv1.PodResourceClaim 迁移到 KubeVirt 自己的类型
- [[PR #17790]](https://github.com/kubevirt/kubevirt/pull/17790)[iholder101] 在 Plugins 功能门控（Alpha）之后引入 Plugin CRD（plugin.kubevirt.io/v1alpha1），通过域钩子、节点钩子和准入引用实现声明式 VM 扩展（VEP-190）。
- [[PR #17944]](https://github.com/kubevirt/kubevirt/pull/17944)[0xFelix] 在 OCIExport 功能门控之后新增 OCI 制品导出支持。使用 `virtctl vmexport download --format=oci` 将 VM 导出为 OCI image layout TAR。
- [[PR #17661]](https://github.com/kubevirt/kubevirt/pull/17661)[oshoval] 通过 `spec.networks[].resourceClaim` 在 `NetworkDevicesWithDRA` 之后新增 DRA SR-IOV 网络的 Alpha 支持，包含 webhook 校验，以及根据 DRA 元数据生成 virt-launcher hostdev。
- [[PR #16674]](https://github.com/kubevirt/kubevirt/pull/16674)[akalenyu] Persistent Reservation 进入 GA
- [[PR #17550]](https://github.com/kubevirt/kubevirt/pull/17550)[machadovilaca] 新增用于监控数据采集的统一 gRPC RPC GetVMStats
- [[PR #17029]](https://github.com/kubevirt/kubevirt/pull/17029)[mhenriks] 借助 v3 热插拔端口拓扑，修复升级过程中的 PCI 地址稳定性
- [[PR #17576]](https://github.com/kubevirt/kubevirt/pull/17576)[ema-aka-young] 缺陷修复：为恢复资源的 targetReadinessPolicy 字段添加了枚举校验。
- [[PR #17028]](https://github.com/kubevirt/kubevirt/pull/17028)[alaypatel07] VEP-10：修复 DRA 设备的缺陷，使 KubeVirt 实现对齐 KEP-5304
- [[PR #17235]](https://github.com/kubevirt/kubevirt/pull/17235)[tiraboschi] 修复：当客户机代理因非故障原因暂时不可达时，GuestAgentPing 存活探针/就绪探针不再导致 Kubernetes 重启 virt-launcher Pod；抑制范围覆盖实时迁移（pre-copy 目标端和 post-copy 源端）以及任何有意或短暂的 VM 暂停，例如用户暂停、快照、保存或转储。
- [[PR #16882]](https://github.com/kubevirt/kubevirt/pull/16882)[alromeros] 快照：为被排除的卷添加 PartialSnapshot 指示
- [[PR #16730]](https://github.com/kubevirt/kubevirt/pull/16730)[alromeros] 将 VMExport 功能门控升级为 GA
- [[PR #16675]](https://github.com/kubevirt/kubevirt/pull/16675)[csomani1] vGPU（mdev）现在支持实时迁移（限制为一个设备）
- [[PR #15298]](https://github.com/kubevirt/kubevirt/pull/15298)[Acedus] 缺陷修复：在线快照现在会正确包含应用到 VM 的实时变更。
- [[PR #16556]](https://github.com/kubevirt/kubevirt/pull/16556)[alaypatel07] VEP-10：更新 DRA 设备实现，改为从元数据文件读取，而不再从 VMI 状态读取
- [[PR #16930]](https://github.com/kubevirt/kubevirt/pull/16930)[Acedus] 引入增量备份拉取模式支持
- [[PR #16746]](https://github.com/kubevirt/kubevirt/pull/16746)[Barakmor1] 在 VmiMemoryOverheadReport 功能门控之后，于 VMI Status 上暴露内存开销
- [[PR #16350]](https://github.com/kubevirt/kubevirt/pull/16350)[orenc1] 新增用于退出 RBAC 聚合的配置选项
- [[PR #15958]](https://github.com/kubevirt/kubevirt/pull/15958)[Aseeef] 新增对 Intel TDX 机密计算平台上证明的支持
- [[PR #16877]](https://github.com/kubevirt/kubevirt/pull/16877)[ShellyKa13] 根据迁移优先级处理备份期间的迁移

### 缺陷修复 {#bug-fix_1}
- [[PR #18466]](https://github.com/kubevirt/kubevirt/pull/18466)[kubevirt-bot] 网络绑定插件边车现在可以使用 `NETWORK_BINDING_PLUGIN_NAME` 环境变量推断其注册名称。
- [[PR #18438]](https://github.com/kubevirt/kubevirt/pull/18438)[iholder101] 缺陷修复：使用 NFS 支撑存储的 VM 在 NFS 暂时不可用期间（例如 NFS 服务器故障转移）不再被错误地重启。
- [[PR #18404]](https://github.com/kubevirt/kubevirt/pull/18404)[kubevirt-bot] 现在会发出 `kubevirt_vmi_guest_os_panic_total` 指标
- [[PR #18405]](https://github.com/kubevirt/kubevirt/pull/18405)[kubevirt-bot] 缺陷修复：导入器 Pod 被 ValidatingAdmissionPolicy 'kubevirt-plugin-sidecar-subpath-policy' 拒绝
- [[PR #18343]](https://github.com/kubevirt/kubevirt/pull/18343)[kubevirt-bot] 缺陷修复：当 VMI 上不存在 panic 设备时，PreferredPanicDeviceModel 现在会创建一个，而不再静默忽略该偏好。
- [[PR #18299]](https://github.com/kubevirt/kubevirt/pull/18299)[lyarwood] 缺陷修复：将 IOMMUFD 套接字 accept 超时从 60 秒增加到 15 分钟，以防止在容器镜像拉取缓慢的环境中静默回退到旧版 VFIO。
- [[PR #18216]](https://github.com/kubevirt/kubevirt/pull/18216)[awels] 缺陷修复：修复取消去中心化实时迁移时未从迁移资源移除 finalizer 的缺陷
- [[PR #17846]](https://github.com/kubevirt/kubevirt/pull/17846)[ksimon1] virt-api：添加 X-Content-Type-Options: nosniff 安全响应头
- [[PR #18103]](https://github.com/kubevirt/kubevirt/pull/18103)[orelmisan] 修复目标 Pod 在准备完成后失败时，由陈旧的 VMI MigrationState 引起的迁移目标 Pod 无限循环。
- [[PR #18105]](https://github.com/kubevirt/kubevirt/pull/18105)[keinsword] 缺陷修复：修复了来自同一底层设备的卷存在多个附加 Pod 时的热插拔挂载解析，并防止 virt-handler 在同一次同步循环中把某个热插拔卷的源 Pod UID 复用到后续卷。
- [[PR #18143]](https://github.com/kubevirt/kubevirt/pull/18143)[akalenyu] 缺陷修复：存储实时迁移的文件系统到块设备失败，libvirt 错误 Code 84：不支持为磁盘的增量存储迁移预先创建存储目标
- [[PR #17960]](https://github.com/kubevirt/kubevirt/pull/17960)[dsanatar] 通过刷新令牌，修复 virtctl image-upload 命令的令牌过期问题。
- [[PR #17993]](https://github.com/kubevirt/kubevirt/pull/17993)[Acedus] 缺陷修复：带有 CBT 和 RWO 后端存储的实时迁移现在会在迁移后正确保留检查点
- [[PR #18052]](https://github.com/kubevirt/kubevirt/pull/18052)[vidit-bhat] 修复了 `pkg/hypervisor` 在 linux/{amd64,arm64,s390x} 以外的 GOOS/GOARCH 组合上的构建失败，该问题此前使下游打包者无法在 riscv64、ppc64le 和 386 等架构上交叉构建 virtctl。在这些架构上，`common.SchedSetScheduler` 现在会返回新的哨兵错误 `common.ErrUnsupportedRTScheduling`。
- [[PR #17984]](https://github.com/kubevirt/kubevirt/pull/17984)[Acedus] 增量备份现在通过合并 base:allocation 和 qemu:dirty-bitmap 上下文，区分被丢弃的块与数据变更，并跳过读出为零的脏 extent，从而减小传输大小。
- [[PR #17947]](https://github.com/kubevirt/kubevirt/pull/17947)[Barakmor1] 在 node-labeller 中使用 --expand-cpu-features 和 --supported-cpu-features，用于
- [[PR #17883]](https://github.com/kubevirt/kubevirt/pull/17883)[yaroslavborbat] 修复了失败或取消的实时迁移之后可能残留的目标端热插拔挂载泄漏。
- [[PR #17959]](https://github.com/kubevirt/kubevirt/pull/17959)[mhenriks] 修复 VMExport 目录处理程序中的符号链接遍历
- [[PR #17825]](https://github.com/kubevirt/kubevirt/pull/17825)[akalenyu] 缺陷修复：附加 Pod 删除/创建的无尽循环
- [[PR #17272]](https://github.com/kubevirt/kubevirt/pull/17272)[ksimon1] 功能：当 GuestAgentPing 探针失败时在 VMI 上记录 K8s 事件
- [[PR #17475]](https://github.com/kubevirt/kubevirt/pull/17475)[samt-ai] 修复 VM 强制重启中的竞态条件：Pod 可能在整个 terminationGracePeriodSeconds 期间卡在 Terminating 状态，而不能及时终止。现在会在触发重启之前修补 VMI 的 terminationGracePeriodSeconds，确保较短的宽限期始终得到遵守。
- [[PR #17031]](https://github.com/kubevirt/kubevirt/pull/17031)[suPer8Hu] N/A
- [[PR #17490]](https://github.com/kubevirt/kubevirt/pull/17490)[oshoval] 通过保留按设备的 claim/request 元组（包括共享 claim 名称但 request 不同的情况），修复了 virt-controller 对 GPU/HostDevice 资源的 DRA claim 渲染。
- [[PR #17527]](https://github.com/kubevirt/kubevirt/pull/17527)[mhenriks] 修复带有 PCI hostdev 的 VM 在热插拔块卷后无法重启的问题
- [[PR #17573]](https://github.com/kubevirt/kubevirt/pull/17573)[lyarwood] 缺陷修复：virt-operator 错误消息不再通过 %+v 转储整个资源结构体，从而避免资源创建失败时 KubeVirt CR 超过 etcd 3MB 对象大小限制
- [[PR #16697]](https://github.com/kubevirt/kubevirt/pull/16697)[dippydocus] 修复了使用未指定 MAC 地址的 bridge 绑定时，实时迁移之后客户机重启导致 DHCP 失败的问题。
- [[PR #17571]](https://github.com/kubevirt/kubevirt/pull/17571)[0xFelix] 修复了 virt-api 在构造 SubjectAccessReview 时截断深层子资源（vnc/screenshot、sev/*、evacuate/cancel）的问题，该问题会导致针对错误的子资源名称进行鉴权检查。
- [[PR #17425]](https://github.com/kubevirt/kubevirt/pull/17425)[shubham-pampattiwar] 修复：VirtualMachineBackup 的打印列（Type、CheckpointName）现在会在 kubectl 输出中正确显示
- [[PR #17297]](https://github.com/kubevirt/kubevirt/pull/17297)[Acedus] 修复：正确处理带有 qcow2 overlay 的磁盘的源解析，避免不正确的磁盘扩容以及错误的缓存/IO 模式检测。
- [[PR #16742]](https://github.com/kubevirt/kubevirt/pull/16742)[keinsword] 改进 Unmount() 清理：处理所有条目，并保留失败路径以便重试。
- [[PR #17102]](https://github.com/kubevirt/kubevirt/pull/17102)[Barakmor1] 缺陷修复：sync-controller 的 healthz 服务器和 virt-exportserver 现在会遵守 KubeVirt CR 中的 TLSConfiguration。
- [[PR #17251]](https://github.com/kubevirt/kubevirt/pull/17251)[iholder101] 修复了 /dev/kvm 不存在时软件仿真下的 VM 启动失败，原因是 cgroup 设备规则没有考虑仿真情形。
- [[PR #17042]](https://github.com/kubevirt/kubevirt/pull/17042)[awels] 修复了在去中心化实时迁移之后执行计算迁移时，迁移未报告成功的问题
- [[PR #16842]](https://github.com/kubevirt/kubevirt/pull/16842)[ksimon1] subresources.kubevirt.io 的 OpenApi V3 路径现已存在
- [[PR #16737]](https://github.com/kubevirt/kubevirt/pull/16737)[keinsword] 通过防止重复的 TargetFile 条目，修复 virt-handler 热插拔挂载目标记录不断增长的问题。
- [[PR #16871]](https://github.com/kubevirt/kubevirt/pull/16871)[awels] 缺陷修复：需要 enlightenment 的 VM 现在可以在去中心化实时迁移之后进行实时迁移
- [[PR #18567]](https://github.com/kubevirt/kubevirt/pull/18567)[kubevirt-bot] 缺陷修复：向 virt-launcher 容器暴露 VFIO cdev 设备（/dev/vfio/devices/vfioN），修复带有 SMMUv3 的 ARM64 系统上基于 IOMMUFD 的 GPU 直通。
- [[PR #18539]](https://github.com/kubevirt/kubevirt/pull/18539)[kubevirt-bot] 缺陷修复：VMI 在附加 Pod 之间交换 RWO PVC 时，热插拔卷分离死锁。
- [[PR #18507]](https://github.com/kubevirt/kubevirt/pull/18507)[kubevirt-bot] 修复了网络接口绑定缺少校验的问题：没有绑定或存在多个绑定的接口现在会在准入时被拒绝，而不再是在运行时静默失败。

### 弃用 {#deprecation_1}
- [[PR #17809]](https://github.com/kubevirt/kubevirt/pull/17809)[michalskrivanek] cgroup v1 支持现已弃用，计划在下一个版本中移除
- [[PR #17065]](https://github.com/kubevirt/kubevirt/pull/17065)[avlitman] 多条记录规则已弃用并改用新名称，以符合记录规则命名约定。kubevirt_vm_created_total 记录规则和 kubevirt_vm_created_by_pod_total 指标已被完全弃用

### SIG-compute {#sig-compute_1}
- [[PR #18412]](https://github.com/kubevirt/kubevirt/pull/18412)[kubevirt-bot] 为 Grace I/O Virtualization 设置显式的 SMMUv3 地址能力默认值。
- [[PR #18356]](https://github.com/kubevirt/kubevirt/pull/18356)[kubevirt-bot] 为基于 VolumeSnapshot 的 DataVolume、DataVolumeTemplate 和 DataSource 支持 inferFromVolume
- [[PR #18011]](https://github.com/kubevirt/kubevirt/pull/18011)[fanzhangio] 启用 Alpha 阶段的 GraceIOVirtualization 功能门控后，KubeVirt 现在会为 NVIDIA Grace GPU 直通配置 SMMUv3/IOMMUFD、ACPI Generic Initiator NUMA 拓扑、NUMA 距离映射以及自动 PCI 64 位孔洞大小调整。
- [[PR #18004]](https://github.com/kubevirt/kubevirt/pull/18004)[avlitman] 新增 VMNonRecoverableOSPanic 告警
- [[PR #18116]](https://github.com/kubevirt/kubevirt/pull/18116)[brianmcarey] 使用 go 1.26.4 构建 KubeVirt
- [[PR #18161]](https://github.com/kubevirt/kubevirt/pull/18161)[machadovilaca] 新增 kubevirt_vmi_gpu_info 指标，用于把 GPU UUID 与 VMI 关联起来
- [[PR #18252]](https://github.com/kubevirt/kubevirt/pull/18252)[kubevirt-bot] 将 virt-template 更新到 v0.2.2
- [[PR #18112]](https://github.com/kubevirt/kubevirt/pull/18112)[xpivarc] SRIOV vGPU 现在支持显示
- [[PR #17818]](https://github.com/kubevirt/kubevirt/pull/17818)[fossedihelm] MigrationPriorityQueue 功能门控已升级为 GA。
- [[PR #18228]](https://github.com/kubevirt/kubevirt/pull/18228)[kubevirt-bot] 将 common-instancetypes 捆绑包更新到 v1.7.0
- [[PR #18220]](https://github.com/kubevirt/kubevirt/pull/18220)[0xFelix] 将 virt-template 更新到 v0.2.1
- [[PR #18188]](https://github.com/kubevirt/kubevirt/pull/18188)[machadovilaca] 新增带有 HTTP 端点的 VMStatsHandler，用于采集 VM 统计信息
- [[PR #17998]](https://github.com/kubevirt/kubevirt/pull/17998)[fanzhangio] 请求 NVIDIA Grace GPU PCI 直通资源的 VMI 现在会经过准入校验。此类 VMI 需要 GraceIOVirtualization、PCINUMAAwareTopology 和 IOMMUFD 功能门控、arm64 架构、virt 机器类型以及专用 CPU 放置；启用 GraceIOVirtualization 时，含糊的 NVIDIA 通配 PCI 选择器会被拒绝。
- [[PR #17886]](https://github.com/kubevirt/kubevirt/pull/17886)[fanzhangio] 对于使用 numa.guestMappingPassthrough 的 VMI，当主机设备无法放置到与 NUMA 对齐的 PCIe 拓扑上时，PCI NUMA 感知放置（PCINUMAAwareTopology）现在会报错失败。非直通 VMI 保留现有的回退行为。
- [[PR #17505]](https://github.com/kubevirt/kubevirt/pull/17505)[alromeros] 为带有 SCSI 持久预留的 VM 启用实时迁移
- [[PR #17956]](https://github.com/kubevirt/kubevirt/pull/17956)[fossedihelm] 在 IOMMUFD Alpha 功能门控之后，向 virt-handler 添加 IOMMUFD 设备插件。启用后，virt-controller 会为每个 launcher Pod 请求 devices.kubevirt.io/iommufd。没有 /dev/iommu 的节点（内核 <6.2）会报告不健康的设备，使 Pod 无法调度到这些节点。此功能还会发出域级别的 <iommufd enabled='yes' fdgroup='iommu'/>，并使用 virDomainFDAssociate；这两者都要求 libvirt/QEMU 栈支持基于 fdgroup 的 IOMMUFD，目前为 libvirt >= 12.2。仅在目标节点和 virt-launcher 镜像提供兼容的内核、libvirt 和 QEMU 支持的集群上启用此门控。
- [[PR #17891]](https://github.com/kubevirt/kubevirt/pull/17891)[vladikr] 为 Grace I/O Virtualization 添加基础类型：GraceIOVirtualization 功能门控、IOMMU domain XML schema、HostDevice 的 IOMMU/ACPI 扩展，以及针对零内存 cell 的 NUMACell 指针变更。
- [[PR #16990]](https://github.com/kubevirt/kubevirt/pull/16990)[lyarwood] 将端口转发和远程执行从硬编码的 SPDY 迁移为以 WebSocket 为主、SPDY 作为回退，修复了在启用 PortForwardWebsockets 功能门控的 Kubernetes 1.31+ 集群上间歇性的流创建失败。
- [[PR #18076]](https://github.com/kubevirt/kubevirt/pull/18076)[jcanocan] 将 virt-template 更新到 v0.2.0
- [[PR #17989]](https://github.com/kubevirt/kubevirt/pull/17989)[UdayYendva] 将 github.com/moby/spdystream 从 v0.5.0 升级到 v0.5.1，以解决 CVE-2026-35469（GHSA-pc3f-x583-g7j2）。
- [[PR #17836]](https://github.com/kubevirt/kubevirt/pull/17836)[iholder101] 新增 Prometheus 计数器指标 kubevirt_vmi_guest_os_panic_total，用于按 VMI 跟踪客户机操作系统 panic 事件，并带有 panic 类型和 hyper-v bugcheck 代码标签
- [[PR #17757]](https://github.com/kubevirt/kubevirt/pull/17757)[Acedus] 修复：在迁移相关代码中处理带有 qcow2 overlay 的磁盘
- [[PR #17798]](https://github.com/kubevirt/kubevirt/pull/17798)[SamAlber] 修复了 virt-handler 的 GetLauncherClient 中的 gRPC 连接泄漏。当多个控制器竞相为同一个 VMI 创建连接时，该泄漏会导致内存无界增长、套接字堆积和 goroutine 泄漏。
- [[PR #17644]](https://github.com/kubevirt/kubevirt/pull/17644)[kubevirt-bot] 将 virt-template 更新到 v0.1.8
- [[PR #17405]](https://github.com/kubevirt/kubevirt/pull/17405)[iholder101] 默认启用所有 Beta 功能门控。用户可以将个别 Beta 功能加入 KubeVirt CR 的 `spec.configuration.developerConfiguration.disabledFeatureGates` 来选择停用。每个版本的产物中都会加入一份包含所有非 GA 功能门控的功能门控报告。
- [[PR #17805]](https://github.com/kubevirt/kubevirt/pull/17805)[lyarwood] 修复了多设备 VFIO 直通 VM 因 "cannot limit locked memory" 而无法启动的问题：按每设备内存锁定扩大 virt-handler 的 memlock rlimit，与 libvirt 在 v8.7.0 中引入的计算方式一致。
- [[PR #17496]](https://github.com/kubevirt/kubevirt/pull/17496)[fanzhangio] --additional-launcher-annotations-sync 和 --additional-launcher-labels-sync 标志现在支持通过末尾 '*' 后缀实现前缀通配（例如 'vendor.io/*'），从而可以把所有匹配的标签/注解从 VM 模板传播到 VMI 和 virt-launcher Pod，而无需逐个枚举键。
- [[PR #17049]](https://github.com/kubevirt/kubevirt/pull/17049)[frenzyfriday] 升级 LiveUpdateNADRef 功能门控
- [[PR #17015]](https://github.com/kubevirt/kubevirt/pull/17015)[Ronilerr] 为 virt 组件补充缺失的指标、记录规则和告警
- [[PR #17557]](https://github.com/kubevirt/kubevirt/pull/17557)[dshchedr] 修复了非 root virt-launcher 的 GuestPanicked 事件详情
- [[PR #17295]](https://github.com/kubevirt/kubevirt/pull/17295)[avlitman] 新增指标 kubevirt_vmi_sync_total，用于跟踪控制器同步某个 VMI 的次数。
- [[PR #17398]](https://github.com/kubevirt/kubevirt/pull/17398)[dasionov] 缺陷修复：virt-handler 现在会在 `domain-notify.sock` 被删除时检测到这一情况，并自动重启 notify 服务器。
- [[PR #16890]](https://github.com/kubevirt/kubevirt/pull/16890)[xpivarc] KubeVirt 的 PCI 设备插件现在支持直通预先设置的 VF（vGPU）
- [[PR #16853]](https://github.com/kubevirt/kubevirt/pull/16853)[dankenigsberg] 带有后端存储卷的 VM 使用并报告的卷名为 `persistent-state-for-this-vm`，而不再尝试把 vm 名称嵌入卷名。
- [[PR #17139]](https://github.com/kubevirt/kubevirt/pull/17139)[dsanatar] 修复热插拔卷状态卡在 Detaching 阶段的问题
- [[PR #17071]](https://github.com/kubevirt/kubevirt/pull/17071)[samt-ai] 使 --vnc-path 对 --vnc-type 变为可选，从而允许自动从 $PATH 查找 VNC 查看器二进制文件
- [[PR #17109]](https://github.com/kubevirt/kubevirt/pull/17109)[dasionov] 缺陷修复：virt-handler 的 domain-notify 服务器意外退出时将其重启。
- [[PR #17106]](https://github.com/kubevirt/kubevirt/pull/17106)[iholder101] 允许在 post-copy 迁移中使用 multifd（并行迁移线程）
- [[PR #16783]](https://github.com/kubevirt/kubevirt/pull/16783)[dasionov] 修复了 ARM64 客户机 VM 内部看不到 SMBIOS 系统信息的问题
- [[PR #16927]](https://github.com/kubevirt/kubevirt/pull/16927)[harshitgupta1337] 为多 hypervisor 支持接口添加 MSHV 后端。
- [[PR #16817]](https://github.com/kubevirt/kubevirt/pull/16817)[ShellyKa13] 处理带有 CBT 和备份检查点的迁移
- [[PR #16632]](https://github.com/kubevirt/kubevirt/pull/16632)[mresvanis] 在 PCINUMAAwareTopology 功能门控（Alpha）之后，为 GPU 和主机设备新增 PCIe NUMA 感知拓扑放置。启用后，设备会自动放置到与其 NUMA 亲和性匹配的 PCIe expander 总线上，以提升性能。
- [[PR #16884]](https://github.com/kubevirt/kubevirt/pull/16884)[EdDev] 'virtctl expose' 现在创建的 Service 带有指向被暴露资源的 ownerReference。
- [[PR #16412]](https://github.com/kubevirt/kubevirt/pull/16412)[frenzyfriday] 允许用户通过实时迁移更新运行中 VM 上某个网络的 NAD 引用（networkName）。
- [[PR #16604]](https://github.com/kubevirt/kubevirt/pull/16604)[dsanatar] 升级 ExpandDisk 功能门控
- [[PR #16399]](https://github.com/kubevirt/kubevirt/pull/16399)[Acedus] 修复：复用备份名称时防止陈旧的 VMI 备份状态更新
- [[PR #16786]](https://github.com/kubevirt/kubevirt/pull/16786)[alromeros] 允许通过注解禁用 virt-launcher 中的 Velero 钩子
- [[PR #18532]](https://github.com/kubevirt/kubevirt/pull/18532)[kubevirt-bot] virt-launcher Pod 会随迁移对象一起被清理

### SIG-storage {#sig-storage_1}
- [[PR #18065]](https://github.com/kubevirt/kubevirt/pull/18065)[0xFelix] Template 功能门控已升级为 Beta，并且现在默认启用。除非被显式禁用，否则会自动部署 Virt-template 组件。
- [[PR #16932]](https://github.com/kubevirt/kubevirt/pull/16932)[dsanatar] 将 HotplugVolumes 和 DeclarativeHotplugVolumes 升级为 Beta
- [[PR #17699]](https://github.com/kubevirt/kubevirt/pull/17699)[dsanatar] 仅当源不是 PVC 时，才用 VM 清单填充 vmexport
- [[PR #17407]](https://github.com/kubevirt/kubevirt/pull/17407)[dsanatar] 为恢复 PVC 保留注解
- [[PR #16846]](https://github.com/kubevirt/kubevirt/pull/16846)[Aneesh-Hegde] 修复 PVC 名称过长时的 VMExport 失败

### SIG-network {#sig-network_1}
- [[PR #18433]](https://github.com/kubevirt/kubevirt/pull/18433)[kubevirt-bot] 现在可以通过 `--primary-network-binding-plugin` 测试标志，使用自定义网络绑定插件运行网络一致性测试。
- [[PR #17845]](https://github.com/kubevirt/kubevirt/pull/17845)[orelmisan] 网络绑定插件现在可以从 network-info downward API 注解中读取接口 MTU。
- [[PR #17536]](https://github.com/kubevirt/kubevirt/pull/17536)[nirdothan] 修复了在带有 IPv6 IPAM 的网络上使用 bridge 绑定时，VMI 状态报告的是 Pod 的 IPv6 地址而不是客户机地址的问题。
- [[PR #17315]](https://github.com/kubevirt/kubevirt/pull/17315)[frenzyfriday] 修复了 Live NAD Ref Update 功能中的缺陷：启用 LiveNADRefUpdate FG 时，没有任何接口/网络的 VM 无法启动。
- [[PR #17041]](https://github.com/kubevirt/kubevirt/pull/17041)[orelmisan] 修复了 virt-controller 与 virt-handler 之间的 VMI 状态无限更新循环，该循环发生在 VMI spec 把主网络接口列在辅助接口之后时。

### SIG-observability {#sig-observability_1}
- [[PR #18141]](https://github.com/kubevirt/kubevirt/pull/18141)[machadovilaca] 启用 VMStatsCollector 时跳过 PrometheusRule 生成
- [[PR #17130]](https://github.com/kubevirt/kubevirt/pull/17130)[sradco] 修复实时迁移期间因 kubevirt_vmi_info 序列重复而导致 VirtualMachineStuckOnNode 和 VMCannotBeEvicted 告警失效的问题
- [[PR #16806]](https://github.com/kubevirt/kubevirt/pull/16806)[machadovilaca] 使用已定义的 Deployment 副本数作为触发低计数告警的基准
- [[PR #16865]](https://github.com/kubevirt/kubevirt/pull/16865)[machadovilaca] 从 kubevirt_allocatable_nodes 中减去不可调度节点

### 其他 {#other_1}
- [[PR #18446]](https://github.com/kubevirt/kubevirt/pull/18446)[kubevirt-bot] vep-10：将 GPUsWithDRA 和 HostDevicesWithDRA 移至 Beta
- [[PR #18445]](https://github.com/kubevirt/kubevirt/pull/18445)[kubevirt-bot] 为 DRA 功能新增 E2E 测试支持。
- [[PR #17837]](https://github.com/kubevirt/kubevirt/pull/17837)[keinsword] Nebius 自 2023 年起被列为 KubeVirt 的采用者。
- [[PR #17540]](https://github.com/kubevirt/kubevirt/pull/17540)[orenc1] 新增 RBAC 角色聚合升级 e2e 测试
- [[PR #17770]](https://github.com/kubevirt/kubevirt/pull/17770)[jschintag] IBM Secure Execution 现已正式可用
- [[PR #18129]](https://github.com/kubevirt/kubevirt/pull/18129)[iholder101] 修复了 kubevirt.io:view ClusterRole 中缺少 plugin.kubevirt.io 的 RBAC 规则的问题，恢复了 cluster-reader 对 Plugin 资源的访问。
- [[PR #17976]](https://github.com/kubevirt/kubevirt/pull/17976)[ethan-gallant] virt-operator：customizeComponents.Patches 现在会应用到 install-strategy Job（使用 resourceType "Job" 和 resourceName "virt-operator-install-strategy" 进行匹配）。该 Job 生成的名称已从 "kubevirt-<id>-job" 改为 "virt-operator-install-strategy-"。
- [[PR #17623]](https://github.com/kubevirt/kubevirt/pull/17623)[MarSik] RebootPolicy 功能已升级为 Beta 状态。
- [[PR #17422]](https://github.com/kubevirt/kubevirt/pull/17422)[jean-edouard] libvirt 已升级到 v11.10
- [[PR #17996]](https://github.com/kubevirt/kubevirt/pull/17996)[ksimon1] virt-operator：修复切换可选功能门控之后的 DeploymentInProgress
- [[PR #16599]](https://github.com/kubevirt/kubevirt/pull/16599)[dasionov] 将 `VideoConfig` FG 升级为正式可用
- [[PR #17548]](https://github.com/kubevirt/kubevirt/pull/17548)[Barakmor1] 为 virt-exportproxy 和 virt-synchronization-controller 的 Deployment 添加容器级 SecurityContext
- [[PR #17755]](https://github.com/kubevirt/kubevirt/pull/17755)[dasionov] 修复：跨命名空间实时迁移现在可以在 IPv6 集群上工作
- [[PR #17691]](https://github.com/kubevirt/kubevirt/pull/17691)[vishnuchalla] NA
- [[PR #17599]](https://github.com/kubevirt/kubevirt/pull/17599)[Sreeja1725] 添加 1.8 版本的 perf-scale 基准测试数据
- [[PR #17488]](https://github.com/kubevirt/kubevirt/pull/17488)[alaypatel07] 将 /var/lib/kubelet 挂载从 Bidirectional 改为 HostToContainer
- [[PR #17497]](https://github.com/kubevirt/kubevirt/pull/17497)[sbiradar10] 通过将 grpc 升级到 1.79.3 来修复 CVE-2026-33186
- [[PR #17512]](https://github.com/kubevirt/kubevirt/pull/17512)[0xFelix] 从 kubevirt.io:edit 中移除 vnc/screenshot
- [[PR #16514]](https://github.com/kubevirt/kubevirt/pull/16514)[varunrsekar] `PanicDevices` 功能已升级为 GA，不再需要启用相关联的功能门控。
- [[PR #17285]](https://github.com/kubevirt/kubevirt/pull/17285)[dhiller] 维护：回退 bazel 服务器网络变更监控
- [[PR #16885]](https://github.com/kubevirt/kubevirt/pull/16885)[akalenyu] 维护：使 bazel 服务器容器在网络变更时重新加载
## v1.8.0 {#v180}

发布日期：2026 年 3 月 25 日，星期三

KubeVirt v1.8 面向 Kubernetes v1.35 构建，并额外支持此前两个版本。更多信息请参阅 [KubeVirt 支持矩阵](https://github.com/kubevirt/sig-release/blob/main/releases/k8s-support-matrix.md)。

要查看为本版本做出贡献的优秀人员名单，请参阅 [KubeVirt v1.8.0 发行标签](https://github.com/kubevirt/kubevirt/releases/tag/v1.8.0)。

### API 变更 {#api-change_2}
- [[PR #17077]](https://github.com/kubevirt/kubevirt/pull/17077)[kubevirt-bot] VEP-10：更新 DRA 设备实现，改为从元数据文件读取，不再从 VMI 状态读取
- [[PR #17017]](https://github.com/kubevirt/kubevirt/pull/17017)[kubevirt-bot] 受 VmiMemoryOverheadReport 功能门控保护，在 VMI 状态中公开内存开销
- [[PR #16993]](https://github.com/kubevirt/kubevirt/pull/16993)[frenzyfriday] 允许用户通过实时迁移更新运行中 VM 上某一网络的 NAD 引用（networkName）。
- [[PR #16977]](https://github.com/kubevirt/kubevirt/pull/16977)[orenc1] 新增一个配置选项，用于选择退出 RBAC 聚合
- [[PR #16687]](https://github.com/kubevirt/kubevirt/pull/16687)[0xFelix] 功能：virtctl 新增与 virt-template / VirtualMachineTemplate 相关的命令（process、create 和 convert）
- [[PR #16662]](https://github.com/kubevirt/kubevirt/pull/16662)[mhenriks] VEP 165：Containerpath 卷
- [[PR #16426]](https://github.com/kubevirt/kubevirt/pull/16426)[Acedus] 处理 CBT 备份的中止请求与失败
- [[PR #16820]](https://github.com/kubevirt/kubevirt/pull/16820)[nirdothan] 支持使用核心 passt 绑定进行无缝迁移（beta）。
- [[PR #16655]](https://github.com/kubevirt/kubevirt/pull/16655)[0xFelix] 新增通过 virt-operator 部署 virt-template 的支持（VEP76）
- [[PR #16802]](https://github.com/kubevirt/kubevirt/pull/16802)[lyarwood] `PrefixTargetName` 现在可用作 `VolumeNamePolicy`，供 `VirtualMachineClone` 使用
- [[PR #16579]](https://github.com/kubevirt/kubevirt/pull/16579)[MarSik] 可以使用 VMI.spec.domain.rebootPolicy 字段来控制域如何处理来自 VM 内部的重启。默认情况下，虚拟机监控程序会在后台静默处理重启；用户也可以选择更可见的行为，由虚拟机监控程序终止该域，再由 KubeVirt 根据 runStrategy 规则处理重启。
- [[PR #16653]](https://github.com/kubevirt/kubevirt/pull/16653)[noamasu] 将 QuiesceFailed 替换为 QuiesceTimeout 指示，并增加 60 秒的 Velero 备份前钩子超时，以便更好地应对 Windows VSS 的限制。
- [[PR #16448]](https://github.com/kubevirt/kubevirt/pull/16448)[ShellyKa13] 通过在 libvirt 中重新定义检查点元数据，使 VM 重启后仍支持增量备份
- [[PR #16370]](https://github.com/kubevirt/kubevirt/pull/16370)[iholder101] 现在可以使用 `kv.spec.configuration.developerConfiguration.disabledFeatureGates` 显式禁用功能门控。
- [[PR #16366]](https://github.com/kubevirt/kubevirt/pull/16366)[elliot-gustafsson] 如果目标是块设备，则让 libvirt 查询实际磁盘大小，以确保与加密磁盘兼容。
- [[PR #16512]](https://github.com/kubevirt/kubevirt/pull/16512)[awels] 去中心化实时迁移现在在 VMI 和 VMIM 上具有单独的状况，用于指示任何问题
- [[PR #16489]](https://github.com/kubevirt/kubevirt/pull/16489)[lyarwood] 为 VirtualMachineRestore 新增 `PrefixTargetName` VolumeRestorePolicy，该策略使用 `{targetVMName}-{volumeName}` 格式创建恢复后的卷名称。这样可以提供可预测且易读的名称，同时避免将快照恢复到不同目标 VM 时发生名称冲突。
- [[PR #16404]](https://github.com/kubevirt/kubevirt/pull/16404)[iholder101] 为 Hyperv TLBFlush 补充缺失的 "Direct" 和 "Extended" 选项
- [[PR #16585]](https://github.com/kubevirt/kubevirt/pull/16585)[Sreeja1725] 在更新期间保留 VM 特定字段
- [[PR #16326]](https://github.com/kubevirt/kubevirt/pull/16326)[harshitgupta1337] 引入 `HypervisorConfigurations` 字段，该字段位于 `KubevirtConfiguration` CRD 中。
- [[PR #16220]](https://github.com/kubevirt/kubevirt/pull/16220)[lyarwood] `DisableMDEVConfiguration` 功能门控现已弃用，并将在未来版本中移除，由新的可配置项 `kubevirt.spec.configuration.mediatedDevicesConfiguration.enabled` 取代
- [[PR #16488]](https://github.com/kubevirt/kubevirt/pull/16488)[lyarwood] VirtualMachineClone API 现在包含 `VolumeNamePolicy` 字段，用于控制卷克隆行为。
- [[PR #15975]](https://github.com/kubevirt/kubevirt/pull/15975)[sradco] kubevirt_vmi_migration_data_total_bytes 已弃用，改为使用 kubevirt_vmi_migration_data_bytes_total，以符合指标命名约定。
- [[PR #16285]](https://github.com/kubevirt/kubevirt/pull/16285)[ShellyKa13] 增加对 VM 增量备份的支持
- [[PR #15815]](https://github.com/kubevirt/kubevirt/pull/15815)[Dsanatar] 新增临时热插拔卷指标和告警
- [[PR #15992]](https://github.com/kubevirt/kubevirt/pull/15992)[Aseeef] * 修复了套接字设备中的一个缺陷，该缺陷导致使用持久预留功能的集群无法正确更新其当前健康状态。
- [[PR #16174]](https://github.com/kubevirt/kubevirt/pull/16174)[dominikholler] 将依赖 golang.org/x/crypto 更新至 v0.45.0
- [[PR #16081]](https://github.com/kubevirt/kubevirt/pull/16081)[ShellyKa13] VMBackup：引入新的 VM 备份 API
- [[PR #16173]](https://github.com/kubevirt/kubevirt/pull/16173)[dominikholler] 将依赖 github.com/opencontainers/selinux 更新至 v1.13.0
- [[PR #16122]](https://github.com/kubevirt/kubevirt/pull/16122)[dasionov] 记录 `spec.runStrategy` 的允许取值。
- [[PR #15922]](https://github.com/kubevirt/kubevirt/pull/15922)[ShellyKa13] 引入新 API UtilityVolumes，作为直接挂接到 virt-launcher 的机制
- [[PR #14892]](https://github.com/kubevirt/kubevirt/pull/14892)[xpivarc] 为能够运行专用 VM 的节点通告 kubevirt.io/cpumanager 标签。
- [[PR #15969]](https://github.com/kubevirt/kubevirt/pull/15969)[Dsanatar] 从运行中的 VM 分离 CD-ROM 时添加 RestartRequired
- [[PR #15957]](https://github.com/kubevirt/kubevirt/pull/15957)[xpivarc] 引入新的子资源 `/evacuate/cancel` 以及 `virtctl evacuate-cancel` 命令，允许用户取消虚拟机实例（VMI）的疏散过程。该操作会清除 VMI 状态中的 `evacuationNodeName` 字段，停止自动创建迁移资源，并完全中止驱逐周期。
- [[PR #15166]](https://github.com/kubevirt/kubevirt/pull/15166)[Sreeja1725] 引入 pool.kubevirt.io/v1beta1
- [[PR #15409]](https://github.com/kubevirt/kubevirt/pull/15409)[noamasu] VMSnapshot：新增 SourceIndications 状态字段，用于列出带说明的快照指示，使含义更加清晰。
- [[PR #15767]](https://github.com/kubevirt/kubevirt/pull/15767)[awels] 缺陷修复：去中心化实时迁移没有准确地使用迁移限制
- [[PR #15638]](https://github.com/kubevirt/kubevirt/pull/15638)[Sreeja1725] VMPool：增加对自愈策略的支持
- [[PR #15604]](https://github.com/kubevirt/kubevirt/pull/15604)[Sreeja1725] VMpool：增加缩容策略支持，包含 Proactive、opportunistic 模式以及 statePreservation
- [[PR #15529]](https://github.com/kubevirt/kubevirt/pull/15529)[Yu-Jack] 支持 v0.32.5 代码生成器
- [[PR #17058]](https://github.com/kubevirt/kubevirt/pull/17058)[mhenriks] 修复使用 v3 热插拔端口拓扑时跨升级的 PCI 地址稳定性
- [[PR #17061]](https://github.com/kubevirt/kubevirt/pull/17061)[ShellyKa13] 修复：重用备份名称时，防止写入过期的 VMI 备份状态
- [[PR #17075]](https://github.com/kubevirt/kubevirt/pull/17075)[kubevirt-bot] 根据迁移优先级处理备份期间的迁移

### 缺陷修复 {#bug-fix_2}
- [[PR #16833]](https://github.com/kubevirt/kubevirt/pull/16833)[akalenyu] 缺陷修复：使用 Google Cloud NetApp Volumes 时存储迁移失败
- [[PR #16791]](https://github.com/kubevirt/kubevirt/pull/16791)[lyarwood] 缺陷修复：virt-operator Deployment 上的 VIRT_*_IMAGE 环境变量覆盖现在会正确传播到各个组件 Deployment（virt-controller、virt-handler 等）。此前，由于镜像取值被排除在安装策略的 Deployment ID 哈希之外，更改这些环境变量不会生效。
- [[PR #16621]](https://github.com/kubevirt/kubevirt/pull/16621)[akalenyu] 缺陷修复：vmsnapshot：报告正在被删除的卷
- [[PR #16229]](https://github.com/kubevirt/kubevirt/pull/16229)[noamasu] 缺陷修复：为内存转储所创建的 PVC 添加标签，以支持 CDI WebhookPvcRendering
- [[PR #16637]](https://github.com/kubevirt/kubevirt/pull/16637)[awels] 缺陷修复：volumeMode 不同的卷之间的去中心化实时迁移现在可以成功完成
- [[PR #16491]](https://github.com/kubevirt/kubevirt/pull/16491)[lyarwood] virt-operator 现在会配置客户端速率限制（默认：200 QPS / 400 burst），以便在处理大量对象时提高协调性能。可以通过 --client-qps 和 --client-burst 标志，或 VIRT_OPERATOR_CLIENT_QPS 和 VIRT_OPERATOR_CLIENT_BURST 环境变量自定义速率限制。
- [[PR #16527]](https://github.com/kubevirt/kubevirt/pull/16527)[lukashes] 修复了在更改详细级别之后，client-go 日志输出中缺少对象上下文的问题。
- [[PR #16463]](https://github.com/kubevirt/kubevirt/pull/16463)[akalenyu] 缺陷修复：迁移指标缺失
- [[PR #16024]](https://github.com/kubevirt/kubevirt/pull/16024)[Sreeja1725] 扩大 KWOK 性能测试的规模，并添加 virt-controller 队列指标
- [[PR #16355]](https://github.com/kubevirt/kubevirt/pull/16355)[Sreeja1725] 改进布尔标志的格式化，以便能够正确解析。
- [[PR #16343]](https://github.com/kubevirt/kubevirt/pull/16343)[ShellyKa13] 缺陷修复：功能门控被禁用时，不要修改 VMI 的 CBT 状态
- [[PR #16060]](https://github.com/kubevirt/kubevirt/pull/16060)[dasionov] 缺陷修复：阻止跨厂商迁移
- [[PR #15868]](https://github.com/kubevirt/kubevirt/pull/15868)[frank-gen] 当 appendIndexToSecretRefs 设置为 true 时，VirtualMachinePool 现在会正确地向 CloudInit Secret 引用追加索引，从而使池中每个 VM 都能拥有唯一的 cloud-init 配置。
- [[PR #15716]](https://github.com/kubevirt/kubevirt/pull/15716)[awels] 去中心化实时迁移的失败会在源端与目标端之间正确传播
- [[PR #16050]](https://github.com/kubevirt/kubevirt/pull/16050)[xpivarc] 缺陷修复：KubeVirt.spec.imagetag 安装再次可以正常工作
- [[PR #17006]](https://github.com/kubevirt/kubevirt/pull/17006)[kubevirt-bot] 缺陷修复：需要 enlightenment 的 VM 在经过去中心化实时迁移之后，现在可以进行实时迁移
- [[PR #17267]](https://github.com/kubevirt/kubevirt/pull/17267)[kubevirt-bot] 缺陷修复：在意外退出时重启 virt-handler 的 domain-notify 服务器。
- [[PR #17236]](https://github.com/kubevirt/kubevirt/pull/17236)[kubevirt-bot] 修复 PVC 名称过长导致的 VMExport 失败

### 弃用 {#deprecation_2}
- [[PR #16023]](https://github.com/kubevirt/kubevirt/pull/16023)[lyarwood] `MultiArchitecture` 功能门控已弃用，不再用于判断准入 Webhook 是否应拒绝架构与控制平面不同的 VirtualMachine

### SIG-compute {#sig-compute_2}
- [[PR #16952]](https://github.com/kubevirt/kubevirt/pull/16952)[kubevirt-bot] 允许通过注解在 virt-launcher 中禁用 Velero 钩子
- [[PR #17018]](https://github.com/kubevirt/kubevirt/pull/17018)[mresvanis] 受 PCINUMAAwareTopology 功能门控（Alpha）保护，为 GPU 和主机设备添加可感知 PCIe NUMA 的拓扑放置。启用后，设备会自动放置到与其 NUMA 亲和性匹配的 PCIe 扩展总线上，以提升性能。
- [[PR #16821]](https://github.com/kubevirt/kubevirt/pull/16821)[nirdothan] 在功能门控的条件下，从 virt-controller 的 ClusterRole 中移除 network-attachment-definition 的 get 权限。
- [[PR #16528]](https://github.com/kubevirt/kubevirt/pull/16528)[Acedus] 修复：启用 CBT 的实时迁移不再因虚拟磁盘大小评估错误而失败。
- [[PR #16582]](https://github.com/kubevirt/kubevirt/pull/16582)[lyarwood] 通过 KUBEVIRT_CENTOS_STREAM_VERSION 环境变量，增加对 CentOS Stream 10 构建的初步支持；在 v1.9.0 及更高版本之前，这些构建不会经过测试
- [[PR #16666]](https://github.com/kubevirt/kubevirt/pull/16666)[iholder101] 将客户机 panic 公开为 Kubernetes 事件
- [[PR #16778]](https://github.com/kubevirt/kubevirt/pull/16778)[Acedus] 修复：如果域因 I/O 错误而暂停，则不会处理域作业完成事件。
- [[PR #16642]](https://github.com/kubevirt/kubevirt/pull/16642)[orelmisan] 仍对辅助接口保留旧式序号命名方案的现有 VM 会自动升级，且无需重启。
- [[PR #16705]](https://github.com/kubevirt/kubevirt/pull/16705)[kubevirt-bot] 将 common-instancetypes 捆绑包更新至 v1.6.0
- [[PR #16600]](https://github.com/kubevirt/kubevirt/pull/16600)[woojoong88] 修复块卷热插拔导致 autoattachVSOCK 失效的问题
- [[PR #16510]](https://github.com/kubevirt/kubevirt/pull/16510)[ShellyKa13] 将 CBT 应用于热插拔卷
- [[PR #16212]](https://github.com/kubevirt/kubevirt/pull/16212)[Barakmor1] 添加目标端预迁移钩子系统
- [[PR #16498]](https://github.com/kubevirt/kubevirt/pull/16498)[lyarwood] 修复实例类型控制器修改 Status 时，VM 协调过程中出现的 ResourceVersion 冲突。实例类型控制器现在会正确地从 PatchStatus 响应中传播 ResourceVersion，从而避免后续 UpdateStatus 调用发生冲突。
- [[PR #15113]](https://github.com/kubevirt/kubevirt/pull/15113)[alromeros] 为内存转储 PVC 添加标签，以支持 CDI WebhookPvcRendering
- [[PR #16329]](https://github.com/kubevirt/kubevirt/pull/16329)[dasionov] 当 VM 与对应的 VMI 已经共享相同的固件 UUID 时，避免产生错误的 restart-required 状况。
- [[PR #16280]](https://github.com/kubevirt/kubevirt/pull/16280)[Dsanatar] 弃用 virtctl add/remove volume 的 --persist 标志
- [[PR #15821]](https://github.com/kubevirt/kubevirt/pull/15821)[SamAlber] 为暂停和取消暂停 VM 的操作添加事件日志，以便与 reset 等其他 VM 生命周期事件保持一致
- [[PR #16159]](https://github.com/kubevirt/kubevirt/pull/16159)[Dsanatar] 不要使用已标记为删除的附加 Pod 来更新热插拔卷状态。
- [[PR #15949]](https://github.com/kubevirt/kubevirt/pull/15949)[xpivarc] 迁移使用专用证书进行 mTLS。
- [[PR #16049]](https://github.com/kubevirt/kubevirt/pull/16049)[fossedihelm] 修复：出现节点压力时，KSM 会在 3 分钟内启用
- [[PR #15694]](https://github.com/kubevirt/kubevirt/pull/15694)[Barakmor1] 允许在 libvirt 升级后宿主机型号发生变化时进行迁移。
- [[PR #15405]](https://github.com/kubevirt/kubevirt/pull/15405)[dasionov] 拒绝针对已暂停 VMI 的停止请求。已暂停的 VMI 必须先取消暂停，然后才能停止。

### SIG-storage {#sig-storage_2}
- [[PR #16429]](https://github.com/kubevirt/kubevirt/pull/16429)[Acedus] 修复：当 DataVolumeTemplates 的 sourceRef 所引用的 DataSource 又指向另一个 DataSource 时，现在可以正确解析后备源。
- [[PR #16333]](https://github.com/kubevirt/kubevirt/pull/16333)[Acedus] 修复：当 VM 没有 CBT 匹配器时，确保 VMI 的 CBT 状态保持禁用。
- [[PR #15913]](https://github.com/kubevirt/kubevirt/pull/15913)[germag] `EnableVirtioFsConfigVolumes` 功能已升级为 GA，不再需要启用相关联的功能门控。
- [[PR #15442]](https://github.com/kubevirt/kubevirt/pull/15442)[Dsanatar] 允许使用来自已完成 Pod 的 PVC 进行 VMExport

### SIG-network {#sig-network_2}
- [[PR #16734]](https://github.com/kubevirt/kubevirt/pull/16734)[orelmisan] 管理员可以禁用 NAD 查询逻辑，并改用 network-resources-injector，从而减少 API 调用
- [[PR #15898]](https://github.com/kubevirt/kubevirt/pull/15898)[bgartzi] 网络 Downward API 的 network-info 包含 MAC 地址
- [[PR #16453]](https://github.com/kubevirt/kubevirt/pull/16453)[nirdothan] 已移除 Macvtap 核心绑定。
- [[PR #16456]](https://github.com/kubevirt/kubevirt/pull/16456)[orelmisan] 已停用的核心 SLIRP 绑定已被完全移除。
- [[PR #16391]](https://github.com/kubevirt/kubevirt/pull/16391)[frenzyfriday] 将 VMI 状态中所报告的仅客户机接口数量限制为 10。这不影响 spec 中指定的接口。
- [[PR #16242]](https://github.com/kubevirt/kubevirt/pull/16242)[orelmisan] 使用 masquerade 绑定时，从状态报告中省略 LLA。
- [[PR #17145]](https://github.com/kubevirt/kubevirt/pull/17145)[kubevirt-bot] 修复了 virt-controller 与 virt-handler 之间的 VMI 状态无限更新循环；当 VMI spec 将主网络接口列在辅助接口之后时，会出现该循环。

### SIG-scale {#sig-scale_1}
- [[PR #16511]](https://github.com/kubevirt/kubevirt/pull/16511)[Ronilerr] 重构 doc-generator
- [[PR #15968]](https://github.com/kubevirt/kubevirt/pull/15968)[sradco] 记录规则 kubevirt_vmi_vcpu_count 的名称更改为 vmi:kubevirt_vmi_vcpu:count

### SIG-observability {#sig-observability_2}
- [[PR #16986]](https://github.com/kubevirt/kubevirt/pull/16986)[kubevirt-bot] 使用所定义的 Deployment 副本数作为触发低数量告警的基准
- [[PR #16987]](https://github.com/kubevirt/kubevirt/pull/16987)[kubevirt-bot] 从 kubevirt_allocatable_nodes 中减去不可调度的节点
- [[PR #16466]](https://github.com/kubevirt/kubevirt/pull/16466)[Ronilerr] 修复 LowReadyVirtOperatorsCount，改为使用 running 而不是 up，并将 kubevirt_virt_operator_ready 改为使用 sum 和 *，而不再使用 count 和 +
- [[PR #16645]](https://github.com/kubevirt/kubevirt/pull/16645)[Ronilerr] 修复语法错误
- [[PR #15278]](https://github.com/kubevirt/kubevirt/pull/15278)[sradco] 将已分配的 CPU 和内存请求报告为带有 source="guest_effective" 标签的简化指标，展示应用实例类型、偏好以及层级之后的最终值。
- [[PR #16342]](https://github.com/kubevirt/kubevirt/pull/16342)[sradco] 新增 VirtLauncherPodsStuckFailed 告警
- [[PR #15237]](https://github.com/kubevirt/kubevirt/pull/15237)[sradco] KubeVirtVMGuestMemoryPressure
- [[PR #16351]](https://github.com/kubevirt/kubevirt/pull/16351)[sradco] 修复 GuestFilesystemAlmostOutOfSpace 中的缺陷，该告警会针对不相关的文件系统类型触发。
- [[PR #15714]](https://github.com/kubevirt/kubevirt/pull/15714)[machadovilaca] 新增 GuestFilesystemAlmostOutOfSpace 告警

### 其他 {#other_2}
- [[PR #16643]](https://github.com/kubevirt/kubevirt/pull/16643)[kwonkwonn] 缺陷修复：正确检测 CDI 和 Prometheus CRD，避免将其误认为其他对象。
- [[PR #16558]](https://github.com/kubevirt/kubevirt/pull/16558)[fossedihelm] MigrationPriorityQueue 功能门控已从 Alpha 提升至 Beta。
- [[PR #14661]](https://github.com/kubevirt/kubevirt/pull/14661)[oujonny] 为热插拔 Pod 添加针对不可调度污点的容忍
- [[PR #16336]](https://github.com/kubevirt/kubevirt/pull/16336)[akalenyu] 维护：修复发布分支可能因节点上已存在相同远程镜像而失败的问题
- [[PR #16354]](https://github.com/kubevirt/kubevirt/pull/16354)[akalenyu] 维护：Windows 通道：针对错误的 NFS 镜像 SEEK_DATA 实现采取变通办法
- [[PR #15863]](https://github.com/kubevirt/kubevirt/pull/15863)[HarshithaMS005] 测试修复：使 Alpine ISO 挂载检查与架构无关
- [[PR #15374]](https://github.com/kubevirt/kubevirt/pull/15374)[xpivarc] NodeRestriction：现在会验证节点更新的来源
- [[PR #15934]](https://github.com/kubevirt/kubevirt/pull/15934)[jschintag] 将 IBM Secure Execution 功能提升至 Beta 阶段。
- [[PR #15970]](https://github.com/kubevirt/kubevirt/pull/15970)[jean-edouard] KubevirtSeccompProfile 功能现已进入 Beta
- [[PR #15960]](https://github.com/kubevirt/kubevirt/pull/15960)[Barakmor1] 将 ImageVolume 功能门控提升至 Beta


## v1.7.0 {#v170}

发布日期：2025 年 11 月 27 日，星期三

KubeVirt v1.7 面向 Kubernetes v1.34 构建，并额外支持此前两个版本。更多信息请参阅 [KubeVirt 支持矩阵](https://github.com/kubevirt/sig-release/blob/main/releases/k8s-support-matrix.md)。

要查看为本版本做出贡献的优秀人员名单，请参阅 [KubeVirt v1.7.0 发行标签](https://github.com/kubevirt/kubevirt/releases/tag/v1.7.0)。

### API 变更 {#api-change_3}
- [[PR #16129]](https://github.com/kubevirt/kubevirt/pull/16129)[kubevirt-bot] 从运行中的 VM 分离 CD-ROM 时添加 RestartRequired
- [[PR #16097]](https://github.com/kubevirt/kubevirt/pull/16097)[kubevirt-bot] 引入新的子资源 `/evacuate/cancel` 以及 `virtctl evacuate-cancel` 命令，允许用户取消虚拟机实例（VMI）的疏散过程。该操作会清除 VMI 状态中的 `evacuationNodeName` 字段，停止自动创建迁移资源，并完全中止驱逐周期。
- [[PR #16033]](https://github.com/kubevirt/kubevirt/pull/16033)[noamasu] VMSnapshot：新增 SourceIndications 状态字段，用于列出带说明的快照指示，使含义更加清晰。
- [[PR #16051]](https://github.com/kubevirt/kubevirt/pull/16051)[kubevirt-bot] 引入 pool.kubevirt.io/v1beta1
- [[PR #16027]](https://github.com/kubevirt/kubevirt/pull/16027)[kubevirt-bot] VMPool：增加对自愈策略的支持
- [[PR #15999]](https://github.com/kubevirt/kubevirt/pull/15999)[kubevirt-bot] VMpool：增加缩容策略支持，包含 Proactive、opportunistic 模式以及 statePreservation
- [[PR #15887]](https://github.com/kubevirt/kubevirt/pull/15887)[fossedihelm] Alpha：KubeVirt 中的通用迁移优先级
- [[PR #14575]](https://github.com/kubevirt/kubevirt/pull/14575)[zhencliu] Intel TDX 的实验性支持
- [[PR #15123]](https://github.com/kubevirt/kubevirt/pull/15123)[Sreeja1725] VMpool：增加 UpdateStrategy 支持，包含 Proactive、Opportunistic 模式以及选择策略
- [[PR #14845]](https://github.com/kubevirt/kubevirt/pull/14845)[alancaldelas] 在 `WorkloadEncruptionSEV` 功能门控保护下，提供对 AMD SEV-SNP 的实验性支持。
- [[PR #15783]](https://github.com/kubevirt/kubevirt/pull/15783)[orelmisan] 通过 virtctl expose 创建 Service 时指定正确的标签选择。virtctl v1.7 及以上版本的 expose 命令将无法与更早的 KubeVirt 版本配合使用。
- [[PR #15698]](https://github.com/kubevirt/kubevirt/pull/15698)[Acedus] 现在可以为 VM 磁盘配置 discard_granularity。
- [[PR #15552]](https://github.com/kubevirt/kubevirt/pull/15552)[nirdothan] NONR
- [[PR #15539]](https://github.com/kubevirt/kubevirt/pull/15539)[tiraboschi] 新增 VirtualMachineInstanceEvictionRequested 状况，用于跟踪驱逐
- [[PR #15798]](https://github.com/kubevirt/kubevirt/pull/15798)[lyarwood] 对 `ioThreads` 这一 VMI 可配置项的支持已加入 `instancetype.kubevirt.io/v1beta1` API，现在可以由实例类型提供 `supplementalPoolThreadCount`。
- [[PR #15398]](https://github.com/kubevirt/kubevirt/pull/15398)[lyarwood] 偏好现在可以表达供 VirtualMachine 使用的首选架构值与必需架构值
- [[PR #15400]](https://github.com/kubevirt/kubevirt/pull/15400)[lyarwood] 已移除弃用的 `instancetype.kubevirt.io/v1alpha{1,2}` API 和 CRD
- [[PR #15681]](https://github.com/kubevirt/kubevirt/pull/15681)[jean-edouard] 现在会在迁移时重新计算内存超分。
- [[PR #15406]](https://github.com/kubevirt/kubevirt/pull/15406)[Sreeja1725] 添加 VMpool 终结器，以确保正确清理
- [[PR #14772]](https://github.com/kubevirt/kubevirt/pull/14772)[ShellyKa13] ChangedBlockTracking：当 VM 匹配标签选择器时，启用 qcow2 overlay 的添加与移除
- [[PR #15238]](https://github.com/kubevirt/kubevirt/pull/15238)[victortoso] 在不使用 VNC 的情况下进行截图
- [[PR #15344]](https://github.com/kubevirt/kubevirt/pull/15344)[SkalaNetworks] 新增 VolumeOwnershipPolicy，用于决定卷在恢复之后如何归属。
- [[PR #14976]](https://github.com/kubevirt/kubevirt/pull/14976)[dasionov] 移除对 ppc64le 架构配置的支持
- [[PR #14983]](https://github.com/kubevirt/kubevirt/pull/14983)[sradco] 此 PR 添加以下告警：GuestPeakVCPUQueueHighWarning、GuestPeakVCPUQueueHighCritical
- [[PR #15096]](https://github.com/kubevirt/kubevirt/pull/15096)[lyarwood] `foregroundDeleteVirtualMachine` 已弃用，并由带域限定的 `kubevirt.io/foregroundDeleteVirtualMachine` 取代。
- [[PR #14879]](https://github.com/kubevirt/kubevirt/pull/14879)[machadovilaca] 新增 GuestAgentInfo 信息指标
- [[PR #15017]](https://github.com/kubevirt/kubevirt/pull/15017)[nekkunti] 在 KubeVirt 集群配置中增加了对 `s390x`（IBM Z）架构特定配置的支持。
- [[PR #15022]](https://github.com/kubevirt/kubevirt/pull/15022)[awels] 同步控制器的迁移网络 IP 地址由 KubeVirt CR 通告
- [[PR #15021]](https://github.com/kubevirt/kubevirt/pull/15021)[awels] 去中心化迁移资源现在会显示同步地址
- [[PR #14365]](https://github.com/kubevirt/kubevirt/pull/14365)[alaypatel07] 增加对 GPU 和主机设备等 DRA 设备的支持。
- [[PR #14882]](https://github.com/kubevirt/kubevirt/pull/14882)[awels] 去中心化实时迁移现已可用，允许跨命名空间和跨集群进行迁移
- [[PR #14875]](https://github.com/kubevirt/kubevirt/pull/14875)[nirdothan] 支持使用 passt 进行无缝 TCP 迁移（alpha）

### 缺陷修复 {#bug-fix_3}
- [[PR #16198]](https://github.com/kubevirt/kubevirt/pull/16198)[kubevirt-bot] 缺陷修复：KubeVirt.spec.imagetag 安装再次可以正常工作
- [[PR #16011]](https://github.com/kubevirt/kubevirt/pull/16011)[kubevirt-bot] 缺陷修复：去中心化实时迁移没有准确地使用迁移限制
- [[PR #15001]](https://github.com/kubevirt/kubevirt/pull/15001)[noamasu] 缺陷修复：为已暂停的 VM 启用 vmsnapshot
- [[PR #15867]](https://github.com/kubevirt/kubevirt/pull/15867)[xpivarc] 缺陷修复：数千次迁移不应导致正在进行的迁移失败
- [[PR #15788]](https://github.com/kubevirt/kubevirt/pull/15788)[mhenriks] 修复热插拔卷的 RestartRequired 处理
- [[PR #15676]](https://github.com/kubevirt/kubevirt/pull/15676)[xpivarc] 缺陷修复：virt-launcher 会被正确回收
- [[PR #15692]](https://github.com/kubevirt/kubevirt/pull/15692)[awels] 缺陷修复：如果恢复策略设置为 VM，从 VMSnapshot 恢复的裸 PVC 现在会正确地归该 VM 所有
- [[PR #15759]](https://github.com/kubevirt/kubevirt/pull/15759)[lyarwood] 现在只会发送一条 `Signaled Graceful Shutdown` 事件，以避免在长时间的优雅关闭尝试期间向事件记录器大量发送事件
- [[PR #13111]](https://github.com/kubevirt/kubevirt/pull/13111)[brianmcarey] 构建：更新至 bazel v6.5.0 和 rules_oci
- [[PR #15605]](https://github.com/kubevirt/kubevirt/pull/15605)[awels] 缺陷修复：能够正确地取消进行中的去中心化实时迁移
- [[PR #15642]](https://github.com/kubevirt/kubevirt/pull/15642)[akalenyu] 缺陷修复：先前经过存储迁移且带有 vTPM 的 Windows VM 无法进行实时迁移
- [[PR #15603]](https://github.com/kubevirt/kubevirt/pull/15603)[akalenyu] 缺陷修复：修复名称过长的 VM 的卷迁移
- [[PR #15509]](https://github.com/kubevirt/kubevirt/pull/15509)[alromeros] 缺陷修复：从导出服务器中排除 lost+found
- [[PR #15557]](https://github.com/kubevirt/kubevirt/pull/15557)[fossedihelm] 修复：handler REST 请求中的 gRPC 客户端会被正确关闭
- [[PR #13500]](https://github.com/kubevirt/kubevirt/pull/13500)[brandboat] 将错误的指标名称 kubevirt_vmi_migration_disk_transfer_rate_bytes 修正为 kubevirt_vmi_migration_memory_transfer_rate_bytes
- [[PR #15470]](https://github.com/kubevirt/kubevirt/pull/15470)[awels] 缺陷修复：去中心化实时迁移失败后无法删除源 VM
- [[PR #15423]](https://github.com/kubevirt/kubevirt/pull/15423)[tiraboschi] 根据 VMI 驱逐状态派生 eviction-in-progress 注解
- [[PR #15170]](https://github.com/kubevirt/kubevirt/pull/15170)[dasionov] 缺陷修复：确保宽限期元数据缓存在 virt-launcher 中保持同步
- [[PR #15397]](https://github.com/kubevirt/kubevirt/pull/15397)[ShellyKa13] 缺陷修复：防止 VMSnapshotContent 因相同的错误消息而反复更新
- [[PR #15167]](https://github.com/kubevirt/kubevirt/pull/15167)[Sreeja1725] 添加用于禁用 Node Labeller 服务的命令行标志
- [[PR #15365]](https://github.com/kubevirt/kubevirt/pull/15365)[tiraboschi] 统一 descheduler 选择退出注解的名称
- [[PR #15093]](https://github.com/kubevirt/kubevirt/pull/15093)[Acedus] 缺陷修复：当关联的 VM 可以实时迁移时，卷热插拔 Pod 不再被驱逐。
- [[PR #15305]](https://github.com/kubevirt/kubevirt/pull/15305)[Acedus] 缺陷修复：存储卷迁移之后，VM 的快照与恢复现在可以正确工作
- [[PR #15314]](https://github.com/kubevirt/kubevirt/pull/15314)[xpivarc] 现在会对聚合 API 强制校验 Common Name
- [[PR #15182]](https://github.com/kubevirt/kubevirt/pull/15182)[akalenyu] 缺陷修复：当 VMExport 的 Secret 中包含点号时，导出失败
- [[PR #15099]](https://github.com/kubevirt/kubevirt/pull/15099)[akalenyu] 缺陷修复：当 VMExport 的名称中包含点号时，导出失败
- [[PR #15067]](https://github.com/kubevirt/kubevirt/pull/15067)[alromeros] 缺陷修复：为上传用 PVC 添加标签，以支持 CDI WebhookPvcRendering
- [[PR #15706]](https://github.com/kubevirt/kubevirt/pull/15706)[ksimon1] 修复：在 MergeCABundle 中，过期证书的移除优先于 50 个证书的数量上限

### SIG-compute {#sig-compute_3}
- [[PR #16168]](https://github.com/kubevirt/kubevirt/pull/16168)[kubevirt-bot] 迁移使用专用证书进行 mTLS。
- [[PR #16150]](https://github.com/kubevirt/kubevirt/pull/16150)[kubevirt-bot] 修复：出现节点压力时，KSM 会在 3 分钟内启用
- [[PR #16108]](https://github.com/kubevirt/kubevirt/pull/16108)[kubevirt-bot] 允许在 libvirt 升级后宿主机型号发生变化时进行迁移。
- [[PR #16089]](https://github.com/kubevirt/kubevirt/pull/16089)[kubevirt-bot] 去中心化实时迁移的失败会在源端与目标端之间正确传播
- [[PR #15936]](https://github.com/kubevirt/kubevirt/pull/15936)[kubevirt-bot] 将 common-instancetypes 捆绑包更新至 v1.5.1
- [[PR #15008]](https://github.com/kubevirt/kubevirt/pull/15008)[fossedihelm] 修复 kv 升级期间迁移可能导致的空指针
- [[PR #15712]](https://github.com/kubevirt/kubevirt/pull/15712)[lyarwood] 提高 `DefaultVirtHandler{QPS,Burst}` 的取值，以确保 `virt-handler` 内部不会形成瓶颈
- [[PR #14902]](https://github.com/kubevirt/kubevirt/pull/14902)[tiraboschi] 从 VM.spec.template.metadata 同步到 VMI、再同步到 virt-launcher Pod 的注解和标签列表现在可以扩展
- [[PR #15503]](https://github.com/kubevirt/kubevirt/pull/15503)[Sreeja1725] 增强 VMPool 单元测试，使其使用 fake client
- [[PR #15422]](https://github.com/kubevirt/kubevirt/pull/15422)[lyarwood] 将 `DefaultVirtWebhookClient{QPS,Burst}` 的取值与 `DefaultVirtWebhookClient{QPS,Burst}` 对齐，以避免在大规模驱逐事件期间，Webhook 客户端因无法处理的请求而达到饱和
- [[PR #15513]](https://github.com/kubevirt/kubevirt/pull/15513)[jean-edouard] 修复了迁移控制器中的优先级提升缺陷
- [[PR #15478]](https://github.com/kubevirt/kubevirt/pull/15478)[0xFelix] virtctl：已从 virtctl 中移除 --local-ssh 标志以及内置的 ssh 和 scp 客户端。从现在起，宿主机上的本地 ssh 和 scp 客户端始终由 virtctl ssh 和 virtctl scp 封装。
- [[PR #15267]](https://github.com/kubevirt/kubevirt/pull/15267)[victortoso] 为 VNC 端点添加 `preserve session` 选项
- [[PR #15475]](https://github.com/kubevirt/kubevirt/pull/15475)[0xFelix] virtctl（portfoward|ssh|scp）：放弃对旧式点号语法的支持。如果使用了旧的点号语法，virtctl 可能会再次要求验证主机密钥。在某些情况下，可能需要手动更新 known_hosts 文件。
- [[PR #15253]](https://github.com/kubevirt/kubevirt/pull/15253)[0xFelix] 将捆绑的 common-instancetypes 升级至 v1.4.0，此版本新增了偏好。
- [[PR #15016]](https://github.com/kubevirt/kubevirt/pull/15016)[fossedihelm] 修复升级期间 postcopy 与 multifd 的兼容性
- [[PR #14685]](https://github.com/kubevirt/kubevirt/pull/14685)[seanbanko] 允许使用实例类型的虚拟机实例指定与该实例类型不冲突的内存字段
- [[PR #14888]](https://github.com/kubevirt/kubevirt/pull/14888)[akalenyu] 清理：libvmi：添加命名一致的 cpu/mem setter
- [[PR #15037]](https://github.com/kubevirt/kubevirt/pull/15037)[jean-edouard] HostDisk：KubeVirt 不再执行 chown/chmod 来弥补不支持 fsGroup 的存储

### SIG-storage {#sig-storage_3}
- [[PR #14973]](https://github.com/kubevirt/kubevirt/pull/14973)[Barakmor1] 支持对容器磁盘镜像经过修改的 ImageVolume 进行实时迁移
- [[PR #15651]](https://github.com/kubevirt/kubevirt/pull/15651)[dcarrier] 为 libdv 添加 WithUploadSource builder

### SIG-network {#sig-network_3}
- [[PR #15669]](https://github.com/kubevirt/kubevirt/pull/15669)[HarshithaMS005] 规范化接口状态，以确保热插拔和热拔出测试的稳定性
- [[PR #15661]](https://github.com/kubevirt/kubevirt/pull/15661)[nirdothan] 支持 Istio 1.25 及以上版本。

### SIG-scale {#sig-scale_2}
- [[PR #15061]](https://github.com/kubevirt/kubevirt/pull/15061)[lyarwood] 对所有 `*_SHASUM` 环境变量的支持已从 `virt-operator` 组件中移除。用户应改用其余的 `*_IMAGE` 环境变量，通过标签、摘要或两者来请求特定的镜像版本。

### SIG-observability {#sig-observability_3}
- [[PR #15504]](https://github.com/kubevirt/kubevirt/pull/15504)[sradco] 将指标 kubevirt_vm_container_free_memory_bytes_based_on_rss 和 kubevirt_vm_container_free_memory_bytes_based_on_working_set_bytes 的名称更新为 kubevirt_vm_container_memory_request_margin_based_on_rss_bytes 和 kubevirt_vm_container_memory_request_margin_based_on_working_set_bytes，以使含义更加清晰
- [[PR #15181]](https://github.com/kubevirt/kubevirt/pull/15181)[avlitman] 新增 kubevirt_vm_labels 指标，用于展示已转换为 Prometheus 标签的 VM 标签，并可以通过带有忽略列表和允许列表的 ConfigMap 进行配置。
- [[PR #15227]](https://github.com/kubevirt/kubevirt/pull/15227)[sradco] 新增 VM 告警：VirtualMachineStuckInUnhealthyState、VirtualMachineStuckOnNode
- [[PR #15464]](https://github.com/kubevirt/kubevirt/pull/15464)[avlitman] 将 virt-launcher 加入 kubevirt_memory_delta_from_requested_bytes 指标以及 cnv_abnormal 指标。

### 其他 {#other_3}
- [[PR #16076]](https://github.com/kubevirt/kubevirt/pull/16076)[kubevirt-bot] NodeRestriction：现在会验证节点更新的来源
- [[PR #16041]](https://github.com/kubevirt/kubevirt/pull/16041)[kubevirt-bot] KubevirtSeccompProfile 功能现已进入 Beta
- [[PR #16039]](https://github.com/kubevirt/kubevirt/pull/16039)[kubevirt-bot] 将 IBM Secure Execution 功能提升至 Beta 阶段。
- [[PR #16005]](https://github.com/kubevirt/kubevirt/pull/16005)[kubevirt-bot] 将 ImageVolume 功能门控提升至 Beta
- [[PR #15939]](https://github.com/kubevirt/kubevirt/pull/15939)[dasionov] Beta：VideoConfig
- [[PR #15878]](https://github.com/kubevirt/kubevirt/pull/15878)[Sreeja1725] 添加 v1.6.0 性能与规模基准数据
- [[PR #15830]](https://github.com/kubevirt/kubevirt/pull/15830)[varunrsekar] Beta：PanicDevices
- [[PR #15615]](https://github.com/kubevirt/kubevirt/pull/15615)[alromeros] 对象图：包含 NAD 和 ServiceAccount
- [[PR #15690]](https://github.com/kubevirt/kubevirt/pull/15690)[lyarwood] `virt-api` 的副本数现在会根据环境中带有 `kubevirt.io/schedulable=true` 标签的节点数量进行扩缩。
- [[PR #15630]](https://github.com/kubevirt/kubevirt/pull/15630)[awels] 允许在 L3 网络上进行去中心化实时迁移
- [[PR #15357]](https://github.com/kubevirt/kubevirt/pull/15357)[dasionov] 确保新创建的 VM 具有默认的 Firmware.Serial 值
- [[PR #15157]](https://github.com/kubevirt/kubevirt/pull/15157)[jean-edouard] virt-operator 不会调度到工作节点上
- [[PR #15118]](https://github.com/kubevirt/kubevirt/pull/15118)[dankenigsberg] 移除对 VM domain.firmaware.serial 所施加的任意限制。任何字符串都会原样传递给 smbios。非法值可能会根据 qemu/smbios 版本被调整或忽略。
- [[PR #14964]](https://github.com/kubevirt/kubevirt/pull/14964)[xpivarc] Beta：NodeRestriction
- [[PR #14986]](https://github.com/kubevirt/kubevirt/pull/14986)[awels] 可以信任额外的 CA，用于验证 KubeVirt 基础设施组件
- [[PR #15531]](https://github.com/kubevirt/kubevirt/pull/15531)[Yu-Jack] 将 prometheus operator 升级至 0.80.1
- [[PR #16026]](https://github.com/kubevirt/kubevirt/pull/16026)[fossedihelm] 支持 v0.32.5 代码生成器
- [[PR #15098]](https://github.com/kubevirt/kubevirt/pull/15098)[dominikholler] 将依赖 golang.org/x/oauth2 更新至 v0.27.0
- [[PR #15100]](https://github.com/kubevirt/kubevirt/pull/15100)[dominikholler] 将依赖 golang.org/x/net 更新至 v0.38.0
- [[PR #15784]](https://github.com/kubevirt/kubevirt/pull/15784)[brianmcarey] 使用 go v1.24.7 构建 KubeVirt
- [[PR #15718]](https://github.com/kubevirt/kubevirt/pull/15718)[Vicente-Cheng] 将 k8s 升级至 v1.33

## v1.6.0 {#v160}

发布日期：2025 年 7 月 31 日，星期四

KubeVirt v1.6 面向 Kubernetes v1.33 构建，并额外支持此前两个版本。更多信息请参阅 [KubeVirt 支持矩阵](https://github.com/kubevirt/sig-release/blob/main/releases/k8s-support-matrix.md)。

要查看为本版本做出贡献的出色贡献者名单，请参阅 [KubeVirt v1.6.0 发行标签](https://github.com/kubevirt/kubevirt/releases/tag/v1.6.0)。

### API 变更 {#api-change_4}
- [[PR #15080]](https://github.com/kubevirt/kubevirt/pull/15080)[kubevirt-bot] 同步控制器的迁移网络 IP 地址由 KubeVirt CR 通告
- [[PR #15039]](https://github.com/kubevirt/kubevirt/pull/15039)[alaypatel07] 增加对 GPU 和主机设备等 DRA 设备的支持。
- [[PR #15014]](https://github.com/kubevirt/kubevirt/pull/15014)[kubevirt-bot] 支持使用 passt 进行无缝 TCP 迁移（alpha）
- [[PR #14935]](https://github.com/kubevirt/kubevirt/pull/14935)[alromeros] 添加 virtctl objectgraph 命令
- [[PR #13103]](https://github.com/kubevirt/kubevirt/pull/13103)[varunrsekar] 功能：支持在 VirtualMachineInstance 中定义 panic 设备，以捕获来自客户机的崩溃信号。
- [[PR #13764]](https://github.com/kubevirt/kubevirt/pull/13764)[xpivarc] KubeVirt 不再使用 PDB
- [[PR #14801]](https://github.com/kubevirt/kubevirt/pull/14801)[arsiesys] VirtualMachinePool 现在支持 `.ScaleInStrategy.Proactive.SelectionPolicy.BasePolicy` 字段，用于控制缩容行为。新的 `"DescendingOrder"` 策略会按序号索引降序删除 VM，从而提供可预测的缩容行为。若未指定，则默认为 `"random"`。
- [[PR #14259]](https://github.com/kubevirt/kubevirt/pull/14259)[orelmisan] 将网卡热插拔与 LiveUpdate 推出策略集成
- [[PR #14673]](https://github.com/kubevirt/kubevirt/pull/14673)[dasionov] 为 VM 添加视频配置字段，以便显式选择视频设备
- [[PR #14681]](https://github.com/kubevirt/kubevirt/pull/14681)[victortoso] 通过 ACPI MSDM 表进行 Windows 离线激活
- [[PR #14723]](https://github.com/kubevirt/kubevirt/pull/14723)[SkalaNetworks] 为 VMRestore 添加 VolumeRestorePolicies 和 VolumeRestoreOverrides
- [[PR #14040]](https://github.com/kubevirt/kubevirt/pull/14040)[jschintag] 增加对 IBM Z 上 Secure Execution VM 的支持
- [[PR #13847]](https://github.com/kubevirt/kubevirt/pull/13847)[mhenriks] 借助 CD-ROM 注入与弹出实现声明式卷热插拔
- [[PR #14807]](https://github.com/kubevirt/kubevirt/pull/14807)[alromeros] 添加对象图子资源
- [[PR #14183]](https://github.com/kubevirt/kubevirt/pull/14183)[aqilbeig] 为 VirtualMachinePool 添加 maxUnavailable 支持
- [[PR #14616]](https://github.com/kubevirt/kubevirt/pull/14616)[awels] VirtualMachineInstanceMigrations 现在可以表明自身是源迁移还是目标迁移
- [[PR #14617]](https://github.com/kubevirt/kubevirt/pull/14617)[SkalaNetworks] 增加了对 VirtualMachineClone 中自定义 JSON 补丁的支持。
- [[PR #14602]](https://github.com/kubevirt/kubevirt/pull/14602)[orelmisan] 对于可实时更新的网络字段，不会在 VM 对象上设置 "RestartRequired" 状况
- [[PR #14267]](https://github.com/kubevirt/kubevirt/pull/14267)[Barakmor1] 使用 ImageVolume 实现容器磁盘功能，并由 ImageVolume 功能门控加以保护。
- [[PR #14539]](https://github.com/kubevirt/kubevirt/pull/14539)[nirdothan] 为 passt 网络绑定插件启用 vhost-user 模式
- [[PR #14449]](https://github.com/kubevirt/kubevirt/pull/14449)[0xFelix] 现在可以通过为 VirtualMachineInstance 添加 kubevirt.io/disablePCIHole 注解来禁用 64 位 PCI 孔。这使得 Windows XP 或 Server 2003 等旧版操作系统能够使用 Q35 机器类型在 KubeVirt 上启动。
- [[PR #14437]](https://github.com/kubevirt/kubevirt/pull/14437)[jschintag] 在校验 VMI 时，对有效机器类型执行更严格的检查
- [[PR #13911]](https://github.com/kubevirt/kubevirt/pull/13911)[avlitman] 已移除 VirtHandlerRESTErrorsHigh、VirtOperatorRESTErrorsHigh、VirtAPIRESTErrorsHigh 和 VirtControllerRESTErrorsHigh 告警。
- [[PR #14277]](https://github.com/kubevirt/kubevirt/pull/14277)[HarshithaMS005] 使用 Diag288 设备模型，在 s390x 上启用 Watchdog 设备支持。
- [[PR #14405]](https://github.com/kubevirt/kubevirt/pull/14405)[jpeimer] 在 ioThreadsPolicy 可能取值的描述中加入了 supplementalPool
- [[PR #14219]](https://github.com/kubevirt/kubevirt/pull/14219)[lyarwood] 创建引用了不存在的实例类型或偏好的 VirtualMachine 的请求不再被拒绝。VirtualMachine 会被创建，但在集群中创建所缺失的资源之前将无法启动。
- [[PR #14048]](https://github.com/kubevirt/kubevirt/pull/14048)[lyarwood] `v1alpha{1,2}` 版本的 `instancetype.kubevirt.io` API 组不再被提供，也不再受支持。
- [[PR #14316]](https://github.com/kubevirt/kubevirt/pull/14316)[lyarwood] 新增的 `Enabled` 属性已加入 `TPM` 设备，允许用户显式禁用该设备，无论引用了何种偏好。
- [[PR #14050]](https://github.com/kubevirt/kubevirt/pull/14050)[lyarwood] `InstancetypeReferencePolicy` 功能已升级为 GA，不再需要启用相关联的功能门控。
- [[PR #14065]](https://github.com/kubevirt/kubevirt/pull/14065)[jean-edouard] VM 持久状态进入 GA
- [[PR #14096]](https://github.com/kubevirt/kubevirt/pull/14096)[ShellyKa13] VMSnapshot：如果冻结失败，则向快照添加 QuiesceFailed 指示
- [[PR #14068]](https://github.com/kubevirt/kubevirt/pull/14068)[jean-edouard] 默认的 VM 推出策略现为 LiveUpdate。重要提示：要保留先前的行为，需要在 KubeVirt CR 中将 rolloutStrategy 设置为 "Stage"。
- [[PR #12725]](https://github.com/kubevirt/kubevirt/pull/12725)[tiraboschi] 支持实时迁移到指定名称的节点
- [[PR #13807]](https://github.com/kubevirt/kubevirt/pull/13807)[Barakmor1] virt-launcher 现在使用 bash 检索磁盘信息并校验容器磁盘文件，因此启动器镜像中需要包含 bash
- [[PR #13744]](https://github.com/kubevirt/kubevirt/pull/13744)[nirdothan] 可以将网络接口状态设置为 `down` 或 `up`，以便在 VM 运行时相应地设置链路状态。也支持在这些状态下热插拔接口。
- [[PR #13536]](https://github.com/kubevirt/kubevirt/pull/13536)[jean-edouard] 中断的迁移现在会在下次 VM 启动时得到协调。
- [[PR #13916]](https://github.com/kubevirt/kubevirt/pull/13916)[lyarwood] 实例类型和偏好的运行时数据现在存储在 `Status.{Instancetype,Preference}Ref` 之下，不再被改写进核心 VirtualMachine 的 `Spec`。
- [[PR #13831]](https://github.com/kubevirt/kubevirt/pull/13831)[ShellyKa13] VMClone：移除检查快照源的 Webhook
- [[PR #13815]](https://github.com/kubevirt/kubevirt/pull/13815)[acardace] 将 ClusterProfiler 功能门控设为 GA，并添加一项用于启用它的配置

### 缺陷修复 {#bug-fix_4}
- [[PR #14130]](https://github.com/kubevirt/kubevirt/pull/14130)[dasionov] 缺陷修复：为现有 VM 持久化固件 UUID
- [[PR #14338]](https://github.com/kubevirt/kubevirt/pull/14338)[dasionov] 缺陷修复：限制 MaxSockets，使最大 vCPU 数不超过 512。
- [[PR #13690]](https://github.com/kubevirt/kubevirt/pull/13690)[dasionov] 缺陷修复：将机器类型添加到 `NodeSelector`，以防止在不受支持的节点上发生破坏性变更
- [[PR #15114]](https://github.com/kubevirt/kubevirt/pull/15114)[kubevirt-bot] 缺陷修复：为上传用 PVC 添加标签，以支持 CDI WebhookPvcRendering
- [[PR #15202]](https://github.com/kubevirt/kubevirt/pull/15202)[kubevirt-bot] 缺陷修复：当 VMExport 的 Secret 中包含点号时，导出失败
- [[PR #15171]](https://github.com/kubevirt/kubevirt/pull/15171)[kubevirt-bot] 缺陷修复：当 VMExport 的名称中包含点号时，导出失败
- [[PR #13898]](https://github.com/kubevirt/kubevirt/pull/13898)[brandboat] 将 kubevirt_vmi_vcpu_seconds_total 指标中的时间单位转换从微秒改为纳秒。
- [[PR #14961]](https://github.com/kubevirt/kubevirt/pull/14961)[akalenyu] 缺陷修复：从 HPP 存储迁移到 OCS 之后，无法实时迁移 Windows VM
- [[PR #14632]](https://github.com/kubevirt/kubevirt/pull/14632)[iholder101] virt-handler：减少 KSM 处理过程中的 Get() 调用
- [[PR #14658]](https://github.com/kubevirt/kubevirt/pull/14658)[alromeros] 缺陷修复：更新后端存储逻辑，使其能够用于命名约定不标准的 PVC
- [[PR #14695]](https://github.com/kubevirt/kubevirt/pull/14695)[alromeros] 缺陷修复：在 PVC 大小发生变化时重新入队 VMI，从而修复在线扩容
- [[PR #14640]](https://github.com/kubevirt/kubevirt/pull/14640)[xpivarc] ARM：CPU 绑核现在不会再引发 panic
- [[PR #13951]](https://github.com/kubevirt/kubevirt/pull/13951)[alromeros] 缺陷修复：截断导出 Pod 中的卷名称
- [[PR #14145]](https://github.com/kubevirt/kubevirt/pull/14145)[ayushpatil2122] 处理 cellToCell 中的空指针解引用
- [[PR #14281]](https://github.com/kubevirt/kubevirt/pull/14281)[ShellyKa13] VMRestore：使 VM 的 RunStrategy 保持与恢复前一致
- [[PR #14309]](https://github.com/kubevirt/kubevirt/pull/14309)[alicefr] 通过改进 pr-helper 中的套接字访问和 multipath 文件，修复了 multipathd 的持久预留支持
- [[PR #14325]](https://github.com/kubevirt/kubevirt/pull/14325)[vamsikrishna-siddu] 修复：disks-images-provider 为 s390x 指向了错误的 Alpine 镜像。
- [[PR #14286]](https://github.com/kubevirt/kubevirt/pull/14286)[machadovilaca] 在初始化时注册 k8s client-go 延迟指标
- [[PR #13870]](https://github.com/kubevirt/kubevirt/pull/13870)[dasionov] 当 VMI 被标记为删除时，确保先终结并删除 launcher Pod，然后再移除 VMI 终结器。
- [[PR #14071]](https://github.com/kubevirt/kubevirt/pull/14071)[alicefr] 为 pr-helper 添加入口点，用于创建指向 multipath 套接字的符号链接

### SIG-compute {#sig-compute_4}
- [[PR #15256]](https://github.com/kubevirt/kubevirt/pull/15256)[kubevirt-bot] 将捆绑的 common-instancetypes 升级至 v1.4.0，此版本新增了偏好。
- [[PR #15214]](https://github.com/kubevirt/kubevirt/pull/15214)[dhiller] 隔离不稳定测试 `[sig-compute]VM state with persistent TPM VM option enabled should persist VM state of EFI across migration and restart`
- [[PR #15178]](https://github.com/kubevirt/kubevirt/pull/15178)[kubevirt-bot] 修复升级期间 postcopy 与 multifd 的兼容性
- [[PR #14744]](https://github.com/kubevirt/kubevirt/pull/14744)[tiraboschi] 若干动态注解会从 VM 模板同步到 VMI，再同步到 virt-launcher Pod
- [[PR #14907]](https://github.com/kubevirt/kubevirt/pull/14907)[mhenriks] 允许热插拔磁盘使用 virtio 总线
- [[PR #14754]](https://github.com/kubevirt/kubevirt/pull/14754)[mhenriks] 为热插拔分配更多 PCI 端口
- [[PR #14705]](https://github.com/kubevirt/kubevirt/pull/14705)[jean-edouard] virt-handler 中的迁移控制器已经过重新架构，迁移应当更加稳定
- [[PR #14793]](https://github.com/kubevirt/kubevirt/pull/14793)[jean-edouard] 失败的后复制迁移现在始终以 VMI 失败结束
- [[PR #14827]](https://github.com/kubevirt/kubevirt/pull/14827)[orelmisan] 修复启用模拟时的网络设置
- [[PR #14768]](https://github.com/kubevirt/kubevirt/pull/14768)[oshoval] 在钩子边车上公开 CONTAINER_NAME。
- [[PR #14728]](https://github.com/kubevirt/kubevirt/pull/14728)[orelmisan] 现在允许在启用网络多队列的情况下进行 CPU 热插拔
- [[PR #14619]](https://github.com/kubevirt/kubevirt/pull/14619)[cloud-j-luna] virtctl vnc 命令现在支持用户提供的 VNC 客户端。
- [[PR #14599]](https://github.com/kubevirt/kubevirt/pull/14599)[HarshithaMS005] 已针对 watchdog 设备模型启用校验
- [[PR #13806]](https://github.com/kubevirt/kubevirt/pull/13806)[iholder101] 脏页速率通过新的 `GetDomainDirtyRateStats()` gRPC 方法进行报告，同时也由 Prometheus 指标 `kubevirt_vmi_dirty_rate_bytes_per_second` 报告。
- [[PR #14520]](https://github.com/kubevirt/kubevirt/pull/14520)[dasionov] 为 ARM64 集群启用 node-labeller，并支持机器类型标签。
- [[PR #13297]](https://github.com/kubevirt/kubevirt/pull/13297)[mhenriks] 热插拔卷：从可热插拔磁盘启动
- [[PR #13422]](https://github.com/kubevirt/kubevirt/pull/13422)[mhenriks] 客户机控制台日志：使 virt-tail 成为正式的边车
- [[PR #14374]](https://github.com/kubevirt/kubevirt/pull/14374)[kubevirt-bot] 将 common-instancetypes 捆绑包更新至 v1.3.1
- [[PR #14288]](https://github.com/kubevirt/kubevirt/pull/14288)[qinqon] 不要将实时迁移结束时隐式的 qemu 域暂停公开为 VMI 状态
- [[PR #14328]](https://github.com/kubevirt/kubevirt/pull/14328)[akalenyu] 清理：在合理且不与宿主机共享 cgroup 的开发环境中修复单元测试
- [[PR #14141]](https://github.com/kubevirt/kubevirt/pull/14141)[jean-edouard] 大量迁移不应再导致正在进行的迁移超时
- [[PR #13939]](https://github.com/kubevirt/kubevirt/pull/13939)[0xFelix] virtctl port-forward/ssh/scp 的语法已更改为 type/name[/namespace]。现在可以正确支持名称中包含点号的资源。
- [[PR #13928]](https://github.com/kubevirt/kubevirt/pull/13928)[kubevirt-bot] 将 common-instancetypes 捆绑包更新至 v1.3.0

### SIG-storage {#sig-storage_4}
- [[PR #14737]](https://github.com/kubevirt/kubevirt/pull/14737)[ShellyKa13] virt-Freeze：如果域未处于运行状态，则跳过冻结
- [[PR #14637]](https://github.com/kubevirt/kubevirt/pull/14637)[alromeros] 为后端 PVC 添加标签，以支持 CDI WebhookPvcRendering

### SIG-network {#sig-network_4}
- [[PR #14887]](https://github.com/kubevirt/kubevirt/pull/14887)[oshoval] 发布 passt CNI 镜像，而不是发布 CNI 二进制文件本身。
- [[PR #14738]](https://github.com/kubevirt/kubevirt/pull/14738)[oshoval] 从已停止的 VM 中清理已不存在的接口及其相关网络。
- [[PR #14509]](https://github.com/kubevirt/kubevirt/pull/14509)[phoracek] 网络一致性测试现在使用 `Conformance` 装饰器进行标记。使用 `--ginkgo.label-filter '(sig-network && conformance)` 来选择这些测试。

### SIG-scale {#sig-scale_3}
- [[PR #13888]](https://github.com/kubevirt/kubevirt/pull/13888)[Sreeja1725] 添加 v1.5.0 性能与规模基准数据

### SIG-observability {#sig-observability_4}
- [[PR #14805]](https://github.com/kubevirt/kubevirt/pull/14805)[machadovilaca] 将指标标签中的 none 值替换为空值
- [[PR #14203]](https://github.com/kubevirt/kubevirt/pull/14203)[machadovilaca] 仅对运行中的 VMI 触发 VMCannotBeEvicted
- [[PR #14327]](https://github.com/kubevirt/kubevirt/pull/14327)[machadovilaca] 在 VM 监控中处理小写的 instancetypes/preference 键
- [[PR #14426]](https://github.com/kubevirt/kubevirt/pull/14426)[avlitman] 新增 kubevirt_vmi_migrations_in_unset_phase，不再将其计入 kubevirt_vmi_migration_failed。
- [[PR #14108]](https://github.com/kubevirt/kubevirt/pull/14108)[machadovilaca] 为 kubevirt_vmi_status_addresses 添加接口名称标签
- [[PR #13805]](https://github.com/kubevirt/kubevirt/pull/13805)[machadovilaca] 使用命名空间键获取非集群范围的实例类型和偏好

### 其他 {#other_4}
- [[PR #15264]](https://github.com/kubevirt/kubevirt/pull/15264)[fossedihelm] 已隔离测试 `should live migrate a container disk vm, with an additional PVC mounted, should stay mounted after migration`
- [[PR #15191]](https://github.com/kubevirt/kubevirt/pull/15191)[kubevirt-bot] 移除对 VM domain.firmaware.serial 所施加的任意限制。任何字符串都会原样传递给 smbios。非法值可能会根据 qemu/smbios 版本被调整或忽略。
- [[PR #15201]](https://github.com/kubevirt/kubevirt/pull/15201)[xpivarc] 已知问题：由于存在竞态条件，ParallelOutboundMigrationsPerNode 可能会被忽略
- [[PR #15047]](https://github.com/kubevirt/kubevirt/pull/15047)[kubevirt-bot] Beta：NodeRestriction
- [[PR #15020]](https://github.com/kubevirt/kubevirt/pull/15020)[kubevirt-bot] 可以信任额外的 CA，用于验证 KubeVirt 基础设施组件
- [[PR #14956]](https://github.com/kubevirt/kubevirt/pull/14956)[RobertoMachorro] 已将 CRC 加入 ADOPTERS 文档。
- [[PR #14538]](https://github.com/kubevirt/kubevirt/pull/14538)[iholder101] 将 cgroup v1 转入维护模式
- [[PR #14823]](https://github.com/kubevirt/kubevirt/pull/14823)[xmulligan] 将 Isovalent 加入采用者名单

- [[PR #14440]](https://github.com/kubevirt/kubevirt/pull/14440)[pstaniec-catalogicsoftware] 将 Catalogic 的 CloudCasa 加入 adopters.md 的集成部分
- [[PR #14428]](https://github.com/kubevirt/kubevirt/pull/14428)[jean-edouard] 要使用 nfs-csi，必须将环境变量 KUBEVIRT_NFS_DIR 设置为宿主机上存放 NFS 数据的位置
- [[PR #13940]](https://github.com/kubevirt/kubevirt/pull/13940)[tiraboschi] node-restriction Validating Admission Policy 在失败时将返回一致的原因
- [[PR #14215]](https://github.com/kubevirt/kubevirt/pull/14215)[dominikholler] 将模块 golang.org/x/oauth2 更新至 v0.27.0
- [[PR #14222]](https://github.com/kubevirt/kubevirt/pull/14222)[dominikholler] 将模块 golang.org/x/net 更新至 v0.36.0
- [[PR #14218]](https://github.com/kubevirt/kubevirt/pull/14218)[dominikholler] 将 golang.org/x/crypto 更新至 v0.35.0
- [[PR #14217]](https://github.com/kubevirt/kubevirt/pull/14217)[dominikholler] 将模块 github.com/opencontainers/runc 更新至 v1.1.14
- [[PR #14101]](https://github.com/kubevirt/kubevirt/pull/14101)[qinqon] libvirt：10.10.0-7，qemu：9.1.0-15
- [[PR #15101]](https://github.com/kubevirt/kubevirt/pull/15101)[dominikholler] 将依赖 golang.org/x/oauth2 更新至 v0.27.0
- [[PR #14304]](https://github.com/kubevirt/kubevirt/pull/14304)[jean-edouard] 将模块 github.com/containers/common 更新至 v0.60.4
- [[PR #14664]](https://github.com/kubevirt/kubevirt/pull/14664)[brianmcarey] 使用 go v1.23.9 构建 KubeVirt
- [[PR #15102]](https://github.com/kubevirt/kubevirt/pull/15102)[dominikholler] 将依赖 golang.org/x/net 更新至 v0.38.0

## v1.5.0 {#v150}

发布日期：Wed Mar 05 2025

KubeVirt v1.5 面向 Kubernetes v1.32 构建，并额外支持此前两个版本。更多信息请参阅 [KubeVirt 支持矩阵](https://github.com/kubevirt/sig-release/blob/main/releases/k8s-support-matrix.md)。

要查看为本版本做出贡献的出色个人名单，请参阅 [KubeVirt v1.5.0 发行标签](https://github.com/kubevirt/kubevirt/releases/tag/v1.5.0)。

### API 变更 {#api-change_5}
- [[PR #13850]](https://github.com/kubevirt/kubevirt/pull/13850)[nirdothan] 可以将网络接口状态设置为 `down` 或 `up`，从而相应地设置链路状态。
- [[PR #13708]](https://github.com/kubevirt/kubevirt/pull/13708)[orelmisan] 将为 VMI spec 中存在的接口报告网络接口的链路状态
- [[PR #13208]](https://github.com/kubevirt/kubevirt/pull/13208)[davidvossel] 为 virtctl 和 API 添加 VM 重置功能
- [[PR #13711]](https://github.com/kubevirt/kubevirt/pull/13711)[ShellyKa13] VMSnapshot：在选择 volumesnapshotclass 时遵循 StorageProfile 的 snapshotClass
- [[PR #13667]](https://github.com/kubevirt/kubevirt/pull/13667)[arnongilboa] 当存储超出配额时设置 VM 状态指示
- [[PR #13288]](https://github.com/kubevirt/kubevirt/pull/13288)[alicefr] VolumeUpdateStrategy 与 VolumeMigration 功能门控已毕业
- [[PR #13520]](https://github.com/kubevirt/kubevirt/pull/13520)[iholder101] 将克隆 API 毕业到 v1beta1，并弃用 v1alpha1
- [[PR #13110]](https://github.com/kubevirt/kubevirt/pull/13110)[alicefr] 添加 iothreads 选项，用于指定要使用的 iothreads 数量
- [[PR #13152]](https://github.com/kubevirt/kubevirt/pull/13152)[akalenyu] VMExport：导出的 DV 使用存储 API
- [[PR #13305]](https://github.com/kubevirt/kubevirt/pull/13305)[ShellyKa13] VMRestore：从 vmrestore webhook 中移除 VMSnapshot 逻辑
- [[PR #13243]](https://github.com/kubevirt/kubevirt/pull/13243)[orelmisan] 动态 Pod 接口命名宣告为 GA
- [[PR #13314]](https://github.com/kubevirt/kubevirt/pull/13314)[EdDev] 网络绑定插件功能宣告为 GA
- [[PR #13195]](https://github.com/kubevirt/kubevirt/pull/13195)[ShellyKa13] Vmrestore - 增加用于处理目标尚未就绪情况的选项
- [[PR #12750]](https://github.com/kubevirt/kubevirt/pull/12750)[lyarwood] 不够灵活的 `PreferredUseEFi` 和 `PreferredUseSecureBoot` 偏好字段已弃用，将在未来版本的 `instancetype.kubevirt.io` API 中移除。用户应改用 `PreferredEfi`，以提供首选的 `EFI` 配置，用于其 `VirtualMachine`。

### 破坏性变更 {#breaking-change}
- [[PR #13497]](https://github.com/kubevirt/kubevirt/pull/13497)[tiraboschi] 作为加固措施（最小权限原则），创建、编辑和删除 `VirtualMachineInstanceMigrations` 的权限不再默认授予命名空间管理员。

### 缺陷修复 {#bug-fix_5}
- [[PR #13803]](https://github.com/kubevirt/kubevirt/pull/13803)[ShellyKa13] 缺陷修复：VMSnapshot：等待卷被绑定，而不是跳过
- [[PR #13713]](https://github.com/kubevirt/kubevirt/pull/13713)[akalenyu] 增强：预先向 libvirt 声明哪些文件系统是共享的，以便在某些基于 NFS 的置备器上允许迁移
- [[PR #13682]](https://github.com/kubevirt/kubevirt/pull/13682)[alromeros] 缺陷修复：支持对带有后端存储的 VM 进行在线快照
- [[PR #13606]](https://github.com/kubevirt/kubevirt/pull/13606)[dasionov] 为 amd64 添加 virtio 视频设备支持
- [[PR #13207]](https://github.com/kubevirt/kubevirt/pull/13207)[alromeros] 缺陷修复：支持对带有后端存储的 VM 进行离线快照
- [[PR #13546]](https://github.com/kubevirt/kubevirt/pull/13546)[akalenyu] 缺陷修复：在 crun >= 1.18 上卷热插拔无法工作
- [[PR #13496]](https://github.com/kubevirt/kubevirt/pull/13496)[0xFelix] virtctl expose 现在使用每个 virt-launcher Pod 上独有的 `vm.kubevirt.io/name` 标签作为 Service 选择器。
- [[PR #13547]](https://github.com/kubevirt/kubevirt/pull/13547)[0xFelix] virtctl create vm 会校验磁盘名称，并阻止那些会导致 VM 在创建时被拒绝的磁盘名称。
- [[PR #13544]](https://github.com/kubevirt/kubevirt/pull/13544)[jean-edouard] 修复了 VM 可能无法获得其所请求的持久 EFI 的缺陷
- [[PR #13460]](https://github.com/kubevirt/kubevirt/pull/13460)[alromeros] 缺陷修复：支持导出后端 PVC
- [[PR #13424]](https://github.com/kubevirt/kubevirt/pull/13424)[fossedihelm] 缺陷修复：修复关机期间可能出现的 virt-handler 竞态条件与卡住情况
- [[PR #13426]](https://github.com/kubevirt/kubevirt/pull/13426)[dasionov] 缺陷修复：阻止对旧迁移进行状态更新
- [[PR #13367]](https://github.com/kubevirt/kubevirt/pull/13367)[xpivarc] 缺陷修复：降低误报警告 "failed to detect socket for containerDisk disk0: ... connection refused" 的概率
- [[PR #13138]](https://github.com/kubevirt/kubevirt/pull/13138)[mhenriks] 获取文件系统开销时避免 NPE
- [[PR #13270]](https://github.com/kubevirt/kubevirt/pull/13270)[ShellyKa13] VMSnapshot：传播冻结失败错误
- [[PR #13260]](https://github.com/kubevirt/kubevirt/pull/13260)[akalenyu] 缺陷修复：带有实例类型和偏好的 VM 出现 VMSnapshot 处于 'InProgress' 且失败的情况
- [[PR #13240]](https://github.com/kubevirt/kubevirt/pull/13240)[awels] 修复当该 VM 存在已成功/已失败的 VMI 时无法启动虚拟机导出的问题
- [[PR #13219]](https://github.com/kubevirt/kubevirt/pull/13219)[jean-edouard] backend-storage 现在会正确使用默认的虚拟化存储类
- [[PR #13197]](https://github.com/kubevirt/kubevirt/pull/13197)[akalenyu] 缺陷修复：OpenShift 上的 VMSnapshot 无法工作

### 弃用 {#deprecation_3}
- [[PR #13871]](https://github.com/kubevirt/kubevirt/pull/13871)[0xFelix] 现在默认使用运行 `virtctl ssh` 的机器上的本地 SSH 客户端。`--local-ssh` 标志现已弃用。
- [[PR #13918]](https://github.com/kubevirt/kubevirt/pull/13918)[0xFelix] virtctl port-forward/ssh/scp 语法中 `type` 为可选项现已弃用。
- [[PR #13817]](https://github.com/kubevirt/kubevirt/pull/13817)[Barakmor1] `AutoResourceLimits` 功能门控现已弃用，该功能状态已毕业为 `GA`，因此默认启用
- [[PR #13437]](https://github.com/kubevirt/kubevirt/pull/13437)[arnongilboa] 移除已弃用的 DataVolume 垃圾回收测试

### SIG-compute {#sig-compute_5}
- [[PR #13936]](https://github.com/kubevirt/kubevirt/pull/13936)[kubevirt-bot] 将 common-instancetypes 捆绑包更新至 v1.3.0
- [[PR #13838]](https://github.com/kubevirt/kubevirt/pull/13838)[iholder101] 为按时间定义的缓存添加 KeepValueUpdated() 方法
- [[PR #13642]](https://github.com/kubevirt/kubevirt/pull/13642)[0xFelix] VMPool 中的 VM 能够通过分别建立索引的 ConfigMap 和 Secret 接收各自的配置。
- [[PR #12624]](https://github.com/kubevirt/kubevirt/pull/12624)[victortoso] 更好地处理带有 Slic 表的不受支持卷类型
- [[PR #13756]](https://github.com/kubevirt/kubevirt/pull/13756)[germag] 支持对带有（virtiofs）文件系统设备的 VMI 进行实时迁移
- [[PR #13777]](https://github.com/kubevirt/kubevirt/pull/13777)[0xFelix] virtctl：名称中带有点号的 VM/VMI 现已在 virtctl portforward、ssh 和 scp 中得到支持。
- [[PR #11266]](https://github.com/kubevirt/kubevirt/pull/11266)[jean-edouard] KubeVirt 将不再在工作节点上部署自定义 SELinux 策略
- [[PR #13562]](https://github.com/kubevirt/kubevirt/pull/13562)[kubevirt-bot] 将 common-instancetypes 捆绑包更新至 v1.2.1
- [[PR #13263]](https://github.com/kubevirt/kubevirt/pull/13263)[jean-edouard] 节点上的 /var/lib/kubelet 现在可以是符号链接
- [[PR #13252]](https://github.com/kubevirt/kubevirt/pull/13252)[iholder101] 无条件禁用 libvirt 的 VMPort 功能，该功能仅与 VMWare 相关
- [[PR #12705]](https://github.com/kubevirt/kubevirt/pull/12705)[iholder101] 自动配置并行的 QEMU 级迁移线程（即 multifd）
- [[PR #12925]](https://github.com/kubevirt/kubevirt/pull/12925)[0xFelix] virtctl：镜像上传最多重试 15 次

### SIG-storage {#sig-storage_5}
- [[PR #13857]](https://github.com/kubevirt/kubevirt/pull/13857)[ShellyKa13] VMSnapshot：允许在源尚不存在时创建快照
- [[PR #13864]](https://github.com/kubevirt/kubevirt/pull/13864)[alromeros] 当源使用后端存储 PVC 时拒绝 VM 克隆
- [[PR #13717]](https://github.com/kubevirt/kubevirt/pull/13717)[alicefr] 拒绝将卷迁移到使用非 CSI 存储类的旧版 datavolume
- [[PR #13586]](https://github.com/kubevirt/kubevirt/pull/13586)[akalenyu] 存储测试：组装面向存储的一致性测试套件
- [[PR #13603]](https://github.com/kubevirt/kubevirt/pull/13603)[akalenyu] 存储测试：消除运行时跳过
- [[PR #12800]](https://github.com/kubevirt/kubevirt/pull/12800)[alicefr] 为已热插拔的卷启用卷迁移
- [[PR #12628]](https://github.com/kubevirt/kubevirt/pull/12628)[ShellyKa13] VM 准入器：从 webhook 中移除对 vm clone 卷的校验
- [[PR #13091]](https://github.com/kubevirt/kubevirt/pull/13091)[acardace] 将 `VMLiveUpdateFeatures` 功能门控毕业为 GA。

### SIG-network {#sig-network_5}
- [[PR #13775]](https://github.com/kubevirt/kubevirt/pull/13775)[sbrivio-rh] 此版本的 KubeVirt 升级了提供用户模式网络的 passt 软件包，以匹配上游版本 2025_01_21.4f2c8e7。
- [[PR #13458]](https://github.com/kubevirt/kubevirt/pull/13458)[orelmisan] 调整 managedTap 绑定，使其适用于启用了地址冲突检测的 VM

### SIG-scale {#sig-scale_4}
- [[PR #13204]](https://github.com/kubevirt/kubevirt/pull/13204)[Sreeja1725] 添加 v1.4.0 发行版的性能与规模基准数据
- [[PR #12546]](https://github.com/kubevirt/kubevirt/pull/12546)[Sreeja1725] 更新 sig-performance 测试中 CPU 与内存指标的 promql 查询

### SIG-observability {#sig-observability_5}
- [[PR #13610]](https://github.com/kubevirt/kubevirt/pull/13610)[avlitman] 添加了 kubevirt_vm_vnic_info 和 kubevirt_vmi_vnic_info 指标
- [[PR #13535]](https://github.com/kubevirt/kubevirt/pull/13535)[machadovilaca] 从 VM 实例类型/偏好收集资源请求与限制
- [[PR #13428]](https://github.com/kubevirt/kubevirt/pull/13428)[machadovilaca] 添加 kubevirt_vmi_migration_(start|end)_time_seconds 指标
- [[PR #13423]](https://github.com/kubevirt/kubevirt/pull/13423)[machadovilaca] 添加 kubevirt_vmi_migration_data_total_bytes 指标
- [[PR #13587]](https://github.com/kubevirt/kubevirt/pull/13587)[sradco] 告警 KubevirtVmHighMemoryUsage 已弃用。
- [[PR #13431]](https://github.com/kubevirt/kubevirt/pull/13431)[avlitman] 添加 kubevirt_vm_create_date_timestamp_seconds 指标
- [[PR #13386]](https://github.com/kubevirt/kubevirt/pull/13386)[machadovilaca] 确保 kubevirt_vmi_status_addresses 指标中的 IP 不为空
- [[PR #13250]](https://github.com/kubevirt/kubevirt/pull/13250)[Sreeja1725] 添加 virt-handler 的 CPU 与内存用量指标
- [[PR #13325]](https://github.com/kubevirt/kubevirt/pull/13325)[machadovilaca] 为迁移指标添加节点标签
- [[PR #13148]](https://github.com/kubevirt/kubevirt/pull/13148)[avlitman] 为 kubevirt_vmi_info 指标添加名为 vmi_pod 的新标签，其中包含运行该 VMI 的当前 Pod 名称。
- [[PR #13294]](https://github.com/kubevirt/kubevirt/pull/13294)[machadovilaca] 将客户机内存与大页内存添加到 kubevirt_vm_resource_requests
- [[PR #12765]](https://github.com/kubevirt/kubevirt/pull/12765)[avlitman] 添加了 kubevirt_vm_disk_allocated_size_bytes 指标，用于监控 VM 大小

### 未分类 {#uncategorized}
- [[PR #11964]](https://github.com/kubevirt/kubevirt/pull/11964)[ShellyKa13] VMClone：移除检查 VM 源的 webhook
- [[PR #11997]](https://github.com/kubevirt/kubevirt/pull/11997)[jcanocan] 放弃 `ExperimentalVirtiofsSupport` 功能门控，改为使用 `EnableVirtioFsConfigVolumes` 共享 ConfigMap、Secret、DownwardAPI 和 ServiceAccount，并使用 `EnableVirtioFsPVC` 共享 PVC。
- [[PR #13109]](https://github.com/kubevirt/kubevirt/pull/13109)[xpivarc] 测试套件：新增 3 个可用于过滤测试的标签：HostDiskGate、requireHugepages1Gi、blockrwo
- [[PR #13588]](https://github.com/kubevirt/kubevirt/pull/13588)[Yu-Jack] 确保 virt-tail 与 virt-monitor 具有相同的超时，防止 virt-monitor 仍在启动时 virt-tail 提前终止
- [[PR #13545]](https://github.com/kubevirt/kubevirt/pull/13545)[alicefr] 升级 virt 技术栈
- [[PR #12844]](https://github.com/kubevirt/kubevirt/pull/12844)[jschintag] 为 s390x 启用 virt-exportproxy 和 virt-exportserver 镜像
- [[PR #13006]](https://github.com/kubevirt/kubevirt/pull/13006)[chomatdam] 为 VM Export 资源添加了标签、注解以及可配置的 Pod 就绪超时
- [[PR #13699]](https://github.com/kubevirt/kubevirt/pull/13699)[brianmcarey] 使用 go v1.23.4 构建 KubeVirt
- [[PR #13641]](https://github.com/kubevirt/kubevirt/pull/13641)[andreabolognani] 此版本的 KubeVirt 包含基于 libvirt 10.10.0 和 QEMU 9.1.0 的升级后虚拟化技术。
- [[PR #13495]](https://github.com/kubevirt/kubevirt/pull/13495)[brianmcarey] 使用 go v1.22.10 构建 KubeVirt


## v1.4.0 {#v140}

发布日期：Wed Nov 13 2024

KubeVirt v1.4 面向 Kubernetes v1.31 构建，并额外支持此前两个版本。更多信息请参阅 [KubeVirt 支持矩阵](https://github.com/kubevirt/sig-release/blob/main/releases/k8s-support-matrix.md)。

要查看为本版本做出贡献的非常优秀的人们名单，请参阅 [KubeVirt v1.4.0 发行标签](https://github.com/kubevirt/kubevirt/releases/tag/v1.4.0)。

### API 变更 {#api-change_6}
- [[PR #13030]](https://github.com/kubevirt/kubevirt/pull/13030) [alicefr] 从 VolumeMigrationState 中移除了 ManualRecoveryRequired 字段，并将其转换为 VM 状况 ManualRecoveryRequired
- [[PR #12933]](https://github.com/kubevirt/kubevirt/pull/12933) [ShellyKa13] VM 准入器：改进对 vm spec datavolumetemplate 的校验
- [[PR #12986]](https://github.com/kubevirt/kubevirt/pull/12986) [lyarwood] `PreferredEfi` 偏好现在仅在用户尚未启用 `EFI` 或 `BIOS` 时应用于底层 `VirtualMachine`。
- [[PR #12169]](https://github.com/kubevirt/kubevirt/pull/12169) [lyarwood] `PreferredDiskDedicatedIoThread` 现在仅应用于 `virtio` 磁盘设备
- [[PR #13090]](https://github.com/kubevirt/kubevirt/pull/13090) [acardace] 允许实时更新 VM 的容忍
- [[PR #12629]](https://github.com/kubevirt/kubevirt/pull/12629) [jean-edouard] backend-storage 现在支持 RWO FS
- [[PR #13086]](https://github.com/kubevirt/kubevirt/pull/13086) [lyarwood] 新增可配置项 `spec.configuration.instancetype.referencePolicy`，已加入 `KubeVirt` CR，并提供对 `reference`（默认）、`expand` 和 `expandAll` 策略的支持。
- [[PR #12967]](https://github.com/kubevirt/kubevirt/pull/12967) [xpivarc] BochsDisplayForEFIGuests 已 GA。若升级前需要保持兼容性，请在 Kubevirt CR 上使用 "kubevirt.io/vga-display-efi-x86" 注解。
- [[PR #13001]](https://github.com/kubevirt/kubevirt/pull/13001) [awels] 放宽 VM 快照期间修改 VM spec 的检查，仅检查磁盘/卷
- [[PR #13018]](https://github.com/kubevirt/kubevirt/pull/13018) [orelmisan] 支持动态主 Pod NIC 名称
- [[PR #13078]](https://github.com/kubevirt/kubevirt/pull/13078) [qinqon] 添加动态 Pod 接口名称功能门控
- [[PR #13059]](https://github.com/kubevirt/kubevirt/pull/13059) [EdDev] 网络热插拔功能宣告为 GA。
- [[PR #12753]](https://github.com/kubevirt/kubevirt/pull/12753) [lyarwood] `CommonInstancetypesDeploymentGate` 功能门控及其底层功能已毕业为 GA，并且现在默认始终启用。同时引入了一个新的 `KubeVirt` 可配置项，使集群管理员可以在需要时显式禁用部署。
- [[PR #12232]](https://github.com/kubevirt/kubevirt/pull/12232) [lyarwood] `NUMA` 功能门控现已弃用，该功能状态已毕业为 `GA`，因此默认启用
- [[PR #12943]](https://github.com/kubevirt/kubevirt/pull/12943) [Barakmor1] `GPU` 功能门控现已弃用，该功能状态已毕业为 `GA`，因此默认启用

### 缺陷修复 {#bug-fix_6}
- [[PR #12829]](https://github.com/kubevirt/kubevirt/pull/12829) [0xFelix] 修复：kubeconfig 中配置的代理会在 client-go 中用于 VNC 或控制台等异步子资源
- [[PR #12733]](https://github.com/kubevirt/kubevirt/pull/12733) [alromeros] 缺陷修复：通过检查可用大小而非请求容量来修复磁盘扩容逻辑
- [[PR #13040]](https://github.com/kubevirt/kubevirt/pull/13040) [awels] 缺陷修复：允许 VMExport 用于名称中带有点号的 VM 卷
- [[PR #12867]](https://github.com/kubevirt/kubevirt/pull/12867) [jschintag] 修复了某些镜像清单中额外损坏的 amd64 镜像
- [[PR #12861]](https://github.com/kubevirt/kubevirt/pull/12861) [ShellyKa13] 缺陷修复：修复 vmrestore 恢复时可能遗漏更新 datavolumename 的问题
- [[PR #12599]](https://github.com/kubevirt/kubevirt/pull/12599) [xpivarc] MaxCpuSockets 不再阻止创建插槽数多于 MaxCpuSockets 所声明数量的 VM
- [[PR #12857]](https://github.com/kubevirt/kubevirt/pull/12857) [akalenyu] 缺陷修复：无法通过 virtctl vmexport create 创建 VMExport
- [[PR #12835]](https://github.com/kubevirt/kubevirt/pull/12835) [ShellyKa13] 缺陷修复：若 vmrestore 出错，则去掉 VM 上的 RestoreInProgress 注解，使其能够启动
- [[PR #12809]](https://github.com/kubevirt/kubevirt/pull/12809) [dasionov] 缺陷修复：确保与 VMI 关联的 PDB 在其达到 Succeeded 或 Failed 阶段时被删除
- [[PR #12813]](https://github.com/kubevirt/kubevirt/pull/12813) [akalenyu] 缺陷修复：无法在 OpenShift 上创建导出 Pod
- [[PR #12764]](https://github.com/kubevirt/kubevirt/pull/12764) [ShellyKa13] 缺陷修复：vmrestore 在创建/更新已恢复的 VM 之前创建 DV
- [[PR #12638]](https://github.com/kubevirt/kubevirt/pull/12638) [akalenyu] 缺陷修复：containerdisk VM 出现 "Cannot allocate memory" 警告
- [[PR #12592]](https://github.com/kubevirt/kubevirt/pull/12592) [awels] 修复了在 VMExport 设置期间实际并未创建 Secret 时仍发出已创建 Secret 事件的问题
- [[PR #12460]](https://github.com/kubevirt/kubevirt/pull/12460) [mhenriks] virt-api：对授权额外头取消编码
- [[PR #12451]](https://github.com/kubevirt/kubevirt/pull/12451) [fossedihelm] 修复：对已完成的 virt-launcher Pod 的驱逐请求不能触发实时迁移
- [[PR #12261]](https://github.com/kubevirt/kubevirt/pull/12261) [fossedihelm] 修复：持久 TPM 可用于名称中包含点号的 VMI
- [[PR #12181]](https://github.com/kubevirt/kubevirt/pull/12181) [akalenyu] 缺陷修复：授予命名空间管理员 RBAC，以便将客户端 USB 直通给 VMI
- [[PR #12096]](https://github.com/kubevirt/kubevirt/pull/12096) [machadovilaca] 修复 VMI 资源缺失的性能指标
- [[PR #12212]](https://github.com/kubevirt/kubevirt/pull/12212) [acardace] 仅对内存 >= 1Gi 的 VM 启用
- [[PR #12193]](https://github.com/kubevirt/kubevirt/pull/12193) [acardace] 修复卡在 Provisioning 阶段的 RerunOnFailure
- [[PR #12180]](https://github.com/kubevirt/kubevirt/pull/12180) [0xFelix] 启用 NetworkInterfaceMultiqueue 且仅有单个插槽的 VM 需要重启，才能热插拔额外的 CPU 插槽。
- [[PR #12128]](https://github.com/kubevirt/kubevirt/pull/12128) [acardace] 内存热插拔的修复与稳定化
- [[PR #11911]](https://github.com/kubevirt/kubevirt/pull/11911) [alromeros] 缺陷修复：在导出服务器和 vmexport 中实现重试机制
- [[PR #12119]](https://github.com/kubevirt/kubevirt/pull/12119) [acardace] 修复将 `LiveUpdate` 用作 `vmRolloutStrategy` 时的 VMPool。
- [[PR #12209]](https://github.com/kubevirt/kubevirt/pull/12209) [orenc1] 修复升级场景中错误的 KubeVirtVMIExcessiveMigrations 告警计算。
- [[PR #13050]](https://github.com/kubevirt/kubevirt/pull/13050) [vamsikrishna-siddu] 修复 s390x 的 CPU 型号问题。
- [[PR #13027]](https://github.com/kubevirt/kubevirt/pull/13027) [awels] 缺陷修复：在没有要导出的卷时停止创建 tokenSecretRef
- [[PR #12613]](https://github.com/kubevirt/kubevirt/pull/12613) [orelmisan] 桥接绑定：通往包含该 Pod NIC IP 地址的子网的静态路由会传递给 VM。
- [[PR #13203]](https://github.com/kubevirt/kubevirt/pull/13203) [kubevirt-bot] 缺陷修复：OpenShift 上的 VMSnapshot 无法工作
- [[PR #13225]](https://github.com/kubevirt/kubevirt/pull/13225) [kubevirt-bot] backend-storage 现在会正确使用默认的虚拟化存储类


### 弃用 {#deprecation_4}
- [[PR #13019]](https://github.com/kubevirt/kubevirt/pull/13019) [0xFelix] virtctl：标志 `--volume-clone-pvc`、`--volume-datasource` 和 `--volume-blank` 已弃用，请改用 `--volume-import` 标志。
- [[PR #12940]](https://github.com/kubevirt/kubevirt/pull/12940) [Barakmor1] 弃用 DockerSELinuxMCS 功能门控
- [[PR #12578]](https://github.com/kubevirt/kubevirt/pull/12578) [dasionov] 将 Running 字段标记为已弃用
- [[PR #11927]](https://github.com/kubevirt/kubevirt/pull/11927) [lyarwood] 所有 `preferredCPUTopology` 常量中以 `Prefer` 为前缀的项已弃用，并将在未来版本的 `instancetype.kubevirt.io` API 中移除。

### SIG-compute {#sig-compute_6}
- [[PR #12848]](https://github.com/kubevirt/kubevirt/pull/12848) [iholder101] 将默认 CompletionTimeoutPerGiB 从 800s 降至 150s
- [[PR #12739]](https://github.com/kubevirt/kubevirt/pull/12739) [lyarwood] 已向偏好新增 `PreferredEfi` 字段，用于表达首选的 `EFI` 配置，适用于给定的 `VirtualMachine`。
- [[PR #12617]](https://github.com/kubevirt/kubevirt/pull/12617) [Acedus] go.mod 中的 grpc 现在会正确地随发行镜像一起提供
- [[PR #12419]](https://github.com/kubevirt/kubevirt/pull/12419) [nunnatsa] 为校验 webhook 添加超时
- [[PR #11881]](https://github.com/kubevirt/kubevirt/pull/11881) [lyarwood] `expand-spec` 子资源 API 现在会对返回的 `VirtualMachine` 应用默认值，以确保其中的 `VirtualMachineInstanceSpec` 更接近启动原始 `VirtualMachine` 时最终使用的版本。
- [[PR #12268]](https://github.com/kubevirt/kubevirt/pull/12268) [fossedihelm] 从 client-go 中移除 `ForceRestart` 和 `ForceStop` 方法
- [[PR #12053]](https://github.com/kubevirt/kubevirt/pull/12053) [vladikr] 每个 VMI 只会配置一个带有 ramfb 的 vgpu 显示选项。
- [[PR #11982]](https://github.com/kubevirt/kubevirt/pull/11982) [RamLavi] 引入 validatingAdmissionPolicy，以限制对 virt-handler 的节点补丁
- [[PR #13053]](https://github.com/kubevirt/kubevirt/pull/13053) [0xFelix] virtctl：用户可以在使用 virtctl create vm 创建的 VM 中指定 sysprep 卷
- [[PR #12855]](https://github.com/kubevirt/kubevirt/pull/12855) [0xFelix] virtctl expose：移除用于设置已弃用 LoadBalancerIP 选项的标志
- [[PR #13008]](https://github.com/kubevirt/kubevirt/pull/13008) [0xFelix] virtctl：允许使用 virtctl create vm 创建基本的 cloud-init 配置
- [[PR #12786]](https://github.com/kubevirt/kubevirt/pull/12786) [0xFelix] virtctl：创建的 VM 现在可以从 PVC、Registry 和 Snapshot 源推断实例类型或偏好。
- [[PR #12557]](https://github.com/kubevirt/kubevirt/pull/12557) [codingben] 可选择使用 virtctl image upload 创建数据源。
- [[PR #13072]](https://github.com/kubevirt/kubevirt/pull/13072) [0xFelix] virtctl：virtctl create vm 现在可以使用 Access Credentials API 向新 VM 添加凭据

### SIG-storage {#sig-storage_6}
- [[PR #12355]](https://github.com/kubevirt/kubevirt/pull/12355) [alicefr] 在 VM 状态中添加卷迁移状态
- [[PR #12726]](https://github.com/kubevirt/kubevirt/pull/12726) [awels] 使用 virtctl 并发执行 addvolume/removevolume 时，即使发生并发修改也不再失败
- [[PR #12582]](https://github.com/kubevirt/kubevirt/pull/12582) [mhenriks] vmsnapshot：检查 VM 是否正在运行时忽略 runStrategy
- [[PR #12605]](https://github.com/kubevirt/kubevirt/pull/12605) [mhenriks] vmexport：为 VirtualMachineExport 启用 status 子资源
- [[PR #12547]](https://github.com/kubevirt/kubevirt/pull/12547) [mhenriks] virt-api：当 DataVolume 已存在时跳过克隆授权检查
- [[PR #12395]](https://github.com/kubevirt/kubevirt/pull/12395) [alicefr] 为 VMIStorageLiveMigratable 添加新状况
- [[PR #12194]](https://github.com/kubevirt/kubevirt/pull/12194) [mhenriks] VM 支持 kubevirt.io/immediate-data-volume-creation: "false"，从而推迟创建 DataVolumeTemplates，直到 VM 启动
- [[PR #12254]](https://github.com/kubevirt/kubevirt/pull/12254) [jkinred] * 降低了未找到 `VolumeSnapshotClass` 时日志消息的严重级别。当某个卷未启用快照时，原因仍会显示在 `status.volumeSnapshotStatuses` 字段中，该字段属于 `VirtualMachine` 资源。
- [[PR #12601]](https://github.com/kubevirt/kubevirt/pull/12601) [mhenriks] vmsnapshot：为 snapshot.kubevirt.io API 组启用 status 子资源

### SIG-network {#sig-network_6}
- [[PR #13024]](https://github.com/kubevirt/kubevirt/pull/13024) [EdDev] 网络绑定插件：引入新的 `managedTap` `domainAttachmentType`
- [[PR #13060]](https://github.com/kubevirt/kubevirt/pull/13060) [EdDev] 网络绑定插件功能宣告为 Beta。
- [[PR #12235]](https://github.com/kubevirt/kubevirt/pull/12235) [orelmisan] 网络绑定插件：允许指定计算内存开销
- [[PR #11802]](https://github.com/kubevirt/kubevirt/pull/11802) [matthewei] 为 VMCloneAPI 添加 newMacAddresses 校验 webhook
- [[PR #11754]](https://github.com/kubevirt/kubevirt/pull/11754) [nickolaev] 添加对 `igb` 网络接口型号的支持
- [[PR #12354]](https://github.com/kubevirt/kubevirt/pull/12354) [qinqon] 在 passt 绑定边车处使用可选接口
- [[PR #13018]](https://github.com/kubevirt/kubevirt/pull/13018) [orelmisan] 支持动态主 Pod NIC 名称

### SIG-scale {#sig-scale_5}
- [[PR #12117]](https://github.com/kubevirt/kubevirt/pull/12117) [Sreeja1725] 将 kwok 与 sig-scale 测试集成
- [[PR #12716]](https://github.com/kubevirt/kubevirt/pull/12716) [Sreeja1725] 更新 kubevirt_rest_client_request_latency_seconds，以便在使用查询参数发起 list 调用时计入这些调用
- [[PR #12247]](https://github.com/kubevirt/kubevirt/pull/12247) [Sreeja1725] 为 v1.3 发行版添加性能规模基准
- [[PR #12116]](https://github.com/kubevirt/kubevirt/pull/12116) [Sreeja1725] 向 kubevirt 基准添加组件的 CPU/内存利用率指标

### 监控 {#monitoring}
- [[PR #13045]](https://github.com/kubevirt/kubevirt/pull/13045) [dasionov] 为 kubevirt_vm_info 指标添加 'machine_type' 标签
- [[PR #12992]](https://github.com/kubevirt/kubevirt/pull/12992) [machadovilaca] 为 kubevirt_vmi_info 指标添加 'outdated' 标签
- [[PR #12645]](https://github.com/kubevirt/kubevirt/pull/12645) [avlitman] 添加 kubevirt_vmsnapshot_succeeded_timestamp_seconds 指标
- [[PR #12718]](https://github.com/kubevirt/kubevirt/pull/12718) [machadovilaca] 添加 kubevirt_vm_info 指标
- [[PR #12737]](https://github.com/kubevirt/kubevirt/pull/12737) [machadovilaca] 为 kubevirt_vmi_info 添加 evictable 标签
- [[PR #12625]](https://github.com/kubevirt/kubevirt/pull/12625) [machadovilaca] 为 CPU 资源添加 kubevirt_vm_resource_requests
- [[PR #12593]](https://github.com/kubevirt/kubevirt/pull/12593) [machadovilaca] 为内存资源添加 kubevirt_vm_resource_requests 指标
- [[PR #12910]](https://github.com/kubevirt/kubevirt/pull/12910) [machadovilaca] 将 kubevirt_vm_resource_requests 的 'vmi' 标签重命名为 'name'
- [[PR #12441]](https://github.com/kubevirt/kubevirt/pull/12441) [machadovilaca] 提高 domainstats 迁移指标的周期性
- [[PR #13071]](https://github.com/kubevirt/kubevirt/pull/13071) [machadovilaca] 添加 kubevirt_vm_resource_limits 指标
- [[PR #12802]](https://github.com/kubevirt/kubevirt/pull/12802) [machadovilaca] 添加 kubevirt_vmi_status_addresses 指标

### 未分类 {#uncategorized_1}
- [[PR #11097]](https://github.com/kubevirt/kubevirt/pull/11097) [vamsikrishna-siddu] 为 kubevirt builder 添加 s390x 支持
- [[PR #10562]](https://github.com/kubevirt/kubevirt/pull/10562) [dhiller] 继续推进 Ginkgo V2 串行运行方面的更改
- [[PR #12476]](https://github.com/kubevirt/kubevirt/pull/12476) [jschintag] 在 s390x 上启用实时迁移和节点标签
- [[PR #12616]](https://github.com/kubevirt/kubevirt/pull/12616) [orenc1] 将 `Update()` 替换为 `Patch()`，用于 `test VirtualMachineInstancesPerNode`
- [[PR #12575]](https://github.com/kubevirt/kubevirt/pull/12575) [Barakmor1] 建议用户在 virt-api 消息中使用 RunStrategy
- [[PR #12195]](https://github.com/kubevirt/kubevirt/pull/12195) [awels] Virt 导出路由的边缘终止方式为 redirect
- [[PR #12594]](https://github.com/kubevirt/kubevirt/pull/12594) [tiraboschi] [测试] 为 Periodic_only 测试引入装饰器
- [[PR #12516]](https://github.com/kubevirt/kubevirt/pull/12516) [vamsikrishna-siddu] 为 s390x 启用初始 e2e 测试。
- [[PR #12466]](https://github.com/kubevirt/kubevirt/pull/12466) [orenc1] tests/vm_tests.go：将 Update() 替换为 Patch()
- [[PR #11856]](https://github.com/kubevirt/kubevirt/pull/11856) [Sreeja1725] 添加单元测试以检查 API 向后兼容性
- [[PR #12425]](https://github.com/kubevirt/kubevirt/pull/12425) [fudancoder] 修复一些注释
- [[PR #12452]](https://github.com/kubevirt/kubevirt/pull/12452) [andreabolognani] 此版本的 KubeVirt 包含基于 libvirt 10.5.0 和 QEMU 9.0.0 的升级后虚拟化技术。
- [[PR #12584]](https://github.com/kubevirt/kubevirt/pull/12584) [brianmcarey] 使用 go v1.22.6 构建 KubeVirt
- [[PR #13052]](https://github.com/kubevirt/kubevirt/pull/13052) [fossedihelm] 将代码生成器更新到 1.31.1
- [[PR #12882]](https://github.com/kubevirt/kubevirt/pull/12882) [brianmcarey] 使用 go v1.22.8 构建 KubeVirt
- [[PR #12729]](https://github.com/kubevirt/kubevirt/pull/12729) [fossedihelm] 将 k8s 依赖更新到 0.31.0
- [[PR #12186]](https://github.com/kubevirt/kubevirt/pull/12186) [kubevirt-bot] 将 common-instancetypes 捆绑包更新至 v1.0.1
- [[PR #12548]](https://github.com/kubevirt/kubevirt/pull/12548) [kubevirt-bot] 将 common-instancetypes 捆绑包更新至 v1.1.0
- [[PR #13082]](https://github.com/kubevirt/kubevirt/pull/13082) [kubevirt-bot] 将 common-instancetypes 捆绑包更新至 v1.2.0
- [[PR #12125]](https://github.com/kubevirt/kubevirt/pull/12125) [ksimon1] 杂项：将 virtio-win 镜像版本提升至 0.1.248

## v1.3.0 {#v130}

发布日期：Wed Jul 17 2024

KubeVirt v1.3 面向 Kubernetes v1.30 构建，并额外支持此前两个版本。更多信息请参阅 [KubeVirt 支持矩阵](https://github.com/kubevirt/sig-release/blob/main/releases/k8s-support-matrix.md)。

要查看为本版本做出贡献的优秀的伙伴名单，请参阅 [KubeVirt v1.3.0 发行标签](https://github.com/kubevirt/kubevirt/releases/tag/v1.3.0)。

### API 变更 {#api-change_7}
- [[PR #11156]](https://github.com/kubevirt/kubevirt/pull/11156) [nunnatsa] 将部分校验从 VMI 创建校验 webhook 移至 CRD
- [[PR #11500]](https://github.com/kubevirt/kubevirt/pull/11500) [iholder101] 支持 HyperV 透传：自动使用所有可用的 HyperV 功能
- [[PR #11641]](https://github.com/kubevirt/kubevirt/pull/11641) [alicefr] 添加 kubevirt.io/testWorkloadUpdateMigrationAbortion 注解，以及中止工作负载更新的机制
- [[PR #11700]](https://github.com/kubevirt/kubevirt/pull/11700) [alicefr] 添加 updateVolumeStrategy 字段
- [[PR #11729]](https://github.com/kubevirt/kubevirt/pull/11729) [lyarwood] 已向偏好引入 `spreadOptions`，以便对 `preferSpread` `preferredCPUTopology` 进行更细粒度的控制。这包括现在能够将 vCPU 分散到客户机可见的插槽、核心和线程。
- [[PR #10545]](https://github.com/kubevirt/kubevirt/pull/10545) [lyarwood] 包含实例类型和偏好的 `ControllerRevisions` 现在会在其所属的 `VirtualMachine` 被 `virt-controller` 重新同步时升级到最新可用版本。
- [[PR #11955]](https://github.com/kubevirt/kubevirt/pull/11955) [mhenriks] 引入 snapshot.kibevirt.io/v1beta1
- [[PR #11956]](https://github.com/kubevirt/kubevirt/pull/11956) [mhenriks] 引入 export.kibevirt.io/v1beta1
- [[PR #11533]](https://github.com/kubevirt/kubevirt/pull/11533) [alicefr] 实现卷迁移，并引入迁移的 updateVolumesStrategy 字段
- [[PR #10490]](https://github.com/kubevirt/kubevirt/pull/10490) [jschintag] 添加在 s390x 上构建和运行 kubevirt 的支持。
- [[PR #11330]](https://github.com/kubevirt/kubevirt/pull/11330) [jean-edouard] 在 VMI / 迁移对象的迁移状态中提供更多信息
- [[PR #11773]](https://github.com/kubevirt/kubevirt/pull/11773) [jean-edouard] 如果 CR 中未指定存储类，持久 TPM/UEFI 将使用默认存储类。

### 缺陷修复 {#bug-fix_7}
- [[PR #12296]](https://github.com/kubevirt/kubevirt/pull/12296) [orelmisan] 网络绑定插件：允许指定计算内存开销
- [[PR #12279]](https://github.com/kubevirt/kubevirt/pull/12279) [fossidhelm] 修复：持久 TPM 可用于名称中包含点号的 VMI
- [[PR #12226]](https://github.com/kubevirt/kubevirt/pull/12226) [awels] Virt 导出路由的边缘终止方式为 redirect
- [[PR #12249]](https://github.com/kubevirt/kubevirt/pull/12249) [machadovilaca] 修复 VMI 资源缺失的性能指标
- [[PR #12237]](https://github.com/kubevirt/kubevirt/pull/12237) [vladikr] 每个 VMI 只会配置一个带有 ramfb 的 vgpu 显示选项。
- [[PR #12064]](https://github.com/kubevirt/kubevirt/pull/12064) [akalenyu] 缺陷修复：对 VM（而非 VMI）资源的任何删除调用都会跳过优雅删除
- [[PR #11996]](https://github.com/kubevirt/kubevirt/pull/11996) [ShellyKa13] 缺陷修复：修复 volumesnapshot 缺失时恢复过程发生 panic 的问题
- [[PR #11973]](https://github.com/kubevirt/kubevirt/pull/11973) [fossedihelm] 缺陷修复：正确反映 RestartRequired 状况
- [[PR #11922]](https://github.com/kubevirt/kubevirt/pull/11922) [alromeros] 缺陷修复：修复导出控制器中的 VM 清单渲染
- [[PR #11367]](https://github.com/kubevirt/kubevirt/pull/11367) [alromeros] 缺陷修复：通过将日志打印到 stderr，允许 vmexport 下载重定向
- [[PR #11219]](https://github.com/kubevirt/kubevirt/pull/11219) [alromeros] 缺陷修复：改进对不兼容总线上 IOThread 的处理
- [[PR #11372]](https://github.com/kubevirt/kubevirt/pull/11372) [xpivarc] 缺陷修复：修复 VM 更新失败时的 nil panic
- [[PR #11267]](https://github.com/kubevirt/kubevirt/pull/11267) [mhenriks] 缺陷修复：确保在 DR 以及备份/恢复的情况下，由 virt-controller 创建的 DataVolume（DataVolumeTemplate）会被重新创建并由 VM 拥有。
- [[PR #10900]](https://github.com/kubevirt/kubevirt/pull/10900) [KarstenB] 缺陷修复：修复了 APIResourceList 不正确的 APIVersion
- [[PR #11306]](https://github.com/kubevirt/kubevirt/pull/11306) [fossedihelm] 修复（ksm）：如果 ksm 由 KubeVirt 管理，则将 `kubevirt.io/ksm-enabled` 节点标签设置为 true，而不是反映实际的 ksm 值。
- [[PR #11264]](https://github.com/kubevirt/kubevirt/pull/11264) [machadovilaca] 修复性能规模分桶错误
- [[PR #11058]](https://github.com/kubevirt/kubevirt/pull/11058) [fossedihelm] 修复（vmclone）：当目标 VM 被删除时删除 vmclone 资源
- [[PR #11265]](https://github.com/kubevirt/kubevirt/pull/11265) [xpivarc] 缺陷修复：VM 控制器不再破坏其缓存
- [[PR #11205]](https://github.com/kubevirt/kubevirt/pull/11205) [akalenyu] 修复在 cgroupsv2 上热插拔块卷之后，若 VM 拥有 rng 设备则迁移中断的问题
- [[PR #11051]](https://github.com/kubevirt/kubevirt/pull/11051) [alromeros] 缺陷修复：改进 fsfreeze 失败时的错误报告
- [[PR #12016]](https://github.com/kubevirt/kubevirt/pull/12016) [acardace] 修复使用 Manual RunStrategy 启动 VM 的问题
- [[PR #11963]](https://github.com/kubevirt/kubevirt/pull/11963) [acardace] 修复 RerunOnFailure RunStrategy
- [[PR #11718]](https://github.com/kubevirt/kubevirt/pull/11718) [fossedihelm] 修复：如果提供了代理服务器配置，client-go 中的 SEV 方法现在会满足该配置
- [[PR #12122]](https://github.com/kubevirt/kubevirt/pull/12122) [kubevirt-bot] 修复将 `LiveUpdate` 用作 `vmRolloutStrategy` 时的 VMPool。
- [[PR #12201]](https://github.com/kubevirt/kubevirt/pull/12201) [kubevirt-bot] 修复卡在 Provisioning 阶段的 RerunOnFailure
- [[PR #12151]](https://github.com/kubevirt/kubevirt/pull/12151) [kubevirt-bot] 缺陷修复：在导出服务器和 vmexport 中实现重试机制
- [[PR #12146]](https://github.com/kubevirt/kubevirt/pull/12146) [kubevirt-bot] 内存热插拔的修复与稳定化
- [[PR #12185]](https://github.com/kubevirt/kubevirt/pull/12185) [kubevirt-bot] 启用 NetworkInterfaceMultiqueue 且仅有单个插槽的 VM 需要重启，才能热插拔额外的 CPU 插槽。
- [[PR #12171]](https://github.com/kubevirt/kubevirt/pull/12171) [kubevirt-bot] `PreferredDiskDedicatedIoThread` 现在仅应用于 `virtio` 磁盘设备

### 弃用 {#deprecation_5}
- [[PR #11701]](https://github.com/kubevirt/kubevirt/pull/11701) [EdDev] SLIRP 核心绑定已弃用并被移除。
- [[PR #11901]](https://github.com/kubevirt/kubevirt/pull/11901) [EdDev] 'macvtap' 核心网络绑定已停止提供并被移除。
- [[PR #11915]](https://github.com/kubevirt/kubevirt/pull/11915) [ormergi] 'passt' 核心网络绑定已停止提供并被移除。
- [[PR #11404]](https://github.com/kubevirt/kubevirt/pull/11404) [avlitman] KubeVirtComponentExceedsRequestedCPU 与 KubeVirtComponentExceedsRequestedMemory 告警已弃用；它们并不表示真正的问题。

### SIG-compute {#sig-compute_7}
- [[PR #11498]](https://github.com/kubevirt/kubevirt/pull/11498) [acardace] 允许为设置了内存限制的 VM 热插拔内存
- [[PR #11479]](https://github.com/kubevirt/kubevirt/pull/11479) [vladikr] 虚拟机实例在 postcopy 迁移中断后将不再卡在不可恢复的状态。这些实例将会失败，并且可以再次启动。
- [[PR #11685]](https://github.com/kubevirt/kubevirt/pull/11685) [fossedihelm] 将 client-go 的 go 版本更新到 1.21
- [[PR #11344]](https://github.com/kubevirt/kubevirt/pull/11344) [aerosouund] 重构设备插件，以使用基础插件并定义通用接口
- [[PR #12025]](https://github.com/kubevirt/kubevirt/pull/12025) [fossedihelm] 添加 descheduler 兼容性
- [[PR #12109]](https://github.com/kubevirt/kubevirt/pull/12109) [acardace] 支持与大页配合使用的内存热插拔
- [[PR #11883]](https://github.com/kubevirt/kubevirt/pull/11883) [orelmisan] 当 CPU 插槽数量减少时，需要重启 VM
- [[PR #11655]](https://github.com/kubevirt/kubevirt/pull/11655) [acardace] 允许为设置了 CPU 请求和/或限制的 VM 热插拔 vCPU
- [[PR #11455]](https://github.com/kubevirt/kubevirt/pull/11455) [lyarwood] 现在支持对使用实例类型的 VM 进行 `LiveUpdates`，其注意事项与更改普通 VM 时相同。
- [[PR #11681]](https://github.com/kubevirt/kubevirt/pull/11681) [lyarwood] `CommonInstancetypesDeployment` 功能及其门控被追溯地从 1.2.0 发行版移至 Beta。
- [[PR #11648]](https://github.com/kubevirt/kubevirt/pull/11648) [kubevirt-bot] 将 common-instancetypes 捆绑包更新至 v1.0.0
- [[PR #12240]](https://github.com/kubevirt/kubevirt/pull/12240) [kubevirt-bot] 将 common-instancetypes 捆绑包更新至 v1.0.1

### SIG-storage {#sig-storage_7}
- [[PR #11095]](https://github.com/kubevirt/kubevirt/pull/11095) [ShellyKa13] 在 vmsnapshot 对象中暴露 volumesnapshot 错误
- [[PR #11312]](https://github.com/kubevirt/kubevirt/pull/11312) [alromeros] 改进 virtctl vmexport 中对导出资源的处理
- [[PR #11770]](https://github.com/kubevirt/kubevirt/pull/11770) [alicefr] 修复卷和磁盘的实时更新
- [[PR #11957]](https://github.com/kubevirt/kubevirt/pull/11957) [mhenriks] snapshot：如果 VMSnapshot 正在删除，则忽略解冻错误

### SIG-network {#sig-network_7}
- [[PR #11653]](https://github.com/kubevirt/kubevirt/pull/11653) [EdDev] 静态构建 `passt`custom CNI 二进制文件，用于 `passt` 网络绑定插件。
- [[PR #11678]](https://github.com/kubevirt/kubevirt/pull/11678) [Vicente-Cheng] 改进升级时对序数 Pod 接口名称的处理
- [[PR #11618]](https://github.com/kubevirt/kubevirt/pull/11618) [AlonaKaplan] 扩展网络绑定插件，以支持 device-info DownwardAPI。
- [[PR #11256]](https://github.com/kubevirt/kubevirt/pull/11256) [andreabolognani] 此版本的 KubeVirt 包含基于 libvirt 10.0.0 和 QEMU 8.2.0 的升级后虚拟化技术。
- [[PR #11788]](https://github.com/kubevirt/kubevirt/pull/11788) [ormergi] network-info 注解现在用于在 SR-IOV 网络与底层设备 PCI 地址之间建立映射
- [[PR #11659]](https://github.com/kubevirt/kubevirt/pull/11659) [iholder101] 要求将基础设施组件调度到控制平面节点
- [[PR #12079]](https://github.com/kubevirt/kubevirt/pull/12079) [EdDev] 网络热插拔功能宣告为 Beta。

### SIG-scale {#sig-scale_6}
- [[PR #11272]](https://github.com/kubevirt/kubevirt/pull/11272) [dharmit] 将 hook 边车注解中的 'image' 字段设为可选。
- [[PR #11676]](https://github.com/kubevirt/kubevirt/pull/11676) [machadovilaca] 重命名 REST 客户端指标，使其包含 kubevirt 前缀
- [[PR #11387]](https://github.com/kubevirt/kubevirt/pull/11387) [alaypatel07] 为 v1.2 发行版添加性能规模基准

### 监控 {#monitoring_1}
- [[PR #11307]](https://github.com/kubevirt/kubevirt/pull/11307) [machadovilaca] 为指标添加 e2e 测试
- [[PR #11294]](https://github.com/kubevirt/kubevirt/pull/11294) [machadovilaca] 修复 kubevirt_vm_created_total 被按 virt-api Pod 拆分的问题
- [[PR #11557]](https://github.com/kubevirt/kubevirt/pull/11557) [avlitman] 新增名为 kubevirt_memory_delta_from_requested_bytes 的内存统计
- [[PR #11283]](https://github.com/kubevirt/kubevirt/pull/11283) [assafad] 将来自客户机代理的 VMI 操作系统信息收集为 `kubevirt_vmi_phase_count` 指标标签
- [[PR #11906]](https://github.com/kubevirt/kubevirt/pull/11906) [machadovilaca] 创建 kubevirt_vmi_info 指标
- [[PR #11934]](https://github.com/kubevirt/kubevirt/pull/11934) [assafad] 添加 kubevirt_vmi_last_connection_timestamp_seconds 指标
- [[PR #12000]](https://github.com/kubevirt/kubevirt/pull/12000) [machadovilaca] 创建 kubevirt_vmi_launcher_memory_overhead_bytes 指标
- [[PR #11484]](https://github.com/kubevirt/kubevirt/pull/11484) [jcanocan] 将 downwardMetrics 服务器每秒最大请求数降至 1。

### 未分类 {#uncategorized_2}
- [[PR #12132]](https://github.com/kubevirt/kubevirt/pull/12132) [kubevirt-bot] 引入 validatingAdmissionPolicy，以限制对 virt-handler 的节点补丁
- [[PR #12009]](https://github.com/kubevirt/kubevirt/pull/12009) [xpivarc] 启用 NodeRestriction 功能门控后，Kubevirt 现在会授权 virt-handler 修改 VM 的请求。
- [[PR #12089]](https://github.com/kubevirt/kubevirt/pull/12089) [jean-edouard] 权限更低的 virt-operator ClusterRole
- [[PR #11969]](https://github.com/kubevirt/kubevirt/pull/11969) [iholder101] 基础设施组件针对控制平面节点的 NoSchedule 容忍
- [[PR #11835]](https://github.com/kubevirt/kubevirt/pull/11835) [talcoh2x] 将 Intel Gaudi 添加到采用者名单。
- [[PR #11790]](https://github.com/kubevirt/kubevirt/pull/11790) [aburdenthehand] 将 Cloudflare 重新加入我们的 ADOPTERS 名单
- [[PR #11331]](https://github.com/kubevirt/kubevirt/pull/11331) [anjuls] 将 cloudraft 添加到采用者名单。
- [[PR #11942]](https://github.com/kubevirt/kubevirt/pull/11942) [ido106] 更新 virtctl，使常规镜像上传和异步镜像上传都使用 v1beta1 端点
- [[PR #11908]](https://github.com/kubevirt/kubevirt/pull/11908) [victortoso] sidecar-shim：允许来自二进制 hook 的 stderr 日志
- [[PR #11846]](https://github.com/kubevirt/kubevirt/pull/11846) [victortoso] SMBios 边车可以在树外构建
- [[PR #11482]](https://github.com/kubevirt/kubevirt/pull/11482) [brianmcarey] 使用 go v1.22.2 构建 KubeVirt
- [[PR #11470]](https://github.com/kubevirt/kubevirt/pull/11470) [brianmcarey] 使用 Go 1.21.8 构建 KubeVirt
- [[PR #12097]](https://github.com/kubevirt/kubevirt/pull/12097) [fossedihelm] 将 k8s 依赖提升至 0.30.0
- [[PR #11416]](https://github.com/kubevirt/kubevirt/pull/11416) [dhiller] 在使用 `FIt` 进行程序化聚焦时发出 k8s 日志
- [[PR #11183]](https://github.com/kubevirt/kubevirt/pull/11183) [dhiller] 扩展 sig-buildsystem 的 OWNERS
- [[PR #11149]](https://github.com/kubevirt/kubevirt/pull/11149) [0xFelix] virtctl：现在可以在创建 VM 时从 GCS 导入卷
- [[PR #11146]](https://github.com/kubevirt/kubevirt/pull/11146) [RamLavi] node-labeller：移除过时功能

## v1.2.0 {#v120}

发布日期：Tue Mar 05 2024

KubeVirt v1.2 面向 Kubernetes v1.29 构建，并额外支持此前两个版本。更多信息请参阅 [KubeVirt 支持矩阵](https://github.com/kubevirt/sig-release/blob/main/releases/k8s-support-matrix.md)。

### API 变更 {#api-change_8}
- [[PR #11064]](https://github.com/kubevirt/kubevirt/pull/11064) [AlonaKaplan] 引入新的 API，用于将绑定插件标记为可迁移。
- [[PR #10970]](https://github.com/kubevirt/kubevirt/pull/10970) [alromeros] 通过 GuestOsInfo 暴露文件系统磁盘信息
- [[PR #10905]](https://github.com/kubevirt/kubevirt/pull/10905) [tiraboschi] 在 VMI（进而在 VM）上聚合 DV 状况
- [[PR #10872]](https://github.com/kubevirt/kubevirt/pull/10872) [RamLavi] IsolateEmulatorThread：添加集群范围的功能对等完成设置
- [[PR #10846]](https://github.com/kubevirt/kubevirt/pull/10846) [RamLavi] 将 vm.status.PrintableStatus 的默认值更改为 "Stopped"
- [[PR #10774]](https://github.com/kubevirt/kubevirt/pull/10774) [victortoso] 借助 ACPI SLIC 表进行 Windows 离线激活
- [[PR #10732]](https://github.com/kubevirt/kubevirt/pull/10732) [AlonaKaplan] 通过向网络绑定插件 API 添加域附加选项来扩展 kubvirt CR。
- [[PR #10658]](https://github.com/kubevirt/kubevirt/pull/10658) [matthewei] 支持使用“克隆 API”过滤 VirtualMachine.spec.template.annotation 和 VirtualMachine.spec.template.label

### 缺陷修复 {#bug-fix_8}
- [[PR #11271]](https://github.com/kubevirt/kubevirt/pull/11271) [kubevirt-bot] 缺陷修复：VM 控制器不再破坏其缓存
- [[PR #11242]](https://github.com/kubevirt/kubevirt/pull/11242) [kubevirt-bot] 修复在 cgroupsv2 上热插拔块卷之后，若 VM 拥有 rng 设备则迁移中断的问题
- [[PR #11069]](https://github.com/kubevirt/kubevirt/pull/11069) [ormergi] 缺陷修复：VM 实时迁移初始阶段出现数据包丢弃 https://issues.redhat.com/browse/CNV-28040
- [[PR #11065]](https://github.com/kubevirt/kubevirt/pull/11065) [fossedihelm] 修复（vmclone）：根据 vmsnapshotcontent 而不是当前 VM 生成 VM 补丁
- [[PR #11050]](https://github.com/kubevirt/kubevirt/pull/11050) [fossedihelm] 将默认集群角色限制为仅已认证用户
- [[PR #11047]](https://github.com/kubevirt/kubevirt/pull/11047) [jschintag] 修复在没有任何 USB 设备的宿主机上尝试列出 USB 设备时可能发生的崩溃
- [[PR #10963]](https://github.com/kubevirt/kubevirt/pull/10963) [alromeros] 缺陷修复：未指定输出时拒绝卷导出
- [[PR #10916]](https://github.com/kubevirt/kubevirt/pull/10916) [orelmisan] 修复 VMI `Status.GuestOSInfo.Version` 的值
- [[PR #10888]](https://github.com/kubevirt/kubevirt/pull/10888) [fossedihelm] [缺陷修复] 克隆带有 WaitForFirstConsumer 绑定模式 PVC 的 VM 现在可以正常工作。
- [[PR #10860]](https://github.com/kubevirt/kubevirt/pull/10860) [akalenyu] 缺陷修复：带过滤器的双重克隆失败
 isolateEmulatorThread 功能（BZ#2228103）。
- [[PR #10845]](https://github.com/kubevirt/kubevirt/pull/10845) [orelmisan] 当目标名称等于源名称时，拒绝创建 VirtualMachineClone
- [[PR #10753]](https://github.com/kubevirt/kubevirt/pull/10753) [victortoso] 修复使用 USB 宿主机直通时的权限
- [[PR #10747]](https://github.com/kubevirt/kubevirt/pull/10747) [acardace] 通过在迁移 VMI 时使用校验和验证 containerdisk，使 KubeVirt 适配 CRIO 1.28
- [[PR #10699]](https://github.com/kubevirt/kubevirt/pull/10699) [qinqon] virt-launcher：修复 qemu 非 root 日志路径
- [[PR #10689]](https://github.com/kubevirt/kubevirt/pull/10689) [akalenyu] 缺陷修复：cgroupsv2 设备允许列表绑定到 virt-handler 内部状态/热插拔时块磁盘设备被覆盖
- [[PR #10593]](https://github.com/kubevirt/kubevirt/pull/10593) [RamLavi] 通过优化修复 virt-launcher Pod 中的 SMT 对齐错误

### 弃用 {#deprecation_6}
- [[PR #10924]](https://github.com/kubevirt/kubevirt/pull/10924) [AlonaKaplan] 弃用 macvtap

### SIG-compute {#sig-compute_8}
- [[PR #11054]](https://github.com/kubevirt/kubevirt/pull/11054) [jean-edouard] 新增集群范围的 `vmRolloutStrategy` 设置，用于定义对 VM 的更改应当始终暂存，还是在可能时实时更新。
- [[PR #11001]](https://github.com/kubevirt/kubevirt/pull/11001) [fossedihelm] 允许 `kubevirt.io:default` clusterRole 对 kubevirt 执行 get、list
- [[PR #10961]](https://github.com/kubevirt/kubevirt/pull/10961) [jcanocan] 缩短节点故障时 VM 的重新调度时间
- [[PR #10918]](https://github.com/kubevirt/kubevirt/pull/10918) [orelmisan] VMClone：在创建恢复失败时发出事件
- [[PR #10898]](https://github.com/kubevirt/kubevirt/pull/10898) [matthewei] VMI 状态的 guestOsInfo 增加了 `Machine`
- [[PR #10840]](https://github.com/kubevirt/kubevirt/pull/10840) [acardace] 使用 CPU/内存热插拔时，现在可以配置请求/限制
- [[PR #10839]](https://github.com/kubevirt/kubevirt/pull/10839) [RamLavi] 将第二个模拟器线程的分配策略更改为尽力而为。
- [[PR #10809]](https://github.com/kubevirt/kubevirt/pull/10809) [orelmisan] 源 virt-launcher：默认记录迁移信息
- [[PR #10783]](https://github.com/kubevirt/kubevirt/pull/10783) [RamLavi] 在 Housekeeping cgroup 中支持多个 CPU
- [[PR #10571]](https://github.com/kubevirt/kubevirt/pull/10571) [tiraboschi] 开启客户机串行控制台日志记录（默认开启）时，VMI 内存占用增加 35M。

### SIG-storage {#sig-storage_8}
- [[PR #10657]](https://github.com/kubevirt/kubevirt/pull/10657) [germag] 使用非特权 virtiofsd 向 VM 暴露文件系统持久卷（PV）。
- [[PR #10529]](https://github.com/kubevirt/kubevirt/pull/10529) [alromeros] 允许热插拔 LUN 磁盘

### SIG-network {#sig-network_8}
- [[PR #10981]](https://github.com/kubevirt/kubevirt/pull/10981) [AlonaKaplan] 报告使用网络绑定插件的接口 IP。
- [[PR #10866]](https://github.com/kubevirt/kubevirt/pull/10866) [AlonaKaplan] 在使用 passt 功能门控或 API 时报告错误。
- [[PR #10800]](https://github.com/kubevirt/kubevirt/pull/10800) [AlonaKaplan] 支持将 macvtap 作为绑定插件
- [[PR #10425]](https://github.com/kubevirt/kubevirt/pull/10425) [ormergi] 引入用于 Passt 网络的网络绑定插件，与 Kubevirt 新的网络绑定插件 API 对接。

### SIG-infra {#sig-infra}
- [[PR #11025]](https://github.com/kubevirt/kubevirt/pull/11025) [0xFelix] 默认允许非特权用户以只读方式访问 VirtualMachineCluster{Instancetypes,Preferences}。
- [[PR #10922]](https://github.com/kubevirt/kubevirt/pull/10922) [kubevirt-bot] 将 common-instancetypes 捆绑包更新至 v0.4.0

### 监控 {#monitoring_2}
- [[PR #10982]](https://github.com/kubevirt/kubevirt/pull/10982) [machadovilaca] 重构监控指标
- [[PR #10962]](https://github.com/kubevirt/kubevirt/pull/10962) [machadovilaca] 更新监控文件结构
- [[PR #10853]](https://github.com/kubevirt/kubevirt/pull/10853) [machadovilaca] 重构监控采集器
- [[PR #10700]](https://github.com/kubevirt/kubevirt/pull/10700) [machadovilaca] 重构监控告警
- [[PR #10693]](https://github.com/kubevirt/kubevirt/pull/10693) [machadovilaca] 移除 MigrateVmiDiskTransferRateMetric
- [[PR #10651]](https://github.com/kubevirt/kubevirt/pull/10651) [machadovilaca] 重构监控记录规则
- [[PR #10570]](https://github.com/kubevirt/kubevirt/pull/10570) [machadovilaca] 修复 LowKVMNodesCount 未触发的问题
- [[PR #10418]](https://github.com/kubevirt/kubevirt/pull/10418) [machadovilaca] 添加已创建 VM 总数指标

### 未分类 {#uncategorized_3}
- [[PR #11144]](https://github.com/kubevirt/kubevirt/pull/11144) [0xFelix] virtctl：创建 VM 并使用 --volume-import 克隆 PVC 或 VolumeSnapshot 时，指定大小现在是可选的
- [[PR #11122]](https://github.com/kubevirt/kubevirt/pull/11122) [brianmcarey] 将 runc 依赖更新至 v1.1.12
- [[PR #11068]](https://github.com/kubevirt/kubevirt/pull/11068) [brianmcarey] 更新容器基础镜像，以使用当前稳定的 debian 12 基础镜像
- [[PR #10914]](https://github.com/kubevirt/kubevirt/pull/10914) [brianmcarey] KubeVirt 现在使用 go 1.21.5 构建
- [[PR #10879]](https://github.com/kubevirt/kubevirt/pull/10879) [brianmcarey] 使用 golang 1.20.12 构建
- [[PR #10863]](https://github.com/kubevirt/kubevirt/pull/10863) [dhiller] 从生成代码的版权声明中移除年份
- [[PR #10787]](https://github.com/kubevirt/kubevirt/pull/10787) [matthewei] virtctl 支持添加模板标签和注解过滤器
- [[PR #10720]](https://github.com/kubevirt/kubevirt/pull/10720) [awels] 将热插拔附加 Pod 的请求/限制恢复为原始值
- [[PR #10637]](https://github.com/kubevirt/kubevirt/pull/10637) [dharmit] 针对带有 ConfigMap 的边车 hook 的功能测试
- [[PR #10615]](https://github.com/kubevirt/kubevirt/pull/10615) [orelmisan] 移除残留的 NonRoot 功能门控
- [[PR #10598]](https://github.com/kubevirt/kubevirt/pull/10598) [alicefr] 为 hook 边车添加 PVC 选项，以便提供额外的调试工具
- [[PR #10596]](https://github.com/kubevirt/kubevirt/pull/10596) [mhenriks] 禁用 HTTP/2 以缓解 CVE-2023-44487
- [[PR #10582]](https://github.com/kubevirt/kubevirt/pull/10582) [orelmisan] 移除残留的 NonRootExperimental 功能门控
- [[PR #10567]](https://github.com/kubevirt/kubevirt/pull/10567) [awels] 附加 Pod 的创建现在受到速率限制
- [[PR #10526]](https://github.com/kubevirt/kubevirt/pull/10526) [cfilleke] 记录构建 KubeVirt builder 容器的步骤
- [[PR #10479]](https://github.com/kubevirt/kubevirt/pull/10479) [dharmit] 能够通过 hook sidecardevice 运行脚本
- [[PR #10244]](https://github.com/kubevirt/kubevirt/pull/10244) [hshitomi] 在“virtctl”下添加了“adm”子命令，并在“adm”下添加了“log-verbosity”子命令。log-verbosity 命令用于：显示一个或多个组件的日志详细程度，设置一个或多个组件的日志详细程度，以及将所有组件的日志详细程度重置为默认值（2）。
- [[PR #10046]](https://github.com/kubevirt/kubevirt/pull/10046) [victortoso] 为 hook 添加 v1alpha3，并修复使用边车时的迁移

## v1.1.0 {#v110}

发布日期：Tue Nov 07 2023

### API 变更 {#api-change_9}
- [#10568][ormergi] 网络绑定插件 API 支持 CNI，并在创建 virt-launcher Pod 时提供新的集成点。
- [#10309][lyarwood] 集群范围的 [`common-instancetypes`](https://github.com/kubevirt/common-instancetypes) 资源现在可以由 `virt-operator` 使用 `CommonInstancetypesDeploymentGate` 功能门控进行部署。
- [#10463][0xFelix] VirtualMachine：在 InstancetypeMatcher 与 PreferenceMatcher 中引入 InferFromVolumeFailurePolicy
- [#10447][fossedihelm] 向 KV CR 添加功能门控，当创建所用命名空间关联了带有内存限制的资源配额时，自动设置内存限制
- [#10477][jean-edouard] 动态启用并配置 KSM
- [#10110][tiraboschi] 从专用容器流式传输客户机串行控制台日志
- [#10015][victortoso] 在 KubeVirt CRD 的 permittedHostDevices 中实现 USB 宿主机直通
- [#10184][acardace] 添加内存热插拔功能
- [#10231][kvaps] 将公钥传播到 cloud-init NoCloud 元数据
- [#9673][germag] DownwardMetrics：通过 virtio-serial 通道暴露 DownwardMetrics。
- [#10086][vladikr] 允许实时更新 VM 亲和性与节点选择器
- [#10272][ormergi] 引入用于 Slirp 网络的网络绑定插件，与 Kubevirt 新的网络绑定插件 API 对接。
- [#10284][AlonaKaplan] 引入网络绑定插件的 API。该功能位于 "NetworkBindingPlugins" 门控之后。
- [#10101][acardace] 弃用 KubeVirt CR 中的 `spec.config.machineType`。
- [#9878][jean-edouard] EFI NVRAM 现在可以配置为在重启之后保持持久
- [#9932][lyarwood] `ControllerRevisions` 在包含 `instancetype.kubevirt.io` `CRDs` 的情况下，现在带有详述底层暂存对象特定元数据的标签
- [#10058][alicefr] 为磁盘添加 errorPolicy 字段
- [#10004][AlonaKaplan] 接口的热插拔/拔出应通过更新 VM spec 模板完成。virtctl 与 REST API 端点已被移除。
- [#9896][ormergi] VM 控制器现在会将 spec 接口的 MAC 地址复制到 VMI spec 中的对应接口。
- [#7708][VirrageS] 已向 VirtualMachineInstancetype spec 添加 `nodeSelector` 和 `schedulerName` 字段。
- [#7197][vasiliy-ul] 通过新的 API 端点实验性支持 SEV 证明
- [#9737][AlonaKaplan] 热拔出时，从 virt-launcher 移除 bridge、tap 和 dummy 接口，并从节点移除缓存（文件缓存与易失缓存）。

### 缺陷修复： {#bug-fixes}
- [#10515][iholder101] 缺陷修复：停止在快照期间将 VMI spec 复制到 VM
- [#10393][iholder101] [缺陷修复] [克隆 API] 双重克隆现在按预期工作。
- [#10391][awels] 缺陷修复：VMExport 现在可以在定义了配额的命名空间中工作。
- [#10380][alromeros] 缺陷修复：允许 image-upload 从 PendingPopulation 阶段恢复
- [#10099][iholder101] 缺陷修复：迁移被取消时，目标 virt-launcher Pod 会挂起。
- [#10165][awels] 缺陷修复：删除热插拔附加 Pod 将不再分离那些并未移除的卷。
- [#10067][iholder101] 缺陷修复：`virtctl create clone` 的编组，以及将 `kubectl` 替换为 `kubectl virt`
- [#9935][xpivarc] 缺陷修复 - 修正容器磁盘中的日志记录
- [#9872][alromeros] 缺陷修复：允许将 lun 磁盘映射到 DataVolume 源
- [#10039][simonyangcj] 修复使用 virtiofs 时 virt-launcher Pod 的保证 QoS 遭到破坏的问题
- [#9861][rmohr] 修复通过 "virtctl restart" 请求强制重启时可能发生数据损坏的问题

### 弃用 {#deprecation_7}
- [#10486][assafad] 针对该 PR 中所列指标的弃用通知。请更新你的系统以使用新的指标名称。
- [#9821][sradco] 针对该 PR 中所列指标的弃用通知。请更新你的系统以使用新的指标名称。

### SIG-compute {#sig-compute_9}
- [#10566][fossedihelm] 为具有 dedicatedCPU 或需要 GuaranteedQos 的 VMI 增加 100Mi 内存开销
- [#10496][fossedihelm] 当创建所用命名空间关联了带有 CPU 限制的资源配额，且启用了 `AutoResourceLimits` 功能门控时，自动设置 CPU 限制
- [#10543][0xFelix] 在忽略推断失败时清除 VM 客户机内存
- [#10320][victortoso] sidecar-shim 实现了 PreCloudInitIso hook
- [#10253][rmohr] 停止尝试在 virt-controller 中创建未使用的目录 /var/run/kubevirt-ephemeral-disk
- [#10050][victortoso] 更新 virt 技术栈：QEMU 8.0.0、libvirt 至 9.5.0、edk2 20230524、passt 20230818、libguestfs 与 guestfs-tools 1.50.1、virtiofsd 1.7.2
- [#9231][victortoso] 引入 sidecar-shim 容器镜像
- [#10254][rmohr] 对于已是最新且就绪、但调度有误的 virt-handler Pod，不要将 KubeVirt 的 "Available" 状况标记为 false。
- [#10182][iholder101] 在查找集群上的最低 TSC 频率时，不再考虑没有 `kubevirt.io/schedulable` 标签的节点
- [#10056][jean-edouard] UEFI 客户机现在使用 Bochs 显示，而不再使用 VGA 模拟
- [#10106][acardace] 在以暂停状态启动 VM 时添加启动菜单等待时间。

### SIG-storage {#sig-storage_9}
- [#10532][alromeros] 在 image-upload 中添加 --volume-mode 标志
- [#10020][akalenyu] 对 DataVolume 使用认证 API，停止导入 kubevirt.io/containerized-data-importer
- [#10400][alromeros] 添加新的 vmexport 标志以下载原始镜像，既可以直接下载（--raw），也可以解压后下载（--decompress）
- [#10148][alromeros] 为 vmexport 添加端口转发功能
- [#10275][awels] 确保新的热插拔附加 Pod 就绪之后，再删除旧的附加 Pod
- [#10118][akalenyu] 更改 exportserver 的默认 UID，以便能够成功导出 CDI 独立 PVC（未挂接到 VM）
- [#9918][ShellyKa13] 修复使用 CDI populator 的 WFFC SCI 存储类上的热插拔

### SIG-network {#sig-network_9}
- [#10366][ormergi] Kubevirt 现在将 Slirp 网络配置委托给 Slirp 网络绑定插件。如果你尚未注册 Slirp 网络绑定插件镜像（即在 Kubevirt 配置中指定），将使用以下默认镜像：`quay.io/kubevirt/network-slirp-binding:20230830_638c60fc8`。在下一版本（v1.2.0）中将不再设置默认镜像，注册镜像将成为必需。
- [#10185][AlonaKaplan] 添加对基于迁移的 SRIOV 热插拔的支持。
- [#10116][ormergi] 状态为 'absent' 的现有已分离接口将从 VMI spec 中清除。
- [#9958][AlonaKaplan] 对 VMI 禁用网络接口热插拔/拔出。该能力将仅支持 VM。
- [#10489][maiqueb] 从 virt-controller 的 RBAC 中移除 network-attachment-definition 的 `list` 与 `watch` 动词

### SIG-infra {#sig-infra_1}
- [#10438][lyarwood] 新增 `instancetype.kubevirt.io:view` `ClusterRole`，可以通过 `ClusterRoleBinding` 绑定到用户，从而提供对集群范围 `VirtualMachineCluster{Instancetype,Preference}` 资源的只读访问。

### SIG-scale {#sig-scale_7}
- [#9989][alaypatel07] 为 VMI 添加性能规模基准

### 未分类 {#uncategorized_4}
- [#9590][xuzhenglun] 修复 virt-operator 中嵌入的版本信息
- [#10044][machadovilaca] 添加 operator-observability 包
- [#10450][0xFelix] virtctl：默认在 create vm 子命令中启用推断
- [#10386][liuzhen21] KubeSphere 已添加到采用者文件！
- [#10167][0xFelix] virtctl：将命名空间应用到所创建的清单
- [#10173][rmohr] 将 coordination/lease 的 RBAC 权限移至 Role
- [#10138][machadovilaca] 将 `kubevirt_vmi_*_usage_seconds` 从 Gauge 改为 Counter
- [#10107][PiotrProkop] 暴露 `kubevirt_vmi_vcpu_delay_seconds_total`，用于报告 VM 在队列中等待而不是运行所花费的秒数。
- [#10070][machadovilaca] 从 `kubevirt_vmi_cpu_affinity` 移除 affinities 标签，并使用总和作为值
- [#9982][fabiand] 引入支持生命周期以及 Kubernetes 目标版本。
- [#10001][machadovilaca] 修复未创建 `kubevirt_vmi_phase_count` 的问题
- [#9840][dhiller] 提高 flake 检查脚本发现 flake 的概率
- [#9988][enp0s3] 始终部署过期 VMI 工作负载告警
- [#9882][dhiller] 为初次贡献者补充关于自动化测试和草稿拉取请求的一些背景说明。
- [#9552][phoracek] gRPC 客户端现在可以正确地与非 Go 的 gRPC 服务器配合工作
- [#9818][akrejcir] 添加了 "virtctl credentials" 命令，用于动态更改 VM 中的 SSH 密钥，以及设置用户密码。
- [#9073][machadovilaca] 修复不正确的 KubevirtVmHighMemoryUsage 描述

## v1.0.0 {#v100}

发布日期：Thu Jul 11 17:39:42 2023 +0000

### API 变更 {#api-changes}
- [PR #9572][fossedihelm] 为新的非高性能 VMI 启用 freePageReporting
- [PR #8156][jean-edouard] 现在可以将 TPM VM 设备设置为持久
- [PR #8575][iholder101] QEMU 级迁移并行（即 multifd）+ 将 QEMU 升级到 7.2.0-11.el9
- [PR #9322][iholder101] 添加客户机相对请求内存的余量比例。
- [PR #9422][awels] 能够为支撑容器（热插拔/容器磁盘/virtiofs/边车）指定 CPU/内存的请求与限制
- [PR #9177][alicefr] 添加 SCSI 持久预留
- [PR #9145][awels] 在 VMExport 状态中显示 VirtualMachine 名称
- [PR #9491][orelmisan] API，AddInterfaceOptions：将 NetworkName 重命名为 NetworkAttachmentDefinitionName，并将 InterfaceName 重命名为 Name
- [PR #9442][EdDev] 移除 VMI Status 接口的 `podConfigDone` 字段，改为在 `infoSource` 中提供新的来源选项。
- [PR #6852][maiqueb] 开发预览：为 VM / VMI 启用网络接口热插拔
- [PR #9193][qinqon] 为实时迁移和桥接 Pod 接口添加注解
- [PR #9421][lyarwood] 现在会拒绝这样的请求：更新目标 `Name`（属于 `{Instancetype,Preference}Matcher`），但未同时更新 `RevisionName`。

### 缺陷修复 {#bug-fixes_1}
- [PR #9591][awels] 缺陷修复：允许多个 NFS 磁盘被使用/热插拔
- [PR #9536][akalenyu] 缺陷修复：执行 get all -n <namespace> 时会显示 virtualmachineclusterinstancetypes/preferences
- [PR #9300][xpivarc] 缺陷修复：当 VM 处于暂停状态时，由 API 和 virtctl 发起的迁移不会被拒绝
- [PR #9189][xpivarc] 缺陷修复：DNS 集成在迁移之后继续工作
- [PR #9241][akalenyu] 缺陷修复：Guestfs 镜像 URL 构造不正确
- [PR #9260][ShellyKa13] 修复可能重新触发内存转储的缺陷
- [PR #9478][xpivarc] 缺陷修复：修复迁移 Pod 被拒绝时迁移不会重试的情况。
- [PR #9330][qinqon] 从 virtctl expose 的 Service 选择器中跳过标签 kubevirt.io/migrationTargetNodeName
- [PR #9603][qinqon] 调整 node-labeller.sh 脚本，使其能够在带有模拟的非 kvm 环境中工作。

### 弃用 {#deprecation_8}
- [PR #9047][machadovilaca] 弃用 VM 卡在某一状态的告警

### SIG-compute {#sig-compute_10}
- [PR #9640][jean-edouard] 启用了 TSC 的 VM 现在可以迁移到频率并不相同（但足够接近）的节点
- [PR #9629][0xFelix] virtctl：创建 VM 时允许指定卷的启动顺序
- [PR #9435][rmohr] 在创建 VM 目标 Pod 之前，确保挂接到该 VMI 的所有 PVC 均已存在。
- [PR #9470][machadovilaca] 在已暂停的 VM 上启用 libvirt GetDomainStats
- [PR #9163][vladikr] 修复启用 isolatedEmulatorThread 的 VM 在请求/限制 CPU 数量上不匹配的问题
- [PR #9250][vladikr] 外部创建的中介设备不会被 virt-handler 删除

### SIG-storage {#sig-storage_10}
- [PR #9376][ShellyKa13] 修复使用可快照 WFFC 存储类时的 vmrestore
- [PR #9392][awels] virtctl 支持获取 VM 导出所用的 VM 清单
- [PR #9188][awels] 克隆与导出的默认 RBAC
- [PR #9133][ShellyKa13] 修复 addvolume 未拒绝添加已存在卷源的问题，并修复 removevolume 允许移除不可热插拔卷的问题

### SIG-network {#sig-network_10}
- [PR #9399][maiqueb] 基于当前域信息而不是接口状态，计算要热插拔的接口。
- [PR #9220][orelmisan] client-go：为 VirtualMachine 的方法添加了 context。

### SIG-infra {#sig-infra_2}
- [PR #9651][0xFelix] virtctl：允许指定所创建 VM 的内存。如果没有指定实例类型，也没有推断出实例类型，则默认为 512Mi。
- [PR #9169][lyarwood] `dedicatedCPUPlacement` 属性再次在 `VirtualMachineInstancetype` 和 `VirtualMachineClusterInstancetype` CRD 中得到支持。此前一项缺陷修复改进了 `VirtualMachine` 校验，确保在尝试验证之前先应用默认值。

### 未分类 {#uncategorized_5}
- [PR #9632][toelke] * 将 Genesis Cloud 添加到采用者名单
- [PR #9596][iholder101] 添加 "virtctl create clone" 命令
- [PR #9407][assafad] 使用环境变量 `RUNBOOK_URL_TEMPLATE` 作为运行手册 URL 模板
- [PR #9327][jcanocan] DownwardMetrics：在 VirtProductInfo 字段中用 qemu 版本替换 KubeVirt 构建信息
- [PR #9367][machadovilaca] 为 vmi_phase_count 指标添加 VM 实例类型和偏好标签
- [PR #8906][machadovilaca] 如果没有可用于运行 VM 的节点，则发出告警
- [PR #9320][darfux] node-labeller：在 handler 侧检查架构
- [PR #9127][fossedihelm] 使用 ECDSA 而不是 RSA 进行密钥生成
- [PR #9228][rumans] 提升 virtiofs 容器限制
- [PR #9159][andreabolognani] 此版本的 KubeVirt 包含基于 libvirt 9.0.0 和 QEMU 7.2.0 的升级后虚拟化技术。
- [PR #8989][rthallisey] 将多架构容器清单集成到 bazel make 配方中
- [PR #8937][fossedihelm] 为虚拟机添加了前台终结器

## v0.59.0 {#v0590}

发布于：Wed Mar 1 16:49:27 2023 +0000

- [PR #9311][kubevirt-bot] 修复启用 isolatedEmulatorThread 的 VM 在 requests/limits 上的 CPU 数量不匹配问题
- [PR #9276][fossedihelm] 为虚拟机添加了前台终结器
- [PR #9295][kubevirt-bot] 修复内存转储可能被再次触发的缺陷
- [PR #9270][kubevirt-bot] 缺陷修复：Guestfs 镜像 URL 未正确构造
- [PR #9234][kubevirt-bot] `dedicatedCPUPlacement` 属性再次可在 `VirtualMachineInstancetype` 与 `VirtualMachineClusterInstancetype` CRD 中使用。近期的一次缺陷修复改进了 `VirtualMachine` 校验，确保在尝试验证之前先应用默认值。
- [PR #9267][fossedihelm] 此版本的 KubeVirt 包含基于 libvirt 9.0.0 与 QEMU 7.2.0 升级的虚拟化技术。
- [PR #9197][kubevirt-bot] 修复 addvolume 未拒绝添加已存在卷源的问题，并修复 removevolume 允许移除非热插拔卷的问题
- [PR #9120][0xFelix] 修复集群角色 kubevirt.io:admin 与 kubevirt.io:edit 对 VM/VMI 端口转发的访问问题
- [PR #9116][EdDev] 允许在网络接口上指定 ACPI Index。
- [PR #8774][avlitman] 新增了虚拟机 CPU 指标：
- [PR #9087][zhuchenwang] 显式打开 `/dev/vhost-vsock`，以确保加载正确的 vsock 模块
- [PR #9020][feitnomore] 添加对 status/scale 子资源的支持，使 VirtualMachinePool 现在支持 HorizontalPodAutoscaler
- [PR #9085][0xFelix] virtctl：添加在创建 VM 时推断 instancetype 与偏好的选项
- [PR #8917][xpivarc] KubeVirt 可以配置 Seccomp 配置文件。现在为 launcher 附带自定义配置文件。
- [PR #9054][enp0s3] 不要将 LimitRange 默认值注入 VMI
- [PR #7862][vladikr] 将已最终确定的 VMI 迁移状态存储在迁移对象中。
- [PR #8878][0xFelix] 为 virtctl 添加 'create vm' 命令
- [PR #9048][jean-edouard] 引入 DisableCustomSELinuxPolicy 功能门控，用于禁用自定义 SELinux 策略
- [PR #8953][awels] VMExport 现在具有包含完整 VM 定义的端点。
- [PR #8976][iholder101] 修复 podman CRI 检测
- [PR #9043][iholder101] 调整 operator 功能测试，以适配自定义镜像的指定方式
- [PR #8875][machadovilaca] 重命名迁移指标，移除 'total' 关键字
- [PR #9040][lyarwood] `inferFromVolume` 现在使用标签而不是注解，以便从所引用的 `Volume` 查找默认实例类型与偏好详情。作出这一更改，是为了让用户能够通过这些标签查找经过适当标注的资源，然后再于 `VirtualMachine` 中指向它们。
- [PR #9039][orelmisan] client-go：为 VirtualMachineInstance 的更多方法添加了 context。
- [PR #9018][orelmisan] client-go：为 VirtualMachineInstance 的更多方法添加了 context。
- [PR #9025][akalenyu] 缺陷修复：热插拔 Pod 具有硬编码的资源请求，不符合 LimitRange 的 maxLimitRequestRatio 为 1 的要求
- [PR #8908][orelmisan] client-go：为 VirtualMachineInstance 的部分方法添加了 context。
- [PR #6863][rmohr] 安装策略 Job 从现在起将遵循基础设施节点放置配置
- [PR #8948][iholder101] 缺陷修复：virt-handler 套接字泄漏
- [PR #8649][acardace] KubeVirt 现在能够在 restricted 命名空间中运行 VM。
- [PR #8992][iholder101] 与 k8s 针对默认 LimitRange 要求的修复保持一致
- [PR #8889][rmohr] 为 vsock WebSocket 连接添加基本 TLS 加密支持
- [PR #8660][huyinhou] 修复 virt-api 日志中的 remoteAddress 字段在其为 IPv6 地址时被截断的问题
- [PR #8961][rmohr] 升级 distroless 基础镜像
- [PR #8952][rmohr] 修复只读 SATA 磁盘校验
- [PR #8657][fossedihelm] 发生 I/O 错误时，在重试启动 VM 之前使用逐渐增大的指数退避。
- [PR #8480][lyarwood] 新的 `inferFromVolume` 属性已被引入到 `{Instancetype,Preference}Matchers`（属于 `VirtualMachine`）。提供该属性后，会检查其所引用的 `Volume` 是否带有下列注解，并据此填充 `{Instancetype,Preference}Matchers`：
- [PR #7762][VirrageS] 服务 `kubevirt-prometheus-metrics` 现在将 `ClusterIP` 设为 `None`，从而使其成为无头服务。
- [PR #8599][machadovilaca] 将 KubevirtVmHighMemoryUsage 阈值从 20MB 更改为 50MB
- [PR #7761][VirrageS] 已向 KubeVirt CR 添加 imagePullSecrets 字段，以支持从私有镜像仓库部署
- [PR #8887][iholder101] 缺陷修复：如果提供了 virt operator 镜像，则使用该镜像
- [PR #8750][jordigilh] 修复了由于 libvirt 依赖 CAP_SYS_NICE 将 vCPU 线程的调度与优先级改为 FIFO 和 1，而导致非 root 配置下无法运行实时工作负载的问题。优先级与调度的更改现在由 virt-launcher 在 root 与非 root 配置下执行，从而消除了对 libvirt 的这一依赖。
- [PR #8845][lyarwood] 空的 `Timer` 现在会正确地从 `Clock` 中省略，从而修复缺陷 #8844。
- [PR #8842][andreabolognani] virt-launcher Pod 不再需要 SYS_PTRACE 权能。
- [PR #8734][alicefr] 更改 libguestfs-tools 镜像，改用 qcow2 格式的 root appliance
- [PR #8764][ShellyKa13] 在 vmSnapshot 中添加包含卷与排除卷的列表
- [PR #8811][iholder101] 自定义组件：支持 gs
- [PR #8770][dhiller] 为串行测试添加 Ginkgo V2 Serial 装饰器，作为简化并行与串行测试运行逻辑的准备
- [PR #8808][acardace] 仅对疏散迁移应用迁移退避。
- [PR #8525][jean-edouard] CR 选项 mediatedDevicesTypes 已弃用，请改用 mediatedDeviceTypes
- [PR #8792][iholder101] 向 csv-generator 与 manifest-templator 暴露新的自定义组件环境变量
- [PR #8701][enp0s3] 驱逐 VM 时考虑 ParallelOutboundMigrationsPerNode
- [PR #8740][iholder101] 修复：使 converter.go 与 template.go 之间的 Reenlightenment 流程保持一致
- [PR #8530][acardace] 对失败的迁移使用指数退避
- [PR #8720][0xFelix] expand-spec 子资源端点已重命名为 expand-vm-spec，并改为命名空间作用域
- [PR #8458][iholder101] 引入对以快照为源的克隆的支持（例如将快照克隆为 VM）
- [PR #8716][rhrazdil] 未指定端口时，计入使用 Passt 绑定的接口开销
- [PR #8619][fossedihelm] virt-launcher：使用 `virtqemud` 守护进程代替 `libvirtd`
- [PR #8736][knopt] 为 rest_client_request_latency_seconds 直方图添加了更精确的桶
- [PR #8624][zhuchenwang] 添加 REST API，以便通过 VSOCK 与客户机 VM 中的应用程序通信。
- [PR #8625][AlonaKaplan] masquerade 绑定不再使用 iptables。仅有 iptables 的节点将无法运行使用 masquerade 绑定的 VM。
- [PR #8673][iholder101] 允许为核心组件指定自定义镜像
- [PR #8622][jean-edouard] 使用 golang 1.19 构建
- [PR #8336][alicefr] 用于设置 guestfs uid 与 gid 的标志
- [PR #8667][huyinhou] 当 virt-launcher 工作目录不是 / 时，连接 VM VNC 失败
- [PR #8368][machadovilaca] 使用收集器设置迁移指标
- [PR #8558][xpivarc] 缺陷修复：当 VMI 缺少命名空间时，LimitRange 集成现在可以正常工作
- [PR #8404][andreabolognani] 此版本的 KubeVirt 包含基于 libvirt 8.7.0、QEMU 7.1.0 与 CentOS Stream 9 升级的虚拟化技术。
- [PR #8652][akalenyu] 缺陷修复：Exporter Pod 不符合 restricted PSA
- [PR #8563][xpivarc] KubeVirt 现在默认以非 root 用户运行
- [PR #8442][kvaps] 将 Deckhouse 加入采用者列表
- [PR #8546][zhuchenwang] 为 KubeVirt VM 提供 Vsock 功能。
- [PR #8598][acardace] 配置了大页的 VM 现在可以使用默认的 container_t SELinux 类型运行
- [PR #8594][kylealexlane] 修复在某些内核版本上进行 SELinux 重新标记时权限被拒绝的问题
- [PR #8521][akalenyu] 新增为 VMExport 对象指定 TTL 的选项
- [PR #7918][machadovilaca] 为处于不健康状态的 VM 添加告警
- [PR #8516][rhrazdil] 使用 Passt 绑定时，virl-launcher 将 unprivileged_port_start 设为 0，以便 passt 可以绑定所有端口。
- [PR #7772][jean-edouard] virt-launcher 的 SELinux 策略已减少到 4 条规则，其中 1 条用于大页，3 条用于 virtiofs。
- [PR #8402][jean-edouard] 大多数 VMI 现在以 SELinux 类型 container_t 运行
- [PR #8513][alromeros] [缺陷修复] 修复 virtctl image-upload 中的错误处理

## v0.58.1 {#v0581}

发布于：Thu Feb 11 00:08:46 2023 +0000

- [PR #9203][jean-edouard] 大多数 VMI 现在以 SELinux 类型 container_t 运行
- [PR #9191][kubevirt-bot] 为克隆与导出提供默认 RBAC
- [PR #9150][kubevirt-bot] 修复集群角色 kubevirt.io:admin 与 kubevirt.io:edit 对 VM/VMI 端口转发的访问问题
- [PR #9128][kubevirt-bot] 重命名迁移指标，移除 'total' 关键字
- [PR #9034][akalenyu] 缺陷修复：热插拔 Pod 具有硬编码的资源请求，不符合 LimitRange 的 maxLimitRequestRatio 为 1 的要求
- [PR #9002][iholder101] 缺陷修复：virt-handler 套接字泄漏
- [PR #8907][kubevirt-bot] 缺陷修复：如果提供了 virt operator 镜像，则使用该镜像
- [PR #8784][kubevirt-bot] 对失败的迁移使用指数退避
- [PR #8816][iholder101] 向 csv-generator、manifest-templator 与 gs 暴露新的自定义组件环境变量
- [PR #8798][iholder101] 修复：使 converter.go 与 template.go 之间的 Reenlightenment 流程保持一致
- [PR #8731][kubevirt-bot] 允许为核心组件指定自定义镜像
- [PR #8785][0xFelix] expand-spec 子资源端点已重命名为 expand-vm-spec，并改为命名空间作用域
- [PR #8806][kubevirt-bot] 驱逐 VM 时考虑 ParallelOutboundMigrationsPerNode
- [PR #8738][machadovilaca] 使用收集器设置迁移指标
- [PR #8747][kubevirt-bot] 为处于不健康状态的 VM 添加告警
- [PR #8685][kubevirt-bot] 缺陷修复：Exporter Pod 不符合 restricted PSA
- [PR #8647][akalenyu] 缺陷修复：新增为 VMExport 对象指定 TTL 的选项
- [PR #8609][kubevirt-bot] 修复在某些内核版本上进行 SELinux 重新标记时权限被拒绝的问题
- [PR #8578][rhrazdil] 使用 Passt 绑定时，virl-launcher 将 unprivileged_port_start 设为 0，以便 passt 可以绑定所有端口。


## v0.58.0 {#v0580}

发布于：Thu Oct 13 00:24:51 2022 +0000

- [PR #8578][rhrazdil] 使用 Passt 绑定时，virl-launcher 将 unprivileged_port_start 设为 0，以便 passt 可以绑定所有端口。
- [PR #8463][Barakmor1] 改进指标文档
- [PR #8282][akrejcir] 改进 instancetype 与偏好控制器的修订版本。这是一项向后不兼容的变更，并为 instancetype 与偏好引入了新的 v1alpha2 API。
- [PR #8272][jean-edouard] kubevirt-cr 清单中不再包含空节
- [PR #8536][qinqon] 如果 ConfigDrive cloud init 具有 UserDataSecretRef 但没有 NetworkDataSecretRef，则不显示失败
- [PR #8375][xpivarc] Virtiofs 可与 Nonroot 功能门控一起使用
- [PR #8465][rmohr] 添加 vnc screenshot REST 端点以及 "virtctl vnc screenshot" 命令，用于 UI 与脚本集成
- [PR #8418][alromeros] 为 VirtualMachineExport 对象启用自动令牌生成
- [PR #8488][0xFelix] virtctl：使用本地 ssh 客户端时降低输出详细程度
- [PR #8396][alicefr] 添加 group 标志，用于在 guestfs 中设置 gid 与 fsgroup
- [PR #8476][iholder-redhat] 允许通过 KubeVirt CR 设置 virt-operator 的日志详细程度
- [PR #8366][rthallisey] 将 KubeVirt 改为 15 周的发布节奏
- [PR #8479][arnongilboa] 在 cluster-deploy 中默认启用 DataVolume GC
- [PR #8474][vasiliy-ul] 修复了在使用 containerd 的系统上，带有 containerdisk 的 VM 迁移失败的问题
- [PR #8316][ShellyKa13] 修复删除未就绪 vmsnapshot 时可能出现的竞态，以及 VM 保持冻结的问题
- [PR #8436][xpivarc] KubeVirt 能够在启用 restricted Pod 安全标准的情况下运行，并自动提升命名空间特权。
- [PR #8197][alromeros] 为 virtctl 添加 vmexport 命令
- [PR #8252][fossedihelm] 向 KubeVirt 配置添加 `tlsConfiguration`
- [PR #8431][rmohr] 修复快照控制器对 VM 执行的影子状态更新与周期性状态更新
- [PR #8359][iholder-redhat] [缺陷修复]：当未暴露 TSC Frequency 时，HyperV Reenlightenment VMI 应能够启动
- [PR #8330][jean-edouard] 重要：如果在启用 SELinux 的情况下使用 docker，请在升级前设置 `DockerSELinuxMCSWorkaround` 功能门控
- [PR #8401][machadovilaca] 重命名指标以遵循命名约定

## v0.57.0 {#v0570}

发布于：Mon Sep 12 14:00:44 2022 +0000

- [PR #8129][mlhnono68] 修复 virtctl，以支持连接到由 RANCHER 代理或带有特殊路径的集群
- [PR #8337][0xFelix] virtctl 的原生 SSH 客户端现在可以在 Windows 控制台中使用，无需变通办法
- [PR #8257][awels] VirtualMachineExport 现在支持 VM 导出源类型。
- [PR #8367][vladikr] 修复客户机内存转换：当未显式提供客户机内存时，将其设置为 resources.requests.memory
- [PR #7990][ormergi] 弃用 SR-IOV 实时迁移功能门控。
- [PR #8069][lyarwood] VirtualMachineInstancePreset 资源已弃用，并将在未来版本中移除。用户应改用 VirtualMachineInstancetype 与 VirtualMachinePreference 资源，以封装其 VirtualMachine 所共享的资源或偏好特征。
- [PR #8326][0xFelix] virtctl：默认不记录被包装的 ssh 命令
- [PR #8325][rhrazdil] 在 virt-handler 中为 masquerade 绑定启用 route_localnet sysctl 选项
- [PR #8159][acardace] 添加对 USB 磁盘的支持
- [PR #8006][lyarwood] 已将 `AutoattachInputDevice` 添加到 `Devices`，从而允许在启动时自动把 `Input` 设备挂接到 `VirtualMachine`。此外，已将 `PreferredAutoattachInputDevice` 添加到 `DevicePreferences`，允许用户用一组偏好控制此行为。
- [PR #8134][arnongilboa] 支持 DataVolume 垃圾回收
- [PR #8157][StefanKro] TrilioVault for Kubernetes 现在支持借助 KubeVirt 进行备份与恢复。
- [PR #8273][alaypatel07] 在创建对象时为 spec.topologySpreadConstraints 添加服务端校验
- [PR #8049][alicefr] 将 guestfs Pod 的 RunAsNonRoot 设为默认值
- [PR #8107][awels] 允许将 VirtualMachineSnapshot 作为 VirtualMachineExport 源
- [PR #7846][janeczku] 添加了对为虚拟机配置拓扑分布约束的支持。
- [PR #8215][alaypatel07] 支持在创建 VMI 期间校验 spec.affinity 字段
- [PR #8071][oshoval] 放宽 networkInterfaceMultiqueue 的语义：多队列只会配置它能够配置的部分（virtio 接口）。
- [PR #7549][akrejcir] 新增用于展开 instancetype 与偏好的 API 子资源。

## v0.56.0 {#v0560}

发布于：Thu Aug 18 20:10:29 2022 +0000

- [PR #7599][iholder-redhat] 引入一种中止未在运行的迁移的机制，修复了“若 virt-launcher Pod 处于 pending 状态则无法取消实时迁移”的缺陷
- [PR #8027][alaypatel07] 在 perfscale 测试中，等待删除一直成功，直到对象被终结
- [PR #8198][rmohr] 改进非 root virt-launcher 工作负载的路径处理
- [PR #8136][iholder-redhat] 修复 cgroups 单元测试：模拟底层 runc cgroup 管理器
- [PR #8047][iholder-redhat] 弃用实时迁移功能门控
- [PR #7986][iholder-redhat] [缺陷修复]：带 WSL2 客户机的 Windows VM 迁移失败
- [PR #7814][machadovilaca] 添加 VMI 文件系统使用量指标
- [PR #7849][AlonaKaplan] [技术预览] 引入 passt，这是一种面向虚拟机的用户态网络新方法
- [PR #7991][ShellyKa13] virtctl 内存转储新增 create 标志，用于创建新的 PVC
- [PR #8039][lyarwood] flavor API 以及 `VirtualMachine{Flavor,ClusterFlavor}` 的相关 CRD 已重命名为 instancetype 与 `VirtualMachine{Instancetype,ClusterInstancetype}`。
- [PR #8112][AlonaKaplan] 将 `virtctl expose` 的 `ip-family` 参数默认值改为空值，而不再是 IPv4。
- [PR #8073][orenc1] 将 runc 升级到 v1.1.2
- [PR #8092][Barakmor1] 将 emicklei/go-restful 的版本从 2.15.0 升级到 2.16.0
- [PR #8053][alromeros] [缺陷修复]：修复当 CDI 资源名称不同时获取 fs 开销的机制
- [PR #8035][0xFelix] 为 scp 命令添加包装本地 scp 客户端的选项
- [PR #7981][lyarwood] 使用 flavor 时，如果 `VirtualMachine` 定义了任何 `CPU` 或 `Memory` 资源请求，现在会引发冲突。
- [PR #8068][awels] 将热插拔磁盘的缓存模式设置为与普通磁盘一致。

## v0.55.0 {#v0550}

发布于：Thu Jul 14 16:33:25 2022 +0000

- [PR #7336][iholder-redhat] 引入克隆 CRD、控制器与 API
- [PR #7791][iholder-redhat] 引入初步的弃用策略
- [PR #7875][lyarwood] `ControllerRevisions`（对应于任意 `VirtualMachineFlavorSpec` 或 `VirtualMachinePreferenceSpec`）会在 `VirtualMachine` 初次启动期间被存储，并用于后续重启，以确保对原始 `VirtualMachineFlavor` 或 `VirtualMachinePreference` 的更改不会修改 `VirtualMachine` 及其创建的 `VirtualMachineInstance`。
- [PR #8011][fossedihelm] 增加 virt-launcher 的内存开销
- [PR #7963][qinqon] 升级 alpine_with_test_tooling
- [PR #7881][ShellyKa13] 允许将内存转储包含在 VMSnapshot 中
- [PR #7926][qinqon] 测试：将主清理函数移至全局 AfterEach，并为 infra_test.go 的每个 Entry 创建一个 VM。
- [PR #7845][janeczku] 修复了当 API 代码注释包含反引号时导致 `make generate` 失败的缺陷。(#7844, @janeczku)
- [PR #7932][marceloamaral] 新增 kubevirt_vmi_migration_phase_transition_time_from_creation_seconds 指标，用于监控 VMI Migration 对象从创建时起转换到特定阶段所需的时间。
- [PR #7879][marceloamaral] 得益于 virt-controller QPS/Burst 的提高，VM 阶段转换更快
- [PR #7807][acardace] 使 cloud-init 的 'instance-id' 在多次重启之间保持不变
- [PR #7928][iholder-redhat] 缺陷修复：node-labeller 现在会移除 "host-model-cpu.node.kubevirt.io/" 与 "host-model-required-features.node.kubevirt.io/" 前缀
- [PR #7841][jean-edouard] 集群禁用非 root 之后，非 root VM 现在会迁移为 root VM。
- [PR #7933][akalenyu] 缺陷修复：修复恢复大小大于 PVC 请求大小时的 VM 恢复
- [PR #7919][lyarwood] 设备偏好现在会应用于运行时添加到 `VirtualMachineInstance` 的任何默认网络接口或缺失的卷磁盘。
- [PR #7910][qinqon] 测试：创建预期的就绪探针，而不是存活探针
- [PR #7732][acardace] 防止 virt-handler 将同一次迁移启动两次
- [PR #7594][alicefr] 允许 libguestfs-tools Pod 以非 root 用户运行
- [PR #7811][raspbeep] 用户现在可以获得客户机代理所不支持的命令类型信息。
- [PR #7590][awels] VMExport 允许将文件系统 PVC 导出为磁盘或目录。
- [PR #7683][alicefr] 为 virtctl ssh 添加 --command 与 --local-ssh-opts 选项，以便使用本地 ssh 方式执行远程命令

## v0.54.0 {#v0540}

发布于：Wed Jun 8 14:15:43 2022 +0000

- [PR #7757][orenc1] 新增告警，用于提示一段时间内 VMI 迁移次数过多。
- [PR #7517][ShellyKa13] 添加 virtctl 内存转储命令
- [PR #7801][VirrageS] XML 中为空（`nil` 值）的 `Address` 与 `Driver` 字段将被省略。
- [PR #7475][raspbeep] 当设置了 EvictionStrategy 但由于其限制而无法进行实时迁移时，将实时迁移失败原因添加到所记录的事件中。
- [PR #7739][fossedihelm] 允许 admin/edit 用户使用 `virtualmachines/migrate` 子资源
- [PR #7618][lyarwood] 定义 `Disk` 或 `Filesystem` 以对应每个 `Volume`（关联于 `VirtualMachine`）的要求已被移除。任何 `Volumes` 若未定义 `Disk` 或 `Filesystem`，则会在运行时把 `Disk` 定义到 `VirtualMachineInstance` 中。
- [PR #7529][xpivarc] NoReadyVirtController 与 NoReadyVirtOperator 应当被正确触发。
- [PR #7465][machadovilaca] 添加迁移及其相应阶段的指标
- [PR #7592][akalenyu] 缺陷修复：virtctl guestfs 错误地假定了镜像名称

## v0.53.1 {#v0531}

发布于：Tue May 17 14:55:54 2022 +0000

- [PR #7749][kubevirt-bot] NoReadyVirtController 与 NoReadyVirtOperator 应当被正确触发。

## v0.53.0 {#v0530}

发布于：Mon May 9 14:02:20 2022 +0000

- [PR #7533][akalenyu] 新增多项 VM 快照指标
- [PR #7574][rmohr] 引入传递依赖已最小化的 CDI 依赖，以方便采用 API
- [PR #7318][iholder-redhat] 快照恢复现在支持恢复到与源不同的目标 VM
- [PR #7474][borod108] 为实时迁移添加了以下指标：kubevirt_migrate_vmi_data_processed_bytes、kubevirt_migrate_vmi_data_remaining_bytes、kubevirt_migrate_vmi_dirty_memory_rate_bytes
- [PR #7441][rmohr] 添加 `virtctl scp`，以便更方便地在 VM 与 VMI 之间复制文件
- [PR #7265][rthallisey] 在 load-generator 工具中支持稳态作业类型
- [PR #7544][fossedihelm] 将 go 版本升级到 1.17.8
- [PR #7582][acardace] 修复实际已成功的迁移被报告为失败的问题。
- [PR #7546][0xFelix] 将 virtio-container-disk 更新到 virtio-win 版本 0.1.217-1
- [PR #7530][iholder-redhat] [外部内核启动]：在未提供自定义内核时不允许使用内核参数
- [PR #7493][davidvossel] 新增 EvictionStrategy "External"，用于由外部控制器处理的阻止驱逐
- [PR #7563][akalenyu] 将 VolumeSnapshot 切换到 v1
- [PR #7406][acardace] 拒绝将 `LiveMigrate` 作为工作负载更新策略，如果未启用 `LiveMigration` 功能门控。
- [PR #7103][jean-edouard] 现已支持非持久 vTPM。请注意，每次关闭后 TPM 状态都会被清除。请勿启用 Bitlocker！
- [PR #7277][andreabolognani] 此版本的 KubeVirt 包含基于 libvirt 8.0.0 与 QEMU 6.2.0 升级的虚拟化技术。
- [PR #7130][Barakmor1] 向 kubevirtCR 添加字段，用于设置 Prometheus ServiceMonitor 对象的命名空间
- [PR #7401][iholder-redhat] virt-api Deployment 现在可扩展，副本数由集群中的节点数量决定
- [PR #7500][awels] 缺陷修复：修复了 admin/edit 用户的 RBAC，以允许 virtualmachine/addvolume 与 removevolume。这使得持久磁盘成为可能
- [PR #7328][apoorvajagtap] 在 `virtctl ssh` 上设置 --local-ssh=true 时，不要忽略 --identity-file
- [PR #7469][xpivarc] 用户现在可以启用 NonRoot 功能门控，以代替 NonRootExperimental
- [PR #7451][fossedihelm] 通过拆分监控进程与 launcher 进程来降低 virt-launcher 的内存用量

## v0.52.0 {#v0520}

发布于：Fri Apr 8 16:17:56 2022 +0000

- [PR #7024][fossedihelm] 如果客户端与服务端的 virtctl 版本不一致，则添加一条警告信息
- [PR #7486][rmohr] 将 stable.txt 移动到更合适的路径
- [PR #7372][saschagrunert] 修复了 `KubeVirtComponentExceedsRequestedMemory` 告警关于不允许多对多匹配的报错。
- [PR #7426][iholder-redhat] 在 KubeVirt CR 中手动确定核心组件副本数时添加警告
- [PR #7424][maiqueb] 提供接口绑定类型的描述，这些描述将出现在 KubeVirt API 中。
- [PR #7422][orelmisan] 修复了为一系列 SR-IOV NIC 设置自定义客户机 pciAddress 与 bootOrder 参数的问题。
- [PR #7421][rmohr] 修复 virtctl ssh 的 knowhosts 文件损坏问题
- [PR #6854][rmohr] 使 virtctl ssh 能够配合 ssh-rsa+ 预认证工作
- [PR #7267][iholder-redhat] 现在可以在 VMI 的 status 中找到已应用的迁移配置
- [PR #7321][iholder-redhat] [迁移策略]：VMI 标签优先于 Namespace 标签
- [PR #7326][oshoval] Ginkgo 依赖已升级到 v2.1.3（主版本升级）
- [PR #7361][SeanKnight] 修复了阻止 virtctl 用于通过 Rancher 认证代理访问的集群，或服务器 URL 包含路径分量的任何其他集群的缺陷。(#3760)
- [PR #7255][tyleraharrison] 用户现在能够指定 `--address [ip_address]`，以便在使用 `virtctl vnc` 时不再只能使用 127.0.0.1
- [PR #7275][enp0s3] 为 virt-operator 添加 observedGeneration，以便以无竞态的方式检测 KubeVirt 配置上线
- [PR #7233][xpivarc] 缺陷修复：成功中止的迁移现在应当被报告
- [PR #7158][AlonaKaplan] 为单栈 IPv6 添加 masquerade VM 支持。
- [PR #7227][rmohr] 从 virt-api 中移除 VMI Informer，以改善 virt-api 的扩展特性
- [PR #7288][raspbeep] 用户现在无需为 `kubectl logs <vmi-pod>` 和 `kubectl exec <vmi-pod>` 指定容器。
- [PR #6709][xpivarc] 如果设置了 NonRoot 功能门控，工作负载将迁移到非 root 实现。（VirtioFS 除外）
- [PR #7241][lyarwood] 修复了无法仅为 sysprep 磁盘提供 unattend.xml configmap 或 secret 作为内容的缺陷。(#7240, @lyarwood)

## v0.51.0 {#v0510}

发布于：Tue Mar 8 21:06:59 2022 +0000

- [PR #7102][machadovilaca] 为 virt-launcher Pod 添加虚拟机名称标签
- [PR #7139][davidvossel] 通过使用控制器修订版本，修复 VirtualMachinePool 对 VM/VMI 不一致的更新
- [PR #6754][jean-edouard] 新建与调整大小后的磁盘现在始终按 1MiB 对齐
- [PR #7086][acardace] 在 KubeVirt CR 中添加作为集群范围设置的 'EvictionStrategy'
- [PR #7232][rmohr] 在迁移期间正确格式化 PDB scale 事件
- [PR #7223][Barakmor1] 为 virt-operator Pod 添加名称标签
- [PR #7221][davidvossel] RunStrategy: Once - 允许声明 VM 应运行一次直至到达终态
- [PR #7091][EdDev] 即使没有活动的客户机代理，SR-IOV 接口现在也会在 VMI status 中报告。
- [PR #7169][rmohr] 改进 virt-handler 中的设备插件注销，并提升部分测试的稳定性
- [PR #6604][alicefr] 添加 shareable 选项，用于标识磁盘是否与其他 VM 共享
- [PR #7144][davidvossel] 垃圾回收已终结的迁移对象，仅保留最近的 5 个对象
- [PR #6110][xpivarc] [非 root] SRIOV 现已可用。

## v0.50.0 {#v0500}

发布于：Wed Feb 9 18:01:08 2022 +0000

- [PR #7056][fossedihelm] 将 k8s 依赖更新到 0.23.1
- [PR #7135][davidvossel] 在整个项目中从 reflects.DeepEquals 切换到 equality.Semantic.DeepEquals()
- [PR #7052][sradco] 更新了记录规则 "kubevirt_vm_container_free_memory_bytes"
- [PR #7000][iholder-redhat] 增加了通过 VMI 注解覆盖默认 libvirt 日志过滤器的能力
- [PR #7064][davidvossel] 修复了移除期间仍存在 VMI 时卸载受阻的相关问题
- [PR #7097][iholder-redhat] [缺陷修复] 如果定义了更多磁盘，带内核启动的 VMI 会卡在 "Terminating" 状态
- [PR #6700][VirrageS] 简化在代理轮询器中替换 `time.Ticker` 的做法，并修复 `qemu-*-interval` 标志的默认值
- [PR #6581][ormergi] 当 SRIOV 网络接口被手动断开或因迁移中止而断开时，现在会对其进行热插拔。
- [PR #6924][EdDev] 已移除对旧式 GPU 定义的支持。请参阅 https://kubevirt.io/user-guide/virtual_machines/host-devices 了解如何定义主机设备。
- [PR #6735][uril] 已向 virtctl 添加命令 `migrate_cancel`。它会取消正在进行的 VM 迁移。
- [PR #6883][rthallisey] 向 cloud-init 元数据添加 instance-type
- [PR #6999][maya-r] 扩展磁盘镜像时，取请求与容量之间的较小值，避免在 NFS、local 等存储上用尽底层文件系统。
- [PR #6946][vladikr] 已分配设备的 NUMA 信息将呈现在设备元数据中
- [PR #6042][iholder-redhat] 完整支持 cgroups v2，包含一个新的内聚软件包，并进行了重大重构。
- [PR #6968][vladikr] 添加了 Writeback 磁盘缓存支持
- [PR #6995][sradco] 告警名称 OrphanedVirtualMachineImages 已更改为 OrphanedVirtualMachineInstances。
- [PR #6923][rhrazdil] 修复带有 Istio 代理的 VMI 上 ssh 不可达的问题
- [PR #6821][jean-edouard] 包含专用 CPU 的 VMI 在迁移后，将在目标上拥有正确的专用 CPU
- [PR #6793][oshoval] 向 vmi.status.interfaces 添加 infoSource 字段。

## v0.49.0 {#v0490}

发布于：Tue Jan 11 17:27:09 2022 +0000

- [PR #7004][iholder-redhat] 缺陷修复：避免对只读磁盘所使用的卷设置块迁移
- [PR #6959][enp0s3] 当目标 Pod 进入 unschedulable 阶段时生成事件
- [PR #6888][assafad] 向告警定义中添加了通用标签
- [PR #6166][vasiliy-ul] 实验性支持 AMD SEV
- [PR #6980][vasiliy-ul] 更新依赖以包含 CVE-2021-43565 的修复（KubeVirt 不受影响）
- [PR #6944][iholder-redhat] 从实时迁移策略中移除禁用 TLS 的配置
- [PR #6800][jean-edouard] CPU 绑核不再需要硬件辅助虚拟化
- [PR #6501][ShellyKa13] 使用 virtctl image-upload 上传归档内容
- [PR #6918][iholder-redhat] 缺陷修复：不可调度的 host-model VMI 告警现在会正确触发
- [PR #6796][Barakmor1] 由 virt-operator 创建的 kubevirt 组件上，'managed-by' 标签中的 'kubevirt-operator' 已改为 'virt-operator'
- [PR #6036][jean-edouard] 现在可以通过专用 Multus 网络进行迁移
- [PR #6933][erkanerol] 为监控测试新增一条泳道
- [PR #6949][jean-edouard] 即使 virt-api 与 virt-controller 仅使用 1 个副本，删除 CR 时现在也应能成功移除 KubeVirt 组件
- [PR #6954][maiqueb] 将 `virtctl` 所暴露服务的 `IPFamilyPolicyType` 默认值更新为 `IPFamilyPolicyPreferDualStack`
- [PR #6931][fossedihelm] 为 AddVolumeOptions 与 RemoveVolumeOptions 添加了 DryRun
- [PR #6379][nunnatsa] 修复问题 https://bugzilla.redhat.com/show_bug.cgi?id=1945593
- [PR #6399][iholder-redhat] 引入实时迁移策略，使系统管理员能够对不同 VM 集合的迁移配置进行细粒度控制。
- [PR #6880][iholder-redhat] 为 `make` 与 `make test` 添加完整的 Podman 支持
- [PR #6702][acardace] 实现 virt-handler 金丝雀升级与回滚，以便更快、更安全地上线
- [PR #6717][davidvossel] 引入 VirtualMachinePools 功能，用于大规模管理有状态 VM
- [PR #6698][rthallisey] 为 virt-controller 工作队列添加追踪
- [PR #6762][fossedihelm] 为 virtcl 的 migrate 命令添加了 DryRun 模式
- [PR #6891][rmohr] 修复未在 pty 中执行 "virtctl console" 时出现的 "Make raw terminal failed: The handle is invalid?" 问题
- [PR #6783][rmohr] 如果未显式提供 RSA 密钥，且默认位置不存在密钥，则跳过 SSH RSA 认证

## v0.48.1 {#v0481}

发布于：Wed Dec 15 15:11:55 2021 +0000

- [PR #6900][kubevirt-bot] 如果未显式提供 RSA 密钥，且默认位置不存在密钥，则跳过 SSH RSA 认证
- [PR #6902][kubevirt-bot] 修复未在 pty 中执行 "virtctl console" 时出现的 "Make raw terminal failed: The handle is invalid?" 问题

## v0.48.0 {#v0480}

发布于：Mon Dec 6 18:26:51 2021 +0000

- [PR #6670][futuretea] 添加了 'virtctl soft-reboot' 命令，用于重启 VMI。
- [PR #6861][orelmisan] virtctl 的错误写入 stderr，而不是 stdout
- [PR #6836][enp0s3] 为 'kubectl get vmim' 的 CLI 输出添加了 PHASE 与 VMI 列
- [PR #6784][nunnatsa] 不再支持使用 kubevirt-config ConfigMap 进行 KubeVirt 配置
- [PR #6839][ShellyKa13] 修复带有 RunStrategy 的 VM 恢复
- [PR #6533][zcahana] 已暂停的 VMI 现在会被标记为未就绪，即使未指定 readinessProbe
- [PR #6858][rmohr] 修复 virtctl 与某些外部认证插件结合使用时的空指针问题
- [PR #6780][fossedihelm] 为 VirtualMachineInstanceInterface 的 Patch 请求添加 PatchOptions
- [PR #6773][iholder-redhat] 当使用 host-model CPU 的 VMI 因没有合适节点而导致迁移卡住时发出告警
- [PR #6714][rhrazdil] 缩短 Istio 代理检测超时
- [PR #6725][fossedihelm] 为 virtcl 的 pause 与 unpause 命令添加了 DryRun 模式
- [PR #6737][davidvossel] 待定的迁移目标 Pod 在不可调度时会在 5 分钟后超时
- [PR #6814][fossedihelm] 更改了部分术语，使其更加包容
- [PR #6649][Barakmor1] 将 apps.kubevirt.io/component 标签指定给 KubeVirt 组件。
- [PR #6650][victortoso] 引入对 ich9 或 ac97 声音设备的支持
- [PR #6734][Barakmor1] 替换用于提取 libvirtd pid 的命令，以避免以下错误：
- [PR #6802][rmohr] 维护一个同步到 kubevirt.io/api 的独立 api 包，以便借助 client-gen 更好地进行第三方集成
- [PR #6730][zhhray] 将 kubevrit 证书 Secret 类型从 Opaque 改为 kubernetes.io/tls
- [PR #6508][oshoval] 在使用子域时，向客户机搜索列表补充缺失的域。
- [PR #6664][vladikr] 默认启用 vGPU 的 display 与 ramfb
- [PR #6710][iholder-redhat] virt-launcher 修复：在关闭并未真正成功时，停止记录成功关闭
- [PR #6162][vladikr] 请求 dedicatedCpusPlacement 时将始终设置 KVM_HINTS_REALTIME
- [PR #6772][zcahana] 缺陷修复：回退 #6565，该更改阻止了升级到 v0.47。
- [PR #6722][zcahana] 移除已过时的 scheduler.alpha.kubernetes.io/critical-pod 注解
- [PR #6723][acardace] 移除由 < 0.41.1 的 virt-controller 创建的过期 PDB
- [PR #6721][iholder-redhat] 即使 KubevirtCR 中未定义，也在 VMI spec 中设置默认 CPU 模型
- [PR #6713][zcahana] 当 PVC/DV 类型的卷引用未绑定的 PVC 时，报告 WaitingForVolumeBinding VM 状态
- [PR #6681][fossedihelm] 用户可以使用 --dry-run 标志
- [PR #6663][jean-edouard] virt-api 与 virt-controller 的副本数现在可在 CSV 中配置
- [PR #5981][maya-r] 始终在启动时将 disk.img 文件调整到最大尺寸。

## v0.47.1 {#v0471}

发布于：Thu Nov 11 15:52:59 2021 +0000

- [PR #6775][kubevirt-bot] 缺陷修复：回退 #6565，该更改阻止了升级到 v0.47。
- [PR #6703][mhenriks] 修复 BZ 2018521：升级时 VirtualMachineSnapshot 变为 Failed
- [PR #6511][knopt] 修复在大型 KubeVirt 部署中使用 Cluster Profiler 时 virt-api 内存占用显著的问题。(#6478, @knopt)
- [PR #6629][awels] 缺陷修复：热插拔多个块设备会导致 IO 错误（#6564）
- [PR #6657][andreabolognani] 此版本的 KubeVirt 包含基于 libvirt 7.6.0 与 QEMU 6.0.0 升级的虚拟化技术。
- [PR #6565][Barakmor1] 由 virt-operator 创建的 kubevirt 组件上，'managed-by' 标签中的 'kubevirt-operator' 已改为 'virt-operator'
- [PR #6642][ShellyKa13] 在在线 VM 快照中包含热插拔磁盘
- [PR #6513][brybacki] 为 virtctl imageupload 添加 force-bind 标志
- [PR #6588][erkanerol] 修复基于 up 指标的记录规则
- [PR #6575][davidvossel] VM 控制器现在会将 VMI 状况同步到对应的 VM 对象
- [PR #6661][rmohr] 使 kubevirt API 与 client-gen 兼容，以便更容易选择兼容的 k8s golang 依赖
- [PR #6535][rmohr] 迁移使用摘要来引用 containerDisk 与内核启动镜像，以确保磁盘一致性
- [PR #6651][ormergi] KubeVirt Conformance 插件现在支持传入测试镜像仓库。
- [PR #6589][iholder-redhat] 用于启动的自定义内核 / initrd 现在会被预先拉取，从而提高稳定性
- [PR #6199][ormergi] KubeVirt Conformance 插件现在支持传入镜像标签或摘要
- [PR #6477][zcahana] 当所引用的 DataVolume 指示错误时，报告 DataVolumeError VM 状态
- [PR #6593][rhrazdil] 从 virt-launcher 与 virt-handler 容器中移除了 python 依赖
- [PR #6026][akrejcir] 实现了最基本的 VirtualMachineFlavor 功能。
- [PR #6570][erkanerol] 对指标上的 namespace 标签使用 honorLabels，而不再使用 labelDrop
- [PR #6182][jordigilh] 添加对实时工作负载的支持
- [PR #6177][rmohr] 将节点基础镜像切换为 centos8 stream
- [PR #6171][zcahana] 当 PVC/DV 类型的卷引用不存在的对象时，报告 ErrorPvcNotFound/ErrorDataVolumeNotFound VM 状态
- [PR #6437][VirrageS] 修复对 watch API 的已弃用用法，以防止报告不正确的指标。
- [PR #6482][jean-edouard] 带有 cloud-init 数据的 VM 现在应当能够正确地从较旧的 KubeVirt 版本迁移
- [PR #6375][dhiller] 测试期间依赖 kubevirtci 安装 CDI

## v0.46.1 {#v0461}

发布于：Tue Oct 19 15:41:10 2021 +0000

- [PR #6557][jean-edouard] 带有 cloud-init 数据的 VM 现在应当能够正确地从较旧的 KubeVirt 版本迁移

## v0.46.0 {#v0460}

发布于：Fri Oct 8 21:12:33 2021 +0000

- [PR #6425][awels] 启用 iothreads 后可以对磁盘进行热插拔。
- [PR #6297][acardace] 变更迁移 PDB，而不是在迁移期间额外创建一个。
- [PR #6464][awels] 缺陷修复：修复 virt-launcher 意外退出时 kubelet 与 virt-handler 之间的热插拔竞态。
- [PR #6465][salanki] 修复来自本地 DHCP 服务器的损坏 DHCP Gateway 选项，该问题会导致 Windows VM 拒绝 IP 配置。
- [PR #6458][vladikr] 带标签的 SR-IOV 接口现在会出现在 config drive 元数据中
- [PR #6446][brybacki] virtctl 镜像上传的访问模式现在是可选的。此版本的 virtctl 现在要求 CDI v1.34 或更高版本
- [PR #6391][zcahana] 清理 virt-operator ClusterRole 中已过时的权限
- [PR #6419][rthallisey] 修复由大量已删除 VMI 事件引起的 virt-controller panic
- [PR #5972][kwiesmueller] 添加 `ssh` 命令到 `virtctl`，可用于打开到 VM/VMI 的 SSH 会话。
- [PR #6403][jrife] 移除了将 Go 模块锁定到 github.com/go-kit/kit 旧版本 (v0.3.0) 的做法
- [PR #6367][brybacki] virtctl imageupload 现在使用 DataVolume.spec.storage
- [PR #6198][iholder-redhat] 当在 virt-api 中检测到大量 REST 失败时触发 Prometheus 告警
- [PR #6211][davidvossel] cluster-profiler pprof 采集工具以及相应的 "ClusterProfiler" 功能门控
- [PR #6323][vladikr] 将实时迁移切换为使用 unix 套接字
- [PR #6374][vladikr] 修复 vmipod 上 CPU requests 的默认设置
- [PR #6283][rthallisey] 记录删除 VMI 所需的时间，并将其作为指标公开
- [PR #6251][rmohr] 更好地将 vcpu 线程放置到宿主机 CPU 上，以形成更高效的直通架构
- [PR #6377][rmohr] 如果 SELinux 处于 permissive 模式，则不要因 SELinux 重新标记失败而使操作失败
- [PR #6308][awels] 缺陷修复：与位于独立设备上的 hostpath 卷一起使用时，热插拔无法工作。
- [PR #6186][davidvossel] 为 rest_client_requests_total 指标添加 resource 与 verb 标签

## v0.45.1 {#v0451}

发布于：Tue Oct 19 15:39:42 2021 +0000

- [PR #6537][kubevirt-bot] 修复来自本地 DHCP 服务器的损坏 DHCP Gateway 选项，该问题会导致 Windows VM 拒绝 IP 配置。
- [PR #6556][jean-edouard] 带有 cloud-init 数据的 VM 现在应当能够正确地从较旧的 KubeVirt 版本迁移
- [PR #6480][kubevirt-bot] 缺陷修复：修复 virt-launcher 意外退出时 kubelet 与 virt-handler 之间的热插拔竞态。
- [PR #6384][kubevirt-bot] 更好地将 vcpu 线程放置到宿主机 CPU 上，以形成更高效的直通架构

## v0.45.0 {#v0450}

发布于：Wed Sep 8 13:56:47 2021 +0000

- [PR #6191][marceloamaral] 新增 perfscale-load-generator，用于执行压力测试以评估控制平面
- [PR #6248][VirrageS] 减少热路径中的日志记录
- [PR #6079][weihanglo] 热插拔卷可以随时拔出，并在 VM 重启后重新挂接。
- [PR #6101][rmohr] 使 k8s 客户端速率限制可配置
- [PR #6204][sradco] 此 PR 为每条告警添加了运行手册 URL，指向提供该告警更多细节及缓解方法的运行手册。
- [PR #5974][vladikr] 现在可以在 KubeVirt CR 中向 kubevirt 提供期望的 mdev 类型列表，以便在相关节点上配置这些设备
- [PR #6147][rmohr] 修复 freeze/unfreeze、addvolume/removevolume、guestosinfo、filesystemlist 与 userlist 的 RBAC 权限
- [PR #6161][ashleyschuett] 移除创建 VMI 时的 HostDevice 校验
- [PR #6078][zcahana] 发生镜像拉取错误时，报告 ErrImagePull/ImagePullBackOff VM 状态
- [PR #6176][kwiesmueller] 修复 virt-handler 中的 goroutine 泄漏，该问题在 VMI 高周转时可能导致故障。
- [PR #6047][ShellyKa13] 为 VM 快照 API 添加阶段，特别是失败阶段
- [PR #6138][ansijain] 不适用

## v0.44.3 {#v0443}

发布于：Tue Oct 19 15:38:22 2021 +0000

- [PR #6518][jean-edouard] 带有 cloud-init 数据的 VM 现在应当能够正确地从较旧的 KubeVirt 版本迁移
- [PR #6532][kubevirt-bot] 变更迁移 PDB，而不是在迁移期间额外创建一个。
- [PR #6536][kubevirt-bot] 修复来自本地 DHCP 服务器的损坏 DHCP Gateway 选项，该问题会导致 Windows VM 拒绝 IP 配置。

## v0.44.2 {#v0442}

发布于：Thu Oct 7 12:55:34 2021 +0000

- [PR #6479][kubevirt-bot] 缺陷修复：修复 virt-launcher 意外退出时 kubelet 与 virt-handler 之间的热插拔竞态。
- [PR #6392][rmohr] 更好地将 vcpu 线程放置到宿主机 CPU 上，以形成更高效的直通架构
- [PR #6251][rmohr] 更好地将 vcpu 线程放置到宿主机 CPU 上，以形成更高效的直通架构
- [PR #6344][kubevirt-bot] 缺陷修复：与位于独立设备上的 hostpath 卷一起使用时，热插拔无法工作。
- [PR #6263][rmohr] 使 k8s 客户端速率限制可配置
- [PR #6207][kubevirt-bot] 修复 virt-handler 中的 goroutine 泄漏，该问题在 VMI 高周转时可能导致故障。
- [PR #6101][rmohr] 使 k8s 客户端速率限制可配置
- [PR #6249][kubevirt-bot] 修复 freeze/unfreeze、addvolume/removevolume、guestosinfo、filesystemlist 与 userlist 的 RBAC 权限

## v0.44.1 {#v0441}

发布于：Thu Aug 12 12:28:02 2021 +0000

- [PR #6219][kubevirt-bot] 为 VM 快照 API 添加阶段，特别是失败阶段

## v0.44.0 {#v0440}

发布于：Mon Aug 9 14:20:14 2021 +0000

- [PR #6058][acardace] 修复 virt-launcher 退出 Pod 的竞态条件
- [PR #6035][davidvossel] 新增 perfscale-audit 工具，用于在压力测试期间审计控制平面性能
- [PR #6145][acardace] virt-launcher：禁用 libvirtd 的未加密 TCP 套接字。
- [PR #6163][davidvossel] 处理处于 defunc（僵尸）状态的 qemu 进程
- [PR #6105][ashleyschuett] 在 KubeVirt CR 的 Spec.Configuration 下添加 VirtualMachineInstancesPerNode
- [PR #6104][zcahana] 发生调度错误时报告 FailedUnschedulable VM 状态
- [PR #5905][davidvossel] VM CrashLoop 检测与指数退避
- [PR #6070][acardace] 使用由 virt-handler 暴露的 unix 套接字发起实时迁移，而不再使用由 virt-launcher 启动的额外 TCP<->Unix 迁移代理
- [PR #5728][vasiliy-ul] 现已启用带有热插拔卷的 VM 的实时迁移
- [PR #6109][rmohr] 修复 virt-controller SCC：在 virt-controller SCC 中体现对 NET_BIND_SERVICE 的需求。
- [PR #5942][ShellyKa13] 将客户机代理集成到在线 VM 快照
- [PR #6034][ashleyschuett] Go 版本已更新到 1.16.6
- [PR #6040][yuhaohaoyu] 通过保留失败 VMI 的环境来提高可调试性。
- [PR #6068][dhiller] 添加检查，确保并非所有测试都被跳过
- [PR #6041][xpivarc] [实验性] virt-launcher 可以以非 root 用户运行
- [PR #6062][iholder-redhat] 用新的、仍在维护的 "stress-ng" 二进制文件替换已废弃的 "stress" 二进制文件
- [PR #6029][mhenriks] 将 CDI 升级到 1.36.0，并支持 DataSource
- [PR #4089][victortoso] 添加通过 usbredir 进行 USB 重定向的支持
- [PR #5946][vatsalparekh] 添加基于客户机代理的 ping 探针
- [PR #6005][acardace] 使 containerDisk 校验的内存用量限制可配置
- [PR #5791][zcahana] 为 "kubectl get vm/vmi" 的表格输出添加了 READY 列
- [PR #6006][awels] 由 DataVolumeTemplates 创建的 DataVolume 将遵循关联 VM 的优先级类。
- [PR #5982][davidvossel] 减少启动期间的 VMI Update 冲突（HTTP 代码 409）
- [PR #5891][akalenyu] 缺陷修复：创建由 WFFC DV 支持的并发批量 VM 时，VMI 停留在 Pending
- [PR #5925][rhrazdil] 修复 Windows VM 未被分配 network-attachment-definition IPAM 中所配置 IP 地址的问题。
- [PR #6007][rmohr] 修复：迁移上的带宽限制不再被忽略。注意：默认带宽限制已从 64Mi 更改为 "unlimited"，以免破坏现有安装。
- [PR #4944][kwiesmueller] 添加 `/portforward` 子资源到 `VirtualMachine` 与 `VirtualMachineInstance`，可通过 WebSocket 流经由 API Server 隧道传输 TCP 流量。
- [PR #5402][alicefr] 集成 libguestfs-tools，并向 virtctl 添加了新命令 `guestfs`
- [PR #5953][ashleyschuett] 允许在使用 `--force --gracePeriod 0` 时停止处于 Failed 状态的 VM
- [PR #5876][mlsorensen] KubeVirt CR 支持通过 'launcherRuntimeClass' 为 virt-launcher Pod 指定运行时类。

## v0.43.1 {#v0431}

发布于：Tue Oct 19 15:36:32 2021 +0000

- [PR #6555][jean-edouard] 带有 cloud-init 数据的 VM 现在应当能够正确地从较旧的 KubeVirt 版本迁移
- [PR #6052][kubevirt-bot] 使 containerDisk 校验的内存用量限制可配置

## v0.43.0 {#v0430}

发布于：Fri Jul 9 15:46:22 2021 +0000

- [PR #5952][mhenriks] 使用 CDI beta API。CDI v1.20.0 现在是 kubevirt 的最低要求。
- [PR #5846][rmohr] 添加 "spec.cpu.numaTopologyPassthrough"，允许为高性能模拟与宿主机对齐的虚拟 NUMA 拓扑
- [PR #5894][rmohr] 向 KubeVirt CR 添加 `spec.migrations.disableTLS`，以允许禁用加密迁移。迁移默认仍然安全。
- [PR #5649][awels] 增强：移除每块磁盘一个挂接 Pod 的限制（升级时若存在正在运行且带有热插拔磁盘的 VM，行为未定义）
- [PR #5742][rmohr] 选择 evictionStrategy `LifeMigrate` 并请求 `invtsc` cpuflag 的 VMI 现在可以实时迁移
- [PR #5911][dhiller] 升级 kubevirtci，并抑制 kubectl.sh 的输出，以避免造成混淆的检查
- [PR #5863][xpivarc] 修复：ioerror 不会再导致 notify 服务器崩溃循环
- [PR #5867][mlsorensen] 新增构建目标，用于将 virt-* 镜像导出为 tar 归档。
- [PR #5766][davidvossel] 新增 kubevirt_vmi_phase_transition_seconds_since_creation，用于监控 VMI 从创建时起转换到特定阶段所需的时间。
- [PR #5823][dhiller] 将默认分支更改为 `main`，适用于 `kubevirt/kubevirt` 仓库
- [PR #5763][nunnatsa] 修复缺陷 1945589：阻止使用 virtiofs 的 VMI 进行迁移
- [PR #5827][mlsorensen] 空 PVC 上自动置备的磁盘镜像现在会留出 128KiB 未使用空间，以避免使卷耗尽空间的边界情况。
- [PR #5849][davidvossel] 修复事件记录导致 virt-controller 段错误的问题
- [PR #5797][rhrazdil] 当 VMI 注解中启用 Istio 时自动添加 serviceAccountDisk
- [PR #5723][ashleyschuett] 允许 virtctl 停止 VM 并忽略优雅关闭期限
- [PR #5806][mlsorensen] 当底层节点存储的块大小为 4k 时，configmap、secret 与 cloud-init 原始磁盘现在可以正常工作。
- [PR #5623][iholder-redhat] [缺陷修复]：使用 host-model CPU 的 VM 现在仅允许迁移到兼容节点
- [PR #5716][rhrazdil] 修复使用 Istio 时 virt-launcher 在迁移后变为 `NotReady` 的问题。
- [PR #5778][ashleyschuett] 如果 ca-bundle 无法被解析，则更新它
- [PR #5787][acardace] 已将 authorization/v1beta1 的引用迁移到 authorization/v1
- [PR #5461][rhrazdil] 当 masquerade 接口未显式指定端口时，添加对 Istio 代理的支持
- [PR #5751][acardace] 即使 virt-launcher 镜像中仅存在 OVMF_CODE.secboot.fd 与 OVMF_VARS.fd，禁用 secureboot 的 EFI VMI 现在也可以启动
- [PR #5629][andreyod] 支持使用 `virtctl start --paused` 在客户机 CPU 处于暂停状态时启动虚拟机
- [PR #5725][dhiller] 在功能测试之后生成 REST API 覆盖率报告
- [PR #5758][davidvossel] 修复 kubevirt_vmi_phase_count，使其包含所有阶段，包括 handler 交接之前出现的阶段。
- [PR #5745][ashleyschuett] 当资源用量超过资源请求时发出告警
- [PR #5759][mhenriks] 将 CDI 更新到 1.34.1
- [PR #5038][kwiesmueller] 为通过 qemu-guest-agent 执行的 VM 存活探针与 readinessProbe 添加 exec 命令。
- [PR #5431][alonSadan] 添加 NFT 与 IPTables 规则，以允许端口转发到 VMI 上未声明的端口。在 VMI 上声明端口将会限制

## v0.42.2 {#v0422}

发布于：Tue Oct 19 15:34:37 2021 +0000

- [PR #6554][jean-edouard] 带有 cloud-init 数据的 VM 现在应当能够正确地从较旧的 KubeVirt 版本迁移
- [PR #5887][ashleyschuett] 允许 virtctl 停止 VM 并忽略优雅关闭期限
- [PR #5907][kubevirt-bot] 修复：ioerror 不会再导致 notify 服务器崩溃循环
- [PR #5871][maiqueb] 修复：不要用网关信息覆盖 DHCP 服务器所通告的 IP。
- [PR #5875][kubevirt-bot] 如果 ca-bundle 无法被解析，则更新它

## v0.42.1 {#v0421}

发布于：Thu Jun 10 01:31:52 2021 +0000

- [PR #5738][rmohr] 停止发布我们 operator 的 jinja2 模板。Kustomize 是进行定制的首选方式。
- [PR #5691][ashleyschuett] 允许多次关闭事件，以确保 ACPI 能够收到该事件
- [PR #5558][ormergi] 移除 virt-launcher 的 SYS_RESOURCE 权能
- [PR #5694][davidvossel] 修复迁移控制器中的空指针解引用
- [PR #5416][iholder-redhat] [功能] 支持使用自定义内核参数，从自定义内核/initrd 镜像启动 VM
- [PR #5495][iholder-redhat] Go 版本已更新到 1.16.1。
- [PR #5502][rmohr] 添加 downwardMetrics 卷，向客户机暴露有限的一组宿主机指标
- [PR #5601][maya-r] 将 libvirt-go 更新到 7.3.0
- [PR #5661][davidvossel] 校验/变更 Webhook 现在显式定义 10 秒超时时间
- [PR #5652][rmohr] 自动发现 kube-prometheus 安装并配置 kubevirt 监控
- [PR #5631][davidvossel] 扩展向后移植策略，以包含日志与调试修复
- [PR #5528][zcahana] 在 VirtualMachine CRD 中引入了 "status.printableStatus" 字段。该字段现在显示在 "kubectl get vm" 的表格输出中。
- [PR #5200][rhrazdil] 添加对配合 masquerade 接口的 Istio 代理流量路由的支持。此功能需要 nftables。
- [PR #5560][oshoval] virt-launcher 现在在进行周期性重新同步时，也会根据客户机代理填充域的 guestOS 信息与接口状态。
- [PR #5514][rhrazdil] 修复当带有 masquerade 接口的 VM 显式指定以下任一端口时实时迁移失败的问题：22222、49152、49153
- [PR #5583][dhiller] 重新启用覆盖率
- [PR #5129][davidvossel] 优雅关闭 virt-api 连接，并确保在正常关闭条件下退出码为零
- [PR #5582][dhiller] 修复不稳定的单元测试
- [PR #5600][davidvossel] 改进 VM/VMI 关闭与重启相关的日志
- [PR #5564][omeryahud] 已放弃对 virtctl rename 的支持
- [PR #5585][iholder-redhat] [缺陷修复] - 拒绝定义了卷但没有匹配磁盘的 VM
- [PR #5595][zcahana] 修复对孤立 DataVolume 的收养
- [PR #5566][davidvossel] 发布分支现在在每月的第一个_工作日_切出，而不是每月的第一天。
- [PR #5108][Omar007] 通过依据设备信息而不是挂载信息来处理，修复了对 /proc/<pid>/mountpoint 的处理
- [PR #5250][mlsorensen] 控制器健康检查将不再主动测试与 Kubernetes API 的连通性。它们将依据其 watch 的健康状况来判断是否具备 API 连通性。
- [PR #5563][ashleyschuett] 在 KubeVirt CR 中设置 KubeVirt 资源标志
- [PR #5328][andreabolognani] 此版本的 KubeVirt 包含基于 libvirt 7.0.0 与 QEMU 5.2.0 升级的虚拟化技术。

## v0.42.0 {#v0420}

发布于：Tue Jun 8 12:09:49 2021 +0000

- [PR #5738][rmohr] 停止发布我们 operator 的 jinja2 模板。Kustomize 是进行定制的首选方式。
- [PR #5691][ashleyschuett] 允许多次关闭事件，以确保 ACPI 能够收到该事件
- [PR #5558][ormergi] 移除 virt-launcher 的 SYS_RESOURCE 权能
- [PR #5694][davidvossel] 修复迁移控制器中的空指针解引用
- [PR #5416][iholder-redhat] [功能] 支持使用自定义内核参数，从自定义内核/initrd 镜像启动 VM
- [PR #5495][iholder-redhat] Go 版本已更新到 1.16.1。
- [PR #5502][rmohr] 添加 downwardMetrics 卷，向客户机暴露有限的一组宿主机指标
- [PR #5601][maya-r] 将 libvirt-go 更新到 7.3.0
- [PR #5661][davidvossel] 校验/变更 Webhook 现在显式定义 10 秒超时时间
- [PR #5652][rmohr] 自动发现 kube-prometheus 安装并配置 kubevirt 监控
- [PR #5631][davidvossel] 扩展向后移植策略，以包含日志与调试修复
- [PR #5528][zcahana] 在 VirtualMachine CRD 中引入了 "status.printableStatus" 字段。该字段现在显示在 "kubectl get vm" 的表格输出中。
- [PR #5200][rhrazdil] 添加对配合 masquerade 接口的 Istio 代理流量路由的支持。此功能需要 nftables。
- [PR #5560][oshoval] virt-launcher 现在在进行周期性重新同步时，也会根据客户机代理填充域的 guestOS 信息与接口状态。
- [PR #5514][rhrazdil] 修复当带有 masquerade 接口的 VM 显式指定以下任一端口时实时迁移失败的问题：22222、49152、49153
- [PR #5583][dhiller] 重新启用覆盖率
- [PR #5129][davidvossel] 优雅关闭 virt-api 连接，并确保在正常关闭条件下退出码为零
- [PR #5582][dhiller] 修复不稳定的单元测试
- [PR #5600][davidvossel] 改进 VM/VMI 关闭与重启相关的日志
- [PR #5564][omeryahud] 已放弃对 virtctl rename 的支持
- [PR #5585][iholder-redhat] [缺陷修复] - 拒绝定义了卷但没有匹配磁盘的 VM
- [PR #5595][zcahana] 修复对孤立 DataVolume 的收养
- [PR #5566][davidvossel] 发布分支现在在每月的第一个_工作日_切出，而不是每月的第一天。
- [PR #5108][Omar007] 通过依据设备信息而不是挂载信息来处理，修复了对 /proc/<pid>/mountpoint 的处理
- [PR #5250][mlsorensen] 控制器健康检查将不再主动测试与 Kubernetes API 的连通性。它们将依据其 watch 的健康状况来判断是否具备 API 连通性。
- [PR #5563][ashleyschuett] 在 KubeVirt CR 中设置 KubeVirt 资源标志
- [PR #5328][andreabolognani] 此版本的 KubeVirt 包含基于 libvirt 7.0.0 与 QEMU 5.2.0 升级的虚拟化技术。

## v0.41.4 {#v0414}

发布于：Tue Oct 19 15:31:59 2021 +0000

- [PR #6573][acardace] 变更迁移 PDB，而不是在迁移期间额外创建一个。
- [PR #6517][jean-edouard] 带有 cloud-init 数据的 VM 现在应当能够正确地从较旧的 KubeVirt 版本迁移
- [PR #6333][acardace] 修复 virt-launcher 退出 Pod 的竞态条件
- [PR #6401][rmohr] 修复 freeze/unfreeze、addvolume/removevolume、guestosinfo、filesystemlist 与 userlist 的 RBAC 权限
- [PR #6147][rmohr] 修复 freeze/unfreeze、addvolume/removevolume、guestosinfo、filesystemlist 与 userlist 的 RBAC 权限
- [PR #5673][kubevirt-bot] 改进 VM/VMI 关闭与重启相关的日志
- [PR #6227][kwiesmueller] 修复 virt-handler 中的 goroutine 泄漏，该问题在 VMI 高周转时可能导致故障。

## v0.41.3 {#v0413}

发布于：Thu Aug 12 16:35:43 2021 +0000

- [PR #6196][ashleyschuett] 允许多次关闭事件，以确保 ACPI 能够收到该事件
- [PR #6194][kubevirt-bot] 允许在使用 `--force --gracePeriod 0` 时停止处于 Failed 状态的 VM
- [PR #6039][akalenyu] 缺陷修复：创建由 WFFC DV 支持的并发批量 VM 时，VMI 停留在 Pending
- [PR #5917][davidvossel] 修复事件记录导致 virt-controller 段错误的问题
- [PR #5886][ashleyschuett] 允许 virtctl 停止 VM 并忽略优雅关闭期限
- [PR #5866][xpivarc] 修复：使用 golang 1.14+ 构建 KubeVirt 时，不会再因内存分配错误而在校验容器磁盘时失败
- [PR #5873][kubevirt-bot] 如果 ca-bundle 无法被解析，则更新它
- [PR #5822][kubevirt-bot] 已将 authorization/v1beta1 的引用迁移到 authorization/v1
- [PR #5704][davidvossel] 修复 virt-controller 在 virt-handler 交接期间覆盖进行中的 VMI 迁移状态
- [PR #5707][kubevirt-bot] 修复迁移控制器中的空指针解引用
- [PR #5685][stu-gott] [缺陷修复] - 拒绝定义了卷但没有匹配磁盘的 VM
- [PR #5670][stu-gott] 校验/变更 Webhook 现在显式定义 10 秒超时时间
- [PR #5653][kubevirt-bot] virt-launcher 现在在进行周期性重新同步时，也会根据客户机代理填充域的 guestOS 信息与接口状态。
- [PR #5644][kubevirt-bot] 修复当带有 masquerade 接口的 VM 显式指定以下任一端口时实时迁移失败的问题：22222、49152、49153
- [PR #5646][kubevirt-bot] 已放弃对 virtctl rename 的支持

## v0.41.2 {#v0412}

发布于：Wed Jul 28 12:13:19 2021 -0400

## v0.41.1 {#v0411}

发布于：Wed Jul 28 12:08:42 2021 -0400

## v0.41.0 {#v0410}

发布于：Wed May 12 14:30:49 2021 +0000

- [PR #5586][kubevirt-bot] 此版本的 KubeVirt 包含基于 libvirt 7.0.0 与 QEMU 5.2.0 升级的虚拟化技术。
- [PR #5344][ashleyschuett] 调和 PrometheusRules 与 ServiceMonitor 资源
- [PR #5542][andreyod] 向 VMI spec 添加 startStrategy 字段，以允许虚拟机在暂停状态下启动。
- [PR #5459][ashleyschuett] 调和 Service 资源
- [PR #5520][ashleyschuett] 调和 ConfigMap 资源上必需的标签与注解
- [PR #5533][rmohr] 修复已发布 kubevirt 镜像的 `docker save` 与 `docker push` 问题
- [PR #5428][oshoval] virt-launcher 现在在进行周期性重新同步时，也会根据客户机代理填充域的 guestOS 信息与接口状态。
- [PR #5410][ashleyschuett] 调和 ServiceAccount 资源
- [PR #5109][Omar007] 添加为磁盘设备指定逻辑块大小与物理块大小的支持
- [PR #5471][ashleyschuett] 调和 APIService 资源
- [PR #5513][ashleyschuett] 调和 Secret 资源
- [PR #5496][davidvossel] 改进迁移代理日志
- [PR #5376][ashleyschuett] 调和 CustomResourceDefinition 资源
- [PR #5435][AlonaKaplan] 在 "virtctl expose" 上支持双栈服务-
- [PR #5425][davidvossel] 修复 EvictionStrategy=LiveMigrate 时驱逐期间 VM 重启的问题
- [PR #5423][ashleyschuett] 为 virt-controller、virt-api、virt-operator 与 virt-handler 添加资源请求
- [PR #5343][erkanerol] 对存储指标进行了一些清理与小幅补充
- [PR #4682][stu-gott] 更新了客户机代理版本兼容性检查。新方法准确得多。
- [PR #5485][rmohr] 修复在 arm64 上当宿主机未使用 nftables 时回退到 iptables 的问题
- [PR #5426][rmohr] 修复当宿主机未使用 nftables 时回退到 iptables 的问题
- [PR #5403][tiraboschi] 为若干记录规则与指标添加了 kubevirt_ 前缀
- [PR #5241][stu-gott] 为证书轮换引入了 Duration 与 RenewBefore 参数。先前的值现已弃用。
- [PR #5463][acardace] 修复从 KubeVirt v0.36 的升级
- [PR #5456][zhlhahaha] 启用 arm64 交叉编译
- [PR #3310][davidvossel] 文档阐述了我们关于 Kubernetes 版本兼容性的承诺
- [PR #3383][EdDev] 添加 `vmIPv6NetworkCIDR` 于 `NetworkSource.pod` 之下，以便在使用 masquerade 绑定时为 vm 网络支持自定义 IPv6 CIDR。
- [PR #3415][zhlhahaha] 使 kubevirt 代码适配 arm64 支持。现阶段尚未针对 arm64 进行测试。
- [PR #5147][xpivarc] 从 virt-launcher Pod 中移除 CAP_NET_ADMIN（第二次尝试）。
- [PR #5351][awels] 支持通过 virtctl 使用 addvolume 与 removevolume 命令进行热插拔
- [PR #5050][ashleyschuett] 当 VMI 孤立超过一小时时触发 Prometheus 告警

## v0.40.1 {#v0401}

发布于：Tue Oct 19 13:33:33 2021 +0000

- [PR #6598][jean-edouard] 带有 cloud-init 数据的 VM 现在应当能够正确地从较旧的 KubeVirt 版本迁移
- [PR #6287][kubevirt-bot] 修复 virt-handler 中的 goroutine 泄漏，该问题在 VMI 高周转时可能导致故障。
- [PR #5559][kubevirt-bot] 修复 kubevirt 镜像的 `docker save` 问题
- [PR #5500][kubevirt-bot] 支持通过 virtctl 使用 addvolume 与 removevolume 命令进行热插拔

## v0.40.0 {#v0400}

发布于：Mon Apr 19 12:25:41 2021 +0000

- [PR #5467][rmohr] 修复从 KubeVirt v0.36 的升级
- [PR #5350][jean-edouard] 移除整个 `permittedHostDevices` 节现在会移除所有用户定义的主机设备插件。
- [PR #5242][jean-edouard] 针对给定 VMI 同时创建超过 1 个迁移现在将会失败
- [PR #4907][vasiliy-ul] cgroupv2 的初步支持
- [PR #5324][jean-edouard] 现在可以在 provider 配置中定义默认功能门控。
- [PR #5006][alicefr] 添加 discard=unmap 选项
- [PR #5022][davidvossel] 修复 operator 添加 Service 与 Webhook 之间的竞态条件，该竞态可能导致安装/卸载失败
- [PR #5310][ashleyschuett] 调和 CRD 资源
- [PR #5102][iholder-redhat] Go 版本已更新到 1.14.14
- [PR #4746][ashleyschuett] 调和 Deployments、DaemonSets、MutatingWebhookConfigurations 与 ValidatingWebhookConfigurations
- [PR #5037][ormergi] 在成功迁移之后，向 VM 热插拔 SR-IOV VF 接口。
- [PR #5269][mlsorensen] 从 virt-handler 抓取的 Prometheus 指标现在由 VMI Informer 缓存提供，而不再回查 Kubernetes API 以获取 VMI 信息。
- [PR #5138][davidvossel] virt-handler 现在会在关闭前最多等待 5 分钟，直至该节点上的所有迁移完成。
- [PR #5191][yuvalturg] 添加了用于监控 CPU 亲和性的指标
- [PR #5215][xphyr] 启用对 Intel GVT-g vGPU 的检测。
- [PR #4760][rmohr] 使 virt-handler 心跳更高效、更稳健：只需一次合并的 PATCH，并且不再需要检测不同的集群类型。
- [PR #5091][iholder-redhat] QEMU SeaBIOS 调试日志会出现在 virt-launcher 日志中。
- [PR #5221][rmohr] 移除工作负载放置校验 Webhook，该 Webhook 会在 VMI 运行时阻止放置更新
- [PR #5128][yuvalturg] 修改了内存相关指标：新增多项指标，并拆分了交换流量字节指标
- [PR #5084][ashleyschuett] 为 KubeVirt 资源上的 CustomizeComponents 对象添加校验
- [PR #5182][davidvossel] 新增 [release-blocker] 功能测试标记，用于标示在发布之前绝不能禁用的测试
- [PR #5137][davidvossel] 在 docs/release-branch-backporting.md 中添加了关于发布分支向后移植的策略
- [PR #5096][yuvalturg] 修改了网络指标：新增指标，按 rx/tx 拆分现有指标，并在可用时使用设备别名作为接口名称
- [PR #5088][awels] 热插拔可用于 hostpath 存储。
- [PR #4908][dhiller] 将 travis 的 tag 与 master 构建移至 kubevirt prow。
- [PR #4741][EdDev] 允许对 SR-IOV VM 进行实时迁移，且不保留 VF 接口。

## v0.39.2 {#v0392}

发布于：Tue Oct 19 13:29:33 2021 +0000

- [PR #6597][jean-edouard] 带有 cloud-init 数据的 VM 现在应当能够正确地从较旧的 KubeVirt 版本迁移
- [PR #5854][rthallisey] 从 virt-handler 抓取的 Prometheus 指标现在由 VMI Informer 缓存提供，而不再回查 Kubernetes API 以获取 VMI 信息。
- [PR #5561][kubevirt-bot] 修复 kubevirt 镜像的 `docker save` 问题

## v0.39.1 {#v0391}

发布于：Tue Apr 13 12:10:13 2021 +0000
## v0.39.0 {#v0390}

发布于：Wed Mar 10 14:51:58 2021 +0000

- [PR #5010][jean-edouard] 迁移后的 VM 保持持久，因此能够在 S3 等情况下存活。
- [PR #4952][ashleyschuett] 如果节点正在运行 VMI Pod 但没有 virt-handler Pod，则创建警告事件 NodeUnresponsive
- [PR #4686][davidvossel] 通过新的 KubeVirt WorkloadUpdateStrategy API 自动更新工作负载
- [PR #4886][awels] 支持 WFFC DataVolume 的热插拔。
- [PR #5026][AlonaKaplan] virt-launcher，masquerade 绑定：优先使用 nft 而非 iptables。
- [PR #4921][borod108] 在 API 中添加了对 Sysprep 的支持。用户现在可以通过 ConfigMap 或 Secret 添加应答文件。用户指南已相应更新。 /kind feature
- [PR #4874][ormergi] 新增功能门控 SRIOVLiveMigration，
- [PR #4917][iholder-redhat] 现在可以将 virt-launcher 日志详细程度设置为大于 5，从而启用 QEMU SeaBIOS 调试日志。
- [PR #4966][arnongilboa] 解决成功上传镜像后 virtctl 出现的 "Error when closing file ... file already closed"
- [PR #4489][salanki] 修复了在通过 Virtio-FS 挂载的文件系统上创建 disk.img 文件的缺陷
- [PR #4982][xpivarc] 修复临时域的处理
- [PR #4984][ashleyschuett] 更改 customizeComponents.patches，使 resourceName 或 resourceType 为 '*' 时匹配全部；补丁的所有字段（type、patch、resourceName、resourceType）现在都是必需的。
- [PR #4972][vladikr] 允许禁用 pvspinlock，以支持较旧的客户机内核
- [PR #4927][yuhaohaoyu] 修复用户定义设备别名的 XML 与 JSON 编组/解组，该问题可能导致迁移失败。
- [PR #4552][rthallisey] 使用桥接网络的 VM 能够在 kubelet 重启后继续运行，因为 kubevirt 会在 virt-launcher Pod 上创建一个哑接口，从而使某些已实现 `CHECK` RPC 调用的 Kubernetes CNI 不会导致 VMI Pod 进入失败状态。
- [PR #4883][iholder-redhat] 缺陷已修复：仅当 debugLogs 标签值为 "true" 时启用 libvirt 调试日志，否则禁用。
- [PR #4840][alicefr] 在 IO 错误时生成 k8s 事件
- [PR #4940][vladikr] permittedHostDevices 将支持设备 ID 中的大写与小写字母

## v0.38.2 {#v0382}

发布于：Tue Oct 19 13:24:57 2021 +0000

- [PR #6596][jean-edouard] 带有 cloud-init 数据的 VM 现在应当能够正确地从较旧的 KubeVirt 版本迁移
- [PR #5853][rthallisey] 从 virt-handler 抓取的 Prometheus 指标现在由 VMI Informer 缓存提供，而不再回查 Kubernetes API 以获取 VMI 信息。

## v0.38.1 {#v0381}

发布于：Mon Feb 8 19:00:24 2021 +0000

- [PR #4870][qinqon] 将 k8s 依赖升级到 0.20.2
- [PR #4571][yuvalturg] 为 kubevirt_vmi_phase_count 指标添加了 os、workflow 与 flavor 标签
- [PR #4659][salanki] 修复了客户机内的非 root 用户无法写入 Virtio-FS 挂载的问题。
- [PR #4844][xpivarc] 修复 limits/requests，使其再次接受 int
- [PR #4850][rmohr] virtio-scsi 现在会遵循 useTransitionalVirtio 标志，而不再根据机器布局分配 virtio 版本
- [PR #4672][vladikr] 允许在 KubeVirt CR 中提高基础设施组件的日志详细程度
- [PR #4838][rmohr] 修复由于证书传播延迟，创建后最长可能数分钟无法更新 KubeVirt CR 的问题
- [PR #4806][rmohr] 将 VMI 与 VM 的变更 Webhook 设为必需，以避免未经正确填充默认值的实体进入集群
- [PR #4779][brybacki] 向 WaitForFirstConsumer DV 执行 virtctl image-upload 时的错误信息
- [PR #4749][davidvossel] 环境变量 KUBEVIRT_CLIENT_GO_SCHEME_REGISTRATION_VERSION，用于精确指定所注册的 client-go scheme 版本
- [PR #4772][jean-edouard] 得益于 virt-controller 中 VMI watch 线程数量的增加，VMI 阶段转换更快
- [PR #4730][rmohr] 添加 spec.domain.devices.useVirtioTransitional 布尔字段，以便为旧客户机支持 virtio-transitional

## v0.38.0 {#v0380}

发布于：Mon Feb 8 13:15:32 2021 +0000

- [PR #4870][qinqon] 将 k8s 依赖升级到 0.20.2
- [PR #4571][yuvalturg] 为 kubevirt_vmi_phase_count 指标添加了 os、workflow 与 flavor 标签
- [PR #4659][salanki] 修复了客户机内的非 root 用户无法写入 Virtio-FS 挂载的问题。
- [PR #4844][xpivarc] 修复 limits/requests，使其再次接受 int
- [PR #4850][rmohr] virtio-scsi 现在会遵循 useTransitionalVirtio 标志，而不再根据机器布局分配 virtio 版本
- [PR #4672][vladikr] 允许在 KubeVirt CR 中提高基础设施组件的日志详细程度
- [PR #4838][rmohr] 修复由于证书传播延迟，创建后最长可能数分钟无法更新 KubeVirt CR 的问题
- [PR #4806][rmohr] 将 VMI 与 VM 的变更 Webhook 设为必需，以避免未经正确填充默认值的实体进入集群
- [PR #4779][brybacki] 向 WaitForFirstConsumer DV 执行 virtctl image-upload 时的错误信息
- [PR #4749][davidvossel] 环境变量 KUBEVIRT_CLIENT_GO_SCHEME_REGISTRATION_VERSION，用于精确指定所注册的 client-go scheme 版本
- [PR #4772][jean-edouard] 得益于 virt-controller 中 VMI watch 线程数量的增加，VMI 阶段转换更快
- [PR #4730][rmohr] 添加 spec.domain.devices.useVirtioTransitional 布尔字段，以便为旧客户机支持 virtio-transitional

## v0.37.2 {#v0372}

发布于：Wed Jan 27 17:49:36 2021 +0000

- [PR #4872][kubevirt-bot] 新增布尔字段 spec.domain.devices.useVirtioTransitional，以便为旧客户机支持 virtio-transitional
- [PR #4855][kubevirt-bot] 修复创建之后因证书传播延迟，最长可达数分钟无法更新 KubeVirt CR 的问题

## v0.37.1 {#v0371}

发布于：Thu Jan 21 16:20:52 2021 +0000

- [PR #4842][kubevirt-bot] 新增环境变量 KUBEVIRT_CLIENT_GO_SCHEME_REGISTRATION_VERSION，用于精确指定所注册的 client-go scheme 版本

## v0.37.0 {#v0370}

发布于：Mon Jan 18 17:57:03 2021 +0000

- [PR #4654][AlonaKaplan] 引入 virt-launcher DHCPv6 服务器。
- [PR #4669][kwiesmueller] 为 KubeVirt 组件添加 nodeSelector，将它们限制为只在 Linux 节点上运行。
- [PR #4648][davidvossel] 将 libvirt 基础容器更新为基于 rhel-av 8.3 中的软件包
- [PR #4653][qinqon] 允许只用 networkData 配置 cloud-init。
- [PR #4644][ashleyschuett] 如果存在正在运行的 VMI，Operator 校验 webhook 将拒绝更新 KubeVirt CR 的 workloads 对象
- [PR #3349][davidvossel] KubeVirt v1 API 达到 GA
- [PR #4645][maiqueb] 重新引入 CAP_NET_ADMIN，以便已经具备该能力的 VM 可以迁移。
- [PR #4546][yuhaohaoyu] 在设计上不支持 EFI 非安全启动的 KubeVirt 环境中，对使用 EFI 非安全启动的 VM 进行故障检测与处理。
- [PR #4625][awels] virtctl upload 在指定访问模式 ReadOnlyMany 时现在会显示错误
- [PR #4396][xpivarc] KubeVirt 现在可以被 explain 了！
- [PR #4517][danielBelenky] 修复客户机代理上报。

## v0.36.2 {#v0362}

发布于：Mon Feb 22 10:20:40 2021 -0500

## v0.36.1 {#v0361}

发布于：Tue Jan 19 12:30:33 2021 +0100

## v0.36.0 {#v0360}

发布于：Wed Dec 16 14:30:37 2020 +0000

- [PR #4667][kubevirt-bot] 将 libvirt 基础容器更新为基于 rhel-av 8.3 中的软件包
- [PR #4634][kubevirt-bot] 在设计上不支持 EFI 非安全启动的 KubeVirt 环境中，对使用 EFI 非安全启动的 VM 进行故障检测与处理。
- [PR #4647][kubevirt-bot] 重新引入 CAP_NET_ADMIN，以便已经具备该能力的 VM 可以迁移。
- [PR #4627][kubevirt-bot] 修复客户机代理上报。
- [PR #4458][awels] 现在可以将 DataVolume 和 PVC 卷热插拔到正在运行的虚拟机中。
- [PR #4025][brybacki] 为处于 WaitForFirstConsumer 状态的 DataVolume 增加特殊处理，以支持 CDI 的延迟绑定模式。
- [PR #4217][mfranczy] 对 qemu-guest-agent 上报的接口只设置 IP 地址。以前设置的是 CIDR。
- [PR #4195][davidvossel] 用于动态注入用户名/密码和 SSH 公钥的 AccessCredentials API
- [PR #4335][oshoval] VMI 状态仅在 SR-IOV 接口原本带有网络名称时才显示该网络名称
- [PR #4408][andreabolognani] 此版本的 KubeVirt 包含基于 libvirt 6.6.0 和 QEMU 5.1.0 升级后的虚拟化技术。
- [PR #4514][ArthurSens] `domain` 标签已从指标 `kubevirt_vmi_memory_unused_bytes` 中移除
- [PR #4542][danielBelenky] 修复节点疏散时的双重迁移
- [PR #4506][maiqueb] 从 virt-launcher Pod 中移除 CAP_NET_ADMIN。
- [PR #4501][AlonaKaplan] 已从 virt-launcher 中移除 CAP_NET_RAW。
- [PR #4488][salanki] 禁用 Virtio-FS 元数据缓存，以防止宿主机出现 OOM。
- [PR #3937][vladikr] 泛化主机设备分配。在 KubeVirt 与外部设备插件之间提供接口。提供主机设备的访问列表机制。
- [PR #4443][rmohr] 所有 KubeVirt webhook 现在都支持试运行。

## v0.35.0 {#v0350}

发布于：Mon Nov 9 13:08:27 2020 +0000

- [PR #4409][vladikr] 将静态内存开销增加 10Mi
- [PR #4272][maiqueb] 添加 `ip-family` 到 `virtctl expose` 命令。
- [PR #4398][rmohr] VMI 会在就绪状况中以 “PodTerminating” 原因反映已被删除且卡住的 virt-launcher Pod。VMIR 检测到该原因后会立即创建用于替换的 VMI。
- [PR #4393][salanki] 在 `virt-launcher` Pod 中禁用旧式 Service 链接，以加快 Pod 实例化，并降低含有大量 Service 的命名空间中的 Kubelet 负载。
- [PR #2935][maiqueb] 添加 macvtap 绑定机制。
- [PR #4132][mstarostik] 修复同时配置 SCSI 和 SATA 驱动器时无法分配唯一设备名称的缺陷
- [PR #3257][xpivarc] 为 KubeVirt 资源添加了 `kubectl explain` 支持。
- [PR #4288][ezrasilvera] 添加 DownwardAPI 卷类型
- [PR #4233][maya-r] 将 Pod 使用的基础镜像更新为 Fedora 31。
- [PR #4192][xpivarc] 现在会在 KubeVirt 中运行 gosec
- [PR #4328][stu-gott] 支持 2.x 版本的 QEMU 客户机代理。
- [PR #4289][AlonaKaplan] Masquerade 绑定：在网桥上设置 virt-launcher Pod 接口的 MTU。
- [PR #4300][maiqueb] 更新 NetworkInterfaceMultiqueue 的 OpenAPI 文档，以便更清楚地说明它在 KubeVirt 中的语义。
- [PR #4277][awels] 现在允许把由 DV 填充的 PVC 用作卷。
- [PR #4265][dhiller] 修复 virtctl 作为插件运行时的帮助文本
- [PR #4273][dhiller] 只对目标为发布分支的 PR 运行 Travis 构建

## v0.34.2 {#v0342}

发布于：Tue Nov 17 08:13:22 2020 -0500

## v0.34.1 {#v0341}

发布于：Mon Nov 16 08:22:56 2020 -0500

## v0.34.0 {#v0340}

发布于：Wed Oct 7 13:59:50 2020 +0300

- [PR #4315][kubevirt-bot] 现在允许把由 DV 填充的 PVC 用作卷。
- [PR #3837][jean-edouard] 按文档所述，使用 BIOS 引导加载程序时，没有 `bootOrder` 的 VM 接口将不再作为启动候选
- [PR #3879][ashleyschuett] 现在应通过 KubeVirt CR 的 `configuration` 键来配置 KubeVirt。kubevirt-config ConfigMap 的用法将来会被弃用。
- [PR #4074][stu-gott] 修复了导致非管理员用户无法暂停/取消暂停 VM 的缺陷
- [PR #4252][rhrazdil] 修复 https://bugzilla.redhat.com/show_bug.cgi?id=1853911
- [PR #4016][ashleyschuett] 允许进行 post-copy VMI 迁移
- [PR #4235][davidvossel] 修复拉取大型 containerDisk 镜像时出现的超时失败
- [PR #4263][rmohr] 为 virt-handler 添加就绪探针和存活探针，以明确表示就绪状态
- [PR #4248][maiqueb] 在纯 Go 构建中始终带 SELinux 支持来编译 KubeVirt。
- [PR #4012][danielBelenky] 为设置了驱逐策略的 VMI 添加驱逐 API 支持。这样，当节点被排空时，或 descheduler 要将 VMI 移到其他节点时，可以对 VMI 进行实时迁移。
- [PR #4075][ArthurSens] 指标 kubevirt_vmi_vcpu_seconds 的 state 标签现在以人类可读的状态暴露，不再是整数
- [PR #4162][vladikr] 引入配置参数 cpuAllocationRatio，按 vCPU 数量对 Pod 所请求的 CPU 数量进行归一化
- [PR #4177][maiqueb] 创建 tap 设备时改用 vishvananda/netlink，替换 songgao/water。
- [PR #4092][stu-gott] 允许指定 nodeSelectors、affinity 和 tolerations，以控制 KubeVirt 组件的运行位置
- [PR #3927][ArthurSens] 新增指标 kubevirt_vmi_memory_unused_bytes
- [PR #3493][vladikr] virtio-fs 作为实验性功能加入，并由功能门控保护，需要管理员在 KubeVirt 配置中启用
- [PR #4193][mhenriks] 将 snapshot.kubevirt.io 添加到 admin/edit/view 角色中
- [PR #4149][qinqon] 将 kubevirtci 升级到 k8s-1.19
- [PR #3471][crobinso] 允许隐藏 VM 正运行在 KVM 上，以便能够直通 Nvidia 显卡
- [PR #4115][phoracek] 添加一致性测试自动化和清单发布
- [PR #3733][davidvossel] 每个 PR 的说明。
- [PR #4082][mhenriks] VirtualMachineRestore 的 API 与实现
- [PR #4154][davidvossel] 修复 KubeVirt 更新期间 Service 端点未能正确就地更新的问题。
- [PR #3289][vatsalparekh] 添加只在 virtctl 中运行 VNC 代理的选项
- [PR #4027][alicefr] 将 memfd 作为大页的默认内存后端。同时引入新注解 kubevirt.io/memfd，用于禁用默认的 memfd，并回退到先前的行为。
- [PR #3612][ashleyschuett] 向 KubeVirt API 添加 `customizeComponents`
- [PR #4029][cchengleo] 修复了导致 virt-operator 无法在自定义命名空间中安装监控资源的问题。
- [PR #4031][rmohr] 初步支持使用 sonobuoy 进行一致性测试

## v0.33.0 {#v0330}

发布于：Tue Sep 15 14:46:00 2020 +0000

- [PR #3226][vatsalparekh] 添加测试，以验证自定义 pciAddress 的槽位和功能号
- [PR #4048][davidvossel] 提高失败迁移重试的可靠性
- [PR #3585][mhenriks] “virtctl image-upload pvc ...” 会在 PVC 不存在时创建该 PVC
- [PR #3945][xpivarc] KubeVirt 现在使用 Go 1.13.14 进行构建
- [PR #3845][ArthurSens] 需要采取行动：正在移除 VMI 指标中的 domain 标签，这可能会破坏使用 domain 标签来识别 VMI 的仪表板。请改用 name 和 namespace 标签
- [PR #4011][dhiller] ppc64le 架构暂时被禁用，参见 https://github.com/kubevirt/kubevirt/issues/4037
- [PR #3875][stu-gott] KubeVirt 所创建资源的标签现在能更清楚地体现关系与角色。
- [PR #3791][ashleyschuett] 在 virt-handler 重启时将节点设为 kubevirt.io/schedulable=false
- [PR #3998][vladikr] 本地 provider 再次可用。
- [PR #3290][maiqueb] 由 virt-handler（KubeVirt 代理）代表 virt-launcher 创建 tap 设备。
- [PR #3957][AlonaKaplan] virt-launcher 在双栈集群上支持 IPv6。
- [PR #3952][davidvossel] 修复域启动前发生故障时 VMI 可能无法正确终止的罕见情况。
- [PR #3973][xpivarc] 修复设置了 clock.timezone 的 VM。
- [PR #3923][danielBelenky] 添加为 VMI 配置 QEMU I/O 模式的支持
- [PR #3889][rmohr] 我们 CRD 的 status 字段现在在普通的 PATCH 和 PUT 操作中受到保护。现在会尽可能使用 /status 子资源来更新状态。
- [PR #3568][xpivarc] 客户机交换区指标现已可用

## v0.32.0 {#v0320}

发布于：Tue Aug 11 19:21:56 2020 +0000

- [PR #3921][vladikr] 在 libvirt XML 中使用正确的内存单位
- [PR #3893][davidvossel] 增加周期性过程，用 rsync 将 virt-launcher 的域与 virt-handler 同步
- [PR #3880][sgarbour] 当输入参数数量与每个参数所期望的数量不符时，给出更好的错误信息。参数数量不正确时会弹出帮助菜单。
- [PR #3785][xpivarc] vCPU 等待指标现已可用
- [PR #3642][vatsalparekh] 增加一种方式，用最新的 Pod IP 更新 Masquerade 绑定的 VMI Status
- [PR #3636][ArthurSens] 将 Kubernetes 的 metadata.labels 添加为 VMI 指标的标签
- [PR #3825][awels] virtctl 现在会在上传出错时打印响应体中的错误信息。
- [PR #3830][davidvossel] 修复 domain notify 服务器因错误事件重启后重新建立 domain notify 客户端连接的问题。
- [PR #3778][danielBelenky] 如果同步处于最终状态的 VMI 失败，则不发出 SyncFailed 事件
- [PR #3803][andreabolognani] 不确定这里该写什么（见上文）
- [PR #2694][rmohr] 使用原生 Go 库处理 SELinux，从而不依赖 semanage 这类并非总是存在的 python-selinux 工具。
- [PR #3692][victortoso] 现在可以从 Pod 外部获取 QEMU 日志
- [PR #3738][enp0s3] 如果 VMI 带有 KubeVirt 组件内部使用的标签，则限制创建该 VMI。
- [PR #3725][danielBelenky] 测试二进制文件现在是发行内容的一部分，可从 GitHub 发布页面获取。
- [PR #3684][rmohr] 如果 virt-handler 希望暴露的关键设备（例如 kvm）在节点上不存在，则记录日志。
- [PR #3166][petrkotas] 引入新的 virtctl 命令：
- [PR #3708][andreabolognani] 通过引入针对 https://bugzilla.redhat.com/show_bug.cgi?id=1822682 的修复，使 QEMU 能够在 GCE 上运行

## v0.31.0 {#v0310}

发布于：Thu Jul 9 16:08:18 2020 +0300

- [PR 3690][davidvossel] 将 go-grpc 依赖更新到 v1.30.0，以提高稳定性
- [PR 3628][AlonaKaplan] 避免 virt-launcher 网络配置出错时 virt-handler 崩溃
- [PR 3635][jean-edouard] 必须启用 “HostDisk” 功能门控才能使用 hostDisks
- [PR 3641][vatsalparekh] 回退 kubevirt/kubevirt#3488，因为 CI 似乎在并非所有测试都通过时就将其合并了
- [PR 3488][vatsalparekh] 增加一种方式，用最新的 Pod IP 更新 Masquerade 绑定的 VMI Status
- [PR 3406][tomob] 如果 PVC 由 DataVolume 创建，则不能把它用作 VM 的卷源。必须改用拥有该 PVC 的 DataVolume。
- [PR 3566][kraxel] 新增：Linux 和 Windows 上的 TigerVNC 支持
- [PR 3529][jean-edouard] 启用 EFI 的同时也会启用 Secure Boot，而这要求启用 SMM。
- [PR 3455][ashleyschuett] 将 KubevirtConfiguration、MigrationConfiguration、DeveloperConfiguration 和 NetworkConfiguration 添加到 API 类型中
- [PR 3520][rmohr] 修复 VMI 在 Scheduled 阶段出错时，在 VMI sync-condition 上出现热循环的问题
- [PR 3220][mhenriks] VirtualMachineSnapshots 的 API 以及控制器/webhook

## v0.30.7 {#v0307}

发布于：Mon Oct 26 11:57:21 2020 -0400

## v0.30.6 {#v0306}

发布于：Wed Aug 12 10:55:31 2020 +0200

## v0.30.5 {#v0305}

发布于：Fri Jul 17 05:26:37 2020 -0400

## v0.30.4 {#v0304}

发布于：Fri Jul 10 07:44:00 2020 -0400

## v0.30.3 {#v0303}

发布于：Tue Jun 30 17:39:42 2020 -0400

## v0.30.2 {#v0302}

发布于：Thu Jun 25 17:05:59 2020 -0400

## v0.30.1 {#v0301}

发布于：Tue Jun 16 13:10:17 2020 -0400

## v0.30.0 {#v0300}

发布于：Fri Jun 5 12:19:57 2020 +0200

- 测试：更多测试修复
- 安全：为 virt-launcher 引入自定义 SELinux 策略
- 为 IPv6 地址提供更易用的默认 IPv6 CIDR
- 通过改用 openapi-gen 修复 OpenAPI 兼容性问题
- 改进了对 EFI 启动的支持（可配置 OVMF 路径，并修复了测试）
- 改进了 VMI IP 上报
- 支持将注解从 VMI 传播到 Pod
- 支持向 virt-launcher 更细粒度地授予 NET_RAW 能力
- 支持与 DataVolume 的最终一致性

## v0.29.2 {#v0292}

发布于：Mon May 25 21:15:30 2020 +0200

## v0.29.1 {#v0291}

发布于：Tue May 19 10:03:27 2020 +0200

## v0.29.0 {#v0290}

发布于：Wed May 6 15:01:57 2020 +0200

- 测试：非常多的测试修复
- 测试：更多测试修复
- CI：添加启用 SELinux 的测试通道
- CI：暂时移除 PPC64 支持
- 移除 Genie 支持
- 为提高安全性，不再在 virt-launcher 中使用 hostPaths
- 为重要组件支持优先级类
- 支持经由 masquerade 绑定使用 IPv6
- 支持基于共享 Secret 的证书轮换
- 支持 VM 就绪状况
- 支持高级节点标签（所支持的 CPU 系列和机器类型）

## v0.28.0 {#v0280}

发布于：Thu Apr 9 23:01:29 2020 +0200

- CI：尝试在合并前发现不稳定测试
- 修复 priorityClasses 的使用
- 修复客户机内存开销计算
- 修复 SR-IOV 设备开销要求
- 修复 virt-handler 初始化期间加载 tun 模块的问题
- 修复若干测试用例
- 为支持以 container_t 运行所做的修复
- 支持重命名 VM
- 支持 ioEmulator 线程绑核
- 为 virt-handler 支持若干告警
- 支持通过客户机代理列出文件系统
- 支持从客户机代理获取数据
- 支持设备角色标记
- 支持将设备分配到 PCI 根总线
- 支持覆盖客户机开销
- 用 C 重写 container-disk，以降低其内存占用

## v0.27.0 {#v0270}

发布于：Fri Mar 6 22:40:34 2020 +0100

- 在 API 中支持更多客户机代理信息
- 支持在 VM 上设置 priorityClasses
- 支持通过 Prometheus 提供额外的控制平面告警
- 支持 io 和 emulator 线程绑核
- 支持为 launcher 设置自定义 SELinux 类型
- 支持由 handler 执行网络配置，替换原先由 launcher 执行的方式
- 支持选择不自动挂接串行控制台
- 支持多种卸载策略以保护数据
- 修复问题，使 QEMU 运行在 qemu 组中
- 修复例如实时迁移之后的客户机代理连通性检查

## v0.26.5 {#v0265}

发布于：Tue Apr 14 15:07:04 2020 -0400

## v0.26.4 {#v0264}

发布于：Mon Mar 30 03:43:48 2020 +0200

## v0.26.3 {#v0263}

发布于：Tue Mar 10 08:57:27 2020 -0400

## v0.26.2 {#v0262}

发布于：Tue Mar 3 12:31:56 2020 -0500

## v0.26.1 {#v0261}

发布于：Fri Feb 14 20:42:46 2020 +0100

## v0.26.0 {#v0260}

发布于：Fri Feb 7 09:40:07 2020 +0100

- 修复不正确的 ownerReferences，以免 VM 被垃圾回收
- 若干测试修复
- 通过把 Secret 相关的过宽权限委托给 kubelet 来修复该问题
- 通过提高内存请求，修复 infra Pod 的 OOM
- 阐明实时迁移相关的设备支持
- 支持卸载策略，以便在卸载期间保护工作负载
- 支持更多 Prometheus 指标和告警规则
- 支持在功能测试中测试 SR-IOV 连通性
- 将 Kubernetes client-go 更新到 1.16.4
- FOSSA 修复与状态

## v0.25.0 {#v0250}

发布于：Mon Jan 13 20:37:15 2020 +0100

- CI：支持 Kubernetes 1.17
- 支持 emulator 线程绑核
- 支持 virtctl restart --force
- 支持 virtctl migrate，以便从 CLI 触发实时迁移

## v0.24.0 {#v0240}

发布于：Tue Dec 3 15:34:34 2019 +0100

- CI：支持 Kubernetes 1.15
- CI：支持 Kubernetes 1.16
- 添加并修复若干测试用例
- 支持暂停和取消暂停 VM
- 将 libvirt 更新到 5.6.0
- 修复与并行抓取 Prometheus 端点相关的缺陷
- 修复问题，以便可靠地测试 VNC

## v0.23.3 {#v0233}

发布于：Tue Jan 21 13:17:20 2020 -0500

## v0.23.2 {#v0232}

发布于：Fri Jan 10 10:36:36 2020 -0500

## v0.23.1 {#v0231}

发布于：Thu Nov 28 09:36:41 2019 +0100

## v0.23.0 {#v0230}

发布于：Mon Nov 4 16:42:54 2019 +0100

- 客户机操作系统信息现在可在 VMI status 下获取
- 已更新到 Go 1.12.8 以及最新的 Bazel
- 将 go-yaml 更新到 v2.2.4，该版本修复了一个 DDoS 漏洞
- 清理并修复了 CRD scheme 注册
- 若干缺陷修复
- 多项 CI 改进（例如测试失败时提供更多日志）

## v0.22.0 {#v0220}

发布于：Thu Oct 10 18:55:08 2019 +0200

- 支持 Nvidia KubeVirt Device Plugin 所暴露的 Nvidia GPU 和 vGPU。
- 如果从 Pod 网络分配到以 0xfe 为前缀的 MAC 地址，VMI 现在可以成功启动
- 在 SELinux Permissive 模式下移除了对宿主机 semanage 的依赖
- 因进入 CNCF 沙箱而做的一些变更（DCO 检查、FOSSA 检查、最佳实践徽章）
- 多个方面的大量缺陷修复与改进
- CI：引入 OKD 4 测试通道
- CI：大量测试得到改进，不稳定性因此降低

## v0.21.0 {#v0210}

发布于：Mon Sep 9 09:59:08 2019 +0200

- CI：支持 Kubernetes 1.14
- 多个方面的大量缺陷修复
- 支持 `virtctl migrate`
- 支持可配置的控制器线程数量
- 支持让 podNetwork 选择不使用 bridge 绑定
- 支持 OpenShift Prometheus 监控
- 支持设置更多 SMBIOS 字段
- 改进了 containerDisk 的内存占用与速度
- 修复 CRI-O 内存限制
- 从 launcher 中移除 spc_t
- 为安全敏感功能添加功能门控

## v0.20.8 {#v0208}

发布于：Thu Oct 3 12:03:40 2019 +0200

## v0.20.7 {#v0207}

发布于：Fri Sep 27 15:21:56 2019 +0200

## v0.20.6 {#v0206}

发布于：Wed Sep 11 06:09:47 2019 -0400

## v0.20.5 {#v0205}

发布于：Thu Sep 5 17:48:59 2019 +0200

## v0.20.4 {#v0204}

发布于：Mon Sep 2 18:55:35 2019 +0200

## v0.20.3 {#v0203}

发布于：Tue Aug 27 16:58:15 2019 +0200

## v0.20.2 {#v0202}

发布于：Tue Aug 20 15:51:07 2019 +0200

## v0.20.1 {#v0201}

发布于：Fri Aug 9 19:48:17 2019 +0200

- 容器磁盘现在是安全的，并且不再在每次启动时被复制。旧的容器磁盘仍可以同样安全的方式使用，但新的容器磁盘不能在较旧的 KubeVirt 版本上使用
- 在 OKD 上创建专用的 SecurityContextConstraints，替换 privileged SCC 的用法
- 为以 PVC 为源的 DataVolume 添加了克隆授权检查
- sidecar 功能现在受功能门控保护
- KubeVirt 部署改用容器镜像 SHA 校验和，替换镜像标签
- 使用 MinAvailable=1 的 PodDisruptionBudget，保护控制平面组件免受自愿驱逐
- 将硬编码的 `virtctl` 替换为调用时的基本名称，这样通过 krew 插件包管理器安装时输出会更美观
- 在测试和示例的所有 Fedora VM 中添加了 RNG 设备（较新的内核在等待熵时可能会阻塞 bootimg）
- 如果指定了内存限制且未指定客户机内存，现在会将虚拟内存设置为与内存限制一致
- 支持 CoreOS 上的 nftable
- 为 virtctl image-upload 命令添加了 block-volume 标志
- 改进了 virtctl console/vnc 的数据流
- 移除了 DataVolumes 功能门控，改为自动检测 CDI 支持
- 移除了 SR-IOV 功能门控，现在默认启用
- 与 VMI 相关的指标已从 `kubevirt_vm_` 重命名为 `kubevirt_vmi_`，以更好地反映其用途
- 添加了用于报告 VMI 数量的指标
- 通过添加 CSV 生成器工具并修改 KubeVirt CR 状况，改进了与 HCO 的集成
- CI 改进：
  - 添加了专用的 SR-IOV 测试通道
  - 改进了日志收集
  - 减少了不稳定测试的数量

## v0.20.0 {#v0200}

发布于：Fri Aug 9 16:42:41 2019 +0200

- 容器磁盘现在是安全的，并且不再在每次启动时被复制。旧的容器磁盘仍可以同样安全的方式使用，但新的容器磁盘不能在较旧的 KubeVirt 版本上使用
- 在 OKD 上创建专用的 SecurityContextConstraints，替换 privileged SCC 的用法
- 为以 PVC 为源的 DataVolume 添加了克隆授权检查
- sidecar 功能现在受功能门控保护
- KubeVirt 部署改用容器镜像 SHA 校验和，替换镜像标签
- 使用 MinAvailable=1 的 PodDisruptionBudget，保护控制平面组件免受自愿驱逐
- 将硬编码的 `virtctl` 替换为调用时的基本名称，这样通过 krew 插件包管理器安装时输出会更美观
- 在测试和示例的所有 Fedora VM 中添加了 RNG 设备（较新的内核在等待熵时可能会阻塞 boot img）
- 如果指定了内存限制且未指定客户机内存，现在会将虚拟内存设置为与内存限制一致
- 支持 CoreOS 上的 nftable
- 为 virtctl image-upload 命令添加了 block-volume 标志
- 改进了 virtctl console/vnc 的数据流
- 移除了 DataVolumes 功能门控，改为自动检测 CDI 支持
- 移除了 SR-IOV 功能门控，现在默认启用
- 与 VMI 相关的指标已从 `kubevirt_vm_` 重命名为 `kubevirt_vmi_`，以更好地反映其用途
- 添加了用于报告 VMI 数量的指标
- 通过添加 CSV 生成器工具并修改 KubeVirt CR 状况，改进了与 HCO 的集成
- CI 改进：
  - 添加了专用的 SR-IOV 测试通道
  - 改进了日志收集
  - 减少了不稳定测试的数量

## v0.19.0 {#v0190}

发布于：Fri Jul 5 12:52:16 2019 +0200

- 修复在 kind 上运行时的问题
- 修复子资源 RBAC
- 限制 Pod 网络接口绑定
- 多个方面的许多额外缺陷修复
- 增加了针对更新、磁盘类型以及使用 NFS 进行实时迁移的测试用例
- 增加了内存超分、块存储、CPU Manager 和无头模式的测试用例
- HyperV 相关改进
- 改进了 runStrategies 的错误处理
- 改进了更新流程
- 改进了网络指标上报（数据包和错误）
- 改进了客户机开销计算
- 改进了 SR-IOV 测试套件
- 支持实时迁移自动收敛
- 支持 config-drive 磁盘
- 支持为容器磁盘设置 pullPolicy
- 支持在使用 SR-IOV 时运行非特权 VM
- 引入项目安全策略

## v0.18.1 {#v0181}

发布于：Thu Jun 13 12:00:56 2019 +0200

## v0.18.0 {#v0180}

发布于：Wed Jun 5 22:25:09 2019 +0200

- 构建：使用 Go modules
- CI：支持 Kubernetes 1.13
- 大量测试用例的修复与新增
- 若干较小的缺陷修复
- 改进了升级文档

## v0.17.4 {#v0174}

发布于：Tue Jun 25 07:49:12 2019 -0400

## v0.17.3 {#v0173}

发布于：Wed Jun 19 12:00:45 2019 -0400

## v0.17.2 {#v0172}

发布于：Wed Jun 5 08:12:04 2019 -0400

## v0.17.1 {#v0171}

发布于：Tue Jun 4 14:41:10 2019 -0400

## v0.17.0 {#v0170}

发布于：Mon May 6 16:18:01 2019 +0200

- 新增若干测试用例
- 改进了 virt-controller 的节点分布
- 改进了版本间迁移的支持
- 支持可配置的默认 MachineType
- 支持在节点带有污点时对 VM 进行实时迁移
- 支持 VM 交换区指标
- 支持带版本的 virt-launcher / virt-handler 通信
- 支持 HyperV 标志
- 支持不同的 VM 运行策略（例如 manual 和 rerunOnFailure）
- 若干实时迁移修复（TLS 支持、受保护的 Pod）

## v0.16.3 {#v0163}

发布于：Thu May 2 23:51:08 2019 +0200

## v0.16.2 {#v0162}

发布于：Fri Apr 26 12:24:33 2019 +0200

## v0.16.1 {#v0161}

发布于：Tue Apr 23 19:31:19 2019 +0200

## v0.16.0 {#v0160}

发布于：Fri Apr 5 23:18:22 2019 +0200

- Bazel 修复
- 支持升级的初步工作（尚未完成）
- 对 HyperV 功能的初步支持
- 支持将 MAC 地址传播到 Multus
- 支持取消实时迁移
- 支持平板输入设备
- 支持生成 OLM 元数据
- 支持在节点污点上触发 VM 实时迁移

## v0.15.0 {#v0150}

发布于：Tue Mar 5 10:35:08 2019 +0100

- CI：若干修复
- 修复可配置的 KVM 设备数量
- 收紧 virt-handler 的权限
- 开发构建改用 Bazel
- 支持使用共享磁盘和非共享磁盘进行实时迁移
- 支持跟踪实时迁移进度
- 支持 EFI 启动
- 支持 libvirt 5.0
- 支持额外的 DHCP 选项
- 支持用于处理 cloud-init 元数据的 hook
- 支持设置 VM 序列号
- 支持暴露基础设施指标和 VM 指标
- 支持 tablet 输入设备
- 支持额外的 CPU 标志
- 支持 Ignition 元数据
- 支持设置默认 CPU 型号
- 更新到 Go 1.11.5

## v0.14.0 {#v0140}

发布于：Mon Feb 4 22:04:14 2019 +0100

- CI：若干稳定性修复
- 文档：为 KubeVirt Razor 编写文档
- 构建：更新 Go
- 更新到 Kubernetes 1.12
- 更新 CDI
- 支持 Ready 和 Created 这两项 Operator 状况
- 支持（基本的）EFI
- 支持生成 cloud-init 的 network-config

## v0.13.7 {#v0137}

发布于：Mon Oct 28 17:02:35 2019 -0400

## v0.13.6 {#v0136}

发布于：Wed Sep 25 17:19:44 2019 +0200

## v0.13.5 {#v0135}

发布于：Thu Aug 1 11:25:00 2019 -0400

## v0.13.4 {#v0134}

发布于：Thu Aug 1 09:52:35 2019 -0400

## v0.13.3 {#v0133}

发布于：Mon Feb 4 15:46:48 2019 -0500

## v0.13.2 {#v0132}

发布于：Thu Jan 24 23:24:06 2019 +0100

## v0.13.1 {#v0131}

发布于：Thu Jan 24 11:16:20 2019 +0100

## v0.13.0 {#v0130}

发布于：Tue Jan 15 08:26:25 2019 +0100

- CI：修复 virt-api 竞态
- API：从 disks 中移除 volumeName

## v0.12.0 {#v0120}

发布于：Fri Jan 11 22:22:02 2019 +0100

- 引入 KubeVirt Operator，用于 KubeVirt 的生命周期管理
- 引入专用的 kubevirt 命名空间
- 支持 VMI 就绪状况
- 支持 vCPU 线程和插槽
- 支持 VMIR 的 scale 和 HPA
- 支持传递与 NTP 相关的 DHCP 选项
- 支持通过 QEMU 客户机代理上报客户机 IP 地址
- 支持使用共享存储进行实时迁移
- 支持基于 CPU 系列调度 VM
- 支持 masquerade 网络接口绑定

## v0.11.1 {#v0111}

发布于：Thu Dec 13 10:21:56 2018 +0200

## v0.11.0 {#v0110}

发布于：Thu Dec 6 10:15:51 2018 +0100

- API：registryDisk 已重命名为 containerDisk
- CI：使用 OKD 3.11
- 修复：如果 PVC 容量小于预期，则予以容忍
- 统一改用 ownerReferences
- 更新到 libvirt-4.10.0
- 支持在 Mac OSX 上使用 VNC
- 支持网络 SR-IOV 接口
- 支持自定义 DHCP 选项
- 支持通过自定义端点重启 VM
- 支持存活探针和就绪探针

## v0.10.0 {#v0100}

发布于：Thu Nov 8 15:21:34 2018 +0100

- 支持 vhost-net
- 支持块设备多队列
- 支持为 virtio 设备设置自定义 PCI 地址
- 支持将 KubeVirt 部署到自定义命名空间
- 支持 ServiceAccount 令牌磁盘
- 支持由 Multus 提供的网络
- 支持由 Genie 提供的网络
- 支持由 Kuryr 提供的网络
- 支持块 PV
- 支持可配置的磁盘设备缓存
- 支持绑核的 IO 线程
- 支持 virtio 网络多队列
- 支持镜像上传（依赖于 CDI）
- 支持包含更多 VM 详情的自定义实体列表（自定义列）
- 支持上报所有 vNIC 的 IP 和 MAC 地址
- 对客户机代理状态上报的基本支持
- 更结构化的日志
- 更好的 libvirt 错误报告
- 更严格的 CR 校验
- 更好的所有者引用
- 若干测试改进

## v0.9.6 {#v096}

发布于：Thu Nov 22 17:14:18 2018 +0100

## v0.9.5 {#v095}

发布于：Thu Nov 8 09:57:48 2018 +0100

## v0.9.4 {#v094}

发布于：Wed Nov 7 08:22:14 2018 -0500

## v0.9.3 {#v093}

发布于：Mon Oct 22 09:04:02 2018 -0400

## v0.9.2 {#v092}

发布于：Thu Oct 18 12:14:09 2018 +0200

## v0.9.1 {#v091}

发布于：Fri Oct 5 09:01:51 2018 +0200

## v0.9.0 {#v090}

发布于：Thu Oct 4 14:42:28 2018 +0200

- CI：NetworkPolicy 测试
- CI：支持外部 provider（使用预先配置的集群进行测试）
- 修复 virtctl console 在 CRI-O 下的问题
- 支持初始化空的 PV
- 支持基本的 CPU 绑核
- 支持设置 IO 线程
- 支持块卷
- 将预设逻辑移到 mutating webhook
- 引入使用 Prometheus 指标的基本指标上报
- 多处的大量稳定性修复

## v0.8.0 {#v080}

发布于：Thu Sep 6 14:25:22 2018 +0200

- 支持 DataVolume
- 支持 Web 浏览器终端所用的子协议
- 支持 virtio-rng
- 支持断开连接的 VM
- 支持设置宿主机型号
- 支持宿主机 CPU 直通
- 支持设置 vNIC 的 MAC 和 PCI 地址
- 支持内存超分
- 支持从网络设备启动
- 默认使用更少的设备，即禁用未使用的设备
- 改进了 VMI 关机状态
- 增加日志以改善可调试性
- 大量小修复，包括错字和文档修复
- 测试中的竞态检测
- hook 改进
- 更新为使用 Fedora 28（包括 libvirt 和 QEMU 等依赖的更新）
- 调整 CI 以支持 Kubernetes 1.11

## v0.7.0 {#v070}

发布于：Wed Jul 4 17:41:33 2018 +0200

- CI：将测试存储移到 hostPath
- CI：添加对 Kubernetes 1.10.4 的支持
- CI：改进了多接口的网络测试
- CI：移除对 Origin 3.9 的支持
- CI：添加在 Origin 上测试模板的测试
- 将 VM 重命名为 VMI
- VM 亲和性与反亲和性
- 增加对多个网络的感知
- 添加大页支持
- 添加基于 device-plugin 的 KVM
- 添加设置网络接口型号的支持
- 添加（基本且初步的）与 Kubernetes 兼容的联网方式（SLIRP）
- 为我们的角色添加角色聚合
- 添加设置磁盘序列号的支持
- 添加指定 CPU 型号的支持
- 添加设置网络接口 MAC 地址的支持
- 重新放置二进制文件以符合 FHS
- 日志改进
- 模板修复
- 修复 OpenShift CRD 校验
- virtctl：改进 vnc 日志
- virtctl：添加 expose
- virtctl：使用 PATCH 替换 PUT

## v0.6.4 {#v064}

发布于：Tue Aug 21 17:29:28 2018 +0300

## v0.6.3 {#v063}

发布于：Mon Jul 30 16:14:22 2018 +0200

## v0.6.2 {#v062}

发布于：Wed Jul 4 17:49:37 2018 +0200

- 为打包重新放置二进制文件
- QEMU 进程检测
- 角色聚合
- CPU 型号选择
- VM 重命名修复

## v0.6.1 {#v061}

发布于：Mon Jun 18 17:07:48 2018 -0400

## v0.6.0 {#v060}

发布于：Mon Jun 11 09:30:28 2018 +0200

- 一系列用于降低不稳定性的测试修复
- Vagrant 安装方式已被弃用
- 更新了 Docker 和 CentOS 版本
- 将 Kubernetes 1.10.3 加入测试矩阵
- 若干 ginkgo 并发修复
- 若干拼写修复
- 一系列基础设施更新
- 尽可能使用 /dev/kvm，否则回退到仿真
- 添加默认的 view/edit/admin RBAC 角色
- 网络 MTU 修复
- CD-ROM 驱动器现在为只读
- 现在可以在 VM 上正确引用 Secret
- 添加磁盘启动顺序
- 添加 virtctl version
- 添加 virtctl expose
- 修复虚拟机内存计算
- 添加基本的虚拟机网络 API

## v0.5.0 {#v050}

发布于：Fri May 4 18:25:32 2018 +0200

- 更好的控制器健康状态信号
- 更好的 virtctl 错误信息
- 为启用 CRI-O 支持所做的改进
- 在稳定版 OpenShift 上运行 CI
- 增加对多个 PVC 的测试覆盖
- 改进了控制器生命周期保证
- 添加 Webhook 校验
- 增加节点驱逐的测试覆盖
- OfflineVirtualMachine 状态改进
- RegistryDisk API 更新

## v0.4.1 {#v041}

发布于：Thu Apr 12 11:46:09 2018 +0200

- VM 关机修复与测试
- CRD 校验的功能测试
- Windows VM 测试
- DHCP 链路本地地址变更

## v0.4.0 {#v040}

发布于：Fri Apr 6 16:40:31 2018 +0200

- 修复若干网络问题
- 向 CI 添加并启用 OpenShift 支持
- 添加有条件的 Windows 测试（如果存在镜像）
- 添加用于控制台访问的子资源
- virtctl 配置与 kubectl 对齐
- 修复 API 参考文档的生成
- 为 OfflineVirtualMachine 提供稳定的 UUID
- 为 MacOS 和 Windows 构建 virtctl
- 将默认架构设置为 x86_64
- CI 基础设施的重大改进（全部容器化）
- 用于启动和停止 VM 的 virtctl 便捷功能

## v0.3.0 {#v030}

发布于：Thu Mar 8 10:21:57 2018 +0100

- 与 Kubernetes 兼容的网络
- 与 Kubernetes 兼容的、基于 PV 的存储
- 支持 VirtualMachinePresets
- 支持 OfflineVirtualMachine
- RBAC 改进
- 默认切换到 q35 机器类型
- 大量测试与 CI 修复
- 支持临时磁盘

## v0.2.0 {#v020}

发布于：Fri Jan 5 16:30:45 2018 +0100

- VM 启动与关机流程改进
- VirtualMachine API 重新设计
- 移除 HAProxy
- 重新设计 VNC/控制台访问
- 对不同 Vagrant provider 的初步支持

## v0.1.0 {#v010}

发布于：Fri Dec 8 20:43:06 2017 +0100

- 为提供正规的 OpenAPI 参考而做的多项 API 改进
- 添加 watchdog 支持
- 大幅改进非 Vagrant 环境上的部署
  - 移除了 nodeSelectors
  - 将内部组件部署与边缘组件部署分开
  - 为开发、测试和发布部署分别创建了清单
- 将组件移到 kube-system 命名空间
- 改进并统一了标志解析

## v0.0.4 {#v004}

发布于：Tue Nov 7 11:51:45 2017 +0100

- 为 VM.Spec 添加节点亲和性支持
- 添加 OpenAPI 规范
- 移除 Swagger 1.2 规范
- virt-launcher 重构
- virt-controller 的领导者选举机制
- 依赖管理从 glide 改为 dep
- 改进 virt-handler 同步循环
- 添加在 oVirt 基础设施上运行功能测试的支持
- 若干测试修复（spice、清理等）
- 添加控制台测试工具
- 改进 libvirt 事件通知

## v0.0.3 {#v003}

发布于：Fri Oct 6 10:21:16 2017 +0200

- 容器化二进制构建
- 基于套接字的容器检测
- 支持 cloud-init
- 支持基于容器的临时磁盘
- 基本 RBAC 配置
- client-go 更新
- 将 VM 重命名为 VirtualMachine
- 引入 VirtualMachineReplicaSet
- 改进了迁移事件
- 改进了 API 文档

## v0.0.2 {#v002}

发布于：Mon Sep 4 21:12:46 2017 +0200

- 使用 CRD
- 将 libvirt 移到 Pod 中
- 引入 `virtctl`
- 使用 glide 替换 govendor
- 基于容器的临时磁盘
- 贡献指南改进
- 支持 Kubernetes 命名空间
