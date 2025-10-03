# 复制过滤

在MySQL 8中配置主从同步的黑名单和白名单，主要是在从库上设置复制过滤规则，这样可以灵活控制需要同步或忽略的数据库或表。



下面的表格汇总了主要的过滤复制参数

| 过滤类型   | 配置级别 | 参数名称                      | 功能说明                            |
| :--------- | :------- | :---------------------------- | :---------------------------------- |
| **白名单** | 库       | `replicate_do_db`             | 只同步指定的数据库                  |
|            | 表       | `replicate_do_table`          | 只同步指定的表）                    |
|            | 通配符   | `replicate_wild_do_table`     | 使用通配符（如`%`）匹配要同步的表名 |
| **黑名单** | 库       | `replicate_ignore_db`         | 忽略指定的数据库，不进行同步        |
|            | 表       | `replicate_ignore_table`      | 忽略指定的表，不进行同步            |
|            | 通配符   | `replicate_wild_ignore_table` | 使用通配符匹配要忽略的表名          |



## 方式一：配置文件（永久）

此方法需修改MySQL配置文件，重启从库MySQL服务后生效，配置会一直保留。

白名单

```ini
[mysqld]
# 只同步 `db1` 和 `db2` 数据库
replicate_do_db = db1
replicate_do_db = db2
```

黑名单

```ini
[mysqld]
# 忽略 `temp_log` 表和 `backup` 数据库下所有以 `bak_` 开头的表
replicate_ignore_table = operational_data.temp_log
replicate_wild_ignore_table = backup.bak_%
```

#### 



## 方法二：在线动态配置（临时）

使用 `CHANGE REPLICATION FILTER` 语句可以在主从运行期间动态设置过滤规则，但**数据库重启后这些设置会丢失**。



#### 1. 在从库上暂停 SQL 线程

```sql
-- sql
STOP SLAVE SQL_THREAD;
```



#### 2. 设置复制过滤规则

```sql
-- 只同步 db1 库和 db2 库
CHANGE REPLICATION FILTER 
		REPLICATE_DO_DB = (db1, db2);

-- 不同步 db3库和db4库下的t4表
CHANGE REPLICATION FILTER
 	 REPLICATE_IGNORE_DB = (db3);
CHANGE REPLICATION FILTER REPLICATE_IGNORE_TABLE = (db4.t4);
```



#### 3. 重新启动SQL线程：

```sql
START SLAVE SQL_THREAD;
```