---
tags: [gcc, compiler, dev-tools]
type: index
aliases:
  - GNU Compiler Collection
  - GCC 编译器
---

> GCC（GNU Compiler Collection）是 Linux 上事实标准的 C/C++ 编译器套件，支持 C、C++、Fortran、Go、Ada 等多种语言。`gcc` 和 `g++` 核心区别在于链接阶段自动链接的运行时库不同。GCC 依赖 **binutils**（`as`/`ld`/`ar`/`nm`/`objdump`/`readelf`）完成汇编、链接和目标文件分析。

## 文档导航

* [编译流程](编译流程.md) — 预处理 → 编译 → 汇编 → 链接，完整流水线
* [常用选项](常用选项.md) — 优化、调试、警告、标准、架构相关选项
* [链接与库](链接与库.md) — 静态库 (`.a`)、动态库 (`.so`)、符号解析、链接器脚本

## 快速参考

| 命令 | 说明 |
|------|------|
| `gcc hello.c -o hello` | 一步编译 + 链接，生成可执行文件 |
| `gcc -c hello.c` | 仅编译到目标文件 (`.o`)，不链接 |
| `gcc -O2 -Wall hello.c -o hello` | 带优化和警告编译 |
| `g++ main.cpp -o main -std=c++17` | 使用 C++17 标准编译 C++ 程序 |
| `gcc -shared -fPIC libfoo.c -o libfoo.so` | 生成动态共享库 |
| `ar rcs libfoo.a foo1.o foo2.o` | 打包静态库 |

## 来源

* [GCC Documentation](https://gcc.gnu.org/onlinedocs/)
* [GNU Binutils](https://www.gnu.org/software/binutils/)
