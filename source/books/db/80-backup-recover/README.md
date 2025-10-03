# 备份和恢复

备份类型

- 冷备。停服备份。
- 温备。不停服务。但是锁表。
- 热备。不停服务，不锁表。

备份内容

- 物理备份。直接将底层物理文件备份。
- 逻辑备份。备份 SQL语句或数据。

备份大小

- 全量。每次都是全量。
- 差异。每次都和第一次全量备份做比较。
- 增量。每次备份都和上一次备份做比较。



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
--single-transaction | gzip > /backups/full$(date +%F).sql.gz
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

这种方式备份需要把 MySQL 的 `datadir` 放在逻辑卷上，使用逻辑卷的快照功能快速备份数据。



#### 创建逻辑卷

1. 准备一个新硬盘 `nvme0n2`

~~~bash
[root@rocky dev]# lsblk
NAME        MAJ:MIN RM  SIZE RO TYPE MOUNTPOINTS
sr0          11:0    1  1.9G  0 rom
nvme0n1     259:0    0   10G  0 disk
├─nvme0n1p1 259:1    0  500M  0 part /boot/efi
├─nvme0n1p2 259:2    0    1G  0 part [SWAP]
└─nvme0n1p3 259:3    0  8.5G  0 part /
nvme0n2     259:4    0    2G  0 disk
~~~

2. 把新硬盘制作逻辑卷

~~~bash
# pv
pvcreate /dev/nvme0n2p1

# vg
vgcreate vg1 /dev/nvme0n2

# lv
lvcreate -L 1G -n lv1_from_vg1 vg1
~~~

3. 格式化制作文件系统并挂载

~~~bash
# 格式化文件系统
mkfs.xfs /dev/vg1/lv1_from_vg1

# 挂载
mount /dev/vg1/lv1_from_vg1 /var/lib/mysql
chown -R mysql.mysql /var/lib/mysql

# 查看
[root@rocky dev]# df -h
...
/dev/mapper/vg1-lv1_from_vg1  960M   39M  922M   5% /var/lib/mysql
~~~



#### 配置 MySQL 

配置 mysql 文件指定 `datadir=/var/lib/mysql`。然后清理文件，重启服务。然后准备数据。

~~~ini
# my.cnf

[mysqld]
datadir=/var/lib/mysql
~~~



#### 备份数据

1. 锁定所有表

~~~sql
-- sql
flush tables with read lock;
~~~

2. 创建快照

~~~bash
# bash
lvcreate -L 200M -s -n lv1_from_vg1_snap /dev/vg1/lv1_from_vg1
~~~

3. 解锁表

~~~sql
-- sql
unlock tables;
~~~

备份数据。把快照打包到备份文件，然后即可删除快照。

~~~bash
# bash
mkdir /snap1
mount -o nouuid /dev/vg1/lv1_from_vg1_snap /snap1
cd /snap1
tar cf /backups/full.tar *
umount /snap1/ -l
lvremove /dev/vg1/lv1_from_vg1_snap
~~~



#### 恢复数据

删除数据

~~~bash
rm -rf /var/lib/mysql/*
~~~

恢复数据

~~~bash
tar xf /backups/full.tar -C /var/lib/mysql/
~~~

重启 MySQL

~~~bash
systemctl restart mysqld
~~~





## Xtrabackup 备份

XtraBackup 是一款由 Percona 公司开发的、专为 MySQL 和 MariaDB 数据库设计的开源热备份工具。

XtraBackup 主要由两个核心工具组成：

1. **`xtrabackup`**：用于备份 InnoDB 表的数据文件，这些文件在备份期间可能被修改，`xtrabackup` 会保存这些修改。
2. **`innobackupex`**（**注意：已过时**）：一个封装了 `xtrabackup` 的 Perl 脚本，它除了能处理 InnoDB 表，还能备份 MyISAM 等非事务引擎的表，并统一提供备份和恢复的命令行接口。

**重要提示**：从 XtraBackup 2.4 版本开始，**`innobackupex` 已被弃用**。官方推荐直接使用 `xtrabackup` 命令，它现在集成了所有 `innobackupex` 的功能。新版本的命令使用 `--backup` 等选项来替代旧模式。



#### 安装版本选择

mysql 5.7以下版本，可以采用percona xtrabackup 2.4版本

mysql 8.0以上版本，可以采用percona xtrabackup 8.0版本，xtrabackup8.0也只支持mysql8.0以上的版本。

#### 安装

~~~bash
# 安装 Percona 仓库
yum install https://repo.percona.com/yum/percona-release-latest.noarch.rpm -y
 
 # 启用 Percona 仓库
 percona-release enable-only tools release
 
# 安装XtraBackup命令
yum install percona-xtrabackup-80 -y --nogpgcheck

# 检查版本
xtrabackup --version
~~~



#### Xtrabackup 全量备份

基于 mysql8.0 + xtrabackup8

1. 创建备份目录

~~~bash
mkdir /backups
~~~

2. 执行全量备份

~~~bash
xtrabackup --backup --target-dir=/backups/full \
           --user=root --password=1

# 常用选项说明：
# --backup: 执行备份操作
# --target-dir: 指定备份文件存放的目录
# --user/--password: 连接数据库的用户名和密码（需要有 RELOAD, PROCESS, LOCK TABLES, REPLICATION CLIENT 等权限）
# --socket: 如果不用默认的 /tmp/mysql.sock，可以用 --socket=/path/to/mysql.sock 指定
~~~

3. 准备备份。如果输出最后有类似 `completed OK!` 的字样，说明准备成功。

~~~bash
xtrabackup --prepare --target-dir=/backups/full
~~~

4. 恢复数据

~~~bash
# 1. 停止 MySQL 服务
systemctl stop mysqld

# 2. （可选但强烈推荐）移动或备份旧的数据目录
mv /var/lib/mysql /var/lib/mysql_old_backup
mv /var/log/mysql /var/lib/mysql_old_log_backup

# 3. 确保新的数据目录为空，并创建它
mkdir -p /var/lib/mysql
mkdir -p /var/log/mysql

# 4. 使用 xtrabackup 复制备份文件到数据目录
# --copy-back 会保留文件的原始属性（如所有者、权限）
xtrabackup --copy-back --target-dir=/backups/full

# 或者使用 --move-back，它会移动文件而不是复制，速度更快，但原备份目录将为空。
# xtrabackup --move-back --target-dir=/backups/full

# 5. 确保数据目录的文件权限正确（通常属于 mysql 用户）
chown -R mysql:mysql /var/lib/mysql
chown -R mysql:mysql /var/log/mysql

# 6. 启动 MySQL 服务
systemctl start mysqld
~~~



#### Xtrabackup 增量备份

基于 mysql8.0 + xtrabackup8

第一次全量备份

~~~bash
xtrabackup --backup --target-dir=/backups/full/2025-09-27_full \
           --user=root --password=
~~~

创建第一次增量备份

~~~bash
xtrabackup --backup --target-dir=/backups/inc/2025-09-27_inc1 \
           --incremental-basedir=/backups/full/2025-09-27_full \
           --user=root --password=
~~~

创建第二次增量备份

~~~bash
xtrabackup --backup --target-dir=/backups/inc/2025-09-27_inc2 \
           --incremental-basedir=/backups/inc/2025-09-27_inc1 \
           --user=root --password=
~~~



准备增量备份。将所有增量备份合并到全量备份中。

~~~bash
# 步骤 1: 准备全量备份（使用 --apply-log-only，只重做日志，不回滚，为接收增量做准备）
xtrabackup --prepare --apply-log-only --target-dir=/backups/full/2025-09-27_full

# 步骤 2: 将第一个增量备份合并到全量备份中
xtrabackup --prepare --apply-log-only --target-dir=/backups/full/2025-09-27_full \
           --incremental-dir=/backups/inc/2025-09-27_inc1

# 步骤 3: 将第二个增量备份合并到全量备份中（最后一个增量备份可以不加 --apply-log-only）
xtrabackup --prepare --target-dir=/backups/full/2025-09-27_full \
           --incremental-dir=/backups/inc/2025-09-27_inc2
~~~

恢复数据

~~~bash
# 1. 停止 MySQL 服务
systemctl stop mysqld

# 2. （可选但强烈推荐）移动或备份旧的数据目录
mv /var/lib/mysql /var/lib/mysql_old_backup
mv /var/log/mysql /var/log/mysql_old_log_backup

# 3. 确保新的数据目录为空，并创建它
mkdir -p /var/lib/mysql
mkdir -p /var/log/mysql

# 4. 使用 xtrabackup 复制备份文件到数据目录
# --copy-back 会保留文件的原始属性（如所有者、权限）
xtrabackup --copy-back --target-dir=/backups/full/2025-09-27_full

# 或者使用 --move-back，它会移动文件而不是复制，速度更快，但原备份目录将为空。
# xtrabackup --move-back --target-dir=/backups/full

# 5. 确保数据目录的文件权限正确（通常属于 mysql 用户）
chown -R mysql:mysql /var/lib/mysql
chown -R mysql:mysql /var/log/mysql

# 6. 启动 MySQL 服务
systemctl start mysqld
~~~



## Xtrabackup 增量备份 + binlog 恢复数据

**基于 mysql8.0 + xtrabackup8**

为了清晰说明，我们设定一个备份计划：

- **周日晚上**：进行一次全量备份（Base Full Backup）
- **周一晚上**：基于周日的全量备份，进行一次增量备份（Incr Backup 1）
- **周二晚上**：基于周一的增量备份，再进行一次增量备份（Incr Backup 2）
- **周三上午10:00**：发生误操作，需要将数据恢复到周二晚上（比如周三凌晨）的某个状态。

**工具与路径：**

- **备份工具**：Percona XtraBackup 8.0
- **全量备份目录**：`/backups/full/20241027/`
- **增量备份目录**：`/backups/incr/20241028/`， `/backups/incr/20241029/`
- **Binlog 目录**：`/var/log/mysql`



### 第一步：备份阶段

#### 1.1 周日：创建全量备份（基准备份）

~~~bash
xtrabackup --backup --host=localhost --user=root --password= \
--target-dir=/backups/full/20241027/
~~~

命令执行成功后，会备份所有 InnoDB 表和数据。同时会刷新 binlog，创建一个新编号的 binlog 文件。`/backups/full/20241027/xtrabackup_binlog_info` 这个文件记录了**备份结束时**的 Binlog 位置点，是后续进行 Binlog 恢复的**起点**。如下所示 `binlog.000003` 就是新 binlog 文件。

~~~bash
[root@rocky ~]# cat /backups/full/20241027/xtrabackup_binlog_info
binlog.000003	157
~~~



#### 1.2 周一：创建第一次增量备份

~~~bash
xtrabackup --backup --host=localhost --user=root --password= \
--target-dir=/backups/incr/20241028/ \
--incremental-basedir=/backups/full/20241027/
~~~

命令执行成功后，会备份所有 InnoDB 表和数据。同时会刷新 binlog，创建一个新编号的 binlog 文件。



#### 1.3 周二：创建第二次增量备份

~~~bash
xtrabackup --backup --host=localhost --user=root --password= \
--target-dir=/backups/incr/20241029/ \
--incremental-basedir=/backups/incr/20241028/
~~~

这次备份后，数据库新增了两条数据，这两条数据使用 binlog 恢复。



### 第二步：模拟周三上午发生误操作

~~~bash
systemctl stop mysqld
rm -rf /var/lib/mysql/*
mv /var/log/mysql /var/log/mysql_old_log
~~~



### 第三步：恢复阶段

恢复时必须严格按照备份的顺序进行：**全量 -> 增量1 -> 增量2 -> ...**

#### 2.1 准备（Prepare）全量备份

~~~bash
xtrabackup --prepare --apply-log-only --target-dir=/backups/full/20241027/
~~~

#### 2.2 应用第一次增量备份到全量备份上

~~~bash
xtrabackup --prepare --apply-log-only --target-dir=/backups/full/20241027/ \
--incremental-dir=/backups/incr/20241028/
~~~

#### 2.3 应用第二次增量备份到合并后的备份上

~~~bash
xtrabackup --prepare --apply-log-only --target-dir=/backups/full/20241027/ \
--incremental-dir=/backups/incr/20241029/
~~~

#### 2.4 最终准备（Final Prepare）

所有增量备份都应用完毕后，进行最终的准备，这会回滚所有未提交的事务，使备份集处于一个完全一致、可启动的状态。**注意这里没有 `--apply-log-only` 或 `--redo-only` 参数。**

下面的命令执行成功后，**`/backups/full/20241027/` 这个目录已经包含了从周日到周二晚上的所有数据。**

~~~bash
xtrabackup --prepare --target-dir=/backups/full/20241027/
~~~

#### 2.5 恢复文件到 MySQL 数据目录

1. 使用 `xtrabackup` 命令恢复（推荐此方式，它能正确处理文件权限）。注意，这个命令会自动 copy 文件到两个目录：`/varlog/mysql` 和 `/var/lib/mysql` 下面，如果这两个目录下面有文件则会发生拷贝失败。

~~~bash
xtrabackup --copy-back --target-dir=/backups/full/20241027/
~~~

2. 属主属组

~~~bash
chown -R mysql:mysql /var/lib/mysql
chown -R mysql:mysql /var/log/mysql
~~~

#### 2.6 应用 Binlog 实现精确时间点恢复

1. 启动 mysql 服务

~~~bash
systemctl start mysqld
~~~

2. 确定恢复的起点和终点

~~~bash
# 起点
[root@rocky 20241027]# cat /backups/full/20241027/xtrabackup_binlog_info
binlog.000005	157

# 终点
# 如果是直接删除文件 rm -rf /var/lib/mysql，那么直接不需要确定终点位置
# 如果是执行 SQL 误操作，需要查看 binlog 确定终点位置
mysqlbinlog /var/log/mysql_old_log/binlog.000005 --base64-output=decode-rows -vvv | less
~~~

3. binlog 恢复

~~~bash
# 将 Binlog 转换为 SQL 并应用到数据库
mysqlbinlog /var/log/mysql_old_log/binlog.000005 \
--start-position=157 | mysql -u root -p
~~~

