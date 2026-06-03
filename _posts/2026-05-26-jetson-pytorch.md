---
layout: post
title: 为jetson系列安装GPU版pytorch家族
date: 2026-05-26 19:48 +0800
categories: [技术经验, 英伟达, Jetson Orin]
tags: [nvidia, jetson orin nx, pytorch] # TAG 名称应始终为小写
---

> 最新的Jetpack 7.2+增加了对orin的支持，并且提供了cuda13，将统一化为Linux SBSA格式。对应也有适用于jetpack 6.2的[cuda-compat-orin-13包](https://forums.developer.nvidia.com/t/test-run-public-wheels-cuda-13-2-jetson-orin-family/364497)
{: .prompt-info }

英伟达的Jetson系列边缘计算平台，集成了CPU与GPU，而所需要的GPU版本的Pytorch安装不能直接按照普通电脑安装的方式，有自己的一套安装方案。下面，本文将介绍jetson系列的GPU版本pytorch的安装。

## 一键预编译安装
英伟达预编译了一些版本的pytorch，安装最简便，但是版本可供选择较少，不够灵活：[英伟达官方论坛](https://forums.developer.nvidia.com/t/pytorch-for-jetson/72048)：这个是最简单的，预编译的pytorch、torchvision、torchaudio。

![alt text](assets/img/jetson-pytorch/nvidia_forum.png)
_英伟达官方论坛的预编译包_

> [JPL仓库](https://pypi.jetson-ai-lab.io/jp6/cu126)提供了更多预编译好的包，需要自行寻找匹配的包。
{: .prompt-warning }

## 手动下载安装预编译包
英伟达提供了一些版本的pytorch，但是需要一些繁琐的步骤确认兼容性，适用于直到jetson thor的jetpack7，但是有些描述比较老旧。我们一步步来：[英伟达官方教程](https://docs.nvidia.com/deeplearning/frameworks/install-pytorch-jetson-platform/index.html)

例如，作者需要在jetson orin，jetpack 6.2.2，cuda12.6安装pytorch，大体可以按照文中的来。但是文中提到的cusparselt的安装脚本，最高支持到cuda12.4，因此我们选择手动安装。访问：[cusparselt安装地址](https://developer.nvidia.com/cusparselt-downloads)。这里作者安装的是0.8.1版本，这个版本有一个专门的jetson架构的选项，然后按照说明，安装cuda-12的版本即可。

![alt text](assets/img/jetson-pytorch/cusparselt.png)
_cuSPARSELT的安装界面，注意安装cuda-12后缀版本，不要直接安装不带后缀的版本（除非你使用cuda13）_

> 从0.9.0版本开始，不再提供jetson-aarch的版本，官方称需要迁移到Arm64 (SBSA)。[发行记录](https://docs.nvidia.com/cuda/cusparselt/release_notes.html)
{: .prompt-warning }

后续关于pytorch的安装，从24.09起，也就是torch2.5，英伟达没有提供预编译好的pytorch，如果需要新版本的pytorch，可以后续手动源码编译，或者docker安装。[这里](https://docs.nvidia.com/deeplearning/frameworks/install-pytorch-jetson-platform-release-notes/pytorch-jetson-rel.html#pytorch-jetson-rel)提供了兼容性说明。表格中的[Wheel](https://developer.download.nvidia.com/compute/redist/jp/)和[Container](https://catalog.ngc.nvidia.com/orgs/nvidia/containers/pytorch)提供了所有英伟达提供的项目列表。

![alt text](assets/img/jetson-pytorch/capability.png)
_预编译docker镜像、pytorch wheel与jetpack的兼容性_

## Docker安装
英伟达为jetson系列制作了一系列docker镜像，包含tensorrt、pytorch等应有尽有，是最简便的安装方式。参考[笔者之前的文章](../jetson-docker-gpu)。

## 源码编译
由于torchvision和torchaudio基本都需要手动编译，只有极少的预编译包在上述提到的官方论坛提供。因此这里提供源码编译的方案。

源码编译时可能会爆内存，可以参考[这个方法](https://www.forecr.io/blogs/programming/how-to-increase-swap-space-on-jetson-modules)增大交换空间。Pytorch源码编译至少要60GB内存，Torchvision至少要24GB内存。作者的orin nx是16GB+55GB(swap)，编译pytorch最高使用了57GB内存。

另外，编译时我一般会去掉`-e`参数，因为我不需要二次开发。但是这样你需要退出源码文件夹，再测试`import torch torchvision torchaudio`，否则会导入当前文件夹没有编译好的代码，导致报错。

如果需要conda环境等隔离，一定要加上`--no-user`参数，如果是安装在系统python环境，请一定要`sudo`，否则如果默认的路径不可写，就会安装到`.local`下，这样由于路径优先级，会导致在任何环境下，都可以访问安装的包。

### Pytorch源码编译

> pytorch源码编译耗时非常久，尤其是在jetson上，实测jetson orin nx可能需要超过12小时，如果不幸中途出现问题，编译好的部分会保留。
{: .prompt-warning }

首先确定自己的平台的计算能力（Compute Capability）。可以运行如下命令：
```bash
nvidia-smi --query-gpu=name,compute_cap --format=csv
```
新版的cuda(13+)无需format参数。参考[cuda-programming-guide](https://docs.nvidia.com/cuda/cuda-programming-guide/05-appendices/compute-capabilities.html)。旧版本的叫做[cuda-c-programming-guide](https://docs.nvidia.com/cuda/archive/12.6.1/pdf/CUDA_C_Programming_Guide.pdf)。

同时，jetpack5及以下版本的jetson不支持`nvidia-smi`命令，可以在[CUDA GPU Compute Capability](https://developer.nvidia.com/cuda/gpus) [Legacy CUDA GPU Compute Capability](https://developer.nvidia.com/cuda/gpus/legacy)找到自己对应的计算能力。

> 计算能力（compute capability, cc）是架构层面的，可以理解为支持哪些cuda功能，而非实际算力。例如，4090和4060的cc都是8.9，但是显然二者算力相差巨大。
{: .prompt-warning }

接下来源码编译Pytorch，有两种思路，一种是官方思路，旨在安装到本机，而不显式编译成wheel。一种是build思路，旨在发布成wheel，显式编译为build而不自动安装到本机。前者的一般命令为：
```bash
pip install -v . --no-build-isolation
```

后者的一般命令为：

```bash
python -m build --wheel --no-isolation #new version, without source code
python -m build --no-isolation #new version, with source code
python setup.py bdist_wheel #old version, recommended for jetson+pytorch
pip wheel . -v -w dist --no-build-isolation #recommended for jetson
```

首先设置环境变量：
```bash
export USE_CUDA=1
export USE_CUDNN=1

#export USE_SVE=0
#export BUILD_IGNORE_SVE_UNAVAILABLE=1 #2.12版的torch把CUDNN和SVE支持都开了，这样设置防止报错

export USE_NCCL=0
#export USE_DISTRIBUTED=0 #有些库会依赖它，比如torch-tensorrt，最好开开设置为1

export USE_QNNPACK=0
export USE_PYTORCH_QNNPACK=0

export USE_MKLDNN=0

export TORCH_CUDA_ARCH_LIST="8.7" #这里使用你的平台的计算能力

export BUILD_TEST=0
export USE_NINJA=1

export PYTORCH_BUILD_VERSION=<version> #e.g., 2.10.0
export PYTORCH_BUILD_NUMBER=1
```

我使用的编译完整参数：
```bash
PyTorch built with:
  - GCC 11.4
  - C++ Version: 201703
  - OpenMP 201511 (a.k.a. OpenMP 4.5)
  - LAPACK is enabled (usually provided by MKL)
  - NNPACK is enabled
  - CPU capability usage: DEFAULT
  - CUDA Runtime 12.6
  - NVCC architecture flags: -gencode;arch=compute_87,code=sm_87
  - CuDNN 90.3
  - Build settings: BLAS_INFO=open, BUILD_TYPE=Release, COMMIT_SHA=449b1768410104d3ed79d3bcfe4ba1d65c7f22c0, CUDA_VERSION=12.6, CUDNN_VERSION=9.3.0, CXX_COMPILER=/usr/bin/c++, CXX_FLAGS= -ffunction-sections -fdata-sections -fvisibility-inlines-hidden -DUSE_PTHREADPOOL -DNDEBUG -DUSE_KINETO -DLIBKINETO_NOROCTRACER -DLIBKINETO_NOXPUPTI=ON -DUSE_PYTORCH_QNNPACK -DAT_BUILD_ARM_VEC256_WITH_SLEEF -DUSE_XNNPACK -DSYMBOLICATE_MOBILE_DEBUG_HANDLE -O2 -fPIC -DC10_NODEPRECATED -Wall -Wextra -Werror=return-type -Werror=non-virtual-dtor -Werror=range-loop-construct -Werror=bool-operation -Wnarrowing -Wno-missing-field-initializers -Wno-unknown-pragmas -Wno-unused-parameter -Wno-strict-overflow -Wno-strict-aliasing -Wno-stringop-overflow -Wsuggest-override -Wno-psabi -Wno-error=old-style-cast -faligned-new -Wno-maybe-uninitialized -fno-math-errno -fno-trapping-math -Werror=format -Wno-stringop-overflow, FORCE_FALLBACK_CUDA_MPI=1, LAPACK_INFO=open, TORCH_VERSION=2.10.0, USE_CUDA=1, USE_CUDNN=1, USE_CUSPARSELT=ON, USE_EIGEN_FOR_BLAS=ON, USE_GFLAGS=OFF, USE_GLOG=OFF, USE_GLOO=ON, USE_MKL=OFF, USE_MKLDNN=0, USE_MPI=ON, USE_NCCL=1, USE_NNPACK=ON, USE_OPENMP=ON, USE_ROCM=OFF, USE_ROCM_KERNEL_ASSERT=OFF, USE_XCCL=OFF, USE_XPU=OFF
```
> 这**不是**最合适的参数，我错误的启用了USE_NCCL=1，但事实上，jetson上完全不需要它，它与并行训练有关。但是这个编译参数可以让pytorch2.10正确地运行在jetson orin上。
{: .prompt-warning }

然后下载pytorch源码：
```bash
git clone https://github.com/pytorch/pytorch
cd pytorch
# if you are updating an existing checkout
git submodule sync
git submodule update --init --recursive

pip install --group dev
```
#### 官方编译
之后你可以选择按照pytorch仓库的描述，进行安装，但是需要注意，jetson的arm架构没有mkl库，所以可以跳过`pip install mkl-static mkl-include`、`.ci/docker/common/install_magma_conda.sh 12.4`这一步。之后可以直接安装：
```bash
export CMAKE_PREFIX_PATH="${CONDA_PREFIX:-'$(dirname $(which conda))/../'}:${CMAKE_PREFIX_PATH}"
python -m pip install --no-build-isolation -v -e .
```

#### Wheel编译
这里我们用的是wheel编译，这样可以迁移到其他orin上。
```bash
python setup.py bdist_wheel
```
得到wheel之后，pip install即可。

> 可以使用如下命令检查pytorch是否启用了cuDNN、cuSPARSELT：`print(torch.__config__.show())`。新版本也可以：`print(torch.backends.cudnn.enabled)`、`print(torch.backends.cusparselt.is_available())`
{: .prompt-info }

### Torchivision源码编译

参考：[csdn](https://blog.csdn.net/Ashuo567/article/details/147442512)、[torchvision官方](https://github.com/pytorch/vision/blob/main/CONTRIBUTING.md#development-installation)、[github上的教程](https://github.com/azimjaan21/jetpack-6.1-pytorch-torchvision-)。注意安装对应torch版本的torchvision。

### Torchaudio源码编译

Torchaudio官方给出了源码编译的[教程](https://docs.pytorch.org/audio/stable/build.jetson.html)。教程都比较老，只考虑编译Torchaudio的代码即可。注意，教程里的`--no-use-pep517`已经在新版的pip中移除，请使用`--no-build-isolation`。

老版本的Torchaudio可能会报一个`FLT_MAX undefined`的问题。因为从cuda12.6起，需要显式`#include "float.h"`才能使用这些预定义宏。所以在`audio`源码下找一个叫`ctc_prefix_decoder_kernel_vx.cu`的文件，其中`vx`是版本号。在最前方加上`#include "float.h"`即可。

![alt text](assets/img/jetson-pytorch/final_results.png)
_最终安装结果_

## 常见问题

### RuntimeError: operator torchvision::nms does not exist
需要退出torchvision的源码目录再import，否则会导入当前目录下的未编译的torchvision

### Torchaudio编译报错：identifier "FLT_MAX" is undefined
audio/src/libtorchaudio/cuctc/src/ctc_prefix_decoder_kernel_v2.cu这个文件要加上`#include "float.h"`

### 在系统python下安装torch，但切进conda环境后仍然可见
这是由于如果不加`sudo`，pip无法访问系统路径，只能装在.local下，而这个路径在conda环境里同样可见，并且位于conda下pip安装路径之前。因此，绝对不要使用`--user`，并且用`sudo`安装系统包。

使用如下命令查看当前python环境变量：
```bash
python -c "import sys; print('\n'.join(sys.path))"
```

使用如下命令查看包的安装位置：
```bash
pip show <python bag>
```

### ModuleNotFoundError: No module named 'pkg_resources'
setuptools 的里程碑式大版本更新（82.0.0及以上）中，官方正式移除了 pkg_resources 模块，因此降级即可：
```bash
python -m pip install "setuptools<82"
```

### Torchaudio编译报错：no such option --no-use-pep517
新版pip已经移除了，使用`--no-build-isolation`替代。

### Pytorch依赖MKL库下载报错：ERROR: Could not find a version that satisfies the requirement mkl-static
arm架构本就不支持它，编译pytorch时加上`export USE_MKLDNN=0`即可。

### Torchaudio编译报错：fatal error: torch/csrc/stable/xxx.h: No such file or directory
如果确认pytorch已经正常安装了，这是因为曾经在`sudo`下编译过，导致构建缓存异常，运行如下命令删掉：
```bash
python setup.py clean
rm -rf build/ dist/ torchaudio.egg-info
```

### Pytorch编译报错：raise ValueError('could not identify license file ' ValueError: could not identify license file xxx)
安装最新的2.12版本的torch的时候，third_party里有些包，例如vcpkg，的LICENSE无法被识别为常见的LICENSE，导致build wheel的时候报错。最简单的方式就是就是改掉对应包的`build_bundled.py`的
```python
except ValueError:
    ident = None
```


> 千问在解决这些问题挺好用，而且jetson上访问比较容易，可以问问它。
{: .prompt-info }