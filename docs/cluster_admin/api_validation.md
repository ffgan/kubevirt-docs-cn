# API 校验 {#api-validation}

KubeVirt 的 VirtualMachineInstance API 使用 Kubernetes 自定义资源定义（CRD）实现。因此，KubeVirt 能够利用 Kubernetes 提供的若干功能，在对象于集群上创建和更新时对我们的 API 执行校验检查。

## API 校验如何工作 {#how-api-validation-works}

### CRD OpenAPIv3 模式 {#crd-openapiv3-schema}

KubeVirt API 在安装时通过一系列 CRD 定义向 Kubernetes 注册。KubeVirt 在这些定义中包含 OpenAPIv3 模式，向 Kubernetes Apiserver 指明关于我们 API 的一些非常基本的信息，例如哪些字段是必需的，以及每个值期望的数据类型。

此 OpenAPIv3 模式校验会自动安装，用户无需额外操作即可启用。

### 准入控制 Webhook {#admission-control-webhooks}

OpenAPIv3 模式校验是有限的。它只校验 KubeVirt 对象的总体结构看起来是否正确。它并不验证该对象的内容是否合理。

仅靠 OpenAPIv3 校验，用户很容易犯简单错误（例如磁盘没有正确引用卷的名称），而集群仍会接受该对象。然而，如果 API 中存在这些错误，VirtualMachineInstance 当然无法启动。理想情况下，我们希望尽早发现配置问题，一旦检测到对象的 Spec 存在问题，就不允许该对象被提交到集群。

为了执行这种高级校验，KubeVirt 实现了自己的准入控制器，并以准入控制器 webhook 的形式向 Kubernetes 注册。此 webhook 在安装时向 Kubernetes 注册。当 KubeVirt 对象被提交到集群时，Kubernetes API server 会在将对象持久化到存储之前，把创建请求转发给我们的 webhook 进行校验。

但请注意，要启用 KubeVirt 准入控制器，需要先在集群上启用相应功能。

### 在 Kubernetes 上启用 KubeVirt 准入控制器 {#enabling-kubevirt-admission-controller-on-kubernetes}

在置备新的 Kubernetes 集群时，请确保 Apiserver 的 **--admission-control** 命令行参数中同时包含 **MutatingAdmissionWebhook** 和 **ValidatingAdmissionWebhook** 这两个值。

以下是我们在开发期间使用的 **--admission-control** 值示例

```
--admission-control='Initializers,NamespaceLifecycle,LimitRanger,ServiceAccount,DefaultStorageClass,DefaultTolerationSeconds,NodeRestriction,MutatingAdmissionWebhook,ValidatingAdmissionWebhook,ResourceQuota'
```

请注意，旧的 --admission-control 标志已在 1.10 中弃用，并被 --enable-admission-plugins 取代。
**MutatingAdmissionWebhook** 和 **ValidatingAdmissionWebhook** 默认启用。

### 在 OKD 上启用 KubeVirt 准入控制器 {#enabling-kubevirt-admission-controller-on-okd}

OKD 同样要求在安装时启用准入控制 webhook。不过过程略有不同。在 OKD 中，我们使用准入插件来启用 webhook。

可以在 openshift-ansible 中通过在 ansible 清单文件里设置以下值来配置这些准入控制插件。

```
openshift_master_admission_plugin_config={"ValidatingAdmissionWebhook":{"configuration":{"kind": "DefaultAdmissionConfig","apiVersion": "v1","disable": false}},"MutatingAdmissionWebhook":{"configuration":{"kind": "DefaultAdmissionConfig","apiVersion": "v1","disable": false}}}
```
