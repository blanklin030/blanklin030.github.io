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


### 左值和右职
#### 左值和右值的概念
+ 左值是可以放在赋值号左边可以被赋值的值；左值必须要在内存中有实体；
+ 右值当在赋值号右边取出值赋给其他变量的值；右值可以在内存也可以在CPU寄存器。
+ 一个对象被用作右值时，使用的是它的内容(值)，被当作左值时，使用的是它的地址。

#### 引用
+ 引用是C++语法做的优化，引用的本质还是靠指针来实现的。引用相当于变量的别名。
+ 引用可以改变指针的指向，还可以改变指针所指向的值。
+ 引用的基本规则：
  + 声明引用的时候必须初始化，且一旦绑定，不可把引用绑定到其他对象；即引用必须初始化，不能对引用重定义；
  + 对引用的一切操作，就相当于对原对象的操作。

#### 左值引用和右值引用
+ 左值引用：
  + 左值引用的基本语法：type &引用名 = 左值表达式；
+ 右值引用：
  + 右值引用的基本语法type &&引用名 = 右值表达式；
  + 右值引用在企业开发人员在代码优化方面会经常用到。
  + 右值引用的“&&”中间不可以有空格。


### operator
> operator是C++的关键字，它和运算符一起使用，表示一个运算符函数，理解时应将operator=整体上视为一个函数名。

1、只有C++预定义的操作符才可以被重载；

2、对于内置类型的操作符，它的预定义不能改变，即不能改变操作符原来的功能；

3、重载操作符不能改变他们的操作符优先级；

4、重载操作符不能改变操作数的个数；

5、除了对（）操作符外，对其他重载操作符提供缺省实参都是非法的； 

### explicit
> 放构造函数前，防止隐式转换，普通构造函数能够被隐式调用，而explicit构造函数只能被显式调用。例子如下
```
class Test1 {
  public:
    Test1(int n) {
      num = n;
    }
  private:
    int num;
}

class Test2 {
  public:
    explicit Test2(int n) {
      num = n;
    }
  private:
    int num;
}

int main(){
  Test1 t1 = 12; // 隐私调用其构造函数Test1，等同于Test1 t1(12)
  Test2 t2 = 100; //编译错误，不能隐式调用其构造函数
  Test2 t3(323); //显示调用成功
}
```


### dynamic_cast
> 将基类的指针或引用安全地转换成派生类的指针或引用，并用派生类的指针或引用调用非虚函数。如果是基类指针或引用调用的是虚函数无需转换就能在运行时调用派生类的虚函数

前提条件：当我们将dynamic_cast用于某种类型的指针或引用时，只有该类型含有虚函数时，才能进行这种转换。否则，编译器会报错。

dynamic_cast运算符的调用形式如下所示：

dynamic_cast<type*>(e)  //e是指针

dynamic_cast<type&>(e)  //e是左值

dynamic_cast<type&&>(e)//e是右值

e能成功转换为type*类型的情况有三种：

1）e的类型是目标type的公有派生类：派生类向基类转换一定会成功。

2）e的类型是目标type的基类，当e是指针指向派生类对象，或者基类引用引用派生类对象时，类型转换才会成功，当e指向基类对象，试图转换为派生类对象时，转换失败。

3）e的类型就是type的类型时，一定会转换成功。

### reinterpt_cast
> 该函数将一个类型的指针转换为另一个类型的指针，这种转换不用修改指针变量值存放格式(不改变指针变量值)，只需在编译时重新解释指针的类型就可做到。
```
int* a = 1;
double* b = reinterpret_cast<double*>(a)
```

### const_cast
> 该函数用于去除指针变量的常量属性，将它转换为一个对应指针类型的普通变量。反过来说，也可以将一个非常量的指针变量转换为一个常量指针变量。这种转换是在编译期间作出的类型更改。
```
const int* a = 1;
int* pk = const_cast<int*>(a); // 相当于 int* pk = (int*) a
```

### static_cast
> 该函数主要用于基本类型之间和具有继承关系的类型之间的转换。这种转换一般会更改变量的内部表示方式，因此，static_cast应用于指针类型转换没有太大意义。

主要有如下几种用法：
+ 用于类层次结构中基类和子类之间指针或引用的转换。
+ 进行上行转换（把子类的指针或引用转换成基类表示）是安全的；
+ 进行下行转换（把基类指针或引用转换成子类指针或引用）时，由于没有动态类型检查，所以是不安全的。
+ 用于基本数据类型之间的转换，如把int转换成char，把int转换成enum。这种转换的安全性也要开发人员来保证。
+ 把void指针转换成目标类型的指针(不安全!!)
+ 把任何类型的表达式转换成void类型。
```
//基本类型转换
int i=0;
double d = static_cast<double>(i);  //相当于 double d = (double)i;

//转换继承类的对象为基类对象
class Base{};
class Derived : public Base{};
Derived d;
Base b = static_cast<Base>(d);     //相当于 Base b = (Base)d;
```

### mutable
mutable的作用有两点：
（1）保持常量对象中大部分数据成员仍然是“只读”的情况下，实现对个别数据成员的修改；
（2）使类的const函数可以修改对象的mutable数据成员。

使用mutable的注意事项：
（1）mutable只能作用于类的非静态和非常量数据成员。
（2）在一个类中，应尽量或者不用mutable，大量使用mutable表示程序设计存在缺陷。

### using
+ 1. 命名空间的使用
> 一般是为了代码的冲突，都会用命名空间，
```
// 命名空间名称
namespace android
class Test {

}
// 直接使用该命名空间
using namespace android;
// 使用该命名空间的Test类
using android::Test;
```

+ 2. 在子类汇总引用基类的成员
> 注意，using只是引用，不参与形参的指定
```
class Base {
public:
  Base() {}
  virtual ~Base() {}
  void hello(){ std::cout << "hello world" << std::endl; }
protected:
  int value;
}
// 私有继承，无法使用基类的public/protected属性的变量和函数
class Common: private Base {
public:
// 使用using方法是来引用，这样Common类就能直接用了
  using Base::hello;
  using Base:value;

  void test() { std::cout << "value is: " << value << std::endll }
}
```

+ 3. 别名指定
> 从可读性上说，typedef 要比 using 好理解，另外typedef是无法使用模版的，而using可以使用
```
// 这样之后就可以使用value_type xx 去代表type xx
using value_type = type;

template<typename T>
using Vec = MyVector<T, MyAlloc<T>>;
//usage
Vec<int> vec;
```

### const char *, char const *, char * const的区别
> 把一个声明从右向左读。( * 读成 pointer to ),C++标准规定，const关键字放在类型或变量名之前等价的。
```
// cp is a const pointer to char
// 定义一个指向字符的指针常数，即const指针
char * const cp;  
 
// p is a pointer to const char
// 定义一个指向字符常数的指针
const char * p; 

// 同上因为C++里面没有const*的运算符，所以const只能属于前面的类型。
char const * p; 
```
