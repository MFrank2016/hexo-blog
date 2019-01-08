---
title: 【LeetCode】无重复字符串最长子串
tags: 
 - Java
 - 算法
 - LeetCode
categories: 编程
date: 2019-01-08 20:00:00
---

## 题目描述

给定一个字符串，请你找出其中不含有重复字符的 最长子串 的长度。

示例 1:

```bash
输入: "abcabcbb"
输出: 3 
解释: 因为无重复字符的最长子串是 "abc"，所以其长度为 3。
```

示例 2:

```bash
输入: "bbbbb"
输出: 1
解释: 因为无重复字符的最长子串是 "b"，所以其长度为 1。
```

示例 3:

```bash
输入: "pwwkew"
输出: 3
解释: 因为无重复字符的最长子串是 "wke"，所以其长度为 3。
     请注意，你的答案必须是 子串 的长度，"pwke" 是一个子序列，不是子串。
```

## 题目解析

这道题的目标是找出最长子串，并且该子串必须不包含重复字符，而且这个子串必须是原字符串中连续的一部分（见示例3中的解释说明）。

拿到题目时先不要心急想什么骚操作，我们先从最普通的操作开始把题目解出来，然后再来看如何优化。

接下来，我们画图分析一下，先随便弄一个长相普通的字符串：`frankissohandsome`，我们要从中找出我们想要的子串，那少不了需要遍历，我们设置两个变量`from`，`to`，分别存储寻找的目标子串在原字符串中的首尾位置。

首先，`from`和`to`的初始值都为0（String的序号从0开始），子串长度`length = 1`，最大子串长度`maxLength = 1`。

{% asset_img day3-step-1.png day3-step-1 %}

然后，我们将`to`的指向往后移动，并判断新遍历的字符是否已经存在于子串中，如果不存在，则将其加入子串中，并将`length`进行自增。

{% asset_img day3-step-2.png day3-step-2 %}

直到找到一个已存在于子串中的字符，或者`to`到达字符串的末尾。这里，我们找到了一个重复的`s`，序号为`7`，此时的子串为`frankis`，将此时的子串长度与最大子串长度相比较（目前为`0`），如果比最大子串长度大，则将最大子串长度设置为当前子串长度`7`。

{% asset_img day3-step-3.png day3-step-3 %}

接下来，我们继续寻找符合条件的子串，这里比较关键的一点是下一个子串的起始位置，这里我们将`from`直接跳到了序号为`7`的位置，因为包含`ss`的子串显然都不能满足要求。

{% asset_img day3-step-4.png day3-step-4 %}

然后我们依照之前的方法，找到第二个候选的子串`sohand`，长度为`6`，比目前的最大子串长度小，所以不是目标子串。

{% asset_img day3-step-5.png day3-step-5 %}

接着继续寻找，找到另一个候选子串`ohands`，长度小于最大子串长度，不是我们的目标子串。

{% asset_img day3-step-6.png day3-step-6 %}

继续寻找。

{% asset_img day3-step-7.png day3-step-7 %}

`to`到达了字符串末尾，找到另一个候选子串`handsome`，长度大于最大子串长度，这就是我们的目标子串。

{% asset_img day3-step-8.png day3-step-8 %}

于是我们的最大子串长度就轻松加愉快的找到了。接下来的事情就是把上面的思路转化成代码。

这里只需要注意一下`from`的跳转即可，每次跳转的序号为`to`指向的字符在子串中出现的位置 + 1。

## 常规解法

```java
class Solution {
    public int lengthOfLongestSubstring(String s) {
        if (s == null || s.length() == 0) return 0;
        int from = 0, to = 1, length = 1, maxLength = 1;
        // to遍历直到字符串末尾
        while (to < s.length()){
            int site = s.substring(from, to).indexOf(s.charAt(to));
            if (site != -1){
                // to指向的字符已存在
                length = to - from;
                if (length > maxLength) maxLength = length;
                // from 跳转到site+1的位置
                from = from + site + 1;
            }
            to++;
        }
        // 处理最后一个子串
        if (to - from > maxLength) {
            maxLength = to - from;
        }
        return maxLength;
    }
}
```

这里没有什么骚操作，考虑好边界情况就行了。有一个小细节需要注意，`site`代表的是子串中字符出现的位置，不是原字符串中的位置，因此`from`在跳转时，需要加上自身原来的序号。还有最后一个子串的处理不要忘记，因为当`to`遍历到字符串末尾时，会结束循环，最后一个子串将不会在循环内处理。

让我们提交一下：

{% asset_img day3-submit-1.png day3-submit-1 %}

击败了`73%`的用户，还不错。

## 常规解法优化

想想看，还有没有优化的空间呢？

那肯定是有的，首先我们想一想，当我们找到的最大子串长度已经比`from`所在位置到字符串末尾的位置还要长了，那就没有必要再继续下去了。

```java
class Solution {
    public int lengthOfLongestSubstring(String s) {
        if (s == null || s.length() == 0) return 0;
        int from = 0, to = 1, length = 1, maxLength = 0;
        // to遍历直到字符串末尾
        while (to < s.length()){
            int site = s.substring(from, to).indexOf(s.charAt(to));
            if (site != -1){
                // to指向的字符已存在
                length = to - from;
                if (length > maxLength) {
                    maxLength = length;
                }
                // 判断是否需要继续遍历
                if (maxLength > s.length() - from + 1) return maxLength;
                from = from + site + 1;
            }
            to++;
        }
        // 处理最后一个子串
        if (to - from > maxLength) {
            maxLength = to - from;
        }
        return maxLength;
    }
}
```

另外要处理类似`bbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbb`这样的字符串，上面的方法还是有很大优化空间的，我们可以用一个`HashSet`来存储所有元素，利用其特性进行去重，如果找到的子串长度已经等于`HashSet`中的元素个数了，那就不用再继续查找了。

```java
class Solution {
    public int lengthOfLongestSubstring(String s) {
        if (s == null || s.length() == 0) return 0;
        int from = 0, to = 1, length = 1, maxLength = 0;
        Set<Character> set = new HashSet<>();
        for (int i = 0; i < s.length(); i++){
            set.add(s.charAt(i));
        }
        // to遍历直到字符串末尾
        while (to < s.length()){
            int site = s.substring(from, to).indexOf(s.charAt(to));
            if (site != -1){
                // to指向的字符已存在
                length = to - from;
                if (length > maxLength) {
                    maxLength = length;
                }
                if (maxLength > s.length() - from + 1) return maxLength;
                if (maxLength >= set.size()) return maxLength;
                from = from + site + 1;
            }
            to++;
        }
        // 处理最后一个子串
        if (to - from > maxLength) {
            maxLength = to - from;
        }
        return maxLength;
    }
}
```

再提交一下：

{% asset_img day3-submit-2.png day3-submit-2 %}

哈哈哈哈，翻车了，所以这里引入一个`HashSet`用空间来换时间的方式不一定合适，看来测试用例里像`bbbbbbbbbbbbbb`这样的用例并不多啊。

那么今天的翻车就到此为止了，如果觉得对你有帮助的话记得点个关注哦。
