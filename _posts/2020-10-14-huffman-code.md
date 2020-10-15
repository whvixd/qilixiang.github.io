---
layout:     post
title:      哈夫曼编码
subtitle:   数据结构
date:       2020-10-14
author:     Static
header-img: 
catalog: true
tags:
    - 数据结构
    
---

## 1. What

### 1. Definition

赫夫曼树（Huffman Tree），又叫作最优二叉树，它的特点是带权路径最短。
1. 路径：从树中一个结点到另一个结点的分支所构成的路线。
2. 路径长度：路径上的分支数目。
3. 树的路径长度：从根到每个结点的路径长度之和。
4. 带权路径长度：结点具有权值，从该结点到根之间的路径乘以结点的权值。
5. 树的带权路径长度(WPL)：树中所有叶子结点的带权路径长度之和。

### 2. 赫夫曼树的构造方法

比如现在有 \[a(5),b(7),c(2),d(4),e(12)] 只有根结点的二叉树。

<html>
    <img src="/img/huffmantree/huffman_1.png" width="400" height="200" /> 
</html>

1, 选出权值最小的两个根 c、d 将它们作为左 右子树，构建成一个新的二叉树，新的二叉树的根结点权值为c d权值之和，同时新的二叉树根结点加入到构建的集合中 \[a(5),b(7),cd(6),e(12)]，如下图

<html>
    <img src="/img/huffmantree/huffman_2.png" width="400" height="400" /> 
</html>

2, 继续选择权值最小的两个根结点，这时候我们选到了 a、cd ，构建成新的二叉树，新结点权值5+6=11，集合:\[acd(11),b(7),e(12)]

<html>
    <img src="/img/huffmantree/huffman_3.png" width="400" height="400" /> 
</html>

3, 接着选择权值最小的两个根，这时候我们选到了 acd、b ，构建成新的二叉树，新结点权值11+7=18，集合:\[acdb(18),e(12)]

<html>
    <img src="/img/huffmantree/huffman_4.png" width="400" height="400" /> 
</html>

4, 最后就剩两个根结点，构建成二叉树，这就是赫夫曼树，计算 WPL=12\*1+7\*2+5*3+(2+4)*4=65;

<html>
    <img src="/img/huffmantree/huffman_5.png" width="400" height="400" /> 
</html>