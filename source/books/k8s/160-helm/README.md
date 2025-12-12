# helm

helm 就是用来安装 chart 包的工具，chart 包就是一系列相关的 yaml 的集合体，按照固定格式组织起来。

对于复杂的应用来说，涉及到相关的 yaml 会非常多，把它都组织到一起形成一个 chart 包，然后用 helm 统一进行安装与卸载会非常方便。

## 使用 helm

#### 安装

到官网下载：https://github.com/helm/helm/releases，选择合适的平台，下载解压得到二进制文件即可使用。



#### 常用命令

仓库里放到的是 chart 包，类似给 yum 添加 repo 源。

~~~bash
# 添加仓库
helm repo add stable http://mirror.azure.cn/kubernetes/charts/

# 查看本地添加的仓库
helm repo list

# 查询仓库中有哪些 chart 包
helm search repo stable

# 下载 chart 包
# 可以不指定版本，默认下载最新的
helm pull stable/prometheus-rabbitmq-exporter --version 0.1.1

# 安装 chart 包
# 支持在线安装和本地离线安装
# 在线安装两种方式
helm install 你的release名字  仓库名字/某个chart包名  --version 1.1
helm install 你起的release名 http://mirror.azure.cn/kubernetes/charts/mariadb-7.3.9.tgz

# 本地安装两种方式
helm install 你的release名字  ./xxxxx-0.5.6.tgz
helm install 你的release名字  /a/b/c/chart包的目录


# 了解
helm repo update # 类似于yum makecahe

# 自动生成名字(了解)
helm install --generate-name ./mysql-1.6.9.tgz --namespace kube-system
~~~

~~~alert type=note
每安装或升级一次chart包都会得到 一条relrease记录
~~~

chart 包中的 yaml 文件一般都是模板，支持定制化。

~~~bash
# 本质查看的chart包下values.yaml的内容
helm show values chart包的来源/路径
helm show values stable/mysql
helm show values ./mysql-1.6.9.tgz
~~~

定制化的参数值（优先级从高到低）：

- 在安装的时候用 --set选型传入key=value，--set imageTag="5.8.0"
- 在安装的时候用 -f 选型执行自己的 /a/b/c/my_values.yaml（常用）
- chart包默认自带的：values.yaml 默认值（常用）

比如使用 helm 安装 nfs-subdir-external-provisioner

~~~bash
helm repo add nfs-subdir-external-provisioner \
https://kubernetessigs.github.io/nfs-subdir-external-provisioner/

helm install nfs-subdir-external-provisioner \
nfs-subdir-external-provisioner/nfs-subdir-external-provisioner \
--set nfs.server=172.16.143.24 \
--set nfs.path=/data/nfs \
--set storageClass.defaultClass=true \
--set image.repository=crpi-hdcg863vh2ayst45.cn-shanghai.personal.cr.aliyuncs.com/liuxu8677/nfs-subdir-external-provisioner \
--set image.tag=v4.0.2 \
-n kube-system
~~~

使用如下命令查看渲染好的 yaml 内容

~~~bash
helm install egon-release-name ./mysql --namespace kube-system -f /a/b/c/my_values.yaml --dry-run
~~~



#### 升级和回滚

已经安装的 chart 包，升级

~~~bash
helm upgrade mysql stable/mysql -f 可以考虑指定自己的my_values.yaml文件
~~~

回滚操作

~~~bash
# 查看安装记录
helm history mysql  # 我的release名为mysql

# 回滚到指定记录
helm rollback mysql 1 # 回滚到某个revision
~~~

