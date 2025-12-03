# apiserver

## API 组织形式

#### apiserver 的三大核心功能

- 对组件的贡献。对下封装了对 etcd 数据库的操作、对上提供了 list-watch 机制。
- 对客户端的贡献。对客户端提供了api接口（总体来是基于RESTFul风格设计api）

~~~bash
https://ip:port/api/apps/v1/namespaces/<namespace>/pods
~~~

- 对安全的贡献。客户端访问的时候要经过一系列权限相关处理，包括：检查凭证、检验授权、准入控制器（两种 webhook：mutate 和 validate）

#### api 的设计

总体来说在 resfulset 风格的基础上做了进一步设计，核心为：分组和多版本。

具体组织：

- 顶级分组：`/api`、`/apis`、`/metrics` 等。
- 细分组名：细分组名全局唯一，清单中指定的就是细分组名，比如：`apps`
- 具体版本：`v1`
- 具体的资源：`job`、`pod` 等。

~~~alert type=note
非核心组api组成：/顶级大类/细分组名/版本/namespaces/具体的名称空间/资源类型 <br>
核心组api组成：/api/版本/namespaces/具体的名称空间/资源类型
~~~

api 版本

~~~bash
1、alpha版：尝鲜版本，新特性一定最先出现在这里
2、Beta版：稳定性提高版本，相对alpha版更稳定一些
3、稳定级别：比如v1版本表示是最稳定版了
~~~

比如 deployment 属于非核心组

~~~bash
[root@k8s-master-01 ~]# kubectl explain deployment
GROUP:      apps
KIND:       Deployment
VERSION:    v1
~~~

一个 deployment 的 yaml 文件，通过 `apiVersion: apps/v1 `指定细分组和版本，完成的 api 是：`https:apiserverip:port/apps/v1/namespaces/default/deployment`

~~~yaml
[root@k8s-master-01 ~]# cat nginx.yaml 
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
  nsmespace: default
  labels:
    app: nginx
spec:
  ...
~~~

pod、service 等属于核心组

~~~bash
[root@k8s-master-01 ~]# kubectl explain pod
KIND:       Pod
VERSION:    v1
~~~

kubectl 指定 apiserver 的地址和端口，查看家目录的

~~~
[root@k8s-master-01 ~]# grep server ~/.kube/config 
    server: https://172.16.143.22:6443
~~~



查看 api 组织结构 ，使用 `kubectl get --raw /`

~~~bash
[root@k8s-master-01 ~]# kubectl get --raw / | head -10
{
  "paths": [
    "/.well-known/openid-configuration",
    "/api",
    "/api/v1",
    "/apis",
    "/apis/",
    "/apis/admissionregistration.k8s.io",
    "/apis/admissionregistration.k8s.io/v1",
    "/apis/apiextensions.k8s.io",
[root@k8s-master-01 ~]# 
~~~

如果想要想看具体信息，可以进一步查看

~~~bash
[root@k8s-master-01 ~]# kubectl get --raw /apis/apps/ | python -m json.tool 
{
    "apiVersion": "v1",
    "kind": "APIGroup",
    "name": "apps",
    "preferredVersion": {
        "groupVersion": "apps/v1",
        "version": "v1"
    },
    "versions": [
        {
            "groupVersion": "apps/v1",
            "version": "v1"
        }
    ]
}
~~~

使用 `kubectl proxy` 命令开启对 apiserver 的访问。

~~~bash
[root@k8s-master-01 ~]# kubectl proxy
Starting to serve on 127.0.0.1:8001

~~~

手动访问

~~~bash
curl http://127.0.0.1:8001/api/v1/namespaces/default/pods
~~~

等同于使用 kubectl

~~~bash
kubectl -n default get pods
~~~



## 认证和授权

限制用户对某个 api 下的某种类型资源拥有某种权限。具体从三个维度做资源权限限制。

- api
- 资源类型
- 具体的操作

做权限限制需要实体账号，每种账号都只是代号而已，必须关联上凭证才能用，k8s 确认用户身份用的是凭证。

账号可以分成两类：

- 给人用的。用户名、组
- 给 pod 用的。serviceaccount，让 pod 内的环境拥有对 k8s 的操作权限。

凭证：包含的是证书、token、归属的名称空间。

k8s 基于 rbac 机制，实现权限控制

~~~
账号 ----- 绑定某种角色 ------- role(权限)
~~~

#### 角色

角色配置权限。角色分两类：

- role (私有角色)。此类角色只能被单个名称空间看到并加以引用。
- clusterrole (公共的角色)。此类角色能够被所有名称空间看到并加以引用。



#### 绑定角色

用户和角色绑定后才有权限。角色绑定方式非两类：

- rolebinding：授予的权限只能在一个名称空间中使用。
- clusterrolebingding：授予的权限能在所有名称空间中使用。



<font style="color: red">强调：角色本身区分名称空间，单独名称空间可见还是全局可见。至于说角色下的 rules 规则能在哪个名称空间执行，取决于角色绑定方式。</font>



#### 角色和绑定角色的搭配使用

- role 使用 rolebinding，授予的权限只能在一个名称空间中用
- clusterrole 使用 clusterrolebingding， 授予的权限能在所有名称空间中用
- clusterrole 使用 rolebinding，依然局限在一个名称空间中使用，但是只需要定义一个公共的角色即可，方便维护权限。



#### apiserver验证流程

- 认证(Authentication)：基于凭证用于确认用户身份。
- 授权(Authorization： 验证该凭证关联的权限（apiGroup、resources、verbs)
- 准入控制(Admission Controllers)：触发两种webhook（mutate与validate）





#### 创建账号及凭证

给人使用的账号和凭证需要手动创建。

~~~bash
# 创建一个私钥文件,得到 kaxon.key
(umask 077;openssl genrsa -out kaxon.key 2048)

# 创建证书签名请求文件 kaxon.csr
# CN 表示用户名，O 表示组
openssl req -new -key kaxon.key -out kaxon.csr -subj "/CN=kaxon/O=developers"

# 使用 Kubernetes CA 签署证书
# 获取集群 CA 证书和私钥（通常位于 /etc/kubernetes/pki 或 ~/.minikube）
# 如果是二进制安装的，证书可能是 
# /opt/kubernetes/ssl/ca.pem      -> ca.crt
# /opt/kubernetes/ssl/ca-key.pem  -> ca.key
# 签署证书，得到文件 kaxon.crt
openssl x509 -req -in kaxon.csr \
  -CA /etc/kubernetes/pki/ca.crt \
  -CAkey /etc/kubernetes/pki/ca.key \
  -CAcreateserial \
  -out kaxon.crt \
  -days 365
  

# 使用整数文件和私钥文件，在集群中创建凭证和上下文context
# 设置用户凭证
kubectl config set-credentials kaxon \
  --client-certificate=kaxon.crt \
  --client-key=kaxon.key \
  --embed-certs=true  # 可选：将证书嵌入 kubeconfig

# 设置上下文（关联用户、集群和命名空间）
kubectl config set-context kaxon-context \
  --cluster=kubernetes \
  --namespace=kube-system \
  --user=kaxon

# 查看当前配置
kubectl config view

# 目前没有权限 
kubectl get pods --context=kaxon-context
Error from server (Forbidden): pods is forbidden: User "kaxon" cannot list resource "pods" in API group "" in the namespace "kube-system"
[root@k8s-master-01 test]# 

# 切换到新上下文
kubectl config use-context kaxon-context
~~~

#### 创建 role 类型的角色

使用命令生成创建 role 的 yaml 文件

~~~bash
kubectl create role role1 --verb=get,list,watch --resource=pods --dry-run=client -o yaml > role-demo.yaml
~~~

得到如下 yaml 文件，顺便修改 namespace 为 kube-system，和上面新建的用户保持一致。

~~~yaml
# role-demo - 定义权限
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  namespace: kube-system
  name: role1
rules:
- apiGroups: [""]		# [""] 表示核心组
  resources: ["pods"]
  verbs: ["get", "watch", "list"]
~~~

使用 apply 创建，然后查看角色。role 类型的角色是私有类型，只能在相应的名称空间看到。

~~~bash
[root@k8s-master-01 test]# kubectl -n kube-system get role | grep role1
role1                                            2025-12-02T06:44:27Z
~~~

导出 context

~~~bash
kubectl config view --minify --flatten --context=kaxon-context > kaxon-context-kubeconfig.yaml
~~~



#### 创建 rolebinding

使用命令创建 yaml 文件

~~~bash
kubectl create rolebinding rolebinding1 --role=role1 --user=kaxon --dry-run=client -o yaml >  rolebinding.yaml
~~~

得到如下 yaml 文件，并修改名称空间，与用户的名称空间保持 一致。

```yaml
# rolebinding.yaml - 绑定用户和角色  
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: rolebinding1
  namespace: kube-system
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: role1
subjects:
- apiGroup: rbac.authorization.k8s.io
  kind: User
  name: kaxon
```

使用 apply 创建后，查看 rolebinding

~~~bash
[root@k8s-master-01 test]# kubectl -n kube-system  get rolebindings.rbac.authorization.k8s.io | grep rolebinding1
rolebinding1               
~~~

绑定关系创建后，具有查看 Pod 的权限

~~~bash
kubectl get pods --context=kaxon-context
NAME                                    READY   STATUS    RESTARTS       AGE
coredns-6d58d46f65-f9w8m                1/1     Running   0              14d
coredns-6d58d46f65-nf5hp                1/1     Running   0              14d
etcd-k8s-master-01                      1/1     Running   0              14d
kube-apiserver-k8s-master-01            1/1     Running   0              6d3h
kube-controller-manager-k8s-master-01   1/1     Running   1 (6d3h ago)   14d
kube-proxy-lnbfd                        1/1     Running   0              14d
kube-proxy-vppzk                        1/1     Running   0              14d
kube-proxy-xctc5                        1/1     Running   0              14d
kube-scheduler-k8s-master-01            1/1     Running   1 (6d3h ago)   14d
metrics-server-85bc58ccff-6hgj8         1/1     Running   0              6d3h
~~~

deployments 看不到

~~~bash
[root@k8s-master-01 test]# kubectl get deployments --context=kaxon-context
Error from server (Forbidden): deployments.apps is forbidden: User "kaxon" cannot list resource "deployments" in API group "apps" in the namespace "kube-system"
~~~

添加查看 deployments 的权限

~~~yaml
# role-demo - 定义权限
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  namespace: kube-system
  name: role1
rules:
- apiGroups: ["", "apps"]		# [""] 表示核心组
  resources: ["pods", "deployments"]
  verbs: ["get", "watch", "list"]  # ["*"] 表示所有权限
~~~

更新 role

~~~bash
kubectl apply -f role-demo.yaml
~~~

有了查看 deployments 的权限

~~~bash
kubectl get deployments.apps --context=kaxon-contextNAME             READY   UP-TO-DATE   AVAILABLE   AGE
coredns          2/2     2            2           14d
metrics-server   1/1     1            1           6d3h
~~~



#### kubectl 命令检索 context 的优先级

优先级最高的是在 kubectl 命令上直接使用 `--kubeconfig=/path/to/your-kubeconfig.yaml` 指定凭证文件路径

~~~bash
kubectl --kubeconfig=/root/.kube/config.yaml get pods
~~~

其次，使用环境变量 `KUBECONFIG`（默认值 /etc/kubernetes/admin.conf）

~~~bash
export KUBECONFIG=/path/to/your-kubeconfig.yaml
~~~

最后，查找 `~/.kube/config `



#### kubectl 上下文管理命令

查看上下文

~~~bash
[root@k8s-master-01 kubernetes]# kubectl config get-contexts 
CURRENT   NAME                          CLUSTER      AUTHINFO           NAMESPACE
          kaxon-context                 kubernetes   kaxon              kube-system
*         kubernetes-admin@kubernetes   kubernetes   kubernetes-admin 

# view
kubectl config view
~~~

切换上下文

~~~bash
kubectl config use-context kaxon-context
~~~

清理和删除

~~~bash
# 删除 context
kubectl config delete-context <context-name>

# 查看用户
kubectl config view -o jsonpath='{.users[*].name}'

# 删除用户
kubectl config unset users.<user-name>
~~~



## 创建 clusterrole 并绑定 clusterrolebinding

#### 创建 clusterrole

创建 clusterrole

~~~bash
kubectl create clusterrole cluster-read --verb=get,list,watch --resource=pods --dry-run=client -o yaml >  clusterrole-demo.yaml 
~~~

查看 yaml 文件

~~~bash
[root@k8s-master-01 test]# cat clusterrole-demo.yaml 
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  creationTimestamp: "2025-12-02T07:48:02Z"
  name: cluster-read
  resourceVersion: "1841325"
  uid: c45d65bf-9b4f-4475-ae2d-aa8a25b68a5b
rules:
- apiGroups:
  - ""
  resources:
  - pods
  verbs:
  - get
  - list
  - watch
~~~

应用，得到 clusterrole，所有名称空间都可以看到 clusterrole

~~~bash
# 应用
kubectl apply -f clusterrole-demo.yaml 
# 查看
[root@k8s-master-01 test]# kubectl get clusterrole | grep cluster-read
cluster-read                                                           2025-12-02T07:52:36Z
~~~

#### 新建 clusterrolebinding

命令行绑定角色

~~~bash
kubectl create clusterrolebinding kaxon-read-all-pods --clusterrole=cluster-read --user=kaxon --dry-run=client -o yaml > clusterrolebingding.yaml
~~~

得到 yaml 文件

~~~yaml
# clusterrolebingding.yaml 
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  creationTimestamp: null
  name: kaxon-read-all-pods
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-read
subjects:
- apiGroup: rbac.authorization.k8s.io
  kind: User
  name: kaxon
~~~

应用后，该账号有权限查看资源

~~~bash
[root@k8s-master-01 test]# kubectl get pods --context=kaxon-context
NAME                                    READY   STATUS    RESTARTS       AGE
coredns-6d58d46f65-f9w8m                1/1     Running   0              14d
coredns-6d58d46f65-nf5hp                1/1     Running   0              14d
etcd-k8s-master-01                      1/1     Running   0              14d
kube-apiserver-k8s-master-01            1/1     Running   0              6d4h
kube-controller-manager-k8s-master-01   1/1     Running   1 (6d4h ago)   14d
kube-proxy-lnbfd                        1/1     Running   0              14d
kube-proxy-vppzk                        1/1     Running   0              14d
kube-proxy-xctc5                        1/1     Running   0              14d
kube-scheduler-k8s-master-01            1/1     Running   1 (6d4h ago)   14d
metrics-server-85bc58ccff-6hgj8         1/1     Running   0              6d4h
~~~

并且可以看到任何名称空间下的资源（默认看到的是当前context所属账号的名称空间）

~~~bash
[root@k8s-master-01 test]# kubectl -n default get pods --context=kaxon-context
NAME                                 READY   STATUS    RESTARTS   AGE
nginx-deployment-595cc44f8b-4b6d5    1/1     Running   0          6h29m
reloader-reloader-78ccff7557-jp2wj   1/1     Running   0          24h
~~~



## 创建 clusterrole 并绑定 rolebinding

权限是全局的，但是想把某个用户的权限限制在一个名称空间下面，此时使用 rolebinging

删除原有绑定关系

~~~bash
kubectl delete -f clusterrolebingding.yaml
~~~

使用命令新建 rolebing 的 yaml 文件

~~~bash
kubectl create rolebinding rolebinding2 --clusterrole=cluster-read --user=kaxon --dry-run=client -o yaml > test.yaml
~~~

得到如下 yaml 文件，编辑名称空间为 `default`

~~~yaml
# test.yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  creationTimestamp: null
  name: rolebinding2
  namespace: default
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-read
subjects:
- apiGroup: rbac.authorization.k8s.io
  kind: User
  name: kaxon
~~~

应用绑定关系

~~~bash
kubectl apply -f test.yaml
~~~

测试发现只能看到 `default `名称空间下的资源

~~~bash
# 用户 kaxon 默认名称空间是 kube-system
[root@k8s-master-01 test]# kubectl get pods --context=kaxon-context
Error from server (Forbidden): pods is forbidden: User "kaxon" cannot list resource "pods" in API group "" in the namespace "kube-system"
[root@k8s-master-01 test]# 
[root@k8s-master-01 test]# 
[root@k8s-master-01 test]# kubectl -n default  get pods --context=kaxon-context
NAME                                 READY   STATUS    RESTARTS   AGE
nginx-deployment-595cc44f8b-4b6d5    1/1     Running   0          6h42m
reloader-reloader-78ccff7557-jp2wj   1/1     Running   0          24h
[root@k8s-master-01 test]# 
[root@k8s-master-01 test]# 
[root@k8s-master-01 test]# 
[root@k8s-master-01 test]# kubectl -n kubec-system  get pods --context=kaxon-context
Error from server (Forbidden): pods is forbidden: User "kaxon" cannot list resource "pods" in API group "" in the namespace "kubec-system"
~~~



## serviceAccount

给 pod 使用的账号，就是 serviceaccount（简称为 sa），一定要区分名称空间。

#### 单个名称空间的 sa 权限控制

具体操作分三步：

- 创建 sa
- 创建 role
- 使用 rolebinding 绑定 sa 和 role

创建 sa，指定名称空间（默认是 default）

~~~bash
kubectl create sa kaxon-sa -n kube-system --dry-run=client -o yaml > sa-role-rolebinding.yaml
~~~

得到如下 yaml 文件，再此基础上增加 role 和 rolebinding，这种权限只能在指定的名称空间中使用。

~~~yaml
# sa-role-rolebinding.yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: kaxon-sa
  namespace: kube-system
---
# 创建 role类型的角色
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  namespace: kube-system
  name: role-kaxon-sa
rules:
- apiGroups: [""]   
  resources: ["pods"]
  verbs: ["get", "watch", "list"]
- apiGroups: ["apps"]   
  resources: ["deployments"]
  verbs: ["*"]
---
# 创建 rolebinding
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: rolebinding2
  namespace: kube-system
roleRef:
  kind: Role
  apiGroup: rbac.authorization.k8s.io
  name: role-kaxon-sa
subjects:
  - kind: ServiceAccount
    name: kaxon-sa
    namespace: kube-system
~~~

创建

~~~bash
[root@k8s-master-01 test]# kubectl apply -f  sa-role-rolebinding.yaml 
serviceaccount/kaxon-sa created
role.rbac.authorization.k8s.io/role-kaxon-sa created
rolebinding.rbac.authorization.k8s.io/rolebinding2 created
~~~

测试方法1

~~~bash
# 返回 no 因为查看的是 default 名称空间
kubectl auth can-i get pods --as=system:serviceaccount:kube-system:kaxon-sa

# 返回 no
kubectl auth can-i get pods -n default --as=system:serviceaccount:kube-system:kaxon-sa

# 返回 yes
kubectl auth can-i get pods -n kube-system --as=system:serviceaccount:kube-system:kaxon-sa
~~~

测试方法2：新建 pod，在 pod 内测试

~~~yaml
# deployment-sa.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: test-deployment
  namespace: kube-system  # 必须指定
  labels:
    app: test-app
spec:
  replicas: 1
  selector:
    matchLabels:
      app: test-app
  template:
    metadata:
      labels:
        app: test-app
    spec:
      # 指定 ServiceAccount
      serviceAccountName: kaxon-sa
      containers:
      - name: centos
        image: crpi-hdcg863vh2ayst45.cn-shanghai.personal.cr.aliyuncs.com/liuxu8677/centos:7
        command: ["tail", "-f", "/dev/null"]
~~~

应用后，进入pod

~~~bash
kubectl -n kube-system exec -it xxx -- bash

# 查看 token
cat /var/run/secrets/kubernetes.io/serviceaccount/token

# 赋值变量
TOKEN=$(cat /var/run/secrets/kubernetes.io/serviceaccount/token)

# curl 如下请求可以拿到结果
curl --header "Authorization: Bearer $TOKEN" --insecure \
https://kubernetes.default.svc/api/v1/namespaces/kube-system/pods

# 如下请求不满足名称空间要求，拿不到结果
[root@test-deployment-6d9b6f8445-85thf /]# curl --header "Authorization: Bearer $TOKEN" --insecure https://kubernetes.default.svc/api/v1/namespaces/default/pods
{
  "kind": "Status",
  "apiVersion": "v1",
  "metadata": {},
  "status": "Failure",
  "message": "pods is forbidden: User \"system:serviceaccount:kube-system:kaxon-sa\" cannot list resource \"pods\" in API group \"\" in the namespace \"default\"",
  "reason": "Forbidden",
  "details": {
    "kind": "pods"
  },
  "code": 403
}
~~~



#### 集群级别的 sa 权限控制

~~~bash
sa  ---- clusterrolebinding ---- clusterrole
~~~

只要是 sa 都要归属一个名称空间，先找到这个名称空间中找到后，才可以使用，因此关联的 Pod 必须和 sa 在一个名称空间中。找到后，如果 sa 绑定的是全局的 clusterole，使用clusterrolebinding 绑定关系，那么权限是不受名称空间限制的。

创建 sa 、clusterrole、clusterrolebinding

~~~yaml
# sa-clusterrole-clusterrolebinding.yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: kaxon-sa2
  namespace: kube-system
---
# 创建 role类型的角色
# 简单起见，直接使用 cluster-admin 这个cluster-role
---
# 创建 clusterrolebinding
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: kaxon-sa2-clusterrolebinding
subjects:
  - kind: ServiceAccount
    name: kaxon-sa2
    namespace: kube-system
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-admin	# 直接使用现有的管理员权限
~~~

测试方法1

~~~bash
# 返回 no。 因为在 default 名称空间找不到 kaxon-sa2
kubectl auth can-i get pods -n kube-system --as=system:serviceaccount:default:kaxon-sa2

# 找到 kaxon-sa2 后，权限不受名称空间限制
# 返回 yes
kubectl auth can-i get pods --as=system:serviceaccount:kube-system:kaxon-sa2

# 返回 yes
kubectl auth can-i get pods -n kube-system --as=system:serviceaccount:kube-system:kaxon-sa2
~~~

也可以在 pod 内使用 curl 的方式手动测试权限。



## security context

Security Context 定义了 Pod 或容器的权限和访问控制设置，主要包括：

- **用户和组 ID**（runAsUser, runAsGroup, fsGroup）
- **权能**（Capabilities）
- **SELinux 上下文**
- **特权模式**
- **只读根文件系统**
- **其他安全设置**



#### 设置 pod 级别的 security context

~~~yaml
# pod-security-basic.yaml
apiVersion: v1
kind: Pod
metadata:
  name: security-context-demo
spec:
  # Pod 级别的 Security Context（应用到所有容器）
  securityContext:
    runAsUser: 1000          # 以非 root 用户运行
    runAsGroup: 3000         # 设置主组
    fsGroup: 2000           # 挂载卷的文件系统组
    runAsNonRoot: true      # 确保不以 root 运行
    supplementalGroups:     # 附加组
    - 1001
    - 1002
    fsGroupChangePolicy: "OnRootMismatch"  # 文件系统组变更策略
  containers:
  - name: sec-ctx-demo
    image: busybox:latest
    command: ["tail", "-f", "/dev/null"]
    # 容器级别的 Security Context（可覆盖 Pod 级别设置）
    securityContext:
      allowPrivilegeEscalation: false
      readOnlyRootFilesystem: true
~~~

#### 设置 container 级别的 security context

~~~yaml
# pod-security-basic.yaml
apiVersion: v1
kind: Pod
metadata:
  name: security-context-demo
spec:
  # Pod 级别的 Security Context（应用到所有容器）
  securityContext:
    runAsUser: 1000          # 以非 root 用户运行
  containers:
  - name: demo1
    image: busybox:latest
    command: ["tail", "-f", "/dev/null"]
    # 容器级别的 Security Context（可覆盖 Pod 级别设置）
    securityContext:
      allowPrivilegeEscalation: false
  - name: demo2
    image: busybox:latest
    command: ["tail", "-f", "/dev/null"]
~~~

#### 开启特权模式

特权模式（Privileged Mode）让容器拥有几乎与宿主机相同的权限。

注意：**特权模式是最后的手段**，应优先考虑使用更细粒度的安全控制（如 Capabilities、Seccomp、AppArmor）。

注意：只能设置容器级别的特权模式。

~~~yaml
# privileged-pod.yaml
apiVersion: v1
kind: Pod
metadata:
  name: fully-privileged-pod
spec:
  containers:
  - name: privileged-container
    image: ubuntu:latest
    command: ["sleep", "infinity"]
    securityContext:
      # 容器也可以有自己的特权设置（覆盖 Pod 级别）
      privileged: true
~~~

进入容器内设置内核参数。

~~~bash
[root@k8s-master-01 test]# kubectl exec -it fully-privileged-pod -- sh
/ # 
/ # echo 0 > /proc/sys/net/ipv4/ip_forward
/ #  
/ # cat /proc/sys/net/ipv4/ip_forward
0
~~~

#### k8s 设置 capabilities

~~~yaml
# basic-capabilities.yaml
apiVersion: v1
kind: Pod
metadata:
  name: capabilities-demo
spec:
  containers:
  - name: demo
    image: busybox:latest
    command: ["tail", "-f", "/dev/null"]
    securityContext:
      capabilities:
        # 删除所有默认 capabilities
        drop: ["ALL"]
        # 添加需要的 capabilities
        add: ["NET_BIND_SERVICE", "NET_RAW"]
~~~







