---
tags: [bus, cxl, hardware]
type: entity
---

> CXL（Compute Express Link）建立在 PCIe Gen5/Gen6 PHY 之上，增加了缓存一致性协议，使 CPU 和加速器（GPU、FPGA、SmartNIC）可以共享内存并保持一致性。

## CXL 协议层次

| CXL 协议 | 用途 |
|----------|------|
| CXL.io | 基础 PCIe 功能（枚举、配置、MMIO、DMA），与标准 PCIe 兼容 |
| CXL.cache | 加速器可以缓存和修改主机内存，保持缓存一致性 |
| CXL.mem | 加速器可以拥有自己的内存并暴露给主机，CPU 通过 load/store 访问 |

## 与 PCIe 的关系

```
CXL.cache / CXL.mem
─────────────────────  ← CXL 在 PCIe PHY 之上新增的缓存一致性层
CXL.io = PCIe 5.0/6.0
─────────────────────  ← 复用 PCIe 物理层和链路层
PCIe PHY (Gen5/6)
```

CXL 设备在物理上是 PCIe 设备，使用相同的电气特性和链路训练流程。CXL.io 提供标准 PCIe 枚举和配置能力，确保向后兼容。

## 典型场景

* **内存扩展**: 通过 CXL 连接的内存模块，CPU 直接访问大容量内存池，延迟约 2-3x 本地 DRAM
* **GPU 一致性**: GPU 和 CPU 共享同一内存空间，减少数据拷贝，避免显式 `cudaMemcpy`
* **池化内存**: 多台主机共享同一物理内存池，按需动态分配

## 参考来源

* [CXL Specification](https://www.computeexpresslink.org/)
* [CXL Consortium: Compute Express Link Whitepaper](https://www.computeexpresslink.org/white-papers)
