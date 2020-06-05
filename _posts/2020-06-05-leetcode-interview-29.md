---
layout:     post
title:      leetcode 29
subtitle:   顺时针打印矩阵
date:       2020-06-05
author:     Static
header-img: 
catalog: true
tags:
    - leetcode之每日一题
    
---
> 题目链接:[https://leetcode-cn.com/problems/shun-shi-zhen-da-yin-ju-zhen-lcof/](https://leetcode-cn.com/problems/shun-shi-zhen-da-yin-ju-zhen-lcof/)

## 题目描述

<html>
    <img src="/img/leetcode/leetcode-interview-29.png" width="700" height="700" /> 
</html>

---

## 分析

#### 1. 官网解析

> 链接 -> [https://leetcode-cn.com/problems/shun-shi-zhen-da-yin-ju-zhen-lcof/solution/shun-shi-zhen-da-yin-ju-zhen-by-leetcode-solution/](https://leetcode-cn.com/problems/shun-shi-zhen-da-yin-ju-zhen-lcof/solution/shun-shi-zhen-da-yin-ju-zhen-by-leetcode-solution/)

---

## 代码实现

```java
public enum QInterview29 {
    instance;

    public int[] spiralOrder(int[][] matrix) {

        if (matrix == null || matrix.length == 0 || matrix[0].length == 0) {
            return new int[0];
        }
        int rows = matrix.length, columns = matrix[0].length;
        int[] order = new int[rows * columns];
        int index = 0;
        int left = 0, right = columns - 1, top = 0, bottom = rows - 1;
        while (left <= right && top <= bottom) {
            for (int column = left; column <= right; column++) {
                order[index++] = matrix[top][column];
            }
            for (int row = top + 1; row <= bottom; row++) {
                order[index++] = matrix[row][right];
            }
            if (left < right && top < bottom) {
                for (int column = right - 1; column > left; column--) {
                    order[index++] = matrix[bottom][column];
                }
                for (int row = bottom; row > top; row--) {
                    order[index++] = matrix[row][left];
                }
            }
            left++;
            right--;
            top++;
            bottom--;
        }
        return order;

    }

    public static void main(String[] args) {
        // assert [1, 2, 3, 6, 9, 8, 7, 4, 5]
        int[][] matrix = {{1, 2, 3}, {4, 5, 6}, {7, 8, 9}};
        SystemUtil.print(QInterview29.instance.spiralOrder(matrix));
    }
}
```