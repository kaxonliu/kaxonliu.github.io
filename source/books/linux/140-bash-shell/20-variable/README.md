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

# x=i++ 先把i的值赋值给x，然后i再++
# y=++j 先把j++，再把j的值赋值给y
[root@rocky ~]# i=1
[root@rocky ~]# j=1
[root@rocky ~]# let x=i++
[root@rocky ~]# let y=++j
[root@rocky ~]# echo $i; echo $j; echo $x; echo $y
2
2
1
2
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

浮点数比较大小需要使用 `bc`。判断正确返回1，判断错误返回0。

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



### 字符串切片 `${variable:start:length}`

#### 语法

~~~bash
${string:start:length}
~~~

- `start` 起始位置（从 0 开始）
- `length` 要提取子串的长度（可选）



#### 仅指定起始位置

仅指定起始位置，不指定长度，表示从起始位置切片到结尾。

~~~bash
[root@rocky ~]# msg='HelloWorld'; echo ${msg:2};
lloWorld
~~~

#### 指定切片长度

~~~bash
[root@rocky ~]# msg='HelloWorld'; echo ${msg:0:4};
Hell

# 其实位置是0的时候，可以简写为
[root@rocky ~]# msg='HelloWorld'; echo ${msg::4};
Hell
~~~

#### 使用变量切片

~~~bash
[root@rocky ~]# s=3;l=3;msg='HelloWorld';echo ${msg:$s:$l};
loW
~~~

#### 从右边开始切

从右边开始切需要在第一个 `:` 和 `start` 之间必须有空格。

~~~bash
[root@rocky ~]# msg='HelloWorld'; echo ${msg: -5:3};
Wor
~~~



### 字符串截断

#### 截断左边的字符 `${variable# }`

使用  `${变量名# } `截断左边的字符。`#` 后面跟上需要截断的子串。

~~~bash
[root@rocky ~]# url='www.sina.com.cn'; echo ${url#www.}
sina.com.cn
[root@rocky ~]# url='www.sina.com.cn'; echo ${url#*c}
om.cn
~~~

使用 `${变量名#* }` 非贪婪截断。截断到第一个子串。

~~~bash
[root@rocky ~]# url='www.sina.com.cn'; echo ${url#*w}
ww.sina.com.cn
~~~

使用 `${变量名##* }` 贪婪截断。截断到最后一个子串。

~~~bash
[root@rocky ~]# url='www.sina.com.cn'; echo ${url##*w}
.sina.com.cn
[root@rocky ~]# url='www.sina.com.cn'; echo ${url##*c}
n
~~~

#### 截断右边的字符  `${variable% }`

使用  `${变量名% } `截断左边的字符。`%` 后面跟上需要截断的子串。

~~~bash
[root@rocky ~]# url='www.sina.com.cn'; echo ${url%.cn}
www.sina.com
~~~

使用 `${变量名%*}` 非贪婪截断。截断到第一个子串。

~~~bash
[root@rocky ~]# url='www.sina.com.cn'; echo ${url%.*}
www.sina.com

# %.* 从右边开始找到第一个 . 然后把右边的(.cn)截掉
~~~

使用 `${变量名%%*}` 贪婪截断。截断到最后一个子串。

~~~bash
[root@rocky ~]# url='www.sina.com.cn'; echo ${url%%.*}
www

# %%.* 从右边开始找，找到最后一个 . 然后把右边的(.sina.com.cn)截掉
~~~



### 字符出替换

#### 普通替换 `${variable/old/new}`

~~~bash
[root@rocky ~]# url='www.sina.com.cn'; echo ${url/n/N}
www.siNa.com.cn
~~~

#### 贪婪替换 `${variable//old/new}`

~~~bash
[root@rocky ~]# url='www.sina.com.cn'; echo ${url//n/N}
www.siNa.com.cN
~~~



### 字符串默认值替代

#### 1. `${var-default}`

当变量仅未定义时返回默认值，否则都返回变量自己的值（即变量值为空，也返回空）。不改变变量的值。

~~~bash
[root@rocky ~]# unset name;echo ${name-"liuxu"};echo $name;
liuxu

[root@rocky ~]# name="";echo ${name-"liuxu"};echo $name;


[root@rocky ~]# name="kaxon";echo ${name-"liuxu"};echo $name;
kaxon
kaxon
~~~

#### 2. `${var:-default}`

当变量存在且值非空则返回变量的值，当变量未定义或值为空则返回默认值。不改变变量的值。

~~~bash
[root@rocky ~]# unset name;echo ${name:-"liuxu"};echo $name;
liuxu

[root@rocky ~]# name="";echo ${name:-"liuxu"};echo $name;
liuxu

[root@rocky ~]# name="kaxon";echo ${name:-"liuxu"};echo $name;
kaxon
kaxon
~~~

#### 3. `${var:=default}`

当变量存在且值非空，则返回变量的值，当变量为定义或值为空则返回默认值，并修改变量的值为默认值。

~~~bash
[root@rocky ~]# unset name;echo ${name:="liuxu"};echo $name;
liuxu
liuxu
[root@rocky ~]# name="";echo ${name:="liuxu"};echo $name;
liuxu
liuxu
[root@rocky ~]# name="kaxon";echo ${name:="liuxu"};echo $name;
kaxon
kaxon
~~~

#### 4. `${var:?eror_msg}`

检查变量的值是否为空。当变量存在且值非空则返回变量的值，否则在终端打印错误信息。不修改变量值。

~~~bash
[root@rocky ~]# unset name;echo ${name:?"没有值啊"};echo $name;
-bash: name: 没有值啊
[root@rocky ~]# name="";echo ${name:?"没有值啊"};echo $name;
-bash: name: 没有值啊
[root@rocky ~]# name="kaxon";echo ${name:?"没有值啊"};echo $name;
kaxon
kaxon
~~~

#### 5. `${var:+replacement}`

替代非空值。当变量存在且值非空则返回替换值，否则返回空。不修改变量值。

~~~bash
[root@rocky ~]# unset name;echo ${name:+"liuxu"};echo $name;


[root@rocky ~]# name="";echo ${name:+"liuxu"};echo $name;


[root@rocky ~]# name="kaxon";echo ${name:+"liuxu"};echo $name;
liuxu
kaxon
~~~



#### 总结表

| 表达式                | 含义                     | 如果 `var` 已设置且非空 | 如果 `var` 未设置或为空                         |
| :-------------------- | :----------------------- | :---------------------- | :---------------------------------------------- |
| `${var:-default}`     | 使用默认值               | 返回 `$var` 的值        | 返回 `default`                                  |
| `${var-default}`      | 使用默认值（仅未定义时） | 返回 `$var` 的值        | 返回 `default` (**空值仍返回空**)               |
| `${var:=default}`     | 赋值并使用默认值         | 返回 `$var` 的值        | 将 `var` 的值设为 `default`，然后返回 `default` |
| `${var:?error_msg}`   | 检查变量是否为空         | 返回 `$var` 的值        | 在 stderr 打印 `error_msg`，并**退出脚本**      |
| `${var:+replacement}` | 替代非空值               | 返回 `replacement`      | 返回空                                          |





## 数组的使用

数组就是一堆数据的集合，一个数组内可以存放多个元素，元素的类型的任意的。数组可以分为两种，一种是**普通数组**，另一种是**关联数组**。

### 定义普通数组

~~~bash
# 方式1
# 使用 () 把多个元素包起来，空格间隔
names=("liuxu" "kaxon" "jack")

# 方式2
# 直接在 () 内指定每个索引上的值，顺序无所谓
data=([0]=18 [1]="liuxu" [2]="male")

# 方式3
# 一个一个指定元素
array[0]="元素1"
array[1]="元素2"
array[2]="元素3"

# 方式4
# 通过命令的结果生成数组
nums=(`seq 1 10`)
~~~

### 定义关联数组

关联数组定义时需要使用 `declare -A` 声明。

~~~bash
declare -A info

info["age"]=18
info["name"]="liuxu"
info["gender"]="male"
~~~



### 查看数组

- 查看声明过的所有数组：`declare -a`
- 查看生命过的关联数组：`declare -A`
- 查找一个，配合使用 `grep`



### 访问数组元素

通过索引的方式访问数组内的元素。普通数组使用数字索引，访问时可以使用负数表示从后往前数。关联数组的索引是字符串那就使用字符串访问，如果关联数组索引中存在数字，就使用数字访问。

~~~bash
# 定义普通数组
nums=(`seq 1 10`)

# 访问第1个和倒数第2个元素
echo ${nums[1]}    # 输出 2
echo ${nums[-2]}    # 输出 9

# 访问所有元素，使用 * 或 @
echo ${nums[*]}    # 输出 1 2 3 4 5 6 7 8 9 10

echo ${nums[@]}    # 输出 1 2 3 4 5 6 7 8 9 10

# 获取数组长度，使用 # 配合 * 或 @
echo ${#nums[*]}    # 输出 10


# 定义关联数组
declare -A info
info=([gender]="male" [0]="IT" [age]="18" [name]="liuxu" )

# 通过索引访问数组元素
echo ${info['name']}    # 输出 liuxu
echo ${info[0]}    # 输出 IT
~~~



### 添加/修改数组元素

修改数组就是把数组的内元素按照索引的方式更新即可，适用普通数组和关联数组。

- 使用不存在的索引就是添加新元素。
- 使用已经存在的所以就是更新值。

~~~bash
nums[2]=300
info["age"]=19
info["score"]=100
~~~



### 删除数组元素

~~~bash
# 删除一个元素
unset nums[9]

# 删除整个数组
unset info
~~~



### 数组切片

**语法：`${array[*]:start:length}`**

示例：从第二个位置切，取4个元素，把切出来的数据放到一个新数组中

~~~bash
[root@rocky ~]# xxx=(${nums[*]:1:4})
[root@rocky ~]# declare -a | grep xxx
declare -a xxx=([0]="2" [1]="3" [2]="4" [3]="5")
~~~





### 遍历数组

#### 方式1: 直接遍历值

无论普通数组还是关联数组，获取数组的所有值然后使用 `for` 循环遍历即可。

~~~bash
for num in ${nums[*]}
do
    echo $num
done
~~~

#### 方式2：遍历索引然后取值

获取数组的所有索序列，然后依次取值。适合普通数组和关联数组。

~~~bash
for idx in ${!info[*]}
do 
    echo ${info[$idx]}
done
~~~



#### 方式3：根据数组长度生成索引值

这种方式只适用普通数组。原理：先获取 数组长度，然后生成索引值序列，依次遍历取值。

~~~bash
for ((i=0; i<${#nums[*]}; i++))
do
    echo ${nums[$i]}
done
~~~





### 示例：统计登陆shell的种类及其个数

~~~bash
#!/bin/bash

declare -A shell_info

while read line
do
     s=$(echo $line | awk -F ':' '{print $NF}')
     ((shell_info[$s]++))
done </etc/passwd

for k in ${!shell_info[*]}
do
    printf "%s %s\n" $k ${shell_info[$k]}
done
~~~



