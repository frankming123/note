# docker跨主机网络

<!-- TOC -->

- [docker跨主机网络](#docker跨主机网络)
    - [跨主机网络概述](#跨主机网络概述)
    - [overlay](#overlay)
        - [实验环境](#实验环境)
        - [创建overlay网络](#创建overlay网络)
        - [在overlay中运行容器](#在overlay中运行容器)
        - [overlay实现跨主机通信](#overlay实现跨主机通信)
        - [overlay的隔离性](#overlay的隔离性)
    - [macvlan](#macvlan)
        - [macvlan实验环境](#macvlan实验环境)
        - [创建macvlan网络](#创建macvlan网络)
        - [macvlan网络结构分析](#macvlan网络结构分析)
        - [用sub-interface实现多macvlan网络](#用sub-interface实现多macvlan网络)
    - [flannel](#flannel)

<!-- /TOC -->

## 跨主机网络概述

跨主机网络方案包括:

1. docker原生的overlay和macvlan
2. 第三方方案:常用的包括flannel,weave和calico

众多网络方案的实现标准:libnetwork & CNM

libnetwork是docker容器网络库,最核心的内容是其定义的Container Network Model(CNM),这个模型对容器网络进行了抽象,由以下三个组件组成:

1. Sandbox

    Sandbox是容器的网络栈,包含容器的interface,路由表和DNS设置.Linux Network Namespace是Sandbox的标准实现.Sandbox可以包含来自不同Network的 Endpoint

2. Endpoint

    Endpoint的作用是将Sandbox接入Network.Endpoint的典型实现是veth pair,后面我们会举例.一个Endpoint只能属于一个网络,也只能属于一个Sandbox

3. Network

    Network包含一组 Endpoint,同一 Network的 Endpoint可以直接通信.Network的实现可以是Linux Bridge,VLAN等

    ![CNM](images/CNM.png)

libnetwork CNM定义了docker容器的网络模型,按照该模型开发出的driver就能与docker daemon协同工作,实现容器网络.docker原生的driver包括none,bridge,overlay和macvlan,第三方driver包括flannel,weave,calico等

## overlay

为支持容器跨主机通信,Docker提供了overlay driver,使用户可以创建基于VxLAN的overlay网络.VxLAN可将二层数据封装到UDP进行传输,VxLAN提供与VLAN相同的以太网二层服务,但是拥有更强的扩展性和灵活性

overlay网络需要一个key-value数据库用于保存网络状态信息,包括Network,Endpoint,IP等.Consul,Etcd和ZooKeeper都是Docker支持的Key-value软件

### 实验环境

host1:192.168.122.10,docker-machine
host2:192.168.122.20,machine
host3:192.168.122.30,machine

1. 在host1上安装Consul(使用容器,通过<http://192.168.122.10:8500>访问Consul)

        ]# docker run -d -p 8500:8500 -h consul --name consul progrium/consul -server -bootstrap

2. 修改host2和host3上docker daemon的配置文件

        ]# vim /etc/systemd/system/docker.service.d/10-machine.conf

        ExecStart= ... --cluster-store=consul://192.168.122.10:8500 --cluster-advertise=eth0:2376

    --cluster-store指定consul的地址
    --cluster-advertise告知consul自己的连接地址

3. 重启docker daemon,访问Consul可以看到host2和host3自动被注册到Consul数据库中

### 创建overlay网络

在host2中创建overlay网络ov_net1

    ]# docker network create -d overlay ov_net1

-d overlay指定driver为overlay

docker network ls查看当前网络,可以看到ov_net1的SCOPE为global,而其他网络为local

在host2上查看网络,也能看到ov_net1.这是因为host1将overlay网络信息存入了consul,host2从consul读取到新网络的数据,之后ov_net的任何变化都会同步到host1和host

查看ov_net1的详细信息

    ]# docker network inspect ov_net1
    ...
    "Name": "ov_net1",
    "Scope": "global",
    "Driver": "overlay",
    "IPAM": {
        "Driver": "default",
        "Options": {},
        "Config": [
            {
                "Subnet": "10.0.0.0/24",
                "Gateway": "10.0.0.1"
            }
        ]
    },
    ...

### 在overlay中运行容器

运行一个busybox容器并连接到ov_net1

    ]# docker run -itd --name bbox1 --network ov_net1 busybox

查看容器的网络配置

    ]# docker exec bbox1 ip r
    default via 172.18.0.1 dev eth1
    10.0.0.0/24 dev eth0 scope link  src 10.0.0.2
    172.18.0.0/16 dev eth1 scope link  src 172.18.0.2

bbox1有两个网络接口eth0和eth1.eth0的IP为10.0.0.2,连接的是overlay网络ov_net1;eth1的IP为172.18.0.2,连接的是bridge网络docker_gwbridge,目的是为所有连接到overlay网络的容器提供访问外网的能力

bbox1可以通过docker_gwbridge访问外网

    ]# docker exec bbox1 ping -c1 baidu.com
    PING baidu.com (220.181.57.216): 56 data bytes
    64 bytes from 220.181.57.216: seq=0 ttl=47 time=62.958 ms

如果外网要访问容器,可通过主机端口映射

### overlay实现跨主机通信

在host3中同样运行busybox

    ]# docker run -itd --name bbox2 --network ov_net1 busybox

bbox2的IP是10.0.0.3,可以直接ping bbox1(需要安装libvirt)

    ]# docker exec bbox2 ping -c1 bbox1
    PING bbox1 (10.0.0.2): 56 data bytes
    64 bytes from 10.0.0.2: seq=0 ttl=64 time=0.385 ms

可见overlay网络中的容器可以直接通信,同时docker也实现了DNS服务

overlay网络的具体实现:

docker会为每个overlay网络创建一个独立的network namespace,其中会有一个linux bridge br0,endpoint还是由veth pair实现,一端连接到容器中(即eth0),另一端连接到namespace的br0上

br0除了连接所有的endpoint,还会连接一个vxlan设备,用于与其他host建立vxlan tunnel.容器之间的数据就是通过这个tunnel通信的

逻辑网络拓扑结构如图所示:

![overlay](images/overlay.png)

执行过ln -s /var/run/docker/netns /var/run/netns后,可以通过ip netns命令查看overlay网络的namespace.两个host上的namespace相同

查看namespace中br0上的设备

    ]# ip netns exec 1-4d1f8bd07f brctl show
    bridge name     bridge id               STP enabled     interfaces
    br0             8000.b280b5114139       no              veth0
                                                            vxlan0

查看vxlan0设备的具体配置信息可知此overlay使用的VNI(VxLAN ID)为256

    ]# ip netns exec 1-4d1f8bd07f ip -d l show vxlan0
    8: vxlan0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1450 qdisc noqueue master br0 state UNKNOWN mode DEFAULT group defat
        link/ether b2:80:b5:11:41:39 brd ff:ff:ff:ff:ff:ff link-netnsid 0 promiscuity 1
        vxlan id 256 srcport 0 0 dstport 4789 proxy

### overlay的隔离性

不同的overlay网络是相互隔离的

创建第二个overlay网络ov_net2

    ]# docker network create -d overlay ov_net2

查看ov_net2的详细信息,可以发现网段和ov_net1不同

    ]# docker network inspect ov_net2
    "Config": [
        {
            "Subnet": "10.0.1.0/24",
            "Gateway": "10.0.1.1"
        }
    ]

当然,创建网络时也可以指定IP空间

    ]# docker network create -d overlay --subnet 10.1.1.0/24 ov_net3

## macvlan

macvlan本身是linux kernel模块,其功能是允许在同一个物理网卡上配置多个MAC地址,即多个interface,每个interface可以配置自己的IP

macvlan本质上是一种虚拟化技术,因而Docker可以使用macvlan实现容器网络

macvlan的最大优点是性能极好.相比其他实现,macvlan不需要创建Linux Bridge,而是直接通过以太interface连接到物理网络

### macvlan实验环境

在host2和host3上,为保证多个mac地址的网络包都可以从eth0通过,需要打开网卡的混杂模式

    ]# ip link set eth0 promisc on

### 创建macvlan网络

在host2和host3上创建macvlan网络mac_net1

    ]# docker network create -d macvlan --subnet 172.16.10.0/24 --gateway 172.16.10.1 -o parent=eth0 mac_net1

macvlan网络是local网络,为了保证跨主机能够通信,用户需要自己管理IP subnet(docker选择的IP subnet可能不同)

与其他网络不同,docker不会为macvlan创建网关,这里的网关应该是真实存在的,否则容器无法路由

在host1中运行容器bbox1并连接到mac_net1(host2运行bbox2)

    ]# docker run -idt --name bbox1 --ip=172.16.10.10 --network mac_net1 busybox

自动分配IP可能会造成IP冲突,最好手动指定

验证bbox1和bbox2的连通性,可以ping通(但无法解析主机名)

    ]# docker exec bbox1 ping -c1 172.16.10.20
    PING 172.16.10.20 (172.16.10.20): 56 data bytes
    64 bytes from 172.16.10.20: seq=0 ttl=64 time=0.550 ms

可见docker并没有为macvlan提供DNS服务

### macvlan网络结构分析

macvlan不依赖Linux Bridge,brctl show可以确认没有创建新的bridge

查看容器bbox1的网络设备

    ]# docker exec bbox1 ip link
    1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue qlen 1000
        link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    7: eth0@if2: <BROADCAST,MULTICAST,UP,LOWER_UP,M-DOWN> mtu 1500 qdisc noqueue
        link/ether 02:42:ac:10:0a:0a brd ff:ff:ff:ff:ff:ff

注意eth0后面的@if2,这表明该interface有一个对应的interface,且全局编号是4.可以发现host2的eth0的编号就是4

可见容器的eth0@if2就是host2中eth0通过macvlan虚拟出来的interface.容器的interface直接与主机的网卡连接,这种方案使得容器无需通过NAT和端口映射就能与外网直接通信(只要有网关),在网络上与其他独立主机没有区别

此时,容器的eth0@if2和host2中的eth0可以认为是同一张网卡的不同namespace实现,并没有明显的主次之分.容器的IP可以在外界存活

![macvlan](images/macvlan.png)

### 用sub-interface实现多macvlan网络

macvlan会独占主机的网卡,也就是说一个网卡只能创建一个macvlan网络,否则会报错.不过好在macvlan也可以连接到sub-interface

VLAN是sub-inteface的一种实现方式,它可以将物理的二层网络划分成多达4094个逻辑网络,这些逻辑网络在二层上隔离

1. 创建VLAN(host2,host3)

        ]# nmcli c add ifname eth0.10 con-name eth0.10 type vlan dev eth0 id 10
        ]# nmcli c add ifname eth0.20 con-name eth0.20 type vlan dev eth0 id 20

2. 创建macvlan网络(host2,host3)

        ]# docker network create -d macvlan --subnet=192.168.10.0/24 --gateway=192.168.10.1 -o parent=eth0
        .10 mac_net10
        ]#docker network create -d macvlan --subnet=192.168.20.0/24 --gateway=192.168.20.1 -o parent=eth0
        .20 mac_net20

3. 运行容器(host2,host3需要修改IP)

        ]# docker run -itd --name bbox1 --ip=192.168.10.10 --network mac_net10 busybox
        ]# docker run -itd --name bbox2 --ip=192.168.20.10 --network mac_net20 busybox

    测试ping,可以发现同一macvlan网络可以通信,不同macvlan网络不能通信

    不同macvlan网络在二层隔离,但在三层上不隔离,可以通过路由转发的方法使它们通信

    ![macvlan_subinterface](images/macvlan_subinterface.png)

4. 在host1上创建VLAN10和VLAN20,并设置网关IP

        ]# nmcli c add ifname eth0.10 con-name eth0.10 type vlan dev eth0 id 10 ip4 192.168.10.1/24
        ]# nmcli c add ifname eth0.20 con-name eth0.20 type vlan dev eth0 id 20 ip4 192.168.20.1/24

5. 添加iptables规则,转发不同VLAN的数据包

        ]# iptables -t nat -A POSTROUTING -o eth0.10 -j MASQUERADE
        ]# iptables -t nat -A POSTROUTING -o eth0.20 -j MASQUERADE

        ]# iptables -A FORWARD -i eth0.10 -o eth0.20 -j ACCEPT
        ]# iptables -A FORWARD -i eth0.20 -o eth0.10 -j ACCEPT

    测试ping,发现不同macvlan网段可以ping通

    ![macvlan_subinterface_diff](images/macvlan_subinterface_diff.png)

## flannel