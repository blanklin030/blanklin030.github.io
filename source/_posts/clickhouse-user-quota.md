---
title: ClickHouse用户资源隔离在滴滴的实践
date: 2021-12-11 18:09:40
tags:
  - ClickHouse
categories:
  - bigdata
---
### 背景介绍
大数据架构`olap`团队对`clickhouse`的使用是从2019年5月开始，最开始我们只在国内永顺机房落地了4台机器做共享集群，就是最开始的`cluster01`集群，4台机器做2个`shard`，每个`shard`2个`replica`，而用户对集群的访问只是通过一个`vip`做简单的负载均衡，关于权限这一块，我们在当时只做到了库级别的管控，简单说就是通过同步`dpp_v3`.`hadoop_account`和`dpp_v3`.`hive_database`库去更新`user.xml`文件，如下图所示，我们并未对租户资源做任何隔离，虽然最开始我们就是使用的多租户模型，但每个租户可以有权限去操作自己名下的所有库，包括读和写，且无限制的进行大量耗资源（`cpu`和`memory`）写入和查询后就会影响到`cluster01`集群下其他的租户，因为`cpu`和`memory`是固定的，而租户的分摊并未做限制。
![avatar](/images/clickhouse/authority/2.png)
### clickhouse配置详解
#### config.xml文件介绍
`clickhouse`的启动方式是通过以下命令处理的，就让 `ClickHouse` 按照`config.xml`配置文件运行，同时 `ClickHouse` 监听配置文件，如有变化，不需要重启就能按新的配置运行。具体介绍可以参考[链接](https://clickhouse.com/docs/en/operations/configuration-files/#implementation-details)
```
./bin/clickhouse-server --config=./conf/config.xml 
```
我们在生产环境下`config.xml`文件主要是如下内容：
```
<?xml version="1.0"?> 
<yandex> 
    <!-- 日志配置 -->
    <logger> 
        <level>trace</level> 
        <log>/tmp/log/clickhouse-server.log</log> 
        <errorlog>/tmp/log/clickhouse-server.err.log</errorlog> 
        <size>1000M</size> 
        <count>10</count> 
    </logger> 
    <!-- 开启查询和写入相关日志配置 -->
    <query_log> 
        <database>system</database> 
        <table>query_log</table> 
        <partition_by>toYYYYMM(event_date)</partition_by> 
        <flush_interval_milliseconds>1000</flush_interval_milliseconds> 
    </query_log> 
    <!-- tcp端口 -->
    <tcp_port>9000</tcp_port> 
    <!-- 运行所有ip访问 -->
    <listen_host>0.0.0.0</listen_host> 
    <!-- 最大连接数 -->
    <max_connections>4096</max_connections>
    <!-- 连接超时时间 -->
    <keep_alive_timeout>90</keep_alive_timeout>
    <!-- 最大并发查询数 -->
    <max_concurrent_queries>1000</max_concurrent_queries>
    <!-- 用户配置文件 -->
    <users_config>users.xml</users_config>
    <!-- 默认配置名称 -->
    <default_profile>default</default_profile>
    <!-- 默认数据库名称 -->
    <default_database>default</default_database>
    <!-- zookeeper配置 -->
    <zookeeper incl="zookeeper-servers" optional="true" />
    <!-- 宏配置 -->
    <macros incl="macros" optional="true" />
    <!-- 权限相关的sql存储路径 -->
    <access_control_path>/var/lib/clickhouse/access</access_control_path>
</yandex> 
```
我们应该注意到 `<users_config>users.xml</users_config>` 绑定了 `config.xml` 当前目录的 `users.xml`,而 `users.xml`就是我们最初版本的用户配置文件
#### users.xml文件介绍
`ClickHouse`支持基于 [RBAC](https://en.wikipedia.org/wiki/Role-based_access_control)（基于角色的访问控制权限）方法的访问控制管理。作为一个分析类型（`OLAP`）的数据库系统，相对于`MySQL`数据库在用户管理方面有很大不同，`clickhouse`支持使用两种方式配置访问实体：
+ 通过`sql`直接设置，这也是官方推荐的，但是需要至少一个用户帐户启用`SQL`驱动的访问控制和帐户管理，这需要使用第二种方式设置`access_management`
+ 通过配置文件`users.xml`，默认位置在`/etc/clickhouse-server`目录下，`ClickHouse`使用它来定义用户相关的配置项。
> 注意，您不能同时通过两种配置方法来管理同一访问实体。（You can’t manage the same access entity by both configuration methods simultaneously.）
`users.xml`有三大块进行说明，分别为：`profiles`，`quotas`，`users`，主要配置如下所示：
+ profiles介绍
[官方链接](https://clickhouse.com/docs/en/operations/settings/settings-profiles/)
```
<!-- profiles相当于role角色配置 -->
<profiles> 
    <!-- 角色名称，可配置多个 -->
    <default> 
        <!-- 单个服务器 最大可使用内存 -->
        <max_memory_usage>10G</max_memory_usage> 
        <!-- 单个服务器 用户查询最大可使用内存 -->
        <max_memory_usage_for_user>10G</max_memory_usage_for_user> 
        <!-- 所有查询可使用的最大内存 -->
        <max_memory_usage_for_all_queries>10G</max_memory_usage_for_all_queries> 
        <!-- 最大查询长度 -->
        <max_query_size>1073741824</max_query_size> 
        <!-- DDL查询：CREATE，ALTER，RENAME，ATTACH，DETACH，DROP TRUNCATE,0:禁止，1:允许 -->
        <allow_ddl>0</allow_ddl>
    </default> 
    <readonly>
      <!-- 只读角色，枚举0:允许所有查询，1:只允许读数据的查询，3:运行读取数据和更改配置 -->
      <readonly>2</readonly>
    </readonly>
</profiles> 
```
 + quotas介绍   
 [官方链接](https://clickhouse.com/docs/en/operations/quotas/)，截取部分代码实现细节如下,从源码中我们能看出，每次查询的时候，都去检查(`checkExceeded()`)是否超过配额
 ![avatar](/images/clickhouse/authority/3.png)，每个 interval 都有多种资源(`resource_type`), 比如 ``<query>1</query>` 是一种 `type`, 检查最大库存 `max`，检查已经使用的配额 `used`, 如果 `used` > `max`, 则报错。
 ![avatar](/images/clickhouse/authority/4.png)
 ```
<!-- 配额，限制用户一段时间内的资源使用，即对一段时间内运行的一组查询施加限制，而不是限制单个查询。 -->
<quotas> 
    <!-- 配额名称 -->
    <default> 
        <!-- 时间间隔 -->
        <interval> 
            <!-- 时间周期 以秒为单位 --> 
            <duration>10</duration> 
            <!-- 10 秒内只能查询一次 --> 
            <queries>2</queries> 
            <errors>0</errors> 
            <!-- 时间周期内允许返回的行数，0表示不限制 -->
            <result_rows>0</result_rows> 
            <!-- 时间周期内运行读取的行数，0表示不限制 -->
            <read_rows>0</read_rows> 
            <!-- 时间周期内查询的可执行时间，0表示不限制 -->
            <execution_time>0</execution_time> 
        </interval>
    </default> 
</quotas> 
```
+ users介绍
[官方链接](https://clickhouse.com/docs/en/operations/quotas/)
```
<!-- 用户配置,定义一个新用户，必须包含以下几项属性：用户名、密码、访问ip、数据库、表等等。它还可以应用上面的profile、quota -->
<users> 
    <!-- 用户名 -->
    <default> 
        <!-- 此设置为用户启用或禁用SQL驱动的访问控制和帐户管理。可能的值：0-禁用。1-启用。默认为0 -->
        <access_management>1</access_management>
        <!-- 密码 -->
        <password>password</password> 
        <!-- 用户可以从中连接到ClickHouse服务器的网络列表 -->
        <networks> 
            <!-- 要为来自任何网络的用户打开访问权限 -->
            <ip>::/0</ip> 
        </networks> 
        <!-- 指定用户的角色 -->
        <profile>default</profile> 
        <!-- 指定用户的配额 -->
        <quota>default</quota> 
    </default> 
</users> 
```
#### 常见的权限报错
+ 超过quota限制
```
Code: 201. DB::Exception: Received from localhost:9000. DB::Exception: Quota for user `default` for 10s has been exceeded: queries = 4/3. Interval will end at 2020-04-02 11:29:40. Name of quota template: `default`.
```
如果在至少一个时间间隔内超过了限制，则将引发一个异常，并显示一条文本：是对于哪一个间隔的，何时新间隔可以开始（何时可以再次发送查询）。
+ 超过profile限制
```
Code: 452, e.displayText() = DB::Exception: Setting max_memory_usage should not be greater than 20000000000.
Code: 452, e.displayText() = DB::Exception: Setting max_memory_usage should not be less than 5000000000.
```
查询超过了最大使用内存
+ user限制
```
Code: 516, e.displayText() = DB::Exception: prod_dlap_manager: Authentication failed: password is incorrect or there is no user with such name (version 206.1.1)
```
用户不存在，或者用户的密码错误
### 改进方案
经过对`clickhouse`的权限相关了解之后，我们在2021年8月进行了一次权限升级方案改造，通过default用户创建出超级管理员super_admin及普通管理员admin（如果在user.xml里定义了super_admin用户，之后就无法修改，若修改则直接报错`Cannot update user admin in users.xml because this storage is readonly`），以超级管理员的身份给超级租户和普通租户赋权，可以通过SQL的方式进行权限的`CRUD`，以达到动态分配集群资源的目的。
![avatar](/images/clickhouse/authority/5.png)
经过拆分后，我们单独自研了`DlapManager`组件，用以管控`Clickhouse`集群读写节点、`zookeeper`组件、`CHProxy`组件（采用开源工具快速接入以实现读写ck角色的区分，达到数据写入均衡查询均衡的 目的，[链接](https://github.com/Vertamedia/chproxy)），而`CHProxy`组件需要的集群读写资源则通过`DlapManager`角色进行实时更新和生产，而外部平台业务方直接调用`DlapManager`组件对外开放的`api`进行库表的所有`DDL`操作，目前我们的所有普通租户均是用户通过数据梦工厂的创建项目开通实时权限后，我们会自动创建一个租户在`Clickhouse`这边，当该租户进行库表`DDL`后，`DlapManager`组件就会生成相应的`DDL`任务队列，队列以异步线程方式实时发往对应的集群，以下的`DlapManager`组件的系统架构设计图：
![avatar](/images/clickhouse/authority/6.png)
### 赋权过程
按照[官方推荐](https://clickhouse.com/docs/en/operations/access-rights/#enabling-access-control)的方式进行赋权过程，以达到不同租户使用相应的配额，实现集群内的资源隔离，保障集群的稳定性。
+ 创建超级租户
```
CREATE USER super_admin; 
GRANT ALL ON *.* TO super_admin WITH GRANT OPTION; 
CREATE USER admin; 
```
+ 创建profiles设置
```
CREATE SETTINGS PROFILE IF NOT EXISTS didi_profile SETTINGS readonly = 2 READONLY 
```
+ 创建quota配额
```
CREATE QUOTA IF NOT EXISTS didi_quota 
FOR INTERVAL 10 second 
MAX queries 1 
```
+ 创建角色
```
CREATE ROLE IF NOT EXISTS didi_role 
```
+ 赋予 Role 权限
```
# 允许didi_role这个角色可以访问库名叫db的所有表的查询权限
GRANT SELECT ON db.* TO didi_role
```
+ 创建一个回收角色，用以回收不使用的profile、quota
```
REATE ROLE IF NOT EXISTS gc_role 
```
+ Role 绑定 Profile, Quota
```
ALTER SETTINGS PROFILE didi_profile TO didi_role;
ALTER QUOTA didi_quota TO didi_role;
```
+ 应用到租户
```
GRANT didi_role TO super_admin
```
+ 验证租户角色
```
SELECT * FROM system.role_grants WHERE user_name LIKE 'admin'
```
+ 修改用户的quota(当用户的读/写超过了限额后需要给用户扩容)
```
# 先创建新quota
CREATE QUOTA IF NOT EXISTS new_quota FOR INTERVAL 5 second MAX queries 1;
# 将原先的didi_quota绑定到gc_role
ALTER QUOTA didi_quota TO gc_role;
# 绑定新quota
ALTER QUOTA new_quota TO didi_role; 
# 刷新admin的角色
revoke didi_role from admin; 
grant didi_role to admin;
# double check检查
SELECT name, apply_to_list FROM system.quotas WHERE name LIKE 'new_quota'
```
+ 修改用户的profile(当用户的读/写超过了限额后需要给用户扩容)
```
# 先创建新profile
CREATE SETTINGS PROFILE IF NOT EXISTS new_profile SETTINGS readonly = 0 READONLY;
# 将原先的didi_profile绑定到gc_role
ALTER SETTINGS PROFILE didi_profile TO gc_role
# 绑定新profile
ALTER SETTINGS PROFILE new_profile TO didi_role;
# 刷新admin的角色
revoke didi_role from admin; 
grant didi_role to admin;
# double check检查
SELECT name, apply_to_list FROM system.settings_profiles WHERE name LIKE 'new_quota'
```
### 权限持久化
+ 存放目录配置
根据[官方文档](https://clickhouse.com/docs/en/operations/server-configuration-parameters/settings/#access_control_path)介绍，通过`sql`进行的权限相关配置，可以通过`config.xml`里的`access_control_path`属性进行路径配置，若未配置则默认是`/var/lib/clickhouse/access/`目录，如下图所示，当`clickhouse`重启后，都会先从`access_control_path`配置的目录里根据`sql`恢复所有权限，如果这些文件被删除，则重启`clickhouse`后之前通过`sql`创建的这些`setting`都会消失，所以注意文件的备份，目前`DlapManager`组件是使用`mysql`数据存储引擎单独存储了这份`RBAC`数据，再发送给`clickhouse`进行权限相关的`CRUD`，这个方式也相当于是`clickhouse`的备份地址，如果`clickhouse`重启后无法刷新这些权限，则仍然可以通过`DlapManager`组件写的脚本工具去重新生产出新的`sql`给`clickhouse`使用。
![avatar](/images/clickhouse/authority/7.png)
+ users.list介绍
每创建一个用户，则会在`users.list`文件里记录到用户名对应的一串`uuid`，可通过`uuid`找到创建该用户的`sql`语句，如下图：
![avatar](/images/clickhouse/authority/8.png)
+ uuid.sql介绍
包含了创建用户（create user）的sql，赋权（GRANT）的sql等
![avatar](/images/clickhouse/authority/9.png)
+ quotas.list介绍
每创建一个`quota`，都会在`quotas.list`文件记录quota名称及对应的一个`uuid`，可通过`uuid`找到创建该`quota`的`sql`语句
+ roles.list介绍
每创建一个`role`，都会在`roles.list`文件记录quota名称及对应的一个`uuid`，可通过`uuid`找到创建该`role`的`sql`语句
+ row_policies.list介绍
每创建一个`row_policy`，都会在`row_policies.list`文件记录quota名称及对应的一个`uuid`，可通过`uuid`找到创建该`row_policy`的`sql`语句
+ settings_profiles.list介绍
每创建一个`profile`，都会在`settings_profiles.list`文件记录quota名称及对应的一个`uuid`，可通过`uuid`找到创建该`profile`的`sql`语句
### 结论
本文从`clickhouse`的用户资源角度来简单介绍了`clickhouse`的相关配置及如何使用，而我们自研的`DlapManager`组件则承担起了 `ClickHouse` 用户管理的角色，通过对`clickhouse`的`profle`、`quota`、`role`等抽象层的配置来达到对`clickhouse`使用资源的租户隔离目的，中心思想是我们不是直接给租户赋予相关权限，而是在租户之上创建了角色的维度，和`RBAC`思想一致，可以达到对租户进行灵活的扩缩容配额，最终来保障我们目前200+`clickhouse`节点的稳定性。最后笔者从2019年6月开始接触`clickhouse`，到现在也已经2年+的时间，但仍然只学到了`clickhouse`的冰山一角，所以以上文字也只能做到抛砖引玉，但仍然希望对阅读的各位有所帮助。