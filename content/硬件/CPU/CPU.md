---
tags: [cpu, hardware]
type: index
aliases:
  - 中央处理器
  - Central Processing Unit
---

> CPU（Central Processing Unit）是计算机的**运算和控制核心**，负责取指令、译码、执行。现代 CPU 通过流水线、乱序执行、分支预测、缓存层次等微架构技术大幅提升指令吞吐。

## 关键指标

### IPC / CPI

| 指标 | 全称 | 定义 | 目标 |
|------|------|------|------|
| **IPC** | Instructions Per Cycle | 每周期完成的指令数 | 越高越好 |
| **CPI** | Cycles Per Instruction | 每条指令所需周期数 | 越低越好 |

关系：`CPI = 1 / IPC`。

CPU 执行时间公式：

```
程序执行时间 = 指令数 × CPI × 时钟周期
            = 指令数 × CPI / 时钟频率
```

### 时钟频率

时钟频率（Clock Frequency）决定 CPU 每秒的周期数，单位 GHz（吉赫兹）。受功耗墙（Power Wall）限制，单核频率在 ~5 GHz 附近停滞，性能增长转向多核和每周期指令数（IPC）提升。

| 时代 | 典型频率 | 驱动因素 |
|------|---------|----------|
| 1990s（Pentium） | 60-300 MHz | 工艺缩进驱动频率飙升 |
| 2000s（Pentium 4） | 1.5-3.8 GHz | 深流水线追求高频（失败） |
| 2010s（Core i） | 2.0-4.0 GHz | 多核 + Turbo Boost |
| 2020s（ARM / x86） | 2.0-5.5 GHz | 能效核 (E-core) + 性能核 (P-core) 异构 |

### 性能公式

```
               指令数 (Instruction Count)
CPU Time =    ────────────────────────────
               主频 × IPC

CPU Time = 指令数 × CPI × 周期时间
```

## 现代 CPU 概述

现代 CPU 综合运用多项微架构技术：

| 技术 | 描述 |
|------|------|
| **流水线 (Pipeline)** | 将指令执行拆分为多个阶段，提高吞吐 |
| **乱序执行 (OoO)** | 指令不按程序顺序执行，提高利用率 |
| **分支预测 (Branch Prediction)** | 预测条件跳转方向，避免流水线冲刷 |
| **超标量 (Superscalar)** | 每周期发射多条指令 |
| **超线程 (SMT)** | 单核模拟多个逻辑核，利用资源空闲 |
| **缓存层次 (Cache Hierarchy)** | 多级缓存隐藏内存延迟 |
| **SIMD** | 单指令多数据，向量化并行 |

## 文档导航

* [CPU 微架构](CPU微架构.md) — 流水线、乱序执行、分支预测、超标量、SMT
* [缓存体系](缓存体系.md) — 缓存层次、Cache Line、MESI 协议、伪共享、缓存优化
* [指令集](指令集.md) — CISC vs RISC、x86-64、ARM AArch64、RISC-V、SIMD

## 来源

* Patterson & Hennessy, *Computer Organization and Design (RISC-V Edition)*, Morgan Kaufmann, 2021
* Hennessy & Patterson, *Computer Architecture: A Quantitative Approach* (6th ed.), Morgan Kaufmann, 2017
* Intel, *Intel 64 and IA-32 Architectures Optimization Reference Manual*
* Fog, Agner, *The Microarchitecture of Intel, AMD and VIA CPUs*, 2022
