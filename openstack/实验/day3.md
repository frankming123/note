# 网络存储Neutron

## Neutron核心功能

Neutron提供网络支持,包括二层交换,三层路由,负载均衡,防火墙和VPN等

1. 二层交换

    Linux Bridge和Open vSwitch.VxLAN和GRE

2. 三层路由

3. 负载均衡

4. 防火墙

## Neutron的基本概念:network

network包括local,flat,vlan,vxlan,gre

- subnet
- port
- ip

    固定ip
    浮动ip

## Neutron组件架构

Neutron Server
Plugin
Agent
Network provider
Queue
Database

## ML2 Core Plugin

Type Driver
Mechansim Driver