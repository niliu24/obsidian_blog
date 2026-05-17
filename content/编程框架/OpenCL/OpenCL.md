---
tags: [opencl, gpu, parallel-computing, heterogeneous]
type: index
---

> OpenCL（Open Computing Language）是 Khronos Group 维护的异构并行计算开放标准，支持 CPU、GPU、FPGA、DSP 等多种设备，是唯一跨厂商、跨硬件类型的并行编程框架。

## 核心抽象

```
Host (CPU)
  │
  ├── Platform (厂商实现: NVIDIA/AMD/Intel)
  │     ├── Device 0 (GPU)  →  CU → PE
  │     ├── Device 1 (FPGA)
  │     └── Device N (DSP)
  │
  ├── Context (执行环境，绑定若干 Device)
  ├── Command Queue (向 Device 提交命令)
  ├── Program + Kernel (编译后的设备端代码)
  └── Memory Objects (Buffer / Image / Pipe)
```

OpenCL 由两部分组成：**OpenCL C**（C99 变体，设备端 kernel 语言）和 **Host API**（C/C++ 接口）。执行模型基于 NDRange 索引空间，work-item 按 work-group 分组并行执行。

## 与 CUDA/HIP 对比

| 特性 | OpenCL | CUDA | HIP |
|------|--------|------|-----|
| 厂商锁定 | 无（开放标准） | NVIDIA 独占 | AMD 为主 |
| 支持设备 | GPU/CPU/FPGA/DSP | NVIDIA GPU | AMD GPU + NVIDIA GPU |
| Kernel 语言 | OpenCL C (C99) | CUDA C++ | HIP C++ |
| 编译器 | 各厂商自有 | nvcc | hipcc |
| 性能 | 取决于实现 | 最优 (NVIDIA) | 接近 CUDA |
| 生态 | 广泛但碎片化 | 最成熟 | 快速成长 |

## 文档导航

* [编程框架](编程框架.md) — 平台模型、执行模型、内存模型、同步机制、对象模型
* [基本接口用法](基本接口用法.md) — 平台/设备发现、Context/CommandQueue 创建、Program 编译、Kernel 启动
* [NDRange 详解](NDRange详解.md) — 全局索引空间、work-item/work-group 组织方式、维度选择与性能影响
* [数据传输](数据传输.md) — Read/Write 显式传输、Map/Unmap 零拷贝映射、异步传输与事件同步
* [内存对象详解](内存对象详解.md) — Buffer（线性内存）、Image（多维纹理）、Pipe（生产者 - 消费者队列）、SVM 共享虚拟内存

## 来源

* [Khronos OpenCL Specification](https://www.khronos.org/opencl/)
* [OpenCL Programming Guide](https://www.oreilly.com/library/view/opencl-programming-guide/9780132488020/)
