# Registry 搭建镜像仓库

使用 Registry 搭建私有镜像仓库的整体思路很简单。首先，选择一个主机当仓库。然后，在这个主机上使用名为 registry 的镜像创建容器启动服务。在这个过程中开放端口，挂在数据卷，还有登陆认证。最后就可以推拉镜像了。



## 主机规划

镜像仓库主机 IP：172.28.203.209

宿主机上存放镜像的文件夹：`/opt/registry`

宿主机上存放登陆认证的文件夹：`/opt/registry-auth/`



## 第一步：下载 registry 镜像

~~~bash
docker pull registry
~~~



## 第二步：创建仓库文件夹

~~~bash
mkdir /opt/registry
~~~



## 第三步：增加登陆认证

#### 安装依赖工具

~~~bash
yum install httpd-tools -y
~~~

#### 创建密码文件夹并设置用户名和密码

~~~bash
mkdir -p /opt/registry-auth
htpasswd -Bbn liuxu 123 >> /opt/registry-auth/htpasswd

# 查看密码文件
cat /opt/registry-auth/htpasswd
liuxu:$4y$01$7ADlxyJvCquMAedt07rszuAZenf.plfLtvQ6RGcDBG8ayYgDhp9d2
~~~



## 第四步：创建容器

创建容器，端口映射为 5000，并挂载数据卷。

~~~bash
docker run -d -p 5000:5000 \
	-v /opt/registry:/var/lib/registry \
	-v /opt/registry-auth/:/auth \
	-e REGISTRY_AUTH=htpasswd \
	-e REGISTRY_AUTH_HTPASSWD_REALM="Registry Realm" \
	-e REGISTRY_AUTH_HTPASSWD_PATH=/auth/htpasswd \
	--restart=always \
	--name registry \
	registry
~~~



## 第五步：测试推拉镜像

在另一台测试主机上，更新 docker 的配置文件，增加如下配置项：`insecure-registries`

配置镜像仓库的主机 IP 和 registry 服务对外暴露的端口号。

~~~bash
{
	"insecure-registries": ["172.28.203.209:5000"]
}
~~~

重启 docker

~~~bash
systemctl restart docker
~~~

登陆仓库。登陆成功后可以看到输出信息中的 `Login Succeeded`

~~~bash
docker login -u liuxu -p 123 172.28.203.209:5000
~~~

镜像打标签

~~~bash
docker tag centos:7 172.28.203.209:5000/kaxonliu/centos:7
~~~

推送镜像

~~~bash
docker push 172.28.203.209:5000/kaxonliu/centos:7
~~~

拉取镜像

~~~bash
docker pull 172.28.203.209:5000/kaxonliu/centos:7
~~~

