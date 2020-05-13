---
title: grafana迁移sqlite3到mysql
date: 2020-05-12 20:01:10
tags:
  - grafana
categories:
  - tools
---

+ grafana版本
生产环境：Grafana v6.6.2
本地docker安装相同版本：
```
sudo docker run -d --name grafana-662-1 \
-p 3000:3000 \
-e GF_DATABASE_TYPE=mysql   \
-e GF_DATABASE_HOST=ip:port   \
-e GF_DATABASE_NAME=grafana   \
-e GF_DATABASE_USER=grafana   \
-e GF_DATABASE_PASSWORD=mysql   \
-e GF_DATABASE_URL=mysql://grafana:pwd@ip:port/grafana   \
-v /data1/docker/grafana:/var/lib/grafana   \
grafana/grafana:6.6.2 
```

> -e GF_xx，这个xx对应的是conf/grafana.ini(默认是default.ini)的[xx]

![grafana-mysql](/images/grafana-mysql.png)

> 以上是用docker方式配置mysql启动grafana

+ 确认mysql的35个表已经创建成功，暂停grafana
```
sudo docker stop $(sudo docker ps -a | grep grafana-662-1 | awk '{print $1}')
```

+ 从本地mysql数据库导出表结构到生产环境
```
mysql -uroot -p -D grafana > grafana.sql
```
+ 生产环境导入表结构
```
CREATE DATABASE grafana CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;
mysql -uroot -p -D grafana < grafana.sql
```
+ 另存脚本sqlitedump.sh
```
#!/bin/bash
DB=$1
TABLES=$(sqlite3 $DB .tables | sed -r 's/(\S+)\s+(\S)/\1\n\2/g' | grep -v migration_log)
for t in $TABLES; do
    echo "TRUNCATE TABLE $t;"
done
for t in $TABLES; do
    echo -e ".mode insert $t\nselect * from $t;"
done | sqlite3 $DB
```
+ 找到grafana.db文件
```
cd /data1/docker/grafana
sh sqlitedump.sh grafana.db > insert.sql
```
+ 生产环境导入insert.sql
```
mysql -uroot -p -D grafana < 生产环境导入insert.sql
``` 
+ 启动grafana的docker
```
sudo docker start $(sudo docker ps -a | grep grafana-662-1 | awk '{print $1}')
```
+ 验证
