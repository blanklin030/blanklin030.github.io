---
title: virtualenv使用介绍
date: 2020-05-19 20:51:10
tags:
  - python
categories:
  - tools
---
> 为了让不同的应用有不同的python环境，相互之间隔绝，互相不影响，我们引入virtualenv。
+ 安装virtualenv
```
pip install virtualenv
```
+ 创建虚拟环境
```
cd project
virtualenv -p $(which python) venv
```
+ 打开虚拟环境
```
source venv/bin/activate
```
> (venv)  ~/Documents/code/python/project
+ 在虚拟环境安装模块
```
pip install xx
```
+ 关闭虚拟环境
```
deactivate
```
> ~/Documents/code/python/project
+ 在pycharm配置virtualenv
![pycharm-virtualenv](/images/pycharm-virtualenv.png)
![pycharm-virtualenv-1](/images/pycharm-virtualenv-1.png)
![pycharm-virtualenv-2](/images/pycharm-virtualenv-2.png)
