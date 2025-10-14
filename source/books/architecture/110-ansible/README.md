# ansible

Ansible 是一款开源的 **IT 自动化工具**。它用于配置管理、应用部署、任务自动化和 IT 编排。

简单来说，Ansible 可以让你用简单的语言（YAML）来描述你的自动化任务，然后它帮你在一台或多台远程服务器上执行这些任务。



## ansible 的特点

- **无代理**（Agentless）：它不需要在目标服务器上安装任何客户端代理。它通过 SSH 协议来管理机器。
- **无服务**（Serverless）：在控制端机器上不用启动任何服务，只需要执行命令即可。
- **简单易读**：使用 YAML （具体形式为 Playbook）来描述配置和流程，语法非常直观，即使不是程序员也能看懂。
- **幂等性**：意味着同一个 Playbook 你可以安全地执行多次，只有在需要改变时才会对系统进行操作。如果目标系统已经处于期望的状态，Ansible 将不会做任何改变。
- **模块化**：Ansible 的所有功能都通过模块实现。有成千上万的社区和维护的模块，可以用于管理软件包、文件、服务、云资源（AWS, Azure, GCP）、容器等等。
- **Python语言开发**，方便二次开发。



## ansible 的组成

~~~bash
1.连接插件connection plugins：用于连接主机/被管理端
2.核心模块core modules      ：总控， 它负责调用具体的模块来做具体的事情
3.自定义模块custom modules  ：根据自己的需求编写具体的模块
4.插件plugins               ：完成模块功能的补充，例如Email、loggin
5.任务副本/剧本Play Books   ：剧本playbookansible的配置文件,将多个任务定义在剧本中，由ansible自动执行
6.主机清单Host inventor     ：定义ansible需要操作的主机信息
 
强调：
1、ansible是模块化的 它所有的操作都依赖于模块
2、模块与插件的区别：
（1）模块会被上传到被控制主机上，在被控制主机上执行
（2）插件在控制主机上，被 Ansible 调度执行
3、尽量使用 Ansible 自带的模块，而不是 shell 脚本，因为 Ansible 的很多模块提供幂等判断机制
~~~

![](D:\kaxonliu.github.io\source\books\architecture\110-ansible\ansible_core.png)

##  ansible 执行流程

1. **解析清单**
   - Ansible 读取你的 `inventory` 文件，确定任务需要在哪些目标主机上执行。
   - 它根据你提供的主机模式（如 `all`, `web_servers`）来筛选出具体的主机列表。
2. **生成 Python 脚本**
   - Ansible 的核心模块是用 Python 编写的。
   - 对于每个任务，Ansible 会在**控制节点**上动态生成一个包含以下内容的 Python 脚本：
     - 要执行的模块代码（例如 `yum`, `copy`, `command` 模块的代码）。
     - 传递给模块的参数（例如 `name=nginx`, `state=present`）。
   - **注意**：自从 Ansible 2.0 以后，它也支持 `PowerShell` 脚本用于 Windows 受管节点。
3. **将脚本传输到受管节点**
   - Ansible 使用 SSH（Linux）或 WinRM（Windows）连接到目标受管节点。
   - 它将生成的 Python/PowerShell 脚本通过 SFTP 或 SCP 复制到目标节点的一个临时目录下（通常是 `~/.ansible/tmp/`）。
4. **在受管节点上执行脚本**
   - Ansible 通过 SSH 会话在目标节点上执行这个临时脚本。
   - 这是实际工作发生的地方：安装软件包、修改文件、启动服务等。
5. **脚本执行并返回结果**
   - 脚本执行其逻辑。由于 Ansible 模块被设计为**幂等**，它会首先检查系统的当前状态。
   - 脚本会收集执行结果，并将其格式化为一个 JSON 字典。
   - 这个 JSON 结果包含了任务是否成功 (`"changed": true/false`)、是否做出了更改、模块输出 (`"msg"`)、以及任何错误信息。
6. **清理临时文件并断开连接**
   - 脚本执行完毕后，Ansible 会清理受管节点上的临时脚本文件。
   - 然后关闭 SSH 连接。
7. **在控制节点上处理结果**
   - 控制节点接收并解析从受管节点返回的 JSON 结果。
   - 根据结果，Ansible 会在你的终端上输出相应信息（OK, CHANGED, FAILED），并决定 Playbook 的后续流程（例如，是否触发处理程序，是否因错误而中止）。



## 安装 ansible

#### 下载 epel 源

~~~bash
wget -O /etc/yum.repos.d/epel.repo http://mirrors.aliyun.com/repo/epel-7.repo
~~~

#### 安装 ansible

~~~bash
yum install -y ansible
~~~



#### ansible 命令

~~~bash
 # 程序/命令
/usr/bin/ansible          `主程序，临时命令执行工具，ansible --version查看版本，man ansible查看帮助`
/usr/bin/ansible-doc      `查看配置文档，模块功能查看工具`
/usr/bin/ansible-galaxy   `下载/上传优秀代码或Roles模块的官网平台`
/usr/bin/ansible-playbook `定制自动化任务，编排剧本工具`
/usr/bin/ansible-pull     `远程执行命令的工具`
/usr/bin/ansible-vault    `文件加密工具`
/usr/bin/ansible-console  `基于Consol`
~~~

#### ansible 配置文件

~~~bash
# 配置文件
/etc/ansible/ansible.cfg `主配置文件，配置ansible工作特性`
/etc/ansible/hosts `主机清单`
/etc/ansible/roles `存放角色的目录`
~~~



## ansible 配置文件

ansible 没有守护进程，每次使用 ansible 命令都会去配置文件中找配置信息。

配置文件可以有多个，并且有优先级之分。一般就使用最后一个，这也是默认的配置文件。

~~~bash
ANSIBLE_CONFIG 环境变量
~/.ansible.cfg
/etc/ansible/ansible.cfg
~~~

ansible.cfg 核心配置项

~~~ini
[root@m01 ~]# cat /etc/ansible/ansible.cfg  
[defaults]
#inventory      = /etc/ansible/hosts      #主机列表配置文件
#library        = /usr/share/my_modules/  #库文件存放目录
#remote_tmp     = ~/.ansible/tmp          #临时py文件存放在远程主机目录
#local_tmp      = ~/.ansible/tmp          #本机的临时执行目录
#forks          = 5                       #forks 是非常重要的、优化时需要重点关注的参数。 Ansible 会创建 forks 个子进程，并发地管理多台被控主机， 默认值是 5，随着被管理主机的增多，需要适当调大
#sudo_user      = root                    #默认sudo用户
#ask_sudo_pass = True                     #每次执行是否询问sudo的ssh密码
#ask_pass      = True                     #每次执行是否询问ssh密码
#remote_port    = 22                      #远程主机端口
#timeout        = 60                      # ssh 连接的超时时间，单位是秒
 
host_key_checking = False                 #Ansible 在通过 ssh 连接被控主机时，是否检查其公钥
log_path = /var/log/ansible.log           #ansible日志
 
#普通用户提权操作
[privilege_escalation]
#become=True
#become_method=sudo
#become_user=root
#become_ask_pass=False
~~~



## ansible 主机清单

主机清单（inventory）是配置被控主机的配置文件。可以是静态的，也可以是动态的。

静态主机清单。指的就是 `/etc/ansible/hosts` 文件，记录被管理主机的连接信息。

动态主机清单。一个可执行程序，ansible 创建一个子进程执行该程序，然后从标准输出中获取主机清单。适用于从其他系统（CMDB、LDAP等）动态获取主机清单。

#### 主机清单示例

~~~ini
[web1]      # 一个组里放一台或多台机器
192.168.71.14 ansible_ssh_user=root ansible_ssh_port=22 ansible_ssh_pass='1'
192.168.71.15 ansible_ssh_user=root ansible_ssh_port=22 ansible_ssh_pass='1'
 
[web2]
192.168.71.16 ansible_ssh_pass='1' # 默认22号端口，用户名默认与ansible命令所处用户保持一致
 
[db]
192.168.71.13
[db:vars] # 为db组指定变量，采用密码登录
ansible_ssh_pass='1'
 
[backup]
192.168.71.12
[backup:vars] # 为backup组指定变量,采用密钥登录，前提是你必须对目标主机做好ssh密码登录才行
ansible_ssh_private_key_file=/root/.ssh/id_rsa
 
[www:children]  # 前面的小组都属于某一个业务，那我们可以整合为一个大组命名为www，www:children代表www这个大组下面包含多个小组
web_group1
nfs
db
backup
~~~

如果被控主机做了 ssh 密钥登录，那只配置 IP 即可。

如果在本地添加了主机名解析，那也可以把 IP 地址换成主机名。



配置完成后，可以使用如下命令查看一个组中包含哪些主机。

~~~bash
ansible web1 --list-host
~~~



## ansible 两种任务执行模式

#### ad-hoc 命令行模式

ad-hoc 就是命令行模式，简单便捷、一次性临时操作。

#### playbook 剧本模式

这种模式把把执行任务的逻辑配置在脚本中，可以重复多次使用。



## 命令行模式

命令行模式的语法

~~~bash
ansible [pattern] -m [module] -a "[module options]"
 
1、pattern：用于匹配主机清单，匹配针对的是组名以及组下定义的主机名或地址
2、-m指定模块：模块是ansible提供的功能，例如ping模块用于测试网络连通性，command模块用于执行一次性命令
模块有很多，后续我们会详细介绍，初学，就单以ping模块来快速学习
3、-a是argv的首字母，代表为模块传递的参数，是可选项
~~~

列出模块

~~~bash
#1、列出所有模块
ansible-doc -l
#2、查看指定模块的用法：ansible-doc -s <module_name>
ansible-doc -s ping
ansible-doc -s command
~~~

返回结果颜色含义

~~~bash
绿色: 代表被管理端主机没有被修改
黄色: 代表被管理端主机发现变更
红色: 代表出现了故障，注意查看提示
~~~



## pattern 匹配

ansible 命令的 pattern 用来过滤满足条件的主机清单。

~~~bash
注意两点：
1、匹配的是存在于主机清单的组名或者组下的主机名或ip地址
2、pattern建议统一都用单引号包裹来进行硬引用，防止特殊符号被bash解析而无法正常传递给ansible
 
具体用法如下
    匹配主机组名：
        ansible 'web_group1' -m ping     # 组名
        ansible 'www' -m ping            # 
 
    也可以匹配到组下的某个主机名或ip地址：
        ansible '192.168.71.14' -m ping  # 组下的主机名或ip地址都可以
 
    All ：表示所有Inventory中的所有主机
        ansible 'all' -m ping
 
    * :通配符：必须是/etc/ansible/hosts内定义的
        ansible '*' -m ping  (*表示所有主机)
        ansible 'web*' -m ping
        ansible 192.168.71.* -m ping
 
    正则表达式：~前缀代表启用正则，用的就是python的正则
        ansible '~(db|web).*' -m ping  # ，匹配的是hosts整个文件包括主机组名与主机名
~~~

也支持使用逻辑与、或、非。**注意一定要用单引号包裹**。

~~~bash
# 逻辑或：':'
ansible 'group1:group2' -m ping 
 
ansible 'group1:&group2' -m ping
 
# 逻辑非 ':!'
# 存在于group2但不存在于group1，
ansible 'group2:!group1' -m ping
~~~



## playbook 剧本模式

剧本模式把任务定义在 yaml 文件中。

如下是一个简单的剧本文件，定义了 2 个 play，每个 play 可以有一个多个任务。

~~~yaml
# 剧本文件 a.yaml

# 1、创建用户
- hosts: all
  remote_user: root
 
  vars:
    user: "www"
 
  tasks:
    - name: create  user
      user:
        name: "{{ user }}"
        shell: /sbin/nologin
 
# 2、部署配置httpd
- hosts: web_group1
  remote_user: root
 
  tasks:
    - name: latest httpd version installed
      yum:
        name: httpd
        state: latest
 
    - name: correct index.html is present
      copy:
        src: /tmp/index.html
        dest: /var/www/html/index.html
 
    - name: Directory authorization
      command: "chown -R www.www /var/www/html"
 
    - name: 干掉之前残留的nginx
      command: "pkill -9 ginx"
 
    - name: start httpd service
      systemd:
        name: httpd
        state: started
        enabled: true
~~~



#### 检查语法

~~~bash
ansible-playbook --syntax-check  a.yaml
~~~

#### 预执行命令

~~~bash
ansible-playbook -C a.yaml 
 
# 注意：-C检查结果在启动httpd上会提示failed，
# 这是正常的，因为目标主机上没有真的安装httpd，根本起不来
~~~

#### 执行命令

~~~bash
ansible-playbook a.yaml -vv
~~~

#### 展示详细信息

~~~bash
-v：打印任务运行结果
-vv：打印任务运行结果以及任务的配置信息
-vvv：包含了远程连接的一些信息
-vvvv：Adds extra verbosity options to the connection plug-ins,including the users being used in the managed hosts to execute scripts, and what scripts have been executed
~~~





