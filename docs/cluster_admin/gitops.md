# 使用 GitOps 管理 KubeVirt {#managing-kubevirt-with-gitops}

GitOps 方式使用 Git 仓库作为单一事实来源来交付基础设施即代码。自动化用于始终保持集群的期望状态与实际状态同步。这意味着对仓库的任何更改都会自动应用到一个或多个集群，而对集群的更改会被自动还原为单一事实来源中所描述的状态。

借助 GitOps，分离测试环境与生产环境、提高应用可用性以及在多集群环境中工作都会变得容易得多。

## 演示仓库 {#demo-repository}

关于如何使用 GitOps 管理 KubeVirt 的详细说明演示见
[此处](https://github.com/0xFelix/gitops-demo)。

该演示使用 [Open Cluster Management](https://open-cluster-management.io/)
和 [ArgoCD](https://argoproj.github.io/cd/) 在多个集群中部署 KubeVirt 和虚拟机。
