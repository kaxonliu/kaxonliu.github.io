# Docker 网络

docker 的网络分为单机网络和跨主机网络。

单机上，默认安装 Docker 后会在主机上创建创建三个网络，分别是 `bridge`、`host`、`none`。

## bridge 桥接网络

Docker 默认使用的网络模式，使用 `docker run` 命令创建容器时如果不指定网络模式，那么就会使用 bridge 模式。

使用这种网络模式的容器，都会挂载到一个网桥设备，默认叫 `docker0`，本质就是一台虚拟机的二层交换机。每创建一个容器，都会生成一个 veth对，veth对 顾名思义是成对出现的，veth  对就好像一根网线，网线的一端接到容器内，被命名为 `eth0`，另外一端接到宿主机上的 `docker0` 网桥上，名为 `vethxxx`。

容器内的服务想要被外部访问需要暴露端口并和宿主机做端口映射。

**运行容器时指定挂载网络**

~~~bash
docker run --network bridge --name my-nginx nginx
# 默认 网络就是 bridge，等价于
docker run  --name my-nginx nginx
~~~

**查看网桥上的接口情况**

~~~bash
yum install -y bridge-utils
brctl show

# 查看docker0维护的mac地址情况
[root@me ~]# brctl showmacs docker0
port no mac addr                is local?       ageing timer
  3     46:6a:51:67:cf:71       yes                0.00
  3     46:6a:51:67:cf:71       yes                0.00
  2     92:06:80:fa:de:01       yes                0.00
  2     92:06:80:fa:de:01       yes                0.00
  1     da:4a:2e:59:7d:8e       yes                0.00
  1     da:4a:2e:59:7d:8e       yes                0.00
~~~

**查看 veth pair** 

~~~bash
# 1、进入容器内，查看容器网卡配对的veth设备的id号
cat /sys/class/net/eth0/iflink  #假设输出的是61

# 2、然后在宿主机上查看veth设备的id号，找到对应的veth设备
ip link show  # 会找到一个编号为61的veth设备
~~~



## host 主机网络

使用宿主机的网络，容器将不会获得一个独立的网络命名空间，配置和宿主机共享，容器将不会隔离宿主机网络，使用宿主机的 IP 和端口。但是其他资源依然是隔离。

直接采用宿主机网络，容器内监听的端口无需任何映射，直接就监听在宿主机上。

~~~bash
docker run --network host --name my-nginx2 nginx
~~~



## none 无网络

禁用网络，容器拥有自己的网络命名空间，但是并不为容器进行任何网络配置。也就是说，这个 Docker 容器没有网卡、IP、路由等信息。需要我们自己为 Docker 容器添加网卡、配置 IP 等。

这个网络模式的容器只适合于只进行数据处理，没有任何网络的应用场景。

~~~bash
docker run --network none --name my-nginx3 nginx
~~~



## container 网络

新建容器不创建网络名称空间，与某个已经存在的容器共享网络。

~~~bash
docker run --network container:<容器名a> --name my-nginx4 nginx
~~~



## docker network 操作

**列出网络**

```
docker network ls
```

**查看网络信息**

```
docker network inspect <network>
```

**创建网络**

```
docker network create <network>
```

- 创建自定义网络时，可以使用参数 `--driver` 指定创建的网络属于什么类型，默认是 `bridge`

**删除网络**

```
docker network rm <network>
```

**运行容器时指定挂载网络**

~~~bash
docker run --network bridge --name my-nginx nginx
~~~

- 不指定 `--network` 即默认挂载在 `bridge`上，即挂载于 `docker0`上。



### 默认 bridge 网络与自定义 bridge 网络的区别

- **默认 bridge 网络**：在默认的 `bridge` 网络中，容器之间只能通过 IP 地址通信，无法通过容器名称通信，因为默认 bridge 网络没有内置 DNS 服务。
- **自定义 bridge 网络**：在自定义 bridge 网络中，Docker 提供了 DNS 服务，因此容器之间可以通过容器名称通信。



新建自定义 bridge，并将两个容器挂在上面

```bash
# 创建自定义 bridge 网络
docker network create my-network

# 运行容器并加入自定义网络
docker run -d --name nginx1 --network my-network nginx
docker run -d --name nginx2 --network my-network nginx
```

在 `nginx1` 中，你可以直接通过容器名称 `nginx2` 来 ping 通它，同理反之亦然。Docker 的 DNS 服务会将 容器名称 解析为它的 IP 地址，从而实现通信。



## docker run --link 不推荐使用

`docker run --link` 是 Docker 早期版本中用于实现容器之间通信的一种机制。它可以将一个容器的网络信息（如 IP 地址和端口）注入到另一个容器的环境变量和 `/etc/hosts` 文件中，从而实现容器之间的通信。但**容器之间是单向通信**。不过，**`--link` 是一个过时的功能**，在 Docker 的现代版本中，推荐使用**自定义网络**（如自定义 `bridge` 网络）来代替 `--link`。

~~~bash
docker run -d --name nginx1  nginx
docker run -d --name nginx2 --link nginx1:mynginx1 nginx
~~~

`--link nginx1:mynginx1`：将 `nginx2` 链接到 `nginx1`，并为 `nginx1` 设置一个别名 `mynginx1`，这样做的结果是，在`nginx2` 可以使用容器名称 `nginx1` 或者 `mynginx1` ping 通过。但是在 `nginx1` 中无法使用容器名 `nginx1` ping 通



## docker 跨主机网络

docker 跨主机网络有两种模式：`macvlan` 和 `overlay`。

docker 允许用户创建基于 vxlan 的 overlay 网络。vxlan 将二层数据封装到 udp 进行传输，vxlan 提供与 vlan 类似的以太网二层服务，但是拥有更强的灵活性和扩展性。即 overlay 基于 vxlan 实现，gre 与 vxlan 都是物理上的三层，虚拟上的二层，即三层转发二层包。



基于 vlan 模式来构建虚拟机跨宿主机通信的网络有两点问题：

- vlan id 采用 12bit 位标识，意味着最多也就4000多个网络，会限制网络的规模
- 物理节点必须处于一个二层网络。

基于 gre 与 vxlan 技术能否突破上述两点限制。其实 gre 与 vxlan 两种协议的基本原理都是利用已有协议来封装自己新的东西。



GRE 技术本身还是存在一些不足之处：

（1）Tunnel 的数量问题

    GRE 是一种点对点（point to point）标准。Neutron 中，所有计算和网络节点之间都会建立 GRE Tunnel。当节点不多的时候，这种组网方法没什么问题。但是，
当你在你的很大的数据中心中有 40000 个节点的时候，又会是怎样一种情形呢？使用标准 GRE的话，将会有 780 millions 个 tunnels。

（2）扩大的广播域

   GRE 不支持组播，因此一个网络（同一个 GRE Tunnel ID）中的一个虚机发出一个广播帧后，GRE 会将其广播到所有与该节点有隧道连接的节点。

（3）GRE 封装的IP包的过滤和负载均衡问题

    目前还是有很多的防火墙和三层网络设备无法解析 GRE Header，因此它们无法对 GRE 封装包做合适的过滤和负载均衡。

vxlan
	特点：
		1、基于udp协议进行分发，所以vxlan是无状态的，而gre是有状态的，
		2、支持组播
	

gre：
	特点：
		1、通过是基于ip协议就信息分发
		2、不支持组播
		3、没新增与一个节点，所有相关节点都需要与其简历GRE隧道链接