---
title: 聊聊go环境安装及go.mod的使用
date: 2019-12-05 18:00:27
tags:
  - go
categories:
  - tools
---

下面流程安装在***unix（MacBook）环境***

## go的开发环境
+ 安装go
> 推荐使用go1.12+版本  
```
// 安装homebrew：
ruby -e "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/master/install)"
// 安装go
brew install go
// 将go的配置加入到系统环境
vim ~/.bask_profile
// 复制下面的内容
export GOPATH="${HOME}/.go"
export GOROOT="$(brew --prefix golang)/libexec"
export PATH=$PATH:$GOROOT/bin:$GOPATH/bin
export GOPROXY=https://goproxy.io
export GO111MODULE=on
// 生效
source ~/.bash_profile
```
+ GO111MODULE
在 1.12 版本之前，使用 Go modules 之前需要环境变量 GO111MODULE：
    + GO111MODULE=off: 不使用 Module-aware mode。
    + GO111MODULE=on: 使用 Module-aware mode，不会去 GOPATH 下面查找依赖包。
    + GO111MODULE=auto或unset: Golang 自己检测是不是使用Module-aware mode。  
> 根据官方描述在不设置GO111MODULE的情况下或者设为auto的时候，如果在当前目录或者父目录中有go.mod文件，那么就使用Module-aware mode， 而go1.12中，如果包位于GOPATH/src下，且GO111MODULE=auto, 即使有go.mod的存在，go仍然使用GOPATH mode:
+ GO PROXY
GOPROXY环境变量是伴随着modules而生，在go1.13中得到了增强，可以设置为逗号分隔的url列表来指定多个代理，其默认值为https://proxy.golang.org,direct。
direct：表示直接连接，所有direct后面逗号隔开都proxy都不会被使用。
> 这里可以设置GOPROXY=https://goproxy.io 

+ go env命令查看所有go的系统环境变量
```
GO111MODULE="on"
GOARCH="amd64"
GOBIN=""
GOCACHE="/Users/xx/Library/Caches/go-build"
GOENV="/Users/xx/Library/Application Support/go/env"
GOEXE=""
GOFLAGS=""
GOHOSTARCH="amd64"
GOHOSTOS="darwin"
GOOS="darwin"
GOPATH="/Users/xx/Documents/go"
// 省略其他的
```
## go.mod的使用
+ 创建项目
在GOPATH目录之外任意创建一个项目
```
// GOPATH="/Users/xx/Documents/go"
cd /Users/xx/Documents/code
mkdir hello-go-mod
```
+ 初始化go.mod
进入到项目根目录
```
cd /Users/xx/Documents/code/hello-go-mod
go mod init hello-go-mod
//提示创建成功： go: creating new go.mod: module hello-go-mod
ll 
// 看到根目录下多了go.mod文件
```
+ go run触发modules工作
在项目的根目录下创建一个main.go文件，复制下面的内容到文件里
```
package main

import(
    "fmt"
    "github.com/valyala/fasthttp"
)

func main() {
   fmt.Print("hello go mod")
}
```
***go run main.go***
发现项目目录下多了一个 go.sum 
go.sum用来记录每个 package 的版本和哈希值。
go.mod 文件正常情况会包含 module 和 require 模块，除此之外还可以包含 replace 和 exclude 模块

## 调教GoLang
File->New->Project
如下图：
![go-mod](/images/go-mod.png)

## golint使用
> 目录cd到$GOPATH/src/github.com
+ clone golint
```
git clone https://github.com/golang/lint.git
```
+ clone gotool
```
git clone https://github.com/golang/tools.git
```
+ go install
```
cd lint/golint
go install
```
+ 调教GoLang
  + 配置golint扩展
  ![go-lint](/images/golint.png)
  + 配置golint快捷键
  ![go-lint-shortcut](/images/golint-shortcut.png)
  > 现在可以在GoLang中使用option+command+l 进行lint了