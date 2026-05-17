---
tags: [python, programming-language]
type: index
---

## Python

Python 是一种解释型、动态类型、高级编程语言，由 Guido van Rossum 于 1991 年创建。它以代码可读性高、「开箱即用」的标准库和庞大的第三方生态著称，广泛用于 **Web 开发、脚本编写、AI/机器学习、数据科学、自动化运维** 等各个领域。

### 核心特性

| 特性 | 说明 |
|------|------|
| **解释型** | 源代码由 CPython 解释器逐行执行，无需显式编译步骤 |
| **动态类型** | 变量不需要声明类型，运行时自动推断 |
| **自动内存管理** | 引用计数 + 分代 GC，开发者无需手动管理内存 |
| **多范式** | 支持面向对象、函数式、过程式编程风格 |
| **可嵌入/扩展** | 可以用 C/C++ 扩展性能关键部分（`ctypes`、`Cython`、`pybind11`） |
| **Batteries Included** | 标准库覆盖文件 I/O、网络、并发、正则、GUI 等 |
| **庞大的生态** | PyPI 拥有超过 50 万个第三方包 |

### Python 版本

* **Python 3.x（推荐）** — 当前活跃版本（3.12 / 3.13），Python 2 已于 2020 年停止维护
* **CPython** — 官方参考实现，用 C 编写，最广泛使用的解释器
* **PyPy** — 基于 JIT 编译的替代实现，对纯 Python 代码有数倍加速
* **Cython** — 将 Python 转为 C 扩展的语言，适合性能敏感场景

### 标准库速览

| 类别 | 模块 |
|------|------|
| 文本处理 | `re`、`string`、`difflib`、`textwrap` |
| 数据结构 | `collections`、`array`、`heapq`、`bisect` |
| 文件与 I/O | `io`、`os`、`pathlib`、`tempfile`、`shutil` |
| 序列化 | `json`、`pickle`、`csv`、`xml` |
| 并发 | `threading`、`multiprocessing`、`concurrent.futures`、`asyncio` |
| 网络 | `socket`、`http`、`urllib`、`smtplib` |
| 测试 | `unittest`、`doctest` |
| 日志 | `logging` |
| 命令行 | `argparse`、`getopt`、`shlex` |

### 文档导航

* [基本语法](基本语法.md) — 缩进、变量、类型、运算符、控制流、推导式、异常、上下文管理器、f-string
* [数据结构](数据结构.md) — list / tuple / dict / set、切片、迭代模式、collections 模块
* [函数与模块](函数与模块.md) — 函数定义、lambda、装饰器、生成器、模块与包、命名空间
* [常用标准库](常用标准库.md) — os、sys、json、re、datetime、collections、pathlib、subprocess、argparse、logging
* [虚拟环境与包管理](虚拟环境与包管理.md) — venv、pip、requirements.txt、conda、poetry、pyproject.toml
