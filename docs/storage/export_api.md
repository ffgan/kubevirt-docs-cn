# 导出 API {#export-api}
有时需要把虚拟机及其相关磁盘导出到集群之外，以便把该虚拟机导入另一个系统或集群。虚拟机磁盘是你最需要导出的内容。导出 API 使以声明方式导出虚拟机磁盘成为可能。也可以导出单个 PVC 及其内容，例如当你从 VM 创建了内存转储，或者正在使用 virtio-fs 让虚拟机填充 PVC 时。

为了不使 Kubernetes API 服务器过载，数据通过专用的导出代理服务器传输。然后可以通过与 Ingress/Route 或 NodePort 关联的服务，把该代理服务器暴露给外部。作为替代方案，可以在 virtctl 集成中使用 `port-forward` 标志，从而无需 Ingress/Route。

### 导出功能门控 {#export-feature-gate}

必须在功能门控中启用 VMExport 支持，该功能才可用。必须扩展 KubeVirt CR 中的
[功能门控](../cluster_admin/activating_feature_gates.md#how-to-activate-a-feature-gate)
字段，向其中添加 `VMExport`。

### 导出令牌 {#export-token}

为了安全地导出虚拟机磁盘，必须创建一个令牌，用于授权访问导出端点的用户。此令牌必须与虚拟机位于同一命名空间。Secret 的内容可以作为令牌头或参数传递给导出 URL。该头或参数的名称是 `x-kubevirt-export-token`，其值与 Secret 的内容匹配。该 Secret 可以使用命名空间中任何有效的 Secret 名称。建议生成至少 12 个字符的字母数字令牌。数据键应为 `token`。例如：

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: example-token
stringData:
  token: 1234567890ab
```
### 导出虚拟机卷 {#export-virtual-machine-volumes}
创建令牌之后，现在可以创建一个 VMExport CR，用于标识要导出的虚拟机。可以创建如下所示的 VMExport：

```yaml
apiVersion: export.kubevirt.io/v1beta1
kind: VirtualMachineExport
metadata:
  name: example-export
spec:
  tokenSecretRef: example-token
  source:
    apiGroup: "kubevirt.io"
    kind: VirtualMachine
    name: example-vm
```

VM 中存在的以下卷将被导出：

* PersistentVolumeClaims
* DataVolumes
* MemoryDump

所有其他卷类型都不会被导出。为避免导出不一致的数据，虚拟机只能在关机状态下导出。如果虚拟机被启动，任何活动的 VM 导出都将被终止。要从正在运行的虚拟机导出数据，必须先创建虚拟机快照（参见下文）。

如果 VM 包含多个可以导出的卷，每个卷都会有自己的 URL 链接。如果 VM 不包含可以导出的卷，VMExport 将进入 `Skipped` 阶段，并且不会启动导出服务器。

### 导出虚拟机快照卷 {#export-virtual-machine-snapshot-volumes}

可以创建一个 VMExport CR，用于标识要导出的虚拟机快照。可以创建如下所示的 VMExport：

```yaml
apiVersion: export.kubevirt.io/v1beta1
kind: VirtualMachineExport
metadata:
  name: example-export
spec:
  tokenSecretRef: example-token
  source:
    apiGroup: "snapshot.kubevirt.io"
    kind: VirtualMachineSnapshot
    name: example-vmsnapshot
```

基于虚拟机快照创建 VMExport 时，控制器会尝试从虚拟机快照中包含的卷快照创建 PVC。一旦所有 PVC 就绪，导出服务器就会启动，然后可以开始导出。如果虚拟机快照包含多个可以导出的卷，每个卷都会有自己的 URL 链接。如果虚拟机快照不包含可以导出的卷，VMExport 将进入 `skipped` 阶段，并且不会启动导出服务器。

### 导出 VirtualMachineTemplate {#export-virtualmachinetemplate}

从 KubeVirt v1.9.0 开始，可以把 `VirtualMachineTemplate` 导出为
OCI 制品。这需要 [`OCIExport`](#oci-format) 和 `Template`
功能门控，并且必须启用 virt-template 部署。`Template` 和
virt-template 部署默认启用。有关
`VirtualMachineTemplates` 的更多细节，请参阅
[VirtualMachine 模板](../user_workloads/vm_templates.md)。

```yaml
apiVersion: export.kubevirt.io/v1beta1
kind: VirtualMachineExport
metadata:
  name: example-export
spec:
  tokenSecretRef: example-token
  source:
    apiGroup: "template.kubevirt.io"
    kind: VirtualMachineTemplate
    name: example-template
```

模板以及所有关联的卷数据会被打包成单个 OCI
制品。与 VM 和快照导出不同，这里不提供按卷的下载链接。

### 导出持久卷声明 {#export-persistent-volume-claim}

可以创建一个 VMExport CR，用于标识要导出的持久卷声明（PVC）。可以创建如下所示的 VMExport：

```yaml
apiVersion: export.kubevirt.io/v1beta1
kind: VirtualMachineExport
metadata:
  name: example-export
spec:
  tokenSecretRef: example-token
  source:
    apiGroup: ""
    kind: PersistentVolumeClaim
    name: example-pvc
```

在此示例中，PVC 名称是 `example-pvc`。请注意，PVC 不必包含虚拟机磁盘，它可以包含任何内容，但主要用例是导出虚拟机磁盘。把此 yaml 提交到集群后，会在与该 PVC 相同的命名空间中创建一个新的导出服务器。如果源 PVC *正被另一个 Pod 使用*（例如 virt-launcher Pod），则导出将保持待处理状态，直到该 PVC 不再被使用。如果导出器服务器处于活动状态，而另一个 Pod 开始使用该 PVC，导出器服务器将被终止，直到该 PVC 不再被使用。


### 导出状态链接 {#export-status-links}

VirtualMachineExport CR 的 status 中会包含指向导出服务的内部链接和外部链接。内部链接仅在集群内部有效，外部链接可通过 Ingress 或 Route 进行外部访问。`cert` 字段将包含为导出服务器证书签名的 CA（用于内部链接），或为 Route 或 Ingress 签名的 CA。

#### KubeVirt 内容类型 {#kubevirt-content-type}
下面是导出包含 KubeVirt 磁盘镜像的 PVC 的示例。控制器通过检查 PVC 上是否有特殊注解、PVC 上是否有 DataVolume ownerReference，或者 PVC 的 volumeMode 是否为 block，来判断 PVC 是否包含 kubevirt 磁盘。

```yaml
apiVersion: export.kubevirt.io/v1beta1
kind: VirtualMachineExport
metadata:
  name: example-export
  namespace: example
spec:
  source:
    apiGroup: ""
    kind: PersistentVolumeClaim
    name: example-pvc
  tokenSecretRef: example-token
status:
  conditions:
  - lastProbeTime: null
    lastTransitionTime: "2022-06-21T14:10:09Z"
    reason: podReady
    status: "True"
    type: Ready
  - lastProbeTime: null
    lastTransitionTime: "2022-06-21T14:09:02Z"
    reason: pvcBound
    status: "True"
    type: PVCReady
  links:
    external:
      cert: |-
        -----BEGIN CERTIFICATE-----
        ...
        -----END CERTIFICATE-----
      volumes:
      - formats:
        - format: raw
          url: https://vmexport-proxy.test.net/api/export.kubevirt.io/v1beta1/namespaces/example/virtualmachineexports/example-export/volumes/example-disk/disk.img
        - format: gzip
          url: https://vmexport-proxy.test.net/api/export.kubevirt.io/v1beta1/namespaces/example/virtualmachineexports/example-export/volumes/example-disk/disk.img.gz
        name: example-disk
    internal:
      cert: |-
        -----BEGIN CERTIFICATE-----
        ...
        -----END CERTIFICATE-----
      volumes:
      - formats:
        - format: raw
          url: https://virt-export-example-export.example.svc/volumes/example-disk/disk.img
        - format: gzip
          url: https://virt-export-example-export.example.svc/volumes/example-disk/disk.img.gz
        name: example-disk
  phase: Ready
  serviceName: virt-export-example-export
```
#### 归档内容类型 {#archive-content-type}
如果无法确定 PVC 包含 KubeVirt 磁盘，则会自动选择归档内容类型。归档将包含 PVC 中的所有文件。

```yaml
apiVersion: export.kubevirt.io/v1beta1
kind: VirtualMachineExport
metadata:
  name: example-export
  namespace: example
spec:
  source:
    apiGroup: ""
    kind: PersistentVolumeClaim
    name: example-pvc
  tokenSecretRef: example-token
status:
  conditions:
  - lastProbeTime: null
    lastTransitionTime: "2022-06-21T14:10:09Z"
    reason: podReady
    status: "True"
    type: Ready
  - lastProbeTime: null
    lastTransitionTime: "2022-06-21T14:09:02Z"
    reason: pvcBound
    status: "True"
    type: PVCReady
  links:
    external:
      cert: |-
        -----BEGIN CERTIFICATE-----
        ...
        -----END CERTIFICATE-----
      volumes:
      - formats:
        - format: dir
          url: https://vmexport-proxy.test.net/api/export.kubevirt.io/v1beta1/namespaces/example/virtualmachineexports/example-export/volumes/example/dir
        - format: tar.gz
          url: https://vmexport-proxy.test.net/api/export.kubevirt.io/v1beta1/namespaces/example/virtualmachineexports/example-export/volumes/example/disk.tar.gz
        name: example-disk
    internal:
      cert: |-
        -----BEGIN CERTIFICATE-----
        ...
        -----END CERTIFICATE-----
      volumes:
      - formats:
        - format: dir
          url: https://virt-export-example-export.example.svc/volumes/example/dir
        - format: tar.gz
          url: https://virt-export-example-export.example.svc/volumes/example/disk.tar.gz
        name: example-disk
  phase: Ready
  serviceName: virt-export-example-export
```

#### 清单 {#manifests}
可以通过访问 VirtualMachineExport status 中的 `manifests` 来获取 VirtualMachine 清单。`all` 类型将返回 VirtualMachine 清单、任何 DataVolume，以及一个 configMap，其中包含外部 URL 的 Ingress/Route 的公共 CA 证书，或内部 URL 的导出服务器的 CA。`auth-header-secret` 将是一个 Secret，其中包含与容器化数据导入器（CDI）兼容的头。此头包含导出令牌的文本形式。

内部链接和外部链接都会包含 `manifests` 字段。如果没有外部链接，那么也不会有任何外部清单。virtualMachine 的 `manifests` 字段仅在源是 `VirtualMachine` 或 `VirtualMachineSnapshot` 时可用。导出 `PersistentVolumeClaim` 不会生成虚拟机清单。

```yaml
apiVersion: export.kubevirt.io/v1beta1
kind: VirtualMachineExport
metadata:
  name: example-export
  namespace: example
spec:
  source:
    apiGroup: ""
    kind: PersistentVolumeClaim
    name: example-pvc
  tokenSecretRef: example-token
status:
  conditions:
  - lastProbeTime: null
    lastTransitionTime: "2022-06-21T14:10:09Z"
    reason: podReady
    status: "True"
    type: Ready
  - lastProbeTime: null
    lastTransitionTime: "2022-06-21T14:09:02Z"
    reason: pvcBound
    status: "True"
    type: PVCReady
  links:
    external:
      ...
      manifests:
      - type: all
        url: https://vmexport-proxy.test.net/api/export.kubevirt.io/v1beta1/namespaces/example/virtualmachineexports/example-export/external/manifests/all
      - type: auth-header-secret
        url: https://vmexport-proxy.test.net/api/export.kubevirt.io/v1beta1/namespaces/example/virtualmachineexports/example-export/external/manifests/secret
    internal:
      ...
      manifests:
      - type: all
        url: https://virt-export-export-pvc.default.svc/internal/manifests/all
      - type: auth-header-secret
        url: https://virt-export-export-pvc.default.svc/internal/manifests/secret
  phase: Ready
  serviceName: virt-export-example-export
```

#### 格式类型 {#format-types}
可能有 4 种格式类型：

* Raw。未经改动的原始 KubeVirt 磁盘镜像。
* Gzip。原始 KubeVirt 磁盘镜像，但经过 gzip 压缩，以提高传输效率。
* Dir。目录列表，使你能够找到 PVC 中包含的文件。
* Tar.gz。PVC 的内容被打包为 tar 并经 gzip 压缩后放在单个文件中。

如果确定 PVC 是 KubeVirt 磁盘，则会选择 Raw 和 Gzip。KubeVirt 磁盘包含单个 disk.img 文件（或者是块设备）。Dir 将返回 PVC 中的文件列表，要下载特定文件，可以把 URL 中的 `/dir` 替换为路径和文件名。例如，如果 PVC 包含文件 `/example/data.txt`，可以把 `/dir` 替换为 `/example/data.txt`，从而只下载 data.txt 文件。或者可以使用 tar.gz URL 来获取 PVC 的全部内容，结果是一个 tar 文件。

#### OCI 格式 {#oci-format}

从 KubeVirt v1.9.0 开始，VM 导出可以下载为 OCI 镜像布局
TAR。这会把 VM 配置和所有磁盘卷打包成单个 OCI
制品，可以使用 skopeo、crane 或 oras 之类的工具，通过标准容器镜像仓库分发。

OCI 导出要求启用 `OCIExport` 功能门控，此外还要启用
`VMExport`。详情请参阅
[激活功能门控](../cluster_admin/activating_feature_gates.md)。

要下载 OCI 格式的导出，请使用 `--format=oci` 标志：

```shell
$ virtctl vmexport download example-export --format=oci --output=example-vm.oci.tar
```

生成的 TAR 文件符合
[OCI 镜像布局规范](https://github.com/opencontainers/image-spec/blob/main/image-layout.md)，
并且可以推送到镜像仓库：

```shell
# podman unshare allows preserving UIDs in the TAR when skopeo is
# extracting it
$ podman unshare skopeo copy oci-archive:example-vm.oci.tar docker://registry.example.com/vms/example-vm:latest
```

#### 内部链接证书 {#internal-link-certificates}
导出服务器证书的有效期为 7 天，之后会通过删除导出服务器 Pod 及相关 Secret 并生成新的证书来轮换。如果导出服务器 Pod 因任何原因终止，关联的 Secret 也会被自动删除，并生成新的 Pod 和 Secret。VirtualMachineExport 对象的 status 会自动更新，以反映新证书。

#### 外部链接证书 {#external-link-certificates}
外部链接证书与指向 KubeVirt operator 所创建服务的 Ingress/Route 相关联。签发 Ingress/Route 的 CA 将成为这些证书的一部分。

### 导出的 TTL（生存时间） {#ttl-time-to-live-for-an-export}
出于各种原因（安全是其中之一），用户应当能够为 VMExport 对象指定 TTL，以限制导出的生存时间。
这通过 `ttlDuration` 字段完成，该字段接受 k8s [时长](https://godoc.org/k8s.io/apimachinery/pkg/apis/meta/v1#Duration)，
未指定时默认为 2 小时。
```yaml
apiVersion: export.kubevirt.io/v1beta1
kind: VirtualMachineExport
metadata:
    name: example-export
spec:
    source:
        apiGroup: "kubevirt.io"
        kind: VirtualMachine
        name: example-vm
    tokenSecretRef: example-token
    ttlDuration: 1h
```

### virtctl 集成：vmexport {#virtctl-integration-vmexport}
virtctl 的 `vmexport` 命令允许用户以便于使用的方式与导出 API 交互。

`vmexport` 使用两个必填参数：

* vmexport **功能**（create|delete|download）。
* VirtualMachineExport **名称**。

这三个**功能**是：

#### 创建 {#create}
```sh
# Creates a VMExport object according to the specified flag.

# The flag should either be:

# --pvc, to specify the name of the pvc to export.
# --snapshot, to specify the name of the VM snapshot to export.
# --vm, to specify the name of the Virtual Machine to export.
# --vmtemplate, to specify the name of the VirtualMachineTemplate to export (OCI only).

$ virtctl vmexport create name [flags]
```

#### 删除 {#delete}
```sh
# Deletes the specified VMExport object.

$ virtctl vmexport delete name
```
#### 下载 {#download}
```sh
# Downloads a volume from the defined VMExport object.

# The main available flags are:

# --output, mandatory flag to specify the output file.
# --volume, optional flag to specify the name of the downloadable volume.
# --vm|--snapshot|--pvc, if specified, are used to create the VMExport object assuming it doesn't exist. The name of the object to export has to be specified.
# --format, optional flag to specify wether to download the file in compressed (default) or raw format.
# --port-forward, optional flag to easily download the volume without the need of an ingress or route. Also, the local port can be optionally specified with the --local-port flag.

$ virtctl vmexport download name [flags]
```

默认情况下，卷将以压缩格式下载。用户可以使用 `format` 标志指定所需格式（gzip 或 raw），如下所示：

```sh
# Downloads a volume from the defined VMExport object and, if necessary, decompresses it.
$ virtctl vmexport download name --format=raw [flags]
```

#### TTL（生存时间） {#ttl-time-to-live}
通过 virtctl 创建 VMExport 时也可以添加 TTL
```sh
$ virtctl vmexport create name --ttl=1h
```

有关用法和示例的更多信息：

```
$ virtctl vmexport --help

Export a VM volume.

Usage:
  virtctl vmexport [flags]

Examples:
  # Create a VirtualMachineExport to export a volume from a virtual machine:
	virtctl vmexport create vm1-export --vm=vm1

	# Create a VirtualMachineExport to export a volume from a virtual machine snapshot
	virtctl vmexport create snap1-export --snapshot=snap1

	# Create a VirtualMachineExport to export a volume from a PVC
	virtctl vmexport create pvc1-export --pvc=pvc1

	# Delete a VirtualMachineExport resource
	virtctl vmexport delete snap1-export

	# Download a volume from an already existing VirtualMachineExport (--volume is optional when only one volume is available)
	virtctl vmexport download vm1-export --volume=volume1 --output=disk.img.gz

	# Download a volume as before but through local port 5410
	virtctl vmexport download vm1-export --volume=volume1 --output=disk.img.gz --port-forward --local-port=5410

	# Create a VirtualMachineExport and download the requested volume from it
	virtctl vmexport download vm1-export --vm=vm1 --volume=volume1 --output=disk.img.gz

	# Create a VirtualMachineExport and get the VirtualMachine manifest in Yaml format
	virtctl vmexport download vm1-export --vm=vm1 --manifest

	# Get the VirtualMachine manifest in Yaml format from an existing VirtualMachineExport including CDI header secret
	virtctl vmexport download existing-export --include-secret --manifest

Flags:
  -h, --help              help for vmexport
      --annotations       Specify custom annotations to VM export object and its associated pod.
      --delete-vme        When used with the 'download' option, specifies that the vmexport object should always be deleted after the download finishes.
      --export-manifest   Instead of downloading a volume, retrieve the VM manifest.
      --format            Used to specify the format of the downloaded image. Valid options are: gzip (default) and raw.
      --include-secret    When used with manifest and set to true, include a secret that contains proper headers for CDI to import using the manifest.
      --insecure          When used with the 'download' option, specifies that the http request should be insecure.
      --keep-vme          When used with the 'download' option, specifies that the vmexport object should always be retained after the download finishes.
      --labels            Specify custom labels to VM export object and its associated pod.
      --local-port        Defines the specific port to be used in port-forward. Default is "0" (random).
      --manifest-output-format
                          Manifest output format. Valid options are yaml (default) or json.
      --output            Specifies the output path of the volume to be downloaded.
      --port-forward      Configures port-forwarding on a random port. Useful to download without proper ingress/route configuration.
      --pvc               Sets PersistentVolumeClaim as vmexport kind and specifies the PVC name.
      --readiness-timeout Specify maximum wait time for the VM export object to be ready.
      --retry             When export server returns a transient error, retry this number of times before giving up.
      --service-url       Specify service URL to use in the returned manifest, instead of the external URL in the VM export status.
      --snapshot          Sets VirtualMachineSnapshot as vmexport kind and specifies the snapshot name.
      --ttl               The time after the export was created that it is eligible to be automatically deleted. Defaults to 2 hours if not specified.
      --vm                Sets VirtualMachine as vmexport kind and specifies the VM name.
      --vmtemplate        Sets VirtualMachineTemplate as vmexport kind and specifies the template name (OCI only).
      --volume            Specifies the volume to be downloaded.

Use "virtctl options" for a list of global command-line options (applies to all commands).
```


### 用例 {#use-cases}

#### 把 VM 从一个集群克隆到另一个集群 {#clone-vm-from-one-cluster-to-another-cluster}

如果要把 KubeVirt 磁盘镜像从源集群传输到另一个目标集群，可以在源集群中使用 VMExport 来暴露磁盘，并在目标集群中使用容器化数据导入器（CDI）把镜像导入目标集群。假设源集群中有一个 Ingress 或 Route，它以下面的示例域名 `virt-exportproxy-example.example.com` 暴露导出代理，并且源集群中有一个带一块磁盘的虚拟机，如下所示：

```yaml
apiVersion: kubevirt.io/v1
kind: VirtualMachine
metadata:
  labels:
    kubevirt.io/vm: vm-example-datavolume
  name: example-vm
spec:
  dataVolumeTemplates:
  - metadata:
      creationTimestamp: null
      name: example-dv
    spec:
      storage:
        accessModes:
        - ReadWriteOnce
        resources:
          requests:
            storage: 20Gi
        storageClassName: local
      source:
        registry:
          url: docker://quay.io/containerdisks/centos-stream:9
  runStrategy: Halted
  template:
    metadata:
      labels:
        kubevirt.io/vm: vm-example-datavolume
    spec:
      domain:
        devices:
          disks:
          - disk:
              bus: virtio
            name: datavolumedisk1
        resources:
          requests:
            memory: 2Gi
      terminationGracePeriodSeconds: 0
      volumes:
      - dataVolume:
          name: example-dv
        name: datavolumedisk1
```

这是一个 VM，它有一个从容器磁盘填充的 DataVolume（DV）`example-dv`，我们希望把该磁盘导出到目标集群。要导出此 VM，我们必须创建一个令牌，以便在目标集群中用来获取对导出的访问权限，也可以让导出控制器为我们生成一个令牌。例如

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: example-token
stringData:
  token: 1234567890ab
```
该令牌的值是 `1234567890ab`，算不上安全的令牌，但这只是一个示例。现在我们可以创建如下所示的 VMExport：

```yaml
apiVersion: export.kubevirt.io/v1beta1
kind: VirtualMachineExport
metadata:
  name: example-export
spec:
  tokenSecretRef: example-token #optional, if omitted the export controller will generate a token
  source:
    apiGroup: "kubevirt.io"
    kind: VirtualMachine
    name: example-vm
```
如果 VM 未在运行，一旦 export-server Pod 运行起来，VMExport 对象的 status 就会被更新，看起来大致如下：

```yaml
apiVersion: export.kubevirt.io/v1beta1
kind: VirtualMachineExport
metadata:
  name: example-export
  namespace: example
spec:
  tokenSecretRef: example-token
  source:
    apiGroup: "kubevirt.io"
    kind: VirtualMachine
    name: example-vm
status:
  conditions:
  - lastProbeTime: null
    reason: podReady
    status: "True"
    type: Ready
  - lastProbeTime: null
    reason: pvcBound
    status: "True"
    type: PVCReady
  links:
    external:
      cert: |-
        -----BEGIN CERTIFICATE-----
        ...
        -----END CERTIFICATE-----
      volumes:
      - formats:
        - format: raw
          url: https://virt-exportproxy-example.example.com/api/export.kubevirt.io/v1beta1/namespaces/example/virtualmachineexports/example-export/volumes/example-dv/disk.img
        - format: gzip
          url: https://virt-exportproxy-example.example.com/api/export.kubevirt.io/v1beta1/namespaces/example/virtualmachineexports/example-export/volumes/example-dv/disk.img.gz
        name: example-disk
    internal:
      cert: |-
        -----BEGIN CERTIFICATE-----
        ...
        -----END CERTIFICATE-----
      volumes:
      - formats:
        - format: raw
          url: https://virt-export-example-export.example.svc/volumes/example-dv/disk.img
        - format: gzip
          url: https://virt-export-example-export.example.svc/volumes/example-dv/disk.img.gz
        name: example-disk
  phase: Ready
  serviceName: virt-export-example-export
```
请注意，在此示例中，我们位于源集群的 `example` 命名空间，这就是内部链接域名以 `.example.svc` 结尾的原因。外部链接是源集群外部可见的内容，因此在导入到目标集群时可以使用它们。

现在我们可以把此磁盘导入目标集群了。为了让 CDI 执行导入，我们需要提供包含以下内容的适当 yaml：
- CA 证书（作为 config map）
- 以 CDI 兼容格式访问磁盘镜像所需的令牌
- VM yaml
- DataVolume yaml（如果它不属于 VM 定义的一部分，则为可选）

virtctl 为 download 命令提供了一个名为 `--manifest` 的附加参数，它会从导出服务器检索适当的信息，并使用 `--output` 参数将其保存到文件，或写入标准输出。默认情况下，此输出不包含头 Secret，因为其中以明文包含令牌。要获取头 Secret，请指定 `--include-secret` 参数。默认输出格式是 `yaml`，但也可以获取 `json` 输出。

假设有一个名为 `example-export` 的正在运行的 VirtualMachineExport，并且目标集群中存在相同的命名空间。目标集群的 kubeconfig 名称为 `kubeconfig-target`，要把该 vm 克隆到目标集群，请运行以下命令：

```bash
$ virtctl vmexport download example-export --manifest --include-secret --output=import.yaml
$ kubectl apply -f import.yaml --kubeconfig=kubeconfig-target
```

第一条命令生成 yaml 并将其写入 `import.yaml`。第二条命令把生成的 yaml 应用到目标集群。可以把这两条命令合并：第一条命令写入标准 `out`，再通过管道传入第二条命令。如果导出令牌不应被写入任何文件，请使用此选项。这将在目标集群中创建 VM，并为目标集群中的 CDI 提供导入磁盘镜像所需的一切。

导入完成后，你应当能够在目标集群中启动该 VM。

#### 使用 virtctl vmexport 在本地下载 VM 卷 {#download-a-vm-volume-locally-using-virtctl-vmexport}

使用 `vmexport` 命令可以大大简化上一节中的若干步骤。

同样，假设集群中有一个 Ingress 或 Route 暴露了导出代理，并且集群中有一个带一块磁盘的虚拟机，如下所示：

```yaml
apiVersion: kubevirt.io/v1
kind: VirtualMachine
metadata:
  labels:
    kubevirt.io/vm: vm-example-datavolume
  name: example-vm
spec:
  dataVolumeTemplates:
  - metadata:
      creationTimestamp: null
      name: example-dv
    spec:
      storage:
        accessModes:
        - ReadWriteOnce
        resources:
          requests:
            storage: 20Gi
        storageClassName: local
      source:
        registry:
          url: docker://quay.io/containerdisks/centos-stream:9
  runStrategy: Halted
  template:
    metadata:
      labels:
        kubevirt.io/vm: vm-example-datavolume
    spec:
      domain:
        devices:
          disks:
          - disk:
              bus: virtio
            name: datavolumedisk1
        resources:
          requests:
            memory: 2Gi
      terminationGracePeriodSeconds: 0
      volumes:
      - dataVolume:
          name: example-dv
        name: datavolumedisk1
```

满足这些要求后，可以通过不同方式完成本地下载卷的过程：

##### 分别执行每个步骤 {#performing-each-step-separately}

我们可以用不同的命令执行每一个步骤来下载卷。首先创建导出对象：

```bash
# We use an arbitrary name for the VMExport object, but specify our VM name in the flag.

$ virtctl vmexport create vmexportname --vm=example-vm
```

然后，把卷下载到指定的输出：

```bash
# Since our virtual machine only has one volume, there's no need to specify the volume name with the --volume flag.

# After the download, the VMExport object is deleted by default, so we are using the optional --keep-vme flag to delete it manually.

$ virtctl vmexport download vmexportname --output=/tmp/disk.img --keep-vme
```

最后，删除 VMExport 对象：

```bash
$ virtctl vmexport delete vmexportname
```

##### 执行单个步骤 {#performing-one-single-step}

前面的所有步骤都可以简化为一条命令：

```bash
# Since we are using a create flag (--vm) with download, the command creates the object assuming the VMExport doesn't exist.

# Also, since we are not using --keep-vme, the VMExport object is deleted after the download.

$ virtctl vmexport download vmexportname --vm=example-vm --output=/tmp/disk.img
```

下载完成后，可以在 `/tmp/disk.img` 找到我们的磁盘。
