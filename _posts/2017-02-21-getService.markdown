---
layout:     post
title:      "getService()的过程"
tags:
    - Android源码
---

- **ServiceManagerProxy **

当某个Binder Server在启动时，会把自己的名称name与对应的Binder句柄值保存在ServiceManager中，调用者通常只知道Binder Server的名称，所以必须先向ServiceManager发起查询请求，就是getService(name)。

而ServiceManager自身也是个Server，就好像互联网上的DNS服务器本身也需要提供IP地址才能访问一样，只不过这个IP地址是预先就设定了的（句柄值为0），因而任何Binder Client都可以直接通过0这个Binder句柄创建一个BpBinder，再通过Binder驱动去获取SM的服务。具体而言，就是调用BinderInternal.getContextObject()来获得ServiceManager的Binder代理，即BpBinder。

BpBinder在Java层以IBinder来表示。对于ServiceManager而言，IBinder的真正持有者与使用者是ServiceManagerProxy。

- **ProcessState和IPCThreadState**

ProcessState是进程相关，IPCThreadState是线程相关，ProcessState负责打开Binder驱动设备，进行mmap()等准备工作，而如何与Binder驱动进行具体的命令通信则由IPCThreadState来完成

在getService()这个场景中，调用者是从Java层的Ibinder.transact()开始，层层往下调用到IPCThreadState.transact()，然后通过waitForResponse进入主循环，直至收到Service Manager的回复后才跳出循环，并将结果再次层层回传到应用层。

真正与Binder驱动打交道的地方是talkWithDriver中的ioctl()，整个流程中多次调用了这个函数（具体有什么指令以及有什么作用参见文末附录1）。

- **Binder驱动**

在这个场景中，主要涉及了Binder驱动提供的binder_ioctl，binder_thread_write，binder_thread_read和binder_transaction等几个重要的函数实现。Binder驱动通过巧妙的机制来时数据的传递更加高效，即只需要一次复制就能实现两个进程间的通信。Binder中还保存着大量的全局以及进程相关的变量，用于管理每个进程/线程的状态一些列复杂的数据信息。一次复制的原理和一些数据结构见文末附录2、3。

- **Service Manager的实现**

ServiceManager在Android系统启动之后就运行起来了，并通过BINDER_SET_CONTEXT_MGR把自己注册成Binder的管理者，它在做完一系列初始化后，在最后一次ioctl的read操作中会进入*睡眠等待*，知道有Binder Client发起服务请求而被Binder驱动*唤醒*。

ServiceManager唤醒后，程序分为两条主线索。

其一，ServiceManager端将接着执行read操作，把调用者的具体请求读取出来，然后利用binder_parse解析，再根据实际情况填写transaction信息，最后把结果通过BR_REPLY命令（也是ioctl）返回Binder驱动。

其二，发起getService请求的Binder Client在等待ServiceManager回复的过程中会进入休眠，知道被Binder驱动再次唤醒——它和ServiceManager一样也是在read中睡眠的，因而醒来后继续执行读取操作，这一次得到的就是ServiceManager对请求的执行结果。程序先把结果填充到reply这个Parcel中，然后通过层层返回到ServiceManagerProxy，在利用Parcel.readStrongBinder生成一个BpBinder，最终经过类型转换为IBinder对象后传给调用者。

- **调用者**

整个ServiceManager.getService()的大体流程分析完了。但对于调用者来说，得到的IBinder对象要先经过asInterface做一次包装，如在native层ServiceManager的BpBinder就被包装成了IServiceManager（实际上就是一个ServiceManagerProxy）。这么做的目的在于可以让应用程序更加方便的使用Service Manager提供的服务（其他Binder Server也类似）。


##### 附录1
![binder_ioctl支持的命令](http://ww1.sinaimg.cn/large/61340919ly1fcyg3qhx71j20pk08d41y)

![BINDER_WRITE_READ中支持的子命令](http://ww1.sinaimg.cn/large/61340919ly1fcyg4j35hnj20lf0p57bh)


##### 附录2

Server和Binder驱动用到了进程虚拟地址空间（vm_area_struct）和内核虚拟地址空间（vm_struct）,在Linux中，vm_area_struct和vm_struct，,都映射到同一个物理页面，当内核空间的数据发生变化时，即这个物理页面发生变化，所以无需再传递数据。思想就是共享内存来节省数据拷贝次数。

##### 附录3
binder_proc是native层中在service_manager.c的main函数里初始化经常要用到的数据结构，它用于保存/dev/binder进程的上下文，其中我们看看下面四个变量的含义

![binder_proc](http://ww1.sinaimg.cn/large/61340919ly1fcyg6pnyq1j20b10cr407)

rb_root即red-black-root，红黑树的根节点

threads:保存binder_proc进程内用于处理用户请求的线程

nodes: 保存binder_proc进程内的Binder实体

refs_by_desc:保存binder_proc进程内的Binder引用，即引用其它进程的Binder实体，以句柄作key值来组织红黑树的构建

refs_by_node:也是保存对其它进程的Binder引用，不过这个是以引用的实体节点的地址值来作key值组织的



