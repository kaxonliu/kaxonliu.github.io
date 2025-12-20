# 部署 EFK

## 1. 安装 Elasticsearch 集群

**准备和规划**

先创建一个名称空间，后续日志相关组件都安装到该名称空间下

~~~bash
kubectl create ns logging
~~~

ElasticSearch 安装有最低安装要求，如果安装后 Pod 无法正常启动，请检查是否符合最低要求的配 置，要求如下：

- elasticsearch master > 2C2G，主节点，用于控制 ES 集群
- elasticsearch data 1C2G，数据节点，用于储存 ES 数据
- elasticsearch client 1C2G，负责处理用户请求，实现请求转发，负载均衡



**准备持久化存储**

为了能够持久化 Elasticsearch 的数据，需要准备一个存储，此处我们使用 NFS 类型的 StorageClass ，如 果你是线上环境建议使用 Local PV 或者 Ceph RBD。



**为 ES 准备证书文件**

~~~bash
mkdir -p /logging/elastic-certs

# 运行容器
nerdctl run --name elastic-certs -v /logging/elastic-certs:/app -it -w /app elasticsearch:7.17.3 /bin/sh

# 在容器内使用自带工具生成证书
elasticsearch-certutil ca --out /app/elastic-stack-ca.p12 --pass '' && \
elasticsearch-certutil cert --name security-master --dns security-master \
--ca /app/elastic-stack-ca.p12 --pass '' --ca-pass '' \
--out /app/elastic-certificates.p12

# 退出容器
exit

# 查看证书
[root@k8s-master-01 ~]# ls /logging/elastic-certs/
elastic-certificates.p12  elastic-stack-ca.p12

# 删除容器
nerdctl rm -f elastic-certs
~~~

**添加证书到 Kubernetes**

制作 secret

~~~bash
cd /logging/elastic-certs

kubectl create secret -n logging generic elastic-certs --from-file=elastic-certificates.p12

# 设置集群用户名密码，用户名为elastic，密码为elastic123
kubectl create secret -n logging generic elastic-auth \
--from-literal=username=elastic \
--from-literal=password=elastic123
~~~



**安装 ES 集群**

使用 helm 的方式安装

~~~bash
helm repo add elastic https://helm.elastic.co
helm repo update
~~~

ElaticSearch 安装需要安装三次，分别安装 Master、Data、Client 节点，Master 节点负责集群间的管 理工作；Data 节点负责存储数据；Client 节点负责代理 ElasticSearch Cluster 集群，负载均衡。

首先使用 helm pull 拉取 Chart 并解压：

~~~bash
cd /logging
helm pull elastic/elasticsearch --untar --version 7.17.3
cd elasticsearch
~~~

在 Chart 目录下面创建用于 Master 节点安装配置的 `values-master.yaml ` 文件：(默认自带的values.yaml不用管，我 们不用它)

~~~yaml
# values-master.yaml

## 设置集群名称
clusterName: 'elasticsearch'
## 设置节点名称
nodeGroup: 'master'

## 设置角色
roles:
  master: 'true'
  ingest: 'false'
  data: 'false'

# ============镜像配置============
## 指定镜像与镜像版本 务必换成国内的镜像源
#image: 'elasticsearch'
image: 'crpi-hdcg863vh2ayst45.cn-shanghai.personal.cr.aliyuncs.com/liuxu8677/elasticsearch'
imageTag: '7.17.3'
imagePullPolicy: 'IfNotPresent'

## 副本数: 
#replicas: 3
# 测试环境资源有限，所以设置为1吧
replicas: 1

# ============资源配置============
## JVM 配置参数
esJavaOpts: '-Xmx1g -Xms1g'
## 部署资源配置(生产环境要设置大些)
resources:
  requests:
    cpu: '2000m'
    memory: '2Gi'
  limits:
    cpu: '2000m'
    memory: '2Gi'
## 数据持久卷配置
persistence:
  enabled: true
## 存储数据大小配置
volumeClaimTemplate:
  storageClassName: nfs-client
  accessModes: ['ReadWriteOnce']
  resources:
    requests:
      storage: 5Gi

# ============安全配置============
## 设置协议，可配置为 http、https
protocol: http
## 证书挂载配置，这里我们挂入上面创建的证书
secretMounts:
  - name: elastic-certs
    secretName: elastic-certs
    path: /usr/share/elasticsearch/config/certs
    defaultMode: 0755

## 允许您在/usr/share/elasticsearch/config/中添加任何自定义配置文件,例如 elasticsearch.yml、log4j2.properties
## ElasticSearch 7.x 默认安装了 x-pack 插件，部分功能免费，这里我们配置下
## 下面注掉的部分为配置 https 证书，配置此部分还需要配置 helm 参数 protocol 值改为 https
esConfig:
  elasticsearch.yml: |
    xpack.security.enabled: true
    xpack.security.transport.ssl.enabled: true
    xpack.security.transport.ssl.verification_mode: certificate
    xpack.security.transport.ssl.keystore.path: /usr/share/elasticsearch/config/certs/elastic-certificates.p12
    xpack.security.transport.ssl.truststore.path: /usr/share/elasticsearch/config/certs/elastic-certificates.p12
    # xpack.security.http.ssl.enabled: true
    # xpack.security.http.ssl.truststore.path: /usr/share/elasticsearch/config/certs/elastic-certificates.p12
    # xpack.security.http.ssl.keystore.path: /usr/share/elasticsearch/config/certs/elastic-certificates.p12
    
## 环境变量配置，这里引入上面设置的用户名、密码 secret 文件
extraEnvs:
  - name: ELASTIC_USERNAME
    valueFrom:
      secretKeyRef:
        name: elastic-auth
        key: username
  - name: ELASTIC_PASSWORD
    valueFrom:
      secretKeyRef:
        name: elastic-auth
        key: password

# ============调度配置============
## 设置调度策略
## - hard：只有当有足够的节点时 Pod 才会被调度，并且它们永远不会出现在同一个节点上
## - soft：尽最大努力调度
antiAffinity: 'soft'
# tolerations:
#   - operator: "Exists" ##容忍全部污点
~~~

然后创建用于 Data 节点安装的 values 文件：`values-data.yaml`

~~~yaml
# values-data.yaml

# ============设置集群名称============
## 设置集群名称
clusterName: 'elasticsearch'
## 设置节点名称
nodeGroup: 'data'
## 设置角色
roles:
  master: 'false'
  ingest: 'true'
  data: 'true'

# ============镜像配置============
## 指定镜像与镜像版本
image: 'crpi-hdcg863vh2ayst45.cn-shanghai.personal.cr.aliyuncs.com/liuxu8677/elasticsearch'
# image: 'elasticsearch' # 此处会去官网下载，对应的访问地址为：docker.io/library/elasticsearch:7.17.3
imageTag: '7.17.3'
## 副本数(建议设置为3，资源不足可以只设置1个副本)
#replicas: 3
replicas: 1

# ============资源配置============
## JVM 配置参数
esJavaOpts: '-Xmx1g -Xms1g'
## 部署资源配置(生产环境一定要设置大些)
resources:
  requests:
    cpu: '1000m'
    memory: '2Gi'
  limits:
    cpu: '1000m'
    memory: '2Gi'
## 数据持久卷配置
persistence:
  enabled: true
## 存储数据大小配置
volumeClaimTemplate:
  storageClassName: nfs-client
  accessModes: ['ReadWriteOnce']
  resources:
    requests:
      storage: 10Gi

# ============安全配置============
## 设置协议，可配置为 http、https
protocol: http
## 证书挂载配置，这里我们挂入上面创建的证书
secretMounts:
  - name: elastic-certs
    secretName: elastic-certs
    path: /usr/share/elasticsearch/config/certs
## 允许您在/usr/share/elasticsearch/config/中添加任何自定义配置文件,例如 elasticsearch.yml
## ElasticSearch 7.x 默认安装了 x-pack 插件，部分功能免费，这里我们配置下
## 下面注掉的部分为配置 https 证书，配置此部分还需要配置 helm 参数 protocol 值改为 https
esConfig:
  elasticsearch.yml: |
    xpack.security.enabled: true
    xpack.security.transport.ssl.enabled: true
    xpack.security.transport.ssl.verification_mode: certificate
    xpack.security.transport.ssl.keystore.path: /usr/share/elasticsearch/config/certs/elastic-certificates.p12
    xpack.security.transport.ssl.truststore.path: /usr/share/elasticsearch/config/certs/elastic-certificates.p12
    # xpack.security.http.ssl.enabled: true
    # xpack.security.http.ssl.truststore.path: /usr/share/elasticsearch/config/certs/elastic-certificates.p12
    # xpack.security.http.ssl.keystore.path: /usr/share/elasticsearch/config/certs/elastic-certificates.p12

    # 禁用了 Elasticsearch 从外部下载 GeoIP 数据库的功能（GeoIP 数据库用于将 IP 地址映射到地理位置）
    # 这在以下场景中非常有用：
    # 1、日志分析：了解用户访问来源的地理位置。
    # 2、安全审计：检测异常的地理位置访问。
    # 3、内容个性化：根据用户位置提供个性化内容
    # 如果你没有需要使用 GeoIP 数据库的特定需求，禁用这个选项是完全可以的，特别是在内网环境中，这样可以避免连接外网的问题
    ingest.geoip.downloader.enabled: false

## 环境变量配置，这里引入上面设置的用户名、密码 secret 文件
extraEnvs:
  - name: ELASTIC_USERNAME
    valueFrom:
      secretKeyRef:
        name: elastic-auth
        key: username
  - name: ELASTIC_PASSWORD
    valueFrom:
      secretKeyRef:
        name: elastic-auth
        key: password

# ============调度配置============
## 设置调度策略
## - hard：只有当有足够的节点时 Pod 才会被调度，并且它们永远不会出现在同一个节点上
## - soft：尽最大努力调度
antiAffinity: 'soft'
## 容忍配置
# tolerations:
#   - operator: "Exists" ##容忍全部污点
~~~

最后一个是用于创建 Client 节点的 values 文件：`values-client.yaml`

~~~yaml
# values-client.yaml

# ============设置集群名称============
## 设置集群名称
clusterName: 'elasticsearch'
## 设置节点名称
nodeGroup: 'client'
## 设置角色
roles:
  master: 'false'
  ingest: 'false'
  data: 'false'

# ============镜像配置============
## 指定镜像与镜像版本
image: 'crpi-hdcg863vh2ayst45.cn-shanghai.personal.cr.aliyuncs.com/liuxu8677/elasticsearch'
#image: 'elasticsearch'
imageTag: '7.17.3'
## 副本数
# 测试环境资源有限，所以设置为1吧
replicas: 1

# ============资源配置============
## JVM 配置参数
esJavaOpts: '-Xmx1g -Xms1g'
## 部署资源配置(生产环境一定要设置大些)
resources:
  requests:
    cpu: '1000m'
    memory: '2Gi'
  limits:
    cpu: '1000m'
    memory: '2Gi'
## 数据持久卷配置
persistence:
  enabled: false

# ============安全配置============
## 设置协议，可配置为 http、https
protocol: http
## 证书挂载配置，这里我们挂入上面创建的证书
secretMounts:
  - name: elastic-certs
    secretName: elastic-certs
    path: /usr/share/elasticsearch/config/certs
## 允许您在/usr/share/elasticsearch/config/中添加任何自定义配置文件,例如 elasticsearch.yml
## ElasticSearch 7.x 默认安装了 x-pack 插件，部分功能免费，这里我们配置下
## 下面注掉的部分为配置 https 证书，配置此部分还需要配置 helm 参数 protocol 值改为 https
esConfig:
  elasticsearch.yml: |
    xpack.security.enabled: true
    xpack.security.transport.ssl.enabled: true
    xpack.security.transport.ssl.verification_mode: certificate
    xpack.security.transport.ssl.keystore.path: /usr/share/elasticsearch/config/certs/elastic-certificates.p12
    xpack.security.transport.ssl.truststore.path: /usr/share/elasticsearch/config/certs/elastic-certificates.p12
    # xpack.security.http.ssl.enabled: true
    # xpack.security.http.ssl.truststore.path: /usr/share/elasticsearch/config/certs/elastic-certificates.p12
    # xpack.security.http.ssl.keystore.path: /usr/share/elasticsearch/config/certs/elastic-certificates.p12
## 环境变量配置，这里引入上面设置的用户名、密码 secret 文件
extraEnvs:
  - name: ELASTIC_USERNAME
    valueFrom:
      secretKeyRef:
        name: elastic-auth
        key: username
  - name: ELASTIC_PASSWORD
    valueFrom:
      secretKeyRef:
        name: elastic-auth
        key: password

# ============Service 配置============
service:
  type: NodePort
  nodePort: '30200'
~~~

切换到 elasticsearch 的 chart 目录下

~~~bash
cd /logging/elasticsearch/
[root@k8s-master-01 elasticsearch]# ls -l
总用量 84
-rw-r--r--  1 root root   341 12月 20 16:10 Chart.yaml
drwxr-xr-x 14 root root   211 12月 20 16:10 examples
-rw-r--r--  1 root root    29 12月 20 16:10 Makefile
-rw-r--r--  1 root root 49860 12月 20 16:10 README.md
drwxr-xr-x  3 root root   297 12月 20 16:10 templates
-rw-r--r--  1 root root  2487 12月 20 16:23 values-client.yaml
-rw-r--r--  1 root root  3575 12月 20 16:23 values-data.yaml
-rw-r--r--  1 root root  2928 12月 20 16:23 values-master.yaml
-rw-r--r--  1 root root  9496 12月 20 16:10 values.yaml
~~~

安装

~~~bash
# 注意install指定的release名字要不同

# 安装 master 节点
helm install es-master ./ -f values-master.yaml --namespace logging
# 安装 data 节点
helm install es-data ./ -f values-data.yaml --namespace logging
# 安装 client 节点
helm install es-client ./ -f values-client.yaml --namespace logging


# # 升级操作
#$ helm upgrade --install es-master -f values-master.yaml --namespace logging .
#$ helm upgrade --install es-data -f values-data.yaml --namespace logging .
#$ helm upgrade --install es-client -f values-client.yaml --namespace logging .
~~~



查看 es 集群中的三个节点

~~~bash
[root@k8s-master-01 elasticsearch]# kubectl -n logging get pods
NAME                     READY   STATUS    RESTARTS   AGE
elasticsearch-client-0   1/1     Running   0          4m38s
elasticsearch-data-0     1/1     Running   0          19m
elasticsearch-master-0   1/1     Running   0          19m
~~~

查看 es 集群中的 svc，一会访问 es 就用用该 elasticsearch-client svc

~~~bash
[root@k8s-master-01 elasticsearch]# kubectl -n logging get svc elasticsearch-client
NAME                   TYPE       CLUSTER-IP       EXTERNAL-IP   PORT(S)                         AGE
elasticsearch-client   NodePort   10.100.182.158   <none>        9200:30200/TCP,9300:31760/TCP   19m
~~~





## 2. 安装 Kibana

下载 chart 包

~~~bash
cd /logging
helm pull elastic/kibana --untar --version 7.17.3
cd kibana
~~~

创建用于安装 Kibana 的 values 文件：`values-prod.yaml`

~~~yaml
# values-prod.yaml

## 指定镜像与镜像版本
image: 'crpi-hdcg863vh2ayst45.cn-shanghai.personal.cr.aliyuncs.com/liuxu8677/kibana'
#image: 'docker.elastic.co/kibana/kibana'
imageTag: '7.17.3'
imagePullPolicy: "IfNotPresent"

## 配置 ElasticSearch 地址，主要用用的es-client的svc
elasticsearchHosts: 'http://elasticsearch-client:9200'

# ============环境变量配置============
## 环境变量配置，这里引入上面设置的用户名、密码 secret 文件
extraEnvs:
  - name: 'ELASTICSEARCH_USERNAME'
    valueFrom:
      secretKeyRef:
        name: elastic-auth
        key: username
  - name: 'ELASTICSEARCH_PASSWORD'
    valueFrom:
      secretKeyRef:
        name: elastic-auth
        key: password

# ============资源配置============
resources:
  requests:
    cpu: '500m'
    memory: '1Gi'
  limits:
    cpu: '500m'
    memory: '1Gi'

# ============配置 Kibana 参数============
## kibana 配置中添加语言配置，设置 kibana 为中文
kibanaConfig:
  kibana.yml: |
    i18n.locale: "zh-CN"
    server.publicBaseUrl: "http://192.168.10.81:30601"   #这里地址改为你访问kibana的地址，不能以 / 结尾
# ============Service 配置============
service:
  type: NodePort
  nodePort: '30601'
~~~

部署

~~~bash
helm install kibana -f values-prod.yaml --namespace logging .
~~~

部署完后查看

~~~bash
[root@k8s-master-01 kibana]# kubectl -n logging get pods
NAME                             READY   STATUS    RESTARTS   AGE
elasticsearch-client-0           1/1     Running   0          114s
elasticsearch-data-0             1/1     Running   0          2m8s
elasticsearch-master-0           1/1     Running   0          2m13s
kibana-kibana-7747ff5b58-rb7m2   1/1     Running   0          3m37s
~~~

上面我们安装 Kibana 的时候指定了 30601 的 NodePort 端口，所以我们可以从任意节点 http://IP:30601 来访问 Kibana





## 3. 安装Fluentd

作为日志收集工具。

工作原理/流程如下：

- 首先 Fluentd 会从给定的多个日志源获取数据，例如日志文件、restful api 
- 然后将这些数据转换成结构化的数据格式并且打上标记 
- 最后根据匹配的标签将数据发送到多个目标服务（比如 Elasticsearch、对象存储等等,当然也可以 存到kafka中）



配置文件 `fluentd-configmap.yaml`

配置会采集物理节点上的 /var/log/containers/*.log 日志，然后进行处理后发送到 elasticsearchclient:9200 服务

~~~yaml
# fluentd-configmap.yaml

kind: ConfigMap
apiVersion: v1
metadata:
  name: fluentd-conf
  namespace: logging
data:
  # 一、配置采集与处理容器日志
  containers.input.conf: |-
    # ====================》1、日志源配置：采集哪些日志 《====================
    <source>
      @id fluentd-containers.log
      @type tail                              # Fluentd 内置的输入方式，其原理是不停地从源文件中获取新的日志
      path /var/log/containers/*.log          # 容器标准输出日志对应的宿主机路径
      pos_file /var/log/es-containers.log.pos  # 记录读取的位置
      tag raw.kubernetes.*                    # 设置日志标签
      read_from_head true                     # 从头读取
      <parse>                                 # 多行格式化成JSON
        # 可以使用我们介绍过的 multiline 插件实现多行日志
        @type multi_format                    # 使用 multi-format-parser 解析器插件
        <pattern>
          format json                         # JSON解析器
          time_key time                       # 指定事件时间的时间字段
          time_format %Y-%m-%dT%H:%M:%S.%NZ   # 时间格式
        </pattern>
        <pattern>
          format /^(?<time>.+) (?<stream>stdout|stderr) [^ ]* (?<log>.*)$/
          time_format %Y-%m-%dT%H:%M:%S.%N%:z
        </pattern>
      </parse>
    </source>


    # ====================》2、在日志输出中检测异常、同时移除日志标签中的 raw 前缀、并将其作为一条日志转发 《====================
    # 综述：处理标签为 raw.kubernetes.** 的日志。它利用 detect_exceptions 插件来识别和处理异常栈信息，同时移除日志标签中的 raw 前缀，将日志消息字段指定为 log。并且多行日志在 5 秒内会被刷新和处理，以形成一条完整的日志记录
    # https://github.com/GoogleCloudPlatform/fluent-plugin-detect-exceptions
    
    # 详解如下
    <match raw.kubernetes.**>           # raw.kubernetes.** 表示匹配所有以 raw.kubernetes. 开头的日志数据。
      @id raw.kubernetes # 为这个输出插件配置指定了一个唯一的 ID raw.kubernetes，便于管理和调试。
      @type detect_exceptions           # 这个插件用于处理日志中的异常栈信息（stack traces）。如果日志中包含异常信息，它会识别并处理这些异常栈信息，以便进行更好的日志分析和错误处理。

      remove_tag_prefix raw             # 设置 remove_tag_prefix 为 raw，表示在将日志发送到目标之前，会移除日志标签中的 raw 前缀。例如，raw.kubernetes.some_log 会被转换为 kubernetes.some_log。这有助于整理和规范日志标签。
      message log                       # 这个设置指定了日志消息字段的名称。在处理日志时，插件会查找名为 log 的字段作为日志消息的主要内容。默认情况下，detect_exceptions 插件会从这个字段中提取异常信息
      multiline_flush_interval 5        # 这个设置确保了多行日志在 5 秒内会被刷新和处理，以形成一条完整的日志记录
    </match>



    # ====================》3、过滤器配置 《====================
    # 1、拼接日志（如果日志内容里是包含换行符多行显示的，那需要拼成一行）
    <filter **> # ** 表示匹配所有日志源。
      @id filter_concat # 唯一id，以便管理和调试。
      @type concat                # 使用 concat 插件，这个插件用于将多行日志拼接成一条完整的日志记录。
      key message                 # 指定要拼接的字段，这里是 message。插件会在这个字段内执行多行拼接操作
      multiline_end_regexp /\n$/  # 以换行符“\n”拼接
      separator "" # 设置拼接后的日志条目之间的分隔符为空字符串。即多行日志将被直接拼接在一起，没有额外的分隔符
    </filter>

    # 2、添加 Kubernetes metadata 数据
    <filter kubernetes.**> # 匹配所有以kubernetes. 开头的日志。
      @id filter_kubernetes_metadata
      @type kubernetes_metadata # 使用 kubernetes_metadata 插件，这个插件用于为日志添加 Kubernetes 元数据。它会根据日志中的 Kubernetes 相关信息（如 Pod 名称、容器名称等）补充相应的元数据
    </filter>

    # 3、修复 ES 中的 JSON 字段
    # 插件地址：https://github.com/repeatedly/fluent-plugin-multi-format-parser
    <filter kubernetes.**> # 匹配所有以kubernetes. 开头的日志。
      @id filter_parser
      @type parser                # 使用multi-format-parser多格式解析器插件
      key_name log                # 指定要解析的日志字段名称是 log。插件会在这个字段内执行解析操作。
      reserve_data true           # 设置为 true，表示在解析过程中保留原始数据。这有助于保留日志数据的原始形式，便于调试和进一步分析。
      remove_key_name_field true  # 设置为 true，表示在成功解析日志后，移除 key_name 字段。这样可以在解析后去掉指定的日志字段名称。
      <parse> # 依次运行多个解析pattern，运行不成功才会进行下一个
        @type multi_format
        <pattern>
          format json
        </pattern>
        <pattern>
          format none # 对于无法解析为 JSON 的数据，none 表示不进行任何处理。可以用于处理非 JSON 格式的日志数据。
        </pattern>
      </parse>
    </filter>

    # 4、删除一些多余的属性
    <filter kubernetes.**>
      @type record_transformer
      remove_keys $.docker.container_id,$.kubernetes.container_image_id,$.kubernetes.pod_id,$.kubernetes.namespace_id,$.kubernetes.master_url,$.kubernetes.labels.pod-template-hash
    </filter>

    # 5、只保留具有logging=true标签的Pod日志
    <filter kubernetes.**>
      @id filter_log
      @type grep
      <regexp>
        key $.kubernetes.labels.logging
        pattern ^true$
      </regexp>
    </filter>


  # 二、监听配置，一般用于日志聚合用
  forward.input.conf: |-
    # 监听通过TCP发送的消息
    <source>
      @id forward
      @type forward # forward 插件会监听通过 TCP 协议发送到 Fluentd 的日志消息。其他 Fluentd 实例或应用程序可以将日志数据发送到这个 Fluentd 实例，该实例会接收并处理这些日志消息
      
      # 默认情况下，forward 插件会监听 TCP 端口 24224。您可以根据需要修改端口号和其他设置。要进行自定义配置，如设置端口号、绑定地址等，可以添加相应的配置参数。例如
      # port 24224  # 设置监听的端口号
      # bind 0.0.0.0  # 绑定所有网络接口
      
      
      # 这个配置通常用于构建多实例 Fluentd 部署，其中一个实例作为集中式日志接收器，接收来自不同实例或服务的日志数据。这种设置有助于集中管理和处理日志数据，并将日志数据进一步转发到其他系统（如 Elasticsearch、Kafka、文件等）。
    </source>


  # 三、配置将日志数据发送到 Elasticsearch
  output.conf: |-
    <match **>
      @id elasticsearch
      @type elasticsearch
      @log_level info
      include_tag_key true

      # 配置访问ES的svc与端口，账号密码
      host elasticsearch-client # 指定ES的client的svc名
      port 9200
      user elastic # FLUENT_ELASTICSEARCH_USER | FLUENT_ELASTICSEARCH_PASSWORD
      password elastic123
      
      # 配置采集的日志使用 logstash 格式，
      # 并且配置日志前缀为k8s，后续在es中索引查询要用到该前缀
      logstash_format true
      logstash_prefix k8s

      request_timeout 30s
      # 缓冲设置
      <buffer>
        @type file
        path /var/log/fluentd-buffers/kubernetes.system.buffer
        flush_mode interval
        retry_type exponential_backoff
        flush_thread_count 2
        flush_interval 5s
        retry_forever
        retry_max_interval 30
        chunk_limit_size 2M
        queue_limit_length 8
        overflow_action block
      </buffer>
    </match>
~~~

然后新建一个 `fluentd-daemonset.yaml` 的文件，文件内容如下：

~~~yaml
# fluentd-daemonset.yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: fluentd-es
  namespace: logging
  labels:
    k8s-app: fluentd-es
    kubernetes.io/cluster-service: 'true'
    addonmanager.kubernetes.io/mode: Reconcile
---
kind: ClusterRole
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: fluentd-es
  labels:
    k8s-app: fluentd-es
    kubernetes.io/cluster-service: 'true'
    addonmanager.kubernetes.io/mode: Reconcile
rules:
  - apiGroups:
      - ''
    resources:
      - 'namespaces'
      - 'pods'
    verbs:
      - 'get'
      - 'watch'
      - 'list'
---
kind: ClusterRoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: fluentd-es
  labels:
    k8s-app: fluentd-es
    kubernetes.io/cluster-service: 'true'
    addonmanager.kubernetes.io/mode: Reconcile
subjects:
  - kind: ServiceAccount
    name: fluentd-es
    namespace: logging
    apiGroup: ''
roleRef:
  kind: ClusterRole
  name: fluentd-es
  apiGroup: ''

---
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: fluentd
  namespace: logging
  labels:
    app: fluentd
    kubernetes.io/cluster-service: 'true'
spec:
  selector:
    matchLabels:
      app: fluentd
  template:
    metadata:
      labels:
        app: fluentd
        kubernetes.io/cluster-service: 'true'
    spec:
      # 1、控制只调度到带有下述标签的节点上，即只采集固定的节点
      #nodeSelector:
      #  beta.kubernetes.io/fluentd-ds-ready: 'true'
      
      # 2、我们是kubeadm部署的k8s，默认情况下master节点有污点，如果想采集master，则需要加上容忍
      tolerations:
        - key: node-role.kubernetes.io/master
          effect: NoSchedule
      serviceAccountName: fluentd-es
      containers:
        - name: fluentd
          # 替换过内镜像
          # image: quay.io/fluentd_elasticsearch/fluentd:v3.4.0
          image: crpi-hdcg863vh2ayst45.cn-shanghai.personal.cr.aliyuncs.com/liuxu8677/fluentd:v3.4.0
          volumeMounts:
            - name: fluentconfig
              mountPath: /etc/fluent/config.d
            - name: varlog
              mountPath: /var/log
      volumes:
        - name: fluentconfig
          configMap:
            name: fluentd-conf
        - name: varlog
          hostPath:
            path: /var/log
~~~

补充一个点：

- 想要只采集某些节点的日志，可以添加一个 nodSelector 属性：

~~~bash
nodeSelector:
  beta.kubernetes.io/fluentd-ds-ready: 'true'
~~~

对应着，你想采集哪个节点的日志，就要给该节点打上标签

~~~bash
kubectl label nodes <node名> beta.kubernetes.io/fluentd-ds-ready=true
~~~

创建

~~~bash
kubectl apply -f fluentd-configmap.yaml
kubectl apply -f fluentd-daemonset.yaml
~~~

Fluentd 启动成功后，这个时候就可以发送日志到 ES 了，但是我们这里是过滤了只采集具有 logging=true 标签的 Pod 日志，所以现在还没有任何数据会被采集。

所以我们来部署一个简单的测试应用，并且为pod打上标签 `logging=true`

创建测试 pod

~~~yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: mytest
spec:
  replicas: 1
  selector:
    matchLabels:
      app: mytest
      
  template:
    metadata:
      labels:
        app: mytest
        logging: 'true' # 一定要具有该标签才会被采集
        logIndex: 'mytest'
    spec:
      containers:
        - name: nmytest
          image: busybox:1.27
          command: ["/bin/sh","-c","while true;do echo 111;sleep 1;done"]
~~~

Pod 创建并运行后，回到 Kibana Dashboard 页面，点击页面下面的 Stack Management ，进入管理 页面，点击左侧 Kibana 下面的 索引模式 ，点击 创建索引模式 开始导入索引数据。



## 4. 安装 Kafka



集群规模大了，日志量巨大，Fluentd采集日子后，直接放入ES对ES的压力是非常大的，需要引入kafka 来削峰、缓解ES压力。

具体怎么做呢，有两种方法 

- 1、使用 kafka-connect-elasticsearch 这样的工具将数据直接打入 ES 
- 2、也可以在加一层 Logstash 去消费 Kafka 的数据，然后通过 Logstash 把数据存入 ES，对比上一种方 法Logstash除了充当kafka与ES之间的桥梁外，还可以利用Lostash的强大功能，对日志收集进行优化处理。

此处我们采用方法2，流程如下

~~~bash
Fluentd-----------> Kafka-----------> Logstash------------> ES--------------> Kibana
~~~

先在k8s中安装kafka

~~~bash
# 配置
helm repo add bitnami https://charts.bitnami.com/bitnami
helm repo update

# 拉取
helm pull bitnami/kafka --untar --version 17.2.3
cd kafka
~~~

在当前 chart目录下创建一个values文件，文件中指定 StorageClass 来提供持久化存储，其他配置 不用写使用默认的

~~~yaml
# values-prod.yaml
## @section Persistence parameters
persistence:
  enabled: true
  storageClass: 'nfs-client'
  accessModes:
    - ReadWriteOnce
  size: 8Gi

  mountPath: /bitnami/kafka

# 配置zk volumes
zookeeper:
  enabled: true
  persistence:
    enabled: true
    storageClass: 'nfs-client'
    accessModes:
      - ReadWriteOnce
    size: 8Gi
~~~

部署

~~~bash
helm upgrade --install kafka -f values-prod.yaml --namespace logging .

# 得到如下输出
Release "kafka" does not exist. Installing it now.
NAME: kafka
LAST DEPLOYED: Sat Dec 20 17:54:42 2025
NAMESPACE: logging
STATUS: deployed
REVISION: 1
TEST SUITE: None
NOTES:
CHART NAME: kafka
CHART VERSION: 17.2.3
APP VERSION: 3.2.0

** Please be patient while the chart is being deployed **

Kafka can be accessed by consumers via port 9092 on the following DNS name from within your cluster:

    kafka.logging.svc.cluster.local

Each Kafka broker can be accessed by producers via port 9092 on the following DNS name(s) from within your cluster:

    kafka-0.kafka-headless.logging.svc.cluster.local:9092

To create a pod that you can use as a Kafka client run the following commands:

    kubectl run kafka-client --restart='Never' --image docker.io/bitnami/kafka:3.2.0-debian-10-r4 --namespace logging --command -- sleep infinity
    kubectl exec --tty -i kafka-client --namespace logging -- bash

    PRODUCER:
        kafka-console-producer.sh \
            
            --broker-list kafka-0.kafka-headless.logging.svc.cluster.local:9092 \
            --topic test

    CONSUMER:
        kafka-console-consumer.sh \
            
            --bootstrap-server kafka.logging.svc.cluster.local:9092 \
            --topic test \
            --from-beginning

~~~

查看安装结果

~~~bash
[root@k8s-master-01 kafka]# kubectl -n logging get pods -w
NAME                             READY   STATUS    RESTARTS   AGE
elasticsearch-client-0           1/1     Running   0          85m
elasticsearch-data-0             1/1     Running   0          86m
elasticsearch-master-0           1/1     Running   0          86m
fluentd-8tggp                    1/1     Running   0          63m
fluentd-cwmmr                    1/1     Running   0          63m
fluentd-k4jv4                    1/1     Running   0          63m
fluentd-rn5ng                    1/1     Running   0          63m
kafka-0                          1/1     Running   0          24m
kafka-zookeeper-0                1/1     Running   0          25m
kibana-kibana-7747ff5b58-rb7m2   1/1     Running   0          87m
~~~

用提示命令创建一个Kafka的测试客户端Pod

~~~bash
kubectl run kafka-client --restart='Never' --image docker.io/bitnami/kafka:3.2.0-debian-10-r4 --namespace logging --command -- sleep infinity
~~~

进入测试pod内，启动生产者

~~~bash
kubectl exec --tty -i kafka-client --namespace logging -- bash

# 在容器内
kafka-console-producer.sh --broker-list kafka-0.kafka-headless.logging.svc.cluster.local:9092 --topic test
~~~

另外一个终端，也进入pod内，启动消费者

~~~bash
kubectl exec --tty -i kafka-client --namespace logging -- bash
# 在容器内
kafka-console-consumer.sh --bootstrap-server kafka.logging.svc.cluster.local:9092 --topic test --from-beginning
~~~

在生产者终端输入消息，在消费者终端能看到消息，证明kafka可用



## 5. 配置 Fluentd 对接 Kafka

之前我们部署的 Fluentd 都是将采集的日志直接打到 ES 中，现在我们改成打到 Kafka 中，分两步实现：

- 将 Fluentd 配置中的 `<match>` 更改为使用 Kafka 插件。
- 在 Fluentd 中输出到 Kafka，需要使用到 `fluent-plugin-kafka` 插件，原始的 Fluentd 镜像中并没有该插件，所以我们在原镜像的基础上重新构建镜像，新增 kafka 插件即可。

使用 nerdctl 构建镜像，创建 Containerfile 文件

~~~
#FROM quay.io/fluentd_elasticsearch/fluentd:v3.4.0
FROM registry.cn-hangzhou.aliyuncs.com/egon-k8s-test/fluentd:v3.4.0
RUN echo "source 'https://mirrors.tuna.tsinghua.edu.cn/rubygems/'" > Gemfile &&
gem install bundler -v 2.4.22
RUN gem install fluent-plugin-kafka -v 0.17.5 --no-document
~~~

在 Containerfile  所在的目录下执行

~~~bash
nerdctl build -t quay.io/fluentd_elasticsearch/fluentd-kafka:v0.17.5 -f Containerfile ./
~~~

找到之前部署 时用到的 Fluentd 的 Configmap 文件，将 output.conf 中的  部分整体替换如下

~~~yaml
# fluentd-configmap.yaml
kind: ConfigMap
apiVersion: v1
metadata:
  name: fluentd-conf
  namespace: logging
data:
  # 一、配置采集与处理容器日志
  containers.input.conf: |-
    # ====================》1、日志源配置：采集哪些日志 《====================
    <source>
      @id fluentd-containers.log
      @type tail                              # Fluentd 内置的输入方式，其原理是不停地从源文件中获取新的日志
      path /var/log/containers/*.log          # 容器标准输出日志对应的宿主机路径
      pos_file /var/log/es-containers.log.pos  # 记录读取的位置
      tag raw.kubernetes.*                    # 设置日志标签
      read_from_head true                     # 从头读取
      <parse>                                 # 多行格式化成JSON
        # 可以使用我们介绍过的 multiline 插件实现多行日志
        @type multi_format                    # 使用 multi-format-parser 解析器插件
        <pattern>
          format json                         # JSON解析器
          time_key time                       # 指定事件时间的时间字段
          time_format %Y-%m-%dT%H:%M:%S.%NZ   # 时间格式
        </pattern>
        <pattern>
          format /^(?<time>.+) (?<stream>stdout|stderr) [^ ]* (?<log>.*)$/
          time_format %Y-%m-%dT%H:%M:%S.%N%:z
        </pattern>
      </parse>
    </source>


    # ====================》2、在日志输出中检测异常、同时移除日志标签中的 raw 前缀、并将其作为一条日志转发 《====================
    # 综述：处理标签为 raw.kubernetes.** 的日志。它利用 detect_exceptions 插件来识别和处理异常栈信息，同时移除日志标签中的 raw 前缀，将日志消息字段指定为 log。并且多行日志在 5 秒内会被刷新和处理，以形成一条完整的日志记录
    # https://github.com/GoogleCloudPlatform/fluent-plugin-detect-exceptions
    
    # 详解如下
    <match raw.kubernetes.**>           # raw.kubernetes.** 表示匹配所有以 raw.kubernetes. 开头的日志数据。
      @id raw.kubernetes # 为这个输出插件配置指定了一个唯一的 ID raw.kubernetes，便于管理和调试。
      @type detect_exceptions           # 这个插件用于处理日志中的异常栈信息（stack traces）。如果日志中包含异常信息，它会识别并处理这些异常栈信息，以便进行更好的日志分析和错误处理。

      remove_tag_prefix raw             # 设置 remove_tag_prefix 为 raw，表示在将日志发送到目标之前，会移除日志标签中的 raw 前缀。例如，raw.kubernetes.some_log 会被转换为 kubernetes.some_log。这有助于整理和规范日志标签。
      message log                       # 这个设置指定了日志消息字段的名称。在处理日志时，插件会查找名为 log 的字段作为日志消息的主要内容。默认情况下，detect_exceptions 插件会从这个字段中提取异常信息
      multiline_flush_interval 5        # 这个设置确保了多行日志在 5 秒内会被刷新和处理，以形成一条完整的日志记录
    </match>



    # ====================》3、过滤器配置 《====================
    # 1、拼接日志（如果日志内容里是包含换行符多行显示的，那需要拼成一行）
    <filter **> # ** 表示匹配所有日志源。
      @id filter_concat # 唯一id，以便管理和调试。
      @type concat                # 使用 concat 插件，这个插件用于将多行日志拼接成一条完整的日志记录。
      key message                 # 指定要拼接的字段，这里是 message。插件会在这个字段内执行多行拼接操作
      multiline_end_regexp /\n$/  # 以换行符“\n”拼接
      separator "" # 设置拼接后的日志条目之间的分隔符为空字符串。即多行日志将被直接拼接在一起，没有额外的分隔符
    </filter>

    # 2、添加 Kubernetes metadata 数据
    <filter kubernetes.**> # 匹配所有以kubernetes. 开头的日志。
      @id filter_kubernetes_metadata
      @type kubernetes_metadata # 使用 kubernetes_metadata 插件，这个插件用于为日志添加 Kubernetes 元数据。它会根据日志中的 Kubernetes 相关信息（如 Pod 名称、容器名称等）补充相应的元数据
    </filter>

    # 3、修复 ES 中的 JSON 字段
    # 插件地址：https://github.com/repeatedly/fluent-plugin-multi-format-parser
    <filter kubernetes.**> # 匹配所有以kubernetes. 开头的日志。
      @id filter_parser
      @type parser                # 使用multi-format-parser多格式解析器插件
      key_name log                # 指定要解析的日志字段名称是 log。插件会在这个字段内执行解析操作。
      reserve_data true           # 设置为 true，表示在解析过程中保留原始数据。这有助于保留日志数据的原始形式，便于调试和进一步分析。
      remove_key_name_field true  # 设置为 true，表示在成功解析日志后，移除 key_name 字段。这样可以在解析后去掉指定的日志字段名称。
      <parse> # 依次运行多个解析pattern，运行不成功才会进行下一个
        @type multi_format
        <pattern>
          format json
        </pattern>
        <pattern>
          format none # 对于无法解析为 JSON 的数据，none 表示不进行任何处理。可以用于处理非 JSON 格式的日志数据。
        </pattern>
      </parse>
    </filter>

    # 4、删除一些多余的属性
    <filter kubernetes.**>
      @type record_transformer
      remove_keys $.docker.container_id,$.kubernetes.container_image_id,$.kubernetes.pod_id,$.kubernetes.namespace_id,$.kubernetes.master_url,$.kubernetes.labels.pod-template-hash
    </filter>

    # 5、只保留具有logging=true标签的Pod日志
    <filter kubernetes.**>
      @id filter_log
      @type grep
      <regexp>
        key $.kubernetes.labels.logging
        pattern ^true$
      </regexp>
    </filter>






  # 二、监听配置，一般用于日志聚合用
  forward.input.conf: |-
    # 监听通过TCP发送的消息
    <source>
      @id forward
      @type forward # forward 插件会监听通过 TCP 协议发送到 Fluentd 的日志消息。其他 Fluentd 实例或应用程序可以将日志数据发送到这个 Fluentd 实例，该实例会接收并处理这些日志消息
      
      # 默认情况下，forward 插件会监听 TCP 端口 24224。您可以根据需要修改端口号和其他设置。要进行自定义配置，如设置端口号、绑定地址等，可以添加相应的配置参数。例如
      # port 24224  # 设置监听的端口号
      # bind 0.0.0.0  # 绑定所有网络接口
      
      
      # 这个配置通常用于构建多实例 Fluentd 部署，其中一个实例作为集中式日志接收器，接收来自不同实例或服务的日志数据。这种设置有助于集中管理和处理日志数据，并将日志数据进一步转发到其他系统（如 Elasticsearch、Kafka、文件等）。
    </source>




  # 三、配置将日志数据发送到 Elasticsearch
  output.conf: |-
    <match **>
      @id kafka
      @type kafka2
      @log_level info

      # list of seed brokers
      #brokers kafka-0.kafka-headless.logging.svc.cluster.local:9092
      brokers kafka.logging.svc.cluster.local:9092
      use_event_time true # 使用事件时间作为 Kafka 消息的时间戳。这样 Kafka 消息中的时间戳将反映日志事件的时间，而不是日志到达 Fluentd 的时间。

      # topic settings
      # topic_key k8slog # 由于 topic_key 是固定的 k8slog，所有消息将被发送到 messages 主题的同一个分区，从而保证消息的顺序性。移除该配置消息将会被均匀分布到所有分区，适合负载均衡，但是不保证顺序性
      default_topic messages  # 注意，kafka中消费使用的是这个topic
      
      # 缓冲设置：缓冲类型、缓冲文件存储路径、每 3 秒钟刷新一次缓冲区中的数据。
      <buffer k8slog>
        @type file
        path /var/log/td-agent/buffer/td
        flush_interval 3s
      </buffer>

      # data type settings
      <format>
        @type json # 指定输出数据为 JSON 格式。
      </format>

      # producer settings
      required_acks -1         # 生产者需要等待所有的副本确认，确保消息的可靠性
      compression_codec gzip   # 使用 GZIP 压缩消息，以减少网络带宽的消耗。

    </match>
~~~

然后找到部署Fluentd的yaml文件，修改镜像地址

~~~yaml
image: registry.cn-hangzhou.aliyuncs.com/egon-k8s-test/fluentd-kafka:v0.17.5
~~~

然后重新部署

~~~bash
kubectl apply -f fluentd-configmap.yaml
kubectl apply -f fluentd-daemonset.yaml
~~~



## 6. 安装 Logstash 对接 Kafka

从 Kafka 中读取数据然后存到 Elasticsearch ，采用更加流行的 Logstash 方案。



拉取并解压 chart

~~~bash
helm pull elastic/logstash --untar --version 7.17.3
cd logstash
~~~

同样在 Chart 根目录下面创建全新的、用于安装的 Values 文件，如下所示

~~~yaml
# values-prod.yaml
resources:
  requests:
    cpu: "100m"
    memory: "1536Mi"
  limits:
    cpu: "1000m"
    memory: "1536Mi"
    
fullnameOverride: logstash

persistence:
  enabled: true

logstashConfig:
  logstash.yml: |
    http.host: 0.0.0.0
    # 如果启用了xpack，需要做如下配置
    xpack.monitoring.enabled: true
    xpack.monitoring.elasticsearch.hosts: ["http://elasticsearch-client:9200"]
    xpack.monitoring.elasticsearch.username: "elastic"
    xpack.monitoring.elasticsearch.password: "elastic123"

# 要注意下格式
logstashPipeline:
  logstash.conf: |
    input { kafka { bootstrap_servers => "kafka.logging.svc.cluster.local:9092" codec => json consumer_threads => 3 topics => ["messages"] } }
    filter {}  # 过滤配置（比如可以删除key、添加geoip等等）
    output { elasticsearch { hosts => [ "elasticsearch-client:9200" ] user => "elastic" password => "elastic123" index => "logstash-k8s-%{+YYYY.MM.dd}" } stdout { codec => rubydebug } }

volumeClaimTemplate:
  accessModes: ['ReadWriteOnce']
  storageClassName: nfs-client
  resources:
    requests:
      storage: 1Gi
~~~

上述配置中最终的是通过 logstashPipeline 配置 logstash 数据流的处理配置，解释如下

- `input ` 指定输入端、即日志源为 kafka
- `output ` 指定输出目标位置，输出到 Elasticsearch
- `output` 定义的输出的配置中，index指定索引为带着前缀 logstash-k8s-，后续在es中会用到该索引
- `rubydebug` 启用调试模式，方便后面查看



使用上述 `values-prod.yaml` 文件进行部署

~~~bash
helm upgrade --install logstash -f values-prod.yaml --namespace logging .
~~~

查看 pod

~~~bash
[root@k8s-master-01 logstash]# kubectl -n logging get pods
NAME                             READY   STATUS    RESTARTS   AGE
elasticsearch-client-0           1/1     Running   0          122m
elasticsearch-data-0             1/1     Running   0          122m
elasticsearch-master-0           1/1     Running   0          122m
fluentd-d25x8                    1/1     Running   0          9m53s
fluentd-jlvj4                    1/1     Running   0          10m
fluentd-x57ng                    1/1     Running   0          10m
fluentd-xvkx7                    1/1     Running   0          10m
kafka-0                          1/1     Running   0          61m
kafka-zookeeper-0                1/1     Running   0          61m
kibana-kibana-7747ff5b58-rb7m2   1/1     Running   0          123m
logstash-0                       1/1     Running   0          2m51s
~~~



然后我们可以查看logstash的从kafka中获取到的日志

~~~bash
kubectl logs -f logstash-0 -n logging
~~~



到 Kibana 可以看到有新的索引数据，创建完成后就可以前往发现页面过滤日志数据了。

至此我们就实现了一个使用 Fluentd+Kafka+Logstash+Elasticsearch+Kibana 的 Kubernetes 日志收集工具栈。



## 7. 定制索引名称

**第一道关卡：规定采集带有指定标签的 pod 的日志**

Fluentd采集到日志后可以对日志进行一些处理。即pod必须带有logging: true的标签，它输出的标注输出的日志才会被采集。

~~~
# 5、只保留具有logging=true标签的Pod日志
<filter kubernetes.**>
  @id filter_log
  @type grep
  <regexp>
    key $.kubernetes.labels.logging
    pattern ^true$
  </regexp>
</filter>
~~~

经过第一道关卡后日志送入了 kafka 中，接下来就是 logstash 会从 kafka 中取出了。



**第二道关卡：定义ES中要用到的索引名称**

logstash 除了可以从 kafka 中取出日志然后放入 es 中，它还可以对日志做进一步的处理， 相当于日志处理的第二道关卡。

例如上一小节我们在配置 logstash 的时候是将日志输出到 "logstash-k8s-%{+YYYY.MM.dd}" 这个 index索引模式的，代表按照日期进行索引，ES中就会使用该索引（ES对接的上游是Logstash而不是 fluentd，所以你用之前的k8s-*是检索不到的）

但可能有的场景下，仅仅只通过日期去区分索引不是很合理，此时我们可以根据自己的需求去修改索引 名称，比如可以根据我们的服务名称来进行区分，那么这个服务名称可以怎么来定义呢？

答：

1、在Pod 的yaml定义中新增一个label 标签来指定

~~~yaml
apiVersion: v1
kind: Pod
metadata:
  name: counter
  labels:
    ...........
    logIndex: 'test' # 指定索引名称
spec:
  containers:
    ...
~~~

2、重新更新 logstash 的配置，修改 values 配置：

~~~
# ......
logstashPipeline:
  logstash.conf: |
    input { kafka { bootstrap_servers => "kafka-0.kafkaheadless.logging.svc.cluster.local:9092" codec => json consumer_threads => 3 topics => ["messages"] } }
    filter {} # 过滤配置（比如可以删除key、添加geoip等等）
    output { elasticsearch { hosts => [ "elasticsearch-client:9200" ] user => "elastic" password => "egon666" index => "k8s-%{[kubernetes][labels][logIndex]}-%{+YYYY.MM.dd}" } stdout { codec => rubydebug } }
# ......
~~~

使用上面的 values 值更新 logstash

~~~bash
helm upgrade --install logstash -f values-prod.yaml --namespace logging .
~~~



测试验证

综合两道关卡，我们的pod必须带有下述两个标签，它的日志才会被顺利送进ES中

~~~yaml
labels:
  logging: 'true'
  logIndex: 'test'
~~~

