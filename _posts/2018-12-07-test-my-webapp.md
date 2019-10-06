---
layout: post
title:  "JUC源码详解一：语言级锁和同步组件实现的核心AQS"
categories: 源码
tags:  hexo JavaScript
author: 彭浩
---

* content
{:toc}

## 关键属性：

A. head，同步队列的头结点，Node节点 

B. tail，同步队列的尾节点，Node节点  

C. state，同步状态，整形变量，表示线程是否获取锁  

D. Node节点，同步队列封装线程的节点，属于AQS的内部类，Node中比较关键的属性：  
　　a. waitStatus，Node节点状态，也表示同步队列中线程的状态，并在Node类中定义了几个常量值表示waitStatus，分别为：SIGNAL（值为-1，表示后继节点需要被唤醒）、CANCELLED（值为1，表示节点中的线程被取消，将取出同步队列）、CONDITION（置为-2，表示节点中的线程等待在condition组件上）、PROPAGATE（值为-3，表示下一次共享的同步状态的获取将无条件传播）  
　　b. prev，同步队列中的当前节点的前驱节点，Node节点  
　　c. next，同步队列中的当前节点的后继节点，Node节点  
　　d. thread，同步队列中的当前节点所封装的线程，Thread  
　　e. nextWaiter，等待队列中当前节点的后继节点，若当前节点是共享的，那么该字段将是一个SHARED常量，也就是说节点类型（独占和共享，实际上使用节点来标识的）和等待队列中的后继节点共用同一个字段。




## 数据结构：

A. 同步状态使用一个volatile修饰的整形表示  

B. 同步队列使用双链表实现，同时记录一个head节点与一个tail节点来管理整个同步队列，其中Node为同步队列中的节点


## 方法剖析

实现：同步状态与同步队列，

（1）同步状态的修改、访问与设置AQS是写好了提供的，getState、setState、compareAndSwapState  

（2）而AQS开放了非阻塞的获取同步状态与释放同步状态的方法，使用者可以重写这些方法来实现自己的锁和同步组件，如重入锁和读写锁，以及CountDownLatch、CyclicBarrier等同步组件，都是通过重写这些开放的获取同步状态和释放同步状态的函数来实现的。  

（3）与此同时AQS也提供了多个模板方法用于实现获取同步状态失败后线程在同步队列中排队的获取同步状态。这是AQS写好的。
也就是说，我们想要自定义锁和同步组件，只需要继承AQS随后重写AQS提供的非阻塞的获取和释放同步状态的重写方法即可。


下面看几个核心的模板方法，

**（1）acquire方法**，独占式的获取同步状态，该方法对中断不敏感，即即使同步队列中的线程发生中断，也不会被移除去同步队列。逻辑为   
        
A. 调用重写方法tryAcquire判断当前线程是否能够获取同步状态，该方法返回的是布尔值，为true代表获取成功，为false则获取失败  

B. 获取同步状态失败的线程，也即当前线程将被封装为节点加入同步队列，之后在同步队列中自旋或者阻塞来等待获取同步状态，逻辑如下C-D  
    
C. 调用addWaitter方法封装当前线程为节点（节点类用于构建获取同步状态失败的线程构成一个双链表，并添加一些状态值），随后将节点加入同步队列的尾部，其逻辑首先尝试使用CAS操作快速添加至同步队列尾部，若失败则调用enq方法死循环的将节点添加至队列尾部（实现就是循环CAS）      

D. 调用acquireQueued方法安排当前节点在同步队列中自旋或者阻塞等待获取同步状态，逻辑为      
　　a. 自旋的获取同步状态，当前节点自旋的判断前驱节点是否为队列的首节点，若是，则尝试获取调用tryAcquire方法获取同步状态，只有这两个条件满足，才会真正获取到同步状态，此时设置当前节点为同步队列首节点，然后将原首节点断链，失败则继续自旋。可以看出在同步队列中的节点获取同步状态是排队的，也就是公平的获取同步状态，但对于新的竞争同步状态的线程则也可能竞争到同步状态，这又是不公平的，之后公平的重入锁就会改进。  
　　b. 阻塞等待获取同步状态，由于自旋会消耗CPU资源，为了避免过早的竞争同步状态，当前节点在设置其前驱节点为waitStatus为SIGNAL之后，就会进入阻塞状态，设置前驱节点状态的函数为shouldParkAfterFailedAcquire，而当前节点进入阻塞是通过parkAndcheckInterrupt方法，其内部调用LockSupport的park方法阻塞当前节点封装的线程。：

```java

//不可重写的方法，也即模板方法
public final void acquire(int arg) {
    //获取同步状态，其中tryAcquire是自定义获取同步状态重写的方法，表示非阻塞的获取同步状态
    if (!tryAcquire(arg) &&
        //获取同步状态失败，则（1）创建并封装线程为Exclusive的节点，（2）随后调用addWaiter方法将节点加入
        //同步队列，（3）调用acquireQueued方法使节点自旋获或者等待取同步状态
        acquireQueued(addWaiter(Node.EXCLUSIVE), arg))
        selfInterrupt();
}

private Node addWaiter(Node mode) {
    //调用Node的构造器，将当前线程封装为独占式的节点
    Node node = new Node(Thread.currentThread(), mode);
    // Try the fast path of enq; backup to full enq on failure
    //尝试一次快速的将当前节点添加至同步队列尾部，同时设置tail指针为新的尾节点
    Node pred = tail;
    if (pred != null) {
        node.prev = pred;
        if (compareAndSetTail(pred, node)) {
            pred.next = node;
            return node;
        }
    }
    //当快速添加新节点为到尾部失败，则调用enq方法死循环的设置当前节点到同步队列尾部
    enq(node);
    return node;
}

private Node enq(final Node node) {
    //死循环方式设置，也叫自旋
    for (;;) {
        //当同步队列为空
        Node t = tail;
        if (t == null) { // Must initialize
            //使用CAS操作线程安全的设置头节点为新节点
            if (compareAndSetHead(new Node()))
                tail = head;
        } else {
            //添加到同步队列尾部，即修改新节点的prev引用
            node.prev = t;
            //使用CAS操作线程安全的设置尾节点为新节点
            if (compareAndSetTail(t, node)) {
                t.next = node;
                return t;
            }
        }
    }
}

final boolean acquireQueued(final Node node, int arg) {
    //在节点在同步队列中阻塞等待或者自旋的获取同步状态时，
    //出现任何非正常退出的场景，该节点将会被取消获取同步状态
    boolean failed = true;
    try {
        //表示节点在阻塞等待或者自旋过程中是否发生了中断
        boolean interrupted = false;
        //自旋的获取同步状态
        for (;;) {
            //获取当前节点的前驱
            final Node p = node.predecessor();
            //若当前节点的前驱为head节点且获取同步状态成功，表示其前驱节点
            //释放了同步状态，此时当前节点将作为获取同步状态的节点成为首节点
            //进行操作一：前驱节点为首节点且当前节点获取同步状态成功
            if (p == head && tryAcquire(arg)) {
                //将当前节点设置为首节点
                setHead(node);
                //前驱节点断链
                p.next = null; // help GC
                failed = false;
                //返回中断标志位，并不影响线程执行
                return interrupted;
            }
            //进行操作二：设置前驱节点等待状态并使当前节点阻塞等待其前驱节点的唤醒
            if (shouldParkAfterFailedAcquire(p, node) &&
                parkAndCheckInterrupt())
                interrupted = true;
        }
    } finally {
        if (failed)
            cancelAcquire(node);
    }
}

private static boolean shouldParkAfterFailedAcquire(Node pred, Node node) {
    int ws = pred.waitStatus;
    //前驱节点已经为SIGNAL状态，将直接返回true
    if (ws == Node.SIGNAL)
        /*
         * This node has already set status asking a release
         * to signal it, so it can safely park.
         */
        return true;
    //若前驱节点等待状态大于0，即为CANCELLED状态，则寻找当前节点中有效的前驱节点    
    if (ws > 0) {
        /*
         * Predecessor was cancelled. Skip over predecessors and
         * indicate retry.
         */
        do {
            node.prev = pred = pred.prev;
        } while (pred.waitStatus > 0);
        pred.next = node;
    } else {
        /*
         * waitStatus must be 0 or PROPAGATE.  Indicate that we
         * need a signal, but don't park yet.  Caller will need to
         * retry to make sure it cannot acquire before parking.
         */
        //当前驱节点的等待状态小于等于0，且不为SIGNAL时，将使用CAS操作将前去节点的waitStatus设置为SIGNAL 
        compareAndSetWaitStatus(pred, ws, Node.SIGNAL);
    }
    return false;
}

private final boolean parkAndCheckInterrupt() {
    LockSupport.park(this);
    //被唤醒或者中断后，将返回中断标志位
    return Thread.interrupted();
}

```

**（2）release方法**，独占式的释放同步状态，即成功获取同步状态的线程将唤醒其后继节点，逻辑为  

A. 调用unparkSuccessor方法唤醒首节点的后继节点，该方法的逻辑为B-C 
 
B. 使用CAS将首节点的waitStatus置为0  

C. 唤醒首节点的有效的后继节点（因为某些节点为CANCELLED状态）   

```java

public final boolean release(int arg) {
    if (tryRelease(arg)) {
        Node h = head;
        if (h != null && h.waitStatus != 0)
            unparkSuccessor(h);
        return true;
    }
    return false;
}

private void unparkSuccessor(Node node) {
    /*
     * If status is negative (i.e., possibly needing signal) try
     * to clear in anticipation of signalling.  It is OK if this
     * fails or if status is changed by waiting thread.
     */
    int ws = node.waitStatus;
    //使用CAS操作清空当前节点的waitStatus为初始状态，即0
    if (ws < 0)
        compareAndSetWaitStatus(node, ws, 0);

    /*
     * Thread to unpark is held in successor, which is normally
     * just the next node.  But if cancelled or apparently null,
     * traverse backwards from tail to find the actual
     * non-cancelled successor.
     */
    //依然要处理当前节点的后继节点为CANCELLED状态时需要查找当前节点的真实后继节点 
    Node s = node.next;
    if (s == null || s.waitStatus > 0) {
        s = null;
        //从尾节点tail开始查找真实的下一个有效节点
        for (Node t = tail; t != null && t != node; t = t.prev)
            if (t.waitStatus <= 0)
                s = t;
    }
    if (s != null)
        //唤醒真实的同步队列的下一个节点
        LockSupport.unpark(s.thread);
}
```


