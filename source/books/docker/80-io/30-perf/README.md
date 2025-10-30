# 硬盘性能分析工具

perf 是一款 Linux 性能分析工具。

我们可以使用 perf 来捕获一个任务在系统中的完整调度栈信息，或者系统整体的情况，以便分析到底是哪个环节拖慢了整体的性能。

#### 安装

~~~bash
yum install perf -y
~~~

#### 测试使用

测试

~~~bash
perf record -a -e cycles -o cycle.perf -g sleep 10
# 注意
perf record代表记录一段时间内系统/进程的性能时间
参数：
 -a：分析整个系统的性能
 -e：选择性能事件
 -o：指定输出文件，默认为perf.data
 -g：记录函数间的调用关系
 sleep 10 表示收集10秒钟。
~~~

采样数据生成报告

~~~bash
perf report -i cycle.perf | less
~~~



## 火焰图

我们可以把 perf 捕捉的信息制作成火焰图来方便查看与分析CPU的调用栈。

火焰图（Flame Graph）是由Linux性能优化大师Brendan Gregg发明的，和所有其他的profiling方法不同的是，火焰图以一个全局的视野来看待时间分布，它从底部往顶部，列出所有可能导致性能瓶颈的调用栈。

下载FlameGraph工具。

~~~bash
git clone https://github.com/brendangregg/FlameGraph.git
~~~



#### 生成火焰图流程

1、捕获堆栈
使用perf捕捉进程运行堆栈信息

```
cd /root
perf record -F max -p 进程的pid -g -- sleep 60 # -p代表捕获某一个进程，你也可以用-a从所有cpu收集数据
```
perf检测完毕，会在当前目录下生成一个perf.data文件，该文件是一个二进制信息文件

2、将perf.data的二进制信息转换为ASCII格式的文件，方便可视化处理：
```
perf script -i /root/perf.data &> /root/perf.unfold
```

3、用stackcollapse-perf.pl将perf解析出的内容perf.unfold中的符号进行折叠
```
./flameGraph/stackcollapse-perf.pl /root/perf.unfold &> /root/perf-folded
```

4、生成火焰图
```
./flameGraph/flamegraph.pl /root/perf-folded > /root/perf.svg
```

5、浏览器打开perf.svg

