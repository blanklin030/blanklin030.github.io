---
title: docker ipv4 forwarding is disabled
date: 2020-09-21 17:00:15
tags:
  - docker
categories:
  - tool
---
> warning ipv4 forwarding is disabled. networking will not work.  

+ 起因
今天早上有个同学找过来问为什么xx平台502了，原谅笔者没有没有对该xx平台配置监控告警，原因是这个平台用户数太少。
+ 进入服务器查询
```
1. 查log
docker logs xx
2. 无报错log，查nginx》error.lo
connection timeout...
3. nginx提示连接超时了,ping服务
curl -i http://ip:port/api/xx/yy
4. 给了200的返回码，去外网再ping，
curl: (7) Failed to connect to ip port: Operation timed out
```
+ 结论，外网无法连通这个ip和端口
```
1. 重启docker container
sudo docker restart container_name
WARNING: IPv4 forwarding is disabled. Networking will not work.
```
+ 开启 ip 转发
```
sudo echo net.ipv4.ip_forward=1 >> /etc/sysctl.d/enable-ip-forward.conf
```
+ 重启 network 服务
```
sudo systemctl restart network
```
+ 查看 network 服务
```
sudo sysctl net.ipv4.ip_forward
```
> 返回为“net.ipv4.ip_forward = 1”则表示成功了
+ 重启容器
```
sudo docker restart container_name
```
+ 如果还提示上面的ipv4问题
```
sudo systemctl restart docker.service
```
+ 删除容器，再run
```
sudo docker rm -f xx
sudo docker run -d --name xx image_name
```