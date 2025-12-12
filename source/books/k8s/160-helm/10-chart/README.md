# Chart 包结构

Chart 包必须要有的文件或目录

~~~bash
mychart/
├── Chart.yaml # 包含当前 chart 信息的 YAML 文件，必须要有该文件
├── values.yaml # 当前 chart 的默认配置 values
└── templates/ # 包含模版信息
    ├── deployment.yaml
    ├── configmap.yaml
    └── service.yaml
~~~

chart 包结构构成如下。chart 包下一些文件名都是保留、固定给helm用的，例如chats、crds、templates等，自己创建的文件夹不要用这种名字。

~~~bash
wordpress/
  Chart.yaml # 包含当前 chart 信息的 YAML 文件
  LICENSE # 可选：是一个包含软件许可协议的文件，它规定了软件的使用、复制、修改和
分发的条款。
  README.md # 可选：一个可读性高的 README 文件
  charts/ # 包含该 chart 依赖的所有 chart 的目录
  crds/ # Custom Resource Definitions
  templates/ # 模板目录，与 values 结合使用时，将渲染生成 Kubernetes 资源清
单文件
  templates/NOTES.txt # 可选: 用于在 Chart 安装后提供一些重要的说明或指导信息
  templates/_helpers.tpl # 因为是`_`开头的，所以不会被当资源清单解析，但可以为templates内
的其他清单文件提供全局的命名
  values.yaml # 当前 chart 的默认配置 values
  values.schema.json # 可选: 一个作用在 values.yaml 文件上的 JSON 模式
~~~



对于一个 chart 包来说 Chart.yaml 文件是必须的。这个文件中有一个非常重要的字段，type，用来定义 chart 包的类型。

目前，Helm 支持的两个主要 Chart 类型类型是： 

- application: 默认类型，用于部署具体应用（例如，一个 Web 服务，数据库等）。 
- library: 库类型，用于共享通用模板和定义，这些 chart 通常不会被单独部署，而是作为其他 chart 的依赖项被引用。

>如果你的 chart 是一个完整的应用程序或服务，那么默认情况下不需要指定 type 字段，因为 application 是默认值。 
>
>如果你正在创建一个共享的库 chart，你应该显式指定 type: library 来标识它。



## chart包所依赖的chart

在 Helm 中，一个 chart 包可能会依赖许多其他的 chart。这些依赖关系可以在两个地方指定

- 使用 Chart.yaml 中的依赖关系字段动态链接
- 也可以引入到 charts/ 目录手动进行管理。

使用 dependencies 字段管理依赖

当前 chart 所需的依赖 chart 需要在 dependencies 字段中进行定义，如下所示

~~~yaml
dependencies:
- name: apache # 所依赖的 chart 的名称
  version: 1.2.3 # 所依赖的 chart 版本
  repository: https://example.com/charts 
  # chart 仓库的完整 URL，不过需要注意，
  # 必须使用 helm repo add 在本地添加该repo
- name: mysql
  version: 3.2.1
  repository: https://another.example.com/charts
~~~

定义了依赖项后，可以运行 `helm dependency update` 来更新依赖项，它将根据你的依赖项文件把你 所有指定的 chart 包下载到 charts/ 目录中。该命令可以简写为 `helm dep update `

对于上面的示例，我们可以在 charts 目录中看到如下的文件

~~~bash
charts/
  apache-1.2.3.tgz
  mysql-3.2.1.tgz
~~~

某些场景下，你需要复用同一个 chart 包多次，此时就需要用到alias字段了，即我们可以通过 Helm 的 alias 字段来实现复用同一个 chart。



如果一个 chart 包有其他子 chart 包依赖，使用前需要有如下操作

~~~bash
helm dep update 
# 必须要执行该命令，
# 会把dependencies.repository指定的依赖chart下载到当前chart包下的charts目录下，是一个压缩包

helm template myapp .
# 这个命令可以查看渲染好的所有的 yaml 文件
~~~



####  Values 文件

chart包下的values.yaml文件用于为templates目录下的模版提供一些必须的value值.

用户也可以自定义包含value值的yaml，然后 helm install -f 指定。你my_values.yaml的内容会与chart包自带的values.yaml合并到一起，当然了遇到 冲突字段，你的my_values.yaml优先级更高。

~~~bash
helm template -f /tmp/my_values.yaml mysql # mysql是你的chart包目录
~~~

#### value值的作用域访问

如果子 chart 声明了全局变量，则该全局变量将向下（传递到子 chart 的子 chart 中）传递，而不会向 上传递到父 chart，子 chart 无法影响 父 chart的值。同样，父 chart 的全局变量优先与子 chart 中的全 局变量。

~~~yaml
# wordpress/value.yaml

title: "My WordPress Site" # 传递到 WordPress 模板
global:
  app: MyWordPress
mysql:
  max_connections: 100 # 传递到 MySQL
  password: "secret"
apache:
  port: 8080 # 传递到 Apache
~~~

上面我们添加了一个全局范围的 value 值： app: MyWordPress ，该值可以通过 .Values.global.app 提供给所有 chart 使用。

mysql 模板可以以 {{ .Values.global.app }} 来访问 app



## 使用 Helm 管理 Charts

helm 工具有几个用于操作 charts 的命令，如下所示。 

创建一个新的 chart 包

~~~bash
helm create mychart
~~~

旦你已经编辑了一个 chart 包，Helm 可以将其打包到一个独立文件中

~~~bash
➜ helm package mychart
Archived mychart-0.1.-.tgz
~~~

~~~bash
运行 helm package mychart 命令时，Helm 会将指定的 Helm chart（在此例中是名为 mychart 的
目录）打包成一个 .tgz 格式的 Helm 包文件。这是将 Chart 准备好以便分发和部署的重要步骤。具体来
说，helm package mychart 主要做了以下几件事情：
# 1、验证 Chart 结构：
检查 mychart 目录是否包含必需的文件和目录，诸如 Chart.yaml、values.yaml、templates 目录
等。
确保 Chart.yaml 文件中定义的 chart 名称、版本等信息是有效的。
# 2、处理依赖关系：
如果 mychart 中的 Chart.yaml 定义了依赖关系（例如包含 dependencies 字段），Helm 会检查这
些依赖是否已经满足。如果依赖的 chart 包不在本地，将操作失败并提示错误，用户需要先运行 helm
dependency update 来获取这些依赖。
# 3、创建包文件：
将 mychart 目录及其所有内容（包括 values.yaml、templates 等）打包成一个 .tgz 文件。这个文
件名由 Chart.yaml 中定义的 chart 名称和版本决定，格式为 mychart-<版本号>.tgz。
~~~



## Chart 仓库

#### 使用 nginx 部署只读仓库

chart 仓库服务端准备

1. 启动一个http服务,监听8888端口
2. 仓库的主要特征是存在一个名为 index.yaml 的特殊文件，该文件具有仓库中提供的所有软件包的 列表以及允许检索和验证这些软件包的元数据。

~~~bash
# 1、创建chart包
cd /usr/share/nginx/html
helm create mychart
# 2、打包chart
helm package mychart
# 3、当前目录下生成一个 index.yaml 文件，每新增一个chart都要重新执行下条命令来生成
index.yaml
helm repo index . --url http://192.168.71.12:8888
helm repo index . --url http://192.168.71.113
~~~

客户端准备

添加私有仓库

~~~bash
helm repo add myprivaterepo http://192.168.71.12:8888
~~~

使用

~~~bash
# 1、更新本地仓库缓存，你的仓库内容变动过，那你就需要更新一下才能看到
helm repo update
# 你也可以curl http://192.168.71.12:8888/index.yaml验证一下目标仓库的该文件是否真的更新
了
# 2、检索仓库
helm search repo myprivaterepo
# 3、安装
helm install myrelease myprivaterepo/mychart
~~~



####  使用 harbor 部署

Harbor 是一个主流的镜像仓库系统，在 v1.6 版本以后的 harbor 中新增加了 helm charts 的管理功能，可以存储Chart文件。 

其实在Harbor 2.8+版本中，Helm Chart支持已经转移到了OCI（Open Container Initiative）格式。这意味着你需要使用 OCI 形式来上传和管理你的Helm Chart（不需要像网上一样，去为harbor开启chart仓 库支持）

0、安装一个nfs存储，提供一个sc默认存储类

~~~bash
# 1、安装
helm repo add nfs-subdir-external-provisioner https://kubernetessigs.github.io/nfs-subdir-external-provisioner/
helm upgrade --install nfs-subdir-external-provisioner nfs-subdir-externalprovisioner/nfs-subdir-external-provisioner --set nfs.server=192.168.71.12 --set
nfs.path=/data/nfs --set storageClass.defaultClass=true -n kube-system
# 2、查看
helm -n kube-system list
# 3、查看nfs_provider的pod
kubectl -n kube-system get pods |grep nfs
nfs-subdir-external-provisioner-849d86787-v96bk 1/1 Running 0
2m19s
# 4、查看sc（已经设置为默认的了）
kubectl -n kube-system get sc nfs-client
NAME PROVISIONER RECLAIMPOLICY VOLUMEBINDINGMODE ALLOWVOLUMEEXPANSION
nfs-client (default) cluster.local/nfs-subdir-external-provisioner Delete
~~~

添加仓库地址

~~~bash
helm repo add harbor https://helm.goharbor.io
helm repo list
~~~

下载Chart包到本地

因为需要修改的参数比较多，在命令行直接helm install比较复杂，我就将Chart包下载到本地，再修改 一些配置，这样比较直观，也比较符合实际工作中的业务环境。

~~~bash
helm pull harbor/harbor # 下载Chart包
tar zxvf harbor-1.14.2.tgz # 解压包
~~~

修改values.yaml

~~~bash
$ cd harbor
$ ls
Chart.yaml LICENSE README.md templates values.yaml
$ vim values.yaml
expose:
type: nodePort # 我这没有Ingress环境，使用NodePort的服务访问方式。
tls:
enabled: false # 关闭tls安全加密认证（如果开启需要配置证书）
...
# 注意用http协议
externalURL: http://192.168.71.12:30002 # 使用nodePort且关闭tls认证，则此处需要修改为
# http协议和
expose.nodePort.ports.http.nodePort
# 指定的端口号，IP即为kubernetes的节点IP地
址
# 持久化存储配置部分
persistence:
enabled: true # 开启持久化存储
resourcePolicy: "keep"
persistentVolumeClaim: # 定义Harbor各个组件的PVC持久卷部分
registry: # registry组件（持久卷）配置部分
existingClaim: ""
storageClass: "nfs-client" # 前面创建的StorageClass，其它组件同样配置
subPath: ""
accessMode: ReadWriteMany # 卷的访问模式，需要修改为ReadWriteMany，允许
多个组件读写，否则有的组件无法读取其它组件的数据
size: 5Gi
#chartmuseum: # chartmuseum组件（持久卷）配置部分，v2.8+ habor版本后不需要再启用
它了
# existingClaim: ""
# storageClass: "nfs-client"
# subPath: ""
# accessMode: ReadWriteMany
# size: 5Gi
jobservice: # 异步任务组件（持久卷）配置部分
existingClaim: ""
storageClass: "nfs-client" #修改，同上
subPath: ""
accessMode: ReadWriteMany
size: 1Gi
database: # PostgreSQl数据库组件（持久卷）配置部分
existingClaim: ""
storageClass: "nfs-client"
subPath: ""
accessMode: ReadWriteMany
size: 1Gi
redis: # Redis缓存组件（持久卷）配置部分
existingClaim: ""
storageClass: "nfs-client"
subPath: ""
accessMode: ReadWriteMany
size: 1Gi
trivy: # Trity漏洞扫描插件（持久卷）配置部分
existingClaim: ""
storageClass: "harbor-storageclass"
subPath: ""
accessMode: ReadWriteMany
size: 5Gi
...
harborAdminPassword: "Harbor12345" # admin初始密码，不需要修改
...
metrics:
enabled: false # 是否启用监控组件（可以使用Prometheus监控Harbor指标），非必须操作，可以
设为false
core:
path: /metrics
port: 8001
registry:
path: /metrics
port: 8001
jobservice:
path: /metrics
port: 8001
exporter:
path: /metrics
port: 8001
###以下的trace为2.4版本的功能，不需要修改
~~~

安装

~~~bash
kubectl create namespace harbor
helm install harbor . -n harbor # 将安装资源部署到harbor命名空间
# 注意
# 1、部署过程可能因为下载镜像慢导致redis尚未启动成功，其他pod会出现启动失败的现象，耐心等一会即
可
# 2、如果下载速度过慢，可以自己制作镜像，或者下载镜像后上传到服务器导入
# nerdctl -n k8s.io load -i xxxxxxxxxxx.tar
~~~

登录http://192.168.71.12:30002，账号：admin 密码：Harbor12345



#### 把harbor当chart库用

~~~bash
# 1、登录
helm registry login -u admin -p Harbor12345 --insecure 192.168.71.12:30002
# 2、本地打包
helm create zzz
helm package zzz # 得到的包格式固定位：zzz-0.1.0.tgz
# 3、push上传：注意包的格式中就包含了push之后的版本信息
helm push zzz-0.1.0.tgz --plain-http oci://192.168.71.12:30002/chart-public
# 4、pull下载
helm pull --plain-http oci://192.168.71.12:30002/chart-public/zzz --version
0.1.0
# 5、你可以重复步骤2，然在package之后修改chart.yaml中的version版本，然后打包后push
$ helm package zzz
Successfully packaged chart and saved it to: /test1/zzz-0.1.1.tgz
helm push zzz-0.1.1.tgz --plain-http oci://192.168.71.12:30002/chart-public
# 6、安装
helm install zzz-release1 --plain-http oci://192.168.71.12:30002/chartpublic/zzz --version 0.1.0 # 版本要与你上传的对应上
# 查看
$ kubectl get pods zzz-release1-55bc676d6d-6hk9d -w
NAME READY STATUS RESTARTS AGE
zzz-release1-55bc676d6d-6hk9d 1/1 Running 0 115s
~~~



#### 把harbor当镜像仓库用

~~~bash
nerdctl login -u admin -p Harbor12345 --insecure-registry
http://192.168.71.12:30002
nerdctl -n k8s.io tag registry.cnhangzhou.aliyuncs.com/google_containers/pause:3.9
192.168.71.12:30002/library/pause:3.9
nerdctl -n k8s.io push --insecure-registry 192.168.71.12:30002/library/pause:3.9
~~~

