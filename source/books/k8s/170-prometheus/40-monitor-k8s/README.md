# 监控 k8s

采集 k8s 集群本身的指标。

- k8s 集群运行的服务。监控组件服务（kube-apiserver、 kube-scheduler、kube-controller-manager、etcd、coredns）的状态；监控业务服务（业务 pod）状态。
- 资源的状态。比如 Deployment 的状态、pod 的状态。
- 监控容器的状态。需要用到组件 `cAdvisor`，而  cAdvisor 已经内置在了 kubelet 组件之中，不需要单独去安装。



## 监控容器

在 prometheus 的配置中增加下述 target，然后 apply -f

~~~yaml
# prometheus-cm.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: prometheus-config
  namespace: monitor
data:
  prometheus.yml: |
    global:
      scrape_interval: 15s
      scrape_timeout: 15s
      evaluation_interval: 15s
    scrape_configs:
    - job_name: "prometheus"
      static_configs:
      - targets: ["localhost:9090"]
    - job_name: 'kubernetes-cadvisor'	# 新增如下配置
      kubernetes_sd_configs:
      - role: node
      scheme: https
      tls_config:
        ca_file: /var/run/secrets/kubernetes.io/serviceaccount/ca.crt
        insecure_skip_verify: true
      bearer_token_file: /var/run/secrets/kubernetes.io/serviceaccount/token
      relabel_configs:
      - action: labelmap
        regex: __meta_kubernetes_node_label_(.+)
        replacement: $1
      - source_labels: [__meta_kubernetes_node_name]
        regex: (.+)
        replacement: /metrics/cadvisor # <nodeip>/metrics -> <nodeip>/metrics/cadvisor
        target_label: __metrics_path__
~~~

#### relabel_configs

如下配置，目的是修改暴露的 api 路径。

为了让 prometheus server 拉取监控指标是使用的 API 路径为：`https://<node ip>:10250/metrics/cadvisor`

~~~yaml
      - source_labels: [__meta_kubernetes_node_name]
        regex: (.+)
        replacement: /metrics/cadvisor # <nodeip>/metrics -> <nodeip>/metrics/cadvisor
        target_label: __metrics_path__
~~~

等一会后，待 pod 内更新了 cm 新配置。

```bash
curl -X POST "http://10.244.2.22:9090/-/reload"
```

> 10.244.2.22 是 prometheus pod 的 ip 地址



使用采集的容器指标计算每个 pod 对 cpu 的使用率

~~~
(sum(rate(container_cpu_usage_seconds_total{image!="",pod!=""}[1m])) by (namespace, pod))
/
(sum(container_spec_cpu_quota{image!="", pod!=""}) by(namespace, pod) / 100000) * 100
~~~

图形展示如下

![](graph.png)

