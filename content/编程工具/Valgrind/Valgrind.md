---
tags: [valgrind, memory-debugging, dev-tools]
type: index
aliases:
  - Valgrind 动态分析框架
---

> Valgrind 是一个动态二进制分析框架，通过在**合成 CPU** 上运行目标程序（无需修改源码），拦截内存读写、线程同步等操作，实现内存错误检测、性能剖析和线程调试。

## 工作原理

Valgrind 将目标程序的机器码动态重编译为其内部中间表示（IR，Intermediate Representation），在合成 CPU（VEX IR）上解释执行。每条指令的副作用（内存访问、寄存器修改）都会被 Valgrind 核心拦截并分发给注册的工具插件。

```
目标程序二进制 → VEX 前端（反汇编为 IR） → Valgrind 工具插件（插桩） → VEX 后端（生成机器码） → 执行
```

这种架构的优势：

* **无需源码**：直接对二进制插桩，支持任何语言编译的程序
* **无需重链接**：运行时动态注入
* **跨工具复用**：核心基础架构（JIT、内存管理）由 Valgrind 提供，工具只需关注插桩逻辑

### 性能开销

| 工具 | 典型减速倍数 | 额外内存 |
|------|-------------|----------|
| Memcheck | 5–20x | 2–4x |
| Helgrind | 10–30x | 2–5x |
| Callgrind | 5–20x | 2–3x |
| Massif | 20–50x | 1–2x |

## 安装

```bash
# Ubuntu/Debian
sudo apt install valgrind

# Fedora/RHEL
sudo dnf install valgrind

# macOS (Homebrew)
brew install valgrind
```

## 基本调用

```bash
# 使用 Memcheck（默认工具，可省略 --tool=memcheck）
valgrind --tool=memcheck ./program

# 其他工具需显式指定
valgrind --tool=helgrind ./threaded_program
valgrind --tool=callgrind ./compute_program
valgrind --tool=massif ./heap_program
valgrind --tool=drd ./threaded_program
valgrind --tool=dhat ./program
```

## 主要工具一览

| 工具 | 用途 | 场景 |
|------|------|------|
| **Memcheck** | 内存错误检测 | 堆溢出、Use-After-Free、内存泄漏、未初始化值使用 |
| **Helgrind** | 线程错误检测 | Data Race、锁顺序死锁、POSIX 线程 API 误用 |
| **DRD** | 轻量级线程错误检测 | 类似 Helgrind，但开销更低，检测略少 |
| **Callgrind** | 指令级性能剖析 | 缓存行为分析、热点函数、调用图 |
| **Massif** | 堆内存 Profiling | 堆使用量随时间变化、峰值分配、内存碎片 |
| **DHAT** | 堆内存行为分析 | 分配/释放模式、访问频率、生命周期 |

## 通用命令行选项

| 选项 | 说明 |
|------|------|
| `--tool=<name>` | 选择工具（默认 memcheck） |
| `--log-file=<file>` | 输出到文件而非 stderr |
| `--xml=yes` | XML 格式输出（CI 集成） |
| `--xml-file=<file>` | XML 输出到文件 |
| `--suppressions=<file>` | 抑制特定告警（忽略已知误报） |
| `--vgdb=yes` | 启用 GDB 前端调试 Valgrind |
| `--trace-children=yes` | 追踪子进程（fork 场景） |
| `--num-callers=<N>` | 栈回溯深度（默认 12） |

## 文档导航

* [基本用法](基本用法.md) — 各工具详细用法、选项解读、代码示例

## 来源

* [Valgrind 官方文档](https://valgrind.org/docs/manual/manual.html)
* [Valgrind Quick Start Guide](https://valgrind.org/docs/manual/quick-start.html)
* [Valgrind User Manual — Core](https://valgrind.org/docs/manual/manual-core.html)
