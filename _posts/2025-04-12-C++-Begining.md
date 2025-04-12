---
layout: post
title: 开始做 C++ 项目
categories: [C++]
description: 要开始做一个 C++ 项目，需首先了解一下项目结构，构建工具和构建流程，辅助工具，语言特性等。
keywords: PC
mermaid: false
sequence: false
flow: false
mathjax: false
mindmap: false
mindmap2: false
---

## 前言

首先，做 C++ 项目需要一份： [C++开发发展路线图](https://roadmap.sh/cpp)。

在做 PC(C++) 项目一段时间后，已经从新手变成了半熟练工，这中间最主要的还是快速学习和快速实践，这比自己单独花费时间去琢磨效率高很多。

虽然做 PC(C++) 项目与做 Android(Kotlin/Java) 项目不同，但是可以延用以前做 Android 项目的很多经验。

不管是做 Android 项目，还是做 PC 项目，它的流程都是：需求 &rarr; 设计 &rarr; 编码 &rarr; 测试 &rarr; 灰度 &rarr; 上线。按照这个流程，在做 PC 项目过程中做出对应的适配即可。


其实抛开语言和环境的不同，做 PC 项目和做 Android 项目的诉求是一样的。比如：项目模块化，静态代码分析，APM监控，CI/CD等。


那么，在做 PC 项目前需要了解什么？要开始做一个 C++ 项目，需了解一下项目结构，构建工具和构建流程，辅助工具，语言特性，这样就可以开始了。

## 项目结构

要想对项目整体有把握和掌控，那么首先需了解项目结构。

一个项目不可能把所有的代码都放在一起，这样不利于开发、测试和维护，所以 C++ 项目也会模块化。那么要了解项目结构，就需要分析项目模块依赖关系。生成模块依赖关系图，可以使用 `CMake` 和 `Graphviz` 。

```
cmake . -B build
cd build
cmake --graphviz=project.dot .
dot -Tpng project.dot -o cmake-deps.png
```
通过生成的模块依赖关系图，不仅可以看到项目整体结构（分层结构，从底层到高层），还可以看到项目使用了哪些技术（本地库，三方库，OpenGL，QT等）。

## 构建工具和构建流程

### 构建工具

要想对项目进行模块化管理，依赖管理，Debug/Release打包管理等，就需要了解构建工具。

C/C++ 项目构建工具有：Make, CMake, Bazel, Ninja, Meson, Autotools。在 PC 项目中，使用 CMake 构建工具。

**CMake** 适合跨平台中大型项目，通过 `CMakeLists.txt` 配置文件构建项目：

```cmake
# 指定最低 CMake 版本
cmake_minimum_required(VERSION 3.10)

# 定义项目名称
project(MyProject)

# 设置 C++ 标准版本
set(CMAKE_CXX_STANDARD 17)
# 强制使用指定 C++ 版本
set(CMAKE_CXX_STANDARD_REQUIRED ON)

# 添加子目录
add_subdirectory(external/SomeLibrary)

# 头文件和源文件
include_directories(${PROJECT_SOURCE_DIR}/include)
aux_source_directory(src SRC_LIST)

# 目标
add_executable(my_program ${SRC_LIST})
target_link_libraries(my_program SomeLibrary)

# 编译选项
target_compile_options(my_program PRIVATE -Wall -Wextra -O2)

# 产物输出
set(CMAKE_RUNTIME_OUTPUT_DIRECTORY ${CMAKE_BINARY_DIR}/bin)
set(CMAKE_LIBRARY_OUTPUT_DIRECTORY ${CMAKE_BINARY_DIR}/lib)

# 平台相关
if(WIN32)
    message(STATUS "Building for Windows")
    add_definitions(-DPLATFORM_WINDOWS)
elseif(APPLE)
    message(STATUS "Building for macOS")
elseif(UNIX)
    message(STATUS "Building for Linux")
endif()

```

模块 = 目录（源码文件）+`CMakeLists.txt`。

### 构建流程

要想知道C++项目是如何运行起来的，那么就需要了解它的构建流程，也就是如何将源代码变成可执行文件。

C/C++ 项目构建流程主要包含：预处理、编译、汇编、链接。

| 阶段                 | 主要作用                 | 工具                                | 产物                                           |
| ------------------ | -------------------- | --------------------------------- | -------------------------------------------- |
| 预处理（Preprocessing） | 处理宏、头文件包含、条件编译指令     | 预处理器（如 `cpp`）                     | `.i`（预处理后的代码）                                |
| 编译（Compilation）    | 将预处理后的代码转换为汇编代码      | 编译器（如 `gcc`、`clang`、`MSVC`）       | `.s`（汇编代码）                                   |
| 汇编（Assembly）       | 将汇编代码转换为机器码（目标文件）    | 汇编器（如 `as`）                       | `.o` 或 `.obj`（目标文件）                          |
| 链接（Linking)        | 合并目标文件和库，生成可执行文件或共享库 | 链接器（如 `ld`、`lld`、`MSVC link.exe`） | 可执行文件（如 `app`、`app.exe`）或动态库（如 `.so`、`.dll`） |

以后遇到构建问题，也助于分析和排查。

## 辅助工具

### IDE

在 PC 上做 C++ 项目开发，主要的 IDE 有 `CLion`、`Visual Studio` 和 `VS Code`等。CLion 来自JetBrains，所以习惯了使用 AndroidStudio，那么选择 CLion 会非常友好和方便。

在 CLion 中使用快捷键，Git，调试，生成（构造函数，析构函数，Setter和Getter等）代码，代码检查等和 AS 一样，这样开始做 C++  项目就会很顺手。

如果使用 Windows 做开发，那么也可以安装 Visual Studio，它的调试，编译，性能分析等都特别强大。

### doxygen
[doxygen](https://www.doxygen.nl/)是文档生成工具，能分析 C++ 代码并生成文档。文档中有：
- 项目 API 说明；
- 类继承关系、文件包含关系、函数调用关系；
- 模块关系图（结合 Graphviz）；
- 导航和搜索，查找特定的类、函数或变量；
- ...

当对项目不熟悉的时候，能帮助阅读代码理解项目。

### WinDbg
如果使用 Windows 做开发，那么可以使用 [WinDbg](https://learn.microsoft.com/en-us/windows-hardware/drivers/debugger/)做崩溃分析。

Windows 应用程序崩溃会生成 dmp 文件，该文件记录了崩溃堆栈和其他线程快照信息。dmp 文件可以使用 Visual Studio 或者 WinDbg 打开。但需要配置正确符号，才能看到完整崩溃堆栈。

符号文件（.pdb）：在构建项目时，链接器会生成 app.exe 对应的 app.pdb。

通过设置系统环境变量：`_NT_SYMBOL_PATH=srv*d:\symbols-sys*https://msdl.microsoft.com/download/symbols;`，可以指定 WinDbg 查找符号文件（.pdb）的默认路径。

上面的 `_NT_SYMBOL_PATH`，会从 [https://msdl.microsoft.com/download/symbols](https://msdl.microsoft.com/download/symbols) 下载符号到本地路径 `d:\symbols-sys`(本地先创建一个文件夹)。

此时，使用 WinDbg 打开 dmp 文件，并执行 `!analyze -v`，就可以看到源代码堆栈信息。

### 静态代码检查

在不熟悉使用 C++ 编写代码的时候，难免会写出一些质量有问题的代码。那么使用静态代码检查工具，就可以帮助检查并修正这些问题。

使用 CLion + Clang 做静态代码检查：
- 代码风格检查：在项目根目录添加 `.clang-format` 规则配置文件;
- 代码质量检查：在项目根目录添加 `.clang-tidy` 规则配置文件或直接使用 CLion 配置;

`.clang-format`规则配置，如：

```
BasedOnStyle: Google       # Google 风格的 C++ 编码规范
IndentWidth: 4             # 缩进宽度设置为 4 个空格（Google默认是 2）
TabWidth: 4                # tab 字符的显示宽度为 4（主要用于对齐，但不实际使用 Tab）
UseTab: Never              # 永远不使用 Tab 缩进，只使用空格（符合大多数团队风格）
ColumnLimit: 120           # 每行最大字符数限制为 120，超出会自动换行（Google默认是 80）
```
`.clang-tidy`规则配置，如：

```
Checks: >                           
  bugprone-*,                      # 检查bug
  performance-*,                   # 检查性能
  modernize-*,                     # 使用现代 C++ 特性（如 nullptr、auto、range-for）
  readability-*,                   # 提高代码可读性（如命名规范、代码结构优化）
  clang-analyzer-*,                # 使用 Clang 内置静态分析器，发现深层次逻辑缺陷
  cppcoreguidelines-*             

WarningsAsErrors: '*'              # 把警告都作为错误处理

HeaderFilterRegex: '.*'            # 过滤要分析的头文件，'.*' 表示分析所有头文件（包括外部）

FormatStyle: file                  # 代码风格使用 .clang-format 文件中的规则配置

CheckOptions:                     
  - key: modernize-use-nullptr.NullMacros
    value: 'NULL'                 # 当发现使用 NULL 宏时，建议替换为 nullptr
  - key: readability-identifier-naming.VariableCase
    value: camelBack              # 设置变量命名风格为驼峰

```
一般可以直接使用 CLion 中规则配置：Settings (Preferences) &rarr; Editor &rarr; Inspections &rarr; C/C++ &rarr; Clang-Tidy
（设置 &rarr; 编辑器 &rarr; 检查 &rarr; C/C++ &rarr; Clang-Tidy）。

在 CLion 中开始静态代码检查（操作和AS一样）：选择代码目录或文件，右键检查代码。

使用静态代码检查工具，可以保证团队代码风格统一和避免有低级错误的代码。

## 语言特性

虽然 C++ 与 Java 不同，但是语言的很多设计思想还是一样的。对于一个 Java 熟练工，C++ 新手，有必要先了解一下它的语言特性。

### 指针与引用

指针是一个变量，它存储的是另一个变量的内存地址。引用是已有变量别名，它们在内存中指向相同地址。

指针与引用语法上的区别：

<table style="border-collapse: collapse; width: 100%; font-family: monospace;">
  <tr>
    <th style="border: 1px solid #aaa; padding: 8px; text-align: center; background-color: #f2f2f2;"></th>
    <th style="border: 1px solid #aaa; padding: 8px; text-align: center; background-color: #f2f2f2;">指针</th>
    <th style="border: 1px solid #aaa; padding: 8px; text-align: center; background-color: #f2f2f2;">说明</th>
    <th style="border: 1px solid #aaa; padding: 8px; text-align: center; background-color: #f2f2f2;">引用</th>
    <th style="border: 1px solid #aaa; padding: 8px; text-align: center; background-color: #f2f2f2;">说明</th>
  </tr>
  <tr>
    <td style="border: 1px solid #aaa; padding: 8px; text-align: center;">定义</td>
    <td style="border: 1px solid #aaa; padding: 8px; text-align: center;"><p><code>int num = 10;<br>int* ptr = &num ;</code></p></td>
    <td style="border: 1px solid #aaa; padding: 8px; text-align: center;">指针 ptr 存储变量 num 的内存地址</td>
    <td style="border: 1px solid #aaa; padding: 8px; text-align: center;"><p><code>int num = 10;<br>int& ref = num;</code></p></td>
    <td style="border: 1px solid #aaa; padding: 8px; text-align: center;">引用 ref 绑定到 num，它们内存地址一样</td>
  </tr>
  <tr>
    <td style="border: 1px solid #aaa; padding: 8px; text-align: center;">get</td>
    <td style="border: 1px solid #aaa; padding: 8px; text-align: center;"><code>*ptr</code></td>
    <td style="border: 1px solid #aaa; padding: 8px; text-align: center;">输出 10</td>
    <td style="border: 1px solid #aaa; padding: 8px; text-align: center;"><code>ref</code></td>
    <td style="border: 1px solid #aaa; padding: 8px; text-align: center;">输出 10</td>
  </tr>
  <tr>
    <td style="border: 1px solid #aaa; padding: 8px; text-align: center;">set</td>
    <td style="border: 1px solid #aaa; padding: 8px; text-align: center;"><code>*ptr = 20;</code></td>
    <td style="border: 1px solid #aaa; padding: 8px; text-align: center;">*ptr 为 20，num 为 20</td>
    <td style="border: 1px solid #aaa; padding: 8px; text-align: center;"><code>ref = 20;</code></td>
    <td style="border: 1px solid #aaa; padding: 8px; text-align: center;">ref 为 20，num 为 20</td>
  </tr>
  <tr>
    <td style="border: 1px solid #aaa; padding: 8px; text-align: center;">重新绑定</td>
    <td style="border: 1px solid #aaa; padding: 8px; text-align: center;"><code>int anotherNum = 100;<br>ptr = &anotherNum;</code></td>
    <td style="border: 1px solid #aaa; padding: 8px; text-align: center;">指针 ptr 存储变量 anotherNum 的内存地址</td>
    <td style="border: 1px solid #aaa; padding: 8px; text-align: center;">不支持</td>
    <td style="border: 1px solid #aaa; padding: 8px; text-align: center;"></td>
  </tr>
  <tr>
    <td style="border: 1px solid #aaa; padding: 8px; text-align: center;">空值</td>
    <td style="border: 1px solid #aaa; padding: 8px; text-align: center;"><code>int* ptr = nullptr;</code></td>
    <td style="border: 1px solid #aaa; padding: 8px; text-align: center;">指针 ptr 存储的内存地址为空</td>
    <td style="border: 1px solid #aaa; padding: 8px; text-align: center;"><code>int& ref;</code></td>
    <td style="border: 1px solid #aaa; padding: 8px; text-align: center;">不支持，不能有空引用</td>
  </tr>
</table>
  

指针与引用本质上的区别是：**指针是可变的内存地址变量，引用是不可变的间接访问方式**。

在实际开发中，什么时候使用指针，什么时候使用引用？

|  | 指针 | 引用 |说明|
| --- | --- |--- |--- |
| 创建空对象 | &#x2713; | &times; |指针可为空，引用必须绑定|
| 延迟初始化 | &#x2713; |&times; | 指针可先定义，后赋值，引用必须绑定|
| 可变对象 | &#x2713; |&times;| 指针可重新指向新的对象，引用不支持|
| 非空对象 | &times; |&#x2713;| 指针可为空，引用必须有值|

例子：

```c++
//只读取集合，集合不可变
void readVector(const std::vector<int>& vec) {
    for (int v : vec) {
        
    }
}

//修改不为空集合
void writeVector(std::vector<int>& vec, int value) {
    vec.push_back(value);
}

//修改可为空集合
void writeVector(std::vector<int>* vec) {
    if (vec) {
        vec->push_back(42);
    }
}
```

总的来说，指针灵活，引用简洁。


#### 智能指针

创建指针，往往忘记释放指针，从而导致内存泄漏。智能指针能解决这一问题。智能指针有`std::unique_ptr`、`std::shared_ptr` 和 `std::weak_ptr`。

内存泄漏：

```c++
void leak() {
    int* ptr = new int(42);
    // 忘记 delete ptr;
}  // ptr 超出作用域，内存永远无法回收（泄漏）
```

使用 `std::unique_ptr`：
```c++
#include <memory>

void unique() {
    std::unique_ptr<int> p = std::make_unique<int>(42);
}  // 离开作用域时自动 delete
```
使用 `std::shared_ptr`：
```c++
#include <memory>

void share() {
    std::shared_ptr<int> p1 = std::make_shared<int>(42);
    std::shared_ptr<int> p2 = p1;  // 引用计数 +1
}  // 引用计数归 0 后，自动释放
```
使用 `std::weak_ptr`：
```c++
#include <memory>

struct B;  // 前向声明

struct A {
    std::shared_ptr<B> b;
};

struct B {
    std::weak_ptr<A> a;  // 防止循环引用
};

```

### 代码组织

头文件（`.h`）：**声明**函数、类等；源文件（`.cpp` 或 `.cc`）：**实现**声明函数、类等。（Google 的 C++ 编码规范推荐使用 `.cc` 作为源文件扩展名。）

头文件 `Person.h`：

```c++
#pragma once
#include <string>

class Person {
public:
    //构造函数
    explicit Person(const std::string& name);
    
    ~Person() = default;

    std::string getName() const;

    void setName(const std::string& name);

private:
    std::string name;
};
```
源文件 `Person.cpp`：

```c++
#include "Person.h"

Person::Person(const std::string& name)
    : name(name) {}

std::string Person::getName() const {
    return Person::name;
}

void Person::setName(const std::string& name) {
    Person::name = name;
}
```
命名空间：用于组织模块，避免全局命名冲突。

头文件：`math_utils.h`：
```c++
// 
#pragma once

namespace math {
    int add(int a, int b);
    int multiply(int a, int b);
}
```
源文件 `math_utils.cpp`：

```c++
#include "math_utils.h"

int math::add(int a, int b) {
    return a + b;
}

int math::multiply(int a, int b) {
    return a * b;
}
```

### 析构函数

析构函数：当对象被销毁时，C++ 编译器会自动调用该对象的析构函数。所以析构函数用于释放内存。

头文件 `Person.h`：

```c++
#pragma once

class Person {
public:
    explicit Person(const char* name);
    ~Person(); // 析构函数
    
private:
    char* m_name;
};
```
源文件 `Person.cpp`：

```c++
#include "Person.h"
#include <cstring>

Person::Person(const char* name)
{
    // 分配内存保存字符串
    size_t len = std::strlen(name) + 1;
    m_name = new char[len];
    std::strcpy(m_name, name);
}

Person::~Person()
{
    delete[] m_name; // 释放动态内存
}
```

### 拷贝

```c++
void Employee::setPerson(const Person& p) {
    person = p;  // 触发 Person 类的赋值运算符
}

const Person& Employee::getPerson() const{
    return person;
}
```
在数据模型类中，属性都有 setter 和 getter，当属性是类对象时，如果提供了 setter 方法，那么该类需要提供拷贝函数（深拷贝）：

```c++
// 拷贝赋值运算符 
Person& operator=(const Person& other) { 
 if (this != &other) { 
   // 防止自赋值 
   this->name = other.name; 
 } 
 return *this; 
}
```
如果不提供拷贝函数，那么就是浅拷贝，原对象和拷贝出来的对象共享同一块资源。

### 传引用和值

传引用：
```c++
void Employee::setPerson(const Person& p) {
    
}
```
传值：
```c++
void Employee::setPerson(const Person p) {
   
}
```

`auto person = Person("C++"); setPerson(person)`， 如果调用`setPerson(const Person& p)`，那么传递的就是 `person` 对象；如果调用`setPerson(const Person p)`，那么传递的就是 `person` 对象副本，person 对象会被拷贝一次。

避免不必要的拷贝开销，使用引用传递。

## 总结

其实，做移动端项目前需要了解什么。那么，在做 C++ 项目前就需要了解什么。

首先需要一份[C++开发发展路线图](https://roadmap.sh/cpp)，然后简单了解一下项目结构，构建工具和构建流程，辅助工具，语言特性等，然后就可以开始做 C++ 项目。

