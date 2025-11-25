---
layout: post
title: 在命令行配置linux的clash
date: 2025-11-25 15:05 +0800
categories: [技术经验, 网络配置]
tags: [linux, clash] # TAG 名称应始终为小写
---

无论是远程服务器的需要，还是Ubuntu2004对图形化clash的弱支持，在命令行配置clash核心都是重要的解决方案。本文将以最新仍在维护的[mihomo内核](https://github.com/MetaCubeX/mihomo/tree/Meta)为例，讲述如何利用命令行配置该内核。

## 配置mihomo

### 下载二进制应用
首先，下载release中对应架构的**二进制**程序。使用deb或者其他安装或许也行，但是作者没有验证过。  

### 下载必备运行库
第一次运行该二进制程序，应该会自动下载一些必备运行库。但是，经常由于网络或者其他原因而失败。所以，为了避免此类情况，作者提前下载好所需的运行库。访问[运行库网站](https://github.com/MetaCubeX/meta-rules-dat)，下载下载地址中所有的文件，放到`~/.config/mihomo/`文件路径下，该路径应该在运行过mihomo之后已经创建完成。

### 给予网络权限
此时，需要运行`sudo setcap cap_net_bind_service,cap_net_admin=+ep /你的二进制文件路径`，来给予网络权限。

### 提供订阅文件
与可视化的程序不同，mihomo内核无法使用订阅地址，需要你显式下载订阅配置文件。你可以直接在浏览器里访问你的订阅地址，会下载一个订阅文件，重命名为config.yaml，放到`~/.config/mihomo/`下，或者将内容复制到`.config/mihomo`里已有的config.yaml文件中。

### 修改订阅文件监听端口
许多Windows上的订阅文件会将dns端口绑定在53，这在linux上是不行的。因此，可以将配置文件里的`dns`条目的`listen`改为1053等端口。

### 使用网页控制内核
你可以在配置文件里配置外部控制`external-controller: 127.0.0.1:9090`，该配置意味着你可以访问本地的9090端口来控制mihomo内核行为。如果你需要远程访问它，可以使用vscode remote ssh来端口转发。或者设置为`0.0.0.0:9090`这样直接使用远程主机ip就可以了。  

远程控制的网页网址：[https://yacd.metacubex.one/#/proxies](https://yacd.metacubex.one/#/proxies)，里面可以设置需要控制的ip。

## 配置clash-core
由于该内核已经停止维护，故不推荐。但是如果实在需要，可以在这个[备份站](https://github.com/Kuingsmile/clash-core/releases/tag/1.18)找到备份。  

clash core的使用比较简单。不需要配置额外的权限。默认配置文件在`~/.config/clash`里。将你的订阅文件替换里面的config.yaml即可。如果遇到`Contry.mmdm`文件下载失败，可以手动下载，放到clash core目录和`~/.config/clash`目录下即可。

> 无论是mihomo还是clash core，一经配置，对所有用户都是有用的。
{: .prompt-warning }