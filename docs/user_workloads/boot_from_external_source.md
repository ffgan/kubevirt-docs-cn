# 从外部源启动 {#booting-from-external-source}

安装新的客户机虚拟机 OS 时，直接从存放在宿主机物理机 OS 中的 kernel 和 initrd 启动通常很有用，这样可以把命令行参数直接传给安装程序。

从[版本 v0.42.0-rc.0](https://github.com/kubevirt/kubevirt/releases/tag/v0.42.0-rc.0)开始，KubeVirt 支持从外部源启动。这样就可以定义一台虚拟机，在启动过程中使用自定义 kernel / initrd 二进制文件，并可以带上自定义参数。

这些二进制文件通过容器镜像提供。
该容器从容器仓库拉取，并驻留在承载这些 VM 的本地节点上。

## 使用场景 {#use-cases}
这方面的一些使用场景可能是：
- 对 kernel 开发者来说，启动被定义为从经常变更的最新 kernel 二进制文件启动的 VM，可能会非常方便。
- 可以把需要在 VM 整个生命周期内驻留在内存中的文件设置到 initrd 中。

## 工作流程 {#workflow}
可以按以下方式定义外部启动源：
```yaml
apiVersion: kubevirt.io/v1
kind: VirtualMachine
metadata:
  name: ext-kernel-boot-vm
spec:
  runStrategy: Manual
  template:
    spec:
      domain:
        devices: {}
        firmware:
          kernelBoot:
            container:
              image: vmi_ext_boot/kernel_initrd_binaries_container:latest
              initrdPath: /boot/initramfs-virt
              kernelPath: /boot/vmlinuz-virt
              imagePullPolicy: Always
              imagePullSecret: IfNotPresent
            kernelArgs: console=ttyS0
        resources:
          requests:
            memory: 1Gi
```

说明：

- `initrdPath` 和 `kernelPath` 定义容器内二进制文件的路径。

- Kernel 和 Initrd 二进制文件的所有者必须是 `qemu` 用户和组。
  - 要更改所有权：执行 `chown qemu:qemu <binary>`，其中 `<binary>` 是该二进制文件。

- `kernelArgs` 只有在提供了 kernel 二进制文件时才能提供（即未定义 `kernelPath`）。这些参数会传给 VM 启动所用的默认 kernel。
  
- `imagePullSecret` 和 `imagePullPolicy` 是可选的

- 如果 `imagePullPolicy` 为 `Always` 且容器镜像已更新，则 VM 重启时会启动到新 kernel


要让 kernel 与根文件系统一起启动，可以把根文件系统指定为[磁盘](https://kubevirt.io/user-guide/storage/disks_and_volumes/)（例如 containerDisk）：
```yaml
apiVersion: kubevirt.io/v1
kind: VirtualMachine
metadata:
  name: ext-kernel-boot-vm
spec:
  runStrategy: Manual
  instancetype:
    name: u1.medium
  preference:
    name: fedora
  template:
    spec:
      domain:
        devices:
          disks:
            - name: kernel-modules
              cdrom:
                bus: sata
        firmware:
          kernelBoot:
            container:
              image: custom-containerdisk:latest
              initrdPath: /boot/initramfs
              kernelPath: /boot/vmlinuz
              imagePullPolicy: Always
            kernelArgs: "no_timer_check console=tty1 console=ttyS0,115200n8 systemd=off root=/dev/vda4 rootflags=subvol=root"
      volumes:
        - name: root-filesystem
          containerDisk:
            image: custom-containerdisk:latest
            imagePullPolicy: Always
        - name: kernel-modules
          containerDisk:
            image: custom-containerdisk:latest
            path: /boot/kernel-modules.isofs
            imagePullPolicy: Always
        - name: cloudinitdisk
          cloudInitNoCloud:
            userData: |-
              #cloud-config
              chpasswd:
                expire: false
              password: fedora
              user: fedora
              runcmd:
                - "sudo mkdir /mnt/kernel-modules-disk"
                - "sudo mount /dev/sr0 /mnt/kernel-modules-disk"
                - "sudo tar -xvf /mnt/kernel-modules-disk/kernel_m.tgz --directory /usr/lib/modules/"
```

说明：

- 不必把根文件系统和 kernel 打包进同一个 `containerDisk`。也可以通过类似 `dataVolumeDisk` 的方式拉取文件系统。

- 如果自定义 kernel 被配置为构建模块，就需要把这些模块安装到根文件系统中。在此示例中，kernel 模块被打包成 tarball 并放入 isofs。然后把该 isofs 加入 `containerDisk`，并用 cloudinit runcmd 解包到根文件系统中。

- 按照 [containerDisk 工作流程示例](https://kubevirt.io/user-guide/storage/disks_and_volumes/#containerdisk-workflow-example)，我们需要把 initramfs、kernel、根文件系统和 kernel 模块 isofs 加入我们的 custom-containerdisk：
  ```dockerfile
  FROM scratch
  ADD --chown=107:107 initramfs /boot/
  ADD --chown=107:107 vmlinuz /boot/
  ADD --chown=107:107 https://download.fedoraproject.org/pub/fedora/linux/releases/42/Cloud/x86_64/images/Fedora-Cloud-Base-Generic-42-1.1.x86_64.qcow2 /disk/
  ADD --chown=107:107 kernel-modules.isofs /boot/
  ```
- `kernelArgs` 必须指定正确的根设备。在此例中，这些参数只是从 cloud 镜像的 `/proc/cmdline` 复制而来。
