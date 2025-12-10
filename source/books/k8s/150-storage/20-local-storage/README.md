# 本地存储

本地存储（Local Storage）是指使用**节点本地磁盘**作为持久化存储的解决方案。与网络存储相比，本地存储提供**更低延迟**和**更高吞吐量**，但牺牲了数据迁移性和高可用性。

## emptyDir 存储卷

emptyDir ： 是 pod 调度到节点上时创建的一个空目录，当 pod 被删除时，emptyDir 中的数据也随即被删除。

emptyDir长用于容器间共享文件（日志采集），或者用于创建临时目录。

注意：emptyDir不能够用来做数据持久化

如下示例为两个容器共享一个存储卷，基于 emptyDir  类型的存储卷。

~~~yaml
# emptydir_demo.yaml

apiVersion: apps/v1
kind: Deployment
metadata:
  name: emptydir
spec:
  selector:
    matchLabels:
      app: emptydir
  template:
    metadata:
      labels:
        app: emptydir
    spec:
      containers:
      - name: nginx
        image: nginx:latest
        volumeMounts:
        - mountPath: /usr/share/nginx/html/
          name: test-emptydir
      - name: busybox
        image: busybox:latest
        volumeMounts:
        - mountPath: /data
          name: test-emptydir
        command: ["/bin/sh","-c","while true;do echo $(date) >> /egon_data/index.html;sleep 1;done"]
      volumes:
      - name: test-emptydir		# 定义存储卷名称
        emptyDir: {}			# 定义存储卷类型
~~~

empty 本质就是在 pod 所在的节点的 `/var/lib/kubelet/pods` 下创建出目录

~~~bash
/var/lib/kubelet/pods/pod的metadata.uid号/volumes/kubernetes.io~empty-dir/testemptydir

# 查看pod的uid
kubectl get pods emptydir-5fd54f6897-fbvrw -o jsonpath='{.metadata.uid}'
dc6747b8-77b8-469e-b2cd-b708ff31e1e5
~~~



## hostPath

hostPath 类型的存储卷，用的就是目标节点上的本地目录。

问题： 

- 1、数据就写在当前节点，后面你的pod调度到其他节点了就用不到之前的数据了 
- 2、hostPath路径所在的节点挂掉，数据就不可用了
- 3、用的就是本地目录，这个目录的数据源如果不是一块完整独立的盘，即与别人共享，存在相互干扰的情况。

总结：hostPath的优缺点

- 缺点：Pod不能随便漂移，需要固定在某个节点上
- 优点：直接使用本地磁盘，如果用的是ssd，那速度嘎嘎快，比远程网络存储必然是要快很多。

~~~yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: my-pv
  labels:
    type: local
spec:
  capacity:
    storage: 1Gi
  accessModes:
    - ReadWriteMany
  hostPath:
    path: /data/hostpath
    type: DirectoryOrCreate
    # 如果目录不存在则会自动创建
~~~



## Local PV

在 hostPath 的基础上，k8s 依靠 pv 与 pvc 实现了一个新特性叫 Local PV，它实现的功能类似于 hostPath+nodeAffinity，本质也是在目录主机创建目录。

强调：不管是 hostpath 还是 Local pv，如果使用的存储介质不是独立挂载的磁盘，都存在和别人共享磁盘的问题。最好是一个 PV 一个盘。

#### 创建 local pv

~~~yaml
# local-pv.yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: local-pv
spec:
  capacity:
    storage: 5Gi
  volumeMode: Filesystem
  accessModes:
    - ReadWriteOnce
  persistentVolumeReclaimPolicy: Retain  # 通常用 Retain
  storageClassName: local-storage
  local:
    path: /data/k8s/localpv  # 节点本地路径
  nodeAffinity:      		 # 必须指定节点亲和性
    required:
      nodeSelectorTerms:
      - matchExpressions:
        - key: kubernetes.io/hostname
          operator: In
          values:
          - k8s-node-01
~~~

应用后查看，PV 的 `STORAGECLASS ` 为 local-storage

~~~bash
[root@k8s-master-01 1209]# kubectl apply -f local-pv.yaml 
persistentvolume/local-pv created
[root@k8s-master-01 1209]# kubectl get pv
NAME       CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS      CLAIM   STORAGECLASS    VOLUMEATTRIBUTESCLASS   REASON   AGE
local-pv   5Gi        RWO            Retain           Available           local-storage   <unset>                          3s
~~~

与普通pv不同的地方

- 定义了一个 local 字段，表明它是一个 Local PV
- 添加了一个节点亲和性 `nodeAffinity` 字段指定 k8s-node-01 这个节点
- pod使用Local PV时，基于Local PV里设置的选择器，会被调度到固定的节点上

~~~alert type=note
对比普通 pod 的创建过程，都是先调度pod到某个节点上，然后再持久化/创建节点上的Volume目录。但对于使用local PV的pod来说，local PV必须事先创建好、存储都需要提前准备好，因为调度器会根据该pv的nodeAffinity把pod调度到固定的节点上，你可以把它叫做“卷亲和”！！！！！！！！
~~~

#### 创建 PVC

不能像使用普通 pv 那样直接创建 pvc 立即绑定一个 local PV，使用 local pv 时，这种做法是错误的。

因为 local pv 是和节点绑定的，如果 pvc 直接绑定 local pv，那么在 pod 中使用 pvc 就会导致 pod 要和 local pv 死绑定，那如果 Local pv 所在的节点资源不满足 pod 的申请需求，此时就会出现新 pod 一直原地 pending 的情况。

正确的做法应该是：**延迟绑定 pvc**，即让 pod 的调度器综合考虑资源、PV 卷来进行调度，调度完成后，再绑定 PVC 与 PV。

####  

#### 延迟绑定

可以通过创建 `StorageClass ` 来指定这个动作，在 `StorageClass ` 中有一个 `volumeBindingMode=WaitForFirstConsumer ` 的属性，就是告诉 k8s 在发现这个 StorageClass 关联的 PVC 与 PV 可以绑定在一起，但不要现在就立刻执行绑定操作（即：设置 PVC 的 VolumeName 字段），而是要等到第一个声明使用该 PVC 的 Pod 出现在调度器之后，调度器再综合考虑所有的调度规则，当然也包括每个 PV 所在的节点位置，来统一决定这个 Pod 声明的 PVC 到底应该跟哪个 PV 进行绑定。

<font style="color:red;font-weight:bold">通过这个延迟绑定机制，原本实时发生的 PVC 和 PV 的绑定过程， 就被延迟到了 Pod 第一次调度的时候在调度器中进行，从而保证了这个绑定结果不会影响 Pod 的正常调度。</font>



具体操作：

- 先创建 StorageClass  对象
- 再分别创建 pv 和 pvc
- 最后创建 pod



**创建 StorageClass 对象**

~~~yaml
# local-storageclass.yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: local-storage
provisioner: kubernetes.io/no-provisioner
volumeBindingMode: WaitForFirstConsumer
~~~

字段解释如下：

- 这个 StorageClass 的名字，叫作 local-storage，也就是我们在 PV 、PVC中声明的 storageClassName 字段值 
- provisioner 字段，指定为 no-provisioner ，代表我们是手动创建的 PV，不需要动态来生成 PV。 
- 另外这个 StorageClass 还定义了一个 volumeBindingMode=WaitForFirstConsumer 的属性，它是 Local PV 里一个非常重要的特性，即：延迟绑定。 

创建并查看

~~~bash
[root@k8s-master-01 1209]# kubectl apply -f local-storageclass.yaml 
storageclass.storage.k8s.io/local-storage created
[root@k8s-master-01 1209]# 
[root@k8s-master-01 1209]# kubectl get storageclasses.storage.k8s.io 
NAME            PROVISIONER                    RECLAIMPOLICY   VOLUMEBINDINGMODE      ALLOWVOLUMEEXPANSION   AGE
local-storage   kubernetes.io/no-provisioner   Delete          WaitForFirstConsumer   false                  8s
~~~

**创建 local pv**

使用 storageClassName  指定上面创建的 sc

~~~yaml
# local-pv.yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: local-pv
spec:
  capacity:
    storage: 5Gi
  volumeMode: Filesystem
  accessModes:
    - ReadWriteOnce
  persistentVolumeReclaimPolicy: Retain
  local:
    path: /data/k8s/localpv
  nodeAffinity:             
    required:
      nodeSelectorTerms:
      - matchExpressions:
        - key: kubernetes.io/hostname
          operator: In
          values:
          - k8s-node-01
  storageClassName: local-storage			# 指定StorageClass的名字
~~~

**创建 pvc**

使用 storageClassName  指定上面创建的 sc

~~~yaml
# local-pvc.yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
 name: local-pvc
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 5Gi
  storageClassName: local-storage   # 指定StorageClass的名字
~~~

应用

~~~bash
kubectl apply -f local-pv.yaml
kubectl apply -f local-pvc.yaml
~~~

查看。发现 pv 没有被绑定，pvc 处于 pending 状态。

~~~bash
[root@k8s-master-01 1209]# kubectl get pv
NAME       CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS      CLAIM   STORAGECLASS    VOLUMEATTRIBUTESCLASS   REASON   AGE
local-pv   5Gi        RWO            Retain           Available           local-storage   <unset>                          3s
[root@k8s-master-01 1209]# 
[root@k8s-master-01 1209]# 
[root@k8s-master-01 1209]# kubectl get pvc
NAME        STATUS    VOLUME   CAPACITY   ACCESS MODES   STORAGECLASS    VOLUMEATTRIBUTESCLASS   AGE
local-pvc   Pending                                      local-storage   <unset>                 7s
~~~

查看并验证 pvc 处于 pending 的原因（在等待 pod）。

~~~bash
[root@k8s-master-01 1209]# kubectl describe pvc local-pvc | grep -i events -A 10
Events:
  Type    Reason                Age                  From                         Message
  ----    ------                ----                 ----                         -------
  Normal  WaitForFirstConsumer  3s (x13 over 2m57s)  persistentvolume-controller  waiting for first consumer to be created before binding
~~~

**新建 pod**

~~~yaml
# local-pod.yaml
apiVersion: v1
kind: Pod
metadata:
  name: local-pv-pod
spec:
  volumes:
    - name: local-pvc
      persistentVolumeClaim:
        claimName: local-pvc
  containers:
    - name: task-local-pv-container
      image: nginx:latest
      volumeMounts:
        - mountPath: '/usr/share/nginx/html'
          name: local-pvc
~~~

应用

~~~bash
[root@k8s-master-01 1209]# kubectl apply -f local-pod.yaml 
pod/local-pv-pod configured
[root@k8s-master-01 1209]# 
[root@k8s-master-01 1209]# 
[root@k8s-master-01 1209]# kubectl get pod -o wide
NAME           READY   STATUS              RESTARTS   AGE     IP       NODE          NOMINATED NODE   READINESS GATES
local-pv-pod   0/1     ContainerCreating   0          3m54s   <none>   k8s-node-01   <none>           <none>
~~~

但是，发现 pod 一直处于 ContainerCreating ，使用 describe 查看事件发现需要手动创建目录。

~~~bash
[root@k8s-master-01 1209]# kubectl describe pod local-pv-pod | grep -i events -A 10
Events:
  Type     Reason       Age                    From               Message
  ----     ------       ----                   ----               -------
  Normal   Scheduled    6m41s                  default-scheduler  Successfully assigned default/local-pv-pod to k8s-node-01
  Warning  FailedMount  4m33s (x9 over 6m41s)  kubelet            MountVolume.NewMounter initialization failed for volume "local-pv" : path "/data/k8s/localpv" does not exist
~~~

切到目标主机上，手动创建目录，创建完毕后，重新部署 pod 即可调度成功。



## local-path-provisioner

**local-path-provisioner** 是 Rancher 开发的一个轻量级动态本地存储供应器。它允许在每个 Kubernetes 节点上使用本地存储，并自动管理 PersistentVolume 的生命周期。

>Rancher 是一家专注于 Kubernetes 管理平台和云原生技术的公司，它提供了名为 **Rancher** 的容器管理平台，使用户能够在任何基础设施上部署、运行和管理 Kubernetes 集群。Rancher 平台本身并不等同于 Kubernetes，而是对 Kubernetes 进行了封装和增强，提供了图形化界面、多集群管理、权限控制、应用商店、监控日志等企业级功能，降低了 Kubernetes 的使用门槛。

~~~bash
用户创建 PVC → local-path-provisioner → 在调度节点上创建目录 → 创建 PV → 绑定 PVC
~~~

它对外提供一个 storageclass 默认名为 `local-path`，可以非常方便的使用。

使用前，可以把 yaml 中默认镜像替换成国内的镜像。

~~~bash
wget https://raw.githubusercontent.com/rancher/local-path-provisioner/master/deploy/local-path-storage.yaml

# 替换镜像
sed -i '/image: rancher/s/rancher\/local-path-provisioner:v0.0.32/crpi-hdcg863vh2ayst45.cn-shanghai.personal.cr.aliyuncs.com\/liuxu8677\/local-path-provisioner:v0.0.32/' local-path-storage.yaml
~~~

部署

~~~bash
kubectl apply -f local-path-storage.yaml
~~~

部署后查看创建的 sc 和 pod

~~~bash
[root@k8s-master-01 1210]# kubectl get sc
NAME            PROVISIONER                    RECLAIMPOLICY   VOLUMEBINDINGMODE      ALLOWVOLUMEEXPANSION   AGE
local-path      rancher.io/local-path          Delete          WaitForFirstConsumer   false                  20s
[root@k8s-master-01 1210]# kubectl -n local-path-storage get pods
NAME                                      READY   STATUS    RESTARTS   AGE
local-path-provisioner-77fc5b4569-dlwsp   1/1     Running   0          110s
~~~



#### 使用方式

安装部署 local-path-provisioner 后，会在集群中创建一个 storageclass 名字为 local-path，并且配合一个 provisioner pod 负责创建 hostPath 类型的 PV。

提交创建 pvc 的请求时，通过设置 `storageClassName: local-path ` 通知 local-path-provisioner pod 自动创建 PV。

后面部署 pod 时，如果使用的控制器是 deployment，那么多个副本是无状态的，只有一个 PVC，只有一个 PV，多副本共享一份数据。

后面部署 pod 时，如果使用的控制器是 statefulset，不需要手动创建 PVC，通过在 yaml 中使用 volumeClaimTemplates  配置 PVC 模板，多个副本会创建多个 PVC，一个 PVC 会自动创建并关联一个 PV，最终的效果是一个副本使用一个 PV。



#### 使用 deployment 部署 pod

创建 pvc，配置 `storageClassName: local-path `，使用 local-path-provisioner 自动创建 pv

**创建 pvc**

~~~yaml
# local-pvc.yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
 name: local-pvc   # pvc 名字
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 5Gi
  # 使用local-path-provisioner 提供的 StorageClass 的名字
  storageClassName: local-path 
~~~

部署完上述 pvc 后，查看处于 pending 状态，这是因为 local-path-provisioner 设置了延时绑定。

**部署 deploy**

~~~yaml
# deploy-local-pvc-demo.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-test
  labels:
    app: web-test
spec:
  replicas: 1
  selector:
    matchLabels:
      app: web-test
  template:
    metadata:
      labels:
        app: web-test
    spec:
      containers:
      - name: nginx
        image: nginx:latest
        resources:
          requests:
            cpu: 10m
            memory: 10Mi
        volumeMounts:
        - name: wwwroot
          mountPath: /usr/share/nginx/html
      volumes:
      - name: wwwroot
        persistentVolumeClaim:	# 设置使用 pvc
          claimName: local-pvc
~~~

使用上述 yaml 部署 pod，待 pod 就绪后，pvc 也完成了绑定 pv 的操作，pv 也是自动创建的。

~~~bash
[root@k8s-master-01 1210]# kubectl get pod -o wide
NAME                        READY   STATUS    RESTARTS   AGE   IP             NODE          NOMINATED NODE   READINESS GATES
web-test-555cf6dfd9-gxksh   1/1     Running   0          63s   10.244.2.182   k8s-node-02   <none>           <none>
[root@k8s-master-01 1210]# kubectl get pvc
NAME        STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS   VOLUMEATTRIBUTESCLASS   AGE
local-pvc   Bound    pvc-85896fe2-0793-446d-9942-137649eb6954   5Gi        RWO            local-path     <unset>                 63s
[root@k8s-master-01 1210]# 
[root@k8s-master-01 1210]# 
[root@k8s-master-01 1210]# kubectl get pv
NAME                                       CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS   CLAIM               STORAGECLASS   VOLUMEATTRIBUTESCLASS   REASON   AGE
pvc-85896fe2-0793-446d-9942-137649eb6954   5Gi        RWO            Delete           Bound    default/local-pvc   local-path     <unset>                          59s
~~~

看到 pod 调度到 k8s-node-02，那么 pv 肯定也创建在 k8s-node-02 上面，具体的文件夹路径由 local-path-provisioner 指定的。

~~~bash
[root@k8s-master-01 1210]# cat local-path-storage.yaml | grep ConfigMap -A 14
kind: ConfigMap
apiVersion: v1
metadata:
  name: local-path-config
  namespace: local-path-storage
data:
  config.json: |-
    {
            "nodePathMap":[
            {
                    "node":"DEFAULT_PATH_FOR_NON_LISTED_NODES",
                    "paths":["/opt/local-path-provisioner"]
            }
            ]
    }
~~~

也可以查看 pv 时看到文件夹创建的具体路径

~~~bash
[root@k8s-master-01 1210]# kubectl describe pv pvc-85896fe2-0793-446d-9942-137649eb6954 
Name:              pvc-85896fe2-0793-446d-9942-137649eb6954
Labels:            <none>
Annotations:       local.path.provisioner/selected-node: k8s-node-02
                   pv.kubernetes.io/provisioned-by: rancher.io/local-path
Finalizers:        [kubernetes.io/pv-protection]
StorageClass:      local-path
Status:            Bound
Claim:             default/local-pvc
Reclaim Policy:    Delete
Access Modes:      RWO
VolumeMode:        Filesystem
Capacity:          5Gi
Node Affinity:     
  Required Terms:  
    Term 0:        kubernetes.io/hostname in [k8s-node-02]
Message:           
Source:
    Type:          HostPath (bare host directory volume)
    #### 如下 Path 就是local.path.provisioner在 k8s-node-02 节点上自动创建 pv 的路径
    Path:          /opt/local-path-provisioner/pvc-85896fe2-0793-446d-9942-137649eb6954_default_local-pvc
    HostPathType:  DirectoryOrCreate
Events:            <none>
~~~

因为 pv 创建在 k8s-node-02 上面，后面使用该 deployment 扩缩副本，无论多少个副本都会部署到 k8s-node-02 上面，并且全部共享一个 PV。



#### 使用 statefulset 部署 pod

无需手动创建 PVC，使用 volumeClaimTemplates 配置创建 PVC 的模板，在模板中使用 `storageClassName: local-path` 指定 storageclass 的名字，就会自动关联 local-path-provisioner 创建 pvc，创建 pv。并且每个副本创建一个 PVC，绑定一个 PV。

~~~yaml
# statefulset-local-pvc-demo.yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: redis
spec:
  serviceName: redis
  replicas: 3
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
        image: busybox:latest
        command: ["tail","-f","/dev/null"]
        volumeMounts:
        - name: redis-data
          mountPath: /data/redis
  
  # 设置使用 pvc
  # 手动指定使用 storageClassName: local-path
  volumeClaimTemplates:
  - metadata:
      name: redis-data
    spec:
      storageClassName: local-path
      # "local-path": NodePath only supports ReadWriteOnce and ReadWriteOncePod (1.22+) access modes
      accessModes: ["ReadWriteOnce"]
      resources:
        requests:
          storage: 100Mi
~~~

