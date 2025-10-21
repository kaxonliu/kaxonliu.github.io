# 镜像的常用指令



**从远程仓库下载镜像**

~~~bash
docker pull <image>
~~~

- 直接使用 `docker pull image` 默认下载最新版本的镜像，即此时的镜像版本标签为 `latest`
- 如果下载指定版本的镜像，则可以加上版本号 `docker pull image-name:tag`
- 该指令等价于 `docker image pull <image>`
- 完整的镜像名称：`仓库地址/项目地址/镜像名：tag`，docker hub 官方项目地址下的镜像完整镜像名称为：`docker.io/library/镜像名:tag`



**查看本地镜像列表**

~~~bash
docker images
~~~
- 等价于 `docker image list`，等价于 `docker iamge ls`
- 仅查看镜像 ID `docker image ls -q`



**删除镜像**

~~~bash
docker rmi 镜像id 或者 镜像名:tag
~~~

- 等价于 `docker image remove`， 等价于 `docker image rm`
- 如果想要强制删除镜像，可以使用删除指令时增加选项 `-f`，需要注意容器在运行时无法删除它的镜像。因为容器需要的 `lowerdir` 在镜像内。
- 支持同时删除多个镜像



**基于 dockerfile 构建镜像**

```bash
docker build -t kaxonliu/hello-docker:v1 -f Dockerfile .
```

- 使用 Dockerfile 构建镜像是主流的推荐方式，我们在后面会详细介绍。
- 参数 `-t` 指定待构建镜像的镜像名和版本
- 参数 `-f` 指定Dockerfile文件的路径，默认是当前路径下的 `Dockerfile` 文件
- 最后的 `.` 表示当前路径下的 context



**基于容器创建镜像**

把一个正在运行的容器 commit 成镜像，把数据保存在了镜像中。

```bash
docker commit <容器id> kaxonliu/my-nginx:v1
```

- 等价于 `docker container commit`
- commit 时可以通过指定容器id 或者容器名来创建镜像。



**把镜像打包为文件**

~~~bash
docker save nginx -o /root/nginx.zip
~~~

- 等价于 `docker iamge save`
- 参数 `-o` 表示打包后的文件存放路径。



**从文件解压得到镜像**

```bash
docker load -i /root/nginx.zip
```

- 等价于 `docker image load`
- 参数 `-i` 表示被解压文件的路径



**上传镜像到远程仓库**

```bash
docker push docker.io/kaxonliu/my-nginx:v1
```

- 等价于 `docker iamge push`
- 上传时需要登录仓库

