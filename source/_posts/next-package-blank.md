---
title: next模板页面空白解决方法
date: 2019-11-27 15:09:55
tags:
  - next
  - hexo
categories:
  - web
---
## 原因
+出现markdown语法出错
## 解决方案
在markdown语法里，不用使用"<>/"这些尖括号斜杆，不然会被markdown反解析后就有语法错误了!
全文搜索有出现有出现就替换掉！
## 原因 
从github仓库clone后，没有next目录
```
ls themes/next
```
是否为空？
## 解决方案
如果确实是空的
则重新clone next仓库到指定目录
```
rm -fr themes/next
git clone https://github.com/theme-next/hexo-theme-next.git next
// 修改主题
cd themes/next/
vim _config.yml
// scheme: Pisces
```
