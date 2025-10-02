# 主库切换优先级

故障迁移时，选择新主库有两种方式。



## 根据优先级

在 mha 配置文件中指定 server 的优先级，被指定的 server 在故障切换时优先成为新主库。

~~~ini
[server default]            
manager_log=/service/mha/manager.log
manager_workdir=/service/mha/app1
ssh_user=root
ssh_port=22
user=mhaadmin
password=Liuxu@123
repl_user=repl  
repl_password=Liuxu@123
ping_interval=1


# 初始 server1是主库
# 从库 为 server2 和 server3
[server1]
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

# server3 设置了以下两个参数，则该从库成为候选主库，优先级最高
# 不管怎样都切到优先级高的主机，一般在主机性能差异的时候用   
candidate_master=1
check_repl_delay=0
~~~





## 根据数据量是否最新

**配置 mha 文件时，所有的 server 不要配置 `candidate_master=1` 和 `check_repl_delay=0` 两个配置参数。**

主库宕机后，在故障迁移时，mha 会判断从库的数据情况，查找出更新到最新数据的从库，把这个从库作为新主库。

模拟时可以把某些从库的 IO 线程停掉，这些从库同步数据就会被延迟，其他从库同步的数据就会相对最新。

~~~sql
-- sql
stop slave io_thread;
~~~

然后在出库中插入新数据，再把主库关掉。mha 可以自动选择出最新数据的从库当主库。然后把所有从库重启开启 slave 同步数据。







