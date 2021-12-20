---
title: "JVM 常用操作集合"
date: 2021-11-30T11:14:17+08:00
tags:
    - JVM
categories: [Java基础]
draft: true
---

# jstat命令

查看jstat支持的选项

`jstat -options`

```
-class 显示ClassLoad的相关信息；
-compiler 显示JIT编译的相关信息；
-gc 显示和gc相关的堆信息；
-gccapacity 　　 显示各个代的容量以及使用情况；
-gcmetacapacity 显示metaspace的大小
-gcnew 显示新生代信息；
-gcnewcapacity 显示新生代大小和使用情况；
-gcold 显示老年代和永久代的信息；
-gcoldcapacity 显示老年代的大小；
-gcutil　　 显示垃圾收集信息；
-gccause 显示垃圾回收的相关信息（通-gcutil）,同时显示最后一次或当前正在发生的垃圾回收的诱因；
-printcompilation 输出JIT编译的方法信息

```

每1秒统计一次gc信息

jstat -gcutil 35001 1000


# jmap 命令

## 查看堆中实例统计

```
jmap -histo
```

# jinfo 命令

## 查看jvm参数

```
jinfo -flags pid 

```

## 查看指定参数

```
jinfo -flag UseConcMarkSweepGC 7628
-XX:+UseConcMarkSweepGC

```


# JVM参数

## 垃圾回收相关

```
-XX:+PrintGC 输出GC日志

-XX:+PrintGCDetails 输出GC的详细日志

-XX:+PrintGCTimeStamps 输出GC的时间戳（以基准时间的形式）

-XX:+PrintGCDateStamps 输出GC的时间戳（以日期的形式，如 2013-05-04T21:53:59.234+0800）

-XX:+PrintHeapAtGC 在进行GC的前后打印出堆的信息

-Xloggc:../logs/gc.log 日志文件的输出路径

```

# 参考

[jstat命令查看jvm的GC情况](https://www.cnblogs.com/yjd_hycf_space/p/7755633.html)

[JVM GC 之「AdaptiveSizePolicy」实战](https://segmentfault.com/a/1190000016427465)

[几种OOM异常分析](https://blog.csdn.net/sunquan291/article/details/79109197)
