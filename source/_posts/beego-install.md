---
title: beego安装和使用
date: 2020-08-17 15:09:55
tags:
  - golang
  - beego
categories:
  - golang
---
+ beego是什么
> beego 是一个快速开发 Go 应用的 HTTP 框架，他可以用来快速开发 API、Web 及后端服务等各种应用，是一个 RESTful 的框架，主要设计灵感来源于 tornado、sinatra 和 flask 这三个框架，但是结合了 Go 本身的一些特性（interface、struct 嵌入等）而设计的一个框架。[beego](https://beego.me)

+ 查看go的安装环境
```
go env || grep GO111MODULE
```
![go-env](/images/go-env.png)

+ 打开GO111MODULE
> go在1.11之后推出了自己的官方版本管理工具mod，需要使用mod必须将go的配置参数GO111MODULE打开，用以支持使用模块，注意在打开GO111MODULE后下载的依赖还是存在$GOPATH/pkg/mod中，而go install的结果放在$GOPATH/bin下，但是项目不再局限放在$GOPATH/src下，在任何位置我们都可以使用到$GOPATH/pkg/mod下的依赖
```
go env -w GO111MODULE=on
```
+ 修改go proxy
> 由于wall的存在，我们无法使用官方的go proxy，所以需要修改proxy地址
```
go env -w GOPROXY=https://goproxy.cn,direct
```

+ 安装beego和bee
```
go get github.com/astaxie/beego
```

+ 安装bee工具
> bee 工具是一个为了协助快速开发 beego 项目而创建的项目，您可以通过 bee 快速创建项目、实现热编译、开发测试以及开发完之后打包发布的一整套从创建、开发到部署的方案。
```
go get github.com/beego/bee
```

+ 把bee添加为系统命令
```
export GOPATH=/Users/blanklin/go
export PATH=$PATH:$GOPATH/bin
```

+ 新建helloworld项目
```
bee new helloworld
```
+ helloworld项目目录结构
helloworld
├── conf
│   └── app.conf
├── controllers
│   └── default.go
├── main.go
├── models
├── routers
│   └── router.go
├── static
│   ├── css
│   ├── img
│   └── js
├── tests
│   └── default_test.go
└── views
    └── index.tpl

+ 新建api项目
```
bee api helloworld
```

+ api项目目录结构
helloworld
├── conf
│   └── app.conf
├── controllers
│   └── object.go
│   └── user.go
├── docs
│   └── doc.go
├── main.go
├── models
│   └── object.go
│   └── user.go
├── routers
│   └── router.go
└── tests
    └── default_test.go

> 和web项目对比，少了 static 和 views 目录，多了一个 test 模块，用来做单元测试的
> 可直接通过参数创建orm：bee api [appname] [-tables=""] [-driver=mysql] [-conn=root:@tcp(127.0.0.1:3306)/test]

+ 使用go mod
```
go mod init helloworld
```

+ 运行beego项目
```
bee run
# 配合conf/app.conf[EnableDocs = true]开启swagger
bee run -downdoc=true -gendoc=true
```
> 打开浏览器访问beego开启的本地端口，一般是8080，所以地址是：http://localhost:8080

+ 使用pack命令部署生产环境
```
bee pack -be GOOS=linux
# 生成一个压缩文件helloworld.tar.gz
# 在生成环境解压后运行
nohup ./helloworld &
```

+ 使用docker部署生产环境
```
FROM golang:1.14.2-buster

ENV TZ Asia/Shanghai
ENV GOKU_LOGLEVEL info

ENV GOKU_RUNMODE prod
ENV GOKU_PORT 8080

RUN go build -v -o ./build/goku ./main.go

WORKDIR /app

COPY ../conf /app/conf
COPY ./build/goku /app/goku

ENTRYPOINT ["./goku"]

```
+ 本项目GitHub
https://github.com/blanklin030/beego-docker
> 提供初始化后支持mysql的基础crud和相关配置