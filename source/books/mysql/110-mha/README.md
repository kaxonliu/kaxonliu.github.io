# MHA

## 环境

- MySQL 版本： mysql-8.0.41-2.el9_5.aarch64
- 操作系统： rockylinux9
- 主库 IP: `192.168.10.18`
- 从库 IP: `192.168.10.16` 和 `192.168.10.17`
- MHA 管理节点：`192.168.10.19`





## 安装部署 MHA

#### 1. 安装依赖

所有主机安装依赖

~~~bash
# 安装 epel-release 包
yum -y install epel-release

# 启用 CRB 仓库
yum config-manager --set-enabled crb
 
# 安装MHA依赖的perl包
yum install -y perl
yum install -y perl-DBD-MySQL perl-Config-Tiny perl-Log-Dispatch perl-Parallel-ForkManager
~~~



#### 2. 安装 mha4mysql-node

在所有主机安装 mha4mysql-node

~~~bash
wget https://qiniu.wsfnk.com/mha4mysql-node-0.58-0.el7.centos.noarch.rpm --no-check-certificate
rpm -ivh mha4mysql-node-0.58-0.el7.centos.noarch.rpm
~~~

#### 3. 安装 mha4mysql-manager

仅在管理主机上安装 manager

~~~bash
wget https://qiniu.wsfnk.com/mha4mysql-manager-0.58-0.el7.centos.noarch.rpm --no-check-certificate
rpm -ivh mha4mysql-manager-0.58-0.el7.centos.noarch.rpm
~~~



#### 4. 创建管理用户

在主库上创建一个管理用户，让 MHA 可以登陆 MySQL 做管理。

~~~sql
-- sql
create user 'mhaadmin'@'%' identified by '666';
grant all on *.* to 'mhaadmin'@'%';
flush privileges;
~~~



#### 5. 修改 MySQL 配置文件

各节点都要开启二进制日志和中继日志，关闭 `relay_log_purge` 功能。

注意：不要在配置文件中配置从库 `read-only` 属性，在命令行中设置。

**主库配置**

~~~ini
[mysqld]
server-id = 1
log-bin = mysql-bin
binlog_format = ROW
gtid_mode = ON
enforce-gtid-consistency = ON
relay_log_purge = 0
~~~



**从库配置**。注意每个从库的 `server-id` 都不一样，并且和主库的也不能一样。

~~~ini
server-id = 2
relay-log = mysql-relay-bin
gtid_mode = ON
enforce-gtid-consistency = ON
log-slave-updates = ON
relay_log_purge = 0
~~~

然后在从库登陆 MySQL，在命令行输入如下命令，使从库只读。

~~~sql
-- sql
set global read_only=1;
~~~



#### 6. 配置免密登陆

所有主机互做免密登陆

~~~bash
#创建秘钥对
#正常创建 ssh-keygen 需要交互 按回车，用以下方法跳过交互
ssh-keygen -t rsa -P '' -f ~/.ssh/id_rsa >/dev/null 2>&1

#发送公钥，包括自己
ssh-copy-id -i /root/.ssh/id_rsa.pub root@192.168.10.18
ssh-copy-id -i /root/.ssh/id_rsa.pub root@192.168.10.19
ssh-copy-id -i /root/.ssh/id_rsa.pub root@192.168.10.16
ssh-copy-id -i /root/.ssh/id_rsa.pub root@192.168.10.17
~~~



#### 7. 配置 MHA Manager

在 manager 节点上创建工作目录。

~~~bash
mkdir -p /service/mha/
mkdir /service/mha/app1
~~~

修改配置  `/service/mha/app1.cnf`

~~~ini
[server default]            
#日志存放路径
manager_log=/service/mha/manager.log
#定义工作目录位置
manager_workdir=/service/mha/app1
#binlog存放目录（如果三台数据库机器部署的路径不一样，可以将配置写到相应的server下面）
#master_binlog_dir=/usr/local/mysql/data
 
#设置ssh的登录用户名
ssh_user=root
#如果端口修改不是22的话，需要加参数，不建议改ssh端口
#否则后续如负责VIP漂移的perl脚本也都得改，很麻烦
ssh_port=22
 
#管理用户
user=mhaadmin
password=666
 
#复制用户
repl_user=repl  
repl_password=123
 
#检测主库心跳的间隔时间
ping_interval=1
 
[server1]
# 指定自己的binlog日志存放目录
master_binlog_dir=/var/lib/mysql
hostname=192.168.10.18
port=3306
 
[server2]
#暂时注释掉，先不使用
#candidate_master=1
#check_repl_delay=0
master_binlog_dir=/var/lib/mysql
hostname=192.168.10.16
port=3306
 
[server3]
master_binlog_dir=/var/lib/mysql
hostname=192.168.10.17
port=3306
 
# 1、设置了以下两个参数，则该从库成为候选主库，优先级最高
# 不管怎样都切到优先级高的主机，一般在主机性能差异的时候用         
candidate_master=1
# 不管优先级高的备选库，数据延时多久都要往那切
check_repl_delay=0
~~~



#### 8. 检查 mha 配置状态

~~~bash
#测试免密连接
1.使用mha命令检测ssh免密登录
masterha_check_ssh --conf=/service/mha/app1.cnf
    ALL SSH ... successfilly 表示ok了
 
2.使用mha命令检测主从状态
masterha_check_repl --conf=/service/mha/app1.cnf
    ... Health is OK
 
#如果出现问题，可能是反向解析问题，配置文件加上
    skip-name-resolve
#还有可能是主从状态，mha用户密码的情况
~~~

