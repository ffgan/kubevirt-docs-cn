# 通用技术评审 - KubeVirt / 孵化 {#general-technical-review-kubevirt-incubation}

!!! note
    本文档是作为一项 CNCF 倡议而创建的，也是项目在不同级别之间迁移的一项要求。它最初是在 KubeVirt 毕业流程中创建的。它旨在成为一份持续维护的文档，但并不期望每个发行版都更新。同样，由 CNCF 维护的[通用技术评审模板](https://github.com/cncf/toc/blob/main/toc_subprojects/project-reviews-subproject/general-technical-questions.md)预计也会演进。
    
    请留意 `Date updated` 和 `Template version` 元数据。
    
    如果你发现差异或不一致之处，欢迎提交 issue 或 PR，以便予以纠正。
    

- **项目：KubeVirt**
- **项目版本：1.4**
- **网站：kubevirt.io**
- **更新日期：** 2025-01-24
- **模板版本：** v1.0
- **描述：** KubeVirt 是一个开源项目，它扩展了 Kubernetes，使虚拟机（VM）能够与容器化工作负载一起得到管理。它通过自定义资源定义（CRD）与 Kubernetes API 无缝集成。用户可以在云原生环境中，同时为传统工作负载和容器化工作负载利用 Kubernetes 的能力。


## 第 0 天 - 规划阶段 {#day-0-planning-phase}

### 范围 {#scope}

* 描述路线图流程，中长期功能的范围是如何确定的，以及路线图如何对应到当前贡献和维护者进阶阶梯？

  KubeVirt 项目的路线图由社区的贡献塑造。优先工作项来自 kubevirt/community 仓库中社区提交的提案。我们还在 sig-release 仓库中记录我们的[即将发生的变更](https://github.com/kubevirt/sig-release/blob/main/upcoming-changes.md)，以展示下一个 KubeVirt 版本中可能会包含的内容。
  增强提案框架提供了一种标准化方式，通过 issue 和社区评审来提议并跟踪功能。
  已批准的增强提案驱动优先级，并指导贡献与代码评审。
  路线图通过让贡献者在治理、优先功能、鼓励新贡献者以及评审工作上做出有影响力的工作来获得成长机会，从而支持维护者进阶阶梯。

* 描述本项目的目标用户画像或用户？

  KubeVirt 的目标受众包括基础设施所有者、平台工程师、DevOps 团队和应用开发者。他们需要使用 Kubernetes 来运行并编排与容器并列的虚拟机，从而在统一平台内衔接遗留工作负载与云原生工作负载。

* 说明本项目的主要使用场景。本项目还支持哪些额外的使用场景？

  KubeVirt 的主要使用场景是让组织能够在 Kubernetes 集群中，与容器化工作负载一起管理和编排虚拟机（VM）。这对于混合环境尤其有价值：遗留应用需要 VM，而现代应用已经容器化，从而可以通过 Kubernetes API 和工作流实现统一管理与应用集成。

  **额外使用场景：**

  * **基础设施现代化：**
    在不中断现有 VM 工作负载的情况下，逐步从传统虚拟化平台过渡到云原生基础设施。

  * **混合应用架构：**
    支持将基于 VM 的组件（例如数据库）与容器化工作负载组合在一起的应用。

  * **开发与测试环境：**
    为开发者提供一种 Kubernetes 原生的方式，在 CI/CD 流水线中创建、测试和运行基于 VM 的工作负载。

  * **边缘计算：**
    在边缘设备上管理 VM，以服务于需要专用资源或特定操作系统环境的工作负载。

  * **隔离：**
    将容器化工作负载与 VM 组合起来，以获得更高程度的隔离。

  * **多租户：**
    由 VM 来承载 Kubernetes 集群，或者将 VM 用作虚拟 Kubernetes 集群中的节点。

* 说明哪些使用场景已被认定为本项目不支持的。

  * 类似 “Delta Cloud” 的抽象。也就是让 KubeVirt API 成为面向不同云平台（例如超大规模云厂商）的单一 API。
  * 支持 KVM 以外的虚拟机监控程序，以及 qemu 以外的虚拟机监控器（VMM）运行时。不过，项目仍然对想法保持开放。

* 描述采用本项目后会受益的预期组织类型。（即金融服务机构、任意软件制造商、提供平台工程服务的组织）？

  KubeVirt 将惠及在混合环境中运营，或希望实现基础设施现代化并衔接传统技术与云原生技术的组织。
  有需求运行由 VM 与容器混合工作负载组成的内部私有云，或计划在未来采用混合工作负载的组织。
  由于虚拟化在 IT 中无处不在，KubeVirt 并不绑定于任何特定的市场细分。

* 请描述任何已经完成的终端用户研究，并链接到任何报告。

  据我们所知，目前没有围绕 KubeVirt 开展过的终端用户研究。


### 可用性 {#usability}

### 可用性 {#usability_1}

* 目标用户画像应当如何与你的项目交互？

  虚拟机所有者和集群管理员——即本项目的用户——都应当主要通过 Kubernetes 原生 API 和 KubeVirt 专用 API 和/或 CLI 与 KubeVirt 交互。

* 描述本项目的用户体验（UX）和用户界面（UI）。

  在可用性方面，KubeVirt 尽量采用 Kubernetes 的 UX 和 API 模式，以便为用户提供“统一”的用户体验——无论他们使用的是虚拟机还是容器。

  没有官方的 KubeVirt 用户界面。不过，平台厂商提供了若干用户界面，例如：

  * SUSE Virtualization
  * Red Hat OpenShift Virtualization
  * Deckhouse（Virtualization）
  * KubeVirt Manager（社区项目）

* 描述本项目在生产环境中如何与其他项目集成。

  **Kubernetes** - KubeVirt 通过引入 VirtualMachine 和 VirtualMachineInstance 等自定义资源定义（CRD）来扩展 Kubernetes。这些资源直接融入 Kubernetes 工作流。用户可以使用 Kubernetes 原生 API 来管理 VM。

  **Prometheus** - KubeVirt 组件暴露与 Prometheus 兼容的端点、告警和运维手册，以便与这一监控方案良好集成。

  **Medik8s** - KubeVirt 社区成员向 Medik8s 做了贡献，以便为裸金属 Kubernetes 集群增加高可用支持，从而支撑 KubeVirt 的使用场景。

  **ovn-kubernetes** 和 **kubernetes-ovn** - KubeVirt 贡献者向这两个项目都做了贡献，使它们能够与 KubeVirt 虚拟机无缝集成。此外还在开展工作，以便 CNI 插件与 multus 一起使用，从而更好地支持辅助网络。

  **Istio** - KubeVirt 贡献者向 Istio 提供了补丁，以便让 KubeVirt VM 开箱即用地与 Istio 集成。

  **ArgoCD** - KubeVirt 贡献者向 Argo 提供了补丁，以便与常见的 Argo 实践保持一致。

  **Tekton** - KubeVirt 维护了一组 Tekton 任务，以便轻松围绕 VM 构建 Tekton Pipeline。

  **Velero** - KubeVirt 贡献者将 Velero 集成进 KubeVirt，以便支持第三方备份厂商。

  **cluster-api-kubevirt** - Cluster API KubeVirt 构建于 KubeVirt 之上。

  **Kubernetes descheduler** - KubeVirt 社区成员向 Kubernetes descheduler 贡献了若干变更，以便让 descheduler 也能与 VM 无缝协作。

  **kubernetes-nmstate** - KubeVirt 社区成员向 kubernetes-nmstate 做了贡献，以提供一种声明式的宿主机网络配置方法——这是裸金属集群中的常见问题。

  **multus** - KubeVirt 利用 Multus API 来为 VM 实现辅助网络。

  以及其他项目，例如 Kubernetes 本身、AAQ、CRI-O。

### 设计 {#design}

* 说明本项目遵循的设计原则和最佳实践。

  KubeVirt 优先考虑与 Kubernetes 的集成以及 Kubernetes 的可用性体验。它利用 Kubernetes 原生 API、CRD（自定义资源定义）以及核心原语和模式，将 VM 作为一等公民来管理。

  以下是有关 KubeVirt 架构和指导原则的更多信息，尤其是 KubeVirt Razor：
  “如果某项能力对 Pod 有用，我们就不应当只针对 VM 实现它”。

* 概述或链接到项目的架构要求。说明它们在概念验证、开发、测试和生产环境中有何不同（如适用）。

  KubeVirt 适用于从单节点部署一直到拥有数百个节点的大规模 Kubernetes 集群。
  对于生产工作负载，主要要求是提供裸金属节点。

  全部细节见 [KubeVirt 安装指南](https://kubevirt.io/user-guide/cluster_admin/installation/)

* 定义项目在集群中所依赖的任何特定服务依赖。

  KubeVirt 要求存在 Kubernetes，除此之外别无其他。
  如果计划运行生产工作负载，集群应当拥有裸金属节点。

* 说明项目如何实现身份与访问管理。

  KubeVirt 利用 Kubernetes 原生的身份与访问管理（IAM）能力，来控制对与虚拟机（VM）相关的资源和操作的访问。

* 说明项目如何应对主权问题。

  KubeVirt 依赖 Kubernetes 社区，但在 Kubernetes SIG 之外运营一个独立社区。这包括但不限于其自身的 [SIG/子项目/工作组结构](https://github.com/kubevirt/community/blob/main/sig-list.md)。

* 说明项目所应对的任何合规要求。

  项目目前没有主动推进任何合规工作。
  不过，一些分发 KubeVirt 的厂商会这样做。

* 说明项目的高可用要求。

  KubeVirt 采用对控制平面资源进行副本复制的最佳实践来实现高可用。
  它也在支持 medik8s 项目，以便为其运行所依赖的 Kubernetes 集群增加高可用能力。

* 说明项目的资源要求，包括 CPU、网络和内存。

  KubeVirt 的资源要求取决于终端用户的规模要求。一个较好的估计是大约 1 个 CPU 和 1GB 内存。KubeVirt 的网络要求与 Kubernetes 相同。

  最终，资源要求取决于环境的规模。规模的若干维度包括节点数量、VM 数量和磁盘数量。
  KubeVirt 社区成员在单节点部署上运行 KubeVirt，也在拥有数百个节点的集群上运行。

* 说明项目的存储要求，包括它对临时存储和/或持久存储的使用。

  KubeVirt 与 Kubernetes 具有相同的存储要求。终端用户定义其工作负载需要何种存储，KubeVirt 将使用 Kubernetes API 来使用所定义的内容。

* 请概述项目的 API 设计

  KubeVirt 是由自定义资源定义所定义的 Kubernetes API 扩展。KubeVirt 的理念是复用 Kubernetes API 来定义基础设施。然后，KubeVirt 会把基础设施资源提供给虚拟机。

  * 说明项目的 API 拓扑与约定

    KubeVirt 的 API 是 RESTful 且声明式的，尽可能遵循 Kubernetes 所确立的先例。

  * 说明项目的默认值

    KubeVirt 提供配置默认值，允许用户“直接”部署 KubeVirt，而无需任何输入。

  * 概述为了合理使用本项目而需要在默认配置之外做出的任何额外配置

    KubeVirt 有一套在各 KubeVirt 组件之间共享的默认配置。

  * 说明因启用并使用本项目而产生的任何新的或变更的 API 类型与调用——包括对云提供商的调用

    大多数 KubeVirt API 是 Kubernetes API 扩展（CRD）。它们的文档位于 [KubeVirt API 参考](http://kubevirt.io/api-reference/)

  * 说明任何新的或变更的 API 与 API 服务器（包括 Kubernetes API 服务器）的兼容性

    兼容性说明见 [KubeVirt 与 Kubernetes 的兼容性](https://github.com/kubevirt/kubevirt/blob/main/docs/kubernetes-compatibility.md)

  * 说明任何新的或变更的 API 的版本管理，包括如何处理破坏性变更

    对于发行版，KubeVirt 遵循语义化版本——正如 Kubernetes 所做的那样。
    对于 CRD，KubeVirt 遵循 Kubernetes 的 API 版本管理标准，以确保其 API 的稳定性、兼容性和清晰的演进。
    KubeVirt API 使用 v1alpha1、v1beta1 和 v1 等标签进行版本管理——其中 alpha 被视为实验性，beta 更为稳定并具备一定的向后兼容性，v1 则是完全成熟的 API，具有强有力的向后兼容性保证。
    KubeVirt v1 API 保持向后兼容，在稳定 API 中会避免不向后兼容的变更。

* 说明项目的发布流程，包括主版本、次版本和补丁版本。

  发布流程描述于 [KubeVirt 发布指南](https://github.com/kubevirt/kubevirt/blob/main/docs/release.md)


### 安装 {#installation}

* 描述项目是如何安装和初始化的，例如用几行代码即可完成的最小安装，还是需要更复杂的集成与配置？

  KubeVirt 提供一个 Operator，用于部署和应用生命周期管理。此外，也可以通过从 kubevirt 和 containerized-data-importer 仓库安装关键组件来手动执行部署。这些都使用 `kubectl` 安装。

  然后还可以安装额外的可选网络插件。

  参考：https://kubevirt.io/user-guide/cluster_admin/installation/#installing-kubevirt-on-kubernetes

* 采用者如何测试并验证安装？

  采用者可以在其部署上运行 KubeVirt 测试套件。

### 安全 {#security}

* 请提供项目的云原生[安全自评估](https://github.com/cncf/tag-security/blob/main/community/assessments/guide/self-assessment.md)链接。

  该自评估正在进行中，跟踪于 [https://github.com/kubevirt/community/issues/335](https://github.com/kubevirt/community/issues/335)。
  一旦完成，本文档将更新并指向该自评估。

* 请查阅来自 TAG Security 的[云原生安全原则](https://github.com/cncf/tag-security/blob/main/security-whitepaper/secure-defaults-cloud-native-8.md)。
  * 你们如何满足云原生安全项目的各项原则？
    目前没有已经到位的流程来强制执行云原生安全原则。

  * 说明每一项云原生原则如何适用于你的项目。
    TBD
    可能的链接：[云原生原则](https://github.com/cloud-native-principles/cloud-native-principles/blob/master/cloud-native-principles.md)

  * 你建议用户如何更改安全默认值，以便“放宽”本项目的安全性？请链接到项目针对这些使用场景所写的任何文档。
    [KubeVirt 安全基础](https://kubevirt.io/2020/KubeVirt-Security-Fundamentals.html)

### 安全卫生 {#security-hygiene}

* 请描述项目用于维持项目基本健康与安全的框架、实践和流程。
  虽然没有用于确保安全实践的正式框架，但讨论安全性是 PR 评审过程的一部分。

* 说明项目如何评估：哪些功能如果得不到项目维护，将会对用户构成安全风险？
  KubeVirt 维护者承认，未得到维护的代码和功能对终端用户构成风险。
  为应对这一挑战而做出的一项结构性改变是：要求每一段代码都由某个 KubeVirt SIG 拥有并维护。
  如果某段代码未被某个 SIG 接受，它就不能合入 KubeVirt，因为它承受着变得无人维护的风险，从而构成安全风险。

### 云原生威胁建模 {#cloud-native-threat-modeling}

* 说明项目所需的最低限度权限，以及需要额外权限的原因。
  按照设计，KubeVirt VM 被设计为可在受限 PSL 下工作。
  还有一条设计原则：用户在运行 VM 时，不得获得相比于该用户所能运行的任何其他 Pod 而言更多的特权或权限。

  特殊权限仅授予基础设施组件，例如 virt-handler（节点代理）。

* 说明项目如何处理证书轮换，以及如何缓解与证书相关的任何问题。
  “在 KubeVirt 中，我们的 CA 和证书都会按照用户定义的重复间隔进行轮换。
  如果 CA 密钥或某张证书遭到泄露，这些信息最终都会变得过时且不可用，无论该泄露是否已被知晓。
  如果已知发生泄露，集群管理员只需删除 KubeVirt 安装命名空间中的相应 Secret，即可发起一次强制的 CA 和证书轮换。”
  - [KubeVirt 安全基础](https://kubevirt.io/2020/KubeVirt-Security-Fundamentals.html)

* 说明项目如何遵循并落实[安全软件供应链最佳实践](https://project.linuxfoundation.org/hubfs/CNCF_SSCP_v1.pdf)
  虽然社区目前没有正在进行的工作来落实 SSSC 最佳实践，但关于如何使其走上正轨的讨论反复出现。

## 第 1 天 - 安装与部署阶段 {#day-1-installation-and-deployment-phase}

### 项目安装与配置 {#project-installation-and-configuration}

* 描述项目的安装与配置是什么样的。

  相关内容描述于
  https://kubevirt.io/user-guide/cluster_admin/installation/#installing-kubevirt-on-kubernetes

### 项目启用与回滚 {#project-enablement-and-rollback}

* 如何在运行中的集群里启用或禁用本项目？请说明控制平面或节点需要的任何停机时间。

  在部署 KubeVirt Operator 之后，这就像部署 KubeVirt CR 一样简单——卸载时则将其移除。

* 说明启用本项目会如何改变集群或正在运行的工作负载的任何默认行为。

  KubeVirt 的安装为 Kubernetes 集群增加了虚拟化能力，允许用户管理和运行虚拟机。尽管默认行为和现有工作负载不应受到影响，但它会引入新的资源与组件以支持 VM。
  集群配置——即 API 服务器配置——本身不会被 KubeVirt 修改。

* 说明项目如何测试启用与禁用。

  安装和卸载 KubeVirt 是 KubeVirt 测试套件的一部分。

* 项目如何清理所创建的任何资源，包括 CRD？

  KubeVirt 使用 Operator 来进行应用生命周期管理。该 Operator 会移除所有与 KubeVirt 相关的组件。
  与 KubeVirt 相关的 CRD 通过 `kubectl delete -f kubevirt.yaml` 删除，所有关联资源都会被删除。

  所有细节记录于：https://kubevirt.io/user-guide/cluster_admin/updating_and_deletion/#deleting-kubevirt



### 推出、升级与回滚规划 {#rollout-upgrade-and-rollback-planning}

* 项目打算如何提供并维持与 Kubernetes 等基础设施和编排管理工具的兼容性，以及以何种频率？

  KubeVirt 有一份已定义的兼容性矩阵：
  https://github.com/kubevirt/kubevirt/blob/main/docs/kubernetes-compatibility.md

  此外，KubeVirt 本身就是一个云原生应用，完全以容器形式交付，从而确保与底层基础设施充分分离。

  https://kubevirt.io/user-guide/cluster_admin/updating_and_deletion/#updating-and-deletion

* 说明项目如何处理回滚流程。

  KubeVirt 被设计为只向前滚动。功能开发、缺陷修复和测试都与这一目标对齐。
  不过，从技术上讲，回滚是可能的。

* 推出或回滚如何会失败？说明对已在运行的工作负载的任何影响。

  正在运行的工作负载不会立即受到失败的推出或回滚的影响。工作负载——就像 Kubernetes 一样——会继续运行一段时间。
  取决于故障和环境，工作负载可能会继续长时间运行，或者最终受到集群中连带问题的影响。

* 说明应当提示进行回滚的任何具体指标。

  KubeVirt Operator 的 “Ready” 状况一直为 “false”。

* 说明升级和回滚是如何测试的，以及升级->降级->升级路径是如何测试的。

  升级测试是预提交测试的一部分。
  回滚以及升级 → 降级 → 升级测试并未覆盖。

* 说明项目如何告知用户功能和 API 的弃用与移除。

  KubeVirt 目前使用 API v1，并一直严格承诺不移除现有的 API 构造。任何曾被考虑移除的功能，都被限制为需要选择启用的功能门控。任何此类功能的移除，都会提前很久在 KubeVirt 开发邮件列表上讨论。

* 说明项目如何允许在推出过程中使用 alpha 和 beta 能力。

  KubeVirt 的功能生命周期采用 Kubernetes 模型。功能最初使用 API 中的 alpha 版本字符串来实现，并受功能门控约束。功能随后毕业进入 beta，并作为发布生命周期的一部分移除功能门控。

## 第 2 天 - 日常运维阶段 {#day-2-day-to-day-operations-phase}

### 可扩展性/可靠性 {#scalabilityreliability}

* 说明项目如何增加现有 API 对象的大小或数量。

  KubeVirt 是一个 Kubernetes 云原生应用，在运行时利用核心 Kube 实体。
  虽然确切数字取决于集群规模，但大致是数十个 Pod、数十个 Secret、若干 Deployment 以及一个 DaemonSet。

  总体而言，与预期运行 KubeVirt 的集群相比，KubeVirt 被认为占用较小。

  根据正在运行的 VM 数量，会产生一定的 etcd 负载。

* 说明项目如何定义服务等级目标（SLO）和服务等级指标（SLI）。

  KubeVirt 没有描述任何 SLO 或 SLI。


* 说明任何会使现有 SLI/SLO 所覆盖的耗时增加的操作。

  N/A

* 说明因启用本项目而导致任何组件资源使用增长的情况，包括 CPU、内存、存储、吞吐量。

  资源使用会增加。
  取决于集群和工作负载的规模，建议维护专用的基础设施节点，或者只是在集群上保留足够的空闲容量。

* 说明在哪些条件下，启用/使用本项目会导致某些节点资源（PID、套接字、inode 等）耗尽。


  每个 VM 都运行在它自己的 Pod 中。因此，通常所有与 Pod 相关的限制也同样适用于 VM。

* 说明已对项目执行的负载测试及其结果。

  KubeVirt 项目作为 SIG scale 工作的一部分，定期进行性能与负载测试。

  像 Red Hat 这样的厂商会在其产品化过程中定期进行性能与规模测试。

* 说明对用户、请求、系统资源等的建议限制，以及这些限制是如何得出的。

  对 VM 数量或节点数量的限制不仅由本项目决定，也取决于底层集群的运维方式。

  高度优化的组织可以轻松地在拥有数百个节点的单个集群上运行数万个 VM。
  建议普通用户根据运维模型和要求来调整其集群规模。

  一个常规的大型 KubeVirt 集群预期在不到 100 个节点上承载数千个 VM。

* 说明项目使用哪种弹性模式以及如何使用，包括熔断器模式。

### 可观测性要求 {#observability-requirements}

* 说明项目正在使用或产生的信号，包括日志、指标、性能剖析和追踪。请包括支持的格式、建议的配置以及数据存储。

  KubeVirt 提供指标端点，并附带包含运维手册的 Prometheus 告警规则。
  KubeVirt 也在其 CR 上一致地使用错误状况。

* 说明项目如何捕获审计日志。

  Kubernetes 审计日志可用于 KubeVirt，因为所有面向用户的 API 调用都经过 Kubernetes API 服务器。

* 说明项目使用或实现的任何仪表板，以及任何仪表板要求。

  KubeVirt 没有实现任何仪表板。
  有一些由社区维护的 Grafana 仪表板可用。

* 说明项目如何向采用者呈现项目的资源要求，以便监控云与基础设施成本，例如 FinOps

  采用者只需继续监控其 Kubernetes 用量。

* 项目覆盖了哪些参数，以确保应用/服务及其工作负载的健康？

  KubeVirt 允许为工作负载设置健康检查。
  KubeVirt Operator 会检查其基础设施组件的健康状况。
  KubeVirt Operator 的健康状况（包括其依赖项）也会通过 Prometheus 端点上报。

* 运维人员如何确定本项目是否正被工作负载使用？

  KubeVirt 提供运行虚拟机工作负载的能力。这些工作负载运行在 Pod 中。因此，用于监控 Pod 的同一机制也可用于监控 VM。

* 使用本项目的人如何知道它对其实例而言正在正常工作？

  他们可以查看 KubeVirt Operator 的状况。如果它没有正常工作，则会设置一个状况。

* 说明本项目的 SLO（服务等级目标）。

  没有定义 SLO。

* 运维人员可以使用哪些 SLI（服务等级指标）来判断服务的健康状况？

  没有定义 SLI。

### 依赖 {#dependencies}

* 说明项目在集群中所依赖的具体运行中的服务。

  KubeVirt 仅依赖 Kubernetes，没有其他强制性依赖。KubeVirt 可以选择与 Containerized Data Importer 和 nmstate 项目协同工作。

* 说明项目的依赖生命周期策略。

  KubeVirt 使用 golang 编写，并使用 go.mod/go.sum 来跟踪加入构建链的依赖。每个新增依赖都会在代码评审/提交过程中受到审查。

* 项目如何在其开发和安全卫生中纳入并考虑源成分分析？说明这种源成分分析（SCA）是如何跟踪的。

  KubeVirt 使用 FOSSA 来确保符合安全与许可证要求。这些检查会应用于本项目的每一个拉取请求。

* 说明项目如何基于源成分分析（SCA）实施变更，以及时间尺度。

  FOSSA 检查应用于每一个拉取请求。此外，KubeVirt 会定期利用 SonarCloud 来检测常见的问题代码模式。

### 故障排查 {#troubleshooting}

* 如果某个关键组件或功能变得不可用，本项目如何恢复？例如 Kubernetes API 服务器、etcd、数据库、leader 节点等。

  如果集群控制平面丢失，工作负载将继续运行。它们会尝试定期重新连接集群控制平面。


* 说明已知的故障模式。

  集群 API 不可用：工作负载继续运行。KubeVirt 控制平面最终会恢复。
  节点/kubelet 不可用：工作负载会停止。如果可能，KubeVirt 会将工作负载重新调度到其他节点。

### 安全 {#security_1}

* 安全卫生
  * 项目如何执行访问控制？

    KubeVirt 依赖 Kubernetes 的访问控制，自身并不执行访问控制。

* 云原生威胁建模

  尚未对 KubeVirt 进行云原生威胁建模。

  * 项目如何确保其安全报告与响应团队能够代表其社区多样性（组织层面和个人层面）？

    KubeVirt 维护者会尽力让多个组织参与安全报告、安全响应团队以及审计工作。

  * 项目如何邀请和轮换安全报告团队成员？

    项目依靠厂商来执行这一轮换。
