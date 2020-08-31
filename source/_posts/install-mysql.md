---
title: centos7下安装mysql5.7
date: 2020-05-13 10:09:55
tags:
  - mysql
categories:
  - tool
---
## 使用rpm包方式安装mysql
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

## 使用源码安装mysql
+ 获取源码安装包
```
wget  https://dev.mysql.com/get/Downloads/MySQL-5.7/mysql-boost-5.7.20.tar.gz
```

+ 安装系统所需依赖
```
yum install -y cmake gcc-c++ gcc ncurses-devel perl-Data-Dumper boost boost-doc boost-devel
```
+ 创建mysql用户
```
useradd mysql  -s /sbin/nologin
mkdir -pv /usr/local/mysql/mydata
mkdir -pv /usr/local/mysql/conf
chown -R mysql:mysql /usr/local/mysql
```
+ 删除mariadbd的my.cnf文件
```
rm -fr /etc/my.cnf
```
+ 解压源码包并安装
```
tar zxvf mysql-boost-5.7.20.tar.gz

cmake \
-DCMAKE_INSTALL_PREFIX=/usr/local/mysql \
-DMYSQL_DATADIR=/usr/local/mysql/mydata \
-DSYSCONFDIR=/usr/local/mysql/conf \
-DMYSQL_USER=mysql \
-DWITH_MYISAM_STORAGE_ENGINE=1 \
-DWITH_INNOBASE_STORAGE_ENGINE=1 \
-DMYSQL_UNIX_ADDR=/usr/local/mysql/mysql.sock \
-DMYSQL_TCP_PORT=3306 \
-DEXTRA_CHARSETS=all \
-DDEFAULT_CHARSET=utf8 \
-DDEFAULT_COLLATION=utf8_general_ci \
-DWITH_DEBUG=0 \
-DMYSQL_MAINTAINER_MODE=0 \
-DWITH_SSL:STRING=bundled \
-DWITH_ZLIB:STRING=bundled \
-DDOWNLOAD_BOOST=1 \
-DWITH_BOOST=./boost

make && make install
```
+ 创建my.cnf
```
vim /etc/my.cnf
# 复制下面内容
[mysqld]
datadir=/usr/local/mysql/mydata  
socket=/tmp/mysql.sock 
```
+ 把mysql添加到系统服务并设置开机启动
```
cp /usr/local/mysql/support-files/mysql.server /etc/init.d/mysqld
chmod +x /etc/init.d/mysqld
chkconfig --add mysqld
chkconfig mysqld on
cp /usr/local/mysql/bin/ /usr/bin/
```

+ 初始化mysql
```
//version <5.7
mysql_install_db
//version >=5.7
mysqld --initialize --user=mysql --basedir=/usr/local/mysql --datadir=/usr/local/mysql/mydata
```
+ 修改my.cnf
```
vim /etc/my.cnf
// 在下面制定启动用户是mysql
[mysqld]
user=mysql
datadir=/usr/local/mysql/mydata      
socket=/tmp/mysql.sock 
```
+ 启动mysqld
```
mysqld
// 若未做上一步，则启动时--user=mysql
# 或者
service mysqld start
```

+ 新建软链接
```
ln -s /tmp/mysql.sock /usr/local/mysql/mysql.sock
service mysqld restart
```

+ 修改mysq.cnf
```
[mysqld]
user=mysql
datadir=/usr/local/mysql/mydata      
socket=/usr/local/mysql/mysql.sock 
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
