# rsync 服务和命令

`rsync` 是一个非常强大的工具，它既可以当客户端命令，又可以当服务端。当客户端使用时它类似 `cp` 命令支持本地复制，类似 `scp` 命令支持远程复制。相比 `cp` 和 `scp` 的单纯复制，`rsync` 还可以实现**增量复制**。当服务端时它支持自己的协议实现拷贝功能。并且 `rsync` 支持 SSH 协议，使用 SSH 作为传输协议时，数据传输是加密的。使用前需要先安装。

~~~bash
# 安装 rsync
yum install -y rsync
~~~



## rsync 本地复制

本地复制的用法和 `cp` 的用法几乎一致，只不过 `rsync` 有自己的参数可以使用。使用 `rsync` 第一次复制是全量拷贝，第二次复制是增量拷贝。`rsync` 在复制前会比对目标文件相对源文件是否发生了变动，如果没有变动则不拷贝，如果变动了再拷贝。

~~~bash
rsync 选项 源路径 目标路径
~~~

### 常用核心选项详解

| 选项                  | 全称               | 作用                                                         |
| :-------------------- | :----------------- | :----------------------------------------------------------- |
| `-v`                  | `--verbose`        | 详细模式，输出同步过程信息。可以多个 `-v` 叠加（如 `-vvv`）以获得更详细输出。 |
| `-a`                  | `--archive`        | **归档模式**。这是**最常用**的选项组合，相当于 `-rlptgoD`。它递归同步、保持几乎所有文件属性（权限、所有者、组、时间戳、符号链接等）。**几乎适用于所有备份场景**。 |
| `-r`                  | `--recursive`      | 递归同步目录及其子目录。                                     |
| `-z`                  | `--compress`       | 在传输过程中进行压缩，以节省带宽。                           |
| `-h`                  | `--human-readable` | 以易读的格式（如 K, M, G）输出数字。                         |
| `-n`                  | `--dry-run`        | **试运行**。模拟同步过程，显示会做什么操作，但**不会实际传输任何文件**。**强烈建议在重要操作前先使用此选项检查！** |
| `--progress`          |                    | 显示每个文件的传输进度。                                     |
| `--delete`            |                    | **删除目标路径中源路径没有的文件**。**慎用！** 它让目标成为源的精确镜像。使用前务必结合 `-n` 测试。 |
| `-e`                  | `--rsh=COMMAND`    | 指定使用的远程 Shell。最常用于指定 SSH 的端口或密钥：`-e "ssh -p 2222 -i /path/to/key"` |
| `-P`                  |                    | 等价于 `--partial --progress`。保留部分传输的文件并显示进度。 |
| `--exclude=PATTERN`   |                    | 排除匹配 PATTERN 的文件或目录。                              |
| `--exclude-from=FILE` |                    | 从指定文件中读取要排除的模式列表（每行一个模式）。           |
| `--include=PATTERN`   |                    | 包含匹配 PATTERN 的文件或目录，通常与 `--exclude` 配合使用。 |
| `--bwlimit=RATE`      |                    | 限制 I/O 带宽，单位是 KB/s。例如 `--bwlimit=1000` 限制为 1000KB/s（约 1MB/s）。 |

### 示例

#### 1. 源路径的斜杠 (`/`)

源路径是文件夹时，如果带了后缀 `/` 表示把文件夹下面的内容同步到目标路径下；如果没有后缀 `/` 表示把文件夹自己同步到目标路径下。

~~~bash
rsync -av ~/Documents/ /media/usb/backup/docs/
rsync -av ~/Documents /media/usb/backup/docs/
~~~



#### 2. 使用 `--delete` 进行精确镜像备份

确保远程备份目录和本地源目录完全一致，删除备份目录中多余的文件。

~~~bash
rsync -av --delete ~/Documents/ /media/usb/backup/docs/
~~~



#### 3. 先测试再执行

**试运行**。模拟同步过程，显示会做什么操作，但**不会实际传输任何文件**。强烈建议在重要操作前先使用此选项检查！

~~~bash
[root@centos data]# rsync -avn a/* b
sending incremental file list
1.txt

sent 47 bytes  received 19 bytes  132.00 bytes/sec
total size is 3  speedup is 0.05 (DRY RUN)
~~~



#### 4. 排除文件和包含文件

同步时希望排除指定文件，可以使用选项 `--exclude`；希望仅同步指定文件，使用选项 `--include`，一般配合`--exclude` 使用。

~~~bash
# 使用 --exclude
rsync -av --exclude='node_modules/' --exclude='*.tmp' /drc/project/ /dst/project

# 使用 --include
rsync -av --include='[0-9].txt' --exclude='*' /drc/project/ /dst/project
~~~



#### 5. 使用 `--backup` 和 `--suffix` 实现同名备份

源文件和目标文件同名，默认会直接覆盖，如果希望备份目标文件的内容，使用 `--backup`，并可以配置 `--suffix` 指定备份文件后缀名。

~~~bash
[root@centos data]# rsync -a --backup a/ b/ && ls b
1.txt  1.txt~

[root@centos data]# rsync -a --backup --suffix='.bak' a/ b/ && ls b
1.txt  1.txt.bak
~~~



## rsync 远程复制

`rsync` 远程复制和本地复制使用的选项都是相同的，不同的是远程复制需要和远程机器上的服务端打交道。打交道的方式分两种：ssh 协议、rsync协议。



### 基于 ssh 协议使用 rsync 远程复制

基于 ssh 协议，需要远程主机上面开启 sshd 服务，同时安装 rsync 即可（不需要开启 rsyncd 服务）。使用方式和 `scp` 远程复制一样。并且如果配置了 ssh 密钥登陆，使用 rsync 时不需要输入密码，否则需要手动输入密码。

~~~bash
rsync -avz a root@10.10.97.210:/data
~~~



#### **使用非标准 SSH 端口** (例如 2022)

~~~bash
rsync -avz -e "ssh -p 2022" /local/path/ user@myserver.com:/remote/path/
~~~





### 基于 rsync 协议使用 rsync 远程复制

基于 rsync 协议，需要远程主机安装 rsync ，同时开启 rsyncd 服务（和 ssh 服务无关）。



#### 使用rsync协议服务端的准备工作

##### 1. 检查 rsyncd.service 服务文件

默认使用 yum 安装 rsync 后会生成服务文件 `/lib/systemd/system/rsyncd.service`，但是如果没有则需要手动创建，手动创建请配置如下内容。

~~~bash
[Unit]
Description=fast remote file copy program daemon
Documentation=man:rsync(1) man:rsyncd.conf(5)
After=network.target

[Service]
EnvironmentFile=/etc/sysconfig/rsyncd
ExecStart=/usr/bin/rsync --daemon --no-detach "$OPTIONS"

[Install]
WantedBy=multi-user.target
~~~

>注意：如果 `/etc/sysconfig/rsyncd` 不存在则创建。
>
>~~~bash
>echo 'OPTIONS=""' > /etc/sysconfig/rsyncd
>~~~



##### 2. 启动 rsyncd 服务

~~~bash
systemctl start rsyncd
systemctl status rsyncd
systemctl enable rsyncd
~~~



##### 3. 配置 rsyncd 的配置文件

如下是一个简单的配置模板。其中，auth users 配置虚拟账号用用于客户端使用 rsync 命令的用户，这个用户通过密码检验后会在服务转转换为 nobody （远端主机真实存在的系统账号nobody）

~~~ini
# /etc/rsyncd.conf

# 全局配置选项，对所有模块生效
uid = nobody
gid = nobody
use chroot = yes
max connections = 10
pid file = /var/run/rsyncd.pid
lock file = /var/run/rsync.lock
log file = /var/log/rsyncd.log
motd file = /etc/rsyncd.motd

# 定义一个模块，客户端通过这个名字来访问
[backup]
    # 要同步的目录路径
    path = /var/www/backup
    # 模块的描述
    comment = Backup Directory
    # 是否只读（yes为只读，no为可读写）
    read only = no
    # 允许访问的客户端IP地址，可以用空格分隔多个IP或网段
    hosts allow = 192.168.1.0/24 10.0.0.100
    # 拒绝所有其他主机
    hosts deny = *
    # 认证的用户名（非系统用户，在secrets file中定义）
    auth users = liuxu
    # 存储用户名和密码的文件
    secrets file = /etc/rsyncd.secrets
~~~

##### 4. 创建密码文件并设置权限

创建在配置文件中指定的密码文件，在文件中按照 `用户名:密码` 的方式一行一行配置用户密码信息。然后设置文件只有 root 可读，否则 rsync 会报错。

~~~bash
echo "liuxu:123" > /etc/rsyncd.secrets
chmod 600 /etc/rsyncd.secrets
chown root:root /etc/rsyncd.secrets
~~~



##### 5. 创建共享目录并设置权限

创建你在配置文件中 `path` 指定的目录，并确保运行用户（这里是 `nobody`）有权限读写它。

~~~bash
sudo mkdir -p /var/www/backup
# 将目录的所有者改为你用于rsync的用户和组（本例是nobody）
sudo chown nobody:nobody /var/www/backup
~~~

##### 6. 配置防火墙

如果系统防火墙（`firewalld`）是开启的，你需要放行 rsync 的默认端口 **873**。或者内网关闭防火墙服务。

~~~bash
sudo firewall-cmd --permanent --add-port=873/tcp
sudo firewall-cmd --reload


# 关闭防火墙
systemctl stop firewalld
systemctl disable firewalld
~~~



##### 7. 重启 rsyncd 服务

~~~bash
systemctl restart rsyncd
~~~



##### 8. 查看 rsyncd 监听的端口。默认监听 873

~~~bash
[root@rocky data]# netstat -tunlp | grep -w 873
tcp        0      0 0.0.0.0:873             0.0.0.0:*               LISTEN      1943/rsync          
tcp6       0      0 :::873                  :::*                    LISTEN      1943/rsync 
~~~





#### 客户端使用 rsync 协议的第一种方式

这种方式和 `scp` 方式类似非常类似。其中，liuxu 是服务端 rsyncd 配置的虚拟账号。backup 是 rsyncd 服务端配置的虚拟账号可以使用的模块。这个模块背后会对应一个路径。

~~~bash
rsync -avz a liuxu@10.10.97.210::backup
~~~



#### 客户端使用 rsync 协议的第二种方式

这种方式显示指定使用 rsync 协议。

~~~bash
rsync -avz a rsync://liuxu@10.10.97.210/backup
~~~



#### 使用密码文件（非交互式适合脚本）

上述两种方式系统都会提示你输入密码（`/etc/rsyncd.secrets` 中为 `rsync_user` 设置的密码）。为了避免手动输入密码，可以在客户端创建一个密码文件（只包含密码）：

~~~bash
echo "your_strong_password_here" > /home/user/rsync_passwd
chmod 600 /home/user/rsync_passwd
~~~

然后在 rsync 命令中使用 `--password-file` 选项：

```bash
rsync -av --password-file=/home/user/rsync_passwd /local/path/ rsync://rsync_user@server_ip/backup
```



#### 限速同步

使用选项 `--bwlimit` 单位是Kb/s

~~~bash
rsync -az --progress --bwlimit=10  abc rsync://liuxu@10.10.97.210/backup
~~~



#### 断点续传

使用选项 `--partial` 

~~~bash
# 可以创建一个大文件测试
dd if=/dev/zero of=abc bs=1G count=5

# 第一次传输
rsync -az --progress --partial --bwlimit=10  abc rsync://liuxu@10.10.97.210/backup

# ctrl + c 停止

# 第二次传输时 重新执行命令后断点续传，先计算断点位置，完毕后续传，然后新内容与旧内容合并到一起
rsync -az --progress --partial --bwlimit=10  abc rsync://liuxu@10.10.97.210/backup
~~~



