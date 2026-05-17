---
tags: [bus, pcie, hardware]
type: entity
---

> PCIe（Peripheral Component Interconnect Express）是当今最主流的系统总线标准，采用点对点串行连接，替代了早期的 PCI 并行总线。

## PCIe 拓扑

```
Root Complex (CPU内部)
    |
PCIe Switch (PCIe Bridge)
   / \
EP1 EP2
```

* **Root Complex (RC)**: CPU 内部的 PCIe 根节点，连接 CPU 和内存子系统，管理所有 PCIe 事务
* **Switch**: 扩展 PCIe 端口的交换设备，内部由多个虚拟 PCI-to-PCI 桥组成
* **Endpoint (EP)**: 终端设备，如 GPU、NVMe SSD、网卡（NIC）
* 拓扑结构为树形，不允许环路

## TLP 类型

PCIe 使用 **TLP（Transaction Layer Packet）** 进行通信，分为四种类型：

| TLP 类型 | 用途 |
|----------|------|
| Memory TLP | 内存读写（最常用，MMIO 和 DMA 均通过 Memory TLP 传输） |
| I/O TLP | I/O 端口读写（x86 兼容，非 x86 平台通常不支持） |
| Configuration TLP | 配置空间访问（枚举设备、分配资源） |
| Message TLP | 中断（MSI/MSI-X）、错误报告、电源管理等 |

## PCIe Lanes

PCIe 物理层由 **lane**（通道）组成，每个 lane 是一对差分信号线（一对发送、一对接收）：

| 通道数 | 典型用途 |
|--------|---------|
| x1 | 网卡、Wi-Fi 模块、SATA 控制器 |
| x4 | NVMe SSD（高端消费级） |
| x8 | 部分 GPU、RAID 控制器 |
| x16 | 旗舰 GPU、AI 加速卡（如 NVIDIA A100/H100） |

Lane 可以绑定工作：x16 相当于 16 个 lane 并行传输，带宽为单 lane 的 16 倍。设备可以协商降级（如 x16 降为 x8），以提高兼容性。

## 代际与带宽

| 代际 | 单 Lane 速率 | 编码方案 | 有效带宽 (x1) | 有效带宽 (x16) | 引入年份 |
|------|-------------|---------|--------------|---------------|---------|
| Gen1 | 2.5 GT/s | 8b/10b | 250 MB/s | 4 GB/s | 2003 |
| Gen2 | 5 GT/s | 8b/10b | 500 MB/s | 8 GB/s | 2007 |
| Gen3 | 8 GT/s | 128b/130b | ~985 MB/s | ~15.75 GB/s | 2010 |
| Gen4 | 16 GT/s | 128b/130b | ~1.97 GB/s | ~31.5 GB/s | 2017 |
| Gen5 | 32 GT/s | 128b/130b | ~3.94 GB/s | ~63 GB/s | 2019 |
| Gen6 | 64 GT/s | 1b/1b (PAM4) | ~8 GB/s | ~128 GB/s | 2022 |

**带宽计算示例**（PCIe Gen4 x16）：

* 原始速率：16 GT/s × 16 lane = 256 GT/s
* 有效带宽（128b/130b 编码，效率 128/130 ≈ 98.46%）：256 × 128/130 = 252.3 Gb/s ≈ 31.5 GB/s
* 双向总带宽：63 GB/s（同时收发）

编码方案对比：Gen1/2 使用 8b/10b（效率 80%），Gen3+ 使用 128b/130b（效率 98.46%），Gen6 使用 PAM4 信令 + 1b/1b 编码（每符号 2 比特，效率大幅提升）。

## 与设备交互

PCIe 设备通过 [BAR](BAR.md) 向系统请求地址空间，通过 [MMIO](MMIO.md) 让 CPU 访问寄存器，通过 [DMA](DMA.md) 自主读写主机内存，通过 [MSI/MSI-X](../中断/MSI.md) 发送中断。

## 参考来源

* [PCI Express Base Specification Revision 6.0](https://pcisig.com/specifications)
* [Linux kernel PCI documentation](https://www.kernel.org/doc/html/latest/PCI/)
* Intel, *Intel 64 and IA-32 Architectures Software Developer's Manual*
