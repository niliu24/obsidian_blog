---
tags: [storage, nvme, ssd]
type: entity
---

> NVMe (Non-Volatile Memory Express) 是基于 PCIe 总线的固态存储协议，专为 NAND Flash 和下一代非易失性存储器设计。与传统的 AHCI 协议相比，NVMe 通过多队列、低延迟命令路径和高效中断机制，充分发挥现代 SSD 的并行性能。

## NVMe 为何取代 AHCI

### AHCI 瓶颈

AHCI (Advanced Host Controller Interface) 是为 HDD 设计的协议，即使在 SSD 时代也强加了三重限制：

1. **单队列 (Single Queue)**：一个命令队列（最大 32 命令），所有 CPU 核心竞争同一队列
2. **HDD 优化假设**：命令排序 (NCQ) 针对机械寻道优化，对 SSD 无意义
3. **中断开销高**：每个命令都触发 MSI/MSI-X 中断

```
AHCI 架构:
                          ┌────┐
  CPU 0 ──┐              │    │
  CPU 1 ──┼── 锁争用 ──→  │ 单队列 │ ←── 32 深度
  CPU N ──┘              │    │
                          └──┬─┘
                             │
                          ┌──▼─┐
                          │ SSD │
                          └────┘
```

### NVMe 多队列

NVMe 支持最多 **64K 个队列对 (Queue Pairs)**，每个队列对包含一个提交队列 (Submission Queue, SQ) 和一个完成队列 (Completion Queue, CQ)：

```
NVMe 架构:
  CPU 0 ─────→ SQ[0] ──┐
                        │
  CPU 1 ─────→ SQ[1] ──┤
                        ├──→ PCIe ──→ SSD 控制器
  CPU 2 ─────→ SQ[2] ──┤
                        │
  CPU N ─────→ SQ[N] ──┘
```

### AHCI vs NVMe 对比

| 特性 | AHCI | NVMe |
|------|------|------|
| 队列深度 | 1 队列 × 32 命令 | 64K 队列 × 64K 命令 |
| 命令路径 | 4 次 MMIO 读/命令 | 1 次 MMIO 写/命令 (Doorbell) |
| 中断 | MSI-X (但每个命令都中断) | MSI-X (可配置中断合并) |
| 并行度 | 单队列锁竞争 | 无锁多队列，CPU 核级亲和 |
| 延迟 | ~6 μs (软件栈) | ~2-3 μs (软件栈) |
| NCQ | 面向 HDD 的命令排序 | 不必要（SSD 原生随机访问） |
| SATA Express | 6 Gbps ~550 MB/s | PCIe 3.0 x4 ~3.5 GB/s, 4.0 x4 ~7 GB/s |

## Queue Pair 机制

### 数据结构

```
Host 内存中:
┌──────────────────────┐
│ Submission Queue (SQ)│  ← 存放命令 (Command Entry)
│  CMD0 | CMD1 | ...   │     Host 写入，SSD 消费
├──────────────────────┤
│ Completion Queue (CQ)│  ← 存放完成状态 (Completion Entry)
│  CPL0 | CPL1 | ...   │     SSD 写入，Host 消费
├──────────────────────┤
│ Doorbell 寄存器       │  ← PCIe MMIO 区域
│ (SQ Tail Doorbell)   │     Host 写完 SQ 后写 Doorbell 通知 SSD
│ (CQ Head Doorbell)   │     Host 消费完 CQ 后写 Doorbell 释放槽位
└──────────────────────┘
```

### 命令提交流程

```
1. Host 准备命令条目 (Command Entry, 64 字节) ↓
2. Host 写入 SQ (环形缓冲区 Tail 位置)         ↓
3. Host 写 SQ Tail Doorbell 寄存器 (MMIO)      ↓
4. SSD 控制器检测到 Doorbell 更新               ↓
5. SSD 控制器通过 DMA 读取 SQ 中的命令           ↓
6. SSD 执行命令 (读/写/擦除 NAND)               ↓
7. SSD 控制器通过 DMA 写入 CQ 完成条目           ↓
8. SSD 发送 MSI-X 中断 (或 Host 轮询 CQ)         ← 可选中断合并
9. Host 读取 CQ 完成条目，处理结果               ↓
10. Host 写 CQ Head Doorbell 释放 CQ 槽位
```

### 队列管理

| 参数 | 说明 |
|------|------|
| 队列对数量 | 可配置，建议 ≥ CPU 核心数 |
| 队列深度 | 可配置，典型 128-1024 |
| 管理员队列 | Admin SQ/CQ（队列对 0），用于设备管理命令 |
| I/O 队列 | I/O SQ/CQ（队列对 1~N），用于数据传输 |
| 仲裁机制 | 轮询 (RR) 或加权轮询 (WRR) |

> **与核心绑定的最佳实践**：每个 CPU 核心拥有独立的 I/O 队列对，避免锁竞争。现代多核系统中，这可将 SSD 性能随核心数线性扩展。

## NVMe 命令集

### 管理员命令 (Admin Commands)

| 命令 | Opcode | 说明 |
|------|--------|------|
| Identify | 06h | 获取控制器/命名空间信息 |
| Create I/O Completion Queue | 05h | 创建 CQ |
| Create I/O Submission Queue | 01h | 创建 SQ |
| Delete I/O SQ/CQ | 00h/04h | 删除队列 |
| Set Features | 09h | 设置特性（中断合并、电源管理等） |
| Get Features | 0Ah | 获取特性 |
| Format NVM | 80h | 格式化命名空间 |

### I/O 命令

| 命令 | Opcode | 说明 |
|------|--------|------|
| NVM Read | 02h | 读取数据到 Host 内存 |
| NVM Write | 01h | 从 Host 内存写入数据 |
| Flush | 00h | 刷新写入缓存到 NAND |
| Compare | 05h | 比较 NAND 数据与 Host 内存 |
| Dataset Management | 09h | 数据管理（Trim/Discard、写提示） |

```c
// NVMe Command Entry 结构 (简化)
struct nvme_command {
    uint8_t  opcode;       // 命令操作码
    uint8_t  flags;        // 标志位 (FUSE, PSDT 等)
    uint16_t command_id;   // 命令 ID
    uint64_t nsid;         // 命名空间 ID
    uint64_t metadata;     // PRP 或 SGL
    uint64_t prp1;         // 物理区域页 1
    uint64_t prp2;         // 物理区域页 2
    uint64_t slba;         // 起始 LBA
    uint16_t nlb;          // 传输块数量 (-1)
    // ...
};
```

## NVMe over Fabrics (NVMe-oF)

NVMe-oF 将 NVMe 协议扩展到网络，使 Host 可以通过网络访问远端 NVMe 存储：

```
Host ──→ RDMA (RoCE/iWARP/InfiniBand) ──→ NVMe-oF Target ──→ NVMe SSD
         └── 或 TCP ──→ (性能较低但成本更低)
```

| 传输层 | 典型延迟 | 特点 |
|--------|---------|------|
| RDMA (RoCE v2) | ~5-15 μs | 硬件卸载，CPU 开销极低 |
| InfiniBand | ~5-10 μs | 专用互连，高端存储 |
| Fibre Channel | ~10-20 μs | 传统 SAN 扩展 |
| TCP | ~50-200 μs | 兼容性好，免专用硬件 |

## SPDK (Storage Performance Development Kit)

SPDK 是一套用户态 NVMe 驱动框架，旨在绕过操作系统内核栈，获得极致性能：

### 架构对比

```
传统内核栈:                        SPDK 用户态驱动:
  Application                        Application
    ↓                                    ↓
  VFS                                  SPDK NVMe 驱动
    ↓                                    ↓
  文件系统                             Polling Mode
    ↓                                    ↓
  Block Layer (BIO)                   Huge Pages
    ↓                                    ↓
  NVMe 内核驱动                        MMIO + DMA
    ↓                                    ↓
  Interrupt → 上下文切换             PCIe 设备
    ↓
  PCIe 设备
```

### SPDK 核心特性

| 特性 | 说明 | 性能收益 |
|------|------|---------|
| **用户态驱动** | 绕过内核，直接在用户态操作 NVMe 寄存器 | 减少系统调用和上下文切换 |
| **轮询模式 (Polling)** | 不依赖中断，持续轮询完成队列 | 降低 P99 尾部延迟 |
| **Huge Pages** | 使用 2MB/1GB 大页，减少 TLB miss | 提高 DMA 映射效率 |
| **CPU 核心绑定** | 每个控制线程绑定到独立核心 | 避免缓存抖动和锁竞争 |
| **无锁数据结构** | 使用无锁队列 (Lock-free Ring) | 线性扩展 |

```c
// SPDK "Hello World" 概念代码
#include <spdk/nvme.h>
#include <spdk/env.h>

int main(void) {
    struct spdk_env_opts opts;

    // 1. 初始化 SPDK 环境 (huge pages + core binding)
    spdk_env_opts_init(&opts);
    opts.name = "hello_world";
    opts.core_mask = "0x1";  // 绑定到 core 0
    spdk_env_init(&opts);

    // 2. 探测并连接 NVMe 设备
    struct spdk_nvme_ctrlr *ctrlr = NULL;
    spdk_nvme_probe(NULL, NULL, probe_cb, attach_cb, NULL);

    // 3. 提交 I/O 请求
    struct spdk_nvme_ns *ns;
    struct spdk_nvme_qpair *qpair;
    struct nvme_request req;  // 用户定义

    // 非阻塞提交读请求
    spdk_nvme_ns_cmd_read(ns, qpair, &buffer, lba, lba_count,
                          completion_cb, &req, 0);

    // 4. 轮询完成队列 (无中断)
    while (!req.done) {
        spdk_nvme_qpair_process_completions(qpair, 0);
        // 可在此处理其他任务
    }

    return 0;
}
```

### SPDK 性能

| 场景 | 内核 NVMe 驱动 | SPDK (用户态) |
|------|--------------|---------------|
| 4K 随机读 IOPS (单核) | ~300K | ~1M+ |
| 4K 随机读延迟 | ~6-10 μs | ~3-5 μs |
| CPU 开销 / IO | ~5000 cycles | ~1500 cycles |

## NVMe 命名空间 (Namespace)

Namespace 是 NVMe 协议中的逻辑存储单元，类似 SCSI 中的 LUN：

* 一个 NVMe 控制器可管理 **最多 N 个 Namespace**（由控制器能力决定）
* 每个 Namespace 有独立的 LBA 空间、容量、块大小
* Namespace 可被创建、删除、调整大小（NVMe 1.3+）
* **NVMe-MI** (Management Interface) 提供带外管理接口

```bash
# 查看 NVMe 设备和 Namespace
sudo nvme list

# 输出示例
Node                  SN                   Model                                    Namespace Usage                      Format           FW Rev
--------------------- -------------------- ---------------------------------------- --------- -------------------------- ---------------- --------
/dev/nvme0n1          S4FYNX0R123456A     Samsung SSD 990 PRO 2TB                   1         366.11  GB /   2.00  TB    512   B +  0 B    3B2QJXD7
/dev/nvme1n1          PHLKF12345678A      KIOXIA CM7-V 3.84TB                       1           3.84  TB /   3.84  TB    4096  B +  0 B    1.4.0

# 查看某个 NVMe 设备的详细信息
sudo nvme id-ctrl /dev/nvme0

# 查看 Namespace 信息
sudo nvme id-ns /dev/nvme0n1

# 创建新 Namespace (需要在设备支持)
sudo nvme create-ns /dev/nvme0 --nsze=1000000 --ncap=1000000 --flbas=0 -c 0

# 执行 Trim (Discard)
sudo nvme format /dev/nvme0n1 --ses=1
```

## NVMe 1.4 / 2.0 关键特性

### NVMe 1.4 (2019)

| 特性 | 说明 |
|------|------|
| **I/O Determinism** | 设置延迟上限的 QoS 保证，减少 Noisy Neighbor |
| **Persistent Event Log** | 断电后记录的扩展事件日志，便于故障诊断 |
| **Flexible Data Placement (FDP)** | Host 可控制数据放在 NAND 的物理位置，优化 GC |
| **Reclaim Group** | 支持 Host 指定哪些数据可优先回收 |
| **NVMe-MI 增强** | 改进管理接口功能 |

### NVMe 2.0 (2021)

NVMe 2.0 是协议的重大重构，将规范拆分为多个模块，并明确区分基础规范和命令集：

| 规范 | 说明 |
|------|------|
| **NVMe Base Specification** | 核心协议（队列、Admin 命令、特性等） |
| **NVM Command Set** | NAND 专用 I/O 命令 |
| **Zoned Namespace (ZNS)** | 分区命名空间：将 NAND 区域暴露给 Host |
| **Key Value (KV) Command Set** | KV 存储命令 |
| **Compute Command Set** | 计算存储命令（在 SSD 上执行计算） |
| **Subsystem Local Memory** | 允许 SSD 控制器访问 Host 内存 |

### ZNS (Zoned Namespace)

ZNS 是 NVMe 2.0 的重要创新，将 NAND 的 Zone (Block) 概念暴露给 Host：

* **Host 管理 GC**：Host 知道数据的物理布局，自主管理 GC
* **更低 WA**：消除 FTL 的 GC 回写，WAF 接近 1.0
* **更高吞吐**：无需 FTL 地址转换开销
* **可预测延迟**：没有 FTL GC 导致的延迟抖动

```
传统 SSD:                        ZNS SSD:
Host 发送随机 LBA                 Host 管理 Zone:
    ↓                                - 按 Zone 顺序写入
FTL 管理所有:                        - 显式 Zone 重置(擦除)
  - L2P 映射                        - 数据在 Zone 内连续
  - GC + WA                        - Host 自行 GC
  - 性能不稳定                      - 性能确定+低 WA
```

## 来源

* [NVM Express Base Specification 2.0](https://nvmexpress.org/developers/)
* [NVM Express 1.4 Specification](https://nvmexpress.org/specification/nvme-1-4/)
* [SPDK 官方文档](https://spdk.io/doc/)
* [Linux NVMe 驱动源码: drivers/nvme/](https://elixir.bootlin.com/linux/latest/source/drivers/nvme)
* [NVMe over Fabrics 介绍](https://nvmexpress.org/wp-content/uploads/NVMe-over-Fabrics.pdf)
* [AHCI vs NVMe: A Protocol Comparison](https://www.delkin.com/blog/ahci-vs-nvme/)
* [ZNS: Avoiding the Block Interface Tax](https://www.snia.org/sites/default/files/ESF/2020/2020-SNIA-ESF-ZNS-Avoiding-the-Block-Interface-Tax.pdf) — SNIA
* [Understanding NVMe Queues](https://nvmexpress.org/blog/understanding-nvme-queues/)
