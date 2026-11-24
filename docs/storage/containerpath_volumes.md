# ContainerPath 卷 {#containerpath-volumes}

ContainerPath 卷允许 VirtualMachine 通过 virtiofs 访问 virt-launcher Pod 文件系统中的文件和目录。这使 VM 能够使用由 Kubernetes 或平台特定机制（例如云提供商工作负载身份 webhook）动态注入到 Pod 中的数据。

## 启用 ContainerPath 卷 {#enabling-containerpath-volumes}

必须通过 KubeVirt CR 中的 `ContainerPathVolumes`
[功能门控](../cluster_admin/activating_feature_gates.md#how-to-activate-a-feature-gate)
启用 ContainerPath 卷支持：

```yaml
apiVersion: kubevirt.io/v1
kind: KubeVirt
metadata:
  name: kubevirt
  namespace: kubevirt
spec:
  configuration:
    developerConfiguration:
      featureGates:
      - ContainerPathVolumes
```

## 动机 {#motivation}

Kubernetes 和云平台提供了在运行时向 Pod 注入凭据的机制：

- **投射的服务账号令牌**，用于工作负载身份（例如 AWS IRSA、Azure Workload Identity）
- **动态密钥**，通过准入 webhook（例如 HashiCorp Vault agent、cert-manager）
- **机密计算**证明令牌，以及由平台基础设施注入的运行时密钥

这些机制把数据直接注入 Pod 容器，但在这些 Pod 内运行的 VM 无法原生访问这些被注入的数据。ContainerPath 卷通过 virtiofs 把 virt-launcher 容器中的特定路径暴露给客户机 VM，从而弥合这一差距。

## 基本用法 {#basic-usage}

ContainerPath 卷在 VM spec 中需要两样东西：

1. 一个带有 `containerPath` 源的 **volume** 条目，用于指定 virt-launcher Pod 内的路径
2. 一个对应的 **filesystem** 设备条目，按名称引用该卷

```yaml
apiVersion: kubevirt.io/v1
kind: VirtualMachine
metadata:
  name: my-vm
spec:
  runStrategy: Always
  template:
    spec:
      domain:
        devices:
          filesystems:
          - name: my-data
            virtiofs: {}
          disks:
          - name: containerdisk
            disk:
              bus: virtio
        resources:
          requests:
            memory: 1Gi
      volumes:
      - name: containerdisk
        containerDisk:
          image: quay.io/containerdisks/fedora:latest
      - name: my-data
        containerPath:
          path: /path/to/data
```

在客户机 VM 内挂载该文件系统：

```bash
mount -t virtiofs my-data /mnt/data
```

## 示例：AWS IRSA {#example-aws-irsa}

一个常见用例是让 VM 使用服务账号的 IAM 角色（IRSA）向 AWS 进行身份验证。

当 Pod 使用带有 IRSA 注解的 ServiceAccount 时，EKS 会自动在 `/var/run/secrets/eks.amazonaws.com/serviceaccount/` 注入一个投射令牌。VM 可以通过 ContainerPath 卷访问此令牌。

有关 AWS IRSA 设置的先决条件，请参阅 [Amazon EKS IRSA 文档](https://docs.aws.amazon.com/eks/latest/userguide/iam-roles-for-service-accounts.html)。

首先，创建一个带有 IRSA 注解的 ServiceAccount：

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: aws-workload-sa
  annotations:
    eks.amazonaws.com/role-arn: "<iam-role-arn>"
```

然后创建一个 VirtualMachine，使用该 ServiceAccount，并通过 ContainerPath 卷暴露注入的令牌：

```yaml
apiVersion: kubevirt.io/v1
kind: VirtualMachine
metadata:
  name: aws-workload
spec:
  runStrategy: Always
  template:
    spec:
      domain:
        devices:
          filesystems:
          - name: sa-volume
            virtiofs: {}
          - name: aws-token
            virtiofs: {}
          disks:
          - name: containerdisk
            disk:
              bus: virtio
        resources:
          requests:
            memory: 2Gi
      volumes:
      - name: containerdisk
        containerDisk:
          image: quay.io/containerdisks/fedora:latest
      - name: sa-volume
        serviceAccount:
          serviceAccountName: aws-workload-sa
      - name: aws-token
        containerPath:
          path: /var/run/secrets/eks.amazonaws.com/serviceaccount
```

**为什么需要这两个卷？**

- `sa-volume`：确保 virt-launcher Pod 使用带有 IRSA 注解的 ServiceAccount，从而触发令牌注入。
- `aws-token`：通过 virtiofs 把注入的 EKS 令牌路径暴露给 VM。

在 VM 内：

```bash
# Mount the IRSA token filesystem
mount -t virtiofs aws-token /mnt/aws-creds

# Configure AWS SDK to use web identity token
export AWS_WEB_IDENTITY_TOKEN_FILE=/mnt/aws-creds/token
export AWS_ROLE_ARN="<iam-role-arn>"

# AWS CLI and SDKs will use these credentials
aws s3 ls
```

## 示例：Azure Workload Identity {#example-azure-workload-identity}

Azure Workload Identity 会把联合令牌注入到 Pod 的 `/var/run/secrets/azure/tokens`。要启用此功能，Pod 模板必须包含标签 `azure.workload.identity/use: "true"`，并使用带有 Azure 托管标识客户端 ID 注解的 ServiceAccount。

有关 Azure Workload Identity 设置的先决条件，请参阅 [AKS 工作负载身份文档](https://learn.microsoft.com/en-us/azure/aks/workload-identity-overview)。

首先，为工作负载身份创建一个 ServiceAccount：

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: azure-workload-sa
  annotations:
    azure.workload.identity/client-id: "<managed-identity-client-id>"
    # Optional if tenant is configured cluster-wide:
    # azure.workload.identity/tenant-id: "<tenant-id>"
```

Azure 托管标识还必须拥有针对此 ServiceAccount 主体的联合凭据（例如 `system:serviceaccount:<namespace>:azure-workload-sa`）。

然后创建一个 VirtualMachine，使用此 ServiceAccount，并暴露注入的令牌路径：

```yaml
apiVersion: kubevirt.io/v1
kind: VirtualMachine
metadata:
  name: azure-workload
spec:
  runStrategy: Always
  template:
    metadata:
      labels:
        azure.workload.identity/use: "true"
    spec:
      domain:
        devices:
          filesystems:
          - name: sa-volume
            virtiofs: {}
          - name: az-token
            virtiofs: {}
          disks:
          - name: containerdisk
            disk:
              bus: virtio
        resources:
          requests:
            memory: 2Gi
      volumes:
      - name: containerdisk
        containerDisk:
          image: quay.io/containerdisks/fedora:latest
      - name: sa-volume
        serviceAccount:
          serviceAccountName: azure-workload-sa
      - name: az-token
        containerPath:
          path: /var/run/secrets/azure/tokens
```

**为什么需要这两个卷？**

- `sa-volume`：确保 virt-launcher Pod 使用工作负载身份 ServiceAccount，从而触发令牌注入。
- `az-token`：通过 virtiofs 把注入的 Azure 令牌路径暴露给 VM。

在 VM 内：

```bash
# Mount the Azure token filesystem
mount -t virtiofs az-token /mnt/az-creds

# Configure Azure SDK to use workload identity token
export AZURE_FEDERATED_TOKEN_FILE=/mnt/az-creds/azure-identity-token
export AZURE_CLIENT_ID="<managed-identity-client-id>"

# Azure CLI and SDKs can use these credentials
```

## Webhook 注入的卷 {#webhook-injected-volumes}

准入 webhook 可以把卷注入 virt-launcher Pod。使用 ContainerPath 将这些卷暴露给 VM：

```yaml
volumes:
- name: injected-config
  containerPath:
    path: /opt/injected-config
```

`spec.domain.devices.filesystems` 中还需要一个对应的文件系统设备。

## 路径要求 {#path-requirements}

- 指定的 `path` 必须是绝对路径，并且存在于 virt-launcher Pod 的 `compute` 容器内
- 该路径必须对应 compute 容器中的某个 volumeMount（或是其子路径）
- 该路径应在 VM 启动前就存在，或者由边车或 webhook 注入的机制持续填充
- 包含 `..` 分量、或符号链接会逸出 volumeMount 边界的路径会被拒绝
- 从 VM 的角度看，这些路径是只读的
- 该路径不得与 KubeVirt 内部挂载点冲突

## 支持的卷类型 {#supported-volume-types}

ContainerPath 卷只支持由以下 Kubernetes 卷类型支持的路径：

| 卷类型 | 描述 |
|---|---|
| ConfigMap | 配置数据 |
| Secret | 凭据之类的敏感数据 |
| Projected | ConfigMap、Secret、DownwardAPI 和 ServiceAccountToken 的组合 |
| DownwardAPI | Pod 和容器元数据 |
| EmptyDir | 临时的 Pod 本地存储 |

其他卷类型（PVC、HostPath 等）不受支持。

## 实时迁移 {#live-migration}

ContainerPath 卷不会阻止实时迁移，但迁移之后数据是否仍然可访问，取决于该路径是如何被填充的：

**通常可以随迁移正常工作：**

- **Secret、ConfigMap、ServiceAccount 令牌** — Kubernetes 会在目标节点上重新投射这些内容
- **由边车填充的卷** — 如果某个边车容器填充该路径（例如 Vault agent），边车会在目标节点上运行并重新填充数据

**无法随迁移正常工作：**

- **没有边车的 EmptyDir 卷** — 数据不会被复制到目标节点

在实时迁移中使用 ContainerPath 卷时，请验证填充容器路径的机制在目标节点上能够正常工作。

## 安全注意事项 {#security-considerations}

- VM 可以访问指定容器路径内的任何文件 — 只暴露包含供 VM 使用的数据的路径
- 使用 RBAC 和准入策略来控制哪些服务账号和角色可以与 VM 一起使用
- ContainerPath 卷继承 virt-launcher Pod 的安全上下文
- 只允许使用[支持的卷类型](#supported-volume-types)

## 故障排除 {#troubleshooting}

### VM 的 Synchronized=False 且原因为 MissingVirtiofsContainers {#vm-has-synchronizedfalse-with-missingvirtiofscontainers}

如果 VM 报告 `Synchronized=False`，原因为 `MissingVirtiofsContainers`，则指定的路径在 virt-launcher Pod 中不存在。

检查 compute 容器中的路径：

```bash
kubectl exec -n <namespace> <virt-launcher-pod> -c compute -- ls -la /path/to/volume
```

常见原因：

- `path` 字段中有拼写错误
- 预期的卷未被注入（使用 `kubectl get pod <virt-launcher-pod> -o yaml` 检查 Pod spec）
- 时序问题：路径在 virtiofs 初始化之后才被填充
- 未启用 `ContainerPathVolumes` 功能门控
