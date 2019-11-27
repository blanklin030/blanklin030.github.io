---
title: 聊聊cgi的二三事
date: 2019-11-07 14:09:08
tags:
  - nginx
  - cgi
  - fastcgi
categories:
  - web
---

## CGI

**通用网关接口**（Common Gateway Interface/CGI）描述了客户端和服务器程序之间传输数据的一种标准，可以让一个客户端，从网页浏览器向执行在网络服务器上的程序请求数据。CGI 独立于任何语言的，CGI 程序可以用任何脚本语言或者是完全独立编程语言实现，只要这个语言可以在这个系统上运行。

![cgi](/images/cgi.jpg)
从上面这站图我们可以窥知几个重点的阶段
1、web browser 收到 Http request 后，将 request 转给 web server
2、web server 启动 CGI 程序，并通过环境变量、标准输入传递数据
3、cgi 进程启动解析器、加载配置（如业务相关配置）、连接其它服务器（如数据库服务器）、逻辑处理等
4、cgi 程将处理结果通过标准输出、标准错误，传递给 web server
5、 web server 收到 cgi 返回的结果，构建 Http Response 返回给客户端，并杀死 cgi 进程

> web erver 与 cgi 通过环境变量、标准输入、标准输出、标准错误互相传递数据这就是 fork and execute 模式，有多少的 http request，就会 fork 出多少的子进程去处理，而每个子进程都需要启动 cgi 解释器、加载配置、链接其他服务器等初始化工作，所以这就是 cgi 性能低下的根本原因，当用户请求非常多的时候，会大量挤占系统的资源如：内存、cpu 时间等。

以下是一些 get 请求常见的环境变量

|   环境变量    |                                内容                                 |
| :-----------: | :-----------------------------------------------------------------: |
|  REMOTE_ADDR  |                       client 端的 host 名称。                       |
| REMOTE_METHOD |              client 端发出请求的方法（如 get、post）。              |
|  SCRIPT_NAME  |              CGI 程式所在的虚拟路径，如/cgi-bin/echo。              |
|  SERVER_NAME  |                   server 的 host 名称或 IP 地址。                   |
| QUERY_STRING  | 传递给 CGI 程式的请求参数，也就是用"?"隔开，添加在 URL 后面的字串。 |

### 标准输入

环境变量的大小是有一定的限制的，当需要传送的数据量大时，储存环境变量的空间可能会不足，造成数据接收不完全，甚至无法执行 CGI 程序。因此后来又发展出另外一种方法：POST，也就是利用 I/O 重新导向的技巧，让 CGI 程序可以由 STDIN 和 STDOUT 直接跟浏览器沟通。
当我们指定用这种方法传递请求的数据时，web 服务器收到数据后会先放在一块输入缓冲区中，并且将数据的大小记录在 CONTENT_LENGTH 这个环境变数，然后调用 CGI 程式并将 CGI 程序的 STDIN 指向这块缓冲区，于是我们就可以很顺利的通过 STDIN 和环境变数 CONTENT_LENGTH 得到所有的资料，再没有资料大小的限制了。
**总结**：CGI 使外部程序与 Web 服务器之间交互成为可能。CGI 程式运行在独立的进程中，并对每个 Web 请求建立一个进程，这种方法非常容易实现，但效率很差，难以扩展。面对大量请求，进程的大量建立和消亡使操作系统性能大大下降。此外，由于地址空间无法共享，也限制了资源重用。

## FastCGI

**快速通用网关接口**（Fast Common Gateway Interface／FastCGI）是通用网关接口（CGI）的改进，描述了客户端和服务器程序之间传输数据的一种标准。FastCGI 致力于减少 Web 服务器与 CGI 程式之间互动的开销，从而使服务器可以同时处理更多的 Web 请求。与为每个请求创建一个新的进程不同，FastCGI 使用持续的进程来处理一连串的请求。这些进程由 FastCGI 进程管理器管理，而不是 web 服务器。
![fastcgi](/images/fastcgi.jpg)
1、Web 服务器启动时载入初始化 FastCGI 执行环境 。 例如 IIS ISAPI、apache mod_fastcgi、nginx ngx_http_fastcgi_module、lighttpd mod_fastcgi

2、FastCGI 进程管理器自身初始化，启动多个 CGI 解释器进程并等待来自 Web 服务器的连接。启动 FastCGI 进程时，可以配置以 ip 和 UNIX 域 socket 两种方式启动。

3、当客户端请求到达 Web 服务器时， Web 服务器将请求采用 socket 方式转发到 FastCGI 主进程，FastCGI 主进程选择并连接到一个 CGI 解释器。Web 服务器将 CGI 环境变量和标准输入发送到 FastCGI 子进程。

4、FastCGI 子进程完成处理后将标准输出和错误信息从同一 socket 连接返回 Web 服务器。当 FastCGI 子进程关闭连接时，请求便处理完成。

5、FastCGI 子进程接着等待并处理来自 Web 服务器的下一个连接。

由于 FastCGI 程序并不需要不断的产生新进程，可以大大降低服务器的压力并且产生较高的应用效率。它的速度效率最少要比 CGI 技术提高 5 倍以上。它还支持分布式的部署， 即 FastCGI 程序可以在 web 服务器以外的主机上执行。

总结：CGI 就是所谓的短生存期应用程序，FastCGI 就是所谓的长生存期应用程序。FastCGI 像是一个常驻(long-live)型的 CGI，它可以一直执行着，不会每次都要花费时间去 fork 一次(这是 CGI 最为人诟病的 fork-and-execute 模式)。

### 消息类型

FastCGI 协议分为了 10 种类型，具体定义如下：

```
typedef enum _fcgi_request_type {
      FCGI_BEGIN_REQUEST   =  1, /* [in] */
      FCGI_ABORT_REQUEST   =  2, /* [in]  (not supported) */
      FCGI_END_REQUEST     =  3, /* [out] */
      FCGI_PARAMS          =  4, /* [in]  environment variables  */
      FCGI_STDIN           =  5, /* [in]  post data   */
      FCGI_STDOUT          =  6, /* [out] response   */
      FCGI_STDERR          =  7, /* [out] errors     */
      FCGI_DATA    =  8, /* [in]  filter data (not supported) */
      FCGI_GET_VALUES      =  9, /* [in]  */
      FCGI_GET_VALUES_RESULT = 10  /* [out] */
} fcgi_request_type;
```

整个 FastCGI 是二进制连续传递的，定义了一个统一结构的消息头，用来读取每个消息的消息体，方便消息包的切割。一般情况下，最先发送的是 FCGI_BEGIN_REQUEST 类型的消息，然后是 FCGI_PARAMS 和 FCGI_STDIN 类型的消息，当 FastCGI 响应处理完后，将发送 FCGI_STDOUT 和 FCGI_STDERR 类型的消息，最后以 FCGI_END_REQUEST 表示请求的结束。FCGI_BEGIN_REQUEST 和 FCGI_END_REQUEST 分别表示请求的开始和结束，与整个协议相关。

## php-fpm

PHP-FPM(**FastCGI Process Manager**：FastCGI 进程管理器)是一个 PHPFastCGI 管理器，是 FastCGI 的实现，并提供了进程管理的功能。进程包含 master 进程和 worker 进程两种进程。master 进程只有一个，负责监听端口，接收来自 Web Server 的请求，而 worker 进程则一般有多个(具体数量根据实际需要配置 pm.max_children = 5)，每个进程内部都嵌入了一个 PHP 解释器，是 PHP 代码真正执行的地方。
![fastcgi](/images/fpm.jpg)

## nginx

Nginx 不只有处理 http 请求的功能，还能做反向代理。Nginx 通过反向代理功能将动态请求转向后端 Php-fpm.
下面我们简单解释一下 nginx 的 server 模块是如何配置的

```
server {
    listen       80; #监听80端口，接收http请求
    server_name  www.example.com; #就是网站地址
    root         /usr/local/etc/nginx/www/example; # 准备存放代码工程的路径
    #路由到网站根目录www.example.com时候的处理
    location / {
        index index.php; #跳转到www.example.com/index.php
    }
    #当请求网站下php文件的时候，反向代理到php-fpm
    location ~ \.php {
        fastcgi_pass        127.0.0.1:9000;#nginx fastcgi进程监听的IP地址和端口
        fastcgi_index       index.php;
        fastcgi_param       SCRIPT_FILENAME $document_root$fastcgi_script_name;
        include             fastcgi_params; #加载nginx的fastcgi模块
    }
}
```
