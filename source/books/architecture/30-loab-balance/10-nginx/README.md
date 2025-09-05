# nginx负载代理配置



## 七层负载均衡配置

配置包括在顶级的 http 内部。解析 http 协议，搭配 location 匹配 url 路径。

~~~bash
http {
    upstream testserver {   
      ip_hash; # 负载均衡算法，后续会详细介绍，不写默认rr轮询
      server 192.168.1.5:8080;
      server 192.168.1.6:8080;
    }
 
    server {
        listen 8080
        location / {
           proxy_pass  http://testserver;
        } 
    }
}
~~~



## 四层负载均衡配置

配置包括在顶级的 stream 内部。基于 IP + PORT 做转发，比如代理 mysql redis等服务就使用四层负载代理。

~~~bash
stream {
    upstream my_servers {
        least_conn;
        # 5s内出现3次错误，该服务器将被熔断5s
        server <IP_SERVER_1>:3306 max_fails=3 fail_timeout=5s;
        server <IP_SERVER_2>:3306 max_fails=3 fail_timeout=5s;
        server <IP_SERVER_3>:3306 max_fails=3 fail_timeout=5s;
    }
    server {
        listen 3306;
        proxy_connect_timeout 5s; 
        proxy_timeout 10s;  
        proxy_next_upstream on; 
        proxy_next_upstream_tries 3; 
        proxy_next_upstream_timeout 10s; 
        proxy_socket_keepalive on; 
 
        proxy_pass my_servers;
    }
}
~~~

### 参数解释

- max_fails=3 fail_timeout=5s;  故障重拾与熔断参数。从第一个失败请求开始计时，5秒内再出现2个失败请求，在标记这个web服务器不可用。在接下来的5秒内都不会再向这个web 服务器转发请求。

- proxy_connect_timeout 5s;  与被代理服务器建立连接的超时时间为5s
- proxy_timeout 10s;    获取被代理服务器的响应最大超时时间为10s
- proxy_next_upstream on;  当被代理的服务器返回错误或超时时，将未返回响应的客户端连接请求传递给upstream中的下一个 web 服务器。
- proxy_next_upstream_tries 3;   转发尝试请求最多3次。
- proxy_next_upstream_timeout 10s;    总尝试超时时间为10s。
- proxy_socket_keepalive on;  开启SO_KEEPALIVE选项进行心跳检测。



### stream 配置的前提

如果你配置 steam 模块无效，请检查一下你使用的版本是否支持 stream。查看nginx安装了哪些模块。

~~~bash
nginx -V

# 或者更方便的命令

[root@rocky ~]# 2>&1 nginx -V | tr ' '  '\n'|grep stream
--with-stream=dynamic
--with-stream_ssl_module
--with-stream_ssl_preread_module
~~~

stream模块是动态加载的，默认情况下未安装动态模块，所以 `/usr/lib64/nginx/modules` 和 `/usr/share/nginx/modules/` 都是空目录。

### 安装 stream模块

~~~bash
yum install -y nginx-mod-stream

# 安装后上述两个模块目录就有 stream 的模块和配置信息了
[root@rocky ~]# ls /usr/lib64/nginx/modules
ngx_stream_module.so
[root@rocky ~]# ls /usr/share/nginx/modules/
mod-stream.conf
[root@rocky ~]# cat /usr/share/nginx/modules/mod-stream.conf
load_module "/usr/lib64/nginx/modules/ngx_stream_module.so";
~~~

补充，如果需要安装 nginx 的所有模块，执行如下命令。

~~~bash
yum install -y nginx-all-modules
~~~

### 配置使用 stream 模块的四层配置

~~~bash
# 使用 include 导入 stream 模块
include /usr/share/nginx/modules/*.conf;

stream {
    upstream lb_servers {
        server 192.168.71.12:3306 max_fails=3 fail_timeout=5s;
        server 192.168.71.13:3306 max_fails=3 fail_timeout=5s;
    }
    server {
        listen 13306;
        proxy_pass lb_servers;
    }
}
~~~





## 演示：四层负载代理数据库从库

### 1. 准备环境

~~~bash
# 机器1 四层代理服务器
 - nginx 监听13306端口代理 mysql 服务
 - ip 192.168.10.15
 
# 机器2 mysql 服务器
 - ip 192.168.10.10
 - 端口 3306
~~~

### 2. 准备 mysql 服务

在机器2 上安装 mariadb ，授权用户远程登陆

~~~bash
# 关闭防火墙和selinux
systemctl stop firewalld
setenforce 0

# 安装 mariadb
yum install -y mariadb*

systemctl start mariadb
systemctl enable mariadb
systemctl status mariadb

# 登陆mysql 
mysql -u root -p

然后授权远程登陆
use mysql;
grant all on * to 'root'@'%' identified by '123';
flush privileges;
~~~

### 3. 准备四层代理

在机器1上安装 nginx 和stream 模块

~~~bash
yum install -y nginx
yum install -y nginx-mod-stream

systemctl start nginx
systemctl enable nginx
~~~

### 4. 配置四层代理

在 `/etc/nginx/nginx.conf`　中新增如下核心配置

~~~bash
include /usr/share/nginx/modules/*.conf;

stream {
    upstream db_servers {
        server 192.168.10.10:3306 max_fails=3 fail_timeout=5s;
    }
    server {
        listen 13306;
	      proxy_pass db_servers;
    }
}
~~~

### 5. 重启nginx

~~~bash
systemctl restart nginx
~~~

### 6. 在负载层访问mysql

在机器1 上安装 mysql 客户端然后使用命令连接本地端口 13306 即可登陆到机器2 上运行的 mysql 服务。

~~~
yum install -y mariabd

mysql -u root -P 13306 -p
~~~

### 7. 查看链接情况

在负载均层器上（机器1）。因为机器1即当负载均衡器又当客户层，所以有两个通信链接。

~~~bash
[root@rocky ~]# netstat -tunlap | grep -w 13306
tcp        0      0 0.0.0.0:13306           0.0.0.0:*               LISTEN      2062/nginx: master
tcp        0      0 127.0.0.1:13306         127.0.0.1:41878         ESTABLISHED 2064/nginx: worker
tcp        0      0 127.0.0.1:41878         127.0.0.1:13306         ESTABLISHED 2120/mysql
~~~

在 mysql 服务机器上（机器2）。可以看到机器1代理的链接。

~~~bash
[root@rocky2 ~]# netstat -tunlap | grep 3306
tcp        0      0 0.0.0.0:3306            0.0.0.0:*               LISTEN      6517/mariadbd
tcp        0      0 192.168.10.10:3306      192.168.10.15:35206     ESTABLISHED 6517/mariadbd
~~~







