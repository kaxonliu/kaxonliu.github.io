# 流程控制



## if 分支判断

控制代码走向不同的分支，使用 `if` 关键词。可以充当条件判断的有很多，只要能表达出 **是** 或 **否** 两种结果就行。

### 完整语法

~~~bash
if 条件1; then
    # 条件为真时执行的命令
elif 条件2; then
    # 另一个条件为真时执行的命令
else
    # 所有条件都为假时执行的命令
fi
~~~

- 条件要以分号结尾，然后跟上 `then` 关键字。
- `elif` 关键字可以有0个或多个。
- 分支判断结束使用 `fi`。

### 条件表达式

#### 1. 使用 `[]`

在这个符号内可以使用 `test` 命令的参数用于条件判断。**使用它需要保证 `[` 和 `]` 前后必须有空格**。

~~~bash
if [ -d "directory" ]; then     # 目录存在
if [ "$str1" = "$str2" ]; then    # 字符串相等
if [ "$str1" != "$str2" ]; then   # 字符串不相等
if [ $a -eq $b ]; then    # 等于数字相等
if [ condition1 ] && [ condition2 ]; then   # 逻辑与
if [ condition1 ] || [ condition2 ]; then   # 逻辑或
if ! [ condition ]; then                    # 逻辑非
~~~

配合使用变量 **`$?`**，这个变量是 shell 预定义变量。它的值为0表示上一条命令执行成功，值非0表示执行失败。

~~~bash
[root@centos ~]# expr 10 + 12 &>/dev/null; echo $?	
0
[root@centos ~]# expr 10 + "abc" &>/dev/null; echo $?
2
~~~

如果判断的形式很简单，可以使用**短路运算，实现一行代码实现**。

~~~bash
[root@centos ~]# score=89;[ $score -gt 60 ] && echo '及格' || echo '不及格'
及格
[root@centos ~]# score=59;[ $score -gt 60 ] && echo '及格' || echo '不及格'
不及格
~~~

示例1：计算两个整数的加和。

~~~bash
[root@centos ~]# cat ping.sh
#!/bin/bash

ping -c2 -W1 $1 &>/dev/nul
if [ $? -eq 0 ]; then
    echo "ok"
else
   echo "failed"
fi
~~~

示例2：检测硬盘根分区的使用率

~~~bash
[root@centos ~]# cat disk_monitor.sh
#!/bin/bash
usage=$(df -P | grep '/$' | awk '{print $5}' | cut -d '%' -f 1)
if [ $usage -gt 10 ]; then
    echo "current disk usage is ${usage}, which exceed 10%"
fi
~~~

示例3：如果内存剩余空间少于阈值则报警。

~~~bash
[root@centos ~]# cat monitor_memory.sh
#!/bin/bash
available=$(free | awk 'NR==2 {print $NF}')
total=`free | awk 'NR==2 {print $2}'`
ava_percent=$(echo "scale=0;$available * 100 / $total" | bc)
if [ $ava_percent -lt 90 ]; then
    echo "警告,内存剩余可用大小为:${available}, 剩余率:${ava_percent}%"
fi
~~~

示例4：根据分数评级

~~~bash
[root@centos ~]# cat score_grade.sh
#!/bin/bash

score=$1

# check input a score
if [ $# -ne 1 ]; then
    echo "input a score please";
    exit;
fi

# check input a interge
expr 1 + $score &>/dev/null;
if [ $? -ne 0 ]; then
    echo "input a integer score please"
    exit;
fi

# grade
if [ $score -ge 90 ]; then
    echo "A"
elif [ $score -ge 80 ]; then
    echo "B"
elif [ $score -gt 70 ]; then
    echo "C"
else
    echo "D"
fi
~~~



#### 2. 使用 `[[]]`

双中括号是现代语法，推荐使用。它的用法和功能兼容 `[]`，同样的它内层的`[]` 和参数之间也要保留空格。并且在它的基础上支持 **正则表达式**。

~~~bash
if [[ $str == "pattern" ]]; then        # 模式匹配
if [[ $str =~ regex ]]; then            # 正则表达式匹配
~~~

示例1。`[]` 实现的条件判断，完全可以使用 `[[]]` 替代。

~~~bash
# 判断文件
if [[ -d /etc ]]; then
    echo "ok"
fi

# 判断数字大小
if [[ $a -ge $b ]]; then
    echo "ok"
fi
~~~

示例2。`[[]]` 内使用正则表达式。需要注意：**正则表达式不能被引号包裹**。

~~~bash
[root@centos ~]# cat test.sh
#!/bin/bash
# 判断第1个参数是否以 root开头
if [[ "$1" =~ ^root ]]; then
    echo "p1 ok"
fi
# 判断第2个参数是否是包含0-9的整数
if [[ $2 =~ ^[0-9]*$ ]]; then
    echo "p2 ok"
fi
~~~



#### 3. 使用 `(())`

C 语言风格的条件表达式，适用于整数比较大小。

~~~bash
if (( $a == $b )); then      # 数值相等
if (( $a < $b )); then       # 数值小于
if (( $a >= $b )); then      # 数值大于等于
~~~



## case 分支判断

能使用 `case` 的场合都可以使用 `if` 实现，能使用 `if` 的场合`case` 大多数实现不了。 但是， `case` 特别适合单变量值的枚举分支判断，比 `if` 用来的简洁。

###  完整语法

~~~bash
case var in
模式1)
    命令序列1
    ;;
模式2)
    命令序列2
    ;;
模式3|模式4)  # 多个模式用 | 分隔
    命令序列3
    ;;
*)  # 默认情况（匹配任何模式）
    默认命令序列
    ;;
esac
~~~

- 每个模式分支必须以 `;;` 结束
- 模式匹配是大小写敏感的。
- 可以使用通配符，其中 `*` 通常作为默认情况。

**注意：`case` 语句之支持 shell 通配符，比如 任意字符`*`、`?`，中括号中的任意一个字符 `[1-9 z-a]`等，但不支持正则表达式的处理。如需要使用正则表达式需要使用 `if [[ $var =~ regex ]]`**

示例

~~~bash
[root@centos ~]# cat tt.sh
#!/bin/bash

case $1 in
*.txt)
    echo "文本文件";;
*.jpg|*.png|*.gif)
    echo "图像文件";;
*.sh)
    echo "shell脚本";;
*.py)
    echo "python脚本";;
*)
    echo "其他类型的文件";;
esac
~~~



## while 条件循环

### 完整语法

条件为真执行循环体内的代码，条件为假退出循环。

~~~bash
while condition
do
    # 循环体代码
done

# 可以写写成如下形式
while condition ; do
    # 循环体代码
done
~~~

循环语句常配合两个关键词**`continue` 和 `break`** ，用于控制循环。

`continue` 结束本轮循环，立即进入下一轮循环。`continue` 语句后面的代码不再执行。

`break` 立即退出本层循环。



### 使用 while 循环逐行读取文件

~~~bash
while read line
do
    echo $line
done </etc/passwd
~~~



示例1：猜数字

~~~bash
[root@centos ~]# cat guess_num.sh
#!/bin/bash
num=$((RANDOM%100 + 1))
fail_count=0

while :;do
    read -p '请输入0-100之间的数字：' x
    [[ ! $x =~ ^[0-9]+$  ]] && echo  "必须输入数字" && continue

    if (( x == num  )); then
        echo '猜对了'
        break
    elif (( x > num  )); then
        echo '猜大了'
    else
         echo '猜小了'
    fi
    let fail_count++
    [ $fail_count -eq 3 ] && echo "猜错了三次你输了" && exit;
done
~~~

>补充知识点：使用 `$RANDOM` 生成 0-32767之间的随机数
>
>~~~bash
># 生成 0-32767 之间的随机数
>echo $RANDOM
>
># 生成指定范围的随机数（0-99）
>echo $((RANDOM % 100))
>
># 生成 1-100 的随机数
>echo $((RANDOM % 100 + 1))
>
># 生成 100-200 的随机数
>echo $((RANDOM % 101 + 100))
>~~~



示例2：红绿灯

~~~bash
#!/bin/bash 
clear
while :
do
    echo -e "\033[31m 红灯亮 \033[0m"; sleep 1; clear;
    echo -e "\033[32m 绿灯亮 \033[0m"; sleep 1; clear
    echo -e "\033[33m 黄灯亮 \033[0m"; sleep 1; clear
done
~~~



示例3：检测网页是否可以访问。

~~~bash
[root@centos ~]# cat tt.sh
#!/bin/bash
url=$1
timeout=$2
fails=0

while true; do
    curl -s -I --connect-timeout $timeout $url &>/dev/null
    if [ $? -eq 0  ]; then
        echo "页面访问成功"
        break
    else
        ((fails+=1))
        echo "fails: ${fails}"
        if (( $fails >=3  )); then
            echo "页面访问失败"
            break
        fi
    fi
done

# 使用
[root@centos ~]# bash tt.sh https://kaxonliu.github.io/ 1
fails: 1
页面访问成功
~~~



## until 条件循环

### 完整语法

它与 `while` 循环相反：**当条件为假时执行循环，直到条件为真时停止**。

~~~bash
until condition
do
    # 循环体代码
done

# 写成一行
until condition; do commands; done
~~~

`until` 依然可以配合 `continue` 和 `breal` 使用。

~~~bash
[root@centos ~]# cat num.sh
#!/bin/bash
count=0
until (( count > 5 ));do
    let count++
    if [ $count -gt 2 -a $count -lt 4  ]; then
        echo 'haha'
        continue
    fi
    echo "计数: ${count}"
done
~~~



## for 循环

for 循环的特点是迭代循环。不同于 while 按条件循环，for 按照迭代次数来循环。

### Bash风格的基本语法

~~~bash
for 变量 in 值列表
do 
    # 循环体代码
done

# 或者写在一行
for 变量 in 值列表; do  命令快; done
~~~

#### 遍历值列表

~~~bash
for fruit in apple banana orange
do
    echo "I like $fruit"
done
~~~

#### 遍历数字范围

~~~bash
# 使用 seq
for i in `seq 1 3`; do echo $i; done

# 指定步长
for i in $(seq 10 -2 0); do echo $i; done

# 使用 {a..b}
for i in {1..3}; do echo $i; done

# 指定步长
for i in {1..10..2}; do echo $i; done

# 数字倒序遍历
for i in {5..1}; do echo $i; done
~~~

#### 遍历命令输出

~~~bash
# 使用反引号取命令的返回结果
[root@rocky /]# for item in `ls /etc | head -3`; do echo $item; done
DIR_COLORS
DIR_COLORS.lightbgcolor
GREP_COLORS


# 使用 $() 取值
for user in $(cat /etc/passwd | cut -d: -f1)
do
    echo $user
done
~~~



### C语言风格的基本语法

~~~bash
for ((初值;条件;步长))
do 
    # 循环体代码
done
~~~

#### 遍历数字

~~~bash
for ((i=1;i<=3;i++))
do
    echo $i;
done
~~~

#### 指定步长

~~~bash
for (( i=0; i<10; i+=2)); do
    echo $i
done
~~~

#### 有初值可以简写

~~~bash
i=0
for ((;i<3;i++)); do
    echo $i
done
~~~

#### 无限循环

~~~bash
#!/bin/bash

# 无限循环（使用 break 退出）
for (( ; ; ))
do
    echo "Press Ctrl+C to stop"
    sleep 1
done
~~~



## select

`select` 是 shell 中用于创建简单文本菜单的强大工具，特别适用于交互式脚本。

#### 基本语法

~~~bash
select variable in list
do
    commands
done
~~~

#### 示例

~~~bash
#!/bin/bash

# 简单的选择菜单
echo "请选择你喜欢的水果:"
select fruit in "苹果" "香蕉" "橙子" "退出"
do
    case $fruit in
        "苹果")
            echo "你选择了苹果"
            ;;
        "香蕉")
            echo "你选择了香蕉"
            ;;
        "橙子")
            echo "你选择了橙子"
            ;;
        "退出")
            echo "再见!"
            break
            ;;
        *)
            echo "无效选择，请重新选择"
            ;;
    esac
done
~~~

**补充**：默认的提示符是 `#?`，需要修改可以使用 `PS3=请输入你的选择 (1-4):`

