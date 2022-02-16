---
title: clickhouse query on cluster源码解读
date: 2022-01-21 15:04:11
tags:
  - clickhouse
categories:
  - bigdata
---
### 分布式DDL执行链路
> 在介绍具体的分布式`DDL`执行链路之前，先为大家梳理一下哪些操作是可以走分布式`DDL`执行链路的，大家也可以自己在源码中查看一下`ASTQueryWithOnCluster`的继承类有哪些：  
![cgi](/images/clickhouse/ddloncluster/1.png)  

+ ASTAlterQuery：
包括`ATTACH_PARTITION`、`FETCH_PARTITION`、`FREEZE_PARTITION`、`FREEZE_ALL`等操作（对表的数据分区粒度进行操作）。
+ ASTCreateQuery：
包括常见的建库、建表、建视图，还有`ClickHouse`独有的`Attach Table`（可以从存储文件中直接加载一个之前卸载的数据表）。
+ ASTCreateQuotaQuery:
包括对租户的配额操作语句，例如`create quaota`，或者`alter quota`语句
+ ASTCreateRoleQuery：
包括对租户角色操作语句，例如`create/alter/drop/set/set default/show create role`语句，或者`show roles`
+ ASTCreateRowPolicyQuery
对表的查询做行级别的策略限制，例如`create row policy` 或者 `alter row policy`
+ ASTCreateSettingsProfileQuery
对角色或者租户的资源限制和约束，例如`create settings profile` 或者 `alter settings profile`
+ ASTCreateUserQuery
对租户的操作语句，例如`create create user` 或者 `alter create user`
+ ASTDropAccessEntityQuery
涉及到了clickhouse权限相关的所有删除语句，包括`DROP USER`,`DROP ROLE`,`DROP QUOTA`,`DROP [ROW] POLICY`,`DROP [SETTINGS] PROFILE`
+ ASTDropQuery：
其中包含了三种不同的删除操作（`Drop` / `Truncate` / `Detach`），`Detach Table`和`Attach Table`对应，它是表的卸载动作，把表的存储目录整个移到专门的`detach`文件夹下，然后关闭表在节点`RAM`中的"引用"，这张表在节点中不再可见。
+ ASTGrantQuery:
这是授权相关的`RBAC`，可以对库/表授予或者撤销读/写等权限命令，例如`GRANT insert on db.tb to acount`，或者`REVOKE all on db.tb from account`。
+ ASTKillQueryQuery：
可以`Kill`正在运行的`Query`，也可以`Kill`之前发送的`Mutation`命令。
+ ASTOptimizeQuery：
这是`MergeTree`表引擎特有的操作命令，它可以手动触发`MergeTree`表的合并动作，并可以强制数据分区下的所有`Data Part`合并成一个。
+ ASTRenameQuery：
修改表名，可更改到不同库下。

### DDL Query Task分发
![cgi](/images/clickhouse/ddloncluster/2.png)
`ClickHouse`内核对每种`SQL`操作都有对应的`IInterpreter`实现类，其中的`execute`方法负责具体的操作逻辑。而以上列举的`ASTQuery`对应的`IInterpreter`实现类中的`execute`方法都加入了分布式`DDL`执行判断逻辑，把所有分布式`DDL`执行链路统一都`DDLWorker::executeDDLQueryOnCluster`方法中。
`executeDDLQueryOnCluster`的过程大致可以分为三个步骤：
#### 检查`DDLQuery`的合法性，
+ 1、校验query规则
![cgi](/images/clickhouse/ddloncluster/3.png)
+ 2、初始化DDLWorker，取config.xml表的配置
![cgi](/images/clickhouse/ddloncluster/4.png)
+ 3、替换query里的数据库名称
![cgi](/images/clickhouse/ddloncluster/5.png)
这里替换库名的逻辑是，
+ 3.1、如果query里有带上库名称，则直接使用，若无，则走2
+ 3.2、metrika.xml里配置了shard的默认库`<default_database>default</default_database>`，则使用默认库，否则走3
+ 3.3、使用当前session的database
![cgi](/images/clickhouse/ddloncluster/6.png)

#### 把`DDLQuery`写入到`Zookeeper`任务队列中
+ 1、构造DDLLogEntry对象，把entry对象加入到queue队列中
![cgi](/images/clickhouse/ddloncluster/7.png)
注意：queue_dir是由**config.xml**配置的，如下
```
<distributed_ddl>
    <path>/clickhouse/task_queue/ddl</path>
</distributed_ddl>
```
+ 2、去zookeeper执行创建znode，把entry序列化存入znode
![cgi](/images/clickhouse/ddloncluster/8.png)
+ 3、在znode下创建active和finished的znode
![cgi](/images/clickhouse/ddloncluster/9.png)
> 下面截图为query-xxx的记录的entry内容
![cgi](/images/clickhouse/ddloncluster/10.png)

#### 等待`Zookeeper`任务队列的反馈把结果返回给用户。
![cgi](/images/clickhouse/ddloncluster/11.png)

#### DDL Query Task执行线程
+ 1、DDLWorker构造函数去取了config.xml配置，并且开启了2个线程，分别是执行线程和清理线程
![cgi](/images/clickhouse/ddloncluster/12.png)
+ 2、执行线程加入到线程池后，执行ddl task
![cgi](/images/clickhouse/ddloncluster/13.png)
+ 3、过滤掉 query 中带有 on cluster xxx的语句，根据不同的query选择不同执行方式
![cgi](/images/clickhouse/ddloncluster/14.png)
+ 4、alter、optimize、truncate语句需要在leader节点执行
![cgi](/images/clickhouse/ddloncluster/15.png)
> 注意：Replcated表的alter、optimize、truncate这些query是会先判断是否leader节点，不是则不处理，在执行时，会先给zookeeper加一个分布式锁，锁住这个任务防止被修改，执行时都是把自己的host:port注册到znode/query-xxx/active下，执行完成后，结果写到znode/query-xxx/finished下。

#### DDL Query Task清理线程
+ 1、DDLWorker构造函数去取了config.xml配置，并且开启了2个线程，分别是执行线程和清理线程
![cgi](/images/clickhouse/ddloncluster/12.png)
+ 2、执行清理逻辑，每次执行后，下一次执行需要过1分钟后才可以接着做清理
![cgi](/images/clickhouse/ddloncluster/16.png)

### 分布式DDL的执行链路总结
+ 1）节点收到用户的分布式`DDL`请求

+ 2）节点校验分布式DDL请求合法性，在`Zookeeper`的任务队列中创建`Znode`并上传**DDL LogEntry（query-xxxx）**，同时在`LogEntry`的`Znode`下创建`active`和`finish`两个状态同步的`Znode`

+ 3）`Cluster`中的节点后台线程消费`Zookeeper`中的`LogEntry`队列执行处理逻辑，处理过程中把自己注册到`acitve Znode`下，并把处理结果写回到`finish Znode`下
![cgi](/images/clickhouse/ddloncluster/12.png)

+ 4）用户的原始请求节点，不断轮询`LogEntry Znode`下的`active`和`finish`状态`Znode`，当目标节点全部执行完成任务或者触发超时逻辑时，用户就会获得结果反馈

这个分发逻辑中有个值得注意的点：分布式`DDL`执行链路中有超时逻辑，如果触发超时用户将无法从客户端返回中确定最终执行结果，需要自己去`Zookeeper`上`check`节点返回结果（也可以通过**system.zookeeper**系统表查看）。**每个节点只有一个后台线程在消费执行DDL任务**，碰到某个DDL任务（典型的是optimize任务）执行时间很长时，会导致DDL任务队列积压从而产生大面积的超时反馈。

可以看出Zookeeper在分布式DDL执行过程中主要充当DDL Task的分发、串行化执行、结果收集的一致性介质。