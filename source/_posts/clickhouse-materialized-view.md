---
title: clickhouse物化视图在滴滴的实践
date: 2022-04-01 14:57:11
tags:
  - clickhouse
categories:
  - bigdata
---
### 什么是物化视图
+ 普通视图（View）是从一张或者多张数据库表查询导出的虚拟表，可以反映出基础表之间的数据变化，但是本身是不存储数据的，每次的查询都会从基础表重新聚合出查询结果，所以普通视图查询其实等同于创建视图时的查询语句的查询效率。
+ 物化视图（Materialized View）是查询结果集的一份持久化存储，也称为底表的快照（snapshot），查询结果集的范围很宽泛，可以是基础表中部分数据的一份简单拷贝，也可以是多表join之后产生的结果或其子集，或者原始数据的聚合指标等等。物化视图不会随着基础表的变化而变化，如果要更新数据的话，需要用户手动进行，如周期性执行SQL，或利用触发器等机制。



![avatar](/images/clickhouse/crud/5.png)

### 如何创建物化视图
> 
#### MergeTree排序引擎

#### ReplacingMergeTree去重引擎

#### AggregatingMergeTree聚合引擎

#### UniqueMergeTree 实时去重引擎

#### SummingMergeTree 求和引擎

#### CollapsingMergeTree 折叠引擎

#### VersionedCollapsingMergeTree 版本折叠引擎


### 如何写入物化视图

#### 底表触发


#### 同步任务触发


#### 直接写入到分布式表

### 总结
