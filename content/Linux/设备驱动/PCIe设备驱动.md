---
tags: [linux, pcie, device-driver]
type: entity
aliases:
  - Linux PCIe 设备驱动
  - pci_driver
---

> PCIe 设备驱动遵循 Linux 设备模型中的**总线-设备-驱动**架构。PCIe 总线在枚举阶段发现设备并分配资源（BAR 地址空间、IRQ 号），驱动通过 `pci_driver` 结构体注册到 PCIe 总线，匹配并接管设备。

## PCIe 设备枚举

系统启动时 BIOS/内核执行 PCIe 枚举，为设备配置 BAR 空间：

```
Root Complex → 扫描 Bus 0
    → 发现 Device 0 (PCIe Switch)
        → 扫描 Switch 下游 Bus
            → 发现 Endpoint (GPU/NVMe/NIC)
                → 读取配置空间 (Vendor ID, Device ID, Class Code)
                → 写入 BAR 探测大小，分配地址空间
                → 分配 IRQ 号 (MSI/MSI-X)
                → 创建设备节点
```

## 核心数据结构

```c
#include <linux/pci.h>

// PCI 设备描述符
struct pci_dev {
    unsigned int    devfn;          // 设备功能号 (device + function)
    unsigned short  vendor;         // Vendor ID
    unsigned short  device;         // Device ID
    unsigned short  subsystem_vendor;
    unsigned short  subsystem_device;
    unsigned int    class;          // 设备类别 (如 0x030000 = VGA)
    u8              revision;
    struct resource resource[6];    // 6 个 BAR 资源
    unsigned int    irq;            // 分配的 IRQ 号
    // ...
};

// PCI 驱动描述符
struct pci_driver {
    const char *name;
    const struct pci_device_id *id_table;  // 支持的设备 ID 表
    int  (*probe)(struct pci_dev *dev, const struct pci_device_id *id);
    void (*remove)(struct pci_dev *dev);
    // ...
};

// 设备 ID 匹配表
struct pci_device_id {
    __u32 vendor, device;
    __u32 subvendor, subdevice;
    __u32 class, class_mask;
    kernel_ulong_t driver_data;
};
```

## 驱动注册与匹配流程

```
1. module_init() → pci_register_driver(&my_pci_driver)
2. PCIe 总线枚举设备 → 遍历驱动列表
3. 匹配 pci_device_id (Vendor + Device + Class)
4. 调用 probe() 函数
5. probe() 中: 使能设备、映射 BAR、注册字符设备、注册中断
```

## 完整 PCIe 设备驱动示例

```c
#include <linux/module.h>
#include <linux/pci.h>
#include <linux/io.h>
#include <linux/interrupt.h>
#include <linux/fs.h>
#include <linux/cdev.h>

#define VENDOR_ID  0x1234  // 示例 Vendor ID
#define DEVICE_ID  0x5678  // 示例 Device ID

struct my_pci_dev {
    struct pci_dev *pdev;
    void __iomem *bar0;        // MMIO 映射
    dma_addr_t dma_handle;
    void *dma_buf;
    int irq;
    dev_t cdev_num;
    struct cdev cdev;
};

static struct my_pci_dev *my_dev;

// PCIe 设备支持的 ID 表 (匹配 Vendor + Device)
static const struct pci_device_id my_pci_ids[] = {
    { PCI_DEVICE(VENDOR_ID, DEVICE_ID) },
    { 0, }
};
MODULE_DEVICE_TABLE(pci, my_pci_ids);

// 中断处理
static irqreturn_t my_pci_irq(int irq, void *data)
{
    struct my_pci_dev *dev = data;
    u32 status = readl(dev->bar0 + 0x10);

    if (!(status & 0x1))
        return IRQ_NONE;

    writel(status, dev->bar0 + 0x10); // ACK 中断
    return IRQ_HANDLED;
}

// probe: 设备被枚举时调用
static int my_pci_probe(struct pci_dev *pdev, const struct pci_device_id *id)
{
    int ret;

    my_dev = kzalloc(sizeof(*my_dev), GFP_KERNEL);
    if (!my_dev)
        return -ENOMEM;
    my_dev->pdev = pdev;

    // 1. 使能 PCIe 设备
    ret = pci_enable_device(pdev);
    if (ret) goto err_free;

    // 2. 设置 DMA 掩码
    ret = dma_set_mask_and_coherent(&pdev->dev, DMA_BIT_MASK(64));
    if (ret) {
        ret = dma_set_mask_and_coherent(&pdev->dev, DMA_BIT_MASK(32));
        if (ret) goto err_disable;
    }

    // 3. 映射 BAR0 (MMIO 寄存器)
    ret = pci_request_region(pdev, 0, "my_pci_bar0");
    if (ret) goto err_disable;
    my_dev->bar0 = pci_iomap(pdev, 0, pci_resource_len(pdev, 0));
    if (!my_dev->bar0) {
        ret = -ENOMEM;
        goto err_release;
    }

    // 4. 分配 DMA 缓冲区
    my_dev->dma_buf = dma_alloc_coherent(&pdev->dev, 4096,
                                          &my_dev->dma_handle, GFP_KERNEL);
    if (!my_dev->dma_buf) {
        ret = -ENOMEM;
        goto err_unmap;
    }

    // 5. 注册中断 (MSI/MSI-X)
    ret = pci_alloc_irq_vectors(pdev, 1, 1, PCI_IRQ_MSI);
    if (ret < 0) goto err_dma;
    my_dev->irq = pci_irq_vector(pdev, 0);
    ret = request_irq(my_dev->irq, my_pci_irq, 0, "my_pci", my_dev);
    if (ret) goto err_irq_vec;

    pci_set_drvdata(pdev, my_dev);
    pr_info("my_pci: device probed (BAR0=%p, IRQ=%d)\n", my_dev->bar0, my_dev->irq);
    return 0;

err_irq_vec:
    pci_free_irq_vectors(pdev);
err_dma:
    dma_free_coherent(&pdev->dev, 4096, my_dev->dma_buf, my_dev->dma_handle);
err_unmap:
    pci_iounmap(pdev, my_dev->bar0);
err_release:
    pci_release_region(pdev, 0);
err_disable:
    pci_disable_device(pdev);
err_free:
    kfree(my_dev);
    return ret;
}

// remove: 设备被移除时调用
static void my_pci_remove(struct pci_dev *pdev)
{
    struct my_pci_dev *dev = pci_get_drvdata(pdev);

    free_irq(dev->irq, dev);
    pci_free_irq_vectors(pdev);
    dma_free_coherent(&pdev->dev, 4096, dev->dma_buf, dev->dma_handle);
    pci_iounmap(pdev, dev->bar0);
    pci_release_region(pdev, 0);
    pci_disable_device(pdev);
    kfree(dev);
    pr_info("my_pci: device removed\n");
}

static struct pci_driver my_pci_driver = {
    .name     = "my_pci",
    .id_table = my_pci_ids,
    .probe    = my_pci_probe,
    .remove   = my_pci_remove,
};

module_pci_driver(my_pci_driver); // 等价于 module_init/exit 封装

MODULE_LICENSE("GPL");
MODULE_DESCRIPTION("A simple PCIe device driver");
```

## sysfs 设备模型

sysfs 挂载在 `/sys/`，以目录树形式导出内核对象：

```
/sys/
├── bus/        → 总线 (pci, usb, platform...)
├── class/      → 设备类 (net, input, tty, misc...)
├── devices/    → 所有设备
└── module/     → 已加载的内核模块
```

驱动可以通过 `sysfs_create_group()` 创建自定义属性文件：

```c
static ssize_t myattr_show(struct kobject *kobj, struct kobj_attribute *attr, char *buf)
{
    return sprintf(buf, "%d\n", my_value);
}

static struct kobj_attribute myattr = __ATTR_RO(myattr);

static struct attribute *attrs[] = {
    &myattr.attr,
    NULL,
};

static struct attribute_group myattr_group = {
    .attrs = attrs,
};

sysfs_create_group(&pdev->dev.kobj, &myattr_group);
```

## 设备文件创建机制

### mknod (手动创建)

```bash
# 语法: mknod <name> <type> <major> <minor>
mknod /dev/mydevice c 240 0   # 字符设备
mknod /dev/myblock b 8 0      # 块设备
```

### udev (自动创建)

现代 Linux 使用 `udev`（基于 `devtmpfs` + `udevd`）自动管理设备文件：

```
内核检测到新设备 → 发送 uevent 到 udevd
    → udevd 匹配规则 (/etc/udev/rules.d/)
    → 在 /dev/ 下创建设备文件
    → 加载固件、设置权限、触发模块加载
```

驱动的 `class_create()` + `device_create()` 会触发内核发送 uevent，从而让 udev 自动创建设备文件。

## 相关文档

- [设备驱动](设备驱动.md) — 设备驱动总览
- [内核模块](内核模块.md) — .ko 文件结构与加载流程
- [DMA 与 MMIO](DMA与MMIO.md) — DMA 和 MMIO 驱动编程
- [字符设备驱动](字符设备驱动.md) — cdev 与 file_operations
- [中断处理](../../中断处理/中断处理.md) — MSI/MSI-X 中断注册与处理
- [PCIe 总线硬件](../../../硬件/总线/PCIe.md) — PCIe 拓扑、TLP 类型
- [BAR 硬件](../../../硬件/总线/BAR.md) — Base Address Register 原理
- [MSI/MSI-X](../../../硬件/中断/MSI.md) — PCIe 消息中断机制

## 相关资料

- Linux 内核源码: `drivers/pci/`、`include/linux/pci.h`
- Linux Device Drivers, 3rd Edition (O'Reilly) — 第 12 章
- `Documentation/PCI/`
- `/sys/bus/pci/devices/` — PCIe 设备树
- `lspci -vvv` — 查看 PCIe 设备详情
