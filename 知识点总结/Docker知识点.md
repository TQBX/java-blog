## 什么是docker

答：开源的**容器化**平台，可以把应用及其依赖 打包 到 一个**轻量级、可移植**的容器中，从而在任何Docker运行的环境中实现一致的运行。（容器化、轻量级、可移植）

## docker和虚拟机的区别

虚拟机在硬件级别虚拟化，拥有独立的内核

docker容器在**os级别**虚拟化，**共享宿主机的内核**（更轻量级，启动快，资源占用少）

## docker镜像

> 镜像和容器的关系：类似于 类和实例的关系

是一个轻量级、只读的模板，用于创建Docker容器。它包含运行容器所需的代码、库、环境变量和配置文件。

怎么创建：`docker run -d -p 8080:80 --name my-container nginx`，

创建一个名为`my-container`的容器，使用Nginx镜像，并将容器的80端口映射到主机的8080端口。 `-d`标志表示容器将在后台运行。

## docker hub是什么

公共的容器镜像仓库，存放、分享、管理docker镜像，可以上传下载私有镜像。

## 常用命令

查看当前运行的docker容器 docker ps （-a 看所有）

停止 docker stop <container_id or name>   启动 docker start

进入正在运行的docker： docker exec -it <container_id or name> /bin/bash

删除镜像： docker rmi <image_id>  删除容器 docker rm <容器id>

查看日志：docker logs <容器id or name>

列出本地的镜像： docker images 拉取远程仓库镜像 docker pull

查看docker容器使用资源：docker stats

## docker的网络模式

怎么创建：docker network create --driver bridge my_bridge_network

1. `bridge`：默认的网络模式，容器与宿主机在同一个网络桥接中，可以通过端口映射来访问容器内的服务。
2. `host`：容器与宿主机共享网络命名空间，容器可以直接访问宿主机的网络接口，适用于对网络性能要求较高的场景。
3. `none`：容器不使用任何网络，需要手动配置网络。
4. `Container`：新创建的容器不会创建自己的网卡和配置自己的 IP，而是和一个指定的容器共享 IP、端口范围等。

容器可以通过Docker网络进行通信。在同一网络中的容器可以使用容器名称互相解析，实现容器间通信。

## docker的卷volume

是一种**持久化**存储数据的机制，独立与容易的生命周期。 还可以实现**数据共享**。

创建：docker volume create,  docker run + -v 将卷挂载到容器内部。

## Dockerfile

包含了构建Docker镜像所需的一系列指令和命令。

利用dockerfile创建镜像：docker bulid -t myimage：latest 

## Docker Compose

用于定义和运行多容器Docker应用程序的工具

怎么启动定义的服务：docker-compose up

