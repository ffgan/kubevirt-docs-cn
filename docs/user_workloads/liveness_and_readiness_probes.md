# 存活探针与就绪探针 {#liveness-and-readiness-probes}

可以按与为[容器配置存活探针和就绪探针](https://kubernetes.io/docs/tasks/configure-pod-container/configure-liveness-readiness-probes/)类似的方式，配置存活探针和就绪探针。

存活探针一旦失败，就会停止该 VirtualMachineInstance，从而允许更高层的控制器（如 VirtualMachine 或 VirtualMachineInstanceReplicaSet）生成新的实例，这些实例有望重新恢复响应。

就绪探针向 Service 和端点表明 VirtualMachineInstance 是否已准备好接收来自 Service 的流量。如果就绪探针失败，VirtualMachineInstance 会从支撑这些服务的端点中移除，直到探针恢复。

看门狗专注于确保操作系统仍然能够响应。它们补充了更以工作负载为中心的探针。看门狗需要来自客户机的内核支持，以及像常用的 `watchdog` 二进制文件这样的额外工具。

Exec 探针是专门面向 VM 的存活探针或就绪探针。这些探针在 VM 内部运行一条命令，并根据其是否成功来判断 VM 的就绪/存活状态。要在 VM 内部运行命令，会使用 qemu-guest-agent 软件包。提供给 exec 探针的命令会在 operator 中由 `virt-probe` 包装，并转发到客户机。

## 定义 HTTP 存活探针 {#define-a-http-liveness-probe}

下面的 VirtualMachine 通过 `spec.template.spec.livenessProbe.httpGet` 配置 HTTP 存活探针，它会在初始延迟 120 秒之后查询该 VirtualMachine 的 1500 端口。VirtualMachine 自身通过 cloud-init 在 1500 端口上安装并运行一个最小的 HTTP 服务器。

```yaml
apiVersion: kubevirt.io/v1
kind: VirtualMachine
metadata:
  name: example-http-liveness-probe
spec:
  runStrategy: Always
  template:
    spec:
      domain:
        devices:
          disks:
          - disk:
              bus: virtio
            name: containerdisk
          - disk:
              bus: virtio
            name: cloudinit
          rng: {}
        resources:
          requests:
            memory: 1Gi
      livenessProbe:
        initialDelaySeconds: 120
        periodSeconds: 20
        httpGet:
          port: 1500
        timeoutSeconds: 10
      terminationGracePeriodSeconds: 0
      volumes:
      - name: containerdisk
        containerDisk:
          image: quay.io/containerdisks/centos-stream:9
      - name: cloudinit
        cloudInitNoCloud:
          userData: |-
            #cloud-config
            password: centos
            user: centos
            chpasswd: { expire: False }
            bootcmd:
              - ["sudo", "dnf", "install", "-y", "nmap-ncat"]
              - ["sudo", "systemd-run", "--unit=httpserver", "nc", "-klp", "1500", "-e", '/usr/bin/echo -e HTTP/1.1 200 OK\\nContent-Length: 12\\n\\nHello World!']
```

## 定义 TCP 存活探针 {#define-a-tcp-liveness-probe}

下面的 VirtualMachine 通过 `spec.template.spec.livenessProbe.tcpSocket` 配置 TCP 存活探针，它会在初始延迟 120 秒之后查询该 VirtualMachine 的 1500 端口。VirtualMachine 自身通过 cloud-init 在 1500 端口上安装并运行一个最小的 HTTP 服务器。

```yaml
apiVersion: kubevirt.io/v1
kind: VirtualMachine
metadata:
  name: example-tcp-liveness-probe
spec:
  runStrategy: Always
  template:
    spec:
      domain:
        devices:
          disks:
          - disk:
              bus: virtio
            name: containerdisk
          - disk:
              bus: virtio
            name: cloudinit
          rng: {}
        resources:
          requests:
            memory: 1Gi
      livenessProbe:
        initialDelaySeconds: 120
        periodSeconds: 20
        tcpSocket:
          port: 1500
        timeoutSeconds: 10
      terminationGracePeriodSeconds: 0
      volumes:
      - name: containerdisk
        containerDisk:
          image: quay.io/containerdisks/centos-stream:9
      - name: cloudinit
        cloudInitNoCloud:
          userData: |-
            #cloud-config
            password: centos
            user: centos
            chpasswd: { expire: False }
            bootcmd:
              - ["sudo", "dnf", "install", "-y", "nmap-ncat"]
              - ["sudo", "systemd-run", "--unit=httpserver", "nc", "-klp", "1500", "-e", '/usr/bin/echo -e HTTP/1.1 200 OK\\nContent-Length: 12\\n\\nHello World!']
```

## 定义就绪探针 {#define-readiness-probes}

就绪探针的配置方式与存活探针类似。所要填写的不是 `spec.template.spec.livenessProbe`，而是 `spec.template.spec.readinessProbe`：

```yaml
apiVersion: kubevirt.io/v1
kind: VirtualMachine
metadata:
  name: example-readiness-probe
spec:
  runStrategy: Always
  template:
    spec:
      domain:
        devices:
          disks:
          - disk:
              bus: virtio
            name: containerdisk
          - disk:
              bus: virtio
            name: cloudinit
          rng: {}
        resources:
          requests:
            memory: 1Gi
      readinessProbe:
        initialDelaySeconds: 120
        periodSeconds: 20
        timeoutSeconds: 10
        failureThreshold: 3
        successThreshold: 3
        httpGet:
          port: 1500
      terminationGracePeriodSeconds: 0
      volumes:
      - name: containerdisk
        containerDisk:
          image: quay.io/containerdisks/centos-stream:9
      - name: cloudinit
        cloudInitNoCloud:
          userData: |-
            #cloud-config
            password: centos
            user: centos
            chpasswd: { expire: False }
            bootcmd:
              - ["sudo", "dnf", "install", "-y", "nmap-ncat"]
              - ["sudo", "systemd-run", "--unit=httpserver", "nc", "-klp", "1500", "-e", '/usr/bin/echo -e HTTP/1.1 200 OK\\nContent-Length: 12\\n\\nHello World!']
```

请注意，对于就绪探针，还可以设置 `failureThreshold` 和 `successThreashold`，从而仅在探针多次成功或多次失败时，才在就绪与未就绪状态之间切换。

## 双栈方面的考虑 {#dual-stack-considerations}

要理解双栈网络配置对就绪探针或存活探针所施加的限制，需要一些背景知识。用户必须充分了解，双栈配置目前仅在使用 masquerade 绑定类型时可用。此外必须记住，使用 masquerade 绑定类型访问 VM 是通过 Pod IP 地址进行的；在双栈模式下，IPv4 和 IPv6 地址都可以用来访问该 VM。

双栈网络配置在使用 HTTP / TCP 探针时存在一项限制——你**不能通过 VMI 的 IPv6 地址来探测它**。原因在于 [HTTP](https://kubernetes.io/docs/reference/generated/kubernetes-api/v1.23/#httpgetaction-v1-core) 和 [TCP](https://kubernetes.io/docs/reference/generated/kubernetes-api/v1.23/#tcpsocketaction-v1-core) 探针动作的 `host` 字段默认都是 Pod 的 IP 地址，而该地址目前始终是 IPv4 地址。

由于在创建 VMI 之前并不知道 Pod 的 IP 地址，因此无法预先填写探针的 host 字段。

## 定义看门狗 {#defining-a-watchdog}

看门狗是一种更以 VM 为中心的方法，关注的是操作系统的响应能力。可以配置 `i6300esb` 看门狗设备：

```yaml
apiVersion: kubevirt.io/v1
kind: VirtualMachine
metadata:
  name: example-watchdog
spec:
  runStrategy: Always
  template:
    spec:
      domain:
        devices:
          watchdog:
            name: mywatchdog
            i6300esb:
              action: "poweroff"
          disks:
          - disk:
              bus: virtio
            name: containerdisk
          - disk:
              bus: virtio
            name: cloudinit
          rng: {}
        resources:
          requests:
            memory: 1Gi
      terminationGracePeriodSeconds: 0
      volumes:
      - name: containerdisk
        containerDisk:
          image: quay.io/containerdisks/fedora:latest
      - name: cloudinit
        cloudInitNoCloud:
          userData: |-
            #cloud-config
            password: fedora
            user: fedora
            chpasswd: { expire: False }
            packages:
                busybox
```

上面的示例使用 `poweroff` 动作来配置它。它定义了操作系统无法再作出响应时会发生什么。其他可能的动作是 `reset` 和 `shutdown`。此示例中的 VM 会把该设备暴露为 `/dev/watchdog`。随后 `watchdog` 二进制文件可以使用这个设备。例如，如果 root 在 VM 内部执行这条命令：

```bash
sudo busybox watchdog -t 2000ms -T 4000ms /dev/watchdog
```

看门狗会每两秒向 `/dev/watchdog` 发送一次心跳，并且在四秒没有心跳之后执行所定义的动作。在本例中是硬 `poweroff`。

## 定义客户机代理 Ping 探针 {#defining-guest-agent-ping-probes}

客户机代理探针基于 qemu-guest-agent 的 `guest-ping`。这会 ping 客户机，如果客户机没有处于启动并运行的状态，则返回错误。要在 VM spec 上方便地定义这一点，请指定 `guestAgentPing: {}`，并把它放在 VM 的 `spec.template.spec.readinessProbe` 中。`virt-controller` 会把这转换成由 `virt-probe` 包装的相应命令。

> 注意：你只能定义其中一种探针类型，即客户机代理 exec
> 探针或 ping 探针。


**重要：** 如果 qemu-guest-agent 没有在 VM 内部安装**并且**启用，探针就会失败。许多镜像默认并不启用该代理，因此请确保你运行的是已经启用它的镜像，或者自行启用它。

请确保为 VM 和该代理上线提供足够的延迟和 failureThreshold。

在下面的示例中，Fedora 镜像默认就提供 qemu-guest-agent。尽管如此，如果 qemu-guest-agent 尚未安装，也会像下面的示例所示通过 cloud-init 安装并启用。此外，cloud-init 会为 `/tmp/healthy.txt` 文件分配正确的 SELinux 上下文，即 virt_qemu_ga_exec_t。否则，SELinux 会拒绝打开 `/tmp/healthy.txt` 文件的尝试，从而导致探针失败。

```yaml
apiVersion: kubevirt.io/v1
kind: VirtualMachine
metadata:
  name: example-agent-readiness-probe
spec:
  runStrategy: Always
  template:
    spec:
      domain:
        devices:
          disks:
            - name: containerdisk
              disk:
                bus: virtio
            - name: cloudinitdisk
              disk:
                bus: virtio
          rng: {}
        resources:
          requests:
            memory: 1Gi
      readinessProbe:
        exec:
          command: ["cat", "/tmp/healthy.txt"]
        failureThreshold: 10
        initialDelaySeconds: 120
        periodSeconds: 10
        timeoutSeconds: 5
      terminationGracePeriodSeconds: 180
      volumes:
        - name: containerdisk
          containerDisk:
            image: quay.io/containerdisks/fedora
        - name: cloudinitdisk
          cloudInitNoCloud:
            userData: |
              #cloud-config
              chpasswd:
                expire: false
              password: password
              user: fedora
              packages:
                qemu-guest-agent
              runcmd:
                - ["touch", "/tmp/healthy.txt"]
                - ["sudo", "chcon", "--type", "virt_qemu_ga_exec_t", "/tmp/healthy.txt"]
                - ["sudo", "systemctl", "enable", "--now", "qemu-guest-agent"]
```

请注意，在上面的示例中，如果你的容器磁盘镜像中没有安装 SELinux，则应从所示的 VM 清单中移除 `chcon` 命令。否则，`chcon` 命令会失败。

`.status.ready` 字段会切换为 `true`，表明探针正在成功返回：

```sh
kubectl wait vmis/vmi-guest-probe --for=condition=Ready --timeout=5m
```

此外，可以在 VM 内部使用下面的命令来观察传入的 qemu-ga 命令：

```sh
journalctl _COMM=qemu-ga --follow
```

## 暂停客户机代理 Ping 探针 {#pausing-guest-agent-ping-probes}

在客户机操作系统更新（尤其是伴随多次重启的长时间 Windows 更新）等维护操作期间，客户机代理可能会暂时不可用。如果配置了 GuestAgentPing 存活探针，这会导致探针失败，并使 kubelet 重启 Pod，从而销毁正在运行的 VM。

为防止这种情况，可以通过向 VMI 添加注解来暂停 GuestAgentPing 探针：

```bash
# Pause probes before maintenance
kubectl annotate vmi my-vm kubevirt.io/pause-guest-agent-probes=true

# Resume probes after maintenance
kubectl annotate vmi my-vm kubevirt.io/pause-guest-agent-probes-
```

当该注解被设置为真值（`"true"`、`"1"`、`"t"` 等，即 Go 的 `strconv.ParseBool` 所接受的值）时，virt-launcher 会立即为 GuestAgentPing 探针返回成功，而不会联系 QEMU 客户机代理。移除该注解（或将其设置为假值）会恢复正常的探针行为。

要验证 virt-launcher 已经读取该注解，请在其日志中检查状态转换消息：

```bash
kubectl logs -n <namespace> virt-launcher-my-vm-<id> -c compute | grep "probe pause state"
```

!!! important
    - 此注解只影响 GuestAgentPing 探针。HTTP、TCP 和 exec
      探针不受影响。
    - 该注解会在数秒内生效（在下一次 virt-launcher
      同步周期中，由 virt-handler 触发）。
    - 若要永久禁用探针，请从 VMI spec 中移除该探针，
      而不是让该注解无限期地保持设置。
    - 针对瞬态的内部探针抑制（例如实时迁移、
      VM 暂停）会自动处理，并且独立于该注解。
      将该注解设置为 `"false"` 或将其移除，并不会覆盖
      内部抑制——在这些操作期间，无论该注解的取值如何，探针都会保持被抑制。
