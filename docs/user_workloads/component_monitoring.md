# 组件监控 {#component-monitoring}

所有 KubeVirt 系统组件都在其 `/metrics` REST 端点上暴露 Prometheus 指标。

完整且最新的指标列表见 [kubevirt/monitoring](https://github.com/kubevirt/monitoring/blob/main/docs/metrics.md)。

## 自定义服务发现 {#custom-service-discovery}

Prometheus 开箱即支持基于 [Pod](https://prometheus.io/docs/prometheus/latest/configuration/configuration/#pod) 和 [端点](https://prometheus.io/docs/prometheus/latest/configuration/configuration/#endpoints) 的服务发现。两者都可用于发现 KubeVirt 服务。

所有暴露指标的 Pod 都带有标签 `prometheus.kubevirt.io`，并包含一个名为 `metrics` 的端口定义。在 KubeVirt 发布清单中，默认的 `metrics` 端口是 `8443`。

上述标签和端口信息由 `Service`（名为 `kubevirt-prometheus-metrics`）收集。Kubernetes 会自动创建一个同名的对应 `Endpoint`：

```bash
kubectl get endpoints -n kubevirt kubevirt-prometheus-metrics -o yaml
```

```yaml
apiVersion: v1
kind: Endpoints
metadata:
  labels:
    kubevirt.io: ""
    prometheus.kubevirt.io: ""
  name: kubevirt-prometheus-metrics
  namespace: kubevirt
subsets:
- addresses:
  - ip: 10.244.0.5
    nodeName: node01
    targetRef:
      kind: Pod
      name: virt-handler-cjzg6
      namespace: kubevirt
      resourceVersion: "4891"
      uid: c67331f9-bfcf-11e8-bc54-525500d15501
  - ip: 10.244.0.6
  [...]
  ports:
  - name: metrics
    port: 8443
    protocol: TCP
```

通过监视此 endpoint，观察 `subsets.addresses` 中新增和移除的 IP，并附上 `metrics` 端口（来自 `subsets.ports`），就可以始终得到一份完整的、可供抓取的 Prometheus 目标列表。

## 与 prometheus-operator 集成 {#integrating-with-the-prometheus-operator}

[prometheus-operator](https://github.com/coreos/prometheus-operator) 可以利用 `kubevirt-prometheus-metrics` 服务自动创建相应的 Prometheus 配置。

KubeVirt 的 `virt-operator` 在为部署创建安装策略时，会检查 `ServiceMonitor` 自定义资源是否存在。KubeVirt 会自动创建 `ServiceMonitor` 资源，并将其放在 `monitorNamespace` 中，同时在 KubeVirt 的命名空间中创建相应的 role 和 rolebinding。

`KubeVirt` 自定义资源中暴露了三项设置，用于指导 KubeVirt 正确创建这些资源：

-   `monitorNamespace`：prometheus-operator 运行所在的命名空间。默认为 `openshift-monitoring`。

-   `monitorAccount`：prometheus-operator 运行所用的 serviceAccount。默认为 `prometheus-k8s`。

-   `serviceMonitorNamespace`：serviceMonitor 运行所在的命名空间。默认是 `monitorNamespace` 

请注意，如果决定设置 `serviceMonitorNamespace`，则此命名空间必须包含在 Prometheus spec 的 `serviceMonitorNamespaceSelector` 字段中。

如果给定部署的 prometheus-operator 使用这些默认值，则可以省略这些值。

展示这些默认值的 KubeVirt 资源示例如下：

```yaml
apiVersion: kubevirt.io/v1
kind: KubeVirt
metadata:
  name: kubevirt
spec:
  monitorNamespace: openshift-monitoring
  monitorAccount: prometheus-k8s
```
## 与 OKD cluster-monitoring-operator 集成 {#integrating-with-the-okd-cluster-monitoring-operator}

在 [cluster-monitoring-operator](https://github.com/openshift/cluster-monitoring-operator) 启动并运行之后，KubeVirt 会检测到 `ServiceMonitor` 资源的存在。由于该定义包含 `openshift.io/cluster-monitoring` 标签，集群监控会自动发现它。

## 关于虚拟机的指标 {#metrics-about-virtual-machines}

这些 endpoint 报告与虚拟机运行时行为相关的指标。所有相关指标都以 `kubevirt_vmi` 为前缀。

这些指标带有标签，可以把它们关联到所指向的 VMI 对象。这些标签至少会暴露相关 VMI 对象的 `node`、`name` 和 `namespace`。

例如，报告的指标可能如下所示

```
kubevirt_vmi_memory_resident_bytes{domain="default_vm-test-01",name="vm-test-01",namespace="default",node="node01"} 2.5595904e+07
kubevirt_vmi_network_traffic_bytes_total{domain="default_vm-test-01",interface="vnet0",name="vm-test-01",namespace="default",node="node01",type="rx"} 8431
kubevirt_vmi_network_traffic_bytes_total{domain="default_vm-test-01",interface="vnet0",name="vm-test-01",namespace="default",node="node01",type="tx"} 1835
kubevirt_vmi_vcpu_seconds_total{domain="default_vm-test-01",id="0",name="vm-test-01",namespace="default",node="node01",state="1"} 19
```

请注意上面示例中的 `domain` 标签。该标签已弃用，并将在未来的发行版中移除。你应当改用 `node`、`namespace`、`name` 标签来标识 VMI。

## 重要查询 {#important-queries}

### 检测 REST 客户端的连接问题 {#detecting-connection-issues-for-the-rest-client}

使用以下查询，获取表明连接问题的所有 REST 调用的计数器：

    rest_client_requests_total{code="<error>"}

如果该计数器持续增长，则表明相应的 KubeVirt 组件在连接 apiserver 时存在普遍问题
