---
layout: post
title: 详解wsl与docker宿主(host)模式下ROS2 DDS中间件的问题
date: 2026-05-31 09:52 +0800
categories: [技术经验, ros2开发]
tags: [ros2, dds, network, wsl, docker] # TAG 名称应始终为小写
---

众所周知，ros2有两个主要的中间件，fastdds和cyclonedds。作者[之前的文章](../ros2-dds-config)提到可以定义它们走的网卡。但是，文末提到了：
如果wsl或者docker使用宿主机网络，而非NAT桥接，已知会有如下的问题：

1. FastDDS有时候无法将话题发送，并且难以稳定复现。
2. CycloneDDS可以正常发送，但是会内存泄漏。

现在，经过作者反复研究，终于将该问题解决。下面将描述问题发生环境和解决方案。

参考网址：
1. [wsl network issue](https://github.com/microsoft/WSL/issues/10855)
2. [cyclonedds shared memory config for ros2 humble](https://github.com/ros2/rmw_cyclonedds/blob/humble/shared_memory_support.md)
3. [cyclonedds runtime config](https://github.com/eclipse-cyclonedds/cyclonedds#run-time-configuration)
4. [fastdds默认使用多个网络接口](https://fast-dds.docs.eprosima.com/en/latest/fastdds/transport/interfaces.html)
5. [cyclonedds默认随机选择一个网络接口](https://cyclonedds.io/docs/cyclonedds/latest/config/network_interfaces.html)

## 环境背景
作者使用最新的wsl内核，安装Ubuntu22.04和ros2 humble，并且在wsl内安装linux版本的docker engine，而并非使用Windows下的docker desktop。wsl使用mirror网络，docker使用`--net=host`。保证Windows、wsl和docker网络一致。

> Windows下的docker desktop的`--net=host`使用的是docker设置里docker wsl后端的网，而非Windows或者wsl的网络，所以多播起来很麻烦。
{: .prompt-info }

## 问题复现

### 同主机
这里同主机指的是wsl与wsl，docker与docker（同一容器）。
同主机时，由于[wsl的网络问题](https://github.com/microsoft/WSL/issues/10855)的存在，`ros2 daemon`会无法自动启动。此时，FastDDS无法正常工作，表现为同主机下无法互相发现。手动`ros2 daemon start`后则解决。而CycloneDDS无此问题。始终可以正常互相发现。而CycloneDDS发，FastDDS接收，也没有问题。而FastDDS发CycloneDDS接收，则无法发现。

但是，CycloneDDS的内存泄露问题，同主机下就会显现。表现为发送激光、视觉等帧率较高，数据量较大的数据时，接收端会卡顿，神奇的是，如果是CycloneDDS发，FastDDS接收，则没有这一内存泄漏现象。经过一番探究，作者发现：**这是因为FastDDS在同主机发送和接收时，会默认走Shared Memory通信，于是，只要daemon启动，则能正常工作。而CycloneDDS则需要显式配置，才可以走Shared Memory通信，默认走网络接口。于是它不依赖于daemon，但是cyclonedds默认只走一个网络接口，如果走了WSL有问题的本机IP（常常是eth0）就会导致非分页缓冲区溢出而卡顿。但是Fastdds默认走所有网络接口，它作为接收端，会启用10.255.255.254这一本机lo接口作为接收，绕过了这一问题。**

可以验证，手动开启CycloneDDS的SHM。如下配置：
```bash
export RMW_IMPLEMENTATION=rmw_cyclonedds_cpp
export CYCLONEDDS_URI='<CycloneDDS><Domain><SharedMemory>
            <Enable>true</Enable>
            <LogLevel>info</LogLevel>
        </SharedMemory></Domain></CycloneDDS>'
```
并在另一个终端，`source /opt/ros/humble/setup.bash`之后，运行`iox-roudi`。此时，CycloneDDS本机通信不再卡顿。

总结一下：
<table>
  <thead>
    <tr>
      <th>中间件发送端</th>
      <th>FastDDS接收</th>
      <th>CycloneDDS接收</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td>FastDDS</td>
      <td>需要开启daemon，走SHM，无卡顿</td>
      <td>无法发现</td>
    </tr>
    <tr>
      <td>CycloneDDS</td>
      <td>走网络，不卡顿</td>
      <td>走网络，卡顿</td>
    </tr>
  </tbody>
</table>

### 不同主机
此时都无法走SHM了。FastDDS作为发送端开启了daemon也无法发现。CycloneDDS作为发送端可以正常发现，但是仍然卡顿，除非使用FastDDS作为接收端。

总结一下：
<table>
  <thead>
    <tr>
      <th>中间件发送端</th>
      <th>FastDDS接收</th>
      <th>CycloneDDS接收</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td>FastDDS</td>
      <td>无法发现</td>
      <td>无法发现</td>
    </tr>
    <tr>
      <td>CycloneDDS</td>
      <td>走网络，不卡顿</td>
      <td>走网络，卡顿</td>
    </tr>
  </tbody>
</table>

### 问题总结
所以问题不在于哪个中间件，而在于中间件的通信方式。只要走了有问题的wsl mirror网络，也就是Windows连接的那个网络，就会卡顿和缓冲区溢出。想要验证也很容易，参考作者[配置dds网络接口](../ros2-dds-config)的方法，强行令FastDDS也走wsl mirror网络（CycloneDDS不配置一般也默认走这个网络），那么FastDDS的表现与CycloneDDS完全一致，也能互相发现，也会卡顿。

## 问题解决
那么现在可以解决问题了。不卡顿的关键在于，无论是wsl还是docker，都是作为一台机器上的程序运行的，它们对外可以走mirror网络，但是对内走本机网络是最正确的。因此，只需要将`10.255.255.254`添加到DDS配置文件所走的网络接口里，就能解决了。由于旧版FastDDS不支持使用网络接口名，而必须使用IP，这会使得每换一个网络，都得重新配置它的mirror接口。因此，最优方案应该是使用CycloneDDS，并且如下配置：
```bash
export RMW_IMPLEMENTATION=rmw_cyclonedds_cpp
export CYCLONEDDS_URI='<CycloneDDS><Domain><General><Interfaces>
                           <NetworkInterface name="eth0" priority="default" multicast="default" /><NetworkInterface address="10.255.255.254" priority="default" multicast="default" />
                        </Interfaces></General></Domain></CycloneDDS>'
```
其中`eth0`是你wsl的mirror网络。

这里也给出FastDDS的配置
<details markdown="1">
  <summary>点击展开</summary>

```bash
export RMW_IMPLEMENTATION=rmw_fastrtps_cpp
export FASTRTPS_DEFAULT_PROFILES_FILE=/path/to/fastdds.xml
```

```xml
<?xml version="1.0" encoding="UTF-8" ?>
<profiles xmlns="http://www.eprosima.com/XMLSchemas/fastRTPS_Profiles">
    <transport_descriptors>
        <transport_descriptor>
            <transport_id>udp_transport</transport_id>
            <type>UDPv4</type>
            <interfaceWhiteList>
                <address>192.168.31.20</address>
                <address>10.255.255.254</address>
            </interfaceWhiteList>
        </transport_descriptor>
    </transport_descriptors>

    <participant profile_name="default_part_profile" is_default_profile="true">
        <rtps>
            <useBuiltinTransports>false</useBuiltinTransports>
            <userTransports>
                <transport_id>udp_transport</transport_id>
            </userTransports>
        </rtps>
    </participant>
</profiles>
```
其中，`192.168.31.20`应改为你的wsl mirror网络的ip。