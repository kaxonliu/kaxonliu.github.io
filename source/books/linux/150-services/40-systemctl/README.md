# systemctl 管理服务

在 Centos7 及更高版本上，`systemctl` 是管理系统和服务的主要工具。使用 `systemctl` 管理服务需要为编写服务文件。服务文件存放在两个文件夹中，
- `/usr/lib/systemd/system/` 系统服务，开机不需要登陆就运行的服务。
- `/usr/lib/systemd/user/` 用户服务，用户登陆后运行的服务。



## 配置模板

在 `/usr/lib/systemd/system/` 下编写服务文件 `服务名.service`，每个服务文件一般分为三部分，如下图。

~~~bash
[Unit]   # 主要是服务说明
Description=test   # 简单描述服务
After=network.target # 描述服务类别，表示本服务需要在network服务启动后在启动
Before=xxx.service #表示需要在某些服务启动之前启动，After和Before字段只涉及启动顺序，不涉及依赖关系。
 
[Service]  # 核心区域
Type=forking     # 表示后台运行模式。
User=user        # 设置服务运行的用户
Group=user       # 设置服务运行的用户组
KillMode=control-group   # 定义systemd如何停止服务
PIDFile=/usr/local/test/test.pid    # 存放PID的绝对路径
Restart=no        # 定义服务进程退出后，systemd的重启方式，默认是不重启
ExecStart=/usr/local/test/bin/startup.sh    # 服务启动命令，命令需要绝对路径
PrivateTmp=true                               # 表示给服务分配独立的临时空间
 
[Install]   
WantedBy=multi-user.target  # 多用户
~~~

其中 `Exec*`的配置项是控制服务的配置，有如下配置字段。

~~~bash
ExecStart：    # 启动服务时执行的命令
ExecReload：   # 重启服务时执行的命令 
ExecStop：     # 停止服务时执行的命令 
ExecStartPre： # 启动服务前执行的命令 
ExecStartPost：# 启动服务后执行的命令 
ExecStopPost： # 停止服务后执行的命令
PrivateTmp=True #表示给服务分配独立的临时空间

# 注意：[Service]部分的启动、重启、停止命令全部要求使用绝对路径，使用相对路径则会报错！
# Exec*后面的命令，仅接受‘指令 参数 参数..’格式，不能接受<> |&等特殊字符，很多bash语法也不支持，如果想要支持bash语法，需要设置Tyep=oneshot
~~~



## 重新加载 systemd 的配置

在修改了任何服务配置文件后，必须运行此命令来重新加载 systemd 的配置。

~~~bash
systemctl daemon-reload
~~~



## systemctl 管理命令

| 命令                            | 作用                                 |
| :------------------------------ | :----------------------------------- |
| `systemctl start <服务名>`      | 启动一个服务                         |
| `systemctl stop <服务名>`       | 停止一个服务                         |
| `systemctl restart <服务名>`    | **先停再重启**一个服务               |
| `systemctl reload <服务名>`     | **平滑重载**服务的配置（不中断服务） |
| `systemctl status <服务名>`     | 查看服务的运行状态和最新日志         |
| `systemctl enable <服务名>`     | 设置服务开机自动启动                 |
| `systemctl disable <服务名>`    | 禁止服务开机自动启动                 |
| `systemctl is-enabled <服务名>` | 检查服务是否设置了开机启动           |

>**注意**：把 shell 脚本制作成 service 服务，需要脚本具有执行权限。





## nginx 源码安装配置 systemctl 管理

#### 1 配置服务文件

保存如下内容到文件 `/lib/systemd/system/nginx.service`

~~~bash
[Unit]
Description=The NGINX HTTP and reverse proxy server
After=syslog.target network-online.target remote-fs.target nss-lookup.target # 注意这里的服务如syslog.target背后对应的是systemd-journald.service，后面的几个都有自己的对应
Wants=network-online.target
 
[Service]
Type=forking
PIDFile=/run/nginx.pid # 注意该路径不能随便指定,必须与你nginx配置文件中的pid路径吻合才行
ExecStartPre=/usr/sbin/nginx -t  # 这里的路径要换成你自己的命令路径
ExecStart=/usr/sbin/nginx        # 这里的路径要换成你自己的命令路径
ExecReload=/usr/sbin/nginx -s reload  # 这里的路径要换成你自己的命令路径
ExecStop=/bin/kill -s QUIT $MAINPID # 从PIDFile里取出主pid
PrivateTmp=true
 
[Install]
WantedBy=multi-user.target 
~~~



#### 2. 重新加载 systemd 的配置

~~~bash
systemctl daemon-reload
~~~



#### 3. 管理服务

~~~bash
systemctl status nginx
systemctl start nginx
systemctl enable nginx
~~~



#### 4. 排错

使用 `journalctl` 命令查看服务启动日志。

~~~bash
journalctl -xeu nginx
~~~

