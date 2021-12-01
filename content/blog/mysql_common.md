---
title: "MySQL常用操作"
date: 2021-11-29T15:56:26+08:00
tags:
    - mysql
categories: [ MySQL ] 
draft: true
---


# 分配数据库权限

允许任何地址，以该账号，密码连接数据库

```
GRANT ALL PRIVILEGES ON *.* TO '数据库账号'@'%' IDENTIFIED BY '数据库密码' WITH GRANT OPTION;

FLUSH PRIVILEGES;
```

# 查看锁状态

```

show engine innodb status \G;

```

# 查看现在正在执行的事务

```SQL
SELECT * FROM information_schema.INNODB_TRX;

```
 
返回的结果中会带有`trx_mysql_thread_id`字段，我们可以使用

`kill trx_mysql_thread_id`命令傻屌这个事务


```SQL

# 查看正在锁的事务
SELECT * FROM INFORMATION_SCHEMA.INNODB_LOCKS;

# 查看等待锁的事务
SELECT * FROM INFORMATION_SCHEMA.INNODB_LOCK_WAITS;

```




# 参考

[mysql 查询正在执行的事务以及等待锁 常用的sql语句](https://blog.csdn.net/u011375296/article/details/51427985)
