---
title: docker的使用
date: 2019-11-12 16:09:50
tags:
- Linux
- Docker
categories:
- Linux
---

# docker 的使用

## Docker 和 虚拟机的区别:

- 实现资源隔离的方式不同:
  - 虚拟机:
    利用独立的 guest os, 并利用 Hypervisor 虚拟化硬件设备. 应用及其依赖在各自的 Guest Os 上运行
  - Docker:
    应用运行在各自的容器之中. 容器只是宿主机中的一个应用进程. Docker 利用 Linux 内核特性实现的隔离, 和宿主机共享资源, 共用一个内核. 使用到的 Linux 内核特性包括: 1. 使用 Namespace 机制实现系统环境的隔离. Namespaces 允许一个进程以及它的子进程从共享的宿主机内核资源里获得一个仅自己可见的隔离区域 2. 使用 CGroups 机制限制每个容器进程能够消耗资源的最大值
- 隔离性:
  虚拟机的隔离性强于 docker, docker 属于进程级别的隔离, 虚拟机可以实现系统级别的隔离. 由此也造成了虚拟机的安全性要强于 docker
- 性能:
  docker 容器和内核交互，几乎没有性能损耗，性能优于虚拟机通过 Hypervisor 层与内核层的虚拟化

## Docker Toolbox

因为我目前使用的 Windows 系统, 所以使用 Docker Toolbox for windows 版本的. 虽然有更好的 Docker for Windows 版本, 但是, 那个只支持 Windows 企业版和专业版. 我的系统是 Windows 家庭版.

## Docker Toolbox 添加镜像加速

可能是因为墙的关系, 安装之后从官方源 pull 容器时, 总是 timeout, 后来使用加速器可以解决这个问题.

可以使用阿里云和 DaoCloud 镜像加速, 感觉 DaoCloud 效果更好

具体的配置修改, 参考:
[http://guide.daocloud.io/dcs/daocloud-9153151.html#docker-toolbox]()

## image, container

### image:

镜像. 是一个包含有文件系统的面向 Docker 引擎的**只读模板**. 可以是从远端拉取的, 也可以是自己制作的

### container:

容器. 类似于一个轻量级的沙盒，可以将其看作一个极简的 Linux 系统环境（包括 root 权限、进程空间、用户空间和网络空间等），以及运行在其中的应用程序. 容器是镜像创建的应用实例. **注意：镜像本身是只读的，容器从镜像启动时，Docker 在镜像的上层创建一个可写层，镜像本身不变。**

## 常用操作

### [https://www.runoob.com/docker/docker-container-usage.html]()

很多常用操作都可以在上面的链接上找到, 我就不写了. 下面介绍一些操作中需要注意的地方

### docker container run

```shell
docker run -it --name string ubuntu:latest /bin/bash
```

参数说明:

- -i: 交互式操作。
- -t: 终端。
- ubuntu:latest: 名称为 ubuntu, tag 为 latest 的镜像。
- /bin/bash：放在镜像名后的是命令，这里我们希望有个交互式 Shell，因此用的是 /bin/bash。
- --name: 重命名. 最好重命名, 如果没有这个参数, docker 会自动为容器创建一个很奇怪的名字

**注意**
**image 每一次 docker run 之后, 都会启动一个不同的 container**, 并且创建出的不同的 container 都会是 image 的初始状态. 如果想要在上一次的 container 的修改上继续操作, 需要执行 container start 命令

### 将容器转化为一个镜像

```shell
docker commit -m="has update" -a="runoob" e218edb10161 runoob/ubuntu:v2
```

- -m: 提交的描述信息
- -a: 指定镜像作者
- e218edb10161：容器 ID
- runoob/ubuntu:v2: 指定要创建的目标镜像名, runoob 为用户名, v2 为 tag

**注意** 如果要制作的 image 打包上传到自己 docker hub, ' / ' 前面的用户名需要和自己 docker hub 的用户名保持一致

### 使用 Dockerfile 创建镜像

没试过... 以后有机会再写上
