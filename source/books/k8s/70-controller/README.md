# 控制器

控制器就是自动化管理与维护资源状态的工具，控制器分为两类：内置控制器（如：replicaset、deployment、daemonset、statefulset、job、cronjob）和自定义控制器。

创建出资源本质仅仅只是往 etcd 数据库中写入了一条数据，针对这些数据必须要有代码负责 watch 该数据，创建出该资源，并且要负责后续的维护，这就是控制器的功能。

~~~alert type=note
控制器的代码会 watch 某种特定的资源类型，相当于在监听一张数据库表 <br>
有新增的记录，控制器会调用 api 来根据资源的 spec 的固定来创建出资源,如果发现 status 与 spec 状态不符，则会进行调谐。
~~~



## replicaset

ReplicaSet 擅长管理副本，并且只擅长管理副本。只感知副本数量的变化。比如修改副本数量，pod 可以动态扩缩，但除此之外，比如更新容器的镜像，replicaset 是不关心的。

虽然 ReplicaSet 可以独立使用，但一般还是建议使用 Deployment 来自动管理 ReplicaSet，这样你才可以实现更高级的功能例如滚动更新。

~~~yaml
# nginx-replicas.yaml

apiVersion: apps/v1
kind: ReplicaSet		# 副本控制器的类型
metadata:
  name: nginx-replicaset	# 副本控制器资源的名字
  labels:
    app: nginx
spec:
  replicas: 3		# 控制副本数量
  selector:			# 选择副本管理 pod 的选择依据
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:latest
~~~

提交创建资源的请求

~~~bash
kubectl apply -f nginx-replicas.yaml
~~~

查看 replicaset 资源，可以看到新建副本资源的名字是 `nginx-replicaset`，它目标管理三个 pod，当前管理三个 pod，并且三个都是 ready 的状态。

~~~bash
[root@k8s-master-01 ~]# kubectl get replicaset --show-labels 
NAME               DESIRED   CURRENT   READY   AGE     LABELS
nginx-replicaset   3         3         3       4m34s   app=nginx
~~~

查看 pod，可以看到三个 pod 的名称前缀是 `nginx-replicaset`，即管理这些 pod 的副本资源的名字。

~~~bash
[root@k8s-master-01 ~]# kubectl get pods
NAME                     READY   STATUS    RESTARTS   AGE
nginx-replicaset-495kl   1/1     Running   0          5m8s
nginx-replicaset-cm6bg   1/1     Running   0          5m8s
nginx-replicaset-tb4k6   1/1     Running   0          5m8s
~~~

也可以使用 describe 查看到 pod 的管理者。

~~~bash
[root@k8s-master-01 ~]# kubectl describe pod nginx-replicaset-495kl | grep Controlled
Controlled By:  ReplicaSet/nginx-replicaset
~~~

进一步查看 `ownerReferences`

~~~bash
[root@k8s-master-01 ~]# kubectl get pod nginx-replicaset-495kl -o yaml | grep ownerReferences -A 8
  ownerReferences:
  - apiVersion: apps/v1
    blockOwnerDeletion: true
    controller: true
    kind: ReplicaSet
    name: nginx-replicaset
    uid: 0d17b8c2-5b80-477e-92fd-41b6215e134c
  resourceVersion: "264435"
  uid: 6d1bf386-3928-4706-9518-b022f4ff2dda
~~~

~~~alert type=note
设置镜像的简便方法:<br>
语法：kubectl set image deployment/<deployment名称> <容器名>=<新镜像>:<标签> <br>
例子：把一个名为 my-app 的 Deployment，它内部有一个名为 nginx-container 的容器，将其更新到 nginx:1.19 <br>
`kubectl set image deployment/my-app nginx-container=nginx:1.19`
~~~



## deployment

Deployment 并不直接管理 Pod，而是通过管理 **ReplicaSet** 来间接管理 Pod。

**关系链：**
`Deployment` -> `ReplicaSet` -> `Pod`

- **ReplicaSet**：它的唯一职责就是确保指定数量的、完全相同的 Pod 副本一直在运行。它是实现“弹性伸缩”和“自我修复”的基础。
- **Deployment**：在 ReplicaSet 的基础上，增加了对 Pod 定义（模板）更新的控制，从而实现了**滚动更新**和**回滚**等高级功能。

**工作流程：**
当你创建一个 Deployment 时，k8s 会做以下事情：

1. Deployment Controller 被你的 YAML 文件触发。
2. 它创建一个 ReplicaSet。
3. ReplicaSet 根据 `replicas` 数量和 `pod template` 创建对应数量的 Pod。
4. 当你更新 Deployment 的 Pod 模板（例如，镜像版本）时，Deployment 会**创建一个新的 ReplicaSet**，然后逐步在新的 ReplicaSet 中创建新 Pod，同时在旧的 ReplicaSet 中删除旧 Pod，直到所有 Pod 都迁移到新版本。这个过程就是“滚动更新”。



#### 使用 deployment

定义 deployment 的配置订单

~~~yaml
# nginx.yaml

apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment		# Deployment具体资源的名称
  labels:
    app: nginx
spec:
  replicas: 2		# 期望的 Pod 副本数，这是核心之一
  selector:			# 标签选择器，用于识别由本 Deployment 管理的 Pod
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx   # 这个标签必须与上面的 selector.matchLabels 一致
    spec:
      containers:
      - name: nginx
        image: nginx:latest
~~~

使用 `kubectl apply -f nginx.yaml` 创建 deployment 资源，然后查看资源信息。

~~~bash
[root@k8s-master-01 ~]# kubectl get deployments.apps nginx-deployment 
NAME               READY   UP-TO-DATE   AVAILABLE   AGE
nginx-deployment   2/2     2            2           8m20s
~~~

查看 pod

~~~bash
[root@k8s-master-01 ~]# kubectl get pods
NAME                               READY   STATUS    RESTARTS   AGE
nginx-deployment-879fcc85f-gjngg   1/1     Running   0          49s
nginx-deployment-879fcc85f-gkfvd   1/1     Running   0          51s
~~~

查看 deployment 自动创建的 replicaset

~~~bash
[root@k8s-master-01 ~]# kubectl get replicasets.apps nginx-deployment-879fcc85f
NAME                         DESIRED   CURRENT   READY   AGE
nginx-deployment-879fcc85f   2         2         2       2m5s
~~~

按照 pod、replicates、deployment 的顺序查看 `ownerReferences`

~~~bash
# 查看 pod 的 controller
# 发现管理 pod 的是 deploment	自动创建的 replicaset
[root@k8s-master-01 ~]# kubectl get pod nginx-deployment-879fcc85f-gjngg -o yaml | grep ownerReferences -A 8
  ownerReferences:
  - apiVersion: apps/v1
    blockOwnerDeletion: true
    controller: true
    kind: ReplicaSet
    name: nginx-deployment-879fcc85f
    uid: 26739a40-ffd7-4194-9633-d251048bb5b6
  resourceVersion: "273688"
  uid: 2c3eccbc-6c9a-41fd-9c45-86ac061c9e9f

# 查看 replicaset 的 controller
[root@k8s-master-01 ~]# kubectl get replicaset nginx-deployment-879fcc85f  -o yaml | grep ownerReferences -A 8
  ownerReferences:
  - apiVersion: apps/v1
    blockOwnerDeletion: true
    controller: true
    kind: Deployment
    name: nginx-deployment
    uid: afd04568-7a6b-4e7a-86f1-c66808d5d75d
  resourceVersion: "273689"
  uid: 26739a40-ffd7-4194-9633-d251048bb5b6
~~~

#### 更新和回滚

~~~bash
# 方法一：通过修改 YAML 文件并应用
kubectl apply -f nginx-deployment.yaml

# 方法二：使用 set image 命令直接更新镜像（触发滚动更新）
kubectl set image deployment/nginx-deployment nginx=nginx:1.16.1

# 回滚到上一个版本
kubectl rollout undo deployment/nginx-deployment

# 查看滚动更新的状态
kubectl rollout status deployment/nginx-deployment

# 查看更新历史（用于回滚）
kubectl rollout history deployment/nginx-deployment

# 回滚到指定版本
kubectl rollout undo deployment/nginx-deployment --to-revision=2
~~~



#### 更新策略

Deployment 有两种更新策略：滚动更新（RollingUpdate）和重建（Recreate），默认且常用的是滚动更新。

| 特性         | `RollingUpdate` （滚动更新）                               | `Recreate` （重建）                                          |
| :----------- | :--------------------------------------------------------- | :----------------------------------------------------------- |
| **原理**     | 逐步替换 Pod，新旧版本同时存在一段时间。                   | **先全部终止**所有旧 Pod，然后再创建所有新 Pod。             |
| **服务中断** | **几乎无中断**（如果配置合理）。                           | **必然有中断**，服务在旧Pod终止后、新Pod启动前不可用。       |
| **适用场景** | **生产环境标配**，要求高可用性的应用（如 Web 服务、API）。 | **开发/测试环境**，或无法同时运行多版本的应用（如数据库 schema 变更）。 |
| **配置**     | 需要精细控制 `maxSurge` 和 `maxUnavailable`。              | 无需额外配置。                                               |



#### 滚动更新

滚动更新策略参数有两个：`maxSurge` 和 `maxUnavailable`

- `maxSurge` 在滚动更新的过程中，在 `spec.replicas` 的基础上可以额外新建最大 pod 数，控制的是创建新 pod 的数量，默认1。
- `maxUnavailable` 在滚动更新的过程中，老pod最大同时干掉几个，控制的是干掉老的  pod 的数量，默认1。

两个策略参数有两种配置方式：正整数、百分比

- 正整数表示 pod 的数量
- 百分比表示被控制 pod 数量占全部 pod 的百分比。

~~~alert type=note
实际配置滚动更新策略时，两个参数不能同时设置为 0。如果把 maxSurge 设置的过大，在滚动发布过程中会创建很多 pod，然后再删除的情况（浪费）。
~~~

<font color='red'>滚动更新时会创建一个新的 replicaset，会逐步设置新 replicates 的期望副本数量，同时减少旧 replicaset 期望的副本数量，最后实现新 pod 全部挂在新 replicaset，旧 replicaset 没有 pod 的结果。这个过程中，新 pod 逐步放在新 replicaset 上，旧 pod 逐步从旧 replicaset 上移除。完成更新后，不会把旧 replicaset 删除，方便后期有回滚需要（回滚操作就是把新 pod 再移到旧 replicaset 上面）。</font>

~~~yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
  labels:
    app: nginx
spec:
  replicas: 4
  strategy:
    type: RollingUpdate # 策略类型，另一个选项是 "Recreate"
    rollingUpdate:
      maxSurge: 25%     # 关键参数1：最大激增 Pod 数
      maxUnavailable: 25% # 关键参数2：最大不可用 Pod 数
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:latest
~~~

#### 重建更新

一种简单粗暴的更新方式，生产环境不用。

- **原理**：**先全部删除**所有旧版本的 Pod，然后**再创建**所有新版本的 Pod。
- **工作机制**：Deployment 会先把旧 ReplicaSet 的副本数缩容到 0（即杀死所有旧 Pod），然后再把新 ReplicaSet 的副本数扩展到期望值。
- **适用场景**：
  - 开发或测试环境。
  - 应用**无法同时运行两个版本**的情况，例如数据库的 Schema 变更。
  - 不关心服务短暂中断的场景。

~~~yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
  labels:
    app: nginx
spec:
  replicas: 4
  strategy:
    type: Recreate
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:latest
~~~



## daemonset

DaemonSet 是 Kubernetes 中的一个核心工作负载对象，它的核心作用是：**确保集群中所有（或部分）符合条件的 Node 上都运行一个 Pod 的副本。**

当有新的 Node 加入集群时，DaemonSet 会自动地在该 Node 上创建一个 Pod 副本。当有 Node 从集群中移除时，它上面由 DaemonSet 创建的 Pod 也会被垃圾回收。删除 DaemonSet 会清理掉所有它创建的 Pod。

#### Daemonset 的典型使用场景

DaemonSet 非常适合运行那些需要提供**节点级别基础设施服务**的 Pod，这些服务通常需要在每个节点上运行，并且对节点有亲和性。例如：

- **集群存储守护进程**：如 `glusterd`、`ceph`，需要在每个节点上运行以提供持久化存储。
- **日志收集守护进程**：如 `fluentd`、`filebeat`，需要在每个节点上收集该节点上所有 Pod 的日志。
- **监控采集守护进程**：如 `Prometheus Node Exporter`、`Datadog agent`，需要在每个节点上采集该节点的硬件和操作系统指标。
- **网络插件**：如 Calico、Flannel 的 CNI 组件，通常以 DaemonSet 形式运行，为每个节点提供网络策略和 Pod 网络。

#### Daemonset 的工作原理

DaemonSet 控制器是 Kubernetes 控制器管理器的一部分，它通过以下流程进行控制循环：

1. **监听**：DaemonSet 控制器通过监听 API Server，关注集群中 Node 和 Pod 的变化。
2. **匹配**：
   - 当有**新的 Node** 加入集群时，控制器会检查该 Node 是否匹配 DaemonSet 的 `nodeSelector` 或 `affinity` 规则。如果匹配，它就会立即在这个新 Node 上创建一个 Pod 副本。
   - 当有 **Node 被删除**时，控制器会清理掉运行在该 Node 上的 Pod。
   - 当 **DaemonSet 本身被更新**（例如，更新了容器镜像），控制器会按照特定的更新策略（如 `RollingUpdate`）逐个更新所有 Node 上的 Pod。
3. **调度的特殊性**：与 Deployment 或 StatefulSet 的 Pod 由 kube-scheduler 调度不同，**DaemonSet 的 Pod 创建是 DaemonSet 控制器直接控制的，不经过默认调度器**。这确保了 Pod 能被“强塞”到指定的节点上。不过，你仍然可以为 DaemonSet 的 Pod 配置 `tolerations` 和 `nodeName` 来绕过调度或容忍污点。



#### 使用 daemonset

如下是 daemonset 管理 pod 的基本设置，默认在非 master 节点外的其余节点创建一个 pod

~~~yaml
# nginx-ds.yaml

apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: nginx-ds
  namespace: default
  labels:
    k8s-app: nginx-ds
spec:
  selector:
    matchLabels:
      name: nginx-ds
  template:
    metadata:
      labels:
        name: nginx-ds
    spec:
      terminationGracePeriodSeconds: 30
      containers:
      - name: nginx
        image: nginx:latest
        resources:
          limits:
            memory: 200Mi
          requests:
            memory: 200Mi
~~~

查看 daemonset 资源

~~~bash
[root@k8s-master-01 ~]# kubectl get ds
NAME       DESIRED   CURRENT   READY   UP-TO-DATE   AVAILABLE   NODE SELECTOR   AGE
nginx-ds   2         2         2       2            2           <none>          11s
~~~

Daemon 资源直接管理创建的 pod

~~~bash
[root@k8s-master-01 ~]# kubectl describe pod nginx-ds-7vtt4 | grep Controlled
Controlled By:  DaemonSet/nginx-ds
~~~

为了让 DaemonSet 能在包括 Master 节点在内的所有节点上运行，需要让 Pod 能够容忍 Master 节点的污点。如下的例子就是容忍了常见的控制平面污点。

~~~yaml
# nginx-ds.yaml

apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: nginx-ds
  namespace: default
  labels:
    k8s-app: nginx-ds
spec:
  selector:
    matchLabels:
      name: nginx-ds
  template:
    metadata:
      labels:
        name: nginx-ds
    spec:
      # 容忍 Master 节点的污点，因为默认情况下 Master 节点不允许运行普通 Pod
      tolerations:
      - key: node-role.kubernetes.io/control-plane
        operator: Exists
        effect: NoSchedule
      - key: node-role.kubernetes.io/master
        operator: Exists
        effect: NoSchedule
      terminationGracePeriodSeconds: 30
      containers:
      - name: nginx
        image: nginx:latest
        resources:
          limits:
            memory: 200Mi
          requests:
            memory: 200Mi
~~~

~~~alert type=important
总结<br>
DaemonSet 控制器默认会在每个节点都启动一个 pod 副本，也可以使用 `NodeSelector` 或者 `NodeAffinity` 来控制哪些节点上启动。
~~~



#### 控制 pod 运行在指定标签的节点

给节点打标签

~~~bash
kubectl label node k8s-node-01 disktype=ssd
~~~

设置 pod 运行在指定标签的节点

~~~yaml
# 在 DaemonSet 的 spec.template.spec 中添加
spec:
  nodeSelector:
    disktype: ssd
~~~

应用 yaml 文件后查看 pod 的调度情况。只在 `k8s-node-01` 上运行 pod

~~~bash
[root@k8s-master-01 ~]# kubectl get pods -o wide
NAME             READY   STATUS    RESTARTS   AGE   IP            NODE          NOMINATED NODE   READINESS GATES
nginx-ds-wccxk   1/1     Running   0          73s   10.244.1.48   k8s-node-01   <none>           <none>
~~~



#### DaemonSet 的更新策略

DaemonSet 支持两种更新策略，通过 `daemonset.spec.updateStrategy.type` 字段指定：

- `RollingUpdate`（默认）： 以滚动更新的方式更新 Pod。你可以通过 `daemonset.spec.updateStrategy.rollingUpdate.maxUnavailable` 来控制一次最多不可用的 Pod 数量（默认是 1）。
- `OnDelete`： 只有当用户手动删除旧的 Pod 时，才会创建新的 Pod。这提供了更手动的控制方式，类似于 StatefulSet 的旧版本行为。



#### DaemonSet  滚动更新

~~~yaml
# nginx-ds.yaml

apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: nginx-ds
  namespace: default
  labels:
    k8s-app: nginx-ds
spec:
  selector:
    matchLabels:
      name: nginx-ds
  updateStrategy:
    type: RollingUpdate	# 默认是RollingUpdate即滚动更新
    rollingUpdate:	# 只有当type=RollingUpdate时才设置rollingUpdate
      maxUnavailable: 1	 # 控制一次最多不可用的 Pod 数量
  template:
    metadata:
      labels:
        name: nginx-ds
    spec:
      containers:
      - name: nginx
        image: nginx:latest
~~~



#### Daemonset 和 Deployment 的区别

| 特性         | DaemonSet                          | Deployment                                 |
| :----------- | :--------------------------------- | :----------------------------------------- |
| **目标**     | 每个节点运行一个 Pod 副本          | 运行指定数量的**无状态** Pod 副本          |
| **调度**     | 由控制器直接管理，不经过默认调度器 | 由 kube-scheduler 调度到**任何**合适的节点 |
| **副本数**   | 由集群中匹配的节点数量决定         | 由 `replicas` 字段明确指定                 |
| **适用场景** | 节点级基础设施服务                 | 微服务、Web 应用等                         |



## statefulset

StatefulSet：译为“有状态的集合”，k8s 内置的控制器，专门用于管理有状态的资源。有状态服务往往都会有自己的独特的状态维护需求，statefulse 无法满足所有场景，针对一些特定的场景还是需要开发自己的控制器来管理。

~~~alert type=note
状态就是数据 <br>
有状态指的是：应用的运行要依赖之前的数据---》限制调度，pod 只能调度到之前状态存在的节点上才可以 <br>
无状态指的是：应用的运行不需要依赖之前的数据--》对调度没有限制，pod 可以随意漂移到最合适的节点
~~~

应用如果对状态维护没有特殊要求，只有一些常规需求如下，就可以用内置 statefulset 控制器

- 稳定的次序：对于 N 个副本的 StatefulSet，每个 Pod 名字后缀从 0-N，按次序启停。
- 稳定的网络标识（网络标识即dns域名）：每个 pod 都有一个固定的域名（FQDN），pod 的 ip 地址随意变化，但这个域名不变。
- 稳定的存储：通过 VolumeClaimTemplate 为每个 Pod 创建一个 PV。删除、减少副本，不会删除相关的卷。



#### statefulset 的两个重要组成部分

Headless Service。可以使得每个 pod 有一个固定 FQDN 名字，上游的访问可以直接访问该域名来访问 pod，不需要 svc 做流量转发、没有负载均衡的效果。

volumeClaimTemplates。创建 pvc 的模版，每个 pod 副本都会创建出一个自己独有的 pvc，不与其他 pod 共享。



#### 无头服务

pod名字固定+创建一个无头服务，就可以让每个pod拥有一个固定的fqdn域名（coredns负责解析）

~~~
fqdn 域名格式：
$(podname).(headless server name).namespace.svc.cluster.local
~~~

编写 statefulset 配置清单，配置无头服务的名字 `serviceName`，配置 PVC 模板 `volumeClaimTemplates`

~~~yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: web
spec:
  serviceName: "nginx"
  replicas: 3
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:latest
        volumeMounts:
        - name: www
          mountPath: /usr/share/nginx/html
  volumeClaimTemplates:	# 为每个 Pod 动态创建 PVC
  - metadata:
      name: www
    spec:
      accessModes:
      - ReadWriteMany
      resources:
        requests:
          storage: 1Gi
~~~

创建无头服务的配置清单

~~~yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx		# 无头服务的名字在 StatefulSet 中被使用 serviceName 字段引用
  labels:
    app: nginx
spec:
  ports:
  - port: 80
    name: web
    targetPort: 80
  clusterIP: None		# 无头服务必须设置clusterIP为None
  selector:
    app: nginx
~~~

手动创建 pv

~~~yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv-001
  labels:
    type: local
spec:
  capacity:
    storage: 1Gi
  accessModes:
    - ReadWriteMany
  hostPath:
    path: "/data/pv-001"
---
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv-002
  labels:
    type: local
spec:
  capacity:
    storage: 1Gi
  accessModes:
    - ReadWriteMany
  hostPath:
    path: "/data/pv-002"
~~~



#### 普通 svc 和 无头 svc 

~~~bash
基于普通svc的访问流程
		普通svc的域名----coredns解析-----》svc的cluster ip----ipvs转发-》pod的endpoint
		
基于无头svc的访问流程
		无头svc的域名----coredns解析-----》所关联pod的ip地址作为dns解析记录来返回
~~~

创建临时 pod，在 pod 内测试有状态 pod 的域名

~~~bash
kubectl run test-nginx --image=nginx:latest --rm -it --restart=Never bash
~~~



#### StatefulSet 内置的顺序控制

使用 `podManagementPolicy` 控制 pod 启动顺序， 默认顺序行为。默认值是 `OrderedReady`。

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: web
spec:
  serviceName: "web"
  replicas: 3
  podManagementPolicy: OrderedReady  # 默认值
```

**默认行为**：

- Pod 按顺序创建（web-0 → web-1 → web-2）
- 前一个 Pod 完全 Ready 后，才创建下一个
- 删除时按逆序进行（web-2 → web-1 → web-0）

想要并发起 pod，设置 `podManagementPolicy: Parallel` ，并行启动，不等待前一个 Ready。





## job

Kubernetes 的 `Job` 控制器用于创建一个或多个 Pod，并确保指定数量的 Pod 成功终止。当一个 Pod 成功完成（即容器以退出码 0 结束），Job 就认为是成功的。pod 正常执行完毕后的状态为 completed。

它与 Deployment/ReplicaSet 的主要区别：

- Deployment/ReplicaSet：用于管理持续运行的、永远不会结束的应用（如 Web 服务器）。
- Job：用于管理运行到完成（Run to Completion）的离散型任务（如批处理作业、数据备份、计算任务等）。

~~~yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: hello-job
spec:
  # 指定需要成功运行的 Pod 数量，默认为 1
  completions: 10
  # 指定并行运行的 Pod 数量，默认为 1
  parallelism: 3
  # Job 的超时时间，超过此时间 Job 会被标记为失败。
  # activeDeadlineSeconds: 30
  # pod 重启次数，# 最多重试 3 次（即总共运行 4 次： 1次初始 + 3次重试）
  # backoffLimit: 3
  
  # Job 在完成后的指定秒数会被自动删除。
  # ttlSecondsAfterFinished: 60

  # 模板，用于创建 Pod
  template:
    spec:
    	restartPolicy: OnFailure
      containers:
      - name: busybox-container
        image: nginx:latest
        imagePullPolicy: IfNotPresent
        # Job 的 Pod 必须设置 restartPolicy 为 Never 或 OnFailure
        command:
        - "bin/sh"
        - "-c"
        - "i=0;while true; do echo $i; sleep 1;let i++;done"
~~~

#### 查看 job 和 pod

~~~bash
kubectl get job
kubectl get pod 
kubectl get pod --selector=job-name=hello-job
~~~



####  重启策略

Job 的 Pod 必须设置 restartPolicy 为 Never 或 OnFailure。

- OnFailure：表示当 pod 运行失败后，会重启该 pod 本身（不会创建新pod），最多重启次数取决于 `backoffLimit`。
- Never。当 pod 运行失败后，不会重启 pod 本身了，但是 job 控制器会创建新的 pod，最多新建 pod 的数量取决于 `backoffLimit` 。

Job 控制器通过 `spec.backoffLimit` 字段来控制重试次数。它默认为 6。失败后的重试间隔会指数级增加（10s, 20s, 40s ...），最大间隔为 6 分钟。



#### 任务超时设置

如果 job 一直处于 running 状态，无法 complete，一旦超过超时时间，就会被判定为 failed。设置超时时间`spec.activeDeadlineSeconds`



#### 并行 job

~~~yaml
kind: Job
metadata:
  name: parallel-job
spec:
  completions: 10   # 总共需要完成 10 个任务
  parallelism: 3    # 最多同时运行 3 个 Pod
~~~

这个 Job 会确保有 10 个 Pod 成功退出，并且任何时候最多只有 3 个 Pod 在同时运行。



#### 自动清理完成的 Job

完成的 Job（无论是成功还是失败）默认会一直保留在系统中，这可能会占用 API Server 的资源。为了自动清理，有两个主要方法：

##### 方法 1： 使用 `ttlSecondsAfterFinished` (Kubernetes v1.21+ 稳定)

这是推荐的方式。Job 在完成后的指定秒数会被自动删除

~~~yaml
# ttl-job.yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: ttl-job
spec:
  ttlSecondsAfterFinished: 100 # 完成后 100 秒自动删除
  template:
    spec:
      containers:
      - name: sleeper
        image: busybox
        command: ["sh", "-c", "echo 'Done!'; sleep 5"]
      restartPolicy: OnFailure
~~~

##### 方法 2： 使用 CronJob 的并发策略

如果你的 Job 是由 CronJob 创建的，可以在 CronJob 的 Spec 中设置 `.spec.concurrencyPolicy` 为 `Replace` 或 `Forbid`，并结合历史记录限制来管理旧 Job。





## CronJob

如果你想定期周期性管理 k8s 中的资源，那就使用 cronjob，就是在 job 一次性任务的基础上加了一个时间调度。

~~~alert type=note
cronjob控制器资源-------------》job控制器资源-------》启动pod
~~~

>Cronjob 设置计划任务的方式和系统级别的计划任务是一样的。
>
>~~~bash
># ┌────────── 分钟 (0 - 59)
># │ ┌──────── 小时 (0 - 23)
># │ │ ┌────── 日 (1 - 31)
># │ │ │ ┌──── 月 (1 - 12)
># │ │ │ │ ┌── 星期 (0 - 7，0 和 7 都代表周日)
># │ │ │ │ │
># * * * * * <user-name> <要执行的命令>
>~~~

#### 主要参数

~~~yaml
# complete-cronjob.yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: complete-cronjob
  namespace: default
spec:
  # 调度时间表（必需）
  schedule: "0 3 * * *"  # 每天凌晨3点执行
  
  # 启动截止时间（秒）
  startingDeadlineSeconds: 600  # 如果错过调度时间，10分钟内仍会启动
  
  # 并发策略
  concurrencyPolicy: Forbid  # 禁止并发执行
  
  # 成功 Job 历史记录保留数量
  successfulJobsHistoryLimit: 3
  
  # 失败 Job 历史记录保留数量  
  failedJobsHistoryLimit: 1
  
  # 是否挂起 CronJob
  suspend: false  # true 表示暂停调度
  
  # Job 模板（必需）
  jobTemplate:
    spec:
      # Job 级别的配置
      activeDeadlineSeconds: 3600  # Job 最多运行1小时
      backoffLimit: 2  # 失败重试次数
      
      template:
        spec:
          restartPolicy: OnFailure
          containers:
          - name: report-generator
            image: busybox
            command:
            - /bin/sh
            - -c
            - |
              echo "开始生成日报..."
              date
              echo "报告生成时间: $(date)" > /tmp/report.txt
              echo "生成完成"
~~~



#### 并发策略

CronJob 有三种并发策略，通过 `concurrencyPolicy` 字段指定：

- Allow。默认值。允许并发 Job 执行。如果一个任务还没有结束，而下一次调度时间到了，新的任务会直接启动，新建另一个 Job
- Forbid。不允许并发执行；如果新 Job 的执行时间到了而老 Job 没有执行完，CronJob 会忽略新 Job 的执行即新 job 不会启动。适用于只允许一个任务实例运行的场景。
- Replace。如果新 Job 的执行时间到了而老 Job 没有执行完，那么老的会被终止 Terminating 掉，CronJob 会用新 Job 替换当前正在运行的 Job。

~~~yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: my-cronjob
spec:
  schedule: "*/1 * * * *"  # 每1分钟执行一次，但每次pod的执行任务故意超过1分钟来验证允许并发
  concurrencyPolicy: Allow	# 并发策略
  jobTemplate:
    spec:
      template:
        spec:
          containers:
          - name: my-container
            image: busybox
            command: ["sh","-c","for i in `seq 1 120`;do echo $i;sleep 1;done"]
          restartPolicy: OnFailure
~~~



#### 历史记录管理

CronJob 会自动保留一定数量的历史 Job 记录

~~~yaml
successfulJobsHistoryLimit: 3  # 保留最近3个成功Job记录，默认值 3
failedJobsHistoryLimit: 1      # 保留最近1个失败Job记录，默认值 1
~~~

手动清理

~~~bash
# 手动清理已完成的 Job
kubectl delete jobs --field-selector status.successful=1

# 查看所有 Job（包括已完成的）
kubectl get jobs --show-all
~~~

#### 常用命令

~~~bash
# 查看所有 CronJob
kubectl get cronjobs --all-namespaces

# 查看特定 CronJob 详情
kubectl describe cronjob <cronjob-name>

# 查看 CronJob 创建的 Job
kubectl get jobs --selector=job-name=<cronjob-prefix>

# 手动触发 CronJob 执行
kubectl create job --from=cronjob/<cronjob-name> manual-$(date +%s)

# 暂停 CronJob
kubectl patch cronjob <cronjob-name> -p '{"spec":{"suspend":true}}'

# 恢复 CronJob  
kubectl patch cronjob <cronjob-name> -p '{"spec":{"suspend":false}}'

# 删除 CronJob（会删除相关的 Job）
kubectl delete cronjob <cronjob-name>
~~~



## HPA

HPA（Horizontal Pod Autoscaler）是 Kubernetes 中用于**自动水平扩展 Pod 数量**的控制器。它根据监控指标（如 CPU、内存使用率或自定义指标）自动调整 Deployment、ReplicaSet 或 StatefulSet 的 Pod 副本数量。

**核心特性：**

- 基于 CPU/内存使用率自动扩缩容
- 支持自定义指标和外部指标
- 可配置扩缩容阈值和行为
- 与 Metrics Server 集成

~~~alert type=note
HPA Controller → 查询 Metrics API → 获取当前指标值 → 计算所需副本数 → 调整 Deployment/ReplicaSet
~~~

HPA 控制器资源可以从一系列的 API 中获取 pod 的运行指标，然后通过自己内部计算算法来计算出应该扩容/缩容的副本数。

- 常规指标。一般由 metrics-server 提供例如cpu、内存指标。
- 自定义指标。自定义指标通常是从 Kubernetes 集群内部的应用程序中收集的。这些指标可以通过 Prometheus、Metrics Server 或其他监控工具采集，通常与特定应用程序或 Pod 的性能相关。
- 外部指标。外部指标允许 HPA 根据 Kubernetes 集群外部的服务指标来进行扩展。通常是由外部的监控系统或服务提供的。 这些指标不依赖于 Kubernetes 内部的资源，可能包括外部服务的响应时间、消息队列的长度、数据库的请求数等。

>HPA v1 版，需要借助 Heapster 获取 CPU 和内存指标。
>
>HPA v2 版，需要借助 Metrics Server 获取需要的指标。

