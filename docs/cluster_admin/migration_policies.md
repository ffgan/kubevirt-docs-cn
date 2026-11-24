# 迁移策略 {#migration-policies}

迁移策略提供了一种将迁移配置应用到虚拟机的新方式。这些策略可以细化 Kubevirt CR 中用于设置集群范围迁移配置的 `MigrationConfiguration`。这样，集群范围的设置作为默认值，可以由迁移策略加以细化（即更改、移除或添加）。

请注意，迁移策略目前处于 `v1alpha1` 版本。这意味着此 API 尚未完全稳定，并且 API 将来可能会发生变化。

## 概述 {#overview}

KubeVirt 支持虚拟机工作负载的[实时迁移](../compute/live_migration.md)。
在引入迁移策略之前，迁移设置只能在集群范围内配置，方法是编辑 [KubevirtCR 的 spec](https://kubevirt.io/api-reference/master/definitions.html#_v1_kubevirtspec)，或更具体地说是 [MigrationConfiguration](https://kubevirt.io/api-reference/master/definitions.html#_v1_migrationconfiguration) CRD。

可以自定义的迁移行为的若干方面（尽管不是全部）包括：
- 带宽
- 自动收敛
- 后拷贝/预拷贝
- 最大并行迁移数
- 超时

迁移策略泛化了定义迁移配置的概念，从而可以对特定的 VM 组应用不同的配置。

这种能力在许多需要区分不同工作负载的用例中都很有用。之所以需要区分不同配置，可能是因为不同工作负载被视为具有不同优先级、需要安全隔离、具有不同需求，需要帮助不适合迁移的工作负载完成收敛，以及许多其他原因。

## API 示例 {#api-examples}

#### 迁移配置 {#migration-configurations}
目前，MigrationPolicy 的 spec 只会包含来自 KubevirtCR 的 MigrationConfiguration 中的以下配置（将来打算添加更多不属于 Kubevirt CR 的配置）：
```yaml
apiVersion: migrations.kubevirt.io/v1alpha1
kind: MigrationPolicy
spec:
    allowAutoConverge: true
    bandwidthPerMigration: 217Ki
    completionTimeoutPerGiB: 23
    allowPostCopy: false
```

以上所有字段都是可选的。省略时，将按 KubevirtCR 的 MigrationConfiguration 中的定义应用配置。这样，KubevirtCR 将作为一组可配置的默认值，同时适用于未绑定任何 MigrationPolicy 的 VM，以及绑定到未定义全部配置字段的 MigrationPolicy 的 VM。

#### 将策略匹配到 VM {#matching-policies-to-vms}

spec 中接下来是选择器，用于定义要应用该策略的 VM 组。可用方式如下。

**此策略适用于位于具有全部所需标签的命名空间中的 VM：**
```yaml
apiVersion: migrations.kubevirt.io/v1alpha1
kind: MigrationPolicy
spec:
  selectors:
    namespaceSelector:
      hpc-workloads: true       # Matches a key and a value 
```

**此策略适用于具有全部所需标签的 VM：**
```yaml
apiVersion: migrations.kubevirt.io/v1alpha1
kind: MigrationPolicy
spec:
  selectors:
    virtualMachineInstanceSelector:
      workload-type: db       # Matches a key and a value 
```

**也可以将前两种方式组合起来：**
```yaml
apiVersion: migrations.kubevirt.io/v1alpha1
kind: MigrationPolicy
spec:
  selectors:
    namespaceSelector:
      hpc-workloads: true
    virtualMachineInstanceSelector:
      workload-type: db
```

#### 完整清单： {#full-manifest}

```yaml
apiVersion: migrations.kubevirt.io/v1alpha1
kind: MigrationPolicy
metadata:
  name: my-awesome-policy
spec:
  # Migration Configuration
  allowAutoConverge: true
  bandwidthPerMigration: 217Ki
  completionTimeoutPerGiB: 23
  allowPostCopy: false
  
  # Matching to VMs
  selectors:
    namespaceSelector:
      hpc-workloads: true
    virtualMachineInstanceSelector:
      workload-type: db
```
## 策略的优先级 {#policies-precedence}

多个策略可能适用于同一个 VMI。在这种情况下，优先级顺序与上面的条目相同（先是 VMI 标签，然后是命名空间标签）。不允许定义两个具有完全相同选择器的策略。

如果多个策略适用于同一个 VMI：
* 将应用最详细的策略，也就是匹配标签数量最多的策略

* 如果多个策略以相同数量的匹配标签匹配到某个 VMI，这些策略将按匹配标签键的字典序排序。将应用此顺序中的第一个。

### 示例 {#example}

例如，设想一个具有以下标签的 VMI：

* size: small

* os: fedora

* gpu: nvidia

并假设该 VMI 所属的命名空间包含以下标签：

* priority: high

* bandwidth: medium

* hpc-workload: true

以下策略按优先级从高到低列出：

1) VMI 标签：`{size: small, gpu: nvidia}`，命名空间标签：`{priority:high, bandwidth: medium}`
   
  * 匹配标签数：4，字典序中的第一个键：`bandwidth`。

2) VMI 标签：`{size: small, gpu: nvidia}`，命名空间标签：`{priority:high, hpc-workload:true}`

  * 匹配标签数：4，字典序中的第一个键：`gpu`。

3) VMI 标签：`{size: small, gpu: nvidia}`，命名空间标签：`{priority:high}`

  * 匹配标签数：3，字典序中的第一个键：`gpu`。

4) VMI 标签：`{size: small}`，命名空间标签：`{priority:high, hpc-workload:true}`

  * 匹配标签数：3，字典序中的第一个键：`hpc-workload`。

5) VMI 标签：`{gpu: nvidia}`，命名空间标签：`{priority:high}`

  * 匹配标签数：2，字典序中的第一个键：`gpu`。

6) VMI 标签：`{gpu: nvidia}`，命名空间标签：`{}`

  * 匹配标签数：1，字典序中的第一个键：`gpu`。

7) VMI 标签：`{gpu: intel}`，命名空间标签：`{priority:high}`

  * VMI 标签不匹配 - 无法应用该策略。
