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

## 什么是函数式编程

一提到Java函数式编程，联想到的很可能便是 Java 中的 `流操作` 和 ` lamada 表达式`，类似下面这样的代码：

```java
List<Integer> numbers = Arrays.asList(3, 2, 2, 3, 5);
List<Integer> squaresList = numbers.stream()
    .distinct()
    .map(i -> i*i)
    .collect(Collectors.toList());
```

Java8 中新增的 `函数式接口` 、 `流操作` 和 `lamada表达式` 确实给 Java 函数式编程带来了生机，但Java的函数式编程却并不是在 Java8 之后才有的。

很多人对于函数式编程的理解上存在一定的误区。很多人觉得，使用了流操作，把 stream 玩的飞起，就掌握了函数式编程的真谛，但其实跟函数式编程的核心思想相去甚远。

>  在学习一个新概念的时候，理解事物是什么和不是什么往往都很重要。

那么函数式编程究竟是什么呢？

> 函数式编程是用函数来编程的一种编程范式。

编程范式即编写程序的方法论，是“结构化编程”的一种，中心思想是尽量用函数来表示运算过程。

既然它是一种编程范式，就与特定语言无关，并非某种语言所特有的性质。函数式编程的主角是“函数”，这里说的函数并不是指普通的方法，可以用方法来表示一个函数，但函数本身有更加广泛的含义，也有更加合适的表示形式，后面文章中会进行说明。

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
  Function<Integer, Function<Integer, Integer>> add = x -> y -> x + y;
  Integer result = add.apply(arg1).apply(arg2);
  assert result.equals(arg1 + arg2);

  BinaryOperator<Integer> addFunction = getAddFunction();
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

这里的 `Function` 和 `BinaryOperator` 都是自定义的类，后面会说到，如果你没有用 `Java` 写过高阶函数，看到这段代码应该会有一种耳目一新的感觉，想不到 `Java` 还可以这么玩吧。 

2. 没有副作用

除了计算结果，调用函数没有别的副作用，不会影响外界的任何状态。没有副作用是函数式编程中十分重要的一点，也是判断代码是否是函数式的重要依据。这里说的副作用可能包含，但不限于：

- 更改文件系统
- 往数据库插入记录
- 发送一个 http 请求
- 改变外部变量
- 打印/log
- 获取用户输入
- 访问系统状态
- 抛出异常
- 打印到控制台或其它设备

这里说的“没有副作用”是指没有可观测的副作用。函数内部仍会进行变量的赋值和操作，但对于调用方来说，这些改变是无感知的，函数就如同一个黑盒：

* 接收一个参数
* 内部做一些不可描述的操作
* 返回一个值

对外部来说，调用函数只是为了获取函数计算的结果，不会打印日志，不会改变参数的值，函数调用跟函数计算的结果是完全等价的。

拿之前的例子进行说明：

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

因为在 `addFunc` 方法中，不仅仅计算了结果，还向输出了一行日志，这就是可观测到的副作用，因此这段代码就不是函数式的。

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

### 函数式编程 vs 命令式编程

与 `函数式编程` 相对的是 `命令式编程` ，



