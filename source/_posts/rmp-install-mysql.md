---
title: centos7下安装mysql5.7
date: 2020-05-13 10:09:55
tags:
  - mysql
categories:
  - tool
---
+ 卸载旧rpm包
```
rpm -qa |grep -i mysql
MySQL-client-5.6.23-1.sles11.x86_64
MySQL-server-5.6.23-1.sles11.x86_64
MySQL-shared-5.6.23-1.sles11.x86_64
MySQL-devel-5.6.23-1.sles11.x86_64
————————————————
// 逐个卸载
rpm -e --nodeps MySQL-client-5.6.23-1.sles11.x86_64
rpm -e --nodeps MySQL-server-5.6.23-1.sles11.x86_64
rpm -e --nodeps MySQL-shared-5.6.23-1.sles11.x86_64
rpm -e --nodeps MySQL-devel-5.6.23-1.sles11.x86_64
// 确认卸载完成
rpm -qa |grep -i mysql
//my.cnf也可以删了
rm -fr /etc/my.cnf
//为空
```
+ 下载rpm包
[地址1](http://repo.mysql.com/)
[地址2](https://centos.pkgs.org/7/mysql-5.7-x86_64/mysql-community-server-5.7.20-1.el7.x86_64.rpm.html)
```
wget http://repo.mysql.com/mysql57-community-release-fc27.rpm
```
+ 安装rpm包
```
sudo rpm -ivh mysql57-community-release-fc27.rpm
```
+ 查看yum repo源
> 会获得两个mysql的yum repo源：/etc/yum.repos.d/mysql-community.repo，/etc/yum.repos.d/mysql-community-source.repo
+ install mysql
```
sudo yum install mysql-server
```
+ 查看mysql安装
```
rpm -qa | grep mysql
```
> rpm -qa | grep mysql-community
mysql-community-server-5.7.27-2.el7.x86_64
mysql-community-release-el7-5.noarch
mysql-community-libs-5.7.27-2.el7.x86_64
mysql-community-common-57.27-2.el7.x86_64
mysql-community-devel-5.7.27-2.el7.x86_64
mysql-community-client-5.7.27-2.el7.x86_64
+ 初始化mysql
```
mysql_install_db
```
+ 修改my.cnf
```
vim /etc/my.cnf
// 在下面制定启动用户是mysql
[mysqld]
user=mysql
```
+ 启动mysqld
```
mysqld
// 若为做上一步，则启动时--user=mysql
```
+ 修改root用户密码
```
mysql -uroot -p
// 首次登陆密码是空的，直接进入
use mysql
update user set password=password('new password') where user='root';
flush privileges;
exit;
```
+ 创建一个远程访问用户
```
//创建用户
create user 'somebody' identified by 'password’;
//给用户授权
Grant ALL PRIVILEGES on *.* to somebody@‘%’;
//刷新
flush privileges;
```
