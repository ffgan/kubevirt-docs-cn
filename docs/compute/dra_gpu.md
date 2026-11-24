# 使用动态资源分配（DRA）分配 GPU {#assigning-gpus-with-dynamic-resource-allocation-dra}

**功能状态：** v1.9.0: Beta

KubeVirt 可以通过 Kubernetes [动态资源分配（DRA）](https://kubernetes.io/docs/concepts/scheduling-eviction/dynamic-resource-allocation/) 将 GPU 挂接到虚拟机。
借助 DRA，GPU 通过 Kubernetes 的 `ResourceClaim` API 请求，由外部 GPU DRA 驱动分配，并通过 `spec.resourceClaims` 和 `spec.domain.devices.gpus` 挂接到 VM。

这是 [主机设备分配](host-devices.md) 中所记录的设备插件流程的替代方案。
使用 DRA 时，无需在 `permittedHostDevices` 中将 GPU 列入允许列表。
相反，DRA 驱动会发布 `ResourceSlice` 对象，KubeVirt 从挂载进 virt-launcher 的文件中读取设备元数据。

!!! Note
    在本指南中，**容器设备接口（CDI）** 指 DRA 驱动用来把设备节点注入 virt-launcher Pod 的 Kubernetes 机制，而不是 KubeVirt 用于磁盘导入工作流的 [容器化数据导入器](../storage/containerized_data_importer.md)。

`GPUsWithDRA` 功能门控处于 Beta，并从 v1.9.0 起默认启用。
唯一更早支持它的发行版是 v1.8.x，在该版本中必须手动启用。

请参阅 [激活功能门控](../cluster_admin/activating_feature_gates.md)，以便在更早的版本上启用它，或者禁用它。

## 前提条件 {#prerequisites}

- **GPU DRA 驱动。** 在集群中安装兼容的外部 GPU DRA 驱动。

    本指南针对生产集群介绍 [NVIDIA k8s-dra-driver-gpu](https://github.com/kubernetes-sigs/dra-driver-nvidia-gpu)。

    该驱动必须发布 [DRA 设备元数据（KEP-5304）](https://github.com/kubernetes/enhancements/tree/master/keps/sig-node/5304-dra-attributes-downward-api)——诸如 `resource.kubernetes.io/pciBusID` 之类的设备属性——这些属性位于挂载进 virt-launcher 的 JSON 文件中。
    KubeVirt 读取这些文件，以便为 GPU 构建 libvirt 域 XML。
    这与 CDI 在运行时挂载的 VFIO 设备节点是分开的：CDI 提供 `/dev/vfio/*` 设备节点；元数据文件标识要挂接的 PCI 设备。
    驱动作者可以在 [访问 DRA 设备元数据](https://kubernetes.io/docs/tasks/configure-pod-container/assign-resources/access-dra-device-metadata/) 中找到发布这些属性的详细说明。

- **直通 GPU 的宿主机准备。** 对于直通分配，GPU 节点必须满足通常的 PCI 直通要求（已启用 IOMMU，并且 `vfio-pci` 可用）。
    请参阅 [主机设备分配](host-devices.md#host-preparation-for-pci-passthrough)。

## 概述 {#overview}

总体工作流程如下：

1. 在集群上安装 GPU DRA 驱动。
2. 创建从该驱动请求 GPU 的 `ResourceClaimTemplate`。
3. 创建引用该 claim 并将其映射到 `spec.domain.devices.gpus` 的 VMI。

## 安装 DRA GPU 驱动 {#installing-the-dra-gpu-driver}

### NVIDIA GPU 的 DRA 驱动 {#dra-driver-for-nvidia-gpus}

安装 [dra-driver-nvidia-gpu](https://github.com/kubernetes-sigs/dra-driver-nvidia-gpu)。

有关安装细节，请参阅 [安装指南](https://dra-driver-nvidia-gpu.sigs.k8s.io/docs/install/)。

有关在 DRA 驱动中启用 GPU 直通支持，请参阅 [KubeVirt VFIO GPU 直通指南](https://dra-driver-nvidia-gpu.sigs.k8s.io/docs/guides/kubevirt-vfio-gpu-passthrough/)。

## 创建 ResourceClaimTemplate {#creating-a-resourceclaimtemplate}

创建从 DRA 驱动请求 GPU 的 `ResourceClaimTemplate`。
将 `vfio.gpu.nvidia.com` 替换为你的驱动所发布的 `DeviceClass` 名称。

在使用面向 KubeVirt 直通的 NVIDIA GPU DRA 驱动时，请包含下面不透明的 `VfioDeviceConfig` 块。
NVIDIA 驱动默认不会注入 VFIO API 设备。
更多细节请参阅驱动指南中的 [VfioDeviceConfig](https://dra-driver-nvidia-gpu.sigs.k8s.io/docs/guides/kubevirt-vfio-gpu-passthrough/#vfiodeviceconfig-parameters) 参数。

```yaml
apiVersion: resource.k8s.io/v1
kind: ResourceClaimTemplate
metadata:
  name: dra-gpu-claim-template
spec:
  spec:
    devices:
      config:
        - requests:
            - dra-gpu
          opaque:
            driver: gpu.nvidia.com
            parameters:
              apiVersion: resource.nvidia.com/v1beta1
              kind: VfioDeviceConfig
              iommu:
                backendPolicy: LegacyOnly
                enableAPIDevice: true
      requests:
        - name: dra-gpu
          exactly:
            allocationMode: ExactCount
            count: 1
            deviceClassName: vfio.gpu.nvidia.com
```

## 创建带有 DRA GPU 的 VMI {#creating-a-vmi-with-a-dra-gpu}

在 `spec.resourceClaims` 中引用 claim 模板，然后把已分配的 GPU 映射到 `spec.domain.devices.gpus`，映射时使用 `claimName` 和 `requestName`。

```yaml
apiVersion: kubevirt.io/v1
kind: VirtualMachineInstance
metadata:
  name: vm-dra-gpu
  labels:
    app: vm-dra-gpu-test
spec:
  resourceClaims:
    - name: dra-gpu-claim
      resourceClaimTemplateName: dra-gpu-claim-template
  domain:
    resources:
      requests:
        memory: 128Mi
    cpu:
      cores: 1
    devices:
      disks:
        - name: containerdisk
          disk:
            bus: virtio
      interfaces:
        - name: default
          masquerade: {}
      gpus:
        - name: gpu0
          claimName: dra-gpu-claim
          requestName: dra-gpu
  networks:
    - name: default
      pod: {}
  volumes:
    - name: containerdisk
      containerDisk:
        image: quay.io/kubevirt/cirros-container-disk-demo:latest
```

!!! warning "重要"
    **这些字段必须互相引用：**

    - `spec.resourceClaims[].resourceClaimTemplateName` 必须匹配 `metadata.name`，也就是第 2 步所创建 `ResourceClaimTemplate` 的名称（本示例中为 `dra-gpu-claim-template`）。
    - `spec.domain.devices.gpus[].claimName` 必须与支撑该设备的 claim 的 `spec.resourceClaims[].name` 匹配（本示例中为 `dra-gpu-claim`）。
    - `spec.domain.devices.gpus[].requestName` 必须匹配请求 `name`，该名称定义于 `ResourceClaimTemplate` 中（本示例中为 `dra-gpu`）。

当 VMI 启动时，virt-launcher 使用元数据文件中的属性（例如 `resource.kubernetes.io/pciBusID`）以及 CDI 挂载的 `/dev/vfio/*` 设备节点，在 libvirt 域 XML 中配置 GPU 直通。
