# nginx 的基本使用

nginx 是一个轻量级的套接字服务软件，可以充当很多角色，并且性能强悍。nginx 可以用在 web 层，代理动态请求、处理静态请求，实现动静分离。nginx 可以充当七层负载均衡实现反向代理功能。nginx 可以充当四层负载均衡实现反向代理功能。在生产环境汇总支持高并发，且内存消耗少，启动快。免费开源稳定好用支持热部署。



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

#### 1. 安装nginx 依赖包

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
   server{
        location  { ... }
    }
}
 
# stream 模块配置
stream{       
   upstream {}
   server{
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

