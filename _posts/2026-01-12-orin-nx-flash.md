---
layout: post
title: 英伟达Orin NX固件刷写教程
date: 2026-01-12 13:42 +0800
categories: [技术经验, 英伟达, Jetson Orin]
tags: [nvidia, jetson orin nx] # TAG 名称应始终为小写
---

## 背景
由于需要使用orin nx进行开发，全盘克隆某台orin的系统盘（SSD），并迁移到其他orin上。但是由于orin出场的固件版本不一样，导致迁移后无法启动。此外，如果购买某些提供商的车架，希望使用他们的系统，但是自己的orin的话，也需要在刷写他们的系统之前，先刷orin的固件，把固件刷成一致的，否则也会失败。

## 基础知识
首先需要了解orin的架构。orin虽然运行Ubuntu系统，但是首先它是arm架构的，同时，它的系统不仅是Ubuntu，而是jetpack——包含了英伟达特殊驱动和固件的系统包。这个系统包中的固件，必须通过英伟达提供的程序刷写，只更换固态硬盘是无效的。只有固件一致的orin，才能通过更换固态硬盘使用不同的系统镜像。

## 固件刷写
如果只是需要英伟达官方的jetpack，而不需要更换自己的系统的话，这一步既是刷写固件，也是刷写系统。
### 降级刷写
如果是将固件降级，比如6.0->5.0，需要额外注意jetpack的兼容性。比如，对于orin nx而言，最低支持的jetpack是5.2，但是有些orin镜像就是基于5.1.1的，怎么办？英伟达提供了5.1.1的补丁包。需要事先将补丁包修补到安装包里。
### 升级刷写
如果是将固件升级，比如5.0->6.0，基本没有额外的要求。常见坑都放在后面说。

现在正式开始刷写固件。

首先准备一台Ubuntu2004的电脑，虚拟机也行。按照链接所示教程进行刷写：[hiwonder刷写教程](https://drive.google.com/drive/folders/1s8_v4E1skUjaOV5M4OIHDgUw2pqiMOQk)，看第二个pdf：Flashing Firmware Using SDK Manager Tool。请注意，如果你正确地连接了usb，那么不需要4.6，有关检查IP的部分。直接使用usb就能成功刷系统。再者注意，如果使用虚拟机，需要全程值守，在刷写时orin可能重启，重启后需要在虚拟机里选择将orin连接到虚拟机里。

这个教程写的十分详细。我只补充需要注意的，会踩坑的几点：
1. 务必使用USB-A连typec的线。也就是，一端typec连接orin，另一端USB-A（就是那个长方体的，最常见的USB口）连接你的电脑或虚拟机。作者使用typec连typec的线，就会失败，失败的毫无理由。
2. 务必接触主机Ubuntu的防火墙：[这篇知乎和评论已经踩了坑](https://zhuanlan.zhihu.com/p/697333833)
3. 如果使用vmware虚拟机，请将usb控制器的兼容性改成3.0。不然可能会报：没有合适的控制器之类的错误。
4. 如果你的orin上电不自动开机，请检查是不是用杜邦线，把DISABLE AUTO ON两个引脚给连起来了。
5. 日志里输出Flash is success就成功了，有时候sdk manager在这一步之后会卡住，但是不影响orin的刷写。

### 降级刷写补丁安装教程

首先，禁用ubuntu系统的usb自动休眠：
```bash
sudo su
echo -1 > /sys/module/usbcore/parameters/autosuspend
```

然后，[安装补丁：](https://developer.nvidia.com/embedded/jetson-linux-r3531)，页面比较复杂，下载下面两个文件即可：
[补丁1](https://developer.nvidia.com/downloads/embedded/l4t/r35_release_v3.1/overlay_xusb_35.3.1.tbz2/)
[补丁2](https://developer.nvidia.com/downloads/embedded/l4t/r35_release_v3.1/overlay_pcn210361_pcn210100_r35.3.1.tbz2)
以上两个文件，解压到linux_for_tegra

之后，使用sdk manager安装即可。可以先勾选，download now install later，下载好jetpack 5.1，然后安装补丁之后再进行install。