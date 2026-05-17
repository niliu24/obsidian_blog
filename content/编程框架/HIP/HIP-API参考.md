---
tags: [hip, gpu, api-reference, rocm]
type: entity
---

> HIP Runtime API 对标 CUDA Runtime API，命名规则 `cudaXxx` → `hipXxx`，参数和语义一致。头文件 `<hip/hip_runtime_api.h>` 和 `<hip/hip_runtime.h>`。

## 设备管理

设备由 0-based 整数 ID 标识。

| 函数 | 说明 |
|------|------|
| `hipGetDeviceCount(int *count)` | 获取可用 GPU 数量 |
| `hipSetDevice(int deviceId)` | 设置当前线程使用的 GPU（Thread-Local） |
| `hipGetDevice(int *deviceId)` | 获取当前线程使用的 GPU ID |
| `hipGetDeviceProperties(hipDeviceProp_t *prop, int deviceId)` | 查询设备属性 |
| `hipDeviceSynchronize()` | 阻塞 Host 直到当前设备所有 Stream 完成（重量级同步） |
| `hipDeviceReset()` | 重置设备，清理所有分配资源 |

### hipDeviceProp_t 关键字段

| 字段 | 说明 |
|------|------|
| `name[256]` | 设备名称 |
| `totalGlobalMem` | 全局内存总量 |
| `sharedMemPerBlock` | 每个 Block 的共享内存 |
| `maxThreadsPerBlock` | Block 最大线程数 |
| `maxThreadsDim[3]` | 每个维度最大线程数 |
| `multiProcessorCount` | SM/CU 数量 |
| `clockRate` | 时钟频率 (kHz) |
| `warpSize` | Wavefront 大小 (AMD=32, NVIDIA=32) |
| `arch` | 架构相关属性 |

### 使用示例

```cpp
int deviceCount;
hipGetDeviceCount(&deviceCount);
hipSetDevice(0);

hipDeviceProp_t props;
hipGetDeviceProperties(&props, 0);
printf("GPU: %s, 显存: %zu MB, CU: %d\n",
       props.name,
       props.totalGlobalMem / (1024 * 1024),
       props.multiProcessorCount);
```

## 内存管理

### 基本 API

| 函数 | 说明 |
|------|------|
| `hipMalloc(void **ptr, size_t size)` | 在 Device 上分配线性内存 |
| `hipFree(void *ptr)` | 释放 Device 内存 |
| `hipMemcpy(void *dst, const void *src, size_t count, hipMemcpyKind kind)` | Host 与 Device 间数据拷贝 |
| `hipMemset(void *ptr, int value, size_t size)` | 用指定值填充 Device 内存 |
| `hipHostMalloc(void **ptr, size_t size, unsigned flags)` | 分配 Pinned（页锁定）Host 内存 |
| `hipHostFree(void *ptr)` | 释放 Pinned Host 内存 |
| `hipMallocManaged(void **ptr, size_t size, unsigned flags)` | 分配统一内存（Host/Device 均可访问） |

### hipMemcpyKind 枚举

| 值 | 含义 |
|----|------|
| `hipMemcpyHostToDevice` | Host → Device |
| `hipMemcpyDeviceToHost` | Device → Host |
| `hipMemcpyDeviceToDevice` | Device → Device |
| `hipMemcpyDefault` | 运行时根据指针自动推断方向（统一内存） |

### 典型内存操作流程

```cpp
float *d_a;
hipMalloc(&d_a, N * sizeof(float));                     // 分配 Device 内存
hipMemcpy(d_a, h_a, N * sizeof(float), hipMemcpyHostToDevice);  // H2D
kernel<<<grid, block>>>(d_a, N);                        // Kernel 操作
hipMemcpy(h_a, d_a, N * sizeof(float), hipMemcpyDeviceToHost);  // D2H
hipFree(d_a);                                           // 释放
```

### Pinned Host 内存

```cpp
float *h_pinned;
hipHostMalloc(&h_pinned, size, hipHostMallocDefault);
// ... 用于高速 PCIe 传输 ...
hipHostFree(h_pinned);
```

* 避免页面交换，提升 PCIe 传输带宽
* 过多使用会减少系统可用内存，影响系统性能

### 统一内存 (Managed Memory)

```cpp
float *data;
hipMallocManaged(&data, size);
// Host 和 Device 均可直接访问
kernel<<<grid, block>>>(data, N);
hipDeviceSynchronize();
// Host 端也可直接读取结果，无需显式 hipMemcpy
hipFree(data);
```

## Stream 管理

Stream 是异步操作的 FIFO 队列，同一 Stream 内操作严格有序，不同 Stream 间可并发。

| 函数 | 说明 |
|------|------|
| `hipStreamCreate(hipStream_t *stream)` | 创建 Stream |
| `hipStreamCreateWithFlags(hipStream_t *stream, unsigned flags)` | 创建带属性的 Stream |
| `hipStreamCreateWithPriority(hipStream_t *stream, unsigned flags, int priority)` | 创建带优先级的 Stream |
| `hipStreamDestroy(hipStream_t stream)` | 销毁 Stream |
| `hipStreamSynchronize(hipStream_t stream)` | 阻塞 Host 直到 Stream 完成 |
| `hipStreamQuery(hipStream_t stream)` | 非阻塞查询 Stream 是否完成 |
| `hipStreamWaitEvent(hipStream_t stream, hipEvent_t event, unsigned flags)` | 让 Stream 等待 Event 后才继续 |

### Stream Flags

| Flag | 说明 |
|------|------|
| `hipStreamDefault` (0) | 默认 Stream，同步行为 |
| `hipStreamNonBlocking` | 非阻塞 Stream，允许与其他 Stream 真正并发 |

### 多 Stream 并发示例

```cpp
const int N_STREAMS = 4;
hipStream_t streams[N_STREAMS];
for (int i = 0; i < N_STREAMS; i++) {
    hipStreamCreate(&streams[i]);
}

int chunkSize = N / N_STREAMS;
dim3 block(256);
dim3 grid((chunkSize + 255) / 256);

for (int i = 0; i < N_STREAMS; i++) {
    int offset = i * chunkSize;
    hipMemcpyAsync(&d_a[offset], &h_a[offset],
                   chunkSize * sizeof(float), hipMemcpyHostToDevice, streams[i]);
    hipLaunchKernelGGL(kernel, grid, block, 0, streams[i],
                       &d_a[offset], &d_b[offset], &d_c[offset], chunkSize);
    hipMemcpyAsync(&h_c[offset], &d_c[offset],
                   chunkSize * sizeof(float), hipMemcpyDeviceToHost, streams[i]);
}

for (int i = 0; i < N_STREAMS; i++) {
    hipStreamSynchronize(streams[i]);
    hipStreamDestroy(streams[i]);
}
```

## Event 管理

Event 用于精确计时和 Stream 间同步。

| 函数 | 说明 |
|------|------|
| `hipEventCreate(hipEvent_t *event)` | 创建 Event |
| `hipEventCreateWithFlags(hipEvent_t *event, unsigned flags)` | 创建带属性的 Event |
| `hipEventDestroy(hipEvent_t event)` | 销毁 Event |
| `hipEventRecord(hipEvent_t event, hipStream_t stream)` | 在 Stream 中记录 Event |
| `hipEventSynchronize(hipEvent_t event)` | 阻塞 Host 直到 Event 完成 |
| `hipEventQuery(hipEvent_t event)` | 非阻塞查询 Event 是否完成 |
| `hipEventElapsedTime(float *ms, hipEvent_t start, hipEvent_t stop)` | 计算两个 Event 之间的耗时 (ms) |

### 计时模式

```cpp
hipEvent_t start, stop;
hipEventCreate(&start);
hipEventCreate(&stop);

hipEventRecord(start, 0);     // 记录开始
kernel<<<grid, block>>>(...); // 执行 Kernel
hipEventRecord(stop, 0);      // 记录结束

hipEventSynchronize(stop);
float ms;
hipEventElapsedTime(&ms, start, stop);
printf("Kernel 耗时: %.3f ms\n", ms);

hipEventDestroy(start);
hipEventDestroy(stop);
```

### Stream 间依赖

```cpp
hipEvent_t event;
hipEventCreate(&event);

// Stream A 执行后记录 Event，Stream B 等待 Event 后再执行
hipEventRecord(event, streamA);
hipStreamWaitEvent(streamB, event, 0);
// Stream B 后续操作确保 Stream A 的 Event 已通过
```

## 异步内存拷贝

```cpp
hipMemcpyAsync(void *dst, const void *src, size_t count,
               hipMemcpyKind kind, hipStream_t stream);
```

与 `hipMemcpy` 的区别：异步执行，不阻塞 Host，需指定 Stream。适用于多 Stream 并发场景中实现数据传输与计算的 Overlap。

## 错误处理

所有 HIP Runtime API 返回 `hipError_t`。

| 宏/函数 | 说明 |
|--------|------|
| `hipGetLastError()` | 获取最近一次 API 调用的错误 |
| `hipGetErrorString(hipError_t error)` | 获取错误描述字符串 |
| `hipGetErrorName(hipError_t error)` | 获取错误名 |
| `hipPeekAtLastError()` | 查看最近错误（不清除） |

### 常用错误检查宏

```cpp
#define HIP_CHECK(call)                                          \
{                                                                \
    hipError_t err = call;                                       \
    if (err != hipSuccess) {                                     \
        fprintf(stderr, "HIP error at %s:%d: %s\n",              \
                __FILE__, __LINE__, hipGetErrorString(err));      \
        exit(1);                                                 \
    }                                                            \
}
```

### 常见错误码

| 错误码 | 含义 |
|--------|------|
| `hipSuccess` | 成功 |
| `hipErrorInvalidValue` | 参数无效 |
| `hipErrorMemoryAllocation` | 内存分配失败 |
| `hipErrorInvalidDevice` | 无效设备 ID |
| `hipErrorNotReady` | 异步操作未完成（用于 Query 非阻塞查询） |
| `hipErrorLaunchFailure` | Kernel 启动失败 |
| `hipErrorInvalidConfiguration` | Kernel 配置无效（如 Block 线程数超限） |

## 函数汇总速查

```
设备管理: hipGetDeviceCount, hipSetDevice, hipGetDevice, hipGetDeviceProperties, hipDeviceSynchronize, hipDeviceReset
内存管理: hipMalloc, hipFree, hipMemcpy, hipMemcpyAsync, hipMemset, hipHostMalloc, hipHostFree, hipMallocManaged
Kernel启动: hipLaunchKernelGGL, <<<>>>
同步:     hipDeviceSynchronize, hipStreamSynchronize, hipEventSynchronize
Stream:   hipStreamCreate, hipStreamCreateWithFlags, hipStreamDestroy, hipStreamQuery, hipStreamWaitEvent
Event:    hipEventCreate, hipEventDestroy, hipEventRecord, hipEventQuery, hipEventElapsedTime
错误处理: hipGetLastError, hipGetErrorString, hipGetErrorName, hipPeekAtLastError
```

## 来源

* [AMD ROCm HIP Runtime API 参考](https://rocm.docs.amd.com/projects/HIP/en/latest/reference/hip_runtime_api.html)
* [AMD ROCm HIP FAQ](https://rocm.docs.amd.com/projects/HIP/en/latest/user_guide/faq.html)
* [AMD HPCTrainingExamples - HIP Programming](https://github.com/amd/HPCTrainingExamples)
