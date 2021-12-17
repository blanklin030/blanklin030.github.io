---
title: mysql锁相关
date: 2021-12-17 12:39:59
tags:
  - mysql
  - lock
categories:
  - database
---
### 背景
`SpringBoot`的`schedule`模块可以支持定时脚本，原理其实就是`SchedulingTaskExecutor`类，它实现了`java.util.concurrent.Executor`接口，这个接口主要是定义了线程的执行，例如我们日常常用的线程池执行器`ThreadPoolExecutor`类就是实现了`Executor`接口。此文重点不是介绍`SpringBoot`的`schedule`模块，所以具体实现逻辑及源码部分解析，在此略过。但问题是`schedule`模块不支持分布式部署，而我们当前的业务需要部署在多个节点上，为了实现多个节点上在某个时刻只执行某个定时脚本，其他节点不重复执行，我们调研了`MYSQL`的锁，用以实现分布式锁场景。

### mysql锁

#### 乐观锁
+ 什么是乐观锁
> 用数据版本（Version）记录机制实现，这是乐观锁最常用的一种实现方式。何谓数据版本？即为数据增加一个版本标识，一般是通过为数据库表增加一个数字类型的 “version” 字段来实现。当读取数据时，将version字段的值一同读出，数据每更新一次，对此version值加1。当我们提交更新的时候，判断数据库表对应记录的当前版本信息与第一次取出来的version值进行比对，如果数据库表当前版本号与第一次取出来的version值相等，则予以更新，否则认为是过期数据。
+ 实现过程
假设表结构如下
```
CREATE TABLE `lock` (
  `id` bigint(20) unsigned NOT NULL AUTO_INCREMENT,
  `name` varchar(255) NOT NULL DEFAULT '0' COMMENT '锁名称',
  `status` tinyint(4) NOT NULL DEFAULT '0' COMMENT '锁状态：0-空闲，1-运行',
  `version` bigint(20) NOT NULL DEFAULT '0' COMMENT '版本',
  PRIMARY KEY (`id`)
) ENGINE=InnoDB AUTO_INCREMENT=1 DEFAULT CHARSET=utf8 COLLATE=utf8_bin;
```
查询方式：
```
select id,name,version from lock where id = #{id}
```
加锁更新方式：
```
update lock set status = 1，version=version+1 where id = #{id} and version = #{version}
```
释放锁更新方式：
```
update lock set status = 0 where id = #{id} and status = 1
```
#### 悲观锁
+ 什么是悲观锁
> 与乐观锁相对应的就是悲观锁了。悲观锁就是在操作数据时，认为此操作会出现数据冲突，所以在进行每次操作时都要通过获取锁才能进行对相同数据的操作，这点跟`java`中的`synchronized`很相似，所以悲观锁需要耗费较多的时间。另外与乐观锁相对应的，悲观锁是由数据库自己实现了的，要用的时候，我们直接调用数据库的相关语句就可以了。
说到这里，由悲观锁涉及到的另外两个锁概念就出来了，它们就是共享锁与排它锁。共享锁和排它锁是悲观锁的不同的实现，它俩都属于悲观锁的范畴。

+ 什么是共享锁
> 共享锁又称读锁 (read lock)，是读取操作创建的锁。其他用户可以并发读取数据，但任何事务都不能对数据进行修改（获取数据上的排他锁），直到已释放所有共享锁。当如果事务对读锁进行修改操作，很可能会造成死锁。

在查询语句后面增加 `LOCK IN SHARE MODE` ，`Mysql`会对查询结果中的每行都加共享锁，当没有其他线程对查询结果集中的任何一行使用排他锁时，可以成功申请共享锁，否则会被阻塞。 其他线程也可以读取使用了共享锁的表，而且这些线程读取的是同一个版本的数据。
加上共享锁后，对于`update`，`insert`，`delete`语句会自动加排它锁。

举例说明
```
# 在A窗口输入
select * from lock where id = 1 lock in shard mode
# 在B窗口输入
update lock set version = version + 1 where id = 1
# B窗口报错
[Err] 1205 - Lock wait timeout exceeded; try restarting transaction
```

+ 什么是排它锁
> 排他锁 exclusive lock（也叫writer lock）又称写锁。
若某个事物对某一行加上了排他锁，只能这个事务对其进行读写，在此事务结束之前，其他事务不能对其进行加任何锁，其他进程可以读取,不能进行写操作，需等待其释放。排它锁是悲观锁的一种实现，在上面悲观锁也介绍过。
若事务 1 对数据对象A加上X锁，事务 1 可以读A也可以修改A，其他事务不能再对A加任何锁，直到事物 1 释放A上的锁。这保证了其他事务在事物 1 释放A上的锁之前不能再读取和修改A。排它锁会阻塞所有的排它锁和共享锁

举例说明
```
# 要使用排他锁，我们必须关闭mysql数据库的自动提交属性，因为MySQL默认使用autocommit模式，也就是说，当你执行一个更新操作后，MySQL会立刻将结果进行提交
# 在A窗口输入
set autocommit = 0;
begin;
select * from lock where id = 1 for update;
update lock set version = version + 1 where id = 1;
commit;

# 在B窗口输入，会看到一直在等待中,直到A窗口释放锁,B窗口才能获取结果
select * from lock where id = 1 for update;
```

#### 行锁
`InnoDB`的行锁是针对索引加的锁，不是针对记录加的锁。并且该索引不能失效，否则都会从行锁升级为表锁。
行锁的劣势：开销大；加锁慢；会出现死锁
行锁的优势：锁的粒度小，发生锁冲突的概率低；处理并发的能力强
加锁的方式：自动加锁。对于`UPDATE`、`DELETE`和`INSERT`语句，`InnoDB`会自动给涉及数据集加排他锁；对于普通`SELECT`语句，`InnoDB`不会加任何锁；当然我们也可以显示的加锁

#### 间隙锁
当我们用范围条件而不是相等条件检索数据，并请求共享或排他锁时，`InnoDB`会给符合条件的已有数据记录的索引项加锁；对于键值在条件范围内但并不存在的记录，叫做“间隙（GAP)”，`InnoDB`也会对这个“间隙”加锁，这种锁机制就是所谓的间隙锁（`Next-Key`锁）
例如

```
# 是一个范围条件的检索，InnoDB不仅会对符合条件的empid值为101的记录加锁，也会对empid大于101（这些记录并不存在）的“间隙”加锁
# InnoDB使用间隙锁的目的，一方面是为了防止幻读，以满足相关隔离级别的要求
# 指同一个事务内多次查询返回的结果集不一样。比如同一个事务 A 第一次查询时候有 n 条记录，但是第二次同等条件下查询却有 n+1 条记录，这就好像产生了幻觉。发生幻读的原因也是另外一个事务新增或者删除或者修改了第一个事务结果集里面的数据，同一个记录的数据内容被修改了，所有数据行的记录就变多或者变少了
Select * from  emp where empid > 100 for update;
```

#### 表锁
在`Innodb`引擎中既支持行锁也支持表锁，那么什么时候会锁住整张表，什么时候只锁住一行呢？ 只有通过**索引条件**检索数据，`InnoDB`才使用行级锁，否则，`InnoDB`将使用表锁，而检索条件是`unique key`、`primary key`时，一定会是**行锁**，而检索条件是`index`时，有可能是行锁 ，也有可能是表锁，取决于当“值重复率”低时，甚至接近主键或者唯一索引的效果，“普通索引”依然是行锁；当“值重复率”高时，`MySQL` 不会把这个“普通索引”当做索引，即造成了一个没有索引的 SQL，此时引发表锁

#### 死锁
死锁（Deadlock）是指两个或两个以上的进程在执行过程中，因争夺资源而造成的一种互相等待的现象，若无外力作用，它们都将无法推进下去。此时称系统处于死锁状态或系统产生了死锁，这些永远在互相等待的进程称为死锁进程。由于资源占用是互斥的，当某个进程提出申请资源后，使得有关进程在无外力协助下，永远分配不到必需的资源而无法继续运行，这就产生了一种特殊现象死锁。
解除正在死锁的状态有两种方法：
第一种：
```
# 查询是否锁表
show OPEN TABLES where In_use > 0;
# 查询进程（如果您有SUPER权限，您可以看到所有线程。否则，您只能看到您自己的线程）
show processlist
# 杀死进程id（就是上面命令的id列）
kill id
```

第二种：
```
# 查看当前的事务
SELECT * FROM INFORMATION_SCHEMA.INNODB_TRX;
# 查看当前锁定的事务
SELECT * FROM INFORMATION_SCHEMA.INNODB_LOCKS;
# 查看当前等锁的事务
SELECT * FROM INFORMATION_SCHEMA.INNODB_LOCK_WAITS;
# 杀死进程
kill 进程ID
```
### MYISAM引擎 和 INNODB引擎的区别
#### MYISAM 读锁
读锁 影响其他进程对该表进行写操作，但不影响其他进程对该表进行读操作
```
lock myisam_table read;
select * from myisam_table where id = 1;
UNLOCK TABLES;
```
#### MYISAM 写锁
写锁 影响其他进程对该表进行读和写操作
```
lock myisam_table write;
select * from myisam_table where id = 1;
UNLOCK TABLES;
```
在自动加锁的情况下也基本如此，`MyISAM` 总是一次获得 `SQL` 语句所需要的全部锁。这也正是 `MyISAM` 表不会出现死锁(`Deadlock Free`)的原因
+ `InnoDB`支持事务(`transaction`)；`MyISAM`不支持事务
+ `Innodb` 默认采用行锁， `MyISAM` 是默认采用表锁。加锁可以保证事务的一致性，可谓是有人(锁)的地方，就有江湖(事务)
+ `MyISAM`不适合高并发（MyISAM 在执行查询语句(SELECT)前,会自动给涉及的所有表加读锁,在执行更新操作 (UPDATE、DELETE、INSERT 等)前，会自动给涉及的表加写锁）
> MyISAM存储引擎有一个系统变量concurrent_insert,专门用以控制其并发插入的行为,其值分别可以为0、1或2。
```
当concurrent_insert设置为0时,不允许并发插入。
当concurrent_insert设置为1时,如果MyISAM表中没有空洞(即表的中间没有被删除的 行),MyISAM允许在一个进程读表的同时,另一个进程从表尾插入记录。这也是MySQL 的默认设置。
当concurrent_insert设置为2时,无论MyISAM表中有没有空洞,都允许在表尾并发插入记录
```

### 解决方法
我们采用**乐观锁** 来处理这次的定时任务多节点执行时分布式锁方案
+ 表结构设计
```
CREATE TABLE `job_lock` (
  `id` bigint(20) unsigned NOT NULL AUTO_INCREMENT,
  `name` varchar(255) NOT NULL DEFAULT '0' COMMENT 'job名称',
  `timeout` bigint(20) NOT NULL DEFAULT '0' COMMENT '任务执行超时间隔,毫秒',
  `status` tinyint(4) NOT NULL DEFAULT '0' COMMENT 'job状态：0-空闲，1-运行',
  `description` varchar(255) NOT NULL DEFAULT '' COMMENT 'job描述',
  `gmt_create` timestamp NOT NULL DEFAULT CURRENT_TIMESTAMP COMMENT '创建时间',
  `gmt_update` timestamp NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP COMMENT '更新时间',
  `version` bigint(20) NOT NULL DEFAULT '0' COMMENT '版本',
  PRIMARY KEY (`id`)
) ENGINE=InnoDB AUTO_INCREMENT=1 DEFAULT CHARSET=utf8 COLLATE=utf8_bin;
```

+ 加锁方法
```
<update id="requireLock" parameterType="java.util.Map">
<![CDATA[
    update job_lock
    set status = 1, version=version + 1
    where id = #{id} and version =#{version} and status = 0
]]>
</update>
```

+ 解锁方法
```
<update id="releaseLock" parameterType="java.util.Map">
<![CDATA[
    update job_lock
    set status = 0
    where id = #{id} and status = 1
]]>
</update>
```

+ 尝试加锁
```
public JobLockDO tryLock(String name) {
  if (ValidateUtils.isNull(name)){
    return null;
  }
  JobLockDO jobLocksDO = getJob(name);
  if (ValidateUtils.isNull(jobLocksDO)) {
    return null;
  }
  // 任务一直在运行中，可能是服务重启等异常情况，造成锁状态一直未更新
  if (jobLocksDO.getStatus().equals(Constant.JOB_LOCK_RUNNING)) {
    // 先判断运行是否超时，未超时，则不处理
    if (System.currentTimeMillis() - jobLocksDO.getGmtUpdate().getTime() <= jobLocksDO.getTimeout()) {
      return null;
    } else {
      // 已超时，更新任务锁状态(释放锁)
      releaseLock(jobLocksDO.getId());
      // 重新加锁
      requireLock(jobLocksDO.getId(), jobLocksDO.getVersion());
      // 返回任务锁
      return jobLocksDO;
    }
  }
  try {
    // 加锁
    if (!requireLock(jobLocksDO.getId(), jobLocksDO.getVersion())) {
      return null;
    }
    return jobLocksDO;
  } catch (Exception e) {
    LOGGER.error("require lock by name:{} fail.", name, e);
  }
  return null;
}
```