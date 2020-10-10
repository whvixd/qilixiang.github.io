---
layout:     post
title:      手写ArrayList
subtitle:   数据结构
date:       2020-10-10
author:     Static
header-img: 
catalog: true
tags:
    - 数据结构
    
---

# 手写ArrayList

> ***源码地址***:[传送门](https://github.com/whvixd/study/blob/master/common/src/main/java/com/github/whvixd/util/datastructure/ArrayList.java)

> ***单测代码***:[传送门](https://github.com/whvixd/study/blob/master/common/src/test/java/com/github/whvixd/util/datastructure/ArrayListTest.java)

## 1. What？

### 1. Definition
Java中ArrayList是一种动态数组，对数组的封装。它的容量能动态增长。它继承于AbstractList，实现了List，RandomAccess，Cloneable，java.io.Serializable这些接口。

### 2. Diagram

<html>
    <img src="/img/data-structure/ArrayListDiagram.png" width="500" height="500" /> 
</html>

> ArrayList中的操作是线程非安全的，所以在多线程中使用 `CopyOnWriteArrayList` 或 `Collections.synchronizedList(new ArrayList<>());`

---

## 2. How？

### 1. Fields

```
    /**
     * 存储元素的数组
     */
    private transient Object[] elements;

    /**
     * 数组存储元素长度
     */
    private int size;

    /**
     * 默认容量长度
     */
    private static final int DEFAULT_CAPACITY=10;

    /**
     * 默认数组
     */
    private static final Object[] DEFAULT_EMPTY_ELEMENTS={};
    
```

> 数组元素不使用泛型的原因：泛型在运行时会被擦除，这里需要在加载时赋值，所以用 `Object`，不用 `T`

### 2. Constructor

```
    /**
     * 默认构造器，初始化时赋值空数组
     */
    public ArrayList(){
        elements=DEFAULT_EMPTY_ELEMENTS;
    }

    /**
     * 有参构造器
     * @param capacity 初始化容量
     */
    public ArrayList(int capacity){
        if(capacity>0){
            elements=new Object[capacity];
        }else if(capacity==0){
            elements=DEFAULT_EMPTY_ELEMENTS;
        }else {
            throw new IllegalArgumentException("capacity is illegal!");
        }
    }
```

### 3. Methods

#### 1. 查

```
    /**
     * 根据下标定位
     * @param index 下标
     * @return 元素
     */
    @SuppressWarnings("unchecked")
    public E get(int index){
        checkIndex(index);
        return elementData(index);
    }

    /**
     * 查询下标
     * @param o 元素
     * @return 下标
     */
    public int indexOf(Object o){
        if(o==null){
            for(int i=0;i<size;i++){
                if(elements[i]==null){
                    return i;
                }
            }
        }else {
            for(int i=0;i<size;i++){
                if(o.equals(elements[i])){
                    return i;
                }
            }
        }
        return -1;
    }

    /**
     * 从数组的尾部开始查询下标
     * @param o 元素
     * @return 下标
     */
    public int lastIndexOf(Object o){
        if(o==null){
            for(int i=size-1;i>=0;i--){
                if(elements[i]==null) {
                    return i;
                }
            }
        }else {
            for(int i=size-1;i>=0;i--){
                if(o.equals(elements[i])){
                    return i;
                }
            }
        }
        return -1;
    }

```

#### 2. 增

```
    /**
     * 添加新元素到数组尾部
     * @param o 新加入的元素
     * @return true
     */
    public boolean add(E o){
        ensureCapacityInternal(size+1);
        elements[size++]=o;
        return true;
    }

    /**
     *
     * @param index 添加到哪个下标
     * @param o 新元素
     * @return true
     */
    public boolean add(int index,E o){
        checkIndex(index);
        ensureCapacityInternal(size+1);
        // 从原来数组的index下标复制到elements中，从index+1开始，到size-index结束
        System.arraycopy(elements,index,elements,index+1,size-index);
        elements[index]=o;
        size++;
        return true;
    }
```

#### 3. 删

```
    /**
     * 删除最后一个元素
     * @return boolean
     */
    public boolean remove(){
        if(size>0){
            elements[--size]=null;
            return true;
        }
        return false;
    }

    /**
     * 根据下标删除
     * @param index 要删除的下标
     * @return boolean
     */
    public boolean remove(int index){
        checkIndex(index);
        System.arraycopy(elements,index+1,elements,index,size-index);
        elements[--size]=null;
        return true;
    }

    /**
     * 根据元素删除
     * @param o 要删除的元素
     * @return boolean
     */
    public boolean remove(Object o){
        if(o==null){
            for(int i=0;i<size;i++){
                if(elements[i]==null){
                    remove(i);
                    return true;
                }
            }
        }else {
            for(int i=0;i<size;i++){
                if(o.equals(elements[i])){
                    remove(i);
                    return true;
                }
            }
        }
        return false;
    }

```

#### 4. 改

```
    /**
     * 修改数组元素
     * @param index 要修改的下标
     * @param o 新的值
     * @return 旧值
     */
    public E set(int index,E o){
        checkIndex(index);
        E oldData=elementData(index);
        elements[index]=o;
        return oldData;
    }
```

#### 5. 其他

```
    /**
     * 大小
     * @return 数组元素容量
     */
    public int size(){
        return size;
    }
    
    /**
     * 是否为空
     * @return boolean
     */
    public boolean isEmpty() {
        return size == 0;
    }

    /**
     * 是否包含元素
     * @param o 元素
     * @return boolean
     */
    public boolean contains(E o){
        return indexOf(o)!=-1;
    }

    /**
     * 清空数组
     */
    public void clear(){
        if(size>0){
            for(int i=0;i<size;i++){
                elements[i]=null;
            }
            size=0;
        }
    }

    @SuppressWarnings("unchecked")
    public ArrayList<E> clone(){
        try {
            ArrayList<E> clone=(ArrayList<E>) super.clone();
            clone.elements=Arrays.copyOf(elements,size);
            return clone;
        } catch (CloneNotSupportedException e) {
            throw new RuntimeException(e);
        }
    }

    /**
     * 转为数组
     * @return 数组
     */
    public Object[] toArray(){
        return Arrays.copyOf(elements,size);
    }

    @SuppressWarnings("unchecked")
    private E elementData(int index){
        return (E) elements[index];
    }

    /**
     * 检查下标是否合法
     * @param index 下标
     */
    private void checkIndex(int index){
        if(index<0||index>=size){
            throw new IllegalArgumentException("index is illegal!");
        }
    }

    /**
     * 扩容机制:当前的长度大于总长度时，扩容原来的1.5倍
     * @param capacity 新的容量
     */
    private void ensureCapacityInternal(int capacity) {
        // 若是初始化的数组，则使用默认的大小
        if(DEFAULT_EMPTY_ELEMENTS==elements){
            capacity=Math.max(capacity,DEFAULT_CAPACITY);
        }
        // 是否超过真实的长度
        if(capacity>elements.length){
            int oldCapacity=elements.length;
            // 扩容1.5倍
            int newCapacity=oldCapacity+(oldCapacity>>1);
            // 初始化时数组长度是0，则需要与默认长度对比大小
            if(oldCapacity-capacity<0){
                newCapacity=capacity;
            }
            // 数组复制是native方法
            elements=Arrays.copyOf(elements,newCapacity);
        }
    }
```

> 扩容机制:当前的长度大于总长度时，扩容原来的1.5倍

---

> ***源码地址***:[传送门](https://github.com/whvixd/study/blob/master/common/src/main/java/com/github/whvixd/util/datastructure/ArrayList.java)

> ***单测代码***:[传送门](https://github.com/whvixd/study/blob/master/common/src/test/java/com/github/whvixd/util/datastructure/ArrayListTest.java)