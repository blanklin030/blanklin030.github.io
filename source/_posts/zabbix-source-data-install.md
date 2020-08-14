---
title: 源码安装zabbix5.0
date: 2020-08-14 11:10:40
tags:
  - zabbix
categories:
  - monitor
---

+ zabbix介绍
> 通过[官网](https://www.zabbix.com/cn/whats_new_5_0)我们可以了解zabbix5.0和之前版本的差异及作出的改进

+ 下载zabbix5.0源码
```
yum -y install  epel-release wget tar
wget https://cdn.zabbix.com/zabbix/sources/stable/5.0/zabbix-5.0.2.tar.gz
tar zxvf zabbix-5.0.2.tar.gz
cd zabbix-5.0.2
```

+ 创建zabbix用户
```
groupadd zabbix
useradd  -g  zabbix zabbix
usermod  -s  /sbin/nologin  zabbix
```

+ mysql相关操作
> 安装mysql5.7请参考[地址](https://blanklin030.github.io/2020/05/13/rmp-install-mysql/)
  + 创建zabbix数据库并且授权给zabbix@123456访问
  ```
  create database zabbix character set utf8 collate utf8_bin;
  grant all on zabbix.* to zabbix@localhost identified by '123456';
  flush priviledges;
  ```
  + 导入zabbix数据到mysql
  ```
  mysql -uzabbix -p123456 zabbix < database/mysql/schema.sql
  mysql -uzabbix -p123456 zabbix < database/mysql/images.sql
  mysql -uzabbix -p123456 zabbix < database/mysql/data.sql
  ```
+ php7.2安装
> 安装php7.2请参考[地址](https://blanklin030.github.io/2020/08/11/php7.2-install/)

+ nginx安装
> 安装nginx请参考[地址](https://blanklin030.github.io/2020/08/11/nginx-install/)

+ 安装zabbix-server
```
./configure --prefix=/usr/local/zabbix  --enable-server --enable-agent --with-mysql --enable-ipv6 --with-net-snmp --with-libcurl --with-libxml2
make&&make install
chown zabbix:zabbix /usr/local/zabbix/ -R
```
+ 创建软链接,把zabbix命令设置为系统命令
```
ln -s /usr/local/zabbix/sbin/zabbix_*  /usr/local/sbin/
```
+ 配置zabbix启动脚本
```
# 在zabbix源码目录下
cp  misc/init.d/tru64/{zabbix_agentd,zabbix_server}  /etc/init.d/;chmod o+x /etc/init.d/zabbix_*
```

+ 配置zabbix-web
```
# 在zabbix源码目录下
cp -a  ui/* /var/www/html/zabbix/
chown -R nginx:nginx /var/www/html/zabbix
```
+ 配置nginx
```
server{
    listen         	80;
    server_name 	localhost;
    set    $host_path 	"/var/www/html/zabbix";
    root "$host_path";
    index index.html index.htm index.php;

    location ~ \.php$ {
        fastcgi_pass      127.0.0.1:9000;
        fastcgi_index     index.php;
        fastcgi_param     SCRIPT_FILENAME $document_root$fastcgi_script_name;
        include           fastcgi_params;
    }
}
```
+ 配置php
```
post_max_size = 16M
max_execution_time = 300
max_input_time = 300
date.timezone = Asia/Shanghai
```

+ 修改 zabbix server 配置文件
```
cd /usr/local/zabbix/etc
cp zabbix_server.conf zabbix_server.conf.bak
# 修改以下配置
LogFile=/tmp/zabbix_server.log
DBHost=localhost
DBName=zabbix
DBUser=zabbix
DBPassword=123456
```

+ 重启zabbix server和nginx和php-fpm
```
/etc/init.d/zabbix_server  restart
nginx -s reload
systemctl restart php
```

+ 访问zabbix web gui进行安装配置
![zabbix-web-1](/images/zabbix-web-1.png)
![zabbix-web-2](/images/zabbix-web-2.png)
![zabbix-web-3](/images/zabbix-web-3.png)

+ 源码部署zabbix agent
```
yum -y install curl curl-devel net-snmp net-snmp-devel perl-DBI
groupadd zabbix
useradd -g zabbix zabbix
usermod -s /sbin/nologin zabbix
tar -xzf zabbix-5.0.2.tar.gz
cd zabbix-5.0.2
./configure  --prefix=/usr/local/zabbix  --enable-agent
make install
```
+ 创建软链接
```
ln -s /usr/local/zabbix/sbin/zabbix_* /usr/local/sbin/
```
+ 配置启动脚本
```
cd zabbix-5.0.2
cp  misc/init.d/tru64/zabbix_agentd  /etc/init.d/zabbix_agentd
chmod o+x /etc/init.d/zabbix_agentd
```

+ 修改配置
```
vim /usr/local/zabbix/etc/zabbix_agentd.conf
LogFile=/tmp/zabbix_agentd.log
Server=192.168.2.214  #server端ip
ServerActive=192.168.2.214 #server端ip
Hostname = 192.168.2.215  #agent端ip或者是主机名都可以
```

+ 启动agent
```
/etc/init.d/zabbix_agentd
```
+ 自动发现agent
![zabbix-web-5](/images/zabbix-web-5.png)