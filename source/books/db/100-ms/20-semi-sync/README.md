# 半同步复制

### 环境假设

- 主库 IP: `192.168.10.15`
- 从库  IP: `192.168.10.16`
- MySQL 版本： mysql-8.0.41-2.el9_5.aarch64
- 操作系统： rockylinux9



### 第一步：配置主库

#### 1.1 编辑 MySQL 配置文件

```ini
[mysqld]
server-id = 1
log-bin = mysql-bin
binlog_format = ROW   
```

#### 1.2 重启 MySQL 服务

```bash
systemctl restart mysqld
```

#### 1.3 创建用于复制的用户

登录 MySQL 并创建一个专门用于从服务器连接和复制的用户。

```bash
mysql -u root -p
```

创建复制用户

```sql
-- 创建复制用户
CREATE USER 'repl'@'192.168.10.16' IDENTIFIED BY '123';
GRANT REPLICATION SLAVE ON *.* TO 'repl'@'192.168.10.16';
FLUSH PRIVILEGES;
```

#### 1.4 安装插件

~~~sql
-- sql
install plugin rpl_semi_sync_master soname 'semisync_master.so';

-- 安装完成后，在plugin表（系统表）中查看一下
select * from mysql.plugin;
~~~

#### 1.5 编辑配置开启半同步

~~~ini
# 半同步复制配置
# 加载半同步主库插件
plugin-load = "rpl_semi_sync_master=semisync_master.so"
# 启用半同步主库（1=启用，0=禁用）
rpl_semi_sync_master_enabled = 1
# 设置主库等待从库响应的超时时间（毫秒）
rpl_semi_sync_master_timeout = 1000
~~~

#### 1.6 重启服务

~~~bash
systemctl restart mysqld
~~~





### 第二步：配置从库

#### 2.1 编辑 MySQL 配置文件

```ini
[mysqld]

server-id = 2
relay-log = mysql-relay-bin
read_only = ON 

# 半同步复制配置
# 加载半同步从库插件
plugin-load = "rpl_semi_sync_slave=semisync_slave.so"
# 启用半同步从库（1=启用，0=禁用）
rpl_semi_sync_slave_enabled = 1
```

#### 2.2 重启 MySQL 服务

```bash
systemctl restart mysqld
```

#### 2.3 安装插件

```sql
-- slq
install plugin rpl_semi_sync_slave soname 'semisync_slave.so';
 
-- 安装完成后，在plugin表（系统表）中查看一下
select * from mysql.plugin;
```

#### 2.4 编辑配置开启半同步

~~~ini
[mysqld]
server-id = 2                # 必须唯一，且不能与主库相同
relay-log = mysql-relay-bin  # 中继日志文件的前缀名
read_only = ON               # 设置从库为只读模式，防止意外写入

# 半同步
rpl_semi_sync_slave_enabled =1
~~~

#### 2.4 重启服务

~~~bash
systemctl restart mysqld
~~~



### 第三步：启动从服务器复制进程

#### 3.1 在从服务器上设置主库信息

登录从服务器的 MySQL，执行以下命令。请将参数替换为您实际的值。

```sql
-- 停止 Slave 线程
STOP SLAVE;

-- 配置主库连接信息
CHANGE MASTER TO
MASTER_HOST='192.168.10.15',      
MASTER_USER='repl',             
MASTER_PASSWORD='123',
MASTER_LOG_FILE='mysql-bin.000003', 
MASTER_LOG_POS=880;     

-- 启动 Slave 线程
START SLAVE;
```

#### 3.2 检查从服务器复制状态

使用以下命令检查 Slave 状态，关键看 `Slave_IO_Running` 和 `Slave_SQL_Running` 是否都为 `Yes`。

```sql
SHOW SLAVE STATUS\G
```



### 第四步：验证半同步复制是否工作

#### 4.1 在主服务器上检查半同步状态

回到主服务器的 MySQL，执行：

```sql
SHOW VARIABLES LIKE 'rpl_semi_sync%';
```

确认 `rpl_semi_sync_master_enabled` 为 `ON`。

```sql
SHOW STATUS LIKE 'Rpl_semi_sync_master_status';
```

如果显示为 `ON`，表示主库的半同步模式已激活。

#### 4.2 在从服务器上检查半同步状态

在从服务器的 MySQL 中执行：

```sql
SHOW VARIABLES LIKE 'rpl_semi_sync%';
```

确认 `rpl_semi_sync_slave_enabled` 为 `ON`。

```sql
SHOW STATUS LIKE 'Rpl_semi_sync_slave_status';
```

如果显示为 `ON`，表示从库的半同步模式已激活。



#### 4.3 功能测试

1. 在主服务器上创建一个新的数据库或表，并插入一些数据。
2. 在从服务器上检查数据是否已同步。
3. 如果能看到数据 ，说明主从复制和半同步功能都正常工作。

