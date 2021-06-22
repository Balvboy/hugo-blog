---
layout:     post
title:      "Java同步机制(四)-LockSupport"
description: "LockSupport"
showonlyimage: false
author:     "zhouyang"
date:     2021-06-15
published: true 
tags:
    - LockSupport
    - park
    - unpark
    - AQS
categories: [ Tech , AQS]
mermaid: true

---


# LockSupport

LockSupport是Java中实现同步的一个重要方式，LockSupport提供了阻塞线程和唤醒线程的功能。


## LockSupport中的方法

LockSupport中提供了一系列的`park`，`unpark`方法供我们进行挂起和唤醒。

### park和unpark

![-w373](/img/16184948458475.jpg)

我们看到6个park方法，可以分为了3类分别是

- park() 一直阻塞
- parkNanos() 阻塞指定的纳秒数
- parkUntil() 阻塞到指定的时间，是时间的毫秒值

还有1个unpark方法

- unpark(Thread)  唤醒指定线程

### Blocker
然后每一类方法，都有一个带有Object参数的版本。比如park(Object)方法。

![](/img/16184951584251.jpg)

- 获得了当前线程，
- 然后把传入的对象，通过Unsafe把对象设置到Thread中的parkObject属性。

![](/img/16184952664712.jpg)

![](/img/16184956955973.jpg)

- 然后会调用Unsafe的park方法挂起当前线程。
- 然后待线程被唤醒后，设置blocker为null

那设置和不设置的区别是什么呢?
这个Blocker对象是用来记录线程被阻塞时被谁阻塞的，主要用于线程监控和分析工具来定位原因的。

![](/img/16184962769943.jpg)


## LockSupport的使用

先用代码来展示一下LockSupport最简单的使用方式。

```java
@Test
    public void testPark() throws InterruptedException {
        Object parkObject = "I am parkObject";
        Thread t1 = new Thread(() -> {
            System.out.println("t1调用park");
            LockSupport.park(parkObject);
            System.out.println("t1被唤醒了");
        });

        t1.start();
        Thread.sleep(1000);
        Thread t2 = new Thread(() -> {
            System.out.println( LockSupport.getBlocker(t1));
            System.out.println("t2调用unpark");
            LockSupport.unpark(t1);
        });
        t2.start();
    }
```

执行结果是

```
t1调用park
t2调用unpark
t1被唤醒了,阻塞了1003ms
```

OK,其实上面的代码就能够展示出LockSupport的绝大部分场景的用法了。一个线程因为某些原因调用park方法阻塞，然后另一个线程在达到某些条件的时候，通过unpark唤醒这个线程。

在LockSupport的注释中有这么一句话

```java
/** 
  Additionally, {@code park} will return if the caller's
  thread was interrupted, and timeout versions are supported. The
  {@code park} method may also return at any other time, for "no
  reason", so in general must be invoked within a loop that rechecks
  conditions upon return. 
**/ 
```

意思是，park()方法会在调用线程被终端的时候返回。
park()方法还可以在其他任何时间“毫无理由”地返回，因此通常必须在重新检查返回条件的循环里调用此方法。

这就意味着，当我们使用了`LockSupport.park()`,进行阻塞。当这个方法突然返回的时候，有可能并不是我们调用了unpark()方法导致的。
所以我们需要在循环代码中是否符合业务逻辑，只有确认了是我们自己调用了unpark()触发的才可以。

所以我们看到AQS中对于`LockSupport.park()`的使用，都是在循环中使用的。


## 简单理解LockSupport阻塞的原理
下面来简单的理解一下LockSupport的原理。

每一个线程内部都有一个permit,许可证的概念
线程在初始化的时候，permit为0。
当在线程中调用LockSupport.park()方法的时候，会消耗掉一个permit。

- 如果此时线程中permit为0，线程就会挂起
- 如果此时permit为1，则park()方法会立刻返回，并消耗一个permit，线程内的permit变为0

调用LockSupport.unpark()方法的时候，会生产一个permit。如果该线程因为调用了park()方法而挂起，同时也会唤醒该线程。

- 不管调用多少次unpark,线程中permit的数量最多就是1。

通过上面描述，我们发现LockSupport的工作原理，很像一个信号量为1的Semaphore。
park为加锁，unpark为解锁。

## LockSupport和线程中断
除了unpark能唤醒park挂起的线程外，调用线程的`interrupt()`方法也能唤醒线程。
对于`interrupt()`方法，我们了解，

- 当一个线程处于sleep，或者wait的阻塞状态的时候。如果这个时候调用线程的`interrupt`方法，线程会抛出`InterruptedException`，并清除掉线程的中断状态。
- 如果线程处于运行状态，那么调用线程的`interrupt()`方法，则不会发生任何异常，只会把线程的中断状态设置为true。
- 如果使用`interrupt()`打断一个，通过park挂起的线程，线程会被唤醒，但是不会抛出异常，并且保留线程的中断状态。

而且如果一个线程的中断状态为true，就算没有permit，park()方法也会失效。
我们通过下面的代码来说明一下。

```java
@Test
    public void testInterruptAndPark() throws InterruptedException {
        Thread thread = new Thread(() -> {
            System.out.println("自己中断");
            Thread.currentThread().interrupt();

            System.out.println("打印线程中断状态：" + Thread.currentThread().isInterrupted());

            System.out.println("开始park");
            long st = System.currentTimeMillis();
            LockSupport.park();
            System.out.println("park结束,持续时间:" + (System.currentTimeMillis() - st));

            st = System.currentTimeMillis();
            LockSupport.park();
            System.out.println("park结束,持续时间:" + (System.currentTimeMillis() - st));
        });
        thread.start();
        Thread.sleep(100000);
    }

```
执行结果
```
自己中断
打印线程中断状态：true
开始park
park结束,持续时间:1
park结束,持续时间:0
```

我们在一个线程中，让它自己调用了interrupt方法，然后调用了两次park()方法，然后两次park()方法都失效了。
所以说线程的中断状态，会影响park()的挂起，下面我我们会从源码上来找一下原因。

### 源码分析

#### park和unpark源码

`LockSupport.park()`方法调用的是本地方法。在JVM中最终调用的是`Parker`类的`park`方法，这个方法针对不同平台有不同的实现，这里我们主要看一下Linux平台下的实现。


```c++

// thread.hpp
class Thread {
  // OS data associated with the thread
  OSThread* _osthread;  // Platform-specific thread information
  ParkEvent * _ParkEvent;    // for synchronized(), wait
  ParkEvent * _SleepEvent;   // for Thread.sleep
  // JSR166 per-thread parker
  Parker*    _parker; // for LockSupport::park
  //...
};
class JavaThread: public Thread {
  // 指向Java Thread实例, oop是HotSpot里指向一个Java level的实例, 一个gc对象.
  oop            _threadObj; // The Java level thread ,
  JavaFrameAnchor _anchor;   // Encapsulation of current java frame and it state
  CompiledMethod*   _deopt_nmethod; // CompiledMethod that is currently being deoptimized
  //
  volatile JavaThreadState _thread_state;
  //...
};

```

我们先来看一下JVM中Thread的定义,Thread 类里有两个 ParkEvent 和一个 Parker, 其实 ParkEvent 和 Parker 实现和功能十分类似。

- `_ParkEvent` 是实现 synchronized 关键字，wait，notify 用的，
- `_SleepEvent` 是给 Thread.sleep 用的。
- `_parker` 是用来实现 J.U.C 的 LockSupport的park/unpark (阻塞 / 唤醒)。


```c++
public:
  Parker() : PlatformParker() {
    _counter       = 0 ;
    FreeNext       = NULL ;
    AssociatedWith = NULL ;
  }

```

我们再来看一下Parker的结构。
我们主要看`_counter`这个字段，这个其实就是我们上面说到的permit。


```c++
// src/hotspot/os/posix/os_posix.cpp
//isAbsolute 表示后面的时间是绝对时间还是相对时间
void Parker::park(bool isAbsolute, jlong time) {
  // 设置_counter为0，并且判断原值
  //  如果别的线程已经unpark了我.  
  //  这里没有使用锁机制，需要Atomic::xchg和barrier保证lock-free代码的正确.
  // We depend on Atomic::xchg() having full barrier semantics
  // since we are doing a lock-free update to _counter.
  if (Atomic::xchg(0, &_counter) > 0) return;

  Thread* thread = Thread::current();
  assert(thread->is_Java_thread(), "Must be JavaThread");
  JavaThread *jt = (JavaThread *)thread;

    // 如果线程被中断，直接返回
  if (Thread::is_interrupted(thread, false)) {
    return;
  }

  // safepoint region相关, 我对细节不详.
  // safepoint region大致的了解, 见RednaxelaFX的回答https://www.zhihu.com/question/29268019
  ThreadBlockInVM tbivm(jt);
  // 再次判断线程是否被中断，如果没有被中断，尝试获得互斥锁，如果获取失败，直接返回
  // 如果别的线程正在unpark我, 而持有了mutex, 我先返回了,没有必要在_mutex上等
  if (Thread::is_interrupted(thread, false) || pthread_mutex_trylock(_mutex) != 0) {
    return;
  }
  // 如果别的线程已经unblock了我, no wait needed
  // 已经拿到了mutex, 所以不需要和前面一样Atomic::xchg了.因为已经拿到了锁
  int status;
  if (_counter > 0)  {
    _counter = 0;
    status = pthread_mutex_unlock(_mutex);
    OrderAccess::fence();
    return;
  }
  // 记录线程的状态
  OSThreadWaitState osts(thread->osthread(), false /* not Object.wait() */);
  jt->set_suspend_equivalent();
  // cleared by handle_special_suspend_equivalent_condition() or java_suspend_self()
  // 这一坨, 就是block自己这个线程了.(Java层当前执行的线程)
  if (time == 0) {
    _cur_index = REL_INDEX; // arbitrary choice when not timed
    status = pthread_cond_wait(&_cond[_cur_index], _mutex);
  } else {
    _cur_index = isAbsolute ? ABS_INDEX : REL_INDEX;
    status = pthread_cond_timedwait(&_cond[_cur_index], _mutex, &absTime);
  }
  _cur_index = -1;
  // 已经从block住状态中恢复返回了, 把_counter设0.
  _counter = 0;
  status = pthread_mutex_unlock(_mutex);
  // 要保证多线程的正确性要十二分小心
  // 这里的memory fence 是一个lock addl 指令, 加上compiler_barrier
  // 保证_counter = 0 是对调用unlock线程是可见的.
  // Paranoia to ensure our locked and lock-free paths interact
  // correctly with each other and Java-level accesses.
  OrderAccess::fence();
  // 已经醒过来, 但如果有别人在suspend我,那么继续suspend自己.
  // If externally suspended while waiting, re-suspend
  if (jt->handle_special_suspend_equivalent_condition()) {
    jt->java_suspend_self();
  }
}


```

```c++

// src/hotspot/os/posix/os_linux.cpp
void Parker::unpark() {
  int status = pthread_mutex_lock(_mutex);
  assert_status(status == 0, status, "invariant");
  const int s = _counter;
  _counter = 1;
  // must capture correct index before unlocking
  int index = _cur_index;
  status = pthread_mutex_unlock(_mutex);
  assert_status(status == 0, status, "invariant");

// s记录的是unpark之前的_counter数，如果s < 1，说明有可能该线程在等待状态，需要唤醒。
  if (s < 1 && index != -1) {
    // 发信号唤醒线程
    status = pthread_cond_signal(&_cond[index]);
    assert_status(status == 0, status, "invariant");
  }
}

```


简单总结一下这两个流程，(不会分析每一行代码的作用)

**park**

- 首先把`_counter`设置为0，并判断如果之前为1的话直接返回
- 检查线程的中断状态，如果处于中断状态，则直接返回
- 设置线程状态
- 调用`pthread`相关函数阻塞线程（线程进入阻塞状态，等待阻塞超时或者被唤醒）
- 阻塞超时结束，或者被唤醒之后，设置`_counter`为0

**unpark**

- 把`_counter`设置为1
- 判断`_counter`原值，如果小于1，则表示有可能有线程在阻塞（这里并不是一定，因为初始的时候`_counter`为0）


#### interrupt源码

我们上面说到，`interrupt()`方法，也能唤醒通过`park`阻塞的线程，那我们就来看一下`interrupt`的源码.

```c++
//hotspot\src\os\linux\vm\os_linux.cpp
void os::interrupt(Thread* thread) {
  assert(Thread::current() == thread || Threads_lock->owned_by_self(),
    "possibility of dangling Thread pointer");
  // 获取
  OSThread* osthread = thread->osthread();
  if (!osthread->interrupted()) {
    osthread->set_interrupted(true);
    // More than one thread can get here with the same value of osthread,
    // resulting in multiple notifications.  We do, however, want the store
    // to interrupted() to be visible to other threads before we execute unpark().
    OrderAccess::fence();
    ParkEvent * const slp = thread->_SleepEvent ;
    if (slp != NULL) slp->unpark() ;
  }
  // For JSR166. Unpark even if interrupt status already was set
  if (thread->is_Java_thread())
    ((JavaThread*)thread)->parker()->unpark();
  ParkEvent * ev = thread->_ParkEvent ;
  if (ev != NULL) ev->unpark() ;
}

```

- 每一个Java线程都与一个osthread一一对应，如果相应的os线程没有被中断，则会设置osthread的interrupt标志位为true。
- 并唤醒线程的`_SleepEvent` 随后唤醒线程的`parker`和`ParkEvent`。


#### 总结

通过查看源码，我们对LockSupport的实现原理有了进一步的了解。

- 当调用`LockSupport.park()`的时候，最终调用的是JVM中Thread对象Parker变量的park()方法
- 在park()方法中，会判断`_counter`属性（也就是permit）,然后检查线程的中断状态
- 然后会调用`pthread_cond_wait`,`pthread_cond_timedwait`阻塞线程
- 当调用`LockSupport.park(Thread)`的时候，最终调用的是JVM中Thread对象Parker变量的unpark()方法
- 在unpark()方法中，也会判断`_counter`属性，然后通过`pthread_cond_signal`来环境阻塞的线程。
- 当调用了线程的`interrupt`方法，最终执行的是`os::interrupt()`(Linux平台)
- 在`os::interrupt()`方法中，会设置线程的中断状态，并对JVM线程持有的`SleepEvent`,`Parker`,`_ParkEvent`三个属性，执行`unpark()`方法





# LockSupport和wait、sleep的对比


![](/img/park.png)


- park、unpark方法和wait、notify()方法有一些相似的地方。都是休眠，然后唤醒。但是wait、notify方法有一个不好的地方，就是我们在编程的时候必须能保证wait方法比notify方法先执行。
如果notify方法比wait方法晚执行的话，就会导致因wait方法进入休眠的线程接收不到唤醒通知的问题。

- 而park、unpark则不会有这个问题，我们可以先调用unpark方法释放一个许可证，这样后面线程调用park方法时，发现已经许可证了，就可以直接获取许可证而不用进入休眠状态了。

- 另外，和wait方法不同，执行park进入休眠后并不会释放持有的锁。

- 对中断的处理，`park()`方法阻塞的时候，调用终端，会取消阻塞，但是不会抛出中断异常。






# 参考

[[并发系列-4] 从AQS到futex(二): HotSpot的JavaThread和Parker](http://kexianda.info/2017/08/16/%E5%B9%B6%E5%8F%91%E7%B3%BB%E5%88%97-4-%E4%BB%8EAQS%E5%88%B0futex-%E4%BA%8C-JVM%E7%9A%84Thread%E5%92%8CParker/)

[LockSupport源码分析](https://jlice.top/p/8dxvk/)

[Java Thread 和 Park](https://www.beikejiedeliulangmao.top/java/concurrent/thread-park/)

[Thread.interrupt()相关源码分析](http://www.fanyilun.me/2016/11/19/Thread.interrupt()%E7%9B%B8%E5%85%B3%E6%BA%90%E7%A0%81%E5%88%86%E6%9E%90/)

[Thread.sleep、Object.wait、LockSupport.park 区别](https://blog.csdn.net/u013332124/article/details/84647915)
