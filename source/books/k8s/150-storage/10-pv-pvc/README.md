# PV 和 PVC

## PV 详解

PV（PersistentVolume）是集群中的一块**网络存储**，由管理员预先配置，或者使用 StorageClass 动态制备。PV 是集群资源，就像节点一样，独立于 Pod 的生命周期，且独立于名称空间。

PV 生命周期

~~~
供应 (Provisioning) → 绑定 (Binding) → 使用 (Using) → 回收 (Reclaiming)
~~~

PV 的声明示例如下

~~~yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv-hostpath
  labels:
    type: local
spec:
  capacity:
    storage: 10Gi
  accessModes:
    - ReadWriteOnce
  persistentVolumeReclaimPolicy: Retain
  storageClassName: manual
  # hostPath 表示 使用宿主机的目录作为存储卷类型
  hostPath:
    path: /data/test1
    type: DirectoryOrCreate
~~~



#### 容量 **capacity**

设定 PV 对象的存储能力，目前只支持存储空间的设置，就是我们这里的 `storage=10Gi` ，不过未来可能会加入 IOPS 、吞吐量等指标的配置。

#### 访问模式 **volumeMode**

AccessModes 是用来对 PV 进行访问模式的设置，用于描述用户应用对存储资源的访问权限，访问权限包括下面几种方式（大都是针对节点级别的）

| 模式             | 缩写 | 说明                                      |
| :--------------- | :--- | :---------------------------------------- |
| ReadWriteOnce    | RWO  | 能被单个节点上的一个或多个pod挂载使用     |
| ReadOnlyMany     | ROX  | 可以同时在多个节点上挂载并被不同的Pod使用 |
| ReadWriteMany    | RWX  | 可以同时在多个节点上挂载并被不同的Pod使用 |
| ReadWriteOncePod | RWOP | 可被单个 Pod 读写（K8s 1.22+）            |

一个 PV 可以设置支持多种访问权限，但是使用者只能基于某一种访问权限来进行访问，不能基于多种。

注意：虽然文档中定义了 ReadWriteOnce 的行为，实际发现 POD 被调度到了不同的节点，在 ReadWriteOnce 模式下依然可以在不同节点共用同一个 PVC 及卷。实际验证中出现不同的行为可能与底层存储的特性或配置有关。

Kubernetes v1.22 提供了第四种访问 PV、PVC 的访问模式： ReadWriteOncePod （单一Pod访问方 式） 当你创建一个带有 pvc 访问模式为 ReadWriteOncePod 的 Pod A 时， Kubernetes 确保整个集群内只有 一个 Pod 可以读写该 PVC 。 此时如果你创建 Pod B 并引用了与 Pod A 相同的 PVC (ReadWriteOncePod)时，那么 Pod B 会由于该 pvc 被 Pod A 引用而启动失败。

只有 CSI 类型存储驱动支持 ReadWriteOncePod ，原生卷插件（如 Hostpath ）并不支持 ReadWriteOncePod 模式。



#### PV 回收策略 **persistentVolumeReclaimPolicy**

- Retain（保留）：当 PVC 被删除时，PV 不会自动删除。它的状态变为“Released”，但保留资源不能被其他 PVC 继续使用，直到管理员手动处理。这允许管理员手动回收资源或者检查数据。重要的数据推荐用该策略。
- Delete（删除）：在这种回收策略下，PVC 被删除时，PV 以及底层的存储资源也会自动被删除。如果 PV 包含重要数据这种策略就很不合适。并且这种模式只支持 hostPath.path 设置为 `/tmp` 下的目录。
- Recycle（回收，已弃用）： PVC 被删除时，PV 上的数据将被简单地删除，而不会销毁 PV 本身。 回收策略会对 PV 进行基本清理（如删除文件内容），然后将其重新标记为 Available ，使其可 以被新的 PVC 再次绑定。由于安全和效能考虑，这个选项在较新的Kubernetes版本中已经被标记为**弃用**。不过需要注意的是，目前只有 NFS 和 HostPath 两种类型支持Recycle这种回收策略，当然一般 来说还是设置为 Retain 这种策略保险一点。ceph存储就不支持Recycle这种回收策略，因为Ceph 等块存储系统需要更复杂的操作来管理存储 资源，简单的文件系统操作无法满足其需求。因此，Kubernetes 不支持在 Ceph 上使用 Recycle 策略。



#### 四种状态

一个 PV 的生命周期中，可能会处于4中不同的阶段： 

- Available（可用）：表示可用状态，还未被任何 PVC 绑定 
- Bound（已绑定）：表示 PV 已经被 PVC 绑定 
- Released（已释放）：PVC 被删除，但是资源还未被集群重新声明 
- Failed（失败）： 表示该 PV 的自动回收失败



#### 存储卷类型

存储卷的类型有很多，我们常用到一般有以下几种： 

- 1、emptyDir（临时目录）:这种存储没有持久化的效果，Pod删除，数据也会被清除，只用于pod内多个容器共享数据的场景 
- 2、hostPath(使用宿主机的目录) 
- 3、网络存储：SAN(iSCSI,FC)、NAS(nfs,cifs,http)存储 
- 4、分布式存储（ceph rbd、cephfs、Longhorn、glusterfs） 
- 5、云存储（EBS，Azure Disk）

查看k8s支持的存储类型

~~~bash
kubectl explain pod.spec.volumes
~~~



## PVC 详解

**PersistentVolumeClaim (PVC)** 是用户对存储的**请求**。它类似于 Pod，Pod 消耗节点资源，而 PVC 消耗 PV 资源。PVC 可以请求特定大小和访问模式的存储。

~~~bash
用户创建 PVC → Kubernetes 匹配/创建 PV → PVC 绑定到 PV → Pod 使用 PVC
~~~

PVC 基本声明如下

~~~yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: myclaim
  namespace: default
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 8Gi
  storageClassName: standard
~~~

## PVC 生命周期管理

### 1. **创建 PVC**

```bash
# 创建 PVC
kubectl apply -f pvc.yaml

# 查看创建状态
kubectl get pvc
kubectl describe pvc my-pvc

# 查看事件
kubectl get events --field-selector involvedObject.kind=PersistentVolumeClaim
```



### 2. **PVC 绑定流程**

```bash
# 查看绑定详情
kubectl get pvc -o yaml my-pvc | grep -A5 -B5 phase

# 如果 PVC 处于 Pending 状态，检查：
# 1. 是否有匹配的 PV
kubectl get pv
# 2. StorageClass 是否正确
kubectl get storageclass
# 3. 查看详细错误
kubectl describe pvc my-pvc
```



### 3. **PVC 扩容**

```bash
# 前提：StorageClass 必须 allowVolumeExpansion: true

# 1. 查看当前容量
kubectl get pvc my-pvc

# 2. 编辑 PVC 扩容
kubectl edit pvc my-pvc
# 修改 spec.resources.requests.storage: 20Gi

# 3. 查看扩容进度
kubectl describe pvc my-pvc | grep -A5 -B5 "resizing"

# 4. 验证文件系统扩容（进入Pod）
kubectl exec my-pod -- df -h
```



### 4. **PVC 删除**

```bash
# 删除 PVC（根据回收策略处理 PV）
kubectl delete pvc my-pvc

# 强制删除（如果卡在 Terminating）
kubectl patch pvc my-pvc -p '{"metadata":{"finalizers":null}}'

# 删除所有命名空间的 PVC
kubectl delete pvc --all --all-namespaces
```