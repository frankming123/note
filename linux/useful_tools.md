# 记一些有用的工具

<!-- TOC -->

- [记一些有用的工具](#记一些有用的工具)
    - [deepin-wine-tim](#deepin-wine-tim)
    - [frp](#frp)

<!-- /TOC -->

## deepin-wine-tim

腾讯偌大的产业却一直没有出qq或tim的linux版,这件事情一直都广受吐槽.不过幸好,deepin的程序员以及广大的编程爱好者们把qq和tim近乎完美的移植到了linux上

注意:arch安装tim需要在/etc/pacman.conf文件中取消对multilib的注释,然后更新数据库:pacman -Sy

## frp

frp的全称为fast reverse proxy,是一个可用于内网穿透的高性能的反向代理应用,支持tcp,udp,http和https协议

frp的github地址:<https://github.com/fatedier/frp/>