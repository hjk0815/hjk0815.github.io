---
title: ptp     
data: 2026-06-05 14:39:21
tags: [ptp]             
categories: [embedded]              
description: ptp  
top_img: /image/jizi.png 
cover: /image/动漫少女.jpg   
---

# PTP (IEEE 1588)

PTP（IEEE 1588） 和 gPTP（IEEE 802.1AS） 都是用于网络设备间高精度时间同步的协议，精度都可以达到微秒甚至纳秒级。
都是通过测量报文在网络中的传输延迟（Propagation Delay）以及主从时钟的偏差（Offset），进而修正从时钟的时间。

PTP 的核心思想：在网络上选举出一个时间基准（Grandmaster），其它节点（Slave）通过交换带时间戳的报文，估算出自己相对主时钟的偏差和链路延迟，然后不断修正本地时钟，使整个网络的时间趋于一致。

整个系统由几个关键部件组成：

- **报文交换**：用一组特定报文（Sync / Follow_Up / Delay_Req / Delay_Resp 或 Pdelay 系列）携带或换取时间戳。
- **硬件时间戳**：在尽可能靠近物理层的位置打戳，消除软件协议栈带来的不确定延迟。
- **计算公式**：根据 4 个时间戳算出 offset 和链路延迟。
- **时钟伺服（servo）**：用一个 PI 控制器平滑地把本地时钟拉向主时钟，而不是硬跳变。
- **BMCA 状态机**：决定谁是主、谁是从，以及端口处于什么角色。

---

## 为什么需要硬件时间戳

时间同步的精度，本质上取决于"我们能多准确地知道报文离开/到达网线的那一刻"。

如果在应用层或内核协议栈打时间戳，报文从打戳点到真正上线（或从到达网卡到被读取）之间，要经过协议栈排队、中断延迟、调度抖动、DMA 等环节。这些延迟不是固定值，而是**抖动（jitter）**，无法通过校准消除，直接限制同步精度到几十微秒甚至毫秒级。

硬件时间戳把打戳点下移到 MAC/PHY，在 SOF（帧起始）经过某个固定参考点的瞬间，由硬件锁存当前 PTP 硬件时钟（PHC）的值。

### 时间戳的层次

| 类型 | 打戳位置 | 典型精度 | 抖动来源 |
| --- | --- | --- | --- |
| 软件时间戳 | 应用层/socket | 毫秒~几十微秒 | 协议栈、调度、中断 |
| 内核软件时间戳 | 驱动 `SO_TIMESTAMPING` | 几十微秒 | 中断、DMA |
| 硬件时间戳 | MAC | 微秒~百纳秒 | MAC 内部流水线（固定，可校准） |
| 硬件时间戳 | PHY | 纳秒级 | 几乎无（最靠近线缆） |

打戳点越靠近物理介质，剩余的不确定路径越短，精度越高。

### PHC（PTP Hardware Clock）

硬件时间戳的"尺子"是 PHC，它是网卡里一个独立的、可调速的计数器：

- 由本地晶振驱动，按固定步进累加，输出 64 位的秒 + 纳秒时间。
- 提供两类调节接口：
  - **频率调整（adjfine/adjfreq）**：改变每个时钟周期累加的步进量，相当于微调晶振快慢，用来纠正频率偏差（drift）。
  - **相位调整（adjtime）**：直接给计数器加/减一个偏移量，用来一次性纠正大的相位差。
- 在 Linux 下表现为 `/dev/ptp0` 等设备，由 `phc2sys`、`ptp4l` 等工具访问。

伺服调优的本质，就是不断地通过 adjfine（必要时 adjtime）去操作 PHC，让它和主时钟对齐。

---

## 同步报文与时间戳采集

### 报文角色

PTP 用一组事件报文（Event message，需要打硬件戳）和通用报文（General message，只携带数据）协作：

| 报文 | 类型 | 方向 | 作用 |
| --- | --- | --- | --- |
| Sync | Event | Master→Slave | 触发主侧发送时刻 t1 的采集 |
| Follow_Up | General | Master→Slave | 携带精确的 t1（两步模式） |
| Delay_Req | Event | Slave→Master | 触发从侧发送时刻 t3 的采集 |
| Delay_Resp | General | Master→Slave | 携带主侧接收时刻 t4 |
| Pdelay_Req/Resp/Resp_Follow_Up | Event/General | 邻居间 | gPTP 测量链路对等延迟 |

**一步（one-step）模式**：Sync 报文在发出的瞬间，由硬件直接把 t1 写进报文，省掉 Follow_Up。
**两步（two-step）模式**：Sync 先发出，硬件记下 t1，再用 Follow_Up 把 t1 告诉对端。硬件实现更简单，应用更广。

### 四个时间戳的采集（延迟请求-响应机制）

经典的 E2E（端到端）延迟测量，依靠 4 个时间戳 t1~t4：

```
   Master                         Slave
     |                              |
     |  Sync         t1 (发出)       |
     |----------------------------->| t2 (收到)
     |  Follow_Up [携带 t1]          |
     |----------------------------->|
     |                              |
     |              t3 (发出) Delay_Req|
     |<-----------------------------|
     | t4 (收到)                     |
     |  Delay_Resp [携带 t4]         |
     |----------------------------->|
     |                              |
   t1: 主发 Sync 的时刻（主时钟）
   t2: 从收 Sync 的时刻（从时钟）
   t3: 从发 Delay_Req 的时刻（从时钟）
   t4: 主收 Delay_Req 的时刻（主时钟）
```

- t1、t4 在主时钟域采集，t2、t3 在从时钟域采集。
- t1 / t4 通过 Follow_Up / Delay_Resp 报文传回从设备，从设备凑齐 4 个值后即可计算。

---

## 计算公式

### 基本假设

设：

- `offset`：从时钟相对主时钟的偏差（从快为正）。
- `delay`：主从之间的单向链路传播延迟。

关键假设是**链路对称**——上行和下行延迟相等。这一假设是 PTP 精度的主要误差来源之一。

### 推导

下行（Sync）：从收到的时刻 = 主发出时刻 + 单向延迟 + 偏差

$$ t2 = t1 + delay + offset $$

上行（Delay_Req）：主收到的时刻 = 从发出时刻 + 单向延迟 - 偏差

$$ t4 = t3 + delay - offset $$

两式相减、相加可解出：

$$ offset = \frac{(t2 - t1) - (t4 - t3)}{2} $$

$$ delay = \frac{(t2 - t1) + (t4 - t3)}{2} $$

得到 offset 后，从设备就知道自己快了/慢了多少，交给伺服去纠正。

### 频率偏差（drift）

仅靠单次 offset 还不够：两个晶振的频率本身有差异（ppm 级），即使某一刻对齐，下一刻又会漂开。通过相邻两次 Sync 估计频率比：

$$ ratio = \frac{t1_{n} - t1_{n-1}}{t2_{n} - t2_{n-1}} $$

`ratio` 偏离 1 的程度就是频率偏差，伺服用它来调整 PHC 的步进（adjfine）。

---

## 时钟伺服（Servo）与调优

算出 offset 后，不能直接把本地时钟"硬跳"到正确值——那会造成时间不连续（时间倒流、应用计时混乱）。伺服的任务是**平滑地、稳定地**把本地 PHC 拉向主时钟，同时抑制网络抖动带来的测量噪声。

主流实现（如 linuxptp 的 PI servo）是一个 PI（比例-积分）控制器。

### PI 控制器

每收到一次 offset 测量，伺服输出一个频率修正量 `freq_adj`（单位 ppb，下发给 adjfine）：

$$ freq\_adj_n = K_p \cdot offset_n + K_i \cdot \sum_{i} offset_i $$

- **比例项 Kp**：对当前 offset 立即响应，决定收敛速度。Kp 越大反应越快，但过大会震荡。
- **积分项 Ki**：累积历史 offset，消除稳态误差（晶振固有频偏导致的常值漂移）。
- 输出的是**频率修正**而非相位修正，所以系统会逐步收敛、不抖动。

linuxptp 默认采用自适应的 PI 增益，根据 Sync 间隔自动整定 Kp、Ki，也可手动覆盖。

### 收敛的两个阶段

```
   offset
     |\
     | \         ① 大相位差：用 adjtime/step 一次性跳变
     |  \____    （首次或 offset 超过 step_threshold 时）
     |       \__
     |          \____  ② 频率锁定：PI 持续微调 adjfine
     |               \________________  稳态：offset 在噪声带内抖动
     +-------------------------------------> 时间
```

1. **粗对齐（step）**：首次同步或 offset 超过门限（`step_threshold`）时，直接 step 时钟，快速进入捕获范围。
2. **细锁定（lock）**：进入门限内后只用频率调整平滑收敛，最终 offset 稳定在亚微秒/纳秒级噪声带内。

### 调优要点

| 参数 | 作用 | 调大 | 调小 |
| --- | --- | --- | --- |
| `step_threshold` | step/adjust 切换门限 | 更少跳变，收敛慢 | 频繁跳变，时间不连续 |
| `sync interval` | Sync 报文间隔 | 抗抖动好，跟踪慢 | 跟踪快，受抖动影响大 |
| Kp（比例增益） | 收敛速度 | 快但易震荡 | 稳但慢 |
| Ki（积分增益） | 消除稳态误差 | 收敛快、可能超调 | 残余漂移消除慢 |

调优的目标：在**收敛速度**和**稳态抖动**之间取得平衡。

- 抖动大、震荡 → 减小 Kp/Ki，或增大 Sync 间隔做更多平滑。
- 收敛太慢、跟不上温漂 → 增大增益，或缩短 Sync 间隔。
- 偶发大 offset 毛刺 → 加入异常值过滤（outlier filter），防止单个坏样本扰动伺服。

实践中还要关注：测量值要先经过 **filter**（如中值滤波/移动平均）去除路径抖动，再喂给 PI；`phc2sys` 把 PHC 同步到系统 CLOCK_REALTIME 时，也是一套独立的 PI servo。

---

# gPTP (IEEE 802.1AS)

gPTP 是 PTP 的一个"特定行业精简/定制版"（主要用于车载以太网、工业 TSN 和音视频 AVB 领域）。它是 IEEE 1588 的一个 **profile**，对协议做了裁剪和强约束，以保证在时间敏感网络（TSN）里的确定性和互操作性。

## gPTP 与通用 PTP 的关键区别

| 维度 | 通用 PTP (1588) | gPTP (802.1AS) |
| --- | --- | --- |
| 延迟测量 | E2E（端到端，Delay_Req/Resp） | **P2P（对等，Pdelay）** |
| 中间节点 | 透明时钟 / 边界时钟均可 | 必须是 **时间感知网桥**（每跳都参与） |
| 传输层 | UDP/IPv4、IPv6、Ethernet | 仅 **Layer 2 以太网**（多播） |
| 时钟选举 | 完整 BMCA | 简化 BMC（Best Master Clock） |
| 链路要求 | 任意 | 每条链路都要支持 gPTP，全程"时间感知" |
| 目标 | 通用高精度同步 | 有界、确定性的网络同步 |

## P2P（对等延迟）机制

gPTP 不测端到端延迟，而是测**每一跳相邻设备之间**的链路延迟（Peer Delay）。每个端口持续地和直连邻居用 Pdelay 报文交换 4 个时间戳：

```
   Initiator                      Responder
     |   Pdelay_Req   t1            |
     |----------------------------->| t2
     |   Pdelay_Resp [t2]  t3       |
     |<-----------------------------| (记录 t3)
     |   Pdelay_Resp_Follow_Up [t3] |
     |<-----------------------------|
     | t4 (收到 Resp)               |
```

链路延迟（假设对称）：

$$ meanLinkDelay = \frac{(t4 - t1) - (t3 - t2)}{2} $$

同时还能算出邻居间的频率比 `neighborRateRatio`，用来把时间戳换算到本地时钟域。

## 逐跳的"剩余时间"传递

Sync 报文沿路径逐跳转发，每个时间感知网桥在转发时会累加：

- **residenceTime（驻留时间）**：报文在本网桥内部停留的时长。
- **链路延迟**：本跳的 meanLinkDelay。

这两者累加进 Sync 的 **correctionField**，于是末端 Slave 收到的 Sync 里，correctionField 等于从 Grandmaster 到自己沿途所有链路延迟 + 驻留时间之和。最终偏差计算：

$$ offset = t_{rx} - (preciseOriginTimestamp + correctionField) $$

这样无论中间隔了多少跳，从设备都能精确还原 Grandmaster 的时间。

---

# 状态机

PTP 系统里有几个并行运转的状态机。最重要的两个：决定"谁当主"的 **BMCA / 端口状态机**，和决定"时钟怎么跟"的 **伺服状态机**。gPTP 还有一个独立的 **Pdelay 状态机**。

## 端口状态机（BMCA 驱动）

每个 PTP 端口都跑一个状态机。BMCA（Best Master Clock Algorithm）周期性地比较本端口收到的 Announce 报文（携带各时钟的优先级、类、精度等数据集），决定本端口应该是 MASTER（对外发时间）、SLAVE（对内收时间）还是 PASSIVE（闭嘴）。

```
                    ┌──────────────┐
          power on  │ INITIALIZING │
         ─────────► │  (初始化)     │
                    └──────┬───────┘
                           │ 初始化完成
                           ▼
                    ┌──────────────┐  fault detected
            ┌──────►│   LISTENING  │◄──────────────┐
            │       │ (监听Announce)│               │
            │       └──────┬───────┘               │
            │              │ BMCA 决策              │
            │      ┌───────┼────────┐              │
            │      ▼       ▼        ▼              │
            │ ┌─────────┐┌──────┐┌─────────┐       │
   recommend│ │PRE_MASTER││SLAVE ││ PASSIVE │       │
    state变化│ │(待转主)  ││(从)  ││(旁路)   │       │
            │ └────┬────┘└──┬───┘└────┬────┘       │
            │      │ 限时到  │         │            │
            │      ▼        │         │            │
            │ ┌─────────┐   │         │            │
            └─┤ MASTER  │   │         │            │
              │ (主/发) │   │         │            │
              └────┬────┘   │         │            │
                   │        │         │            │
                   └────────┴─────────┴────────────┘
                        FAULTY (故障) ─► 回到 INITIALIZING
```

各状态含义：

| 状态 | 含义 |
| --- | --- |
| INITIALIZING | 端口初始化，不收发 PTP 报文 |
| LISTENING | 监听 Announce，等待确定角色 |
| PRE_MASTER | 即将成为 Master 的过渡态（防抖） |
| MASTER | 本端口是主，周期发送 Sync/Announce |
| PASSIVE | 网络中已有更优主，本口保持沉默（避免成环） |
| SLAVE | 本端口是从，接收并同步到上游 Master |
| UNCALIBRATED | 刚选为 Slave，伺服尚未锁定的中间态 |
| FAULTY | 端口故障，退出协议 |

**SLAVE 的进入路径**通常是：LISTENING → UNCALIBRATED → SLAVE。UNCALIBRATED 表示已确定跟随某主，但本地时钟还没收敛，伺服锁定后转为 SLAVE。

## BMCA 决策（谁更优）

BMCA 按数据集字段逐级比较，选出"最优主时钟"：

```
priority1  →  clockClass  →  clockAccuracy  →  offsetScaledLogVariance
   →  priority2  →  clockIdentity（兜底，保证唯一）
```

数值越小越优。`priority1` 是人工干预的最高优先级旋钮（比如强制指定某台设备做 Grandmaster），`clockIdentity` 作为最终 tie-breaker 保证一定能选出唯一胜者。

## 伺服状态机

伺服（servo）内部也是一个状态机，描述时钟从"完全没对齐"到"锁定跟踪"的过程。以 linuxptp 的 PI servo 为例：

```
        首次 offset
   ┌─────────────────┐
   │   UNLOCKED      │  没有有效频率估计，
   │  (未锁定)        │  靠 step 直接跳变对齐
   └────────┬────────┘
            │ 已获得 1~2 个样本，
            │ 可估算频率
            ▼
   ┌─────────────────┐   offset 超 step_threshold
   │  JUST_ACQUIRED   │◄──────────────────────┐
   │  (刚捕获/采样中)  │                        │
   └────────┬────────┘                        │
            │ 样本足够，PI 接管                  │
            ▼                                  │
   ┌─────────────────┐                         │
   │     LOCKED       │─────────────────────────┘
   │  (锁定/PI跟踪)    │   offset 在门限内：
   │                  │   仅用 adjfine 平滑微调
   └─────────────────┘
```

| 状态 | 行为 |
| --- | --- |
| UNLOCKED | 无频率估计；用 step（adjtime）一次性把相位拉到位 |
| JUST_ACQUIRED | 已采到样本、算出初步频率，过渡态，准备交给 PI |
| LOCKED | PI 控制器稳定运行，只做频率微调（adjfine），offset 收敛在噪声带内 |

触发跳回（重新 step）的条件：offset 突然超过 `step_threshold`（比如链路切换、主时钟跳变、长时间失联后重连）。这时直接 step 比慢慢调更快回到捕获范围。

## gPTP 的 Pdelay 状态机

gPTP 每个端口为链路延迟测量维护一个独立的 Pdelay 状态机（IEEE 802.1AS MDPdelayReq）：

```
      ┌──────────────────┐
      │  NOT_ENABLED      │
      └────────┬─────────┘
               │ 端口使能 asCapable 测量
               ▼
      ┌──────────────────┐  发出 Pdelay_Req，启动定时
   ┌─►│  SEND_PDELAY_REQ  │
   │  └────────┬─────────┘
   │           │ 周期到 / 收到完整响应
   │           ▼
   │  ┌──────────────────┐
   │  │ WAITING_FOR_RESP  │  等 Pdelay_Resp(+Follow_Up)
   │  └────────┬─────────┘
   │     ┌─────┴───────┐
   │收齐 │             │ 超时/丢失/延迟异常
   │t1-t4│             ▼
   │     │      ┌──────────────┐
   │     │      │  lostResponses│ 连续丢失超阈值
   │     │      │   ++          │ → asCapable=false
   │     │      └──────┬───────┘
   │     ▼             │
   │ ┌──────────────┐  │
   └─┤ 计算LinkDelay │  │
     │ rateRatio    │◄─┘ 重新发起
     │ asCapable=T  │
     └──────────────┘
```

要点：

- 连续多次 Pdelay 丢失或超时 → 把端口标记为 `asCapable = false`，意味着这条链路不再被认为支持 gPTP，Sync 不再沿此口传播。
- meanLinkDelay 超过门限（`neighborPropDelayThresh`）也会判定链路不合格，这是 802.1AS 保证"全程时间感知"的硬约束。
- 只有 asCapable 的端口，才参与 BMCA 和 Sync 转发。

---

# 小结

- **硬件时间戳**把打戳点下移到 MAC/PHY，消除协议栈抖动，是高精度同步的物理基础；PHC 提供频率（adjfine）和相位（adjtime）两类调节接口。
- **计算**靠 4 个时间戳在链路对称假设下解出 `offset = ((t2−t1)−(t4−t3))/2` 和 `delay = ((t2−t1)+(t4−t3))/2`。
- **伺服**用 PI 控制器把 offset 平滑收敛——大偏差时 step、小偏差时频率微调，调优在收敛速度与稳态抖动间权衡。
- **gPTP** 是 1588 的 TSN profile：P2P 逐跳测延迟、每跳累加 correctionField、纯 L2、简化 BMC，换取车载/工业网络的确定性。
- **状态机**：端口状态机（BMCA 驱动选主/从）、伺服状态机（unlocked→locked）、gPTP 的 Pdelay 状态机（维护 asCapable）协同工作，构成完整的时间同步系统。



