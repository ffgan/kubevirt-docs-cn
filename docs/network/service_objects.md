# Service 对象 {#service-objects}

VirtualMachineInstance 启动之后，为了连接到 VirtualMachineInstance，可以为它创建一个 `Service` 对象。目前支持三种 Service：`ClusterIP`、`NodePort` 和 `LoadBalancer`。默认类型是 `ClusterIP`。

> **注意**：VirtualMachineInstance 上的标签会传递到 Pod，因此只需把用于创建 Service 的标签加到 VirtualMachineInstance 上。此后的做法与暴露任何其他 k8s 资源相同，即在 Service 中引用这些标签。


## 将 VirtualMachineInstance 暴露为 ClusterIP Service {#expose-virtualmachineinstance-as-a-clusterip-service}

给定一个带有标签 `special: key` 的 VirtualMachineInstance：

```yaml
apiVersion: kubevirt.io/v1
kind: VirtualMachineInstance
metadata:
  name: vmi-ephemeral
  labels:
    special: key
spec:
  domain:
    devices:
      disks:
      - disk:
          bus: virtio
        name: containerdisk
    resources:
      requests:
        memory: 64M
  volumes:
  - name: containerdisk
    containerDisk:
      image: kubevirt/cirros-registry-disk-demo:latest
```

可以通过创建 `ClusterIP` Service 来暴露其 SSH 端口（22）：

```yaml
apiVersion: v1
kind: Service
metadata:
  name: vmiservice
spec:
  ports:
  - port: 27017
    protocol: TCP
    targetPort: 22
  selector:
    special: key
  type: ClusterIP
```

创建这个 `ClusterIP` Service 时，使用 `kubectl`：

    $ kubectl create -f vmiservice.yaml

也可以使用 `virtctl` 命令来暴露该 VirtualMachineInstance：

    $ virtctl expose virtualmachineinstance vmi-ephemeral --name vmiservice --port 27017 --target-port 22

说明：\* 如果未设置 `--target-port`，它将取与 `--port` 相同的值 \* 集群 IP 通常会自动分配，但也可以使用 `--cluster-ip` 标志强制指定一个值（假定该值处于有效范围且未被占用）

查询该 Service 对象：

    $ kubectl get service
    NAME        TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)     AGE
    vmiservice   ClusterIP   172.30.3.149   <none>        27017/TCP   2m

可以在集群网络内部，通过 Service IP 和 Service 端口连接到该 VirtualMachineInstance：

    $ ssh cirros@172.30.3.149 -p 27017


## 将 VirtualMachineInstance 暴露为 NodePort Service {#expose-virtualmachineinstance-as-a-nodeport-service}

要通过 Kubernetes Service（例如 NodePort、ClusterIP、LoadBalancer）暴露 VirtualMachineInstance（VMI），请确保关联的 VirtualMachine（VM）模板包含相应的标签。这些标签会被 VMI 所创建的底层 Pod 继承，从而使该 Service 能够通过其选择器定位到正确的 Pod。

更多细节请参阅 Kubernetes 关于 [Service 与选择器](https://kubernetes.io/docs/concepts/services-networking/service/#services-with-selectors) 的文档。

通过创建 `NodePort` Service，暴露运行于 KubeVirt 上的 VirtualMachineInstance 的 SSH 端口（22）：

```yaml
apiVersion: v1
kind: Service
metadata:
  name: nodeport
spec:
  externalTrafficPolicy: Cluster
  ports:
  - name: nodeport
    nodePort: 30000
    port: 27017
    protocol: TCP
    targetPort: 22
  selector:
    special: key
  type: NodePort
```

创建这个 `NodePort` Service 时，使用 `kubectl`：

    $ kubectl create -f nodeport.yaml

也可以使用 `virtctl` 命令来暴露该 VirtualMachineInstance：

    $ virtctl expose virtualmachineinstance vmi-ephemeral --name nodeport --type NodePort --port 27017 --target-port 22 --node-port 30000

说明：\* 如果未设置 `--node-port`，其值会被动态分配（范围为 30000 以上）\* 如果设置了 `--node-port`，该值必须在所有 Service 中保持唯一

可以通过查询 Service 对象来列出该 Service：

    $ kubectl get service
    NAME           TYPE       CLUSTER-IP      EXTERNAL-IP   PORT(S)           AGE
    nodeport       NodePort   172.30.232.73   <none>        27017:30000/TCP   5m

在集群网络外部，使用节点 IP 和节点端口连接到该 VirtualMachineInstance：

    $ ssh cirros@$NODE_IP -p 30000


## 将 VirtualMachineInstance 暴露为 LoadBalancer Service {#expose-virtualmachineinstance-as-a-loadbalancer-service}

通过创建 `LoadBalancer` Service，暴露运行于 KubeVirt 上的 VirtualMachineInstance 的 RDP 端口（3389）。示例如下：

```yaml
apiVersion: v1
kind: Service
metadata:
  name: lbsvc
spec:
  externalTrafficPolicy: Cluster
  ports:
  - port: 27017
    protocol: TCP
    targetPort: 3389
  selector:
    special: key
  type: LoadBalancer
```

创建这个 `LoadBalancer` Service 时，使用 `kubectl`：

    $ kubectl create -f lbsvc.yaml

也可以使用 `virtctl` 命令来暴露该 VirtualMachineInstance：

    $ virtctl expose virtualmachineinstance vmi-ephemeral --name lbsvc --type LoadBalancer --port 27017 --target-port 3389

请注意，可以使用 `--external-ip` 标志把该 Service 的外部 IP 强制设为某个值（不会对该值做校验）。

可以通过查询 Service 对象来列出该 Service：

    $ kubectl get svc
    NAME      TYPE           CLUSTER-IP       EXTERNAL-IP                   PORT(S)           AGE
    lbsvc     LoadBalancer   172.30.27.5      172.29.10.235,172.29.10.235   27017:31829/TCP   5s

使用 `vinagre` 客户端，通过公共 IP 和端口连接到你的 VirtualMachineInstance。

请注意，这里的外部端口（31829）是动态分配的。

## Service 生命周期 {#service-lifecycle}

Service 对象可以由用户显式创建，也可以通过 `virtctl expose` 命令隐式创建。

显式创建 Service 时，需要由用户显式清理该 Service。从 v1.9 起，通过 `virtctl expose` 隐式创建 Service 时，会向该 Service 对象添加 [`ownerReference`](https://kubernetes.io/docs/concepts/overview/working-with-objects/owners-dependents/)，从而把 Service 绑定到 VM/VMI。因此，一旦被引用的对象（例如 VMI）被删除，k8s 垃圾回收器就会回收该 Service。
