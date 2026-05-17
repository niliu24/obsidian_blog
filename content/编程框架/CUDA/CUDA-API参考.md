---
tags: [cuda, gpu, api-reference, nvidia]
type: entity
---

> CUDA Runtime API 提供对 GPU 设备的完整控制。头文件 `<cuda_runtime.h>`。命名规则以 `cuda` 开头，Driver API 以 `cu` 开头，功能更底层但使用更复杂。

## 设备管理

设备由 0-based 整数 ID 标识。

| 函数 | 说明 |
|------|------|
| `cudaGetDeviceCount(int *count)` | 获取可用 GPU 数量 |
| `cudaSetDevice(int deviceId)` | 设置当前线程使用的 GPU（Thread-Local） |
| `cudaGetDevice(int *deviceId)` | 获取当前线程使用的 GPU ID |
| `cudaGetDeviceProperties(cudaDeviceProp *prop, int deviceId)` | 查询设备属性 |
| `cudaSetDeviceFlags(unsigned flags)` | 设置设备调度行为（如 `cudaDeviceScheduleBlockingSync`） |
| `cudaDeviceSynchronize()` | 阻塞 Host 直到当前设备所有 Stream 完成（重量级同步） |
| `cudaDeviceReset()` | 重置设备，清理所有分配资源 |

### cudaDeviceProp 关键字段

| 字段 | 说明 |
|------|------|
| `name[256]` | 设备名称 |
| `totalGlobalMem` | 全局内存总量 |
| `sharedMemPerBlock` | 每个 Block 的最大共享内存 |
| `maxThreadsPerBlock` | Block 最大线程数 |
| `maxThreadsDim[3]` | 每个维度最大线程数 |
| `multiProcessorCount` | SM 数量 |
| `clockRate` | 时钟频率 (kHz) |
| `warpSize` | Warp 大小（固定 32） |
| `maxSharedMemoryPerBlockOptin` | 可选配置的最大共享内存 |
| `concurrentKernels` | 是否支持多 Kernel 并发执行 |
| `computeMode` | 计算模式（default/process-exclusive 等） |
| `unifiedAddressing` | 是否支持统一寻址 |

### 使用示例

```cpp
int deviceCount;
cudaGetDeviceCount(&deviceCount);
cudaSetDevice(0);

cudaDeviceProp props;
cudaGetDeviceProperties(&props, 0);
printf("GPU: %s, 显存: %zu MB, SM: %d, Warp: %d\n",
       props.name,
       props.totalGlobalMem / (1024 * 1024),
       props.multiProcessorCount,
       props.warpSize);
```

## 内存管理

### 基本 API

| 函数 | 说明 |
|------|------|
| `cudaMalloc(void **ptr, size_t size)` | 在 Device 上分配线性内存 |
| `cudaFree(void *ptr)` | 释放 Device 内存 |
| `cudaMemcpy(void *dst, const void *src, size_t count, cudaMemcpyKind kind)` | Host 与 Device 间同步数据拷贝 |
| `cudaMemcpyAsync(void *dst, const void *src, size_t count, cudaMemcpyKind kind, cudaStream_t stream)` | 异步数据拷贝 |
| `cudaMemset(void *ptr, int value, size_t size)` | 用指定值填充 Device 内存 |
| `cudaHostAlloc(void **ptr, size_t size, unsigned flags)` | 分配 Pinned（页锁定）Host 内存 |
| `cudaFreeHost(void *ptr)` | 释放 Pinned Host 内存 |
| `cudaMallocManaged(void **ptr, size_t size, unsigned flags)` | 分配统一内存（Host/Device 均可访问） |
| `cudaMemPrefetchAsync(void *ptr, size_t count, int device, cudaStream_t stream)` | 预取统一内存页面到指定设备 |

### cudaMemcpyKind 枚举

| 值 | 含义 |
|----|------|
| `cudaMemcpyHostToDevice` | Host → Device |
| `cudaMemcpyDeviceToHost` | Device → Host |
| `cudaMemcpyDeviceToDevice` | Device → Device |
| `cudaMemcpyDefault` | 运行时自动推断方向（统一内存/UVA） |

### cudaHostAlloc Flags

| Flag | 说明 |
|------|------|
| `cudaHostAllocDefault` | 基本 Pinned 内存 |
| `cudaHostAllocMapped` | 映射到 Device 地址空间（零拷贝，GPU 可直接访问） |
| `cudaHostAllocPortable` | 对所有 CUDA 设备可用 |
| `cudaHostAllocWriteCombined` | 优化 Host 写入→Device 传输速度（不适合 Host 读取） |

### 典型内存操作流程

```cpp
float *d_a, *d_b, *d_c;
cudaMalloc(&d_a, N * sizeof(float));
cudaMalloc(&d_b, N * sizeof(float));
cudaMalloc(&d_c, N * sizeof(float));

cudaMemcpy(d_a, h_a, N * sizeof(float), cudaMemcpyHostToDevice);
cudaMemcpy(d_b, h_b, N * sizeof(float), cudaMemcpyHostToDevice);

kernel<<<grid, block>>>(d_a, d_b, d_c, N);

cudaMemcpy(h_c, d_c, N * sizeof(float), cudaMemcpyDeviceToHost);

cudaFree(d_a); cudaFree(d_b); cudaFree(d_c);
```

### Pinned Host 内存

```cpp
float *h_pinned;
cudaHostAlloc(&h_pinned, size, cudaHostAllocDefault);
// 利用 DMA 实现 ~2x 传输带宽
cudaFreeHost(h_pinned);
```

`cudaHostAlloc` 分配的内存不会被 OS swap out，DMA 可直接操作，传输带宽约为普通 `malloc` 内存的 2 倍。但过多分配会减少系统可用物理内存。

### 统一内存 (Managed Memory)

```cpp
float *data;
cudaMallocManaged(&data, size);
// 同一指针 Host 和 Device 均可访问
kernel<<<grid, block>>>(data, N);
cudaDeviceSynchronize();
// Host 端直接读取结果，无需显式 cudaMemcpy
cudaFree(data);
```

**工作原理**: 依赖 UVM（Unified Virtual Memory）驱动的页错误机制。GPU 访问一个不在 Device 内存中的页时，触发缺页，驱动自动通过 PCIe 迁移整个 4KB 页面。

**性能对比**（Titan Xp, 256MB）:

| 内存类型 | 吞吐量 |
|----------|--------|
| Pinned Host | ~43 MB/s（每次访问走 PCIe） |
| Managed（首访，触发全量迁移） | ~1.5 MB/s |
| Managed（迁移后） | ~1000-1300 MB/s |
| Device（cudaMalloc） | ~1000 MB/s |

**最佳实践**: 先用 `cudaMemPrefetchAsync` 提前将页面迁移到目标设备，避免运行时缺页开销。

### 内存类型选择指南

| 场景 | 推荐 |
|------|------|
| 数据完全在 GPU 上操作 | `cudaMalloc` |
| 需要高速 H↔D 传输 | `cudaHostAlloc` |
| 简化代码、无需手动拷贝 | `cudaMallocManaged` |
| GPU 偶尔读取小量 Host 数据 | `cudaHostAllocMapped` |

## Stream 管理

Stream 是异步操作的 FIFO 队列。同一 Stream 内严格有序，不同 Stream 间可并发。默认 Stream (0) 是同步串行的，显式创建 Stream 才能实现多流并发。

| 函数 | 说明 |
|------|------|
| `cudaStreamCreate(cudaStream_t *stream)` | 创建 Stream |
| `cudaStreamCreateWithFlags(cudaStream_t *stream, unsigned flags)` | 创建带属性的 Stream |
| `cudaStreamCreateWithPriority(cudaStream_t *stream, unsigned flags, int priority)` | 创建带优先级的 Stream |
| `cudaStreamDestroy(cudaStream_t stream)` | 销毁 Stream |
| `cudaStreamSynchronize(cudaStream_t stream)` | 阻塞 Host 直到 Stream 完成 |
| `cudaStreamQuery(cudaStream_t stream)` | 非阻塞查询（`cudaSuccess` = 完成，`cudaErrorNotReady` = 未完成） |
| `cudaStreamWaitEvent(cudaStream_t stream, cudaEvent_t event, unsigned flags)` | 让 Stream 等待 Event 后才继续 |

### Stream Flags

| Flag | 说明 |
|------|------|
| `cudaStreamDefault` (0) | 默认 Stream |
| `cudaStreamNonBlocking` | 非阻塞 Stream，与其他 Stream 可真正并发 |

### 多 Stream 并发示例

```cpp
const int N_STREAMS = 4;
cudaStream_t streams[N_STREAMS];
for (int i = 0; i < N_STREAMS; i++) {
    cudaStreamCreate(&streams[i]);
}

int chunkSize = N / N_STREAMS;
dim3 block(256);
dim3 grid((chunkSize + 255) / 256);

for (int i = 0; i < N_STREAMS; i++) {
    int offset = i * chunkSize;
    cudaMemcpyAsync(&d_a[offset], &h_a[offset],
                    chunkSize * sizeof(float), cudaMemcpyHostToDevice, streams[i]);
    cudaMemcpyAsync(&d_b[offset], &h_b[offset],
                    chunkSize * sizeof(float), cudaMemcpyHostToDevice, streams[i]);
    kernel<<<grid, block, 0, streams[i]>>>(&d_a[offset], &d_b[offset], &d_c[offset], chunkSize);
    cudaMemcpyAsync(&h_c[offset], &d_c[offset],
                    chunkSize * sizeof(float), cudaMemcpyDeviceToHost, streams[i]);
}

for (int i = 0; i < N_STREAMS; i++) {
    cudaStreamSynchronize(streams[i]);
    cudaStreamDestroy(streams[i]);
}
```

## Event 管理

Event 用于 GPU 端精确计时和 Stream 间依赖同步。

| 函数 | 说明 |
|------|------|
| `cudaEventCreate(cudaEvent_t *event)` | 创建 Event |
| `cudaEventCreateWithFlags(cudaEvent_t *event, unsigned flags)` | 创建带属性的 Event |
| `cudaEventDestroy(cudaEvent_t event)` | 销毁 Event |
| `cudaEventRecord(cudaEvent_t event, cudaStream_t stream)` | 在 Stream 中记录 Event |
| `cudaEventSynchronize(cudaEvent_t event)` | 阻塞 Host 直到 Event 完成 |
| `cudaEventQuery(cudaEvent_t event)` | 非阻塞查询 Event 是否完成 |
| `cudaEventElapsedTime(float *ms, cudaEvent_t start, cudaEvent_t stop)` | 计算两个 Event 之间的耗时 (ms) |

### Event Flags

| Flag | 说明 |
|------|------|
| `cudaEventDefault` (0) | 默认 |
| `cudaEventBlockingSync` | `cudaEventSynchronize` 时使用阻塞等待而非自旋 |
| `cudaEventDisableTiming` | 禁用计时功能，减少开销（仅用于同步场景） |
| `cudaEventInterprocess` | 进程间共享 |

### 计时模式

```cpp
cudaEvent_t start, stop;
cudaEventCreate(&start);
cudaEventCreate(&stop);

cudaEventRecord(start, 0);     // 记录开始
kernel<<<grid, block>>>(...);  // 执行 Kernel
cudaEventRecord(stop, 0);      // 记录结束

cudaEventSynchronize(stop);
float ms;
cudaEventElapsedTime(&ms, start, stop);
printf("Kernel 耗时: %.3f ms\n", ms);

cudaEventDestroy(start);
cudaEventDestroy(stop);
```

### Stream 间依赖

```cpp
cudaEvent_t event;
cudaEventCreate(&event);

// Stream A 完成后记录 Event，Stream B 等待 Event 后继续
cudaEventRecord(event, streamA);
cudaStreamWaitEvent(streamB, event, 0);
// Stream B 的后续操作确保 Stream A 已通过 Event 点
```

## 错误处理

所有 CUDA Runtime API 返回 `cudaError_t`。

| 函数 | 说明 |
|------|------|
| `cudaGetLastError()` | 获取并清除最近一次 API 调用的错误 |
| `cudaPeekAtLastError()` | 查看最近错误（不清除） |
| `cudaGetErrorString(cudaError_t error)` | 获取错误描述字符串 |
| `cudaGetErrorName(cudaError_t error)` | 获取错误名 |

### 同步 vs 异步 错误

* **同步错误**: 由直接 API 调用返回（如 `cudaMalloc` 失败），即时捕获
* **异步错误**: Kernel launch 或 async 操作产生的错误，**延迟**到下一次同步点返回

### 双重错误检查模式

```cpp
#define CUDA_CHECK(call)                                        \
{                                                               \
    cudaError_t err = call;                                     \
    if (err != cudaSuccess) {                                   \
        fprintf(stderr, "CUDA error at %s:%d: %s\n",             \
                __FILE__, __LINE__, cudaGetErrorString(err));     \
        exit(1);                                                \
    }                                                           \
}

// 1. 检查 Launch 配置错误（同步）
kernel<<<grid, block>>>(args);
CUDA_CHECK(cudaGetLastError());

// 2. 检查 Kernel 执行错误（异步）
CUDA_CHECK(cudaDeviceSynchronize());
```

### 常见错误码

| 错误码 | 含义 |
|--------|------|
| `cudaSuccess` | 成功 |
| `cudaErrorInvalidValue` | 参数无效 |
| `cudaErrorMemoryAllocation` | 设备内存不足 |
| `cudaErrorInvalidDevice` | 无效设备 ID |
| `cudaErrorLaunchFailure` | Kernel 启动失败 |
| `cudaErrorInvalidConfiguration` | Kernel 配置无效（如 Block 线程数超限） |
| `cudaErrorIllegalAddress` | Kernel 内非法内存访问（越界） |
| `cudaErrorNotReady` | 异步操作未完成（用于 `cudaStreamQuery` 等） |

### 调试技巧

* `cudaSetDeviceFlags(cudaDeviceScheduleBlockingSync)` 强制同步执行，便于定位错误
* `nvcc -G` 生成 Device 调试信息，配合 `cuda-gdb` 定位 Kernel 内崩溃
* `nvidia-smi` 监控 GPU 显存使用，确认分配可行性

## 函数汇总速查

```
设备管理: cudaGetDeviceCount, cudaSetDevice, cudaGetDevice, cudaGetDeviceProperties, cudaDeviceSynchronize, cudaDeviceReset
内存管理: cudaMalloc, cudaFree, cudaMemcpy, cudaMemcpyAsync, cudaMemset, cudaHostAlloc, cudaFreeHost, cudaMallocManaged, cudaMemPrefetchAsync
Kernel启动: <<<>>>
同步:     cudaDeviceSynchronize, cudaStreamSynchronize, cudaEventSynchronize
Stream:   cudaStreamCreate, cudaStreamCreateWithFlags, cudaStreamDestroy, cudaStreamQuery, cudaStreamWaitEvent
Event:    cudaEventCreate, cudaEventDestroy, cudaEventRecord, cudaEventQuery, cudaEventElapsedTime
错误处理: cudaGetLastError, cudaPeekAtLastError, cudaGetErrorString, cudaGetErrorName
```

## 来源

* [NVIDIA CUDA Runtime API 参考](https://docs.nvidia.com/cuda/cuda-runtime-api/)
* [NVIDIA CUDA C++ Programming Guide](https://docs.nvidia.com/cuda/cuda-c-programming-guide/)
* [NVIDIA Developer Forums — CUDA Error Handling](https://forums.developer.nvidia.com/)
