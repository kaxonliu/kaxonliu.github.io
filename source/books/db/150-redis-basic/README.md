# Redis 登录和发布订阅

## 设置密码

在配置文件（redis.cnf）中永久设置。

~~~ini
requirepass 123
~~~

在终端交互中临时设置，重启 redis 后失效。

~~~bash
127.0.0.1:6379> config get requirepass
1) "requirepass"
2) ""
127.0.0.1:6379> config set requirepass 12345
OK
~~~

## 客户端密码登录

方式一：直接登录时使用选项 `a` 提供密码。

~~~bash
redis-cli -h 192.168.10.33 -a '12345'
~~~

方式二：进入终端交互环境后使用命令 `auth` 登录。

~~~bash
[root@localhost ~]# redis-cli -h 192.168.10.33 
192.168.10.33:6379> auth 12345
OK
~~~



## 数据库

redis 默认有16个数据库，编号分别是 0-15。数据库之间数据隔离。

进入数据库时使用选项 `-n` 指定编号

~~~bash
[root@localhost ~]# redis-cli  -n 3 -a '12345'
127.0.0.1:6379[3]> 
~~~

终端交互环境中使用 `select` 切换数据库

~~~redis
127.0.0.1:6379> select 10
OK
127.0.0.1:6379[10]> 
~~~





## 发布订阅

redis 支持发布订阅功能，使用简单方便。

比如有一个名为 `chat` 的频道，订阅者订阅这个频道，发布者向这个频道发布内容。只要一发布新内容，所有的订阅者就可以收到新消息。

订阅者执行 `SUBSCRIBE chat` 等到接收消息。

发布者执行 `PUBLISH chat "hello world"` 向所有的订阅者发送消息。





