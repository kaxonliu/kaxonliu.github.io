# 自动迁移主库配置 VIP 漂移

VIP 漂移有两种方式：使用 keepalived 、使用 mha 自带脚本（推荐）。

下文使用 mha 自带脚本的方式配置 vip。



#### 1. mha 配置文件指定脚本路径

~~~ini
# vim /etc/mha/app1.cnf

[server default]
#指定自动化切换主库后，执行的 vip 迁移脚本路径
master_ip_failover_script=/service/mha/master_ip_failover
~~~



#### 2. 编写脚本文件

要求所有数据库服务器的网卡名称必须一致。

~~~bash
#!/usr/bin/env perl
use strict;
use warnings FATAL => 'all';
use Getopt::Long;
 
my (
$command,   $ssh_user,  $orig_master_host,
$orig_master_ip,$orig_master_port, $new_master_host, $new_master_ip,$new_master_port
);
 
#定义VIP变量
my $vip = '192.168.10.88/24';
my $key = '1';
my $ssh_start_vip = "/sbin/ifconfig ens33:$key $vip";
my $ssh_stop_vip = "/sbin/ifconfig ens33:$key down"; 
 
GetOptions(
'command=s'     => \$command,
'ssh_user=s'        => \$ssh_user,
'orig_master_host=s'    => \$orig_master_host,
'orig_master_ip=s'  => \$orig_master_ip,
'orig_master_port=i'    => \$orig_master_port,
'new_master_host=s' => \$new_master_host,
'new_master_ip=s'   => \$new_master_ip,
'new_master_port=i' => \$new_master_port,
);
 
exit &main();
 
sub main {
print "\n\nIN SCRIPT TEST====$ssh_stop_vip==$ssh_start_vip===\n\n";
if ( $command eq "stop" || $command eq "stopssh" ) {
my $exit_code = 1;
eval {
print "Disabling the VIP on old master: $orig_master_host \n";
&stop_vip();
$exit_code = 0;
};
if ($@) {
warn "Got Error: $@\n";
exit $exit_code;
}
exit $exit_code;
}
 
elsif ( $command eq "start" ) {
my $exit_code = 10;
eval {
print "Enabling the VIP - $vip on the new master - $new_master_host \n";
&start_vip();
$exit_code = 0;
};
 
if ($@) {
warn $@;
exit $exit_code;
}
exit $exit_code;
}
 
elsif ( $command eq "status" ) {
print "Checking the Status of the script.. OK \n";
exit 0;
}
else {
&usage();
exit 1;
}
}
 
sub start_vip() {
`ssh $ssh_user\@$new_master_host \" $ssh_start_vip \"`;
}
sub stop_vip() {
return 0 unless ($ssh_user);
`ssh $ssh_user\@$orig_master_host \" $ssh_stop_vip \"`;
}
sub usage {
print
"Usage: master_ip_failover --command=start|stop|stopssh|status --orig_master_host=host --orig_master_ip=ip --orig_master_port=port --new_master_host=host --new_master_ip=ip --new_master_port=port\n";
}
~~~

#### 3. 授权可执行权限

~~~bash
chmod +x /service/mha/master_ip_failover
~~~

#### 4. 手动绑定 vip 到主库

规划 VIP 为 192.168.10.88

~~~bash
# ssh到主库所在服务器，然后执行
ifconfig ens33:1 192.168.10.88/24
~~~

#### 5. 模拟插入数据

在三个数据库服务器之外的服务器上模拟客户端用户，往 vip 对应的数据库插入数据。

~~~
# 强调：先执行一下ssh root@192.168.15.250，把yes输入一下，退出后
# 再执行下述脚本
for i in `seq 1 10000`
do
    mysql -uroot -h 192.168.10.88 -p'Liuxu@123' -e "insert into db1.t1 values($i);"
    sleep 1
    echo "$i ok"
done
~~~

#### 6. 启动 MHA

~~~bash
nohup masterha_manager --conf=/service/mha/app1.cnf --remove_dead_master_conf --ignore_last_failover < /dev/null > /service/mha/manager.log 2>&1 &
~~~

#### 7. 观察迁移过程

使用 `systemctl stop mysqld`  把主库服务停止，观察客户端的异常情况，同时观看 mha 迁移过程日志。