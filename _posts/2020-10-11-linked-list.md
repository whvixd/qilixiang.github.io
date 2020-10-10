---
layout:     post
title:      手写LinkedList
subtitle:   数据结构
date:       2020-10-11
author:     Static
header-img: 
catalog: true
tags:
    - 数据结构
    
---

# 手写LinkedList

> ***源码地址***:[传送门](https://github.com/whvixd/study/blob/master/common/src/main/java/com/github/whvixd/util/datastructure/LinkedList.java)

> ***单测代码***:[传送门](https://github.com/whvixd/study/blob/master/common/src/test/java/com/github/whvixd/util/datastructure/LinkedListTest.java)

## 1. What？

### 1. Definition

Java中LinkedList是双向链表,链表中的每个节点都包含了对前一个和后一个节点的引用。继承AbstractSequentialList，实现了List，Deque，Cloneable，java.io.Serializable这些接口。

### 2. Diagram

<html>
    <img src="/img/datastructure/LinkedListDiagram.png" width="500" height="500" /> 
</html>

> LinkedList中的操作是线程非安全的，所以在多线程中使用 `ConcurrentLinkedQueue` 或 `Collections.synchronizedList(new LinkedList<>());`

---

## 2. How？

> 双向链表:(head) <-pn-> (O) <-pn-> (O) <-pn-> (tail)

### 1. Node Constructor

```java
private static class Node<T>{
    T value;
    Node<T> next;
    Node<T> prev;
    public Node(T value, Node<T> next, Node<T> prev){
        this.value = value;
        this.next=next;
        this.prev=prev;
    }
    public Node(T value){
        this.value = value;
    }
    public Node(){}

    public T getValue() {
        return value;
    }

    @Override
    public String toString() {
        return "Node.value="+String.valueOf(value);
    }
}
```

### 2. Constructor

> head、tail是空值指针，在初始化时赋值。

```java
public class LinkedList<T> implements Queue<T>,Cloneable,java.io.Serializable{
    // 头节点
    private transient Node<T> head;
    // 尾节点
    private transient Node<T> tail;
    // 长度
    private transient int size;
    
    public LinkedList(){
        init(this);
    }
        
    private void init(LinkedList list){
        list.head = new Node<>();
        list.tail = new Node<>();
        list.head.next=list.tail;
        list.tail.prev=list.head;
    }
}
    
```


### 3. Methods

#### 1. 查

```
    public T get(int index){
        Node<T> node=getNode(index);
        return node==null?null:node.getValue();
    }

    private Node<T> getNode(int index){
        if(index<0||index>size)throw new Error("index not greater size!");
        Node<T> point=this.head;
        for(int i=0;point.next!=null;i++){
            if(i==index){
                return point.next;
            }else {
                point=point.next;
            }
        }
        return null;
    }

    private Node<T> getElementNode(Object o){
        Node<T> point=this.head.next;
        while (point!=null){
            if(o.equals(point.value)){
                return point;
            }else {
                point=point.next;
            }
        }
        return null;
    }

```

#### 2. 增

```
    // 头插
    public boolean addFirst(T o){
        Node<T> newNode = new Node<>(o);
        if(head.next!=null){
            newNode.next=head.next;
            newNode.prev=head;
            head.next.prev=newNode;
            head.next=newNode;
        }else {
            head.next=newNode;
            newNode.prev=head;
        }
        size++;
        return true;
    }

    // 尾插
    public boolean addLast(T o){
        Node<T> newNode = new Node<>(o);
        if(tail.prev!=null){
            newNode.next=tail;
            newNode.prev=tail.prev;
            tail.prev.next=newNode;
            tail.prev=newNode;
        }else {
            tail.prev=newNode;
            newNode.next=newNode;
        }
        size++;
        return true;
    }
```

#### 3. 删

```
    // 头删
    public T removeFirst(){
        Node<T> node = removedNodeFirst();
        return node==null?null:node.getValue();
    }

    private Node<T> removedNodeFirst(){
        if(size>0){
            Node<T> deleteNode = head.next;
            Node<T> pointNode = deleteNode.next;
            head.next=pointNode;
            pointNode.prev=head;
            size--;
            return deleteNode;
        }
        return null;
    }

    // 尾删
    public T removeLast(){
        Node<T> node = removedNodeLast();
        return node==null?null:node.getValue();
    }

    private Node<T> removedNodeLast(){
        if(size>0){
            Node<T> deleteNode = tail.prev;
            Node<T> pointNode = deleteNode.prev;
            pointNode.next=tail;
            tail.prev=pointNode;
            size--;
            return deleteNode;
        }
        return null;
    }

    // 根据值删除节点
    public boolean removeElement(Object o){
        Node<T> node = getElementNode(o);
        if(node==null){
            return false;
        }else {
            Node<T> prev=node.prev;
            Node<T> next=node.next;
            if(next==null){
                prev.next=null;
            }else {
                prev.next=next;
                next.prev=prev;
            }
            size--;
            return true;
        }
    }

    // 删除具体索引的节点
    public Object remove(int index){
        if(index<0||index>size-1)throw new Error("index not greater size!");
        Node<T> point=this.head;
       for(int i=0;point.next!=null;i++){
           if(i==index){
               Node<T> removed=point.next;
               Node<T> next=point.next.next;
               if(next!=null){
                   point.next=next;
                   next.prev=point;
               }else {
                   point.next=null;
               }
               size--;
               return removed.value;
           }else {
               point=point.next;
           }
       }
       return null;
    }

```

#### 4. 改

```
    // 改
    public boolean set(int index,T value){
        Node<T> node=getNode(index);
        if(node==null){
            return false;
        }else {
            node.value=value;
            return true;
        }
    }
```

#### 5. 其他

```
    // 是否包含
    public boolean contains(T value){
        Node<T> node = getElementNode(value);
        return node!= null;
    }
    
    // 大小
    public int size(){
        return this.size;
    }

    // 清除链表
    public void clear(){
        for(Node node=this.head;node!=null;){
            Node point=node.next;
            node.prev=null;
            node.next=null;
            node.value=null;
            node=point;
        }
        head=null;
        tail=null;
        this.size=0;
    }

    // 深度克隆
    @SuppressWarnings("unchecked")
    public LinkedList<T> clone(){
        LinkedList<T> clone;
        try {
           clone= (LinkedList<T>) super.clone();
        } catch (CloneNotSupportedException e) {
            throw new Error(e);
        }
        clone.init(clone);
        for(Node<T> point=this.head.next;point!=tail&&point!=null;point=point.next){
            clone.addLast(point.value);
        }
        return clone;
    }

    // 实现 Queue
    // 入队
    @Override
    public boolean add(T value){
        return addFirst(value);
    }

    // 出队
    @Override
    public T poll(){
        return removeLast();
    }

    // 打印 节点
    public String print(Node<T> node){
        Node<T> pointNode = node==null?head.next:node;
        StringBuilder sb=new StringBuilder();
        while (pointNode!=tail&&pointNode!=null){
            sb.append(pointNode.value);
            System.out.print(String.valueOf(pointNode.value));
            if(pointNode.next!=tail){
                sb.append("->");
                System.out.print("->");
            }
            pointNode=pointNode.next;
        }
        System.out.println();
        return sb.toString();
    }

    public String print(){
        return print(null);
    }
```

---

> ***源码地址***:[传送门](https://github.com/whvixd/study/blob/master/common/src/main/java/com/github/whvixd/util/datastructure/LinkedList.java)

> ***单测代码***:[传送门](https://github.com/whvixd/study/blob/master/common/src/test/java/com/github/whvixd/util/datastructure/LinkedListTest.java)