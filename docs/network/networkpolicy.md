# NetworkPolicy {#networkpolicy}

创建 NetworkPolicy 对象之前，请确认你使用的网络方案支持 NetworkPolicy。网络隔离完全由 NetworkPolicy 对象控制。默认情况下，命名空间中的所有 VMI 都可以被其他 VMI 和网络端点访问。要隔离项目中的一个或多个 VMI，可以在该命名空间中创建 NetworkPolicy 对象，以标明允许的入站连接。

> 注意：网络策略对 VMI 和 Pod 一视同仁，因为标签会传递到承载正在运行的 VMI 的 Pod。换句话说，VMI 上的标签可以被策略中的 `spec.podSelector` 匹配。

## 创建用于拒绝全部流量的 NetworkPolicy {#create-networkpolicy-to-deny-all-traffic}

要使项目“默认拒绝”，请添加一个匹配所有 VMI、但不接受任何流量的 NetworkPolicy 对象。

```yaml
kind: NetworkPolicy
apiVersion: networking.k8s.io/v1
metadata:
  name: deny-by-default
spec:
  podSelector: {}
  ingress: []
```

## 创建仅接受来自命名空间内 VMI 连接的 NetworkPolicy {#create-networkpolicy-to-only-accept-connections-from-vmis-within-namespaces}

要使 VMI 接受来自同一命名空间内其他 VMI 的连接，并拒绝来自其他命名空间中 VMI 的所有其他连接：

```yaml
kind: NetworkPolicy
apiVersion: networking.k8s.io/v1
metadata:
  name: allow-same-namespace
spec:
  podSelector: {}
  ingress:
  - from:
    - podSelector: {}
```

## 创建仅允许 HTTP 和 HTTPS 流量的 NetworkPolicy {#create-networkpolicy-to-only-allow-http-and-https-traffic}

要只允许对 VMI 进行 HTTP 和 HTTPS 访问，请添加类似如下的 NetworkPolicy 对象：

```yaml
kind: NetworkPolicy
apiVersion: networking.k8s.io/v1
metadata:
  name: allow-http-https
spec:
  podSelector: {}
  ingress:
  - ports:
    - protocol: TCP
      port: 8080
    - protocol: TCP
      port: 8443
```

## 创建按标签拒绝流量的 NetworkPolicy {#create-networkpolicy-to-deny-traffic-by-labels}

要使带有标签 `type: test` 的某一个特定 VMI 拒绝来自其他 VMI 的全部流量，请创建：

```yaml
kind: NetworkPolicy
apiVersion: networking.k8s.io/v1
metadata:
  name: deny-by-label
spec:
  podSelector:
    matchLabels:
      type: test
  ingress: []
```

Kubernetes NetworkPolicy 文档见此处：[Kubernetes NetworkPolicy](https://kubernetes.io/docs/concepts/services-networking/network-policies/)
