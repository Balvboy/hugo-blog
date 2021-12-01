---
title: "MySQL Insert加锁分析"
date: 2021-11-24T10:53:40+08:00
author: "zhouyang"
tags:
    - mysql
    - insert_lock
categories: [ MySQL ]   

---


# Insert语句加锁分析

这里来单独分析一下 Insert 语句的加锁情况。

前面提到了5种行锁
- 普通行锁 ：lock_mode X/S locks rec but not gap
- 间隙锁   ：lock_mode X/S locks gap before rec
- 邻间锁   ：lock_mode X/S 
- 插入意向锁 : Insert Intention Locks
- 隐式锁

其中和插入相关是后面两种 插入意向锁 和隐式锁

## 插入意向锁

前面我们已经介绍了插入意向锁的概念。

当一个事务想要向一个区间插入一条数据的时候，但是这个区间被另一个事务锁定了，这时候就需要插入一个插入意向锁。

这时候如果有多个事务，想要向这个区间插入记录，那就会插入多个插入意向锁。

### 插入意向锁分析

事务A执行

```SQL
mysql> begin;
Query OK, 0 rows affected (0.00 sec)
mysql> select * from user where id >=214 for update;
+-----+--------+------+------+------+
| id  | number | age  | sex  | name |
+-----+--------+------+------+------+
| 214 |   NULL | NULL | NULL | NULL |
+-----+--------+------+------+------+
1 row in set (0.00 sec)
```

事务B执行,发现会被阻塞

```SQL
mysql> begin;
Query OK, 0 rows affected (0.00 sec)

mysql> insert into user (id) values (215);

```

此时查看锁状态

```yml

------------
TRANSACTIONS
------------
Trx id counter 6217
Purge done for trx's n:o < 6214 undo n:o < 0 state: running but idle
History list length 0
LIST OF TRANSACTIONS FOR EACH SESSION:
---TRANSACTION 421847766411040, not started
0 lock struct(s), heap size 1136, 0 row lock(s)
---TRANSACTION 421847766408520, not started
0 lock struct(s), heap size 1136, 0 row lock(s)
---TRANSACTION 421847766407680, not started
0 lock struct(s), heap size 1136, 0 row lock(s)
---TRANSACTION 421847766406840, not started
0 lock struct(s), heap size 1136, 0 row lock(s)
---TRANSACTION 6216, ACTIVE 5 sec inserting
# 事务B产生的锁
mysql tables in use 1, locked 1
LOCK WAIT 2 lock struct(s), heap size 1136, 1 row lock(s)
MySQL thread id 20, OS thread handle 123145362980864, query id 3611 localhost 127.0.0.1 root update
insert into user (id) values (215)
# 事务B 已经等待下面的锁 5秒钟
------- TRX HAS BEEN WAITING 5 SEC FOR THIS LOCK TO BE GRANTED:
# 事务B 添加的 插入意向锁
RECORD LOCKS space id 11 page no 4 n bits 88 index PRIMARY of table `db_test`.`user` trx id 6216 lock_mode X insert intention waiting
Record lock, heap no 1 PHYSICAL RECORD: n_fields 1; compact format; info bits 0
 0: len 8; hex 73757072656d756d; asc supremum;;

------------------
# 事务B 添加的 独占意向锁
TABLE LOCK table `db_test`.`user` trx id 6216 lock mode IX
RECORD LOCKS space id 11 page no 4 n bits 88 index PRIMARY of table `db_test`.`user` trx id 6216 lock_mode X insert intention waiting
Record lock, heap no 1 PHYSICAL RECORD: n_fields 1; compact format; info bits 0
 0: len 8; hex 73757072656d756d; asc supremum;;

---TRANSACTION 6214, ACTIVE 272 sec
# 事务A 产生了3个锁结构 ，其中有2个行锁
3 lock struct(s), heap size 1136, 2 row lock(s)
MySQL thread id 13, OS thread handle 123145362374656, query id 3612 localhost 127.0.0.1 root starting
show engine innodb status
Trx read view will not see trx with id >= 6214, sees < 6214
# 事务A 产生的独占意向锁 表锁
TABLE LOCK table `db_test`.`user` trx id 6214 lock mode IX
# 事务A 产生的 普通行锁，锁住 id=214的主键索引
RECORD LOCKS space id 11 page no 4 n bits 88 index PRIMARY of table `db_test`.`user` trx id 6214 lock_mode X locks rec but not gap
Record lock, heap no 4 PHYSICAL RECORD: n_fields 7; compact format; info bits 0
 0: len 4; hex 800000d6; asc     ;;
 1: len 6; hex 000000001841; asc      A;;
 2: len 7; hex 820000008e0110; asc        ;;
 3: SQL NULL;
 4: SQL NULL;
 5: SQL NULL;
 6: SQL NULL;

# 事务A 产生的next-key lock，锁住 从214 到最大值的区间
RECORD LOCKS space id 11 page no 4 n bits 88 index PRIMARY of table `db_test`.`user` trx id 6214 lock_mode X
Record lock, heap no 1 PHYSICAL RECORD: n_fields 1; compact format; info bits 0
 0: len 8; hex 73757072656d756d; asc supremum;;
```



## 隐式锁

在没有锁冲突的情况下，Insert操作是不加锁的，除了会加一个表级别的 独占意向锁。

那么就会发生这种情况

一个事务向表中插入了条记录，然后另一个事务

- 立即使用SELECT ... LOCK IN SHARE MODE语句读取这条事务，也就是在要获取这条记录的S锁，或者使用SELECT ... FOR UPDATE语句读取这条事务或者直接修改这条记录，也就是要获取这条。
记录的X锁，？ 
- 立即修改这条记录，也就是要获取这条记录的X锁？

这中情况下，就会利用到隐式锁。之所以叫做隐式锁，那就是因为，在没有事务冲突的情况下，隐式锁可以认为是没有锁，当有事务冲突之后，隐式锁就会转化为显示锁。下面我们看一下这个转化的过程。

### 隐式锁的转化

隐式锁的转化需要依赖 trx_id这个隐藏字段

#### 聚簇索引

对于聚簇索引记录来说，有一个trx_id隐藏列，该隐藏列记录着最后改动该记录的事务id。那么如果在当前事务A中新插入一条聚簇索引记录后，该记录的trx_id隐藏列代表的的就是当前事务的事务id。

这是如果另一个事务B，想要对这条记录加锁，就会先看一下这条记录的trx_id,是否还在活跃列表中，如果还在活跃列表，那就表示Insert这条记录的事务还没有提交。

<span style="color: red;">那这时，想要加锁的事务B就会替事务A创建一个排他锁(isWaiting为false表示持有了锁)，然后给自己也创建一个锁结构(isWaiting为true表示等待锁)，然后进入等待状态</span>


#### 二级索引

对于二级索引记录来说，本身并没有trx_id隐藏列，
但是在二级索引页面的Page Header部分有一个PAGE_MAX_TRX_ID属性，该属性代表对该页面做改动的最大的事务id，如 果PAGE_MAX_TRX_ID属性值小于当前最小的活跃事务id，那么
说明对该页面做修改的事务都已经提交了，否则就需要在页面中定位到对应的二级索引记录，然后回表找到它对应的聚簇索引记 录，然后再重复情景一的做法。



### 隐式锁转化分析

事务A 首先我们在一个事务中 插入一条记录,然后查看锁状态

```SQL
mysql> begin;
Query OK, 0 rows affected (0.01 sec)

mysql> insert into user (id) values (225);
Query OK, 1 row affected (0.01 sec)

mysql> show engine innodb status \G

```

只有一个表级别的独占意向锁，没有任何的行锁

```
---TRANSACTION 6217, ACTIVE 50 sec
1 lock struct(s), heap size 1136, 0 row lock(s), undo log entries 1
MySQL thread id 13, OS thread handle 123145362374656, query id 3618 localhost 127.0.0.1 root starting
show engine innodb status
TABLE LOCK table `db_test`.`user` trx id 6217 lock mode IX

```

事务B ，尝试锁定这条记录所在的区间,

```SQL

mysql> begin;
Query OK, 0 rows affected (0.01 sec)

mysql> select * from user where id = 225 for update;

```

发现会被阻塞，这是我们在事务A的窗口查看 锁状态

```
# 事务B 一个锁结构
---TRANSACTION 6218, ACTIVE 88 sec
1 lock struct(s), heap size 1136, 1 row lock(s)
MySQL thread id 20, OS thread handle 123145362980864, query id 3620 localhost 127.0.0.1 root
TABLE LOCK table `db_test`.`user` trx id 6218 lock mode IX
---TRANSACTION 6217, ACTIVE 246 sec
# 事务 A 现在持有了2个锁
2 lock struct(s), heap size 1136, 1 row lock(s), undo log entries 1
MySQL thread id 13, OS thread handle 123145362374656, query id 3621 localhost 127.0.0.1 root starting
show engine innodb status
TABLE LOCK table `db_test`.`user` trx id 6217 lock mode IX
RECORD LOCKS space id 11 page no 4 n bits 88 index PRIMARY of table `db_test`.`user` trx id 6217 lock_mode X locks rec but not gap
Record lock, heap no 3 PHYSICAL RECORD: n_fields 7; compact format; info bits 0
 0: len 4; hex 800000e1; asc     ;;
 1: len 6; hex 000000001849; asc      I;;
 2: len 7; hex 82000000910110; asc        ;;
 3: SQL NULL;
 4: SQL NULL;
 5: SQL NULL;
 6: SQL NULL;


```

发现 6217事务，也就是 事务A，多持有了一个普通行锁，这个就是事务B替 事务A创建的锁。

不过有一个问题是，并没有体现出来 事务B给自己创建的锁,我们上面的例子使用的数据库版本是 8.1.25

下面我们在再使用5.x 版本测试一下


```

---TRANSACTION 2331, ACTIVE 17 sec starting index read
mysql tables in use 1, locked 1
LOCK WAIT 2 lock struct(s), heap size 1136, 1 row lock(s)
MySQL thread id 65, OS thread handle 22553189902080, query id 249 61.149.179.190 root statistics
select * from user where id  = 211 for update
------- TRX HAS BEEN WAITING 17 SEC FOR THIS LOCK TO BE GRANTED:
RECORD LOCKS space id 25 page no 3 n bits 80 index PRIMARY of table `db_test`.`user` trx id 2331 lock_mode X locks rec but not gap waiting
Record lock, heap no 13 PHYSICAL RECORD: n_fields 7; compact format; info bits 0
 0: len 4; hex 800000d3; asc     ;;
 1: len 6; hex 00000000091a; asc       ;;
 2: len 7; hex b60000012a0110; asc     *  ;;
 3: SQL NULL;
 4: SQL NULL;
 5: SQL NULL;
 6: SQL NULL;

------------------
TABLE LOCK table `db_test`.`user` trx id 2331 lock mode IX
# 事务B自己创建的锁结构，isWaiting 为true
RECORD LOCKS space id 25 page no 3 n bits 80 index PRIMARY of table `db_test`.`user` trx id 2331 lock_mode X locks rec but not gap waiting
Record lock, heap no 13 PHYSICAL RECORD: n_fields 7; compact format; info bits 0
 0: len 4; hex 800000d3; asc     ;;
 1: len 6; hex 00000000091a; asc       ;;
 2: len 7; hex b60000012a0110; asc     *  ;;
 3: SQL NULL;
 4: SQL NULL;
 5: SQL NULL;
 6: SQL NULL;

# 事务B替事务A创建的锁
---TRANSACTION 2330, ACTIVE 45 sec
2 lock struct(s), heap size 1136, 1 row lock(s), undo log entries 1
MySQL thread id 64, OS thread handle 22553181468416, query id 248 61.149.179.190 root
Trx read view will not see trx with id >= 2330, sees < 2330
TABLE LOCK table `db_test`.`user` trx id 2330 lock mode IX
RECORD LOCKS space id 25 page no 3 n bits 80 index PRIMARY of table `db_test`.`user` trx id 2330 lock_mode X locks rec but not gap
Record lock, heap no 13 PHYSICAL RECORD: n_fields 7; compact format; info bits 0
 0: len 4; hex 800000d3; asc     ;;
 1: len 6; hex 00000000091a; asc       ;;
 2: len 7; hex b60000012a0110; asc     *  ;;
 3: SQL NULL;
 4: SQL NULL;
 5: SQL NULL;
 6: SQL NULL;


```

我们看到 5.x版本的MySQL和上面的描述比较一致

- 替事务A创建一个锁
- 然后在给自己创建一个锁

所以看来，MySQL的各个版本也是在对锁这部分在进行优化，但是基本的逻辑应该是没有变化的。


# Insert on duplicate key update加锁分析


```sql
begin;
insert into user (id) values (214) ON DUPLICATE KEY UPDATE number = 214;
```

```
2 lock struct(s), heap size 1136, 1 row lock(s)
MySQL thread id 25, OS thread handle 123145362980864, query id 3710 localhost root starting
show engine innodb status
TABLE LOCK table `db_test`.`user` trx id 6224 lock mode IX
RECORD LOCKS space id 11 page no 4 n bits 88 index PRIMARY of table `db_test`.`user` trx id 6224 lock_mode X locks rec but not gap
Record lock, heap no 19 PHYSICAL RECORD: n_fields 7; compact format; info bits 0
 0: len 4; hex 800000d6; asc     ;;
 1: len 6; hex 00000000184e; asc      N;;
 2: len 7; hex 010000014c0af5; asc     L  ;;
 3: len 4; hex 800000d6; asc     ;;
 4: SQL NULL;
 5: SQL NULL;
 6: SQL NULL;

```

当发生唯一键冲突的时候，Insert会加锁，加的锁和 执行Update一致

# 参考

[读 MySQL 源码再看 INSERT 加锁流程](https://www.aneasystone.com/archives/2018/06/insert-locks-via-mysql-source-code.html)

[如何读懂 MySQL rw-lock 锁的统计信息](https://blog.csdn.net/shaochenshuo/article/details/109531445)




