---
layout:     post
title:      leetcode offer 09
subtitle:   用两个栈实现队列
date:       2020-06-30
author:     Static
header-img: 
catalog: true
tags:
    - leetcode之每日一题
    
---

> 题目链接:[https://leetcode-cn.com/problems/yong-liang-ge-zhan-shi-xian-dui-lie-lcof/](https://leetcode-cn.com/problems/yong-liang-ge-zhan-shi-xian-dui-lie-lcof/)

## 题目描述

<html>
    <img src="/img/leetcode/leetcode-offer-09.png" width="700" height="700" /> 
</html>

---

## 分析

#### 1. 利用栈的 `FILO` 特点

> 上代码

---

## 代码实现

```java
public enum QOffer09 {

    instance;

    static class CQueue {

        private Stack<Integer> inStack = new Stack<>();
        private Stack<Integer> outStack = new Stack<>();

        public CQueue() {

        }

        public void appendTail(int value) {
            // 若out不为空，则全部进in栈中，老的放栈底
            while (!outStack.empty()) {
                inStack.push(outStack.pop());
            }
            // 新的放栈顶
            inStack.push(value);
        }

        public int deleteHead() {
            // 若in栈不为空，则全部进入out栈中，老的在栈顶
            while (!inStack.empty()) {
                outStack.push(inStack.pop());
            }
            // out不为空，则从栈顶取数据
            if (!outStack.empty()) {
                return outStack.pop();
            }
            return -1;
        }
    }

    /**
     * Your CQueue object will be instantiated and called as such:
     * CQueue obj = new CQueue();
     * obj.appendTail(value);
     * int param_2 = obj.deleteHead();
     */

    public static void main(String[] args) {
        CQueue obj = new CQueue();
        obj.appendTail(3);
        // assert 3
        System.out.println(obj.deleteHead());
        // assert -1
        System.out.println(obj.deleteHead());

        obj.appendTail(5);
        obj.appendTail(2);
        // assert 5
        System.out.println(obj.deleteHead());
        // assert 2
        System.out.println(obj.deleteHead());
        // assert -1
        System.out.println(obj.deleteHead());

        obj.appendTail(1);
        obj.appendTail(2);
        obj.appendTail(3);
        // assert 1
        System.out.println(obj.deleteHead());
        // assert 2
        System.out.println(obj.deleteHead());

        obj.appendTail(4);
        // assert 3
        System.out.println(obj.deleteHead());

        // assert 4
        System.out.println(obj.deleteHead());

    }
}
```

---

**Greed makes people stop at nothing. (Italy) Dante**

> 贪欲使人无所不为  -- \[意大利] 但丁