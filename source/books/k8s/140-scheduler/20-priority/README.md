# 优先级调度

在 Kubernetes 中，Pod 优先级调度是确保集群关键工作负载即使在资源紧张时也能被优先调度的重要机制。它通过 `PriorityClass` 和 `priority` 字段来实现。

#### 工作原理

1. **调度顺序**：当 kube-scheduler 需要调度 Pod 时，它会根据 Pod 的优先级进行排序。优先级高的 Pod 会被**优先尝试调度**。这并不意味着高优先级 Pod 一定能调度成功（如果资源不足，它也会处于 Pending 状态），但它会比低优先级 Pod 更早地进入调度流程。
2. **抢占**：这是优先级调度的关键特性。
   - 当一个高优先级的 Pod 无法被调度（例如，没有满足条件的节点有足够资源）时，调度器会尝试**抢占**。
   - 调度器会寻找那些运行着较低优先级 Pod 的节点。如果抢占这些低优先级 Pod 的资源后，能够满足高优先级 Pod 的需求，调度器就会执行抢占。
   - 被抢占的低优先级 Pod 会被**优雅终止**（收到 SIGTERM 信号），并从该节点上移除，从而为高优先级 Pod 腾出资源。

~~~alert type=important
调度（scheduling）指的是确保Pod匹配到合适的节点，然后由kubelet创建 pod，负责调度的组件是：kube-scheduler。<br>
抢占（Preemption）指的是终止低优先级的 pod 以便高优先级的 Pod 可以调度运行的过程。触发的时机：创建一个高优先级的 pod，但是因为资源不足而迟迟无法被调度时。依据的规则是pod的priorityClassName优先级类里规定的优先级值来作为判定优先级高低的用依据，而不是Qos。<br>
某个节点遇到不可压缩资源剩余量达预设的阈值时则会触发驱逐,负责驱逐的组件是：kubelet,驱逐 pod 的依据是 Qos 等级。
~~~



#### Pod 优先级和服务质量 QoS

<font color="blue" size="3">调度器的抢占逻辑在选择抢占目标时不考虑 QoS</font>。 抢占会考虑 Pod 优先级并尝试选择一组优先级最低的目标，仅当移除优先级最低的 Pod 不足以让调度程序调度抢占式 Pod， 或者最低优先级的 Pod 受 PodDisruptionBudget （简称PDB策略）保护时，才会考虑优先级较高一些的Pod，总之一句话抢占逻辑依据的是优先级高低，而不是Qos。

kubelet 使用优先级Qos来确定 [节点压力驱逐](https://kubernetes.io/zh-cn/docs/concepts/scheduling-eviction/node-pressure-eviction/) Pod 的顺序。 

当某 Pod 的资源用量未超过其请求时，kubelet 节点压力驱逐不会驱逐该 Pod。 如果优先级较低的 Pod 的资源使用量没有超过其请求，则不会被驱逐。 另一个优先级较高且资源使用量超过其请求的 Pod 可能会被驱逐。

注意：优先级抢占的优先程度高于其他调度规则例如反亲和。



#### 具体操作

创建一个优先级类，类中定义好优先级的值，值越大优先级越高。然后把 pod 关联该优先级类，此时就会按照优先级来调度。优先级调度可能会引发抢占行为。抢占（Preemption）指的是终止低优先级的 pod 以便高优先级的 Pod 可以调度运行的过程。



创建 PriorityClass。不需要指定名称空间（因为 PriorityClass 是跨名称空间使用的）

~~~yaml
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: high-priority
value: 1000000            # 高优先级值
globalDefault: false      # 仅适用于明确指定此类的 Pod
description: "desc"
~~~

在 Pod 中引用 PriorityClass

~~~yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx-high-priority
spec:
  containers:
  - name: nginx
    image: nginx
  priorityClassName: high-priority 	# 关键字段
~~~

- 抢占可能导致服务中断（低优先级 Pod 被终止）。
- 调度器会遵循 PodDisruptionBudget，如果抢占违反 PDB 规则，则抢占不会发生。
- 系统 Pod（如 `kube-system` 命名空间下的 DaemonSet）通常具有很高的优先级（如 `system-cluster-critical`, `system-node-critical`），以防止关键系统组件被抢占。
- 抢占只考虑资源（CPU， 内存），而不考虑其他约束（如节点选择器、亲和性、污点容忍等）。如果高优先级 Pod 有特殊的节点选择要求，即使腾出资源，也可能无法调度到目标节点。

#### 查看优先级的命令

~~~bash
# 查看所有 PriorityClass
kubectl get priorityclass
kubectl get pc

# 查看特定 PriorityClass 的详情
kubectl describe priorityclass <name>

# 查看 Pod 的优先级信息
# kubectl get pod <pod-name> -o custom-columns=NAME:.metadata.name,PRIORITYCLASS:.spec.priorityClassName
[root@k8s-master-01 test5]# kubectl get pod nginx-high-priority -o custom-columns=NAME:.metadata.name,PRIORITYCLASS:.spec.priorityClassName
NAME                  PRIORITYCLASS
nginx-high-priority   high-priority
~~~



## 服务的质量等级 QoS

申请资源与真正使用是两回事，k8s 调度就根据 pod 的 request 设置的资源申请来判定调度的，pod 的 request 只是用来申请资源的，当 pod 运行起来之后并不一定会用这么多资源。

k8s 判断某个节点的资源可分配量，也不是按照本节点上资源的使用量去判定的，而是本节点上所有存在的 pod 的 request 之和代表已经分配走的资源。

建议每个pod都设置好requests值，配合使用 validate webhook 来硬性限制遵循该规范。



使用 describe  查看节点的资源，是以 reques t为基础来进行判定的，即只用于调度时的分配，而不管资源的真实使用，也就是说资源可以超配的。关于资源的真实使用的管理，如果用超了，会有 kubelet 的驱逐机制进行管理。

~~~bash
[root@k8s-master-01 test5]# kubectl describe nodes k8s-node-01
Name:               k8s-node-01
...
...
Capacity:
  cpu:                2
  ephemeral-storage:  41103804Ki
  hugepages-1Gi:      0
  hugepages-2Mi:      0
  memory:             1919448Ki
  pods:               110
# Allocatable 是可分配的资源
Allocatable:
  cpu:                2
  ephemeral-storage:  37881265704
  hugepages-1Gi:      0
  hugepages-2Mi:      0
  memory:             1817048Ki
  pods:               110
...
...
# Allocated resources已经分配的资源，按照 request 申请的数据统计的
Allocated resources:
  (Total limits may be over 100 percent, i.e., overcommitted.)
  Resource           Requests     Limits
  --------           --------     ------
  cpu                325m (16%)   0 (0%)
  memory             345Mi (19%)  0 (0%)
  ephemeral-storage  0 (0%)       0 (0%)
  hugepages-1Gi      0 (0%)       0 (0%)
  hugepages-2Mi      0 (0%)       0 (0%)
~~~

#### 服务质量 QoS 类型

从高到低排序分别如下：
- **Guaranteed**(完全可靠的/有保证的)：pod中的所有容器对所有资源类型都定义了Limits与requests，且二者值相等、且大于0	
- **Burstable**(弹性被动、较可靠的)：具体来说此级别涉及两种情况：
  1. Pod 中的一部分容器在一种或者多种类型的资源配置中定义了Requests值和Limits值（都不为0），且Requests值小于Limits
  2. Pod 中的一部分容器未定义资源配置（Requests值和Limits值都未定义）
- **BestEffort**(尽力而为、不太可靠的)：pod中的所有容器都未定义资源配置（Requests与Limits都未定义），那么该pod的Qos级别就是 BestEffort



当节点的资源真实使用量超过了 kubelet 预设的阈值，kubelet 则会按照 QoS 等级来进行 pod 驱逐按照顺序，先驱逐 **BestEffort** 类型的 pod，其次是 **Burstable**，最后才是 **Guaranteed**。

