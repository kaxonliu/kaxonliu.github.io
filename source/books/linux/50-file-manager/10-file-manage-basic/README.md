# 文件管理-基础篇

linux的文件系统是单根结构（windows的是多根结构），所有文件的起点是根 `/`。整个目录系统是一个树型结构，树根是 `/`。



## 绝对路径和相对路径

在用户的角度来看文件系统就是一系列目录文件和文件，使用时会涉及到**绝对路径和相对路径**。绝对路径是起点从 `/` 开始的路径，相对路径是起点从当前位置开始的路径。

几个特殊的符号：

~~~bash
cd .	 				# 切换到当前目录
cd ..					# 切换到上层目录
cd ~					# 切换到普通用户家目录
cd #					# 切换到root用户的家目录

cd ~liuxu			# 切换到用户刘旭的家目录，等价于 cd /home/liuxu
~~~



## 文件的种类

linux的理念是一切皆文件，任何操作都是在操作文件，因此文件具有不同的类型，标识相应的资源。

**普通文件**。普通文件（文本、图片、音视频等）的标识符为 `-`

~~~bash
[root@node1 ~]# ls -l
total 0
-rw-r--r-- 1 root root 0 Aug 11 13:47 a.txt
~~~

**目录文件**。文件夹的标识符是 `d`

**软连接文件**。软连接的标识符是 `l`

~~~bash
[root@node1 /]# ls -l
total 72
lrwxrwxrwx.   1 root root     7 Mar  7  2019 bin -> usr/bin
dr-xr-xr-x.   5 root root  4096 Jul  8  2024 boot
drwxr-xr-x    2 root root  4096 Nov  5  2019 data
drwxr-xr-x   19 root root  3020 Aug 10 22:04 dev
~~~

**块文件**。标识符是 `b`。硬盘的最小读写单位是1个扇区。操作系统最小的读写单位是8个扇区。

~~~bash
[root@node1 dev]# ls -l /dev/vda
brw-rw---- 1 root disk 253, 0 Aug 10 22:04 /dev/vda
~~~

**字符文件**。标识符是 `c` 登陆进来的shell解释器就是一个字符终端

~~~bash
[root@node1 dev]# who
lighthouse pts/0        2025-08-11 13:15 (orcaterm)
lighthouse pts/1        2025-08-11 13:15 (orcaterm)
[root@node1 dev]# ls -l /dev/null /dev/zero /dev/pts/0
crw-rw-rw- 1 root       root         1, 3 Aug 10 22:04 /dev/null
crw--w---- 1 lighthouse lighthouse 136, 0 Aug 11 13:59 /dev/pts/0
crw-rw-rw- 1 root       root         1, 5 Aug 10 22:04 /dev/zero
~~~

**套接字文件**。标识符是 `s`。表示一个网络通信的socket文件

~~~bash
[root@node1 ~]# ls -l /run/chrony/chronyd.sock 
srwxr-xr-x 1 chrony chrony 0 Aug 11 10:47 /run/chrony/chronyd.sock
~~~



## Linux根目录下文件夹的用途

Linux 根目录（`/`）下的每个文件夹都有特定的用途，遵循 **Filesystem Hierarchy Standard (FHS)** 标准。以下是主要目录的作用。

| **目录**          | **功能**                                              | **举例**                                    |
| :---------------- | :------------------------------------------------------- | :------------------------------------------ |
| **`/bin`**        | 存放系统基础命令（所有用户可用）                         | `/bin/ls`, `/bin/cp`, `/bin/bash`           |
| **`/sbin`**       | 系统管理命令（需root权限）                              | `/sbin/iptables`, `/sbin/reboot`            |
| **`/boot`**       | 存放内核、引导加载程序（GRUB）和启动文件                 | `/boot/vmlinuz-*`, `/boot/grub/grub.cfg`    |
| **`/dev`**        | 设备文件（硬件和虚拟设备接口）                           | `/dev/sda`（硬盘）, `/dev/null`（黑洞设备） |
| **`/etc`**        | 系统全局配置文件                                      | `/etc/passwd`, `/etc/ssh/sshd_config`       |
| **`/home`**       | 普通用户的家目录（每个用户独立子目录）                   | `/home/alice`, `/home/liuxu`                  |
| **`/root`**       | root用户的家目录（普通用户无权访问）                     | `/root/.bashrc`                             |
| **`/lib`**        | 系统共享库和内核模块（32位）                             | `/lib/libc.so.6`, `/lib/modules/`           |
| **`/lib64`**      | 64位系统的共享库（仅64位系统存在）                       | `/lib64/ld-linux-x86-64.so.2`               |
| **`/media`**      | 自动挂载可移动设备（U盘、光盘等）                        | `/media/usb`, `/media/cdrom`                |
| **`/mnt`**        | 临时手动挂载点（如硬盘分区、NFS）                        | `/mnt/disk`, `/mnt/nas`                     |
| **`/opt`**        | 第三方独立安装的软件                                     | `/opt/google/chrome/`                       |
| **`/proc`**       | 虚拟文件系统（内核和进程实时信息）                       | `/proc/cpuinfo`, `/proc/meminfo`            |
| **`/run`**        | 运行时数据（PID文件、套接字等，重启后丢失）              | `/run/sshd.pid`, `/run/docker.sock`         |
| **`/tmp`**        | 临时文件（所有用户可读写，重启可能清空）                 | `/tmp/install.log`                          |
| **`/usr`**        | 用户程序资源（二级文件系统，类似Windows的Program Files） | `/usr/bin/python`, `/usr/lib/libssl.so`     |
| **`/var`**        | 可变数据（日志、缓存、数据库等）                         | `/var/log/syslog`, `/var/lib/mysql/`        |
| **`/srv`**        | 服务相关数据（如网站、FTP文件）                          | `/srv/www/`, `/srv/ftp/`                    |
| **`/sys`**        | 虚拟文件系统（内核设备、驱动参数配置）                   | `/sys/class/net/eth0`                       |
| **`/lost+found`** | `fsck`修复文件系统时恢复的碎片文件（ext3/ext4专用）      | `/lost+found/file123`                       |

>补充说明：
>
>- **虚拟目录**：`/proc`、`/sys`、`/run` 不占用磁盘空间，由内核动态生成。
>- **权限控制**：`/root` 仅限 root 访问，`/home` 下用户目录归属相应用户。
>- **历史目录**：`/srv` 和 `/opt` 常用于规范第三方服务的数据存储。





## 文件管理基础命令

**新建文件**

~~~bash
touch abc.txt						# 新建一个文件
touch abc{1..3}.txt			# 新建后缀连续的三个文件
touch abc{4,6,10}.txt		# 新建后缀不同的三个文件
~~~

**新建文件夹**

~~~bash
mkdir aaa								# 新建文件夹 aaa
mkdir me{1..3}					# 新建文件夹 me1 me2 me3
mkdir me{4,7,9}					# 新建文件夹 me4 me7 me9

mkdir -p /tmp/xxx/yyy   # 递归创建文件夹 /tmp/xxx 和 /tmp/xxx/yyy			
~~~

**复制文件和文件夹**

~~~bash
cp <src> <dst>					# 把原目标复制到目标位置

cp abc.txt	/tmp/				# 把当前文件夹下的abc.txt 复制到/tmp/文件夹下，文件名依然为 abc.txt
cp abc.txt  /tmp/1.txt	# 把当前文件夹下的abc.txt 复制到/tmp/文件夹下，文件名为 1.txt


cp -r xxx/ /tmp/				# 把当前文件夹递归着复制到 /tmp/文件夹下，文件夹名字依然为 xxx
~~~

**移动文件或文件夹**

~~~bash
mv abc.txt /tmp/				# 把当前文件夹下的abc.txt文件复制到 /tmp文件夹下，文件名依然为 abc.txt
mv abc.txt abcc.txt			# 把abc.txt 重命名为abcc.txt
mv xxx xxyy							# 把文件夹xxx 重命名为 xxyy
~~~

**删除文件或文件夹**

~~~bash
rm abc.txt							# 删除abc.txt
rm *.txt								# 批量删除以 .txt结尾的文件
rm -f abc.txt						# 强制删除文件
rm -rf xxx							# 强制并递归删除文件夹
~~~

**查看文件或文件夹的元数据**

~~~bash
ls -l 1.txt							# 展示文件1.txt的信息
ls -l /tmp							# 展示/tmp里面所有文件的信息
ls -l -d /tmp						# 展示/tmp这个文件夹自己的信息
ls -a -l ~							# 展示家目录下的所有文件，包括隐藏文件
~~~



## 文件的元数据

命令 `ls -l` 展示文件的元数据。

~~~bash
[root@node1 ~]# ls -l /boot/ | head -5
total 98156
-rw-r--r--  1 root root   153661 Jun  4  2024 config-3.10.0-1160.119.1.el7.x86_64
drwxr-xr-x. 3 root root     4096 Mar  7  2019 efi
drwxr-xr-x. 2 root root     4096 Mar  7  2019 grub
drwx------. 5 root root     4096 Jul  8  2024 grub2
~~~

其中展示的信息表示的含义很重要。



## 文件的3个时间

使用命令 `stat` 可以查看文件的3个时间和其他信息

~~~bash
[root@node1 ~]# stat abx.txt 
  File: ‘abx.txt’
  Size: 0               Blocks: 0          IO Block: 4096   regular empty file
Device: fd01h/64769d    Inode: 394296      Links: 1
Access: (0644/-rw-r--r--)  Uid: (    0/    root)   Gid: (    0/    root)
Access: 2025-08-11 22:05:39.148952510 +0800
Modify: 2025-08-11 21:45:30.750088368 +0800
Change: 2025-08-11 21:45:30.750088368 +0800
 Birth: -
~~~

其中，

- `atime` ，access time，表示文件内容最近一次被查看的时间
- `mtime`，modify time，表示文件内容最近一次发生变化的时间
- `ctime`，change time，表示文件内容或者文件元数据（文件名，大小，权限等）最近一次发生变化的时间





## 查看文本文件的内容

使用的命令常用的有：
- 查看文件全部内容，`cat abc.txt`
- 翻页查看文件内容，`less abc.txt`
- 查看文件前10行，`head -10 abc.txt`
- 查看文件后10行，`tail -10 abc.txt`
- 动态查看文件尾部追加的内容，`tail -f db.log`
- 使用文本编辑工具编辑文件，`vi abc.txt`

