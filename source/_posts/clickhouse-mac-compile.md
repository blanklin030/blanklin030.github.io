---
title: mac下编译clickhouse
date: 2021-10-01 10:09:55
tags:
  - clickhouse
categories:
  - bigdata
---

### 安装Homebrew
> Homebrew要安装正确, 确保/usr/local下面出现各种share/include等目录
```
/bin/zsh -c "$(curl -fsSL https://gitee.com/cunkai/HomebrewCN/raw/master/Homebrew.sh)"
```
### 安装环境所需插件
```
brew install cmake ninja libtool gettext ccache
```
### 修正objcopy的问题
```
brew install binutils
ln -s /usr/local/opt/binutils/bin/objcopy /usr/local/bin/objcopy
```
### clion打开clickouse local不成功，无法正常编译debug
1> 安装llvm
```
brew install llvm
```
2> 配置clion的工具链ToolChain
![avatar](/images/clickhouse/1.png)
> cmake和make是新版本就可以了，配置好c和c++编译器(compiler)使用刚装好的llvm下的clang

### 使用支持ninja的CLion版本(可选, 最新版是支持的)
> CLion中的CMake使用选项
```
-DCMAKE_CXX_COMPILER_LAUNCHER=ccache
-DCMAKE_BUILD_TYPE=Debug
-GNinja
-DENABLE_CLICKHOUSE_ALL=OFF
-DENABLE_CLICKHOUSE_SERVER=ON
-DENABLE_CLICKHOUSE_CLIENT=ON
-DUSE_STATIC_LIBRARIES=OFF
-DCLICKHOUSE_SPLIT_BINARY=ON
-DSPLIT_SHARED_LIBRARIES=ON
-DENABLE_LIBRARIES=OFF
-DENABLE_UTILS=OFF
-DENABLE_TESTS=ON
-DUSE_ROCKSDB=ON
-DENABLE_ROCKSDB=ON
-DUSE_INTERNAL_ROCKSDB_LIBRARY=ON
```
![avatar](/images/clickhouse/2.png)

### 点Tools->CMake→Reset Cache and Reload Project
![avatar](/images/clickhouse/3.png)

> load过程可能会遇到的错误
```
CMake Error at contrib/croaring-cmake/CMakeLists.txt:22 (add_library):Cannot find source file:…
```
> 执行git submodule update --init --recursive 重新拉取相关依赖

### 编译clickhouse-server
1> 点右上角锤子进行编译
![avatar](/images/clickhouse/4.png)
2> 查看编译进度
![avatar](/images/clickhouse/5.png)

### debug方式运行
+ 1. debug时需要指定配置文件config.xml路径
```
--config-file=/Users/blanklin/Code/cpp/clickhouse-debug/conf/config.xml
```
![avatar](/images/clickhouse/7.png)
+ 2. 点右侧的蜘蛛按钮进行debug
![avatar](/images/clickhouse/6.png)


### cpp快速入门
```
#include <iostream>
using namespace std;
 
// main() 是程序开始执行的地方
 
int main()
{
   cout << "Hello World"; // 输出 Hello World
   return 0;
}
```
接下来我们讲解一下上面这段程序：

+ C++ 语言定义了一些头文件，这些头文件包含了程序中必需的或有用的信息。上面这段程序中，包含了头文件 <iostream>。
+ using namespace std; 告诉编译器使用 std 命名空间。命名空间是 C++ 中一个相对新的概念。
+ // main() 是程序开始执行的地方 是一个单行注释。单行注释以 // 开头，在行末结束。
+ int main() 是主函数，程序从这里开始执行。
+ cout << "Hello World"; 会在屏幕上显示消息 "Hello World"。
+ return 0; 终止 main( )函数，并向调用进程返回值 0。