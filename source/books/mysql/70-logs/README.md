# 日志

MySQL 中多有种类型的日志。错误日志、普通日志、慢日志（slow log）、二进制日志（binlog）。



## 错误日志

MySQL 错误日志记录了 MySQL 服务器运行时发生的各种事件，包括错误、警告、以及一些关键的系统状态信息。

查看错误日志的默认路径。可以看出，错误日志被自定义到了 `/var/log/mysql/error.log`。

~~~sql
mysql> SHOW VARIABLES LIKE 'log_error';
+---------------+---------------------------+
| Variable_name | Value                     |
+---------------+---------------------------+
| log_error     | /var/log/mysql/mysqld.log |
+---------------+---------------------------+
~~~



#### 配置错误日志

从 MySQL 5.7 开始，错误日志的配置有了显著变化，引入了更灵活的组件架构。你可以选择将错误日志输出到文件或系统日志（如 Linux 的 syslog）。

配置参数 `log_error`： 定义错误日志的输出目标，通常值为文件名表示输出到文件。

配置参数 `log_error_services`： 控制错误日志事件的处理流程。它是一组由分号分隔的组件列表。

- 默认值通常是 `log_filter_internal; log_sink_internal` 或 `log_filter_internal; log_sink_json`。
- `log_filter_internal`： 内置的过滤组件，根据日志级别进行过滤。
- `log_sink_internal`： 传统的错误日志格式输出组件。
- `log_sink_json`： 将错误日志以 JSON 格式输出的组件。
- `log_sink_syseventlog`： （Windows）输出到 Windows 事件日志。
- `log_sink_syslog`： （Linux）输出到系统的 syslog。

配置参数 `log_warning`：是 mysql5.6 上的参数。控制警告信息是否记录到错误日志。

- 值为 0 表示不记录警告信息。
- 值为 1 表示启用警告记录。
- 值为 2 表示更加详细的记录警告信息。

配置参数 `log_error_verbosity` ：是 mysql5.7 引入了一个新的参数用来替代`log_warning`。它的值有三个

- 值为 1，表示记录错误信息。
- 值为 2，表示记录错误信息和警告信息（推荐）。
- 值为 3，表示记录错误信息、警告信息和通知信息。

~~~sql
mysql> SHOW VARIABLES LIKE 'log_error_verbosity';
+---------------------+-------+
| Variable_name       | Value |
+---------------------+-------+
| log_error_verbosity | 2     |
+---------------------+-------+
~~~



**配置示例**

把错误日志以 JSON 格式输出到文件 `/var/log/mysql/error-log.json`。配置参数如下，然后重启服务。

~~~ini
# /etc/my.cnf
[mysqld]
log_error = /var/log/mysql/error-log.json
log_error_services = 'log_filter_internal; log_sink_json'
~~~



## 通用查询日志

通用查询日志是一个记录了 MySQL 服务器接收到的每一个客户端连接和执行的每一条 SQL 语句的文本文件。

检查通用日志是否已经开启以及日志文件的路径。`general_log`参数的值为 OFF 表示未开启通用日志。一般默认不开启（消耗性能）。

~~~sql
mysql> SHOW VARIABLES LIKE 'general_log%';
+------------------+---------------------------+
| Variable_name    | Value                     |
+------------------+---------------------------+
| general_log      | OFF                       |
| general_log_file | /var/lib/mysql/centos.log |
+------------------+---------------------------+
~~~



配置通用查询日志。修改保存后，需要重启 MySQL 服务使配置生效。

~~~ini
# /etc/my.cnf
[mysqld]
general_log = 1                  # 1 表示开启，0 表示关闭
general_log_file = /path/to/your/general.log  # 指定自定义日志路径和文件名
log_output = FILE                # 指定输出方式
~~~

也可以动态设置。临时生效，重启后失效。在 MySQL 命令行中执行：

```sql
mysql> SET GLOBAL general_log = 'ON';
mysql> SET GLOBAL general_log = 'OFF';
```



## 慢查询日志

慢查询日志是 MySQL 提供的一种日志记录，用于记录执行时间超过指定阈值（`long_query_time`）的 SQL 语句。默认情况下，这个阈值是 10 秒。此外，它还可以记录没有使用索引的 SQL 语句（通过 `log_queries_not_using_indexes` 参数控制）。



#### 永久配置

配置参数如下。保存配置文件并重启 MySQL 服务使配置生效。

~~~ini
# /etc/my.cnf

[mysqld]
# 启用慢查询日志，1 为开启，0 为关闭
slow_query_log = 1

# 指定慢查询日志文件的路径和名称
slow_query_log_file = /var/lib/mysql/mysql-slow.log

# 定义慢查询的阈值时间（单位：秒），超过此时间的 SQL 会被记录
long_query_time = 2

# 记录未使用索引的查询（可选，但开启后日志量可能剧增，需谨慎）
log_queries_not_using_indexes = 1

# 限制每分钟记录未使用索引的 SQL 次数，防止日志爆炸（可选）
# 需要 log_queries_not_using_indexes=1
log_throttle_queries_not_using_indexes = 10

# 记录管理语句（如 ALTER TABLE, ANALYZE TABLE等）
log_slow_admin_statements = 1

# 记录执行较慢的从库复制线程的查询
log_slow_slave_statements = 1
~~~



#### 分析慢日志工具

使用 `mysqldumpslow`（MySQL 官方工具）。这个工具可以对慢查询日志进行归类、统计和排序，使分析变得更简单。

~~~bash
# 得到返回记录集最多的10个SQL
mysqldumpslow -s r -t 10 /var/lib/mysql/mysql-slow.log

# 得到访问次数最多的10个SQL
mysqldumpslow -s c -t 10 /var/lib/mysql/mysql-slow.log

# 得到按照时间排序的前10条里面含有左连接的SQL
mysqldumpslow -s t -t 10 -g "left join" /var/lib/mysql/mysql-slow.log

# 结合 more 查看，防止爆屏
mysqldumpslow -s t -t 10 /var/lib/mysql/mysql-slow.log | more
~~~

**参数解释：**

- `-s`：排序方式，可选值：
  - `c`：访问次数
  - `l`：锁定时间
  - `r`：返回记录数
  - `t`：查询时间（默认）
  - `al`：平均锁定时间
  - `ar`：平均返回记录数
  - `at`：平均查询时间
- `-t`：返回前面多少条的数据。
- `-g`：后接正则表达式，进行过滤。



使用 `pt-query-digest`（Percona Toolkit 工具，功能强大）。这是 Percona 公司提供的顶级工具，分析功能比 `mysqldumpslow` 更详细、强大。





## 二进制日志 binlog

binlog 是一个由 MySQL 服务器层创建并维护的、记录所有对数据库内容进行更改的“写操作”的日志文件。

- **它记录的是逻辑操作**，例如 `INSERT`、`UPDATE`、`DELETE`、`ALTER TABLE` 等 SQL 语句（取决于日志格式）。
- **它不记录** `SELECT`、`SHOW` 这类不修改数据的查询操作。
- **它是追加写入的**，文件按顺序记录，当文件达到一定大小后，会切换到下一个文件，形成一个文件序列（如 `binlog.000001`, `binlog.000002` ...）。



#### binlog 主要作用

**数据复制**。这是 binlog 最主要的作用。在主从复制（Master-Slave Replication）中，主库（Master）将其 binlog 发送给从库（Slave）。从库读取并重放（Replay）binlog 中的事件，从而使得从库的数据与主库保持同步。这是实现读写分离、负载均衡和高可用性的基础。

**数据恢复**。结合**全量备份**和 **binlog**，可以实现任意时间点的数据恢复。

**审计**。通过解析 binlog，可以查看谁在什么时候对数据库做了什么操作，用于安全审计和问题排查。



#### binlog 的三种记录格式

binlog 记录SQL语句的方式有多种，这由 `binlog_format` 参数控制，主要有三种：

| 格式      | 描述                                                         | 优点                                                         | 缺点                                                         |
| :-------- | :----------------------------------------------------------- | :----------------------------------------------------------- | :----------------------------------------------------------- |
| statement | 记录原始的 SQL 语句本身。                                    | - 日志文件小。 - 写入量少，I/O 性能好。                      | - **主从数据可能不一致**。如果 SQL 中使用了 `UUID()`, `NOW()`, `RAND()` 等非确定性函数，在主库和从库上执行的结果可能不同。 - 对某些特殊语句（如 `LOAD_FILE()`）的复制不友好。 |
| row       | 记录**每一行数据是如何被修改的**。例如，一个 `UPDATE` 语句更新了 10 行，binlog 中会记录这 10 行修改前和修改后的完整数据。 | - **数据一致性最安全**。因为它记录的是行的实际变化，能保证主从数据绝对一致。 - 是目前**推荐的默认方式**（MySQL 5.7.7 及以后）。 | - 日志文件会很大。尤其是批量更新或删除操作。 - 从 binlog 中无法直接看到执行了哪些 SQL 语句，需要工具解析。 |
| mixed     | 混合模式。默认使用 STATEMENT 格式，仅在可能引起主从不一致的情况下（如使用了非确定性函数），自动切换为 ROW 格式。 | - 兼顾了文件大小和数据安全性。                               | - 仍存在极少数边缘情况可能导致不一致。                       |

**建议：** 在追求数据一致性的现代数据库架构中，**强烈建议使用 row 格式**。



#### 常用的 binlog 操作命令

1. 查看是否启用了 binlog

 ```sql
 mysql> SHOW VARIABLES LIKE 'log_bin';
 +---------------+-------+
 | Variable_name | Value |
 +---------------+-------+
 | log_bin       | ON    |
 +---------------+-------+
 ```

2. 查看所有 binlog 文件列表
 ```sql
 mysql> SHOW BINARY LOGS;
 +---------------+-----------+-----------+
 | Log_name      | File_size | Encrypted |
 +---------------+-----------+-----------+
 | binlog.000001 |      4502 | No        |
 | binlog.000002 |    403972 | No        |
 +---------------+-----------+-----------+
 ```

3. 查看当前正在写入的 binlog 文件及位置
 ```sql
 mysql> SHOW MASTER STATUS;
 +---------------+----------+--------------+------------------+-------------------+
 | File          | Position | Binlog_Do_DB | Binlog_Ignore_DB | Executed_Gtid_Set |
 +---------------+----------+--------------+------------------+-------------------+
 | binlog.000002 | 403972   |              |                  |                   |
 +---------------+----------+--------------+------------------+-------------------+
 ```

4. 刷新 binglog，关闭当前二进制文件并创建一个新文件。

~~~sql
flush logs;
~~~

5. 删除所有 binlog 文件，并重新开始编号（谨慎使用！）

```sql
RESET MASTER;
```

6. 删除指定 binlog 文件之前的文件

```sql
-- 删除 000002 之前的所有文件
PURGE BINARY LOGS TO 'binlog.000002';  

-- 删除指定时间之前的文件
PURGE BINARY LOGS BEFORE '2023-10-27 22:46:26'; 
```



#### 常用 binlog 配置参数

```ini
# /etc/my.cnf

[mysqld]
# 启用 binlog，这是必须的
# 指定 binlog 文件的前缀和路径
log_bin = /var/lib/mysql/binlog  

# 服务器唯一ID，在主从复制中至关重要，每个服务器必须不同
server_id = 1

# 设置 binlog 格式，推荐 ROW
binlog_format = ROW

# 设置每个 binlog 文件的最大大小（默认 1GB）
max_binlog_size = 100M

# 设置 binlog 的过期时间（秒），默认 0 表示不过期
# 。也可用 `expire_logs_days`（天）
binlog_expire_logs_seconds = 604800  # 7天 = 7 * 24 * 60 * 60

# 为保证复制和恢复的完整性，建议开启。
# 每次事务提交时，都会将 binlog 刷写到磁盘。
sync_binlog = 1
```

#### sql_log_bin

`sql_log_bin` 是一个用于控制当前数据库会话（Session/Connection）是否将执行的操作记录到二进制日志（binlog）中的开关。设置值为布尔值（`ON` 或 `OFF`）

`SET SESSION sql_log_bin = OFF;` 当前会话执行的修改 SQL 不在记录到 binlog。常用于数据恢复或数据迁移时，避免产生“重复操作”。





#### 查看binlog

查看日志名、状态和事件。

~~~sql
show binary logs;
show master logs;
show master status;
show binlog events in 'binlog.000001';
show binlog events in 'binlog.000001' limit 3;
~~~

查看日志内容

~~~bash
# 查看全部：
mysqlbinlog binlog.000001
 
# 按时间：
mysqlbinlog binlog.000001 --start-datetime="2022-11-05 10:02:56" --stop-datetime="2022-11-05 11:02:54" 
 
# 按字节数：
mysqlbinlog binlog.000001 --start-position=337 --stop-position=662
 
# 如果是行级模式，想要看懂详细内容则需要加上额外参数，
# 但是仅用于看懂内容，如果要用于还原数据，
# 还是应该去掉额外的参数并将内容定位到文件中
# 仅用于查看，不能用于日后的数据恢复
mysqlbinlog --base64-output=decode-rows -vvv binlog.000001  
 
# /tmp/1.sql可用于日后的数据恢复
mysqlbinlog binlog.000001 --start-position=100 > /tmp/1.sql 
~~~

如果查看过程遇到如下报错。原因是 mysql5.6中，mysqlbinlog这个工具无法识别binlog中的配置中的default-character-set=utf8mb4这个配置项目。
~~~bash
unknown variable 'default-character-set=utf8mb4' 
~~~

两个方法可以解决这个问题：

- 在 MySQL 的配置` /etc/my.cnf` 中将` default-character-set=utf8mb4` 修改为 `character-set-server = utf8mb4`，但是这需要重启MySQL服务，如果你的MySQL服务正在忙，那这样的代价会比较大。
- 加参数 `mysqlbinlog --no-defaults binlog.000001` 



