# 监控服务

监控服务或者应用。



## 前置知识

监控 target 分为三大类：

- 服务/应用软件（如 redis）。这种需要使用第三方提供的 expoter（如 redis expoter）
- 物理节点。需要使用 node_exporter
- k8s 集群。使用服务发现。



监控 target 的配置有两种方式：

- 服务发现。一般用于 k8s 集群的监控。本质就是通过标签或注解信息进行筛选，符合规则都会被选出来作为 prometheus 周期性拉取的 target。
- 配置 /metrics 接口。Prometheus server 朝该接口发请求，则会返回固定格式的数据。注意：有现成的 /metrics 接口就可用，没有则部署 exporter 来采集指标并暴漏该接口。



## 服务自带 /metrics 接口

如果服务自带 `/metrics` 接口，直接监控。

比如 k8s 集群中的 coredns 服务。从 coredns 的 svc 中看到该服务为 prometheus 提供了 /metrics 接口，并监听在 9153 端口。

~~~bash
[root@k8s-master-01 ~]# 
[root@k8s-master-01 ~]# kubectl -n kube-system describe  svc kube-dns 
Name:                     kube-dns
Namespace:                kube-system
Labels:                   k8s-app=kube-dns
                          kubernetes.io/cluster-service=true
                          kubernetes.io/name=CoreDNS
Annotations:              prometheus.io/port: 9153		
                          prometheus.io/scrape: true
Selector:                 k8s-app=kube-dns
Type:                     ClusterIP
IP Family Policy:         SingleStack
IP Families:              IPv4
IP:                       10.96.0.10
IPs:                      10.96.0.10
Port:                     dns  53/UDP
TargetPort:               53/UDP
Endpoints:                10.244.0.14:53,10.244.2.19:53
Port:                     dns-tcp  53/TCP
TargetPort:               53/TCP
Endpoints:                10.244.0.14:53,10.244.2.19:53
Port:                     metrics  9153/TCP
TargetPort:               9153/TCP
Endpoints:                10.244.0.14:9153,10.244.2.19:9153
Session Affinity:         None
Internal Traffic Policy:  Cluster
Events:                   <none>
[root@k8s-master-01 ~]# 
[root@k8s-master-01 ~]# 
root@k8s-master-01 ~]# kubectl -n kube-system get svc
NAME       TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)                  AGE
kube-dns   ClusterIP   10.96.0.10   <none>        53/UDP,53/TCP,9153/TCP   25h
[root@k8s-master-01 ~]# 
[root@k8s-master-01 ~]# 
[root@k8s-master-01 ~]# kubectl -n kube-system get pod coredns-6d58d46f65-9f9hk -o yaml | grep metrics -B 1
    - containerPort: 9153
      name: metrics
~~~

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
      scrape_interval: 15s # Prometheus每隔15s就会从所有配置的目标端点抓取最新的数据
      scrape_timeout: 15s  # 某个抓取操作在 15 秒内未完成，会被视为超时，不会包含在最新的数据中。
      evaluation_interval: 15s #  # 每15s对告警规则进行计算
    scrape_configs:
      - job_name: "prometheus"
        static_configs:
          - targets: ["localhost:9090"]
      - job_name: "coredns"
        static_configs:
          - targets: ["kube-dns.kube-system.svc.cluster.local:9153"]
~~~

等一会后，待 pod 内更新了 cm 新配置。

~~~bash
curl -X POST "http://10.244.2.22:9090/-/reload"
~~~

>10.244.2.22 是 prometheus pod 的 ip 地址
>
>~~~bash
>[root@k8s-master-01 k8s-prometheus-yamls]# kubectl -n monitor get pods -o wide
>NAME                          READY   STATUS    RESTARTS   AGE    IP            NODE            NOMINATED NODE   READINESS GATES
>prometheus-7b8c8c7669-t296b   1/1     Running   0          128m   10.244.2.22   k8s-master-03   <none>           <none>
>~~~



## 安装 exporter

如果应用软件没有自带自带 /metrics 接口，需要安装对应的 exporter。

exporter官网地址：https://prometheus.io/docs/instrumenting/exporters/

下面以监控 redis 为里介绍如何使用 exporter 采集 redis 的监控数据。

#### 安装 redis

在 k8s 的一个节点（节点 IP：192.168.10.83）安装 redis

~~~bash
yum install redis -y
			
sed -ri 's/bind 127.0.0.1/bind 0.0.0.0/g' /etc/redis.conf
sed -ri 's/port 6379/port 16379/g' /etc/redis.conf


cat >> /etc/redis.conf << "EOF"
requirepass 123456
EOF

systemctl restart redis
systemctl status redis

# 测试
redis-cli -h 192.168.10.83 -p 16379 -a 123456
~~~



#### 安装 redis_exporter

到 exporter 官网查找 redis exporter，然后下载到部署 redis 的节点上。

redis exporter 本质上是一个二进制程序，下载解压后放到 path 路径下即可使用。

~~~bash
# 下载
wget https://github.com/oliver006/redis_exporter/releases/download/v1.80.1/redis_exporter-v1.80.1.linux-amd64.tar.gz

# 2、安装
tar xf redis_exporter-v1.80.1.linux-amd64.tar.gz
mv redis_exporter-v1.80.1.linux-amd64/redis_exporter /usr/bin/
~~~



#### 制作系统服务

把 redis_exporter 做成系统服务，后台运行，提供 /metrics 接口，暴露 9122 端口

~~~bash
cat > /usr/lib/systemd/system/redis_exporter.service << 'EOF'
[Unit]
Description=Redis Exporter
Wants=network-online.target
After=network-online.target

[Service]
User=root
Group=root
Type=simple
ExecStart=/usr/bin/redis_exporter --redis.addr=redis://127.0.0.1:16379 --redis.password=123456 --web.listen-address=0.0.0.0:9122 --exclude-latency-histogram-metrics 

[Install]
WantedBy=multi-user.target

EOF
~~~

然后启动服务

~~~bash
systemctl restart redis_exporter
systemctl status redis_exporter
~~~



#### 增加监控项目

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
      - job_name: "redis-server"  # 添加这一条
        static_configs:
          - targets: ["192.168.10.83:9122"]
~~~

等一会后，待 pod 内更新了 cm 新配置。

~~~bash
curl -X POST "http://10.244.2.22:9090/-/reload"
~~~

>10.244.2.22 是 prometheus pod 的 ip 地址



#### 补充

如果你的 redis-server 跑在 k8s 中，那我们通常不会像上面一样裸部署一个redis_exporter，而是会以`sidecar` 的形式将 redis_exporter 和主应用 redis_server 部署在同一个 Pod 中。

~~~yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: redis
  namespace: monitor
spec:
  selector:
    matchLabels:
      app: redis
  template:
    metadata:
      labels:
        app: redis
    spec:
      containers:
      - name: redis
        image: redis:4
        resources:
          requests:
            cpu: 100m
            memory: 100Mi
        ports:
        - containerPort: 6379
      - name: redis-exporter
        image: oliver006/redis_exporter:latest
        resources:
          requests:
            cpu: 100m
            memory: 100Mi
        ports:
        - containerPort: 9121
---
apiVersion: v1
kind: Service
metadata:
  name: redis
  namespace: monitor
spec:
  selector:
    app: redis
  ports:
  - name: redis
    port: 6379
    targetPort: 6379
  - name: prom
    port: 9121
    targetPort: 9121
    
# 可以用该 svc 的 clusterip 结合9121端口来访问/metrics接口
# 添加监控项，直接用svc名字即可， 更新 prometheus-cm.yaml 的配置文件如下
- job_name: 'redis'
  static_configs:
    - targets: ['redis:9121']
~~~

