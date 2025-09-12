# 配置 events

`events` 块是配置 nginx 如何处理连接和请求的核心部分，它直接影响到服务器的性能和并发处理能力。 

~~~nginx
events {
    # 事件模型配置
    use epoll;
    
    # 连接数配置
    worker_connections 1024;  # 注意：这个1024并不是开启1024个线程
    
    # 连接处理优化
    multi_accept on;
    accept_mutex off;
}
~~~



## 字段 worker_connections

设置每个 worker 进程同时打开的最大连接数。可以调大，但是**文件描述**也要一起调大，并且要综合考虑硬件性能。

当前 Nginx 服务器能够处理的并发请求数量计算公式

~~~
最大并发连接数 = worker_processes × worker_connections
~~~



## 字段 use

指定使用的事件处理模型（通常让Nginx自动选择最佳方案）。

~~~nginx
events {
    use epoll;  # Linux系统推荐，性能最强。
    use kqueue;  # FreeBSD系统
    use select;  # 兼容模式（性能最差）
}
~~~



## 字段 multi_accept 和 accept_mutex

`multi_accept` 控制一个 worker 进程在一次事件循环中是否尽可能多地接收多个新连接，默认关闭。`accept_mutex` 是一个互斥锁，用于控制多个 worker 进程如何竞争接收新连接，解决"惊群问题"，默认开启。

>惊群问题（thundering herd problem）：新连接到达时，如果没有互斥锁，所有空闲的worker进程都会被唤醒并尝试接受连接，但只有一个能成功，其他进程白白被唤醒，造成资源浪费。

**场景：连接数适中**

~~~nginx
events {
    multi_accept on;      # 一次接受多个连接
    accept_mutex on;      # 避免惊群问题
    worker_connections 1024;
}
~~~

**场景：高并发**

~~~nginx
events {
    multi_accept on;      # 必须开启
    accept_mutex off;     # 禁用互斥锁以减少延迟
    worker_connections 8192;
}
~~~

**场景：机器性能低**

~~~nginx
events {
    multi_accept off;     # 节省内存
    accept_mutex on;      # 节省CPU
    worker_connections 512;
}
~~~

#### 调整策略

1. **先启用 `multi_accept on`**
2. **根据负载决定 `accept_mutex`**：
   - 连接数 < 2000/秒：`accept_mutex on`
   - 连接数 > 5000/秒：尝试 `accept_mutex off`
3. **监控CPU使用率**：如果CPU成为瓶颈，重新启用互斥锁

#### 总结

- **`multi_accept`**：主要影响单个 worker 进程的处理效率，**建议通常开启**
- **`accept_mutex`**：主要影响多 worker 进程间的协调，需要根据并发量调整

