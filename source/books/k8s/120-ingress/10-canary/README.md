# 金丝雀发布

## 发布策略

#### 滚动发布

就一套环境，在升级过程中，并不是一下子启用所有新版本，而是先启动一个新的，再停一个老版本，一点点向新版本滚动，直到完成全部升级。
如果环境中有10台机器，只需要新增1台机器，就可以滚动起来了

	优点：
	 1、节省机器、最小化停机时间、平滑过渡
	
	缺点：
	1、新老版本共存，在整个升级过程中极不稳定，一旦访问出问题，无法确定是新版本的问题还是老版本的问题。关键是缺少对流量的控制

#### 蓝绿发布（蓝老绿新）

环境维度，以整套环境基本单位来进行流量分发。

核心是部署两套环境，两套环境的机器配置都一样。老环境部署老版本，新环境部署新版本。

在两套环境的基础上做流量的分配，旧80%，新20%，然后逐步增加新环境的流量。如果原环境中有10台机器，新环境也需要有10台机器。

	优点：随时切换，快速切回老环境，也能抗住压力，因为老环境原封不动的放着呢
	缺点：耗费资源

#### 金丝雀发布

有时候会与灰度发布代表一个意思，总体来说都是小范围尝试，然后逐步扩大，站在这个维度很多人会将金丝雀发布与灰度当成一个意思，但细说的话有用法层面的区别。

流量维度，站在整体流量的维度进行分配，没有细分用户特征。蓝绿有两套一样配置的环境，流量分分配是以一个整个环境为单位分发的。

金丝雀，先部署少数几台新版本，然后采用流量的控制的方式，逐步一点一点扩大范围，适用于迭代版本，而不是大版本更新。

#### 灰度发布

用户群体特征，从所有用户中选出一些充当小白鼠。

关键区别：金丝雀发布的流量分配不区分用户

#### 总结

金丝雀发布可以被视为灰度发布的一种实现方式，但灰度发布的范围更广，涵盖了更多逐步发布的策略。



## ingress 实现金丝雀发布

在 k8s 中，使用 Ingress 实现金丝雀发布的核心：通过为 Ingress 资源添加特定的注解（Annotation），将少量、可控的流量逐步引导至新版本的应用。最常见的实现方式是基于 Nginx Ingress，它提供了灵活、简单的金丝雀流量规则。

#### 流量控制的两类方式

可以选择如下两种基础策略来分配流量：

| 策略类型              | 核心注解（Nginx Ingress）                                    | 适用场景                                                     |
| :-------------------- | :----------------------------------------------------------- | :----------------------------------------------------------- |
| **基于权重 (Weight)** | `nginx.ingress.kubernetes.io/canary: "true"` `nginx.ingress.kubernetes.io/canary-weight: "10"` | **最常用**。适用于根据百分比（如10%）随机分发流量，用于初步验证新版本的稳定性和性能。 |
| **基于用户请求**      | `nginx.ingress.kubernetes.io/canary: "true"` `nginx.ingress.kubernetes.io/canary-by-header: "X-Canary"` `nginx.ingress.kubernetes.io/canary-by-header-value: "true"`<br>`nginx.ingress.kubernetes.io/canary-by-cookie: "user-value"` | **用于精准测试**。只将带有特定请求头或者 cookie 值的请求路由到新版本，适用于向内部测试人员或特定用户群开放新功能。 |

> 注：**不同Ingress控制器的注解会有所不同**。例如，阿里云ALB Ingress的注解是 `alb.ingress.kubernetes.io/canary-weight`，而Nginx Ingress使用的是 `nginx.ingress.kubernetes.io/canary-weight`。你需要根据自己集群中安装的Ingress控制器来调整。



#### 基于Nginx Ingress 的权重发布

核心在 ingress 对象的注解中使用如下两个配置

~~~
nginx.ingress.kubernetes.io/canary: 'true'       # 启用 Canary
nginx.ingress.kubernetes.io/canary-weight: '30'  # 分配30%流量到新版本
~~~

##### 第一步：准备业务服务 svc 、 pod、ingress

~~~yaml
# app.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: production
  name: production
spec: 
  replicas: 1  # 为了测试方便，设置1个副本
  selector: 
    matchLabels:
      app: production     
  strategy: {}
  template:                
    metadata:
      labels:
        app: production
    spec:                  
      containers:
      - image: nginx:latest
        name: nginx
---
apiVersion: v1
kind: Service
metadata:
  labels:
    app: production
  name: production
spec:
  ports:
  - port: 9999
    protocol: TCP
    targetPort: 80
  selector:
    app: production
  type: ClusterIP
---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: production
spec:
  ingressClassName: nginx
  rules:
    - host: test.ingress.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: production
                port:
                  number: 9999
~~~

##### 第二步：上线业务服务

~~~bash
[root@k8s-master-01 test2]# kubectl apply -f app.yaml 
deployment.apps/production created
service/production created
ingress.networking.k8s.io/production created
[root@k8s-master-01 test2]# 
[root@k8s-master-01 test2]# 
[root@k8s-master-01 test2]# kubectl get ingress -w
NAME         CLASS   HOSTS              ADDRESS   PORTS   AGE
production   nginx   test.ingress.com             80      12s
production   nginx   test.ingress.com   172.16.143.23,172.16.143.24   80      37s
~~~

##### 第三步：准备新版本的业务服务

容器的镜像版本更新，新的 svc 和旧的 svc 名称需要不一样。

~~~yaml
# canary.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: canary
  name: canary
spec: 
  replicas: 1
  selector: 
    matchLabels:
      app: canary     
  strategy: {}
  template:                
    metadata:
      labels:
        app: canary
    spec:                  
      containers:
      - image: nginx:latest # 采用新镜像，新镜像里有新代码(为了测试继续使用latest版本)
        name: nginx
---
apiVersion: v1
kind: Service
metadata:
  labels:
    app: canary
  name: canary		# svc 是新的名称
spec:
  ports:
  - port: 9999
    protocol: TCP
    targetPort: 80
  selector:
    app: canary
  type: ClusterIP
~~~

##### 第四步：新服务跑起来

~~~bash
[root@k8s-master-01 test2]# kubectl apply -f canary.yaml 
deployment.apps/canary created
service/canary created
[root@k8s-master-01 test2]# 
[root@k8s-master-01 test2]# 
[root@k8s-master-01 test2]# kubectl get svc
NAME         TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)    AGE
canary       ClusterIP   10.97.91.44    <none>        9999/TCP   5s
kubernetes   ClusterIP   10.96.0.1      <none>        443/TCP    17d
production   ClusterIP   10.97.51.185   <none>        9999/TCP   6m14s
~~~

##### 第五步：编写新的 ingress 实现 金丝雀发布

~~~yaml
# c1.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: canary
  annotations:
    # 要开启金丝雀发布机制，首先需要启用 Canary
    nginx.ingress.kubernetes.io/canary: 'true'
    # 分配30%流量到当前Canary版本
    nginx.ingress.kubernetes.io/canary-weight: '30'
spec:
  ingressClassName: nginx
  rules:
    - host: test.ingress.com	# 这个和旧版本的配置都是一样的
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: canary	# 只是新服务的名字不一样
                port:
                  number: 9999
~~~

应用新的 ingress 

~~~bash
[root@k8s-master-01 test2]# kubectl apply -f  c1.yaml 
ingress.networking.k8s.io/canary created
[root@k8s-master-01 test2]# 
[root@k8s-master-01 test2]# 
[root@k8s-master-01 test2]# kubectl get ingress
NAME         CLASS   HOSTS              ADDRESS                       PORTS   AGE
canary       nginx   test.ingress.com   172.16.143.23,172.16.143.24   80      15s
production   nginx   test.ingress.com   172.16.143.23,172.16.143.24   80      10m
~~~

如上新 ingress 对象应用后，会往 ingresss pod 中插入一个新配置。upstream 中有两个 server，反向代理的新服务的流量控制权重为 30%



#### 基于Nginx Ingress 的请求头发布

核心在 ingress 对象的注解中使用如下四个配置

~~~
nginx.ingress.kubernetes.io/canary: 'true'       # 启用 Canary
nginx.ingress.kubernetes.io/canary-weight: '30'  # 分配30%流量到新版本
nginx.ingress.kubernetes.io/canary-by-header: canary	# 设置请求头的key
nginx.ingress.kubernetes.io/canary-by-header-value: user-value  # 自定义请求头的value
~~~

编写新版的  ingress 对象的配置清单

~~~yaml
# c2.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: canary
  annotations:
    # 启用金丝雀发布机制
    nginx.ingress.kubernetes.io/canary: 'true'
    # 请求头匹配不满足时默认使用权限控制流量
    nginx.ingress.kubernetes.io/canary-weight: '30'
    nginx.ingress.kubernetes.io/canary-by-header: canary
    nginx.ingress.kubernetes.io/canary-by-header-value: xxx
    
spec:
  ingressClassName: nginx
  rules:
    - host: test.ingress.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: canary
                port:
                  number: 9999
~~~

详解如下：

- 如果设置了 `canary-by-header` 和 `canary-weight` 则表示 **基于请求头优先、权重为后备**的智能金丝雀发布策略。
- 如果设置了 `canary-by-header` 没有设置  `canary-by-header-value`，那么请求头的值可以有如下设置情况。
  - `curl -H "canary: never" test.ingress.com` 值为 **never**，表示流量全部到旧服务
  - `curl -H "canary: always" test.ingress.com` 值为 **alwasy**，表示流量全部到新服务
  - 只要不满足上述两种情况，则不使用 基于请求头的方式控制流量。
- 如果设置了 `canary-by-header-value` 满足自定义的请求头值的请求全部到新服务。比如设置 `canary-by-header-value: xxx`，则如下请求全部到新服务。
  - `curl -H "canary: xxx" test.ingress.com`，表示流量全部到新服务



#### 基于Nginx Ingress 的 cookie 发布

核心在 ingress 对象的注解中使用如下三个配置。请求头和 cookie 只能使用一种，不要同时使用。

~~~
nginx.ingress.kubernetes.io/canary: 'true'       # 启用 Canary
nginx.ingress.kubernetes.io/canary-weight: '30'  # 分配30%流量到新版本
nginx.ingress.kubernetes.io/canary-by-cookie: 'canary'	 # 设置cookie的key
~~~

编写新版的  ingress 对象的配置清单

~~~yaml
# c3.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: canary
  annotations:
    nginx.ingress.kubernetes.io/canary: 'true' # 要开启金丝雀发布机制，首先需要启用 Canary
    nginx.ingress.kubernetes.io/canary-by-cookie: 'canary'
    nginx.ingress.kubernetes.io/canary-weight: '30' # 分配30%流量到当前Canary版本
spec:
  ingressClassName: nginx
  rules:
    - host: test.ingress.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: canary
                port:
                  number: 9999
~~~

详解如下：

- 如果设置了 `canary-by-cookie` 和 `canary-weight` 则表示 **基于cookie优先、权重为后备**的智能金丝雀发布策略。

-  `canary-by-cookie` 只是设置 cookie 的 key，value 值可以有如下三种情况

  - `curl -b "canary: never" test.ingress.com` 值为 **never**，表示流量全部到旧服务
  - `curl -b "canary: always" test.ingress.com` 值为 **alwasy**，表示流量全部到新服务
  - 只要不满足上述两种情况，则不使用 基于 cookie 的方式控制流量。

- 基于 cookie 的方式不支持自定义 cookie key 的 value值。



  