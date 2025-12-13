# 安装 Prometheus

安装 prometheus server



## 二进制安装

官网下载地址

- 最新版：https://prometheus.io/download
- 历史版本：https://github.com/prometheus/prometheus/tags



#### 下载 prometheus server

~~~bash
# 文件夹
mkdir /monitor && cd /monitor

# 下载压缩包
wget https://github.com/prometheus/prometheus/releases/download/v3.5.0/prometheus-3.5.0.linux-amd64.tar.gz

# 解压
tar xf prometheus-3.5.0.linux-amd64.tar.gz

# 移除安装包
mv prometheus-3.5.0.linux-amd64.tar.gz /tmp/

# 做个软连接方便后续升级
ln -s /monitor/prometheus-3.5.0.linux-amd64 /monitor/prometheus

# 创建tsdb数据目录
mkdir /monitor/prometheus/data
~~~

#### 添加系统服务

~~~bash
cat > /usr/lib/systemd/system/prometheus.service << 'EOF'
[Unit]
Description=prometheus server daemon

[Service]
Restart=on-failure
ExecStart=/monitor/prometheus/prometheus --config.file=/monitor/prometheus/prometheus.yml --storage.tsdb.path=/monitor/prometheus/data  --storage.tsdb.retention.time=30d  --web.enable-lifecycle 

[Install]
WantedBy=multi-user.target

EOF
~~~

#### 启动服务

~~~bash
systemctl daemon-reload 
systemctl enable prometheus
systemctl start prometheus
systemctl status prometheus

# 查看端口
netstat -tunalp |grep 9090

# 浏览器访问 WEB UI 界面
http://<节点的ip>:9090
~~~



#### 构建测试程序

下载并构建测试程序，作为被监控者。如下是一个 golang 实现的 web 应用，提供并对外暴露 `/metrics` 接口。

~~~bash
# 本地构建 golang 环境
yum install golang -y

# 下载源码包并构建 web 应用
git clone https://github.com/prometheus/client_golang.git
cd client_golang/examples/random

export GO111MODULE=on
export GOPROXY=https://goproxy.cn

# 构建。得到一个二进制命令random
go build

# 分别启动三个进程，对外暴露端口8080、8081、8082
./random -listen-address=:8080 &
./random -listen-address=:8081 &
./random -listen-address=:8082 &
~~~



#### 添加监控项

上述三个进程都对外暴漏了 `/metrics` 接口，并且数据格式遵循 prometheus 规范，所以我们可以在 `prometheus.yml` 中添加监控项。

~~~bash
scrape_configs:
  - job_name: 'example-random'
    scrape_interval: 5s
    static_configs:
      - targets: ['192.168.10.81:8080', '192.168.10.81:8081']
        labels:
          group: 'production'
      - targets: ['192.168.10.81:8082']
        labels:
          group: 'canary'
~~~

重启 prometheus

~~~bash
systemctl restart prometheus
~~~





## k8s 集群安装 Prometheus server

安装 Prometheus server 到 k8s 集群。



#### 创建 namespace

~~~yaml
# prometheus-ns.yaml
apiVersion: v1
kind: Namespace
metadata:
  name: monitor
  labels:
    name: monitor
~~~



#### 创建 configmap

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
~~~



#### 创建 PV & PVC

本地存储使用

~~~yaml
# prometheus-pv-pvc.yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: prometheus-local
  labels:
    app: prometheus
spec:
  accessModes:
    - ReadWriteOnce
  capacity:
    storage: 5Gi
  storageClassName: local-storage
  local:
    path: /data/k8s/prometheus	# 储存数据路径
  nodeAffinity:
    required:
      nodeSelectorTerms:
        - matchExpressions:
            - key: kubernetes.io/hostname
              operator: In
              values:
                - k8s-master-03
  persistentVolumeReclaimPolicy: Retain
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: prometheus-data
  namespace: monitor
spec:
  selector:
    matchLabels:
      app: prometheus
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 5Gi
  storageClassName: local-storage
~~~



#### 创建 RBAC 相关的资源

~~~yaml
# prometheus-rbac.yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: prometheus
  namespace: monitor
---
apiVersion: rbac.authorization.k8s.io/v1
# 由于我们要获取的资源信息，在每一个 namespace 下面都有可能存在
# 所以我们这里使用的是 ClusterRole 的资源对象
kind: ClusterRole
metadata:
  name: prometheus
rules:
  - apiGroups:
      - ''
    resources:
      - nodes
      - services
      - endpoints
      - pods
      - nodes/proxy
    verbs:
      - get
      - list
      - watch
  - apiGroups:
      - 'extensions'
    resources:
      - ingresses
    verbs:
      - get
      - list
      - watch
  - apiGroups:
      - ''
    resources:
      - configmaps
      - nodes/metrics
    verbs:
      - get
  - nonResourceURLs: # 用来对非资源型 metrics 进行操作的权限声明
      - /metrics
    verbs:
      - get
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: prometheus
roleRef:
  apiGroup: rbac.authorization.k8s.io
  # 由于我们要获取的资源信息，在每一个 namespace 下面都有可能存在
  # 所以我们这里使用的是 ClusterRole 的资源对象
  kind: ClusterRole
  name: prometheus
subjects:
  - kind: ServiceAccount
    name: prometheus
    namespace: monitor
~~~



#### 创建 deployment

~~~bash
# prometheus-deploy.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: prometheus
  namespace: monitor
  labels:
    app: prometheus
spec:
  selector:
    matchLabels:
      app: prometheus
  template:
    metadata:
      labels:
        app: prometheus
    spec:
      # 赋予在pod 操作 k8s 集群操作的权限
      serviceAccountName: prometheus
      # 设置容器内以 root账号运行
      securityContext:
        runAsUser: 0
      volumes:
        - name: data
          persistentVolumeClaim:
            claimName: prometheus-data
        - configMap:
            name: prometheus-config
          name: config-volume
      containers:
        # 如果无法拉取image那就换成国内镜像
        - image: prom/prometheus:v2.53.0
          name: prometheus
          args:
            - '--config.file=/etc/prometheus/prometheus.yml'
            - '--storage.tsdb.path=/prometheus' # 指定tsdb数据路径
            - '--storage.tsdb.retention.time=24h'
            - '--web.enable-admin-api' # 控制对admin HTTP API的访问，其中包括删除时间序列等功能
            - '--web.enable-lifecycle' # 支持热更新，直接执行localhost:9090/-/reload立即生效
          ports:
            - containerPort: 9090
              name: http
          volumeMounts:
            - mountPath: '/etc/prometheus'
              name: config-volume
            - mountPath: '/prometheus'
              name: data
          resources:
            requests:
              cpu: 100m
              memory: 512Mi
            limits:
              cpu: 100m
              memory: 512Mi
~~~



#### 创建 prometheus svc

使用 NodePort 类型的 svc 对外暴露服务。对外暴露端口号为：9090

~~~yaml
# prometheus-svc.yaml
apiVersion: v1
kind: Service
metadata:
  name: prometheus
  namespace: monitor
  labels:
    app: prometheus
spec:
  selector:
    app: prometheus
  type: NodePort
  ports:
    - name: web
      port: 9090
      targetPort: 9090
~~~



#### 应用资源

~~~bash
# 在pv所亲和的节点上创建
mkdir -p /data/k8s/prometheus

# 应用资源
kubectl apply -f prometheus-ns.yaml
kubectl apply -f prometheus-cm.yaml
kubectl apply -f prometheus-pv-pvc.yaml
kubectl apply -f prometheus-rbac.yaml 
kubectl apply -f prometheus-deploy.yaml 
kubectl apply -f prometheus-svc.yaml
~~~



#### 查看 svc & pod

~~~bash
[root@k8s-master-01 ~]# kubectl -n monitor get pods -o wide
NAME                          READY   STATUS    RESTARTS   AGE     IP            NODE            NOMINATED NODE   READINESS GATES
prometheus-7b8c8c7669-t296b   1/1     Running   0          2m31s   10.244.2.22   k8s-master-03   <none>           <none>
[root@k8s-master-01 ~]# 
[root@k8s-master-01 ~]# 
[root@k8s-master-01 ~]# 
[root@k8s-master-01 ~]# kubectl -n monitor get svc
NAME         TYPE       CLUSTER-IP      EXTERNAL-IP   PORT(S)          AGE
prometheus   NodePort   10.103.207.14   <none>        9090:30266/TCP   20m
~~~



#### 访问 web ui

k8s 集群内任意一个节点的 物理 ip + svc 的 nodeport 

浏览器访问：http://192.168.10.81:30266



#### 添加监控项

更新 prometheus-cm.yaml 文件

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
      - job_name: 'example-random'
        scrape_interval: 5s
        static_configs:
          - targets: ['192.168.10.81:8080', '192.168.10.81:8081']
            labels:
              group: 'production'
          - targets: ['192.168.10.81:8082']
            labels:
              group: 'canary'
~~~



#### 重载服务

~~~bash
kubectl apply -f prometheus-cm.yaml
~~~

等待一会，手动触发 prometheus 的热更新

~~~bash
curl -X POST "http://10.244.0.32:9090/-/reload"
~~~

>10.244.0.32 是 prometheus pod 的 ip 地址



