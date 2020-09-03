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
### 源码安装
+ 安装系统依赖
```
yum install epel-release -y
yum install autoconf libtool re2c bison libxml2-devel bzip2-devel libcurl-devel libpng-devel libicu-devel gcc-c++ libmcrypt-devel libwebp-devel libjpeg-devel openssl-devel -y
```
+ 下载源码
```
curl -O -L https://github.com/php/php-src/archive/php-7.2.3.tar.gz
tar -zxvf php-7.2.3.tar.gz
cd php-src-php-7.2.3
```
+ 编译安装
```
./buildconf --force
./configure --prefix=/usr/local/php --enable-fpm --disable-short-tags --with-openssl --with-pcre-regex --with-pcre-jit --with-zlib --enable-bcmath --with-bz2 --enable-calendar --with-curl --enable-exif --with-gd --enable-intl --enable-mbstring --with-mysqli --enable-pcntl --with-pdo-mysql --enable-soap --enable-sockets --with-xmlrpc --enable-zip --with-webp-dir --with-jpeg-dir --with-png-dir
make clean
make
make install
```
+ php-fpm setup
```
cp /usr/local/php/etc/php-fpm.conf.default /usr/local/php/etc/php-fpm.conf
cp /usr/local/php/etc/php-fpm.d/www.conf.default /usr/local/php/etc/php-fpm.d/www.conf
cp php.ini-development /usr/local/php/etc/php.ini
```
+ 修改配置
```
vim /usr/local/php/etc/php.ini
expose_php = Off
short_open_tag = ON
max_execution_time = 300
max_input_time = 300
memory_limit = 128M
post_max_size = 32M
date.timezone = Asia/Shanghai
mbstring.func_overload=2
```

```
vim /usr/local/php/etc/php-fpm.d/www.conf
listen = /var/run/www/php-cgi.sock
listen.owner = www
listen.group = www
listen.mode = 0660
listen.allowed_clients = 127.0.0.1
pm = dynamic
listen.backlog = -1
pm.max_children = 180
pm.start_servers = 50
pm.min_spare_servers = 50
pm.max_spare_servers = 180
request_terminate_timeout = 120
request_slowlog_timeout = 50
slowlog = var/log/slow.log
```

```
vim /usr/local/php/etc/php-fpm.conf
pid = /usr/local/php/var/run/php-fpm.pid
```
+ php加入系统环境
```
vim /etc/bashrc
export PHP_PATH=/usr/local/php
export PATH=$PHP_PATH/bin:$PATH
```
+ 启动php-fpm
```
/usr/local/php/sbin/php-fpm
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