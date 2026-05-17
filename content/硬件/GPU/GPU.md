---
tags: [gpu, hardware]
type: index
aliases:
  - GPU 架构概述
  - GPU vs CPU 架构差异
---

> GPU（Graphics Processing Unit）最初为图形渲染设计，现已演变为通用并行计算加速器。其核心设计哲学是**吞吐优先**——用大量简单核心实现大规模并行，与 CPU 的**延迟优先**设计形成鲜明对比。

## CPU vs GPU 架构对比

| 维度 | CPU | GPU |
|------|-----|-----|
| **设计目标** | 低延迟，单线程性能 | 高吞吐，大规模并行 |
| **核心数量** | 少（4-32 核） | 极多（数千核） |
| **核心复杂度** | 复杂（大缓存、分支预测、乱序执行） | 简单（小缓存、无复杂控制逻辑） |
| **缓存层次** | 大 L1/L2/L3（MB 级） | 小 L1/L2（KB-MB），大全局显存（GB 级 HBM/GDDR） |
| **控制单元** | 复杂（分支预测、超标量、乱序、预取） | 简单（SIMT 调度为主） |
| **内存带宽** | 几十 GB/s（DDR5/DDR4） | 几百 GB/s 至 TB/s 级（HBM2/HBM3/GDDR6X） |
| **适用场景** | 串行任务、操作系统、逻辑控制 | 数据并行、矩阵运算、渲染 |

```
CPU:                    GPU:
┌──────┐                ┌────────────────────────┐
│ Core │ ← 大控制单元   │ SM 0  │ SM 1  │ SM 2  │ ← 简单控制
│ Core │ ← 大 L2 缓存   │ SM 3  │ SM 4  │ SM 5  │ ← 小缓存
│ Core │ ← 强分支预测   │ SM 6  │ SM 7  │ SM 8  │ ← 多个 SM
│ Core │                │ Warp Sched │ Mem Ctrl │
└──────┘                └───────┬────────────────┘
                               │
                        HBM2e / GDDR6X
                        (TB/s 级带宽)
```

## 从图形处理器到通用计算

GPU 架构经历了从**固定功能管线**到**统一着色器**再到**通用计算**的演进：

| 阶段 | 时期 | 代表架构 | 关键特性 |
|------|------|----------|----------|
| **固定功能管线** | 1990s-2000 | NVIDIA GeForce 256 | 硬件 T&L（Transform & Lighting），不可编程 |
| **可编程着色器** | 2001-2006 | NVIDIA GeForce 3（VS+PS），Radeon 9700 | 顶点/像素着色器，可编程但分离 |
| **统一着色器** | 2006 | NVIDIA Tesla（G80） | 统一计算单元，CUDA 诞生，GPGPU 起点 |
| **通用计算成熟** | 2010s | Fermi→Kepler→Maxwell→Pascal | 双精度、ECC、统一内存、NVLink |
| **AI 加速** | 2017- | Volta→Turing→Ampere→Hopper→Blackwell | Tensor Core、Transformer Engine、NVLink Switch |
| **芯片级集成** | 2020s | Grace Hopper、DGX、GH200 | CPU+GPU 统一封装、大共享内存池 |

### 关键转折点

* **2006 — G80 / CUDA**: NVIDIA 发布 Tesla 架构（G80），统一顶点和像素着色器为流处理器，宣布 CUDA 编程模型，GPU 首次对通用编程开放。
* **2010 — Fermi**: 完整 ECC 支持、L1/L2 缓存层次、真正意义上的 GPGPU 架构。
* **2017 — Volta**: 引入 **Tensor Core**，专门加速矩阵乘加运算，开启 GPU 在深度学习训练/推理中的统治地位。
* **2022 — Hopper**: 引入 Transformer Engine 和 DPX 指令，为 LLM 训练优化。
* **2024 — Blackwell**: 支持 FP4/FP6 精度、NVLink 5.0、第二代 Transformer Engine。

## 文档导航

* [GPU 架构与渲染管线](GPU架构与渲染管线.md) — SM/CU 微架构、内存层次、图形渲染管线、TBDR vs IMR
* [GPU 通用计算](GPU通用计算.md) — GPGPU 概念、SIMT 执行模型、Warp/Wavefront、Tensor Core、Roofline 模型
* [GPU-CPU 数据传输](GPU-CPU数据传输.md) — PCIe DMA、BAR/MMIO 控制面、cudaMemcpy 原理、NVLink/CXL 高速互联

## 相关文档

* [CUDA 编程框架](../../编程框架/CUDA/CUDA.md) — NVIDIA 通用并行计算平台
* [HIP 编程框架](../../编程框架/HIP/HIP.md) — AMD 异构计算可移植性接口
* [vLLM 推理引擎](../../大模型/推理引擎/vLLM/vLLM.md) — GPU 上的 LLM 推理框架 (PageAttention + Continuous Batching)
* [LLM 量化](../../大模型/模型压缩/LLM-量化.md) — 量化技术作用于低精度硬件单元 (Tensor Core)

## 来源

* [NVIDIA CUDA C++ Programming Guide — Hardware Implementation](https://docs.nvidia.com/cuda/cuda-c-programming-guide/index.html#hardware-implementation)
* [NVIDIA GPU Architecture — Fermi through Blackwell (WikiChip)](https://en.wikichip.org/wiki/nvidia)
* [Radeon Feature Matrix — AMD GPUOpen](https://gpuopen.com/learn/radeon-feature-matrix/)
* [GPU Performance: Latency vs Throughput](https://www.nvidia.com/en-us/on-demand/session/gtc2010-s1230/)
