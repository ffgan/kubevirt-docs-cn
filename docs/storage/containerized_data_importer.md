# 容器化数据导入器 {#containerized-data-importer}

[容器化数据导入器](https://github.com/kubevirt/containerized-data-importer)（CDI）项目提供了相应能力，使[持久卷声明](https://kubernetes.io/docs/concepts/storage/persistent-volumes/)（PVC）能够通过 [DataVolume](https://github.com/kubevirt/containerized-data-importer/blob/main/doc/datavolumes.md) 用作 KubeVirt VM 的磁盘。
CDI 的三个主要用例是：

-   将磁盘镜像从 Web 服务器或容器镜像仓库导入到 DataVolume
-   将现有 PVC 克隆到 DataVolume
-   将本地磁盘镜像上传到 DataVolume

本文讨论第三种用例。因此，集群中应已安装 CDI，你应准备好希望上传的 VM 磁盘，并且 virtctl 位于你的 PATH 中。

!!! Note
    你很可能需要配置容器运行时接口（CRI），以便通过安全上下文处理设备所有权。
    更多信息请参阅 [CDI 文档](https://github.com/kubevirt/containerized-data-importer/blob/main/doc/block_cri_ownership_config.md)。

## 安装 CDI {#install-cdi}

在[此处](https://github.com/kubevirt/containerized-data-importer/releases)安装最新的 CDI 发行版

    export TAG=$(curl -s -w %{redirect_url} https://github.com/kubevirt/containerized-data-importer/releases/latest)
    export VERSION=$(echo ${TAG##*/})
    kubectl create -f https://github.com/kubevirt/containerized-data-importer/releases/download/$VERSION/cdi-operator.yaml
    kubectl create -f https://github.com/kubevirt/containerized-data-importer/releases/download/$VERSION/cdi-cr.yaml

## 暴露 cdi-uploadproxy 服务 {#expose-cdi-uploadproxy-service}

`cdi-uploadproxy` 服务必须能够从集群外部访问。以下是一些做法：

-   [NodePort 服务](https://kubernetes.io/docs/concepts/services-networking/service/#nodeport)

-   [Ingress](https://kubernetes.io/docs/concepts/services-networking/ingress/)

-   [Route](https://docs.openshift.com/container-platform/4.10/networking/routes/route-configuration.html)

-   [kubectl port-forward](https://kubernetes.io/docs/tasks/access-application-cluster/port-forward-access-application-cluster/)（不建议用于生产集群）

在[此处](https://github.com/kubevirt/containerized-data-importer/blob/main/doc/upload.md)查看示例清单。

## 支持的镜像格式 {#supported-image-formats}

CDI 支持 qemu 所支持的 `raw` 和 `qcow2` 镜像格式。
更多细节请参阅 [qemu 文档](https://www.qemu.org/docs/master/system/images.html#disk-image-file-formats)。可引导的 ISO 镜像也可以使用，并会被当作 `raw` 镜像处理。镜像可以使用 `gz` 或 `xz` 格式压缩。

本文中的示例使用[此](http://download.cirros-cloud.net/0.4.0/cirros-0.4.0-x86_64-disk.img) [CirrOS](https://launchpad.net/cirros) 镜像

## virtctl image-upload {#virtctl-image-upload}

virtctl 有一个 image-upload 命令，选项如下：

    virtctl image-upload --help
    Upload a VM image to a DataVolume/PersistentVolumeClaim.

    Usage:
      virtctl image-upload [flags]

    Examples:
      # Upload a local disk image to a newly created DataVolume:
      virtctl image-upload dv fedora-dv --size=10Gi --image-path=/images/fedora30.qcow2

      # Upload a local disk image to an existing DataVolume
      virtctl image-upload dv fedora-dv --no-create --image-path=/images/fedora30.qcow2

      # Upload a local disk image to a newly created PersistentVolumeClaim
      virtctl image-upload pvc fedora-pvc --size=10Gi --image-path=/images/fedora30.qcow2

      # Upload a local disk image to a newly created PersistentVolumeClaim and label it with a default instance type and preference
      virtctl image-upload pvc fedora-pvc --size=10Gi --image-path=/images/fedora30.qcow2 --default-instancetype=n1.medium --default-preference=fedora

      # Upload a local disk image to an existing PersistentVolumeClaim
      virtctl image-upload pvc fedora-pvc --no-create --image-path=/images/fedora30.qcow2

      # Upload to a DataVolume with explicit URL to CDI Upload Proxy
      virtctl image-upload dv fedora-dv --uploadproxy-url=https://cdi-uploadproxy.mycluster.com --image-path=/images/fedora30.qcow2

      # Upload a local disk archive to a newly created DataVolume:
      virtctl image-upload dv fedora-dv --size=10Gi --archive-path=/images/fedora30.tar

    Flags:
        --access-mode string                 The access mode for the PVC.
        --archive-path string                Path to the local archive.
        --default-instancetype string        The default instance type to associate with the image.
        --default-instancetype-kind string   The default instance type kind to associate with the image.
        --default-preference string          The default preference to associate with the image.
        --default-preference-kind string     The default preference kind to associate with the image.
        --force-bind                         Force bind the PVC, ignoring the WaitForFirstConsumer logic.
        -h, --help                           help for image-upload
        --image-path string                  Path to the local VM image.
        --insecure                           Allow insecure server connections when using HTTPS.
        --no-create                          Don't attempt to create a new DataVolume/PVC.
        --size string                        The size of the DataVolume to create (ex. 10Gi, 500Mi).
        --storage-class string               The storage class for the PVC.
        --uploadproxy-url string             The URL of the cdi-upload proxy service.
        --volume-mode string                 Specify the VolumeMode (block/filesystem) used to create the PVC. Default is the storageProfile default. For archive upload default is filesystem.
        --wait-secs uint                     Seconds to wait for upload pod to start. (default 300)

    Use "virtctl options" for a list of global command-line options (applies to all commands).

`virtctl image-upload` 的工作方式是：创建所请求大小的 DataVolume，发送 `UploadTokenRequest` 到 `cdi-apiserver`，并将文件上传到 `cdi-uploadproxy`。

    virtctl image-upload dv cirros-vm-disk --size=500Mi --image-path=/home/mhenriks/images/cirros-0.4.0-x86_64-disk.img --uploadproxy-url=<url to upload proxy service>

## 上传镜像时处理证书问题 {#addressing-certificate-issues-when-uploading-images}

可以使用 `--insecure` 标志规避证书问题，以阻止 virtctl 命令校验远程主机。
更好的做法是解决那些阻止使用 `virtctl image-upload` 命令上传镜像的证书问题，而不要使用 `--insecure` 标志。

以下是一些常见的证书问题以及一些简单的修复方法。

### 不包含任何 IP SAN {#does-not-contain-any-ip-sans}

当尝试使用 IP 地址而不是可解析名称上传镜像时，会出现此问题。
例如，尝试上传到 IP 地址 192.168.39.32 的端口 31001 会产生以下错误。

    virtctl image-upload dv f33 \
      --size 5Gi \
      --image-path Fedora-Cloud-Base-33-1.2.x86_64.raw.xz \
      --uploadproxy-url https://192.168.39.32:31001

    PVC default/f33 not found
    DataVolume default/f33 created
    Waiting for PVC f33 upload pod to be ready...
    Pod now ready
    Uploading data to https://192.168.39.32:31001

     0 B / 193.89 MiB [-------------------------------------------------------]   0.00% 0s

    Post https://192.168.39.32:31001/v1beta1/upload: x509: cannot validate certificate for 192.168.39.32 because it doesn't contain any IP SANs

向本地名称解析服务添加一条记录即可轻松修复。
这可以是 DNS 服务器或本地 hosts 文件。
用于上传代理的 URL 应改为反映该可解析名称。

证书中的 `Subject` 和 `Subject Alternative Name` 包含可用于解析的有效名称。
这些名称中只需要有一个可解析。
使用 `openssl` 命令查看 cdi-uploadproxy 服务的名称。

    echo | openssl s_client -showcerts -connect 192.168.39.32:31001 2>/dev/null \
         | openssl x509 -inform pem -noout -text \
         | sed -n -e '/Subject.*CN/p' -e '/Subject Alternative/{N;p}'

        Subject: CN = cdi-uploadproxy
            X509v3 Subject Alternative Name:
                DNS:cdi-uploadproxy, DNS:cdi-uploadproxy.cdi, DNS:cdi-uploadproxy.cdi.svc

如果 /etc/hosts 文件提供名称解析，向其中添加以下条目应能修复此问题。
任何为系统提供名称解析的服务都可以使用。

    echo "192.168.39.32  cdi-uploadproxy" >> /etc/hosts

现在上传应该可以成功。

    virtctl image-upload dv f33 \
      --size 5Gi \
      --image-path Fedora-Cloud-Base-33-1.2.x86_64.raw.xz \
      --uploadproxy-url https://cdi-uploadproxy:31001

    PVC default/f33 not found
    DataVolume default/f33 created
    Waiting for PVC f33 upload pod to be ready...
    Pod now ready
    Uploading data to https://cdi-uploadproxy:31001

     193.89 MiB / 193.89 MiB [=============================================] 100.00% 1m38s

    Uploading data completed successfully, waiting for processing to complete, you can hit ctrl-c without interrupting the progress
    Processing completed successfully
    Uploading Fedora-Cloud-Base-33-1.2.x86_64.raw.xz completed successfully

### 证书由未知机构签名 {#certificate-signed-by-unknown-authority}
出现此情况是因为 cdi-uploadproxy 证书是自签名的，并且系统不信任 cdi-uploadproxy 作为证书颁发机构。

    virtctl image-upload dv f33 \
      --size 5Gi \
      --image-path Fedora-Cloud-Base-33-1.2.x86_64.raw.xz \
      --uploadproxy-url https://cdi-uploadproxy:31001

    PVC default/f33 not found
    DataVolume default/f33 created
    Waiting for PVC f33 upload pod to be ready...
    Pod now ready
    Uploading data to https://cdi-uploadproxy:31001

     0 B / 193.89 MiB [-------------------------------------------------------]   0.00% 0s

    Post https://cdi-uploadproxy:31001/v1beta1/upload: x509: certificate signed by unknown authority

可以通过将该证书添加到系统的信任库来修复。
下载 cdi-uploadproxy-server-cert。

    kubectl get secret -n cdi cdi-uploadproxy-server-cert \
      -o jsonpath="{.data['tls\.crt']}" \
      | base64 -d > cdi-uploadproxy-server-cert.crt

将此证书添加到系统的信任库。在 Fedora 上，可以按如下方式操作。

    sudo cp cdi-uploadproxy-server-cert.crt /etc/pki/ca-trust/source/anchors

    sudo update-ca-trust

现在上传应该可以成功。

    virtctl image-upload dv f33 \
      --size 5Gi \
      --image-path Fedora-Cloud-Base-33-1.2.x86_64.raw.xz \
      --uploadproxy-url https://cdi-uploadproxy:31001

    PVC default/f33 not found
    DataVolume default/f33 created
    Waiting for PVC f33 upload pod to be ready...
    Pod now ready
    Uploading data to https://cdi-uploadproxy:31001

     193.89 MiB / 193.89 MiB [=============================================] 100.00% 1m36s

    Uploading data completed successfully, waiting for processing to complete, you can hit ctrl-c without interrupting the progress
    Processing completed successfully
    Uploading Fedora-Cloud-Base-33-1.2.x86_64.raw.xz completed successfully

## 设置 cdi-upload 代理服务的 URL {#setting-the-url-of-the-cdi-upload-proxy-service}
为 cdi-upload 代理服务设置 URL，可以让 `virtctl image-upload` 命令在不指定 `--uploadproxy-url` 标志的情况下上传镜像。
通过修补 CDI 配置来永久设置该 URL。

以下操作会将默认上传代理设置为使用 cdi-uploadproxy 的 31001 端口。
也可以使用 IP 地址代替 DNS 名称。

关于为何选择 cdi-uploadproxy，以及使用 IP 地址时可能遇到的问题，请参阅“上传镜像时处理证书问题”一节。

    kubectl patch cdi cdi \
      --type merge \
      --patch '{"spec":{"config":{"uploadProxyURLOverride":"https://cdi-uploadproxy:31001"}}}'

## 创建 VirtualMachineInstance {#create-a-virtualmachineinstance}

要从 DataVolume 创建 `VirtualMachineInstance`，可以执行以下操作：

```shell
cat <<EOF | kubectl apply -f -
apiVersion: kubevirt.io/v1
kind: VirtualMachineInstance
metadata:
  name: cirros-vm
spec:
  domain:
    devices:
      disks:
      - disk:
          bus: virtio
        name: dvdisk
    machine:
      type: ""
    resources:
      requests:
        memory: 64M
  terminationGracePeriodSeconds: 0
  volumes:
  - name: dvdisk
    dataVolume:
      name: cirros-vm-disk
status: {}
EOF
```

## 连接到 VirtualMachineInstance 控制台 {#connect-to-virtualmachineinstance-console}

使用 `virtctl` 连接到新建的 `VirtualMachineInstance`。

    virtctl console cirros-vm
