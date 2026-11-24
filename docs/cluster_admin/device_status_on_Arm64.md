# Arm64 上的设备状态 {#device-status-on-arm64}

本页基于 https://github.com/kubevirt/kubevirt/issues/8916

设备 | 描述 | Arm64 上的状态
-- | -- | --
DisableHotplug |   | 受支持
磁盘 | sata/ virtio 总线 | 支持 virtio 总线
看门狗 | i6300esb | 不受支持
UseVirtioTransitional | virtio-transitional | 受支持
接口 | e1000/ virtio-net-device | 支持 virtio-net-device
输入 | tablet virtio/usb 总线 | 受支持
AutoattachPodInterface | 连接到 /net/tun（devices.kubevirt.io/tun） | 受支持
AutoattachGraphicsDevice | 创建 virtio-gpu 设备 / vga 设备 | 支持 virtio-gpu
AutoattachMemBalloon | virtio-balloon-pci-non-transitional | 受支持
AutoattachInputDevice | 自动添加 tablet | 受支持
Rng | virtio-rng-pci-non-transitional 宿主机:/dev/urandom | 受支持
BlockMultiQueue | "driver":"virtio-blk-pci-non-transitional","num-queues":$cpu_number | 受支持
NetworkInterfaceMultiQueue | -netdev tap,fds=21:23:24:25,vhost=on,vhostfds=26:27:28:29,id=hostua-default#fd 数量等于队列数量 | 受支持
GPUs |   | 未验证
文件系统 | virtiofs、vhost-user-fs-pci，需要启用功能门控：ExperimentalVirtiofsSupport | 受支持
ClientPassthrough | https://www.linaro.org/blog/kvm-pciemsi-passthrough-armarm64/在 x86_64 上，需要启用 iommu | 未验证
声音 | ich9/ ac97 | 不受支持
TPM | tpm-tis-devicehttps://qemu.readthedocs.io/en/latest/specs/tpm.html | 受支持
Sriov | vfio-pci | 未验证
