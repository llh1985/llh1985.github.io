---
layout: post
title: docker-compose简单例子
categories: Other
description: docker-compose简单例子
keywords: docker, compose, docker-compose
---

2017-09-14-docker-compose-example.md
   
   
--- 
#转载自[**Docker—从入门到实践** >  **Docker三剑客之Compose项目** > **使用** ](https://yeasy.gitbooks.io/docker_practice/content/compose/usage.html)


## 使用

### 术语
首先介绍几个术语。

* 服务（service）：一个应用容器，实际上可以运行多个相同镜像的实例。
* 项目(project)：由一组关联的应用容器组成的一个完整业务单元。

可见，一个项目可以由多个服务（容器）关联而成，Compose 面向项目进行管理。

### 场景
下面，我们创建一个经典的 Web 项目：一个 [Haproxy](http://www.haproxy.org/)，挂载三个 Web 容器。

创建一个 `compose-haproxy-web` 目录，作为项目工作目录，并在其中分别创建两个子目录：`haproxy` 和 `web`。

### web 子目录

这里用 Python 程序来提供一个简单的 HTTP 服务，打印出访问者的 IP 和 实际的本地 IP。

#### index.py

编写一个 `index.py` 作为服务器文件，代码为
```bash
#!/usr/bin/python
#authors: yeasy.github.com
#date: 2013-07-05

import sys
import BaseHTTPServer
from SimpleHTTPServer import SimpleHTTPRequestHandler
import socket
import fcntl
import struct
import pickle
from datetime import datetime
from collections import OrderedDict

class HandlerClass(SimpleHTTPRequestHandler):
    def get_ip_address(self,ifname):
        s = socket.socket(socket.AF_INET, socket.SOCK_DGRAM)
        return socket.inet_ntoa(fcntl.ioctl(
            s.fileno(),
            0x8915,  # SIOCGIFADDR
            struct.pack('256s', ifname[:15])
        )[20:24])
    def log_message(self, format, *args):
        if len(args) < 3 or "200" not in args[1]:
            return
        try:
            request = pickle.load(open("pickle_data.txt","r"))
        except:
            request=OrderedDict()
        time_now = datetime.now()
        ts = time_now.strftime('%Y-%m-%d %H:%M:%S')
        server = self.get_ip_address('eth0')
        host=self.address_string()
        addr_pair = (host,server)
        if addr_pair not in request:
            request[addr_pair]=[1,ts]
        else:
            num = request[addr_pair][0]+1
            del request[addr_pair]
            request[addr_pair]=[num,ts]
        file=open("index.html", "w")
        file.write("<!DOCTYPE html> <html> <body><center><h1><font color=\"blue\" face=\"Georgia, Arial\" size=8><em>HA</em></font> Webpage Visit Results</h1></center>")
        for pair in request:
            if pair[0] == host:
                guest = "LOCAL: "+pair[0]
            else:
                guest = pair[0]
            if (time_now-datetime.strptime(request[pair][1],'%Y-%m-%d %H:%M:%S')).seconds < 3:
                file.write("<p style=\"font-size:150%\" >#"+ str(request[pair][1]) +": <font color=\"red\">"+str(request[pair][0])+ "</font> requests " + "from &lt<font color=\"blue\">"+guest+"</font>&gt to WebServer &lt<font color=\"blue\">"+pair[1]+"</font>&gt</p>")
            else:
                file.write("<p style=\"font-size:150%\" >#"+ str(request[pair][1]) +": <font color=\"maroon\">"+str(request[pair][0])+ "</font> requests " + "from &lt<font color=\"navy\">"+guest+"</font>&gt to WebServer &lt<font color=\"navy\">"+pair[1]+"</font>&gt</p>")
        file.write("</body> </html>")
        file.close()
        pickle.dump(request,open("pickle_data.txt","w"))

if __name__ == '__main__':
    try:
        ServerClass  = BaseHTTPServer.HTTPServer
        Protocol     = "HTTP/1.0"
        addr = len(sys.argv) < 2 and "0.0.0.0" or sys.argv[1]
        port = len(sys.argv) < 3 and 80 or int(sys.argv[2])
        HandlerClass.protocol_version = Protocol
        httpd = ServerClass((addr, port), HandlerClass)
        sa = httpd.socket.getsockname()
        print "Serving HTTP on", sa[0], "port", sa[1], "..."
        httpd.serve_forever()
    except:
        exit()
```

#### index.html
生成一个临时的 `index.html` 文件，其内容会被 index.py 更新。
```bash
$ touch index.html
```

#### Dockerfile
生成一个 Dockerfile，内容为
```bash
FROM python:2.7
WORKDIR /code
ADD . /code
EXPOSE 80
CMD python index.py
```

### haproxy 目录
在其中生成一个 `haproxy.cfg` 文件，内容为
```bash
global
  log 127.0.0.1 local0
  log 127.0.0.1 local1 notice

defaults
  log global
  mode http
  option httplog
  option dontlognull
  timeout connect 5000ms
  timeout client 50000ms
  timeout server 50000ms

listen stats
    bind 0.0.0.0:70
    stats enable
    stats uri /

frontend balancer
    bind 0.0.0.0:80
    mode http
    default_backend web_backends

backend web_backends
    mode http
    option forwardfor
    balance roundrobin
    server weba weba:80 check
    server webb webb:80 check
    server webc webc:80 check
    option httpchk GET /
    http-check expect status 200
```
### docker-compose.yml
编写 docker-compose.yml 文件，这个是 Compose 使用的主模板文件。内容十分简单，指定 3 个 web 容器，以及 1 个 haproxy 容器。

```bash
weba:
    build: ./web
    expose:
        - 80

webb:
    build: ./web
    expose:
        - 80

webc:
    build: ./web
    expose:
        - 80

haproxy:
    image: haproxy:latest
    volumes:
        - ./haproxy:/haproxy-override
        - ./haproxy/haproxy.cfg:/usr/local/etc/haproxy/haproxy.cfg:ro
    links:
        - weba
        - webb
        - webc
    ports:
        - "80:80"
        - "70:70"
    expose:
        - "80"
        - "70"
```
  

### 运行 compose 项目
现在 compose-haproxy-web 目录长成下面的样子。
```bash
compose-haproxy-web
├── docker-compose.yml
├── haproxy
│   └── haproxy.cfg
└── web
    ├── Dockerfile
    ├── index.html
    └── index.py
```
在该目录下执行 `docker-compose up` 命令，会整合输出所有容器的输出。
```
$sudo docker-compose up
Recreating composehaproxyweb_webb_1...
Recreating composehaproxyweb_webc_1...
Recreating composehaproxyweb_weba_1...
Recreating composehaproxyweb_haproxy_1...
Attaching to composehaproxyweb_webb_1, composehaproxyweb_webc_1, composehaproxyweb_weba_1, composehaproxyweb_haproxy_1
```

此时访问本地的 80 端口，会经过 haproxy 自动转发到后端的某个 web 容器上，刷新页面，可以观察到访问的容器地址的变化。

访问本地 70 端口，可以查看到 haproxy 的统计信息。

当然，还可以使用 consul、etcd 等实现服务发现，这样就可以避免手动指定后端的 web 容器了，更为灵活。


------
------


#转载自[Docker Compose 命令说明](https://yeasy.gitbooks.io/docker_practice/content/compose/commands.html)

## Compose 命令说明

### 命令对象与格式
对于 Compose 来说，大部分命令的对象既可以是项目本身，也可以指定为项目中的服务或者容器。如果没有特别的说明，命令对象将是项目，这意味着项目中所有的服务都会受到命令影响。

执行 `docker-compose [COMMAND] --help` 或者 `docker-compose help [COMMAND]` 可以查看具体某个命令的使用格式。

Compose 命令的基本的使用格式是

```sh
docker-compose [-f=<arg>...] [options] [COMMAND] [ARGS...]
```

### 命令选项

* `-f, --file FILE` 指定使用的 Compose 模板文件，默认为 `docker-compose.yml`，可以多次指定。
* `-p, --project-name NAME` 指定项目名称，默认将使用所在目录名称作为项目名。
* `--x-networking` 使用 Docker 的可拔插网络后端特性（需要 Docker 1.9 及以后版本）。
* `--x-network-driver DRIVER` 指定网络后端的驱动，默认为 `bridge`（需要 Docker 1.9 及以后版本）。
* `--verbose` 输出更多调试信息。
* `-v, --version` 打印版本并退出。

### 命令使用说明

#### `build`
格式为 `docker-compose build [options] [SERVICE...]`。

构建（重新构建）项目中的服务容器。

服务容器一旦构建后，将会带上一个标记名，例如对于 web 项目中的一个 db 容器，可能是 web_db。

可以随时在项目目录下运行 `docker-compose build` 来重新构建服务。

选项包括：

* `--force-rm` 删除构建过程中的临时容器。
* `--no-cache` 构建镜像过程中不使用 cache（这将加长构建过程）。
* `--pull` 始终尝试通过 pull 来获取更新版本的镜像。

#### `help`

获得一个命令的帮助。

#### `kill`
格式为 `docker-compose kill [options] [SERVICE...]`。

通过发送 `SIGKILL` 信号来强制停止服务容器。

支持通过 `-s` 参数来指定发送的信号，例如通过如下指令发送 `SIGINT` 信号。

```sh
$ docker-compose kill -s SIGINT
```

#### `logs`
格式为 `docker-compose logs [options] [SERVICE...]`。

查看服务容器的输出。默认情况下，docker-compose 将对不同的服务输出使用不同的颜色来区分。可以通过 `--no-color` 来关闭颜色。

该命令在调试问题的时候十分有用。

#### `pause`
格式为 `docker-compose pause [SERVICE...]`。

暂停一个服务容器。

#### `port`
格式为 `docker-compose port [options] SERVICE PRIVATE_PORT`。

打印某个容器端口所映射的公共端口。

选项：

* `--protocol=proto` 指定端口协议，tcp（默认值）或者 udp。
* `--index=index` 如果同一服务存在多个容器，指定命令对象容器的序号（默认为 1）。

#### `ps`
格式为 `docker-compose ps [options] [SERVICE...]`。

列出项目中目前的所有容器。

选项：

* `-q` 只打印容器的 ID 信息。 

#### `pull`
格式为 `docker-compose pull [options] [SERVICE...]`。

拉取服务依赖的镜像。

选项：

* `--ignore-pull-failures` 忽略拉取镜像过程中的错误。

#### `restart`
格式为 `docker-compose restart [options] [SERVICE...]`。

重启项目中的服务。

选项：

* `-t, --timeout TIMEOUT` 指定重启前停止容器的超时（默认为 10 秒）。

#### `rm`
格式为 `docker-compose rm [options] [SERVICE...]`。

删除所有（停止状态的）服务容器。推荐先执行 `docker-compose stop` 命令来停止容器。

选项：

* `-f, --force` 强制直接删除，包括非停止状态的容器。一般尽量不要使用该选项。
* `-v` 删除容器所挂载的数据卷。

#### `run`
格式为 `docker-compose run [options] [-p PORT...] [-e KEY=VAL...] SERVICE [COMMAND] [ARGS...]`。

在指定服务上执行一个命令。

例如：

```sh
$ docker-compose run ubuntu ping docker.com
```

将会启动一个 ubuntu 服务容器，并执行 `ping docker.com` 命令。

默认情况下，如果存在关联，则所有关联的服务将会自动被启动，除非这些服务已经在运行中。

该命令类似启动容器后运行指定的命令，相关卷、链接等等都将会按照配置自动创建。

两个不同点：

* 给定命令将会覆盖原有的自动运行命令；
* 不会自动创建端口，以避免冲突。

如果不希望自动启动关联的容器，可以使用 `--no-deps` 选项，例如

```sh
$ docker-compose run --no-deps web python manage.py shell
```

将不会启动 web 容器所关联的其它容器。

选项：

* `-d` 后台运行容器。
* `--name NAME` 为容器指定一个名字。
* `--entrypoint CMD` 覆盖默认的容器启动指令。
* `-e KEY=VAL` 设置环境变量值，可多次使用选项来设置多个环境变量。
* `-u, --user=""` 指定运行容器的用户名或者 uid。
* `--no-deps` 不自动启动关联的服务容器。
* `--rm` 运行命令后自动删除容器，`d` 模式下将忽略。
* `-p, --publish=[]` 映射容器端口到本地主机。
* `--service-ports` 配置服务端口并映射到本地主机。
* `-T` 不分配伪 tty，意味着依赖 tty 的指令将无法运行。

#### `scale`
格式为 `docker-compose scale [options] [SERVICE=NUM...]`。

设置指定服务运行的容器个数。

通过 `service=num` 的参数来设置数量。例如：

```sh
$ docker-compose scale web=3 db=2
```

将启动 3 个容器运行 web 服务，2 个容器运行 db 服务。

一般的，当指定数目多于该服务当前实际运行容器，将新创建并启动容器；反之，将停止容器。

选项：

* `-t, --timeout TIMEOUT` 停止容器时候的超时（默认为 10 秒）。

#### `start`
格式为 `docker-compose start [SERVICE...]`。

启动已经存在的服务容器。

#### `stop`
格式为 `docker-compose stop [options] [SERVICE...]`。

停止已经处于运行状态的容器，但不删除它。通过 `docker-compose start` 可以再次启动这些容器。

选项：

* `-t, --timeout TIMEOUT` 停止容器时候的超时（默认为 10 秒）。

#### `unpause`
格式为 `docker-compose unpause [SERVICE...]`。

恢复处于暂停状态中的服务。

#### `up`
格式为 `docker-compose up [options] [SERVICE...]`。

该命令十分强大，它将尝试自动完成包括构建镜像，（重新）创建服务，启动服务，并关联服务相关容器的一系列操作。

链接的服务都将会被自动启动，除非已经处于运行状态。

可以说，大部分时候都可以直接通过该命令来启动一个项目。

默认情况，`docker-compose up` 启动的容器都在前台，控制台将会同时打印所有容器的输出信息，可以很方便进行调试。

当通过 `Ctrl-C` 停止命令时，所有容器将会停止。

如果使用 `docker-compose up -d`，将会在后台启动并运行所有的容器。一般推荐生产环境下使用该选项。

默认情况，如果服务容器已经存在，`docker-compose up` 将会尝试停止容器，然后重新创建（保持使用 `volumes-from` 挂载的卷），以保证新启动的服务匹配 `docker-compose.yml` 文件的最新内容。如果用户不希望容器被停止并重新创建，可以使用 `docker-compose up --no-recreate`。这样将只会启动处于停止状态的容器，而忽略已经运行的服务。如果用户只想重新部署某个服务，可以使用 `docker-compose up --no-deps -d <SERVICE_NAME>` 来重新创建服务并后台停止旧服务，启动新服务，并不会影响到其所依赖的服务。

选项：

* `-d` 在后台运行服务容器。
* `--no-color` 不使用颜色来区分不同的服务的控制台输出。
* `--no-deps` 不启动服务所链接的容器。
* `--force-recreate` 强制重新创建容器，不能与 `--no-recreate` 同时使用。
* `--no-recreate` 如果容器已经存在了，则不重新创建，不能与 `--force-recreate` 同时使用。
* `--no-build` 不自动构建缺失的服务镜像。
* `-t, --timeout TIMEOUT` 停止容器时候的超时（默认为 10 秒）。

#### `migrate-to-labels`
格式为 `docker-compose  migrate-to-labels`。

重新创建容器，并添加 label。

主要用于升级 1.2 及更早版本中创建的容器，添加缺失的容器标签。

实际上，最彻底的办法当然是删除项目，然后重新创建。

#### `version`
格式为 `docker-compose version`。

打印版本信息。