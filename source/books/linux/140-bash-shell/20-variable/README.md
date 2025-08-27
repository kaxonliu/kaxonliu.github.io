# 变量

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
[root@rocky b]# [ 1 -eq 1 ]
[root@rocky b]# echo $?
0
[root@rocky b]# [ 1 -eq 2 ]
[root@rocky b]# echo $?
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

