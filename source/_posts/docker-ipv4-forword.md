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
+ docker network bridge
![docker-bridge](/images/docker-bridge.png)
> 由于宿主机的IP地址与容器veth pair的 IP地址均不在同一个网段，故仅仅依靠veth pair和namespace的技术，还不足以使宿主机以外的网络主动发现容器的存在。为了使外界可以访问容器中的进程，docker采用了端口绑定的方式，也就是通过iptables的NAT，将宿主机上的端口流量转发到容器内的端口上。 
举例说明 
```
1. 创建容器，并将宿主机的3306端口绑定到容器的3306端口
docker run -tid --name db -p 3306:3306 MySQL
2. 宿主机上，可以通过iptables -t nat -L -n，查到一条DNAT规则
DNAT tcp -- 0.0.0.0/0 0.0.0.0/0 tcp dpt:3306 to:172.17.0.5:3306
3. 以上的172.17.0.5即为bridge模式下，创建的容器IP
```

+ 开启 ip 转发
```
sudo echo net.ipv4.ip_forward=1 >> /etc/sysctl.d/enable-ip-forward.conf
或者
sudo echo 1 > /proc/sys/net/ipv4/ip_forward
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