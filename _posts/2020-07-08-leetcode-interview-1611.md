---
layout:     post
title:      leetcode interview 1611
subtitle:   跳水板
date:       2020-07-08
author:     Static
header-img: 
catalog: true
tags:
    - leetcode之每日一题
    
---

> 题目链接:[https://leetcode-cn.com/problems/diving-board-lcci/](https://leetcode-cn.com/problems/diving-board-lcci/)

## 题目描述

<html>
    <img src="/img/leetcode/leetcode-interview-1611.png" width="700" height="700" /> 
</html>

---

## 分析

#### 1. shorter * (k - i) + longer * i

---

## 代码实现

```java
public enum QInterview1611 {
    instance;
    
    public int[] divingBoard(int shorter, int longer, int k) {
        if(k == 0) return new int[]{};
        if(shorter == longer) return new int[]{shorter * k};
        int[] results = new int[k + 1];
        for (int i = 0; i <= k; i++) {
            // 短的和长的组合
            results[i] = shorter * (k - i) + longer * i;
        }
        return results;
    }
    
    public int[] divingBoard1(int shorter, int longer, int k) {
        if(k<1)return new int[0];
        if(shorter == longer) return new int[]{shorter * k};
        int[] result=new int[k+1];
        for(int i=0;i<=k;i++){
            int sum=0;
            for(int j=i;j<k;j++){
                sum+=shorter;
            }
            for(int z=0;z<i;z++){
                sum+=longer;
            }
            result[i]=sum;
        }
        return result;
    }

    public static void main(String[] args) {
        //[3, 4, 5, 6]
        SystemUtil.print(QInterview1611.instance.divingBoard(1,2,3));
        // [110,120,130,140,150,160,170,180,190,200,210,220]
        SystemUtil.print(QInterview1611.instance.divingBoard(10,20,11));
        SystemUtil.print(QInterview1611.instance.divingBoard1(10,20,11));
        // []
        SystemUtil.print(QInterview1611.instance.divingBoard(1,1,0));
    }

}
```

---

**It is impossible to realize the persistence of pine tree if it has not come through the bitter winter.It is impossible to realize the noble character of a man if he has not overcome the adversities.  Xun Kuang**

> 岁不寒，无以知松柏；事不难，无以知君子。 -- 荀況