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

1. 首先读取配置在 `server` 块内的重写指令，如果有则尝试匹配重写 URI。
2. 然后拿着最新的 URI 按照 `location` 优先级依次匹配 。如果匹配成功某个 `location` 块，则进入该 `location` 块内部，按照自上而下的顺序运行和重写有关的指令。在这个 `location` 块内，URI 可能被重写一次或多次，也可能没有被重写。然后进入下一轮匹配 `location` 块并尝试重写的过程，直到遇到了 `break` 或 `return` 指令，或者循环了10次则停止循环。



**注意：一定要区分 什么是重写配置和重写模块代码运行逻辑**



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



