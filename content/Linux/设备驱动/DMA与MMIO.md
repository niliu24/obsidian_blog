---
tags: [linux, dma, mmio, device-driver]
type: entity
aliases:
  - Linux DMA 与 MMIO
---

> DMA（Direct Memory Access）和 MMIO（Memory-Mapped I/O）是 Linux 设备驱动与硬件交互的两种核心方式。MMIO 让 CPU 访问设备寄存器，DMA 让设备自主读写主机内存。

## MMIO (Memory-Mapped IO)

- 设备的控制/状态寄存器映射到 CPU 的物理地址空间
- 使用 `ioremap()` 或 `ioremap_wc()` 建立内核虚拟地址映射
- 通过 `readl()/writel()` 等访问寄存器（保证顺序性）

```c
// PCIe BAR 映射示例
void __iomem *bar0 = pci_iomap(pdev, 0, pci_resource_len(pdev, 0));
writel(value, bar0 + REG_OFFSET);  // 写寄存器
u32 val = readl(bar0 + REG_STATUS); // 读状态
```

## DMA (Direct Memory Access)

- 设备直接读写主机内存，无需 CPU 逐字搬运
- 驱动使用 DMA API 分配一致性/流式 DMA 缓冲区

```c
// DMA 分配示例
dma_addr_t dma_handle;
void *cpu_addr = dma_alloc_coherent(dev, size, &dma_handle, GFP_KERNEL);

// 告诉设备 DMA 地址，设备直接读写该内存
writel(dma_handle, bar0 + DMA_ADDR_REG);
writel(size, bar0 + DMA_SIZE_REG);
writel(1, bar0 + DMA_START_REG); // 启动 DMA
```

## DMA API 对比

| DMA API | 用途 |
|---------|------|
| `dma_alloc_coherent()` | 分配一致性 DMA 缓冲区（cache 一致） |
| `dma_map_single()` | 映射流式 DMA 缓冲区 |
| `dma_unmap_single()` | 解除流式 DMA 映射 |
| `dma_alloc_noncoherent()` | 分配非一致性缓冲区（需手动 flush） |

## 相关文档

- [设备驱动](设备驱动.md) — 设备驱动总览
- [PCIe 设备驱动](PCIe设备驱动.md) — PCIe BAR 与 DMA 在 PCIe 设备中的应用
- [DMA 硬件原理](../../../硬件/总线/DMA.md) — DMA 硬件层面的完整分析
- [MMIO 硬件原理](../../../硬件/总线/MMIO.md) — MMIO 硬件层面的完整分析
- [中断处理](../../中断处理/中断处理.md) — DMA 完成后的中断处理

## 相关资料

- Linux 内核源码: `include/linux/dma-mapping.h`、`kernel/dma/`
- `Documentation/core-api/dma-api.rst`
- `Documentation/core-api/dma-api-howto.rst`
