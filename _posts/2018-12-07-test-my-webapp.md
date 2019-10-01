---
layout: post
title:  "JUC源码详解一：语言级锁和同步组件实现的核心AQS"
categories: yuanma
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

**acquire方法**，独占式的获取同步状态，该方法对中断不敏感，即即使同步队列中的线程发生中断，也不会被移除去同步队列。逻辑为   

    A. 调用重写方法tryAcquire判断当前线程是否能够获取同步状态，该方法返回的是布尔值，为true代表获取成功，为false则获取失败  
    
    B. 获取同步状态失败的线程，也即当前线程将被封装为节点加入同步队列，之后在同步队列中自旋或者阻塞来等待获取同步状态，逻辑如下C-D  
    
    C. 调用addWaitter方法封装当前线程为节点（节点类用于构建获取同步状态失败的线程构成一个双链表，并添加一些状态值），随后将节点加入同步队列的尾部，其逻辑首先尝试使用CAS操作快速添加至同步队列尾部，若失败则调用enq方法死循环的将节点添加至队列尾部（实现就是循环CAS）  
    
    D. 调用acquireQueued方法安排当前节点在同步队列中自旋或者阻塞等待获取同步状态，逻辑  
    
        a. 自旋的获取同步状态，当前节点自旋的判断前驱节点是否为队列的首节点，若是，则尝试获取调用tryAcquire方法获取同步状态，只有这两个条件满足，才会真正获取到同步状态，此时设置当前节点为同步队列首节点，然后将原首节点断链，失败则继续自旋。可以看出在同步队列中的节点获取同步状态是排队的，也就是公平的获取同步状态，但对于新的竞争同步状态的线程则也可能竞争到同步状态，这又是不公平的，之后公平的重入锁就会改进。
        
        b. 阻塞等待获取同步状态，由于自旋会消耗CPU资源，为了避免过早的竞争同步状态，当前节点在设置其前驱节点为waitStatus为SIGNAL之后，就会进入阻塞状态，设置前驱节点状态的函数为shouldParkAfterFailedAcquire，而当前节点进入阻塞是通过parkAndcheckInterrupt方法，其内部调用LockSupport的park方法阻塞当前节点封装的线程。：

```js

spencer@spencer-it1 MINGW64 /f/myself/643435675.github.io (master)
$ gem install jekyll
Successfully installed jekyll-3.6.2
Parsing documentation for jekyll-3.6.2
Done installing documentation for jekyll after 2 seconds
1 gem installed

spencer@spencer-it1 MINGW64 /f/myself/643435675.github.io (master)
$ jekyll s
Configuration file: F:/myself/643435675.github.io/_config.yml
            Source: F:/myself/643435675.github.io
       Destination: F:/myself/643435675.github.io/_site
 Incremental build: disabled. Enable with --incremental
      Generating...
                    done in 5.499 seconds.
  Please add the following to your Gemfile to avoid polling for changes:
    gem 'wdm', '>= 0.1.0' if Gem.win_platform?
 Auto-regeneration: enabled for 'F:/myself/643435675.github.io'
    Server address: http://127.0.0.1:4001/
  Server running... press ctrl-c to stop.


spencer@spencer-it1 MINGW64 /f/myself/643435675.github.io (master)
$ jekyll serve
Configuration file: F:/myself/643435675.github.io/_config.yml
            Source: F:/myself/643435675.github.io
       Destination: F:/myself/643435675.github.io/_site
 Incremental build: disabled. Enable with --incremental
      Generating...
                    done in 2.859 seconds.
  Please add the following to your Gemfile to avoid polling for changes:
    gem 'wdm', '>= 0.1.0' if Gem.win_platform?
 Auto-regeneration: enabled for 'F:/myself/643435675.github.io'
    Server address: http://127.0.0.1:4001/
  Server running... press ctrl-c to stop.

```

主要进行了以上操作：
cd 643435675.github.io  文件夹下

执行命令：

`gem install jekyll `

下一步：
`jekyll s`

接下来：
`jekyll serve`

然后这样就可以在浏览器中输入：http://127.0.0.1:4001/ 进行刚刚文章的访问了。


补充：

因为首页与详情页显示某篇文章的内容是不一样的，在首页中进行文章的裁剪显示：
需要进行加入四个空行：
```md
## 目的：

写这篇文章的目的主要是为了测试在本地进行md文件的编写是否能使用hexo进行html生成，然后上传到github上，通过访问https://day21.top 这个网站查看能否看到最新的文章




## 操作步骤：
```

不进行添加的话，首页会把整篇文章完全展示出来，添加了四个空行之后就只会显示部分了，当然了，这篇文章在首页是显示的部分，也就是**目的下的内容**


