# 使用 gdb 启动 QEMU，并用本地 gdb 客户端进行连接 {#launch-qemu-with-gdb-and-connect-locally-with-gdb-client}

本指南适用于 QEMU 遇到非常早期的故障、因而很难在之后的时间点再与之同步的情况。

## 镜像创建与 PVC 填充 {#image-creation-and-pvc-population}

此场景相对[关于启动 strace 的指南](../debug_virt_stack/launch-qemu-strace.md)只做了少许变化，因此镜像构建和 PVC 填充方面的部分细节在此直接略过，并在另一节中说明。

在本示例中，QEMU 将与 [`gdbserver`](https://man7.org/linux/man-pages/man1/gdbserver.1.html) 一起启动，之后我们将使用本地 `gdb` 客户端连接到它。

包装脚本如下：
```bash
#!/bin/bash

LD_LIBRARY_PATH=$LD_LIBRARY_PATH:/var/run/debug/usr/lib64 /var/run/debug/usr/bin/gdbserver \
	localhost:1234 \
	/usr/libexec/qemu-kvm $@ &
printf "%d" $(pgrep gdbserver) > /run/libvirt/qemu/run/default_vmi-debug-tools.pid

```

首先，我们需要构建并推送带有包装脚本和 gdbserver 的镜像：
```dockerfile
FROM quay.io/centos/centos:stream9 as build

ENV DIR /debug-tools
ENV DEBUGINFOD_URLS https://debuginfod.centos.org/
RUN mkdir -p ${DIR}/logs

RUN yum  install --installroot=${DIR} -y gdb-gdbserver && yum clean all

COPY ./wrap_qemu_gdb.sh $DIR/wrap_qemu_gdb.sh
RUN chmod 0755 ${DIR}/wrap_qemu_gdb.sh
RUN chown 107:107 ${DIR}/wrap_qemu_gdb.sh
RUN chown 107:107 ${DIR}/logs
```

然后，我们可以像在 [strace 示例](../debug_virt_stack/launch-qemu-strace.md)中那样，创建并填充 `debug-tools` PVC：
```console
$ k apply -f debug-tools-pvc.yaml
persistentvolumeclaim/debug-tools created
$ kubectl  apply -f populate-job-pvc.yaml
job.batch/populate-pvc created
$ kubectl  get jobs
NAME           COMPLETIONS   DURATION   AGE
populate-pvc   1/1           7s         2m12s
```

ConfigMap：
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: my-config-map
data:
  my_script.sh: |
    #!/bin/sh
    tempFile=`mktemp --dry-run`
    echo $4 > $tempFile
    sed -i "s|<emulator>/usr/libexec/qemu-kvm</emulator>|<emulator>/var/run/debug/wrap_qemu_gdb.sh</emulator>|" $tempFile
    cat $tempFile
```

作为最后一步，我们需要创建这些用于修改 VM XML 的 ConfigMap：
```console
$ kubectl apply -f configmap.yaml
configmap/my-config-map created
```

### 构建客户端镜像 {#build-client-image}

在此场景中，我们使用一个额外的容器镜像，其中包含 `gdb` 以及与要调试的目标进程相同的 QEMU 二进制文件。该镜像将在本地使用 `podman` 运行。

为了构建此镜像，我们需要确定要调试的 `virt-launcher` 容器所用的镜像。根据 KubeVirt 的安装情况，KubeVirt CR 的命名空间和名称可能有所不同。在本示例中，我们假定 KubeVirt CR 名为 `kubevirt`，并安装在 `kubevirt` 命名空间中。

你可以在集群中通过下面的搜索，轻松查出正确的名称：
```console
$ kubectl get kubevirt -A
NAMESPACE   NAME       AGE     PHASE
kubevirt    kubevirt   3h11m   Deployed
```

构建镜像的步骤如下：

1. 获取 KubeVirt 安装所用镜像的镜像仓库：
```console
$ export registry=$(kubectl get kubevirt kubevirt -n kubevirt  -o jsonpath='{.status.observedDeploymentConfig}' |jq '.registry'|tr -d "\"")
$ echo $registry
"registry:5000/kubevirt"
```

2. 获取 virt-launcher 镜像的 shasum：
```console
$ export tag=$(kubectl get kubevirt kubevirt -n kubevirt  -o jsonpath='{.status.observedDeploymentConfig}' |jq '.virtLauncherSha'|tr -d "\"")
$ echo $tag
"sha256:6c8b85eed8e83a4c70779836b246c057d3e882eb513f3ded0a02e0a4c4bda837"
```

Dockerfile 示例：
```dockerfile
ARG registry
ARG tag
FROM ${registry}/kubevirt/virt-launcher${tag} AS launcher
FROM quay.io/centos/centos:stream9 as build

RUN yum  install -y gdb && yum clean all

COPY --from=launcher /usr/libexec/qemu-kvm /usr/libexec/qemu-kvm
```

3. 使用前面步骤中获取到的 `registry` 和 `tag` 来构建镜像：
```console
$ podman build \
    -t gdb-client \
    --build-arg registry=$registry  \
    --build-arg tag=@$tag \
    -f Dockerfile.client .
```

Podman 会替换命令行上提供的 registry 和 tag 参数。这样，我们就可以为要调试的 KubeVirt 版本指定镜像仓库和 shasum。

## 运行 VM 进行故障排查 {#run-the-vm-to-troubleshoot}

在本示例中，我们添加一个注解，以便即使出现任何错误，virt-launcher Pod 也会继续运行：
```yaml
metadata:
  annotations:
    kubevirt.io/keep-launcher-alive-after-failure: "true"
```

然后，我们可以启动该 VM：
```console
$ kubectl apply -f debug-vmi.yaml
virtualmachineinstance.kubevirt.io/vmi-debug-tools created
$ kubectl  get vmi
NAME              AGE   PHASE       IP    NODENAME   READY
vmi-debug-tools   28s   Scheduled         node01     False
$ kubectl  get po
NAME                                  READY   STATUS      RESTARTS   AGE
populate-pvc-dnxld                    0/1     Completed   0          4m17s
virt-launcher-vmi-debug-tools-tfh28   4/4     Running     0          25s
```


包装脚本会启动 `gdbserver`，并在容器内部暴露端口 `1234`。为了能够远程连接到该 gdbserver，我们可以使用 `kubectl port-forward` 命令，在我们的机器上暴露 gdb 端口。

```console
$ kubectl  port-forward virt-launcher-vmi-debug-tools-tfh28 1234
Forwarding from 127.0.0.1:1234 -> 1234
Forwarding from [::1]:1234 -> 1234

```

最后，我们可以在容器中启动 gdb 客户端：
```console
$ podman run -ti --network host gdb-client:latest
$ gdb /usr/libexec/qemu-kvm -ex 'target remote localhost:1234'
GNU gdb (GDB) Red Hat Enterprise Linux 10.2-12.el9
Copyright (C) 2021 Free Software Foundation, Inc.
License GPLv3+: GNU GPL version 3 or later <http://gnu.org/licenses/gpl.html>
This is free software: you are free to change and redistribute it.
There is NO WARRANTY, to the extent permitted by law.
Type "show copying" and "show warranty" for details.
This GDB was configured as "x86_64-redhat-linux-gnu".
Type "show configuration" for configuration details.
For bug reporting instructions, please see:
<https://www.gnu.org/software/gdb/bugs/>.
Find the GDB manual and other documentation resources online at:
    <http://www.gnu.org/software/gdb/documentation/>.

For help, type "help".
--Type <RET> for more, q to quit, c to continue without paging--
Type "apropos word" to search for commands related to "word"...
Reading symbols from /usr/libexec/qemu-kvm...

Reading symbols from /root/.cache/debuginfod_client/26221a84fabd219a68445ad0cc87283e881fda15/debuginfo...
Remote debugging using localhost:1234
Reading /lib64/ld-linux-x86-64.so.2 from remote target...
warning: File transfers from remote targets can be slow. Use "set sysroot" to access files locally instead.
Reading /lib64/ld-linux-x86-64.so.2 from remote target...
Reading symbols from target:/lib64/ld-linux-x86-64.so.2...
Downloading separate debug info for /system-supplied DSO at 0x7ffc10eff000...
0x00007f1a70225e70 in _start () from target:/lib64/ld-linux-x86-64.so.2
```

为简单起见，我们使用 `--network host` 选项启动了 podman，这样一来，容器就能够访问映射到宿主机上的任何端口。
