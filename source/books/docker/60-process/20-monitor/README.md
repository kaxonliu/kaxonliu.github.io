# 监控容器 CPU

宿主机上的进程的 cpu 使用率可以使用 top 命令查看，但是 容器内无法通过 top 命令获取真实 cpu 使用率。



## 单个进程对 cpu 的使用率

#### cpu 使用率

某个用户进程对 cpu 的使用率 = 用户进程占用 cpu 时间（us + sy） / cpu 经历的这段总时间。

- 进程对 cpu 的使用率为 100% 表示使用了1颗 CPU；200% 则表示使用 2颗 CPU。
- 如果只有4颗 CPU，那么某个进程 cpu 使用率最大就是 400%。
- 总结：cpu 使用率反应的是 cpu 的利用情况。

#### 平均负载 load average

在某段时间内平均活跃的进程数量，包括运行状态的进程和不可中断状态的进程。如果有4颗 CPU，那么平均负载可以超过4。

总结：平均负载反应 CPU 的工作量。



进程从启动那一刻，linux 操作系统就开始累计该进程对 cpu 资源的占用时间，具体来说负责记录进程对 cpu 资源累计占用的是 linux 系统中的 proc 文件系统。top 命令就是查看 proc 文件系统中每个进程对应的 stat 文件中的2个数字完成统计的。

~~~bash
# 1、测试脚本
[root@test04 ~]# cat a.sh 
i=0
while true;
do
    let i++
done
[root@test04 ~]# 
 
# 2、执行
[root@test04 ~]# sh a.sh &
[1] 43157
 
# 3、查看
[root@test04 ~]# cat /proc/43157/stat | awk '{print $14,$15}'
1314 14
~~~

stat 中的内容有50多项，包括进程 pid、名字、运行状态、优先级、内存使用等。要统计 cpu 使用率，主要关注第14个项目（utime）和第15个项目（stime）。

其中：

- utime 表示进程的用户态部分在系统调度中获取 cpu 的 ticks
- stime 表示进程的内核态部分在系统调度中获取 cpu 的 ticks

ticks 是 linux 系统中的一个时间单位，表示一次中断的周期，这个周期需要耗费的时间由中断频率（HZ）决定，中断频率在 linux 系统中默认是 100。

~~~bash
# 查看中断频率
getconf CLK_TCK
100
~~~

HZ 为 100 代表 1s 内发生 100 次中断。一个 tick 具体时长就是 1/100，如果 utime 等于 150 ticks，那时长就相当于是 150 * （1/100） = 1.5 秒，表示进程从启动那一刻到现在总共在用户态运行的时间是 1.5 秒。



#### 统计进程对 cpu 的占用

进程对 cpu 的利用率 = ((utime_2 – utime_1) + (stime_2 – stime_1)) / et * HZ

- et 为统计时间间隔。

上述公式简化为：

进程的 CPU 使用百分率 =（某段时间内进程占内核态与用户态ticks / 该段时间内单个 CPU 总 ticks）*100.0

- 最后乘以那个100代表转换为百分比。



##  整个系统对 CPU 的使用率

top 命令可以统计整个系统中所有进程对一个 CPU 的使用率，具体原理很简单。还是要借助 proc 文件系统，查看 `/proc/stat`

~~~bash
[root@me 18795]# cat /proc/stat | grep cpu
cpu  8027096 666 1950988 922405384 83864 0 7318 0 0 0
cpu0 4066345 309 987657 461089735 53501 0 5784 0 0 0
cpu1 3960750 356 963331 461315648 30362 0 1533 0 0 0
~~~

- 从 cpu 名开始数，关于数据列总共有10列，前8列对应的就是该 cpu 上的  us/sy/ni/id/wa/hi/si/st 累计的 ticks。
- 如果计算某一项对整个 cpu 的占用率，那就统计一个时间段内所有项的总 ticks，然后用某一项的ticks 除以总 ticks 就好了。



## 获取容器对 cpu 的真实占用率

容器的本质是进程，方法类似。

运行容器后，找到容器 id，然后查看文件 `/sys/fs/cgroup/cpu/system.slice/docker-<container_id>.scope/cpuacct.stat`，就可以看到

~~~
user 2313123
system 12
~~~

- user 后面的数字就是该容器内所有进程在用户态的 ticks
- system 后面的数字就是该容器内所有进程在内核态的 ticks

然后就可以使用公式计算整个容器的 cpu 占用率了。

~~~bash
进程的 CPU 使用率 =((utime_2 – utime_1) + (stime_2 – stime_1)) * 100.0 / (HZ * et * 1 )
~~~



## docker 容器监控

#### docker stats

使用 docker 的原生命令 `docker stats` 查看容器的基本资源使用数据，

~~~bash
# docker stats 容器名或id号 
CONTAINER ID   NAME      CPU %     MEM USAGE / LIMIT    MEM %     NET I/O     BLOCK I/O   PIDS
487ecf4fc7fd   c3        398.82%   3.199MiB / 3.84GiB   0.08%     586B / 0B   0B / 0B     5
~~~



#### cadvisor

这个工具可以对节点机器上的资源和容器做时时监控和性能数据采集，包括 cpu 使用情况、内存使用情况、网络吞吐量及文件系统使用情况。

Cadvisor 使用 golang 语言开发，利用 linux 的 cgroup 获取容器的资源使用信息，在 k8s 中集成在 kubelet 里作为默认启动项。

##### 安装

~~~bash
docker pull google/cadvisor:latest
docker run \
  --volume=/:/rootfs:ro \
  --volume=/var/run:/var/run:rw \
  --volume=/sys:/sys:ro \
  --volume=/var/lib/docker/:/var/lib/docker:ro \
  --volume=/dev/disk/:/dev/disk:ro \
  --publish=8080:8080 \
  --detach=true \
  --name=cadvisor \
  google/cadvisor:latest
~~~

##### 访问

- 查看宿主机上容器资源占用：`http://localhost:8080/containers/`
- 查看某个容器的情况：`http://localhost:8080/docker/`




#### 容器监控三剑客

cadvisor + influxdb + granfana

监控收集 + 时序数据库存数据 + 图表展示