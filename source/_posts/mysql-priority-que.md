---
title: mysql在使用order by limit时出现数据重复的问题
date: 2019-12-04 17:39:59
tags:
  - mysql
categories:
  - database
---
### 背景
用户反馈说分页数据重复，查看日志，发现说这样的sql：
```
select column1,column2 
from table
where columnx = 'xxx'
order by columny
limit current,size
```
### 原因
+ mysql版本
mysql5.7.20  
查看mysql的[changes list](https://dev.mysql.com/doc/relnotes/mysql/5.6/en/news-5-6-16.html)  
> MySQL 5.6 has only one way one can check whether filesort used priority queue. You need to enable optimizer_trace (set optimizer_trace=1), and then run the query (not EXPLAIN, but the query itself). Then, you can look into the optimizer trace 
+ order by
MySQL 执行查询语句， 对于order by谓词，可能会使用filesort或者temporary。比如explain一条语句的时候，会看到Extra字段中可能会出现，using filesort和using temporary。下面我们就来探讨下两个的区别和适用场景。
+ filesort
filesort主要用于查询数据结果集的排序操作，首先MySQL会使用**sort_buffer_size**大小的内存进行排序，如果结果集超过了**sort_buffer_size**大小，会把这一个排序后的**chunk**转移到**file**上，最后使用**多路归并排序**完成所有数据的排序操作。
filesort有两种使用模式:
模式1: sort的item保存了所需要的所有字段，排序完成后，没有必要再回表扫描。
模式2: sort的item仅包括，待排序完成后，根据rowid查询所需要的columns。
> 很明显，模式1能够极大的减少回表的随机IO。
+ priority query
1. 打开optimizer_trace
```
mysql> set session optimizer_trace=’enabled=on';
```
2. 查询information_schema.optimizer_trace表
主要分为三个部分 
join_preparation：SQL的准备阶段，sql被格式化
对应函数 JOIN::prepare
join_optimization：SQL优化阶段
对应函数JOIN::optimize
join_execution：SQL执行阶段
对应函数：JOIN::exec
3. 查看sql执行阶段
```
"join_execution": {
"select#": 1,
"steps": [
    {
    "filesort_information": [
        {
        "direction": "asc",
        "table": "`film`",
        "field": "actor_age"
        }
    ],
    "filesort_priority_queue_optimization": {
        "usable": false,
        "cause": "not applicable (no LIMIT)"
    },
    "filesort_execution": [
    ],
    "filesort_summary": {
        "rows": 1,
        "examined_rows": 5,
        "number_of_tmp_files": 0,
        "sort_buffer_size": 261872,
        "sort_mode": "<sort_key, packed_additional_fields>"
    }
    }
]
}
```
当使用order by limit时，filesort_priority_queue_optimization给出的useable为true，在mysql5.6之后（包括5.6）做了一个优化，即**priority queue**，使用**priority queue**的目的是为了加快查询，如果sql中未使用到索引有序性时，**order by limit**会在满足**buffer size**设置后返回只保证返回**limit size**，但不保证排序出来的结果和读出来的数据顺序一致，这就是堆排序对效果，先select出符合条件的rows，再order by column，如果这个column没有明确的唯一性，则再limit后，只是会随机出limit的size条数，这个随机性不保证数据是否重复
### 解决方法
+ 使用有序索引排序
```
select from table where xx order by column desc,id desc limit size
```
+ 给order by 的column加上索引
```
alter table add index (column)
```
+ 强制使用索引
```
select from table force index(column) order by column limit size
```

