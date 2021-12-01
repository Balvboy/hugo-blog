---
title: "MySQL问题整理"
date: 2021-12-01T11:07:47+08:00
tags:
    - MySQL
categories: [ MySQL ]
draft: true
---

# InnoDB为什么使用B+树作为索引结构

首先常见的索引结构有

- Hash
- B树(Balance Tree)
- B+树

首先Hash结构，在针对指定索引的查询时，效率最高，为O(1)。
但是MySQL作为一种关系型数据库，经常会有范围查询的需求，这种范围查询对于Hash结构是致命的，如果使用Hash结构进行范围查询，只能通过全表扫描，因为经过Hash之后，即使相邻的数据，也可能分布到相隔很远的数据页中。
所以Hash结构只能淘汰。

B树和B+树有相同点，也有区别

B树
!()[/img/btree.png]

B+树
!()[/img/b+tree.webp]

- 相同点是，他们都是多叉平衡树
- 不同点是
	- B树可以在非叶子节点存储数据
	- B+树，只能在叶子节点存储数据
	- B+数，同一层的节点包括叶子节点之间都是有序的，并且通过链表相连(包括数据页和数据行之间都是相连的)

在查询效率方面，B数因为在非叶子节点也存储数据，所以和数据的位置相关，最好为O(1),最差为O(logN)
B+树因为所有的数据都在叶子节点中存储，所以查询效率和位置无关(忽略覆盖索引)，都是O(logN)

另外还有一点，InnoDB是已数据页为单位来存储数据的，默认情况下数据页大小为16KB。

我们假设MySQL每次只加载一个数据页，因为B+数在非叶子节点中并不保存数据，那么可以肯定，一个数据页中包含的索引数量，B+树是远大于B树的。所以在同样的一次查询中，B树就可能造成更多的IO。

另外B+数的同层节点都是以一个链表的形式相连的，所以从一个数据页跳转到相邻的另一个数据页这个操作是十分高效的。

## 参考

(why use b+tree)[https://medium.com/@mena.meseha/what-is-the-difference-between-mysql-innodb-b-tree-index-and-hash-index-ed8f2ce66d69]

(为什么MySQL使用B+树)[https://draveness.me/whys-the-design-mysql-b-plus-tree/]

(MySQL数据库索引为什么选择使用B+树)[https://developpaper.com/why-does-mysql-database-index-choose-to-use-b-tree/]

(B+树在磁盘存储中的应用)[https://www.cnblogs.com/nullzx/p/8978177.html]

(B+树的几点总结)[https://blog.csdn.net/love_u_u12138/article/details/50285655?spm=1001.2014.3001.5501]


# MySQL Server 和存储引擎之间的执行过程

(MySQL中包含IN子句的语句是怎样执行的)[https://juejin.cn/post/6844904048798203911]


# MySQL limit的问题和优化方案

(MySQL的LIMIT这么差劲的吗)[https://juejin.cn/post/7018170284687491080]


# 如何向MySQL中插入大量数据

拼接大SQL 和 小SQL的对比

(10万条数据批量插入，到底怎么做才快？)[https://juejin.cn/post/7025876113943445518?utm_source=gold_browser_extension]

# MySQL中的一些限制

针对版本 5.7

- 单表列限制
	限制1000列左右
- 单表索引数量限制
	64个索引
- 单个联合索引使用字段数
	16个

(InnoDB表的限制)[https://zhuanlan.zhihu.com/p/79987871]


