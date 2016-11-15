---
layout:     post
title:      "Android启动流程(1)"
tags:
    - Android源码
---
《Android安全技术揭秘与防范》读书笔记

---

在讲述Android启动过程前，先简单介绍一下Android的层次结构和分区结构
## Android系统的层次结构

整体架构分为如下5层
- 应用层(Application)
- 框架层(Framework)
- 核心库与运行环境层(Libraries & Android Runtime)
- Linux内核层(Linux Kernel)

## Android系统的分区结构

分区是逻辑层存储单位用来区分设备内部的永久性存储结构。不同厂商和平台有不同的分区布局。然而，有几个分区最常见，即Boot、Data、Recovery和Cache分区。通常的情况下NAND闪存的设备都具备一下分区布局

- Boot Loader分区

  中文名称“系统加载区”，它的作用相当于电脑的BIOS，在手机进入系统之前初始化软硬件环境、加载硬件设备，最终让手机成功启动。
- Boot分区

  存储着Android的Boot镜像，其中包含着Linux kernel(zImage)与initrd等文件。

- Splash分区

  主要是存储系统启动后第一屏显示的内容，一般都是一些公司的Logo或者动画，存储在Boot Loader中。

- Radio分区

  这个事基带所在的分区，存储着一些与通信质量相关的Linux驱动，如电话、GPS等。常用的驱动是可以打包存在于Linux的内核Boot分区，但为了提升设备的通信质量，所以单独开辟了Radio分区。

- Recovery分区

  存储着一个mini型的Android Boot镜像文件，主要的作用是用来做故障维修和系统恢复。

- System分区

  存储着Android系统的镜像文件，镜像文件中包含着Android的Framework、Libraries、Binaries和一些预装应用。系统挂载后既/system目录

- User Data分区

  也称为数据分区，它是设备的内部存储分区，如应用产生的图片、声音等数据文件。系统挂载后在/data目录下

- Cache区

  用于存储各种使用的文件，如回复日志和OTA下载的更新包。在应用程序安装在SD卡上时，它也可能包含Dalvik缓存文件夹，其中存储了Dalvik虚拟机的缓存文件。


## Android启动流程
主要分为6个阶段：BootLoader加载阶段、加载Kernel与initrd阶段、初始化设备服务阶段、加载系统服务阶段、虚拟机初始化阶段、启动完成阶段。

![Android系统启动流程](/img/201610/android-start-process.png)

- Boot Loader阶段
Boot Loader是在物理电源按下之后第一个加载的。在此阶段会运行一些制造商自定义的初始化代码。Boot Loader内部也分为多个阶段，对此不做深入讨论。

- 加载Kernel与initrd阶段
Boot分区加载Linux kernel与initrd到RAM，最后跳转到Kernel继续完成启动。

- 初始化设备服务阶段
Android kernel则会启动所有Android系统设备所必须的服务，如初始化Memory、初始化IO、内存保护、中断处理程序、CPU调度、设备驱动，最后还会挂载文件系统，启动第一个用户进程init。

- 加载系统服务阶段
init是Linux系统中用户空间的第一个进程，其进程PID是1，父进程为Linux Kernel核的0号进程。init具有特殊的初始化使命，它会加载一个初始化脚本文件init.rc，启动Android系统的一些核心服务，如针对通话的rild、针对VPN链接的mtpd、提供adb相关功能adbd、支持存储外设的热插拨功能vold、负责进程孵化服务的Zygote、Service Manager等。

- 虚拟机初始化阶段
其中启动的Zygote进程会创建Dalvik VM，会启动第一个Java组件系统服务，最后是Android Framework服务，如Activity Manager、Package Manager、Window Manager

- 启动完成阶段
当系统完成启动之后，载入Home（桌面应用程序），然后做一些应用层的初始化的工作，如播放一个全局的广播ACTION_BOOT_COMPLETED.
- LinkedHashMap基于链表存储的，移动节点的操作复杂度仅为O(1)，这就是LruCache选择LinkedHashMap作为存储结构的原因。
