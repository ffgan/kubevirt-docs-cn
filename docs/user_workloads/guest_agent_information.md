# 客户机代理信息 {#guest-agent-information}

客户机代理（GA）是可以在虚拟机内部运行的可选组件。
GA 提供大量关于正在运行的操作系统（OS）的额外运行时信息。
关于可用 GA 命令的更多技术细节见[此处](https://qemu.weilnetz.de/doc/3.1/qemu-ga-ref.html)。


## 虚拟机状态中的客户机代理信息 {#guest-agent-info-in-virtual-machine-status}

虚拟机中是否存在 GA，由 VirtualMachineInstance 状态中的一个状况来表明。
该状况说明 GA 已连接并可以使用。

VirtualMachineInstance 上的 GA 状况

```yaml
status:
  conditions:
  - lastProbeTime: "2020-02-28T10:22:59Z"
    lastTransitionTime: null
    status: "True"
    type: AgentConnected
```

当 GA 已连接时，状态中会显示额外的 OS 信息。
这些信息包括：

   - 客户机信息，其中包含 OS 运行时数据
   - 接口信息，其中显示与 GA 接口信息合并后的 QEMU 接口。

下面是 VirtualMachineInstance 状态中所显示信息的示例。

合并进状态的 GA 信息

```yaml
status:
  guestOSInfo:
    id: fedora
    kernelRelease: 4.18.16-300.fc29.x86_64
    kernelVersion: '#1 SMP Sat Oct 20 23:24:08 UTC 2018'
    name: Fedora
    prettyName: Fedora 29 (Cloud Edition)
    version: "29"
    versionId: "29"
  interfaces:
  - infoSource: domain, guest-agent
    interfaceName: eth0
    ipAddress: 10.244.0.23/24
    ipAddresses:
    - 10.244.0.23/24
    - fe80::858:aff:fef4:17/64
    mac: 0a:58:0a:f4:00:17
    name: default
```

当虚拟机中不存在客户机代理时，不会显示客户机代理信息。由于客户机代理是可选组件，因此不会报告错误。

infoSource 字段表明信息从何处收集。有效值：

   - domain：信息基于 domain spec
   - guest-agent：信息基于客户机代理报告
   - domain, guest-agent：信息同时基于 domain spec 和客户机代理报告

## 通过 API 获取的客户机代理信息 {#guest-agent-info-available-through-the-api}

VirtualMachineInstance 状态中显示的数据只是可用信息的子集。
其余数据可通过 Kubernetes `kube-api` 服务器暴露的 REST API 获取。

VirtualMachineInstance 对象新增了三个子资源：

    - guestosinfo
    - userlist
    - filesystemlist


完整的 GA 数据通过 API 端点背后的 `guestosinfo` 子资源返回。

    /apis/subresources.kubevirt.io/v1/namespaces/{namespace}/virtualmachineinstances/{name}/guestosinfo


GuestOSInfo 示例数据：

```json
{
    "fsInfo": {
        "disks": [
            {
                "diskName": "vda1",
                "fileSystemType": "ext4",
                "mountPoint": "/",
                "totalBytes": 0,
                "usedBytes": 0
            }
        ]
    },
    "guestAgentVersion": "2.11.2",
    "hostname": "testvmi6m5krnhdlggc9mxfsrnhlxqckgv5kqrwcwpgr5mdpv76grrk",
    "metadata": {
        "creationTimestamp": null
    },
    "os": {
        "id": "fedora",
        "kernelRelease": "4.18.16-300.fc29.x86_64",
        "kernelVersion": "#1 SMP Sat Oct 20 23:24:08 UTC 2018",
        "machine": "x86_64",
        "name": "Fedora",
        "prettyName": "Fedora 29 (Cloud Edition)",
        "version": "29 (Cloud Edition)",
        "versionId": "29"
    },
    "timezone": "UTC, 0"
}
```

FSInfo 和 UserList 项的最大容量被限制为 10 项，以防虚拟机拥有成千上万的用户。

文件系统的完整列表可通过作为端点提供的子资源 `filesystemlist` 获取。

    /apis/subresources.kubevirt.io/v1/namespaces/{namespace}/virtualmachineinstances/{name}/filesystemlist

文件系统示例数据：

```json
{
    "items": [
        {
            "diskName": "vda1",
            "fileSystemType": "ext4",
            "mountPoint": "/",
            "totalBytes": 3927900160,
            "usedBytes": 1029201920
        }
    ],
    "metadata": {}
}
```

用户的完整列表可通过作为端点提供的子资源 `userlist` 获取。

    /apis/subresources.kubevirt.io/v1/namespaces/{namespace}/virtualmachineinstances/{name}/userlist

Userlist 示例数据：

```json
{
    "items": [
        {
            "loginTime": 1580467675.876078,
            "userName": "fedora"
        }
    ],
    "metadata": {}
}
```

用户的 LoginTime 是自纪元时间起的小数秒。转换到所需格式的工作留给使用方完成。
