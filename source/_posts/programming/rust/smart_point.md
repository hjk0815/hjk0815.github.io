---
title: Rust smart point        
data: 2026-05-21 15:12:32
tags: [rust]             
categories: [smart point]              
description: smart point  
top_img: /image/jizi.png 
cover: /image/动漫少女.jpg   
---


# Box<T> 堆对象分配

表达式不能隐式地解引用

## 特意的将数据分配在堆上

```rs
fn main() {
    let a = Box::new(3);
    println!("a = {}", a); // a = 3

    // 下面一行代码将报错
    // let b = a + 1; // cannot add `{integer}` to `Box<{integer}>`
}
```
智能指针往往都实现了 Deref 和 Drop 特征，因此：  
println! 可以正常打印出 a 的值，是因为它隐式地调用了 Deref 对智能指针 a 进行了解引用  
最后一行代码 let b = a + 1 报错，是因为在表达式中，我们无法自动隐式地执行 Deref 解引用操作，你需要使用 * 操作符 let b = *a + 1，来显式的进行解引用  
a 持有的智能指针将在作用域结束（main 函数结束）时，被释放掉，这是因为 Box 实现了 Drop 特征  


## 数据较大时，又不想在转移所有权时进行数据拷贝

栈上数据转移所有权时，实际上是把数据拷贝了一份，最终新旧变量各自拥有不同的数据，因此所有权并未转移。  
堆上则不然，底层数据并不会被拷贝，转移所有权仅仅是复制一份栈中的指针，再将新的指针赋予新的变量，然后让拥有旧指针的变量失效，最终完成了所有权的转移：  

```rs
fn main() {
    // 在栈上创建一个长度为1000的数组
    let arr = [0;1000];
    // 将arr所有权转移arr1，由于 `arr` 分配在栈上，因此这里实际上是直接重新深拷贝了一份数据
    let arr1 = arr;

    // arr 和 arr1 都拥有各自的栈上数组，因此不会报错
    println!("{:?}", arr.len());
    println!("{:?}", arr1.len());

    // 在堆上创建一个长度为1000的数组，然后使用一个智能指针指向它
    let arr = Box::new([0;1000]);
    // 将堆上数组的所有权转移给 arr1，由于数据在堆上，因此仅仅拷贝了智能指针的结构体，底层数据并没有被拷贝
    // 所有权顺利转移给 arr1，arr 不再拥有所有权
    let arr1 = arr;
    println!("{:?}", arr1.len());
    // 由于 arr 不再拥有底层数组的所有权，因此下面代码将报错
    // println!("{:?}", arr.len());
}
```

## 类型的大小在编译期无法确定，但是我们又需要固定大小的类型时

其中一种无法在编译时知道大小的类型是递归类型：  
```rs
/// 报错
// enum List {
//     Cons(i32, List),
//     Nil,
// }

enum List {
    Cons(i32, Box<List>),
    Nil,
}

```

## 特征对象，用于说明对象实现了一个特征，而不是某个特定的类型

在 Rust 中，想实现不同类型组成的数组只有两个办法：枚举和特征对象，前者限制较多，因此后者往往是最常用的解决办法。  
```rs
trait Draw {
    fn draw(&self);
}

struct Button {
    id: u32,
}
impl Draw for Button {
    fn draw(&self) {
        println!("这是屏幕上第{}号按钮", self.id)
    }
}

struct Select {
    id: u32,
}

impl Draw for Select {
    fn draw(&self) {
        println!("这个选择框贼难用{}", self.id)
    }
}

fn main() {
    let elems: Vec<Box<dyn Draw>> = vec![Box::new(Button { id: 1 }), Box::new(Select { id: 2 })];

    for e in elems {
        e.draw()
    }
}
```
以上代码将不同类型的 Button 和 Select 包装成 Draw 特征的特征对象，放入一个数组中，Box 就是特征对象。  
其实，特征也是 DST 类型，而特征对象在做的就是将 DST 类型转换为固定大小类型。  

## Box::leak

Box::leak，它可以消费掉 Box 并且强制目标值从内存中泄漏  

需要一个在运行期初始化的值，但是可以全局有效，也就是和整个程序活得一样久，那么就可以使用 Box::leak，例如有一个存储配置的结构体实例，它是在运行期动态插入内容，那么就可以将其转为全局有效，虽然 Rc/Arc 也可以实现此功能，但是 Box::leak 是性能最高的。  

# Deref 

```rs
use std::ops::Deref;

impl<T> Deref for MyBox<T> {
    type Target = T;

    fn deref(&self) -> &Self::Target {
        &self.0
    }
}
```

Self::Target  “当前类型解引用后所得到的目标类型”
