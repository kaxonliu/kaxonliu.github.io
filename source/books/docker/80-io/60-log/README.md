# 容器日志

每个容器都有自己的日志，用来接收标准输出的内容，容器日志默认写在宿主机上。

查看日志路径

~~~bash
docker inspect <containerid> | grep -i logpath
~~~



## 控制容器日志的大小

#### 单个容器设置

启动容器时，我们可以通过参数来控制日志的文件个数和单个文件的大小，使用选项 `--log-opt` 配合两个参数：

- `max-size` 设置单个日志文件体积；
- `max-file` 设置日志文件数量。

~~~bash
docker run -it --log-opt max-size=10m --log-opt max-file=3 redis
~~~



#### 全局设置

增加 docker 配置参数 `/etc/docker/daemon.json`

~~~json
{
    "log-driver": "json-file",
    "log-opts": {
        "max-size" :"50m","max-file":"3"
    }
}
~~~

重启 docker 服务后生效。不过已存在的容器不会生效。