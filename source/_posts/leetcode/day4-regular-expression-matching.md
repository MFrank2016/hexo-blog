---
title: 【LeetCode】正则表达式匹配
date: 2019-03-03 15:40:30
tags: 
 - Java
 - 算法
 - LeetCode
categories: 编程
---

## 题目描述

给定一个字符串 (s) 和一个字符模式 (p)。实现支持 '.' 和 '*' 的正则表达式匹配。

'.' 匹配任意单个字符。
'*' 匹配零个或多个前面的元素。
匹配应该覆盖整个字符串 (s) ，而不是部分字符串。

说明:

s 可能为空，且只包含从 a-z 的小写字母。
p 可能为空，且只包含从 a-z 的小写字母，以及字符 . 和 *。
示例 1:

```bash
输入:
s = "aa"
p = "a"
输出: false
解释: "a" 无法匹配 "aa" 整个字符串。
```

示例 2:
```bash
输入:
s = "aa"
p = "a*"
输出: true
解释: '*' 代表可匹配零个或多个前面的元素, 即可以匹配 'a' 。因此, 重复 'a' 一次, 字符串可变为 "aa"。
```

示例 3:
```bash
输入:
s = "ab"
p = ".*"
输出: true
解释: ".*" 表示可匹配零个或多个('*')任意字符('.')。
```

示例 4:
```bash
输入:
s = "aab"
p = "c*a*b"
输出: true
解释: 'c' 可以不被重复, 'a' 可以被重复一次。因此可以匹配字符串 "aab"。
```

示例 5:
```bash
输入:
s = "mississippi"
p = "mis*is*p*."
输出: false
```

题目难度：⭐⭐⭐

## 题目解析

这是一道有点难度的题，如果你看了一遍题目之后，没有什么好的想法，不用心急，深呼吸，让我们一起来探索如何解决这道题。

其实题目的要求，就是实现一个最简单的正则表达式，即`.`与`*`的匹配，一提到正则表达式，你也许会想到形如 `^[A-Z]:\\{1,2}[^/:\*\?<>\|]+\.(jpg|gif|png|bmp)$` 之类的一大串乱七八糟的代码，觉得看着都蛋疼，还要让我来实现？？？emmmm，不要方，问题不大，不要被`正则表达式`这个名号给吓到，要相信，问题总比方法多🤣。何况这里只需要解析两个特殊字符，岂不是小菜一碟。

明人不说骚话，撸起袖子就开干。

先重新阅读一遍题目，对题目要求的理解和把握很关键，这决定了之后的思考会不会跑偏，后面的几个示例可以用来验证自己理解是否正确。

从后面给的栗子里可以看出，题目的意思是要求字符串s与字符模式p能完全匹配才能算是通过，而不是在s中找到一个p能匹配的子字符串。

脑袋一拍，那一个字符一个字符来匹配不就完事了？嗯，先试试看。把题中的栗子拿出来画成图，然后进行观察。

{% asset_img example1.png example1 %}

{% asset_img example2.png example2 %}

{% asset_img example3.png example3 %}

{% asset_img example4.png example4 %}

在形成自己的思路后，一定要对这几个栗子进行验证，不然代码写完以后才发现理解错了题目的意思就很尴尬了。🌝

对于一个位于字符模式p中的字符c来说，只有三种情况：

1. c == '.'
2. c == '*'
3. c 为其他普通字符

我们先来看第一种情况，当`c == '.'`的时候，因为可以匹配任意字符，那么，直接跳过即可，对于第三种情况，那么只要`s`中对应的字符字符`c`相同即可，你看，很简单吧，我们已经完成三分之二了。接下来，再来看看最后一种情况。

如果`c == *`，那么代表可以匹配零个或者多个前面的字符，比如`a*`可以匹配`a`、`aaaa`、`aaaaa`也可以匹配空字符，所以它其实是个修饰符，用来修饰它前面的字符，必须要跟其他字符一起使用，所以在我们在一个个遍历模式串中的字符的时候，还需看看后面跟的字符是不是`*`，如果是的话，那么就要进行特殊处理了。

`*`代表匹配0个或多个它前面的字符，所以有两种情况，一种是0个，一种是多个。

梳理一下思路，每次从p中拿出一个字符来与s中的字符进行匹配，如果该字符后续的字符不是`*`，那么直接与s中对应字符进行匹配判断即可，如果匹配上了，那么就将两个游标都往后移动一位。如果匹配过程中遇到不相等的情况，则直接返回false。如果后续字符是`*`，那么就如上面所分析的，分成两种情况，一种是匹配0个，那么只需要跳过p中的这两个字符，继续与s中的字符进行比较即可，如果是匹配多个，那么将s中的游标往后移动一个，继续进行判断，这两个条件只要其中一个能满足即可。

对于上面分析`*`字符的说明也许还不够清晰，继续画图：

{% asset_img solution1-1.png solution1 %}

{% asset_img solution1-2.png solution1 %}

{% asset_img solution1-3.png solution1 %}

{% asset_img solution1-4.png solution1 %}

等等，你有没有闻到一丝递归的味道，既然对于每个在模式串中的字符都可以采用相同的策略进行处理，那不就是暗示这里可以使用递归吗。机智如我😝

## 递归解法

先来写一下伪代码来继续理清思路，毕竟这可是一道复杂度为三星级别的题，万万不可轻敌。

```java
boolean isMatch (String s, String p){
    从p中取出字符c1，从s中取出字符d1
    从p中再取一个字符c2
    if (c2 == '*'){
        跳过c1与c2或者将s的游标往后移动一位
        return isMatch(s,p.subString(2)) || (( c1 == '.' || c1 == d1) && isMatch(s.subString(1),p)));
    } else if(c1 == '.'){
        直接跳过
        return isMatch(s.subString(1),p.subString(1);
    } else {
        普通字符直接比较
        return c1 == d1 && isMatch(s.subString(1), p.subString(1));
    }
}
```

emmm，这个伪代码好像不太合格，几乎把代码写完了，23333，接下来只需要考虑一下边界情况，把代码补全就行了，当然，还可以将代码美化一下：

```java
public boolean isMatch(String s, String p){
    if (p.length() <= 0) return s.length() <= 0;
    boolean match = (s.length() > 0 && (s.charAt(0) == p.charAt(0) || p.charAt(0) == '.'));
    if (p.length() > 1 && p.charAt(1) == '*'){
        return isMatch(s, p.substring(2)) || (match && isMatch(s.substring(1), p));
    } else {
        return match && isMatch(s.substring(1), p.substring(1));
    }
}
```

大功告成，提交一下。

{% asset_img result1.png result1 %}

emmm，递归的效率一般都比较差，只击败了28%的用户。

当然，一般能用递归解决的地方，都可以使用非递归的方式解决，下面，我们来使用另一种解决方案。

## 动态规划解法

### 动态规划简介

动态规划？？？emmm，如果你不经常接触算法的话，也许对这个名词不太熟悉，所以我先简单的介绍一下。

动态规划，简单来说就是，动态的去进行，规划。😂言归正传，其实动态规划也是一种分治的思想，将问题分解成一个个子问题，通过解决所有子问题，来求得原问题的解，一般用于求解最优问题。但是跟分治法不同的地方在于，动态规划的子问题往往是相互关联的，拿最简单的斐波拉契数列来说，我们使用分治的思想，对于求`fib(6)`，使用的公式是`fib(6) = fib(5) + fib(4)`，于是将原来的问题便转化为求解`fib(5)`和`fib(4)`，继续递归，`fib(5) = fib(4) + fib(3)`，然后再继续递归`fib(4) = fib(3) + fib(2) `、`fib(3) = fib(2) + fib(1) `这里`fib(1) = 1` 和 `fib(2) = 1`为初始条件，于是就能求出`fib(6)`，初看起来似乎没什么毛病，但是仔细想一想，由于每次递归都是无状态的，所以其实做了很多重复的计算，画个图来感受一下：

{% asset_img fib6.png fib %}

这里将fib(4)重复算了2次，fib(3)算了3次，这还只是算fib(6)，如果是fib(66)呢？那将会有大量的重复计算，这是非常浪费时间的。

动态规划就可以很好的解决这个问题，动态规划的思想跟上面是一样的，但不同的是，动态规划会将每次计算的结果存起来，因此就解决了。简单一点理解，就是在分治的基础上加入了一个状态数组，来存储中间计算的结果，以减少重复计算的耗时。当然，动态规划又分为两种，一种是自顶向下，就是刚才所说的方法，另一个种是自底向上，还是拿上面的斐波拉契数列来说，要计算fib(6)，因此我们先计算`fib(3) = fib(2) + fib(1) `，再计算`fib(4) = fib(3) + fib(2) `和`fib(5) = fib(4) + fib(3)`，这样，就能算出`fib(6) = fib(5) + fib(4)`的结果了。

在动态规划中有几个比较关键的概念：子问题，状态，状态空间，初始状态，状态转移方程。

子问题：与原问题形式相同或者类似，只不过规模变小了，子问题都解决后，原问题即解决。

状态：与子问题相关的各个变量的一组取值即为状态，状态与子问题是一对一或一对多的关系，代表着子问题的解。上面的栗子，状态就是`fib(n)`的值。

状态空间：由所有状态构成的集合，上面的栗子比较简单，状态空间是一维空间。

状态初始条件：即状态的初始状态，上面的栗子里`fib(1) = 1`和`fib(2) = 1`就是初始条件。

状态转移方程：用来表示状态之间是如何转换的方程，即如何从一个或者多个已知的状态求出另一个状态，可以使用递推公式表示。上面栗子的公式为`fib(n) = f(n - 1) + f(n -2)  (n > 2)`

### 算法过程

关于动态规划的介绍就结束了，接下来我们来看如何在这道题上面使用。

我们先来考虑自顶向下的算法。为方便起见，假定使用符号`s[i:]`表示字符串s中从第i个字符到最后一个字符组成的子串，p[j:]则表示模式串p中，从第j个字符到最后一个字符组成的子串，使用 `match(i,j)` 表示`s[i:]`与`p[j:]`的匹配情况，如果能匹配，则置为true，否则置为false。这就是各个子问题的状态。

那么对于`match(i,j)`的值，取决于`p[j + 1]`是否为'*'。

curMatch = i < s.length() && s[i] == p[j] || p[j] == '.';
1. p[j + 1] != '*'，match(i,j) = curMatch && match(i + 1, j + 1)
2. p[j + 1] == '*'，match(i,j) = match(i, j + 2) || curMatch && match(i + 1, j)

这样表述一下是不是就清晰了不少。

以`s = "aab"; p = "c*a*b"`为例，先构建一个二维状态空间来存储中间计算得出的状态值。横向的值代表i，纵向的值代表j，match(0,0)的值即问题的解，用`f`代表`false`，`t`代表`true`。

{% asset_img solution2-1.png solution2 %}

接下来描述一下后续的计算过程：

```java
1. 求match(0,0): i = 0; j = 0; curMatch = false;
2. p[1] == * -> match(0,0) = match(0,2) || false && match(1,0)
3. 转化为求子问题match(0,2)和match(1,0)
4. 求match(0,2): i = 0; j = 2; curMatch = true;
5. p[1] == * -> match(0,2) = match(0,4) || true && match(1,2)
6. 求match(0,4): i = 0; j = 4; curMatch = false;
7. j + 1 == 5 >= p.length() -> match(0,4) = curMatch = false;
8. match(0,4) = false;
9. 回溯到第五步，求match(1,2): i = 1; j = 2; curMatch = true;
10. p[3] == * -> match(1,2) = match(1,4) || true && match(2,2)
11. 求match(1,4): i = 1; j = 4; curMatch = false;
12. j + 1 == 5 >= p.length() -> match(1,4) = curMatch = false;
13. match(1,4) = false;
14. 回溯到第10步，求match(2,2): i = 2; j = 2; curMatch = false;
15. p[3] == * -> match(2,2) = match(2,4) || false && match(3,2)
16. 求match(2,4): i = 2; j = 4; curMatch = true;
17.  j + 1 == 5 >= p.length() -> match(2,4) = curMatch = true;
18. match(2,4) = true;
19. 回溯到第15步。
20. match(2,2) = true;
21. 回溯到第10步。
22. match(1,2) = true;
23. 回溯到第5步。
24. match(0,2) = true;
25. 回溯到第2步。
26. match(0,0) = true;
27. 问题解决
```

{% asset_img solution2-2.png solution2 %}

你看，其实很简单吧。😅

接下来转化成代码：

```java
enum Result {
    TRUE, FALSE
}

class Solution {
    // 状态空间
    Result[][] memo;

    public boolean isMatch(String text, String pattern) {
        memo = new Result[text.length() + 1][pattern.length() + 1];
        return match(0, 0, text, pattern);
    }

    public boolean match(int i, int j, String text, String pattern) {
        if (memo[i][j] != null) {
            return memo[i][j] == Result.TRUE;
        }
        boolean ans;
        if (j == pattern.length()){
            ans = i == text.length();
        } else{
            boolean curMatch = (i < text.length() &&
                                   (pattern.charAt(j) == text.charAt(i) ||
                                    pattern.charAt(j) == '.'));

            if (j + 1 < pattern.length() && pattern.charAt(j+1) == '*'){
                ans = (match(i, j+2, text, pattern) ||
                       curMatch && match(i+1, j, text, pattern));
            } else {
                ans = curMatch && match(i+1, j+1, text, pattern);
            }
        }
        memo[i][j] = ans ? Result.TRUE : Result.FALSE;
        return ans;
    }
}
```

来跑一下结果：

{% asset_img result2.png result2 %}

击败了99.95%，不错不错。

已经很晚了，但我还是想把另一种方法也一起写完。🙄

还有一种方法，叫做自底向上方法，也是动态规划中的一种，这种方法的思路其实很简单粗暴，即从最后一个字符开始反向匹配，还是以刚才的栗子为例，从i = 3, j = 5 开始依次往左往上循环计算，match(3,5) == true，核心的逻辑并没有变。因为最边缘的值的匹配都是可以直接计算出来的，下面推算其中的一部分：

```java
1. match(3,5) = true;
2. 求match(3,4): i = 3; j = 4; curMatch = false;
3. j + 1 == 5 >= p.length() -> match(3,4) = curMatch = false;
4. match(3,4) = false;
5. 求match(3,3): i = 3; j = 3; curMatch = false;
6. p[4] == b -> match(3,3) = curMatch = false;
7. match(3,3) = false;
8. 求match(3,2): i = 3; j = 2; curMatch = false;
9. p[3] == * -> match(3,2) = match(3,4) || false && match(4,2)
10. match(3,2) = false;
11. 求match(3,1): i = 3; j = 1; curMatch = false;
12. p[2] == a -> match(3,1) = curMatch = false;
13. match(3,1) = false;
14. 求match(3,0): i = 3; j = 0; curMatch = false;
15. p[1] == * -> match(3,0) = match(3,2) || false && match(4,0)
16. match(3,0) = false;
17. ....
```

剩下的部分可以自行推导。代码如下：

```java 
class Solution {
    public boolean isMatch(String text, String pattern) {
        boolean[][] memo = new boolean[text.length() + 1][pattern.length() + 1];
        memo[text.length()][pattern.length()] = true;

        for (int i = text.length(); i >= 0; i--){
            for (int j = pattern.length() - 1; j >= 0; j--){
                boolean curMatch = (i < text.length() &&
                                       (pattern.charAt(j) == text.charAt(i) ||
                                        pattern.charAt(j) == '.'));
                if (j + 1 < pattern.length() && pattern.charAt(j+1) == '*'){
                    memo[i][j] = memo[i][j+2] || curMatch && memo[i+1][j];
                } else {
                    memo[i][j] = curMatch && memo[i+1][j+1];
                }
            }
        }
        return memo[0][0];
    }
}
```

提交一下：

{% asset_img result3.png result3 %}

效率也是相当高的，虽然比自顶向下方法多计算了不少值，但是减少了方法调用次数，省去了多次递归调用方法的开销，而且每次计算的过程相当简单，所以并不能说它的效率比自顶向下的方法低，要视具体情况而定。

## 总结

写到这，今天的题总算是完成的差不多了，长呼一口，来回顾一下今天的收获吧：

首先我们用分治法，使用递归来解决，但是效率偏低。

于是我们用了动态规划的思想来解决这个问题，与分治法最大的不同便在于动态规划会存储中间的计算状态，以减少重复计算。

先是用了自顶向下的方法，跟分治法几乎没有差异，只是多使用了一个二维数组。

接着用自底向上的方法来解决，从最后的字符开始匹配，将多次递归调用转为在一个循环体中完成。

总结一下动态规划的步骤：

1. 抽象问题。将问题分解为多个子问题，子问题的解一旦求出就会被保存。
2. 确定状态。确认我们要求解的子问题的状态空间，并设置初始状态。
3. 确定状态转移方程。这一步是最难也是最重要的一步。
