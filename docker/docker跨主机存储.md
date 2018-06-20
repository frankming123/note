# docker跨主机存储

<!-- TOC -->

- [docker跨主机存储](#docker跨主机存储)
    - [使用跨主机存储的目的](#使用跨主机存储的目的)
    - [跨主机存储原理](#跨主机存储原理)
    - [Rex-Ray](#rex-ray)
        - [Rex-Ray的安装和配置](#rex-ray的安装和配置)

<!-- /TOC -->

## 使用跨主机存储的目的

容器分为有状态容器和无状态容器

无状态容器在运行时不需要保存数据,比如工具类容器

有状态容器需要保存数据,而且数据会发生变化,比如数据库容器

对于有状态的容器,可以通过data volume存储容器的状态.实现存储的高可用可以通过定期备份数据,但更好的解决方案是专门的storage provider.这样即使Host挂了,也可以立刻在其他可用Host上启动相同镜像的容器,挂载之前使用的volume,这样就不会有数据丢失

## 跨主机存储原理

Docker通过volume driver实现跨主机管理data volume

任何一个data volume都是由driver管理的,创建volume时如果不特别指定,将使用local类型的driver.如果要支持跨主机的volume,则需要使用第三方driver

目前已经有很多可用的driver,比如Azure File Storage的driver,使用GlusterFS的driver等等,完整列表可参考:<https://docs.docker.com/engine/extend/legacy_plugins/#volume-plugins>

## Rex-Ray

下面将以Rex-Ray driver示例,原因是:

1. Rex-Ray是开源的,而且社区活跃
2. 支持多种backend,VirtualBox的Virtual Media,Amazon EBS,Ceph RBD,OpenStack Cinder等
3. 支持多种操作系统,Ubuntu,CentOS,RHEL和CoreOS
4. 支持多种容器编排引擎,Docker Swarm,Kubernetes和Mesos
5. Rex-Ray安装使用方法非常简单

### Rex-Ray的安装和配置

