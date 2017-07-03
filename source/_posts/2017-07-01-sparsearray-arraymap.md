---
title: SparseArray和ArrayMap的优缺点和使用场景
date: 2017-07-01 22:10:06
toc: true
tags: Android
---

SparseArray和ArrayMap以及其衍生类都是Android独有的集合类，它们的衍生类主要有以下几个：

**SparseArray**
+ SparseBooleanArray
+ SparseIntArray
+ SparseLongArray
+ LongSparseArray

**ArrayMap**
+ ArraySet
 
Android平台设计这些集合类，它们的用途是作为HashMap和HashSet的替代品，但是也并非所有场景都适合使用SparseArray和ArrayMap。接下来我们先分析上面提到的几个类的原理，然后分析其优缺点以及使用场景。


# HashMap的原理以及缺点

首先我们先来分析HashMap的原理。

![HashMap](/images/hashmap.png)


上图绘制了HashMap的数据结构，HashMap使用一个数组存储数据，数组的每个成员是一个Entry<K,V>。而Entry<K, V>的数据结构是这样的：

```java
//参考自Android25最新的源代码
static class HashMapEntry<K,V> implements Map.Entry<K,V> {
    final K key;
    V value;
    HashMapEntry<K,V> next;
    int hash;
    ...
}
```
其中除了存储`key`，`hash`和`value`，还有一个`next`引用，所以这其实是一个链表，即HashMap中的数组存储的每个数据其实是一个由Entry<K, V>组成的链表。(这种方法就是“拉链法”，解决hash冲突的一种方法)。基于以上描述的HashMap的数据结构，查找的数据的流程主要分两步，第一步：输入key，根据某个hash算法求得该key在数组中的位置，取出数组中这个位置的Entry链表，一般链表的第一个结点就是当前key对应的Entry，如果不是，则查找链表的下一个Entry.

HashMap中还有两个重要的成员变量，影响着HashMap的性能，分别是“初始容量“和”加载因子“，“初始容量“代表以上提到的数组的初始长度，当数组的数据填充到一定量，则需要扩展数组（长度增大1倍），而什么触发这个增长呢？这就由”加载因子“决定，”加载因子“默认大小为`0.75`，代表当数组使用了其中的四分之三时，则需要扩展长度，而且，扩展长度时，需要重新执行一次hash构建。


讲完了HashMap最重要的几个特点，我们再观察上面的图片，正如图中反应的，HashMap的一大缺点就是消耗内存，数组中大多位置都未真正存储数据，特别是当数组需要增长时，增长一倍的长度会带来很大一片内存开销。而这个特点在Android平台上影响不容小觑，毕竟移动平台内存的占用一直是移动开发中需要谨慎把握的一个点。而且每次增长数组都需要重新构建hash。


正因为HashMap的以上特点，Android引入了另一个集合类：ArrayMap


# ArrayMap

![ArrayMap](/images/arraymap.png)


ArrayMap维护了两个数组，一个是保存hash值的数组，另一个数组则是保存key对象和value对象的数组，每一对key-value相邻存储在数组中。查找数据的过程大致这样，第一步：通过给定的key计算出对应hash值，找出这个hash值在第一个数组中的位置index，然后在第二个数组中找到2*index的位置，找到当前位置的key和value，如果不是对应的key，则在前后位置继续查找。


ArrayMap相比HashMap的优化点在于两点：第一、直接存储了key对象和value对象，而不用Entry对象；第二、当数组需要增长时，无需重新构建hash。

而ArrayMap相比HashMap的缺点在于，ArrayMap查找过程是两次二分查找法，而二分查找法的时间复杂度是O(logN)，这明显低于HashMap，大部分情况下，HashMap中hash冲突是比较少的，所以数据都存储在Entry链表的第一个结点，这种情况下查找复杂度只需一次hash，时间复杂度为O(1)。但是这个时间复杂度的缺点，在查找数据量小于1000时（这个数字是官方推荐的），影响微乎其微。

所以，综上，ArrayMap适用于数据量比较小的时候，替代HashMap。

另外补充一点，ArrayMap相比HashMap的缺点还有一点，ArrayMap没有实现Serializable，不利于在Android中借助Bundle传输。


文章最开始讲到ArraySet是ArrayMap的衍生类，我们介绍ArraySet之前，先介绍一下HashSet，


```java

public class HashSet<E> extends AbstractSet<E> implements Set<E>, Cloneable, java.io.Serializable {
    
    static final long serialVersionUID = -5024744406713321676L;

    private transient HashMap<E,Object> map;

    // Dummy value to associate with an Object in the backing Map
    private static final Object PRESENT = new Object();


    //...

    public boolean add(E e) {
        return map.put(e, PRESENT)==null;
    }

}


```

HashSet其实只是简单封装了一个HashMap，然后每个key值都插入同一个value。既然是基于HashMap，那么HashSet也有HashMap的缺点。于是Android平台推出了ArraySet，ArraySet和ArrayMap很相似，主要逻辑都是依靠两个数组，第一个数组保存hash值，第二个数组则不同，只需要保存key值（你可以这样理解，ArraySet和HashSet一样，所以key都对应同一个value）。而ArraySet除了具有ArrayMap节约内存的优点，同时也兼备了ArrayMap查找速度慢的缺点.


# SparseArray


讲到这里，我们已经介绍完ArrayMap和ArraySet。接下来介绍其他`XXXSparseXXX`集合，`Sparse`的意思是`稀疏的，稀少的`。它们基本都是为了解决HashMap的auto-boxing的问题，

```java

	HashMap hashmap;

	//...

	hashmap.put(5, object);

```

比如以上代码中，`5`这个int类型会被自动装箱为Integer。


所以`XXXSparseXXX`这一系列集合就是为了在ArrayMap基础上进一步解决auto-boxing的问题。而这一系列集合的实现原理和ArrayMap也相似，都是借助两个数组，不过ArrayMap的第一个数组保存的是key的hash值，而`XXXSparseXXX`这一系列集合的第一个数组保存的是key值，第二个数组直接保存value值。

以下是SparseArray的代码片段

```java

public class SparseArray<E> implements Cloneable {
    

    //...

    private int[] mKeys;
    private Object[] mValues;

    //...

}
```

以下是SparseBooleanArray的代码片段


```java
public class SparseBooleanArray implements Cloneable {

    //...

    private int[] mKeys;
    private boolean[] mValues;

    //...

}
```

以下做个总结，几个`XXXSparseXXX`集合类各自的key和value的数据类型。

```java
SparseArray          <int, Object>
SparseBooleanArray   <int, boolean>
SparseIntArray       <int, int>
SparseLongArray      <int, long>
LongSparseArray      <long, Object>
```


# 总结

ArrayMap和SparseArray（包括其衍生类）都是围绕时间换空间的方式来解决移动平台内存有限的问题。但是ArrayMap和SparseArray使用场景只能用于数据量比较低的情况，官方推荐不超过1000。而如果key或者value为基础类型，则可以进一步考虑SparseArray的以及其衍生的集合。


# 参考

https://stackoverflow.com/a/31413003/1290235
https://code.facebook.com/posts/857070764436276/memory-optimization-for-feeds-on-android/
http://www.jianshu.com/p/7b9a1b386265







