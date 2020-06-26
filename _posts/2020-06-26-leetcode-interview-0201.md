---
layout:     post
title:      leetcode interview 0201
subtitle:   移除重复节点
date:       2020-06-26
author:     Static
header-img: 
catalog: true
tags:
    - leetcode之每日一题
    
---

> 题目链接:[https://leetcode-cn.com/problems/remove-duplicate-node-lcci/](https://leetcode-cn.com/problems/remove-duplicate-node-lcci/)

## 题目描述

<html>
    <img src="/img/leetcode/leetcode-interview-0201.png" width="700" height="700" /> 
</html>

---

## 分析

#### 1. 双重循环

---

## 代码实现

```java
public enum QInterview0201 {
    instance;
    
    public ListNode removeDuplicateNodes(ListNode head) {
        ListNode p=head;
        while (p!=null){
            ListNode r=p;
            while (r.next!=null){
                if(p.val==r.next.val){
                    // 删除重复节点
                    r.next=r.next.next;
                    continue;
                }
                r=r.next;
            }
            p=p.next;
        }
        return head;
    }

    public static void main(String[] args) {
        ListNode node1 = ListNode.createByArray(new int[]{1, 2, 3, 3, 2, 1});
        QInterview0201.instance.removeDuplicateNodes(node1);
        // assert [1, 2, 3]
        ListNode.print(node1);

        ListNode node2 = ListNode.createByArray(new int[]{1, 1, 1, 1, 2});
        QInterview0201.instance.removeDuplicateNodes(node2);
        // assert [1, 2]
        ListNode.print(node2);
    }

}
```
> **ListNode.java**

```java
public class ListNode {
    public  int val;
    public ListNode next;

    public ListNode(){}

    public ListNode(int x) {
        val = x;
    }

    public static void print(ListNode l, String preMessage, String postMessage) {
        preMessage=preMessage==null?"":preMessage;
        postMessage=postMessage==null?"":postMessage;
        ListNode head = l;
        StringBuilder sb = new StringBuilder();
        sb.append(preMessage);
        while (head != null) {
            sb.append(head.val);
            head = head.next;
            if (head != null) {
                sb.append("->");
            }
        }
        sb.append(postMessage);
        System.out.println(sb);
    }

    public static void print(ListNode l) {
        print(l, null, null);
    }

    public static void print(String preMessage, ListNode l) {
        print(l, preMessage, null);
    }

    public static void print(ListNode l, String postMessage) {
        print(l, null, postMessage);
    }

    public static ListNode createByArray(int[] array){
        if(array.length==0) return null;
        ListNode head =new ListNode(0);
        ListNode point=head;
        for(int e:array){
            point=point.next=new ListNode(e);
        }
        return head.next;
    }
}
```

**If you want to live in happiness,you should love your life. (Britain)Johnson**

> 要想生活快乐，就必须热爱生活。-- \[英] 约翰逊