# 软件包安装和管理

红帽系（如RHEL、CentOS、Fedora）中三种软件安装方式分别是：rpm包、源码包、二进制包。

| **分类**     | **特点**                                                     | **优缺点**                                                   |
| :----------- | :----------------------------------------------------------- | :----------------------------------------------------------- |
| **RPM包**    | **文件格式**：`.rpm` 文件。<br>**安装方式**：`rpm -iv h` 或 `yum/dnf install` | **优点**：安装简单、自动依赖管理（yum/dnf）、支持版本控制和卸载。 **缺点**：灵活性低，无法自定义优化。 |
| **源码包**   | **文件格式**：`.tar.gz`, `.tar.bz2`<br>**安装方式**：解压后 `./configure`, `make`, `make install` | **优点**：可定制编译选项、优化性能、适用于最新版本或特殊需求。 **缺点**：依赖需手动安装，安装复杂，卸载不便。 |
| **二进制包** | **文件格式**：`.tar.gz`, `.bin`<br>**安装方式**：解压后直接运行 | **优点**：开箱即用，无需编译，适合闭源软件。 **缺点**：可能缺乏系统集成，依赖需自行处理。 |



## RPM包

**RPM**（**RPM Package Manager**，原名 *Red Hat Package Manager*）是红帽系 Linux（如 RHEL、CentOS、Fedora）使用的软件包格式，用于软件的安装、升级、查询和卸载。

### rpm包名称

~~~bash
apr-1.4.8-7.el7.x86_64.rpm

# apr 软件名称
# 1.4.8 版本号
# 7 发版次数
# el7 适用于操纵系统版本
# x86_64 适用的平台
# rpm 扩展名
~~~



## 使用 rpm 命令安装管理 rpm包

因为 `rpm` 命令安装包不会解决依赖问题，使用 `rpm` 安装 rpm 包经常遇到很麻烦的依赖需要解决，所以一般会使用 `yum` 命令安装 rpm 包，使用 `rpm` 命令查询安装的包信息。

| **命令**                    | **说明**                                                     |
| :-------------------------- | :----------------------------------------------------------- |
| `rpm -ivh <包名>.rpm`       | **安装** RPM 包（`i`=安装，`v`=详细信息，`h`=进度条）。      |
| `rpm -Uvh <包名>.rpm`       | **升级** RPM 包（若未安装则执行安装）。                      |
| `rpm -e <包名>`             | **卸载** 已安装的 RPM 包（`e`=erase）。                      |
| `rpm -qa`                   | **查询所有已安装** 的 RPM 包（`q`=query，`a`=all）。         |
| `rpm -qi <包名>`            | **查看软件包详细信息**（如版本、发行号、安装时间等）。`rpm -qi zip` |
| `rpm -ql <包名>`            | **列出包内所有文件**（`l`=list）。                           |
| `rpm -qc <包名>`            | 列出包内的所有配置文件。`rpm -qc yum`                        |
| `rpm -qf <文件路径>`        | **查询某个文件属于哪个 RPM 包**（`f`=file）。`rpm -qf /usr/sbin/ifconfig` |
| `rpm -qR <包名>`            | **列出包的依赖**（`R`=requires）。                           |
| `rpm --checksig <包名>.rpm` | **检查 RPM 包的签名**（验证完整性）。                        |
| `rpm -qpi <包名>.rpm`       | **查看未安装的 RPM 包信息**（`p`=package）。                 |
| `rpm -qpl <包名>.rpm`       | **列出未安装的 RPM 包内文件**。                              |





## 使用 yum 命令安装管理 rpm 包

YUM（**Yellowdog Updater Modified**）是 RHEL/CentOS/Fedora 等 Linux 发行版中的 **高级包管理工具**，基于 RPM 构建，主要功能包括：

- **自动解决依赖关系**
- **从仓库（Repository）安装/更新软件**
- **支持事务（Transaction）回滚**

> 注意：rockylinux 9中，yum 命令被链接到了 dnf 上，dnf的用法和yum一样。def 是yum 的第一代版本。相较于 yum，dnf具有更好的性能，解决依赖关系更加精准并且使用更方便。



### yum 命令

~~~bash
# 查询yum仓库信息，加上 all 表示一块查看启用的和禁用的仓库。
# yum仓库的配置文件是 /etc/yum.repos.d/下面的 *.repo文件
# 修改 repo文件中的 enabled=1表示启用，等于0表示禁用
yum repolist
yum repolist all


# 查看yum仓库中所有的软件包
yum list | grep less


# 安装软件
yum install -y httpd
# 重新安装
yum reinstall -y 
# 卸载，只会卸载软件本省，不会一块卸载依赖包
yum remove httpd

# 更新所有软件包，包括内核，通常只在刚装完系统时执行
yum update -y			


# 更新指定软件包，更新时会保留旧的软件包，这对依赖旧包的应用很友好
yum update httpd -y


# 使用 upgrade更新，但是会删除旧的软件包
yum upgrade

 
# 查看yum仓库中的软件组（就是一类软件包的集合，比如网络工具包的集合 Networking Tools）
yum grouplist
 
# 安装软件组
yum groupinstall "Networking Tools" -y
 
# 写在软件组
yum groupremove "Networking Tools" -y


yum clean all  # 清理缓存
yum makecache  # 制作缓存，缓存的是元数据，不是 npm 包


# 查看使用 yum 的历史记录
yum history
yum history info <id>				# 查看具体信息
yum history undo <id>				# 撤销执行的历史命令
~~~



### yum源

yum 源也叫 yum 仓库，是存放 rpm 包的仓库，使用 `yum` 命令就是从这个仓库下载 rpm 包然后安装的。并且这个仓库也解决了包之间的依赖关系，因此使用 `yum` 命令就可以自动解决依赖关系。yum仓库可以是官方提供的，也可以是互联网大厂提供的，当然我们也可以制作自己的 yum 仓库。

yum仓库的信息都配置在 `/etc/yum.repos.d/*.repo` 这些 以 `repo` 结尾的文件中。这些文件都是按照固定格式配置信息的。下面是腾讯云服务器上的一个配置信息。其中：`baseurl`提供的地址是rpm包的地址，我们使用 `yum` 命令就是到这个地址下载 rpm 包的；`enbaled=1`表示这个配置信息是启动的。一个 repo文件中可以配置多条仓库信息。

~~~bash
[extras]
name=Qcloud centos extras - $basearch
baseurl=http://mirrors.tencentyun.com/centos-vault/7.9.2009/extras/$basearch/
enabled=1
gpgcheck=1
gpgkey=http://mirrors.tencentyun.com/centos-vault/RPM-GPG-KEY-CentOS-7
~~~

`baseurl`是仓库的地址，这个可以使用很多协议方式提供地址信息。比如

- http：`baseurl=http://mirrors.tencentyun.com/xxx`
- https：`baseurl=https://mirrors.tencentyun.com/xxx` 
- ftp：`baseurl=ftp://mirrors.tencentyun.com/xxx`
- file：`baseurl=file:///opt/test`



### 镜像文件作为 yum 源

#### **1. 挂在操作系统镜像**

~~~bash
# 方式1：
mount /dev/cdrom /opt/
 
# 方式2：
mount /dev/sr0 /opt/
 
# 方式3 直接把iso镜像文件
mount -o loop /xxx.iso /opt
 
# 查看光盘里的rpm包
ls /opt/Packages/ 
~~~

#### **2. 编辑repo文件**

~~~bash
cd /etc/yum.repos.d/
vim local.repo  # 文件名自定义，必须以.repo结尾
[local]  # 仓库的实际名字,任意
name=local # 仓库的描述，任意
baseurl=file:///opt  # 仓库位置,可以是 http:// https:// ftp:// file://
enabled=1  # 启用仓库，默认就是启用的
gpgcheck=0 # 检查安装的rpm是否是合法的，0表示不检验
~~~

#### **3. 检查仓库是否可用**

~~~bash
yum repolist
~~~

**4 使用yum命令**

~~~bash
yum install httpd -y
~~~





### 制作自己的yum仓库

一个yum仓库需要满足两部分：1 提供 npm包；2 提供 npm 包之间的依赖关系。

#### **1. 下载 npm 包存放到一个文件夹中**

下载的方式有很多种：（1）选择网上的yum源下载npm包；（2）开启 yum 命令的缓存功能（`keepcache=1`），在 `cachedir` 中搜集缓存的 npm 包；（3） 直接使用 yum 命令提供的仅下载 npm 包的功能。

~~~bash
yum install --downloadonly  --downloaddir=/my_repos httpd -y
~~~



#### **2. 提供 npm 包之间的依赖关系**

这一步使用工具包 `createrepo` 帮我们解决。要安装后才能使用它。

~~~bash
# 下载 createrepo
yum install -y createrepo

# 生成依赖关系
createrepo /my_repos
~~~



#### **3. 制作 repo 文件**

在 `/etc/yum.repos.d` 文件夹中新建一个 `local.repo` 文件，配置如下信息。

~~~bash
[local]
name=local
baseurl=file:///my_repos
enabled=1
gpgcheck=0
~~~



#### 4. 使用 ftp 共享

服务端需要需要提供 yum 仓库，然后提供一个 ftp 服务，让客户端通过 ftp 的方是用使用该仓库；客户端只需要在本地的 `/etc/yum.repos.d/`文件夹下新建一个 repo 文件，选择使用 ftp 的方式配置 `baseurl`。

**服务端提供 ftp 服务**

~~~bash
# 安装 vsftpd
yum install vsftpd -y

# 把服务端机器上的yum仓库放到 /var/ftp 目录下
ln -s /my_repos /var/ftp/my_repos
~~~

**客户端编写 repo 文件**

~~~bash
# /etc/yum.repos.d/xxx.repo

[xxx]
name=xxx
baseurl=ftp://服务端ip/my_repos
enabled=1
gpgcheck=0
~~~



### yum命令的配置

使用 `yum` 命令安装软件包有一些缓存机制，是否开启缓存，缓存在什么位置。这些都可以在文件 `/etc/yum.conf` 中做配置。其中，`cachedir` 指定缓存数据存放的目录；`keepcache` 配置是否开启缓存；`metadata_expire` 配置缓存过去时间，单位：`s秒 m分钟 h小时 d天`，默认单位是秒。

~~~bash
[main]
cachedir=/var/cache/yum/$basearch/$releasever
keepcache=0
debuglevel=2
logfile=/var/log/yum.log
exactarch=1
obsoletes=1
gpgcheck=1
plugins=1
installonly_limit=5
bugtracker_url=http://bugs.centos.org/set_project.php?project_id=23&ref=http://bugs.centos.org/bug_report_page.php?category=yum
distroverpkg=centos-release
metadata_expire=90m
~~~



## 使用阿里云yum仓库

阿里云 repo 镜像配置页：**https://mirrors.aliyun.com/repo/**

### 使用 x86-64架构的yum源

~~~bash
wget -O /etc/yum.repos.d/CentOS-Base.repo https://mirrors.aliyun.com/repo/Centos-7.repo
wget -O /etc/yum.repos.d/epel.repo https://mirrors.aliyun.com/repo/epel-7.repo
yum clean all
yum makecache
~~~

### 使用 aarch64 架构的 yum 源

~~~bash
wget -O /etc/yum.repos.d/CentOS-Base.repo https://mirrors.aliyun.com/repo/Centos-altarch-7.repo
wget -O /etc/yum.repos.d/epel.repo https://mirrors.aliyun.com/repo/epel-7.repo
yum clean all
yum makecache
~~~







## 源码包安装 nginx

#### **下载源码包** 

~~~bash
# nginx官网官网找下载链接
wget https://nginx.org/download/nginx-1.28.0.tar.gz
~~~



#### **预先安装编译需要的依赖库**

~~~bash
yum -y install gcc gcc-c++ autoconf automake make      
yum -y install zlib zlib-devel openssl openssl-devel pcre pcre-devel
 
# 或者   LANG=C yum -y groupinstall "Development tools"
~~~



#### **解压缩**

~~~bash
# 解压缩 放到/tmp文件夹下
tar -xvf nginx-1.28.0.tar.gz -C /tmp/
~~~



#### **执行配置**

~~~bash
cd /tmp/nginx-1.28.0/
./configure


# 执行配置
# useradd www
# ./configure --user=www --group=www --prefix=/usr/local/nginx --with-http_stub_status_module --with-http_ssl_module --with-stream --with-http_gzip_static_module --with-http_sub_module
 
#1、--prefix 指定安装的目录,/usr/local/nginx 是安装目录
#2、带ssl  stub_status模块 添加stream模块 –with-stream，这样就能传输tcp协议了
#3、http_stub_status_module  状态监控
#4、http_ssl_module    配置https
#5、stream  配置tcp得转发
#6、http_gzip_static_module 压缩
#7、http_sub_module  替换请求
~~~



#### **编译和安装**

~~~bash
make && make install
~~~

>补充：
>
>`make -j 4` 	并发4个任务去编译
>
>`make ; make install`		使用分号连接命令，左边的命令执行成功与否都是执行右边的
>
>`make && make install`   使用 && 连接明明，左边的命令执行成功了才会执行右边的







## Ubuntu安装和管理软件包

待更新......