# 备份和恢复

无论何种备份，最好把 binlog 日志的路径从 `datadir` 中移除，方便后面数据恢复。

备份的目的是为了数据恢复，整体的指导思路为：使用备份数据恢复大部分数据，再使用 binlog 恢复从备份时间点到宕机的数据。



## CP 备份与恢复

cp 备份为全量备份。简单粗暴。

~~~bash
mkdir /backups
cp -a /var/lib/mysql/* /backups
~~~

恢复数据

~~~bash
mv /backups/* /var/lib/mysql/
systemctl restart mysqld
~~~





## mysqldump 备份与恢复

#### Mysqldump 命令基本使用

~~~bash
# 基本语法
# mysqldump  -h 服务器  -u用户名  -p密码  选项与参数 > 备份文件.sql
 
===选项与参数
1、-A/--all-databases             所有库
2、-B/--databases bbs db1 db2     多个数据库
3、db1                          数据库名
4、db1 t1 t2                 db1数据库的表t1、t2
5、-F                           备份的同时刷新binlog
6、-R 备份存储过程和函数数据（如果开发写了函数和存储过程，就备，没写就不备）
7、--triggers 备份触发器数据（现在都是开发写触发器）
8、-E/--events 备份事件调度器
9、-d 仅表结构
10、-t 仅数据
11、--master-data=2  这个选项会将当前 binlog 的文件名和位置（Position）以注释的形式写入到 dump 文件的头部。恢复时我们需要这个信息。=2 表示以注释形式写入，不会在导入时执行。
12、--lock-all-tables 备份过程中所有表从头锁到尾，简单粗暴
13、--single-transaction： 快照备份 （搭配--master-data可以做到热备）
~~~

热备份完整语句

~~~bash
mysqldump -uroot -p -A -E -R --triggers --master-data=2 \
--single-transaction > /backups/full.sql
~~~

导出数据压缩文件

~~~bash
mysqldump -uroot -p -A -E -R --triggers --master-data=2 \
--single-transaction | gzip > /tmp/full$(date +%F).sql.gz
~~~

倒入解压文件

~~~bash
zcat /backup/full$(date +%F).sql.gz | mysql -uroot -p
~~~



#### 备份和恢复数据流程

1. 全量备份。

~~~bash
# bash
mysqldump -uroot -p -A -R --triggers --master-data=2 \
--single-transaction | gzip > /backups/full.sql.gz
~~~

参数 `--master-data` 即将废弃，推荐使用 `--source-data`。

2. 数据备份之后，刷新 binlog，方便日后查找。

~~~bash
# bash
mysql -uroot -p -e "flush logs"
~~~

3. 恢复数据前在当前终端关闭 binlog 记录

~~~sql
-- sql
set sql_log_bin=0;
~~~

4. 先恢复全量数据

~~~sql
-- sql
system zcat /backups/full.sql.gz | mysql -u root -p
~~~

5. 查询确认 binlog 增量数据的起始位置，并保存数据

~~~bash
# bash
mysqlbinlog /var/log/mysql/binlog.000001 --start-position=100 \
--stop-position=500 > /backups/last_bin.log
~~~

6. 恢复 binlog 数据

~~~sql
-- sql
source /backups/last_bin.log
~~~

7. 开始记录 binlog

~~~sql
-- sql
set sql_log_bin=1;
~~~



其中：步骤 5 和 6 可以合并一块执行。并且可以同时指定多个 binlog 文件，工具会按顺序处理。

~~~bash
# bash
mysqlbinlog --no-defaults \
  --start-position=154 \
  --stop-datetime="2024-10-27 14:50:00" \
  /var/log/mysql/binlog.000001 /var/log/mysql/binlog.000002  \
  | mysql -u root -p
~~~





## 逻辑卷备份和恢复

