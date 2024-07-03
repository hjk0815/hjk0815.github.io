---
title: C++ 常用函数
---
Welcome to [C++]

## Quick Start

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
