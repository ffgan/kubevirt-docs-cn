# Hook Sidecar 容器 {#hook-sidecar-container}

!!! warning "自 v1.9 起弃用"
    Hook Sidecar 机制自 v1.9 起已弃用。移除版本尚未确定。请改用新的[插件](../cluster_admin/plugins.md)机制，它提供结构化的 domain hook、节点 hook 和准入引用。

## 简介 {#introduction}

在 KubeVirt 中，Hook Sidecar 容器是一个 sidecar 容器（与主应用容器在同一个 Pod 中一起运行的辅助容器），用于在虚拟机初始化之前应用定制。之所以提供此能力，是因为 VMI 规范中的可配置元素并未覆盖全部 [libvirt domain XML](https://libvirt.org/formatdomain.html) 元素。

sidecar 容器通过套接字以 gRPC 协议与主容器通信。主要有两种 sidecar hook：

1. `onDefineDomain`：此 hook 用于定制 libvirt 的 XML，并通过 gRPC 返回新的 XML，以供创建 VM。
2. `preCloudInitIso`：此 hook 用于定制 cloud-init 配置。它处理并返回 JSON 格式的 cloud-init 数据。

## 启用 `Sidecar` 功能门控 {#enabling-sidecar-feature-gate}

可以按照[激活功能门控](../cluster_admin/activating_feature_gates.md)中提到的步骤启用 `Sidecar` 功能门控。

如果是使用 kubevirtci 创建的开发集群，请按照[开发者文档](https://github.com/kubevirt/kubevirt/blob/main/docs/getting-started.md#compile-and-run-it)中提到的步骤启用该功能门控。

## Sidecar-shim 容器镜像 {#sidecar-shim-container-image}

要运行带有自定义修改的 VM，[sidecar-shim-image](https://quay.io/repository/kubevirt/sidecar-shim) 负责实现与主容器的通信。

该镜像包含 `sidecar-shim` 二进制文件，它由 [`sidecar_shim.go`](https://github.com/kubevirt/kubevirt/blob/main/cmd/sidecars/sidecar_shim.go) 构建，并应保持为容器的入口点。该二进制文件会在 `$PATH` 中搜索以 hook 名称命名的二进制文件（例如 `onDefineDomain` 和 `preCloudInitIso`）并运行它们。用户必须以命令行选项（标志）的形式提供必要的参数。

对于 `onDefineDomain`，参数是 JSON 字符串形式的 VMI 信息（例如 `--vmi vmiJSON`）以及当前的 domain XML（例如 `--domain domainXML`）。它把修改后的 domain XML 输出到标准输出。

对于 `preCloudInitIso`，参数是 JSON 字符串形式的 VMI 信息（例如 `--vmi vmiJSON`）以及 CloudInitData（例如 `--cloud-init cloudInitJSON`）。它把修改后的 CloudInitData（JSON 形式）输出到标准输出。

可以把 Shell 或 python 脚本放到预期位置（根据 hook 的不同，为 `/usr/bin/onDefineDomain` 或 `/usr/bin/preCloudInitIso`），以此作为该二进制文件的替代。

KubeVirt 发行版中附带一个名为 `sidecar-shim` 的预构建镜像，能够运行 Shell 或 Python 脚本。

## Go、Python、Shell——任选其一 {#go-python-shell-pick-any-one}

虽然二进制文件并不严格要求由 Go 代码生成，脚本也并不严格要求必须是 Shell 或 Python 之一，但就本指南而言，我们将以它们为例。

### Go 二进制文件 {#go-binary}

修改 [SMBIOS 系统信息](https://libvirt.org/formatdomain.html#smbios-system-information) 的 Go 代码示例可以在 [KubeVirt 仓库](https://github.com/kubevirt/kubevirt/tree/main/cmd/sidecars/smbios)中找到。由此代码生成的二进制文件，若在 sidecar-shim-image 中位于 `/usr/bin/ondefinedomain`，会在创建 VMI 之前立即运行，并把基板制造商的值修改为 [VMI spec](https://github.com/kubevirt/kubevirt/blob/main/examples/vmi-with-sidecar-hook.yaml) 里 `smbios.vm.kubevirt.io/baseBoardManufacturer` 注解所提供的内容。

### Shell 或 Python 脚本 {#shell-or-python-script}

如果你更愿意编写 shell 或 python 脚本而不是 Go 程序，请创建 Kubernetes ConfigMap，并使用注解确保该脚本在创建 VMI 之前运行。流程如下：

1. 创建一个 ConfigMap，其中包含你要运行的 shell 或 python 脚本
1. 创建一个 VMI，其中包含注解 `hooks.kubevirt.io/hookSidecars`，并在其中写明 ConfigMap 信息。
1. 在这种情况下，可以使用预定义镜像来处理与主容器的通信。

#### 包含 shell 脚本的 ConfigMap {#configmap-with-shell-script}

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: my-config-map
data:
  my_script.sh: |
    #!/bin/sh
    tempFile=`mktemp --dry-run`
    echo $4 > $tempFile
    sed -i "s|<baseBoard></baseBoard>|<baseBoard><entry name='manufacturer'>Radical Edward</entry></baseBoard>|" $tempFile
    cat $tempFile
```

#### 包含 python 脚本的 ConfigMap {#configmap-with-python-script}

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: my-config-map
data:
  my_script.sh: |
    #!/usr/bin/env python3

    import xml.etree.ElementTree as ET
    import sys

    def main(s):
        # write to a temporary file
        f = open("/tmp/orig.xml", "w")
        f.write(s)
        f.close()

        # parse xml from file
        xml = ET.parse("/tmp/orig.xml")
        # get the root element
        root = xml.getroot()
        # find the baseBoard element
        baseBoard = root.find("sysinfo").find("baseBoard")

        # prepare new element to be inserted into the xml definition
        element = ET.Element("entry", {"name": "manufacturer"})
        element.text = "Radical Edward"
        # insert the element
        baseBoard.insert(0, element)

        # write to a new file
        xml.write("/tmp/new.xml")
        # print file contents to stdout
        f = open("/tmp/new.xml")
        print(f.read())
        f.close()

    if __name__ == "__main__":
        main(sys.argv[4])
```

创建上述 ConfigMap 之一后，使用[此示例](https://github.com/kubevirt/kubevirt/blob/main/examples/vmi-with-sidecar-hook-configmap.yaml)中的清单创建 VMI。这里的重点是存储在注解中的 ConfigMap 信息：

```yaml
annotations:
  hooks.kubevirt.io/hookSidecars: >
    [
        {
            "args": ["--version", "v1alpha2"],
            "configMap": {"name": "my-config-map", "key": "my_script.sh", "hookPath": "/usr/bin/onDefineDomain"}
        }
    ]
```

`name` 字段表示集群上包含你要执行的脚本的 ConfigMap 名称。`key` 字段表示 ConfigMap 中包含待执行脚本的键。最后，`hookPath` 表示你希望挂载该脚本的路径。它可以是 `/usr/bin/onDefineDomain` 或 `/usr/bin/preCloudInitIso`，取决于你要执行的 hook。
如果需要自定义镜像，可以用 `"image"` 键指定一个可选值；如果省略，将使用与其他 KubeVirt 镜像一起构建的默认 Sidecar-shim 镜像。
默认的 Sidecar-shim 镜像如果没有用自定义值覆盖，也会像其他镜像一样，按照[更新 KubeVirt 工作负载](../cluster_admin/updating_and_deletion/#updating-kubevirt-workloads)进行更新。

### 验证一切正常 {#verify-everything-works}

无论你使用的是上述示例中的 Go 二进制文件还是 Shell/Python 脚本，都应当能看到新创建的 VMI 具有修改后的基板制造商信息。创建 VMI 之后，请验证它处于 `Running` 状态，连接到它的控制台，并查看基板制造商的预期变更是否已体现：

```shell
# Once the VM is ready, connect to its display and login using name and password "fedora"
cluster/virtctl.sh vnc vmi-with-sidecar-hook-configmap

# Check whether the base board manufacturer value was successfully overwritten
sudo dmidecode -s baseboard-manufacturer
```
