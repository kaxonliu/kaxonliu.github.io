# 重写 URI

重写指的是把 URL 中的 URI 路径改写、或者重定向为一个新的URI路径、或者改为另一个完整 URL 地址。重写功能依赖于 nginx 的 `ngx_http_rewrite_module` 模块。参考：[官方文档](https://nginx.org/en/docs/http/ngx_http_rewrite_module.html)。

重写功能配置时可以使用四个指令，分别是：

- `break`，用于终止当前级别的重写行为，不再往下查询。
- `if` ，用于逻辑判断。
- `return`，结束重写逻辑，直接返回客户端。
- `rewrite`，定义 URI 重写逻辑。

这些指令可以配置在 `server` 块、`location` 块、`if` 块内。



## 重写的使用场景

- 实现 URL 伪静态，方便被搜索引擎收录/SEO 优化。
- 网站换新域名，把旧域名重定向到新域名。
- http 协议的URL 重定向到 https 协议。
- 根据特殊客户端信息跳转指定 URL。





## 重写的执行逻辑

收到客户端请求 URI，重写模块会去配置文件中找到个重写相关的指令，然后按照如下顺序执行重写逻辑。
1. 第一阶段：首先执行 `server` 块内的重写指令。可能成功重写也可以没有重写。
    - 如果没有满足重写，则进入下一阶段。
    - 如果重写成功，把原 URI 重写为新 URI，则进入下一个阶段。
    - 如果把原 URI 重写为一个带有域名的完整 URL，则直接跳转到新 URL 地址。
2. 第二阶段：进入重写循环模式，循环匹配 `location` 块并执行块内重写指令。可能成功匹配也可能匹配失败。
    - 如果所有的 `location` 块匹配失败，则退出重写循环模式。直接使用 `main` 或者 `server` 块内的 `root` 指定的根目录查找资源路径。
    - 如果成功匹配到一个 `location` 块，则执行块内重写相关指令。按照自上而下的顺序依次执行。执行过程中可能遇到 `last` 指令、`break` 指令、 `return` 指令、`rewrite` 指令、`if` 块。
        - 遇到 `last` 指令，立即进入下一轮，使用最新的 URI 匹配 `location`
        - 遇到 `break` 指令，立即停止重写循环模式。在当前 `location` 块内执行非重写相关指令。
        - 遇到 `return` 指令，结束重写，直接返回（返回状态或 跳转到 URL 地址）。
        - 遇到 `if` 块，判断是否满足条件，如果满足条件则进入 `if` 块内执行重写相关指令。
        - 遇到 `rewrite` 指令，判断是否可以重写。如果满足符合重写规则，有两种情况。
            - 重写为新 URI，则进入下一轮 `location` 匹配。
            - 重写为一个带有域名的完整 URL，则停止重写跳转到新 URL 地址。

注意：重写循环不会超过 10 次。





## rewrite 语法

~~~nginx
rewrite regex replacement [flag];
 
regex           #用于匹配uri路径的正则表达式
replacement     #替换的新路径
[flag];         #标记位
~~~

解析如下：

- `rewrite` 只能放在 `server` 块、`location` 块、 `if`  块内部。
- regex正则部分只匹配 URI 路径部分。
- replacement可以是不带域名的uri路径（默认当前域名），也可以是一个带着新域名的完整url地址。
    - 如果是 URI，那么会进行下一轮的匹配和重写逻辑。
    - 如果是完整的带有域名的 URL，则直接跳转到这个新地址。
- flag 的值有四个。分别是 `last`、`break`、`redirect`、`permanent`。
    - last: 相当于continue指令，会结束当前location内的重写指令，然后进入下一次loop。拿着改写后的新uri，从头开始匹配location，从第一次之后，即从第二次算起，最多不能超过10次loop。即不在rewrite指令所在的当前级别查找新的uri路径，而是从头开始在server块中查找location。
    - break: break掉重写模块的loop，所有重写模块相关的指令整体结束掉无论你在何处，但肯定不会影响其他模块，所以结论就是会在本location内继续执行其他非重写模块相关指令。
    - redirect:  返回302临时重定向，浏览器地址会显示跳转后的URL地址。
    - permanent:  返回301永久重定向，浏览器地址会显示跳转后URL地址。



## 示例

~~~nginx
server {
    listen       8080;
    root /var/www/html/;
    rewrite_log on;
    error_log /var/log/nginx/error.log notice;

    location /test1 {  # 第一个location
        rewrite .* /test2;
    }
    location /test2 { # 第二个location
        #rewrite .* /test3 last;
        #rewrite .* /test3 break;
        rewrite .* /test666;
        rewrite .* /test777 ;
        rewrite .* https://kaxonliu.github.io redirect;

        root /var/nginx/html;
        #default_type text/html;
        #return 200 "hello world";
    }
    location /test3 { # 第三个location
        return 301 https://baidu.com;
    }
}
~~~





## 开启 rewrite 日志

~~~nginx
server {
    listen 80;
    server_name domain.com;
 
    rewrite_log on;
}
~~~

还需要设置 error_log 指令，并且把日志登记设置为 notice 或者更高级别。

~~~nginx
error_log  /var/log/nginx/error.log notice;
~~~



## rewrite 实现URL 伪静态

前端请求用的是静态地址 `/product-123.html`，后端收到的依然是 ` /product.php?id=123`

~~~nginx
server {
    listen  80;
    server_name  www.example.com;
    location / {
        rewrite ^/product-([0-9]+)\.html$  /product.php?id=$1  last;
    }
}
~~~



## 重写相关指令 set

set 用来定义变量。

~~~nginx
server {
    listen 8080;
    set $page "books/architecture/nginx-conf/server/index.html";
    location / {
        if ($http_user_agent ~* chrome) {
            set $page "books/architecture/nginx-conf/server/index.html";
            rewrite .* https://kaxonliu.github.io/$page;
        }
    }
}
~~~



