# 控制每个组件的 libvirt 日志 {#control-libvirt-logging-for-each-component}

通常，集群管理员可以在 KubeVirt CR 中控制每个 KubeVirt 组件的日志详细级别。更多细节请查阅 [KubeVirt 文档](https://kubevirt.io/user-guide/operations/debug/#log-verbosity)。

尽管如此，普通用户也可以调整 qemu 组件的日志记录，以便对它进行更精细的控制。注解 `kubevirt.io/libvirt-log-filters` 使你能够修改每个组件的日志级别。

示例：
```yaml
apiVersion: kubevirt.io/v1
kind: VirtualMachineInstance
metadata:
  annotations:
    kubevirt.io/libvirt-log-filters: "2:qemu.qemu_monitor 3:*"
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

然后，就可以从 virt-launcher 的输出中取得这些日志：

```console
$ kubectl get pods
NAME                                  READY   STATUS    RESTARTS   AGE
virt-launcher-vmi-debug-tools-fk64q   3/3     Running   0          64s
$ kubectl  logs virt-launcher-vmi-debug-tools-fk64q
[..]
{"component":"virt-launcher","level":"info","msg":"QEMU_MONITOR_RECV_EVENT: mon=0x7faa8801f5d0 event={\"timestamp\": {\"seconds\": 1698324640, \"microseconds\": 523652}, \"event\": \"NIC_RX_FILTER_CHANGED\", \"data\": {\"name\": \"ua-default\", \"path\": \"/machine/peripheral/ua-default/virtio-backend\"}}","pos":"qemuMonitorJSONIOProcessLine:205","subcomponent":"libvirt","thread":"80","timestamp":"2023-10-26T12:50:40.523000Z"}
{"component":"virt-launcher","level":"info","msg":"QEMU_MONITOR_RECV_EVENT: mon=0x7faa8801f5d0 event={\"timestamp\": {\"seconds\": 1698324644, \"microseconds\": 165626}, \"event\": \"VSERPORT_CHANGE\", \"data\": {\"open\": true, \"id\": \"channel0\"}}","pos":"qemuMonitorJSONIOProcessLine:205","subcomponent":"libvirt","thread":"80","timestamp":"2023-10-26T12:50:44.165000Z"}
[..]
{"component":"virt-launcher","level":"info","msg":"QEMU_MONITOR_RECV_EVENT: mon=0x7faa8801f5d0 event={\"timestamp\": {\"seconds\": 1698324646, \"microseconds\": 707666}, \"event\": \"RTC_CHANGE\", \"data\": {\"offset\": 0, \"qom-path\": \"/machine/unattached/device[8]\"}}","pos":"qemuMonitorJSONIOProcessLine:205","subcomponent":"libvirt","thread":"80","timestamp":"2023-10-26T12:50:46.708000Z"}
[..]
```

该注解从容器创建之时起就启用过滤器。不过，在某些情况下，你可能希望在容器和 libvirt 都已经启动之后，动态地更改日志级别。这时，[`virt-admin`](https://libvirt.org/manpages/virt-admin.html) 就能派上用场。

示例：
```console
$ kubectl get pods
NAME                                READY   STATUS    RESTARTS   AGE
virt-launcher-vmi-ephemeral-nqcld   3/3     Running   0          26m
$ kubectl  exec -ti virt-launcher-vmi-ephemeral-nqcld -- virt-admin  -c virtqemud:///session  daemon-log-filters  "1:libvirt 1:qemu 1:conf 1:security 3:event 3:json 3:file 3:object 1:util"
$ kubectl  exec -ti virt-launcher-vmi-ephemeral-nqcld -- virt-admin  -c virtqemud:///session  daemon-log-filters
 Logging filters: 1:*libvirt* 1:*qemu* 1:*conf* 1:*security* 3:*event* 3:*json* 3:*file* 3:*object* 1:*util*
```

或者，如果你更希望把输出重定向到文件并在稍后获取它，可以依靠 `kubectl cp` 来取回该文件。在这种情况下，我们把文件保存在 `/var/run/libvirt` 目录中，因为 compute 容器拥有向该目录写入的权限。

示例：
```console
$ kubectl get pods
NAME                                READY   STATUS    RESTARTS   AGE
virt-launcher-vmi-ephemeral-nqcld   3/3     Running   0          26m
$ kubectl  exec -ti virt-launcher-vmi-ephemeral-nqcld -- virt-admin  -c virtqemud:///session daemon-log-outputs "1:file:/var/run/libvirt/libvirtd.log"
$ kubectl cp virt-launcher-vmi-ephemeral-nqcld:/var/run/libvirt/libvirtd.log libvirt-kubevirt.log
tar: Removing leading `/' from member names
```
