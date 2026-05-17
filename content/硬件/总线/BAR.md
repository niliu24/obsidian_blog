---
tags: [bus, pcie, bar, hardware]
type: entity
---

> BAR（Base Address Register）是 PCIe 配置空间中设备用来向系统请求地址空间的寄存器。系统通过 BAR 为设备分配 MMIO 和 I/O 地址范围，CPU 和 DMA 引擎据此访问设备。

## BAR 在配置空间中的位置

PCIe 配置空间共 4 KB（标准头部 64 字节 + Capability 结构）。对于 Type 0（Endpoint）设备，BAR 位于配置空间偏移 0x10~0x24，共 **6 个 BAR**（每个 32 位宽，64 位 BAR 占两个槽位）：

```
偏移 0x10: BAR0 (32-bit)    ← 每个 BAR 32位，但64位BAR需两个连续槽位
偏移 0x14: BAR1 (32-bit)
偏移 0x18: BAR2 (32-bit)
偏移 0x1C: BAR3 (32-bit)
偏移 0x20: BAR4 (32-bit)
偏移 0x24: BAR5 (32-bit)
```

每个 BAR 的最低几位标识 BAR 属性：

| Bit | 含义 |
|-----|------|
| bit 0 | 0 = Memory Space, 1 = I/O Space |
| bit 1-2 (Memory) | 00 = 32-bit, 10 = 64-bit |
| bit 3 (Memory) | 0 = Non-prefetchable, 1 = Prefetchable |

## BAR 分配过程（PCIe Enumeration）

BIOS 或 OS 在启动时执行 PCIe 枚举，为每个设备分配 BAR 地址空间：

1. 软件向 BAR 写入全 1（`0xFFFFFFFF`），读取返回值
2. 返回值的低有效位指示地址空间大小（32 位对齐，大小必须是 2 的幂）
3. 软件在可用地址范围内分配一块未使用的空间，写入 BAR
4. 对 64 位 BAR，需两次写入（高 32 位 + 低 32 位）

## 示例：AMD GPU BAR 布局

| BAR | 大小 | 用途 | 属性 |
|-----|------|------|------|
| BAR0 + BAR1 | 16 GB | VRAM 映射（显存直接访问） | 64-bit, Prefetchable |
| BAR2 + BAR3 | 4 MB | Doorbell（门铃寄存器） | 64-bit, Non-prefetchable |
| BAR5 | 256 KB | MMIO 寄存器（控制/状态） | 32-bit, Non-prefetchable |

## 查看 BAR

```bash
# 查看设备 PCIe 配置空间中的 BAR
lspci -vvv -s 01:00.0

# 输出示例：
# Region 0: Memory at <addr> (64-bit, prefetchable) [size=16G]
# Region 2: Memory at <addr> (64-bit, non-prefetchable) [size=4M]
# Region 5: Memory at <addr> (32-bit, non-prefetchable) [size=256K]
```

```bash
# 查看当前系统分配的 PCIe 地址空间映射
cat /proc/iomem | grep PCI
```

## 参考来源

* [PCI Express Base Specification Revision 6.0](https://pcisig.com/specifications)
* Intel, *Intel 64 and IA-32 Architectures Software Developer's Manual*, Volume 3A
