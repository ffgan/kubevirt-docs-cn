# 客户端透传 {#client-passthrough}

KubeVirt 已支持借助 virtctl 命令，将客户端机器上的设备重定向到 VMI。

## USB 重定向 {#usb-redirection}

对客户端 USB 设备的重定向支持在 v0.44 版本中引入。此功能默认未启用。要启用它，请在 devices 下添加一个空的 `clientPassthrough`，如下所示：

```yaml
spec:
  domain:
    devices:
      clientPassthrough: {}
```

此配置目前会为 VMI 添加 4 个只能配合 virtctl 使用的 USB 插槽。

重定向同一 USB 设备有两种方式：使用该设备的厂商和产品信息，或者使用实际的总线和设备地址信息。在 Linux 中，可以用 `lsusb` 收集这些信息，下面是经过删减的示例：

```shell
> lsusb
Bus 002 Device 008: ID 0951:1666 Kingston Technology DataTraveler 100 G3/G4/SE9 G2/50
Bus 001 Device 003: ID 13d3:5406 IMC Networks Integrated Camera
Bus 001 Device 010: ID 0781:55ae SanDisk Corp. Extreme 55AE
```

### 使用厂商和产品 {#using-vendor-and-product}

重定向 Kingston 存储设备。
```
virtctl usbredir 0951:1666 vmi-name
```


### 使用总线和设备地址 {#using-bus-and-device-address}

重定向集成摄像头。
```
virtctl usbredir 01-03 vmi-name
```

### `virtctl usbredir` 的要求 {#requirements-for-virtctl-usbredir}

`virtctl` 命令使用名为 `usbredirect` 的应用程序来处理客户端的 USB 设备：将该设备从客户端操作系统中拔出，并在该设备与 VMI 之间建立通信通道。

#### usbredirect {#usbredirect}

`usbredirect` 二进制文件来自 [usbredir][] 项目，大多数 Linux 发行版都提供支持。你可以获取 [最新版本][]，或者获取用于 Windows 支持的 [MSI 安装程序][]。

[usbredir]: https://gitlab.freedesktop.org/spice/usbredir/
[最新版本]: https://www.spice-space.org/download/usbredir/
[MSI 安装程序]: https://www.spice-space.org/download/windows/usbredirect/

#### 权限 {#permissions}

在大多数操作系统中，管理 USB 设备需要特权访问。运行 `virtctl usbredir` 的用户需要具备特权，或者以特权方式运行它（例如：使用 `sudo`）。

#### Windows 支持 {#windows-support}

- 在 Windows 上重定向 USB 设备需要安装 [UsbDk][]。
- 请确保 `usbredirect` 已包含在 PATH 环境变量中。

[UsbDk]: https://github.com/daynix/UsbDk
