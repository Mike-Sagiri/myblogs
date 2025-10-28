---
layout: post
title: docker的可视化及远程连接时的可视化
date: 2025-10-28 12:47 +0800
categories: [技术经验, 远程服务]
tags: [ssh] # TAG 名称应始终为小写
---

## 本地主机上运行docker容器的可视化

docker的可视化，针对不同系统有不同的命令，但是Windows与Ubuntu其实是一致的，因为Windows的docker基本是基于WSL完成的。由于Ubuntu2204（及之后的版本）和WSL的底层显示程序都是Wayland，且都包含X11，想要实现docker容器内运行GUI程序，本质都是将X11和（或）Wayland接口映射到docker容器内来实现的。

### Ubuntu  
只使用X11进行可视化的命令如下所示：
```shell
docker run -it  \
 --name ros2-humble   \
 --env="DISPLAY"   \
 --volume="/tmp/.X11-unix:/tmp/.X11-unix:rw" \
 osrf/ros:humble-desktop-full
```
这里以运行一个ROS2镜像为例，`-e`将`DISPLAY`环境变量映射到容器内，`-v`将X11的本地套接字映射到容器内，这样容器就可以调用本机的X11 server来进行图形化显示了。

只使用Wayland可视化的命令如下所示：
```shell
docker run -it --network host --name ros2 \
 -e XDG_RUNTIME_DIR=/tmp \
 -e WAYLAND_DISPLAY=$WAYLAND_DISPLAY \
 -v $XDG_RUNTIME_DIR/$WAYLAND_DISPLAY:/tmp/$WAYLAND_DISPLAY \
 osrf/ros:humble-desktop-full
```
这里除了类似的环境变量和套接字映射以外，还增加了`--network host`，意思是让docker容器使用主机的网络（相同ip）。就Wayland显示而言，这是不必要的。

也可以将上面两个命令组合起来，就可以得到一个既能使用X11显示的，又能使用Wayland显示的容器了。对于gedit这种兼容两种显示接口的程序，会优先使用Wayland显示。

### Windows
对于Windows而言，基于WSL后端的docker与Ubuntu中的docker是类似的。只是命令需要稍微有所改变。由于显示所需的X11和Wayland都在WSL目录里，所以需要将所有路径前面加上`/run/desktop/mnt/host/wslg`，完整的命令如下所示：
```shell
docker run -it --rm --name foxy --net host \
 -v /run/desktop/mnt/host/wslg/.X11-unix:/tmp/.X11-unix \
 -v /run/desktop/mnt/host/wslg:/mnt/wslg \ 
 -e DISPLAY=:0 -e WAYLAND_DISPLAY=wayland-0 \ 
 -e XDG_RUNTIME_DIR=/mnt/wslg/runtime-dir \ 
 -e PULSE_SERVER=/mnt/wslg/PulseServer \ 
 osrf/ros:humble-desktop-full
```
这里路径是写死的，代表一种默认情况。因为你无法在Windows终端里访问WSL的环境变量。但是Windows的docker提供了能在WSL终端内访问的接口，可以在WSL内的终端运行，如下所示：
```shell
docker run -it --rm --net=host \ 
-e WAYLAND_DISPLAY -e XDG_RUNTIME_DIR \ 
-v $XDG_RUNTIME_DIR/$WAYLAND_DISPLAY:$XDG_RUNTIME_DIR/$WAYLAND_DISPLAY \ 
-e DISPLAY -v /tmp/.X11-unix:/tmp/.X11-unix \ 
-e PULSE_SERVER -v /mnt/wslg/PulseServer:/mnt/wslg/PulseServer \ 
osrf/ros:humble-desktop-full
```