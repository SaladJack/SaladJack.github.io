---
layout:     post
title:      "轻量级的Java事件总线的实现"
tags:
    - Java
---

具体代码见：https://gist.github.com/SaladJack/fdae3bec2b6589f02287de29bd415960


使用方法：
以Android为例，注册和注销回调

```java
public class TestActivity extends Activity {
    @Override protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        MessageBus.addListener(MessageID.TestDemo,this::handle0);
        MessageBus.addListener(MessageID.TestDemo1,this::handle1);
        MessageBus.addListener(MessageID.TestDemo2,this::handle2);
    }

    @Override protected void onDestroy() {
        super.onDestroy();
        MessageBus.removeListener(MessageID.TestDemo,this::handle0);
        MessageBus.removeListener(MessageID.TestDemo1,this::handle1);
        MessageBus.removeListener(MessageID.TestDemo2,this::handle2);
    }
    
    void handle0() {
        System.out.println("no param");
    }

    void handle1(String para1){
        System.out.println(para1);
    }

    void handle2(String para1,String para2){
        System.out.println(para1+para2);
    }

    void handle3(String para1,String para2,String para3){
        System.out.println(para1+para2+para3);
    }

 
}
```

然后在需要触发回调的地方调用事件broadcast方法

```java
    MessageBus.broadCast(MessageID.TestDemo); 
    MessageBus.broadCast(MessageID.TestDemo1,"hello");
    MessageBus.broadCast(MessageID.TestDemo2,"hello","world");
    MessageBus.broadCast(MessageID.TestDemo2,"hello","world","haha!");
```

因为该事件总线是以字符串来唯一标识事件的，所以建议建立一个MessageID的类来管理字符串事件ID。

```java
class MessageID {
    public static final String TestDemo = "TestDemo";
    public static final String TestDemo1 = "TestDemo1";
    public static final String TestDemo2 = "TestDemo2";
}
```

注意：

1、如果没注册(addListener)事件就广播(broadcast)出去，将不会有任何作用。

2、支持一个事件广播给多个回调，只要保证事件ID都是相同即可。

3、传参暂时支持0、1、2、3个，要支持更多数量的参数，可以仿照源码自己扩展。








