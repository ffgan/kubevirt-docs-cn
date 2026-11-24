# 模板 {#templates}

!!! Note
    在 OpenShift 之上部署 KubeVirt 后，用户可以使用 [OpenShift 模板](https://docs.openshift.com/container-platform/4.10/virt/vm_templates/virt-creating-vm-template.html) 功能。

!!! Note
    从 KubeVirt v1.8.0 开始，原生的 [VirtualMachine 模板](vm_templates.md) 功能
    取代了这一基于 OpenShift 的旧式模板机制。新部署应改用
    [VirtualMachine 模板](vm_templates.md)。

## 虚拟机模板 {#virtual-machine-templates}

### 什么是虚拟机模板？ {#what-is-a-virtual-machine-template}

KubeVirt 项目提供了一组
[模板](https://docs.okd.io/latest/openshift_images/using-templates.html)，用于
创建 VM 以应对常见使用场景。这些模板把若干关键因素组合在一起，
可以进一步自定义并处理，从而得到一个 Virtual Machine 对象。定义
模板的关键因素包括

-   工作负载 大多数虚拟机应为 **server** 或 **desktop**，以获得最大的
    灵活性；**highperformance** 工作负载会牺牲其中一部分
    灵活性，以提供更好的性能。

-   客户机操作系统（OS） 这可以确保模拟
    硬件与客户机操作系统兼容。此外，它还可以
    最大限度地提高 VM 的稳定性，并允许进行性能
    优化。

-   规格（flavor） 定义分配给
    VM 的资源量（CPU、内存）。

更多文档见 [通用模板
子项目](https://github.com/kubevirt/common-templates)

### 访问虚拟机模板 {#accessing-the-virtual-machine-templates}

如果使用受支持的方法安装了 KubeVirt，集群中应该已经
预装了通用模板。如果希望升级
这些模板，或从头安装它们，可以使用某个
[受支持的
发行版](https://github.com/kubevirt/common-templates/releases)

安装模板：

```console
    $ export VERSION=$(curl -s https://api.github.com/repos/kubevirt/common-templates/releases | grep tag_name | grep -v -- '-rc' | head -1 | awk -F': ' '{print $2}' | sed 's/,//' | xargs)
    $ oc create -f https://github.com/kubevirt/common-templates/releases/download/$VERSION/common-templates-$VERSION.yaml
```

#### 可编辑字段 {#editable-fields}

你可以编辑模板中那些决定 VM 将获得多少
资源的字段。

每个模板都可以列出一组不同的、应被视为
可编辑的字段。这些字段用作用户
界面的提示，也供集群中的其他组件使用。

可编辑字段取自模板中的注解。下面的
片段展示了最常见的几个可编辑字段：

```console
    metadata:
      annotations:
        template.kubevirt.io/editable: |
          /objects[0].spec.template.spec.domain.cpu.sockets
          /objects[0].spec.template.spec.domain.cpu.cores
          /objects[0].spec.template.spec.domain.cpu.threads
          /objects[0].spec.template.spec.domain.resources.requests.memory
```

可编辑字段列表中的每一项都必须是
[jsonpath](https://kubernetes.io/docs/reference/kubectl/jsonpath/)。
jsonpath 的根是模板的 objects: 元素。实际
可编辑的字段是路径的最后一项（“叶子”）。例如，
下面这段最简片段标出了你可以编辑的字段：

```console
    objects:
      spec:
        template:
          spec:
            domain:
              cpu:
                sockets:
                  VALUE # this is editable
                cores:
                  VALUE # this is editable
                threads:
                  VALUE # this is editable
              resources:
                requests:
                  memory:
                    VALUE # this is editable
```

### 模板与 VM 的关系 {#relationship-between-templates-and-vms}

模板经过处理后会生成可在集群中使用的 VM 对象。由模板
生成的 VM 会带有 `vm.kubevirt.io/template` 标签，
其值是父模板的名称，例如
`fedora-desktop-medium`：

```console
      metadata:
        labels:
          vm.kubevirt.io/template: fedora-desktop-medium
```

此外，这些 VM 还可以包含一个可选标签
`vm.kubevirt.io/template-namespace`，其值是父模板所在的命名空间，
例如：

```console
      metadata:
        labels:
          vm.kubevirt.io/template-namespace: openshift
```

如果未定义该标签，则预期模板与该 VM 属于
同一命名空间。

这样就可以查询由任意模板构建的所有 VM。

示例：

```console
    oc process -o yaml -f dist/templates/rhel8-server-tiny.yaml NAME=rheltinyvm SRC_PVC_NAME=rhel SRC_PVC_NAMESPACE=kubevirt
```

以及输出：

```yaml
apiVersion: v1
items:
- apiVersion: kubevirt.io/v1
  kind: VirtualMachine
  metadata:
    annotations:
      vm.kubevirt.io/flavor: tiny
      vm.kubevirt.io/os: rhel8
      vm.kubevirt.io/validations: |
        [
          {
            "name": "minimal-required-memory",
            "path": "jsonpath::.spec.domain.resources.requests.memory",
            "rule": "integer",
            "message": "This VM requires more memory.",
            "min": 1610612736
          }
        ]
      vm.kubevirt.io/workload: server
    labels:
      app: rheltinyvm
      vm.kubevirt.io/template: rhel8-server-tiny
      vm.kubevirt.io/template.revision: "45"
      vm.kubevirt.io/template.version: 0.11.3
    name: rheltinyvm
  spec:
    dataVolumeTemplates:
    - apiVersion: cdi.kubevirt.io/v1beta1
      kind: DataVolume
      metadata:
        name: rheltinyvm
      spec:
        storage:
          accessModes:
          - ReadWriteMany
        source:
          pvc:
            name: rhel
            namespace: kubevirt
    runStrategy: Halted
    template:
      metadata:
        labels:
          kubevirt.io/domain: rheltinyvm
          kubevirt.io/size: tiny
      spec:
        domain:
          cpu:
            cores: 1
            sockets: 1
            threads: 1
          devices:
            disks:
            - disk:
                bus: virtio
              name: rheltinyvm
            - disk:
                bus: virtio
              name: cloudinitdisk
            interfaces:
            - masquerade: {}
              name: default
            networkInterfaceMultiqueue: true
            rng: {}
          resources:
            requests:
              memory: 1.5Gi
        networks:
        - name: default
          pod: {}
        terminationGracePeriodSeconds: 180
        volumes:
        - dataVolume:
            name: rheltinyvm
          name: rheltinyvm
        - cloudInitNoCloud:
            userData: |-
              #cloud-config
              user: cloud-user
              password: lymp-fda4-m1cv
              chpasswd: { expire: False }
          name: cloudinitdisk
kind: List
metadata: {}
```

你可以一次性把由模板生成的 VM 加入集群

```console
    oc process rhel8-server-tiny NAME=rheltinyvm SRC_PVC_NAME=rhel SRC_PVC_NAMESPACE=kubevirt | oc apply -f -
```

请注意，在生成步骤之后，VM 对象与模板对象之间除了上述标签之外没有任何关系。模板中的更改不会自动影响 VM，反之亦然。

### 通用模板自定义 {#common-template-customization}

kubevirt 项目提供的模板包含一组
约定与注解，用于增强
[OpenShift
模板](https://docs.okd.io/latest/openshift_images/using-templates.html) 的基本功能。你可以
通过编辑这些注解来自定义 kubevirt 提供的模板，也可以
把它们添加到现有模板中，使
kubevirt 服务能够使用这些模板。

下面说明 kubevirt 注解。除非另有
说明，以下键应作为模板 metadata 的顶层条目，例如

```yaml
apiVersion: v1
kind: Template
metadata:
  name: windows-10
  annotations:
    openshift.io/display-name: "Generic demo template"
```

以下所有注解都以
`defaults.template.kubevirt.io` 为前缀，下文为简洁起见省略了该前缀。因此
你实际应使用的注解看起来会像这样

```yaml
apiVersion: v1
kind: Template
metadata:
  name: windows-10
  annotations:
    defaults.template.kubevirt.io/disk: default-disk
    defaults.template.kubevirt.io/volume: default-volume
    defaults.template.kubevirt.io/nic: default-nic
    defaults.template.kubevirt.io/network: default-network
```

除非另有说明，所有注解都旨在作为兼顾性能与兼容性的安全
默认值，同时也是感知 CNV 的 UI 与工具所用的提示。

#### disk {#disk}

请参见下方的 `references` 一节。

示例：

```yaml
apiVersion: v1
kind: Template
metadata:
  name: Linux
  annotations:
    defaults.template.kubevirt.io/disk: rhel-disk
```

#### nic {#nic}

请参见下方的 `references` 一节。

示例：

```yaml
apiVersion: v1
kind: Template
metadata:
  name: Windows
  annotations:
    defaults.template.kubevirt.io/nic: my-nic
```

#### volume {#volume}

请参见下方的 `references` 一节。

示例：

```yaml
apiVersion: v1
kind: Template
metadata:
  name: Linux
  annotations:
    defaults.template.kubevirt.io/volume: custom-volume
```

#### network {#network}

请参见下方的 `references` 一节。

示例：

```yaml
apiVersion: v1
kind: Template
metadata:
  name: Linux
  annotations:
    defaults.template.kubevirt.io/network: fast-net
```

#### references {#references}

network、nic、volume、disk 的默认值应当是文档后文某个小节的
**名称**，UI 会找到并
使用该小节，以取得对应类型的默认值。
例如，对于注解
`defaults.template.kubevirt.io/disk: my-disk`：我们假定文档后文
存在一个名为 `my-disk` 的元素，UI 可以用它
找到所需的数据。只要这些名称对 Kubernetes 合法，并且与文档
内容一致，名称本身并不重要。

#### 完整示例 {#complete-example}

`demo-template.yaml`

```yaml
apiversion: v1
items:
- apiversion: kubevirt.io/v1
  kind: virtualmachine
  metadata:
    labels:
      vm.kubevirt.io/template: rhel7-generic-tiny
    name: rheltinyvm
    osinfoname: rhel7.0
    defaults.template.kubevirt.io/disk: rhel-default-disk
    defaults.template.kubevirt.io/nic: rhel-default-net
  spec:
    runStrategy: Halted
    template:
      spec:
        domain:
          cpu:
            sockets: 1
            cores: 1
            threads: 1
          devices:
            rng: {}
          resources:
            requests:
              memory: 1g
        terminationgraceperiodseconds: 0
        volumes:
        - containerDisk:
          image: registry:5000/kubevirt/cirros-container-disk-demo:devel
          name: rhel-default-disk
        networks:
        - genie:
          networkName: flannel
          name: rhel-default-net
kind: list
metadata: {}
```

处理之后会变成：
`demo-vm.yaml`

```yaml
apiVersion: kubevirt.io/v1
kind: VirtualMachine
metadata:
  labels:
    vm.kubevirt.io/template: rhel7-generic-tiny
  name: rheltinyvm
  osinfoname: rhel7.0
spec:
  runStrategy: Halted
  template:
    spec:
      domain:
        cpu:
          sockets: 1
          cores: 1
          threads: 1
        resources:
          requests:
            memory: 1g
        devices:
          rng: {}
          disks:
          - disk:
            name: rhel-default-disk
        interfaces:
        - bridge: {}
          name: rhel-default-nic
      terminationgraceperiodseconds: 0
      volumes:
      - containerDisk:
          image: registry:5000/kubevirt/cirros-container-disk-demo:devel
        name: containerdisk
      networks:
      - genie:
          networkName: flannel
        name: rhel-default-nic
```

## 虚拟机创建 {#virtual-machine-creation}

### 概述 {#overview}

KubeVirt 项目提供了一组
[模板](https://github.com/kubevirt/kubevirt/tree/main/examples)，用于
创建 VM 以应对常见使用场景。这些模板把若干关键因素组合在一起，
可以进一步自定义并处理，从而得到一个 Virtual Machine 对象。

定义模板的关键因素包括 - 工作负载 大多数虚拟
机应为 **server** 或 **desktop**，以获得最大的灵活性；
**highperformance** 工作负载会牺牲其中一部分灵活性，以
提供更好的性能。- 客户机操作系统（OS） 这可以
确保模拟硬件与客户机操作系统兼容。
此外，它还可以最大限度地提高 VM 的稳定性，并允许进行
性能优化。- 规格（flavor） 定义分配给 VM 的
资源量（CPU、内存）。

### OpenShift 控制台 {#openshift-console}

虚拟机可以通过 [OpenShift 集群控制台 UI](https://github.com/openshift/console) 创建。
此 UI 支持使用模板以及模板的各项
功能（flavor 与工作负载配置）来创建 VM。要从模板创建 VM，请在
左侧面板中选择 WorkLoads >> 选择 Virtualization >> 点击蓝色的 “Create Virtual Machine”
按钮 >> 选择 “Create from wizard”。接下来，你会看到
“Create Virtual Machine” 窗口

### 通用模板 {#common-templates}

这里有 [common-templates](https://github.com/kubevirt/common-templates/) 子项目。它提供官方准备好的实用模板。你也可以手工创建模板。示例见下方“示例模板”一节。

### 示例模板 {#example-template}

要通过 OpenShift CLI 创建虚拟机，你需要
提供一个模板，用以定义相应的对象及其 metadata。

**注意** 目前仅支持 `VirtualMachine` 对象。

下面是一个定义
`VirtualMachine` 对象实例的示例模板：

```yaml
apiVersion: template.openshift.io/v1
kind: Template
metadata:
  name: fedora-desktop-large
  annotations:
    openshift.io/display-name: "Fedora 32+ VM"
    description: >-
      Template for Fedora 32 VM or newer.
      A PVC with the Fedora disk image must be available.
      Recommended disk image:
      https://download.fedoraproject.org/pub/fedora/linux/releases/32/Cloud/x86_64/images/Fedora-Cloud-Base-32-1.6.x86_64.qcow2
    tags: "hidden,kubevirt,virtualmachine,fedora"
    iconClass: "icon-fedora"
    openshift.io/provider-display-name: "KubeVirt"
    openshift.io/documentation-url: "https://github.com/kubevirt/common-templates"
    openshift.io/support-url: "https://github.com/kubevirt/common-templates/issues"
    template.openshift.io/bindable: "false"
    template.kubevirt.io/version: v1alpha1
    defaults.template.kubevirt.io/disk: rootdisk
    template.kubevirt.io/editable: |
      /objects[0].spec.template.spec.domain.cpu.sockets
      /objects[0].spec.template.spec.domain.cpu.cores
      /objects[0].spec.template.spec.domain.cpu.threads
      /objects[0].spec.template.spec.domain.resources.requests.memory
      /objects[0].spec.template.spec.domain.devices.disks
      /objects[0].spec.template.spec.volumes
      /objects[0].spec.template.spec.networks
    name.os.template.kubevirt.io/fedora32: Fedora 32 or higher
    name.os.template.kubevirt.io/fedora33: Fedora 32 or higher
    name.os.template.kubevirt.io/silverblue32: Fedora 32 or higher
    name.os.template.kubevirt.io/silverblue33: Fedora 32 or higher
  labels:
    os.template.kubevirt.io/fedora32: "true"
    os.template.kubevirt.io/fedora33: "true"
    os.template.kubevirt.io/silverblue32: "true"
    os.template.kubevirt.io/silverblue33: "true"
    workload.template.kubevirt.io/desktop: "true"
    flavor.template.kubevirt.io/large: "true"
    template.kubevirt.io/type: "base"
    template.kubevirt.io/version: "0.11.3"
objects:
- apiVersion: kubevirt.io/v1
  kind: VirtualMachine
  metadata:
    name: ${NAME}
    labels:
      vm.kubevirt.io/template: fedora-desktop-large
      vm.kubevirt.io/template.version: "0.11.3"
      vm.kubevirt.io/template.revision: "45"
      app: ${NAME}
    annotations:
      vm.kubevirt.io/os: "fedora"
      vm.kubevirt.io/workload: "desktop"
      vm.kubevirt.io/flavor: "large"
      vm.kubevirt.io/validations: |
        [
          {
            "name": "minimal-required-memory",
            "path": "jsonpath::.spec.domain.resources.requests.memory",
            "rule": "integer",
            "message": "This VM requires more memory.",
            "min": 1073741824
          }
        ]
  spec:
    dataVolumeTemplates:
    - apiVersion: cdi.kubevirt.io/v1beta1
      kind: DataVolume
      metadata:
        name: ${NAME}
      spec:
        storage:
          accessModes:
            - ReadWriteMany
        source:
          pvc:
            name: ${SRC_PVC_NAME}
            namespace: ${SRC_PVC_NAMESPACE}
    runStrategy: Halted
    template:
      metadata:
        labels:
          kubevirt.io/domain: ${NAME}
          kubevirt.io/size: large
      spec:
        domain:
          cpu:
            sockets: 2
            cores: 1
            threads: 1
          resources:
            requests:
              memory: 8Gi
          devices:
            rng: {}
            networkInterfaceMultiqueue: true
            inputs:
              - type: tablet
                bus: virtio
                name: tablet
            disks:
            - disk:
                bus: virtio
              name: ${NAME}
            - disk:
                bus: virtio
              name: cloudinitdisk
            interfaces:
            - masquerade: {}
              name: default
        terminationGracePeriodSeconds: 180
        networks:
        - name: default
          pod: {}
        volumes:
        - dataVolume:
            name: ${NAME}
          name: ${NAME}
        - cloudInitNoCloud:
            userData: |-
              #cloud-config
              user: fedora
              password: ${CLOUD_USER_PASSWORD}
              chpasswd: { expire: False }
          name: cloudinitdisk
parameters:
- description: VM name
  from: 'fedora-[a-z0-9]{16}'
  generate: expression
  name: NAME
- name: SRC_PVC_NAME
  description: Name of the PVC to clone
  value: 'fedora'
- name: SRC_PVC_NAMESPACE
  description: Namespace of the source PVC
  value: kubevirt-os-images
- description: Randomized password for the cloud-init user fedora
  from: '[a-z0-9]{4}-[a-z0-9]{4}-[a-z0-9]{4}'
  generate: expression
  name: CLOUD_USER_PASSWORD
```

请注意，上面的模板定义了自由参数（`NAME`、
`SRC_PVC_NAME`、`SRC_PVC_NAMESPACE`、`CLOUD_USER_PASSWORD`），并且 `NAME`
参数没有指定默认值。

OpenShift 模板必须通过
`oc process` 命令转换为 JSON 文件，该命令也允许你设置模板
参数。

完整示例见 [KubeVirt
仓库](https://raw.githubusercontent.com/kubevirt/kubevirt/main/examples/vm-template-fedora.yaml)。

!> 你需要先通过 `oc login` 命令登录。

```console
$ oc process -f cluster/vmi-template-fedora.yaml\
    -p NAME=testvmi \
    -p SRC_PVC_NAME=fedora \
    -p SRC_PVC_NAMESPACE=kubevirt \
{
    "kind": "List",
    "apiVersion": "v1",
    "metadata": {},
    "items": [
        {
```

JSON 文件通常直接应用，做法是把处理后的输出
通过管道传给 `oc create` 命令。

```console
$ oc process -f cluster/examples/vm-template-fedora.yaml \
    -p NAME=testvm \
    -p SRC_PVC_NAME=fedora \
    -p SRC_PVC_NAMESPACE=kubevirt \
    | oc create -f -
virtualmachine.kubevirt.io/testvm created
```

上述命令会根据
模板给出的规范创建一个 Kubernetes 对象\\(在本例中，它是
VirtualMachine 对象的一个实例\\)。

可以使用以下命令获取可用参数的列表：

```console
$ oc process -f dist/templates/fedora-desktop-large.yaml --parameters
NAME                  DESCRIPTION                                          GENERATOR           VALUE
NAME                  VM name                                              expression          fedora-[a-z0-9]{16}
SRC_PVC_NAME          Name of the PVC to clone                                                 fedora
SRC_PVC_NAMESPACE     Namespace of the source PVC                                              kubevirt-os-images
CLOUD_USER_PASSWORD   Randomized password for the cloud-init user fedora   expression          [a-z0-9]{4}-[a-z0-9]{4}-[a-z0-9]{4}
```

## 从已创建的对象启动虚拟机 {#starting-virtual-machine-from-the-created-object}

创建出的对象现在是一个常规的 VirtualMachine 对象，此后
可以通过访问 Kubernetes API 资源来控制它。在 OpenShift 环境中，
首选做法是
使用 `oc patch` 命令。

```console
$ oc patch virtualmachine testvm --type merge -p '{"spec":{"runStrategy":"Always"}}'
virtualmachine.kubevirt.io/testvm patched
```

不要忘记 virtctl 工具。在实际场景中使用它，比起
直接使用 Kubernetes API，可能会更方便。示例：

```console
$ virtctl start testvm
VM testvm was scheduled to start
```

VM 一旦启动，Kubernetes 就会创建一种新类型的对象
VirtualMachineInstance。它的名称与 VirtualMachine 相似。示例
（并非完整输出，内容太长）：

```console
$ kubectl describe vm testvm
name:         testvm
Namespace:    myproject
Labels:       kubevirt-vm=vm-testvm
              kubevirt.io/os=fedora33
Annotations:  <none>
API Version:  kubevirt.io/v1
Kind:         VirtualMachine
```

## Cloud-init 脚本与参数 {#cloud-init-script-and-parameters}

KubeVirt 的 VM 模板与 KubeVirt 的 VM/VMI yaml 配置一样，支持
[cloud-init 脚本](https://cloudinit.readthedocs.io/en/latest/)

## **变通做法** - 使用预先下载的镜像 {#hack-use-pre-downloaded-image}

KubeVirt 的 VM 模板与 KubeVirt 的 VM/VMI yaml 配置一样，可以使用
预先下载的 VM 镜像，这在
调试/开发/测试场景中尤其有用。VM 模板或 VM/VMI yaml 配置中
不需要特殊参数。主要思路是创建与文件系统中
已有镜像相对应的 Kubernetes
PersistentVolume 和 PersistentVolumeClaim。示例：

```yaml
---
kind: PersistentVolume
apiVersion: v1
metadata:
  name: mypv
  labels:
    type: local
spec:
  storageClassName: manual
  capacity:
    storage: 10G
  accessModes:
    - ReadWriteOnce
  hostPath:
    path: "/mnt/sda1/images/testvm"
---
kind: PersistentVolumeClaim
apiVersion: v1
metadata:
  name: mypvc
spec:
  storageClassName: manual
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 10G

```

## 使用 DataVolume {#using-datavolumes}

KubeVirt 的 VM 模板使用 dataVolumeTemplates。
在使用 dataVolume 之前，必须先在
集群中安装 CDI。之后可以创建源 DataVolume。

```yaml
---
apiVersion: cdi.kubevirt.io/v1beta1
kind: DataVolume
metadata:
  name: fedora-datavolume-original
  namespace: kubevirt
spec:
  source:
    registry:
      url: "image_url"
  storage:
    resources:
      requests:
        storage: 30Gi
```

导入完成后，可以创建 VM：
```console
$ oc process -f cluster/examples/vm-template-fedora.yaml \
    -p NAME=testvmi \
    -p SRC_PVC_NAME=fedora-datavolume-original \
    -p SRC_PVC_NAMESPACE=kubevirt \
    | oc create -f -
virtualmachine.kubevirt.io/testvm created
```

## 更多信息 {#additional-information}
你可以参阅 [虚拟机生命周期指南](../user_workloads/lifecycle.md) 以获取进一步的参考。
