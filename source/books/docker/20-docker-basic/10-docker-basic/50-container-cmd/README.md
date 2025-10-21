# 容器的常用指令



**创建容器（仅创建，不运行）**

```bash
docker create <image>
```

- 等价于 `docker container create <iamge>`



**启动容器**

```bash
docker start <container>
```

- 等价于 `docker container start <container>`



**创建并运行容器**

```bash
docker run <image>
```

- 等价于  `docker container run <image>`
- 如果镜像不存在，则先下载镜像再运行容器



**查看容器列表**

```
docker ps
```

- 等价于 `docker container ls`
- 该命令只列出正在运行的容器，如果想要列出所有容器，加上参数 `-a`
- 如果只想要展示容器 ID，加上选项 `-q`



**停止容器**

```bash
docker stop <container>
```



**重启容器**

```bash
docker restart <container>
```



**删除容器**

```bash
docker rm <container>
```

- 等价于 `docker container rm`
- 删除容器，可以通过 容器id 或者 容器名 删除
- 如果要删除正在运行的容器，需要强制删除，加上参数 `-f`
- 删除容器后，upperdir 层的数据就丢失了。如果仅是退出容器那数据不丢失。



**进入容器**

```bash
docker exec -it <container> bash
```

- 以交互模式进入容器

**在容器中执行命令**

~~~bash
docker exec -it <container> <cmd>
~~~

- 执行的命令必须是容器内有的命令。
- 如果想要执行的命令容器内没有，可以使用 `nsenter -t <容器pid> -n netstat -an`
- 查看容器的PID: `docker inspect <container> | grep -i pid`



**文件拷贝**

宿主机文件拷贝到容器

~~~bash
docker container cp <宿主机文件> 容器名:容器内文件路径
~~~

容器内文件拷贝到宿主机

~~~bash
docker container cp 容器名:容器内文件路径 <宿主机文件> 
~~~



**查看容器日志**

```bash
docker logs <container>
```



**查看容器详细信息**

~~~bash
docker container inspect <container>
~~~

**动态展示容器的资源使用情况**

~~~BASH
docker container stats da74f4fef776
CONTAINER ID   NAME                   CPU %     MEM USAGE / LIMIT     MEM %     NET I/O     BLOCK I/O     PIDS
da74f4fef776   distracted_zhukovsky   0.14%     8.316MiB / 1.715GiB   0.47%     656B / 0B   7.33MB / 0B   5
~~~

**展示容器内运行的进程**

~~~bash
[root@me ~]# docker container top da74f4fef776
UID                 PID                 PPID                C                   STIME               TTY                 TIME                CMD
polkitd             5755                5735                0                   21:07               ?                   00:00:00            redis-server *:6379
~~~



