---
title: Ubuntu18.0.4.1编译openjdk8
subtitle: 之前写过在mac os中编译openjdk 8,但是因为使用不是官网的代码，编译过中遇到错误也会修改源代码，可能会导致jvm莫名其妙的crash非常痛苦。实践证明，要研究jvm最好还是在linux中编译和实践。
cover: /images/jdk.jpg
author: 
  nick: 诣极
  link: https://github.com/zonghaishang
tags:
 - jvm
 - openjdk
categories:
 - jvm
date: 2018-10-26 14:09:19
---

## Ubuntu18.0.4.1 编译openjdk 8

之前写过在`mac os`中编译`openjdk 8`,但是因为使用不是官网的代码，编译过中遇到错误也会修改源代码，可能会导致jvm莫名其妙的crash非常痛苦。实践证明，要研究jvm最好还是在linux中编译和实践。

这里要非常感谢` Debian `系统开发者(`
John Paul Adrian Glaubitz`)提供的帮助。

### 概述

这次安装采用最新的Ubuntu18.0.4.1LTS版本，直接百度下载对应的操作系统，然后在虚拟机安装即可，我采用的是`VMware Fusion`作为虚拟机，比`Virtualbox`稳定一些。

在上面虚拟机安装完成的基础上，先更新国内源(用于加速ubuntu软件下载速度)。

在替换操作系统默认源前，建议先备份：

```shell
sudo mv  /etc/apt/sources.list  /etc/apt/sources.list.bak

# 如果没有安装vim, 执行`sudo apt-get install vim`
sudo vim /etc/apt/sources.list
```

在新创建的文件`sources.list`加入下面地址：

```
# 中科大
deb https://mirrors.ustc.edu.cn/ubuntu/ bionic main restricted universe multiverse
deb-src https://mirrors.ustc.edu.cn/ubuntu/ bionic main restricted universe multiverse
deb https://mirrors.ustc.edu.cn/ubuntu/ bionic-updates main restricted universe multiverse
deb-src https://mirrors.ustc.edu.cn/ubuntu/ bionic-updates main restricted universe multiverse
deb https://mirrors.ustc.edu.cn/ubuntu/ bionic-backports main restricted universe multiverse
deb-src https://mirrors.ustc.edu.cn/ubuntu/ bionic-backports main restricted universe multiverse
deb https://mirrors.ustc.edu.cn/ubuntu/ bionic-security main restricted universe multiverse
deb-src https://mirrors.ustc.edu.cn/ubuntu/ bionic-security main restricted universe multiverse
deb https://mirrors.ustc.edu.cn/ubuntu/ bionic-proposed main restricted universe multiverse
deb-src https://mirrors.ustc.edu.cn/ubuntu/ bionic-proposed main restricted universe multiverse

# 阿里
deb http://mirrors.aliyun.com/ubuntu/ bionic main restricted universe multiverse
deb http://mirrors.aliyun.com/ubuntu/ bionic-security main restricted universe multiverse
deb http://mirrors.aliyun.com/ubuntu/ bionic-updates main restricted universe multiverse
deb http://mirrors.aliyun.com/ubuntu/ bionic-proposed main restricted universe multiverse
deb http://mirrors.aliyun.com/ubuntu/ bionic-backports main restricted universe multiverse
deb-src http://mirrors.aliyun.com/ubuntu/ bionic main restricted universe multiverse
deb-src http://mirrors.aliyun.com/ubuntu/ bionic-security main restricted universe multiverse
deb-src http://mirrors.aliyun.com/ubuntu/ bionic-updates main restricted universe multiverse
deb-src http://mirrors.aliyun.com/ubuntu/ bionic-proposed main restricted universe multiverse
deb-src http://mirrors.aliyun.com/ubuntu/ bionic-backports main restricted universe multiverse
```

最后在终端执行更新：

```shell
sudo apt-get update
```

根据` Debian `系统开发者(`
John Paul Adrian Glaubitz`)的建议，使用如下命令安装jdk环境所有依赖：

```shell
sudo apt build-dep openjdk-8
```

这个会自动下载需要的依赖，也会安装openjdk。

### 安装准备

#### 源码下载

这次采用官方源码安装，官网地址：`http://openjdk.java.net/projects/jdk8u/`:

```
hg clone http://hg.openjdk.java.net/jdk8u/jdk8u
cd jdk8u
bash get_source.sh
```

可以反复执行`bash get_source.sh`知道下载所有子模块的源代码。

### 安装依赖

* boot 启动jdk

我在线尝试各种安装openjdk7作为引导都失败，所以执行手动安装，可以下载这里面的附件：

- [openjdk-7-jdk_7u161-2.6.12-1_amd64.deb](https://zonghaishang.github.io/attachments/openjdk-7-jdk_7u161-2.6.12-1_amd64.deb)
- [openjdk-7-jre_7u161-2.6.12-1_amd64.deb](https://zonghaishang.github.io/attachments/openjdk-7-jre_7u161-2.6.12-1_amd64.deb)
- [openjdk-7-jre-headless_7u161-2.6.12-1_amd64.deb](https://zonghaishang.github.io/attachments/openjdk-7-jre-headless_7u161-2.6.12-1_amd64.deb)
-  [libjpeg62-turbo_1.5.2-2+b1_amd64.deb](https://zonghaishang.github.io/attachments/libjpeg62-turbo_1.5.2-2+b1_amd64.deb)

上面4个文件下载后，在包含这个目录下面执行：

```shell
sudo dpkg -i openjdk-7-* libjpeg62-turbo*
```

如果报错如下：

```
Errors were encountered while processing:
openjdk-7-jre:amd64
openjdk-7-jre-headless:amd64
openjdk-7-jdk:amd64
```

再执行下面命令安装缺少的依赖即可：

```
sudo apt install -f
```

这个时候可以切换到jdk7的版本：

```
sudo update-java-alternatives -s java-1.7.0-openjdk-amd64
```

我手动安装openjdk7的执行结果如下：

```
yiji@ubuntu:~$ java -version
java version "1.7.0_161"
OpenJDK Runtime Environment (IcedTea 2.6.12) (7u161-2.6.12-1)
OpenJDK 64-Bit Server VM (build 24.161-b01, mixed mode)
```

ubuntu中可以使用如下指令快速切换不同版本的jdk:

```
 yiji@ubuntu:~$ sudo update-alternatives --config java
[sudo] password for yiji: 
There are 3 choices for the alternative java (providing /usr/bin/java).

  Selection    Path                                            Priority   Status
------------------------------------------------------------
  0            /usr/lib/jvm/java-11-openjdk-amd64/bin/java      1101      auto mode
  1            /usr/lib/jvm/java-11-openjdk-amd64/bin/java      1101      manual mode
* 2            /usr/lib/jvm/java-7-openjdk-amd64/jre/bin/java   1071      manual mode
  3            /usr/lib/jvm/java-8-openjdk-amd64/jre/bin/java   1081      manual mode

Press <enter> to keep the current choice[*], or type selection number: 

```

- 设置环境变量

在`~/.bashrc`文件最后一行添加`export LANG=C`， 然后执行`source ~/.bashrc`生效。

## 开始编译

因为在linux环境中编译，唯一改动的文件（`~/jdk8u/hotsport/make/linux/makefiles/gcc.make`）：

```
# Compiler warnings are treated as errors
# WARNINGS_ARE_ERRORS = -Werror
```

注释了这个脚本，主要处理忽略警告，否则警告会当做错误处理导致无法编译。

注意，我代码迁出在自己的`home`目录, 改动文件要进入自己clone代码的位置。

### 生成配置

为了方便调试jvm，我开启了`--enable-debug`会生成`fastdebug`版本的jdk:

```
MAKE_VERBOSE=y QUIETLY= LOG=debug sh ./configure --enable-debug --with-jvm-variants=server --with-boot-jdk=/usr/lib/jvm/java-7-openjdk-amd64/ --disable-precompiled-headers --with-freetype-include=/usr/include/freetype2/ --with-freetype-lib=/usr/lib/x86_64-linux-gnu
```

 执行命令后我电脑输出：

```
====================================================
A new configuration has been successfully created in
/home/yiji/jdk8u/build/linux-x86_64-normal-server-fastdebug
using configure arguments '--enable-debug --with-jvm-variants=server --with-boot-jdk=/usr/lib/jvm/java-7-openjdk-amd64/ --disable-precompiled-headers --with-freetype-include=/usr/include/freetype2/ --with-freetype-lib=/usr/lib/x86_64-linux-gnu'.

Configuration summary:
* Debug level:    fastdebug
* JDK variant:    normal
* JVM variants:   server
* OpenJDK target: OS: linux, CPU architecture: x86, address length: 64

Tools summary:
* Boot JDK:       java version "1.7.0_161" OpenJDK Runtime Environment (IcedTea 2.6.12) (7u161-2.6.12-1) OpenJDK 64-Bit Server VM (build 24.161-b01, mixed mode)  (at /usr/lib/jvm/java-7-openjdk-amd64)
* Toolchain:      gcc (GNU Compiler Collection)
* C Compiler:     Version 7.3.0 (at /usr/bin/gcc)
* C++ Compiler:   Version 7.3.0 (at /usr/bin/g++)

Build performance summary:
* Cores to use:   3
* Memory limit:   3921 MB

WARNING: The result of this configuration has overridden an older
configuration. You *should* run 'make clean' to make sure you get a
proper build. Failure to do so might result in strange build problems.
```

### 生成jdk8

执行下面命令进行构建jdk:

```
make JOBS=8 MAKE_VERBOSE=y QUIETLY= LOG=debug all
```

生成openjdk8成功会输出耗时信息：

```
## Finished docs (build time 00:02:17)

----- Build times -------
Start 2018-10-26 13:36:24
End   2018-10-26 13:51:32
00:00:24 corba
00:00:19 demos
00:02:17 docs
00:07:56 hotspot
00:00:23 images
00:00:14 jaxp
00:00:20 jaxws
00:02:32 jdk
00:00:29 langtools
00:00:14 nashorn
00:15:08 TOTAL
-------------------------
Finished building OpenJDK for target 'all'
```

## 使用openjdk8

1. 生成的jdk Home 在源码目录build ：

```
~/jdk8u/build/linux-x86_64-normal-server-fastdebug/images/j2sdk-image
```

直接在intellij idea 或者 eclipse 中指定上面的Home即可。

2. 验证jdk版本

```
yiji@ubuntu:~/jdk8u/build/linux-x86_64-normal-server-fastdebug/jdk/bin$ ./java -version
openjdk version "1.8.0-internal-fastdebug"
OpenJDK Runtime Environment (build 1.8.0-internal-fastdebug-yiji_2018_10_26_13_36-b00)
OpenJDK 64-Bit Server VM (build 25.71-b00-fastdebug, mixed mode)
```

### 为什么要编译JDK源码

1. 已发布jdk版本去除了调试信息和运行时信息，降低内存占用提升运行速度，但是不适合开发者调试jdk代码

2. 深入jvm细节，自己动手编译为深入学习打基础


### 参考
1. http://openjdk.java.net/projects/jdk8u/
2. https://askubuntu.com/questions/761127/how-do-i-install-openjdk-7-on-ubuntu-16-04-or-higher
3. https://bugs.openjdk.java.net/browse/JDK-8144695
4. https://marcin-chwedczuk.github.io/debugging-openjdk8-with-netbeans-on-ubuntu
5. http://hg.openjdk.java.net/jdk8u/jdk8u/raw-file/tip/README-builds.html#linux
6. https://blog.csdn.net/xiangxianghehe/article/details/80112149