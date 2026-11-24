# 实例类型与偏好 {#instance-types-and-preferences}

**功能状态：**

* `instancetype.kubevirt.io/v1alpha1`（实验性），自 [`v0.56.0`](https://github.com/kubevirt/kubevirt/releases/tag/v0.56.0) KubeVirt 发行版起
* `instancetype.kubevirt.io/v1alpha2`（实验性），自 [`v0.58.0`](https://github.com/kubevirt/kubevirt/releases/tag/v0.58.0) KubeVirt 发行版起
* `instancetype.kubevirt.io/v1beta1`，自 [`v1.0.0`](https://github.com/kubevirt/kubevirt/releases/tag/v1.0.0) KubeVirt 发行版起

更多细节请参见 [版本历史](#version-history) 一节。

## 简介 {#introduction}

KubeVirt 的 [`VirtualMachine`](https://kubevirt.io/api-reference/main/definitions.html#_v1_virtualmachine) API 包含许多用于调优 VM 性能的高级选项，超出了典型用户需要了解的范围。此前，用户无法在简单定义希望分配给其 VM 的存储/网络之后，再用概括性的术语声明该 VM 所需的资源质量与性能特征。

实例类型和偏好提供了一种方式，用来定义一组资源、性能以及其他运行时特征，使用户可以在多个 `VirtualMachines` 之间复用这些定义。

## VirtualMachineInstancetype {#virtualmachineinstancetype}

```yaml
---
apiVersion: instancetype.kubevirt.io/v1beta1
kind: VirtualMachineInstancetype
metadata:
  name: example-instancetype
spec:
  cpu:
    guest: 1
  memory:
    guest: 128Mi
```

KubeVirt 为实例类型提供了两个 [`CRDs`](https://kubernetes.io/docs/concepts/extend-kubernetes/api-extension/custom-resources/)，一个是集群范围的 [`VirtualMachineClusterInstancetype`](https://kubevirt.io/api-reference/main/definitions.html#_v1beta1_virtualmachineclusterinstancetype)，另一个是命名空间范围的 [`VirtualMachineInstancetype`](https://kubevirt.io/api-reference/main/definitions.html#_v1beta1_virtualmachineinstancetype)。这些 `CRDs` 封装了 `VirtualMachine` 的以下资源相关特征，这是通过共享的 [`VirtualMachineInstancetypeSpec`](https://kubevirt.io/api-reference/main/definitions.html#_v1beta1_virtualmachineinstancetypespec) 实现的：

* [`CPU`](https://kubevirt.io/api-reference/main/definitions.html#_v1beta1_cpuinstancetype) ：呈现给客户机的所需 vCPU 数量
* [`Memory`](https://kubevirt.io/api-reference/main/definitions.html#_v1beta1_memoryinstancetype) ：呈现给客户机的所需内存量
* [`GPUs`](https://kubevirt.io/api-reference/main/definitions.html#_v1_gpu) ：要直通的可选 vGPU 列表
* [`HostDevices`](https://kubevirt.io/api-reference/main/definitions.html#_v1_hostdevice) ：要直通的可选 `HostDevices` 列表
* `IOThreadsPolicy` ：要使用的可选 `IOThreadsPolicy`
* [`LaunchSecurity`](https://kubevirt.io/api-reference/main/definitions.html#_v1_launchsecurity)：要使用的可选 `LaunchSecurity`

实例类型中提供的任何内容都不能在 `VirtualMachine` 中被覆盖。例如，由于 `CPU` 和 `Memory` 都是实例类型的必填属性，如果用户对 `CPU` 或 `Memory` 资源提出任何请求，而这些请求位于底层 `VirtualMachine` 之中，就会与实例类型冲突，该请求会在创建时被拒绝。

## VirtualMachinePreference {#virtualmachinepreference}

```yaml
---
apiVersion: instancetype.kubevirt.io/v1beta1
kind: VirtualMachinePreference
metadata:
  name: example-preference
spec:
  devices:
    preferredDiskBus: virtio
    preferredInterfaceModel: virtio
```

KubeVirt 还另外提供了两个基于偏好的 `CRDs`，同样是一个集群范围的 [`VirtualMachineClusterPreference`](https://kubevirt.io/api-reference/main/definitions.html#_v1beta1_virtualmachineclusterpreference) 和一个命名空间范围的 [`VirtualMachinePreference`](https://kubevirt.io/api-reference/main/definitions.html#_v1beta1_virtualmachinepreference)。这些 `CRDs` 封装了运行给定工作负载所需的、`VirtualMachine` 其余任何属性的首选值，同样是通过共享的 [`VirtualMachinePreferenceSpec`](https://kubevirt.io/api-reference/main/definitions.html#_v1beta1_virtualmachinepreferencespec) 实现的。

与实例类型不同，偏好只表示首选值，因此可以被用户在 `VirtualMachine` 中提供的值覆盖。

在下面的示例中，用户提供了一个 `VirtualMachine`，其中磁盘总线已经在 [`DiskTarget`](https://kubevirt.io/api-reference/main/definitions.html#_v1_disktarget) 内定义，*并且*还选择了一组带有 [`DevicePreference`](https://kubevirt.io/api-reference/main/definitions.html#_v1beta1_devicepreferences) 和 `preferredDiskBus` 的偏好，因此会使用用户在 `VirtualMachine` 与 `DiskTarget` 中的原始选择：

```yaml
$ kubectl apply -f - << EOF
---
apiVersion: instancetype.kubevirt.io/v1beta1
kind: VirtualMachinePreference
metadata:
  name: example-preference-disk-virtio
spec:
  devices:
    preferredDiskBus: virtio
---
apiVersion: kubevirt.io/v1
kind: VirtualMachine
metadata:
  name: example-preference-user-override
spec:
  preference:
    kind: VirtualMachinePreference
    name: example-preference-disk-virtio
  runStrategy: Halted
  template:
    spec:
      domain:
        memory:
          guest: 128Mi
        devices:
          disks:
          - disk:
              bus: sata
            name: containerdisk
          - disk: {}
            name: cloudinitdisk
        resources: {}
      terminationGracePeriodSeconds: 0
      volumes:
      - containerDisk:
          image: registry:5000/kubevirt/cirros-container-disk-demo:devel
        name: containerdisk
      - cloudInitNoCloud:
          userData: |
            #!/bin/sh

            echo 'printed from cloud-init userdata'
        name: cloudinitdisk
EOF
virtualmachinepreference.instancetype.kubevirt.io/example-preference-disk-virtio created
virtualmachine.kubevirt.io/example-preference-user-override configured


$ virtctl start example-preference-user-override
VM example-preference-user-override was scheduled to start

# We can see the original request from the user within the VirtualMachine lists `containerdisk` with a `SATA` bus
$ kubectl get vms/example-preference-user-override -o json | jq .spec.template.spec.domain.devices.disks
[
  {
    "disk": {
      "bus": "sata"
    },
    "name": "containerdisk"
  },
  {
    "disk": {},
    "name": "cloudinitdisk"
  }
]

# This is still the case in the VirtualMachineInstance with the remaining disk using the `preferredDiskBus` from the preference of `virtio`
$ kubectl get vmis/example-preference-user-override -o json | jq .spec.domain.devices.disks
[
  {
    "disk": {
      "bus": "sata"
    },
    "name": "containerdisk"
  },
  {
    "disk": {
      "bus": "virtio"
    },
    "name": "cloudinitdisk"
  }
]
```

### PreferredCPUTopology {#preferredcputopology}

偏好可以选择包含 `PreferredCPUTopology`，它定义如何根据实例类型提供的 vCPU 构建 `VirtualMachineInstance` 对客户机可见的 CPU 拓扑。

```yaml
apiVersion: instancetype.kubevirt.io/v1beta1
kind: VirtualMachinePreference
metadata:
  name: example-preference-cpu-topology
spec:
  cpu:
    preferredCPUTopology: cores
```

`PreferredCPUTopology` 的允许取值包括：

* `sockets`（默认）- 以插槽形式向客户机提供 vCPU
* `cores` - 以核的形式向客户机提供 vCPU
* `threads` - 以线程的形式向客户机提供 vCPU
* `spread` - 默认把 vCPU 分散到插槽与核上。更多细节见以下 [SpreadOptions](#### SpreadOptions) 一节。
* `any` - 以插槽形式向客户机提供 vCPU，这也用于表示该偏好要求以任意方式分配 vCPU。在定义不与实例类型一起使用的偏好时很有用。

请注意，对原有 `preferSockets`、`preferCores`、`preferThreads` 和 `preferSpread` 这些 `PreferredCPUTopology` 取值的支持自 `v1.4.0` 起弃用，并将在未来的发行版中移除。

#### SpreadOptions {#spreadoptions}

当把 `spread` 作为 `PreferredCPUTopology` 的值时，可以使用 `SpreadOptions` 进一步自定义 vCPU 如何分散到客户机可见的 CPU 拓扑上：

```yaml
apiVersion: instancetype.kubevirt.io/v1beta1
kind: VirtualMachinePreference
metadata:
  name: example-preference-cpu-topology
spec:
  cpu:
    preferredCPUTopology: spread
    spreadOptions:
      across: SocketsCoresThreads
      ratio: 4
```

[`spreadOptions`](https://kubevirt.io/api-reference/main/definitions.html#_v1beta1_spreadoptions) 提供以下可配置项：

* `across` - 定义 vCPU 应如何分散到客户机可见的 CPU 拓扑上
* `ratio`  - 定义分散 vCPU 时所采用的比例（默认为 2）

`across` 的允许取值包括：

* `SocketsCores`（默认）- 按 1:N 的比例把 vCPU 分散到插槽与核上，其中 N 为所提供的比例。
* `SocketsCoresThreads` - 按 1:N:2 的比例把 vCPU 分散到插槽、核与线程上，其中 N 为所提供的比例。
* `CoresThreads` - 按强制的 1:2 比例把 vCPU 分散到核与线程上。（要求实例类型至少提供 2 个 vCPU）

## VirtualMachine {#virtualmachine}

```yaml
---
apiVersion: kubevirt.io/v1
kind: VirtualMachine
metadata:
  name: example-vm
spec:
  instancetype:
    kind: VirtualMachineInstancetype
    name: example-instancetype
  preference:
    kind: VirtualMachinePreference
    name: example-preference
```

先前的实例类型与偏好 CRD 通过匹配器与给定的 `VirtualMachine` 对应。每个匹配器包含以下内容：

* `Name`（string）：被引用资源的名称
* `Kind`（string）：可选，若未提供，则默认为集群范围的 CRD kind `VirtualMachineClusterInstancetype` 或 `VirtualMachineClusterPreference`
* `RevisionName`（string）：可选，一个 [`ControllerRevision`](https://kubernetes.io/docs/reference/kubernetes-api/workload-resources/controller-revision-v1/) 的名称，其中包含 `VirtualMachineInstancetypeSpec` 或 `VirtualMachinePreferenceSpec` 的副本，该副本是在 `VirtualMachine` 首次创建时取得的。关于如何以及为何捕获该副本的更多细节，见下方 [版本管理](#versioning) 一节。
* `InferFromVolume`（string）：可选，更多细节见下方 [从卷推断默认值](#inferring-defaults-from-a-volume) 一节。

## 创建实例类型、偏好与 VirtualMachine {#creating-instancetypes-preferences-and-virtualmachines}

可以使用 virtctl 命令行工具来简化实例类型、偏好和虚拟机的创建。更多内容请参见 [创建 VirtualMachine](../user_workloads/creating_vms.md#creating-virtualmachines)。

## 版本管理 {#versioning}

### KubeVirt <= v1.4.0 {#kubevirt-v140}

对这些资源进行版本管理是必需的，以确保最终创建的 `VirtualMachineInstance` 在启动 `VirtualMachine` 时不会在各次重启之间发生变化，即使在该 `VirtualMachine` 的生命周期内更新了所引用的实例类型或偏好集。

目前这是通过使用 `ControllerRevision` 来保留 `VirtualMachineInstancetype` 或 `VirtualMachinePreference` 的副本实现的，该副本取自 `VirtualMachine` 创建之时。对这些 `ControllerRevisions` 的引用随后会保留在 [`InstancetypeMatcher`](https://kubevirt.io/api-reference/main/definitions.html#_v1_instancetypematcher) 和 [`PreferenceMatcher`](https://kubevirt.io/api-reference/main/definitions.html#_v1_preferencematcher) 中，位于 `VirtualMachine` 之内，供以后使用。

```yaml
$ kubectl apply -f examples/csmall.yaml -f examples/vm-cirros-csmall.yaml
virtualmachineinstancetype.instancetype.kubevirt.io/csmall created
virtualmachine.kubevirt.io/vm-cirros-csmall created

$ kubectl get vm/vm-cirros-csmall -o json | jq .spec.instancetype
{
  "kind": "VirtualMachineInstancetype",
  "name": "csmall",
  "revisionName": "vm-cirros-csmall-csmall-72c3a35b-6e18-487d-bebf-f73c7d4f4a40-1"
}

$ kubectl get controllerrevision/vm-cirros-csmall-csmall-72c3a35b-6e18-487d-bebf-f73c7d4f4a40-1 -o json | jq .
{
  "apiVersion": "apps/v1",
  "data": {
    "apiVersion": "instancetype.kubevirt.io/v1beta1",
    "kind": "VirtualMachineInstancetype",
    "metadata": {
      "creationTimestamp": "2022-09-30T12:20:19Z",
      "generation": 1,
      "name": "csmall",
      "namespace": "default",
      "resourceVersion": "10303",
      "uid": "72c3a35b-6e18-487d-bebf-f73c7d4f4a40"
    },
    "spec": {
      "cpu": {
        "guest": 1
      },
      "memory": {
        "guest": "128Mi"
      }
    }
  },
  "kind": "ControllerRevision",
  "metadata": {
    "creationTimestamp": "2022-09-30T12:20:19Z",
    "name": "vm-cirros-csmall-csmall-72c3a35b-6e18-487d-bebf-f73c7d4f4a40-1",
    "namespace": "default",
    "ownerReferences": [
      {
        "apiVersion": "kubevirt.io/v1",
        "blockOwnerDeletion": true,
        "controller": true,
        "kind": "VirtualMachine",
        "name": "vm-cirros-csmall",
        "uid": "5216527a-1d31-4637-ad3a-b640cb9949a2"
      }
    ],
    "resourceVersion": "10307",
    "uid": "a7bc784b-4cea-45d7-8432-15418e1dd7d3"
  },
  "revision": 0
}


$ kubectl delete vm/vm-cirros-csmall
virtualmachine.kubevirt.io "vm-cirros-csmall" deleted

$ kubectl get controllerrevision/controllerrevision/vm-cirros-csmall-csmall-72c3a35b-6e18-487d-bebf-f73c7d4f4a40-1
Error from server (NotFound): controllerrevisions.apps "vm-cirros-csmall-csmall-72c3a35b-6e18-487d-bebf-f73c7d4f4a40-1" not found
```

用户可以选择改用更新一代的实例类型或偏好，做法是移除所引用的 `revisionName`，它位于 `VirtualMachine` 对象内相应的匹配器中。这会导致捕获并使用新的 `ControllerRevisions`。

下面的示例先使用 csmall 实例类型的初始版本创建 `VirtualMachine`，然后再增加该实例类型提供的 vCPU 数量：

```yaml
$ kubectl apply -f examples/csmall.yaml -f examples/vm-cirros-csmall.yaml
virtualmachineinstancetype.instancetype.kubevirt.io/csmall created
virtualmachine.kubevirt.io/vm-cirros-csmall created

$ kubectl get vm/vm-cirros-csmall -o json | jq .spec.instancetype
{
  "kind": "VirtualMachineInstancetype",
  "name": "csmall",
  "revisionName": "vm-cirros-csmall-csmall-3e86e367-9cd7-4426-9507-b14c27a08671-1"
}

$ virtctl start vm-cirros-csmall
VM vm-cirros-csmall was scheduled to start

$ kubectl get vmi/vm-cirros-csmall -o json | jq .spec.domain.cpu
{
  "cores": 1,
  "model": "host-model",
  "sockets": 1,
  "threads": 1
}

$ kubectl patch VirtualMachineInstancetype/csmall --type merge -p '{"spec":{"cpu":{"guest":2}}}'
virtualmachineinstancetype.instancetype.kubevirt.io/csmall patched
```

为了让这一更改在 `VirtualMachine` 中生效，我们需要停止正在运行的 `VirtualMachine`，并清除所引用的 `revisionName`，它由 `InstancetypeMatcher` 引用：

```yaml
$ virtctl stop vm-cirros-csmall
VM vm-cirros-csmall was scheduled to stop

$ kubectl patch vm/vm-cirros-csmall --type merge -p '{"spec":{"instancetype":{"revisionName":""}}}'
virtualmachine.kubevirt.io/vm-cirros-csmall patched

$ kubectl get vm/vm-cirros-csmall -o json | jq .spec.instancetype
{
  "kind": "VirtualMachineInstancetype",
  "name": "csmall",
  "revisionName": "vm-cirros-csmall-csmall-3e86e367-9cd7-4426-9507-b14c27a08671-2"
}
```

如上所示，`InstancetypeMatcher` 现在引用一份新的 `ControllerRevision`，其中包含该实例类型的第 2 代。现在可以再次启动 `VirtualMachine`，并看到 `VirtualMachineInstance` 正在使用的新 vCPU 数量：

```yaml
$ virtctl start vm-cirros-csmall
VM vm-cirros-csmall was scheduled to start

$ kubectl get vmi/vm-cirros-csmall -o json | jq .spec.domain.cpu
{
  "cores": 1,
  "model": "host-model",
  "sockets": 2,
  "threads": 1
}
```

### KubeVirt >= v1.5.0 {#kubevirt-v150}

自 v1.5.0 发行起，跟踪这些 ControllerRevision 引用的方法有所改变，以避免在提交之后改动 `VirtualMachine` 的核心 spec。这些引用现在存储在 `VirtualMachine` 的 status 之下：

```shell
$ virtctl create vm --preference fedora --instancetype u1.medium --name example | kubectl apply -f -
virtualmachine.kubevirt.io/example created

$ kubectl get vms/example -o json | jq .spec.instancetype,.spec.preference,.status.instancetypeRef,.status.preferenceRef
{
  "kind": "virtualmachineclusterinstancetype",
  "name": "u1.medium"
}
{
  "kind": "virtualmachineclusterpreference",
  "name": "fedora"
}
{
  "controllerRevisionRef": {
    "name": "example-u1.medium-v1beta1-0444161e-f8d6-4461-808d-5481dc4f3348-1"
  },
  "kind": "virtualmachineclusterinstancetype",
  "name": "u1.medium"
}
{
  "controllerRevisionRef": {
    "name": "example-fedora-v1beta1-1f7b1ffb-7311-4331-b663-4f7769d96816-1"
  },
  "kind": "virtualmachineclusterpreference",
  "name": "fedora"
}
```

> 注意：用户仍然可以通过原有的 `revisionName` 字段显式引用某个 ControllerRevision，
> 该值会被复制到 `VirtualMachine` 的 status 中并加以使用。从快照恢复或克隆
> `VirtualMachine` 时也会使用这种方式。

### InstancetypeReferencePolicy {#instancetypereferencepolicy}

**功能状态：**

* 自 [v1.4.0](https://github.com/kubevirt/kubevirt/releases/tag/v1.4.0) KubeVirt 发行版起为 Alpha（实验性）
* 自 [v1.5.0](https://github.com/kubevirt/kubevirt/releases/tag/v1.5.0) KubeVirt 发行版起为 Beta
* 自 [v1.6.0](https://github.com/kubevirt/kubevirt/releases/tag/v1.6.0) KubeVirt 发行版起为 GA

实例类型与偏好的版本管理及引用行为可通过以下受支持的策略进行配置：

* `reference`（默认）- 如上所述，关联的实例类型或偏好的副本存储在 ControllerRevision 中，并由 VirtualMachine 引用
* `expand` - 新建 VirtualMachine 所引用的任何实例类型或偏好都会直接展开到该 VirtualMachine 的 spec 中
* `expandAll` - 集群中任何 VirtualMachine 所引用的任何实例类型或偏好都会直接展开到该 VirtualMachine 的 spec 中

在 KubeVirt <= v1.5.0 中，必须先启用 `InstancetypeReferencePolicy` 功能门控，该功能才可使用：

```sh
kubectl patch -n kubevirt kv/kubevirt --type merge -p '{"spec":{"configuration":{"developerConfiguration":{"featureGates":["InstancetypeReferencePolicy"]}}}}'
```

随后可以使用可调参数 `spec.configuration.instancetype.referencePolicy` 来配置上述策略之一：

```sh
kubectl patch -n kubevirt kv/kubevirt --type merge -p '{"spec":{"configuration":{"instancetype":{"referencePolicy": "expand"}}}}'
```

如上所述，`expand` 策略会使 VirtualMachine 所引用的任何实例类型或偏好在所创建的对象中被直接展开：

```
 $ virtctl create vm --instancetype u1.medium --name test | kubectl apply -f -
virtualmachine.kubevirt.io/test created

$ kubectl get vm/test -o json | jq .spec.instancetype
null
```

而 `expandAll` 策略适用于任何已存在的 VirtualMachine：

```
$ kubectl patch -n kubevirt kv/kubevirt --type merge -p '{"spec":{"configuration":{"instancetype":{"referencePolicy": "reference"}}}}'

$ virtctl create vm --instancetype u1.medium --name test | kubectl apply -f -
virtualmachine.kubevirt.io/test created

$ kubectl get vm/test -o json | jq .spec.instancetype
{
  "kind": "virtualmachineclusterinstancetype",
  "name": "u1.medium"
}

$ kubectl patch -n kubevirt kv/kubevirt --type merge -p '{"spec":{"configuration":{"instancetype":{"referencePolicy": "expandAll"}}}}'

$ kubectl get vm/test -o json | jq .spec.instancetype
null
```

## inferFromVolume {#inferfromvolume}

`inferFromVolume` 是 `InstancetypeMatcher` 与 `PreferenceMatcher` 共有的属性，用户可以借此请求从卷推断默认值。发出请求后，KubeVirt 会在底层 `PVC`、`DataSource` 或 `DataVolume` 上查找以下标签，以确定默认的名称与 kind：

* `instancetype.kubevirt.io/default-instancetype`
* `instancetype.kubevirt.io/default-instancetype-kind`（可选，默认为 `VirtualMachineClusterInstancetype`）
* `instancetype.kubevirt.io/default-preference`
* `instancetype.kubevirt.io/default-preference-kind`（可选，默认为 `VirtualMachineClusterPreference`）

随后，这些值会由变更准入 webhook 写入相应的匹配器，并在 `VirtualMachine` 被正式接受之前的校验过程中使用。

校验行为可以由提供给 `inferFromVolumeFailurePolicy` 的值来控制，该字段位于 `InstancetypeMatcher` 或 `PreferenceMatcher` 中，而这二者属于某个 `VirtualMachine`。

`Reject` 这一默认值会在找不到所引用的 `Volume`，或找不到底层资源上的标签时，拒绝该请求。

如果提供的是 `Ignore`，则失败时会改为清除相应的 `InstancetypeMatcher` 或 `PreferenceMatcher`。

使用隐式默认值 `Reject` 的示例：

```yaml
$ kubectl apply -k https://github.com/kubevirt/common-instancetypes.git
[..]
$ virtctl image-upload pvc cirros-pvc --size=1Gi --image-path=./cirros-0.5.2-x86_64-disk.img
[..]
$ kubectl label pvc/cirros-pvc \
  instancetype.kubevirt.io/default-instancetype=server.tiny \
  instancetype.kubevirt.io/default-preference=cirros
[..]
$ kubectl apply -f - << EOF
---
apiVersion: cdi.kubevirt.io/v1beta1
kind: DataSource
metadata:
  name: cirros-datasource
spec:
  source:
    pvc:
      name: cirros-pvc
      namespace: default
---
apiVersion: kubevirt.io/v1
kind: VirtualMachine
metadata:
  name: cirros
spec:
  instancetype:
    inferFromVolume: cirros-volume
  preference:
    inferFromVolume: cirros-volume
  runStrategy: Halted
  dataVolumeTemplates:
    - metadata:
        name: cirros-datavolume
      spec:
        storage:
          resources:
            requests:
              storage: 1Gi
          storageClassName: local
        sourceRef:
          kind: DataSource
          name: cirros-datasource
          namespace: default
  template:
    spec:
      domain:
        devices: {}
      volumes:
        - dataVolume:
            name: cirros-datavolume
          name: cirros-volume
EOF
[..]
kubectl get vms/cirros -o json | jq '.spec.instancetype, .spec.preference'
{
  "kind": "virtualmachineclusterinstancetype",
  "name": "server.tiny",
  "revisionName": "cirros-server.tiny-76454433-3d82-43df-a7e5-586e48c71f68-1"
}
{
  "kind": "virtualmachineclusterpreference",
  "name": "cirros",
  "revisionName": "cirros-cirros-85823ddc-9e8c-4d23-a94c-143571b5489c-1"
}
```

显式指定 `Ignore` 的示例：

```yaml
$ virtctl image-upload pvc cirros-pvc --size=1Gi --image-path=./cirros-0.5.2-x86_64-disk.img
$ kubectl apply -f - << EOF
---
apiVersion: cdi.kubevirt.io/v1beta1
kind: DataSource
metadata:
  name: cirros-datasource
spec:
  source:
    pvc:
      name: cirros-pvc
      namespace: default
---
apiVersion: kubevirt.io/v1
kind: VirtualMachine
metadata:
  name: cirros
spec:
  instancetype:
    inferFromVolume: cirros-volume
    inferFromVolumeFailurePolicy: Ignore
  preference:
    inferFromVolume: cirros-volume
    inferFromVolumeFailurePolicy: Ignore
  runStrategy: Halted
  dataVolumeTemplates:
    - metadata:
        name: cirros-datavolume
      spec:
        storage:
          accessModes:
            - ReadWriteOnce
          resources:
            requests:
              storage: 1Gi
          storageClassName: local
        sourceRef:
          kind: DataSource
          name: cirros-datasource
          namespace: default
  template:
    spec:
      domain:
        devices: {}
      volumes:
        - dataVolume:
            name: cirros-datavolume
          name: cirros-volume
EOF
[..]
kubectl get vms/cirros -o json | jq '.spec.instancetype, .spec.preference'
null
null
```

## 热插拔 {#hotplug}

基于实例类型的 vCPU 与内存热插拔支持在 KubeVirt 1.3 中引入，并建立在现有的 [vCPU](../compute/cpu_hotplug.md) 热插拔、[内存](../compute/memory_hotplug.md) 热插拔以及 [LiveUpdate](./vm_rollout_strategies.md) 支持之上。

把新的实例类型热插拔到正在运行的 `VirtualMachine` 时，这些功能的全部要求与限制都适用。

在 KubeVirt >= v1.5.0 中，要触发基于实例类型的 vCPU 和/或内存热插拔，用户应更新所引用实例类型的 `name`，例如：

```shell
kubectl patch vm/my-vm --type merge -p '{"spec":{"instancetype":{"name": "new-instancetype"}}'
```

对于 KubeVirt <= v1.4.0，要触发基于实例类型的 vCPU 和/或内存热插拔，用户应更新所引用实例类型的 name，并清除其 revisionName，例如：

```shell
kubectl patch vm/my-vm --type merge -p '{"spec":{"instancetype":{"name": "new-instancetype", revisionName: ""}}'
```

在满足上述要求的前提下，这会触发与普通 VirtualMachine 相同的 vCPU 和内存热插拔逻辑。

否则会施加 `RestartRequired` 状况到 `VirtualMachine` 上，表示需要重启才能使所有更改生效。

## common-instancetypes {#common-instancetypes}

[`kubevirt/common-instancetypes`](https://github.com/kubevirt/common-instancetypes) 提供了一组 [实例类型与偏好](../user_workloads/instancetypes.md)，用于帮助创建 KubeVirt [`VirtualMachines`](http://kubevirt.io/api-reference/main/definitions.html#_v1alpha1_virtualmachine)。

关于如何部署它们，请参见 [部署 common-instancetypes](../user_workloads/deploy_common_instancetypes.md)。

## 示例 {#examples}

[`kubevirt`](https://github.com/kubevirt/kubevirt) 仓库的 [`/examples`](https://github.com/kubevirt/kubevirt/tree/main/examples) 下提供了各种示例。下面使用一个示例 `VirtualMachine`，它由 [`containerdisk/fedora` 仓库](https://quay.io/repository/containerdisks/fedora) 提供，并用等价的实例类型和偏好替换了 `DomainSpec` 的大部分内容：

```yaml
$ kubectl apply -f - << EOF
---
apiVersion: instancetype.kubevirt.io/v1beta1
kind: VirtualMachineInstancetype
metadata:
  name: cmedium
spec:
  cpu:
    guest: 1
  memory:
    guest: 1Gi
---
apiVersion: instancetype.kubevirt.io/v1beta1
kind: VirtualMachinePreference
metadata:
  name: fedora
spec:
  devices:
    preferredDiskBus: virtio
    preferredInterfaceModel: virtio
    preferredRng: {}
  features:
    preferredAcpi: {}
    preferredSmm: {}
  firmware:
    preferredUseEfi: true
    preferredUseSecureBoot: true    
---
apiVersion: kubevirt.io/v1
kind: VirtualMachine
metadata:
  creationTimestamp: null
  name: fedora
spec:
  instancetype:
    name: cmedium
    kind: virtualMachineInstancetype
  preference:
    name: fedora
    kind: virtualMachinePreference
  runStrategy: Always
  template:
    metadata:
      creationTimestamp: null
    spec:
      domain:
        devices: {}
      volumes:
      - containerDisk:
          image: quay.io/containerdisks/fedora:latest
        name: containerdisk
      - cloudInitNoCloud:
          userData: |-
            #cloud-config
            ssh_authorized_keys:
              - ssh-rsa AAAA...
        name: cloudinit
EOF
```

## 版本历史 {#version-history}

### `instancetype.kubevirt.io/v1alpha1`（实验性） {#instancetypekubevirtiov1alpha1-experimental}

* 初始开发版本。

### `instancetype.kubevirt.io/v1alpha2`（实验性） {#instancetypekubevirtiov1alpha2-experimental}

* 此版本会捕获完整的 `VirtualMachine{Instancetype,ClusterInstancetype,Preference,ClusterPreference}` 对象，并将其放入所创建的 `ControllerRevisions` 中

* 此版本向后兼容 `instancetype.kubevirt.io/v1alpha1`。

### `instancetype.kubevirt.io/v1beta1` {#instancetypekubevirtiov1beta1}

* 新增了以下实例类型属性：
  * [`Spec.Memory.OvercommitPercent`](https://kubevirt.io/api-reference/main/definitions.html#_v1beta1_memoryinstancetype)

* 新增了以下偏好属性：
  * [`Spec.CPU.PreferredCPUFeatures`](https://kubevirt.io/api-reference/main/definitions.html#_v1beta1_cpupreferences)
  * [`Spec.Devices.PreferredInterfaceMasquerade`](https://kubevirt.io/api-reference/main/definitions.html#_v1beta1_devicepreferences)
  * [`Spec.PreferredSubdomain`](https://kubevirt.io/api-reference/main/definitions.html#_v1beta1_virtualmachinepreferencespec)
  * [`Spec.PreferredTerminationGracePeriodSeconds`](https://kubevirt.io/api-reference/main/definitions.html#_v1beta1_virtualmachinepreferencespec)
  * [`Spec.Requirements`](https://kubevirt.io/api-reference/main/definitions.html#_v1beta1_preferencerequirements)

* 此版本向后兼容 `instancetype.kubevirt.io/v1alpha1` 和 `instancetype.kubevirt.io/v1alpha2` 对象，无需修改现有的 `VirtualMachine{Instancetype,ClusterInstancetype,Preference,ClusterPreference}` 或 `ControllerRevisions`。

* 与迁移到 [`kubevirt.io/v1`](https://github.com/kubevirt/kubevirt/blob/main/docs/updates.md#v100-migration-to-new-storage-versions) 时一样，建议此前使用 `instancetype.kubevirt.io/v1alpha1` 或 `instancetype.kubevirt.io/v1alpha2` 的用户使用 [`kube-storage-version-migrator`](https://github.com/kubernetes-sigs/kube-storage-version-migrator)，把任何已存储的对象升级到 `instancetype.kubevirt.io/v1beta1`。
