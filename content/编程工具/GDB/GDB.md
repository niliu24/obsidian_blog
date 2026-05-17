---
tags: [gdb, debugging, c, c++]
type: index
---

> GDB（GNU Debugger）是 Linux 下标准的命令行调试器，支持 C/C++/Go/Rust 等多种语言，底层依赖 `ptrace` 系统调用实现进程控制。

## 核心能力

- **断点调试** — 行断点、函数断点、条件断点、watchpoint
- **单步执行** — step（进入函数）/ next（跳过函数）/ until（到指定行）
- **栈回溯** — backtrace 查看调用栈，frame 切换栈帧
- **变量检查** — print 打印变量，display 持续监控
- **内存检查** — x 命令以任意格式查看内存
- **Core Dump 分析** — 加载 core 文件回溯崩溃现场
- **远程调试** — gdbserver 实现交叉调试和远程调试
- **多线程调试** — 线程列表、切换、同步/异步模式
- **反向调试** — record/reverse-step/reverse-continue（x86/ARM）

## 文档导航

- [GDB 基本用法](基本用法.md) — 启动调试、断点操作、单步执行、栈与变量检查、TUI 模式

## 来源

- [GDB Documentation](https://sourceware.org/gdb/documentation/)
