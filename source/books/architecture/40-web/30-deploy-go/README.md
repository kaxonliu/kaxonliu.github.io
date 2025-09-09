# 部署 go

Go web 服务部署相对 php 或 python 会简单很多。因为 go 程序的 web 服务和 web 应用会被打包成一个可执行的二进制文件。对上游直接使用 http 协议对接。因此一般在 go web 服务加一个 nginx ，它们整体为 web 层。

>其实 go 应用部署最佳实践是基于 docker，后续在介绍容器化部署。



## 下载 go 环境

go 环境[下载地址](https://go.dev/dl/)

~~~bash
wget https://go.dev/dl/go1.25.1.linux-arm64.tar.gz
tar xf go1.22.1.linux-amd64.tar.gz
mv go /usr/local/go
 
vim /etc/profile
PATH=/usr/local/go/bin:$PATH
export PATH
~~~



## 构建二进制命令

~~~bash
# 0. 七牛 CDN 加速下载包
go env -w  GOPROXY=https://goproxy.cn,direct

# 切到程序工作目录下使用 go build 编译二进制文件
go build -o <输出二进制文件名>
~~~



## 配置 nginx 转发

~~~nginx
server {
    listen      80; 
    server_name chitchat.test www.chitchat.test;
 
    location /static {
        root        /go_pro/chitchat-master/public;
        expires     1d; # 静态资源缓存一天
        add_header  Cache-Control public;
        access_log  off;
        try_files $uri @goweb; # try_files指令会先查找路径/go_pro/chitchat-master/public/$uri是否存在，存在则返回，否则交给@goweb处理
    }
 
    location / {
        try_files /_not_exists_     @goweb;  # /_not_exists路径肯定不存在，使得所有请求都会转向名为@goweb的location块
    }
 
    location @goweb { # 定义一个名为@goweb的localtion块，用于处理动态请求
        proxy_set_header Host $http_host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Scheme $scheme;
        proxy_redirect off;
        proxy_pass http://127.0.0.1:8080;
    }
}
~~~



## 使用 supervisor 维护应用守护进程

Supervisor（Supervisor: A Process Control System）是用 Python 开发的一个client/server 服务，是Linux/Unix 系统下的一个进程管理工具，不支持 Windows 系统。它可以很方便的监听、启动、停止、重启一个或多个进程。用 Supervisor 管理的进程，当一个进程意外被杀死，supervisort 监听到进程死后，会自动将它重新拉起，很方便的做到进程自动恢复的功能，不再需要自己写 shell 脚本来控制。

#### 下载 supervisor

~~~bash
yum install supervisor -y
~~~

#### 配置服务 

在 `/etc/supervisord.d/` 目录下为 web 应用程序增加服务配置 `www.ini`，在其中配置进程启动目录和命令，是否自动重启等。

~~~ini
[program:www]
process_name=%(program_name)s
directory=/your/program/path
command=/your/program/run/command
autostart=true
autorestart=true
user=root
redirect_stderr=true
stdout_logfile=/log/path
~~~



#### 启动

~~~bash
systemctl start supervisord

supervisorctl reread	# 读取新配置
supervisorctl update  # # 启动配置有改动的程序
supervisorctl start www
~~~



