---
title: clickhouse离线导入大揭秘
date: 2022-09-26 22:57:11
tags:
  - clickhouse
categories:
  - bigdata
---
### 如何按照hive表的表结构创建clickhouse表
点击[链接](http://studio.data.didichuxing.com/stream/create-table?tableType=clickHouse)通过数梦的实时平台可以快捷创建对应hive表结构的ck表，如下图所示：
![clickhouse](/images/clickhouse/hive2clickhouse/1.png)

### hive表的数据如何导入clickhouse表
#### 创建同步任务
点击[链接](http://sync.data-pre.didichuxing.com/job/offline/edit/818177?step=1)通过数梦的同步中心可以快捷创建hive表映射到ck表的同步任务，如下图所示
![clickhouse](/images/clickhouse/hive2clickhouse/2.png)

#### 同步任务流程
如下图所示，我们按照这个流程处理hive数据导入到clickhouse
![clickhouse](/images/clickhouse/hive2clickhouse/3.png)
+ 1. 创建临时表
按照ck表的表结构，我们会在集群的所有写节点创建同样表结构的单机表（MergeTree）引擎
+ 2. 起spark任务，利用clickhouse-local工具将hive表导入到临时表
```
cat data.orc | clickhouse-local \
--format=Native \
--query='CREATE TABLE input (col1 String, col2 String, col3 String) ENGINE = File(ORC, stdin);CREATE TABLE target_table (col1 String, col2 String, col3 String) ENGINE = MergeTree() partition by tuple() order by col1;insert into target_table select *,"$year" as year,"$month" as month, "$day" as day from input;optimize table target_table final' \
--config-file=config.xmls
```