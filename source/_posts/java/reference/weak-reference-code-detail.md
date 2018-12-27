---
prev: ./soft-reference-code-detail
next: ./phantom-reference-code-detail
---

# WeakReference源码详解

## 定义

::: tip 
WeakReference是弱引用，该引用不会影响垃圾回收器对对象的回收，不会影响对象的生命周期。
:::

## 说明

当虚拟机在某个时间点决定要回收一个弱可达（weakly-reachable）对象时，会自动清除该对象的所有弱引用。并且会将对象变为finalizable状态，然后把这些刚清除的弱引用放到其注册的引用队列中。

[前面](./weak-reference.md)已经说明过WeakReference的用法了，本篇仅对WeakReference从源码角度做一些补充。

## 源码

```java
public class WeakReference<T> extends Reference<T> {
    public WeakReference(T referent) {
        super(referent);
    }
    
    public WeakReference(T referent, ReferenceQueue<? super T> q) {
        super(referent, q);
    }
    
}
```

嗯，十行代码，可以说是很简单的一个类了，只有两个构造函数，一个传引用队列，另一个不传，没有覆盖父类Reference的任何方法。

## WeakHashMap

说到WeakReference，自然不能不说WeakHashMap，这个map的用法与hashmap基本一致，它的特点便是使用弱引用作为key，这就让它有一个很重要的特性，它可以自动清除自身，这样就不需要再像之前SoftReference那样需要手动去释放引用实例。

如果想了解关于WeakHashMap更详细的内容，可以戳[这里](../collections/weakhashmap-code-detail.md)。

<img src="./06.png" width="40"/>好像。。。没什么可讲的了。在前面弱引用一篇里基本都讲完了。 