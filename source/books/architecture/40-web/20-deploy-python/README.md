# 部署 python

## uwsgi 拉起 web 应用

uwsgi 服务和 web 应用之间使用 wsgi 协议，虽然 uwsgi 服务支持 http 协议，可以直接对外使用，但是建议在 uwsgi 前面加一个 nginx 服务。uwsgi 服务和 nginx 服务之间可以使用高效率的 uwsgi 协议也可以使用普通的 http 协议。

**安装 uwsgi**

~~~bash
pip install uwsgi
~~~



典型的生产环境架构：
**客户端 -> Nginx (反向代理/处理静态文件) -> uWSGI (WSGI服务器) -> Flask App**



**配置 uwsgi.ini**

~~~ini
[uwsgi]
# 使用 uwsgi 协议监听uwsgi的端口
socket = :8080
# 使用 http 协议监听的http协议端口
# http = :8080
# 启动后切换到该目录下作为项目根目录
chdir = /blog/DjangoBlog-master/
# 完整查找路径：应用中 wsgi.py 文件里的application对象
module = djangoblog.wsgi:application
# wsgi-file = djangoblog/wsgi.py  # 与上面的配置等价
# 启动的工作进程数量
processes = 4
# 每个进程开启的线程数量
threads = 2
master=True
pidfile=/tmp/project-master.pid
vacuum=True
max-requests=5000
daemonize=/var/log/uwsgi.log
~~~

**配置 nginx.conf**

~~~nginx
server {
    listen 8111;
    server_name localhost;
  
    location /static {
        alias /path/to/your/project/static;
        expires 30d;
    }
		
    # 使用 uwsgi 协议对接 uwsgi 服务
    location / {
        include uwsgi_params;
        uwsgi_pass 127.0.0.1:8080;
    
        # 透传数据
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
  
    # 使用 http 协议对接 uwsgi 服务
    # location / {
    #    proxy_pass http://127.0.0.1:8080;
    # }
}
~~~

**启动 uwsgi 服务**

~~~bash
# cd 到 uwsgi.ini 所在的目录
uwsgi --ini uwsgi.ini
~~~



## gunicorn 拉起 web 应用

Gunicorn (Green Unicorn) 是一个用纯 Python 编写的 WSGI HTTP 服务器，用于 UNIX 系统。它以其高性能、轻量级和易用性而闻名。gunicorn 服务和 web 应用之间使用 wsgi 协议，gunicorn 服务和 nginx 服务之间 http 协议。

典型的生产环境架构：
**客户端 -> Nginx (反向代理/处理静态文件) -> Gunicorn (WSGI服务器) -> Flask App**



**配置 gunicorn.py**

~~~python
import os
import socket
import traceback

# 服务器绑定的地址和端口
bind = "0.0.0.0:8000"

# The maximum number of requests a worker will process before restarting.
max_requests = 0  # keep running forever

# 工作目录
chdir = ‘/path/to/your/project‘

# The type of workers to use.
worker_class = "gevent"

# 启动的 worker 进程数，通常为 (CPU核心数 * 2) + 1
workers = multiprocessing.cpu_count() * 2 + 1
# 每个 worker 处理的线程数
threads = 2

# The maximum number of simultaneous clients.
worker_connections = 250

# The number of seconds to wait for requests on a Keep-Alive connection.
keepalive = 60

# Workers silent for more than this many seconds are killed and restarted
timeout = 300
# Add timestamp to log filename before a worker process exits
TIME_FORMAT = "%Y-%m-%dT%H-%M-%S.%f"


def worker_int(worker):
    pid = worker.pid
    hostname = socket.gethostname()
    # 处理代码


worker_abort = worker_int


def child_exit(server, worker):
    try:
        worker_int(worker)
    except:
        traceback.print_exc()


worker_exit = child_exit
~~~

**配置 nginx.conf**

~~~nginx
server {
    listen 80;
    server_name your_domain.com www.your_domain.com; # 你的域名或服务器IP

    # 处理静态文件（Nginx效率更高）
    location /static {
        alias /path/to/your/project/static;
        expires 30d;
    }

    # 将所有非静态文件请求转发给 Gunicorn
    location / {
        proxy_pass http://127.0.0.1:8000; # 指向Gunicorn绑定的地址
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
~~~

**启动 gunicorn**

~~~bash
gunicorn wsgi:app -c gunicorn.py
~~~



## uvicorn 拉起 web 应用

Uvicorn 是一个基于 uvloop 和 httptools 构建的轻量级、超快速的 ASGI 服务器，特别适合部署 FastAPI、Starlette 等异步 Web 框架。

典型的生产环境架构：
**客户端 -> Nginx (反向代理/处理静态文件) -> Gunicorn -> Unicorn(ASGI服务器) ->FastAPI App**

>补充：使用 Gunicorn 作为进程管理器， Uvicorn做为 worker
>
>~~~bash
>gunicorn -w 4 -k uvicorn.workers.UvicornWorker main:app
>~~~



**配置 unicorn.py**

~~~python
# uvicorn_config.py
import multiprocessing

# 服务器配置
host = "0.0.0.0"
port = 8000
workers = multiprocessing.cpu_count() * 2 + 1
reload = False  # 生产环境设为 False

# 日志配置
log_level = "info"
~~~

**配置 nginx.conf**

~~~nginx
server {
    listen 80;
    server_name example.com;
    
    location / {
        proxy_pass http://127.0.0.1:8000;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    }
}
~~~

**启动 uvicorn**

~~~bash
uvicorn main:app --config uvicorn.py
~~~

使用 Gunicorn + Uvicorn worker

~~~bash
gunicorn -w 4 -k uvicorn.workers.UvicornWorker main:app
~~~

