---
title: 按日切割使用docker安装的nginx容器
date: 2020-02-17 21:00:42
tags:
  - nginx
  - docker
  -  logrotate
categories:
  - linux
---
## 了解几个相关命令
+ 查看docker nginx容器 进程 pid
```
// 请务必确认您的docker container name是什么，笔者名称是nginx
// 命令1
sudo docker inspect --format='{{.State.Pid}}' $(sudo docker ps -a | grep nginx | awk '{print $1}')

//命令2是命令1的缩写
sudo docker inspect  -f '{{.State.Pid}}' $(sudo docker ps -a | grep nginx | awk '{print $1}')

//命令3是命令2的缩写
sudo docker inspect -f '{{ .State.Pid }}' nginx

// 命令4
sudo docker top $(sudo docker ps -a | grep nginx | awk '{print $1}') | grep master | awk '{print $2}'
```
+ 通知nginx重新打开日志文件
> Nginx 官方其实给出了如何滚动日志的说明：
Rotating Log-files
In order to rotate log files, they need to be renamed first. After that USR1 signal should be sent to the master process. The master process will then re-open all currently open log files and assign them an unprivileged user under which the worker processes are running, as an owner. After successful re-opening, the master process closes all open files and sends the message to worker process to ask them to re-open files. Worker processes also open new files and close old files right away. As a result, old files are almost immediately available for post processing, such as compression.
以上英文的大致意思是
1.先把旧的日志文件重命名
2.然后给 nginx master 进程发送 USR1 信号
3.nginx master 进程收到信号后会做一些处理，然后要求工作者进程重新打开日志文件
4.工作者进程打开新的日志文件并关闭旧的日志文件
```
kill -USR1 pid
```
+  logrotate使用帮助

  +  配置说明
  
| 参数 | 参数说明 |
| :------ | :------: |
| daily | 日志的轮替周期是每天 |
| weekly | 日志的轮替周期是每周 |
| monthly | 日志的轮替周期是每月 |
| rotate | 数字 保留的日志文件个数，0指没有备份 |
| compress | 当进行日志轮替时，对旧的日志进行压缩 |
| create | mode owner group 建立新日志，同时指定新日志的权限与所有者和所属组，如 create 0644 root root |
| copytruncate | 用于还在打开中的日志文件，把当前日志备份并截断 |
| mail | address 当进行日志轮替时，输出内容通过邮件发送到指定的邮件地址，如 mail test@gmail.com |
| missingok | 如果日志不存在，则忽略改日志的警告信息 |
| notifempty | 如果日志为空文件，则不进行日志轮替 |
| minsize | 大小 日志只有大于指定大小才进行日志轮替，而不是按照时间轮替，如 size 100k |
| dateext | 使用日期作为日志轮替文件的后缀，如 access.log-2019-07-12 |
| sharedscripts | 在此关键词之后的脚本只执行一次 |
| prerotate/endscript | 在日志轮替之前执行脚本命令。endscript 标识 prerotate 脚本结束 |
| postrotate/endscript | 在日志轮替之前执行脚本命令。endscript 标识 postrotate脚本结束 |


  +  参数说明



| 参数 | 参数说明 |
| :------ | :------: |
| -?或–help | 在线帮助 |
| -d或–debug | 详细显示指令执行过程，便于排错或了解程序执行的情况 |
| -f或–force | 强行启动记录文件维护操作，纵使logrotate指令认为没有需要亦然 |
| -s<状态文件>或–state=<状态文件> | 使用指定的状态文件 |
| -v或–version | 显示指令执行过程 |
| -usage | 显示指令基本用法 |
## 脚本详解
```
sudo vim /etc/ logrotate.d/nginx
//复制以下脚本，第一行是您的nginx log路径
/home/xx/docker/nginx/logs/*log {
        daily
        dateext
        missingok
        rotate 52
        compress
        delaycompress
        notifempty
        su hadoop hadoop
        create 0664 hadoop hadoop
        sharedscripts
        postrotate
            sudo docker inspect -f '{{ .State.Pid }}' nginx | xargs sudo kill -USR1
        endscript
}
sudo  logrotate -vf /etc/ logrotate.d/nginx
```
![loratate](/images/logrotate.png)

## crontab 加定时脚本
```
crontab -e
*/30 * * * * /usr/sbin/logrotate /etc/logrotate.d/nginx > /dev/null 2>&1 &
```

## 使用docker安装nginx
+ docker run(运行容器)的参数说明

| 参数 | 说明 |
| :------ | :------: |
| –-name | 容器名称 |
| –-link | 连接某个镜像 |
| –port | 端口映射 |
| –volum | 持久化存储 |

+ 获取nginx 镜像
```
sudo docker pull nginx
```
+ 运行nginx容器
```
// --rm标示删除旧容器 -d 标示daemon守护进程方式运行 --name标示容器名称
docker run --name my-custom-nginx-container -v /host/path/nginx.conf:/etc/nginx/nginx.conf:ro -d nginx
```
+ 拷贝宿主机的nginx配置到容器内，指定目录
```
docker cp nginx:/etc/nginx /etc/nginx
```
+ Dockerfile编写
```
FROM nginx
 COPY content /usr/share/nginx/html
 COPY conf /etc/nginx
 VOLUME /var/log/nginx/log
```
+ docker-compose.yml编写
```
web:
  image: nginx
  volumes:
   - ./mysite.template:/etc/nginx/conf.d/mysite.template
  ports:
   - "8080:80"
  environment:
   - NGINX_HOST=foobar.com
   - NGINX_PORT=80
  command: /bin/bash -c "envsubst < /etc/nginx/conf.d/mysite.template > /etc/nginx/conf.d/default.conf && exec nginx -g 'daemon off;'"
```
