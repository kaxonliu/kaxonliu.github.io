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



## Centos7.9配置IP地址

在centos7.9 上，想要永久的保存网卡的配置信息需要把配置数据写到配置文件中（配置文件是 `/etc/sysconfig/network-scripts/` 目录下面 `ifcfg-ens` 文件，一个网卡一个 `ifcfg`文件）。管理网卡的服务有两个，分别是：`network` 和 `NetworkManager`，前者是主流的使用方式，但后者的使用也非常方便。前者的使用特征是：手动编辑网卡配置文件  `ifcfg` ，然后重启 `network` 服务。后者的使用特征是：一个命令 `nmcli` 搞定配置文件 `ifcfg` 和重启服务。



### 使用 network 服务配置IP地址

#### 1. 编辑网卡配置文件 ifcfg

~~~bash
TYPE=Ethernet
BOOTPROTO=static
NAME=ens160
UUID=dad59d0b-0ff6-4db8-94a0-5dd4e22fe011
DEVICE=ens160
ONBOOT=yes
IPADDR=10.10.98.66
PREFIX=21
GATEWAY=10.1.1.1 
DNS1=114.114.114.114
DNS2=8.8.8.8
~~~

上面是网卡配置文件 `ifcfg-ens160` 常用的配置字段，其中：

-  `TYPE=Ethernet` 表示网卡类型为以太网网卡；
- `BOOTPROTO` 指定获取 IP 的方式，值为 `static` 表示静态 IP，值为 `dhcp` 表示动态获取，值为 `none` 默认也是静态IP的方式。
- `DEVICE` 网卡的设备名。
- `NAME` 指定网卡的名字（建议和 `DEVICE` 保持一致），使用 `nmcli`命令会用到这个(connection name)。
- `ONBOOT` 配置重启时时候自动激活网卡。`yes` 表示自动激活，`no` 不自动激活。
- `IPADDR` 指定 IPV4 地址。
- `GATEWAY` 指定网关地址。
- `PREFIX` 指定子网掩码，使用位数表示。想要配置IP形式的子网掩码使用 `NETMASK=255.255.255.0`
- `DNS1`/`DNS2` 指定 DNS 的地址，多个时通过尾部编号表示。



#### 2. 重启 network 服务

~~~bash
systemctl restart network
~~~



#### 3. 查看配置是否生效

~~~bash
# 查看 ip 地址
ifconfig

# 查看 dns
cat /etc/resolv.conf
~~~



#### 4. 补充其他配置字段

~~~bash
# 该文件是否被 如果NetworkManager服务管理
# 配置该字段值为no, 重启 network服务; cmcli conn show 查不到该网卡信息
NM_CONTROLLED=no     

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



### 使用 NetworkManager 服务配置IP地址

在 Centos7.9 上可以使用 `NetworkManager` 服务管理网卡，具体管理使用命令 `nmcli`。当 `NetworkManager` 启用时可以使用，当这个服务停掉后，就无法使用 `nmcli` 命令。

在 Centos7.9 上如果希望某块网卡不要被 `NetworkManager` 管理， 可以在配置文件 `ifcfg` 中明确设置**`NM_CONTROLLED=no`**，那么这个网卡接口就不会由 `NetworkManager` 来管理。`NetworkManager` 会忽略这个配置文件，因此使用命令 `nmcli conn show` 也就不会显示它。

在 Centos 7.9 中，`nmcli` 是**强大且首选**的网络配置工具。它与传统的编辑 `/etc/sysconfig/network-scripts/ifcfg-*` 文件的方式是**等效且同步的**——当你用 `nmcli` 修改配置后，它会自动写入对应的 `ifcfg` 文件，反之亦然。使用 `nmcli` 更不容易产生语法错误，并且命令可脚本化。

#### 1. 启用 NetworkManager 服务

~~~bash
systemctl enable NetworkManager
systemctl start NetworkManager
systemctl status NetworkManager
~~~

#### 2. 查看网络连接

连接是配置文件的逻辑概念，它可以被应用到设备上。

~~~bash
nmcli connection show
# 或者简写
nmcli con show

# 显示更详细的信息
nmcli con show --active
~~~

#### 3. 查看所有网络设备

设备是物理或虚拟的网络接口。

~~~bash
nmcli device status
# 或者简写
nmcli dev status

# 查看指定设备的详细信息（如 ens160）
[root@centos ~]# nmcli dev show ens160
GENERAL.DEVICE:                         ens160
GENERAL.TYPE:                           ethernet
GENERAL.HWADDR:                         00:10:56:3B:6B:42
GENERAL.MTU:                            1500
GENERAL.STATE:                          100（已连接）
GENERAL.CONNECTION:                     ens160
GENERAL.CON-PATH:                       /org/freedesktop/NetworkManager/ActiveConnection/8
WIRED-PROPERTIES.CARRIER:               开
IP4.ADDRESS[1]:                         10.10.98.66/21
IP4.GATEWAY:                            10.10.100.254
IP4.ROUTE[1]:                           dst = 10.10.96.0/21, nh = 0.0.0.0, mt = 102
IP4.ROUTE[2]:                           dst = 0.0.0.0/0, nh = 10.10.100.254, mt = 102
IP4.DNS[1]:                             114.114.114.114
IP4.DNS[2]:                             8.8.8.8
IP6.ADDRESS[1]:                         fe80::250:56ff:fe3b:6b42/64
IP6.GATEWAY:                            --
IP6.ROUTE[1]:                           dst = fe80::/64, nh = ::, mt = 102
~~~

#### 4. 配置静态IP

新建连接 ens256，管理网卡设备 ens256，使用手动（静态ip）的方式，配置ip地址/子网掩码/网关/DNS等信息。命令执行后会新建一个名为 `ifcfg-ens256` 的网卡配置文件。

~~~bash
nmcli conn add \
			type ethernet \
			con-name "ens256" \
			ifname ens256 \
			ipv4.addresses 192.168.1.100/24 \
			ipv4.gateway 192.168.1.1 \
			ipv4.dns "8.8.8.8,1.1.1.1" \
			ipv4.method manual
~~~

**参数解释：**
- `con add`: 添加一个新连接。
- `type ethernet`: 类型为以太网。
- `con-name "my-static-ip"`: 连接名称。
- `ifname ens192`: 应用的物理设备名。
- `ipv4.addresses 192.168.1.100/24`: 静态 IP 地址和子网掩码。
- `ipv4.gateway 192.168.1.1`: 默认网关。
- `ipv4.dns "8.8.8.8,1.1.1.1"`: DNS 服务器，用逗号分隔。
- `ipv4.method manual`: 使用手动配置（静态IP）。

#### 5. 配置动态IP

配置动态IP，使用 `ipv4.method auto`。命令执行后会新建一个名为 `ifcfg-ens256` 的网卡配置文件。

~~~bash
nmcli con add type ethernet con-name "ens256" ifname ens256 ipv4.method auto
~~~

#### 6. 激活链接

~~~bash
nmcli conn up ens256
~~~

#### 7. 修改现有连接

已经存在的连接使用，可以修改它的配置数据。

~~~bash
# 修改 IP 地址
nmcli con mod "ens256" ipv4.addresses 192.168.1.150/24

# 添加一个额外的 DNS 服务器
nmcli con mod "ens256" +ipv4.dns 192.168.1.53

# 修改网关
nmcli con mod "ens256" ipv4.gateway 192.168.1.254

# 完全更改方法为 DHCP（会清除静态IP配置）
nmcli con mod "ens256" ipv4.method auto

# 再重新应用连接以使更改生效
nmcli con down "ens256" &&nmcli con up "ens256"
~~~

#### 8. 连接管理

~~~bash
# 启用/禁用连接
nmcli con up "ens256"

# 禁用一个连接（会断开设备）
nmcli con down "ens256"

# 删除连接  !!!删除连接不会删除物理设备，只是删除了配置文件
nmcli con del "ens256"
~~~





## RockyLinux9.6配置IP地址

在RockyLinux9.6上已经不再使用 network 服务管理网卡接口了，只使用 NetworkManager 服务。配置文件也从 Centos7时代的 `/etc/sysconfig/network-scripts/ifcfg-ens*` 转为 `/etc/NetworkManager/system-connections/ens*.nmconnection`。

在 RockyLinux9.6 上使用 `nmcli` 命令管理网卡，用法和在Centos7.9 上完全一样。下面是网卡 ens160 的配置文件信息（不用手动编辑，nmcli 命令会直接修改它）。

~~~bash
[root@rocky system-connections]# cat ens160.nmconnection
[connection]
id=ens160
uuid=cece4e39-4c36-30cb-82c2-88d8a2e035bf
type=ethernet
autoconnect-priority=-999
interface-name=ens160
timestamp=1755864492

[ethernet]

[ipv4]
method=auto

[ipv6]
addr-gen-mode=eui64
method=auto

[proxy]
~~~



## Ubuntu24.04.3 LTS 配置IP地址

在 Ubuntu24中，**`systemd-networkd`** 是管理网络的首选方式之一，它是一个强大的系统级网络管理守护进程，它由 `systemd` 提供。它使用纯文本的 `.network`、`.netdev` 和 `.link` 文件进行配置，这些文件位于 `/etc/systemd/network/` 和 `/lib/systemd/network/` 目录。但是，除非你有特殊需求直接操作 `systemd-networkd`，否则通常**更推荐使用 Netplan** 来管理配置，因为它是发行版推荐的标准方式，能更好地处理与其他系统组件的集成。

Netplan 本身不配置网络，它只是一个前端，根据其 YAML 配置文件（在 `/etc/netplan/` 目录下）生成后端的配置（systemd-networkd 的配置文件）。



#### 1. 检查服务状态

需要检查两个服务的状态：`systemd-networkd`  和 `systemd-resolved`。通常它俩配合工作，后者负责 DNS 解析。

~~~bash
sudo systemctl status systemd-networkd.service
sudo systemctl status systemd-resolved.service
~~~



#### 2. 使用 netplan 配置ip

使用命令 `netplan` 编辑目录 `/etc/netplan/` 下以 `yaml` 结尾的文件。编辑前先备份。配置静态ip需要把 `dhcp4` 的值修改为 `false`，同时提供 IP地址/网关/dns等信息。动态ip只需要把 `dhcp4` 的值修改为 `true` 即可。

~~~bash
# 备份
cp 50-cloud-init.yaml 50-cloud-init.yaml.bak

# 编辑文件50-cloud-init.yaml, 配置ens160网卡设备使用静态ip
network:
  version: 2
  ethernets:
    ens160:
      dhcp4: false
      addresses: [10.10.97.41/21]
      routes:
        - to: default
          via: 10.10.100.254
      nameservers:
        addresses: [8.8.8.8,114.114.114.114]
~~~



#### 3. 应用配置

执行 `netplan apply` 命令，Netplan 生成 `systemd-networkd` 配置，通知 `systemd-networkd` 重新加载。



#### 4. 验证 ip地址配置

~~~bash
ip address show ens160

# 使用 ifconfig 需要安装包 net-tools
apt install net-tools
ifconfig ens160
~~~



#### 5. 验证 dns 配置

在 Ubuntu 24.04 中，`/etc/resolv.conf` 通常是一个指向 `/run/systemd/resolve/stub-resolv.conf` 的**软链接**。Ubuntu 默认使用 `systemd-resolved` 作为 DNS 解析管理器。它充当了一个本地 DNS 解析器的角色，并管理着所有网络接口的 DNS 配置。

`systemd-resolved` 会动态生成 `/run/systemd/resolve/stub-resolv.conf` 文件。这个文件的目的**不是**为了直接显示您配置的所有 DNS 服务器，而是为了将本地应用程序的 DNS 查询请求**转发**到 `systemd-resolved` 服务本身（通过 127.0.0.53），所以 `cat /etc/resolve.conf` 不会直接显示您配置的所有 DNS 服务器。

想要看具体 DNS 服务器，可以使用如下命令查看。

~~~bash
root@ubuntu:/etc/netplan# resolvectl status ens160
Link 2 (ens160)
    Current Scopes: DNS
         Protocols: +DefaultRoute -LLMNR -mDNS -DNSOverTLS DNSSEC=no/unsupported
       DNS Servers: 8.8.8.8 114.114.114.114
~~~



## 路由route

### 交换

交换指的是同网络访问。两台主机连在同一个交换机上，配置同网段的 IP 地址就可以直接通讯。

### 路由

不在一个网段内的主机，跨网访问需要使用路由。



### Linux 处理数据包的过程

数据包从网卡流入后需要对它做路由决策，根据其目标决定是流入本机的用户空间，还是在内核空间就直接转发给其他主机。

- **决策后如果发现是流入本机的数据**，那数据就会从内核流入本级用户空间，如果本机用户空间的应用程序不需要对此数据回包，就不再涉及从某个网卡流出数据；但如果需要回包，那在流出数据前也要做一遍路由决策，根据目标决定从那个网卡流出。
- **决策后发现不是流入本机用户空间的数据**，那只需要把数据包经由本机完整转发给其他主机。此时需要 Linux 主机有转发包的能力才行，但一般 Linux  主机默认都是未开启 `ip_forward` 功能，此时就会把数据包丢弃。

![](route.png)



### Linux 开启路由转发功能

**Linux 主机能转发数据包有两个条件：1本级被当作网关；2.本级开启了路由转发功能。**



#### 1. 临时开启

临时开启的方式，重启网络服务则失效。

~~~bash
# 方式1
echo 1 > /proc/sys/net/ipv4/ip_forward

# 方式2
sysctl -w net.ipv4.ip_forward=1
~~~

#### 2. 永久开启

永久开启，需要写入配置文件。

~~~bash
# 在CentOS 6中:
将/etc/sysctl.conf文件中的"net.ipv4.ip_forward"值改为1即可

 
#在CentOS 7中:
echo "net.ipv4.ip_forward=1" > /etc/sysctl.d/ip_forward.conf
~~~



#### 3. 查看是否开启转发功能

~~~bash
sysctl net.ipv4.ip_forward
cat /proc/sys/net/ipv4/ip_forward
sysctl -a | grep ip_forward
~~~





## 路由种类

在 Linux 上可以使用 `route` 命令查看本机的路由规则，这个命令需要安装 `net-tools` 才能使用。路由表中可以看到三种类型的路由。分别是：主机路由、网络路由、默认路由。

#### 1. 主机路由

主机路由指的就是转发给指定主机的路由规则，它的子网掩码是32位，因为精确到一个具体的主机，因此主机路由也被称为静态路由，它的优先级最高。

#### 2. 网络路由

网络路由指的是妆发给指定网络内的路由规则，它的子网掩码小于32位，优先级仅次于主机路由。

#### 3. 默认路由

不走主机路由的和网络路由的、全部都走默认路由。操作系统上设置的默认路由一般也称为网关。子网掩码通常为0。优先级最低。

#### 4. 优先级判断
- 主机范围越小，越精确则优先级越高。缩小主机的方式就是子网掩码，子网掩码长度越长则优先级越高。
- 如果路由规则的掩码长度相同，则比较节点之间的管理距离（metric），管理距离短的生效。



## route 命令管理路由

使用 `route` 命令查看和管理路由表。

#### 1. 查看路由表

使用 `route -n` 命令，`-n` 表示不解析主机名。

~~~bash
[root@centos ~]# route -n
Kernel IP routing table
Destination     Gateway         Genmask         Flags Metric Ref    Use Iface
0.0.0.0         192.168.10.1    0.0.0.0         UG    100    0        0 ens161
0.0.0.0         192.168.10.1    0.0.0.0         UG    101    0        0 ens192
192.168.10.0    0.0.0.0         255.255.255.0   U     100    0        0 ens161
192.168.10.0    0.0.0.0         255.255.255.0   U     101    0        0 ens192
192.168.10.100  0.0.0.0         255.255.255.255 UH    0      0        0 ens161

# Destination，目标网络或目标主机
# Gateway，到达目标需要经过的网关。如果没有网关则显示为0.0.0.0
# Genmask，和目标主机 ip配对使用的网络掩码
# Flags 参数，U表示路由up，该路由可用；H表示主机路由；G表示路由通向另一个网关
# Metric，衡量距离，一般由于选择使用小的。
# Use，路由被使用的次数
# Iface，路由使用的网络设备接口
~~~



#### 2. 增加和删除路由

使用 `route` 命令的参数 `add` 增加路由、参数 `del` 删除路由。

~~~bash
route [add/del] [-host/-net/default] [address[/mask]] [netmask] [gw] [dev]
 
选项说明：
add/del：增加或删除路由条目
-net：增加或删除的是一条网络路由
-host：增加或删除的是一条主机路由
default：增加或删除的是一条默认路由
netmask：明确使用netmask关键字指定掩码，要可以不使用该选项直接在地址上使用cidr格式的掩码，即IP/MASK。
gw：指定下一跳的地址。要求下一跳地址必须是能到达的，且一般是和本网段直连的接口。
dev：强制将路由条目关联到指定的接口上。一般内核会自动判断路由条目应该关联到哪个网络接口。
~~~



#### 增加路由示例

~~~bash
route add -host 192.168.10.111/32 dev ens161
route add -net 192.168.11.0/24 dev ens161
route add default gw 192.168.11.11 dev ens161
~~~

>注意：`route` 增加的路由都是临时，重启网络服务则失效。`systemctl restart network`



#### 删除路由示例

~~~bash
route del -host 192.168.10.111/32
route del -net 192.168.11.0/24
route del default gw 192.168.11.11
~~~



## Centos7.9 配置永久路由

路由配置文件在目录 `/etc/sysconfig/network-scripts/` 下面，比如配置网卡设备 ens161的路由规则，则新建一个名为 `route-ens161` 的文件。路由配置文件的配置格式很简单，一行配置一个路由规则，先是要达到的目标地址，然后是关键词 `via` 最后是下一跳的地址。要求下一跳必须能够达到。

#### 示例：新增 ens161的路由条目

~~~bash
touch /etc/sysconfig/network-scripts/route-ens161

cat > /etc/sysconfig/network-scripts/route-ens161 << "EOF"
default via 192.168.10.2
192.168.20.0/24 via  0.0.0.0
192.168.21.22/32 via 0.0.0.0
EOF
~~~

>注意：如果网卡 ens161 上已经有默认路由了则不能再新建新的默认路由
>
>注意：配置的 IP 地址必须可达

保存配置文件后，重启服务生效。



## RockyLinux9.6 配置永久路由

路由配置文件在目录 `/etc/NetworkManager/system-connections/` 下面，网卡设备 ens161的配置信息都在文件 `ens161.nmconnection` 中，一般不直接编辑这个文件，而是使用 `nmcli` 命令操作路由的添加和删除。

### 1. 增加路由

~~~bash
# 新建 默认路由 网络路由 主机路由
# 删除路由使用 -ipv4.routes 
nmcli conn mod ens161 +ipv4.routes "0.0.0.0/0 192.168.10.14"
nmcli conn mod ens161 +ipv4.routes "192.168.20.0/24 0.0.0.0"
nmcli conn mod ens161 +ipv4.routes "192.168.20.11/32 192.168.10.14"

# 重启生效
nmcli conn up ens161
~~~

>注意：配置的 IP 地址必须可达



