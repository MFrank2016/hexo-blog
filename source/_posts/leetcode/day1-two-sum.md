---
title: 【LeetCode】两数之和
tags: 
 - Java
 - 算法
 - LeetCode
categories: 编程
date: 2019-01-05 01:00:00
---

## 题目说明

给定一个整数数组 nums 和一个目标值 target，请你在该数组中找出和为目标值的那两个整数，并返回他们的数组下标。

你可以假设每种输入只会对应一个答案。但是，你不能重复利用这个数组中同样的元素。

示例:

```bash
给定 nums = [2, 7, 11, 15], target = 9

因为 nums[0] + nums[1] = 2 + 7 = 9
所以返回 [0, 1]
```

## 解题思路1：穷举法

从题目意思理解，就是从给定的整数数组中找到两个整数，使得它们的和与给定的数相等。那最简单粗暴的方式就是枚举了，嗯，先来试试最简单的。

```java
class Solution {
    public int[] twoSum(int[] nums, int target) {
        return exhaustAlgorithm(nums,target);
    }
    // 穷举法
    private int[] exhaustAlgorithm(int[] nums, int target){
        int length = nums.length;
        int i = 0;
        int j = 1;
        while (nums[i] + nums[j] != target) {
            j++;
            if (j >= length){
                i++;
                if (i >= length - 1){
                    break;
                }
                j = i + 1;
            }
        }
        // 说明不存在这样的组合
        if (nums[i] + nums[j] != target) return null;
        int[] result = {i,j};
        return result;
    }
}
```

时间复杂度：$O(n^2)$

运行结果如下：

{% asset_img exhaust-algorithm.png exhaust-algorithm %}

80ms，才击败了11.13%的用户，说明优化空间还很大。

## 解题思路2：倒推法

穷举法的效率一般都比较差，所以需要尝试一些新姿势。我们再来分析一下上面的穷举算法，要从一个集合中找出两个数，使得它们的和与给出的数`target`相等，使用穷举算法时，当我们选出第一个数`a`后，需要循环遍历之后的数，然后一一进行加和判断，但实际上，我们只需要知道剩下的数里，有没有数等于`target - a`即可，而每次从数组中找到某个数是否存在，都需要遍历一次，因此，更好的做法是将数与对应的序号存到一个map中，这样就能将查找效率从$O(n)$提高到$O(1)$。

```java
class Solution {
    public int[] twoSum(int[] nums, int target) {
        return mapSolution(nums,target);
    }
    // 倒推法
    private int[] mapSolution(int[] nums, int target){
        Map<Integer, Integer> map = new HashMap<>();
        for (int i = 0; i < nums.length; i++){
            map.put(nums[i],i);
        }

        for (int i = 0; i < nums.length; i++){
            int num = target - nums[i];
            // 判断num是否存在，如果已经存在，则直接返回
            if (map.get(num) != null){
                return new int[] { map.get(num), i};
            }
        }
        return null;
    }
}
```

这里我们对nums数组进行了两次遍历，第一次遍历是将所有元素都存入map中，第二次遍历是查找目标的整数对是否存在。

但再仔细想想，是否还能再优化呢？

答案是肯定的，在这个题中，要寻找的整数是成对存在的，所以我们可以只进行一次遍历。

如果`target`减去当前遍历数值后的数不存在于`map`中，则将当前数值与序号的映射关系存入`map`中。也许你会问，那找到第一个要寻找的数时，第二个数显然还不在`map`中，那怎么办呢？别着急，前面已经说过了，因为要寻找的数是成对存在的，这里我们假设为`a`和`b`，所以遇到第一个数`a`时，由于`b`还没有存入`map`，所以先将`a`存入`map`中，我们在找到第二个数`b`后，此时`a`已经在`map`中了，所以就能在一次遍历中顺利找到了这对我们想要的整数了。

```java
class Solution {
    public int[] twoSum(int[] nums, int target) {
        return mapSolution(nums,target);
    }
    // 倒推法
    private int[] mapSolution(int[] nums, int target){
        Map<Integer, Integer> map = new HashMap<>();
        for (int i = 0; i < nums.length; i++){
            int num = target - nums[i];
            // 判断num是否存在，如果已经存在，则直接返回
            if (map.get(num) != null){
                return new int[] { map.get(num), i};
            }
            // 不存在则当前数值与序号的映射关系存入map中
            map.put(nums[i], i);
        }
        return null;
    }
}
```

时间复杂度：$O(n)$

空间复杂度：$O(n)$

运行结果如下：

{% asset_img map-solution.png map-solution %}

一下降到了9ms，效率大大提升，击败85%的用户，嗯，看来效果确实很显著。

本题链接：https://leetcode-cn.com/problems/two-sum/solution/

如果你有更好的想法，也欢迎留言交流讨论~