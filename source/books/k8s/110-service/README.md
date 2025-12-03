# service

service 是一种智能负载均衡器，用来代理pod的服务的，解决 Pod 动态变化的访问问题。

解决的问题：

- pod 可能重启，迁移，ip 变化。
- 将流量分发到多个 pod，动态上下线。
- 为应用提供稳定的访问入口。

svc 通过标签来选中被管理的 pod，能够感知到 pod 的副本变化，动态感知 pod 的 ip 地址变化、pod 内服务的变化（就绪探针检测成功加入 endpoints，检测失败移除 endpoints）。

## svc 负载均衡模式

svc负载均衡有两种模式：

- 基于iptables
- 基于ipvs（效率更高）

两种模式的相同点：

- 在所有安装有 kube-proxy 组件的 k8s 节点上都会有一模一样的转发规则，并且是动态感知。在任何一个节点都可以发起对 svc 的请求（svc 必须有 clusterip）

~~~alert type=note
请求 svc 的链路<br>
请求---svc--基于转发模式计算出应该访问的podip--》用网络插件例如vxlan模式来封包发送
~~~

强调：当 kube-proxy 以 ipvs 代理模式启动时，kube-proxy 将验证节点上是否安装了IPVS模块，如果未安装，则 kube-proxy 将回退到 iptables 代理模式。

注意：网络通信是靠网络插件实现的（例如flannel的vxlan模式），svc 只是用来做负载均衡不负责转发流量。



## svc 的类型

service 有四中类型，clusterip、nodeport、loadbalancer、exterlname



#### ClusterIp

这是默认的类型，仅在集群内可访问，会分配一个集群内部虚拟 IP

~~~yaml
# svc-clusterip-demo.yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx-svc
  labels:
    app: nginx-app
spec:
  selector:
    app: nginx  # 必须添加selector来匹配 Pod
  ports:
  - port: 8080     # 指定 Service 的端口
    protocol: TCP 
    targetPort: 80 # 转发到后端端口
  type: ClusterIP  # 服务类型，默认是ClusterIP
~~~

查看 svc

~~~bash
[root@k8s-master-01 test1]# kubectl get svc nginx-svc 
NAME        TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)    AGE
nginx-svc   ClusterIP   10.110.195.104   <none>        8080/TCP   85s
~~~

在 k8s 集群节点上访问：`http://10.110.195.104:8080` 即可访问被代理的 pod 服务。

在 k8s 内部的 pod 内支持 svc 的域名访问，域名由如下几部分组成

~~~
svc域名：<svc-name>.<namespace>.svc.cluster.local
~~~

因此，上述 svc 域名为：`nginx-svc.default.svc.cluster.local`

这一点可以在任意一个 pod 中查看到。这个域名只能在 Pod 内使用。

~~~bash
root@nginx-deployment-7584b6f84c-wvpbs:/# cat /etc/resolv.conf 
search default.svc.cluster.local svc.cluster.local cluster.local
nameserver 10.96.0.10
options ndots:5
~~~

#### NodePort Service

在 ClusterIP 基础上，在每个节点上开放端口，外部可通过 `节点IP:节点端口` 访问。

~~~yaml
# nodeport.yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx-svc
  labels:
    app: nginx-svc
spec:
  selector:
    app: nginx  # 必须添加selector来匹配 Pod
  ports:
  - port: 8080     # 指定 Service 的端口
    protocol: TCP 
    targetPort: 80 # 转发到后端端口
    nodePort: 30080  # 节点端口可选，范围 30000-32767
  type: NodePort 
~~~

查看 svc

~~~bash
[root@k8s-master-01 test1]# kubectl get svc nginx-svc 
NAME        TYPE       CLUSTER-IP     EXTERNAL-IP   PORT(S)          AGE
nginx-svc   NodePort   10.96.167.97   <none>        8080:30080/TCP   72s
~~~

在物理节点上访问 svc，使用集群内部任意一个物理节点 ip+nodeport

~~~bash
[root@k8s-master-01 test1]# curl http://172.16.143.22:30080 -I
HTTP/1.1 200 OK
Server: nginx/1.21.5
Date: Wed, 03 Dec 2025 07:01:44 GMT
Content-Type: text/html
Content-Length: 615
Last-Modified: Tue, 28 Dec 2021 15:28:38 GMT
Connection: keep-alive
ETag: "61cb2d26-267"
Accept-Ranges: bytes
~~~

#### LoadBalancer

向云提供商申请一个独立于 k8s 的负载均衡器，该负载均衡器会将流量转发到每个物理节点，形式为：<NodeIP>:NodePort
只要把 svc 的 `type=NodePort` 改为 `type=LoadBalancer` 即可，k8s 会自动帮我们创建一个对应的负载均衡器实例，并返回它的 ip 地址供外部客户端使用。

其他公有云提供商只要实现了支持此特性的驱动，则也可以达到上述目的。

~~~bash
[root@k8s-master-01 test1]# kubectl get svc nginx-svc 
NAME        TYPE           CLUSTER-IP      EXTERNAL-IP   PORT(S)          AGE
nginx-svc   LoadBalancer   10.100.28.147   <pending>     8080:30080/TCP   25s
~~~

#### ExternalName

将 svc 映射为一个外部的域名或 ip，通过 `externalName` 字段进行设置。

**如果外部服务有可以解析的域名，直接添加。**

~~~yaml
kind: Service
apiVersion: v1
metadata:
  name: my-service
spec:
  type: ExternalName
  externalName: www.baidu.com
~~~

查看 svc

~~~bash
[root@k8s-master-01 test1]# kubectl get svc my-service 
NAME         TYPE           CLUSTER-IP   EXTERNAL-IP     PORT(S)   AGE
my-service   ExternalName   <none>       www.baidu.com   <none>    57s
~~~

在 pod 内部测试

~~~bash
[root@k8s-master-01 test1]# kubectl run -it --rm my-pod --image=centos:7 -- bash
[root@my-pod /]# curl -H "Host: www.baidu.com" my-service -I
HTTP/1.1 200 OK
Cache-Control: private, no-cache, no-store, proxy-revalidate, no-transform
Content-Length: 0
Content-Type: text/html
Pragma: no-cache
Server: bfe
Date: Wed, 03 Dec 2025 07:33:19 GMT
~~~

**如果外部服务没有域名，而只有ip+port，那我们无法指定ExternalName，此时只能通过自建endpoint来实现**

注意：

- svc 的 clusterIP 必须设置为 None
- endpoint 的名字要与 svc 的名字保持一致

~~~yaml
apiVersion: v1
kind: Service
metadata:
  name: mysql-k8s
spec:
  type: ClusterIP
  clusterIP: None	# None
  ports:
    - name: port
      port: 13306

---
apiVersion: v1
kind: Endpoints
metadata:
  name: mysql-k8s # 名称必须和 Service 一致
subsets:
  - addresses:
      - ip: 192.168.71.114 # Service 将连接重定向到 endpoint
    ports:
      - name: port
        port: 3306
~~~



## Headless Service（无头服务）

用于 StatefulSet，DNS 返回所有 Pod IP

```yaml
apiVersion: v1
kind: Service
metadata:
  name: my-headless
spec:
  clusterIP: None  # 关键设置
  selector:
    app: stateful-app
  ports:
  - port: 80
    targetPort: 8080
```

#### 普通 svc 和 无头 svc

```bash
基于普通svc的访问流程
		普通svc的域名----coredns解析-----》svc的cluster ip----ipvs转发-》pod的endpoint
		
基于无头svc的访问流程
		无头svc的域名----coredns解析-----》所关联pod的ip地址作为dns解析记录来返回
```

~~~alert type=note
Node IP：物理节点的静态ip <br>
Pod IP：pod的ip地址，重启便会变化 <br>
Cluster IP：Service的ip地址，是新建的时候k8s分配的 <br>
~~~

## 会话保持

同一个客户端ip来访问，没有负载均衡的效果，用户代理到同一个pod

~~~yaml
spec:
  sessionAffinity: ClientIP  # 基于客户端 IP
  sessionAffinityConfig:
    clientIP:
      timeoutSeconds: 10800
~~~

## 获取客户端真实ip地址

~~~yaml
spec:
  externalTrafficPolicy: Local
  # Local: 保留客户端源 IP，流量仅转发到本节点 Pod
  # Cluster: 默认，可能丢失客户端源 IP
~~~

注意，设置了 `externalTrafficPolicy: Local` 后，意味着当你用某个物理节点的 ip+nodeport 来访问时，优先从该物理节点找pod。如果一直用某个物理节点的ip来访问，意味着没有负载均衡的效果，一直会访问固定pod，因为该参数的本意是尽量减少网络跳数。