# 异步复制

配置异步主从同步有两种场景。分别是停止主库配置主从、主库不停配置主从。配置主从主要目的是为了读写分离，这种场景尽可能最求从库和主库的数据没有延迟。但是有一种情况例外，那就是延迟从库，配置延迟从库的目的是为了备份数据，为了数据安全。比如主库执行了一个删库的指令，因为延迟从库的存在，那就有时间差，可以在延迟从库上备份数据然后恢复到主库。



## 停止主库配置主从

主库不在线，停服配置主从同步。



### 环境假设

- 主库 IP: `192.168.10.15`
- 从库  IP: `192.168.10.16`
- MySQL 版本： mysql-8.0.41-2.el9_5.aarch64
- 操作系统： rockylinux9



### 第一步：主库配置

#### 1. 修改主库配置文件

~~~ini
# /etc/my.cnf

[mysqld]
# 启用二进制日志，这是复制的基础
server-id = 1                # 必须唯一，主库通常设为1
log-bin = mysql-bin          # 二进制日志文件的前缀名
binlog_format = ROW          # 推荐使用ROW格式，数据一致性更好
expire_logs_days = 7         # 自动清理7天前的二进制日志，可选但建议设置
max_binlog_size = 100M       # 每个二进制日志文件的最大大小，可选
sync_binlog = 1		         # 刷盘策略 值为1最安全
binlog_cache_size = 4m
max_binlog_cache_size = 512m
~~~

#### 2. 重启主库 MySQL 服务

~~~bash
systemctl start mysqld
~~~

#### 3. 创建用于复制的专用用户

登录到主库的 MySQL，创建一个专门用于从库连接和复制的用户。

~~~sql
-- 创建用户 ‘repl'，并允许从 ‘192.168.10.16’ 这个从库 IP 地址连接
create user 'repl'@'192.168.10.16' IDENTIFIED by '123';

-- 授予复制权限
GRANT REPLICATION SLAVE ON *.* TO 'repl'@'192.168.10.16';

-- 刷新权限
FLUSH PRIVILEGES;
~~~

#### 4. 导出历史数据

如果主库有历史数据，最好一次性导出然后在从库导入。

~~~bash
mysqldump -uroot -p -A -E -R --triggers --triggers --master-data=2 --single-transaction > /tmp/all.sql
~~~

#### 5. 查看主库状态

执行以下命令，记录下返回结果中的 `File` 和 `Position` 的值，**在配置从库时会用到**。

~~~sql
mysql> show master status;
+------------------+----------+--------------+------------------+-------------------+
| File             | Position | Binlog_Do_DB | Binlog_Ignore_DB | Executed_Gtid_Set |
+------------------+----------+--------------+------------------+-------------------+
| mysql-bin.000002 |     1530 |              |                  |                   |
+------------------+----------+--------------+------------------+-------------------+
~~~



### 第二步：从库配置

#### 1. 修改从库配置文件

~~~ini
# /etc/my.cnf

[mysqld]
server-id = 2                # 必须唯一，且不能与主库相同
relay-log = mysql-relay-bin  # 中继日志文件的前缀名
read_only = ON               # 设置从库为只读模式，防止意外写入


# 禁用从库的binlog功能
# 从库也可以开启binlog功能，但是通常关闭。
skip-log-bin
~~~

#### 2. 重启从库 MySQL 服务

~~~bash
systemctl restart mysqld
~~~

#### 3. 导入数据

拷贝主库的基础数据，然后导入到从库。

~~~bash
# 拷贝数据文件
scp root@192.168.10.15:/tmp/all.sql /tmp/all.sql

# 导入
mysql -u root -p < /tmp/all.sql
~~~

#### 4. 测试同步主库账号

测试主库创建的同步账号是会否可以正常连接到主库。

~~~bash
mysql -u repl -h 192.168.10.15 -p
~~~

#### 5. 配置从库连接主库

第一步登陆到从库

~~~bash
mysql -u root -p
~~~

第二步执行如下命令

~~~sql
-- sql
CHANGE MASTER TO
MASTER_HOST='192.168.10.15',            -- 主库的IP地址
MASTER_USER='repl',                     -- 在主库创建的复制用户名
MASTER_PASSWORD='123', 					  -- 复制用户的密码
MASTER_LOG_FILE='mysql-bin.000002',     -- 主库 SHOW MASTER STATUS 得到的 File
MASTER_LOG_POS=1530;    
~~~

#### 6. 启动从库复制进程

~~~sql
-- sql
START SLAVE;
~~~

#### 7. 检查从库复制状态

使用以下命令检查复制是否正常运行：

```sql
-- sql
SHOW SLAVE STATUS\G
```

**关键检查项（\G 使输出以行显示，更易读）：**

- `Slave_IO_State`: 显示等待主库发送事件等状态，表示 IO 线程正常。
- `Slave_IO_Running`: **必须为 `Yes`**。表示 IO 线程（负责接收二进制日志）已启动并运行。
- `Slave_SQL_Running`: **必须为 `Yes`**。表示 SQL 线程（负责执行中继日志）已启动并运行。
- `Seconds_Behind_Master`: **主从延迟的秒数**。为 `0` 表示完全同步，非 `0` 是正常延迟。
- `Last_IO_Error` 和 `Last_SQL_Error`: 错误信息。如果复制有问题，这里会显示详细的错误原因。

如果 `Slave_IO_Running` 和 `Slave_SQL_Running` 都是 `Yes`，并且没有错误，恭喜你，主从复制已经配置成功！



### 第三步：测试主从同步

1. 在主库上创建一个新数据库或表，或者插入一条数据。
2. 在从库上查询，检查数据是否已经同步过来。
3. 如果能查询到在主库插入的数据，说明同步成功。





## 主库在线配置主从

主库在线不停服，持续有客户端连接主库写入数据，不停服的情况下配置主从同步。



### 核心原理

通过备份主库的**某一时刻的数据快照**，并记录该时刻对应的二进制日志位置点。从库应用此快照后，再从记录的位置点开始，持续地从主库获取新的二进制日志事件并进行重放，从而实现数据同步。



### 配置流程

配置流程的核心与主库停服时的情况几乎一致，再次只记录不同之处。

#### 1. 在主库上创建数据一致性快照

使用 `mysqldump` 命令在主库服务器上执行备份。**核心参数是 `--master-data`**，它会自动在备份文件中记录 `SHOW MASTER STATUS` 的位置点。

```bash
mysqldump -u root -p -A --master-data=2 --single-transaction --flush-logs > /tmp/all.sql
```

- `-A`: 备份所有数据库。
- `--master-data=2`:（默认是1，我们常用2）这个参数会将主库的 `File` 和 `Position` 信息以 **CHANGE MASTER TO** 命令的形式注释在备份文件的开头，并且位置点是**备份开始时的一致性位置**。
- `--single-transaction`: 对 InnoDB 表进行一致性备份，确保数据快照的一致性，而不需要锁整个库（对于MyISAM表无效）。这是实现“热备份”的关键。
- `--flush-logs`: 备份完成后刷新日志，方便后续的日志管理。



#### 2. 在主库上把备份文件传输到从库

~~~bash
scp /tmp/all.sql root@192.168.10.16:/tmp/
~~~



#### 3. 从库配置

配置如下。然后重启服务。

~~~ini
[mysqld]
server-id = 2          
relay-log = mysql-relay-bin 
read_only = ON  
skip-log-bin
~~~

#### 4. 在从库上导入数据

在从库服务器上，将备份数据导入：

```bash
mysql -u root -p < /tmp/all.sql
```

#### 5. 从备份文件中获取 binlog 文件和位置

~~~bash
[root@rocky ~]# head -50 /tmp/all.sql|grep 'MASTER_LOG_POS'
-- CHANGE MASTER TO MASTER_LOG_FILE='mysql-bin.000003', MASTER_LOG_POS=157;
~~~

#### 6. 在从库配置配置主从数据

~~~sql
-- sql
CHANGE MASTER TO
MASTER_HOST='192.168.10.15',            -- 主库的IP地址
MASTER_USER='repl',                     -- 在主库创建的复制用户名
MASTER_PASSWORD='123', 							    -- 复制用户的密码
MASTER_LOG_FILE='mysql-bin.000003',     -- 停止同步时从库同步到的文件
MASTER_LOG_POS=148629;                  -- 停止同步时从库同步到的位置
~~~

#### 7. 在从库开启同步

~~~sql
-- 开启同步
START SLAVE;

-- 查看状态
SHOW SLAVE STATUS\G
~~~





## 延迟从库

用延时从库可以做备份，主库执行删除的时候，从库还没有删除，可以把表数据拿出来恢复回去。企业中一般会延时 3-6 小时。

延时从库是在 SQL 线程做的手脚，IO 线程已经把数据放到 relay-log 里了。SQL 线程在执行的时候，会延迟你设定的时间长度。



延迟从库本质上也是从库，只不过多一个配置延迟的字段。如果已经配置了主从，那修改为延迟从库的配置流程如下。

#### 1. 从库停止 SQL 线程

~~~sql
-- sql
stop slave;
~~~

#### 2. 设置延时时间

完整配置如下

~~~sql
-- sql
change master to
master_host='192.168.10.15',
master_user='repl',
master_password='123',
master_log_file='mysql-bin.000001',
master_log_pos=2752,
master_delay=180;
~~~

在基本从库配置的基础上可以增加延时配置，可以使用如下命令。

~~~sql
-- MySQL 8.0+ 语法
CHANGE REPLICATION SOURCE TO SOURCE_DELAY = 3600;

-- MySQL 5.7 及以前语法
CHANGE MASTER TO MASTER_DELAY = 3600;
~~~

#### 3. 开始延时从库

~~~sql
-- sql
start slave;
~~~

