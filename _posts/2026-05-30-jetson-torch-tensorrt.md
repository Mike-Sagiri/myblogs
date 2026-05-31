---
layout: post
title: 为Jetson安装Torch-TensorRT
date: 2026-05-30 13:34 +0800
categories: [技术经验, 英伟达, Jetson Orin]
tags: [nvidia, jetson orin nx, pytorch, tensorrt] # TAG 名称应始终为小写
---

Torch-TensorRT是一个结合了Pytorch和TensorRT的库，可以便捷地将torch模型转换为TensorRT推理模型，加速模型推理速度，而无需复杂的转换，和手动维护TensorRT的CUDA操作。然而，该库对Jetson Orin系列的支持年久失修，作者已经将修改提了个pull request，下面介绍手动编译安装该库的方法。

> 最简单安装这些的方法是使用英伟达的pytorch docker，包含这些所有库。可以参考读者另一篇文章：[为jetson orin nx配置GPU版docker](../jetson-docker-gpu)
{: .prompt-info }

> 注意，jetson orin最稳定的搭配是：torch2.10+tensorrt10.7+torch-tensorrt2.10。如果用最新版的torch-tensorrt2.12，则需要cuda13，否则由于cuda12版本的tensorrt缺少`add_attention`算子，会导致报错，除非禁用掉`torch.backends.cuda.enable_flash_sdp(False)`, `torch.backends.cuda.enable_mem_efficient_sdp(False)`, `torch.backends.cuda.enable_math_sdp(True)`。
{: .prompt-warning }

首先，该库的大版本应该与pytorch版本对应。编译它需要修改以下三个文件：
1. MODULE.bazel：决定编译依赖的位置和版本
2. setup.py：决定编译好的wheel的依赖
3. third_party/tensorrt/local/BUILD：决定TensorRT依赖库的文件匹配

修改好后，使用
```bash
python setup.py bdist_wheel --jetpack
```
编译出.whl文件，放在dist目录下，pip install ./*.whl即可

下面一次介绍需要的修改：

## MODULE.bazel

该文件决定了编译使用的`cuda`、`torch`、`tensorrt`的位置。由于英伟达早已不再维护新版torch、tensorrt和cuda的jetson版本，许多jetson的东西都得自己来弄，所以这里的设置尤其重要。想要手动安装新版torch和tensorrt的读者，可以看作者的其他文章：[TensorRT的安装和使用](../tensorrt-installation)和[为jetson系列安装GPU版pytorch家族](../jetson-pytorch)

### cuda
```
# for Jetson
# These versions can be selected using the flag `--//toolchains/dep_collection:compute_libs="jetpack"`
new_local_repository(
    name = "cuda_l4t",
    build_file = "@//third_party/cuda:BUILD",
    path = "/usr/local/cuda-12.6",
)
```
这里需要把`cuda`对应的path改成自己的，一般jetson orin就只支持12.6，不用改。然后记得把对应的

### torch
这里，需要把
```
new_local_repository(
    name = "torch_l4t",
# please modify this path according to your local python wheel path
    path = "/usr/local/lib/python3.11/dist-packages/torch",
    build_file = "third_party/libtorch/BUILD"
)
```
中的`torch`的路径改成自己的，如果你不知道，可以用`pip show torch`得知它的安装位置。

然后把对应的，从网络下载`torch`的命令注释掉。注意，因为`Torch-TensorRT`库与`torch`版本一般是对应的，所以网络下载的大概率用不了。
```
#http_archive(
#    name = "torch_l4t",
#    build_file = "@//third_party/libtorch:BUILD",
#    strip_prefix = "torch",
#    type = "zip",
#    urls = ["https://pypi.jetson-ai-lab.io/jp6/cu126/+f/62a/1beee9f2f1470/torch-2.8.0-cp310-cp310-linux_aarch64.whl"],
#)
```

### TensorRT
这里一样的，把
```
new_local_repository(
   name = "tensorrt_l4t",
   path = "/usr/",
   build_file = "@//third_party/tensorrt/local:BUILD"
#)
```
的路径改成自己的，请注意需要是TensorRT的根路径，而不是bin、lib等。
然后注释掉从网上下载的
```
#http_archive(
#    name = "tensorrt_l4t",
#    build_file = "@//third_party/tensorrt/archive:BUILD",
#    strip_prefix = "TensorRT-10.16.1.11",
#    urls = [
#        "https://developer.nvidia.com/downloads/compute/machine-learning/tensorrt/10.16.1/tars/TensorRT-10.16.1.11.Linux.aarch64-gnu.cuda-13.2.tar.gz",
#    ],
#)
```

## setup.py
这里主要注意一下，要把
```python
def get_jetpack_requirements(base_requirements):
    requirements = base_requirements + ["numpy<2.0.0"]
    if IS_DLFW_CI:
        return requirements
    else:
        return requirements + ["torch>=2.8.0,<2.9.0", "tensorrt>=10.3.0,<10.4.0"]
```
的numpy<2.0.0去掉，当然不去也没关系。然后把对应的torch和tensorrt版本改成自己的，虽然不改也可以装(`--no-deps`)，因为编译跟它没关系，但是老是会让pip报dependency的错。

## third_party/tensorrt/local/BUILD
这个库对应tensorrt要寻找的文件，它完全忽略了`--jetpack`参数，所以会导致归并到`aarch64`去。要修改的地方很多。

首先，在最开始添加：
```
config_setting(
    name = "jetpack",
    constraint_values = [
        "@platforms//cpu:aarch64",
        "@platforms//os:linux",
    ],
    flag_values = {
        "@//toolchains/dep_collection:compute_libs": "jetpack",
    },
)
```

然后对于每一个`select`，照着其他项，添加`":jetpack"`。涉及到`cuda`的，要在`":jetpack"`把对应的`cuda`改成`cuda_l4t`。

例如：
```
cc_library(
    name = "nvinfer_headers",
    hdrs = select({
                ":jetpack": glob(
            [
                "include/NvInfer*.h",
            ],
            allow_empty = True,
            exclude = [
                "include/NvInferPlugin.h",
                "include/NvInferPluginUtils.h",
            ],
        ),
        ":aarch64_linux": glob(
            [
                "include/aarch64-linux-gnu/NvInfer*.h",
            ],
            allow_empty = True,
            exclude = [
                "include/aarch64-linux-gnu/NvInferPlugin.h",
                "include/aarch64-linux-gnu/NvInferPluginUtils.h",
            ],
        ),
        ":ci_rhel_x86_64_linux": glob(
            [
                "include/NvInfer*.h",
            ],
            allow_empty = True,
            exclude = [
                "include/NvInferPlugin.h",
                "include/NvInferPluginUtils.h",
            ],
        ),
        ":windows": glob(
            [
                "include/NvInfer*.h",
            ],
            allow_empty = True,
            exclude = [
                "include/NvInferPlugin.h",
                "include/NvInferPluginUtils.h",
            ],
        ),
        "//conditions:default": glob(
            [
                one_of([
                    "include/NvInfer.h",
                    "include/x86_64-linux-gnu/NvInfer.h",
                ]).replace("NvInfer.h", "NvInfer*.h"),
            ],
            allow_empty = True,
            exclude = [
                "include/NvInferPlugin.h",
                "include/NvInferPluginUtils.h",
                "include/x86_64-linux-gnu/NvInferPlugin.h",
                "include/x86_64-linux-gnu/NvInferPluginUtils.h",
            ],
        ),
    }),
    includes = select({
        ":jetpack": ["include/"],
        ":aarch64_linux": ["include/aarch64-linux-gnu"],
        ":ci_rhel_x86_64_linux": ["include/"],
        ":windows": ["include/"],
        "//conditions:default": [
            "include/",
            "include/x86_64-linux-gnu/",
        ],
    }),
    visibility = ["//visibility:private"],
)
```

最后，对于`2.12`版本，完整的如下：
<details markdown="1">
  <summary>点击展开</summary>
```
load("@//third_party/tensorrt/local:path_helpers.bzl", "any_of", "one_of")
load("@rules_cc//cc:defs.bzl", "cc_import", "cc_library")

package(default_visibility = ["//visibility:public"])

config_setting(
    name = "aarch64_linux",
    constraint_values = [
        "@platforms//cpu:aarch64",
        "@platforms//os:linux",
    ],
)

config_setting(
    name = "windows",
    constraint_values = [
        "@platforms//os:windows",
    ],
)

config_setting(
    name = "ci_rhel_x86_64_linux",
    constraint_values = [
        "@platforms//cpu:x86_64",
        "@platforms//os:linux",
        "@//toolchains/distro:ci_rhel",
    ],
)

config_setting(
    name = "jetpack",
    constraint_values = [
        "@platforms//cpu:aarch64",
        "@platforms//os:linux",
    ],
    flag_values = {
        "@//toolchains/dep_collection:compute_libs": "jetpack",
    },
)

cc_library(
    name = "nvinfer_headers",
    hdrs = select({
                ":jetpack": glob(
            [
                "include/NvInfer*.h",
            ],
            allow_empty = True,
            exclude = [
                "include/NvInferPlugin.h",
                "include/NvInferPluginUtils.h",
            ],
        ),
        ":aarch64_linux": glob(
            [
                "include/aarch64-linux-gnu/NvInfer*.h",
            ],
            allow_empty = True,
            exclude = [
                "include/aarch64-linux-gnu/NvInferPlugin.h",
                "include/aarch64-linux-gnu/NvInferPluginUtils.h",
            ],
        ),
        ":ci_rhel_x86_64_linux": glob(
            [
                "include/NvInfer*.h",
            ],
            allow_empty = True,
            exclude = [
                "include/NvInferPlugin.h",
                "include/NvInferPluginUtils.h",
            ],
        ),
        ":windows": glob(
            [
                "include/NvInfer*.h",
            ],
            allow_empty = True,
            exclude = [
                "include/NvInferPlugin.h",
                "include/NvInferPluginUtils.h",
            ],
        ),
        "//conditions:default": glob(
            [
                one_of([
                    "include/NvInfer.h",
                    "include/x86_64-linux-gnu/NvInfer.h",
                ]).replace("NvInfer.h", "NvInfer*.h"),
            ],
            allow_empty = True,
            exclude = [
                "include/NvInferPlugin.h",
                "include/NvInferPluginUtils.h",
                "include/x86_64-linux-gnu/NvInferPlugin.h",
                "include/x86_64-linux-gnu/NvInferPluginUtils.h",
            ],
        ),
    }),
    includes = select({
        ":jetpack": ["include/"],
        ":aarch64_linux": ["include/aarch64-linux-gnu"],
        ":ci_rhel_x86_64_linux": ["include/"],
        ":windows": ["include/"],
        "//conditions:default": [
            "include/",
            "include/x86_64-linux-gnu/",
        ],
    }),
    visibility = ["//visibility:private"],
)

cc_import(
    name = "nvinfer_static_lib",
    static_library = select({
        ":jetpack": "lib/libnvinfer_static.a",
        ":aarch64_linux": "lib/aarch64-linux-gnu/libnvinfer_static.a",
        ":ci_rhel_x86_64_linux": "lib/libnvinfer_static.a",
        ":windows": "lib/nvinfer_10.lib",
        "//conditions:default": one_of([
            "lib/libnvinfer_static.a",
            "lib/x86_64-linux-gnu/libnvinfer_static.a",
        ]),
    }),
    visibility = ["//visibility:private"],
)

cc_import(
    name = "nvinfer_lib",
    shared_library = select({
        ":jetpack": "lib/libnvinfer.so",
        ":aarch64_linux": "lib/aarch64-linux-gnu/libnvinfer.so",
        ":ci_rhel_x86_64_linux": "lib/libnvinfer.so",
        ":windows": "bin/nvinfer_10.dll",
        "//conditions:default": one_of([
            "lib/libnvinfer.so",
            "lib/x86_64-linux-gnu/libnvinfer.so",
        ]),
    }),
    visibility = ["//visibility:private"],
)

cc_library(
    name = "nvinfer",
    visibility = ["//visibility:public"],
    deps = [
        "nvinfer_headers",
        "nvinfer_lib",
    ] + select({
        ":jetpack": ["@cuda_l4t//:cudart"],
        ":windows": [
            "nvinfer_static_lib",
            "@cuda_win//:cudart",
        ],
        "//conditions:default": ["@cuda//:cudart"],
    }),
)

####################################################################################

cc_import(
    name = "nvparsers_lib",
    shared_library = select({
        ":jetpack": "lib/libnvparsers.so",
        ":aarch64_linux": "lib/aarch64-linux-gnu/libnvparsers.so",
        ":ci_rhel_x86_64_linux": "lib/libnvparsers.so",
        ":windows": "lib/nvparsers.dll",
        "//conditions:default": one_of([
            "lib/libnvparsers.so",
            "lib/x86_64-linux-gnu/libnvparsers.so",
        ]),
    }),
    visibility = ["//visibility:private"],
)

cc_library(
    name = "nvparsers_headers",
    hdrs = select({
        ":jetpack": [
            "include/NvCaffeParser.h",
            "include/NvOnnxConfig.h",
            "include/NvOnnxParser.h",
            "include/NvOnnxParserRuntime.h",
            "include/NvUffParser.h",
        ],
        ":aarch64_linux": [
            "include/aarch64-linux-gnu/NvCaffeParser.h",
            "include/aarch64-linux-gnu/NvOnnxConfig.h",
            "include/aarch64-linux-gnu/NvOnnxParser.h",
            "include/aarch64-linux-gnu/NvOnnxParserRuntime.h",
            "include/aarch64-linux-gnu/NvUffParser.h",
        ],
        ":ci_rhel_x86_64_linux": [
            "include/NvCaffeParser.h",
            "include/NvOnnxConfig.h",
            "include/NvOnnxParser.h",
            "include/NvOnnxParserRuntime.h",
            "include/NvUffParser.h",
        ],
        ":windows": [
            "include/NvCaffeParser.h",
            "include/NvOnnxConfig.h",
            "include/NvOnnxParser.h",
            "include/NvOnnxParserRuntime.h",
            "include/NvUffParser.h",
        ],
        "//conditions:default": any_of([
            "include/NvCaffeParser.h",
            "include/NvOnnxConfig.h",
            "include/NvOnnxParser.h",
            "include/NvOnnxParserRuntime.h",
            "include/NvUffParser.h",
            "include/x86_64-linux-gnu/NvCaffeParser.h",
            "include/x86_64-linux-gnu/NvOnnxConfig.h",
            "include/x86_64-linux-gnu/NvOnnxParser.h",
            "include/x86_64-linux-gnu/NvOnnxParserRuntime.h",
            "include/x86_64-linux-gnu/NvUffParser.h",
        ]),
    }),
    includes = select({
        ":jetpack": ["include/"],
        ":aarch64_linux": ["include/aarch64-linux-gnu"],
        ":ci_rhel_x86_64_linux": ["include/"],
        ":windows": ["include/"],
        "//conditions:default": [
            "include/",
            "include/x86_64-linux-gnu/",
        ],
    }),
    visibility = ["//visibility:private"],
)

cc_library(
    name = "nvparsers",
    visibility = ["//visibility:public"],
    deps = [
        "nvinfer",
        "nvparsers_headers",
        "nvparsers_lib",
    ],
)

####################################################################################

cc_import(
    name = "nvonnxparser_lib",
    shared_library = select({
        ":jetpack": "lib/libnvonnxparser.so",
        ":aarch64_linux": "lib/aarch64-linux-gnu/libnvonnxparser.so",
        ":ci_rhel_x86_64_linux": "lib/libnvonnxparser.so",
        ":windows": "lib/nvonnxparser.dll",
        "//conditions:default": one_of([
            "lib/libnvonnxparser.so",
            "lib/x86_64-linux-gnu/libnvonnxparser.so",
        ]),
    }),
    visibility = ["//visibility:private"],
)

cc_library(
    name = "nvonnxparser_headers",
    hdrs = select({
        ":jetpack": [
            "include/NvOnnxConfig.h",
            "include/NvOnnxParser.h",
            "include/NvOnnxParserRuntime.h",
        ],
        ":aarch64_linux": [
            "include/aarch64-linux-gnu/NvOnnxConfig.h",
            "include/aarch64-linux-gnu/NvOnnxParser.h",
            "include/aarch64-linux-gnu/NvOnnxParserRuntime.h",
        ],
        ":ci_rhel_x86_64_linux": [
            "include/NvOnnxConfig.h",
            "include/NvOnnxParser.h",
            "include/NvOnnxParserRuntime.h",
        ],
        ":windows": [
            "include/NvOnnxConfig.h",
            "include/NvOnnxParser.h",
            "include/NvOnnxParserRuntime.h",
        ],
        "//conditions:default": any_of([
            "include/NvOnnxConfig.h",
            "include/NvOnnxParser.h",
            "include/NvOnnxParserRuntime.h",
            "include/x86_64-linux-gnu/NvOnnxConfig.h",
            "include/x86_64-linux-gnu/NvOnnxParser.h",
            "include/x86_64-linux-gnu/NvOnnxParserRuntime.h",
        ]),
    }),
    includes = select({
        ":jetpack": ["include/"],
        ":aarch64_linux": ["include/aarch64-linux-gnu"],
        ":ci_rhel_x86_64_linux": ["include/"],
        ":windows": ["include/"],
        "//conditions:default": [
            "include/",
            "include/x86_64-linux-gnu/",
        ],
    }),
    visibility = ["//visibility:private"],
)

cc_library(
    name = "nvonnxparser",
    visibility = ["//visibility:public"],
    deps = [
        "nvinfer",
        "nvonnxparser_headers",
        "nvonnxparser_lib",
    ],
)

####################################################################################

cc_import(
    name = "nvonnxparser_runtime_lib",
    shared_library = select({
        ":jetpack": "lib/libnvonnxparser_runtime.so",
        ":aarch64_linux": "lib/x86_64-linux-gnu/libnvonnxparser_runtime.so",
        ":ci_rhel_x86_64_linux": "lib/libnvonnxparser_runtime.so",
        ":windows": "lib/nvonnxparser_runtime.dll",
        "//conditions:default": one_of([
            "lib/libnvonnxparser_runtime.so",
            "lib/x86_64-linux-gnu/libnvonnxparser_runtime.so",
        ]),
    }),
    visibility = ["//visibility:public"],
)

cc_library(
    name = "nvonnxparser_runtime_header",
    hdrs = select({
        ":jetpack": [
            "include/NvOnnxParserRuntime.h",
        ],
        ":aarch64_linux": [
            "include/aarch64-linux-gnu/NvOnnxParserRuntime.h",
        ],
        ":ci_rhel_x86_64_linux": [
            "include/NvOnnxParserRuntime.h",
        ],
        ":windows": [
            "include/NvOnnxParserRuntime.h",
        ],
        "//conditions:default": any_of([
            "include/NvOnnxParserRuntime.h",
            "include/x86_64-linux-gnu/NvOnnxParserRuntime.h",
        ]),
    }),
    includes = select({
        ":jetpack": ["include/"],
        ":aarch64_linux": ["include/aarch64-linux-gnu"],
        ":ci_rhel_x86_64_linux": ["include/"],
        ":windows": ["include/"],
        "//conditions:default": [
            "include/",
            "include/x86_64-linux-gnu/",
        ],
    }),
    visibility = ["//visibility:private"],
)

cc_library(
    name = "nvonnxparser_runtime",
    visibility = ["//visibility:public"],
    deps = [
        "nvinfer",
        "nvparsers_headers",
        "nvparsers_lib",
    ],
)

####################################################################################

cc_import(
    name = "nvcaffeparser_lib",
    shared_library = select({
        ":jetpack": "lib/libnvcaffe_parsers.so",
        ":aarch64_linux": "lib/aarch64-linux-gnu/libnvcaffe_parsers.so",
        ":ci_rhel_x86_64_linux": "lib/libnvcaffe_parsers.so",
        ":windows": "lib/nvcaffe_parsers.dll",
        "//conditions:default": one_of([
            "lib/libnvcaffe_parsers.so",
            "lib/x86_64-linux-gnu/libnvcaffe_parsers.so",
        ]),
    }),
    visibility = ["//visibility:private"],
)

cc_library(
    name = "nvcaffeparser_headers",
    hdrs = select({
        ":jetpack": [
            "include/NvCaffeParser.h",
        ],
        ":aarch64_linux": [
            "include/aarch64-linux-gnu/NvCaffeParser.h",
        ],
        ":ci_rhel_x86_64_linux": [
            "include/NvOnnxParserRuntime.h",
        ],
        ":windows": [
            "include/NvCaffeParser.h",
        ],
        "//conditions:default": any_of([
            "include/NvCaffeParser.h",
            "include/x86_64-linux-gnu/NvCaffeParser.h",
        ]),
    }),
    includes = select({
        ":jetpack": ["include/"],
        ":aarch64_linux": ["include/aarch64-linux-gnu"],
        ":ci_rhel_x86_64_linux": ["include/"],
        ":windows": ["include/"],
        "//conditions:default": [
            "include/",
            "include/x86_64-linux-gnu/",
        ],
    }),
    visibility = ["//visibility:private"],
)

cc_library(
    name = "nvcaffeparser",
    visibility = ["//visibility:public"],
    deps = [
        "nvcaffeparser_headers",
        "nvcaffeparser_lib",
        "nvinfer",
    ],
)

####################################################################################

cc_library(
    name = "nvinferplugin",
    srcs = select({
        ":jetpack": ["lib/libnvinfer_plugin.so"],
        ":aarch64_linux": ["lib/aarch64-linux-gnu/libnvinfer_plugin.so"],
        ":ci_rhel_x86_64_linux": ["lib/libnvinfer_plugin.so"],
        ":windows": ["lib/nvinfer_plugin_10.lib"],
        "//conditions:default": [one_of([
            "lib/libnvinfer_plugin.so",
            "lib/x86_64-linux-gnu/libnvinfer_plugin.so",
        ])],
    }),
    hdrs = select({
        ":jetpack": glob(
            ["include/NvInferPlugin*.h"],
            allow_empty = True,
        ),
        ":aarch64_linux": glob(
            ["include/aarch64-linux-gnu/NvInferPlugin*.h"],
            allow_empty = True,
        ),
        ":ci_rhel_x86_64_linux": glob(
            ["include/NvInferPlugin*.h"],
            allow_empty = True,
        ),
        ":windows": glob(
            ["include/NvInferPlugin*.h"],
            allow_empty = True,
        ),
        "//conditions:default": any_of([
            "include/NvInferPlugin*.h",
            "include/x86_64-linux-gnu/NvInferPlugin*.h",
        ]),
    }),
    copts = [
        "-pthread",
    ],
    includes = select({
        ":jetpack": ["include/"],
        ":aarch64_linux": ["include/aarch64-linux-gnu/"],
        ":ci_rhel_x86_64_linux": ["include/"],
        ":windows": ["include/"],
        "//conditions:default": [
            "include/",
            "include/x86_64-linux-gnu/",
        ],
    }),
    linkopts = [
        "-lpthread",
    ] + select({
        ":jetpack": ["-Wl,--no-as-needed -ldl -lrt -Wl,--as-needed"],
        ":aarch64_linux": ["-Wl,--no-as-needed -ldl -lrt -Wl,--as-needed"],
        "//conditions:default": [],
    }),
    deps = [
        "nvinfer",
    ] + select({
        ":windows": ["@cuda_win//:cudart"],
        ":jetpack": ["@cuda_l4t//:cudart"],
        "//conditions:default": ["@cuda//:cudart"],
    }),
    alwayslink = True,
)
```

</details>
