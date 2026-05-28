---
title: cpp note        
data: 2026-05-06 10:50:21
tags: [cpp]             
categories: [note]              
description: note  
top_img: /image/jizi.png 
cover: /image/动漫少女.jpg   
---


# 操作符字符串化（#）& 符号连接操作符 （##）

```cpp
#define example( instr )  printf("xxx %s\n", #instr)  // “把参数 instr 直接转为字符串字面量”。
#define example1(instr)   #instr  // 把参数文本直接转成字符串字面量，替换到调用处
#define example2(n)         num ## n

int main() {
    example(TEST::LIC_EDFA);   // xxx TEST::LIC_EDFA
    std::string str = example1(abc);
    printf("%s\n", str.c_str());  // abc
    printf("xxx %s\n", example1(yesai));  // yesai

    int num = 9;
    int num9 = example2(9);
    return 0;
}
```

# volatile

1.并行设备的硬件寄存器
- 存储器映射的硬件寄存器通常加volatile，因为寄存器随时可以被外设硬件修改。当声明指向设备寄存器的指针时一定要用volatile，它会告诉编译器不要对存储在这个地址的数据进行假设。
2.一个中断服务程序中修改的供其他程序检测的变量
- volatile提醒编译器，它后面所定义的变量随时都有可能改变。因此编译后的程序每次需要存储或读取这个变量的时候，都会直接从变量地址中读取数据。如果没有volatile关键字，则编译器可能优化读取和存储，可能暂时使用寄存器中的值，如果这个变量由别的程序更新了的话，将出现不一致的现象。
3.多线程应用中被几个任务共享的变量
- 单地说就是防止编译器对代码进行优化.比如：
  ```c
    XBYTE[2]=0x55;
    XBYTE[2]=0x56;
    XBYTE[2]=0x57;
    XBYTE[2]=0x58;
  ```
对外部硬件而言，上述四条语句分别表示不同的操作，会产生四种不同的动作，但是编译器却会对上述四条语句进行优化，认为只有XBYTE[2]=0x58(即忽略前三条语句，只产生一条机器代码）。如果键入volatile，编译器会逐一的进行编译并产生相应的机器代码（产生四条代码）