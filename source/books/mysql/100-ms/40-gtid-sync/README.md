# GTID 复制

在 MySQL 8 中使用 GTID 模式配置主从同步是目前推荐的最佳实践。GTID 极大地简化了复制管理和故障恢复。GTID 复制的优势

- **一致性保障**：每个事务都有一个全局唯一的 ID，保证了主从数据的一致性。
- **故障切换简化**：无需再通过 `MASTER_LOG_FILE` 和 `MASTER_LOG_POS` 来定位同步点，CHANGE MASTER TO 命令更加简单可靠。
- **高可用基础**：是构建 MHA 等高可用方案的基础。





## 第一步：配置主库

#### 1. 配置文件

配置文件做如下修改，主要是开启GTID模式并强制GTID一致性，保证事务安全。保存配置然后重启服务。

~~~ini
[mysqld]
server-id = 1
log-bin = mysql-bin
binlog_format = ROW

# 开启GTID模式
# 强制GTID一致性，保证事务安全
gtid_mode = ON
enforce-gtid-consistency = ON

# 跳过域名解析（非必须）
skip-name-resolve
~~~



#### 2. 创建复制账号

登录主库的 MySQL，创建一个专门用于复制的用户。

~~~sql
-- sql
CREATE USER 'repl'@'%' IDENTIFIED BY 'YourStrongPassword123!';
GRANT REPLICATION SLAVE ON *.* TO 'repl'@'%';
FLUSH PRIVILEGES;
~~~

>如果想允许一个网段，可以使用 `'repl'@'192.168.1.%'`。



#### 3. 直接查询当前 GTID 位置

```sql
mysql> SHOW MASTER STATUS\G
*************************** 1. row ***************************
             File: mysql-bin.000001
         Position: 1738
     Binlog_Do_DB:
 Binlog_Ignore_DB:
Executed_Gtid_Set: afaddb8b-9c34-11f0-a600-000c29e921e0:1-7
1 row in set (0.01 sec)
```

输出会包含 `Executed_Gtid_Set`，记录下这个值，在配置从库时会用到。



## 第二步：配置从库

#### 1. 配置文件

修改从服务器的配置文件，然后重启服务。

~~~ini
[mysqld]
# 服务器唯一ID，必须与主库和其他从库不同
server-id = 2
# 启用中继日志
relay-log = mysql-relay-bin
# 开启GTID模式
gtid_mode = ON
# 强制GTID一致性
enforce-gtid-consistency = ON
# 将从库应用的重做日志也记录到自己的二进制日志中
# 如果该从库还会作为其他服务器的主库，则需要开启
log-slave-updates = ON
# 设置从库为只读，防止误操作（具有SUPER权限的用户仍可写）
read_only = ON
skip-name-resolve # 跳过域名解析（非必须）
relay_log_purge = 0 # 关闭relay_log自动清除功能，保障故障时的数据一致
~~~



#### 2. 配置主库连接信息

~~~sql
-- sql
-- 如果是新的从库，可以跳过这一步
STOP SLAVE;

CHANGE MASTER TO
MASTER_HOST = '主库ip',
MASTER_USER = 'repl',
MASTER_PASSWORD = 'YourStrongPassword123!',
MASTER_AUTO_POSITION = 1; -- 关键！启用GTID自动定位
~~~

>`MASTER_AUTO_POSITION = 1` 告诉从库使用 GTID 来自动寻找同步点，无需手动指定文件和位置。



#### 3. 启动复制进程

~~~sql
--sql
start slave;
show slave status\G
~~~

查看输出结果，**重点关注以下两个字段**：

- `Slave_IO_Running`: `Yes`
- `Slave_SQL_Running`: `Yes`
- `Seconds_Behind_Master`: `0` (表示已完全同步，刚开始可能不为0，稍等片刻)
- 在 `Retrieved_Gtid_Set` 和 `Executed_Gtid_Set` 中可以看到正在同步的 GTID 集合。

>~~~sql
>Retrieved_Gtid_Set: afaddb8b-9c34-11f0-a600-000c29e921e0:1-7
>Executed_Gtid_Set: afaddb8b-9c34-11f0-a600-000c29e921e0:1-7
>~~~





done





~~~
CHANGE MASTER TO
MASTER_HOST = '192.168.10.18',
MASTER_USER = 'repl',
MASTER_PASSWORD = '123',
MASTER_AUTO_POSITION = 1; -- 关键！启用GTID自动定位
~~~

