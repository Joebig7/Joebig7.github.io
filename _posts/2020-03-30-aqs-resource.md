---
layout:     post
title:      "AQS源码分析"
subtitle:   "以ReentrantLock为例"
date:       2020-03-30 12:00:00
author:     "JoeBig7"
header-style: text
catalog: true
tags:
    - Java
    - Collection
---

## 前言
当我们在Java并发编程中，可能会时常使用**ReentrantLock**、**Semaphore**、**CountDwonLatch**等同步工具保证线程安全，但是我们可能对它们到底是如何保证线程安全并不是很清楚。本篇文章就通过详细解析他们的共同依赖类`AbstractQueuedSynchronizer`来探究它们是如何实现对资源的控制的。为了便于分析，本篇文章主要从**ReentrantLock**为切入点，阅读本文之后你们可以自行分析其他实现类的实现逻辑。

## 案例
首先，我们来回顾一下ReentrantLock的一个使用案例：
```
public class Bootstrap {
    private static final Lock lock = new ReentrantLock();
    private int count = 0;
    public void count() {
        try {
            lock.lock();
            count++;
        } finally {
            lock.unlock();
        }
    }
}
```
这个demo很简单，就是对count进行计数，并且通过ReentrantLock保证多线程环境下的数据安全问题。

## 源码分析
### 初始化
上面的代码首先通过ReentrantLock构造函数创建了一个Lock对象，我们来看一下实际创建了哪些对象。
```
    public ReentrantLock() {
        sync = new NonfairSync();
    }
```
从字面就可以看出创建了一个非公平锁对象，不过相信大家之前就知道ReentrantLock默认支持非公平锁，但是同样也支持公平锁，因此如果你通过`new ReentrantLock(true)`就可以创建一个公平锁对象,具体源码如下:
```
    public ReentrantLock(boolean fair) {
        sync = fair ? new FairSync() : new NonfairSync();
    }
```

到这边我们知道创建锁的两种方式，接下来通过具体的代码看一下这两种锁有什么区别：

**公平锁实现部分源码**：
```
    static final class FairSync extends Sync {
        private static final long serialVersionUID = -3000897897090466540L;

        final void lock() {
            acquire(1);
        }

        ...
    }
```

**非公平锁实现部分源码**:
```
 static final class NonfairSync extends Sync {
        private static final long serialVersionUID = 7316153563782823691L;
        final void lock() {
            if (compareAndSetState(0, 1))
                setExclusiveOwnerThread(Thread.currentThread());
            else
                acquire(1);
        }

        ...
    }
```
可以看见非公平锁多了一个判断条件，现在我暂时告诉你这个条件是让当前线程尝试去获取锁，如果获取成功则直接将它设置成独占锁模式。这也正好体现了非公平的原则，直接尝试获取锁，不成功再去和别的线程一样等待。

由于NonfairSync的逻辑包含了FairSync，因此接下来就从NonfairSync分析下去，在此之前看一下NonfairSync继承图:
![Gpl0sO.png](https://s1.ax1x.com/2020/03/26/Gpl0sO.png)

可以看到核心类Sync继承了AQS，NonfairSync和FairSync都继承了Sync。通过这一套继承机制实现了ReentrantLock的加锁和释放锁的逻辑，从这里就可以看出，AQS并没有具体的实现逻辑，它只是规定了管理线程等待和获取锁的机制（这是下面要讲解的重点），而ReentrantLock（类似的其他实现类像Semaphore等）实现了具体该何时获取锁，何时释放锁的逻辑。

### AQS源码分析
在讲解AQS之前，我先用一张图来描述一下它的运作原理。

![GuSKbD.png](https://s1.ax1x.com/2020/03/30/GuSKbD.png)

当一个线程没有获取到锁的时候，AQS会通过内部的Node类将Thread封装成一个节点，并且通过双向链表的形式在等待锁的过程中将节点加入到队列中去（这边使用CLH队列的一个变种，有兴趣的可以自己去了解一下什么是CLH队列）。

--------------------------------------------------------------------

接下来让我们从头开始探索一下**NonfairSync**完整的运作机制：

#### compareAndSetState

当案例中调用`lock.lock()`方法时，就会进入到`NonfairSync`的lock()方法中，第一步是通过`compareAndSetState(0, 1)`来获取锁，我们看一下具体的代码：

位于AQS类中：
```
    protected final boolean compareAndSetState(int expect, int update) {
        // See below for intrinsics setup to support this
        return unsafe.compareAndSwapInt(this, stateOffset, expect, update);
    }
```
可以看到，通过**Unsafe**类直接操作内存，实现CAS方式来更新state值，这边引出了**state**的这个属性，它是AQS的一个**volatile**类型的整数值，通过该值表示如果是0那么锁**没有被占用**，如果是1表示锁**已经被占用**。所以非公平锁先调用这个方法去**获取锁**看看是否能够成功，如果成功了就会进入`setExclusiveOwnerThread(Thread.currentThread());`方法，我们再来看一下这个方法的代码 

#### setExclusiveOwnerThread

```
    protected final void setExclusiveOwnerThread(Thread thread) {
        exclusiveOwnerThread = thread;
    }

```
这个方法存在于`AbstractOwnableSynchronizer`类中，通过上面的继承图可以看到它是AQS的父类。这个方法将exclusiveOwnerThread设置成了我们制定的线程，表示该线程正在占有锁。到这边第一个条件结束了，表示如果能够直接获取到锁，那么就将当前线程锁定，流程就结束了。

--------------------------------------------------------------------

#### acquire

如果上面的条件不成立，那么就会进入`acquire(1)`这个方法去获取锁，我们先看一下这个方法的代码：
```
    public final void acquire(int arg) {
        if (!tryAcquire(arg) &&
            acquireQueued(addWaiter(Node.EXCLUSIVE), arg))
            selfInterrupt();
    }
```
我们可以看到，主要涉及了两个条件，接下来就逐一对它们进行说明：

#### tryAcquire
这个方法在AQS中没有具体实现，这也是我们实现AQS需要自定义的方法。`NonfairSync`对应的实现如下：
```
     final boolean nonfairTryAcquire(int acquires) {
            final Thread current = Thread.currentThread(); 
      #1    int c = getState(); 
      #2    if (c == 0) {
               if (compareAndSetState(0, acquires)) {
                    setExclusiveOwnerThread(current);
                    return true;
                }
            }
      #3    else if (current == getExclusiveOwnerThread()) { 
                int nextc = c + acquires;
                if (nextc < 0) // overflow
                    throw new Error("Maximum lock count exceeded");
                setState(nextc);
                return true;
            }
            return false;
        }
```
这边我对主要步骤进行编号后再讲解：

首先是获取到当前线程对象，然后通过`getState()`方法获取到当前锁的状态(**#1**);<br/>
如果状态是0，表示锁还没有被占用，尝试直接获取锁(这一步其实就是第一个条件做的事),获取成功将当前线程设置成独占锁模式(**#2**）;<br/>
如果当前线程已经是独占锁模式了，表示不用再次获取锁，直接将state进行累加就行不过需要注意state溢出，最后设置好新的state即可（这个条件其实就表明了ReentrantLock是可重入锁）(**#3**);<br/>
如果上面的条件都不满足就返回false。

####  acquireQueued(addWaiter(Node.EXCLUSIVE), arg)
当直接获取锁不成功的时候，那么就要将线程加入到队列等待，对于如何将它加入队列就涉及到了`addWaiter(Node.EXCLUSIVE)`方法，接下来先看一下它的具体实现代码：
```
    private Node addWaiter(Node mode) {
        Node node = new Node(Thread.currentThread(), mode);
        // Try the fast path of enq; backup to full enq on failure
        Node pred = tail;
        if (pred != null) {
            node.prev = pred;
            if (compareAndSetTail(pred, node)) {
                pred.next = node;
                return node;
            }
        }
        enq(node);
        return node;
    }
```
可以看到首先会通过Node的初始化方法将Thread进行封装，并且设置了节点的类型，这边是`EXCLUSIVE`;<br/>
对于整个链表会有head节点和tail节点的概念，分别指向了链表的头和尾。新加入的节点首先通过tail节点来快速进行设置，tail不为空那么直接将node的prev指向该节点并且把新的tail节点指向node节点;<br/>
如果失败了就要执行`enq(node)`方法，重新进行入队操作，enq方法的具体实现如下：
```
    private Node enq(final Node node) {
        for (;;) {
            Node t = tail;
     #1     if (t == null) { // Must initialize
                if (compareAndSetHead(new Node()))
                    tail = head;
            } else {
                node.prev = t;
                if (compareAndSetTail(t, node)) {
                    t.next = node;
                    return t;
                }
            }
        }
    }
```
可以看见这是一个自旋的操作，首先会判断tail节点是否为空，如果是空的代表链表还没有节点，那么会初始化一个**空节点**并把tail和head节点都指向它(**#1**);<br/>
如果tail存在了的话那么还是通过上面的方式将节点设置在链表尾部；<br/>
到这边就可以看出，node的入队操作是依赖于旧的tail节点，通过新节点的前置节点来设置新的tail节点。
--------------------------------------------------------------------

#### acquireQueued
入队操作完成后，我们需要进一步探究Node在队列中是如何获取锁的，接下来我们看一下`acquireQueued`方法的具体实现：
```
    final boolean acquireQueued(final Node node, int arg) {
        boolean failed = true;
        try {
            boolean interrupted = false;
            for (;;) {
                final Node p = node.predecessor();
   #1           if (p == head && tryAcquire(arg)) {
                    setHead(node);
                    p.next = null; // help GC
                    failed = false;
                    return interrupted;
                }
   #2           if (shouldParkAfterFailedAcquire(p, node) &&
                    parkAndCheckInterrupt())
                    interrupted = true;
            }
        } finally {
            if (failed)
                cancelAcquire(node);
        }
    }
```
首先两个属性我们需要知道，`failed`表示获取锁是否失败，`interupted`表示线程是否被中断过;<br/>
此外这个方法是一个自旋的操作，主要包括两个步骤：<br/>
判断当前节点的前驱节点是否是**头结点**,如果是的话则再次尝试**获取锁**，获取成功就把新的头结点设置成当前节点，然后返回interrupted；(**#1**)<br/>
如果当前节点的前驱节点**不是头结点**或者**获取锁失败**的话，那么就需要判断是否需要将当前线程进行阻塞，这边涉及到两个判断条件`shouldParkAfterFailedAcquire`和`parkAndCheckInterrupt()`,下面依次进行讲解：

```
    private static boolean shouldParkAfterFailedAcquire(Node pred, Node node) {
        int ws = pred.waitStatus;
    #1  if (ws == Node.SIGNAL)
            /*
             * This node has already set status asking a release
             * to signal it, so it can safely park.
             */
            return true;
    #2  if (ws > 0) {
            /*
             * Predecessor was cancelled. Skip over predecessors and
             * indicate retry.
             */
            do {
                node.prev = pred = pred.prev;
            } while (pred.waitStatus > 0);
            pred.next = node;
    #3  } else {
            /*
             * waitStatus must be 0 or PROPAGATE.  Indicate that we
             * need a signal, but don't park yet.  Caller will need to
             * retry to make sure it cannot acquire before parking.
             */
            compareAndSetWaitStatus(pred, ws, Node.SIGNAL);
        }
        return false;
    }

```
这个方法主要针对当前节点，去判断截取节点的状态，如果waitStatus是`SIGNAL`直接返回true(**#1**);<br/>
如果waitStatus是大于0，表示前驱节点是取消状态，那么直接去获取之前的节点知道前驱节点的waiteStatus不是大于0(**#2**);<br/>
如果waitStatus是0或者3,那么直接将前驱节点的waitStatus设置为`SIGNAL`(**#3**)。<br/>
通过这种方式来决定当前节点的线程是否需要阻塞。

如果是true那么需要调用`parkAndCheckInterrupt`来实现阻塞，我们看一下具体实现:
```
    private final boolean parkAndCheckInterrupt() {
        LockSupport.park(this);
        return Thread.interrupted();
    }
```
可以看到使用了LockSupport类的park方法来进行线程的阻塞，该操作也依赖于Unsafe方法，直接操作具体的线程对应的permit执照来决定是否需要阻塞1。最后返回了线程的阻塞标识。

最后我们再回到acquire方法，线程阻塞过，需要执行`selfInterrupt`方法这是对中断机制的一个响应操作。需要注意的是，如果是
`lockInterruptibly`方法，就是在阻塞操作有不一样的地方，上面的是将interrupted设置为了true,但是lockInterruptibly方法是直接` throw new InterruptedException();`中断操作了，并且在finally中会去取消获取锁。

## 总结
最后做个总结，ReentrantLock的实现是依赖于AQS的，ReentrantLock层面只是重写了获取锁的逻辑上的代码。并且AQS是通过一个双向队列来存储包含Thread信息的Node节点，并且通过`state`变量来表示锁的获取状态。线程通过CAS方式设置锁的状态，通过节点的等待信息来决定是否去获取锁还是去阻塞线程。相信通过本篇文章大家也对AQS的实现原理有了一个直观的了解


##  参考
- [1] [JDK 8源码](https://www.oracle.com/java/technologies/javase-jdk8-doc-downloads.html)
- [2] [从ReentrantLock的实现看AQS的原理及应用](https://tech.meituan.com/2019/12/05/aqs-theory-and-apply.html)
- [3] [LockSupport原理剖析](https://segmentfault.com/a/1190000008420938)

> 转载请注明出处！