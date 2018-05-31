# troubleshooting

## kde桌面进不去,黑屏只有一个鼠标

可能问题:安装nvidia驱动导致与核显驱动冲突

解决方案:卸载nvidia驱动,重启

## qemu-kvm开启嵌套虚拟化

1. 查看kvm_intel模块是否开启透传功能("Y"为开启,"N"为关闭)

        ]# systool -m kvm_intel -v  | grep -i nested:

2. 若没有开启,编辑vim /etc/modprobe.d/kvm-intel.conf文件

        options kvm_intel nested=1

3. 禁用,并启用kvm_intel模块

        ]# rmmod kvm_intel
        ]# modprobe kvm_intel
        ]# lsmod kvm_intel

4. 编辑虚拟机配置,启用嵌套虚拟机

        ]# virsh edit compute
         <cpu mode='custom' match='exact' check='partial'>
            <model fallback='allow'>Westmere</model>
            <feature policy='require' name='vmx'/>
        </cpu>

5. 开启虚拟机,查看是否已有嵌套虚拟化,完成

        ]# lsmod|grep kvm