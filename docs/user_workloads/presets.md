# 预设 {#presets}

**功能状态：**

* `VirtualMachineInstancePresets` 自 [`v0.57.0`](https://github.com/kubevirt/kubevirt/releases/tag/v0.57.0) 发行版起已被弃用，并将在未来的发行版中移除。
* 用户应转而考虑使用[实例类型和偏好](../user_workloads/instancetypes.md)作为替代。

`VirtualMachineInstancePresets` 是对一般 `VirtualMachineInstance` 配置的扩展，其行为很像 Kubernetes 中的 `PodPresets`。当创建 `VirtualMachineInstance` 时，任何适用的 `VirtualMachineInstancePresets` 都会应用到该 `VirtualMachineInstance` 的现有 spec 上。这样就可以复用应当应用于多个 `VirtualMachineInstances` 的通用设置。

## 创建 VirtualMachineInstancePreset {#create-a-virtualmachineinstancepreset}

你可以在 YAML 文件中描述 `VirtualMachineInstancePreset`。例如，下面的 `vmi-preset.yaml` 文件描述了一个 `VirtualMachineInstancePreset`，它请求在创建 `VirtualMachineInstance` 时带有 64M RAM 的资源请求。

```yaml
apiVersion: kubevirt.io/v1
kind: VirtualMachineInstancePreset
metadata:
  name: small-qemu
spec:
  selector:
    matchLabels:
      kubevirt.io/size: small
  domain:
    resources:
      requests:
        memory: 64M
```

-   基于该 YAML 文件创建 `VirtualMachineInstancePreset`：

```console
    kubectl create -f vmipreset.yaml
```

### 必填字段 {#required-fields}

与大多数 Kubernetes 资源一样，`VirtualMachineInstancePreset` 需要 `apiVersion`、`kind` 和 `metadata` 字段。

此外，`VirtualMachineInstancePresets` 还需要一个 `spec` 部分。虽然从语法上讲并非严格必需，但强烈建议包含 `Selector`，并把它放在 `spec` 部分中，否则 `VirtualMachineInstancePreset` 会匹配某个命名空间中的所有 `VirtualMachineInstances`。

### VirtualMachine 选择器 {#virtualmachine-selector}

KubeVirt 使用 Kubernetes 的 `Labels` 和 `Selectors` 来确定哪些 `VirtualMachineInstancePresets` 适用于给定的 `VirtualMachineInstance`，这与 Kubernetes 中 `PodPresets` 的工作方式类似。如果来自 `VirtualMachineInstancePreset` 的某项设置被应用到 `VirtualMachineInstance`，该 `VirtualMachineInstance` 会在完成时被标上一条注解。

可以在 `spec`（属于某个 `VirtualMachineInstancePreset`）中列出任何 domain 结构，例如 Clock、Features、Memory、CPU，或者网络接口等 Devices。该 `spec` 部分（属于 `VirtualMachineInstancePreset`）中的所有元素都会应用到 `VirtualMachineInstance`。

一旦 `VirtualMachineInstancePreset` 成功应用到 `VirtualMachineInstance`，该 `VirtualMachineInstance` 会被标上一条注解，以表明它已被应用。如果在应用 `VirtualMachineInstancePreset` 的过程中发生冲突，该 `VirtualMachineInstancePreset` 的相应部分会被跳过。

可以匹配任何有效的 `Label`，但建议的经验法则是使用 os/shortname，例如 `kubevirt.io/os: rhel7`。

## 更新 VirtualMachineInstancePreset {#updating-a-virtualmachineinstancepreset}

如果修改了 `VirtualMachineInstancePreset`，变更将*不会*应用到现有的 `VirtualMachineInstances`。这既适用于 `Selector`（它表明应当匹配哪些 `VirtualMachineInstances`），也适用于 `Domain` 部分（它列出应当应用到 `VirtualMachine` 的设置）。

## 覆盖 {#overrides}

`VirtualMachineInstancePresets` 使用与 Kubernetes `PodPresets` 类似的冲突解决策略。如果 domain spec 的某一部分同时存在于 `VirtualMachineInstance` 和 `VirtualMachineInstancePreset` 中，并且两个资源具有完全相同的信息，那么 `VirtualMachineInstance` 的创建会正常继续。但如果这些资源之间存在差异，则会创建一个事件，指出哪个 `DomainSpec` 元素、属于哪个 `VirtualMachineInstancePreset` 被覆盖。例如：如果 `VirtualMachineInstance` 和 `VirtualMachineInstancePreset` 都定义了 `CPU`，但使用了不同数量的 `Cores`，KubeVirt 会记下这一差异。

如果 `VirtualMachineInstancePreset` 中的任何设置被成功应用，该 `VirtualMachineInstance` 会被加上注解。

如果 `Domains` 在 `VirtualMachineInstance` 与 `VirtualMachineInstancePreset` 之间存在差异，KubeVirt 会创建一个 `Event`。可以使用 `kubectl get events` 来显示所有 `Events`。例如：

```console
    $ kubectl get events
    ....
    Events:
      FirstSeen                         LastSeen                        Count From                              SubobjectPath                Reason    Message
      2m          2m           1         myvmi.1515bbb8d397f258                       VirtualMachineInstance                                     Warning   Conflict                  virtualmachineinstance-preset-controller   Unable to apply VirtualMachineInstancePreset 'example-preset': spec.cpu: &{6} != &{4}
```

## 用法 {#usage}

`VirtualMachineInstancePresets` 是命名空间作用域的资源，因此应当创建在将要使用它们的 `VirtualMachineInstances` 所在的同一命名空间中：

`kubectl create -f <preset>.yaml [--namespace <namespace>]`

KubeVirt 会确定哪些 `VirtualMachineInstancePresets` 适用于某个特定的 `VirtualMachineInstance`，方式是匹配 `Labels`。例如：

```yaml
apiVersion: kubevirt.io/v1
kind: VirtualMachineInstancePreset
metadata:
  name: example-preset
  selector:
    matchLabels:
      kubevirt.io/os: win10
  ...
```

会匹配同一命名空间中的任何 `VirtualMachineInstance`，其 `Label` 为 `flavor: foo`。例如：

```yaml
apiVersion: kubevirt.io/v1
kind: VirtualMachineInstance
version: v1
metadata:
  name: myvmi
  labels:
    kubevirt.io/os: win10
  ...
```

## 冲突 {#conflicts}

当多个 `VirtualMachineInstancePresets` 匹配某个特定的 `VirtualMachineInstance` 时，如果它们在 Domain 内指定了相同的设置，这些设置必须一致。如果两个 `VirtualMachineInstancePresets` 的设置发生冲突（例如所请求的 CPU 核数），就会发生错误，`VirtualMachineInstance` 会进入 `Failed` 状态，并发出一条 `Warning` 事件，说明哪些 `VirtualMachineInstancePresets` 的哪些设置存在问题。

## 匹配多个 `VirtualMachineInstances` {#matching-multiple-virtualmachineinstances}

`VirtualMachineInstancePresets` 的主要用例是创建可以跨各种机器应用的可复用设置。有多种方法可以使用选择器来匹配 `VirtualMachineInstance` 的标签。

-   matchLabels：每个 `VirtualMachineInstance` 都可以使用一个由所有实例共享的特定标签。\* matchExpressions：可以使用集合的逻辑运算符来匹配多个标签。

使用 matchLabels 时，`VirtualMachineInstancePreset` 中所使用的标签必须匹配 `VirtualMachineInstance` 的标签之一：

```yaml
selector:
  matchLabels:
    kubevirt.io/memory: large
```

会匹配

```yaml
metadata:
  labels:
    kubevirt.io/memory: large
    kubevirt.io/os: win10
```

或

```yaml
metadata:
  labels:
    kubevirt.io/memory: large
    kubevirt.io/os: fedora27
```

使用 matchExpressions 可以匹配 `VirtualMachineInstances` 的多个标签，而无需显式列出某个标签。

```yaml
selector:
  matchExpressions:
    - {key: kubevirt.io/os, operator: In, values: [fedora27, fedora26]}
```

会同时匹配：
```yaml
metadata:
  labels:
    kubevirt.io/os: fedora26

metadata:
  labels:
    kubevirt.io/os: fedora27
```

Kubernetes [文档](https://kubernetes.io/docs/concepts/overview/working-with-objects/labels/) 有详细说明。下面提供了示例。

## 排除 {#exclusions}

由于 `VirtualMachineInstancePresets` 使用 `Selectors` 来表明它们的设置应当应用到哪些 `VirtualMachineInstances`，因此需要存在一种机制，使 `VirtualMachineInstances` 能够完全选择退出 `VirtualMachineInstancePresets`。这通过注解来完成：

```yaml
kind: VirtualMachineInstance
version: v1
metadata:
  name: myvmi
  annotations:
    virtualmachineinstancepresets.admission.kubevirt.io/exclude: "true"
  ...
```

## 示例 {#examples}

### 简单的 `VirtualMachineInstancePreset` 示例 {#simple-virtualmachineinstancepreset-example}

```yaml
apiVersion: kubevirt.io/v1
kind: VirtualMachineInstancePreset
version: v1
metadata:
  name: example-preset
spec:
  selector:
    matchLabels:
      kubevirt.io/os: win10
  domain:
    features:
      acpi: {}
      apic: {}
      hyperv:
        relaxed: {}
        vapic: {}
        spinlocks:
          spinlocks: 8191
---
apiVersion: kubevirt.io/v1
kind: VirtualMachineInstance
version: v1
metadata:
  name: myvmi
  labels:
    kubevirt.io/os: win10
spec:
  domain:
    firmware:
      uuid: c8f99fc8-20f5-46c4-85e5-2b841c547cef
```

一旦 `VirtualMachineInstancePreset` 应用到 `VirtualMachineInstance`，得到的资源会如下所示：

```yaml
apiVersion: kubevirt.io/v1
kind: VirtualMachineInstance
metadata:
  annotations:
    presets.virtualmachineinstances.kubevirt.io/presets-applied: kubevirt.io/v1
    virtualmachineinstancepreset.kubevirt.io/example-preset: kubevirt.io/v1
  labels:
    kubevirt.io/os: win10
    kubevirt.io/nodeName: master
  name: myvmi
  namespace: default
spec:
  domain:
    devices: {}
    features:
      acpi:
        enabled: true
      apic:
        enabled: true
      hyperv:
        relaxed:
          enabled: true
        spinlocks:
          enabled: true
          spinlocks: 8191
        vapic:
          enabled: true
    firmware:
      uuid: c8f99fc8-20f5-46c4-85e5-2b841c547cef
    machine:
      type: q35
    resources:
      requests:
        memory: 8Mi
```

### 冲突示例 {#conflict-example}

这是一个合并冲突的示例。在此情况下，`VirtualMachineInstance` 和 `VirtualMachineInstancePreset` 请求了不同数量的 CPU。

```yaml
apiVersion: kubevirt.io/v1
kind: VirtualMachineInstancePreset
version: v1
metadata:
  name: example-preset
spec:
  selector:
    matchLabels:
      kubevirt.io/flavor: default-features
  domain:
    cpu:
      cores: 4
---
apiVersion: kubevirt.io/v1
kind: VirtualMachineInstance
version: v1
metadata:
  name: myvmi
  labels:
    kubevirt.io/flavor: default-features
spec:
  domain:
    cpu:
      cores: 6
```

在此情况下，`VirtualMachineInstance` 的 Spec 将保持不被修改。使用 `kubectl get events` 来显示事件。

```yaml
apiVersion: kubevirt.io/v1
kind: VirtualMachineInstance
metadata:
  annotations:
    presets.virtualmachineinstances.kubevirt.io/presets-applied: kubevirt.io/v1
  generation: 0
  labels:
    kubevirt.io/flavor: default-features
  name: myvmi
  namespace: default
spec:
  domain:
    cpu:
      cores: 6
    devices: {}
    machine:
      type: ""
    resources: {}
status: {}
```

调用 `kubectl get events` 会有类似下面这样的一行：

```console
2m 2m 1 myvmi.1515bbb8d397f258 VirtualMachineInstance Warning Conflict virtualmachineinstance-preset-controller Unable to apply VirtualMachineInstancePreset example-preset: spec.cpu: &{6} != &{4}
```

### 使用 MatchLabels 匹配多个 VirtualMachineInstance {#matching-multiple-virtualmachineinstances-using-matchlabels}

这些 `VirtualMachineInstances` 带有多个标签，其中一个是独有的，另一个是共享的。

注意：此示例为了演示目的，打破了使用 os-shortname 作为 `Label` 的惯例。

```yaml
apiVersion: kubevirt.io/v1
kind: VirtualMachineInstancePreset
metadata:
  name: twelve-cores
spec:
  selector:
    matchLabels:
      kubevirt.io/cpu: dodecacore
  domain:
    cpu:
      cores: 12
---
apiVersion: kubevirt.io/v1
kind: VirtualMachineInstance
metadata:
  name: windows-10
  labels:
    kubevirt.io/os: win10
    kubevirt.io/cpu: dodecacore
spec:
---
apiVersion: kubevirt.io/v1
kind: VirtualMachineInstance
metadata:
  name: windows-7
  labels:
    kubevirt.io/os: win7
    kubevirt.io/cpu: dodecacore
spec:
  terminationGracePeriodSeconds: 0
```

添加这个 `VirtualMachineInstancePreset` 以及这些 `VirtualMachineInstances` 会得到：

```yaml
apiVersion: kubevirt.io/v1
kind: VirtualMachineInstance
metadata:
  annotations:
    presets.virtualmachineinstances.kubevirt.io/presets-applied: kubevirt.io/v1
    virtualmachineinstancepreset.kubevirt.io/twelve-cores: kubevirt.io/v1
  labels:
    kubevirt.io/cpu: dodecacore
    kubevirt.io/os: win10
  name: windows-10
spec:
  domain:
    cpu:
      cores: 12
    devices: {}
    resources:
      requests:
        memory: 4Gi
---
apiVersion: kubevirt.io/v1
kind: VirtualMachineInstance
metadata:
  annotations:
    presets.virtualmachineinstances.kubevirt.io/presets-applied: kubevirt.io/v1
    virtualmachineinstancepreset.kubevirt.io/twelve-cores: kubevirt.io/v1
  labels:
    kubevirt.io/cpu: dodecacore
    kubevirt.io/os: win7
  name: windows-7
spec:
  domain:
    cpu:
      cores: 12
    devices: {}
    resources:
      requests:
        memory: 4Gi
  terminationGracePeriodSeconds: 0
```

### 使用 MatchExpressions 匹配多个 VirtualMachineInstance {#matching-multiple-virtualmachineinstances-using-matchexpressions}

这个 `VirtualMachineInstancePreset` 带有一个 matchExpression，它会匹配两个标签：`kubevirt.io/os: win10` 和 `kubevirt.io/os: win7`。

```yaml
apiVersion: kubevirt.io/v1
kind: VirtualMachineInstancePreset
metadata:
  name: windows-vmis
spec:
  selector:
    matchExpressions:
      - {key: kubevirt.io/os, operator: In, values: [win10, win7]}
  domain:
    resources:
      requests:
        memory: 128M
---
apiVersion: kubevirt.io/v1
kind: VirtualMachineInstance
metadata:
  name: smallvmi
  labels:
    kubevirt.io/os: win10
spec:
  terminationGracePeriodSeconds: 60
---
apiVersion: kubevirt.io/v1
kind: VirtualMachineInstance
metadata:
  name: largevmi
  labels:
    kubevirt.io/os: win7
spec:
  terminationGracePeriodSeconds: 120
```

把该预设应用到两台 VM 会得到：

```yaml
apiVersion: v1
items:
- apiVersion: kubevirt.io/v1
  kind: VirtualMachineInstance
  metadata:
    annotations:
      presets.virtualmachineinstances.kubevirt.io/presets-applied: kubevirt.io/v1
      virtualmachineinstancepreset.kubevirt.io/windows-vmis: kubevirt.io/v1
    labels:
      kubevirt.io/os: win7
    name: largevmi
  spec:
    domain:
      resources:
        requests:
          memory: 128M
    terminationGracePeriodSeconds: 120
- apiVersion: kubevirt.io/v1
  kind: VirtualMachineInstance
  metadata:
    annotations:
      presets.virtualmachineinstances.kubevirt.io/presets-applied: kubevirt.io/v1
      virtualmachineinstancepreset.kubevirt.io/windows-vmis: kubevirt.io/v1
    labels:
      kubevirt.io/os: win10
    name: smallvmi
  spec:
    domain:
      resources:
        requests:
          memory: 128M
    terminationGracePeriodSeconds: 60
```
