---
layout:     post
title:      "Java同步机制(五)-Lock接口"
description: "Lock接口"
showonlyimage: false
author:     "zhouyang"
date:     2021-06-23
published: true
tags:
    - Lock接口
    - AQS
categories: [Java同步机制]
mermaid: true
---

# Lock接口

`Lock`接口是`J.U.C`中的一个接口，为我们提供和`Synchronized`相似的并发控制功能，但是使用起来比`Synchronized`更加灵活。
下面我们通过接口中定义的方法来分析一下

<!--more-->

```Java

public interface Lock {

    //能够保证获得锁
    //如果锁再当前不是空闲状态，则会挂起当前线程
    void lock();

    //解锁
    void unlock();

    //保证获得锁
    //但是在获取锁的过程中，能够响应线程中断，并且抛出中断异常
    void lockInterruptibly() throws InterruptedException;

    //尝试获取锁，不能保证获取成功
    //如果获取失败则返回false
    //如果获取成功则返回true
    boolean tryLock();

    //在一段时间内尝试获取锁
    //在这段时间内如果锁不是空闲状态则会挂起当前线程
    //在获取锁的过程中可以相应中断，抛出终端异常
    boolean tryLock(long time, TimeUnit unit) throws InterruptedException;


    Condition newCondition();

```

我们看到，接口中定义了`lock`和`unlock`方法，我们可以理解为分别对应了`Synchronized`的`monitorenter`和`monitorexit`。
这两个方法可以说是体现了`Lock`接口和`Synchronized`相似的部分。

那么下面的几个方法就体现了`Lock`接口和`Synchronized`的区别，和更加灵活的地方。

## lockInterruptibly方法
这个方法就体现了`Lock`接口和`Synchronized`的一个主要区别，就是可以响应中断。

- 在使用`Synchronized`获取锁，在等待的过程中，如果我们想停止获取锁，是没有办法实现的。

- 而在使用`Lock`接口的`lockInterruptibly`方法获取锁的时候，我们可以通过调用线程的`interrupt`方法，停止线程获取锁。

## tryLock方法
`tryLock`就是`Lock`接口和`Synchronized`的另一个主要区别了，可以尝试获取锁，从而避免获取锁失败的时候陷入阻塞状态。

- 在使用`Synchronized`,是没有尝试获取锁这个说法的，你要么获取到锁，要么获取失败，陷入阻塞状态等待获取锁。
- 使用`Lock`接口的`tryLock()`方法,如果获取成功则返回try,如果获取失败则返回false，并不会阻塞线程。

下面另一个版本的tryLock方法，区别不大，只不过是给定了一个尝试获取的时间，并且在这段时间内会响应中断。

## newCondition方法
`newCondition`方法，会返回一个`Condition`对象。对象中的`await`方法，和`signal`方法，在功能上大致和`wait`,`notify`相同。


# Lock接口和Synchronized的区别

## Lock更加灵活
我们都说Lock比Synchronized更灵活。但是具体怎么灵活呢，通过上面Lock接口定义的几个方法，我理解主要在下面几个方面。

1. Lock提供了尝试获取方法tryLock()
    - Lock提供了尝试获取方法，既拿不到锁立即返回
    - 相对应的Synchronized则没有，一旦使用，则必须获取成功才能继续往下执行

2. Lock提供了可响应中断的获取锁方法
    - Lock提供了可中断的获取锁方法，`lockInterruptibly()`和`tryLock(long time,TimeUnit unit)`。既线程在获取Lock的挂起过程中是可以响应中断的，开发者可以通过这种方法来终止获取锁。
    - Synchronized则不可以，线程在Synchronized的挂起过程中，不可响应中断操作，一旦开始，就无法停止。

3. Condition机制
    - Condition还需要在好好理解一下，还是不太懂  `todo`

## 性能与便利性
1. 性能
    - 我自己理解，性能方面已经不是Lock和Synchronized的主要区别了，我们知道Lock主要是依赖AQS来实现的，但是熟悉Synchronized应该也知道，Synchronized优化后，性能强了不少。并且Synchronized的重量级锁实现和AQS有很多类似的地方。

2. 方便性
    - 在使用便利性上Synchronized还有有一些优势的，因为Synchronized可以省掉一步手动释放锁的操作。JVM帮我们完成了释放的操作。
    - 另一点，除了开发上的便利性，在调试上Synchronized也是有一定优势的，因为毕竟是JVM原生提供的，所以在使用JVM的命令,比如jstack调试的时候，能够方便的看到Synchronized相关的信息。而Lock就不可以了。


# Lock合AQS的关系
Lock接口是定义了一种锁的行为，并没有规定具体的实现方式。
我们常见的Lock接口的实现类`ReentranLock`，就是使用的AQS，实现的Lock接口中定义的各种行为。

所以说`Lock`接口是，定义了锁的行为，AQS是其中的一种实现方式。
