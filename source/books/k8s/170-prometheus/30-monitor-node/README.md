# 监控物理节点

指标： cpu、load、disk、memory 等指标



## 物理节点不属于 k8s 集群

物理节点不属于 k8s 集群，那就安装并启动 node_exporter，每个节点都要手动做一次。

exporter官网地址：https://prometheus.io/docs/instrumenting/exporters/



#### 安装 node_exporter

到 exporter 官网查找 node exporter，然后下载到部署到待监控的节点上。

node exporter 本质上是一个二进制程序，下载解压后放到 path 路径下即可使用。

```bash
# 下载
wget https://github.com/prometheus/node_exporter/releases/download/v1.10.2/node_exporter-1.10.2.linux-amd64.tar.gz

# 2、安装
tar xf node_exporter-1.10.2.linux-amd64.tar.gz
mv node_exporter-1.10.2.linux-amd64/node_exporter /usr/bin/
```

#### 制作系统服务

把 node_exporter 做成系统服务，后台运行，提供 /metrics 接口，暴露 9122 端口

```bash
cat > /usr/lib/systemd/system/node_exporter.service << 'EOF'
[Unit]
Description=Node Exporter
Wants=network-online.target
After=network-online.target

[Service]
User=root
Group=root
Type=simple
ExecStart=/usr/bin/node_exporter --web.listen-address=0.0.0.0:9100

[Install]
WantedBy=multi-user.target

EOF
```

然后启动服务

```bash
systemctl restart node_exporter
systemctl status node_exporter
```

#### 增加监控项目

在 prometheus 的配置中增加下述 target，然后 apply -f

```yaml
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
      - job_name: "nodes"  # 添加这一条
        static_configs:
          - targets: ["192.168.10.83:9100"]
```

等一会后，待 pod 内更新了 cm 新配置。

```bash
curl -X POST "http://10.244.2.22:9090/-/reload"
```

> 10.244.2.22 是 prometheus pod 的 ip 地址



##  k8s 集群中的物理节点

使用 pod 的方式部署 node exporter，并且配合 DaemonSet 可以采集所有 k8s 节点的监控数据。

#### 创建 node exporter

~~~yaml
# prometheus-node-exporter.yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: node-exporter
  namespace: monitor
  labels:
    app: node-exporter
spec:
  selector:
    matchLabels:
      app: node-exporter
  template:
    metadata:
      labels:
        app: node-exporter
    spec:
      # 由于我们要获取到的数据是主机的监控指标数据，而我们的 node-exporter 是运行在容器中的，
      # 所以我们必须确保在容器内可以看到宿主机上的一些状态信息，为此我们做了两件事
      # 第一：配置了如下三条Pod安全策略为true，确保在pod内可以访问到宿主机上的PID namespace、IPC namespace 以及主机网络
      hostPID: true
      hostIPC: true
      hostNetwork: true
      # 第二：将主机的 /dev、/proc、/sys这些目录挂载到容器中，物理节点的很多数据都在这些目录中
      volumes:
      - name: proc
        hostPath:
          path: /proc  # top命令查看到的cpu状态就来自于/proc/stat, 命令free -h看到的内存状态就来自于/proc/meminfo
      - name: dev
        hostPath:
          path: /dev
      - name: sys
        hostPath:
          path: /sys
      - name: root
        hostPath:
          path: /
      # 此外由于我们集群使用的是 kubeadm 搭建的，所以如果希望 master 节点也一起被监控，则需要添加相应的容忍
      # 下面这条容忍设置的含义是：无论节点上存在何种 Taints，该 Pod 都会容忍这些 Taints，从而能够分布到所有节点上。
      # 这是确保监控工具 DaemonSet 可被部署到 Kubernetes 集群所有节点上的常见做法。
      tolerations:
      - operator: 'Exists'
      # 我们所有节点都有这个标签
      nodeSelector:
        kubernetes.io/os: linux
      containers:
      - name: node-exporter
        image: quay.io/prometheus/node-exporter:latest
        args: # 选项定制详见：https://github.com/prometheus/node_exporter
        - --web.listen-address=$(HOSTIP):9100
        - --path.procfs=/host/proc
        - --path.sysfs=/host/sys
        - --path.rootfs=/host/root
        - --no-collector.hwmon # 禁用不需要的一些采集器
        - --no-collector.nfs
        - --no-collector.nfsd
        - --no-collector.nvme
        - --no-collector.dmi
        - --no-collector.arp
        - --collector.filesystem.ignored-mount-points=^/(dev|proc|sys|var/lib/containerd/.+|/var/lib/docker/.+|var/lib/kubelet/pods/.+)($|/)
        - --collector.filesystem.ignored-fs-types=^(autofs|binfmt_misc|cgroup|configfs|debugfs|devpts|devtmpfs|fusectl|hugetlbfs|mqueue|overlay|proc|procfs|pstore|rpc_pipefs|securityfs|sysfs|tracefs)$
        ports:
        - containerPort: 9100
        env:
        - name: HOSTIP
          valueFrom:
            fieldRef:
              fieldPath: status.hostIP
        resources:
          requests:
            cpu: 150m
            memory: 180Mi
          limits:
            cpu: 150m
            memory: 180Mi
        securityContext:
          runAsNonRoot: true
          runAsUser: 65534
        volumeMounts:
        - name: proc
          mountPath: /host/proc
        - name: sys
          mountPath: /host/sys
        - name: root
          mountPath: /host/root
          mountPropagation: HostToContainer
          readOnly: true
~~~



#### 手动增加监控项目

在 prometheus 的配置中增加下述 target，然后 apply -f

```yaml
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
      # - job_name: "nodes"
      #  static_configs:
      #    - targets: ["192.168.10.83:9100"]
      - job_name: "node-exporter"  # 添加这一条
        static_configs:
          - targets: ["192.168.10.81:9100","192.168.10.82:9100","192.168.10.83:9100","192.168.10.84:9100"]
```

等一会后，待 pod 内更新了 cm 新配置。

```bash
curl -X POST "http://10.244.2.22:9090/-/reload"
```

> 10.244.2.22 是 prometheus pod 的 ip 地址





## 自动发现 k8s 节点

如果监控 k8s 集群内的物理节点，没必要手动添加监控项目，可以使用服务发现的方式自动实现。

在 prometheus 的配置中修改下述 target，然后 apply -f，后面再新加节点就不需要手动添加监控项目了。

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
    - job_name: 'sd-node-exporter'
      kubernetes_sd_configs:
      - role: node
      relabel_configs:
      - source_labels: [__address__]  # 获取到地址
        regex: '(.*):10250' # 用正则regex匹配上面获取到的地址，(.*)获取到的是ip地址部分，加括号代表存入一个正则分组
        replacement: '${1}:9100' # 取出上面正则分组的内容即ip地址，然后后面拼一个9100端口，即完成端口替换
        target_label: __address__ # 替换的目标
        action: replace           # 动作为replace
      - action: labelmap # 增加
        regex: __meta_kubernetes_node_label_(.+)
~~~

#### relabel_configs

如下配置，prometheus 默认向 `节点 ip:9100/metrics` 发请求获取监控指标。但是 k8s 节点上默认暴露出来的端口是 10250，因此使用 `relabel_configs` 的方式替换端口。

~~~yaml
- job_name: 'sd-node-exporter'
  kubernetes_sd_configs:
  - role: node
~~~

另外，为了在 prometheus 上展示的 node 标签信息和 k8s 上一致。使用 relabel_congis 做了如下修改

~~~yaml
- action: labelmap # 修正节点标签
  regex: __meta_kubernetes_node_label_(.+)
~~~

等一会后，待 pod 内更新了 cm 新配置。

```bash
curl -X POST "http://10.244.2.22:9090/-/reload"
```

> 10.244.2.22 是 prometheus pod 的 ip 地址



## 自动发现 kubelet

每个 k8s 节点都有一个 kubelct，负责维护节点信息，因此也有必要通过监控 kubelet 进而监控节点信息。

使用服务发现功能，自动发现 kubelet，类似服务发现 node

在 prometheus 的配置中修改下述 target，然后 apply -f，后面再新加节点就不需要手动添加监控项目了。

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
    - job_name: 'sd-node-exporter'
      kubernetes_sd_configs:
      - role: node
      relabel_configs:
      - source_labels: [__address__]  # 获取到地址
        regex: '(.*):10250' # 用正则regex匹配上面获取到的地址，(.*)获取到的是ip地址部分，加括号代表存入一个正则分组
        replacement: '${1}:9100' # 取出上面正则分组的内容即ip地址，然后后面拼一个9100端口，即完成端口替换
        target_label: __address__ # 替换的目标
        action: replace           # 动作为replace
      - action: labelmap # 增加
        regex: __meta_kubernetes_node_label_(.+)
    - job_name: 'kubelet'
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
~~~

等一会后，待 pod 内更新了 cm 新配置。

```bash
curl -X POST "http://10.244.2.22:9090/-/reload"
```

> 10.244.2.22 是 prometheus pod 的 ip 地址



