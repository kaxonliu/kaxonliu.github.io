# 配置 server

**本节介绍在 `http` 块内部配置 `server` 块**。一个 `http` 块中可以包含多个 `server` 块，可以在主配置文件中直接定义 `server` ，也可以在 `/etc/nginx/conf.d/*.conf` 文件中定义。后者要求在 `http` 块内使用 `include` 导入配置文件（`/etc/nginx/conf.d/*.conf`）

~~~nginx
http {
    include /etc/nginx/conf.d/*.conf;
    
    server {
        listen    80;
        listen [::]:80;
        server_name  www.kaxonliu.top;
        
        root /usr/share/nginx/html; 
        index index.html index.htm;
    
        error_page 404 /404.html;
        location = /404.html {
        }

        error_page 500 502 503 504 /50x.html;
        location = /50x.html {
        }
}
~~~



## 字段 listen

指定服务器监听的**IP地址和端口**。不写 ip 默认监听所有地址。

~~~nginx
listen 80;               # 监听所有IPv4地址的80端口。
listen 443 ssl;          # 监听所有IPv4地址的443端口，并启用SSL。
listen [::]:80;          # 监听所有IPv6地址的80端口（[::] 是IPv6的通配符）。
listen 127.0.0.1:8080;   # 只监听本地回环地址的8080端口。
~~~



## 字段 server_name

字段 `server_name` 用来设置虚拟主机的域名。Nginx 用它来匹配请求头中的 `Host` 字段，决定使用哪个 `server` 块。

~~~nginx
server {
    server_name www.kaxonliu.top kaxonliu.top;	# 精准匹配
    server_name *.example.com;                  # 通配符匹配
    server_name ~^www\d+\.example\.com$;        # 正则匹配
}

# 通配符匹配（以 * 开头或结尾）
  server_name *.example.com; 匹配 a.example.com, b.a.example.com (多级子域名也匹配)
  server_name www.example.*; 匹配 www.example.com, www.example.org

# 正则表达式匹配（以 ~ 开头）:
  server_name ~^www\d+\.example\.com$; 匹配 www1.example.com, www123.example.

# 匹配优先级
  精确匹配 > 最长通配符匹配（开头） > 最长通配符匹配（结尾） > 第一个匹配的正则表达式。
~~~

#### 补充：虚拟主机

一个 `http` 块内可以定义多个 `server` ，表示多个虚拟主机。在 Nginx 里，“虚拟主机”就是一台物理服务器上“虚拟”出来的多台网站主机。对浏览器来说，它们各自是独立的站点；对服务器来说，它们只是**同一份 Nginx 配置文件里的不同 server{} 块**。

**三类虚拟主机**

| 类型     | 区分方式                 | 适用场景             | 示例 listen 指令                |
| -------- | ------------------------ | -------------------- | ------------------------------- |
| 基于域名 | 请求头里的 Host 字段     | 一台机子跑多个网站   | `listen 80;`                    |
| 基于端口 | 不同端口号               | 内部服务、测试环境   | `listen 8080;` / `listen 9000;` |
| 基于 IP  | 网卡上绑定的多个 IP 地址 | 老架构、SSL 早期方案 | `listen 192.168.1.10:80;`       |



## 字段 location

指令 `location` 在 `server` 块中用于根据请求的 URI（即域名后的路径部分）来匹配和处理请求。简单来说，它的作用就是：**当客户端请求的 URI 匹配某个特定路径时，告诉 Nginx 应该去哪里找文件、执行什么操作，或者将请求转发给哪里**。

~~~nginx
location [匹配规则] [匹配的URI] {
    # 处理指令，例如：
    root   /path/to/files;
    index  index.html index.htm;
    proxy_pass http://backend_servers;
    # ... 其他指令
}
~~~

#### 前缀匹配之精准匹配（`=`）

只匹配完全相等的 URI。优先级最高。匹配的 URI 必须以 `/` 开头。

~~~nginx
location = /login {
    # 只有当请求的 URI 严格等于 “/login” 时，才会进入这个区块
    # 例如：https://example.com/login
    # 不会匹配：/login/ 或 /login.html
    proxy_pass http://auth_server;
}
~~~

#### 前缀匹配之最佳匹配（`^~`）

如果 URI 匹配这个长前缀，Nginx 会停止搜索其他正则表达式 `location`，直接使用此配置。其优先级高于正则匹配。匹配的 URI 必须以 `/` 开头。

~~~nginx
location ^~ /images/ {
    # 对于以 /images/ 开头的 URI，即使后面有能匹配的正则 location，也优先使用这个
    root /special/storage;
}
~~~

#### 前缀匹配之普通匹配

匹配以指定字符串开头的 URI。这是最常用的匹配方式。匹配的 URI 必须以 `/` 开头。

~~~nginx
location /static/ {
    # 匹配任何以 “/static/” 开头的 URI
    # 例如：/static/css/style.css, /static/js/app.js, /static/images/logo.png
    root /data/www;
    # 最终文件路径为：/data/www/static/css/style.css
}
~~~

>**注意**： 最佳匹配和普通匹配的 匹配 URI 不能一样，nginx 启动会报错说重复。



#### 正则表达式匹配（`~` 和 `~*`）

使用强大的正则表达式进行复杂、灵活的匹配。优先级低于精确匹配，但高于普通前缀匹配。

- `~` 区分大小写的正则匹配。
- `~*` 不区分大小写的正则匹配。

~~~nginx
location ~ \.(gif|jpg|png|js|css)$ {
    # 匹配所有以 .gif, .jpg, .png, .js, .css 结尾的请求
    root /data/cache;
    expires 30d; # 设置缓存过期时间
}

location ~* \.php$ {
    # 不区分大小写地匹配所有 .php 结尾的请求（如 .PHP, .Php）
    # 例如：/index.php, /api/User.PHP
    proxy_pass http://php_backend;
}
~~~

#### 匹配过程和逻辑

匹配原则为：先是用相对明确的前缀匹配，再使用相对模糊的正则匹配。

匹配过程：

1. 第一阶段，使用精确匹配（`=`），如果 URI 和匹配 URI 完全一样，则匹配成功就立即使用。
2. 第二阶段，使用前缀匹配的最佳匹配（`^~`）和普通匹配（无修饰符）来比对 URI，哪个匹配的更精确就使用哪个。
    - 如果最佳匹配（`^~`）更精确，则立即使用，不再关心后面是否有正则匹配模式。
    - 如果普通匹配（无修饰符）更精确，则还要再使用正则表达式比对 URI。
3. 第三阶段，使用正则表达式匹配 URI，按照配置时的顺序从上往下依次比对请求 URI，第一个匹配成功的正则表达式会被使用。
4. 第四阶段，所有正则表达式都匹配失败，会选择满足匹配要求的普通匹配。

#### 注意：

- 正则表达式匹配的定义有先后顺序要求。
- 其他类型的匹配模式和正则表达式相比，没有配置时先后顺序的要求。





## 字段 root 和 index

这俩字段可以放在 `server` 块内，作为整个 server 的全局默认配置。但更**推荐在 `location` 内使用**，`location` 内部使用可以覆盖全局配置。

**配置在 server 内**

~~~nginx
server {
    root /var/share/nginx/html;
    index index.html index.htm index.php;
}
~~~

配置含义如下：

- 如果 URI 是 `/css/style.css`，则查找的完整路径为：`/var/share/nginx/html/css/style.css`
- 如果 URI 是 `/`，则第一个查找的完整路径为：`/var/share/nginx/html/index.html`



**配置在 location 内**

~~~nginx
location /wiki {
    root /var/share/nginx/html;
    index index.html index.htm index.php;
}
~~~

配置含义如下：

- 如果 URI 是 `/wiki/1.txt`，则查找的完整路径为：`/var/share/nginx/html/wiki/1.txt`
- 如果 URI 是  `/wiki/` ，第一个查找的资源完整路径为：`/var/share/nginx/html/wiki/index.html`



#### 总结

- **`root`**：会将请求的 URI 附加到指定的目录后面，形成最终的文件路径。
- `index` 的含义是，没有指定资源文件时，默认使用的资源文件，可以配置有多个，依次查找。



## 字段 alias

指令 `alias` 指定的路径是一个**精确的替换**。Nginx 会将 `location` 中匹配到的部分替换为 `alias` 指定的路径，URI 中剩下的部分直接附加到替换后的路径后。**注意**：`alias` 指令后面必须用 `/` 结尾；`alias` 只能在 `location` 块内使用。

~~~nginx
server {
    location /images/ {
        alias /data/storage/photos/;
    }
}
~~~

解释如下，请求的 URI 是：`/images/cat.jpg`，则最终服务器文件路径为：`/data/storage/photos/cat.jpg`

### 对比 `root` 和 `alias`

| 特性           | `root`                              | `alias`                             |
| :------------- | :---------------------------------- | :---------------------------------- |
| **工作方式**   | **拼接** URI 到指定路径后           | **替换**匹配部分为指定路径          |
| **路径结尾**   | 可带或不带 `/`                      | **必须**以 `/` 结尾                 |
| **常见用途**   | 定义静态资源根目录                  | 将URL映射到不同目录结构             |
| **Location块** | 可用于 `http`, `server`, `location` | 通常只在 `location` 中              |
| **最终路径**   | `root路径` + `请求URI`              | `alias路径` + `请求URI去除匹配部分` |

### 使用 alias 的场景

假设你的日志文件在 `/var/log/nginx/`，但你不想让用户通过 `/var/log/nginx/access.log` 这样的路径访问，而是通过 `/logs/access.log`。使用 `alias`。

~~~nginx
location /logs/ {
    alias /var/log/nginx/;
}
~~~

- **请求：** `https://example.com/logs/access.log`
- **最终路径：** `/var/log/nginx/access.log` ✅



## 字段 return

`return` 指令用于**立即停止处理当前请求**，并直接向客户端返回一个指定的 HTTP 状态码、可选的重定向 URL 或一段文本内容。`location` 块中 `return` 后面的指令不再执行。

~~~nginx
server {
    # 将整个旧域名重定向到新域名
    server_name old-site.com;
    return 301 https://new-site.com$request_uri;
    # $request_uri 变量会保留原始请求的路径和参数
}

location /old-page.html {
    # 将特定旧页面重定向到新页面
    return 301 https://$host/new-page.html;
}

location /go-to-google {
    return https://www.google.com;
}
# 等价于 return 302 https://www.google.com;

location /admin {
    # 直接返回403错误，不解释原因
    return 403;
}

location /secret.txt {
    # 返回403并附带一条文本消息
    return 403 "You are not allowed to access this file!";
}
~~~



## 字段 try_files

`try_files` 指令用于按顺序检查文件或路径是否存在，并使用找到的第一个文件或路径进行请求处理。如果所有指定的选项都未找到，它将执行最后一个参数作为 fallback（后备方案）。

#### 示例1：静态网站优化

访问域名时 linux.try.com/sdsdasd，返回的结果是 index.html。因为 `sdsdasd` 是当前请求路径，则 `$uri` 不存在，继续查找下一个 `$uri/` 也不存在。那就使用最后一个，返回首页。

~~~nginx
server {
    listen 80;
    server_name linux.try.com;
  
    location / {
        index index.html;
        try_files $uri $uri/ /index.html;
    }
}
~~~



#### 示例2：与命名 location 结合使用

最后一个参数经常是一个**命名 location**（以 `@` 开头），用于更复杂的后备逻辑。

~~~nginx
location / {
    try_files $uri $uri/ @backend;
}

# 一个命名的location，只能被内部重定向使用
location @backend {
    # 一些额外的代理设置
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    # 将请求代理到应用服务器
    proxy_pass http://backend_app;
}
~~~



## 字段 error_page

当服务器出现特定错误时，向客户端返回自定义的错误页面，而不是默认的、不友好的 Nginx 默认错误页。

```nginx
server {
    # 当发生 404 (Not Found) 错误时，内部重定向到 /404.html 这个 URI
    error_page 404 /404.html;
    # 定义一个 location 块来处理 /404.html 这个请求
    location = /404.html {
        # 注意：这个 location 块内部是空的，这意味着它将使用最外层的配置（如 root, index 等）
        # Nginx 会在 root 指令指定的目录下寻找 404.html 文件并返回
    }

    # 当发生 500 (Internal Server Error), 502 (Bad Gateway), 503 (Service Unavailable), 504 (Gateway Timeout) 错误时，
    # 内部重定向到 /50x.html 这个 URI
    error_page 500 502 503 504 /50x.html;
    # 定义一个 location 块来处理 /50x.html 这个请求
    location = /50x.html {
        # 同样，这个块也是空的，使用外层配置寻找 50x.html 文件
    }
}
```

#### 补充使用指令 internal

这是非常重要的优化。它标记这个 location 为“内部的”，意味着只能由 Nginx 的内部重定向（如 `error_page`、`index`、`try_files`）访问。

- **没有它**：用户直接访问 `https://yourdomain.com/404.html` 会成功返回页面，并且 HTTP 状态码是 `200`，这不符合逻辑。
- **有它**：用户直接访问 `https://yourdomain.com/404.html` 会得到真实的 `404` 错误，这才是正确的行为。

~~~nginx
server {
    error_page 404 /404.html;
    location = /404.html {
        internal;
    }
}
~~~









## 配置 http 七层反向代理

Nginx 有三条常用的代理命令，均属于反向代理。

- `proxy_pass`，代理http协议
- `fastcgi_pass`，代理fastcgi协议（php php-fpm 服务）
- `uwsgi_pass`，代理uwsgi协议（python uwsgi 服务）

代理和负载均衡是两回事，代理指的就是把请求转发给其他人，转发的目标可以是一台机器，也可以是多台机器。如果是多台机器，那需要使用 `upstream` 把多台机器放在一个组内。

~~~nginx
upstream webs {
    192.168.10.101:8000;
    192.168.10.102:8000;
    192.168.10.103:8000;
}

server {
    listen 80;
    
    # 代理一台机器
    location / {
        proxy_pass http://192.168.10.111;
    }
  
    # 代理一组机器
    location /api {
        proxy_pass http://webs;
    }
}
~~~

>代理的类型
>
>正向代理是客户端的代理，服务器不知道真正的客户端是谁
>反向代理是服务器的代理，客户端不知道真正的服务器是谁
>
>透明代理是客户端的代理，必须和客户端在一个网络内，且客户端无感知。一般用于用户的上网行为分析。





## http 透传 ip

~~~nginx
location / {
    proxy_pass http://backend_server;

    # 设置真实/原始客户端的 IP 地址
    proxy_set_header X-Real-IP $remote_addr;  
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for; 
    proxy_set_header Host $host;
}
~~~



## 代理优化参数

优化参数可以单独写入一个文件 proxy_params 中通过 `include` 导入，当然你也可以直接与 `proxy_pass` 并列放置。

~~~nginx
server {
    listen 80;
    location / {
        proxy_pass xxxx;
        include proxy_params;
    }
}
~~~

代理配置文件 `/etc/nginx/proxy_params `

~~~nginx
# 转发原始请求的 host 头部
proxy_set_header X-Real-IP $remote_addr; 
proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for; 
proxy_set_header Host $host;

# 代理到后端的TCP连接、响应、返回等超时时间
proxy_connect_timeout 10s; # nginx代理与后端服务器连接超时时间(代理连接超时)
proxy_read_timeout 10s; # nginx代理等待后端服务器的响应时间
proxy_send_timeout 10s; #后端服务器数据回传给nginx代理超时时间

# proxy_buffer代理缓冲区
# nignx会把后端返回的内容先放到缓冲区当中，
# 然后再返回给客户端，边收边传, 不是全部接收完再传给客户端
proxy_buffering on;
proxy_buffer_size 8k; # 设置nginx代理保存用户头信息的缓冲区大小
proxy_buffers 8 8k;   #proxy_buffers 缓冲区
~~~





## 动静分离

把动态请求和静态请求分开。静态资源使用 nginx 处理，动态接口交给 web 服务端处理。可以基于请求分离，也可以基于扩展名分离。

~~~nginx
upstream backends {
    server 192.168.43.100:8080;
    server 192.168.43.101:8080;
    server 192.168.43.102:8080;
}
server {
    listen 8888;
    server_name localhost;
  
    location /api {
        proxy_pass http://backends; 
    }
  
    location ~ \.(jpg|png|gif|css|js)$ {
        root /var/share/nginx/html/static;
    }
}
~~~

## 资源分离

使用 `if` 判断，请求不同资源

~~~nginx
upstream android {
    server 10.0.0.7;
}
upstream iphone {
    server 10.0.0.8;
}
upstream pc {
    server 10.0.0.9;
}
 
server {
    listen 80;
    server_name linux.sj.com;
 
    location / {
        if ($http_user_agent ~* "Android") {
            proxy_pass http://android;    
        }
        if ($http_user_agent ~* "iPhone") { 
            proxy_pass http://iphone;  
        }
        if ($http_user_agent ~* "WOW64") {
            return 403;     
        }
       
        # 默认代理到 PC
        proxy_pass http://pc;            
        include proxy_params;
    }
}
~~~





## 半连接队列和全连接队列

TCP 三次握手时，linux 内核会维护两个队列：半连接队列（SYN 队列）和全连接队列（ACCEPT 队列）。

服务端收到客户端的 SYN 请求后，内核会把这个连接放入半连接池（半连接队列），然后回复客户端 SYN-ACK，接着客户端返回 ACK，服务端收到后会把连接从半连接队列中移除，并放入全连接队列，等待应用程序从全连接对垒中取出使用。半连接队列和全连接队列都有最大长度限制，超出限制时，请求就会被丢弃并返回 RST。

Linux 内核中这两个最大长度都是可以设置的，默认都是 128。
- 设置全连接池大小：`sysctl -w net.core.somaxconn=65530`
- 设置半连接池大小：`sysctl -w net.ipv4.tcp_max_syn_backlog=10240`



## nginx 设置连接池大小

nginx 配置中 `backlog` 用来配置 SYN 队列的大小和 ACCEPT 队列的大小，它俩默认值是 511

~~~nginx
server {
    listen 80 backlog=10240;
} 
~~~

>注意：`backlog` 设置的值要配合内核中 `somaxconn` 和 `tcp_max_syn_backlog`。在提高内核值的基础上，再提高nginx 的配置，这样才能起到提高并发能力的效果。

查看队列是否溢出的命令：

- 查看全连接队列的溢出情况：`netstat -s | grep "overflowed"`
- 查看半连接队列的溢出情况：`netstat -s | grep "SYNs to LISTEN"`



## nginx 中 worker 的工作模式

Nignx 里面的 master 进程负责管理多个 worker 进程，worker 进程负责处理请求任务。worker 进程从全连接队列中取出请求然后处理请求。worker 使用全连接队列的方式分为两种情况（工作模式）。

#### 多个 worker 共享一个全连接队列

这是 nginx 默认的方式。只有一个 ACCEPT 队列，所有的 worker 都从其中取出连接然后处理请求任务。

- **优点**：因为共享一个全连接队列，耗时的任务只会阻塞一个 worker，其他空闲的 worker 可以继续取出连接并处理请求，不会拖慢全部连接的处理。
- **缺点**：会发生进程级别的争抢导致资源额外消耗。



#### 每个 worker 独享一个全连接队列

这种模式下每个 worker 都有一个自己专属的全连接队列，取连接时不会发生进程级别的争抢现象。所有 worker进程会都监听相同的接口。然后由内核负责将请求负载均衡到这些监听进程中去，相当于每个进程独享自己的全链接队列。

- **优点**：减少进程级别争抢，减少额外消耗。
- **缺点**：worker 处理耗时任务时，它队列里面的连接无法分享给其他空闲的 worker 进程。导致单个连接请求的延迟增大，CPU 分配不均匀。



## nginx 开启 reuseport 

nginx 使用 `reuseport` 启用复用端口，开启多个 worker 独享一个全连接队列模式。

~~~nginx
server {
    listen 80 reuseport;
}
~~~

开启后查看 listen 的 socket 端口情况

~~~bash
# worker 进程设置为4个, 4个worker进程 reuse了相同的端口
[root@rocky ~]# ss -lnt |grep 8888
LISTEN 0      511               0.0.0.0:8888       0.0.0.0:*   
LISTEN 0      511               0.0.0.0:8888       0.0.0.0:*   
LISTEN 0      511               0.0.0.0:8888       0.0.0.0:*   
LISTEN 0      511               0.0.0.0:8888       0.0.0.0:*   

# 因为是resuse重用的同一个端口，
# 所以在系统层面只能看到一个端口
[root@rocky ~]# netstat -tunlap | grep -w 8888
tcp        0      0 0.0.0.0:8888            0.0.0.0:*               LISTEN      31432/nginx: master 
~~~



#### 开启 reuseport 的效果

- CPU 负载下降
- CPU 使用率下降
- 上下文写换次数下降
- Nginx 服务平均延迟下降，单个请求的最高延迟可能会增加
- Nginx 慢请求数量下降



## 启动 worker 多线程模式

一个 worker 进程内只有一个线程，这意味着每个工作进程在同一时间只有一个在处理客户端请求。尽管每个工作进程是单线程的，但 nginx 通过事件驱动和非阻塞I/O的方式（epoll 模型）能够处理大量并发请求，并且单线程的模式避免了上下文切换的开销，实现高性能和高吞吐量。这种设计在处理静态内容和反向代理等场景下表现出色。

单线程的 worker 可能因为耗时任务阻塞全连接池，此时在 worker 进程内使用多线程模式，可以让 CPU 分配相对均匀、提高 CPU 利用率。

**配置 worker 进程开启多线程**

~~~nginx
# 定义一个名为‘my_pool’的线程池，3个线程，最大队列1024
thread_pool my_pool threads=3 max_queue=1024;

http {
    # 使用自定义线程池
    aio threads=my_pool; 
}
~~~

其中，
- 指令 `thread_pool` 定义和命名一个线程池。在 `main`  中使用。其中 `max_queue` 指定等待线程池处理的任务队列的最大长度。当所有线程都在忙时，新任务会进入队列等待。如果队列也满了，任务会被拒绝并记录错误。
- 指令 `aio` 指定启用线程池。可以在 `http`, `server`, `location` 中使用。
  - 使用自定义线程池  `aio threads=my_pool; ` 
  - 使用默认线程池 `aio threads; `  Nginx 会使用名为 `default` 的线程池。如果你没有定义名为 `default` 的线程池，Nginx 会自动创建一个（使用默认参数：`threads=1, max_queue=65536`）。

通过合理配置线程池，可以让 Nginx 在处理大文件下载等高磁盘 I/O 负载的场景时，依然保持极高的并发能力和整体性能。

>扩展：在 k8s 中 ingress-nginx 启用了端口复用，开启了 aio 线程池。



**启用线程池的前提**

**确保 Nginx 编译时包含了线程池支持**。通过以下命令来检查，如果输出 `with-threads`，则说明支持线程池。

```bash
nginx -V 2>&1 | grep -o with-threads
```

