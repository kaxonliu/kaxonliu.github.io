# 用户管理

## 基本概念

Linux系统支持多用户，那么对于用户就需要做管理，比如管理文件的归属、文件的读写执行权限、进程的归属权限的。为了管理用户的权限，Linux设计了用户、组、权限等概念。Linux系统中每个用户都有一个uid，默认超级管理员的 uid 是0，通常超管只有一个。系统用户的 uid 是1-999。普通用户的 uid 从1000开始。



用户有自己的权限，用户属于的组有组的权限，其他人有其他权限。下面展示的是文件权限信息。

~~~bash
[root@node1 ~]# ls -l
-rw-r--r-- 1 root root 508 Aug 12 10:09 pp
~~~

比如：pp 文件的权限信息有两部分：

`-rw-r--r--` ，其中：

- 第一个字符 `-` 表示 pp 是一个普通文件，
- 第一组 `rw-` 表示这个文件的创建者（属主）对它的读写权限，没有执行权限使用 `-` 占位
- 第二组 `r--` 表示这个文件的创建这所在的组（属组）对它有读的权限，没有写和执行权限
- 第三组 `r--` 表示除了属主和属组内的用户，其他人对它只有读的权限

`root root`，其中的第一个 root 表示文件的属主是root用户，第二个 root 表示文件的属组是root组





## 用户管理的命令

**新增用户**

~~~bash
useradd liuxu
~~~

**设置密码**

~~~bash
passwd liuxu
~~~

**新增组**

~~~bash
groupadd dev
~~~

**把用户追加到一个组中**

~~~bash
usermod -a -G hr liuxu

# 把用户 liuxu 增加到 hr 组中
# -a 表示给用户追加到组中
# -G 指定组
~~~

**查看用户属于哪些组**

~~~bash
id liuxu
~~~

**把用户从组中移除**

~~~bash
# 方式1
usermod -G dev,sre liuxu			# 把用户重新设置新组，这种方式会覆盖原组

# 方式2
gpasswd -d liuxu hr						# 把liuxu从 r组中删除
~~~

**删除用户**

~~~bash
userdel -r liuux							# -r 彻底删除用户，推荐使用 -r
~~~

**强制用户下线**

~~~bash
pkill -KILL -u liuxu
~~~







**用户管理指令的本质都是编辑文件**。

| 文件              | 说明                           |
| ----------------- | ------------------------------ |
| `/etc/passwd`     | 存放用户详细信息               |
| `/etc/shadow`     | 存放用户密码                   |
| `/etc/group`      | 存放组信息                     |
| `/etc/gshadow`    | 存放组密码                     |
| `/etc/skel`       | 存放用户家目录下隐藏文件的模版 |
| `/var/spool/mail` | 存放用户邮箱                   |



**useradd 命令常用选项**

| **选项**                  | **说明**                                                     |
| :------------------------ | :----------------------------------------------------------- |
| `-u, --uid UID`           | 指定用户的 UID（用户 ID），如 `-u 1001`。                    |
| `-g, --gid GID`           | 指定用户的**主组**（Primary Group），可以是组名或 GID。      |
| `-G, --groups GROUPS`     | 指定用户的**附加组**（Supplementary Groups），多个组用逗号分隔（如 `-G wheel,dev`）。 |
| `-d, --home HOME_DIR`     | 指定用户的主目录路径（如 `-d /data/john`）。                 |
| `-m, --create-home`       | **自动创建用户主目录**（默认不创建，需配合 `-d` 或使用默认 `/home/用户名`）。 |
| `-s, --shell SHELL`       | 指定用户的登录 Shell（如 `-s /bin/bash`，默认是 `/bin/sh`）。 |
| `-c, --comment COMMENT`   | 添加用户备注（通常是全名或描述，如 `-c "John Doe"`）。       |
| `-e, --expiredate DATE`   | 设置账户过期日期（格式 `YYYY-MM-DD`），过期后用户无法登录。  |
| `-f, --inactive INACTIVE` | 密码过期后的宽限天数（`0` 立即失效，`-1` 禁用此功能）。      |
| `-N, --no-user-group`     | **不创建与用户同名的组**（默认会创建，如用户 `john` 对应组 `john`）。 |
| `-p, --password PASSWORD` | **设置加密后的密码**（需用 `crypt` 加密的字符串，不安全，建议用 `passwd` 命令）。 |



## 组管理的命令

**新增组**

~~~bash
groupadd dev		# 新增 dev 组
~~~

**修改组**

~~~bash
groupmod -n dev development			# -n 修改组名，把dev修改为development	
~~~

**删除组**

~~~bash
groupdel -r development
~~~

**增加组内的用户**

~~~bash
# 增加一个用户入组
gpasswd -a liuxu it

# 批量增加多个用户入组
gpasswd -M liuxu,jack it
~~~

**从组中删除用户**

~~~bash
gpasswd -d liuxu hr						# 把liuxu从 r组中删除
~~~

