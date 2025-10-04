# 安装 Redis

Redis 安装一般有两种方式，第一种是 yum 安装，第二种是源码编译安装。

yum 安装的版本相对较老，安装简单。

源码编译安装可以使用最新的稳定的版本，建议到[官网下载](https://redis.io/downloads/)源码包，然后编译安装。



## yum 安装

~~~bash
yum install -y redis
~~~

安装后使用 systemctl 管理

~~~bash
systemctl start redis
systemctl enable redis
systemctl status redis
~~~

查看 redis 服务端和客户端的版本

~~~bash
[root@localhost yum.repos.d]# redis-server -v
Redis server v=3.2.12 sha=00000000:0 malloc=jemalloc-3.6.0 bits=64 build=7897e7d0e13773f

[root@localhost yum.repos.d]# redis-cli -v
redis-cli 3.2.12
~~~



## 源码编译安装

#### 安装依赖

~~~bash
yum install -y gcc gcc-c++ make cmake wget ctl
~~~

#### 下载源码包

~~~bash
wget https://download.redis.io/redis-stable.tar.gz
~~~

#### 解压

~~~bash
tar xf redis-stable.tar.gz
~~~

#### 进入源码包

~~~bash
cd redis-stable
~~~

#### 编译

编译时可以使用并发的方式加快编译过程。

~~~bash
make

# 并发2个任务编译(数量控制在CPU个数2倍以内)
make -j2
~~~

#### 安装

安装时指定安装目录

~~~bash
make PREFIX=/usr/local/redis install
~~~

#### 添加环境变量

按照如下内容添加环境变量。

~~~bash
# /etc/profile.d/redis.sh
export PATH=/usr/local/redis/bin:$PATH
~~~

然后加载到当前终端 `source /etc/profile.d/redis.sh` 

#### 修改配置文件

从源码包中拷贝配置模板，然后载模板的基础上修改配置

~~~ini
# 1、拷贝模版配置
mkdir -p /usr/local/redis/conf/
cp /root/redis-stable/redis.conf /usr/local/redis/conf/
 
# 2、修改配置文件 redis.conf 
# 主要修改如下三项
# 修改:以守护进程的方式运行
daemonize yes
# 修改：0.0.0.0意味着接受任意IP的连接请求。
bind 0.0.0.0
# 增加：设置redis的登录密码
requirepass 123
~~~

#### 使用 systemd 管理

新建文件 `/usr/lib/systemd/system/redis.service` 然后复制如下内容。

~~~bash
[Unit]
Description=Redis
After=network.target
 
[Service]
Type=forking
ExecStart=/usr/local/redis/bin/redis-server /usr/local/redis/conf/redis.conf
ExecReload=/bin/kill -s HUP $MAINPID
ExecStop=/bin/kill -s QUIT $MAINPID
PrivateTmp=true
 
[Install]
WantedBy=multi-user.target
~~~

**重新加载**（不要忘记了）

~~~bash
systemctl daemon-reload
~~~



#### 启动

~~~bash
systemctl start redis.service 
systemctl enable redis.service
systemctl status redis.service
~~~



#### 查看版本

~~~bash
redis-server --version
Redis server v=8.2.2 sha=00000000:0 malloc=jemalloc-5.3.0 bits=64 build=9c47c46f5389f182

redis-cli --version
redis-cli 8.2.2
~~~



#### 客户端登录测试

~~~bash
redis-cli -h 192.168.10.33 -p 6379 -a '123'
~~~





