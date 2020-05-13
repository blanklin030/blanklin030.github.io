---
title: grafana开发环境搭建（MacBook）
date: 2020-05-12 21:04:11
tags:
  - grafana
categories:
  - tools
---

+ 安装go
```
brew install go
```
+ 安装nodejs
```
brew install nodejs
# npm切换淘宝源
npm config set registry https://registry.npm.taobao.org
```
+ 安装yarn
```
npm install -g yarn
```
+ clone项目
git项目地址：https://github.com/grafana/grafana
```
go get github.com/grafana/grafana
```
> 如果无法get下来，就直接clone 项目后，放到$GOPATH/src/github.com/grafana/grafana
+ 前端环境
```
cd $GOPATH/src/github.com/grafana/grafana
npm install -g node-gyp
# 安装依赖
yarn install --pure-lockfile
# 执行编译
yarn start
```
> 编译完成后，在public文件夹会看到多了个build文件夹
+ 后端环境
```
cd $GOPATH/src/github.com/grafana/grafana
go run build.go setup
go run build.go build
```
> 编译完成后，会看到多了个bin文件夹
+ 运行

```
bin/grafana-server start
``` 

+ 打包
```
go run build.go build package
```

+ 安装

```
sudo dpkg -i grafana_xxxx.deb 
# 项目文件
/usr/share/grafana
# 配置文件
/etc/grafana/grafana.ini
```
