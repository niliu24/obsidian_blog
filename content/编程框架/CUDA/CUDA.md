---
tags: [cuda, gpu, parallel-computing, nvidia]
type: index
aliases:
  - CUDA 异构计算
---

> CUDA（Compute Unified Device Architecture）是 NVIDIA 的并行计算平台和编程模型，通过 C++ 扩展在 GPU 上调度海量线程执行并行任务。

## 核心概念

CUDA 采用 **Host（CPU）+ Device（GPU）** 异构架构，Host 管理执行流程和内存分配，Device 运行大规模并行 Kernel。代码通过 `nvcc` 编译器处理，一套源码同时生成 Host 端 C++ 代码和 Device 端 PTX/SASS。

## 文档导航

* [CUDA 编程框架](CUDA-编程框架.md) — 线程层次 (Grid/Block/Warp)、Kernel 语言、SM 硬件映射、编译模型、内存层次
* [CUDA API 参考](CUDA-API参考.md) — 内存管理、Stream/Event、设备管理、错误处理
