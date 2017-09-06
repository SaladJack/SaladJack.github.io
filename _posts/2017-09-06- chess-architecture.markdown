---
layout:     post
title:      "一种MVC架构的改良方式"
tags:
    - 实习
---

在国际象棋项目里，核心对局采用常见的MVC架构，但由于对局逻辑相当复杂，如果不解耦就会导致C层代码非常多。由于大部分逻辑都是请求-响应的结构，所以便有了把请求（Cmd）和响应（Action）从C层文件中抽离出来的想法。

## 具体结构图如下：
![](http://ww1.sinaimg.cn/large/61340919ly1fj9x9gwjrij20ek0afjri.jpg)

## Cmd和Action的定义：

每一个单独的请求都对应一个CmdXXX类，如进房、坐下、等待、开始、退出、再来一局、走棋、放弃等，每一个CmdXXX是IChessCmd接口的实现，CmdXXX只需要定义DoCmd具体的逻辑即可。

![](http://ww1.sinaimg.cn/large/61340919ly1fj9xpvkgdxj20e703eq2w.jpg)

同理每一个请求的响应都对应一个ActionXXX类，每一个ActionXXX类是IChessAction接口的实现，ActionXXX只需要定义DoAction具体的逻辑即可。
![](http://ww1.sinaimg.cn/large/61340919ly1fj9xvcce6yj20f5039gll.jpg)

IChessCmd中DoCmd的形参为发出请求所需要的参数
IChessAction中DoAction的形参为请求响应的参数

##### Cmd和Action的目录结构
![](http://ww1.sinaimg.cn/large/61340919ly1fj9xn1vly4j2079099mx8.jpg)
![](http://ww1.sinaimg.cn/large/61340919ly1fj9xfbostmj2074092mx8.jpg)

## 在Controller初始化Cmd和Action

在Controller利用字典映射的方式，以key为枚举值，value为对应的Cmd或者Action，初始化_dictCmd和_dictAction。

![](http://ww1.sinaimg.cn/large/61340919ly1fj9xluri9ej20oh0g4jv0.jpg)

![](http://ww1.sinaimg.cn/large/61340919ly1fj9y1j1rfdj20mw0gqafj.jpg)


## Cmd和Action的执行

在Controller层定义DoCmd和DoAction方法，当发出一个请求是，比如要发走棋协议，就执行`ChessGameController.GetInstance().DoCmd(EnumChessCmd.eCmdMove, _srcX, _srcY, _dstX, _dstY, (int)selectType);`selectType为所走的棋子类型。当协议回包到达时，Controller就调用`DoAction(EnumChessAction.eActionMove) `走回包响应的逻辑。

![](http://ww1.sinaimg.cn/large/61340919ly1fj9y2xszvoj20ef0ag3z7.jpg)

在项目中，Controller有一个自己的消息接收队列，每一帧轮询队列，如果队列有信息，将信息pop出来，并执行对应的Action操作。



## 优缺点

优点：

1、减少C层代码

2、结构清晰，适合涉及多协议交互的逻辑

3、在项目后期有新成员加入，CodeReview的时候只需要花10分钟讲解一下这种架构，新成员都反馈说这种架构很清晰，一听就懂，学习成本很低。

缺点：

1、不适合简单协议交互的逻辑，比如个人信息界面，只涉及拉取个人信息的逻辑，不需要也不建议用这种架构









