---
title: hashmap 原理解析
date: 2015-07-21 17:38:32
tags:
	- java
categories:
	- 程序人生
---
HashMap的原理在面试时经常问到，也有很多人分析过，自己也写一写，仅供参考，部分内容参考别人的文章

## HashMap的数据结构
数组和链表是最基本的数据结构，但这两个基本是两个极端

### 数组

数组存储区间是连续的，占用内存严重，故空间复杂的很大。但数组的二分查找

时间复杂度小，为O(1)；数组的特点是：寻址容易，插入和删除困难；

### 链表

链表存储区间离散，占用内存比较宽松，故空间复杂度很小，但时间复杂度很大，达O（N）。链表的特点是：寻址困难，插入和删除容易。

 Hashmap实际上是一个数组和链表的结合体

![](http://img.blog.csdn.net/20150721153329490?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQv/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

hashmap中存数据的过程，通过key得到hashcode，利用hashcode可以得到在数组中的下标，如果下标已经有东西了，就插入到链表的表头。取的话，通过key得到hashcode，确定下标，如果不止一个元素，就通过equals遍历链表取得。

hashmap大小是2的幂次方，是因为这样可以尽可能的减小碰撞的发生，碰撞越多，性能越差。

hashmap中有个loadFactor参数，为0.75，这个值用来决定什么时候要resize hashmap（目的是减少碰撞的发生，但是这件事情很耗时间）

## HashMap部分源码

### Entry 键值对
HashMa加载因子 默认0.75
若:加载因子越大,填满的元素越多,好处是,空间利用率高了,但:冲突的机会加大了.链表长度会越来越长,查找效率降低。
反之,加载因子越小,填满的元素越少,好处是:冲突的机会减小了,但:空间浪费多了.表中的数据将过于稀疏
理Hash冲突的，形成链表

### 关键属性

加载因子 默认0.75

若:加载因子越大,填满的元素越多,好处是,空间利用率高了,但:冲突的机会加大了.链表长度会越来越长,查找效率降低。

反之,加载因子越小,填满的元素越少,好处是:冲突的机会减小了,但:空间浪费多了.表中的数据将过于稀疏

### hash算法
hashCode的算法就不讲解了，在hashmap中要找到某个元素，需要根据key的hash值来求得对应数组中的位置。如何计算这个位置就是hash算法。hashmap的数据结构是数组和链表的结合，所以我们当然希望这个hashmap里面的元素位置尽量的分布均匀些，方法就是取模运算，这样元素的分布比较均匀，java中是这么做的
```
	/**
     * Returns index for hash code h.
     */
    static int indexFor(int h, int length) {
        // assert Integer.bitCount(length) == 1 :  "length must be a non-zero power of 2";
        return h & (length-1);
    }
```
为什么数组的大小要是2的n次方大小，这样HashMap的性能最高。当length=2^n时，hashcode & (length-1) ==hashcode % length，位运算当然比取余效率高。length为2的整数次幂的话，为偶数，这样length-1为奇数，奇数的最后一位是1，这样便保证了h&(length-1)的最后一位可能为0，也可能为1（这取决于h的值），即与后的结果可能为偶数，也可能为奇数，这样便可以保证散列的均匀性

而如果length为奇数的话，很明显length-1为偶数，它的最后一位是0，这样h&(length-1)的最后一位肯定为0，即只能为偶数，这样任何hash值都只会被散列到数组的偶数下标位置上，这便浪费了近一半的空间。

在存储大容量数据的时候，最好预先指定hashmap的size为2的整数次幂次方。就算不指定的话，也会以大于且最接近指定值大小的2次幂来初始化的
```
	/**
     * Constructs an empty <tt>HashMap</tt> with the specified initial
     * capacity and load factor.
     *
     * @param  initialCapacity the initial capacity
     * @param  loadFactor      the load factor
     * @throws IllegalArgumentException if the initial capacity is negative
     *         or the load factor is nonpositive
     */
    public HashMap(int initialCapacity, float loadFactor) {
        if (initialCapacity < 0)
            throw new IllegalArgumentException("Illegal initial capacity: " +
                                               initialCapacity);
        if (initialCapacity > MAXIMUM_CAPACITY)
            initialCapacity = MAXIMUM_CAPACITY;
        if (loadFactor <= 0 || Float.isNaN(loadFactor))
            throw new IllegalArgumentException("Illegal load factor: " +
                                               loadFactor);

        // Find a power of 2 >= initialCapacity
        int capacity = 1;
       <span style="color:#ff6666;"> </span><span style="color:#3366ff;">while (capacity < initialCapacity)
            capacity <<= 1;

        this.loadFactor = loadFactor;
        threshold = (int)(capacity * loadFactor);
        table = new Entry[capacity];
        init();
    }
```
### put方法

```
public V put(K key, V value) {
        if (key == null)
            return putForNullKey(value);
        int hash = hash(key.hashCode());
        int i = indexFor(hash, table.length);
        for (Entry<K,V> e = table[i]; e != null; e = e.next) {
            Object k;
            if (e.hash == hash && ((k = e.key) == key || key.equals(k))) {
                V oldValue = e.value;
                e.value = value;
                e.recordAccess(this);
                return oldValue;
            }
        }

        modCount++;
        addEntry(hash, key, value, i);
        return null;
    }

//第2和3行的作用就是处理key值为null的情况，我们看看putForNullKey(value)方法

private V putForNullKey(V value) {
        for (Entry<K,V> e = table[0]; e != null; e = e.next) {
            if (e.key == null) {
                V oldValue = e.value;
                e.value = value;
                e.recordAccess(this);
                return oldValue;
            }
        }
        modCount++;
        addEntry(0, null, value, 0);
        return null;
    }
```
如果有null的元素侧替换掉

如果key为null的话，hash值为0，对象存储在数组中索引为0的位置。即table[0]
### get方法

```
 public V get(Object key) {
        if (key == null)
            return getForNullKey();
        int hash = hash(key.hashCode());
        for (Entry<K,V> e = table[indexFor(hash, table.length)];
             e != null;
             e = e.next) {
            Object k;
            if (e.hash == hash && ((k = e.key) == key || key.equals(k)))
                return e.value;
        }
        return null;
    }
```

### resize方法
在addEntry方法中判断如果map的大小超过阈值则进行扩容  

```
void addEntry(int hash, K key, V value, int bucketIndex) {
	Entry<K,V> e = table[bucketIndex];
        table[bucketIndex] = new Entry<K,V>(hash, key, value, e);
        if (size++ >= threshold)
            resize(2 * table.length);
    }
 void resize(int newCapacity) {
        Entry[] oldTable = table;
        int oldCapacity = oldTable.length;
        if (oldCapacity == MAXIMUM_CAPACITY) {
            threshold = Integer.MAX_VALUE;
            return;
        }

        Entry[] newTable = new Entry[newCapacity];
        transfer(newTable);
        table = newTable;
        threshold = (int)(newCapacity * loadFactor);
    } 
   
//调用了比较消耗性能的transfer方法

void transfer(Entry[] newTable) {
        Entry[] src = table;
        int newCapacity = newTable.length;
        for (int j = 0; j < src.length; j++) {
            Entry<K,V> e = src[j];
            if (e != null) {
                src[j] = null;
                do {
                    Entry<K,V> next = e.next;
                    int i = indexFor(e.hash, newCapacity);
                    e.next = newTable[i];
                    newTable[i] = e;
                    e = next;
                } while (e != null);
            }
        }
    }
```    

transfer方法，将HashMap的全部元素添加到新的HashMap中,并重新计算元素在新的数组中的索引位置，在HashMap数组扩容之后，最消耗性能的点就出现了：原数组中的数据必须重新计算其在新数组中的位置，并放进去，这就是resize，默认的情况下，数组大小16，当元素个数超过16*0.75=12时，就把数组扩展一倍32，重新计算每个元素在数组中的位置，数组的复制非常消耗性能，所以如果预知map的大小，那么初始元素个数能够提高map的性能。






