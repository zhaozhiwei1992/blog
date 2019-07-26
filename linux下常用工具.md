---
title: linux下常用工具
date: <2019-07-24 三>
tags: [linux, 工具]
---

arch linux (linux 命令很强大， 命令才是王道啊\~\~\~)
====================================================

1.  f.lux 根据经纬度调整屏幕亮度

2.  网易云音乐 （这段时间跪了, 打开白板 ）

3.  sublime text 偶尔当下记事本

4.  为知笔记 笔记软件， 因为跨平台我就支持了

5.  input

    ``` {.example}
    #export XIM=fcitx
    #export XIM_PROGRAM=fcitx
    #export GTK_IM_MODULE=fcitx
    #export QT_IM_MODULE=fcitx
    #export XMODIFIERS="@im=fcitx"

    #firefox keyi dazile
    export LC_ALL=zh_CN.UTF-8
    export GTK_IM_MODULE=fcitx
    export QT_IM_MODULE=fcitx
    export XMODIFIERS=“@im=fcitx”
    exec openbox-session
    #exec xfce4-session
    #exec gnome-session
    ```

6.  文件比较: meld vimdiff diff等

7.  文本编辑器 emacs vim

8.  webtorrenter-desktop 播放bt文件

9.  图形界面连接网络 sudo pacman -S network-manager-applet （需要启动
    systemctl start NetworkManager）

10. rescue time 有兴趣可以看看， 这个是记录你一天在干嘛的软件, 后台运行
    nohut rescuetime &(会在当前文件夹下生成nohup.out文件)

11. 跨平台文件传输 feige
    url:<http://www.feige360.com/ipmsg/download.html>

12. 锁屏 yaourt -S xlockmore

13. create~ap~ github下的一个项目， 共享热点必备

    ``` {.example}
    配置启动: nohup sudo create_ap wlp2s0 enp0s20u2u4 archer 1234567890 > /tmp/create_ap.log & tail -f /tmp/create_ap.log
    ```

14. 数据传输工具， 类似飞球， yaourt -s iptux,
    但是最新的0.75版本有问题， 不能扫描局域网用户,
    目前使用0.6.4版本进行源码编译安装

15. 使用zsh + oh my zsh

16. 开发工具使用idea, community

    -   插件:
        -   ideavim: vi模式， 用过都说好
        -   Key promoter: 快捷键菜鸡必备，比如我
        -   Markdown support

17. sqldeveloper安装

    -   oracle 限制， 需要登录下载，
        所以先到官网下载sqldeveloper18.nojar.zip
    -   pacman -S oracle-sqldeveloper
    -   安装过程中将刚刚下载的zip包放到其他pass文件同级的目录, 继续即可

18. postman curl 模拟请求

    -   curl -H \"Accept:application/json\" -H
        \"Content-Type:application/json\" -X POST -d \'{\"id\":1}\'
        <http://127.0.0.1:8080/user/save>

19. processon/graphviz 流程图

    -   c-c c-c编译执行dot文件

20. dropbox

    -   [离线版安装地址](https://www.dropbox.com/downloading?src=index&full=1)

21. ag搜索 the~silversearcher~

22. xml处理工具 xmlstarlet, jq处理json, shyaml处理yaml

23. 命令行剪贴板工具xclip

24. pandoc 文档转换工具

    -   <https://pandoc.org/MANUAL.html>
    -   pandoc linux下常用工具.org -o /tmp/linux常用工具.md

25. cairo-dock dock工具，样式挺好看，不过习惯了launcher运行, 不喜欢点,
    不使用

问题处理
========

1.  网易云音乐白板 : rm \`locate netease \|grep cache\` 删掉缓存即可

远程连接windows桌面
===================

➜ \~ rdesktop 192.168.2.144 -p lt\@180606 -u lt1 -g workarea
