# 记一些有用的工具

<!-- TOC -->

- [记一些有用的工具](#记一些有用的工具)
    - [ansible](#ansible)
    - [cobbler](#cobbler)
    - [deepin-wine-tim](#deepin-wine-tim)
    - [frp](#frp)
    - [vysor](#vysor)

<!-- /TOC -->

## ansible

## cobbler

## deepin-wine-tim

腾讯偌大的产业却一直没有出qq或tim的linux版,这件事情一直都广受吐槽.不过幸好,deepin的程序员以及广大的编程爱好者们把qq和tim近乎完美的移植到了linux上

注意:arch安装tim需要在/etc/pacman.conf文件中取消对multilib的注释,然后更新数据库:pacman -Sy

## frp

frp的全称为fast reverse proxy,是一个可用于内网穿透的高性能的反向代理应用,支持tcp,udp,http和https协议

frp的github地址:<https://github.com/fatedier/frp/>

## vysor

vysor是一款手机投屏软件,它可以将通过usb连接的手机(只能安卓?)的屏幕连接至电脑,且既有windows版,也有linux版,还有chrome app版

vysor有普通和pro两个版本,chrome app版可以通过修改其源文件使其伪装成pro版

    ]# vim /home/staight/.config/google-chrome/Default/Extensions/gidgenkbbabolejbgbpnhbimgjbffefm/1.9.3_0/uglify.js

查找"Thank"关键字,其上找到var e,t=!1,o=!1并改为var e,t=1,o=1,保存后再次打开vysor,则发现已伪装为pro版