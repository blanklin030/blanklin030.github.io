---
title: clickhouse集群cpu飙高问题排查
date: 2022-12-08 19:35:11
tags:
  - clickhouse
categories:
  - bigdata
---
### 背景介绍
clickhouse是分布式系统，一条查询sql会经过pipeline处理后通过后台的查询线程池开启多线程查询，而经常会收到cpu飙高告警，则是因为这条sql开启了系统所有的cpu资源（多个线程）进行计算

### 告警图
如下图所示，提示cpu飙高
![clickhouse](/images/clickhouse/cpu_high/1.png)
如下图所示，ganglia上展示了当时的cpu顶着线在跑
![clickhouse](/images/clickhouse/cpu_high/2.png)
### system.query_log
> [地址](https://clickhouse.com/docs/en/operations/system-tables/query_log)，该表包含了所有进入到ck的sql语句。
该表包含字段如下：
+ type (Enum8) — 执行查询时的事件类型. 值:
  + 'QueryStart' = 1 — 查询成功启动.
  + 'QueryFinish' = 2 — 查询成功完成.
+ 'ExceptionBeforeStart' = 3 — 查询执行前有异常.
+ 'ExceptionWhileProcessing' = 4 — 查询执行期间有异常.
+ event_date (Date) — 查询开始日期.
+ event_time (DateTime) — 查询开始时间.
+ event_time_microseconds (DateTime64) — 查询开始时间（毫秒精度）.
+ query_start_time (DateTime) — 查询执行的开始时间.
+ query_start_time_microseconds (DateTime64) — 查询执行的开始时间（毫秒精度）.
+ query_duration_ms (UInt64) — 查询消耗的时间（毫秒）.
+ read_rows (UInt64) — 从参与了查询的所有表和表函数读取的总行数. 包括：普通的子查询, IN 和 JOIN的子查询. 对于分布式查询 read_rows 包括在所有副本上读取的行总数。 每个副本发送它的 read_rows 值，并且查询的服务器-发起方汇总所有接收到的和本地的值。 缓存卷不会影响此值。
+ read_bytes (UInt64) — 从参与了查询的所有表和表函数读取的总字节数. 包括：普通的子查询, IN 和 JOIN的子查询. 对于分布式查询 read_bytes 包括在所有副本上读取的字节总数。 每个副本发送它的 read_bytes 值，并且查询的服务器-发起方汇总所有接收到的和本地的值。 缓存卷不会影响此值。
+ written_rows (UInt64) — 对于 INSERT 查询，为写入的行数。 对于其他查询，值为0。
+ written_bytes (UInt64) — 对于 INSERT 查询时，为写入的字节数。 对于其他查询，值为0。
+ result_rows (UInt64) — SELECT 查询结果的行数，或INSERT 的行数。
+ result_bytes (UInt64) — 存储查询结果的RAM量.
+ memory_usage (UInt64) — 查询使用的内存.
+ query (String) — 查询语句.
+ exception (String) — 异常信息.
+ exception_code (Int32) — 异常码.
+ stack_trace (String) — Stack Trace. 如果查询成功完成，则为空字符串。
+ is_initial_query (UInt8) — 查询类型. 可能的值:
  + 1 — 客户端发起的查询.
  + 0 — 由另一个查询发起的，作为分布式查询的一部分.
+ user (String) — 发起查询的用户.
+ query_id (String) — 查询ID.
+ address (IPv6) — 发起查询的客户端IP地址.
+ port (UInt16) — 发起查询的客户端端口.
+ initial_user (String) — 初始查询的用户名（用于分布式查询执行）.
+ initial_query_id (String) — 运行初始查询的ID（用于分布式查询执行）.
+ initial_address (IPv6) — 运行父查询的IP地址.
+ initial_port (UInt16) — 发起父查询的客户端端口.
+ interface (UInt8) — 发起查询的接口. 可能的值:
  + 1 — TCP.
  + 2 — HTTP.
+ os_user (String) — 运行 clickhouse-client的操作系统用户名.
+ client_hostname (String) — 运行clickhouse-client 或其他TCP客户端的机器的主机名。
+ client_name (String) — clickhouse-client 或其他TCP客户端的名称。
+ client_revision (UInt32) — clickhouse-client 或其他TCP客户端的Revision。
+ client_version_major (UInt32) — clickhouse-client 或其他TCP客户端的Major version。
+ client_version_minor (UInt32) — clickhouse-client 或其他TCP客户端的Minor version。
+ client_version_patch (UInt32) — clickhouse-client 或其他TCP客户端的Patch component。
+ http_method (UInt8) — 发起查询的HTTP方法. 可能值:
  + 0 — TCP接口的查询.
  + 1 — GET
  + 2 — POST
+ http_user_agent (String) — http请求的客户端参数
+ quota_key (String) — 在quotas 配置里设置的“quota key” （见 keyed).
+ revision (UInt32) — ClickHouse revision.
+ ProfileEvents (Map(String, UInt64))) — 不同指标的计数器 
+ Settings (Map(String, String)) — 当前请求里的setting部分参数
+ thread_ids (Array(UInt64)) — 参与查询的线程数.
+ Settings.Names (Array（String)) — 客户端运行查询时更改的设置的名称。 要启用对设置的日志记录更改，请将log_query_settings参数设置为1。
+ Settings.Values (Array（String)) — Settings.Names 列中列出的设置的值。

### 解决过程
+ 查询当前时间内耗cpu最高的sql前10条
```
select initial_user, event_time, query,
read_rows, read_bytes,
written_rows, written_bytes,
result_rows, result_bytes,
memory_usage, length(thread_ids) as thread_count
from system.query_log
WHERE event_time > '2022-10-27 18:30:00' AND event_time < '2022-10-27 18:35:00' 
and initial_user<>'default'
order by thread_count desc
limit 10;
```
+ 