---
title: mysql之MVCC和BufferPool缓存机制
date: 2022-03-31 10:39:24
tags:
    - mysql
categories:
    - mysql
---

# mysql之MVCC和BufferPool缓存机制

# MVCC

mvcc（Multiversion Concurrency Control）多版本并发控制机制。MVCC提升事务并发处理能力。要介绍MVCC必须先介绍undo日志链和read-view一致性视图。MVCC是使用这两个来实现的。
> 读已提交和可重复读都实现MVCC

## undo日志链
undo日志版本链是指一行数据被多个事务依次修改后，在每个事务修改后，mysql会保留修改前的数据。这个就是undo日志。<font color='red'><b>其中使用两个隐藏的字段trx_id（事务id）和roll_pointer（指针）把undo给串联起来。</b></font>
![undo日志链](/images/mysql-6-1-undo_log.png)

<!-- more -->

## read-view一致性视图
在可重复读的隔离级别下，只要开启事务，然后在执行任何sql前，就会生成事务的一致性视图read-view。见下图。在事务结束这里也不会修改视图。

read-view视图下面由两个部分组成：<font color='red'><b>所有未提交事务id数组(min_id：数组里面最小的id)、最大事务id(max_id)</b></font>

> 读已提交的情况下是每次查询前，就会生成一致性视图read-view。


![read-view](/images/mysql-6-2-read-view.png)

版本比对的规则：
1. 如果查询行的时候，从上到下比对undo日志版本链，如果trx_id(事务id) < min_id，就是表示事务已经提交，这个数据是可见的。
2. 如果查询行的时候，min_id <= trx_id <= max_id。这个还需要查一下一致性视图。
    * 如果 trx_id 在一致性视图里面的未提交集合里面，则不可见。
    * 否则，可见。
3. 如果查询行的时候，trx_id > max_id，表示当前生成视图的时候还未发生的事务，不可见。


## MVCC例子
仔细阅读下例子，就能清楚知道MVCC

![MVCC例子](/images/mysql-6-3-MVCC例子.png)


# Buffer Pool
流程图：
![Buffer-Pool](/images/mysql-6-4-buffer-pool.png)

流程：
1. 从磁盘load数据到Buffer Pool里面
2. 写Undo日志
3. 更新Buffer Pool的值
4. 写redo log buffer里面
5. 开始提交事务的时候，将redo log buffer里面的数据写入Redo log里面
6. 写binlog
7. commmit事务时，将commit标记写入redo log里面，保证事务提交后redo log和binlog数据一致
8. buffer poll 异步提交任务到I/O线程里，异步page刷盘

> 有一点要注意，Buffer Pool里面的数据是新的，有可能磁盘的数据都是老的。操作都是在Buffer Pool里面

## Redo Log
顾名思义：重做。重做什么呢？其实是备份Buffer Pool里面的内容，保证Buffer Pool宕机了，数据还能被恢复。

## 为什么要写Buffer Pool？不直接写磁盘？
磁盘的操作是随机的（有可能库里的数据被删了，导致存储的page里面是不相连的），性能会非常慢。而写日志是顺序写日志。读取操作是非常快的。数据的一致性也能保证。性能也是非常高。这套机制是非常好的。