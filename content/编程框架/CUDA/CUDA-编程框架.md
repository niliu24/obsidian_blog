---
tags: [cuda, gpu, parallel-computing, programming-model]
type: entity
---

> CUDA 编程框架建立在 Host-Device 异构模型之上，以**线程层次**和 SIMT 执行模型为核心抽象，通过 Kernel 函数在 GPU 上调度海量线程并行计算。

## 程序基本结构

CUDA 程序在 Host（CPU）和 Device（GPU）两个独立内存空间上运行：

```
1. 选择 GPU 设备 (cudaSetDevice)
2. 分配 Device 内存 (cudaMalloc)
3. 拷贝数据 Host → Device (cudaMemcpy, H2D)
4. 启动 Kernel (<<<grid, block>>>)
5. 同步等待完成 (cudaDeviceSynchronize)
6. 拷贝结果 Device → Host (cudaMemcpy, D2H)
7. 释放 Device 内存 (cudaFree)
```

### 最简示例

```cpp
#include <cuda_runtime.h>
#include <stdio.h>

__global__ void hello(int N) {
    int gid = blockIdx.x * blockDim.x + threadIdx.x;
    if (gid < N) {
        printf("Hello from thread %d\n", gid);
    }
}

int main() {
    hello<<<4, 64>>>(256);
    cudaDeviceSynchronize();
    return 0;
}
```

编译: `nvcc hello.cu -o hello`

## 线程层次

```
Grid → Block → Warp (32 threads) → Thread
```

| 层级 | 说明 | 内建变量 |
|------|------|----------|
| **Grid** | 一次 Kernel 启动的所有 Block | `gridDim` |
| **Block** | 同一 SM 内执行的线程组，可共享 `__shared__` 内存并同步 | `blockDim`, `blockIdx` |
| **Warp** | 硬件调度基本单元，固定 **32 线程**，SIMT 执行 | — |
| **Thread** | 最小执行单元 | `threadIdx` |

### 维度支持

Grid 和 Block 均可为 1D/2D/3D，方便映射到多维数据（图像、矩阵、体素）。

| 限制 | 值 |
|------|-----|
| Grid X 维度 | 最长 2³¹−1 个 Block |
| Block 总线程数 | 最大 **1024** |
| Block 各维度最大 | x: 1024, y: 1024, z: 64 |

### 线程索引计算

**1D**:

```cpp
int gid = blockIdx.x * blockDim.x + threadIdx.x;
```

**2D Grid + 2D Block**:

```cpp
int x = blockIdx.x * blockDim.x + threadIdx.x;
int y = blockIdx.y * blockDim.y + threadIdx.y;
int gid = y * (gridDim.x * blockDim.x) + x;
```

## Warp —— 硬件执行单元

Warp 是 GPU 硬件调度和执行的基本单位，固定 **32 个线程**。程序员编写的是单线程代码，硬件以 Warp 为单位执行。

### SIMT 执行模型

* 一个 Warp 内所有线程同时执行**同一条指令**（Single Instruction, Multiple Threads）
* 每个线程有独立的寄存器和程序计数器
* Block 被划分为整数个 Warp（如 70 线程 → 3 个 Warp，最后一个 Warp 只有 6 个活跃 lane）

### Warp Divergence（线程束分化）

同一 Warp 内线程走不同分支时，Warp 必须**串行执行两条路径**——一条执行时另一条被屏蔽。

```
if (threadIdx.x < 16) {
    // 路径 A: lane 0-15 执行，lane 16-31 屏蔽
} else {
    // 路径 B: lane 16-31 执行，lane 0-15 屏蔽
}
```

**缓解策略**:

* 将分支对齐到 Warp 边界
* 用算术替代分支（谓词化）
* 重组数据使同一 Warp 走相同路径

### Block 大小建议

* Block 内线程数应为 **32 的倍数**（最大 Warp 利用率）
* 每个 Block 至少 **128-256 线程**以隐藏延迟
* Block 数量远大于 SM 数量以保证负载均衡

## 硬件映射

| 软件抽象 | 硬件单元 |
|----------|----------|
| Thread | CUDA Core（Warp 内一个 lane） |
| Warp (32 threads) | Warp Scheduler 调度单元 |
| Block | Streaming Multiprocessor (SM) |
| Grid | 整个 GPU |

### SM（Streaming Multiprocessor）

```
SM 内部组成:
┌──────────────────────────────────┐
│ Register File (如 Blackwell 64K) │
│ Shared Memory / L1 Cache (可配)  │
│ Warp Schedulers → Dispatch Units │
│ CUDA Cores │ Tensor Cores │ SFU  │
└──────────────────────────────────┘
```

* 一个 SM 可同时驻留多个 Block
* Warp 切换**零开销**——当一个 Warp 等待内存时，调度器立即切到另一个就绪 Warp
* **Occupancy** = 活跃 Warp 数 ÷ SM 最大 Warp 数，高 Occupancy 有助于隐藏延迟

## Kernel 语言关键字

| 关键字 | 用途 |
|--------|------|
| `__global__` | 标记 Kernel 函数，Host 调用，Device 执行 |
| `__device__` | 仅 Device 端可调用 |
| `__host__` | 仅 Host 端可调用（默认） |
| `__host__ __device__` | Host 和 Device 均可调用 |
| `__shared__` | Block 内线程共享内存 |
| `__constant__` | 常量内存（Device 只读，有专用缓存） |

### Device 端特性

* `printf` — Kernel 内打印（SM 2.0+）
* `__syncthreads()` — Block 内线程同步屏障
* `__threadfence()` / `__threadfence_block()` — 内存 fence
* Warp-level 原语: `__shfl_sync`, `__ballot_sync`, `__any_sync`, `__all_sync`
* 动态并行: Kernel 内再启动 Kernel（SM 3.5+）
* `atomicAdd` / `atomicCAS` 等原子操作

## Kernel 启动

### Launch 语法

```cpp
kernel<<<dim3(gridDims), dim3(blockDims), sharedMemBytes, stream>>>(args);
```

| 参数 | 类型 | 说明 |
|------|------|------|
| `gridDims` | `dim3` | Grid 各维度 Block 数量 |
| `blockDims` | `dim3` | Block 各维度线程数 |
| `sharedMemBytes` | `size_t` | 动态共享内存字节数（可选，默认 0） |
| `stream` | `cudaStream_t` | 异步 Stream（可选，默认 0） |

### Launch 过程

1. 参数打包（marshaling）
2. 运行时校验（线程数、共享内存等硬件限制）
3. 命令入队到 CUDA Stream
4. **Host 异步返回**——GPU 在资源就绪时执行

> **注意**: `<<<>>>` 是**异步**的，Host 立即返回，必须显式同步（`cudaDeviceSynchronize` 或 `cudaStreamSynchronize`）。

## 内存层次

| 内存类型 | 位置 | 作用域 | 速度 | 生命周期 |
|----------|------|--------|------|----------|
| **Registers** | On-chip (SM) | Thread | 最快 | Thread |
| **Shared Memory** (`__shared__`) | On-chip (SM) | Block | 很快 | Block |
| **L1 Cache** | On-chip (SM) | SM 内 | 快 | — |
| **L2 Cache** | On-chip | 全设备 | 中等 | — |
| **Global Memory** | Device DRAM | Grid + Host | 慢 | 程序 |
| **Constant Memory** (`__constant__`) | Device DRAM (有缓存) | Grid | 快（命中） | 程序 |
| **Local Memory** | Device DRAM | Thread | 慢 | Thread |

### 关键优化点

* **合并访问（Coalesced Access）**: Warp 内线程访问连续地址，减少内存事务
* **Bank Conflict**: Shared Memory 分 32 Bank，避免多线程同时访问同一 Bank
* **寄存器溢出（Register Spilling）**: 寄存器不足时溢出到 Local Memory，性能剧降

## 编译模型

### 编译器: nvcc

`nvcc` 分离 Host/Device 代码：

* Device 代码 → PTX（中间表示）→ SASS（GPU 机器码）
* Host 代码 → 系统 C++ 编译器

### 编译选项

| 选项 | 说明 |
|------|------|
| `-arch=sm_xx` | 指定目标架构（如 `sm_80`=A100, `sm_90`=H100, `sm_120`=Blackwell） |
| `-G` | 生成 Device 调试信息（`cuda-gdb` 可用） |
| `-O2` / `-O3` | 优化级别 |
| `-lineinfo` | 保留行号信息 |
| `--ptxas-options=-v` | 查看寄存器/共享内存使用量 |

### 架构代号对照

| 架构 | Compute Capability | 代表 GPU |
|------|-------------------|----------|
| Volta | 7.0 / 7.2 | V100 |
| Turing | 7.5 | T4, RTX 2080 |
| Ampere | 8.0 / 8.6 | A100, RTX 3090 |
| Hopper | 9.0 | H100 |
| Blackwell | 12.0 | B100/B200 |

## 来源

* [NVIDIA CUDA C++ Programming Guide](https://docs.nvidia.com/cuda/cuda-c-programming-guide/)
* [Professional CUDA C Programming — CUDA Execution Model](https://blog.devgenius.io/professional-cuda-c-programming-3-cuda-execution-model-d5849c98528a)
