# MHA

MHA 就是 Master High Availability Manager for MySQL，MySQL 高可用性管理工具。它是一个用于自动化 MySQL 主从复制架构 中 主库故障切换 和 从库提升 的 Perl 脚本套件。简单来说，它的核心目标就是确保 MySQL 数据库在主机宕机时能够快速、自动地恢复服务，最大限度地减少停机时间。

MHA 通过引入一个 MHA Manager 节点来自动化整个故障切换过程。

**主要组件：**

- **MHA Manager**: 管理节点，可以单独部署在一台服务器上，它负责监控所有 MySQL 节点（主库和从库），并在主库故障时协调故障转移。
- **MHA Node**: 数据节点，需要安装在**每一台** MySQL 服务器上（包括主库和所有从库）。它负责执行 Manager 发出的具体命令，如解析二进制日志、应用差异日志等。

**工作原理：**

1. **监控**: MHA Manager 会定期 ping 主库，检查其是否存活。
2. **故障检测**: 一旦检测到主库无法访问，Manager 会通过 SSH 连接到各个 Node 进行二次确认。
3. **故障切换**:
   - **识别最新从库**: Manager 会识别出哪个从库拥有最新的数据（即复制延迟最小）。
   - **保存二进制日志**: 如果可能，Manager 会尝试从宕机的主库服务器上保存尚未被复制的二进制日志。
   - **应用差异中继日志**: Manager 会从中继日志中识别出其他从库比“最新从库”缺少的事务，并将这些差异事务应用到其他从库上，确保所有从库数据一致。
   - **提升新主库**: 将选定的“最新从库”提升为新的主库。
   - **重定向其他从库**: 让其他所有从库开始从这台新的主库进行复制。
4. **虚拟 IP 切换（可选）**: MHA 可以配合脚本来完成虚拟 IP 的切换，使得应用程序无需修改配置就能连接到新的主库。





## 环境

- MySQL 版本： mysql-community-server-5.7.44-1.el7.x86_64
- 操作系统： centos7.9
- 主库 IP: `192.168.10.101`
- 从库 IP: `192.168.10.102` 和 `192.168.10.103`
- MHA 管理节点：`192.168.10.100`





## 安装部署 MHA

#### 1. GTID 同步配置

主从数据库服务器使用 gtid 的复制方式，所有数据库服务器都开启 binlog 日志和 relay log，并且关闭 `relay_log_purge` 功能。各 从节点在命令行模式下显示开启 `read_only` （不要配置到文件中）。

**配置文件**。主从配置除了 `server-id` 不同，其他配置保持一样。

~~~ini
[mysqld]
datadir=/var/lib/mysql

# 主从相关配置
server-id=100
binlog_format=row
log-bin=mysql-bin
relay-log = mysql-relay-bin

gtid-mode=on 
enforce-gtid-consistency=true 
# slave更新是否记入日志（5.6必须的）
log-slave-updates=1 
 # 关闭relay_log自动清除功能，保障故障时的数据一致
relay_log_purge = 0
# 不要域名解析
skip-name-resolve 
~~~

**从库设置只读**。只在从库设置只读

~~~sql
set global read_only=1;
~~~

**重启服务**

~~~bash
systemctl restart mysqld
~~~

**主库查看状态**

~~~sql
show master status\G
~~~

**从库开启同步**

~~~sql
-- sql
stop slave;
reset slave;

change master to 
    master_host='192.168.10.101',
    master_user='repl',
    master_password='Liuxu@123',
    MASTER_AUTO_POSITION=1;
    
start slave;
show slave status\G
~~~

**在主库上创建 mha 管理账号**

~~~sql
-- sql
grant all on *.* to 'mhaadmin'@'%' identified by 'Liuxu@123';
flush privileges;
~~~



#### 2. 配置免密登录

~~~bash
-- bash
#创建秘钥对
#正常创建 ssh-keygen 需要交互 按回车，用以下方法跳过交互
ssh-keygen -t rsa -P '' -f ~/.ssh/id_rsa >/dev/null 2>&1
#发送公钥，包括自己
ssh-copy-id -i /root/.ssh/id_rsa.pub root@192.168.10.100
ssh-copy-id -i /root/.ssh/id_rsa.pub root@192.168.10.101
ssh-copy-id -i /root/.ssh/id_rsa.pub root@192.168.10.102
ssh-copy-id -i /root/.ssh/id_rsa.pub root@192.168.10.103
~~~



#### 3. 安装软件依赖包

~~~bash
# 安装yum源
yum -y install epel-release
 
# 安装MHA依赖的perl包
yum install -y perl-DBD-MySQL perl-Config-Tiny perl-Log-Dispatch perl-Parallel-ForkManager
~~~



#### 4. 在所有主机上安装 node 包

~~~bash
wget https://qiniu.wsfnk.com/mha4mysql-node-0.58-0.el7.centos.noarch.rpm --no-check-certificate
rpm -ivh mha4mysql-node-0.58-0.el7.centos.noarch.rpm
~~~



#### 5. 在管理机器安装 manager 包

~~~bas
wget https://qiniu.wsfnk.com/mha4mysql-manager-0.58-0.el7.centos.noarch.rpm --no-check-certificate
rpm -ivh mha4mysql-manager-0.58-0.el7.centos.noarch.rpm
~~~



#### 6. 配置mha manager

创建工作目录

~~~bash
mkdir -p /service/mha/
mkdir /service/mha/app1
~~~



#### 7. 创建配置

在文件 `/service/mha/app1.cnf` 文件中保存如下配置。

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
password=Liuxu@123
 
#复制用户
repl_user=repl  
repl_password=Liuxu@123

#检测主库心跳的间隔时间
ping_interval=1
 
[server1]
# 指定自己的binlog日志存放目录
master_binlog_dir=/var/lib/mysql
hostname=192.168.10.101
port=3306
 
[server2]
master_binlog_dir=/var/lib/mysql
hostname=192.168.10.102
port=3306
 
[server3]
master_binlog_dir=/var/lib/mysql
hostname=192.168.10.103
port=3306
 
# 1、设置了以下两个参数，则该从库成为候选主库，优先级最高
# 不管怎样都切到优先级高的主机，一般在主机性能差异的时候用         
candidate_master=1
# 不管优先级高的备选库，数据延时多久都要往那切
check_repl_delay=0
~~~



#### 8. 检查 MHA 配置状态

~~~bash
# 使用mha命令检测ssh免密登录
# 看到 ALL SSH ... successfilly 表示ok了
masterha_check_ssh --conf=/service/mha/app1.cnf
    
# 使用mha命令检测主从状态
# 看到... Health is OK
masterha_check_repl --conf=/service/mha/app1.cnf
~~~



#### 9. 启动 mha

~~~bash
nohup masterha_manager --conf /service/mha/app1.cnf --remove_dead_master_conf \
  --ignore_last_failover < /dev/null > /service/mha/manager.log 2>&1 &
~~~



#### 10. 测试故障自动迁移

把主库停掉，观察 mha 的日志，等完成故障修复后。把停掉的主库重启，然后配置成从库执行新的主库。

时时查看日志

~~~bash
tail -f /service/mha/manager.log
~~~

停掉主库

~~~bash
# 192.168.10.101
systemctl stop mysqld
~~~

查看新主库

~~~bash
[root@manager mha]# grep -i 'change master to' /service/mha/manager.log 
Thu Oct  2 10:58:30 2025 - [info]  All other slaves should start replication from here. Statement should be: CHANGE MASTER TO MASTER_HOST='192.168.10.103', MASTER_PORT=3306, MASTER_AUTO_POSITION=1, MASTER_USER='repl', MASTER_PASSWORD='xxx';
~~~

启动宕机的主库

~~~bash
# 192.168.10.101
systemctl start mysqld
~~~

把宕机的主库配置成从库

~~~sql
--- sql
stop slave;
reset slave;

change master to 
    master_host='192.168.10.103',
    master_user='repl',
    master_password='Liuxu@123',
    MASTER_AUTO_POSITION=1;
    
start slave;
show slave status\G
~~~

在 mha 配置文件中添加 server1（server1 是之前的宕机的主库）

~~~ini
[server1]
hostname=192.168.10.101
master_binlog_dir=/var/lib/mysql
port=3306
~~~

再次启动 mha。（故障迁移一次 mha 就停掉了，故障修复后需要重新启动。）

~~~bash
nohup masterha_manager --conf /service/mha/app1.cnf --remove_dead_master_conf \
  --ignore_last_failover < /dev/null > /service/mha/manager.log 2>&1 &
~~~

检查 mha 启动状态

~~~bash
[root@manager mha]# masterha_check_status --conf=/service/mha/app1.cnf 
app1 (pid:1831) is running(0:PING_OK), master:192.168.10.103
~~~

