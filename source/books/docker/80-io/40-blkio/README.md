# 限制磁盘读写性能

一台宿主机上运行的多个容器，其实大家本质都是在用宿主机的资源，包含 I/O 资源。所以说，容器的 IO 速率是彼此影响的，为了规避这种影响，应该给每个容器都设置合理的 IO 资源。

使用 fio 测试可以发现，多个容器同时进行写操作，IOPS 和 BW 都会下降。



## 容器内的文件系统不适合频繁读写

因为容器里的文件系统是 overlayFS，容器内发起的写操作到 overlayFS 之后，到操作系统之后还需要转换成操作系统的文件系统（ext4、xfs等）的写操作。所以如果容器内涉及到频繁的写操作，建议给容器挂载单独的数据卷。

>补充：在 linux 内核4.15之后，为了 OverlayFS 添加了自己的 read/write 接口，从而不再直接调用 OverlayFS 后端文件系统如 XFS 或 Ext4 的读写接口，但也只实现了同步 I/O（sync I/O），没有实现异步 I/O。如果我们用 fio 工具去测试文件系统性能，若采用异步 io 模式，在 linux 5.4 内核上就无法测试 OverlayFS 的最高性能指标。
>
>OverlayFS 在 Linux 内核中还在不断的完善中，linux5.6 内核通过打补丁的方式修复了上述问题。



## 基于 cgroup v1 限制容器磁盘 IO

我们可以用 Cgroups 机制限制容器的 cpu、内存资源使用，同样可以用它来限制容器的磁盘性能。具体如何限制，需要分 cgroup v1 与 cgroup v2 两个版本来看。
Cgroup v1 中有 blkio 子系统，它可以来限制磁盘的 I/O，blkio cgroup 的虚拟机文件系统挂载点通常在 `/sys/fs/cgroup/blkio`，有四个重要参数：
- 读与写的 IOPS  
  - blkio.throttle.read_iops_device  
  - blkio.throttle.write_iops_device  
- 读与写的带宽 BW
  - blkio.throttle.read_bps_device  
  - blkio.throttle.write_bps_device



#### 示例：限制容器 BPS

第一步：常见容器 test1

第二步：获取容器 test1 的路径

~~~bash
CONTAINER_NAME="test1"
CONTAINER_ID=$(docker ps --format "{{.ID}}\t{{.Names}}" | grep -i $CONTAINER_NAME | awk '{print $1}')
echo $CONTAINER_ID 
CGROUP_CONTAINER_PATH=$(find /sys/fs/cgroup/blkio/ -name "*$CONTAINER_ID*") 
 
echo $CGROUP_CONTAINER_PATH
~~~

第三步：需要获取目录挂载源的设备的主、次设备号。我们要查看的设备的主次设备号，而不是某一个分区的，如下所示 `/dev/vda` 的主次设备号分别为 253 和 0

~~~bash
[root@me ~]# ls -l /dev/vda
brw-rw---- 1 root disk 253, 0 Sep  3 16:52 /dev/vda
~~~

第四步：限制磁盘性能只需要配置带宽。配置容器 test1 对设备 `/dev/vda` 的读写带宽均为10M/s。

~~~bash
echo "253:0 10485760" > $CGROUP_CONTAINER_PATH/blkio.throttle.read_bps_device
echo "253:0 10485760" > $CGROUP_CONTAINER_PATH/blkio.throttle.write_bps_device
~~~



##  基于 Cgroup v2 限制 buffer io

Cgroup v2 要想能够限制 buffer io 模式，必须同时考虑限制磁盘与内存，即 blkio Cgroup + memory Cgroup 可以设置到一起，而不是像 cgroup v1 那样分为两个子系统。

这也是 cgroup v1 与 cgroup v2 的最大区别，前者在限制磁盘 IO 时无法考虑 page cache，这个问题在 cgroup v2 中得到了解决。cgroup v2 从架构上允许一个控制组里有多个子系统协同工作，也就是说，在同一个控制组里只要同时有 io 与 memory 子系统，就可以对buffered IO 模式做磁盘限速。

虽然 cgroup v2 解决了对 buffered io 模式下的磁盘限速问题，但是你要知道的是迁移到cgroup v2 的时机尚不成熟。

尽管 cgroups v2 旨在替代 cgroups v1，但是较旧的系统继续存在（出于兼容性原因，不太可能被立即删除）。目前，cgroups v2 仅实现 cgroups v1 中可用的控制器子集，v1 控制器和 v2 控制器都可以安装在同一系统上。

还有一点就是，cgroup v2 是从 linux 4.5 之后正式发布的，你的内核并不一定支持。判断当前系统内核是否支持 cgroup v2，如果显示有 cgroup2 则说明内核是支持的：

```
[root@test04 ~]# grep cgroup /proc/filesystems
nodev cgroup
nodev cgroup2
```

