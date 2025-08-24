# 计划任务

`crond` 是 Linux 系统中的一个**守护进程**（daemon），用于在指定的时间、日期或间隔周期性地执行预定的命令或脚本。有了这个服务，可以使用 `crontab` 命令管理计划任务的命令。用户使用 `crontab` 命令来创建、编辑、列出和删除自己的计划任务。

大多数 Linux 发行版默认是安装并启动 `crond` 服务的。值得一提的是，计划任务都是配置在文件中的，每次编辑文件后不需要重启 `crond` 服务。

~~~bash
systemctl status crond
sudo systemctl start crond
sudo systemctl enable crond
# 重新加载 crond 服务配置（在修改了某些系统级配置后可能需要）
sudo systemctl reload crond
~~~



## 计划任务类型

Crontab定时任务一般可以分为两种主要类型：用户crontab任务、系统crontab任务。

### 用户级定时任务

通常由用户自己维护的定时任务，系统管理员当然也可以维护。定时任务的配置最终都会记录到文件中，用户级的定时任务文件位于 `/var/spool/cron/` 目录下，文件名是用户名。

~~~bash
# 查看自己的定时任务
crontab -l

# 编辑自己的定时任务（本质上就是编辑 var/spool/cron/和当前用户同名的文件）
crontab -e

# 查看别人的定时任务（需要root权限）
crontab -l -u liuxu
~~~



### 系统级定时任务

只有系统管理员可以编辑。系统级定时任务文件位于/etc目录下，可以在下面这些文件中配置定时任务。

~~~bash
/etc/crontab：这是主crontab文件，包含了系统任务表，以及在预设时间执行的工作。
/etc/cron.d/：此目录存放任何想被cron守护进程执行的任务。
/etc/cron.daily/：此目录下存放的任务脚本会被系统安排在每天凌晨执行。
/etc/cron.hourly/：此目录下存放的任务脚本会被系统安排每小时执行一次。
/etc/cron.monthly/：此目录下存放的任务脚本会被系统安排在每月的某一天执行。
/etc/cron.weekly/：此目录下的任务脚本会被系统安排在每周的某一天执行。

# 注意：放在这个文件夹中的待执行的脚本文件需要有 x 权限
~~~





## crontab 文件的格式

不论是用户级定时任务，还是系统级定时任务都需要编写计划任务文件（crontab 文件），这个文件使用特殊的格式来自定义计划任务执行的时间。

一个 crontab 文件由多行组成，每一行代表一个独立的定时任务，其格式如下：

```bash
# ┌────────── 分钟 (0 - 59)
# │ ┌──────── 小时 (0 - 23)
# │ │ ┌────── 日 (1 - 31)
# │ │ │ ┌──── 月 (1 - 12)
# │ │ │ │ ┌── 星期 (0 - 7，0 和 7 都代表周日)
# │ │ │ │ │
# * * * * * <user-name> <要执行的命令>
```

### 时间字段的特殊符号

| 符号 | 含义             | 示例                                                         |
| :--- | :--------------- | :----------------------------------------------------------- |
| `*`  | 代表所有可能的值 | `* * * * *` 每分钟执行一次                                   |
| `,`  | 指定一个列表     | `0 8,12,18 * * *` 在每天 8点，12点，18点的0分钟执行一次      |
| `-`  | 指定一个范围     | `0 9-17 * * 1-5` 周一至周五的 9点到17点，每小时执行一次（整点） |
| `/`  | 指定间隔频率     | `*/10 * * * *` 每 10 分钟执行一次                            |

### 日期穿刺

当日期字段和星期字段都被指定值的时候，那么只要满足其中任何一个字段的条件，计划任务就会被执行。

~~~bash
00 02 14 * 7 # 每月14日或每周日的凌晨2点都执行
00 02 14 2 7  # 每年的2月14日或每年2月的周天的凌晨2点执行
~~~





## 使用命令 `run-parts`

命令 `run-parts` 可以执行一个文件夹下面所有的脚本文件，需要脚本文件都有 x 权限，配合计划任务非常方便。

~~~bash
# 配置一个计划任务，每周1的晚上18:30分 liuxu 用户执行 /test 下面的所有脚本。
# 需要 liuxu 对 /test下面的脚本文件都有 x 权限

# /etc/crontab
30 18 * * 1 liuxu run-parts /test
~~~



## 处理命令的错误输出

定时任务不可避免会出现各种各样的错误，计划任务执行失败，邮件服务就会自动错误信息发送到用户的邮件文件中，日积月累就会耗费过多的硬盘空间。此时可以在计划任务的命令后添加  `&>/dev/null`，把执行任务的结果丢掉，就不会再收到邮件了。或者觉得邮件服务没有使用的必要，干脆把邮件服务关掉。

~~~bash
[root@localhost ~]# crontab -l
* * * * * /usr/sbin/ntpdate ntp.aliyun.com &>/dev/null


# 关闭邮件服务
systemctl stop postfix.service
~~~



## 查看定时任务执行日志

~~~bash
tail -f /var/log/cron
~~~



## 定时任务黑名单

`/etc/cron.deny` 是定时任务的黑名单，把指定用户加入，该用户的定时任务就不会再被执行，该用户也无法再编辑定时任务。

~~~bash
echo "jack" >> /etc/cron.deny 

# 加入黑名单后就无法再创建定时任务
[jack@localhost ~]$ crontab -e
You (jack) are not allowed to use this program (crontab)
See crontab(1) for more information
~~~





## 注意事项

### 1. 定时任务规则前加上注释。

### 2. 使用脚本执行定时任务，统一脚本存放的位置。

### 3. 执行脚本一定要使用命令的绝对路径。

### 4. crontab文件中直接使用 date 命令，需要把百分号转译。

~~~bash
# /var/spool/cron/root
* * * * * /bin/echo "hello world `date '+\%F \%T'`" &>> /tmp/run.log
~~~

### 5. 命令或脚本结果定向到黑洞文件（&>/dev/null）。

### 6. 避免不必要的程序及命令输出（tar -v）。

### 7. 系统与命令位置相关的环境变量问题，在脚本中重新定义PATH。

~~~bash
# /root/bak.sh 
#!bin/bash
export PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/root/bin # 防止下述命令出问题
mkdir /backup
cd /
tar -czf backup/$(hostname)_$(date +%F_%T)_etc.tar.gz /etc   
find backup -type f -name "*.tar.gz" -mtime +3 |xargs rm -rf
~~~



