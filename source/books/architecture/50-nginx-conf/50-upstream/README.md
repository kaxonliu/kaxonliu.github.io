# nginx 配置

nginx 是一个轻量级的套接字服务软件，可以充当很多角色，并且性能强悍。nginx 可以用在 web 层，代理动态请求、处理静态请求，实现动静分离。nginx 可以充当七层负载均衡实现反向代理功能。nginx 可以充当四层负载均衡实现反向代理功能。在生产环境中支持高并发，且内存消耗少，启动快。免费开源稳定好用，支持热部署。



## nginx 同类软件

~~~bash
1.apache：httpd，最早期使用的web服务，性能不高，操作难
2.nginx
    tengine：Tengine是由淘宝网发起的Web服务器项目。它在Nginx的基础上，针对大访问量网站的需求，添加了很多高级功能和特性
    openresty-nginx：OpenResty 是一个基于 Nginx 与 Lua 的高性能 Web 平台，其内部集成了大量精良的 Lua 库、第三方模块以及大多数的依赖项。用于方便地搭建能够处理超高并发、扩展性极高的动态 Web 应用、Web 服务和动态网关。
3.IIS：windows下的web服务
4.lighttpd：是一个德国人领导的开源 Web 服务器软件，其根本的目的是提供一个专门针对高性能网站，安全、快速、兼容性好并且灵活的 Web Server 环境。具有非常低的内存开销，CPU 占用率低，效能好，以及丰富的模块等特点。
5.GWS：google web server
6.BWS：baidu web server
~~~



## 安装 nginx 

两种安装方式，yum 和 源码编译安装。

### yum 安装

参考 nginx 官网提供的[安装指南](https://nginx.org/en/linux_packages.html)

### 源码安装

#### 1. 安装 nginx 依赖包

~~~bash
yum install gcc* glibc* -y
yum install -y pcre pcre-devel
yum install -y zlib zlib-devel
yum install -y openssl openssl-devel

### 说明
gcc：安装 nginx 需要先将官网下载的源码进行编译，编译依赖 gcc 环境
pcre pcre-devel：pcre 是一个 perl 库，包括 perl 兼容的正则表达式库，nginx 的 http 模块使用 pcre 来解析正则表达式
zlib zlib-devel：zlib 库提供了很多种压缩和解压缩方式，nginx 使用 zlib 对 http 包的内容进行 gzip
openssl openssl-devel：OpenSSL 是一个强大的安全套接字层密码库，囊括主要的密码算法、常用的密钥和证书封装管理功能及 SSL 协议，并提供丰富的应用程序供测试或其它目的使用。nginx 不仅支持 http 协议，还支持 https（即在ssl协议上传输http），所以需要在 Centos 安装 OpenSSL 库
~~~

#### 2. 官网下载源码包

~~~bash
# 下载
wget https://nginx.org/download/nginx-1.28.0.tar.gz


# 解压
tar -zxvf nginx-1.28.0.tar.gz
~~~

#### 3. 配置并安装

~~~bash
# 进入解压目录
cd nginx-1.28.0/
 
# 配置参数（具体参数可以./configure --help命令查看）
./configure

# 下面都是默认配置信息
nginx path prefix: "/usr/local/nginx"
nginx binary file: "/usr/local/nginx/sbin/nginx"
nginx modules path: "/usr/local/nginx/modules"
nginx configuration prefix: "/usr/local/nginx/conf"
nginx configuration file: "/usr/local/nginx/conf/nginx.conf"
nginx pid file: "/usr/local/nginx/logs/nginx.pid"
nginx error log file: "/usr/local/nginx/logs/error.log"
nginx http access log file: "/usr/local/nginx/logs/access.log"
nginx http client request body temporary files: "client_body_temp"
nginx http proxy temporary files: "proxy_temp"
nginx http fastcgi temporary files: "fastcgi_temp"
nginx http uwsgi temporary files: "uwsgi_temp"
nginx http scgi temporary files: "scgi_temp"
 

# 编译并安装
# 安装成功后的文件在 /usr/local/nginx目录下面
make && make install

 
# 验证
/usr/local/nginx/sbin/nginx -v
nginx version: nginx/1.28.0
~~~



#### 4. 配置环境变量

~~~bash
# /etc/profile 加入下面这一行
export PATH=$PATH:/usr/local/nginx/sbin

# 更新
source /etc/profile
~~~

#### 5. 使用 systemd 管理 nginx

~~~bash
# system管理配置
cat > /usr/lib/systemd/system/nginx.service << "EOF"
[Unit]
Description=nginx - high performance web server
Documentation=http://nginx.org/en/docs/
After=network-online.target remote-fs.target nss-lookup.target
Wants=network-online.target
 
[Service]
Type=forking
PIDFile=/usr/local/nginx/logs/nginx.pid
ExecStart=/usr/local/nginx/sbin/nginx -c /usr/local/nginx/conf/nginx.conf
ExecReload=/usr/local/nginx/sbin/nginx -s reload
ExecStop=/usr/local/nginx/sbin/nginx -s stop
 
[Install]
WantedBy=multi-user.target
EOF


# 加载新的unit （*.service）配置文件
systemctl daemon-reload


# 使用 systemctl 管理nginx
systemctl status nginx
systemctl start nginx
systemctl restart nginx
~~~





## nginx 常用命令

安装完 nginx 后可以直接使用 nginx 的命令。有以下常用的命令。

~~~bash
nginx #启动nginx。 等价于systemctl start nginx
 
nginx -s reopen #重启Nginx。   等价于systemctl restart nginx
 
nginx -s reload #重新加载Nginx配置文件，然后以优雅的方式重启Nginx。 等价于systemctl reload nginx
 
nginx -s stop #强制停止Nginx服务。 等价于systemctl stop nginx
 
nginx -s quit #优雅地停止Nginx服务（即处理完所有请求后再停止服务）
 
nginx -t #检测配置文件是否有语法错误，然后退出
 
nginx -?,-h #打开帮助信息
 
nginx -v #显示版本信息并退出
 
nginx -V #显示版本和配置选项信息，然后退出
 
nginx -V 2>&1 | sed "s/\s\+--/\n --/g" #模块分行输出，格式化输出
 
killall nginx #杀死所有nginx进程
 
systemctl enable nginx  #加入开机自启
    Centos6：
        启动：nginx
            service nginx start
            /etc/init.d/nginx start
        加入开机自启：
            chkconfig nginx on
 
nginx -T #检测配置文件是否有语法错误，转储并退出
 
nginx -q #在检测配置文件期间屏蔽非错误信息
 
nginx -p prefix #设置前缀路径(默认是:/usr/share/nginx/)
 
nginx -c filename #设置配置文件(默认是:/etc/nginx/nginx.conf)
 
nginx -g directives #设置配置文件外的全局指令
~~~





## nginx 平滑升级

**所谓平滑升级就是要依赖软连接**，安装好新版本nginx 之后，把 `/usr/local/nginx `和旧版本的nginx 安装目录删除软连接，然后把新版本的安装目录软连接到 `/usr/local/nginx `



## nginx 配置文件

### 1. 配置主体结构

~~~nginx
# 全局配置
 
# 事件配置
events { ... }
 
# http 模块配置
http{          
   upstream {}
   server {
        location  { ... }
    }
}
 
# stream 模块配置
stream{       
   upstream {}
   server {
        location  { ... }
    }
}
~~~

### 2. 全局配置

~~~nginx
# 1、运行用户
user nobody;
 
# 2、工作的worker进程数，通过与cpu核数保持一致，设置为auto，nginx会自动配置为核数。
#    注意：nginx启动后还有一个master进程负责与用户交互响应会话，具体干活的是worker进程
worker_processes  auto;  #
 
# 3、全局错误日志：日志级别有：debug、info、notice、warn、error、crit、alert
# 错误日志不能指定格式
#error_log  logs/error.log;
error_log  logs/error.log  notice;
error_log  logs/error.log  info;
 
# 4、PID文件
pid        logs/nginx.pid;
 
# 5、include用来引入配置文件
#    下述指令的含义：nginx启动时会加载动态dynamic模块的所有配置，确保动态模块都加载到nginx中
include /usr/share/nginx/modules/*.conf;
~~~



### 3. 配置 events

这个模块里面配置网络IO模型和连接数上限制。

~~~nginx
events {
    # 1、epoll仅用于linux2.6以上内核，还有其他io多路复用模型如select、poll等
    use  epoll; 
    #2、单个后台worker process进程的最大并发链接数，可以调大，但是文件描述也要一起调大，并且你要
    #  综合考虑硬件性能。
    # 当前 Nginx 服务器能够处理的并发请求数量 = worker_connections × worker_processes
    worker_connections  1024; # 注意：这个1024并不是开启1024个线程
}
~~~

>epoll：（最强）每个网络io都捆绑自己的事件（回调函数）一旦数据到达，就立即自己主动触发事件，告知自己的数据已经抵达。nginx只需要去队列去现成的链接来处理即可。
>
>select：遍历，链接越多效率越低。
>
>worker_connections：配置时需要结合硬件和 **文件描述符的数量**。



### 4. 配置 http

~~~nginx
http {
    #1、设定日志输出模板的名字和模板格式
    log_format  main  '$remote_addr - $remote_user [$time_local] "$request" '
                      '$status $body_bytes_sent "$http_referer" '
                      '"$http_user_agent" "$http_x_forwarded_for"';
  	
  	# 访问日志的位置和使用的模板名
    access_log  logs/access.log  main;
 
    #2、网络io优化参数
    #2.1 开启sendfile特性可以显著提高静态文件（如图片、CSS、JavaScript文件等）传输的效率
    # 原理如下
    # 开启sendfile后可以减少数据在操作系统内核和用户空间之间的拷贝次数。
    # 没有 sendfile 的情况下，传输文件通常需要经历如下步骤，数据需要在用户空间和内核空间之间进行两次拷贝。
    # （1）从硬盘读取文件到用户空间的缓冲区（内核→用户空间）
    # （2）将读取的数据从用户空间缓冲区写入到操作系统的网络缓冲区（用户空间→内核）
    # （3）操作系统网络缓冲区发送数据到网络（内核→网络）
    # 当开启了 sendfile 之后，这个操作简化了，并且可以避免将数据从内核空间拷贝到用户空间。sendfile 系统调用可以直接在内核中传输数据：
    # （1）内核会将数据从文件系统缓冲区直接发送到网络堆栈，无需先拷贝到用户空间，然后再从用户空间拷贝到内核的网络堆栈
    # （2）数据可以直接在内核地址空间中移动，避免了额外的上下文切换和数据复制开销
    # 所以，sendfile 能够减少 CPU 的使用，减少操作系统的上下文切换，并允许更快地传输文件数据，特别是对于大型文件的传输
    sendfile     on;
  
    #2.2 通常与sendfile一起用，当 tcp_nopush 设置为 on 时，Nginx 将尽可能发送较大的 TCP 数据包，减少 TCP 报文的数量，提高传输效率。这对于减少网络传输中的 TCP 慢启动阶段，减少网络延迟和提高网络吞吐量非常有用。这在传输大型文件或使用 HTTP/1.1 的 Keep-Alive 长连接时特别有效。
    tcp_nopush     on; 
  
    #2.3 开启Nagle算法，数据将会尽快发送出去，而不是等待缓冲区满或者接收到ACK。这会减少延迟，但可能会造成网络利用率低。这个选项在处理需要快速响应的短数据流（例如HTTP/1.1的keep-alive连接）时非常有用。
    tcp_nodelay     on;
  
    #2.4、控制长连接的两个参数：
    # 开启长连接：如果客户端在65秒内没有再次发送新的请求，那么Nginx将关闭这个连接，反之如果在65秒内有新的请求到来，那么这个连接会保持开启，等待处理新的请求
    keepalive_timeout  65; 
  
    # 默认情况下，Nginx的keepalive_requests 是设置为100，这个设置针对的是每个长连接在关闭前能处理的最大请求数量。你可以根据需要调整这个值。
    keepalive_requests 100; 
 
    #2.5开启gzip压缩，节省带块加速网络传输
    gzip  on;
 
    # 3、控制客户端请求头的缓冲区大小和数量：应对请求头过大的情况
    #  设定用于保存客户端请求头的缓冲区的大小，当客户端发送的请求头超过这个大小时，Nginx 将使用临时文件来保存请求头。这可以防止恶意客户端发送大量数据来消耗服务器资源
    client_header_buffer_size    128k;  
    # 如果请求头过大，可以使用多个缓冲区来保存。格式为 <数量> <大小>。<数量>代表缓冲区数量，<大小>代表每个缓冲区的大小。
    large_client_header_buffers  4 128k; 
 
  
    #4、mime.types定义了nginx可以识别的网络资源类型，例如css、js、jpg等
    include  /etc/nginx/mime.types; 
    
    # 5、http响应header中，如果没有明确指定Content-Type，则默认使用default_type指定的
    # application/octet-stream。这是一种二进制的数据类型，意味着这种内容不会被浏览器解析，
    # 而是作为一个下载文件来处理。这主要用于那些不适合以普通文本或者其他MIME类型表示的文件，
    # 例如可执行文件。
    default_type application/octet-stream; 
 
    # 5、设定虚拟主机配置
    server {
        # 侦听80端口
        listen    80;
    
        # [::]: 这代表 IPv6 地址中的一个缩写，它等同于所有的 IPv6 地址，类似于 
        listen [::]:80; IPv4 中的 0.0.0.0
      
        #定义使用 www.nginx.cn访问
        server_name  www.nginx.cn;
    
        #定义服务器的默认网站根目录位置
       # 强烈建议用绝对路径
        root /usr/share/nginx/html; 
    
        #设定本虚拟主机的访问日志，优先级高于外部定义的
        # 依然建议使用绝对路径
        access_log  /var/log/nginx/access.log  main;
    
        # 默认请求
        location / {
            # 优先级高于外部
            root /a/b/c; 
      
            # 定义首页索引文件的名称
            index index.php index.html index.htm;   
        }
    
        # 定义错误提示页面，出现以下状态码，uri定位到/50x.html,
        # 然后触发二次localtion匹配，对上location = /50x.html
        error_page   500 502 503 504 /50x.html;
        
        # =号的匹配更精确，所以优先级更高
        # {} 没有指定root目录，所以向外层查找，
        # 找到server层里定义的root，去该root指定的目录下找/50x.html
        location = /50x.html {
        }
 
        #静态文件，nginx自己处理
        location ~ ^/(images|javascript|js|css|flash|media|static)/ {
            # 过期30天，静态文件不怎么更新，过期可以设大一点，
            # 如果频繁更新，则可以设置得小一点。
            expires 30d;
            # 这里没有指定root，那去外层也就是server层定义的root指定的目录里找文件
        }
 
        # PHP 脚本请求全部转发到 FastCGI处理. 使用FastCGI默认配置.
        location ~ .php$ {
            fastcgi_pass 127.0.0.1:9000;
            fastcgi_index index.php;
            fastcgi_param  SCRIPT_FILENAME  $document_root$fastcgi_script_name;
            include fastcgi_params;
        }
 
        # 禁止访问 .htxxx 文件
        location ~ /.ht {
            deny all;
        }
    }
}
~~~



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



### 7. 日志格式

访问日志的格式可以自定义，错误日志的格式不能自定义。

~~~nginx
# 定义日志格式的 命令
log_format 日志名 日志格式

# 访问日志定义
access_log 日志文件路径 日志格式
~~~

日志格式可以使用的变量参数

~~~nginx
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

# 注:如果Nginx位于负载均衡器，nginx反向代理之后， web服务器无法直接获取到客 户端真实的IP地址。
# $remote_addr获取的是反向代理的IP地址。 反向代理服务器在转发请求的http头信息中，
# 增加X-Forwarded-For信息，用来记录客户端IP地址和客户端请求的服务器地址。
~~~



### 8. nginx 日志级别

~~~bash
Nginx的日志级别从低到高包括以下几种：
debug: 这是最低级别的日志，用于调试Nginx。这个级别的日志记录了大量的详细信息，对于调试很有价值，但是在生产环境中通常是禁用的，因为它会产生大量的日志数据。
info: 这个级别的日志记录了一般的运行时事件信息。
notice: 这个级别的日志记录了比较重要的正常运行时事件。
warn: 警告级别的日志表示可能出现的一些问题，虽然这些问题不会立即影响服务器的运行，但是可能需要引起注意。
error: 错误级别的日志记录了导致某个特定操作或请求失败的错误事件，如反向代理一个失败的请求或者连不上FastCGI服务器。
crit: 这是比错误更严重的一类事件，如连接池耗尽、服务器宕掉一个子进程等。
alert: 这个级别的日志记录了需要立即采取行动的问题，如系统磁盘被写满。
emerg: 这是最高级别的日志，这类日志事件表示系统不可用，如非预期的主进程崩溃或者服务器声明“Out of memory”。
在默认情况下，如果不指定日志级别，Nginx会记录error级别及以上的日志。如果指定了日志级别，Nginx会记录指定级别及以上的所有日志。例如，如果设置了warn级别，Nginx会记录warn、error、crit、alert、emerg这几个级别的日志。
~~~



### 9. 日志切割

使用 `logrotate` 工具管理，基于 crontab 每日切割一次。logrotate 工具不运行为一个系统服务，因此你不能使用 systemctl status logrotate 命令查看其状态，因为它不是一个持续运行的后台服务，而是由cron定时任务调度的一个程序。logrotate 通常是作为一个定时任务来运行的。它的运行通常由系统的 cron daemon 。默认情况下，logrotate 的定时任务通常在 `/etc/cron.daily/` 目录下，且文件名为 logrotate。

使用 yum 安装的 nginx 就会 自动生成下面这个文件 `/etc/logrotate.d/nginx `。

~~~bash
/var/log/nginx/*.log { # 指定要切割的日志
 
        daily   #每天切割日志
        missingok   #忽略日志丢失
        rotate 52   #日志保留时间 52天
        compress   #日志压缩
        delaycompress  #延时压缩
        not if empty    #不切割空日志
        create 640 nginx adm    #切割好的日志权限
        sharedscripts   #开始执行脚本
        postrotate  #标注脚本内容
                if [ -f /var/run/nginx.pid ]; then#判断nginx启动
                        kill -USR1 `cat /var/run/nginx.pid`   #access.log不存在时，重新生成一个，存在则继续用已存在的不会产生新的access.log
 
                fi
        endscript   #脚本执行完毕
}
~~~



#### 手动执行

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



#### 使用 crontab

新建一个脚本文件 `/etc/cron.daily/logrotate`，每天执行一次。

~~~bash 
/usr/sbin/logrotate /etc/logrotate.conf
EXITVALUE=$?
if [ $EXITVALUE -eq 0 ]; then
    /usr/bin/logger -t logrotate "ALERT exited abnormally with [$EXITVALUE]"
fi
exit 0
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
