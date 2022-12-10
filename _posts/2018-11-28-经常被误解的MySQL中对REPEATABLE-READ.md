---
layout:     post
title:      "详细分析MySQL如何用REPEATABLE-READ解决幻读问题"
subtitle:   "MySQL's REPEATABLE-READ and PhantomRead Misconception"
date:       2018-11-28
author:     S.L
header-img: img/home-bg-o.jpg
catalog: true
tags: 
    - MySQL
---

# MySQL解决幻读

## 啥是幻读
> The so-called phantom problem occurs within a transaction when the same query produces different sets of rows at different times. For example, if a SELECT is executed twice, but returns a row the second time that was not returned the first time, the row is a “phantom” row.

上面这句话摘自MySQL的官方手册。它只说明了读会读到上一次没有返回的记录，看起来是幻影一般。如果你理解到这里，那么恭喜你，你会遇到各种困惑。
其实幻读的现象远不止于此，更不仅仅只是『两次「读」，第二次「读」时发现有幻行』。

## MySQL的隔离级别
MySQL的InnoDb存储引擎默认的隔离级别是REPEATABLE-READ，即可重复读。
那什么是「可重复读」呢，简单来说就是一个事务里的两个相同条件的查询查到的结果应该是一致的，即结果是「可以重复读到的」，所以就解决了「幻读」。

OK，听起来很简单，一个隔离级别就可以搞定了，但是内部的机制和原理并不简单，并且有些概念的作用可能大家并不知道具体解决了什么问题。

## 如何解决

### MVCC
### MVCC的原理
MVCC(Multi-Version Concurrency Control多版本并发控制)：
- MVCC每次更新操作都会复制一条新的记录，新纪录的创建时间为当前事务ID
- 优势为读不加锁，读写不冲突
- InnoDb存储引擎中，每行数据包含了一些隐藏字段 `DB_TRX_ID`，`DB_ROLL_PTR`，`DB_ROW_ID`，`DELETE BIT`，其中：
    - `DB_TRX_ID`字段记录了数据的最后一次更新（`INSERT`/`UPDATE`)的事务ID，这个时间指的是对数据进行操作的事务的id
    - `DB_ROLL_PTR`指向当前数据的`undo log`记录，回滚数据就是通过这个指针
    - `DELETE BIT`位用于标识该记录是否被删除，这里的不是真正的删除数据，而是标志出来的删除。真正意义的删除是在mysql进行数据的GC，清理历史版本数据的时候。
如[MySQL官方手册](https://dev.mysql.com/doc/refman/5.7/en/innodb-multi-versioning.html)

> Internally, InnoDB adds three fields to each row stored in the database. A 6-byte DB_TRX_ID field 
> indicates the transaction identifier for the last transaction that inserted or updated the row. Also, a 
> deletion is treated internally as an update where a special bit in the row is set to mark it as 
> deleted. Each row also contains a 7-byte DB_ROLL_PTR field called the roll pointer. The roll 
> pointer points to an undo log record written to the rollback segment. If the row was updated, 
> the undo log record contains the information necessary to rebuild the content of the row before 
> it was updated. A 6-byte DB_ROW_ID field contains a row ID that increases monotonically as new 
> rows are inserted. If InnoDB generates a clustered index automatically, the index contains row 
> ID values. Otherwise, the DB_ROW_ID column does not appear in any index.
  
以具体的DML举例：
- `INSERT`：创建一条新数据，`DB_TRX_ID`中的创建时间为当前事务ID，`DB_ROLL_PT`为NULL，即没有需要回滚的数据指向
- `DELETE`：将当前行的`DB_TRX_ID`中的删除时间设置为当前事务ID，`DELETE BIT`设置为1
- `UPDATE`：复制了一行，新行的DB_TRX_ID中的创建时间为当前事务ID，删除时间为空，`DB_ROLL_PT`指向了上一个版本的记录，事务提交后`DB_ROLL_PT`置为NULL

可知，为了提高并发度，InnoDb提供了这个「非锁定读」，即不需要等待访问行上的锁释放，读取行的一个快照即可，也可以成为快照读。

> 既然是多版本读，那么肯定读不到隔壁事务的新插入数据了，所以解决了「幻读」。

### READ_VIEW
MVCC实现了多个并发事务更新同一行记录会时产生多个记录版本，那问题来了，新开始的事务如果要查询这行记录，应该获取到哪个版本呢？即哪个版本对这个事务是可见的。这个问题就是行记录的可见性问题。
每个事务在第一个SQL开始的时候都会根据当前系统的活跃事务链表创建一个read_view

### MVCC与隔离级别
- `Read Uncommitted`每次都读取记录的最新版本，会出现脏读，未实现MVCC
- `Serializable`对所有读操作都加锁，读写发生冲突，不会使用MVCC
- SELECT
    - RR级别：InnoDb检查每行数据，确保它们符合两个标准：
        - 只查找创建时间早于当前事务ID的记录，这确保当前事务读取的行都是事务之前已经存在的，或者是由当前事务创建或修改的行
        - 行的`DELETE BIT`为1时，查找删除时间晚于当前事务ID的记录，确定了当前事务开始之前，行没有被删除
    - RC级别：每次重新计算read-view，read-view的范围为InnoDb中最大的事务ID，为避免脏读读取的是DB_ROLL_PT指向的记录

就这么简单吗？
其实幻读有很多种出现形式，简单的`SELECT`不加条件的查询在RR下肯定是读不到隔壁事务提交的数据的。但是仍然可能在执行`INSERT`/`UPDATE`时遇到幻读现象。因为`SELECT`
不加锁的快照读行为是无法限制其他事务对新增重合范围的数据的插入的。

所以还要引入第二个机制。

### Next-Key Lock
其实更多的幻读现象是通过写操作来发现的，如`SELECT`了3条数据，`UPDATE`的时候可能返回了4个成功结果，或者`INSERT`某条不存在的数据时忽然报错「唯一索引冲突」等。

首先来了解一下InnoDb的锁机制，InnoDB有三种行锁：
- Record Lock：单个行记录上的锁
- Gap Lock：间隙锁，锁定一个范围，但不包括记录本身。GAP锁的目的，是为了防止同一事务的两次「当前读」，出现幻读的情况
- Next-Key Lock：前两个锁的加和，锁定一个范围，并且锁定记录本身。对于行的查询，都是采用该方法，主要目的是解决幻读的问题

如果是带排他锁操作（除了`INSERT`/`UPDATE`/`DELETE`这种，还包括`SELECT FOR UPDATE`/`LOCK IN SHARE MODE`等），它们默认都在操作的记录上加了`Next-Key 
Lock`。只有使用了这里的操作后才会在相应的记录周围和记录本身加锁，即`Record Lock` + `Gap Lock`，所以会导致有冲突操作的事务阻塞进而超时失败。

### 性能
隔离级别越高并发度越差，性能越差，虽然MySQL默认的是RR，但是如果业务不需要严格的没有幻读现象，是可以降低为RC的或修改配置`innodb_locks_unsafe_for_binlog`为1
来避免`Gap Lock`的。

> 注意有的时候MySQL会自动对`Next-Key Lock`进行优化，退化为只加`Record Lock`，不加`Gap Lock`，如相关条件字段为主键时直接加`Record Lock`。


# REPEATABLE-READ的误解
## 误解零
> 凡是在REPEATABLE-READ中执行的语句均不会遇到幻读现象。

这个显然是错误的。REPEATABLE-READ只是有机制可以用来防止幻读的发生，但如果你没有「使用」或「激活」它相关机制，你仍然会遇到幻读现象。

## 误解一
> REPEATABLE-READ肯定不会读到隔壁事务已经提交的数据，即使某个数据已经由隔壁事务提交，当前事务插入不会报错，否则就是发生了幻读。

简单来说前半句话是对的，后半句有什么问题呢？可REPEATABLE-READ中如何「读」是我们自己来写SELECT
的，如果不加锁则属于快照读，当前事务读不到的数据并不一定是不存在的，如果已经存在对应的数据，那么当前事务尝试插入的时候是可能会失败的。
而插入失败的原因可能是因为主键冲突导致数据库报异常，跟隔离级别无直接关系。任何隔离级别下插入已经存在的数据都会报错。

一句话，看不到并不代表没有，并不代表可以自以为然的插入无忧。

## 误解二
> REPEATABLE-READ的事务里查不到的数据一定是不存在的，所以我可以放心插入，100%成功。

这个观点也是错的，查不到只能说明当前事务里读不到，并不代表此时其他事务没有插入这样的数据。
如何保证判断某个数据不存在以后其他事务也不会插入成功？答案是上Next-Key Lock。不上锁是无法阻止其他事务插入的。
> SELECT * FROM table1 WHERE id >100

上面这个语句在事务里判断后如果不存在数据是无法保证其他事务插入符合条件的数据的，需要加锁
> SELECT * FROM table1 WHERE id >100 FOR UPDATE;

此时如果有隔壁事务尝试插入大于100的id的数据则会等待当前事务释放锁，直到超时后中断当前事务。
> (waiting for lock ...
   then timeout)
  ERROR 1205 (HY000):
  Lock wait timeout exceeded;
  try restarting transaction
  
但是如果当前事务使用的加锁的条件仅仅是某一个行锁的话最多会在前后加Next-Key Lock，影响范围较小，但仍然可能阻塞其他事务的插入，如恰好新数据的位置被GAP 
Lock锁住了，那只能等待当前事务释放锁了。

说了这么多，有一点要注意，就是这个Next-Key Lock一定是在REPEATABLE-READ下才有，READ-COMMITTED是不存在的。

> To prevent phantoms, InnoDB uses an algorithm called next-key locking that combines index-row locking with gap locking.
  You can use next-key locking to implement a uniqueness check in your application: If you read your data in share mode and do not see a duplicate for a row you are going to insert, then you can safely insert your row and know that the next-key lock set on the successor of your row during the read prevents anyone meanwhile inserting a duplicate for your row. Thus, the next-key locking enables you to “lock” the nonexistence of something in your table.
  
即InnoDb在REPEATABLE-READ下提供Next-Key Lock机制，但是需要业务自己去加锁，如果不加锁，只是简单的SELECT查询，是无法限制并行事务的插入的。


## 误解三
> 凡是REPEATABLE-READ中的读都无法读取最新的数据。

这个观点也是错误的，虽然我们读取的记录都是可重复读取的，但是如果你想读取最新的记录可以用加锁的方式读。

> If you want to see the “freshest” state of the database, you should use either the READ COMMITTED isolation level or a locking read:

以下任意一种均：
- SELECT * FROM table1 LOCK IN SHARE MODE;
- SELECT * FROM table1 FOR UPDATE;

但这里要说明的是这样做跟SERIALIZABLE没有什么区别，即读也加了锁，性能大打折扣。

## 误解四
> 如果使用了当前读加了锁，但是锁的行并不存在则不会阻止隔壁事务插入符合条件的数据。

其实记录存在与否和事务加锁成功与否无关，如SELECT * FROM user WHERE id = 5 FOR 
UPDATE，此时id=5的记录不存在，隔壁事务仍然无法插入记录（假设当前自增的主键id已经是4了）。因为锁定的是索引，故记录实体存在与否没关系。

## 误解五
> MySQL中的幻读只有在读的时候才会发生，读这里特指SELECT操作。

其实INSERT也是隐式的读取，只不过是在MySQL的机制中读取的，插入数据也是要先读取一下有没有主键冲突才能决定是否执行插入的。
不可重复读测试「读-读」，而幻读侧重「读-写」，用写来证实读的是幻影。为啥幻读不是侧重「读-读」呢？因为MVCC保证了一个事务是不可能读到另外一个事务的新插入数据的，所以这种场景下不会发生幻读。

# 参考
- https://dev.mysql.com/doc/refman/8.0/en/innodb-next-key-locking.html
- https://dev.mysql.com/doc/refman/8.0/en/innodb-consistent-read.html

> 本文首次发布于 [S.L's Blog](https://liushuo.me), 作者 [@stuartlau](http://github.com/stuartlau) ,转载请保留原文链接.
