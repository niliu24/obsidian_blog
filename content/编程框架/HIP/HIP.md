---
tags: [hip, gpu, parallel-computing, amd, rocm]
type: index
aliases:
  - HIP 异构计算
---

> HIP（Heterogeneous-Compute Interface for Portability）是 AMD 的异构计算可移植性接口，让开发者用同一套 C++ 代码同时支持 AMD GPU（ROCm 栈）和 NVIDIA GPU（CUDA 栈）。

## 核心定位

HIP 是 CUDA 的**语法级兼容层**，非重新实现。API 命名规则：`cudaXxx` → `hipXxx`，参数和语义基本一致。AMD 提供 `hipify` 工具自动转换 CUDA 代码。

## 编译路径

```
同一份 .hip 源码
    ├── ROCm/HIP-Clang → AMD GPU (GCN/CDNA/RDNA)
    └── NVCC → NVIDIA GPU (CUDA)
```

| 平台 | 编译器 | 运行时 |
|------|--------|--------|
| AMD GPU | `hipcc` (基于 Clang) | ROCm |
| NVIDIA GPU | `nvcc` (HIP 头文件映射到 CUDA) | CUDA |

## 文档导航

* [HIP 编程框架](HIP-编程框架.md) — 线程层次、Kernel 语言、编译模型、程序结构
* [HIP API 参考](HIP-API参考.md) — 内存管理、Stream/Event、设备管理、错误处理
