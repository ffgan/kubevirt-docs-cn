# 用于 VM 状态 PVC 的 CSI 叠加 {#csi-overlay-for-vm-state-pvcs}

VM 状态 PVC 最初被引入，是为需要使用 TPM 或 EFI 设备的 VM 提供持久存储：

* `spec.domain.devices.tpm.persistent: true`
* `spec.domain.devices.efi.persistent: true`

VM 状态 PVC 目前只需要存储少量数据，大约 ~10MB。然而，许多存储提供程序对 PVC 有最小大小要求，从 1GB 到 50GB 不等。当这一要求应用于集群中的所有 VM 时，这些 VM 状态 PVC 中就会产生大量被浪费的存储，从而可能让你更快触及存储和平台限制。

解决方案是部署一个经过修改的 Hostpath Provisioner（HPP），将其“叠加”在具备 RWX 文件系统能力的存储提供程序（例如 NFS）之上。

我们可以通过利用 HPP 的 [PVCTemplate 存储池](https://github.com/kubevirt/hostpath-provisioner-operator?tab=readme-ov-file#custom-resource-with-pvctemplate-storage-pool) 来实现这种叠加。这允许我们定义一个使用单个 RWX 共享 PVC 的存储池。底层存储提供程序会在网络共享位置创建该 RWX PVC，HPP operator 会在集群的每个节点上挂载此卷。
存储池设置完成后，任何使用此设置的新卷都将由 HPP 置备，生成的 PVC 会作为网络共享之上的一个目录被创建，从而可以在任意节点上访问。

这使 VM 状态 PVC 能够绕过某些存储置备器施加的最小 PVC 大小要求，并消除在 VM 迁移期间重新创建这些 PVC 的需要。

下面是部署和配置此 CSI 叠加的步骤。

## 部署 HPP Operator {#deploy-hpp-operator}

hostpath provisioner operator 要求在部署该 operator 之前已安装 [cert-manager](https://github.com/cert-manager/cert-manager)。这是因为该 operator 有一个 validating webhook，用于验证 CR 的内容是否有效。在部署该 operator 之前，需要安装 cert-manager：

```bash
kubectl create -f https://github.com/cert-manager/cert-manager/releases/latest/download/cert-manager.yaml
```

确保 cert-manager 完全可用
```bash
kubectl wait --for=condition=Available -n cert-manager --timeout=120s --all deployments
```


接下来，创建 hostpath provisioner 命名空间：
```bash
kubectl create -f https://github.com/kubevirt/hostpath-provisioner-operator/releases/latest/download/namespace.yaml
```

现在部署 webhook
```bash
kubectl create -f https://github.com/kubevirt/hostpath-provisioner-operator/releases/latest/download/webhook.yaml -n hostpath-provisioner
```

然后创建该 operator
```bash
kubectl create -f https://github.com/kubevirt/hostpath-provisioner-operator/releases/latest/download/operator.yaml -n hostpath-provisioner
```

等待 operator 就绪
```bash
kubectl rollout status -n hostpath-provisioner deployment/hostpath-provisioner-operator --timeout=120s
```

## 部署 HPP 自定义资源（CR） {#deploy-the-hpp-custom-resource-cr}
既然 operator 已经安装，我们需要部署 hostpath provisioner CR，以表明我们希望将该 provisioner 用作存储叠加方案。

我们可以通过把 `storagePool` 配置为使用定义了 RWX PVC 的 `pvcTemplate` 来做到这一点。

!!! Note
    只有当 `pvcTemplate.accessModes` 设置为 `ReadWriteMany`（位于 `storagePool` 定义中）时，该 operator 才会部署并配置叠加资源
    
当 HPP Operator 看到存储池以这种方式配置时，它会把 RWX 卷挂载到集群中的每个节点，然后创建一个新的 `StorageClass`，用于从该存储池置备新卷。

!!! Note 
    可以通过设置可选的 `overlayClassName` 字段（位于 `storagePool` 定义中）来定义新存储类的名称。如果未提供，默认名称将是 `hpp-overlay`。

现在，当任何新的 PVC 被配置为使用这个新存储类时，它们的卷将作为网络共享内的新目录被创建。（在此示例中位于 `/nfs-vol`）



```bash
cat <<- END | kubectl apply -n hostpath-provisioner -f -
apiVersion: hostpathprovisioner.kubevirt.io/v1beta1
kind: HostPathProvisioner
metadata:
  name: hostpath-provisioner
spec:
  featureGates: 
    - "Snapshotting"
  imagePullPolicy: IfNotPresent
  storagePools:
    - name: "nfs-backend"
      path: "/nfs-vol"
      snapshotProvider: "reflink"
      overlayClassName: "hpp-overlay"
      pvcTemplate:
        accessModes:
          - ReadWriteMany
        storageClassName: "nfs-csi"
        resources:
          requests:
            storage: 10Gi
  workload:
    nodeSelector:
      kubernetes.io/os: linux
END
```

等待 hotpath provisioner 可用
```bash
kubectl wait hostpathprovisioners.hostpathprovisioner.kubevirt.io/hostpath-provisioner --for=condition=Available --timeout=480s
```


## 修改 KubeVirt CR {#modify-kubevirt-cr}

一旦我们修改后的 HPP CR 已部署并且新的叠加存储类已创建，就需要把 KubeVirt CR 中的 `vmStateStorageClass` 配置变量设置为使用这个新的叠加存储类。此后，任何需要 VM 状态 PVC 的新 VM 都将采用这种新的叠加 CSI 技术。

```bash
kubectl edit kubevirt -n kubevirt
```

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
        - VMPersistentState
    vmStateStorageClass: hpp-overlay  # NOTE! this name needs to match with newly created overlay storage class .
```

## 快照 / 恢复支持 {#snapshot-restore-support}

要对使用此 CSI 叠加作为其 VM 状态 PVC 的 VM 创建快照，必须向集群部署一个 `volumesnapshotclass`。

!!! Note 
    下面的 snapshotclass 被设置为指向名为 "nfs-backend" 的存储池，该存储池在上面的示例 CR 中定义。

```bash
kubectl create -f https://github.com/kubevirt/hostpath-provisioner-operator/releases/latest/download/volumesnapshotclass.yaml
```
