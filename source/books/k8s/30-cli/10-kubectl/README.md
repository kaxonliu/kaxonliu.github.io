# kubectl

`kubectl` 是 Kubernetes 官方的命令行工具，用于与 Kubernetes 集群的 API Server 进行交互。它是管理和控制 Kubernetes 集群最常用、最强大的工具。



## 基本语法

~~~bash
kubectl [command] [TYPE] [NAME] [flags]
~~~

- **command**： 要执行的操作，如 `get`, `create`, `apply`, `delete` 等。
- **TYPE**： 资源类型，如 `pod`, `service`, `deployment`, `node` 等。支持单数、复数或缩写形式（如 `po`, `svc`, `deploy`）。
- **NAME**： 资源的具体名称（区分大小写）。如果不指定，则对所有资源进行操作。

~~~bash
# 查看多个pod
kubectl get pod xxx yyy

# 查看 多个资源
kubectl get pod/xxx deployment/aaa
~~~

- **flags**： 可选的标志，如 `-n` 指定命名空间，`--all-namespaces` 指定所有命名空间，`-o wide` 表示以更宽的格式输出信息。



## 常用用法

#### 查看和查找资源

| 命令               | 作用                                 | 示例                                                         |
| :----------------- | :----------------------------------- | :----------------------------------------------------------- |
| `kubectl get`      | 列出一个或多个资源                   | `kubectl get pods`、 `kubectl get deploy, pod/xxx`           |
| `kubectl describe` | 显示资源的详细状态和信息（用于调试） | `kubectl describe pod/my-pod`                                |
| `kubectl logs`     | 查看 Pod 中容器的日志                | `kubectl logs my-pod` `kubectl logs -f my-pod` （实时跟踪） `kubectl logs my-pod -c my-container` （多容器Pod） |
| `kubectl exec`     | 在 Pod 的容器中执行命令              | `kubectl exec -it my-pod -- /bin/sh` `kubectl exec my-pod -- ls /app` |

补充：`kubectl describe` 展示的结果字段 `Controlled By` 表示该资源的管理者。`Events` 记录了创建过程，如果操作失败可以在这里查找原因。





####  创建和管理资源

| 命令             | 作用                                                         | 示例                                                         |
| :--------------- | :----------------------------------------------------------- | :----------------------------------------------------------- |
| `kubectl create` | 从文件或标准输入创建资源                                     | `kubectl create -f deployment.yaml`                          |
| `kubectl apply`  | **（推荐）** 从文件或标准输入应用配置。如果资源不存在则创建，存在则更新。 | `kubectl apply -f deployment.yaml` 、`kubectl apply -f ./manifests/` （应用目录下所有文件） |
| `kubectl delete` | 删除资源                                                     | `kubectl delete -f deployment.yaml`、 `kubectl delete pod my-pod`、 `kubectl delete deploy my-deploy --force --grace-period=0` （强制删除） |



#### 更新和修复资源

| 命令            | 作用                         | 示例                                    |
| :-------------- | :--------------------------- | :-------------------------------------- |
| `kubectl edit`  | 直接编辑服务器上的资源       | `kubectl edit deploy web`               |
| `kubectl scale` | 扩缩容 Deployment/ReplicaSet | `kubectl scale deploy/web --replicas=3` |



#### 交互与调试

| 命令           | 作用                          | 示例                                                         |
| :------------- | :---------------------------- | :----------------------------------------------------------- |
| `kubectl cp`   | 在本地和 Pod 容器之间复制文件 | `kubectl cp /local/file my-pod:/app/file` `kubectl cp my-pod:/app/logs/ /local/dir` |
| `kubectl exec` | 进入POD                       | `kubectl exec -ti pod名 bash`                                |



## 常用技巧和标志

- **指定命名空间：** `-n <namespace>` 或 `--namespace <namespace>`
```bash
kubectl get pods -n kube-system
```
- **所有命名空间：** `-A` 或 `--all-namespaces`
```bash
kubectl get pods -A
```
- **输出格式：** `-o wide`, `-o json`, `-o yaml`, `-o name`, `-o custom-columns=...`

```bash
kubectl get nodes -o wide        # 显示更详细的信息
kubectl get pod my-pod -o yaml   # 以 YAML 格式输出
kubectl get pods -o name         # 只输出资源名称
```
- **标签筛选器：** `-l <selector>`
```bash
kubectl get pods -l app=nginx
kubectl delete pods -l app=nginx
```
- **动态查看：** `-w` 或 `--watch`
```bash
kubectl get pods -w      
```



## kubectl 插件

创建 kubectl 插件，就是自定义 kubectl 子命令。创建过程如下：

第一步：创建一个可执行文件。文件名必须以 `kubectl-` 开头，后面根上子命令的名字。比如，文件名为 `kubectl-hello` 的文件。

~~~bash
#!/bin/bash

echo "hello world"
~~~



第二步：添加可执行权限

~~~bash
chmod +x kubectl-hello
~~~



第三步：把插件放到 `$PATH` 下面，例如，移动到 `/usr/local/bin/`

~~~bash
mv ./kubectl-hello /usr/local/bin
~~~



使用插件

~~~bash
kubectl hello
~~~

删除插件

~~~bash
rm -rf /usr/local/bin/kubectl-hello
~~~



查看所有插件

~~~bash
kubectl plugin list
~~~





## 创建和删除资源

K8S 资源有 pod、service、namespace、replicaset、deployment、statefulSet等等。

| 类别               | 名称                                                         |
| ------------------ | ------------------------------------------------------------ |
| 工作负载型资源对象 | Pod、ReplicaSet、ReplicationController、Deployment、StatefulSet、DaemonSet、Job、CronJob |
| 服务发现及负载均衡 | Service、Ingress                                             |
| 配置与存储         | Volume、Persistent Volume、CSI、ConfigMap、Secret            |
| 集群资源           | Namespace、Node、Role、ClusterRole、RoleBinding、ClusterRoleBinding |
| 元数据资源         | HPA、PodTemplate、LimitRange                                 |



#### kubectl 命令直接创建资源

kubectl 命令直接创建，在命令行中通过参数指定资源属性。特点：简单、直观、方便。

~~~bash
# 该命令只能创建pod，创建了一个名为pod-test的pod资源
kubectl run pod-test --image=nginx:1.8.1

# 也可以启用临时容器进行测试
kubectl run alpine --rm -ti --image=alpine -- /bin/sh # exit退出后则删除
kubectl run net-test --image=alpine sleep 36000
~~~

若非测试，不推荐使用 `kubectl run`，因为该命令只创建 pod，该 pod 没有被管理，一旦挂掉了就挂掉了。
建议使用 `kubectl create` 来创建控制器 pod，这样 pod 会被控制器管理起来，死掉了也可以被控制器重启。

~~~bash
kubectl create deployment web --image=nginx:1.14
~~~



#### 使用 yaml 文件创建

创建 yaml 文件，然后使用 `kubectl create`  或 `kubectl apply` 创建资源。在 yaml 文件中描述创建参数。

~~~bash
# 只能创建
kubectl create -f nginx.yaml

# 创建或更新
kubectl apply -f nginx.yaml
~~~

配置文件 nginx.yaml

~~~yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
  labels:
    app: nginx
spec:
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
        image: nginx:1.18
        ports:
        - containerPort: 80
~~~



#### 删除资源

~~~bash
# 方式1
kubectl delete deploy web

# 方式2
kubectl delete -f web.yaml
~~~



## 水平扩缩容

使用如下命令删除 yaml 模板。主要点在于：`--dry-run=client` 表示测试不实际执行命令。

~~~bash
kubectl create deployment web --image=nginx:1.14 --dry-run=client -o yaml > web.yaml
~~~

#### 扩缩容三种方式

1. 手动打开 yaml 文件，修改 `replicas` 的值，然后使用 `kubectl apply -f xxx.yaml` 完成扩缩容。
2. 使用 `kubectl edit deploy  centos-deployment` 自动打开 yaml 文件，修改 `replicas` 的值，保存退出后自动扩缩容。
3. 使用 `kubectl sacle deploy web --replicas=3` 完成扩缩容。



## 节点污点

调度 pod 前需要预选节点，预选时有一些规则，比如节点污点。打污点以后就不会被调度。

#### 查看节点污点信息

~~~bash
kubectl describe node k8s-master-01 | grep Taints
~~~



#### 污点策略说明

- NoSchedule：Pod 一定不会被调度到该节点
- PreferNoSchedule：Pod 尽量不被调度到该节点（仍有被调度的可能）
- NoExecute：Pod 不会被调度，并且会驱逐节点上已有的 Pod



~~~alert type=note
静态 pod 不受污点影响，直接由 kubelet 在当前节点直接拉起运行，后面不会被调度到其他节点。
~~~





#### 污点管理命令

~~~bash
# 添加污点：
kubectl taint nodes k8s-master-01 node-role.kubernetes.io/control-plane:NoSchedule

#删除污点：
kubectl taint nodes k8s-master-01 node-role.kubernetes.io/control-plane:NoSchedule-


kubectl taint nodes k8s-node-01 node:NoExecute
kubectl taint nodes k8s-node-01 node:NoExecute-
~~~



#### 容忍机制

容忍机制允许 Pod 被调度到带有匹配污点的节点上，它通过给 pod 配置与污点的键、值和效果相匹配的规则来实现。





## 查看 YAML 文件字段信息

编写 yaml 文件时忘记每个层级字段信息，可以使用 `kubectl explain` 查看每个层级字段信息提示。

~~~bash
kubectl explain deployment
kubectl explain deployment.spec.template
kubectl explain deployment.spec.template.spec.containers
kubectl explain deployment.spec.template.spec.containers.resources.limits
~~~





## pod 调度策略

调度分为：预选阶段、优先阶段。在预选阶段会使用一些调度策略。常用的有三种调度策略。

#### 根据资源

定义 Pod 时会使用 `requests` 和 `limits` 指定 pod 需要的资源和资源上限。那么此时会根据这两个参数选择满足条件的节点。

~~~yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: centos-deployment
  labels:
    app: centos
spec:
  replicas: 3
  selector:
    matchLabels:
      app: centos
  template:
    metadata:
      labels:
        app: centos
    spec:
      containers:
      - name: centos
        image: centos:7
        command: ["sh", "-c", "while true;do echo 111 ;sleep 1;done"]
        resources:
          limits:
            memory: "3Gi"
          requests:
            memory: "1Gi"
~~~

~~~alert type=note
如果资源不满足，会影响 pod 调度，pod 会一直处于 Pending 的状态。
~~~



#### 节点标签选择器

节点可以打标签，创建 pod 时可以指定 pod 调度在特定标签的节点上。甚至还可以指定 pod 创建在特定名字的节点上。

**查看节点的标签**

~~~bash
# 查看节点标签
kubectl get nodes --show-labels

# 使用 describe 查看节点信息，在输出中查找 Labels 部分
kubectl describe node k8s-node-01
~~~

**给节点打标签**

打标签就是给节点添加一个键值对信息。

~~~bash
kubectl label node k8s-node-01 gpu=available
~~~

**删除节点标签**

~~~bash
kubectl label node k8s-node-01 gpu-
~~~

**使用 `nodeSelector` 指定 pod 调度时选择的节点**

~~~yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: centos-deployment
  labels:
    app: centos
spec:
  replicas: 3
  selector:
    matchLabels:
      app: centos
  template:
    metadata:
      labels:
        app: centos
    spec:
      containers:
      - name: centos
        image: centos:7
        command: ["sh", "-c", "while true;do echo 111 ;sleep 1;done"]
      nodeSelector:
        gpu: available
~~~

可以使用多标签，此时 Pod 只会调度到同时满足所有标签条件的节点

~~~yaml
apiVersion: v1
kind: Pod
metadata:
  name: multi-label-pod
spec:
  containers:
  - name: app
    image: myapp:latest
  nodeSelector:
    disktype: ssd
    zone: us-east
    environment: staging
~~~



**使用 `nodeName` 直接指定节点名** 

~~~yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web
spec:
  template:
    spec:
      nodeName: node01  # 直接指定节点名称
      containers:
      - name: nginx
        image: nginx:1.14
~~~



#### 节点亲和

节点亲和分为硬亲和与软亲和。前者必须满足，后者尝试满足、不强制。

节点亲和和节点标签选择器类似，只不过多了一些操作符表达式，比如：`In`、`NotIn`、`Exists`、`Gt`、`Lt` 、`DoesNotExists` 等。

注意：在 pod 资源基于节点亲和规则调度到某个节点之后，如果该节点的标签发生了变化，调度器不会将 pod 从该节点移除。因为该规则仅对新建的 pod 有效。

##### 硬亲和

硬亲和使用 `requiredDuringSchedulingIgnoredDuringExecution` 表示必须运行在满足条件的节点上。比如下面的例子表示必须运行在带有 `kubernetes.io/hostname=node01` 的节点上。

~~~yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web
spec:
  template:
    spec:
      containers:
      - image: nginx: 1.14
        name: nginx
      affinity:
        nodeAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:   # 硬亲和
            nodeSelectorTerms:
            - matchExpressions:
              - key: kubernetes.io/hostname
                operator: In
                values:
                - node01  # 指定具体节点名
~~~

##### 软亲和

软亲和使用 `preferredDuringSchedulingIgnoredDuringExecution` 软亲和性，优先但不强制，如果没有合适节点，仍会调度到其他节点。可以指定权重值，范围是 1-100

~~~yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web
spec:
  template:
    spec:
      containers:
      - image: nginx: 1.14
        name: nginx
      affinity:
        nodeAffinity:
          preferredDuringSchedulingIgnoredDuringExecution:
          - weight: 80  # 高优先级：优先选择SSD节点
            preference:
              matchExpressions:
              - key: disktype
                operator: In
                values:
                - ssd
          - weight: 20  低优先级：优先选择所在区域
            preference:
              matchExpressions:
              - key: zone
                operator: In
                values:
                - zone-a
~~~



## Secret 配置管理

Secret 用于存储敏感信息，如密码、OAuth令牌、ssh密钥等。

#### 使用 YAML 文件创建 secret 资源

编写 secret.yaml 文件，然后使用 `kubectl apply` 创建资源。

~~~yaml
apiVersion: v1
kind: Secret
metadata:
  name: my-secret
type: Opaque
data:
  username: YWRtaW4=  # base64 编码的 "admin"
  password: c2VjcmV0MTIz  # base64 编码的 "secret123"
~~~

编码方法

~~~bash
echo -n "admin" | base64
echo -n "secret123" | base64
~~~

创建 secret 资源

~~~bash
kubectl apply -f secret.yaml
~~~

查看

~~~bash
kubectl get secret
kubectl get secret my-secret
~~~

#### 作为文件挂载

yaml 文件如下，使用 `volumeMounts` 指定挂载路径。

~~~yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: centos-deployment
  labels:
    app: centos
spec:
  replicas: 3
  selector:
    matchLabels:
      app: centos
  template:
    metadata:
      labels:
        app: centos
    spec:
      containers:
      - name: centos
        image: centos:7
        command: ["sh", "-c", "while true;do echo 111 ;sleep 1;done"]
        volumeMounts:
        - name: secret-volume
          mountPath: /etc/secrets
      # 声明卷
      volumes:
      - name: secret-volume
        secret:
          secretName: my-secret
~~~

创建好 pod 之后，进入 pod ，然后查看文件（文件名默认是 secret 中 `data` 下面配置的字段名）。会自动 base64 解码。

~~~bash
cat /etc/secrets/username
cat /etc/secrets/username
~~~



#### 作为环境变量

yaml 文件如下，使用 `env` 指定环境变量。

~~~yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: centos-deployment
  labels:
    app: centos
spec:
  replicas: 3
  selector:
    matchLabels:
      app: centos
  template:
    metadata:
      labels:
        app: centos
    spec:
      containers:
      - name: centos
        image: centos:7
        command: ["sh", "-c", "while true;do echo 111 ;sleep 1;done"]
        env:
        - name: XXX_USERNAME
          valueFrom:
            secretKeyRef:
              name: my-secret
              key: username
        - name: XXX_PASSWORD
          valueFrom:
            secretKeyRef:
              name: my-secret
              key: password
~~~

创建好 pod 之后，进入 pod ，然后查看环境变量。会自动 base64 解码。

~~~bash
[root@centos-deployment-5f7ff9c6cc-7879m /]# env | grep XXX
XXX_USERNAME=admin
XXX_PASSWORD=secret123
~~~



## ConfigMap

ConfigMap 用于存储非敏感的配置数据，如配置文件、环境变量、命令行参数等。

创建一个配置文件 redis.conf

~~~ini
redis.host=127.0.0.1
redis.port=6379
redis.password=12345
~~~

基于文件 redis.conf 创建一个名为 redis-config 的 ConfigMap

~~~bash
kubectl create configmap redis-config --from-file=redis.conf
~~~

查看 redis-config

~~~bash
[root@k8s-master-01 ~]# kubectl get cm redis-config -o yaml
apiVersion: v1
data:
  # 属性形式 key-value
  redis.conf: |
    redis.host=127.0.0.1
    redis.port=6379
    redis.password=12345
kind: ConfigMap
metadata:
  creationTimestamp: "2025-11-08T03:06:25Z"
  name: redis-config
  namespace: default
  resourceVersion: "62545"
  uid: f4277ba0-ddd0-493b-b41a-b2955a6a31a8
~~~

#### 作为文件挂载

使用 `configMap` 配置配置卷。

~~~yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: centos-deployment
  labels:
    app: centos
spec:
  replicas: 3
  selector:
    matchLabels:
      app: centos
  template:
    metadata:
      labels:
        app: centos
    spec:
      containers:
      - name: centos
        image: centos:7
        command: ["sh", "-c", "while true;do echo 111 ;sleep 1;done"]
        volumeMounts:
        - name: config-volume
          mountPath: /etc/redis.conf
          subPath: redis.conf
      volumes:
      - name: config-volume
        configMap:
          name: redis-config
~~~

创建 pod 后，在 pod 中的文件 `/etc/redis.conf` 里面查看到配置数据。

~~~bash
[root@centos-deployment-7496cff6d8-psgz4 ~]# cat /etc/redis.conf 
redis.host=127.0.0.1
redis.port=6379
redis.password=12345
~~~

如果不使用 `subPath`，则会有如下如下层级关系。

~~~bash
[root@centos-deployment-6fc875c7c-6kdvn /]# ls /etc/redis.conf/
redis.conf
[root@centos-deployment-6fc875c7c-6kdvn /]# cat /etc/redis.conf/redis.conf 
redis.host=127.0.0.1
redis.port=6379
redis.password=12345
~~~



#### 作为环境变量

yaml 文件如下，使用 `envFrom` 指定环境量。

~~~yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: centos-deployment
  labels:
    app: centos
spec:
  replicas: 3
  selector:
    matchLabels:
      app: centos
  template:
    metadata:
      labels:
        app: centos
    spec:
      containers:
      - name: centos
        image: centos:7
        command: ["sh", "-c", "while true;do echo 111 ;sleep 1;done"]
        
        # 所有键值对作为环境变量
        envFrom:
        - configMapRef:
            name: redis-config
~~~

创建好 pod 之后，进入 pod ，然后查看环境变量。

因为上述 configMap `redic.conf` 内部定义的是 key-value 的属性形式，无法单独在 yaml 文件中单独设置环境变量，更推荐使用 单个属性，方便使用。

game.yaml

~~~yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: game-config
data:
  log-level: "DEBUG"
  server-url: "api.example.com"
~~~

创建 game-config

~~~bash
kubectl apply -f game.yaml
~~~

deployment 中以环境变量的方式使用

~~~yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: centos-deployment
  labels:
    app: centos
spec:
  replicas: 3
  selector:
    matchLabels:
      app: centos
  template:
    metadata:
      labels:
        app: centos
    spec:
      containers:
      - name: centos
        image: centos:7
        command: ["sh", "-c", "while true;do echo 111 ;sleep 1;done"]
        
        env:
        # 单个环境变量
        - name: GAME_LOG_LEVEL
          valueFrom:
            configMapKeyRef:
              name: game-config
              key: log-level
~~~

pod 中查看环境变量值

~~~bash
[root@centos-deployment-86d45ff5fc-trzn4 /]# echo $GAME_LOG_LEVEL
DEBUG
~~~



## 存储编排

自动挂载所选存储系统，包括本地存储、诸如 GCP 或 AWS 之类公有云提供商所提供的存储或者诸如 NFS、iSCSI、Gluster、Ceph、Cinder 或 Flocker 这类网络存储系统。

提到存储就不得不说 K8s 中的 PV 和 PVC 了，解释如下：

- PV：PersistentVolume，持久化卷，用于声明底层存储类型，关联的是底层存储，例如存储路径
- PVC：PersistentVolumeClaim，持久化卷声明，用于上层pod定制自己的对存储的需求参数，例如大小（不关心底层存储是什么）

PV 说白了就是一层存储的抽象，底层的存储可以是本地磁盘，也可以是网络磁盘比如 NFS、Ceph 之类，既然有了 PV 那为什么又要搞一个 PVC 呢？

PVC 其实在 Pod 和 PV 之前又增加了一层抽象，这样做的目的是将 Pod 的存储行为与具体的存储设备解耦，好处是：如果哪天我们变更底层存储（例如更换路径、IP、或者换成其他存储），只需要变更 PV 即可，POD 对存储的需求参数都放在 PVC 里根本不需要变动，如此，便十分灵活。



### 本地存储 hostPath

本地存储对应的是 k8s 中的 hostPath。操作过程如下：

- 准备本地硬盘
- 创建 PV
- 创建 PVC
- 创建 deploy



#### 准备本地硬盘

比如说直接使用根分区所在的硬盘，计划选择 `/data/hostpath` 作为存储目录。



#### 创建 PV

新建一个 pv.yaml，声明本地存储路径为：`/data/hostpath`， 1GB 容量，支持多节点读写访问。

~~~yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: my-pv
  labels:
    type: local
spec:
  storageClassName: manual
  capacity:
    storage: 1Gi
  accessModes:
    - ReadWriteMany
  hostPath:
    path: "/data/hostpath"
~~~

创建 pv，查看 pv

~~~bash
[root@k8s-master-01 ~]# kubectl apply -f pv.yaml 
persistentvolume/my-pv created
[root@k8s-master-01 ~]# 
[root@k8s-master-01 ~]# kubectl get pv
NAME    CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS      CLAIM   STORAGECLASS   VOLUMEATTRIBUTESCLASS   REASON   AGE
my-pv   1Gi        RWX            Retain           Available           manual         <unset>                          5s
~~~

**强调说明**：

- 新建的 PV 状态是 `Available`，此时还没有被 PVC 绑定。绑定后状态会变化。
- 创建 PV 后在 k8s 集权中任何一个节点都可以使用 `kubectl get pv` 查看到。

**访问模式**，用来对 PV 进行访问模式的设置，用于描述用户应用对存储资源的访问权限。有如下模式：

- ReadWriteOnce (RWO)：读写权限，但是只能被单个节点挂载
- ReadOnlyMany (ROX)：只读权限，可以被多个节点挂载
- ReadWriteMany (RWX)：读写权限，可以被多个节点挂载





#### 创建 PVC

新建一个 pvc.yaml，申请 1GB 容量、支持多节点读写访问的持久化存储卷。

~~~yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: my-pvc
spec:
  storageClassName: manual
  accessModes:
  - ReadWriteMany
  resources:
    requests:
      storage: 1Gi
~~~

创建 pvc，查看 pvc

~~~bash
[root@k8s-master-01 ~]# kubectl apply -f pvc.yaml 
persistentvolumeclaim/my-pvc created
[root@k8s-master-01 ~]# 
[root@k8s-master-01 ~]# kubectl get pvc
NAME     STATUS   VOLUME   CAPACITY   ACCESS MODES   STORAGECLASS   VOLUMEATTRIBUTESCLASS   AGE
my-pvc   Bound    my-pv    1Gi        RWX            manual         <unset>                 5s
[root@k8s-master-01 ~]# 
[root@k8s-master-01 ~]# kubectl get pv
NAME    CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS   CLAIM            STORAGECLASS   VOLUMEATTRIBUTESCLASS   REASON   AGE
my-pv   1Gi        RWX            Retain           Bound    default/my-pvc   manual         <unset>                          12m
~~~

创建 PVC 之后，k8s 就会去查找满足条件（`storageClassName`、`accessModes` 以及容量等要求）的 PV，如果找到满足条件的 PV，就会将 PV 和 PVC 绑定在一起。一个 PV 只能被一个 PVC 绑定。绑定后，PVC 和 PV 的状态都为 `Bound`



#### 创建 Deploy

新建 centos.yaml，使用 PVC，将名为 `my-pvc` 的持久化存储卷挂载到容器的 `/xxx/yyy` 目录。

~~~yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: centos-deployment
  labels:
    app: centos
spec:
  replicas: 3
  selector:
    matchLabels:
      app: centos
  template:
    metadata:
      labels:
        app: centos
    spec:
      containers:
      - name: centos
        image: centos:7
        command: ["sh", "-c", "while true;do echo 111 ;sleep 1;done"]
        # 挂载到容器内
        volumeMounts:
        - name: my-volume
          mountPath: /xxx/yyy
        
      # PVC声明
      volumes:
      - name: my-volume
        persistentVolumeClaim:
          claimName: my-pvc
~~~

创建应用

~~~bash
kubectl apply -f centos.yaml
~~~

创建后，会把 主机的目录 `/data/hostpath` 绑定到容器内的 `/xxx/yyy` 目录，此时容器内目录是空的，因为主机上目前还没有创建目录 `/data/hostpath`。



#### 创建主机目录

pod 在哪个节点上就在哪个节点上创建目录 `/data/hostpath`，然后在目录下面创建文件。

创建后在 pod 内就可以看到绑定目录下面的文件了。pod 所在的节点上没有创建目录，pod 内就看不到文件。

创建目录和文件

~~~bash
mkdir -p /data/hostpath
echo "hello world" > /data/hostpath/hello.txt
~~~

查看 pod 所在的节点

~~~bash
kubectl get pods -o wide
~~~

进入 pod

~~~bash
kubectl exec -it <pod> -- bash
~~~

~~~alert type=important
一般不会使用本地存储持久化数据，因为 Pod 调度不是固定的。一般会使用网络的方式存储数据。
~~~



### 网络存储

使用 nfs 作为网络存储工具。可以使用任意一台机器作为 NFS 服务器，k8s 集群内或者集群外都的都可以。

#### 部署 nfs

nfs 客户端安装软件（指的就是 k8s 集群内的所有 node 节点）

~~~bash
yum install -y nfs-utils
~~~

nfs 服务端安装软件

~~~bash
yum install -y nfs-utils rpcbind
~~~

nfs 服务端创建共享目录

~~~bash
mkdir -p /data/nfs
~~~

nfs 服务端配置共享目录

~~~bash
cat > /etc/exports << EOF
/data/nfs *(rw,no_root_squash)
EOF
~~~

nfs 服务端启动 nfs 服务

~~~bash
systemctl start nfs
~~~

查看是否启动成功

~~~bash
ps aux | grep nfs
~~~

#### 修改 PV

使用 NFS 存储的 PersistentVolume 配置，创建了一个 1GB 容量、支持多节点读写访问的远程持久化卷。

~~~yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: my-pv
  labels:
    type: remote
spec:
  storageClassName: manual
  capacity:
    storage: 1Gi
  accessModes:
    - ReadWriteMany
  nfs:
    path: /data/nfs
    server: 192.168.10.81
~~~

如果由上面的 本地存储转为 NFS ，只需要修改 pv.yaml 文件，pvc 和 deploy 的 yaml 文件不需要修改。



#### 运行服务

依次删除上面使用本地存储创建的 deployment、pvc、pv

~~~bash
kubectl delete -f centos.yaml
kubectl delete -f pvc.yaml
kubectl delete -f pv.yaml
~~~

然后再创建新的  pv pvc deployment

~~~bash
kubectl apply -f pv.yaml
kubectl apply -f pvc.yaml
kubectl apply -f centos.yaml
~~~

在 nfs 服务端写入文件

~~~bash
echo "hello world" > /data/nfs/hello.txt
~~~

进入 pod 并查看数据

~~~bash
[root@k8s-node-01 ~]# kubectl  get pod -o wide
NAME                                READY   STATUS    RESTARTS   AGE    IP             NODE          NOMINATED NODE   READINESS GATES
centos-deployment-b557b856d-6b2xs   1/1     Running   0          102s   10.244.2.163   k8s-node-02   <none>           <none>
centos-deployment-b557b856d-8stqm   1/1     Running   0          102s   10.244.1.108   k8s-node-01   <none>           <none>
centos-deployment-b557b856d-jkjxs   1/1     Running   0          102s   10.244.1.109   k8s-node-01   <none>           <none>
[root@k8s-node-01 ~]#
[root@k8s-node-01 ~]# kubectl  exec -it centos-deployment-b557b856d-jkjxs -- bash
[root@centos-deployment-b557b856d-jkjxs /]# cat /xxx/yyy/hello.txt 
hello world
~~~



## 服务发现和负载均衡

#### 1. 为什么要使用 service

一个应用有多个副本（多个 pod），访问时必须要引入负载均衡。引入负载均衡要考虑两件事情：

- 流量转发及其效率问题
- 副本增加或减少，更新负载均衡配置。

~~~alert type=note
访问者 --- service --- 转发流量 --- pod
~~~

k8s 中的逻辑概念 service 就是用来做这些事情的，

- service 采用 ipvs 高效转发流程
- 扩缩容时，service 代理的 pod 会自动更新。

~~~alert type=note
kube-proxy 组件 --- service --- pod(pod的IP+端口) <br>
service 称之为智能负载均衡 <br>
service 基于 ip + 端口代理 pod, 是四层负载均衡
~~~



#### 2. 创建 pod 服务

新建 nginx.yaml 文件，声明创建 nginx 应用，有 3 个 副本，**每个应用定义标签 `app=nginx-app`**

~~~yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
  labels:
    app: nginx-app
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nginx-app
  template:
    metadata:
      labels:
        app: nginx-app
    spec:
      containers:
      - name: nginx
        image: nginx:1.14
~~~

创建 deploy，并查看每个 pod 的信息。

~~~bash
kubectl apply -f nginx.yaml
[root@k8s-master-01 ~]# kubectl get pod -o wide
NAME                               READY   STATUS    RESTARTS   AGE     IP             NODE          NOMINATED NODE   READINESS GATES
nginx-deployment-85999cd7d-qdpqg   1/1     Running   0          3m17s   10.244.1.123   k8s-node-01   <none>           <none>
nginx-deployment-85999cd7d-sh5jc   1/1     Running   0          4m58s   10.244.2.171   k8s-node-02   <none>           <none>
nginx-deployment-85999cd7d-x7lfb   1/1     Running   0          10m     10.244.2.170   k8s-node-02   <none>           <none>
~~~



定制每个Pod 中 nginx 服务的默认页面，方便测试。

~~~bash
 kubectl exec nginx-deployment-85999cd7d-qdpqg -- sh -c "echo 111 > /usr/share/nginx/html/index.html"
 kubectl exec nginx-deployment-85999cd7d-sh5jc -- sh -c "echo 222 > /usr/share/nginx/html/index.html"
 kubectl exec nginx-deployment-85999cd7d-x7lfb -- sh -c "echo 333 > /usr/share/nginx/html/index.html"
~~~

测试页面

~~~bash
[root@k8s-master-01 ~]# curl http://10.244.1.123
111
[root@k8s-master-01 ~]# curl http://10.244.2.171
222
[root@k8s-master-01 ~]# curl http://10.244.2.170
333
~~~



#### 3. 创建 service

新建 nginx-service.yaml，声明将 Service 的 8080 端口流量转发到后端 Pod 的 80 端。

~~~yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx-app
  labels:
    app: nginx-app
spec:
  selector:
    app: nginx-app  # 必须添加selector来匹配Pod
  ports:
  - port: 8080     # 指定 Service 的端口
    protocol: TCP 
    targetPort: 80 # 转发到后端端口
  type: ClusterIP  # 服务类型，默认是ClusterIP
~~~

创建 service 资源

~~~bash
kubectl apply -f nginx-service.yaml
~~~

并查看 service 信息

~~~bash
[root@k8s-master-01 ~]# kubectl get svc -o wide
NAME         TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)    AGE     SELECTOR
kubernetes   ClusterIP   10.96.0.1        <none>        443/TCP    3d17h   <none>
nginx-app    ClusterIP   10.100.113.241   <none>        8080/TCP   88s     app=nginx-app
~~~

- k8s 集群内部可以的地址为 `ClusterIp+8080`，即 `http://10.100.113.241:8080`
- SELECTOR 值为 ` app=nginx-app`，表示负载均衡时选择的 pod 标签。



查看 service 详细信息

~~~bash
[root@k8s-master-01 ~]# kubectl describe svc nginx-app 
Name:              nginx-app
Namespace:         default
Labels:            app=nginx-app
Annotations:       <none>
Selector:          app=nginx-app
Type:              ClusterIP
IP Family Policy:  SingleStack
IP Families:       IPv4
IP:                10.100.113.241
IPs:               10.100.113.241
Port:              <unset>  8080/TCP
TargetPort:        80/TCP
Endpoints:         10.244.1.123:80,10.244.2.170:80,10.244.2.171:80
Session Affinity:  None
Events:            <none>
~~~

- `TargetPort` 表示转发的端口和协议
- `Endpoints` 表示代理的 endpoints，就是多个 pod的ip和服务端口。
- `TargetPort`  + `Endpoints` 可以理解为 svc  作为负载均衡的配置信息。



查看 endpoint 信息

~~~bash
[root@k8s-node-01 ~]# kubectl  get endpoints
NAME         ENDPOINTS                                         AGE
kubernetes   192.168.10.81:6443                                3d17h
nginx-app    10.244.1.123:80,10.244.2.170:80,10.244.2.171:80   10m
~~~



综上，可以看到创建的 3个 pod 携带标签 `app=nginx-app`，创建的 service 服务选择代理这个标签的所有 pod，当流量访问 svc 的 ip + 8080 端口，svc 就会转发流量到代理的 endpoints，endpoint在创建 svc 是会自动创建。



#### 4. 测试 service 代理服务

~~~bash
[root@k8s-master-01 ~]# curl http://10.100.113.241:8080/index.html
333
[root@k8s-master-01 ~]# curl http://10.100.113.241:8080/index.html
111
[root@k8s-master-01 ~]# curl http://10.100.113.241:8080/index.html
222
~~~



#### 5. 测试 svc 使用 ipvs 代理 pod 内的服务

使用 `ipvsadm` 工具查看 svc 的转发信息

~~~bash
[root@k8s-master-01 ~]# ipvsadm -Ln
IP Virtual Server version 1.2.1 (size=4096)
Prot LocalAddress:Port Scheduler Flags
  -> RemoteAddress:Port           Forward Weight ActiveConn InActConn
TCP  10.96.0.1:443 rr
  -> 192.168.10.81:6443           Masq    1      2          0         
TCP  10.96.0.10:53 rr
  -> 10.244.0.2:53                Masq    1      0          0         
  -> 10.244.2.12:53               Masq    1      0          0         
TCP  10.96.0.10:9153 rr
  -> 10.244.0.2:9153              Masq    1      0          0         
  -> 10.244.2.12:9153             Masq    1      0          0         
TCP  10.100.113.241:8080 rr
  -> 10.244.1.123:80              Masq    1      0          0         
  -> 10.244.2.170:80              Masq    1      0          0         
  -> 10.244.2.171:80              Masq    1      0          0         
UDP  10.96.0.10:53 rr
  -> 10.244.0.2:53                Masq    1      0          0         
  -> 10.244.2.12:53               Masq    1      0          0         
~~~

可以看到 svc 的 ip + 端口（10.100.113.241:8080），使用轮询（`rr`）的方式转发流量到三个 pod 服务上。



#### 6. 测试 svc 支持扩缩容时自动更新负载均衡配置信息

水平扩缩容后，查看 svc 的详细信息、查看 endpoint 的详细信息，都可以发现负载均衡配置自动更新了。

~~~bash
[root@k8s-master-01 ~]# kubectl scale deployment nginx-deployment --replicas=4
deployment.apps/nginx-deployment scaled
[root@k8s-master-01 ~]# kubectl describe svc nginx-app 
Name:              nginx-app
Namespace:         default
Labels:            app=nginx-app
Annotations:       <none>
Selector:          app=nginx-app
Type:              ClusterIP
IP Family Policy:  SingleStack
IP Families:       IPv4
IP:                10.100.113.241
IPs:               10.100.113.241
Port:              <unset>  8080/TCP
TargetPort:        80/TCP
Endpoints:         10.244.1.123:80,10.244.1.124:80,10.244.2.170:80 + 1 more...
Session Affinity:  None
Events:            <none>
[root@k8s-master-01 ~]# kubectl describe endpoints nginx-app 
Name:         nginx-app
Namespace:    default
Labels:       app=nginx-app
Annotations:  endpoints.kubernetes.io/last-change-trigger-time: 2025-11-08T09:21:53Z
Subsets:
  Addresses:          10.244.1.123,10.244.1.124,10.244.2.170,10.244.2.171
  NotReadyAddresses:  <none>
  Ports:
    Name     Port  Protocol
    ----     ----  --------
    <unset>  80    TCP

Events:  <none>
~~~





#### 7. 使用集群内部域名访问 svc

集群内部任意一个节点都可以访问 svc，任何一个 Pod 内都可以访问 svc，网络都是通的，并且还可以使用域名访问。

使用域名访问更合适，因为 svc 的 ip 是动态的。

集群内的域名的完整规则是：`<svc-name>.<namespace>.svc.cluseter.local`，比如上述的 nginx-app 这个 svc 的完整域名就是：`nginx-app.default.svc.cluster.local`



新建一个测试 pod，在pod 内部使用 svc 的 ip 和 域名访问

~~~bash
[root@k8s-master-01 ~]# kubectl run -i --tty --image centos:7 test --rm bash
[root@test /]# curl http://nginx-app.default.svc.cluster.local:8080/index.html
111
~~~



之所以可以在 pod 内部使用域名访问，肯定存在域名解析服务，那就是 coredns

在 pod 内查看 `/etc/resolv.conf`

~~~bash
[root@test /]# cat /etc/resolv.conf 
search default.svc.cluster.local svc.cluster.local cluster.local
nameserver 10.96.0.10
options ndots:5
~~~

可以看到两个事情，

- 域名可以有多种简写方式，比如可以简写为：`nginx-app`、`nginx-app.default`、`nginx-app.default.svc` 等。

- 另外，可以看到域名解析服务的 IP



  可以发现这个 IP 是一个名为 `kube-dns` 的svc 的 ip

~~~BASH
[root@k8s-node-01 ~]# kubectl get -n kube-system svc
NAME       TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)                  AGE
kube-dns   ClusterIP   10.96.0.10   <none>        53/UDP,53/TCP,9153/TCP   3d18h
~~~

进一步查看，发现这个 svc 代理两个 pod

~~~bash
[root@k8s-node-01 ~]# kubectl -n kube-system describe svc kube-dns 
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
Endpoints:         10.244.0.2:53,10.244.2.12:53
Port:              dns-tcp  53/TCP
TargetPort:        53/TCP
Endpoints:         10.244.0.2:53,10.244.2.12:53
Port:              metrics  9153/TCP
TargetPort:        9153/TCP
Endpoints:         10.244.0.2:9153,10.244.2.12:9153
Session Affinity:  None
Events:            <none>
~~~

最后，查看 pod，发现就是 coredns 

~~~bash
[root@k8s-node-01 ~]# kubectl get -n kube-system pod -o wide
NAME                                    READY   STATUS    RESTARTS      AGE     IP              NODE            NOMINATED NODE   READINESS GATES
coredns-6d58d46f65-gcnb5                1/1     Running   4 (9h ago)    3d18h   10.244.2.12     k8s-node-02     <none>           <none>
coredns-6d58d46f65-n9lhz                1/1     Running   0             9h      10.244.0.2      k8s-master-01   <none>           <none>
etcd-k8s-master-01                      1/1     Running   3 (10h ago)   3d18h   192.168.10.81   k8s-master-01   <none>           <none>
kube-apiserver-k8s-master-01            1/1     Running   3 (10h ago)   3d18h   192.168.10.81   k8s-master-01   <none>           <none>
kube-controller-manager-k8s-master-01   1/1     Running   3 (10h ago)   3d18h   192.168.10.81   k8s-master-01   <none>           <none>
kube-proxy-4f9md                        1/1     Running   3 (10h ago)   3d17h   192.168.10.82   k8s-node-01     <none>           <none>
kube-proxy-lvdzw                        1/1     Running   3 (10h ago)   3d17h   192.168.10.83   k8s-node-02     <none>           <none>
kube-proxy-vjm5t                        1/1     Running   3 (10h ago)   3d18h   192.168.10.81   k8s-master-01   <none>           <none>
kube-scheduler-k8s-master-01            1/1     Running   3 (10h ago)   3d18h   192.168.10.81   k8s-master-01   <none>           <none>
~~~



总结：k8s 集群内网络是互通的，svc 可以使用域名访问，这是因为集群内存在一个域名解析服务。域名解析服务由两个 coredns pod 组成，统一由一个名为 `kube-dns` 的 svc 负载均衡代理。



#### 8. service 的四种类型

不同类型的 service 提供的服务不同，上述创建的 svc 是默认的类型，即 `ClusterIp`，这种类型的 svc 支持集群内部访问，集群外部无法访问。除此之外，service 还支持另外三种类型。

- ClusterIp：集群内部访问（默认）
- NodePort：集群外部访问（包含了ClusterIp）。路由到 ClusterIp 服务，这个 ClusterIP 服务会自动创建。
- LoadBalancer：对外访问应用使用，公有云。使用云厂商提供的负载均衡器，可以对外暴露服务。内部的负载均衡器可以路由到 NodePort 服务和 ClusterIP 服务。

- ExternalName：返回 CNAME 和它的值，可以将服务映射到外部域名。



##### 使用 NodePort 创建服务

使用 NodePort 创建的服务，会把内部服务映射到主机上，即可以使用任意 k8s 节点的 IP 和 端口号访问 k8s 集群内的 svc。

因此，创建这种 service 时除了要指定 svc 的端口，还要指定一个 nodePort



**第一步：编写创建 NodePort 服务的yaml 文件**

~~~yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx-app
  labels:
    app: nginx-app
spec:
  type: NodePort  
  selector:
    app: nginx-app  
  ports:
  - port: 8080     
    protocol: TCP 
    targetPort: 80
    nodePort: 38080 
~~~

**第二步：修改 nodePort 的范围**

默认范围是：`30000-32767`。想要调大范围需要修改 apiserver 的配置（`/etc/kubernetes/manifests/kube-apiserver.yaml`），增加参数如下，修改后 apiserver 会自动重启。如果存在多个 apiserver 则需要每个都修改。

~~~bash
- --service-node-port-range=1024-65535
~~~

**第三步：创建 svc **

~~~bash
kubectl apply -f nginx-service.yaml
~~~

**第四步：检查更新与否**

~~~bash
[root@k8s-master-01 ~]# kubectl describe svc nginx-app 
Name:                     nginx-app
Namespace:                default
Labels:                   app=nginx-app
Annotations:              <none>
Selector:                 app=nginx-app
Type:                     NodePort
IP Family Policy:         SingleStack
IP Families:              IPv4
IP:                       10.100.113.241
IPs:                      10.100.113.241
Port:                     <unset>  8080/TCP
TargetPort:               80/TCP
NodePort:                 <unset>  38080/TCP
Endpoints:                10.244.1.123:80,10.244.1.124:80,10.244.2.170:80 + 1 more...
Session Affinity:         None
External Traffic Policy:  Cluster
Events:                   <none>
~~~

**第五步：测试外部访问**

使用 k8s 几点上任意一个节点的 IP + 配置的 `nodePort`端口号访问。

~~~bash
root@k8s-master-01 ~]# kubectl get nodes -o wide
NAME            STATUS   ROLES           AGE     VERSION    INTERNAL-IP     EXTERNAL-IP   OS-IMAGE                KERNEL-VERSION                CONTAINER-RUNTIME
k8s-master-01   Ready    control-plane   3d19h   v1.30.14   192.168.10.81   <none>        CentOS Linux 7 (Core)   5.4.274-1.el7.elrepo.x86_64   containerd://1.6.33
k8s-node-01     Ready    <none>          3d19h   v1.30.14   192.168.10.82   <none>        CentOS Linux 7 (Core)   5.4.274-1.el7.elrepo.x86_64   containerd://1.6.33
k8s-node-02     Ready    <none>          3d19h   v1.30.14   192.168.10.83   <none>        CentOS Linux 7 (Core)   5.4.274-1.el7.elrepo.x86_64   containerd://1.6.33
[root@k8s-master-01 ~]# 
[root@k8s-master-01 ~]# curl http://192.168.10.82:38080/index.html
333
~~~



~~~alert type=note
在老版本 k8s 中，nodePort 的端口你直接在所有物理节点用netstat -an就能看到，即占用的就是物理机的端口，但是新版 k8s 则看不到了，并且仍可以使用（不再占用物理机端口，这是一个极大的优化，让你可以放心大胆的使用nodePort 模式，不必再像过去那样唯唯诺诺生怕 nodePort 指定的端口与物理机某个端口冲突）
~~~



#### 9. 快速创建 service 配置模板

使用命令创建

~~~bash
kubectl expose deployment nginx-deployment \
--port=8080 \
--target-port=80 \
--protocol=TCP \
--type=NodePort \
--dry-run=client -o yaml > demo-service.yaml
~~~

参数解释：

- expose 指定要暴露的是一个 deployment，后面跟上 deployment 的名字
- `--port` 指定集群内部访问的端口
- `--target-port` 指定容器服务的端口
- `--type` 指定 service 的类型



## 自我修复

pod 会发生故障，不同类型的故障有不同的修复机制。



### 故障类型

- 某个 pod 挂掉了，导致副本数量比预期少，触发协调机制，使用触发器重启 pod
- pod 数量没有变化，但是某个 pod 中的容器挂掉了。触发 pod 重启机制。
- 副本数量没有变化，pod 内的容器也没有挂掉，但是以为服务的代码存在 bug，导致无法对外提供服务。触发 pod 健康检查机制。检查机制分两类：一类是重启 Pod；另一类是从 svc 中把出问题 pod 代理的配置清除。





### Pod 重启机制

重启策略主要分为以下三种：

- Always：当容器终止退出后，总是重启容器，默认策略
- OnFailure：当容器异常退出（退出状态码非0）时，才重启
- Never：当容器终止退出，从不重启容器

这些重启策略可以在Pod的spec中通过`restartPolicy`字段进行配置。



裸 pod 和结合 deployment 使用的 pod 在触发重启机制时有一些不一样。

- 裸 Pod，可以设置三种重启策略。
- 结合 Deployment 使用的 pod，默认重启机制是 Always， 并且只能设置为 Always



~~~alert type=note
使用 kubectl get pod -o wide 查看 pod 在哪个节点上 <br>
使用 crictl 查看节点上 pod 所在的 container id <br>
使用 crictl inspect 07fe2065703a4 | grep -i pid 查看容器内一号进程的pid号 <br>
使用 kill -9 <pid> 杀死进程
~~~



### Pod 健康检查

健康检查顾名思义就是检查 Pod 是否健康，怎么来定义健康呢？下述两种情况下服务均无法访问，为不健康状态：

1.  程序内部错误但进程仍在运行。
2. 容器已启动但服务未就绪 。



K8S 中定义了两种检查机制：

- **livenessProbe**：存活检查（存活探针），如果检查失败，将杀死容器，根据 Pod 的 restartPolicy 来操作。
- **readinessProbe**：就绪检查（就绪探针），如果检查失败，k8s 会把 Pod 从 svc endpoints 中删除，不再代理该 pod。



不论哪种检查机制，具体的检查方式支持三种：

- **http Get** ：HTTP 请求检查，发送 HTTP 请求，返回 200 - 400 范围状态码为成功。
- **exec**：命令执行检查，执行 Shell 命令返回状态码是 0 为成功。
- **tcpSocket**：TCP 连接检查，发起 TCP Socket 建立成功。



#### 就绪检查

使用 `readinessProbe` 指定就绪检查，具体检查方式选择 `exec` 执行命令的方式，只有 pod 就绪了才会被加入到 svc 的负载均衡中。

创建 svc，新建 nginx-service.yaml

~~~yaml
apiVersion: v1
kind: Service
metadata:
  name: health-test
  labels:
    app: health-test
spec:
  selector:
    app: health-test
  ports:
  - port: 8080     
    protocol: TCP 
    targetPort: 80
~~~

创建 svc 服务，并监听 svc 的 endpoints

~~~bash
# 创建 svc
kubectl apply -f nginx-service.yaml

# 监控查看 svc
# 可以发现此时 Endpoints 为空
watch -n 1 kubectl describe svc health-test
~~~



创建结合 deployment 的 pod，文件 nginx.yaml

~~~yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: health-test
  labels:
    app: health-test
spec:
  replicas: 1
  selector:
    matchLabels:
      app: health-test
  template:
    metadata:
      labels:
        app: health-test
    spec:
      containers:
      - name: nginx
        image: nginx:1.14
        ports:
        - containerPort: 80
        # 重启策略
        # restartPolicy: Always
        # 就绪检查
        readinessProbe:
          exec:
            command:
            - cat
            - /tmp/healthy1
          initialDelaySeconds: 3  # 初始化检查延迟时间
          periodSeconds: 5         # 隔多少秒检查一次
~~~

创建 deployment

~~~bash'
kubectl apply -f nginx.yaml
~~~

查看 pod，发现 pod 一直处于没有 Ready 的状态，svc 中的 endpoints 也一直是空。这是因为 就绪检查执行的命令没有执行成功（因为 pod 内没有文件 `/tmp/healthy1`，`cat` 肯定是报错的）。

~~~bash
[root@k8s-master-01 ~]# kubectl get pod -w
NAME                           READY   STATUS    RESTARTS   AGE
health-test-5d84cf54bb-kslp9   0/1     Running   0          3s
~~~

手动在 pod 内创建 就绪检查文件。创建文件后，pod Ready了，svc endpoints中有 pod 的 ip 了。

~~~bash
kubectl exec health-test-5d84cf54bb-kslp9 -- sh -c "touch /tmp/healthy1"
~~~



#### 存活检查

使用 `livenessProbe` 指定存活检查，具体检查方式选择 `exec` 执行命令的方式，只有 pod 没有存活检查失败就会被重启。

创建结合 deployment 的 pod，文件 nginx.yaml

~~~yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: health-test
  labels:
    app: health-test
spec:
  replicas: 1
  selector:
    matchLabels:
      app: health-test
  template:
    metadata:
      labels:
        app: health-test
    spec:
      containers:
      - name: nginx
        image: nginx:1.14
        ports:
        - containerPort: 80
        # 重启策略
        # restartPolicy: Always
        # 存活检查
        livenessProbe:
          exec:
            command:
            - cat
            - /tmp/healthy2
          initialDelaySeconds: 3  # 初始化检查延迟时间，一般设置比就绪检查的该时间大一点
          periodSeconds: 5         # 隔多少秒检查一次
~~~

创建 deployment

~~~bash'
kubectl apply -f nginx.yaml
~~~

查看 pod，发现 pod默认是 Ready 的，svc 是负载 pod 的。但是，pod 会一直重启（pod 内没有存活检查文件 `/etc/healthy2`），达到重启最大次数后，进入 `CrashLoopBackOff` 状态，此时 svc 中也不再代理该 pod 了。

~~~bash
kubectl get pod po -w
~~~

手动删除旧 pod，再新建 pod，在存活检查期间为 pod 创建存活检查文件，创建后 pod 不再重启。

~~~bash
kubectl delete -f nginx.yaml
kubectl apply -f nginx.yaml
kubectl exec health-test-6f58fd6f4f-wckfh -- sh -c "touch /tmp/healthy2"
~~~



## 自动上线和回滚

### 升级上线

一般是更新镜像版本，因为镜像版本变化意味着代码发生了变化。

有两种方式

- 编辑 yaml 文件，修改镜像版本
- 使用命令，`kubectl set image`

~~~bash
kubectl set image deployment myapp myapp=my-nginx:v2
~~~

这种升级更新方式称之为滚动发布，由 deployment 管理创建 replicaset，replicaset 负责 pod 副本创建和删除。

滚动发布过程中，deployment 会保留旧的 replicaset，然后新建一个新的 replicaset。

- 旧的 replicaset持续移除旧版本的 pod 副本，新的 replicaset负责持续创建新版本 pod 的多副本，移除一个旧 pod，新增一个新 pod。
- 滚动发布完成后，不会自动移除旧的 replicaset，因为可以留作回滚时使用。



### 回滚

回滚过程也是滚动发布的形式。

使用命令回滚操作，如果没有指定回滚版本，则回滚到上一个版本。

~~~bash
kubectl rollout undo deployment nginx nginx-app
~~~

