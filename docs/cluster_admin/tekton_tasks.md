# KubeVirt Tekton {#kubevirt-tekton}

### 前提条件 {#prerequisites}
- [Tekton](https://tekton.dev/)
- [KubeVirt](https://kubevirt.io/)
- [CDI](https://github.com/kubevirt/containerized-data-importer)

## KubeVirt Tekton 任务 {#kubevirt-tekton-tasks}
### 什么是 KubeVirt Tekton 任务？ {#what-are-kubevirt-tekton-tasks}
面向 KubeVirt 的 Tekton 任务，主要关注：

- 创建和管理资源（VM、DataVolume）
- 在 VM 中执行命令
- 使用 libguestfs 工具处理磁盘镜像

KubeVirt Tekton 任务和示例 Pipeline 可在 [artifacthub.io](https://artifacthub.io/packages/search?org=kubevirt&sort=relevance&page=1) 获取，你可以轻松地将它们部署到集群中。

### 现有任务 {#existing-tasks}

#### 创建虚拟机 {#create-virtual-machines}
- create-vm-from-manifest - 根据提供的清单或使用 virtctl 创建 VM。
- create-vm-from-template - 从模板创建 VM（仅适用于 OpenShift）。

#### 使用模板 {#utilize-templates}
- copy-template - 复制给定模板并创建一个新模板（仅适用于 OpenShift）。
- modify-vm-template - 使用用户提供的数据修改模板（仅适用于 OpenShift）。

#### 修改数据对象 {#modify-data-objects}
- modify-data-object - 创建 / 修改 / 删除 datavolume / datasource

#### 生成 SSH 密钥 {#generate-ssh-keys}
- generate-ssh-keys - 生成私钥和公钥对，并将其注入 VM。

#### 在虚拟机中执行命令 {#execute-commands-in-virtual-machines}
- execute-in-vm - 通过 SSH 在 VM 中执行命令。
- cleanup-vm - 执行命令和/或停止/删除 VM。

#### 使用 libguestfs 工具处理 PVC {#manipulate-pvcs-with-libguestfs-tools}
- disk-virt-customize - 在 PVC 中执行 virt-customize 命令。
- disk-virt-sysprep- 在 PVC 中执行 virt-sysprep 命令。

#### 等待虚拟机实例状态 {#wait-for-virtual-machine-instance-status}
- wait-for-vmi-status - 等待 VMI 进入运行状态。

#### 修改 Windows ISO {#modify-windows-iso}
- modify-windows-iso-file - 修改 Windows ISO（将带提示的引导加载程序替换为无提示的引导加载程序），并用更新后的 ISO 替换 PVC 中的原始 ISO。这有助于在 EFI 引导模式下自动安装 Windows。默认情况下，EFI 引导模式下的 Windows 使用带提示的引导加载程序，在按下按键之前不会继续引导过程。将其替换为无提示引导加载程序后，无需按键即可引导进入 Windows 安装程序。

### 示例 Pipeline {#example-pipeline}
所有这些任务都可用于创建 [Pipeline](https://github.com/tektoncd/pipeline/blob/main/docs/pipelines.md)。
我们准备了示例 Pipeline，展示可以用 KubeVirt 任务做什么。

- [Windows EFI 安装程序](https://github.com/kubevirt/kubevirt-tekton-tasks/blob/main/release/pipelines/windows-efi-installer/windows-efi-installer.yaml) - 此 Pipeline 将准备一个已安装 virtio 驱动的 Windows 10/11/2k22 datavolume。用户必须提供一个可用的 Windows 10/11/2k22 ISO 文件链接。此 Pipeline 适用于需要 EFI 的 Windows 版本（例如 Windows 10/11/2k22）。关于该 Pipeline 的更多信息见[此处](https://github.com/kubevirt/kubevirt-tekton-tasks/blob/main/release/pipelines/windows-efi-installer/README.md)

- [Windows 定制](https://github.com/kubevirt/kubevirt-tekton-tasks/blob/main/release/pipelines/windows-customize/windows-customize.yaml) - 此 Pipeline 将在 Windows VM 中安装 SQL Server 或 VS Code。关于该 Pipeline 的更多信息见[此处](https://github.com/kubevirt/kubevirt-tekton-tasks/blob/main/release/pipelines/windows-customize/README.md)

!!! note
    - 如果为 Pipeline 和任务定义了不同的命名空间，则必须创建一个[集群解析器](https://tekton.dev/docs/pipelines/cluster-resolver/)对象。<br />
    - 默认情况下，示例 Pipeline 会在 `kubevirt-os-images` 命名空间中创建生成的 datavolume。<br />
    - 如果希望在不同命名空间中创建生成的 datavolume（通过在 Pipeline 中指定 `baseDvNamespace` 属性），则需要额外的 RBAC 权限（所需的全部 RBAC 权限列表见[此处](https://github.com/kubevirt/kubevirt-tekton-tasks/tree/main/release/tasks/modify-data-object#usage-in-different-namespaces)）。<br />
    - 如果希望在给定 Pipeline 运行期间对 VM 进行实时迁移，必须满足以下[前提条件](../operations/live_migration.md#limitations)。
