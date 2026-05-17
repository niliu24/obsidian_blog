---
tags: [gpu, pcie, dma, data-transfer, hardware]
type: entity
aliases:
  - GPU 与 CPU 数据传输
  - GPU DMA 传输
  - cudaMemcpy 原理
---

> GPU 与 CPU 之间的数据传输是异构计算的核心瓶颈之一。物理上通过 PCIe 总线连接，数据路径由 GPU 的 DMA 引擎自主完成，控制路径通过 MMIO 写入 Doorbell 寄存器触发。软件层面，CUDA 提供了 cudaMemcpy、Pinned Memory、Unified Memory 等多层抽象。

## 整体架构

CPU 和 GPU 是独立的计算设备，各有自己的内存空间，通过 PCIe 总线连接：

```
┌─────────────────────┐          PCIe          ┌─────────────────────┐
│        CPU           │    ◄══════════►       │        GPU           │
│                      │                       │                      │
│  ┌────────────────┐  │                       │  ┌────────────────┐  │
│  │   用户进程       │  │                       │  │   CUDA Kernel  │  │
│  │  (Host 内存)    │  │    ┌───────────┐      │  │  (Device 内存)  │  │
│  │                │  │    │ DMA Engine│       │  │                │  │
│  │  ┌──────────┐  │  │    │  (GPU 侧) │       │  │  ┌──────────┐  │  │
│  │  │ Pinned   │◄─┼──┼────┤          ├───────┼──┼─►│ 显存 VRAM │  │  │
│  │  │ Memory   │  │  │    │ RDMA/WRRD│       │  │  │ (HBM/GDDR)│  │  │
│  │  └──────────┘  │  │    └───────────┘      │  │  └──────────┘  │  │
│  │                │  │                       │  │                │  │
│  │  ┌──────────┐  │  │    ┌───────────┐      │  │  ┌──────────┐  │  │
│  │  │ 普通分页  │  │  │    │  MMIO     │      │  │  │ BAR 映射  │  │  │
│  │  │ 内存     │  │  │    │ 寄存器    │◄─────┼──┼──┤ (寄存器)  │  │  │
│  │  └──────────┘  │  │    └───────────┘      │  │  └──────────┘  │  │
│  └────────────────┘  │                       │  └────────────────┘  │
│                      │                       │                      │
│  MMIO 访问 (控制面)   │                       │  DMA 读写 (数据面)    │
└─────────────────────┘                       └─────────────────────┘
```

核心概念区分：

| 维度 | 控制面 | 数据面 |
|------|--------|--------|
| **做什么** | CPU 告诉 GPU " 传输数据 "/" 执行 Kernel" | 实际的数据搬运 |
| **谁发起** | CPU (通过 MMIO 写 Doorbell) | GPU DMA 引擎 (自主执行) |
| **传输内容** | 命令、描述符指针、少量寄存器值 | 大量数据 (MB~GB 级) |
| **路径** | CPU → MMIO → GPU 寄存器 | GPU DMA → PCIe → 系统内存 |
| **软件接口** | `iowrite32()` / Command Buffer 提交 | `cudaMemcpy()` → GPU 提交 DMA 描述符 |

## 控制面：CPU 如何对 GPU 下命令

### 1. BAR 暴露 GPU 资源

GPU 通过 [BAR](BAR.md) 将自己的寄存器和部分显存映射到 CPU 的物理地址空间：

```
典型 GPU BAR 布局:

BAR0+BAR1 (64-bit, 16GB, Prefetchable) → VRAM 映射 (CPU 可直接读写显存)
BAR2+BAR3 (64-bit, 4MB,  Non-prefetchable) → Doorbell 寄存器
BAR5      (32-bit, 256KB, Non-prefetchable) → MMIO 控制寄存器
```

系统启动时 BIOS/OS 执行 PCIe 枚举，为 BAR 分配物理地址范围。BAR 类型决定了 CPU 的访问策略：

| BAR 属性 | 访问方式 | 用途 | 示例 |
|----------|---------|------|------|
| **Prefetchable** | 可缓存、可合并读、无副作用 | 显存映射 (CPU 可以直接读 VRAM 内容) | BAR0+BAR1 |
| **Non-prefetchable** | 不可缓存 (UC)、读写有副作用 | 寄存器访问 (Doorbell、控制寄存器) | BAR2+BAR3, BAR5 |

### 2. MMIO 控制寄存器

CPU 通过 [MMIO](MMIO.md) 访问 GPU 的控制寄存器，主要分为两类：

**命令提交通道（Command Ring/Queue）**：

GPU 维护一个环形命令缓冲区（Ring Buffer），CPU 写入命令，GPU 消费命令：

```
CPU (Host)                              GPU (Device)
   │                                       │
   ├─ 写入命令描述符到 Ring Buffer          │
   │  (GPU 可 DMA 读取的 Host 内存)         │
   │                                       │
   ├─ iowrite32(tail, doorbell) ──────────►│ 收到 Doorbell
   │                                       ├─ DMA 读取 Ring 中的新命令
   │                                       ├─ 解析并执行命令
   │                                       └─ 更新 Head 指针
```

**Doorbell 寄存器**：

CPU 写入 Doorbell 是通知 GPU " 有新任务 " 的最关键操作：

```c
// GPU 驱动将 Doorbell 寄存器映射为 Write-Combine 内存
void __iomem *doorbell = ioremap_wc(doorbell_phys_addr, PAGE_SIZE);

// 写入 Doorbell：告知 GPU Ring Buffer 的新 Tail 位置
iowrite32(new_tail_index, doorbell + QUEUE_DOORBELL_OFFSET);
```

使用 Write-Combine 映射而非 Uncacheable，是为了允许 CPU 将连续写入合并为一次 PCIe TLP，显著提升命令提交性能。

### 3. 命令提交全流程

```
用户空间:
  cuLaunchKernel() 或 cuMemcpyHtoD()
       │
       ▼
CUDA 用户态驱动:
  构造 Command Buffer（GPU 可执行的命令包）
       │
       ▼
CUDA 内核态驱动:
  1. 将 Command Buffer 写入 GPU 可 DMA 访问的 Host 内存
  2. 更新 Ring Buffer Tail 指针
  3. iowrite32(tail, doorbell) → 通知 GPU
       │
       ▼
GPU:
  4. 收到 Doorbell 信号
  5. DMA 读取 Ring Buffer 中的新命令
  6. 解析命令 → 发现是 "DMA 传输" 命令
  7. 启动 DMA Engine 执行数据传输 (见下一节)
  8. 传输完成 → 可选发送 MSI/MSI-X 中断通知 CPU
```

## 数据面：GPU DMA 如何搬运数据

### 1. DMA 传输方向

GPU 的 DMA Engine 可以自主发起 PCIe Memory TLP 读写系统内存：

| 方向 | GPU 侧操作 | PCIe TLP 类型 | 典型场景 |
|------|-----------|---------------|----------|
| **H2D (Host to Device)** | GPU DMA 读系统内存，写入 VRAM | Memory Read Request | 将输入数据送入 GPU |
| **D2H (Device to Host)** | GPU DMA 读 VRAM，写入系统内存 | Memory Write Request | 取回计算结果 |
| **D2D (Device to Device)** | GPU DMA 读写对端 GPU 显存 | Memory Read/Write (通过 PCIe Switch 或 NVLink) | 多 GPU 数据交换 |

### 2. DMA 传输的硬件流程（H2D 为例）

```
Step 1: CPU 准备数据
  CPU 在系统内存中准备待传输数据
  关键：必须使用 Pinned Memory（页锁定），否则页面可能被换出

Step 2: CPU 提交 DMA 命令
  CPU 构造 DMA 描述符（包含：源地址、目标地址、传输大小）
  CPU 通过 Doorbell 通知 GPU

Step 3: GPU DMA Engine 执行传输
  GPU DMA Engine 读取描述符
  → 发起 PCIe Memory Read TLP，从系统内存读取数据
  → CPU Root Complex 将 TLP 转发到内存控制器
  → 内存控制器返回数据（通过 PCIe Completion TLP）
  → GPU DMA Engine 将数据写入 VRAM

Step 4: 传输完成
  GPU 更新完成标志（Fence）
  → 可选：发送 MSI/MSI-X 中断通知 CPU
  → CPU 检查 Fence 值确认传输完成
```

### 3. DMA 描述符（Simplified View）

GPU 命令中包含 DMA 描述符，描述一次传输的源、目标、大小：

```
┌─────────────────────────────┐
│  DMA 描述符                  │
├─────────────────────────────┤
│  Src Address: 系统内存物理地址│ ← CPU 填充 (Pinned Memory)
│  Dst Address: VRAM 偏移      │ ← GPU 内部地址
│  Size:        传输字节数      │
│  Flags:       方向/选项      │
│  Fence ID:    完成信号 ID    │ ← CPU/GPU 通过 Fence 同步
└─────────────────────────────┘
```

多个 DMA 描述符可以链接成链，GPU 顺序或并行处理（取决于 GPU 的 DMA Engine 数量）。

## 软件层抽象：CUDA 内存模型

### 1. 基本数据拷贝：cudaMemcpy

最基础的显式数据传输 API：

```cpp
// Host → Device
cudaMemcpy(d_A, h_A, bytes, cudaMemcpyHostToDevice);

// Device → Host
cudaMemcpy(h_C, d_C, bytes, cudaMemcpyDeviceToHost);

// Device → Device (同一 GPU 内)
cudaMemcpy(d_B, d_A, bytes, cudaMemcpyDeviceToDevice);
```

`cudaMemcpy` 是**同步**的：调用阻塞直到传输完成。底层流程：

1. CUDA Runtime → CUDA Driver → 内核态驱动
2. 驱动提交 DMA 命令到 GPU Command Queue
3. 驱动等待 GPU 完成（轮询 Fence 或等待中断）

同步传输意味着 Host 线程空闲等待，浪费 CPU。数据传输时间 = 数据量 / PCIe 有效带宽。

### 2. Pinned Memory（页锁定内存）

普通 `malloc` 分配的内存是可分页的（Pageable），GPU DMA 不能直接访问——因为 OS 可能随时将页面换出到磁盘。

```cpp
// 错误：Pageable 内存，cudaMemcpy 内部会先拷贝到临时 Pinned Buffer
float *h_A = (float*)malloc(bytes);
cudaMemcpy(d_A, h_A, bytes, cudaMemcpyHostToDevice);
// 实际流程: Pageable Mem → 内部 Pinned Buffer → DMA → VRAM
//           (多一次 CPU 内存拷贝，带宽减半)

// 正确：Pinned Memory，GPU DMA 直接读写
float *h_A_pinned;
cudaHostAlloc(&h_A_pinned, bytes, cudaHostAllocDefault);
cudaMemcpy(d_A, h_A_pinned, bytes, cudaMemcpyHostToDevice);
// 流程: Pinned Mem → DMA → VRAM (无中间拷贝)
```

对比：

| 特性 | Pageable Memory | Pinned Memory |
|------|----------------|---------------|
| **分配函数** | `malloc` / `new` | `cudaHostAlloc()` |
| **GPU DMA 直接访问** | 否（需内部 staging buffer） | 是 |
| **传输带宽** | 约 PCIe 带宽的 50% | 接近 PCIe 理论带宽 |
| **传输需要** | 2 次拷贝 | 1 次拷贝 |
| **系统影响** | 无 | 减少可用物理页 (不可换出) |
| **适用场景** | 小数据量、非频繁传输 | 高频传输、大块数据 |

> Pinned Memory 会减少操作系统可换出的物理页数量，分配过多会导致系统内存紧张。因此应仅对需要频繁 GPU 传输的缓冲区使用。

### 3. 异步传输与 Stream 重叠

```cpp
cudaStream_t stream;
cudaStreamCreate(&stream);

// 异步拷贝：立即返回，不等待完成
cudaMemcpyAsync(d_A, h_A_pinned, bytes, cudaMemcpyHostToDevice, stream);

// Kernel 可以在同一 Stream 上排队等待传输完成
kernel<<<grid, block, 0, stream>>>(d_A, d_C, N);

// Host 继续执行其他工作
do_something_on_cpu();

// 最终同步
cudaStreamSynchronize(stream);
```

关键限制：异步传输的 Host 端内存必须是 **Pinned Memory**，否则 CUDA Runtime 会退化为同步传输。

**传输与计算重叠**（使用多个 Stream）：

```
Stream 1:  [H2D chunk1] [Kernel chunk1] [D2H chunk1]
Stream 2:       [H2D chunk2] [Kernel chunk2] [D2H chunk2]
Stream 3:            [H2D chunk3] [Kernel chunk3] [D2H chunk3]
                 ↑             ↑                ↑
           传输与计算重叠    不同 Stream 间的 Kernel 共享 SM

时间 ──────────────────────────────────────────────────────►
```

这种流水线设计可以隐藏 PCIe 传输延迟和 Kernel 启动延迟。

### 4. Unified Memory（统一内存）

CUDA 6.0 引入的统一内存提供单一地址空间，CPU 和 GPU 共享指针：

```cpp
// 分配统一内存
float *data;
cudaMallocManaged(&data, bytes);

// CPU 访问
for (int i = 0; i < N; i++) data[i] = i;

// GPU 直接访问（Page Fault 触发自动迁移）
kernel<<<grid, block>>>(data, N);
cudaDeviceSynchronize();

// CPU 再次访问（又一次 Page Fault 迁移回来）
printf("result[0] = %f\n", data[0]);

cudaFree(data);
```

**Unified Memory 的 Page Fault 机制**（Pascal+）：

```
CPU 访问 data[i]                         GPU 访问 data[i]
     │                                        │
     ▼                                        ▼
  TLB Miss ──► Page Table Walk              TLB Miss ──► GPU Page Table
     │                                        │
     ▼                                        ▼
  页面在 VRAM 中? ────是───► 触发 PCIe       页面在系统内存? ──是──► GPU
  Page Fault，从 VRAM                         MMU 触发 Page Fault
  迁移页面到系统内存                          从系统内存迁移到 VRAM
     │                                        │
     ▼                                        ▼
  CPU 继续访问                               GPU 继续访问
```

**与显式 cudaMemcpy 的对比**：

| 特性 | 显式 cudaMemcpy | Unified Memory |
|------|----------------|----------------|
| **编程复杂度** | 手动管理拷贝 | 自动迁移 (透明) |
| **性能** | 可控 (手动优化时机) | 可能产生意外 Page Fault |
| **内存上限** | Device 显存大小 | CPU RAM + GPU VRAM (Oversubscription) |
| **适合场景** | 性能敏感、数据流清晰 | 快速原型、复杂数据结构 (树/图) |
| **Page Fault 延迟** | 无 | 微秒级 (PCIe 往返延迟) |
| **预取控制** | 不适用 | `cudaMemPrefetchAsync()` |

**优化 Unified Memory 性能的最佳实践**：

```cpp
float *data;
cudaMallocManaged(&data, bytes);

// 在 CPU 访问前预取到 CPU (避免 GPU→CPU Page Fault)
cudaMemPrefetchAsync(data, bytes, cudaCpuDeviceId);

// CPU 初始化数据
init_data(data);

// 在 GPU 访问前预取到 GPU (避免 CPU→GPU Page Fault)
int gpu_id = 0;
cudaMemPrefetchAsync(data, bytes, gpu_id);

// GPU Kernel
kernel<<<grid, block>>>(data, N);
cudaDeviceSynchronize();

// 预取结果回 CPU
cudaMemPrefetchAsync(data, bytes, cudaCpuDeviceId);
cudaDeviceSynchronize();
```

> 从 Pascal 架构开始，Unified Memory 支持硬件 Page Fault 和按需页面迁移，消除了软件 TLB Shootdown 的开销，使按需迁移延迟降至微秒级。

## 高速互联：超越 PCIe

### NVLink —— NVIDIA 专有互联

NVLink 是 NVIDIA 的专有高带宽互联，GPU 之间直接通信，不经过 PCIe Switch：

| 特性 | PCIe Gen5 x16 | NVLink 4.0 (H100) | NVLink 5.0 (B200) |
|------|--------------|-------------------|-------------------|
| **单链路带宽 (单向)** | ~63 GB/s | 50 GB/s | 100 GB/s |
| **链路数** | x16 | 18 条 | 18 条 |
| **总带宽 (单向)** | ~63 GB/s | 900 GB/s | 1800 GB/s |
| **拓扑** | 树形 (通过 Switch) | 全互联/网状 | 全互联/网状 |
| **场景** | CPU ↔ GPU | GPU ↔ GPU | GPU ↔ GPU |

NVLink 也支持 CPU-GPU 互联（如 Grace Hopper 中 CPU 与 GPU 通过 NVLink-C2C 互联，带宽达 900 GB/s）。

### CXL —— 开放标准的缓存一致性互联

[CXL](总线/CXL.md)（Compute Express Link）基于 PCIe 物理层，增加了缓存一致性协议：

```
CXL 三种协议:

CXL.io   ── 基于 PCIe，用于设备发现、配置、MMIO
CXL.cache ── 设备可缓存主机内存（低延迟访问）
CXL.mem   ── 主机可缓存设备内存（内存扩展）
```

CXL 3.0 支持设备间直接 P2P 通信和多级交换，可能成为未来开放 GPU 互联的基础。

### 多 GPU 数据交换

```
方案1: 通过 PCIe Switch
  GPU0 ↔ PCIe Switch ↔ GPU1
  延迟: 较高 (经过 Switch 转发)
  带宽: 共享 PCIe 上行带宽

方案2: 通过 NVLink
  GPU0 ↔ NVLink Bridge/Switch ↔ GPU1
  延迟: 低 (直连协议)
  带宽: 高 (数百 GB/s)，不占用 PCIe 带宽

方案3: GPUDirect RDMA
  GPU0 ↔ NIC ↔ 网络 ↔ NIC ↔ GPU1
  场景: 跨节点 GPU 通信
  优势: 绕过 CPU 内存，GPU 直接读写 RDMA 网卡
```

CUDA 中对 P2P 访问的支持：

```cpp
// 启用 GPU Peer Access
int canAccessPeer;
cudaDeviceCanAccessPeer(&canAccessPeer, 0, 1);
if (canAccessPeer) {
    cudaDeviceEnablePeerAccess(1, 0);  // GPU1 允许 GPU0 访问

    // 直接 P2P Memcpy
    cudaMemcpy(d_data_gpu1, d_data_gpu0, bytes, cudaMemcpyDeviceToDevice);
    // 如果通过 NVLink，则走 NVLink；否则走 PCIe Switch
}
```

## 完整数据流全景

从用户空间代码到硬件执行的完整路径（以 `cudaMemcpy(dst, src, size, H2D)` 为例）：

```
用户代码:
  cudaMemcpy(d_A, h_A, bytes, cudaMemcpyHostToDevice);
       │
       ▼
CUDA Runtime (libcudart.so):
  └► CUDA Driver API (libcuda.so)
       │  ioctl(/dev/nvidiaX, ...)
       ▼
内核态驱动 (nvidia.ko):
  1. 锁定 Host 内存页面 (若为 Pageable，先做内部 staging copy)
  2. 获取 Host 缓冲区的物理地址 (DMA 地址)
  3. 获取 Device 缓冲区在 VRAM 中的偏移
  4. 构造 DMA 描述符 (Src=Host物理地址, Dst=VRAM偏移, Size=N)
  5. 写入 GPU Command Ring Buffer
  6. iowrite32(tail, doorbell) ─────────── MMIO (控制面)
       │
       ▼
GPU 硬件:
  7. 收到 Doorbell ─► 解析 Ring Buffer ─► 发现 DMA 命令
  8. GPU DMA Engine 发起 PCIe Memory Read TLP   (数据面)
  9. Root Complex 将 TLP 转发到内存控制器
  10. 内存控制器返回数据 ─► PCIe Completion TLP ─► GPU VRAM
  11. 传输完成 ─► GPU 写入 Fence 值
       │
       ▼
CPU 轮询或中断:
  12. 驱动检测 Fence 完成 (或收到 MSI 中断)
  13. 解除 Host 页面锁定
  14. 返回到用户空间
```

延迟拆解（典型值，PCIe Gen4 x16）：

| 阶段 | 延迟 | 说明 |
|------|------|------|
| 用户态 → 内核态 | ~1 μs | ioctl 系统调用开销 |
| 内核态构造命令 + Doorbell | ~1-2 μs | MMIO 写入 |
| GPU 解析命令 + 启动 DMA | ~2-5 μs | GPU 调度延迟 |
| PCIe 传输延迟 | ~500 ns | TLP 往返延迟 (含 RC + Memory Controller) |
| 实际数据传输 | 数据量 / 带宽 | Gen4 x16: ~31.5 GB/s 有效带宽 |
| GPU Fence 写入 + 中断延迟 | ~1-3 μs | 中断处理路径 |
| 总开销 (小数据) | ~10 μs | 固定开销主导 |
| 总开销 (1 MB 数据) | ~40 μs | 31.5 GB/s → ~32 μs 传输 + ~10 μs 开销 |

> 对于小数据量传输，固定开销（命令提交、GPU 调度、中断）远大于实际数据传输时间。这是 CUDA Kernel Launch 也需要批量化的原因。

## 性能优化要点

| 优化策略 | 原理 | 适用场景 |
|----------|------|----------|
| **使用 Pinned Memory** | 避免内部 staging buffer 拷贝 | 所有高频传输 |
| **批量传输** | 将多次小传输合并为一次大传输 | 分散的小数据 |
| **异步 + Stream 重叠** | 隐藏 PCIe 传输延迟 | 连续数据流 |
| **双缓冲 (Double Buffering)** | 一个缓冲区传输时另一个在计算 | 迭代算法 |
| **Unified Memory + Prefetch** | 按需预取，减少 Page Fault | 复杂数据结构 |
| **GPUDirect RDMA** | GPU ↔ 网卡直传，绕过 CPU 内存 | 多节点分布式训练 |
| **NVLink P2P** | 绕过 PCIe，GPU 间直接通信 | 多 GPU 训练/推理 |
| **减少固定开销** | 使用 CUDA Graph 合并多次 Launch/Copy | 重复性工作流 |

## 相关文档

* [GPU 通用计算](GPU通用计算.md) — SIMT 执行模型、Tensor Core、Roofline
* [GPU 架构与渲染管线](GPU架构与渲染管线.md) — SM/CU 微架构、内存层次
* [PCIe](../总线/PCIe.md) — 总线拓扑、TLP 类型、代际带宽
* [BAR](../总线/BAR.md) — GPU 内存和寄存器如何映射到 CPU 地址空间
* [MMIO](../总线/MMIO.md) — CPU 如何访问 GPU 寄存器
* [DMA](../总线/DMA.md) — Direct Memory Access 原理与 Linux API
* [CXL](../总线/CXL.md) — 缓存一致性互联
* [MSI/MSI-X](../中断/MSI.md) — PCIe 消息中断机制
* [CUDA 编程框架](../../编程框架/CUDA/CUDA-编程框架.md) — CUDA 线程模型与内存层次
* [CUDA API 参考](../../编程框架/CUDA/CUDA-API参考.md) — 内存管理 API 详情

## 来源

* [NVIDIA CUDA C++ Programming Guide — Device Memory Access](https://docs.nvidia.com/cuda/cuda-c-programming-guide/index.html)
* [NVIDIA CUDA Best Practices Guide — Data Transfer](https://docs.nvidia.com/cuda/cuda-c-best-practices-guide/index.html#data-transfer-between-host-and-device)
* [PCI Express Base Specification Revision 6.0](https://pcisig.com/specifications)
* [NVIDIA NVLink Overview](https://www.nvidia.com/en-us/data-center/nvlink/)
* [CXL Consortium — Compute Express Link Specification](https://www.computeexpresslink.org/)
* [Linux kernel DMA API documentation](https://www.kernel.org/doc/html/latest/core-api/dma-api.html)
* [NVIDIA GPU Direct RDMA](https://developer.nvidia.com/gpudirect)
