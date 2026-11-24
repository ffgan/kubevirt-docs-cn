# 调试 {#debug}

本页说明如何调试 KubeVirt。

这对 KubeVirt 开发者以及希望深入了解幕后究竟发生了什么的高级用户都很有用。

## 日志详细级别 {#log-verbosity}

KubeVirt 会在整个代码库中产生大量日志。部分日志条目设置了详细级别。为日志条目设置的详细级别，决定了要显示该日志条目所需的最低详细级别。

在代码中，日志条目看起来类似于：`log.Log.V(verbosity).Infof("...")`，其中 `verbosity` 是该条目的最低详细级别。

例如，如果某条日志条目的日志详细级别为 `3`，那么只有当日志详细级别被定义为等于或大于 `3` 时，该日志才会被显示，否则它会被过滤掉。

目前，日志详细级别可以按组件或按节点来定义。最新的 API 详见[此处](https://kubevirt.io/api-reference/master/definitions.html#_v1_logverbosity)。

### 按 KubeVirt 组件设置详细级别 {#setting-verbosity-per-kubevirt-component}

提高日志详细级别的一种方式，是在 `KubeVirt` CR 中为不同组件手动设定：
```yaml
apiVersion: kubevirt.io/v1
kind: KubeVirt
metadata:
  name: kubevirt
  namespace: kubevirt
spec:
  configuration:
    developerConfiguration:
      logVerbosity:
        virtLauncher: 2
        virtHandler: 3
        virtController: 4
        virtAPI: 5
        virtOperator: 6
```

此选项最适合调试特定组件。

#### 根据 virt-launcher 的日志详细级别设置 libvirt virtqemud.conf 中的 log_filters {#libvirt-virtqemudconf-set-log_filters-according-to-virt-launcher-log-verbosity}

详细级别 | virtqemud.conf 中的 log_filters
-- | --
5 | log_filters="3:remote 4:event 3:util.json 3:util.object 3:util.dbus 3:util.netlink 3:node_device 3:rpc 3:access 3:util.threadjob 3:cpu.cpu 3:qemu.qemu_monitor 3:qemu.qemu_monitor_json 3:conf.domain_addr 1:*" |
6 | 3:remote 4:event 3:util.json 3:util.object 3:util.dbus 3:util.netlink 3:node_device 3:rpc 3:access 3:util.threadjob 3:cpu.cpu 3:qemu.qemu_monitor 1:*
7 | 3:remote 4:event 3:util.json 3:util.object 3:util.dbus 3:util.netlink 3:node_device 3:rpc 3:access 3:util.threadjob 3:cpu.cpu 1:*
8 及以上 | 3:remote 4:event 3:util.json 3:util.object 3:util.dbus 3:util.netlink 3:node_device 3:rpc 3:access 1:*

用户可以通过 VMI 配置中的注解 `kubevirt.io/libvirt-log-filters` 设置自定义日志过滤器。例如：
```yaml
kind: VirtualMachineInstance
metadata:
  name: my-vmi
  annotations:
    kubevirt.io/libvirt-log-filters: "3:remote 4:event 1:*"

```

### 按节点设置详细级别 {#setting-verbosity-per-nodes}

另一种方式是按节点设置详细级别：
```yaml
apiVersion: kubevirt.io/v1
kind: KubeVirt
metadata:
  name: kubevirt
  namespace: kubevirt
spec:
  configuration:
    developerConfiguration:
      nodeVerbosity:
        "node01": 4
        "otherNodeName": 6
```

`nodeVerbosity` 本质上是一个从字符串到整数的映射，其中键是节点名称，值是详细级别。该详细级别会为该节点上所有不同的组件设定（例如 `virt-handler`、`virt-launcher` 等）。

### 如何获取 KubeVirt 组件的日志 {#how-to-retrieve-kubevirt-components-logs}

在 Kubernetes 中，日志定义在 Pod 级别。因此，首先需要列出 KubeVirt 核心组件的 Pod。为此，我们可以先列出 KubeVirt 安装命名空间下的 Pod。

例如：
```bash
$> kubectl get pods -n <KubeVirt Install Namespace>
NAME                               READY   STATUS    RESTARTS   AGE
disks-images-provider-7gqbc        1/1     Running   0          32m
disks-images-provider-vg4kx        1/1     Running   0          32m
virt-api-57fcc4497b-7qfmc          1/1     Running   0          31m
virt-api-57fcc4497b-tx9nc          1/1     Running   0          31m
virt-controller-76c784655f-7fp6m   1/1     Running   0          30m
virt-controller-76c784655f-f4pbd   1/1     Running   0          30m
virt-handler-2m86x                 1/1     Running   0          30m
virt-handler-9qs6z                 1/1     Running   0          30m
virt-operator-7ccfdbf65f-q5snk     1/1     Running   0          32m
virt-operator-7ccfdbf65f-vllz8     1/1     Running   0          32m
```

然后，我们可以挑选其中一个 Pod 并获取其日志。例如：
```bash
$> kubectl logs -n <KubeVirt Install Namespace> virt-handler-2m86x | head -n8
{"component":"virt-handler","level":"info","msg":"set verbosity to 2","pos":"virt-handler.go:453","timestamp":"2022-04-17T08:58:37.373695Z"}
{"component":"virt-handler","level":"info","msg":"set verbosity to 2","pos":"virt-handler.go:453","timestamp":"2022-04-17T08:58:37.373726Z"}
{"component":"virt-handler","level":"info","msg":"setting rate limiter to 5 QPS and 10 Burst","pos":"virt-handler.go:462","timestamp":"2022-04-17T08:58:37.373782Z"}
{"component":"virt-handler","level":"info","msg":"CPU features of a minimum baseline CPU model: map[apic:true clflush:true cmov:true cx16:true cx8:true de:true fpu:true fxsr:true lahf_lm:true lm:true mca:true mce:true mmx:true msr:true mtrr:true nx:true pae:true pat:true pge:true pni:true pse:true pse36:true sep:true sse:true sse2:true sse4.1:true ssse3:true syscall:true tsc:true]","pos":"cpu_plugin.go:96","timestamp":"2022-04-17T08:58:37.390221Z"}
{"component":"virt-handler","level":"warning","msg":"host model mode is expected to contain only one model","pos":"cpu_plugin.go:103","timestamp":"2022-04-17T08:58:37.390263Z"}
{"component":"virt-handler","level":"info","msg":"node-labeller is running","pos":"node_labeller.go:94","timestamp":"2022-04-17T08:58:37.391011Z"}

```

显然，对于上面两个示例，都需要把 `<KubeVirt Install Namespace>` 替换为 KubeVirt 实际安装所在的命名空间。

## KubeVirt PProf 分析器 {#kubevirt-pprof-profiler}

使用 `cluster-profiler` 客户端工具，开发者可以获取 KubeVirt 控制平面中每个组件的 PProf 性能分析数据。以下是一份用户指南：

### 编译 `cluster-profiler` {#compile-cluster-profiler}

从源代码构建。
```
$ git clone https://github.com/kubevirt/kubevirt.git
$ cd kubevirt/tools/cluster-profiler
$ go build
```

### 启用功能门控 {#enable-the-feature-gate}

在 KubeVirt 配置中添加 `ClusterProfiler`。
```
$ cat << END > enable-feature-gate.yaml

---
apiVersion: kubevirt.io/v1
kind: KubeVirt
metadata:
  name: kubevirt
  namespace: kubevirt
spec:
  configuration:
    developerConfiguration:
      featureGates:
        - ClusterProfiler
END

$ kubectl apply -f enable-feature-gate.yaml
```

### 执行性能分析 {#do-the-profiling}

开始 CPU 性能分析。
```
$ cluster-profiler --cmd start

2023/05/17 09:31:09 SUCCESS: started cpu profiling KubeVirt control plane
```
停止 CPU 性能分析。
```
$ cluster-profiler --cmd stop

2023/05/17 09:31:14 SUCCESS: stopped cpu profiling KubeVirt control plane
```
转储 pprof 结果。
```
$ cluster-profiler --cmd dump

2023/05/17 09:31:18 Moving already existing "cluster-profiler-results" => "cluster-profiler-results-old-67fq"
SUCCESS: Dumped PProf 6 results for KubeVirt control plane to [cluster-profiler-results]
```
PProf 结果可以在文件夹 `cluster-profiler-results` 中找到。
```
$ tree cluster-profiler-results

cluster-profiler-results
├── virt-api-5f96f84dcb-lkpb7
│   ├── allocs.pprof
│   ├── block.pprof
│   ├── cpu.pprof
│   ├── goroutine.pprof
│   ├── heap.pprof
│   ├── mutex.pprof
│   └── threadcreate.pprof
├── virt-controller-5bbd9554d9-2f8j2
│   ├── allocs.pprof
│   ├── block.pprof
│   ├── cpu.pprof
│   ├── goroutine.pprof
│   ├── heap.pprof
│   ├── mutex.pprof
│   └── threadcreate.pprof
├── virt-controller-5bbd9554d9-qct2w
│   ├── allocs.pprof
│   ├── block.pprof
│   ├── cpu.pprof
│   ├── goroutine.pprof
│   ├── heap.pprof
│   ├── mutex.pprof
│   └── threadcreate.pprof
├── virt-handler-ccq6c
│   ├── allocs.pprof
│   ├── block.pprof
│   ├── cpu.pprof
│   ├── goroutine.pprof
│   ├── heap.pprof
│   ├── mutex.pprof
│   └── threadcreate.pprof
├── virt-operator-cdc677b7-pg9j2
│   ├── allocs.pprof
│   ├── block.pprof
│   ├── cpu.pprof
│   ├── goroutine.pprof
│   ├── heap.pprof
│   ├── mutex.pprof
│   └── threadcreate.pprof
└── virt-operator-cdc677b7-pjqdx
    ├── allocs.pprof
    ├── block.pprof
    ├── cpu.pprof
    ├── goroutine.pprof
    ├── heap.pprof
    ├── mutex.pprof
    └── threadcreate.pprof
```
