---
lang: zh-CN
title: linux命令操作android手机
date: 2022-09-19 
tags: [linux, android, 命令控制手机, adb]
---

# 目的

通过一台linux机器操作android手机做一些常用的操作

复杂的操作都是由简单操作开始的, 可以自行发掘

# 环境

``` example
笔记本: thinkpad t480

操作系统: archlinux

adb版本: 31.0.3p2-android-tools

手机: 小米手机 muui13.09
```

# 操作步骤

## 笔记本安装adb环境

``` example
sudo pacman -S android-tools
```

### 测试

``` example
adb --version

Android Debug Bridge version 1.0.41
Version 31.0.3p2-android-tools
Installed as /usr/bin/adb
```

## 手机打开usb调试开关

设置----全部参数-----狂戳miui版本，这样就打开开发者模式了。

设置-----更多设置------开发者选项---开启开发者选项。

需要开启的按钮有:USB调试、USB安装、USB调试（安全设置）、关闭启动MIUI优化。

## 连接手机

通过usb连接手机后,
需要在小米手机上面的USB的用途上面选择传输文件或者传输照片（除了仅充电就行\~）

测试连接: 查看连接设备 adb devices, 正常输出说明连接成功,
usb连通后也可以用wifi连接 :D

## 使用命令操作手机拍照

### 启动相机

``` example
adb shell am start -a android.media.action.STILL_IMAGE_CAMERA
```

### camera键 拍照

``` example
adb shell input keyevent 27 
```

### back键 暂退相机

``` example
adb shell input keyevent 4
```

### 注

这里只是个简单操作, 并且相机的启动入口基本是固定的,
如果是其它的app需要先获取到应用的入口, 可以自行搜索或联系我

对于按键操作, 最好是网上搜索一份按键编码对应表
