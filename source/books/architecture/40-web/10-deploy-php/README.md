# 部署 php



主要就是 lnmp 环境。

## 系统环境

~~~bash
sed -ri 's/enforcing/disabled/g' /etc/sysconfig/selinux 
systemctl stop firewalld
systemctl disable firewalld
 
setenforce 0
iptables -t filter -F
~~~



## 安装 nginx

两种安装方式。到官网找源码，使用源码编译安装。第二种找 nginx 的 yum 源，使用yum 安装。

参考官网提供的 repo ，使用 yum 安装。[点击链接](https://nginx.org/en/linux_packages.html)

安装 `yum-utils`

~~~bash
sudo yum install yum-utils
~~~

新建 repo 文件`/etc/yum.repos.d/nginx.repo`

~~~bash
[nginx-stable]
name=nginx stable repo
baseurl=http://nginx.org/packages/centos/$releasever/$basearch/
gpgcheck=1
enabled=1
gpgkey=https://nginx.org/keys/nginx_signing.key
module_hotfixes=true

[nginx-mainline]
name=nginx mainline repo
baseurl=http://nginx.org/packages/mainline/centos/$releasever/$basearch/
gpgcheck=1
enabled=0
gpgkey=https://nginx.org/keys/nginx_signing.key
module_hotfixes=true
~~~

安装

~~~bash
yum install -y nginx
~~~



## 安装 mysql

可以安装 mysql 也可以安装 mariadb

### 安装 mariadb

~~~bash
# 安装
yum remove mysql* -y  # 如果你要安装mariadb，那就要清理掉机器上安装过的mysql，因为二者会有冲突
yum install mariadb* -y
 
rm -rf /var/lib/mysql/*
systemctl start mariadb

# 创建库、创建远程登录账号、授权
mysql -uroot -p
-- 执行以下命令，创建 MariaDB 数据库。例如 “wordpress”。
CREATE DATABASE wordpress;
-- 执行以下命令，创建一个新用户。例如 “user”，登录密码为 123456。
CREATE USER 'liuxu'@'localhost' IDENTIFIED BY '123456';
-- 执行以下命令，赋予用户对 “wordpress” 数据库的全部权限。
GRANT ALL PRIVILEGES ON wordpress.* TO 'liuxu'@'localhost';
~~~

### 安装 mysql

~~~bash
#1 如果之前有残留，请先卸载，否则会与下面的安装冲突
yum remove mysql* mariadb* -y 
 
#2 下载并安装yum源
wget -i -c http://dev.mysql.com/get/mysql57-community-release-el7-10.noarch.rpm
yum -y install mysql57-community-release-el7-10.noarch.rpm # 安装
 
#3 安装mysql
yum -y install mysql-community-server --nogpgcheck

#4 启动
systemctl start mysqld
systemctl enable mysqld
systemctl status mysqld

#5 获取临时密码
grep 'temporary password' /var/log/mysqld.log


#6 加固mysql
mysql_secure_installation

Securing the MySQL server deployment.
 
Enter password for user root:    #输入上一步骤中获取的安装MySQL时自动设置的root用户密码
The existing password for the user account root has expired. Please set a new password.
 
New password:  #设置新的root用户密码
 
Re-enter new password:   #再次输入密码
The 'validate_password' plugin is installed on the server.
The subsequent steps will run with the existing configuration of the plugin.
Using existing password for root.
 
Estimated strength of the password: 100
Change the password for root ? ((Press y|Y for Yes, any other key for No) : N   #输入N否则会需要再设置一遍
 
 ... skipping.
By default, a MySQL installation has an anonymous user,
allowing anyone to log into MySQL without having to have
a user account created for them. This is intended only for
testing, and to make the installation go a bit smoother.
You should remove them before moving into a production
environment.
 
Remove anonymous users? (Press y|Y for Yes, any other key for No) : Y   #是否删除匿名用户，输入Y
Success.
 
Normally, root should only be allowed to connect from 'localhost'. This ensures that someone cannot guess at the root password from the network.
 
Disallow root login remotely? (Press y|Y for Yes, any other key for No) : Y   #禁止root远程登录，输入Y
Success.
 
By default, MySQL comes with a database named 'test' that anyone can access. This is also intended only for testing, and should be removed before moving into a production environment.
 
Remove test database and access to it? (Press y|Y for Yes, any other key for No) : Y   #是否删除test库和对它的访问权限，输入Y
 - Dropping test database...
Success.
 
 - Removing privileges on test database...
Success.
 
Reloading the privilege tables will ensure that all changes
made so far will take effect immediately.
 
Reload privilege tables now? (Press y|Y for Yes, any other key for No) : Y   #是否重新加载授权表，输入Y
Success.
 
All done!


#7 新密码登陆
mysql -u root -p

#修改密码
用
   SET PASSWORD FOR 'root'@'localhost' = 'new_password';
或者
   ALTER USER 'root'@'localhost' IDENTIFIED BY 'new_password';
 
mysql8创建新账户用一条命令会报错
grant all on *.* to root@'192.168.71.%' identified by '123';
需要拆分为三条命令执行才行
create user 'root'@'192.168.71.%';
alter user 'root'@'192.168.71.%' identified by '123';
grant all on *.* to 'root'@'192.168.71.%';
flush privileges;
~~~



## 安装启动 php-fpm

### 安装 php-fpm

~~~bash
# 安装epel源。
yum -y install epel-release
 
# 安装remi源。
yum -y install http://rpms.remirepo.net/enterprise/remi-release-7.rpm
 
# 安装Yum源管理工具。
yum -y install yum-utils
 
# 安装PHP 7.4。
yum -y install php74-php-gd php74-php-pdo php74-php-mbstring php74-php-cli php74-php-fpm php74-php-mysqlnd
 
# 查看PHP的安装版本。
php74 -v
 
 
# 配置信息（如果是centos9默认以socket方式启动，想用端口方式，需要改listen=127.0.0.1:9000）
cat /etc/opt/remi/php74/php-fpm.d/www.conf |grep -v '^;' |grep -v '^$'
 
 
#启动PHP服务并设置开机自启动。
systemctl start php74-php-fpm
systemctl enable php74-php-fpm
~~~



## 修改 nginx 配置文件

删除 `/etc/nginx/nginx.conf` 文件中 http 模块下面的 server 配置信息，保留 `include /etc/nginx/conf.d/*.conf` 配置。在这个文件夹下面新建配置文件 www.conf

~~~nginx
server {
    listen       80;
    server_name  localhost;
    #access_log /var/log/nginx/host.access.log  main;

    # 非.php结尾请求从root指定的目录里直接拿，例如/、/1.jpg、/a/b/2.css
    location / {
        root   /usr/share/nginx/html;
        index index.php index.html index.htm a.txt;
    }

    # 以.php结尾的请求，交给fastcgi程序处理,下面的配置没有一点是多余的
    location ~ \.php$ {
        include        fastcgi_params;    
        fastcgi_pass   127.0.0.1:9000;
        fastcgi_param  SCRIPT_FILENAME /usr/share/nginx/html$fastcgi_script_name;    
    }
}
~~~



重启 nginx 服务

~~~bash
systemctl restart nginx
~~~



## 部署 wordpress

#### 1. 初始化数据库 wordpress 需要的数据库

~~~bash
# 1、执行以下命令，并按照提示信息输入MySQL的root用户，登录到MySQL命令行。
mysql -u root -p
 
# 2、执行以下命令，创建一个新的数据库。
CREATE DATABASE wordpress charset=utf8mb4;
 
其中，“wordpress”为数据库名，可以自行设置。
 
# 3、执行以下命令，为数据库创建用户并为用户分配数据库的完全访问权限。
GRANT ALL ON wordpress.* TO wordpressuser@localhost IDENTIFIED BY 'BLOck@123';
 
# 4、执行以下命令，退出MySQL命令行。
exit
 
# 5、（可选）依次执行以下命令，验证数据库和用户是否已成功创建，并退出MySQL命令行。
mysql -u wordpressuser -p
SHOW DATABASES
~~~



#### 2. 下载 wordpress

~~~bash
# 官网：https://wordpress.org/download/
wget https://wordpress.org/latest.zip

# 解压到 /usr/share/nginx/html/
# 解压后生成一个“wordpress”的文件夹。
unzip latest.zip -d  /usr/share/nginx/html
~~~

#### 3. 修改配置文件

~~~bash
cd /usr/share/nginx/html/wordpress
cp wp-config-sample.php wp-config.php

# 修改数据库连接需要的信息
vi wp-config-php

# 保存退出
~~~

#### 4. 测试

访问连接：`http://nginx所在的ip和端口/wordpress`

