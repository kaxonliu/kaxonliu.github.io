# nginx 优化



## 半连接队列和全连接队列

TCP 三次握手时，linux 内核会维护两个队列：半连接队列（SYN 队列）和全连接队列（ACCEPT 队列）。

服务端收到客户端的 SYN 请求后，内核会把这个连接放入半连接池（半连接队列），然后回复客户端 SYN-ACK，接着客户端返回 ACK，服务端收到后会把连接从半连接队列中移除，并放入全连接队列，等待应用程序从全连接对垒中取出使用。半连接队列和全连接队列都有最大长度限制，超出限制时，请求就会被丢弃并返回 RST。

Linux 内核中这两个最大长度都是可以设置的，默认都是 128。
- 设置全连接池大小：`sysctl -w net.core.somaxconn=65530`
- 设置半连接池大小：`sysctl -w net.ipv4.tcp_max_syn_backlog=10240`



### nginx 设置连接池大小

nginx 配置中 `backlog` 用来配置 SYN 队列的大小和 ACCEPT 队列的大小，它俩默认值是 511

~~~nginx
server {
    listen 80 backlog=10240;
} 
~~~

>注意：`backlog` 设置的值要配合内核中 `somaxconn` 和 `tcp_max_syn_backlog`。在提高内核值的基础上，再提高nginx 的配置，这样才能起到提高并发能力的效果。



查看队列是否溢出的命令：

- 查看全连接队列的溢出情况：`netstat -s | grep "overflowed"`
- 查看半连接队列的溢出情况：`netstat -s | grep "SYNs to LISTEN"`



### nginx 中 worker 的工作模式

Nignx 里面的 master 进程负责管理多个 worker 进程，worker 进程负责处理请求任务。worker 进程从全连接队列中取出请求然后处理请求。worker 使用全连接队列的方式分为两种情况（工作模式）。

#### 多个 worker 共享一个全连接队列

这是 nginx 默认的方式。只有一个 ACCEPT 队列，所有的 worker 都从其中取出连接然后处理请求任务。

- **优点**：因为共享一个全连接队列，耗时的任务只会阻塞一个 worker，其他空闲的 worker 可以继续取出连接并处理请求，不会拖慢全部连接的处理。
- **缺点**：会发生进程级别的争抢导致资源额外消耗。



#### 每个 worker 独享一个全连接队列

这种模式下每个 worker 都有一个自己专属的全连接队列，取连接时不会发生进程级别的争抢现象。所有 worker进程会都监听相同的接口。然后由内核负责将请求负载均衡到这些监听进程中去，相当于每个进程独享自己的全链接队列。

- **优点**：减少进程级别争抢，减少额外消耗。
- **缺点**：worker 处理耗时任务时，它队列里面的连接无法分享给其他空闲的 worker 进程。导致单个连接请求的延迟增大，CPU 分配不均匀。



#### nginx 开启 reuseport 

nginx 使用 `reuseport` 启用复用端口，开启多个 worker 独享一个全连接队列模式。

~~~nginx
server {
    listen 80 reuseport;
}
~~~

开启后查看 listen 的 socket 端口情况

~~~bash
# worker 进程设置为4个, 4个worker进程 reuse了相同的端口
[root@rocky ~]# ss -lnt |grep 8888
LISTEN 0      511               0.0.0.0:8888       0.0.0.0:*   
LISTEN 0      511               0.0.0.0:8888       0.0.0.0:*   
LISTEN 0      511               0.0.0.0:8888       0.0.0.0:*   
LISTEN 0      511               0.0.0.0:8888       0.0.0.0:*   

# 因为是resuse重用的同一个端口，
# 所以在系统层面只能看到一个端口
[root@rocky ~]# netstat -tunlap | grep -w 8888
tcp        0      0 0.0.0.0:8888            0.0.0.0:*               LISTEN      31432/nginx: master 
~~~



#### 开启 reuseport 的效果

- CPU 负载下降
- CPU 使用率下降
- 上下文写换次数下降
- Nginx 服务平均延迟下降，单个请求的最高延迟可能会增加
- Nginx 慢请求数量下降



#### 启动 worker 多线程模式

一个 worker 进程内只有一个线程，这意味着每个工作进程在同一时间只有一个在处理客户端请求。尽管每个工作进程是单线程的，但 nginx 通过事件驱动和非阻塞I/O的方式（epoll 模型）能够处理大量并发请求，并且单线程的模式避免了上下文切换的开销，实现高性能和高吞吐量。这种设计在处理静态内容和反向代理等场景下表现出色。

单线程的 worker 可能因为耗时任务阻塞全连接池，此时在 worker 进程内使用多线程模式，可以让 CPU 分配相对均匀、提高 CPU 利用率。

**配置 worker 进程内开启多线程**

~~~nginx
# 定义一个名为‘my_pool’的线程池，3个线程，最大队列1024
thread_pool my_pool threads=3 max_queue=1024;

http {
    # 使用自定义线程池
    aio threads=my_pool; 
}
~~~

其中，
- 指令 `thread_pool` 定义和命名一个线程池。在 `main`  中使用。其中 `max_queue` 指定等待线程池处理的任务队列的最大长度。当所有线程都在忙时，新任务会进入队列等待。如果队列也满了，任务会被拒绝并记录错误。
- 指令 `aio` 指定启用线程池。可以在 `http`, `server`, `location` 中使用。
  - 使用自定义线程池  `aio threads=my_pool; ` 
  - 使用默认线程池 `aio threads; `  Nginx 会使用名为 `default` 的线程池。如果你没有定义名为 `default` 的线程池，Nginx 会自动创建一个（使用默认参数：`threads=1, max_queue=65536`）。

通过合理配置线程池，可以让 Nginx 在处理大文件下载等高磁盘 I/O 负载的场景时，依然保持极高的并发能力和整体性能。

>扩展：在 k8s 中 ingress-nginx 启用了端口复用，开启了 aio 线程池。



#### 启用线程池的前提

**确保 Nginx 编译时包含了线程池支持**。通过以下命令来检查，如果输出 `with-threads`，则说明支持线程池。

```bash
nginx -V 2>&1 | grep -o with-threads
```




