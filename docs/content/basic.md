### 下载镜像并运行容器
`docker run --rm -d --name web -p 5001:5000 nginx:latest --restart `  
`01d272e5d1f4ea00427e4cdbc89d5f3d7b75a87cec414564a89492fbb6386b95`

容器启动后会在终端输出该容器 ID，一般会用一个变量存储该 ID以供其他命令使用。
* run：从本地寻找镜像，若本地没有则从 dockerhub 寻找镜像并启动容器
* -d：后台运行
* --name：定义容器名
* nginx:latest：nginx 为要下载的镜像，:latest 为镜像标签表示最新版本
* -p：将本地的 5001 端口映射到容器的 5000 端口
* --rm：容器停止时删除容器内数据

### 交互式运行容器
`docker run -it --name web_test busybox:1.29 /bin/sh`  
* -it：交互式运行容器  
* busybox:1.29：表示下载的镜像为 1.29 版本的 busybox  
* /bin/sh：放在镜像名后的是命令，这里我们希望有个交互式 Shell，因此用的是 /bin/bash

退出容器：exit

### 运行容器时指定容器内环境变量
`docker run -d --name wp-db -e MYSQL_DB_USER=root -e MYSQL_ROOT_PASSWORD=ch2demo mysql:5`

* -e：设置容器的环境变量（镜像中包含初始化脚本会读取命令中的变量）

### Docker 容器重启策略

##### 启动重启并设置重启策略
`docker run -d --name web -p 5001:5000 nginx:latest --restart=always `

* --restart：配置重启策略

##### 重启配置项 `--restart`
功能|命令
--|:--:
默认策略，在容器退出时不重启容器|`no`|
在容器非正常退出时（退出状态非0），才会重启容器|`on-failure`|
在容器非正常退出时重启容器，最多重启3次|`on-failure:3`|
在容器退出时总是重启容器|`always`|
在容器退出时总是重启容器，但是不考虑在Docker守护进程启动时就已经停止了的容器|`unless-stopped`|

### 常见命令

##### 启动/停止/删除容器
功能|命令
--|:--:
查看正在运行的容器|`$ docker ps`|
查看所有运行的容器|`$ docker ps -a`|
查看所有镜像|`$ docker images`|
查看所有镜像（包含镜像中间层）|`$ docker images -a`|
创建容器，但不启动|`$ docker create`|
启动一个已停止的容器|`$ docker start b750bbbcfd88`|
停止容器|`$ docker stop 1e560fca3906`|
重启容器|`$ docker restart 1e560fca3906`|
删除容器|`$ docker rm 1e560fca3906`|
删除镜像|`$ docker rmi 1e560fca3906`|
停止并删除容器|`$ docker rm -f 1e560fca3906`|

##### 查看容器信息
功能|命令
--|:--:
查看容器日志|`$ docker logs 1e560fca3906`|
持续查看更新的容器日志|`$ docker logs -f 1e560fca3906`|
查看容器的元数据|`$ docker stop 1e560fca3906 `|
查看容器中哪些进程正在运行|`$ docker top 1e560fca3906`|
查看容器底层信息|`$ docker inspect 1e560fca3906`|

##### 导入导出
功能|命令
--|:--:
导入容器快照|`$ docker import - test/ubuntu:v1.0`|
导出容器|`$ docker export 7691a814370e > ubuntu.tar`|
导入镜像|`$ docker load -i spring-boot-docker.tar  `|
导出镜像|`$ docker save spring-boot-docker  -o  /home/wzh/docker/spring-boot-docker.tar`|

##### 其它容器命令
功能|命令
--|:--:
在运行的容器中执行命令 |`$ docker exec web ps`|
容器重命名 |`$ docker rename old-name new-name`|
进入到容器中|`$ docker attach 1e560fca3906`|
进入到容器中（推荐使用，退出容器终端不会导致容器的停止）|`$ docker exec -it 243c32535da7 /bin/bash`|
进入到容器中（推荐使用，退出容器终端不会导致容器的停止）|`$ docker exec -it 243c32535da7 /bin/bash`|

### 常见重启操作
##### 获取容器 ID  
1. 使用 shell 脚本创建容器
```shell script
CID=$(docker create nginx:latest)
echo $CID
```

##### 获取最后创建的容器 ID  
```shell script
CID=$(docker ps --latest --quiet)
echo $CID
```

##### 根据容器 ID 变量启动容器 
```shell script
docker start $AGENT_CID
```

2. `docker create --cidfile /tmp/web.cid nginx`  
* --cidfile /tmp/web.cid: 在`/tmp/web.cid`文件生成容器 ID（cid file）

##### 以只读方式启动容器
`docker run -d --name wp --read-only wordpress`  
* --read-only: 以只读方式启动容器，使它在文件系统层面上是“只读”的；这个功能可以让你为容器中运行的应用限定特定的文件写入路径；
此功能结合“数据卷”（volumes）使用可以确保容器中运行的程序只能将数据写入到事先指定的路径下。

##### 查看容器元数据
`docker inspect 1e560fca3906`  
* docker inspect：用于显示 docker 为容器维护的所有元数据

`docker inspect --format "{{.State.Running}}" wp`
* --format：用来转换元数据的格式，上面的命令可以查看`wp`容器是否正在运行

##### 自动重启容器
创建容器时使用`--restart`标识配置重启策略，有以下4种策略：
* no：默认策略，在容器退出时不重启容器
* on-failure：在容器非正常退出时（退出状态非0），才会重启容器
* on-failure:3：在容器非正常退出时重启容器，最多重启3次
* always：在容器退出时总是重启容器
* unless-stopped：在容器退出时总是重启容器，但是不考虑在Docker守护进程启动时就已经停止了的容器

 

### 其它
##### Docker 若有自动化的需求，可以将 containerID 输出到指定的文件中
`docker create --cidfile ./web.cid nginx`