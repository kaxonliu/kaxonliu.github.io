# 控制器



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



#### 滚动更新策略





