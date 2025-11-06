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
2. 使用 `kubectl edit xxx.yaml` 自动打开 yaml 文件，修改 `replicas` 的值，保存退出后自动扩缩容。
3. 使用 `kubectl sacle deploy web --replicas=3` 完成扩缩容。



## 节点污点

调度 pod 前需要预选节点，预选时有一些规则，比如节点污点。打污点以后就不会被调度。

#### 查看节点污点信息

~~~bash
kubectl describe node k8s-master-01 | grep Taints
~~~



#### 污点策略说明

- NoSchedule：Pod一定不会被调度到该节点
- PreferNoSchedule：Pod尽量不被调度到该节点（仍有被调度的可能）
- NoExecute：Pod不会被调度，并且会驱逐节点上已有的Pod



> 补充：静态 pod 不受污点影响，直接由 kubelet 直接拉起。



#### 污点管理命令

~~~bash
# 添加污点：
kubectl taint nodes k8s-master-01 node-role.kubernetes.io/control-plane:NoSchedule

#删除污点：
kubectl taint nodes k8s-master-01 node-role.kubernetes.io/control-plane:NoSchedule-
~~~



#### 容忍机制

容忍机制允许 Pod 被调度到带有匹配污点的节点上，它通过给 pod 配置与污点的键、值和效果相匹配的规则来实现。







## pod 调度策略

调度分为：预选阶段、优先阶段。

