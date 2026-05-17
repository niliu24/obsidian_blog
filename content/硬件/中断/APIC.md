---
tags: [interrupt, hardware, x86, apic]
type: entity
---

> APIC（Advanced Programmable Interrupt Controller）是为 SMP 系统设计的中断架构，由 Local APIC 和 I/O APIC 两个组件构成。x2APIC 是 APIC 的扩展版本，支持更大规模的处理器系统。

## 架构总览

```
                   +-----------+
                   |  I/O APIC |  ← 位于芯片组（PCH / Chipset）
                   +-----+-----+
                         | 中断总线 / 系统总线（DMI / PCIe）
       +-----------------+------------------+
       |                 |                  |
+------+------+   +------+------+   +------+------+
| Local APIC 0 |   | Local APIC 1 |   | Local APIC N |  ← 每个 CPU 核心一个
+-------------+   +-------------+   +-------------+
```

## Local APIC（本地 APIC）

每个 CPU 核心内部包含一个 Local APIC，寄存器通过 MMIO 访问（默认物理地址 `0xFEE00000`），每个寄存器 32 位，间隔 16 字节。

### 核心寄存器

| 寄存器 | 全称 | 功能 |
|--------|------|------|
| IRR | Interrupt Request Register | 256 位 bitmap，记录已到达但尚未分发的中断 |
| ISR | In-Service Register | 256 位 bitmap，记录正在被 CPU 处理的中断 |
| TMR | Trigger Mode Register | 256 位 bitmap，记录中断是边沿还是电平触发 |
| ICR | Interrupt Command Register | 发送 IPI 的寄存器 |
| LVT | Local Vector Table | 配置本地中断源（Timer、LINT0/1、Thermal 等） |
| TPR | Task Priority Register | 设置当前 CPU 的中断优先级阈值 |

### 优先级仲裁

当多个中断同时到达时，Local APIC 按以下规则选择最高优先级的中断派发给 CPU：

1. 向量号越大优先级越高（向量 0-255，每 16 个一组）
2. 比较 ISR 中最高优先级向量与 IRR 中新的请求向量
3. 只有当新请求优先级 > 当前服务中中断的优先级时，才会抢占

### IPI（Inter-Processor Interrupt）

一个 CPU 通过写入 ICR 向其他 CPU 发送中断，典型用途：

| IPI 类型 | Vector | 用途 |
|----------|--------|------|
| Fixed | 0-255 | 通用 IPI，调度器唤醒远程 CPU |
| NMI | 2 | 不可屏蔽中断，调试用 |
| SMI | — | 系统管理模式，固件使用 |
| INIT | — | 重置目标 CPU |
| Start-Up | — | AP 启动流程 |

```c
// 发送 IPI 示例（向目标 CPU 发送向量 0xEF 的 Fixed IPI）
void send_ipi(int dest_apic_id, u8 vector) {
    // 写 ICR: [Destination << 56 | Vector << 0]
    u32 icr_low = (dest_apic_id << 24) | vector;
    writel(icr_low, APIC_BASE + ICR_OFFSET_LOW);
}
```

### Local APIC Timer

每个核心独立的定时器，基于 CPU 总线时钟，三种模式：

| 模式 | 行为 |
|------|------|
| One-Shot | 到期后产生一次中断，需重新编程 |
| Periodic | 每隔周期时间产生中断 |
| TSC-Deadline | 写入绝对 TSC 值，到达时中断（最精确，需 INVARIANT_TSC） |

## I/O APIC

I/O APIC 位于芯片组中，负责将外部硬件中断路由到各个 Local APIC：

* 提供 **24 个可编程输入引脚**（旧版 16 个，现代 I/O APIC 通常 24 个）
* 每个引脚对应一个 **Redirection Table Entry（RTE）**

### 重定向表项（RTE）格式

| 字段 | 位域 | 说明 |
|------|------|------|
| Vector | bits 0-7 | 中断向量号（0-255） |
| Delivery Mode | bits 8-10 | Fixed / Lowest Priority / SMI / NMI / INIT / ExtINT |
| Destination Mode | bit 11 | Physical（APIC ID 直接匹配）vs Logical（位掩码或集群） |
| Delivery Status | bit 12 | 0 = Idle, 1 = Send Pending |
| Pin Polarity | bit 13 | Active High / Active Low |
| Remote IRR | bit 14 | Level-triggered 中断的远端 IRR 状态 |
| Trigger Mode | bit 15 | Edge / Level |
| Mask | bit 16 | 1 = 屏蔽该中断 |
| Destination | bits 56-63 | Physical Mode: APIC ID; Logical Mode: 逻辑目标 |

I/O APIC 的所有 RTE 初始时被屏蔽（Mask=1），由 OS 逐个配置并启用。

### 中断投递模式

| Delivery Mode | 值 | 说明 |
|---------------|-----|------|
| Fixed | 000 | 投递到 Destination 指定的处理器 |
| Lowest Priority | 001 | 投递到 Destination 集合中 TPR 最低的处理器 |
| SMI | 010 | 触发系统管理中断 |
| NMI | 100 | 不可屏蔽中断 |
| INIT | 101 | 初始化目标处理器 |
| ExtINT | 111 | 外部中断（兼容 8259 模式） |

## x2APIC

x2APIC 是 APIC 架构的扩展，解决大规模多核系统中的 APIC ID 不足问题：

| 特性 | xAPIC | x2APIC |
|------|-------|--------|
| APIC ID 宽度 | 8 位（最多 256 个逻辑处理器） | 32 位（最多 4,294,967,296 个） |
| 寄存器访问方式 | MMIO（`0xFEE00000`） | MSR（`IA32_X2APIC_*`，使用 `rdmsr`/`wrmsr`） |
| IPI 发送 | 写入 ICR（MMIO） | 写入 MSR `IA32_X2APIC_ICR` |
| 性能 | MMIO 访问较慢 | MSR 访问更快 |
| 支持平台 | 所有 SMP x86 | Intel Nehalem+，AMD Family 15h+ |

## 参考来源

* Intel, *Intel 64 and IA-32 Architectures Software Developer's Manual*, Volume 3A: System Programming Guide
* AMD, *AMD64 Architecture Programmer's Manual*, Volume 2: System Programming
