# configmap 和 secret

configmap 和 secret 是两种配置资源，以挂载卷或者环境变量的方式为 pod 提供配置数据，一个 configmap 或者 secret 资源可以给多个 pod 使用。configmap 与 secret 类似，但通常 configmap 用来存放不敏感的字符串。

~~~alert type=note
1. 只有通过 k8s api 创建的 Pod 才可以使用configmap 和 secret，其他方式创建的 pod(比如静态 pod)不能使用。<br>
2. configmap 文件大小限制为 1MB。<br>
3. configmap 更新后，关联的 pod 内能够同步更新，但是 POD 内的进程不会重新。
~~~



## 创建 configmap

有两种创建 configmap 的方式，分别是 命令行创建和 yaml 配置清单创建。

#### 使用 yaml 创建configmap

配置数据在 data 下，key:value形式，value 值需要是字符串、非字符串的话请加上引号。

~~~yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: test1-config
  namespace: default
data:
  xxx: "111"
  yyy: "222"
  data.1: hello
  data_2: world
  lines: |
    我是第一行
      我是第二行
    我是第三行
  value1: |-
    hello
    world
  value2: |+
    hello
    world
  value3: >
    我是第一行
    我也是第一行
    
    我是第二行
    我也是第二行
~~~

其中，配置数据时可以使用一些符号，比如：竖线符 `|`、加号 `+`、减号 `-`、大于号 `>`

- 竖线符 `|`: 在 yaml 中代表保留换行，但是每行的缩进和行尾空白都会被去掉，而额外的缩进会被保留。
- 竖线符搭配 `+` 或 `-` 号： `+` 表示保留文字块末尾的换行符， `-` 表示删除字符串末尾的换行符。
- 大于号 `>`： 在 yaml 中表示折叠换行，内容最末尾的换行会保留，但文中部分只有空白行才会被识别 为换行，原来的换行符都会被转换成空格。



使用 json 的格式输出配置数据

~~~bash
[root@k8s-master-01 config]# kubectl get cm test1-config -o json
{
    "apiVersion": "v1",
    "data": {
        "data.1": "hello",
        "data_2": "world",
        "lines": "我是第一行\n  我是第二行\n我是第三行\n",
        "value1": "hello\nworld",
        "value2": "hello\nworld\n",
        "value3": "我是第一行 我也是第一行\n我是第二行 我也是第二行\n",
        "xxx": "111",
        "yyy": "222"
    },
    "kind": "ConfigMap",
    "metadata": {
        "annotations": {
            "kubectl.kubernetes.io/last-applied-configuration": "{\"apiVersion\":\"v1\",\"data\":{\"data.1\":\"hello\",\"data_2\":\"world\",\"lines\":\"我是第一行\\n  我是第二行\\n我是第三行\\n\",\"value1\":\"hello\\nworld\",\"value2\":\"hello\\nworld\\n\",\"value3\":\"我是第一行 我也是第一行\\n我是第二行 我也是第二行\\n\",\"xxx\":\"111\",\"yyy\":\"222\"},\"kind\":\"ConfigMap\",\"metadata\":{\"annotations\":{},\"name\":\"test1-config\",\"namespace\":\"default\"}}\n"
        },
        "creationTimestamp": "2025-12-01T06:26:56Z",
        "name": "test1-config",
        "namespace": "default",
        "resourceVersion": "1704473",
        "uid": "5d38dd4f-88f9-4c74-98a1-1cdc51a473e1"
    }
}
~~~

#### 命令行创建 configmap

基于文件夹创建，使用 `--from-file` 。文件名作为 key，文件内容为 value

~~~bash
[root@k8s-master-01 myconfig]# pwd
/config/myconfig
[root@k8s-master-01 myconfig]# cat mysql 
host=127.0.0.1
port=3306
[root@k8s-master-01 myconfig]# cat redis 
host=127.0.0.1
port=6379[root@k8s-master-01 myconfig]# kubectl create configmap my-config --from-file=/config/myconfig
configmap/my-config created
[root@k8s-master-01 myconfig]# kubectl get cm my-config -o yaml
apiVersion: v1
data:
  mysql: |
    host=127.0.0.1
    port=3306
  redis: |
    host=127.0.0.1
    port=6379
kind: ConfigMap
metadata:
  creationTimestamp: "2025-12-01T06:44:21Z"
  name: my-config
  namespace: default
  resourceVersion: "1705432"
  uid: b08ba1bd-9b53-4f6b-bd12-eb83c9a7a2c5
~~~

基于文件创建，使用 `--from-file` 。key 可以自定义。`--from-file` 参数可以使用多次。

~~~bash
[root@k8s-master-01 myconfig]# kubectl create configmap my-config --from-file=key1=/config/myconfig/redis
configmap/my-config created
[root@k8s-master-01 myconfig]# kubectl get cm my-config -o yaml
apiVersion: v1
data:
  key1: |
    host=127.0.0.1
    port=6379
kind: ConfigMap
metadata:
  creationTimestamp: "2025-12-01T06:48:55Z"
  name: my-config
  namespace: default
  resourceVersion: "1705846"
  uid: 951ec7d0-bb97-4a98-b944-ac155ce55bb8
~~~

基于值创建，使用 `--from-literal`，也可以用多次。

~~~bash
[root@k8s-master-01 myconfig]# kubectl create configmap my-config --from-literal=name=kaxonliu --from-literal=pawd=12345
configmap/my-config created
[root@k8s-master-01 myconfig]# kubectl get cm my-config -o yaml
apiVersion: v1
data:
  name: kaxonliu
  pawd: "12345"
kind: ConfigMap
metadata:
  creationTimestamp: "2025-12-01T06:53:28Z"
  name: my-config
  namespace: default
  resourceVersion: "1706257"
  uid: 1f52f984-80d7-43d0-abe5-1e701f9bf25d
~~~

基于 env 文件创建，使用 `--from-env-file` 指定文件，可以使用多次。

~~~bash
[root@k8s-master-01 myconfig]# cat abc.env 
DB_PORT=3306
DB_HOST=localhost
[root@k8s-master-01 myconfig]# kubectl create configmap my-config --from-env-file=/config/myconfig/abc.env 
configmap/my-config created
[root@k8s-master-01 myconfig]# kubectl get cm my-config -o yaml
apiVersion: v1
data:
  DB_HOST: localhost
  DB_PORT: "3306"
kind: ConfigMap
metadata:
  creationTimestamp: "2025-12-01T06:56:59Z"
  name: my-config
  namespace: default
  resourceVersion: "1706575"
  uid: 4bb2bc20-890e-4240-8df3-1b4e3bd794bc
~~~



## 使用 configmap 的三种方式

在 pod 中可以按照如下三种方式使用 configmap 的配置数据。

#### 设置环境变量

~~~yaml
apiVersion: v1
kind: Pod
metadata:
  name: testcm1-pod
spec:
  containers:
  - name: testcm1
    image: nginx
    command: [ "/bin/sh", "-c", "tail -f /dev/null" ]
    env:
    - name: DB_HOST
      valueFrom:
        configMapKeyRef:
          name: my-config
          key: DB_HOST
    - name: DB_PORT
      valueFrom:
        configMapKeyRef:
          name: my-config
          key: DB_PORT
~~~

#### 在容器的命令里引入变量

~~~yaml
apiVersion: v1
kind: Pod
metadata:
  name: testcm2-pod
spec:
  containers:
  - name: testcm2
    image: nginx
    command: [ "/bin/sh", "-c", "echo $(DB_HOST) $(DB_PORT)" ]
    env:
    - name: DB_HOST
      valueFrom:
        configMapKeyRef:
          name: my-config
          key: DB_HOST
    - name: DB_PORT
      valueFrom:
        configMapKeyRef:
          name: my-config
          key: DB_PORT
~~~

查看

~~~bash
[root@k8s-master-01 config]# kubectl logs -f testcm2-pod 
localhost 3306
~~~

#### 数据卷挂载配置文件

准备 cm

~~~bash
[root@k8s-master-01 config]# kubectl get cm my-config -o yaml
apiVersion: v1
data:
  mysql: |
    host=127.0.0.1
    port=3306
kind: ConfigMap
metadata:
  ...
~~~

以挂在卷的方式使用 cm

~~~yaml
apiVersion: v1
kind: Pod
metadata:
  name: testcm3-pod
spec:
  # 声明数据卷
  volumes:
  - name: my-volume
    configMap:
      name: my-config
  containers:
  - name: testcm3
    image: nginx
    command: [ "/bin/sh", "-c", "cat /etc/config/mysql" ]
    # pod 中使用 数据卷
    volumeMounts:
    - name: my-volume
      mountPath: /etc/config
~~~

指定挂载的子路径

~~~yaml
apiVersion: v1
kind: Pod
metadata:
  name: testcm4-pod
spec:
  volumes:
  - name: my-volume
    configMap:
      name: test2-config
      items:
      - key: mysql.conf
        path: path1/to/msyql.conf
      - key: redis.conf
        path: path2/to/redis.conf
  containers:
  - name: testcm4
    image: busybox
    command: [ "/bin/sh","-c","cat /etc/config/path1/to/msyql.conf ;sleep 10
;cat /etc/config/path2/to/redis.conf" ]
    volumeMounts:
    - name: my-volume
      mountPath: /etc/config
~~~



## configmap 热更新

在 Kubernetes 中，当你对 ConfigMap 进行更新或删除重建时，如果 ConfigMap 被挂载为 Volume 到 了 Pod 内，那么这些更新是可以被反映到 Pod 文件系统上的文件内容中的，这也就是我们所说的"热更 新"。

虽然文件内容确实会被更新，但这并不意味着运行在 Pod 中的应用会自动知道这些更改。事实上，很多的应用在启动时只会读取一次配置文件，而在运行时不会再对配置文件做任何检查。这就意味着，即使配置文件的内容被更新了，这些应用也不会感知到这种变化，必须重启应用才能加载新的配置。

 所以，虽然 Kubernetes 提供了一种机制可以让 ConfigMap 的更新被反映到 Pod 文件系统中，但是要让应用感知到这些更改，通常还需要应用本身支持重新加载配置，或者你需要额外地重启应用来加载新的配置。

 这时可以增加一些监测配置文件变更的脚本，然后重加载对应服务就可以实现应用的热更新。

## 实现应用热更新

实现思路很简单，部署一个 sidecar，监控 configmap 的变化，一旦发生变化就重启相关的 POD。

#### 部署 sidecar

~~~bash
wget --no-check-certificate https://raw.githubusercontent.com/stakater/Reloader/master/deployments/kubernetes/reloader.yaml
~~~

下载后部署

~~~bash
kubectl apply -f reloader.yaml
~~~

准备 配置文件

~~~bash
[root@k8s-master-01 config]# cat /config/myconfig/nginx.conf 
server {
    listen   8080;
    root /usr/share/nginx/html;
}
~~~

创建 configmap

~~~bash
# 创建 cm
kubectl create configmap my-config --from-file=/config/myconfig/nginx.conf
# 查看
[root@k8s-master-01 config]# kubectl get cm my-config -o yaml
apiVersion: v1
data:
  nginx.conf: |
    server {
        listen   8080;
        root /usr/share/nginx/html;
    }
kind: ConfigMap
metadata:
  ...
~~~

如果某 deployment 需要随着 configmap 的更新而自动重启 pods 只需要添加注释 reloader.stakater.com/auto: "true" 即可。

~~~yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
  # 添加监控
  annotations:  
    reloader.stakater.com/auto: "true"
  labels:
    app: nginx
spec:
  replicas: 1
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      volumes:
      - name: my-volume
        configMap:
          name: my-config
      containers:
      - name: nginx
        image: nginx:latest
        volumeMounts:
        - name: my-volume
          mountPath: /etc/nginx/conf.d
~~~

一旦 cm发生变化，pod 就会自动重新，使用最新的 cm 配置。



## secret

Secret意为秘密，那在K8S中是啥意思呢？在K8S中表示一个存储在etcd中的配置，这个配置是秘密的， 是安全的，通常用Base64编码，此配置可以通过挂载卷或者环境变量的方式供Pod访问。

Secret对象的数据存储和打印格式为Base64编码的字符串，因此用户在创建Secret对象时，也需要提供 该类型的编码格式的数据。在容器中以环境变量或存储卷的方式访问时，会自动解码为明文格式。需要 注意的是，如果是在Master节点上，Secret对象以非加密的格式存储在etcd中，所以需要对etcd的管理 和权限进行严格控制。



#### Secret有多种类型

- Opaque ：base64编码格式的Secret，用来存储账号密码；
-  kubernetes.io/tls：存放证书以及私钥 
- kubernetes.io/dockerconfigjson ：用来存储私有docker registry的认证信息，类型标识为 docker-registry。



## 使用 secret (Opaque类型)

Secret 资源提供了2个可用的键值对： data 和 stringData，data 和 stringData 的键必须由字母、数字、 - ， _ 或 . 组成。

- data 字段的数据必须是 base64 编码的任意数据 
- stringData 字段允许 Secret 使用未编码的字符串



#### 使用创建 data 字段

~~~yaml
apiVersion: v1
kind: Secret
metadata:
  name: test-secret
data:
  username: a2F4b25saXU=
  password: MTIzNDU2
~~~

- 需要提前把明文手动转为密文

>~~~bash
>kaxonliu[root@k8s-master-01 config]# echo -n "kaxonliu" | base64
>a2F4b25saXU=
># 反解 base64
>[root@k8s-master-01 config]# echo -n "a2F4b25saXU=" | base64 -d
>kaxonliu[root@k8s-master-01 config]# 
>~~~

#### 使用 stringData 字段

某些场景下想将非base64编码的字符串放入Secret中，就需要用到stringData字段

~~~yaml
apiVersion: v1
kind: Secret
metadata:
  name: my-secret
stringData:
  username: kaxonliu
  password: "123456"
~~~



#### 数据卷的方式使用 secret

创建好 Secret 对象后，有两种方式来使用它：环境变量或者数据卷的方式。

~~~yaml
apiVersion: v1
kind: Pod
metadata:
  name: testsecret-pod
spec:
  # 声明数据卷
  volumes:
  - name: my-volume
    secret:
      secretName: my-secret
  containers:
  - name: testcm3
    image: nginx
    # 挂载到容器内，挂载为一个目录
    volumeMounts:
    - name: my-volume
      mountPath: /etc/my-secret
~~~

创建后到 pod 内查看

~~~bash
root@testsecret-pod:/# cd /etc/my-secret/
root@testsecret-pod:/etc/my-secret# ls
password  username
root@testsecret-pod:/etc/my-secret# cat password 
123456root@testsecret-pod:/etc/my-secret# cat username 
kaxonliuroot@testsecret-pod:/etc/my-secret# 
~~~

热更新

与configmap一样，这只是配置层面的热更新，想让pod随之一起更新，需要在部署完reloader后在控制器的metadata里添加

~~~yaml
metadata:
  annotations:
    reloader.stakater.com/auto: "true" # 注意添加这一行
~~~

#### 环境变量的方式使用 secret

环境变量的方式，不支持热更新。

~~~yaml
apiVersion: v1
kind: Pod
metadata:
  name: testsecret-pod
spec:
  containers:
  - name: testse4
    image: nginx
    env:
    - name: USERNAME
      valueFrom:
        secretKeyRef:
          name: my-secret
          key: username
    - name: PASSWORD
      valueFrom:
        secretKeyRef:
          name: my-secret
          key: password
~~~

创建后查看

~~~bash
[root@k8s-master-01 config]# kubectl exec -it testsecret-pod -- env | grep USERNAME
USERNAME=kaxonliu
~~~



## 使用 secret(dockerconfigjson类型)

dockerconfigjson 类型可以用来存储 docker registy 的认证信息，如果我们在下载镜像时需要登录账号密码，则需要设置它。

~~~bash
kubectl create secret generic my-registry-secret \
--from-file=.dockerconfigjson=/root/.docker/config.json \
--type=kubernetes.io/dockerconfigjson
~~~

>登录私有仓库库会在家目录下生成 `.docker/config.json` 文件，里面存储下载镜像需要的登录认证信息。

查看 secret 信息

~~~bash
[root@k8s-master-01 ~]# kubectl get secrets 
NAME                 TYPE                             DATA   AGE
my-registry-secret   kubernetes.io/dockerconfigjson   1      5s
[root@k8s-master-01 ~]# 
[root@k8s-master-01 ~]# 
[root@k8s-master-01 ~]# kubectl describe secrets my-registry-secret 
Name:         my-registry-secret
Namespace:    default
Labels:       <none>
Annotations:  <none>

Type:  kubernetes.io/dockerconfigjson

Data
====
.dockerconfigjson:  130 bytes
[root@k8s-master-01 ~]# 
[root@k8s-master-01 ~]# 
[root@k8s-master-01 ~]# kubectl get secrets my-registry-secret -o yaml
apiVersion: v1
data:
  .dockerconfigjson: e...fQ==
kind: Secret
metadata:
  name: my-registry-secret
  ...
type: kubernetes.io/dockerconfigjson	# 类型
~~~

如果我们需要拉取私有仓库中的 Docker 镜像的话就需要使用到上面的 my-registry-secret 这个 Secret

~~~yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: test
  labels:
    app: test
spec:
  replicas: 1
  selector:
    matchLabels:
      app: test
  template:
    metadata:
      labels:
        app: test
    spec:
      containers:
      - image: nginx
        name: nginx
      imagePullSecrets:
      - name: my-registry-secret
~~~

~~~alert type=important
ImagePullSecrets 与 Secrets 不同，因为 Secrets 可以挂载到 Pod 中，但是
ImagePullSecrets 只能由 Kubelet 访问。
~~~



## 使用 secret (kubernetes.io/tls)

~~~bash
kubectl create secret tls webhook-certs \
--cert=webhook.crt \
--key=webhook.key \
--namespace=default \
--dry-run=client -o yaml | kubectl apply -f -
~~~

查看资源情况

~~~bash
[root@k8s-master-01 work]# kubectl get secrets 
NAME            TYPE                DATA   AGE
webhook-certs   kubernetes.io/tls   2      18s
[root@k8s-master-01 work]# 
[root@k8s-master-01 work]# 
[root@k8s-master-01 work]# kubectl describe secrets webhook-certs
Name:         webhook-certs
Namespace:    default
Labels:       <none>
Annotations:  <none>

Type:  kubernetes.io/tls

Data
====
tls.key:  1675 bytes
tls.crt:  1265 bytes
~~~



## kubernetes.io/service-accounttoken

ServiceAccount 是什么？ 

- 1、ServiceAccount 即服务账号，每个sa与某一个名称空间绑定，它不是给人的账号，是Pod 与集 群 apiserver 通讯的访问凭证 

- 2、ServiceAccount 只是一个名字，只是一个代号而已，集群中真正用的是该代号关联的凭证，凭 证里包含了认证相关的秘钥、证书等，在1.24.0版之前每创建一个 sa，k8s 都会自动为其创建对应 的凭据，凭据默认会被创建成 secret，而且是固定死的 secrets

ServiceAccount，它所关联的凭据在1.24.0版本前用的就是固定死的 secret，存在诸多问题的如下（所以在后续的版本中做了优化）

- 泄露的风险。
- 扩容的问题。每一个 ServiceAccount 都需要创建一个与之对应的 Secret，在大规模的应用部署下存在弹性和容 量风险。

为了解决上述问题，k8s专门为 ServiceAccount 这种账户提供了 ServiceAccount Token 投影机制即令牌 卷投影机制，用于增强 ServiceAccount 的灵活性与安全性。

~~~bash
在 Kubernetes v1.24.0 及更高版本中，创建为ServiceAccount后，不会自动为其创建secret凭证
了，你也不需要去创建
因为但凡某个pod中引用了该ServiceAccount，k8s都会自动为该sa创建一个token然后挂载到 Pod 目录
`/var/run/secrets/kubernetes.io/serviceaccount` 中
该目录下存放有如下文件，存放着凭据所包含的ca证书、sa账户关联的名称空间、会自动更新的token值
# ls /var/run/secrets/kubernetes.io/serviceaccount/
ca.crt namespace token
token值是由 kubelet 组件维护的，在令牌快要到期的时候刷新它。应用程序需要自己负责在令牌被轮换时
重新加载其内容。
~~~

自己创建 sa

~~~yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: build-robot
---
apiVersion: v1
kind: Pod
metadata:
  name: nginx
spec:
  containers:
  - image: nginx
     name: nginx
     volumeMounts:
     - mountPath: /var/run/secrets/tokens
       name: vault-token
  # 在1.24.0后，只要你引用sa，k8s就会自动为其关联一个token并挂载到默认目录下
  serviceAccountName: build-robot
  # 我们来模仿默认行为，来声明一个我们自己的token，并挂载到我们自己的目录下
  volumes:
  - name: vault-token
    projected:
      sources:
      - serviceAccountToken:
        path: vault-token
        expirationSeconds: 600 # 10分钟后会自动更新，不能小于10分钟
        audience: vault
~~~



## Secret vs ConfigMap 

#### 相同点

- key/value 的形式

- 属于某个特定的命名空间 

- 可以导出到环境变量 

- 可以通过目录/文件形式挂载 

- 通过 volume 挂载的配置信息均可热更新

#### 不同点 

- Secret 可以被 ServerAccount 关联。
- Secret 可以存储 docker register 的鉴权信息，用在 ImagePullSecret 参数中，用于拉取私有仓库的镜像。
- Secret 支持 Base64 转码。
- Secret 分为 kubernetes.io/service-account-token 、 kubernetes.io/dockerconfigjson 、Opaque 三种类型，而 Configmap 不区分类型。

同样 Secret 文件大小限制为 1MB （ETCD 的要求）；Secret 虽然采用 Base64 编码，但是我们还是可以很方便解码获取到原始信息，所以对于非常重要的数据还是需要慎重考虑，可以考虑使用 Vault 来进行加密管理。