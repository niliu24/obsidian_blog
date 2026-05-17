---
tags: [linux, cgroup, namespace, container]
type: entity
aliases:
  - Linux cgroup 与 namespace
  - cgroup
  - 命名空间
  - 容器技术基础
---

> cgroup（Control Group）和 namespace 是 Linux 容器技术的基础。namespace 提供**资源隔离**（每个容器看到自己的 PID/网络/文件系统），cgroup 提供**资源限制**（限制 CPU、内存、I/O 的使用量）。两者结合构成了 Docker、Kubernetes 等容器平台的底层支撑。

## 核心对比

| 特性 | namespace | cgroup |
|------|-----------|--------|
| **作用** | 资源隔离（看到的范围） | 资源控制（使用量上限） |
| **类比** | 每个容器有自己的"世界观" | 每个容器有自己的"预算" |
| **创建方式** | `clone()` + flags (CLONE_NEW*) / `unshare()` | `mkdir /sys/fs/cgroup/<subsys>/<group>` |
| **可见性** | 进程内部可见 | 通常对进程透明 |

## namespace 类型

| namespace | 隔离内容 | CLONE_ 标志 | 引入内核 |
|-----------|---------|------------|---------|
| **Mount (mnt)** | 文件系统挂载点 | `CLONE_NEWNS` | 2.4.19 |
| **UTS** | 主机名和域名 | `CLONE_NEWUTS` | 2.6.19 |
| **IPC** | System V IPC、POSIX 消息队列 | `CLONE_NEWIPC` | 2.6.19 |
| **PID** | 进程 PID 编号 | `CLONE_NEWPID` | 2.6.24 |
| **Network (net)** | 网络设备、IP 地址、端口 | `CLONE_NEWNET` | 2.6.29 |
| **User** | 用户 ID 和组 ID | `CLONE_NEWUSER` | 3.8 |
| **Cgroup** | cgroup 根目录视图 | `CLONE_NEWCGROUP` | 4.6 |
| **Time** | 系统时间 (CLOCK_MONOTONIC/BOOTTIME) | `CLONE_NEWTIME` | 5.6 |

### namespace 关键特性

- 每个进程属于每种 namespace 的一个实例
- namespace 可通过 `/proc/<pid>/ns/` 查看和操作
- 父子进程可共享 namespace，子进程通过 `unshare()` 脱离

```c
// 创建子进程并加入新的 namespace
pid_t pid = clone(child_fn, child_stack, 
                  CLONE_NEWNS |    // 新 mount namespace
                  CLONE_NEWPID |   // 新 PID namespace
                  CLONE_NEWNET,    // 新 network namespace
                  NULL);
```

## cgroup v2 核心概念

cgroup v2 (统一层级) 简化了 cgroup v1 的多层次结构：

```
/sys/fs/cgroup/
    ├── cgroup.controllers    → 启用的控制器 (cpu, memory, io...)
    ├── cgroup.procs          → 属于该 cgroup 的进程 PID
    ├── cpu.max               → CPU 限额
    ├── memory.max            → 内存限制
    ├── memory.current        → 当前内存使用
    ├── io.max                → I/O 限制
    └── <child-group>/        → 子 cgroup
```

### cgroup 控制器

| 控制器 | 限制内容 | 关键文件 |
|--------|---------|---------|
| **cpu** | CPU 使用时间 | `cpu.max` (quota period) |
| **memory** | 内存使用上限 | `memory.max`, `memory.high` |
| **io** | 磁盘 I/O 带宽 | `io.max` (rbps/wbps/riops/wiops) |
| **pids** | 进程数量上限 | `pids.max` |
| **cpuset** | CPU 核和 NUMA 节点亲和性 | `cpuset.cpus`, `cpuset.mems` |
| **hugetlb** | 大页面内存限额 | `hugetlb.<size>.max` |

### cgroup 使用示例

```bash
# 创建名为 "myapp" 的 cgroup
mkdir /sys/fs/cgroup/myapp

# 限制最大 4 个 CPU 核心
echo "400000 100000" > /sys/fs/cgroup/myapp/cpu.max

# 限制最大 512MB 内存
echo "536870912" > /sys/fs/cgroup/myapp/memory.max

# 将进程 12345 移入 myapp cgroup
echo 12345 > /sys/fs/cgroup/myapp/cgroup.procs

# 运行新进程并放入 myapp
echo $$ > /sys/fs/cgroup/myapp/cgroup.procs
```

## 容器技术栈

```
应用层:        Docker / containerd / Podman
                    │
运行时层:      runc / crun / kata-containers
                    │
内核层:        namespace (隔离) + cgroup (限制)
                    │
安全层:        seccomp / AppArmor / SELinux / capabilities
```

## 与 Docker/K8s 的关系

- **Docker**: 每个容器是一个独立的 namespace 组 + 一个 cgroup
- **Kubernetes Pod**: 共享 Network/PID namespace，各自独立的 cgroup
- **容器镜像**: 本质上是一个 rootfs (目录树)，通过 pivot_root 切换
- **资源请求/限制**: K8s 的 `requests`/`limits` 分别映射为 cgroup 的不同阈值

## 相关文档

- [进程管理](进程管理.md) — clone() 系统调用与 namespace 创建
- [进程调度](进程调度.md) — CFS 调度器与 cgroup CPU 调度
- [内存管理](../内存管理/内存管理.md) — cgroup 内存限制与 OOM
- [eBPF](../../eBPF.md) — cgroup 相关的 eBPF hook

## 相关资料

- [Linux man pages — cgroups(7)](https://man7.org/linux/man-pages/man7/cgroups.7.html)
- [Linux man pages — namespaces(7)](https://man7.org/linux/man-pages/man7/namespaces.7.html)
- [Linux kernel — Control Group v2](https://www.kernel.org/doc/html/latest/admin-guide/cgroup-v2.html)
- [The Linux Programming Interface (Kerrisk) — Ch. 10](https://man7.org/tlpi/)
