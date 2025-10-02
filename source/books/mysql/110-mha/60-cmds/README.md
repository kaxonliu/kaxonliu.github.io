# 命令汇总

~~~bash
#1、重置slave
stop slave;
reset slave;
 
#2、查看
SHOW SLAVE STATUS;          #查看从库复制状态
SHOW MASTER STATUS;         #查看当前binlog位点
SHOW SLAVE HOSTS;           #查看从库列表
 
#3、重做slave指向主库
stop slave;
CHANGE MASTER TO MASTER_HOST='192.168.15.100', MASTER_PORT=3306, MASTER_AUTO_POSITION=1, MASTER_USER='egon', MASTER_PASSWORD='123';
start slave;
 
#4、停止mha
masterha_stop --conf=/service/mha/app1.cnf
 
#5、查看ssh与主从状态
masterha_check_ssh --conf=/service/mha/app1.cnf
masterha_check_repl --conf=/service/mha/app1.cnf
 
#6、启动mha
nohup masterha_manager --conf=/service/mha/app1.cnf --remove_dead_master_conf --ignore_last_failover < /dev/null > /service/mha/manager.log 2>&1 &
 
#7、查看mha状态
masterha_check_status --conf=/service/mha/app1.cnf
 

#8、修改mha配置文件
cat >> /service/mha/app1.cnf << EOF
[server default]
manager_log=/service/mha/manager.log
manager_workdir=/service/mha/app1
master_ip_failover_script=/service/mha/master_ip_failover
master_ip_online_change_script="/service/mha/master_online_change"
password=666
ping_interval=1
repl_password=123
repl_user=egon
ssh_port=22
ssh_user=root
user=mhaadmin
 
[server1]
hostname=192.168.10.101
master_binlog_dir=/var/lib/mysql
port=3306
 
[server2]
hostname=192.168.10.102
master_binlog_dir=/var/lib/mysql
port=3306
 
[server3]
hostname=192.168.10.103
master_binlog_dir=/var/lib/mysql
port=3306
 
EOF
~~~

