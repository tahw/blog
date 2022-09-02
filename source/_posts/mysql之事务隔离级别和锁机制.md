---
title: mysql之事务隔离级别和锁机制
date: 2022-03-18 18:29:12
tags:
    - mysql
categories:
    - mysql
---
# 事务

## 概念
事务是由一组sql语句组成的逻辑单元，他们要么成功提交，要么失败都回滚掉。
> 原文：Transactions are atomic units of work that can be committed or rolled back. When a transaction makes multiple changes to the database, either all the changes succeed when the transaction is committed, or all the changes are undone when the transaction is rolled back. <br>
Database transactions, as implemented by InnoDB, have properties that are collectively known by the acronym ACID, for atomicity, consistency, isolation, and durability.

## ACID
事务的特性ACID

### A - atomicity（原子性）
事务是一个原子的操作单元，要么全部提交，要么全部回滚

> 原文：Transactions are atomic units of work that can be committed or rolled back. When a transaction makes multiple changes to the database, either all the changes succeed when the transaction is committed, or all the changes are undone when the transaction is rolled back.


### C - consistency（一致性）
数据需要保持一致的状态，无论事务修改数据，读的都是一致数据。不能是中间数据

> 原文：The database remains in a consistent state at all times — after each commit or rollback, and while transactions are in progress. If related data is being updated across multiple tables, queries see either all old values or all new values, not a mix of old and new values.

### I - isolation（隔离性）
事务是隔离的。事务处理过程中的数据是不会被外部可见的。

> 原文：Transactions are protected (isolated) from each other while they are in progress; they cannot interfere with each other or see each other's uncommitted data. This isolation is achieved through the locking mechanism. Experienced users can adjust the isolation level, trading off less protection in favor of increased performance and concurrency, when they can be sure that the transactions really do not interfere with each other.

### D - durability（持久性）
事务结果是持久性的，即使系统出现故障，也能恢复。

> 原文：The results of transactions are durable: once a commit operation succeeds, the changes made by that transaction are safe from power failures, system crashes, race conditions, or other potential dangers that many non-database applications are vulnerable to. Durability typically involves writing to disk storage, with a certain amount of redundancy to protect against power failures or software crashes during write operations. (In InnoDB, the doublewrite buffer assists with durability.)

<!-- more -->

# 隔离级别
隔离级别是做什么呢？隔离级别是因为在事务过程中有可能产生”脏写“、”脏读“、”不可重复读“、”幻读“的问题。mysql通过隔离级别来解决这些问题。<font color='red'><b>事务既然有ACID特性，怎么产生”脏写“、”脏读“、”不可重复读“、”幻读“，这里一定要记住，单个事务不会产生，但是并发情况下，多个事务情况下就有可能产生</b></font>

链接：https://dev.mysql.com/doc/refman/8.0/en/innodb-transaction-isolation-levels.html

|命令|对应结果|
| :-- | :-- |
| 查询 | select @@tx_isolation;|
| 设置 | SET {SESSION or GLOBAL} TRANSACTION ISOLATION LEVEL { READ UNCOMMITTED or READ COMMITTED or REPEATABLE READ or SERIALIZABLE };|


| 隔离级别 | 说明 | 原文 |
| :-- | :-- | :-- |
|READ-UNCOMMITTED|读未提交|The isolation level that provides the least amount of protection between transactions. Queries employ a locking strategy that allows them to proceed in situations where they would normally wait for another transaction. However, this extra performance comes at the cost of less reliable results, including data that has been changed by other transactions and not committed yet (known as <font color='red'><b>dirty read</b></font>). Use this isolation level with great caution, and be aware that the results might not be consistent or reproducible, depending on what other transactions are doing at the same time. Typically, transactions with this isolation level only do queries, not insert, update, or delete operations.|
|READ-COMMITTED|读已提交|An isolation level that uses a locking strategy that relaxes some of the protection between transactions, in the interest of performance. <font color='red'><b>Transactions cannot see uncommitted data from other transactions, but they can see data that is committed by another transaction after the current transaction started. </b></font>Thus, a transaction never sees any bad data, but the data that it does see may depend to some extent on the timing of other transactions.<br>When a transaction with this isolation level performs UPDATE ... WHERE or DELETE ... WHERE operations, other transactions might have to wait. The transaction can perform SELECT ... FOR UPDATE, and LOCK IN SHARE MODE operations without making other transactions wait.<br>SELECT ... FOR SHARE replaces SELECT ... LOCK IN SHARE MODE in MySQL 8.0.1, but LOCK IN SHARE MODE remains available for backward compatibility.|
|REPEATABLE-READ|可重复读|<font color='red'><b>The default isolation level for InnoDB.</b></font> It prevents any rows that are queried from being changed by other transactions, <font color='red'><b>thus blocking non-repeatable reads but not phantom reads（幻读）</b></font>. It uses a moderately strict locking strategy so that all queries within a transaction see data from the same snapshot, that is, the data as it was at the time the transaction started.<br>When a transaction with this isolation level performs UPDATE ... WHERE, DELETE ... WHERE, SELECT ... FOR UPDATE, and LOCK IN SHARE MODE operations, other transactions might have to wait.<br>SELECT ... FOR SHARE replaces SELECT ... LOCK IN SHARE MODE in MySQL 8.0.1, but LOCK IN SHARE MODE remains available for backward compatibility.|
|SERIALIZABLE|可串行化|The isolation level that uses the most conservative locking strategy, to prevent any other transactions from inserting or changing data that was read by this transaction, until it is finished. This way, the same query can be run over and over within a transaction, and be certain to retrieve the same set of results each time. Any attempt to change data that was committed by another transaction since the start of the current transaction, cause the current transaction to wait.<br>This is the default isolation level specified by the SQL standard. In practice, this degree of strictness is rarely needed.|

|隔离级别|脏读（dirty read）|不可重复读（non-repeatable reads）|幻读（phantom read）|
| :-- | :-- | :-- | :-- |
| READ-UNCOMMITTED |✅|✅|✅|
| READ-COMMITTED |❌|✅|✅|
| REPEATABLE-READ(默认)|❌|❌|✅|
| SERIALIZABLE |❌|❌|❌|

* 脏写
当两个或者多个事务操作同一行时，由于每个事务不知道其他事务的存在，就会发生事务覆盖更新问题
![脏写](/images/mysql-5-1.png)

* 脏读
<font color='red'><b>一个事务A读取到另外一个事务B已经修改的数据但是没有提交的，还对此做的操作，这就叫做脏读。如果事务B回滚操作，不符合一致性的要求。</b></font>
![脏读](/images/mysql-5-2.png)

* 不可重复读
一个事务在读取某些数据后，然后再次读取以前的数据，却发现其读取到的数据已经发生改变，这种就是不可重复读。
<font color='red'><b>同一个事务在不同时间内读到的结果不一致，不符合隔离性。</b></font>

![不可重复读](/images/mysql-5-3.png)

* 幻读
一个事务按相同的查询条件重新读取以前检索过的数据，却发现其他事务满足条件的新数据，这种就是幻读。
<font color='red'><b>事务A读取到事务B提交的新增数据，不符合隔离性。</b></font>

![幻读](/images/mysql-5-4.png)

## 脏读、不可重复读、幻读的区别
<font color='red'><b>

1. 不可重复读强调的是同一事务下，两次读取的数据不一样
2. 幻读强调的是另外一个事务是新增或者删除，两次读取的记录不一样
3. 脏读强调的是另外一个事务是修改，读的不够新

</b></font>

# 锁
锁也是解决多个事务产生问题的一种方式

## 分类
* 性能
    * 乐观锁（可以采用版本的字段来维护）
    * 悲观锁
* 数据库操作的类型
    * 读锁（悲观锁）：共享锁、S(share)锁，针对同一份数据同时读取互不影响，可以读取
    * 写锁（悲观锁）：排他锁、X(exclusive)锁，针对当前数据的写，会阻塞其他的写锁和读锁
* 数据操作的粒度
    * <font color='red'><b>表锁：每次锁住整张表，开销小、加锁快、不会出现死锁、锁的粒度大、并发低（一般用作数据迁移）</b></font>
    * <font color='red'><b>行锁：每次锁住一行数据，开销大、加锁慢，会出现死锁、锁的粒度小、并发大</b></font>

# 锁例子

## 表锁
```sql
# 表锁命令
lock tables 表 read(write),表 read(write);

# 释放表锁
unlock tables;
```

### 表读锁
```sql
lock tables tahw.person read ;
```
|事务|读|写|
| :-- | :-- | :-- |
|当前事务| ✅ | ❌（Table 'person' was locked with a READ lock and can't be updated）|
|其他事务| ✅ | ❌（等待一直阻塞，直到当前事务释放，其他事务就会成功）|

如何查询表锁？
```sql
show open tables;
```
![查询表锁](/images/mysql-5-6.jpg)

### 表写锁
```sql
lock tables tahw.person write ;
```
|事务|读|写|
| :-- | :-- | :-- |
|当前事务| ✅ | ✅|
|其他事务| ❌（等待一直阻塞，直到当前事务释放，其他事务就会成功） | ❌（等待一直阻塞，直到当前事务释放，其他事务就会成功）|

![写锁-当前事务](/images/mysql-5-7.png)
![写锁-另外事务](/images/mysql-5-8.png)

## 行锁
不同的隔离级别在行锁表现是不一样的

### read-uncommited
![读未提交](/images/mysql-5-9.png)
1. 在第5步的时候就出现脏读

### read-committed
![读已提交](/images/mysql-5-10.png)
1. 在第7步的时候没有出现脏读
2. 但是第7步和第9步的结果不一致，出现了不可重复读

### repeatable-read
![不可重复读](/images/mysql-5-11.png)
1. 在第8步的时候没有出现脏读
2. 在第8步和第10步结果没有不一致，解决了不可重复读
3. 但是在11步的时候更新id=9的数据，第12步的name也变成了beijing。这个结果就是幻读。我只是更新了input_time的字段，name没有调整。
4. <font color='red'><b>第14步为什么读取的之前的数据？数据一致性没有破坏，可重复读下面使用了MVCC机制(多版本并发控制)，select不会更新版本号，读取的是历史版本，而insert、update和delete是会更新版本的。</b></font>

#### 间隙锁(Gap Lock)
间隙锁在某种情况下可以解决可重复读下产生的幻读。什么是间隙锁？其实就是行的空隙。在相互更新的情况下会相互影响
![间隙锁](/images/mysql-5-13.png)
<font color='red'><b>id的间隙就是(1,9)、（10，20）、（20，无穷大）这三个区间，事务A更新id在（8，19）数据。这个时候事务B如果更新id = 20不能更新，更新id = 1可以更新。这个就是事务A在（8，19）上加了间隙锁，但是为什么20包括在内，20是临键锁。那其实是（1，20]都被加上锁了。其他是临键锁。</b></font>

![间隙锁](/images/mysql-5-14.png)

#### 临键锁(Next-key locks)
临键锁是配合间隙锁来说。行锁和间隙锁的组合。（1，20]整个区间都叫做临键锁

### serializable
![幻读](/images/mysql-5-12.png)
1. 第9步直接阻塞，不会查询结果，不会出现脏读、不可重复读、幻读。这种隔离级别并发性低，但是不是很方便，很容易阻塞
<font color='red'><b>为什么第9步会阻塞，这个就是查询全部，这个所有行记录所在的间隙区间范围都会被加锁（该行数据还未插入也会被加锁），这种就是间隙锁。</b></font>

## 行锁升级为表锁
<font color='red'><b>Innodb的行锁都是针对索引添加的锁，不是针对记录添加的。只要索引不失效，否则都会从行锁升级为表锁。</b></font>
![行锁升级表锁](/images/mysql-5-15.png)

## 共享锁
```sql
select * from animals lock in share mode ;
```

## 排他锁
```
select * from animals for update ;
```

## 行锁分析
```sql
show status like 'innodb_row_lock%';
```
![行锁分析](/images/mysql-5-16.png)

* Innodb_row_lock_current_waits：当前正在等待锁定的数量
* Innodb_row_lock_time：从系统启动到现在锁定总时间长度
* Innodb_row_lock_time_avg：每次等待所花平均时间
* Innodb_row_lock_time_max：从系统启动到现在等待最长的一次所花时间
* Innodb_row_lock_waits：从系统启动后到现在总共等待的次数

## 死锁
![死锁](/images/mysql-5-17.png)

1. 直接就返回dead lock
2. `show engine innodb status;`查询结果，也可以看出死锁

```text

=====================================
2022-03-28 14:48:45 0x7f8aa4222700 INNODB MONITOR OUTPUT
=====================================
Per second averages calculated from the last 12 seconds
-----------------
BACKGROUND THREAD
-----------------
srv_master_thread loops: 31 srv_active, 0 srv_shutdown, 12959 srv_idle
srv_master_thread log flush and writes: 12990
----------
SEMAPHORES
----------
OS WAIT ARRAY INFO: reservation count 2578
OS WAIT ARRAY INFO: signal count 2006
RW-shared spins 0, rounds 49, OS waits 2
RW-excl spins 0, rounds 31, OS waits 1
RW-sx spins 0, rounds 0, OS waits 0
Spin rounds per wait: 49.00 RW-shared, 31.00 RW-excl, 0.00 RW-sx
------------------------
LATEST DETECTED DEADLOCK
------------------------
2022-03-28 14:45:59 0x7f8aa411a700
*** (1) TRANSACTION:
TRANSACTION 1057876, ACTIVE 46 sec starting index read
mysql tables in use 1, locked 1
LOCK WAIT 3 lock struct(s), heap size 1136, 2 row lock(s)
MySQL thread id 19, OS thread handle 140233435645696, query id 3715 127.0.0.1 root statistics
select * from animals where id = 21 for update
*** (1) WAITING FOR THIS LOCK TO BE GRANTED:
RECORD LOCKS space id 29 page no 3 n bits 80 index PRIMARY of table `tahw`.`animals` trx id 1057876 lock_mode X locks rec but not gap waiting
Record lock, heap no 2 PHYSICAL RECORD: n_fields 5; compact format; info bits 0
 0: len 4; hex 80000015; asc     ;;
 1: len 6; hex 00000010244a; asc     $J;;
 2: len 7; hex c0000001370110; asc     7  ;;
 3: len 3; hex 616161; asc aaa;;
 4: len 5; hex 99ac78e645; asc   x E;;

*** (2) TRANSACTION:
TRANSACTION 1057877, ACTIVE 41 sec starting index read
mysql tables in use 1, locked 1
3 lock struct(s), heap size 1136, 2 row lock(s)
MySQL thread id 20, OS thread handle 140233434834688, query id 3716 127.0.0.1 root statistics
select * from animals where id = 1 for update
*** (2) HOLDS THE LOCK(S):
RECORD LOCKS space id 29 page no 3 n bits 80 index PRIMARY of table `tahw`.`animals` trx id 1057877 lock_mode X locks rec but not gap
Record lock, heap no 2 PHYSICAL RECORD: n_fields 5; compact format; info bits 0
 0: len 4; hex 80000015; asc     ;;
 1: len 6; hex 00000010244a; asc     $J;;
 2: len 7; hex c0000001370110; asc     7  ;;
 3: len 3; hex 616161; asc aaa;;
 4: len 5; hex 99ac78e645; asc   x E;;

*** (2) WAITING FOR THIS LOCK TO BE GRANTED:
RECORD LOCKS space id 29 page no 3 n bits 80 index PRIMARY of table `tahw`.`animals` trx id 1057877 lock_mode X locks rec but not gap waiting
Record lock, heap no 8 PHYSICAL RECORD: n_fields 5; compact format; info bits 0
 0: len 4; hex 80000001; asc     ;;
 1: len 6; hex 000000102449; asc     $I;;
 2: len 7; hex 3f00000188043f; asc ?     ?;;
 3: len 1; hex 32; asc 2;;
 4: len 5; hex 99abc20000; asc      ;;

*** WE ROLL BACK TRANSACTION (2)
------------
TRANSACTIONS
------------
Trx id counter 1057878
Purge done for trx's n:o < 1057872 undo n:o < 0 state: running but idle
History list length 0
LIST OF TRANSACTIONS FOR EACH SESSION:
---TRANSACTION 421708774710928, not started
0 lock struct(s), heap size 1136, 0 row lock(s)
---TRANSACTION 421708774709088, not started
0 lock struct(s), heap size 1136, 0 row lock(s)
---TRANSACTION 1057876, ACTIVE 212 sec
3 lock struct(s), heap size 1136, 2 row lock(s)
MySQL thread id 19, OS thread handle 140233435645696, query id 3715 127.0.0.1 root
Trx read view will not see trx with id >= 1057876, sees < 1057876
--------
FILE I/O
--------
I/O thread 0 state: waiting for completed aio requests (insert buffer thread)
I/O thread 1 state: waiting for completed aio requests (log thread)
I/O thread 2 state: waiting for completed aio requests (read thread)
I/O thread 3 state: waiting for completed aio requests (read thread)
I/O thread 4 state: waiting for completed aio requests (read thread)
I/O thread 5 state: waiting for completed aio requests (read thread)
I/O thread 6 state: waiting for completed aio requests (write thread)
I/O thread 7 state: waiting for completed aio requests (write thread)
I/O thread 8 state: waiting for completed aio requests (write thread)
I/O thread 9 state: waiting for completed aio requests (write thread)
Pending normal aio reads: [0, 0, 0, 0] , aio writes: [0, 0, 0, 0] ,
 ibuf aio reads:, log i/o's:, sync i/o's:
Pending flushes (fsync) log: 0; buffer pool: 0
7315 OS file reads, 371 OS file writes, 241 OS fsyncs
0.00 reads/s, 0 avg bytes/read, 0.00 writes/s, 0.00 fsyncs/s
-------------------------------------
INSERT BUFFER AND ADAPTIVE HASH INDEX
-------------------------------------
Ibuf: size 1, free list len 0, seg size 2, 0 merges
merged operations:
 insert 0, delete mark 0, delete 0
discarded operations:
 insert 0, delete mark 0, delete 0
Hash table size 34679, node heap has 0 buffer(s)
Hash table size 34679, node heap has 0 buffer(s)
Hash table size 34679, node heap has 0 buffer(s)
Hash table size 34679, node heap has 0 buffer(s)
Hash table size 34679, node heap has 0 buffer(s)
Hash table size 34679, node heap has 0 buffer(s)
Hash table size 34679, node heap has 0 buffer(s)
Hash table size 34679, node heap has 0 buffer(s)
0.00 hash searches/s, 0.00 non-hash searches/s
---
LOG
---
Log sequence number 601588829
Log flushed up to   601588829
Pages flushed up to 601588829
Last checkpoint at  601588820
0 pending log flushes, 0 pending chkp writes
158 log i/o's done, 0.00 log i/o's/second
----------------------
BUFFER POOL AND MEMORY
----------------------
Total large memory allocated 137428992
Dictionary memory allocated 195260
Buffer pool size   8192
Free buffers       1024
Database pages     7168
Old database pages 2626
Modified db pages  0
Pending reads      0
Pending writes: LRU 0, flush list 0, single page 0
Pages made young 97, not young 30
0.00 youngs/s, 0.00 non-youngs/s
Pages read 7179, created 52, written 175
0.00 reads/s, 0.00 creates/s, 0.00 writes/s
No buffer pool page gets since the last printout
Pages read ahead 0.00/s, evicted without access 0.00/s, Random read ahead 0.00/s
LRU len: 7168, unzip_LRU len: 0
I/O sum[0]:cur[0], unzip sum[0]:cur[0]
--------------
ROW OPERATIONS
--------------
0 queries inside InnoDB, 0 queries in queue
1 read views open inside InnoDB
Process ID=1, Main thread ID=140233358173952, state: sleeping
Number of rows inserted 4140, updated 13, deleted 0, read 1061269
0.00 inserts/s, 0.00 updates/s, 0.00 deletes/s, 0.00 reads/s
----------------------------
END OF INNODB MONITOR OUTPUT
============================
```

# 锁优化建议

1. <font color='red'><b>尽量使用索引来操作，不要让行锁升级为表锁</b></font>
2. <font color='red'><b>尽可能减少范围更新，避免间隙锁</b></font>
3. <font color='red'><b>合理使用索引，尽量缩小锁的范围</b></font>

# 附
mysql 名词解释地址：https://dev.mysql.com/doc/refman/8.0/en/glossary.html