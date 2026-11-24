# Windows virtio 驱动 {#windows-virtio-drivers}

本文档旨在说明如何为在完全虚拟化客户机中运行的 Microsoft Windows 安装 virtio 驱动。

## 我需要 virtio 驱动吗？ {#do-i-need-virtio-drivers}

需要。没有 virtio 驱动，就无法正确使用半虚拟化硬件。它要么无法工作，要么会有严重的性能损失。

有关 VirtIO 和半虚拟化的更多信息，见 [VirtIO 与半虚拟化](https://wiki.libvirt.org/page/Virtio)

有关配置 VirtIO 驱动的更多细节，请参阅[在新的 Windows 虚拟机上安装 VirtIO 驱动](https://docs.openshift.com/container-platform/4.10/virt/virtual_machines/virt-installing-virtio-drivers-on-new-windows-vm.html)和[在现有 Windows 虚拟机上安装 VirtIO 驱动](https://docs.openshift.com/container-platform/4.10/virt/virtual_machines/virt-installing-virtio-drivers-on-existing-windows-vm.html)。


## 我需要安装哪些驱动？ {#which-drivers-i-need-to-install}

要让 Windows 在虚拟化环境中顺畅运行，通常最多可能需要 8 种设备。KubeVirt 目前只支持：

- **viostor**，块设备驱动，适用于“其他设备”组中的 SCSI 控制器。

- **viorng**，熵源驱动，适用于“其他设备”组中的 PCI 设备。

- **NetKVM**，网络驱动，适用于“其他设备”组中的以太网控制器。仅在配置了 virtio NIC 时可用。

其他已经存在、未来可能得到支持的 virtio 驱动：

- Balloon，balloon 驱动，适用于“其他设备”组中的 PCI 设备

- vioserial，半虚拟串行驱动，适用于“其他设备”组中的 PCI 简单通信控制器。

- vioscsi，SCSI 块设备驱动，适用于“其他设备”组中的 SCSI 控制器。

- qemupciserial，仿真 PCI 串行驱动，适用于“其他设备”组中的 PCI 串行端口。

- qxl，半虚拟视频驱动，应用于“显示适配器”组中的 Microsoft 基本显示适配器。

- pvpanic，半虚拟 panic 驱动，适用于“其他设备”组中的未知设备。

> **注意**
>
> 有些驱动在安装阶段就是必需的。当你把 Windows 安装到 virtio 块存储上时，必须提供合适的 virtio 驱动。具体来说，请为你的 Microsoft Windows 版本选择 viostor 驱动，例如运行 Windows 10 时不要安装 XP 驱动。
>
> 其他驱动可以在 Windows 成功安装之后再安装。同样，请只安装与你的 Windows 版本匹配的驱动。

### 如何在 Windows 安装过程中安装？ {#how-to-install-during-windows-install}

要在 Windows 开始其安装之前安装驱动，请确保已将 virtio-win 软件包作为 SATA CD-ROM 挂接到你的 VirtualMachine。在 Windows 安装程序中，选择高级安装并加载驱动。然后请转到已加载的 Virtio CD-ROM，并根据你所设置的类型安装 viostor 或 vioscsi 之一。

分步截图：

![Windows 安装](../assets/virtio_custom_install_0.png)

![自定义安装](../assets/virtio_custom_install_1.png)

![磁盘安装](../assets/virtio_custom_install_2.png)

![选择介质](../assets/virtio_custom_install_3.png)

![选择驱动](../assets/virtio_custom_install_4.png)

![继续安装](../assets/virtio_custom_install_5.png)

### 如何在 Windows 安装之后安装？ {#how-to-install-after-windows-install}

Windows 安装之后，请转到[设备管理器](https://support.microsoft.com/en-us/help/4026149/windows-open-device-manager)。在那里，你应当能在“可用设备”部分看到未被检测到的设备。你可以遍历此列表，逐个安装 virtio 驱动。

![未知设备](../assets/virtio_driver_install_0.png)

![按 id 选择驱动](../assets/virtio_driver_install_1.png)

![安装驱动](../assets/virtio_driver_install_2.png)

![驱动已安装](../assets/virtio_driver_install_3.png)

有关如何选择合适的驱动以及如何安装驱动的更多细节，请参阅 [Red Hat Enterprise Linux 7 上的 Windows 客户机虚拟机](https://access.redhat.com/articles/2470791)。

## 如何获取 virtio 驱动？ {#how-to-obtain-virtio-drivers}

virtio Windows 驱动以 [containerDisk](../storage/disks_and_volumes.md#containerdisk) 的形式分发，可以简单地挂载到 VirtualMachine。包含该磁盘的容器镜像位于：<https://quay.io/repository/kubevirt/virtio-container-disk?tab=tags>，该镜像可以像任何其他 docker 容器一样拉取：

    docker pull quay.io/kubevirt/virtio-container-disk

不过，并不需要手动拉取镜像；部署 VirtualMachine 时，如果镜像不存在，Kubernetes 会下载它。

## 挂接到 VirtualMachine {#attaching-to-virtualmachine}

KubeVirt 以容器磁盘的形式分发适用于 Microsoft Windows 的 virtio 驱动。该软件包包含 virtio 驱动和 QEMU 客户机代理。该磁盘已在 Microsoft Windows Server 2012 上经过测试。支持的 Windows 版本为 XP 及以上。

该软件包旨在用作挂接到装有 Microsoft Windows 的虚拟机上的 CD-ROM。它可以在安装阶段用作 SATA CDROM，也可以为现有的 Windows 安装提供驱动。

只需向你的 VirtualMachine 添加 ContainerDisk，即可挂接 virtio-win 软件包。

    spec:
      domain:
        devices:
          disks:
            - name: virtiocontainerdisk
              # Any other disk you want to use, must go before virtioContainerDisk.
              # KubeVirt boots from disks in order ther are defined.
              # Therefore virtioContainerDisk, must be after bootable disk.
              # Other option is to choose boot order explicitly:
              #  - https://kubevirt.io/api-reference/v0.13.2/definitions.html#_v1_disk
              # NOTE: You either specify bootOrder explicitely or sort the items in
              #       disks. You can not do both at the same time.
              # bootOrder: 2
              cdrom:
                bus: sata
    volumes:
      - containerDisk:
          image: quay.io/kubevirt/virtio-container-disk
        name: virtiocontainerdisk

安装完 virtio 驱动之后，只需从 yaml 规范中移除该磁盘并重启 VirtualMachine，即可移除 virtio 容器磁盘。
