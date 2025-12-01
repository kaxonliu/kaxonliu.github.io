# 客户端工具

部署 k8s 后，可以使用的客户端命令有多个：`docker`、`ctr` 、`crictl`、`nerdctl`、`kubectl`。其中，

- `ctr` 是 containerd 的一个客户端工具，支持名称空间。
- `crictl` 是遵循 CRI 接口规范的一个工具，通常用来检查和管理 kubectl 节点上的容器运行时和镜像。只展示 `k8s.io` 这个名称空间下的容器和镜像。
- `nerdctl` 是一个第三方工具。用法和 `docker` 几乎完全一样，且支持名称空间，构建镜像等。
- `kubectl` 是 k8s 的命令行工具，用于与 k8s 集群进行交互。



## 镜像与容器存放的名称空间

可以用 `-n` 指定名称空间，可以将镜像或者容器分散到不同的空间内，但并非所有命令都支持。

| 事项     | docker | ctr (containerd)                                        | crictl (kubernetes)                  | nerdctl                          |
| -------- | ------ | ------------------------------------------------------- | ------------------------------------ | -------------------------------- |
| 是否支持 | 不支持 | 支持                                                    | 不支持                               | 支持                             |
| 示例     |        | `ctr ns ls`                                             |                                      | `nerdctl -n k8s.io image ls`     |
|          |        | `ctr -n k8s.io image ls` # 查看指定名称空间下的镜像     | 默认所有操作都在 `k8s.io` 名称空间下 | `nerdctl -n k8s.io container ls` |
|          |        | `ctr -n k8s.io container ls` # 查看指定名称空间下的容器 |                                      |                                  |



## 镜像相关

| 命令             | docker           | ctr (containerd)                                             | crictl (kubernetes)                                          | nerdctl        |
| ---------------- | ---------------- | ------------------------------------------------------------ | ------------------------------------------------------------ | -------------- |
| 查看本地镜像列表 | `docker images`  | `ctr image ls`                                               | `crictl images`                                              | 同 docker 命令 |
| 下载/拉取镜像    | `docker pull`    | `ctr image pull`                                             | `crictl pull`                                                | 同 docker 命令 |
| 给镜像打标签     | `docker tag`     | `ctr image tag`                                              | 无                                                           | 同 docker 命令 |
| 上传/推送镜像    | `docker push`    | `ctr image push`                                             | 无                                                           | 同 docker 命令 |
| 删除本地镜像     | `docker rmi`     | `ctr image rm`                                               | `crictl rmi`                                                 | 同 docker 命令 |
| 查看镜像详情     | `docker inspect` | 无                                                           | `crictl inspecti` # 注意末尾是小写字母 i                     | 同 docker 命令 |
| 导出镜像         | `docker save`    | `ctr image export`                                           | 无                                                           | 同 docker 命令 |
| 导入镜像         | `docker load`    | `ctr image import xx.tar docker.io/library/nginx:alpine`     | 无                                                           | 同 docker 命令 |
| 登录镜像仓库     | `docker login`   | 不支持，理由：ctr 工具主要是用于底层的容器操作和管理，而不是直接与镜像仓库进行交互如推拉镜像等。 | 不支持，理由：主要用于管理 Pod 和容器，而非直接与镜像仓库交互。 | 支持           |



## 容器相关

| 命令                     | docker               | ctr (containerd)                   | crictl (kubernetes)  | nerdctl        |
| ------------------------ | -------------------- | ---------------------------------- | -------------------- | -------------- |
| 显示本地运行的容器列表   | `docker ps`          | `ctr task ls` / `ctr container ls` | `crictl ps`          | 同 docker 命令 |
| 创建一个新的容器         | `docker create`      | `ctr container create`             | `crictl create`      | 同 docker 命令 |
| 运行一个新的容器         | `docker run`         | `ctr run`                          | 无（最小单元为 pod） | 同 docker 命令 |
| 启动容器                 | `docker start`       | `ctr task start`                   | `crictl start`       | 同 docker 命令 |
| 关闭容器                 | `docker stop`        | `ctr task kill`                    | `crictl stop`        | 同 docker 命令 |
| 删除容器                 | `docker rm`          | `ctr container rm`                 | `crictl rm`          | 同 docker 命令 |
| 查看容器详情             | `docker inspect`     | `ctr container info`               | `crictl inspect`     | 同 docker 命令 |
| 查看容器日志             | `docker logs`        | 无                                 | `crictl logs`        | 同 docker 命令 |
| 查看容器资源             | `docker stats`       | 无                                 | `crictl stats`       | 同 docker 命令 |
| 登录或在容器内部执行命令 | `docker exec`        | 无                                 | `crictl exec`        | 同 docker 命令 |
| 清空不用的容器           | `docker image prune` | 无                                 | `crictl rmi --prune` | 同 docker 命令 |



## pod 相关

只有 `crictl` 可以操作 pod，了解即可，主要通过 k8s 启停管理 pod。

| 命令        | docker | ctr (containerd) | crictl (kubernetes)         | nerdctl |
| ----------- | ------ | ---------------- | --------------------------- | ------- |
| 显示POD列表 | 无     | 无               | `crictl pods`               | 无      |
| 查看POD详情 | 无     | 无               | `crictl inspectp pod的id号` | 无      |



## 配置镜像加速

给 containerd 配置加速，建议修改配置文件 `/etc/containerd/config.toml` 文件，在该文件中指定 `config_path=/etc/containerd/certs.d`，在这个文件件中编写镜像加速的配置。

~~~toml
[root@k8s-master-01 certs.d]# cat docker.io/hosts.toml
server = "https://docker.io"
[host."https://p...d.mirror.aliyuncs.com"]
capabilities = ["pull", "resolve"]
[host."https://xxx.com"]
capabilities = ["pull", "resolve"]
~~~

比如，为 docker.io 配置镜像加速，则新建 docker.io 文件夹，文件夹中新建 hosts.toml 文件，其中按照上述格式配置一个或多个加速地址。为其他镜像仓库设置加速的方式类似。



## 安装 nerdctl

~~~bash
# 下载
wget https://github.com/containerd/nerdctl/releases/download/v1.7.6/nerdctl-1.7.6-linux-amd64.tar.gz

# 创建文件件
mkdir -p /usr/local/containerd/bin/

# 解压
tar -zxvf nerdctl-1.7.6-linux-amd64.tar.gz nerdctl

# 移动命令文件
mv nerdctl /usr/local/containerd/bin/

# 软连接
ln -s /usr/local/containerd/bin/nerdctl /usr/local/bin/nerdctl

# 验证
nerdctl version
~~~



安装后如果有警告，提示 buildkit 未安装，可以安装 buildkit 解决警告。

~~~bash
# 安装 buildkit
wget https://github.com/moby/buildkit/releases/download/v0.13.2/buildkit-v0.13.2.linux-amd64.tar.gz

tar -xzvf buildkit-v0.13.2.linux-amd64.tar.gz

mkdir -p /usr/local/buildctl
mv bin /usr/local/buildctl

cat >> /etc/profile << "EOF"
export PATH=/usr/local/buildctl/bin:$PATH
EOF

source /etc/profie
~~~



