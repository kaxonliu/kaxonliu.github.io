# grafana

Grafana 是一个开源的数据可视化和监控平台，主要用于**将各种数据源中的指标数据转化为直观的图表、仪表盘和警报**。

官网：https://grafana.com/

图像模版官网地址：https://grafana.com/grafana/dashboards/

Docker 镜像官方仓库：https://hub.docker.com/r/grafana/grafana



## 部署 grafana

下文使用原生 Kubernetes YAML 部署 grafana。

#### 创建持久化存储 pvc

~~~yaml
# grafana-pvc.yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: grafana-pvc
  namespace: monitor
  labels:
    app: grafana
spec:
  storageClassName: nfs-client	# 使用 nfs 网络存储
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 2Gi
~~~

因为使用了 nfs 网络存储，并且是基于 storage clsass 自动创建 pv，因此需要提前做好两件事情。

- 选择一个节点安装 nfs server，所有 k8s 节点安装 nfs client
- k8s 集群内安装 `nfs-subdir-external-provisioner` ，提供 storage clsass，用于自动创建 PV



#### 创建 grafana 服务

~~~yaml
# grafana-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: grafana
  namespace: monitor
spec:
  selector:
    matchLabels:
      app: grafana
  template:
    metadata:
      labels:
        app: grafana
    spec:
      volumes:
        - name: storage
          persistentVolumeClaim:
            claimName: grafana-pvc
      securityContext:
        runAsUser: 0  # 必须以root身份运行
      containers:
        - name: grafana
          image: grafana/grafana # 默认lastest最新，也可以指定版本grafana/grafana:10.4.4
          imagePullPolicy: IfNotPresent
          ports:
            - containerPort: 3000
              name: grafana
          env: # 配置 grafana 的管理员用户和密码的，
            - name: GF_SECURITY_ADMIN_USER
              value: admin
            - name: GF_SECURITY_ADMIN_PASSWORD
              value: admin321
          readinessProbe:
            failureThreshold: 10
            httpGet:
              path: /api/health
              port: 3000
              scheme: HTTP
            periodSeconds: 10
            successThreshold: 1
            timeoutSeconds: 30
          livenessProbe:
            failureThreshold: 3
            httpGet:
              path: /api/health
              port: 3000
              scheme: HTTP
            periodSeconds: 10
            successThreshold: 1
            timeoutSeconds: 1
          resources:
            limits:
              cpu: 150m
              memory: 512Mi
            requests:
              cpu: 150m
              memory: 512Mi
          volumeMounts:
            - mountPath: /var/lib/grafana	# 指定挂载点
              name: storage
~~~



#### 创建 grafana 服务

使用 nodeport 类型的 svc，可以直接使用 节点 ip 访问

~~~yaml
# grafana-svc.yaml
apiVersion: v1
kind: Service
metadata:
  name: grafana
  namespace: monitor
spec:
  type: NodePort
  ports:
    - port: 3000
  selector:
    app: grafana
~~~



#### 浏览器登录

先查看 grafana  svc 的 nodeport

~~~bash
[root@k8s-master-01 grafana]# kubectl -n monitor get svc
NAME         TYPE       CLUSTER-IP      EXTERNAL-IP   PORT(S)          AGE
grafana      NodePort   10.99.76.165    <none>        3000:30381/TCP   27m
prometheus   NodePort   10.103.207.14   <none>        9090:30266/TCP   2d17h
~~~

使用 node ip + nodeport 浏览器访问如下地址即可访问 grafana 

~~~bash
http://192.168.10.81:30381
~~~





## 使用 grafana

添加 prometheus 作为 data source

导入 dashsboard

~~~bash
当下可以参考的用的模版
node_exporter
可以使用模版8919、1860、15172(基于11074模板做的优化)

redis_exporter
推荐模版ID：14615

mysqld_exporter 
推荐模板ID：7362、 11323
配置主从复制推荐模板ID：11323
如果安装的是Mariadb推荐模板ID：13106

haproxy_exporter 
推荐模板ID： 367和2428

Nginx_exporter
推荐模版ID：14900
~~~







