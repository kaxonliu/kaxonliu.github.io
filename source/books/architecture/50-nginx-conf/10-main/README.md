# 全局配置

全局配置的字段有很多，下面是常用的全局配置字段。

~~~nginx
user nobody;
worker_processes  auto; 
error_log /var/log/nginx/error.log info;
pid /run/nginx.pid;

include /usr/share/nginx/modules/*.conf;
~~~



## 字段 user 

使用 `user` 配置 worker 进程的属主。所有的 worker 进程都是 master 进程创建的。 



## 字段 worker_processes

使用 `worker_processes` 配置创建 worker 进程的个数。默认配置值是 `auto`，表示根据服务器 CPU 个数来配置，获取到的逻辑 CPU 核心数（包括超线程技术提供的虚拟核心）就是 worker 进程的个数。当然也可以自己指定固定的个数，建议小于 CPU 核心数。

查看 CPU 的命令

~~~bash
nproc        # 直接显示逻辑核心数
lscpu        # 显示详细的 CPU 信息，包括核心数（Core(s) per socket）和线程数（CPU(s)）
grep -c processor /proc/cpuinfo # 另一种查看逻辑核心数的方法
~~~



## 字段 error_log

Nginx 错误日志的格式不能自定义，但是可以配置日志的路径、级别。

**配置格式**

~~~nginx
# 配置错误日志的路径、日志级别
error_log /var/log/nginx/error.log warn;
~~~

错误日志配置的位置非常灵活，可以配置 `mian`  全局、`http` 块、`server` 块、`location` 块。优先使用自己块内的配置。

- 在 `nginx.conf` 文件的最顶层（不在任何 `{}` 块内）配置，对所有请求生效（推荐）。
- 在 `http { ... }` 块内配置，作用于所有虚拟主机。
- 在 `server { ... }` 块内配置针对特定虚拟主机配置错误日志。
- 在 `location { ... }` 针对特定URL路径配置错误日志（较少使用）。

>补充，错误日志级别
>
>错误日志支持以下级别（严重程度从高到低）：
>
>- `emerg` - 紧急情况（系统不可用）
>- `alert` - 需要立即采取行动
>- `crit` - 严重错误
>- `error` - 错误
>- `warn` - 警告（**默认级别**）
>- `notice` - 需要注意的信息
>- `info` - 一般信息
>- `debug` - 调试信息



## 字段 pid

指定自定义的PID文件路径。



## 字段 include

配置 `include /usr/share/nginx/modules/*.conf;` 是用于**动态加载Nginx模块**。配置在 `events` 和 `http` 块之前。

当Nginx启动时，会读取 `/usr/share/nginx/modules/*.conf` 中的所有文件，每个.conf 文件中的 `load_module` 指令告诉 Nginx 要加载哪个共享对象文件（.so）。

查看系统中可用的模块

~~~bash
ls -la /usr/share/nginx/modules/
~~~

查看 nginx 已加载的模块

~~~bash
nginx -V 2>&1 | grep -o 'with-[^ ]*'
nginx -T | grep load_module
~~~

**示例：加载 stream 模块**

~~~bash
# 安装 stream 模块
yum install -y nginx-mod-stream

# 安装后目录就有 stream 的模块和配置信息了
[root@rocky ~]# ls /usr/lib64/nginx/modules
ngx_stream_module.so
[root@rocky ~]# ls /usr/share/nginx/modules/
mod-stream.conf
[root@rocky ~]# cat /usr/share/nginx/modules/mod-stream.conf
load_module "/usr/lib64/nginx/modules/ngx_stream_module.so";
~~~

