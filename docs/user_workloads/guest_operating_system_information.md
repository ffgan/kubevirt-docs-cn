# 客户机操作系统信息 {#guest-operating-system-information}

VirtualMachineInstance 的客户机操作系统标识将由标签 `kubevirt.io/os` 提供：

```yaml
metadata:
  name: myvmi
  labels:
    kubevirt.io/os: win2k12r2
```

`kubevirt.io/os` 标签基于 [libosinfo](https://libosinfo.org/) 数据库中的短 OS 标识符。目前支持以下短 ID：

<table>
<colgroup>
<col style="width: 20%" />
<col style="width: 20%" />
<col style="width: 20%" />
<col style="width: 20%" />
<col style="width: 20%" />
</colgroup>
<thead>
<tr class="header">
<th>短 ID</th>
<th>名称</th>
<th>版本</th>
<th>系列</th>
<th>ID</th>
</tr>
</thead>
<tbody>
<tr class="odd">
<td><p><strong>win2k12r2</strong></p></td>
<td><p>Microsoft Windows Server 2012 R2</p></td>
<td><p>6.3</p></td>
<td><p>winnt</p></td>
<td><p><a href="https://www.microsoft.com/en-us/evalcenter/evaluate-windows-server-2012-r2">https://www.microsoft.com/en-us/evalcenter/evaluate-windows-server-2012-r2</a></p></td>
</tr>
</tbody>
</table>

## 与预设一起使用 {#use-with-presets}

带有 `kubevirt.io/os` 标签、代表某一操作系统的 VirtualMachineInstancePreset，可以应用到任何同样带有并匹配该 `kubevirt.io/os` 标签的 VirtualMachineInstance 上。

当前发行版包含上述 OS 标识符的默认预设。

### Windows Server 2012R2 `VirtualMachineInstancePreset` 示例 {#windows-server-2012r2-virtualmachineinstancepreset-example}

```yaml
apiVersion: kubevirt.io/v1
kind: VirtualMachineInstancePreset
metadata:
  name: windows-server-2012r2
  selector:
    matchLabels:
      kubevirt.io/os: win2k12r2
spec:
  domain:
    cpu:
      cores: 2
    resources:
      requests:
        memory: 2G
    features:
      acpi: {}
      apic: {}
      hyperv:
        relaxed: {}
        vapic: {}
        spinlocks:
          spinlocks: 8191
    clock:
      utc: {}
      timer:
        hpet:
          present: false
        pit:
          tickPolicy: delay
        rtc:
          tickPolicy: catchup
        hyperv: {}
---
apiVersion: kubevirt.io/v1
kind: VirtualMachineInstance
metadata:
  labels:
    kubevirt.io/os: win2k12r2
  name: windows2012r2
spec:
  terminationGracePeriodSeconds: 0
  domain:
    firmware:
      uuid: 5d307ca9-b3ef-428c-8861-06e72d69f223
    devices:
      disks:
      - name: server2012r2
        disk:
          dev: vda
  volumes:
    - name: server2012r2
      persistentVolumeClaim:
        claimName: my-windows-image
```

将 `VirtualMachineInstancePreset` 应用到 `VirtualMachineInstance` 之后，得到的资源如下所示：

```yaml
apiVersion: kubevirt.io/v1
kind: VirtualMachineInstance
metadata:
  annotations:
    presets.virtualmachineinstances.kubevirt.io/presets-applied: kubevirt.io/v1
    virtualmachineinstancepreset.kubevirt.io/windows-server-2012r2: kubevirt.io/v1
  labels:
    kubevirt.io/os: win2k12r2
  name: windows2012r2
spec:
  terminationGracePeriodSeconds: 0
  domain:
    cpu:
      cores: 2
    resources:
      requests:
        memory: 2G
    features:
      acpi: {}
      apic: {}
      hyperv:
        relaxed: {}
        vapic: {}
        spinlocks:
          spinlocks: 8191
    clock:
      utc: {}
      timer:
        hpet:
          present: false
        pit:
          tickPolicy: delay
        rtc:
          tickPolicy: catchup
        hyperv: {}
    firmware:
      uuid: 5d307ca9-b3ef-428c-8861-06e72d69f223
    devices:
      disks:
      - name: server2012r2
        disk:
          dev: vda
  volumes:
    - name: server2012r2
      persistentVolumeClaim:
        claimName: my-windows-image
```

更多信息见 [VirtualMachineInstancePresets](../user_workloads/presets.md)

### HyperV 优化 {#hyperv-optimizations}

KubeVirt 支持相当多所谓的 “HyperV enlightenment”，它们是针对 Windows 客户机的优化。其中一些优化可能需要较新的宿主机内核支持才能正常工作，或才能带来最大的性能提升。

KubeVirt 可以在运行启用了 Hyper-V 的 VM 之前对宿主机执行额外检查，以确保宿主机在 Hyper-V 支持方面没有已知问题，正确暴露所有必需的功能，从而可以预期获得最优性能。出于向后兼容，并且因为这些检查依赖于 [node-feature-discovery](https://github.com/kubernetes-sigs/node-feature-discovery) 以及额外配置，这些检查默认禁用。

要启用严格的宿主机检查，用户可以扩展 KubeVirt CR 中的 `featureGates` 字段，向其中加入 `HypervStrictCheck`。

```yaml
apiVersion: kubevirt.io/v1
kind: Kubevirt
metadata:
  name: kubevirt
  namespace: kubevirt
spec:
  ...
  configuration:
    developerConfiguration:
      featureGates:
        - "HypervStrictCheck"
```

另外，用户可以编辑现有的 kubevirt CR：

```bash
kubectl edit kubevirt kubevirt -n kubevirt
```

```yaml
...
spec:
  configuration:
    developerConfiguration:
      featureGates:
        - "HypervStrictCheck"
        - "CPUManager"
```
