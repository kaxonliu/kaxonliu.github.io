# 手动迁移主库配置 VIP 漂移



#### 1. mha 配置文件指定脚本路径

~~~ini
# /service/mha/app1.cnf 

[server default]
#指定手动切换主库后，执行的vip迁移脚本路径
master_ip_online_change_script="/service/mha/master_online_change"
~~~



#### 2. 编写脚本文件

要求所有数据库服务器的网卡名称必须一致。

~~~bash
#!/bin/bash
source /root/.bash_profile
 
vip=`echo '192.168.10.88/24'`  #设置VIP
key=`echo '1'`
 
command=`echo "$1" | awk -F = '{print $2}'`
orig_master_host=`echo "$2" | awk -F = '{print $2}'`
new_master_host=`echo "$7" | awk -F = '{print $2}'`
orig_master_ssh_user=`echo "${12}" | awk -F = '{print $2}'`
new_master_ssh_user=`echo "${13}" | awk -F = '{print $2}'`
 
#要求服务的网卡识别名一样，都为eth0(这里是)
stop_vip=`echo "ssh root@$orig_master_host /usr/sbin/ifconfig ens33:$key down"`
start_vip=`echo "ssh root@$new_master_host /usr/sbin/ifconfig ens33:$key $vip"`
 
if [ $command = 'stop' ]
  then
    echo -e "\n\n\n****************************\n"
    echo -e "Disabled thi VIP - $vip on old master: $orig_master_host \n"
    $stop_vip
    if [ $? -eq 0 ]
      then
    echo "Disabled the VIP successfully"
      else
    echo "Disabled the VIP failed"
    fi
    echo -e "***************************\n\n\n"
  fi
 
if [ $command = 'start' -o $command = 'status' ]
  then
    echo -e "\n\n\n*************************\n"
    echo -e "Enabling the VIP - $vip on new master: $new_master_host \n"
    $start_vip
    if [ $? -eq 0 ]
      then
    echo "Enabled the VIP successfully"
      else
    echo "Enabled the VIP failed"
    fi
    echo -e "***************************\n\n\n"
fi
~~~

#### 3. 授权可执行权限

添加执行权限，否则 mha 无法启动

~~~bash
chmod +x /service/mha/master_online_change
~~~

#### 4. 手动绑定 vip 到主库

规划 VIP 为 192.168.10.88

~~~bash
# ssh到主库所在服务器，然后执行
ifconfig ens33:1 192.168.10.88/24
~~~

#### 5. 手动切换主库

手动切换主库，验证 VIP 飘移。

~~~bash
# 先停掉mha
masterha_stop --conf=/service/mha/app1.cnf
 
# 将主库切换到192.168.10.102
masterha_master_switch --conf=/service/mha/app1.cnf --master_state=alive --new_master_host=192.168.10.102 --orig_master_is_new_slave --running_updates_limit=10000 --interactive=0
~~~

