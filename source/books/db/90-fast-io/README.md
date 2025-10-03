# 快速导出和导入



## 设置安全目录才行

~~~ini
# /etc/my.cnf
[mysqld]
secure-file-priv=/tmp
~~~

## 快速导出

~~~sql
SELECT * FROM db1.t2
    INTO OUTFILE '/tmp/db1_t2.txt'
    FIELDS TERMINATED BY ','      -- 定义字段分隔符
    OPTIONALLY ENCLOSED BY '"'    -- 定义字符串使用什么符号括起来
    LINES TERMINATED BY '\n';     -- 定义换行符
~~~

查看在文件系统中的位置

~~~bash
[root@rocky tmp]# sudo find / -name "db1_t2.txt" 2>/dev/null
/tmp/systemd-private-b54fa803fd8f4691aa6ca7eb55ea9924-mysqld.service-Ap9N73/tmp/db1_t2.txt
~~~



## 快速导入

~~~sql
LOAD DATA INFILE '/tmp/db1_t2.txt'
INTO TABLE db1.t2
FIELDS TERMINATED BY ','
OPTIONALLY ENCLOSED BY '"'
LINES TERMINATED BY '\n';
~~~



## 其他导出

#### 导出文本

~~~bash
mysql -u root -p -e 'select * from db1.t1' > /tmp/db1_t1.txt
~~~

#### 导出 xml

~~~bash
mysql -u root -p --xml -e 'select * from db1.t1' > /tmp/db1_t1.xml
~~~

#### 导出 html

~~~bash
mysql -u root -p --html -e 'select * from db1.t1' > /tmp/db1_t1.html
~~~



## MySQL迁移数据库方案

方案1：数据库直接导出，拷贝文件到新服务器，在新服务器导入。

方案2：使用第三方迁移工具。比如：MySQLMigrationTool。

方案3：新服务使用使用相同配置的MySQL。数据文件和库表结构拷贝到新服务器。（/var/lib/mysql）

