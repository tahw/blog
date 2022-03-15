---
title: mysql之Explain详解
date: 2022-03-01 22:11:37
tags:
    - mysql
categories:
    - mysql
---

# mysql之Explain详解
explain也叫做执行计划。
官方说明：<b>The EXPLAIN statement provides information about how MySQL executes statements. EXPLAIN works with SELECT, DELETE, INSERT, REPLACE, and UPDATE statements.</b>

# 前言
这里是使用mysql5.7来介绍

# explain 输出列

| Column | JSON Name | Meaning |
| :-- | :-- | :-- |
|id|select_id|The SELECT identifier|
|select_type|None|The SELECT type|
|table|table_name|The table for the output row|
|partitions|partitions|The matching partitions|
|type|access_type|The join type|
|possible_keys|possible_keys|The possible indexes to choose|
|key|key|The index actually chosen|
|key_len|key_length|The length of the chosen key|
|ref|ref|The columns compared to the index|
|rows|rows|Estimate of rows to be examined|
|filtered|filtered|Percentage of rows filtered by table condition|
|Extra|None|Additional information|

<!-- more -->

## 数据准备

```sql
create table animals
(
    id int primary key auto_increment,
    name varchar(20) not null,
    input_time datetime not null
) engine = innodb ;

INSERT INTO tahw.animals (id, name, input_time) VALUES (1, '朝阳动物园', '2022-03-03 11:15:59');
INSERT INTO tahw.animals (id, name, input_time) VALUES (2, '顺义动物园', '2022-03-03 11:16:22');
INSERT INTO tahw.animals (id, name, input_time) VALUES (3, '通州动物园', '2021-03-03 11:16:40');
INSERT INTO tahw.animals (id, name, input_time) VALUES (4, '昌平动物园', '2021-09-03 11:16:59');
INSERT INTO tahw.animals (id, name, input_time) VALUES (5, '海淀动物园', '2018-03-03 11:17:28');

create table zoo
(
    id int primary key auto_increment,
    name varchar(20) not null,
    key index_name(`name`)
) engine = innodb ;

INSERT INTO tahw.zoo (id, name) VALUES (6, '兔子');
INSERT INTO tahw.zoo (id, name) VALUES (3, '大象');
INSERT INTO tahw.zoo (id, name) VALUES (5, '海豚');
INSERT INTO tahw.zoo (id, name) VALUES (1, '猴子');
INSERT INTO tahw.zoo (id, name) VALUES (2, '长颈鹿');
INSERT INTO tahw.zoo (id, name) VALUES (4, '鲨鱼');

create table zoo_animals
(
    id int primary key auto_increment,
    zoo_id int not null,
    animals_id int not null,
    remark varchar(200) default null,
    unique key index_animals_id_zoo_id(`animals_id`,`zoo_id`)
) engine = innodb;

INSERT INTO tahw.zoo_animals (id, zoo_id, animals_id, remark) VALUES (3, 1, 1, null);
INSERT INTO tahw.zoo_animals (id, zoo_id, animals_id, remark) VALUES (4, 1, 2, '随便测试数据');
INSERT INTO tahw.zoo_animals (id, zoo_id, animals_id, remark) VALUES (5, 2, 3, null);
INSERT INTO tahw.zoo_animals (id, zoo_id, animals_id, remark) VALUES (6, 2, 1, '哈哈');
INSERT INTO tahw.zoo_animals (id, zoo_id, animals_id, remark) VALUES (7, 5, 6, null);

```

## 使用方法

```sql
{EXPLAIN | DESCRIBE | DESC}
    tbl_name [col_name | wild]

{EXPLAIN | DESCRIBE | DESC}
    [explain_type]
    {explainable_stmt | FOR CONNECTION connection_id}

explain_type: {
    EXTENDED
  | PARTITIONS
  | FORMAT = format_name
}

format_name: {
    TRADITIONAL
  | JSON
}

explainable_stmt: {
    SELECT statement
  | DELETE statement
  | INSERT statement
  | REPLACE statement
  | UPDATE statement
}
```
一般我们在sql前面加`EXPLAIN`，也会加`EXPLAIN EXTENDED`、`EXPLAIN PARTITIONS`方式，其中`EXPLAIN EXTENDED`会在`EXPLAIN`基础上添加一些额外的信息，类似于extra字段信息，`EXPLAIN PARTITIONS`会比`EXPLAIN`增加partitions信息

## id & table

```sql
set session optimizer_switch='derived_merge=off';
explain select (select name from animals where id = 1) from (select id from zoo_animals where animals_id = 3) a;
# 恢复 set session optimizer_switch='derived_merge=on';
```
结果
![执行结果](/images/mysql-2-1.jpg)

1. id 列的编号就是select的序列号
2. 有几个select就有几个id
3. 执行的顺序按id降序的，id多大优先级越高，其中id会null最后执行
4. 看下上面的结构，table执行简单查询的时候，就是表名，但是其中id为1是最后的执行的，其中的table是由id和select_type组合而成，方式是<derivedN>，这是id为1是等待id 3执行完成，还有其他的形式
    * <unionM,N>: The row refers to the union of the rows with id values of M and N.
    * <derivedN>: The row refers to the derived table result for the row with an id value of N. A derived table may result, for example, from a subquery in the FROM clause.
    * <subqueryN>: The row refers to the result of a materialized subquery for the row with an id value of N. 

详细看：https://dev.mysql.com/doc/refman/5.7/en/explain-output.html#explain_table

## select_type
表示对应行是简单还是复杂查询

|序号|select_type Value|	JSON Name|	Meaning|备注|
| :--| :-- | :-- | :-- | :-- |
|1|SIMPLE|	None|	Simple SELECT (not using UNION or subqueries)|简单查询|
|2|PRIMARY	|None	|Outermost SELECT|复杂查询中最外层的select|
|3|UNION|	None|	Second or later SELECT statement in a UNION|在union中第二个和随后的select|
|4|DEPENDENT UNION|	dependent (true)|	Second or later SELECT statement in a UNION, dependent on outer query||
|5|UNION RESULT|	union_result|	Result of a UNION.||
|6|SUBQUERY|	None|	First SELECT in subquery|子查询，from之前的查询|
|7|DEPENDENT SUBQUERY|	dependent (true)|	First SELECT in subquery, dependent on outer query||
|8|DERIVED|	None|	Derived table|衍生表，在from之后的查询，临时查询结果|
|9|MATERIALIZED|materialized_from_subquery|	Materialized subquery||
|10|UNCACHEABLE SUBQUERY|	cacheable (false)|	A subquery for which the result cannot be cached and must be re-evaluated for each row of the outer query||
|11|UNCACHEABLE UNION|	cacheable (false)|	The second or later select in a UNION that belongs to an uncacheable subquery (see UNCACHEABLE SUBQUERY)||

### simple
简单查询
```sql
explain select name from animals where id = 1;
```
![执行结果](/images/mysql-2-2.jpg)

### primary & derived & subquery
primary：复杂查询中最外层的select
derived：衍生表，在from之后的查询，临时查询结果
subquery：子查询，from之前的查询

```sql
set session optimizer_switch='derived_merge=off';
explain select (select name from animals where id = 1) from (select id from zoo_animals where animals_id = 3) a;
# 恢复 set session optimizer_switch='derived_merge=on';
```
![执行结果](/images/mysql-2-1.jpg)

### union
```sql
explain select 1 from zoo_animals
union all
select 2 from zoo_animals
union all
select 3;
```
![执行结果](/images/mysql-2-3.jpg)


## partitions
分区信息，The partitions from which records would be matched by the query. The value is NULL for nonpartitioned tables. 

## type
表关联类型

|序号（序号越在前索引优化越好）|类型|意义|备注|
| :-- | :-- | :-- | :-- |
| 1 | system | The table has only one row (= system table). |system是const一种特例，表里只有一条元祖匹配|
| 2 | const | const is used when you compare all parts of a PRIMARY KEY or UNIQUE index to constant values. |主键查询或者唯一主键查询|
| 3 | eq_ref |It is used when all parts of an index are used by the join and the index is a PRIMARY KEY or UNIQUE NOT NULL index. eq_ref can be used for indexed columns that are compared using the = operator |主键、唯一索引关联查询|
| 4 | ref | ref is used if the join uses only a leftmost prefix of the key or if the key is not a PRIMARY KEY or UNIQUE index. ref can be used for indexed columns that are compared using the = or <=> operator|非主键、非唯一索引、唯一索引部分前缀|
| 5 | fulltext |The join is performed using a FULLTEXT index.||
| 6 | ref_or_null |This join type is like ref, but with the addition that MySQL does an extra search for rows that contain NULL values.||
| 7 | index_merge |This join type indicates that the Index Merge optimization is used||
| 8 | unique_subquery |unique_subquery is just an index lookup function that replaces the subquery completely for better efficiency.||
| 9 | index_subquery | it works for nonunique indexes in subqueries of the following form||
| 10 | range |range can be used when a key column is compared to a constant using any of the =, <>, >, >=, <, <=, IS NULL, <=>, BETWEEN, LIKE, or IN() operators:|范围查找|
| 11 | index |MySQL can use this join type when the query uses only columns that are part of a single index.|扫描二级索引就能拿到结果|
| 12 | ALL |A full table scan is done for each combination of rows from the previous tables|全表扫描，没有使用索引|

### null
下面值为null，这表示mysql在执行的时候是需要访问表和索引，直接能拿到结果
```sql
explain select min(id) from animals ;
```
![执行结果](/images/mysql-2-4.jpg)

### system & const
system是const一种特例，表里只有一条元祖匹配
const就是主键查询或者唯一主键查询
```sql
set session optimizer_switch='derived_merge=off';
explain select * from (select * from animals where id = 1) a;
set session optimizer_switch='derived_merge=on';
show warnings ;
```
```sql
# 优化结果
select '1' AS `id`,'朝阳动物园' AS `name`,'2022-03-03 11:15:59' AS `input_time` from dual
```
![执行结果](/images/mysql-2-5.jpg)

### eq_ref
主键、唯一索引关联查询
```sql
explain select * from zoo_animals left join zoo z on zoo_animals.zoo_id = z.id;
```
![执行结果](/images/mysql-2-6.jpg)

### ref
非主键、非唯一索引、唯一索引部分前缀
```sql
explain select * from zoo_animals where animals_id = 1;
```
![执行结果](/images/mysql-2-7.jpg)

### range
范围查找
```sql
explain select * from zoo_animals where id > 1;
```
![执行结果](/images/mysql-2-8.jpg)

### index
扫描二级索引就能拿到结果
```sql
explain select id,animals_id,zoo_id from zoo_animals;
```
![执行结果](/images/mysql-2-9.jpg)

### ALL
全表扫描，没有使用索引或者使用全表扫描更加合适
```sql
explain select id,animals_id,zoo_id,remark from zoo_animals;
```
![执行结果](/images/mysql-2-10.jpg)

## possible_keys
可能用到的主键key

## key
mysql实际采用那个索引来优化sql，如果没有使用索引，则该值为null

## key_len
这一类展示的是mysql索引里使用的字节数，通过这个值可以算出具体使用索引中的哪些字段
```sql
set session optimizer_switch='derived_merge=off';
explain select * from (select * from zoo_animals where animals_id = 1 and zoo_id = 1) a;
set session optimizer_switch='derived_merge=on';
show warnings ;
```
![执行结果](/images/mysql-2-5.jpg)

其中key_len = 4;这里计算规则是：

| 类型 | 大小(字节) | 备注 |
| :-- | :-- | :-- |
|char(n)|汉字就是3n|n是代表字符数，utf-8格式下，一个数字或字母占一个字节，一个汉字占3个字节，不同的编码格式下字符数是不一样的|
|varchar(n)|汉字就是3n+2|n是代表字符数，utf-8格式下，一个数字或字母占一个字节，一个汉字占3个字节，其中多的2个字节用来存储字符串长度，varchar变长，不同的编码格式下字符数是不一样的|
|tinyint|1||
|smallint|2||
|int|4||
|bigint|8||
|date|3||
|timestamp|4||
|datetime|8||
|字段允许为null|额外增加1字节||
索引最大值768字节

## ref
这个值显示key列使用的索引引用所使用的列或常量，const(常量)或列

```sql
explain select * from zoo_animals left join zoo z on zoo_animals.zoo_id = z.id;
```
![执行结果](/images/mysql-2-6.jpg)


## rows & filtered
rows：mysql估计要读取并检测的行数，不是实际扫描的行数
rows * filtered / 100 可以估算出与表关联连接的行数

## Extra
附加信息
详细看https://dev.mysql.com/doc/refman/5.7/en/explain-output.html#explain-extra-information

### Using index
使用覆盖索引，就会显示Using index，覆盖索引：key值存在索引，如果直接能从key值所在的B+tree能获取到所需要的信息，则就用到了覆盖索引

```sql
explain select (select name from zoo where name = '猴子') from zoo where id = 1;
```
![执行结果](/images/mysql-2-11.jpg)


### Using where
使用where语句来处理结果，没有使用覆盖索引

```sql
explain select * from zoo_animals where remark = '哈哈';
```
![执行结果](/images/mysql-2-12.jpg)

### Using index condition
查询的列不完全被索引覆盖
```sql
explain select * from zoo_animals where animals_id > 1 limit 2;
```
![执行结果](/images/mysql-2-13.jpg)

### Using filesort
order by 操作，不使用索引的方式

```sql
explain select * from animals order by name;
```
![执行结果](/images/mysql-2-25.png)



# 索引实践

## 数据准备
```sql
create table person
(
    id int auto_increment not null primary key comment '主键',
    name varchar(40) not null comment '姓名',
    age int not null comment '年纪',
    province varchar(20) not null comment '省份',
    gender int null comment '性别',
    card varchar(40) not null comment '身份证号',
    create_time datetime not null default current_timestamp comment '创建时间',
    update_time datetime null comment '更新时间',
    unique key unique_index_card(`card`),
    key index_name_province_gender(`name`,`province`,`gender`)
) engine = innodb auto_increment = 1 default charset = utf8;

INSERT INTO tahw.person (id, name, age, province, gender, card, create_time, update_time) VALUES (1, '张三', 18, '湖北省', 1, '202203041034', '2022-03-04 10:34:30', null);
INSERT INTO tahw.person (id, name, age, province, gender, card, create_time, update_time) VALUES (2, '李四', 20, '湖南省', 0, '202203051037', '2022-03-04 10:39:04', null);
INSERT INTO tahw.person (id, name, age, province, gender, card, create_time, update_time) VALUES (3, '王五', 13, '江西省', 0, '202103050437', '2022-03-04 10:39:04', null);
INSERT INTO tahw.person (id, name, age, province, gender, card, create_time, update_time) VALUES (4, '赵六', 15, '广东省', 1, '202507052037', '2022-03-04 10:39:04', null);
INSERT INTO tahw.person (id, name, age, province, gender, card, create_time, update_time) VALUES (5, '孙七', 55, '河北省', 0, '202210111232', '2022-03-04 10:39:04', null);
INSERT INTO tahw.person (id, name, age, province, gender, card, create_time, update_time) VALUES (6, '周八', 34, '天津省', 1, '201207240110', '2022-03-04 10:39:04', null);
INSERT INTO tahw.person (id, name, age, province, gender, card, create_time, update_time) VALUES (7, '吴九', 98, '辽宁省', 1, '201604328713', '2022-03-04 10:39:04', null);
INSERT INTO tahw.person (id, name, age, province, gender, card, create_time, update_time) VALUES (8, '郑十', 30, '山东省', 1, '201932332546', '2022-03-04 10:39:04', null);
```

## key_len计算
```sql
# key_len = 40 * 3 + 2
# ref列是const
explain select * from person where name = '张三';
```

![执行结果](/images/mysql-2-14.jpg)

```sql
# key_len = 40 * 3 + 2 + 20 * 3 + 2 + 4 + 1
# ref列是const,const,const
explain select * from person where name = '张三' and province = '湖北省' and gender = 1;
```
![执行结果](/images/mysql-2-15.jpg)

```sql
# 根据key_len来确定只使用了name索引字段
explain select * from person where name = '张三' and gender = 1;
```
![执行结果](/images/mysql-2-16.jpg)

## type区分

| sql | type | 解释 | 执行结果 |
| :-- | :-- | :-- | :-- |
|explain select * from person where <font color='red'><b>name > '张三'</b></font>;|ALL|根据索引最左侧原理，name也可以单独使用索引，possible_keys是使用name索引的，但是其实mysql根据表的结果，查询全部字段，数量不多，就不采用查询二级索引然后回查的方式，直接全表扫描效率更高|![执行结果](/images/mysql-2-17.jpg)|
|explain select * from person where <font color='red'><b>name != '张三'</b></font>;|ALL|根据索引最左侧原理，name也可以单独使用索引，possible_keys是使用name索引的，但是其实mysql根据表的结果，查询全部字段，数量不多，就不采用查询二级索引然后回查的方式，直接全表扫描效率更高|![执行结果](/images/mysql-2-17.jpg)|
|explain select * from person where <font color='red'><b>name <> '张三'</b></font>;|ALL|根据索引最左侧原理，name也可以单独使用索引，possible_keys是使用name索引的，但是其实mysql根据表的结果，查询全部字段，数量不多，就不采用查询二级索引然后回查的方式，直接全表扫描效率更高|![执行结果](/images/mysql-2-17.jpg)|
|explain select * from person where <font color='red'><b>name not in ('张三')</b></font>;|ALL|根据索引最左侧原理，name也可以单独使用索引，possible_keys是使用name索引的，但是其实mysql根据表的结果，查询全部字段，数量不多，就不采用查询二级索引然后回查的方式，直接全表扫描效率更高|![执行结果](/images/mysql-2-17.jpg)|
|explain select * from person where <font color='red'><b>name in ('张三')</b></font>;|ref|使用索引|![执行结果](/images/mysql-2-18.jpg)|
|explain select * from person where <font color='red'><b>name = '张三'</b></font>;|ref|使用索引|![执行结果](/images/mysql-2-18.jpg)|


下面两个sql执行前，先加一下索引`alter table person add index index_age(`age`) using btree ;`

| sql | type | 解释 | 执行结果 |
| :-- | :-- | :-- | :-- |
|explain select * from person where <font color='red'><b>age > 10</b></font>;|ALL|根据索引最左侧原理，name也可以单独使用索引，possible_keys是使用索引的，但是其实mysql根据表的结果，查询全部字段，数量不多，就不采用查询二级索引然后回查的方式，直接全表扫描效率更高，注意这里extra为Using where|![执行结果](/images/mysql-2-20.jpg)|
|explain select * from person where <font color='red'><b>age > 10 and age < 20</b></font>;|range|使用索引，为什么？其实这也是优化手段，分页查询会减少查询|![执行结果](/images/mysql-2-19.jpg)|



| sql | type | 解释 | 执行结果 |
| :-- | :-- | :-- | :-- |
|explain select <font color='red'><b>name,province</b></font> from person where substring(name,0,1) = '张';|Index|possible_keys识别出来没有使用索引，截取字符串函数是不能使用索引的，但是获取的是name索引相关的字段，二级索引是最优的方式，这里也就使用二级索引字段的索引，而不是name字段索引，key_len进一步来验证这一点|![执行结果](/images/mysql-2-21.jpg)|
|explain select <font color='red'><b>*</b></font> from person where substring(name,0,1) = '张';|ALL|没有使用索引，不识别函数，查主键索引最方便|![执行结果](/images/mysql-2-22.jpg)|



| sql | type | 解释 | 执行结果 |
| :-- | :-- | :-- | :-- |
|explain select * from person where name like <font color='red'><b>'%十'</b></font>;|ALL|前面百分号不识别，熟知B+Tree即可知道|![执行结果](/images/mysql-2-23.jpg)|
|explain select * from person where name like <font color='red'><b>'李%'</b></font>;|range|后面百分号可以使用索引，最左匹配原则|![执行结果](/images/mysql-2-24.jpg)|







# 附
官方地址：https://dev.mysql.com/doc/refman/5.7/en/explain-output.html

