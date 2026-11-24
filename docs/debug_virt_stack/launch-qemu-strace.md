# 使用 strace 启动 QEMU {#launch-qemu-with-strace}

本指南说明如何在 virt-launcher Pod 中使用调试工具来启动 QEMU。此方法可用于调试早期故障，或者把 QEMU 作为依赖 [ptrace](https://man7.org/linux/man-pages/man2/ptrace.2.html) 的调试工具的子进程来启动。第二点在进程运行于非特权环境中时尤其重要，因为否则就需要 root 访问权限，才能对该进程进行 ptrace。

[临时容器](https://kubernetes.io/docs/concepts/workloads/pods/ephemeral-containers/)是弥补原始镜像中缺少调试工具的新兴技术之一。不过，这种方案也存在若干限制。例如，可以在待调试应用的同一个 Pod 内启动一个新容器，并共享同一个 PID 命名空间。尽管它们共享同一个 PID 命名空间，但由于 KubeVirt 使用非特权容器，例如就无法对正在运行的容器进行 ptrace。因此，这种技术并不适合我们的需求。

出于安全性和缩减镜像体积的考虑，KubeVirt 容器镜像基于 [distroless 容器](https://github.com/GoogleContainerTools/distroless)。这类镜像对部署极为有益，但它们很难排查，因为没有软件包管理，而这会阻止在运行过程中安装额外的工具。

把 QEMU 二进制文件包装进脚本，是调试由 libvirt 启动的 QEMU 的一种实用方法。该脚本把 QEMU 作为本进程的子进程，与调试工具（例如 [strace](https://man7.org/linux/man-pages/man1/strace.1.html) 或 [valgrind](https://valgrind.org/)）一起启动。

最后需要添加的部分，是让 libvirt 使用包装后的脚本、而不是直接调用 QEMU 程序的配置。

可以借助 [KubeVirt sidecar](https://github.com/kubevirt/kubevirt/tree/main/cmd/sidecars) 来修改生成的 XML。这使我们能够用包装脚本代替内置模拟器。

此配置背后的主要思路是：所有额外的工具、脚本以及最终的输出文件，都将存放在一个[持久卷声明（PVC）](https://kubernetes.io/docs/concepts/storage/persistent-volumes/)中，本指南将其称为 `debug-tools`。我们希望调试的那个 virt-launcher Pod 会附加这个 PVC。

PVC：
```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: debug-tools
spec:
  accessModes:
    - ReadWriteOnce
  volumeMode: Filesystem
  resources:
    requests:
      storage: 1Gi
```

在本指南中，我们将应用上述概念，使用 strace 调试 virt-launcher 内部的 QEMU，而无需构建自定义的 virt-launcher 镜像。

你可以查看这一配置的完整演示：
[![asciicast](https://asciinema.org/a/1Cm87DcjtUWprRrWPBzKfxVmp.svg)](https://asciinema.org/a/1Cm87DcjtUWprRrWPBzKfxVmp)

## 如何把调试工具和包装脚本放进 distroless 容器 {#how-to-bring-the-debug-tools-and-wrapping-script-into-distroless-containers}

本节举例说明如何使用 Dockerfile，把额外工具以 PVC 的形式提供到 distroless 容器之中。虽然有若干种方法可以做到这一点，这里介绍一种相对简单的技术。或者，你也可以运行一个 Pod，并通过 exec 进入该 Pod 来手动填充 PVC。

Dockerfile：
```dockerfile
FROM quay.io/centos/centos:stream9 as build

ENV DIR /debug-tools
RUN mkdir -p ${DIR}/logs

RUN  yum install  --installroot=${DIR} -y strace && yum clean all

COPY ./wrap_qemu_strace.sh $DIR/wrap_qemu_strace.sh
RUN chmod 0755 ${DIR}/wrap_qemu_strace.sh
RUN chown 107:107 ${DIR}/wrap_qemu_strace.sh
RUN chown 107:107 ${DIR}/logs
```

目录 `debug-tools` 存放的是稍后会复制到 `debug-tools` PVC 内部的内容。我们实质上是用 `yum install --installroot=${DIR}}` 把所缺少的工具添加到自定义目录中，并且父镜像与 virt-launcher 的父镜像相匹配。

`wrap_qemu_strace.sh` 是用来以 `strace` 启动 QEMU 的包装脚本，做法与 `valgrind` 示例类似。
```bash
#!/bin/bash

LD_LIBRARY_PATH=$LD_LIBRARY_PATH:/var/run/debug/usr/lib64 /var/run/debug/usr/bin/strace \
        -o /var/run/debug/logs/strace.out \
        /usr/libexec/qemu-kvm $@
```

重要的是把动态库路径 `LD_LIBRARY_PATH` 设置为该 PVC 将在 virt-launcher 容器中挂载到的路径。

然后，你只需要构建该镜像，你的调试环境就准备好了。Dockerfile 以及脚本 `wrap_qemu_strace.sh` 需要位于你运行该命令的同一目录中。
```console
$ podman build -t debug .
```

第二步是填充 PVC。这可以轻松地借助 Kubernetes 的 [`Job`](https://kubernetes.io/docs/concepts/workloads/controllers/job/) 来完成，例如：
```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: populate-pvc
spec:
  template:
    spec:
      volumes:
        - name: populate
          persistentVolumeClaim:
            claimName: debug-tools
      containers:
        - name: populate
          image: registry:5000/debug:latest
          command: ["sh", "-c", "cp -r /debug-tools/* /vol"]
          imagePullPolicy: Always
          volumeMounts:
            - mountPath: "/vol"
              name: populate
      restartPolicy: Never
  backoffLimit: 4
```

`Job` 中引用的镜像，就是我们在上一步构建的镜像。应用此项并且该 Job 完成后，`debug-tools` PVC 就可以使用了。

## 如何由调试工具（例如 strace）启动 QEMU {#how-to-start-qemu-launched-by-a-debugging-tool-eg-strace}

这一部分通过 ConfigMap 和 KubeVirt sidecar 来实现（更多细节见[使用 ConfigMap 运行自定义脚本](https://github.com/kubevirt/kubevirt/tree/main/cmd/sidecars#using-configmap-to-run-custom-script)一节）。

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
    sed -i "s|<emulator>/usr/libexec/qemu-kvm</emulator>|<emulator>/var/run/debug/wrap_qemu_strace.sh</emulator>|" $tempFile
    cat $tempFile
```

用于在 XML 中把 QEMU 二进制文件替换为包装脚本的脚本，存放在 ConfigMap `my-config-map` 中。该脚本将作为钩子运行，KubeVirt sidecar 的文档对此作了完整说明。

所有对象都创建完成之后，我们终于可以运行要调试的客户机了。

VMI：
```yaml
apiVersion: kubevirt.io/v1
kind: VirtualMachineInstance
metadata:
  annotations:
    hooks.kubevirt.io/hookSidecars: '[{"args": ["--version", "v1alpha2"],
    "image":"registry:5000/kubevirt/sidecar-shim:devel",
    "pvc": {"name": "debug-tools","volumePath": "/debug", "sharedComputePath": "/var/run/debug"},
    "configMap": {"name": "my-config-map","key": "my_script.sh", "hookPath": "/usr/bin/onDefineDomain"}}]'
  labels:
    special: vmi-debug-tools
  name: vmi-debug-tools
spec:
  domain:
    devices:
      disks:
      - disk:
          bus: virtio
        name: containerdisk
      - disk:
          bus: virtio
        name: cloudinitdisk
      rng: {}
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

该 VMI 示例是一个简单的 VM 实例声明，值得关注的部分是钩子的注解：
* `image` 指的是已经构建并随 KubeVirt 一同发布的 sidecar-shim
* `pvc` 指的是已填充调试环境的 PVC。`name` 指的是声明名称，`volumePath` 是该卷在 sidecar 容器内部的挂载路径，而 `sharedComputePath` 是同一卷在 compute 容器内部的路径。
* `configMap` 指的是包含那段为包装脚本修改 XML 的脚本的 ConfigMap

VM 一经声明，钩子就会修改 emulator 部分，libvirt 将调用包装脚本，而不是直接调用 QEMU。

## 如何获取输出 {#how-to-fetch-the-output}

包装脚本把 `strace` 配置为将输出存放在 PVC 中。这样一来，就可以在之后的某个时间取回该输出文件，例如使用下面这样一个额外的 Pod：

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: fetch-logs
spec:
  securityContext:
    runAsUser: 107
    fsGroup: 107
  volumes:
    - name: populate
      persistentVolumeClaim:
        claimName: debug-tools
  containers:
    - name: populate
      image: busybox:latest
      command: ["tail", "-f", "/dev/null"]
      volumeMounts:
        - mountPath: "/vol"
          name: populate
```

然后，就可以把该文件复制到本地：
```console
$ kubectl cp fetch-logs:/vol/logs/strace.out strace.out
```
