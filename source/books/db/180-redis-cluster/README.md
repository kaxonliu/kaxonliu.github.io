# Redis 集群

单台 Redis 存在性能上限和高可用缺陷等问题。需要配合主从、哨兵模式、集群模式实现高可用、提高服务性能。



## Redis 主从模式

主从模式，指的就是主库和从库模式。主库负责写数据，从库负责读数据。一个主库可以有多个从库。

主从模式刚建立时会进行全量同步数据，然后进行增量同步。

主从模式仅仅负责主从数据同步，这种模式没有高可用、故障修复自动切换等功能。



#### 部署主从模式

##### 1. 规划服务器

- 主库：192.168.10.33:6379
- 从库1：192.168.10.102:6379
- 从库2：192.168.10.103:6379

##### 2. 部署 Redis 服务

每个服务器上都安装相同版本的 Redis 服务。

主库使用如下配置

~~~bash
cat > /usr/local/redis/conf/redis.conf << 'EOF'
daemonize yes
bind 0.0.0.0
port 6379
 
masterauth 123456
requirepass 123456 

EOF
~~~

其中，如下两个密码配置建议保持一致。

- `masterauth` 配置从库连结主库需要的密码；理论上只需要给主库配置，但是主库后期也可能变从库，因此建议主库也配置该参数。
- `requirepass` 是 Redis 服务自己的密码（不论主从）



从库配置文件

~~~bash
cat > /usr/local/redis/conf/redis.conf << 'EOF'
daemonize yes
bind 0.0.0.0
port 6379
requirepass 123456
slave-read-only yes

# 配置主节点的ip和端口
slaveof 192.168.10.33 6379
# 主节点有登录密码，则必须指定
masterauth 123456

EOF
~~~



##### 3. 配置系统服务

每个服务器都配置系统服务，均保持一致。

~~~bash
cat > /lib/systemd/system/redis.service  << "EOF"
[Unit]
Description=Redis
After=network.target

[Service]
Type=forking
ExecStart=/usr/local/redis/bin/redis-server /usr/local/redis/conf/redis.conf
ExecReload=/bin/kill -s HUP $MAINPID
ExecStop=/bin/kill -s QUIT $MAINPID
PrivateTmp=true
 
[Install]
WantedBy=multi-user.target

EOF
~~~

重启加载服务配置

~~~bash
systemctl daemon-reload
~~~

##### 4. 启动服务

~~~bash
systemctl start redis
systemctl enable redis
systemctl status redis
~~~

##### 5. 增加环境变量

~~~bash
cat > /etc/profile.d/redis.sh << "EOF"
export PATH=/usr/local/redis/bin:$PATH
EOF
~~~

当前终端更新环境变量

~~~bash
source etc/profile.d/redis.sh
~~~

##### 6. 检查主从状态

在主从服务器上登录 Redis，输出如下命令，可以看到对应的信息。主库上可以看到角色为 master，挂载的从库信息。从库上可以看到角色是 slave，主库信息。

~~~redis
info replication
~~~

##### 7. 故障修复

主从模式没有自动故障修复功能。遇到主库故障不可用时，需要手动选择一个从库，把它设置成主库，然后把其他从库指向新主库。然后旧主库修复好上线后也要首当指向新主库。

选择新主库。在其中一个从库上执行 `slaveof on one` 即可把它变成主库。

~~~bash
slaveof on one
~~~

其他从库指向新主库。主库修复后也要手动操作指向新主库。

~~~bash
slaveof 192.168.10.102 6379
~~~



## Sentinel 哨兵模式

主从模式的核心缺点是无法高可用，主要体现在主库挂点后，系统无法故障修复，无法自当切换出一个新主库。

哨兵模式就是为了解决主从模式这个问题。

哨兵模式也需要一个集群，集群中主机的格式需要是奇数个。

每个哨兵节点都需要安装 Redis 服务，但是哨兵节点不参与 Redis 集群读写数据，哨兵集群负责监控 Redis 集群中主库和每个从库的状态。当主库挂掉后，负责选择一个新主库，并自动完成其余从库指向新主库。



#### 哨兵模式的工作原理

哨兵是一个分布式系统，一个哨兵集群中可以有多个哨兵进程，一般是奇数个。一个哨兵进程就是一个 Redis 进程，只不过这个进程只负责监控任务不负责读写。哨兵进程之间使用 Gossip 协议（流言协议）来通信判断主节点是否下线，判断依据是心跳检测机制。使用投票协议来决定哪个哨兵节点负责执行自动故障修复迁移的任务。

心跳检测机制。哨兵监控每个一主节点和从节点。在创建哨兵集群时需要指定 Redis 集群的主库主库 ip + port，从这个信息中，哨兵可以知道整个集群的主从节点信息。然后采用 ping/pong的模式对每个节点做心跳检测。

判断主节点是否下线。一个哨兵判断主节点下线属于主观下线，需要半数以上的哨兵都认为主节点下线，此时主节点才算是客观下线。主观判断下线的标准就是主节点恢复心跳检测的时间超过 `down-after-milliseconds` 配置的阈值（默认 30s）。

基于 Raft 协议选举哨兵Leader。选举出来的哨兵进程负责执行故障转移任务。从多个从节点中选择出来一个最合适的作为新主，其余从节点指向新主。

整个故障转移自动完成，并且转移之后，哨兵也会修改每个主从节点的配置信息。

哨兵模式是基于主从模式的，它解决了主从模式无法自当修复，自动故障转移的问题。但是它无法突破单台 Redis 内存容量限制的问题。



#### 部署哨兵模式

##### 1. 规划集群

主从集群

- 主库：192.168.10.33:6379
- 从库1：192.168.10.102:6379
- 从库2：192.168.10.103:6379

哨兵集群

- 192.168.10.33:26379
- 192.168.10.102:26379
- 192.168.10.103:26379

生产环境建议哨兵集群单独部署在独立的机器上。



##### 2. 哨兵集群每个节点都要安装 Redis，然后每个哨兵节点上安装的配置文件保持一致。

配置文件内容如下

~~~bash
cat > /usr/local/redis/conf/sentinel.conf << 'EOF'
 
#1、哨兵默认端口
protected-mode no
bind 0.0.0.0
port 26379
 
#2、常规配置：目录要创建好
daemonize yes
pidfile /var/run/redis-sentinel.pid
loglevel notice
logfile "/soft/sentinel/sentinel.log"
dir "/soft/sentinel/data"
 
#3、表示配置哨兵，有2个哨兵作出同样的决策，才有决策权
sentinel monitor mymaster 192.168.10.33 6379 2
 
#4、登录密码
sentinel auth-pass mymaster 123456
 
#5、master被sentinel认定为失效的间隔时间
sentinel down-after-milliseconds mymaster 30000
 
#6、剩余的slaves重新和新的master做同步的并行个数
sentinel parallel-syncs mymaster 2
 
#7、主备切换的超时时间，如果哨兵老大操作超时，交由其他哨兵完成
sentinel failover-timeout mymaster 180000
 
#8、其他配置
acllog-max-len 128
sentinel deny-scripts-reconfig yes
SENTINEL resolve-hostnames no
SENTINEL announce-hostnames no
SENTINEL master-reboot-down-after-period mymaster 0
 
EOF
~~~



##### 3. 创建目录

~~~bash
mkdir -p /soft/sentinel/
mkdir -p /soft/sentinel/data
~~~



##### 4. 启动哨兵服务

~~~bash
# 删除旧哨兵服务
pkill -9 redis-sentinel
redis-sentinel /usr/local/redis/conf/sentinel.conf
~~~



##### 5. 查看端口状态

~~~bash
netstat -an |grep 26379
~~~

##### 6. 查看哨兵日志

可以停掉主库做测试，会发现 30s 就会判定 ok 完成自动选主。

~~~bash
tail -f /soft/sentinel/sentinel.log
~~~





## Cluster 集群模式

集群模式中主库负责读写数据，从库负责备份。一个主库下面可以挂在多个从库。

如果主库挂了，从它下面的从库中自动选择一个当出库，其余从库指向新主库。

一个集群中，有多个主库。如果集群中半数以上的主库挂掉了，则整个集群不可用。

集群模式，可以扩容整个 Redis 服务的内存容量，完全去中心化，多主多从，自动化故障转移。

集群模式下，每个读写都会达到主库，集群会根据 KEY 的哈希值把读写分配到指定的主库上，即使访问一个从库，也会重定向到一个主库上。



#### 集群模式的工作原理

集群模式通过数据分片和分布式存储实现了负载均衡和好可用。在集群模式下，Redis 将所有的键值对数据分散在多个节点上，每个节点只负责一部分数据，成为槽位。通过对数据分片，集群模式可以突破单个节点的内存限制，实现大规模数据内存存储。

 Redis Cluster 分了 16384 个槽位，每个节点只负责一部分槽位。写 KEY 时，Cluster 会计算 KEY 的哈希值将请求路由到响应的节点。

~~~bash
>>> Performing Cluster Check (using node 192.168.10.81:6379)
M: 96d61061c49f5c3c6ad0326964a1ed5f69934192 192.168.10.81:6379
   slots:[0-5460] (5461 slots) master
   1 additional replica(s)
M: cbb07c3e6690092479468d8fac5ba9ba254b605e 192.168.10.82:6379
   slots:[5461-10922] (5462 slots) master
   1 additional replica(s)
M: 5bbecf31803959f5a68ff8cbfe7d2253c8e5b32e 192.168.10.83:6379
   slots:[10923-16383] (5461 slots) master
   1 additional replica(s)
~~~

对客户端来说，真个集群当作一个整体，客户端可以连接任意一个节点进行读写操作。当客户端操作的 KEY 没有分配到该节点时，Redis 会返回转向指令，执行正确的节点。

~~~bash
192.168.10.86:6379> keys *
1) "name"
192.168.10.86:6379> get name
-> Redirected to slot [5798] located at 192.168.10.82:6379
"liuxu"
192.168.10.82:6379> get gender
-> Redirected to slot [15355] located at 192.168.10.83:6379
"male"
192.168.10.83:6379> 
~~~



#### 部署 Cluster 集群

##### 1. 规划集群

三个主库

- 主库1：192.168.10.81:6379
- 主库2：192.168.10.82:6379
- 主库3：192.168.10.83:6379

三个从库

- 从库1：192.168.10.84:6379
- 从库2：192.168.10.85:6379
- 从库3：192.168.10.86:6379



##### 2. 配置集群

集群中每个主机都使用相同的配置。使用相同的系统服务配置，做好环境变量。

~~~bash
cat > /usr/local/redis/conf/redis.conf << "EOF"
daemonize yes
bind 0.0.0.0
port 6379
protected-mode no

# 开启集群模式
cluster-enabled yes
# 节点超时时间
cluster-node-timeout 15000

EOF
~~~

##### 3. 开启 Redis 服务

开启服务后，每个 Redis 服务都还是独立的。

~~~bash
systemctl start redis
~~~



##### 4. 创建集群

使用如下命令将6个 Redis 服务添加到一个集群中，执行完成后自动生产集群配置的配置文件 `/nodes.conf `

~~~bash
redis-cli --cluster create \
  192.168.10.81:6379 \
  192.168.10.82:6379 \
  192.168.10.83:6379 \
  192.168.10.84:6379 \
  192.168.10.85:6379 \
  192.168.10.86:6379 \
  --cluster-replicas 1
~~~

参数解释：

- `--cluster-replicas 1`：表示每个主节点有1个从节点
- 前3个节点会被自动设置为主节点，后3个节点作为从节点





##### 5. 验证测试

~~~bash
# 1、登录集群：选择任一节点
# -c，使用集群方式登录，有密码的话就在加上参数-a 123456
redis-cli -c -h 192.168.10.81 -p 6379 
~~~

登录集群后，查看集群信息

~~~redis
# 查看集群信息
CLUSTER INFO

# 列出节点信息
CLUSTER NODES
~~~





