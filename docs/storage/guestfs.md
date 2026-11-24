# libguestfs-tools 与 virtctl guestfs 的用法 {#usage-of-libguestfs-tools-and-virtctl-guestfs}

[Libguestfs 工具](https://libguestfs.org/)是一套用于访问和修改 VM 磁盘镜像的实用程序。`virtctl guestfs` 命令有助于部署一个交互式容器，其中包含 libguestfs-tools，并挂接 PVC。如果用户需要修改、检查或调试 PVC 上的 VM 磁盘，此命令特别有用。
```bash
$ virtctl guestfs -h
Create a pod with libguestfs-tools, mount the pvc and attach a shell to it. The pvc is mounted under the /disks directory inside the pod for filesystem-based pvcs, or as /dev/vda for block-based pvcs

Usage:
  virtctl guestfs [flags]

Examples:
  # Create a pod with libguestfs-tools, mount the pvc and attach a shell to it:
  virtctl guestfs <pvc-name>

Flags:
  -h, --help                 help for guestfs
      --image string         libguestfs-tools container image
      --kvm                  Use kvm for the libguestfs-tools container (default true)
      --pull-policy string   pull policy for the libguestfs image (default "IfNotPresent")

Use "virtctl options" for a list of global command-line options (applies to all commands).
```

默认情况下，`virtctl guestfs` 会为交互式容器设置 `kvm`。由于 libguestfs-tools 使用 QEMU，这会显著加快它们的执行速度。如果集群中没有任何支持 kvm 的节点，用户必须通过设置选项 `--kvm=false` 来禁用 kvm。如果未设置，libguestfs-tools Pod 将保持 Pending，因为它无法被调度到任何节点。

该命令会自动使用 KubeVirt 在 http 端点 `/apis/subresources.kubevirt.io/<kubevirt-version>/guestfs` 下暴露的镜像，但可以通过选项 `--image` 将其配置为使用自定义镜像。用户还可以通过设置选项 `pull-policy` 来覆盖镜像的拉取策略。

该命令会检查 PVC 是否正被另一个 Pod 使用，如果是，则命令会失败。然而，一旦 libguestfs-tools 已经启动，此设置并不会阻止新的 Pod 启动并使用同一个 PVC。在启动访问同一 PVC 的 VM 之前，用户需要确认没有活动的 virtctl guestfs Pod。

目前，`virtctl guestfs` 只支持单个 PVC。未来版本可能会支持向交互式 Pod 挂接多个 PVC。

## 示例与用例 {#examples-and-use-cases}
一般来说，用户可以利用 `virtctl guestfs` 命令完成 libguestfs-tools 的所有典型用法。强烈建议查阅[官方文档](https://libguestfs.org/guestfs-recipes.1.html)。此命令的目的只是帮助在安装了 KubeVirt 的 Kubernetes 集群中配置正确的容器化环境。

对于所有示例，用户都必须在 `virtctl guestfs` 命令中引用 PVC 来启动交互式容器。这将部署交互式 Pod，并挂接 stdin 和 stdout。

示例：

```bash
$ virtctl guestfs pvc-test
Use image: registry:5000/kubevirt/libguestfs-tools@sha256:6644792751b2ba9442e06475a809448b37d02d1937dbd15ad8da4d424b5c87dd 
The PVC has been mounted at /disk 
Waiting for container libguestfs still in pending, reason: ContainerCreating, message:  
Waiting for container libguestfs still in pending, reason: ContainerCreating, message:  
bash-5.0#
```
一旦 libguestfs-tools Pod 部署完成，用户就可以访问磁盘并执行所需的命令。之后，当用户完成磁盘上的操作后，只需 `exit` 该容器，Pod 就会被自动终止。
 
1. 检查磁盘文件系统，以获取磁盘上操作系统的版本：
```bash
bash-5.0# virt-cat -a disk.img /etc/os-release 
NAME=Fedora
VERSION="34 (Cloud Edition)"
ID=fedora
VERSION_ID=34
VERSION_CODENAME=""
PLATFORM_ID="platform:f34"
PRETTY_NAME="Fedora 34 (Cloud Edition)"
ANSI_COLOR="0;38;2;60;110;180"
LOGO=fedora-logo-icon
CPE_NAME="cpe:/o:fedoraproject:fedora:34"
HOME_URL="https://fedoraproject.org/"
DOCUMENTATION_URL="https://docs.fedoraproject.org/en-US/fedora/34/system-administrators-guide/"
SUPPORT_URL="https://fedoraproject.org/wiki/Communicating_and_getting_help"
BUG_REPORT_URL="https://bugzilla.redhat.com/"
REDHAT_BUGZILLA_PRODUCT="Fedora"
REDHAT_BUGZILLA_PRODUCT_VERSION=34
REDHAT_SUPPORT_PRODUCT="Fedora"
REDHAT_SUPPORT_PRODUCT_VERSION=34
PRIVACY_POLICY_URL="https://fedoraproject.org/wiki/Legal:PrivacyPolicy"
VARIANT="Cloud Edition"
VARIANT_ID=cloud
```
2. 添加用户（例如在使用 [CDI](https://github.com/kubevirt/containerized-data-importer) 导入磁盘之后）
```bash
bash-5.0# virt-customize -a disk.img --run-command 'useradd -m test-user -s /bin/bash' --password  'test-user:password:test-password'
[   0.0] Examining the guest ...
[   4.1] Setting a random seed
[   4.2] Setting the machine ID in /etc/machine-id
[   4.2] Running: useradd -m test-user -s /bin/bash
[   4.3] Setting passwords
[   5.3] Finishing off
```
3. 运行 virt-rescue，并修复损坏的分区或 initrd（例如通过运行 dracut）
```bash
bash-5.0# virt-rescue -a disk.img
[...]
The virt-rescue escape key is ‘^]’.  Type ‘^] h’ for help.

------------------------------------------------------------

Welcome to virt-rescue, the libguestfs rescue shell.

Note: The contents of / (root) are the rescue appliance.
You have to mount the guest’s partitions under /sysroot
before you can examine them.
><rescue> fdisk -l
Disk /dev/sda: 6 GiB, 6442450944 bytes, 12582912 sectors
Disk model: QEMU HARDDISK   
Units: sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes
Disklabel type: gpt
Disk identifier: F8DC0844-9194-4B34-B432-13FA4B70F278

Device       Start      End  Sectors Size Type
/dev/sda1     2048     4095     2048   1M BIOS boot
/dev/sda2     4096  2101247  2097152   1G Linux filesystem
/dev/sda3  2101248 12580863 10479616   5G Linux filesystem


Disk /dev/sdb: 4 GiB, 4294967296 bytes, 8388608 sectors
Disk model: QEMU HARDDISK   
Units: sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes
><rescue> mount /dev/sda3 sysroot/
><rescue> mount /dev/sda2 sysroot/boot
><rescue> chroot sysroot/
><rescue> ls boot/
System.map-5.11.12-300.fc34.x86_64
config-5.11.12-300.fc34.x86_64
efi
grub2
initramfs-0-rescue-8afb5b540fab48728e48e4196a3a48ee.img
initramfs-5.11.12-300.fc34.x86_64.img
loader
vmlinuz-0-rescue-8afb5b540fab48728e48e4196a3a48ee
><rescue> dracut -f boot/initramfs-5.11.12-300.fc34.x86_64.img 5.11.12-300.fc34.x86_64
[...]
><rescue> exit # <- exit from chroot
><rescue> umount sysroot/boot
><rescue> umount sysroot
><rescue> exit
```

4. 从头安装操作系统
```bash
bash-5.0# virt-builder centos-8.2 -o disk.img --root-password password:password-test
[   1.5] Downloading: http://builder.libguestfs.org/centos-8.2.xz
######################################################################## 100.0%#=#=#                                                    ######################################################################## 100.0%
[  58.3] Planning how to build this image
[  58.3] Uncompressing
[  65.7] Opening the new disk
[  70.8] Setting a random seed
[  70.8] Setting passwords
[  72.0] Finishing off
                   Output file: disk.img
                   Output size: 6.0G
                 Output format: raw
            Total usable space: 5.3G
                    Free space: 4.0G (74%)

```
5. 识别磁盘上的分区和文件系统
````bash
bash-5.0# virt-filesystems -a disk.img --partitions --filesystem --long
Name       Type        VFS   Label  MBR  Size        Parent
/dev/sda2  filesystem  ext4  -      -    1023303680  -
/dev/sda4  filesystem  xfs   -      -    4710203392  -
/dev/sda1  partition   -     -      -    1048576     /dev/sda
/dev/sda2  partition   -     -      -    1073741824  /dev/sda
/dev/sda3  partition   -     -      -    644874240   /dev/sda
/dev/sda4  partition   -     -      -    4720689152  /dev/sda
````
## 限制 {#limitations}
目前无法调整 xfs 文件系统的大小。
