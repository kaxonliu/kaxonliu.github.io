# 配置 upstream

`upstream` 模块是 Nginx 实现负载均衡（Load Balancing）的核心模块。它允许你定义一个服务器组（也称为“上游服务器组”），Nginx 可以将收到的客户端请求按照指定的规则（如轮询、权重等）代理到这个组中的服务器。

Nginx 可以实现七层负载均衡，也可以实现四层负载均衡。



## 基本使用

一个最简单的 `upstream` 配置通常包含两个部分：定义 upstream 服务器组、在 `server` 块中引用这个组

~~~nginx
# 在 http 块中定义 upstream 组，名为 'backend_servers'
http {
    upstream backend_servers {
        server 192.168.1.10:8080;
        server 192.168.1.11:8080;
        server 192.168.1.12:8080;
    }

    # 在 server 块中引用 upstream 组
    server {
        listen 80;
        server_name example.com;

        location / {
            # 将所有匹配此 location 的请求代理到 'backend_servers' 组
            proxy_pass http://backend_servers;
            
            # 以下是一些常用的代理设置（可选但重要）
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_set_header X-Forwarded-Proto $scheme;
        }
    }
}
~~~



## 服务器指令参数

在 `upstream` 的 `server` 指令中，可以配置丰富的参数来精细控制服务器状态：

~~~nginx
upstream backend {
    server 192.168.1.10:8080 weight=5 max_fails=3 fail_timeout=30s;
    server 192.168.1.11:8080 weight=1 down;
    server 192.168.1.11:8080 weight=2 backup;
    
    # 保持连接（Nginx Plus 或 开源版需特定版本以上）
    keepalive 32; 
}
~~~

**参数解释**

| 参数          | 解释                                                         |
| ------------- | ------------------------------------------------------------ |
| weight=       | 设置服务器权重，默认=1                                       |
| max_fails=    | 在 `fail_timeout` 时间内，最大失败次数。超过这个次数，Nginx 会认为服务器不可用。 |
| fail_timeout= | 与 `max_fails` 结合使用，定义判断服务器不可用的时间窗口。服务器被标记为不可用后，等待多久后再次尝试连接它。 |
| down          | 手动将服务器标记为永久下线，该节点不再参与负载均衡服务。     |
| backup        | 该服务器标记为**备份服务器**。只有当所有非备份服务器都不可用时，请求才会被转发到备份服务器。 |
| slow_start    | (Nginx Plus) 当一台恢复的服务器重新加入负载均衡池时，让其权重从 0 慢慢恢复到设定值，避免刚启动就被大量请求冲垮。慢启动。 |



### 慢启动方案

付费版才能使用慢启动参数 。开源版本可以降低轮训权重的方式减少启动时的流量压力。

~~~nginx
upstream backend {
    server backend1.example.com slow_start=30s;
    server backend2.example.com;
    server backend3.example.com;
}
~~~



### 健康检查

Nginx 开源版默认的被动健康检查是通过 `max_fails` 和 `fail_timeout` 实现的。如果代理请求失败（如连接超时、返回 5xx 错误），Nginx 会记录一次失败。当失败次数达到 `max_fails` 时，在 `fail_timeout` 时间内不再向这台服务器发送请求。

使用 `proxy_next_upstream` 遇到指定报错信息，第二次再出现这个报错就会把请求交给下一个节点。它确保了即使某台后端服务器出现临时故障（如网络波动、进程繁忙、应用错误等），请求仍然有机会被其他健康的服务器处理，而不是直接向客户端返回错误。

~~~nginx
upstream blog {
    server 172.16.1.7;
    server 172.16.1.8;
    server 172.16.1.9;
}
 
server {
    listen 80;
    location / {
        proxy_pass http://blog;
    
        # 在错误、超时或遇到特定HTTP状态码时重试
        proxy_next_upstream error timeout http_500 http_502 http_503 http_504 http_403 http_404;
    
        # 最多尝试3次（即最多重试2次）
        proxy_next_upstream_tries 3;
        
        # 所有重试必须在10秒内完成
        proxy_next_upstream_timeout 10s;
    }
}
~~~

**重试流程**：

假设有三台服务器：A, B, C。配置如下：

~~~nginx
proxy_next_upstream error timeout http_500;
proxy_next_upstream_tries 3;
~~~

1. 请求首先发给A，A 返回 500，进行第一次重试，把请求发往服务器 B；
2. 如果服务器 B 返回成功则请求流程结束。
3. 如果服务器 B 也返回 500，则进行第二次重试，继续重试把请求发往服务器 C。
4. 因为总尝试次数为3，因此服务器 C 是最后一次尝试处理请求。





## 负载均衡算法

#### 默认轮询 (Round Robin)

请求按时间顺序逐一分配到不同的后端服务器。这是默认的方法，无需额外配置。如果后端服务器 down 了，可以自动剔除。

~~~nginx
upstream backend {
    server backend1.example.com;
    server backend2.example.com; # 默认权重都是 1
}
~~~

#### 加权轮询 (Weighted Round Robin)

使用 `weight` 参数指定服务器的权重。权重越高，被分配到的请求越多。适用于服务器性能不均的场景。

~~~nginx
upstream backend {
    server backend1.example.com weight=3; # 处理 3/5 的请求
    server backend2.example.com weight=2; # 处理 2/5 的请求
}
~~~

#### IP 哈希 (IP Hash)

根据客户端 IP 地址的哈希结果分配请求。这样可以保证来自同一个 IP 的请求总是被分配到同一台后端服务器（会话保持）。

~~~nginx
upstream backend {
    ip_hash; # 启用 IP Hash 算法
    server backend1.example.com;
    server backend2.example.com;
}
~~~

#### 最少连接 (Least Connections)

将请求优先分配给当前连接数最少的后端服务器，以避免某台服务器过载。但是要注意，当服务器处理请求的速度差异很大时，最快的服务器可能会收到更多的请求，这可能会导致最快的服务器过载。这种情况下，应该根据具体业务需求和服务器性能，使用轮训或权重的负载策略。

~~~nginx
upstream backend {
    least_conn; # 启用最少连接算法
    server backend1.example.com;
    server backend2.example.com;
}
~~~

#### URL 哈希(URL Hash)

按照访问 URL 的哈希结果分配请求，相同的 URL 分配到同一个后端服务器。想要使用需要安装并启用 `nginx-sticky-module` 模块。

~~~nginx
upstream backend {
    sticky path;
    server backend1.example.com;
    server backend2.example.com;
}
~~~

#### 公平调度算法（fair）

该算法可以根据页面大小、加载时间长短智能的进行负载均衡。 Nginx 本身不支持 fair，如果需要这种调度算法，这需要安装并启用 `nginx_upstream_fair` 模块。

~~~nginx
upstream backend {
    fair;
    server backend1.example.com;
    server backend2.example.com;
}
~~~

