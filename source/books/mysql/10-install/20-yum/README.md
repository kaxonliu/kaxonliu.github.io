# yum 安装



## 安装 mariadb

在 centos7.9 上使用 yum 的方式安装 mariadb-5.5.68-1.el7.aarch64



### 配置 yum 源

#### 使用 x86-64架构的yum源

```bash
wget -O /etc/yum.repos.d/CentOS-Base.repo https://mirrors.aliyun.com/repo/Centos-7.repo
wget -O /etc/yum.repos.d/epel.repo https://mirrors.aliyun.com/repo/epel-7.repo
yum clean all
yum makecache
```

#### 使用 aarch64 架构的 yum 源

```bash
wget -O /etc/yum.repos.d/CentOS-Base.repo https://mirrors.aliyun.com/repo/Centos-altarch-7.repo
wget -O /etc/yum.repos.d/epel.repo https://mirrors.aliyun.com/repo/epel-7.repo
yum clean all
yum makecache
```

### 安装

~~~bash
yum install mariadb-server -y
yum install -y mariadb
~~~

### 启动

~~~bash
systemctl start mariadb
systemctl status mariadb
systemctl enable mariadb
~~~

### 运行保护mysql

~~~bash
mysql_secure_installation
~~~





## 安装 mysql-8.0

在 rockylinux9 上使用 yum 安装 mysql-8.0.41-2.el9_5.aarch64

~~~bash
yum install -y mysql-server
~~~



Mysql8.0 GRANT 权限要分三步，分别设置远程登陆和本地登陆。

第一步：创建用户（并设置密码）

~~~mysql
-- 远程登陆
CREATE USER 'root'@'%' IDENTIFIED BY '123';
~~~

第二步：授予权限

~~~mysql
-- 远程登陆
GRANT ALL PRIVILEGES ON *.* TO 'root'@'%' WITH GRANT OPTION;
~~~

第三步：刷新权限（可选但建议）

~~~mysql
FLUSH PRIVILEGES;
~~~

