---
tags: [asan, address-sanitizer, debugging, memory]
type: index
---

> AddressSanitizer（ASan）是 Google 开发的快速内存错误检测工具，集成在 GCC/Clang 编译器中，运行时开销仅约 2x，远低于 Valgrind，适合开发调试和测试阶段使用。

## 核心能力

ASan 通过编译时插桩（instrumentation）和运行时影子内存（shadow memory）机制，可检测以下内存错误：

- **堆越界** — `new[]` / `malloc` 分配的内存越界读写
- **栈越界** — 栈上局部变量/数组越界
- **全局变量越界** — 全局/静态变量缓冲区溢出
- **Use-After-Free** — 释放后使用
- **Double-Free** — 重复释放
- **内存泄漏** — 程序退出时报告泄漏（ASan 自带，或配合 LSAN）

## 与 Valgrind 对比

| 特性 | ASan | Valgrind (Memcheck) |
|------|------|---------------------|
| 实现方式 | 编译时插桩 | 动态二进制翻译 |
| 运行时开销 | ~2x | ~10-20x |
| 内存开销 | ~2-3x | 无额外开销（但极慢） |
| 检测精度 | 高（精确到字节级） | 高 |
| 无需重编译 | 否 | 是 |

## 文档导航

- [ASAN 基本用法](ASAN基本用法.md) — 编译选项、环境变量配置、常见错误报告解读、抑制规则

## 来源

- [AddressSanitizer — Google Sanitizers](https://github.com/google/sanitizers/wiki/AddressSanitizer)
