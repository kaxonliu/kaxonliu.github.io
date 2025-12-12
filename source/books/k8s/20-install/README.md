# 安装 k8s

使用 kubeadm 部署 k8s， 集群主机为 centos7.9， x86_64



## 阶段1：准备工作

#### 0. 主机规划

准备三台主机，一个 master 节点、两个 node 节点。每个主机的内存至少要 2G。

- master01 192.168.10.81
- node01 192.168.10.82
- node02 192.168.10.83



#### 1. 修改主机名和解析

~~~bash
# 1、修改主机名
hostnamectl set-hostname k8s-master-01
hostnamectl set-hostname k8s-node-01
hostnamectl set-hostname k8s-node-02

# 2、三台机器添加host解析
cat >> /etc/hosts << "EOF"
192.168.10.81 k8s-master-01
192.168.10.82 k8s-node-01
192.168.10.83 k8s-node-02
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
ssh-copy-id -i root@k8s-node-01
ssh-copy-id -i root@k8s-node-02
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

maste 节点和外网同步时间，node 节点向 master 节点同步时间。

##### master 节点

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
~~~



##### node 节点

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

三个节点都要安装 containerd。

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



## 阶段3：安装 k8s

#### 1. 三台机器准备 k8s 源

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



#### 2. master 节点操作

在 k8s master节点上执行的初始化操作，node 节点不需要操作。

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
  advertiseAddress: 192.168.10.81
  bindPort: 6443
nodeRegistration:
  criSocket: unix:///var/run/containerd/containerd.sock
  imagePullPolicy: IfNotPresent
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
imageRepository: registry.cn-hangzhou.aliyuncs.com/google_containers
kind: ClusterConfiguration
kubernetesVersion: 1.30.0
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

- 修改 `advertiseAddress: 192.168.10.81` 为主节点的 IP
- 修改 `imageRepository: registry.cn-hangzhou.aliyuncs.com/google_containers`  换成阿里云镜像仓库地址



部署

~~~bash
kubeadm init --config=kubeadm.yaml --ignore-preflight-errors=SystemVerification --ignore-preflight-errors=Swap
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
....
~~~



查看 node

~~~bash
[root@k8s-master-01 ~]# kubectl get nodes
NAME            STATUS     ROLES           AGE     VERSION
k8s-master-01   NotReady   control-plane   3m14s   v1.30.14
[root@k8s-master-01 ~]# kubectl -n kube-system get pods
NAME                                    READY   STATUS    RESTARTS   AGE
coredns-6d58d46f65-gcnb5                0/1     Pending   0          3m17s
coredns-6d58d46f65-n8vg8                0/1     Pending   0          3m17s
etcd-k8s-master-01                      1/1     Running   0          3m34s
kube-apiserver-k8s-master-01            1/1     Running   0          3m34s
kube-controller-manager-k8s-master-01   1/1     Running   0          3m34s
kube-proxy-vjm5t                        1/1     Running   0          3m18s
kube-scheduler-k8s-master-01            1/1     Running   0          3m34s
~~~



#### 3. node 节点加入集群

到两外两个 node 执行

~~~bash
kubeadm join 192.168.10.81:6443 --token abcdef.0123456789abcdef \
	--discovery-token-ca-cert-hash sha256:697b98effa0859738a0e5a20944d1366ef296b31c7214354871028bbeb44b5c8
~~~

加入后，可以看到如下输出信息

~~~bash
This node has joined the cluster:
* Certificate signing request was sent to apiserver and a response was received.
* The Kubelet was informed of the new secure connection details.

Run 'kubectl get nodes' on the control-plane to see this node join the cluster.
...
~~~

master 节点此时查看新的 node 信息

~~~bash
[root@k8s-master-01 ~]# kubectl get nodes
NAME            STATUS     ROLES           AGE     VERSION
k8s-master-01   NotReady   control-plane   6m18s   v1.30.14
k8s-node-01     NotReady   <none>          58s     v1.30.14
k8s-node-02     NotReady   <none>          48s     v1.30.14
~~~



#### 4. 部署网络插件

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



#### 5. 安装 kubectl 命令提示

所有节点安装，node 节点的家目录可能没有 `./kube` 目录，没有的话手动创建。

~~~bash
yum install bash-completion* -y
 
kubectl completion bash > ~/.kube/completion.bash.inc
echo "source '$HOME/.kube/completion.bash.inc'" >> $HOME/.bash_profile
source $HOME/.bash_profile
~~~

