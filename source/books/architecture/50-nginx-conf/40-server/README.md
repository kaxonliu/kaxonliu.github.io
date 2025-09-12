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

- 如果请求是 `/css/style.css`，那么资源查找的完整路径为：`/var/share/nginx/html/css/style.css`
- 如果没有指定具体的文件，请求以 `/` 结尾。那么会尝试按照 `index` 配置的文件顺序查找。
  - 第一个查找的资源完整路径为：`/var/share/nginx/html/index.html`

**配置在 location 内**

~~~nginx
location /wiki {
    root /var/share/nginx/html;
    index index.html index.htm index.php;
}
~~~

配置含义如下：

- 如果请求是 `/wiki/1.txt`，那么资源查找的完整路径为：`/var/share/nginx/html/wiki/1.txt`
- 如果请求是  `/wiki/` ，第一个查找的资源完整路径为：`/var/share/nginx/html/wiki/index.html`



#### 总结

`root` 的含义是把 URI 拼接到后面，`index` 的含义是指定首页文件列表。







### 5. 配置 http 七层反向代理

~~~nginx
http {
   upstream backend {
       server backend1.example.com;
       server backend2.example.com;
       keepalive 32;
   }
  server {
    location /api {
      proxy_pass http://backend
    }
  }
}
~~~



### 6. http 透传 ip

~~~nginx
server {
    listen 80;
    server_name example.com;
 
    location / {
        proxy_pass http://backend_server;
    
        # 设置真实/原始客户端的 IP 地址
        proxy_set_header X-Real-IP $remote_addr;  
        # 追加原始客户端的 IP 地址到 X-Forwarded-For 头信息中。
        # 在后端服务器中就可以通过读取 X-Forwarded-For 头信息来获取原始客户端的 IP 地址。
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for; 
        proxy_set_header Host $host;
    }
}
~~~





## 半连接队列和全连接队列

TCP 三次握手时，linux 内核会维护两个队列：半连接队列（SYN 队列）和全连接队列（ACCEPT 队列）。

服务端收到客户端的 SYN 请求后，内核会把这个连接放入半连接池（半连接队列），然后回复客户端 SYN-ACK，接着客户端返回 ACK，服务端收到后会把连接从半连接队列中移除，并放入全连接队列，等待应用程序从全连接对垒中取出使用。半连接队列和全连接队列都有最大长度限制，超出限制时，请求就会被丢弃并返回 RST。

Linux 内核中这两个最大长度都是可以设置的，默认都是 128。
- 设置全连接池大小：`sysctl -w net.core.somaxconn=65530`
- 设置半连接池大小：`sysctl -w net.ipv4.tcp_max_syn_backlog=10240`



### nginx 设置连接池大小

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



### nginx 中 worker 的工作模式

Nignx 里面的 master 进程负责管理多个 worker 进程，worker 进程负责处理请求任务。worker 进程从全连接队列中取出请求然后处理请求。worker 使用全连接队列的方式分为两种情况（工作模式）。

#### 多个 worker 共享一个全连接队列

这是 nginx 默认的方式。只有一个 ACCEPT 队列，所有的 worker 都从其中取出连接然后处理请求任务。

- **优点**：因为共享一个全连接队列，耗时的任务只会阻塞一个 worker，其他空闲的 worker 可以继续取出连接并处理请求，不会拖慢全部连接的处理。
- **缺点**：会发生进程级别的争抢导致资源额外消耗。



#### 每个 worker 独享一个全连接队列

这种模式下每个 worker 都有一个自己专属的全连接队列，取连接时不会发生进程级别的争抢现象。所有 worker进程会都监听相同的接口。然后由内核负责将请求负载均衡到这些监听进程中去，相当于每个进程独享自己的全链接队列。

- **优点**：减少进程级别争抢，减少额外消耗。
- **缺点**：worker 处理耗时任务时，它队列里面的连接无法分享给其他空闲的 worker 进程。导致单个连接请求的延迟增大，CPU 分配不均匀。



#### nginx 开启 reuseport 

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



#### 启动 worker 多线程模式

一个 worker 进程内只有一个线程，这意味着每个工作进程在同一时间只有一个在处理客户端请求。尽管每个工作进程是单线程的，但 nginx 通过事件驱动和非阻塞I/O的方式（epoll 模型）能够处理大量并发请求，并且单线程的模式避免了上下文切换的开销，实现高性能和高吞吐量。这种设计在处理静态内容和反向代理等场景下表现出色。

单线程的 worker 可能因为耗时任务阻塞全连接池，此时在 worker 进程内使用多线程模式，可以让 CPU 分配相对均匀、提高 CPU 利用率。

**配置 worker 进程内开启多线程**

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



#### 启用线程池的前提

**确保 Nginx 编译时包含了线程池支持**。通过以下命令来检查，如果输出 `with-threads`，则说明支持线程池。

```bash
nginx -V 2>&1 | grep -o with-threads
```





## 虚拟主机

一个 nginx 服务可以当多个 web 使用。比如配置多个服务都监听相同的端口号，根据域名来区分不同的服务。如果客户端继续使用 IP 访问，可以选择一个默认服务，配置 `listen 8080 default` 充当默认服务响应客户端。还可以给 server_name 配置别名 `server_name somename alias another.alias;`。

~~~nginx
server {
    listen 8080;
    server_name www.xxx.com;
    root /usr/share/nginx/html/www;
}

server {
    listen 8080 default;
    server_name bbs.xxx.com;
    root /usr/share/nginx/html/bbs;
}
~~~



## location 匹配规则

Nginx 配置文件中的 `server` 模块下的子模块`location` 包含了一套与请求 URI 进行匹配的规则，在 `server` 中可以配置多个 `location` ，`location` 只会和 URI 匹配，`?` 后面的查询参数不参加匹配。

>参考：
>
>- 官方：https://nginx.org/en/docs/http/ngx_http_core_module.html#location
>- 中文翻译：https://tengine.taobao.org/nginx_docs/cn/docs/http/ngx_http_core_module.html#location

location 有两类匹配规则：前缀匹配、正则匹配

### 前缀匹配

#### 1. 使用 `=`

~~~bash
localtion = /static/img/logo.jpg

# 使用 = 号，后面都是普通字符且必须以左斜杠开头
# 只有当请求的 URI 完全一字不落地等于 /static/img/logo.jpg  时，这个location才会被匹配。
~~~

#### 2. 使用 `^~` 

~~~bash
location ^~ /static/img

# 使用 ^~ 装饰符，后面都是普通字符且必须以左斜杠开头
# 当请求的 URI 以 /static/img 开头时，这个location以非正则表达式的方式被匹配。
~~~

#### 3. 不加任何修饰符的前缀匹配

~~~bash
location /static/img/logo
# 当请求的URI以 /static/img/logo 开头时，这个location会被匹配。
# 一样的，左斜杠开头是必须的。
~~~



### 正则匹配

正则匹配就简单了，分两种

~~~bash
location  ~   路径部分   # 大小写敏感
location  ~*  路径部分   # 大小写不敏感

# 路径部分可以加入正则表达式
# 对于请求的uri路径，会按照正则的规则匹配，并非一定要从左斜杠作为开头/前缀
~~~



### 匹配流程

总体原则：先使用相对明确的前缀匹配，再使用相对模糊的正则匹配。

**第一阶段**：
- 先使用 `=` 的完全匹配。如果匹配成功，则停止不再继续匹配。

**第二阶段**：
- 进行前缀匹配中的常规字符串匹配（包含带修饰符的 `^~` 与不带修饰符的两大类）。这两类没有优先级之分，谁匹配规则写的越精确（匹配规则写的越长）就使用谁。它俩的区别在于：如果被使用的 location 里带有 `^~` 修饰符，那本次搜索就此结束，不会再有后续；如果这个 location 里不带有任何修饰符，那本次搜索不会结束，还会进行第三阶段的正则匹配。

**第三阶段**：
- 正则表达式匹配按照在配置文件中自上而下的先后顺序依次检查正则匹配的规则，但凡匹配成功一个，则搜索停止。
- 如果正则匹配失败，要分两种情况看。第一种：前面第二阶段匹配失败进入的第三阶段，那最终就是匹配失败。第二种：如果在第二阶段时，匹配成功并记录下了一个路径部分最长最精准的 location，只不过该location 属于没有带任何修饰符的类型，才进入了第三阶段的正则匹配的。现在所有正则匹配都失效了，那也只能用该 location了，因为也只有它是最精准的了。



## server之return 指令

`return` 指令的作用是停止处理请求，直接返回响应内容。执行 return 指令后，location 中的后续指令不会被执行。

~~~nginx
# 用法
return code [text];
return code URL;		# 常用语重定向
return URL          # 需要 http 或 https 开头的 URL


# 示例
location ^~ /return00 {
    return 403;
}

location ^~ /return01 {
    default_type text/html;  # 必须告诉浏览器return内容的类型
    return 200 'request success';
}

location ^~ /return02 {
    return 302 /hello;
}

location ^~ /return03 {
    return http://kaxonliu.top;   # 默认响应状态码是 302
}

location ^~ /return05 {
    default_type application/octet-stream;   # 下载文件
    return 200 'egon say hello';
}
~~~



>扩展：**301 和 302 重定向**
>
>301 永久重定向。第一个请求由服务端处理，后面就不再请求服务端（浏览器缓存记录的结果）。
>
>302 临时重定向。每次请求都打向服务端，走一遍重定向流程。



## root 指令和 alias 指令

相同点：都是用来指定资源的查找路径

不同点：

- root 指令是为当前location 定义了访问的根目录。完整的资源路径是: root + 请求 uri

比如用户访问：http://192.168.10.112:8080/books/cat/xxx/yyy

root配置如下，则完整资源路径为：/var/www/html/books/cat/xxx/yyy

~~~nginx
location /books/cat {
    root /var/www/html;
}
~~~

- alias 指令也规定了一个路径，但是不是根目录。alias 会把自己的路径替换 location 规则匹配的路径，得到一个完整的资源路径。

比如用户访问：http://192.168.10.112:8080/books/cat/xxx/yyy

alias 配置如下，则完整资源路径为：/a/b/c/xxx/yyy

~~~nginx
location /books/cat {
    alias /a/b/c;
}
~~~



## index 指令

没有指定资源文件时，默认使用的资源文件，可以配置有多个，依次查找

~~~nginx
location / {
    index index.html 1.txt 2.txt;
}
~~~



## 代理的类型

正向代理

- 代理客户端。用户需要设置

反向代理

- 代理服务端

透明代理

- 代理客户端的，用户无感知（即不需要设置），需要和客户端在一个局域网。用做用户上网行为分析。



## upstream 负载均衡配置健康检查

upstream 中的配置选项

~~~bash
down 节点下线
backup 备用节点，平是不工作，其他节点全部挂掉后才开始工作
max_fails 允许请求失败的次数
fail_timeout 经过 max_fails失败后，该节点被暂停代理的时间
max_conn 限制最大的接收连接受


#上游服务列表
upstream test_balance {
    server 192.168.43.100:8080 max_fails=3 fail_timeout=5s;
    server 192.168.43.101:8080 max_fails=3 fail_timeout=5s;
    server 192.168.43.102:8080 backup;
    server 192.168.43.103:8080 down;
}

server {
    listen 8888;
    server_name localhost;
    location ^~ /test/ {
        proxy_pass http://test_balance;
    }
}
~~~



### 慢启动方案

付费版才能使用慢启动参数 。开源版本可以降低轮训权重的方式减少启动时的流量压力。

~~~nginx
upstream backend {
    server backend1.example.com slow_start=30s;
    server backend2.example.com;
    server backend3.example.com;
}
~~~



### 健康检查

参数 max_fails=3 fail_timeout=5s; 只能检查服务节点是否挂掉。但是后端代码内部报错返回 5xx 状态码。此时需要额外健康检查处理。使用 `proxy_next_upstream` 遇到指定报错信息，第二次再出现这个报错就会把请求交给下一个节点。

~~~nginx
upstream blog {
    server 172.16.1.7;
    server 172.16.1.8;
}
 
server {
    listen 80;
    server_name linux.wp.com;
 
    location / {
        proxy_pass http://blog;
        include proxy_params; 
        #可以配置在这里，当然也可以写到include指定的文件里
        proxy_next_upstream error timeout http_500 http_502 http_503 http_504 http_403 http_404;
    }
}
~~~



## 负载均衡算法

轮巡、加权轮训、ip_hash、url_hash、最少连接调度least_conn、公平调度fair
