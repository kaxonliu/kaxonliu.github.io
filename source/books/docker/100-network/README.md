# 容器网络

## 手动构建 bridge 网络模式

#### 1. 创建一个容器，选择none网络模式

~~~bash
docker run -d --network none --name test centos:7 tail -f /dev/null
~~~

使用如下命令查看容器内只有一个本地回环网卡

~~~bash
nsenter -t <container_pid> -n ip a
~~~

>补充，`<container_pid>` 指的是容器内一号进程在宿主机上对应的 PID 号。查看方法如下：
>
>~~~bash
>docker inspect <container> |grep -i pid
>~~~



#### 2. 在宿主机创建一个 veth 对

veth 对 用来联通两个名称空间。默认两个veth设备都创建到了宿主机上。

~~~bash
ip link add name veth_host type veth peer name veth_container
~~~

#### 3. 把 veth 对的一端接到容器名称空间内

~~~bash
# 把你要操作的名称空间链接到ip命令指定的目录下才可以被ip命令处理
ln -s /proc/<container_pid>/ns/net /var/run/netns/<container_pid>
ip link set veth_container netns <container_pid>

# 处理容器内的veth设备，保证其可用
# 改名为 eth0
ip netns exec <container_pid> ip link set veth_container name eth0

# 激活
ip netns exec <container_pid> ip link set eth0 up

# 配置ip
ip netns exec <container_pid> ip addr add 172.17.1.2/16 dev eth0 

# 配置网关
ip netns exec <container_pid> ip route add default via 172.17.0.1
~~~



#### 4. 把 veth 的的另外一端放到宿主机上

忽略，因为默认就在。但是要激活

~~~bash
ip link set veth_host up
~~~



#### 5. 把宿主机上的那个 veth 设备插到虚拟交换机上，即docker0上

~~~bash
ip link set veth_host master docker0
~~~

#### 6. docker0 网桥转发到物理网卡

~~~bash
# 宿主机开启路由转发功能
echo 1 > /proc/sys/net/ipv4/ip_forward

# 把FORWARD链设置为ACCEPT
iptables -P FORWARD ACCEPT
~~~





## 使用抓包工具排序网络故障

抓包工具使用 `tcpdump`，把通信链路上涉及到的所有网络设备都抓包分析。

#### 容器内的 eth0

~~~bash
# 查看容器一号进程pid
docker container inspect <container> |grep -i pid

# 抓包
nsenter -t <pid> -n tcpdump -i eth0 host www.baidu.com -nnv
# 抓包
nsenter -t <pid> -n tcpdump -i eth0  -p icmp -nnv and host  www.baidu.com
~~~

#### 宿主机上的 veth_host

~~~bash
tcpdump -i veth_host -p icmp -nnv and host www.baidu.com
~~~

#### 宿主机上的网桥设备 docker0

~~~bash
tcpdump -i docker0 -p icmp -nnv and host www.baidu.com
~~~

####  宿主机的物理网卡 ens32

~~~bash
tcpdump -i ens32 -p icmp -nnv and host www.baidu.com
~~~

#### 防火墙的转发规则

~~~bash
iptables -t nat -L -n
~~~

#### 宿主机是否开启了路由转发功能

结果必须为 1

~~~bash
cat /proc/sys/net/ipv4/ip_forward
~~~

