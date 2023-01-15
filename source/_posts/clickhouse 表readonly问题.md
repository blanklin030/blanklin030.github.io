---
title: clickhouse 表写入报readonly排查
date: 2022-12-08 16:04:11
tags:
  - clickhouse
categories:
  - bigdata
---
### 背景介绍
clickhouse的Replicated*MergeTree表是通过zookeeper来完成同一个shard之间的副本数据的同步，当**Table is in readonly mode**的原因是zookeeper当下压力过大，这个可以从system.replication_queue看到同步队列的数量，或者通过system.part_log看当前part的处理数量

### system.replicas
> 如[官方地址](https://clickhouse.com/docs/en/operations/system-tables/replicas)所解释,该表记录了驻留在本地服务器上的复制表的信息和状态，如下图所示，我们看到每个表的每个副本在zookeeper上的状态

```
select table,zookeeper_path,replica_path from `system`.replicas limit 10
```
![clickhouse](/images/clickhouse/readonly/1.png)
如上图所示，**zookeeper_path**字段可以查看到在哪个shard上的哪个副本处于readonly状态。

### system.part_log
> 如[官方地址](https://clickhouse.com/docs/en/operations/system-tables/part_log)，该表包含part操作的所有记录，包括drop/merge/download/new等。  
```
SELECT *    
FROM system.part_log
WHERE  concat(database, '.', table) = 'i9xiaoapp_stream.dwd_pope_core_action_diff_di_local'
and event_time >= '2023-01-15 12:00:00'
order by event_time desc
```
![clickhouse](/images/clickhouse/readonly/2.png)



### 解决副本不同步问题
> 注意，此时先让用户把写入任务停掉，已方便观察！
+ 可以通过system.part_log查询mergepart状态的写入是否都已经完成
+ 根据system.replicas表给出的readonly状态的zookeeper_path，去到当前所指向的副本机器上，删除这个副本的表
```
drop table i9xiaoapp_stream.dwd_pope_core_action_diff_di_local
```
+ 操作之后，可以通过查询system.replicas里看是否还有处于readonly，发现已经消失
```
select table,zookeeper_path,replica_path from `system`.replicas where concat(database, '.', table) = 'i9xiaoapp_stream.dwd_pope_core_action_diff_di_local' limit 10
```
+ 再在出问题的副本上，重新创建该本，之后可通过system.replicas找到该副本再次出现，并且已恢复
```
create table
```
+ 如果还没恢复，则去对应出错的副本节点，去检查一下zookeeper上的对应路径是否也已经删除
```
rmr zookeeper_path
```
+ 此时再查询system.replicas表readonly的队列应该已经被清理掉了
可以继续操作元数据修改


