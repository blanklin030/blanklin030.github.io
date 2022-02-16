---
title: clickhouse报错Metadata on replica的解决方法
date: 2022-01-13 19:04:11
tags:
  - clickhouse
categories:
  - bigdata
---
### 背景介绍
```
ClickHouse exception, code: 517, host: xx.xx.xx.xx, port: xxxx; Code: 517, e.displayText() = DB::Exception: Metadata on replica is not up to date with common metadata in Zookeeper. Cannot alter: Bad version
```
我们在修改表结构（例如`alter table drop column xxx`）时经常会遇到以上报错，原因副本上的元数据和在zookeeper上的元数据不一致，无法更新，因为版本号不一样。

### 查找system.replication_queue
+ 表介绍
[链接地址](https://clickhouse.com/docs/en/operations/system-tables/replication_queue/)包含了所有`ReplicatedMergeTree`复制表家族在`zookeeper`上存储的副本任务队列的相关信息。
+ 列信息
  + database (String) — 数据库名称.
  + table (String) — 表名称.
  + replica_name (String) — zookeeper上副本名称，同表不同副本有不同名字.
  + position (UInt32) — 当前任务队列的位置.
  + node_name (String) — ZooKeeper上节点名称.
  + type (String) — 任务队列的名称，分别是:
    + GET_PART — 从另一个副本拿到part.
    + ATTACH_PART — 加载part, 有可能来自我们自己的副本 (如果是在detached目录上发现). 您可以认为它是带有一些优化的GET_PART，因为他们接近相似.
    + MERGE_PARTS — 合并part.
    + DROP_RANGE — 删除指定范围内的指定分区列表.
    + CLEAR_COLUMN — 注意：从指定分区删除特殊列（已弃用）.
    + CLEAR_INDEX — 注意：从指定分区删除指定索引（已弃用）.
    + REPLACE_RANGE — 删除一定范围内的part，并用新part替换.
    + MUTATE_PART — 对part应用用一个或多个mutation.
    + ALTER_METADATA — 根据/metadata和/columns路径应用alter修改.
  + create_time (Datetime) — 任务被提交执行后的时间.
  + required_quorum (UInt32) — 等待任务完成并且确认完成的副本数. 这个字段仅跟GET_PARTS任务相关.
  + source_replica (String) — 源副本的名称.
  + new_part_name (String) — 新part的名称.
  + parts_to_merge (Array (String)) — 要更新或者合并的part名称.
  + is_detach (UInt8) — DETACH_PARTS任务是否在任务队列里的标示位.
  + is_currently_executing (UInt8) — 当然任务是否正在执行的标示位.
  + num_tries (UInt32) — 尝试完成任务失败的次数.
  + last_exception (String) — 上次错误发生的详情.
  + last_attempt_time (Datetime) — 任务最后一次尝试的时间
  + num_postponed (UInt32) — 延期任务数.
  + postpone_reason (String) — 任务延期时间.
  + last_postpone_time (Datetime) — 任务上次延期的时间.
  + merge_type (String) — 当前合并的类型. 如果是mutation则为空.

+ 通过该表捞出zookeeper的节点副本信息
```
select node_name from system.replication_queue where database = 'xx' and table = 'xx' and is_currently_executing = 1
```
### system.replcas

### system.mutations

### 处理方式
+ 