# haproxy 负载代理配置

HAProxy（High Availability Proxy）是一个开源、高性能的 **TCP/HTTP 负载均衡器**和**代理服务器**。以为其专注于负载均衡这一件事情，因此比起 nginx 在负载均衡这件事情上做的更好更专业。

- 支持 tcp/http 两种协议层的负载均衡。
- 支持8种负载均衡算法，尤其在 http 模式下，有许多非常使用的算法。
- 性能优秀，基于单进程处理模式（和 nginx 类似）。
- 拥有出色的监控和监控界面。
- 功能强大的 ACL 支持，实现分发逻辑。



## 安装

~~~bash
# 安装
yum install -y haproxy

# 启动
systemctl start haproxy
~~~





## haproxy实现七层负载

在七层代理服务器上编辑 haproxy 的配置文件 `/etc/haproxy/haproxy.cfg`，配置如下信息。记得做配置的文件备份工作。然后重启 haproxy 服务。

~~~bash
global # 全局参数
    log         127.0.0.1 local2 info # 日志服务器
    pidfile     /var/run/haproxy.pid
    maxconn     4000   #最大连接数（优先级低于后续的maxconn设置）
    user        haproxy
    group       haproxy
    daemon             #守护进程方式后台运行
    nbproc 1           #工作进程数量  cpu内核是几就写几
defaults # 用于为其他配置段提供默认参数
    mode                    http  #工作模式是http （tcp 是 4 层,http是 7 层）
    log                     global
    retries                 3   #健康检查。3次连接失败就认为服务器不可用，主要通过后面的check检查
    option                  redispatch  #服务不可用后重定向到其他健康服务器。
    maxconn                 4000  #优先级中
    timeout connect         5000  #ha服务器与后端服务器连接超时时间，单位毫秒ms
    timeout client          50000 #客户端超时
    timeout server          50000 #后端服务器超时

frontend  web # haproxy作为负载均衡的配置
    mode                   http          # 七层
    bind                    *:80         # haproxy作为负载均衡对外暴漏的ip和端口
    option                 httplog       # 日志类别 http 日志格式
 
    default_backend httpservers  # 默认的代理的组
 
backend httpservers
    balance     roundrobin	# 轮训
    server  web1 192.168.10.100:80 maxconn 2000 weight 1  check inter 1s rise 2 fall 2
    server  web2 192.168.10.111:80 maxconn 2000 weight 1  check inter 1s rise 2 fall 2
 
# inter表示健康检查的间隔，单位为毫秒 可以用1s等，
# fall代表健康检查失败2回后放弃检查。rise代表连续健康检查成功2此后将认为服务器可用。
# 默认的，haproxy认为服务时永远可用的，除非加上check让haproxy确认服务是否真的可用。
~~~

### 重启服务

~~~bash
systemctl restart haproxy

# 查看 haproxy 监听的端口为配置文件中配置的 80
[root@rocky ~]# netstat -tunlap | grep -w 80
tcp        0      0 0.0.0.0:80              0.0.0.0:*               LISTEN      1558/haproxy
~~~



## haproxy实现四层负载

在四层代理服务器上编辑 haproxy 的配置文件 `/etc/haproxy/haproxy.cfg`，配置如下信息。

~~~bash
global
    log 127.0.0.1 local0 notice
    maxconn 2000
    user haproxy
    group haproxy

defaults
    log     global
    mode    tcp
    option  tcplog
    option  dontlognull
    timeout connect 5000
    timeout client  50000
    timeout server  50000

frontend ft_web
    bind *:80
    default_backend bk_web

backend bk_web
    balance roundrobin
    server web1 192.168.10.101:80 check
    server web2 192.168.10.102:80 check
~~~





## haproxy七层透传客户端IP

只需要在七层负载配置文件中添加 ` option forwardfor`。这个配置项可以在两个地方，

- 配置在 `frontend` 下面，对所有的 server 生效。
- 配置在某个 `backend` 下面，只对当前 `backend` 里面的 server 有效。

~~~bash
# 配置在 frontend 下面
frontend  web # haproxy作为负载均衡的配置
    option forwardfor
    mode                   http       
    bind                    *:80      
    option                 httplog  
 
    default_backend httpservers  # 默认的代理的组
~~~

重启 haproxy 服务，即可完成客户端真实 IP 透传到下游 web 服务。下游 web 服务如果是 nginx 不需要特殊处理就可以从变量 `$http_x_forwarded_for` 获取客户端 IP，需要如下处理才能替换变量 `$remote_addr` 的值为客户端 IP，并且使用 `real_ip_header X-Real-IP` 方是无法替换 `$remote_addr`的值。

~~~bash
set_real_ip_from 0.0.0.0/0; # 信任所有上游源，根据实际情况调整
real_ip_header X-Forwarded-For;   # 将X-Forwarded-For字段包含的IP替换原$remote_addr值
#real_ip_header X-Real-IP;        # 将X-Real-IP字段包含的IP替换原$remote_addr值
real_ip_recursive on;
~~~



## haproxy四层透传客户端IP

四层使用 haproxy 代理，七层也是 haproxy 代理的情况，需要在四层开启 proxy 协议，在七层处理 proxy 协议。然后就可以在 web 服务中拿到客户端真实 IP。

#### 四层开启 proxy

在 `backend` 中的每个 server 上面加上 `send-proxy` 即可。

~~~bash
frontend ft_web
    bind *:80
    default_backend bk_web

backend bk_web
    balance roundrobin
    server web1 192.168.10.101:80 check send-proxy
    server web2 192.168.10.102:80 check send-proxy
~~~

#### 七层处理处理 proxy

和 `option forwardfor` 类似，在 `frontend` 下面使用 `accept-proxy` 就是多所有 server 生效。

~~~bash
frontend  web # haproxy作为负载均衡的配置
    option forwardfor
    mode                   http        
    bind                    *:80  accept-proxy  # 处理 proxy
    option                 httplog 

    default_backend httpservers
~~~

#### 在七层设置 HTTP 头字段（X-Forwarded-For, X-Real-IP）

参数 `option forwardfor` 会自动在请求中添加 `X-Forwarded-For` 头。如果请求中已经存在这个头，HAProxy 会将自己的 `src`（客户端 IP）**追加**到该头的末尾。格式如下：

~~~bash
X-Forwarded-For: original-client-ip, haproxy-ip
~~~

如果要设置 请求头 `X-Real-IP` 的值需要手动 添加配置项 `http-request set-header X-Real-IP %[src]`，它将 `X-Real-IP` 头的值**设置**（覆盖已存在的）为 haproxy 看到的客户端源 IP (`%[src]`)。

~~~bash
frontend http_in
    bind *:80
    mode http # 必须是 mode http
    # 添加 X-Forwarded-For 头，将客户端IP附加到现有列表（如果存在）的末尾
    option forwardfor

    # 设置 X-Real-IP 头，将客户端的真实IP值直接赋予这个头
    http-request set-header X-Real-IP %[src]

    default_backend web_servers
~~~



## haproxy四层+nginx七层透传客户端IP

这个也很简单，只需要四层haproxy配置自己的任务（开启 proxy），nginx 七层配置自己的任务（处理 proxy）。

#### 四层 haproxy 使用 send-proxy

把 sever 设置为 nginx 代理七层的 ip:port

~~~bash
backend bk_web
    balance roundrobin
    server web1 192.168.10.101:8081 check send-proxy
    server web2 192.168.10.102:8081 check send-proxy
~~~

#### 七层 nginx 使用 proxy_protocol

~~~bash
http {
    server {
        listen   8081   proxy_protocol;
        set_real_ip_from 0.0.0.0/0;
        real_ip_header proxy_protocol;
        location / {
            proxy_pass   http://webs;
            # -------------------》只加上下面这一段即可
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_set_header Host $http_host;
            proxy_set_header X-Forwarded-Proto $scheme;
        }
    }
}
~~~



## haproxy在七层使用 ACL 实现动动静分离

在 `frontend` 下面使用 `acl` 指定按照 URL 的匹配结果转发到指定的 server 来处理请求。其中，
- 字段 `url_reg` 表示使用正则匹配 url
- 字段 `path_beg` 表示URL以什么开头
- 字段 `path_end` 表示URL以什么结尾
- 字段 `-i` 表示忽略大小写。
- `use_backend static1 if xxx`  表示 如果满足acl xxx规则，则代理给后端服务器组static

~~~bash
frontend  web
    mode                   http        
    bind                    *:80        
    option                 httplog   
 
    acl xxx   url_reg   -i  \.html$  #acl相当于nginx的location。url_reg定义自己的正则匹配url，-i忽略大小写，下同
    acl xxx   url_reg   -i  \/$  #针对访问末尾不加任何路径的情况例如http://xx.xx.xx        
    #acl xxx  path_beg  -i /static /images /javascript /stylesheets # path_beg匹配路径开头
    #acl xxx  path_end  -i .jpg .gif .png .css .js  # path_end匹配路径结尾
 
    acl yyy   path_end  -i .css .js  # path_end匹配路径结尾
 
    use_backend static1 if xxx  #如果满足acl xxx规则，则代理给后端服务器组static
    use_backend static2 if yyy #如果满足acl yyy规则，则代理给httpservers组 
    default_backend httpservers  #上述规则都匹配失败后默认的代理的组
 
backend static1
    balance     roundrobin
    server      static1_a 192.168.71.14:8080 check
backend static2
    balance     roundrobin
    server      static2_a 192.168.71.15:8080 check
backend httpservers
    balance     roundrobin
    server  myhttp1 192.168.71.16:8080 maxconn 2000 weight 1  check inter 1s rise 2 fall 2
    #server  myhttp2 192.168.71.17:8080 maxconn 2000 weight 1  check inter 1s rise 2 fall 2
~~~



## haproxy在七层的监控服务

~~~bash

listen stats # haproxy自带的状态监控服务
    mode                   http
    bind                   *:81
    stats                  enable
    stats uri              /haproxy    
    stats auth             liuxu:666     #用户认证信息
    monitor-uri      /monitoruri
~~~

上述配置表示，启动服务后，可以在七层负载器上部署一个 http 服务。

在使用浏览器访问 http://<七层服务器IP>:81/haproxy，可以进入网页，输入用户名和密码验证后，可以看到监控页面。



