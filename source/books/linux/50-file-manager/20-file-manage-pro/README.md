# vi-sed-awk-grep



## vi 和 vim

vi 编辑器是基础文本编辑器，vim 是它的增强版。vi 编辑文本的原理是**一次性从硬盘读取内容，加载到内存进行编辑，然后保存到硬盘**。从它的原理可以知道 vi 适合小文件。



### 三种模式

vi 使用有三种模式，分别是：命令行模式、插入模式、末行模式。

**命令模式（Command Mode）**

- 默认进入 vi 时的初始模式。
- 功能：执行编辑器命令（如复制、粘贴、删除、移动光标等），不能直接输入文本。
- 从其他模式切回命令模式，按 `Esc`。

| **快捷键** | **功能**             | **示例**               |
| :--------- | :------------------- | :--------------------- |
| `dd`       | 删除当前行           | 按 `dd` 删除光标所在行 |
| `3dd`      | 删除当前行往下的3行  |                        |
| `yy`       | 复制当前行           | 按 `yy` 复制，`p` 粘贴 |
| `3yy`      | 复制当前行往下的3行  |                        |
| `p`        | 在光标下一行粘贴内容 |                        |
| `0`        | 数字0光标跳到行首    |                        |
| `$`        | 光标跳到行尾         |                        |
| `gg`       | 光标跳到文件首行     |                        |
| `G`        | 光标跳到文件尾行     |                        |
| `3G`       | 光标跳到第3行        |                        |
| `H`        | 光标跳到屏幕的首行   |                        |
| `M`        | 光标跳到屏幕的中间   |                        |
| `L`        | 光标跳到屏幕的尾行   |                        |
| `u`        | 撤销操作             |                        |

**插入模式（Insert Mode）**
- 功能：直接编辑文本内容。
- 进入方式：在命令模式下按 `i`（光标前插入）、`a`（光标后插入）、`o`（新行插入）等。
- 退出方式：按 `Esc` 返回命令模式。



**末行模式（Ex Mode 或 Last-Line Mode）**
- 功能：执行保存、退出、搜索替换等高级操作。
- 进入方式：在命令模式下按 `:`（冒号）。

| **命令**              | **功能**                                | **说明**                                   |
| :-------------------- | :-------------------------------------- | :----------------------------------------- |
| `:q`                  | 退出 vi                                 | 按 `i` 后开始输入文本                      |
| `:w`                  | 保存                                    | 按 `dd` 删除光标所在行                     |
| `:wq`                 | 保存并退出，等价于 `x`                  | 按 `yy` 复制，`p` 粘贴                     |
| `:!`                  | 强制执行命令                            | 输入 `:q！` 强制退出、`wq!` 强制保存并退出 |
| `:set nu`             | 显示行号                                |                                            |
| `:set nonu`           | 不显示行号                              |                                            |
| `:set hlsearch`       | 设置查询高亮                            |                                            |
| `:set nohlsearch`     | 取消查询高亮                            |                                            |
| `:/查找的字符`        | 查找字符                                | 按 `n` 查找下一个，按 `N` 查找上一个。     |
| `:s/<old>/<new>/`     | 把第1行的内容替换，只替换匹配到的第一个 |                                            |
| `:s/<old>/<new>/g`    | 匹配的全部都替换，使用 `g`              |                                            |
| `:2s/<old>/<new>/g`   | 把第2行的内容做替换                     |                                            |
| `:%s/<old>/<new>/g`   | 把所有行的内容做替换                    |                                            |
| `:/^root/s/bash/ash/` | 使用正则定位，然后做替换操作            |                                            |





## sed

sed（Stream EDitor，流编辑器）是一个非交互式的文本编辑器，它按行处理输入流（文件或管道输入），执行指定的编辑操作，然后输出结果。其核心工作原理如下：

1. **逐行处理**：sed每次从输入中读取一行到内存；
2. **执行命令**：对内存中的内容执行指定的命令，比如：打印\替换\删除等；
3. **输出结果**：除非指定不输出（使用 `-n` 选项），否则处理后的内容会默认输出（如果内容被删除了则不输出）；
4. **循环处理**：重复上述过程直到所有行处理完毕



### sed 命令格式

```bash
sed [选项] '定位规则 + 命令' 输入文件
```

### 常用选项

| 选项 | 说明                                           |
| :--- | :--------------------------------------------- |
| `-n` | 取消默认输出行为                               |
| `-i` | 直接修改文件内容（慎用，建议先不加 `-i` 测试） |
| `-e` | 指定要执行的命令，可指定多个                   |
| `-r` | 使用扩展正则表达式                             |
| `-f` | 指定包含sed命令的脚本文件                      |

### 定位规则

**按行匹配**

- 只匹配第一行然后打印
  ~~~bash
  sed -n '1p' /etc/passwd
  ~~~
- 连续匹配第一行到第五行
  ~~~bash
  sed -n '1,5p' /etc/passwd
  ~~~
- 只匹配最后一行
  ~~~bash
  sed -n '$p' /etc/passwd
  ~~~
- 连续匹配第3行到最后一行
  ~~~bash
  sed -n '3,$p' /etc/passwd
  ~~~

**使用正则匹配**
- 正则匹配的固定结构为：`//`，比如匹配以 `bash` 结尾的行
  ~~~bash
  sed -n '/bash$/p' /etc/passwd
  ~~~


**指定行结合正则**
- 匹配从第3行开始到 `nologin` 结尾的所有行
  ~~~bash
  [root@rocky ~]# sed -n '3,/nologin$/p' /etc/passwd
  daemon:x:2:2:daemon:/sbin:/sbin/nologin
  adm:x:3:4:adm:/var/adm:/sbin/nologin
  
  # 找到第一个符合结尾标准的就结束
  ~~~

  

### 常用命令

| 命令         | 说明                                        |
| :----------- | :------------------------------------------ |
| `p`          | 打印                                        |
| `s///`       | 替换字符串，配合使用的flag 有 `i`、`p`、`g` |
| `d`          | 删除行                                      |
| `i\插入内容` | 在匹配行的前面插入一行                      |
| `a\追加内容` | 在匹配行的后面追加一行                      |
| `c\新内容`   | 匹配行的内容全部替换为新内容                |



### 常用示例

##### 替换文本

~~~bash
# root开头的行把root替换为 aaa
# 因为取消了默认输出，所以执行下面的命令后看不到替换后的文本
[root@rocky ~]# sed -n  '/^root/s/root/aaa/' /etc/passwd


# 打印替换后的文本 p
[root@rocky ~]# sed -n  '/^root/s/root/aaa/p' /etc/passwd
aaa:x:0:0:root:/root:/bin/bash

# 全部替换 g
[root@rocky ~]# sed -n  '/^root/s/root/aaa/pg' /etc/passwd
aaa:x:0:0:aaa:/aaa:/bin/bash

# 只替换指定位置的文本
# 如只替换第个写数字2
[root@rocky ~]# sed -n  '/^root/s/root/aaa/p2' /etc/passwd
root:x:0:0:aaa:/root:/bin/bash

# 忽略大小写 i
[root@rocky ~]# sed -n  '/^root/s/root/aaa/pgi' /etc/passwd
aaa:x:0:0:aaa:/aaa:/bin/bash
~~~

##### 删除行

~~~bash
sed '3d' 文件        		# 删除第3行
sed '1,5d' 文件     		# 删除1-5行
sed '/pattern/d' 文件 	# 删除匹配pattern的行
~~~

##### 插入

~~~bash
# 在指定的行前面插入 i
sed '3i\插入内容' 文件 

# 在指定的行后面插入 a
sed '3a\追加内容' 文件 
~~~

##### 替换整行文本

~~~bash
sed '3c\新内容' 文件    # 替换第3行内容
sed '/pattern/c\新内容' 文件 # 替换匹配行
~~~

##### 使用正则表达式匹配行

~~~bash
# 使用基本元字符集    
^, $, ., *, [], [^], \< \>,\(\),\{\}
 
# 使用扩展元字符集    
?, +, { }, |, ( )


# 匹配数字开头的行
sed -n '/^[0-9]+/p' abc.txt

# 把连续字母+连续数字的行做替换，替换为连续数字+连续字母
[root@rocky ~]# sed -nr 's/^([a-zA-Z]+)([0-9]+)/\2\1/p' abc.txt
12a
12223asdb
121111abc
~~~



##### 正则符号冲突需要转义

~~~bash
# 把以 sbin/nologin结尾的行中的 sbin/nologin 替换为 bin/bash
sed -n '/sbin\/nologin$/s/sbin\/nologin/bin\/bash/gp' file.txt
~~~



##### 自定义正则分隔符

~~~bash
# 使用自定义分隔符代替 //  任意字符都可以，但是需要先使用 \ 标识
# 比如使用 # 作为正则分隔符

sed -n '\#sbin/nologin$#s#sbin/nologin#/bin/bash#gp' file.txt
~~~



##### 多个编辑命令

- 方式1：把命令使用分号间隔
- 方式2：使用参数 `-e`

~~~bash
sed -n '1p;3p' abc.txt
sed -n -e '1p' -e '3p' abc.txt
~~~



##### 替换文件中的文本（先测试，确认无误再加 `-i`）

~~~bash
sed 's/old/new/g' file.txt          # 只输出到屏幕
sed -i 's/old/new/g' file.txt       # 直接修改文件
~~~



##### 删除空白行

~~~bash
sed '/^$/d' file.txt
~~~

##### 给文件每行都注释掉

~~~bash
sed 's/^/#/' filt.txt
~~~



##### 使用 sed 脚本

~~~bash
# 使用 sed.sh 中编写的匹配规则和命令
sed -f sed.sh file.txt

# sed.sh
1p;3p;5p;12d
~~~





### 模式空间和保持空间

sed 有两个内置的存储空间。模式空间用于 sed 执行的正常流程中，是一个缓冲区，用于存放、修改从输入文件中读取的内容。保持空间是另一个缓冲区，用来存放临时数据。sed 可以在两个空间交换数据，但是不能在保持空间上执行普通 sed 命令。

~~~bash
x：命令x(exchange) 用于交换模式空间和保持空间的内容
 
h：模式空间复制/覆盖到保持空间
H：模式空间追加到保持空间
 
g：保持空间复制/覆盖到模式空间
G：保持空间追加到模式空间
 
n：读取下一行到/覆盖到模式空间
N：将下一行添加到模式空间
 
d：删除pattern space中的所有行，并读入下一新行到pattern space中
~~~





### sed版本

sed 的版本有两大类：GNU sed  和 BSD sed，不同版本的sed可能有细微差异。Centos 和 Ubuntu 上都是 GNU sed；Mac OS 上是 BSD sed

查看 sed 的版本，`sed --version`
- GNU sed 展示信息
  ~~~bash
  sed (GNU sed) 4.2.2
  ~~~
- BSD sed 报错
  ~~~bash
  sed: illegal option -- -
  ~~~



### sed支持管道
~~~bash
ubuntu@master:~$ cat /etc/passwd | sed -n '/^liuxu/p'
liuxu:x:1001:1001::/home/liuxu:/bin/sh
~~~





### 注意 `-i` 选项的行为不同

GNU sed 和 BSD sed 在处理 `-i`（直接修改文件）时有不同。GNU sed：`-i` 可以直接使用，不需要备份。
```bash
sed -i 's/foo/bar/' file.txt
```


BSD sed（macOS）：`-i` 必须带一个备份扩展名（空字符串 `''` 表示不备份），如果直接 `sed -i 's/foo/bar/' file.txt`，会报错。

```bash
sed -i '' 's/foo/bar/' file.txt
```





## awk

aw k是一种强大的文本处理工具，其名称取自三位创始人Alfred Aho、Peter Weinberger和Brian Kernighan的姓氏首字母。它本质上是一种模式扫描和处理语言，**主要功能是根据匹配模式格式化输出**。awk 将每行输入视为由字段组成的记录，默认以空白字符（空格或制表符）作为字段分隔符。



### awk基本用法

~~~bash
awk -F  'pattern {action}' file
~~~
awk 支持管道输入，
~~~bash
command | awk　-F 'pattern {action}'
~~~

其中，
- `pattern` 是匹配模式，可以是正则表达式或者条件表达式
- `action`  是匹配后执行的命令，必须放在 `{}` 中
- `-F` 选项用来指定分隔符号，比如使用冒号分隔：`-F ':'`
- 注意：`pattern` 和 `action` 都是可选的。省略前者表示对文件的所有行执行 `action`，省略 `action` 则打印匹配到的行。

### 常用内置变量
| 变量        | 描述                |
| :---------- | :------------------ |
| `$0`        | 当前整行内容        |
| `$1` - `$n` | 当前行的第1-n个字段 |
| `NF`        | 当前行的字段数      |
| `NR`        | 当前处理的行号      |

### 常用命令和示例

##### 基本打印操作

| **命令**                    | **说明**         |
| :-------------------------- | :--------------- |
| `awk '{print $0}' file`     | 打印文件所有行   |
| `awk '{print $1, $3}' file` | 打印第1和第3列   |
| `awk '{print NF}' file`     | 打印每行的字段数 |
| `awk '{print NR, $0}' file` | 打印行号+内容    |

##### 条件过滤

| **命令**                             | **功能**                   |
| :----------------------------------- | :------------------------- |
| `awk '/pattern/' file`               | 使用正则匹配 `/pattern/`   |
| `awk 'NR==3' file`                   | 匹配到第3行                |
| `awk '$2 > 100' file`                | **指定字段做比较实现过滤** |
| `awk 'NR>=5 && NR<=10' file`         | 匹配到5-10行               |
| `awk '$1 ~ /^root/{print $0}' file`  | **指定字段做正则匹配**     |
| `awk '{if($3>300) {print $0}}' file` | **使用条件表达式**         |

##### 字段分隔符设置

| **命令**                         | **功能**        |
| :------------------------------- | :-------------- |
| `awk -F':' '{print $1}' /etc/passwd` | 以 `:` 为分隔符 |
| `awk -F'[,;]' '{print $1}' file` |   以 `,` 和 `;`为分隔符             |

##### 文本处理

| **命令**                           | **功能**             |
| :--------------------------------- | :------------------- |
| `awk '{print length($0)}' file`    | 计算每行长度         |
| `awk '{print toupper($1)}' file`   | 第1列转大写          |
| `awk '{gsub(/old/, "new")}1' file` | 替换文本             |
| `awk '!seen[$0]++' file`           | 去重（保留首次出现） |

##### 从文件读取规则和命令

~~~bash
awk -f awk_script_file file.txt

# awk_script_file
NR==1{print $1}
~~~







### 扩展了解

`awk` 的命令部分总共由三部分组成

~~~bash
BEGIN{}  {}  END{}
~~~

- 在 `BEGIN{}` 的花括号内编写所有行执行前执行的内容。比如设置分隔符 `BEGIN{OPS='---'}`
- 在 ` END{}` 的花括号内编写所有行结束的内容。比如打印`END{echo '结束啦'}`
- 中间的花括号，就是编写读一行处理一行的命令的位置。



### awk 格式化输出

~~~bash
# print
[root@rocky ~]# awk -F: '{print "用户名:", $1, "uid:", $3}' /etc/passwd | head -3
用户名: root uid: 0
用户名: bin uid: 1
用户名: daemon uid: 2


# printf
[root@rocky ~]# awk -F: '{printf "用户名:%s 用户id:%s\n",$1,$3}' /etc/passwd | head -3
用户名:root 用户id:0
用户名:bin 用户id:1
用户名:daemon 用户id:2

[root@rocky ~]# awk -F: '{printf "|%-15s| %-10s| %-15s|\n", $1,$2,$3}' /etc/passwd | head -3
|root           | x         | 0              |
|bin            | x         | 1              |
|daemon         | x         | 2              |

##
# %s 字符类型
# %d 数值类型
# 占15格的字符串
# - 表示左对齐，默认是右对齐
# printf默认不会在行尾自动换行，加\n
~~~





## grep

`grep`（**Global Regular Expression Print**）是 Linux/Unix 系统中最常用的文本搜索工具之一，用于在文件中查找匹配指定模式的行，并输出结果。它支持 **正则表达式**，功能强大，常用于日志分析、数据过滤等任务。`grep` **逐行读取输入文件（或标准输入），检查每一行是否满足匹配条件（可以是正则也可以是文本）。如果匹配成功，则输出该行**（默认行为）。



### grep基本用法

#### 命令格式
~~~bash
grep [选项] "模式" [文件1 文件2 ...]
~~~
grep 支持管道，
~~~bash
command | grep [选项] "模式"
~~~

**常用选项**

| **选项**    | **功能**                                                     |
| :---------- | :----------------------------------------------------------- |
| `-i`        | 忽略大小写                                                   |
| `-o`        | 只显示匹配的内容                                             |
| `-q`        | 静默输出不要输出，可以配合 `$?` 判断匹配成功与否             |
| `-v`        | **反向匹配**（输出不匹配的行）                               |
| `-n`        | 显示匹配行的行号                                             |
| `-c`        | 统计匹配的行数                                               |
| `-l`        | 仅显示包含匹配项的文件名，通常和 `-r` 一块使用。如 `grep -rl "root" /etc` |
| `-r` / `-R` | **递归搜索**（用于目录）                                     |
| `-w`        | 匹配整个单词（避免部分匹配）                                 |
| `-A n`      | 显示匹配行及其后 **n 行**                                    |
| `-B n`      | 显示匹配行及其前 **n 行**                                    |
| `-C n`      | 显示匹配行及其前后 **n 行**                                  |
| `-E`        | 等同于 `egerp` 扩展正则                                      |



#### 参数 n

~~~bash
# 显示行号
[root@rocky ~]# grep -n  "root" /etc/passwd
1:root:x:0:0:root:/root:/bin/bash
10:operator:x:11:0:operator:/root:/sbin/nologin
~~~



#### 参数 o

~~~bash
# 只显示匹配的内容
[root@rocky ~]# grep "liuxu" /etc/passwd
liuxu:x:1000:1000::/home/liuxu:/bin/bash
[root@rocky ~]# grep -o  "liuxu" /etc/passwd
liuxu
liuxu
~~~



#### 参数 r 和 l

~~~bash
# 只使用 r 会显示文件中匹配出的信息；配合 l 可以只显示文件名
[root@rocky ~]# grep -r "liuxu" /etc
/etc/group:liuxu:x:1000:
/etc/gshadow:liuxu:!::
/etc/passwd:liuxu:x:1000:1000::/home/liuxu:/bin/bash
/etc/shadow:liuxu:!!:20330:0:99999:7:::
/etc/subgid:liuxu:100000:65536
/etc/subuid:liuxu:100000:65536
[root@rocky ~]#
[root@rocky ~]# grep -rl  "liuxu" /etc
/etc/group
/etc/gshadow
/etc/passwd
/etc/shadow
/etc/subgid
/etc/subuid
~~~



#### 关闭过滤时的默认输出

~~~bash
# grep “nginx” 这个命令本身展示了如下信息
[root@rocky ~]# ps aux | grep "nginx"
root        6290  0.0  0.0   3584  1664 pts/1    S+   04:53   0:00 grep --color=auto nginx
[root@rocky ~]#


# 方式1
[root@rocky ~]# ps aux | grep "[n]ginx"
[root@rocky ~]# echo $?
1

# 方式2
[root@rocky ~]# ps aux | grep "nginx"|grep -v "grep"
[root@rocky ~]# echo $?
1
~~~

