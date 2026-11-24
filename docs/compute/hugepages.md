# 大页支持 {#hugepages-support}

要支持大页，至少需要 Kubernetes 版本 `1.9`。

## 启用功能门控 {#enable-feature-gate}

要在 Kubernetes 上启用大页，请查阅[官方文档](https://kubernetes.io/docs/tasks/manage-hugepages/scheduling-hugepages/)。

要在 OKD 上启用大页，请查阅[官方文档](https://docs.openshift.org/3.9/scaling_performance/managing_hugepages.html#huge-pages-prerequisites)。

## 在节点上预分配大页 {#pre-allocate-hugepages-on-a-node}

要在启动时预分配大页，需要在内核启动参数中指定大页 `hugepagesz=2M hugepages=64`，然后重启机器。

你可以在[官方文档](https://www.kernel.org/doc/Documentation/vm/hugetlbpage.txt)中找到关于大页的更多信息。
