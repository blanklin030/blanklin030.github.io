---
title: 使用clickhouse实现逻辑CRUD功能
date: 2022-02-03 22:57:11
tags:
  - clickhouse
categories:
  - bigdata
---
### 什么是CRUD
> `CRUD`是指在做计算处理时的增加(`Create`)、检索(`Retrieve`)、更新(`Update`)和删除(`Delete`)几个单词的首字母简写。`crud`主要被用在描述软件系统中数据库或者持久层的基本操作功能

### clickhouse物理上的crud
+ 1. create创建表
```
create table account(
  time DateTime default now() comment '创建时间',
  name String default '' comment '唯一标示',
  alias String default '' comment '别名',
  age UInt64 default 0 comment '年龄',
  version UInt64 default 0 comment '版本号',
  is_delete UInt8 default 0 comment '是否删除，0否，1是'
) 
engine = MergeTree 
partition by toYYYYMMDD(time) 
order by name
```
![avatar](/images/clickhouse/crud/5.png)

+ 2. insert写入数据
```
insert into default.account(time, name, alias, age, version, is_delete) values ('2022-01-01 11:11:11', 'blanklin', 'superhero', 20, 1, 0)
```
> 写入数据后会按照partitionby生成对应的分区part目录
![avatar](/images/clickhouse/crud/6.png)

+ 3. update更新
```
alter table default.account
update alias = 'super_hero'
where alias = 'superhero'
```
> 这里是 mutation 操作,会生成一个mutation_version.txt
![avatar](/images/clickhouse/crud/4.png)

+ 4. delete删除
```
alter table delete where id = 1
```
> 这里是 mutation 操作,会生成一个mutation_version.txt
![avatar](/images/clickhouse/crud/7.png)

+ 5. retrieve检索
```
select time, name, alias, age, version, is_delete from account where is_delete = 0 order by version desc
```

+ 6. 可以通过system.mutations查询执行计划
![avatar](/images/clickhouse/crud/9.png)
当mutation操作执行完成后，system.mutations表中对应的mutation记录中is_done字段的值会变为1。

+ 7. 可以通过system.parts查询执行结果
![avatar](/images/clickhouse/crud/8.png)
当旧的数据片段移除后，system.parts表中旧数据片段对应的记录会被移除。

> 可以看到mutation操作完成后，之前的目录已经被删除
![avatar](/images/clickhouse/crud/10.png)

### clickhouse的mutation是什么
#### 官方文档解释
从官方对于mutaiton的解释[链接](https://clickhouse.com/docs/zh/sql-reference/statements/alter/#alter-mutations)中，我们需要注意到几个关键词，如下
+ 1. manipulate table data 
操作表数据
+ 2. asynchronous background processes
异步后台处理
+ 3. rewriting whole data parts
重写全部数据part
+ 4. a SELECT query that started executing during a mutation will see data from parts that have already been mutated along with data from parts that have not been mutated yet
在突变期间的查询语句，将看到已经完成突变的数据part和还未发生突变的part
+ 5. data that was inserted into the table before the mutation was submitted will be mutated and data that was inserted after that will not be mutated
在提交突变之前插入表中的数据将被突变，而在此之后插入的数据将不会被突变
+ 6. There is no way to roll back the mutation once it is submitted, but if the mutation is stuck for some reason it can be cancelled with the KILL MUTATION query
突变一旦被提交就没有方式可以回滚，但是如果突变由于一些原因被卡住，可以使用`KILL MUTATION`取消突变

#### 源码解读
当用户执行一个如上的Mutation操作获得返回时，ClickHouse内核其实只做了两件事情：
![avatar](/images/clickhouse/crud/1.png)
+ 1. 检查Mutation操作是否合法；
> 主体逻辑在MutationsInterpreter::validate函数
+ 2. 保存Mutation命令到存储文件中，唤醒一个异步处理merge和mutation的工作线程；
> 主体逻辑在StorageMergeTree::mutate函数中。

Merge逻辑
**StorageMergeTree::merge**函数是`MergeTree`异步`Merge`的核心逻辑，`Data Part Merge`的工作除了通过后台工作线程自动完成，用户还可以通过`Optimize`命令来手动触发。自动触发的场景中，系统会根据后台空闲线程的数据来启发式地决定本次`Merge`最大可以处理的数据量大小，**max_bytes_to_merge_at_min_space_in_pool**: 决定当空闲线程数最大时可处理的数据量上限（默认150GB）
**max_bytes_to_merge_at_max_space_in_pool**: 决定只剩下一个空闲线程时可处理的数据量上限（默认1MB）
当用户的写入量非常大的时候，应该适当调整工作线程池的大小和这两个参数。当用户手动触发`merge`时，系统则是根据`disk`剩余容量来决定可处理的最大数据量。

Mutation逻辑
系统每次都只会订正一个`Data Part`，但是会聚合多个`mutation`任务批量完成，这点实现非常的棒。因为在用户真实业务场景中一次数据订正逻辑中可能会包含多个`Mutation`命令，把这多个`mutation`操作聚合到一起订正效率上就非常高。系统每次选择一个排序键最小的并且需要订正`Data Part`进行操作，本意上就是把数据从前往后进行依次订正。
![avatar](/images/clickhouse/crud/11.png)
![avatar](/images/clickhouse/crud/13.png)
![avatar](/images/clickhouse/crud/12.png)

mutation和merge相互独立执行。看完本文前面的分析，大家应该也注意到了目前Data Part的merge和mutation是相互独立执行的，Data Part在同一时刻只能是在merge或者mutation操作中。对于MergeTree这种存储彻底Immutable的设计，数据频繁merge、mutation会引入巨大的IO负载。实时上merge和mutation操作是可以合并到一起去考虑的，这样可以省去数据一次读写盘的开销。对数据写入压力很大又有频繁mutation的场景，会有很大帮助


### clickhouse逻辑CRUD
#### 插入数据
```
insert into default.account(time, name, alias, age, version, is_delete) values ('2022-01-01 11:11:11', 'blanklin', 'superhero', 20, 1, 0)
```

#### 更新数据
+ 1. 先找出要更新的这条数据
```
select time, name, alias, age, version, is_delete from default.account
where age = 'blanklin' and is_delete = 0
```
+ 2. 假设要更新alias=update_super_hero，其他值不变，将version由1.捞出的值上+1，类似以下sql
```
insert into default.account(time, name, alias, age, version, is_delete) values ('2022-01-01 11:11:11', 'blanklin', 'update_superhero', 20, 2, 0)
```
+ 3. 捞出更新后的那条数据
```
select time, name, alias, age, version, is_delete from default.account
where age = 'blanklin' and is_delete = 0
order by version desc
```

#### 删除数据
+ 1. 先找出要更新的这条数据
```
select time, name, alias, age, version, is_delete from default.account
where age = 'blanklin' and is_delete = 0
```
+ 2. 假设要删除alias=update_super_hero，其他值不变，将version由1.捞出的值上+1，将is_delete=1,类似以下sql
```
insert into default.account(time, name, alias, age, version, is_delete) values ('2022-01-01 11:11:11', 'blanklin', 'update_superhero', 20, 2, 1)
```
+ 3. 捞出更新后的那条数据
```
select time, name, alias, age, version, is_delete from default.account
where age = 'blanklin' and is_delete = 0
order by version desc
```

#### 注意项
+ 1. 数据过期问题
定期回收is_delete=1的数据