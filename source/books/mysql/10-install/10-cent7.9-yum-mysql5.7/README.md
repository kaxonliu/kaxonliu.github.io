# Centos7.9 yum 安装 mysql5.7

适合在 x86-64 平台安装mysql



## 下载并安装 yum 源

安装后会在 `/etc/yum.repo.d` 文件夹下面创建 mysql 的 repo 文件。

```bash
wget -i -c http://dev.mysql.com/get/mysql57-community-release-el7-10.noarch.rpm
yum -y install mysql57-community-release-el7-10.noarch.rpm # 安装
```



## 安装 mysql

~~~bash
yum -y install mysql-community-server --nogpgcheck
~~~



## 启动

~~~bash
systemctl start mysqld
systemctl enable mysqld
systemctl status mysqld
~~~



## 查看临时密码

~~~bash
[root@me mysql]# grep 'temporary password' /var/log/mysqld.log
2025-09-21T03:15:56.587850Z 1 [Note] A temporary password is generated for root@localhost: QhelcBu*p8ur
~~~



## 安全加固

按照提示输入旧密码（临时密码）和设置新密码，然后做一些选择设置。

~~~bash
mysql_secure_installation
~~~



## 使用新密码登陆

按照提示输入上一步为 root 账号设置的新密码。

~~~bash
mysql -u root -p
~~~

