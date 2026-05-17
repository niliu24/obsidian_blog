---
tags: [bus, mmio, hardware]
type: entity
---

> MMIO（Memory-Mapped I/O）是一种 CPU 访问设备寄存器的机制：将设备寄存器映射到 CPU 的物理地址空间，CPU 使用普通的 load/store 指令即可读写设备。

## MMIO vs Port I/O

| 特性 | MMIO | Port I/O |
|------|------|----------|
| 访问方式 | 普通 load/store 指令 | 专用指令（x86: `in`/`out`） |
| 地址空间 | 统一在物理地址空间中 | 独立的 I/O 地址空间 |
| 缓存行为 | 不可缓存（uncacheable） | 不可缓存 |
| 寄存器宽度对齐 | 必须按宽度对齐访问（4 字节对齐的 32 位访问） | 对齐要求放宽 |
| 非 x86 平台 | 支持（ARM/RISC-V 没有 Port I/O） | 仅 x86 |
| 性能开销 | 较低（现代 CPU 深度优化） | 较高 |

## MMIO 访问方式

MMIO 区域必须在页边界对齐（4 KB），且标记为 **uncacheable (UC)**，防止 CPU 缓存导致读写不一致。

```c
#include <linux/io.h>

/* 第一步：从物理地址映射到内核虚拟地址空间 */
void __iomem *mmio_base = ioremap(phys_addr, size);
if (!mmio_base) {
    pr_err("ioremap failed\n");
    return -ENOMEM;
}

/* 第二步：通过 ioread/iowrite 系列函数访问 */
u32 val = ioread32(mmio_base + REG_OFFSET);   /* 读 32 位寄存器 */
iowrite32(0x1, mmio_base + REG_CONTROL);      /* 写 32 位寄存器 */

/* 也支持按字节/16位/64位访问 */
u8  b = ioread8(mmio_base + REG_BYTE);
u16 w = ioread16(mmio_base + REG_WORD);
u64 q = ioread64(mmio_base + REG_QUAD);

/* 第三步：使用结束后取消映射 */
iounmap(mmio_base);
```

## Write-Combine 优化

对于写密集型场景（如 GPU 命令提交），可以使用 WC（Write-Combine）映射，允许 CPU 将多次连续写入合并为一次 PCIe TLP，显著提升性能：

```c
/* 使用 ioremap_wc 替代 ioremap：允许写入合并 */
void __iomem *doorbell = ioremap_wc(phys_addr, size);

/* 写入 doorbell 触发设备操作 */
iowrite32(tail_index, doorbell + DOORBELL_OFFSET);
```

## MMIO 与内存屏障

由于 MMIO 写入可能被 CPU 重排（Out-of-Order），PCIe 写入也可能是 Posted 的（即不等待完成确认），需要使用内存屏障保证顺序：

```c
/* 设置 DMA 描述符地址 */
iowrite64(dma_addr, mmio + DMA_DESC_ADDR);
/* 屏障：确保 DMA 地址已写入设备 */
wmb();  /* write memory barrier */
/* 然后触发 DMA */
iowrite32(1, mmio + DMA_START);
```

| 屏障 | 作用 |
|------|------|
| `wmb()` | 确保屏障前的所有写入在屏障后的写入之前完成 |
| `mmiowb()` | MMIO 写屏障，与 spinlock 配合使用 |

> `readl()`/`writel()` 是 Linux 早期版本使用的访问函数，现已推荐使用 `ioread32()`/`iowrite32()`。

## 参考来源

- Intel, *Intel 64 and IA-32 Architectures Software Developer's Manual*, Volume 3A
- [Linux kernel: ioremap.c](https://elixir.bootlin.com/linux/latest/source/lib/ioremap.c)
- Ulf Hansson, "MMIO Write Combining," LWN.net
