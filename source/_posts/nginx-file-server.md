---
title: 使用nginx搭建加密的文件服务器
date: 2020-02-17 20:51:10
tags:
  - nginx
categories:
  - tools
---

+ 安装epel
EPEL 仓库中有 Nginx 的安装包。如果你还没有安装过 EPEL，可以通过运行下面的命令来完成安装：
```
sudo yum install epel-release
```
+ 安装nginx
```
sudo yum install nginx
```
+ 配置文件
  + 日志文件
    ```
    /var/log/nginx/access.log
     /var/log/nginx/error.log
    ```
  + conf文件
    ```
    /etc/nginx/conf.d/
    ```
  + 项目文件
    ```
    /var/www/html/<site_name>
    /usr/share/nginx/html/<site_name>
    ```
  + nginx命令
    ```
    /usr/sbin/nginx
    ```
+ 启动nginx
```
/usr/sbin/nginx
```
+ 生成密钥
```
# 使用openssl生成密钥
printf "admin:$(openssl passwd -crypt 123456)\n" >> /etc/nginx/conf/htpasswd
cat /etc/nginx/conf/htpasswd
admin:xyJkVhXGAZ8tM
# 使用htpasswd生成密钥
yum install httpd-tools -y
htpasswd -c -d /etc/nginx/conf/htpasswd admin
```
+ 添加server配置
```
server{
  listen       80;
  server_name  localhost;

  auth_basic   "登录认证";
  auth_basic_user_file /etc/nginx/conf/htpasswd;

  autoindex on;
  autoindex_exact_size on; #显示文件大小
  autoindex_localtime on; #显示文件时间

  root   /var/www/html;
  index  index.html index.php
}
```
+ 重启nginx
```
/usr/sbin/nginx -s reload
```
![nginx-file-server](/images/nginx-file-server.png)