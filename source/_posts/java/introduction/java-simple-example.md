---
title: Java简单示例
tags: 
 - Java入门
 - Java
categories: 编程
date: 2018-12-28 19:00:06
---

> 人都有两面，一面天使，一面恶魔。

## 前言

为了统一风格，也为了增加趣味性，以后的文章都会以关卡的形式进行展示，这样也能让大家明确每一篇的目标，带着目的来阅读会更有方向感。

之前已经写过了Hello World，所以这次就换一个吧，这次的小目标便是——Java简易版计算器。

## 功能说明

第一版的Java计算器仅需要支持加法运算，用户输入两个数字，然后输出它们的和。

## 方法预习

如果是刚开始接触编程，也许你会毫无头绪，莫方，这是很正常的现象，因为你对于如何与计算机尤其是命令行进行交互毫无头绪。

所以在开始编码前，需要先预习一些必要的知识。

### 输出信息

先来认识一个方法 `System.out.println()`，这个方法会将你传入的信息输出到控制台中，emmm，什么是控制台？你运行程序后出现的那个黑不拉几的东西就叫做控制台，它是我们与计算机交互的一个最简单原始的方式。

来试验一下，还记得如何用idea创建一个类吗？不记得的话翻看一下[这里](java-ide.md)。

这次我们继续在hello包下面创建一个类叫做PrintTest，然后添加以下方法：

```java
package hello;

public class PrintTest {
    public static void main(String[] args){
        System.out.println("输出测试");
    }
}
```

然后我们点击运行，或者按键（`ctrl+shift+R`），就能看到下面的输出了：

{% asset_img java-simple-example-1.png java-example %}

先不要问这个System是个什么东西，只要知道这样可以输出一行信息就行了，需要注意的是这个方法默认会在末尾加一个换行符，如果想要不换行，可以试试 `System.out.print() `。

### 输入信息

上面已经说过如何输出信息了，现在来看看如何输入信息并进行读取。

先来认识一下Scanner类，Scanner类是用来从各种输入源读取信息的，它可以从各种输入源中读取信息，最常用的当然就是控制台输入。那如何用Scanner读取控制台的输入呢？这就要用到System.in了，来看栗子，我们再新建一个InputTest类：

```java
package hello;

import java.util.Scanner;

public class InputTest {
    public static void main(String[] args){
        Scanner in = new Scanner(System.in);
        System.out.print("请输入一个整数:");
        int a = in.nextInt();
        System.out.println("刚才输入的整数为：" + a);
    }
}
```

然后再运行一下，并且输入一个数字然后按回车键，这里我输入的是20：

{% asset_img java-simple-example-2.png java-example %}

如果对于具体的细节还不清楚，没有关系，先照做就好了。

## 代码编写

那么接下来就开始设计这个最简单的计算器了，首先我们要提示用户输入一个整数，就像上面那样：

```java
System.out.print("请输入一个整数:");
```

然后用一个int变量来存储，什么？不知道变量是干嘛的？没事，你可以先把它当做一个盒子，用来把用户输入的信息存放进去。

```java
Scanner in = new Scanner(System.in);
int a = in.nextInt();
```

然后提示用户再输入一个整数：

```java
System.out.print("请输入另一个整数:");
```

然后再用一个int变量来存储。

```java
int b = in.nextInt();
```

接下来计算两者的和，并存到第三个变量中：

```java
int sum = a + b;
```

最后输出这个和：

```java
System.out.print("这两个数的和为:" + sum);
```

所以整体代码如下：

```java
package hello;

import java.util.Scanner;

public class SimpleCalculator {
    public static void main(String[] args){
        System.out.print("请输入一个整数:");
        Scanner in = new Scanner(System.in);
        int a = in.nextInt();
        System.out.print("请再输入一个整数:");
        int b = in.nextInt();
        int sum = a + b;
        System.out.print("这两个数的和为:" + sum);
    }
}
```

输出如下：

{% asset_img java-simple-example-3.png java-example %}

这样，我们的简易版计算器就完成了。虽然简单，但还是建议你在自己电脑上实现一次，看代码和写代码是完全不一样的两种体验。

回顾一下本篇，我们设计了一个类`SimpleCalculator`，并编写了一个main方法，并在里面完成了简易版计算器的逻辑设计。也许你对于自己写的东西还有很多疑问，对象是什么？类又是什么含义？Scanner还可以做什么？前面的package有什么作用，import又是在干嘛？别着急，接着看后面的文章，相信你的疑问会一点一点消失。
