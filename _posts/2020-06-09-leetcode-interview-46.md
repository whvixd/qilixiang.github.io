---
layout:     post
title:      leetcode interview 46
subtitle:   把数字翻译成字符串
date:       2020-06-09
author:     Static
header-img: 
catalog: true
tags:
    - leetcode之每日一题
    
---
> 题目链接:[https://leetcode-cn.com/problems/ba-shu-zi-fan-yi-cheng-zi-fu-chuan-lcof/](https://leetcode-cn.com/problems/ba-shu-zi-fan-yi-cheng-zi-fu-chuan-lcof/)

## 题目描述

<html>
    <img src="/img/leetcode/leetcode-interview-46.png" width="700" height="700" /> 
</html>

---

## 分析

#### 1. 递归

> 见代码

---

## 代码实现

```java
public enum QInterview46 {
    instance;

    public int translateNum(int num) {
        if(num<10)return 1;
        int r=num%100;
        // 比如 r=9，9/10=0 <10，下次递归时，返回 1；r=26一样，26/10=2 <10 then 返回1
        if(r<10||r>25){
            return translateNum(num/10);
            // 比如 r=21，按条件可再分为2和1，也可是一个整体21
        }else {
            return translateNum(num/10)+translateNum(num/100);
        }
    }

    public static void main(String[] args) {
        // assert 5
        System.out.println(QInterview46.instance.translateNum(12258));
    }

}
```