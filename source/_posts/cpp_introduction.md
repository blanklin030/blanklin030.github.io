---
title: cpp奇葩语法介绍
date: 2022-01-13 19:04:11
tags:
  - cpp
categories:
  - introduction
---

### const


### friend
> 友邻函数

### extern


### vitrual
> 基类希望其派生类进行覆盖（`override`）的函数。这种函数，基类通常将其定义为虚函数（加`virtual`）。当我们使用基类的指针或者引用调用虚函数时，该调用将被**动态绑定**
#### 用法解释：
```
class Base {
  public:
    virtual void print() {
      cout << "==BASE==";
    }
}
class Derived: public Base {
  public:
    void print(){
      cout << "==DERIVED=="
    }
}
int main() {
  Base *pointer = new Derived();
  // ==DERIVED==
  pointer->print();
}
```
> 类`Base`中加了`Virtual`关键字的函数就是虚拟函数（例如函数`print`），于是在`Base`的派生类`Derived`中就可以通过重写虚拟函数来实现对基类虚拟函数的覆盖。当基类`Base`的指针`point`指向派生类`Derived`的对象时，对`point`的`print`函数的调用实际上是调用了`Derived`的`print`函数而不是`Base`的`print`函数。这是面向对象中的多态性的体现 
#### 纯虚函数
> 纯虚函数声明如下： `virtual void funtion1()=0;` 纯虚函数一定没有定义，纯虚函数用来规范派生类的行为，即接口。包含纯虚函数的类是抽象类，抽象类不能定义实例，但可以声明指向实现该抽象类的具体类的指针或引用。

#### 注意点
+ 定义一个函数为虚函数，不代表函数为不被实现的函数。
+ 定义一个虚函数是为了允许用基类的指针或引用来调用子类的这个函数。
+ 定义一个函数为纯虚函数，才代表函数没有被实现。
+ 定义纯虚函数是为了实现一个接口，起到一个规范的作用，规范继承这个类的程序员必须实现这个函数。
+ 任何友元(friend)/构造(construct)/static静态函数之外的函数都可以是虚函数。
+ 关键字virtual只能出现在类内部的声明语句之前而不能用于类外部的函数定义
+ 基类定义了virtual，继承类的该函数也具有了virtual属性
### static


### template


### inline

### auto
> 在声明变量时根据变量初始值的类型自动为此变量选择匹配的类型。
#### 用法解释：
```
auto f = 3.14;  //double
auto s("hello");  //const char*
auto z = new auto(9);  //int *
```
#### 注意点
+ 可以用valatile，pointer（*），reference（&），rvalue reference（&&） 来修饰auto
```
auto k = 5;  
auto* pK = new auto(k);  
auto** ppK = new auto(&k);  
const auto n = 6;  
```
+ 用auto声明的变量必须初始化
+ auto不能与其他类型组合连用
+ 函数和模板参数不能被声明为auto
+ 定义在堆上的变量，使用了auto的表达式必须被初始化
+ 以为auto是一个占位符，并不是一个他自己的类型，因此不能用于类型转换或其他一些操作，如sizeof和typeid
+ 定义在一个auto序列的变量必须始终推导成同一类型
```
// 错误，必须是初始化为同一类型
auto x1 = 5, x2 = 5.0, x3='r';  
```
+ auto不能自动推导成CV-qualifiers (constant & volatile qualifiers)
+ auto会退化成指向数组的指针，除非被声明为引用

### template