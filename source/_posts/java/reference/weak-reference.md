---
title: 弱引用
tags: 
 - Java
 - 引用类型
categories: 编程
date: 2018-12-29 19:57:51
---

## 定义

弱引用是使用WeakReference创建的引用，弱引用也是用来描述非必需对象的，它是比软引用更弱的引用类型。在发生GC时，只要发现弱引用，不管系统堆空间是否足够，都会将对象进行回收。

## 说明

弱引用，从名字来看就很弱嘛，这种引用指向的对象，一旦在GC时被扫描到，就逃脱不了被回收的命运。<img src="./0040.png" width="50"/>

但是，弱引用指向的对象也并不一定就马上会被回收，如果弱引用对象较大，直接进到了老年代，那么就可以苟且偷生到Full GC触发前，所以弱引用对象也可能存在较长的一段时间。一旦一个弱引用对象被垃圾回收器回收，便会加入到一个引用队列中（如果有的话）。

弱引用对应的类为WeakReference，举个栗子：

```java
String s = new String("Frank");    
WeakReference<String> weakRef = new WeakReference<String>(s);
s = null;
```

这里我们把s设置为null后，字符串对象便只有弱引用指向它。

::: tip 弱可达
如果一个对象与GC Roots之间仅存在弱引用，则称这个对象为`弱可达(weakly reachable)`对象。
:::

::: warning 注意
在垃圾回收器回收一个对象前，WeakReference类所提供的get方法会返回其引用对象的强引用，一旦垃圾回收器回收掉该对象之后，get方法将返回null。所以在获取弱引用对象的代码中，一定要判断是否为null，以免出现NullPointerException异常导致应用崩溃。<img src="./0019.png" width="50"/>
:::

下面的代码会让s再次持有对象的强引用：

```java
s = weakRef.get();
```

如果在weakRef包裹的对象被回收前，用强引用关联该对象，那这个对象又会变成强可达状态。

来看一个简单的栗子了解一下WeakReference引用的对象是何时被回收的：

```java
public class WeakReferenceTest {
    private static final List<Object> TEST_DATA = new LinkedList<>();
    private static final ReferenceQueue<TestClass> QUEUE = new ReferenceQueue<>();

    public static void main(String[] args) {
        TestClass obj = new TestClass("Test");
        WeakReference<TestClass> weakRef = new WeakReference<>(obj, QUEUE);
        //可以重新获得OOMClass对象，并用一个强引用指向它
        //oomObj = weakRef.get();

        // 该线程不断读取这个弱引用，并不断往列表里插入数据，以促使系统早点进行GC
        new Thread(() -> {
            while (true) {
                TEST_DATA.add(new byte[1024 * 100]);
                try {
                    Thread.sleep(1000);
                } catch (InterruptedException e) {
                    e.printStackTrace();
                    Thread.currentThread().interrupt();
                }
                System.out.println(weakRef.get());
            }
        }).start();

        // 这个线程不断读取引用队列，当弱引用指向的对象呗回收时，该引用就会被加入到引用队列中
        new Thread(() -> {
            while (true) {
                Reference<? extends TestClass> poll = QUEUE.poll();
                if (poll != null) {
                    System.out.println("--- 弱引用对象被jvm回收了 ---- " + poll);
                    System.out.println("--- 回收对象 ---- " + poll.get());
                }
            }
        }).start();

        //将强引用指向空指针 那么此时只有一个弱引用指向TestClass对象
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

设置一下虚拟机参数：

```bash
-verbose:gc -Xms4m -Xmx4m -Xmn2m
```

运行结果如下：

```bash
[GC (Allocation Failure)  1017K->464K(3584K), 0.0014345 secs]
[GC (Allocation Failure)  1483K->536K(3584K), 0.0017221 secs]
[GC (Allocation Failure)  1560K->648K(3584K), 0.0036572 secs]
TestClass - Test
TestClass - Test
TestClass - Test
[GC (Allocation Failure)  1621K->984K(3584K), 0.0011455 secs]
--- 弱引用对象被jvm回收了 ---- java.lang.ref.WeakReference@51a947fe
--- 回收对象 ---- null
null
...省略n个null和几次GC信息
[Full GC (Ergonomics)  2964K->2964K(3584K), 0.0025450 secs]
[Full GC (Allocation Failure)  2964K->2964K(3584K), 0.0021907 secs]
java.lang.OutOfMemoryError: Java heap space
Dumping heap to java_pid6860.hprof ...
Heap dump file created [3912229 bytes in 0.011 secs]
Exception in thread "Thread-0" java.lang.OutOfMemoryError: Java heap space
	at weakhashmap.WeakReferenceTest.lambda$main$0(WeakReferenceTest.java:22)
	at weakhashmap.WeakReferenceTest$$Lambda$1/764977973.run(Unknown Source)
	at java.lang.Thread.run(Thread.java:748)
```

可以看到，其实弱引用也并不是一发生GC就被回收掉了。

## 应用场景

如果一个对象仅仅是偶尔使用，并且希望在使用时随时就能获取到，但又不想影响此对象的垃圾收集，那么你应该用 WeakReference 来引用该对象。 

弱引用可以和一个引用队列（ReferenceQueue）联合使用，如果弱引用所引用的对象被垃圾回收，Java虚拟机就会把这个弱引用加入到与之关联的引用队列中。

一般来说，很少直接使用WeakReference，而是使用WeakHashMap。在WeakHashMap中，内部有一个引用队列，插入的元素会被包裹成WeakReference，并加入队列中，用来做缓存再合适不过。

在Tomcat的缓存中，其实就用到了WeakHashMap：

```java
public final class ConcurrentCache<K,V> {
    private final int size;
    private final Map<K,V> eden;
    private final Map<K,V> longterm;

    public ConcurrentCache(int size) {
        this.size = size;
        this.eden = new ConcurrentHashMap<>(size);
        this.longterm = new WeakHashMap<>(size);
    }

    public V get(K k) {
        // 先从eden中取
        V v = this.eden.get(k);
        if (v == null) {
            // 如果取不到再从longterm中取
            synchronized (longterm) {
                v = this.longterm.get(k);
            }
            // 如果取到则重新放到eden中
            if (v != null) {
                this.eden.put(k, v);
            }
        }
        return v;
    }

    public void put(K k, V v) {
        if (this.eden.size() >= size) {
            // 如果eden中的元素数量大于指定容量，将所有元素放到longterm中
            synchronized (longterm) {
                this.longterm.putAll(this.eden);
            }
            this.eden.clear();
        }
        this.eden.put(k, v);
    }
}
```

这里有eden和longterm的两个map，如果对jvm堆了解的话，可以看出tomcat在这里是使用ConcurrentHashMap和WeakHashMap做了类似分代缓存的操作。

在put方法里，在插入键值对时，先检查eden缓存的容量是否超出设定的大小。如果没有则直接放入eden缓存，如果超了则锁定longterm将eden中所有的键值对都放入longterm。再将eden清空并插入该键值对。

在get方法中，也是优先从eden中找对应的key，如果没有则进入longterm缓存中查找，找到后就加入eden缓存并返回。 

经过这样的设计，相对常用的对象都能在eden缓存中找到，不常用（有可能被销毁的对象）的则进入longterm缓存。而longterm的key的实际对象没有其他引用指向它时，gc就会自动回收heap中该弱引用指向的实际对象，并将弱引用放入其引用队列中。

## 弱引用与软引用对比

弱引用与软引用的区别在于：

1. 只具有弱引用的对象拥有更短暂的生命周期。
2. 被垃圾回收器回收的时机不一样，在垃圾回收器线程扫描它所管辖的内存区域的过程中，一旦发现了只具有弱引用的对象，不管当前内存空间足够与否，都会回收它的内存。而被软引用关联的对象只有在内存不足时才会被回收。
3. 弱引用不会影响GC，而软引用会一定程度上对GC造成影响。

相似之处：都是用来描述非必需对象的。

那么什么时候用SoftReference，什么时候用WeakReference呢？

如果缓存的对象是比较大的对象，使用频率相对较高的对象，那么使用SoftReference会更好，因为这样能让缓存对象有更长的生命周期。

如果缓存对象都是比较小的对象，使用频率一般或者相对较低，那么使用WeakReference会更合适。

当然，如果实在不知道选哪个，一般而言，用作缓存时使用WeakHashMap都不会有太大问题。<img src="./195.png" width="50"/>

## 小结

+ 弱引用是比软引用更弱的引用类型 
+ 弱引用不能延长对象的生命周期，一旦对象只剩下弱引用，它就随时可能会被回收
+ 可以通过弱引用获取对象的强引用
+ 弱引用适合用作缓存









