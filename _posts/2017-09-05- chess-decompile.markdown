---
layout:     post
title:      "实习之竞品分析"
tags:
    - 实习
---

在腾讯IEG实习的时候，项目里是要做一个类似竞品的习题集需求，但是组里没有习题资源，按照竞品的题目一个个拼也不现实，所以便有了扒取竞品Apk数据的想法。

注：组里大部分人对安卓原生不是很了解，了解的可能也对逆向分析不是很熟悉，之前因为兴趣我学过一点逆向的东西，然后主动要求把这工作让我试一下，虽然现在看来这真的是个大坑哈哈。



竞品效果见下图
![](http://ww1.sinaimg.cn/large/61340919ly1fj9236lrdej20ku112qfn.jpg)

![](http://ww1.sinaimg.cn/large/61340919ly1fj923crklwj20ku112te7.jpg)


逆向一开始还是老套路，ApkTool、JD-GUI走起，先大致浏览一遍逆出来的AndroidManifest，对Activity界面的名称有个大致的了解。然后再用JD-GUI大致浏览一遍各个文件包名。

因为要扒的是棋盘数据，在确定数据是存在本地的情况下，本能的看看databases有啥，结果发现有个叫book.db的东西。
![](http://ww1.sinaimg.cn/large/61340919gy1fj92cj05utj20hz02ldg1.jpg)

用sqlite命令打开看看都有什么表
![](http://ww1.sinaimg.cn/large/61340919gy1fj92fn1s4vj20h201wq2w.jpg)

其中horse、kill、polgar、tactical是我们想要的习题集数据。

看看polgar存的数据好了，结果发现数据一堆乱码
![](http://ww1.sinaimg.cn/large/61340919gy1fj92i0szekj20be0e5t9c.jpg)

发现其中两个字段_fen和_pgn都是BLOB（fen和pgn分别是国际象棋里棋谱和答案的专业名称）
![](http://ww1.sinaimg.cn/large/61340919gy1fj92l2jyx0j208e04f74d.jpg)

一开始以为是编码问题，所以把book.db放到另外一个apk，试着用UTF-16,UTF-8, ASCII等各种编码试了下，发现没什么用，导出的依旧是乱码，所以猜测数据里应该是做了加密。

随后开始看代码，先通过JD-GUI找到对应读取DB的代码地方，具体定位方法最快的方法是直接搜字符串“_fen”或“_pgn”。

![](http://ww1.sinaimg.cn/large/61340919gy1fj92pxkmwtj20ma09n42a.jpg)

以获取fen为例，在拿到fen的BLOB数据后，传到了localj对象的b方法，b方法最终会调用如下a方法，a方法会转化成一个v对象。

![](http://ww1.sinaimg.cn/large/61340919gy1fj92tgitebj20fj0h5407.jpg)

代码各种位运算实在让人头疼，一开始是打算在smali代码注入日志，然后重新打包，但是

1.这个v对象有很多参数，到底哪个参数有用以及怎么用还不确定 
，虽然看代码能看出个大概，但不好把握。

2.竞品对重新打包做了处理，一直都打包失败。

3.通过阅读代码可以看出棋盘界面是刷出来的时候才读db对应的数据，1w多道题如果通过日志刷出的话，还要把每一页都滑动一下才能走到读对应db的逻辑，不太现实。

随后想试试调试看看有没有什么新发现。

关于调试，我用的是AndroidStudio调试smali代码的方法。

具体环境搭建可参照[Smalidea+IntelliJ IDEA/Android Studio动态调试安卓app教程](http://blog.csdn.net/linchaolong/article/details/51146492)

手机打开到6个棋盘展示的界面，然后通过
`adb shell dumpsys activity | grep "mFocusedActivity"`
获取当前界面对应的Activity名，然后找到对应代码，跟踪相应的逻辑（此处略）在棋盘初始化的时候加个断点，然后调试，意外的发现了一个fen字符串。

![](http://ww1.sinaimg.cn/large/61340919gy1fj934yrhzrj20ya0ljgrl.jpg)

仿佛看到了曙光，对比了下，这个j就是对应的之前说的v类的对象，打开j发现有个长度64的数组。

![](http://ww1.sinaimg.cn/large/61340919gy1fj93dd1qpxj20750ayt8u.jpg)

很容易想到棋盘就是64个，很有可能跟这有关系，经过逐一比对，发现数据是以棋盘左下角为原点建立的数组，0代表无棋，1代表白王，2代表白后。。然后7代表黑王，8代表黑后，因为国象一共就6种棋型，所以这种就是x白、x+6黑的编码方式，以此内推可以推出马、象、车、兵的id。

之后把BLOB数据转换成16进制排布，发现数据其实就是利用棋子id和奇偶对调的方式处理的，比如如果有数据0201，其实它的在棋盘的排布是2010，也就是第1个格子是白后，第3个格子是白王。考虑到版权问题，这部分细节就不公开了。

上述说的是获取棋盘信息fen的过程，还有一个就是获取答案pgn的过程。pgn在代码里的使用是通过坐标转换的，BLOB的数据转化规律定位到了如下函数：

![](http://ww1.sinaimg.cn/large/61340919gy1fj93rorw8xj208q03c3yi.jpg)

l的构造函数有三个形参a,b,c（从左至右），分别代表原坐标、目标坐标、兵升变参数。

整体逆向的感受还是挺麻烦的，这个过程会遇到很多坑，比如长期看混淆的代码会被一些函数调用绕的晕头转向，不过之后也懂得放弃这种死磕代码的做法，换一种角度看问题，尝试调试看看能看出什么东西来，不然可能这个项目的习题获取就一直做不成了。还有学过的东西肯定会有用的，虽然工作用的都是unity和c#，跟原生Android已经相去甚远，但学过的以后肯定会有用武之地，所以学东西不能一味的太功利化，提升自己能力才是关键。


![](http://ww1.sinaimg.cn/large/61340919gy1fj94du55qsj20ph08yter.jpg)

国象已经上线应用宝了，见证了项目从0到1的过程还是会有很多感慨啊~











