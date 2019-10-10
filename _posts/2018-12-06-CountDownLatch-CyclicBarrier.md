---
layout: post
title:  "JUC源码详解三：工具类--同步组件"
categories: 源码
tags:  JUC
author: 彭浩
---

* content
{:toc}

# 同步组件（使用+实现）

（1）CountDownLatch，用法很简单，

    A.使用，在需要等待的线程中调用CountDownLatch的countDown方法减少计数器的值，在主线程中调用CountDownLatch的await方法阻塞主线程，直到计数器的值为0，主线程才继续向下执行。
    B. 实现，实现就是基于AQS（半共享锁的实现），那么同样是重写需要重写的方法，根据CountDownLatch的特性，会传入一个计数器的值，那么该计数器就是同步状态的值，而主线程调用await方法会等待同步状态值为0才会继续执行，也就是说await方法执行的就是获取同步状态的逻辑，而其他线程调用countDown方法减少计数器即同步状态的值，很典型的就是释放同步状态的逻辑（类似重入锁的释放，当释放到0时，主线程就可以获取了），由于多个线程可以同时释放同步状态，所以该CountDownLatch的实现就是典型的共享式锁的获取与释放，所以就很简单了，看如下源码，
注意：从源码可以看出，不止是一个线程可以调用CountDownLatch的await方法，多个线程可以调用，那么当同步状态的值减为0时，所有这些线程都将被传播唤醒，从而继续执行。同样可以看出tryAcquireShared的实现，直接根据同步状态的值来返回正负值，也就是说同步状态并不会增加，只会由tryReleaseShared释放而减少，所以CountDownLatch是一次性的。支持中断支持超时
```java
Sync(int count) {
    setState(count);
}

protected int tryAcquireShared(int acquires) {
    return (getState() == 0) ? 1 : -1;
}

protected boolean tryReleaseShared(int releases) {
    // Decrement count; signal when transition to zero
    for (;;) {
        int c = getState();
        if (c == 0)
            return false;
        int nextc = c-1;
        if (compareAndSetState(c, nextc))
            return nextc == 0;
    }
}

//响应中断
public void await() throws InterruptedException {
    sync.acquireSharedInterruptibly(1);
}

public boolean await(long timeout, TimeUnit unit)
    throws InterruptedException {
    return sync.tryAcquireSharedNanos(1, unit.toNanos(timeout));
}

public void countDown() {
    sync.releaseShared(1);
}
```

（2）CyclicBarrier，同步屏障，当一定数目的线程都运行到某个点时才能继续执行。

    A. 使用，该同步组件控制多个平行组件运行到某个点时都才会一起执行，该组件还可以实现，当所有线程都到达同步点后，可以根据入参传入的Runnable对象优先执行该Runnable对象，随后才会执行其他所有的线程，初始化传入一个计数器，在需要控制的线程中调用CyclicBarrier的await方法即可实现。
    B. 实现，该组件没有通过继承AQS实现，但是内置了一个重入锁和等待通知组件Condition来实现等待通知机制，想想也是，多个线程运行到调用await的位置，将直接进入阻塞等待状态，主动放弃CPU和锁资源，从而加入等待队列，直到在最后一个调用await方法的内部，当前条件计数器为0，此时则由这最后一个线程充当唤醒线程，所以在await方法内部，应该同时内置了Condition的await方法与signalAll方法，而循环条件就是初始时传入的计数器的值是否已经为0，若不为0，则每次在调用await方法时，就会减一，CyclicBarrier的await方法逻辑如下：
        a. 加锁，减少count的值，也即减少计数器的值，每次调用await方法都会减少
        b. 循环条件：若count不为0，此时调用Condition的await方法阻塞等待，加入等待队列
        c. 循环条件：若count为0，此时调用Condition的signalAll方法唤醒所有等待在Condition上的线程，在此之前，要判断构造器时传入的对象barrierAction是否为null，不为null则需要先执行高Runnable。
        d. 释放锁
注意：CyclicBarrier提供了重置计数器的方法，可以重复使用，提供了getWaiter方法，即获取等待在Condition上的线程数。支持中断、支持超时


（3）Semaphore，信号量，控制一次性有多少个线程可以同时执行，也可以换个角度，控制多个线程访问许可数

    A. 使用，典型的共享锁的实现，重入锁是独占锁的实现，读写锁是独占和共享一起的实现，而semaphore就是共享锁的实现，其中的许可证是构造函数注入的同步状态，所以在获取许可证前调用acquire方法（注意该方法不是AQS的acquire方法，而是Semaphore定义的，里面调用的是acquireSharedInterruptibly，重写的是tryAcquireShared方法）
    B. 实现，在使用中说了，就是共享锁的实现，只是同步状态的值是构造函数注入的，重写的tryAcquireShared方法，直接就是死循环的CAS操作同步状态减一即可（读写的实现就是不行就返回-1，这里因为初始有同步状态传入，直接减到负就不能获取了），当减到负时，表明同步状态已经获取完，此时就可以返回负值，使得其他线程不再可以获取，而tryReleaseShared方法就是将同步状态加一即可，注意超过整形最大值将抛出异常。


（4）Exchanger，不咋用，只知道使用exchange可以实现线程间数据交换，类似管道的实现，需要相互配对。可用于校对工作。

（5）Condition组件，