#### DockerFile 最佳实践
### 使用`.dockerignore`忽略不需要的文件
使用`.dockerignore`来忽略掉镜像构建上下文中不需要发送给 Docker 守护进程的文件。

### 使用多阶段构建
使用多阶段构建能够大幅度减少最终的镜像大小。

### 不要安装不必要的包
为了减少复杂性、依赖关系、文件大小和构建时间，应避免安装额外的或不必要的软件包。

### 尽量减少层数
只有`RUN`, `COPY`,`ADD`会创建镜像层。其他指令创建临时中间图像，并且不增加构建的大小。

### 对多行参数进行排序
对多个参数进行排序，使得`dockerfile`更易阅读
```dockerfile
RUN apt-get update && apt-get install -y \
  bzr \
  cvs \
  git \
  mercurial \
  subversion \
  && rm -rf /var/lib/apt/lists/*
```

### LABEL
为镜像添加标签，记录更多信息
```dockerfile
# Set one or more individual labels
LABEL com.example.version="0.0.1-beta"
LABEL vendor1="ACME Incorporated"
LABEL vendor2=ZENITH\ Incorporated
LABEL com.example.release-date="2015-02-12"
LABEL com.example.version.is-production=""
```

### 上下文依赖的命令在同一层完成（一个RUN指令）
由于`docker build cache`机制，有上下文依赖的命令应该放到同一个`RUN`指令中执行。
示例：假设`dockerfile`如下
```dockerfile
...
RUN apt-get update
RUN apt-get install -y nginx
...
```
若用户想修改`dockerfile`给镜像中添加`python`程序，则如下所示：  

```dockerfile
...
RUN apt-get update
RUN apt-get install -y nginx python
...
```
因为`RUN apt-get update`这一层已有缓存，所以不会执行`apt-get update`命令，因此安装的`python`并不是最新版本。