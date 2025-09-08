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



## Keepalived 用在哪

Keepalived 可以应用在七层 四层，可以配合多种软件。但核心是**用在存在单点故障的位置**，解决单点故障解决高可用。



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



## 七层负载均衡器部署keepalived

集群架构：三台 web 服务器，三台七层负载均衡器。七层负载均衡器之间使用 keepalived 实现高可用避免单点故障。整个集群对外暴露一个 VIP 供用户使用。

### 主机规划

~~~bash
VIP 10.10.98.99

# 七层负载器
10.10.98.150:8081
10.10.98.151:8081
10.10.98.152:8081

# web服务器
10.10.98.200:8080
10.10.98.201:8080
10.10.98.203:8080
~~~



### 基础环境配置

- 关防火墙，关 selinux
- 配置静态 ip
- 同步时间



### 部署 web 服务器

三台 web 服务器使用 nginx 对外提供一个简单页面。在三个 web 服务器上部署 nginx，使用相同的 nginx 配置

~~~nginx
http {
  server {
    listen 8080;
    root usr/share/nginx/html;
  }
}
~~~



### 部署七层负载器

三台 七层负载器使用 nginx 代理，三个的配置全部一样。

~~~nginx
http {
  upstream webs {
    server 10.10.98.200:8080 weight=1;
    server 10.10.98.201:8080 weight=1;
    server 10.10.98.203:8080 weight=1;
  }
  
  server {
    listen 8081;
    location / {
      proxy_pass http://webs;
    }
  }
}
~~~



### 七层负载上部署 keepalived 服务

#### 1. 安装 keepalived

三台七层负载器上安装 keepalived

~~~bash
yum install -y keepalived
~~~

#### 2. 配置 keepalived

keepalived 配置一个组，配置文件 `/etc/keepalived/keepalived.conf`。需要注意每个七层负载机器的网卡名称和优先级需要对应修改。其他参数都保持一样即可。

~~~ini
global_defs {
   script_user root 
   enable_script_security
}
 
vrrp_script chk_nginx {
    # 1、定时检测被高可用的服务的存活状态
    script "/etc/keepalived/check_port.sh"
    # 2、每隔3s运行一次上面的脚本
    interval 3
    # 3、如果脚本运行失败，则降低权重-20，配置下面的设置的priority值基础上减
    # 一旦priority发生变化，基于下述advert_int的设置1s内vrrp就能检测到然后完成主备切换
    weight -20
}
 
vrrp_instance VI_1 {
    # 1、表示状态是MASTER主机还是备用机BACKUP，初始都设置为BACKUP，然后通过priority来高低来决定谁是主
    state BACKUP
    # 2、一定要与你的网卡对应上，否则错误
    interface ens160
    # 3、所有主备节点保持一致，代表在一个路由组里
    virtual_router_id 251
    # 4、当前节点的优先级，数字越大，优先级越高，节点优先级，值范围0~254，MASTER>BACKUP
    priority 100
    # 5、设定VRRP设备发送广播消息的时间间隔为1s一次，以确定VRRP群组成员的状态同步，确保主挂掉后能够快速进行主备切换
    advert_int 1
    # 6、非抢占式，详解在下面
    nopreempt
 
    # 7、认证权限密码，防止非法节点进入
    authentication {
        auth_type PASS
        auth_pass 11111111
    }
    # 8、健康检查脚本，在这个脚本里写检测逻辑，某个服务挂掉了，就停掉keepalived重新选出主，vip会飘过去
    track_script {
         chk_nginx
    }
    # 9、虚拟出来的ip，可以有多个（vip）
    virtual_ipaddress {
        10.10.98.99
    }
} 
~~~

> 补充：通常如果master服务死掉后backup会变成master，但是当master服务又好了的时候 master此时会抢占VIP，这样就会发生两次切换对业务繁忙的网站来说是不好的。所以我们要在配置文件加入 nopreempt 非抢占，但是这个参数只能用于state 为backup，故我们在用HA的时候最好master 和backup的state都设置成backup 让其通过priority来竞争。



#### 3. 编写检测脚本

三个负载器上完全一样。

~~~bash
touch /etc/keepalived/check_port.sh
chmod +x /etc/keepalived/check_port.sh
 
# 强调
# 强调
# 强调：你脚本的运行时间超过了配置文件中的interval时间，就会被kill -15杀掉
vi /etc/keepalived/check_port.sh 
 
#!/bin/bash
count=$(ps -C nginx --no-header|wc -l)
#1.判断 Nginx 是否存活,如果不存活则尝试启动 Nginx
if [ $count -eq 0 ];then
    systemctl start nginx &>/dev/null &  # 放后台运行防止超时
    sleep 1  # 不要sleep太久，超过了keepalived.conf中配置的interval时间就麻烦了
    #2.等待 1 秒后再次获取一次 haproxy 状态# 
    count=$(ps -C  nginx --no-header|wc -l)
    #3.再次进行判断, 如haproxy 还不存活则停止 Keepalived,让地址进行漂移,并退出脚本
    if [ $count -eq 0 ];then
        systemctl stop keepalived
    fi
fi
~~~

#### 4. 启动 keepalived 服务

三个服务依次

~~~bash
systemctl enable keepalived
systemctl start keepalived
systemctl status keepalived
~~~

启动后可以看到 VIP 放在了负载器1上面

~~~bash
[root@rocky ~]# ip a
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host 
       valid_lft forever preferred_lft forever
2: ens160: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc fq_codel state UP group default qlen 1000
    link/ether 00:50:56:25:e8:8b brd ff:ff:ff:ff:ff:ff
    altname enp2s0
    inet 10.10.98.150/21 brd 10.10.103.255 scope global noprefixroute ens160
       valid_lft forever preferred_lft forever
    inet 10.10.98.99/32 scope global ens160
       valid_lft forever preferred_lft forever
    inet6 fe80::250:56ff:fe25:e88b/64 scope link noprefixroute 
       valid_lft forever preferred_lft forever
~~~



## keepalived双主热备高可用

上面的结构存在一个现象：三个负载均衡器同一时间只有一个在使用，另外两个闲置。为了提高负载器的使用率，避免闲置发生，可以换一种方式使用 keepalived。比如配置两个路由组，在路由组1中，负载器1是主，其余两个是备份；在路由组2中，负载器2是主，其余两个是备份。这样需要有2个 VIP 。这样的话，如果只有2个七层负载均衡器，可以采用双主热备的架构方案。两个负载器互为主从。

### 主机规划

~~~bash
VIP 10.10.98.99
VIP 10.10.98.199

# 七层负载器
10.10.98.150:8081
10.10.98.151:8081

# web服务器
10.10.98.200:8080
10.10.98.201:8080
10.10.98.203:8080
~~~

### 编写 keepalived 配置文件

配置文件中定义两个路由组，有两个 VIP。

- 路由组1：路由组编号51。负载器1为主，负载器2为从。使用 VIP 10.10.98.99
- 路由组2：路由组编号52。负载器1为从，负载器2为主。使用 VIP 10.10.98.199

~~~bash
# cat /etc/keepalived/keepalived.conf 

global_defs {
   script_user root 
   enable_script_security
}
 
vrrp_script chk_nginx {
    script "/etc/keepalived/check_port.sh"
    interval 2
    weight -20
}

# 路由组1
vrrp_instance VI_1 {
    state MASTER
    interface ens160
    virtual_router_id 51
    priority 100
    advert_int 1
    nopreempt
    authentication {
        auth_type PASS
        auth_pass 1111
    }
    track_script {
         chk_nginx
    }
    virtual_ipaddress {
        10.10.98.99
    }
}

# 路由组2
vrrp_instance VI_2 {
    state BACKUP
    interface ens160
    virtual_router_id 52
    priority 80
    advert_int 1
    nopreempt
    track_script {
         chk_nginx
    }
    authentication {
        auth_type PASS
        auth_pass 1111
    }
    virtual_ipaddress {
        10.10.98.199
    }
}
~~~

### 注意上述配置文件修改

- 在负载器1上面，直接上述文件。
- 在负载器2上面，做如下修改。

~~~bash
vrrp_instance VI_1 {
    state BACKUP
    priority 97
}
vrrp_instance VI_2 {
    state MATER
    priority 100
}
~~~



### 启动服务然后测试

编辑配置文件后重启 keepalived 服务，使用 `ip -a` 可以看到两个 VIP 分别挂在不同的负载器上。把其中一个负载器上的 keepalived 关闭，VIP 会飘到另一个备份机上。开启 keepalived 服务后会再次加入路由组分担两个 VIP 的负载均衡工作。





## DNS 轮询

DNS 轮询是一种**简单的负载均衡技术**。它的工作原理是，**为一个域名配置多个 A 记录（即多个 IP 地址）**。当客户端向 DNS 服务器发起域名解析请求时，DNS 服务器会按照预设的顺序（通常是轮转的方式）返回这一组 IP 地址中的某一个。目的：将客户端的访问请求分散到多个服务器上，从而减轻单台服务器的压力。

优点：简单易行，成本低廉（需要多个 公网 ip），一定程度上分散流量。

缺点：无法保证负载均衡，缺乏故障转移机制，转发不可控。

使用：一般是配合集群架构辅助负载均衡。



## 脑裂问题

涉及到 VIP 故障飘逸的集群都可能存在脑裂问题，当然了 keepalived 服务也可能出现。出现的原因重要是服务之间的通信出现了障碍。MASTER 没有挂，但是和从节点之间的网络通信出现故障，从节点认为主挂掉了，于是就选出了一个新的主，出现了一个 VIP 同时在多个负载器上。原本这个系统只有一个入口，现在分裂为两个独立的入口，开始争抢资源，系统混乱，可能出现数据不一致的问题。



### 脑裂出现的原因
1、服务器上开启了防火墙阻挡了心跳消息传输。
2、网络环境出现软硬件故障，导致vrrp心跳包无法正常通信
3.Keepalived配置里同一 VRRP实例如果virtual_router_id两端参数配置不一致也会导致裂脑问题发生。

总结：**全都是两个节点彼此之间的通信出现了问题**



### 解决方案

~~~text
1、当下已经遇到了脑裂
应急处理，恢复业务摆在第一位，先kill一台，保证集群正常
 
2、预防脑裂
（1）做好对裂脑的监控报警（如邮件及手机短信等或值班）.在问题发生时人为第一时间介入仲裁，最大程度降低损失
例如将报警消息发送到管理员手机上，管理员可以通过手机回复对应数字或简单的字符串操作返回给服务器.
让服务器根据指令自动处理相应故障，这样解决故障的时间更短.
（2）一种更彻底的方法是使用第三方检测机制，例如 STONITH（Shoot The Other Node In The Head），也称为"击败"。STONITH 方法需要硬件支持，通过硬件来关闭或重启认为自己是主节点的其他节点，确保任何时候都只有一个节点在运行。
最后，这是一个复杂的主题，需要根据你的特定环境和需求来决定最佳的策略。一般来说，建议通过专门的网络管理和监视工具来监控你的网络状态，及时发现并解决任何可能导致脑裂的问题。
（3）引入分布式协调服务如ZooKeeper等服务器，用于节点间的健康检查和领导选举。
~~~



### 监测脑裂出现的简单方法

使用 `arping` 命令，一个 VIP 如果出现出现多个 mac 地址，则存在脑裂问题。

~~~bash
[root@rocky ~]# arping -I ens160 10.10.98.99 -c 1
ARPING 10.10.98.99 from 10.10.98.200 ens160
Unicast reply from 10.10.98.99 [00:50:56:3C:6C:67]  1.706ms
Unicast reply from 10.10.98.99 [00:50:56:25:E8:8B]  1.923ms
Sent 1 probes (1 broadcast(s))
Received 2 response(s)
~~~



