# 正则表达式

**正则表达式**是一种强大的工具，用于在文本中**查找、匹配和操作符合某种模式**的字符串。你可以把它理解为一个**高级的“查找”或“通配符”功能**，但比简单的通配符（如`*`）要强大和精确得多。它的核心作用是：**用一系列特殊的字符来定义一个搜索规则，然后根据这个规则去处理文本。**



shell 中的元字符（也成通配符），和正则表达式使用的特殊符号都是一样的，但是它们用在不同的地方，含义可能也不一样。元字符是shell解释器来解析并使用的，比如 `rm -rf *.pdf` shell 解释器将其解析为i任意多个字符。在 shell 中正则表达式由各种执行模式匹配的程序来解析，比如 vi grep sed awk等。



## 正则表达式使用的特殊字符

| 符号     | 含义与示例                                                   | 扩展字符 |
| :------- | :----------------------------------------------------------- | -------- |
| `.`      | **匹配任意单个字符**（除了换行符）。 例如：`a.c` 匹配 `abc`, `a@c`, `a c` |          |
| `*`      | **匹配前面的字符零次或多次**。 例如：`ab*c` 匹配 `ac`, `abc`, `abbc`, `abbbc`... |          |
| `+`      | **匹配前面的字符一次或多次**。 例如：`ab+c` 匹配 `abc`, `abbc`，但不匹配 `ac` |          |
| `?`      | **匹配前面的字符零次或一次**（即该字符是可选的）。 例如：`colou?r` 匹配 `color` 和 `colour` |          |
| `[abc]`  | **匹配方括号内的任意一个字符**。 例如：`[aeiou]` 匹配任意一个元音字母。 `[0-9]` 匹配任意一个数字（等同于 `\d`）。 |          |
| `[^abc]` | **匹配任何**不在**方括号内的字符**。 例如：`[^0-9]` 匹配任意一个非数字字符。 |          |
| `^`      | **匹配字符串的开始位置**。 例如：`^Hello` 匹配以 `Hello` 开头的字符串。 |          |
| `$`      | **匹配字符串的结束位置**。 例如：`world$` 匹配以 `world` 结尾的字符串。 |          |
| `\b`     | **匹配一个单词的开始或结束位置**。 例如：`\bcat\b` 匹配单词 `cat`，但不匹配 `category` 或 `scatter`。 |          |
| `{n,m}`  | **匹配前面的字符至少 n 次，至多 m 次**。 例如：`a{2,4}` 匹配 `aa`, `aaa`, `aaaa`。 | ✅        |
| `\d`     | **匹配任意一个数字**，等价于 `[0-9]`。 例如：`\d\d` 匹配 `42`, `00`, `99` | ✅        |
| `\w`     | **匹配字母、数字、下划线**，等价于 `[A-Za-z0-9_]`。 例如：`\w+` 匹配一个完整的英文单词，如 `hello`, `user123` | ✅        |
| `\s`     | **匹配任意空白字符**，包括空格、制表符、换页符等。 例如：`a\sb` 匹配 `a b` | ✅        |

>提示：
>
>- `{0, }` 等价于 `*`
>- `{1, }` 等价于 `+`
>- `{0, 1}` 等价于 `?`
>- `{n, n}` 指定固定的长度



## grep 使用正则

普通正则表达式，使用 `grep`，默认是贪婪匹配，非贪婪匹配使用 `grep -P`

扩展正则表达式，使用 `grep -E` 或者 `egrep`

#### 示例：匹配80端口

~~~bash
[root@rocky ~]# netstat -tunpl | grep -w "80"
tcp        0      0 0.0.0.0:80              0.0.0.0:*               LISTEN      6541/nginx: master
tcp6       0      0 :::80                   :::*                    LISTEN      6541/nginx: master
~~~

#### 示例：匹配只有两位数字的行

~~~bash
[root@rocky ~]# grep "^[0-9][0-9]$" abc.txt
~~~

#### 示例：过滤掉所有的注释行和所有的空行

~~~bash
[root@rocky ~]# grep -v "^#" /etc/ssh/sshd_config | grep -v "^ *$"
Include /etc/ssh/sshd_config.d/*.conf
PermitRootLogin yes
AuthorizedKeysFile	.ssh/authorized_keys
Subsystem	sftp	/usr/libexec/openssh/sftp-server

# 或者
grep -v "^#" /etc/ssh/sshd_config | grep -v "^\s*$"
~~~

#### 示例：过滤出所有登陆用户

~~~bash
[root@rocky ~]# grep "/bin/bash$" /etc/passwd
root:x:0:0:root:/root:/bin/bash
liuxu:x:1000:1000::/home/liuxu:/bin/bash
~~~

#### 示例：过滤出长度为3字符的，且是非数字开头的行

~~~bash
[root@rocky ~]# grep "^[^0-9]..$" abc.txt
abb
[root@rocky ~]# grep -E  "^[^0-9]{3,3}$" abc.txt
abb
~~~

#### 示例：统计 /etc 下面包含字符串 root 的文件的个数

~~~bash
[root@rocky ~]# grep -rl  "root" /etc | wc -l
81
~~~

