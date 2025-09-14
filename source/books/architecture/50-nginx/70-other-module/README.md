# 常用模块



## ngx_http_autoindex_module

**目录索引模块**。当客户端请求的 URL 以 `/` 结尾，并且该目录下不存在默认索引文件（如 `index.html`, `index.htm`）时，此模块会自动生成一个列出该目录下所有文件和子目录的 HTML 页面。这个模块默认是编译进 Nginx 的，所以通常你不需要额外安装。

~~~nginx
server {
    listen 80;
    server_name www.autoindex.com;
    charset utf8;

    location / {
    root /code/autoindex;
    
    # 启用或禁用目录列表功能。
    autoindex on;   
    
     # 默认是显示字节大小，配置为off之后，显示具体大小 M/G/K
    autoindex_exact_size off;  
    
    # 默认显示的时间与真实时间相差8小时，所以配置 on
    autoindex_localtime on; 
    }
}
~~~



## ngx_http_access_module

**访问控制模块**。这是一个非常核心且常用的模块，用于基于客户端 IP 地址进行访问控制，也就是我们常说的**IP 黑白名单**。常用在 `server` 块和 `location` 块内。

~~~nginx
server {
    listen 80;
    server_name www.autoindex.com;
    charset utf8;
 
    location / {
        root /code/autoindex;
        
        # 全部禁止访问，但允许指定 ip 可以访问
        allow 10.0.0.1;
        deny all;
    }
    location /abc {
        root /code/abc;
        
        # 全部可以访问，但禁用指定 ip
        deny 10.0.0.1;
        allow all;
    }
}
~~~

>注意：如果使用 `all` 一定要放在最后。



## ngx_http_auth_basic_module

**开启登陆认证的模块**。这是一个用于实现**HTTP基本认证（HTTP Basic Authentication）**的模块，可以为你的网站或特定位置添加一个简单的用户名和密码登录框。

~~~nginx
server {
    listen 80;
    server_name www.autoindex.com;
    charset utf8;
    access_log /var/log/nginx/www.autoindex.com.log main;
 
    location /admin {
        root /code/autoindex;
        # 开启登陆认证
        auth_basic "Admin Area - Please Login";
        # 登陆认证密码文件
        auth_basic_user_file /etc/nginx/auth_basic;
    }
}
~~~

#### 创建密码文件

安装 nginx 后可以使用 `htpasswd` 命令，该命令用来创建密码文件，新增用户和密码。

~~~bash
# 第一次使用需要使用 -c 选项，表示创建文件的意思。
htpasswd -c /etc/nginx/auth_basic liuxu
 
# 第二在再添加用户不需要使用 -c 参数
htpasswd /etc/nginx/auth_basic jack


#密码文件内容
cat /etc/nginx/auth_basic
liuxu:$apr1$A7d4BWYe$HzlIA7pjdMHBDJPuLBkvd/
jack:$apr1$psp0M3A5$601t7Am1BG3uINvuBVbFV0
~~~



## ngx_http_stub_status_module

**nginx 状态监控模块**。应使用--with-http_stub_status_module配置参数启用它，1.26版默认启用，可用nginx -V查看

~~~nginx
server {
    listen 80;
    server_name www.autoindex.com/basic_status

    location / {
        root /code/autoindex;
    }
 		
    # 开启监控
    location = /basic_status {
        stub_status;
    }
}
~~~

#### 浏览器访问

访问地址：http://www.autoindex.com/basic_status

~~~bash
#nginx七种状态
Active connections: 2 
server accepts handled requests
 4 4 21 
Reading: 0 Writing: 1 Waiting: 1 
 
# 累计值
Active connections      #当前活跃的连接数
[root@web01 /usr/share/nginx/html]# netstat -ant |grep 80
tcp 0 0 0.0.0.0:80 0.0.0.0:* LISTEN 
tcp 0 0 192.168.71.112:80 192.168.71.7:65491 ESTABLISHED
tcp 0 0 192.168.71.112:80 192.168.71.7:65492 ESTABLISHED
 
accepts                 #nginx启动开始算，累计接收的TCP连接总数
handled                 #累计成功的TCP连接数
requests                #累计成功的请求数
Reading                 #当前0个链接正在读请求
Writing                 #当前1个链接正在响应数据给客户端
Waiting                 #当前等待的请求数，即当前处于keep-alive状态的连接数
 
# 注意, 一次TCP的连接，可以发起多次http的请求, 如下参数可配置进行验证
keepalive_timeout  0;   # 类似于关闭长连接
keepalive_timeout  65;  # 65s没有活动则断开连接
~~~





## ngx_http_limit_conn_module

**连接限制模块**。这是一个非常重要的**流量控制和连接管理**模块，用于限制来自单个客户端 IP 地址的**并发连接数**。

~~~nginx
#设置一个存储ip地址，空间名字为conn_zone,空间大小为10M的空间
http {
    limit_conn_zone $binary_remote_addr zone=conn_zone:10m;    

    server {
        listen 80;
        server_name www.autoindex.com;
        charset utf8;;
        # 在整个 server 范围内，每个IP最多允许10个并发连接
        limit_conn conn_zone 10;

        location / {
            root /code/autoindex;
            index index.html;
        }
    }
}
~~~



## ngx_http_limit_req_module

**请求限制模块**。这是 Nginx 流量控制体系中至关重要的一环，用于限制**请求的处理速率**，是防御洪水攻击（DDoS、CC攻击）和应用级滥用的核心工具。它使用**漏桶算法（Leaky Bucket Algorithm）** 来实现平滑的流量整形和突发请求处理。

#### 示例 1：基础速率限制（无突发）

对于单个IP，第1到第10个请求正常处理。如果在同一秒内发起第11个请求，Nginx 会立即返回 429 错误。**这种配置非常严格，可能会误伤正常用户的短暂突发操作。**

~~~nginx
http {
    # 定义限制区域：每个IP每秒最多10个请求
    limit_req_zone $binary_remote_addr zone=per_ip:10m rate=10r/s;

    server {
        listen 80;
        server_name example.com;

        location / {
            # 应用限制：无突发缓冲，第11个请求立刻被拒
            limit_req zone=per_ip;
            # 可选：返回429状态码
            limit_req_status 429;

            root /usr/share/nginx/html;
            index index.html;
        }
    }
}
~~~

#### 示例 2：允许突发（排队）

更常用的配置，允许一定量的突发请求，它们会被延迟处理以保证平均速率。速率：平均 ≤ 10 req/s，如果瞬间收到 30 个请求：前 10 个被立即处理（当前的速率容量）。接下来的 20 个进入队列（`burst=20`），按照 `10r/s` 的速率被延迟处理。第 31 个及之后的请求会被立即拒绝（因为桶满了），直到队列中的请求被处理完。

```nginx
http {
    limit_req_zone $remote_addr zone=per_ip:10m rate=10r/s;

    server {
        listen 80;
        server_name example.com;

        location /api/ {
            # 应用限制：允许最多20个请求的突发队列
            limit_req zone=per_ip burst=20;

            proxy_pass http://backend_api;
        }
    }
}
```

#### 示例 3：重点保护登录接口

防止暴力破解密码。

```nginx
http {
    # 对登录接口进行非常严格的限制：每分钟最多5次尝试
    limit_req_zone $binary_remote_addr zone=login:10m rate=5r/m;

    server {
        listen 80;
        server_name example.com;

        location = /wp-login.php {
            # 无突发，超过频率直接拒绝
            limit_req zone=login;
            limit_req_status 429;

            # 记录拒绝日志
            limit_req_log_level warn;

            fastcgi_pass php_handle;
            # ... other fastcgi params ...
        }
    }
}
```



## 补充

- 在限流模块中必须使用 `$binary_remote_addr`，**永远使用 `$binary_remote_addr`**。这是官方文档的推荐做法，也是性能最佳实践。除非你有非常特殊的理由，否则不要使用 `$remote_addr`。
- 使用 `$remote_addr` 的场景，
    1. **用于访问日志 (`access_log`)**：因为日志是需要人类阅读和分析的，字符串格式一目了然。
    2. **`proxy_set_header` 传递 IP**：因为上游服务（如 Python, Go）期望接收的是可读的字符串格式的 IP。



