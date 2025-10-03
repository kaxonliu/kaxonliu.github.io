# binlog server 备份

准备一个 server 安装一个 MySQL 服务，和主库版本保持一致。这个 server 专门用来备份主库的 binlog，不参与 mha 集群。

#### 1. 新建 binlog 目录

创建备份文件夹，并设置权限。

~~~bash
mkdir -p /bak/binlog/
chown -R mysql.mysql /bak/binlog
~~~



#### 2. 手动执行备份 binlog

查看所有从库 `show slave status\G`，确认同步 binlog 起始文件。

~~~bash
cd /bak/binlog
mysqlbinlog  -R --host=192.168.10.102 --user=mhaadmin --password=Liuxu@123 --raw --stop-never mysql-bin.000012 &
~~~



#### 4. 登录主库验证

看到 binlog server 主机上 /bak/binlog 目录下新增日志。

~~~sql
-- sql
flush logs; 
~~~



#### 5. 停止 MHA

~~~bash
masterha_stop --conf=/service/mha/app1.cnf
~~~



#### 6. 编辑 MHA 配置文件

app1.cnf 文件增加如下配置，表示 binlog server 不参与 mha 集群，只用来同步 binlog

~~~ini
[binlog1]
no_master=1
# binlogserver主机的ip地址
hostname=192.168.10.103
# 不能跟当前机器数据库的binlog存放目录一样
master_binlog_dir=/bak/binlog/
~~~

#### 7. 启动 mha

~~~bash
nohup masterha_manager --conf=/service/mha/app1.cnf --remove_dead_master_conf --ignore_last_failover < /dev/null > /service/mha/manager.log 2>&1 &
~~~

