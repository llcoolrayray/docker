## DockerFile 介绍
镜像的定制实际上就是定制每一层所添加的配置、文件。如果我们可以把每一层修改、安装、构建、操作的命令都写入一个脚本，
用这个脚本来构建、定制镜像，那么构建镜像会变得非常简单。这个脚本就是 Dockerfile。

Dockerfile 是一个文本文件，其中包含了一条条的 指令(Instruction)，每一条指令构建一层，因此每一条指令的内容，
就是描述该层应当如何构建。

### FROM 指定基础镜像
以构建 nginx 镜像为例，其 DockerFile 如下：
```dockerfile
FROM nginx
RUN echo '<h1>Hello, Docker!</h1>' > /usr/share/nginx/html/index.html
```

构建镜像需要以一个镜像为基础，在其基础上进行修改，添加等操作来产生新的镜像。上图中`FROM`指令就是用于指定基础镜像。
在`DockerFile`中`FROM`指令是必备的指令，且必须是第一条指令。`Docker Hub`中有很多官方的基础镜像，如`node:12-alpine`，
`maven`，`openjdk:8-jdk-alpine`等。

除了选择现有镜像为基础镜像外，Docker 还存在一个特殊的镜像，名为 scratch。这个镜像是虚拟的概念，并不实际存在，
它表示一个空白的镜像。
```dockerfile
FROM scratch
...
```

### RUN 执行命令
`RUN`指令是用来执行命令行命令的。它有两种格式：
* shell 格式：`RUN` <命令>，就像直接在命令行中输入的命令一样
```dockerfile
RUN echo '<h1>Hello, Docker!</h1>' > /usr/share/nginx/html/index.html
```
* exec 格式：`RUN ["命令", "参数1", "参数2"]`

Dockerfile 中每一个指令都会建立一层，为了避免镜像内容冗余，尽量将`RUN`指令写在一起。像如下这种错误的写法会将很多运行时不需要的东西，
都被装进了镜像里，比如编译环境、更新的软件包等等。结果就是产生非常臃肿、非常多层的镜像，不仅仅增加了构建部署的时间，也很容易出错。  
错误写法：
```dockerfile
FROM debian:stretch

RUN apt-get update
RUN apt-get install -y gcc libc6-dev make wget
RUN wget -O redis.tar.gz "http://download.redis.io/releases/redis-5.0.3.tar.gz"
RUN mkdir -p /usr/src/redis
RUN tar -xzf redis.tar.gz -C /usr/src/redis --strip-components=1
RUN make -C /usr/src/redis
RUN make -C /usr/src/redis install
```
正确写法：使用一个`RUN`指令，通过`&&`将不同的命令连接起来，使用`\`进行换行，按照命令首字母进行排序。
另外，下方的`dockerfile`中的最后一组命令为清理容器的命令。这很重要，因为镜像是多层存储的，每一层的东西并不会在下一层被删除而会被一直保留下来。
因此构建时要确保每一层只添加真正需要的东西，任何无关的东西都需要删除。
```dockerfile
FROM debian:stretch

RUN set -x; buildDeps='gcc libc6-dev make wget' \
    && apt-get update \
    && apt-get install -y $buildDeps \
    && wget -O redis.tar.gz "http://download.redis.io/releases/redis-5.0.3.tar.gz" \
    && mkdir -p /usr/src/redis \
    && tar -xzf redis.tar.gz -C /usr/src/redis --strip-components=1 \
    && make -C /usr/src/redis \
    && make -C /usr/src/redis install \
    && rm -rf /var/lib/apt/lists/* \
    && rm redis.tar.gz \
    && rm -r /usr/src/redis \
    && apt-get purge -y --auto-remove $buildDeps
```

`RUN`指令执行时会在当前镜像的基础上启动一个容器，并在该容器中执行所要求的命令。命令执行结束后会在当前镜像层之上提交新层并删除刚才所用到的容器。
### 构建镜像
在 Dockerfile 文件所在目录执行  
`docker build -t llcoolray/smart-infra-sight:v3.1 .`  
* `-t`：指定镜像名
* `:v3.1`：指定镜像`tag`
* `.`：构建上下文路径

```dockerfile
$ docker build -t nginx:v3 .
Sending build context to Docker daemon 2.048 kB
Step 1 : FROM nginx
 ---> e43d811ce2f4
Step 2 : RUN echo '<h1>Hello, Docker!</h1>' > /usr/share/nginx/html/index.html
 ---> Running in 9cdc27646c7b
 ---> 44aa4490ce2c
Removing intermediate container 9cdc27646c7b
Successfully built 44aa4490ce2c
```

### 镜像构建上下文
Docker 在运行时分为 Docker 引擎（也就是服务端守护进程）和客户端工具。Docker 的引擎提供了一组 REST API，被称为 Docker Remote API，
而如 docker 命令这样的客户端工具，则是通过这组 API 与 Docker 引擎交互，从而完成各种功能。  

构建时用户需要指定镜像构建上下文路径，`docker build`会将该路径下的所有文件发送到服务端守护进程。在大多数情况下，
最好从一个空目录作为上下文开始，并将 Dockerfile 保存在该目录中，仅添加构建 Dockerfile 所需的文件。

使用`.dockerignore`来忽略掉镜像构建上下文中不需要发送给 Docker 守护进程的文件。

### `docker build`的其它用法
#### 直接用 Git repo 进行构建
`docker build`可以通过 git url 构建镜像。  
`docker build -t hello-world https://github.com/docker-library/hello-world.git#master:amd64/hello-world`
* `-t`：指定镜像名
* `https://github.com/docker-library/hello-world.git`：git url
* `#master`：git 分支
* `amd64/hello-world`：构建目录

```dockerfile
# $env:DOCKER_BUILDKIT=0
# export DOCKER_BUILDKIT=0

$ docker build -t hello-world https://github.com/docker-library/hello-world.git#master:amd64/hello-world

Step 1/3 : FROM scratch
 --->
Step 2/3 : COPY hello /
 ---> ac779757d46e
Step 3/3 : CMD ["/hello"]
 ---> Running in d2a513a760ed
Removing intermediate container d2a513a760ed
 ---> 038ad4142d2b
Successfully built 038ad4142d2b
```

#### 用给定的 tar 压缩包构建
如果所给出的 URL 不是个 Git repo，而是个 tar 压缩包，那么 Docker 引擎会下载这个包，并自动解压缩，以其作为上下文，开始构建。
```dockerfile
$ docker build http://server/context.tar.gz
```

## DockerFile 指令
### COPY 复制文件
COPY 含义：  
将构建上下文中路径或文件复制到以当前层级镜像启动的容器的文件系统中

COPY 有两种格式：
```dockerfile
COPY [--chown=<user>:<group>] <src>... <dest>
COPY [--chown=<user>:<group>] ["<src>",... "<dest>"]
```

可使用通配符进行文件复制：
```dockerfile
COPY hom* /mydir/
COPY hom?.txt /mydir/
```

复制文件时可改变文件的权限：
```dockerfile
COPY --chown=55:mygroup files* /mydir/
COPY --chown=bin files* /mydir/
COPY --chown=1 files* /mydir/
COPY --chown=10:11 files* /mydir/
```

### ADD 更高级的复制文件
ADD 指令和 COPY 指令的功能基本一致。区别如下：
* ADD 指令复制时`<源路径>`可以是 URL，复制时 Docker 引擎会试图去下载这个链接的文件并放到`<目标路径>`下。
下载后的文件权限自动设置为 600，如果这并不是想要的权限，那么还需要增加额外的一层 RUN 进行权限调整，
另外，如果下载的是个压缩包，需要解压缩，也一样还需要额外的一层 RUN 指令进行解压缩。（还不如直接使用 RUN 指令，然后使用 wget 或者 curl 工具下载，
处理权限、解压缩、然后清理无用文件更合理。因此，这个功能其实并不实用，而且不推荐使用。）
* 如果`<源路径>`为一个 tar 压缩文件的话，压缩格式为 gzip, bzip2 以及 xz 的情况下，ADD 指令将会自动解压缩这个压缩文件到`<目标路径>`去。

```dockerfile
FROM scratch
ADD ubuntu-xenial-core-cloudimg-amd64-root.tar.gz /
```

Docker 官方的 Dockerfile 最佳实践文档 中要求，尽可能的使用 COPY，因为 COPY 的语义很明确，就是复制文件而已，
而 ADD 则包含了更复杂的功能，其行为也不一定很清晰。最适合使用 ADD 的场合，就是所提及的需要自动解压缩的场合。

另外需要注意的是，ADD 指令会令镜像构建缓存失效，从而可能会令镜像构建变得比较缓慢。

在使用该指令的时候还可以加上 --chown=<user>:<group> 选项来改变文件的所属用户及所属组。
```dockerfile
ADD --chown=55:mygroup files* /mydir/
ADD --chown=bin files* /mydir/
ADD --chown=1 files* /mydir/
ADD --chown=10:11 files* /mydir/
```
### CMD 容器启动命令
`CMD`指令的作用是为容器启动时提供默认的启动命令，或为`ENTRYPOINT`指令提供默认参数

`CMD`指令有 3 种形式：  
* `CMD`["executable","param1","param2"]：`exec`格式，建议使用这种方式  
* `CMD`["param1","param2"]：作为`ENTRYPOINT`的默认参数  
* `CMD` command param1 param2：`shell`格式  

`Dockerfile`中只能有一条`CMD`指令，如果有多条`CMD`指令，那么只有最后一条会生效

在运行时可以指定新的命令来替代`CMD`作为启动命令。比如，`ubuntu`镜像默认的`CMD`命令是`/bin/bash`，
执行`docker run -it ubuntu`命令会直接进入`bash`。也可以在运行时指定运行别的命令，如 `docker run -it ubuntu cat /etc/os-release`。
这就是用`cat /etc/os-release`命令替换了默认的`/bin/bash`命令了，输出了系统版本信息。

### ENTRYPOINT 入口点
`ENTRYPOINT`的作用和`CMD`一样，都是指定容器启动命令。`ENTRYPOINT`在运行时也可以替代，不过比`CMD`要略显繁琐，
需要通过`docker run`的参数`--entrypoint`来指定。

当`dockerfile`指定了`ENTRYPOINT`后，`CMD`的含义就发生了改变，不再是直接的运行其命令，而是将`CMD`的内容作为参数传给`ENTRYPOINT`指令。  

示例：通过`ENTRYPOINT`和`CMD`做容器启动的准备工作
下面的`dockerfile`是`redis`的官方镜像，可以看到`ENTRYPOINT`指定了一个脚本用于容器启动时执行。而`CMD`指令为该脚本提供参数
```dockerfile
FROM alpine:3.4
...
RUN addgroup -S redis && adduser -S -G redis redis
...
ENTRYPOINT ["docker-entrypoint.sh"]

EXPOSE 6379
CMD [ "redis-server" ]
```

该脚本的目的是根据`CMD`的参数判断用什么用户来启动`redis`,如果是`redis-server`则切换到`redis`用户身份启动服务，否则使用`root`身份启动服务。
```dockerfile
#!/bin/sh
...
# allow the container to be started with `--user`
if [ "$1" = 'redis-server' -a "$(id -u)" = '0' ]; then
	find . \! -user redis -exec chown redis '{}' +
	exec gosu redis "$0" "$@"
fi

exec "$@"
```

### ENV 设置环境变量
`ENV`指令的作用就是设置环境变量，可供其后的`RUN`，`ADD`等指令使用，或让运行中的容器使用。  

格式有两种： 
* `ENV <key> <value>`
* `ENV <key1>=<value1> <key2>=<value2>...`

示例：`node`官方`Dockerfile`中使用`ENV`设置`node`版本供其后的`RUN`指令使用。
```dockerfile
ENV NODE_VERSION 7.2.0

RUN curl -SLO "https://nodejs.org/dist/v$NODE_VERSION/node-v$NODE_VERSION-linux-x64.tar.xz" \
  && curl -SLO "https://nodejs.org/dist/v$NODE_VERSION/SHASUMS256.txt.asc" \
  && gpg --batch --decrypt --output SHASUMS256.txt SHASUMS256.txt.asc \
  && grep " node-v$NODE_VERSION-linux-x64.tar.xz\$" SHASUMS256.txt | sha256sum -c - \
  && tar -xJf "node-v$NODE_VERSION-linux-x64.tar.xz" -C /usr/local --strip-components=1 \
  && rm "node-v$NODE_VERSION-linux-x64.tar.xz" SHASUMS256.txt.asc SHASUMS256.txt \
  && ln -s /usr/local/bin/node /usr/local/bin/nodejs
```

`ENV`设置的环境变量支持这些指令使用：`ADD`、`COPY`、`ENV`、`EXPOSE`、`FROM`、`LABEL`、`USER`、`WORKDIR`、`VOLUME`、`STOPSIGNAL`、`ONBUILD`、`RUN`。

### ARG 构建参数
`ARG`指令的作用和`ENV`类似，都是设置环境变量。不同的是`ARG`设置的环境变量在容器运行时是不生效的。

通过`docker history`可查看`ARG`指令设置的环境变量。

`ARG`指令有生效范围，如果在`FROM`指令之前指定，那么只能用于`FROM`指令中。

例如下面的`dockerfile`中无法输出`${DOCKER_USERNAME}`变量的值。要想正常输出，必须在`FROM`之后再次指定`ARG`
```dockerfile
ARG DOCKER_USERNAME=library

FROM ${DOCKER_USERNAME}/alpine

RUN set -x ; echo ${DOCKER_USERNAME}
```

对于多阶段构建，在`FROM`指令之后都指定，只能适用与该阶段。

### VOLUME 定义匿名卷
`VOLUME`的作用是为容器定义匿名卷。  
格式有两种： 
* `VOLUME ["<路径1>", "<路径2>"...]`
* `VOLUME <路径>`

示例：容器的`/data`目录挂载为匿名卷，任何向`/data`中写入的数据都会记录到宿主机中。另外在运行时可以覆盖这个挂载设置，
使用`mydata`这个命名卷挂载到`/data`上，`docker run -d -v mydata:/data xxxx`。
```dockerfile
VOLUME /data
```

### EXPOSE 暴露端口
`EXPOSE`指令的作用是声明运行时容器提供服务端口，这只是一个声明，在运行时并不会因为这个声明应用就会开启这个端口的服务。  

格式为` EXPOSE <端口1> [<端口2>...]`

在`Dockerfile`中写入这样的声明有两个好处：
1. 帮助镜像使用者理解这个镜像服务的守护端口，以方便配置映射；
2. 在运行时使用随机端口映射时，也就是`docker run -P`时，会自动随机映射`EXPOSE`的端口。

### WORKDIR 指定工作目录
`WORKDIR`指令的作用是指定工作目录（当前目录），以后各层的当前目录就被改为指定的目录，如该目录不存在，`WORKDIR`会帮你建立目录。

### USER 指定当前用户
`USER`指令作用和`WORKDIR`相似，都是改变环境状态并影响以后的层。`WORKDIR`是改变工作目录，`USER`则是改变之后层的执行`RUN`,
`CMD`以及`ENTRYPOINT`这类命令的身份。

示例：
```dockerfile
RUN groupadd -r redis && useradd -r -g redis redis
USER redis
RUN [ "redis-server" ]
```

注意：`USER`只是帮助你切换到指定用户而已，这个用户必须是事先建立好的，否则无法切换。

### HEALTHCHECK 健康检查
`HEALTHCHECK`指令的作用是配置`Docker`容器的健康检查

格式有两种： 
* `HEALTHCHECK [选项] CMD <命令>`：设置检查容器健康状况的命令。命令的返回值决定了该次健康检查的成功与否：0：成功；1：失败
* `HEALTHCHECK NONE`：如果基础镜像有健康检查指令，使用这行可以屏蔽掉其健康检查指令

当在一个镜像指定了`HEALTHCHECK`指令后，用其启动容器，初始状态会为`starting`，在`HEALTHCHECK`指令检查成功后变为`healthy`，
如果连续一定次数失败，则会变为`unhealthy`。

`HEALTHCHECK`支持下列选项：
* `--interval=<间隔>`：两次健康检查的间隔，默认为 30 秒；
* `--timeout=<时长>`：健康检查命令运行超时时间，如果超过这个时间，本次健康检查就被视为失败，默认 30 秒；
* `--retries=<次数>`：当连续失败指定次数后，则将容器状态视为 unhealthy，默认 3 次。

示例：通过`curl`判断服务是否正常  
```dockerfile
FROM nginx
RUN apt-get update && apt-get install -y curl && rm -rf /var/lib/apt/lists/*
HEALTHCHECK --interval=5s --timeout=3s \
  CMD curl -fs http://localhost/ || exit 1
```

可通过`docker inspect`查看容器健康检查的数据

### ONBUILD 为他人作嫁衣裳
`ONBUILD`是一个特殊的指令，它后面跟的是其它指令，比如`RUN`,`COPY` 等，而这些指令，在当前镜像构建时并不会被执行。
只有当以当前镜像为基础镜像，去构建下一级镜像的时候才会被执行。

### LABEL 为镜像添加元数据
`LABEL`指令用来给镜像以键值对的形式添加一些元数据（metadata）。可以通过这些元数据来标明作者，文档地址等信息8
```dockerfile
LABEL <key>=<value> <key>=<value> <key>=<value> ...
```

## DockerFile 多阶段构建
多阶段构建最大的作用是缩小镜像大小。
示例：使用多阶段构建，可以在一个`DockerFile`中使用多个`FROM`语句，每一个`FROM`语句表示一个新的构建的开始。可以选择性的将上一阶段的构建复制到另一个
阶段，使得最终镜像只保留它所必须的，从而减少镜像大小。

将`builder`阶段的`/go/src/github.com/alexellis/href-counter/app`复制到`./`
`COPY --from=builder /go/src/github.com/alexellis/href-counter/app ./`

```dockerfile
# syntax=docker/dockerfile:1
# syntax=docker/dockerfile:1
FROM golang:1.16 AS builder
WORKDIR /go/src/github.com/alexellis/href-counter/
RUN go get -d -v golang.org/x/net/html  
COPY app.go    ./
RUN CGO_ENABLED=0 GOOS=linux go build -a -installsuffix cgo -o app .

FROM alpine:latest  
RUN apk --no-cache add ca-certificates
WORKDIR /root/
COPY --from=builder /go/src/github.com/alexellis/href-counter/app ./
CMD ["./app"]  
```

也可以使用`COPY --from`指令从外部的镜像中复制文件
`COPY --from=nginx:latest /etc/nginx/nginx.conf /nginx.conf`