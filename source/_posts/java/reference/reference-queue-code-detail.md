---
title: ReferenceQueue源码详解
tags: 
 - Java
 - 引用类型
categories: 编程
date: 2018-12-30 20:10:51
---

## 定义

ReferenceQueue是引用队列，用于存放待回收的引用对象.

## 说明

对于软引用、弱引用和虚引用，如果我们希望当一个对象被垃圾回收器回收时能得到通知，进行额外的处理，这时候就需要使用到引用队列了。 

在一个对象被垃圾回收器扫描到将要进行回收时，其相应的引用包装类，即reference对象会被放入其注册的引用队列queue中。可以从queue中获取到相应的对象信息，同时进行额外的处理。比如反向操作，数据清理，资源释放等。

## 使用例子

```java
public class ReferenceQueueTest {
    private static ReferenceQueue<byte[]> rq = new ReferenceQueue<>();
    private static int _1M = 1024 * 1024;

    public static void main(String[] args) {
        Object value = new Object();
        Map<WeakReference<byte[]>, Object> map = new HashMap<>();
        Thread thread = new Thread(ReferenceQueueTest::run);
        thread.setDaemon(true);
        thread.start();

        for(int i = 0;i < 100;i++) {
            byte[] bytes = new byte[_1M];
            WeakReference<byte[]> weakReference = new WeakReference<>(bytes, rq);
            map.put(weakReference, value);
        }
        System.out.println("map.size->" + map.size());
        
        int aliveNum = 0;
        for (Map.Entry<WeakReference<byte[]>, Object> entry : map.entrySet()){
            if (entry != null){
                if (entry.getKey().get() != null){
                    aliveNum++;
                }
            }
        }
        System.out.println("100个对象中存活的对象数量：" + aliveNum);
    }

    private static void run() {
        try {
            int n = 0;
            WeakReference k;
            while ((k = (WeakReference) rq.remove()) != null) {
                System.out.println((++n) + "回收了:" + k);
            }
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
    }
}
```

这里有一个小栗子，main方法中，创建了一条线程，使用死循环来从引用队列中获取元素，监控对象被回收的状态。然后循环往map中添加了100个映射关系，以下是运行结果：

```java
...前面省略了大量相似输出
85回收了:java.lang.ref.WeakReference@7106e68e
86回收了:java.lang.ref.WeakReference@1f17ae12
87回收了:java.lang.ref.WeakReference@c4437c4
map.size->100
100个对象中存活的对象数量：12
```

通过配合使用ReferenceQueue，可以较好的监控对象的生存状态。

## 成员变量

ReferenceQueue中内部成员变量也很少，主要有这么几个：

```java
static ReferenceQueue<Object> NULL = new Null<>();
static ReferenceQueue<Object> ENQUEUED = new Null<>();
```

有两个用来做为特殊标记的静态成员变量，一个是NULL，一个是ENQUEUE，上一篇中说的ReferenceQueue.NULL和ReferenceQueue.ENQUEUED就是这两个家伙。

来看看Null长什么样：

```java
private static class Null<S> extends ReferenceQueue<S> {
    boolean enqueue(Reference<? extends S> r) {
        return false;
    }
}
```

只是简单继承了ReferenceQueue的一个类，emmm，为什么不直接new一个ReferenceQueue呢？这里自然是有它的道理的，如果直接使用ReferenceQueue，就会导致有可能误操作这个NULL和ENQUEUED变量，因为ReferenceQueue中enqueue方法是需要使用lock对象锁的，这里覆盖了这个方法并直接返回false，这样就避免了乱用的可能性，也避免了不必要的资源浪费。

```java
static private class Lock { };
private Lock lock = new Lock();
```

跟Reference一样，有一个lock对象用来做同步对象。

```java
private volatile Reference<? extends T> head = null;
```

head用来保存队列的头结点，因为Reference是一个单链表结构，所以只需要保存头结点即可。

```java
private long queueLength = 0;
```

queueLength用来保存队列长度，在添加元素的时候+1，移除元素的时候-1，因为在添加和移除操作的时候都会使用synchronized进行同步，所以不用担心多线程修改会不会出错的问题。

## 内部方法

```java
// 这个方法仅会被Reference类调用
boolean enqueue(Reference<? extends T> r) { 
    synchronized (lock) {
        // 检测从获取这个锁之后，该Reference没有入队，并且没有被移除
        ReferenceQueue<?> queue = r.queue;
        if ((queue == NULL) || (queue == ENQUEUED)) {
            return false;
        }
        assert queue == this;
        // 将reference的queue标记为ENQUEUED
        r.queue = ENQUEUED;
        // 将r设置为链表的头结点
        r.next = (head == null) ? r : head;
        head = r;
        queueLength++;
        // 如果r的FinalReference类型，则将FinalRef+1
        if (r instanceof FinalReference) {
            sun.misc.VM.addFinalRefCount(1);
        }
        lock.notifyAll();
        return true;
    }
}
```

这里是入队的方法，使用了lock对象锁进行同步，将传入的r添加到队列中，并重置头结点为传入的节点。

```java
public Reference<? extends T> poll() {
    if (head == null)
        return null;
    synchronized (lock) {
        return reallyPoll();
    }
}

private Reference<? extends T> reallyPoll() {     
    Reference<? extends T> r = head;
    if (r != null) {
        head = (r.next == r) ?
            null : r.next;
        r.queue = NULL;
        r.next = r;
        queueLength--;
        if (r instanceof FinalReference) {
            sun.misc.VM.addFinalRefCount(-1);
        }
        return r;
    }
    return null;
}
```

poll方法将头结点弹出。嗯，没错，弹出的是头结点而不是尾节点，名义上，它叫ReferenceQueue，实际上是一个ReferenceStack（滑稽）。惊不惊喜，意不意外。<img src="/images/0001.png" width="50"/>

```java
/**
  * 移除并返回队列首节点，此方法将阻塞到获取到一个Reference对象或者超时才会返回
  * timeout时间的单位是毫秒
  */
public Reference<? extends T> remove(long timeout)
    throws IllegalArgumentException, InterruptedException{
    if (timeout < 0) {
        throw new IllegalArgumentException("Negative timeout value");
    }
    synchronized (lock) {
        Reference<? extends T> r = reallyPoll();
        if (r != null) return r;
        long start = (timeout == 0) ? 0 : System.nanoTime();
        // 死循环，直到取到数据或者超时
        for (;;) {
            lock.wait(timeout);
            r = reallyPoll();
            if (r != null) return r;
            if (timeout != 0) {
                // System.nanoTime方法返回的是纳秒，1毫秒=1纳秒*1000*1000
                long end = System.nanoTime();
                timeout -= (end - start) / 1000_000;
                if (timeout <= 0) return null;
                start = end;
            }
        }
    }
}

/**
 * 移除并返回队列首节点，此方法将阻塞到获取到一个Reference对象才会返回
 */
public Reference<? extends T> remove() throws InterruptedException {
	return remove(0);
}
```

这里两个方法都是从队列中移除首节点，与poll不同的是，它会阻塞到超时或者取到一个Reference对象才会返回。

聪明的你可能会想到，调用remove方法的时候，如果队列为空，则会一直阻塞，也会一直占用lock对象锁，这个时候，有引用需要入队的话，不就进不来了吗？

嗯，讲道理确实是这样的，但是注意注释，enqueue只是给Reference调用的，在Reference的public方法enqueue中可以将该引用直接入队，但是虚拟机作为程序的管理者可不吃这套，而是通过其它方式将Reference对象塞进去的，所以才会出现之前的栗子中，死循环调用remove方法，并不会阻塞引用进入队列中的情况。

## 应用场景

ReferenceQueue一般用来与SoftReference、WeakReference或者PhantomReference配合使用，将需要关注的引用对象注册到引用队列后，便可以通过监控该队列来判断关注的对象是否被回收，从而执行相应的方法。

主要使用场景：

1、使用引用队列进行数据监控，类似前面栗子的用法。

2、队列监控的反向操作

反向操作，即意味着一个数据变化了，可以通过Reference对象反向拿到相关的数据，从而进行后续的处理。下面有个小栗子：

```java
public class TestB {

    private static ReferenceQueue<byte[]> referenceQueue = new ReferenceQueue<>();
    private static int _1M = 1024 * 1024;

    public static void main(String[] args) throws InterruptedException {
        final Map<Object, MyWeakReference> hashMap = new HashMap<>();
        Thread thread = new Thread(() -> {
            try {
                int n = 0;
                MyWeakReference k;
                while(null != (k = (MyWeakReference) referenceQueue.remove())) {
                    System.out.println((++n) + "回收了:" + k);
                    //反向获取，移除对应的entry
                    hashMap.remove(k.key);
                    //额外对key对象作其它处理，比如关闭流，通知操作等
                }
            } catch(InterruptedException e) {
                e.printStackTrace();
            }
        });
        thread.setDaemon(true);
        thread.start();

        for(int i = 0;i < 10000;i++) {
            byte[] bytesKey = new byte[_1M];
            byte[] bytesValue = new byte[_1M];
            hashMap.put(bytesKey, new MyWeakReference(bytesKey, bytesValue, referenceQueue));
        }
    }

    static class MyWeakReference extends WeakReference<byte[]> {
        private Object key;
        MyWeakReference(Object key, byte[] referent, ReferenceQueue<? super byte[]> q) {
            super(referent, q);
            this.key = key;
        }
    }
}
```

这里通过referenceQueue监控到有引用被回收后，通过map反向获取到对应的value，然后进行资源释放等。

## 小结

+ ReferenceQueue是用来保存需要关注的Reference队列
+ ReferenceQueue内部实现实际上是一个栈
+ ReferenceQueue可以用来进行数据监控，资源释放等
