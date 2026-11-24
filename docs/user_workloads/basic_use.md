# 基本用法 {#basic-use}

如果你习惯使用 Kubernetes，使用 KubeVirt 应当相当自然。

使用 KubeVirt 的主要方式是操作 Kubernetes API 中的 KubeVirt kind：

    $ kubectl create -f vmi.yaml
    $ kubectl wait --for=condition=Ready vmis/my-vmi
    $ kubectl get vmis
    $ kubectl delete vmis testvmi

以下页面介绍如何使用和发现 API，以及如何管理和访问虚拟机。

## 用户界面 {#user-interface}

KubeVirt 不附带 UI，它只是用虚拟化功能扩展 Kubernetes API。
