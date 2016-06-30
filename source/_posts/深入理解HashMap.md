---
title: 深入理解HashMap
date: 2016-06-27 17:12:53
tags: [java,HashMap]
---

对于一个Java开发者来说，最常用的Map结构莫过于HashMap。但我发现很多人对其内部是如何进行存储和查找的基本没什么概念或者有点概念但却是错误的或不全面的，对于靠这个吃饭的人来说，如果你不了解他，你怎么能放心的把你的数据交给他呢，这就好比把自己的饭碗交给了一个不认识的人，that's terrible! 所以本文就带你深入理解一下HashMap, 内容大致涵盖如下几个方面:

1. 比较HashMap在java7和java8中的不同点
2. 性能
4. 可能的问题

# 存储
HashMap实现了`Map<K,V>`，所以包含了以下几个主要的方法:
```
V put(K key, V value)
V get(Object key)
V remove(Object key)
Boolean containsKey(Object key)
```

HashMap使用`Entry<K, V>`作为内部存储值的结构,基本结构:
```
static class Entry<K,V> implements Map.Entry<K,V> {
    final K key; // 键值
    V value;// 实际值
    Entry<K,V> next; // 下一条记录，构成一个单向链表
    int hash;// key的hash值，出于性能上的考虑，这个值主要是为了避免重复计算hash值
}
```

HashMap将数据存储在多个单向链表里面，这个单向链表通常叫做bucket，使用一个Entry<K,V>[]array来存储这些bucket. 这个array的默认大小为16.

```
/**
  * The default initial capacity - MUST be a power of two.
  */
static final int DEFAULT_INITIAL_CAPACITY = 1 << 4; // aka 16
```

这张示意图可以大致表示HashMap是怎么存的数据:

![图例](http://ww4.sinaimg.cn/mw690/50508d62gw1f59y3ncsu4j20ta0ji0ty.jpg)

`bucket array`的大小为n,i<sub>0</sub> 存储了3个entry构成的singly linked list,i<sub>1</sub>存了个null,i<sub>n-1</sub>只存了1个entry。

当调用put(K key, V value)或者get(Object key)时,就会根据key值计算相应的bucket在array中的index，然后进行添加或者获取Entry.

## 索引的计算方式

```
// java7
final int hash(Object k) {
    int h = hashSeed;
    if (0 != h && k instanceof String) {
        return sun.misc.Hashing.stringHash32((String) k);
    }

    h ^= k.hashCode();

    // This function ensures that hashCodes that differ only by
    // constant multiples at each bit position have a bounded
    // number of collisions (approximately 8 at default load factor).
    h ^= (h >>> 20) ^ (h >>> 12);
    return h ^ (h >>> 7) ^ (h >>> 4);
}

// java8
/**
 + Computes key.hashCode() and spreads (XORs) higher bits of hash
 + to lower.  Because the table uses power-of-two masking, sets of
 + hashes that vary only in bits above the current mask will
 + always collide. (Among known examples are sets of Float keys
 + holding consecutive whole numbers in small tables.)  So we
 + apply a transform that spreads the impact of higher bits
 + downward. There is a tradeoff between speed, utility, and
 + quality of bit-spreading. Because many common sets of hashes
 + are already reasonably distributed (so don't benefit from
 + spreading), and because we use trees to handle large sets of
 + collisions in bins, we just XOR some shifted bits in the
 + cheapest possible way to reduce systematic lossage, as well as
 + to incorporate impact of the highest bits that would otherwise
 + never be used in index calculations because of table bounds.
 */
static final int hash(Object key) {
    int h;
    return (key == null) ? 0 : (h = key.hashCode()) ^ (h >>> 16);
}

/**
 * Returns index for hash code h.
 */
static int indexFor(int h, int length) {
    // assert Integer.bitCount(length) == 1 : "length must be a non-zero power of 2";
    return h & (length-1);
}
```

为了高效的存取数据，bucket array的长度必须为2的指数次方，从上面默认大小16的注释上可以看到，接下来就解释一下为什么这个是必须的。

假设默认bucket array的默认大小是17，index即为h&(17-1),16的二进制表示为00010000，此时无论h的是什么，h&(17-1)的值只可能是16或者0，所以bucket array只有bucket 16和bucket 0能够被用到，其他都浪费了。但是如果bucket array的大小为2的指数次方，如默认大小2<sup>4</sup>，h&15的结果值为0-15，每个bucket都被使用了，不存在浪费情况。

由此可以看出bucket array的大小为2<sup>n</sup>是很重要的，当你指定一个非2<sup>n</sup>得大小时，HashMap采取的策略是向上取下一个2的指数次方。

# 自动扩展

在获取到bucket的index之后，就可以遍历bucket中的linked list了。设想我们的bucket中存储了大量数据，遍历linked list的时候就可能会产生性能问题。所以HashMap在发现自己存储的数据量超过某个阀值的时候就会自动对bucket array升级，即扩容。

HashMap提供了一个可指定初始容量和loadFactor的构造器:
```
public HashMap(int initialCapacity, float loadFactor)
```

默认的`initialCapacity`为16，`loadFactor`为0.75.

每次调用put()时都会执行检查是否需要扩容，当`map.size>(threshold=capacity*loadFactor)`bucket array的大小会扩展至原来的2倍,bucket的数量也增加了，所以随即发生的就是重新分布所有的元素。问题来了，为什么要这么做呢？扩展array bucket的主要目的就是减小bucket中的linked list的大小，这样才能快速的在其上进行CRUD操作。 在元素的重新分布之后，有相同hash值的存在同一个bucket中，之前在同一个bucket中但是hash值不一样的元素可能分配之后就不在同一个bucket中了。

过程如下图:

![初始结构](http://ww2.sinaimg.cn/mw690/50508d62gw1f59y3lvrypj20q20lo75b.jpg)
![扩展后的结构](http://ww3.sinaimg.cn/mw690/50508d62gw1f59y3mj4myj20y80cwgmt.jpg)

__HashMap虽然提供了扩容的机制，但没有提供相应的缩容机制。__

# 线程安全

大家都知道HashMap是非线程安全的，但深究过为什么麽？ 考虑如果一个写线程和一个读线程同时操作一个HashMap，写的时候如果发生扩容，那么读的时候就可能出现失败的现象，因为索引和list结构都变了。

HashMap的小伙伴HashTable的实现是线程安全的，因为其CRUD都做了同步操作，性能自然是不能同HashMap相比，除非在特殊场景，否则不推荐使用。

# key值可变性
通常我们可能会看到有人说最好是使用String and Integer作为Map的key值，但是为什么呢？ 主要原因是因为其`immutable`的特性。如果使用自己定义的对象作为key,请确保其hash不可变，否则可能丢失之前存储的数据。

# Java8的变化

从代码量上来看就知道java8做了一些改进，java7的HashMap只有1000行左右的代码，java8却用了2000多行。
Java8新增了一个Node对象，结构和Entry是一样的:
```
static class Node<K,V> implements Map.Entry<K,V> {
    final int hash;
    final K key;
    V value;
    Node<K,V> next;
}
```
bucket中存储的是这个新的Node对象。与Java7不同的地方是，Node可以扩展为TreeNode,TreeNode的实现是一个红黑树，这样增删等操作可以降到`O(log(n))`的级别。

TreeNode部分代码
```
static final class TreeNode<K,V> extends LinkedHashMap.Entry<K,V> {
    TreeNode<K,V> parent;  // red-black tree links
    TreeNode<K,V> left;
    TreeNode<K,V> right;
    TreeNode<K,V> prev;    // needed to unlink next upon deletion
    boolean red;
    TreeNode(int hash, K key, V val, Node<K,V> next) {
        super(hash, key, val, next);
    }

    /**
     + Returns root of tree containing this node.
     */
    final TreeNode<K,V> root() {
        for (TreeNode<K,V> r = this, p;;) {
            if ((p = r.parent) == null)
                return r;
            r = p;
        }
    }
}
```

使用红黑树的一个主要好处就是当某个bucket中存在很多元素时，查找的时间为`O(log(n)) `,而原来的为`O(n)`。

还有一点值得注意的是bucket中既可存放linked list,也可以存放TreeNode. 何时使用哪种结构取决于bucket中的元素个数。源码中定义了一个阀值:

```
/**
 + The bin count threshold for using a tree rather than list for a
 + bin.  Bins are converted to trees when adding an element to a
 + bin with at least this many nodes. The value must be greater
 + than 2 and should be at least 8 to mesh with assumptions in
 + tree removal about conversion back to plain bins upon
 + shrinkage.
 */
static final int TREEIFY_THRESHOLD = 8;
```

超过8个就转换成TreeNode，否则就使用Linked List.

# 性能问题

HashMap性能是否过关，主要取决于hash function的设计是否合理。 hash function如果设计不合理，bucket array就不能够被充分利用，假设有`n(n>=16)`个元素，bucket array长度为L,bucket<sub>i</sub>存储`k(k<n/2)`个元素,另一个bucket<sub>j</sub>存储n-k个,如果是使用java7，查找最坏情况需要`O(n-k)`, 但如果均匀分布的话,java7就只需要`O(n/L)`.

之前说推荐使用String or Integer作为Key就是因为其有不错的hash function。

# 自动扩展的开销

前面已经讲过，当put()的时候达到临界值时就会对bucket array进行扩容，每次扩容都是一个开销很大的过程，如果提前知道自己需要多大的容量，推荐在创建HashMap的时候就设置好initialCapacity and loadFactor.

# 结语

对于日常开发来说，上述各方面可能会让你觉得没太大用处，但对于一个认真的开发者来说，我们不能仅仅做到知其然，更要做到知其所以然，这样我们才能好的利用这些工具为我们解决问题，并且在出现问题的时候能够快速定位和得出解决问题的方案。That is what makes you different!





