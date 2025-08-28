# bash shell 入门

## 介绍

shell 是一个编程语言，解释型的、弱类型的、动态语言。shell 本身是一个语言，可以在 bash 解释器上执行，也可以在其他解释器上执行，比如 ash、sh等。和 Python 这种解释型语言一样，shell 语言也分为交互式和非交互式两种运行方式。交互式指的是每次接收一条命令然后立马返回结果。非交互式指的是每次接收一个文件，文件中可以有多条命令，shell 解释器会从上往下依次运行每个命令，最终得到一个结果。这样的文件我们称之为脚本程序（或者脚本文件），比如 shell 脚本、python 脚本等。

>补充，linux用户登陆时第一次执行的命令就是 /bin/bash，结果就是登陆进来就进入了一个交互式的程序中。这一点可以在文件 `/etc/passwd` 中看到。

**交互式和非交互式除了执行命令的方式不一样，也存在内置环境变量值的区别**。比如环境变量 `PS1` 在交互式环境下有值，在非交互式环境下没有值。因此也可以通过这个特点在代码层面判断当前是在那个模式下。

~~~bash
[root@centos ~]# echo $PS1
[\u@\h \W]\$
[root@centos ~]# cat > a.sh << "EOF"
> echo $PS1
> echo haha
> EOF
[root@centos ~]# chmod +x a.sh
[root@centos ~]# ./a.sh

haha
~~~



## 变量

shell 变量是用来存储数据的标识符，这些数据可以是数字、字符串、文件名等。shell 变量主要分为两种，

- **系统变量（环境变量）**：操作系统或 shell 自己创建和管理的变量，通常用于配置 shell 环境和系统信息。它们的名字通常是大写的。`HOME` 家目录、`PATH` 系统查找命令的目录列表、`USER` 当前登陆的用户名等等。
- **用户自定义变量**：由用户在脚本或命令行中创建和赋值的变量。



### 1. 定义变量

定义变量时，变量名、等号、值之间不能有空格。（不要被 Python 中 PEP8 规范影响哦）

~~~bash
[root@centos ~]# name="liuxu"
[root@centos ~]# age=18
~~~

### 2. 变量命名规则

- 命名只能使用英文字母、数字和下划线，首个字符不能是数字。
- 中间不能有空格，可以使用下划线 `_`。
- 不能使用标点符号。
- 不能使用 bash 里的关键字（可用 `help` 命令查看保留关键字）。

### 3. 引用变量

引用变量就是使用变量，使用一个定义过的变量，只要在变量名前面美元符号 `$` 即可。或者使用 `${变量名}`

~~~bash 
echo $name
echo ${name}

# 比如放在字符串中引用变量
[root@centos ~]# echo "my name is $name, i am ${age} years old"
my name is liuxu, i am 18 years old
~~~

**`${变量名}` 格式的优点：**
这种写法可以帮助解释器识别变量的边界，避免歧义。

~~~bash
[root@centos ~]# echo "$name is good"
liuxu is good
[root@centos ~]# echo "$nameis good"
 good
[root@centos ~]# echo "${name}is good"
liuxuis good
~~~

### 4. 删除变量

使用 `unset` 命令可以删除变量。变量被删除后不能再次使用。

```bash
unset 变量名
```

### 5. 只读变量

使用 `readonly` 命令可以将变量定义为只读，其值不能被改变，也不能被 `unset`（删除）。

~~~bash
readonly haha="hello"
~~~

### 6. 查看变量

- set 查看所有变量（包括自定义和环境变量）
- env 只看环境变量



## 变量的作用域

Shell 变量的默认作用域是**当前 Shell 会话**（当前脚本或当前终端窗口）。
- **局部变量**：在脚本或命令中定义，仅在当前Shell实例中有效，其他Shell启动的程序不能访问。
- **环境变量（全局变量）**：使用 `export` 命令可以将局部变量提升为环境变量，所有子Shell（由当前Shell启动的程序或脚本）都可以访问它。
如果想要环境变量在 shell 中都存在，可以把变量的定义放在用户登陆时系统默认执行的文件中。

~~~bash
/etc/profile
/etc/profile.d/*.sh
/etc/bashrc
~/.bash_profile
~/.bashrc
~~~



## 三种不同类型的引号

### 1. 双引号

双引号是弱引用，会把字符串中的变量替换为变量的值。

~~~bash
[root@centos ~]# echo "hello $name"
hello liuxu
~~~

### 2. 单引号

但引号是强引用，直接使用单引号中的文本，不做变量值替换。

~~~bash
[root@centos ~]# echo 'hello $name'
hello $name
~~~

### 3. 反引号

反引号可以获取引号中的结果。

~~~bash
[root@centos ~]# touch `date +%F`.txt
[root@centos ~]# ls
2025-08-23.txt
~~~

### 4. 嵌套使用

三种引号要配合使用，比如双引号包裹着单引号，不能双引号包裹双引号。





## 元字符

元字符就是 shell 语法中的一些特殊字符，它们在 shell 语法中有特殊的含义和用法。

### 1. 反引号和 `$()`

他俩都是取命令的结果，简单来说，就是把一个命令的输出，作为另一个命令的参数或者赋值给一个变量。**反引号是传统用法；`$()` 是现代用法，它比反引号更清晰，也更容易嵌套使用。**

~~~bash
[root@centos ~]# echo `date "+%F"`
2025-08-23


# 统计nginx访问日志，今日使用Chrome浏览器的访问次数，把次数打印出来
# 设置语言
LANG="en_US.UTF-8"

# $() 取结果，可以嵌套
echo $(grep $(date '+%d/%b/%Y') /var/log/nginx/access.log | grep "Chrome" | wc -l)

# `` 取结果，不可以嵌套
echo `grep $(date '+%d/%b/%Y') /var/log/nginx/access.log | grep "Chrome" | wc -l`


# 反引号不能嵌套使用，下面的的查询会报错
[root@centos ~]# echo `grep `date '+%d/%b/%Y'` /var/log/nginx/access.log | grep "Chrome" | wc -l`
Usage: grep [OPTION]... PATTERN [FILE]...
Try 'grep --help' for more information.
-bash: /var/log/nginx/access.log: Permission denied
date +%d/%b/%Y0
~~~



### 2.  `~`、`.`、`..`

这三个分别表示家目录，当前目录，上级目录。配合 `cd` 命令切换使用方便。



### 3. 取反符号 `!` 和 `^`

取反符号 `!` 用于过滤不满足匹配条件的数据，经常配配合 `find` 命令、`[]` 等使用。`^` 和 `!` 含义一样，但是只能用在 `[]`中间表示取反。

~~~bash
# 查询 /test文件夹下面不是txt文件的文件
find /test ! -name '*.txt'

# 展示/test文件夹里面的 jpg前面只有一个字符，但是非数字的jpg文件
ls /test/[!0-9].jpg
~~~



### 4. @ 、 #、$

`@` 用来当作分隔符号 ，本身无意义；

`#` 用于文件内容的注释，放在被注释文本的行首。

`$` 取变量值

~~~bash
x=1
echo $x
echo "x is ${x}"
count=$(grep $(date '+%d/%b/%Y') /var/log/nginx/access.log | wc -l)
~~~

### 5. 运算符 `%`、`+`、`-`、`*`、`/`

shell语法中使用 `(())` 数学运算的结果为整数

~~~bash
echo $((1+1))
echo $((4-1))
echo $((4*3))
# / 除法的结果为整数，向下取整
echo $((10/3))

# % 取余数
echo $((10%3))
~~~

可以使用 `bc` 命令实现浮点运算，通过 `scale` 参数设置结果的小数点位数。

~~~bash
# 非交互式使用 bc
[root@centos test]# echo "scale=2;10/3"|bc
3.33
~~~

使用 `$(())` 不支持浮点数。

~~~bash
[root@centos test]# echo $((10/3))
3
~~~

使用 let 方便赋值，不支持赋值，不支持直接输出。

~~~bash
[root@centos test]# let x=10/6
[root@centos test]# echo $x
1
~~~



### 6. 任意多个或零个字符 `*`

~~~bash
[root@centos test]# ls a*.txt
aaa.txt  ab.txt  a.txt
~~~



### 7. 在子shell中执行 `()`

想要在子shell中临时执行某些命令，又不想切到子 shell 中，可以在 `()` 中放入待执行的命令。

~~~bash
# xx在子shell中赋值，在当前shell中找不到
[root@centos test]# (xx=1)
[root@centos test]# echo $xx

# 临时在子shell中创建同一个特殊权限的文件
[root@centos test]# (umask 066;touch x.txt)
[root@centos test]# ls -l x.txt
-rw-------. 1 root root 0 Aug 23 21:31 x.txt
~~~



### 8. 下划线`_` 和 赋值符号 `=`

下划线无特殊意思，用在变量的命名上。`=` 赋值使用，或者判断想等使用。

~~~bash
# 判断两个是否相等，[]内需要使用空格把每个数据和符号分开
# echo $? 返回上一次判断的结果；如果相等，echo $? 返回的结果为0，否则返回1
[root@centos test]# [ 1 = 1 ]
[root@centos test]# echo $?
0
[root@centos test]# [ 1 = 2 ]
[root@centos test]# echo $?
1
[root@centos test]#
~~~

 

### 9. 管道符号 `|`

~~~bash
ps aux | grep python
 
# 不支持管道的命令需要借助 xargs 参数传递，把上一个命令的结果作为下一个命令的参数
find /home/ -type d -name "test*" |xargs ls
~~~



### 10. 特殊转译符号 `\`

新建文件或文件夹名字包含空格，可以使用 `\ ` 转译空格，也可以使用 双引号包括。但是**非常不建议文件名中包含空格。**

~~~bash
[root@centos test]# touch "a b".txt
[root@centos test]# touch aa\ bb.txt
[root@centos test]# ls -l
-rw-r--r--. 1 root root 0 Aug 23 21:45 aa bb.txt
-rw-r--r--. 1 root root 0 Aug 23 21:45 a b.txt
~~~



### 11. 连接多个命令的三个符号

三个连接命令的符号：

- `;` 效果是无论前一条命令运行成功与否，都会执行后一条命令。
- `&&` 效果是只有前一条命令执行成功，才会执行后一条命令。
- `||` 效果是前一条命令执行不成功，才会执行后一条命令。



### 12.  `&`、`/` 、`{}`

批量创建文件和文件夹使用 `{}`

~~~bash
touch data{1..4}.txt
~~~

后台运行使用 `&`；文件分隔符是左斜杠 `/`



### 13. 从定向

0标准输入、1标准正确输出、2标准错误输出，&标准正确和错误输出

~~~bash
pwd 1>a.txt
asasasqasq 2>a.txt
asasasasasa &>/dev/null
~~~

覆盖 `>`   追加 ` >>` 

~~~bash
[root@centos home]# cat >> a.txt << EOF
> 111
> 222
> 333
> EOF
~~~

< << 输入重定向

~~~bash
# mysql
mysql -uroot -p123 < bbs.sql

# 过滤
[root@centos home]# grep root < /etc/passwd
root:x:0:0:root:/root:/bin/bash
operator:x:11:0:operator:/root:/sbin/nologin
~~~



### 14. 任意一个字符 `?`

任意一个字符，言内意思是必须占位一个字符，字符是什么无所谓。

~~~bash
[root@centos ~]# ls ??.txt
aa.txt
[root@centos ~]# ls a?c.txt
a1c.txt
~~~



### 15. 范围中的任意一字字符 `[]`

`[]` 如果任意字符来自一段连续的符号，可以使用 `-`  连接；如果不是则直接填入有哪些可能的字符。

~~~bash
# 列出可能出现的 a.txt ~ z.txt
# 不分区字母的大小写
[root@centos ~]# ls [A-Z].txt
a.txt  A.txt


# 列出可能的 1.txt a.txt c.txt
[root@centos test]# ls [1ac].txt
1.txt  a.txt
~~~





### 16. 测试运算符 `[]`

命令 `test` 可以用来测试，支持测试的内容很多，比如 字符串比较、整形数字比较、文件夹/文件判断等等。在元字符中 `[]` 可以实现和 `test` 相同的效果，并且它俩使用的参数完全相同。即 `[]` 是 `test` 命令的元字符版本。不管是 `test` 命令，还是 `[]` 实现的测试，它们的测试结果都保存在变量 `$?` 上，值为0表示测试成功，值非0表示测试失败。 

#### 测试文件夹是否存在

~~~bash
# 使用 test 命令，判断文件夹 /etc 是否存在
[root@rocky ~]# test -d /etc; echo $?
0
[root@rocky ~]# test -d /ddddd; echo $?
1

# [] 版本的，需要注意保留符号两边的空格
[root@rocky ~]# [ -d /test ]; echo $?
1
~~~



#### 测试文件内容是否非空

~~~bash
[root@rocky ~]# touch a.txt; [ -s a.txt ]; echo $?
1
[root@rocky ~]# [ -s /etc/passwd ]; echo $?
0

# 等价于 test -s a.txt
~~~



#### 测试文件是否是标准文件

~~~bash
[root@rocky ~]# [ -f /etc/passwd ]; echo $?
0
[root@rocky ~]# [ -f /dev/null ]; echo $?
1
~~~



#### 测试字符串

| 符号                        | 含义                |
| --------------------------- | ------------------- |
| `[ a == b ]` 或 `[ a = b ]` | 判断 a和b是否相等   |
| `[ a != b ]`                | 判断 a和b不想等     |
| `[ -n var ]`                | 判断字符串长度不为0 |
| `[ -z var]`                 | 判断 字符串长度为0  |



#### 测试数字

| 符号                     | 含义               |
| ------------------------ | ------------------ |
| `[ a -eq b ]`            | 判断 a和b是否相等  |
| `[ a -ne b ]`            | 判断 a和b不想等    |
| `[ a -gt b ]`            | 判断 a 大于 b      |
| `[ a -ge b ]`            |                    |
| `[ a -lt b ]`            |                    |
| `[ a -le b ]`            |                    |
| `[ a -le b -a c -gt d ]` | 判断 a<=b 并且 c>d |
| `[ a -le b -o c -gt d ]` | 判断 a<=b 或者 c>d |



### 17. 测试运算符 `[[]]`

`[[]]` 和 `[]` 基本一样，不同的是前者支持正则匹配。不过需要注意在内层中括号内的左右两侧加空格。

`[[]]` 内使用正则表达式的标志是 `=~`，注意：**正则表达式不要加引号**。

~~~bash
# test 命令的方式，[[]] 完全等价于 []
[root@rocky ~]# [ "$USER" == "root" ];echo $?
0
[root@rocky ~]#
[root@rocky ~]# [[ "$USER" == "root" ]];echo $?
0


# [[]] 内部使用正则比较，正则表达式不能加引号!!!
[root@rocky ~]# [[ "$USER" =~ ^root$ ]];echo $?
0

# 判断 num 是否是整数
[root@rocky ~]# num=100;[[ $num =~ ^[0-9]+$ ]]; echo $?
0
~~~



