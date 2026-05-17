---
tags: [bus, dma, hardware]
type: entity
---

> DMA（Direct Memory Access）允许外设直接读写主机内存，无需 CPU 逐字节搬运。这是高性能 I/O 的基础。

## DMA vs CPU Polling

```
CPU Polling:
  CPU: [读取状态寄存器] → 等待 → [读取状态寄存器] → 等待 → ... → [数据就绪, 开始搬运]
  问题: CPU 被占用，无法处理其他任务

DMA:
  CPU: [设置 DMA 描述符] → [通知设备开始] → (继续其他任务)
  设备: [自主传输数据到内存]
  设备完成: [触发 MSI 中断]
  CPU: [在中断处理中消费数据]
```

## Linux DMA API

Linux 提供了两套 DMA 映射 API：

### 1. 一致性 DMA（Coherent DMA）

适用于需要 CPU 和设备共享访问的缓冲区，驱动不需要手动管理缓存一致性：

```c
#include <linux/dma-mapping.h>
#include <linux/pci.h>

dma_addr_t dma_handle;
void *cpu_addr;
size_t size = 4096;

/* 分配一致性 DMA 缓冲区（保证 cache coherency） */
cpu_addr = dma_alloc_coherent(dev, size, &dma_handle, GFP_KERNEL);
if (!cpu_addr) {
    pr_err("dma_alloc_coherent failed\n");
    return -ENOMEM;
}

/* CPU 通过 cpu_addr 访问，设备通过 dma_handle 访问 */
memset(cpu_addr, 0, size);    /* CPU 写入 */
/* 设备可以直接 DMA 到相同物理内存，无需手动 cache 刷新 */

/* 释放 */
dma_free_coherent(dev, size, cpu_addr, dma_handle);
```

### 2. 流式 DMA（Streaming DMA）

适用于临时映射的缓冲区，性能更高，但需要手动管理同步：

```c
/* 分配普通内核缓冲区 */
void *buffer = kmalloc(size, GFP_KERNEL);

/* 映射为 DMA 地址 */
dma_addr_t dma_handle = dma_map_single(dev, buffer, size, DMA_BIDIRECTIONAL);
if (dma_mapping_error(dev, dma_handle)) {
    pr_err("dma_map_single failed\n");
    kfree(buffer);
    return -ENOMEM;
}

/* 设备使用：通知设备使用 dma_handle 进行 DMA 传输 */
/* ... 启动 DMA ... */

/* 设备完成传输后，同步缓存（确保 CPU 看到最新数据） */
dma_sync_single_for_cpu(dev, dma_handle, size, DMA_FROM_DEVICE);

/* 处理完数据后，可选映射回设备 */
dma_sync_single_for_device(dev, dma_handle, size, DMA_FROM_DEVICE);

/* 取消映射 */
dma_unmap_single(dev, dma_handle, size, DMA_BIDIRECTIONAL);
kfree(buffer);
```

### 3. Scatter-Gather DMA

SG-DMA 支持对非连续物理内存做一次 DMA 传输，避免大块连续内存分配失败：

```c
struct scatterlist sglist[NR_SG];
int nents;

/* 初始化 scatterlist（来自页数组等） */
sg_init_table(sglist, NR_SG);
for (i = 0; i < NR_SG; i++) {
    sg_set_page(&sglist[i], pages[i], PAGE_SIZE, 0);
}

/* 映射 SG 列表 */
nents = dma_map_sg(dev, sglist, NR_SG, DMA_TO_DEVICE);

/* 遍历映射后的 SG 条目，填充 DMA 描述符 */
for_each_sg(sglist, sg, nents, i) {
    dma_addr_t addr = sg_dma_address(sg);
    unsigned int len = sg_dma_len(sg);
    /* 将 addr/len 填入 DMA 描述符链表 */
}

/* 完成时取消映射 */
dma_unmap_sg(dev, sglist, nents, DMA_TO_DEVICE);
```

## 完整 DMA 传输流程

```
1. dma_alloc_coherent() / dma_map_single()
   → 分配/映射主机内存缓冲区

2. （可选）CPU 填充数据到缓冲区
   → 如网络发送：拷贝 skb->data 到 DMA 缓冲区
   → 如 GPU：写入命令描述符

3. iowrite64(dma_handle, mmio + DESC_ADDR)
   → 通过 MMIO 告诉设备 DMA 缓冲区的物理地址

4. iowrite32(1, mmio + DOORBELL)
   → 写入 Doorbell 寄存器，通知设备开始 DMA

5. 设备执行 DMA（设备自主读写主机内存）

6. 设备完成 → 发送 MSI/MSI-X 中断

7. CPU 在中断处理中：
   → dma_sync_single_for_cpu()（流式 DMA）
   → 消费数据
   → dma_unmap_single() / dma_free_coherent()
```

## 参考来源

* [Linux kernel DMA API documentation](https://www.kernel.org/doc/html/latest/core-api/dma-api.html)
* [Linux kernel DMA-API-HOWTO](https://www.kernel.org/doc/html/latest/core-api/dma-api-howto.html)
