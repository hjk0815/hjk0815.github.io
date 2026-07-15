# 通道回退（Channel Fallback）深度分析

## 问题现象

在 [bbe_dma_adapter.c:140-159](bbe_dma_adapter.c#L140-L159) 中，BBE #1 和 #3 没有使用自己的源数据，而是**降级回退**到 BBE #0 和 #2 的数据：

```c
case 1:
    // 原设计应该用 src_ch1
    // src_addr = (Addr_t) (&src_ch1[0]);
    // src_meta_addr = (Addr_t) &src_meta_ch1[0];
    
    // 实际使用 src_ch0（回退！）
    src_addr = (Addr_t) (&src_ch0[0]);
    src_meta_addr = (Addr_t) &src_meta_ch0[0];
    break;

case 3:
    // 原设计应该用 src_ch3
    // src_addr = (Addr_t) (&src_ch3[0]);
    // src_meta_addr = (Addr_t) &src_meta_ch3[0];
    
    // 实际使用 src_ch2（回退！）
    src_addr = (Addr_t) (&src_ch2[0]);
    src_meta_addr = (Addr_t) &src_meta_ch2[0];
    break;
```

---

## 根本原因：硬件约束

### 1. 内存映射现实

查看 [mem.h](mem.h) 的数据定义，**只有2个真实的数据源**存在：

```c
// rxc0 ds0 FIFO（接收链路0）
extern int16_t src_ch0[CHIRPS_PER_CHIRPFRAME][MAG2_SIZE];        // ✓ 存在
extern meta_id_t src_meta_ch0[CHIRPS_PER_CHIRPFRAME];

// rxc1 ds0 FIFO（接收链路1）
extern int16_t src_ch2[CHIRPS_PER_CHIRPFRAME][MAG2_SIZE];        // ✓ 存在
extern meta_id_t src_meta_ch2[CHIRPS_PER_CHIRPFRAME];

// src_ch1 和 src_ch3？→ 不存在！❌
```

**关键发现**：
- `src_ch0` 位于 **RXC0 (接收链路0)**
- `src_ch2` 位于 **RXC1 (接收链路1)** ← 注意标号是ch2不是ch1
- **没有定义 `src_ch1` 和 `src_ch3`**

### 2. 硬件接收链路拓扑

Surya芯片的实际硬件架构：

```
天线 → RF前端 → ADC
                  │
        ┌─────────┼─────────┐
        │         │         │
    RXC0链路   RXC1链路    (其他)
        │         │
        ▼         ▼
    src_ch0   src_ch2      ← 只有这2个数据流！
    (SRAM)    (SRAM)
        │         │
        ▼         ▼
    BBE#0    BBE#2  (正常映射)
    BBE#1    BBE#3  (无法正常映射 → 回退)
```

### 3. BBE处理引擎分配

4个BBE处理引擎与2个物理接收链路的不匹配：

```
理想情况（不存在）：
BBE#0 ← RXC0 ← src_ch0 ✓
BBE#1 ← RXC1 ← src_ch1 ✗ 不存在！
BBE#2 ← RXC0 ← src_ch2 ✓
BBE#3 ← RXC1 ← src_ch3 ✗ 不存在！

实际硬件：
┌────────────────────────────┐
│   RXC0 (接收链路0)         │
│   src_ch0[4096×4]          │
│   src_meta_ch0[4]          │
│          │                 │
│    ┌─────┴─────┐           │
│    ▼           ▼           │
│   BBE#0       BBE#1(回退)  │  ← 都读同一个src_ch0
└────────────────────────────┘

┌────────────────────────────┐
│   RXC1 (接收链路1)         │
│   src_ch2[4096×4]          │
│   src_meta_ch2[4]          │
│          │                 │
│    ┌─────┴─────┐           │
│    ▼           ▼           │
│   BBE#2       BBE#3(回退)  │  ← 都读同一个src_ch2
└────────────────────────────┘
```

---

## 数据重复问题分析

### 问题：会不会重复读取同一份数据？

**答案：是的，会重复！但这是设计决策，不是bug。**

### 重复读取的具体情形

```
时刻T0：
┌─────────────────────────────────────────┐
│  src_ch0 数据缓冲区（来自RXC0）         │
│  [Chirp#0] [Chirp#1] [Chirp#2] [Chirp#3]│
└─────────────────────────────────────────┘
    │           │
    DMA         DMA
    ▼           ▼
BBE#0的       BBE#1的
描述符#0      描述符#0
(读src_ch0)  (读src_ch0←回退)
    │           │
    ▼           ▼
┌──────────┐  ┌──────────┐
│chirp_    │  │chirp_    │
│buffer[0] │  │buffer[1] │
└──────────┘  └──────────┘

结果：两个DMA都从同一物理地址读取相同数据！
```

### 为什么这个设计是合理的？

#### **原因1：硬件约束无法改变**
- 芯片只有2条接收链路（RXC0、RXC1）
- SRAM只分配了2个源数据缓冲区（src_ch0、src_ch2）
- 不能凭空创造 `src_ch1` 和 `src_ch3`

#### **原因2：可能的实际用途**

**场景A：调试/测试模式**
```c
// 故意让BBE#1和BBE#3处理相同数据
// 用于验证处理算法的一致性
if (DMABufferConfigComplexMode(1, 2048, &desc_cnt) == NoError)
{
    // BBE#1 会处理与BBE#0相同的 src_ch0 数据
    // 如果输出结果相同 → 证明算法正确
}
```

**场景B：冗余/备份**
```c
// 同一数据由2个BBE独立处理，确保可靠性
// BBE#0 处理 src_ch0 → 输出到 result_ch0
// BBE#1 处理 src_ch0(回退) → 输出到 result_ch1
// 对比 result_ch0 和 result_ch1 确保无差异
```

**场景C：级联处理**
```c
// BBE#0 和 BBE#1 都读 src_ch0
// 但使用不同的圆形缓冲偏移和处理参数
// 实现不同的滤波或变换操作
```

#### **原因3：SYSMEM备选方案被注释**

代码注释明确说明：

```c
case 1:
    // Src address should be SYSMEM  ← 原设计意图
    // src_addr = (Addr_t) (&src_ch1[0]);
    // src_meta_addr = (Addr_t) &src_meta_ch1[0];
```

**"should be SYSMEM"** 表示：
- 原设计者想让 BBE#1/3 从**系统内存（SYSMEM）**读取数据
- 而不是从SRAM的RXC链路读取
- 但这需要额外的系统配置和内存映射
- 当前版本未完成这个功能 → 所以临时回退

---

## 内存布局真相

### 实际分配情况

```
高性能SRAM（受限资源）：
├─ .sram.rxc0_ds0
│  └─ src_ch0[4][4096] = 32KB ← RXC0链路的输入数据
├─ .sram.rxc0_ms0
│  └─ src_meta_ch0[4] = 32B ← RXC0的元数据
├─ .sram.rxc1_ds0
│  └─ src_ch2[4][4096] = 32KB ← RXC1链路的输入数据
└─ .sram.rxc1_ms0
   └─ src_meta_ch2[4] = 32B ← RXC1的元数据

低性能DRAM（充足空间）：
├─ .dram0.data
│  ├─ chirp_buffer[4][4096] ← 4个BBE的目标缓冲
│  ├─ chirp_metadata[4]
│  ├─ mag2[4][4096]
│  └─ ... (其他处理缓冲)
└─ .dram1.data
   └─ raw_fft_data, fft_eq, ...

关键：src_ch1 和 src_ch3 在这里完全没有定义！
```

---

## 通道回退的流程图

```
调用 DMABufferConfigComplexMode(bbe_id=1, fft_size=2048, ...)
          │
          ▼
    ┌─────────────┐
    │ switch(1)   │
    └─────────────┘
          │
          ▼
    ┌──────────────────────────────────────┐
    │ case 1:                              │
    │   // 评估情况                         │
    │   // 能找到 src_ch1 吗？ → NO ❌     │
    │   // 原始代码虽然存在但被注释掉了    │
    └──────────────────────────────────────┘
          │
          ▼
    ┌──────────────────────────────────────┐
    │ 决策：降级回退 (Graceful Fallback)   │
    │ → 使用 src_ch0 替代                   │
    │ → 使用 src_meta_ch0 替代             │
    └──────────────────────────────────────┘
          │
          ▼
    ┌──────────────────────────────────────┐
    │ 配置完成                              │
    │ BBE#1 的描述符现在指向:              │
    │ - 源: src_ch0 (与BBE#0相同)        │
    │ - 元数据: src_meta_ch0 (相同)      │
    │ 风险: 数据竞争或重复处理            │
    └──────────────────────────────────────┘
```

---

## 风险评估

### 🔴 高风险：数据竞争

```c
// 时刻T0，两个DMA同时读取同一地址
BBE#0: DMA读取 src_ch0[0] → chirp_buffer[0]
BBE#1: DMA读取 src_ch0[0] → chirp_buffer[1]
           ↑ 竞争！

// 如果src_ch0不是双缓冲或多缓冲，可能导致：
// - 数据损坏
// - 处理结果不一致
// - 中断丢失
```

### 🟡 中风险：业务逻辑错误

```c
// 调用者可能期望：
// BBE#1 处理独立的信号链
// 但实际上 BBE#1 处理的是 BBE#0 的副本

uint16_t desc_cnt[4];
DMABufferConfigComplexMode(0, 2048, &desc_cnt[0]);  // BBE#0 ← src_ch0
DMABufferConfigComplexMode(1, 2048, &desc_cnt[1]);  // BBE#1 ← src_ch0 (隐式重复!)
DMABufferConfigComplexMode(2, 2048, &desc_cnt[2]);  // BBE#2 ← src_ch2
DMABufferConfigComplexMode(3, 2048, &desc_cnt[3]);  // BBE#3 ← src_ch2 (隐式重复!)

// 调用者不知道BBE#1和BBE#3实际上在处理冗余数据
```

### 🟢 低风险：可能是功能设计

```c
// 某些场景下，有意让BBE#1和BBE#3处理相同数据：
// 1. 算法验证（运行同样处理，对比结果）
// 2. 红线冗余（关键数据处理2次确保正确）
// 3. 临时代码（等待SYSMEM映射完成）
```

---

## 解决方案

### 方案1：明确的错误处理（推荐 ⭐）

```c
Error_t DMABufferConfigComplexMode(uint32_t bbe_id, uint16_t fft_size,
    uint16_t *idma_descriptors_per_chirp)
{
    // ...
    
    switch (bbe_id)
    {
    case 0:
        src_addr = (Addr_t) (&src_ch0[0]);
        src_meta_addr = (Addr_t) &src_meta_ch0[0];
        break;
    
    case 1:
        // FIXME: src_ch1/src_meta_ch1 不存在于硬件
        // TODO: 实现SYSMEM映射方案，待完成
        return Error;  // ← 显式失败，而不是隐式回退！
        // 这样调用者会立即发现问题
    
    case 2:
        src_addr = (Addr_t) (&src_ch2[0]);
        src_meta_addr = (Addr_t) &src_meta_ch2[0];
        break;
    
    case 3:
        // FIXME: src_ch3/src_meta_ch3 不存在于硬件
        return Error;  // ← 显式失败
    
    default:
        assert(0);
        break;
    }
    
    // ...
}
```

### 方案2：实现SYSMEM映射（长期 ⭐⭐）

```c
// 在mem.h中添加SYSMEM缓冲区定义
extern int16_t __attribute__((aligned(32), section(".sysmem"))) 
    src_ch1_sysmem[CHIRPS_PER_CHIRPFRAME][MAG2_SIZE];
extern int16_t __attribute__((aligned(32), section(".sysmem"))) 
    src_ch3_sysmem[CHIRPS_PER_CHIRPFRAME][MAG2_SIZE];

// 在bbe_dma_adapter.c中使用真实地址
case 1:
    src_addr = (Addr_t) (&src_ch1_sysmem[0]);
    src_meta_addr = (Addr_t) &src_meta_ch1_sysmem[0];
    break;

case 3:
    src_addr = (Addr_t) (&src_ch3_sysmem[0]);
    src_meta_addr = (Addr_t) &src_meta_ch3_sysmem[0];
    break;
```

### 方案3：添加调试日志（临时 ⭐）

```c
case 1:
    // Src address should be SYSMEM
    // src_addr = (Addr_t) (&src_ch1[0]);
    // src_meta_addr = (Addr_t) &src_meta_ch1[0];
    
    // WARNING: Falling back to src_ch0 (BBE#1 will process same data as BBE#0)
    fprintf(stderr, "[DMA] WARNING: BBE#1 fallback to src_ch0 "
                    "(intended for SYSMEM)\n");
    src_addr = (Addr_t) (&src_ch0[0]);
    src_meta_addr = (Addr_t) &src_meta_ch0[0];
    break;
```

---

## 总结

| 问题 | 答案 | 根本原因 |
|------|------|--------|
| **为什么回退？** | 物理缓冲区不存在 | 硬件只有2条RXC链路，SRAM中未定义src_ch1/ch3 |
| **数据会重复吗？** | 会的 | BBE#0和BBE#1都被配置为读取src_ch0 |
| **这是bug吗？** | 可能是（需确认） | 取决于设计意图：<br>1. 临时workaround → 等待SYSMEM实现 <br>2. 有意设计 → 用于验证/冗余 |
| **应该如何修复？** | 明确处理 | 返回Error而不是隐式回退，或完整实现SYSMEM支持 |

**建议**：与硬件团队确认 `src_ch1` 和 `src_ch3` 是否计划在SYSMEM中实现，或者这个回退行为是否符合预期。
