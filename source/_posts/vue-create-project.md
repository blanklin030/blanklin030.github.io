---
title: 教你新建一个vue项目
date: 2019-11-27 14:10:40
tags:
  - vuejs
  - vue-cli
  - webpack
categories:
  - web
---

+ 全局安装vue脚手架
```
npm install vue-cli -g
```
+ 使用webpack初始化项目
```
vue init webpack my-project
```
下面是安装过程系统给出的提示
```
//项目名称
? Project name devops
//项目描述
? Project description bigdata devops platform
//项目作者
? Author blank <luck.linxiaoyu@gmail.com>
? Vue build standalone
//安装vue-router（vue的路由）
? Install vue-router? Yes
// 使用eslint格式化你的代码
? Use ESLint to lint your code? Yes
//选择Airbnb作为eslint的格式标准
? Pick an ESLint preset Airbnb
//选一个单元测试工具
? Set up unit tests Yes
// 我选的是jest
? Pick a test runner jest
//使用nightwatch开始e2e单元测试
? Setup e2e tests with Nightwatch? Yes
//项目创建使用使用npm install进行其他包安装
? Should we run `npm install` for you after the project has been created? 


# Project initialization finished!
# ========================
// 安装完成给出提示如何启动项目
To get started:
  cd new
  npm run dev

```
