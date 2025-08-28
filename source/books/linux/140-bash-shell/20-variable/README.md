# 变量和数据类型

变量要先定义后使用。定义变量很简单，语法格式为：`变量名=值`。需要注意的是等号两边不能有空格。变量名的定义有规范。

~~~bash
# 定义变量
name="liuxu"

# 引用变量
echo "$name"
echo "${name}"
~~~



## 变量名规范

~~~bash
# 变量名的命令应该见名知意，同时遵循如下规则
以字母或下划线开头，剩下的部分可以是：字母、数字、下划线，最好遵循下述规范：
    1.以字母开头
    2.使用中划线或者下划线做单词的连接
    3.同类型的用数字区分
    4.对于文件名的命名最好在末尾加上拓展名
 
例如: sql_bak.tar.gz,log_bak.tar.bz2  
    5、不要带有空格、？、*等特殊字符
    6、不能使用bash中的关键字，例如if，for，while，do等
    7、不要和系统环境变量冲突
~~~



## 变量值的三个来源

#### 1. 直接赋值

~~~bash
ip="192.168.10.111"
today=`date +%F`
today=$(date +%F)
~~~



#### 2. 位置参数获变量值

调用脚本时传入的位置参数可以在脚本文件中获取到，文件自己赋值到 `$0`，第1个参数赋赋值给 `$1`，第2个参数赋赋值给 `$2`，依此类推， 第 n 个参数分别赋值到 `$N`，需要注意的是，第10个参数开始必须使用 `${N}` 取值，前面的可以使用 `$N` 直接获取。

~~~bash
# cat test.sh
#!/bin/bash
echo "${0}, ${1}, ${2}, ${9}, ${10}"

# 执行脚本 
bash test.sh 1 2 3 4 5 6 7 8 9 10 11

# 打印机结果
test.sh, 1, 2, 9, 1
~~~



#### 3. 与用户交互取值

使用 `read` 命令接收用户输入的值，保存给变量。其中，

- 参数 `-p` 表示提示信息
- 参数 `-t` 指定等待用户输入的秒数
- 参数 `-n` 指定结束字符的长度

~~~bash
read -p "please input your name:>>>" name
eche $name	# 变量name接收用户输入的数据


# 等待 5 秒，获取输入的前10个字符
read -t 5 -n 10 name
~~~





## 特殊的预定变量

### `$*` 获取所有的位置参数

获取 shell 脚本执行时传入的所有位置参数，但是不包含脚本文件。

~~~bash
[root@rocky b]# cat test.sh
for i in $*;
do
    echo $i;
done
[root@rocky b]# bash test.sh 1 2 3 4 5 6
1
2
3
4
5
6
~~~

### `$@` 获取所有的位置参数

`$@` 和 `$*` 在绝大多数的情况下的功能都是一样，但是当位置参数中含有被引号包起来的带有空格的参数时，它来的效果就不一样了。

~~~bash
# 实验1 test.sh 使用 "$*"
for i in "$*";
do
    echo $i;
done
[root@rocky b]# bash test.sh 1 2 3 4 "hello world"
1 2 3 4 hello world


# 实验2 test.sh 把 "$*" 换成 "$@"
[root@rocky b]# bash test.sh 1 2 3 4 "hello world"
1
2
3
4
hello world
~~~



### `$#` 获取所有的位置参数的个数

获取 shell 脚本执行时传入的所有位置参数的总数，但是不包含脚本文件。

~~~bash
[root@rocky b]# cat test.sh
echo "$#"
[root@rocky b]# bash test.sh 1 2 3 4 "hello world"
5
~~~



### `$$` 表示当前进程的pid

### `$?` 上一个命令的返回值

~~~bash
[root@centos ~]# [ 1 -eq 1 ]; echo $?
0
[root@centos ~]# [ 1 -ne 1 ]; echo $?
1
~~~





## 常量

常量就是不变的量，设置为常量后就只能读不能修改。

~~~bash
x=111
readonly x

# 或者直接声明加赋值
[root@rocky b]# readonly name="liuxu"
[root@rocky b]# name="xxx"
-bash: name: readonly variable
~~~



## 整数的使用

### 数字算术运算

数字分为整数和浮点数，数字的算术运算使用5个算术运算符，分别是：加法 `+`、减法 `-`、乘法 `*`、除法 `/`、取余 `%`。算术运算符需要配合一些工具和元字符一块使用。

#### 1. 使用 bc 工具

`bc` 工具支持整数运算，也支持浮点数运算。如果没有需安装：`yum install bc -y`。它的使用方式是：**将包含计算表达式的字符串通过管道（`|`）传递给 `bc` 命令**。

**打印计算结果**

~~~bash
[root@centos ~]# echo "1+1"|bc
2
~~~

**计算结果赋值给变量**

~~~bash
[root@centos ~]# res=`echo 1+1|bc`; echo $res
2
[root@centos ~]# res=$(echo "1+1"|bc); echo $res
2
~~~

**浮点数计算控制精度**

~~~bash
[root@centos ~]# echo "10.1-3.3"|bc
6.8
[root@centos ~]# echo "scale=2;10/3"|bc
3.33
~~~

**注意**：`scale` 只影响除法（`/`）、取余（`%`）和乘方（`^`）运算，不影响加（`+`）、减（`-`）、乘（`*`）。



#### 2. 使用 `$(( ))` 

标准的 Shell 算术 `$(( ))` **只支持整数，不支持浮点数**。

~~~bash
[root@centos ~]# echo $((2*1))
2
[root@centos ~]# res=$((2+1)); echo $res
3
~~~



#### 3. 使用 `$[]`

用法同  `$(())`，只支持整数不支持浮点数运算。

~~~bash
[root@centos ~]# echo $[3*4]
12
[root@centos ~]# res=$[3+5];echo $res
8
~~~



#### 4. 使用 `expr`

它的用法是：`expr 表达式`。`expr` 会计算给出的表达式，并将结果打印到标准输出。需要注意 `expr` **只支持整数运算，并且数字和运算符中间要有空格**。

**输出表达式的计算结果**

~~~bash
[root@centos ~]# expr 3+4			# 没有空格字直接输出
3+4
[root@centos ~]# expr 3 + 4
7
~~~

**把表达式结果赋值给变量**

~~~bash
[root@centos ~]# res=`expr 3 + 4`; echo $res
7
[root@centos ~]# res=$(expr 4 - 2); echo $res
2
~~~

**特殊符号需要转译**

~~~bash
[root@centos ~]# expr 3 * 5
expr: 语法错误
[root@centos ~]# expr 3 \* 5
15
[root@centos ~]# res=$(expr 2 \* 2);echo $res
4
~~~

**使用 `expr` 判断变量是否为整数**。

~~~bash
[root@centos ~]# x=10
[root@centos ~]# y="hello"
[root@centos ~]# expr 1 + $x &>/dev/null; echo $?
0
[root@centos ~]# expr 1 + $y &>/dev/null; echo $?
2

# 因为 expr 不支持非整数运算，因此可以配合预定变量 $? 判断变量是否是整数。
# $? 返回0 表示 expr 执行成功，即变量是整数
# $? 返回不等于0 表示 expr 执行失败，即变量不是整数
~~~



#### 5. 使用 let 

命令 `let` 只支持整数，不支持浮点运算，而且不支持直接输出，只能把结果赋值给变量。

~~~bash
[root@centos ~]# let res=1+1; echo $res
2
[root@centos ~]# x=10; let res=x++; echo $x; echo $res
11
10
~~~

### 数字比较大小

#### 1. 使用 `test` 或 `[]`

可以使用 `test` ， 也可以使用符号 `[]`，两者在比较数字时使用的参数都是一样的，但是**需要注意 `[]` 内第一个数字左边空一格，第二个数字的右边空一格。**比较的结果存放在变量 `$?` 中。

**两种方式仅支持整数比较**。

~~~bash
# -eq 等于
# -ne	不等于
# -gt 大于
# -lt 小于
# -ge 大于等于
# -le 小于等于
# -a 并且
# -o 或者

[root@centos ~]# [ 10 -eq 3 ]; echo $?
1
[root@centos ~]# [ 10 -ne 3 ]; echo $?
0
[root@centos ~]# a=10;b=3
[root@centos ~]# [ $a -ge $b ]; echo $?
0
[root@centos ~]# [ $a -ge $b -a 3 -lt 5 ]; echo $?
0
[root@centos ~]# [ $a -lt $b -o 3 -gt 5 ]; echo $?
1
~~~

**使用示例**

~~~bash
# 检测 /root 挂载磁盘的使用率，如果大于20%则发出警告

disk_usage=$(df -P |grep '/$' | awk '{print $5}' |cut -d % -f 1 )
if [ $disk_usage -gt 10 ]; then
    echo "warn message!!!"
fi
~~~



#### 2. 使用 `(())`

使用 C 语言风格的关系运算符比较数字大小。配合使用的关系运算符有：`<`、`>` 、`<=`、`>=`、`==`、`!=` 、`&&` 、`||`。**只能用在整数比较**。

~~~bash
[root@centos ~]# ((1>=10)); echo $?
1
[root@centos ~]# ((1<10)); echo $?
0
[root@centos ~]# ((1>10 || 2<10)); echo $?
0
[root@centos ~]# ((1>10)) || ((2<10)); echo $?
0
[root@centos ~]# ((4>2)) && ((2<10)); echo $?
0
~~~



#### 3. 浮点数比较大小

浮点数比较大小需要使用 `bc`。判断正确 `bc` 返回1，判断错误返回0。

~~~bash
[root@centos ~]# echo "19.3 >= 5.8" | bc
1
[root@centos ~]# echo "19.3 < 5.8" | bc
0
~~~

结合 `[]` 判断

~~~bash
[root@centos ~]# [ `echo "10.3 > 19.2" | bc` -eq 1 ]; echo $?
1
~~~





### 算术复合赋值

#### 基本语法和含义

```bash
(( variable *= expression ))

# 完全等价于：
variable=$(( variable * expression ))
```

具体示例

```bash
[root@centos ~]# a=10; ((a+=5)); echo $a
15

# 完全等价于
a=10;a=$((a-4));echo $a
```

#### 其他常用的算术复合赋值运算符

| 运算符 | 示例            | 等价于            | 含义                       |
| :----- | :-------------- | :---------------- | :------------------------- |
| `+=`   | `(( a += 5 ))`  | `a=$(( a + 5 ))`  | **加后赋值**               |
| `-=`   | `(( a -= 3 ))`  | `a=$(( a - 3 ))`  | **减后赋值**               |
| `*=`   | `(( a *= 4 ))`  | `a=$(( a * 4 ))`  | **乘后赋值**               |
| `/=`   | `(( a /= 2 ))`  | `a=$(( a / 2 ))`  | **除后赋值**（整数除法）   |
| `%=`   | `(( a %= 3 ))`  | `a=$(( a % 3 ))`  | **取模后赋值**             |
| `<<=`  | `(( a <<= 1 ))` | `a=$(( a << 1 ))` | **左移后赋值**（按位操作） |
| `&=`   | `(( a &= 1 ))`  | `a=$(( a & 1 ))`  | **按位与后赋值**           |
| `^=`   | `(( a ^= 1 ))`  | `a=$(( a ^ 1 ))`  | **按位异或后赋值**         |



## 字符串的使用

字符串是被引号包起来的一串字符，如果字符串中没有空格，那可以不使用引号包裹（强烈建议加上引号），但是如果包含空格则必须使用引号。可以使用单引号和双引号，但是它俩的效果是不一样的。

### 单引号和双引号

**双引号是弱引用**，双引号内的特殊字符有意义。比如引用的变量会被替换到字符串中。

~~~bash
[root@centos ~]# name="liuxu"; echo "my name is ${name}"
my name is liuxu
~~~

**单引号是强引用**，单引号内的所有特殊字符都没有意义。

~~~bash
[root@centos ~]# name="liuxu";echo 'my name is ${name}'
my name is ${name}
~~~



### 获取字符串的长度

#### 1. 使用 `${# }`

它是 Shell 的内建功能，不需要启动外部命令

~~~bash
[root@centos ~]# name="liuxu"; echo ${#name};
5
~~~

#### 2. 使用 `wc -L`

~~~bash
[root@centos ~]# name="liuxu"; echo $name | wc -L;
5
~~~

#### 3. 使用 `awk`

~~~bash
[root@centos ~]# name="liuxu"; echo $name | awk '{print length}'
5
~~~

#### 4. 使用 `expr length`

`expr` 是一个外部命令，可以用于计算字符串长度。这是比较古老的方法，不推荐。

~~~bash
# length 是一个函数，如果name有空格，则必须使用引号把变量包起来
[root@centos ~]# name="liu xu"; expr length "$name"
6
~~~

