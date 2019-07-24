---
title: thinkpadt480安装archlinux实战
date: <2019-07-24 三>
tags: [archlinux, thinkpad, grub]
---

制作U盘启动盘
=============

-   下载archlinux镜像
-   下载powerison
-   将镜像刻录到U盘中

设置bios
========

-   如果已经安装了win8+系统,
    开机重启然后按f1，进入设置界面，加载启动项中将usb放到最前
-   保存后，插入启动盘重启，按f12， 选择启动项，选择archlinux

分区
====

-   工具:fdisk , 使用m指令查看支持的操作
-   重建分区表干掉所有分区 fdisk /dev/sda, g(是gpt分区表)
    w(这个指令后所做的配置就会生效)
-   创建分区 n{ /dev/sda1(efi), *dev/sda2(*), /dev/nvmxxx(/home)},
    我的分区是nvmxx(固态在前, 所以分区方案是/dev/nvm1(efi),
    *dev/nvmxx2(*)...)
-   格式化分区
    -   mkfs.vfat -F32 /dev/sda1
    -   mkfs.ext4 /dev/sda2 , 其他分区都使用ext4
-   分区挂载
    -   mount /dev/sda2 /mnt
    -   mount /dev/sda1 /mnt/boot/efi
    -   mount /dev/sda3 /mnt/home

网络连接
========

-   有网线的情况下直接跳过第二步
-   连接wifi
    -   首先查看网卡 ip link命令列出网卡id
    -   systemctl stop dhcpxxx.service 关闭现有网络
    -   wifi-menu -o wlp3s0,选择可连接网络连接
-   配置服务镜像
    -   如果是在中国， 就把mirrorlist中的china相关镜像提前,
        我这里只把163的镜像放最前面

安装系统
========

-   安装基本包
    -   pacman -sy 刷新系统更新文件
    -   安装基础系统及后续工具, base-devel也可以装好系统后再安装
        \#pacstrap -i /mnt base base-devel
-   配置fstab
    -   ``` {.example}
        # /dev/nvme0n1p2
        UUID=7bdc64d4-20ed-4073-bcef-dacefcee8e9a       /               ext4            rw,relatime     0 1

        # /dev/nvme0n1p1
        UUID=FAB8-E796          /boot/efi       vfat            rw,relatime,fmask=0022,dmask=0022,codepage=437,iocharset=iso8859-1,shortname=mixed,utf8,errors=remount-ro       0 2

        # /dev/nvme0n1p3
        UUID=48256cfb-3694-4102-85fb-33d9142f554c       /home           ext4            rw,relatime     0 2
        ```

    -   挂载机械硬盘/dev/sda1时需要注意，直接在fstab中配置方式导致开机无法启动，后查看官方文档，改为按需挂载,
        这种情况下正常启动，如果有外部对改磁盘进行了引用也会自动挂载

        UUID=6e3a49e9-131b-4003-b251-cc26eae107ca /mnt/d ext4
        noauto,x-systemd.automount 0 0

-   安装引导工具(这里一定要弄对，否则进不去系统， 严格顺序执行)
    后续操作全部在安装的系统里进行
    -   下面这个必须执行, 生成grub.cfg配置文件， 默认安装系统会自带一个

        ``` {.example}
        arch-chroot /mnt
        GRUB启动, UEFI的系统，要装grub-efi-x86_64和efibootmgr
        #pacman -S grub-efi-x86_64, efibootmgr
        把GRUB装到EFI分区里，这样就多一条GRUB启动项了。
        #grub-install --efi-directory=/boot/efi --bootloader-id=arch-grub --recheck
        生成grub配置文件(否则启动一致会报grub的错误)
        pacman -S os-prober
        #grub-install /dev/sda(根目录下, 如果出现grub缺少文件时候再执行)
        grub-mkconfig -o /boot/grub/grub.cfg
        ```

-   为了下次可以连无线最好安装下 pacman -S iw wpa~supplicant~ dialog
-   exit， 取消必要的挂载后重启， 系统已经安装完成

一些必要的优化及软件安装
========================

重启后进入自己安装好的系统
--------------------------

时区，编码
----------

-   生成localtime的软链就算设置时区了。 \#ln -s
    /usr/share/zoneinfo/Asia/Shanghai /etc/localtime
-   设置系统时间是当地时间，Linux默认是UTC时间，Windows默认是当地时间，改成一致。
    \#hwclock --localtime

编码
----

-   打开locale.gen文件，把en~US~.UTF-8, zh~CN~.UTF-8,
    zh~CN~.GBK前面的\#去掉。
-   运行locale-gen命令，重建编码表。

host设置
--------

-   设置电脑名，随你喜欢。

    -   \#echo arch-zhao \> /etc/hostname

-   设置hosts文件 vim /etc/hosts

    ``` {.example}
    127.0.0.1 localhost
    ::1       localhost
    127.0.1.1 arch-zhao.localdomain arch-zhao
    ```

图形界面
--------

-   安装xorg pacman -S xorg
-   触摸板驱动 pacman -S xf86-input-synaptics
-   根据自己显卡类型安装对应的驱动
    -   确认型号 lspci \| grep VGA

    -   官方提供驱动

        ``` {.example}
        通用———————————-xf86-video-vesa 
        intel———————————-xf86-video-intel 
        Geforce7+————————–xf86-video-nouveau 
        Geforce6/7————————-xf86-video-304xx
        ```

-   安装字体(图形界面需要) pacman -S ttf-dejavu wqy-microhei
-   安装yaourt vim /etc/pacman.conf \[archlinuxcn\] SigLevel=Never
    Server = <https://mirrors.ustc.edu.cn/archlinuxcn/$arch>
-   binutils, yaourt 需要进行二进制编译(安装了base-devel后不需要)
-   安装桌面环境 xde(好看), xfce(轻量), openbox(定制化)
    -   kde桌面安装 pacman -S plasma
    -   文件管理器 pacman -S dolphin
    -   命令行 pacman -S konsole
    -   kde工具套件 pacman -S kde-applications(这个我没安装)
-   启动网络管理 systemctl enable NetworkManager , 安装前端管理工具
    network-manage-applet, 或者pacman -S plasma-nm
-   安装slim (启动管理器， 用习惯了，类似还有很多), 执行systemctl enable
    slim(允许开机自启)
-   .xinintrc配置 cp *etc/X11/xinit/xinitrc \~*.xinitrc vim \~/.xinitrc
    加入exec startkde 启动kde
-   安装openssh sudo pacman -S openssh (非必须)
-   安装chromium
-   reflector 镜像刷新工具, 可以扫描最快的镜像提前
    1.  备份原镜像 cp /etc/pacman.d/mirrorlist
        /etc/pacman.d/mirrorlist.bak
    2.  查找最快200个源并写入到镜像文件 reflector --verbose -l 200 -p
        http --sort rate --save /etc/pacman.d/mirrorlist
-   后续软件按个人需要安装
