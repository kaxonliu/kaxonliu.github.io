# 权限管理

普通文件和目录文件的权限管理是统一的。可以分为两部分：**属主属组的管理、文件读写执行权限**。

对文件的权限管理本质上是给进程使用的，因为进程在运行时都有一个身份，默认这个身份来自启动进程的用户，当然也可以在启动进程时配置用户身份信息。进程在运行过程中可能需要对文件有读写执行的需求，通过管理这些权限，就可以控制进程的执行权限，进而控制用户对文件的权限。



## 属主和属组管理

**同时设置属主和属组**

~~~bash
chown liuxu.group1 abc.txt			# 把abc文件的属主设为liuxu 属组设为group1
~~~

**只设置属主**

~~~bash
chown liuxu abc.txt
~~~

**只设置属组**

~~~bash
chown .group1 abc.txt
~~~

**给文件夹递归设置属主属组**

~~~bash
chown -R liuxu.liuxu abc/
~~~



## 文件读写执行权限管理

### 基本描述方法

文件的权限使用三个符号和三个数字表示，作用于文件和文件夹的表达的含义不同。

| 权限 | 符号和数字 | 对文件的权限                    | 对文件夹的权限                 |
| ---- | ---------- | ------------------------------- | ------------------------------ |
| 读   | `r`， 4    | 可以查看文件内容 cat            | 查看文件夹下面的内容 ls        |
| 写   | `w`， 2    | 可以写文件内容 echo 111>abc.txt | 删除移动重命名文件夹下面的文件 |
| 执行 | `x`， 1    | 执行二进制文件或脚本文件        | 进入文件夹 cd                  |

文件的权限对用户单独设置，文件的创建者叫属主，文件归属的组叫属组，除此之外的其他用户叫其他人。

~~~bash
root@node1 c]# ls -l abc.txt 
-rw-r--r-- 1 root root 5 Aug 15 10:52 abc.txt

# rw-r--r--			权限信息
# root root     属主和属组
~~~

使用命令 `ls -l ` 可以查看文件的权限信息和属主属组信息。

9段权限的前3个表示文件属主对文件拥有的读写执行权限，没有的位置使用 `-` 占位。

9段权限的中间3个表示文件属组内的成员对文件拥有的读写执行权限，没有的位置使用 `-` 占位。

9段权限的后3个表示其他人对文件拥有的读写执行权限，没有的位置使用 `-` 占位。



### 设置文件权限

**使用加减号**
~~~bash
# 给属主增加写和执行权限
# 给属组增加执行权限
# 给其他人增加执行权限
chmod u+wx,g+x,o+r abc.txt

# 给属主减去写和执行权限
# 给属组减去执行权限
# 给其他人减去执行权限
chmod u-wx,g-x,o-r abc.txt
~~~

**直接赋权**
~~~bash
# 属主有读写和执行权限
# 属组有读和执行权限
# 给其他人只有读权限
chmod u=rwx,g=rx,o=r abc.txt

# 使用 a 表示全部的意思
# 设置文件对所有人全部为只读
chmod a=r abc.txt		#等价于 chmod u=r,g=r,o=r abc.txt

# 设置文件对所有人没有任何权限
chmod a=- abc.txt		#等价于 chmod a=--- abc.txt
~~~

**使用数字**
~~~bash
chmod 777 abc.txt
# 表示 u 有 r w x 的权限（r+w+x==4+2+1 == 7）
# 表示 g 有 r w x 的权限（r+w+x==4+2+1 == 7）
# 表示 o 有 r w x 的权限（r+w+x==4+2+1 == 7）
~~~

**设置文件夹权限**

~~~bash
chown -R 777 data/		# -R 表示递归设置所有子文件和子文件夹
~~~



### 总结
- 二进制文件想要执行只需 `x` 权限。
- 脚本文件想要执行需要 `r+x`，因为脚本解释器需要先读内容再执行命令。
- 对于操作文件夹 `/a/b/c` ，必须对沿途所有文件夹有 `x` 的权限，想要查看 c文件夹的内容需要对c文件夹有 `r` 的权限，想要在c文件夹中移动/删除/创建文件等需要对c文件夹有 `w` 权限。
- 对于操作文件 `/a/b/c/1.txt`，必须对沿途所有文件夹有 `x` 的权限，想要读 `1.txt` 内容需要对它有 `r`的权限，想要给它写数据需要对它有 `w` 权限，想要执行它需要有 `r+x` 权限。





## 特殊权限

### suid

设置在 二进制命令 上的权限。好处就是任何用户使用这个命令时表现的有效属主是这个命令的属主。实现的效果是在不改变文件权限的条件下，允许其他人使用这个命令的功能。具备这个权限的命令有：`passwd`、`su` 等。
~~~bash
[root@node1 ~]# ls -l `which passwd`
-rwsr-xr-x 1 root root 27856 Apr  1  2020 /bin/passwd
~~~

可以看到 `passwd` 的属主的权限为 `rws` ，其中的 `s` 就是这个权限的表达形式。如果想把一个命令设置为这个权限，可以使用 `chmod u+s`。
~~~bash
[root@node1 ~]# ls -l `which cat`
-rwxr-xr-x 1 root root 54080 Nov 17  2020 /bin/cat
[root@node1 ~]# chmod u+s /bin/cat
[root@node1 ~]# ls -l /bin/cat
-rwsr-xr-x 1 root root 54080 Nov 17  2020 /bin/cat
[root@node1 ~]# chmod u-s /bin/cat
[root@node1 ~]# ls -l /bin/cat
-rwxr-xr-x 1 root root 54080 Nov 17  2020 /bin/cat
~~~

**注意**：权限 `s` 能使用的前提是命令本身具备 `x` 权限，如果没有 `x` 权限，使用 `chmod u+s` 后，命令的权限表现形式是 `S`，但是没有执行的能力。



### sgid

said 权限有两个用法，**第一用在命令上**，和 suid 类似，实现的功能就是任何人执行该命令，该命令在运行时表现的有效属组是这个命令自己的属组。 **第二用在文件夹上**，实现的效果是任何人在这个文件夹下新建的子文件/子文件夹的属组都和这个文件夹的属组保持一致。

~~~bash
[root@node1 ~]# ls -ld /aaa
drwxr-xr-x 2 root root 4096 Aug 15 14:44 /aaa
[root@node1 ~]# chmod g+s /aaa
[root@node1 ~]# ls -ld /aaa
drwxr-sr-x 2 root root 4096 Aug 15 14:44 /aaa

# 以后任何人在/aaa文件夹下新建的子文件/夹 的属组都是 root (因为/aaa的属组是root)
~~~



###  sticky

在 Linux 中，**Sticky Bit（粘滞位）** 是一种特殊的文件权限设置，主要用于**目录**，其核心作用是**限制用户只能删除或重命名自己拥有的文件**，即使该目录对所有用户开放写权限（`rwx`）。例如：根下面的 `/tmp` 文件夹就具备权限 `rwxrwxrwt`。

当目录设置了 Sticky Bit 后，其权限标志会显示为 `t` 或 `T`（取决于是否同时有执行权限 `x`）：
- `rwxrwxrwt`（有执行权限 `x`，小写 `t`）
- `rwxrwxrwT`（无执行权限 `x`，大写 `T`）

~~~bash
[root@node1 ~]# mkdir /share
[root@node1 ~]# chmod 777 /share/
[root@node1 ~]# chmod o+t /share/
[root@node1 ~]# ls -ld /share/
drwxrwxrwt 2 root root 4096 Aug 15 15:51 /share/
[root@node1 ~]# su - liuxu -c 'touch /share/1.txt'
[root@node1 ~]# su - jack -c 'touch /share/2.txt'
[root@node1 ~]# ls -l /share/
total 0
-rw-rw-r-- 1 liuxu liuxu 0 Aug 15 15:51 1.txt
-rw-rw-r-- 1 jack  jack  0 Aug 15 15:51 2.txt
[root@node1 ~]# 
[root@node1 ~]# su - liuxu -c 'rm -f /share/2.txt'
rm: cannot remove ‘/share/2.txt’: Operation not permitted
[root@node1 ~]# su - liuxu -c 'rm -f /share/1.txt'
[root@node1 ~]# su - liuxu -c 'echo 111>> /share/2.txt'
-bash: /share/2.txt: Permission denied
[root@node1 ~]# 
~~~



### umask

在Linux中，`umask`（用户文件创建掩码）是一个重要的权限管理机制，用于**控制新创建文件或目录的默认权限**。它通过“屏蔽”（屏蔽）不需要的权限位，确保新文件或目录的权限符合系统安全要求。

**目录**的默认权限：`777 - umask`（例如 `umask=022` → 目录权限为 `755`）

**文件**的默认权限：`666 - umask`（例如 `umask=022` → 文件权限为 `644`）



## su 切换用户

`su`（Switch User）命令用于切换当前用户身份到另一个用户，通常用于切换到 `root` 用户或其他普通用户。

**基本语法**

```bash
su [选项] [用户名]
```

- **不指定用户名**：默认切换到 `root`（需要输入 `root` 密码）。
- **指定用户名**：切换到目标用户（需要输入该用户的密码）。

**常用选项**

| 选项        | 说明                                                         |
| :---------- | :----------------------------------------------------------- |
| `-` 或 `-l` | 模拟完整登录（加载目标用户的环境变量，如 `~/.bashrc` 和 `PATH`）。 |
| `-c`        | 以目标用户身份执行**单条命令**后退出（不进入交互式 shell）。`su -l liuxu -c "pwd"` |
| `-s`        | 指定要使用的 shell（如 `su -s /bin/bash alice`）             |
| `-m`        | 保持当前环境变量（不加载目标用户的环境）。                   |



**登陆级别的shell**。比如：login、`su - liuxu` 。进入会按顺序加载一些文件

~~~bash
# 系统全局配置
/etc/profile						# 设置所有用户共享的环境变量（如PATH、LANG）。
/etc/profile.d/*.sh			# 由/etc/profile调用的脚本目录

# 用户级配置文件
~/.bash_profile					# 优先级最高，若存在则忽略其他
~/.bash_login						# （次优先）	
~/.profile							# （最后兜底）
~/.bashrc								# 通常由~/.bash_profile或~/.profile显式调用
~~~



**非登陆级别的shell**。比如：bash直接进入或su liuxu。进入会按顺序加载一些文件

~~~bash
~/.bashrc
/etc/bashrc
/etc/profile.d/*.sh
~~~

对于环境变量的永久添加，可以把下面的文本写在 `/etc/profile` 文件中。只要用户就会加载所有的环境变量。

~~~bash
PATH = $PATH:/your/now/path
export PATH
~~~

> 补充，`export` 的效果是子shell中也可以有PATH



## sudo 提权

sudo 可以让普通用户临时获取某些管理员权限才能执行的命令。这些命令都是被提前配置好的，配置在文件 `/etc/sudoers` 中。有一个专门的命令 `visudo` 用来编辑该文件，并且可以使用 `visudo -c` 检查配置文件格式是否正确。

~~~bash
root   ALL=(ALL)     ALL
liuxu  ALL=(ALL)     ALL							# liuxu使用可以在所有命令前面使用sudo 获取root权限
jack   ALL=(ALL)     NOPASSWD:ALL     # sudo 时不用输入密码
mack   ALL=(ALL)     /usr/bin/cat,/bin/touch	# mack使用sud只能提权2个命令	
user01 ALL=(ALL)		ALL,!/usr/bin/vim					# !表示取反。这里表示提权所有命令但是除了vim

%dev   ALL=(ALL)     ALL		# dev组的用户提权
~~~

>补充，sudo时配置 `NOPASSWD`时需要输入用户自己的密码。输入一次后就会被缓存，第二次使用时就不再需要输入密码。想要清理缓存使用命令 `sudo -k`



