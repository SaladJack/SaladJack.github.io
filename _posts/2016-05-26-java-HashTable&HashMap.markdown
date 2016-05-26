---
layout:     post
title:      "HashTable和HashMap的区别"
subtitle:   "Java基础"
date:       2016-05-26
author:     "SaladJack"
header-img: "img/in-post/java.png"
tags:
    - Java
---

废话不多说，直入正题。

### 结论
关于HashTable和HashMap的区别，为了方便叙述，我们先陈述结论：

|HashTable|HashMap|
| -----|:----:| ----:|
|线程安全|非线程安全|
|不允许null|允许null(key只允许一个取null)|
|效率低|效率高|
|sychronized|无sychronized|
|11,old\*2\+1|16,空间左移1位至不小于实际需要的长度|

说明：第三行的效率是特指多线程环境中，第四行代表使用的机制，第五行第一个数代表默认开辟的长度,数字后面代表空间不够时新增空间的规则。

对结论的补充和扩展：

HashTable是基于synchronize操作的，sychronized锁住对象整体，所以HashTable是同步的，而HashMap（没有基于sychronized）不是。

ConcurrentHashMap是基于lock操作，lock锁住的不是对象整体，所以ConcurrentHashMap是同步的（是HashMap线程安全的实现），且性能比HashTable好。

ConcurrentHashMap基于concurrentLevel划分出多个Segment来对key-value进行存储，从而避免每次锁定整个数组，在默认的情况下，允许16个线程并发无阻塞的操作集合对象，尽可能地减少并发是的阻塞现象。



###原理分析

首先看HashTable和HashMap的定义


```java
public class HashTable extends Dictionary implements Map,Cloneable,java.io.Serializable

public class HashMap extends AbstractMap implements Map,Cloneable,java.io.Serializable
```

可见HashTable 继承自 Dictionary(较陈旧) 而 HashMap继承自AbstractMap


HashTable的put方法

```java
public synchronized V put(K key, V value) {  //###### 注意这里1  
  // Make sure the value is not null  
  if (value == null) { //###### 注意这里 2  
    throw new NullPointerException();  
  }  
  // Makes sure the key is not already in the hashtable.  
  Entry tab[] = table;  
  int hash = key.hashCode(); //###### 注意这里 3  
  int index = (hash & 0x7FFFFFFF) % tab.length;  
  for (Entry e = tab[index]; e != null; e = e.next) {  
    if ((e.hash == hash) && e.key.equals(key)) {  
      V old = e.value;  
      e.value = value;  
      return old;  
    }  
  }  
  modCount++;  
  if (count >= threshold) {  
    // Rehash the table if the threshold is exceeded  
    rehash();  
    tab = table;  
    index = (hash & 0x7FFFFFFF) % tab.length;  
  }  
  // Creates the new entry.  
  Entry e = tab[index];  
  tab[index] = new Entry(hash, key, value, e);  
  count++;  
  return null;  
}  
```

注意1 方法是同步的(因为使用了synchronized)

注意2 方法不允许value==null

注意3 方法调用了key的hashCode方法，如果key==null,会抛出空指针异常 HashMap的put方法如下

HashMap的put方法

```java
public V put(K key, V value) { //###### 注意这里 1  
  if (key == null)  //###### 注意这里 2  
    return putForNullKey(value);  
  int hash = hash(key.hashCode());  
  int i = indexFor(hash, table.length);  
  for (Entry e = table[i]; e != null; e = e.next) {  
    Object k;  
    if (e.hash == hash && ((k = e.key) == key || key.equals(k))) {  
      V oldValue = e.value;  
      e.value = value;  
      e.recordAccess(this);  
      return oldValue;  
    }  
  }  
  modCount++;  
  addEntry(hash, key, value, i);  //###### 注意这里   
  return null;  
}  
```

注意1 方法是非同步的

注意2 方法允许key==null

注意3 方法并没有对value进行任何调用，所以允许为null

由于HashTable线程安全，而HashMap非线程安全，在多线程环境中，开销上显然HashTable比HashMap大，所以也就有HashTable效率低，HashMap效率高的结论了。

关于其他结论和ConcurrentHashMap的介绍，参见[此文](http://blog.csdn.net/tgxblue/article/details/8479147)。