---
title: centos7源码安装nginx
date: 2020-08-11 14:10:40
tags:
  - nginx
categories:
  - web server
---

### yum安装过程
+ 安装nginx最新源
```
yum localinstall http://nginx.org/packages/centos/7/noarch/RPMS/nginx-release-centos-7-0.el7.ngx.noarch.rpm
```
+ 安装nginx
```
yum -y install nginx
```
+ 启动nginx
```
yum -y install nginx
```
+ 设置nginx开机启动
```
systemctl enable nginx.service
# 检查是否成功
systemctl list-dependencies | grep nginx
```
+ 通过 egrep 查询 nginx 服务器的用户和用户组
```
egrep '^(user|group)' /etc/nginx/nginx.conf
```
  + 结果示例：
  ```
  user nginx;
  ```

+ 路径整理
```
# nginx 配置文件
/etc/nginx/nginx.conf

# nginx 默认项目路径
/usr/share/nginx/html
```