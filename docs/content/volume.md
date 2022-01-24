# Docker 存储和卷

## 绑定挂载  
`docker run --name web -dp 5001:5000 nginx:latest --mount type=bind,source=/src/webapp,target=/usr/share，readonly=true`

* -mount type=bind：将本地`/src/webapp`目录挂载到容器的`/usr/share`目录中(如果本地目录不存在，docker 会报错)
* readonly=true：创建只读挂载，容器无法修改卷的内容

绑定挂载有两个缺点：
* 容器与特定的主机文件系统绑定在一起，如果容器依赖主机文件系统特定位置的内容，那么无法移植到无法访问特定位置的主机
* 可能会造成容器之间的冲突。因为可能多个容器依赖主机上同一个文件。

## 常驻内存存储（将容器数据挂载到本机内存中）  
`docker run --name web -dp 5001:5000 nginx:latest --mount type=tmpfs,target=/usr/share`

* -mount type=tmpfs：将容器的`/usr/share`目录挂载到本地`内存`目录中
* target=/usr/share：在`/usr/shar`创建的所有文件都将写入内存而不是磁盘中

## Docker 卷（由 Docker 管理挂载，默认存储在`/var/lib/docker/volumes`中） 

##### 使用 Docker 具名卷启动容器
`sudo docker run -dp 3000:3000 -w /app -v todo-db:/app:ro node:12-alpine sh -c "yarn install && yarn run dev"`
* -d：后台运行
* -p：端口映射，本地：容器
* -w /app：设置“工作目录”或命令将运行的当前目录
* -v todo-db:/app：将宿主机`todo-db 卷`挂载到容器`/app`目录下
* ro/re：设置读写权限，ro 表示只读（说明这个路径只能通过宿主机来操作，容器内部是无法操作！）；rw 表示可读可写
* node:12-alpine：要使用的镜像
* sh -c：将字符串按照完整命令执行
* "yarn install && yarn run dev"：安装依赖并运行

##### 使用 Docker 匿名卷启动容器
`docker run -d -P --name nginx01 -v /etc/nginx nginx`

##### 查看 Docker 卷
`docker volume inspect todo-db`
```json
[
    {
        "CreatedAt": "2019-09-26T02:18:36Z",
        "Driver": "local",
        "Labels": {},
        "Mountpoint": "/var/lib/docker/volumes/todo-db/_data",
        "Name": "todo-db",
        "Options": {},
        "Scope": "local"
    }
]
```

##### 创建卷
`docker volume create --driver local --label example todo-db`
* --driver local：使用名为 local 的插件引擎创建卷
* --label：设置标签为 example

##### 复制卷（`--volumes-from`：将一个或多个容器中挂载的卷复制到新的容器中去  ）
`docker run --name target --volumes-from source1 --volumes-from source2 alpine:latest`
* 将 source1 和 source2 容器里面的挂载复制到新的 target 容器中

`--volumes-from`在以下 3 种情况下无法使用：
1. 如果要构建的容器需要将共享卷安装包其他位置，就不能使用`--volumes-from`
2. 当要复制的卷在文件系统的位置与其他已存在的卷或新建的卷冲突时，就不能使用`--volumes-from`。如果冲突，卷的使用者只能收到一个卷的定义
3. 当用户需要修改卷的读写权限时不能使用`--volumes-from`。因为`--volumes-from`复制了卷的完整的定义，包括权限

##### 清理卷
匿名卷有 2 种清理方式：
1. 当卷挂载的容器被自动清理时，匿名卷也会被自动删除。（可通过 docker run -rm 或 docker rm -v 命令实现）
2. 通过`docker volume remove`命令手动删除匿名卷

命令卷与匿名卷不同必须手动删除
`docker volume rm todo-db`

##### Docker 卷命令
功能|命令
--|:--:
创建卷|`$ docker volume create todo-db`|
检查卷|`$ docker volume inspect todo-db`|
查看所有卷|`$ docker volume ls`|
删除卷|`$ docker volume rm todo-db`|
删除容器并删除容器挂载的数据卷|`$ docker rm -v nginx01`|
删除所有数据卷|`$ docker volume prune`|
根据过滤条件删除所有数据卷|`$ docker volume prune --filter example=cassandra`|
取消用户确认步骤自动根据过滤条件删除所有数据卷|`$ docker volume prune --filter example=location --force`|
