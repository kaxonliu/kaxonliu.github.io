# 测试硬盘读写性能

方式很多，推荐使用 fio 

## dd 命令

测试读：从 `/dev/vda` 读数据写到 `/dev/null`，每次读 8K，读1000次

~~~bash
[root@me ~]# time dd if=/dev/vda of=/dev/null bs=8k count=10000  
10000+0 records in
10000+0 records out
81920000 bytes (82 MB) copied, 0.237815 s, 344 MB/s

real	0m0.239s
user	0m0.010s
sys	0m0.039s
~~~

测试写。要测试实际速度 ，要在末尾加上 `oflag=direct` 测到的才是真实的 IO 速度。

~~~bash
[root@me ~]# time dd if=/dev/zero of=/a.txt bs=8k count=10000 oflag=direct
10000+0 records in
10000+0 records out
81920000 bytes (82 MB) copied, 6.80587 s, 12.0 MB/s

real	0m6.822s
user	0m0.073s
sys	0m0.396s
~~~



## fio

实际读写硬盘涉及到多因素：

- 应用程序。可能并发 IO，提交读写任务分同步提交和异步提交。
- 操作系统。两种 IO 模式，bufferd io 和 direct io
- 硬盘。两种访问方式，随机访问、顺序访问



fio 是测试 IOPS 的非常好的工具，用来对磁盘进行压力测试和验证。磁盘 IO 是检查磁盘性能的重要指标，可以按照负载情况分成顺序读写、随机读写两大类。fio 可以产生很多线程或进程并执行用户指定的特定类型 I/O 操作，fio 的典型用途是编写和模拟的 I/O 负载匹配的作业文件。也就是说 fio 是一个多线程 io 生成工具，可以生成多种 IO 模式，用来测试磁盘设备的性能（也包含文件系统：如针对网络文件系统NFS的IO测试）。

另外还有 GFIO 则是 fio 的图形监测工具，它提供了图形界面的参数配置，和性能监测图像。

fio 在 github 上的仓库：https://github.com/axboe/fio。

#### 安装 fio

~~~bash
yum install fio libaio-devel -y  # 后者为异步io引擎依赖包
~~~



#### 测试读

~~~bash
fio -thread=1 -numjob=3 -ioengine=libaio -iodepth=16 \
-direct=1 -rw=read -bs=4k -size=5G -runtime=60 \
-name "my read test BS 4K" \
-filename=/dev/vda
~~~

#### 测试写

千万注意，不要直接指定设备，而是创建一个写目录。

~~~bash
mkdir /aaa
fio -thread=1 -numjob=3 -ioengine=libaio -iodepth=16 \
-direct=1 -rw=write -bs=4k -size=5G -runtime=60 \
-name "my write test BS 4K" -directory=/aaa
~~~



#### fio 参数说明

~~~bash
fio -directory=/test/ ……
 
参数说明
-name=mytest 指定本次测试任务名，自定义即可
-filename: 指定文件(设备)的名称。可以通过冒号分割同时指定多个文件，如filename=/dev/sda:/dev/sdb。
-directory: 设置filename的路径前缀。在后面的基准测试中，采用这种方式来指定设备。
 
-direct: bool类型，如果设置成true或1，表示不使用io buffer，代表从内存直接写入磁盘，测试过程绕过OS自带的buffer，使测试磁盘的结果更真实，即采用O_DIECT的方式，规避buffer写缓冲区带来的影响（实际测试的时候发现即便设置-direct=1也会使使用buffer，但确实设置为1与设置为0的写速度，前者更快，难道是设置direct=1之后会写入buffer然后立即刷入磁盘以此来模拟O_DIECT的方式？这个有待进一步研究）
 
-rw=read 测试顺序读 
-rw=randread 测试随机读 
-rw=write 测试顺序写
-rw=randwrite 测试随机写 
-rw=randrw 测试混合随机读写模式 
 
-ioengine: I/O引擎，现在fio支持19种ioengine（sync，mmap，libaio，posixaio，SG v3，splice，null，network，syslet，guasi，solarisaio等）。默认值是sync同步阻塞I/O，libaio代表异步读取，提交完io请求后可以继续提交下一个
iodepth=16  则代表总共提交了多少个io请求，结合上一个参数的异步提交，此处的io提交是提交完一个立即提交下一个
 
-bs=4k -size=5G 代表单次IO块大小为4k，总共读取5G数据
 
-thread=1  代表使用pthread_create创建线程，另一种是使用fork创建进程。进程的开销比线程要大，一般都采用thread测试。
-numjob=1 代表启动一个任务来执行上述的IO测试，一个任务代表一个线程
 
-runtime=60 代表测试多长时间，测试时间为60秒，如果不写则一直将5g文件分4k每次写完为止。
 
group_reporting: 当同时指定了numjobs了时，输出结果按组显示，汇总每个进程的信息。
 
bsrange=512-2048    同上-bs=4k类似，此处为指定数据块的大小范围
 
Block Devices（RBD） 无需使用内核RBD驱动程序（rbd.ko）。该参数包含很多ioengine，如：libhdfs/rdma等
rwmixwrite=30       在混合读写的模式下，写占30%
此外
lockmem=1g          只使用1g内存进行测试。
zero_buffers        用0初始化系统buffer。
nrfiles=8           每个进程生成文件的数量。
~~~



#### fio 测试结果指标

~~~bash
io=执行了多少M的IO
bw=平均IO带宽
iops=IOPS
runt=线程运行时间
slat=提交延迟，提交该IO请求到kernel所花的时间（不包括kernel处理的时间）
clat=完成延迟, 提交该IO请求到kernel后，处理所花的时间
lat=响应时间
bw=带宽
cpu=利用率
IO depths=io队列
IO submit=单个IO提交要提交的IO数
IO complete=Like the above submit number, but for completions instead.
IO issued=The number of read/write requests issued, and how many of them were short.
IO latencies=IO完延迟的分布
io=总共执行了多少size的IO
aggrb=group总带宽
minb=最小.平均带宽.
maxb=最大平均带宽.
mint=group中线程的最短运行时间.
maxt=group中线程的最长运行时间.
ios=所有group总共执行的IO数.
merge=总共发生的IO合并数.
ticks=Number of ticks we kept the disk busy.
io_queue=花费在队列上的总共时间.
util=磁盘利用率
~~~

