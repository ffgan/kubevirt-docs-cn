
## 自定义 KubeVirt 组件 {#customize-kubevirt-components}


### 使用补丁自定义组件 {#customize-components-using-patches}

> :warning: 如果创建的补丁无效，KubeVirt 将无法更新或部署系统。此功能面向特殊用例，除非你清楚自己在做什么，否则不应使用。

有效的资源类型有：Deployment、DaemonSet、Service、ValidatingWebhookConfiguraton、MutatingWebhookConfiguration、APIService 和 CertificateSecret。更多信息见 [API 规范](http://kubevirt.io/api-reference/master/definitions.html#_v1_customizecomponentspatch)。


自定义补丁示例：
```yaml
---
apiVersion: kubevirt.io/v1
kind: KubeVirt
metadata:
  name: kubevirt
  namespace: kubevirt
spec:
  certificateRotateStrategy: {}
  configuration: {}
  customizeComponents:
    patches:
    - resourceType: Deployment
      resourceName: virt-controller
      patch: '[{"op": "remove", "path": "/spec/template/spec/containers/0/livenessProbe"}]'
      type: json
    - resourceType: Deployment
      resourceName: virt-controller
      patch: '{"metadata":{"annotations":{"patch": "true"}}}'
      type: strategic
```

上面的示例会更新 `virt-controller` Deployment，在其 metadata 中添加内容为 `patch: true` 的注解，并从容器定义中移除 livenessProbe。


### 自定义标志 {#customize-flags}

> :warning: 如果标志无效，或在更新后变为无效，组件将无法运行


使用自定义标志选项时，对于要应用标志的组件，所有默认标志都会被移除，只使用指定的标志。可以更改标志的可用资源是 `api`、`controller` 和 `handler`。关于该 API 的更多细节见 [API 规范](http://kubevirt.io/api-reference/master/definitions.html#_v1_flags)。

```yaml
apiVersion: kubevirt.io/v1
kind: KubeVirt
metadata:
  name: kubevirt
  namespace: kubevirt
spec:
  certificateRotateStrategy: {}
  configuration: {}
  customizeComponents:
    flags:
      api:
        v: "5"
        port: "8443"
        console-server-port: "8186"
        subresources-only: "true"
```

上面的示例会生成一个 `virt-api` Pod，其命令如下

```yaml
...
spec:
  ....
  container:
  - name: virt-api
    command:
    - virt-api
    - --v
    - "5"
    - --console-server-port
    - "8186"
    - --port
    - "8443"
    - --subresources-only
    - "true"
    ...
```
