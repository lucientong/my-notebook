# ARM 架构深入

---

## 📑 目录

### 架构基础
1. [ARM 架构演进](#1-arm-架构演进)
2. [AArch64 寄存器模型](#2-aarch64-寄存器模型)
3. [指令集与编码](#3-指令集与编码)

### 系统架构
4. [异常级别 EL0-EL3](#4-异常级别-el0-el3)
5. [内存模型与页表](#5-内存模型与页表)
6. [中断与 GIC](#6-中断与-gic)

### 高级特性
7. [ARM 虚拟化](#7-arm-虚拟化)
8. [TrustZone 安全架构](#8-trustzone-安全架构)
9. [SIMD：NEON/SVE/SME](#9-simdneonsve)
10. [面试题自查](#10-面试题自查)

---

## 1. ARM 架构演进

### 1.1 ARM 架构版本

| 版本 | 年代 | 位宽 | 关键特性 |
|------|------|------|---------|
| ARMv7 | 2007 | 32位 | Cortex-A 系列，NEON |
| ARMv8.0 | 2011 | 64位 | AArch64，Crypto扩展 |
| ARMv8.1 | 2016 | 64位 | 原子操作增强，VHE |
| ARMv8.2 | 2017 | 64位 | SVE，FP16 |
| ARMv8.3 | 2018 | 64位 | 指针认证 PAC |
| ARMv8.4 | 2019 | 64位 | 嵌套虚拟化 |
| ARMv9.0 | 2021 | 64位 | SVE2，机密计算 |

### 1.2 执行状态

```
ARM 执行状态：

AArch64 (64位)
├── 64位通用寄存器
├── A64 指令集
└── 支持ARMv8+所有特性

AArch32 (32位兼容)
├── 32位通用寄存器
├── A32 (ARM) 指令集
├── T32 (Thumb) 指令集
└── 向后兼容 ARMv7

状态切换：
- 只能在异常级别改变时切换
- EL0/EL1 可以是 AArch32 或 AArch64
- EL2/EL3 通常只支持 AArch64
```

### 1.3 ARM 服务器生态

| 厂商 | 产品 | 核心数 | 场景 |
|------|------|--------|------|
| AWS | Graviton3/4 | 64-96 | 云计算 |
| Ampere | Altra Max | 128 | 云原生 |
| 华为 | 鲲鹏 920 | 64 | 通用服务器 |
| NVIDIA | Grace | 72 | AI/HPC |

---

## 2. AArch64 寄存器模型

### 2.1 通用寄存器

```
AArch64 通用寄存器（31个64位）：

64-bit    32-bit    说明
┌─────────┬─────────┬────────────────────────┐
│   X0    │   W0    │  参数/返回值           │
│   X1    │   W1    │  参数/返回值           │
│  ...    │  ...    │                        │
│   X7    │   W7    │  参数（共8个）         │
├─────────┼─────────┼────────────────────────┤
│   X8    │   W8    │  间接返回值地址        │
├─────────┼─────────┼────────────────────────┤
│  X9-X15 │ W9-W15  │  临时寄存器（调用者保存）│
├─────────┼─────────┼────────────────────────┤
│ X16-X17 │W16-W17  │  IP0/IP1（链接器用）   │
│   X18   │  W18    │  平台保留              │
├─────────┼─────────┼────────────────────────┤
│ X19-X28 │W19-W28  │  被调用者保存          │
├─────────┼─────────┼────────────────────────┤
│   X29   │  W29    │  帧指针 FP             │
│   X30   │  W30    │  链接寄存器 LR         │
└─────────┴─────────┴────────────────────────┘

SP  - 栈指针（每个异常级别独立）
PC  - 程序计数器（不可直接访问）
XZR/WZR - 零寄存器（写入忽略，读取为0）
```

### 2.2 特殊寄存器

```
系统寄存器（MRS/MSR 访问）：

PSTATE 组件：
├── N, Z, C, V      - 条件标志
├── D, A, I, F      - 中断掩码
├── CurrentEL       - 当前异常级别
├── SP              - 栈指针选择
└── nRW             - 执行状态 (0=AArch64)

关键系统寄存器：
  SCTLR_ELn     - 系统控制
  TCR_ELn       - 翻译控制
  TTBR0/1_ELn   - 翻译表基址
  ESR_ELn       - 异常综合寄存器
  FAR_ELn       - 错误地址
  VBAR_ELn      - 向量基址
  MAIR_ELn      - 内存属性
```

### 2.3 调用约定 (AAPCS64)

```
ARM64 调用约定：

参数传递：
  整数：X0-X7（前8个）
  浮点：V0-V7（前8个）
  更多参数：压栈

返回值：
  整数：X0 (128位用 X0+X1)
  浮点：V0

调用者保存：X0-X18, V0-V7, V16-V31
被调用者保存：X19-X28, V8-V15

栈对齐：16字节
```

---

## 3. 指令集与编码

### 3.1 A64 指令格式

```
A64 指令编码（固定32位）：

数据处理（寄存器）：
┌─────┬─────┬─────┬─────┬─────┬─────┬─────┐
│ sf  │ op  │  S  │shift│ Rm  │imm6 │ Rn  │ Rd  │
│ 1位 │ 2位 │ 1位 │ 2位 │ 5位 │ 6位 │ 5位 │ 5位 │
└─────┴─────┴─────┴─────┴─────┴─────┴─────┘

sf: 0=32位，1=64位
op: 操作类型
Rm, Rn: 源寄存器
Rd: 目标寄存器

示例：ADD X0, X1, X2
编码：1 00 01011 00 0 00010 000000 00001 00000
      sf op       shift Rm    imm6   Rn    Rd
```

### 3.2 常用指令

```assembly
# 数据传送
MOV X0, #100         // X0 = 100
MOV X0, X1           // X0 = X1
LDR X0, [X1]         // X0 = *X1
STR X0, [X1]         // *X1 = X0
LDP X0, X1, [X2]     // Load Pair
STP X0, X1, [X2]     // Store Pair

# 算术
ADD X0, X1, X2       // X0 = X1 + X2
ADD X0, X1, #10      // X0 = X1 + 10
SUB X0, X1, X2, LSL #2  // X0 = X1 - (X2 << 2)
MUL X0, X1, X2       // X0 = X1 * X2
MADD X0, X1, X2, X3  // X0 = X1 * X2 + X3

# 逻辑
AND X0, X1, X2
ORR X0, X1, X2
EOR X0, X1, X2       // XOR

# 分支
B label              // 无条件跳转
BL func              // 函数调用（保存返回地址到LR）
BR X0                // 寄存器跳转
RET                  // 返回（等价于 BR LR）
CBZ X0, label        // Compare and Branch if Zero
CBNZ X0, label       // Compare and Branch if Not Zero

# 条件
CMP X0, X1           // 设置标志
B.EQ label           // 相等时跳转
B.NE label           // 不等时跳转
B.LT/GT/LE/GE        // 有符号比较
B.LO/HI/LS/HS        // 无符号比较
CSEL X0, X1, X2, EQ  // Conditional Select

# 系统
SVC #0               // Supervisor Call (系统调用)
HVC #0               // Hypervisor Call
SMC #0               // Secure Monitor Call
MRS X0, SCTLR_EL1    // Move from System Register
MSR SCTLR_EL1, X0    // Move to System Register
```

---

## 4. 异常级别 EL0-EL3

### 4.1 异常级别概述

```
ARM 异常级别：

EL3 ─── Secure Monitor (最高权限)
│       └── ATF/OP-TEE 的 Secure Monitor
│
EL2 ─── Hypervisor
│       └── KVM, Xen
│
EL1 ─── OS Kernel
│       └── Linux, Windows
│
EL0 ─── User Application (最低权限)
        └── 普通应用程序

权限递增：EL0 < EL1 < EL2 < EL3

安全状态：
┌─────────────────────────────────────────────┐
│  Secure World       │   Non-secure World    │
├─────────────────────┼───────────────────────┤
│  Secure EL0 (可选)  │   EL0 (应用)          │
│  Secure EL1 (TEE OS)│   EL1 (Normal OS)     │
│                     │   EL2 (Hypervisor)    │
├─────────────────────┴───────────────────────┤
│               EL3 (Secure Monitor)          │
└─────────────────────────────────────────────┘
```

### 4.2 异常类型与向量表

```
异常类型：

同步异常（Synchronous）：
├── SVC (EL0→EL1 系统调用)
├── HVC (EL1→EL2 Hypervisor调用)
├── SMC (→EL3 安全监控调用)
├── 数据/指令中止
├── 未对齐访问
└── 未定义指令

异步异常（Asynchronous）：
├── IRQ (普通中断)
├── FIQ (快速中断)
└── SError (系统错误)

异常向量表（每个EL独立，VBAR_ELn指向）：
┌─────────────────────────────────────────────┐
│ Offset │ 来源异常级别  │ 类型                │
├────────┼───────────────┼─────────────────────┤
│ 0x000  │ 当前EL，SP_EL0│ Synchronous         │
│ 0x080  │ 当前EL，SP_EL0│ IRQ                 │
│ 0x100  │ 当前EL，SP_EL0│ FIQ                 │
│ 0x180  │ 当前EL，SP_EL0│ SError              │
├────────┼───────────────┼─────────────────────┤
│ 0x200  │ 当前EL，SP_ELx│ Synchronous         │
│ 0x280  │ 当前EL，SP_ELx│ IRQ                 │
│ 0x300  │ 当前EL，SP_ELx│ FIQ                 │
│ 0x380  │ 当前EL，SP_ELx│ SError              │
├────────┼───────────────┼─────────────────────┤
│ 0x400  │ 低EL，AArch64 │ Synchronous         │
│ 0x480  │ 低EL，AArch64 │ IRQ                 │
│ 0x500  │ 低EL，AArch64 │ FIQ                 │
│ 0x580  │ 低EL，AArch64 │ SError              │
├────────┼───────────────┼─────────────────────┤
│ 0x600  │ 低EL，AArch32 │ Synchronous         │
│ ...    │ ...           │ ...                 │
└────────┴───────────────┴─────────────────────┘
```

---

## 5. 内存模型与页表

### 5.1 虚拟地址空间

```
AArch64 虚拟地址空间（48位）：

0xFFFF_FFFF_FFFF_FFFF ┌─────────────────────┐
                      │   Kernel Space      │
                      │   (TTBR1_EL1)       │
0xFFFF_0000_0000_0000 ├─────────────────────┤
                      │                     │
                      │   Unused / Hole     │
                      │                     │
0x0000_FFFF_FFFF_FFFF ├─────────────────────┤
                      │   User Space        │
                      │   (TTBR0_EL1)       │
0x0000_0000_0000_0000 └─────────────────────┘

地址位63-48/52必须全0或全1（符号扩展）
用户空间：TTBR0 (VA[63]=0)
内核空间：TTBR1 (VA[63]=1)
```

### 5.2 页表格式

```
4级页表（4KB页，48位VA）：

虚拟地址分解：
┌────────┬────────┬────────┬────────┬─────────────┐
│[47:39] │[38:30] │[29:21] │[20:12] │   [11:0]    │
│  L0    │   L1   │   L2   │   L3   │   Offset    │
│(9位)   │ (9位)  │ (9位)  │ (9位)  │  (12位)     │
└───┬────┴───┬────┴───┬────┴───┬────┴─────────────┘
    │        │        │        │
TTBR→[L0表]─→[L1表]─→[L2表]─→[L3表]─→物理页

页表项格式（64位）：
[63]    - NSTable/NS (安全属性)
[62:59] - 保留
[58:55] - PXNTable, UXNTable (执行权限继承)
[54:53] - 保留
[52]    - Contiguous
[51:48] - 保留（或 OA[51:48] 用于52位PA）
[47:12] - 下一级表地址 / 输出地址
[11]    - nG (non-Global)
[10]    - AF (Access Flag)
[9:8]   - SH (Shareability)
[7:6]   - AP (Access Permission)
[5]     - NS (Non-Secure)
[4:2]   - AttrIdx (MAIR索引)
[1]     - 类型 (0=Block, 1=Table/Page)
[0]     - Valid

大页支持：
- 1GB (L1 Block)
- 2MB (L2 Block)
- 4KB (L3 Page)
```

### 5.3 内存属性

```
MAIR_ELn 定义内存属性（8组，每组8位）：

常见配置：
┌─────────────────────────────────────────────────┐
│ AttrIdx │ 编码    │ 含义                        │
├─────────┼─────────┼─────────────────────────────┤
│    0    │ 0x00    │ Device-nGnRnE (强序设备)   │
│    1    │ 0x04    │ Device-nGnRE              │
│    2    │ 0x44    │ Normal, Non-Cacheable     │
│    3    │ 0xFF    │ Normal, Write-Back Cacheable│
└─────────┴─────────┴─────────────────────────────┘

内存类型：
- Device：设备内存，严格顺序
- Normal：普通内存，可缓存可重排

共享性 (SH)：
- Non-shareable (00)
- Outer Shareable (10)：所有观察者
- Inner Shareable (11)：同一内部域
```

---

## 6. 中断与 GIC

### 6.1 GIC 架构

```
GIC (Generic Interrupt Controller) v3/v4：

┌─────────────────────────────────────────────────────────────┐
│                      GIC Distributor                        │
│  - 全局中断管理                                             │
│  - 中断优先级、启用/禁用                                    │
│  - 中断路由到 CPU                                           │
└────────────────────────┬────────────────────────────────────┘
                         │
         ┌───────────────┼───────────────┐
         ▼               ▼               ▼
┌─────────────┐  ┌─────────────┐  ┌─────────────┐
│Redistributor│  │Redistributor│  │Redistributor│
│   (Core 0)  │  │   (Core 1)  │  │   (Core N)  │
└──────┬──────┘  └──────┬──────┘  └──────┬──────┘
       │                │                │
       ▼                ▼                ▼
┌─────────────┐  ┌─────────────┐  ┌─────────────┐
│ CPU Interface│  │ CPU Interface│  │ CPU Interface│
│   (Core 0)  │  │   (Core 1)  │  │   (Core N)  │
└─────────────┘  └─────────────┘  └─────────────┘
```

### 6.2 中断类型

| 类型 | ID范围 | 说明 |
|------|--------|------|
| SGI | 0-15 | 软件生成，核间通信 |
| PPI | 16-31 | 私有外设，如定时器 |
| SPI | 32-1019 | 共享外设，外部设备 |
| LPI | 8192+ | 消息中断 (GICv3+) |

---

## 7. ARM 虚拟化

### 7.1 VHE (Virtualization Host Extensions)

```
传统虚拟化 vs VHE：

传统（Type 2）：
EL2 ─── KVM (Hypervisor)     ← 独立代码
EL1 ─── Host Linux + Guest   ← 频繁切换
EL0 ─── Applications

VHE 启用后：
EL2 ─── Host Linux + KVM     ← 内核直接运行在 EL2
EL1 ─── Guest Kernel
EL0 ─── Guest Applications

优势：
- Host 内核无需 EL1/EL2 切换
- 减少虚拟化开销
- 寄存器重命名（SCTLR_EL1 → SCTLR_EL2 等）
```

### 7.2 Stage-2 页表

```
两阶段地址转换：

Guest VA → Guest PA (Stage-1, Guest 页表)
        → Host PA  (Stage-2, Hypervisor 控制)

┌───────────┐      Stage-1      ┌───────────┐
│ Guest VA  │ ─────────────────►│ Guest PA  │
│(Guest OS) │    Guest 页表     │   (IPA)   │
└───────────┘                   └─────┬─────┘
                                      │
                                      │ Stage-2
                                      │ Hypervisor 页表
                                      ▼
                                ┌───────────┐
                                │  Host PA  │
                                │(物理内存) │
                                └───────────┘

Stage-2 页表：
- VTTBR_EL2 指向
- 控制 Guest 可见的物理内存
- 实现内存隔离
```

---

## 8. TrustZone 安全架构

### 8.1 TrustZone 概述

```
TrustZone 硬件隔离：

┌─────────────────────────────────────────────────────────────┐
│                    Normal World                             │
│  ┌─────────────────────────────────────────────────────┐   │
│  │  Rich OS (Linux/Android)                            │   │
│  │  - 普通应用                                         │   │
│  │  - 网络、存储、UI                                   │   │
│  └─────────────────────────────────────────────────────┘   │
└─────────────────────────────┬───────────────────────────────┘
                              │ SMC (Secure Monitor Call)
┌─────────────────────────────┼───────────────────────────────┐
│                    EL3 Secure Monitor                       │
│                    (ATF/OP-TEE)                             │
└─────────────────────────────┬───────────────────────────────┘
                              │
┌─────────────────────────────┼───────────────────────────────┐
│                    Secure World                             │
│  ┌─────────────────────────────────────────────────────┐   │
│  │  Trusted OS (OP-TEE)                                │   │
│  │  - 密钥管理                                         │   │
│  │  - DRM                                              │   │
│  │  - 安全支付                                         │   │
│  └─────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────┘

硬件隔离：
- CPU：NS 位标记当前世界
- 内存：TZASC 控制安全内存访问
- 外设：可配置为安全/非安全
```

### 8.2 安全状态切换

```
世界切换流程：

Normal World (EL1)
     │
     │ SMC #0
     ▼
Secure Monitor (EL3)
     │
     │ 保存 Normal World 上下文
     │ 恢复 Secure World 上下文
     │ 设置 SCR_EL3.NS = 0
     ▼
Secure World (S-EL1)
     │
     │ 执行安全操作
     │ SMC 返回
     ▼
Secure Monitor (EL3)
     │
     │ 保存 Secure World 上下文
     │ 恢复 Normal World 上下文
     │ 设置 SCR_EL3.NS = 1
     ▼
Normal World (EL1)
```

---

## 9. SIMD：NEON/SVE/SME

### 9.1 NEON

```
NEON (Advanced SIMD)：

128位向量寄存器 V0-V31
├── 可视为 16×8bit, 8×16bit, 4×32bit, 2×64bit
└── 支持整数和浮点

示例：
// 4个32位整数相加
ADD V0.4S, V1.4S, V2.4S

// 8个16位整数乘法
MUL V0.8H, V1.8H, V2.8H

// 向量加载
LD1 {V0.16B}, [X0]
```

### 9.2 SVE (Scalable Vector Extension)

```
SVE 特点：

- 可变向量长度：128-2048位
- 编译一次，适配不同硬件
- 谓词寄存器支持条件执行

向量长度寄存器 VL：由硬件实现决定
谓词寄存器 P0-P15：每位对应一个向量元素

// SVE 示例
WHILELT P0.S, X0, X1      // 生成谓词
LD1W Z0.S, P0/Z, [X2]     // 谓词加载
ADD Z0.S, P0/M, Z0.S, Z1.S // 谓词加法
ST1W Z0.S, P0, [X3]       // 谓词存储

SVE vs NEON：
- SVE：可变长度，适合 HPC
- NEON：固定 128位，兼容性好
```

### 9.3 SME (Scalable Matrix Extension)

```
SME (ARMv9.2+)：

用于矩阵运算加速（AI/ML）

ZA 矩阵存储：SVL × SVL 的二维数组
SMSTART/SMSTOP：启用/停用 SME

// 外积累加
FMOPA ZA0.S, P0/M, Z0.S, Z1.S
// ZA0 += outer_product(Z0, Z1)
```

---

## 10. 面试题自查

**Q1: ARM 的异常级别 EL0-EL3 分别运行什么软件？**

EL0 运行用户应用；EL1 运行操作系统内核；EL2 运行 Hypervisor；EL3 运行 Secure Monitor（可信固件）。权限 EL0 < EL1 < EL2 < EL3。

**Q2: ARM 和 x86 虚拟化的主要区别？**

ARM 使用异常级别（EL2 专用于 Hypervisor），x86 使用 VMX Root/Non-root 模式。ARM Stage-2 页表类似 x86 EPT。ARM VHE 允许 Host 内核直接运行在 EL2。

**Q3: 什么是 TrustZone？有什么应用场景？**

TrustZone 是 ARM 的硬件安全隔离技术，将系统分为 Secure World 和 Normal World，通过 NS 位区分。应用：密钥存储、DRM、安全支付、生物识别。

**Q4: ARM 页表和 x86 页表的主要区别？**

ARM 使用 TTBR0（用户空间）和 TTBR1（内核空间）两个页表基址寄存器；x86 只有 CR3。ARM 页表项包含内存属性索引（MAIR），x86 直接在 PTE 中编码。

**Q5: 什么是 VHE？解决了什么问题？**

Virtualization Host Extensions，允许 Host 内核直接运行在 EL2，避免 Type-2 Hypervisor 中 Host 内核在 EL1 和 Hypervisor 在 EL2 之间频繁切换，降低虚拟化开销。

**Q6: ARM 的 GIC 有哪些组件？SGI/PPI/SPI 分别是什么？**

GIC 包含 Distributor（全局管理）、Redistributor（每核）、CPU Interface。SGI（0-15）是软件生成的核间中断；PPI（16-31）是核私有外设中断；SPI（32+）是共享外设中断。

**Q7: ARM 内存模型（弱序）和 x86（TSO）的区别？**

ARM 弱序允许几乎所有内存操作重排，需要显式屏障（DMB/DSB/ISB）保证顺序；x86 TSO 保证 Store-Store 和 Load-Load 有序，只有 Store-Load 可能重排。

**Q8: NEON 和 SVE 的区别？**

NEON 固定 128位向量长度，SVE 支持 128-2048位可变长度。SVE 有谓词寄存器支持条件执行，编译一次可在不同 VL 的硬件上运行。

**Q9: ARM Stage-2 页表的作用是什么？**

实现 Guest 物理地址（IPA）到 Host 物理地址（PA）的第二阶段转换，由 Hypervisor 控制，实现 Guest 内存隔离和物理内存虚拟化。

**Q10: ARM 的 SVC/HVC/SMC 指令分别用于什么？**

SVC：Supervisor Call，EL0→EL1 系统调用；HVC：Hypervisor Call，EL1→EL2 Hypervisor 服务；SMC：Secure Monitor Call，任意 EL→EL3 安全服务。

**Q11: ARM 大小核架构（big.LITTLE/DynamIQ）是什么？**

将高性能大核（如 Cortex-A77）和低功耗小核（如 Cortex-A55）集成在同一 SoC，操作系统根据负载动态迁移任务，平衡性能和功耗。DynamIQ 允许混合核心共享 L3 Cache。

**Q12: ARM 指针认证（PAC）的作用？**

Pointer Authentication Code，在指针高位存储签名，防止攻击者伪造返回地址或函数指针。ARMv8.3+ 支持，是防 ROP 攻击的硬件措施。

**Q13: ARM64 有多少个通用寄存器？调用约定中参数如何传递？**

31个64位通用寄存器（X0-X30）。AAPCS64 中前8个整数参数通过 X0-X7 传递，前8个浮点参数通过 V0-V7 传递，返回值在 X0（整数）或 V0（浮点）。

**Q14: 什么是 ARM 的内存属性（MAIR）？**

Memory Attribute Indirection Register，定义8组内存属性（设备/普通，可缓存性等）。页表项通过 AttrIdx 字段索引 MAIR 中的属性定义，比 x86 直接在 PTE 编码更灵活。

**Q15: ARM 服务器相比 x86 服务器的优劣势？**

优势：能效比高、核心数多、成本低。劣势：软件生态不如 x86 成熟、部分应用需要移植/重编译、单核性能通常稍弱。
