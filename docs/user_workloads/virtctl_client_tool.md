# 下载并安装 virtctl 命令行界面 {#download-and-install-the-virtctl-command-line-interface}

## 下载 `virtctl` 客户端工具 {#download-the-virtctl-client-tool}

基本的 VirtualMachineInstance 操作可以用标准的 `kubectl` 工具完成。不过，要使用以下高级功能，则需要 `virtctl` 二进制工具：

-   串行控制台和图形控制台访问

它还为以下操作提供便捷命令：

-   启动和停止 VirtualMachineInstance

-   对 VirtualMachineInstance 进行实时迁移以及取消实时迁移

-   上传虚拟机磁盘镜像

获取它有两种方式：

-   可以从[官方发布页面](https://github.com/kubevirt/kubevirt/releases)获取该工具的最新版本

-   可以使用 [krew](https://krew.dev/) 将其安装为 `kubectl` 插件

示例：

```
export VERSION=$(curl https://storage.googleapis.com/kubevirt-prow/release/kubevirt/kubevirt/stable.txt)
wget https://github.com/kubevirt/kubevirt/releases/download/${VERSION}/virtctl-${VERSION}-linux-amd64
```

## 安装 `virtctl`（使用 `krew`） {#install-virtctl-with-krew}

需要事先[安装 `krew` 插件管理器](https://github.com/kubernetes-sigs/krew/#installation)。如果已安装 `krew`，`virtctl` 可以通过 `krew` 安装：

    $ kubectl krew install virt

之后可以将 `virtctl` 用作 kubectl 插件。要列出可用命令，请运行：

    $ kubectl virt help

本指南中每一处出现的

    $ ./virtctl <command>...

都应理解为

    $ kubectl virt <command>...
