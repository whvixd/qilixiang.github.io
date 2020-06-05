---
layout:     post
title:      leetcode interview 64
subtitle:   求1+2+…+n
date:       2020-06-02
author:     Static
header-img: 
catalog: true
tags:
    - leetcode之每日一题
    
---

> 题目链接:[https://leetcode-cn.com/problems/qiu-12n-lcof/](https://leetcode-cn.com/problems/qiu-12n-lcof/)

## 题目描述

<html>
    <img src="/img/leetcode/leetcode-interview-64.png" width="700" height="700" /> 
</html>

---

## 分析

#### 1. 等差数列求和

> f(x)=x,{x\| 1<=x<=1000}; sum={(1+f(x))*x}/2; 因不能用乘除,可以用 x>>1=x/2 , Math.pow(x,2)=x^2

#### 2. 递归

> 不能用条件语句，则可以用&& 符号的特性，A&&B 若A为false，则不执行B

---

## 代码实现

```java
public enum QInterview64 {
    instance;

    // 等差数列
    public int sumNums(int n) {
        return (int)(n+Math.pow(n,2))>>1;
    }

    // 递归
    public int sumNums1(int n) {
        int sum=n;
        boolean f=n>0&&(sum+=sumNums1(n-1))>0;
        return sum;
    }

    public static void main(String[] args) {
        // assert 45
        SystemUtil.print(QInterview64.instance.sumNums(9));
        // assert 45
        SystemUtil.print(QInterview64.instance.sumNums1(9));
    }

}
```