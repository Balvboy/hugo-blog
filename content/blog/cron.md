---
layout:     post
title:      "cron表达式 "
description: "cron是在定时任务中很常用的一种表达式。"
showonlyimage: false
excerpt: "crontab命令常见于Unix和类Unix的操作系统之中，用于设置周期性被执行的指令。该命令从标准输入设备读取指令，并将其存放于“crontab”文件中，以供之后读取和执行。该词来源于希腊语chronos(χρόνος)，原意是时间。"
author:     "zhouyang"
date:     2021-01-04
published: true 
tags:
    - cron
    - crontab
categories: [ tech ]    
---


# 一、cron介绍
> crontab命令常见于Unix和类Unix的操作系统之中，用于设置周期性被执行的指令。该命令从标准输入设备读取指令，并将其存放于“crontab”文件中，以供之后读取和执行。该词来源于希腊语chronos(χρόνος)，原意是时间。
> 通常，crontab储存的指令被守护进程激活，crond常常在后台运行，每一分钟检查是否有预定的作业需要执行。这类作业一般称为cron jobs。
<!--more-->

我理解cron是一个表达式，用来表示一个任务的周期性执行时间。

# 二、cron格式

## 2.1、完整格式

```
┌─秒(0-59)
| ┌─分鐘(0-59)
| │ ┌─小時(0-23)
| | │ ┌─日 day of month(1-31)
| | | | ┌─月(1-12)
| | | | | ┌─星期 day of week(0-7，0和7都表示星期日)
| | | | | | ┌─年(可以省略不写)
* * * * * * *
```

开始的时候我把cron的格式分为了两种，分别是Linux crontab中的格式和Java中使用的格式。后面想了想，觉得cron的格式应该只有一种，就是上面的这种包含 7个字段的格式。然后不同的cron实现对于cron格式的实现程度不同。

比如Linux的crontab，它只能支持5个字段，不能支持秒和年

## 2.2、crontab格式

```
┌─分鐘（0 - 59）
│ ┌─小時（0 - 23）
| │ ┌─日 day of month(1-31)
| | | ┌─月（1 - 12）
| | | | ┌─星期 day of week(0-7，0和7都表示星期日)
* * * * * 
```
[linux-crontab-校验](https://crontab.guru/#5_4_L_*_*)

## 2.3、cron格式总结

因为在网上一直没有找到准确的官方的cron的定义，所以上面的都是我自己的理解。我认为cron这种很通用的表达式，它的定义应该不会太分裂

# 三、day of week中数字的定义
在查看文档和测试的过程中发现，同样的一个表达式，会得出不同的执行结果，后来发现是因为对于day-of-week的定义大致分为两种

## 3.1、crontab中的定义
在类Unix的系统中的crontab命令中，数字0和7都代表周日,然后1-6分别表示MON-SAT(周一到周六)；

这里0和7都表示周日，是有一定的历史原因的，是为了兼容之前不同版本的Unix系统。

> This is a matter of portability. In early Unices, some versions of cron accepted 0 as Sunday, and some accepted 7 as Sunday -- this format is an attempt to be portable with both.

[Day of week {0-7} in crontab has 8 options, but we have only 7 days in a week](https://unix.stackexchange.com/questions/106008/day-of-week-0-7-in-crontab-has-8-options-but-we-have-only-7-days-in-a-week)


## 3.2、Java中的定义
在Java的不同实现中,我找到了2个，分别是Spring的实现和Quartz的实现

* Spring的CronSequenceGenerator和Linux保持一致；顺便说一下这个类是Spring中`@Scheduled`注解的默认cron解析类

* Quartz的CronExpression的实现`CronExpression`和Linux不同，它不支持0，然后1-7分别是SUN-SAT(周日-周六)

## 3.3、解决办法
鉴于不同的实现中，对于day of week数字的定义的区别，最好的办法就是使用星期的简写来代替数字，例如SUN,MON,TUE等等，这样就能保证你的cron不管在哪个Java的cron实现中都能正确运行。

# 四、cron每个字段支持的输入

| Field Name	| 	Allowed Values	 |	Allowed Special Characters|
|--|--|--|
|Seconds	  | 	0-59	     |	, - * /|
|Minutes	  | 	0-59	     | 	, - * /|
|Hours	 	  |     0-23	 	 |  , - * /  |
|Day-of-month | 	1-31	     | 	, - * ? / L W|
|Month	      |	1-12 or JAN-DEC	 |	, - * /|
|Day-of-Week  |	 	0-7 or SUN,MON,TUE,WED,THU,FRI,SAT|	, - * ? / L #|
|Year (Optional)| 	empty, 1970-2199|, - * /|

# 五、cron特殊字符说明
这里先说一下我的理解，对于cron中的特殊字符，应该是有一套比较统一的规则定义，只不过在在不同的cron的实现里，有的只选择实现了一部分的规则。

就像Linux中的crontab命令，只实现了下面的几种基本特殊字符。

Java的实现中，Quartz算是实现的功能比较全的，基本上完全实现了所有的特殊字符的功能。

`所以在我们写cron表达式的时候，一定要清楚，我们的表示是会用在什么地方，支持什么样的写法，像我工作中比较常用的就是在Spring中@Scheduled注解，和elastic-job创建定时任务（cron解析使用Quartz）`

## 5.1、基本特殊字符

| 字符 | 描述 | 举例   |  解释  |
| ------ | ---  | --- | ----- |
| , |表示几个会生效的值 |1,2,3 in Minutes |表示在第1和第2和第3分钟 都会生效|
| - |表示生效的范围    |1-10 in Minutes |表示在第1到第10分钟会生效|
| * |表示所有的值都会生效， |* in Minutes |表示每分钟都会生效|
| / |表示增量，可以理解为每隔多长时间 |0/30 in Minutes|表示第0分钟生效，然后每隔30分钟生效一次|

## 5.2、? 字符说明
> The '?' character is allowed for the day-of-month and day-of-week fields. It is used to specify 'no specific value'. This is useful when you need to specify something in one of the two fields, but not the other.

> ?号只能用在day-of-month和day-of-week字段中。通常用来表示为'没有指定的值'。当你需要在两个字段之一中指定某些内容而不是另一个字段时，这非常有用。

我们使用cron的初衷是想让这个任务定期的执行，比如每个月1日执行什么，或者每周1执行什么。当一个任务在每个月1日，执行的时候，我们通常不会要求它是星期几，同样每周1执行的任务，我们也不会关心当天是几月几号。也就是说，在极大多数的情况下，我们不需要这两个字段同时满足。

`所以说，通常情况下，我们在指定了这两个其中一个字段之后，会把另一个字段设置为'?'。`

或许有人说，我就要每个月1号、并且还必须得是星期一的0点0分0秒，它才能执行，当然我们也能实现(这种同时制定month和week的表达式，不是所有的都支持,比如Quartz就不支持week和month同时设置)
```
0 0 0 1 * 1

接下来7次的执行时间
--------------------------
2020-06-01 00:00:00
2021-02-01 00:00:00
2021-03-01 00:00:00
2021-11-01 00:00:00
2022-08-01 00:00:00
2023-05-01 00:00:00
2024-01-01 00:00:00
```
[spring-cron-online](https://tool.lu/crontab/)

那这个时间真的是你想要的吗？

## 5.3、L 字符说明
L其实是LAST的简写，就是最后的意思，它可以用在 day-of-month和day-of-week字段中。
它在两个字段中的意思有一些不同

* day-of-month：直接使用L表示这个月的最后一天，如果是1月那就是31号，如果是平年的2月，那就是28号
* day-of-month: 使用 L-3,表示这个月的倒数第3天
* day-of-week：如果直接使用L,则表示星期六（是从周日开始的，所以周六是最后一天）
* day-of-week: MONL(为了避免数字造成的混淆直接使用字母)，表示这个月的最后一个周一

## 5.4、W 字符说明
W在这里代表的是WeekDay(工作日的意思)，它只能使用在day-of-month字段中。
在表达式的使用中，它的作用是指定在`同一个月`内，离指定日期`最近`的`工作日`。

举例1

```
0 0 0 15W * ?
这个表达式字面意思是，在离每个月15号最近的工作日的 0点0分0秒触发
1. 如果15号是 周一到周五中的某一天，那么就在当天触发
2. 如果15号是 周六，那么离周六最近的工作日就是周五(前一天)，那么就会在周五触发；
3. 如果15号是 周日，那么离周日最近的工作日是周一(后一天)，那么就是在周一触发；

```
这个有一个大前提就是，必须是同一个月内的。通过下面的例子来说明一下

举例2

```
0 0 0 1W * ?
这个表达式字面意思是，在离每个月1号最近的工作日的 0点0分0秒触发
1. 如果1号是 周一到周五中的某一天，那么就在当天触发（这个没问题）
2. 如果1号是 周日，那么离周日最近的工作日是周一(后一天)，那么就是在周一触发；
3. 如果1号是 周六，关键在这里，距离周六最近的工作日是周五(前一天，1号的前一天已经是上个月了)，但是因为大前提是必须是同一个月的，所以只能在后两天的周一触发

```

## 5.5、# 字符说明

> The '#' character is allowed for the day-of-week field. This character is used to specify "the nth" XXX day of the month.
> 
> If the '#' character is used, there can only be one expression in the day-of-week field ("3#1,6#3" is not valid, since there are two expressions).

`#` 号只能用在 day-of-week字段中，表示某个月的第几个星期几; 同时说明一下 `#`前面的是表示星期几，`#`后面的数字表示第几，还有如果day-of-week使用了`#`。

如果day-of-week字段中出现了`#`,那么day-of-week中就只能有且只有这一种表达式。


举例

```
0 0 0 ? * MON#2
每个月的第二个星期一 0点0分0秒

```



# 六、常见cron

每个月的工作日上午9点0分0秒
```
0 0 9 ? * MON-FRI
```

每个月的最后一个周一上午9点0分0秒
```
0 0 9 ? * MONL
```

每个月的最后一个工作日上午9点0分0秒
```
0 0 9 LW * ?
这里需要注意的就是 W必须要写在后面
```

在工作日每隔10分钟执行一次
```
0 */10 * ? * MON-FRI
```


# 六、参考文章
[Quartz-CronExpression](https://www.quartz-scheduler.org/api/2.1.7/org/quartz/CronExpression.html)

[quartz-cron-校验](https://www.freeformatter.com/cron-expression-generator-quartz.html)

[linux-man-crontab](http://www.manpagez.com/man/5/crontab/)

[linux-easycron](https://www.easycron.com/faq/What-cron-expression-does-easycron-support)

[cron-wiki](https://zh.wikipedia.org/wiki/Cron)
