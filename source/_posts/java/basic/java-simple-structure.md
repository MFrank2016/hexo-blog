---
prev: java-basic
next: java-variable

---

# Java 程序基本结构

>“No matter how small and unimportant what we are doing may seem, if we do it well, it may soon become the step that will lead us to better things.”				— Channing Pollock, Writer
>
>「不管我们现在所做的事看起来有么的微不足道或不重要，如果我们认真的做，它可能很快就会成为通往美好事物的踏石阶。」								– 詹宁‧布鲁克 (作家)

## 关卡说明

>关卡描述：Java程序具有一些固定的形式，本篇将来对此进行简单的说明介绍。
>
>过关条件：理解Java程序的基本组成结构
>
>过关奖励：经验+10
>
>关卡难度：⭐️

## 一个简单的程序

为了简单起见，本篇以及之后的很多篇里，都会设计很多“玩具”代码，与实际中Java的设计可能相去甚远，本系列中的代示例都是为了说明一些相关概念，让你能够更好的理解Java的相关特性。

下面再来看看上一篇中的栗子：

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

这是一个非常简单的能够运行的Java程序，它有一些基本的结构，下面将对其一一介绍。

前面两句的作用已经在[上一篇](../introduction/java-package.md)中进行说明了，这里就不赘述了。接下来是类的定义：

```java
public class SimpleCalculator {
    ...
}
```

这里是定义了一个叫做`SimpleCalculator`的类，`public` 称为*访问修饰符*，用于控制代码的访问级别，关于这部分的内容，会在之后的章节中进行详细的介绍。在这里，`public` 表示这个类的访问权限为任何外部类均可以访问。`class` 是定义一个类的关键字，它的后面是类。关于类的命名规范在之后也会有详细说明。不要忘了后面还有一对大括号，表示类的内部内容。

需要注意的是，**Java是区分大小写的**，如果出现了大小写拼写错误，程序是无法正确运行的。

```java
public static void main(String[] args){
    ...
}
```

在类的内部定义了一个main方法，为什么说它是一个方法而不是类呢？因为它位于一个类的内部，并且没有用class关键字修饰，而且符合方法的定义规范。`public`同样可以修饰方法，表示这个方法对于外部类是公开的，可以访问的，`static`表示这是一个静态方法（先不要纠结什么是静态方法），`void`表示这个方法的返回值类型，`main`为方法名，后面跟上一对小括号，里面是参数列表，这里为`String[] args`，表示它接受一个String数组作为参数，`args`为参数名。

也许上面还有许多概念你还不清楚是什么意思，不要着急，先不要纠结于这些细节，先从整体上把握，继续看下去，后面的文章中会有说明。

需要注意的是，`main`方法是java程序中一个十分特殊的方法，它是整个程序的入口，也就是说，程序会从`main`方法开始执行，因此，如果想要代码能够执行，在类的源文件中必须包含一个`main`方法。

方法中的方法体则是我们为了实现功能而设计的自定义代码，在Java中，每个句子必须用分号结束。

```java
System.out.print("请输入一个整数:");
Scanner in = new Scanner(System.in);
int a = in.nextInt();
System.out.print("请再输入一个整数:");
int b = in.nextInt();
int sum = a + b;
System.out.print("这两个数的和为:" + sum);
```

Java中，点号是用来调用方法或者使用对象的，如：`System.out.print(...)`表示使用`System.out`对象并调用它的`print`方法。

至此，本篇的Java的基本结构就介绍完毕了，希望通过本篇，你能知道一个简单Java程序的结构是怎样的以及main方法有什么作用。

