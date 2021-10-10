---
title: wireshark3.4.8保姆级入门教程
date: 2021-10-02 10:09:55
tags:
  - wireshark
categories:
  - introduction
---

## wireshark3.4.8，主界面
![avatar](/images/wireshark/1.png)

## 通过ping简单入门
+ 启动wireshark后，wireshark处于抓包状态中
![avatar](/images/wireshark/2.png)
+ 在终端输入ping命令后
```
ping www.baidu.com
```
![avatar](/images/wireshark/3.png)
+ 通过过滤栏设置过滤条件进行数据包列表过滤
```
ip.addr == 180.101.49.12 and icmp
```
![avatar](/images/wireshark/4.png)

> ICMP（Internet Control Message Protocol）Internet控制报文协议。它是TCP/IP协议簇的一个子协议，用于在IP主机、路由器之间传递控制消息。控制消息是指网络通不通、主机是否可达、路由是否可用等网络本身的消息。这些控制消息虽然并不传输用户数据，但是对于用户数据的传递起着重要的作用。 ICMP协议是一种面向无连接的协议。

## wireshark抓包界面介绍
![avatar](/images/wireshark/5.png)
+ Display Filter(显示过滤器)，  用于设置过滤条件进行数据包列表过滤。菜单路径：Analyze --> Display Filters。
+  Packet List Pane(数据包列表)， 显示捕获到的数据包，每个数据包包含编号，时间戳，源地址，目标地址，协议，长度，以及数据包信息。 不同协议的数据包使用了不同的颜色区分显示
+ Packet Details Pane(数据包详细信息), 在数据包列表中选择指定数据包，在数据包详细信息中会显示数据包的所有详细信息内容。数据包详细信息面板是最重要的，用来查看协议中的每一个字段。各行信息分别为  
（1）Frame:   物理层的数据帧概况  
（2）Ethernet II: 数据链路层以太网帧头部信息  
（3）Internet Protocol Version 4: 互联网层IP包头部信息  
（4）Transmission Control Protocol:  传输层T的数据段头部信息，此处是TCP  
（5）Hypertext Transfer Protocol:  应用层的信息，此处是HTTP协议  
![avatar](/images/wireshark/6.png)

### TCP包的具体内容
![avatar](/images/wireshark/7.png)

### 抓包过滤器语法
+ 抓包过滤器语法和实例
> 抓包过滤器类型Type（host、net、port）、方向Dir（src、dst）、协议Proto（ether、ip、tcp、udp、http、icmp、ftp等）、逻辑运算符（&& 与、|| 或、！非）

（1）协议过滤

  比较简单，直接在抓包过滤框中直接输入协议名即可。

  TCP，只显示TCP协议的数据包列表

  HTTP，只查看HTTP协议的数据包列表

  ICMP，只显示ICMP协议的数据包列表

（2）IP过滤

  host 192.168.1.104

  src host 192.168.1.104

  dst host 192.168.1.104

（3）端口过滤

  port 80

  src port 80

  dst port 80

（4）逻辑运算符&& 与、|| 或、！非

  src host 192.168.1.104 && dst port 80 抓取主机地址为192.168.1.80、目的端口为80的数据包

  host 192.168.1.104 || host 192.168.1.102 抓取主机为192.168.1.104或者192.168.1.102的数据包

  ！broadcast 不抓取广播数据包

2、显示过滤器语法和实例

（1）比较操作符

  比较操作符有== 等于、！= 不等于、> 大于、< 小于、>= 大于等于、<=小于等于。

（2）协议过滤

  比较简单，直接在Filter框中直接输入协议名即可。注意：协议名称需要输入小写。

  tcp，只显示TCP协议的数据包列表

  http，只查看HTTP协议的数据包列表

  icmp，只显示ICMP协议的数据包列表  

（3） ip过滤

   ip.src ==192.168.1.104 显示源地址为192.168.1.104的数据包列表

   ip.dst==192.168.1.104, 显示目标地址为192.168.1.104的数据包列表

   ip.addr == 192.168.1.104 显示源IP地址或目标IP地址为192.168.1.104的数据包列表

（4）端口过滤

  tcp.port ==80,  显示源主机或者目的主机端口为80的数据包列表。

  tcp.srcport == 80,  只显示TCP协议的源主机端口为80的数据包列表。

  tcp.dstport == 80，只显示TCP协议的目的主机端口为80的数据包列表。  

（5） Http模式过滤

  http.request.method=="GET",   只显示HTTP GET方法的。

（6）逻辑运算符为 and/or/not

  过滤多个条件组合时，使用and/or。比如获取IP地址为192.168.1.104的ICMP数据包表达式为ip.addr == 192.168.1.104 and icmp

### wireshark抓包分析tcp3次握手
> 以某天气预报api举例,http://jisutianqi.market.alicloudapi.com/weather/query
+ 启动wireshark
+ 用浏览器访问[地址](http://jisutianqi.market.alicloudapi.com/weather/query)
+ 在终端输入，得到实际ip地址
```
ping jisutianqi.market.alicloudapi.com
```
![avatar](/images/wireshark/8.png)
+ 在wireshark上过滤
```
ip.addr == 47.97.242.71
```
![avatar](/images/wireshark/9.png)
> 图中可以看到wireshark截获到了三次握手的三个数据包。第四个包才是HTTP的， 这说明HTTP的确是使用TCP建立连接的。
+ 第一次握手抓包
客户端发送一个TCP，标志位为SYN，序列号为0， 代表客户端请求建立连接。 如下图。
![avatar](/images/wireshark/10.png)
数据包的关键属性如下：
  + SYN ：标志位，表示请求建立连接
  + Seq = 0 ：初始建立连接值为0，数据包的相对序列号从0开始，表示当前还没有发送数据
  + Ack =0：初始建立连接值为0，已经收到包的数量，表示当前没有接收到数据
+ 第二次握手抓包
服务器发回确认包, 标志位为 SYN,ACK. 将确认序号(Acknowledgement Number)设置为客户的I S N加1以.即0+1=1, 如下图
![avatar](/images/wireshark/11.png)
数据包的关键属性如下：
  + [SYN + ACK]: 标志位，同意建立连接，并回送SYN+ACK
  + Seq = 0 ：初始建立值为0，表示当前还没有发送数据
  + Ack = 1：表示当前端成功接收的数据位数，虽然客户端没有发送任何有效数据，确认号还是被加1，因为包含SYN或FIN标志位。（并不会对有效数据的计数产生影响，因为含有SYN或FIN标志位的包并不携带有效数据）
+ 第三次握手抓包
客户端再次发送确认包(ACK) SYN标志位为0,ACK标志位为1.并且把服务器发来ACK的序号字段+1,放在确定字段中发送给对方.并且在数据段放写ISN的+1, 如下图:
![avatar](/images/wireshark/12.png)
数据包的关键属性如下：
  + ACK ：标志位，表示已经收到记录
  + Seq = 1 ：表示当前已经发送1个数据
  + Ack = 1 : 表示当前端成功接收的数据位数，虽然服务端没有发送任何有效数据，确认号还是被加1，因为包含SYN或FIN标志位（并不会对有效数据的计数产生影响，因为含有SYN或FIN标志位的包并不携带有效数据)。

+ 在TCP层，有个FLAGS字段，这个字段有以下几个标识：SYN, FIN, ACK, PSH, RST, URG。如下
![avatar](/images/wireshark/13.png)
其中，对于我们日常的分析有用的就是前面的五个字段。它们的含义是：
  + SYN表示建立连接，
  + FIN表示关闭连接，
  + ACK表示响应，
  + PSH表示有DATA数据传输，
  + RST表示连接重置

