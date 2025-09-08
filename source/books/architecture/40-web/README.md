# WEB部署协议

一个完整的 web 服务系统由两部分组成，分别是 web服务 和 web 应用。

- **Web 服务**。负责处理网络相关的事情。
- **Web应用**。负责实现服务具体业务逻辑。



在互联网早期，一般都是静态页面，这个时候没有所谓的 web 服务和 web 应用，网络相关和业务相关的事情全部在一个 web 程序中实现，浏览器和 web 程序之间直接按照 HTTP 协议实现通信。此时的开发效率很低。



## cgi 协议

CSI（Common Gateway Interface）就是通用网关协议，它的出现是为了解决 web 服务和 web 应用解耦问题。这个协议规定了 web 服务器和 web 应用的负责的事情。规定如下：

- Web 服务器应该接收并处理 http 协议，把请求头和请求体数据处理成 cgi 协议规定的数据格式传给 web 应用。来一个请求就创建一个进程用来执行 cgi 程序。
- Web 应用接收 cgi 协议规定的数据，响应请求，然后按照协议格式返回响应结果。

#### 优点

- 和语言无关
- 简单易懂
- 进程隔离。CGI 程序在独立进程中运行，即使它崩溃了也不会影响 Web 服务器本身。

#### 缺点

- **性能极差**：这是最大的问题。**每个请求都会创建一个新的操作系统进程**。进程的创建、销毁和上下文切换是非常消耗资源和时间的（“高并发fork”问题）。如果访问量很大，服务器会瞬间被压垮。
- **资源无法共享**：由于每个请求都是独立的进程，程序之间无法共享数据库连接等资源，每次都需要重新建立连接，进一步降低了效率。



## fastcgi 协议

**FastCGI（Fast Common Gateway Interface）** 是在 cgi 基础之上定义的新**协议**。它的核心目标是**克服传统 CGI 协议“每个请求创建一个新进程”** 所带来的性能开销，从而能够高效地处理高并发请求。FastCGI 通过引入**持久化应用进程**的概念完美地解决了这些问题。



#### 核心工作原理

- **启动管理器**。FastCGI 程序首先会启动一个**主进程**。这个主进程负责读取配置、绑定网络端口（是的，FastCGI 通常使用**网络 Socket** 而不是标准输入输出来通信）、启动和管理一系列**工作进程（Worker Processes）**。
- **预派生工作进程**。主进程会根据配置，预先创建（fork）好一批子进程，形成一个“进程池”。这些工作进程在启动后就会初始化（例如，预先连接好数据库），然后进入空闲状态，等待任务的到来。这些进程是**持久化**的，会处理完一个请求后，不会退出，而是继续等待下一个请求。
- **Web 服务器与 FastCGI 进程通信**。Web 服务器（如 Nginx）不再通过 fork 来执行程序，而是通过 **Unix Domain Socket** 或 **TCP/IP Socket** 与 FastCGI 进程池建立连接。当一个 HTTP 请求到来时，Web 服务器将 CGI 环境变量和请求数据通过这个 Socket 连接转发给 FastCGI 进程池。
- **处理请求**。FastCGI 主进程将请求分配给一个空闲的工作进程。该工作进程处理请求（执行 PHP、Python 等代码），然后将处理结果通过同一个 Socket 连接返回给 Web 服务器。处理完成后，该工作进程**不被销毁**，而是释放资源（如关闭不必要的文件描述符，但可以保持数据库连接）并重新回到空闲池中，等待下一个任务。
- **返回响应**。Web 服务器接收到 FastCGI 工作进程返回的结果后，将其包装成 HTTP 响应发送给客户端。



**PHP-FPM（PHP FastCGI Process Manager）** 是 FastCGI 协议最著名、最成功的实现之一。它专门用于管理 PHP 的 FastCGI 进程。

php-fpm 不支持 http 协议，所以要在它的前面使用 Nginx 或 Apahe等 Web 服务器做 fastcgi 协议的服务部分，php-fpm做为 fastcgi 协议的应用部分。

Nginx 做 web 服务器，无法直接使用 FastCGI，需要启用模块 `ngx_http_fastcgi_module` 进行代理配置。





## wsgi 协议

WSGI (Web Server Gateway Interface)，web 服务器网关接口，它是专门为 Python 程序专门定制的，**用在 web 服务器和 web 应用之间通信的一种协议**，是 Python 的 CGI 精神继承者。

WSGI 的诞生解决了两个问题：

- **互通性**：让任何符合 WSGI 规范的 Web 服务器都能运行任何符合 WSGI 规范的 Web 应用程序。开发者可以自由组合服务器和框架。
- **分层设计**：在 Web 服务器和应用程序之间引入中间件（Middleware）的概念，允许像洋葱一样一层层地处理请求和响应，实现身份验证、日志、路由等功能。



#### 核心工作原理

WSGI 规范极其简单，它只定义了两样东西：

- **应用程序端（Application）**：一个可调用对象（callable），通常是函数或一个实现了 `__call__` 方法的对象。它接收两个参数：`environ`  和 `start_response`。
- **服务器端（Server/Gateway）**：负责调用应用程序端的可调用对象，传递 `environ` 和 `start_response`，并处理其返回的响应体，最终返回给客户端。

~~~python
def simple_app(environ, start_response):
    # 1. 设置 HTTP 状态码和响应头
    status = '200 OK'
    response_headers = [('Content-type', 'text/plain; charset=utf-8')]
    start_response(status, response_headers)

    # 2. 返回响应体（必须是可迭代的，包含字节串）
    return [b"Hello, WSGI World!\n"]

# 应用程序对象 `simple_app` 可以被任何 WSGI 服务器调用
~~~



#### 常见的 WSGI 服务器
- **Gunicorn** (Green Unicorn)：纯 Python 编写，简单易用，是很多项目的首选。
- **uWSGI**：一个功能极其丰富的全能型应用服务器（C 编写），性能强大。
- **mod_wsgi**：Apache 的一个模块，让 Apache 可以直接托管 WSGI 应用。
- **waitress**：一个纯 Python 编写的、性能不错的服务器。





## uwsgi 协议

uwsgi 协议是一个**二进制的线路协议（Wire Protocol）**。它是由 uWSGI 项目自定义的，用于 **uWSGI 服务器和上游服务器（如 Nginx） 之间进行高效通信**。

在 uWSGI 服务器出现之初，它通过 HTTP 与 Nginx 通信。但 HTTP 是一种为人类可读、通用 Web 设计的**文本协议**，其头部（Headers）解析和处理相对繁琐，对于本地或内网中两个高性能服务之间的通信来说，显得过于“沉重”和低效。为了解决这个问题，uWSGI 规定了 uwsgi 协议。

uwsgi 协议的设计目标非常明确：

1. **极高性能**：采用**二进制格式**，解析速度远超文本协议（如 HTTP、FastCGI）。
2. **极低开销**：设计极其精简，报文结构紧凑，最大限度地减少网络传输和解析所需的 CPU 和内存资源。
3. **高可用性**：支持请求流水线（Pipelining）、路由、订阅等多种高级功能。



### uWSGI

**uWSGI** 是一个**全功能的、高性能的 C 语言编写的应用服务器容器**。它的目标是提供完整的堆栈，用于构建托管服务。**关键点：**

- 它是一个**具体的软件项目**，一个**程序**，你可以 `pip install uwsgi` 来安装它。
- 它的名字里带“WSGI”，是因为它**原生支持并完美实现了 WSGI 协议**。但它**远不止于此**。

uWSGI 非常强大，甚至可以称之为“瑞士军刀”：

1. **协议支持广泛**：它不仅是 WSGI 服务器，还支持其他协议，如 **uWSGI 协议**（它的自有协议，效率更高）、PHP 的 PSGI、Perl 的 PSGI、Ruby 的 Rack 等。
2. **进程管理**：像 FastCGI 一样，它使用主进程管理 worker 进程池，实现高并发。
3. **负载均衡与监控**：内置了负载均衡器和强大的管理统计接口。
4. **静态文件服务**：可以直接提供静态文件，虽然生产环境通常还是交给 Nginx。
5. **热重载**：可以在不中断服务的情况下重新加载应用代码。

虽然 uWSGI 服务器可以通过 WSGI 协议直接与 Python 应用通信，但当它和 Nginx 交互时，使用自有的 **uwsgi 协议**（而不是 HTTP 或 FastCGI）可以获得更高的效率和更低的开销。





## asgi 协议









