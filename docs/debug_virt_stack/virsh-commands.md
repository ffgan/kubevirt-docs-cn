# 在 virt-launcher Pod 中执行 virsh 命令 {#execute-virsh-commands-in-virt-launcher-pod}

一个用来检查并排查 VM 状态的强大工具是 [`virsh`](https://www.libvirt.org/manpages/virsh.html)，而且该工具已经安装在 virt-launcher Pod 的 `compute` 容器中。

例如，可以运行任何 QMP 命令。

如需 QMP 命令的完整列表，请参阅 [QEMU 文档](https://qemu-project.gitlab.io/qemu/interop/qemu-qmp-ref.html)。

```console
$ kubectl get po
NAME                                READY   STATUS    RESTARTS   AGE
virt-launcher-vmi-ephemeral-xg98p   3/3     Running   0          44m
$ kubectl exec -ti virt-launcher-vmi-debug-tools-fk64q -- bash
bash-5.1$ virsh list
 Id   Name                      State
-----------------------------------------
 1    default_vmi-debug-tools   running
bash-5.1$ virsh qemu-monitor-command default_vmi-debug-tools query-status --pretty
{
  "return": {
    "status": "running",
    "singlestep": false,
    "running": true
  },
  "id": "libvirt-439"
}
$ virsh qemu-monitor-command default_vmi-debug-tools query-kvm --pretty
{
  "return": {
    "enabled": true,
    "present": true
  },
  "id": "libvirt-438"
}
```

另一个有用的 virsh 命令是 `qemu-monitor-event`。一经调用，它就会观察并报告 QEMU 事件。

下面的示例展示因暂停和取消暂停客户机而产生的事件。

```console
$ kubectl get po
NAME                                READY   STATUS    RESTARTS   AGE
virt-launcher-vmi-ephemeral-nqcld   3/3     Running   0          57m
$ kubectl exec -ti virt-launcher-vmi-ephemeral-nqcld -- virsh qemu-monitor-event --pretty --loop
```

然后，你可以例如先暂停、再取消暂停该客户机，并查看被触发的事件：
```console
$ virtctl pause vmi vmi-ephemeral
VMI vmi-ephemeral was scheduled to pause
 $ virtctl unpause vmi vmi-ephemeral
VMI vmi-ephemeral was scheduled to unpause
```

来自被监视的事件：
```console
$ kubectl exec -ti virt-launcher-vmi-ephemeral-nqcld -- virsh qemu-monitor-event --pretty --loop
event STOP at 1698405797.422823 for domain 'default_vmi-ephemeral': <null>
event RESUME at 1698405823.162458 for domain 'default_vmi-ephemeral': <null>
```
