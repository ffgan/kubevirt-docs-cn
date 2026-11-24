# KubeVirt 调度器 {#kubevirt-scheduler}
调度是将 Pod/VM 匹配到节点的过程。默认使用的调度器是
[kube-scheduler](https://kubernetes.io/docs/reference/command-line-tools-reference/kube-scheduler/)。
更多细节见[Kubernetes 调度器文档](https://kubernetes.io/docs/concepts/scheduling-eviction/kube-scheduler/)。

如果默认调度器不能满足你的需求，可以使用自定义调度器。例如，你可能希望使用感知负载的调度器来调度 VM，例如 [Trimaran 调度器](https://cloud.redhat.com/blog/improving-the-resource-efficiency-for-openshift-clusters-via-trimaran-schedulers)。

## 创建自定义调度器 {#creating-a-custom-scheduler}
KubeVirt 与自定义调度器兼容。配置步骤在[官方 Kubernetes 文档](https://kubernetes.io/docs/tasks/extend-kubernetes/configure-multiple-schedulers)中有说明。
请注意，KubeVirt 所运行的 Kubernetes 版本与用于构建自定义调度器的 Kubernetes 版本必须一致。
要获取 KubeVirt 所运行的 Kubernetes 版本，可以运行以下命令：

```shell
$ kubectl version
Client Version: version.Info{Major:"1", Minor:"22", GitVersion:"v1.22.13", GitCommit:"a43c0904d0de10f92aa3956c74489c45e6453d6e", GitTreeState:"clean", BuildDate:"2022-08-17T18:28:56Z", GoVersion:"go1.16.15", Compiler:"gc", Platform:"linux/amd64"}
Server Version: version.Info{Major:"1", Minor:"22", GitVersion:"v1.22.13", GitCommit:"a43c0904d0de10f92aa3956c74489c45e6453d6e", GitTreeState:"clean", BuildDate:"2022-08-17T18:23:45Z", GoVersion:"go1.16.15", Compiler:"gc", Platform:"linux/amd64"}
```

请注意 `Server` 这一行。
在本例中，Kubernetes 版本是 `v1.22.13`。
你必须检出匹配的 Kubernetes 版本并[构建 Kubernetes 项目](https://kubernetes.io/docs/tasks/extend-kubernetes/configure-multiple-schedulers/#package-the-scheduler)：

```shell
$ cd kubernetes
$ git checkout v1.22.13
$ make
```

然后，你可以按照[此处](https://kubernetes.io/docs/tasks/extend-kubernetes/configure-multiple-schedulers/)描述的配置步骤操作。
此外，ClusterRole `system:kube-scheduler` 需要具备对 StorageClasses 使用 `watch`、`list` 和 `get` 动词的权限。

```yaml
- apiGroups:                                                                                                   
  - storage.k8s.io                                                                                             
  resources:                                                                                                   
  - storageclasses                                                                                             
  verbs:                                                                                                       
  - watch                                                                                                      
  - list                                                                                                       
  - get 
```


## 使用自定义调度器调度 VM {#scheduling-vms-with-the-custom-scheduler}

第二个调度器应当已经启动并运行。你可以用以下命令检查：

```shell
$ kubectl get all -n kube-system
```

如果一切设置正确，Deployment `my-scheduler` 应当已经启动并运行。
要使用自定义调度器启动 VM，需要将 VM spec 中的 `SchedulerName` 设置为 `my-scheduler`。
以下是一个 VM 定义示例：

```yaml
apiVersion: kubevirt.io/v1
kind: VirtualMachine
metadata:
  name: vm-fedora
spec:
  runStrategy: Always
  template:
    spec:
      schedulerName: my-scheduler
      domain:
        devices:
          disks:
            - name: containerdisk
              disk:
                bus: virtio
            - name: cloudinitdisk
              disk:
                bus: virtio
          rng: {}
        resources:
          requests:
            memory: 1Gi
      terminationGracePeriodSeconds: 180
      volumes:
        - containerDisk:
            image: quay.io/containerdisks/fedora:latest
          name: containerdisk
        - cloudInitNoCloud:
            userData: |-
              #cloud-config
              chpasswd:
                expire: false
              password: fedora
              user: fedora
          name: cloudinitdisk
```
如果指定的 `SchedulerName` 与任何现有调度器都不匹配，`virt-launcher` Pod 将保持
[Pending](https://kubernetes.io/docs/tasks/extend-kubernetes/configure-multiple-schedulers/#verifying-that-the-pods-were-scheduled-using-the-desired-schedulers)
状态，直到找到指定的调度器。
你可以检查 VM 是否已使用 `my-scheduler` 完成调度，方法是查看与该 VM 关联的 `virt-launcher` Pod 事件。该 Pod 应当已由 `my-scheduler` 调度。

```shell
$ kubectl get pods
NAME                             READY   STATUS    RESTARTS   AGE
virt-launcher-vm-fedora-dpc87    2/2     Running   0          24m

$ kubectl describe pod virt-launcher-vm-fedora-dpc87
[...] 
Events:
  Type    Reason     Age   From              Message
  ----    ------     ----  ----              -------
  Normal  Scheduled  21m   my-scheduler  Successfully assigned default/virt-launcher-vm-fedora-dpc87 to node01
[...]
```