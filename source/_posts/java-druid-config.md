---
title: alibaba.druid配置
date: 2022-03-21 15:09:55
tags:
  - java
  - druid
categories:
  - sre
---

#### 两种思路
+ 后端服务主动淘汰空闲超时120s的连接，然后再主动创建连接。
+ 后端服务对空闲连接保活，周期小于120秒进行一次mysql.ping或select 1操作。（确认过dba两个操作均可使dbproxy保活连接）

#### hikari连接池能否支持上述方案：
+ idleTimeout：设置最大空闲时间小于120秒，空闲超时主动淘汰链接，并且hikari会自动创建新连接填充。实际发现该参数不能解决此问题，因为该参数优先级低于minimumIdle，若池中已经保持低于minimumIdle的连接数就不会再进行空闲连接淘汰。所以该参数不能解决问题。
+ maxLifetime：设置最大生存时间小于120秒，生存超时主动淘汰连接，并且hikari会自动创建新连接填充。能够解决该问题，但是会每120s就要淘汰且新建连接。

#### druid连接池能否支持上述方案：
+ testWhileIdle：设置true开启周期性健康检查 会主动淘汰掉失效的连接。
+ maxEvictableIdleTimeMillis：最大空闲时间设置小于120s，该参数不受minIdle参数约束，只要超过最大空闲时间就会自动淘汰连接。
看似上面两个参数均能解决，但是由于durid默认不会主动创建新链接，所以上面两个参数使连接清空后 查询时仍旧会因创建连接而耗时增加。
+ keepAlive：设置true开启保活机制，可解决该问题

```
druid:
# 初始化大小
initialSize: 10
# 最大连接数
maxActive: 30
# 最小空闲数（周期性清除时保留的最小连接数）
minIdle: 10
# 最大空闲数（返还连接时若超过最大空闲连接数就丢弃）
maxIdle: 20
# 有效性校验
validationQuery: select 1
# 周期性check空闲连接
testWhileIdle: true
# 借出时check（影响性能）
testOnBorrow: false
# 返还时check（影响性能）
testOnReturn: false


# 周期性检查的间隔时间，单位是毫秒
timeBetweenEvictionRunsMillis: 60000
# 连接的最小空闲时间，小于该时间不会被周期性check清除。单位是毫秒
minEvictableIdleTimeMillis: 50000
# 连接的最大空闲时间，大于该时间会被周期性check清除，且优先级大于minIdle。单位是毫秒
maxEvictableIdleTimeMillis: 100000
# 开启连接保活策略。作用：https://www.bookstack.cn/read/Druid/d90f9643acdca5c0.md
keepAlive: true
```