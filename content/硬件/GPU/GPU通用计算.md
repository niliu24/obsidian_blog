---
tags: [gpu, gpgpu, compute]
type: entity
aliases:
  - GPGPU
  - 通用 GPU 计算
  - 异构计算
---

> GPGPU（General-Purpose computing on Graphics Processing Units）利用 GPU 的大规模并行架构执行非图形计算任务。从 2006 年 CUDA 发布至今，GPGPU 已从学术探索演变为现代计算的核心支柱——驱动深度学习、科学模拟和高性能计算。

## SIMT 执行模型

GPU 采用 **SIMT（Single Instruction, Multiple Threads）** 执行模型，与 CPU 的 SIMD 有本质区别：

| 特性 | SIMD (CPU) | SIMT (GPU) |
|------|-----------|------------|
| **粒度** | 向量寄存器 (如 AVX-512: 16×FP32) | 独立线程 (Warp: 32 / Wavefront: 64) |
| **编程抽象** | 显式向量化或 auto-vectorization | 标量线程代码，硬件管理并行 |
| **分支处理** | 掩码操作 (masked) | 分支导致 Warp Divergence (串行化) |
| **灵活性** | 需手动处理数据对齐/步长 | 每个线程有独立 PC 和寄存器 |

SIMT 的工作原理：

```
指令流:                  ADD R0, R1, R2
                         │
Warp (32 threads) ───────┤
  ┌── Lane 0:  R0 = R1 + R2 (活跃)
  ├── Lane 1:  R0 = R1 + R2 (活跃)
  ├── Lane 2:  R0 = R1 + R2 (活跃)
  ├── ...
  └── Lane 31: R0 = R1 + R2 (活跃 或 屏蔽)
```

每个线程执行同一条指令，但有独立的寄存器——数据可以是完全不同的值。这种 " 单指令多数据 " 的变体是 GPU 高吞吐的核心原因。

## Warp / Wavefront 调度

**Warp**（NVIDIA）和 **Wavefront**（AMD）是 GPU 硬件调度的基本单位：

| 特性 | NVIDIA Warp | AMD Wavefront |
|------|-------------|---------------|
| **线程数** | 32 | 64 |
| **调度方式** | Warp Scheduler 每周期选一个就绪 Warp 发射 | Wave Scheduler 类似 |
| **Warp Divergence** | 同一 Warp 内分支 → 路径被串行化 | 同一 Wavefront 内分支 → 两个 32 线程子组 |
| **零开销切换** | 切换一个 Warp 仅需一个时钟周期 | 类似 |
| **Occupancy 定义** | 活跃 Warp / 最大 Warp (每 SM) | 活跃 Wavefront / 最大 Wavefront (每 CU) |

### Divergence 影响

```cpp
__global__ void divergent_kernel(float* data, int N) {
    int idx = blockIdx.x * blockDim.x + threadIdx.x;
    if (idx < N) {
        // 依赖 idx 的分支: 只有当分歧对齐到 Warp 边界时才无害
        if (idx % 2 == 0) {
            data[idx] = sqrt(data[idx]);   // 路径 A (一半线程)
        } else {
            data[idx] = exp(data[idx]);    // 路径 B (另一半线程)
        }
        // Warp 内的 32 个线程: lane 0,2,4,... 走 A, lane 1,3,5,... 走 B
        // → 两条路径串行执行: 先 A (lane 1,3,5... 被屏蔽), 再 B (lane 0,2,4... 被屏蔽)
    }
}
```

**缓解 Divergence 策略**:

1. 保持 Warp 内线程走相同分支（数据重组）
2. 用谓词化替代分支（使用三元运算符、min/max、整数算术）
3. 使用 `__ballot_sync` / `__any_sync` 等 Warp-Level 原语

## Tensor Core —— 矩阵运算专有硬件

Tensor Core 是 GPU 上最重要的计算加速单元，专门执行 **D = A × B + C** 矩阵乘加运算：

| 代数 | 精度 | Tensor Core 吞吐 (H100 SXM) | 典型用途 |
|------|------|---------------------------|----------|
| **1st Gen** (Volta, 2017) | FP16 | 125 TFLOPS | 早期深度学习训练 |
| **2nd Gen** (Turing, 2018) | FP16/INT8 | 500 TOPS | 推理加速 |
| **3rd Gen** (Ampere, 2020) | FP16/BF16/INT8/INT4 | 624 TFLOPS (稀疏) | 混合精度训练 |
| **4th Gen** (Hopper, 2022) | FP16/BF16/FP8/INT8 | 2000 TFLOPS | Transformer 训练推理 |
| **5th Gen** (Blackwell, 2024) | FP16/BF16/FP8/FP6/FP4/INT8 | 9000 TOPS (FP4) | LLM 训练推理 |

### Tensor Core 工作原理

Tensor Core 在一个时钟周期内完成 **4×4 矩阵块**的乘加运算（实际上 Warp-Level MMA 操作 16×16×16 矩阵块）：

```
Warp 内 32 线程协作:
每个线程持有 A 矩阵的一个片段(2 个元素)和 B 矩阵的一个片段(2 个元素)
→ Tensor Core 计算 A×B+C → 输出 D 矩阵
```

代码示例 (CUDA Warp Matrix Multiply-Accumulate):

```cpp
#include <mma.h>
using namespace nvcuda;

// 16×16×16 半精度矩阵乘法 (Warp 级)
wmma::fragment<wmma::matrix_a, 16, 16, 16, half, wmma::row_major> a_frag;
wmma::fragment<wmma::matrix_b, 16, 16, 16, half, wmma::col_major> b_frag;
wmma::fragment<wmma::accumulator, 16, 16, 16, float> c_frag;
wmma::fragment<wmma::accumulator, 16, 16, 16, float> d_frag;

// 初始化 c_frag = 0
wmma::fill_fragment(c_frag, 0.0f);

// 迭代加载 A/B 矩阵块，累加计算
for (int k = 0; k < K; k += 16) {
    wmma::load_matrix_sync(a_frag, A + k * 16, K);
    wmma::load_matrix_sync(b_frag, B + k * 16, K);
    wmma::mma_sync(d_frag, a_frag, b_frag, c_frag);
    c_frag = d_frag;
}

// 写回结果
wmma::store_matrix_sync(C, c_frag, K, wmma::mem_row_major);
```

## 内存带宽瓶颈 —— "Memory Wall"

GPU 计算面临的核心制约是**内存带宽**，而非计算吞吐。H100 的 FP8 Tensor Core 算力为 4000 TFLOPS = 4 × 10¹⁵ ops/s，而 HBM3 带宽仅 3.35 TB/s。这意味着：

* 如果每次计算需要读取 1 个 FP8 元素 (1 byte)：需要 4 × 10¹⁵ bytes/s 的带宽 → **实际带宽仅满足需求的 0.08%**
* 因此 GPU 性能高度依赖**计算密度**（计算量/内存访问量），如矩阵乘法中重用元素

### 典型计算的 Arithmetic Intensity (计算密度)

| 操作 | Arithmetic Intensity (FLOP/Byte) | 瓶颈类型 |
|------|----------------------------------|----------|
| 向量加法 | ~0.5 (8B FP64: 1 op / 16B load+store) | **Memory Bound** |
| 矩阵乘法 (GEMM, M=N=K=4096) | ~682 (2×4096³ ops / 3×4096²×4B) | **Compute Bound** |
| 卷积 (ResNet-50) | ~100-500 | Compute Bound (大 Batch) |
| Softmax (Attention) | ~2-10 | Memory Bound |
| GELU / ReLU | ~1-2 | Memory Bound |
| FFT | ~10-50 | Varies |

### Roofline 模型

Roofline 模型是分析 GPU Kernel 性能瓶颈的标准工具：

```
Performance              ▲
(TFLOPS)                 │      ┌──── 算力上限 (Peak TFLOPS)
                         │     ╱
                         │    ╱  Compute Bound
                         │   ╱    (受限于算力)
                         │  ╱
                         │ ╱
                         │╱  Memory Bound
                         ╱    (受限于带宽)
                        ╱
                       ╱
                      ┌──────────────────────────► Arithmetic Intensity (FLOP/Byte)
                       Ridge Point = Peak TFLOPS / Bandwidth
```

* **Ridge Point**: 算力上限 / 带宽 = 临界计算密度。A 的密度高于 Ridge Point → Compute Bound；低于 → Memory Bound
* **优化方向**:
  * Memory Bound Kernel: 优化数据复用、合并访问、使用 Shared Memory (tiling)
  * Compute Bound Kernel: 提高 Occupancy、减少指令数、使用 Tensor Core

## GPGPU 典型应用场景

| 领域 | 具体应用 | 关键硬件需求 | 瓶颈类型 |
|------|----------|-------------|----------|
| **深度学习训练** | Transformer (LLM)、CNN、GAN | Tensor Core、大 HBM、NVLink | Compute Bound (矩阵乘法) |
| **深度学习推理** | LLM 部署 (vLLM)、图像分类 | Tensor Core、低延迟 NPU 替代 | Memory Bound (KV Cache) |
| **科学模拟** | 分子动力学 (NAMD, GROMACS)、CFD (OpenFOAM)、气象模拟 | FP64 算力、大内存 | Varies |
| **视频编解码** | NVENC (H.264/H.265/AV1)、NVDEC | 专用视频编码/解码单元 | 固定功能，延迟关键 |
| **金融计算** | 蒙特卡洛模拟、风险分析 | FP64 算力、ECC 内存 | Compute Bound |
| **数据库加速** | GPU-DB (HeavyDB, BlazingSQL) | 高带宽、大容量显存 | Memory Bound |
| **渲染** | 实时光栅化、路径追踪 | RT Core、Texture Unit | Varies |
| **密码学 / 挖矿** | PoW 哈希、密码破解 | INT32 算力、Warp 效率 | Compute Bound |

### 深度学习 vs 科学计算对比

| 特性 | AI 训练 | 科学模拟 |
|------|---------|---------|
| **精度要求** | FP16/BF16/FP8 即可 | 通常需要 FP64 |
| **算力需求** | Tensor Core (矩阵乘法) | CUDA Core (FP64 算力) |
| **内存需求** | 几十 GB (模型参数 + Optimizer State) | 百 GB 级 (网格/粒子数据) |
| **典型 GPU** | H100, B200, A100 | H100, MI300X (FP64 强化) |
| **核函数特征** | 规律 GEMM + Attention | 不规则访存 + 随机原子操作 |

## GPGPU vs 专用 AI 加速器

| 特性 | GPU (H100/B200) | TPU (Google v5p) | NPU (Ascend 910B) | 专用 ASIC |
|------|----------------|------------------|-------------------|-----------|
| **架构** | 通用可编程 (CUDA) | 脉动阵列 + 标量核 | Da Vinci Core (Cube + Vector + Scalar) | 完全专用 |
| **编程模型** | CUDA / HIP / OpenCL | XLA / TensorFlow | CANN / MindSpore | 框架绑定 |
| **精度支持** | FP64~FP4 | BF16/INT8 | FP32/FP16/INT8 | 固定精度 |
| **灵活性** | 高 (通用计算 + 图形 + AI) | 中 (限于 ML 模型) | 中 (限于 AI 推理/训练) | 低 (单一算法) |
| **显存/内存** | 80 GB HBM3 + NVLink | 95 GB HBM2e (v4) | 64 GB HBM2e | 通常小 |
| **功耗** | 700 W | ~200 W (per chip) | 310 W | 5-50 W (边缘) |
| **典型部署** | 数据中心 LLM 训练/推理 | Google 内部 TPU Pod | 华为云 | 边缘设备、手机 |

### CUDA Kernel 示例 —— 向量加法

最经典的 GPGPU 入门示例，展示从 Host 到 Device 的完整程序流程：

```cpp
#include <cuda_runtime.h>
#include <stdio.h>

// Kernel 定义: __global__ 表示该函数在 GPU 上运行，由 CPU 调用
__global__ void vectorAdd(const float* A, const float* B, float* C, int N) {
    // 线程索引计算: Block 内线程索引 + Block 偏移
    int idx = blockIdx.x * blockDim.x + threadIdx.x;

    // 边界检查: 确保线程不会越界访问
    if (idx < N) {
        C[idx] = A[idx] + B[idx];
    }
}

int main() {
    int N = 1 << 20;  // 1M 元素
    size_t bytes = N * sizeof(float);

    // 1. Host 端分配内存并初始化
    float *h_A = (float*)malloc(bytes);
    float *h_B = (float*)malloc(bytes);
    float *h_C = (float*)malloc(bytes);
    for (int i = 0; i < N; i++) {
        h_A[i] = 1.0f;
        h_B[i] = 2.0f;
    }

    // 2. Device 端分配显存
    float *d_A, *d_B, *d_C;
    cudaMalloc(&d_A, bytes);
    cudaMalloc(&d_B, bytes);
    cudaMalloc(&d_C, bytes);

    // 3. Host → Device 数据拷贝
    cudaMemcpy(d_A, h_A, bytes, cudaMemcpyHostToDevice);
    cudaMemcpy(d_B, h_B, bytes, cudaMemcpyHostToDevice);

    // 4. 启动 Kernel
    int threadsPerBlock = 256;
    int blocksPerGrid = (N + threadsPerBlock - 1) / threadsPerBlock;  // 向上取整
    vectorAdd<<<blocksPerGrid, threadsPerBlock>>>(d_A, d_B, d_C, N);

    // 5. 同步等待 GPU 完成
    cudaDeviceSynchronize();

    // 6. Device → Host 数据拷贝
    cudaMemcpy(h_C, d_C, bytes, cudaMemcpyDeviceToHost);

    // 7. 验证结果
    for (int i = 0; i < 10; i++) {
        printf("C[%d] = %f\n", i, h_C[i]);
    }

    // 8. 释放内存
    cudaFree(d_A);
    cudaFree(d_B);
    cudaFree(d_C);
    free(h_A);
    free(h_B);
    free(h_C);

    return 0;
}
```

编译: `nvcc vector_add.cu -o vector_add`

## 来源

* [NVIDIA CUDA C++ Programming Guide — SIMT Architecture](https://docs.nvidia.com/cuda/cuda-c-programming-guide/index.html#simt-architecture)
* [NVIDIA Tensor Core Programming](https://developer.nvidia.com/blog/programming-tensor-cores-cuda-9/)
* [Roofline Model — CS 267, UC Berkeley](https://people.eecs.berkeley.edu/~kubitron/cs267/handouts/roofline.pdf)
* [AMD ROCm Architecture Overview](https://rocm.docs.amd.com/en/latest/)
* [Google TPU v4 Architecture](https://arxiv.org/abs/2304.01433)
* [华为 Ascend 910B 产品概述](https://www.hiascend.com/products/server/atlas-900)
