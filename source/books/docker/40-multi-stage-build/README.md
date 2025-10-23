# 多阶段构建镜像

多阶段构建可以优化镜像体积。下面分别使用 golang 和 python 程序演示多阶段构建镜像。



## Golang 程序

#### 1. 准备 go 源代码

如下是 go 程序源代码 server.go

~~~go
package main
import (
    "fmt"
)

func main(){
  fmt.Println("hello world")
}
~~~

#### 2. 普通构建的 dockerfile

如下是文件： `dockerfile`

~~~dockerfile
FROM golang:1.10.3

WORKDIR /app

COPY server.go .

# 构建优化
RUN CGO_ENABLED=0 GOOS=linux \
    go build -a -installsuffix cgo -ldflags='-s -w' -o server

# 运行应用
CMD ["/app/server"]
~~~

#### 3. 多阶段构建版本的 dockefile

如下是文件： `dockerfile_multi_stage`

~~~dockerfile
# 构建阶段
FROM golang:1.10.3 as builder

WORKDIR /app

COPY server.go .

# 优化构建参数
RUN CGO_ENABLED=0 GOOS=linux \
    go build -a -installsuffix cgo \
    -ldflags='-s -w -extldflags "-static"' \
    -o server server.go

# 运行阶段 - 使用更小的基础镜像
FROM scratch

# 复制可执行文件
COPY --from=builder /app/server /app/

# 运行应用
CMD ["/app/server"]
~~~

其中，
- `FROM scratch` 表示使用一个非常精简的镜像。
- `COPY --from=builder` 表示从名为 `builder` 的阶段复制文件。这里也可以使用阶段索引的方式，第一个阶段的索引为 `0`，那可以如下使用：`COPY --from=0`
- `COPY --from` 不仅可以从前面的阶段拷贝文件，还可以从一个已经存在的镜像拷贝文件。直接将 etcd 镜像中的程序拷贝到当前镜像，这样生成程序镜像时，就不需要编译 etcd 编码了。

~~~dockerfile
  FROM ubuntu:16.04
  
  COPY --from=quay.io/coreos/etcd:v3.3.9 /usr/local/bin/etcd /usr/local/bin/
~~~



#### 4. 构建镜像

~~~bash
# 普通构建
go build -t goapp:v1 -f dockerfile .

# 多阶段构建
go build -t goapp:v2 -f dockerfile_multi_stage .
~~~

#### 5. 查看镜像体积

~~~bash
[root@me ttt]# docker images | grep goapp
goapp                                 v2            1d004e532a05   3 minutes ago       1.21MB
goapp                                 v1            16faec1d0941   5 minutes ago       800MB
~~~



## Python 程序

下面以 flask web 项目为例，介绍多阶段构建。

#### 1. 准备项目

**app.py**

~~~python
from flask import Flask


app = Flask(__name__)


@app.route('/')
def hello():
    return 'Hello World!~'                       
~~~

**requirements.txt**

~~~tex
flask
~~~



#### 2. 普通构建

~~~dockerfile
# dockerfile

FROM python:3.10-alpine
WORKDIR /code
EXPOSE 5000

COPY app.py .
COPY requirements.txt .

RUN pip install --no-cache-dir -r requirements.txt \
    -i https://mirrors.aliyun.com/pypi/simple

ENV FLASK_APP=app.py FLASK_RUN_HOST=0.0.0.0

CMD ["flask", "run", "--debug"]
~~~



#### 3. 多阶段构建

~~~dockerfile
# dockerfile_multi_stage

# 构建阶段
FROM python:3.10-alpine as builder
WORKDIR /code
EXPOSE 5000

COPY app.py .
COPY requirements.txt .

RUN pip install --user --no-cache-dir -r requirements.txt \
    -i https://mirrors.aliyun.com/pypi/simple


# 运行阶段
FROM python:3.10-alpine
WORKDIR /code
EXPOSE 5000

COPY --from=builder /root/.local /root/.local
COPY app.py .
COPY requirements.txt .

# 只设置运行所需的环境变量
ENV PATH=/root/.local/bin:$PATH
ENV FLASK_APP=app.py FLASK_RUN_HOST=0.0.0.0

CMD ["flask", "run", "--debug"]
~~~



#### 4. 构建镜像

~~~bash
docker build -f dockerfile -t flask-app:v1 .
docker build -f dockerfile_multi_stage -t flask-app:v2 .
~~~



#### 5. 查看镜像体积

~~~bash
[root@me ~]# docker images | grep flask-app
flask-app                             v2            29c28c2a2207   45 minutes ago   49.3MB
flask-app                             v1            8d7f3a25d678   54 minutes ago   55.7MB
~~~
