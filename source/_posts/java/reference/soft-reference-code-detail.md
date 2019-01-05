---
title: SoftReference源码详解
tags: 
 - Java
 - 引用类型
categories: 编程
date: 2018-12-29 20:20:51
---

## 定义

SoftReference是软引用，其引用的对象在内存不足的时候会被回收。只有软引用指向的对象称为软可达（softly-reachable）对象。

## 说明

垃圾回收器会在内存不足，经过一次垃圾回收后，内存仍旧不足的时候回收掉软可达对象。在虚拟机抛出OOM之前，会保证已经清除了所有指向软可达对象的软引用。

如果内存足够，并没有规定回收软引用的具体时间，所以在内存充足的情况下，软引用对象也可能存活很长时间。

JVM会根据当前内存的情况来决定是否回收softly-reachable对象，但只要referent有强引用存在，该referent就一定不会被清理，因此SoftReference适合用来实现memory-sensitive caches。软引用的回收策略在不同的JVM实现会略有不同。

另外，JVM不仅仅只会考虑当前内存情况，还会考虑软引用所指向的referent最近使用情况和创建时间来综合决定是否回收该referent。

一般而言，SoftReference对象会在垃圾回收器回收其内部referent后，才会被放入其注册的引用队列中（如果创建时注册了的话）。

```ba&#39;sh
Soft reference objects, which are cleared at the discretion of the garbage collector in response to memory demand. 
```

就是说，软引用具体什么时候回收最终还是由虚拟机自己决定的，所以不同虚拟机对软引用的回收方式会有些不一样。

## SoftReference源码

```java
public class SoftReference<T> extends Reference<T> {
    /**
     * 由垃圾回收器负责更新的时间戳
     */
    static private long clock;

    /**
     * 在get方法调用时更新的时间戳，当虚拟机选择软引用进行清理时，可能会参考这个字段。
     */
    private long timestamp;

    public SoftReference(T referent) {
        super(referent);
        this.timestamp = clock;
    }

    public SoftReference(T referent, ReferenceQueue<? super T> q) {
        super(referent, q);
        this.timestamp = clock;
    }

    /**
     * 返回引用指向的对象，如果referent已经被程序或者垃圾回收器清理，则返回null。
     */
    public T get() {
        T o = super.get();
        if (o != null && this.timestamp != clock)
            this.timestamp = clock;
        return o;
    }
}
```

SoftReference类内部代码很少，两个成员变量，clock是一个静态变量，是由垃圾回收器负责更新的时间戳，在JVM初始化时，会对变量clock进行初始化，同时，在JVM发生GC时，也会更新clock的值，所以clock会记录上次GC发生的时间点。

timestamp是在创建和更新时更新的时间戳，将其更新为clock的值，垃圾回收器在回收软引用对象时可能会参考timestamp。

SoftReference类有两个构造函数，一个是不传引用队列，一个传引用队列。在创建时，都会更新timestamp，将其赋值为clock的值，get方法也并没有什么骚操作，只是简单的调用 super.get() 并在返回值不为null时更新timestamp。

## 软引用何时回收

前面说过，软引用会在内存不足的时候进行回收，但是回收时并不会一次性全部回收，而是会使用一定的回收策略。

下面以最常用的虚拟机HotSpot进行说明。下面是Oracle文档中的说明：

```java
The default value is 1000 ms per megabyte, which means that a soft reference will survive (after the last strong reference to the object has been collected) for 1 second for each megabyte of free space in the heap
```

默认的生存周期为1000ms/Mb，举个具体的栗子：

假设，堆内存为512Mb，并且可用内存为400Mb，我们创建一个object A，用软引用创建一个引用A的缓存对象cache，以及另一个object B 引用object A。此时，由于B持有A的强引用，所以对象A是强可达并且不会被垃圾回收器回收。

{% asset_img soft-reference-3.png soft-reference-3 %}

如果B被删除了，那么A仅剩下一个软引用cache引用它，如果A在400s内没有再次被强引用关联，它将会在超时后被删除。

{% asset_img soft-reference-2.png soft-reference-2 %}

下面是一个控制软引用的栗子：

```java
public class SoftRefTest {
    public static class A{
    }
    public static class B{
        private A strongRef;
 
        public void setStrongRef(A ref) {
            this.strongRef = ref;
        }
    }
    public static SoftReference<A> cache;
 
    public static void main(String[] args) throws InterruptedException{
        //用一个A类实例的软引用初始化cache对象
        SoftRefTest.A instanceA = new SoftRefTest.A();
        cache = new SoftReference<SoftRefTest.A>(instanceA);
        instanceA = null;
        // instanceA 现在是软可达状态，并且会在之后的某个时间被垃圾回收器回收
        Thread.sleep(10000);
 
        ...
        SoftRefTest.B instanceB = new SoftRefTest.B();
        //由于cache仅持有instanceA的软引用，所以无法保证instanceA仍然存活
        instanceA = cache.get();
        if (instanceA == null){
            instanceA = new SoftRefTest.A();
            cache = new SoftReference<SoftRefTest.A>(instanceA);
        }
        instanceB.setStrongRef(instanceA);
        instanceA = null;
        // instanceA现在与cache对象存在软引用并且与B对象存在强引用，所以它不会被垃圾回收器回收
 
        ...
    }
}
```

但是需要注意的是，被软引用对象关联的对象会自动被垃圾回收器回收，但是软引用对象本身也是一个对象，这些创建的软引用并不会自动被垃圾回收器回收掉，所以在之前一篇中说明里的[栗子](./soft-reference.md#说明)里，软引用是不会被释放掉的。

所以，你仍然需要手动去清理它们，否则也会导致OOM的产生，这里也举一个小栗子：

```java
public class SoftReferenceTest{

    public static class MyBigObject{
        int[] data = new int[128];
    }

    public static int CACHE_INITIAL_CAPACITY = 100_000;
    // 静态集合保存软引用，会导致这些软引用对象本身无法被垃圾回收器回收
    public static Set<SoftReference<MyBigObject>> cache = new HashSet<>(CACHE_INITIAL_CAPACITY);

    public static void main(String[] args) {
        for (int i = 0; i < 100_000; i++) {
            MyBigObject obj = new MyBigObject();
            cache.add(new SoftReference<>(obj));
            if (i%10_000 == 0){
                System.out.println("size of cache:" + cache.size());
            }
        }
        System.out.println("End");
    }
}
```

使用的虚拟机参数为：

```java
-Xms4m -Xmx4m -Xmn2m
```

输出如下：

```bash
size of cache:1
size of cache:10001
size of cache:20001
size of cache:30001
Exception in thread "main" java.lang.OutOfMemoryError: GC overhead limit exceeded
```

最终抛出了OOM，但这里的原因却并不是`Java heap space`，而是` GC overhead limit exceeded ` ，之所以会抛出这个错误，是由于虚拟机一直在不断回收软引用，回收进行的速度过快，占用的cpu过大（超过98%），并且每次回收掉的内存过小（小于2%），导致最终抛出了这个错误。

对于这里，合适的处理方式是注册一个引用队列，每次循环之后将引用队列中出现的软引用对象从cache中移除。

```java
public class SoftReferenceTest{

    public static int removedSoftRefs = 0;

    public static class MyBigObject{
        int[] data = new int[128];
    }

    public static int CACHE_INITIAL_CAPACITY = 100_000;
    // 静态集合保存软引用，会导致这些软引用对象本身无法被垃圾回收器回收
    public static Set<SoftReference<MyBigObject>> cache = new HashSet<>(CACHE_INITIAL_CAPACITY);
    public static ReferenceQueue<MyBigObject> referenceQueue = new ReferenceQueue<>();

    public static void main(String[] args) {
        for (int i = 0; i < 100_000; i++) {
            MyBigObject obj = new MyBigObject();
            cache.add(new SoftReference<>(obj, referenceQueue));
            clearUselessReferences();
        }
        System.out.println("End, removed soft references=" + removedSoftRefs);
    }

    public static void clearUselessReferences() {
        Reference<? extends MyBigObject> ref = referenceQueue.poll();
        while (ref != null) {
            if (cache.remove(ref)) {
                removedSoftRefs++;
            }
            ref = referenceQueue.poll();
        }
    }
}
```

使用同样的虚拟机配置，输出如下：

```bash
End, removed soft references=97319
```

## HotSpot虚拟机对于软引用的处理

就HotSpot虚拟机而言，常用的回收策略是基于当前堆大小的LRU策略（LRUCurrentHeapPolicy），会使用clock的值减去timestamp，得到的差值，就是这个软引用被闲置的时间，如果闲置足够长时间，就认为是可被回收的。

```c++
bool LRUCurrentHeapPolicy::should_clear_reference(oop p,
                                                  jlong timestamp_clock) {
  jlong interval = timestamp_clock - java_lang_ref_SoftReference::timestamp(p);
  assert(interval >= 0, "Sanity check");

  if(interval <= _max_interval) {
    return false;
  }

  return true;
}
```

这里 `timestamp_clock` 即SoftReference中clock的值，即上次GC时间。java_lang_ref_SoftReference::timestamp(p)可以获取引用中timestamp的值。

那么这个足够长的时间 `_max_interval`是怎么计算的呢？

```c++
void LRUCurrentHeapPolicy::setup() {
  _max_interval = (Universe::get_heap_free_at_last_gc() / M) * SoftRefLRUPolicyMSPerMB;
  assert(_max_interval >= 0,"Sanity check");
}
```

其中`SoftRefLRUPolicyMSPerMB`默认1000，所以可以看出这个回收时间与上次GC后的剩余空间大小有关，可用空间越大，`_max_interval`就越大。

如果GC之后，堆的可用空间还很大的话，SoftReference对象可以长时间的在堆中而不被回收。反之，如果GC之后，只剩下很少的内存可用，那么SoftReference对象便会很快进行回收。

SoftReference在一定程度上会影响垃圾回收，如果软可达对象中对应的referent多次垃圾回收仍然不满足释放条件，那么它会停留在堆的老年代，占据很大部分空间，在JVM没有抛出OutOfMemoryError前，它有可能会导致频繁的Full GC，会对性能有一定的影响。 

## 小结

+ 软引用的具体回收时间与具体虚拟机有关
+ 软引用中会在创建和调用get方法的时候更新内部timestamp，提供给虚拟机回收时进行参考
+ hotspot虚拟机对于软引用使用的是LRU策略，回收时会根据软引用被闲置的时间和当前内存综合进行判断

