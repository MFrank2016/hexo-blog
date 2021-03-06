---
title: 【Java函数式编程】（一）开篇
date: 2019-10-10 09:12:01
tags: 
- 函数式编程
categorys:
- 编程
- Java
---

最近一直在研究Java函数式编程相关的内容，收获颇丰，对函数式编程也有了更深刻的理解，特此记录，希望能与大家分享和探讨。

## 说明

一提到Java函数式编程，联想到的很可能便是 Java 中的 `函数式接口`、 `流操作` 和 ` lamada 表达式`，脑海中浮现的可能会是类似下面这样的代码：

```java
List<Integer> numbers = Arrays.asList(3, 2, 2, 3, 5);
List<Integer> squaresList = numbers.stream()
    .distinct()
    .map(i -> i*i)
    .collect(Collectors.toList());
```

Java8 中新增的 `函数式接口` 、 `流操作` 和 `lamada表达式` 确实给 Java 函数式编程带来了更多的便利性，但 Java 函数式编程却并不是在 Java8 之后才有的。

很多人对于函数式编程的理解上存在一定的误区。很多人觉得，使用了流操作，把 `stream` 和 `lamada表达式` 玩的飞起，就掌握了函数式编程的真谛，但其实跟函数式编程的核心思想相去甚远。

>  在学习一个新概念的时候，理解事物是什么和不是什么往往都很重要。

## 什么是函数式编程

那么函数式编程究竟是什么呢？

> 函数式编程是用函数来编程的一种编程范式。

编程范式即编写程序的方法论，是“结构化编程”的一种，中心思想是尽量用函数来表示运算过程。

既然它是一种编程范式，就与特定语言无关，并非某种语言所特有的特性。

函数式编程的主角是“函数”，这里说的函数并不是指普通的方法。可以使用方法来表示一个“函数”，但“函数”本身有更加广泛的含义，也有更加合适的表示形式，后面文章中会进行说明。

### 函数式编程的特点

函数式编程有以下特点：

1. 函数是一等公民

即函数与其他数据类型一样，处于平等的地位，可以把函数赋值给其他变量，也可以当做参数进行传递，或者作为其他函数的返回值。

如果不太理解是什么意思，可以参考一下下面的代码：

```java
@Test
public void addFuntion() {
  Integer arg1 = 3;
  Integer arg2 = 7;
  // 定义加法函数并赋值给 add 变量
  Function<Integer, Function<Integer, Integer>> add = x -> y -> x + y;
  // 应用函数
  Integer result = add.apply(arg1).apply(arg2);
  assert result.equals(arg1 + arg2);

  // 从方法中返回函数
  BinaryOperator<Integer> addFunction = getAddFunction();
  // 将函数当成方法参数传入
  Integer result1 = execBiFunction(addFunction, arg1, arg2 * 2);
  assert result1.equals(arg1 + arg2 * 2);
}

private BinaryOperator<Integer> getAddFunction(){
  return x -> y -> x + y;
}

private <T> T execBiFunction(BinaryOperator<T> func, T arg1, T arg2){
  return func.apply(arg1).apply(arg2);
}
```

这里的 `add` 和 `addFunction` 都是一个函数，通过使用 `apply` 方法来传入参数，即可得到相应结果。

这里的 `Function` 和 `BinaryOperator` 都是自定义的类，后面会说到，如果你没有用 `Java` 写过高阶函数，看到这段代码应该会有一种耳目一新的感觉，想不到 `Java` 还可以这么玩吧。 

2. 没有副作用

除了计算结果，调用函数没有别的副作用，不会影响外界的任何状态。没有副作用是函数式编程中十分重要的一点，也是判断代码是否是函数式的重要依据。

这里说的副作用可能包含，但不限于：

- 更改文件系统
- 往数据库中插入、更新或删除数据
- 发送一个 http 请求
- 改变外部变量的值
- 访问系统状态
- 抛出异常
- 打印到控制台或其它设备

这里说的“没有副作用”是指没有可观测的副作用。函数内部仍会进行变量的赋值和操作，但对于调用方来说，这些改变是无感知的，函数就如同一个黑盒：

* 接收一个参数
* 内部做一些不可描述的操作
* 返回一个值

对外部来说，调用函数只是为了获取函数计算的结果，不会打印日志，不会改变参数的值，函数调用跟函数计算的结果是完全等价的。

我们再拿之前的例子进行说明：

```java
Integer arg1 = 3;
Integer arg2 = 7;
Function<Integer, Function<Integer, Integer>> add = x -> y -> x + y;
Integer result = add.apply(arg1).apply(arg2);
```

这段代码跟下面这段代码是完全等价的：

```java
Integer arg1 = 3;
Integer arg2 = 7;
Integer result = 3 + 7;
```

但跟下面这段代码就不等价了：

```java
@Test
public void addFuntion() {
    Integer arg1 = 3;
    Integer arg2 = 7;
    Integer result2 = addFunc(arg1, arg2);
}

private int addFunc(int i, int j){
    log.info("入参为：" + i + "," + j);
    return i + j;
}
```

因为在 `addFunc` 方法中，不仅仅计算了结果，还输出了一行日志，这就是可观测到的副作用，因此这段代码就不是函数式的。

当然，函数不可能完全没有副作用，函数会在某个时候返回一个值，而这个值可能是变化的，这就是一个副作用。它可能会造成一个内存耗尽的错误，或者堆栈溢出的错误，导致程序崩溃，这也是某种意义上的可观测到的副作用。此外，它还会造成写内存、加载线程、上下文切换等其他会影响到外界观测的作用。

所以函数式编程其实是编写“非故意的副作用”的程序，副作用是程序预期结果的一部分，非故意的副作用也应该越少越好。

3. 引用透明

没有副作用表示函数式编程不会对外界有所影响，但没有副作用并不足以让程序变成函数式的。同样，函数式编程也不能被外界所影响，换句话说，函数式程序的输出只能取决于自己的参数，这就意味着函数式代码不能从数据库、网络、文件、控制台等外部依赖中读取数据。

> 不被外界所影响的代码就是引用透明的

函数的运行不依赖于外部变量或状态，只依赖于输入的参数，任何时候，相同参数总是能得到相同的结果。而在其他编程范式中，返回值往往与系统状态有关，在不同的系统状态之下，返回值是不一样的。

引用透明的代码有以下特性：

* 独立存在
  * 不依赖任何外部设备，因此可以在任何上下文中独立使用它
* 确定性
  * 意味着相同的参数总能得到相同的结果
* 不会抛出任何 `Exception`
  * 它可能抛出错误如 `OOM` 或者 `SOE` ，但这些错误表示代码有bug，并不是调用方应该处理的
* 不会导致其他代码意外失败
  * 因为它不会改变参数或者外界数据
* 不会因为外部设备不可用、太慢或坏掉而崩溃
  * 因为它不会依赖于外部设备

下面两个图可以直观的看出引用透明和非引用透明的程序直接的区别。

![1.jpg](https://i.loli.net/2019/10/14/9yfuAgFINDzKSoO.jpg)

引用透明的程序除了获取参数和输出结果之外，不会影响外界，它的结果只取决于参数。

![2.jpg](https://i.loli.net/2019/10/14/RJUSOqiT8co24aW.jpg)

非引用透明的程序可能会从外界读写数据、写日志、改变外界对象、读取键盘输入、输出到屏幕等，结果是不可预测的。

看到这里，你可能会觉得很奇怪，不改变外部状态，也不依赖于外部的状态，这样的代码如何在复杂系统中交互呢？在复杂的软件系统中，往往不可避免的需要获取用户输入、在数据库中查询或者修改数据、通过 `RPC` 或 `HTTP` 接口来交互数据，那函数式编程岂不是就不适用了？

小场面，不要慌。函数式编程是对操作的一种高层次抽象，后面会有实例来介绍如何处理输入和输出数据，展示抽象在函数式编程中的魅力。

### 函数式编程 vs 命令式编程

与 `函数式编程` 相对的是 `命令式编程` ， `命令式编程` 是日常开发中使用最多的编程范式，因为它简单粗暴，不需要太多思考和设计，思路怎样跑，代码就怎样写，非常的随（wei）心（suo）所（yu）欲（wei）。

在命令式编程的风格中，程序由“做”事情的要素构成。“做”事情意味着一个初始状态、一个转换过程和一个终止状态。

传统的命令式风格的代码通常描述了一系列由条件判断区分的改变。

而函数式编程中，程序由“是”什么的元素组成，而不是“做”什么。

```java
Integer arg1 = 3;
Integer arg2 = 7;
Function<Integer, Function<Integer, Integer>> add = x -> y -> x + y;
Integer result = add.apply(arg1).apply(arg2);
```

以上面这段代码为例，3 和 7 应用于 add 并不会造出一个结果 10 ，它就是 10。这样描述好像有点奇怪，换句话来说，不管你参数如何，它的结果其实早就已经存在了，调用函数只是为了获取这个结果。

那么命令式编程中不是也能这样做吗？有时确实如此，但有时不改变程序的结果就无法做到，不改变外部变量和状态并不是命令式编程中要关注的内容，所以实际情况大多是“可以，但没必要”。

> 命令式编程和函数式编程的最大不同是，函数式编程没有副作用。

## 为什么要使用函数式编程

说了这么多，那到底是为什么要使用函数式编程呢？收益点是什么呢？毕竟看起来，函数式编程的理解和应用门槛就很高，要把平时的面条式代码改成这样麻烦的编程方式并非易事。

其实从前面的描述中，也能大致看出函数式编程的诸多优势：

* 函数式编程更容易推断。
  * 因为它们具有“确定性”。对于特定的参数，不同运行多少次，总是能得到相同的结果。在绝大部分情况下，你都可以证明程序是正确的，而不是在大量的测试后仍然不确定程序是否会在意外情况下出错。
* 函数式代码更容易测试。
  * 因为函数式程序没有副作用，所以不需要那些单元测试中经常出现的各种类型的mock。
* 函数式程序更加模块化。
  * 因为函数式程序的结果只取决于传入的参数，我们不必处理副作用，不必捕获异常，不必处理上下文变化，不必共享变化的状态，也没有并发的修改。
* 函数式编程让复合和重新复合变得更加简单。
  * 编写函数式程序，需要编写各种必要的基础函数，然后将它们复合成更高级别的函数，一直重复这个过程，直到得到一个能解决你的问题的函数。因为函数都是引用透明的，所以它们无需修改便能重用。
* 函数式程序是线程安全的
  * 因为函数式编程防止了共享状态的变化。

## 函数式编程实例

> talk is cheap, show me the code.

说再多，也不如一段代码来得更有解释力。

下面将展示一个示例，将命令式代码转化为函数式代码，希望从中能让你感受到函数式编程的魅力所在。

这是一个很简单的程序：用信用卡购买一个甜甜圈。

```java
public class DonutShop{
    
    public static Donut buyDonut(CreditCard creditCard){
        Donut donut = new Donut();
        creditCard.charge(Donut.price);
        return donut;
    }
}
```

这是常用的命令式风格的代码，在这段代码中，使用信用卡支付是一个副作用，信用卡支付往往包含调用银行接口，校验信用卡是否可用，是否授权，是否超出最大可用金额等。该函数返回一个甜甜圈。

这段代码的问题在于难以进行独立测试，如果你想要进行单元测试，要么就需要先联系银行获取测试用的信用卡信息，要么就必须要 `mock` 一个信用卡对象来代替真实的 `charge` 方法，并在测试之后验证 `mock` 对象的状态。

通常而言，每个 `mock` 都代表着对一个复杂对象的依赖，如果你想移除掉这些 `mock` 对象，那么就应该移除副作用。但由于最终还是需要执行“支付”这个动作，这是程序中不可避免的副作用，一个不错的解决方案就是返回值里添加一些额外的信息，来表示需要支付的账单。

因此，可以抽象出一个账单的概念，用 `Payment` 类来表示：

```java
public class Payment {
    
    public final CreditCard creditCard;
    public final int amount;
    
    public Payment(CreditCard creditCard, int amount){
        this.creditCard = creditCard;
        this.amount = amount;
    }
}
```

在函数式风格的代码中，程序由大量函数和不可变对象组成，`Payment` 类的字段均为 `public final`，表示属性公开，且不可修改，当然，改成 `private` 并添加相应的 `get` 方法也能达到同样的效果。

这个类包含了表示支付的数据，由一张信用卡和需要支付的金额组成。由于 `buyDonut` 方法需要返回 `Donut` 对象和 `Payment` 对象，因此需要创建一个类来包装。比如：

```java
public class Purchase {
    
    public final Donut donut;
    public final Payment payment;
    
    public Purchase(Donut donut, Payment payment){
        this.donut = donut;
        this.payment = payment;
    }
}
```

但这样组合并没有特别明确的意义，只是为了将两个值返回而已，如果下次是购买饼干，则需要创建一个新的类，这样未免太浪费资源了，因此更好的写法应该是进行泛化：

```java
public class Purchase<T> {
    
    public final T item;
    public final Payment payment;
    
    public Purchase(T item, Payment payment){
        this.item = item;
        this.payment = payment;
    }
}
```

这样就能传入任意类型的对象了。

其实在函数式编程中，经常会遇到类似需要返回两个对象的场景，因此可以进行进一步的抽象，使用 `Tuple` 元组类：

```java
public class Tuple<T, U> {
    
    public final T _1;
    public final U _2;
    
    public Tuple(T t, U u){
        this._1 = t;
        this._2 = u;
    }
}
```

有了这两个类，我们再来优化一下最开始的代码：

```java
public class DonutShop {
    public static Tuple<Donut, Payment> buyDonut(CreditCard creditCard){
        Donut donut = new Donut();
        Payment payment = new Payment(creditCard, Donut.price);
        return new Tuple<>(donut, payment);
    }
}
```

