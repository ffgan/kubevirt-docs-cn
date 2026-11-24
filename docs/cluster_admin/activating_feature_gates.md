# 启用与停用功能门控 {#activating-and-deactivating-feature-gates}

KubeVirt 有一组尚未成熟到足以默认启用的功能。因此，它们受到 Kubernetes 中称为
[功能门控](https://kubernetes.io/docs/reference/command-line-tools-reference/feature-gates/) 的概念保护。

## 功能门控生命周期 {#feature-gate-lifecycle}

KubeVirt 功能会经历多个成熟度阶段，每个阶段的默认行为不同：

| 阶段 | 默认状态 | 如何更改 |
|---|---|---|
| **Alpha** | 已禁用 | 通过 `featureGates` 启用 |
| **Beta** | 已启用（自 v1.9 起） | 通过 `disabledFeatureGates` 停用 |
| **GA** | 始终启用 | 无法停用 |

- **Alpha** 功能是实验性的，必须显式启用。
  它们可能不完整、包含破坏性 API 变更，或被完全移除。
- **Beta** 功能被认为已足够稳定，可供广泛测试。
  从 KubeVirt v1.9 起，所有 Beta 功能门控
  **默认启用**。完整细节见
  [VEP 229：Beta 功能默认开启](https://github.com/kubevirt/enhancements/blob/main/veps/meta-VEPs/229-beta-features-on-by-default/vep.md)
  以及[博客文章](https://kubevirt.io/2026/Beta-Features-On-By-Default-In-v1-9.html)。
- **GA** 功能始终启用，且无法切换。在此阶段，
  功能门控本身被视为已过时。

## 功能门控报告 {#feature-gate-report}

从 v1.9 起，KubeVirt 会在每个发行版的制品中附带一份**功能门控报告**。
该 JSON 文件列出每一个非 GA 功能门控及其当前状态（Alpha、Beta 或已弃用）。

你可以在每个版本的
[发行制品](https://github.com/kubevirt/kubevirt/releases)
中找到它，也可以从 KubeVirt 源码树自行生成：

```bash
make feature-gate-report
```

完整的功能门控列表也可以直接从
[源代码](https://github.com/kubevirt/kubevirt/blob/main/pkg/virt-config/featuregate/active.go) 查看。

## 如何启用功能门控 {#how-to-activate-a-feature-gate}

Alpha 功能必须显式启用。你可以在 KubeVirt 的 CR 中启用功能门控：

```bash
cat << END > enable-feature-gate.yaml
---
apiVersion: kubevirt.io/v1
kind: KubeVirt
metadata:
  name: kubevirt
  namespace: kubevirt
spec:
  configuration:
    developerConfiguration:
      featureGates:
        - AlphaFeatureToEnable
END

kubectl apply -f enable-feature-gate.yaml
```

也可以编辑现有的 KubeVirt CR：

```bash
kubectl edit kubevirt kubevirt -n kubevirt
```

```yaml
...
spec:
  configuration:
    developerConfiguration:
      featureGates:
        - AlphaFeatureToEnable
```

**注意：**功能门控的名称区分大小写。

上面的代码片段假定 KubeVirt 安装在 `kubevirt`
命名空间中。请根据你的安装更改命名空间。

## 如何停用功能门控 {#how-to-disable-a-feature-gate}

从 KubeVirt v1.8 起，你可以使用 `disabledFeatureGates` 字段显式停用功能门控。从 v1.9 起，这是退出默认启用的 Beta 功能的唯一方式。

**建议在生产环境中停用所有 Beta 功能门控**，除非你已针对自己的环境明确验证过它们。可使用发行制品中的功能门控报告以编程方式填充此列表。

要停用特定功能门控，请将它们添加到
`disabledFeatureGates` 列表中：

```bash
cat << END > disable-feature-gate.yaml
---
apiVersion: kubevirt.io/v1
kind: KubeVirt
metadata:
  name: kubevirt
  namespace: kubevirt
spec:
  configuration:
    developerConfiguration:
      disabledFeatureGates:
        - BetaFeatureToDisable
END

kubectl apply -f disable-feature-gate.yaml
```

也可以编辑现有的 KubeVirt CR：

```bash
kubectl edit kubevirt kubevirt -n kubevirt
```

```yaml
...
spec:
  configuration:
    developerConfiguration:
      disabledFeatureGates:
        - BetaFeatureToDisable
```

!!! warning "重要"
    功能门控不能同时列在 `featureGates` 和
    `disabledFeatureGates` 中。否则将导致
    校验错误。

## 升级注意事项 {#upgrade-considerations}

升级到 v1.9 或更高版本时，请注意所有 Beta 功能
将默认变为启用状态。请查阅目标版本的功能门控报告，
并按需配置 `disabledFeatureGates`。

`disabledFeatureGates` 机制在 v1.8 中引入。如果
你从 v1.7 或更早版本直接升级到 v1.9，Beta
功能将无条件启用，直到你在升级后修补
KubeVirt CR。建议先升级到 v1.8。
