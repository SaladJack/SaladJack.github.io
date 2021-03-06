---
layout:     post
title:      "LruCache源码解析"
author:     "SaladJack"
header-img: "img/aidl-bg.jpg"
tags:
    - Android源码
---

## 初始化

我们先看LruCache初始化的实现

```java
public class LruCache<K, V> {
       private int size; // 当前缓存大小
       private int maxSize;// 允许缓存的最大值
       private int putCount;// 放入缓存数
       private int createCount;// 创建数据数
       private int evictionCount;// 移除数目
       private int hitCount;// 命中数
       private int missCount;// miss数
       public LruCache(int maxSize) {
           if (maxSize <= 0) {
               throw new IllegalArgumentException("maxSize <= 0");
           }
           this.maxSize = maxSize;
           //此处第三个参数accessOrder设为true，用法后面阐述
           this.map = new LinkedHashMap<K, V>(0, 0.75f, true);
       }

       ...
       protected int sizeOf(K key, V value) {
         return 1;
       }
   }
```
参数含义后已含有注释，值得注意的是
- LruCache底层数据结构为LinkedHashMap
- 在继承了该类的时候，要重写 sizeOf 方法来确定每一个元素的大小，默认返回是1

## put方法

接下来我们再看put方法

```java
public final V put(K key, V value) {
        if (key == null || value == null) {
            throw new NullPointerException("key == null || value == null");
        }

        V previous;
        synchronized (this) {
            putCount++;
            // 增加当前缓存，safeSizeOf：求该key-value的大小
            size += safeSizeOf(key, value);
            // 缓存到map中，并获取是否有冲突的值
            previous = map.put(key, value);
            if (previous != null) {
              // 该元素已存在，返回之前的value值，之前增加的当前缓存大小要减去
                size -= safeSizeOf(key, previous);
            }
        }

        if (previous != null) {
          // 该方法提供给使用者自定义是覆盖还是取消（该方法在LruCache内是空实现）
            entryRemoved(false, key, previous, value);
        }
        // 检查大小并调整
        trimToSize(maxSize);
        // 返回冲突值
        return previous;
    }
```
从代码中我们可以看出以下几点
- LruCache的Key和Value非空
- put操作是线程安全的
- 如果key-value之前已存在，则返回冲突对应的value值
- Lru算法的的具体实现在trimToSize方法中

## trimToSize



```java
public void trimToSize(int maxSize) {
  	// 首先进入一个循环
    while (true) {
        K key;
        V value;
        synchronized (this) {
          	// 如果当前的容量小于0，或者实际为空但 size不为0 抛异常
            if (size < 0 || (map.isEmpty() && size != 0)) {
                throw new IllegalStateException(getClass().getName()
                        + ".sizeOf() is reporting inconsistent results!");
            }
			// 如果当前大小小于最大容量，跳出循环，不需要调整
            if (size <= maxSize) {
                break;
            }
      // 到这一步则说明当前大小已经大于了允许的最大容量
			// 找到缓存中最老的那个元素
            Map.Entry<K, V> toEvict = map.eldest();
            if (toEvict == null) {
                break;
            }
			      // 从集合移除这个元素，重新计算当前大小，移除的数量＋1
          	// 进入下一次循环判断
            key = toEvict.getKey();
            value = toEvict.getValue();
            map.remove(key);
            size -= safeSizeOf(key, value);
            evictionCount++;
        }
        entryRemoved(true, key, value, null);
    }
}
```

- 这个方法就是根据允许的最大容量来调整存储的元素，是否需要清理掉。


## get方法

我们再来看看get方法

```java
public final V get(K key) {
    if (key == null) {
        throw new NullPointerException("key == null");
    }
    V mapValue;
    synchronized (this) {
      	// 从缓存中找元素
        mapValue = map.get(key);
        if (mapValue != null) {
          	// 找到后 命中数＋1 返回这个元素
            hitCount++;
            return mapValue;
        }
      	// 没找到就把 miss 数＋1
        missCount++;
    }
	// 以下的代码会试图创建一个数据返回
    /*
     * Attempt to create a value. This may take a long time, and the map
     * may be different when create() returns. If a conflicting value was
     * added to the map while create() was working, we leave that value in
     * the map and release the created value.
     */
	// 如果创建的元素为 null 直接返回 null
    V createdValue = create(key);
    if (createdValue == null) {
        return null;
    }
    synchronized (this) {
      	// 说明创建元素成功
        createCount++;
      	// 添加到缓存中
        mapValue = map.put(key, createdValue);

        if (mapValue != null) {
            // There was a conflict so undo that last put
          	// 发生了冲突，则回滚操作，留给使用者自己处理
            map.put(key, mapValue);
        } else {
          	// 计算当前大小
            size += safeSizeOf(key, createdValue);
        }
    }
    if (mapValue != null) {
      	// 发生了冲突，交给调用者处理
        entryRemoved(false, key, createdValue, mapValue);
        return mapValue;
    } else {
      	// 添加缓存成功，重新判断大小并处理
        trimToSize(maxSize);
        return createdValue;
    }
}
```
具体的逻辑已在注释注明，而在Lru算法中，已存在的项会被调到链表尾部，那这部分逻辑在哪里呢？

显然在map.get(key)这个操作上

我们看LinkedHashMap中的get方法

```java
public V get(Object key) {
        Node<K,V> e;
        if ((e = getNode(hash(key), key)) == null)
            return null;
        //这里accessOrder在LruCache初始化时已设为true
        if (accessOrder)
            afterNodeAccess(e);
        return e.value;
    }

    void afterNodeAccess(Node<K,V> e) { // move node to last
            LinkedHashMap.Entry<K,V> last;
            if (accessOrder && (last = tail) != e) {
                LinkedHashMap.Entry<K,V> p =
                    (LinkedHashMap.Entry<K,V>)e, b = p.before, a = p.after;
                p.after = null;
                if (b == null)
                    head = a;
                else
                    b.after = a;
                if (a != null)
                    a.before = b;
                else
                    last = b;
                if (last == null)
                    head = p;
                else {
                    p.before = last;
                    last.after = p;
                }
                tail = p;
                ++modCount;
            }
        }
```

由代码可见
- accessOrder代表是否会将每个数据节点移到末端
- afterNodeAccess方法是该数据节点移到末端的实现
- LinkedHashMap基于链表存储的，移动节点的操作复杂度仅为O(1)，这就是LruCache选择LinkedHashMap作为存储结构的原因。
