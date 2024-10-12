---
title: C++ 常用函数
---
Welcome to [C++]
[CPP合集(github)](https://github.com/huihut/interview?tab=readme-ov-file#cc)
[虚函数表实现机制](https://blog.twofei.com/496/)

## Quick Start
- 多引用自github上的[C++项目](https://github.com/huihut/interview?tab=readme-ov-file#cc)

## 成员变量或函数的命名约定
public : 公共的成员变量不建议直接暴露，而是通过公共成员函数来获取
protect : 结尾加下划线, 比如member_
private  : 结尾加下划线, 比如member_ （我觉得也可用开头加下划线 _member）

## UML类图
### 箭头
1. 普通箭头： 箭头通常表示关联关系（Association）。它指示两个类之间的连接，表示类之间的关联或依赖关系。
2. 空心箭头：空心箭头通常用于表示继承关系（Inheritance）。它表示一个类继承自另一个类，即子类继承父类的属性和方法。
### 菱形符号
菱形通常用于表示聚合关系（Aggregation）或组合关系（Composition）。
聚合关系：表示整体与部分之间的关系，部分可以存在独立于整体。
组合关系：表示整体与部分之间的关系，部分不能存在独立于整体。
### 实线和虚线
实线： 实线通常表示强关联关系，表示连接的两个类之间有较强的关联性。
虚线：通常表示弱关联关系或依赖关系，表示两个类之间的关联性较弱或依赖关系。


### 谓词（Predicate）
-[x] 是一种可调用的对象，用于对给定的输入值进行判断，并返回一个布尔值作为结果。谓词通常用于条件判断、过滤和排序等场景。
-[x] 谓词可以是函数指针、函数对象、Lambda 表达式或可以转换为函数指针或函数对象的可调用对象。它们通常被用作算法函数的参数，以确定算法在处理元素时应该如何操作。
-[x] 谓词在 C++ 中常用于标准库算法函数（例如 std::find_if、std::remove_if、std::sort）中的条件判断，用于确定元素是否满足特定的条件。谓词接受一个或多个参数，并返回一个布尔值，表示给定的条件是否成立。


More info: [Writing-JK](https://hexo.io/docs/writing.html)

### Run server

``` bash
$ hexo server
```

More info: [Server](https://hexo.io/docs/server.html)

### Generate static files

``` bash
$ hexo generate
```

More info: [Generating](https://hexo.io/docs/generating.html)

### Deploy to remote sites

``` bash
$ hexo deploy
```

More info: [Deployment](https://hexo.io/docs/one-command-deployment.html)

### 静态变量
静态变量是在程序加载前创建由编译器和系统管理, 局部静态变量在第一次访问时初始化，C++11 以后，局部静态变量的初始化是线程安全的。

### 内联函数 inline
编译器将函数插入到每个调用点，而不是生成函数调用，可以消除函数调用的开销提高运行时新能，特别是短小 频繁调用的函数


### 模板参数类型的推导
在模板函数的调用中，编译器会根据传递给函数的实参类型来推导模板参数的类型。

```cpp
// 定义一个模板函数，接受格式化字符串和可变参数
template<typename... Args>
void printValues(const std::string& fmt, Args&&... args) {
    std::cout << "Format string: " << fmt << "\n";

    (std::cout << ... << args) << "\n";
}

```

`Args&&...` : &&是万能引用，使得函数模板能够接收任何类型的参数，同时保持其原始的值类别（左值或右值）。它也常被称为 “转发引用”（Forwarding Reference）。
`T&&` 通常表示 右值引用，但在模板函数参数中，当 T 是一个模板类型参数时，T&& 会被视为 万能引用。万能引用可以绑定到：

左值（Lvalue）：即可以绑定到有名字的、可修改的变量。
右值（Rvalue）：即临时对象、字面量等，不是左值的对象。
万能引用的语法形式是 T&&，但只有在模板参数的上下文中才能被识别为万能引用。

`std::forward<Args>(args)...`：完美转发参数，保持其原始的值类别，避免不必要的拷贝和移动。

`std::forward` : 实现参数转发

### 原子操作
`std::atomic<bool> data` : 该类型用于实现原子的布尔类型操作，确保对其进行的读写操作是原子的。
该类型的load方法可用获取该值 可指定读取顺序，比如load(std::memory_order_relaxed)是以最轻量级读取


### 资源锁
`std::lock_guard` : 一个管理互斥锁的类（std::mutx等互斥锁）自动管理锁的生命周期


### 引用成员变量和const成员变量
引用成员变量和const成员变量只能在初始化列表中初始化，因为他们不能通过赋值来进行初始化，只能在申明的时候进行初始化。


## RALL (Resource Acquisition Is Initialization)
资源获取就是初始化:
**可以指定对象具有构造函数和析构函数，这些构造函数和析构函数在适当的时候由编译器自动调用，这为管理给定对象的内存提供了更为方便的方法。**
RAII总结如下：

- 资源在析构函数中被释放

- 该类的实例是堆栈分配的
- 资源是在构造函数中获取的。 

RAII代表“资源获取是初始化”。

常见的例子有：

- 文件操作

- 智能指针 std :: shared_ptr，std :: unique_ptr

- 互斥量 std :: lock_guard