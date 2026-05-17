---
tags: [gpu, architecture, rendering]
type: entity
aliases:
  - GPU 硬件架构
  - 图形渲染管线
  - GPU microarchitecture
---

> GPU 硬件以 **SM（Streaming Multiprocessor，NVIDIA）** / **CU（Compute Unit，AMD）** 为基本构建块，通过海量线程并行执行图形渲染或通用计算任务。图形渲染管线则定义了从 3D 模型到 2D 像素的完整流水线。

## SM/CU 微架构

SM（NVIDIA）和 CU（AMD）是 GPU 的核心计算单元，每个 SM/CU 内部包含多类执行单元和片上存储：

```
SM (Streaming Multiprocessor) — NVIDIA Ada Lovelace (RTX 40 系列) 为例:
┌──────────────────────────────────────────────────┐
│  Register File (65536 × 32-bit = 256 KB)         │
│  L1 Data Cache / Shared Memory (128 KB, 可配置)  │
│  ┌────────────┬────────────┬────────────┬──────┐ │
│  │ Warp       │ Warp       │ Warp       │ Warp │ │
│  │ Scheduler  │ Scheduler  │ Scheduler  │...   │ │
│  ├─────┬──────┼─────┬──────┼─────┬──────┼──────┤ │
│  │CUDA │CUDA  │CUDA │CUDA  │CUDA │CUDA  │  SFU │ │
│  │Core │Core  │Core │Core  │Core │Core  │      │ │
│  │ x16 │ x16  │ x16 │ x16  │ x16 │ x16  │      │ │
│  ├─────┴──────┴─────┴──────┴─────┴──────┴──────┤ │
│  │   Tensor Core (4 个) + RT Core (1 个)        │ │
│  │   Texture Units (16 个)                      │ │
│  └──────────────────────────────────────────────┘ │
└──────────────────────────────────────────────────┘
```

### NVIDIA SM 内部组件

| 组件 | 功能 |
|------|------|
| **CUDA Core** (ALU) | 整数/浮点算术逻辑单元，执行基本运算 (FP32/INT32) |
| **Tensor Core** | 矩阵乘加加速器，支持 FP64/FP32/FP16/BF16/FP8/INT8/FP4 |
| **RT Core** | 光线追踪加速硬件 (BVH 遍历、三角形求交) |
| **SFU** (Special Function Unit) | 特殊函数 (sin, cos, exp, log, sqrt, rcp) |
| **Texture Unit** | 纹理采样、过滤、映射 (常用于 Graphics/Image Processing) |
| **Warp Scheduler** | Warp 分发调度，零开销上下文切换 |
| **Register File** | 每个 Thread 私有寄存器 |
| **L1/Shared Memory** | L1 Cache 与 Shared Memory 共享片上存储，可配置比例 |

### AMD CU (Compute Unit) 对比

| NVIDIA SM | AMD CU (RDNA 3) |
|-----------|-----------------|
| CUDA Core | Stream Processor (SIMD Unit) |
| Tensor Core | Matrix Core (WMMA 指令) |
| RT Core | Ray Accelerator |
| Warp (32 threads) | Wavefront (64 threads) |
| L1/Shared Memory | L0/L1 Cache + Shared Memory (LDS) |
| Warp Scheduler | Wave Scheduler |

## 内存层次

GPU 的内存层次遵循**带宽递增、容量递减**的原则：

```
Global Memory / HBM        HBM2e: 2 TB/s, 80 GB   ← 慢、大
    ↓
L2 Cache                   ~40-80 MB               ← 片上
    ↓
L1 Cache / Shared Memory   ~128-256 KB per SM      ← 片上
    ↓
Registers                  ~64K × 32b per SM       ← 最快、最小
```

| 内存类型 | 位置 | 作用域 | 速度 | 容量 (H100 SXM) |
|----------|------|--------|------|-----------------|
| **Global Memory** (HBM3) | 片外 HBM 堆叠 | 全 GPU + Host | ~3.35 TB/s | 80 GB |
| **L2 Cache** | 片上 (GPC 间共享) | 全 GPU | ~10 TB/s | 50 MB |
| **L1 Cache / Shared Memory** | 片上 (SM 内) | SM 内 | ~30+ TB/s | 128 KB 每 SM |
| **Register File** | 片上 (SM 内) | Thread | 最快 | 256 KB 每 SM |
| **Constant Memory** | 片外 (有专用缓存) | Grid | 快 (命中) | 64 KB |

### HBM vs GDDR

| 特性 | HBM (High Bandwidth Memory) | GDDR (Graphics DDR) |
|------|-----------------------------|---------------------|
| 使用场景 | 数据中心 GPU (H100, A100, MI300X) | 消费级显卡 (RTX 4090, RX 7900 XTX) |
| 位宽 | 4096-bit (通过硅中介层堆叠) | 256/384-bit (PCB 走线) |
| 带宽 | 2-3.5 TB/s | 0.5-1 TB/s |
| 功耗 | 更低 (短距离、低电压) | 更高 (长 PCB 走线) |
| 容量 | 最大 144 GB (H100 80GB / MI300X 192GB) | 最大 24 GB (RTX 4090) / 48 GB (RTX 6000 Ada) |

## 关键 GPU 规格

| 规格 | 含义 | 示例 (H100 SXM) |
|------|------|-----------------|
| **TFLOPS (FP32)** | 单精度浮点算力 | 60 TFLOPS |
| **TFLOPS (FP16/BF16)** | 半精度浮点 (Tensor Core 加速) | 2000 TFLOPS (稀疏) |
| **TFLOPS (FP8)** | 8 位浮点 (Hopper Transformer Engine) | 4000 TFLOPS |
| **Memory Bandwidth** | 显存带宽 | 3.35 TB/s |
| **VRAM Capacity** | 显存容量 | 80 GB HBM3 |
| **SM Count** | SM 数量 | 132 SM |
| **CUDA Cores** | FP32 核心总数 | 16896 |
| **Tensor Cores** | 矩阵加速器数量 | 528 |
| **ROPs** | 光栅化输出单元 | 162 |
| **TMUs** | 纹理映射单元 | 528 |
| **TDP** | 热设计功耗 | 700 W |

## 图形渲染管线

现代 GPU 图形管线将 3D 场景渲染为 2D 图像：

```
Vertex Data (模型顶点)
    ↓
[Vertex Shader]           ← 顶点变换、蒙皮、顶点动画
    ↓
[Tessellation] (Hull+Domain)   ← 细分曲面，增加几何细节
    ↓
[Geometry Shader]         ← 点/线/三角形 → 新图元 (可选)
    ↓
[Mesh Shader] (现代替代)  ← 替代 VS + Tess + GS，基于计算着色器
    ↓
[Rasterization]           ← 顶点 → 像素片段 (固定功能)
    ↓
[Fragment / Pixel Shader] ← 逐像素着色、贴图、光照
    ↓
[Output Merger] (ROPs)    ← 深度测试、模板测试、混合、写入 Framebuffer
    ↓
Frame Buffer → Display
```

### 各阶段说明

| 阶段 | 可编程? | 说明 |
|------|---------|------|
| **Vertex Shader** | 可编程 | 对每个顶点执行变换 (Model→World→View→Projection)、顶点着色、蒙皮权重等 |
| **Tessellation** | 可编程 (Hull & Domain Shader) | 将粗糙多边形网格细分为精细曲面 (曲面细分因子控制 LOD) |
| **Geometry Shader** | 可编程 (可选) | 输入单个图元，输出零到多个新图元 (常用于粒子系统、草地生成) |
| **Mesh Shader** | 可编程 (现代) | 替代 VS+Tess+GS 三个阶段，通过计算着色器直接生成图元 (NVIDIA Turing+, AMD RDNA 3+) |
| **Rasterization** | 固定功能 | 将三角形转换为像素片段 (覆盖测试、深度插值、背面剔除) |
| **Fragment Shader** | 可编程 | 最繁重阶段，逐像素计算颜色、纹理采样、光照 |
| **Output Merger** (ROPs) | 固定功能 | 深度/模板测试、Alpha 混合、颜色写入 Framebuffer |

### GLSL 片段着色器示例

```glsl
#version 460 core

// 输入: 顶点着色器传递的插值数据
in vec2 UV;
in vec3 WorldNormal;
in vec3 WorldPosition;

// 输出: 片段颜色
out vec4 FragColor;

// Uniforms
uniform sampler2D diffuseTexture;   // 漫反射纹理
uniform vec3 lightDirection;        // 光源方向 (世界空间)
uniform vec3 cameraPosition;        // 相机位置
uniform vec3 lightColor;            // 光源颜色

void main() {
    vec3 N = normalize(WorldNormal);
    vec3 L = normalize(-lightDirection);
    vec3 V = normalize(cameraPosition - WorldPosition);
    vec3 H = normalize(L + V);

    // 纹理采样
    vec3 baseColor = texture(diffuseTexture, UV).rgb;

    // 漫反射 (Lambert)
    float NoL = max(dot(N, L), 0.0);
    vec3 diffuse = baseColor * NoL;

    // 镜面高光 (Blinn-Phong)
    float NoH = max(dot(N, H), 0.0);
    vec3 specular = lightColor * pow(NoH, 32.0);

    // 环境光
    vec3 ambient = baseColor * 0.05;

    FragColor = vec4(ambient + diffuse + specular, 1.0);
}
```

### HLSL 计算着色器 (Mesh Shader 风格)

```hlsl
// Demonstrated with DirectX 12 Agility SDK - Mesh Shader Amplification

#define MS_GROUPSHARED  groupshared

// 通过放大着色器 (Amplification Shader/Dispatch Mesh) 间接启动
[ numthreads( 32, 1, 1 ) ]
void main(
    uint gtid : SV_GroupThreadID,
    uint gid  : SV_GroupID
)
{
    // 从 GPU 场景缓冲区提取顶点数据
    Vertex vert = VertexBuffer[ gid * 32 + gtid ];

    // 构建三角形输出
    MeshOutput< Triangle > output;
    output.SetVertex( 0, vert.Position, vert.Normal, vert.UV );
    output.SetVertex( 1, ... );
    output.SetVertex( 2, ... );
    output.SetIndex( 0, 0 ); output.SetIndex( 1, 1 ); output.SetIndex( 2, 2 );

    // 输出一个三角形
    output.Emit();
}
```

## 现代 GPU 特性

### 光线追踪核心 (RT Core)

RT Core 以硬件加速 **BVH (Bounding Volume Hierarchy) 遍历**和**三角形/包围盒求交**，取代传统光栅化管线中的软件光线追踪：

| 特性 | 说明 |
|------|------|
| **BVH Traversal** | 硬件管理 BVH 树的遍历，每时钟周期可追踪多条光线 |
| **Ray-Triangle Intersection** | 硬件的三角形求交单元，比 Shader 实现快数百倍 |
| **Ray-Box Intersection** | 包围盒相交测试，加速 BVH 剪枝 |
| **Opacity Micromap** (Ada) | 高效处理透明/半透明纹理的遮罩 |
| **Displaced Micromesh** (Ada) | 用于微网格位移，以低内存开销存储极高细节几何 |
| **Ray Reconstruction** (DLSS 3.5) | AI 降噪，从有噪光线追踪输出重建高质量图像 |

| 架构 | RT Core 代数 | 光线/三角形求交速率 |
|------|-------------|-------------------|
| Turing (RTX 20) | 1st Gen | 10 Giga Rays/s |
| Ampere (RTX 30) | 2nd Gen | 40 Giga Rays/s |
| Ada Lovelace (RTX 40) | 3rd Gen | 200 Giga Rays/s |
| Blackwell (RTX 50) | 4th Gen | ~500 Giga Rays/s |

### DLSS / FSR / XeSS 超分辨率

| 技术 | 厂商 | 原理 | 硬件需求 |
|------|------|------|----------|
| **DLSS 2** | NVIDIA | 时序超采样 + AI 神经网络重建 | Tensor Core (Turing+) |
| **DLSS 3** (Frame Gen) | NVIDIA | AI 插帧 + 光流加速器 (OFA) | Ada Lovelace |
| **DLSS 3.5** (Ray Reconstruction) | NVIDIA | 降噪 + AI 重建 | Ada Lovelace |
| **FSR 2/3** | AMD | 空域 + 时序重建 (非 AI) | 无专用硬件要求 |
| **XeSS** | Intel | AI 超采样 (DP4a / XMX) | Xe Matrix eXtensions (Arc) |

### Mesh Shader

Mesh Shader 是计算着色器的变体，替代传统 VS + Tess + GS 三阶段管线：

* **Turing (RTX 20)** 引入，AMD **RDNA 3** 跟进
* 两种模式:
  * **Task Mesh Shader** — 两级: Task Shader (粗粒度剔除/生成) → Mesh Shader (细粒度顶点/图元输出)
  * **Mesh Shader Only** — 单级直接输出
* **优势**: GPU-driven 渲染、动态 LOD、可见性裁剪、程序化几何生成，减少 CPU 提交开销

## Tile-Based Rendering (TBDR) vs Immediate Mode Rendering (IMR)

### IMR — NVIDIA / AMD (桌面)

```
传统 IMR Pipeline:
发布所有三角形 → 逐三角形光栅化 → 逐像素写入 Framebuffer

优点: 实现简单，无 Tile 边界开销
缺点: 大量片外带宽消耗 (深度/颜色缓冲读写)，Early Z 有限
```

### TBDR — PowerVR / Apple GPU / Mali (移动)

```
TBDR Pipeline:
几何阶段 → 按屏幕分 Tile (如 16×16/32×32) → 每个 Tile 的片段在片上完成所有处理 → 一次性写入 Framebuffer

优点: 带宽节省巨大 (片上深度/混合)，功耗低
缺点: 需要 Deferred 处理（一个 Tile 内按深度排序后着色），Overdraw 高的场景有优势
```

| 特性 | IMR (NVIDIA/AMD 桌面) | TBDR (PowerVR/Apple/Mali) |
|------|----------------------|--------------------------|
| **Framebuffer 访问** | 频繁片外写入 (VRAM) | 片上 On-chip Memory，完成后一次性写入 |
| **带宽消耗** | 高 | 低 (带宽 = 功耗) |
| **Overdraw 处理** | 所有绘制都着色 (Early Z 缓解) | Hide Surface → 仅最终可见片段着色 |
| **功耗** | 高 (~100-450W) | 低 (~1-10W) |
| **延迟** | 逐三角形提交 | 需收集 Tile 内所有三角形 (增加延迟) |
| **代表硬件** | GeForce, Radeon (RX/RTX), Quadro | PowerVR GPU (Apple A/M 系列), Mali (ARM), Adreno (Qualcomm) |
| **典型场景** | 桌面游戏、工作站、数据中心 | 手机、平板、嵌入式、游戏主机 (Switch) |

> **混合模式**: 部分现代 GPU 采用混合策略，如 AMD 的 **Primitive Bin** (RDNA 2+) 和 NVIDIA 的 **Tile Caching**，在 IMR 基础上引入分块缓存以节省带宽。

## 来源

* [NVIDIA Ada GPU Architecture Whitepaper](https://images.nvidia.com/aem-dam/Solutions/geforce/ada/nvidia-ada-gpu-architecture.pdf)
* [AMD RDNA 3 Architecture Whitepaper](https://www.amd.com/system/files/documents/amd-rdna-3-whitepaper.pdf)
* [NVIDIA Turing Mesh Shader](https://developer.nvidia.com/blog/introduction-turing-mesh-shaders/)
* [PowerVR Hardware Architecture — Imagination Technologies](https://www.imgtec.com/hardware/powervr/)
* [GPU Gems — Tile-Based Rendering](https://developer.nvidia.com/gpugems/gpugems2/part-i-geometric-complexity/chapter-7-tile-based-rendering)
* [Microsoft DirectX Mesh Shader Specification](https://microsoft.github.io/DirectX-Specs/d3d/MeshShader.html)
