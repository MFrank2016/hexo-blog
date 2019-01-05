---
title: 软引用
tags: 
 - Java
 - 引用类型
categories: 编程
date: 2018-12-29 20:20:51
---

## 定义

软引用是使用SoftReference创建的引用，强度弱于强引用，被其引用的对象在内存不足的时候会被回收，不会产生内存溢出。

## 说明

软引用，顾名思义就是比较“软”一点的引用。<img src="/images/0003.png" width="50"/>

当一个对象与GC Roots之间存在强引用时，无论何时都不会被GC回收掉。如果一个对象与GC Roots之间没有强引用与其关联而存在软引用关联时，那么垃圾回收器对它的态度就取决于内存的紧张程度了。如果内存空间足够，垃圾回收器就不会回收这个对象，但如果内存空间不足了，它就难逃被回收的厄运。<img src="/images/0005.png" width="50"/>

> 软可达
> 如果一个对象与GC Roots之间不存在强引用，但是存在软引用，则称这个对象为`软可达（soft reachable）`对象。

在垃圾回收器没有回收它的时候，软可达对象就像强可达对象一样，可以被程序正常访问和使用，但是需要通过软引用对象间接访问，需要的话也能重新使用强引用将其关联。所以软引用适合用来做内存敏感的高速缓存。

```java
String s = new String("Frank");    // 创建强引用与String对象关联，现在该String对象为强可达状态
SoftReference<String> softRef = new SoftReference<String>(s);     // 再创建一个软引用关联该对象
s = null;        // 消除强引用，现在只剩下软引用与其关联，该String对象为软可达状态
s = softRef.get();  // 重新关联上强引用
```

这里变量s持有对字符串对象的强引用，而softRef持有对该对象的软引用，所以当执行s = null后，字符串对象就只剩下软引用了，这时如果因为内存不足发生Full GC，就会把这个字符串对象回收掉。

> 注意
> 在垃圾回收器回收一个对象前，SoftReference类所提供的get方法会返回Java对象的强引用，一旦垃圾线程回收该对象之后，get方法将返回null。所以在获取软引用对象的代码中，一定要先判断返回是否为null，以免出现NullPointerException异常而导致应用崩溃。<img src="/images/2030.png" width="50"/>

下面的代码会让s再次持有对象的强引用：

```java
s = softRef.get();
```

如果在softRef指向的对象被回收前，用强引用指向该对象，那这个对象又会变成强可达。

来看一个使用SoftReference的栗子：

```java
public class TestA {
    static class OOMClass{
        private int[] oom = new int[1024 * 100];// 100KB
    }

    public static void main(String[] args) throws InterruptedException {
        ReferenceQueue<OOMClass> queue = new ReferenceQueue<>();
        List<SoftReference> list = new ArrayList<>();
        while(true){
            for (int i = 0; i < 100; i++) {
                list.add(new SoftReference<OOMClass>(new OOMClass(), queue));
            }
            Thread.sleep(500);
        }
    }
}
```

> 注意
> ReferenceQueue中声明的类型为OOMClass，即与SoftReference引用的类型一致。

设置一下虚拟机参数：

```bash
-verbose:gc -Xms4m -Xmx4m -Xmn2m
```

运行结果：

```bash
[GC (Allocation Failure)  1017K->432K(3584K), 0.0017239 secs]
[GC (Allocation Failure)  1072K->472K(3584K), 0.0099237 secs]
[GC (Allocation Failure)  1323K->1296K(3584K), 0.0009528 secs]
[GC (Allocation Failure)  2114K->2136K(3584K), 0.0009951 secs]
[Full GC (Ergonomics)  2136K->1992K(3584K), 0.0040658 secs]
[Full GC (Ergonomics)  2807K->2791K(3584K), 0.0036280 secs]
[Full GC (Allocation Failure)  2791K->373K(3584K), 0.0032477 secs]
[Full GC (Ergonomics)  2786K->2773K(3584K), 0.0034554 secs]
[Full GC (Allocation Failure)  2773K->373K(3584K), 0.0032667 secs]
[Full GC (Ergonomics)  2798K->2775K(3584K), 0.0036231 secs]
[Full GC (Allocation Failure)  2775K->375K(3584K), 0.0055482 secs]
[Full GC (Ergonomics)  2799K->2776K(3584K), 0.0031358 secs]
...省略n次GC信息
```

在TestA中，我们使用死循环不断的往list中添加新对象，如果是强引用，会很快因为内存不足而抛出OOM，因为这里的堆内存大小设置为了4M，而一个对象就有100KB，一个循环添加100个对象，也就是差不多10M，显然一个循环都跑不完就会内存不足，而这里，因为使用的是软引用，所以JVM会在内存不足的时候将软引用回收掉。

```bash
[Full GC (Allocation Failure)  2791K->373K(3584K), 0.0032477 secs]
```

从这一条可以看出，在内存不足发生Full GC时，回收掉了大部分的软引用指向的对象，释放了大量的内存。

因为这里新生代只分配了2M，所以很快就会发生GC，如果你的程序运行没有看到这个结果，请先确认一下虚拟机参数是否设置正确，如果设置正确还是没有看到，那么将循环次数由1000改为10000或者100000在试试看。<img src="/images/141.png" width="50"/>

## 应用场景

软引用关联的对象，只有在内存不足的时候JVM才会回收该对象。这一点可以很好地用来解决OOM的问题，并且这个特性很适合用来实现缓存：比如网页缓存、图片缓存等。 

现在考虑这样一个场景 ，在很多应用中，都会出现大量的默认图片，比如说QQ的默认头像，应用内的默认图标等等，这些图片很多地方会用到。

如果每次都去读取图片，由于读取文件速度较慢，大量重复的读取会导致性能下降。所以可以考虑将图片缓存起来，需要的时候直接从内存中读取。但是，由于图片占用内存空间比较大，缓存的图片过多会占用比较多的内存，就可能比较容易发生OOM。这时候，软引用就派得上用场了。<img src="/images/0009.png" width="50"/>

> 注意
> SoftReference对象是用来保存软引用的，但它同时也是一个Java对象。所以，当软可及对象被回收之后，虽然这个SoftReference对象的get()方法返回null，但SoftReference对象本身并不是null，而此时这个SoftReference对象已经不再具有存在的价值，需要一个适当的清除机制，避免大量SoftReference对象带来的内存泄漏。

ReferenceQueue就是用来保存这些需要被清理的引用对象的。软引用可以和一个引用队列（ReferenceQueue）联合使用，如果软引用所引用的对象被垃圾回收器回收，Java虚拟机就会把这个软引用加入到与之关联的引用队列中。

下面用SoftReference来实现一个简单的缓存类：

```java
public class SoftCache<T> {
    // 引用队列
    private ReferenceQueue<T> referenceQueue = new ReferenceQueue<>();
    // 保存软引用集合，在引用对象被回收后销毁
    private List<Reference<T>> list = new ArrayList<>();

    // 添加缓存对象
    public synchronized void add(T obj){
        // 构建软引用
        Reference<T> reference = new SoftReference<T>(obj, referenceQueue);
        // 加入列表中
        list.add(reference);
    }

    // 获取缓存对象
    public synchronized T get(int index){
        // 先对无效引用进行清理
        clear();
        if (index < 0 || list.size() < index){
            return null;
        }
        Reference<T> reference = list.get(index);
        return reference == null ? null : reference.get();
    }

    public int size(){
        return list.size();
    }

    @SuppressWarnings("unchecked")
    private void clear(){
        Reference<T> reference;
        while (null != (reference = (Reference<T>) referenceQueue.poll())){
            list.remove(reference);
        }
    }
}
```

然后测试一下这个缓存类：

```java
public class SoftCacheTest {
    private static int num = 0;

    public static void main(String[] args){
        SoftCache<OOMClass> softCache = new SoftCache<>();
        for (int i = 0; i < 40; i++) {
            softCache.add(new OOMClass("OOM Obj-" + ++num));
        }
        System.out.println(softCache.size());
        for (int i = 0; i < softCache.size(); i++) {
            OOMClass obj = softCache.get(i);
            System.out.println(obj == null ? "null" : obj.name);
        }
        System.out.println(softCache.size());
    }

    static class OOMClass{
        private String name;
        private int[] oom = new int[1024 * 100];// 100KB

        public OOMClass(String name) {
            this.name = name;
        }
    }
}
```

仍使用之前的虚拟机参数：

```bash
-verbose:gc -Xms4m -Xmx4m -Xmn2m
```

运行结果：

```java
[GC (Allocation Failure)  1017K->432K(3584K), 0.0012236 secs]
[GC (Allocation Failure)  1117K->496K(3584K), 0.0016875 secs]
[GC (Allocation Failure)  1347K->1229K(3584K), 0.0015059 secs]
[GC (Allocation Failure)  2047K->2125K(3584K), 0.0018090 secs]
[Full GC (Ergonomics)  2125K->1994K(3584K), 0.0054759 secs]
[Full GC (Ergonomics)  2822K->2794K(3584K), 0.0023167 secs]
[Full GC (Allocation Failure)  2794K->376K(3584K), 0.0036056 secs]
[Full GC (Ergonomics)  2795K->2776K(3584K), 0.0042365 secs]
[Full GC (Allocation Failure)  2776K->376K(3584K), 0.0035122 secs]
[Full GC (Ergonomics)  2795K->2776K(3584K), 0.0054760 secs]
[Full GC (Allocation Failure)  2776K->376K(3584K), 0.0036965 secs]
[Full GC (Ergonomics)  2802K->2777K(3584K), 0.0044513 secs]
[Full GC (Allocation Failure)  2777K->376K(3584K), 0.0041400 secs]
[Full GC (Ergonomics)  2796K->2777K(3584K), 0.0025255 secs]
[Full GC (Allocation Failure)  2777K->376K(3584K), 0.0037690 secs]
[Full GC (Ergonomics)  2817K->2777K(3584K), 0.0037759 secs]
[Full GC (Allocation Failure)  2777K->377K(3584K), 0.0042416 secs]
缓存列表大小：40
OOM Obj-37
OOM Obj-38
OOM Obj-39
OOM Obj-40
缓存列表大小：4
```

可以看到，缓存40个软引用对象之后，如果一次性全部存储，显然内存大小无法满足，所以在不断创建软引用对象的过程中，不断发生GC来进行垃圾回收，最终只有4个软引用未被清理掉。

## 强引用与软引用对比

没有对比就没有伤害，来将强引用和软引用对比一下：

```java
public class Test {

    static class OOMClass{
        private int[] oom = new int[1024];
    }

    public static void main(String[] args) {
        testStrongReference();
        //testSoftReference();
    }

    public static void testStrongReference(){
        List<OOMClass> list = new ArrayList<>();
        for (int i = 0; i < 1000; i++) {
            list.add(new OOMClass());
        }
    }

    public static void testSoftReference(){
        ReferenceQueue<OOMClass> referenceQueue = new ReferenceQueue<>();
        List<SoftReference> list = new ArrayList<>();
        for (int i = 0; i < 1000; i++) {
            OOMClass oomClass = new OOMClass();
            list.add(new SoftReference(oomClass, referenceQueue));
            oomClass = null;
        }
    }
}
```

运行testStrongReference方法的结果如下：

```java
[GC (Allocation Failure)  1019K->384K(3584K), 0.0033595 secs]
[GC (Allocation Failure)  1406K->856K(3584K), 0.0013098 secs]
[GC (Allocation Failure)  1880K->1836K(3584K), 0.0014382 secs]
[Full GC (Ergonomics)  1836K->1756K(3584K), 0.0039761 secs]
[Full GC (Ergonomics)  2778K->2758K(3584K), 0.0021269 secs]
[Full GC (Ergonomics)  2779K->2770K(3584K), 0.0016329 secs]
[Full GC (Ergonomics)  2779K->2775K(3584K), 0.0023157 secs]
[Full GC (Ergonomics)  2775K->2775K(3584K), 0.0015927 secs]
[Full GC (Ergonomics)  3037K->3029K(3584K), 0.0025071 secs]
[Full GC (Ergonomics)  3067K->3065K(3584K), 0.0017529 secs]
[Full GC (Allocation Failure)  3065K->3047K(3584K), 0.0033445 secs]
[Full GC (Ergonomics)  3068K->3059K(3584K), 0.0016623 secs]
[Full GC (Ergonomics)  3070K->3068K(3584K), 0.0028357 secs]
[Full GC (Allocation Failure)  3068K->3068K(3584K), 0.0017616 secs]
java.lang.OutOfMemoryError: Java heap space
Dumping heap to java_pid3352.hprof ...
Exception in thread "main" java.lang.OutOfMemoryError: Java heap space
Heap dump file created [3855956 bytes in 0.017 secs]
[Full GC (Ergonomics)  3071K->376K(3584K), 0.0032068 secs]
	at reference.Test$OOMClass.<init>(Test.java:11)
	at reference.Test.testStrongReference(Test.java:22)
	at reference.Test.main(Test.java:15)

Process finished with exit code 1
```

可以看到，很快就抛出了OOM，原因是Java heap space，也就是堆内存不足。

如果运行testSoftReference方法，将会得到如下结果：

```java
[GC (Allocation Failure)  1019K->464K(3584K), 0.0019850 secs]
[GC (Allocation Failure)  1484K->844K(3584K), 0.0015920 secs]
[GC (Allocation Failure)  1868K->1860K(3584K), 0.0043236 secs]
[Full GC (Ergonomics)  1860K->1781K(3584K), 0.0044581 secs]
[Full GC (Ergonomics)  2802K->2754K(3584K), 0.0041726 secs]
[Full GC (Ergonomics)  2802K->2799K(3584K), 0.0031293 secs]
[Full GC (Ergonomics)  3023K->3023K(3584K), 0.0024830 secs]
[Full GC (Ergonomics)  3071K->3068K(3584K), 0.0035025 secs]
[Full GC (Allocation Failure)  3068K->405K(3584K), 0.0040672 secs]
[GC (Allocation Failure)  1512K->1567K(3584K), 0.0011170 secs]
[Full GC (Ergonomics)  1567K->1496K(3584K), 0.0048438 secs]
```

可以看到，并没有抛出OOM，而是进行多次了GC，可以明显的看到这一条：

```bash
[Full GC (Allocation Failure)  3068K->405K(3584K), 0.0040672 secs]
```

当内存不足时进行了一次Full GC，回收了大部分内存空间，也就是将大部分软引用指向的对象回收掉了。

## 小结

+ 软引用弱于强引用
+ 软引用指向的对象会在内存不足时被垃圾回收清理掉
+ JVM会优先回收长时间闲置不用的软引用对象，对那些刚刚构建的或刚刚使用过的软引用对象会尽可能保留
+ 软引用可以有效的解决OOM问题
+ 软引用适合用作非必须大对象的缓存

至此，本篇就告一段落了，这里只简单的介绍了软引用的作用以及用法。其实软引用并没有这么好，它的使用有一些可能是致命的缺点，如果想要更深入的了解软引用的运行原理以及软引用到底是在何时进行回收，又是如何进行回收的话，可以查看翻阅后续的章节。 



