# Atlas 中间件

Atlas 是由 Qihoo 360 公司 Web 平台部基础架构团队开发维护的一个基于 MySQL 协议的数据中间层项目。它在MySQL官方推出的MySQL-Proxy 0.8.2 版本的基础上，修改了大量 bug，添加了很多功能特性。

主要功能：

- 读写分离
- 从库负载均衡
- IP 过滤
- 自动分表
- DBA 可平滑上下线 DB
- 自动摘除宕机的 DB



## 安装 Atlas

从[官方](https://github.com/Qihoo360/Atlas/releases )下载 RPM 包

~~~bash
wget https://github.com/Qihoo360/Atlas/releases/download/2.2.1/Atlas-2.2.1.el6.x86_64.rpm

sudo rpm –i Atlas-2.2.1.el6.x86_64.rpm
~~~

注意事项：

- Atlas 只能安装运行在 64 位的系统上
- Centos 6.X 安装 Atlas-XX.el6.x86_64.rpm
- 如果已经安装过旧版本 Atlas-1.0.3-1.x86_64，需要先使用 `sudo rpm –e Atlas-1.0.3-1.x86_64` 将旧版本移除后再安装新版本。
- 实际操作发现 el6 的包在 centos7.9 上也可以安装。





## 配置文件修改

Atlas 运行需要依赖一个配置文件（test.cnf）。在运行 Atlas 之前，需要对该文件进行配置。Atlas 的安装目录是 /usr/local/mysql-proxy，进入安装目录下的 conf 目录，可以看到已经有一个名为 test.cnf 的默认配置文件，我们只需要修改里面的某些配置项，不需要从头写一个配置文件。

~~~ini
# /usr/local/mysql-proxy/conf/test.cnf

[mysql-proxy]
# 这两项是用来进入Atlas的管理界面的
# 与后端连接的MySQL没有关系
admin-username = liuxu 
admin-password = 123   

# 主库的IP和端口(写节点)
proxy-backend-addresses = 192.168.10.101:3306

# 多个从库的IP和端口(只读节点),可设置多项，用逗号分隔。
# 如果想让主库也能分担读请求的话，只需要将主库信息加入到下面的配置项中。
proxy-read-only-backend-addresses = 192.168.10.102:3306,192.168.10.103:3306
 
# 用来登录 mysql 的账号和加密密码
# 用户名与密码之间用冒号分隔
# 主从数据库上需要先创建该用户并设置密码
# 密码使用PREFIX/bin目录下的加密程序encrypt加密
# /usr/local/mysql-proxy/bin/encrypt 123，得到123的加密密码
pwds = root:3yb5jEku5h4=, egon:3yb5jEku5h4= 

# 后台运行
daemon = true       
keepalive = true    # 监测节点心跳
event-threads = 4   # 并发数量，设置cpu核数一半
log-level = message   # 日志级别
log-path = /usr/local/mysql-proxy/log  # 日志目录
sql-log = On   # sql记录（可做审计）
proxy-address = 0.0.0.0:3306    # 业务连接端口
admin-address = 0.0.0.0:2345    # 管理连接端口
charset=utf8   # 字符集
~~~



## 创建账号

创建的 mysql 账号用来给 atlas 使用，就是配置在 atlas 配置文件中的 `pwds`

~~~sql
-- sql
grant all on *.* to 'root'@'%' identified by 'Liuxu@123';
~~~



## 启动服务

~~~bash
#1、启动，配置文件名为test.conf对应此处的test
/usr/local/mysql-proxy/bin/mysql-proxyd test start
 
#2、验证启动（没起来他也显示OK）
ps -ef|grep [m]ysql-proxy
netstat -lntup|grep [m]ysql-proxy
 
#3、查看日志定位问题
tail -f /usr/local/mysql-proxy/log/test.log
~~~



## Atlas代理

altas 启动后，随便找一台机器模拟客户端，客户端访问 atlas 服务配置的 ip + port `proxy-address = 0.0.0.0:3306 `，假设 atlas 服务所在机器的 IP 为 192.168.10.100。再配合使用配置文件中 `pwds` 配置的用户名和密码，即可通过 atlas 代理读写数据库集群。

~~~bash
mysql -u root -p -h 192.168.10.100 -P 3306
~~~





## Atlas 管理界面

atlas 启动后 使用配置文件中的账号信息 `admin-username` 和 `admin-password` 登录管理界面，登录地址为配置文件中的 `admin-address = 0.0.0.0:2345`

~~~bash
mysql -u liuxu -p 123 -h 127.0.0.1 -P 2345
~~~

登录后，可以使用如下命令管理 mysql 节点。

~~~sql
MySQL [(none)]> select * from help;
+----------------------------+---------------------------------------------------------+
| command                    | description                                             |
+----------------------------+---------------------------------------------------------+
| SELECT * FROM help         | shows this help                                         |
| SELECT * FROM backends     | lists the backends and their state                      |
| SET OFFLINE $backend_id    | offline backend server, $backend_id is backend_ndx's id |
| SET ONLINE $backend_id     | online backend server, ...                              |
| ADD MASTER $backend        | example: "add master 127.0.0.1:3306", ...               |
| ADD SLAVE $backend         | example: "add slave 127.0.0.1:3306", ...                |
| REMOVE BACKEND $backend_id | example: "remove backend 1", ...                        |
| SELECT * FROM clients      | lists the clients                                       |
| ADD CLIENT $client         | example: "add client 192.168.1.2", ...                  |
| REMOVE CLIENT $client      | example: "remove client 192.168.1.2", ...               |
| SELECT * FROM pwds         | lists the pwds                                          |
| ADD PWD $pwd               | example: "add pwd user:raw_password", ...               |
| ADD ENPWD $pwd             | example: "add enpwd user:encrypted_password", ...       |
| REMOVE PWD $pwd            | example: "remove pwd user", ...                         |
| SAVE CONFIG                | save the backends to config file                        |
| SELECT VERSION             | display the version of Atlas                            |
+----------------------------+---------------------------------------------------------+
16 rows in set (0.00 sec)

MySQL [(none)]> select * from backends;
+-------------+---------------------+-------+------+
| backend_ndx | address             | state | type |
+-------------+---------------------+-------+------+
|           1 | 192.168.10.101:3306 | up    | rw   |
|           2 | 192.168.10.102:3306 | up    | ro   |
|           3 | 192.168.10.103:3306 | up    | ro   |
+-------------+---------------------+-------+------+
3 rows in set (0.00 sec)
~~~

