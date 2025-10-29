# 内存

主要讨论 OOM 和 swap 分区在系统级别和容器级别的影响。



## OOM

OOM 全称 Out of Memory（内存不足/内存溢出），是 linux 系统的一种保护机制。

当宿主机上运行的进程对内存的占用达到了一定的量，就会触发 OOM 机制然后杀死某个正在运行的进程。主要有两种触发情况：

- 达到宿主机最大内存使用量，那一定会触发 OOM 机制，站在整个系统的角度去杀进程，所有进程都是目标。会按照特定打分标准，把选出来的进程杀掉。
- 在宿主机内存够用的前提下，某个容器内的进程对内存的使用量达到了 memory cgroup 的限制，也会触发 OOM 机制，此时只能杀掉容器控制组内的进程。



#### 为何触发系统级别 OOM 机制一定要杀掉进程

因为 linux 系统的内存是超配（overcommit），这意味着有很多进程申请了很多虚拟内存，但是实际大多数只使用了小部分内存，这些用的都是实际的物理内存。如果此时物理内存不足了，意味着此时如果那些超配给进程的内存空间真的要投入使用，而实际上已经没有物理内存可使用了，那意味着正常运行的进程会挂掉。所以说，一旦发生内存不足，就会立即触发 OOM 机制来杀进程释放内存。

#### OOM 打分标准

当系统级 OOM 发生时，并不会随机杀进程，具体杀哪个进程是会调用内核了一个 `oom_badness()` 函数来评定，主要就是两个指标：

- 进程已经使用的物理内存页面数
- 每个进程的 OOM 校准值 （oom_score_adj），值的范围是 -1000 ~ 1000

评分 = 系统总的可用页面数 * oom_score_adj 值 + 进程已经使用的物理页面数

评分值越大，被 OOM 杀掉的概率就越大。

>在 `/proc` 文件系统中，每个进程都有一个 `/proc/<PID>/>oom_score_adj` 的接口文件，我们可以在该文件中输入 -1000 到 1000 之间的任意一个数值，来调整该进程被OOM kill 的概率。



#### memory cgroup

每创建一个容器都会创建一个容器的 memory cgroup 控制组，在目录 `/sys/fs/cgroup/memory/system.slice/docker-<containerID>` 下面有需要关注的参数。

主要关注三个参数：

- `memory.limit_in_bytes` 。可以配置。控制的是容器内所有进程可以占用的物理内存上限。注意：如果父级 group 设置的 `memory.limit_in_bytes` 为 500M，那么子 group 最多只能设置到 500M。
- `memory.oom_control` 。可以设置。默认值为0 代表开启 OOM。可以设置为 1，代表关闭。`echo 1 > memory.oom_control` 
- `memory.usage_in_bytes`。只读参数。里面的内容是容器内所有进程占用的物理内存总量，等于实际占用物理内存 RSS + 读写缓存 page cache



## 如何分析 OOM 日志

查看日志文件 `/var/log/messages`，如果看到发生 OOM 的是操作系统的进程还是 容器或者控制组的进程，可以断定发生的是系统级别的 OOM 还是哪个容器发生 OOM，在看到被 OOM killer 掉的进程 PID.

有上述信息后，接下来有两种处理方案。

- 被 OOM 干掉的进程可能本身就是需要很大的内存，我们需要调大 `memory.limit_in_bytes`
- 被 OOM 干掉的额进程存在 BUG，导致内存泄漏，从而达到了 `memory.limit_in_bytes` 的限制，此时就需要解决bug了。



## 容器内内存的使用

参数 `memory.usage_in_bytes` 表示容器内所有进程的内存使用量，这个值包含两部分数据，一个是进程实际占用的物理内存 RSS，另一个是 page cache 的大小。其中，page cache 部分的值可以被释放回收。

所以当容器的参数 `memory.usage_in_bytes` 的值超过了参数 `memory.limit_in_bytes` 的值之后，即使开启了 OOM，容器也可能不会被 OOM 杀掉。因为新申请的内存空间可以通过释放 page cache 的空间来补充。

~~~bash
# 手动释放 page cache
echo 3 > /proc/sys/vm/drop_caches
~~~

>补充：free 命令看到的的 buffer 写缓冲与 cache 读缓存统称为 page cache，过去确实分为两个，现在统一为一个概念。



#### 内存回收机制具体回收方式

在 malloc() 申请内存发现内存不够用时会触发 linux 的 page frame reclain 内存页面回收机制，该机制会根据系统里空闲物理内存是否低于某个阈值 wartermark，低于则启动内存回收。具体回收哪些内存，会根据 LRU（Least Recently Used）算法计算。



#### 查看容器实际内存占用

要看容器内所有进程占用的实际内存，`memory.usage_in_bytes` 是不准确的，看 rss 才最准确的，那就要查看 `memory.stat`，过滤 rss `cat memory.stat | grep rss`，可以看到如下信息。

~~~bash
rss 98304
rss_huge 0
total_rss 98304
total_rss_huge 0
~~~



## swap 分区

swap 分区的本质就是拓展出来的一块内存，是在磁盘空间上临时存放进程的内存数据。

启动了 swap 分区的物理机，可用内存 = 物理可用内存大小 + swap 分区的大小。当物理内存不足时，就会开始使用 swap 分区。虽然会降低系统的运行效率，但是可以保证系统不会奔溃。当交换分区也不够用时，就会触发 OOM 机制。



#### swap 分区对系统级 OOM 的影响

系统级的 OOM 会在可用内存不足时触发，如果宿主机启用 swap 分区，那详细的说
系统 OOM 会在物理内存用完，然后 swap 分区紧接着用完的情况下触发执行。

#### swap 分区对容器级 OOM 的影响

前提我们没有关闭容器级的 OOM，swap 分区开启的情况下，对容器带来的影响是，即便容器内所有进程占用的内存量达到了容器最大内存限制也不会触发 OOM，会开始使用 swap 分区。

设置容器的物理内存使用上限 200M，物理内存和 SWAP 上限是 300M，即交换空间最大是 100M。

~~~bash
docker run -m 200M --memory-swap=300M
~~~

#### 如何控制全局和局部使用 swap

需求：宿主机上一些容器需要使用 swap 分区，另外一些容器则不要用。

第一步：启动 swap

第二步：在系统设置 `swappiness` 参数，控制 swap 分区也就内存内存与 page cache 释放的比重。

~~~bash
echo 0 > /proc/sys/vm/swappiness  
# swappiness=0 后并不代表禁用swap分区，只是告诉内核，能少用到swap分区就尽量少用到
# swappiness=100，则表示尽量使用swap分区，默认的值是60
~~~

第三步：针对个别容器如果想要关闭 swap 分区的使用

~~~bash
docker run -d --memory-swappiness=0 ubuntu:16.04 
~~~

容器的 memory.swappiness 与全局的 swappiness 有两点不同：

1、容器自己的优先级更高。

2、如果把 memory.swappiness 设置为 0，对于容器来说就是彻底关闭了对 swap 分区的使用。而如果把系统级的 swappiness 设置为 0，则代表尽量不用，并非是彻底不用。



#### 补充

swap 和 page cache 都是内存优化的方案，问题是当内存不足时，是释放 page cache 还是写入 swap 分区？单独使用谁都不合适，因此需要在两者之间找到一种平衡，这个平衡参数就是 `swappiness`。 它的值范围是：0-100，默认值是 60。

- 值为 100，等比例释放。
- 值为 60，page cache 释放比例要优先于 swap 交换。
- 值为 0，表示表示尽可能释放 page cache，尽量不使用 swap，但是在内存紧张时依然会使用 swap 回收匿名内存。