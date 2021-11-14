---
title: 基于Docker编译&调试WebRTC源码
date: 2021-11-14 08:55:40
tags:
- Android
- WebRTC
- 音视频
---

学习音视频，绕不过WebRTC这个大名鼎鼎的框架，今天来看下如何使用Docker编译&调试WebRTC吧。

<!-- more -->

## 前言

本文将带大家学习如何在Docker下，配置WebRTC的编译&调试开发环境——基于Ubuntu 18.04系统，并记录了对WebRTC Sdk的Java和native进行调试的方法。



WebRTC是谷歌的一个著名开源项目之一，托管在谷歌的服务器上。由于众所周知的原因，谷歌的产品服务难以在国内访问。请各位合理使用网络，准备上网加速工具，可以避免很多问题。



WebRTC官网：[WebRTC](https://webrtc.org/)。

## 编译

## 配置Docker环境

> 本文主要以MacOS和Linux为例，Windows用户请找到对应命令的图形化操作路径即可。

#### 1. 安装Docker

请参考Docker官方文档进行安装：[Install Docker Engine](https://docs.docker.com/engine/install/)。

#### 2. 获取Ubuntu 18.04镜像

> 强烈建议镜像使用 Ubuntu 18.04及以上版本，避免后续环境配置的问题！！！

打开终端，执行：

```bash
docker pull ubuntu:18.04
```

即可获取到Ubuntu 18.04镜像。

#### 3. 创建一个Ubuntu 18.04容器

```bash
docker run -d -p 22:2222 ubuntu:18.04
```

> 这里映射了容器内的22端口到主机的2222端口，为的是后续使用`rsync`同步文件方便。

#### 4. 容器内系统的基础配置

```bash
apt update && apt upgrade -y
apt install -y git curl python3 python3-pip zsh lsb-release
```

### 安装源码下载工具

下载WebRTC源码需要用到Google开发的[depot tools](https://chromium.googlesource.com/chromium/tools/depot_tools.git)，它是为Chromium项目专门开发的一套源码管理工具，底层用到了Git。

> depot tools的使用文档：[depot tools(7) Manual Page](https://commondatastorage.googleapis.com/chrome-infra-docs/flat/depot_tools/docs/html/depot_tools.html)

```bash
mkdir ~/.local
git clone https://chromium.googlesource.com/chromium/tools/depot_tools.git ~/.local/depot_tools
# 如果使用的是zsh，请将.bashrc改为.zshrc
echo "export PATH=$HOME/.local/depot_tools:$PATH" >> ~/.bashrc
source ~/.bashrc
```

### 下载源码

因为主要在Android平台上进行开发，所以仅下载WebRTC核心+Android部分的代码。

```bash
mkdir ~/code/webrtc
cd ~/code/webrtc
# 下载WebRTC核心源码+用于Android平台的Sdk源码
# 源码下载完成后，位于~/code/webrtc/src目录
fetch --nohooks --no-history webrtc_android
# 同步下依赖
gclient sync
```

上面拉取的是最新的主线代码，强烈建议使用`--no-history`参数，加快下载速度，减少磁盘占用。

> 2021年11月13日 完整下完整套代码，约27G～

一般项目开发中会选择一个正式的release版本。截止到2021年11月13日，最新的版本是`M85`，下面基于M85创建一个自己的开发分支。

> Release信息可以在这查询：[Chrome Release Notes | WebRTC](https://webrtc.github.io/webrtc-org/release-notes/)

```bash
# 基于M85 release（对应的分支名为branch-heads/4183）创建一个名为m85的分支，并且切换过去
git checkout -b m85 branch-heads/4183
# 再次同步下依赖
gclient sync
```

### 安装编译依赖

在WebRTC源码中，包含了**自动安装编译依赖**的脚本，直接使用即可。

```bash
# 如果没有安装python命令，需要软链接下
ln -s /usr/bin/python3 /usr/bin/python
# 安装WebRTC基础的依赖
./build/install-build-deps.sh
# 安装编译Web RTC安卓Sdk需要的依赖
./build/install-build-deps-android.sh
```

### 开始编译

同样，WebRTC源码中包含了编译Android AAR的脚本，直接使用。

```bash
./tools_webrtc/android/build_aar.py --build-dir Build --arch arm64-v8a
```

上面的命令中，我们指定了**编译中间产物的路径**和**目标架构**。



可以通过`./tools_webrtc/android/build_aar.py -h`查看可传入的参数列表及详细的信息：

![编译AAR脚本输出的帮助信息](/images/build_tools_help_info.png)



从图中可以看到WebRTC支持编译`armeabi-v7a`、`arm64-v8a`、`x86`、`x86_64`这四种架构。

> 建议只编译所需架构，减少非必要的时间。笔者用nuci5beh（豆子峡谷）单编译arm64-v8a架构，花了**十分钟**。



最终编译产物（aar文件）将会在`~/code/webrtc/src/`目录下，即`~/code/webrtc/src/libwebrtc.aar`。

## 调试

光说不练假把式，来看下如何使用Android Studio调试WebRTC Android Sdk的Java和native代码吧。



WebRTC官方给的Android Demo是这个：[examples/androidapp - src - Git at Google (googlesource.com)](https://webrtc.googlesource.com/src/+/main/examples/androidapp/)。

但由于一些问题，无法使用Android Studio进行调试。原因可看：[9282 - Support Android Studio for Android development - webrtc (chromium.org)](https://bugs.chromium.org/p/webrtc/issues/detail?id=9282)。



幸运的是，国内的小伙伴[mthli (Matthew Lee) (github.com)](https://github.com/mthli)提供了一份可导入Android Studio进行调试的Demo：[YaaRTC](https://github.com/mthli/YaaRTC)。我们用它来演示如何进行调试吧。

### 编译Debug包

通常，调试需要用到包含符号信息等调试信息的包，即Debug包。但是在编译时，默认会把调试信息strip掉。因此需要修改下编译脚本，并重新构建生成`aar`文件。

> **建议参考**[断点调试 - WebRTC 学习指南 (mthli.com)](https://webrtc.mthli.com/basic/webrtc-breakpoint/) 。这里抄录下相关配置。

```bash
# 修改~/code/webrtc/src/build/toolchain/android/BUILD.gn文件
# 注意，~/code/webrtc/src/build/是自动生成的文件夹
...
template("android_clang_toolchain") {
  ...
  _prefix = rebase_path("$clang_base_path/bin", root_build_dir)
  cc = "$_prefix/clang"
  cxx = "$_prefix/clang++"
  ar = "$_prefix/llvm-ar"
  ld = cxx
  readelf = _tool_prefix + "readelf"
  nm = _tool_prefix + "nm"
  # 注释掉下面两行配置，即可实现 unstrip
  # strip = rebase_path("//buildtools/third_party/eu-strip/bin/eu-strip",
  #                     root_build_dir)
  # use_unstripped_as_runtime_outputs = android_unstripped_runtime_outputs
  ...
}
...
```

编译时，注意还需要添加一些参数：

```bash
./tools_webrtc/android/build_aar.py --build-dir=Build --arch arm64-v8a --extra-gn-args "is_debug=true symbol_level=2 android_full_debug=true"
```

等待编译完成，这样就得到了带调试信息的Debug包。记得用它替换掉项目中正在用的`libwebrtc.aar`。

### 配置`build.gradle`

除了**需要Debug包**以外，还需要在引用了WebRTC Android Sdk模块的`build.gradle`中，做如下修改：

```groovy
android {
  ...
  buildTypes {
    ...
    debug {    
      ...
      debuggable true
      jniDebuggable true
      minifyEnabled false
    }
  }
  
  ...
  packagingOptions {
    ...
    // 这里禁止strip了arm64-v8a架构的so文件
    // 如果使用的是其它架构，按类似的方法加即可
    doNotStrip "*/arm64-v8a/*.so"
  }
}
```

[YaaRTC](https://github.com/mthli/YaaRTC)已经配置好了，不放心的小伙伴可以检查下。

### 开始调试

#### 同步源码从容器到宿主机

> 如果仅需要调试WebRTC SDK的Java代码，可跳过此步骤。

调试Native层需要用到源码，由于WebRTC的源码位于容器中，因此需要先把它弄到宿主机。可以直接使用`docker cp`命令：

```bash
# 使用Docker cp命令
docker cp {container-id}:{path/to/webrtc/} ~/code/webrtc
# 上面这条命令将会把容器中的webrtc源码拷贝到本地的～/code/webrtc
```

很明显，这样做有缺点：

1. 宿主机和容器中各有一份完全相同的源码，占据磁盘空间。
2. 这样会把**整份文件夹**（包括check用的metadata）拷贝到宿主机，其实没有必要。

关于第一点，笔者认为可以通过sambad等方式，将容器的源码挂在到本地实现（尚未验证）。因为本地不需要编译，理论不会对容器内的编译产生影响。

关于第二点，事实上只需要同步部分native代码。因此，不嫌麻烦，可以手动**只同步**特定文件夹。如`~/code/webrtc/src/pc/`。



其实第二点还有一个更好的解决方案，就是使用`rsync`。它支持**增量同步**和**指定同步/屏蔽特定文件**。可以完美解决我们的第二点需求。此处不做讨论。

#### 配置调试选项

如果需要调试Java和native，需要修改`debug type`为`Dual`。操作如下图：

![编辑配置](/images/edit-configurations.png)

![Debug Type选择Dual](/images/select-debug-type.png)



在LLDB Debugger启动前，还需要对源码文件进行映射（mapping）——因为Sdk是在容器中编译的，和宿主机上实际的位置并不一致。

按如下操作添加映射规则：

```bash
settings set target.source-map {符号文件中的路径} {宿主机中实际的路径}
```

![添加源文件映射规则](/images/mapping-source-files.png)

其中，**符号文件中的路径**，需要「开启Debug」+「进行一次呼叫然后挂断」，再按下图步骤，使用如下命令查看：

```bash
image lookup -vrn webrtc::PeerConnection::RestartIce
```

![查询符号文件中的路径](/images/check-mapping-dir.png)

> 注意到有一个有趣的现象：该应用在挂断时**会发生Crash**。开始不明白为什么，以为是个Bug，等到写此文时突然意识到：如果不crash，那我们恐怕无法拿到符号文件中的路径——因为没办法进行上图中第4步。

#### 调试Java

跟普通应用的调试方法一样，在需要调试的位置添加断点即可，如下图所示。

![java debug](/images/java-debug.png)

#### 调试native

通过Android Studio打开C++源码文件，然后在需要调试位置加上断点，再进入Debug。如下图所示：

![native debug](/images/native-debug.png)

## 结束语
非常感谢[@mthil](https://github.com/mthli)分享的经验！

## 参考资料
* [WebRTC Android development (googlesource.com)](https://webrtc.googlesource.com/src/+/main/docs/native-code/android/index.md)
* [depot tools tutorial(7) (commondatastorage.googleapis.com)](https://commondatastorage.googleapis.com/chrome-infra-docs/flat/depot_tools/docs/html/depot_tools_tutorial.html)
* [编译源码 - WebRTC 学习指南 (mthli.com)](https://webrtc.mthli.com/basic/webrtc-compilation/)
* [断点调试 - WebRTC 学习指南 (mthli.com)](https://webrtc.mthli.com/basic/webrtc-breakpoint/)
