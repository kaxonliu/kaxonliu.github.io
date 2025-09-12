# nginx 配置

nginx 是一个轻量级的套接字服务软件，可以充当很多角色，并且性能强悍。nginx 可以用在 web 层，代理动态请求、处理静态请求，实现动静分离。nginx 可以充当七层负载均衡实现反向代理功能。nginx 可以充当四层负载均衡实现反向代理功能。在生产环境中支持高并发，且内存消耗少，启动快。免费开源稳定好用，支持热部署。



## nginx 同类软件

- apache：httpd 最早期使用的web服务，性能不高，操作难。
- tengine：Tengine 是由淘宝网发起的Web服务器项目。它在Nginx的基础上，针对大访问量网站的需求，添加了很多高级功能和特性。
- openresty-nginx：OpenResty 是一个基于 Nginx 与 Lua 的高性能 Web 平台，其内部集成了大量精良的 Lua 库、第三方模块以及大多数的依赖项。用于方便地搭建能够处理超高并发、扩展性极高的动态 Web 应用、Web 服务和动态网关。
- IIS：windows下的web服务
- lighttpd：是一个德国人领导的开源 Web 服务器软件，其根本的目的是提供一个专门针对高性能网站，安全、快速、兼容性好并且灵活的 Web Server 环境。具有非常低的内存开销，CPU 占用率低，效能好，以及丰富的模块等特点。
- GWS：google web server
- BWS：baidu web server



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
~~~

>**解释说明：**
>
>- gcc：安装 nginx 需要先将官网下载的源码进行编译，编译依赖 gcc 环境。
>- pcre pcre-devel：pcre 是一个 perl 库，包括 perl 兼容的正则表达式库，nginx 的 http 模块使用 pcre 来解析正则表达式。
>- zlib zlib-devel：zlib 库提供了很多种压缩和解压缩方式，nginx 使用 zlib 对 http 包的内容进行 gzip
>- openssl openssl-devel：OpenSSL 是一个强大的安全套接字层密码库，囊括主要的密码算法、常用的密钥和证书封装管理功能及 SSL 协议，并提供丰富的应用程序供测试或其它目的使用。nginx 不仅支持 http 协议，还支持 https（即在ssl协议上传输http），所以需要在 Centos 安装 OpenSSL 库。



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

**所谓平滑升级就是要依赖软连接**，安装好新版本 nginx 之后，把 `/usr/local/nginx `和旧版本的 nginx 安装目录删除软连接，然后把新版本的安装目录软连接到 `/usr/local/nginx `



## nginx 配置文件

nginx 的配置很简单，结构也非常模块化。在主配置文件中通过 `include /etc/nginx/conf.d/*.conf;`  把子配置文件导入。

- 主配置文件是 `/etc/nginx/nginx.conf`
- 子配置文件是 `/etc/nginx/conf.d/*.conf`



#### 配置主体结构

~~~nginx
# 全局配置
 
# 事件配置
events { ... }
 
# http 模块配置
http {          
   upstream {}
   server {
        location  { ... }
    }
}
 
# stream 模块配置
stream {       
   upstream {}
   server {
        location  { ... }
    }
}
~~~

