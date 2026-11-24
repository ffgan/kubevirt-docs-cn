# 主机设备分配 {#host-devices-assignment}

KubeVirt 提供了一种把主机设备分配给虚拟机的机制。
该机制是通用的，可以分配各种类型的 PCI 设备，例如加速器（包括 GPU 和 vGPU），或挂接在 PCI 总线上的任何其他设备。它还允许使用同一机制分配 [Linux 中介设备](https://www.kernel.org/doc/html/latest/driver-api/vfio-mediated-device.html)，例如预先配置好的虚拟 GPU。


## PCI 直通的宿主机准备 {#host-preparation-for-pci-passthrough}

 * 主机设备直通要求在 BIOS 中启用虚拟化扩展和 IOMMU 扩展（Intel VT-d 或 AMD IOMMU）。

 * 要启用 IOMMU，需根据 CPU 类型，让宿主机带上额外的内核参数启动：Intel 使用 `intel_iommu=on`，AMD 使用 `amd_iommu=on`。

将这些参数追加到 grub 配置文件中 GRUB_CMDLINE_LINUX 行的末尾。

```
# vi /etc/default/grub
...
GRUB_CMDLINE_LINUX="nofb splash=quiet console=tty0 ... intel_iommu=on
...

# grub2-mkconfig -o /boot/grub2/grub.cfg

# reboot
```

 * 应在宿主机上启用 vfio-pci 内核模块。
```
# modprobe vfio-pci
```

## 为直通准备 PCI 设备 {#preparation-of-pci-devices-for-passthrough}

目前，KubeVirt 只能分配使用 `vfio-pci` 驱动的 PCI 设备，[SR-IOV vGPU](#preparation-of-vgpu-devices-with-sr-iov) 除外。要为设备分配准备特定设备，应先将其从原来的驱动解绑，再绑定到 `vfio-pci` 驱动。

 * 查找所需设备的 PCI 地址：

```
$ lspci -DD|grep NVIDIA
0000.65:00.0 3D controller [0302]: NVIDIA Corporation TU104GL [Tesla T4] [10de:1eb8] (rev a1)
```

 * 将该设备绑定到 `vfio-pci` 驱动：
```
echo 0000:65:00.0 > /sys/bus/pci/drivers/nvidia/unbind
echo "vfio-pci" > /sys/bus/pci/devices/0000\:65\:00.0/driver_override
echo 0000:65:00.0 > /sys/bus/pci/drivers/vfio-pci/bind
```

## 准备 vGPU 等中介设备 {#preparation-of-mediated-devices-such-as-vgpu}

一般来说，中介设备（mdevs，例如 vGPU）的配置应按照厂商说明进行。
KubeVirt 现在可以协助在集群节点上创建中介设备 / vGPU。这假定所需的厂商驱动已经安装在节点上。
有关此功能的更多信息，请参阅[中介设备与虚拟 GPU](../compute/mediated_devices_configuration.md)。

一旦配置好 mdev，KubeVirt 就能够发现它，并将其用于设备分配。

## 使用 SR-IOV 准备 vGPU 设备 {#preparation-of-vgpu-devices-with-sr-iov}

一般来说，带 SR-IOV 的 vGPU 应按照厂商说明进行配置。
KubeVirt 不支持创建虚拟功能（VF），也不支持把 vGPU profile 分配给该功能。
KubeVirt 只是便于将这些 VF 公开出来供 VM 使用（通过通用 PCI 设备插件），但并不像 mdev 那样对各种 profile 做细粒度区分。因此，在 profile 异构的情况下，建议使用厂商的设备插件来公开这些功能。

要使用支持 SR-IOV 的 vGPU 设备，请遵循[列出允许的设备](#listing-permitted-devices)中与 PCI 相关的说明。

## 列出允许的设备 {#listing-permitted-devices}

管理员可以控制哪些主机设备被公开并允许在集群中使用。集群中允许的主机设备需要按其 `vendor:product` 选择器（针对 PCI 设备）或中介设备名称，在 KubeVirt CR 中列入允许列表。

```
configuration:
  permittedHostDevices:
    pciHostDevices:
    - pciVendorSelector: "10DE:1EB8"
      resourceName: "nvidia.com/TU104GL_Tesla_T4"
      externalResourceProvider: true
    - pciVendorSelector: "8086:6F54"
      resourceName: "intel.com/qat"
    mediatedDevices:
    - mdevNameSelector: "GRID T4-1Q"
      resourceName: "nvidia.com/GRID_T4-1Q"
```

 * `pciVendorSelector` 是形式为 `vendor_id:product_id` 的 PCI 厂商 ID 与产品 ID 元组。该元组可以标识宿主机上的特定类型设备。例如，上面所示的标识符 `10de:1eb8` 可以使用 `lspci` 找到。

        $ lspci -nnv|grep -i nvidia
        65:00.0 3D controller [0302]: NVIDIA Corporation TU104GL [Tesla T4] [10de:1eb8] (rev a1)

 * `mdevNameSelector` 是中介设备类型的名称，可以标识宿主机上的特定类型中介设备。

    可以通过检查
    `/sys/bus/pci/devices/SLOT:BUS:DOMAIN.FUNCTION/mdev_supported_types/TYPE/name`
    的内容，查看给定 PCI 设备支持哪些中介类型。
    例如，如果你的系统上有一块 NVIDIA T4 GPU，并且把上述路径名中的 `SLOT`、`BUS`、`DOMAIN` 和 `FUNCTION` 替换为适用于你的系统的值，你会看到 `TYPE` 为 `nvidia-226` 的类型包含选择器字符串 `GRID T4-2A`，该字符串位于其 `name` 文件中。

    将 `GRID T4-2A` 指定为 `mdevNameSelector` 后，KubeVirt 就可以把它与 `/sys/class/mdev_bus/SLOT:BUS:DOMAIN.FUNCTION/$mdevUUID/mdev_type/name` 进行匹配，从而针对某些 `SLOT:BUS:DOMAIN.FUNCTION` 和 `$mdevUUID` 值找到对应的中介设备。

 * 外部提供者：
`externalResourceProvider` 字段表示此资源由外部设备插件提供。在这种情况下，KubeVirt 只会允许在集群中使用此设备，而把分配和监控留给外部设备插件。

## 谁可以分配已允许的设备 {#who-can-allocate-a-permitted-device}

对于 `permittedHostDevices` 中每一条未设置 `externalResourceProvider` 的条目，virt-handler 会注册设备插件，并且 ValidatingAdmissionPolicy 仅当调用者是 virt-controller 服务账户时，才准入请求该资源名称的 Pod。虚拟机在 virt-controller 所创建的 virt-launcher Pod 上获得该设备。

该策略读取 KubeVirt 自定义资源。当该对象缺失时，准入会允许该 Pod，以免在整个集群范围内使 Pod 创建失败，因此该限制仅在 KubeVirt CR 存在期间有效。

升级之后，请求 `permittedHostDevices` 中所列资源的非 KubeVirt Pod 会被拒绝。当另一个工作负载必须继续直接分配该资源时，请在该条目上设置 `externalResourceProvider: true`。此后 KubeVirt 不会为该名称注册自己的插件，准入策略也不会限制它。


## 启动虚拟机 {#starting-a-virtual-machine}

可以通过 `gpus` 和 `hostDevices` 字段将主机设备分配给虚拟机。`deviceNames` 既可以引用 PCI 设备资源名称，也可以引用中介设备资源名称。

```yaml
kind: VirtualMachineInstance
spec:
  domain:
    devices:
      gpus:
      - deviceName: nvidia.com/TU104GL_Tesla_T4
        name: gpu1
      - deviceName: nvidia.com/GRID_T4-1Q
        name: gpu2
      hostDevices:
      - deviceName: intel.com/qat
        name: quickaccess1
```

## NVMe PCI 直通 {#nvme-pci-passthrough}

要对 NVMe 设备进行直通，过程与 GPU 的情况非常相似。该设备需要列在 `permittedHostDevice` 之下，并出现在 VM 声明的 `hostDevices` 中。

目前，KubeVirt 设备插件不允许用户通过指定地址来选择特定设备。因此，如果集群中存在多个厂商 ID 和产品 ID 都相同的 NVMe 设备，它们可能会被随机分配给某个 VM。如果这些设备不在同一节点上，则 nodeSelector 可以缓解该问题。

示例：

修改 `permittedHostDevice`

```yaml
configuration:
  permittedHostDevices:
    pciHostDevices:
    - pciVendorSelector: 8086:5845
      resourceName: devices.kubevirt.io/nvme
```

VMI 声明：
```yaml
kind: VirtualMachineInstance
metadata:
  labels:
    special: vmi-nvme
  name: vmi-nvme
spec:
  nodeSelector: 
    kubernetes.io/hostname: node03   # <--
  domain:
    devices:
      disks:
      - disk:
          bus: virtio
        name: containerdisk
      - disk:
          bus: virtio
        name: cloudinitdisk
      hostDevices:  # <--
      - name: nvme  # <--
        deviceName: devices.kubevirt.io/nvme  # <--
    resources:
      requests:
        memory: 1024M
  terminationGracePeriodSeconds: 0
  volumes:
  - containerDisk:
      image: registry:5000/kubevirt/fedora-with-test-tooling-container-disk:devel
    name: containerdisk
  - cloudInitNoCloud:
      userData: |-
        #cloud-config
        password: fedora
        chpasswd: { expire: False }
    name: cloudinitdisk
```

## USB 主机直通 {#usb-host-passthrough}

自 KubeVirt v1.1 起，我们可以把插入某个节点的 USB 设备提供给在同一节点上运行的 VM。

### 要求 {#requirements}

需要集群管理员权限才能编辑 KubeVirt CR，以便：

- 启用 `HostDevices` [功能门控](../cluster_admin/activating_feature_gates.md)
- 编辑 `permittedHostDevices` 配置，把节点 USB 设备公开给集群

### 公开 USB 设备 {#exposing-usb-devices}

要把 USB 设备分配给你的 VMI，需要在某个资源名称下将这些设备向集群公开。设备允许列表可以在 KubeVirt CR 的 `configuration.permittedHostDevices.usb` 下编辑。

在此示例中，我们将使用 `kubevirt.io/storage` 资源名称，对应设备为 `vendor:
"46f4"` and `product: "0001"` [^1]。


```yaml
spec:
  configuration:
    permittedHostDevices:
      usb:
        - resourceName: kubevirt.io/storage
          selectors:
            - vendor: "46f4"
              product: "0001"
```

在把 `usb` 配置添加到 KubeVirt CR 的 `permittedHostDevices` 之下以后，KubeVirt 的 device-plugin 会公开此资源名称，你可以在 VMI 中使用它。


### 把 USB 添加到你的 VM {#adding-usb-to-your-vm}

现在，在 VMI 配置中，可以添加 `devices.hostDevices.deviceName`，引用上一步提供的资源名称，并给它一个本地 `name`，例如：

```yaml
spec:
  domain:
    devices:
      hostDevices:
      - deviceName: kubevirt.io/storage
        name: usb-storage
```

可以在 [examples/vmi-usb.yaml](https://github.com/kubevirt/kubevirt/blob/main/examples/vmi-usb.yaml) 中找到一个可运行的示例，该示例使用 QEMU 模拟的 USB 存储。


### 一组 USB 设备 {#bundle-of-usb-devices}

你可能希望把不止一个 USB 设备重定向到某个 VMI，例如键盘、鼠标和智能卡设备。KubeVirt CR 支持在同一资源名称下分配多个 USB 设备，因此可以这样做：

```yaml
spec:
  configuration:
    permittedHostDevices:
      usb:
        - resourceName: kubevirt.io/peripherals
          selectors:
            - vendor: "045e"
              product: "07a5"
            - vendor: "062a"
              product: "4102"
            - vendor: "072f"
              product: "b100"
```

添加到 VMI 配置中：

```yaml
spec:
  domain:
    devices:
      hostDevices:
      - deviceName: kubevirt.io/peripherals
        name: local-peripherals 
```

请注意，所有 USB 设备都必须在位，分配才能生效。

[^1]: 请注意，`vendor:product` 的值可以用 `lsusb` 命令轻松找到。
