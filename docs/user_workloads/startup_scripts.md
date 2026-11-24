# 启动脚本 {#startup-scripts}

KubeVirt 支持为 VirtualMachineInstance 实例分配启动脚本的能力，该脚本会在 VM 初始化时自动执行。

这些脚本通常用于自动向 VM 注入用户和 SSH 密钥，以便提供对该机器的远程访问。例如，启动脚本可用于向 VM 注入凭据，使运行在远程主机上的 Ansible 作业能够访问并置备该 VM。

不过，启动脚本并不限于任何特定用例。它们可以在 VM 启动时运行任意脚本。

## Cloud-init {#cloud-init}

cloud-init 是一个被广泛采用的项目，用于 VM 的早期初始化。AWS 和 GCP 等云提供商都在使用它，cloud-init 已成为向 VM 提供启动脚本的事实标准方法。

Cloud-init 文档可以在此处找到：[Cloud-init 文档](https://cloudinit.readthedocs.io/en/latest/)。

KubeVirt 支持 cloud-init 的 [NoCloud](https://cloudinit.readthedocs.io/en/latest/reference/datasources/nocloud.html) 和 [ConfigDrive](https://cloudinit.readthedocs.io/en/latest/reference/datasources/configdrive.html) 数据源，它们通过临时磁盘将启动脚本注入 VM 实例。安装了 cloud-init 软件包的 VM 会检测到该临时磁盘，并在启动时执行自定义的用户数据脚本。

## Ignition {#ignition}

Ignition 是 cloud-init 的替代方案，可用于在首次启动时配置 VM 磁盘。你可以在[此处](https://coreos.github.io/ignition/)找到 Ignition 文档。你也可以在[此处](https://www.flatcar.org/docs/latest/provisioning/ignition/#ignition-vs-coreos-cloudinit)找到 cloud-init 与 Ignition 的对比。

在 KubeVirt 中，可以通过使用 `cloudInitConfigDrive` 卷来使用 Ignition。

## Sysprep {#sysprep}

Sysprep 是用于 Windows 的自动化工具，可自动完成 Windows 的安装、设置以及自定义软件置备。

一般流程如下：

1. 使用 Sysprep 工具封装 VM 镜像，例如运行：

        %WINDIR%\system32\sysprep\sysprep.exe /generalize /shutdown /oobe /mode:vm

    !!! Note
        我们需要确保基础 VM 不会重新启动，这可以通过将 VM 的运行策略设置为 `RerunOnFailure` 来实现。
          
          VM runStrategy：

            spec:
              runStrategy: RerunOnFailure

    更多信息可以在此处找到：

    * [Sysprep 流程概述](https://docs.microsoft.com/en-us/windows-hardware/manufacture/desktop/sysprep-process-overview)
    * [对 Windows 安装执行 Sysprep（通用化）](https://docs.microsoft.com/en-us/windows-hardware/manufacture/desktop/sysprep--generalize--a-windows-installation)

    !!! Note
        重要的是，在触发 Sysprep 工具时不能检测到应答文件，因为 Windows 安装程序会在每个配置阶段开始时搜索应答文件并缓存它。如果发生这种情况，操作系统启动时就只会使用缓存的应答文件，而忽略我们通过 Sysprep API 提供的那个应答文件。更多信息可以在[此处](https://docs.microsoft.com/en-us/windows-hardware/manufacture/desktop/windows-setup-automation-overview#implicit-answer-file-search-order)找到。

2. 在附加介质中提供名为 `autounattend.xml` 的应答文件。应答文件可以在 ConfigMap 或 Secret 中提供，其键为 `autounattend.xml`。

    该配置文件可以使用 [Windows SIM](https://docs.microsoft.com/en-us/windows-hardware/customize/desktop/wsim/windows-system-image-manager-overview-topics) 生成，也可以根据此处找到的信息手动指定：

    * [应答文件（unattend.xml）](https://docs.microsoft.com/en-us/windows-hardware/manufacture/desktop/update-windows-settings-and-scripts-create-your-own-answer-file-sxs)
    * [应答文件参考](https://docs.microsoft.com/en-us/windows-hardware/customize/desktop/wsim/answer-files-overview)
    * [应答文件组件参考](https://docs.microsoft.com/en-us/windows-hardware/customize/desktop/unattend/components-b-unattend)

    !!! Note
        此外，网上还有许多容易找到的工具可用于创建应答文件。

## Cloud-init 示例 {#cloud-init-examples}

### 用户数据 {#user-data}

KubeVirt 支持 cloud-init 的 NoCloud 和 ConfigDrive 数据源，它们通过使用挂接到 VM 的磁盘来注入启动脚本。

要使用此方法为 VirtualMachineInstance 分配自定义用户数据脚本，用户必须在 VirtualMachineInstance 的 spec 中为 NoCloud 或 ConfigDrive 数据源定义磁盘和卷。

#### 数据源 {#data-sources}

在大多数情况下，用户应坚持使用 NoCloud 数据源，因为它是最简单的 cloud-init 数据源。仅当 cloud-init 实现不支持 NoCloud 时（例如 [coreos-cloudinit](https://github.com/coreos/coreos-cloudinit)），用户才应把数据源切换为 ConfigDrive。

把 cloud-init 数据源切换为 ConfigDrive 非常简单，只需把 VirtualMachineInstance spec 中的卷类型从 `cloudInitNoCloud` 改为 `cloudInitConfigDrive`。

NoCloud 数据源：

    volumes:
      - name: cloudinitvolume
        cloudInitNoCloud:
          userData: "#cloud-config"

ConfigDrive 数据源：

    volumes:
      - name: cloudinitvolume
        cloudInitConfigDrive:
          userData: "#cloud-config"

使用 ConfigDrive 数据源时，`networkData` 部分必须采用 [OpenStack 元数据服务网络](https://specs.openstack.org/openstack/nova-specs/specs/liberty/implemented/metadata-service-network-info.html) 格式：
```
  spec:
    domain:
      interfaces: 
        - name: secondary-net
          bridge: {}
          macAddress: '02:26:19:00:00:30'
          model: virtio
    networks: 
        - multus:
            networkName: my-ns/my-net
          name: secondary-net
    volumes:
      - name: cloudinitvolume
        cloudInitConfigDrive:
          networkData: |
            {"links":[{"id":"enp2s0","type":"phy","ethernet_mac_address":"02:26:19:00:00:30"}],"networks":[{"id":"NAD1","type":"ipv4","link":"enp2s0","ip_address":"10.184.0.244","netmask":"255.255.240.0","routes":[{"network":"0.0.0.0","netmask":"0.0.0.0","gateway":"23.253.157.1"}],"network_id":""}],"services":[]}
          userData: "#cloud-config"
```

> **注意**
> 辅助接口的 MAC 地址应当预先定义，并且在网络接口和 cloud-init 的 networkData 中保持相同。

请查看下面的示例，以获取更完整的 cloud-init 示例。

#### 以明文形式提供的 Cloud-init 用户数据 {#cloud-init-user-data-as-clear-text}

在下面的示例中，SSH 密钥以明文形式存储在 cloudInitNoCloud 卷的 userData 字段中。有一个对应的 disks 条目引用该 cloud-init 卷，并将其分配给 VM 的设备。

    # Create a VM manifest with the startup script
    # a cloudInitNoCloud volume's userData field.

    cat << END > my-vmi.yaml
    apiVersion: kubevirt.io/v1
    kind: VirtualMachineInstance
    metadata:
      name: myvmi
    spec:
      terminationGracePeriodSeconds: 5
      domain:
        resources:
          requests:
            memory: 64M
        devices:
          disks:
          - name: containerdisk
            disk:
              bus: virtio
          - name: cloudinitdisk
            disk:
              bus: virtio
      volumes:
        - name: containerdisk
          containerDisk:
            image: kubevirt/cirros-container-disk-demo:latest
        - name: cloudinitdisk
          cloudInitNoCloud:
            userData: |
              #cloud-config
              ssh_authorized_keys:
                - ssh-rsa AAAAB3NzaK8L93bWxnyp test@test.com

    END

    # Post the Virtual Machine spec to KubeVirt.

    kubectl create -f my-vmi.yaml

#### 以 base64 字符串形式提供的 Cloud-init 用户数据 {#cloud-init-user-data-as-base64-string}

在下面的示例中，一段简单的 bash 脚本经过 base64 编码，并存储在 cloudInitNoCloud 卷的 userDataBase64 字段中。有一个对应的 disks 条目引用该 cloud-init 卷，并将其分配给 VM 的设备。

*用户也可以选择把启动脚本存放在 Kubernetes Secret 中，并在 VM 的 spec 里引用该 Secret。文档后面的示例说明了具体做法。*

    # Create a simple startup script

    cat << END > startup-script.sh
    #!/bin/bash
    echo "Hi from startup script!"
    END

    # Create a VM manifest with the startup script base64 encoded into
    # a cloudInitNoCloud volume's userDataBase64 field.

    cat << END > my-vmi.yaml
    apiVersion: kubevirt.io/v1
    kind: VirtualMachineInstance
    metadata:
      name: myvmi
    spec:
      terminationGracePeriodSeconds: 5
      domain:
        resources:
          requests:
            memory: 64M
        devices:
          disks:
          - name: containerdisk
            disk:
              bus: virtio
          - name: cloudinitdisk
            disk:
              bus: virtio
      volumes:
        - name: containerdisk
          containerDisk:
            image: kubevirt/cirros-container-disk-demo:latest
        - name: cloudinitdisk
          cloudInitNoCloud:
            userDataBase64: $(cat startup-script.sh | base64 -w0)
    END

    # Post the Virtual Machine spec to KubeVirt.

    kubectl create -f my-vmi.yaml

#### 以 k8s Secret 形式提供的 Cloud-init 用户数据 {#cloud-init-userdata-as-k8s-secret}

不希望把 cloud-init 用户数据直接存放在 VirtualMachineInstance spec 中的用户，可以选择把用户数据存入 Kubernetes Secret，并在 spec 中引用该 Secret。

多个 VirtualMachineInstance spec 可以引用同一个包含 cloud-init 用户数据的 Kubernetes Secret。

下面的示例说明如何创建包含启动脚本的 Kubernetes Secret，并在 VM 的 spec 中引用该 Secret。

    # Create a simple startup script

    cat << END > startup-script.sh
    #!/bin/bash
    echo "Hi from startup script!"
    END

    # Store the startup script in a Kubernetes Secret
    kubectl create secret generic my-vmi-secret --from-file=userdata=startup-script.sh

    # Create a VM manifest and reference the Secret's name in the cloudInitNoCloud
    # Volume's secretRef field

    cat << END > my-vmi.yaml
    apiVersion: kubevirt.io/v1
    kind: VirtualMachineInstance
    metadata:
      name: myvmi
    spec:
      terminationGracePeriodSeconds: 5
      domain:
        resources:
          requests:
            memory: 64M
        devices:
          disks:
          - name: containerdisk
            disk:
              bus: virtio
          - name: cloudinitdisk
            disk:
              bus: virtio
      volumes:
        - name: containerdisk
          containerDisk:
            image: kubevirt/cirros-registry-disk-demo:latest
        - name: cloudinitdisk
          cloudInitNoCloud:
            secretRef:
              name: my-vmi-secret
    END

    # Post the VM
    kubectl create -f my-vmi.yaml

#### 使用 Cloud-init 的 Cloud-config 注入 SSH 密钥 {#injecting-ssh-keys-with-cloud-inits-cloud-config}

到目前为止的示例中，cloud-init 用户数据脚本一直是 bash 脚本。Cloud-init 有自己的配置，可以处理一些常见任务，例如创建用户和注入 SSH 密钥。

更多 cloud-config 示例可以在此处找到：[Cloud-init 示例](https://cloudinit.readthedocs.io/en/latest/topics/examples.html)

<!-- markdown-link-check-disable -->
下面的示例使用 cloud-config，为 [Fedora Atomic](https://www.projectatomic.io/) 磁盘镜像的默认用户（本例中为 fedora）注入 SSH 密钥。
<!-- markdown-link-check-enable -->

    # Create the cloud-init cloud-config userdata.
    cat << END > startup-script
    #cloud-config
    password: atomic
    chpasswd: { expire: False }
    ssh_pwauth: False
    ssh_authorized_keys:
        - ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABAQC6zdgFiLr1uAK7PdcchDd+LseA5fEOcxCCt7TLlr7Mx6h8jUg+G+8L9JBNZuDzTZSF0dR7qwzdBBQjorAnZTmY3BhsKcFr8Gt4KMGrS6r3DNmGruP8GORvegdWZuXgASKVpXeI7nCIjRJwAaK1x+eGHwAWO9Z8ohcboHbLyffOoSZDSIuk2kRIc47+ENRjg0T6x2VRsqX27g6j4DfPKQZGk0zvXkZaYtr1e2tZgqTBWqZUloMJK8miQq6MktCKAS4VtPk0k7teQX57OGwD6D7uo4b+Cl8aYAAwhn0hc0C2USfbuVHgq88ESo2/+NwV4SQcl3sxCW21yGIjAGt4Hy7J fedora@localhost.localdomain
    END

    # Create the VM spec
    cat << END > my-vmi.yaml
    apiVersion: kubevirt.io/v1
    kind: VirtualMachineInstance
    metadata:
      name: sshvmi
    spec:
      terminationGracePeriodSeconds: 0
      domain:
        resources:
          requests:
            memory: 1024M
        devices:
          disks:
          - name: containerdisk
            disk:
              bus: virtio
          - name: cloudinitdisk
            disk:
              bus: virtio
      volumes:
        - name: containerdisk
          containerDisk:
            image: kubevirt/fedora-atomic-registry-disk-demo:latest
        - name: cloudinitdisk
          cloudInitNoCloud:
            userDataBase64: $(cat startup-script | base64 -w0)
    END

    # Post the VirtualMachineInstance spec to KubeVirt.
    kubectl create -f my-vmi.yaml

    # Connect to VM with passwordless SSH key
    ssh -i <insert private key here> fedora@<insert ip here>

#### 使用自定义 Shell 脚本注入 SSH 密钥 {#inject-ssh-key-using-a-custom-shell-script}

取决于所用的启动镜像，用户在使用 cloud-init 的 cloud-config 创建用户并注入 SSH 密钥时，体验可能参差不齐。

下面的示例使用脚本而不是 cloud-config 来创建用户，并为该用户注入 SSH 密钥。

    cat << END > startup-script.sh
    #!/bin/bash
    export NEW_USER="foo"
    export SSH_PUB_KEY="ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABAQC6zdgFiLr1uAK7PdcchDd+LseA5fEOcxCCt7TLlr7Mx6h8jUg+G+8L9JBNZuDzTZSF0dR7qwzdBBQjorAnZTmY3BhsKcFr8Gt4KMGrS6r3DNmGruP8GORvegdWZuXgASKVpXeI7nCIjRJwAaK1x+eGHwAWO9Z8ohcboHbLyffOoSZDSIuk2kRIc47+ENRjg0T6x2VRsqX27g6j4DfPKQZGk0zvXkZaYtr1e2tZgqTBWqZUloMJK8miQq6MktCKAS4VtPk0k7teQX57OGwD6D7uo4b+Cl8aYAAwhn0hc0C2USfbuVHgq88ESo2/+NwV4SQcl3sxCW21yGIjAGt4Hy7J $NEW_USER@localhost.localdomain"

    sudo adduser -U -m $NEW_USER
    echo "$NEW_USER:atomic" | chpasswd
    sudo mkdir /home/$NEW_USER/.ssh
    sudo echo "$SSH_PUB_KEY" > /home/$NEW_USER/.ssh/authorized_keys
    sudo chown -R ${NEW_USER}: /home/$NEW_USER/.ssh
    END

    # Create the VM spec
    cat << END > my-vmi.yaml
    apiVersion: kubevirt.io/v1
    kind: VirtualMachineInstance
    metadata:
      name: sshvmi
    spec:
      terminationGracePeriodSeconds: 0
      domain:
        resources:
          requests:
            memory: 1024M
        devices:
          disks:
          - name: containerdisk
            disk:
              bus: virtio
          - name: cloudinitdisk
            disk:
              bus: virtio
      volumes:
        - name: containerdisk
          containerDisk:
            image: kubevirt/fedora-atomic-registry-disk-demo:latest
        - name: cloudinitdisk
          cloudInitNoCloud:
            userDataBase64: $(cat startup-script.sh | base64 -w0)
    END

    # Post the VirtualMachineInstance spec to KubeVirt.
    kubectl create -f my-vmi.yaml

    # Connect to VM with passwordless SSH key
    ssh -i <insert private key here> foo@<insert ip here>

### 网络配置 {#network-config}

可以设置 cloud-init 的[网络版本 1](https://cloudinit.readthedocs.io/en/latest/reference/network-config-format-v1.html)配置，以便在启动时配置网络。

要使 cloud-init 解析 *network-config*，**必须**设置 Cloud-init [用户数据](#user-data)，哪怕它只是用户数据配置头：

    #cloud-config

#### 以明文形式提供的 Cloud-init network-config {#cloud-init-network-config-as-clear-text}

在下面的示例中，一段简单的 cloud-init network-config 以明文形式存储在 cloudInitNoCloud 卷的 networkData 字段中。有一个对应的 disks 条目引用该 cloud-init 卷，并将其分配给 VM 的设备。

    # Create a VM manifest with the network-config in
    # a cloudInitNoCloud volume's networkData field.

    cat << END > my-vmi.yaml
    apiVersion: kubevirt.io/v1alpha2
    kind: VirtualMachineInstance
    metadata:
      name: myvmi
    spec:
      terminationGracePeriodSeconds: 5
      domain:
        resources:
          requests:
            memory: 64M
        devices:
          disks:
          - name: containerdisk
            volumeName: registryvolume
            disk:
              bus: virtio
          - name: cloudinitdisk
            volumeName: cloudinitvolume
            disk:
              bus: virtio
      volumes:
        - name: registryvolume
          containerDisk:
            image: kubevirt/cirros-container-disk-demo:latest
        - name: cloudinitvolume
          cloudInitNoCloud:
            userData: "#cloud-config"
            networkData: |
              network:
                version: 1
                config:
                - type: physical
                name: eth0
                subnets:
                  - type: dhcp

    END

    # Post the Virtual Machine spec to KubeVirt.

    kubectl create -f my-vmi.yaml

#### 以 base64 字符串形式提供的 Cloud-init network-config {#cloud-init-network-config-as-base64-string}

在下面的示例中，一段简单的 network-config 经过 base64 编码，并存储在 cloudInitNoCloud 卷的 networkDataBase64 字段中。有一个对应的 disks 条目引用该 cloud-init 卷，并将其分配给 VM 的设备。

*用户也可以选择把 network-config 存放在 Kubernetes Secret 中，并在 VM 的 spec 里引用该 Secret。文档后面的示例说明了具体做法。*

    # Create a simple network-config

    cat << END > network-config
    network:
      version: 1
      config:
      - type: physical
      name: eth0
      subnets:
        - type: dhcp
    END

    # Create a VM manifest with the networkData base64 encoded into
    # a cloudInitNoCloud volume's networkDataBase64 field.

    cat << END > my-vmi.yaml
    apiVersion: kubevirt.io/v1alpha2
    kind: VirtualMachineInstance
    metadata:
      name: myvmi
    spec:
      terminationGracePeriodSeconds: 5
      domain:
        resources:
          requests:
            memory: 64M
        devices:
          disks:
          - name: containerdisk
            volumeName: registryvolume
            disk:
              bus: virtio
          - name: cloudinitdisk
            volumeName: cloudinitvolume
            disk:
              bus: virtio
      volumes:
        - name: registryvolume
          containerDisk:
            image: kubevirt/cirros-container-disk-demo:latest
        - name: cloudinitvolume
          cloudInitNoCloud:
            userData: "#cloud-config"
            networkDataBase64: $(cat network-config | base64 -w0)
    END

    # Post the Virtual Machine spec to KubeVirt.

    kubectl create -f my-vmi.yaml

#### 以 k8s Secret 形式提供的 Cloud-init network-config {#cloud-init-network-config-as-k8s-secret}

不希望把 cloud-init network-config 直接存放在 VirtualMachineInstance spec 中的用户，可以选择把 network-config 存入 Kubernetes Secret，并在 spec 中引用该 Secret。

多个 VirtualMachineInstance spec 可以引用同一个包含 cloud-init network-config 的 Kubernetes Secret。

下面的示例说明如何创建包含 network-config 的 Kubernetes Secret，并在 VM 的 spec 中引用该 Secret。

    # Create a simple network-config

    cat << END > network-config
    network:
      version: 1
      config:
      - type: physical
      name: eth0
      subnets:
        - type: dhcp
    END

    # Store the network-config in a Kubernetes Secret
    kubectl create secret generic my-vmi-secret --from-file=networkdata=network-config

    # Create a VM manifest and reference the Secret's name in the cloudInitNoCloud
    # Volume's secretRef field

    cat << END > my-vmi.yaml
    apiVersion: kubevirt.io/v1alpha2
    kind: VirtualMachineInstance
    metadata:
      name: myvmi
    spec:
      terminationGracePeriodSeconds: 5
      domain:
        resources:
          requests:
            memory: 64M
        devices:
          disks:
          - name: containerdisk
            volumeName: registryvolume
            disk:
              bus: virtio
          - name: cloudinitdisk
            volumeName: cloudinitvolume
            disk:
              bus: virtio
      volumes:
        - name: registryvolume
          containerDisk:
            image: kubevirt/cirros-registry-disk-demo:latest
        - name: cloudinitvolume
          cloudInitNoCloud:
            userData: "#cloud-config"
            networkDataSecretRef:
              name: my-vmi-secret
    END

    # Post the VM
    kubectl create -f my-vmi.yaml

### 调试 {#debugging}

取决于所用的操作系统发行版，cloud-init 的输出通常会在启动时打印到控制台输出中。在开发用户数据脚本时，用户可以在启动期间连接到 VM 的控制台进行调试。

使用 virtctl 连接到控制台的示例：

    virtctl console <name of vmi>

### 设备角色标记 {#device-role-tagging}

KubeVirt 提供一种机制，让用户可以为网络接口等设备标记特定角色。该标记将与设备的硬件地址相匹配，并且这一映射会通过 cloud-init 暴露给客户机操作系统。

这些额外的元数据将帮助拥有多个网络接口的客户机操作系统用户识别可能具有特定角色的设备，例如专用于特定服务的网络设备，或打算由特定应用程序（数据库、网页缓存等）使用的磁盘。

此功能已经存在于 OpenStack 等平台中。KubeVirt 将以类似的格式提供这些数据，这种格式为 cloud-init 等用户和服务所熟知。

例如：
```yaml
kind: VirtualMachineInstance
spec:
  domain:
    devices:
      interfaces:
      - masquerade: {}
        name: default
      - bridge: {}
        name: ptp
        tag: ptp
      - name: sriov-net
        sriov: {}
        tag: nfvfunc
  networks:
  - name: default
    pod: {}
  - multus:
      networkName: ptp-conf
    name: ptp
  - multus:
      networkName: sriov/sriov-network
    name: sriov-net

The metadata will be available in the guests config drive `openstack/latest/meta_data.json`

{
  "devices": [
    {
        "type": "nic",
        "bus": "pci",
        "address": "0000:00:02.0",
        "mac": "01:22:22:42:22:21",
        "tags": ["ptp"]
    },
    {
        "type": "nic",
        "bus": "pci",
        "address": "0000:81:10.1",
        "mac": "01:22:22:42:22:22",
        "tags": ["nfvfunc"]
    },
  ]
}
```

## Ignition 示例 {#ignition-examples}

Ignition 数据可以使用明文、base64 字符串或 k8s Secret 传入 `cloudInitConfigDrive` 源。

一些 Ignition 配置示例可以在 Ignition 文档给出的[示例](https://coreos.github.io/ignition/examples/)中找到。

### 以明文形式提供的 Ignition {#ignition-as-clear-text}

下面是一个完整示例，展示使用 Ignition 的 KubeVirt VM 如何在首次启动时向 `coreos` 用户添加 SSH 密钥：

```yaml
apiVersion: kubevirt.io/v1alpha3
kind: VirtualMachine
metadata:
  name: ign-demo
spec:
  runStrategy: Halted
  template:
    metadata:
      labels:
        kubevirt.io/size: small
        kubevirt.io/domain: ign-demo
    spec:
      domain:
        devices:
          disks:
            - name: containerdisk
              disk:
                bus: virtio
            - name: cloudinitdisk
              disk:
                bus: virtio
          interfaces:
            - name: default
              masquerade: {}
        resources:
          requests:
            memory: 2G
      networks:
        - name: default
          pod: {}
      volumes:
        - name: containerdisk
          containerDisk:
            image: quay.io/containerdisks/rhcos:4.9
        - name: cloudinitdisk
          cloudInitConfigDrive:
            userData: |
              {
                "ignition": {
                  "config": {},
                  "proxy": {},
                  "security": {},
                  "timeouts": {},
                  "version": "3.2.0"
                },
                "passwd": {
                  "users": [
                    {
                      "name": "coreos",
                      "sshAuthorizedKeys": [
                        "ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAIPL3axFGHI3db9iJWkPXVbYzD7OaWTtHuqmxLvj+DztB user@example"
                      ]
                    }
                  ]
                },
                "storage": {},
                "systemd": {}
              }
```

可以看到，Ignition 配置只是被传入 `userData` 注解，该注解位于 `cloudInitConfigDrive` 卷中。

### 以 base64 形式提供的 Ignition {#ignition-as-base64}

也可以通过使用 `userDatabase64` 注解，以 base64 字符串的形式传入 Ignition 配置：

```yaml
...
cloudInitConfigDrive:
  userDataBase64: eyJpZ25pdGlvbiI6eyJjb25maWciOnt9LCJwcm94eSI6e30sInNlY3VyaXR5Ijp7fSwidGltZW91dHMiOnt9LCJ2ZXJzaW9uIjoiMy4yLjAifSwicGFzc3dkIjp7InVzZXJzIjpbeyJuYW1lIjoiY29yZW9zIiwic3NoQXV0aG9yaXplZEtleXMiOlsic3NoLWVkMjU1MTlBQUFBQzNOemFDMWxaREkxTlRFNUFBQUFJUEwzYXhGR0hJM2RiOWlKV2tQWFZiWXpEN09hV1R0SHVxbXhMdmorRHp0QiB1c2VyQGV4YW1wbGUiXX1dfSwic3RvcmFnZSI6e30sInN5c3RlbWQiOnt9fQ==
```

你可以在终端中执行 `cat ignition.json | base64 -w0` 来获得该 base64 字符串。

### 以 k8s Secret 形式提供的 Ignition {#ignition-as-k8s-secret}

如果你不想把 Ignition 配置存入 VM 配置，可以使用 k8s Secret。

首先，创建其中包含 ignition 数据的 Secret：

```bash
kubectl create secret generic my-ign-secret --from-file=ignition=ignition.json
```

然后在 VM 配置中指定该 Secret：

```yaml
...
cloudInitConfigDrive:
  secretRef:
    name: my-ign-secret
```

## Sysprep 示例 {#sysprep-examples}

### ConfigMap 中的 Sysprep {#sysprep-in-a-configmap}

应答文件可以在 ConfigMap 中提供：

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: sysprep-config
data:
  autounattend.xml: |
    <?xml version="1.0" encoding="utf-8"?>
    <unattend xmlns="urn:schemas-microsoft-com:unattend">
    ...
    </unattend>
```

并像这样挂接到 VM：

```yaml
kind: VirtualMachine
metadata:
  name: windows-with-sysprep
spec:
  runStrategy: Halted
  template:
    metadata:
      labels:
        kubevirt.io/domain: windows-with-sysprep
    spec:
      domain:
        cpu:
          cores: 3
        devices:
          disks:
          - bootOrder: 1
            disk:
              bus: virtio
            name: harddrive
          - name: sysprep
            cdrom:
              bus: sata
        machine:
          type: q35
        resources:
          requests:
            memory: 6G
      volumes:
      - name: harddrive
        persistentVolumeClaim:
          claimName: windows_pvc
      - name: sysprep
        sysprep:
          configMap:
            name: sysprep-config
```

### Secret 中的 Sysprep {#sysprep-in-a-secret}

应答文件可以在 Secret 中提供：

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: sysprep-config
stringData:
data:
  autounattend.xml: |
    <?xml version="1.0" encoding="utf-8"?>
    <unattend xmlns="urn:schemas-microsoft-com:unattend">
    ...
    </unattend>
```

并像这样挂接到 VM：

```yaml
kind: VirtualMachine
metadata:
  name: windows-with-sysprep
spec:
  runStrategy: Halted
  template:
    metadata:
      labels:
        kubevirt.io/domain: windows-with-sysprep
    spec:
      domain:
        cpu:
          cores: 3
        devices:
          disks:
          - bootOrder: 1
            disk:
              bus: virtio
            name: harddrive
          - name: sysprep
            cdrom:
              bus: sata
        machine:
          type: q35
        resources:
          requests:
            memory: 6G
      volumes:
      - name: harddrive
        persistentVolumeClaim:
          claimName: windows_pvc
      - name: sysprep
        sysprep:
          secret:
            name: sysprep-secret
```

### 基础 Sysprep VM {#base-sysprep-vm}

在下面的示例中，使用包含 `autounattend.xml` 文件的 ConfigMap 来修改从 Microsoft 下载的 Windows ISO 镜像，并创建一台已完成基础安装的 Windows 机器，其中安装了 virtio 驱动，且执行了 `post-install.ps1` 中的全部命令。要使下面的清单生效，需要具备 `win10-iso` DataVolume。

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: win10-template-configmap
data:
  autounattend.xml: |-
    <?xml version="1.0" encoding="utf-8"?>
    <unattend xmlns="urn:schemas-microsoft-com:unattend">
      <settings pass="windowsPE">
        <component xmlns:wcm="http://schemas.microsoft.com/WMIConfig/2002/State" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" name="Microsoft-Windows-International-Core-WinPE" processorArchitecture="amd64" publicKeyToken="31bf3856ad364e35" language="neutral" versionScope="nonSxS">
          <SetupUILanguage>
            <UILanguage>en-US</UILanguage>
          </SetupUILanguage>
          <InputLocale>0409:00000409</InputLocale>
          <SystemLocale>en-US</SystemLocale>
          <UILanguage>en-US</UILanguage>
          <UILanguageFallback>en-US</UILanguageFallback>
          <UserLocale>en-US</UserLocale>
        </component>
        <component xmlns:wcm="http://schemas.microsoft.com/WMIConfig/2002/State" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" name="Microsoft-Windows-PnpCustomizationsWinPE" processorArchitecture="amd64" publicKeyToken="31bf3856ad364e35" language="neutral" versionScope="nonSxS">
          <DriverPaths>
            <PathAndCredentials wcm:keyValue="4b29ba63" wcm:action="add">
              <Path>E:\amd64\2k19</Path>
            </PathAndCredentials>
            <PathAndCredentials wcm:keyValue="25fe51ea" wcm:action="add">
              <Path>E:\NetKVM\2k19\amd64</Path>
            </PathAndCredentials>
          </DriverPaths>
        </component>
        <component xmlns:wcm="http://schemas.microsoft.com/WMIConfig/2002/State" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" name="Microsoft-Windows-Setup" processorArchitecture="amd64" publicKeyToken="31bf3856ad364e35" language="neutral" versionScope="nonSxS">
          <DiskConfiguration>
            <Disk wcm:action="add">
              <CreatePartitions>
                <CreatePartition wcm:action="add">
                  <Order>1</Order>
                  <Type>Primary</Type>
                  <Size>100</Size>
                </CreatePartition>
                <CreatePartition wcm:action="add">
                  <Extend>true</Extend>
                  <Order>2</Order>
                  <Type>Primary</Type>
                </CreatePartition>
              </CreatePartitions>
              <ModifyPartitions>
                <ModifyPartition wcm:action="add">
                  <Format>NTFS</Format>
                  <Label>System Reserved</Label>
                  <Order>1</Order>
                  <PartitionID>1</PartitionID>
                  <TypeID>0x27</TypeID>
                </ModifyPartition>
                <ModifyPartition wcm:action="add">
                  <Format>NTFS</Format>
                  <Label>OS</Label>
                  <Letter>C</Letter>
                  <Order>2</Order>
                  <PartitionID>2</PartitionID>
                </ModifyPartition>
              </ModifyPartitions>
              <DiskID>0</DiskID>
              <WillWipeDisk>true</WillWipeDisk>
            </Disk>
          </DiskConfiguration>
          <ImageInstall>
            <OSImage>
              <InstallFrom>
                <MetaData wcm:action="add">
                  <Key>/Image/Description</Key>
                  <Value>Windows 10 Pro</Value>
                </MetaData>
              </InstallFrom>
              <InstallTo>
                <DiskID>0</DiskID>
                <PartitionID>2</PartitionID>
              </InstallTo>
            </OSImage>
          </ImageInstall>
          <UserData>
            <AcceptEula>true</AcceptEula>
            <FullName/>
            <Organization/>
            <ProductKey>
              <Key/>
            </ProductKey>
          </UserData>
        </component>
      </settings>
      <settings pass="offlineServicing">
        <component xmlns:wcm="http://schemas.microsoft.com/WMIConfig/2002/State" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" name="Microsoft-Windows-LUA-Settings" processorArchitecture="amd64" publicKeyToken="31bf3856ad364e35" language="neutral" versionScope="nonSxS">
          <EnableLUA>false</EnableLUA>
        </component>
      </settings>
      <settings pass="specialize">
        <component xmlns:wcm="http://schemas.microsoft.com/WMIConfig/2002/State" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" name="Microsoft-Windows-International-Core" processorArchitecture="amd64" publicKeyToken="31bf3856ad364e35" language="neutral" versionScope="nonSxS">
          <InputLocale>0409:00000409</InputLocale>
          <SystemLocale>en-US</SystemLocale>
          <UILanguage>en-US</UILanguage>
          <UILanguageFallback>en-US</UILanguageFallback>
          <UserLocale>en-US</UserLocale>
        </component>
        <component xmlns:wcm="http://schemas.microsoft.com/WMIConfig/2002/State" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" name="Microsoft-Windows-Security-SPP-UX" processorArchitecture="amd64" publicKeyToken="31bf3856ad364e35" language="neutral" versionScope="nonSxS">
          <SkipAutoActivation>true</SkipAutoActivation>
        </component>
        <component xmlns:wcm="http://schemas.microsoft.com/WMIConfig/2002/State" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" name="Microsoft-Windows-SQMApi" processorArchitecture="amd64" publicKeyToken="31bf3856ad364e35" language="neutral" versionScope="nonSxS">
          <CEIPEnabled>0</CEIPEnabled>
        </component>
      </settings>
      <settings pass="oobeSystem">
        <component xmlns:wcm="http://schemas.microsoft.com/WMIConfig/2002/State" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" name="Microsoft-Windows-Shell-Setup" processorArchitecture="amd64" publicKeyToken="31bf3856ad364e35" language="neutral" versionScope="nonSxS">
          <OOBE>
            <HideEULAPage>true</HideEULAPage>
            <HideOEMRegistrationScreen>true</HideOEMRegistrationScreen>
            <HideOnlineAccountScreens>true</HideOnlineAccountScreens>
            <HideWirelessSetupInOOBE>true</HideWirelessSetupInOOBE>
            <NetworkLocation>Work</NetworkLocation>
            <SkipUserOOBE>true</SkipUserOOBE>
            <SkipMachineOOBE>true</SkipMachineOOBE>
            <ProtectYourPC>3</ProtectYourPC>
          </OOBE>
          <AutoLogon>
            <Password>
              <Value>123456</Value>
              <PlainText>true</PlainText>
            </Password>
            <Enabled>true</Enabled>
            <Username>Administrator</Username>
          </AutoLogon>
          <UserAccounts>
            <AdministratorPassword>
              <Value>123456</Value>
              <PlainText>true</PlainText>
            </AdministratorPassword>
          </UserAccounts>
          <RegisteredOrganization/>
          <RegisteredOwner/>
          <TimeZone>Eastern Standard Time</TimeZone>
          <FirstLogonCommands>
            <SynchronousCommand wcm:action="add">
              <CommandLine>powershell -ExecutionPolicy Bypass -NoExit -NoProfile f:\post-install.ps1</CommandLine>
              <RequiresUserInput>false</RequiresUserInput>
              <Order>1</Order>
              <Description>Post Installation Script</Description>
            </SynchronousCommand>
          </FirstLogonCommands>
        </component>
      </settings>
    </unattend>

  
  post-install.ps1: |-
    # Remove AutoLogin
    # https://docs.microsoft.com/en-us/windows-hardware/customize/desktop/unattend/microsoft-windows-shell-setup-autologon-logoncount#logoncount-known-issue
    reg add "HKEY_LOCAL_MACHINE\SOFTWARE\Microsoft\Windows NT\CurrentVersion\Winlogon" /v AutoAdminLogon /t REG_SZ /d 0 /f

    # install Qemu Tools (Drivers)
    Start-Process msiexec -Wait -ArgumentList '/i e:\virtio-win-gt-x64.msi /qn /passive /norestart'

    # install Guest Agent
    Start-Process msiexec -Wait -ArgumentList '/i e:\guest-agent\qemu-ga-x86_64.msi /qn /passive /norestart'

    # Rename cached unattend.xml to avoid it is picked up by sysprep
    mv C:\Windows\Panther\unattend.xml C:\Windows\Panther\unattend.install.xml

    # Eject CD, to avoid that the autounattend.xml on the CD is picked up by sysprep
    (new-object -COM Shell.Application).NameSpace(17).ParseName('F:').InvokeVerb('Eject')

    # Run Sysprep and Shutdown
    C:\Windows\System32\Sysprep\sysprep.exe /generalize /oobe /shutdown /mode:vm

---

apiVersion: kubevirt.io/v1
kind: VirtualMachine
metadata:
  annotations:
    name.os.template.kubevirt.io/win10: Microsoft Windows 10
    vm.kubevirt.io/validations: |
      [
        {
          "name": "minimal-required-memory",
          "path": "jsonpath::.spec.domain.resources.requests.memory",
          "rule": "integer",
          "message": "This VM requires more memory.",
          "min": 2147483648
        }, {
          "name": "windows-virtio-bus",
          "path": "jsonpath::.spec.domain.devices.disks[*].disk.bus",
          "valid": "jsonpath::.spec.domain.devices.disks[*].disk.bus",
          "rule": "enum",
          "message": "virto disk bus type has better performance, install virtio drivers in VM and change bus type",
          "values": ["virtio"],
          "justWarning": true
        }, {
          "name": "windows-disk-bus",
          "path": "jsonpath::.spec.domain.devices.disks[*].disk.bus",
          "valid": "jsonpath::.spec.domain.devices.disks[*].disk.bus",
          "rule": "enum",
          "message": "disk bus has to be either virtio or sata or scsi",
          "values": ["virtio", "sata", "scsi"]
        }, {
          "name": "windows-cd-bus",
          "path": "jsonpath::.spec.domain.devices.disks[*].cdrom.bus",
          "valid": "jsonpath::.spec.domain.devices.disks[*].cdrom.bus",
          "rule": "enum",
          "message": "cd bus has to be sata",
          "values": ["sata"]
        }
      ]
  name: win10-template
  namespace: default
  labels:
    app: win10-template
    flavor.template.kubevirt.io/medium: 'true'
    os.template.kubevirt.io/win10: 'true'
    vm.kubevirt.io/template: windows10-desktop-medium
    vm.kubevirt.io/template.namespace: openshift
    vm.kubevirt.io/template.revision: '1'
    vm.kubevirt.io/template.version: v0.14.0
    workload.template.kubevirt.io/desktop: 'true'
spec:
  runStrategy: RerunOnFailure
  dataVolumeTemplates:
    - metadata:
        name: win10-template-windows-iso
      spec:
        storage: {}
        source:
          pvc:
            name: windows10-iso
            namespace: default
    - metadata:
        name: win10-template
      spec:
        storage:
          resources:
            requests:
              storage: 25Gi
          volumeMode: Filesystem
        source:
          blank: {}
  template:
    metadata:
      annotations:
        vm.kubevirt.io/flavor: medium
        vm.kubevirt.io/os: windows10
        vm.kubevirt.io/workload: desktop
      labels:
        flavor.template.kubevirt.io/medium: 'true'
        kubevirt.io/domain: win10-template
        kubevirt.io/size: medium
        os.template.kubevirt.io/win10: 'true'
        vm.kubevirt.io/name: win10-template
        workload.template.kubevirt.io/desktop: 'true'
    spec:
      domain:
        clock:
          timer:
            hpet:
              present: false
            hyperv: {}
            pit:
              tickPolicy: delay
            rtc:
              tickPolicy: catchup
          utc: {}
        cpu:
          cores: 1
          sockets: 1
          threads: 1
        devices:
          disks:
            - bootOrder: 1
              disk:
                bus: virtio
              name: win10-template
            - bootOrder: 2
              cdrom:
                bus: sata
              name: windows-iso
            - cdrom:
                bus: sata
              name: windows-guest-tools
            - name: sysprep
              cdrom:
                bus: sata
          inputs:
            - bus: usb
              name: tablet
              type: tablet
          interfaces:
            - masquerade: {}
              model: virtio
              name: default
        features:
          acpi: {}
          apic: {}
          hyperv:
            reenlightenment: {}
            ipi: {}
            synic: {}
            synictimer:
              direct: {}
            spinlocks:
              spinlocks: 8191
            reset: {}
            relaxed: {}
            vpindex: {}
            runtime: {}
            tlbflush: {}
            frequencies: {}
            vapic: {}
        machine:
          type: pc-q35-rhel8.4.0
        resources:
          requests:
            memory: 4Gi
      hostname: win10-template
      networks:
        - name: default
          pod: {}
      volumes:
        - dataVolume:
            name: win10-iso
          name: windows-iso
        - dataVolume:
            name: win10-template-windows-iso
          name: win10-template
        - containerDisk:
            image: quay.io/kubevirt/virtio-container-disk
          name: windows-guest-tools
        - name: sysprep
          sysprep:
            configMap:
              name: win10-template-configmap

```

### 从模板启动 VM {#launching-a-vm-from-template}

在上面的示例中，当 `post-install.ps1` 中的 sysprep 命令执行完毕并且 VM 处于关机状态后，可以从基础 `win10-template` 启动一台新 VM，并应用下面 `unattend.xml`（位于 `sysprep-config` 中）所提到的额外更改。新 VM 进入运行状态最多可能需要 5 分钟，因为 Windows 会在后台按照下面 `unattend.xml` 文件中指定的自定义项进行 oobe 设置。

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: sysprep-config
data:
  autounattend.xml: |-
    <?xml version="1.0" encoding="utf-8"?>
    <!-- responsible for installing windows, ignored on sysprepped images -->
  unattend.xml: |-
    <?xml version="1.0" encoding="utf-8"?>
    <unattend xmlns="urn:schemas-microsoft-com:unattend">
      <settings pass="oobeSystem">
        <component xmlns:wcm="http://schemas.microsoft.com/WMIConfig/2002/State" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" name="Microsoft-Windows-Shell-Setup" processorArchitecture="amd64" publicKeyToken="31bf3856ad364e35" language="neutral" versionScope="nonSxS">
          <OOBE>
            <HideEULAPage>true</HideEULAPage>
            <HideOEMRegistrationScreen>true</HideOEMRegistrationScreen>
            <HideOnlineAccountScreens>true</HideOnlineAccountScreens>
            <HideWirelessSetupInOOBE>true</HideWirelessSetupInOOBE>
            <NetworkLocation>Work</NetworkLocation>
            <SkipUserOOBE>true</SkipUserOOBE>
            <SkipMachineOOBE>true</SkipMachineOOBE>
            <ProtectYourPC>3</ProtectYourPC>
          </OOBE>
          <AutoLogon>
            <Password>
            <Value>123456</Value>
              <PlainText>true</PlainText>
            </Password>
            <Enabled>true</Enabled>
        <Username>Administrator</Username>
    </AutoLogon>
    <UserAccounts>
         <AdministratorPassword>
                <Value>123456</Value>
                <PlainText>true</PlainText>
        </AdministratorPassword>
          </UserAccounts>
          <RegisteredOrganization>Kuebvirt</RegisteredOrganization>
          <RegisteredOwner>Kubevirt</RegisteredOwner>
          <TimeZone>Eastern Standard Time</TimeZone>
                <FirstLogonCommands>
                    <SynchronousCommand wcm:action="add">
                    <CommandLine>powershell -ExecutionPolicy Bypass -NoExit -WindowStyle Hidden -NoProfile d:\customize.ps1</CommandLine>
                        <RequiresUserInput>false</RequiresUserInput>
                        <Order>1</Order>
                <Description>Customize Script</Description>
            </SynchronousCommand>
                </FirstLogonCommands>
        </component>
      </settings>
    </unattend>
  customize.ps1: |-
    # Enable RDP
    Set-ItemProperty -Path 'HKLM:\System\CurrentControlSet\Control\Terminal Server' -name "fDenyTSConnections" -value 0
    Enable-NetFirewallRule -DisplayGroup "Remote Desktop"


    # https://docs.microsoft.com/en-us/windows-server/administration/openssh/openssh_install_firstuse
    # Install the OpenSSH Server
    Add-WindowsCapability -Online -Name OpenSSH.Server~~~~0.0.1.0
    # Start the sshd service
    Start-Service sshd

    Set-Service -Name sshd -StartupType 'Automatic'

    # https://docs.microsoft.com/en-us/windows-server/administration/openssh/openssh_server_configuration
    # use powershell as default shell for ssh
    New-ItemProperty -Path "HKLM:\SOFTWARE\OpenSSH" -Name DefaultShell -Value "C:\Windows\System32\WindowsPowerShell\v1.0\powershell.exe" -PropertyType String -Force


    # Add ssh authorized_key for administrator
    # https://docs.microsoft.com/en-us/windows-server/administration/openssh/openssh_keymanagement
    $MyDir = $MyInvocation.MyCommand.Path | Split-Path -Parent
    $PublicKey = Get-Content -Path $MyDir\id_rsa.pub
    $authrized_keys_path = $env:ProgramData + "\ssh\administrators_authorized_keys" 
    Add-Content -Path $authrized_keys_path -Value $PublicKey
    icacls.exe $authrized_keys_path /inheritance:r /grant "Administrators:F" /grant "SYSTEM:F"


    # install application via exe file installer from url
    function Install-Exe {
      $dlurl = $args[0]
      $installerPath = Join-Path $env:TEMP (Split-Path $dlurl -Leaf)
      Invoke-WebRequest -UseBasicParsing $dlurl -OutFile $installerPath
      Start-Process -FilePath $installerPath -Args "/S" -Verb RunAs -Wait
      Remove-Item $installerPath

    }

    # Wait for networking before running a task at startup
    do {
      $ping = test-connection -comp kubevirt.io -count 1 -Quiet
    } until ($ping)

    # Installing the Latest Notepad++ with PowerShell
    $BaseUri = "https://notepad-plus-plus.org"
    $BasePage = Invoke-WebRequest -Uri $BaseUri -UseBasicParsing
    $ChildPath = $BasePage.Links | Where-Object { $_.outerHTML -like '*Current Version*' } | Select-Object -ExpandProperty href
    $DownloadPageUri = $BaseUri + $ChildPath
    $DownloadPage = Invoke-WebRequest -Uri $DownloadPageUri -UseBasicParsing
    $DownloadUrl = $DownloadPage.Links | Where-Object { $_.outerHTML -like '*npp.*.Installer.x64.exe"*' } | Select-Object -ExpandProperty href
    Install-Exe $DownloadUrl
  id_rsa.pub: |-
    ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABAQC6zdgFiLr1uAK7PdcchDd+LseA5fEOcxCCt7TLlr7Mx6h8jUg+G+8L9JBNZuDzTZSF0dR7qwzdBBQjorAnZTmY3BhsKcFr8Gt4KMGrS6r3DNmGruP8GORvegdWZuXgASKVpXeI7nCIjRJwAaK1x+eGHwAWO9Z8ohcboHbLyffOoSZDSIuk2kRIc47+ENRjg0T6x2VRsqX27g6j4DfPKQZGk0zvXkZaYtr1e2tZgqTBWqZUloMJK8miQq6MktCKAS4VtPk0k7teQX57OGwD6D7uo4b+Cl8aYAAwhn0hc0C2USfbuVHgq88ESo2/+NwV4SQcl3sxCW21yGIjAGt4Hy7J fedora@localhost.localdomain


```
