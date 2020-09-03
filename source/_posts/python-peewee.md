---
title: python的orm框架peewee使用介绍
date: 2020-05-19 19:21:10
tags:
  - python
categories:
  - tools
---
> ORM(object relational mapping)对象关系映射，用来把对象模型表示的对象映射到基于SQL的关系型数据库结构中去，好处是我们在具体的操作实体对象时，不需要再去和复杂的sql语句打交道，只需要简单的操作实体对象的属性和方法。  

+ peewee介绍
  + 优点：
    + Django式的API，使其易用 
    + 轻量实现，很容易和任意web框架集成 
  + 缺点：
    + 不支持自动化 schema 迁移 
    + 多对多查询写起来不直观  

> 具体使用请访问peewee的[官方文档](http://docs.peewee-orm.com/en/latest/peewee/installation.html)  

+ 安装peewee
```
pip install pymysql
pip install peewee
```
+ 自动创建model(库和表已经在mysql创建)
```
cd project
// username是数据库用户名，host是数据库连接地址，database是数据库名
python -m pwiz -e mysql -u username -H host --password database > model.py
// 创建成功后会在model.py里看到对应的类名即表名驼峰
```
+ 新增
```
from model import Person
p = Person(name="xx",age=18)
p.save()
```
+ 删除
```
p = Person.get(Person.id == 1)
p.delete_instance() # 返回删除的行数
// 或者
query = Person.delete().where(Person.id == 1)
query.execute() # 返回删除的行数
```
+ 修改
```
query = Person.update(name="yy").where(Person.id == 1)
query.execute() # 返回修改的行数
```
+ 查询
```
// 单条查询
query = Person.get(Person.id == 1)
print(query.name)
// 多条查询
for query in Person.select().where(Person.id == 1):
  print(query.name)
```