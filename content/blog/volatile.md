---
layout:     post
title:      "Java同步机制(二)-Volatile"
description: "volatile"
showonlyimage: false
author:     "zhouyang"
date:     2021-05-21
published: true 
tags:
    - volatile
categories: [ tech ]  
mermaid: true

---

# volatile在Java中的语义

对于volatile我们都比较熟悉，`volatile`在Java中有两种作用

- 保障字段在多线程之间的可见性
- 防止指令进行重排序(编译器层面和CPU层面，后面会说明)

下面我就来看一下jvm是如何实现这两种作用的

# JVM对volatile的实现
`volatile`关键字只能用来修饰属性。对于属性有获取和设置两种操作，所以我们就从这两种操作入手分析一下JVM对volitile的处理。

上面的两种操作在字节码中对应着 `getfield`，`getstatic`和`putfield`，`putfield`这四种字节码。

我们去`bytecodeinterpreter.cpp`看一下对应的实现逻辑。(说明一下，JVM现在使用的是模板编译器的，但是字节码编译器可读性比较好，用来学习还是比较合适的)

我们找到上面几个字节码的执行位置

```c++
...
CASE(_getfield):
CASE(_getstatic):
{
   ...
   if (cache->is_volatile()) {
     if (support_IRIW_for_not_multiple_copy_atomic_cpu) {
       OrderAccess::fence();
     }
     ...
   }
  ...
}

```


```c++
...
CASE(_putfield):
CASE(_putstatic):
{
   ...
  if (cache->is_volatile()) {
    ...
    OrderAccess::storeload();
  } 
  ...
}

```

可以看到，在访问对象字段的时候，会先判断它是不是`volatile`的，如果是的话，并且当前CPU平台支持多核核atomic操作的话（现代的绝大多数的CPU都支持），然后就调用`OrderAccess::fence()`。
在设置字段的时候，会使用`OrderAccess::storeload();`
这两个就是JVM提供的内存屏障。

## JVM提供的内存屏障

JVM中，所有内存屏障的使用都由OrderAccess来提供。在`OrderAccess.hpp`中说明了JVM提供的几种内存屏障。

```c++

//                Memory Access Ordering Model
//
// This interface is based on the JSR-133 Cookbook for Compiler Writers.
//
// In the following, the terms 'previous', 'subsequent', 'before',
// 'after', 'preceding' and 'succeeding' refer to program order.  The
// terms 'down' and 'below' refer to forward load or store motion
// relative to program order, while 'up' and 'above' refer to backward
// motion.
//
// We define four primitive memory barrier operations.
//
// LoadLoad:   Load1(s); LoadLoad; Load2
//
// Ensures that Load1 completes (obtains the value it loads from memory)
// before Load2 and any subsequent load operations.  Loads before Load1
// may *not* float below Load2 and any subsequent load operations.
//
// StoreStore: Store1(s); StoreStore; Store2
//
// Ensures that Store1 completes (the effect on memory of Store1 is made
// visible to other processors) before Store2 and any subsequent store
// operations.  Stores before Store1 may *not* float below Store2 and any
// subsequent store operations.
//
// LoadStore:  Load1(s); LoadStore; Store2
//
// Ensures that Load1 completes before Store2 and any subsequent store
// operations.  Loads before Load1 may *not* float below Store2 and any
// subsequent store operations.
//
// StoreLoad:  Store1(s); StoreLoad; Load2
//
// Ensures that Store1 completes before Load2 and any subsequent load
// operations.  Stores before Store1 may *not* float below Load2 and any
// subsequent load operations.

// 省略 acquire and release 部分内容

// Finally, we define a "fence" operation, as a bidirectional barrier.
// It guarantees that any memory access preceding the fence is not
// reordered w.r.t. any memory accesses subsequent to the fence in program
// order. This may be used to prevent sequences of loads from floating up
// above sequences of stores.


```

通过上面的注释我们知道，JVM根据`JSR-133 Cook Book`定义了4种基本的内存屏障操作，并由下面的几种作用。

| 内存屏障 | 使用方法          |作用|
| ----| --------- | ---- |
| LoadLoad     | Load1 LoadLoad Load2 |确保Load1一定是在<br>Load2以及其后的指令之前完成|
| StoreStore   | Store1 StoreStore Store2   |确保 Store1 一定是在<br> Store2 以及其后的指令之前完成<br>（同时，Store1的写入数据会立即被其他 CPU看到|
| LoadStore    | Load1 LoadStore Store2     |确保 Load1 一定是在<br> Store2 以及其后的指令之前完成|
| StoreLoad    | Store1 StoreLoad Load2     |确保 Store1 一定在 <br>Load2 以及其后的指令之前完成|

我们看到，在常见的4中基本内存屏障之后，还单独定义了`fence`操作，`fence`操作的功能可以说涵盖了上面的4中基本屏障。它可以保证`fence`前的任何操作op1都在op2之前完成。

JVM为每种屏障类型都定义了一个单独的方法，带代码中如果需要某种屏障可能直接调用。

```c++

  static void     loadload();
  static void     storestore();
  static void     loadstore();
  static void     storeload();

  static void     acquire();
  static void     release();
  static void     fence();


```

# volatile和硬件的关系

`volatile`在上面提到的两个功能，都是通过JVM定义的内存屏障来实现的。
而JVM定义的内存屏障可以理解是一个规范，它要求不管在什么平台上都要有同样的效
果。但是最终还是要依靠硬件来实现。所以们可以看到，在JVM中对于各种硬件平台都有对应的内存屏障实现。

![](/img/fence.png)


因为我们在服务器领域还是使用Linux比较多，而且大部分使用的是Intel的CPU，所以下面我们先关注一下`linux_x86`的实现

## linux_x86内存屏障实现

```c++

static inline void compiler_barrier() {
  __asm__ volatile ("" : : : "memory");
}

inline void OrderAccess::loadload()   { compiler_barrier(); }
inline void OrderAccess::storestore() { compiler_barrier(); }
inline void OrderAccess::loadstore()  { compiler_barrier(); }
inline void OrderAccess::storeload()  { fence();            }

inline void OrderAccess::acquire()    { compiler_barrier(); }
inline void OrderAccess::release()    { compiler_barrier(); }

inline void OrderAccess::fence() {
  if (os::is_MP()) {
    // always use locked addl since mfence is sometimes expensive
#ifdef AMD64
    __asm__ volatile ("lock; addl $0,0(%%rsp)" : : : "cc", "memory");
#else
    __asm__ volatile ("lock; addl $0,0(%%esp)" : : : "cc", "memory");
#endif
  }
  compiler_barrier();
}

```
我们看上面用到的`fence()`和`storeload()`方法，都其实最终调用的都是`fence()`方法。其他的都是调用的`compiler_barrier()`方法。
然后这里两个方法里面都是使用的内嵌汇编指令的方式，所以我们下来分析一下这个内嵌汇编指令的格式。

### 内嵌汇编指令

内嵌汇编指令的格式是 

`__asm__　__volatile__("Instruction List" : Output : Input : Clobber/Modify);`


- `__asm__`或`asm` 用来声明一个内联汇编表达式，所以任何一个内联汇编表达式都是以它开头的，是必不可少的。
- <span style='color:red'>`__volatile__`或`volatile` 是可选的。如果用了它，表示防止编译器对代码块进行优化。</span>
	- 而这里的优化是针对代码块而言的，使用嵌入式汇编的代码分成三块：
	- 嵌入式汇编之前的代码块
	- 嵌入式汇编代码块
	- 嵌入式汇编之后的c代码块
	- 所以使用了`volatile`修饰的内嵌汇编的意思是，防止编译器对汇编代码块及前后的代码进行重排序等优化。
- `Instruction List` 是要执行的汇编指令序列。它可以是空的。
- `Output`和`Input`是汇编指令中的输入和输出，都可以为空，这里我们不做过多分析
- `Clobber/Modify`是寄存器/内存修改标示。通知GCC当前内联汇编语句可能会对某些寄存器或内存进行修改,希望GCC在编译时能够将这一点考虑进去,这会对GCC在编译的时候有一些影响，但具体是什么影响我们就不深究了。


---

了解了内嵌汇编代码的格式，我们再来看上面的两个方法。

`compiler_barrier()`：` __asm__ volatile ("" : : : "memory");` 

禁止编译器对汇编代码前后的代码块，进行重排序等优化，并且告诉编译器我修改了memory中的内容

`fence()`:` __asm__ volatile ("lock; addl $0,0(%%esp)" : : : "cc", "memory");`

禁止编译器对汇编代码前后的代码块，进行重排序等优化,并且执行 </br>`lock; addl $0,0(%%esp)`这条汇编指令，并且告诉编译器我修改了memory中的内容


### Lock前缀指令

通过上面我们已经知道，JVM通过使用带有`volatile`关键字的内嵌汇编的方便，解决了编译器重排序的问题。那么CPU级别的重排序，和内存间的可见性是怎么实现的呢，下面我们就要用到Lock指令了。

我们看到在`fence()`方法内嵌的汇编代码中，使用了`lock前缀`指令，那`lock前缀`指令在这里起到的是什么作用呢。

#### Lock前缀指令的作用

在Intel 用户开发手册中关于`lock前缀`有这样的描述。

[Intel开发手册下载地址](https://www.intel.cn/content/dam/www/public/us/en/documents/manuals/64-ia-32-architectures-software-developer-system-programming-manual-325384.pdf)


`8.1- LOCKED ATOMIC OPERATIONS` 

> The processor uses three interdependent mechanisms for carrying out locked atomic operations:

>• Guaranteed atomic operations 
>
>• Bus locking, using the LOCK# signal and the LOCK instruction prefix
>
>• Cache coherency protocols that ensure that atomic operations can be carried out on cached data structures (cache lock); this mechanism is present in the Pentium 4, Intel Xeon, and P6 family processors

意思是，我们有3种方式可以实现CPU的原子操作

1. 使用被保证的原子操作，比如读写1byte等
2. 使用`lock`指令作为指令前缀
3. 使用缓存一致性协议

所以使用`lock`指令作为前缀，能够把它后面的一个或者几个操作'包装'为一个原子操作。不过你可能会想，那这么说来`lock`的作用和我们使用它的目前好像不大一样啊，可见性呢？重排序呢？别急我们继续看。


#### Lock前缀保证可见性

`lock前缀`指令的实现方法，在早期CPU和现代CPU中有很大不同,我们还是引用开发手册中的描述

`8.1.2-Bus Locking` 

>In the case of the Intel386, Intel486, and Pentium processors, explicitly locked instructions will result in the asser- tion of the LOCK# signal. It is the responsibility 
>of the hardware designer to make the LOCK# signal available in system hardware to control memory accesses among processors.
>
>For the P6 and more recent processor families, if the memory area being accessed is cached internally in the processor, the LOCK# signal is generally not asserted; 
>instead, locking is only applied to the processor’s caches

意思是，在早期的CPU中，当使用`lock前缀`指令时候，会导致产生一个`LOCK#`信号，通过总线锁定对应的内存，其它 CPU 对内存的读写请求都会被阻塞，直到锁释放。
在后来的处理器的处理逻辑中，如果要操作的内存，已经cache到了处理器的缓存中，那么将不会产生`LOCK#`信号，则通过缓存一致性协议来完成原子性的保证。

`8.1.4-Effects of a LOCK Operation on Internal Processor Caches` 

>For the P6 and more recent processor families, if the area of memory being locked during a LOCK operation is
cached in the processor that is performing the LOCK operation as write-back memory and is completely contained
in a cache line, the processor may not assert the LOCK# signal on the bus. Instead, it will modify the memory location internally and allow it’s cache coherency mechanism to ensure that the operation is carried out atomically. This
operation is called “cache locking.” The cache coherency mechanism automatically prevents two or more processors that have cached the same area of memory from simultaneously modifying data in that area.

意思是：<span style="color: red;">在LOCK操作中被锁定的内存区域是缓存在执行`LOCK`操作的处理器中，作为回写存储器，并且完全包含在缓存行中。在一个缓存行中，处理器可能不会在总线上发出`LOCK#`信号。相反，它将在内部修改内存位置，并允许它的缓存一致性机制确保该操作是原子式进行的。这种操作被称为"高速缓存锁定"。缓存一致性机制会自动防止两个或更多的处理器在缓存同一区域的内存上同时修改该区域的数据。</span>


显而易见，使用`总线锁定`的方式代价要大得多。不过目前在大多数情况下，我们在使用`lock`指令的时候，都是通过缓存一致性协议来保证的后面操作的原子性操作。但是有一些情况同样也会导致不能使用`高速缓存锁定`，而只能使用`总线锁定`，比如涉及到的数据跨越多个CacheLine，CPU不支持缓存锁定等。

- 在 Pentium 和早期的 IA-32 处理器中，lock 前缀会使处理器执行当前指令时产生一个 LOCK# 信号，会对总线进行锁定，其它 CPU 对内存的读写请求都会被阻塞，直到锁释放
- 后来的处理器，加锁操作是由高速缓存锁代替总线锁来处理。 因为锁总线的开销比较大，锁总线期间其他CPU没法访问内存。 这种场景多缓存的数据一致通过缓存一致性协议(MESI)来保证。

所以总结一下，当我们使用`lock前缀`指令的时候会发生这两件事

- 要操作的数据在缓存中，会导致其他CPU的缓存行失效
- 如果数据不再内存中，则会使用总线锁定，把内存中的数据读入或者写出到内存。同时也会导致其他CPU的缓存行失效。

所以使用`lock前缀`指令，会通过触发缓存一致性协议导致关联的缓存行失效，从而保证可见性。

可见性我们知道了，那么防止指令重排序的作用呢？我们继续在用户手册中寻找一下答案。

#### Lock前缀防止指令重排

`8.2.5 Strengthening or Weakening the Memory-Ordering Model`

> The Intel 64 and IA-32 architectures provide several mechanisms for strengthening or weakening the memory-
> ordering model to handle special programming situations. These mechanisms include:

>• The I/O instructions, locking instructions, the LOCK prefix, and serializing instructions force stronger 
>ordering on the processor. 
>
>• The SFENCE instruction (introduced to the IA-32 architecture in the Pentium III processor) and the LFENCE 
>and MFENCE instructions (introduced in the Pentium 4 processor) provide memory-ordering and serialization 
>capabilities for specific types of memory operations.
>
> ... 省略部分
> 
>Synchronization mechanisms in multiple-processor systems may depend upon a strong memory-ordering model. 
>Here, a program can use a locking instruction such as the XCHG instruction or the LOCK prefix to ensure that
>a read-modify-write operation on memory is carried out atomically. Locking operations typically operate like
> I/O operations in that they wait for all previous instructions to complete and for all buffered writes to 
>drain to memory

意思是，Intel处理器提供了几种机制用来加强或者削弱内存排序。其中使用IO相关指令、锁定指令、LOCK前缀指令、序列化相关指令都能够强化
排序。(<span style="color: red;">这里强化的意思是，更加按照指令流的顺序来执行,也就是减少处理器的重排序</span>)

在程序中可以使用锁定指令，或者`lock前缀`指令来强化排序，它们会等待所有先前的指令完成，并等待所有缓冲的写操作耗尽到内存中。

所以我们看到，JVM通过内嵌`lock前缀`的汇编指令，保证了可见性和防止了内存重排序。


### Lock前缀指令的其他应用

我们经常使用的CAS的底层，也是使用`lock前缀`实现的，使用`lock`指令保证了后面的操作的原子性。


### 为何x86只实现了storeload内存屏障

还有一点我们在上面的x86的实现中看到，只有`fence()`方法，和`storeload()`方法使用`lock前缀`指令，其他的几种都是只是实现了编译器级别的内存屏障，也就是只能防止编译器的指令重排序。这是为什么呢？

同样我们在操作手册中寻找一下答案，在操作手册的 `8.2.3-Examples Illustrating the Memory-Ordering`章节，说明x86处理器对`store`和`load`指令的重排序情况

- 8.2.3.2 Neither Loads Nor Stores Are Reordered with Like Operations
- 8.2.3.3 Stores Are Not Reordered With Earlier Loads
- 8.2.3.4 Loads May Be Reordered with Earlier Stores to Different Locations

意思是

- `load`指令和`store`指令，都不能喝同类型的指令之间发生重排序，也就是 load1,load2 和 store1，store2 是不会发生重排序的
- `store`指令和前面出现的`load`指令，不会重排序，也就是 load1,store2 不会发生重排序。
- `load`指令，和它前面出现的`store`指令之间可能会发生重排序。

看到这，我们能知道，x86的CPU，在不添加任何内存屏障的情况下，已经支持了`loadload`,`storestore`,`loadstore`屏障了。只有`storeload`这种需要单独添加内存屏障来保证不会重排序。

所以对于x86处理器原生支持的3种屏障，只需要保证编译器不会发生重排序即可。

所以说，volatile的实现和硬件平台关系非常密切。下面这个是在不同硬件下，发生重排序的情况。

|处理器|	Load-Load	|Load-Store|	Store-Store|	Store-Load|	数据依赖|
|----|-------|-----|-----|----|----|
|x86|	N	|N	|N|	Y|	N|
|PowerPC	|Y|	Y|	Y|	Y|	N|
|ia64	|Y|	Y|	Y|	Y|	N|

我们看到，x86在`loadload`,`loadStore`,`storestore` 和有数据依赖的时候不会发生重排序。
另外两种平台，只有在有数据依赖的时候不会发生重排序。

所以JVM需要根据不同平台来进行不同的处理。


### 为何x86下JVM使用LOCK前缀实现内存屏障

在Intel的处理器平台下内存屏障分为两类：

- 本身是内存屏障，比如“lfence”，“sfence”和“mfence”汇编指令
- 本身不是内存屏障，但是被LOCK指令前缀修饰，其组合成为一个内存屏障。在X86指令体系中，其中一类内存屏障常使用“LOCK指令前缀加上一个空操作”方式实现，比如`lock addl $0x0,(%esp)`

所以有一个疑问，为什么JVM选择使用`lock前缀`指令来实现内存屏障而不使用专门的内存屏障指令呢？

我在JVM的源码中搜索了一下`mfence`,搜索到了下面几段注释

`macroAssembler_x86.cpp`
```c++
// We have a classic Dekker-style idiom:
//    ST m->_owner = 0 ; MEMBAR; LD m->_succ
// There are a number of ways to implement the barrier:
// (1) lock:andl &m->_owner, 0
//     is fast, but mask doesn't currently support the "ANDL M,IMM32" form.
//     LOCK: ANDL [ebx+Offset(_Owner)-2], 0
//     Encodes as 81 31 OFF32 IMM32 or 83 63 OFF8 IMM8
// (2) If supported, an explicit MFENCE is appealing.
//     In older IA32 processors MFENCE is slower than lock:add or xchg
//     particularly if the write-buffer is full as might be the case if
//     if stores closely precede the fence or fence-equivalent instruction.
//     See https://blogs.oracle.com/dave/entry/instruction_selection_for_volatile_fences
//     as the situation has changed with Nehalem and Shanghai.
// (3) In lieu of an explicit fence, use lock:addl to the top-of-stack
//     The $lines underlying the top-of-stack should be in M-state.
//     The locked add instruction is serializing, of course.
// (4) Use xchg, which is serializing
//     mov boxReg, 0; xchgl boxReg, [tmpReg + Offset(_owner)-2] also works
// (5) ST m->_owner = 0 and then execute lock:orl &m->_succ, 0.
//     The integer condition codes will tell us if succ was 0.
//     Since _succ and _owner should reside in the same $line and
//     we just stored into _owner, it's likely that the $line
//     remains in M-state for the lock:orl.
//
// We currently use (3), although it's likely that switching to (2)
// is correct for the future.
```

`orderAccess_linux_x86.inline.hpp`
```c++
// always use locked addl since mfence is sometimes expensive
```

大概意思就是说，`mfence`目前有几个缺点 

1.并不是所有cpu都支持这个指令。
2. 在最早前的CPU中性能比`lock前缀`差一些。
3. 有时`mfence`的性能损耗比较严重

所以基于以上考虑，目前还是使用`lock前缀`(我查看的jvm源码是jdk11的)，但是未来很有可能改为使用`mfence`指令。

[Instruction selection for volatile fences : MFENCE vs LOCK:ADD](https://blogs.oracle.com/dave/instruction-selection-for-volatile-fences-%3a-mfence-vs-lock%3aadd)


## 缓存一致性协议
在上面我们不止一次的提到了缓存一致性协议，那么缓存一致性协议具体是什么样的呢？下面我们来简单的了解一下。

现在处理器处理能力上要远胜于主内存（DRAM），主内存执行一次内存读写操作，所需的时间可能足够处理器执行上百条的指令，为了弥补处理器与主内存处理能力之间的鸿沟，引入了高速缓（Cache),来保存一些CPU从内存读取的数据，下次用到该数据直接从缓存中获取即可，以加快读取速度，随着多核时代的到来,每块CPU都有多个内核，每个内核都有自己的缓存，这样就会出现同一个数据的副本就会存在于多个缓存中，在读写的时候就会出现数据 不一致的情况。

### 缓存行
数据在缓存中不是以独立的项来存储的，它不是一个单独的变量，也不是一个单独的指针,它在数据缓存中以缓存行存在的，也称缓存行为缓存条目。目前主流的CPU Cache的Cache Line大小通常是`64字节`，并且它有效地引用主内存中的一块地址。

#### 局部性原理
局部性原理：在CPU访问存储设备时，无论是存取数据或存取指令，都趋于聚集在一片连续的区域中，这就被称为局部性原理。

- 时间局部性（`Temporal Locality`）：如果一个信息项正在被访问，那么在近期它很可能还会被再次访问。比如程序中的循环、递归对数据的循环访问，
主要体现在指令读取的局部性
- 空间局部性（`Spatial Locality`）：如果一个存储器的位置被引用，那么将来他附近的位置也会被引用。比如程序中的数据组的读取或者对象的连续创建，
对内存都是顺序的读写，主要体现在对程序数据引用的局部性


### MESI协议
`MESI`是众多缓存一致性协议中的一种，也在Intel系列中广泛使用的缓存一致性协议
缓存行（Cache line）的状态有`Modified`、`Exclusive`、 `Share` 、`Invalid`，而MESI 
命名正是以这4中状态的首字母来命名的。该协议要求在每个缓存行上维护两个状态位，使得每个数据单位可能处于M、E、S和I这四种状态之一。

| 状态      | 描述    |  监听任务  |
| -------   | -----  | :----:  |
| M         |该Cache line有效，数据被修改了，和内存中的数据不一致，数据只存在于本Cache中。|   缓存行必须时刻监听所有试图读该缓存行相对就主存的操作，这种操作必须在缓存将该缓存行写回主存并将状态变成S（共享）状态之前被延迟执行。     |
| E         | 该Cache line有效，数据和内存中的数据一致，数据只存在于本Cache中。 |   缓存行也必须监听其它缓存读主存中该缓存行的操作，一旦有这种操作，该缓存行需要变成S（共享）状态。|
| S         | 该Cache line有效，数据和内存中的数据一致，数据存在于很多Cache中。 |  缓存行也必须监听其它缓存使该缓存行无效或者独享该缓存行的请求，并将该缓存行变成无效（Invalid）  |
| I         | 该Cache line无效。 |  无  |


#### 监听任务

- 一个处于M状态的缓存行，必须时刻监听所有试图读取该缓存行对应的主存地址的操作，如果监听到，则必须在此操作执行前把其缓存行中的数据写回主内存
- 一个处于S状态的缓存行，必须时刻监听使该缓存行无效或者独享该缓存行的请求，如果监听到，则必须把其缓存行状态设置为I。
- 一个处于E状态的缓存行，必须时刻监听其他试图读取该缓存行对应的主存地址的操作，如果监听到，则必须把其缓存行状态设置为S


#### 嗅探协议
上面提到的监听任务大多数都是通过嗅探(snooping)协议来完成的

>“窥探”背后的基本思想是，所有内存传输都发生在一条共享的总线上，而所有的处理器都能看到这条总线：缓存本身是独立的，但是内存是共享资源，
>所有的内存访问都要经过仲裁（arbitrate）：同一个指令周期中，只有一个缓存可以读写内存。窥探协议的思想是，缓存不仅仅在做内存传输的时候才和总线打交道，
>而是不停地在窥探总线上发生的数据交换，跟踪其他缓存在做什么。所以当一个缓存代表它所属的处理器去读写内存时，其他处理器都会得到通知，
>它们以此来使自己的缓存保持同步。只要某个处理器一写内存，其他处理器马上就知道这块内存在它们自己的缓存中对应的段已经失效。



#### MESI读取缓存流程

CPU1需要读取数据X，会根据数据的地址在自己的缓存L1A中找到对应的缓存行,然后判断缓存行的状态

- 如果缓存行的状态是`M、E、S`，说明该缓存行的数据对于当前读请求是可用的，则可以直接使用
- 如果缓存行的状态是I，则说明该缓存行的数据是无效的，则CPU1会向总线发送`Read`消息，说’我现在需要地址A的数据，谁可以提供？‘，其它处理器会CPU2监听总线上的消息，收到消息后，会从消息中解析出需要读取的地址，
然后在自己缓存中查找缓存行，这时候根据找到缓存行的状态会有以下几种情况
	- 状态为`S/E` , CPU2会构造`Read Response`消息，将相应缓存行中的数据放到消息中，发送到总线同时更新自己缓存行的状态为`S`，CPU1收到响应消息后，会将消息中的数据存入相应的缓存行中，同时更新缓存行的状态为`S`
	- 状态为`M`，会先将自己缓存行中的数据写入主内存，并响应`Read Response`消息同时将L1B中的相应的缓存行状态更新为`S`
	- 状态为I或者在自己的缓存中不存在X的数据，那么主内存会构造`Read Response`消息，从主内存读取包含指定地址的块号数据放入消息（缓存行大小和内存块大小一致所以可以存放的下），并将消息发送到总线
- CPU1获接收到总线消息之后，解析出数据保存在自己的缓存中


#### MESI写缓存流程

CPU1 需要写入数据X

- 为`E/M`时，说明当前CPU1已经拥有了相应数据的所有权，此时CPU1会直接将数据写入缓存行中，并更新缓存行状态为`M`，此时不需要向总线发送任何消息。
- `S`时，说明数据被共享，其它CPU中有可能存有该数据的副本，则CPUA向总线发送`Invalidate` 消息以获取数据的所有权，其它处理器（CPU2)收到`Invalidate`消息后,会将其高速缓存中相应的缓存行状态更新为`I`，表示已经逻辑删除相应的副本数据，
并回复`Invalidate Acknowledge`消息，CPU1收到所有处理器的响应消息后，会将数据更新到相应的缓存行之中，同时修改缓存行状态为`E`，此时拥有数据的所有权，会对缓存行数据进行更新，最终该缓存行状态为`M`
- `I`时，说明当前处理器中不包含该数据的有效副本，则CPU1向总线发送`Read Invalidate`消息` ，表明`我要读数据X，希望主存告诉我X的值，同时请示其它处理器将自己缓存中包含该数据的缓存行并且状态不是I的缓存行置为无效`
-  其它处理器（CPUB)收到`Invalidate`消息后，如果缓存行不为`I`的话，会将其高速缓存中相应的缓存行状态更新为`I`，表示已经逻辑删除相应的副本数据，并回复`Invalidate Acknowledge消息`
-  主内存收到`Read`消息后，会响应`Read Response`消息将需要读取的数据告诉CPU1
-  CPU1收到所有处理器的`Invalidate Acknowledge`消息和主内存的`Read Response`消息后，会将数据更新到相应的缓存行之中，同时修改缓存行状态为`E`，此时拥有数据的所有权，会对缓存行数据进行更新，最终该缓存行状态为`M`


#### MESI的状态变化

![](/img/20200220235854870.png)

- Local Read：表示本内核读本Cache中的值
- Local Write：表示本内核写本Cache中的值
- Remote Read：表示其它内核读其Cache中的值
- Remote Write：表示其它内核写其Cache中的值
- 箭头表示本Cache line状态的迁移，环形箭头表示状态不变




### StoreBuffer和Invalidate Queue

![](/img/aHR0cDovL3d3dy53b3dvdGVjaC5uZXQvY29udGVudC91cGxvYWRmaWxlLzIwMTUxMi85N2FiZTRkOTNlNmYwNjM5N2FjMWNjZDQ1OWNhNzVlOTIwMTUxMjEwMTExMDM3LmdpZg.png)

说了缓存一致性协议，好像就能够解决问题了，但是，在这里你会发现，又有新的问题出现了。MESI协议中：当cpu0写数据到本地cache的时候，如果不是M或者E状态，需要发送一个`invalidate`消息给cpu1，只有收到cpu1的ack之后cpu0才能继续执行，
在这个过程中cpu0需要等待，这大大影响了性能。于是CPU设计者引入了`Store Buffer`，这个buffer处于CPU与cache之间。


![](/img/storebuffer1.png)

`Store Buffer`增加了CPU连续写的性能，同时把各个CPU之间的通信的任务交给缓存一致性协议。但是`Store Buffer`仍然引入了一些复杂性,那就是缓存数据和`Store Buffer`数据不一致的问题。


#### Store Forwarding

为了解决上面的问题，修改为了下面的架构，这种设计叫做`Store forwarding`，当CPU执行load操作的时候，不但要看cache，还有看`Store Buffer`是否有内容，如果`Store Buffer`有该数据，那么就采用`Store Buffer`中的值。

![](/img/20200221154806152.png)


#### Invalid Queue

同样的问题也会出现在其他线程发送`Invalidate Acknowledge`消息的时候，通常`Invalidate Cacheline`操作没有那么快完成,尤其是在Cache繁忙的时候，这时CPU往往进行密集的`load`和`store`的操作，而来自其他CPU的，
对本CPU Cache的操作需要和本CPU的操作进行竞争，只有完成了`invalidate`操作之后，本CPU才会发生`invalidate acknowledge`。此外，如果短时间内收到大量的`invalidate`消息，CPU有可能跟不上处理，从而导致其他CPU不断的等待。
为了解决这个问题，引入了`Invalid Queue`,系统架构如下。

![](/img/invaidQueue.png)

有了`Invalidate Queue`的CPU，在收到`invalidate`消息的时候首先把它放入`Invalidate Queue`，同时立刻回送`acknowledge` 消息，无需等到该cacheline被真正invalidate之后再回应。当然，如果本CPU想要针对某个cacheline向总线发送invalidate消息的时候，
那么CPU必须首先去`Invalidate Queue`中看看是否有相关的cacheline，如果有，那么不能立刻发送，需要等到`Invalidate Queue`中的cacheline被处理完之后再发送。

另外说一下，x86的CPU不含有 `invalidate queue`,这也是上面提到的，x86平台下只有`storeload`会发生重排序的原因。





## 乱序执行/重排序

了解完上面的这些知识，我们再来整体的总结一下乱序执行或者说重排序的问题。

重排序从发生的环节上来分，可以分为2大类

- 编译器重排序
- 处理器重排序

下面我们分别来说一下。 


### as-if-serial语义
无论是处理器还是编译器,不管怎么重排都要保证(单线程)程序的执行结果不能被改变,这就是as-if-serial语义。

### 编译器重排序

编译器重排序：编译器会对高级语言的代码进行分析，当编译器认为你的代码可以优化的话，编译器会选择对代码进行优化，重新排序，然后生成汇编代码。当然这个优化是有原则的，原则就是在保证单线程下的执行结果不变。

编译器在编译时候，能够获得最底层的信息，比如要读写哪个寄存器，读写哪块内存。所以编译就根据这些信息对代码进行优化，包括不限于减少无用变量、修改代码执行顺序等等。
`优秀的编译器优化能够提升程序在CPU上的运行性能，更好的使用寄存器以及现代处理器的流水线技术，减少汇编指令的数量，降低程序执行需要的CPU周期，减少CPU读写主存的时间`。

当然，编译器在优化的时候，只能保证在单线程下的结果，在多线程情况下，不加限制就可能会导致错误的结果。所以在多线程情况就需要开发者对编译器进行适当的指引，防止编译器进行错误的优化。

### 处理器乱序

CPU接受到编译器编译的指令，通常为了为了提高`CPU流水线的工作效率`，也会对指令进行分析，然后进行乱序执行。当然不同的硬件进行乱序的规则也不大相同，我们还把上面的表格哪来参考一下。


|处理器|	Load-Load	|Load-Store|	Store-Store|	Store-Load|	数据依赖|
|----   |-------|-----|-----|----|----|
|x86 	|N	    |  N  | N   |	Y|	N |
|PowerPC|Y      |	Y |	Y   |	Y|	N |
|ia64	|Y      |	Y |	Y   |	Y|	N |

我们对CPU发生乱序执行情况进行一下分类

- 在没有数据依赖的情况下，发生的乱序。
- CPU顺序执行，但是因为`StoreBuffer`和 `Invalidate Queue`的存在产生了乱序的效果。
	- 关于`StoreBuffer`和 `Invalidate Queue`引起的乱序稍微复杂一点，就不在这里展开了，有兴趣的同学可以查看 [Why Memory Barriers](https://blog.csdn.net/reliveIT/article/details/105902477?spm=1001.2014.3001.5501)

同样上面的这些乱序，都是能够保证在单线程的情况下执行结果的正确行。在多线程环境下，如果需要保证有序，那就需要开发者使用CPU提供的`内存屏障`指令，或者带有内存屏障作用的其他指令，来告诉CPU不要进行乱序执行了。



## volatile作用总结
上面我们对volatile涉及到相关之后都了解了一下，下面对再对volatile的作用做一下总结。

1. 可以防止编译器的重排序优化，会禁止编译器对操作volatile变量的上下两部分代码段进行重排序。
2. 可以防止CPU的重排序优化
  - 使用volatile之后，在某些硬件平台下，会使用内存屏障，防止CPU进行指令重排序，使用内存屏障后，就算是没有相关性的指令，CPU也不会进行排序
3. 可以保证变量在线程之间的可见性
  - 我们上面了解到，在MESI协议之外，有些处理器还有Store Buffer，这就会导致虽然有缓存一致性协议，但是并不能实时生效，所以还是没有办法保证可见性。当使用了volatile之后，使用了内存屏障指令之后，会强制处理器清空Store Buffer，这样就能保证每次操作volatile之后，缓存一致性协议立即生效，这样就能够保证了可见性。



## 伪共享问题

伪共享的意思是，我们在开发中定义了共享变量，然后在多个线程中访问。看起来像是这个变量在内存中，多个CPU共享，但是实际情况是每个CPU都访问自己的缓存，并不是直接访问的整个变量。

伪共享导致的问题就是，如果几个字段因为局部性原理，被加载到了同一个缓存行中，然后被几个线程分别访问。这时就会因为MESI协议，导
致缓存频繁失效，从而CPU每次只能从内存中加载数据，从而降低程序的执行效率。

下面我们通过代码验证一下，
在下面代码中我们我们定义了两个长度为2的long数组，其中一个使用`volatile`关键字修饰，另一个没有。

```java
public class FalseShareTest {
    private final long CYCLE_TIMES = 10_0000_0000L;

    private volatile long[] array = new long[2];

    private long[] array1 = new long[2];

    /**
     * 启动2个线程修改volatile数组的第1，2位 1000w次
     */
    @Test
    public void test() throws InterruptedException {
        Thread t1 = new Thread(() -> {
            for (long i = 0; i < CYCLE_TIMES; i++) {
                array[0] = i;
            }
        });
        Thread t2 = new Thread(() -> {
            for (long i = 0; i < CYCLE_TIMES; i++) {
                array[1] = i;
            }
        });
        long start = System.nanoTime();
        t1.start();
        t2.start();
        t1.join();
        t2.join();
        long end = System.nanoTime();
        System.out.println("Duration: " + (end - start) / 100_0000);
    }


    /**
     * 启动2个线程修改非volatile数组的第1，2位 1000w次
     */
    @Test
    public void test1() throws InterruptedException {
        Thread t1 = new Thread(() -> {
            for (long i = 0; i < CYCLE_TIMES; i++) {
                array1[0] = i;
            }
        });
        Thread t2 = new Thread(() -> {
            for (long i = 0; i < CYCLE_TIMES; i++) {
                array1[1] = i;
            }
        });
        long start = System.nanoTime();
        t1.start();
        t2.start();
        t1.join();
        t2.join();
        long end = System.nanoTime();
        System.out.println("Duration: " + (end - start) / 100_0000);
    }
}

```

每个方法我们都执行5次看一下耗时

```
volatile数组
Duration: 3281
Duration: 3052
Duration: 3071
Duration: 3079
Duration: 3111
非volatile数组
Duration: 358
Duration: 347
Duration: 552
Duration: 545
Duration: 548

```

我们发现修改volatile修饰的数组，消耗的时间是没有volatile数组的10倍。
这个时间花在什么地方呢。
我们来分析一下。

- 缓存行的大小一般为64字节
- 定义的数据长度为2，类型是long，所以数据大小就是16字节。根据局部性原理，所以他们很有可能放到了同一个缓存行中
- 两个线程并发的读取修改第一个数据和第二个数据，这里我们可以理解为CPU1修改第一个数据X，CPU2修改第二个数据Y
- CPU1修改了X，因为数据定义为volatile，所以必须保证多线程之间的可见性，所以会强制使用`lock前缀`指令，让CPU2中对应的缓存行失效
- CPU2想要修改Y，这时候发现这个缓存行已经失效了，所以需要重新从内存中读取，然后修改，同时又会导致CPU1的缓存行失效。
- 所以耗时发生在，volatile的可见性要求导致缓存行一直失效，需要不断的从内存中读取所导致。
- 而非volatile数据，并没有可见性要求，并且因为store buffer的存在，缓存并不会立即失效，所以效率要高的多。

## volatile的优化

知道了上面伪共享和volatile带来的性能问题，那么有没有办法进行优化呢？
答案肯定是有的，我们思考一下，上面的主要问题是，两个数据都存在于同一个缓存行中，对任何一个字段的读写，都会导致缓存行失效，那么我们把这两个数据分开，放到不同的缓存行呢？

我们把上面的代码改一下，把数组长度改为16，两个线程分别修改第1个，和第9个，因为缓存行是64字节，long是8个字节，所以第1个和第9个肯定不会再同一个缓存行中。

```java
public class FalseShareTest {
    private final long CYCLE_TIMES = 10_0000_0000L;

    private volatile long[] array = new long[2];

    private volatile long[] array1 = new long[16];

    public void test() throws InterruptedException {
        Thread t1 = new Thread(() -> {
            for (long i = 0; i < CYCLE_TIMES; i++) {
                array[0] = i;
            }
        });
        Thread t2 = new Thread(() -> {
            for (long i = 0; i < CYCLE_TIMES; i++) {
                array[1] = i;
            }
        });
        long start = System.nanoTime();
        t1.start();
        t2.start();
        t1.join();
        t2.join();
        long end = System.nanoTime();
        System.out.println("Duration: " + (end - start) / 100_0000);
    }


    public void test1() throws InterruptedException {
        Thread t1 = new Thread(() -> {
            for (long i = 0; i < CYCLE_TIMES; i++) {
                array1[0] = i;
            }
        });
        Thread t2 = new Thread(() -> {
            for (long i = 0; i < CYCLE_TIMES; i++) {
                array1[8] = i;
            }
        });
        long start = System.nanoTime();
        t1.start();
        t2.start();
        t1.join();
        t2.join();
        long end = System.nanoTime();
        System.out.println("Duration: " + (end - start) / 100_0000);
    }

    public static void main(String[] args) throws InterruptedException {
        FalseShareTest falseShareTest = new FalseShareTest();
        System.out.println("未优化");
        falseShareTest.test();
        falseShareTest.test();
        falseShareTest.test();
        falseShareTest.test();
        falseShareTest.test();
        System.out.println("优化后");
        falseShareTest.test1();
        falseShareTest.test1();
        falseShareTest.test1();
        falseShareTest.test1();
        falseShareTest.test1();
    }
}

```

执行结果
```
未优化
Duration: 6589
Duration: 7006
Duration: 7527
Duration: 7278
Duration: 7813
优化后
Duration: 1961
Duration: 2011
Duration: 1974
Duration: 2056
Duration: 1953
```

我们发现把数据移动到不同的缓存行之后，运行时间缩短了三分之二左右。

所以对volatile的优化的主要逻辑就是，让volatile的字段，自己处于一个缓存行中，不要让其他字段的变化，影响volatile变量所处缓存行的状态。


### false share的优化案例

这种优化，在很多高性能的框架中还是挺常见的。比如在caffeine中就有这样的应用

![](/img/falseshare.png)

依据JVM对象继承关系中父类属性与子类属性，内存地址连续排列布局,PadReadCounter中的字段会作为前置填充，PadWriteCounter中的字段会作为后置填充。
保证readCounter字段能够独占一个缓存行。

我们看到这里前后各使用的15个long字段作为填充，这应该是为了防止不同的系统下缓存行大小不一致的情况。






# 参考

[[关键字: volatile详解]](https://www.pdai.tech/md/java/thread/java-thread-x-key-volatile.html)

[[汇编指令的LOCK指令前缀]](https://dslztx.github.io/blog/2019/06/08/%E6%B1%87%E7%BC%96%E6%8C%87%E4%BB%A4%E7%9A%84LOCK%E6%8C%87%E4%BB%A4%E5%89%8D%E7%BC%80/)

[[一文了解内存屏障]](https://monkeysayhi.github.io/2017/12/28/%E4%B8%80%E6%96%87%E8%A7%A3%E5%86%B3%E5%86%85%E5%AD%98%E5%B1%8F%E9%9A%9C/)

[[volatile与内存屏障]](https://blog.csdn.net/qq_26222859/article/details/52240256)

[[内存屏障及其在JVM内的应用]](https://segmentfault.com/a/1190000022508589)

[[CPU缓存一致性协议MESI]](https://www.cnblogs.com/yanlong300/p/8986041.html)

[[x86架构]](https://zh.wikipedia.org/wiki/X86)

[[聊聊缓存一致性协议]](https://albk.tech/%E8%81%8A%E8%81%8A%E7%BC%93%E5%AD%98%E4%B8%80%E8%87%B4%E6%80%A7%E5%8D%8F%E8%AE%AE.html)

[[浅论Lock与X86 Cache一致性]](https://zhuanlan.zhihu.com/p/24146167)

[[Why Memory Barriers？中文翻译（上）]](https://blog.csdn.net/reliveIT/article/details/105902477?spm=1001.2014.3001.5501)

[[Volatile全面解析，实现原理及作用分析]](https://blog.csdn.net/b_x_p/article/details/104382767)

[[编译器重排序]](https://blog.csdn.net/CringKong/article/details/99759216)

[[一篇对伪共享、缓存行填充和CPU缓存讲的很透彻的文章]](https://blog.csdn.net/qq_27680317/article/details/78486220)

[[为什么volatile注释变量，对其下面的变量也会有影响？]](https://www.zhihu.com/question/348513270)

[[一次深入的volatile研究]](https://createchance.github.io/post/%E4%B8%80%E6%AC%A1%E6%B7%B1%E5%85%A5%E9%AA%A8%E9%AB%93%E7%9A%84-volatile-%E7%A0%94%E7%A9%B6/)

[[既然CPU有缓存一致性协议（MESI），为什么JMM还需要volatile关键字？]](https://www.zhihu.com/question/296949412)
