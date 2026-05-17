---
tags: [interrupt, hardware, pcie, msi]
type: entity
---

> MSI（Message Signaled Interrupts）是 PCIe 提出的**消息中断**机制，设备通过发送 PCIe Memory Write TLP 来触发中断，无需物理中断线。MSI-X 是 MSI 的增强版本，为多队列设备提供更大的灵活性和扩展性。

## MSI 工作原理

```
PCIe 设备
  ↓  发送 Memory Write TLP
  ↓  目标地址: 0xFEE00000 + (Dest_ID << 12)
  ↓  写入数据: [Vector | Delivery Mode | Trigger Mode]
  ↓
CPU → Local APIC 匹配地址 → 从数据载荷取出 Vector → 注入 CPU 中断流水线
```

设备将中断视为写入特定内存地址的数据，Local APIC 监听该地址范围并直接识别中断。这消除了 IRQ 线和 I/O APIC 中断引脚的限制。

## 消息格式

**消息地址**（通常为 `0xFEExxxxx`）:

```
0xFEE0_0F00h  (示例：Dest_ID=0x0F, RH=0, DM=Physical)
```

**消息数据**:

| 位域 | 说明 |
|------|------|
| bits 7-0 | Vector（中断向量号，0-255） |
| bits 10-8 | Delivery Mode（000=Fixed, 001=Lowest Priority, …） |
| bit 11 | Redirection Hint（RH） |
| bit 12 | Destination Mode（0=Physical, 1=Logical） |
| bits 31-16 | 保留 / 其他属性 |

## MSI 与 MSI-X 对比

| 特性 | MSI | MSI-X |
|------|-----|-------|
| 最大向量数 | 32（1, 2, 4, 8, 16, 32，必须是 2 的幂） | 2048（任意数量，无需 2 的幂） |
| 每向量独立屏蔽 | 否（所有向量共享一个 Mask bit） | 是（每个向量有独立 Mask 控制） |
| 每向量独立地址 | 否（所有向量共享一个 Message Address） | 是（每向量有独立的 Message Address + Data） |
| 向量表位置 | PCIe Capability 结构中 | 设备 BAR 空间中（Function 可映射） |
| 典型场景 | 传统 PCIe 设备、少量中断 | 多队列设备（NVMe、高速 NIC、GPU） |

**为什么 MSI-X 对多队列重要**：

* NVMe SSD 可以为每个队列分配一个独立的 MSI-X 向量
* 网卡可以为每个 RX/TX 队列分配独立的 MSI-X 向量
* OS 可以将不同向量绑定到不同 CPU 核心，实现负载均衡（Receive Side Scaling, RSS）
* 每向量独立屏蔽：在队列重配置时仅屏蔽对应向量，不影响其他队列

## Linux 驱动示例

```c
#include <linux/interrupt.h>
#include <linux/pci.h>

/* 中断处理函数（上半部 - Top Half） */
static irqreturn_t my_device_isr(int irq, void *dev_id)
{
    struct my_dev *dev = (struct my_dev *)dev_id;

    u32 status = ioread32(dev->mmio_base + REG_STATUS);
    if (!(status & STATUS_IRQ_PENDING))
        return IRQ_NONE;

    if (status & STATUS_DMA_COMPLETE) {
        dev->dma_done = true;
        return IRQ_WAKE_THREAD;
    }

    iowrite32(status, dev->mmio_base + REG_STATUS);
    return IRQ_HANDLED;
}

/* 中断处理下半部（Threaded IRQ） */
static irqreturn_t my_device_thread(int irq, void *dev_id)
{
    struct my_dev *dev = (struct my_dev *)dev_id;

    dma_unmap_single(dev->pdev, dev->dma_handle, dev->buf_size,
                     DMA_FROM_DEVICE);

    return IRQ_HANDLED;
}

/* 在驱动初始化中注册 */
int my_probe(struct pci_dev *pdev, const struct pci_device_id *id)
{
    struct my_dev *dev;
    int irq = pci_irq_vector(pdev, 0);
    int ret;

    ret = pci_alloc_irq_vectors(pdev, 1, 1, PCI_IRQ_MSIX | PCI_IRQ_MSI);
    if (ret < 0) {
        pr_err("Failed to allocate MSI/MSI-X vectors\n");
        return ret;
    }

    ret = request_threaded_irq(irq, my_device_isr, my_device_thread,
                               IRQF_SHARED | IRQF_ONESHOT,
                               "my_device", dev);
    if (ret) {
        pr_err("Failed to register IRQ\n");
        pci_free_irq_vectors(pdev);
        return ret;
    }

    return 0;
}

void my_remove(struct pci_dev *pdev)
{
    struct my_dev *dev = pci_get_drvdata(pdev);
    free_irq(pci_irq_vector(pdev, 0), dev);
    pci_free_irq_vectors(pdev);
}
```

### MSI-X 中断亲和性

```bash
# 查看 IRQ 亲和性
cat /proc/irq/28/smp_affinity

# 将不同 MSI-X 向量绑定到不同 CPU
echo 1 > /proc/irq/25/smp_affinity   # nvmeq1 → CPU0
echo 2 > /proc/irq/26/smp_affinity   # nvmeq2 → CPU1
echo 4 > /proc/irq/27/smp_affinity   # nvmeq3 → CPU2
echo 8 > /proc/irq/28/smp_affinity   # nvmeq4 → CPU3
```

## 参考来源

* [PCI Express Base Specification Revision 6.0](https://pcisig.com/specifications)
* Intel, *Intel 64 and IA-32 Architectures Software Developer's Manual*, Volume 3A
* [Linux kernel MSI documentation](https://www.kernel.org/doc/html/latest/PCI/msi-howto.html)
