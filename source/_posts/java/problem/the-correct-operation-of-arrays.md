---
title: 【问题总结】万万没想到，竟然栽在了List手里
date: 2019-05-18 09:00:02
tags: 
 - Java
 - 常见问题
categories: 编程
---

## 说明

昨天同事开发的时候遇到了一个奇怪的问题。

![](https://i.loli.net/2019/05/18/5cdf7e9f2047e23793.png)

使用Guava做缓存，往里面存一个List，为了方便描述，称它为列表A，在另一个地方取出来，再跟列表B中的元素进行差集处理，简单来说，就像是下面这样：

```java
public class ArrayListTest {
    // 方便起见，这里用HashMap来做缓存
    private Map<String, List<Long>> cache = new HashMap<>();
    
    private void save(){
        List<Long> listA = createListA();
        cache.put("listA", listA);
    }
    
    private void get(){
        List<Long> listB = createListB();
        List<Long> listA = cache.get("listA");
        listA.removeAll(listB);
    }
    
    private List<Long> createListA(){
        ···
    }

    private List<Long> createListB(){
        ···
    }

    public static void main(String[] args){
        ArrayListTest test = new ArrayListTest();
        test.save();
        test.get();
    }
}
```

先调用save方法，然后调用get方法，然后就抛出了异常：

![](https://i.loli.net/2019/05/18/5cdf7ec98fddd17705.png)

```shell
Exception in thread "main" java.lang.UnsupportedOperationException
	at java.util.AbstractList.remove(AbstractList.java:161)
	at java.util.AbstractList$Itr.remove(AbstractList.java:374)
	at java.util.AbstractCollection.removeAll(AbstractCollection.java:376)
    ...
```

![](https://i.loli.net/2019/05/18/5cdf6539270f631562.jpg)

## 问题探索

究竟是人性的泯灭还是道德的沦丧，一个小小的List竟然也玩不转了，面对突如其来的打击，我跟同事都开始反思，复制粘贴一时爽，debug火葬场。

但作为一名优秀的程序猿，怎么能被这点困难所难倒呢？于是开始了问题排查之旅。

先来验证一下自己对ArrayList是否有什么误解：

```java
@Test
public void testArrayList() {
    List<Long> listA = new ArrayList<>();
    listA.add(1L);
    listA.add(2L);
    List<Long> listB = new ArrayList<>();
    listB.add(2L);
    listB.add(3L);
    listA.removeAll(listB);
    System.out.println(JSON.toJSONString(listA));
}
```

输出如下：

```
[1]
```

嗯，看来并没有。

![](https://i.loli.net/2019/05/18/5cdf7f0e86fbc84596.png)

再回过头看看，抛出的异常是 `UnsupportedOperationException` 异常，而且是在 `AbstractList` 里抛出的，于是打开了 `AbstractList`的源码。

```java
public E remove(int index) {
    throw new UnsupportedOperationException();
}
```

`AbstractList` 类对remove方法的默认实现就是直接抛出一个异常，所以如果子类并没有覆盖该方法，就会出现上面的问题。

那么问题应该就出在列表A的创建方式上。

结果一找，发现列表A是通过 `Arrays.asList()` 创建的，再跟进代码：

```java
public static <T> List<T> asList(T... a) {
    return new ArrayList<>(a);
}
```

感觉好像也没哪里不对，这里也是创建一个 `ArrayList` ，讲道理的话，应该没问题才对，不过等等，`ArrayList` 好像没有能传入可变长参数的构造函数吧，于是朝着这个`ArrayList`小手一点，终于发现了问题所在。

原来通过 `Arrays.asList()` 创建的 `List` 对象是通过实例化 `Arrays` 内部类 `ArrayList` 来创建的，所以这个 `ArrayList` 并不是我们常用的那个 `ArrayList`。

![20190518101356.png](https://i.loli.net/2019/05/18/5cdf6a655276187010.png)

![20190518101255.png](https://i.loli.net/2019/05/18/5cdf6a292b1d070453.png)

这个内部类并没有覆盖父类 `AbstractList` 的 `remove` 方法，所以调用的时候就会直接调用父类的 `remove` 方法，于是便发生了上面的异常。

## Arrays.asList的正确打开方式

为了更好的使用这里方法，我们先来看看它的注释说明：

```
 /**
* Returns a fixed-size list backed by the specified array.  (Changes to
* the returned list "write through" to the array.)  This method acts
* as bridge between array-based and collection-based APIs, in
* combination with {@link Collection#toArray}.  The returned list is
* serializable and implements {@link RandomAccess}.
*
* <p>This method also provides a convenient way to create a fixed-size
* list initialized to contain several elements:
* <pre>
*     List&lt;String&gt; stooges = Arrays.asList("Larry", "Moe", "Curly");
* </pre>
*
* @param <T> the class of the objects in the array
* @param a the array by which the list will be backed
* @return a list view of the specified array
*/
```

从说明可以发现，有这么几点需要注意：

1、该方法返回的是一个固定长度的列表

所以它的长度是不能被改变的，也就不能对它进行添加和删除元素的操作，从它的内部类ArrayList的方法列表也可以看出，并没有覆盖add和remove方法，因此对这两个方法的调用都会导致抛出异常。

虽然不能改变列表的长度，但是可以改变列表中的元素，以及元素的位置。比如通过set方法来重新设值，通过replaceAll方法来批量替换，通过sort方法来排序等等。

2、任何对列表的改动都会回写到原来是数组

也就是说对返回的列表进行的任何修改操作，都会导致原数组的改变。可以通过一个Test来测试一下：

```java
@Test
public void testArrays() {
    Long[] longs = {1L,2L,4L,3L};
    List<Long> longList = Arrays.asList(longs);
    System.out.println("longList:" + JSON.toJSONString(longList) + "longs:" + JSON.toJSONString(longs));

    longList.set(1, 5L);
    System.out.println("longList:" + JSON.toJSONString(longList) + "longs:" + JSON.toJSONString(longs));

    longList.replaceAll(a -> a + 1L);
    System.out.println("longList:" + JSON.toJSONString(longList) + "longs:" + JSON.toJSONString(longs));

    longList.sort(Long::compareTo);
    System.out.println("longList:" + JSON.toJSONString(longList) + "longs:" + JSON.toJSONString(longs));

    longs[2] = 7L;
    System.out.println("longList:" + JSON.toJSONString(longList) + "longs:" + JSON.toJSONString(longs));
}
```

输出如下：

```
longList:[1,2,4,3]longs:[1,2,4,3]
longList:[1,5,4,3]longs:[1,5,4,3]
longList:[2,6,5,4]longs:[2,6,5,4]
longList:[2,4,5,6]longs:[2,4,5,6]
longList:[2,4,7,6]longs:[2,4,7,6]
```

注意最后一个输出，我们修改原数组的元素，也会导致列表元素的改变，究其原因，当然是因为列表只是将数组封装了起来而已，最终指向的都是同一个内存地址，因此修改自然也是同步的。

3、不能使用基本数据类型数组来作为参数

举个栗子：

```java
@Test
public void testArrays2() {
    int[] ints = { 1, 2, 3 };
    List list = Arrays.asList(ints);
    System.out.println(list.size());
}
```

这里并不会报错，而是会输出`1`。为什么呢？

再回过头去看下说明：

```java
@param <T> the class of the objects in the array
```

参数的类型T指的是数组中的元素类型，如果数组中元素类型是基本类型，就会把整个数组当成一个元素，我们把上面的栗子稍微修改一下就清楚了。

```java
@Test
public void testArrays2() {
    int[] ints = { 1, 2, 3 };
    System.out.println(ints.getClass());
    List list = Arrays.asList(ints);
    System.out.println(JSON.toJSONString(list));
}
```

输出如下：

```
class [I
[[1,2,3]]
```

注意第二行的输出是一个二维数组。变长参数本质上就是一个对象数组，所以如果传入一个Integer数组，就能正常接收：

```java
@Test
public void testArrays2() {
    Integer[] ints = { 1, 2, 3 };
    System.out.println(ints.getClass());
    List list = Arrays.asList(ints);
    System.out.println(list.size());
}
```

```
class [Ljava.lang.Integer;
3
```

## 总结

至此，关于 `Arrays.asList()` 的探索之旅就结束了，遇到问题一般跟一跟源码就差不多能解决了，但对于常用的类，如果对其内部的运行机制不熟悉的话，代码就会容易出现一些不符合预期的行为，报错的异常并不可怕，因为可以根据异常很快定位，最怕的就是不报错，能正常运行，但是数据处理却是错误的，等到真正发现的时候，可能已经造成了难以挽回的损失。

![](https://i.loli.net/2019/05/18/5cdf7f4a7282234332.png)

看来主动阅读源码还是相当有必要的，其实`Arrays.asList()`并不难使用，推而广之，就像Guava、fastjson这些模块，或者spring、redis、dubbo之类，学习使用并不难，但如果不熟悉内部运行机制，仅仅当成一个黑盒的话，无法探索内部的精妙设计，遇到问题也比较难处理，如果只是把功能框定在其设定的能力范围之内，就没有办法进行定制化的改造。

嗯，看来我的历练路程还很长啊。最后用荀子的一句话来共勉吧。

“路虽弥，不行不至，

事虽小，不做不成。”

![](https://i.loli.net/2019/05/18/5cdf7daf74e9241031.png)