---
title: 【LeetCode】两数相加
tags: 
 - Java
 - 算法
 - LeetCode
categories: 编程
date: 2019-01-06 20:00:00
---

## 题目描述

给出两个`非空`的链表用来表示两个非负的整数。其中，它们各自的位数是按照`逆序`的方式存储的，并且它们的每个节点只能存储`一位`数字。

如果，我们将这两个数相加起来，则会返回一个新的链表来表示它们的和。

您可以假设除了数字`0`之外，这两个数都不会以`0`开头。

示例：

```bash
输入：(2 -> 4 -> 3) + (5 -> 6 -> 4)
输出：7 -> 0 -> 8
原因：342 + 465 = 807
```

## 题目解析

这个题目的意思看起来其实很简单，提供了两个链表，每个链表代表一个非负整数，它们各自的位数是按照`逆序`方式存储的，例如：`(2 -> 4 -> 3)`代表整数`342`，`(5 -> 6 -> 4)`则代表整数`465`，两数相加的结果自然是807，这就是我们要给出的答案，但是要用链表的形式返回`7 -> 0 -> 8`。题目中说明了是非空链表，所以就不用考虑链表为null的情况了。

乍眼一看，很简单啊，不就是把两个数相加嘛，我先把它整成整数，然后相加，最后把结果整成链表，完美，哈哈哈哈，简直被自己的聪明才智给折服。

### 翻车尝试1：虾扯蛋法

```java
class Solution {
    public ListNode addTwoNumbers(ListNode l1, ListNode l2) {
        ListNode head1 = l1;
        ListNode head2 = l2;
        int i = 0;
        int j = 0;
        int index = 0;
        // 将链表l1转化为整数
        while (head1 != null) {
            i += head1.val * Math.pow(10, index);
            index++;
            head1 = head1.next;
        }
        index = 0;
        // 将链表l2转化为整数
        while (head2 != null) {
            j += head2.val * Math.pow(10, index);
            index++;
            head2 = head2.next;
        }
        int sum = i + j;
        ListNode newHead = new ListNode(0);
        ListNode tmpHead = newHead;
        int sign = 0;
        // 将结果转化为链表
        while (sum > 0 || sign == 0) {
            int tmp = sum % 10;
            sum = sum / 10;
            tmpHead.next = new ListNode(tmp);
            tmpHead = tmpHead.next;
            sign++;
        }
        return newHead.next;
    }
}
```

简直轻松加愉快，让我们来提交一下。

{% asset_img solution-1.png solution-1 %}

![](/images/black_smile.png)

怎么肥四，小老弟，翻车了啊。让我们看看错误原因：

```bash
输入：
[9]
[1,9,9,9,9,9,9,9,9,9]
输出：
[0]
预期：
[0,0,0,0,0,0,0,0,0,0,1]
```

看样子应该是整数型溢出了。。。难不倒我，改成long型不就完事了。

### 翻车尝试2：虾扯蛋升级法

```java
class Solution {
    public ListNode addTwoNumbers(ListNode l1, ListNode l2) {
        ListNode head1 = l1;
        ListNode head2 = l2;
        long i = 0;
        long j = 0;
        long index = 0;
        while (head1 != null) {
            i += head1.val * Math.pow(10, index);
            index++;
            head1 = head1.next;
        }
        index = 0;
        while (head2 != null) {
            j += head2.val * Math.pow(10, index);
            index++;
            head2 = head2.next;
        }
        long sum = i + j;
        ListNode newHead = new ListNode(0);
        ListNode tmpHead = newHead;
        int sign = 0;
        while (sum > 0 || sign == 0) {
            int tmp = (int)(sum % 10);
            sum = sum / 10;
            tmpHead.next = new ListNode(tmp);
            tmpHead = tmpHead.next;
            sign++;
        }
        return newHead.next;
    }
}
```
这次总没事了吧，再提交一下：

{% asset_img solution-2.png solution-2 %}

![](/images/black_smile-2.png)

这个磨人的小妖精，整出个这么大的数来折腾我，long型也溢出了。。。

逼我用绝招，是时候祭出我的`BigInteger`了。

### 翻车尝试3：虾扯蛋终极法

```java
class Solution {
    public ListNode addTwoNumbers(ListNode l1, ListNode l2) {
        ListNode head1 = l1;
        ListNode head2 = l2;
        BigInteger i = new BigInteger(0);
        BigInteger j = new BigInteger(0);
        long index = 0;
        while (head1 != null) {
            i.add(BigInteger.valueOf(head1.val * Math.pow(10, index)));
            index++;
            head1 = head1.next;
        }
        index = 0;
        while (head2 != null) {
            j.add(BigInteger.valueOf(head2.val * Math.pow(10, index)));
            index++;
            head2 = head2.next;
        }
        BigInteger sum = i.add(j);
        ListNode newHead = new ListNode(0);
        ListNode tmpHead = newHead;
        int sign = 0;
        while (sum.compareTo(0) == 1 || sign == 0) {
            int tmp = sum.mod(10).intValue();
            sum = sum.divide(10);
            tmpHead.next = new ListNode(tmp);
            tmpHead = tmpHead.next;
            sign++;
        }
        return newHead.next;
    }
}
```

这次，连编译都不通过了，emmmm，看来不准用`BigInteger`这个类。

{% asset_img solution-2.png solution-2 %}

## 常规解法

既然邪门歪道走不通，那就还是用常规操作来解决吧，仔细想想，其实也很简单，我们从两个链表的头节点开始，一起遍历，将相加得到的结果存入新的链表中即可。

{% asset_img solution-4.png solution-4 %}

这里需要注意的就是要考虑进位的情况，比如：`4 + 6 = 10`，那么在处理后一个节点`3 + 4`的时候，需要再加1，因此需要有一个进位标志来表示是否需要进位。

另外，两个链表的长度并不一定相等，需要考虑像上面那样一个很长，一个很短，而且后续一直进位的情况：

```bash
[9]
[1,9,9,9,9,9,9,9,9,9]
```

所以我们可以定义一个叫`carry`的变量来表示是否需要进位。

```java
class Solution {
    public ListNode addTwoNumbers(ListNode l1, ListNode l2) {
        ListNode head1 = l1;
        ListNode head2 = l2;
        ListNode newHead = new ListNode(0);
        ListNode head3 = newHead;
        // 进位标志
        boolean carry = false;
        while (head1 != null || head2 != null) {
            // 获取对应位置的值然后相加
            int x = (head1 != null) ? head1.val : 0;
            int y = (head2 != null) ? head2.val : 0;
            int sum = carry ? (x + y + 1) : (x + y);
            // 处理进位
            if (sum >= 10){
                sum -= 10;
                carry = true;
            } else {
                carry = false;
            }
            // 新增节点
            head3.next = new ListNode(sum % 10);
            head3 = head3.next;
            if (head1 != null) head1 = head1.next;
            if (head2 != null) head2 = head2.next;
        }
        if (carry) {
            head3.next = new ListNode(1);
        }
        return newHead.next;
    }
}
```

{% asset_img result-5.png result-5 %}

嗯，这下就没什么问题了。😜

[本题中文版链接](https://leetcode-cn.com/problems/add-two-numbers/)

[本题英文版链接](https://leetcode.com/problems/add-two-numbers/)

如果你有更好的解法，欢迎留言讨论~