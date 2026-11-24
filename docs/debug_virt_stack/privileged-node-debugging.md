# 在节点上进行特权调试 {#privileged-debugging-on-the-node}

本文介绍你可以创建特权 Pod、并获得集群节点 root 访问权限的那些场景。

借助特权 Pod，你可以访问 `/dev` 中的设备，使用宿主机命名空间并对节点上正在运行的进程进行 ptrace，以及使用 `hostPath` 卷把节点目录挂载到容器中。

一种快速验证你是否被允许创建特权 Pod 的方法，是使用 `--dry-run=server` 选项创建一个示例 Pod，例如：

```console
$ kubectl apply -f debug-pod.yaml --dry-run=server
```

# 构建容器镜像 {#build-the-container-image}

KubeVirt 使用 [distroless 容器](https://github.com/GoogleContainerTools/distroless)，而这些镜像没有软件包管理器，因此无法把该镜像用作父镜像来安装额外的软件包。

在某些调试场景中，这些工具要求恰好有相同的二进制文件可用。不过，如果调试工具运行在另一个容器中，这一点会特别困难，因为容器的文件系统是相互隔离的。

本节将介绍如何构建一个容器镜像，其中包含调试工具，以及你想要调试的那个 KubeVirt 版本的二进制文件。

根据你的安装情况，KubeVirt CR 的命名空间和名称可能有所不同。在本示例中，我们假定 KubeVirt CR 名为 `kubevirt`，并安装在 `kubevirt` 命名空间中。你可以在自己的集群中通过 `kubectl get kubevirt -A` 进行搜索，轻松查出它的名称。
这样做是必要的，因为我们需要获取原始的 `virt-launcher` 镜像，以便拥有我们想要调试的完全相同的 QEMU 二进制文件。

获取 KubeVirt 安装所用镜像的镜像仓库：
```console
$ export registry=$(kubectl get kubevirt kubevirt -n kubevirt  -o jsonpath='{.status.observedDeploymentConfig}' |jq '.registry'|tr -d "\"")
$ echo $registry
"registry:5000/kubevirt"
```

获取 virt-launcher 镜像的 shasum：
```console
$ export tag=$(kubectl get kubevirt kubevirt -n kubevirt  -o jsonpath='{.status.observedDeploymentConfig}' |jq '.virtLauncherSha'|tr -d "\"")
$ echo $tag
"sha256:6c8b85eed8e83a4c70779836b246c057d3e882eb513f3ded0a02e0a4c4bda837"
```

Dockerfile：
```dockerfile
ARG registry
ARG tag
FROM ${registry}/kubevirt/virt-launcher${tag} AS launcher

FROM quay.io/centos/centos:stream9

RUN yum install -y \
        gdb \
        kernel-devel \
        qemu-kvm-tools \
        strace \
        systemtap-client \
        systemtap-devel \
    && yum clean all
COPY --from=launcher / /
```

然后，我们可以使用前面步骤中获取到的 `registry` 和 `tag` 来构建该镜像：
```console
$ podman build \
    -t debug-tools \
    --build-arg registry=$registry  \
    --build-arg tag=@$tag \
    -f Dockerfile .
```

## 部署特权调试 Pod {#deploy-the-privileged-debug-pod}

下面这个示例就如何定义你的调试 Pod 给出几条建议：

```yaml
kind: Pod
metadata:
  name: node01-debug
spec:
  containers:
  - command:
    - /bin/sh
    image: registry:5000/debug-tools:latest
    imagePullPolicy: Always
    name: debug
    securityContext:
      privileged: true
      runAsUser: 0
    stdin: true
    stdinOnce: true
    tty: true
    volumeMounts:
    - mountPath: /host
      name: host
    - mountPath: /usr/lib/modules
      name: modules
    - mountPath: /sys/kernel
      name: sys-kernel
  hostNetwork: true
  hostPID: true
  nodeName: node01
  restartPolicy: Never
  volumes:
  - hostPath:
      path: /
      type: Directory
    name: host
  - hostPath:
      path: /usr/lib/modules
      type: Directory
    name: modules
  - hostPath:
      path: /sys/kernel
      type: Directory
    name: sys-kernel
```

要能够访问节点上几乎所有的资源，`privileged` 选项是必需的。

`nodeName` 确保调试 Pod 会被调度到所期望的节点上。为了选择正确的节点，你可以使用 `-owide` 选项配合 `kubectl get po`，这样就会报告该 Pod 正在运行的节点。

示例：
```console
 k get pods -owide
NAME                                READY   STATUS    RESTARTS   AGE     IP               NODE     NOMINATED NODE   READINESS GATES
local-volume-provisioner-4jtkb      1/1     Running   0          152m    10.244.196.129   node01   <none>           <none>
node01-debug                        1/1     Running   0          44m     192.168.66.101   node01   <none>           <none>
virt-launcher-vmi-ephemeral-xg98p   3/3     Running   0          2m54s   10.244.196.148   node01   <none>           1/1
```

在 `volumes` 部分，你可以指定希望被直接挂载进调试容器的那些目录。例如，如果你需要加载某些内核模块，`/usr/lib/modules` 就特别有用。

借助 `hostPID` 选项共享宿主机 PID 命名空间，使你能够看到节点上的所有进程，并用 `gdb` 和 `strace` 之类的工具附加到进程上。

对 Pod 执行 `exec` 会给你一个 shell，让你拥有对节点的特权访问，外加你安装进该镜像的那些工具：

```console
$ kubectl exec -ti debug -- bash
```

下面的示例假定你已经 exec 进入了 `node01-debug` 这个 Pod。

### 验证宿主机是否适合虚拟化 {#validating-the-host-for-virtualization}

工具 [`vist-host-validate`](https://libvirt.org/manpages/virt-host-validate.html) 是用来验证宿主机能否运行 libvirt 虚拟机监控程序的实用工具。例如，可以用它来检查某个特定节点是否具备 kvm 能力。

示例：
```console
$  virt-host-validate
  QEMU: Checking for hardware virtualization                                 : PASS
  QEMU: Checking if device /dev/kvm exists                                   : PASS
  QEMU: Checking if device /dev/kvm is accessible                            : PASS
  QEMU: Checking if device /dev/vhost-net exists                             : PASS
  QEMU: Checking if device /dev/net/tun exists                               : PASS
  QEMU: Checking for cgroup 'cpu' controller support                         : PASS
  QEMU: Checking for cgroup 'cpuacct' controller support                     : PASS
  QEMU: Checking for cgroup 'cpuset' controller support                      : PASS
  QEMU: Checking for cgroup 'memory' controller support                      : PASS
  QEMU: Checking for cgroup 'devices' controller support                     : PASS
  QEMU: Checking for cgroup 'blkio' controller support                       : PASS
  QEMU: Checking for device assignment IOMMU support                         : PASS
  QEMU: Checking if IOMMU is enabled by kernel                               : PASS
  QEMU: Checking for secure guest support                                    : WARN (Unknown if this platform has Secure
```

### 直接在节点上运行命令 {#run-a-command-directly-on-the-node}

调试容器在卷部分把宿主机文件系统挂载在 `/host` 之下。如果你想访问节点文件系统，或者直接在宿主机上执行命令，这一点会特别有用。不过，该工具必须已经存在于节点上。

```console
# chroot /host
sh-5.1# cat /etc/os-release
NAME="CentOS Stream"
VERSION="9"
ID="centos"
ID_LIKE="rhel fedora"
VERSION_ID="9"
PLATFORM_ID="platform:el9"
PRETTY_NAME="CentOS Stream 9"
ANSI_COLOR="0;31"
LOGO="fedora-logo-icon"
CPE_NAME="cpe:/o:centos:centos:9"
HOME_URL="https://centos.org/"
BUG_REPORT_URL="https://bugzilla.redhat.com/"
REDHAT_SUPPORT_PRODUCT="Red Hat Enterprise Linux 9"
REDHAT_SUPPORT_PRODUCT_VERSION="CentOS Stream"
```

### 附加到正在运行的进程（例如 strace 或 gdb） {#attach-to-a-running-process-eg-strace-or-gdb}

这需要字段 `hostPID: true`，这样一来，你就能够列出节点上正在运行的所有进程。

```console
$ ps -ef |grep qemu-kvm
qemu       50122   49850  0 12:34 ?        00:00:25 /usr/libexec/qemu-kvm -name guest=default_vmi-ephemeral,debug-threads=on -S -object {"qom-type":"secret","id":"masterKey0","format":"raw","file":"/var/run/kubevirt-private/libvirt/qemu/lib/domain-1-default_vmi-ephemera/master-key.aes"} -machine pc-q35-rhel9.2.0,usb=off,dump-guest-core=off,memory-backend=pc.ram,acpi=on -accel kvm -cpu Skylake-Client-IBRS,ss=on,vmx=on,pdcm=on,hypervisor=on,tsc-adjust=on,clflushopt=on,umip=on,md-clear=on,stibp=on,flush-l1d=on,arch-capabilities=on,ssbd=on,xsaves=on,pdpe1gb=on,ibpb=on,ibrs=on,amd-stibp=on,amd-ssbd=on,rdctl-no=on,ibrs-all=on,skip-l1dfl-vmentry=on,mds-no=on,pschange-mc-no=on,tsx-ctrl=on,fb-clear=on,hle=off,rtm=off -m size=131072k -object {"qom-type":"memory-backend-ram","id":"pc.ram","size":134217728} -overcommit mem-lock=off -smp 1,sockets=1,dies=1,cores=1,threads=1 -object {"qom-type":"iothread","id":"iothread1"} -uuid b56f06f0-07e9-4fe5-8913-18a14e83a4d1 -smbios type=1,manufacturer=KubeVirt,product=None,uuid=b56f06f0-07e9-4fe5-8913-18a14e83a4d1,family=KubeVirt -no-user-config -nodefaults -chardev socket,id=charmonitor,fd=21,server=on,wait=off -mon chardev=charmonitor,id=monitor,mode=control -rtc base=utc -no-shutdown -boot strict=on -device {"driver":"pcie-root-port","port":16,"chassis":1,"id":"pci.1","bus":"pcie.0","multifunction":true,"addr":"0x2"} -device {"driver":"pcie-root-port","port":17,"chassis":2,"id":"pci.2","bus":"pcie.0","addr":"0x2.0x1"} -device {"driver":"pcie-root-port","port":18,"chassis":3,"id":"pci.3","bus":"pcie.0","addr":"0x2.0x2"} -device {"driver":"pcie-root-port","port":19,"chassis":4,"id":"pci.4","bus":"pcie.0","addr":"0x2.0x3"} -device {"driver":"pcie-root-port","port":20,"chassis":5,"id":"pci.5","bus":"pcie.0","addr":"0x2.0x4"} -device {"driver":"pcie-root-port","port":21,"chassis":6,"id":"pci.6","bus":"pcie.0","addr":"0x2.0x5"} -device {"driver":"pcie-root-port","port":22,"chassis":7,"id":"pci.7","bus":"pcie.0","addr":"0x2.0x6"} -device {"driver":"pcie-root-port","port":23,"chassis":8,"id":"pci.8","bus":"pcie.0","addr":"0x2.0x7"} -device {"driver":"pcie-root-port","port":24,"chassis":9,"id":"pci.9","bus":"pcie.0","addr":"0x3"} -device {"driver":"virtio-scsi-pci-non-transitional","id":"scsi0","bus":"pci.5","addr":"0x0"} -device {"driver":"virtio-serial-pci-non-transitional","id":"virtio-serial0","bus":"pci.6","addr":"0x0"} -blockdev {"driver":"file","filename":"/var/run/kubevirt/container-disks/disk_0.img","node-name":"libvirt-2-storage","cache":{"direct":true,"no-flush":false},"auto-read-only":true,"discard":"unmap"} -blockdev {"node-name":"libvirt-2-format","read-only":true,"discard":"unmap","cache":{"direct":true,"no-flush":false},"driver":"qcow2","file":"libvirt-2-storage"} -blockdev {"driver":"file","filename":"/var/run/kubevirt-ephemeral-disks/disk-data/containerdisk/disk.qcow2","node-name":"libvirt-1-storage","cache":{"direct":true,"no-flush":false},"auto-read-only":true,"discard":"unmap"} -blockdev {"node-name":"libvirt-1-format","read-only":false,"discard":"unmap","cache":{"direct":true,"no-flush":false},"driver":"qcow2","file":"libvirt-1-storage","backing":"libvirt-2-format"} -device {"driver":"virtio-blk-pci-non-transitional","bus":"pci.7","addr":"0x0","drive":"libvirt-1-format","id":"ua-containerdisk","bootindex":1,"write-cache":"on","werror":"stop","rerror":"stop"} -netdev {"type":"tap","fd":"22","vhost":true,"vhostfd":"24","id":"hostua-default"} -device {"driver":"virtio-net-pci-non-transitional","host_mtu":1480,"netdev":"hostua-default","id":"ua-default","mac":"7e:cb:ba:c3:71:88","bus":"pci.1","addr":"0x0","romfile":""} -add-fd set=0,fd=20,opaque=serial0-log -chardev socket,id=charserial0,fd=18,server=on,wait=off,logfile=/dev/fdset/0,logappend=on -device {"driver":"isa-serial","chardev":"charserial0","id":"serial0","index":0} -chardev socket,id=charchannel0,fd=19,server=on,wait=off -device {"driver":"virtserialport","bus":"virtio-serial0.0","nr":1,"chardev":"charchannel0","id":"channel0","name":"org.qemu.guest_agent.0"} -audiodev {"id":"audio1","driver":"none"} -vnc vnc=unix:/var/run/kubevirt-private/3a8f7774-7ec7-4cfb-97ce-581db52ee053/virt-vnc,audiodev=audio1 -device {"driver":"VGA","id":"video0","vgamem_mb":16,"bus":"pcie.0","addr":"0x1"} -global ICH9-LPC.noreboot=off -watchdog-action reset -device {"driver":"virtio-balloon-pci-non-transitional","id":"balloon0","free-page-reporting":true,"bus":"pci.8","addr":"0x0"} -sandbox on,obsolete=deny,elevateprivileges=deny,spawn=deny,resourcecontrol=deny -msg timestamp=on
$ gdb -p 50122 /usr/libexec/qemu-kvm
```

### 使用 `crictl` 进行调试 {#debugging-using-crictl}

[`Crictl`](https://github.com/kubernetes-sigs/cri-tools/blob/master/docs/crictl.md) 是面向 CRI 运行时的命令行工具，对排查容器故障会特别有用（如需更详细的指南，请参阅这篇 [Kubernetes 文章](https://kubernetes.io/docs/tasks/debug/debug-cluster/crictl/)）。

在本示例中，我们将专注于找出 libvirt 在 virt-launcher Pod 的 `compute` 容器里于何处创建文件和目录。

```console
$ crictl ps |grep compute
67bc7be3222da       5ef5ba25a087a80e204f28be6c9250bbf378fd87fa927085abd516188993d695                                                       25 minutes ago      Running             compute                   0                   7b045ea9f485f       virt-launcher-vmi-ephemeral-xg98p
$ crictl inspect 67bc7be3222da
[..]
    "mounts": [
      {
      {
        "containerPath": "/var/run/libvirt",
        "hostPath": "/var/lib/kubelet/pods/2ccc3e93-d1c3-4f22-bb31-321bfa74edf6/volumes/kubernetes.io~empty-dir/libvirt-runtime",
        "propagation": "PROPAGATION_PRIVATE",
        "readonly": false,
        "selinuxRelabel": true
      },
[..]
$ ls /var/lib/kubelet/pods/2ccc3e93-d1c3-4f22-bb31-321bfa74edf6/volumes/kubernetes.io~empty-dir/libvirt-runtime/
common	    qemu		 virtlogd-sock	virtqemud-admin-sock  virtqemud.conf
hostdevmgr  virtlogd-admin-sock  virtlogd.pid	virtqemud-sock	      virtqemud.pid
```
