---
title: 虚引用
tags: 
 - Java
 - 引用类型
categories: 编程
date: 2018-12-29 19:57:51
---

## 定义

虚引用是使用PhantomReference创建的引用，虚引用也称为幽灵引用或者幻影引用，是所有引用类型中最弱的一个。一个对象是否有虚引用的存在，完全不会对其生命周期构成影响，也无法通过虚引用获得一个对象实例。

## 说明

虚引用，正如其名，对一个对象而言，这个引用形同虚设，有和没有一样。

> 虚可达
> 如果一个对象与GC Roots之间仅存在虚引用，则称这个对象为`虚可达（phantom reachable）`对象。

当试图通过虚引用的get()方法取得强引用时，总是会返回null，并且，虚引用必须和引用队列一起使用。既然这么虚，那么它出现的意义何在？？

别慌别慌，自然有它的用处。它的作用在于跟踪垃圾回收过程，在对象被收集器回收时收到一个系统通知。 当垃圾回收器准备回收一个对象时，如果发现它还有虚引用，就会在垃圾回收后，将这个虚引用加入引用队列，在其关联的虚引用出队前，不会彻底销毁该对象。 所以可以通过检查引用队列中是否有相应的虚引用来判断对象是否已经被回收了。

如果一个对象没有强引用和软引用，对于垃圾回收器而言便是可以被清除的，在清除之前，会调用其finalize方法，如果一个对象已经被调用过finalize方法但是还没有被释放，它就变成了一个虚可达对象。

与软引用和弱引用不同，显式使用虚引用可以阻止对象被清除，只有在程序中显式或者隐式移除这个虚引用时，这个已经执行过finalize方法的对象才会被清除。想要显式的移除虚引用的话，只需要将其从引用队列中取出然后扔掉（置为null）即可。

同样来看一个栗子：

```java
public class PhantomReferenceTest {
    private static final List<Object> TEST_DATA = new LinkedList<>();
    private static final ReferenceQueue<TestClass> QUEUE = new ReferenceQueue<>();

    public static void main(String[] args) {
        TestClass obj = new TestClass("Test");
        PhantomReference<TestClass> phantomReference = new PhantomReference<>(obj, QUEUE);

        // 该线程不断读取这个虚引用，并不断往列表里插入数据，以促使系统早点进行GC
        new Thread(() -> {
            while (true) {
                TEST_DATA.add(new byte[1024 * 100]);
                try {
                    Thread.sleep(1000);
                } catch (InterruptedException e) {
                    e.printStackTrace();
                    Thread.currentThread().interrupt();
                }
                System.out.println(phantomReference.get());
            }
        }).start();

        // 这个线程不断读取引用队列，当弱引用指向的对象呗回收时，该引用就会被加入到引用队列中
        new Thread(() -> {
            while (true) {
                Reference<? extends TestClass> poll = QUEUE.poll();
                if (poll != null) {
                    System.out.println("--- 虚引用对象被jvm回收了 ---- " + poll);
                    System.out.println("--- 回收对象 ---- " + poll.get());
                }
            }
        }).start();

        obj = null;

        try {
            Thread.currentThread().join();
        } catch (InterruptedException e) {
            e.printStackTrace();
            System.exit(1);
        }
    }

    static class TestClass {
        private String name;

        public TestClass(String name) {
            this.name = name;
        }

        @Override
        public String toString() {
            return "TestClass - " + name;
        }
    }
}
```

使用的虚拟机设置如下：

```bash
-verbose:gc -Xms4m -Xmx4m -Xmn2m
```

运行结果如下：

```bash
[GC (Allocation Failure)  1024K->432K(3584K), 0.0113386 secs]
[GC (Allocation Failure)  1455K->520K(3584K), 0.0133610 secs]
[GC (Allocation Failure)  1544K->648K(3584K), 0.0008654 secs]
null
null
null
[GC (Allocation Failure)  1655K->973K(3584K), 0.0008111 secs]
null
...省略几个null的输出
[GC (Allocation Failure)  1980K->1997K(3584K), 0.0009289 secs]
[Full GC (Ergonomics)  1997K->1870K(3584K), 0.0048483 secs]
--- 弱引用对象被jvm回收了 ---- java.lang.ref.PhantomReference@74cbe23d
--- 回收对象 ---- null
null
...省略几个null和几次Full GC的输出
[Full GC (Ergonomics)  2971K->2971K(3584K), 0.0024850 secs]
[Full GC (Allocation Failure)  2971K->2971K(3584K), 0.0022460 secs]
Exception in thread "Thread-0" java.lang.OutOfMemoryError: Java heap space
	at weakhashmap.PhantomReferenceTest.lambda$main$0(PhantomReferenceTest.java:20)
	at weakhashmap.PhantomReferenceTest$$Lambda$1/2065951873.run(Unknown Source)
	at java.lang.Thread.run(Thread.java:748)
```

因为设置的虚拟机堆大小比较小，所以创建一个100k的对象时直接进入了老年代，等到发生Full GC时才会被扫描然后回收。

## 适用场景

使用虚引用的目的就是为了得知对象被GC的时机，所以可以利用虚引用来进行销毁前的一些操作，比如说资源释放等。这个虚引用对于对象而言完全是无感知的，有没有完全一样，但是对于虚引用的使用者而言，就像是待观察的对象的把脉线，可以通过它来观察对象是否已经被回收，从而进行相应的处理。

事实上，虚引用有一个很重要的用途就是用来做堆外内存的释放，DirectByteBuffer就是通过虚引用来实现堆外内存的释放的。 

## 小结

+ 虚引用是最弱的引用
+ 虚引用对对象而言是无感知的，对象有虚引用跟没有是完全一样的
+ 虚引用不会影响对象的生命周期
+ 虚引用可以用来做为对象是否存活的监控