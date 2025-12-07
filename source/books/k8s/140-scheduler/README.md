# 调度策略

Kubernetes 调度策略是一个核心功能，它决定了 Pod 如何被分配到合适的节点上运行。调度器（kube-scheduler）负责根据多种策略和约束条件做出决策。

调度指的就是创建新 pod 的时候，应该分配给哪个节点来创建该 pod，由调度器负责做出决策，具体负责创建 pod 的是该节点上的kubelet。

~~~alert type=note
何时创建新: pod<br>
1. 使用者提交一个全新的创建pod的请求。<br>
2. 某个节点挂掉、资源不足了，导致该节点上pod被驱逐到其他节点。<br>
3. pod是被控制器管理者，当副本数与预期不符，引发调谐过程也会发起创建新pod的请求。 <br>

注意: <br>
静态pod不参与pod的调度流程，静态固定在某个节点上。
~~~

## 调度流程

调度过程分为两个阶段：

1. 预选：根据资源申请，选择器、亲和与反亲和、污点与容忍等规则，先筛选出一些机器，选出符合基本条件的节点。
2. 优选：对预选出来的节点打分，选择最优节点。

具体的调度控制方式常见的有：节点选择器、节点亲和、Pod 亲和/反亲和、污点与容忍等规则。



## 节点选择 NodeSelector

事先给目标节点打上标签（依据该节点的特性），依据节点的标签，把 pod 调度到某个节点。

~~~yaml
# 把 pod 调度到带有具有标签 disktype=ssd 的节点上
apiVersion: v1
kind: Pod
metadata:
  name: nginx
spec:
  nodeSelector:
    disktype: ssd
  containers:
  - name: test-pod
    image: nginx
~~~

可以同时指定多个节点标签选择要求，必须同时满足条件的节点才会被调度

~~~yaml
nodeSelector:
  disktype: ssd      # 节点必须有标签 disktype=ssd
  gpu: "true"        # 节点必须有标签 gpu=true

# Pod 必须被调度到同时满足以下两个条件的节点
# 1.节点拥有标签 disktype，且其值为 ssd
# 2.节点拥有标签 gpu，且其值为 true
~~~

给节点打标签

~~~bash
# 给节点打标签
kubectl label node k8s-node-01 disktype=ssd
# 删除节点标签
kubectl label node k8s-node-02 disktype-
~~~

注意：

- 如果我们给多个Node都定义了相同的标签（disk=ssd），则kube-scheduler会根据调度算法从这组Node中挑选一个可用的Node
- NodeSelector一定会调度到包含执行标签的节点，没有这种节点则调度失败，pod无法创建



## 亲和/反亲和

亲和指的是把一些彼此依赖的 pod 需要调度到同一个"地方"。反亲和指的是把一些 pod 调到不同的 "地方。

亲和性与反亲和性，有两种维度：

- 节点维度：标签选择器定位的是节点的标签
- pod维度：标签选择器定位的是pod的标签，而不是节点

无论何种维度的亲和性与反亲和性，都对应有两种具体的策略：

- `硬策略`：是 Pod 调度时必须满足的规则，否则 Pod 对象的状态会一直是 Pending。
- `软策略`：在 Pod 调度时可以尽量满足其规则，在无法满足规则时，可以调度到一个不匹配规则的节点之上，并且可以设置权重。

~~~bash
# 软策略名为:preferredDuringSchedulingIgnoredDuringExecution
# 硬策略名为:requiredDuringSchedulingIgnoredDuringExecution

需要注意的是软硬策略名字中后半段字符串`IgnoredDuringExecution`表示的是，在Pod资源基于节点亲和性规则调度到某个节点之后，如果节点的标签发生了改变，该pod不再符合该节点的亲和性要求，调度器不会将Pod从该节点上移除，因为该规则仅对新建的Pod对象有效。
~~~



## 节点亲和

nodeAffinity 是用于替换 nodeSelector 的全新调度策略，硬策略与软策略可以一起用，也可以单独用。

给节点打标签

~~~bash
kubectl label nodes k8s-node-01 disktype=gpu
kubectl label nodes k8s-node-01 gpu=true
kubectl label nodes k8s-node-02 disktype=gpu
~~~

节点亲和示例如下（节点亲和本质也是按照节点的标签，调度 pod 到满足规则的节点上）

~~~yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: node-affinity
  labels:
    app: node-affinity
spec:
  replicas: 8 # 副本数多一点，以便验证资源不充足的情况下，软限制的特点
  selector:
    matchLabels:
      app: node-affinity
  template:
    metadata:
      labels:
        app: node-affinity
    spec:
      containers:
      - name: nginx
        image: nginx:latest
      
      # 亲和
      affinity:
        # 节点亲和
        nodeAffinity:
          # 硬亲和
          requiredDuringSchedulingIgnoredDuringExecution:
            nodeSelectorTerms:
            - matchExpressions:
              - key: disktype
                operator: In
                values:
                - ssd
          # 软亲和
          preferredDuringSchedulingIgnoredDuringExecution:
          - weight: 60
            preference:
              matchExpressions:
              - key: gpu
                operator: In
                values:
                - "true"
~~~

部署上述 yaml。调度规则解释如下：

- 硬策略规定必须调度到 disktype=ssd 的节点上，如果没有满足的节点则 Pod 状态处于 Pending 状态
- 软策略要求尽量调度到 `gpu=true` 的节点（权重60），如果没有满足的节点，也会被调度



#### NodeAffinity 语法支持的操作符

- In：label 的值在某个列表中 
- NotIn：label 的值不在某个列表中
- Gt：label 的值大于某个值
- Lt：label 的值小于某个值
- Exists：某个 label 存在
- DoesNotExist：某个 label 不存在



#### NodeAffinity 节点亲和的规则设置注意事项

- 如果同时设置了节点选择器 nodeSeletor 与 nodeAffinity，那么必须两个条件都得到满足才行。
- 如果 nodeAffinity 中设置的多个 nodeSelectorTerms 满足任一即可。
- 如果 nodeSelectorTerms 中设置的多个 matchExpression 是逻辑与的关系，必须同时满足才行。
- 节点维度本身并没有反亲和这种排斥功能，但是用 NotIn、DoesNotExist 就可以实现排斥的功能。



matchExpression 也支持使用如下 json 的配置定义规则

~~~yaml
apiVersion: v1
kind: Pod
metadata:
  name: hard-affinity-pod
spec:
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
        - matchExpressions: 
          - {key: disktype, operator: In, values: ["ssd"]}
  containers:
  - name: test-pod
    image: nginx
~~~



## pod 亲和

所谓 pod 亲和指的是，新创建的 pod 在调度时选择和已经存在的 Pod 亲和，即靠近这个 pod，也就是调度到这个 pod 所在的节点。

选择标准是 pod 的标签，即根据**其他 Pod 的标签**来调度 Pod，而不是节点标签。

~~~yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-affinity-hard
  labels:
    app: web
spec:
  containers:
  - name: test-pod
    image: nginx
  affinity:
    # pod 亲和
    podAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
      - labelSelector:
          matchExpressions:
          - key: security
            operator: In
            values:
            - S1
        topologyKey: kubernetes.io/hostname # 指定拓扑域
        namespaces:		# 指定名称空间
~~~

解释如下：

- 使用的是硬策略，如果找不到满足条件的 Pod，则调度失败，创建 pod 处于 Pending 状态。
- labelSelector.matchExpressions 匹配的是 pod 的标签，不是节点的标签。
- 与Label Selector 同级，还可以设置 namespaces 来指定去跨名称空间查找 pod 来进行亲和，如果 namespaces 省略、或者被设置为空 null 或空列表，那么 Pod Affinity 规则将仅应用于与当前 Pod 在同一命名空间中的 Pods。
- 同理，有硬亲和度即有软亲和度，Pod 也支持使用 preferredDuringSchedulingIgnoredDuringExecuttion 属性进行定义 Pod 的软亲和性，调度器会尽力满足亲和约束的调度，在满足不了约束条件时，也允许将该 Pod 调度到其他节点上运行。



## pod 反亲和

pod 反亲和性则是反着来的，比如一个节点上运行了某个 pod，那么我们的 pod 则希望被调度到其他节点上去，同样我们把上面的 podAffinity 直接改成 podAntiAffinity

~~~yaml
# 确保同一节点不运行相同应用
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-server
spec:
  replicas: 3
  selector:
    matchLabels:
      app: web
  template:
    metadata:
      labels:
        app: web
    spec:
      containers:
      - name: web
        image: nginx
      # 亲和
      affinity:
        # pod 反亲和
        podAntiAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
          - labelSelector:
              matchExpressions:
              - key: app
                operator: In
                values:
                - web
            topologyKey: kubernetes.io/hostname  # 节点级别隔离
~~~



## 拓扑域

我们把 Node 物理节点归属的区域称之为拓扑域，而 Node 物理节点归属的区域通常按照物理服务器的摆放位置来进行划分。

从最小的维度看，单个Node就独自构成一个拓扑域，该拓扑域里只有它自己这台机器，比如 k8s 种默认每个节点都有一个标签 kubernetes.io/hostname 标识主机名。

~~~bash
kubernetes.io/hostname=k8s-node-01
~~~

范围扩大，同一个机架上摆放的n个Node也是一个拓扑域

范围继续扩大，一个机房里摆放的n个Node是一个更大的拓扑域

范围继续扩大，一个地域（Zone）中摆放的n个Node是一个更更大的拓扑域，例如华北地区、华南地区

#### Kubernetes 自动添加的标签

```bash
# 查看节点上的内置拓扑标签
kubectl get nodes --show-labels | grep -E "kubernetes.io|topology"
```

| 拓扑键                               | 描述         | 示例值                     |
| :----------------------------------- | :----------- | :------------------------- |
| `kubernetes.io/hostname`             | 节点主机名   | `node-1`, `node-2`         |
| `topology.kubernetes.io/zone`        | 可用区（AZ） | `us-west-2a`, `us-east-1c` |
| `topology.kubernetes.io/region`      | 区域         | `us-west-2`, `us-east-1`   |
| `beta.kubernetes.io/zone` (已弃用)   | 可用区       |                            |
| `beta.kubernetes.io/region` (已弃用) | 区域         |                            |

为何要用拓扑域：

- 通过这种方式，我们就可以将各个 Pod 进行跨集群、跨机房、跨地区的调度了。

如何标识机器归属的拓扑域：

- 打标签就行，比如1～5节点打上zone=A标签，6～10节点有zone=B标签。

创建新pod时如何指定拓扑域

- 通过 topologyKey 来指定， topology 顾名思义就是 拓扑 / 拓扑域 的意思 topologyKey 对应的值是 node 上的一个标签名称，该名称对应的值相同的节点就属于同一个拓扑域 如果1～5节点打上zone=A标签，6～10节点有zone=B标签 那么pod affinity topologyKey定义为zone时，那么1～5节点属于一个位置，6～10属于另外一个位置。



#### 示例

需求：当前有两个机房（ beijing，shanghai），需要部署一个nginx产品，副本为两个，为了保证机房 容灾高可用场景，需要在两个机房分别部署一个副本。

前提：

- beijing机房的物理节点已经为node打上标签zone=beijing
- shanghai机房的物理节点已经为node打上标签zone=shanghai

示例

~~~yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: nginx-affinity-test
spec:
  serviceName: nginx-service
  replicas: 2
  selector:
    matchLabels:
      service: nginx
  template:
    metadata:
      name: nginx
      labels:
        service: nginx
    spec:
      affinity:
        # 下面的pod反亲和策略总结一句话就是：新pod不能调度到--->带有标签service=nginx的pod所在的zone
        # 创建第一个pod时，service=nginx标签的pod是全新的，尚未存在于任何一个拓扑域中，所以可以调度
        # 创建第二个pod时，service=nginx标签的pod已经存在，已存在于某一个拓扑域中，所以第二个pod排斥
        podAntiAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
          - labelSelector:
              matchExpressions:
              - key: service
                operator: In
                values:
                - nginx
            topologyKey: zone		# 使用拓扑域具体就是指定节点标签的key
      containers:
      - name: test
        image: nginx:latest
~~~

应用

~~~bash
[root@k8s-master-01 test4]# kubectl label node k8s-node-01 zone=shanghai
node/k8s-node-01 labeled
[root@k8s-master-01 test4]# kubectl label node k8s-node-02 zone=beijing
node/k8s-node-02 labeled
[root@k8s-master-01 test4]# 
[root@k8s-master-01 test4]# kubectl apply -f 5.yaml 
statefulset.apps/nginx-affinity-test created
[root@k8s-master-01 test4]# kubectl get pods -o wide
NAME                    READY   STATUS    RESTARTS   AGE   IP             NODE          NOMINATED NODE   READINESS GATES
nginx-affinity-test-0   1/1     Running   0          6s    10.244.1.40    k8s-node-01   <none>           <none>
nginx-affinity-test-1   1/1     Running   0          5s    10.244.2.152   k8s-node-02   <none>           <none>
~~~



## 污点与容忍

什么是污点（taints）

- 对于 nodeAffinity 无论是硬策略还是软策略方式，都是调度 pod 到预期节点上，而 Taints污点 恰好 与之相反，如果一个节点标记为 Taints ，除非 pod 也被标识为可以容忍污点节点，否则该 Taints 节点 不会被调度 pod。 

什么是容忍（tolerations）

- 污点（taints）是定义在节点上的一组键值型属性数据，用来让节点拒绝将 Pod 调度到该节点上，除非该 Pod 对象具有容纳节点污点的容忍度。而容忍度（tolerations）是定义在 Pod 对象上的键值型数据，用来配置让 Pod 对象可以容忍节点的污点。



污点与容忍的应用场景

- 比如用户希望把 Master 节点保留给 Kubernetes 系统组件使用，或者把一组具有特殊硬件设备的节点 预留给某些 pod，或者让一部分节点专门给一些特定应用使用/独占，则污点就很有用了，pod 不会再被 调度到 taint 标记过的节点。

我们使用 kubeadm 搭建的集群默认就给 master 节点添加了一个污点标记，所以我们平时的 pod 都没有被调度到 master 上去。

~~~bash
[root@k8s-master-01 test4]# kubectl describe node k8s-master-01  | grep -i taint
Taints:             node-role.kubernetes.io/control-plane:NoSchedule
~~~

#### 为节点打上污点

污点是定义是在节点上的，而容忍度的定义是在Pod中的podSpec，都属于键值型数据，两种方式都支 持一个 effect 标记，语法格式为 key=value: effect 

~~~bash
# 语法：
kubectl taint nodes node1 key=value: effect

# 示例
# 该污点Taint的键为key，值为value
# 该Taint的效果是NoSchedule
kubectl taint nodes node1 key=value1:NoSchedule 

# 注意：
kubectl taint nodes node1 key=:NoSchedule # value可以省略，代表空值
~~~

其中 effect 是用来定义对Pod对象的排斥等级，主要包含以下3种类型： 

- NoSchedule：一定不被调度。属于强制约束，节点现存的Pod对象不受影响。 
- PreferNoSchedule：NoSchedule 的软策略版本，表示尽量不调度到污点节点上去 。
- NoExecute：该选项意味着一旦 Taint 生效，如该节点内正在运行的 pod 没有对应 Tolerate 设 置，会直接被逐出 ，属于强制约束。



#### 节点去掉污点

~~~bash
# 删除污点语法：
kubectl taint nodes <node-name> <key>[: <effect>]-

#删除node01上的node-type标识为NoSchedule的污点
$ kubectl taint nodes k8s-node01 node-type:NoSchedule-

#删除指定键名的所有污点
$ kubectl taint nodes k8s-node01 node-type-

#补丁方式删除节点上的全部污点信息
$ kubectl patch nodes k8s-node01 -p '{"spec":{"taints":[]}}
~~~



#### 为 pod 定义容忍

在Pod对象上定义容忍度时，其支持2种operator操作符： Equal 和 Exists

- Equal：等值比较，表示容忍度和污点必须在 key、value、effect 三者之上完全匹配
- Exists：存在性判断，表示二者的 key 和 effect 必须完全匹配，而容忍度中的 value 字段使用空值
- 如果不指定 operator 属性，则默认值为 Equal

另外，还有两个特殊值

- 空的 key 如果再配合 Exists 就能匹配所有的 key 与 value，也是是能容忍所有 node 的所有 Taints
- 空的 effect 匹配所有的 effect



**示例**

如果仍 然希望某个 pod 调度到 taint 节点上，则必须在 Spec 中做出 Toleration 定义，才能调度到该节点，比如现在我们想要将一个 pod 调度到 master 节点

先查看 master 节点的污点策略，发现 只有 key 和 effect 

~~~bash
[root@k8s-master-01 test4]# kubectl describe node k8s-master-01  | grep -i taint
Taints:             node-role.kubernetes.io/control-plane:NoSchedule
~~~

定义 pod 的容忍度

~~~yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: taint-demo
  labels:
    app: taint-demo
spec:
  replicas: 6
  selector:
    matchLabels:
      app: taint-demo
  template:
    metadata:
      labels:
        app: taint-demo
    spec:
      containers:
      - name: nginx
        image: nginx:latest
      # 定义容忍
      tolerations:
      - key: "node-role.kubernetes.io/control-plane"
        operator: "Exists"
        effect: "NoSchedule"
~~~



#### 定义 Pod 驱逐时间

针对 NoExecute 可以指定 tolerationSeconds

NoExecute 这个 Taint 的效果会对节点上正在运行的 pod 有以下影响：

- 没有设置 Toleration 容忍的 Pod 会被立即驱逐。
- 配置了相应的 Toleration 的 Pod，没有配置 tolerationSeconds 赋值的 pod，则会一直保留在该节点上。
- 配置了相应的 Toleration 的 Pod，并且为 tolerationSeconds 赋了值的 Pod，则会在指定的时间后被驱逐。

~~~alert type=note
注意如果你使用了tolerationSeconds，那么effect必须为NoExecute
~~~



测试：比如把 k8s-node-02 节点设置污点 NoExecute 

~~~bash
apiVersion: apps/v1
kind: Deployment
metadata:
  name: taint-demo2
  labels:
    app: taint-demo2
spec:
  replicas: 6
  selector:
    matchLabels:
      app: taint-demo2
  template:
    metadata:
      labels:
        app: taint-demo2
    spec:
      containers:
      - name: nginx
        image: nginx:latest
      # 定义容忍
      tolerations:
      - key: "test"
        operator: "Exists"
        effect: "NoExecute"
        tolerationSeconds: 30
~~~

应用后，发现有部分 pod 调度到 k8s-node-02 节点上

然后给 k8s-node-02 节点打上污点 NoExecute，观察 pod 的调度变化。

~~~bash
kubectl taint nodes k8s-node-02 test:NoExecute
~~~



####  k8s 会自动为 Pod 添加 Toleration

k8s会自动为Pod添加下面几种Toleration

- key 为 node.kubernets.io/not-ready，并配置 tolerationSeconds=300;
- key 为 node.kubernets.io/unreachable，并配置 tolerationSeconds=300

以上添加的这种自动机制保证了在某些节点发生 一些临时性问题时，Pod默认能够继续留在当前节点运 行5min等待节点恢复，而不是立即被驱逐，从而避免系统的异常波动。

上述的默认的 tolerationSeconds 时间设置可以修改 apiserver 启动时默认配置

~~~bash
$ vim /etc/kubernetes/manifests/kube-apiserver.yaml
spec:
  containers:
  - command:
    - kube-apiserver
    - --default-not-ready-toleration-seconds=30
    - --default-unreachable-toleration-seconds=30
~~~



## 练习:使用Deployment实现Daemonset一样的效果

Deployment+污点容忍+pod反亲和：实现每个节点各分配一个pod

~~~yaml
# ds-demo.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: ds-demo
spec:
  replicas: 3
  selector:
    matchLabels:
      app: ds-demo
  template:
    metadata:
      labels:
        app: ds-demo
    spec:
      containers:
      - name: nginx
        image: nginx:latest
      affinity:
        podAntiAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
          - labelSelector:
              matchExpressions:
              - key: app
                operator: In
                values:
                - ds-demo
            topologyKey: kubernetes.io/hostname
      tolerations:
      - key: "node-role.kubernetes.io/control-plane"
        operator: "Exists"
        effect: "NoSchedule"
~~~

