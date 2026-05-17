---
tags: [memory, dram, ddr]
type: entity
---

> 主存（Main Memory）是计算机系统中 CPU 可以直接寻址的存储空间，通常由 **DRAM (Dynamic Random Access Memory)** 芯片组成。DRAM 因其高密度、低成本成为主存的绝对主流，但也因需要周期性刷新和较长的访问延迟而产生了从 DDR3 到 DDR5 乃至 HBM 的持续演进。

## DRAM 基本单元原理

DRAM 存储单元称为 **1T1C**（1 Transistor + 1 Capacitor）：

```
Wordline (WL)
    │
    ┌─┴─┐
    │   │  ← 存储电容 (Capacitor)
    │   │
    └─┬─┘
    │
Bitline (BL)
```

* **写操作**：Wordline 导通，Bitline 驱动电容充电（1）或放电（0）
* **读操作**：Wordline 导通，电容电荷释放到 Bitline，通过 Sense Amplifier 检测微小电压差
* **刷新需求**：电容会自然漏电，需要定期刷新（典型 64ms 内刷新所有行）

| 参数 | DRAM | SRAM |
|------|------|------|
| 存储单元 | 1T1C（1 晶体管 + 1 电容） | 6T（6 晶体管） |
| 密度 | 高（每单元面积小） | 低（每单元面积大 6-10x） |
| 速度 | 慢 (~50-100ns 随机访问) | 快 (~1-5ns) |
| 功耗 | 需刷新，有静态功耗 | 无需刷新，静态功耗低 |
| 易失性 | 断电丢失 | 断电丢失 |
| 制造成本 | 低 | 高 |
| 主要用途 | 主存 (DIMM) | CPU Cache (L1/L2/L3) |

> **注意**：虽然 SRAM 速度远快于 DRAM，但其 6T 结构导致密度低、成本高。SRAM 仅用于 CPU 内部高速缓存，主存必须使用 DRAM 才能在合理成本内达到所需容量。

## DDR 演进

### DDR3 → DDR4 → DDR5

| 特性 | DDR3 | DDR4 | DDR5 |
|------|------|------|------|
| 推出年份 | 2007 | 2014 | 2020 |
| 传输速率 (MT/s) | 800-2133 | 1600-3200 | 4800-8800 |
| 标准电压 | 1.5 V | 1.2 V | 1.1 V |
| Bank Group | 无 | 4 | 8 |
| 突发长度 (BL) | 8 | 8 | 16 |
| 预取宽度 | 8n | 8n | 16n |
| 单 Die 最大容量 | 8 Gb | 16 Gb | 64 Gb |
| DIMM 最大容量 | 32 GB | 128 GB | 512 GB+ |
| ECC 支持 | 可选 | 可选 | 片内 ECC (ODECC) |
| PMIC 集成 | 主板控制 | 主板控制 | DIMM 自带 PMIC |

### DDR5 关键改进

1. **双通道 DIMM 架构**：一个 DIMM 内部包含两个独立的 32-bit 子通道，提高访问并发度
2. **片内 ECC (ODECC)**：Die 内部集成错误纠正，提高可靠性
3. **DIMM 自带 PMIC (Power Management IC)**：改善电源完整性和信号质量
4. **更高 Bank 数**：Bank Group 数量从 4 增加到 8，减少 Bank 冲突
5. **训练/Equalization 增强**：支持更高频率的信号完整性

## DIMM 形态因子

| 类型 | 全称 | 特点 | 典型场景 |
|------|------|------|----------|
| **UDIMM** | Unbuffered DIMM | 无缓冲，CPU 直接访问 | 台式机、笔记本 |
| **RDIMM** | Registered DIMM | 地址/命令注册缓冲，减少电气负载 | 服务器 |
| **LRDIMM** | Load-Reduced DIMM | 数据也通过缓冲器，进一步降低负载 | 高端服务器 (大容量) |
| **NVDIMM** | Non-Volatile DIMM | 集成 NAND/SuperCap，断电持久化 | 数据中心（逐渐被 CXL 替代） |
| **SODIMM** | Small Outline DIMM | 小型化封装，尺寸约 UDIMM 一半 | 笔记本、紧凑型设备 |

## 内存通道 (Memory Channels)

内存通道是 CPU 与内存控制器之间的独立数据路径。增加通道数可线性提升理论带宽。

| 配置 | 通道数 | 总带宽 (DDR5-4800) |
|------|--------|-------------------|
| 单通道 | 1 | ~38.4 GB/s |
| 双通道 | 2 | ~76.8 GB/s |
| 四通道 | 4 | ~153.6 GB/s |
| 八通道 | 8 | ~307.2 GB/s |

> **实际性能**：多通道需配合多个 DIMM 插满对应通道才能发挥全部带宽。不对称配置（如只插 3 根 DIMM 的 4 通道系统）会降为单通道或双通道模式。

## Rank 与 Bank

### Rank

Rank 是 DIMM 上一组共享片选信号（Chip Select）的 DRAM 芯片集合。一个 Rank 共同构成 DIMM 的数据总线宽度（通常是 64-bit，带 ECC 时 72-bit）。

```
一个 RDIMM 的 Rank 组织:
┌─────────────────────────────────────┐
│  Rank 0: 芯片 0-7 (64-bit 数据总线)     │
│  ┌──┐ ┌──┐ ┌──┐          ┌──┐       │
│  │ 0│ │ 1│ │ 2│ ... ...  │ 7│       │
│  └──┘ └──┘ └──┘          └──┘       │
│  Rank 1: 芯片 8-15                    │
│  ┌──┐ ┌──┐ ┌──┐          ┌──┐       │
│  │ 8│ │ 9│ │10│ ... ...  │15│       │
│  └──┘ └──┘ └──┘          └──┘       │
└─────────────────────────────────────┘
```

* **单 Rank (1R)**：一个 DIMM 一个 Rank，容量最小
* **双 Rank (2R)**：两个 Rank，通过片选信号分时访问
* **四 Rank (4R)**：四个 Rank，更大容量

多 Rank 可提供 **Rank Interleaving** 并行访问，提升带宽。

### Bank

每个 DRAM 芯片内部划分为多个 Bank，Bank 内部再分为行 (Row) 和列 (Column)：

```
DRAM 芯片内部结构:
┌─────────────────────────────────────┐
│  Bank 0      Bank 1      Bank N      │
│  ┌──────┐    ┌──────┐    ┌──────┐    │
│  │Row 0 │    │Row 0 │    │Row 0 │    │
│  │Row 1 │    │Row 1 │    │Row 1 │    │
│  │ ...  │    │ ...  │    │ ...  │    │
│  │Row M │    │Row M │    │Row M │    │
│  └──────┘    └──────┘    └──────┘    │
└─────────────────────────────────────┘
```

* Bank 内部的行缓冲区 (Row Buffer) 可以缓存最近打开的行
* 不同 Bank 可以独立操作（预充电、激活、读写），实现 **Bank-Level Parallelism**
* Bank Group（DDR4/DDR5）进一步分层：同一 Bank Group 内的 Bank 共享一些资源，不同 Bank Group 间完全独立

## ECC 内存

ECC (Error-Correcting Code) 内存通过额外校验位实现硬件级错误检测与纠正。

| 特性 | 非 ECC 内存 | ECC 内存 |
|------|------------|---------|
| 数据总线宽度 | 64-bit | 72-bit（额外 8-bit ECC） |
| 纠错能力 | 无 | SECDED (Single Error Correct, Double Error Detect) |
| 适用场景 | 消费级桌面/笔记本 | 服务器、工作站、关键任务 |
| 成本 | 低 | 高约 10-20% |
| DIMM 类型 | UDIMM | RDIMM / LRDIMM |

### ECC 工作原理

```
64-bit 数据 + 8-bit 校验码
        │
        ▼
ECC 编码器 (使用 Hamming 码)
        │
        ▼
   存储到 DRAM
        │
        ▼
ECC 解码器 (读取时校验)
        │
        ├── 无错误 → 正常返回
        ├── 单 bit 错 → 自动纠正并返回正确数据
        └── 双 bit 错 → 报告无法纠正的错误 (Uncorrectable Error, UE)
```

> **DDR5 的片内 ECC (ODECC)**：DDR5 每个 Die 内部自带 ECC，用于保护 DRAM 内部的可靠性（特别是高频率下的信号完整性），但对系统透明——系统层面仍需额外 ECC 才能实现端到端保护。

## 内存带宽计算

理论内存带宽公式：

```
带宽 (GB/s) = 内存时钟频率 (MHz) × 数据传输率 (每时钟传输次数) × 总线宽度 (Bytes) × 通道数
```

对于 DDR 内存，数据传输率 = 2（Double Data Rate），因此可以简化为：

```
带宽 (GB/s) = DDR 速率 (MT/s) × 总线宽度 (Bytes) × 通道数
```

常见配置：

| 配置 | 计算 | 理论带宽 |
|------|------|---------|
| DDR4-3200 双通道 | 3200 × 8 × 2 = 51200 MB/s | **51.2 GB/s** |
| DDR5-4800 双通道 | 4800 × 8 × 2 = 76800 MB/s | **76.8 GB/s** |
| DDR5-5600 双通道 | 5600 × 8 × 2 = 89600 MB/s | **89.6 GB/s** |
| DDR5-4800 八通道 | 4800 × 8 × 8 = 307200 MB/s | **307.2 GB/s** |
| HBM2e (4-Hi) | ~2 GT/s × 1024-bit × 8 = … | **~1.6 TB/s** |

> **注意**：实际带宽因协议开销、Bank 冲突、行切换等因素通常只能达到理论带宽的 60-80%。

## HBM (High Bandwidth Memory)

HBM 是面向 GPU/高性能加速器/ASIC 的高带宽内存方案，通过 **2.5D 硅中介层 (Interposer) 堆叠**实现远超 DDR 的带宽。

### HBM 结构

```
GPU Die (Logic)
  │ TSV (Through Silicon Via)
  ┌──────┐
  │ DRAM │ ← HBM Stack (4/8/12 Hi)
  │ DRAM │
  │ DRAM │
  │ DRAM │
  └──┬───┘
     │ μbumps + Interposer
  ┌──┴───┐
  │ 封装基板  │
  └──────┘
```

### HBM 演进

| 代次 | 带宽/栈 | 容量/栈 | 堆叠层数 | 典型应用 |
|------|---------|---------|---------|---------|
| HBM | 128 GB/s | 1 GB | 4-Hi | AMD Fiji / NVIDIA P100 |
| HBM2 | 256-307 GB/s | 2-8 GB | 4/8-Hi | NVIDIA A100 / AMD MI250 |
| HBM2e | 410 GB/s | 16 GB | 8-Hi | NVIDIA H100 |
| HBM3 | 819 GB/s | 16-64 GB | 8/12-Hi | NVIDIA H200 / B200 |
| HBM4 | ~1.6 TB/s+ | 64 GB+ | 16-Hi | 规划中 (2026+) |

### HBM vs DDR

| 对比维度 | HBM (HBM2e) | DDR5 |
|---------|-------------|------|
| 内存接口宽度 | **1024-bit**/栈 | 64-bit/通道 |
| 典型总带宽 | ~2-4 TB/s (多栈) | ~50-300 GB/s |
| 功耗效率 | ~5 pJ/bit | ~20 pJ/bit |
| 容量 | 16-64 GB/栈 | 最高 ~2 TB/系统 |
| 互连距离 | 芯片内 (Interposer) | PCB 走线 |
| 成本 | 高 | 低 |

## CXL 内存扩展

CXL (Compute Express Link) 基于 PCIe 5.0/6.0 物理层，提供缓存一致的设备互连，使内存可以**池化扩展**：

* **CXL Type 3 设备**：纯内存扩展器 (Memory Expander)
* 服务器可通过 CXL 连接数 TB 的扩展内存，延缓 DRAM 昂贵成本
* 但 CXL 访问延迟比本地 DRAM 高（约 2-3x），适用于**冷数据/大容量场景**

## 查看内存信息

### dmidecode

```bash
# 查看所有 DIMM 信息
sudo dmidecode -t memory

# 输出示例（精简）
# Handle 0x003C, DMI type 17, 40 bytes
# Memory Device
# 	Array Handle: 0x003B
# 	Error Information Handle: Not Provided
# 	Total Width: 72 bits  （含 ECC）
# 	Data Width: 64 bits
# 	Size: 32 GB
# 	Form Factor: DIMM
# 	Set: None
# 	Locator: CPU0_DIMM_A1
# 	Bank Locator: NODE 0
# 	Type: DDR5
# 	Type Detail: Synchronous Registered (Buffered)
# 	Speed: 5600 MT/s
# 	Manufacturer: Samsung
# 	Serial Number: 12345678
# 	Asset Tag: 01020304
# 	Part Number: M321R4GA3BB6-CQK
# 	Rank: 2
```

### lshw

```bash
# 查看内存概览
sudo lshw -class memory

# 输出示例
# *-memory
#    description: System Memory
#    physical id: 1
#    slot: System board or motherboard
#    size: 256GiB
#  *-bank:0
#       description: DIMM DDR5 Synchronous Registered (Buffered)
#       product: M321R4GA3BB6-CQK
#       vendor: Samsung
#       physical id: 0
#       serial: 12345678
#       slot: CPU0_DIMM_A1
#       size: 32GiB
#       width: 72 bits
#       clock: 5600MHz
#  *-bank:1
#       ...
```

## 来源

* [JEDEC DDR5 Standard (JESD79-5)](https://www.jedec.org/standards-documents/docs/jesd79-5)
* [JEDEC HBM3 Standard (JESD238A)](https://www.jedec.org/standards-documents/docs/jesd238a)
* [DDR5 vs DDR4: All the Design Challenges & Advantages](https://www.synopsys.com/designware-ip/technical-bulletin/ddr5-design-challenges.html)
* [What Every Programmer Should Know About Memory](https://people.freebsd.org/~lstewart/articles/cpumemory.pdf) — Ulrich Drepper
* [Intel Memory Bandwidth Documentation](https://www.intel.com/content/www/us/en/developer/articles/technical/memory-bandwidth.html)
* [HBM: Memory Solution for High-Performance Processors](https://ieeexplore.ieee.org/document/7073669) — JEDEC HBM Whitepaper
* [CXL 3.0 Specification](https://www.computeexpresslink.org/download-the-specification)
