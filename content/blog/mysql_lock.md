---
title: "MySQL的事务、日志、锁和MVCC"
date: 2021-07-28T10:53:40+08:00
author:     "zhouyang"
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

  >ACID里的AID都是数据库的特征,也就是依赖数据库的具体实现.而唯独这个C,实际上它依赖于应用层,也就是依赖于开发者.这里的一致性是指系统从一个正确的状态,迁移到另一个正确的状态.什么叫正确的状态呢?就是当前的状态满足预定的约束就叫做正确的状态.而事务具备ACID里C的特性是说通过事务的AID来保证我们的一致性.

`既从一个正确的状态迁移到另一个正确的状态`

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

### 行锁的种类

MySQL中为了实现不同的功能定义了多种行锁

- 普通行锁 ：lock_mode X/S locks rec but not gap
- 间隙锁   ：lock_mode X/S locks gap before rec
- 邻间锁   ：lock_mode X/S 
- 插入意向锁 : Insert Intention Locks
- 隐式锁

#### 普通行锁

普通行锁仅仅把一条记录锁锁住。

普通行锁也可以分为

- S型 共享锁
- X型 独占锁

当一个事务获取了一条记录的S型普通行锁后，其他事务也可以继续获取该记录的S型普通行锁，但不可以继续获取X型普通行锁；
当一个事务获取了一条记录的X型普通行锁后，其他事务既不可以继续获取该记录的S型普通行锁，也不可以继续获取X型普通行锁；

#### 间隙锁 Gap Lock


>Gap locks in InnoDB are “purely inhibitive”, which means that their only purpose is to prevent other transactions from inserting to the gap
>
> InnoDB 中的间隙锁是“纯粹的抑制性”，这意味着它们的唯一目的是防止其他事务插入间隙

MySQL中给Gap Lock的命名是：`locks gap before rec`，也就是锁住记录之前的间隙。

<span style="color: red;">需要注意的是，这个的之前，是指的指定索引顺序的之前之后，而不是表中记录的顺序。如果我们使用的主键索引，那么它的顺序会和表中数据一致，如果使用的是二级索引，那就不一定了。</span>

间隙锁锁的不是记录，而是记录和记录之间的间隙。举例来说明一下：

现在user表有2条记录

|id|  number  |age| sex|
|---|---|---|---|
|1  | 1 |18 |1  |
|5  | 5 |17 |0  |

我们现在给id=5的记录添加间隙锁，那么就会锁住id=5的记录和他前一条记录之前的间隙，也就是(1-5)这个区间(暂时不考虑开闭区间的问题)。
如果有事务向id(1,5)在这个区间内插入记录，那么将会阻塞。

那么问题来了，如果我想锁住第一条记录之前的间隙，和最后一条记录之后的间隙那改怎么办呢？

这时候就需要用到数据页中的两条伪记录

- Infimum记录，表示该页面中最小的记录。 给 id =1 加间隙锁，会锁住(-∞,1)
- Supremum记录，表示该页面中最大的记录。给 Supremum 加间隙锁，会锁住(5,+∞)


#### 邻间锁 Next-Key Locks

> A next-key lock is a combination of a record lock on the index record and a gap lock on the gap before the index record.
>
> next-key 锁是索引记录上的记录锁和索引记录之前的间隙上的间隙锁的组合。

所以邻间锁既会锁住记录，也会锁住这条记录和前面一条记录之间的间隙。(`所以叫邻间锁，相邻的间隙的意思`)。

>By default, InnoDB operates in REPEATABLE READ transaction isolation level. In this case, InnoDB uses next-key locks for searches and index scans, which prevents phantom rows (see Section 15.7.4, “Phantom Rows”).
>
> 默认情况下，InnoDB 在 REPEATABLE READ 事务隔离级别下运行。在这种情况下，InnoDB 使用 next-key 锁进行搜索和索引扫描，从而防止幻读。

邻间锁就可以理解为普通行锁和间隙锁的结合，既能锁住某一行，也能锁住记录之前的间隙。


#### 插入意向锁

一个事务在插入一条记录时需要判断一下插入位置是不是被别的事务加了间隙锁，如果有的话，插入操作需要等待，直到拥有gap锁的那个事务提交。
但是设计InnoDB的大叔规定事务在等待的时候也需要在内存中生成一个锁结构，表明有事务想在某个间隙中插入新记录，但是现在在等待。

所以插入意向锁的意思就是，有一个事务有意向往一个区间中插入记录，但是这个区间被其他事务锁定了，所以只能阻塞。

这个我们在后面单独分析Insert语句的加锁情况时会单独分析

### 隔离级别对行锁的影响

行锁和数据库使用的隔离界别有很大的关系

- 在`读已提交`下
  - 只有普通记录锁这一种
- 在`可重复读`下有
  - 普通记录锁
  - 间隙锁
  - 邻间锁
  - 插入意向锁

RC隔离级别下，只能锁住某一行记录，但是不是锁住记录之间的间隙，所以不能防止幻读。

#### 修改隔离级别

MySQL 5.x版本

```SQL
-- 查看全局、会话隔离级别
select @@global.tx_isolation,@@tx_isolation;
-- 设置会话隔离级别
set session transaction isolation level read committed;
-- 设置全局隔离级别
set global transaction isolation level read committed;

```

MySQL-8.0.3版本以后

`在MySQL 8.0.3 中，tx_isolation 变量被 transaction_isolation 变量替换了。`

```SQL
/**查看当前会话级别**/
SELECT @@global.transaction_isolation,@@session.transaction_isolation;
/**修改当前会话隔离级别**/
set session transaction isolation level repeatable read;
```

#### 不同隔离级别下的加锁情况

下面我们通过例子来说明一下，我们分别在隔离级别RR和RC下都执行下面的SQL,然后使用`show engine innodb status\G`查看锁状态。

```SQL
begin;
select * from user where age = 10 for update;

```

##### 可重复读

![](/img/rr-lock.png)
我们看到这里出现了3种行锁类型

##### 读已提交

![](/img/rc-lock2.png)

我们看到只有普通行锁一种锁类型


## 加锁情况分析

因为`读已提交`级别下的几所情况比较简单，只会给查询出的记录加普通行锁。
所以我们下面的分析中，都会在`可重复读`级别下进行。

### 开启MySQL锁监控

在分析之前，我们先解决一个问题。
有的同学可能也知道`show engine innodb status`这个命令，但是执行之后会发现，并没有输出这么多的锁信息。
这是因为`innodb_status_output_locks` 这个参数导致的。默认情况下是关闭的。我们需要打开它就可以看到详细的锁信息了。

```
SET GLOBAL innodb_status_output_locks=ON;

```

### show engine innodb status分析

下面我们通过一个针对二级索引加锁的SQL来说明一下

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

下面我们来逐行分析一下 TRANSACTION 部分

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
  - (10,15]  邻间锁
  - (15,20)  间隙锁
- 主键索引
  - [15]     普通行锁
  - [25]     普通行锁


### 查询主键索引

#### 主键等值查询
```SQL
select * from user where id = 1 for update;
/** in 也算作等值查询 **/
select * from user where id in (1,2,,3) for update;

```

![](/img/record2.png)
这种是最简单一种情况，主键等值查询，会给查出记录的主键加普通行锁。

#### 主键范围查询

```SQL
mysql> begin;
mysql> select * from user where id < 10 for update;
+----+--------+------+------+------+
| id | number | age  | sex  | name |
+----+--------+------+------+------+
|  1 |      1 |    1 |    0 | NULL |
|  3 |      3 |    3 |    1 | NULL |
|  4 |      4 |    4 |    1 | NULL |
|  5 |      5 |    5 |    1 | NULL |
|  7 |      7 |    4 |    1 | NULL |
+----+--------+------+------+------+
5 rows in set (0.00 sec)

mysql> show engine innodb status\G
```

```yml
3 lock struct(s), heap size 1136, 6 row lock(s)
MySQL thread id 43, OS thread handle 123145565257728, query id 3298 localhost root starting
show engine innodb status
Trx read view will not see trx with id >= 2008, sees < 2008
TABLE LOCK table `db_test`.`user` trx id 2008 lock mode IX
# 首先是表上的 排他意向锁
RECORD LOCKS space id 11 page no 4 n bits 80 index PRIMARY of table `db_test`.`user` trx id 2008 lock_mode X
# 然后是主键上的邻间锁
Record lock, heap no 2 PHYSICAL RECORD: n_fields 7; compact format; info bits 0
 0: len 4; hex 80000001; asc     ;;
# 对主键为1的索引加next-key lock，锁住的范围是 主键为1记录和上一条记录之间的间隙，和1这条记录本身。因为1之前没有记录了，
# 所以锁住范围是 (-∞,1]。
# 这里因为1之前没有记录了，所以使用的数据页中一条虚拟的记录:Infimum，用来表示负无穷
 1: len 6; hex 00000000073a; asc      :;;
# 最近更新的事务id
 2: len 7; hex 01000000f00c66; asc       f;;
# 回滚指针
 3: len 4; hex 80000001; asc     ;;
 4: len 4; hex 80000001; asc     ;;
 5: len 1; hex 80; asc  ;;
 6: SQL NULL;

Record lock, heap no 3 PHYSICAL RECORD: n_fields 7; compact format; info bits 0
 0: len 4; hex 80000003; asc     ;;
# 对主键为3的索引添加 next-key lock
# 锁住的范围为(1,3]
 1: len 6; hex 0000000006fc; asc       ;;
 2: len 7; hex 82000000ce0110; asc        ;;
 3: len 4; hex 80000003; asc     ;;
 4: len 4; hex 80000003; asc     ;;
 5: len 1; hex 81; asc  ;;
 6: SQL NULL;

Record lock, heap no 4 PHYSICAL RECORD: n_fields 7; compact format; info bits 0
 0: len 4; hex 80000004; asc     ;;
# 对主键为4的索引添加 next-key lock
# 锁住的范围为(3,4]
 1: len 6; hex 0000000006f3; asc       ;;
 2: len 7; hex 81000000cb0110; asc        ;;
 3: len 4; hex 80000004; asc     ;;
 4: len 4; hex 80000004; asc     ;;
 5: len 1; hex 81; asc  ;;
 6: SQL NULL;

Record lock, heap no 5 PHYSICAL RECORD: n_fields 7; compact format; info bits 0
 0: len 4; hex 80000005; asc     ;;
# 对主键为5的索引添加 next-key lock
# 锁住的范围为(4,5]
 1: len 6; hex 0000000006e9; asc       ;;
 2: len 7; hex 81000000c8011d; asc        ;;
 3: len 4; hex 80000005; asc     ;;
 4: len 4; hex 80000005; asc     ;;
 5: len 1; hex 81; asc  ;;
 6: SQL NULL;

Record lock, heap no 6 PHYSICAL RECORD: n_fields 7; compact format; info bits 0
 0: len 4; hex 80000007; asc     ;;
# 对主键为7的索引添加 next-key lock
# 锁住的范围为(5,7]
 1: len 6; hex 0000000006f4; asc       ;;
 2: len 7; hex 82000000cb0110; asc        ;;
 3: len 4; hex 80000007; asc     ;;
 4: len 4; hex 80000004; asc     ;;
 5: len 1; hex 81; asc  ;;
 6: SQL NULL;

RECORD LOCKS space id 11 page no 4 n bits 80 index PRIMARY of table `db_test`.`user` trx id 2008 lock_mode X locks gap before rec
# 这里又对主键，添加了一个 gap lock
Record lock, heap no 7 PHYSICAL RECORD: n_fields 7; compact format; info bits 0
 0: len 4; hex 8000000a; asc     ;;
# 对主键为10的索引添加 gap lock
# 锁住的范围是 (7,10)
 1: len 6; hex 0000000006e9; asc       ;;
 2: len 7; hex 81000000c8012a; asc       *;;
 3: len 4; hex 8000000a; asc     ;;
 4: len 4; hex 8000000a; asc     ;;
 5: len 1; hex 81; asc  ;;
 6: SQL NULL;

```

综上，我们可以得出针对 `select * from user where id < 10 for update;`这个语句，锁住的区间是

- (-∞,1]
- (1,3]
- (3,4]
- (4,5]
- (5,7]
- (7,10)

其中 [1,3,4,5,7]都是sql查询出来的主键，所以我们大概可以得出一个结论就是

- 主键范围查询，会锁住查询出来的主键之间的间隙，以及查询出来的记录
- 另外还会锁住查询出来的最小的记录和上一条记录的间隙，以及最大的记录和下一条记录的间隙。
  - 如果没有上一小记录，就是到负无穷
  - 如果没有下一条，就是到正无穷


下面我们来验证一下

```SQL
mysql> begin;
mysql> select * from user where id >7 and id < 20 for update;
+----+--------+------+------+------+
| id | number | age  | sex  | name |
+----+--------+------+------+------+
| 10 |     10 |   10 |    1 | NULL |
| 15 |     15 |   15 |    1 | NULL |
+----+--------+------+------+------+
```

根据上面的结论可以得出，最终锁定的区间是

- （7,10]
- (10,15]
- (15,20)

查后我们查看锁状态

```yml
3 lock struct(s), heap size 1136, 3 row lock(s)
MySQL thread id 43, OS thread handle 123145565257728, query id 3358 localhost root starting
show engine innodb status
Trx read view will not see trx with id >= 2009, sees < 2009
TABLE LOCK table `db_test`.`user` trx id 2009 lock mode IX
RECORD LOCKS space id 11 page no 4 n bits 80 index PRIMARY of table `db_test`.`user` trx id 2009 lock_mode X
# 首先是 next-key lock
Record lock, heap no 7 PHYSICAL RECORD: n_fields 7; compact format; info bits 0
 0: len 4; hex 8000000a; asc     ;;
# 主键10 添加 next-key lock
# 区间为(7,10]
 1: len 6; hex 0000000006e9; asc       ;;
 2: len 7; hex 81000000c8012a; asc       *;;
 3: len 4; hex 8000000a; asc     ;;
 4: len 4; hex 8000000a; asc     ;;
 5: len 1; hex 81; asc  ;;
 6: SQL NULL;

Record lock, heap no 8 PHYSICAL RECORD: n_fields 7; compact format; info bits 0
 0: len 4; hex 8000000f; asc     ;;
# 主键15 添加 next-key lock
# 区间为(10,15]
 1: len 6; hex 0000000006e9; asc       ;;
 2: len 7; hex 81000000c80137; asc       7;;
 3: len 4; hex 8000000f; asc     ;;
 4: len 4; hex 8000000f; asc     ;;
 5: len 1; hex 81; asc  ;;
 6: SQL NULL;

RECORD LOCKS space id 11 page no 4 n bits 80 index PRIMARY of table `db_test`.`user` trx id 2009 lock_mode X locks gap before rec
# 然后是 间隙锁
Record lock, heap no 9 PHYSICAL RECORD: n_fields 7; compact format; info bits 0
 0: len 4; hex 80000014; asc     ;;
# 主键20 添加间隙锁
 1: len 6; hex 0000000006e9; asc       ;;
 2: len 7; hex 81000000c80144; asc       D;;
 3: len 4; hex 80000014; asc     ;;
 4: len 4; hex 80000014; asc     ;;
 5: len 1; hex 81; asc  ;;
 6: SQL NULL;

```

ok 全中！说明这个结论是符合的。

### 查询唯一索引

#### 唯一索引的等值查询

```SQL
mysql> begin;
Query OK, 0 rows affected (0.17 sec)

mysql> select * from user where number = 10 for update;
+----+--------+------+------+------+
| id | number | age  | sex  | name |
+----+--------+------+------+------+
| 10 |     10 |   10 |    1 | NULL |
+----+--------+------+------+------+
1 row in set (0.03 sec)

mysql> show engine innodb status\G
```

```yml
3 lock struct(s), heap size 1136, 2 row lock(s)
MySQL thread id 43, OS thread handle 123145565257728, query id 3362 localhost root starting
show engine innodb status
TABLE LOCK table `db_test`.`user` trx id 2010 lock mode IX
# 表上加一项排它锁
RECORD LOCKS space id 11 page no 5 n bits 80 index uk_number of table `db_test`.`user` trx id 2010 lock_mode X locks rec but not gap
# 在唯一索引上添加 普通记录锁
Record lock, heap no 7 PHYSICAL RECORD: n_fields 2; compact format; info bits 0
 0: len 4; hex 8000000a; asc     ;;
 # 唯一索引
 1: len 4; hex 8000000a; asc     ;;
 # 唯一索引，对应的主键索引

RECORD LOCKS space id 11 page no 4 n bits 80 index PRIMARY of table `db_test`.`user` trx id 2010 lock_mode X locks rec but not gap
# 在主键上添加 普通记录锁
Record lock, heap no 7 PHYSICAL RECORD: n_fields 7; compact format; info bits 0
 0: len 4; hex 8000000a; asc     ;;
 1: len 6; hex 0000000006e9; asc       ;;
 2: len 7; hex 81000000c8012a; asc       *;;
 3: len 4; hex 8000000a; asc     ;;
 4: len 4; hex 8000000a; asc     ;;
 5: len 1; hex 81; asc  ;;
 6: SQL NULL;

```

我们看到唯一索引的等值查询和主键索引和相似，都是对记录所在的索引添加普通记录锁。
只不过会多添加一个对唯一索引的记录锁。

`通过这里我们可以发现，当我们使用不同的索引进行查询的时候，虽然查询的记录有可能是同一条，但是加锁情况是不尽相同的。原因就是InnoDB的锁时加载索引上，而不是记录上的。`

#### 唯一索引的范围查询

```SQL
mysql> begin;
Query OK, 0 rows affected (0.00 sec)

mysql> select * from user where number > 10 for update;
+----+--------+------+------+------+
| id | number | age  | sex  | name |
+----+--------+------+------+------+
| 15 |     15 |   15 |    1 | NULL |
| 20 |     20 |   20 |    1 | NULL |
| 25 |     25 |   15 |    0 | NULL |
+----+--------+------+------+------+
3 rows in set (0.00 sec)

mysql> show engine innodb status\G
```

在查看锁状态之前我们先做一个猜测

- 既然唯一索引的等值查询，是和主键基本类似，只是多加了一个对唯一索引的锁
- 那么范围查询会不会也是一样呢，只是在主键的基础上，添加对唯一索引一样的逻辑呢？

下面我们来验证一下

```yml
3 lock struct(s), heap size 1136, 7 row lock(s)
MySQL thread id 43, OS thread handle 123145565257728, query id 3371 localhost root starting
show engine innodb status
Trx read view will not see trx with id >= 2012, sees < 2012
TABLE LOCK table `db_test`.`user` trx id 2012 lock mode IX
# 表上添加意向排他锁
RECORD LOCKS space id 11 page no 5 n bits 80 index uk_number of table `db_test`.`user` trx id 2012 lock_mode X
#在uk_number 上添加 next-key lock
Record lock, heap no 1 PHYSICAL RECORD: n_fields 1; compact format; info bits 0
 0: len 8; hex 73757072656d756d; asc supremum;;
# supremum 是数据页中虚拟的最大值,一定会大于所有记录的索引值，给这个索引值添加 next-key lock
# 表示锁上的区间为查询出来的记录中索引最大值，到正无穷，也就是(25,+∞)
Record lock, heap no 8 PHYSICAL RECORD: n_fields 2; compact format; info bits 0
 0: len 4; hex 8000000f; asc     ;;
# 给查询出来的记录的唯一索引加锁，唯一索引为 15
# 范围是 (10,15]
 1: len 4; hex 8000000f; asc     ;;

Record lock, heap no 9 PHYSICAL RECORD: n_fields 2; compact format; info bits 0
 0: len 4; hex 80000014; asc     ;;
 # 给查询出来的记录的唯一索引加锁，唯一索引为 20
 # 范围是 (15,20]
 1: len 4; hex 80000014; asc     ;;

Record lock, heap no 10 PHYSICAL RECORD: n_fields 2; compact format; info bits 0
 0: len 4; hex 80000019; asc     ;;
 # 给查询出来的记录的唯一索引加锁，唯一索引为 25
 # 范围是 (20,25]
 1: len 4; hex 80000019; asc     ;;

RECORD LOCKS space id 11 page no 4 n bits 80 index PRIMARY of table `db_test`.`user` trx id 2012 lock_mode X locks rec but not gap
# 给查询出来的对应记录的主键加锁，注意这里添加是不是 next-key Lock了，而是普通的记录锁了
Record lock, heap no 8 PHYSICAL RECORD: n_fields 7; compact format; info bits 0
 0: len 4; hex 8000000f; asc     ;;
# 给值15的主键索引加锁
 1: len 6; hex 0000000006e9; asc       ;;
 2: len 7; hex 81000000c80137; asc       7;;
 3: len 4; hex 8000000f; asc     ;;
 4: len 4; hex 8000000f; asc     ;;
 5: len 1; hex 81; asc  ;;
 6: SQL NULL;

Record lock, heap no 9 PHYSICAL RECORD: n_fields 7; compact format; info bits 0
 0: len 4; hex 80000014; asc     ;;
# 给值20的主键索引加锁
 1: len 6; hex 0000000006e9; asc       ;;
 2: len 7; hex 81000000c80144; asc       D;;
 3: len 4; hex 80000014; asc     ;;
 4: len 4; hex 80000014; asc     ;;
 5: len 1; hex 81; asc  ;;
 6: SQL NULL;

Record lock, heap no 10 PHYSICAL RECORD: n_fields 7; compact format; info bits 0
 0: len 4; hex 80000019; asc     ;;
# 给值25的主键索引加锁
 1: len 6; hex 000000000700; asc       ;;
 2: len 7; hex 82000000d00110; asc        ;;
 3: len 4; hex 80000019; asc     ;;
 4: len 4; hex 8000000f; asc     ;;
 5: len 1; hex 80; asc  ;;
 6: SQL NULL;

```

ok，现在结论出来了，`select * from user where number > 10 for update`的加锁结果如下

- 唯一索引
  - (10,15]
  - (15,20]
  - (20,25]
  - (25,+∞]
- 主键索引
  - [15]
  - [20]
  - [25]

和我们刚刚预测的有些许不同

- 唯一索引确实是按照刚才的逻辑添加的锁。
  - 但又不是完全是，下面我们用另一个sql再测试一下
- 主键只是对查询出来的那些记录的主键，添加了普通记录锁。


在一次测试，这次我们使用和上面主键查询一样的范围

```SQL
mysql> begin;
mysql> select * from user where number < 10 for update;
+----+--------+------+------+------+
| id | number | age  | sex  | name |
+----+--------+------+------+------+
|  1 |      1 |    1 |    0 | NULL |
|  3 |      3 |    3 |    1 | NULL |
|  4 |      4 |    4 |    1 | NULL |
|  5 |      5 |    5 |    1 | NULL |
|  7 |      7 |    4 |    1 | NULL |
+----+--------+------+------+------+
5 rows in set (0.00 sec)

mysql> show engine innodb status\G

```

锁状态如下：

```yml
3 lock struct(s), heap size 1136, 11 row lock(s)
MySQL thread id 43, OS thread handle 123145565257728, query id 3386 localhost root starting
show engine innodb status
TABLE LOCK table `db_test`.`user` trx id 2015 lock mode IX
RECORD LOCKS space id 11 page no 5 n bits 80 index uk_number of table `db_test`.`user` trx id 2015 lock_mode X
Record lock, heap no 2 PHYSICAL RECORD: n_fields 2; compact format; info bits 0
 0: len 4; hex 80000001; asc     ;;
 1: len 4; hex 80000001; asc     ;;

Record lock, heap no 3 PHYSICAL RECORD: n_fields 2; compact format; info bits 0
 0: len 4; hex 80000003; asc     ;;
 1: len 4; hex 80000003; asc     ;;

Record lock, heap no 4 PHYSICAL RECORD: n_fields 2; compact format; info bits 0
 0: len 4; hex 80000004; asc     ;;
 1: len 4; hex 80000004; asc     ;;

Record lock, heap no 5 PHYSICAL RECORD: n_fields 2; compact format; info bits 0
 0: len 4; hex 80000005; asc     ;;
 1: len 4; hex 80000005; asc     ;;

Record lock, heap no 6 PHYSICAL RECORD: n_fields 2; compact format; info bits 0
 0: len 4; hex 80000007; asc     ;;
 1: len 4; hex 80000007; asc     ;;

Record lock, heap no 7 PHYSICAL RECORD: n_fields 2; compact format; info bits 0
 0: len 4; hex 8000000a; asc     ;;
 1: len 4; hex 8000000a; asc     ;;

RECORD LOCKS space id 11 page no 4 n bits 80 index PRIMARY of table `db_test`.`user` trx id 2015 lock_mode X locks rec but not gap
Record lock, heap no 2 PHYSICAL RECORD: n_fields 7; compact format; info bits 0
 0: len 4; hex 80000001; asc     ;;
 1: len 6; hex 00000000073a; asc      :;;
 2: len 7; hex 01000000f00c66; asc       f;;
 3: len 4; hex 80000001; asc     ;;
 4: len 4; hex 80000001; asc     ;;
 5: len 1; hex 80; asc  ;;
 6: SQL NULL;

Record lock, heap no 3 PHYSICAL RECORD: n_fields 7; compact format; info bits 0
 0: len 4; hex 80000003; asc     ;;
 1: len 6; hex 0000000006fc; asc       ;;
 2: len 7; hex 82000000ce0110; asc        ;;
 3: len 4; hex 80000003; asc     ;;
 4: len 4; hex 80000003; asc     ;;
 5: len 1; hex 81; asc  ;;
 6: SQL NULL;

Record lock, heap no 4 PHYSICAL RECORD: n_fields 7; compact format; info bits 0
 0: len 4; hex 80000004; asc     ;;
 1: len 6; hex 0000000006f3; asc       ;;
 2: len 7; hex 81000000cb0110; asc        ;;
 3: len 4; hex 80000004; asc     ;;
 4: len 4; hex 80000004; asc     ;;
 5: len 1; hex 81; asc  ;;
 6: SQL NULL;

Record lock, heap no 5 PHYSICAL RECORD: n_fields 7; compact format; info bits 0
 0: len 4; hex 80000005; asc     ;;
 1: len 6; hex 0000000006e9; asc       ;;
 2: len 7; hex 81000000c8011d; asc        ;;
 3: len 4; hex 80000005; asc     ;;
 4: len 4; hex 80000005; asc     ;;
 5: len 1; hex 81; asc  ;;
 6: SQL NULL;

Record lock, heap no 6 PHYSICAL RECORD: n_fields 7; compact format; info bits 0
 0: len 4; hex 80000007; asc     ;;
 1: len 6; hex 0000000006f4; asc       ;;
 2: len 7; hex 82000000cb0110; asc        ;;
 3: len 4; hex 80000007; asc     ;;
 4: len 4; hex 80000004; asc     ;;
 5: len 1; hex 81; asc  ;;
 6: SQL NULL;

```
 这里的大部分和[主键范围查询](/blog/mysql_lock/#主键范围查询)是一致的。

 <span style="color:green">唯一的不同就是针对最后一条唯一索引，并没有使用 gap lock</span>

 主键范围查询针对最后一条记录是一个 间隙锁

 ```yml
 RECORD LOCKS space id 11 page no 4 n bits 80 index PRIMARY of table `db_test`.`user` trx id 2008 lock_mode X locks gap before rec
# 这里又对主键，添加了一个 gap lock
Record lock, heap no 7 PHYSICAL RECORD: n_fields 7; compact format; info bits 0
 0: len 4; hex 8000000a; asc     ;;
# 对主键为10的索引添加 gap lock
# 锁住的范围是 (7,10)
 1: len 6; hex 0000000006e9; asc       ;;
 2: len 7; hex 81000000c8012a; asc       *;;
 3: len 4; hex 8000000a; asc     ;;
 4: len 4; hex 8000000a; asc     ;;
 5: len 1; hex 81; asc  ;;
 6: SQL NULL;

 ```

 唯一索引是这样的，是一个next-key lock

```yml
RECORD LOCKS space id 11 page no 5 n bits 80 index uk_number of table `db_test`.`user` trx id 2015 lock_mode X
Record lock, heap no 7 PHYSICAL RECORD: n_fields 2; compact format; info bits 0
 0: len 4; hex 8000000a; asc     ;;
 1: len 4; hex 8000000a; asc     ;;
```

简单来说就是 主键查询和 唯一索引查出来的记录都是 [1,3,4,5,7]。id为7点下一条也同样都是 10。
只不过，针对 10这条记录的索引

- 主键加的是一个 gap lock，不包括 10这条记录
- 唯一索引，假的是 next-key lock，包括10这条记录。
- 也就是使用唯一索引和主键索引当查询出的记录一致的情况下，唯一索引会比主键的锁范围大一点。

所以唯一索引范围查询的结论需要更新一下

- 对查询出来的记录的唯一索引，添加next-key lock，也就是锁住记录，和上一条记录值之间的间隙
- 对查询出记录中唯一索引最大的，下一个索引，添加next-key lock，
  - 比如这次查询出来的记录中最大的是7，那么对7的下一个10（索引顺序，非记录顺序），添加 next-key lock
  - 也就(7,10],这里虽然没有查出来10，但是也把是10锁住。
- 对查询出来记录的主键，添加普通记录锁

![](/img/uk1.png)
我们先看唯一索引的操作，发现右边的左右被阻塞了。

![](/img/uk2.png)
我们再看针对主键的操作，发现右边的操作没有被阻塞。


### 查询二级索引

#### 二级索引的等值查询

这个可以直接参考[show-engine-innodb-status分析](/blog/mysql_lock/#show-engine-innodb-status分析)。我们这里总结一下结论

比如使用idx_age 查询的

- 给查询出来记录的idx_age索引添加next-key lock
  - 也就是会锁住该条记录的二级索引
  - <span style="color:red">和这个二级索引，和它上一个二级索引之间的间隙(这个上一条的概念，不是记录中的上一条，而是在索引结构中的上一个索引)</span>
  - 这个我觉得是可以理解的，因为记录的顺序是和聚簇索引保持一致的，而一个表中只会有一个聚簇索引，那就是主键。所以二级索引的顺序必然和记录顺序不一致。
  - 而innodb的索引是加在索引上的，所以在对二级索引加锁的时候，必然要按照二级索引的顺序来处理。
- 对查询出记录中索引值最大的，下一个索引，添加gap lock
  - 比如样例中，查出来记录的索引最大为15，它的下一个索引为20
  - 也就是(15,20)
- 给查询出来记录的主键索引，添加普通记录锁

#### 二级索引使用范围查询

```SQL
mysql> begin;
Query OK, 0 rows affected (0.00 sec)

mysql> select * from user where age >10 for update;
+----+--------+------+------+------+
| id | number | age  | sex  | name |
+----+--------+------+------+------+
| 15 |     15 |   15 |    1 | NULL |
| 25 |     25 |   15 |    0 | NULL |
| 20 |     20 |   20 |    1 | NULL |
+----+--------+------+------+------+
3 rows in set (0.01 sec)

mysql> show engine innodb status\G
```
在查看之前我们还是结合着上面唯一索引范围查询的处理逻辑，预测一下


- 对于查询出来记录的二级索引，添加next-key lock
- 对于查出来记录中索引值最大的，的下一条索引，同样添加 next-key lock
- 针对主键索引，只会对查询出来的记录加普通行锁

来验证一下

```yml

3 lock struct(s), heap size 1136, 7 row lock(s)
MySQL thread id 43, OS thread handle 123145565257728, query id 3377 localhost root starting
show engine innodb status
TABLE LOCK table `db_test`.`user` trx id 2013 lock mode IX
# 表上添加排他意向锁
RECORD LOCKS space id 11 page no 6 n bits 80 index idx_age of table `db_test`.`user` trx id 2013 lock_mode X
# 给idx_age 添加 next-key lock
Record lock, heap no 1 PHYSICAL RECORD: n_fields 1; compact format; info bits 0
 0: len 8; hex 73757072656d756d; asc supremum;;
# 和上面唯一索引一样的逻辑，锁住记录中age最大值，到正无穷的区间
# (20,+∞)
Record lock, heap no 8 PHYSICAL RECORD: n_fields 2; compact format; info bits 0
 0: len 4; hex 8000000f; asc     ;;
# 给idx_age 为15 的索引 添加 next-key lock
# 锁住的区间是 (10,15]
 1: len 4; hex 8000000f; asc     ;;

Record lock, heap no 9 PHYSICAL RECORD: n_fields 2; compact format; info bits 0
 0: len 4; hex 8000000f; asc     ;;
 # 给另一个idx_age 为15 的索引 添加 next-key lock
 # 虽然索引值都是15，但是毕竟是2个索引，所以都要加锁
 # 锁住的区间是 (15,15]
 1: len 4; hex 80000019; asc     ;;

Record lock, heap no 10 PHYSICAL RECORD: n_fields 2; compact format; info bits 0
 0: len 4; hex 80000014; asc     ;;
 # 给idx_age 为20 的索引 添加 next-key lock
 # 锁住的区间是 (15,20]
 1: len 4; hex 80000014; asc     ;;

RECORD LOCKS space id 11 page no 4 n bits 80 index PRIMARY of table `db_test`.`user` trx id 2013 lock_mode X locks rec but not gap
# 给主键添加普通行锁，下面就不一一写了，参照上面的即可
Record lock, heap no 8 PHYSICAL RECORD: n_fields 7; compact format; info bits 0
 0: len 4; hex 8000000f; asc     ;;
 1: len 6; hex 0000000006e9; asc       ;;
 2: len 7; hex 81000000c80137; asc       7;;
 3: len 4; hex 8000000f; asc     ;;
 4: len 4; hex 8000000f; asc     ;;
 5: len 1; hex 81; asc  ;;
 6: SQL NULL;

Record lock, heap no 9 PHYSICAL RECORD: n_fields 7; compact format; info bits 0
 0: len 4; hex 80000014; asc     ;;
 1: len 6; hex 0000000006e9; asc       ;;
 2: len 7; hex 81000000c80144; asc       D;;
 3: len 4; hex 80000014; asc     ;;
 4: len 4; hex 80000014; asc     ;;
 5: len 1; hex 81; asc  ;;
 6: SQL NULL;

Record lock, heap no 10 PHYSICAL RECORD: n_fields 7; compact format; info bits 0
 0: len 4; hex 80000019; asc     ;;
 1: len 6; hex 000000000700; asc       ;;
 2: len 7; hex 82000000d00110; asc        ;;
 3: len 4; hex 80000019; asc     ;;
 4: len 4; hex 8000000f; asc     ;;
 5: len 1; hex 80; asc  ;;
 6: SQL NULL;

```

首先我们看到，行锁中只有两类，next-key lock 和 普通行锁
下面关注看 next-key lock

- 首先查询出来的3条记录的索引全部被加上了next-key lock 没有问题

下面我们来看猜测规则的第二条，`给记录中索引值最大的下一条索引，添加next-key lock`。

记录中二级索引最大的是20，在user表中，已经没有其他记录的idx_age索引比它更大了，InnoDB给数据页中的最大虚拟记录的索引添加了 next-key lock。

```yml
Record lock, heap no 1 PHYSICAL RECORD: n_fields 1; compact format; info bits 0
 0: len 8; hex 73757072656d756d; asc supremum;;
```

ok，这样说明我们猜测的结论还是正确的

- 对于查询出来记录的二级索引，添加next-key lock
- 对于查出来记录中索引值最大的，的下一条索引，同样添加 next-key lock
  - 如果没有下一条索引了，则给`supremum`添加 next-key lock
- 针对主键索引，只会对查询出来的记录加普通行锁


### 不使用索引进行查询


```SQL
begin;
Query OK, 0 rows affected (0.00 sec)

mysql> select * from user where sex = 1 for update;
+----+--------+------+------+------+
| id | number | age  | sex  | name |
+----+--------+------+------+------+
|  3 |      3 |    3 |    1 | NULL |
|  4 |      4 |    4 |    1 | NULL |
|  5 |      5 |    5 |    1 | NULL |
|  7 |      7 |    4 |    1 | NULL |
| 10 |     10 |   10 |    1 | NULL |
| 15 |     15 |   15 |    1 | NULL |
| 20 |     20 |   20 |    1 | NULL |
+----+--------+------+------+------+
7 rows in set (0.00 sec)

mysql> show engine innodb status\G
```

```yml
2 lock struct(s), heap size 1136, 10 row lock(s)
MySQL thread id 43, OS thread handle 123145565257728, query id 3427 localhost root starting
show engine innodb status
TABLE LOCK table `db_test`.`user` trx id 2023 lock mode IX
# 排他意向锁
RECORD LOCKS space id 11 page no 4 n bits 80 index PRIMARY of table `db_test`.`user` trx id 2023 lock_mode X
# 给主键添加 next-key lock
Record lock, heap no 1 PHYSICAL RECORD: n_fields 1; compact format; info bits 0
 0: len 8; hex 73757072656d756d; asc supremum;;
# 给最大值添加 next-key lock
Record lock, heap no 2 PHYSICAL RECORD: n_fields 7; compact format; info bits 0
 0: len 4; hex 80000001; asc     ;;
 1: len 6; hex 00000000073a; asc      :;;
 2: len 7; hex 01000000f00c66; asc       f;;
 3: len 4; hex 80000001; asc     ;;
 4: len 4; hex 80000001; asc     ;;
 5: len 1; hex 80; asc  ;;
 6: SQL NULL;

Record lock, heap no 3 PHYSICAL RECORD: n_fields 7; compact format; info bits 0
 0: len 4; hex 80000003; asc     ;;
 1: len 6; hex 0000000006fc; asc       ;;
 2: len 7; hex 82000000ce0110; asc        ;;
 3: len 4; hex 80000003; asc     ;;
 4: len 4; hex 80000003; asc     ;;
 5: len 1; hex 81; asc  ;;
 6: SQL NULL;

Record lock, heap no 4 PHYSICAL RECORD: n_fields 7; compact format; info bits 0
 0: len 4; hex 80000004; asc     ;;
 1: len 6; hex 0000000006f3; asc       ;;
 2: len 7; hex 81000000cb0110; asc        ;;
 3: len 4; hex 80000004; asc     ;;
 4: len 4; hex 80000004; asc     ;;
 5: len 1; hex 81; asc  ;;
 6: SQL NULL;

Record lock, heap no 5 PHYSICAL RECORD: n_fields 7; compact format; info bits 0
 0: len 4; hex 80000005; asc     ;;
 1: len 6; hex 0000000006e9; asc       ;;
 2: len 7; hex 81000000c8011d; asc        ;;
 3: len 4; hex 80000005; asc     ;;
 4: len 4; hex 80000005; asc     ;;
 5: len 1; hex 81; asc  ;;
 6: SQL NULL;

Record lock, heap no 6 PHYSICAL RECORD: n_fields 7; compact format; info bits 0
 0: len 4; hex 80000007; asc     ;;
 1: len 6; hex 0000000006f4; asc       ;;
 2: len 7; hex 82000000cb0110; asc        ;;
 3: len 4; hex 80000007; asc     ;;
 4: len 4; hex 80000004; asc     ;;
 5: len 1; hex 81; asc  ;;
 6: SQL NULL;

Record lock, heap no 7 PHYSICAL RECORD: n_fields 7; compact format; info bits 0
 0: len 4; hex 8000000a; asc     ;;
 1: len 6; hex 0000000006e9; asc       ;;
 2: len 7; hex 81000000c8012a; asc       *;;
 3: len 4; hex 8000000a; asc     ;;
 4: len 4; hex 8000000a; asc     ;;
 5: len 1; hex 81; asc  ;;
 6: SQL NULL;

Record lock, heap no 8 PHYSICAL RECORD: n_fields 7; compact format; info bits 0
 0: len 4; hex 8000000f; asc     ;;
 1: len 6; hex 0000000006e9; asc       ;;
 2: len 7; hex 81000000c80137; asc       7;;
 3: len 4; hex 8000000f; asc     ;;
 4: len 4; hex 8000000f; asc     ;;
 5: len 1; hex 81; asc  ;;
 6: SQL NULL;

Record lock, heap no 9 PHYSICAL RECORD: n_fields 7; compact format; info bits 0
 0: len 4; hex 80000014; asc     ;;
 1: len 6; hex 0000000006e9; asc       ;;
 2: len 7; hex 81000000c80144; asc       D;;
 3: len 4; hex 80000014; asc     ;;
 4: len 4; hex 80000014; asc     ;;
 5: len 1; hex 81; asc  ;;
 6: SQL NULL;

Record lock, heap no 10 PHYSICAL RECORD: n_fields 7; compact format; info bits 0
 0: len 4; hex 80000019; asc     ;;
 1: len 6; hex 000000000700; asc       ;;
 2: len 7; hex 82000000d00110; asc        ;;
 3: len 4; hex 80000019; asc     ;;
 4: len 4; hex 8000000f; asc     ;;
 5: len 1; hex 80; asc  ;;
 6: SQL NULL;

```

我们看到，当使用非索引字段查询，并加锁时

- 会给主键添加 next-key lock
- 然后最重要的一点是，会锁住所有的记录和间隙
  - 所以最终的锁定区间是
  - (-∞,1]
  - ... n条记录和它们的间隙
  - (20,+∞)


那还有另一种情况，如果这个表中连主键都没有呢？

```SQL
mysql> begin;
Query OK, 0 rows affected (0.04 sec)

mysql> select * from user_not_index where number = 1 for update;
+----+--------+------+------+------+
| id | number | age  | sex  | name |
+----+--------+------+------+------+
|  1 |      1 |    1 |    0 | NULL |
+----+--------+------+------+------+
1 row in set (0.02 sec)

mysql> show engine innodb status\G

```

```yml
2 lock struct(s), heap size 1136, 10 row lock(s)
MySQL thread id 43, OS thread handle 123145565257728, query id 3421 localhost root starting
show engine innodb status
TABLE LOCK table `db_test`.`user_not_index` trx id 2022 lock mode IX
# 表上添加意向排他锁
RECORD LOCKS space id 12 page no 4 n bits 80 index GEN_CLUST_INDEX of table `db_test`.`user_not_index` trx id 2022 lock_mode X
# 给GEN_CLUST_INDEX 索引添加 next-key lock
Record lock, heap no 1 PHYSICAL RECORD: n_fields 1; compact format; info bits 0
 0: len 8; hex 73757072656d756d; asc supremum;;

Record lock, heap no 2 PHYSICAL RECORD: n_fields 8; compact format; info bits 0
 0: len 6; hex 000000000200; asc       ;;
 1: len 6; hex 00000000075f; asc      _;;
 2: len 7; hex 820000013b0110; asc     ;  ;;
 3: len 4; hex 80000001; asc     ;;
 4: len 4; hex 80000001; asc     ;;
 5: len 4; hex 80000001; asc     ;;
 6: len 1; hex 80; asc  ;;
 7: SQL NULL;

Record lock, heap no 3 PHYSICAL RECORD: n_fields 8; compact format; info bits 0
 0: len 6; hex 000000000201; asc       ;;
 1: len 6; hex 00000000075f; asc      _;;
 2: len 7; hex 820000013b011f; asc     ;  ;;
 3: len 4; hex 80000003; asc     ;;
 4: len 4; hex 80000003; asc     ;;
 5: len 4; hex 80000003; asc     ;;
 6: len 1; hex 81; asc  ;;
 7: SQL NULL;

Record lock, heap no 4 PHYSICAL RECORD: n_fields 8; compact format; info bits 0
 0: len 6; hex 000000000202; asc       ;;
 1: len 6; hex 00000000075f; asc      _;;
 2: len 7; hex 820000013b012e; asc     ; .;;
 3: len 4; hex 80000004; asc     ;;
 4: len 4; hex 80000004; asc     ;;
 5: len 4; hex 80000004; asc     ;;
 6: len 1; hex 81; asc  ;;
 7: SQL NULL;

Record lock, heap no 5 PHYSICAL RECORD: n_fields 8; compact format; info bits 0
 0: len 6; hex 000000000203; asc       ;;
 1: len 6; hex 00000000075f; asc      _;;
 2: len 7; hex 820000013b013d; asc     ; =;;
 3: len 4; hex 80000005; asc     ;;
 4: len 4; hex 80000005; asc     ;;
 5: len 4; hex 80000005; asc     ;;
 6: len 1; hex 81; asc  ;;
 7: SQL NULL;

Record lock, heap no 6 PHYSICAL RECORD: n_fields 8; compact format; info bits 0
 0: len 6; hex 000000000204; asc       ;;
 1: len 6; hex 00000000075f; asc      _;;
 2: len 7; hex 820000013b014c; asc     ; L;;
 3: len 4; hex 80000007; asc     ;;
 4: len 4; hex 80000007; asc     ;;
 5: len 4; hex 80000004; asc     ;;
 6: len 1; hex 81; asc  ;;
 7: SQL NULL;

Record lock, heap no 7 PHYSICAL RECORD: n_fields 8; compact format; info bits 0
 0: len 6; hex 000000000205; asc       ;;
 1: len 6; hex 00000000075f; asc      _;;
 2: len 7; hex 820000013b015b; asc     ; [;;
 3: len 4; hex 8000000a; asc     ;;
 4: len 4; hex 8000000a; asc     ;;
 5: len 4; hex 8000000a; asc     ;;
 6: len 1; hex 81; asc  ;;
 7: SQL NULL;

Record lock, heap no 8 PHYSICAL RECORD: n_fields 8; compact format; info bits 0
 0: len 6; hex 000000000206; asc       ;;
 1: len 6; hex 00000000075f; asc      _;;
 2: len 7; hex 820000013b016a; asc     ; j;;
 3: len 4; hex 8000000f; asc     ;;
 4: len 4; hex 8000000f; asc     ;;
 5: len 4; hex 8000000f; asc     ;;
 6: len 1; hex 81; asc  ;;
 7: SQL NULL;

Record lock, heap no 9 PHYSICAL RECORD: n_fields 8; compact format; info bits 0
 0: len 6; hex 000000000207; asc       ;;
 1: len 6; hex 00000000075f; asc      _;;
 2: len 7; hex 820000013b0179; asc     ; y;;
 3: len 4; hex 80000014; asc     ;;
 4: len 4; hex 80000014; asc     ;;
 5: len 4; hex 80000014; asc     ;;
 6: len 1; hex 81; asc  ;;
 7: SQL NULL;

Record lock, heap no 10 PHYSICAL RECORD: n_fields 8; compact format; info bits 0
 0: len 6; hex 000000000208; asc       ;;
 1: len 6; hex 00000000075f; asc      _;;
 2: len 7; hex 820000013b0188; asc     ;  ;;
 3: len 4; hex 80000019; asc     ;;
 4: len 4; hex 80000019; asc     ;;
 5: len 4; hex 8000000f; asc     ;;
 6: len 1; hex 80; asc  ;;
 7: SQL NULL;

```

我们从锁状态中看到，innodb 会生成一个`GEN_CLUST_INDEX`索引，然后给这个索引添加 next-key lock

- 给所有记录的`GEN_CLUST_INDEX`索引添加 next-key lock
- 然后依然会锁住所有的记录和间隙
  - 所以最终的锁定区间是
  - (-∞,1]
  - ... n条记录和它们的间隙
  - (20,+∞)

### 查询不存在的主键索引


```SQL
mysql> select * from user where id = 200;
Empty set (0.00 sec)

```

```yml
2 lock struct(s), heap size 1136, 1 row lock(s)
MySQL thread id 10, OS thread handle 123145362374656, query id 895 localhost 127.0.0.1 root
TABLE LOCK table `db_test`.`user` trx id 6173 lock mode IX
RECORD LOCKS space id 11 page no 4 n bits 80 index PRIMARY of table `db_test`.`user` trx id 6173 lock_mode X
Record lock, heap no 1 PHYSICAL RECORD: n_fields 1; compact format; info bits 0
 0: len 8; hex 73757072656d756d; asc supremum;;

```

我们发现一共添加了2个锁结构

- 表上添加的独占意向锁
- 给最大值记录添加了邻间锁
  - 锁定了表中最大值210和数据页中的最大索引记录(210,+∞)这个区间


### 使用多个索引查询

我这里只测试3种情况

#### OR操作

```SQL
begin;
select * from user where id = 1 or age = 15 for update;

```

这里就是取的两种情况的并集。

#### 主键索引和二级AND操作

```SQL
begin;
select * from user where id = 15 or age = 15 for update;

```

这个sql应该是被优化了，优化为只根据主键查询，

所以最终是只对id = 15 记录的主键索引添加了普通行锁

#### 二级索引和无索引字段AND操作

```SQL
begin;
select * from user where age = 15 or sex = 1 for update;

```

这里因为sex字段并没有索引，所以在计算锁的时候，会忽略掉sex字段

- 所以这个sql的加锁情况和`select * from user where age = 15`一致。
- 虽然这个sql最终的记录只有一条，但是同样都会锁住。



#### 结论
写到一半的时候，我突然想到了一个问题，然后就觉得没有必要再写一下去了。
那就是

- <span style="color:red">再一次查询中，通常MySQL只会使用一个索引</span>
- 所以我们只要判断出这个SQL中会使用哪个索引就好了，加锁的情况只会和使用的索引有关，和查询中其他没有使用索引没有任何关系
- 所以第二个操作和第三个操作就都能解释了。
- 关于第一个OR操作，我explain了一下

```SQL
mysql> explain select * from user where id = 1 or age = 15 ;
+----+-------------+-------+------------+-------------+-----------------+-----------------+---------+------+------+----------+-------------------------------------------+
| id | select_type | table | partitions | type        | possible_keys   | key             | key_len | ref  | rows | filtered | Extra                                     |
+----+-------------+-------+------------+-------------+-----------------+-----------------+---------+------+------+----------+-------------------------------------------+
|  1 | SIMPLE      | user  | NULL       | index_merge | PRIMARY,idx_age | PRIMARY,idx_age | 4,5     | NULL |    3 |   100.00 | Using union(PRIMARY,idx_age); Using where |
+----+-------------+-------+------------+-------------+-----------------+-----------------+---------+------+------+----------+-------------------------------------------+
1 row in set, 1 warning (0.00 sec)

```
发现他使用的是`index_merge`类型，也就是说两个索引都会用到。
所以加锁的情况也就是 2种索引加锁的情况的并集了。


## 加锁情况分析总结

分析了上面几条SQL的加锁情况之后，我感觉有一种通用的规则。

那就是在可重复读级别下，所使用的的加锁方式，都是围绕这个可重复读来添加的，也就是同一个事务中第一次读和第二次读的结果必须是一致的。

我们还是举例说明一下

```SQL
mysql> select * from user where id <= 10;
+----+--------+------+------+------+
| id | number | age  | sex  | name |
+----+--------+------+------+------+
|  3 |      3 |    4 |    1 | test |
|  4 |      4 |   34 |    1 | name |
|  5 |      5 |   51 |    1 | name |
|  7 |      7 |    4 |    1 | test |
| 10 |     10 |   10 |    1 | name |
+----+--------+------+------+------+
5 rows in set (0.00 sec)

```

为了保证这个查询结果不会发生变化，我们需要在什么范围内加锁呢？

 所有id小于等于10的记录和间隙是不是都需要锁住？

- 首先肯定现在表上添加独占意向锁
- 然后这些查询出来的记录都要加普通记录锁,还有记录之间的间隙也要加间隙锁
  - 所以就是直接加next-key邻间锁
- 所以现在锁住的区间就有(-∞,3](3,4](4,5](5,7](7,10]


### update、delete 和 select for update 的区别

实际测试了一下，这几种当前读的加锁方式，有稍微的不同。

delete 和 update 完全相同

delete 和 update 在范围查询情况下，往往比 select for update 范围要大一点，就是在临界行的判断上。

参考一下：

[ 常见 SQL 语句的加锁分析](https://www.aneasystone.com/archives/2017/12/solving-dead-locks-three.html)


### MySQL 范围加锁的一个Bug

在 MySQL 8.0.18之前都有一个Bug。也就是针对上面的那个SQL。

MySQL同样会给 10后面的一条记录添加邻间锁

```SQL

mysql> select version();
+------------+
| version()  |
+------------+
| 5.7.34-log |
+------------+
1 row in set (0.00 sec)

mysql> select * from user where id <=10 for update;
+----+--------+------+------+------+
| id | number | age  | sex  | name |
+----+--------+------+------+------+
|  3 |      3 |    4 |    1 | NULL |
|  4 |      4 |   34 |    1 | NULL |
|  5 |      5 |   51 |    1 | NULL |
|  7 |      7 |    4 |    1 | NULL |
| 10 |     10 |   10 |    1 | NULL |
+----+--------+------+------+------+
5 rows in set (0.00 sec)

mysql> show engine innodb status \G;

```

```
TRANSACTIONS
------------
Trx id counter 2323
Purge done for trx's n:o < 2320 undo n:o < 0 state: running but idle
History list length 0
LIST OF TRANSACTIONS FOR EACH SESSION:
---TRANSACTION 2322, ACTIVE 10 sec
2 lock struct(s), heap size 1136, 6 row lock(s)
MySQL thread id 19, OS thread handle 22553189902080, query id 50 localhost 127.0.0.1 root starting
show engine innodb status
TABLE LOCK table `db_test`.`user` trx id 2322 lock mode IX
RECORD LOCKS space id 25 page no 3 n bits 80 index PRIMARY of table `db_test`.`user` trx id 2322 lock_mode X
Record lock, heap no 2 PHYSICAL RECORD: n_fields 7; compact format; info bits 0
 0: len 4; hex 80000003; asc     ;;
 1: len 6; hex 000000000907; asc       ;;
 2: len 7; hex a70000011b0110; asc        ;;
 3: len 4; hex 80000003; asc     ;;
 4: len 4; hex 80000004; asc     ;;
 5: len 1; hex 81; asc  ;;
 6: SQL NULL;

Record lock, heap no 3 PHYSICAL RECORD: n_fields 7; compact format; info bits 0
 0: len 4; hex 80000004; asc     ;;
 1: len 6; hex 000000000907; asc       ;;
 2: len 7; hex a70000011b011c; asc        ;;
 3: len 4; hex 80000004; asc     ;;
 4: len 4; hex 80000022; asc    ";;
 5: len 1; hex 81; asc  ;;
 6: SQL NULL;

Record lock, heap no 4 PHYSICAL RECORD: n_fields 7; compact format; info bits 0
 0: len 4; hex 80000005; asc     ;;
 1: len 6; hex 000000000907; asc       ;;
 2: len 7; hex a70000011b0128; asc       (;;
 3: len 4; hex 80000005; asc     ;;
 4: len 4; hex 80000033; asc    3;;
 5: len 1; hex 81; asc  ;;
 6: SQL NULL;

Record lock, heap no 5 PHYSICAL RECORD: n_fields 7; compact format; info bits 0
 0: len 4; hex 80000007; asc     ;;
 1: len 6; hex 000000000907; asc       ;;
 2: len 7; hex a70000011b0134; asc       4;;
 3: len 4; hex 80000007; asc     ;;
 4: len 4; hex 80000004; asc     ;;
 5: len 1; hex 81; asc  ;;
 6: SQL NULL;

Record lock, heap no 6 PHYSICAL RECORD: n_fields 7; compact format; info bits 0
 0: len 4; hex 8000000a; asc     ;;
 1: len 6; hex 000000000907; asc       ;;
 2: len 7; hex a70000011b0140; asc       @;;
 3: len 4; hex 8000000a; asc     ;;
 4: len 4; hex 8000000a; asc     ;;
 5: len 1; hex 81; asc  ;;
 6: SQL NULL;

Record lock, heap no 7 PHYSICAL RECORD: n_fields 7; compact format; info bits 0
 0: len 4; hex 8000000f; asc     ;;
 1: len 6; hex 000000000907; asc       ;;
 2: len 7; hex a70000011b014c; asc       L;;
 3: len 4; hex 8000000f; asc     ;;
 4: len 4; hex 8000000f; asc     ;;
 5: len 1; hex 81; asc  ;;
 6: SQL NULL;

```

我们看到最下一条 id=15的记录也添加了 邻间锁。

在8.0.18以前的版本中，对加锁规则：在RR可重复读级别下，唯一索引上的范围查询，需要访问到不满足条件的第一个值为止

MySQL在 8.0.18版本修复了这个Bug

(MySQL加锁Bug修复)[https://mp.weixin.qq.com/s/xDKKuIvVgFNiKp5kt2NIgA]


# MVCC

首先MVCC只会在RC 和RR隔离级别下生效。

MVCC是 多版本并发控制(Multi Version Cucurrent Control)的简称.MVCC在MySQL InnoDB中的实现主要是为了提高数据库并发性能，用更好的方式去处理读-写冲突，做到即使有读写冲突时，也能做到不加锁，非阻塞并发读

这里的多版本指的就是，不同事务操作数据库所产生的不同的历史快照。MVCC可以根据查询语句，来判断是需要加锁读取当前最新数据，还是读取历史版本数据，如果是要读取历史版本数据，则会根据可见性规则判断读取哪一个版本的数据。

所以这里就会涉及到MVCC的两种读取方式

- 当前读
- 快照读

## 当前读

当前读的意思就是要读取这个记录行中最新的数据。因为当前最新的数据可能有多个事务在操作，所以要加锁。触发当前读的操作有

- select * from user in share mode; 共享锁
- select * from user for update; 排它锁
- update 操作
- delete 操作
- insert 操作


## 快照读

我们普通的select 操作，就是快照读。它会根据可见性算法选取可见的版本进行读取。它在很多情况下，避免了加锁操作，降低了开销，提高了数据库的读取并发性能。

## MVCC的实现原理

简单来说MVCC的实现主要依靠

- 隐藏字段
- undo log
- Read View

下面我们来分别分析一下

### 隐藏字段

简单来说就是在数据库的每行纪录中都保存着3个隐藏字段(当然可能还有其他的，这里我们不讨论)

- A 6-byte DB_TRX_ID
  - 最近修改(修改/插入)事务ID：记录创建这条记录/最后一次修改该记录的事务ID
  - 其中有一个bit 为用来表示记录是否被删除了，所以删除操作对于InnoDB来就就算是更新操作
  - InnoDB会有专门的purge线程来

> In the InnoDB multi-versioning scheme, a row is not physically removed from the database immediately when you delete it with an SQL statement. InnoDB only physically removes the corresponding row and its index records when it discards the update undo log record written for the deletion. This removal operation is called a purge, and it is quite fast, usually taking the same order of time as the SQL statement that did the deletion.

> 当SQL执行删除操作时，InnoDB并不会立即物理删除这条记录，只有当这条删除操作的undo log被抛弃的时候，才会真物理删除这条记录。

- A 7-byte DB_ROLL_PTR
  - 回滚指针，指定当事务将要回滚的时候，要回滚到哪个版本，指向这条记录的上一个版本
- A 6-byte DB_ROW_ID
  - 隐含的自增ID（隐藏主键），如果数据表没有主键，InnoDB会自动以DB_ROW_ID产生一个聚簇索引，如果有主键则不会生成


### undo log

我们上面说过，undo log中，记录了和操作相反的记录，用来回滚。

- Insert undo log ：插入一条记录时，至少要把这条记录的主键值记下来，之后回滚的时候只需要把这个主键值对应的记录删掉就好了。
- Update undo log：修改一条记录时，至少要把修改这条记录前的旧值都记录下来，这样之后回滚时再把这条记录更新为旧值就好了。
- Delete undo log：删除一条记录时，至少要把这条记录中的内容都记下来，这样之后回滚时再把由这些内容组成的记录插入到表中就好了。
  - 我们上面说过，当我们执行删除的时候，数据库并没有执行物理删除，只是记录了一下删除状态位
  -

从另一个角度来看，undo log也可以看成每条记录的不同版本，我们拿update来举例

比如现在我们有一个person表，有name 和age 两个字段

- 比如一个有个事务插入person表插入了一条新记录，记录如下，name为Jerry, age为24岁，隐式主键是1，事务ID和回滚指针，我们假设为NULL
![](/img/undo.png)
- 现在来了一个事务1对该记录的name做出了修改，改为Tom
![](/img/undo1.png)
- 又来了个事务2修改person表的同一个记录，将age修改为30岁
![](/img/undo2.png)

从上面，我们就可以看出，不同事务或者相同事务的对同一记录的修改，会导致该记录的undo log成为一条记录版本线性表，既链表，undo log的链首就是最新的旧记录，链尾就是最早的旧记录。

那么undo log会一直记录下去吗？当然是不会的。后面我们了解完 read view后在来说一下这个问题。


[15.6.6 Undo Logs](https://dev.mysql.com/doc/refman/8.0/en/innodb-undo-logs.html)

### read view

什么是Read View，说白了Read View就是事务进行快照读操作的时候生产的读视图(Read View)，在该事务执行的快照读的那一刻，会生成数据库系统当前的一个快照，记录并维护系统当前活跃事务的ID(当每个事务开启时，都会被分配一个ID, 这个ID是递增的，所以最新的事务，ID值越大)

#### 可见性算法
 read view的核心其实就是一个可见性算法。计算可见性的关键数据有

 - DB_TRX_ID 记录的最后修改事务ID
 - trx_list，当前正在活跃的事务ID列表
 - up_limit_id，活跃事务列表中，最小的事务ID
 - low_limit_id, 生成readview的时刻，系统尚未分配的下一个事务ID，也就是目前已出现过的事务ID的最大值+1

 下面我们来描述一下可见性算法的规则

 首先一个前置的条件就是当前执行查询的这个事务，肯定是存在于`trx_list`中的。

 - <span style="color:green">首先比较`DB_TRX_ID < up_limit_id`</span>
  - <span style="color:green">如果是,则当前事务可以看到这条记录。`因为这表示修改这条记录的事务不在活跃列表中，并且已经提交了。`</span>
  - 如果不是,则进入下面判断
 - <span style="color:green">接下来判断`DB_TRX_ID >= low_limit_id`</span>
  - <span style="color:red">如果是，则当前事务看不到这条记录的变更。`因为trx_list是在生成read view的这个时刻，活跃的事务列表。判断为true，则表示DB_TRX_ID这个事务，是在read view生成之后修改的这条记录，所以不可见`</span>
  - 如果否，则走下面的逻辑
 - <span style="color:green">接下来判断`DB_TRX_ID`事务，是否在当前活跃的事务列表中。</span>
  - <span style="color:red">如果在，则对当前事务不可见，因为这表示，在生成read view的时候，`DB_TRX_ID`事务还没有提交。</span>
  - <span style="color:green">如果不在，则对当前事务可见，以为这表示在生成 read view之前，`DB_TRX_ID`事务就已经提交了。</span>

  通过上面的逻辑我们可以知道，这个可见性算法的关键，就是看，在生成 read view的那一刻,修改记录的事务有没有提交。

  - 如果提交了，则可见。
  - 如果没提交，则不可见。

#### 生成 read view

根据上面的逻辑，我们可以知道，每次生成read view的关键就是，记录了一份这个时刻的事务信息。

- trx_list，当前正在活跃的事务ID列表
- up_limit_id，活跃事务列表中，最小的事务ID
- low_limit_id, 生成readview的时刻，系统尚未分配的下一个事务ID，也就是目前已出现过的事务ID的最大值+1

然后在查询的时候，系统会使用read view记录的事务信息，进行可见性算法的计算，然后得到合适的记录。

#### 生成read view的策略

在不同的隔离级别下，对于read view的生成策略也是不一样的。

比如对于下面这个操作

```SQL
begin;
select * from user where age = 10;
xxx
select * from user where age = 10;
```

- <span style="color:green">在RC(Read Committed)级别下，每次select 操作都会生成一个read view</span>
- <span style="color:green">在RR(REPEATABLE Read)级别下，只有事务中的第一次select会生成read view</span>

这种区别下，造成的影响就是，在两个select 操作之间，如果有另一个事务T2插入了一条age =10 的记录，并提交了事务。

- 在RC(Read Committed)级别下，会读到这条记录，也就是会发生幻读
  - 原因就是第二次select，会重新生成read view,这时候另一个事务已经提交。判断条件DB_TRX_ID >= low_limit_id，肯定不成立
  - 并且事务T2已经不在活跃事务列表中，所以这条记录会对当前事务可见。
- 在RR(REPEATABLE Read)级别下，则不会读到这条记录
  - 因为这时还沿用第一次select时的read view，这时候不管 T2是在第一次select之前还是之后生成的，这个时刻T2是没有提交的
  - 所以这个事务T2的所有变更对当前事务都是不可见的。

验证一下
- 首先把两个session的隔离级别都设置为RC

```SQL
mysql> set session transaction isolation level read committed;
Query OK, 0 rows affected (0.01 sec)

mysql> select @@session.transaction_isolation;
+---------------------------------+
| @@session.transaction_isolation |
+---------------------------------+
| READ-COMMITTED                  |
+---------------------------------+
1 row in set (0.00 sec)

```

![](/img/mvcc.png)

我们看按照图片中的序号执行，在另一个事务提交前，无法读取到，但是提交之后，就能读取到了

- 接着测试一下RR级别，首先同样先设置隔离级别为RR

```SQL
set session transaction isolation level repeatable read;
mysql> select @@session.transaction_isolation;
+---------------------------------+
| @@session.transaction_isolation |
+---------------------------------+
| REPEATABLE-READ                 |
+---------------------------------+
1 row in set (0.00 sec)

```

![](/img/mvcc2.png)

我们看到，在RR级别下，另一个事务提交后，仍然没有读到插入的数据。


### undo log 什么时候删除

undo log不会一直保存，当undo log能够确保没有事务会用到的时候，会设置delete bit，然后等待purge线程回收。

- insert undo log：事务在插入新记录产生的undo log，当事务提交之后可以直接丢弃，因为这是新插入的数据，肯定没有事务需要查询它；
- update undo log：事务在进行 update 或者 delete 的时候产生的 undo log，在快照读的时候还是需要的，所以不能直接删除，只有当系统没有比这个log更早的read-view了的时候才能删除。ps：所以长事务会产生很多老的视图导致undo log无法删除 大量占用存储空间。


## RR级别下如何解决幻读

通过[生成read-view的策略](/blog/mysql_lock/#生成read-view的策略)这一部分我们了解，在RR级别下，通过生成Read-View的策略解决了`快照读的幻读问题`。

那么当前读会不会有幻读问题呢？

答案是肯定的。继续沿用上面的RR隔离级别下的SQL，我们在左边的事务中执行一个当前读的操作

```sql
mysql> update user set sex = 0 where age = 10;
Query OK, 3 rows affected(0.00 sec)

```

我们发现，虽然上面查询只查出来了2条，但是我们执行更新的时候却更新了3条，这在某种意义上来说也算是一种幻读 `一种针对当前读的幻读`



# 本篇文章使用的表结构


## 数据库版本

因为8.0.18版本之前的MySQL加锁有Bug，所以如果你使用的是5.x版本，和我查看的加锁结构可能会有锁区别

```
mysql> select version();
+-----------+
| version() |
+-----------+
| 8.0.25    |
+-----------+
1 row in set (0.00 sec)

```

## 表结构

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

表数据

```SQL
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

# 参考

[15.7.1 InnoDB Locking](https://dev.mysql.com/doc/refman/8.0/en/innodb-locking.html)

[15.3 InnoDB Multi-Versioning](https://dev.mysql.com/doc/refman/8.0/en/innodb-multi-versioning.html)

[15.6.6 Undo Logs](https://dev.mysql.com/doc/refman/8.0/en/innodb-undo-logs.html)

[自增锁模式](https://www.cnblogs.com/gaogao67/p/11123772.html)

[你真的懂MVCC吗？来手动实践一下](https://juejin.cn/post/6844903969815265288)

[MySQL原子性与持久性的保证](https://blog.csdn.net/anying5823/article/details/104675987)

[解决死锁之路（终结篇）- 再见死锁](https://cloud.tencent.com/developer/article/1494926)

[读 MySQL 源码再看 INSERT 加锁流程](https://www.aneasystone.com/archives/2018/06/insert-locks-via-mysql-source-code.html)

[MySQL锁系列（一）之锁的种类和概念](https://keithlan.github.io/2017/06/05/innodb_locks_1/)

[InnoDB这个将近20年的"bug"修复了-范围查询加锁Bug](https://mp.weixin.qq.com/s/xDKKuIvVgFNiKp5kt2NIgA)

[15.7.3 Locks Set by Different SQL Statements in InnoDB](https://dev.mysql.com/doc/refman/8.0/en/innodb-locks-set.html)
