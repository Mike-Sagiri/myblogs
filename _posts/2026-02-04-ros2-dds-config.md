---
layout: post
title: 为ROS2指定使用的网卡
date: 2026-02-04 10:13 +0800
categories: [技术经验, ros2开发]
tags: [ros2, dds, network] # TAG 名称应始终为小写
---

由于许多雷达和其他设备都可能使用网络接口通信，此外，有时机器也会有一个内网接口和一个外网接口。因此，有时需要指定ros2使用的网卡。常见的ros2多播中间件有FastDDS和CycloneDDS。就实践经验而言，前者对高带宽网络处理更好，后者则对复杂网络多播更好。因此，本文将分别介绍使用上述两种中间件时，指定网卡的方法。

参考：
1. [fastdds documentation](https://fast-dds.docs.eprosima.com/en/latest/fastdds/transport/interfaces.html)
2. [cyclonedds documentation](https://cyclonedds.io/docs/cyclonedds/0.10.5/config/cyclonedds_specifics.html#network-and-discovery-configuration)
3. [husarnet](https://husarnet.com/docs/ros2/custom-cyclonedds-xml)

首先，需要知道如何指定中间件，使用环境变量：
```bash
export RMW_IMPLEMENTATION=rmw_fastrtps_cpp
export RMW_IMPLEMENTATION=rmw_cyclonedds_cpp
```
选择你需要的那个中间件即可，上面的是fastdds，下面的是cyclonedds。

## FastDDS配置
首先需要编写配置文件，为一个.xml文件。如下所示：
```xml
<?xml version="1.0" encoding="UTF-8" ?>
<profiles xmlns="http://www.eprosima.com/XMLSchemas/fastRTPS_Profiles">
    <transport_descriptors>
        <transport_descriptor>
            <transport_id>CustomTcpTransportAllowlist</transport_id>
            <type>UDPv4</type>
            <interfaces>
                <allowlist>
                    <interface name="eth0"/>
                </allowlist>
            </interfaces>
        </transport_descriptor>
    </transport_descriptors>

    <participant profile_name="CustomTcpTransportAllowlistParticipant" is_default_profile="true">
        <rtps>
            <useBuiltinTransports>false</useBuiltinTransports>
            <userTransports>
                <transport_id>CustomTcpTransportAllowlist</transport_id>
            </userTransports>
        </rtps>
    </participant>
</profiles>
```
这里需要注意的是，`type`标签需要是UDPv4，因为ros2是使用UDP的。`interface`标签使用网卡名称或者ip都行。`participant`标签必须要有`is_default_profile="true"`。

然后修改环境变量，使之生效。
```bash
export RMW_IMPLEMENTATION=rmw_fastrtps_cpp
export FASTDDS_DEFAULT_PROFILES_FILE=/path/to/your/file.xml
export RMW_FASTRTPS_USE_QOS_FROM_XML=1
```

## CycloneDDS配置
一样的，首先需要编写一个xml文件。但是不一样的是，cyclonedds可以直接把xml文件的内容作为环境变量给出。
```bash
export RMW_IMPLEMENTATION=rmw_cyclonedds_cpp
export CYCLONEDDS_URI='<CycloneDDS><Domain><General><Interfaces>
                            <NetworkInterface name="eth1" priority="default" multicast="default" />
                        </Interfaces></General></Domain></CycloneDDS>'
```
这里`NetworkInterface`标签的内容就是你指定的网卡。如果你更希望使用文件，而不是直接给出环境变量，也可以
```bash
export CYCLONEDDS_URI=file:///path/to/your/file.xml
```
请注意，只有在humble和更新版本使用上述配置。如果是foxy等旧版本的ros2，请改为
```bash
export CYCLONEDDS_URI='<CycloneDDS><Domain><General>
                        <NetworkInterfaceAddress>eth0</NetworkInterfaceAddress>
                        <AllowMulticast>false</AllowMulticast>
                       </General></Domain></CycloneDDS>'
```

## 使用wsl和docker的常见问题
如果wsl或者docker使用宿主机网络，而非NAT桥接，已知会有如下的问题。
1. fastdds无法将话题发送，即使关闭防火墙和开启主机地址环回。但是这并不一定能复现，许多情况下fastdds可以正常工作。
2. cyclonedds内存泄漏，将非分页缓冲区占用大量空间。这一定能复现，如果使用cyclonedds，和wsl内部的docker，并使用宿主网络，当使用rviz2可视化gazebo的相机的时候，就会复现。表现为rviz2可视化的相机不显示画面或者非常卡顿。非分页缓冲区迅速增大。可以改成fastdds解决此问题。
3. ros2 topic list等无反应。这不是中间件的问题，而是wsl的问题，时至今日仍然没有修复。使用ros2 daemon start即可解决。
4. 无法通过127.0.0.1访问wsl或docker内部端口。在确信已经开启镜像或者宿主网络模式之后，这大概是因为没有在wsl setting开启主机地址环回。
