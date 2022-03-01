---
title: mysql之索引底层数据结构
date: 2022-02-20 17:58:03
tags:
    - mysql
categories:
    - mysql
---

# mysql之索引底层数据结构
索引：
<b>A database index is a <font color='red'>data structure</font> that improves the speed of operations in a table. Indexes can be created using one or more columns, providing the basis for both <font color='red'>rapid random lookups and efficient ordering</font> of access to records.</b>

# 索引数据结构
1. Hash表
2. 二叉树  
3. 红黑树
4. Btree

## Hash表
![Hash表结构](/images/mysql-1-1.png)
1. hash表结构通过hash的方式来获取数据存储的位置
2. 很多时候hash索引要比B+tree索引高效
3. <font color='red'><b>仅仅支持“=、in"的方式查找，范围查找不行</b></font>
4. <font color='red'><b>hash冲突，hash表扩大</b></font>


<!--more -->

## 二叉树
![二叉树结构](/images/mysql-1-2.png)
1. 过半查找
2. <font color='red'><b>二叉树跟节点插入的顺序息息相关，也影响查找的效率</b></font>
3. <font color='red'><b>一旦节点数量变的很多，树的深度会变得非常大，也影响查找的效率</b></font>

## 红黑树
![红黑树结构](/images/mysql-1-3.png)
1. 过半查找
2. 对插入的顺序不在意
3. <font color='red'><b>这里的问题是一旦数据量大的，树的深度也是非常大</b></font>

## Btree
![Btree](/images/mysql-1-4.png)
1. 过半查找
2. 范围查找
3. 对插入的顺序不在意
4. <font color='red'><b>叶子节点具有相同的深度，叶节点的指针为空</b></font>
5. <font color='red'><b>所有的节点数据都不为空</b></font>
6. <font color='red'><b>节点中的数据从左至右递增</b></font>

> 相信上面一个对比分析，Btree是最满足我们的需求的，mysql的底层是用的Btree吗?答案是的，但是mysql底层对Btree做了改良，更加适用于自己的数据结构，<font color='red'><b>B+Tree</b></font>

## B+Tree
![B+tree](/images/mysql-1-5.png)
B+Tree是在Btree上面做的改良，下面是B+Tree一些的特性
1. <font color='red'><b>非叶子节点不存储真实数据，只存储索引，这样可以放更多的索引</b></font>
2. <font color='red'><b> 叶子节点包含所有的索引字段</b></font>
3. <font color='red'><b>叶子节点用指针相互链接，提供区间的访问的性能</b></font>
4. 还有一个特性需要看下，二级节点里面的数据都是叶子节点的首节点，这样可以提高定位能力
5. 从左至右递增


### B+Tree结构
在mysql里面底层使用的B+Tree，一个节点就是一页的数据，在mysql innodb里面一页数据是多少？下面的命令就是16KB。
```mysql
SHOW GLOBAL STATUS like 'Innodb_page_size';
```
|Variable_name|Value|
|:--:|:--:|
|Innodb_page_size|16384|

那么一页的结构是什么？见下图
![B+tree page结构](/images/mysql-1-6.png)
```text
// 非叶子节点能存储多少索引记录
(页 - File Header - Page Hear - File Trailer - ...) / (主键 + 记录的偏移量 + ...)
(16kB - 38B - 56B - 8B - ...) / (4B + 4B + ...)
约等于2000
```
```text
// 叶子节点能存储多少索引记录
(页 - File Header - Page Hear - File Trailer - ...) / (主键 + 数据 + 记录的偏移量 + ...)
(16kB - 38B - 56B - 8B - ...) / (4B + 1KB + 4B + ...)
约等于15
```
<font color='red'><b>树高为3的B+Tree能存多少索引数据呢？ 2000 * 2000 * 15 = 60000000 条记录，千万级数量级</b></font>

# BTree和B+Tree不同之处
|类型|非叶子节点是否有数据|叶子节点之间是否有指针|
|:--:|:--:|:--:|
|B|有|没有|
|B+|没有|有|

# 存储引擎
mysql中的数据用各种不同的技术存储在文件（或者内存）中。这些技术中的每一种技术都使用不同的存储机制、索引技术、锁定水平并且最终提供广泛的不同的功能和能力。

mysql常见的存储引擎有哪些？
```mysql
SHOW ENGINES;
```
|  Engine   | Support  |  Comment   | Transactions  |  XA   | Savepoints  |
|  :----  | :----:  |  :----  | :----:  |  :----:  | :----:  |
| InnoDB  | DEFAULT | Supports transactions, row-level locking, and foreign keys  | YES | YES  | YES |
| MRG_MYISAM  | YES | Collection of identical MyISAM tables  | NO | NO  | NO |
| MEMORY  | YES | Hash based, stored in memory, useful for temporary tables  | NO | NO  | NO |
| BLACKHOLE  | YES | /dev/null storage engine (anything you write to it disappears)  | NO | NO  | NO |
| MyISAM  | YES | MyISAM storage engine  | NO | NO  | NO |
| CSV  | YES | CSV storage engine  | NO | NO  | NO |
| ARCHIVE  | YES | Archive storage engine | NO | NO  | NO |
| PERFORMANCE_SCHEMA  | YES | Performance Schema  | NO | NO  | NO |
| FEDERATED  | NO | Federated MySQL storage engine  |  |   |  |

<font color='red'><b>存储引擎是基于表的，而不是数据库。mysql默认存储引擎就是innodb</b></font>

这里说明下比较常用的两种存储引擎Myisam和Innodb

## Myisam（非聚集索引）
```mysql
create table myisam_test
(
    id   int  not null primary key,
    name varchar(40) null
) engine = MyISAM;
```
创建上面这张表，指定存储引擎MyISAM，创建成功后会生成下面三个文件`*.frm、*.MYI、*.MYD`，其中前缀是表名。
```mysql
*.frm framework简写，表名相关信息
*.MYI MyISAM Index，索引信息
*.MYD MyISAM DATA，数据
```
![MyISAM存储文件](/images/mysql-1-7.jpg)


![MyISAM索引存储数据](/images/mysql-1-9.png)
<b>叶子节点存储的是地址，拿到地址后，从*.MYD文件里面拿真正的行数据</b>

## Innodb
```mysql
create table innodb_test
(
    id   int  not null primary key,
    name varchar(40) null
) engine = innodb;
```
创建上面这张表，指定存储引擎Innodb，创建成功后会生成下面三个文件`*.frm、*.ibd`，其中前缀是表名。
![](/images/mysql-1-8.jpg)
```mysql
*.frm framework简写，表名相关信息
*.ibd 索引信息和数据
```

![Innodb主键索引存储数据](/images/mysql-1-10.png)
上图中，<b><font color='red'>主键索引（聚集索引）</font>中，叶子节点存储的是这一行数据</b>

![Innodb非主键索引存储数据](/images/mysql-1-11.png)
上图中，<b><font color='red'>非主键索引（非聚集索引）</font>中，叶子节点存储的是这一行数据主键值，然后拿到主键值后再到主键索引B+Tree拿到那一行数据</b>

> 聚集索引就是叶子节点包含完整的数据

## Innodb引申问题
* 为什么建议Innodb创建表时要建主键，并且必须是使用整形的自增主键？
    1. 在创建存储引擎是Innodb的表时，一定会创建主键索引，指定了主键那就使用主键，如果没有指定主键，那就依次的比较列，如果有一列值都不一样的话，就用这个列做主键，来构建主键索引B+Tree，如果这个列没有找到合适的，mysql会把隐藏列（类似于rowid）做主键。<font color='red'><b>这里显而易见，如果我们不建主键，mysql会做上面的事情，多么消耗时间，我们自己创建好，节省时间和计算能力</b></font>
    2. 为什么要使用整形呢？<font color='red'><b>在我们B+Tree的时候，节点从左到右都是递增的，在范围查找的时候，整形可以比较很快拿到数据，反而（拿uuid做例子来比较）构建索引的时候需要一个一个字符比较，非常繁琐，也浪费空间，整形也是非常节省空间的</b></font>
    3. 为什么使用自增呢？这个就比较难理解了，要理解可以见下图，<font color='red'><b>先插入5后插4，发现B+Tree会拆节点和平衡树，而自增插入是不会的</b></font>
    ![自增](/images/mysql-1-13.png)
* 为什么非主键索引（非聚集索引）叶子节点的内容存储的是主键值？
    1. 保证数据的一致
    2. 节省存储的空间

# 索引最左匹配规则
先看下联合索引的底层存储结构
![联合索引的底层存储结构](/images/mysql-1-12.png)
1. 单个节点内的数据是递增的
2. 节点间从左到右也是递增的
3. 先根据第一个字段排序，然后再去根据第二个字段排序，再去根据第三个字段排序

# 附
在线的数据结构的网址：https://www.cs.usfca.edu/~galles/visualization/Algorithms.html
