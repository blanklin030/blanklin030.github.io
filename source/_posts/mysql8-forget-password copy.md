---
title: mysql8忘记root密码
date: 2020-09-03 22:12:55
tags:
  - mysql
categories:
  - tool
---
+ 关闭mysqld
```
ps aux | grep mysqld | grep -v grep | awk '{print $2}' | xargs kill -9

```
+ 修改my.conf
```
vim /etc/my.conf
# 找到[mysqld]这个section,添加skip-grant-tables,意思是忽略所有表的授权
skip-grant-tables
```
+ 开启mysqld服务
```
mysqld &
```
+ 进入mysql，设置密码为空
```
mysql -u root
mysql> use mysql;
mysql> update user set authentication_string='' where user='root';
mysql> flush privileges;
mysql> exit;
```
+ 撤回skip-grant-tables
```
vim /etc/my.conf
# 找到[mysqld]这个section,注释掉skip-grant-tables
```
+ 重启mysqld
```
ps aux | grep mysqld | grep -v grep | awk '{print $2}' | xargs kill -9
mysqld &
```
+ 无密码登录mysql
```
mysql -uroot -p

mysql> use mysql;
mysql> ALTER user 'root'@'localhost' IDENTIFIED BY 'Qian123#';
mysql> flush privileges;
mysql> exit;
```
