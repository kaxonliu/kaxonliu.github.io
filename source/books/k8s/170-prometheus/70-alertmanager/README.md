# altertmanager

**Alertmanager** 是一个独立的告警处理系统，它负责接收来自客户端（如 Prometheus Server）的告警，进行**去重、分组、静默、抑制和路由**，最终通过多种渠道（如邮件、Slack、Webhook 等）发送通知。



核心特性：

-   分组（Grouping）。将同一类型的告警合并为单个通知，避免告警风暴。
- 抑制（Inhibition）。当某些严重告警触发时，抑制其他相关的次要告警。
- 静默（Silence）。临时屏蔽特定告警，用于维护或已知问题。
- 路由（Routing）。根据标签将告警路由到不同的接收器。
- 去重（Deduplication）。相同告警只发送一次通知。

架构与工作原理

~~~bash
Prometheus Server → 告警规则 → 触发告警 → Alertmanager → 通知渠道
        ↑                              ↓
    指标采集                       告警处理
        |                         (分组/抑制/路由)
    数据存储                          ↓
                                 发送通知
~~~



一个报警信息在生命周期内有下面 3 种状态：

- `pending`: 当某个监控指标触发了告警表达式的条件，但还没有持续足够长的时间，即没有超过 `for` 阈值设定的时间，这个告警状态被标记为 `pending``
- `firing`: 当某个监控指标触发了告警条件并且持续超过了设定的 `for` 时间，告警将由pending状态改成 `firing`。
- `inactive`: 当某个监控指标不再满足告警条件或者告警从未被触发时，这个告警状态被标记为 `inactive`

>Prometheus 在 firing 状态下将告警信息发送至 Alertmanager。
>Alertmanager 应用路由规则，将通知发送给配置的接收器，例如邮件。



报警原理

- prometheus server 中配置监控指标和阈值，一旦触发阈值就发送报警信息给 alertmanager
- alertmanager 中配置路由规则，按照路由规则匹配警告信息，匹配到交由指定的 receiver 处理。
- alertmanager  中配置 receivers ，指定接收警告的方式。
- alertmanager  中还可以配置警告信息的分组合并、重复时长、静默、抑制等。



## 安装 altertmanager

>官网二进制包：https://github.com/prometheus/alertmanager/releases
>
>官方文档： https://prometheus.io/docs/alerting/configuration/ 

在 k8s yaml 的方式手动安装。

~~~yaml
# alertmanager.yaml

# service
apiVersion: v1
kind: Service
metadata:
  name: alertmanager
  namespace: monitor
  labels:
    app: alertmanager
spec:
  selector:
    app: alertmanager
  type: NodePort
  ports:
    - name: web
      port: 9093
      targetPort: http

---
# deplyment
apiVersion: apps/v1
kind: Deployment
metadata:
  name: alertmanager
  namespace: monitor
  labels:
    app: alertmanager
spec:
  selector:
    matchLabels:
      app: alertmanager
  template:
    metadata:
      labels:
        app: alertmanager
    spec:
      volumes:
        - name: alertcfg
          configMap:
            name: alert-config		# 关联 cm
      containers:
        - name: alertmanager
          # 换成国内的镜像
          image: prom/alertmanager:v0.30.0
          imagePullPolicy: IfNotPresent
          args:
            - '--config.file=/etc/alertmanager/config.yml'
          ports:
            - containerPort: 9093
              name: http
          volumeMounts:
            - mountPath: '/etc/alertmanager'
              name: alertcfg
          resources:
            requests:
              cpu: 100m
              memory: 256Mi
            limits:
              cpu: 100m
              memory: 256Mi

---
# configmap
apiVersion: v1
kind: ConfigMap
metadata:
  name: alert-config
  namespace: monitor
data:
  config.yml: |-
    # 全局配置
    global:
      # 当alertmanager持续多长时间未接收到告警后标记告警状态为 resolved（解决了）
      resolve_timeout: 5m
      
      # 配置发邮件的邮箱
      smtp_smarthost: 'smtp.163.com:25'
      smtp_from: 'xxx@163.com'
      smtp_auth_username: 'xxx@163.com'
      smtp_auth_password: 'xxxxxx'  # 填入你开启pop3时获得的码
      smtp_hello: '163.com'
      smtp_require_tls: false
      
    # 设置报警的路由分发策略
    route:
      # 定义用于告警分组的标签。当有多个告警消息有相同的 alertname 和 cluster 标签时，这些告警消息将会被聚合到同一个分组中
      # 例如，接收到的报警信息里面有许多具有 cluster=XXX 和 alertname=YYY 这样的标签的报警信息将会批量被聚合到一个分组里面
      group_by: ['alertname', 'cluster']
      # 当一个新的报警分组被创建后，需要等待至少 group_wait 时间来初始化通知，
      # 这种方式可以确保您能有足够的时间为同一分组来获取/累积多个警报，然后一起触发这个报警信息。
      group_wait: 30s
      # 短期聚合: group_interval 确保在短时间内，同一分组的多个告警将会合并/聚合到一起等待被发送，避免过于频繁的告警通知。
      group_interval: 30s
      # 长期提醒: repeat_interval确保长时间未解决的告警不会被遗忘
      # Alertmanager每隔一段时间定期提醒相关人员，直到告警被解决。
      repeat_interval: 120s  # 实验环境想快速看下效果，可以缩小该时间，比如设置为120s
      # 上述两个参数的综合解释：
      #（1）当一个新的告警被触发时，会立即发送初次通知
      #（2）然后开始一个 group_interval 窗口（例如 30 秒）。
      #    在 group_interval 窗口内，任何新的同分组告警会被聚合到一起，但不会立即触发发送。
      #（3）聚合窗口结束后，
      #    如果刚好抵达 repeat_interval 的时间点，聚合的告警会和原有未解决的告警一起发送通知。
      #    如果没有抵达 repeat_interval 的时间点，则原有未解决的报警不会重复发送，直到到达下一个 repeat_interval 时间点。

      # 这两个参数一起工作，确保短时间内的警报状态变化不会造成过多的重复通知，同时在长期未解决的情况下提供定期的提醒。

      # 默认的receiver
      receiver: default 
      
      # 子路由规则
      # 子路由继承父路由的所有属性，可以进行覆盖和更具体的规则匹配。
      routes:
      - receiver: email  #  匹配此子路由的告警将发送到的接收器，该名字也与下面的receivers中定义的name呼应
        group_wait: 10s  #  等待时间，可覆盖父路由的
        group_by: ['instance'] # 根据instance做分组
        match: # 告警标签匹配条件，只有匹配到特定条件的告警才会应用该子路由规则。
          team: node # 只有拥有 team=node 标签的告警才会路由到 email 接收器。
        continue: true  # 把匹配到的报警消息继续送往下一个receiver
    
    # 定义接收器
    receivers: 
    - name: 'default'  # 默认接收器配置，兜底作用
      email_configs:
      - to: 'xxx@qq.com'
        send_resolved: true  # 当告警恢复时是否也发送通知。
        
    - name: 'email'    # 名为email的接收器配置
      email_configs:
      - to: 'xxx@163.com'
        send_resolved: true
~~~

应用后查看 svc 和 pod

~~~bash
[root@k8s-master-01 monitor]# kubectl -n monitor get svc alertmanager 
NAME           TYPE       CLUSTER-IP       EXTERNAL-IP   PORT(S)          AGE
alertmanager   NodePort   10.107.197.114   <none>        9093:30779/TCP   86s
~~~

浏览器访问

~~~bash
http://192.168.10.81:30779
~~~





## 配置警告指标和阈值

在 Prometheus server  中配置警告指标和阈值。

比如监控节点的内存使用率超过 20% 就报警。在 prometheus-cm.yaml 做如下配置。

- 在 prometheus.yml 中配置 alerting，执行 alertmanager 服务
- 在 prometheus.yml 中配置 rule_files 文件的地址，指定警告规则配置文件的路径
- data 下新增 rules.yml 配置文件做为警告规则文件。

~~~yaml
# prometheus-cm.yaml

apiVersion: v1
kind: ConfigMap
metadata:
  name: prometheus-config
  namespace: monitor
data:
  prometheus.yml: |
    global:
      scrape_interval: 15s
      scrape_timeout: 15s
      evaluation_interval: 15s
    alerting:
      alertmanagers:
        - static_configs:
            - targets:
              - alertmanager:9093 
    rule_files:
    - /etc/prometheus/rules.yml 
  rules.yml: |
    - name: test-node-mem
      rules:
      - alert: NodeMemoryUsage 
        expr: (node_memory_MemTotal_bytes - (node_memory_MemFree_bytes + node_memory_Buffers_bytes + node_memory_Cached_bytes)) / node_memory_MemTotal_bytes * 100 > 20
        for: 2m
        labels:
          team: node
          severity: critical
        annotations:
          summary: "{{$labels.instance}}: High Memory usage detected"
          description: "{{$labels.instance}}: Memory usage is above 20% (current value is: {{ $value }}"
~~~

详解说明：

- rules.yml 和 prometheus.yml  是 prometheus server pod 中的配置文件，其中 prometheus.yml是主配置文件，rules.yaml 是告警规则的配置文件。
- 警告规则文件 rules.yml 中可以使用 

#### 报警计算规则的性能优化

基于Recording Rule机制，可以把多条报警规则里共用的表达式。提取出来，由prometheus server周期性计算、计算的结果当成一个指标/时序数据存放下来。如此，其他的报警规则直接共用该现成的结果即可，避免重复的计算的开销。

~~~yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: prometheus-config
  namespace: monitor
data:
  rules.yml: |
    groups:
    - name: recording_rules
      rules:
      - record: job:node_memory_MemFree_bytes:percent 
        expr: (node_memory_MemTotal_bytes - (node_memory_MemFree_bytes + node_memory_Buffers_bytes + node_memory_Cached_bytes)) / node_memory_MemTotal_bytes * 100
    - name: test-node-mem
      rules:
      - alert: NodeMemoryUsage 
        expr: job:node_memory_MemFree_bytes:percent > 20
        for: 2m
        labels:
          team: node
          severity: critical
        annotations:
          summary: "{{$labels.instance}}: High Memory usage detected"
          description: "{{$labels.instance}}: Memory usage is above 20% (current value is: {{ $value }}"
~~~





## 邮件接收报警信息

alertmanager 配置文件中配置邮件信息接收警告。

比如配置 QQ 邮箱

~~~yaml
# alertmanager-config.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: alert-config
  namespace: monitor
data:
  # 自定义邮件消息模板
  template_email.tmpl: |-
    {{ define "email.html" }}
    {{- if gt (len .Alerts.Firing) 0 -}}
        @报警<br>
        {{- range .Alerts }}
        <strong>实例:</strong> {{ .Labels.instance }}<br>
        <strong>概述:</strong> {{ .Annotations.summary }}<br>
        <strong>详情:</strong> {{ .Annotations.description }}<br>
        <strong>时间:</strong> {{ (.StartsAt.Add 28800e9).Format "2006-01-02 15:04:05" }}
        <br>
        {{- end -}}
    {{- end }}
    {{- if gt (len .Alerts.Resolved) 0 -}}
        @恢复<br>
        {{- range .Alerts }}
        <strong>实例:</strong> {{ .Labels.instance }}<br>
        <strong>信息:</strong> {{ .Annotations.summary }}<br>
        <strong>恢复:</strong> {{ (.StartsAt.Add 28800e9).Format "2006-01-02 15:04:05" }}
        <br>
        {{- end -}}
    {{- end }}
    {{ end }}
  config.yml: |-
    # 指定使用模板文件
    templates:
    - '/etc/alertmanager/template_email.tmpl'
    
    global:
      resolve_timeout: 5m
      
      # 配置发邮件的邮箱
      smtp_smarthost: 'smtp.qq.com:465'
      smtp_from: 'xxx@qq.com'
      smtp_auth_username: 'xxx@qq.com'
      smtp_auth_password: 'iyilxxxhbxagh'  # 填入你开启pop3时获得的码
      smtp_require_tls: false
      
    route:
      group_by: ['alertname', 'cluster']
      group_wait: 30s
      group_interval: 30s
      repeat_interval: 120s 
      receiver: default 
      routes: 
      - receiver: email 
        group_wait: 10s 
        group_by: ['instance'] 
        match: 
          team: node # 只有拥有 team=node 标签的告警才会路由到 email 接收器。
        continue: true  # 把匹配到的报警消息继续送往下一个receiver
      
    receivers: 
    - name: 'default' 
      email_configs:
      - to: 'xx@qq.com'
        send_resolved: true
    - name: 'email' 
      email_configs:
      - to: 'xx@qq.com'
        send_resolved: true
        html: '{{ template "email.html" . }}'
~~~



当某些严重告警触发时，抑制其他相关的次要告警

比如 prometheus server 配置了两个相关的警告指标

- NodeMemoryUsage （假定为主要警告）标签为：`team=node`、`severity=critical`
- NodeLoad（假定为次要警告）标签为：`team: node`、`severity=normal`

~~~yaml
# prometheus-cm.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: prometheus-config
  namespace: monitor
data:
  rules.yml: |
    groups:
    - name: recording_rules
      rules:
      - record: job:node_memory_MemFree_bytes:percent 
        expr: (node_memory_MemTotal_bytes - (node_memory_MemFree_bytes + node_memory_Buffers_bytes + node_memory_Cached_bytes)) / node_memory_MemTotal_bytes * 100
    - name: test-node-mem
      rules:
      - alert: NodeMemoryUsage 
        expr: job:node_memory_MemFree_bytes:percent > 20
        for: 2m
        labels:
          team: node
          severity: critical
        annotations:
          summary: "{{$labels.instance}}: High Memory usage detected"
          description: "{{$labels.instance}}: Memory usage is above 20% (current value is: {{ $value }}"
    - name: test-node-load
      rules:
        - alert: NodeLoad
          expr: node_load5 < 1  # 故意设置这个值，让它报警，正常应该是超过某个值才报警，而不是小于
          for: 2m
          labels:
            team: node
            severity: normal
          annotations:
            summary: '{{ $labels.instance }}: Low node load deteched'
            description: '{{ $labels.instance }}: node load is below 1 (current value is: {{ $value }})'
  prometheus.yml: |
    global:
      scrape_interval: 15s 
      scrape_timeout: 15s 
      evaluation_interval: 15s 
    alerting:
      alertmanagers:
        - static_configs:
            - targets:
              - alertmanager:9093 
    rule_files:
    - /etc/prometheus/rules.yml 
~~~



在 alertmanager 配置抑制告警规则

~~~yaml
# alertmanager-config.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: alert-config
  namespace: monitor
data:
  config.yml: |-
    # 通过 inhibit_rules 配置规则的依赖关系
    inhibit_rules:
     # source_match 配置主要报警（源告警）
     # target_match 配置次要报警（目标告警）
     # equal 用于指定标签匹配的条件。它决定了源告警（source）和目标告警（target）之间在哪些标签上需要值相等，才会触发抑制。
      - source_match:
          alertname: NodeMemoryUsage  # 这个标签是自动加上去的
          severity: critical	# 这个标签是手动加的
        target_match:
          severity: normal  # 这个标签是手动加的
        equal:
          - instance      # instance 是每条报警规则自带的标签，值为对应的节点名
~~~

如上规则表明：

- **源告警**：`NodeMemoryUsage` 且 `severity: critical`
- **目标告警**：`severity: normal`
- **抑制条件**：**只有当两个告警来自同一个 `instance`（节点）时**，才会抑制





## 钉钉接收报警信息

基于webhook对接钉钉报警流程

~~~
prometheus（报警规则）----》alertmanager组件------钉钉的webhook软件------》钉钉
~~~



#### 钉钉部分配置

下载钉钉，官网：https://www.dingtalk.com/

新建群，并且在群内创建自定义机器人，获得两个参数：

- secret 密钥 SEC639256e512ce744fa5fb8a7941c8325cc812913c5dc0da8aa51ed03783497c65
- webhook url  https://oapi.dingtalk.com/robot/send?access_token=fc174dcf1d2ff6233c634c5a02f004ee0e91888e71bcc7804f5319d43b688622

使用如下 python 脚本测试是否可用。

~~~python
#python 3.8
import time
import sys
import hmac
import hashlib
import base64
import urllib.parse
import requests


# 替换成自己的值
secret = 'SEC6...c65'
webhook_url =' https://oapi.dingtalk.com/robot/send?access_token=fc1.....945'

timestamp = str(round(time.time() * 1000))
secret_enc = secret.encode('utf-8')
string_to_sign = '{}\n{}'.format(timestamp, secret)
string_to_sign_enc = string_to_sign.encode('utf-8')
hmac_code = hmac.new(secret_enc, string_to_sign_enc, digestmod=hashlib.sha256).digest()
sign = urllib.parse.quote_plus(base64.b64encode(hmac_code))
print(timestamp)
print(sign)

MESSAGE = sys.argv[1]
request_url =f'{webhook_url}&timestamp={timestamp}&sign={sign}'
response = requests.post(request_url, headers={'Content-Type': 'application/json'},json={"msgtype": "text","text": {"content":f"'{MESSAGE}'"}})
print(response.text)
print(response.status_code)
~~~



#### 二进制安装部署钉钉 webhook

prometheus-webhook-dingtalk官网：https://github.com/timonwong/prometheus-webhook-dingtalk

下载地址：https://github.com/timonwong/prometheus-webhook-dingtalk/releases

**安装 webhook** 

~~~bash
# 下载
wget https://github.com/timonwong/prometheus-webhook-dingtalk/releases/download/v2.1.0/prometheus-webhook-dingtalk-2.1.0.linux-amd64.tar.gz

# 解压
tar xf prometheus-webhook-dingtalk-2.1.0.linux-amd64.tar.gz

# 移动
mv prometheus-webhook-dingtalk-2.1.0.linux-amd64 /usr/local/

# 软连接
ln -s /usr/local/prometheus-webhook-dingtalk-2.1.0.linux-amd64 /usr/local/prometheus-webhook-dingtalk

# 配置文件
cat > /usr/local/prometheus-webhook-dingtalk/config.yml << "EOF"
# 可以定义多个target，下面的webhook1、webhook_mention_all、webhook_mention_users是你定义的名字，访问该webhook时会用到
targets:
  webhook1:
    url: https://oapi.dingtalk.com/robot/send?access_token=3a...d8b
    secret: SEC...aea0
    # 设置@人员
    # mention:
    #   all: true
    # mention:
    #   mobiles: ['134---5678']
EOF

# 做成系统服务
cat > /lib/systemd/system/dingtalk.service << 'EOF'
[Unit]
Descripton=dingtalk
Documentation=https://github.com/timonwong/prometheus-webhook-dingtalk/
After=network.target

[Service]
Restart=on-failure
WorkingDirectory=/usr/local/prometheus-webhook-dingtalk
ExecStart=/usr/local/prometheus-webhook-dingtalk/prometheus-webhook-dingtalk --web.listen-address=0.0.0.0:8060 --config.file=/usr/local/prometheus-webhook-dingtalk/config.yml

[Install]
WantedBy=multi-user.target

EOF

# 启动
systemctl daemon-reload
systemctl start dingtalk
systemctl status dingtalk
~~~

**对接 alertmanager**

修改 alertmanager 的配置文件。增加 webhook 类型的 receiver 并指定使用 路由发送警告信息。

~~~yaml
# alertmanager-config.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: alert-config
  namespace: monitor
data:
  config.yml: |-
    global:
      resolve_timeout: 5m
      
    route:
      group_by: ['alertname', 'cluster']
      group_wait: 30s
      group_interval: 30s
      repeat_interval: 120s
      receiver: default 
		
	  # 子路由规则
      routes:
      - receiver: email
        group_wait: 10s
        group_by: ['instance'
        match:
          team: node			# 因为这两个子路由都是使用 team=node筛选警告信息的
        continue: true			# continue: true 的作用是邮件发送报警信息后，继续使用 mywebhook 发送报警信息
      - receiver: mywebhook		# 使用 webhook
        group_wait: 10s
        group_by: ['instance']
        match:
          team: node 
   
    receivers: 
    - name: 'default'  # 默认接收器配置，未匹配任何特定路由规则的告警会发送到此接收器。
      email_configs:
      - to: 'xxx@qq.com'
        send_resolved: true  # : 当告警恢复时是否也发送通知。
        
    - name: 'mywebhook'
      webhook_configs:
      # webhook 二进制程序部署在192.168.10.81 机器上
      # url路径中的 webhook1 就是在 /usr/local/prometheus-webhook-dingtalk/config.yml 配置的 target 的名字
      - url: 'http://192.168.10.81:8060/dingtalk/webhook1/send'
~~~



#### k8s 安装部署钉钉 webhook

**创建 svc** 

因为是内部使用，不需要使用 nodeport 类型的 svc

~~~yaml
apiVersion: v1
kind: Service
metadata:
  name: promoter
  namespace: monitor
  labels:
    app: promoter
spec:
  selector:
    app: promoter
  ports:
    - port: 8080
      targetPort: 8060
~~~

**创建 deployment**

~~~yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: promoter
  namespace: monitor
  labels:
    app: promoter
spec:
  selector:
    matchLabels:
      app: promoter
  template:
    metadata:
      labels:
        app: promoter
    spec:
      volumes:
      - name: promotercfg
        configMap:
          name: promoter-conf	# 使用configmap
      containers:
        - name: promoter
          # 换成国内的镜像
          image: timonwong/prometheus-webhook-dingtalk:v2.1.0
          imagePullPolicy: IfNotPresent
          args:
            - --web.listen-address=:8060
            - '--config.file=/etc/promoter/config.yaml'
          ports:
            - containerPort: 8060
          volumeMounts:
            - mountPath: '/etc/promoter'
              name: promotercfg
~~~

**创建 cm**

~~~yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: promoter-conf
  namespace: monitor
data:
  # 使用自定义报警模板
  template.tmpl: |
    {{ define "default.tmpl" }}
 
    {{- if gt (len .Alerts.Firing) 0 -}}
    {{- range $index, $alert := .Alerts -}}
 
    ============ = **<font color='#FF0000'>告警</font>** = =============
    ![警报 图标](https://egonimages.oss-cn-beijing.aliyuncs.com/gaojing1.jpg)

    **Egon test**
    **告警名称:**    {{ $alert.Labels.alertname }}   
    **告警级别:**    {{ $alert.Labels.severity }} 级   
    **告警状态:**    {{ .Status }}   
    **告警实例:**    {{ $alert.Labels.instance }} {{ $alert.Labels.device }}   
    **告警概要:**    {{ .Annotations.summary }}   
    **告警详情:**    {{ $alert.Annotations.message }}{{ $alert.Annotations.description}}   
    **故障时间:**    {{ ($alert.StartsAt.Add 28800e9).Format "2006-01-02 15:04:05" }}  
    ============ = end = =============  
    {{- end }}
    {{- end }}
 
    {{- if gt (len .Alerts.Resolved) 0 -}}
    {{- range $index, $alert := .Alerts -}}
 
    ============ = <font color='#00FF00'>恢复</font> = =============   #绿色字体
 
    **告警实例:**    {{ .Labels.instance }}   
    **告警名称:**    {{ .Labels.alertname }}  
    **告警级别:**    {{ $alert.Labels.severity }} 级   
    **告警状态:**    {{   .Status }} 
    **告警概要:**    {{ $alert.Annotations.summary }}  
    **告警详情:**    {{ $alert.Annotations.message }}{{ $alert.Annotations.description}}  
    **故障时间:**    {{ ($alert.StartsAt.Add 28800e9).Format "2006-01-02 15:04:05" }}  
    **恢复时间:**    {{ ($alert.EndsAt.Add 28800e9).Format "2006-01-02 15:04:05" }}  
 
    ============ = **end** = =============
    {{- end }}
    {{- end }}
    {{- end }}
  
  # 主配置文件
  # targets 中替换自己的 url 和 secret
  # 
  config.yaml: |
    templates:
      - /etc/promoter/template.tmpl
    targets:
      webhook1:
        url: https://oapi.dingtalk.com/robot/send?access_token=3a...d8b
        secret: SEC67...1aea0
        # 哪个target 需要引用模版，就增加这一小段配置，其中default.tmpl就是你一会要定义的模版
        message:  # 哪个target需要引用模版，就增加这一小段配置，其中default.tmpl就是你要定义的模版
          text: |
            {{ template "default.tmpl" . }}
        # 如下设置艾特哪些人员
        # mention:
        #   all: true
        # mention:
        #   mobiles: ["17611111234", "13411115678"]
~~~

如果想要艾特所有人

~~~yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: promoter-conf
  namespace: monitor
data:
  template.tmpl: |
  {{ define "default.tmpl" }}
  ...
  ...
  config.yaml: |
    templates:
      - /etc/promoter/template.tmpl
    targets:
      webhook1:
        url: https://oapi.dingtalk.com/robot/send?access_token=3a...d8b
        secret: SEC67...1aea0
        message:
          text: {{ template "default.tmpl" . }}
        mention:
          all: true
~~~

如果想要艾特个别人员

~~~yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: promoter-conf
  namespace: monitor
data:
  template.tmpl: |
  {{ define "default.tmpl" }}
  ...
  ...
  config.yaml: |
    templates:
      - /etc/promoter/template.tmpl
    targets:
      webhook1:
        url: https://oapi.dingtalk.com/robot/send?access_token=3a...d8b
        secret: SEC67...1aea0
        message:
          text: |
            {{ template "default.tmpl" . }}
            @17611111234 @13411115678
        mention:
          mobiles: ["17611111234", "13411115678"]
~~~

>补充 报警图片
>
>~~~
>https://egonimages.oss-cn-beijing.aliyuncs.com/gaojing1.jpg
>https://egonimages.oss-cn-beijing.aliyuncs.com/gaojing2.jpg
>https://egonimages.oss-cn-beijing.aliyuncs.com/gaojing3.png
>https://egonimages.oss-cn-beijing.aliyuncs.com/gaojing4.jpg
>https://egonimages.oss-cn-beijing.aliyuncs.com/gaojing5.jpg
>https://egonimages.oss-cn-beijing.aliyuncs.com/gaojing6.png
>~~~



**对接 alertmanager**

修改 alertmanager 的配置文件。增加 webhook 类型的 receiver 并指定使用 路由发送警告信息。

~~~yaml
# alertmanager-config.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: alert-config
  namespace: monitor
data:
  config.yml: |-
    global:
      resolve_timeout: 5m
      
    route:
      group_by: ['alertname', 'cluster']
      group_wait: 30s
      group_interval: 30s
      repeat_interval: 120s
      receiver: default 
		
	  # 子路由规则
      routes:
      - receiver: email
        group_wait: 10s
        group_by: ['instance'
        match:
          team: node			# 因为这两个子路由都是使用 team=node筛选警告信息的
        continue: true			# continue: true 的作用是邮件发送报警信息后，继续使用 mywebhook 发送报警信息
      - receiver: mywebhook		# 使用 webhook
        group_wait: 10s
        group_by: ['instance']
        match:
          team: node 
   
    receivers: 
    - name: 'default'  # 默认接收器配置，未匹配任何特定路由规则的告警会发送到此接收器。
      email_configs:
      - to: 'xxx@qq.com'
        send_resolved: true  # : 当告警恢复时是否也发送通知。
        
    - name: 'mywebhook'
      webhook_configs:
      # webhook 直接使用svc名字加端口即可
      # url路径中的 webhook1 就是在 /usr/local/prometheus-webhook-dingtalk/config.yml 配置的 target 的名字
      - url: 'http:/promoter:8080/dingtalk/webhook1/send'
~~~

