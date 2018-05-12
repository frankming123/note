# 一、基础环境准备


## 1.1 系统

### 控制节点(2c4G,磁盘10G)

```
#主机名
hostnamectl set-hostname controller
```

### 计算节点(2c4G,磁盘10G)
```
#主机名
hostnamectl set-hostname compute
```

```

# 配置ip地址
[root@controller ~]# vim /etc/sysconfig/network-scripts/ifcfg-eth0 
TYPE=Ethernet
BOOTPROTO=none
DEVICE=eth0
ONBOOT=yes
IPADDR=192.168.56.21
PREFIX=24
GATEWAY=192.168.56.1

# 修改hosts文件
echo '192.168.56.21   controller
192.168.56.22   compute' >> /etc/hosts

# 停止网络管理和防火墙
systemctl stop NetworkManager firewalld
systemctl disable NetworkManager firewalld

# 禁用sshd使用DNS
sed -i 's/^#UseDNS yes /UseDNS no/'  /etc/ssh/sshd_config 

# 关闭selinux
setenforce 0
sed -i 's/SELINUX=enforcing/SELINUX=disabled/' /etc/selinux/config 

```

**配置节点互信**
```
[root@controller ~]# ssh-keygen (一直回车)
[root@controller ~]# ssh-copy-id -i .ssh/id_rsa.pub compute
```

## 1.2 配置yum源

**配置centos7 系统yum源**

参考帮助
[科大yum源centos](https://mirrors.ustc.edu.cn/help/centos.html)

```
rm -f  /etc/yum.repos.d/*
cat > /etc/yum.repos.d/centos7.repo  << EOF

[base]
name=CentOS-$releasever - Base
baseurl=https://mirrors.ustc.edu.cn/centos/7/os/$basearch/
gpgcheck=1
gpgkey=file:///etc/pki/rpm-gpg/RPM-GPG-KEY-CentOS-7

[updates]
name=CentOS-$releasever - Updates
baseurl=https://mirrors.ustc.edu.cn/centos/7/updates/$basearch/
gpgcheck=1
gpgkey=file:///etc/pki/rpm-gpg/RPM-GPG-KEY-CentOS-7

[extras]
name=CentOS-$releasever - Extras
baseurl=https://mirrors.ustc.edu.cn/centos/7/extras/$basearch/
gpgcheck=1
gpgkey=file:///etc/pki/rpm-gpg/RPM-GPG-KEY-CentOS-7
EOF
```

**安装openstack newton 源**
 
```
yum search  newton
yum install centos-release-openstack-newton.noarch -y


cat > /etc/yum.repos.d/CentOS-OpenStack-newton.repo << EOF
# CentOS-OpenStack-newton.repo

[centos-openstack-newton]
name=CentOS-7 - OpenStack newton
baseurl=http://mirrors.ustc.edu.cn/centos/7/cloud/$basearch/openstack-newton/
gpgcheck=1
enabled=1
gpgkey=file:///etc/pki/rpm-gpg/RPM-GPG-KEY-CentOS-SIG-Cloud

EOF

# 在控制节点上安装openstackclient
yum install python-openstackclient -y
```


## 1.3 配置时间同步


```
yum install chrony -y   

#控制节点作为ntp源
[root@controller ~]#vim /etc/chrony.conf
# Allow NTP client access from local network.
allow 192.168.0.0/16
# Serve time even if not synchronized to a time source.
local stratum 10

# 计算节点
[root@compute  ~]# grep -Ev "^$|^#"  /etc/chrony.conf 
server controller iburst



systemctl start chronyd
systemctl enable  chronyd

[root@compute ~]# chronyc  sources -v
210 Number of sources = 1
MS Name/IP address         Stratum Poll Reach LastRx Last sample               
===============================================================================
^* controller                   10   6    17     2  -1057ns[  -17us] +/- 1211us
```


## 1.4 安装 数据库 mariadb 

```
yum install mariadb mariadb-server python2-PyMySQL -y

systemctl enable mariadb.service
systemctl start mariadb.service

ss -anlt | grep 3306
netstat -anlt 

# 初始化mysql
mysql_secure_installation
```

## 1.5 安装消息队列Message queue
```
# 安装软件
yum install rabbitmq-server -y

systemctl enable rabbitmq-server.service
systemctl start rabbitmq-server.service

# 添加openstack用户
[root@controller ~]# rabbitmqctl add_user openstack RABBIT_PASS
Creating user "openstack" ...

# 设置openstackt用户权限
[root@controller ~]# rabbitmqctl set_permissions openstack ".*" ".*" ".*"
Setting permissions for user "openstack" in vhost "/" ...

```

## 1.6 安装 Memcached

```
yum install memcached python-memcached -y

# 配置Memcached监听管理口ip
vim  /etc/sysconfig/memcached
  OPTIONS="-l 192.168.56.21,::1"
 
systemctl enable memcached.service
systemctl start memcached.service
```


# 二、 keystone

## 概念
service
endpoints

## 2.1 配置keystone数据库
```
[root@controller ~]# mysql  -u root -pmysql123
# 创建数据库
CREATE DATABASE keystone;
# 配置keystone数据库用户和权限
GRANT ALL PRIVILEGES ON keystone.* TO 'keystone'@'localhost' \
  IDENTIFIED BY 'KEYSTONE_DBPASS';
GRANT ALL PRIVILEGES ON keystone.* TO 'keystone'@'%' \
  IDENTIFIED BY 'KEYSTONE_DBPASS';
# 刷新权限
flush PRIVILEGES;
exit;
```

## 2.2 安装keystone软件
```
 yum install openstack-keystone httpd mod_wsgi -y
```
## 2.3 配置 keystone


```

vim /etc/keystone/keystone.conf
[database]
connection = mysql+pymysql://keystone:KEYSTONE_DBPASS@controller/keystone
[token]
provider = fernet

## 根据配置文件生成数据库的表
su -s /bin/sh -c "keystone-manage db_sync" keystone
```


keystone的四种token为：UUID token、PKI token、PKIZ token和Fernet token。
(https://www.jianshu.com/p/e1b206ebda5b)


```
keystone-manage fernet_setup --keystone-user keystone --keystone-group keystone
keystone-manage credential_setup --keystone-user keystone --keystone-group keystone
```

```
keystone-manage bootstrap --bootstrap-password ADMIN_PASS \
  --bootstrap-admin-url http://controller:35357/v3/ \
  --bootstrap-internal-url http://controller:35357/v3/ \
  --bootstrap-public-url http://controller:5000/v3/ \
  --bootstrap-region-id RegionOne
```


ln -s /usr/share/keystone/wsgi-keystone.conf /etc/httpd/conf.d/
e db_sync
# systemctl enable httpd.service
# systemctl start httpd.service

export OS_USERNAME=admin
export OS_PASSWORD=ADMIN_PASS
export OS_PROJECT_NAME=admin
export OS_USER_DOMAIN_NAME=Default
export OS_PROJECT_DOMAIN_NAME=Default
export OS_AUTH_URL=http://controller:35357/v3
export OS_IDENTITY_API_VERSION=3