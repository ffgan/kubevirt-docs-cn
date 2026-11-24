# 机密计算 {#confidential-computing}

## AMD 安全加密虚拟化（SEV） {#amd-secure-encrypted-virtualization-sev}

**功能状态：**
  - Alpha - `v0.49.0`（实验性支持）
  - Beta  - `v1.9.0`（默认启用）

[安全加密虚拟化（SEV）](https://developer.amd.com/sev/) 是 AMD EPYC CPU 的一项特性，允许在运行过程中对虚拟机内存进行加密。

KubeVirt 支持在具备 SEV 特性的 AMD EPYC 硬件上运行机密 VM。

### 前提条件 {#preconditions}

要运行 SEV 客户机，必须满足以下条件：

- 在 KubeVirt v1.9 之前：必须启用 `WorkloadEncryptionSEV` [功能门控](../cluster_admin/activating_feature_gates.md#how-to-activate-a-feature-gate)。
- 客户机必须支持 [UEFI 启动](../compute/virtual_hardware.md#biosuefi)
- 必须为客户机 VM 禁用 SecureBoot

### 运行 SEV 客户机 {#running-an-sev-guest}

可以通过在 VMI 定义中设置 `spec.domain.launchSecurity.sev` 元素来请求 SEV 内存加密：

```yaml
apiVersion: kubevirt.io/v1
kind: VirtualMachineInstance
metadata:
  labels:
    special: vmi-fedora
  name: vmi-fedora
spec:
  domain:
    launchSecurity:
      sev: {}
    firmware:
      bootloader:
        efi:
          secureBoot: false
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


### 当前限制 {#current-limitations}

- 经过 SEV 加密的 VM 不能包含可直接访问的主机设备（即 PCI 直通）
- 不支持实时迁移
- 这些 VM 不会经过证明

## AMD 安全加密虚拟化 - 安全嵌套分页（SEV-SNP） {#amd-secure-encrypted-virtualization-secure-nested-paging-sev-snp}

**功能状态：**
  - Alpha - `v1.7.0`（实验性支持）
  - Beta  - `v1.9.0`（默认启用）


KubeVirt 支持在具备 SEV-SNP 支持的 AMD EPYC 硬件上运行机密 VM。

### 前提条件 {#prerequisites}

- 在 KubeVirt v1.9 之前：必须启用 `WorkloadEncryptionSEV` [功能门控](../cluster_admin/activating_feature_gates.md#how-to-activate-a-feature-gate)。
- 客户机必须支持 [UEFI 启动](../compute/virtual_hardware.md#biosuefi)。
- 必须为客户机 VM 禁用 SecureBoot。
- 能够运行 [SEV-SNP](https://docs.amd.com/v/u/en-US/58207-using-sev-with-amd-epyc-processors) 的 AMD EPYC 硬件。

### 部署启用 AMD SEV-SNP 的 VM {#deploying-amd-sev-snp-enabled-vms}

可以通过在 VMI 定义中设置 `spec.domain.launchSecurity.snp` 元素来请求 SEV-SNP 内存加密：


```yaml
apiVersion: kubevirt.io/v1
kind: VirtualMachineInstance
metadata:
  labels:
    special: vmi-fedora
  name: vmi-fedora
spec:
  domain:
    launchSecurity:
      snp: {}
    firmware:
      bootloader:
        efi:
          secureBoot: false
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



### 限制 {#limitations}

- SEV/SEV-ES 与 SNP 互斥，同时运行 SEV 和 SNP 将无法运行。
- 使用 QEMU 所用的默认策略（例如 Policy: 0x30000）。
- 不支持实时迁移。
- 这些 VM 不会经过证明。

## 面向 Linux 的 IBM Secure Execution（Secure Execution） {#ibm-secure-execution-for-linux-secure-execution}

面向 Linux 的 IBM Secure Execution 是随 IBM z15 和 LinuxONE III 引入的一项 s390x 安全技术。它可以保护在 KVM 客户机中运行的工作负载数据，使其不会被服务器环境检查或修改。

特别是，硬件管理员、KVM 代码以及 KVM 管理员都无法访问以 IBM Secure Execution 客户机方式启动的客户机中的数据。

更多细节请阅读[官方文档](https://www.ibm.com/docs/en/linux-on-systems?topic=execution-introduction)。

### **功能状态：** {#feature-state}

* Alpha - [`v1.6.0`](https://github.com/kubevirt/kubevirt/releases/tag/v1.6.0)
* Beta - [`v1.7.0`](https://github.com/kubevirt/kubevirt/releases/tag/v1.7.0)
* GA - [`v1.9.0`](https://github.com/kubevirt/kubevirt/releases/tag/v1.9.0)（默认启用）

### 前提条件 {#prerequisites_1}

- IBM z15/LinuxONE III 或更新版本
- 带有 LPAR 工作节点的 Kubernetes 集群（无嵌套虚拟化）
- 集群上已部署 KubeVirt 和 CDI
- 已为 Secure Execution 准备好的客户机。参见[官方文档](https://www.ibm.com/docs/en/linux-on-systems?topic=execution-workload-owner-tasks)。
- 在 KubeVirt 1.9 之前：必须启用 `SecureExecution` [功能门控](../cluster_admin/activating_feature_gates.md#how-to-activate-a-feature-gate)。

### 准备宿主机 {#preparing-the-hosts}

在所有应当运行 Secure Execution VM 的 LPAR 工作节点上，需要执行以下步骤，使它们能够承载这些工作负载：

1. 通过 SSH 登录该节点
2. 将 `prot_virt=1` 添加到内核命令行：
  - 编辑 `zipl.conf`：
  ```
  # vi zipl.conf
  ...
  parameters=" ...prot_virt=1"
  ...
  ```
  - 运行 `zipl`，以确保下次重启时会采用该变更
  - 重启该节点
3. 验证 Secure Execution 现已启用：
```
$ cat /sys/firmware/uv/prot_virt_host
1
```


### 运行 Secure Execution 客户机 VM {#running-a-secure-execution-guest-vm}

确保你的工作负载已[使用 CDI 上传到数据卷](../storage/containerized_data_importer.md#virtctl-image-upload)。

可以通过在 VMI 定义中设置 `spec.domain.launchSecurity` 元素来请求基于 Secure Execution 的内存保护。
```
apiVersion: kubevirt.io/v1
kind: VirtualMachine
metadata:
  labels:
    kubevirt.io/vm: secure-execution-vm
  name: secure-execution-vm
spec:
  runStrategy: Always
  template:
    metadata:
      labels:
        kubevirt.io/vm: secure-execution-vm
    spec:
      domain:
        launchSecurity: {}
        devices:
          disks:
          - disk:
              bus: virtio
            name: rootfs
        resources:
          requests:
            memory: 4Gi
      terminationGracePeriodSeconds: 0
      volumes:
        - name: rootfs
          dataVolume:
            name: secure-execution-dv
```

### 限制 {#limitations_1}

由于宿主机不被允许访问客户机内存，某些功能无法与 Secure Execution VM 一起使用：
- 实时迁移
  - VM 需要改为离线迁移
  - 如果在不同机器之间迁移，VM 需要使用两台机器的 hostkey 进行加密
- 从磁盘保存和恢复 VM
- 内存转储
- 大页
- 主机设备直通，例如 PCI 和 CCW。
- 通过 virtio-balloon 设备进行内存气球调节。
