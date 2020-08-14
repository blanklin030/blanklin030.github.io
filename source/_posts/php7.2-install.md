---
title: centos7源码安装php7.2
date: 2020-08-11 14:10:40
tags:
  - php7.2
categories:
  - web server
---

### yum安装过程
+ 安装 EPEL 软件包
```
yum install epel-release -y
```
+ 安装 remi 源
```
yum -y install http://rpms.remirepo.net/enterprise/remi-release-7.rpm
```
+ 安装 yum 扩展包
```
yum -y install yum-utils
```
+ 启用 remi 仓库
```
yum-config-manager --enable remi-php72
yum update
```
+ 安装php7.2
```
yum install -y php72-php-fpm php72-php-gd php72-php-json php72-php-mbstring php72-php-mysqlnd php72-php-xml php72-php-xmlrpc php72-php-opcache
```
+ 设置开机自启
```
systemctl enable php72-php-fpm.service
```
+ 修改执行 php-fpm 的权限
```
vi /etc/opt/remi/php72/php-fpm.d/www.conf
# 设置用户和用户组为 nginx
user = nginx
group = nginx
```
+ 路径整理
```
/etc/opt/remi/php72
```

> php 5.3.3 以后的php-fpm 不再支持 php-fpm 以前具有的 /usr/local/php/sbin/php-fpm (start|stop|reload)等命令，所以不要再看这种老掉牙的命令了

+ 使用信号控制
```
INT, TERM 立刻终止
QUIT 平滑终止
USR1 重新打开日志文件
USR2 平滑重载所有worker进程并重新载入配置和二进制模块
```

+ 查看php-fpm的master进程
```
ps aux | grep php-fpm | grep -v grep 

```
![php-fpm-process](/images/php-fpm-process.png)
 + 重启php-fpm
```
# 最后的一串数字就是master进程的id
kill -USR2 44551
```