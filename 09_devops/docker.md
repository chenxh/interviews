## 容器是如何工作的

![](https://raw.githubusercontent.com/chenxh/interviews/master/09_devops/imgs/docker.jpg "图片title")
Docker采用的是Client/Server架构。客户端向服务器发送请求，服务器负责构建、运行和分发容器。客户端和服务器可以运行在同一个Host上，客户端也可以通过socket或REST API与远程的服务器通信。

**Docker客户端:** Docker客户端是docker命令。通过docker我们可以方便地在Host上构建和运行容器.
**Docker服务器:** Docker daemon是服务器组件，以Linux后台服务的方式运行. 默认配置下，Docker daemon只能响应来自本地Host的客户端请求。如果要允许远程客户端请求，需要在配置文件中打开TCP监听.
**Docker镜像:** 可将Docker镜像看成只读模板，通过它可以创建Docker容器。
**Docker容器:** Docker容器就是Docker镜像的运行实例
**Registry:** Registry是存放Docker镜像的仓库.



## 镜像


**base镜像：** （1）不依赖其他镜像，从scratch构建；（2）其他镜像可以以之为基础进行扩展。通常都是各种Linux发行版的Docker镜像，比如Ubuntu、Debian、CentOS等

```
    docker pull centos 
```

* base镜像提供的是最小安装的Linux发行版.
* base镜像只是在用户空间与发行版一致，kernel版本与发行版是不同的。
* 容器只能使用Host的kernel，并且不能修改
* 新镜像是从base镜像一层一层叠加生成的。每安装一个软件，就在现有镜像的基础上增加一层.
* 当容器启动时，一个新的可写层被加载到镜像的顶部。
* 容器层保存的是镜像变化的部分，不会对镜像本身进行任何修改 .Copy-on-Write


## docker 存储
两种存放数据的资源：
* 由storage driver管理的镜像层和容器层。无状态应用，比如前端。 适合存储配置文件，部署包等。
* Data Volume。 Data Volume本质上是Docker Host文件系统中的目录或文件，能够直接被mount到容器的文件系统中。

**Data Volume**
* bind mount .  
```
# -v的格式为 <host path>:<container path>
docker run -d -p -v ~/htdocs:/usr/local/apache/htdocs httpd

```
* docker managed volume
docker managed volume与bind mount在使用上的最大区别是不需要指定mount源，指明mount point就行了

```
# -v告诉docker需要一个data volume，并将其mount到/usr/local/apache2/htdocs
# docker inspect命令 . 查看 Mounts 部分，可以找到 data volume 实际目录
docker run -d -p -v /usr/local/apache/htdocs httpd
```

## volume container 容器之间共享数据

volume container是专门为其他容器提供volume的容器。它提供的卷可以是bind mount，也可以是docker managed volume。

```
# volume container 容器命名为 vc_data
docker create --name vc_data -v ~/htdocs:/usr/local/apache/htdocs -v /other/useful/tools  busybox

# 其它容器使用 vc_data

docker run --name web1 -d -p 80 --volumes-from vc_data httpd

docker run --name web2 -d -p 80 --volumes-from vc_data httpd


```

## data-packed volume container 
将数据打包到镜像中，然后通过docker managed volume共享.

```
# dockfile
FROM busybox:latest
add htdocs /usr/local/apache2/htdocs

# VOLUME的作用与 -v等效，用来创建docker managed volume, mount point为/usr/local/apache2/htdocs
volume /usr/local/apache2/htdocs


# 构建镜像 datapacked
docker build -t datapacked .

# 新镜像创建data-packed volume container。 
docker create --name vc_data datapacked

# 使用 vc_data
docker run --name web2 -d -p 80 --volumes-from vc_data httpd


```



