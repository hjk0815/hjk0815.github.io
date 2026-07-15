---
title: BBE32EP Floating-Point   
data: 2026-07-14 13:08:56
tags: [dsp]       
categories: [cadence tensilica]        
description: connx bbe32ep dsp floating point operations
top_img: /image/jizi.png 
cover: /image/动漫少女.jpg
---
# Half Precision Data

半精度浮点数：
Half Precision，也称 FP16，就是使用 16 bit 表示一个浮点数。

IEEE754 定义如下：

| 位       | 长度   | 含义 |
| -------- | ------ | ---- |
| Sign     | 1 bit  | 符号 |
| Exponent | 5 bit  | 指数 |
| Fraction | 10 bit | 尾数 |

表示形式：
$$(-1)^s \times 1.f \times 2^{e-15}$$

fraction尾数部分：
$$1 \times 2^{-bit9} + 1 \times 2^{-bit8} + \cdots + 1 \times 2^{-bit0}$$

其中  
- bias = 15  
- 指数范围（正常数）：  
    - e = 1~30
- e=0 和 e=31 有特殊意义

FP16 长这样：  
```txt
15      10        0
+--------+----------+
|S|Exp(5)|Frac(10)  |
+--------+----------+

e.g
0 01111 0000000000
```

## Subnormal Number

次正规数：Subnormal（又叫 Denormal）就是：Exponent = 00000 && Fraction ≠ 0  
> 指数已经不能再减小时，放弃隐藏的最高位 1，继续表示更小的数。

| 类型           | 总位数 | 指数 | 尾数 |
| ------------ | --- | -- | -- |
| FP64(Double) | 64  | 11 | 52 |
| FP32(Single) | 32  | 8  | 23 |
| FP16(Half)   | 16  | 5  | 10 |


