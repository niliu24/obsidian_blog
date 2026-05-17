---
tags: [memory, numa, hardware]
type: entity
---

> NUMA (Non-Uniform Memory Access) 是多路 CPU 系统中主存访问延迟不均衡的内存架构。与 UMA (Uniform Memory Access) 不同，NUMA 系统中每个 CPU Socket 都有本地 DRAM，访问本地内存延迟远低于访问远端 Socket 的内存。理解 NUMA 拓扑对于高性能服务器应用的性能调优至关重要。

## UMA vs NUMA

### UMA (Symmetric Multiprocessing, SMP)

传统 UMA 架构中，所有 CPU 核心通过共享前端总线 (FSB) 或内存控制器访问统一的内存池：

```
    CPU 0         CPU 1
      │             │
      └──────┬──────┘
             │
        内存控制器
             │
           DRAM
```

* 延迟：所有 CPU → 所有内存地址延迟一致
* 瓶颈：多核争用总线带宽
* 局限：总线频率和带宽难以随核心数扩展

### NUMA 架构

```
Socket 0                  Socket 1
┌─────────────┐          ┌─────────────┐
│ CPU Cores   │          │ CPU Cores   │
│ L1/L2/L3    │          │ L1/L2/L3    │
│ DDR5 控制器   │          │ DDR5 控制器   │
│ ┌─────────┐ │          │ ┌─────────┐ │
│ │ 本地 DRAM │ │          │ │ 本地 DRAM │ │
│ └─────────┘ │          │ └─────────┘ │
└──────┬──────┘          └──────┬──────┘
       │                       │
       └─────────┬─────────────┘
                 │
             互联 (UPI / Infinity Fabric / CXL)
```

* 每个 Socket 有自己的内存控制器和本地 DRAM
* 本地访问：低延迟、高带宽
* 远端访问：跨 Socket 互联（如 Intel UPI / AMD Infinity Fabric），延迟更高、带宽更低

## NUMA 节点拓扑

### 节点距离矩阵 (Node Distance)

Linux 内核通过 `SLIT` (System Locality Information Table, ACPI) 报告各 NUMA 节点之间的 " 距离 "：

```
# 4 节点系统示例（2 路 × 每路 2 个 NUMA 域）
        Node 0  Node 1  Node 2  Node 3
Node 0   10      20      30      30
Node 1   20      10      30      30
Node 2   30      30      10      20
Node 3   30      30      20      10
```

* 距离 10：自身（最小）
* 距离 20：同一 Socket 的不同 NUMA 域，或相邻 Socket
* 距离 30：跨 Socket 远端

### NUMA Factor

NUMA Factor = 远端访问延迟 / 本地访问延迟

| 架构 | 互连技术 | 本地延迟 | 远端延迟 | NUMA Factor |
|------|---------|---------|---------|-------------|
| Intel Xeon (2S) | UPI 11.2 GT/s | ~80-100 ns | ~140-170 ns | ~1.5-1.8x |
| AMD EPYC (2S) | Infinity Fabric | ~100-130 ns | ~180-230 ns | ~1.5-2x |
| AMD EPYC (4S+) | xGMI | 同上 | ~250-350 ns | ~2-3x |

> **注意**：NUMA Factor 在 1.5-2x 之间时，应用性能影响较小；超过 2x 时需要显式 NUMA 亲和性管理。

## 查看 NUMA 拓扑

### numactl --hardware

```bash
numactl --hardware

# 输出示例
available: 4 nodes (0-3)
node 0 cpus: 0 1 2 3 4 5 6 7
node 0 size: 65536 MB
node 0 free: 32210 MB
node 1 cpus: 8 9 10 11 12 13 14 15
node 1 size: 65536 MB
node 1 free: 28945 MB
node 2 cpus: 16 17 18 19 20 21 22 23
node 2 size: 65536 MB
node 2 free: 18734 MB
node 3 cpus: 24 25 26 27 28 29 30 31
node 3 size: 65536 MB
node 3 free: 24567 MB
node distances:
node   0   1   2   3
  0:  10  20  30  30
  1:  20  10  30  30
  2:  30  30  10  20
  3:  30  30  20  10
```

### /sys 文件系统

```bash
# 查看 NUMA 节点列表
ls /sys/devices/system/node/
# node0  node1  node2  node3

# 查看节点 0 的 CPU 列表
cat /sys/devices/system/node/node0/cpulist
# 0-7

# 查看节点间的距离矩阵
cat /sys/devices/system/node/node0/distance
# 10 20 30 30
```

### numastat

```bash
# 监控 NUMA 分配统计
numastat

# 输出示例
                           node0           node1
numa_hit                  1234567         7654321
numa_miss                     123            456   ← 本地内存不足时请求远端
numa_foreign                  456            123   ← 其他节点发来的远端请求
interleave_hit                  0              0
local_node                 1234000         7654000
other_node                     178            321   ← 跨节点访问次数
```

* `numa_hit`：成功在期望节点分配的次数
* `numa_miss`：在非期望节点分配的次数（通常意味着远端访问）
* `numa_foreign`：其他节点预期在本节点但未成功的次数
* `other_node`：本节点 CPU 访问远端内存的次数

## Linux NUMA 感知内存管理

### First-Touch 策略

Linux 默认采用 **First-Touch** 分配策略：**哪个 CPU 线程首次访问某个内存页，该页就分配在那个线程所在节点的本地内存**。

```c
// 例子：Node 0 的线程分配并首次访问数组 → 数组分配在 Node 0 的 DRAM
// 随后 Node 1 的线程访问同一数组 → 远端访问！
int *data = malloc(N * sizeof(int));

// Thread on CPU 0 (Node 0):
for (int i = 0; i < N; i++)
    data[i] = i;     // ← 首次接触 (Page Fault)，物理页分配在 Node 0

// Thread on CPU 1 (Node 1): 后续访问 data 将产生远端访问
```

**对程序员的启示**：

* 将数据初始化的线程与频繁访问的线程放在同一个 NUMA 节点
* 对大数据集使用 **local allocation** —— 在哪个节点使用就在哪个节点初始化
* OpenMP 中使用 `numactl` 控制或修改 `OMP_PLACES` 环境变量

### mbind / set_mempolicy

通过系统调用显式控制内存分配策略：

```c
#include <numa.h>
#include <numaif.h>

// 将特定内存区域绑定到指定节点
struct bitmask *nodes = numa_allocate_nodemask();
numa_bitmask_setbit(nodes, 0);  // 绑定到 Node 0

mbind(ptr, size, MPOL_BIND, nodes->maskp, nodes->size, MPOL_MF_MOVE);
```

| 策略 | 宏 | 说明 |
|------|---|------|
| 默认 | MPOL_DEFAULT | First-Touch |
| 绑定 | MPOL_BIND | 严格绑定到指定节点集 |
| 首选 | MPOL_PREFERRED | 优先使用指定节点，不够再其他节点 |
| 交织 | MPOL_INTERLEAVE | 以页为单位轮询分配到多个节点（适合大内存 HPC） |

### move_pages -- 在线页迁移

```c
#include <numaif.h>

// 将进程的指定页面迁移到目标节点
int move_pages(int pid, unsigned long count,
               void **pages, const int *nodes,
               int *status, int flags);
```

实际使用：

```bash
# 安装 numactl
sudo apt-get install numactl

# 查看进程当前的 NUMA 内存分布
numastat -p <PID>

# 使用 numactl 绑定进程到特定节点
numactl --membind=0 --cpunodebind=0 ./my_app
```

## 使用 numactl 绑定进程

```bash
# 查看硬件拓扑
numactl --hardware

# 将程序绑定到 Node 0 的 CPU 和内存
numactl --cpunodebind=0 --membind=0 ./my_app

# 只绑定内存（CPU 可自由调度）
numactl --membind=0 ./my_app

# 内存交织（高带宽场景）
numactl --interleave=all ./my_app

# 指定 CPU 核集 + 内存节点
numactl --physcpubind=0-7 --membind=0 ./my_app
```

### 性能对比

```bash
# 不绑定的跨节点访问
time numactl --cpunodebind=0 ./stream
# 带宽: ~45 GB/s (远端)
# 延迟: ~170 ns

# 绑定到同一节点
time numactl --cpunodebind=0 --membind=0 ./stream
# 带宽: ~85 GB/s (本地) ← 近 2x 提升
# 延迟: ~95 ns
```

## NUMA 在虚拟化 / 云环境

### 虚拟机 vNUMA

* 大型虚拟机（>1 个 vNUMA 节点）需要向 Guest OS 暴露 NUMA 拓扑
* 虚拟机管理程序将 vNUMA 节点映射到物理 NUMA 节点
* **未暴露 NUMA 拓扑的虚拟机**：Guest OS 会以为所有内存访问延迟一致，导致性能不可预测

```bash
# QEMU/KVM 启用 vNUMA
qemu-system-x86_64 \
  -numa node,cpus=0-7,memdev=mem0 \
  -numa node,cpus=8-15,memdev=mem1 \
  -numa dist,src=0,dst=1,val=20 \
  ...
```

### 云主机中的 NUMA (Kubernetes)

* 使用 **Topology Manager** 管理 CPU 和设备的 NUMA 亲和性
* **NUMA Aware Scheduling**：将 Pod 调度到与所需资源 (CPU/设备) 同一 NUMA 节点的 Worker Node
* SR-IOV 设备和 GPU 亲和性依赖正确的 NUMA 映射

## Sub-NUMA Clustering (SNC)

Intel Granite Rapids / AMD Zen 4+ 引入 SNC，将一个物理 NUMA 节点进一步拆分为多个 Sub-NUMA 域：

```
传统 NUMA:
  Socket 0 (1 NUMA Node: Node 0)
   ┌────────────────────┐
   │  L3 Cache           │
   │  ┌──┐┌──┐┌──┐┌──┐  │
   │  │0 ││1 ││2 ││3 │  │
   │  └──┘└──┘└──┘└──┘  │
   │  DRAM Controller    │
   └────────────────────┘

SNC 模式 (2 SNC Domains):
  Socket 0
   ┌────────────────────┐
   │  Node 0   Node 4    │
   │  ┌──┐┌──┐ ┌──┐┌──┐ │
   │  │0 ││1 │ │2 ││3 │ │
   │  └──┘└──┘ └──┘└──┘ │
   │  ┌─────┐ ┌─────┐   │
   │  │DRAM │ │DRAM │   │
   │  │ Ch0  │ │ Ch1  │   │
   │  └─────┘ └─────┘   │
   └────────────────────┘
```

**SNC 优势**：

* 更小的本地域：L3 Cache 的部分切分，减少跨域延迟
* 更好的内存带宽隔离：每域独立的内存通道
* 提升内存密集型应用的 locality 表现

## 来源

* [Linux Kernel NUMA Documentation](https://www.kernel.org/doc/html/latest/admin-guide/numa.html)
* [NUMA Deep Dive Series — Frank Denneman](https://frankdenneman.nl/2016/07/07/numa-deep-dive-series/)
* [Linux 内核源码: mm/mempolicy.c](https://elixir.bootlin.com/linux/latest/source/mm/mempolicy.c)
* [numactl man page](https://man7.org/linux/man-pages/man8/numactl.8.html)
* [Intel Xeon Processor Scalable Family Technical Overview](https://www.intel.com/content/www/us/en/products/docs/processors/xeon/scalable/xeon-scalable-platform-brief.html)
* [AMD EPYC Processor Architecture](https://www.amd.com/en/processors/epyc-architecture)
* [Kubernetes Topology Manager](https://kubernetes.io/docs/tasks/administer-cluster/topology-manager/)
