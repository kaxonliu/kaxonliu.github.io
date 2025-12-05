# coredns 和 localdns

## coredns

CoreDNS 是 Kubernetes 中**默认的、云原生的 DNS 服务器**，从 Kubernetes 1.11 版本开始取代了之前的 kube-dns。

coredns 就是用来负责域名解析的组件，域名包括两种：svc 的 fqdn 域名、pod 的域名。

- svc 的 fqdn 域名

~~~
svc的名字.名称空间.svc.cluster.local.
~~~

- 名字固定的 pod（statefulset创建的 pod；裸 pod） + 无头服务

~~~
pod的名字.无头svc的名字.名称空间.svc.cluster.local.
~~~

k8s 非常擅长跑微服务，在微服务架构中必须要有的核心组件：服务注册与发现组件，而 coredns+svc 二者结合就实现服务注册与发现的效果。

- svc 负责与被管理的 pod 动态建立关系 + pod 里要设置就绪探针做健康检查
- coredns 负责提供一个可用的固定的域名

服务发现组件有很多种，但很明显，coredns 就是服务发现组件的一种，它对外提供的稳定访问的名字 即域名。

DNS 服务不是一个独立的系统服务，而是作为一种 addon 插件而存在，现在比较推荐的两个插件： kube-dns 和 CoreDNS，实际上在比较新点的版本中已经默认是 CoreDNS 了，因为 kube-dns 默认一 个 Pod 中需要 3 个容器配合使用，CoreDNS 只需要一个容器即可，我们在前面使用 kubeadm 搭建集 群的时候直接安装的就是 CoreDNS 插件：

~~~bash
[root@k8s-master-01 ~]# kubectl -n kube-system get pods -l k8s-app=kube-dns -o wide
NAME                       READY   STATUS    RESTARTS   AGE   IP           NODE            NOMINATED NODE   READINESS GATES
coredns-6d58d46f65-f9w8m   1/1     Running   0          17d   10.244.0.2   k8s-master-01   <none>           <none>
coredns-6d58d46f65-nf5hp   1/1     Running   0          17d   10.244.0.3   k8s-master-01   <none>           <none>
[root@k8s-master-01 ~]# 
[root@k8s-master-01 ~]# 
[root@k8s-master-01 ~]# 
[root@k8s-master-01 ~]# kubectl -n kube-system get svc kube-dns
NAME       TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)                  AGE
kube-dns   ClusterIP   10.96.0.10   <none>        53/UDP,53/TCP,9153/TCP   17d
[root@k8s-master-01 ~]# 
[root@k8s-master-01 ~]# # kube-dns svc 代理两个 coredns pod
[root@k8s-master-01 ~]# kubectl -n kube-system describe svc kube-dns
Name:              kube-dns
Namespace:         kube-system
Labels:            k8s-app=kube-dns
                   kubernetes.io/cluster-service=true
                   kubernetes.io/name=CoreDNS
Annotations:       prometheus.io/port: 9153
                   prometheus.io/scrape: true
Selector:          k8s-app=kube-dns
Type:              ClusterIP
IP Family Policy:  SingleStack
IP Families:       IPv4
IP:                10.96.0.10
IPs:               10.96.0.10
Port:              dns  53/UDP
TargetPort:        53/UDP
Endpoints:         10.244.0.2:53,10.244.0.3:53
Port:              dns-tcp  53/TCP
TargetPort:        53/TCP
Endpoints:         10.244.0.2:53,10.244.0.3:53
Port:              metrics  9153/TCP
TargetPort:        9153/TCP
Endpoints:         10.244.0.2:9153,10.244.0.3:9153
Session Affinity:  None
Events:            <none>
~~~

pod 内默认使用的 nameserver 就是 coredns svc 的 clusterip 地址

~~~bash
[root@k8s-master-01 ~]# kubectl run -it test --rm --image=nginx -- bash
If you don't see a command prompt, try pressing enter.
root@test:/# 
root@test:/# cat /etc/resolv.conf 
search default.svc.cluster.local svc.cluster.local cluster.local
nameserver 10.96.0.10
options ndots:5
~~~

#### 解析记录

~~~
1、普通的 Service：会生成 servicename.namespace.svc.cluster.local 的域名，会解析到
Service 对应的 ClusterIP 上，在 Pod 之间的调用可以简写成 servicename.namespace，如果处于
同一个命名空间下面，甚至可以只写成 servicename 即可访问
2、Headless Service：无头服务，就是把 clusterIP 设置为 None 的，会被解析为指定 Pod 的 IP
列表，同样还可以通过 podname.servicename.namespace.svc.cluster.local 访问到具体的某一个
Pod。
~~~

## 添加 coredns 自定义解析记录

~~~bash
# 1、
kubectl edit configmap coredns -n kube-system
# 2、编辑为以下内容(添加了hosts块)：
apiVersion: v1
data:
  Corefile: |
    .:53 {
        errors
        health {
           lameduck 5s
        }
        ready
        kubernetes cluster.local in-addr.arpa ip6.arpa {
           pods insecure
           fallthrough in-addr.arpa ip6.arpa
           ttl 30
        }
        # 手动添加如下自定义解析记录
        hosts {
           11.11.11.11 www.test11.com
           11.11.11.22 www.test22.com
           fallthrough in-addr.arpa ip6.arpa
        }
        prometheus :9153
        forward . /etc/resolv.conf {
           max_concurrent 1000
        }
        cache 30
        loop
        reload
        loadbalance
    }
kind: ConfigMap
metadata:
  creationTimestamp: "2025-11-18T05:44:28Z"
  name: coredns
  namespace: kube-system
  resourceVersion: "255"
  uid: 070b336c-035c-4057-a58b-f9122e013e4b

# 3、重启coredns
kubectl scale deployment coredns -n kube-system --replicas=0
kubectl scale deployment coredns -n kube-system --replicas=2

# 4、测试
➜ kubectl run -ti test --image=centos:7 --rm /bin/sh
sh-4.4#
sh-4.4# ping www.test11.com
PING www.test11.com (11.11.11.11) 56(84) bytes of data.
~~~

## 为 pod 定制解析记录

强调： 想让 pod 拥有可解析到 pod ip 的 fqdn 域名，必须满足两个前提：

1. pod的名字固定（两种方式可以得到名字固定的 pod，一种是用 statefulset 创建 pod，另外一种自己 创建裸 pod）
2. 必须创建无头服务来选中 pod

对于 statefulset 控制器创建的无头服务 headless svc，k8s 会为自动每个 pod 创建一个解析记录 

~~~
pod名字-0.无头服务svc名.default.svc.cluster.local
pod名字-1.无头服务svc名.default.svc.cluster.local
~~~

并且 pod 的名字、无头 svc 名都是固定的，所以上面的名字都是固定的，你用上面的名字可以直接解析到 pod的ip地址，所以这套方案整体很ok。

用 deployment 控制器创建 pod + 无头服务，这是不行的。



单独创建pod

~~~yaml
apiVersion: v1
kind: Pod
metadata:
  name: busybox1
  labels:
    name: busybox # 这个标签两个pod需要一样，方便后续svc统一选中
spec:
  hostname: busybox-1
  subdomain: default-subdomain # 这里的名字就是一会要创建出的无头服务的名字
  containers:
  - image: nginx
    command: ["sleep", "10000"]
    name: busybox
~~~

创建无头 svc

~~~yaml
apiVersion: v1
kind: Service
metadata:
  name: default-subdomain # 名字与subdomain保持一致
spec:
  selector:
    name: busybox
  clusterIP: None
~~~

在宿主机上可以使用 dig 命令测试域名的解析记录。

~~~bash
[root@k8s-master-01 test3]# dig @10.96.0.10 busybox-1.default-subdomain.default.svc.cluster.local

; <<>> DiG 9.11.4-P2-RedHat-9.11.4-26.P2.el7_9.16 <<>> @10.96.0.10 busybox-1.default-subdomain.default.svc.cluster.local
; (1 server found)
;; global options: +cmd
;; Got answer:
;; WARNING: .local is reserved for Multicast DNS
;; You are currently testing what happens when an mDNS query is leaked to DNS
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 54720
;; flags: qr aa rd; QUERY: 1, ANSWER: 1, AUTHORITY: 0, ADDITIONAL: 1
;; WARNING: recursion requested but not available

;; OPT PSEUDOSECTION:
; EDNS: version: 0, flags:; udp: 4096
;; QUESTION SECTION:
;busybox-1.default-subdomain.default.svc.cluster.local. IN A

;; ANSWER SECTION:
busybox-1.default-subdomain.default.svc.cluster.local. 30 IN A 10.244.1.196

;; Query time: 13 msec
;; SERVER: 10.96.0.10#53(10.96.0.10)
;; WHEN: Fri Dec 05 22:28:24 CST 2025
;; MSG SIZE  rcvd: 151

~~~

上述的 `ANSWER SECTION` 看到域名的解析 ip，就是 svc 代理的一个 pod ip

~~~bash
[root@k8s-master-01 test3]# kubectl get pods -o wide
NAME       READY   STATUS    RESTARTS   AGE     IP             NODE          NOMINATED NODE   READINESS GATES
busybox1   1/1     Running   0          2m55s   10.244.1.196   k8s-node-01   <none>           <none>
~~~



官方文档中有一段 Pod’s hostname and subdomain fields 说明：

>Pod 规范中包含一个可选的 hostname 字段，可以用来指定 Pod 的主机名。当这个字段被设置 时，它将优先于 Pod 的名字成为该 Pod 的主机名。举个例子，给定一个 hostname 设置为 "myhost" 的 Pod， 该 Pod 的主机名将被设置为 "my-host"。Pod 规约还有一个可选的 subdomain 字段，可以用来指定 Pod 的子域名。举个例子，某 Pod 的 hostname 设置为 “foo”，subdomain 设置为 “bar”， 在名字空间 “my-namespace” 中对应的完全限定域名为 “foo.bar.mynamespace.svc.cluster-domain.example”。



## Pod 的 DNS 策略

DNS 策略可以单独对 Pod 进行设定，目前 Kubernetes 支持以下特定 Pod 的 DNS 策略。这些策略可以在 Pod 规范中的 `dnsPolicy ` 字段设置：

- Default：自己的 pod 中 `/etc/resolv.conf` 的内容与自己所在的宿主机上的 `/etc/resolv.conf` 内容一致。
- ClusterFirst：这是默认的 DNS 策略。优先使用集群内的 dns，即 coredns
- ClusterFirstWithHostNet：有 hostNet 网络的 pod，但是优先使用 clusterfirst 的

~~~yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx
  namespace: default
spec:
  hostNetwork: true
  dnsPolicy: ClusterFirstWithHostNet
  containers:
  - image: nginx:latest
    command: ["sleep", "1000"]
    name: nginx
    
# 先注释掉上面的dnsPolicy，在hostNetwork为true的情况下
# 默认pod内的/etc/resolv.conf的内容与宿主机一模一样

# 如果设置 dnsPolicy: ClusterFirstWithHostNet
# 即便hostNetwork为true，依然让pod还是优先用k8s的coredne
~~~

- None：不让 k8s 生成 `/etc/resolve.conf` 的内容，在 yaml 中自己定义

~~~yaml
apiVersion: v1
kind: Pod
metadata:
  namespace: default
  name: dns-example
spec:
  containers:
  - name: test
    image: nginx
  dnsPolicy: 'None' # 表示不使用集群的DNS设置，而是使用自定义的
  dnsConfig:
    nameservers: # 指定dns查询时应该使用的DNS服务器ip地址
      - 1.2.3.4
    searches:
      - ns1.svc.cluster-domain.example # 指定搜索域列表，当只提供主机名而不是完整域名时
# 这些搜索域会被用来补全完整域名
      - my.dns.search.suffix
    options: # 设置额外的选项
      - name: ndots
        value: '5' # 如果域名中点的数量少于 ndots 指定的值，DNS 客户端将按照 search 选项
中指定的搜索域名后缀顺序，依次尝试将这些后缀加到域名的末尾，并进行 DNS 查询。如果本身就有五个
点，则直接搜索，一个完整的fqdn域名正好5个点：kube-dns.kube-system.svc.cluster.local. 末
尾还有一个点
     - name: edns0 # 启用选项 edns0，这通常用于启用 DNS 扩展协议
~~~

创建上面的 Pod 后，容器 test 会在其 /etc/resolv.conf 文件中获取以下内容

~~~bash
nameserver 1.2.3.4
search ns1.svc.cluster-domain.example my.dns.search.suffix
options ndots:2 edns0
~~~



## 使用本地 DNS 缓存

集群规模大的情况，CoreDNS 可能会出现超时 5s 的情况（dns 请求超时，客户端默认策略时等待 5s 自动 重试，如果重试成功，我们看到的现象就是 dns 请求有 5s 的延时），这对我们影响很大，因为 k8s 集群都依赖它来实现域名解析。

超时的原因是因为：无论你用的是iptables模式还是ipvs模式，它们的底层都会用到 conntrack内核模块 来组织dns的udp查询包，高并发场景，会出现conntrack冲突问题，导致一些dns请求包被丢掉，从而导致超时。

尝试使用 NodeLocal DNSCache 来提升 DNS 的性能和可靠性。

#### NodeLocal DNSCache部署

部署 NodeLocal DNSCache 采用的是 DaemonSet 控制器，会确保每个节点都有一个 NodeLocal DNSCache 实例，来提升集群 DNS 的性能和可靠性。

NodeLocal DNSCache 会在每个物理节点上缓存 DNS 解析，这可以缩短 DNS 查找的延迟时间、使 DNS 查找时间更加一致，以及减少发送到 kube-dns 的 DNS 查询次数。

要安装 NodeLocal DNSCache 也非常简单，直接获取官方的资源清单即可：

~~~bash
# 官方下载地址：wget
https://github.com/kubernetes/kubernetes/raw/master/cluster/addons/dns/nodelocal
dns/nodelocaldns.yaml
# 如果无法wget，你直接用浏览器打开拷贝就行了
#下载完毕后，要进行一些编辑配
~~~

~~~yaml
# nodelocaldns.yaml
# Copyright 2018 The Kubernetes Authors.
#
# Licensed under the Apache License, Version 2.0 (the "License");
# you may not use this file except in compliance with the License.
# You may obtain a copy of the License at
#
#     http://www.apache.org/licenses/LICENSE-2.0
#
# Unless required by applicable law or agreed to in writing, software
# distributed under the License is distributed on an "AS IS" BASIS,
# WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
# See the License for the specific language governing permissions and
# limitations under the License.
#

apiVersion: v1
kind: ServiceAccount
metadata:
  name: node-local-dns
  namespace: kube-system
  labels:
    kubernetes.io/cluster-service: "true"
    addonmanager.kubernetes.io/mode: Reconcile
---
apiVersion: v1
kind: Service
metadata:
  name: kube-dns-upstream
  namespace: kube-system
  labels:
    k8s-app: kube-dns
    kubernetes.io/cluster-service: "true"
    addonmanager.kubernetes.io/mode: Reconcile
    kubernetes.io/name: "KubeDNSUpstream"
spec:
  ports:
  - name: dns
    port: 53
    protocol: UDP
    targetPort: 53
  - name: dns-tcp
    port: 53
    protocol: TCP
    targetPort: 53
  selector:
    k8s-app: kube-dns
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: node-local-dns
  namespace: kube-system
  labels:
    addonmanager.kubernetes.io/mode: Reconcile
data:
  Corefile: |
    __PILLAR__DNS__DOMAIN__:53 {
        errors
        cache {
                success 9984 30
                denial 9984 5
        }
        reload
        loop
        bind __PILLAR__LOCAL__DNS__ __PILLAR__DNS__SERVER__
        forward . __PILLAR__CLUSTER__DNS__ {
                force_tcp
        }
        prometheus :9253
        health __PILLAR__LOCAL__DNS__:8080
        }
    in-addr.arpa:53 {
        errors
        cache 30
        reload
        loop
        bind __PILLAR__LOCAL__DNS__ __PILLAR__DNS__SERVER__
        forward . __PILLAR__CLUSTER__DNS__ {
                force_tcp
        }
        prometheus :9253
        }
    ip6.arpa:53 {
        errors
        cache 30
        reload
        loop
        bind __PILLAR__LOCAL__DNS__ __PILLAR__DNS__SERVER__
        forward . __PILLAR__CLUSTER__DNS__ {
                force_tcp
        }
        prometheus :9253
        }
    .:53 {
        errors
        cache 30
        reload
        loop
        bind __PILLAR__LOCAL__DNS__ __PILLAR__DNS__SERVER__
        forward . __PILLAR__UPSTREAM__SERVERS__
        prometheus :9253
        }
---
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: node-local-dns
  namespace: kube-system
  labels:
    k8s-app: node-local-dns
    kubernetes.io/cluster-service: "true"
    addonmanager.kubernetes.io/mode: Reconcile
spec:
  updateStrategy:
    rollingUpdate:
      maxUnavailable: 10%
  selector:
    matchLabels:
      k8s-app: node-local-dns
  template:
    metadata:
      labels:
        k8s-app: node-local-dns
      annotations:
        prometheus.io/port: "9253"
        prometheus.io/scrape: "true"
    spec:
      priorityClassName: system-node-critical
      serviceAccountName: node-local-dns
      hostNetwork: true
      dnsPolicy: Default  # Don't use cluster DNS.
      tolerations:
      - key: "CriticalAddonsOnly"
        operator: "Exists"
      - effect: "NoExecute"
        operator: "Exists"
      - effect: "NoSchedule"
        operator: "Exists"
      containers:
      - name: node-cache
        # # 这个镜像要换成国内的
        image: registry.k8s.io/dns/k8s-dns-node-cache:1.26.4 
        resources:
          requests:
            cpu: 25m
            memory: 5Mi
        args: [ "-localip", "__PILLAR__LOCAL__DNS__,__PILLAR__DNS__SERVER__", "-conf", "/etc/Corefile", "-upstreamsvc", "kube-dns-upstream" ]
        securityContext:
          capabilities:
            add:
            - NET_ADMIN
        ports:
        - containerPort: 53
          name: dns
          protocol: UDP
        - containerPort: 53
          name: dns-tcp
          protocol: TCP
        - containerPort: 9253
          name: metrics
          protocol: TCP
        livenessProbe:
          httpGet:
            host: __PILLAR__LOCAL__DNS__
            path: /health
            port: 8080
          initialDelaySeconds: 60
          timeoutSeconds: 5
        volumeMounts:
        - mountPath: /run/xtables.lock
          name: xtables-lock
          readOnly: false
        - name: config-volume
          mountPath: /etc/coredns
        - name: kube-dns-config
          mountPath: /etc/kube-dns
      volumes:
      - name: xtables-lock
        hostPath:
          path: /run/xtables.lock
          type: FileOrCreate
      - name: kube-dns-config
        configMap:
          name: kube-dns
          optional: true
      - name: config-volume
        configMap:
          name: node-local-dns
          items:
            - key: Corefile
              path: Corefile.base
---
# A headless service is a service with a service IP but instead of load-balancing it will return the IPs of our associated Pods.
# We use this to expose metrics to Prometheus.
apiVersion: v1
kind: Service
metadata:
  annotations:
    prometheus.io/port: "9253"
    prometheus.io/scrape: "true"
  labels:
    k8s-app: node-local-dns
  name: node-local-dns
  namespace: kube-system
spec:
  clusterIP: None
  ports:
    - name: metrics
      port: 9253
      targetPort: 9253
  selector:
    k8s-app: node-local-dns
~~~

该资源清单文件中包含几个变量值得注意，其中：

~~~
__PILLAR__DNS__SERVER__ ：表示 kube-dns 这个 Service 的 ClusterIP，可以通过命令
kubectl get svc -n kube-system | grep kube-dns | awk '{ print $3 }' 获取（我们这里
就是 10.96.0.10 ）
__PILLAR__LOCAL__DNS__ ：表示 DNSCache 本地的 IP，默认为 169.254.20.10
__PILLAR__DNS__DOMAIN__ ：表示集群域，默认就是 cluster.local
~~~

另外还有两个参数 `__PILLAR__CLUSTER__DNS__` 和 `__PILLAR__UPSTREAM__SERVERS__` ，这两个参数 会通过镜像 1.15.16 版本去进行自动配置，对应的值来源于 kube-dns 的 ConfigMap 和定制的 Upstream Server 配置。直接执行如下所示的命令即可安装

~~~bash
# 一、定义变量
#获取coredns的IP：kubectl get svc kube-dns -n kube-system -o jsonpath={.spec.clusterIP}

kubedns=10.96.0.10
# 表示集群域，默认就是 cluster.local
domain=cluster.local
# 表示 DNSCache 本地的 IP，默认为169.254.20.10
localdns=169.254.20.10

# 二、替换(二选一)

# 如果 kube-proxy 运行在 IPTABLES 模式：
sed -i "s/__PILLAR__LOCAL__DNS__/$localdns/g;s/__PILLAR__DNS__DOMAIN__/$domain/g;s/__PILLAR__DNS__SERVER__/$kubedns/g" nodelocaldns.yaml

# 如果 kube-proxy 运行在 IPVS 模式：
sed -ri "s/__PILLAR__LOCAL__DNS__/$localdns/g;s/__PILLAR__DNS__DOMAIN__/$domain/g; s/,?__PILLAR__DNS__SERVER__//g;s/__PILLAR__CLUSTER__DNS__/$kubedns/g" nodelocaldns.yaml
# 三、镜像地址要用之前在ingress章节里教大家的方法搞成自己的
sed -ri 's@registry.k8s.io/dns/k8s-dns-node-cache:1.26.4@crpi-hdcg863vh2ayst45.cn-shanghai.personal.cr.aliyuncs.com/liuxu8677/nodelocaldns:1.26.4@g' nodelocaldns.yaml
~~~

部署并查看可以通过如下命令来查看对应的 Pod 是否已经启动成功。

~~~bash
[root@k8s-master-01 test3]# kubectl get pods -n kube-system -l k8s-app=node-local-dns
NAME                   READY   STATUS    RESTARTS   AGE
node-local-dns-j56rw   1/1     Running   0          8m20s
node-local-dns-jp2k6   1/1     Running   0          8m20s
node-local-dns-t7s4t   1/1     Running   0          8m20s
~~~

需要注意的是这里使用 DaemonSet 部署 node-local-dns 使用了 hostNetwork=true ，会占用宿主机 的 8080 端口，所以需要保证该端口未被占用。

但是到这里还没有完，如果 kube-proxy 组件使用的是 ipvs 模式的话我们还需要修改 每个节点上的 kubelet 的 --cluster-dns 参数，将其指向 169.254.20.10 

~~~bash
$ vim /var/lib/kubelet/config.yaml
clusterDNS:
- 169.254.20.10 # 原值为10.96.0.10
$ systemctl restart kubelet # 然后重启kubelet
~~~

Daemonset 会在每个节点创建一个网卡来绑这个 IP，Pod 向本节点这个 IP 发 DNS 请求，缓存没有命 中的时候才会再代理到上游集群 DNS 进行查询。

~~~bash
# 宿主机上执行
$ ip a
.......
25: nodelocaldns: <BROADCAST,NOARP> mtu 1500 qdisc noop state DOWN group default 
    link/ether 52:f9:c3:fb:0d:7d brd ff:ff:ff:ff:ff:ff
    inet 169.254.20.10/32 scope global nodelocaldns
       valid_lft forever preferred_lft forever
~~~

