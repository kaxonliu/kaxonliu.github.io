# 进程锁

为了让脚本不被重复执行，可以使用进程锁。

~~~bash
#!/bin/bash

lock_file=/tmp/echo1.lock

#判断进程是否正在运行
if [ -f $lock_file ]; then
    pid=`cat $lock_file`
    ps $pid &>/dev/null
    # [ $? -eq 0 ] && echo "Script1 is running..." && exit 1

    if [ $? -eq 0 ]; then
        echo "Script1 is running..."
        exit
    else
        rm -f $lock_file
    fi

fi

#创建锁
echo $$ > $lock_file

echo "lock1 begin..."
sleep 500
echo "lock1 end"

#释放锁
rm -rf $lock_file
~~~

