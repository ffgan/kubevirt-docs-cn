# Arm64 操作 {#arm64-operations}

本页汇总所有在 Arm64 上不受支持的操作。

## 热插拔网络接口 {#hotplug-network-interfaces}

热插拔网络接口在 Arm64 上不受支持，因为镜像 ghcr.io/k8snetworkplumbingwg/multus-cni:snapshot-thick 不支持 Arm64 平台。更多信息请参阅 https://github.com/k8snetworkplumbingwg/multus-cni/pull/1027。

## 大页支持 {#hugepages-support}

大页功能在 Arm64 上不受支持。大页机制在 X86_64 与 Arm64 之间有所不同。目前我们只在 4k 页大小的系统上验证 KubeVirt。

## 导出 API {#export-api}

导出 API 在 Arm64 平台上部分受支持。由于 CDI 尚不受支持，DataVolume 和 MemoryDump 的导出在 Arm64 上不受支持。

## 虚拟机内存转储 {#virtual-machine-memory-dump}

如上所述，MemoryDump 需要 CDI，目前在 Arm64 上尚不受支持。

## 中介设备与虚拟 GPU {#mediated-devices-and-virtual-gpus}

此项尚未在 Arm64 平台上验证。
