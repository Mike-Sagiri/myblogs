---
layout: post
title: 为jetson orin nx配置GPU版docker
date: 2026-02-28 16:24 +0800
categories: [技术经验, 英伟达, Jetson Orin]
tags: [nvidia, jetson orin nx, docker] # TAG 名称应始终为小写
---

由于开发需要，jetson orin nx需要配置docker，并且docker内部需要能够使用GPU。由于jetson的特殊性，不仅是arm架构，其GPU也是独特的。因此，与常规docker开启GPU的方法不一样，并需要额外配置。本文讲解具体配置方案。

参考：
1. [csdn](https://blog.csdn.net/zhuyonghou/article/details/153562227)
2. [英伟达docker镜像](https://catalog.ngc.nvidia.com/?tab=container)
3. [jetson pytorch](https://catalog.ngc.nvidia.com/orgs/nvidia/containers/l4t-pytorch?version=r35.2.1-pth2.0-py3)
4. [jetson tensorrt](https://catalog.ngc.nvidia.com/orgs/nvidia/containers/l4t-tensorrt?version=r10.3.0-devel)
上面两个不再更新了，说是变成了[pytorch](https://catalog.ngc.nvidia.com/orgs/nvidia/containers/pytorch?version=26.02-py3)和[tensorrt](https://catalog.ngc.nvidia.com/orgs/nvidia/containers/tensorrt?version=26.02-py3)。

## 系统检查与docker安装

### 系统信息确认
首先确认设备的基本信息：

```bash
# 检查系统版本
cat /etc/nv_tegra_release

#检查jetpack版本信息
sudo apt-cache show nvidia-jetpack

# 检查GPU状态
tegrastats
# 输出显示GPU频率和温度信息，确认GPU正常工作

# 检查设备信息
sudo jetson_clocks --show
# 显示详细的CPU/GPU频率信息
```

### 现有NVIDIA软件检查

```bash
# 检查已安装的NVIDIA相关包
dpkg -l | grep nvidia
# 显示系统中已安装的所有NVIDIA相关软件包

# 检查PCI设备
lspci | grep -i nvidia
# 确认系统能够识别到NVIDIA硬件
```

### docker安装
建议使用[鱼香ROS一键安装](https://fishros.org.cn/forum/topic/20/%E5%B0%8F%E9%B1%BC%E7%9A%84%E4%B8%80%E9%94%AE%E5%AE%89%E8%A3%85%E7%B3%BB%E5%88%97)

```bash
wget http://fishros.com/install -O fishros && . fishros
```
docker安装完后要重启或者log out再登录，刷新用户组。

## Nvidia Container Toolkit

### 安装NVIDIA容器工具包

```bash
sudo apt-get install -y nvidia-container-toolkit
```

验证安装：

```bash
nvidia-ctk --version
```

### 配置NVIDIA容器运行时

```bash
# 配置Docker使用NVIDIA运行时
sudo nvidia-ctk runtime configure --runtime=docker

# 重启Docker服务
sudo systemctl restart docker

# 验证配置
docker info | grep -i runtime
# 输出应显示nvidia作为可用运行时
```

## 下载镜像并验证

```bash
docker pull nvcr.io/nvidia/pytorch:24.09-py3-igpu
docker run -it --rm --runtime=nvidia --gpus all nvcr.io/nvidia/pytorch:24.09-py3-igpu
```
这里注意与直接`--gpus all`不一样。否则会出现报错，让你不能`directly invoke like --gpus all`类似这种。这里我直接使用官方的pytorch镜像了，里面包含torch，torchvision，tensorrt，都是gpu版。

> 注意，jetson的gpu是整合的，与cpu共享内存。这种gpu被称为igpu，而传统桌面gpu被称为dgpu。在`jetpack 6.0`开始，`nvidia-smi`开始部分被支持在igpu上运行。而此前的版本，无法支持`nvidia-smi`。此外，igpu的driver也是整合进系统的，无法升级，除非重新刷写系统。但是cuda版本是可以正常通过环境变量修改的，只要driver支持。详细的说明可以参考[官方的回答](https://forums.developer.nvidia.com/t/nvidia-smi-command-not-found/326686/4)。
{: .prompt-info }

> 部分官方的镜像对driver作了严格的限制，cuda版本也与jetson上的driver不兼容。因此有人`apt install nvidia-jetpack`手动安装了更高版本的。但是还是报不兼容，并且driver还是旧版。但是官方回答说是一个已知的问题，不影响使用。详细的说明可以参考[官方的回答](https://forums.developer.nvidia.com/t/jetpack-6-1-upgrade-issue-host-stuck-on-driver-540-4-0-despite-successful-apt-upgrade/354467)。
{: .prompt-info }

```bash
ls -la /dev/nvidia*
```
正常来讲应该有你之前看到的三个设备。但是你也可以直接运行python，看看pytorch能否使用gpu。解决方案参考上面给的csdn链接。

```bash
# 备份原始配置
sudo cp /etc/nvidia-container-runtime/host-files-for-container.d/l4t.csv /etc/nvidia-container-runtime/host-files-for-container.d/l4t.csv.backup

# 添加缺失的设备配置
echo "dev, /dev/nvidia0" | sudo tee -a /etc/nvidia-container-runtime/host-files-for-container.d/l4t.csv
echo "dev, /dev/nvidia-modeset" | sudo tee -a /etc/nvidia-container-runtime/host-files-for-container.d/l4t.csv

# 验证添加结果
sudo tail -3 /etc/nvidia-container-runtime/host-files-for-container.d/l4t.csv
```
