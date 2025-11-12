# 资源清单

k8s 中有控制器和控制器管理器，控制器属于资源类型，控制器管理器使用资源类型管理资源。比如说，Deployment 就是一种资源类型，Pod 也是一种资源类型，等等。使用这些资源类型可以创建具体的资源，Controller-Manager 根据资源的类型（类型不同功能也不同）管理资源。

## 创建资源的方式

可以使用命令行的方式创建，也可以使用 yaml 文件。

命令行的方式简单方便，但是 yaml 文件的方式可以保存配置资源的方式，便于维护。推荐使用 yaml 的方式。



#### 命令行方式创建资源

~~~bash
kubectl create deployment nginx-deployment --image=nginx:1.14 --replicas=3
~~~

命令解释

- `kubectl create deployment`：创建 Deployment 的命令。
- `nginx-deployment`：为你创建的 Deployment 资源指定的名称。
- `--image=nginx`：指定要使用的容器镜像，这里是从 Docker Hub 拉取最新的官方 Nginx 镜像。
- `--replicas=3`：指定这个 Deployment 要创建并维持 3 个完全相同的 Pod 副本。

#### yaml 文件创建资源（推荐）

~~~bash
kubectl apply -f nginx.yaml
~~~

提前编写 yaml

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
        image: nginx:1.14
~~~



使用 yaml 文件编写资源配置，就是按照固定格式，为了创建资源提供必要的参数。相当于在这种资源类型的表中插入一条数据记录，ControllerManager 监听到数据变动就会执行创建资源的代码，最终创建了具体资源。



## 配置清单格式

大部分资源的配置清单格式都由 5 个一级字段组成：

~~~bash
apiVersion: group/version　　指明api资源属于哪个群组和版本，同一个组可以有多个版本
        $ kubectl api-versions

kind: 资源类别，标记创建的资源类型，k8s主要支持以下资源类别
        Pod,ReplicaSet,Deployment,StatefulSet,DaemonSet,Job,Cronjob
    
metadata:元数据，主要提供以下字段
        name：同一类别下的name是唯一的
        namespace：对应的对象属于哪个名称空间
        labels：标签，每一个资源都可以有标签，标签是一种键值数据
        annotations：资源注解，与label不同的地方在于，annotations不能用于挑选资源对象，仅用于为对象提供"元数据"，没有键值长度限制。
        
        每个的资源引用方式(selflink)：
            /api/GROUP/VERSION/namespace/NAMESPACE/TYPE/NAME
    
spec: 定义目标资源的期望状态（disired state），资源对象中最重要的字段
    
status: 显示资源的当前状态（current state），本字段由kubernetes进行维护
~~~



## 创建配置模板

可以使用命令快速创建配置模板，主要是借助 `--dry-run=client` 的功能。

#### 示例：生成 nginx 的 deployment 的 yaml 模板

~~~bash
[root@k8s-master-01 ~]# kubectl create deployment nginx-deploy --image=nginx:latest --dry-run=client -o yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  creationTimestamp: null
  labels:
    app: nginx-deploy
  name: nginx-deploy
spec:
  replicas: 1
  selector:
    matchLabels:
      app: nginx-deploy
  strategy: {}
  template:
    metadata:
      creationTimestamp: null
      labels:
        app: nginx-deploy
    spec:
      containers:
      - image: nginx:latest
        name: nginx
        resources: {}
status: {}
~~~

#### 示例：生成 nginx 的 service 的 yaml 模板

~~~bash
[root@k8s-master-01 ~]# kubectl create service clusterip nginx-service --tcp=8080:80 --dry-run=client -o yaml
apiVersion: v1
kind: Service
metadata:
  creationTimestamp: null
  labels:
    app: nginx-service
  name: nginx-service
spec:
  ports:
  - name: 8080-80
    port: 8080
    protocol: TCP
    targetPort: 80
  selector:
    app: nginx-service
  type: ClusterIP
status:
  loadBalancer: {}
~~~



## 查看配置字段解析

K8s 存在内嵌的格式说明，可以使用 `kubectl explain` 进行查看，如查看 Pod 这个资源的定义：

~~~bash
kubectl explain pods.spec.containers
~~~

说明该字段可以存在多个二级字段，那么可以使用如下命令继续查看二级字段的定义方式：

```bash
kubectl explain pods.metadata
kubectl explain pods.spec
kubectl explain pods.spec.containers
kubectl explain pods.spec.containers.livenessProbe 
# 只要参数是object就可以点下去看详细参数
```



## spec 的 containers 必需字段解析

imagePullPolicy 镜像拉取策略

- imagePullPolicy：本地不存在时才拉取（默认值）
- Always：每次创建 Pod 时都会从镜像仓库拉取镜像
- Never：只使用本地已有的镜像，如果不存在则失败



#### 拉取私有仓库镜像使用的 secret

需要使用 `imagePullSecrets` 配置 secret， 一般使用基于文件制作 secret ，方便自动化运维。制作流程如下：

1. 登录私有仓库

~~~bash
nerdctl login --username=<your-username> crpi-hdcg863vh2ayst45.cn-shanghai.personal.cr.aliyuncs.com
~~~

2. 登录成功后生成登录配置文件

~~~bash
[root@k8s-master-01 ~]# cat /root/.docker/config.json 
{
	"auths": {
		"crpi-hdcg863vh2ayst45.cn-shanghai.personal.cr.aliyuncs.com": {
			"auth": "bGl1...JCVe"
		}
	}
}
~~~

3. 然后基于配置文件创建 secret

~~~bash
kubectl create secret generic my-registry-secret \
  --from-file=.dockerconfigjson=/root/.docker/config.json \
  --type=kubernetes.io/dockerconfigjson
~~~

4. 查看 secret

~~~bash
[root@k8s-master-01 ~]# kubectl get secrets | grep my-registry-secret
my-registry-secret   kubernetes.io/dockerconfigjson   1      33s
~~~



详细版 spec 的 containers 必需字段解析

~~~yaml
apiVersion: v1 # 必选，指定api接口资源版本
kind: Pod    # 必选，定义资源接口类型/角色。pod为容器资源
metadata:    # 必选，定义资源的元数据信息
  name: nginx    # 必选，定义资源名称，在同一个namespace中必须是唯一的
  namespace: web-testing # 可选，不指定默认为default，指定资源所在的命名空间
  labels:    # 可选，定义资源标签
    - app: nginx
  annotations:    # 可选，注释列表
    - app: nginx
spec:    # 必选，用于定义容器的详细信息
  containers:    # 必选，容器列表
  - name: nginx    # 必选，符合RFC 1035规范的容器名称
    image: nginx:v1 # 必选，容器所用的镜像的地址
    imagePullPolicy: Always   # 可选，镜像拉取策略
    workingDir: /usr/share/nginx/html    # 可选，容器的工作目录
    volumeMounts:    # 可选，存储卷配置
    - name: webroot # 存储卷名称
      mountPath: /usr/share/nginx/html # 挂载目录
      readOnly: true    # 只读
    ports:    # 可选，容器需要暴露的端口号列表
    - name: http    # 端口名称
      containerPort: 80    # 端口号
      protocol: TCP    # 端口协议，默认TCP
    env:    # 可选，环境变量配置
    - name: TZ    # 变量名
      value: Asia/Shanghai #变量
    - name: LANG
      value: en_US.utf8
    resources:    # 可选，资源限制和资源请求限制
      limits:    # 最大限制设置
        cpu: 1000m
        memory: 1024MiB  # MiB 计算使用 1024
      requests:    # 启动所需的资源
        cpu: 100m  # 1m 表示 1000分之1个核，1000m 表示使用一个 cpu
        memory: 512MB # 计算使用 1000
    readinessProbe: # 可选，容器状态检查
      httpGet:    # 检测方式
        path: /    # 检查路径
        port: 80    # 监控端口
      timeoutSeconds: 2    # 超时时间 
      initialDelaySeconds: 60    # 初始化时间
    livenessProbe:    # 可选，监控状态检查
      exec:    # 检测方式
        command: 
        - cat
        - /health
      httpGet:    # 检测方式
        path: /_health
        port: 8080
        httpHeaders:
        - name: end-user
          value: jason
      tcpSocket:    # 检测方式
        port: 80
      initialDelaySeconds: 60    # 初始化时间
      timeoutSeconds: 2    # 超时时间
      periodSeconds: 5    # 检测间隔
      successThreshold: 2 # 检查成功为2次表示就绪
      failureThreshold: 1 # 检测失败1次表示未就绪
    securityContext:    # 可选，限制容器不可信的行为
      provoleged: false
  restartPolicy: Always    # 可选，默认为Always
  nodeSelector:    # 可选，指定Node节点
    region: subnet7
  imagePullSecrets:    # 可选，拉取镜像使用的secret
  - name: default-dockercfg-86258
  hostNetwork: false    # 可选，是否为主机模式，如是，会占用主机端口
  volumes:    # 共享存储卷列表
  - name: webroot # 名称，与上述对应
    emptyDir: {}    # 共享卷类型，空
    hostPath:        # 共享卷类型，本机目录
      path: /etc/hosts
    secret:    # 共享卷类型，secret模式，一般用于密码
      secretName: default-token-tf2jp # 名称
      defaultMode: 420 # 权限
      configMap:    # 一般用于配置文件
      name: nginx-conf
      defaultMode: 420
~~~

