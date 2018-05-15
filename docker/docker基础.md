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
        - [容器层与镜像层](#容器层与镜像层)
        - [构建镜像](#构建镜像)
        - [镜像的缓存特性](#镜像的缓存特性)
        - [调试Dockerfile](#调试dockerfile)
        - [镜像命名](#镜像命名)
    - [docker容器](#docker容器)
        - [如何运行容器](#如何运行容器)
        - [进入容器的方法](#进入容器的方法)
        - [容器常用操作](#容器常用操作)
        - [容器操作图解](#容器操作图解)
        - [限制容器的资源](#限制容器的资源)
        - [实现容器的底层技术](#实现容器的底层技术)
    - [Docker网络](#docker网络)
        - [Docker网络类型](#docker网络类型)
        - [容器间通信的方式](#容器间通信的方式)
        - [容器与外部世界通信](#容器与外部世界通信)
    - [Docker存储](#docker存储)
        - [两类存储资源](#两类存储资源)
        - [共享数据](#共享数据)
        - [volume生命周期管理](#volume生命周期管理)

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

    ![alispeed](images/alispeed.png)

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

docker支持通过扩展现有镜像,创建新的镜像

Dockerfile示例:

    FROM debian
    RUN apt-get install emacs
    RUN apt-get install apache2
    CMD ["/bin/bash"]

![Dockerfile](images/Dockerfile.png)

通过分层结构,多个镜像可以共享资源

### 容器层与镜像层

当容器启动时,一个新的可写层被加载到镜像的顶部.这一层通常被称作容器层,容器层之下都叫镜像层

![container_image_layer](images/container_image_layer.png)

所有对容器的改动都只会发生在容器层中

只有容器层是可写的,容器层下面的所有镜像层都是只读的

镜像层数量可能会很多,所有镜像层会联合在一起组成一个统一的文件系统.如果不同层中有一个相同路径的文件,比如/a,上层的/a会覆盖下层的/a,也就是说用户只能访问到上层中的文件/a.在容器层中,用户看到的是一个叠加之后的文件系统

1. 添加文件

    在容器中创建文件时,新文件被添加到容器层中

2. 读取文件

    在容器中读取某个文件时,Docker会从上往下依次在各镜像层中查找此文件.一旦找到,立即将其复制到容器层,然后打开并读入内存

3. 修改文件

    在容器中修改已存在的文件时,Docker会从上往下依次在各镜像层中查找此文件.一旦找到,立即将其复制到容器层,然后修改之

4. 删除文件

    在容器中删除文件时,Docker也是从上往下依次在镜像层中查找此文件.找到后,会在容器层中记录下此删除操作

只有当需要修改时才复制一份数据,这种特性被称作Copy-on-Write.可见,容器层保存的是镜像变化的部分,不会对镜像本身进行任何修改

### 构建镜像

Docker提供了两种构建镜像的方法:

1. docker commit命令
2. Dockerfile构建镜像

- docker commit

    docker commit是创建新镜像最直观的方法,其过程包括:

    1. 运行容器
    2. 修改容器
    3. 将容器保存为新的镜像

    示例:

        ]# docker run -it ubuntu
        ]# apt-get install -y vim
        ]# docker commit silly_goldberg ubuntu-with-vi

- Dockerfile构建镜像

    Dockerfile示例:

        FROM ubuntu
        RUN apt-get update && apt-get install -y vim

    使用Dockerfile构建:

        ]# ls
        Dockerfile
        ]# docker build -t ubuntu-with-vi-dockerfile .

    构建后可以通过docker images查看新的镜像

###　镜像的缓存特性

Docker会缓存已有镜像的镜像层.构建新镜像时,如果某镜像层已经存在,就直接使用,无需重新创建

Dockerfile中每一个指令都会创建一个镜像层,上层是依赖于下层的.无论什么时候,只要某一层发生变化,其上面所有层的缓存都会失效

### 调试Dockerfile

Dockerfile构建镜像的中途可能会出错,此时Docker所创建的临时容器依旧保留着

如果想要调试Dockerfile,可以进入临时容器中查看信息

### 镜像命名

一个特定镜像的名字由两部分组成:repository和tag

如果执行docker buil时没有使用指定tag,会使用默认值latest

tag除了标识一个镜像外,没有什么特殊的含义

## docker容器

### 如何运行容器

运行容器一般使用docker run命令

可用三种方式指定容器启动时执行的命令:

1. CMD指令
2. ENDPOINT指令
3. 在docker run命令行中指定

容器在执行完指令后会自动退出.如果想要容器保持running,则需要一直执行的命令

### 进入容器的方法

有两种方法可以进入容器:attach和exec

- docker attach

    通过docker attach可以attach到容器启动命令的终端

- docker exec

    在容器中打开新的终端,并连接至该终端

attch与exec主要区别:

1. attach直接进入容器启动命令的终端,不会启动新的进程
2. exec则是在容器中打开新的终端,并且可以启动新的进程

attach命令可以查看启动命令的输出,当然docker logs也可以做到这一点

如果没有什么特殊情况,一般使用exec命令

### 容器常用操作

- stop/start/restart容器

    通过docker stop可以停止运行的容器

    stop本质上是向该进程发送一个SIGTERM信号.如果想强制停止容器,可以使用docker kill命令,其作用是向容器进程发送SIGKILL信号

    对于处于停止状态的容器,可以通过docker start重新启动.start会保留容器的第一次启动时的所有参数

    docker restart可以重启容器,相当于依次执行stop和start

    对于服务类容器,还可以在其运行时添加--restart=always参数使其在退出时自动重启

- pause/unpause容器

    pause可以使容器处于暂停状态,其内存和硬盘数据依旧保持存在.而stop命令则无法保持其内存数据

    unpause用来恢复处于暂停状态的容器

- 删除容器

    使用docker一段使用后,host上可能会有大量已经退出了的容器

    这些容器依然会占用host上的存储资源,如果确认不会再重启此类容器,可以通过docker rm删除

    移除所有容器:

        ]# docker rm -v $(docker ps -aq -f status=exited)

### 容器操作图解

![container_perform](images/container_perform.png)

### 限制容器的资源

- 内存限额

    在创建docker容器时,可以加上-m和--memory-swap参数来控制容器内存的使用量

    官方的progrium/stress镜像非常适合测试容器的计算,存储资源

- CPU限制

    Docker可以通过-c或--cpu-shares设置容器使用CPU的权重.如果不指定,默认值是1024

    与内存限额不同,通过-c设置的cpu share并不是CPU资源的绝对数量,而是一个相对的权重值.某个容器最终能分配到的CPU资源取决于它的cpu share占所有容器cpu share总和的比例

- Block IO限制

    Block IO指的是磁盘的读写,docker可通过设置权重,限制bps和iops的方式控制容器读写磁盘的带宽

    --blkio-weight可用来改变容器block IO的优先级(10-1000)

    bps是byte per second,每秒读写的数据量;iops是io per second,每秒IO的次数

    可以通过以下参数控制容器的bps和iops:

    - --device-read-bps,限制读某个设备的bps
    - --device-write-bps,限制写某个设备的bps
    - --device-read-iops,限制读某个设备的iops
    - --device-write-iops,限制写某个设备的iops

### 实现容器的底层技术

cgroup和namespace是最重要的两种技术.cgroup实现资源限额,namespace实现资源隔离

- cgroup

    cgroup全称Control Group.Linux操作系统通过cgroup可以设置进程使用CPU,内存和IO资源的限额

    在/sys/fs/cgroup中可以找到docker对容器的限额

- namespace

    namespace管理着host中全局唯一的资源,并可以让每个容器都觉得只有自己在使用它.换句话说,namespace实现了容器间资源的隔离

    Linux使用了六种namespace,分别对应六种资源:Mount,UTS,IPC,PID,Network和User

## Docker网络

### Docker网络类型

Docker安装时会自动在host上创建3个网络:bridge,host,none,可用docker network ls命令查看

- none网络

    none网络即没有网络,挂在这个网络下的容器除了lo,没有其他任何网卡

    容器创建时可以使用--network=none指定使用none网络

    一般对安全性要求高且不需要联网的应用可以使用none网络

- host网络

    连接到host网络的容器共享Docker host的网络栈,容器的网络配置与host完全一样

    可以通过--network=host指定使用host网络

    对网络传输效率有较高要求的应用可以使用host网络,不过要考虑端口冲突问题

- bridge网络

    bridge网络是创建容器时的默认网络

    Docker安装时会创建一个命名为docker0的linux bridge,如果不指定--network,创建的容器默认都会挂到docker0上

    容器的网络拓扑:

    ![bridge](images/bridge.png)

- 自定义容器网络

    除了none,host,bridge这三个自动创建的网络,用户也可以根据需要创建user-defined网络

    Docker提供三种user-defined网络驱动:bridge,overlay 和macvlan

    可以通过bridge驱动创建类似默认的bridge网络

        ]# docker network create -d bridge --subnet 172.22.16.0/24 --gateway 172.22.16.1 my_net

    容器创建时可以用--network参数指定新的网络.不过只有使用--subnet创建的网络才允许容器创建时使用--ip参数指定一个静态ip

    ![bridge2](images/bridge2.png)

### 容器间通信的方式

- IP通信

    由于Docker设置iptables的drop策略,两个容器要能通信,必须要有属于同一个网络的网卡

    可以通过--network以及docker network connect将容器加入到指定网络

- Docker DNS Server

    通过ip访问容器可能无法确定ip,不够灵活.可以通过docker自带的DNS服务解决

    docker daemon实现了一个内嵌的DNS server,使容器可以直接通过"容器名"通信.方法很简单,只要在启动时用--name为容器命名就可以了.不过只能在user-defined网络中使用

        ]# docker run -it --name=busybox1 --network=my_net busybox
        ]# docker run -it --name=busybox2 --network=my_net busybox
        ]# ping -c1 busybox1

- joined容器

    joined容器非常特别,它可以使两个或多个容器共享一个网络栈,共享网卡和配置信息,joined容器之间可以通过127.0.0.1直接通信

    首先需要先创建一个容器:

        ]# docker run -d --name=httpd httpd
        ]# docker run -it --network=container:httpd busybox
        ]# wget 127.0.0.1 && cat index.html
        <html><body><h1>It works!</h1></body></html>

    joined容器非常适合不同容器中的程序通过loopback高效通信,例如web server和app server,也适合容器网络流量的监控

### 容器与外部世界通信

- 容器访问外部世界

    默认情况下,容器可以访问外网

    在bridge网络下,容器使用的是私有地址(172.17.0.0/16),容器通过NAT连通外网

        ]# iptables -t nat -S
        -A POSTROUTING -s 172.17.0.0/16 ! -o docker0 -j MASQUERADE

    从docker内部到外网的过程:

    ![snat](images/snat.png)

    1. busybox发送ping包:172.17.0.2 > www.bing.com
    2. docker0收到包,发现是发送到外网的,交给NAT处理
    3. NAT将源地址换成enp0s3的IP:10.0.2.15 > www.bing.com
    4. ping包从enp0s3发送出去,到达www.bing.com

- 外部世界访问容器

    外部世界访问容器的方法是端口映射

    docker可将容器对外提供服务的端口映射到host的某个端口,外网通过该端口访问容器.容器启动时通过-p参数映射端口

    以0.0.0.0:32773->80/tcp为例分析整个过程:

    ![dnat](images/dnat.png)

    1. docker-proxy监听host的32773端口
    2. 当curl访问10.0.2.15:32773时,docker-proxy转发给容器172.17.0.2:80
    3. httpd容器响应请求并返回结果

## Docker存储

### 两类存储资源

Docker为容器提供了两类存放数据的资源:

1. 由storage driver管理的镜像层和容器层
2. Data Volume

- storage driver

    容器由最上面一个可写的容器层,以及若干只读的镜像层组成,容器的数据就存放在这些层中.这样的分层结构最大的特性是Copy-on-Write

    分层结构使镜像和容器的创建,共享以及分发变得非常高效,而这些都要归功于Docker storage driver.正是storage driver实现了多层数据的堆叠并为用户提供一个单一的合并之后的统一视图

    Docker支持多种storage driver,有AUFS,Device Mapper,Btrfs,OverlayFS,VFS和ZFS.它们都能实现分层的架构,同时又有各自的特性.Docker优先使用Linux发行版默认的storage driver

    可以运行docker info查看系统的默认driver

    对于那些无状态的应用,直接使用storage driver存放数据是一个很好的选择,因为它们没有需要持久化的数据

- data volume

    Data Volume本质上是Docker Host文件系统中的目录或文件,能够直接被mount到容器的文件系统中.Data Volume有以下特点:

    1. Data Volume是目录或文件,而非没有格式化的磁盘(块设备)
    2. 容器可以读写volume中的数据
    3. volume数据可以被永久的保存,即使使用它的容器已经销毁

    在具体的使用上,docker提供了两种类型的volume:bind mount和docker managed volume

    - bind mount

        bind mount是将host上已存在的目录或文件mount到容器

            docker run -d -p 80:80 -v ~/htdocs:/usr/local/apache2/htdocs:ro httpd

        bind mount默认可读写,容器销毁对bind mount没有影响

        bind mount的使用直观高效,但是由于需要指定host的特定路径,限制了容器的可移植性

    - docker managed volume

        docker managed volume与bind mount在使用上的最大区别是不需要指定mount源,指明mount point就行了

            docker run -d -p 80:80 -v /usr/local/apache2/htdocs:ro httpd

        data volume的具体位置可以通过docker inspect命令找到,默认在/var/lib/docker/volumes目录下

        docker managed volume的创建过程:

        1. 容器启动时,简单的告诉docker"我需要一个volume存放数据,帮我mount到目录/abc"
        2. docker在/var/lib/docker/volumes中生成一个随机目录作为mount源
        3. 如果/abc已经存在,则将数据复制到mount源
        4. 将volume mount到/abc

        除了docker inspect,也可以通过docker volume命令查看volume的信息

    两种data volume的对比:

    1. 相同点:两者都是host文件系统的某个路径
    2. 不同点:

    | | bind mount | docker managed volume |
    :-: | :-: | :-:
    | volume 位置 | 可任意指定 | /var/lib/docker/volumes/... |
    | 对已有mount point影响 | 隐藏并替换为 volume | 原有数据复制到volume |
    | 是否支持单个文件 | 支持 | 不支持，只能是目录 |
    | 权限控制 | 可设置为只读,默认为读写权限 | 无控制，均为读写权限 |
    | 移植性 | 移植性弱,与 host path 绑定 | 移植性强,无需指定host目录 |

### 共享数据

- 容器与host共享数据

    对于bind mount,直接将要共享的目录mount到容器

    对于docker managed volume,由于volume在容器启动时才生成,所以需要将共享数据拷贝到volume中

        ]# docker cp /home/staight/code inspiring_dubinsky:/home

- 容器间共享数据

    1. 将数据放在bind mount里

        将共享数据放在bind mount中,然后将其mount到多个容器.例如web server集群使用相同的html文件

    2. 使用volume container

        volume container是专门为其他容器提供 volume的容器.它提供的卷可以是bind mount,也可以是docker managed volume

        创建一个volume container,由于不需要运行,所以使用create:

            ]# docker create --name=vc_container -v /home/staight/code:/home -v /usr/local/apache2/htdocs busybox

        其他容器可以通过--volume-from使用vc_container这个volume container

            ]# docker run --name=web1 -d --volumes-from=vc_container -p 80:80 httpd
            ]# docker run --name=web2 -d --volumes-from=vc_container -p 81:80 httpd
            ]# curl 127.0.0.1:80
            ]# curl 127.0.0.1:81

        volume container的优点:

        1. 与bind mount相比,不必为每一个容器指定host path,所有path都在volume container中定义好了,容器只需与volume container关联,实现了容器与host的解耦

        2. 使用volume container的容器其mount point是一致的,有利于配置的规范和标准化,但也带来一定的局限,使用时需要综合考虑

    3. data-packed volume container

        data-packed volume container可以将数据完全放到volume container中,同时又能与其他容器共享

        data-packed volume container的原理是将数据打包到镜像中,然后通过docker managed volume共享

        首先使用Dockerfile构建镜像

            FROM busybox
            ADD index.html /usr/local/apache2/htdocs/
            VOLUME /usr/local/apache2/htdocs //因为该目录就是ADD添加的目录,所以会将已有的数据拷贝到volume中

        build新镜像data

            ]# docker build -t data .

        使用新镜像创建data-packed volume container

            ]# docker create --name=vc_data 9d15e05e0f31

        因为在Dockerfile中已经使用了VOLUME指令,这里就不需要指定volume的mount point了.启动httpd容器并使用 data-packed volume container

            ]# docker run -d --volumes-from=vc_data -p 80:80 httpd
            ]# curl 127.0.0.1

        data-packed volume container是自包含的,不依赖host提供数据,具有很强的移植性,非常适合只使用静态数据的场景,比如应用的配置信息,web server的静态文件等

### volume生命周期管理

- 备份

    因为volume实际上是host文件系统中的目录和文件,所以volume的备份实际上是对文件系统的备份.例如/myregistry

- 恢复

    volume的恢复也很简单,如果数据损坏了,直接用之前备份的数据拷贝到/myregistry就可以了

- 迁移

    如果我们想使用更新版本的Registry,这就涉及到数据迁移.方法是:

    1. docker stop当前Registry容器
    2. 启动新版本容器并mount原有volume

- 销毁

    可以删除不再需要的volume,但一定要确保知道自己正在做什么,volume删除后数据是找不回来的

    docker不会销毁bind mount,删除数据的工作只能由host负责.对于docker managed volume,在执行docker rm删除容器时可以带上-v参数,docker会将容器使用到的volume一并删除,但前提是没有其他容器mount该volume,目的是保护数据

    如果删除容器时没有带-v参数,就会产生孤儿volume.这样的话可以使用docker volume子命令对其进行批量删除