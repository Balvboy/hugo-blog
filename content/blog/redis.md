---
layout:     post
title:      "Redis为何使用单线程 "
description: "Redis为何使用单线程"
author:     "zhouyang"
date:     2021-03-19
published: true 
tags:
    - redis
categories: [ tech ]    
---
# Redis的线程
从接触到Redis开始，就了解到Redis的一个重要特性就是单线程。
带着这个特性，我通过命令`top -H -p 2582`查看了Redis Server内部开启的线程，发现Redis中并非只有1个线程，而是有4个。
![](/img/15785618967309.jpg)
这里面肯定有一主线程是负责Redis的操作的，那剩下的3个线程是负责什么的呢。

我们在Redis的源码中寻找一下答案
# Redis源码分析
## 1.main方法
```c
server.supervised = redisIsSupervised(server.supervised_mode);
int background = server.daemonize && !server.supervised;
//判断Redis的启动模式
if (background) daemonize();
  
initServer(); //初始化server服务
if (background || server.pidfile) createPidFile();
redisSetProcTitle(argv[0]);
redisAsciiArt();
checkTcpBacklogSettings();
```
## 2.initServer函数
```c
if (server.cluster_enabled) clusterInit();
replicationScriptCacheInit();
scriptingInit(1);
slowlogInit();
latencyMonitorInit();
//初始化 background io
bioInit();
server.initial_memory_usage = zmalloc_used_memory();
```
这里我们主要关注`bioInit(); `方法，`bio`这里就是background IO的简写

## 3.BIO
```
 /* Background I/O service for Redis.
   *
   * This file implements operations that we need to perform in the background.
   * Currently there is only a single operation, that is a background close(2)
   * system call. This is needed as when the process is the last owner of a
   * reference to a file closing it means unlinking it, and the deletion of the
   * file is slow, blocking the server.
   *
   * In the future we'll either continue implementing new things we need or
   * we'll switch to libeio. However there are probably long term uses for this
   * file as we may want to put here Redis specific background tasks (for instance
   * it is not impossible that we'll need a non blocking FLUSHDB/FLUSHALL
   * implementation).
```

从上面的描述可以看出BIO目前只包括一个操作，就是后台 close内核函数操作，因为这个操作牵扯到很重的文件IO，文件IO会严重阻塞redis-server，所以需要开辟线程来单独处理这些操作。

```c
  void bioInit(void) {
      pthread_attr_t attr;
      pthread_t thread;
      size_t stacksize;
      int j;
  
      /* Initialization of state vars and objects */
      for (j = 0; j < BIO_NUM_OPS; j++) {
          pthread_mutex_init(&bio_mutex[j],NULL);
          pthread_cond_init(&bio_newjob_cond[j],NULL);
          pthread_cond_init(&bio_step_cond[j],NULL);
          bio_jobs[j] = listCreate();
          bio_pending[j] = 0;
      }
  
      /* Set the stack size as by default it may be small in some system */
      pthread_attr_init(&attr);
      pthread_attr_getstacksize(&attr,&stacksize);
      if (!stacksize) stacksize = 1; /* The world is full of Solaris Fixes */
      while (stacksize < REDIS_THREAD_STACK_SIZE) stacksize *= 2;
      pthread_attr_setstacksize(&attr, stacksize);
  
      /* Ready to spawn our threads. We use the single argument the thread
       * function accepts in order to pass the job ID the thread is
       * responsible of. */
      for (j = 0; j < BIO_NUM_OPS; j++) { //循环创建bio线程
          void *arg = (void*)(unsigned long) j;
          if (pthread_create(&thread,&attr,bioProcessBackgroundJobs,arg) != 0) {
              serverLog(LL_WARNING,"Fatal: Can't initialize Background Jobs.");
              exit(1);
          }
          bio_threads[j] = thread;
      }
 }
```
这个函数，从名字就可以看出它的主要功能是完成BIO的初始化操作，在源代码中我们找到了线程开辟操作，这里的思路是线程池，那么问题来了，BIO线程池中到底开辟几个线程呢？

通过观察第121行，可以看到BIO_NUM_OPS参数影响了BIO线程池的线程量，那么这个数值到底为多少呢，我们稍微跟踪一下就可以获取：
```
42  #define BIO_NUM_OPS       3   //bio.h文件
```
现在可以看出BIO_NUM_OPS默认数值为3，这个数值加上1个主线程，正好是图片中的4个线程。


## 4. BIO线程执行的操作
```c
      while(1) {
          listNode *ln;
  
          /* The loop always starts with the lock hold. */
          if (listLength(bio_jobs[type]) == 0) {
              pthread_cond_wait(&bio_newjob_cond[type],&bio_mutex[type]);
              continue;
          }
          /* Pop the job from the queue. */
          ln = listFirst(bio_jobs[type]); //从任务队列中获取任务
          job = ln->value;
          /* It is now possible to unlock the background system as we know have
           * a stand alone job structure to process.*/
          pthread_mutex_unlock(&bio_mutex[type]);
  
          /* Process the job accordingly to its type. */
          if (type == BIO_CLOSE_FILE) { //文件关闭操作
              close((long)job->arg1);
          } else if (type == BIO_AOF_FSYNC) { //异步文件同步操作
              redis_fsync((long)job->arg1);
          } else if (type == BIO_LAZY_FREE) { //redis内存懒释放
              /* What we free changes depending on what arguments are set:
               * arg1 -> free the object at pointer.
               * arg2 & arg3 -> free two dictionaries (a Redis DB).
               * only arg3 -> free the skiplist. */
              if (job->arg1)
                  lazyfreeFreeObjectFromBioThread(job->arg1); //懒释放对象
              else if (job->arg2 && job->arg3)
                  lazyfreeFreeDatabaseFromBioThread(job->arg2,job->arg3);//懒释数据库
              else if (job->arg3)
                  lazyfreeFreeSlotsMapFromBioThread(job->arg3);//懒释放槽位型Map内存
          } else {
              serverPanic("Wrong job type in bioProcessBackgroundJobs().");
          }
          zfree(job);
```

通过最初的现象，我们可以看出Redis-server开辟了四个线程，并通过源代码分析，我们可以看出后三个线程是BIO线程，这三个线程完成的功能是一样的，主要包括：从BIO任务队列中取出任务，文件描述符关闭、磁盘文件同步、内存对象懒释放操作。
其他的任务均由主线程完成。

# Redis单线程的优势
我们看到Redis在处理大多数命令的时候，是通过单线程来处理的这可能给Redis带来下面的优势
1. 使用单线程模型也能并发的处理客户端的请求；
2. 使用单线程模型能带来更好的可维护性，方便开发和调试；
3. Redis 服务中运行的绝大多数操作的性能瓶颈都不是 CPU；

## 单线程也能并发
这里首先要说一下一个老生常谈的问题，并发和并行的区别。
这里用一个例子来说明
> 例如，一个调酒师能够照顾几个顾客，而一次只能准备一种饮料。因此，他可以在没有并行的情况下提供并发。

`所以说并发是，在一个时间段内能够处理多个请求即可，而并行是在同一个时刻能够处理多个请求。`

而Redis就可以理解为，上面的例子中提到的调酒师，Redis通过`IO 多路复用技术`就能够方便的实现同时照看成百上千个`顾客`,但是Redis在同一时刻只能处理一个`顾客`的请求。

但是因为Redis几乎所有操作都是在内存中完成，所以他的每个操作的耗时都非常短，这里有一个计算机各个硬件执行时间的对比。
![](media/15786476237474.jpg)

我们看到内存的读取访问时间是在纳秒级别，而到了硬盘就到了毫秒级。所以Redis这样的以内存操作为主的服务，能够达到每秒 10W甚至100W的并发都是有可能的(当然这个和很多情况有关，比如key的大小，命令的时间复杂度等等)。
同样在这种主要针对内存操作的情况下，Redis对于CPU的消耗是相对比较小的。所以CPU通常并不会成为Redis的瓶颈。
> It’s not very frequent that CPU becomes your bottleneck with Redis, as usually Redis is either memory or network bound. For instance, using pipelining Redis running on an average Linux system can deliver even 1 million requests per second, so if your application mainly uses O(N) or O(log(N)) commands, it is hardly going to use too much CPU.
> [FAQ](https://redis.io/topics/faq)

如果说你想更好的让Redis利用CPU，或者说Redis的并发量还不能满足你的要求，Redis给出的官方建议是，使用分片的方式将不同的请求交给不同的 Redis 服务器来处理，而不是在同一个 Redis 服务中引入大量的多线程操作。
> However, to maximize CPU usage you can start multiple instances of Redis in the same box and treat them as different servers. At some point a single box may not be enough anyway, so if you want to use multiple CPUs you can start thinking of some way to shard earlier.
> [FAQ](https://redis.io/topics/faq)


## 可维护性
可维护性对于一个项目来说非常重要，如果代码难以调试和测试，问题也经常难以复现，这对于任何一个项目来说都会严重地影响项目的可维护性。多线程模型虽然在某些方面表现优异，但是它却引入了程序执行顺序的不确定性，代码的执行过程不再是串行的，多个线程同时访问的变量如果没有谨慎处理就会带来诡异的问题。

## 性能瓶颈
上面也提到了，CPU一般不会成为Redis的瓶颈，Redis 并不是 CPU 密集型的服务，如果不开启 AOF 备份，所有 Redis 的操作都会在内存中完成不会涉及任何的 I/O 操作，这些数据的读写由于只发生在内存中，所以处理速度是非常快的；`整个服务的瓶颈在于网络传输带来的延迟和等待客户端的数据传输，也就是网络 I/O`，所以使用多线程模型处理全部的外部请求可能不是一个好的方案。

>AOF 是 Redis 的一种持久化机制，它会在每次收到来自客户端的写请求时，将其记录到日志中，每次 Redis 服务器启动时都会重放 AOF 日志构建原始的数据集，保证数据的持久性。

多线程虽然会帮助我们更充分地利用 CPU 资源，但是操作系统上线程的切换也不是免费的，线程切换其实会带来额外的开销，其中包括：

1. 保存线程 1 的执行上下文；
2. 加载线程 2 的执行上下文；


频繁的对线程的上下文进行切换可能还会导致性能地急剧下降，这可能会导致我们不仅没有提升请求处理的平均速度，反而进行了负优化，所以这也是为什么 Redis 对于使用多线程技术非常谨慎。

# 引入多线程
> However with Redis 4.0 we started to make Redis more threaded. For now this is limited to deleting objects in the background, and to blocking commands implemented via Redis modules. For the next releases, the plan is to make Redis more and more threaded.

从4.0版本开始，Redis加入了一些可以被其他线程异步处理的删除操作。
## 删除操作
我们可以在 Redis 在中使用 DEL 命令来删除一个键对应的值，如果待删除的键值对占用了较小的内存空间，那么哪怕是同步地删除这些键值对也不会消耗太多的时间。

但是对于 Redis 中的一些超大键值对，几十 MB 或者几百 MB 的数据并不能在几毫秒的时间内处理完，Redis 可能会需要在释放内存空间上消耗较多的时间，这些操作就会阻塞待处理的任务，影响 Redis 服务处理请求的 PCT99 和可用性。

# 总结
Redis 选择使用单线程模型处理客户端的请求主要还是因为 CPU 不是 Redis 服务器的瓶颈，所以使用多线程模型带来的性能提升并不能抵消它带来的开发成本和维护成本，系统的性能瓶颈也主要在网络 I/O 操作上；（因为单线程的Redis已经这么快了，而且瓶颈不在CPU，并且开发起来更简单，所以顺理成章的使用了单线程）
而 Redis 引入多线程操作也是出于性能上的考虑(目前只是针对一些大键的删除)，对于一些大键值对的删除操作，通过多线程非阻塞地释放内存空间也能减少对 Redis 主线程阻塞的时间，提高执行的效率。

# 参考
[Redis-Server 线程模型源码剖析](https://blog.icorer.com/index.php/archives/389/)
[为什么Redis使用单线程模型](https://draveness.me/whys-the-design-redis-single-thread)
[Redis is single-threaded, then how does it do concurrent I/O?](https://stackoverflow.com/questions/10489298/redis-is-single-threaded-then-how-does-it-do-concurrent-i-o)
[为什么说Redis是单线程的以及Redis为什么这么快！](https://blog.csdn.net/xlgen157387/article/details/79470556)