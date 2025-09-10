# 部署 java

java web 服务部署分两类：部署 jar 包、部署 war 包。这两个都是 java 程序打包的结果。其中，war 包只包含业务逻辑，部署时需要在前面加上 web 服务器 tomcat。jar 包内置了服务器 tomcat ，因此可以直接运行。但一般都会在最前面加上 nginx。



## 部署 jar 包

#### 1. 准备环境

~~~bash
关防火墙
关 selinux
设置静态ip
同步时间
~~~

#### 2. 准备打包环境

**安装 jdk** 版本号是1.8

~~~bash
# 安装
yum -y install java-1.8.0-openjdk* 
 
# 2、验证安装
[root@rocky ~]# java -version
openjdk version "1.8.0_462"
OpenJDK Runtime Environment (build 1.8.0_462-b08)
OpenJDK 64-Bit Server VM (build 25.462-b08, mixed mode)

# 配置环境变量
find /usr/lib/jvm -name 'java-1.8.0-openjdk-1.8.0*'

# 把 find 的结果赋值给变量JAVA_HOME，配置在 /etc/profile
JAVA_HOME=/usr/lib/jvm/java-1.8.0-openjdk-1.8.0.462.b08-3.el9.aarch64
PATH=$PATH:$JAVA_HOME/bin
CLASSPATH=.:$JAVA_HOME/lib/dt.jar:$JAVA_HOME/lib/tools.jar
export JAVA_HOME CLASSPATH PATH

# 加载配置
source /etc/profile
~~~

**安装 maven**

~~~bash
# 官网下载:https://maven.apache.org/download.cgi
wget https://dlcdn.apache.org/maven/maven-3/3.9.11/binaries/apache-maven-3.9.11-bin.tar.gz

# 解压
tar xf apache-maven-3.9.11-bin.tar.gz -C /usr/local
 
# 添加环境变量
vim /etc/profile 
PATH=/usr/local/apache-maven-3.9.11/bin:$PATH
export PATH

# 加载配置
source /etc/profile
~~~

#### 3. 准备数据库

#### 4. 准备项目代码

需要配置项目对外暴露的端口，以及依赖第三方应用的配置。



#### 5. 导出 jar 包

~~~bash
cd /your/java/project/workdir
mvn clean package -Dmaven.test.skip=true
~~~

#### 6. 启动 jar 包

~~~bash
java -jar /path/to/jar-path/*.jar
~~~

#### 7. 配置 nginx

jar 包运行后，可以直接使用浏览器访问监听的 ip + port。但推荐在前面加一个 nginx

~~~nginx
server {
    listen        80; 
    server_name   localhost;
 
    location / {
        rewrite ^(.*)$ /web1$1;
    }
    location /web1 { 
        proxy_pass http://127.0.0.1:8080;
        proxy_set_header Host $http_host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Scheme $scheme;
        proxy_redirect off;
    }
}
~~~



## 部署 war 包

#### 1. 导出 war 包

导出 war 包的准备工作和 jar 包类似，需要做一些配置让idea 导出 war 包。拿到 war 包开始部署。tomcat 和 war 包之间不走协议。 



#### 2. 安装 web 服务器 tomcat

tomcate [官网下载安装包](https://tomcat.apache.org/)。 注意：jdk1.8 配合使用 tomcat9

~~~bash
# 下载
wget https://dlcdn.apache.org/tomcat/tomcat-9/v9.0.109/bin/apache-tomcat-9.0.109.tar.gz

# 解压
tar -zxvf apache-tomcat-9.0.109.tar.gz

# 移动
mv apache-tomcat-9.0.109 /usr/local/tomcat/

# 授权
useradd www
chown -R www.www /usr/local/tomcat/
~~~

#### 3. 启动 tomcat

~~~bash
[root@rocky ~]# /usr/local/tomcat/bin/catalina.sh start
Using CATALINA_BASE:   /usr/local/tomcat
Using CATALINA_HOME:   /usr/local/tomcat
Using CATALINA_TMPDIR: /usr/local/tomcat/temp
Using JRE_HOME:        /usr/lib/jvm/java-1.8.0-openjdk-1.8.0.462.b08-3.el9.aarch64
Using CLASSPATH:       /usr/local/tomcat/bin/bootstrap.jar:/usr/local/tomcat/bin/tomcat-juli.jar
Using CATALINA_OPTS:   
Tomcat started.
~~~

#### 4. 测试

浏览器访问 `http://<ip>:8080`。注意：可能要等一会才可以访问，看监听端口。

~~~bash
[root@rocky ~]# netstat -tunlpa | grep -w 8080
tcp6       0      0 :::8080                 :::*                    LISTEN      25909/java          
tcp6       0      0 10.10.97.210:8080       10.10.99.168:64252      TIME_WAIT   -                   
tcp6       0      0 10.10.97.210:8080       10.10.99.168:64253      TIME_WAIT   -                   
~~~

**修改 tomcat 监听的端**

~~~bash
vim /usr/local/tomcat/conf/server.xml 
<Connector port="18080" protocol="HTTP/1.1"

# 重启 tomcat
/usr/local/tomcat/bin/catalina.sh stop
/usr/local/tomcat/bin/catalina.sh start

# 查看端口
netstat -tunlap | grep -w 18080
tcp6       0      0 :::18080                :::*                    LISTEN      26337/java          
tcp6       0      0 10.10.97.210:18080      10.10.99.168:64924      ESTABLISHED 26337/java          
tcp6       0      0 10.10.97.210:18080      10.10.99.168:64923      ESTABLISHED 26337/java          
~~~



#### 5. 把 war 包放在 tomcat 管理的文件夹下面

默认配置是 `webapps`。放到这个文件夹下面 tomcat 会自动解压缩 war 包。

~~~bash
[root@rocky webapps]# pwd
/usr/local/tomcat/webapps
~~~



#### 6. 配置nginx 代理 tomcat

tomcat 运行后，可以直接使用浏览器访问监听的 ip + port。但推荐在前面加一个 nginx。

~~~nginx
server {
    listen        80; 
    server_name   localhost;
 
    location / {
        rewrite ^(.*)$ /web1$1;
    }
    location /web1 { 
        proxy_pass http://127.0.0.1:8080;
        proxy_set_header Host $http_host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Scheme $scheme;
        proxy_redirect off;
    }
}
~~~



## Tomcat 配置

~~~xml
<server> 
    <service> 
        <connector PORT /> 
        <engine>
            <host name=www.a.com appBase=/www/a >
                <context path="" docBase=/www/a /> 
                <context path="/xuexi" docBase=/www/a/xuexi />
            </host>
 
            <host>
                <context />
            </host>
        </engine>
    </service>
</server>
~~~

解释：

- `<server>` 中可以监听一个端口，从此端口上可以远程向该实例发送 shutdown 关闭命令。
- `<service>` 是一个逻辑组件，用于绑定 connector 和 container，有一个 service 就是一个服务。
- `<connector>`  相当于 nginx 中的 listen， 此处监听的端口才是对外暴露服务的端口，connector 称之为连接器，它监听到的请求会交给engine具体处理处理。
- `<engine>`  接请求，按照分析的结果将相关参数传递给匹配出的虚拟主机。
- `<host>`  相当于 nginx 中的 server_name
- `<context> ` 相当于 nginx 中 的localtion



