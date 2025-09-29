# MHA

安装部署 mha

环境 rockylinux9

~~~bash
[root@rocky ~]# uname -r
5.14.0-570.17.1.el9_6.aarch64
~~~

安装MHA

~~~bash
# 安装 epel-release 包
sudo dnf -y install epel-release

# 启用 CRB 仓库
sudo dnf config-manager --set-enabled crb
 
# 安装MHA依赖的perl包
yum install -y perl
yum install -y perl-DBD-MySQL perl-Config-Tiny perl-Log-Dispatch perl-Parallel-ForkManager

# 安装 node
wget https://qiniu.wsfnk.com/mha4mysql-node-0.58-0.el7.centos.noarch.rpm --no-check-certificate
rpm -ivh mha4mysql-node-0.58-0.el7.centos.noarch.rpm


# 安装manager
wget https://qiniu.wsfnk.com/mha4mysql-manager-0.58-0.el7.centos.noarch.rpm --no-check-certificate
rpm -ivh mha4mysql-manager-0.58-0.el7.centos.noarch.rpm
~~~





