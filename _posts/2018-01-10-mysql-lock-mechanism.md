---
layout: post
title:  "Mysql 5.7 InnoDB 锁机制"
date:   2018-01-10 12:29:18 +0800 
categories: 数据库
tags: mysql lock
author: Tommy.Tesla
mathjax: true
---

这个部分我们介绍一下InnoDB所使用的**锁**。

#### 共享(shared lock)和排他锁(exclusive lock)

InnoDB 实现了标准的行级锁，主要分为两类：共享锁和排他锁。

 - 共享锁（s）允许事务获取锁来读取某行记录。
 - 排他锁（x）允许事务获得锁来更新或者删除某行记录。

如果事务T1获得某记录(*r*)的一个共享锁（*s*），那么就r记录来说，来自其他事务（T2）的请求会按照下面两种情况被处理：

 - T2发出了对r记录的s锁请求：立即获得s锁，这样T1, T2就都获得了s锁。
 - T2发出了对r记录的x锁请求：无法获取，需要等待。

如果T1事务获取了对r记录的x排他锁，那么来自其他事务（T2）的请求，无论是s锁请求还是x锁请求，都无法立即获取，而是等待T1释放了对r记录的x锁后才能获取。

#### 意向锁
Innodb 支持*多粒度加锁*，这种锁允许事务同时对记录和表加锁。为了实现*多粒度加锁*，我们创建了一种新类型的锁即**意向锁**。在Innodb中，意向锁本身属于表级锁，它将决定事务在请求对某行记录加锁时使用共享锁还是排他锁。相对应的，意向锁也分为两种：

 - 意向共享锁(*IS*)：事务T尝试获得某些独立的行的s共享所。
 - 意向排他锁(*IX*)：事务T尝试获得这些行的x排他锁。

比如，**select ... lock in share mode** 将会尝试获得IS锁，**select ... for update** 则尝试获得IX锁。

意向锁的分配协议如下：

 - 在一个事务获得对表T的*s*锁之前，它必须先获得对表的意向锁*IS*，或者更强的锁。
 - 在一个事务获得对表T的*x*锁之前，它必须获得对该表的*IX*锁。

简单的总结下，这两个规则可以被描述为以下的*锁类型匹配矩阵*:
<center>
|  type  | X | IX | s | IS | 
|  --- | --- | --- | --- | --- | 
| x | 互斥 | 互斥| 互斥| 互斥 | 
| IX | 互斥| 共存 | 互斥| 共存 | 
| S | 互斥| 互斥| 共存| 共存 | 
| IS | 互斥| 共存| 共存| 共存 | 
</center>
两锁互斥代表事务在已经获取了其中某个锁后无法再获取另外一个锁，共存同理。在互斥的情况下，一个事务如果已经获得了某个锁，当它尝试获得另外一个锁时会进入等待队列。当一个锁请求与当前获得的锁冲突并且无法获得锁（为了防止死锁）时，会报错。

所以，意向锁不会锁任何东西，除非事务直接尝试对整个表加锁（比如：lock tables ... write)。*IX*和*IS* 锁主要是用来表达某个事务正在使用某一记录，或者将要锁住表中的某行。

意向锁的事务结构在**[ SHOW ENGINE INNODB STATUS ](https://dev.mysql.com/doc/refman/5.7/en/show-engine.html)**和[InnoDB Monitor](https://dev.mysql.com/doc/refman/5.7/en/innodb-standard-monitor.html) 中表现为类似于下面这样：
```
TABLE LOCK table `test`.`t` trx id 10080 lock mode IX
```

#### 记录锁
记录锁是指加在索引记录上的锁。比如，**select c1 from t where c1 = 10 for update；** 这条语句可以防止其他事务插入、更新、或者删除t.c1字段为10的行。

记录永远只锁住索引记录，哪怕一个表没有定义任何索引。在这种情况下，InnoDB会创建一个隐藏的簇索引并且用它来进行记录加锁。具体方法参考[簇索引和第二索引]();

记录锁的事务结构在**[ SHOW ENGINE INNODB STATUS ](https://dev.mysql.com/doc/refman/5.7/en/show-engine.html)**和[InnoDB Monitor](https://dev.mysql.com/doc/refman/5.7/en/innodb-standard-monitor.html) 中表现为类似于下面这样：
```
RECORD LOCKS space id 58 page no 3 n bits 72 index `PRIMARY` of table `test`.`t` 
trx id 10078 lock_mode X locks rec but not gap
Record lock, heap no 2 PHYSICAL RECORD: n_fields 3; compact format; info bits 0
 0: len 4; hex 8000000a; asc     ;;
 1: len 6; hex 00000000274f; asc     'O;;
 2: len 7; hex b60000019d0110; asc        ;;
```

#### 间隙锁（Gap Lock）
间隙锁是对指定记录之间间隙所加的锁，这个间隙包括：记录与记录之间的间隙，第一条记录之前的间隙以及最后一条记录后面的间隙。举个例子：**select c1 from t where c1 between 10 and 20 for update** 将防止其他事务插入15这个数字到t.c1字段中，无论是否已经有一个这样的值在这个字段中了，因为值在这个范围内的所有记录都被锁住了。

一个间隙可以是一个单个的索引值，可以是多个索引值，甚至可以是空的。

间隙锁是性能和并发平衡的结果，只在某些事务隔离级别中才会被使用。

当使用某个唯一索引查询某个记录的时候，锁住这条索引对应的行是用不到间隙锁的。（不包括组合索引的情况，组合索引会导致间隙锁）。举个例子，如果**id** 字段上建立了一个唯一索引，以下SQL语句将会并且只会使用一个索引记录锁，并且不会影响其他事务在其前面的间隙进行数据插入。
```
SELECT * FROM child WHERE id = 100;
```
而如果**id** 没有被索引，或者是一个非唯一索引，这条语句将会锁住索引记录前面的空隙。

同样值得注意的是，间隙上的不同冲突类型的锁可以同时被多个不同的事务获取。比如，事务A获取了某个间隙上的一个共享间隙锁（间隙 S-锁），同时事务B可以获取这个间隙上的互斥锁（间隙 X-锁）。**多冲突类型间隙锁**之所以能存在是因为：如果某个记录从索引中删除时，这条记录上的间隙锁（多个事务持有的）一定会被合并。

间隙锁在InnoDB中是**专一功能**(purely inhibitive)，这意味着它们只能防止其他事务在这个间隙中插入数据，而无法阻止不同的事务在同样的间隙上获取间隙锁。所以就间隙锁来说，S锁和X锁效果一样。

间隙锁可以被显式的禁用。当你把事务的隔离级别降为READ_COMMITTED或者打开了*innodb_locks_unsafe_for_binlog*系统变量（目前已经过时）的时候，间隙锁就会被禁用。在这些情况下，间隙锁只会在外键约束检查和重复主键校验时用到，像搜索和索引扫描时这些动作时就不会再用到了。

使用READ_COMMITTED隔离级别或者打开*innodb_locks_unsafe_for_binlog*系统变量会引发其他一些问题。在MySQL评估了WHERE条件后，那些不匹配的行所对应的记录锁就会被立即释放。对于更新操作来说，InnoDB会读取最新版本提交的记录，来判断最新的值是否和WHERE语句后面的条件匹配。
> 翻译注解：这里指在事务处理过程中会实时读取其他事务提交的数据改动，这就是为何READ_COMMITTED隔离级别会导致**不可重复读**的问题。


#### NK锁(Next-Key)
NK锁是*记录锁*和目标记录*前部间隙锁*的结合体。当InnoDB数据库搜索扫描表索引时，会对匹配了索引条件的索引记录进行加锁（s锁或者x锁），这就是行级锁的实现方式。所以行级锁实际上就是指索引记录锁。NK锁不仅会锁住索引记录，同时也会影响这条索引记录之前的间隙。这就是说，一个NK锁=索引记录锁 + 记录的前间隙锁。如果一个事务会话已经拥有了对索引中记录R的共享或者排他锁，另外一个事务将无法在此索引记录之前插入一个新的索引记录。

假设一个索引序列包含了10,11,13,20。在这个索引中，可能的NK锁包含以下锁：
```
(负无穷, 10]
(10, 11]
(11, 13]
(13, 20]
(20, positive 正无穷)
```
其中，圆括号代表不包括当前边界，方括号则代表包含当前边界值。

在最后一个间隔中，NK锁将会锁住从20到最大值之间的区间，这个最大值伪记录比当前索引里的任何一个值都大。这个伪记录不是一个真实的索引记录。所以，事实上，NK锁只是锁住了最大索引记录的尾部。

默认来说，InnoDB的事务隔离级别为**REPEATABLE READ**。在这种情况下，InnoDB使用NK锁来搜索和扫描索引，可以防止幻读（phantom read）。

NK锁的事务结构在**[ SHOW ENGINE INNODB STATUS ](https://dev.mysql.com/doc/refman/5.7/en/show-engine.html)**和[InnoDB Monitor](https://dev.mysql.com/doc/refman/5.7/en/innodb-standard-monitor.html) 中表现为类似于下面这样：

```
RECORD LOCKS space id 58 page no 3 n bits 72 index `PRIMARY` of table `test`.`t` 
trx id 10080 lock_mode X
Record lock, heap no 1 PHYSICAL RECORD: n_fields 1; compact format; info bits 0
 0: len 8; hex 73757072656d756d; asc supremum;;

Record lock, heap no 2 PHYSICAL RECORD: n_fields 3; compact format; info bits 0
 0: len 4; hex 8000000a; asc     ;;
 1: len 6; hex 00000000274f; asc     'O;;
 2: len 7; hex b60000019d0110; asc        ;;
```
 
#### 插入意向锁
插入意向锁是指INSERT操作在执行真实的行插入之前，加于记录上的一个间隙锁。这个锁表达了事务的插入意向（简单说就是你想往哪插），假设存在几个不同的事务同时要向这个索引间隙里插入数据，如果他们插的不是同一个地方，那么就可以不用互相等待了，可以直接插入。假设现在有两个索引记录4和7，有两个不一样的事务要插入索引5和6，在获取目标插入行的互斥锁之前，他们两个都使用了**插入意向锁**锁住了4-7这个间隙，但他们不会阻塞，因为5和6不一样，并不冲突。

下面我们举个例子来说明插入记录中插入意向锁的作用过程。这个例子涉及两个客户端：A和B。

客户端A创建了一个表，包含了两个索引记录（90和102），然后启动了一个事务来在目标索引记录上加互斥锁（目标索引记录值大于100）。这个互斥锁包含了102之前的一个间隙锁：
```
mysql> CREATE TABLE child (id int(11) NOT NULL, PRIMARY KEY(id)) ENGINE=InnoDB;
mysql> INSERT INTO child (id) values (90),(102);

mysql> START TRANSACTION;
mysql> SELECT * FROM child WHERE id > 100 FOR UPDATE;

+-----+
| id  |
+-----+
| 102 |
+-----+
```
客户端B也打开了一个事务准备向这个间隙里插入一个记录。当它在等待获取排他锁的过程中同时持有了一个插入意向锁。
```
mysql> START TRANSACTION;
mysql> INSERT INTO child (id) VALUES (101);
```
插入意向锁的事务结构在**[ SHOW ENGINE INNODB STATUS ](https://dev.mysql.com/doc/refman/5.7/en/show-engine.html)**和[InnoDB Monitor](https://dev.mysql.com/doc/refman/5.7/en/innodb-standard-monitor.html) 中表现为类似于下面这样：
```
RECORD LOCKS space id 31 page no 3 n bits 72 index `PRIMARY` of table `test`.`child`
trx id 8731 lock_mode X locks gap before rec insert intention waiting
Record lock, heap no 3 PHYSICAL RECORD: n_fields 3; compact format; info bits 0
 0: len 4; hex 80000066; asc    f;;
 1: len 6; hex 000000002215; asc     " ;;
 2: len 7; hex 9000000172011c; asc     r  ;;...
```

#### 自增锁（AUTO-INC)
自增锁是在事务插入一条含有自增字段的记录时用到的一个特殊的表级锁。在这个最简单的情况下，如果一个事务正在向表中插入数据，任何其他事务必须等待，然后第一个事务插入的数据行才能被赋予连续的主键值。

**innodb_autoinc_lock_mode**这个配置选项用来控制自增锁所使用的算法。它允许我们在可预见自增序列和最大插入并发性中做出平衡。

点击[InnoDB自增机制](https://dev.mysql.com/doc/refman/5.7/en/innodb-auto-increment-handling.html)查看更多信息。

#### 空间索引的预测锁

InnoDB 支持包含空间列的空间索引（[优化空间分析](https://dev.mysql.com/doc/refman/5.7/en/optimizing-spatial-analysis.html)）。

在管理空间索引操作相关的锁时，NK锁对REPEATABLE_READ和SERIALIZABLE两种隔离级别的支持并不好。对多维度的数据集来说，没有绝对的排序顺序，所以我们并不知道哪个键是"下一个"。

为了使用了空间索引的表支持多个隔离级别，InnoDB使用了**预测锁**。一个空间索引是由一系列最小化弹跳矩阵（MBR）组成的，InnoDB通过锁住一个查询所使用的MBR来进行一致性读取。

其他事务都无法插入或者修改查询条件所匹配的行。


* content
{:toc}