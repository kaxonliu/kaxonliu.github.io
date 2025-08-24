# SSH远程管理服务

SSH（Secure Shell）是一种加密的网络传输协议，用于在不安全的网络中提供安全的远程登录、命令执行、文件传输等服务。在 Linux 中，最广泛使用的 SSH 服务实现是 **OpenSSH**，它是CS架构的软件。

一个完整的 SSH 环境由两部分组成：

1. **SSH 服务端 (OpenSSH-Server)**：运行在需要被远程管理的服务器上，监听来自客户端的连接请求。
2. **SSH 客户端 (OpenSSH-Client)**：用户使用它来连接到远程服务器。Linux 和 macOS 系统通常自带 `ssh` 命令，Windows 用户可以使用 PuTTY、Xshell、Windows Terminal（集成 OpenSSH）等。



## SSH服务端

有些 Linux 发行版上，OpenSSH 服务器可能没有预装。可以使用使用包管理器来安装它。

### 安装SSH

~~~bash
# Ubuntu
sudo apt install openssh-server

# Centos
yum install openssh-server
~~~



### 启动服务

安装后，需要启动 SSH 服务并设置为开机自动启动。

~~~bash
sudo systemctl start sshd    # 启动服务
sudo systemctl enable sshd   # 设置开机自启
sudo systemctl status sshd   # 检查服务状态
~~~

>注意，Ubuntu 上使用的是服务名是 ssh，而不是 sshd



### 配置服务

SSH 服务端的配置文件是 `/etc/ssh/sshd_config`，修改配置文件后需要重启 SSH 服务才能生效。

~~~bash
systemctl restart sshd
~~~

#### 关键服务器配置选项 (`sshd_config`)

编辑配置文件时务必小心，错误的配置可能导致无法远程登录。**建议修改前备份配置文件**。

| 配置项                         | 默认值              | 说明与建议                                                   |
| :----------------------------- | :------------------ | :----------------------------------------------------------- |
| `Port`                         | `22`                | SSH 监听端口。为了安全，可以改为一个非标准端口（如 2022）。**修改后记得更新防火墙规则。** |
| `PermitRootLogin`              | `prohibit-password` | 是否允许 root 用户直接登录。**出于安全考虑，建议设置为 `no`**，然后使用普通用户登录后再 `su` 或 `sudo` 切换权限。 |
| `PubkeyAuthentication`         | `yes`               | 是否启用公钥认证。**应设置为 `yes`**，这是使用密钥登录的基础。 |
| `PasswordAuthentication`       | `yes`               | 是否允许使用密码认证。**在使用密钥登录并测试成功后，建议设置为 `no`**，极大增强安全性（防暴力破解）。 |
| `PermitEmptyPasswords`         | `no`                | 是否允许空密码登录。**必须保持为 `no`**。                    |
| `X11Forwarding`                | `no`                | 是否允许转发图形界面。如果需要运行远程图形程序，可设置为 `yes`。 |
| `ClientAliveInterval`          | `0`                 | 服务器向客户端请求响应的间隔（秒）。设置为 `300` 可以防止不活跃的连接永远不断开。 |
| `ClientAliveCountMax`          | `3`                 | 上述请求的最大次数。                                         |
| `UseDNS`                       | `yes`               | 对客户端IP反向DNS解析，建议取消**。应设置为 `no`**，这样也可以加快远程连接速度。 |
| `GSSAPIAuthentication`         | `yes`               | SSH 客户端和服务器仍然会尝试进行 GSSAPI 认证协商。**如果不确定是否需要使用，强烈建议把它设置为 `no`**。 |
| `KbdInteractiveAuthentication` | `no`                | 是否开启键盘交互式认证。建议保持默认设置`no`                 |

#### 增强安全的配置示例

~~~bash
Port 2222
PermitRootLogin no
PubkeyAuthentication yes
PasswordAuthentication no
~~~



### 保存客户端公钥

相比于密码，使用公钥/私钥对进行认证更安全、更方便。使用密钥认证需要把客户端的公钥保存在服务端，服务端会在文件 `~/.ssh/authorized_keys` 中保存客户端的公钥。





## SSH客户端

ssh 客户端可以使用的命令有很多，必须使用 `ssh` 命令远程连接、使用 `scp` 命令远程拷贝文件、使用 `ssh-keygen` 命令在本机生成私钥和公钥、使用`ssh-copy-id` 命令上传公钥等。



### 使用 ssh 远程连接

远程连接的方式有两种，使用密码的方式和使用密钥（就是所谓的免密登陆）。

#### 密码登陆

如果服务器 SSH 服务不在默认的 22 端口，使用 `-p` 参数指定端口号。使用的登陆用户名和服务器IP地址之间使用 `@` 符号连接。

~~~bash
ssh -p 22 username@remote_server_ip
~~~

- 如果是第一次连接，系统会提示你确认远程主机的指纹（公钥），输入 `yes` 继续。
- 然后会提示你输入远程服务器上对应用户的密码。

#### 密钥登陆

**第一步：在客户端生成密钥对（如果该没有的话）**

~~~bash
ssh-keygen -t rsa -C "your_email@example.com"

# 一路回车，默认在当前用户家目录创建私钥和共钥匙
# 私钥 ~/.ssh/id_rsa	
# 共钥 ~/.ssh/id_rsa.pub
~~~

**第二步：将公钥上传到服务器**

使用命令 `ssh-copy-id` 实现，它会提示你输入密码，之后会自动将你的公钥追加到服务器对应用户的 `~/.ssh/authorized_keys` 文件中。

~~~bash
ssh-copy-id -p 22 user@192.168.1.100
~~~

>注意，如果服务端关闭了密码认证，则需要要公要给系统管理员让其手动添加。

**第三步：密钥登陆**

~~~bash
ssh -p 22 user@192.168.1.100	# 就不要输入密码了
~~~



### 使用 scp 远程拷贝

SSH 还提供了安全的文件传输命令，通过 `scp` 复制文件。

~~~bash
# 上传本地文件到服务器
scp -P 2222 /local/path/file.txt user@192.168.1.100:/remote/path/

# 从服务器下载文件到本地
scp -P 2222 user@192.168.1.100:/remote/path/file.txt /local/path/
~~~

>**注意**：`scp` 用 `-P` 指定端口，而 `ssh` 用 `-p`



### 配置SSH Config文件

上述使用 `ssh` 命令、`scp` 命令每次都需要输入服务器上的用户名和IP地址，尤其是需要记住服务端的IP地址。为了方便登陆，可以在客户端机器上配置 ssh config 文件。

编辑本地 `~/.ssh/config` 文件（不存在则新建），添加以下内容并保存。如果有多台服务器，可以在 `~/.ssh/config` 文件中添加多条记录。

~~~bash
# ~/.ssh/config
Host centos						  				# 自定义别名（替代IP/域名）
    HostName 192.168.1.100      # 服务器IP或域名
    User root                   # 登录用户名
    Port 22                     # SSH端口（默认22可省略）
    IdentityFile ~/.ssh/id_rsa  # 指定私钥路径（默认~/.ssh/id_rsa）
    IdentitiesOnly yes          # 强制使用指定的密钥认证，否则会尝试使用默认密钥
~~~

直接使用别名登陆，无需输入用户名和IP

~~~bash
ssh centos
~~~


