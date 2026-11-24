# 运行旧版 Windows {#running-legacy-windows-versions}

Windows XP 或 Windows Server 2003 等旧版 Windows 无法在 KubeVirt 上开箱即用地启动。这是因为 Q35 机器类型与向客户机报告的 64 位 PCI 空洞组合在一起，而这些操作系统并不支持这种组合。

要在 KubeVirt 上运行旧版 Windows，需要禁用对 64 位 PCI 空洞的报告。这可以通过添加 `kubevirt.io/disablePCIHole64` 注解（值为 `true`）到 `VirtualMachineInstance` 的注解中来实现。

## 示例 {#example}

使用此 `VirtualMachine` 定义，旧版 Windows 客户机即可启动：

```yaml
apiVersion: kubevirt.io/v1
kind: VirtualMachine
metadata:
  name: xp
spec:
  runStrategy: Always
  template:
    metadata:
      annotations:
        kubevirt.io/disablePCIHole64: "true"
    spec:
      domain:
        devices: {}
        memory:
          guest: 512Mi
      terminationGracePeriodSeconds: 180
      volumes:
      - containerDisk:
          image: my/windowsxp:containerdisk
        name: xp-containerdisk-0
```
