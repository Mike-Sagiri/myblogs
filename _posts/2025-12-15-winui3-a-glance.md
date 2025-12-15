---
layout: post
title: 浅尝winui3
date: 2025-12-15 15:15 +0800
categories: [技术经验, Windows编程]
tags: [C++, Windows] # TAG 名称应始终为小写
mermaid: true
---

到底该用什么框架做开发？这么多平台，这么多语言，这么多框架争论不休。就目前的趋势来看，简单的小玩意似乎用什么都行。哪怕是win32+MFC，都能开发出像样的软件。但是人总得尝点新东西，尤其是微软的跨平台MAUI的Windows实现——WinUI3。它到底怎么样？先说结论：性能不如UWP（WinUI2），效果与动画在Windows上独树一帜，只有electron能与之争锋，但是WinUI3的程序会小很多。如果考虑Qt，忽略它商用付费的政策的话，倒是普遍的选择。然而，WinUI3的开发太累了，它甚至没有可视化的Designer（直到2025年，它已经问世两年了）。有这功夫，为什么不用WPF甚至Imgui和MFC，或者，跨平台的话，avalonia、flutter、electron、javafx？再考虑到国内信创的需求，要么，avalonia——与WPF一致的开发体验，C#的首选。要么，flutter和electron，大家都用，上手难度也低，但是flutter得学dart语言，electron得会web开发，并且软件庞大，漏洞也多——但无所谓，学习路径可平缓多了（不过也意味着AI也很擅长它们）。Java在国外已经被C#蚕食了半壁江山，但是国内仍然稳居软开第一宝座。哎呀，与其学一堆开发工具，不如触及本质——它们只是你的全栈的一部分罢了！

闲话写给自己，读者只看一乐:)。

下面来试试WinUI3配合C++/WinRT的开发——开发一个将初配准点云全部ICP精配准的小工具。这里需要提醒，除非你想不开或者希望用一些特有的特性，不然请用C#开发winui3。

**参考**：
1. [Create Your Fisrt WinUI3 Project](https://learn.microsoft.com/en-us/windows/apps/winui/winui3/create-your-first-winui3-app)
2. [demo](https://github.com/Mike-Sagiri/icp_winui3)

## 创建项目
请在Visual Studio里创建一个winui3（空白应用，已打包）项目。创建成功后，你需要做一些基础的设置。这里请参考[Create Your Fisrt WinUI3 Project](https://learn.microsoft.com/en-us/windows/apps/winui/winui3/create-your-first-winui3-app)。我使用了不打包，也就是可以随时复制到其他Windows电脑上运行。如果你想让它发布并成功安装，请忽略这一条。

## 添加依赖项
你需要右键项目->属性，将c++标准改为c++20。此外，在c/c++中，把eigen和nanoflann库的地址添加到附加包含目录里。最后，右键项目，管理nuget包，把c++/winrt, Windows Implementation Libraries, win2d都加到项目里。如果你直接克隆了demo仓库，请右键解决方案，还原nuget包。如果你不打包，请修改.vcxproj文件里的ApplicationIcon标签内容（注：许多反馈声称它并不改变app实际图标，目前存疑）

## 文件说明
一个基础的winui3项目有如下代码：App.xaml.cpp，这是程序入口，执行启动后渲染渲染窗口等基础工作。demo里改变窗口大小、图标都是在这里完成的。MainWindow.xaml.cpp这是主窗体，绘制各种控件，执行运算逻辑，都是在这里面完成的。

## 编写用户界面
由于winui3没有可视化的designer，一切都得在mainwindow.xaml里写。这里按照自己的喜好设计即可，但是需要注意，CanvasControl包含在canvas的头部声明里，因此需要单独声明为：
```xaml
<canvas:CanvasControl
```

## 实现逻辑
代码核心集中在App.xaml.cpp和MainWindow.xaml.cpp里。首先，App.xaml.cpp完成启动时窗口大小和居中。窗口启动后，在窗口上做的操作都应该在MainWindow.xaml.cpp里完成。正常c++语法就行。只是注意一下xaml里的控件绑定的事件函数，比如click什么的必须存在。同时获取控件直接用它的名字调用就行了。注意一下UI进程和后台运算进程的异步操作。参见：[考虑线程亲和性](https://learn.microsoft.com/en-us/windows/uwp/cpp-and-winrt-apis/concurrency-2#programming-with-thread-affinity-in-mind)
```c++
IAsyncAction DoWorkAsync(TextBlock textblock)
{
    winrt::apartment_context ui_thread; // Capture calling context.

    co_await winrt::resume_background();
    // Do compute-bound work here.

    co_await ui_thread; // Switch back to calling context.

    textblock.Text(L"Done!"); // Ok if we really were called from the UI thread.
}
```

不然肯定会出现各种各样的奇怪bug。

## 构建
记得用release构建，那快得多。因为没用到Assets/，可以把它删了。同时一系列调试用的文件也可以删掉。但是如果你不选择打包的话，你需要[Windows App Runtime](https://apps.microsoft.com/detail/9pcmpl33xp5m?hl=zh-CN&gl=CN)，或者[Windows App SDK](https://learn.microsoft.com/en-us/windows/apps/windows-app-sdk/downloads)


