---
tags: [linux, ebpf, kernel]
type: entity
aliases:
  - eBPF
  - Extended Berkeley Packet Filter
  - BPF
---

> eBPF（Extended Berkeley Packet Filter）是 Linux 内核中的一个沙箱虚拟机，允许在内核中安全地运行用户编写的程序。eBPF 最初源于网络数据包过滤（BPF），现已扩展到性能分析、安全审计、可观测性、网络处理等几乎所有内核子系统，成为 Linux 内部最核心的可编程基础设施。

## 架构概览

```
用户空间                          内核空间
┌──────────────┐              ┌──────────────────┐
│  BPF 程序源码  │              │                  │
│  (C / Rust)  │   编译       │   eBPF 字节码     │
▼              │   → clang    │                  │
│  BPF 字节码   │              │   验证器 (Verifier)│
│              │   load       │   ├─ 无环路        │
▼              │   → bpf()   │   ├─ 无越界访问    │
│  BPF 对象    │   syscall   │   ├─ 无未初始化     │
│  (map/prog)  │              │   └─ 限制指令数    │
│              │              ├──────────────────┤
│ 用户空间程序  │   交互       │   JIT 编译器      │
│  (maps读写)  │   ←→        │   字节码 → 原生码  │
│              │              ├──────────────────┤
│              │              │   挂载点 (Hook)    │
│              │              │   kprobe/tracept  │
│              │              │   XDP/TC/cgroup   │
│              │              └──────────────────┘
```

## 核心概念

| 概念 | 说明 |
|------|------|
| **eBPF 程序** | 内核运行的沙箱字节码，C/Rust 编写，clang 编译 |
| **eBPF Maps** | 内核态和用户态共享数据的 KV 存储 |
| **验证器 (Verifier)** | 静态分析 eBPF 字节码，保证安全性 |
| **JIT 编译器** | 将字节码编译为原生指令，提升性能 |
| **Helper 函数** | 内核暴露给 eBPF 程序的安全 API |
| **挂载点 (Attach Point)** | eBPF 程序挂载到内核的位置 |

## 常用 Hook 点

| 类型 | Hook 点 | 用途 |
|------|---------|------|
| **XDP** | 网卡驱动层（最早） | 高性能包处理、DDoS 防御 |
| **TC (Traffic Control)** | 内核协议栈入/出口 | 网络策略、负载均衡 |
| **cgroup** | cgroup 网络出口 | Kubernetes 网络策略 (Cilium) |
| **kprobe/kretprobe** | 任意内核函数入口/出口 | 内核级动态追踪 |
| **tracepoint** | 内核静态追踪点 | 性能分析、调试 |
| **uprobe** | 用户空间函数入口/出口 | 用户程序追踪 |
| **USDT** | 用户程序静态追踪点 | 应用性能监控 |
| **LSM** | Linux Security Modules | 安全策略扩展 |

## 示例：跟踪系统调用

```c
// trace_syscall.c — 使用 kprobe 跟踪 sys_execve
#include <linux/bpf.h>
#include <bpf/bpf_helpers.h>

SEC("kprobe/sys_execve")
int bpf_prog(struct pt_regs *ctx)
{
    char msg[] = "execve called\n";
    bpf_trace_printk(msg, sizeof(msg));
    return 0;
}

char LICENSE[] SEC("license") = "GPL";
```

编译和加载：

```bash
clang -O2 -target bpf -c trace_syscall.c -o trace_syscall.o
bpftool prog load trace_syscall.o /sys/fs/bpf/trace_syscall autoattach
```

## 关键用户空间工具

| 工具 | 用途 | 层级 |
|------|------|------|
| **bpftool** | eBPF 程序/map 管理 | 底层 |
| **bcc (BPF Compiler Collection)** | Python 绑定，快速开发 | 中层 |
| **libbpf** | C/C++ eBPF 加载库 | 底层 |
| **cilium/ebpf** | Go eBPF 库 | 底层 |
| **bpftrace** | DTrace 风格的动态追踪 | 高层 |

## eBPF 在云原生中的应用

| 项目 | 用途 |
|------|------|
| **Cilium** | Kubernetes CNI，基于 eBPF 替代 kube-proxy |
| **Falco** | 容器运行时安全监控 |
| **Pixie** | K8s 无侵入可观测性 |
| **Paraca** | eBPF Agent 框架 |
| **Katran** | Facebook 的 L4 负载均衡器 (XDP) |

## 相关文档

- [网络协议栈](../网络协议栈/网络协议栈.md) — eBPF 在网络栈中的应用 (XDP/TC)
- [cgroup 与命名空间](../进程管理/cgroup与命名空间.md) — cgroup 相关的 eBPF hook
- [系统调用](../系统调用/系统调用.md) — bpf() syscall
- [中断处理](../中断处理/中断处理.md) — 动态追踪与 kprobe

## 相关资料

- [eBPF.io — What is eBPF?](https://ebpf.io/what-is-ebpf/)
- [BPF and XDP Reference Guide (Cilium)](https://docs.cilium.io/en/stable/bpf/)
- [Linux kernel eBPF Documentation](https://www.kernel.org/doc/html/latest/bpf/)
- [Brendan Gregg — BPF Performance Tools](https://www.brendangregg.com/bpf-performance-tools-book.html)
