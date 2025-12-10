# 存储

容器中的文件是临时的，当容器重启时文件会丢失。Kubernetes 提供了多种持久化存储方案。



## 核心概念

#### **Volume（卷）**

- Pod 级别的存储
- 生命周期与 Pod 相同
- 多种类型：emptyDir、hostPath、NFS、云存储等。

#### **PersistentVolume (PV)**

- 集群级别的存储资源
- 由管理员预先创建
- 支持多种后端存储

#### **PersistentVolumeClaim (PVC)**

- 用户对存储的请求
- PVC 绑定到合适的 PV
- 在 Pod 中通过 PVC 使用存储



PV 说白了就是一层存储的抽象，底层的存储可以是本地磁盘，也可以是网络磁盘比如 NFS、Ceph 之类。

PVC 其实在 Pod 和 PV 之前又增加了一层抽象，这样做的目的在于将 Pod 的存储行为与具体的存储设备解耦。如果 pv 发生变化，不需要改动 pod 中的声明，只需要修改 pvc 即可。

~~~bash
pod <----> pvc <----> pv
~~~

创建 PVC 之后，k8s 就会去查找满足我们声明要求的 PV，如果满足要求就会将 PV 和 PVC 绑定 在一起，目前 PV 和 PVC 之间是一对一绑定的关系，也就是说一个 PV 只能被一个 PVC 绑定。



## 创建 PV

PV 是跨名称空间，创建时不需要指定名称空间。

比如，手动创建一个 PV，容量为 5Gi，访问模式为 RWO

~~~yaml
# pv.yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv-hostpath		# 指定 pv 的名字
  labels:
    type: local
spec:
  capacity:
    storage: 5Gi
  accessModes:
    - ReadWriteOnce
  persistentVolumeReclaimPolicy: Retain	 # 默认回收策略
  hostPath:
    path: '/data/test1'
    type: DirectoryOrCreate

  # 注释：
  # 1. 使用hostPath类型，将节点上的/data/test1目录作为持久化存储
  # 2. DirectoryOrCreate类型：如果目录不存在会自动创建
  # 3. Retain策略：当PVC被删除时，PV和数据会被保留
~~~

创建后查看 PV，状态为 `Available` 表示为可用的 PV，等待 PVC 绑定。

~~~bash
[root@k8s-master-01 1209]# kubectl apply -f pv.yaml 
persistentvolume/pv-hostpath created
[root@k8s-master-01 1209]# kubectl get pv pv-hostpath 
NAME          CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS      CLAIM   STORAGECLASS   VOLUMEATTRIBUTESCLASS   REASON   AGE
pv-hostpath   5Gi        RWO            Retain           Available                          <unset>                          15s
~~~



## 创建 PVC

PVC 存在名称空间的限制，创建时需要声明名称空间。

比如，创建一个 PVC，请求至少 3G 容量的卷，该卷至少可以为一个节点提供读写访问即 ReadWriteOnce

~~~yaml
# pvc.yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: pvc-hostpath
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 3Gi
# 注释：
# 1. 此PVC会查找符合条件的PV（容量≥3Gi,访问模式为ReadWriteOnce）
~~~

应用后，查看 pvc，发现 pvc 已经找到符合条件的 PV（如之前定义的 `pv-hostpath`，容量5Gi），则会自动绑定。PV 和 PVC 的状态都为 `Bound`，表示已经绑定在一块。

~~~bash
[root@k8s-master-01 1209]# kubectl apply -f pvc.yaml 
persistentvolumeclaim/pvc-hostpath created
[root@k8s-master-01 1209]# kubectl get pvc
NAME           STATUS   VOLUME        CAPACITY   ACCESS MODES   STORAGECLASS   VOLUMEATTRIBUTESCLASS   AGE
pvc-hostpath   Bound    pv-hostpath   5Gi        RWO                           <unset>                 3s
[root@k8s-master-01 1209]# 
[root@k8s-master-01 1209]# kubectl get pvc
NAME           STATUS   VOLUME        CAPACITY   ACCESS MODES   STORAGECLASS   VOLUMEATTRIBUTESCLASS   AGE
pvc-hostpath   Bound    pv-hostpath   5Gi        RWO                           <unset>                 12s
~~~

虽然 PVC 需要 3Gi 容量，PV 是 5Gi 容量，但是目前集群中只有这个 PV 满足，所有会自动绑定。如果集群中有更合适更匹配的 PV（比如容量为 5Gi），那就会绑定容量是 3Gi 的 PV。 





## 创建 POD 来应用 PVC

在 pod 的配置订单中使用 pvc，创建 pod 后，pod 调度在哪个节点，就会在对应的节点上创建 pv 中定义的文件夹路径。之所以会自动创建文件夹是因为 pv 中定义了 hostPath.type=DirectoryOrCreate

~~~yaml
# pod.yaml
apiVersion: v1
kind: Pod
metadata:
  name: pv-hostpath-pod
spec:
  volumes:
    # 声明使用PVC作为存储卷
    # 通过 persistentVolumeClaim 引用名为 pvc-hostpath 的PVC
    - name: pv-hostpath1
      persistentVolumeClaim:
        claimName: pvc-hostpath
  containers:
    - name: task-pv-container
      image: nginx:latest
      # 使用存储卷
      # 将PVC挂载到容器的 /usr/share/nginx/html 目录
      volumeMounts:
        - mountPath: '/usr/share/nginx/html'
          name: pv-hostpath1
~~~

随后，发现 pod 调度到 k8s-node-01 节点上。切换 k8s-node-01 节点，发现文件夹 /data/test1 已经存在，在这个文件夹中创建 index.html 就相当于往 nginx 容器中创建 /usr/share/nginx/html/index.html 文件。因此随后可以使用 curl 命令访问到。

~~~bash
[root@k8s-master-01 1209]# kubectl get pod -o wide
NAME              READY   STATUS    RESTARTS   AGE     IP             NODE          NOMINATED NODE   READINESS GATES
pv-hostpath-pod   1/1     Running   0          8m15s   10.244.1.144   k8s-node-01   <none>           <none>
[root@k8s-master-01 1209]# 
[root@k8s-master-01 1209]# 
[root@k8s-master-01 1209]# ssh k8s-node-01
Last login: Tue Dec  9 20:36:33 2025 from 172.16.143.22

Welcome to Alibaba Cloud Elastic Compute Service !

[root@k8s-node-01 ~]# ls /data/test1/
[root@k8s-node-01 ~]# echo 11111 > /data/test1/index.html
[root@k8s-node-01 ~]# exit
logout
Connection to k8s-node-01 closed.
[root@k8s-master-01 1209]# curl 10.244.1.144
11111
~~~



## StorageClass

为了解决下面两个问题，诞生了 StorageClass 

- 1、大规模场景下，手动创建pv太费劲 
- 2、不同存储设备IO性能不同、有快有慢，种类不同，需要进行归类，方便pod使用



在 k8s 中，动态配置的 PersistentVolumes（动态PV）指的是通过 PersistentVolumeClaims（PVCs）自动创建的 PV，与手动预先创建PVs 并等待用户去绑定它们相比，动态配置允许根据 PVC 的请求自动创建和分配存储资源。这种机制是通过 StorageClasses 实现的。

~~~bash
用户创建PVC → 指定StorageClass → 动态创建PV → 绑定PVC
~~~









