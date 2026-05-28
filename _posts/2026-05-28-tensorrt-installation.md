---
layout: post
title: TensorRT的安装和使用
date: 2026-05-28 13:55 +0800
categories: [技术经验, 英伟达, TensorRT]
tags: [nvidia, jetson orin nx, pytorch, tensorrt] # TAG 名称应始终为小写
---

TensorRT是英伟达推出的推理加速库，可以加速英伟达显卡上模型的推理速度。下面介绍在Windows、Linux和Jetson上安装TensorRT的方式。本文考虑TensorRT 10版本，同时附上latest版本的链接。

参考：
1. [Nvidia TensorRT 10.x.x](https://docs.nvidia.com/deeplearning/tensorrt/10.x.x/installing-tensorrt/prerequisites.html)
2. [Nvidia TensorRT latest](https://docs.nvidia.com/deeplearning/tensorrt/latest/index.html)

首先确保你所安装的版本与你的平台是对应的，包括cuda等。英伟达非常贴心地给了[Support Matrix 10.x.x](https://docs.nvidia.com/deeplearning/tensorrt/10.x.x/getting-started/support-matrix.html#support-matrix)、[Prerequites 10.x.x](https://docs.nvidia.com/deeplearning/tensorrt/10.x.x/installing-tensorrt/prerequisites.html)

最简单的安装方式，是通过`pip`直接安装，但是这样安装会缺失`trtexec`转换程序，同时也难以切换版本，并且对于Jetson而言，许多版本无法直接安装。因此，本文全部使用源码安装（并不进行编译）。这样安装的另一个好处是，安装方法对Windows、Linux、Jetson都是一样的。只有设置环境变量的时候不甚相同。

首先，访问[下载页面](https://developer.nvidia.com/tensorrt/download)，登录你的英伟达账号，并选择对应版本下载。这里本文选择最新的10.16版本（对于Jetson，使用10.7版本，新版本的SBSA的不再支持cuda12，Jetson用不了，它的driver是540，详见：[CUDA Compatibility](https://docs.nvidia.com/deploy/cuda-compatibility/minor-version-compatibility.html)。~~但是如有必须，可以通过[CUDA Forward Compatibility](https://docs.nvidia.com/deploy/cuda-compatibility/forward-compatibility.html)，安装对应的`cuda-compat-13-3`包，来尽量支持。~~**Jetson并没有对应的cuda-compat包**）。

注意选择对应你的架构：
![alt text](assets/img/tensorrt-install/install_page.png)
_安装页面选择源码_

> 新版本的TensorRT将支持Orin iGPU（也就是统一架构的CPU+GPU）与Linux SBSA结合了，不再单独提供Jetson aarch的版本。
{: .prompt-info }

然后解压即可。接下来就是添加环境变量环境，无论是Windows还是Linux还是Jetson都是一样的，用各自系统添加环境变量的方式即可。

1. Windows：
![alt text](assets/img/tensorrt-install/windows_path.png)
_Windows环境变量的设置_

2. Linux & Jetson
```bash
export LD_LIBRARY_PATH=<TensorRT Path>/lib:$LD_LIBRARY_PATH
export PATH=<TensorRT Path>/bin:$PATH
```

另外，如果你需要对应不同的conda环境，安装不同的TensorRT，可以利用conda安装环境的`envs/<your env name>/etc/conda/activate.d`、`envs/<your env name>/etc/conda/deactivate.d`，这是conda激活和取消激活的时候会运行的脚本，目录不存在就创建一个。新建一个`.sh`文件，叫什么随意，把下面的代码放进去

```bash
# activate.d
export OLD_LD_LIBRARY_PATH="$LD_LIBRARY_PATH"
export OLD_LIBRARY_PATH="$LIBRARY_PATH"
export OLD_CPATH="$CPATH"
export OLD_PATH="$PATH"

export TENSORRT_HOME=<Your TensorRT Path>
export PATH="$TENSORRT_HOME/bin:$PATH"
export LD_LIBRARY_PATH="$TENSORRT_HOME/lib:$LD_LIBRARY_PATH"
export LIBRARY_PATH="$TENSORRT_HOME/lib:$LIBRARY_PATH"
export CPATH="$TENSORRT_HOME/include:$CPATH"
```

```bash
# deactivate.d
export LD_LIBRARY_PATH="$OLD_LD_LIBRARY_PATH"
export LIBRARY_PATH="$OLD_LIBRARY_PATH"
export CPATH="$OLD_CPATH"
export PATH="$OLD_PATH"

unset OLD_LD_LIBRARY_PATH OLD_LIBRARY_PATH OLD_CPATH OLD_PATH
unset TENSORRT_HOME
```

这之后，就可以安装对应的python包了。在TensorRT目录下的python文件夹里，有预编译好的python包，选择对应的版本安装即可。

1. Windows
![alt text](assets/img/tensorrt-install/windows_py.png)
_Windows的python包_

2. Linux & Jetson
![alt text](assets/img/tensorrt-install/linux_py.png)
_Linux的python包_

最后检查安装是否正确：
![alt text](assets/img/tensorrt-install/check.png)
_检查命令_
以下两种都是正常的：
![alt text](assets/img/tensorrt-install/windows_correct.png)
_Windows下经常要等待一会_
![alt text](assets/img/tensorrt-install/linux_correct.png)
_Linux下有时会有这样的warning，不用管它_

如果报错，那么大概率是cuda版本对应不对。

现在就可以参考[官方指南](https://github.com/NVIDIA/TensorRT/blob/main/quickstart/IntroNotebooks/2.%20Using%20PyTorch%20through%20ONNX.ipynb)，尝试TensorRT了，这个指南每个版本都会更新一次，CUDA13+不再支持pycuda。使用pycuda时，最好不要使用`import pycuda.autoinit`而使用`import pycuda.autoprimaryctx`。前者在ros这样的按帧调用的环境下，会出现context问题。