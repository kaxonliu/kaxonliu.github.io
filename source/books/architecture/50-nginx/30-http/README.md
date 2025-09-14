# 配置 http

`http` 块是 Nginx 配置中至关重要且最常用的部分，它包含了所有处理 HTTP 和 HTTPS 流量的指令。它通常位于 `main`（全局）上下文中，并且一个配置文件中只能有一个 `http` 块。

~~~nginx
http {
    # 文件格式
    include /etc/nginx/mime.types;
    default_type application/octet-stream;
    
    # 日志
    log_format main '$remote_addr - $remote_user [$time_local] "$request" '
                    '$status $body_bytes_sent "$http_referer" '
                    '"$http_user_agent" "$http_x_forwarded_for"';

    access_log /var/log/nginx/access.log main;
		
    # 高并发
    sendfile on;
    tcp_nopush on;
    tcp_nodelay on;
    keepalive_timeout 65;
    keepalive_requests 100;
		
    # 压缩
    gzip on;
    gzip_min_length 1k;
    gzip_types text/plain text/css application/json application/javascript text/xml application/xml application/xml+rss text/javascript;
		
    # 导入配置文件
    include /etc/nginx/conf.d/*.conf;
    
    # 上游服务
    upstream myapp { 
        server 10.0.0.1:8080;
    }
    
    # 定义虚拟主机
    server { 
        listen 80; 
        server_name example.com; 
    }
}
~~~



## http 块的核心作用

1. **作为容器**：包含所有 `server`、`upstream`、`location` 等配置块的顶层容器。
2. **设置默认值**：在 `http` 块中定义的指令为其内部所有的 `server` 块提供了默认值。你可以在 `server` 或 `location` 块中覆盖这些默认值。
3. **高效管理**：通过 `include` 指令，可以将配置拆分到多个文件中，使管理大型配置变得更加清晰和模块化（例如，每个网站在单独的文件中）。





## 配置访问日志

访问日志的设置有两个字段，分别是：`log_format` 和 `access_log`。前者用来定义一种日志格式，后者用来设置访问日志的路径和格式。在 `http` 块内设置访问日志的路径和格式，为该块内所有 `server` 提供默认设置。

**定义日志格式的变量**

~~~bash
$remote_addr        # 记录客户端IP地址
$remote_user        # 记录客户端用户名
$time_local         # 记录通用的本地时间
$time_iso8601       # 记录ISO8601标准格式下的本地时间
$request            # 记录请求的方法以及请求的http协议
$status             # 记录请求状态码(用于定位错误信息)
$body_bytes_sent    # 发送给客户端的资源字节数，不包括响应头的大小
$bytes_sent         # 发送给客户端的总字节数
$msec               # 日志写入时间。单位为秒，精度是毫秒。
$http_referer       # 记录从哪个页面链接访问过来的
$http_user_agent    # 记录客户端浏览器相关信息
$http_x_forwarded_for #记录经过的所有服务器的IP地址
$X-Real-IP         #记录起始的客户端IP地址和上一层客户端的IP地址
$request_length     # 请求的长度（包括请求行， 请求头和请求正文）。
$request_time       # 请求花费的时间，单位为秒，精度毫秒
~~~



## logrotate 日志切割 

nginx 日志切割是一个至关重要的运维任务，可以防止日志文件无限增长，占满磁盘空间，同时便于日志的归档和管理。定期切割日志，可以使用 linux 自带的 `logrotate` 工具。

使用 yum 安装的 nginx 就会自动生成文件 `/etc/logrotate.d/nginx ` 

~~~bash
/var/log/nginx/*.log {  # 监控 /var/log/nginx/ 目录下的所有 .log 文件
        daily           # 每天切割一次
        missingok       # 如果日志文件丢失，不报错，继续处理下一个
        rotate 52       # 保留 52 个备份（即 52 天）
        compress        # 使用 gzip 压缩旧的日志文件
        dateext         # 使用日期作为后缀 access.log-20240912
        delaycompress   # 延迟压缩，下一次切割时再压缩上一次的日志文件
        not if empty    # 不切割空日志
        create 640 nginx adm    # 切割好的日志权限
        sharedscripts           # 开始执行脚本
        postrotate              # 标注脚本内容
            # 向 Nginx 主进程发送 USR1 信号，通知它重新打开日志文件
            # access.log 不存在时，重新生成一个，存在则继续用已存在的不会产生新的 access.log
            if [ -f /var/run/nginx.pid ]; then
                kill -USR1 `cat /var/run/nginx.pid`
            fi
        endscript
}
~~~

**定时触发**：`logrotate` 通常由 `cron` 每天定时运行（例如在 `/etc/cron.daily/logrotate`）。

~~~bash
# /etc/cron.daily/logrotate
/usr/sbin/logrotate /etc/logrotate.conf
EXITVALUE=$?
if [ $EXITVALUE -eq 0 ]; then
    /usr/bin/logger -t logrotate "ALERT exited abnormally with [$EXITVALUE]"
fi
exit 0
~~~

**手动触发**

~~~bash
# 调试运行（干跑），只显示过程，不实际进行任何操作
sudo logrotate -d /etc/logrotate.conf
sudo logrotate -d /etc/logrotate.d/nginx

# 强制运行，即使时间未到也执行一次轮转
sudo logrotate -f /etc/logrotate.conf
sudo logrotate -f /etc/logrotate.d/nginx

# 指定状态文件，使用-v显示详细信息
sudo logrotate -vf /etc/logrotate.d/nginx
~~~



## 字段 sendfile

启用或禁用 `sendfile` 系统调用。**建议：在提供静态内容服务时，务必保持开启**。

~~~nginx
http {
    sendfile on;	  # 启用
    sendfile off;	  # 禁用
}
~~~

当传输一个文件（比如静态文件如 CSS、JS、图片）到客户端时，默认的文件传输过程涉及 **2 次上下文切换** 和 **4 次数据拷贝**，效率较低。

~~~bash
# 开启前的传输过程
1. 从磁盘读取文件内容到内核空间的缓冲区。
2. 应用程序（Nginx）将数据从内核缓冲区读入用户空间的缓冲区。
3. 应用程序再将数据从用户缓冲区写入内核空间的 socket 缓冲区。
4. 最后，数据从 socket 缓冲区发送到网卡。
~~~

启动 `sendfile` 后传输流程简化，避免了数据在**用户空间**和**内核空间**之间的来回拷贝，减少了上下文切换和数据拷贝次数，**极大地提升了静态文件的传输效率**，降低了 CPU 占用。

~~~bash
# 开启后的传输过程
1. 内核直接从磁盘读取文件内容到内核缓冲区。
2. 内核直接将数据从内核缓冲区拷贝到 socket 缓冲区。
3. 数据从 socket 缓冲区发送到网卡。
~~~



## 字段 tcp_nopush

与 `sendfile on;` 指令配合使用。只在将 `sendfile` 设置为 `on` 时才有效。

启用时，nginx 会尝试将多个数据包在内核缓冲区中整合成一个更大的数据包，然后一次性发送出去。这样做的好处是**减少了网络上的报文数量**，提高了网络效率，有助于减少网络拥塞。它优化了数据包数量。



## 字段 tcp_nodelay

启用或禁用 Nagle's 算法。Nagle's 算法是为了减少小网络包的数量而设计的。这虽然节约了带宽，但**增加了数据的延迟**（等待ACK的时间）。设置 `tcp_nodelay on;` 意味着禁用该算法，允许小数据包立即被发送，**降低了延迟**，提高了响应速度。它优化了发送数据的“速度”。



## tcp_nopush 和 tcp_nodelay 协同工作

看起来 `tcp_nopush on;`（攒包）和 `tcp_nodelay on;`（立即发）是矛盾的，但实际上 Nginx 可以巧妙地让它们协同工作：

1. 当同时启用 `sendfile on;`, `tcp_nopush on;`, `tcp_nodelay on;` 时：
2. Nginx 会先利用 `tcp_nopush` 的特性，在发送完整的大文件时，尽可能攒包以提高网络利用率。
3. 当需要发送最后一个小数据包时，Nginx 会禁用 `tcp_nopush`，并利用 `tcp_nodelay` 的设置立即发送这个小包，以避免延迟。

这种组合实现了**效率和延迟的最佳平衡**：既充分利用了带宽来传输数据，又能确保最终的响应及时发送。



## 字段 keepalive_timeout

设置保持连接的超时时间，它定义了客户端与服务器之间的一个 TCP 连接在**空闲状态**下可以保持打开的最长时间（单位：秒）。保持连接的好处：**允许客户端在同一个 TCP 连接上发送多个请求，避免了为每个请求都进行三次握手建立新连接的开销**，减少了延迟和服务器资源消耗。

~~~nginx
# 设置值为65，表示如果客户端
# 在 65 秒内没有发起新的请求，Nginx 会主动关闭这个连接。
http {
    keepalive_timeout 65;
}
~~~

## 字段 keepalive_requests

设置一个保持连接上最多可以服务的请求数量。单个客户端通过一个 TCP 连接可以发送的最大请求数。达到这个数量后，Nginx 会主动关闭该连接。这有助于**平衡负载**，防止某个客户端长期占用一个连接。同时，定期重建连接也有助于释放分配给该连接的内存资源。默认值是 `100`，对于现代浏览器来说，这个值通常足够。

>补充，如下配置的含义：超过 65s，或者请求数量超过 100个，nginx 都会主动关闭这个连接。
>
>~~~nginx
>http {
>    keepalive_timeout 65;
>    keepalive_requests 100;
>}
>~~~

上面这 5个指令的组合是 Nginx 能够实现高性能、高并发处理能力的重要组成部分。



## 配置请求头缓存区

Nginx 中 `client_header_buffer_size` 和 `large_client_header_buffers` 两个指令是用来控制 Nginx **处理来自客户端（通常是浏览器）的 HTTP 请求头时使用的缓冲区大小**。如果请求头的大小超过了配置的缓冲区限制，Nginx 会返回一个 414 (Request-URI Too Large) 或 400 (Bad Request)错误。

#### client_header_buffer_size

设置了**用于读取客户端请求头**的默认缓冲区大小。这个值应该设置为比你的典型请求头稍大一点，以避免为每个请求都分配过大的内存。

**工作流程：**

1. 当 Nginx 开始处理一个请求时，它会首先分配一个大小为 `client_header_buffer_size` 的缓冲区来读取请求头。
2. 在绝大多数情况下，正常的请求头（通常小于 1KB）都可以在这个缓冲区中被完整地读取和解析。
3. 如果 Nginx 在解析过程中发现请求头**超出了**这个初始缓冲区的大小，它会自动切换到使用由 `large_client_header_buffers` 指令配置的更大、数量更多的缓冲区。

#### large_client_header_buffers

设置用于读取超大请求头或超长 URI 的缓冲区的数量和每个的大小。

~~~nginx
http {
    # 设置默认请求头缓冲区为 1KB（对于绝大多数请求足够了）
    client_header_buffer_size 1k;

    # 设置大请求头缓冲区：最多 4 个缓冲区，每个 8KB
    large_client_header_buffers 4 8k;
}
~~~

#### 设置依据

- 将这些值设置得**过大**会浪费内存，并可能让服务器更容易受到**缓冲区溢出攻击**（DoS攻击的一种）。
- 设置得**过小**则会拒绝合法的、但头信息较大的请求。
- 原则是：在满足业务需求的前提下，使用尽可能小的值。





## 开启压缩 gzip

在 Nginx 中开启响应压缩（Gzip）是一个非常有效的性能优化手段，它可以显著减少传输数据的大小，从而提高网站加载速度并节省带宽。压缩功能主要由 `ngx_http_gzip_module` 模块提供，它通常是默认内置的

如下是一个使用于大多数啊网站的推荐配置。可以将其放在 Nginx 配置文件的 `http`、`server` 或 `location` 块中。通常放在 `http` 块中使其全局生效。

~~~nginx
http {
  
    # 开启 Gzip 压缩, 减少传输数据量
    gzip on;

    # 设置压缩级别，1-9，6 是较好的权衡（越高CPU消耗越大）
    gzip_comp_level 6;

    # 设置最小压缩文件大小，小于该值的文件不压缩
    gzip_min_length 1k;

    # 配置压缩缓冲区，默认是申请4个16k的内存空间
    gzip_buffers 4 16k;

    # 设置启用压缩的最低 HTTP 版本（通常不需要改）
    gzip_http_version 1.1;

    # 设置需要压缩的 MIME 类型
    # 文本、js、css、xml、json 都是非常常见的可压缩类型
    # 图片类如 jpg, png, gif 本身已经是压缩格式，再压缩效果不好且浪费CPU
    gzip_types text/plain text/css text/xml application/json application/javascript application/xml+rss application/atom+xml image/svg+xml;

    # 在响应头中添加 Vary: Accept-Encoding，告知缓存服务器缓存压缩和未压缩两种版本
    gzip_vary on;

    # 针对古老的、不支持压缩的 IE 浏览器禁用压缩
    gzip_disable "MSIE [1-6]\.";

}
~~~



## MIME 文件类型组合配置

设置 HTTP 响应头 `Content-Type` 的值。如果请求资源的后缀在指定文件中，则 nginx 会自动设置；如果不在，则默认使用 `default_type` 指定的值。

~~~nginx
http {
    # 引入 MIME 类型文件，这样 Nginx 就知道如何根据文件扩展名设置 Content-Type 头
    include       /etc/nginx/mime.types;

    # 如果找不到对应的 MIME 类型，使用此默认类型（通常是二进制流），浏览器会当成文件下载
    default_type  application/octet-stream;
}
~~~





## 配置 http 七层反向代理

Nginx 七层反向代理，负载均衡。

~~~nginx
http {
   upstream backend {
       server 192.168.10.100:8080;
    	 server 192.168.10.101:8080;
       server 192.168.10.102:8080;
       keepalive 32;
  }
  server {
      location /api {
          proxy_pass http://backend
      }
  }
}
~~~


