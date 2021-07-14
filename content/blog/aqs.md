---
layout:     post
title:      "Java同步机制(六)- AQS"
description: "AbstractQueuedSynchronizer"
showonlyimage: false
author:     "zhouyang"
date:     2021-06-24
published: true
tags:
    - AQS
    - ReentranLock
categories: [ Tech , AQS]
mermaid: true
---

终于来到了重头戏-`AQS`,AQS可以说是整个`J.U.C`的核心，整个工具包中的大部分同步工具都是借助于AQS来实现的。接下来我们将通过`ReentranLock`的实现来了解AQS的原理

# AQS结构

## 同步状态

首先在AQS中维护了一个名叫state的字段，是由volatile修饰的，它就是所谓的同步状态：
```Java
private volatile int state;
```

并且提供了几个访问这个字段的方法：

|方法名称|描述|
|:----|:----|
|protected final int getState()|获取state的值|
|protected final void setState(int newState)|设置state的值|
|protected final boolean compareAndSetState(int expect, int update)|使用CAS方式更新state的值|

可以看到这几个方法都是final修饰的，说明子类中无法重写它们。另外它们都是protected修饰的，说明只能在子类中使用这些方法。


## 同步队列
AQS使用一个Volatile的int类型的成员变量`state`来表示同步状态，通过内置的FIFO同步队列来完成资源获取的排队工作。

![](/img/clhqueue.webp)

<span style="color:red;">有一点值得注意，就是这里的头结点是一个虚节点，它的thread为空，头结点的存在更多意义上是为了编程方便。当然为了方便理解，我们可以认为头结点就是获取了锁的线程的节点，但是thread被清空了</span>

当线程获取到锁的时候，会把线程所在的节点设置为头结点，设置为头结点后，会把不需要的属性设置为null。

```
/**
 * Sets head of queue to be node, thus dequeuing. Called only by
 * acquire methods.  Also nulls out unused fields for sake of GC
 * and to suppress unnecessary signals and traversals.
 *
 * @param node the node
 */
private void setHead(Node node) {
    head = node;
    node.thread = null;
    node.prev = null;
}
```


AQS中定义一个头节点引用，一个尾节点引用：
```java
private transient volatile Node head; //执行队列的头结点
private transient volatile Node tail; //执行队列的尾节点
```

上面图中的每个线程，都对应着一个Node节点，Node是AQS中的一个内部类，下面解释一下几个方法和属性值的含义：

|方法和属性|含义|
|:----:|:----:|
|waitStatus|当前节点在队列中的状态|
|thread|表示处于该节点的线程|
|prev|指向前一个节点的指针|
|next|指向后一个节点的指针|
|predecessor|返回前一个节点，没有的话抛出npe|
|nextWaiter|指向下一个处于CONDITION状态的节点(暂不多做介绍)|



| 枚举   |   含义|
|:----:|:----:|
|0|Node初始化的默认值|
|CANCELLED|为1，表示节点已经取消获取锁|
|CONDITION|为-2，表示节点在等待队列中，节点线程等待唤醒|
|PROPAGATE|为-3，当前线程处在SHARED情况下，该字段才会使用|
|SIGNAL|为-1，表示该节点的继任者(下一个节点)，已经或者将要被block|

这里把java源码中的注释也贴出来，方便大家理解

```java
/**
  * Status field, taking on only the values:
  *   SIGNAL:     The successor of this node is (or will soon be)
  *               blocked (via park), so the current node must
  *               unpark its successor when it releases or
  *               cancels. To avoid races, acquire methods must
  *               first indicate they need a signal,
  *               then retry the atomic acquire, and then,
  *               on failure, block.
  *   CANCELLED:  This node is cancelled due to timeout or interrupt.
  *               Nodes never leave this state. In particular,
  *               a thread with cancelled node never again blocks.
  *   CONDITION:  This node is currently on a condition queue.
  *               It will not be used as a sync queue node
  *               until transferred, at which time the status
  *               will be set to 0. (Use of this value here has
  *               nothing to do with the other uses of the
  *               field, but simplifies mechanics.)
  *   PROPAGATE:  A releaseShared should be propagated to other
  *               nodes. This is set (for head node only) in
  *               doReleaseShared to ensure propagation
  *               continues, even if other operations have
  *               since intervened.
  *   0:          None of the above
  *
  * The values are arranged numerically to simplify use.
  * Non-negative values mean that a node doesn't need to
  * signal. So, most code doesn't need to check for particular
  * values, just for sign.
  *
  * The field is initialized to 0 for normal sync nodes, and
  * CONDITION for condition nodes.  It is modified using CAS
  * (or when possible, unconditional volatile writes).
  */
  volatile int waitStatus;
```

todo 后面我们在补充一下


## 独占和共享模式
在一些线程协调的场景中，一个线程在进行某些操作的时候其他的线程都不能执行该操作，比如持有锁时的操作，在同一时刻只能有一个线程持有锁，我们把这种情景称为`独占模式`；
在另一些线程协调的场景中，可以同时允许多个线程同时进行某种操作，我们把这种情景称为`共享模式`。
我们可以通过修改AQS中state字段代表的同步状态来实现多线程的独占模式或者共享模式。

- 独占模式
在独占模式下，我们设置state为0，线程要进行独占操作的时候，需要使用CAS操作，把0修改为1，我们把这个过程称为`尝试获取同步状态`。然后在执行完成后，同样再通过CAS操作修改回来,把这个操作成为`释放同步状态`。这洋就能保证任何时候，都只能有一个线程独占。

- 共享模式
在共享模式下，我们设置state为能支持共享的最大线程数，比如10。线程在执行共享操作时，每次都判断state是否大于0，如果大于0，通过CAS把state的值减1,我们把这个过程成为`尝试获取共享同步状态`。然后每个线程再执行完之后，再通过CAS把state的值+1,我们把这个操作成为`释放共享同步状态`。

在后面的代码分析中会详细讲解独占模式和共享模式。

对于上面提到的`获取共享状态`和`释放共享状态`以及`尝试获取共享同步状态`和`释放共享同步状态`，AQS中定义了几个方法

|方法名称|描述|
|:----|:----|
|protected boolean tryAcquire(int arg)    |独占式的获取同步状态，获取成功返回true，否则false|
|protected boolean tryRelease(int arg)    |独占式的释放同步状态，释放成功返回true，否则false|
|protected int tryAcquireShared(int arg)  |共享式的获取同步状态，获取成功返回true，否则false|
|protected boolean tryReleaseShared(int arg)|共享式的释放同步状态，释放成功返回true，否则false|
|protected boolean isHeldExclusively()    |在独占模式下，如果当前线程已经获取到同步状态，则返回 true；其他情况则返回 false|

这几个方法，AQS都没有实现，而是要求子类去实现。如果我们自定义的同步工具需要在独占模式下工作，那么我们就重写tryAcquire、tryRelease和isHeldExclusively方法。

在了解了上面这些内容之后，我们通过`ReentranLock`来详细了解一下AQS的独占模式是如何工作的。

# ReentranLock
先来看一下`ReentranLock`的代码结构：

```Java
public class ReentrantLock implements Lock, java.io.Serializable {
    /** Synchronizer providing all implementation mechanics */
    private final Sync sync;

    abstract static class Sync extends AbstractQueuedSynchronizer{
      ...
    }

    static final class NonfairSync extends Sync {
      ...
    }

    static final class FairSync extends Sync {
      ...
    }

    public ReentrantLock() { sync = new NonfairSync(); }

    public ReentrantLock(boolean fair) {
        sync = fair ? new FairSync() : new NonfairSync();
    }

    public void lock() { sync.lock();}

    public void lockInterruptibly() throws InterruptedException {
        sync.acquireInterruptibly(1);
    }

    public boolean tryLock() {
        return sync.nonfairTryAcquire(1);
    }

    public boolean tryLock(long timeout, TimeUnit unit)
            throws InterruptedException {
        return sync.tryAcquireNanos(1, unit.toNanos(timeout));
    }

    public void unlock() {
        sync.release(1);
    }

    public Condition newCondition() {
        return sync.newCondition();
    }
}
```

上面我们把`ReentranLock`的主要结构，罗列出来。、

- 首先定义了一个Sync属性sync
- 接下来是Sync的内部类，Sync继承自`AbstractQueuedSynchronizer`
- 然后是两个Sync的子类，分别是非公平锁和公平锁的实现
- 接下来是两个构造方法，可以看到，默认是使用的非公平锁
- 然后就是实现自Lock接口的方法，可以看到方法都是最终调用了sync，也就是我们前面说的，`ReentranLock`借助于AQS实现了`Lock`。

我们看到`ReentranLock`的几个方法中，都是调用的Sync的相关方法，下面我们就来看一下Sync和它的两个子类`NonfairSync`与`FairSync`:

## Sync内部类

`Sync`继承自`AbstractQueuedSynchronizer`,并重写了`tryRelease`和`isHeldExclusively`方法。独占模式要求的另一个方法`tryAcquire`，由他的子类`NonfairSync`和`FairSync`分别实现。

看到这里，我们可以知道什么，<span style="color:red;">那就是非公平锁和公平锁只在获取锁上有区别，在释放锁的上没有任何区别</span>

```java
abstract static class Sync extends AbstractQueuedSynchronizer {

    abstract void lock();

    /**
     * Performs non-fair tryLock.  tryAcquire is implemented in
     * subclasses, but both need nonfair try for trylock method.
     */
    //注释说到， tryAcquire在子类中实现，但是tryLock是使用nofair的，所以放到了父类中
    final boolean nonfairTryAcquire(int acquires) {
        final Thread current = Thread.currentThread();
        int c = getState();
        if (c == 0) {
            //如果当前没有线程持有，则直接尝试获取一次，不管后面后又没其他线程再等待
            if (compareAndSetState(0, acquires)) {
                setExclusiveOwnerThread(current);
                return true;
            }
        }
        //判断是否为锁重入
        else if (current == getExclusiveOwnerThread()) {
            //记录重入
            int nextc = c + acquires;
            //判断int overflow的情况，所以最大的重入次数为int的最大值
            if (nextc < 0) // overflow
                throw new Error("Maximum lock count exceeded");
            setState(nextc);
            return true;
        }
        return false;
    }

    //尝试释放锁
    protected final boolean tryRelease(int releases) {

        int c = getState() - releases;
        if (Thread.currentThread() != getExclusiveOwnerThread())
            throw new IllegalMonitorStateException();
        boolean free = false;
        //如通c == 0，表示所有的锁都释放完了
        //则设置持有锁线程为null
        //返回释放锁成功
        if (c == 0) {
            free = true;
            setExclusiveOwnerThread(null);
        }
        //如果c != 0 则表示发生了锁重入，只是里层释放了锁
        //返回没有完全释放锁
        setState(c);
        return free;
    }

    protected final boolean isHeldExclusively() {
        // While we must in general read state before owner,
        // we don't need to do so to check if current thread is owner
        return getExclusiveOwnerThread() == Thread.currentThread();
    }
    // 省略掉一些不太重要的方法。。。。
}

```
下面在看`NonfairSync`这个子类

```java
/**
 * Sync object for non-fair locks
 */
static final class NonfairSync extends Sync {
    private static final long serialVersionUID = 7316153563782823691L;

    /**
     * performs lock.  Try immediate barge, backing up to normal
     * acquire on failure.
     */
    final void lock() {
        //非公平锁上来就先尝试获取一次,就是这里体现了不公平，
        if (compareAndSetState(0, 1))
            setExclusiveOwnerThread(Thread.currentThread());
        else
            //如果没有获取成功,通过acquire方法，加入队列中等待
            acquire(1);
    }

    protected final boolean tryAcquire(int acquires) {
        return nonfairTryAcquire(acquires);
    }
}

```

在看一下`FairSync`这个子类
```Java
/**
 * Sync object for fair locks
 */
static final class FairSync extends Sync {
    private static final long serialVersionUID = -3000897897090466540L;

    final void lock() {
        acquire(1);
    }

    /**
     * Fair version of tryAcquire.  Don't grant access unless
     * recursive call or no waiters or is first.
     */
    //公平锁的tryAcquire和非公平锁的差别就是，非公平锁会直接尝试一次CAS获取锁
    //而公平锁则会先判断一下等待队列是不是为空。
    //如果都尝试获取失败，之后的处理逻辑都是一样的
    protected final boolean tryAcquire(int acquires) {
        final Thread current = Thread.currentThread();
        int c = getState();
        if (c == 0) {
            //如果队列为空，或者当前线程已经处于队列的最前，则尝试一次通过CAS获取锁
            //否则不获取
            if (!hasQueuedPredecessors() &&
                compareAndSetState(0, acquires)) {
                setExclusiveOwnerThread(current);
                return true;
            }
        }
        //判断重入的逻辑
        else if (current == getExclusiveOwnerThread()) {
            int nextc = c + acquires;
            if (nextc < 0)
                throw new Error("Maximum lock count exceeded");
            setState(nextc);
            return true;
        }
        //获取锁失败，并且不是重入，则tryAcquire失败
        return false;
    }
}

```

### 公平锁和非公平锁在获取锁时的区别
我们看到`NonfairSync`和`FairSync`都是只有两个方法，`tryAcquire`和`lock`方法。其中`tryAcquire`方法是实现自AQS。
那么公平锁和非公平锁的区别就体现在，这两个类，对着两个方法的实现上。

我们先看`tryAcquire`方法的区别。
- `NonfairSync`是直接调用的父类的`nonfairTryAcquire`方法，它呢不会管当前有没有其他线程再等待，只要当前没有线程持有，就尝试获取一次
- `FairSync`是会先判断是否有其他线程再等待，只有没有线程等待或者它本身就是最前面的节点的时候，才会尝试获取。

我们再看`lock`方法
- `NonfairSync`,上来直接使用CAS，尝试获取一次，获取不到则调用AQS的`acquire`方法，在`acquire`方法中会调用`NonfairSync`的`tryAcquire`，如果仍然失败，加入到等待队列中。
- `FairSync`,直接调用父类AQS的`acquire`方法，在`acquire`方法中会调用`FairSync`的`tryAcquire`)，如果失败，加入到等待队列中。

我们看到公平锁和非公平锁的区别就在，非公平锁不会管现在又没有线程在等待，而是直接尝试获取。

### nonfairTryAcquire方法为何定义在父类
看这个方法名称`nonfairTryAcquire`,感觉应该写在子类`NonfairSync`中会比较合适,这里会写到父类中呢？方法的注释中写到

```java
/**
 * Performs non-fair tryLock.  tryAcquire is implemented in
 * subclasses, but both need nonfair try for trylock method.
 */

```
`执行非公平的tryLock。 tryAcquire在子类中实现。但两者都需要非公平的trylock方法。`
意思是在调用`Lock.tryLock()`方法的时候，都会用到`nonfairTryAcquire`,所以需要写到父类中。

那么为什么`tryLock`都要用非公平的`nonfairTryAcquire`呢？

我们再来看一下`tryLock`方法的注释

![](/img/trylock.png)

注释中写到，
- 在当前没有线程持有这个锁的时候，那么`tryLock`方法就可以立即抢占这个锁，即使使用的公平锁的策略。这种行为在有些情况下能够带来更好的性能，尽管它会破坏公平性。
- 如果不想破坏公平性，那么可以使用`tryLock(0, TimeUnit.SECONDS)`方法。

### 公平锁和非公平锁的性能对比

上面我们说到`tryLock`方法一直使用非公平的方式尝试获取锁，是因为非公平锁的性能要更好。那下面我们来分析一下原因和验证一下

- 因为公平锁在获取锁时，永远是等待时间最长的线程获取到锁，这样当线程T1释放锁以后，如果还想继续再获取锁，它也得去同步队列尾部排队，这样就会频繁的发生线程的上下文切换，当线程越多，对CPU的损耗就会越严重。
- 而且在唤醒队首的线程后，线程不会立即执行，而是需要等待CPU分配时间片，才能获取到锁。
- 而非公平锁，是有机会跳过等待队列，和等待CPU分配时间片的这个情况的，所以性能会好一些。
- 非公平锁性能虽然优于公平锁，但是会存在导致线程饥饿的情况。在最坏的情况下，可能存在某个线程一直获取不到锁。不过相比性能而言，饥饿问题可以暂时忽略。

下面我们通过代码来验证一下：

```java
public class FairVsNonFairLock {

    // 公平锁
    private static Lock fairLock = new ReentrantLock(true);
    // 非公平锁
    private static Lock nonFairLock = new ReentrantLock(false);
    // 计数器
    private static int fairCount = 0;
    // 计数器
    private static int nonFairCount = 0;

    private static int threadCount = 1;

    public static void main(String[] args) throws InterruptedException {
        System.out.println("公平锁耗时:   " + testFairLock(threadCount) + " ms");
        System.out.println("非公平锁耗时: " + testNonFairLock(threadCount) + " ms");
    }

    public static long testFairLock(int threadNum) throws InterruptedException {
        CountDownLatch countDownLatch = new CountDownLatch(threadNum);
        // 创建threadNum个线程，让其以公平锁的方式，对fairCount进行自增操作
        List<Thread> fairList = new ArrayList<>();
        for (int i = 0; i < threadNum; i++) {
            fairList.add(new Thread(() -> {
                for (int j = 0; j < 10000; j++) {
                    fairLock.lock();
                    fairCount++;
                    fairLock.unlock();
                }
                countDownLatch.countDown();
            }));
        }
        long startTime = System.currentTimeMillis();
        for (Thread thread : fairList) {
            thread.start();
        }
        // 让所有线程执行完
        countDownLatch.await();
        return System.currentTimeMillis() - startTime;
    }

    public static long testNonFairLock(int threadNum) throws InterruptedException {
        CountDownLatch countDownLatch = new CountDownLatch(threadNum);
        // 创建threadNum个线程，让其以非公平锁的方式，对nonFairCountCount进行自增操作
        List<Thread> nonFairList = new ArrayList<>();
        for (int i = 0; i < threadNum; i++) {
            nonFairList.add(new Thread(() -> {
                for (int j = 0; j < 10000; j++) {
                    nonFairLock.lock();
                    nonFairCount++;
                    nonFairLock.unlock();
                }
                countDownLatch.countDown();
            }));
        }
        long startTime = System.currentTimeMillis();
        for (Thread thread : nonFairList) {
            thread.start();
        }
        // 让所有线程执行完
        countDownLatch.await();
        return System.currentTimeMillis() - startTime;
    }
}

```

- 线程数为1,跑3次
```
公平锁耗时:   4 ms
非公平锁耗时: 2 ms
-----
公平锁耗时:   5 ms
非公平锁耗时: 3 ms
-----
公平锁耗时:   6 ms
非公平锁耗时: 3 ms
```

- 线程数为5，跑3次
```
公平锁耗时:   337 ms
非公平锁耗时: 11 ms
------
公平锁耗时:   497 ms
非公平锁耗时: 11 ms
------
公平锁耗时:   421 ms
非公平锁耗时: 13 ms
```

- 线程数为10，跑3次
```
公平锁耗时:   991 ms
非公平锁耗时: 22 ms
------
公平锁耗时:   903 ms
非公平锁耗时: 31 ms
------
公平锁耗时:   768 ms
非公平锁耗时: 18 ms

```
我们看到随着线程数的增加，非公平锁的性能要明显高于非公平锁了。所以这应该就是`ReentranLock`默认使用非公平锁的原因了。
其实我们看`synchronized`的源码，它也是非公平锁。

## 独占式同步状态获取和释放

我们先来看一下获取同步状态的方法调用流程

![](/img/acquire.png)

调用`ReentranLock`的`lock`方法，调用的是`FairSync`或者`NonfairSync`的`lock()`方法。
- NonfairSync会先尝试CAS获取一次，如果获取失败则调用AQS的`acquire()`方法
- FairSync则是直接调用AQS的`acquire`方法。
- `acquire`方法中，首先会调用`FairSync`或者`NonfairSync`的`tryAcquire()`方法
- 如果`tryAcquire()`失败，则会进入等待队列中。

下面我们一个方法一个方法的分析，先来看`acquire()`。

## 加锁逻辑

### acquire方法

```java
/**
 * Acquires in exclusive mode, ignoring interrupts.  Implemented
 * by invoking at least once {@link #tryAcquire},
 * returning on success.  Otherwise the thread is queued, possibly
 * repeatedly blocking and unblocking, invoking {@link
 * #tryAcquire} until success.  This method can be used
 * to implement method {@link Lock#lock}.
 *
 * @param arg the acquire argument.  This value is conveyed to
 *        {@link #tryAcquire} but is otherwise uninterpreted and
 *        can represent anything you like.
 */
public final void acquire(int arg) {
    if (!tryAcquire(arg) &&
        acquireQueued(addWaiter(Node.EXCLUSIVE), arg))
        //自我中断，用来传递线程在获取锁的过程中被中断的状态
        selfInterrupt();
}

```
- 注释中提到，这个方法是忽略中断的。也就是说线程在获取锁的过程中，无法通过调用线程的`interrupt()`方法中断获取锁的行为。
- 会至少调用一次`tryAcquire()`方法，如果获取成功则返回。
- 所以说，如果一个线程是第一获取锁的线程，那么它会直接获取成功，并不会创建节点，只有第二个线程来获取，才会创建节点，然后加入等待队列。
- 如果失败线程会进入等待队列。
- 在队列中获取到锁之后会返回，
  - 如果返回true则表示线程在获取锁过程中被调用过中断，则需要调用`selfInterrupt()`重新中断一下自己，把中断状态传传出来
  - 如果返回false则表示没有被调用过中断。

### addWaiter封装线程为Node节点

```java
/**
  * Creates and enqueues node for current thread and given mode.
  *
  * @param mode Node.EXCLUSIVE for exclusive, Node.SHARED for shared
  * @return the new node
  */
 private Node addWaiter(Node mode) {
     //构建新的node节点
     Node node = new Node(Thread.currentThread(), mode);
     // Try the fast path of enq; backup to full enq on failure
     //这里进行一次快速入队，而不是直接走完整的入队方法的考虑是
     //完整入队操作，是通过死循环，然后调用CAS来实现的。JVM或者操作系统，在处理循环的时候，需要很多额外的资源和开销，
     //所以先进行一次单独的CAS操作，如果成功的话，就能节省掉死循环的开销,如果不成功再使用enq，也不会有太大的开销
     Node pred = tail;
     //如果尾节点不为空，把当前节点的前置节点设置为尾节点
     //这里入队的时候，先尝试快速入队
     if (pred != null) {
         //先和前面的节点建立联系，保证从后面遍历的时候，能够遍历到所有的节点
         node.prev = pred;
         //使用CAS设置当前节点为尾节点
         if (compareAndSetTail(pred, node)) {
             //如果设置成功，把前置节点的next指向当前节点
             pred.next = node;
             return node;
         }
     }
     //如果CAS失败，或者尾结点为空，则需要进入完整的入队逻辑。
     //CAS失败，则说明，有另一个线程进入了队列，并成为了尾结点。
     enq(node);
     return node;
 }
```

- 首先通过当前线程构建一个Node节点
- 然后判断如果尾节点不为空，则尝试一次快速入队(通过单次CAS操作)
  - <span style="color:red;">这里有一点需要注意，就是快速入队的时候，是先设置的`node.prev = pred`,然后在设置的`pred.next = node`，包括enq里面的操作也是，这里是有原因的，下面我们会详细的分析一下</span>
- 如果快速入队失败，则调用`enq(node)`,执行完整的入队操作

### enq完整入队
我们说他是一个完整入队逻辑的原因是，它在for循环里，执行CAS操作，直到成功加入队列才会跳出循环

这种操作也叫做 `slow path`

```java
/**
 * Inserts node into queue, initializing if necessary. See picture above.
 * @param node the node to insert
 * @return node's predecessor
 */
private Node enq(final Node node) {
    for (;;) {
        Node t = tail;
        if (t == null) { // Must initialize
            //走到这里的话说明，说明Node还没有初始化，需要创建一个节点，当做头结点
            //初始化的时候，只有一个节点，所以这个节点既是头结点又是尾结点
            if (compareAndSetHead(new Node()))
                tail = head;
        } else {
            //当初始化完成之后，会继续循环，直到把当前节点设置为尾结点。
            node.prev = t;
            if (compareAndSetTail(t, node)) {
                t.next = node;
                return t;
            }
        }
    }
}

```

- 如果当前队列为空，或者快速入队失败，则会执行完整的入队逻辑
- 完整入队的其实就是循环调用快速入队的逻辑，直到成功。
- 如果当前队列为空，这时候需要先把头结点创建出来，然后下次循环的时候，在把真正的节点加入到头结点的后面。
- <span style="color:red;">所以，头结点是被第二个获取锁的线程，获取锁失败的时候，创建的。</span>


### acquireQueued方法

这个方法就是线程等待队列中获取锁，在这个方法中，线程会不断的进行`尝试获取->挂起->唤醒->尝试获取`知道获得锁

```java
/**
 * Acquires in exclusive uninterruptible mode for thread already in
 * queue. Used by condition wait methods as well as acquire.
 *
 * @param node the node
 * @param arg the acquire argument
 * @return {@code true} if interrupted while waiting 如果在等待过程，被中断过，获取锁之后返回true
 */
final boolean acquireQueued(final Node node, int arg) {
    boolean failed = true;
    try {
        boolean interrupted = false;
        for (;;) {
            final Node p = node.predecessor();
            //如果这个节点的前置节点是头结点，那么则表示这个节点有获取锁的资格。
            //顺便说一下，在AQS的队列中个，头结点是一个虚节点，是为了在编程的时候更为方便，或者可以理解为已经获取到锁的节点
            //调用tryAcquire()尝试获取锁
            if (p == head && tryAcquire(arg)) {
                setHead(node);
                //把之前的头结点踢出等待队列
                p.next = null; // help GC
                failed = false;
                //获取锁成功之后，返回中断状态
                return interrupted;
            }
            //如果没有资格获取锁，或者获取锁失败
            //则需要通过shouldParkAfterFailedAcquire()，判断这个线程是否需要挂起
            //如果当前节点，是头结点的下一个节点，那么shouldParkAfterFailedAcquire()方法会返回false，直接跳过挂起，再执行上面的逻辑获取一次
            //如果需要park，则调用parkAndCheckInterrupt()方法，进行挂起。
            //线程在挂起的时候，有两种唤醒的方式，
            // 1.使用unpark
            // 2.使用线程中断方法
            //如果这个线程是被线程中断方法唤醒，那么会把这个interrupted变量置为true
            //然后在等到这个线程获取到锁的时候，会把这个中断状态传出去
            //这里使用到的一个特性就是，被park挂起的线程，当线程中断的时候不会抛出线程中断异常
            if (shouldParkAfterFailedAcquire(p, node) &&
                //如果返回true，表示是被中断唤醒的
                parkAndCheckInterrupt())
                //记录中断状态
                interrupted = true;
        }
    } finally {
        if (failed)
            cancelAcquire(node);
    }
}

```

- 线程被封装成Node节点加入等待队列之后，进入这个方法。
- 首先判断这个节点的前置节点是不是头结点，<span style="color:red;">也就是说，这个节点是等待队列的第二个节点</span>。
- 在AQS中，头结点是虚节点(或者理解为持有锁的节点)，只有第二个节点是有资格去获取锁。
- 然后调用`tryAcquire`尝试获取锁，获取成功后
  - 会调用setHeader()
  - 把当前线程的Node节点设置为头结点
  - 并踢掉之前的头结点
  - 并返回线程的中断状态，记录线程在获取锁的过程中，有没有被中断过。
- 如果不是第二个节点，或者`tryAcquire`失败，会调用`shouldParkAfterFailedAcquire`,判断线程是否应该挂起。

### shouldParkAfterFailedAcquire方法
在这个方法中，会根据节点的状态判断线程是否要被挂起,并在某些情况下修改前置节点状态

```Java
/**
 * Checks and updates status for a node that failed to acquire.
 * Returns true if thread should block. This is the main signal
 * control in all acquire loops.  Requires that pred == node.prev.
 *
 * @param pred node's predecessor holding status
 * @param node the node
 * @return {@code true} if thread should block
 */
//在这个方法判断node节点的线程，是否需要挂起
private static boolean shouldParkAfterFailedAcquire(Node pred, Node node) {
    //获取前置节点的waitStatus
    int ws = pred.waitStatus;
    if (ws == Node.SIGNAL)
        //如果pred节点的waitStatus是SIGNAL，则表示pred节点正在等待别的通知来唤醒
        //所以肯定还轮不到node节点来获取锁，直接挂起
        //这个节点已经设置了请求释放资源的时候来通知它的状态，所以可以安全的挂起
        /*
         * This node has already set status asking a release
         * to signal it, so it can safely park.
         */
        return true;
    if (ws > 0) {
        /*
         * Predecessor was cancelled. Skip over predecessors and
         * indicate retry.
         */

        //waitStatus > 0 只有可能是canceled，那么则踢出pred节点
        //让node的prev指向 pred节点的prev，就相当于跳过了pred节点。
        //这里是一个循环操作，表示会把前面所有的连续的cancel状态的节点都踢出队列
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
        //执行到这里的话，waitStatus只能是0 或者 PROPAGATE。
        //这时候会把前置节点的waitStatus 状态修改为signal，这样在下次循环的时候，当前节点就会执行挂起操作了
        //每个节点的waitStatus都是由他的下一个节点来修改的
        compareAndSetWaitStatus(pred, ws, Node.SIGNAL);
    }
    return false;
}

```

- 首先拿到当前节点的前置节点的`waitStatus`,然后开始判断
- 如果`waitStatus`是`SIGNAL`状态,则返回true，`表示新加入的节点已经告诉了他的前置节点，当前置节点释放的时候唤醒他，他可以放心的挂起了;`下面的逻辑都会返回false，表示不会挂起。
- 如果`waitStatus`大于0，也就是`CANCELLED`状态，表示当前节点的前置节点已经被取消了，那么将这个前置节点踢出等待队列
- 并会把前面所有的连续的`CANCELLED`状态的节点都踢出队列
- 如果`waitStatus`不大于0，那么则`waitStatus`是0,或者`PROPAGATE`，那么就把它设置为`SIGNAL`状态，这样会再次尝试获取一次,如果仍然没有获取到，则会在下次判断`shouldParkAfterFailedAcquire`的时候，因为前置节点是`SIGNAL`而挂起。

这里的重点是对于`SIGNAL`状态的理解，我们再来看一下Java的解释，Java的解释有三个地方

![](/img/signal.png)

![](/img/signal2.png)

![](/img/waitQueue.png)

最后一张图片的注释中提到，AQS中使用和CLH相同的策略，在节点中保存一个后置节点的控制信息。当一个节点的前置节点被释放的时候，通知它的后置节点。
`只有SIGNAL状态表示的是对后置节点的操作，其他状态都是表示的本身节点的状态`

 上面提到，一个节点的`waitStatus`设置为`SIGNAL`，表示的是这个节点的后置节点正在或者将要被阻塞，也可以理解为这个节点的后置节点等待它前面节点唤醒(这个更符合`SIGNAL`的定义)。
 - 所以说，一个节点的处于`SIGNAL`状态，表示的是，当这个节点被释放的时候，需要唤醒它的后续节点。
 - 那么既然这个状态会指导对后置节点的操作，所以这个状态的变更也应该有后置节点来触发。
 - 所以每个节点的初始状态都是0，当有后置节点加入队列的时候，会在后置节点执行`shouldParkAfterFailedAcquire`的时候，把这个节点的状态修改为`SIGNAL`


### parkAndCheckInterrupt挂起并检查中断状态
当经过上面的方法判断，线程需要被挂起，就会执行这个方法，使用`LockSupport.park()`挂起当前线程。

```java
//当线程被unpark()唤醒，或者被终端的时候返回当前线程的中断状态
//使用park()挂起的一个特点就是，当调用线程的中断方法时，不会抛出中断异常，同时也会保留线程的中断状态
private final boolean parkAndCheckInterrupt() {
    LockSupport.park(this);
    //这里注意一下，如果是通过调用线程interrupt方法唤醒了挂起的线程。
    //这里调用的是 interrupted(),这个方法的作用是，返回当前线程的中断状态，然后清除掉中断状态，
    //这里为什么必须要清除中断状态呢？
    //因为线程被唤醒之后，需要再次去尝试获取锁，但是并不能保证肯定成功，如果失败了，还需要继续挂起。
    //如果这里不清除中断状态，那么下次获取失败，需要再次挂起的时候，因为线程有中断状态，所以LockSuport.park()会失效，则导致线程无法挂起。
    return Thread.interrupted();
}

```

- 线程被挂起后就会停在这行代码，等待被唤醒后继续向下执行。
- 线程被唤醒后，继续执行`return Thread.interrupted();`,返回当前线程的中断状态，然后清除掉中断状态。
- 所以如果线程是被中断方法唤醒的，`parkAndCheckInterrupt()`方法就会返回true，然后在`acquireQueued()`方法中就会执行`interrupted = true;`记录下来中断状态。
- 然后在线程成功获得锁之后，将记录的中断状态返回出去。


## 释放锁逻辑

### release方法释放锁

当调用`lock.unlock()`方法释放锁时，会调用`AQS`的`release()`方法。

```Java
/**
 * Releases in exclusive mode.  Implemented by unblocking one or
 * more threads if {@link #tryRelease} returns true.
 * This method can be used to implement method {@link Lock#unlock}.
 *
 * @param arg the release argument.  This value is conveyed to
 *        {@link #tryRelease} but is otherwise uninterpreted and
 *        can represent anything you like.
 * @return the value returned from {@link #tryRelease}
 *
 */
public final boolean release(int arg) {
    //tryRelease，尝试释放
    //这里如果返回false，则表示发生了锁重入，这里只是释放了里层的，外层仍然还在持有锁
    //如果返回true，则表示所有的重入都已经被释放了，可以唤醒下面的等待线程了
    if (tryRelease(arg)) {
        Node h = head;
        //如果有head不为null，则表示至少有一个线程进入过等待队列，因为head节点是在后面线程进入等待队列的时候初始化的
        //h.waitStatus != 0,为什么要加这个判断呢?
        //因为waitStatus=0，是一个节点的初始状态，如果头结点处于这个状态，说明要么后面加入的线程还在执行中，没有执行到 shouldParkAfterFailedAcquire方法中的compareAndSetWaitStatus
        //要么就是后面的线程在执行到 shouldParkAfterFailedAcquire()方法之前，已经超时了
        //这两种情况都不需要唤醒
        //第一种线程正在执行的情况，后面的线程会在第二次循环的时候tryAcquire(),能够拿到锁
        if (h != null && h.waitStatus != 0)
            unparkSuccessor(h);
        return true;
    }
    return false;
}

```
1.首先调用`Sync.tryRelease()`方法，尝试释放锁

```java
//尝试释放锁
protected final boolean tryRelease(int releases) {

    int c = getState() - releases;
    if (Thread.currentThread() != getExclusiveOwnerThread())
        throw new IllegalMonitorStateException();
    boolean free = false;
    //如通c == 0，表示所有的锁都释放完了
    //则设置持有锁线程为null
    //返回释放锁成功
    if (c == 0) {
        free = true;
        setExclusiveOwnerThread(null);
    }
    //如果c != 0 则表示发生了锁重入，只是里层释放了锁
    //返回没有完全释放锁
    setState(c);
    return free;
}

```
这里主要处理了一个重入的逻辑，只有所有重入的锁都释放完毕了，才返回true。
`值得注意的是，这个tryRelease()方法是写在Sync这个类中的，而不是FairSycn和NonFairSync，也就可以说明公平锁和非公平锁的释放逻辑是完全相同的`。

2.`tryRelease()`返回true，则需要判断是否需要唤醒后面的节点。

我们来看这里的判断条件

```java
Node h = head;
if (h != null && h.waitStatus != 0)
    unparkSuccessor(h);
return true;
```
- `head!=null`,因为head节点是，后续线程获取锁失败后，加入队列时创建的，所以`head!=null`就表示有后续节点。
- `h.waitStatus != 0`一个节点的初始状态是0，在后置节点第一次判断是否应该挂起的时候，会把他的前置节点状态修改为`SIGNAL`也就是1。所以`h.waitStatus=0`就可以理解为
  - 后置节点正在执行中，还没有挂起，所以不需要唤醒
  - 后置节点调用的是带有超时的`acquire()`方法，已经等待所超时了
  - 这几种情况都不需要唤醒。

### unparkSuccessor唤醒后置节点
当一个线程释放了锁，并且确定有线程需要唤醒的时候，会调用`unparkSuccessor()`唤醒后面的节点。

```java

/**
 * Wakes up node's successor, if one exists.
 *
 * @param node the node
 */
private void unparkSuccessor(Node node) {
    /*
     * If status is negative (i.e., possibly needing signal) try
     * to clear in anticipation of signalling.  It is OK if this
     * fails or if status is changed by waiting thread.
     */
     // 这里传入的是头结点，头结点已经完成了唤醒后面节点的操作，那么则可以把这个状态改回默认值了
    int ws = node.waitStatus;
    if (ws < 0)
        compareAndSetWaitStatus(node, ws, 0);

    /*
     * Thread to unpark is held in successor, which is normally
     * just the next node.  But if cancelled or apparently null,
     * traverse backwards from tail to find the actual
     * non-cancelled successor.
     */
    /**
     这里从后向前遍历的原因是，在新节点node入队的时候，都是先设置
     node.prev = p（之前的tail节点）(也就是说从后向前可以遍历整个队里了)
     然后通过cas，把node节点设置为tail
     如果cas成功，才设置 p.next = node (这时候从head节点向后才能遍历整个队列)

     以为cas成功，和 设置 p.next = node 这两个步骤并不是原子的
     所以如果cas成功之后，在执行 p.next = node之前，unparkSuccessor()被调用。
     线程开始从前往后遍历，这时候 p.next 还是为null的，所以就不能遍历队列所有的节点了。
     **/

    /**
     * 因为Node是一个双向队列
     * 在插入新节点的时候，总是会先把新节点和前面的节点(也就是之前的尾结点)建立联系。node.prev = p;
     * 然后当新节点成功的成为尾结点之后，才会把 前面的节点和新节点建立联系 p.next = node;
     *
     * 这是可以理解，也是必须的。
     * 因为 p(之前的尾结点)，已经是队列中确定的一个节点，如果在node节点确定加入队列之前就修改了p节点的next属性，如果node能在紧接着的
     * cas操作成功还好，如果cas失败，那就是让p.next指向了一个队列外的节点。这肯定是不行的。
     *
     * 所以说要么把入队和与p.next = node 这两个操作搞成一个原子操作(但是这样肯定会影响性能)，要么想出一个方案，在节点已经入队(已经成为了尾结点)，但是前面节点p还没和node节点建立联系的情况下，也能顺利的遍历所有的节点。
     *
     * 先执行node.prev = p，就是解决这个问题的办法。
     *
     * 所以导致了这里必须采用从队尾开始遍历的方案
     *
     */

    //拿到头结点的下一个节点
    Node s = node.next;
    //如果为空，或者状态是已取消，则从尾结点开始找到一个非canceled状态的节点，然后唤醒它
    if (s == null || s.waitStatus > 0) {
        s = null;
        for (Node t = tail; t != null && t != node; t = t.prev)
            if (t.waitStatus <= 0)
                s = t;
    }
    //如果找到了合适的节点，则使用unpark()方法唤醒
    if (s != null)
        LockSupport.unpark(s.thread);
}

```
- 首先拿到头结点的后置节点，如果为空或者状态是`CANCELLED`则从队列中寻找一个可用的节点
- <span style="color:red"> 这里有一个关键就是从队尾开始寻找 </span>
- 如果找到了，则使用`LockSupport.unpark(s.thread)`唤醒指定的线程。
- 我们上面说过了，线程挂起之后就会停在 `parkAndCheckInterrupt()`方法的`LockSupport.park()`这里，被唤醒后会继续向下执行，就会继续在`acquireQueued()`方法中尝试获取锁
- 不断的循环`尝试获取->失败->挂起->唤醒->尝试获取`这个过程，直到成功获取。



## 重点问题总结
下面的一些重点问题，其实在上面方法的部分中已经提到了，这里专门整理出来，也是为了更方便的查看，AQS中的细节很多，如果全部都去了解需要耗费太多的精力。

### 为什么线程被唤醒之后要调用Thread.interrupted()清除中断状态
[parkAndCheckInterrupt挂起并检查中断状态](/blog/aqs/#parkandcheckinterrupt%e6%8c%82%e8%b5%b7%e5%b9%b6%e6%a3%80%e6%9f%a5%e4%b8%ad%e6%96%ad%e7%8a%b6%e6%80%81)
我们在上面的这个标题中个，讨论了这个问题。
主要原因就是线程在被中断唤醒之后，去获取锁不一定会成功，如果失败了，还需要继续挂起，如果不清除中断状态，就会无法挂起了，所以这里必须要清除


### 为什么需要调用selfInterrupt进行自我中断
上面我们说到，线程在等待锁的状态下，被调用了中断。
- 在执行parkAndCheckInterrupt()方法后，会被清除掉线程的中断状态。
- 但是会在`acquireQueued`方法中记录下来被调用了中断的这个事情`interrupted = true;`
- 线程在获取到锁之后，会返回中断状态
- 如果这时候返回的是true，表示线程在这段时间内被中断过，但是线程的中断状态已经被清除了
- 所以需要调用`selfInterrupt`自我中断一下，设置中断状态，一遍后面的代码可能会有用到中断状态的地方。
- <span style="color:red"> AQS在这里秉持的一个原则就是，我可以不响应中断，但是我不能报中断状态吞掉</span>


### 唤醒后置节点的时候为什么从Tail开始遍历

这个问题我们在[unparksuccessor](/blog/aqs/#unparksuccessor%e5%94%a4%e9%86%92%e5%90%8e%e7%bd%ae%e8%8a%82%e7%82%b9)的代码注释中有详细的说明。

### fast path和slow path
在查看`synchronized`原码的时候就很多次看到`fast path`和`slow path`这两个词。这次又在AQS的代码中看到相关的概念，稍微有一点想法，尝试着解释一下。

`fast path`

- 我忍为可以理解为一种，使用一种相对比较轻量级的方式来达成目标，但是不能保证一定成功。比如在AQS中这里的`fast path`指的就是一次CAS操作。

`slow path`

-相对的slow path，就是指的一种重量级的方式，可以确保一定会成功。在AQS中，`slow path`指的就是循环调用CAS操作。操作系统在执行循环操作的时候，会需要一些额外的资源。

一般在编写过程中，可以先尝试使用一次`fast path`
- 如果成功了则节省了很多资源
- 如果失败了那么在调用`slow path`，比直接调用`slow path`只多出了一次`fast path`操作，消耗也可以接受。





# 参考
[java并发编程系列：牛逼的AQS（上）](https://juejin.cn/post/6844903839062032398)

[从ReentrantLock的实现看AQS的原理及应用](https://tech.meituan.com/2019/12/05/aqs-theory-and-apply.html)

[公平锁与非公平锁的对比](https://juejin.cn/post/6844903985745231886)
