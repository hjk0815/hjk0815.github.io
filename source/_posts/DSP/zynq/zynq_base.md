---
title: zynq base       
data: 2026-06-14 10:50:21
tags: [dsp]             
categories: [zynq]              
description: zynq base note  
top_img: /image/jizi.png 
cover: /image/动漫少女.jpg   
---

# AXI(Advanced eXtensible Interface)

AXI：Advanced eXtensible Interface（高级可扩展接口）  
- AXI 总线是连接 PS（Processing System，处理器系统，即 ARM 核心） 和 PL（Programmable Logic，可编程逻辑，即 FPGA 部分） 的核心桥梁。它并不是一条单一的物理总线，而是一套高性能、高带宽、低延迟的片上总线接口标准。通过 AXI 总线，ARM 核心可以像读写内存一样控制 FPGA 中的自定义 IP 核，而 FPGA 中的 IP 核也可以直接访问 DDR 内存或外设。  
  
## Zynq 中 AXI 的三种主要接口类型  
为了满足不同的数据传输需求，AXI 规范在 Zynq 中衍生出了三种具体的接口类型：  

1. AXI4 (AXI4-Full)  
- 特点： 高性能、高带宽。支持突发传输（Burst Transfer），即只需给出首地址，就可以连续读写最多 256 个数据。
- 应用场景： 适用于需要大批量数据搬运的场景，例如视频流传输、DMA（直接内存访问）控制器、高速缓存（Cache）与 DDR 之间的通信。

2. AXI4-Lite  
- 特点： 简化版的 AXI4。去掉了突发传输功能，每次只能读写单个数据（通常是 32 位或 64 位）。
- 应用场景： 适用于控制和配置。比如 ARM 用于读写 FPGA 中某个自定义寄存器的状态、控制开关、配置参数等。因为结构简单，它占用的 FPGA 逻辑资源非常少。

3. AXI4-Stream  
- 特点： 数据流接口。它没有地址概念，数据像流水一样单向传输（只有主设备发送、从设备接收）。它包含主控信号（如数据有效 TVALID、接收准备好 TREADY 和结束标志 TLAST）。
- 应用场景： 专门用于高速数字信号处理（DSP）、视频处理、无线通信等需要连续处理数据流、不关心内存地址的场景。

## PS 与 PL 之间的 AXI 通道
在 Zynq 架构中，PS 和 PL 之间预留了多个物理级别的 AXI 接口（通常称为接口或通道），主要包括：  
- AXI_GP (General Purpose)： 通用接口（通常是 AXI4-Lite）。用于低速控制，比如 ARM 配置 PL 外设。
- AXI_HP (High Performance)： 高性能接口。拥有独立的 FIFO，PL 可以通过它高速读写 DDR 内存。
- AXI_ACP (Accelerator Coherency Port)： 加速器一致性端口。PL 可以通过它直接访问 ARM 的 L2 Cache，保证数据一致性，非常适合硬件加速器。


