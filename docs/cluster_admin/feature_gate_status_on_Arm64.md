# Arm64 上的功能门控状态 {#feature-gate-status-on-arm64}

本页基于 https://github.com/kubevirt/kubevirt/issues/9749
它记录 Arm64 平台上的功能门控状态。以下是状态说明：

- **受支持**：该功能门控在 Arm64 平台上受支持。
- **尚不支持**：该功能门控的某些依赖尚不支持 Arm64，因此此功能目前不受支持。我们将来可能会支持这些依赖。
- **不支持**：该功能门控在 Arm64 上不受支持。
- **未验证**：该功能尚未经过验证。


功能门控 | 状态 | 备注
-- | -- | --
CPUManager | 受支持 | 使用 taskset 进行 CPU 绑定，不支持 kvm-hint-dedicated（这仅在 x86 平台上有效）
NUMAFeatureGate | 尚不支持 | 需要在 Arm64 上支持大页
IgnitionGate | 受支持 | 此功能仅用于 CoreOS/RhCOS
LiveMigrationGate | 受支持 | 已使用 masquerade 网络验证实时迁移
SRIOVLiveMigrationGate | 未验证 | 需要两台相同的机器和 SRIOV 设备
HypervStrictCheckGate | 不支持 | Hyperv 在 Arm64 上无法工作
SidecarGate | 受支持 |  
GPUGate | 未验证 | 需要 GPU 设备
HostDevicesGate | 未验证 | 需要 GPU 或声卡
SnapshotGate | 受支持 | 需要 snapshotter 支持 https://github.com/kubernetes-csi/external-snapshotter
VMExportGate | 部分受支持 | 需要 snapshotter 支持 https://kubevirt.io/user-guide/operations/export_api/，支持导出 pvc，不支持导出依赖 CDI 的 DataVolume 和 MemoryDump
HotplugVolumesGate | 尚不支持 | 依赖 datavolume 和 CDI
HostDiskGate | 受支持 |  
VirtIOFSGate | 受支持 |  
MacvtapGate | 尚不支持 | quay.io/kubevirt/macvtap-cni 不支持 Arm64，https://github.com/kubevirt/macvtap-cni#deployment
PasstGate | 受支持 | VM 与 Pod 具有相同的 IP；为网络启动进程 /usr/bin/passt --runas 107 -e -t 8080
DownwardMetricsFeatureGate | 需要更多信息 | 它用于让客户机获取宿主机信息，在 Arm64 和 x86_64 上均失败。<br><br>该块已成功挂载，可以看到以下信息：<br>  `-blockdev {"driver":"file","filename":"/var/run/kubevirt-private/downwardapi-disks/vhostmd0","node-name":"libvirt-1-storage","cache":{"direct":true,"no-flush":false},"auto-read-only":true,"discard":"unmap"}`<br><br>但无法通过 `vm-dump-metrics` 获取信息：<br><br>`LIBMETRICS: read_mdisk(): Unable to read metrics disk`<br>`LIBMETRICS: get_virtio_metrics(): Unable to export metrics: open(/dev/virtio-ports/org.github.vhostmd.1) No such file or directory`<br>`LIBMETRICS: get_virtio_metrics(): Unable to read metrics`
NonRootDeprecated | 受支持 |  
NonRoot | 受支持 |  
Root | 受支持 |  
ClusterProfiler | 受支持 |
WorkloadEncryptionSEV | 不支持 | SEV 仅在 x86_64 上可用
VSOCKGate | 受支持 |  
HotplugNetworkIfacesGate | 尚不支持 | 需要设置 *multus-cni* 和 *multus-dynamic-networks-controller*：https://github.com/k8snetworkplumbingwg/multus-cni <br>`cat ./deployments/multus-daemonset-thick.yml \| kubectl apply -f -`https://github.com/k8snetworkplumbingwg/multus-dynamic-networks-controller <br>`kubectl apply -f manifests/dynamic-networks-controller.yaml` <br><br>目前，镜像 ghcr.io/k8snetworkplumbingwg/multus-cni:snapshot-thick 不支持 Arm64 服务器。更多信息请参阅 https://github.com/k8snetworkplumbingwg/multus-cni/pull/1027。
CommonInstancetypesDeploymentGate | 尚不支持 | common-instancetypes instancetypes 的支持需要测试，面向 ARM 工作负载的 common-instancetypes 偏好仍然缺失
