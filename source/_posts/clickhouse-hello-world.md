---
title: 带你开箱真香ClickHouse
date: 2019-11-26 18:09:40
tags:
  - ClickHouse
categories:
  - bigdata
---
## OLAP介绍
#### OLAP概述
OLAP，也叫联机分析处理（Online Analytical Processing）系统，就是我们说的数据仓库。在这样的系统中，语句的执行量不是考核标准，因为一条语句的执行时间可能会非常长，读取的数据也非常多。所以，在这样的系统中，考核的标准往往是磁盘子系统的吞吐量（带宽），如能达到多少MB/s的流量。
在OLAP系统中，比较典型的应用是BI平台，常使用分区技术、并行技术。 分区技术在OLAP系统中的重要性主要体现在数据库管理上，比如数据库加载，可以通过分区交换的方式实现，备份可以通过备份分区表空间实现，删除数据可以通过分区进行删除，至于分区在性能上的影响，它可以使得一些大表的扫描变得很快（只扫描单个分区）。另外，如果分区结合并行的话，也可以使得整个表的扫描会变得很快。总之，分区主要的功能是管理上的方便性，它并不能绝对保证查询性能的提高，有时候分区会带来性能上的提高，有时候会降低。
#### OLAP特点
+ 大多数是读请求
+ 数据总是以相当大的批(> 1000 rows)进行写入
+ 不修改已添加的数据
+ 每次查询都从数据库中读取大量的行，但是同时又仅需要少量的列
+ 宽表，即每个表包含着大量的列
+ 较少的查询(通常每台服务器每秒数百个查询或更少)
+ 对于简单查询，允许延迟大约50毫秒
+ 列中的数据相对较小： 数字和短字符串(例如，每个URL 60个字节)
+ 处理单个查询时需要高吞吐量（每个服务器每秒高达数十亿行）
+ 事务不是必须的
+ 对数据一致性要求低
+ 每一个查询除了一个大表外都很小
+ 查询结果明显小于源数据，换句话说，数据被过滤或聚合后能够被盛放在单台服务器的内存中
> 很容易可以看出，OLAP场景与其他流行场景(例如,OLTP或K/V)有很大的不同， 因此想要使用OLTP或Key-Value数据库去高效的处理分析查询是没有意义的，例如，使用OLAP数据库去处理分析请求通常要优于使用MongoDB或Redis去处理分析请求。  

## 行式数据库系统介绍
常见的行式数据库系统有： MySQL、Postgres和MS SQL Server。
处于同一行中的数据总是被物理的存储在一起。  

| Row | WatchID | JavaEnable |
| :------ | :------: | ------: |
| #0 | 11111 | 1 |
| #1 | 22222 | 0 |
| #3 | 33333 | 1 |

## 列式数据库系统介绍
常见的列式数据库有： Vertica、 Paraccel (Actian Matrix，Amazon Redshift)、 Sybase IQ、 Exasol、 Infobright、 InfiniDB、 MonetDB (VectorWise， Actian Vector)、 LucidDB、 SAP HANA、 Google Dremel、 Google PowerDrill、 Druid、 kdb+。列式数据库总是将同一列的数据存储在一起，不同列的数据也总是分开存储。  

| Row | #0 | #1 | #2 ｜
| :------ | :------: | ------: | ------: |
| WatchID | 11111 | 22222 | 33333 |
| JavaEnable | 1 | 0 | 1 |

 ## clickhouse介绍
 ClickHouse是一个用于联机分析(**OLAP**)的**列式数据库**管理系统(DBMS)，是俄罗斯的百度——yandex公司开发的。yandex公司在处理自己公司日常数据业务中，开发了一套数据管理系统，随后进行了开源分享，命名为[clickhouse](https://clickhouse.yandex/docs/en/)。
 ### clickhouse优势
+ 线性可扩展
   + 可以部署在虚拟机上，也可以部署在有成百上千个节点（机器）的集群上,单个节点可容纳万亿行数据或超过100TB数据

+ 高效的硬件利用
   + 多核并行处理,数据高效压缩
+ 容错性和高可靠性
   + ClickHouse支持多shard（master/slave）异步复制并且能部署在多个数据中心上
   + 单节点或者整个数据中心宕机不会影响系统读写的可靠性
   + 分布式读取模式，自动（将吞吐压力）均衡于各可用的备份节点上从而避免高时延
   + 宕机恢复后备份间数据自动同步或半自动同步
+ 查询速度快
   + 面向列的DBMS，使用向量化引擎，数据压缩，每秒能处理亿级行级别的数据，简单查询时峰值处理性能可达到单服务器每秒2TB
+ 丰富的特征
   + 支持sql语法，丰富的内置分析统计函数/组件，比如基数、百分位数计算；日期、时间、时区数据函数；URL和IP地址处理函数
   + 数据组织和存储格式丰富，比如存储复杂数据的arrays, array joins, tuples and nested data structures，这些数据结构都做了读写和计算优化，因此用来处理和查询非结构化数据（也可以叫半结构化）也是非常高效率的
   + 支持本地join和分布式join，支持额外定义的字典、从外部导入的维度表，通过简单的语法句子就能无缝join数据
   + 支持近似查询处理，提高获得结果的速度，特别是在处理TB/PB级别数据的时候（比如抽样部分数据计算一个结果然后推广到总体）
   + 支持按条件汇总函数，可以用非常简单的句法查询极值和汇总数据查询  
## clickhouse缺点
+ 不支持Transaction：想快就别想Transaction
+ 聚合结果必须小于一台机器的内存大小：不是大问题
+ 缺少完整的Update/Delete操作
## clickhouse架构
![clickhouse](/images/clickhouse.png)
## clickhou接入
1. http接入
  传入用户名密码的方式
  ```
  // 读方式
  echo 'SELECT 1' | curl 'http://user:password@localhost:8123/?database=dbname' -d @-
  // 写方式
  echo 'SELECT 1' | curl 'http://localhost:8123/?database=dbname&user=user&password=password' -d @-
  ```
2. JDBC驱动
  + ClickHouse官方有 JDBC 的驱动。 见[这里](https://github.com/yandex/clickhouse-jdbc)。

  + 第三方提供的 JDBC 驱动 [clickhouse4j](https://github.com/blynkkk/clickhouse4j) （推荐，依赖较少）。

  ```
  // jdbc的连接字符串是
  jdbc:clickhouse://<host>/<database>
  ```

3. 可视化操作界面Tabix
  ClickHouse Web 界面 [Tabix](https://github.com/tabixio/tabix)

    主要功能：
  + 浏览器直接连接 ClickHouse，不需要安装其他软件。
  + 高亮语法的编辑器。
  + 自动命令补全。
  + 查询命令执行的图形分析工具。
  + 配色方案选项。

4. 命令行客户端
  参照 https://github.com/hatarist/clickhouse-cli 使用，要求Python 3.4+

  pip3安装请[参照](https://www.runoob.com/w3cnote/python-pip-install-usage.html)

  [内容查看](https://github.com/hatarist/clickhouse-cli#configuration-file)
  步骤如下：
  ```
  pip3 install clickhouse-cli
  vi ~/.clickhouse-cli.rc 
  ```
## 表引擎
表引擎（即表的类型）决定了：
+ 数据的存储方式和位置，写到哪里以及从哪里读取数据
+ 支持哪些查询以及如何支持。
+ 并发数据访问。
+ 索引的使用（如果存在）。
+ 是否可以执行多线程请求。
+ 数据复制参数。
### MergeTree
适用于高负载任务的最通用和功能最强大的表引擎。这些引擎的共同特点是可以快速插入数据并进行后续的后台数据处理。 MergeTree系列引擎支持数据复制（使用Replicated* 的引擎版本），分区和一些其他引擎不支持的其他功能
该类型的引擎： 
+ MergeTree 
+ ReplacingMergeTree 
+ SummingMergeTree 
+ AggregatingMergeTree 
+ CollapsingMergeTree
+ VersionedCollapsingMergeTree 
+ GraphiteMergeTree

### 数据副本
只有 MergeTree 系列里的表可支持副本：
+ ReplicatedMergeTree
+ ReplicatedSummingMergeTree
+ ReplicatedReplacingMergeTree
+ ReplicatedAggregatingMergeTree
+ ReplicatedCollapsingMergeTree
+ ReplicatedVersionedCollapsingMergeTree
+ ReplicatedGraphiteMergeTree
副本是表级别的，不是整个服务器级的。所以，服务器里可以同时有复制表和非复制表。
副本不依赖分片。每个分片有它自己的独立副本。
> 对于 INSERT 和 ALTER 语句操作数据的会在压缩的情况下被复制（从master-》slave ）。  
而 CREATE，DROP，ATTACH，DETACH 和 RENAME 语句只会在单个服务器上执行，不会被复制。  
所以在使用集群操作表时需要使用ON CLUSTER cluster_name来通知集群所有机器都操作这个动作。

+ The CREATE TABLE 在运行此语句的服务器上创建一个新的可复制表。如果此表已存在其他服务器上，则给该表添加新副本。
+ The DROP TABLE 删除运行此查询的服务器上的副本。
+ The RENAME 重命名一个副本。换句话说，可复制表不同的副本可以有不同的名称。
要使用副本，需在配置文件中设置 ZooKeeper 集群的地址。例如：
```
<zookeeper>
    <node index="1">
        <host>example1</host>
        <port>2181</port>
    </node>
    <node index="2">
        <host>example2</host>
        <port>2181</port>
    </node>
    <node index="3">
        <host>example3</host>
        <port>2181</port>
    </node>
</zookeeper>
```
注意：ZooKeeper 3.4.5 或更高版本。
### 创建复制表
在表引擎名称上加上 Replicated 前缀。例如：ReplicatedMergeTree。  
Replicated*MergeTree 参数
+ zoo_path — ZooKeeper 中该表的路径。
+ replica_name — ZooKeeper 中的该表的副本名称。  

使用ON CLUSTER cluster功能在集群创建本地表
```
CREATE TABLE db.table_local 
ON CLUSTER cluster_name (
  `name` String COMMENT '姓名',
  `age` String,
  `pt` String,
) 
ENGINE = ReplicatedMergeTree(
  '/clickhouse/tables/{shard}/db/table_local',
  '{replica}'
) 
PARTITION BY (pt)
ORDER BY (age) 
SETTINGS index_granularity = 8192
```
如上例所示，这些参数可以包含宏替换的占位符，即大括号的部分。它们会被替换为配置文件里 'macros' 那部分配置的值。示例：
```
<macros>
    <shard>02</shard>
    <replica>example05-02-1.yandex.ru</replica>
</macros>
```
ZooKeeper 中该表的路径对每个可复制表都要是唯一的。不同分片(shard)上的表要有不同的路径。 这种情况下，路径包含下面这些部分：
/clickhouse/tables/ 是公共前缀，我们推荐使用这个。
{shard} 是分片标识部分。保留 {shard} 占位符即可，它会替换展开为分片标识。
db/table_local 是该表在 ZooKeeper 中的名称。使其与 ClickHouse 中的表名相同加后缀_local来区分。  
注意：也可以显式指定这些参数，而不是使用宏替换。对于测试和配置小型集群这可能会很方便。但是，这种情况下，则不能使用分布式 DDL 语句（ON CLUSTER）。
+ 使用ON CLUSTER cluster功能在集群创建分布式表
```
CREATE TABLE db.table 
ON CLUSTER cluster_name
AS db.table_local
ENGINE = Distributed (
  cluster_name, 
  db,
  table_local,
  rand()
)
```
### 