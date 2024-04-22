## 容器是如何工作的

![](https://raw.githubusercontent.com/chenxh/interviews/master/09_devops/imgs/docker.jpg "图片title")
Docker采用的是Client/Server架构。客户端向服务器发送请求，服务器负责构建、运行和分发容器。客户端和服务器可以运行在同一个Host上，客户端也可以通过socket或REST API与远程的服务器通信。

**Docker客户端:** Docker客户端是docker命令。通过docker我们可以方便地在Host上构建和运行容器.
**Docker服务器:** Docker daemon是服务器组件，以Linux后台服务的方式运行. 默认配置下，Docker daemon只能响应来自本地Host的客户端请求。如果要允许远程客户端请求，需要在配置文件中打开TCP监听.
**Docker镜像:** 可将Docker镜像看成只读模板，通过它可以创建Docker容器。
**Docker容器:** Docker容器就是Docker镜像的运行实例
**Registry:** Registry是存放Docker镜像的仓库.

