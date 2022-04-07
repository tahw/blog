---
title: mysql之索引优化选择
date: 2022-03-10 10:13:21
tags:
    - mysql
categories:
    - mysql
---

# mysql之索引优化选择

## 数据准备
```sql
-- auto-generated definition
create table person
(
    id          int auto_increment comment '主键'
        primary key,
    name        varchar(40)                        not null comment '姓名',
    age         int                                not null comment '年纪',
    province    varchar(20)                        not null comment '省份',
    gender      int                                null comment '性别',
    card        varchar(40)                        not null comment '身份证号',
    create_time datetime default CURRENT_TIMESTAMP not null comment '创建时间',
    update_time datetime                           null comment '更新时间',
    constraint unique_index_card
        unique (card)
)
    charset = utf8;

create index index_age
    on person (age);

create index index_name_province_gender
    on person (name, province, gender);


```
通过存储过程的方式来生成测试数据，随便添加测试数据，尽可能让测试数据
```sql
drop procedure insert_emp;
delimiter ;
create procedure insert_emp()
 begin
     declare i int;
     set i=1;
     while(i<=10000) do
         insert into person(name,age,province,gender,card) values(CONCAT('吴九',i),i,'广东省',0,UUID());
         set i=i+1;
         end while ;
 end ;
delimiter ;
call insert_emp();
```
<!-- more -->
## 索引计算花销trace工具
```sql
explain select * from person where name like '李四%' and province = '河北省' and gender = 0;
explain select * from person where name like '张三%' and province = '广东省' and gender = 1;
```
同样的一张表，查询方式都是一样，只是查询的值不一样，选择的索引却是不一致的。对于这种情况，其实我们需要分析下索引选择

```sql
# 查看mysql trace配置
show variables like '%optimizer_trace%';
```
![执行结果](/images/mysql-4-10.jpg)


```sql
# 执行下面的sql，开启trace
SET session OPTIMIZER_TRACE="enabled=on";
SET session optimizer_trace_limit=5;
set session optimizer_trace_offset=5;
select * from person where name like '李四%' and province = '河北省' and gender = 0;
select * from information_schema.OPTIMIZER_TRACE;
# 恢复默认值
SET session OPTIMIZER_TRACE="enabled=off";
SET session optimizer_trace_limit=1;
set session optimizer_trace_offset=-1;
```
![执行结果](/images/mysql-4-11.jpg)
看到结果后，我们直接看TRACE字段
```json
{
    "steps":[
        {
            "join_preparation":{ # 第一阶段：sql准备阶段，格式化sql
                "select#":1,
                "steps":[
                    {
                        "expanded_query":"/* select#1 */ select `person`.`id` AS `id`,`person`.`name` AS `name`,`person`.`age` AS `age`,`person`.`province` AS `province`,`person`.`gender` AS `gender`,`person`.`card` AS `card`,`person`.`create_time` AS `create_time`,`person`.`update_time` AS `update_time` from `person` where ((`person`.`name` like '李四%') and (`person`.`province` = '河北省') and (`person`.`gender` = 0))"
                    }
                ]
            }
        },
        {
            "join_optimization":{ # 第二阶段：sql优化阶段
                "select#":1,
                "steps":[
                    {
                        "condition_processing":{ # 条件处理
                            "condition":"WHERE",
                            "original_condition":"((`person`.`name` like '李四%') and (`person`.`province` = '河北省') and (`person`.`gender` = 0))",
                            "steps":[
                                {
                                    "transformation":"equality_propagation",
                                    "resulting_condition":"((`person`.`name` like '李四%') and (`person`.`province` = '河北省') and multiple equal(0, `person`.`gender`))"
                                },
                                {
                                    "transformation":"constant_propagation",
                                    "resulting_condition":"((`person`.`name` like '李四%') and (`person`.`province` = '河北省') and multiple equal(0, `person`.`gender`))"
                                },
                                {
                                    "transformation":"trivial_condition_removal",
                                    "resulting_condition":"((`person`.`name` like '李四%') and (`person`.`province` = '河北省') and multiple equal(0, `person`.`gender`))"
                                }
                            ]
                        }
                    },
                    {
                        "substitute_generated_columns":{

                        }
                    },
                    {
                        "table_dependencies":[ # 表依赖结构
                            {
                                "table":"`person`",
                                "row_may_be_null":false,
                                "map_bit":0,
                                "depends_on_map_bits":[

                                ]
                            }
                        ]
                    },
                    {
                        "ref_optimizer_key_uses":[

                        ]
                    },
                    {
                        "rows_estimation":[ # 行估计
                            {
                                "table":"`person`",
                                "range_analysis":{
                                    "table_scan":{ # 表扫描情况
                                        "rows":1043609, # 扫描行数
                                        "cost":214907 # 花费成本
                                    },
                                    "potential_range_indexes":[ # 可能使用的索引
                                        {
                                            "index":"PRIMARY", # 主键索引
                                            "usable":false,
                                            "cause":"not_applicable"
                                        },
                                        {
                                            "index":"unique_index_card", # 唯一索引
                                            "usable":false,
                                            "cause":"not_applicable"
                                        },
                                        {
                                            "index":"index_name_province_gender",
                                            "usable":true, # 使用
                                            "key_parts":[
                                                "name",
                                                "province",
                                                "gender",
                                                "id"
                                            ]
                                        },
                                        {
                                            "index":"index_age",
                                            "usable":false,
                                            "cause":"not_applicable"
                                        }
                                    ],
                                    "setup_range_conditions":[

                                    ],
                                    "group_index_range":{
                                        "chosen":false,
                                        "cause":"not_group_by_or_distinct"
                                    },
                                    "analyzing_range_alternatives":{ # 分析各个索引使用成本
                                        "range_scan_alternatives":[
                                            {
                                                "index":"index_name_province_gender",
                                                "ranges":[
                                                    "李四\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000 <= name <= 李四￿￿￿￿￿￿￿￿￿￿￿￿￿￿￿￿￿￿￿￿￿￿￿￿￿￿￿￿￿￿￿￿￿￿￿￿￿￿"
                                                ],
                                                "index_dives_for_eq_ranges":true,
                                                "rowid_ordered":false, # 使用该索引获取的记录是否按照主键排序
                                                "using_mrr":false,
                                                "index_only":false, # 是否使用覆盖索引
                                                "rows":94112, # 扫描行数
                                                "cost":112935, # 索引使用成本
                                                "chosen":true # 是否选择该索引
                                            }
                                        ],
                                        "analyzing_roworder_intersect":{
                                            "usable":false,
                                            "cause":"too_few_roworder_scans"
                                        }
                                    },
                                    "chosen_range_access_summary":{ 
                                        "range_access_plan":{
                                            "type":"range_scan",
                                            "index":"index_name_province_gender",
                                            "rows":94112,
                                            "ranges":[
                                                "李四\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000 <= name <= 李四￿￿￿￿￿￿￿￿￿￿￿￿￿￿￿￿￿￿￿￿￿￿￿￿￿￿￿￿￿￿￿￿￿￿￿￿￿￿"
                                            ]
                                        },
                                        "rows_for_plan":94112,
                                        "cost_for_plan":112935,
                                        "chosen":true
                                    }
                                }
                            }
                        ]
                    },
                    {
                        "considered_execution_plans":[ # 执行计划
                            {
                                "plan_prefix":[

                                ],
                                "table":"`person`",
                                "best_access_path":{ # 最优访问路径
                                    "considered_access_paths":[ # 最终确定的访问路径
                                        {
                                            "rows_to_scan":94112,
                                            "access_type":"range", # 访问类型，range，范围查询
                                            "range_details":{
                                                "used_index":"index_name_province_gender" # 使用索引
                                            },
                                            "resulting_rows":941.12,
                                            "cost":131758,
                                            "chosen":true # 确定选择
                                        }
                                    ]
                                },
                                "condition_filtering_pct":100,
                                "rows_for_plan":941.12,
                                "cost_for_plan":131758,
                                "chosen":true
                            }
                        ]
                    },
                    {
                        "attaching_conditions_to_tables":{
                            "original_condition":"((`person`.`gender` = 0) and (`person`.`name` like '李四%') and (`person`.`province` = '河北省'))",
                            "attached_conditions_computation":[

                            ],
                            "attached_conditions_summary":[
                                {
                                    "table":"`person`",
                                    "attached":"((`person`.`gender` = 0) and (`person`.`name` like '李四%') and (`person`.`province` = '河北省'))"
                                }
                            ]
                        }
                    },
                    {
                        "refine_plan":[
                            {
                                "table":"`person`",
                                "pushed_index_condition":"((`person`.`gender` = 0) and (`person`.`name` like '李四%') and (`person`.`province` = '河北省'))",
                                "table_condition_attached":null
                            }
                        ]
                    }
                ]
            }
        },
        {
            "join_explain":{ # 执行阶段
                "select#":1,
                "steps":[

                ]
            }
        }
    ]
}
```


```sql
# 执行下面的sql，开启trace
SET session OPTIMIZER_TRACE="enabled=on";
SET session optimizer_trace_limit=5;
set session optimizer_trace_offset=5;
select * from person where name like '张三%' and province = '广东省' and gender = 1;
select * from information_schema.OPTIMIZER_TRACE;
# 恢复默认值
SET session OPTIMIZER_TRACE="enabled=off";
SET session optimizer_trace_limit=1;
set session optimizer_trace_offset=-1;
```
看到结果后，我们直接看TRACE字段
```json
{
    "steps":[
        {
            "join_preparation":{
                "select#":1,
                "steps":[
                    {
                        "expanded_query":"/* select#1 */ select `person`.`id` AS `id`,`person`.`name` AS `name`,`person`.`age` AS `age`,`person`.`province` AS `province`,`person`.`gender` AS `gender`,`person`.`card` AS `card`,`person`.`create_time` AS `create_time`,`person`.`update_time` AS `update_time` from `person` where ((`person`.`name` like '张三%') and (`person`.`province` = '广东省') and (`person`.`gender` = 1))"
                    }
                ]
            }
        },
        {
            "join_optimization":{
                "select#":1,
                "steps":[
                    {
                        "condition_processing":{
                            "condition":"WHERE",
                            "original_condition":"((`person`.`name` like '张三%') and (`person`.`province` = '广东省') and (`person`.`gender` = 1))",
                            "steps":[
                                {
                                    "transformation":"equality_propagation",
                                    "resulting_condition":"((`person`.`name` like '张三%') and (`person`.`province` = '广东省') and multiple equal(1, `person`.`gender`))"
                                },
                                {
                                    "transformation":"constant_propagation",
                                    "resulting_condition":"((`person`.`name` like '张三%') and (`person`.`province` = '广东省') and multiple equal(1, `person`.`gender`))"
                                },
                                {
                                    "transformation":"trivial_condition_removal",
                                    "resulting_condition":"((`person`.`name` like '张三%') and (`person`.`province` = '广东省') and multiple equal(1, `person`.`gender`))"
                                }
                            ]
                        }
                    },
                    {
                        "substitute_generated_columns":{

                        }
                    },
                    {
                        "table_dependencies":[
                            {
                                "table":"`person`",
                                "row_may_be_null":false,
                                "map_bit":0,
                                "depends_on_map_bits":[

                                ]
                            }
                        ]
                    },
                    {
                        "ref_optimizer_key_uses":[

                        ]
                    },
                    {
                        "rows_estimation":[
                            {
                                "table":"`person`",
                                "range_analysis":{
                                    "table_scan":{
                                        "rows":1043609,
                                        "cost":214907
                                    },
                                    "potential_range_indexes":[
                                        {
                                            "index":"PRIMARY",
                                            "usable":false,
                                            "cause":"not_applicable"
                                        },
                                        {
                                            "index":"unique_index_card",
                                            "usable":false,
                                            "cause":"not_applicable"
                                        },
                                        {
                                            "index":"index_name_province_gender",
                                            "usable":true,
                                            "key_parts":[
                                                "name",
                                                "province",
                                                "gender",
                                                "id"
                                            ]
                                        },
                                        {
                                            "index":"index_age",
                                            "usable":false,
                                            "cause":"not_applicable"
                                        }
                                    ],
                                    "setup_range_conditions":[

                                    ],
                                    "group_index_range":{
                                        "chosen":false,
                                        "cause":"not_group_by_or_distinct"
                                    },
                                    "analyzing_range_alternatives":{
                                        "range_scan_alternatives":[
                                            {
                                                "index":"index_name_province_gender",
                                                "ranges":[
                                                    "张三\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000 <= name <= 张三￿￿￿￿￿￿￿￿￿￿￿￿￿￿￿￿￿￿￿￿￿￿￿￿￿￿￿￿￿￿￿￿￿￿￿￿￿￿"
                                                ],
                                                "index_dives_for_eq_ranges":true,
                                                "rowid_ordered":false,
                                                "using_mrr":false,
                                                "index_only":false,
                                                "rows":521804,
                                                "cost":626166,
                                                "chosen":false,
                                                "cause":"cost"
                                            }
                                        ],
                                        "analyzing_roworder_intersect":{
                                            "usable":false,
                                            "cause":"too_few_roworder_scans"
                                        }
                                    }
                                }
                            }
                        ]
                    },
                    {
                        "considered_execution_plans":[ # 考虑执行计划
                            {
                                "plan_prefix":[

                                ],
                                "table":"`person`",
                                "best_access_path":{
                                    "considered_access_paths":[
                                        {
                                            "rows_to_scan":1043609,
                                            "access_type":"scan", # 全表扫描
                                            "resulting_rows":5218,
                                            "cost":214905, # 这里的cont 214905花销比index_name_province_gender 626166少，直接使用全表扫描
                                            "chosen":true # 确定
                                        }
                                    ]
                                },
                                "condition_filtering_pct":100,
                                "rows_for_plan":5218,
                                "cost_for_plan":214905,
                                "chosen":true
                            }
                        ]
                    },
                    {
                        "attaching_conditions_to_tables":{
                            "original_condition":"((`person`.`gender` = 1) and (`person`.`name` like '张三%') and (`person`.`province` = '广东省'))",
                            "attached_conditions_computation":[

                            ],
                            "attached_conditions_summary":[
                                {
                                    "table":"`person`",
                                    "attached":"((`person`.`gender` = 1) and (`person`.`name` like '张三%') and (`person`.`province` = '广东省'))"
                                }
                            ]
                        }
                    },
                    {
                        "refine_plan":[
                            {
                                "table":"`person`"
                            }
                        ]
                    }
                ]
            }
        },
        {
            "join_explain":{
                "select#":1,
                "steps":[

                ]
            }
        }
    ]
}
```
> 不同数据集导致的花销是不一样的，所选择的索引也是不同的。通过trace工具我们可以很好的分析使用索引的花销

## 索引选择

### 前言
```sql
explain select * from person where name = '张三1' and province = '广东省' and gender = 1;
```
![执行结果](/images/mysql-4-1.jpg)

<font color='red'><b>很明显，这个sql使用了联合索引里面的三个字段，explain里面的ref也是const,const,const也能说明。那我们在优化sql的时候，索引长度也是能确定用到联合索引里面的哪些字段的。
name索引长度为122，使用name、province索引长度为184，使用name、province、gender索引长度为189</b></font>

### > <范围查找
范围查找使用联合索引第一个字段，是不会走索引的
```sql
explain select * from person where name > '张' and province = '广东省' and gender = 0;
```
![执行结果](/images/mysql-4-2.jpg)

```sql
# 采用覆盖索引的方式来使用索引
explain select name, province, gender from person where name > '张' and province = '广东省' and gender = 0;
```
![执行结果](/images/mysql-4-3.jpg)

1. 我们使用的索引只有name字段，其他字段是没有使用到的，详细可以结合联合索引树来理解
2. 对比上面两个结果，possible_keys是有值的。<font color='red'><b>但是第一个真正执行的时候是没有使用索引的，而第二个执行的时候是使用索引的。这是为什么？当联合索引的第一个字段采用范围查找的时候不会走索引，mysql内部会觉得第一个字段得到的结果集范围很大，回表效率不高，直接走聚簇索引。而第二个直接在二级索引就能得到结果，还是会走索引。其实第二个就是对第一个的优化，采用覆盖索引的方式</b></font>

#### 强制索引
```sql
explain select * from person force index(index_name_province_gender) where name > '张' and province = '广东省' and gender = 1;
```
![执行结果](/images/mysql-4-4.jpg)

> 上面使用force index方式强制使用索引，但是注意使用强制索引不一定是优化，我们看下执行结果花费时间，见下图，基本上mysql选择索引内部已经非常精准了，基本上我们是不需要强制使用索引的

为了对比结果更精准，需要关闭查询缓存query_cache
```sql
# 关闭query_cache
> set global query_cache_type  = 0;
> set global query_cache_size  = 0;
> select * from person where name > '张' and province = '广东省' and gender = 1;
[2022-03-13 12:01:48] 500 rows retrieved starting from 1 in 83 ms (execution: 14 ms, fetching: 69 ms)

# 关闭query_cache
> set global query_cache_type  = 0;
> set global query_cache_size  = 0;
> select * from person force index(index_name_province_gender) where name > '张' and province = '广东省' and gender = 1;
[2022-03-13 12:01:59] 500 rows retrieved starting from 1 in 111 ms (execution: 17 ms, fetching: 94 ms)
```

### in、or 查询
in、or在表数据量大的情况下，使用索引，在数据量小的情况下，会走聚簇索引，不用回表，效率高。
```sql
# 使用复合索引全量字段
explain select * from person where name in ('赵六','张三265') and province = '广东省' and gender = 1;
```
![执行结果](/images/mysql-4-5.jpg)


```sql
# person_small是和person一样的结构，但是数据只有三条
explain select * from person_small where name in ('赵六','张三265') and province = '广东省' and gender = 1;
```
![执行结果](/images/mysql-4-6.jpg)

### like 查询
like查询，如果是后%的方式，一般都走索引

```sql
explain select * from person where name like '李四%' and province = '河北省' and gender = 0;
```
![执行结果](/images/mysql-4-7.jpg)
这里可以分析下，其中这个sql是会走索引的，<font color='red'><b>但是走的应该只有name索引，结合索引树，province和gender是没办法走索引的，但是我们发现结果是走了name、province、gender的。这是什么原因，其实这跟like里面索引下推有关系</b></font>

#### 索引下推
流程图
![索引下推](/images/mysql-4-8.png)
<font color='red'><b>索引下推只在like查询的时候才会使用，其他都不会使用，准确来说，其实是对数据集小的来使用</b></font>
1. 在mysql5.6之前，根据name直接查询二级索引树后得到主键id，然后拿到id后再去聚簇索引里面拿到对应结果，然后拿到province、gender字段结果比较，然后得到数据集。
2. <font color='red'><b>在mysql5.6之后，根据name拿到结果后，正好二级索引树上面有其他字段，如果where条件里面有字段查询的，直接比较，然后拿到id，然后回表聚簇索引，这样回表的数据集肯定更加小的。效率更高。这个就是索引下推。</b></font>

```sql
explain select * from person where name like '张三%' and province = '广东省' and gender = 1;
```
![执行结果](/images/mysql-4-9.jpg)
这个直接就不走索引了，直接采用聚簇索引扫描，为什么？看下rows，量太大，更不会走索引下推。

> 那这里会有疑问？范围查找<、>怎么不使用索引下推？这个就跟数据集有关系，我们一般认为范围查找是数据结果集大，like查询数据集小才会使用索引下推。like数据集大也不会走索引下推。

### 查询结合order by排序使用
索引的选择先是查询，然后再根据order by选择索引

#### sort buffer
提起order by，一定会有sort buffer，这个是把对应数据拿过来后在这里进行排序，这个大小只有1M
```sql
show variables like '%innodb_sort_buffer_size%';
```
![执行结果](/images/mysql-4-11-1.jpg)


#### 例子1
```sql
explain select * from person where name = '张三' and gender = 1 order by province;
```
![执行结果](/images/mysql-4-12.jpg)
根据上面的结果，使用的name索引，由于最左匹配原则，gender是没有使用的。排序的时候由于Extra是Using index condition。这里最终使用索引是name、province

#### 例子2
```sql
explain select * from person where name = '张三' order by province,gender;
```
![执行结果](/images/mysql-4-13.jpg)

```sql
explain select * from person where name = '张三' order by gender,province;
```
![执行结果](/images/mysql-4-14.jpg)

```sql
explain select * from person where name = '张三' order by province asc,gender desc;
```
![执行结果](/images/mysql-4-14.jpg)

> 都使用了name索引，但是下面的Extra是Using filesort，没有使用其他索引，在排序的时候也是有顺序的

#### 例子3
```sql
explain select * from person where name > '张三' order by name;
```
![执行结果](/images/mysql-4-15.jpg)

```sql
explain select name,province,gender from person where name > '张三' order by name;
```
![执行结果](/images/mysql-4-16.jpg)

> 数据量太大，没有使用name索引，采用覆盖索引的方式来优化

#### Using filesort
<font color='red'><b>文件排序里面分为单路排序和多路排序，其中是通过系统变量`max_length_for_sort_data`(1024字节)来决定的。当需要查询的字段总长度小于`max_length_for_sort_data`就是用单路排序，否则使用多路排序</b></font>

##### 单路排序
<font color='red'><b>一次性取出满足行的所有字段，然后再sort buffer里面排序。在trace里面表现的是filesort_summary下面有个sort_mode。
表现方式是：<sort_key, additional_fields> 或者 <sort_key, packed_additional_fields></b></font>

```sql
# mysql 查看trace值
SET session OPTIMIZER_TRACE="enabled=on",end_markers_in_json=on;;
SET session optimizer_trace_limit=5;
set session optimizer_trace_offset=5;
# 这里不能加explain
select * from person where name = '张三' order by gender;
SELECT * FROM INFORMATION_SCHEMA.OPTIMIZER_TRACE limit 30 ;
SET session OPTIMIZER_TRACE="enabled=off",end_markers_in_json=off;
SET session optimizer_trace_limit=1;
set session optimizer_trace_offset=-1;
```
```json
{
  "steps": [
    {
      "join_preparation": {
        "select#": 1,
        "steps": [
          {
            "expanded_query": "/* select#1 */ select `person`.`id` AS `id`,`person`.`name` AS `name`,`person`.`age` AS `age`,`person`.`province` AS `province`,`person`.`gender` AS `gender`,`person`.`card` AS `card`,`person`.`create_time` AS `create_time`,`person`.`update_time` AS `update_time` from `person` where (`person`.`name` = '张三') order by `person`.`gender`"
          }
        ] /* steps */
      } /* join_preparation */
    },
    {
      "join_optimization": {
        "select#": 1,
        "steps": [
          {
            "condition_processing": {
              "condition": "WHERE",
              "original_condition": "(`person`.`name` = '张三')",
              "steps": [
                {
                  "transformation": "equality_propagation",
                  "resulting_condition": "(`person`.`name` = '张三')"
                },
                {
                  "transformation": "constant_propagation",
                  "resulting_condition": "(`person`.`name` = '张三')"
                },
                {
                  "transformation": "trivial_condition_removal",
                  "resulting_condition": "(`person`.`name` = '张三')"
                }
              ] /* steps */
            } /* condition_processing */
          },
          {
            "substitute_generated_columns": {
            } /* substitute_generated_columns */
          },
          {
            "table_dependencies": [
              {
                "table": "`person`",
                "row_may_be_null": false,
                "map_bit": 0,
                "depends_on_map_bits": [
                ] /* depends_on_map_bits */
              }
            ] /* table_dependencies */
          },
          {
            "ref_optimizer_key_uses": [
              {
                "table": "`person`",
                "field": "name",
                "equals": "'张三'",
                "null_rejecting": false
              }
            ] /* ref_optimizer_key_uses */
          },
          {
            "rows_estimation": [
              {
                "table": "`person`",
                "range_analysis": {
                  "table_scan": {
                    "rows": 1043609,
                    "cost": 214907
                  } /* table_scan */,
                  "potential_range_indexes": [
                    {
                      "index": "PRIMARY",
                      "usable": false,
                      "cause": "not_applicable"
                    },
                    {
                      "index": "unique_index_card",
                      "usable": false,
                      "cause": "not_applicable"
                    },
                    {
                      "index": "index_name_province_gender",
                      "usable": true,
                      "key_parts": [
                        "name",
                        "province",
                        "gender",
                        "id"
                      ] /* key_parts */
                    },
                    {
                      "index": "index_age",
                      "usable": false,
                      "cause": "not_applicable"
                    }
                  ] /* potential_range_indexes */,
                  "setup_range_conditions": [
                  ] /* setup_range_conditions */,
                  "group_index_range": {
                    "chosen": false,
                    "cause": "not_group_by_or_distinct"
                  } /* group_index_range */,
                  "analyzing_range_alternatives": {
                    "range_scan_alternatives": [
                      {
                        "index": "index_name_province_gender",
                        "ranges": [
                          "张三 <= name <= 张三"
                        ] /* ranges */,
                        "index_dives_for_eq_ranges": true,
                        "rowid_ordered": false,
                        "using_mrr": false,
                        "index_only": false,
                        "rows": 1,
                        "cost": 2.21,
                        "chosen": true
                      }
                    ] /* range_scan_alternatives */,
                    "analyzing_roworder_intersect": {
                      "usable": false,
                      "cause": "too_few_roworder_scans"
                    } /* analyzing_roworder_intersect */
                  } /* analyzing_range_alternatives */,
                  "chosen_range_access_summary": {
                    "range_access_plan": {
                      "type": "range_scan",
                      "index": "index_name_province_gender",
                      "rows": 1,
                      "ranges": [
                        "张三 <= name <= 张三"
                      ] /* ranges */
                    } /* range_access_plan */,
                    "rows_for_plan": 1,
                    "cost_for_plan": 2.21,
                    "chosen": true
                  } /* chosen_range_access_summary */
                } /* range_analysis */
              }
            ] /* rows_estimation */
          },
          {
            "considered_execution_plans": [
              {
                "plan_prefix": [
                ] /* plan_prefix */,
                "table": "`person`",
                "best_access_path": {
                  "considered_access_paths": [
                    {
                      "access_type": "ref",
                      "index": "index_name_province_gender",
                      "rows": 1,
                      "cost": 1.2,
                      "chosen": true
                    },
                    {
                      "access_type": "range",
                      "range_details": {
                        "used_index": "index_name_province_gender"
                      } /* range_details */,
                      "chosen": false,
                      "cause": "heuristic_index_cheaper"
                    }
                  ] /* considered_access_paths */
                } /* best_access_path */,
                "condition_filtering_pct": 100,
                "rows_for_plan": 1,
                "cost_for_plan": 1.2,
                "chosen": true
              }
            ] /* considered_execution_plans */
          },
          {
            "attaching_conditions_to_tables": {
              "original_condition": "(`person`.`name` = '张三')",
              "attached_conditions_computation": [
              ] /* attached_conditions_computation */,
              "attached_conditions_summary": [
                {
                  "table": "`person`",
                  "attached": null
                }
              ] /* attached_conditions_summary */
            } /* attaching_conditions_to_tables */
          },
          {
            "clause_processing": {
              "clause": "ORDER BY",
              "original_clause": "`person`.`gender`",
              "items": [
                {
                  "item": "`person`.`gender`"
                }
              ] /* items */,
              "resulting_clause_is_simple": true,
              "resulting_clause": "`person`.`gender`"
            } /* clause_processing */
          },
          {
            "added_back_ref_condition": "((`person`.`name` <=> '张三'))"
          },
          {
            "reconsidering_access_paths_for_index_ordering": {
              "clause": "ORDER BY",
              "steps": [
              ] /* steps */,
              "index_order_summary": {
                "table": "`person`",
                "index_provides_order": false,
                "order_direction": "undefined",
                "index": "index_name_province_gender",
                "plan_changed": false
              } /* index_order_summary */
            } /* reconsidering_access_paths_for_index_ordering */
          },
          {
            "refine_plan": [
              {
                "table": "`person`",
                "pushed_index_condition": "(`person`.`name` <=> '张三')",
                "table_condition_attached": null
              }
            ] /* refine_plan */
          }
        ] /* steps */
      } /* join_optimization */
    },
    {
      "join_execution": { # sql执行阶段
        "select#": 1,
        "steps": [
          {
            "filesort_information": [ # 文件排序信息
              {
                "direction": "asc",
                "table": "`person`",
                "field": "gender"
              }
            ] /* filesort_information */,
            "filesort_priority_queue_optimization": {
              "limit": 501,
              "rows_estimate": 4573370,
              "row_size": 331,
              "memory_available": 262144,
              "chosen": true
            } /* filesort_priority_queue_optimization */,
            "filesort_execution": [
            ] /* filesort_execution */,
            "filesort_summary": { # 文件排序信息
              "rows": 1,            # 预计扫描行数
              "examined_rows": 1,   # 参与排序行
              "number_of_tmp_files": 0, # 使用临时文件个数，这个值为0代表全部使用sort_buffer排序，否则使用的是磁盘文件排序
              "sort_buffer_size": 170184, # 排序缓存大小，单位Byte
              "sort_mode": "<sort_key, additional_fields>" # 排序方式，单路排序
            } /* filesort_summary */
          }
        ] /* steps */
      } /* join_execution */
    }
  ] /* steps */
}
```

##### 多路排序
<font color='red'><b>也叫做回表排序，首先根据查询条件取出排序字段和可以直接定位的行的主键id，然后再sort buffer里面排序，排序完后再次回表取出其他需要的字段
在trace里面表现的是filesort_summary下面有个sort_mode。
表现方式是： <sort_key, rowid></b></font>

```sql
# 调整sort buffer大小，可以直接使用双路排序
set max_length_for_sort_data = 10;
SET session OPTIMIZER_TRACE="enabled=on",end_markers_in_json=on;;
SET session optimizer_trace_limit=5;
set session optimizer_trace_offset=5;
# 这里不能加explain
select * from person where name = '张三' order by gender;
SELECT * FROM INFORMATION_SCHEMA.OPTIMIZER_TRACE limit 30 ;
SET session OPTIMIZER_TRACE="enabled=off",end_markers_in_json=off;
SET session optimizer_trace_limit=1;
set session optimizer_trace_offset=-1;
set max_length_for_sort_data = 1024
```
```json
{
  "steps": [
    {
      "join_preparation": {
        "select#": 1,
        "steps": [
          {
            "expanded_query": "/* select#1 */ select `person`.`id` AS `id`,`person`.`name` AS `name`,`person`.`age` AS `age`,`person`.`province` AS `province`,`person`.`gender` AS `gender`,`person`.`card` AS `card`,`person`.`create_time` AS `create_time`,`person`.`update_time` AS `update_time` from `person` where (`person`.`name` = '张三') order by `person`.`gender`"
          }
        ] /* steps */
      } /* join_preparation */
    },
    {
      "join_optimization": {
        "select#": 1,
        "steps": [
          {
            "condition_processing": {
              "condition": "WHERE",
              "original_condition": "(`person`.`name` = '张三')",
              "steps": [
                {
                  "transformation": "equality_propagation",
                  "resulting_condition": "(`person`.`name` = '张三')"
                },
                {
                  "transformation": "constant_propagation",
                  "resulting_condition": "(`person`.`name` = '张三')"
                },
                {
                  "transformation": "trivial_condition_removal",
                  "resulting_condition": "(`person`.`name` = '张三')"
                }
              ] /* steps */
            } /* condition_processing */
          },
          {
            "substitute_generated_columns": {
            } /* substitute_generated_columns */
          },
          {
            "table_dependencies": [
              {
                "table": "`person`",
                "row_may_be_null": false,
                "map_bit": 0,
                "depends_on_map_bits": [
                ] /* depends_on_map_bits */
              }
            ] /* table_dependencies */
          },
          {
            "ref_optimizer_key_uses": [
              {
                "table": "`person`",
                "field": "name",
                "equals": "'张三'",
                "null_rejecting": false
              }
            ] /* ref_optimizer_key_uses */
          },
          {
            "rows_estimation": [
              {
                "table": "`person`",
                "range_analysis": {
                  "table_scan": {
                    "rows": 1043609,
                    "cost": 214907
                  } /* table_scan */,
                  "potential_range_indexes": [
                    {
                      "index": "PRIMARY",
                      "usable": false,
                      "cause": "not_applicable"
                    },
                    {
                      "index": "unique_index_card",
                      "usable": false,
                      "cause": "not_applicable"
                    },
                    {
                      "index": "index_name_province_gender",
                      "usable": true,
                      "key_parts": [
                        "name",
                        "province",
                        "gender",
                        "id"
                      ] /* key_parts */
                    },
                    {
                      "index": "index_age",
                      "usable": false,
                      "cause": "not_applicable"
                    }
                  ] /* potential_range_indexes */,
                  "setup_range_conditions": [
                  ] /* setup_range_conditions */,
                  "group_index_range": {
                    "chosen": false,
                    "cause": "not_group_by_or_distinct"
                  } /* group_index_range */,
                  "analyzing_range_alternatives": {
                    "range_scan_alternatives": [
                      {
                        "index": "index_name_province_gender",
                        "ranges": [
                          "张三 <= name <= 张三"
                        ] /* ranges */,
                        "index_dives_for_eq_ranges": true,
                        "rowid_ordered": false,
                        "using_mrr": false,
                        "index_only": false,
                        "rows": 1,
                        "cost": 2.21,
                        "chosen": true
                      }
                    ] /* range_scan_alternatives */,
                    "analyzing_roworder_intersect": {
                      "usable": false,
                      "cause": "too_few_roworder_scans"
                    } /* analyzing_roworder_intersect */
                  } /* analyzing_range_alternatives */,
                  "chosen_range_access_summary": {
                    "range_access_plan": {
                      "type": "range_scan",
                      "index": "index_name_province_gender",
                      "rows": 1,
                      "ranges": [
                        "张三 <= name <= 张三"
                      ] /* ranges */
                    } /* range_access_plan */,
                    "rows_for_plan": 1,
                    "cost_for_plan": 2.21,
                    "chosen": true
                  } /* chosen_range_access_summary */
                } /* range_analysis */
              }
            ] /* rows_estimation */
          },
          {
            "considered_execution_plans": [
              {
                "plan_prefix": [
                ] /* plan_prefix */,
                "table": "`person`",
                "best_access_path": {
                  "considered_access_paths": [
                    {
                      "access_type": "ref",
                      "index": "index_name_province_gender",
                      "rows": 1,
                      "cost": 1.2,
                      "chosen": true
                    },
                    {
                      "access_type": "range",
                      "range_details": {
                        "used_index": "index_name_province_gender"
                      } /* range_details */,
                      "chosen": false,
                      "cause": "heuristic_index_cheaper"
                    }
                  ] /* considered_access_paths */
                } /* best_access_path */,
                "condition_filtering_pct": 100,
                "rows_for_plan": 1,
                "cost_for_plan": 1.2,
                "chosen": true
              }
            ] /* considered_execution_plans */
          },
          {
            "attaching_conditions_to_tables": {
              "original_condition": "(`person`.`name` = '张三')",
              "attached_conditions_computation": [
              ] /* attached_conditions_computation */,
              "attached_conditions_summary": [
                {
                  "table": "`person`",
                  "attached": null
                }
              ] /* attached_conditions_summary */
            } /* attaching_conditions_to_tables */
          },
          {
            "clause_processing": {
              "clause": "ORDER BY",
              "original_clause": "`person`.`gender`",
              "items": [
                {
                  "item": "`person`.`gender`"
                }
              ] /* items */,
              "resulting_clause_is_simple": true,
              "resulting_clause": "`person`.`gender`"
            } /* clause_processing */
          },
          {
            "added_back_ref_condition": "((`person`.`name` <=> '张三'))"
          },
          {
            "reconsidering_access_paths_for_index_ordering": {
              "clause": "ORDER BY",
              "steps": [
              ] /* steps */,
              "index_order_summary": {
                "table": "`person`",
                "index_provides_order": false,
                "order_direction": "undefined",
                "index": "index_name_province_gender",
                "plan_changed": false
              } /* index_order_summary */
            } /* reconsidering_access_paths_for_index_ordering */
          },
          {
            "refine_plan": [
              {
                "table": "`person`",
                "pushed_index_condition": "(`person`.`name` <=> '张三')",
                "table_condition_attached": null
              }
            ] /* refine_plan */
          }
        ] /* steps */
      } /* join_optimization */
    },
    {
      "join_execution": {
        "select#": 1,
        "steps": [
          {
            "filesort_information": [
              {
                "direction": "asc",
                "table": "`person`",
                "field": "gender"
              }
            ] /* filesort_information */,
            "filesort_priority_queue_optimization": {
              "limit": 501,
              "rows_estimate": 4573370,
              "row_size": 9,
              "memory_available": 262144,
              "chosen": true
            } /* filesort_priority_queue_optimization */,
            "filesort_execution": [
            ] /* filesort_execution */,
            "filesort_summary": {
              "rows": 1,
              "examined_rows": 1,
              "number_of_tmp_files": 0,
              "sort_buffer_size": 8536,
              "sort_mode": "<sort_key, rowid>" # 双路排序
            } /* filesort_summary */
          }
        ] /* steps */
      } /* join_execution */
    }
  ] /* steps */
}
```

##### 单路排序和多路排序对比

| sql | 单路排序执行流程 | 多路排序执行流程 |
| :-- | :---     | :---    |
|select * from person where name = '张三' order by gender;|1. 从索引name找到第一个name='张三'条件的主键<br><font color='red'><b>2. 然后根据聚簇索引查找所有字段放入sort_buffer里面<b></font><br>3. 重复1.2操作，直到找不到name='张三'<br>4. 对sort_buffer里面数据按照gender排序<br>5. 结果集返回客户端|1. 从索引name找到第一个name='张三'条件的主键<br><font color='red'><b>2. 然后根据聚簇索引后，拿出gender和主键id放入sort_buffer里面<b></font><br>3. 重复1.2操作，直到找不到name='张三'<br>4. 对sort_buffer里面数据按照gender排序<br><font color='red'><b>5. 拿出排序好的id，然后回表聚簇索引，拿出所有的数据集返回客户端<b></font>|

> 如果sort buffer配置的比较小，那我们其实可以适当把`max_length_for_sort_data`大小调整小一点，尽量使用双路排序算法，sort buffer可以排序更多的行，只是在回表再查一下需要的字段，这样的效率会更高。反之也是这样。

### limit分页
```sql
select * from person limit 100000,10;
```
<font color='red'><b>查出的逻辑是：先从person取出100010行数据。然后抛去100000行数据，然后取出后面10行。如果表足够大，执行效率非常低。</b></font>
![执行结果](/images/mysql-4-17.jpg)


```sql
select id from person limit 100000,10;
```
![执行结果](/images/mysql-4-18.jpg)
看下结果，只是查询字段id，得到的结果却和上面不一致？为什么？<font color='red'><b>看下下面的执行计划，使用的索引，使用的索引是不一致的。上面是使用全表扫描，下面是使用age索引，最简单二级索引，二级索引里面是存在主键id字段。所以拿到的结果是不一致的。</b></font>

| sql | 结果|
| :-- | :-- |
| explain select * from person limit 100000,10;|![执行结果](/images/mysql-4-19.jpg)|
| explain select id from person limit 100000,10;|![执行结果](/images/mysql-4-20.jpg)|

#### 主键分页
```sql
select * from person limit 100000,10;
# => 可以转换成下面这样
select * from person where id > 100000 limit 10;
```
<font color='red'><b>主键排序虽然可以优化成这样，但是这种场景只使用主键自增场景，如果删除一条数据，就会导致结果不一致</b></font>

#### 非主键分页
```sql
explain select * from person order by name limit 100000,10;
```
![执行结果](/images/mysql-4-21.jpg)
查询大概花费1s，使用filesort，没有使用索引

```sql
explain select * from person p inner join (select id from person order by name limit 100000,10) b on b.id = p.id;
```
![执行结果](/images/mysql-4-22.jpg)
转换成使用覆盖索引，使用Using Index，然后primary，然后查询<derived2>全部，这里其实只有10条，是非常快的。看到的rows是其实是估算的，没有执行过估算数据。

### join关联
表关联常见有两种算法：Nested-Loop Join算法、Block Nested-Loop Join算法。那关联中一定会有join buffer的影响、驱动表的区分。

#### join buffer
提起join，一定会有join buffer，这个是把对应数据拿过来互相关联得到结果，这个大小只有256k
```sql
show variables like '%join_buffer_size%';
```
![执行结果](/images/mysql-4-23.jpg)

#### 驱动表和被驱动表
这个会影响关联的效率和算法
1. 在两表关联的时候，优化器会将小表做驱动表，大表为被驱动表。
2. 当使用left join时，左表是驱动表，右表是被驱动表
3. 当使用right join时，右表是驱动表，左表是被驱动表
4. inner join时，排在前面的表不一定是驱动表，优化器会计算数量级小为驱动表
5. select * from a straight_join b on b.id  = a.id。`straight_join`强制让a做驱动表，`straight_join`只适应于inner join
6. <font color='red'><b>优化器判断数量级的时候是增加where条件的，最终扫描的行数。</b></font>
7. 尽可能让优化器自己选择驱动表，大部分mysql自己优化是最优的


#### Nested-Loop Join算法（嵌套循环算法NLJ）
```sql
explain select * from person_small inner join person p on person_small.age = p.age;
```
![执行结果](/images/mysql-4-26.jpg)

1. 通过age来关联，person_small全表扫描，因为只有3条数据，而person会走索引。驱动表是先执行的person_small，而被驱动表是person。
2. 关联中Extra没有出现Using join buffer (Block Nested Loop)，则就是使用NLJ

![NLJ流程图](/images/mysql-4-24.png)

#### Block Nested-Loop Join算法（基于块的嵌套循环算法BNL）
```sql
explain select * from person_small inner join person p on person_small.province = p.province;
```
![执行结果](/images/mysql-4-27.jpg)

![BNL流程图](/images/mysql-4-25.png)

#### NLJ 和 BNL 对比
person_small 是100行数据，person是10000行数据
```sql
# NLJ
explain select * from person_small inner join person p on person_small.age = p.age;
# BNL
explain select * from person_small inner join person p on person_small.province = p.province;
```
| 指标 | NLJ | BNL |
| :-- | :-- | :-- |
|执行过程|1. 首先区分person_small为驱动表<br> 2. <font color='red'><b>然后从person_small（如果有where条件，是先会筛选的）里面读取一行数据到join buffer里面</b></font><br> 3. 取出age字段，然后到person里面查找<br> 4. 拿到匹配的行合并到数据集<br> 5. 重复2、3、4操作最终得到数据集返回客户端 | 1.首先区分person_small为驱动表<br> 2. <font color='red'><b>然后从person_small所有的数据到join buffer里面（如果join buffer不够，则会分段放）</b></font><br> 3. 把person的每一行拿出来和join buffer里面的数据比较<br> 4. 拿出结果集返回给客户端 |
|扫描行数| <font color='red'><b>person_small扫描了全表100行，person走的索引也是100行，总共200行</b></font> | 1. <font color='red'><b>person_small扫描了全表100行，person没有走索引也是全表扫描10000行，总共10100行</b></font><br> 2. <font color='red'><b>如果join buffer不够的话，只能存50行person_small，比较完50行的person_small后就会清空join buffer，然后扫描接下来的50条，重新扫描person</b></font>，对应的会扫描100行的person_small+两次的person10000行，就是20100行 |
|判断次数| 一行一行比较就是100次 | 100 * 10000次 |

#### 如果BNL算法采用NLJ算法关联效率
拿person_small 是100行数据，person是10000行数据距离，扫描的行数会变成100 * 10000。这个扫描其实是磁盘的操作，肯定没有内存计算的100 * 10000快。

#### 优化方式
1. 关联字段加索引，尽量使用NLJ算法
2. 小表驱动大表

### in、exist
也是小表驱动大表

在person里面找出和person_small相同age的记录

```sql
# in 方式
select * from person where age in (select age from person_small);
```
```sql
# exists 方式
select * from person_small where exists (select age from person);
```
### count(*)
| sql | 直接结果 |
| :-- | :-- |
| explain select count(*) from person; |![执行结果](/images/mysql-4-28.jpg)|
| explain select count(1) from person;|![执行结果](/images/mysql-4-28.jpg)|
| explain select count(age) from person; |![执行结果](/images/mysql-4-28.jpg)|
| explain select count(id) from person; |![执行结果](/images/mysql-4-28.jpg)|
| explain select count(create_time) from person;|![执行结果](/images/mysql-4-29.jpg)|

1. <font color='red'><b>除了最后一个没有走索引，其他都是走相同的索引，前面都是效率差不多的</b></font>
2. <font color='red'><b>只有根据某个字段count时，不会算null值的行，其他都算</b></font>

我们理解上面四个都是走一样的索引，效率其实是差不多的，但是如果非要比较的话，还是有点差别。比较后的性能其实是
<font color='red'><b>count(*) ≈ count(1) > count(二级索引字段) > count(主键id) > count(非二级索引字段)</b></font>
1. count(1)是不需要取出字段，直接用1来统计，扫描一行就直接内存+1
2. <font color='red'><b>count(*)是比较例外的，mysql并不会把所有字段取出来，专门做了优化，不取字段值，直接累加</b></font>
3. count(二级索引字段)是二级索引树，肯定比主键树轻量级，肯定count(二级索引字段) 比 count(主键id) 快，<font color='red'><b>这里虽然count(主键id)也是走的二级索引，但是主键id是叶子节点，而二级索引字段都不需要取节点值，直接节点字段。肯定也是快的。</b></font>


## 设计规则
1. 代码先行，索引后上
2. 联合索引尽量覆盖索引
    1. create index(name,provice,gender,age)
    2. select * from persion where name = '张三' and provice = '湖北省' and age > 20 转换成下面 select * from persion where name = '张三' and provice = '湖北省' and age in (0,1) and age > 20，即可都使用索引
3. 不要在小基数建立索引
4. 长字符串可以采用前缀索引（index(name(20),province,gender)）
5. 范围索引字段，尽量放在最后面 
6. where与order by冲突时，优先where
7. join使用索引关联，小表驱动大表