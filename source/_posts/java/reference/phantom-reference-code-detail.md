---
title: PhantomReference源码详解
tags: 
 - Java
 - 引用类型
categories: 编程
date: 2018-12-29 20:20:51
---

## 定义

PhantomReference是虚引用，该引用不会影响不会影响对象的生命周期，也无法从虚引用中获取对象实例。

## 说明

源码介绍部分其实也没多大内容，主要内容都在前面介绍中说完了。PhantomReference类的源码和WeakReference类一样简单：

```java
public class PhantomReference<T> extends Reference<T> {
    public T get() {
        return null;
    }

    /**
     * 这里传入的引用队列也可以为null，但是这样的引用没有任何意义，因为永远不会入队
     */
    public PhantomReference(T referent, ReferenceQueue<? super T> q) {
        super(referent, q);
    }
}
```

可以看到，get方法直接返回null，有一个两个参数的构造方法，传入被引用的对象和引用队列。

<img src="/images/06.png" width="40"/>那么，这篇也先告一段落吧。 