# DNS 记录 {#dns-records}

为了给每个 VirtualMachineInstance 创建唯一的 DNS 记录，可以设置 `spec.hostname` 和 `spec.subdomain`。如果设置了子域，并且存在一个名称与该子域匹配的无头 Service，kube-dns 就会为每一个匹配该 Service 选择器的 VirtualMachineInstance 创建唯一的 DNS 条目。更多信息请参阅 [Service 与 Pod 的 DNS 文档](https://kubernetes.io/docs/concepts/services-networking/dns-pod-service/#pods-hostname-and-subdomain-fields)。

下面的示例包含一个 VirtualMachine，以及一个与该 VirtualMachineInstance 的标签和子域相匹配的无头 Service：

```yaml
apiVersion: kubevirt.io/v1
kind: VirtualMachineInstance
metadata:
  name: vmi-fedora
  labels:
    expose: me
spec:
  hostname: "myvmi"
  subdomain: "mysubdomain"
  domain:
    devices:
      disks:
      - disk:
          bus: virtio
        name: containerdisk
      - disk:
          bus: virtio
        name: cloudinitdisk
    resources:
      requests:
        memory: 1024M
  terminationGracePeriodSeconds: 0
  volumes:
  - name: containerdisk
    containerDisk:
      image: kubevirt/fedora-cloud-registry-disk-demo:latest
  - cloudInitNoCloud:
      userDataBase64: IyEvYmluL2Jhc2gKZWNobyAiZmVkb3JhOmZlZG9yYSIgfCBjaHBhc3N3ZAo=
    name: cloudinitdisk
---
apiVersion: v1
kind: Service
metadata:
  name: mysubdomain
spec:
  selector:
    expose: me
  clusterIP: None
  ports:
  - name: foo # Actually, no port is needed.
    port: 1234
    targetPort: 1234
```

因此，例如当我们通过 `virtctl console vmi-fedora` 进入 VirtualMachineInstance，并对 `myvmi.mysubdomain` 执行 ping 时，会看到存在一条 DNS 条目 `myvmi.mysubdomain.default.svc.cluster.local`，它指向 `10.244.0.57`，该地址是 VirtualMachineInstance 的 IP（不是 Service 的 IP）：

    [fedora@myvmi ~]$ ping myvmi.mysubdomain
    PING myvmi.mysubdomain.default.svc.cluster.local (10.244.0.57) 56(84) bytes of data.
    64 bytes from myvmi.mysubdomain.default.svc.cluster.local (10.244.0.57): icmp_seq=1 ttl=64 time=0.029 ms
    [fedora@myvmi ~]$ ip a
    2: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc fq_codel state UP group default qlen 1000
        link/ether 0a:58:0a:f4:00:39 brd ff:ff:ff:ff:ff:ff
        inet 10.244.0.57/24 brd 10.244.0.255 scope global dynamic eth0
           valid_lft 86313556sec preferred_lft 86313556sec
        inet6 fe80::858:aff:fef4:39/64 scope link
           valid_lft forever preferred_lft forever

于是，`spec.hostname` 和 `spec.subdomain` 会被转换成如下形式的 DNS A 记录：`<vmi.spec.hostname>.<vmi.spec.subdomain>.<vmi.metadata.namespace>.svc.cluster.local`。如果没有设置 `spec.hostname`，就会回退到 VirtualMachineInstance 自身的名称。此时得到的 DNS A 记录如下：`<vmi.metadata.name>.<vmi.spec.subdomain>.<vmi.metadata.namespace>.svc.cluster.local`。

> **注意** 要借助搜索域解析 `myvmi.mysubdomain` 这样的短名称，客户机的 `/etc/resolv.conf` 必须包含 `options ndots:2` 或更高的值。
> 在 Linux 客户机中，如果未指定 `options ndots`，系统默认使用 `ndots:1`，如 resolver spec](https://man7.org/linux/man-pages/man5/resolv.conf.5.html) 所示。
> KubeVirt 不会配置客户机系统的解析器。它可以预先配置在 VM 镜像中，也可以在运行时使用 cloud-init 进行配置。
