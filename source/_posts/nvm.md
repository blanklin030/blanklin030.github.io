---
title: macbook多版本使用nodejs
date: 2020-07-16 20:00:15
tags:
  - nodejs
categories:
  - tool
---
> nvm:node version manager
+ 安装
```
brew install nvm
```
+ 把nvm加入系统环境
```
echo "source $(brew --prefix nvm)/nvm.sh" >> ~/.bash_profile
source ~/.bash_profile
```
+ 查看远程版本
```
nvm ls-remote
```
+ 安装指定的版本
```
nvm install <version>
```
+ 安装稳定版本
```
nvm install stable
```
+ 查看本地安装的版本列表
```
nvm ls
```
+ 使用某版本
```
nvm use <version>
```
+ 删除某版本
```
nvm uninstall <version>
```