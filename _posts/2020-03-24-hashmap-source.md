---
layout:     post
title:      "HashMap源码分析"
subtitle:   "基于jdk1.8"
date:       2020-03-24 12:00:00
author:     "JoeBig7"
header-style: text
catalog: true
tags:
    - Java
    - Collection
---

## 前言
首先，我们都知道HashMap是Java中提供的一种容器，它是以key-value对的形式进行数据存储。本篇文章主要是对HashMap的存储原理以及Jdk1.8中对HashMap的优化来进行讲解。在此之前可以看一下HashMap的类继承结构图如下：

<center>
<img src="https://s1.ax1x.com/2020/03/24/8b7cse.png" alt="8b7cse.png" width=500 />
 
</center>

## 使用案例
在对源码进行解析的之前，我们先来看一个简单的创建存储案例，本文之后的分析基于该案例进行:

```
 public class Bootstrap {
    public static void main(String[] args) {
        Map<String, String> map = new HashMap<>();
        map.put("name", "joe");
    }
}
```


## 源码分析
上面的这个案例很简单，创建了一个HashMap对象并且在该容器放入了**name-joe**这对属性。接下来就从初始化和put这两个主要的操作来说明HashMap是如何工作的。

### 初始化
在我们的工作中，最常用的可能就是是`HashMap map = new HashMap()`这种方式来进行初始化，那就先从这个入口进行：
```
    public HashMap() {
        this.loadFactor = DEFAULT_LOAD_FACTOR; // all other fields defaulted
    }
```
通过这种方式进行初始化我们可以看到只是给**loadFactor**进行了默认的赋值，关于该变量的解释我会在**put**方法中进行详细说明，这边只需要知道这个负载因子和HashMap的扩容有关系。

除了这种初始化的方式，还可以使用`new HashMap(int initialCapacity, float loadFactor)`的方式来指定初始化的**容器大小**和**负载因子**。具体代码如下：
```
    public HashMap(int initialCapacity, float loadFactor) {
        if (initialCapacity < 0)
            throw new IllegalArgumentException("Illegal initial capacity: " +
                                               initialCapacity);
        if (initialCapacity > MAXIMUM_CAPACITY)
            initialCapacity = MAXIMUM_CAPACITY;
        if (loadFactor <= 0 || Float.isNaN(loadFactor))
            throw new IllegalArgumentException("Illegal load factor: " +
                                               loadFactor);
        this.loadFactor = loadFactor;
        this.threshold = tableSizeFor(initialCapacity);
    }
```
这种初始化的方式和上面的相比多了指定的两个参数：

对于**loadFactor负载因子**没有太多好说的（不过一般默认是0.75不会修改），只要满足是Float类型的数据就好；对于**initialCapatity**有几点需要注意：容器的大小需要在合法的限定范围内，大于0并且小于等于`1<<30`，另外还需要注意的是HashMap会将容器大小维持成**2的幂次方**，这么做的原因是为了改善hash寻址和扩容的效率，这个我也会在put和扩容的解析中进行详细的说明。


最后来看一下HashMap如何将传入的**initialCapatity**转换成2的幂次方的（tableSizeFor函数）。
```
    static final int tableSizeFor(int cap) {
        int n = cap - 1;
        n |= n >>> 1;
        n |= n >>> 2;
        n |= n >>> 4;
        n |= n >>> 8;
        n |= n >>> 16;
        return (n < 0) ? 1 : (n >= MAXIMUM_CAPACITY) ? MAXIMUM_CAPACITY : n + 1;
    }
```

HashMap通过无符号右移和按位或运算来将目标值转换为2的幂次方，转换过程如下：
```
 # 假设initialCapatity的值是60，那么n=59，至于为何要减1再做运算，是为了保证获取到的值大于或者是等于n的
 # 1. n|=n>>>1
 0 0 1 1 1 0 1 1  
 0 0 0 1 1 1 0 1
 ---------------
 0 0 1 1 1 1 1 1
 # 2. n|=n>>>2
 0 0 1 1 1 1 1 1
 0 0 0 0 1 1 1 1
 ---------------
 0 0 1 1 1 1 1 1
 
 #3. n|=n>>>4
 0 0 1 1 1 1 1 1
 0 0 0 0 0 0 1 1
 ---------------
 0 0 1 1 1 1 1 1
 
 #4. n|=n>>>8
 0 0 1 1 1 1 1 1
 0 0 0 0 0 0 0 0 
 ---------------
 0 0 1 1 1 1 1 1
 
 #5. n|=n>>>16
 0 0 1 1 1 1 1 1
 0 0 0 0 0 0 0 0 
 ---------------
 0 0 1 1 1 1 1 1====>63
 
 # 6.返回n+1 64
```


通过这种位运算的方式可以高效获得2的幂次方的值

> 到这边初始化的工作基本就完成了，接下来就分析一下如何进行值的存放


### put原理

#### 存储机制
在说明put是如何进行的之前，有必要先说一下HashMap的数据结构。首先HashMap是基于一个数组以及双向链表(红黑树，这是jdk1.8的优化)。这里我用一个图来进行表示：
<center>
<img src="https://s1.ax1x.com/2020/03/24/8q2Dqx.png" alt="8q2Dqx.png" border="0" width=750/>
</center>
<center><h7>图示：HashMap数据存储结构图</h7></center>

#### put源码解析
从上面的图中可以了解到HashMap完整的存储结构，现在来分析一下执行具体的put操作后底层到底发生了什么。

当案例中执行`map.put("name", "joe");`的时候，具体的代码如下
```
 public V put(K key, V value) {
        return putVal(hash(key), key, value, false, true);
    }
```
可以看见，调用了`putVal`方法，在进入该方法之前，首先针对key进行了hash值的运算，具体代码如下：
```
static final int hash(Object key) {
        int h;
        return (key == null) ? 0 : (h = key.hashCode()) ^ (h >>> 16);
    }
```
从这个代码块来看，首先可以确定HashMap的key是可以为null的，它对应的hash就是0；除此之外，针对hash会采用让高位参与按位运算的方式计算hash值，至于为何要通过这种方式是因为这样的话尽量少的发生hash碰撞。因为所有的key都会通过hash和数组的长度来进行数组位置的定位（取余运算）。并且在HashMap中会通过`(n-1)&hash`值来进行取余运算，而有一些key比如Float类型的值，计算出的hash值地位都是0，这样的话会存在大量的key都定位在0位置从而hash分布极不均匀，效率会大大降低。下面我通过一个例子来演示一下
```
# 假设数组长度为16，如果传入的key是一个Float a=1111.000001F;那么它对应的hash值为1149952000，转换成二进制数，此时做(n-1)&hash运算
01000100100010101110000000000000
00000000000000000000000000001101
--------------------------------
00000000000000000000000000000000
```
可以发现如果低位都是0计算出来的结果也是0，但是当通过`(h = key.hashCode()) ^ (h >>> 16)`后我们再来一下结果
```
# h = key.hashCode()) ^ (h >>> 16)
01000100100010101110000000000000
-------------------------------
00000000000000000100010010001010

# 再进行(n-1)&hash运算
00000000000000000100010010001010
00000000000000000000000000001101
-------------------------------
00000000000000000000000000001000
```
> 通过让高位参与运算后结果变成了8，借助这种方式可以更大程度的避免hash冲突，提高存储的效率


对key进行hash求值后，接下来就要在数组中进行数据的添加，涉及到的核心方法如下：
```
 final V putVal(int hash, K key, V value, boolean onlyIfAbsent,
                   boolean evict) {
        Node<K,V>[] tab; Node<K,V> p; int n, i;
        if ((tab = table) == null || (n = tab.length) == 0)                           #1
            n = (tab = resize()).length;
        if ((p = tab[i = (n - 1) & hash]) == null)                                    #2
            tab[i] = newNode(hash, key, value, null);                                 #3           
        else {                                                                        #4 
            Node<K,V> e; K k;
            if (p.hash == hash &&
                ((k = p.key) == key || (key != null && key.equals(k))))               #5 
                e = p;
            else if (p instanceof TreeNode)
                e = ((TreeNode<K,V>)p).putTreeVal(this, tab, hash, key, value);
            else {                                                                    #6 
                for (int binCount = 0; ; ++binCount) {                                #7 
                    if ((e = p.next) == null) {
                        p.next = newNode(hash, key, value, null);                     #8      
                        if (binCount >= TREEIFY_THRESHOLD - 1) // -1 for 1st       
                            treeifyBin(tab, hash);                                    #9
                        break;
                    }
                    if (e.hash == hash &&
                        ((k = e.key) == key || (key != null && key.equals(k))))       #10
                        break;
                    p = e;
                }
            }
            if (e != null) { // existing mapping for key
                V oldValue = e.value;                                                 #11
                if (!onlyIfAbsent || oldValue == null)
                    e.value = value;                                                  #12
                afterNodeAccess(e);
                return oldValue;
            }
        }
        ++modCount;
        if (++size > threshold)
            resize();                                                                 #13
        afterNodeInsertion(evict);
        return null;
    }

```
首先我对主要的步骤进行编号和讲解，并且对比较重要的步骤可能会扩展补充，编号以**#**开头，在上面的代码中可以看见。

当一个put操作进来之后，会判断table数组时候存在，如果不存在（**#1**）需要对数组进行初始化(关于扩容和初始化下面会额外讲解)；

如果table已经存在，通过`(n-1)&hash`<该操作在计算hash的时候已经有过说明>求出对应的数组下标，并且如果下标处的结点为空(**#2**)，则表示当前位置还没有链表存在，直接创建一个新的节点即可（**#3**）

对应的如果判断节点已经存在(**#4 **),则首先判断头结点是否就是我们需要的那个节点（需要判断hash和key都相等，**#5**）,这边可以引申出为什么在Java中重写了equals方法还要从hashcode方法，首先Java中的同一个对象如果equals方法相等，那么hashcode返回也应该相等。如果不重写，这对自定义的类，可能我们定义了只要指定属性相等equals就返回true。但是如果没有重写hashcode方法的话，两个对象返回的仍然是Object定义的默认hashcode结果（实际上就是对象地址），这样的话在比较的时候我们逻辑上应该需要返回true但是会返回false，因此就会出现错误。并且代码中借用hashcode先行判断提高了效率，如果不符合直接返回false了。

在上面的情况都不满足的话，那么说明节点已经存在并且不是头结点(**#6**)。这样的话就要遍历这个单向链表来查找对应的节点。如果遍历到下一个节点为空了，那么就在链表尾部添加一个新的节点(**#8**),并且在添加换成之后，判断节点数有没有达到8个，如果达到了的话转换成红黑树(**#9**)。如果在遍历的过程中发现，有对应的节点已经存在，那么结束遍历，此时节点保存在临时变量e中(**#10**)。

判断是否是节点已经存在，如果是，获取到旧的值（**#11**）并且把新值赋值给节点（**#12**）。到这一步put操作已经完成

最后是判断put好之后，元素的个数是否已经大于阈值threshold（阈值会在扩容的时候提及），判断是否进行扩容，如果需要进行扩容的话，还需要执行`resize()`方法。（**#13**）

#### resize源码解析
在put的过程中，有几处地方需要进行扩容，接下分析下具体的扩容情况：

```
final Node<K,V>[] resize() {
        Node<K,V>[] oldTab = table;
        int oldCap = (oldTab == null) ? 0 : oldTab.length;
        int oldThr = threshold;
        int newCap, newThr = 0;
        if (oldCap > 0) {
            if (oldCap >= MAXIMUM_CAPACITY) {                                               #1
                threshold = Integer.MAX_VALUE;
                return oldTab;
            }
            else if ((newCap = oldCap << 1) < MAXIMUM_CAPACITY &&                           #2        
                     oldCap >= DEFAULT_INITIAL_CAPACITY)
                newThr = oldThr << 1; // double threshold                              
        }
        else if (oldThr > 0) // initial capacity was placed in threshold
            newCap = oldThr;                                                                #3 
        else {               // zero initial threshold signifies using defaults
            newCap = DEFAULT_INITIAL_CAPACITY;                                              #4
            newThr = (int)(DEFAULT_LOAD_FACTOR * DEFAULT_INITIAL_CAPACITY);                 #5
        }
        if (newThr == 0) {                                                                  #6
            float ft = (float)newCap * loadFactor;                                           
            newThr = (newCap < MAXIMUM_CAPACITY && ft < (float)MAXIMUM_CAPACITY ?           
                      (int)ft : Integer.MAX_VALUE);
        }
        threshold = newThr;
        @SuppressWarnings({"rawtypes","unchecked"})
        Node<K,V>[] newTab = (Node<K,V>[])new Node[newCap];                                 #7
        table = newTab;
        if (oldTab != null) {
            for (int j = 0; j < oldCap; ++j) {                                              #8
                Node<K,V> e;
                if ((e = oldTab[j]) != null) {
                    oldTab[j] = null;
                    if (e.next == null)
                        newTab[e.hash & (newCap - 1)] = e;
                    else if (e instanceof TreeNode)
                        ((TreeNode<K,V>)e).split(this, newTab, j, oldCap);
                    else { // preserve order
                        Node<K,V> loHead = null, loTail = null;
                        Node<K,V> hiHead = null, hiTail = null;
                        Node<K,V> next;
                        do {
                            next = e.next; 
                            if ((e.hash & oldCap) == 0) {                                   #9
                                if (loTail == null)
                                    loHead = e;
                                else
                                    loTail.next = e;
                                loTail = e;
                            }
                            else {
                                if (hiTail == null)
                                    hiHead = e;
                                else
                                    hiTail.next = e;
                                hiTail = e;
                            }
                        } while ((e = next) != null);
                        if (loTail != null) {
                            loTail.next = null;
                            newTab[j] = loHead;
                        }
                        if (hiTail != null) {
                            hiTail.next = null;
                            newTab[j + oldCap] = hiHead;
                        }
                    }
                }
            }
        }
        return newTab;
    }

```
同样，我还是会对关键步骤进行编号和讲解：

首先，会对table的容量（oldCap）和阈值（oldThr）进行判断，如果table容量大于0，表示已经存在了，那么如果容量的一半小于
MAXIMUM_CAPACITY，则将容量值扩大两倍（**#1,#2**）。

如果table容量没有大于0，但是oldThr已经大于0了，表示初始化的时候设置了threshold（事实上就是初始化的时候initialCapacity进行2的幂次方的值），这种情况就把新的容量设置为oldThr即可（**#3**）。

如果上述情况都不满足，代表这是一个新的HashMap，还没有任何值。此时把newCap设置为`DEFAULT_INITIAL_CAPACITY`也就是16，并且把threshold设置为`newCap * loadFactor`,到这边就可以解释loadFactor是用来设置阈值用的默认为0.75(**#4,#5*)。

如果是oldThr但是table是空的情况，则表示知识指定了oldThr的值，但是table还没有被初始化，这种情况newThr的值是`newCap * loadFactor`,和上一步的区别是这边指定了容器大小(**#6*)。

接下来是初始化一个新的table数组，如果是第一次进来，对应的就是put中数组为空时的resize，那么到这边resize就结束了，返回数组即可(**#7*)。

如果旧的数组已经有值了，那么接下来就需要resize的核心操作，将旧的链表数据移动到新的数组中去（rehash）。对于如何移动我用一面一张图来展示:
<center>
<img src="https://s1.ax1x.com/2020/03/24/8L3UN8.png" alt="8L3UN8.png" border="0" width=700/>
</center>
<center><h7>图示：HashMap扩容机制图</h7></center>

可以看到由于hashMap是以2倍扩容，所以将容量初始化成2的幂次方也是提高了扩容的效率，因为这样子的话rehash的时候不要重新进行计算，因为定位到的位置要么是在原来的索引处，要么是在`oldCap+原来的索引数`。可以看到HashMap还用`e.hash & oldCap==0`将原来单链表中的数据进行了划分，如果满足条件的节点还是待在原来的索引处，不满足条件的移动到`oldCap+原来的索引数`这样子的话让hash更加的分散，效率也会有所提升，这也是jdk1.8的优化的地方

> 到这边HashMap put和resize的源码基本解读完毕，对于树的转换由于篇幅原因这边不详细讲解，有需要的可以自行查看源码

## 总结
通过本篇文章可以了解到HashMap底层的运作原理，并且可以发现，它的扩容相对来说还是比较耗时的，因此如果是存储比较多的数据的时候，最好初始化的时候就指定合适的大小。

另外，jdk1.8已经在减少hash碰撞（通过高位参与运算以及通过按位与运算代替取余操作）和扩容（保持容器大小2的幂次方不用重新rehash）以及数据结构（加入红黑树）上都做了优化，进一步提高了效率。


## 参考
- [1] [JDK 8源码](https://www.oracle.com/java/technologies/javase-jdk8-doc-downloads.html)
- [2] [Java 8系列之重新认识HashMap](https://tech.meituan.com/2016/06/24/java-hashmap.html)
- [3] [Java编程思想第四版](https://book.douban.com/subject/2130190/)