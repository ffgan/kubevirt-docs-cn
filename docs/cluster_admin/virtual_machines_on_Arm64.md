# Arm64 上的虚拟机 {#virtual-machines-on-arm64}

本页汇总 Arm64 平台上所有不受支持的虚拟机配置以及不同的默认设置。

## 虚拟硬件 {#virtual-hardware}

### 机器类型 {#machine-type}

目前，我们只支持一种机器类型 `virt`，并且默认即使用该类型。

### BIOS/UEFI {#biosuefi}

在 Arm64 平台上，我们只支持默认设置的 UEFI 引导。不支持 UEFI 安全启动。

### CPU {#cpu}

#### Node-labeller {#node-labeller}

目前，Node-labeller 在 Arm64 平台上部分受支持。它尚不支持解析 virsh_domcapabilities.xml 和 capabilities.xml，以及提取 CPU 特性等相关信息。

#### 型号 {#model}

`host-passthrough` 是 Arm64 上唯一受支持的型号。在 Arm64 平台上默认设置该 CPU 型号。

### 时钟 {#clock}

`kvm` 和 `hyperv` 计时器在 Arm64 平台上不受支持。

### 视频与图形设备 {#video-and-graphics-device}

我们不支持 vga 设备，默认使用 virtio-gpu。

### 大页 {#hugepages}

大页在 Arm64 平台上不受支持。

### 资源请求与限制 {#resources-requests-and-limits}

Arm64 平台支持 CPU 绑定。

## NUMA {#numa}

由于大页是 NUMA 功能的前提条件，而 Arm64 平台未启用大页，因此 NUMA 功能在 Arm64 上无法工作。

## 磁盘与卷 {#disks-and-volumes}

Arm64 只支持 virtio 和 scsi 磁盘总线类型。

## 接口与网络 {#interface-and-networks}

### macvlan {#macvlan}

我们不支持 `macvlan` 网络，因为项目 https://github.com/kubevirt/macvtap-cni 不支持 Arm64。

### SRIOV {#sriov}

此类设备尚未在 Arm64 平台上验证。

## 存活探针与就绪探针 {#liveness-and-readiness-probes}

`Watchdog` 设备在 Arm64 平台上不受支持。
