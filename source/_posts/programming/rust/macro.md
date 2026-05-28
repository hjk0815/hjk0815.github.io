---
title: Rust smart point        
data: 2026-05-21 15:12:32
tags: [rust]             
categories: [smart point]              
description: smart point  
top_img: /image/jizi.png 
cover: /image/动漫少女.jpg   
---

# 声明式宏 `macro_rules!`

## 模式解析

简化的vec!  

```rs
#[macro_export] // 宏导出
macro_rules! vec { // 宏定义
    ( $( $x:expr ),* ) => {  // 模式匹配
        {
            let mut temp_vec = Vec::new();
            $(
                temp_vec.push($x);
            )*
            temp_vec
        }
    };
}
```

### `( $( $x:expr ),* )`的含义

圆括号 () 将整个宏模式包裹其中。紧随其后的是 `$()`，跟括号中模式相匹配的值(传入的 Rust 源代码)会被捕获，然后用于代码替换。在这里，模式 `$x:expr` 会匹配任何 Rust 表达式并给予该模式一个名称：`$x`

$() 之后的逗号说明 $() 所匹配的代码使用逗号分隔符分割，紧随逗号之后的 * 说明 * 之前的模式($()内的部分)会被匹配零次或任意多次(类似正则表达式)(且以逗号分割)。

# 过程宏

过程宏是使用源代码作为输入参数，基于代码进行一系列操作后，再输出一段全新的代码。注意，过程宏中的 derive 宏输出的代码并不会替换之前的代码，这一点与声明宏有很大的不同！

```rs
use hello_macro::HelloMacro;
use hello_macro_derive::HelloMacro;

#[derive(HelloMacro)]
struct Sunfei;

#[derive(HelloMacro)]
struct Sunface;

fn main() {
    Sunfei::hello_macro();
    Sunface::hello_macro();
}
```

单独的宏包 */src/lib.rs

```rs
extern crate proc_macro;

use proc_macro::TokenStream;
use quote::quote;
use syn;
use syn::DeriveInput;

#[proc_macro_derive(HelloMacro)]
pub fn hello_macro_derive(input: TokenStream) -> TokenStream {
    // 基于 input 构建 AST 语法树
    let ast:DeriveInput = syn::parse(input).unwrap();

    // 构建特征实现代码
    impl_hello_macro(&ast)
}
```