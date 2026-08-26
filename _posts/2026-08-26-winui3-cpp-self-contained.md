---
layout: post
title: 创建自包含的c++ Winui3项目
date: 2026-08-26 20:44 +0800
categories: [技术经验, Windows编程]
tags: [C++, Windows] # TAG 名称应始终为小写
mermaid: true
---

参考：
1. [自包含项目的部署-Microsoft Learn](https://learn.microsoft.com/en-us/windows/apps/package-and-deploy/self-contained-deploy/deploy-self-contained-apps)
2. [分发一个未打包的winui3项目-Microsoft Learn](https://learn.microsoft.com/en-us/windows/apps/package-and-deploy/unpackage-winui-app)
3. [参考样例](https://github.com/microsoft/WindowsAppSDK-Samples/blob/43404afcc4e72294b3e2706d2eff12418dbb815a/Samples/SelfContainedDeployment/cpp-winui-unpackaged)

上述微软官方样例解释的其实比较清楚，但是本文主要解释以下问题：
1. 如何配置以创建一个c++的winui3自包含项目。
2. 上述两个链接讲述的自包含的区别。

## 如何配置以创建

首先，将项目配置为不打包：
参照微软官方的说明，在C++的winui3项目的`.vcxproj`文件，作如下修改：
```xml
<Project ...>
  ...
  <PropertyGroup Label="Globals">
    ...
    <AppxPackage>false</AppxPackage><!-- update this -->
    <WindowsPackageType>None</WindowsPackageType><!-- add this -->
    ...
  </PropertyGroup> 
  ...
</Project>
```

不打包的结果是，你的程序可以不走应用商店那种安装方式，与一个普通程序一样迁移。但是，winui3依赖windows app sdk，一种方案是将[windows app sdk的安装程序](https://learn.microsoft.com/zh-cn/windows/apps/windows-app-sdk/downloads)附在你的程序里，供用户选择性地进行安装。

另一种方案就是本文要说的，把你的程序变成自包含的，它将自带所有的依赖，可以直接迁移。

根据官方说明，自包含需要在`.vcxproj`里这样设置：
```xml
<PropertyGroup Label="Globals">
  <WindowsPackageType>None</WindowsPackageType>
  <WindowsAppSDKSelfContained>true</WindowsAppSDKSelfContained>
</PropertyGroup>
```

最终结果类似于：
```xml
  <PropertyGroup Label="Globals">
    <CppWinRTOptimized>true</CppWinRTOptimized>
    <CppWinRTRootNamespaceAutoMerge>true</CppWinRTRootNamespaceAutoMerge>
    <MinimalCoreWin>true</MinimalCoreWin>
    <ProjectGuid>{95642739-055e-4fcc-9c55-8961bcdf2058}</ProjectGuid>
    <ProjectName>UDPServer</ProjectName>
    <RootNamespace>UDPServer</RootNamespace>
    <!--
      $(TargetName) should be same as $(RootNamespace) so that the produced binaries (.exe/.pri/etc.)
      have a name that matches the .winmd
    -->
    <TargetName>$(RootNamespace)</TargetName>
    <DefaultLanguage>zh-CN</DefaultLanguage>
    <MinimumVisualStudioVersion>16.0</MinimumVisualStudioVersion>
    <AppContainerApplication>false</AppContainerApplication>
    <AppxPackage>false</AppxPackage>
    <ApplicationType>Windows Store</ApplicationType>
    <ApplicationTypeRevision>10.0</ApplicationTypeRevision>
    <WindowsTargetPlatformVersion>10.0</WindowsTargetPlatformVersion>
    <WindowsTargetPlatformMinVersion>10.0.17763.0</WindowsTargetPlatformMinVersion>
    <UseWinUI>true</UseWinUI>
    <EnableMsixTooling>true</EnableMsixTooling>
    <WindowsPackageType>None</WindowsPackageType>
    <WindowsAppSDKSelfContained>true</WindowsAppSDKSelfContained>
  </PropertyGroup>
```
这样你的程序build之后，得到的就是不依赖Windows app sdk的了。

## 自包含的区别

最好分发的single-file exe，c++是不支持的，只有c#设置后publish才支持。而上述自包含其实还是有缺陷的，那就是不含c++的运行库。[自包含项目的部署-Microsoft Learn](https://learn.microsoft.com/en-us/windows/apps/package-and-deploy/self-contained-deploy/deploy-self-contained-apps)链接的提示里明确提到，如果要让c++**真正地**自包含，还需要**Hybrid CRT**。

因此，[参考样例](https://github.com/microsoft/WindowsAppSDK-Samples/blob/43404afcc4e72294b3e2706d2eff12418dbb815a/Samples/SelfContainedDeployment/cpp-winui-unpackaged)，在项目里添加两个文本文件：`Directory.Build.props`、`HybridCRT.props`。

**Directory.Build.props:**
```txt
<?xml version="1.0" encoding="utf-8"?>
<Project ToolsVersion="14.0" xmlns="http://schemas.microsoft.com/developer/msbuild/2003">
  <Import Project="$(MSBuildThisFileDirectory)HybridCRT.props" />
</Project>
```

**HybridCRT.props:**
```txt
<?xml version="1.0" encoding="utf-8"?>
<!-- Copyright (c) Microsoft Corporation. All rights reserved. Licensed under the MIT License. See LICENSE in the project root for license information. -->
<Project ToolsVersion="14.0" xmlns="http://schemas.microsoft.com/developer/msbuild/2003">

	<ItemDefinitionGroup Condition="'$(Configuration)'=='Debug'">
		<ClCompile>
			<!-- We use MultiThreadedDebug, rather than MultiThreadedDebugDLL, to avoid DLL dependencies on VCRUNTIME140d.dll and MSVCP140d.dll. -->
			<RuntimeLibrary>MultiThreadedDebug</RuntimeLibrary>
		</ClCompile>
		<Link>
			<!-- Link statically against the runtime and STL, but link dynamically against the CRT by ignoring the static CRT
           lib and instead linking against the Universal CRT DLL import library. This "hybrid" linking mechanism is
           supported according to the CRT maintainer. Dynamic linking against the CRT makes the binaries a bit smaller
           than they would otherwise be if the CRT, runtime, and STL were all statically linked in. -->
			<IgnoreSpecificDefaultLibraries>%(IgnoreSpecificDefaultLibraries);libucrtd.lib</IgnoreSpecificDefaultLibraries>
			<AdditionalOptions>%(AdditionalOptions) /defaultlib:ucrtd.lib</AdditionalOptions>
		</Link>
	</ItemDefinitionGroup>
	<ItemDefinitionGroup Condition="'$(Configuration)'=='Release'">
		<ClCompile>
			<!-- We use MultiThreaded, rather than MultiThreadedDLL, to avoid DLL dependencies on VCRUNTIME140.dll and MSVCP140.dll. -->
			<RuntimeLibrary>MultiThreaded</RuntimeLibrary>
		</ClCompile>
		<Link>
			<!-- Link statically against the runtime and STL, but link dynamically against the CRT by ignoring the static CRT
           lib and instead linking against the Universal CRT DLL import library. This "hybrid" linking mechanism is
           supported according to the CRT maintainer. Dynamic linking against the CRT makes the binaries a bit smaller
           than they would otherwise be if the CRT, runtime, and STL were all statically linked in. -->
			<IgnoreSpecificDefaultLibraries>%(IgnoreSpecificDefaultLibraries);libucrt.lib</IgnoreSpecificDefaultLibraries>
			<AdditionalOptions>%(AdditionalOptions) /defaultlib:ucrt.lib</AdditionalOptions>
		</Link>
	</ItemDefinitionGroup>

</Project>
```

最终项目结构类似于：
![alt text](assets/img/winui3-selfcontained/project.png)
_项目结构_

