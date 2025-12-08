# pod 驱逐

Pod 驱逐（Pod Eviction）是指将 Pod 从当前运行的节点上移除的过程。这个过程可能会导致 Pod 被终 止并重新调度到其他节点。 存在以下两种情况下会发生pod的驱逐 

- 节点不可用。 Kube-controller-manager 会周期性检查所有节点状态，当节点处于 NotReady 状态超过一段时间后，驱逐该节点上所有 pod。
- 节点资源压力（不可压缩资源内存、磁盘），由 kubelet 负责驱逐该节点上的 pod。Kubelet 周期性检查本节点资源，当资源不足时，按照pod的Qos优先级驱逐部分 pod。



## Kube-controller-manger 发起的驱逐

Kube-controller-manager （包含的 node controller 控制器代码）周期性检查节点资源的状态信息，从检查节点状态并判断为 NotReady 异常，到完成工作负载 pod 的驱逐总周期默认约等于为6分钟。

kubernetes节点失效后pod的调度/驱逐过程： 

- 1、前提：kubelet 会将自己节点的状态信息（节点健康信息、资源使用情况等）定期更新状态到 apiserver，通过参数 –node-status-update-frequency 指定上报频率，默认是 10s 上报一次。 
- 2、kube-controller-manager 中的节点控制器 会每隔 `–node-monitor-period` 时间去检查 kubelet 汇报上来的节点状态，默认是 5s。 
- 3、当 node 失联一段时间后，默认通过 `–node-monitor-grace-period` 参数配置默认值为40s， kubernetes 判定 node 为 notready 状态。 
- 4、当node失联后一段时间后，kubernetes 开始删除原 node 上的 pod，这段时长配置项为 `pod-eviction-timeout `，默认5m0s，即300s，这5分钟包含判定失联的1min，即在判定失联后再过 4min即可以驱逐。



强调，在 Kubernetes v1.27 版本中，`--pod-eviction-timeout  `参数被标记为已弃用，取而代之的是节点挂掉后 k8s 为节点添加`NoExecute ` 的污点，并且在此之前默认为每个 pod 都添加了容忍且设置了多久后才会驱逐。

~~~yaml
apiVersion: v1
kind: Pod
metadata:
  name: example-pod
spec:
  tolerations:
  # 容忍节点不可达污点，100秒后驱逐（默认300秒）
  - key: "node.kubernetes.io/unreachable"
    operator: "Exists"
    effect: "NoExecute"
    tolerationSeconds: 100
  
  # 容忍节点未就绪污点，100秒后驱逐（默认300秒）
  - key: "node.kubernetes.io/not-ready"
    operator: "Exists"
    effect: "NoExecute"
    tolerationSeconds: 100
~~~

默认事件就是 300s，如果你想修改改默认值，修改 apiserver 的配置

~~~bash
# Kubernetes 为 Pod 自动添加的针对 unreachable / not-ready 
# 污点的容忍时长由 APIServer中的相应参数控制，如需修改请逐台在所有 Master 节点上进行如下操作
$ vim /etc/kubernetes/manifests/kube-apiserver.yaml
spec:
containers:
- command:
- kube-apiserver
- --default-not-ready-toleration-seconds=100    #对该污点的容忍时长设置为30s
- --default-unreachable-toleration-seconds=100  #对该污点的容忍时长设置为30s
~~~



## Kubelet 发起的驱逐-节点压力驱逐

#### 节点压力驱逐

节点压力驱逐是 kubelet 主动终止 Pod 以回收节点上资源的过程。

kubelet 监控集群节点的内存、磁盘 空间和文件系统的 inode 等资源。 当这些资源中的一个或者多个达到特定的消耗水平， kubelet 可以主动地使节点上一个或者多个 Pod 失效，以回收资源防止饥饿。

在节点压力驱逐期间，kubelet 将所选 Pod 的阶段设置为 Failed 并终止 Pod（其内部的容器会被全 部停止）。 kubelet会上报自己的节点资源的真实使用情况不足这一些信息给apiserver存入etcd，节点控制器会 watch这一信息会变更node的状态：`MemoryPressure: True` ，一旦节点被标注为该状态，那你自己用一些调度规则将pod调度到该节点，该pod不会正常启动会处于 Evicted。



在k8s集群中，节点最重要的资源包括cpu、内存和磁盘。

>cpu —— 可压缩资源，可压缩资源不会导致pod驱逐，因为在资源紧缺时系统内核会重新分配权重
>
>内存/磁盘 —— 不可压缩资源

如果不可压缩资源不足，那系统中很多系统进程、用户进程随时可能申请更多的内存或磁盘资源，内存不足，则会触发系统级的 OOM Killer，系统可不会惯着你，你的k8s进程都可能被干掉，就会导致 k8s 不稳定。

为了避免出现这种系统级别的OOM而导致k8s集群崩溃掉，k8s设计和实现了一套自动化的Pod驱逐机制，该机制会自动从资源紧张的节点上驱逐一定数据量的Pod，以便空出资源。

具体是由 kubelet 负责的，而 kubelet 并不会随机驱逐 Pod，而是由自己的一套驱逐机制，每个节点上的 kubelet 都会通过 cAdvisor 提供的资源使用指标来监控自身节点的资源使用量，并根据这些指标的变化做 出响应的驱逐决定和操作。



#### 驱逐信号和阈值

kubelet 使用各种参数来做出驱逐决定： 驱逐信号、驱逐条件、监控间隔

#### 驱逐信号

驱逐信号是特定资源在特定时间点的当前状态。 kubelet 使用驱逐信号，通过将信号与驱逐条件进行比 较来做出驱逐决定， 驱逐条件是节点上应该可用资源的最小量/阈值，达到驱逐条件则会按照优先级驱逐 pod

| 节点状态           | 驱逐信号                      | 描述                        |
| :----------------- | :---------------------------- | :-------------------------- |
| **MemoryPressure** | `memory.available`            | 节点可用内存                |
| **MemoryPressure** | `allocatableMemory.available` | 节点可分配内存              |
| **DiskPressure**   | `nodefs.available`            | 根文件系统可用空间          |
| **DiskPressure**   | `nodefs.inodesFree`           | 根文件系统可用inode数       |
| **DiskPressure**   | `imagefs.available`           | 容器镜像文件系统可用空间    |
| **DiskPressure**   | `imagefs.inodesFree`          | 容器镜像文件系统可用inode数 |
| **PIDPressure**    | `pid.available`               | 可用PID数量                 |

在上表中，描述列显示了 kubelet 如何获取信号的值。每个信号支持百分比值或者是字面值。 kubelet 计算相对于与信号有关的总量的百分比值。 

注意： memory.available 的值来自 cgroupfs，而不是像 free -m 这样的工具。 这很重要，因为 free -m 在容器中不起作用。

上表主要涉及三个方面，memory、pid和file system。

说到文件系统，kubelet 可识别以下两个特定的文件系统标识符： 

1. nodefs ：节点的主要文件系统，用于本地磁盘卷、emptyDir 卷、日志存储等。 nodefs 包含 /var/lib/kubelet/ 
2. imagefs ：可选文件系统，供容器运行时存储容器镜像和容器可写层。imagesfs 包含 /var/kib/containerd



#### 驱逐条件

驱逐条件的格式就是：nodefs.available < 某个阈值。

例如，如果一个节点的总内存为 10GiB 并且你希望在可用内存低于 1GiB 时触发驱逐， 则可以将驱逐条件定义为 memory.available<10% 或 memory.available< 1G （你不能同时使用二者）。

#### 软/硬驱逐条件

soft 软驱逐：驱逐发生时，有宽限期。

~~~bash
#### 软驱逐(Soft Eviction Thresholds)
软驱逐机制表示，当node的 内存/磁盘 空间达到一定的阈值后，我要观察一段时间，如果改善到低于阈
值就不进行驱逐，若这段时间一直高于阈值就进行驱逐。
一般由三个参数配合使用：
--evictionsoft=memory.available<10%,nodefs.available<15%,imagefs.available<15% \ # 触发
软驱逐策略的阀值
--eviction-soft-graceperiod=memory.available=2m,nodefs.available=2m,imagefs.available=2m \ #宽限
期，表示达到阈值后到开始驱逐动作之前
的时间，如果在此时间内还未恢复到阀值以下，则会开始驱逐pod
--eviction-max-pod-grace-period=30 # 满足驱逐条件需要 Kill Pod时等待Pod 优雅退出的
时间。超过这个时间会强制杀死
注意：
软驱逐的停止条件是资源使用量降到软阈值以下。
没有专门的参数像硬驱逐中的 --eviction-minimum-reclaim 一样来控制软驱逐停止时的回收量。
软驱逐的目的是尽量恢复资源到正常状态，而不是确保回收特定的资源量
~~~

hard 硬驱逐：驱逐发生时，没有宽限期

~~~bash
#### 硬驱逐(Hard Eviction Thresholds)
硬驱逐简单粗暴，没有宽限期，一旦达到阈值配置，kubelet立马回收关联的短缺资源，将pod kill
掉，而不是优雅终止。
# 1、
--evictionhard=memory.available<256Mi,nodefs.available<1Gi,imagefs.available<1Gi \
# 2、
--eviction-minimumreclaim=memory.available=512Mi,nodefs.available=1Gi,imagefs.available=1Gi \
# 表示每一次驱逐必须至少回收多少资源。该参数可以避免在某些情况下，驱逐 Pod 只会回收少量的资
源，导致反复触发驱逐。
~~~

软硬限制的检测判定

~~~bash
--eviction-pressure-transition-period=30s # 持续30s才判定为达到阈值
驱逐等待时间。当节点出现资源饥饿时，节点需要等待一定的时间才一更新节点 Conditions ，然后才开启
驱逐 Pod，默认为5分钟。该参数可以防止在某些情况下，节点在软驱逐条件上下振荡而出现错误的驱逐决
策。
控制的是从资源压力检测到开始驱逐之间的延迟，目的是避免因短期波动而做出不必要的驱逐决策。
~~~

#### 驱逐监测间隔

kubelet 根据其配置的 housekeeping-interval （默认为 10s ）评估驱逐条件

#### 节点状况波动

在某些情况下，节点在软驱逐条件上下振荡，而没有保持定义的宽限期。 这会导致报告的节点条件在 true 和 false 之间不断切换，从而导致错误的驱逐决策。 

为了防止振荡，可以使用 `eviction-pressure-transition-period` 标志， 该标志控制 kubelet 在将节点条件转换为不同状态之前必须等待的时间。 过渡期的默认值为 5m 。 这个参数其实对软硬策略都有用。

~~~bash
# 编辑 kubelet 配置文件
vim /var/lib/kubelet/config.yaml

# 在文件顶头添加以下配置（不用缩进）
evictionHard:
    memory.available: "256Mi"  # 当可用内存低于256Mi时，触发回收操作
    nodefs.available: "1Gi"
    imagefs.available: "1Gi"

evictionMinimumReclaim:
    memory.available: "512Mi"  # 设置了回收操作尝试回收至少达到512Mi的可用内存
    nodefs.available: "1Gi"
    imagefs.available: "1Gi"

evictionPressureTransitionPeriod: "30s"  # 发起杀pod请求之前的冷静期

# 重新加载并重启 kubelet
systemctl daemon-reload
systemctl restart kubelet
~~~



## 驱逐举例

#### 内存不足引发驱逐

当内存不足时，Kubelet会根据Pod的QoS（质量服务等级）来进行排序并逐出。优先逐出的是 BestEffort，接着是Burstable，最后是Guaranteed。这种排序方式确保了资源消耗较少且非关键性的 Pod先被逐出，从而保障关键性服务。

#### 磁盘不足引发驱逐

k8s关于磁盘的容量监测，主要就是监测就监测两种文件系统，本质就是对应两个文件夹。

~~~bash
nodefs ---- /var/lib/kubelet（pod运行相关的各种卷和日志存储） ------ 驱逐掉pod（按照对nodefs的使用量来区分） 

images----- /var/lib/containerd（容器运行时的只读层、可写层）-----删掉停掉的容器、没有被使用的镜像
~~~

k8s 包括两种文件系统 nodefs 与 imagefs，前者统计的是一个节点上的占用、后者统计的是一个节点上的容器占用的，所以二者会有一些共同的占用。 

kubelet检测到下面任意条件满足时（配置文件位置： /var/lib/kubelet/config.yaml），都会触发Pod的驱逐行为。

~~~bash
nodefs.available < 10%
nodefs.inodesFree < 5%
imagefs.available < 15%
imagefs.inodesFree < 15%
~~~

##### 针对 nodefs

- 如果某个节点的 nodefs 达到驱逐阈值，kubelet 会首先删除所有已经失效的pod及其容器实例对应的磁盘文件，如果还不够用怎么办呢？
- 当某个节点达到了 nodefs 的阈值时，在这种情况下引发的驱逐，kubelet 并不会参考 pod 的 Qos 等级，而是按照 pod 对 nodefs 的使用量来进行排序来驱逐。

##### 针对 imagesfs

- 如果某个节点的 imagesfs 达到驱逐阈值，kubelet 并不会参考 pod 的 Qos 等级，kubelet 会删除所有未使用的容器镜像。



## 节点资源紧缺情况下的操作系统行为

#### 调度器的行为

在节点资源紧缺时，节点上的kubelet组件会向Master报告这一情况，在Master上进行的调度器不会再向该节点调度Pod



#### Node的OOM行为

kubelet无法及时观测到内存压力，可能会出现系统级OOM提前触发的情况。

kubelet是周期性执行驱逐pod来释放资源的，如果在周期抵达之前遭遇到了系统的OOM（内存不 足），节点则依赖oom_killer的设置来杀掉某些进程，杀掉哪些进程看评分。

kubelet根据pod的Qos为每个容器都设置了一个oom_score_adj值。

| QoS 等级       | OOM_SCORE_ADJ 值 | 计算公式                                                     | 说明                        |
| :------------- | :--------------- | :----------------------------------------------------------- | :-------------------------- |
| **Guaranteed** | **-998**         | 固定值                                                       | 最后被 OOM Kill，优先级最高 |
| **BestEffort** | **1000**         | 固定值                                                       | 最先被 OOM Kill，优先级最低 |
| **Burstable**  | 动态计算         | `min(max(2, 1000 - (1000 * memoryRequestBytes) / machineMemoryCapacityBytes), 999)` |                             |

如果kubelet无法再系统OOM killer之前回收足够的内存的话，则oom_killer会根据内存的使用比率来计 算oom_score，将得出的结果和oom_score_adj相加，得分最高的Pod首先被杀掉。

这个策略的思路是，Qos最低且相对于调度的Request来说消耗最多内存的Pod会首先被杀掉，来保障内存的回收。

#### 对DaemonSet类型的Pod驱逐

通过DaemonSet创建Pod具有在节点上自动重启的特性，因此我们不希望kubelet驱逐这种Pod，然而 kubelet目前并没有能力分辨DaemonSet的Pod，所以无法单独为其定制驱逐策略。 

所以强烈建议不要在DaemonSet中创建BestEffort类型的Pod，最好是Guaranteed级，避免产生驱逐方 面的问题，然后Daemonset又自动重启Pod，反复横跳/震荡，导致不稳定。



## 如何更改kubelet的默认驱逐阀值呢

#### 百分比或数值

例如一个节点有10GiB内存，想在可用内存不足1GiB时进行pod驱逐，可以有两种形式设置阈值

- 百分比：memory.available < 10%
- 数值: memory.available < 1GiB

#### 硬阀值的设置

将以下策略加入到 `/etc/kubernetes/kubelet.env` 即kublet启动参数即可

~~~bash
# 1、
--evictionhard=memory.available<256Mi,nodefs.available<1Gi,imagefs.available<1Gi \
# 2、
--eviction-minimumreclaim=memory.available=512Mi,nodefs.available=1Gi,imagefs.available=1Gi \
# 3、
--eviction-pressure-transition-period=30s
~~~

- 第一行表示当memory<256Mi，nodefs<1G，imagesfs<1G，时才会触发硬驱逐策略。
- 第二行表示最小驱逐回收策略，就是一旦驱逐策略被触发，则要一直驱逐直到达到此行策略的阀值 为止，为了就是防止刚刚触发硬驱逐策略，驱逐完之后没一会资源又涨上来了，导致要反复驱逐的现象。
- 第三行是为了防止node节点状态振荡。

如果你是kubeadm安装的集群，那你可以用ps aux |grep kubelet查看配置文件路径，或者systemctl status kubelet也可以看到，然后进行修改

~~~bash
$ vim /var/lib/kubelet/config.yaml # 顶头写，不用缩进
evictionHard:
  memory.available: "256Mi" # 当可用内存低于256Mi时，触发回收操作
  nodefs.available: "1Gi"
  imagefs.available: "1Gi"
evictionMinimumReclaim:
  memory.available: "512Mi" # 设置了回收操作尝试回收至少达到512Mi的可用内存
  nodefs.available: "1Gi"
  imagefs.available: "1Gi"
evictionPressureTransitionPeriod: "30s" # 发起杀pod请求之前的冷静期

$ systemctl daemon-reload
$ systemctl restart kubelet
~~~

#### 软阀值的设置

~~~bash
--eviction-soft=memory.available<10%,nodefs.available<15%,imagefs.available<15%
\
--eviction-soft-graceperiod=memory.available=2m,nodefs.available=2m,imagefs.available=2m \
--eviction-max-pod-grace-period=30
~~~

- 第一行表示软驱逐阀值，使用百分比代替
- 第二行表示触发软驱逐后，等待执行的时间，若此时间内低于阀值，则不再进行驱逐
- 第三行为pod宽限时间

~~~bash
$ vim /var/lib/kubelet/config.yaml # 顶头写，不用缩进
evictionSoft:
  memory.available: "512Mi" # 当可用内存低于 512Mi 时，进入软驱逐条件
  nodefs.available: "2Gi"
  imagefs.available: "2Gi"
evictionSoftGracePeriod:
  memory.available: "1m" # 当满足软驱逐条件时，给出 1 分钟的宽限期
  nodefs.available: "5m"
  imagefs.available: "5m"
evictionMaxPodGracePeriod: 120 # 在软驱逐条件下，允许 Pod 在 120 秒内优雅地终止
~~~

#### 限制最大保留的Evicted pod数

当资源不够或发生争夺有pod被驱逐后，pod的状态会变为Eviction，该状态pod并不会被立即删除掉， 需要等到垃圾回收机（周期性执行）触发时，才会进行回收，垃圾回收机制执行时会检查被驱逐的pod 数如果没有达到限制则不会立即清理。

如果资源一直无法协调过来，或者资源真的不够用了，那么会产生大量的Eviction状态的Pod，会影响整 个集群的使用，如何限制Eviction的Pod的数量呢？

此配置其实是由 kube-contoller-manager 来管理的，所以想要修改要修改kube-contollermanager.yaml的参数，一般在/etc/kubernetes/manifests/kube-controller-manager.yaml 添加

~~~bash
--terminated-pod-gc-threshold=1 
# 当垃圾回收机器运行时，发现已终止的pod数达到1，则会清掉这1个之外的所有其他的被驱逐的pod。
# 默认为12500，最小可以设置为1，注意如果你设置为0，则表示没有限制
~~~

注意：

- Kubernetes的GC机制针对多种资源，包括终止的Pod、已完成的Job、以及没有所有者引用的对象进行清 理，系统会自动进行垃圾回收以保持集群资源的健康管理，但具体的回收时机和周期是由系统自动管理的，不 提供直接配置用于精确控制被驱逐Pod的垃圾回收时间的参数。



## 落地经验

**kube-controller-manager的驱逐时间不要过短**

对于 kubelet 发起的驱逐，往往是资源不足导致，它优先驱逐 BestEffort 类型的容器，这些容器多为离线批处理类业务，对可靠性要求低。驱逐后释放资源，减缓节点压力，弃卒保帅，保护了该节点的其它 容器。无论是从其设计出发，还是实际使用情况，该特性非常 nice。

对于由 kube-controller-manager 发起的驱逐，效果需要商榷。正常情况下，计算节点周期上报心跳给 master，如果心跳超时，则认为计算节点 NotReady，当 NotReady 状态达到一定时间后，kubecontroller-manager 发起驱逐。然而造成心跳超时的场景非常多，例如

- 原生 bug：kubelet 进程彻底阻塞
- 误操作：误把 kubelet 停止
- 基础设施异常：如交换机故障演练，NTP 异常，DNS 异常
- 节点故障：硬件损坏，掉电等

从实际情况看，真正因计算节点故障造成心跳超时概率很低，反而由原生 bug，基础设施异常造成心跳超时的概率更大，造成不必要的驱逐。

理想的情况下，驱逐对无状态且设计良好的业务方影响很小。但是并非所有的业务方都是无状态的，也并非所有的业务方都针对 k8s 优化其业务逻辑。例如，对于有状态的业务，如果没有共享存 储，异地重建后的 pod 完全丢失原有数据；即使数据不丢失，对于 Mysql 类的应用，如果出现双写， 重则破坏数据。对于关心 IP 层的业务，异地重建后的 pod IP 往往会变化，虽然部分业务方可以利用 service 和 dns 来解决问题，但是引入了额外的模块和复杂性。

除非满足如下需求，不然请尽量关闭 kube-controller-manager 的驱逐功能，即把驱逐的超时时间设置 非常长，同时把一级／二级驱逐速度设置为 0。否则，非常容易就搞出大大小小的故障，血泪的教训。

- 业务方要用正确的姿势使用容器，如数据与逻辑分离，无状态化，增强对异常处理等
- 分布式存储
- 可靠的 Service／DNS 服务或者保持异地重建后的 IP 不变

