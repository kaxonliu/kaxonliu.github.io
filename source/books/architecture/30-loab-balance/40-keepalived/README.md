# keepalived 高可用

keepalived 是集群管理中保证集群高可用的一个服务软件，用来防止单点故障，实现高可用。

keepalived 的作用是检测服务器的状态，如果有一台服务器死机或工作出现故障，Keepalived 将检测到，并将有故障的服务器从系统中剔除，当服务器工作正常后 Keepalived 自动将服务器加入到服务器群中，这些工作全部自动完成，不需要人工干涉，需要人工做的只是修复故障的 web 服务器。keepalived 管理的机器组会对外提供一个 vip，组内的一台机器挂掉后，vip 自动飘移到另外一台机器，确保继续提供服务。

~~~bash
# 具体使用场景
1、lvs负载均衡+keepalived
2、nginx负载均衡+keepalived
3、haproxy负载均衡+keepalived
~~~



## Keepalived 原理

Keepalived 的高可用是基于 VRRP 协议来实现的，VRRP全称Virtual Router Redundancy Protocol，即虚拟路由冗余协议。使用 VRRP 协议管理服务器，会把多个服务器组成一个组，也就是虽然它名字叫虚拟路由冗余协议，但并非一定要管理路由器，你用它管理服务器当然也可以。这个组里面有一个master 和多个 backup，master 上面有一个对外提供服务的  VIP (Virtual IP Address)。master 会发组播，当 backup 收不到 VRRP 包时就认为 master 宕掉了，这时就需要根据 VRRP 的优先级来选举一个 backup 当 master。VIP 会自动漂移过去。这样的话就可以保证路由器的高可用了。

#### 注意：
- VRRP 依赖局域网 LAN 多播或广播，所以要求被 keepalived 管理的机器必须在同一个局域网内。
- VRRP 通告消息直接封装在 IP 数据包中发送，因此 VRRP 数据包本质上就是普通的 IP 数据包，所以说 VRRP 协议不需要特殊的网络设备来解析。
- keepalived 使用非抢占式的方式选举新主，旧主修复后不会抢占 VIP，只会做一个从机器。整个过程发生一次切换 master 的行为。



#### Keepalived三个主要模块
- core 模块为 keepalived 的核心，负责主进程的启动、维护以及全局配置文件的加载和解析。
- check 负责健康检查，包括常见的各种检查方式。
- vrrp 模块是来实现 VRRP 协议的。



**部署 keepalived 服务的相关节点之间不需要专门的心跳线物理直连**。Keepalived 的心跳通信是发生在网络 IP 层面发送接收 VRRP 协议包，也就是说节点之间的VRRP通信基于现有的网络环境通信就行，并不需要连特定的物理连线。在keepalived中，相关节点都是连接到一个网络交换机上，它们通过该交换机交换 VRRP 的心跳包，这是一种网络层面的"心跳线"，而非物理层面的直连线 物理层面的直连线做心跳线确实也有，但通常都是物理链接两台交换机的，而非 keealived 集群节点。



在阿里云上自己部署 keepalived 用到的 VIP 需要申请 HaVip，然后关联 EIP。[详见链接](https://help.aliyun.com/zh/vpc/highly-available-virtual-ip-address-havip)



## Keepalived 安装

### yum 安装

~~~bash
 yum install keepalived -y
~~~

### 源码安装

~~~bash
# 下载地址：
https://www.keepalived.org/download.html
 
# 解压
tar -zxvf keepalived-2.2.8.tar.gz
cd  keepalived-2.2.8
./configure --prefix=/usr/local/keepalived --sysconf=/etc
make && make install
 
 
# 解释：
--sysconf：keepalived核心配置文件所在位置，固定位置，改成其他位置则keepalived启动不了，/var/log/messages中会报错。
如果报错：the build will not support IPVS with IPv6.Please install libnl/libnl-3 dev xxxxxxxxxx
解决方案：
安装 libnl/libnl-3 依赖 yum -y install libnl libnl-devel libnl3* libnl3-devel*，重新configure一下就好了
~~~



