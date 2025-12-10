# NFS 网络存储

对速度有特殊要求的的场景（多为有状态服务）会用到本地存储，需要付出的代价是 pod 无法随意漂移。 如果对速度没有极致的要求、想实现 pod 可以随意漂移，那就不能用本地存储了，需要用到网络存储， 网络存储里最简单的就是 NFS 。



## 安装 NFS

在 k8s 集群内找个节点安装 nfs server，也可以在集群外安装。同时为了保证 nfs server 高可用，避免单点故障，可以做相关配套设置。但是这里简单起见，随便找一个节点安装 一个 nfs server 即可。

在机器 172.16.143.24 安装 nfs server

~~~bash
systemctl stop firewalld.service
systemctl disable firewalld.service

# 服务端软件安装
# 安装nfs-utils和rpcbind两个包
yum install -y nfs-utils rpcbind

# 创建共享目录
mkdir -p /data/nfs
chmod 755 /data/nfs

# 配置共享目录
cat > /etc/exports <<EOF
/data/nfs *(rw,sync,no_root_squash)
EOF

# *：表示任何人都有权限连接，当然也可以是一个网段，一个 IP，也可以是域名
# rw：读写的权限
# sync：表示文件同时写入硬盘和内存
# no_root_squash：当登录 NFS 主机使用共享目录的使用者是 root 时，
# 其权限将被转换成为匿名使用者，通常它的 UID 与 GID，都会变成 nobody 身份

# 启动nfs服务
systemctl start rpcbind
systemctl enable rpcbind
systemctl status rpcbind

systemctl start nfs
systemctl enable nfs
systemctl status nfs

# 如下显示则ok
$ rpcinfo -p|grep nfs
    100003    3   tcp   2049  nfs
    100003    4   tcp   2049  nfs
    100227    3   tcp   2049  nfs_acl
    100003    3   udp   2049  nfs
    100227    3   udp   2049  nfs_acl
~~~

在 k8s 集群的所有节点上安装 nfs 客户端软件

~~~bash
yum install -y nfs-utils
~~~

在客户端宿主机上验证下能否使用

~~~bash
➜ showmount -e 172.16.143.24
Export list for 172.16.143.24:
/data/nfs *
➜ mount -t nfs 172.16.143.24:/data/nfs /mnt # 可千万别挂载到/opt下哈，/opt/cni/bin放着网络插件呢
➜ touch /mnt/a.txt # 成功后，另外一台客户端上挂载查看验证
~~~



## 手动使用 nfs 类型的 pv

创建 nfs 类型的 pv

~~~yaml
# nfs-pv.yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: nfs-pv
spec:
  storageClassName: manual		# 目前没有意义
  capacity:
    storage: 10Mi
  accessModes:
    - ReadWriteOnce
  persistentVolumeReclaimPolicy: Retain
  nfs:
    path: /data/nfs
    server: 172.16.143.24		# 指定 nfs server ip
~~~

创建 pvc

~~~yaml
# nfs-pvc.yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: nfs-pvc
spec:
  storageClassName: manual	# # 目前没有意义
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 10Mi
~~~

应用

~~~bash
kubectl apply -f nfs-pv.yaml
kubectl apply -f nfs-pvc.yaml
~~~

查看 pvc 和 pv，发现立即绑在一块，说明可以直接使用了。

~~~bash
[root@k8s-master-01 nfs-demo]# kubectl get pv
NAME     CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS   CLAIM             STORAGECLASS   VOLUMEATTRIBUTESCLASS   REASON   AGE
nfs-pv   10Mi       RWO            Retain           Bound    default/nfs-pvc   manual         <unset>                          3m3s
[root@k8s-master-01 nfs-demo]# 
[root@k8s-master-01 nfs-demo]# 
[root@k8s-master-01 nfs-demo]# kubectl get pvc
NAME      STATUS   VOLUME   CAPACITY   ACCESS MODES   STORAGECLASS   VOLUMEATTRIBUTESCLASS   AGE
nfs-pvc   Bound    nfs-pv   10Mi       RWO            manual         <unset>                 83s
~~~

创建 pod

~~~yaml
# nfs-pod.yaml
apiVersion: v1
kind: Pod
metadata:
  name: test-volumes
spec:
  volumes:
    - name: nfs
      persistentVolumeClaim:
        claimName: nfs-pvc		# 指定使用 pvc
  containers:
    - name: web
      image: nginx:latest
      volumeMounts:
        - name: nfs
          # 使用子路径，即会在 nfs server 的 /data/nfs 下面创建 test-volumes 文件夹
          # 子路径 test-volumes 挂载到容器内的 /usr/share/nginx/html 目录
          subPath: test-volumes
          mountPath: '/usr/share/nginx/html'
~~~



## 使用 storageClass 动态创建 PV

手动搭建 StorageClass+NFS，大致有以下几个步骤:

~~~bash
1.创建一个可用的NFS Server
2.创建Service Account.这是用来管控NFS provisioner在k8s集群中运行的权限
3.创建StorageClass.负责建立PVC并调用NFS provisioner进行预定的工作,并让PV与PVC建立管理
4.创建NFS provisioner.有两个功能,一个是在NFS共享目录下创建挂载点(volume),另一个则是建了PV
并将PV与NFS的挂载点建立
~~~

下面我们使用现成的工具（**[nfs-subdir-external-provisioner](https://github.com/kubernetes-sigs/nfs-subdir-external-provisioner)**），自动搭建 StorageClass+NFS。

nfs-subdir-external-provisioner 的官网地址：https://github.com/kubernetes-sigs/nfs-subdir-external-provisioner

nfs-subdir-external-provisioner  是一个自动配置器，它使用您现有的已配置的 NFS 服务器，通过持久卷声明 (PVC) 支持 k8s 持久卷的动态配置。持久卷的配置方式如下`${namespace}-${pvcName}-${pvName}`

#### 安装 nfs-subdir-external-provisioner 

使用 helm 的方式安装 

~~~bash
helm repo add nfs-subdir-external-provisioner \
https://kubernetessigs.github.io/nfs-subdir-external-provisioner/

# 使用 --set nfs.server=172.16.143.24 指定nfs server
# 如果没有为containerd设置镜像加速，那可能需要指定国内的镜像地址：
# --set image.repository=nfs-subdir-external-provisioner
helm upgrade --install nfs-subdir-external-provisioner \
nfs-subdir-external-provisioner/nfs-subdir-external-provisioner \
--set nfs.server=172.16.143.24 \
--set nfs.path=/data/nfs \
--set storageClass.defaultClass=true \
--set image.repository=crpi-hdcg863vh2ayst45.cn-shanghai.personal.cr.aliyuncs.com/liuxu8677/nfs-subdir-external-provisioner \
--set image.tag=v4.0.2 \
-n kube-system

# 看到如下输入说明安装成功
Release "nfs-subdir-external-provisioner" does not exist. Installing it now.
NAME: nfs-subdir-external-provisioner
LAST DEPLOYED: Wed Dec 10 14:10:19 2025
NAMESPACE: kube-system
STATUS: deployed
REVISION: 1
TEST SUITE: None
~~~

安装后查看

~~~bash
helm -n kube-system list
~~~

安装后查看 sc 和 pod。可以看到 nfs-client，并且是默认 sc

~~~bash
[root@k8s-master-01 nfs-demo]# kubectl get sc
NAME                   PROVISIONER                                     RECLAIMPOLICY   VOLUMEBINDINGMODE      ALLOWVOLUMEEXPANSION   AGE
local-path             rancher.io/local-path                           Delete          WaitForFirstConsumer   false                  3h17m
nfs-client (default)   cluster.local/nfs-subdir-external-provisioner   Delete          Immediate              true                   20s
[root@k8s-master-01 nfs-demo]# kubectl -n kube-system get pod | grep nfs
nfs-subdir-external-provisioner-97ccbc5dc-npxrh   1/1     Running     0             91s
~~~

sc 只有一是默认 sc，是因为 nfs-subdir-external-provisioner 给 nfs-client 做了如下配置

~~~yaml
[root@k8s-master-01 nfs-demo]# kubectl get sc nfs-client -o yaml | grep metadata -A 5
metadata:
  annotations:
    meta.helm.sh/release-name: nfs-subdir-external-provisioner
    meta.helm.sh/release-namespace: kube-system
    storageclass.kubernetes.io/is-default-class: "true"		#### 设置为默认 sc
  creationTimestamp: "2025-12-10T06:34:35Z"
~~~



#### 使用 sc 自动创建 pv

后面只需要定义 pvc 的 yaml 文件，在文件中指定 storageClassName的值为 nfs-client 即可触发 sc 自动创建 pv

创建 pvc

~~~yaml
# nfs-client-pvc.yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: test-claim-xxx
  namespace: default  # 可根据需要指定命名空间
spec:
  storageClassName: nfs-client  # 明确指定使用 nfs-client StorageClass
  accessModes:
    - ReadWriteMany
  resources:
    requests:
      storage: 10Mi
~~~

应用后，查看 PVC 和 PV，发现已经创建好  pvc 和 pv，并且立即做好了绑定。这是因为 sc nfs-client 中设置了 `olumeBindingMode: Immediate` 立即绑定的模式。这也是合理的，因为 nfs 支持 pod 可以在任意节点飘移，不需要延时绑定。

~~~bash
[root@k8s-master-01 nfs-demo]# kubectl apply -f nfs-client-pvc.yaml 
persistentvolumeclaim/test-claim-xxx created
[root@k8s-master-01 nfs-demo]# kubectl get pvc
NAME             STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS   VOLUMEATTRIBUTESCLASS   AGE
test-claim-xxx   Bound    pvc-ff98d9c9-7289-4a89-90c7-7302da375776   10Mi       RWX            nfs-client     <unset>                 4s
[root@k8s-master-01 nfs-demo]# 
[root@k8s-master-01 nfs-demo]# 
[root@k8s-master-01 nfs-demo]# kubectl get pv
NAME                                       CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS   CLAIM                    STORAGECLASS   VOLUMEATTRIBUTESCLASS   REASON   AGE
pvc-ff98d9c9-7289-4a89-90c7-7302da375776   10Mi       RWX            Delete           Bound    default/test-claim-xxx   nfs-client     <unset>                          7s
~~~

上面创建的 pv 在 nfs server 机器上的 /data/nfs 文件夹下创建了一个子文件夹 `default-test-claim-xxx-pvc-ff98d9c9-7289-4a89-90c7-7302da375776`



创建控制器部署 pod

如下使用 deployment 管理pod，只需要指定 pvc 的名字即可，多个副本公用一个 pv。如果使用 statefulset 的需要使用 volumeClaimTemplates

~~~yaml
# deploy-nfs-client-pod.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: web-test
  name: web-test
spec:
  replicas: 3
  selector:
    matchLabels:
      app: web-test
  strategy: {}
  template:
    metadata:
      labels:
        app: web-test
    spec:
      containers:
      - image: nginx:latest
        name: nginx
        volumeMounts:
        - name: wwwroot
          mountPath: /usr/share/nginx/html
      volumes:
      - name: wwwroot
        persistentVolumeClaim:
          claimName: test-claim-xxx  # 与 PVC 名称保持一致
~~~





## 如何设置默认的 StorageClass

#### 用 kubectl patch 命令来更新

~~~bash
# 查了当前sc，有default标识就是默认的sc
[root@k8s-master-01 nfs-demo]# kubectl -n kube-system get sc nfs-client 
NAME                   PROVISIONER                                     RECLAIMPOLICY   VOLUMEBINDINGMODE   ALLOWVOLUMEEXPANSION   AGE
nfs-client (default)   cluster.local/nfs-subdir-external-provisioner   Delete          Immediate           true                   21m


# 关闭默认
kubectl patch storageclass nfs-client -p '{"metadata": {"annotations":{"storageclass.kubernetes.io/is-default-class":"false"}}}'

# 设置为默认
kubectl patch storageclass nfs-client -p '{"metadata": {"annotations":{"storageclass.kubernetes.io/is-default-class":"true"}}}'
~~~



#### 编辑YAML文件

通过注解 `storageclass.kubernetes.io/is-default-class: "true"` 将其设置为默认存储类

~~~yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: managed-nfs-storage
  annotations:
    storageclass.kubernetes.io/is-default-class: "true"
provisioner: kaxon-nfs-storage  # 必须与 Deployment 中的环境变量 PROVISIONER_NAME 匹配
parameters:
  # archiveOnDelete: "false"
  archiveOnDelete: "true" 
  # 用于控制当 PVC 被删除时，PV 的处理方式，
  # 设置为true代表启用数据备份或归档，即在 PVC 被删除时，存储提供者可能会将数据保存到归档存储中，以便以后可以恢复。
  # 需要相应的存储提供者支持才能生效。
  
  reclaimPolicy: Retain 
  # 即使数据被归档，PV 对象也会被保留，直到手动处理。
  # 可以设置为 Retain或Delete，不要设置为已废弃的Recycle
~~~

