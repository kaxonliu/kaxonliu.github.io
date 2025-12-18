# prometheus operator

**Prometheus Operator** 是一个 Kubernetes Operator，用于简化和自动化 Prometheus 监控栈在 Kubernetes 集群中的部署、配置和管理。它扩展了 Kubernetes API，允许用户通过声明式配置来管理 Prometheus 实例和相关组件。

Prometheus Operator 把 prometheus 监控体系涉及到的所以组件及相关配置都变成自定义资源，然后由自定义的控制器来实现自动化管理。



## 工作原理

自定义的控制器 watch 自定义的资源，发生变化则采取行动。

- **Prometheus**: 创建运行prometheus server的 pod。
- **ServiceMonitor**: 自动发现和监控服务。会转换成监控的 target 然后注入到 prometheus server 的配置文件中，并且自动 reload。
- **PrometheusRule**: 定义告警规则和记录规则。会转换成报警规则然后注入到 prometheus server 的配置文件中，并且自动 reload。
- **Alertmanager**: 管理 Alertmanager 实例。
- **AlertmanagerConfig** 配置资源。会转换成配置然后注入到 alertmanager 服务的配置中，并且自动重载。
- Probe: 监控 Ingress、静态目标等。

~~~bash
┌─────────────────────────────────────────────────┐
│          Custom Resources                       │
│  Prometheus ─ ServiceMonitor ─ PrometheusRule   │
│  AlertManager ─ PodMonitor ─ Probe - etc.       │
└────────────┬────────────────────────────────────┘
             │
┌────────────▼────────────────────────────────────┐
│          Prometheus Operator                    │
│   - 监视CRDs变化                                 │
│   - 生成实际配置                                  │
│   - 管理 Pod/Service 生命周期                     │
└────────────┬────────────────────────────────────┘
             │
┌────────────▼────────────────────────────────────┐
│         Managed Components                      │
│   ┌────────────┐  ┌──────────────┐              │
│   │ Prometheus │  │ Alertmanager │              │
│   └────────────┘  └──────────────┘              │
└─────────────────────────────────────────────────┘
~~~

~~~alert type=note
Prometheus资源----ServiceMonitor资源----service资源---提供/metrics接口的endpoint <br>
Prometheus资源----prometheusRule资源
~~~



## 使用 Operator 清单安装

官网：https://github.com/prometheus-operator/prometheus-operator

想要安装完整版监控栈请移步 [kube-prometheus](https://github.com/prometheus-operator/kube-prometheus)

安装前注意版本兼容问题，我的 k8s 集群是 1.31，因此我选择安装 kube-prometheus 的版本为 release-0.16

| kube-prometheus stack                                        | Kubernetes 1.29 | Kubernetes 1.30 | Kubernetes 1.31 | Kubernetes 1.32 | Kubernetes 1.33 | Kubernetes 1.34 |
| ------------------------------------------------------------ | --------------- | --------------- | --------------- | --------------- | --------------- | --------------- |
| [`release-0.14`](https://github.com/prometheus-operator/kube-prometheus/tree/release-0.14) | ✔               | ✔               | ✔               | x               | x               | x               |
| [`release-0.15`](https://github.com/prometheus-operator/kube-prometheus/tree/release-0.15) | x               | x               | ✔               | ✔               | ✔               | x               |
| [`release-0.16`](https://github.com/prometheus-operator/kube-prometheus/tree/release-0.16) | x               | x               | ✔               | ✔               | ✔               | ✔               |
| [`main`](https://github.com/prometheus-operator/kube-prometheus/tree/main) | x               | x               | x               | ✔               | ✔               | ✔               |



下载安装包

~~~bash
# 下载
wget https://github.com/prometheus-operator/kube-prometheus/archive/refs/tags/v0.16.0.tar.gz

# 解压
tar xf kube-prometheus-0.16.0.tar.gz

# 切换目录
cd kube-prometheus-0.16.0
~~~



替换国内镜像，把如下文件中的镜像替换成国内的镜像（可能还有其他文件中的镜像需要替换，遇到了再说）

~~~bash
manifests/prometheusAdapter-deployment.yaml
manifests/kubeStateMetrics-deployment.yaml
manifests/prometheusOperator-deployment.yaml
manifests/nodeExporter-daemonset.yaml
manifests/alertmanager-alertmanager.yaml
manifests/blackboxExporter-deployment.yaml
manifests/grafana-deployment.yaml
manifests/prometheus-prometheus.yaml
~~~



部署

~~~bash
# 创建一个名为 monitoring 的命名空间，以及相关的 CRD 资源对象声明
kubectl apply --server-side -f manifests/setup


# 在原地等待这些资源上面的CRD资源处于可用状态
# 命令结果输出xxxxxxxxxxxxxxxxx condition met代表ok
kubectl wait \
--for condition=Established \
--all CustomResourceDefinition \
--namespace=monitoring

# 部署 Operator的资源清单以及各种监控对象声明
kubectl apply -f manifests/
~~~

清理

~~~bash
kubectl delete -f manifests/
kubectl delete -f manifests/setup/ # 删除前面的诸多CRD声明、包括名称空间

# 当名称空间的删除阻塞在原地时，建议等一会
~~~

prometheus 及 alertmanager 的配置都是 secret 形式，可以使用如下三种方式查看

~~~bash
1、进入pod内直接查看明文
2、在web页面中查看
3、对secret进行解码
kubectl -n monitoring get secrets prometheus-k8s \
-o json| jq -r '.data."prometheus.yaml.gz"' | base64 -d | gzip -d
~~~

修改三个 svc 的类型为 nodeport，以便外部访问

~~~bash
kubectl -n monitoring edit svc alertmanager-main
kubectl -n monitoring edit svc grafana
kubectl -n monitoring edit svc prometheus-k8s
~~~



安装完 Prometheus Operator 之后，k8s 的节点、k8s 组件都被监控起来了，也就是说大多数监控都被监控起来了、并且对接好了grafana 可以出图。

但还有可以完善的地方：

- 报警还没有做
- 除了 Kubernetes 集群中的一些资源对象、节点以及组件需要监控，有的时候我们可能还需要根据实 际的业务需求去添加自定义的监控项。
- 持久化存储卷。默认都是 emptyDir，pod 重启数据丢失。



## 自定义监控

#### 前置知识点

ServiceMonitor 和 PrometheusRule 解决了 Prometheus 配置难维护问题，开发者不再需要通过修改 ConfigMap 把配置文件更新到 Pod 内再手动触发 webhook 热更新，只需要修改这两个对象资源就可以了.

- ServiceMoinitor 对应的是生成 target 监控指标。
- PrometheusRule 对应的是产生报警规则。



#### 原理

在 Prometheus Operator 中添加一个自定义的监控项非常简单。

- 第一步。确认被监控的应用对外暴漏了一个 `/metrics` 接口（可以是服务自带、也可以是安装的 exporter 提供的，该接口的服务不一定要跑在 k8s 中，只要是 endpoint 格式即可：http://ip:port 可以访问）
- 第二步。创建一个 svc 来关联上面的 endpoint
- 第三步。创建一个 ServiceMonitor 来选中 svc



#### 示例：监控 etcd

etcd 自身就提供了 `/metrics` 接口，不用额外安装 exporter 了，但是端口监听在 `127.0.0.1`上，需要改为 `0.0.0.0`，否则无法暴漏出来，导致 Prometheus server 无法跨节点拉取监控指标。

所有安装 etcd 的节点都要修改，修改后，每个节点上的 kubelet 发现变化后会自动更新静态 pod

~~~yaml
# 修改后的样子
[root@k8s-master-01 monitor]# grep 'listen-metrics-urls' /etc/kubernetes/manifests/etcd.yaml 
    - --listen-metrics-urls=http://0.0.0.0:2381
~~~

创建 svc 代理所有 etcd pod

~~~yaml
# etcd-svc.yaml
apiVersion: v1
kind: Service
metadata:
  name: etcd-svc	
  namespace: kube-system
  labels:
    app.kubernetes.io/component: etcd
    app.kubernetes.io/name: etcd
    k8s-app: etcd
spec:
  clusterIP: None
  ports:
  - name: port	# servicemonitor 会使用到这个名字
    port: 2381
    targetPort: 2381
    protocol: TCP
  selector:
    component: etcd	  # 筛选 scheduler pod
~~~

创建 serviceMonitor

~~~yaml
# kubernetesControlPlane-serviceMonitorEtcd.yaml
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: etcd-k8s
  namespace: monitoring
  labels:
    k8s-app: etcd-k8s
spec:
  jobLabel: k8s-app
  endpoints:
    - port: port
      interval: 15s
  selector:
    matchLabels:
      k8s-app: etcd  # 选中svc
  namespaceSelector:
    matchNames:
      - kube-system
~~~

上述创建 svc 、servicemonitor 即可完成监控 etcd。

下面再看看如何使用一种更加通用的方式监控服务。

也可以把 etcd 当成独立于集群之外的服务，直接用 `ip:port` 来关联的，所以下面我们可以使用另一种 svc 并没有定义一个 selector 选择器来选中相关 pod，也正因为没有用标签选择器去选中 pod，所以我们需要自定义 endpoint 来找到被 svc 关联的应用。

~~~yaml
# etcd-svc.yaml
apiVersion: v1
kind: Service
metadata:
  name: etcd-k8s
  namespace: kube-system
  labels:
    k8s-app: etcd  # 这里的标签就是为serviceMonitor准备的，一会serviceMonitor就用该标签
spec:
  type: ClusterIP
  clusterIP: None
  ports:
  - name: port
    port: 2381
---
apiVersion: v1
kind: Endpoints
metadata:
  name: etcd-k8s
  namespace: kube-system
  labels:
    k8s-app: etcd   # 这里的标签必须要与svc的保持一致
subsets:
  # 指定etcd节点地址，如果是集群则继续向下添加
  - addresses:
      - ip: 192.168.10.81
        nodeName: k8s-master-01
      - ip: 192.168.10.82
        nodeName: k8s-master-02
      - ip: 192.168.10.83
        nodeName: k8s-master-03
    ports:
      - name: port
        port: 2381
~~~



## 配置报警

我们去查看 Prometheus Dashboard 的 Alert 页面下面就已经有很多报警规则。

这一系列的规则其实都来自于项目 https://github.com/kubernetes-monitoring/kubernetes-mixin， 我们都通过 Prometheus Operator 安装配置上了。

只需要使用 PrometheusRule 资源创建配置清单即可。应用后，会自动往 prometheus server pod 内的配置路径下塞一个新的报警规则

~~~yaml
rule_files:
# 这下面所有的yaml结果的文件都是报警规则
- /etc/prometheus/rules/prometheus-k8s-rulefiles-0/*.yaml 
~~~

之所以添加 PrometheusRule 资源，可以被找到并放到Prometheus server 的配置中，是因为 Prometheus server 是由 Prometheus 资源产生的，在该资源中使用了选择器来选中 PrometheusRule 资源。

~~~yaml
[root@k8s-master-01 /monitor]# kubectl -n monitoring get prometheus k8s -o yaml
apiVersion: monitoring.coreos.com/v1
kind: Prometheus
metadata:
  ..............................
  name: k8s
  namespace: monitoring
spec:
  .....
  ruleSelector: {} 
  # 这个就是用来选中PrometheusRule的规则选择器，这里设置的空，代表可以匹配所有
~~~

如果你想只匹配 prometheus=k8s 和 role=alert-rules 标签的 PrometheusRule 资源对象，则可以添加下面的配置：

~~~yaml
ruleSelector:
  matchLabels:
    prometheus: k8s
    role: alert-rules
~~~

对应着你创建的 PrometheusRule 资源就需要带上相关标签。

但是此处人家设置的是 `ruleSelector: {}` 代表可以选中所有 PrometheusRule 对象，所以我们创建 PrometheusRule 对象时不必加什么专门的标签，这是非常方便的。



#### 报警示例

需求：如下创建一个 etcd 的报警规则，不可用 etcd 数超过一半就触发报警。

做法：

1. 创建一个 PrometheusRule 资源，定义报警的指标和阈值。
2. 创建一个 alertmanager 的配置，配置报警消息发出的媒介。

其中，创建 alertmanager  的配置具体有两种方式

- 方式1：直接修改 alertmanager-secert 主配置文件。在这个文件中添加报警媒介的使用规则。
- 方式2：使用 alertmanagerConfig 资源，直接为 alertmanager 资源提供配置。不需要修改重启 alertmanager 实例。但是有一个小 BUG（会自动把标签 `namespace=monitoring` 自动加到报警消息的匹配逻辑上，如果 PrometheusRule  中没有给报警消息加上这个标签，则匹配失败）。



#### 使用方式1：实现邮件和钉钉同步发送报警消息

如下创建一个 etcd 的报警规则，不可用 etcd 数超过一半就触发报警。

~~~yaml
# prometheus-etcdRules.yaml
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: etcd-rules
  namespace: monitoring
  # labels:  # 可选：如果需要被特定的 Prometheus 实例选择
  #   prometheus: k8s
  #   role: alert-rules
spec:
  groups:
  - name: etcd
    rules:
    - alert: EtcdClusterUnavailable
      annotations:
        summary: etcd cluster small
        description: If one more etcd peer goes down the cluster will be unavailable
      expr: |
        count(up{job="etcd"} == 0) > ceil(count(up{job="etcd"}) / 2)
      for: 1m
      # for: 3m  # 通常生产环境建议 3m 或更长
      labels:
        severity: critical
~~~

部署 钉钉 webhook

- 略

**【不建议】**直接修改一下 altermanager 的配置的原始 yaml 文件 `alertmanager-secret.yaml`

~~~yaml
# /manifest/alertmanager-secret.yaml
apiVersion: v1
kind: Secret
metadata:
  labels:
    app.kubernetes.io/component: alert-router
    app.kubernetes.io/instance: main
    app.kubernetes.io/name: alertmanager
    app.kubernetes.io/part-of: kube-prometheus
    app.kubernetes.io/version: 0.28.1
  name: alertmanager-main
  namespace: monitoring
stringData:
  alertmanager.yaml: |-
    "global":
      "resolve_timeout": "5m"
      "smtp_smarthost": "smtp.qq.com:465"
      "smtp_from": "1505797244@qq.com
      "smtp_auth_username": "1505797244@qq.com"
      "smtp_auth_password": "uliijzrcgpngiied" 
      "smtp_require_tls": false
    "receivers":
    - "name": "Default"
    - "name": "Watchdog"
    - "name": "Critical"
    - "name": "null"
    - "name": "mywebhook"
      "webhook_configs":
        - "url": "http://promoter.monitor:8080/dingtalk/webhook1/send"
          "send_resolved": true
    - "name": "email"
       "email_configs":
         - "to": "1505797244@qq.com"
           "send_resolved": true
    "route":
      "group_by":
      - "namespace"
      "group_interval": "1m"
      "group_wait": "30s"
      "repeat_interval": "1m"
      "receiver": "mywebhook"
      "routes":
      - "matchers":
        - "severity = kaxon-critical"
        "receiver": "mywebhook"
        "continue": true
      - "matchers":
        - "severity = kaxon-critical"
        "receiver": "email"
        "continue": true
type: Opaque
~~~

修改保存后，apply -f 应用，得到一个名为 `alertmanager-main` 的 secret 资源。



**【建议的方式】**当然了，上述直接直接文件的方式比较麻烦，也可以先把 `alertmanager-main` 删除，然后使用自己的文件创建这个 secret 资源

1. 删除 altermanager 原来的主配置资源 `alertmanager-main`

~~~bash
kubecte -n monitoring delete secret alertmanager-main
~~~

2. 新建 myself.conf 文件

~~~
global:
  resolve_timeout: 5m
  smtp_smarthost: 'smtp.qq.com:465'
  smtp_from: '1505797244@qq.com'
  smtp_auth_username: '1505797244@qq.com'
  smtp_auth_password: 'uliijzrcgpngiied' 
  smtp_require_tls: false

route:
  group_interval: 1m
  group_wait: 30s
  receiver: mywebhook
  repeat_interval: 1m
  routes:
  - matchers:
    - severity = kaxon-critical
    receiver: mywebhook
    continue: true
  - matchers:
    - severity = kaxon-critical
    receiver: email
    continue: true

receivers:
- name: mywebhook
  webhook_configs:
  - url: 'http://promoter.monitor:8080/dingtalk/webhook1/send'
    send_resolved: true

- name: email
  email_configs:
  - to: '1505797244@qq.com'
    send_resolved: true
~~~

3. 创建新的 altermanager   资源 `alertmanager-main`

~~~bash
kubectl create secret generic alertmanager-main --from-file=alertmanager.yaml -n monitoring
~~~

4. 查看 secret ，如下 alertmanager-main-generated 就是 alertmanager-main 创建配置的结果。

~~~bash
[root@k8s-master-01 monitor]# kubectl -n monitoring get secret
NAME                                                TYPE     DATA   AGE
alertmanager-main                                   Opaque   1      2m50s
alertmanager-main-cluster-tls-config                Opaque   1      4h19m
alertmanager-main-generated                         Opaque   1      4h19m
alertmanager-main-tls-assets-0                      Opaque   0      4h19m
alertmanager-main-web-config                        Opaque   1      4h19m
grafana-config                                      Opaque   1      4h19m
grafana-datasources                                 Opaque   1      4h19m
prometheus-k8s                                      Opaque   1      4h19m
prometheus-k8s-thanos-prometheus-http-client-file   Opaque   1      4h19m
prometheus-k8s-tls-assets-0                         Opaque   0      4h19m
prometheus-k8s-web-config                           Opaque   1      4h19m
~~~

可以使用如下命令查看配置是否满足需求，当然也可以到 web ui 上看（`http://<node ip>:<nodeport>/#/status`）

~~~bash
[root@k8s-master-01 monitor]# kubectl get secret alertmanager-main-generated -n monitoring -o json | jq -r '.data."alertmanager.yaml.gz"' | base64 -d | gzip -d
global:
  resolve_timeout: 5m
  smtp_smarthost: 'smtp.qq.com:465'
  smtp_from: 'xxx@qq.com'
  smtp_auth_username: 'xxx@qq.com'
  smtp_auth_password: 'i....h' 
  smtp_require_tls: false

route:
  group_interval: 1m
  group_wait: 30s
  receiver: mywebhook
  repeat_interval: 1m
  routes:
  - matchers:
    - severity = kaxon-critical
    receiver: mywebhook
    continue: true
  - matchers:
    - severity = kaxon-critical
    receiver: email
    continue: true

receivers:
- name: mywebhook
  webhook_configs:
  - url: 'http://promoter.monitor:8080/dingtalk/webhook1/send'
    send_resolved: true

- name: email
  email_configs:
  - to: 'xxx@qq.com'
    send_resolved: true
~~~

然后就可以同时收到邮件消息和钉钉消息了。

**使用自定义邮件模板**

创建 alertmanager.yaml 配置文件，使用 `templates` 指定模板文件在容器内的路径。使用 eamil 的  receiver 下面使用 `html` 配置使用模板的名字为 `email.html`

~~~yaml
templates:
    - '/etc/alertmanager/config/template_email.tmpl'
global:
  resolve_timeout: 5m
  smtp_smarthost: 'smtp.qq.com:465'
  smtp_from: '1505797244@qq.com'
  smtp_auth_username: '1505797244@qq.com'
  smtp_auth_password: 'uliijzrcgpngiied' 
  smtp_require_tls: false

route:
  group_interval: 1m
  group_wait: 30s
  receiver: mywebhook
  repeat_interval: 1m
  routes:
  - matchers:
    - severity = kaxon-critical
    receiver: mywebhook
    continue: true
  - matchers:
    - severity = kaxon-critical
    receiver: email
    continue: true

receivers:
- name: mywebhook
  webhook_configs:
  - url: 'http://promoter.monitor:8080/dingtalk/webhook1/send'
    send_resolved: true

- name: email
  email_configs:
  - to: '1505797244@qq.com'
    send_resolved: true
    html: '{{ template "email.html" . }}'
~~~

创建一个模板文件 template_email.tmp，其中定义了模板的名字为  `email.html`

~~~
{{ define "email.html" }}
{{- if gt (len .Alerts.Firing) 0 -}}
    @报警<br>
    {{- range .Alerts }}
    <strong>实例:</strong> {{ .Labels.instance }}<br>
    <strong>概述:</strong> {{ .Annotations.summary }}<br>
    <strong>详情:</strong> {{ .Annotations.description }}<br>
    <strong>时间:</strong> {{ (.StartsAt.Add 28800e9).Format "2006-01-02 15:04:05" }}
    <br>
    {{- end -}}
{{- end }}
{{- if gt (len .Alerts.Resolved) 0 -}}
    @恢复<br>
    {{- range .Alerts }}
    <strong>实例:</strong> {{ .Labels.instance }}<br>
    <strong>信息:</strong> {{ .Annotations.summary }}<br>
    <strong>恢复:</strong> {{ (.StartsAt.Add 28800e9).Format "2006-01-02 15:04:05" }}
    <br>
    {{- end -}}
{{- end }}
{{ end }}

~~~

删除当前的 alertmanager-secret 资源，创建新的 alertmanager-secret

~~~bash
kubectl create secret generic alertmanager-main \
--from-file=alertmanager.yaml \
--from-file=template_email.tmpl \
-n monitoring
~~~





#### 使用方式2：实现邮件和钉钉同步发送报警消息

即使用创建 AlertmanagerConfig 资源对象的方式实现报警消息通知的配置。

~~~bash
# 在 altermanager 资源中设置标签来选中你的 AltermanagerConfig 资源
$ vim kube-prometheus/manifests/alertmanager-alertmanager.yaml
...
serviceAccountName: alertmanager-main
version: 0.27.0
alertmanagerConfigSelector: # 匹配 AlertmanagerConfig 的标签
  matchLabels:
    alertmanagerConfig: example
~~~

然后重新部署

~~~bash
kubectl apply -f kube-prometheus/manifests/alertmanager-alertmanager.yaml
~~~

然后创建 AlertmanagerConfig 资源，注意必须加上标签 alertmanagerConfig: example

下面以发邮件通知为例介绍

创建 邮件密码 secret

~~~bash
kubectl create secret generic my-email-secret --from-literal=password='ias...agh' -n monitoring
~~~

创建 AlertmanagerConfig 资源

~~~yaml
# alertmanager-config.yaml
# 配置2 邮件
apiVersion: monitoring.coreos.com/v1alpha1
kind: AlertmanagerConfig
metadata:
  name: config1
  namespace: monitoring
  labels:
    alertmanagerConfig: example  # 必须加上标签，用于 Alertmanager 选择
spec:
  route:
    receiver: email
    groupWait: 30s
    groupInterval: 1m
    repeatInterval: 1m
    continue: true
    matchers:
    - name: severity
      value: kaxon-critical
      regex: false
  
  receivers:
  - name: email
    emailConfigs:
    - to: 'xxx@qq.com'
      from: 'xxx@qq.com'
      hello: qq.com
      smarthost: 'smtp.qq.com:465'
      authUsername: 'xxx@qq.com'
      authIdentity: 'xxx@qq.com'
      authPassword:
        name: my-email-secret		# 引用 secret 
        key: password
      requireTLS: false
      sendResolved: true
      
---
# 配置1 钉钉
apiVersion: monitoring.coreos.com/v1alpha1
kind: AlertmanagerConfig
metadata:
  name: config2
  namespace: monitoring
  labels:
    alertmanagerConfig: example  # 必须加上标签，用于 Alertmanager 选择
spec:
  route:
    receiver: mywebhook
    groupWait: 30s
    groupInterval: 1m
    repeatInterval: 1m
    continue: true
    
    matchers:
    - name: severity
      value: kaxon-critical
      regex: false
     
  receivers:
  - name: mywebhook
    webhookConfigs:
    - url: "http://promoter.monitor:8080/dingtalk/webhook1/send"
      sendResolved: true 
~~~

应用后然后查看

~~~bash
[root@k8s-master-01 monitor]# kubectl apply -f alertmanager-config.yaml 
alertmanagerconfig.monitoring.coreos.com/config1 created
alertmanagerconfig.monitoring.coreos.com/config2 created
[root@k8s-master-01 monitor]# 
[root@k8s-master-01 monitor]# 
[root@k8s-master-01 monitor]# kubectl -n monitoring  get alertmanagerconfigs.monitoring.coreos.com
NAME      AGE
config1   7s
config2   7s
[root@k8s-master-01 monitor]# kubectl get secret alertmanager-main-generated -n monitoring -o json | jq -r '.data."alertmanager.yaml.gz"' | base64 -d | gzip -d
global:
  resolve_timeout: 5m
route:
  receiver: Default
  group_by:
  - namespace
  routes:
  - receiver: monitoring/config1/email
    matchers:
    - severity="kaxon-critical"
    - namespace="monitoring"
    continue: true
    group_wait: 30s
    group_interval: 1m
    repeat_interval: 1m
  - receiver: monitoring/config2/mywebhook
    matchers:
    - severity="kaxon-critical"
    - namespace="monitoring"
    continue: true
    group_wait: 30s
    group_interval: 1m
    repeat_interval: 1m
  - receiver: Watchdog
    matchers:
    - alertname = Watchdog
  - receiver: "null"
    matchers:
    - alertname = InfoInhibitor
  - receiver: Critical
    matchers:
    - severity = critical
  group_wait: 30s
  group_interval: 1m
  repeat_interval: 1m
inhibit_rules:
- target_matchers:
  - severity =~ warning|info
  source_matchers:
  - severity = critical
  equal:
  - namespace
  - alertname
- target_matchers:
  - severity = info
  source_matchers:
  - severity = warning
  equal:
  - namespace
  - alertname
- target_matchers:
  - severity = info
  source_matchers:
  - alertname = InfoInhibitor
  equal:
  - namespace
receivers:
- name: Default
- name: Watchdog
- name: Critical
- name: "null"
- name: monitoring/config1/email
  email_configs:
  - send_resolved: true
    to: xxx@qq.com
    from: xxx@qq.com
    hello: qq.com
    smarthost: smtp.qq.com:465
    auth_username: xxx@qq.com
    auth_password: xxx
    auth_identity: xxx@qq.com
    require_tls: false
- name: monitoring/config2/mywebhook
  webhook_configs:
  - send_resolved: true
    url: http://promoter.monitor:8080/dingtalk/webhook1/send
templates: []
~~~

直接部署 alertmanagerConfig 资源的话，prometheus operator 总会帮你加一个标签 `namespace="monitoring"`

~~~yaml
routes:
- receiver: monitoring/dinghook/email
  group_by:
  - severity
  match:
    mywarn: kaxon-critical
  matchers:
  - namespace="monitoring" # 就是这个标签
~~~

而 math 的与 matchers 二者的标签匹配是一个 and 的关系，就会要求你的报警规则必须同时拥有这两个标签才可以，否则不会发送给该接收器。

可以修改 promtheusRule，多加一个标签 namespace="monitoring"，邮件报警就可以收到了。

~~~yaml
# prometheus-etcdRules.yaml
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: etcd-rules
  namespace: monitoring
spec:
  groups:
  - name: etcd
    rules:
    - alert: EtcdClusterUnavailable
      annotations:
        summary: etcd cluster small
        description: If one more etcd peer goes down the cluster will be unavailable
      expr: |
        count(up{job="etcd"} == 0) > ceil(count(up{job="etcd"}) / 2)
      for: 1m
      # for: 3m  # 通常生产环境建议 3m 或更长
      labels:
        severity: critical
        namespace: monitoring   ############# 增加这个标签
~~~

部署该 yaml，过一会后确认规则更新 ok，并且重新 firing

注意：使用该方式无法直接指定邮件使用的模板