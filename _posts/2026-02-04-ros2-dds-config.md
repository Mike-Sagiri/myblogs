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
4. [ROS2 DDS Config Example](https://github.com/ros2/ros2_dds_profiles_examples/tree/main)这个样例很丰富，可以直接用
5. [2.6.11 FastDDS Interface Documentation](https://fast-dds.docs.eprosima.com/en/v2.6.11/fastdds/transport/whitelist.html?highlight=interface)
6. [2.13.6 FastDDS Interface Documentation](https://fast-dds.docs.eprosima.com/en/v2.13.6/fastdds/transport/whitelist.html)
7. [3.4.2 FastDDS Interface Documentation](https://fast-dds.docs.eprosima.com/en/latest/fastdds/transport/whitelist.html#whitelist-interfaces)
8. [3.4.2 FastDDS Interfaces Configuation](https://fast-dds.docs.eprosima.com/en/latest/fastdds/transport/interfaces.html#ifaces-config)

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
                    <interface name="eth0" netmask_filter="ON"/>
                    <interface name="192.168.1.10" netmask_filter="OFF"/>
                </allowlist>
                <blocklist>
                    <interface name="127.0.0.1"/>
                    <interface name="docker0"/>
                </blocklist>
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
export FASTRTPS_DEFAULT_PROFILES_FILE=/path/to/your/file.xml
```
如果是ros2 rolling等humble后的版本，`FASTRTPS_DEFAULT_PROFILES_FILE`将被弃用，因此需改为：
```bash
export FASTDDS_DEFAULT_PROFILES_FILE=/path/to/your/file.xml
```

> 这虽然不是ROS2官方的写法，但是添加的`network_filter`项非常好用，会事先检查目标是否在同一子网，不在就不发，提高网络利用效率，在传图像的时候十分有用
{: .prompt-info }

请注意，部分教程会添加：
```bash
export RMW_FASTRTPS_USE_QOS_FROM_XML=1
```
***不要这么做***，除非需要自定义不同话题走不同的中间件或者接口这样复杂的需求，不然修改它会出现`NotEnoughMemory`和`buffer`的问题。  

***另请注意***，上述方法只在最新版的fastdds有效，目前是3.4X版本。ros2 humble默认的版本是2.6.11，这个版本不支持`interface`标签，必须指定ip。如下所示：
```xml
<?xml version="1.0" encoding="UTF-8" ?>
<profiles xmlns="http://www.eprosima.com/XMLSchemas/fastRTPS_Profiles">
    <transport_descriptors>
        <transport_descriptor>
            <transport_id>udp_transport</transport_id>
            <type>UDPv4</type>
            <interfaceWhiteList>
                <address>192.168.1.61</address>
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
在2.X版本末期，添加了`interface`的支持，可以这么写：
```xml
<?xml version="1.0" encoding="UTF-8" ?>
<profiles xmlns="http://www.eprosima.com/XMLSchemas/fastRTPS_Profiles">
    <transport_descriptors>
        <transport_descriptor>
            <transport_id>udp_transport</transport_id>
            <type>UDPv4</type>
            <interfaceWhiteList>
                <interface>eth0</interface>
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
这种写法直到3.X版本依然有效。
```

## CycloneDDS配置
一样的，首先需要编写一个xml文件。但是不一样的是，cyclonedds可以直接把xml文件的内容作为环境变量给出，并且配置更为简单。
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

## 验证
可以将网卡设置为一个不存在的网卡或者ip，再打开`rviz2`，正常的话会报错，说明生效了。
![alt text](assets/img/ros2-dds/fastdds.png)
_FastDDS的报错信息，虽然报错但是rviz2可以打开_

![alt text](assets/img/ros2-dds/cyclonedds.png)
_CycloneDDS的报错信息，rviz2无法正常打开_

## 使用wsl和docker的常见问题
如果wsl或者docker使用宿主机网络，而非NAT桥接，已知会有如下的问题。
1. fastdds无法将话题发送，即使关闭防火墙和开启主机地址环回。但是这并不一定能复现，许多情况下fastdds可以正常工作。
2. cyclonedds内存泄漏，将非分页缓冲区占用大量空间。这一定能复现，如果使用cyclonedds，和wsl内部的docker，并使用宿主网络，当使用rviz2可视化gazebo的相机的时候，就会复现。表现为rviz2可视化的相机不显示画面或者非常卡顿。非分页缓冲区迅速增大。可以改成fastdds解决此问题。
3. ros2 topic list等无反应。这不是中间件的问题，而是wsl的问题，时至今日仍然没有修复。使用ros2 daemon start即可解决。
4. 无法通过127.0.0.1访问wsl或docker内部端口。在确信已经开启镜像或者宿主网络模式之后，这大概是因为没有在wsl setting开启主机地址环回。
