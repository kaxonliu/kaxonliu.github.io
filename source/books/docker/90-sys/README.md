# 修改容器内核参数

在容器内修改内核参数，方式有很多种。



## 特权模式启动容器

如果我们用特权模式启动，在容器中，可以任意修改 sysctl 内核参数，但是你需要知道的是 
1、开启特权模式会增加安全风险。
2、当你开启特权模式之后，在容器内修改的一些内核参数，可能会影响到宿主机（进而影响到该宿主机上所有其他容器），也可能不会影响。

~~~bash
docker container run -ti --rm --privileged --name test centos:7 sh
~~~



目录 `/proc/sys` 下的内核参数并不是所有的都是名称空间的隔离。隔离的参数有：

	kernel.shm*,
	kernel.msg*,
	kernel.sem,
	fs.mqueue.*,
	net
注意， vm.并没有namespace化。 比如vm.max_map_count， 在主机或者一个容器中设置它， 其他所有容器都会受影响，都会看到最新的值。



## 宿主机上使用 nsenter

如果你可以用 root 账号登录到宿主机，那么可以借助 nsenter 命令进入容器名称空间修改，这种不需要开启特权模式，但效果与其是类似的。即如果修改的是 namespaced 的参数，则不会影响宿主机和其他容器。反之，则会影响它们。

~~~bash
# 1、root 用户
nsenter -t <容器的pid> -n bash -c 'echo 600 > /proc/sys/net/ipv4/tcp_keepalive_time'
 
# 2、非root用户
sudo nsenter -t <pid> -n sudo bash -c 'echo 600 > /proc/sys/net/ipv4/tcp_keep
alive_time' 
~~~



## 启动容器时修改

为何一定要在容器启动之初完成修改呢？

因为容器一旦启动之后，其内部运行的服务有可能就会与客户端建立好了网络连接了。在这之后，你再去修改容器内的内核参数，是影响不到已经创建的链接的。

使用  `--sysctl` 实现启动容器时修改。

~~~bash
docker run -d --sysctl net.ipv4.ip_forward=0 --name test1 centos:7 tail -f /dev/null
~~~

注意：只能指定namespaced隔离的参数。

