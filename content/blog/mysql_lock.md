---
title: "MySQL的事务、日志、锁和MVCC"
date: 2021-07-28T10:53:40+08:00
author:     "zhouyang"
draft: true
tags:
    - mysql
categories: [ MySQL ]   

---
# MySQL的存储引擎
MySQL有多种可选的存储引擎，常见的有

- InnoDB
- MyISAM
- Memory

其中InnoDB是最常用的存储引擎，并且也是目前MySQL的默认存储引擎。
其中最重要的原因就是，InnoDB支持事务，支持行级锁定。

想了解更多的不同存储引擎的区别，可以查看
[MySQL存储引擎的区别和比较](https://blog.csdn.net/zgrgfr/article/details/74455547)

# MySQL的事务
事务是数据库执行过程中的一个逻辑单位，它可能由一条或多条数据库操作组成。

## 开启事务
在MySQL中

- 对于单条SQL语句，数据库系统自动将其作为一个事务执行，并自动提交事务，这种事务被称为隐式事务。
- 对于多条操作，如果想要放到一个事务中，则需要手动开启事务，并提交

```SQL
begin;
UPDATE accounts SET balance = balance - 100 WHERE id = 1;
UPDATE accounts SET balance = balance + 100 WHERE id = 2;
commit;

```

## 事务的特性

ACID是我们非常熟悉的事务的特性，它指的是数据库管理系统（DBMS）在写入或更新资料的过程中，为保证事务（Transaction）是正确可靠的，所必须具备的四个特性:

- 原子性(Atomicity):

  一个事务中的操作要么全部完成，要么全部不完成，不可分割

- 一致性(Consistency):

  下面首先给出wiki中关于一致性行的定义

  >在事务开始之前和事务结束以后，数据库的完整性没有被破坏。这表示写入的资料必须完全符合所有的预设约束、触发器、级联回滚等

  但是我认为这个解释不够清晰，我个人更倾向于知乎[如何理解数据库的一致性](https://www.zhihu.com/question/31346392)中的一个解释

  >保证事务开始和结束之间的中间状态不会被其他事务看到

- 隔离性(Isolation):

  隔离性的原本意思是，保证事务之间的操作是互相不可见，也就是每个事务都是隔离开的。
  但是实现最高的隔离性，性能很低，同时很多业务场景也并不需要最高的隔离性和一致性。
  所以就提出了几种不同的隔离级别来适当的破坏一致性来提升性能与并行度。


- 持久性(Durability):

  事务处理结束后，对数据的修改就是永久的，即便系统故障也不会丢失。

## 关于事务的理解

同样还是在上面提到的那个知乎回答中[如何理解数据库的一致性](https://www.zhihu.com/question/31346392)，提到了为什么会有事务的问题，我觉得挺对的。

总结一下，意思大概是`事务这个概念本身，并不是数据库天生就有的。而是伴随着应用开发出现的各种各样的问题，为了解决这样的问题，并简化应用程序的编程模型而逐渐产生，或者说被提炼出来的。`

对于这种概念性的知识确实比较容易搞偏，陷入不必要，所以我们打住，来看一下技术实现。

## InnoDB对事务特性的实现

可以说InnoDB中的锁、redo log、undo log、MVCC等机制，都是为了实现事务的特性而实现的。下面我们就来简单的分析一下他们都是如何实现的。

### 原子性

MySQL原子性的保证–undo log

原子性的概念是一个事务内的操作要么全执行，要么全不执行。为了解决这个问题，就有了事务回滚的操作。一旦事务执行过程中发生了异常，那么可以通过回滚，把数据回滚。

undo log从字面意思就是撤销日志。负责撤销的日志。

简单理解一下，就是undo log中存储的是和每条纪录相反的操作。比如

- 执行insert，undo log中就存储delete
- 执行delete，undo log中就存储insert
- 执行update，会记录一条相反的update

通过这样的方式，保证在事务中执行失败的时候，数据能够回滚。

当然undo log的作用不仅于此，我们后面会继续分析。


### 持久性
持久性保证的就是，一旦事务提交后，数据一定会保存下来，就算数据库发生错误也不会导致数据丢失。

持久性是依赖 InnoDB的redo log 和 bin log来实现的。
下面我们分析一下MySQL存储一条数据到磁盘的可能方式。
1. 第一种就是每一次更新操作都需要写进磁盘，然后磁盘也要找到对应的那条记录，然后在更新，整个过程IO成本，查找成本都很高
2. 第二种先写到内存中的一块区域也就是redo log，就是MySQL里面经常说到的WAL技术，WAL全称为Write-Ahead Logging，他的关键点就是先写日志，再写磁盘。等服务不太忙的时候再把数据写到磁盘。

- 执行器先找到引擎取ID=2这一行。ID是主键，引擎直接用树搜索找到这一行，如果ID=2这一行所在的数据页本来就在内存中，就直接返回给执行器；否则，需要先从磁盘读入内存，然后再返回。
- 执行器拿到引擎给的行数据，把这个值加上1，比如原来是N，现在是N+1，得到新的一行数据，再调用引擎接口写入这行新数据。
- 引擎将这行新数据更新到内存中，同时将这个更新操作记录到redo log里面，此时redo log处于prepare状态。然后告知执行器执行完成了，随时可以提交事务。
- 执行器生成这个操作的binlog，并把binlog写入磁盘。
执行器调用引擎提交事务接口，引擎把刚刚写入的redo log改成提交状态，更新完成

参考[MySQL原子性与持久性的保证](https://blog.csdn.net/anying5823/article/details/104675987)

### 隔离性
隔离性最简单的实现方式就是各个事务都串行执行，如果前面的事务还没有执行完毕，后面的事务就都等待。但是这样的实现方式很明显并发效率不高，并不适合在实际环境中使用。
为了解决上述问题，实现不同程度的并发控制，SQL的标准制定者提出了不同的隔离级别，其中最高级隔离级别就是序列化读，而在其他隔离级别中，由于事务是并发执行的，所以或多或少允许出现一些问题。

- 未提交读（read uncommitted）可能发生 脏读、不可重复读、幻读
- 提交读（read committed）可能发生 不可重复读、幻读
- 可重复读（repeatable read）可能发生 幻读
- 序列化读（serializable）没有问题

### 一致性

这个特性其实比较特殊，这个应该是事务的最终追求。所以也可以说是上面的三种特性的实现共同实现了一致性。

# MySQL中的锁

从锁的类型来分，可以分为

- 共享锁 Share Lock
- 排它锁 Exclusive Lock

从锁粒度来分可以分为

- 全局锁： 对整个数据库实例加锁，使用的不多
  - 使用 `flush tables with read lock`加锁
  - 使用 `unlock tables`解锁
- 表锁: 对整个表加锁
- 行锁: 对记录所在的索引加锁
- 页面锁: 页面锁接触不多，我们可以暂时忽略

还有几个比较特殊的表级锁

- AUTO-INC Locks 自增主键用到的锁
- 元数据锁(meta data lock) 处理增删改查和修改数据库结果之间的冲突
- 共享意向锁
- 排他意向锁

行锁从类型又有可以分为

- 普通行锁 (record lock)
- 间隙锁 (gap lock)
- 邻间锁 (next-key lock)
- 插入意向锁 (insert intention lock)

下面我们稍微来分析一下

## 表锁

表锁，顾名思义直接锁定一张表，这个锁粒度比较大，但是优点是比较简单。

Memory、MyISAM 引擎都只支持表锁。

InnoDB在默认情况下不会使用普通的表锁，想要使用表锁的话需要使用`lock table`命令。

![](/img/locktable.png)

我们发现锁表之后，右边插入操作被阻塞住了。

接着我们使用下面命令查看一下锁状态，这个命令输出内容有很多，我们主要关注`TRANSACTION`部分

```SQL
show engine innodb status\G;
```
![](/img/transaction.png)

发现有一个表被锁定了。

![](/img/unlock.png)
当我们执行解锁命令，右面的操作就完成了。

表锁我们使用不多，所以不做过多了解，有兴趣的同学可以查看[表锁命令](https://blog.csdn.net/weixin_41282397/article/details/84791087)，进行了解。


### 意向锁

意向锁的作用很简单，就是为了平衡表锁和行锁之间的冲突。我们想一下这个场景

- 一个事务T1在表A中对某些记录加了独占行锁。
- 另一个事务T2，想要对表A添加表锁。

这时候T2怎么判断表中又没其他事务加了行锁呢？遍历所有的记录是一个方式，但是这样效率太慢了，所以就提出了意向锁的概念。

- 如果想要共享行锁，那么就现在表上添加一个共享意向锁IS，然后在行记录的索引上添加S锁
- 如果想要独占行锁，那么就现在表上添加一个独占意向锁IX，然后在行记录的索引上添加X锁

有了这个机制，上面的事务T2，就不需要遍历就能知道当前这个表中有记录被添加了独占锁，下面来操作一下

![](/img/intentionLock.png)

- 首先在一个事务中使用`select * from user where id = 1 for update` 给表中的一条记录添加独占锁
- 然后在另一个事务中尝试锁表，

结果发现锁表的这个操作被阻塞了。我们来看一下锁状态

![](/img/lock1.png)
我们可以看到`user`表被添加了一个IX的意向锁。

![](/img/lock2.png)

- 接着我们提交事务
- 另一个事务中，获取表锁成功，说明在事务提交后，意向锁被释放了

接着我们测试一下共享行锁下的情况

![](/img/lock3.png)

- 首先我们使用 `select * from user where id = 1 lock in share mode;`给记录的索引添加共享行锁
- 接着分别使用 `lock table user read` 和 `lock table user write` 来分别尝试获取表级的读锁和写锁。

结果，在存在共享行锁的情况下，获取表级读锁可以成功，获取表级写锁会被阻塞。

下面是各种类型表锁之间的兼容性

|   |	X	|IX |S  |IS |
|---|---|---|---|---|
|X  |	Conflict	|Conflict	|Conflict|Conflict|
|IX | Conflict|	Compatible|	Conflict|Compatible|
|S  | Conflict|	Conflict|	Compatible|Compatible|
|IS | Conflict|	1Compatible|	Compatible|Compatible|


### AUTO-INC锁

顾名思义，AUTO-INC锁就是当表里有字段设置了自增之后，那么向表里插入数据时，就需要获得这个锁。AUTO-INC锁可以分为3中模式

- 传统模式(Traditonal)
- 连续模式(Consecutive)
- 交叉模式(Interleaved)

分别对应配置项 innodb_autoinc_lock_mode 的值0、1、2.

- 在MYSQL 5.1.22版本前，自增列使用AUTO_INC Locking方式来实现，即采用一种特殊的表锁机制来保证并发插入下自增操作依然是串行操作，为提高插入效率，该锁会在插入语句完成后立即释放，而不是插入语句所在事务提交时释放。(传统模式)
  - 该设计并发性能太差，尤其在大批量数据在一条语句中插入时(INSERT SELECT ), 会导致该语句长时间持有这个“表锁”，从而阻塞其他事务的插入操作。
- 在MYSQL 5.1.22版本开始，InnoDB存储引使用一种轻量级互斥锁(Mutex)来控制自增列增长，并提供innodb_autoinc_lock_mode参数来控制。这种是5.1.22之后的默认插入模式。（连续模式）
  - 在连续模式下，如果插入的数量是确定的(simple insert)，则使用轻量级互斥锁，因为只需要吧这次插入所需要的id，预留出来就好，不需要等待插入完成。
  - 如果插入的数量不确定，比如`select into xxx select * from xxx`这种，则还需要使用自增锁
- 在MySQL8.0后，把默认插入模式修改为了2，也就是交叉模式。
  - 在交叉模式中，所有insert语句都使用轻量级互斥锁。这是三种模式中并发度最高的，也是性能最好的。
  - 但是这种模式有一个缺点，当多个insert语句并发的时候，会造成一个事务插入的id并不是连续的。因为 AUTO_INCREMENT 的值分配会在多个 INSERT 语句中来回交叉执行。

#### 交叉模式的缺陷
要了解缺陷是什么，还得先了解一下 MySQL 的 Binlog。Binlog 一般用于 MySQL 的数据复制，通俗一点就是用于主从同步。在 MySQL 中 Binlog 的格式有 3 种，分别是：

- *Statement* 基于语句，只记录对数据做了修改的SQL语句，能够有效的减少binlog的数据量，提高读取、基于binlog重放的性能
- *Row* 只记录被修改的行，所以Row记录的binlog日志量一般来说会比Statement格式要多。基于Row的binlog日志非常完整、清晰，记录了所有数据的变动，但是缺点是可能会非常多，例如一条update语句，有可能是所有的数据都有修改；再例如alter table之类的，修改了某个字段，同样的每条记录都有改动。
- *Mixed* Statement和Row的结合，怎么个结合法呢。例如像alter table之类的对表结构的修改，采用Statement格式。其余的对数据的修改例如update和delete采用Row格式进行记录。

如果 Binlog 采用的格式为 Statement ，那么 MySQL 的主从同步实际上同步的就是一条一条的 SQL 语句。如果此时我们采用了交叉模式，那么并发情况下 INSERT 语句的执行顺序就无法得到保障，将会导致主从之间同行的数据主键 ID 不同。而这对主从同步来说是灾难性的。

所以说如果想要提高插入性能，并且保证主从数据一致，那么可以使用 交叉模式+Row格式的Binlog。


[深入剖析 MySQL 自增锁](https://juejin.cn/post/6968420054287253540)

### 元数据锁(MDL)

元数据锁的作用就是，防止在事务在进行增删改查的时候，另一个事务对表结构进行修改，所造成的问题。

在 MySQL 5.5 之后加入了MDL，当事务对一个表做增删改查操作的时候，会加MDL读锁，当对表做结构变更操作的时候，加了MDL写锁。

所有当有事务在进行增删改查的时候，修改表结构的语句会被阻塞。

![](/img/mdl.png)
我们看到，当一个事务正在执行查询的时候，哪怕是没有加锁的正常查询，也会阻塞修改表结构的语句。

![](/img/mdl1.png)
我们看到，当事务提交之后，DDL语句也执行成功了。

那么如果我们是先执行DDL语句呢，那么在事务提交之前会阻塞查询吗？

![](/img/mdl3.png)
这次我们先执行的DDL语句，然后并没有提交，然后在另一个事务中执行的查询操作，然后并没有阻塞,这也就说明，MDL写锁是在DDL语句执行完就释放了。

[MySQL 全局锁和表锁是什么](https://blog.csdn.net/weixin_38118016/article/details/90384191)

## 行锁

首先关于行锁要说的就是，行锁和数据库使用的隔离界别有很大的关系。这里我们只说比较常用的`读已提交 RC`和`可重复读 RR`这两个级别。

- 在`RC`隔离级别下行锁只有普通记录锁这一种；
- 在`RR`隔离界别下行锁种类比较多
  - 普通记录锁
  - 间隙锁
  - 邻间锁
  - 插入意向锁

下面我们通过例子来说明一下，我们分别在默认隔离级别RR和RC下都执行下面的SQL,然后使用`show engine innodb status\G`查看锁状态。

```SQL
begin;
select * from user where age = 10 for update;

```

1. RR
![](/img/rr-lock.png)
我们看到这里出现了3种锁类型

2. RC
`提示：在MySQL 8.0.3 中，tx_isolation 变量被 transaction_isolation 变量替换了。`

```SQL
/**修改当前会话隔离级别**/
set session transaction isolation level read committed;
/**查看当前会话级别**/
SELECT @@session.transaction_isolation;
begin；
select * from user where age = 10 for update;
```

![](/img/rc-lock.png)

![](/img/rc-lock2.png)

我们看到只有一种锁类型

我们在上面通过`show engine innodb status\g`查看锁状态，发现了几种锁类型，他们分别是这样对应的

- lock_mode X 表示邻间锁
- lock_mode X locks rec but not gap 表示普通行锁
- lock_model X locks gap 表示间隙锁

下面我们分别来分析一下这几种锁，在说下面几种锁的时候我们需要了解一些前提

- 下面的分析都会在默认隔离级别下进行。
- 在InnoDB中，所有的行锁都是添加在索引上的（这一条先记住，后面会慢慢涉及到）


### show engine innodb status分析
下面在了解行锁之前，我们先学会怎么分析 MySQL的锁日志，下面我们通过一段日志来分析一下。

下面是表的记录情况：
```
+----+--------+------+------+------+
| id | number | age  | sex  | name |
+----+--------+------+------+------+
|  1 |      1 |    1 |    0 | NULL |
|  3 |      3 |    3 |    1 | NULL |
|  4 |      4 |    4 |    1 | NULL |
|  5 |      5 |    5 |    1 | NULL |
|  7 |      7 |    4 |    1 | NULL |
| 10 |     10 |   10 |    1 | NULL |
| 15 |     15 |   15 |    1 | NULL |
| 20 |     20 |   20 |    1 | NULL |
| 25 |     25 |   15 |    0 | NULL |
+----+--------+------+------+------+

```

```SQL
/** 查询一个二级索引  **/
mysql> begin ;
Query OK, 0 rows affected (0.00 sec)

mysql> select * from user where age = 15 for update;
+----+--------+------+------+------+
| id | number | age  | sex  | name |
+----+--------+------+------+------+
| 15 |     15 |   15 |    1 | NULL |
| 25 |     25 |   15 |    0 | NULL |
+----+--------+------+------+------+
2 rows in set (0.00 sec)

mysql> show engine innodb status\G
```
![](/img/lockstatus2.png)

下面我们来逐行分析一下

```yml
TRANSACTIONS

Trx id counter 1906
# - 下一个事务的id为1906
Purge done for trx's n:o < 1900 undo n:o < 0 state: running but idle
# Purge线程已经将trxid小于1900的事务都purge了，目前purge线程的状态为idle   
History list length 0
# undo中未被清除的事务数量，如果这个值非常大，说明系统来不及回收undo，需要人工介入了。
LIST OF TRANSACTIONS FOR EACH SESSION:
# 所有当前正在活跃的事务列表
---TRANSACTION 422179967867392, not started
0 lock struct(s), heap size 1136, 0 row lock(s)
---TRANSACTION 422179967866552, not started
0 lock struct(s), heap size 1136, 0 row lock(s)
---TRANSACTION 1905, ACTIVE 14227 sec
# 当前事务ID为 1905，活跃了 14227秒
4 lock struct(s), heap size 1136, 5 row lock(s)
# 产生了4个锁对象结构，占用内存大小1136字节，5条记录被锁住
MySQL thread id 42, OS thread handle 123145566167040, query id 3222 localhost root starting
show engine innodb status
Trx read view will not see trx with id >= 1905, sees < 1905
# mvvc read view可见性，这个事务的read view 将不会看到事务id大于 1905的事务造成的变化，只能看到小于1905的
TABLE LOCK table `db_test`.`user` trx id 1905 lock mode IX
# 在user表上面有一个表锁，这个锁的模式为IX（排他意向锁）  
RECORD LOCKS space id 11 page no 6 n bits 80 index idx_age of table `db_test`.`user` trx id 1905 lock_mode X
# 在space id=11（a表的表空间），page no=6的页上，对表user上的idx_age索引加了记录锁，锁模式为：lock_mode X 也就是邻间锁 next-key lock
# 该页上面的位图锁占有80bits  
Record lock, heap no 8 PHYSICAL RECORD: n_fields 2; compact format; info bits 0
# heap no 8 的记录被锁住了  
 0: len 4; hex 8000000f; asc     ;;
# 这是第一条记录的二级索引，15被锁住
# 因为 next-key lock，所以会锁住记录本身，和它与上一条记录之间的间隙
# 所以这里锁定的是 (10,15]
 1: len 4; hex 8000000f; asc     ;;
# 二级索引指向的主键是15，所以主键值15也会被锁住

Record lock, heap no 9 PHYSICAL RECORD: n_fields 2; compact format; info bits 0
# heap no 9 的记录被锁住了，这是被锁住的第二条二级索引
 0: len 4; hex 8000000f; asc     ;;
# 这是是第二条记录的二级索引，同样是15
 1: len 4; hex 80000019; asc     ;;
# 二级索引指向的主键是25，所以主键25也会被锁住

RECORD LOCKS space id 11 page no 4 n bits 80 index PRIMARY of table `db_test`.`user` trx id 1905 lock_mode X locks rec but not gap
# 在space id=11（a表的表空间），page no=4的页上，对表user上的 primary(主键)索引加了记录锁，锁模式为：lock_mode X locks rec but not gap也就是普通行锁
# 因为上面两条记录的二级所以被锁住，所以对应的他们的主键索引也会被锁住
Record lock, heap no 8 PHYSICAL RECORD: n_fields 7; compact format; info bits 0
 0: len 4; hex 8000000f; asc     ;;
 # 被锁住的主键为 15，长度为4
 1: len 6; hex 0000000006e9; asc       ;;
 # 该字段为6个字节的事务id，这个id表示最近一次被更新的事务id，6e9十进制为 1769，也就是这条记录最后一次是被1769这个事务修改的
 2: len 7; hex 81000000c80137; asc       7;;
 # 这个是隐藏字段，为7个字节的回滚指针，用于mvcc
 3: len 4; hex 8000000f; asc     ;;
 # 这个是此记录的第2个字段 number，值为15，长度为4
 4: len 4; hex 8000000f; asc     ;;
 # 这个是此记录的第3个字段 age，值为15，长度为4
 5: len 1; hex 81; asc  ;;
 # 这个是此记录的第4个字段 sex，值为1，长度为1
 6: SQL NULL;
 # 这个是此记录的第5个字段 name，为null

Record lock, heap no 10 PHYSICAL RECORD: n_fields 7; compact format; info bits 0
# 这个是第二条记录的主键，同样也被锁住
 0: len 4; hex 80000019; asc     ;;
# 被锁住的主键为 25，长度为4
 1: len 6; hex 000000000700; asc       ;;
# 次记录最后一次被修改的事务id
 2: len 7; hex 82000000d00110; asc        ;;
# 回滚指针
 3: len 4; hex 80000019; asc     ;;
# 这个是此记录的第2个字段 number，值为25，长度为4
 4: len 4; hex 8000000f; asc     ;;
# 这个是此记录的第3个字段 age，值为15，长度为4
 5: len 1; hex 80; asc  ;;
# 这个是此记录的第4个字段 sex，值为1，长度为1
 6: SQL NULL;
 # 这个是此记录的第5个字段 name，为null

RECORD LOCKS space id 11 page no 6 n bits 80 index idx_age of table `db_test`.`user` trx id 1905 lock_mode X locks gap before rec
# # 在space id=11（a表的表空间），page no=6的页上，对表user上的 idx_age索引加了记录锁，锁模式为：lock_mode X locks gap before rec也就是间隙锁
Record lock, heap no 10 PHYSICAL RECORD: n_fields 2; compact format; info bits 0
 0: len 4; hex 80000014; asc     ;;
# 这是对应记录的二级索引,为20,
# 这里锁住的是20这个索引，和它上一个索引(也就是15)之间的间隙
# 所以这里的意思是锁住二级索引 idx_age (15,20)的这个区间，注意这个间隙锁，是不包括记录本身的
 1: len 4; hex 80000014; asc     ;;
# 这是二级索引锁对应的主键,同样也为20

```

所以根据上面的情况分析，我们得出最终锁定的区间是

- idx_age 索引
  - (10,15]
  - (15,20)
- 主键索引
  - [15]
  - [25]



### 普通行锁

这种是最简单的行锁了,意思就是仅仅把这一条记录对应的索引锁上。

```SQL
begin;
select * from user where id = 1 for update;
```
- 当我们使用主键索引`等值`查询时,会给这条记录的主键索引加上行锁

![](/img/record2.png)

```SQL
begin;
select * from user where number = 1 for update;
```
- 当使用唯一索引`等值`查询是，会给这条记录的主键索引和唯一索引分别添加行锁
![](/img/record3.png)

### 间隙锁 gap lock

>Gap locks in InnoDB are “purely inhibitive”, which means that their only purpose is to prevent other transactions from inserting to the gap
>
> InnoDB 中的间隙锁是“纯粹的抑制性”，这意味着它们的唯一目的是防止其他事务插入间隙

间隙锁锁的不是记录，而是记录和记录之间的间隙。举例来说明一下：

现在user表有2条记录

|id|	number	|age|	sex|
|---|---|---|---|
|1 |	1	|18	|1|
|5 | 5|	17|	0|

如果是给2条记录添加间隙锁，那么就会锁住2条记录id之间的间隙，也就是
(1-5)这个区间(暂时不考虑开闭区间的问题)。
如果有事务向表内插入id在这个区间内的记录，那么将会阻塞。

### 邻间锁 next-key lock

> A next-key lock is a combination of a record lock on the index record and a gap lock on the gap before the index record.
>
> next-key 锁是索引记录上的记录锁和索引记录之前的间隙上的间隙锁的组合。

所以邻间锁既会锁住记录，也会锁住这条记录和前后两条记录之间的间隙。(`所以叫邻间锁，相邻的间隙的意思`)。

>By default, InnoDB operates in REPEATABLE READ transaction isolation level. In this case, InnoDB uses next-key locks for searches and index scans, which prevents phantom rows (see Section 15.7.4, “Phantom Rows”).
>
> 默认情况下，InnoDB 在 REPEATABLE READ 事务隔离级别下运行。在这种情况下，InnoDB 使用 next-key 锁进行搜索和索引扫描，从而防止幻读。

我们这里先简单的了解一下概念，下面我们会根据不同的情况来分析。

接下来我们会分析一下，使用不同索引来加锁时的情况。

### 查询主键索引
首先我们来看根据主键索引查询的情况

#### 主键等值查询
```SQL
select * from user where id = 1 for update;
/** in 也算作等值查询 **/
select * from user where id in (1,2,,3) for update;

```

![](/img/record2.png)
这种是最简单一种情况，主键等值查询，会给查出记录的主键加普通行锁。

#### 主键范围查询

```


```




### 查询唯一索引


### 查询二级索引


### 查询不存在的主键索引




# 本篇文章使用的表结构

```SQL
CREATE TABLE `user` (
  `id` int NOT NULL,
  `number` int DEFAULT NULL,
  `age` int DEFAULT NULL,
  PRIMARY KEY (`id`),
  UNIQUE KEY `uk_number` (`number`) USING BTREE,
  KEY `idx_age` (`age`) USING BTREE
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_general_ci;

```

```SQL
CREATE TABLE `user_not_index` (
  `id` int NOT NULL,
  `number` int DEFAULT NULL,
  `age` int DEFAULT NULL
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_general_ci;

```

# 参考

[15.7.1 InnoDB Locking](https://dev.mysql.com/doc/refman/8.0/en/innodb-locking.html)
