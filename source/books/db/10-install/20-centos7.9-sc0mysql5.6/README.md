# Centos7.9 源码安装 mysql5.6





## 安装依赖

~~~bash
yum install -y ncurses-devel libaio-devel gcc gcc-c++ glibc cmake autoconf openssl openssl-devel
~~~



## 下载安装包

~~~bash
wget https://downloads.mysql.com/archives/get/p/23/file/mysql-5.6.46.tar.gz
~~~



## 解压

~~~bash
tar xvf mysql-5.6.46.tar.gz
~~~



## 生成 cmake

~~~bash
cd mysql-5.6.46/

# cmake 
cmake . -DCMAKE_INSTALL_PREFIX=/service/mysql-5.6.46 \
-DMYSQL_DATADIR=/service/mysql-5.6.46/data \
-DMYSQL_UNIX_ADDR=/service/mysql-5.6.46/tmp/mysql.sock \
-DDEFAULT_CHARSET=utf8mb4 \
-DDEFAULT_COLLATION=utf8mb4_general_ci \
-DWITH_EXTRA_CHARSETS=all \
-DWITH_INNOBASE_STORAGE_ENGINE=1 \
-DWITH_FEDERATED_STORAGE_ENGINE=1 \
-DWITH_BLACKHOLE_STORAGE_ENGINE=1 \
-DWITHOUT_EXAMPLE_STORAGE_ENGINE=1 \
-DWITH_ZLIB=bundled \
-DWITH_SSL=system \
-DENABLED_LOCAL_INFILE=1 \
-DWITH_EMBEDDED_SERVER=1 \
-DENABLE_DOWNLOADS=1 \
-DWITH_DEBUG=0
~~~



## 编译并安装

~~~bash
make && make install
~~~



## 查看安装后程序目录

~~~bash
[root@centos mysql-5.6.46]# ls -l /service/mysql-5.6.46/
总用量 248
drwxr-xr-x.  2 root root   4096 9月  16 22:19 bin
drwxr-xr-x.  3 root root     18 9月  16 22:19 data
drwxr-xr-x.  2 root root     55 9月  16 22:19 docs
drwxr-xr-x.  3 root root   4096 9月  16 22:19 include
drwxr-xr-x.  3 root root   4096 9月  16 22:19 lib
-rw-r--r--.  1 root root 221739 9月  27 2019 LICENSE
drwxr-xr-x.  4 root root     30 9月  16 22:19 man
drwxr-xr-x. 10 root root   4096 9月  16 22:19 mysql-test
-rw-r--r--.  1 root root    587 9月  27 2019 README
drwxr-xr-x.  2 root root     30 9月  16 22:19 scripts
drwxr-xr-x. 28 root root   4096 9月  16 22:19 share
drwxr-xr-x.  4 root root   4096 9月  16 22:19 sql-bench
drwxr-xr-x.  2 root root    136 9月  16 22:19 support-files
~~~



## 制作软连接

方便后期 MySQL 服务版本升级，只需要把软连接指向新的程序目录即可。

~~~bash
ln -s /service/mysql-5.6.46 /service/mysql
~~~



## 创建数据库用户

~~~bash
useradd mysql -s /sbin/nologin -M
~~~



## 拷贝配置文件

拷贝 mysql 的默认配置文件，放到 `/etc/my.cnf`

~~~bash
cd /service/mysql/support-files/
cp my-default.cnf /etc/my.cnf
~~~

修改配置文件 `/etc/my.cnf` 为如下内容。

~~~bash
[mysqld]
datadir=/service/mysql/data
socket=/service/mysql/tmp/mysql.sock
# Disabling symbolic-links is recommended to prevent assorted security risks
symbolic-links=0
# Settings user and group are ignored when systemd is used.
# If you need to run mysqld under a different user or group,
# customize your systemd unit file for mariadb according to the
# instructions in http://fedoraproject.org/wiki/Systemd

[mysqld_safe]
log-error=/var/log/mariadb/mariadb.log
pid-file=/var/run/mariadb/mariadb.pid

#
# include all files from the config directory
#
!includedir /etc/my.cnf.d
~~~



## 配置 systemd 管理mysql 服务

~~~bash
cat > /usr/lib/systemd/system/mysql.service << "EOF"
[Unit]
Description=MySQL Server
Documentation=man:mysqld(8)
Documentation=https://dev.mysql.com/doc/refman/en/using-systemd.html
After=network.target
After=syslog.target
[Install]
WantedBy=multi-user.target
[Service]
User=mysql
Group=mysql
ExecStart=/service/mysql/bin/mysqld --defaults-file=/etc/my.cnf
LimitNOFILE = 5000
EOF
~~~

#### 重新加载启动文件列表

~~~bash
systemctl daemon-reload
~~~



## 初始化数据库

~~~bash
cd /service/mysql/scripts/

./mysql_install_db --user=mysql --basedir=/service/mysql --datadir=/service/mysql/data
~~~



## 创建 socket 文件目录

二进制安装不需要这一步

~~~bash
mkdir /service/mysql/tmp
~~~

## 授权数据库目录

~~~bash
chown -R mysql.mysql /service/mysql
chown -R mysql.mysql /service/mysql-5.6.46
~~~



## systemctl 启动管理 mysql

~~~bash
systemctl start mysql
systemctl status mysql
systemctl enable mysql

# 查看 mysql进程
ps -ef | grep [m]ysql

# 查看端口
netstat -lntp | grep 3306
~~~



## 修改 root 账号密码

~~~bash
mysql -u root -p 

# 方式1
SET PASSWORD=PASSWORD('123456');

# 在旧版本的 MySQL（主要是 5.7 及之前）中，
# 这是一种标准的修改当前用户密码的方法。 
# 但在新版本（8.0+）中，这条命令已经失效并会报错。


# 方式2
grant all privileges on *.* to root@'%' identified by '123';


# 刷新
flush privileges;
~~~



