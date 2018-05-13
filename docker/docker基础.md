# docker基础

<!-- TOC -->

- [docker基础](#docker基础)
    - [安装docker](#安装docker)
    - [容器概念](#容器概念)
        - [什么是容器](#什么是容器)
        - [为什么需要容器](#为什么需要容器)
        - [容器怎样工作](#容器怎样工作)
    - [Docker镜像](#docker镜像)
        - [镜像的内部结构](#镜像的内部结构)
        - [base镜像](#base镜像)
        - [镜像的分层结构](#镜像的分层结构)
    - [Docker命令](#docker命令)

<!-- /TOC -->

## 安装docker

Docker分为开源免费的CE(Community Edition)版本和收费的EE(Enterprise Edition)版本

Docker支持几乎所有的Linux发行版,也支持Mac和Windows.各操作系统的安装方法可以访问<https://docs.docker.com/install/>

Fedora上的安装方法:

1. 配置软件源

        ]# dnf config-manager \
        --add-repo \
        https://download.docker.com/linux/fedora/docker-ce.repo

2. 安装docker

        ]# dnf update
        ]# dnf install docker-ce

3. 开启国内镜像加速

    由于docker hub的服务器在国外,下载镜像非常慢,可以使用国内的镜像服务进行加速,例如阿里云

    ![alispeed](alispeed.png)

4. 启用docker daemon

        ]# systemctl start docker
        ]# systemctl enable docker

## 容器概念

### 什么是容器

容器是一种轻量级,可移植,自包含的软件打包技术,使应用程序可以在几乎任何地方以相同的方式运行

容器由应用程序和其依赖的库或其他软件组成

容器在Host操作系统的用户空间中运行,与操作系统的其他进程隔离

### 为什么需要容器

现阶段软件开发需要考虑依赖库,部署环境,服务迁移等多种挑战

容器的优势:

- 对于开发人员 - Build Once, Run Anywhere

    开发人员只需为应用创建一次运行环境,然后打包成容器便可在其他机器上运行.另外,容器环境与所在的Host环境是隔离的,就像虚拟机一样,但更快更简单

- 对于运维人员 - Configure Once, Run Anything

    只需要配置好标准的runtime环境,服务器就可以运行任何容器.这使得运维人员的工作变得更高效,一致和可重复.容器消除了开发,测试,生产环境的不一致性

### 容器怎样工作

Docker的核心组件包括:

1. Docker客户端-Client
2. Docker 服务器 - Docker daemon
3. Docker 镜像 - Image
4. Registry
5. Docker 容器 - Container

Docker架构:

![docker_arch](images/docker_arch.png)

Docker采用的是Client/Server架构.客户端向服务器发送请求,服务器负责构建,运行和分发容器.客户端和服务器可以运行在同一个Host上,客户端也可以通过socket或REST API与远程的服务器通信

- Docker客户端

    最常用的Docker客户端是docker命令.通过docker可以方便的在Host上构建和运行容器

- Docker服务器

    Docker daemon是服务器组件,以Linux后台服务的方式运行

    Docker daemon运行在Docker host上,负责创建,运行,监控容器,构建,存储镜像

- Docker镜像

    可将Docker镜像看成只读模板,通过它可以创建Docker容器

    镜像有多种生成方法:

    1. 可以从无到有开始创建镜像
    2. 也可以下载并使用别人创建好的现成的镜像
    3. 还可以在现有镜像上创建新的镜像

- Docker容器

    Docker容器就是Docker镜像的运行实例

    用户可以通过CLI(docker)或是API启动,停止,移动或删除容器.可以这么认为,对于应用软件,镜像是软件生命周期的构建和打包阶段,而容器则是启动和运行阶段

- Registry

    Registry是存放Docker镜像的仓库,Registry分私有和公有两种

## Docker镜像

### 镜像的内部结构

Dockerfile是镜像的描述文件,定义了如何构建Docker镜像.Dockerfile的语法简洁且可读性强

hello-world的Dockerfile内容:

    FROM scratch //此镜像是白手起家,从0开始构建
    COPY hello / //将文件"hello"复制到镜像的根目录
    CMD ["/hello"] //容器启动时,执行/hello

### base镜像

base镜像有两层含义:

1. 不依赖其他镜像,从scratch构建
2. 其他镜像可以为之基础进行扩展

通常能称作base镜像的都是各种Linux发行版的Docker镜像,比如Ubuntu,Debian,CentOS等等

对于一个base镜像,底层直接用Host的kernel(bootfs),自己只需要提供rootfs.因此base镜像的内核与Host内核一致

![bootfs_rootfs](images/bootfs_rootfs.png)

base镜像提供的是最小安装的Linux发行版

CentOS镜像的Dockerfile的内容:

    FROM scratch
    ADD centos-7-docker.tar.xz / //自动解压镜像的tar包到/目录下
    CMD ["/bin/bash"]

### 镜像的分层结构

## Docker命令

docker pull [OPTIONS] NAME[:TAG|@DIGEST]:从一个registry上下载一个镜像或者仓库

docker images [OPTIONS] [REPOSITORY[:TAG]]:列出镜像

docker run [OPTIONS] IMAGE [COMMAND] [ARG...]:在一个新的容器中运行一个命令