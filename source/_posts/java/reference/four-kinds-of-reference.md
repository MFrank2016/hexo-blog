---
title: 强引用、软引用、弱引用和虚引用深入探讨
tags: 
 - Java
 - 引用类型
categories: 编程
date: 2018-12-29 19:55:51
---

## 前言

为了更灵活的控制对象的生命周期，在JDK1.2之后，引用被划分为强引用、软引用、弱引用、虚引用四种类型。

引用类型在日常开发中并不常关注，也很少注意到，因此很多人忽略了它们的存在，而事实上，引用类型在Java体系中扮演着十分重要的角色，如果要想对Java体系有一个更深层次的理解，那么了解和掌握这些引用的用法是十分必要的。

在正式开始前，我们先来上两道开胃菜。<img src="/images/0046.png" width="50"/>

## 为什么需要回收

每一个Java程序中的对象都会占用一定的计算机资源，最常见的，如：每个对象都会在堆空间上申请一定的内存空间。但是除了内存之外，对象还会占用其它资源，如文件句柄，端口，socket等等。当你创建一个对象的时候，必须保证它在销毁的时候会释放它占用的资源。否则程序将会在OOM中结束它的使命。<img src="/images/0005.png" width="50"/>

在Java中，程序员不需要关心对象的内存具体如何分配和如何释放，不需要了解其中回收的细节，也不需要担心会将同一个对象释放两次而导致内存损坏。JVM有自动进行内存管理的神器——垃圾回收器，垃圾回收器会自动回收那些不再使用的对象，并释放掉它们的内存，你只需要保证那些不再被使用的对象的所有强引用都已经被释放掉了。  

虽然垃圾回收器确实让Java中的内存管理比C、C++中的内存管理容易许多，但是你不能对于内存完全不关心。如果你不清楚JVM到底会在什么条件下才会对对象进行回收，那么就有可能会不小心在代码中留下内存泄漏的bug。

因此，关注对象的回收时机，理解JVM中垃圾收集的机制，可以提高对于这个问题的敏感度，也能在发生内存泄漏问题时更快的定位问题所在。想了解更多关于垃圾回收相关的细节，可以参考[这篇文章](../jvm/garbage-collection.html)。

## 为什么需要引用类型

在JDK 1.2以前的版本中，如果一个对象没有被任何变量引用，那么程序就无法再使用这个对象。也就是说，只有当对象处于可达（reachable）状态时，程序才能使用它。只有在对象没有任何其他对象引用它时，垃圾回收器才会对它进行回收。对象只有被引用和没有被引用两种状态。这种方式无法描述一些“食之无味，弃之可惜”的对象。 而很多时候，

我们希望存在这样一些对象：当内存空间足够时，可以将它们保存在内存中，不进行回收；当内存空间变得紧张时，允许JVM回收这些对象。大部分缓存都符合这样的场景。

从JDK 1.2版本开始，Java对引用的概念进行了扩充，对象的引用分成了4种级别，从而使程序开发者能更加灵活地控制对象的生命周期，更好的控制创建的对象何时被释放和回收。

引用类型是与JVM密切合作的类型，有些引用类型甚至允许其引用对象在程序中仍需要的时候被JVM释放。

这4种引用类型的强度由高到低依次为：**强引用**、**软引用**、**弱引用**和**虚引用**。

有了这些引用类型之后，可以一定程度上增加对垃圾回收的粒度把控，可以让垃圾回收器在更合适的时机回收掉那些可以被回收掉的对象，而并不仅仅是只回收不再使用的对象。

这些引用类型各有特点，各有各的适用场景，清楚的了解和掌握它们的用法可以帮助你写出更加健壮的代码。<img src="/images/148.png" width="50"/>

## 实力翻车

下面欢迎来到大型翻车现场，接下来将实力演示一波因为强引用过多导致的翻车例子。

如果你需要在整个程序运行期间保存一些对象（因为它们的初始化很耗费时间和资源），你可能会使用静态集合对象来存储并且在代码中随处使用它们。

```java
public static Map<K, V> storedObjs = new HashMap<>();
```

但是这样，你就能成功阻止垃圾回收器对集合中的对象进行回收和销毁。从而顺利引发OOM。例如：

```java
public class OOMTest {
    public static List<Integer> cachedObjs = new ArrayList<>();
 
    public static void main(String[] args) {
        for (int i = 0; i < 100_000_000; i++) {
            cachedObjs.add(i);
        }
    }
}
```

输出如下：

```bash
Exception in thread “main” java.lang.OutOfMemoryError: Java heap space
```

<img src="/images/0005.png" width="50"/>这样就符合预期的翻车了。但你也许会说，谁会这么无聊，创建这么多变量。

嗯，确实是的，但是别忘了，一个程序可能会运行很长时间，几个月，甚至几年（如果你的代码和公司足够健壮的话），如果期间不断的创建变量而不清理的话（像上面那样把HashMap当缓存使用，不断往里面添加内容但是却不做删除），是有可能会导致这种情况发生的。

## 内容编排

 接下来的文章将从以下几方面对这四种引用进行介绍：

+ <LabelBlock>简要介绍    </LabelBlock>
  + [强引用](./strong-reference.html)
  + [软引用](./soft-reference.html)
  + [弱引用](./weak-referecen.html)
  + [虚引用](./phantom-reference.html)
+ <LabelBlock>源码剖析    </LabelBlock>
  + [Reference源码详解](./reference-code-detail.html)
  + [ReferenceQueue源码详解](./reference-queue-code-detail.html)
  + [SoftReference源码详解](./soft-reference-code-detail.html)
  + [WeakReference源码详解](./weak-reference-code-detail.html)
  + [PhantomReference源码详解](./phantom-reference-code-detail.html)
  + [FinalReference与Finalizer详解](./final-reference-code-detail.html)
+ <LabelBlock>总结    </LabelBlock>
  + [四种引用类型总结](./reference-summary.html)

> 注意
> 本系列文章都是以`JDK1.8` 版本的代码进行分析，不同版本中代码会略有差异。 

如果只是想要对这些引用进行简单了解，那么看完简要介绍部分即可，如果想要有更深入的研究，可以继续查阅源码剖析部分。<img src="/images/0003.png" width="50"/>