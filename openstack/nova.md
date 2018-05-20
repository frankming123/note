# nova

<!-- TOC -->

- [nova](#nova)
    - [nova架构](#nova架构)
        - [主要组件简介](#主要组件简介)
        - [物理部署方案](#物理部署方案)
        - [子服务协同工作的方式](#子服务协同工作的方式)
        - [OpenStack通用设计思路](#openstack通用设计思路)

<!-- /TOC -->

## nova架构

Nova是OpenStack最核心的服务,负责维护和管理云环境的计算资源

![nova](images/nova.png)

可以看出,Nova处于OpenStack的中心,其他组件都为Nova提供支持:

1. Glance为VM提供image
2. Cinder和Swift分别为VM提供块存储和对象存储
3. Neutron为VM提供网络连接

### 主要组件简介

Nova架构如下:

![nova_architecture](images/nova_architecture.png)

Nova的架构比较复杂,包含很多组件.这些组件以子服务(后台daemon进程)的形式运行,可以分为以下几类:

1. API

    **nova-api**
    接收和响应客户的API调用

2. Compute Core

    **nova-scheduler**
    虚机调度服务,负责决定在哪个计算节点上运行虚机
    **nova-compute**
    管理虚机的核心服务,通过调用Hypervisor API实现虚机生命周期管理
    **Hypervisor**
    计算节点上跑的虚拟机管理程序,虚机管理最底层的程序
    不同的虚拟化技术提供自己的Hypervisor
    常用的Hypervisor有KVM,Xen,VMWare等
    **nova-conductor**
    nova-compute经常需要更新数据库,比如更新虚机的状态
    出于安全和伸缩性的考虑,nova-compute并不会直接访问数据库,而是将这个任务委托给nova-conductor

3. Console Interface

    **nova-console**
    用户可以通过多种方式访问虚机的控制台:

    1. nova-novncproxy:基于Web浏览器的VNC访问
    2. nova-spicehtml5proxy:基于HTML5浏览器的SPICE访问
    3. nova-xvpnvncproxy:基于Java客户端的VNC访问

    **nova-consoleauth**
    负责对访问虚机控制台提供Token认证

    **nova-cert**
    提供x509证书支持

4. Database

    Nova会有一些数据需要存放到数据库中,一般使用MySQL
    数据库安装在控制节点上
    Nova使用命名为"nova"的数据库

5. Message Queue

    Nova众多的子服务之间需要相互协调和通信
    为解耦各个子服务,Nova通过Message Queue作为子服务的信息中转站

    OpenStack默认是使用RabbitMQ作为Message Queue

### 物理部署方案

对于Nova,它的子服务会部署在两类节点上:计算节点和控制节点

可以通过nova service-list命令查看nova-*子服务都分布在哪些节点上

![nova_service-list](images/nova_service-list.png)

### 子服务协同工作的方式

虚机创建流程:

![nova_create_VM](images/nova_create_VM.png)

1. 客户(可以是OpenStack最终用户,也可以是其他程序)向API(nova-api)发送请求:"帮我创建一个虚机"
2. API对请求做一些必要处理后,向Messaging(RabbitMQ)发送了一条消息:"让Scheduler创建一个虚机"
3. Scheduler(nova-scheduler)从Messaging获取到API发给它的消息,然后执行调度算法,从若干计算节点中选出节点A
4. Scheduler向Messaging发送了一条消息:"在计算节点A上创建这个虚机"
5. 计算节点A的Compute(nova-compute)从Messaging中获取到Scheduler发给它的消息,然后在本节点的Hypervisor上启动虚机
6. 在虚机创建的过程中,Compute如果需要查询或更新数据库信息,会通过Messaging向 Conductor(nova-conductor)发送消息,Conductor负责数据库访问

### OpenStack通用设计思路

- API前端服务

    每个OpenStack组件可能包含若干子服务,其中必定有一个API服务负责接收客户请求

    以Nova为例,nova-api作为Nova组件对外的唯一窗口,向客户暴露Nova能够提供的功能

    当客户需要执行虚机相关的操作,能且只能向nova-api发送REST请求

    设计API前端服务的好处:

    1. 对外提供统一接口,隐藏实现细节
    2. API提供REST标准调用服务,便于与第三方系统集成
    3. 可以通过运行多个API服务实例轻松实现API的高可用,比如运行多个nova-api进程

- Scheduler调度服务

    对于某项操作,如果有多个实体都能够完成任务,那么通常会有一个scheduler负责从这些实体中挑选出一个最合适的来执行操作

    以Nova为例,Nova有多个计算节点.当需要创建虚机时,nova-scheduler会根据计算节点当时的资源使用情况选择一个最合适的计算节点来运行虚机

- Worker工作服务

    调度服务只管分配任务,真正执行任务的是Worker工作服务

    在Nova中,这个Worker就是nova-compute

    将Scheduler和Worker从职能上进行划分使得OpenStack非常容易扩展:

    1. 当计算资源不够了无法创建虚机时,可以增加计算节点(增加Worker)
    2. 当客户的请求量太大调度不过来时,可以增加Scheduler

- Driver框架

    OpenStack采用基于Driver的框架

    Nova-compute为这些Hypervisor定义了统一的接口,hypervisor只需要实现这些接口,就可以以driver的形式即插即用到OpenStack中

    ![Nova_Hypervisor](images/Nova_Hypervisor.png)

    在nova-compute的配置文件/etc/nova/nova.conf中由compute_driver配置项指定该计算节点使用哪种Hypervisor的driver

    Glance有本地文件系统,Cinder,Ceph,Swift等Driver框架,Cinder,Neutron也有driver框架

- Messaging服务

    Messaging是nova-*子服务交互的中枢

    ![nova_messaging](images/nova_messaging.png)

    程序之间的调用通常分两种:同步调用和异步调用

    **同步调用**
    API直接调用Scheduler的接口就是同步调用
    其特点是API发出后需要一直等待,直到Scheduler完成对Compute的调度,将结果返回给API后API才能够继续做后面的工作

    **异步调用**
    API通过Messaging间接调用Scheduler就是异步调用
    其特点是API发出请求后不需要等待,直接返回,继续做后面的工作
    Scheduler从Messaging接收到请求后执行调度操作,完成后将结果也通过Messaging发送给API
    </br>
    在OpenStack这类分布式系统中,通常采用异步调用的方式,其好处是:

    1. 解耦各子服务

        子服务不需要知道其他服务在哪里运行,只需要发送消息给Messaging就能完成调用

    2. 提高性能

        异步调用使得调用者无需等待结果返回.这样可以继续执行更多的工作,提高系统总的吞吐量

    3. 提高伸缩性

        子服务可以根据需要进行扩展,启动更多的实例处理更多的请求,在提高可用性的同时也提高了整个系统的伸缩性,而且这种变化不会影响到其它子服务,也就是说变化对别人是透明的

- Database

    OpenStack各组件都需要维护自己的状态信息
    比如Nova中有虚机的规格,状态,这些信息都是在数据库中维护的
    每个OpenStack组件在MySQL中有自己的数据库