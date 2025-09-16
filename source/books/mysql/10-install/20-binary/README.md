# yum 安装

在 centos7.9 上使用 yum 的方式安装 mariadb-5.5.68-1.el7.aarch64



## 配置 yum 源

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



## 安装

~~~bash
yum install mariadb-server -y
yum install -y mariadb
~~~



## 启动

~~~bash
systemctl start mariadb
systemctl status mariadb
systemctl enable mariadb
~~~



## 运行保护mysql

~~~bash
mysql_secure_installation
~~~



