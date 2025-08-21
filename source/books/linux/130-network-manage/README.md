# 网络管理

## 查看网卡信息

#### 1. 使用 ip 命令查看

~~~bash
ip address
~~~

#### 2. 使用 ifconfig 命令查看

需要安装 `net-tools` 才能使用 `ifconfig` 命令

~~~bash
# 安装 
yum install -y net-tools

# 查看网卡信息
ifconfig -a 		# 查看所有网卡信息，包括未激活的
ifconfig ens161	# 查看指定网卡信息
~~~

#### 3. 使用 lspci 命令查看所有以太网卡信息

需要安装 `pciutils` 工具包才能使用 `lspci` 命令。

~~~bash
lspci | grep -i ethernet
~~~

#### 4. 使用 mii-tool 命令查看网卡网线连接情况

需要安装 `net-tools` 工具包才能使用 `mii-tool` 命令。

~~~bash
[root@centos ~]# mii-tool ens161
ens161: negotiated 1000baseT-FD flow-control, link ok
~~~



##  ifconfig 查看的信息

~~~bash
[root@centos ~]# ifconfig ens161
ens161: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1500
				# 接口已启用，在使用中。支持广播、组播，MTU:1500（最大传输单元）：1500字节
        
        inet 192.168.10.16  netmask 255.255.255.0  broadcast 192.168.10.255
        # PV4地址：192.168.10.16 
        # 子网掩码：255.255.255.0
        # 广播地址： 192.168.10.255
        
        inet6 2409:8a20:4d49:6d04:136b:d2c2:a821:31f6  prefixlen 64  scopeid 0x0<global>
        inet6 fe80::a2f8:87e:a7b:9896  prefixlen 64  scopeid 0x20<link>
        ether 00:0c:29:79:72:0a  txqueuelen 1000  (Ethernet)
        # 以太网卡mac地址：ether 00:0c:29:79:72:0a
        # 网卡传输队列长度：txqueuelen 1000
        
        RX packets 2806  bytes 686679 (670.5 KiB)
     		# 表示开机后此接口累积接收的报文个数和总字节数：RX packets 2806  bytes 686679 (670.5 KiB)
     		
        RX errors 0  dropped 0  overruns 0  frame 0
        # 此接口累积接收报文错误数，丢弃数，溢出数（由于速度过快而丢失的数据包数），冲突的帧数
        
        TX packets 1039  bytes 155891 (152.2 KiB)
        # 表示开机后此接口累积发送的报文个数，总字节数
        
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0
        # 此接口累积发送报文错误数，丢弃数，溢出数（由于速度过快而丢失的数据包数），
        # carrier 载荷数(发生carrier错误而丢失的数据包数)
        # collisions 冲突数
        device interrupt 48  memory 0x3fd00000-3fd20000
~~~



## 网卡CRC校验

CRC，全称循环冗余校验（Cyclic Redundancy Check），是一种常用的校验方式，用于发现网络通信中传输数据的错误。网卡接收数据的每一帧数据都包含一个CRC值。这个CRC值是基于数据帧中的数据计算得来的。发送数据的设备，如网卡，会在发送数据的时候计算CRC，并将CRC值附加在数据帧的尾部。接收数据的设备则会重新计算接收到的数据帧的CRC值，然后与接收到的CRC值进行比较。如果两个CRC值不相等，则表示数据在传输过程中发生了错误（数据在传输中被篡改）。



## 网卡工作流程

**网卡发包流程**：

1. ip包+14个字节的 mac 头变成数据帧 frame。
2. frame 拷贝到网卡芯片内部的缓冲区，由网卡处理。
3. 网卡芯片为 frame 添加头部同步信息和CRC校验，此时才是真正可以发送的packet。

**网卡收包流程**：

1. 网络包 packet 到达网卡，网卡先检查包 packet 的 CRC 校验，保证其完整性和正确性，然后去掉它的头得到frame。
2. 网卡将 frame 拷贝到网卡内部的 FIFO 缓冲区。
3. 网卡驱动程序产生硬件中断，把 frame 从网卡拷贝到内存中，接下来就交给内核处理。

**网卡丢包现象**

内核通常需要快速的拷贝网络数据包到系统内存！！！因为网卡上接收网络数据包的缓存大小固定，而且相比系统内存也要小得多。所以上述拷贝动作一旦被延迟，必然造成网卡 FIFO 缓存溢出 - 进入的数据包占满了网卡的缓存，后续的包只能被丢弃，这也应该就是 `ifconfig` 里的 `overrun` 的来源。



## 网卡丢包问题解决

#### 1.先看硬件

查看工作模式是否正常，使用 `ethtool` 命令查看网卡的速度和是否为全双工模式

~~~bash
# 安装
yum install -y ethtool

# 查看
[root@centos ~]# ethtool ens161 | egrep 'Speed|Duplex'
	Speed: 1000Mb/s
	Duplex: Full
~~~

#### 2. 查看 CRC 校验是否正常

CRC 值大通常是因为服务区外部的网络环境有问题导致的。如果 Speed, Duplex,CRC都没问题，基本排除物理层面的问题。

~~~bash
[root@centos ~]# ethtool -S ens161 | grep crc
     rx_crc_errors: 0
~~~



#### 3. 通过 ifconfig 查看 overruns

观察 overruns 指标是否一直增大

~~~bash
watch -n 1 'ifconfig ens161 | grep RX | grep overruns'
~~~



#### 4. 调整网卡缓冲区

使用 `ethtool -g` 查看 网卡的缓冲区设置情况，使用 `ethtool -G` 设置缓冲区参数。

~~~bash
# 查看当前的设置情况
[root@centos ~]# ethtool -g ens161
Ring parameters for ens161:
Pre-set maximums:
RX:		4096
RX Mini:	0
RX Jumbo:	0
TX:		4096
Current hardware settings:
RX:		256
RX Mini:	0
RX Jumbo:	0
TX:		256

# 调大缓冲区
[root@centos ~]# ethtool -G ens161 rx 2048	# 调大收包
[root@centos ~]# ethtool -G ens161 tx 2048  # 调大发包
~~~



## ping 命令

测试两台主机网络是否相同使用 `ping` 命令。使用该命令需要提前安装 `iputils` 工具包。

~~~bash
# 安装工具包
yum install -y iputils

# 测试对方主机ip
ping -c <次数> <目标IP地址>

 
# 在自己的机器上执行如下命令，禁用别人ping自己
echo 1 > /proc/sys/net/ipv4/icmp_echo_ignore_all 
~~~



## network 服务

在 centos7.9 上网卡信息的配置需要编辑 `/etc/sysconfig/network-scripts/` 目录下的文件 `ifcfg-ens*`

~~~bash
# 配置网卡类型
TYPE=Ethernet

# 名字和设备
NAME=ens192
DEVICE=ens192

# 配置ip获取方式
# dchp  动态获取IP
# static  手工指定IP
# none  根据其他选项决定动态还是静态
BOOTPROTO=static

# 配置服务启动是否激活状态 yes or no
ONBOOT=yes 

# 该文件是否被 如果NetworkManager服务管理
NM_CONTROLLED=no    

# 配置 IP地址 子网掩码 默认网关 NDS服务（可以有多个）
IPADDR=10.1.1.11 　　　　 
NETMASK=255.255.255.0   
GATEWAY=10.1.1.1         
DNS1=10.1.1.1           
DNS2=8.8.8.8         

# MAC地址
HWADDR=14:da:e9:eb:a9:61
# uuid
UUID=4b0737cc-f19c-4c56-a33a-a23dd940a9b5

# 是否允许普通用户启动或者停止该网卡
USERCTL=no 　

# 是否在该网卡上启动IPV6的功能
IPV6INIT=yes

# 是否允许网卡在启动时向DHCP服务器查询DNS信息
# 设置为yes时，此文件设置的DNS将覆盖/etc/resolv.conf，
# 若开启了DHCP，则默认为yes，所以dhcp的dns也会覆盖/etc/resolv.conf
PEERDNS=yes 　　
~~~

上面是这个文件常用的配置字段及其解释，配置保存后，重启 network 服务，即可得到新的网卡信息。

~~~bash
systemctl restart network
~~~



## NetworkManager 服务

在 centos7.9 虽然主流使用 network服务，但是也可以使用 NetworkManager 服务，并且它也是基于

`/etc/sysconfig/network-scripts/` 下面网卡配置文件的。这个运行时可以使用 `nmcli` 命令来修改网卡配置信息（通过命令修改，不需要手动编辑文件了），NetworkManager 服务关闭时不可以使用 `nmcli` 命令。

~~~bash
nmcli con mod ens33 ipv4.addresses "192.168.1.100/24"
nmcli con mod ens33 ipv4.gateway "192.168.1.1"
nmcli con mod ens33 ipv4.dns "8.8.8.8,8.8.4.4"  # 多个dns
nmcli con mod ens33 ipv4.method manual
 
nmcli con down ens33
nmcli con up ens33
~~~

