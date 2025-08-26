# 系统优化

## 1. 显示输出

修改配置文件 `/etc/bashrc`的第41行，修改为如下内容

~~~bash
[ "$PS1" = "\\s-\\v\\\$ " ] && PS1="[\[\e[34;1m\]\u@\[\e[0m\]\[\e[32;1m\]\H\[\e[0m\] \[\e[31;1m\]\w\[\e[0m\]]\\$ "
~~~



## 2. 配置登陆欢迎信息

修改配置文件 `/etc/motd`，修改内容如下

~~~bash
[root@localhost ~]# cat /etc/motd 
+--------------------------------------------+
|                                            |
|    你当前登录的是支付业务后台数据库服务          |
|    请不要删库                                |
|                                            |
+--------------------------------------------+
~~~

>补充，通常我们会清空/etc/issue、/etc/issue.net的内容，去除系统及内核版本登录前的屏幕显示。





## 3. 配置yum仓库和包更新

配置 阿里云的 yum 源头，请参考 https://kaxonliu.github.io/books/linux/package-manage/index.html

安装常用的库

~~~bash
yum install net-tools vim tree htop iftop iotop  bash-completion bash-completion-extras lrzsz sysstat sl lsof unzip telnet nmap nc psmisc dos2unix bash-completion wget  nethogs ntpdate nfsutils rsync glances gcc gcc-c++ glibc yum-utils httpd-tools -y
 
yum -y install tree nmap sysstat lrzsz  telnet bash-completion bash-completion-extras vim  lsof  net-tools rsync ntpdate nfs-utils
~~~



## 4. 配置主机名

~~~bash
hostnamectl set-hostname 主机名
~~~



## 5. 添加 hosts 文件实现集群主机名解析

修改配置文件 `/etc/hosts`，然后集群内的那个机器都发一份

~~~bash
172.16.10.11 nc1
172.16.10.12 nc2 
172.16.10.13 nc3
...
~~~



## 6. 关闭 selinux

SELinux（Security-Enhanced Linux）是美国国家安全局（NSA）对于强制访问控制的实现，权限控制的非常严格，使用不方便。

**临时关闭**

~~~bash
setenforce  0
~~~

**永久关闭**

~~~bash
sed -i 's#SELINUX=enforcing#SELINUX=disabled#g' /etc/selinux/config
~~~



## 7. 关闭防火墙

在企业环境中，一般只有配置外网IP的linux服务器才需要开启防火墙，但即使是有外网IP，对于高并发高流量的业务服务器仍是不能开的，因为会有较大性能损失，导致网站访问很慢，这种情况下只能在前端加更好的硬件防火墙了。

**临时关闭**

~~~bash
systemctl  stop firewalld
~~~

**永久关闭**

~~~bash
systemctl  stop firewalld
systemctl  disable  firewalld

# 清空规则
iptables -t raw -F 
iptables -t mangle -F 
iptables -t nat -F
iptables -t filter -F
~~~



## 8. 自动时间同步 chrony

Centos7.9以后的版本，`chrony` 都是默认的网络时间服务。

#### 安装

安装 `chrony` 之后，可以使用两个命令，一个是客户端 `chronyc` 命令，另一个是服务端 `chornyd`服务。

~~~bash
yum install -y chrony
~~~



#### 服务端

开启 `chronyd` 服务后，修改配置文件 `/etc/chrony.conf`。使用阿里云提供的时间服务。

~~~bash
cat > /etc/chrony.conf << EOF
server ntp1.aliyun.com iburst minpoll 4 maxpoll 10
server ntp2.aliyun.com iburst minpoll 4 maxpoll 10
server ntp3.aliyun.com iburst minpoll 4 maxpoll 10
server ntp4.aliyun.com iburst minpoll 4 maxpoll 10
server ntp5.aliyun.com iburst minpoll 4 maxpoll 10
server ntp6.aliyun.com iburst minpoll 4 maxpoll 10
server ntp7.aliyun.com iburst minpoll 4 maxpoll 10
driftfile /var/lib/chrony/drift
makestep 10 3
rtcsync
allow 0.0.0.0/0
local stratum 10
keyfile /etc/chrony.keys
logdir /var/log/chrony
stratumweight 0.05
noclientlog
logchange 0.5
EOF
~~~

重启服务

~~~bash
systemctl restart chronyd.service # 最好重启，这样无论原来是否启动都可以重新加载配置
systemctl enable chronyd.service
systemctl status chronyd.service
~~~



#### 客户端

所有的内网客户端都向集群中的一个时间服务器同步时间。每个客户端机器都安装 `chrony` 然后修改配置文件 `/etc/chrony.conf` ，如下配置（记得修改服务端ip）

~~~bash
cat > /etc/chrony.conf << EOF
server <服务端的ip地址或可解析的主机名> iburst
driftfile /var/lib/chrony/drift
makestep 10 3
rtcsync
local stratum 10
keyfile /etc/chrony.key
logdir /var/log/chrony
stratumweight 0.05
noclientlog
logchange 0.5
EOF
~~~

重启服务，然后验证

~~~bash
# 启动chronyd
systemctl restart chronyd.service
systemctl enable chronyd.service
systemctl status chronyd.service
 
# 验证, 出现 ^* 后面的 ip 就是正在使用的同步时间的ip
chronyc sources -v 
~~~





## 9. 系统内核优化

使用 `ulimit` 命令对系统作内核资源进行控制。



#### 查看和设置用户级能打开的最大进程数

~~~bash
# 查看
ulimit -u

# 设置
ulimit -u 102400
~~~

>回忆，每个用户登陆系统时就会执行一个命令 `bin/bash`，配置在 `/etc/passwd` 中的默认登陆命令。



#### 查看和设置用户级能分配使用的文件描述符

~~~bash
# 查看
ulimit -n

# 设置 
ulimit -n 1048576
~~~

>**文件描述符**：进程每打开一个文件，操作系统内核就会为该打开的文件分配一个编号(非负整数)，该编号称之为文件描述符（File Descriptor），也有人将其译为文件句柄。



#### 永久设置

`ulimit` 命令的设置都是临时的，想要永久设置，需要把配置数据保存在文件中 `/etc/security/limits.conf`，这个文件可以用之用户最多打开的进程数量和文件数量。



#### 调整 pid 的数量

~~~bash
# 临时设置
echo 4194303 > /proc/sys/kernel/pid_max

# 永久设置
echo "kernel.pid_max=4194303" | tee -a /etc/sysctl.conf
kernel.pid_max=4194303

# 查看
sysctl -p
kernel.pid_max = 4194303
~~~



#### 其他内核优化

| 参数                                     | 分类     | 默认值（通常）     | 建议值（示例）      | 核心作用与说明                                               |
| :--------------------------------------- | :------- | :----------------- | :------------------ | :----------------------------------------------------------- |
| **`net.ipv4.tcp_fin_timeout`**           | 连接管理 | 60                 | 2-30                | **控制 `FIN-WAIT-2` 状态超时时间**。降低此值可激进地回收资源，防止半关闭连接堆积。 |
| **`net.ipv4.tcp_tw_reuse`**              | 连接管理 | 0                  | 1                   | **允许安全地重用 `TIME-WAIT` 状态的端口用于新的出站连接**。解决主动关闭连接导致的端口耗尽问题。 |
| **`net.ipv4.tcp_max_tw_buckets`**        | 连接管理 | 依赖系统           | 2000000             | **系统允许的 `TIME-WAIT` 连接最大数量**。超过此值会直接销毁新产生的，防止内存耗尽。 |
| **`net.ipv4.tcp_max_syn_backlog`**       | 连接管理 | 1024               | 8192+               | **半连接队列（SYN_RECV）最大长度**。增大以应对高并发连接请求，需与 `somaxconn` 配合。 |
| **`net.core.somaxconn`**                 | 连接管理 | 128                | 4096+               | **每个端口的全连接队列（ESTABLISHED）最大长度**。增大以防止 `accept()` 队列溢出导致连接被拒绝。 |
| **`net.ipv4.tcp_syncookies`**            | 连接管理 | 1                  | 1                   | **启用 SYN Cookie**。用于抵御 SYN Flood 攻击，在队列满时仍可处理新连接。应保持开启。 |
| **`net.ipv4.ip_local_port_range`**       | 端口管理 | 32768 60999        | 1024 65000          | **定义出站连接可用的本地端口范围**。扩大范围以增加最大出站连接数。 |
| **`net.core.rmem_max` / `wmem_max`**     | 缓冲区   | 依赖系统           | 67108864 (64MB)     | **单个 socket 收/发缓冲区的最大硬限制**。为高性能应用提供更大的缓冲空间。 |
| **`net.ipv4.tcp_rmem` / `tcp_wmem`**     | 缓冲区   | 4096 87380 4194304 | 4096 87380 67108864 | **TCP socket 缓冲区的动态调整范围 (min default max)**。增大 max 值有益于高速高延迟网络。 |
| **`net.core.netdev_max_backlog`**        | 缓冲区   | 1000               | 100000+             | **网卡接收队列的最大包数**。增大可在流量爆发时减少网卡丢包。 |
| **`net.ipv4.tcp_slow_start_after_idle`** | 协议栈   | 1                  | 0                   | **禁用空闲后重启慢启动**。对长连接有益，避免短暂空闲后吞吐量骤降，保持高性能。 |
| **`net.ipv4.tcp_fastopen`**              | 协议栈   | 0                  | 3                   | **启用 TCP Fast Open (TFO)**。允许在握手完成前发送数据，减少一个 RTT 的延迟，需应用支持。 |
| **`net.ipv4.tcp_congestion_control`**    | 协议栈   | cubic              | bbr                 | **切换TCP拥塞控制算法**。`bbr` 在多数现代网络中能提供比默认 `cubic` 更低的延迟和更高的吞吐量。 |
| **`net.netfilter.nf_conntrack_max`**     | 连接追踪 | 65536              | 1048576 (1M)+       |                                                              |



## 10. OOM与内存优化

当 Centos7 系统内存快要用完的时候，操作系统会按下述顺序运行。

- 阶段1，清理 buffer/cache。
- 阶段2，启动 swapping。
- 阶段3，启动 OOM。



在宿主机上，配置内核参数 `swappiness`，该参数决定系统会有对频繁的使用交换分区。值越小表示尽可能少地使用 swap。当值为0，也不代表禁用swap。

~~~bash
[root@centos ~~]# cat /proc/sys/vm/swappiness 
60
[root@centos ~~]# echo "vm.swappiness = 10" | tee -a /etc/sysctl.conf
~~~





## 11. 网络优化

###  关闭 NetworkManager

在 centos 上直接使用 network 服务即可。如果两种都配置会引起冲突。

~~~bash
# 临时关闭
systemctl stop NetworManager

# 永久关闭
systemctl disable NetworManager
~~~



### 提高 MTU 值

万兆网卡MTU=9000，交换机也要设置，对端的MTU也要设置为9000。如果使用默认值太浪费了。



### 网卡绑定

两个网卡绑在在一块是用，减少网卡单点故障。



### 禁用 ping

~~~bash
echo 1 > /proc/sys/net/ipv4/icmp_echo_ignore_all
~~~

