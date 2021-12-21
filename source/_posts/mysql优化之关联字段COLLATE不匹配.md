---
title: mysql优化之关联字段COLLATE不匹配
date: 2021-12-10 11:44:58
tags:
    - mysql
categories:
    - mysql
---

# mysql优化之关联字段COLLATE不匹配

&nbsp;&nbsp;&nbsp;&nbsp;最近正在学习dubbo知识，这里临时插入一个mysql的知识点，也算给自己扫盲。也就是下面两个知识点
* [x] mysql join collate 不一样，转换查询非常耗时
* [x] mysql唯一索引不区分大小写，该字段（重复名称）无法区分（例如w,W就不能区分）

## collate
&nbsp;&nbsp;&nbsp;&nbsp;这个字段是来说明mysql如果对该列进行排序和比较，会影响order by的顺序，会影响到where条件中大于小于筛选出来的结果，也会影响distinct、group by、having语句的查询结果

### utf8mb4_bin & utf8_bin
* UTF-8是使用1~4个字节，一种变长的编码格式，字符编码。mb4即 most bytes 4，使用4个字节来表示完整的UTF-8
* mysql的 utf8 编码最大字符长度为 3 字节，如果遇到 4 字节的宽字符就会插入异常了

<!-- more -->

### 问题sql
```sql
SELECT DISTINCT
  p.project_type_id 
FROM
  project p
  INNER JOIN project_qualifications pq ON pq.project_code = p.project_code 
WHERE
  p.project_status = 20 
  AND pq.is_delete = FALSE 
  AND pq.qualifications_id IN (
    598,
    597,
    599,
    596,
    593,
    595,
    594,
    595,
    592,
    1234,
    1279,
    1231 
  );
```
### 优化前
两张表关联查询，然后distinct返回，两张表其实都不大，索引也是正常使用，project_code也是varchar(40)类型，然后耗时1s多，有点不正常

![优化前](/images/pasted-102.jpg)

### 优化后
经过阿里云监控描述是存在强制转换，然后对比表，发现collate导致，修改后直接效率完全升上去了

![优化后](/images/pasted-103.jpg)


## mysql唯一索引不区分大小写
* mysql 增加Binary方式来解决
* 通过程序来解决


## 临时插入知识点最后

<p>临时插入的小知识点......给自己扫盲而已</p>