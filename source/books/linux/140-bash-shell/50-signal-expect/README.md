# 信号处理与expect



## 捕获信号

想要查看有哪些信号，可以使用 `kill -l`，当 shell 脚本运行的过程中，如果收到这个信号，该如何处理呢？我们可以自定义如何处理接收的指定信号。除了两个信号无法捕获处理，其余的都可以自定义处理。`SIGKILL` 和 `SIGSTOP` 这两个特殊信号无法被进程捕获或忽略，即操作系统立即处理目标进程。

~~~bash
[root@rocky ~]# kill -l
 1) SIGHUP	 2) SIGINT	 3) SIGQUIT	 4) SIGILL	 5) SIGTRAP
 6) SIGABRT	 7) SIGBUS	 8) SIGFPE	 9) SIGKILL	10) SIGUSR1
11) SIGSEGV	12) SIGUSR2	13) SIGPIPE	14) SIGALRM	15) SIGTERM
16) SIGSTKFLT	17) SIGCHLD	18) SIGCONT	19) SIGSTOP	20) SIGTSTP
21) SIGTTIN	22) SIGTTOU	23) SIGURG	24) SIGXCPU	25) SIGXFSZ
26) SIGVTALRM	27) SIGPROF	28) SIGWINCH	29) SIGIO	30) SIGPWR
31) SIGSYS	34) SIGRTMIN	35) SIGRTMIN+1	36) SIGRTMIN+2	37) SIGRTMIN+3
38) SIGRTMIN+4	39) SIGRTMIN+5	40) SIGRTMIN+6	41) SIGRTMIN+7	42) SIGRTMIN+8
43) SIGRTMIN+9	44) SIGRTMIN+10	45) SIGRTMIN+11	46) SIGRTMIN+12	47) SIGRTMIN+13
48) SIGRTMIN+14	49) SIGRTMIN+15	50) SIGRTMAX-14	51) SIGRTMAX-13	52) SIGRTMAX-12
53) SIGRTMAX-11	54) SIGRTMAX-10	55) SIGRTMAX-9	56) SIGRTMAX-8	57) SIGRTMAX-7
58) SIGRTMAX-6	59) SIGRTMAX-5	60) SIGRTMAX-4	61) SIGRTMAX-3	62) SIGRTMAX-2
63) SIGRTMAX-1	64) SIGRTMAX
~~~



### 使用 `trap` 命令捕获信号

~~~bash
# 操作1：捕捉信号、执行引号内的操作
trap "echo 已经识别中断信号:ctrl+c" INT
 
# 示例2：捕捉信号、不执行任何操作
trap "" INT  
 
# 示例3：也可以同时捕获多个信号
trap "" HUP INT QUIT TSTP
~~~

>注意：trap 命令定义在脚本的开头





## expect

expect 是一个用来实现自动交互的工具，需要提前安装。

~~~bash
yum install -y expect
~~~

使用 `expect` 很简单，一共只有4个命令可以使用，

| 命令     | 作用                     |
| -------- | ------------------------ |
| spawn    | 启动新进程               |
| expect   | 从进程中接收期待的字符串 |
| send     | 向进程发送字符串         |
| interact | 允许用户交互             |



### 示例：自动ssh连接的脚本

~~~bash
#!/usr/bin/expect

set timeout 3

spawn ssh root@192.168.10.8

expect {
    "(yes/no" {send "yes\n"; exp_continue}
    "password" {send "1\n"}
}

expect "#"
send "cat /etc/passwd\r"

expect eof
~~~

解释：

- `set timeout 3` 设置接收输入等待的超时时间
- `spawn` 后面的命令会在一个子进程中执行
- `expcet` 捕获需要交互的位置，可以使用特殊文本，可以以使用简单的通配符匹配。
- `send` 向交互的位置输入值，结尾使用 `\n` 或者 `\r` 表示回车确定。
- `expect eof` 最后关闭匹配。



注意：

- expect 脚本不能使用 bash 解释器执行，并且脚本的首行内容是 `#!/usr/bin/expect`
- expcet 的语法和 bash shell 的语法不一样，但是可以配合使用。



### expect 脚本传参

expect 脚本语法和 shell 脚本语法不一样，不过变量引用是一样的。

~~~bash
#!/usr/bin/expect

set timeout 3

set user "root"
set ip "192.168.10.8"
set passwd "1"

spawn ssh $user@$ip

expect {
    "(yes/no" {send "yes\n"; exp_continue}
    "password" {send "$passwd\n"}
}

expect "#"
send "cat /etc/passwd\r"

expect eof
~~~



### shell 与 expect 配合使用

所谓的配合就是把 expect脚本中的内容当成一条命令整体执行。此时编写的就是 shell 脚本了。

~~~bash
#!/bin/bash

user="root"
ip="192.168.10.8"
passwd="1"

expect << EOF
    set timeout 3
    spawn ssh $user@$ip
    expect {
        "(yes/no" {send "yes\n"; exp_continue}
        "password" {send "$passwd\n"}
    }
    expect "#"
    send "cat /etc/passwd\r"
    expect eof
EOF
~~~



### 示例：批量拷贝公钥

~~~bash
#!/bin/bash

function copy_pub_id () {
    user=$1
    ip=$2
    passwd=$3

# 下面的内容不能缩进，否则会报错
expect << EOF
    set timeout 3
    spawn ssh-copy-id $user@$ip
    expect {
        "(yes/no" {send "yes\n"; exp_continue}
        "password" {send "$passwd\n"}
    }
    expect eof
EOF
}


while read line; do
    # 忽略注释内容
    if [[ "$line" =~ ^# ]]; then
        continue
    fi
    user=$(echo $line |cut -d: -f 1)
    ip=$(echo $line |cut -d: -f 2)
    passwd=$(echo $line |cut -d: -f 3)

    copy_pub_id $user $ip $passwd

done < ~/hosts.txt
~~~

配置文件（~/hosts.txt）

~~~tex
# user:ip:passwd
root:192.168.10.8:1
root:192.168.10.8:1
root:192.168.10.8:1
~~~

