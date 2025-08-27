# shell脚本

shell 有两层含义，一个指的是 shell 这个编程语言，另一个指的是解释 shell 语法的 shell 解释器，我们通常使用的是 bash。shell 是一门解释型、弱类型、动态语言。



## shell 脚本程序

一个 shell 脚本程序包含三部分，第一行需要执行 shell 解释器，然后是注释部分，最后才是代码程序。

#### 指定解释器

第一行指定使用的 shell 解释是什么吗，固定格式。

~~~bash
# 指定解释器
#!/bin/bash

# 等价
#!/usr/bin/env bash
~~~

#### 注释

Shell 中使用 `#` 作为注释符号，可以在一行的行首注释，整行都是注释内容；也可以在代码的尾部注释，这一行 `#` 后面的是注释部分。

#### 程序代码

程序代码是使用 shell 语言编写，实现指定任务。



## 运行 shell 脚本的方式

#### 方式1：绝对路径 + 指定解释器

这种方式需要脚本的行首指定 shell 解释器，然后 shell 脚本文件所在的层层上级文件夹都具备 `x` 权限，脚本自己需要具备 `r` 和 `x` 权限。

~~~bash
[root@rocky c]# /a/b/c/hello.sh
hello world
~~~

#### 方式2：相对路径 + 指定解释器

这种方式需要脚本的行首指定 shell 解释器，然后 shell 脚本文件所在的层层上级文件夹都具备 `x` 权限，脚本自己需要具备 `r` 和 `x` 权限。执行时使用 `./` 作为前缀。

~~~bash
[root@rocky b]# pwd
/a/b
[root@rocky b]# c/hello.sh
hello world
~~~



#### 方式3：解释器 + 文件路径

这种方式直接使用解释器命令，shell 文件作为参数。文件路径可以是绝对路径也可以是相对路径。此时需要沿途文件夹都具备 `x` 权限，目标文件具备 `r` 文件即可。

~~~bash
[root@rocky c]# ls -l hello.sh
-rw-r--r--. 1 root root 59 Aug 28 05:30 hello.sh
[root@rocky c]# bash hello.sh
hello world
[root@rocky c]# cd ../
[root@rocky b]# bash c/hello.sh
hello world
~~~



#### 方式4：source + 文件路径

上述三种方式都是在子 shell 中执行的脚本文件，如果就想在当前 shell 中执行脚本文件，使用 `source` 命令。此时需要沿途文件夹都具备 `x` 权限，目标文件具备 `r` 权限。 文件路径可以是相对路径也可以是绝对路径。

~~~bash
[root@rocky c]# source hello.sh
hello world
[root@rocky c]# cd ..
[root@rocky b]# source c/hello.sh
hello world
~~~



#### 方式5：. + 文件路径

这种方式和方式4的一样的，只是把 `source` 命令换成 `.`，其他用法和要求都一样，依然是在当前 shell 中执行脚本。



#### 总结：在当前shell执行脚本时，脚本中使用的变量在当前shell中都可以使用。



## 调试shell程序

#### 调试的方式运行

~~~bash
bash -vx hello.sh
~~~

#### 检查语法错误

~~~bash
bash -n hello.sh
~~~

#### 仅调试脚本中的一部分

在脚本中把需要调试的代码使用 `set -x` 和 `set +x` 包起来。再使用 `bash -vx` 调试。

~~~bash
#!/bin/bash

set -x
echo "hello world"
set +x

echo "over and over"
~~~




