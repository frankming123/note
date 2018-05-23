# Docker命令

<!-- TOC -->

- [Docker命令](#docker命令)
    - [docker命令](#docker命令)
        - [image相关](#image相关)
        - [container相关](#container相关)
        - [network相关](#network相关)
        - [volume相关](#volume相关)
    - [Dockerfile格式](#dockerfile格式)

<!-- /TOC -->

## docker命令

docker info:显示系统层面的信息

docker inspect [OPTIONS] NAME|ID [NAME|ID...]:展示一个或多个docker对象(容器,镜像,网络,存储等)的信息

### image相关

docker build [OPTIONS] PATH | URL | -:从一个Dockerfile构建镜像

docker commit [OPTIONS] CONTAINER [REPOSITORY[:TAG]]:从一个改动的容器创建一个新的镜像

docker history [OPTIONS] IMAGE:查看一个镜像的构建历史,相当于展示了镜像的分层结构

docker images [OPTIONS] [REPOSITORY[:TAG]]:列出镜像

docker pull [OPTIONS] NAME[:TAG|@DIGEST]:从一个registry上下载一个镜像或者仓库

docker push [OPTIONS] NAME[:TAG]:把一个镜像或者仓库推送至registry上

docker rmi [OPTIONS] IMAGE [IMAGE...]:移除一个或多个镜像

docker search [OPTIONS] TERM:搜索Docker Hub中的镜像

docker tag SOURCE_IMAGE[:TAG] TARGET_IMAGE[:TAG]:为image打上标签

### container相关

docker attach [OPTIONS] CONTAINER:attach到容器启动命令的终端

- OPTIONS

    -t name[:tag]:为新镜像命名
    -f Dockerfile_path:指定Dockerfile的位置
    --no-cache:构建镜像时不使用缓存

- PATH:指明build context的位置

docker cp [OPTIONS] CONTAINER:SRC_PATH DEST_PATH|- / SRC_PATH|- CONTAINER:DEST_PATH:在一个容器和本地文件系统间拷贝文件或文件夹

docker create [OPTIONS] IMAGE [COMMAND] [ARG...]:创建一个新的容器,命令格式与run相同

docker exec [OPTIONS] CONTAINER COMMAND [ARG...]:在一个正在运行的容器里运行一个命令

- OPTIONS

    -d:在后台运行命令
    -it:以交互模式打开一个伪终端,并执行命令

docker kill [OPTIONS] CONTAINER [CONTAINER...]:强制关闭一个或多个正在运行的容器

- OPTIONS

    -s string:向容器发送的信号(默认是SIGKILL)

docker logs [OPTIONS] CONTAINER:得到一个容器的日志

- OPTIONS

    -f:跟踪日志输出

docker restart [OPTIONS] CONTAINER [CONTAINER...]:重启一个或多个容器,相当于一次执行stop和start

- OPTIONS

    -t int:等待容器停止的秒数,若超过这个时间,则强制停止它(默认10秒)

docker rm [OPTIONS] CONTAINER [CONTAINER...]:移除一个或多个容器

- OPTIONS

    -f:强制移除一个正在使用的容器(使用SIGKILL)
    -v:移除与容器相关联的volume

docker run [OPTIONS] IMAGE [COMMAND] [ARG...]:在一个新的容器中运行一个命令,相当于先create再start

- OPTIONS

    -it:以交互模式进入容器,并打开终端
    -d:使容器在后台运行,并打印容器ID
    --name string:为容器分配一个名称
    -h string:容器的hostname
    --network string:连接一个容器至一个网络(默认是bridge网络)
    --ip string:指定分配一个ip(默认随机)
    -p, --publish list:将一个容器的端口映射至host.例如80:80
    -v, --volume list:绑定挂载一个volume.格式为[host path:]container path[:OPTIONS].如果没有host path,docker会自动创建;OPTIONS可选ro只读
    </br>
    -m bytes:限制容器的内存使用量.如果不指定
    --memory-swap则默认使用相同容量的交换内存
    --memory-swap bytes:限制容器的内存和交换内存总共的使用量(-1指不限制)
    -c int:设置CPU shares(是一个相对的权重值,默认1024)
    --cpus decimal:设置可使用CPU的数量
    --blkio-weight uint16:设置容器block IO的权重,数值在10至1000之间,0为不允许(默认是0)
    --device-read-bps list:限制从一个设备读取的速率(字节每秒),默认为[].例如:/dev/sda:30MB
    --device-read-iops list:限制从一个设备读取的次数(次数每秒)
    --device-write-bps list:限制从一个设备写入的速率
    --device-write-iops list:限制从一个设备写入的次数
    </br>
    --restart string:当一个容器退出时,重启的策略(默认是no)

    - string

        - no:不自动重启
        - on-failure[:max-retries]:容器进程退出代码非0则重启容器.可选一个最大重启次数的限制
        - always:始终重启容器
        - unless-stopped:始终重启容器,除非该容器被手动关闭

docker stop [OPTIONS] CONTAINER [CONTAINER...]:停止一个或多个正在运行的容器

- OPTIONS:

    -t int:等待容器停止的秒数,若超过这个时间,则强制停止它(默认10秒)

### network相关

docker network connect [OPTIONS] NETWORK CONTAINER:将一个容器连接至一个网络

docker network create [OPTIONS] NETWORK:创建一个网络

- OPTIONS

    -d, --driver string:管理网络的驱动(默认是bridge)
    --subnet strings:指定网络的网段(x.x.x.x/x格式)
    --gateway strings:指定网络的网关
    -o, --opt map:视驱动而定的可选的选项,例如:parent=eth0,指定使用的网络接口为eth0

docker network inspect NETWORK [NETWORK...]:展示一个或多个网络的细节

docker network ls:列出网络

### volume相关

docker volume inspect [OPTIONS] VOLUME [VOLUME...]:展示一个或多个volume的详细信息

docker volume ls:列出volume

docker volume rm [OPTIONS] VOLUME [VOLUME...]:移除一个或多个volume

docker volume prune [OPTIONS]:移除所有不用的本地volume

## Dockerfile格式

FROM image_name:从名为image_name的镜像上构建Dockerfile(scratch表示从零开始构建)

RUN command:在临时容器中运行command指令

CMD command:在容器启动时运行指定的命令(多个CMD指令只有最后一个生效,如果docker run之后有参数则会被替换)

ENTRYPOINT command:设置容器启动时运行的命令(多个ENTRYPOINT指令只有最后一个生效,CMD或docker run之后的参数会被当做参数传递给ENTRYPOINT)

COPY src dest:把Host指定context的src文件拷贝到临时容器的dest

MAINTAINER author:设置镜像的作者,可以使任意字符串

ADD src dest:与COPY类似.不过如果src是归档文件,则会被自动解压到dest

ENV key value:设置环境变量,可被后面的指令使用

EXPOSE port [port ...]:指定容器中的进程会监听某个端口

VOLUME path:将文件或目录声明为volume,与-v等效

WORKDIR path:为后面的RUN,CMD,ENTRYPOINT,ADD或COPY指令设置镜像中的当前工作目录

"#"开头的为注释