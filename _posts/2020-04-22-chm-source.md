---
layout:     post
title:      "ConcurrentHashMap源码分析"
subtitle:   "基于jdk1.8"
date:       2020-04-22 12:00:00
author:     "JoeBig7"
header-style: text
catalog: true
tags:
    - Java
    - Collection
---

## 前言
看标题就知道本篇文章主角**是ConcurrentHashMap**，在讲解它之前有几个问题有必要弄清楚。(本文都是基于jdk1.8进行分析)
### 什么是ConcurrentHashMap?
如果看过我之前写的关于HashMap的文章，应该知道它是基于**key-value**形式存储数据的一种hash存储表，ConcurrentHashMap的存储结构和HashMap是一样的。

### ConcurrentHashMap和HashMap的不同点
虽然都可以用来进行**key-value**对存储，但是HashMap是线程不安全的，但是ConcurrentHashMap是线程安全的。这也意味着前者在多线程环境可能会出现数据丢失的情况，后者则可以保证读取数据的安全性。

### ConcurrentHashMap继承图
![GWtdEV.png](https://s1.ax1x.com/2020/04/08/GWtdEV.png)

> 为了方便简洁，之后ConcurrentHashMap都简称为CHM

## 案例
在分析具体的代码之前，让我们先看一个CHM的使用案例
```
public class Bootstrap {
    private static ConcurrentHashMap<String, String> map = new ConcurrentHashMap<>();
//  private static HashMap<String, String> map = new HashMap<>();
    public static void main(String[] args) {
        Thread thread1 = new Thread(() -> {
            for (int i = 0; i < 400; i++) {
                map.put(String.valueOf(i), String.valueOf(i));
            }
        });
        Thread thread2 = new Thread(() -> {
            for (int i = 0; i < 200; i++) {
                map.put(String.valueOf(i), String.valueOf(i));
            }
        });
        Thread thread3 = new Thread(() -> {
            for (int i = 0; i < 400; i++) {
                map.remove(String.valueOf(i));
            }
        });

        thread1.start();
        thread2.start();
        thread3.start();
    }
}
```
这个案例很简单，定义了三个线程同时对CHM进行数据的增加和删除操作，对于CHM来说可以保证数据不会丢失，确保多线程下数据安全性问题。同样如果用HashMap来代替的话是可能出现数据被覆盖的风险的（关于HashMap可以看我之前的文章），本文就通过案例来进入具体源码的讲解
## 源码解析
对于CHM源码的解析，我主要通过三个方向进行阐述：分别是CHM的初始化、添加元素的操作（put）、扩容机制。通过这三个方面就可以比较清楚CHM是进行数据保存已经如何确保线程安全的。

### 初始化
首先从初始化开始，先从我们案例的初始化开始：
```
    public ConcurrentHashMap() {
    }
```
可以看见，如果你没有指定任何参数，初始化的时候CHM是不会做任何操作，当然你也可以提前指定好一些参数：
```
    public ConcurrentHashMap(int initialCapacity) {
        if (initialCapacity < 0)
            throw new IllegalArgumentException();
        int cap = ((initialCapacity >= (MAXIMUM_CAPACITY >>> 1)) ?
                   MAXIMUM_CAPACITY :
                   tableSizeFor(initialCapacity + (initialCapacity >>> 1) + 1));
        this.sizeCtl = cap;
    }
    
    
    //涉及到的常量:
    //MAXIMUM_CAPACITY是给定的可能的最大容量值
    private static final int MAXIMUM_CAPACITY = 1 << 30;

```
```
   private static final int tableSizeFor(int c) {
        int n = c - 1;
        n |= n >>> 1;
        n |= n >>> 2;
        n |= n >>> 4;
        n |= n >>> 8;
        n |= n >>> 16;
        return (n < 0) ? 1 : (n >= MAXIMUM_CAPACITY) ? MAXIMUM_CAPACITY : n + 1;
    }
```
上面的代码是指定了**initialCapacity**，通过initialCapacity可以计算出对应的容量:判断它是否比MAXIMUM_CAPACITY的一半还大，如果是的话直接将容量设置为MAXIMUM_CAPACITY,否则通过tableSizeFor函数将它设置为离c最近的2的幂次方的数（此函数我在HashMap源码解析中有具体的说明），最后将计算好的值赋值给sizeCtl（这个变量是CHM的关键变量，之后会有详细讲解）。到这边CHM的初始化就结束了。
### put方法
初始化完成后，就可以调用put方法来存放值了，具体的代码如下：
```
    public V put(K key, V value) {
        return putVal(key, value, false);
    }

    final V putVal(K key, V value, boolean onlyIfAbsent) {
        //需要注意的是CHM的key和value都是不能为空的
        if (key == null || value == null) throw new NullPointerException();
    #1  int hash = spread(key.hashCode());
        //该值用来计算链表的节点数
        int binCount = 0;
        //进入自旋操作
        for (Node<K,V>[] tab = table;;) {
            Node<K,V> f; int n, i, fh;
    #2      if (tab == null || (n = tab.length) == 0)
                tab = initTable();
            //如果数组不为空则需要去查找对应的节点，通过CAS的方式去定
            else if ((f = tabAt(tab, i = (n - 1) & hash)) == null) {
    #3      if (casTabAt(tab, i, null,
                             new Node<K,V>(hash, key, value, null)))
                    break;                   // no lock when adding to empty bin
            }
    #4      else if ((fh = f.hash) == MOVED)
                tab = helpTransfer(tab, f);
            else {
                V oldVal = null;
                //通过synchronized将节点进行锁定
    #5          synchronized (f) {
                    //判断在锁定之前节点有没有被修改，防止其他线程的操作
                    if (tabAt(tab, i) == f) {
                        //判断节点hash是否大于0，大于0则表示是链表
                        if (fh >= 0) {
                           //binCount的值设置为1，binCount用来计算链表结点的个数
                            binCount = 1;
                            //进行循环遍历，如果key和hash都是相等的，更新值 
                            //注意binCount的值计算的是新节点添加的之前的链表的个数，后面会在新节点到来的时候去判断，如果加入新的节点，链表长度已经达到了8，就会进行树的转换，
                            for (Node<K,V> e = f;; ++binCount) {
                                K ek;
                                if (e.hash == hash &&
                                    ((ek = e.key) == key ||
                                     (ek != null && key.equals(ek)))) {
                                    oldVal = e.val;
                                    //这边判断是否只在不存在的时候更新
                                    if (!onlyIfAbsent)
                                        e.val = value;
                                    break;
                                }
                                //如果更新值失败表示节点不存在，那么继续遍历链表，直到下一个是空节点的时候，创建新节点,此时数据就添加完成了
                                Node<K,V> pred = e;
                                if ((e = e.next) == null) {
                                    pred.next = new Node<K,V>(hash, key,
                                                              value, null);
                                    break;
                                }
                            }
                        }
                        //否则节点属于树
                        else if (f instanceof TreeBin) {
                            Node<K,V> p;
                            binCount = 2;
                            //需要在树中添加节点
                            if ((p = ((TreeBin<K,V>)f).putTreeVal(hash, key,
                                                           value)) != null) {
                                oldVal = p.val;
                                if (!onlyIfAbsent)
                                    p.val = value;
                            }
                        }
                    }
                }
        #6      if (binCount != 0) {
                    if (binCount >= TREEIFY_THRESHOLD)
                        treeifyBin(tab, i);
                    if (oldVal != null)
                        return oldVal;
                    break;
                }
            }
        }
    #7  addCount(1L, binCount);
        return null;
    }
    
    涉及到的常量：
    //CHM存放数据的表
    transient volatile Node<K,V>[] table;
    //转换成树的临界点
    static final int TREEIFY_THRESHOLD = 8;
    
```
这一块我主要是针对数据存放来讲解，涉及到扩容会放到下一节统一分析。首先我们可以思考一个问题：CHM存放数据和HashMap比有什么挑战？很显然，是需要考虑在多线程环境如何让数据安全的存放到指定的节点，我对put的主要步骤会进行**编号**，下面我们来一一分析：

**#1**：第一步很显然，我们需要对key进行hash值的计算，但是它的计算方法和HashMap有点诧异我们看一下代码：
```
    static final int spread(int h) {
        return (h ^ (h >>> 16)) & HASH_BITS;
    }
    
    //涉及常量:
    static final int HASH_BITS = 0x7fffffff;
```
可以看见总体上是和HashMap类似，但是多了一个HASH_BITS,它是一个16进制的数，转换成二进制数的话除了最高位是0其余都是1，通过和它取余，那么获取到的hash值永远是**大于0**的。至于为何要确保hash值大于0，是因为CHM存在hash的负数的其他节点用来标识一些特别的功能。

**#2**：这一步是在table为空情况下，需要进行初始化操作，这边先用一张图来解释多线程下如何确保安全初始化：

![JtRJPK.png](https://s1.ax1x.com/2020/04/22/JtRJPK.png)

**具体代码如下**:
```
    private final Node<K,V>[] initTable() {
        //定义一个临时节点变量和一个int类型变量sc
        Node<K,V>[] tab; int sc;
        //当table为空或者长度等于0的情况下进行循环
        while ((tab = table) == null || tab.length == 0) {
            //这里判断sizeCtl的值是否小于0，这个值在初始化的时候提到过，这边用它来判断是否需要进行初始化
            //小于0是因为已经有别的线程正在初始化table数组，需要暂停线程
            if ((sc = sizeCtl) < 0)
                Thread.yield(); // lost initialization race; just spin
            //否则通过CAS的方式直接操作内存将sizeCtl设置为-1，表示当前线程正在初始化
            else if (U.compareAndSwapInt(this, SIZECTL, sc, -1)) {
                try {
                    //此时第二次判断table是否为空，来防止其他线程已经初始化了
                    if ((tab = table) == null || tab.length == 0) {
                        //设置sizeCtl的值，如果>0直接用否则设置成默认值16
                        int n = (sc > 0) ? sc : DEFAULT_CAPACITY;
                        @SuppressWarnings("unchecked")
                        //初始化节点数组赋值给table
                        Node<K,V>[] nt = (Node<K,V>[])new Node<?,?>[n];
                        table = tab = nt;
                        //设置新的sizeCtl用来判断下一次扩容的临界点
                        sc = n - (n >>> 2);
                    }
                } finally {
                    sizeCtl = sc;
                }
                break;
            }
        }
        return tab;
    }

    //涉及常量:
    //默认的表容量
    private static final int DEFAULT_CAPACITY = 16;
```

**#3**：这一步是数组存在，但是定位到的节点不存在，那么直接通过CAS的方式操作内存，来对节点进行创建

**#4**：这一步定位到的节点的hash值是MOVED，表示别的线程正在对该节点进行扩容操作，那么就需要去加入扩容。这边不详细讲扩容

**#5**: 这步是真正的添加新的数据，主要步骤我在代码中进行注释说明。

**#6**：在数据添加完成后，通过binCount判断链表的长度是否大于了8，如果是的话需要将链表转换成红黑树

**#7**：最后一步是发生在添加数据完成之后，主要有两个功能：添加计数以及判断是否需要扩容，具体代码如下:
```
 private final void addCount(long x, int check) {
        CounterCell[] as; long b, s;
        //如果counterCells数组不为空或者CAS方式设置baseCount失败的话，表示多个线程修改了table
        if ((as = counterCells) != null ||
            !U.compareAndSwapLong(this, BASECOUNT, b = baseCount, s = b + x)) {
            CounterCell a; long v; int m;
            boolean uncontended = true;
            if (as == null || (m = as.length - 1) < 0 ||
                (a = as[ThreadLocalRandom.getProbe() & m]) == null ||
                !(uncontended =
                  U.compareAndSwapLong(a, CELLVALUE, v = a.value, v + x))) {
                fullAddCount(x, uncontended);
                return;
            }
            if (check <= 1)
                return;
            s = sumCount();
        }
        
        //...省略扩容相关
    }
```
CHM中计数涉及到baseCount和CounterCell两个概念，一种情况是没有竞争关系，那么直接通过CAS的方式添加baseCount即可,如果是有竞争的话，就用CounterCell数组去计数，因为没有线程都有唯一的Probe值通过该值每一个线程都会对应到CounterCell数组的一个位置然后进行数的叠加。如果在叠加过程还是有冲突，则需要对CounterCell数组的操作也进行自旋操作直到数添加成功。

CHM通过这种方式来提高多线程环境下累加count的效率，类似分而治之的思想。



### 扩容
对于扩容，主要有两处地方涉及到：

#### 新增元素后
这是在put完成后会判断当前元素容量时候需要进行扩容，相关代码如下：
```
 private final void addCount(long x, int check) {
   .....
   
   //如果链表之前的节点数大于等于0，就要去判断一次，添加了新元素后情况如何
      if (check >= 0) {
            Node<K,V>[] tab, nt; int n, sc;
            //首先是要满足条件:
            //1. 当前元素个数比sizeCtl大,达到扩容的阈值
            //2. table表不能为空并且table的容量小于MAXIMUM_CAPACITY
            while (s >= (long)(sc = sizeCtl) && (tab = table) != null &&
                   (n = tab.length) < MAXIMUM_CAPACITY) {
                //
                int rs = resizeStamp(n);
                //如果sc小于0表示有线程正在进行扩容
                if (sc < 0) {
                    //如果满足以下几个条件的任意一种，那么就不在需要扩容
                    //1. (sc >>> RESIZE_STAMP_SHIFT) != rs 当sizeCtl无符号右移16位后不等rs
                    //2. sc==rs+1
                    //3. sc==rs+MAX_RESIZERS
                    //4. (nt = nextTable) == null
                    //5. transferIndex<=0
                    if ((sc >>> RESIZE_STAMP_SHIFT) != rs || sc == rs + 1 ||
                        sc == rs + MAX_RESIZERS || (nt = nextTable) == null ||
                        transferIndex <= 0)
                        break;
                    //如果上面的条件都不满足，那么通过CAS的方式去设置sc，将它+1，如果成功的话表示当前线程需要进行扩容    
                    if (U.compareAndSwapInt(this, SIZECTL, sc, sc + 1))
                        //进行扩容操作
                        transfer(tab, nt);
                }
                //如果sc>0,表示还没有线程正在进行扩容，将sc设置为 (rs << RESIZE_STAMP_SHIFT) + 2，它是一个负数
                else if (U.compareAndSwapInt(this, SIZECTL, sc,
                                             (rs << RESIZE_STAMP_SHIFT) + 2))
                    //进行扩容
                    transfer(tab, null);
                s = sumCount();
            }
    
    }
```

resizeStamp方法:计算出扩容的标志位
```
   static final int resizeStamp(int n) {
        return Integer.numberOfLeadingZeros(n) | (1 << (RESIZE_STAMP_BITS - 1));
    }
    

    0000 0000 0000 0000 0000 0000 0000 1000
    0000 0000 0000 0000 1000 0000 0000 0000
    ---------------------------------------
    0000 0000 0000 0000 1000 0000 0000 1000
```

**具体扩容方法**:
```
    private final void transfer(Node<K,V>[] tab, Node<K,V>[] nextTab) {
        //定义两个变量，n表示tab的长度，stride(步的意思)变量
        int n = tab.length, stride;
        //这边去计算cpu核数，如果大于1那么就将stride赋值为(n >>> 3) / NCPU，否则就赋值为tab的长度
        //然后判断stride是否比MIN_TRANSFER_STRIDE（16）小，如果比它小则将stride赋值为16
        if ((stride = (NCPU > 1) ? (n >>> 3) / NCPU : n) < MIN_TRANSFER_STRIDE)
            stride = MIN_TRANSFER_STRIDE; // subdivide range
        //如果nextTab等于空，需要做如下操作
        if (nextTab == null) {            // initiating
            try {
                @SuppressWarnings("unchecked")
                //去初始化新的节点数组，该数组用来存放扩容后的数据，并且length是原来的两倍
                Node<K,V>[] nt = (Node<K,V>[])new Node<?,?>[n << 1];
                //然后将nt辅助给nextTable
                nextTab = nt;
            } catch (Throwable ex) {      // try to cope with OOME
                //如果操作错误，则将sizeCtl设置为Integer.MAX_VALUE，并且停止扩容
                sizeCtl = Integer.MAX_VALUE;
                return;
            }
            //做完后此时的nextTable就是初始化的新的数组
            nextTable = nextTab;
            //transferIndex设置为16，表示转移数据从原来的最后一个槽开始
            transferIndex = n;
        }
        
        //如果nextTable已经初始化过了，那么获取新数组的长度
        int nextn = nextTab.length;
        //并且将nextTab封装为ForwardingNode节点
        ForwardingNode<K,V> fwd = new ForwardingNode<K,V>(nextTab);
        //定义两个标志位，advance和finishing
        boolean advance = true;
        boolean finishing = false; // to ensure sweep before committing nextTab
        //进入循环，并且初始化i和bound为0
        for (int i = 0, bound = 0;;) {
            //定义一个节点变量f和一个int变量fh
            Node<K,V> f; int fh;
            //进入循环，当advance为true的时候
            while (advance) {
                //这边又定义了两个变量nextIndex和nextBound
                int nextIndex, nextBound;
                //如果满足将i减1后大于bound或者finishing的值为true的时候，就把advance设置为false
                if (--i >= bound || finishing)
                    advance = false;
                    
                //如果nextIndex也就是transefer<=0的时候将i-1并且把advance设置为false    
                else if ((nextIndex = transferIndex) <= 0) {
                    i = -1;
                    advance = false;
                }
                //除此之外，将transfer设置为nextbound，也就是判断nextIndex是否>16,如果是的话设置为 nextIndex - stride
                else if (U.compareAndSwapInt
                         (this, TRANSFERINDEX, nextIndex,
                          nextBound = (nextIndex > stride ?
                                       nextIndex - stride : 0))) {
                    //将区间下标设置为nextBound
                    bound = nextBound;
                    //如果是16则i=15,这是对i的第一次赋值
                    i = nextIndex - 1;
                    //然后将推进标识设置为false
                    advance = false;
                }
            }
            //在循环结束后，就有如下几种情况
            //如果i<0或者i>=数组长度或者i+n的值大于新数组的长度
            if (i < 0 || i >= n || i + n >= nextn) {
                int sc;
                //就去判断finishing是否为true，第一次进来肯定为false
                if (finishing) {
                    //是的话将nextTable设置为null
                    //nextTab设置为talbe的值
                    //设置新的sizeCtl的值
                    //最后返回标识扩容结束
                    nextTable = null;
                    table = nextTab;
                    sizeCtl = (n << 1) - (n >>> 1);
                    return;
                }
                //如果finishing不为true，第一次会进入这个条件
                //将sizeCtl的值-1
                if (U.compareAndSwapInt(this, SIZECTL, sc = sizeCtl, sc - 1)) {
                    如果sc-2不等于那么表示没有结束帮助扩容 
                    if ((sc - 2) != resizeStamp(n) << RESIZE_STAMP_SHIFT)
                        return;
                    //如果表示帮助扩容结束那么就将finishing和advance都设置为true
                    finishing = advance = true;
                    i = n; // recheck before commit
                }
            }
            //第二种情况是根据i，这边i是15获取对应的节点，如果为空
            else if ((f = tabAt(tab, i)) == null)
                //那么就去将fwd的节点设置进去，用去占位表示正在扩容
                advance = casTabAt(tab, i, null, fwd);
           
           //如果节点不是空的并且已经是fwd节点了
            else if ((fh = f.hash) == MOVED)
                //那么就继续推进，区别的节点进行扩容
                advance = true; // already processed
            else {
                //如果是正常的节点，那么就要进行扩容操作
                //通过synchronized将节点进行同步
                synchronized (f) {
                    //在此之前，再次判断节点是否变化，没变的话正式进行扩容
                    if (tabAt(tab, i) == f) {
                        定义ln和hn两个节点变量
                        Node<K,V> ln, hn;
                        如果hash>=0,表示是正常的节点
                        if (fh >= 0) {
                            //通过fh & n去计算节点在扩容后是否在原来的位置
                            int runBit = fh & n;
                            Node<K,V> lastRun = f;
                            for (Node<K,V> p = f.next; p != null; p = p.next) {
                                遍历链表，判断是否和runBit相等，如果不是则将runBit设置为新的值，并且将lastRun设置为当前节点
                                int b = p.hash & n;
                                if (b != runBit) {
                                    runBit = b;
                                    lastRun = p;
                                }
                            }
                            //0代表还在原来的位置
                            if (runBit == 0) {
                                ln = lastRun;
                                hn = null;
                            }
                            //代表在n+原来的位置 
                            else {
                                hn = lastRun;
                                ln = null;
                            }
                            //去遍历链表，然后创建新的节点，根据时候在原来的位置分别进行赋值，这个HashMap扩容机制类似
                            //需要注意的是新数组中的链表数据的顺序可能和原来的是不一样的
                            for (Node<K,V> p = f; p != lastRun; p = p.next) {
                                int ph = p.hash; K pk = p.key; V pv = p.val;
                                if ((ph & n) == 0)
                                    ln = new Node<K,V>(ph, pk, pv, ln);
                                else
                                    hn = new Node<K,V>(ph, pk, pv, hn);
                            }
                            setTabAt(nextTab, i, ln);
                            setTabAt(nextTab, i + n, hn);
                            setTabAt(tab, i, fwd);
                            advance = true;
                        }
                        else if (f instanceof TreeBin) {
                           ...由于代码过长，这边省略对树的扩容
                        }
                    }
                }
            }
        }
    }
```
 

#### 新增元素时
在添加新的元素时，可能会遇到节点正在进行扩容的情况，那么当前线程就要去协助扩容，具体代码如下：

```
   final Node<K,V>[] helpTransfer(Node<K,V>[] tab, Node<K,V> f) {
        Node<K,V>[] nextTab; int sc;
         //如果当前节点属于FowWardingNode并且table不为空，并且nextTable也不为空
        if (tab != null && (f instanceof ForwardingNode) &&
            (nextTab = ((ForwardingNode<K,V>)f).nextTable) != null) {
            int rs = resizeStamp(tab.length);
            while (nextTab == nextTable && table == tab &&
                   (sc = sizeCtl) < 0) {
                if ((sc >>> RESIZE_STAMP_SHIFT) != rs || sc == rs + 1 ||
                    sc == rs + MAX_RESIZERS || transferIndex <= 0)
                    break;
                //去进行扩容操作，将sc+1    
                if (U.compareAndSwapInt(this, SIZECTL, sc, sc + 1)) {
                    transfer(tab, nextTab);
                    break;
                }
            }
            return nextTab;
        }
        return table;
    }

```

## 总结
本文主要讲了CHM如何进行数据的存放已经如何进行扩容，并且解析了它是如何在多线程环境保证数据的安全性的


> 转载请注明出处！链接