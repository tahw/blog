---
title: mysql之一个sql是如何执行的
date: 2022-03-04 19:00:32
tags:
    - mysql
categories:
    - mysql
---

# mysql之一个sql是如何执行的
架构图
![架构图](/images/mysql-3-1.png)
<!-- more -->
根据架构图可以看出，mysql可以分为服务层和存储引擎层。这一篇章主要介绍下服务层。

## 服务层
服务层是所有存储引擎所共享的，这里面包括连接器、查询缓存、词法分析器、优化器、执行器等，涵盖mysql大多核心服务功能，所有内置的函数、存储过程、触发器、视图都在这一层实现的

### 连接器
顾名思义，客户端发起通信跟服务端建立连接就是使用的连接器。连接和权限校验都是连接器做的。
![连接器](/images/mysql-3-2.png)

1. 客户端建立通信的是长链接，这个长链接是在连接器中。（mysql的长链接是链接成功后，如果客户端持续请求，则一直使用同一连接。）
2. <font color='red'><b>其中每个用户和连接器连接成功后，该用户的权限管理就会缓存在连接器中的。这里会有问题，如果连接成功后，我使用root权限修改了该用户的权限，这个对该长连接是不起作用的，但是一旦重新连接后，权限就会读取最新的。为什么mysql会这样设计，在高并发情况下，如果都去读mysql磁盘，压力会很大</b></font>
3. `show processlist ;`如果客户端长时间不发送命令到服务端就是sleep，其中里面有time，这个表示等待时间，如果这个时间超过wait_timeout（`show global variables like 'wait_timeout%';` 默认值是8小时），连接器则会自动断开。
![结果](/images/mysql-3-3.jpg)
![结果](/images/mysql-3-4.jpg)
4. 开发中我们使用的都是长链接，而长链接都是放在pool管理，那如果我们长时间不释放链接，会导致连接对象里面特别大，占用内存太大，就会oom了

### 查询缓存
连接器完成后，你就可以使用查询缓存了。具体流程见下面
![查询缓存](/images/mysql-3-5.png)
<font color='red'><b>但是大多情况下这个缓存比较鸡肋，没什么用</b></font>
1. 缓存失效非常频繁，只要对表做了增、删、改的操作，都会使这个缓存清空
2. 但是这种场景肯定有使用之处，一般这种是给修改操作不频繁的表使用的。城市、字典表

查询是否开启缓存 `show global variables like '%query_cache_type%';`，默认是关闭的
查询缓存命中`show status like '%Qcache%';`

这里就不过多介绍了

> mysql 8.0已经移除了查询缓存功能

### 词法分析器
没有命中查询缓存，则开始真正执行语句。<font color='red'><b>那这个时候分析器是做什么呢？其实你要执行sql，mysql需要知道你的sql是要干什么？分析器就是来解析你的sql。</b></font>
![流程](/images/mysql-3-6.png)


idea 插件 ANTLR v4可以看到语法树，`select a,b from c where d = 1`形成的语法树见下图
![语法树](/images/mysql-3-7.png)
> 这里的语法树其实有一个场景使用：分库分表，指定某个字段来区分

这样上面分析器的工作就结束了。

### 优化器
那知道你的sql要做什么了，接下来真正执行的时候，还需要经过优化器。<font color='red'><b>那优化器是干什么呢？优化器是在表里有多个索引的时候，决定使用哪个索引，或者多表关联的时候，决定表的连接顺序</b></font>

```sql
select id,name from zoo ;
```

![执行结果](/images/mysql-3-8.jpg)

上面这个sql可以看出是没有使用索引的，也没有条件查询，但是看下explain执行计划，是有使用索引index_name的。


> <b>优化器阶段完成后，这个语句的执行方案就确定下来如何执行。</b>

### 执行器
开始执行的时候，会校验有没有对表有查询、操作权限，如果没有提示错误，然后验证完后调用引擎提供的接口，然后拿到结果放到结果集后返回客户端

### binlog
<font color='red'><b>
特性：
1. binlog是服务层备份的文件。是所有的存储引擎公用的
2. binlog为逻辑日志，记录的是一条语句的原始逻辑
3. binlog是不限大小的，追加写入，不会覆盖以前的日志

</b></font>

#### 开启binlog

`show variables like '%log_bin%';` 查看是否开启binlog日志，默认是不开启的

![执行结果](/images/mysql-3-9.jpg)

开启binlog只需要修改mysql配置文件my.cnf文件。
```txt
# 文件目录：/etc/mysql/my.cnf
[mysqld]
log_bin=/var/lib/mysql/binlog/binlog
# 其中binlog_format格式有三种，statement、row、mixed
binlog_format=row
sync-binlog=1
server-id=1
```

#### binlog日志格式
1. row
日志中会记录成每一行数据被修改的形式，然后在 slave 端再对相同的数据进行修改。<font color='red'><b>优点是会清除记下每一行数据修改的细节，非常容易理解和恢复。但是缺点会导致大量的日志内容</b></font>

2. statement
每一条会修改数据的 SQL 都会记录到 master 的 bin-log 中。slave 在复制的时候 SQL 进程会解析成和原来 master 端执行过的相同的 SQL 再次执行。<font color='red'><b>优点是减少binlog日志量，节省的I/O。但是缺点是有可能无法正确恢复数据。</b></font>

3. mixed
statement和row的模式的组合。遇到表结构修改的时候是以statement方式来存储，而表数据的修改是以row方式来存储。

#### 查看binlog
```sql
mysqlbinlog /var/lib/mysql/binlog/binlog.000001
mysqlbinlog /var/lib/mysql/binlog/binlog.000002
```
```sql
# 经过flush logs;刷盘日志后，会生成一个新的日志文件，binlog.000001会保留，然后会接着binlog.000001的时间点生成binlog.000002文件
mysql > flush logs; 
# 查看最后一个binlog文件的信息
mysql>  show master status;
```
![执行结果](/images/mysql-3-10.png)
![执行结果](/images/mysql-3-13.jpg)

在看binlog文件里面，可以看到begin、commmit这种关键字，那这中间就是一个完整的事务逻辑。
![执行结果](/images/mysql-3-11.png)


#### binlog恢复数据
mysqlbinlog可以通过很多方式恢复数据，根据位点、时间、全部日志文件来恢复
```sql
# 情况表数据
truncate animals;
# 回车输入密码，详细看下mysqlbinlog --help看下
mysqlbinlog --start-position=0 --stop-position=787 binlog.000002|mysql -uroot -p tahw（这个是数据库名）
# 下面结果恢复只有两条，这是因为mysqlbinlog开启后才记录两条操作，开启之前的数据是无法恢复的
```
![执行结果](/images/mysql-3-12.jpg)

## 存储引擎层
存储引擎层是和数据打交道的，负责数据的存储和提取。架构模式是插件式的。支持Innodb、MyISAM等多个存储引擎。
