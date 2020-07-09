---
layout:     post
title:      leetcode interview 1713
subtitle:   恢复空格
date:       2020-07-09
author:     Static
header-img: 
catalog: true
tags:
    - leetcode之每日一题
    
---

> 题目链接:[https://leetcode-cn.com/problems/re-space-lcci/](https://leetcode-cn.com/problems/re-space-lcci/)

## 题目描述

<html>
    <img src="/img/leetcode/leetcode-interview-1713.png" width="700" height="700" /> 
</html>

---

## 分析

#### 1. 背包题

> @see 官网 -> [传送门](https://leetcode-cn.com/problems/re-space-lcci/solution/hui-fu-kong-ge-by-leetcode-solution/)

---

## 代码实现

```java
public enum  QInterview1713 {
    instance;

    public int respace(String[] dictionary, String sentence) {
        int n = sentence.length();
        int m = dictionary.length;
        int[] dp = new int[n + 1];
        for (int i = 1; i <= n; i++) {
            dp[i] = dp[i-1];
            for (int j = 0; j < m ; j++) {
                // 找到最短的一个单词
                if (i < dictionary[j].length()) continue;
                // 截取字符比较是否相等
                if (sentence.substring(i - dictionary[j].length(), i).equals(dictionary[j])) {
                    // 若有连续的单词，取最长的那个，为了使未识别的单词尽量少
                    dp[i] = Math.max(dp[i - dictionary[j].length()] + dictionary[j].length(), dp[i]);
                }
            }
        }
        return n - dp[n];
    }

    public static void main(String[] args) {
        // assert 7
        System.out.println(QInterview1713.instance.respace(new String[]{"looked","just","like","her","brother"},"jesslookedjustliketimherbrother"));
    }
}
```

---

**There are three key issues for reading: mind should concentrate, eyes should focus and mouth should read for specific purpose.  Zhu xi**

> 读书有三到，谓心到，眼到，口到。 -- 荀況