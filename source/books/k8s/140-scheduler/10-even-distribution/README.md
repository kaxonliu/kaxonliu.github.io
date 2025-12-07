# 均匀调度

均匀调度的含义就是让同一标签的 pod 副本均匀分散到不同的拓扑域中。

引入一种判定机制，可以用标签将每个域中包含的带有此标签的 pod 给筛选出来，这便得到了每个域中带有指定标签的 pod 的分布情况。然后以此作为判定依据，来决定新 pod 应该创建到哪个域中，实现均匀分布。

均匀分布是实现容灾和高可用的核心，将业务 Pod 尽可能均匀的分布在不同可用拓扑域中是非常重要的。

使用均匀调度的前提是 k8s 集群开启了 Even Pod Spreading 模式。从 **Kubernetes v1.25** 开始，Even Pod Spreading 的评分机制默认开启，但为了确保预期的分布行为，生产环境仍应**显式配置拓扑分布约束**。

#### 实现均匀调度的步骤
- 创建出不同的域，即给节点打标签。
- 为新 pod 声明均匀调度相关字段
  - topologyKey 指定拓扑域的维度
  - labelSelector 用来帅选 pod 的标签
  - maxSkew 设置最大 skew 值
  - whenUnsatisfiable 指定硬策略/软策略
- 计算出新 pod 应该调度到哪个域中才均匀
  - 先基于：topologyKey拓扑域+labelSelector标签选择器，来统计出每个域中包含的带有指定标签的pod数。
  - 然后计算出每个域的skew值。当前拓扑域的skew = 当前拓扑域中被标签选出的 Pod 个数 - min{所有拓扑域中被标签选中的 Pod 个数的最小值}。
  - 最后，判读每个域的skew值是否满足 skew<=maxSkew，满足条件则可以在假定的域中创建新Pod，否则需要换另一个域，然后重新计算和判断。
  - 补充：maxSkew值越小，越均匀。

~~~bash
# 如果有多条均匀调度策略，相冲突，解决冲突的方法，是设置软硬策略
whenUnsatisfiable 的值有种设置
- DoNotSchedule   # 默认值。告诉调度器不要调度该 Pod，因此也可以叫作硬策略；
- ScheduleAnyway  # 告诉调度器根据每个 Node 的 skew 值打分排序后仍然调度，因此也可以叫作软策略。
~~~



## Pod 拓扑分布约束

这是最直接的均匀调度方法，可以控制 Pod 在不同拓扑域（如节点、可用区等）的分布。

先给节点打标签

~~~bash
kubectl label node k8s-master-01 zone=zone1
kubectl label node k8s-node-01 zone=zone1
kubectl label node k8s-node-02 zone=zone2
~~~

拓扑分布约束配置。可以有多个约束配置，如果多个约束配置的计算结果不一致，则必须要有软/硬策略之分，否则就会创建 pod 不成功，一直处于 pending 状态。

~~~yaml
kind: Pod
apiVersion: v1
metadata:
  name: mypod6
  labels:
    foo: pod-demo
spec:
  # 均匀调度配置
  topologySpreadConstraints:
  # 第一个约束配置
  # whenUnsatisfiable 配置的值为 DoNotSchedule，表示为硬策略
  - maxSkew: 1
    topologyKey: zone
    whenUnsatisfiable: DoNotSchedule
    labelSelector:
      matchLabels:
        foo: pod-demo
  # 第二个约束配置
  # whenUnsatisfiable 配置的值为 ScheduleAnyway，表示为软策略
  - maxSkew: 1
    topologyKey: kubernetes.io/hostname
    whenUnsatisfiable: ScheduleAnyway
    labelSelector:
      matchLabels:
        foo: pod-demo
  containers:
  - name: nginx
    image: nginx:latest
~~~

#### 注意点

- 只有与新的 Pod 具有相同命名空间的 Pod 才能作为匹配候选者。
- 物理节点上的标签必须与 topologySpreadConstraints[*].topologyKey 所指定的保持一致，否则该节点将会被均匀调度算法忽略
- 只有被 topologySpreadConstraints[*].labelSelector 选中的 pod 才会参与计算，未被选中的 pod 对计算结果无影响。
如果新 Pod 定义了 spec.nodeSelector 或 spec.affinity.nodeAffinity，他们的优先级更高，会先依据他们来选择或排除一些节点，然后在剩下的节点里进行均匀调度，如果你有多套环境env=prod、env=staging、env=qa时，先定位到某一个环境中，然后再进行均匀调度就非常棒。



示例：同一个 deployment 控制器下管理多个 pod 副本，要求让每个节点上运行它的2个pod副本

~~~yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-topo-demo-pod
spec:
  replicas: 6  # 副本数为6，正好对应三个物理节点，每个节点分2个
  selector:
    matchLabels:
      app: topo-demo
  template:
    metadata:
      labels:
        app: topo-demo
    spec:
      containers:
      - image: nginx:latest
        name: nginx
      # 容忍master节点的污点
      tolerations:
      - operator: "Exists"
        effect: "NoSchedule"

      # 均匀分布策略
      topologySpreadConstraints:
        - maxSkew: 1
          topologyKey: kubernetes.io/hostname # 使用该拓扑域，即每个节点就独立构成一个拓扑域
          whenUnsatisfiable: DoNotSchedule
          labelSelector:
            matchLabels:
              app: topo-demo
~~~



## Descheduler

Descheduler是一个均衡器，用来解决 k8s 运行过程中节点上 pod 分布的不均匀问题。

>均匀分布机制 topologySpreadConstraints，只负责创建时的均匀，不负责运行时。

k8s集群是非常动态的，例如为了停机维护某一个物理节点，先执行驱逐/排空操作，使得该节点上的pod都被驱逐到了其他物理节点上。<font color="red" size="3">但是，当我们停机维护完成后，之前的pod并不会自动回到该节上，因为Pod一旦被绑定了节点是不会自动触发重新调度的，这就会导致集群在一段时间内出现不均衡的状态。</font>

真正的解决这个问题的办法，是引入一个均衡器来重新平衡集群，Descheduler 就是做这个事情的工具。

#### Descheduler的核心原理

Descheduler 本身并不会参与调度，它只负责把应该被驱逐的 pod 计算出来，驱逐后重建新 pod 的调度的任务还是会交给默认的调度器 kube-scheduler 来完成。



#### 安装 deseheduler 

使用 helm 的方式安装 deseheduler。deseheduler 可以以 job、cronjob 或者 deployment 的形式运行子 k8s 集群中，可以以 helm chart 来安装 deseheduler

安装 helm

~~~bash
# 官网 https://github.com/helm/helm/releases

# 下载
wget https://get.helm.sh/helm-v3.15.1-linux-amd64.tar.gz

# 解压
tar xf helm-v3.15.1-linux-amd64.tar.gz

# 拷贝命令
cp linux-amd64/helm  /usr/bin/

# 验证
[root@k8s-master-01 tmp]# helm version
version.BuildInfo{Version:"v3.15.1", GitCommit:"e211f2aa62992bd72586b395de50979e31231829", GitTreeState:"clean", GoVersion:"go1.22.3"}
~~~





安装 deseheduler 

~~~bash
# 添加仓库
helm repo add descheduler https://kubernetes-sigs.github.io/descheduler/
helm repo update
# 查看可用的版本
helm search repo descheduler --versions

# 安装
# 使用默认配置安装
helm upgrade --install descheduler descheduler/descheduler \
  --set podSecurityPolicy.create=false \
  --namespace kube-system
~~~

上述安装命令执行后，看到如下类似输出则表明安装成功

~~~bash
NAME: descheduler
LAST DEPLOYED: Sun Dec  7 22:49:25 2025
NAMESPACE: kube-system
STATUS: deployed
REVISION: 1
TEST SUITE: None
NOTES:
Descheduler installed as a CronJob.
A DeschedulerPolicy has been applied for you. You can view the policy with:

kubectl get configmap -n kube-system descheduler -o yaml

If you wish to define your own policies out of band from this chart, you may define a configmap named descheduler.
To avoid a conflict between helm and your out of band method to deploy the configmap, please set deschedulerPolicy in values.yaml to an empty object as below.

deschedulerPolicy: {}
~~~

安装成功后，可以查看 cronjob 、job、和 pod

~~~bash
[root@k8s-master-01 ~]# kubectl -n kube-system get cronjobs
NAME          SCHEDULE      TIMEZONE   SUSPEND   ACTIVE   LAST SCHEDULE   AGE
descheduler   */2 * * * *   <none>     False     1        10m             10m
[root@k8s-master-01 ~]# kubectl -n kube-system get jobs
NAME                   STATUS    COMPLETIONS   DURATION   AGE
[root@k8s-master-01 ~]# 
[root@k8s-master-01 ~]# 
descheduler-29418650   Running   0/1           10m        10m
[root@k8s-master-01 ~]# 
[root@k8s-master-01 ~]# 
[root@k8s-master-01 ~]# kubectl -n kube-system get pod | grep desc
descheduler-29418650-lkm82              0/1     ImagePullBackOff   0             11m
~~~

发现 Pod 拉取镜像失败，这是因为默认 cronjob 配置中使用的是国外的镜像，导致下载失败。这里需要替换为过内镜像

~~~bash
[root@k8s-master-01 ~]# kubectl -n kube-system get cronjobs.batch descheduler -o yaml | grep -i image
            image: registry.k8s.io/descheduler/descheduler:v0.34.0
            imagePullPolicy: IfNotPresent
~~~

使用 kubectl edit 的方式临时修改为国内镜像

~~~bash
[root@k8s-master-01 ~]# kubectl -n kube-system edit cronjobs.batch descheduler 
cronjob.batch/descheduler edited
[root@k8s-master-01 ~]# 
[root@k8s-master-01 ~]# # 删除历史 Job 和 pod
[root@k8s-master-01 ~]# kubectl -n kube-system delete job descheduler-29418650 
job.batch "descheduler-29418650" deleted
[root@k8s-master-01 ~]# kubectl -n kube-system delete pod descheduler-29418668-xmbcz 
pod "descheduler-29418668-xmbcz" deleted
[root@k8s-master-01 ~]# 
[root@k8s-master-01 ~]#  # 发现新建的 job 和 pod 是正常执行的
[root@k8s-master-01 ~]# kubectl -n kube-system get job
NAME                   STATUS    COMPLETIONS   DURATION   AGE
descheduler-29418668   Running   0/1           19s        19s
[root@k8s-master-01 ~]# 
[root@k8s-master-01 ~]# kubectl -n kube-system get pod | grep desc
descheduler-29418668-zmrlh              0/1     Completed   0             3m7s
~~~

安装时指定镜像

~~~bash
# 安装时指定镜像
helm install descheduler descheduler/descheduler \
  --namespace descheduler-system \
  --set image.repository=registry.k8s.io/descheduler/descheduler \
  --set image.tag=v0.34.0 \
  --set image.pullPolicy=IfNotPresent

# 或者简写形式
helm install descheduler descheduler/descheduler \
  --namespace descheduler-system \
  --set image="registry.k8s.io/descheduler/descheduler:v0.34.0"
~~~



#### descheduler 配置

~~~bash
[root@k8s-master-01 ~/selector]# kubectl -n kube-system get cm descheduler -o yaml
apiVersion: v1
kind: ConfigMap
metadata:
  。。。。。。
data:
  policy.yaml: |
    apiVersion: "descheduler/v1alpha2"
    kind: "DeschedulerPolicy"
    profiles:
    - name: default
      pluginConfig:
      - args:
          evictLocalStoragePods: true
          ignorePvcPods: true
        name: DefaultEvictor
      - name: RemoveDuplicates
      - args:
          includingInitContainers: true
          podRestartThreshold: 100
        name: RemovePodsHavingTooManyRestarts
      - args:
          nodeAffinityType:
          - requiredDuringSchedulingIgnoredDuringExecution
        name: RemovePodsViolatingNodeAffinity
      - name: RemovePodsViolatingNodeTaints
      - name: RemovePodsViolatingInterPodAntiAffinity
      - name: RemovePodsViolatingTopologySpreadConstraint
      - args:
          targetThresholds:
            cpu: 50
            memory: 50
            pods: 50
          thresholds:
            cpu: 20
            memory: 20
            pods: 20
        name: LowNodeUtilization
      # 启用的相关插件/策略
      plugins:
        balance:
          enabled:
          # 删除调度到同一节点上的相同 Pod（通常指具有相同 Pod Template Hash 的 Pod），以避免不必要的冗余和资源浪费。
          - RemoveDuplicates
          # 移除违反拓扑分布约束（Topology Spread Constraints）的 Pod，以确保 Pod 在集群中的分布更加均匀。
          - RemovePodsViolatingTopologySpreadConstraint
          # 将工作负载从资源利用率低的节点迁移到其他节点，以释放资源或关闭低利用率节点。
          - LowNodeUtilization
        deschedule:
          enabled:
          # 移除因频繁重启而可能存在问题的 Pod。
          - RemovePodsHavingTooManyRestarts
          # 移除违反节点污点（Node Taints）规则的 Pod。
          - RemovePodsViolatingNodeTaints
          # 移除违反节点亲和性（Node Affinity）规则的 Pod。
          - RemovePodsViolatingNodeAffinity
          # 移除违反 Pod 间反亲和性（Inter-Pod Anti-Affinity）规则的 Pod。
          - RemovePodsViolatingInterPodAntiAffinity
~~~



示例：模拟 descheduler 工作

- 第一步先部署

~~~yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: descheduler-demo-pod
spec:
  replicas: 6 
  selector:
    matchLabels:
      app: descheduler-demo
  template:
    metadata:
      labels:
        app: descheduler-demo
    spec:
      tolerations:
      - key: "node-role.kubernetes.io/control-plane" # 容忍master节点的污点,允许调度到master上
        operator: "Exists"
        effect: "NoSchedule"

      - key: "node.kubernetes.io/unreachable" # 节点挂掉后，k8s默认为集群打上该污点
        operator: "Exists"
        effect: "NoExecute"
        tolerationSeconds: 10 # 我们自己设置容忍多久后驱逐，该参数默认300s，替代了K8S旧版本的--pod-eviction-timeout（controller-manager的启动参数）
      
      
      containers:
      - image: nginx:latest
        name: nginx
~~~

- 模拟某个节点故障，驱逐其上的pod ，比如模拟 k8s-node-02 挂掉，在 node02机器上执行`systemctl stop kubelet`
- 过一会后发现node02上的pod被驱逐
- 模拟node02恢复，在 node02 机器上执行 `systemctl start kubelet `



## 添加 PDB 策略防止单点

引入descheduler之后，它会根据策略对 pod 进行驱逐以便进行重新调度，使 k8s 的资源达到一个平衡状态。

但是会有问题

1、如果一个服务只有一个副本，你为了资源平衡，把它驱逐走了，虽然会在新节点上拉起来，但过程中会导致业务中断

```
针对该情况，首先建议做成多副本来避免单点故障，然后尽量打撒，可以考虑用反亲和，但即便如此，也并不能完全解决问题，见2
```

2、如果你有多个副本，也打散在多个节点上，但是这只是你的副本均匀，并不代表节点的资源分配是均匀的，当 descheduler 判定节点资源不均衡时，就会触发 pod 驱逐，也就是说你的 pod 副本是有可能被全都驱逐的，同样会导致业务中断。

针对上述 pod 副本被全部驱逐导致业务中断的问题，我们可以通过配置`PDB（PodDisruptionBudget）` 对象来避免所有副本同时被删除，比如我们可以设置在驱逐的时候某应用最多只有一个副本不可用。

则创建如下所示的资源清单即可，用标签选中你要保护的 pod

```yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: pdb-demo
  namespace: default
spec:
  maxUnavailable: 1 # 设置最多不可用的副本数量，或者使用 minAvailable，可以使用整数或百分比
  selector:
    matchLabels: # 匹配Pod标签------------》针对被选中的pod，最大不可用副本为1
      app: demo
```

关于 PDB 的更多详细信息可以查看官方文档：https://kubernetes.io/docs/tasks/run-application/configure-pdb/。

所以如果我们使用 `descheduler` 来重新平衡集群状态，那么我们强烈建议给应用创建一个对应的 `PodDisruptionBudget` 对象进行保护。



## 注意事项

当使用 descheduler 驱除 Pods 的时候，需要注意以下几点：
- 关键性 Pod 不会被驱逐，比如 `priorityClassName` 设置为 `system-cluster-critical` 或 `system-node-critical` 的 Pod
```
  [root@k8s-master-01 ~]# kubectl  -n kube-system get cronjobs.batch descheduler -o yaml |grep -i priority
            priorityClassName: system-cluster-critical
```
- 不属于 RS、Deployment 或 Job 管理的 Pods 不会被驱逐
- DaemonSet 创建的 Pods 不会被驱逐
- 使用 `LocalStorage` 的 Pod 不会被驱逐，除非设置 `evictLocalStoragePods: true`
- 具有 PVC 的 Pods 不会被驱逐，除非设置 `ignorePvcPods: true`
- 在 `LowNodeUtilization` 和 `RemovePodsViolatingInterPodAntiAffinity` 策略下，Pods 按优先级从低到高进行驱逐，如果优先级相同，`Besteffort` 类型的 Pod 要先于 `Burstable` 和 `Guaranteed` 类型被驱逐
```
  默认启用的策略
      RemoveDuplicates
  	RemovePodsViolatingInterPodAntiAffinity
      LowNodeUtilization
      
      RemovePodsHavingTooManyRestarts
  	RemovePodsViolatingNodeTaints
  	RemovePodsViolatingNodeAffinity
  	RemovePodsViolatingTopologySpreadConstraint
```
- `annotations` 中带有 `descheduler.alpha.kubernetes.io/evict` 字段的 Pod 都可以被驱逐，该注释用于覆盖阻止驱逐的检查，用户可以选择驱逐哪个 Pods
- 如果 Pods 驱逐失败，可以设置 `--v=4` 从 `descheduler` 日志中查找原因，如果驱逐违反 PDB 约束，则不会驱逐这类 Pods
