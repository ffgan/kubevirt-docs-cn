# 热插拔网络接口 {#hotplug-network-interfaces}

版本：

- v1.1.0: Alpha
- v1.3.0: Beta
- v1.4.0: GA

KubeVirt 支持向正在运行的虚拟机（VM）热插入和热拔出网络接口。

热插拔支持通过 [bridge 绑定](http://kubevirt.io/api-reference/main/definitions.html#_v1_interfacebridge) 或 [SR-IOV 绑定](http://kubevirt.io/api-reference/main/definitions.html#_v1_interfacesriov) 连接、并且使用 `virtio` 模型的接口。

热拔出仅支持通过 [bridge 绑定](http://kubevirt.io/api-reference/main/definitions.html#_v1_interfacebridge) 连接的接口。

## 要求 {#requirements}

向 KubeVirt 虚拟机添加接口，首先需要向正在运行的 Pod 添加接口。这并不简单，并且有一些要求：

- [Multus Dynamic Networks Controller](https://github.com/k8snetworkplumbingwg/multus-dynamic-networks-controller)：该守护进程会监听注解变更，并触发 Multus 为该 Pod 配置一个新的附加网络。
- 以[厚插件](https://github.com/k8snetworkplumbingwg/multus-cni/blob/master/docs/thick-plugin.md)方式运行的 Multus CNI：这一版 Multus 会暴露一个端点，以便按需为指定 Pod 创建附加网络。

> **注意**：对于较旧的 KubeVirt 版本（从 v1.1 到 v1.3），必须启用 `HotplugNICs` [功能门控](https://kubevirt.io/user-guide/operations/activating_feature_gates/#how-to-activate-a-feature-gate))。从 KubeVirt v1.4 起，不再需要该 FG；如果已经设置，应当将其移除。

## 向正在运行的 VM 添加接口 {#adding-an-interface-to-a-running-vm}

首先启动一个 VM。可以参考下面的示例：
```yaml
---
apiVersion: kubevirt.io/v1
kind: VirtualMachine
metadata:
  name: vm-fedora
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
          interfaces:
          - masquerade: {}
            name: defaultnetwork
          rng: {}
        resources:
          requests:
            memory: 1024M
      networks:
      - name: defaultnetwork
        pod: {}
      terminationGracePeriodSeconds: 0
      volumes:
      - containerDisk:
          image: quay.io/kubevirt/fedora-with-test-tooling-container-disk:devel
        name: containerdisk
```

你应当配置一个网络附加定义，Pod 接口配置就保存在其中。下面的片段展示了一个非常简单的示例：
```yaml
apiVersion: k8s.cni.cncf.io/v1
kind: NetworkAttachmentDefinition
metadata:
  name: new-fancy-net
spec:
    config: '{
      "cniVersion": "0.3.1",
      "type": "bridge",
      "mtu": 1300,
      "name":"new-fancy-net"
    }'
```
请参阅 [Multus 文档](https://github.com/k8snetworkplumbingwg/multus-cni/blob/master/docs/how-to-use.md#create-network-attachment-definition)以了解更多信息。

一旦虚拟机正在运行，并且附加配置已经就绪，用户就可以通过编辑 VM spec 模板、添加所需的接口和网络，来请求接口热插拔操作：
```yaml
apiVersion: kubevirt.io/v1
kind: VirtualMachine
metadata:
  name: vm-fedora
template:
  spec:
    domain:
      devices:
        interfaces:
        - name: defaultnetwork
          masquerade: {}
          # new interface
        - name: dyniface1
          bridge: {}
    networks:
    - name: defaultnetwork
      pod: {}
      # new network
    - name: dyniface1
      multus:
        networkName: new-fancy-net
 ...
```
> **注意**：`virtctl` 的 `addinterface` 和 `removeinterface` 命令已不再可用，接口的热插拔/热拔出通过编辑 VM spec 模板来完成。

KubeVirt 也会把该接口和网络添加到对应的 VMI 对象中。

现在可以检查 VMI 状态，确认这个新接口是否存在：
```bash
kubectl get vmi vm-fedora -ojsonpath="{ @.status.interfaces }"
```

## 从正在运行的 VM 移除接口 {#removing-an-interface-from-a-running-vm}

沿用上面的示例，用户可以通过编辑 VM spec 模板，并把所需接口的状态设置为 `absent`，来请求接口热拔出操作：
```yaml
apiVersion: kubevirt.io/v1
kind: VirtualMachine
metadata:
  name: vm-fedora
template:
  spec:
    domain:
      devices:
        interfaces:
          - name: defaultnetwork
            masquerade: {}
          # set the interface state to absent 
          - name: dyniface1
            state: absent
            bridge: {}
    networks:
      - name: defaultnetwork
        pod: {}
      - name: dyniface1
        multus:
          networkName: new-fancy-net
```
KubeVirt 也会把对应 VMI 对象中该接口的状态设置为 'absent'。

> **注意**：v0.59.0 及更低版本的现有 VM 不支持接口热拔出。

## 基于迁移的热插拔 {#migration-based-hotplug}

如果集群中的 Multus 不是以[厚插件](https://github.com/k8snetworkplumbingwg/multus-cni/blob/master/docs/thick-plugin.md)方式运行，并且没有运行 [Multus Dynamic Networks controller](https://github.com/k8snetworkplumbingwg/multus-dynamic-networks-controller)，则可以通过迁移 VM 来热插拔接口。

> ⚠️ **注意**：从 KubeVirt v1.5 起（[kubevirt/kubevirt#13497](https://github.com/kubevirt/kubevirt/issues/13497)），命名空间管理员默认**不允许创建 VMIM**（VirtualMachineInstanceMigrations）。基于迁移的热插拔需要此权限。集群管理员必须显式授予该权限。

实际的附加不会立即发生；迁移完成后，新接口才会在客户机中可用。

> **注意**：从 KubeVirt v1.6 开始，如果在 KubeVirt 自定义资源（CR）上配置了 [LiveUpdate 滚动更新策略](../user_workloads/vm_rollout_strategies.md#liveupdate)，虚拟机（VM）会在网卡（NIC）热插拔或热拔出操作之后自动进行迁移。
>
> 在这些场景中，不再需要手动迁移 VM。

### 添加新接口 {#add-new-interface}

把所需的接口和网络添加到 VM spec 模板：
```yaml
apiVersion: kubevirt.io/v1
kind: VirtualMachine
metadata:
  name: vm-fedora
template:
  spec:
    domain:
      devices:
        interfaces:
        - name: defaultnetwork
          masquerade: {}
          # new interface
        - name: dyniface1
          bridge: {}
    networks:
    - name: defaultnetwork
      pod: {}
      # new network
    - name: dyniface1
      multus:
        networkName: new-fancy-net
 ...
```

此时，该接口和网络也会被添加到对应的 VMI 对象中，但不会附加到客户机。

#### 迁移 VM {#migrate-the-vm}
```bash
cat <<EOF kubectl apply -f -
apiVersion: kubevirt.io/v1
kind: VirtualMachineInstanceMigration
metadata:
  name: migration-job
spec:
  vmiName: vmi-fedora
EOF
```
更多信息请参阅[实时迁移](../compute/live_migration.md)文档。

迁移完成后，这个新接口将附加到 VM 上。

> **注意**：建议避免在热插拔操作的同时执行迁移。更稳妥的做法是，先确认热插拔已经成功，或者至少已经写入 VMI 规约，然后再发起迁移。

### 移除接口 {#remove-interface}

在 VM spec 模板中把所需接口的状态设置为 `absent`：
```yaml
apiVersion: kubevirt.io/v1
kind: VirtualMachine
metadata:
  name: vm-fedora
template:
  spec:
    domain:
      devices:
        interfaces:
          - name: defaultnetwork
            masquerade: {}
          # set the interface state to absent 
          - name: dyniface1
            state: absent
            bridge: {}
    networks:
      - name: defaultnetwork
        pod: {}
      - name: dyniface1
        multus:
          networkName: new-fancy-net
```

此时，目标接口应当已经从客户机分离，但仍然存在于 Pod 中。

> **注意**：v0.59.0 及更低版本的现有 VM 不支持接口热拔出。

#### 迁移 VM {#migrate-the-vm_1}
```bash
cat <<EOF kubectl apply -f -
apiVersion: kubevirt.io/v1
kind: VirtualMachineInstanceMigration
metadata:
  name: migration-job
spec:
  vmiName: vmi-fedora
EOF
```
更多信息请参阅[实时迁移](../compute/live_migration.md)文档。

VM 完成迁移后，该接口将不会存在于迁移目标 Pod 中。

> **注意**：建议避免在热拔出操作的同时执行迁移。更稳妥的做法是，先确认热拔出已经成功，或者至少已经写入 VMI 规约，然后再发起迁移。

### SR-IOV 接口 {#sr-iov-interfaces}

KubeVirt 支持向正在运行的 VM 热插拔 SR-IOV 接口。

与 bridge 绑定的接口类似，编辑 VM spec 模板，并添加所需的 SR-IOV 接口和网络：
```yaml
apiVersion: kubevirt.io/v1
kind: VirtualMachine
metadata:
  name: vm-fedora
template:
  spec:
    domain:
      devices:
        interfaces:
        - name: defaultnetwork
          masquerade: {}
          # new interface
        - name: sriov-net
          sriov: {}
    networks:
    - name: defaultnetwork
      pod: {}
      # new network
    - name: sriov-net
      multus:
        networkName: sriov-net-1
 ...
```
有关 SR-IOV 网络的更多信息，请参阅[接口与网络](../network/interfaces_and_networks.md/#sriov)文档。

此时，该接口和网络也会被添加到对应的 VMI 对象中，但不会附加到客户机。

#### 迁移 VM {#migrate-the-vm_2}
```bash
cat <<EOF kubectl apply -f -
apiVersion: kubevirt.io/v1
kind: VirtualMachineInstanceMigration
metadata:
  name: migration-job
spec:
  vmiName: vmi-fedora
EOF
```
更多信息请参阅[实时迁移](../compute/live_migration.md)文档。

VM 完成迁移后，该接口将不会存在于迁移目标 Pod 中。由于 Kubernetes 设备插件 API 无法动态分配资源，SR-IOV 设备插件不能为 KubeVirt 分配额外的 SR-IOV 资源来进行热插拔。因此，无论 Multus 是否为 “thick” 版本，SR-IOV 接口热插拔都仅限于基于迁移的热插拔。

## Virtio 限制 {#virtio-limitations}

热插拔的接口具有 `model: virtio`。这带来若干限制：每个接口都会占用 VM 中的一个 PCI 插槽，而总数最多为 32。此外，其他设备也会使用这些 PCI 插槽（例如磁盘、guest-agent 等）。

KubeVirt 会为 4 个接口预留资源，以便稍后执行热插拔操作。实际可用资源的上限取决于机器类型（例如 q35 会再增加一个 PCI 插槽）。有关上限的更多信息，请参阅 [libvirt 文档](https://libvirt.org/pci-hotplug.html)。

不过，在 VM 重启之后，热插拔的接口会成为标准网络的一部分；这可以缓解每种机器类型的最大热插拔接口数限制。

> **注意**：用户可以对已停止的 VM 执行此命令，也就是没有关联 VMI 的 VM。发生这种情况时，KubeVirt 会代表用户改写 VM spec 模板。

