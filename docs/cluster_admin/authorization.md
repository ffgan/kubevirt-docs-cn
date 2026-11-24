# 授权 {#authorization}

KubeVirt 授权通过 Kubernetes 的基于资源的授权控制体系
（RBAC）完成。RBAC 允许集群管理员通过将 RBAC 角色绑定到用户，
授予对集群资源的访问权限。

例如，管理员创建一个 RBAC 角色，表示
创建 VirtualMachineInstance 所需的权限。管理员随后
可以将该角色绑定到用户，从而授予他们
启动 VirtualMachineInstance 所需的权限。

借助 RBAC 角色，管理员可以向用户授予针对各种
KubeVirt 功能的针对性访问权限。

## KubeVirt 默认 RBAC ClusterRole {#kubevirt-default-rbac-clusterroles}

KubeVirt 附带一组预定义的 RBAC ClusterRole，可用于
向用户授予访问 KubeVirt 资源的权限。

### 默认查看角色 {#default-view-role}

**kubevirt.io:view** ClusterRole 授予用户查看集群中所有
KubeVirt 资源的权限。此角色不包含创建、删除、
修改任何 KubeVirt 资源的权限，也不包含查看资源
spec 之外的访问权限。这意味着拥有此角色的用户
可以看到某个 VirtualMachineInstance 正在运行，但不能关闭它，
也不能通过控制台/VNC 访问该 VirtualMachineInstance。

### 默认编辑角色 {#default-edit-role}

**kubevirt.io:edit** ClusterRole 授予用户修改
集群中所有 KubeVirt 资源的权限。例如，拥有此
角色的用户可以创建新的 VirtualMachineInstance、删除
VirtualMachineInstance，并同时访问控制台和 VNC。

### 默认管理员角色 {#default-admin-role}

**kubevirt.io:admin** ClusterRole 授予用户对
所有 KubeVirt 资源的完整权限，包括删除资源
集合的能力。

管理员角色还授予用户查看和修改 KubeVirt
运行时配置的权限。此配置位于 KubeVirt operator 正在运行的命名空间中的 Kubevirt 自定义资源的
`configuration` 键下。

> *注意* 只有在使用 ClusterRoleBinding 时，才能保证用户
> 可以修改 kubevirt
> 运行时配置。RoleBinding
> 只有在
> 其目标命名空间与 kubevirt CR 所在命名空间相同时，才能提供对 kubevirt CR 的访问。

### 将默认 ClusterRole 绑定到用户 {#binding-default-clusterroles-to-users}

通过创建
ClusterRoleBinding 或 RoleBinding 对象，将 KubeVirt 默认 ClusterRole 授予用户。

#### 在所有命名空间内绑定 {#binding-within-all-namespaces}

使用 ClusterRoleBinding 时，用户会在所有命名空间中获得该
角色所授予的权限。

#### 在单个命名空间内绑定 {#binding-within-single-namespace}

使用 RoleBinding 时，用户仅在目标命名空间内获得该角色
所授予的权限。

## RBAC 角色聚合 {#rbac-role-aggregation}

KubeVirt 使用 Kubernetes 的
[聚合 ClusterRole](https://kubernetes.io/docs/reference/access-authn-authz/rbac/#aggregated-clusterroles)
功能，自动用 KubeVirt 权限扩展默认的 Kubernetes 角色（`admin`、
`edit`、`view`）。这意味着默认情况下，
任何已经拥有这些标准 Kubernetes 角色之一的用户
都会在其命名空间内自动获得相应的 KubeVirt 权限，
无需任何额外配置。

KubeVirt ClusterRole 与 Kubernetes 默认角色之间的映射
如下：

| KubeVirt ClusterRole | 聚合到的 Kubernetes 角色 |
|----------------------|-------------------------------|
| `kubevirt.io:admin`  | `admin`                       |
| `kubevirt.io:edit`   | `edit`                        |
| `kubevirt.io:view`   | `view`                        |

此聚合由 KubeVirt ClusterRole 上的 `aggregate-to-*` 标签控制（例如
`rbac.authorization.k8s.io/aggregate-to-admin: "true"`）。

### 控制角色聚合策略 {#controlling-role-aggregation-strategy}

> *注意* 此功能处于 Alpha 阶段，需要 `OptOutRoleAggregation`
> 功能门控。从 KubeVirt v1.8 起可用。

虽然自动 RBAC 聚合对许多部署来说很方便，
注重安全的环境可能需要对
权限进行更细粒度的控制。KubeVirt 在
KubeVirt CR 配置中提供 `roleAggregationStrategy` 字段，允许集群管理员
选择退出自动聚合。

`roleAggregationStrategy` 字段接受两个值：

- **`AggregateToDefault`**（默认）：KubeVirt ClusterRole
  会聚合到默认的 Kubernetes 角色。拥有标准
  `admin`、`edit` 或 `view` 角色的用户会自动获得
  相应的 KubeVirt 权限。未设置该字段时，
  这就是默认行为。

- **`Manual`**：KubeVirt ClusterRole **不会**聚合到
  默认的 Kubernetes 角色。必须通过专用的 RoleBinding 或 ClusterRoleBinding
  显式授予用户 KubeVirt 权限。

#### 启用手动角色聚合 {#enabling-manual-role-aggregation}

要使用 `Manual` 策略，请先启用 `OptOutRoleAggregation`
功能门控，然后将 `roleAggregationStrategy` 设置为 `Manual`：

```yaml
apiVersion: kubevirt.io/v1
kind: KubeVirt
metadata:
  name: kubevirt
  namespace: kubevirt
spec:
  configuration:
    developerConfiguration:
      featureGates:
        - OptOutRoleAggregation
    roleAggregationStrategy: Manual
```


有关启用功能门控的更多
细节，请参阅[激活功能门控](activating_feature_gates.md)。

> *注意* 将 `roleAggregationStrategy` 设置为 `Manual` 时，如果未启用
> `OptOutRoleAggregation` 功能门控，会被
> 准入 webhook 拒绝。值 `AggregateToDefault`（或将该
> 字段留空）不需要该功能门控。

#### 重新启用聚合 {#re-enabling-aggregation}

要恢复默认的聚合行为，请将 `roleAggregationStrategy`
设置为 `AggregateToDefault`，或完全移除该字段：

```yaml
apiVersion: kubevirt.io/v1
kind: KubeVirt
metadata:
  name: kubevirt
  namespace: kubevirt
spec:
  configuration:
    roleAggregationStrategy: AggregateToDefault
```


该变更会在下一次协调周期生效，无需
重新安装 KubeVirt。

#### 行为摘要 {#behavior-summary}

| 配置 | 全新安装 | 现有安装 |
|---------------|---------------|------------------|
| `AggregateToDefault` 或未设置 | 创建时带有聚合标签的 ClusterRole | 若聚合标签先前被移除，则会恢复 |
| `Manual`（启用功能门控） | 创建时不带聚合标签的 ClusterRole | 从现有 ClusterRole 上移除聚合标签 |

#### 使用手动策略授予权限 {#granting-permissions-with-manual-strategy}

当 `roleAggregationStrategy` 设置为 `Manual` 时，拥有
标准 Kubernetes `admin`、`edit` 或 `view` 角色的用户**不会**
自动获得 KubeVirt 权限。相反，集群
管理员必须把 KubeVirt ClusterRole 显式绑定到用户
或组。

**在单个命名空间中授予完整的 KubeVirt 管理员权限：**

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: kubevirt-admin
  namespace: my-namespace
subjects:
  - kind: User
    name: jane
    apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: ClusterRole
  name: kubevirt.io:admin
  apiGroup: rbac.authorization.k8s.io
```


**授予跨所有命名空间的只读访问权限：**

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: kubevirt-viewers
subjects:
  - kind: Group
    name: vm-viewers
    apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: ClusterRole
  name: kubevirt.io:view
  apiGroup: rbac.authorization.k8s.io
```


**在特定命名空间中向某个组授予编辑权限：**

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: kubevirt-editors
  namespace: dev-team
subjects:
  - kind: Group
    name: developers
    apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: ClusterRole
  name: kubevirt.io:edit
  apiGroup: rbac.authorization.k8s.io
```


> *注意* 无论 `roleAggregationStrategy` 如何设置，所有
> 已认证用户都保留对 KubeVirt CR 本身的读取权限
> （`get`、`list`，针对 `kubevirts.kubevirt.io`）。这由
> **kubevirt.io:default** ClusterRole 授予，该角色绑定到
> `system:authenticated` 组，并且不受角色聚合影响。

## 创建自定义 RBAC 角色 {#creating-custom-rbac-roles}

如果默认的 KubeVirt ClusterRole 表达能力不足，管理员
可以创建自己的自定义 RBAC 角色，以向用户授予对 KubeVirt
资源的访问权限。创建 RBAC 角色时只能授予访问权限，
也就是说无法拒绝访问。访问权限只能被授予。

下面是 KubeVirt 默认管理员 ClusterRole 的示例。
可以通过减少此示例角色中的权限来创建自定义 RBAC 角色。

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: my-custom-rbac-role
  labels:
    kubevirt.io: ""
rules:
  - apiGroups:
      - subresources.kubevirt.io
    resources:
      - virtualmachineinstances/console
      - virtualmachineinstances/vnc
    verbs:
      - get
  - apiGroups:
      - kubevirt.io
    resources:
      - virtualmachineinstances
      - virtualmachines
      - virtualmachineinstancepresets
      - virtualmachineinstancereplicasets
    verbs:
      - get
      - delete
      - create
      - update
      - patch
      - list
      - watch
      - deletecollection
```

