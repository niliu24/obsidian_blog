---
tags: [hip, gpu, parallel-computing, programming-model]
type: entity
---

> HIP 编程框架在语法和模型上与 [CUDA](../CUDA/CUDA.md) 高度一致，核心差异在于编译工具链和后端适配。

## 程序基本结构

HIP 程序运行在 **Host（CPU）+ Device（GPU）** 异构架构上：

```
1. 选择 GPU 设备 (hipSetDevice)
2. 分配 Device 内存 (hipMalloc)
3. 拷贝数据 Host → Device (hipMemcpy, H2D)
4. 启动 Kernel (hipLaunchKernelGGL 或 <<<>>>)
5. 同步等待完成 (hipDeviceSynchronize)
6. 拷贝结果 Device → Host (hipMemcpy, D2H)
7. 释放 Device 内存 (hipFree)
```

### 最简示例

```cpp
#include <hip/hip_runtime.h>
#include <stdio.h>

__global__ void hello(int N) {
    int gid = blockIdx.x * blockDim.x + threadIdx.x;
    if (gid < N) {
        printf("Hello from block %d, thread %d\n", blockIdx.x, threadIdx.x);
    }
}

int main() {
    int gridSize = 4, blockSize = 4;
    int N = gridSize * blockSize;
    hello<<<gridSize, blockSize>>>(N);
    hipDeviceSynchronize();
    return 0;
}
```

编译: `hipcc hello.cpp -o hello`

## 线程层次

```
Grid → Block → Thread (Wavefront)
```

| 层级 | 说明 | HIP 内建变量 |
|------|------|-------------|
| **Grid** | 一次 Kernel 启动的所有线程 | `gridDim` |
| **Block** | Grid 的子集，Block 内线程可同步、共享 `__shared__` 内存 | `blockDim`, `blockIdx` |
| **Thread** | 最小执行单元 | `threadIdx` |
| **Wavefront (Warp)** | SIMD 硬件调度单元，AMD 通常 64 线程（NVIDIA Warp 为 32） | `__AMDGCN_WAVEFRONT_SIZE` |

### 线程索引计算 (1D)

```cpp
int gid = blockIdx.x * blockDim.x + threadIdx.x;
```

### 索引计算 (2D)

```cpp
int x = blockIdx.x * blockDim.x + threadIdx.x;
int y = blockIdx.y * blockDim.y + threadIdx.y;
int gid = y * (gridDim.x * blockDim.x) + x;
```

## Kernel 语言关键字

| 关键字 | 用途 |
|--------|------|
| `__global__` | 标记 GPU Kernel 函数，Host 调用，Device 执行 |
| `__device__` | 仅 Device 端可调用的函数 |
| `__host__` | 仅 Host 端可调用的函数（默认） |
| `__host__ __device__` | Host 和 Device 均可调用 |
| `__shared__` | Block 内线程共享内存 |
| `__constant__` | 常量内存（Device 只读） |

### Device 端支持的特性

* `printf` — Kernel 内打印调试
* 动态 `malloc`/`free` — 有限设备堆内存分配
* `__syncthreads()` — Block 内线程同步屏障
* `__threadfence()` — 全局内存 fence
* 跨 lane 操作: `__shfl`, `__ballot`, `__any`, `__all`
* 虚函数调用（有限支持）

## Kernel 启动方式

### 方式一: Chevron 语法 (`<<<>>>`)

```cpp
kernel<<<dim3(gridX, gridY), dim3(blockX, blockY), sharedMemBytes, stream>>>(args);
```

### 方式二: `hipLaunchKernelGGL` 宏（推荐）

```cpp
hipLaunchKernelGGL(
    kernel,            // Kernel 函数名
    dim3(gridSize),    // Grid 维度
    dim3(blockSize),   // Block 维度
    0,                 // 动态共享内存字节数
    0,                 // Stream (0 = 默认 Stream)
    arg1, arg2, ...    // Kernel 参数
);
```

> **注意**: `hipLaunchKernelGGL` 和 `<<<>>>` 都是**异步**的，需显式同步。

### 约束

* 任意维度 `gridDim * blockDim` 必须 **< 2³²**
* Block 内线程总数受设备 `maxThreadsPerBlock` 限制（通常 1024）

## 编译模型

### 编译器: hipcc

`hipcc` 是 Perl 脚本，根据目标平台自动选择后端编译器：

| 目标 | 后端编译器 | 产物 |
|------|-----------|------|
| AMD GPU | `clang++` (AMDGPU 目标) | amdgcn-amd-amdhsa .o |
| NVIDIA GPU | `nvcc` | CUDA .o |
| Host 代码 | 系统 C++ 编译器 (g++/clang++) | x86_64 .o |

### 平台宏

```cpp
#ifdef __HIP_PLATFORM_AMD__
    // AMD 平台特定代码
#endif

#ifdef __HIP_PLATFORM_NVIDIA__
    // NVIDIA 平台特定代码
#endif
```

### 编译选项

| 选项 | 说明 |
|------|------|
| `--offload-arch=<target>` | 指定 GPU 架构目标，如 `gfx90a`, `gfx942`, `gfx1200` |
| `-O2` / `-O3` | 优化级别 |
| `-g` | 生成调试信息 |
| `--hipstdpar` | 启用 C++17 标准并行支持 |

## 内存模型

| 内存类型 | 位置 | 作用域 | 生命周期 |
|----------|------|--------|----------|
| **Global Memory** | Device DRAM | 所有线程 | 程序结束 |
| **Shared Memory** (`__shared__`) | LDS (Local Data Share) | Block 内 | Block 结束 |
| **Constant Memory** (`__constant__`) | Device DRAM (缓存) | 所有线程 | 程序结束 |
| **Local Memory** | 寄存器溢出到 DRAM | Thread | Thread 结束 |
| **Register** | SM 寄存器文件 | Thread | Thread 结束 |

### Block/Grid 分配原则

1. Block 内线程数为 Wavefront 大小的倍数（AMD 为 64 的倍数）
2. 每个 Block 至少 128-256 个线程以隐藏延迟
3. Block 数量远大于 SM(CU) 数量，保证负载均衡
4. 根据寄存器/LDS 使用量调整 Block 大小

## 来源

* [AMD ROCm HIP 官方文档](https://rocm.docs.amd.com/projects/HIP/en/latest/)
* [AMD HPCTrainingExamples](https://github.com/amd/HPCTrainingExamples)
