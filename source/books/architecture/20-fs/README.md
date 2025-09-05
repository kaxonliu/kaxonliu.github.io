# 文件服务器层

文件服务器层的用途是提供集中的共享文件。生产服务器一般使用如下四种文件服务器层。

1. 单 NFS + 备份 NFS。好处是简单便于维护，缺点是单点故障，需要人工干预。
2. DRBM+HeartBeat + NFS 高可用文件服务器。维护方便没有单点故障，但不适合大流量高并发场景。
3. 分布式文件系统 MFS、Gluster、CEPH。
4. 自建分布式文件系统。

下面主要介绍最简单入门的 NFS。



## 什么是NFS

NFS 是 Network File System 的缩写，即网络文件系统。NFS 主要功能是通过局域网络让不同的主机系统之间可以共享文件或目录。有了 NFS ，可以实现多台服务器之间数据共享；实现多台服务器之间数据一致。



## 如何使用 NFS

使用很简单，分为两个部分。在服务端使用 nfs 共享一个文件夹，在客户端挂载上服务端暴露的文件夹。



## NFS 服务端

#### 1. 关闭防火墙 和 selinux

~~~bash
# 关防火墙
systemctl stop firewalld
systemctl disable firewalld

# 关selinux
sed -i 's#SELINUX=enforcing#SELINUX=disabled#g' /etc/selinux/config
reboot
~~~

#### 2. 安装 NFS 和 rpcbind

~~~bash
yum install -y nfs-utils rpcbind
 
#注意：
Centos6 需要安装rpcbind
Centos7 默认已经安装好了rpcbind，并且默认是开机自启动
~~~

#### 3. 配置 nfs

NFS默认的配置文件是 `/etc/exports`

~~~bash
echo "/data 192.168.10.0/24(rw,sync,all_squash,anonuid=1001,anongid=1001)" > /etc/exports

# 含义
# /data 表示共享的文件夹
# 192.168.10.0/24 表示NFS 允许连接的客户端ip
# (rw,sync,all_squash,anonuid=1001,anongid=1001) 表示允许操作的权限
# 配置了all_squash 表示所有来自客户读端的用户和用户组，都会被影射为一个匿名用户和组
~~~

#### 4. 创建共享目录

~~~bash
mkdir /data
~~~

#### 5. 创建匿名用户和组，并授权共享目录

~~~bash
useradd nfsnobody -s /sbin/nologin -M

chown -R nfsnobody.nfsnobody /data/
~~~

#### 6. 启动服务

~~~bash
systemctl start rpcbind

# centos7.9
systemctl start nfs

# rockylinux9 使用 nfs-server
systemctl start nfs-server
~~~

#### 7. 验证 nfs 配置

~~~bash
[root@rocky data]# cat /var/lib/nfs/etab
/data	/data	192.168.10.0/24(rw,sync,wdelay,hide,nocrossmnt,secure,root_squash,all_squash,no_subtree_check,secure_locks,acl,no_pnfs,anonuid=1001,anongid=1001,sec=sys,rw,secure,root_squash,all_squash)
~~~



## NFS 客户端

#### 1. 关闭防火墙和selinux

#### 2. 安装 nfs

~~~bash
yum install -y rpcbind nfs-utils
~~~

#### 3. 查看挂载点

使用 `showmount` 命令查看时指定服务端 IP，返回可以挂载的信息。

~~~bash
[root@rocky ~]# showmount -e 192.168.10.15
Export list for 192.168.10.15:
/data 192.168.10.0/24
~~~

#### 4. 挂载

~~~bash
[root@rocky ~]# mkdir /data
[root@rocky ~]# mount -t nfs 192.168.10.15:/data /data
[root@rocky ~]# df -h
Filesystem           Size  Used Avail Use% Mounted on
devtmpfs             4.0M     0  4.0M   0% /dev
tmpfs                977M     0  977M   0% /dev/shm
tmpfs                391M  6.7M  385M   2% /run
efivarfs             256K   32K  225K  13% /sys/firmware/efi/efivars
/dev/nvme0n1p3       8.5G  1.5G  7.1G  17% /
/dev/nvme0n1p1       500M  7.4M  493M   2% /boot/efi
tmpfs                196M     0  196M   0% /run/user/0
192.168.10.15:/data   18G  1.8G   16G  10% /data
~~~

#### 5. 写数据测试

~~~bash
[root@rocky data]# touch abc
[root@rocky data]# ls -l abc
-rw-r--r--. 1 nfsnobody nfsnobody 0 Sep  4 22:06 abc
~~~

>**注意**：客户端看到文件的属主和属组是 uid 和 gid，这里是因为在客户端预先创建了匿名用户和组。

#### 6. 设置开机挂载

~~~bash
#编辑fstab文件
[root@web01 ~]# vim /etc/fstab
192.168.10.15:/data   /data         nfs     defaults        0 0
 
 
#验证fstab是否写正确
[root@web01 ~]# mount -a
~~~



## 统一用户

 服务端和客户端机器上**统一用户**，方便管理，并且是普通用户启动，更安全。

#### 所有机器都是用相同的用户

~~~bash
[root@web01 ~]# groupadd www -g 666
[root@web01 ~]# useradd www -u 666 -g 666
 
[root@web02 ~]# groupadd www -g 666
[root@web02 ~]# useradd www -u 666 -g 666
 
[root@nfs ~]# groupadd www -g 666
[root@nfs ~]# useradd www -u 666 -g 666
~~~





## NFS 单点故障如何解决

解决单点故障可以制作镜像站，具体实现如下，

- 使用 rsync 实现增量拷贝。
- 使用 inotify 实现自动检测文件更改。
- 使用 sersync（对 rsync + inotify 的封装）。
- 使用分布式文件存储服务。



## 远程镜像站-实时同步

使用 `inotify` 自动检测主文件服务器文件变化，使用 `rsync` 向备份文件服务器增量同步，实现一个远程镜像站做到实时同步。

[rsync 的基本使用请点击阅读](../../linux/services/rsync/index.html)。下面先介绍 `inotify` 的使用，然后看两者如何配合实现一个镜像站。



### inofity 基本使用

#### 安装 inotify

`inotify` 由 `inotify-tools` 包提供。安装之前，确保内核高于 2.6.13, 并检查 `/proc/sys/fs/inotify/` 目录下面由以下三项，这表示系统支持 `inotify`监控。

~~~bash
[root@rocky data]# ls -l /proc/sys/fs/inotify/
total 0
-rw-r--r--. 1 root root 0 Sep  5  2025 max_queued_events
-rw-r--r--. 1 root root 0 Sep  5  2025 max_user_instances
-rw-r--r--. 1 root root 0 Sep  5  2025 max_user_watches
~~~

epel 源提供 `inotify-tools` 工具

~~~bash
yum install epel-release* -y
yum install inotify-tools -y
~~~

`inotify` 安装后提供了两个命令可以使用

~~~bash
[root@rocky data]# rpm -ql inotify-tools | grep bin/
/usr/bin/inotifywait
/usr/bin/inotifywatch
~~~

#### 监控文件夹内的变动

使用 `inotifywait` 监听 `/data` 文件夹内的变化，执行如下命令，指定监听的信号类型，一旦有文件的增删改就会手动信号，信号按照指定的格式发出。

~~~bash
inotifywait -mrq --timefmt '%Y-%m-%d %H:%M:%S' --format '%w%f:%Xe:%T' -e create,delete,modify,move,attrib,close_write /data
~~~

比如新建文件 `touch 3.txt` 就会收到如下信息

~~~bash
/data/3.txt:CREATE:2025-09-05 15:34:44
/data/3.txt:ATTRIB:2025-09-05 15:34:44
/data/3.txt:CLOSE_WRITEXCLOSE:2025-09-05 15:34:44
~~~



#### inotify 信号类型

| 类型          | 解释说明                                           |
| :------------ | :------------------------------------------------- |
| access        | 文件被访问（读取）。                               |
| modify        | 文件被修改（内容发生改变）。                       |
| attrib        | 文件的元数据发生变化（如权限、时间戳等）。         |
| close_write   | 以写模式打开的文件被关闭。表示文件写入完成并保存。 |
| close_nowrite | 以只读模式打开的文件被关闭。                       |
| close         | close_write 和 close_nowrite 的结合。              |
| moved_from    | 文件被移出受监控的目录（移动操作的来源）。         |
| moved_to      | 文件被移入受监控的目录（移动操作的目标）。         |
| move          | moved_from 和 moved_to 的结合。                    |
| create        | 在受监控的目录内创建了新文件或子目录。             |
| delete        | 在受监控的目录内有文件或子目录被删除。             |
| delete_self   | 受监控的文件或目录本身被删除。监控停止。           |
| move_self     | 受监控的文件或目录本身被移动或重命名。             |



### rsync + notify 脚本

在主文件服务器上执行同步镜像的脚本，把变动的数据同步到远程镜像文件服务器。如果 rsync 远程同步使用 rsync 协议，则备份服务器要开启 rsyncd 服务，否则不需要。注意：不论 rsync 使用 ssh 协议还是 rsync 协议，主文件服务器和备份文件服务器都要安装 rsync 命令。

~~~bash
[root@rocky scripts]# cat rsync_inotify.sh 
watch_dir=/data/               # 本地被监控目录
user="rsync_user"                    # 虚拟用户
export RSYNC_PASSWORD=123      # 虚拟用户密码
module="backup"                   # 远程模块名
ip=10.10.98.66               # 远程主机ip
 
# 先整体同步一次
rsync -azc --delete ${watch_dir} ${user}@${ip}::${module}
 
# 切换到被监控目录下，然后用inotifywait监控./目录，这样后期就可以用-R选项同步新增的子目录
cd $watch_dir  
/usr/bin/inotifywait -mrq --timefmt '%Y-%m-%d %H:%M:%S' --format '%w%f:%Xe:%T' -e create,delete,modify,move,attrib,close_write ./ \
--exclude=".*.swp" | \
while read line
do
    # $line的输出format为：文件路径:事件:时间
    FILE=$(echo $line | awk -F: '{print $1}')  # 获取文件的绝对路径
    EVENT=$(echo $line | awk -F: '{print $2}') # 获取监控的事件
 
    # 监控到对文件的下述行为后，只把文件同步到远端
    if [[ $EVENT =~ 'CREATE' ]] || [[ $EVENT =~ 'MODIFY' ]] || [[ $EVENT =~ 'CLOSE_WRITE' ]] || [[ $EVENT =~ 'MOVED_TO' ]] || [[ $EVENT =~ 'ATTRIB' ]];then
        rsync -azcR ${FILE} ${user}@${ip}::${module}
    fi
 
    # 监控到涉及到目录的改动，将目录同步到远端，例如用dirname ${FILE}获取目录
    if [[ $EVENT =~ 'DELETE' ]] || [[ $EVENT =~ 'MOVED_FROM' ]];then
        rsync -azcR --delete $(dirname ${FILE}) ${user}@${ip}::${module} &>/dev/null
    fi
done &
# 末尾的&符号，代表在子shell中提交命令，这样进程的ppid就变为1，当前窗口关闭，该进程依然存活
~~~

