---
tags:
  - opencl
  - gpu
  - parallel-computing
  - ndrange
type: note
---

NDRange（N-Dimensional Range）是 OpenCL 执行模型的核心概念。它定义了 kernel 并行执行的多维索引空间，决定了每个 work-item 处理的数据位置。

**NDRange 的维度 = 每个 work-item 索引坐标的维度**。1D 的 NDRange 中每个 work-item 只有一个标量 ID，2D 时每个 work-item 拥有 `(x, y)` 坐标，3D 时拥有 `(x, y, z)` 坐标。work-item 总数由各维 global size 的乘积决定——全局 1024 个 work-item，可以组织为 1D `(1024)`、2D `(32, 32)` 或 3D `(16, 8, 8)`。但不同维度选择不仅仅是索引形式的变化，还决定了 work-group 的形状（1D/2D/3D 分组）、dim0 连续访存方向以及 barrier 同步的范围——选错维度映射会导致访存发散和并行效率下降。

## 概念层次

```
NDRange (全局索引空间)
│
├── Work-Group 0
│   ├── Work-Item (0,0)
│   ├── Work-Item (0,1)
│   └── ...
│
├── Work-Group 1
│   ├── Work-Item (1,0)
│   └── ...
│
└── Work-Group N-1
    └── ...
```

每个 work-item 执行**完全相同的 kernel 代码**，通过索引区分要处理的数据。这是 SIMT（Single Instruction Multiple Thread）/SPMD 模型的体现。

## 核心概念

| 概念 | 说明 | 获取方式 |
|------|------|---------|
| Global ID | work-item 在全局索引空间中的坐标 | `get_global_id(dim)` |
| Local ID | work-item 在所在 work-group 内的坐标 | `get_local_id(dim)` |
| Group ID | work-group 的编号 | `get_group_id(dim)` |
| Global Size | 每维 work-item 总数 | `get_global_size(dim)` |
| Local Size | 每维每 work-group 的 work-item 数 | `get_local_size(dim)` |
| Num Groups | 每维 work-group 总数 | `get_num_groups(dim)` |

关系：`global_id[dim] = group_id[dim] * local_size[dim] + local_id[dim]`

## 1D NDRange

```c
// Host 端
size_t global_size = 1024;
size_t local_size  = 256;
clEnqueueNDRangeKernel(queue, kernel, 1, NULL, &global_size, &local_size, 0, NULL, NULL);

// Kernel 端
__kernel void process_1d(__global float *data) {
    int gid = get_global_id(0);   // 0 ~ 1023, 唯一标识
    int lid = get_local_id(0);    // 0 ~ 255, 组内相对位置
    int gid2 = get_group_id(0);   // 0 ~ 3, 第几个 work-group
    // global_size/0 = 1024, local_size/0 = 256, num_groups/0 = 4
    data[gid] = lid * 2.0f;
}
```

```
Global Size = 1024, Local Size = 256, 4 Work-Groups

┌────────────────────┬────────────────────┬────────────────────┬────────────────────┐
│     Group 0        │     Group 1        │     Group 2        │     Group 3        │
│ global_id: 0..255  │ global_id: 256..511│ global_id: 512..767│ global_id: 768..1023│
│ local_id:  0..255  │ local_id:  0..255  │ local_id:  0..255  │ local_id:  0..255  │
└────────────────────┴────────────────────┴────────────────────┴────────────────────┘
```

## 2D NDRange

以矩阵类比：整个 NDRange 就是一张 `width × height` 的网格，**每个 work-item 对应网格中的一个元素**，其 `(col, row)` 坐标即该元素的 `(global_id(0), global_id(1))`。一个 work-group 则是网格中的一个矩形子块。

2D 常用于图像处理、矩阵运算：

```c
// Host 端
size_t global[] = {width, height};
size_t local[]  = {16, 16};
clEnqueueNDRangeKernel(queue, kernel, 2, NULL, global, local, 0, NULL, NULL);
```

```c
// Kernel 端 —— 图像卷积
__kernel void convolution_2d(__global const float *input,
                             __global float *output,
                             int width, int height) {
    int x = get_global_id(0);  // 列 (0 ~ width-1)
    int y = get_global_id(1);  // 行 (0 ~ height-1)

    int lx = get_local_id(0);
    int ly = get_local_id(1);

    if (x < width && y < height) {
        output[y * width + x] = /* 以 (x, y) 为中心的卷积计算 */;
    }
}
```

2D NDRange 布局示意（Global=8×4, Local=4×2）：

```
dim0 (x/列):  0    1    2    3  │  4    5    6    7
dim1 (y/行):
0            [0,0][0,1][0,2][0,3]│[0,4][0,5][0,6][0,7]    Group (0,0): dim1=0..1, dim0=0..3
1            [1,0][1,1][1,2][1,3]│[1,4][1,5][1,6][1,7]    Group (0,1): dim1=0..1, dim0=4..7
             ────────────────────┼────────────────────
2            [2,0][2,1][2,2][2,3]│[2,4][2,5][2,6][2,7]    Group (1,0): dim1=2..3, dim0=0..3
3            [3,0][3,1][3,2][3,3]│[3,4][3,5][3,6][3,7]    Group (1,1): dim1=2..3, dim0=4..7
```

每个格子内的坐标 `[y, x]` 为该 work-item 的 `(global_id(1), global_id(0))`。总共 4 个 work-group，每个 work-group 8 个 work-item。

## 3D NDRange

3D 用于体渲染、物理模拟、3D 卷积：

```c
size_t global[] = {256, 256, 64};
size_t local[]  = {8, 8, 4};
clEnqueueNDRangeKernel(queue, kernel, 3, NULL, global, local, 0, NULL, NULL);

// Kernel 端
__kernel void process_3d(__global float *volume) {
    int x = get_global_id(0);  // 变化最快
    int y = get_global_id(1);
    int z = get_global_id(2);  // 变化最慢
}
```

内存布局上 dim0 连续（x 维 stride=1），适合合并访问。

## NDRange 与硬件映射

```
Software                           Hardware
─────────                          ────────
NDRange                            整个 GPU
└── Work-Group                     └── Compute Unit (SM/CU)
    └── Work-Item (SIMD 宽度个)        └── PE 组 (warp/wavefront)
```

* 一个 **work-group** 被调度到一个 **Compute Unit** 上执行
* work-group 内的 work-item 以 **warp/wavefront**（NVIDIA 32 线程/AMD 64 线程）为单位执行
* 同一 work-group 内的 work-item 共享 Local Memory 和 barrier
* **不同 work-group 之间无执行顺序保证**，不能跨 work-group 同步
* 这也意味着 work-group 大小设置为 warp/wavefront 大小的整数倍时硬件利用率最高

## 多维度的内存含义

```c
// 1D: 线性处理
__kernel void vec_add(__global const float *a,
                      __global const float *b,
                      __global float *c) {
    int gid = get_global_id(0);         // 全局线性 ID
    c[gid] = a[gid] + b[gid];
}

// 2D: dim0 连续 → 合并访存
__kernel void matrix_mul(__global const float *A,
                         __global const float *B,
                         __global float *C,
                         int width) {
    int row = get_global_id(1);         // 行
    int col = get_global_id(0);         // 列 (连续访问)

    float sum = 0;
    for (int k = 0; k < width; k++)
        sum += A[row * width + k] * B[k * width + col];
    C[row * width + col] = sum;
}
```

dim0 变化最快，同一 warp 内的 work-item 访问连续地址，合并为一次内存事务。

## global_work_offset

非零偏移量支持从 NDRange 中间开始执行：

```c
size_t global[]  = {1024};
size_t local[]   = {256};
size_t offset[]  = {512};  // 从 global_id = 512 开始
clEnqueueNDRangeKernel(queue, kernel, 1, offset, global, local, 0, NULL, NULL);
// kernel 中 get_global_id(0) 返回 512 ~ 1535
```

常用于分块处理超大数组，或在不同设备上分别执行不同区域。

## local_work_size 为 NULL

```c
clEnqueueNDRangeKernel(queue, kernel, 1, NULL, &global, NULL, 0, NULL, NULL);
```

此时 OpenCL 运行时自动选择 local size。自动选择的 size 取决于驱动实现和设备特性，通常小于最大 work-group size 且能整除 global size。适合快速原型，但手动调优往往能获得更优性能——自动选择不了解 work-group 内的 local memory 用量，可能导致 occupancy 不高。

## global size 不能被 local size 整除

OpenCL 要求 `global_size[dim]` 必须能被 `local_size[dim]` 整除。不整除时需向上取整 + kernel 内边界检查：

```c
// 任意大小 N 的向上取整
size_t local  = 256;
size_t global = ((N + local - 1) / local) * local;
clEnqueueNDRangeKernel(queue, kernel, 1, NULL, &global, &local, 0, NULL, NULL);

// Kernel 内
__kernel void safe_process(__global float *data, int N) {
    int gid = get_global_id(0);
    if (gid < N) {  // 边界保护，防止越界
        data[gid] *= 2.0f;
    }
}
```

## Work-Group Size 约束

```c
// 查询设备限制
cl_uint max_wg_size;
size_t max_wg[3];
clGetDeviceInfo(device, CL_DEVICE_MAX_WORK_GROUP_SIZE, sizeof(max_wg_size), &max_wg_size, NULL);
clGetDeviceInfo(device, CL_DEVICE_MAX_WORK_ITEM_SIZES, sizeof(max_wg), max_wg, NULL);

printf("最大 work-group 总大小: %u\n", max_wg_size);
printf("每维最大: (%zu, %zu, %zu)\n", max_wg[0], max_wg[1], max_wg[2]);
// 约束: local_size[0]*local_size[1]*local_size[2] <= max_wg_size
//  且  local_size[dim] <= max_wg[dim]
```

work-group size 还受 kernel 的 local memory 和私有变量（寄存器压力）限制，可用 `clGetKernelWorkGroupInfo` 查询特定 kernel 的支持值。

## 调优要点

* **合并访问**：让 dim0 的相邻 work-item 访问连续内存地址
* **warp/wavefront 对齐**：local size 设为 32（NVIDIA）或 64（AMD）的倍数
* **occupancy**：local memory 和寄存器用量决定每个 CU 上可同时驻留的 work-group 数
* **不要跨组同步**：算法设计必须接受这一约束；需全局同步时拆成多个 kernel 依次提交
* **避免分支发散**：同一 warp 内所有 work-item 执行同一分支路径，分支发散会导致串行化
