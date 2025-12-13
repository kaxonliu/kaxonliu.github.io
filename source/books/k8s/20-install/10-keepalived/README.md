# k8s 集群高可用

部署 k8s 集群，node节点本身就是多个，没有单点故障，高可用的关键是 master 相关组件的高可用，负载均衡+keepalived 代理多个master 对外提供一个 vip，所有 node 节点访问这个 vip 即可。

细分方案，具体部署方式安装 etcd 的对接方式不同可以分为两种方案：

- With stacked control plane nodes（内部 etcd 服务）
- With an external etcd cluster（外部 etcd 服务）

两种方案各有利弊，在文档 Options for Highly Available topology 中有所描述，需要根据各自需求进行选择。

 参考文档：https://kubernetes.io/zh-cn/docs/setup/production-environment/tools/kubeadm/highavailability/



## 内部 etcd 服务

该类型高可用集群，提供数据存储的 etcd 服务在集群（由节点组成，由运行主节点组件的 kubeadm 管理）内部。

- 每个 master 节点会创建本地 etcd 成员，它也是只与本地的 kube-apiserver 通讯。
- master 节点运行 kube-apiserver、kube-scheduler、kube-controller-manager 组件，本节点上的 kube-controller-manager、kube-scheduler 只与本节点 kube-apiserver 通讯
- 最后由本节点的 kube-apiserver 负责对外暴露服务

优点：

- 节省主机。该集群 etcd 成员与主节点在相同节点，相对于外部 etcd 服务集群，该方案需要的主机数量少；
- 部署简单：集群部署比较简单，复制易于管理；

缺点：

- 故障风险变大。如果某一个主节点故障，其上的管理组件 kube-apiserver、kube-scheduler、kube-controller-manager 挂掉了，同时该节点上的 etcd 也一起挂掉了。



## 外部 etcd 服务

该类型高可用集群，提供数据存储的 etcd 服务在集群外部（由节点组成，由运行主节点组件的 kubeadm 管理）。

- master 节点运行 kube-apiserver、kube-scheduler、kube-controller-manager 组件。
- etcd 运行在独立主机，每个 etcd 服务与每个 master 节点上的 kube-apiserver 通讯。



优点：

- 鸡蛋没有放在一个篮子里：etcd放在主节点外的节点上，因此主节点故障或者 etcd 故障不会影响到集群冗余；

缺点：

- 更多主机：至少三台主机运行 etcd 服务，至少三台主机运行主机节点；



## etcdctl

etcdctl 是官方提供的 etcd 命令行客户端工具，用于与 etcd 集群进行交互和管理。

#### 安装

etcd 官网下载安装二进制命令。官网：https://github.com/etcd-io/etcd/releases

~~~bash
# 下载
wget https://github.com/etcd-io/etcd/releases/download/v3.6.6/etcd-v3.6.6-linux-amd64.tar.gz

# 解压
tar xzvf etcd-v3.6.6-linux-amd64.tar.gz
cd etcd-v3.6.6-linux-amd64

# 拷贝
cp etcdctl /usr/bin/
cp etcdutl /usr/bin/
~~~



#### 常用命令

可以访问单节点上的 etcd，也可以访问 etcd 集群。

- etcd 单节点：`--endpoints=https://127.0.0.1:2379`
- etcd 集群：`https://192.168.71.103:2379,https://192.168.71.101:2379,https://192.168.71.102:2379 `

~~~bash
# 查看 etcd 集群的成员列表
ETCDCTL_API=3 etcdctl \
--endpoints=https://192.168.71.103:2379,https://192.168.71.101:2379,https://192.168.71.102:2379 \
--cacert=/etc/kubernetes/pki/etcd/ca.crt \
--cert=/etc/kubernetes/pki/etcd/server.crt \
--key=/etc/kubernetes/pki/etcd/server.key \
member list

	
ETCDCTL_API=3 etcdctl \
--endpoints=https://127.0.0.1:2379 \
--cacert=/etc/kubernetes/pki/etcd/ca.crt \
--cert=/etc/kubernetes/pki/etcd/server.crt \
--key=/etc/kubernetes/pki/etcd/server.key \
member list
	
# 查看 etcd 中的所有key
ETCDCTL_API=3 etcdctl \
--endpoints=https://127.0.0.1:2379 \
--cacert=/etc/kubernetes/pki/etcd/ca.crt \
--cert=/etc/kubernetes/pki/etcd/server.crt \
--key=/etc/kubernetes/pki/etcd/server.key \
get / --prefix --keys-only


# 查看 etcd 中的指定 key
ETCDCTL_API=3 etcdctl \
--endpoints=https://127.0.0.1:2379 \
--cacert=/etc/kubernetes/pki/etcd/ca.crt \
--cert=/etc/kubernetes/pki/etcd/server.crt \
--key=/etc/kubernetes/pki/etcd/server.key \
get /registry/deployments/default/mynginx


# 数据备份---》etcd（/var/lib/etcd）
ETCDCTL_API=3 etcdctl \
--endpoints=https://127.0.0.1:2379 \
--cacert=/etc/kubernetes/pki/etcd/ca.crt \
--cert=/etc/kubernetes/pki/etcd/server.crt \
--key=/etc/kubernetes/pki/etcd/server.key \
snapshot save etcdbackupfile.db # 把当前节点的etcd数据导出为快照

# 数据恢复
mv /var/lib/etcd /var/lib/etcd_bak
mkdir /var/lib/etcd

ETCDCTL_API=3 etcdctl \
--endpoints=https://127.0.0.1:2379 \
--cacert=/etc/kubernetes/pki/etcd/ca.crt \
--cert=/etc/kubernetes/pki/etcd/server.crt \
--key=/etc/kubernetes/pki/etcd/server.key \
snapshot restore etcdbackupfile.db --data-dir=/var/lib/etcd
~~~



## etcd 数据备份和恢复

~~~bash
# 前提：备份应该周期性的（写到脚本里做成计划任务定期执行）
ETCDCTL_API=3 etcdctl \
--endpoints=https://127.0.0.1:2379 \
--cacert=/etc/kubernetes/pki/etcd/ca.crt \
--cert=/etc/kubernetes/pki/etcd/server.crt \
--key=/etc/kubernetes/pki/etcd/server.key \
snapshot save etcdbackupfile.db # 把当前节点的etcd数据导出为快照
		
		
# etcd数据库恢复步骤

# 1、把使用该etcd实例的服务给停掉
mv /etc/kubernetes/manifests /etc/kubernetes/manifests_bak

systemctl stop kubelet


# 2、数据清理
mv /var/lib/etcd /var/lib/etcd_bak

# 3、数据库还原
mkdir /var/lib/etcd

ETCDCTL_API=3 etcdctl \
--endpoints=https://127.0.0.1:2379 \
--cacert=/etc/kubernetes/pki/etcd/ca.crt \
--cert=/etc/kubernetes/pki/etcd/server.crt \
--key=/etc/kubernetes/pki/etcd/server.key \
snapshot restore etcdbackupfile.db --data-dir=/var/lib/etcd
		

# 4、重启启动服务
mv /etc/kubernetes/manifests_bak /etc/kubernetes/manifests 
systemctl restart kubelet
~~~







# 高可用部署

## 阶段1：准备工作

#### 具体部署规划

| 主机          | ip            |
| ------------- | ------------- |
| k8s-master-01 | 192.168.10.81 |
| k8s-master-02 | 192.168.10.82 |
| k8s-master-03 | 192.168.10.83 |
| k8s-node-01   | 192.168.10.84 |

- nginx+keepalived 对外提供的VIP地址为：192.168.10.200
- 每台机器 内存 >= 2G CPU >= 2



#### 1. 修改主机名和解析

~~~bash
# 1、修改主机名
hostnamectl set-hostname k8s-master-01
hostnamectl set-hostname k8s-master-02
hostnamectl set-hostname k8s-master-03
hostnamectl set-hostname k8s-node-01

# 2、三台机器添加host解析
cat >> /etc/hosts << "EOF"
192.168.10.81 k8s-master-01
192.168.10.82 k8s-master-02
192.168.10.83 k8s-master-03
192.168.10.84 k8s-node-01
192.168.10.200 api-server
EOF
~~~



#### 2. 关闭常规服务

~~~bash
# 1、关闭selinux
sed -i 's#enforcing#disabled#g' /etc/selinux/config
setenforce 0
 
# 2、禁用防火墙，网络管理，邮箱
systemctl disable --now firewalld NetworkManager postfix
 
# 3、关闭swap分区
swapoff -a 

# 注释swap分区
cp /etc/fstab /etc/fstab_bak
sed -i '/swap/d' /etc/fstab
~~~



#### 3. sshd 服务优化

~~~bash
# 1、加速访问
sed -ri 's@^#UseDNS yes@UseDNS no@g' /etc/ssh/sshd_config 
sed -ri 's#^GSSAPIAuthentication yes#GSSAPIAuthentication no#g' /etc/ssh/sshd_config 
grep ^UseDNS /etc/ssh/sshd_config 
grep ^GSSAPIAuthentication /etc/ssh/sshd_config
systemctl restart sshd
 
# 2、密钥登录（主机点做）:为了让后续一些远程拷贝操作更方便
ssh-keygen
ssh-copy-id -i root@k8s-master-01
ssh-copy-id -i root@k8s-master-02
ssh-copy-id -i root@k8s-master-03
ssh-copy-id -i root@k8s-node-01
~~~



#### 4. 增大文件描述符数量

修改后退出当前会话立即生效

~~~bash
cat > /etc/security/limits.d/k8s.conf << "EOF" 
* soft nofile 65535
* hard nofile 131070
EOF
 
ulimit -Sn 
ulimit -Hn
~~~



#### 5. 配置模块自动加载

所有节点都要配置，内核模块自动加载，该步骤必须要做。

~~~bash
modprobe br_netfilter
modprobe ip_conntrack
cat >>/etc/rc.sysinit<<EOF
#!/bin/bash
for file in /etc/sysconfig/modules/*.modules ; do
[ -x $file ] && $file
done
EOF
echo "modprobe br_netfilter" >/etc/sysconfig/modules/br_netfilter.modules
echo "modprobe ip_conntrack" >/etc/sysconfig/modules/ip_conntrack.modules
chmod 755 /etc/sysconfig/modules/br_netfilter.modules
chmod 755 /etc/sysconfig/modules/ip_conntrack.modules
lsmod | grep br_netfilter
~~~





#### 6. 同步集群时间

其中一个 maste 节点和外网同步时间，其他节点向 master 节点同步时间。

**其中一个 master 节点（比如：k8s-master-01 ）**

~~~bash
# 1、安装
yum -y install chrony

# 2、修改配置文件
mv /etc/chrony.conf /etc/chrony.conf.bak
cat > /etc/chrony.conf << EOF
server ntp1.aliyun.com iburst minpoll 4 maxpoll 10
server ntp2.aliyun.com iburst minpoll 4 maxpoll 10
server ntp3.aliyun.com iburst minpoll 4 maxpoll 10
server ntp4.aliyun.com iburst minpoll 4 maxpoll 10
server ntp5.aliyun.com iburst minpoll 4 maxpoll 10
server ntp6.aliyun.com iburst minpoll 4 maxpoll 10
server ntp7.aliyun.com iburst minpoll 4 maxpoll 10
driftfile /var/lib/chrony/drift
makestep 10 3
rtcsync
allow 0.0.0.0/0
local stratum 10
keyfile /etc/chrony.keys
logdir /var/log/chrony
stratumweight 0.05
noclientlog
logchange 0.5
 
EOF

# 3、启动chronyd服务
systemctl restart chronyd.service
systemctl enable chronyd.service
systemctl status chronyd.service

# 4、验证
chronyc sources -v

# 看到如下输出
^* 223.4.249.80                  2   4   377     4  -8407ns[  +99us] +/-   24ms
^+ 203.107.6.88                  2   4   373     9   +562us[ +562us] +/-   22ms
~~~

**其他节点（向：k8s-master-01  看起）**

~~~bash
# 1、安装 chrony
yum -y install chrony

# 2、需改客户端配置文件
mv /etc/chrony.conf /etc/chrony.conf.bak
cat > /etc/chrony.conf << EOF
server k8s-master-01 iburst
driftfile /var/lib/chrony/drift
makestep 10 3
rtcsync
local stratum 10
keyfile /etc/chrony.key
logdir /var/log/chrony
stratumweight 0.05
noclientlog
logchange 0.5
 
EOF

# 3、启动chronyd
systemctl restart chronyd.service
systemctl enable chronyd.service
systemctl status chronyd.service
 
# 4、验证
chronyc sources -v

# 看到如下输出
^* k8s-master-01                 3   6    17     8    -26us[  -90us] +/-   24ms
~~~



#### 7. 更新 yum 源

三个节点都要做。

~~~bash
# 1、清理
rm -rf /etc/yum.repos.d/*
yum remove epel-release -y
rm -rf /var/cache/yum/x86_64/6/epel/
 
# 2、安装阿里的base与epel源
curl -s -o /etc/yum.repos.d/CentOS-Base.repo https://mirrors.aliyun.com/repo/Centos-7.repo 
curl -s -o /etc/yum.repos.d/epel.repo http://mirrors.aliyun.com/repo/epel-7.repo
yum clean all
yum makecache
~~~

#### 8. 更新系统软件

更新系统软件，但是排除更新内核

~~~bash
 yum update -y --exclud=kernel*
~~~

#### 9. 安装基础软件

~~~bash
yum install -y expect wget jq psmisc vim net-tools telnet yum-utils device-mapper-persistent-data lvm2 git ntpdate chrony bind-utils rsync unzip git
~~~

#### 10. 更新内核

内核版本最好是 4.4 以上。

~~~bash
# 下载两个内核包
wget https://elrepo.org/linux/kernel/el7/x86_64/RPMS/kernel-lt-5.4.274-1.el7.elrepo.x86_64.rpm
wget https://elrepo.org/linux/kernel/el7/x86_64/RPMS/kernel-lt-devel-5.4.274-1.el7.elrepo.x86_64.rpm


# 安装
yum localinstall -y kernel-lt*
 
#调到默认启动
grub2-set-default 0 && grub2-mkconfig -o /etc/grub2.cfg 
 
#查看当前默认启动的内核
grubby --default-kernel
 
#重启系统
reboot
~~~

#### 11. 安装 ipvs

三个节点都要安装 ipvs

~~~bash
# 1、安装ipvsadm等相关工具
yum -y install ipvsadm ipset sysstat conntrack libseccomp 
 
# 2、配置加载
cat > /etc/sysconfig/modules/ipvs.modules <<"EOF" 
#!/bin/bash 
ipvs_modules="ip_vs ip_vs_lc ip_vs_wlc ip_vs_rr ip_vs_wrr ip_vs_lblc ip_vs_lblcr ip_vs_dh ip_vs_sh ip_vs_fo ip_vs_nq ip_vs_sed ip_vs_ftp nf_conntrack" 
 
for kernel_module in ${ipvs_modules}; 
do 
	/sbin/modinfo -F filename ${kernel_module} > /dev/null 2>&1 
	if [ $? -eq 0 ]; then 
		/sbin/modprobe ${kernel_module} 
	fi 
done 
EOF

# 授权
chmod 755 /etc/sysconfig/modules/ipvs.modules && bash /etc/sysconfig/modules/ipvs.modules && lsmod | grep ip_vs
~~~

#### 12. 修改内核参数

三个节点都要修改内核参数

~~~bash
cat > /etc/sysctl.d/k8s.conf << EOF
net.ipv4.ip_forward = 1
net.bridge.bridge-nf-call-iptables = 1
net.bridge.bridge-nf-call-ip6tables = 1
fs.may_detach_mounts = 1
vm.overcommit_memory=1
vm.panic_on_oom=0
fs.inotify.max_user_watches=89100
fs.file-max=52706963
fs.nr_open=52706963
net.ipv4.tcp_keepalive_time = 600
net.ipv4.tcp.keepaliv.probes = 3
net.ipv4.tcp_keepalive_intvl = 15
net.ipv4.tcp.max_tw_buckets = 36000
net.ipv4.tcp_tw_reuse = 1
net.ipv4.tcp.max_orphans = 327680
net.ipv4.tcp_orphan_retries = 3
net.ipv4.tcp_syncookies = 1
net.ipv4.tcp_max_syn_backlog = 16384
net.ipv4.ip_conntrack_max = 65536
net.ipv4.tcp_max_syn_backlog = 16384
net.ipv4.top_timestamps = 0
net.core.somaxconn = 16384
EOF
 
# 立即生效
sysctl --system
~~~



## 阶段2：安装 containerd

四个节点都要安装 containerd。

>因为从 k8s 1.24 以后，直接对接 containerd，默认不再支持 docker。



#### 1. 升级 libseccomp

centos7.9 默认的 libseccomp 版本为 2.3.1，不满足 containerd 的需求，需要更新到 2.4 以上版本。

~~~bash
# 查看老版本
rpm -qa | grep libseccomp
libseccomp-2.3.1-4.el7.x86_64

# 卸载旧版本
rpm -e libseccomp-2.3.1-4.el7.x86_64 --nodeps

# 下载新版本（阿里云）
wget https://mirrors.aliyun.com/centos/8/BaseOS/x86_64/os/Packages/libseccomp-2.5.1-1.el8.x86_64.rpm

# 安装新版本
rpm -ivh libseccomp-2.5.1-1.el8.x86_64.rpm
~~~



#### 2. 安装 containerd

推荐使用阿里云的源安装。

~~~bash
# 1、卸载之前的
yum remove docker docker-ce containerd docker-common docker-selinux docker-engine -y

# 2、准备repo
cd /etc/yum.repos.d/
wget http://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo

# 3、安装
yum install containerd* -y
~~~



#### 3. 配置 containerd

~~~bash
# 1. 生成配置文件
mkdir -pv /etc/containerd
containerd config default > /etc/containerd/config.toml


# 2、替换默认 pause 镜像地址
grep sandbox_image /etc/containerd/config.toml
sed -i 's/registry.k8s.io/registry.cn-hangzhou.aliyuncs.com\/google_containers/' /etc/containerd/config.toml

grep sandbox_image /etc/containerd/config.toml
# 请务必确认新地址是可用的：sandbox_image = "registry.cn-hangzhou.aliyuncs.com/goggle_containers/pause:3.6"

# 3、配置 systemd 作为容器的 group driver
grep SystemdCgroup /etc/containerd/config.toml
sed -i 's/SystemdCgroup \= false/SystemdCgroup \= true/' /etc/containerd/config.toml
grep SystemdCgroup /etc/containerd/config.toml

# 4、配置加速器 (必须配置)
# 否则后续安装cn1网络插件时无法从docker.io里下载镜像
# 参考：https://github.com/containerd/containerd/blob/main/docs/cr1/config.md#registry-configuration
# 添加 config_path = "/etc/containerd/certs.d"

grep config_path /etc/containerd/config.toml
# 设置一个配置路径 config_path = "/etc/containerd/certs.d"
sed -i 's/config_path\ =.*/config_path = \"\/etc\/containerd\/certs.d\"/g' /etc/containerd/config.toml

# 创建新的配置路径
mkdir -p /etc/containerd/certs.d/docker.io
cat > /etc/containerd/certs.d/docker.io/hosts.toml << EOF
server = "https://docker.io"
[host."https://dockerproxy.com"]
capabilities = ["pull", "resolve"]

[host."https://docker.m.daocloud.io"]
capabilities = ["pull", "resolve"]

[host."https://docker.agsv.top"]
capabilities = ["pull", "resolve"]

[host."https://registry.docker-cn.com"]
capabilities = ["pull", "resolve"]

EOF


# 5 配置 containerd 开启自启
systemctl daemon-reload
systemctl restart containerd
systemctl enable --now containerd
systemctl status containerd

# 查看 containerd 版本
ctr version
~~~



## 阶段3：部署负载均衡+keepalived

部署负载均衡+keepalived 对外提供 vip：192.168.10.200



#### 1. master 上部署 nginx

所有的 master 节点都要安装部署 nginx ，所有节点上的安装部署配置完全一样。

这里是为了方便，直接在 master 节点上直接安装的 nginx，实际上可以在集群外部署 nginx 代理服务。

~~~bash
# 1、添加repo源
cat > /etc/yum.repos.d/nginx.repo << "EOF"
[nginx-stable]
name=nginx stable repo
baseurl=http://nginx.org/packages/centos/$releasever/$basearch/
gpgcheck=1
enabled=1
gpgkey=https://nginx.org/keys/nginx_signing.key
module_hotfixes=true
[nginx-mainline]
name=nginx mainline repo
baseurl=http://nginx.org/packages/mainline/centos/$releasever/$basearch/
gpgcheck=1
enabled=0
gpgkey=https://nginx.org/keys/nginx_signing.key
module_hotfixes=true
EOF

# 2、安装
yum install nginx -y

# 3、配置
cat > /etc/nginx/nginx.conf <<'EOF'
user nginx nginx;
worker_processes auto;

events {
    worker_connections 20240;
    use epoll;
}

error_log /var/log/nginx_error.log info;

stream {
    # Kubernetes API Server 负载均衡配置
    upstream kube-servers {
        hash $remote_addr consistent;
        
        # 主服务器节点配置
        server k8s-master-01:6443 weight=5 max_fails=1 fail_timeout=3s;
        server k8s-master-02:6443 weight=5 max_fails=1 fail_timeout=3s;
        server k8s-master-03:6443 weight=5 max_fails=1 fail_timeout=3s;
    }
    
    server {
        listen 8443 reuseport;  # 监听 8443 端口
        proxy_connect_timeout 3s;
        proxy_timeout 3000s;
        proxy_pass kube-servers;
        
        # 可选：添加访问日志
        # access_log /var/log/nginx_stream_access.log;
    }
}
EOF


# 4、安装并启动
systemctl restart nginx
systemctl enable nginx
systemctl status nginx
~~~



#### 2. nginx 部署 keepalived

为了方便，直接在 master 节点上直接安装的 nginx。这里也是在 master 节点上使用 keepalived 高可用 nginx 代理服务。

~~~bash
# 1. 安装
yum -y install keepalived

# 2. 修改keepalive的配置文件
# 根据实际环境，interface eth0可能需要修改为interface ens33

# ==================================> k8s-master-01
cat > /etc/keepalived/keepalived.conf <<'EOF'
! Configuration File for keepalived

global_defs {
    router_id 192.168.10.81
}

vrrp_script chk_nginx {
    script "/etc/keepalived/check_port.sh 8443"
    interval 2
    weight -20
}

vrrp_instance VI_1 {
    state BACKUP
    interface ens33
    virtual_router_id 100
    priority 100
    advert_int 1
    mcast_src_ip 192.168.10.81
    
    # 注意：这行注释掉，否则即使一个具有更高优先级的备份节点出现，
    # 当前的 MASTER 也不会被抢占，直至 MASTER 失效。
    # nopreempt
    
    authentication {
        auth_type PASS
        auth_pass 11111111
    }
    
    track_script {
        chk_nginx
    }
    
    virtual_ipaddress {
        192.168.10.200
    }
}
EOF

# ==================================> k8s-master-02
cat > /etc/keepalived/keepalived.conf <<'EOF'
! Configuration File for keepalived

global_defs {
    router_id 192.168.10.82
}

vrrp_script chk_nginx {
    script "/etc/keepalived/check_port.sh 8443"
    interval 2
    weight -20
}

vrrp_instance VI_1 {
    state BACKUP
    interface ens33
    virtual_router_id 100
    priority 100
    advert_int 1
    mcast_src_ip 192.168.10.82
    
    # 注意：这行注释掉，否则即使一个具有更高优先级的备份节点出现，
    # 当前的 MASTER 也不会被抢占，直至 MASTER 失效。
    # nopreempt
    
    authentication {
        auth_type PASS
        auth_pass 11111111
    }
    
    track_script {
        chk_nginx
    }
    
    virtual_ipaddress {
        192.168.10.200
    }
}
EOF

# ==================================> k8s-master-03
cat > /etc/keepalived/keepalived.conf <<'EOF'
! Configuration File for keepalived

global_defs {
    router_id 192.168.10.83
}

vrrp_script chk_nginx {
    script "/etc/keepalived/check_port.sh 8443"
    interval 2
    weight -20
}

vrrp_instance VI_1 {
    state BACKUP
    interface ens33
    virtual_router_id 100
    priority 100
    advert_int 1
    mcast_src_ip 192.168.10.83
    
    # 注意：这行注释掉，否则即使一个具有更高优先级的备份节点出现，
    # 当前的 MASTER 也不会被抢占，直至 MASTER 失效。
    # nopreempt
    
    authentication {
        auth_type PASS
        auth_pass 11111111
    }
    
    track_script {
        chk_nginx
    }
    
    virtual_ipaddress {
        192.168.10.200
    }
}
EOF


# 3. 所有master节点上创建健康检查脚本
cat > /etc/keepalived/check_port.sh << 'EOF'
vi /etc/keepalived/check_port.sh 
 
#!/bin/bash
count=$(ps -C nginx --no-header|wc -l)
#1.判断 Nginx 是否存活,如果不存活则尝试启动 Nginx
if [ $count -eq 0 ];then
    systemctl start nginx &>/dev/null &  # 放后台运行防止超时
    sleep 1  # 不要sleep太久，超过了keepalived.conf中配置的interval时间就麻烦了
    #2.等待 1 秒后再次获取一次 haproxy 状态# 
    count=$(ps -C  nginx --no-header|wc -l)
    #3.再次进行判断, 如haproxy 还不存活则停止 Keepalived,让地址进行漂移,并退出脚本
    if [ $count -eq 0 ];then
        systemctl stop keepalived
    fi
fi
EOF

chmod +x /etc/keepalived/check_port.sh


# 4. 安装并启动
systemctl restart keepalived
systemctl enable keepalived
systemctl status keepalived

# 5. 查看 vip
# 在其中一个 master 上可以看到 vip
# 如下命令发现 vip 现在在 k8s-master-03 机器上
[root@k8s-master-03 ~]# ip addr show dev ens33
2: ens33: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UP group default qlen 1000
    link/ether 00:0c:29:4b:81:5f brd ff:ff:ff:ff:ff:ff
    inet 192.168.10.83/24 brd 192.168.10.255 scope global ens33
       valid_lft forever preferred_lft forever
    inet 192.168.10.200/32 scope global ens33
       valid_lft forever preferred_lft forever
    inet6 2409:8a20:4d41:9694:20c:29ff:fe4b:815f/64 scope global mngtmpaddr dynamic 
       valid_lft 84911sec preferred_lft 2111sec
    inet6 fe80::20c:29ff:fe4b:815f/64 scope link 
       valid_lft forever preferred_lft forever
[root@k8s-master-03 ~]# 
[root@k8s-master-03 ~]#
~~~



## 阶段4：安装 k8s

#### 1. 为所有集群机器准备 k8s 源

参考阿里云源使用说明：https://developer.aliyun.com/mirror/kubernetes/

~~~bash
# 准备 yum 源
cat > /etc/yum.repos.d/kubernetes.repo << "EOF"
[kubernetes]
name=kubernetes
baseurl=https://mirrors.aliyun.com/kubernetes-new/core/stable/v1.30/rpm/
enabled=1
gpgcheck=1
gpgkey=https://mirrors.aliyun.com/kubernetes-new/core/stable/v1.30/rpm/repodata/repomd.xml.key
EOF

setenforce 0
yum install -y kubelet-1.30* kubeadm-1.30* kubectl-1.30*
systemctl enable kubelet && systemctl start kubelet && systemctl status kubelet
~~~



#### 2. k8s-master-01上操作

初始化 master 节点（仅在 k8s-master-01 节点上执行，其他 master 节点和 node 节点不需要操作，后面直接加入即可。）

使用 `kubeadm config images list` 命令查看 Kubernetes 1.30.0 版本需要下载的镜像列表。

~~~bash
[root@k8s-master-01 ~]# kubeadm config images list
I1104 23:11:04.953335    1715 version.go:256] remote version is much newer: v1.34.1; falling back to: stable-1.30
registry.k8s.io/kube-apiserver:v1.30.14
registry.k8s.io/kube-controller-manager:v1.30.14
registry.k8s.io/kube-scheduler:v1.30.14
registry.k8s.io/kube-proxy:v1.30.14
registry.k8s.io/coredns/coredns:v1.11.3
registry.k8s.io/pause:3.9
registry.k8s.io/etcd:3.5.15-0
~~~

生成配置文件，修改编辑。

~~~bash
cd /root
kubeadm config print init-defaults > kubeadm.yaml
~~~

生成的配置文件

~~~yaml
apiVersion: kubeadm.k8s.io/v1beta3
bootstrapTokens:
- groups:
  - system:bootstrappers:kubeadm:default-node-token
  token: abcdef.0123456789abcdef
  ttl: 24h0m0s
  usages:
  - signing
  - authentication
kind: InitConfiguration
localAPIEndpoint:
  # 修改
  # 统一监听在0.0.0.0即可
  advertiseAddress: 0.0.0.0
  bindPort: 6443
nodeRegistration:
  criSocket: unix:///var/run/containerd/containerd.sock
  imagePullPolicy: IfNotPresent
  # 主机名
  name: k8s-master-01
  taints: null
---
apiServer:
  timeoutForControlPlane: 4m0s
apiVersion: kubeadm.k8s.io/v1beta3
certificatesDir: /etc/kubernetes/pki
clusterName: kubernetes
controllerManager: {}
dns: {}
etcd:
  local:
    dataDir: /var/lib/etcd
# # 换成阿里云镜像仓库地址
imageRepository: registry.cn-hangzhou.aliyuncs.com/google_containers
kind: ClusterConfiguration
kubernetesVersion: 1.30.0
# 增加如下一行配置
# 指定你的vip地址192.168.10.200与负载均可暴露的端口，建议用主机名
controlPlaneEndpoint: "api-server:8443"
networking:
  dnsDomain: cluster.local
  serviceSubnet: 10.96.0.0/12
  podSubnet: 10.244.0.0/16 # 增加一行，指定pod网段
scheduler: {}

# 在文件最后，插入以下内容，（复制时，要带着--）：
---
apiVersion: kubeproxy.config.k8s.io/v1alpha1
kind: KubeProxyConfiguration
mode: ipvs
---
apiVersion: kubelet.config.k8s.io/v1beta1
kind: KubeletConfiguration
cgroupDriver: systemd
~~~

- 修改 `advertiseAddress: 0.0.0.0` 统一监听在 0.0.0.0 即可
- 修改 `imageRepository: registry.cn-hangzhou.aliyuncs.com/google_containers`  换成阿里云镜像仓库地址
- 增加 `controlPlaneEndpoint: "api-server:8443"`



部署

~~~bash
kubeadm init --config=kubeadm.yaml \
--ignore-preflight-errors=SystemVerification \
--ignore-preflight-errors=Swap
~~~



看到如下输出信息说明部署好了，然后按照提示执行就好了。

~~~bash
Your Kubernetes control-plane has initialized successfully!

To start using your cluster, you need to run the following as a regular user:

  mkdir -p $HOME/.kube
  sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
  sudo chown $(id -u):$(id -g) $HOME/.kube/config

Alternatively, if you are the root user, you can run:

  export KUBECONFIG=/etc/kubernetes/admin.conf

You should now deploy a pod network to the cluster.
Run "kubectl apply -f [podnetwork].yaml" with one of the options listed at:
  https://kubernetes.io/docs/concepts/cluster-administration/addons/

You can now join any number of control-plane nodes by copying certificate authorities
and service account keys on each node and then running the following as root:

  kubeadm join api-server:8443 --token abcdef.0123456789abcdef \
	--discovery-token-ca-cert-hash sha256:374c0cd0dad40f089c319b9d7966020f0a93897fdbf97ea10c54c064d7e671e5 \
	--control-plane 

Then you can join any number of worker nodes by running the following on each as root:

kubeadm join api-server:8443 --token abcdef.0123456789abcdef \
	--discovery-token-ca-cert-hash sha256:374c0cd0dad40f089c319b9d7966020f0a93897fdbf97ea10c54c064d7e671e5 
~~~



根据上述输出结果执行命令

~~~bash
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
~~~



查看 node

~~~bash
root@k8s-master-01 ~]# kubectl get nodes
NAME            STATUS     ROLES           AGE     VERSION
k8s-master-01   NotReady   control-plane   3m14s   v1.30.14
[root@k8s-master-01 ~]# 
[root@k8s-master-01 ~]# 
[root@k8s-master-01 ~]# kubectl -n kube-system get pods
NAME                                    READY   STATUS    RESTARTS   AGE
coredns-6d58d46f65-bl7tv                0/1     Pending   0          3m30s
coredns-6d58d46f65-wt2vf                0/1     Pending   0          3m29s
etcd-k8s-master-01                      1/1     Running   0          3m44s
kube-apiserver-k8s-master-01            1/1     Running   0          3m44s
kube-controller-manager-k8s-master-01   1/1     Running   0          3m43s
kube-proxy-4pkbs                        1/1     Running   0          3m30s
kube-scheduler-k8s-master-01            1/1     Running   0          3m43s
~~~



#### 3. 多个 master 节点加入集群

~~~bash
# 1. 上传/统一证书
# 将集群控制平面所需的证书上传到集群中，并存储在一个叫做 kubeadm-certs 的 Secret 中
# 如果在安装的时候 kubeadm init 的默认加上选项--upload-certs来上传证书，那这一步就不需要了
# 在 k8s-master-01 上执行如下命令
kubeadm init phase upload-certs --upload-certs

输出结果
...
[upload-certs] Using certificate key:
cbe98d53f45487993c51d9ece749a8b4cec03f3093068d75ecf99fa2dd5bd7ee 
# 得到该值，后续添加master需要用到它
# 上述指令做了3件事
- 1. 上传证书到集群：将已经生成的控制平面证书上传到 Kubernetes 集群，使得这些证书可以在多个Master 节点之间共享。
- 2. 生成证书密钥：生成一个唯一的密钥用于加密和保护这些证书。
- 3. 输出证书密钥：在终端输出这个证书密钥，后续添加新的 Master 节点时需要使用。

# 上述指令目的是：
- 方便其他 Master 节点在加入集群时能够共享这些证书。这个步骤极大简化了多 Master 节点的配置过程，确保所有 Master 节点使用相同的证书
是否一定要做这一步？
- 不一定。如果你没有这一步骤，那么每一个新加入的 Master 节点会自己生成一份新的证书，这样可能导致集群内的证书不一致，增加管理复杂度。使用 kubeadm init phase upload-certs --upload-certs 可以确保所有控制平面节点使用相同的证书，简化管理并且提高安全性和一致性。因此，尽管这一步不是绝对必要的，但强烈推荐使用它


# 2. 添加 master 节点到集群
# 在其余 master 节点上执行如下命令 以 master 身份加入集群
kubeadm join api-server:8443 --token abcdef.0123456789abcdef \
--discovery-token-ca-cert-hash sha256:374c0cd0dad40f089c319b9d7966020f0a93897fdbf97ea10c54c064d7e671e5 \
--control-plane \
--certificate-key cbe98d53f45487993c51d9ece749a8b4cec03f3093068d75ecf99fa2dd5bd7ee

# 3. 成功加入后在其余的 master 节点上按照提示执行
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
~~~



#### 4. worker 节点加入集群

~~~bash
kubeadm join api-server:8443 --token abcdef.0123456789abcdef \
	--discovery-token-ca-cert-hash sha256:374c0cd0dad40f089c319b9d7966020f0a93897fdbf97ea10c54c064d7e671e5
~~~



#### 5. 查看所有节点

在 master 节点上执行查看 node 的指令： 最开始时 NotReady 状态正常，因为网络组件没有部署 ok

~~~bash
[root@k8s-master-01 ~]# kubectl -n kube-system get pods
NAME                                    READY   STATUS    RESTARTS   AGE
coredns-6d58d46f65-bl7tv                0/1     Pending   0          17m
coredns-6d58d46f65-wt2vf                0/1     Pending   0          17m
etcd-k8s-master-01                      1/1     Running   0          17m
etcd-k8s-master-02                      1/1     Running   0          3m14s
etcd-k8s-master-03                      1/1     Running   0          2m50s
kube-apiserver-k8s-master-01            1/1     Running   0          17m
kube-apiserver-k8s-master-02            1/1     Running   0          3m14s
kube-apiserver-k8s-master-03            1/1     Running   0          2m38s
kube-controller-manager-k8s-master-01   1/1     Running   0          17m
kube-controller-manager-k8s-master-02   1/1     Running   0          3m14s
kube-controller-manager-k8s-master-03   1/1     Running   0          2m38s
kube-proxy-4pkbs                        1/1     Running   0          17m
kube-proxy-drdk6                        1/1     Running   0          40s
kube-proxy-kst5f                        1/1     Running   0          3m14s
kube-proxy-zldbq                        1/1     Running   0          3m
kube-scheduler-k8s-master-01            1/1     Running   0          17m
kube-scheduler-k8s-master-02            1/1     Running   0          3m14s
kube-scheduler-k8s-master-03            1/1     Running   0          2m33s
~~~



#### 6. 安装网络插件

下载网络插件

~~~bash
wget https://github.com/flannel-io/flannel/releases/latest/download/kube-flannel.yml
 
[root@master01 flannel]# vim kube-flannel.yml 
 
apiVersion: v1
data:
  ...
  net-conf.json: |
    {
      "Network": "10.244.0.0/16", # 与--pod-network-cidr保持一致
      "Backend": {
        "Type": "vxlan"
      }
    }
~~~

部署

~~~bash
kubectl apply -f kube-flannel.yml
~~~

- 如果镜像拉取失败，需要换过内镜像源



测试

~~~bash
kubectl -n kube-flannel get pods
kubectl -n kube-flannel get pods -o wide
[root@k8s-master-01 ~]# kubectl get nodes # 全部ready
[root@k8s-master-01 ~]# kubectl -n kube-system get pods # 两个coredns的pod也都ready
~~~



#### 7. 安装 kubectl 命令提示

所有节点安装，node 节点的家目录可能没有 `./kube` 目录，没有的话手动创建。

~~~bash
yum install bash-completion* -y
 
kubectl completion bash > ~/.kube/completion.bash.inc
echo "source '$HOME/.kube/completion.bash.inc'" >> $HOME/.bash_profile
source $HOME/.bash_profile
~~~



# 升级 k8s

## 注意事项

（1）升级前最好备份所有组件及数据，例如 etcd 

（2）不要跨两个大版本进行升级，可能会存在版本 bug，如：

~~~bash
1.19.4-->1.21.4
可以先从1.19.4 升级到1.20.4
然后再从1.20.4 升级到 1.21.4
~~~

（3）升级前要在测试环境进行充分的演练，充分考虑到回滚方案（事先将回滚动作制作成脚本，方便生 产环境遇到问题时快速响应）

## 升级方案

对于线上环境，升级 k8s 需要做到不中断当前业务容器的灰度升级，常见方案有如下两种 

（1）方案一：蓝绿方式 

仿照现有k8s集群，部署一套新环境，在新环境里部署指定版本的 k8s，然后把业务应用也部署到新环境。然后将流量切换到新环境 

（2）方案二：先更新 master 上的 k8s 服务版本，再逐个批量更新 Node 上的 k8s 服务版本（高版本 的master通常可以管理更低版本的 node，但版本差异过大也会有问题） 

蓝绿的方式没啥没说的（就是部署一套全新环境，然后切流量即可），我们直接看方案二。



## 升级步骤

#### 1. 配置新的 yum 源

配置阿里云 yum 源，每台节点都需要配置（参考： https://developer.aliyun.com/mirror/kubernetes/）

老版本的 k8s 是 1.30 现在计划升级到 1.31版本，修改把 yum 源做如下修改。所有的节点都要更新 yum 源。

~~~bash
# 老版本
[root@k8s-master-01 ~]# cat /etc/yum.repos.d/kubernetes.repo 
[kubernetes]
name=kubernetes
baseurl=https://mirrors.aliyun.com/kubernetes-new/core/stable/v1.30/rpm/
enabled=1
gpgcheck=1
gpgkey=https://mirrors.aliyun.com/kubernetes-new/core/stable/v1.30/rpm/repodata/repomd.xml.key


# 新版本
cat > /etc/yum.repos.d/kubernetes.repo << 'EOF'
[kubernetes]
name=kubernetes
baseurl=https://mirrors.aliyun.com/kubernetes-new/core/stable/v1.31/rpm/
enabled=1
gpgcheck=1
gpgkey=https://mirrors.aliyun.com/kubernetes-new/core/stable/v1.31/rpm/repodata/repomd.xml.key
EOF
~~~

配置后更新yum源，执行命令：

~~~bash
yum clean all
yum makecache
~~~



#### 2. 确认升级次数

假设我的版本是 v1.20.4，假设官网最新版本已经到1.25.0。升级到最新版本需要逐步升级：v1.20.4-- >v1.21.4-->v1.22.4-->v1.23.4-->v1.24.4-->v1.25.0，共升级5次。

目前我的版本是 v1.30.14 ，需要升级到 v1.31.14：只需要升级一次即可。



#### 3. 升级节点步骤综述

整体顺序是：先升 master 再升级 worker

级或更新节点步骤：（其他node节点如果要升级，则采用一样的步骤）

1. 先隔离Node节点的业务流量，备份好数据（如果是master节点则记着备份/var/lib/etcd目录） 
2. cordon禁止新pod调度到当前 node  # kubectl cordon node master01 
3. 对关键服务创建PDB保护策略，确保下一步的排空时，关键服务的pod至少有1个副本可用（在 当前节点以外的节点上有分布） 
4. drain 排空业务pod（静态pod不可能被排空，我们要升级的就是静态pod及kubelet等组件）
5. 升级当前 node 上的软件 
6. uncordon 当前 node



迁移的带来的服务中断问题

- 当 node 节点关机后，k8s 在等待大约6（检测（10s+40s）+300s容忍时间）分钟后，会自动将停机 node 节点上的 pod 自动迁移到其他 node 节点上。如果某个服务的 pod 只有一个副本，那么在此期间服务是中断的。

如何避免等待默认的6分钟带来的服务中断问题

- 一方面：某一个服务的 pod 副本应该有多个，而且你应该在迁移前就确保这多个副本并没有完全集中在同一个节点上，如果有，那么需要提供控制调度到其他节点。
- 另外一方面：在节点不可用之前（停机维护或者升级），将 pod 先驱逐到别的节点（以此规避不必要的6分钟等 待） 需要用到 cordon、drain、uncordor 三个命令实现节点的主动维护。

~~~bash
cordon：标记节点不可调度，后续新的pod不会被调度到此节点，但是该节点上的pod可以正常对外服务；
drain：驱逐节点上的pod至其他可调度节点；
uncordon：重新标记节点可调度；

# 1.标记节点不可调度
kubectl cordon k8s-master-01

# 2.驱逐pod
kubectl drain k8s-master-01 --delete-local-data --ignore-daemonsets --force

# 参数如下：
--delete-local-data 删除本地数据（关键数据要提前备份好），即使emptyDir也将删除（新版本不再
支持该选型）
--ignore-daemonsets 忽略DeamonSet，否则DeamonSet被删除后，仍会自动重建；
--force 不加force参数只会删除该node节点上的ReplicationController, ReplicaSet,
DaemonSet,StatefulSet or Job，加上后所有pod都将删除

# 3. 查看驱逐，
kubectl get pod -n test -o wide
~~~

为了实现平滑迁移 pod，我们可以添加 PDB 策略来对一些关键服务进行硬性限制，加了之后再去 drain 排空的时候， 如果存在一些受 PDB 保护的应用，则会 drain 失败，此时会硬性要求你先扩容副本，因为你已经 cordon 过了，再启新副本一定是在新节点上。

#### 4. 开始升级

先升级 master 节点中的一个节点。比如选择首先升级 k8s-master-01 节点。

1、先隔离 Node 节点的业务流量，备份好数据（如果是 master 节点则记着备份 /var/lib/etcd 目录）

2、cordon禁止新pod调度到当前node

~~~bash
kubectl cordon k8s-master-01
~~~

3、对关键服务创建 PDB 保护策略(无则略)

4、drain 排空业务 pod（静态pod不可能被排空，我们要升级的就是静态 pod 及 kubelet 等组件）

~~~bash
kubectl drain k8s-master-01 --delete-local-data --ignore-daemonsets --force
~~~

5、升级当前 node 上的软件

6、如果 master 节点是高可用集群，那么需要在当前 master 节点之外的其他 master 节点上执行下述命令

~~~bash
kubeadm upgrade node
~~~

7、uncordon 当前 node

~~~bash
kubectl uncordon k8s-master-01
~~~



#### 5. 升级 node 节点的软件

如果是高可用集群，找其中一台 master 执行以下命令。比如选择先升级 k8s-master-01

~~~bash
# 在 k8s-master-01 节点上依次执行如下命令
# 安装新版本 kubeadm
yum install -y kubeadm-1.31* --disableexcludes=kubernetes

# kubeadm 更新计划
# kubeadm 更新计划会打印出目前能支持到的版本
# 在打印信息中可以看到，升级集群每个组件对应的当前版本和升级后的版本。
# 而且升级的组件只包括 kube-apiserver，kube-controller-manager，kube-scheduler，kube-proxy，CoreDNS，etcd
# 不包括kubectl,kubelet,docker和网络组件flannel等
kubeadm upgrade plan

# 更新
# 根据上面的输出提示，执行如下命令以升级
kubeadm upgrade apply v1.31.14
# 看到如下信息表示 OK
[upgrade/successful] SUCCESS! Your cluster was upgraded to "v1.31.14". Enjoy!

[upgrade/kubelet] Now that your control plane is upgraded, please proceed with upgrading your kubelets if you haven't already done so.


# 在 k8s-master-01 继续安装 kubelet kubectl
yum install -y kubelet-1.31* kubectl-1.31* --disableexcludes=kubernetes

# 重启 kubelet
systemctl daemon-reload && systemctl restart kubelet

# 查看新版本信息
[root@k8s-master-01 ~]# kubectl get nodes
NAME            STATUS                     ROLES           AGE    VERSION
k8s-master-01   Ready,SchedulingDisabled   control-plane   3h6m   v1.31.14
k8s-master-02   Ready                      control-plane   172m   v1.30.14
k8s-master-03   Ready                      control-plane   172m   v1.30.14
k8s-node-01     Ready                      <none>          169m   v1.30.14
~~~



#### 6. 按照上述流程继续升级其他 master 节点

#### 7. 按照上述流程继续升级 node 节点

升级流程和 master 节点设计保持一致。不过少了如下步骤

~~~bash
kubeadm upgrade apply v1.31.14
~~~



#### 8. 检查集群状态

版本升级完成。

~~~bash
[root@k8s-node-01 ~]# kubectl get node
NAME            STATUS   ROLES           AGE     VERSION
k8s-master-01   Ready    control-plane   3h27m   v1.31.14
k8s-master-02   Ready    control-plane   3h12m   v1.31.14
k8s-master-03   Ready    control-plane   3h12m   v1.31.14
k8s-node-01     Ready    <none>          3h10m   v1.31.14
~~~





## k8s 升级注意事项

~~~bash
1、公司有一定规模，架构时：采用异地双活的方案，DNS层轮询AB两个机房
	升级时，dns解析去掉一个机房，进行专门的升级（重部一套新的），后再加入，然后再升级另外一个机房
	-----》本质就是蓝绿发布
	
2、如果公司规模很小，就一套k8s集群，但用的都是云主机，那建议还是蓝绿最靠谱
 
3、如果公司规模很小，就一套k8s集群，并且用的都是物理机，这种只能按部就班的升级，提前在测试环境演练好，做好回滚方案，在晚上进行，开发、测试、运维都一齐加班
Kubernetes的升级过程需要考虑多个维度，包括集群中的主节点和工作节点、在Pod内运行的应用程序以及API版本。下面是一个基本的步骤来平滑地升级您的Kubernetes集群：
1、备份和计划：在进行任何升级操作之前，一定要备份你的etcd数据存储以及任何相关的配置文件。同时，明确升级时间，通知涉及人员等。
2、先升级主节点：使用kubeadm upgrade plan命令检查可用的升级。然后，使用kubeadm upgrade apply命令升级你的主节点。这将包括Kubernetes API server，控制管理器，调度程序，和默认DNS服务器等组件。
3、升级工作节点：升级完主节点后，你可以逐个将工作节点从集群中切出，升级，并重新加入集群。这个过程可以使用kubectl drain（用于准备和停止节点上的工作负荷），kubeadm upgrade node（用于实际升级节点），和kubectl uncordon（让节点可以被调度）命令来完成。
4、升级网络插件：主节点和工作节点完成升级后，如果你使用了网络插件，还需要按照插件提供商的指导进行升级。
5、验证升级：升级完成后，使用kubectl get nodes命令确认所有的节点都已经升级到新的版本。此外，还需要对你的应用程序进行测试，保证升级没有产生负面影响。
这只是一个大致的流程，实际的升级操作可能会更加复杂，因为可能会涉及到集群内的不同程序以及API对象的升级。在操作之前建议每一步都要计划充分并再三测试以确保环境的稳定。 
~~~

