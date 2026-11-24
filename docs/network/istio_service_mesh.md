# Istio 服务网格 {#istio-service-mesh}

服务网格可以监控、可视化并控制 Pod 之间的流量。KubeVirt 支持把 VM 作为 Istio 服务网格的一部分运行。

## 限制 {#limitations}

- Istio 服务网格仅支持 Pod 网络的 masquerade 或 passt 绑定。

- Istio 会把[一组端口](https://istio.io/latest/docs/ops/deployment/requirements/#ports-used-by-istio)用于自身目的，这些端口不得在 VMI 接口中显式指定。

- Istio 仅支持 IPv4。

## 前提条件 {#prerequisites}

- 本指南假定 Istio 已经部署，并且使用 Istio CNI 插件。更多信息请参阅 [Istio 文档](https://istio.io/latest/docs/)。

- 可选：用于故障排查的 `istioctl` 二进制文件。请参阅 Istio [安装说明](https://istio.io/latest/docs/setup/getting-started/)。

- 创建 VM 的目标命名空间必须带有 `istio-injection=enabled` 标签。

- 如果使用 Multus 管理 CNI，则应用命名空间中需要如下 `NetworkAttachmentDefinition`：
```yaml
apiVersion: "k8s.cni.cncf.io/v1"
kind: NetworkAttachmentDefinition
metadata:
  name: istio-cni
```

## 创建启用了 Istio 代理注入的 VirtualMachineInstance {#create-a-virtualmachineinstance-with-enabled-istio-proxy-injecton}

下面的示例指定了一个使用 masquerade 网络接口的 VMI，以及用于把 VM 注册到服务网格的 `sidecar.istio.io/inject` 注解。

```yaml
apiVersion: kubevirt.io/v1
kind: VirtualMachineInstance
metadata:
  annotations:
    sidecar.istio.io/inject: "true"
  labels:
    app: vmi-istio
  name: vmi-istio
spec:
  domain:
    devices:
      interfaces:
        - name: default
          masquerade: {}
      disks:
        - disk:
            bus: virtio
          name: containerdisk
    resources:
      requests:
        memory: 1024M
  networks:
    - name: default
      pod: {}
  terminationGracePeriodSeconds: 0
  volumes:
    - name: containerdisk
      containerDisk:
        image: registry:5000/kubevirt/fedora-cloud-container-disk-demo:devel
```

Istio 期望每个应用至少关联一个 Kubernetes Service。请创建下面这个暴露 8080 端口的 Service：

```yaml
apiVersion: v1
kind: Service
metadata:
  name: vmi-istio
spec:
  selector:
    app: vmi-istio
  ports:
    - port: 8080
      name: http
      protocol: TCP
```

**注意：** 每个启用了 Istio 的 VMI 都必须带有 `sidecar.istio.io/inject` 注解，以指示 KubeVirt 执行必要的网络配置。

## 验证 {#verification}

使用 `istioctl proxy-status` 命令，验证 istio-proxy 边车已经部署，并且能够与 Istio 控制平面同步。关于 `proxy-status` 子命令的更多信息，请参阅 Istio 文档中的[调试 Envoy 和 Istiod](https://istio.io/latest/docs/ops/diagnostic-tools/proxy-cmd/)一节。

```shell
$ kubectl get pods
NAME                             READY   STATUS    RESTARTS   AGE
virt-launcher-vmi-istio-ncx7r    3/3     Running   0          7s

$ kubectl get pods virt-launcher-vmi-istio-ncx7r -o jsonpath='{.spec.containers[*].name}'
compute volumecontainerdisk istio-proxy

$ istioctl proxy-status
NAME                                    CDS        LDS        EDS        RDS          ISTIOD                      VERSION
...
virt-launcher-vmi-istio-ncx7r.default   SYNCED     SYNCED     SYNCED     SYNCED       istiod-7c4d8c7757-hshj5     1.10.0
```

## 故障排查 {#troubleshooting}

### Istio 边车未部署 {#istio-sidecar-is-not-deployed}

```shell
$ kubectl get pods
NAME                             READY   STATUS    RESTARTS   AGE
virt-launcher-vmi-istio-jnw6p    2/2     Running   0          37s

$ kubectl get pods virt-launcher-vmi-istio-jnw6p -o jsonpath='{.spec.containers[*].name}'
compute volumecontainerdisk
```

**解决办法：** 请确认已向目标命名空间添加 `istio-injection=enabled`。如果问题仍然存在，请查阅 [Istio 文档的相关部分](https://istio.io/latest/docs/ops/configuration/mesh/injection-concepts/)。

### Istio 边车未就绪 {#istio-sidecar-is-not-ready}
```shell
$ kubectl get pods
NAME                             READY   STATUS    RESTARTS   AGE
virt-launcher-vmi-istio-lg5gp    2/3     Running   0          90s

$ kubectl describe pod virt-launcher-vmi-istio-lg5gp
  ...
  Warning  Unhealthy  2d8h (x3 over 2d8h)  kubelet            Readiness probe failed: Get "http://10.244.186.222:15021/healthz/ready": dial tcp 10.244.186.222:15021: connect: no route to host
  Warning  Unhealthy  2d8h (x4 over 2d8h)  kubelet            Readiness probe failed: Get "http://10.244.186.222:15021/healthz/ready": context deadline exceeded (Client.Timeout exceeded while awaiting headers)
```

**解决办法：** 请确认所创建的 VMI 中定义了 `sidecar.istio.io/inject: "true"` 注解，并且 Pod 网络接口使用了 masquerade 或 passt 绑定。

### VMI 的 virt-launcher Pod 停在初始化阶段 {#virt-launcher-pod-for-vmi-is-stuck-at-initialization-phase}
```shell
$ kubectl get pods
NAME                             READY   STATUS     RESTARTS   AGE
virt-launcher-vmi-istio-44mws    0/3     Init:0/3   0          29s

$ kubectl describe pod virt-launcher-vmi-istio-44mws
  ...
  Multus: [default/virt-launcher-vmi-istio-44mws]: error loading k8s delegates k8s args: TryLoadPodDelegates: error in getting k8s network for pod: GetNetworkDelegates: failed getting the delegate: getKubernetesDelegate: cannot find a network-attachment-definition (istio-cni) in namespace (default): network-attachment-definitions.k8s.cni.cncf.io "istio-cni" not found

```

**解决办法：** 请确认已在目标命名空间中创建 `istio-cni` NetworkAttachmentDefinition（见[前提条件](#prerequisites)一节）。
