# 配置SSH远程登陆

在 Linux 服务器上启用远程连接，通常需要配置 **SSH（Secure Shell）** 服务，这是最常用且安全的远程管理方式。大多数 Linux 发行版默认安装了 OpenSSH 服务。

OpenSSH服务是 CS 程序，服务端上运行openssh-server服务，然后客户端使用ssh连接到服务端程序。因此，想要正确使用 ssh 登陆到远程主机，需要在远程主机启动 openssh-server服务，客户端机器安装ssh客户端程序。

~~~bash
# windows可使用的常用工具：xshell、putty、finalshel等
# mac上可以直接自带的 ssh
~~~



## 远程服务器是Centos

### 检查服务器安装并运行 `openssh-server`

~~~bash
# 检查是否安装openssh-server
rpm -qa | grep openssh-server

# 安装 openssh-server
yum install -y openssh-server

# 查看运行状态
systemctl status sshd
~~~



### 使用账号密码远程登陆

~~~bash
liuxu@mac-mini .ssh % ssh root@122.51.164.240
The authenticity of host '122.51.164.240 (122.51.164.240)' can't be established.
ED25519 key fingerprint is SHA256:ukyrI8h6t74IG4MK5i9NFrngs0kqxNHAxgTmPPHh5Bk.
This key is not known by any other names.
Are you sure you want to continue connecting (yes/no/[fingerprint])? yes
Warning: Permanently added '122.51.164.240' (ED25519) to the list of known hosts.
root@122.51.164.240's password:
Last login: Mon Aug 11 16:31:15 2025 from 221.224.161.250
[root@node1 ~]#
~~~

这种方式每次登陆都需要远程服务器上的三个信息： IP地址、用户名、密码。



### 使用SSH 密钥远程登陆

#### 1. 本地创建密钥对

~~~bash
ssh-keygen -t rsa -C "your_email@example.com"

# 默认在当前用户家目录创建私钥和共钥匙
# 私钥 ~/.ssh/id_rsa	
# 共钥 ~/.ssh/id_rsa.pub
~~~



#### 2. 把共钥上传到远程服务器

~~~bash
ssh-copy-id -p 22 root@122.51.164.240

# 远程服务器上登陆的用户名和IP地址；-p 指定SSH服务器监听的端口，默认是22
# 按照提示输入密码上传后共钥匙被保存在服务器上的指定文件中 /root/.ssh/authorized_keys
~~~



#### 3. 远程登陆

~~~bash
liuxu@mac-mini .ssh % ssh  root@122.51.164.240
Last login: Mon Aug 11 16:43:07 2025 from 221.224.161.250
[root@node1 ~]#

# 无需再次输入密码
~~~



### 配置SSH Config文件

上述使用 ssh 远程连接比较麻烦，因为每次都需要输入用户名和IP地址。为了方便登陆，可以在客户端机器上配置 ssh config 文件。



编辑本地 `~/.ssh/config` 文件（不存在则新建），添加以下内容并保存。

~~~bash
Host txy						  				  # 自定义别名（替代IP/域名）
    HostName 122.51.164.240     # 服务器IP或域名
    User root                   # 登录用户名
    Port 22                     # SSH端口（默认22可省略）
    IdentityFile ~/.ssh/id_rsa  # 指定私钥路径（默认~/.ssh/id_rsa）
    IdentitiesOnly yes          # 强制使用指定的密钥认证，否则会尝试使用默认密钥
~~~

直接使用别名登陆，无需再次输入用户名和IP

~~~bash
liuxu@mac-mini ~ % ssh txy
Last login: Mon Aug 11 16:57:08 2025 from 221.224.161.250
[root@node1 ~]#
~~~

如果有多台服务器，可以在 `~/.ssh/config` 文件中添加多条记录。



## 远程服务器是Ubuntu

ubuntu和centos在命令使用上一点点区别，整体流程和原理都是一样的。

### 检查服务器安装并运行 `openssh-server`

~~~bash
# 检查是否安装openssh-server
ubuntu@master:~$ sudo apt list --installed | grep openssh-server
openssh-server/noble-updates,now 1:9.6p1-3ubuntu13.13 arm64 [installed]


# 安装 openssh-server
sudo apt update && sudo apt install openssh-server


# 查看运行状态
systemctl status ssh
~~~



### 确保ssh服务端支持root账号登陆

远程主机上修改 ssh server 的配置文件修改以下参数

~~~bash
# sudo vim /etc/ssh/sshd_config

PermitRootLogin yes
PasswordAuthentication yes
KbdInteractiveAuthentication yes
~~~

重启 ssh server

~~~bash
systemctl restart ssh
systemctl status ssh
~~~



上述信息修改后，即可在客户端机器上使用 ssh 远程登陆。



