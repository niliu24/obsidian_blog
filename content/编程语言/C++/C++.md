---
tags: [c++, programming-language]
type: index
aliases:
  - C++ 编程语言
---

> C++ 是一种**多范式**编程语言，支持面向过程、面向对象（OOP）、泛型编程和函数式编程风格。由 Bjarne Stroustrup 于 1985 年在 C 语言基础上扩展而来，至今仍是系统编程、游戏引擎、高频交易、嵌入式和高性能计算领域的核心语言。

## 语言特性

C++ 的核心设计哲学是**零开销抽象**（Zero Overhead Abstraction）——你不需要为不使用的特性付出代价，使用的特性也不会比手写的 C 代码更慢。

| 范式 | 支持方式 |
|------|----------|
| **过程式** | C 子集：函数、指针、结构体 |
| **面向对象** | class、继承、虚函数、多态 |
| **泛型编程** | 模板（函数模板、类模板、模板特化） |
| **函数式** | Lambda、`std::function`、范围算法（C++20 Ranges） |
| **元编程** | 模板元编程（TMP）、`constexpr`、`if constexpr`（C++17）、Concepts（C++20） |

## 标准演进

| 标准 | 年份 | 关键特性 |
|------|------|----------|
| **C++98** | 1998 | 首个 ISO 标准，模板、STL、异常、RTTI |
| **C++03** | 2003 | 缺陷修复，无重大新特性 |
| **C++11** | 2011 | **现代 C++ 起点**：auto、右值引用/移动语义、智能指针、Lambda、`unordered_container`、`nullptr`、`constexpr`、可变参数模板 |
| **C++14** | 2014 | 泛型 Lambda、`constexpr` 放宽、`std::make_unique` |
| **C++17** | 2017 | `if constexpr`、折叠表达式、结构化绑定、`std::optional`/`variant`/`any`、文件系统库 |
| **C++20** | 2020 | **划时代版本**：Concepts、Ranges、协程（Coroutines）、模块（Modules）、三路比较（`<=>`）、`std::span`、`constexpr` 大幅扩展 |
| **C++23** | 2023 | `std::expected`、`std::mdspan`、`std::print`、`if consteval`、Deducing `this` |

## 编译模型

C++ 采用**分离编译**（Separate Compilation）模型：

```
源代码 (*.cpp) → 编译器 → 目标文件 (*.o)
头文件 (*.hpp/*.h) → 预处理器/include → 与 .cpp 一同编译
链接器 → 合并目标文件 → 可执行文件/库
```

| 阶段 | 说明 |
|------|------|
| **预处理** | 处理 `#include`、`#define`、条件编译 |
| **编译** | 将 C++ 源码翻译为汇编/机器码（.o/.obj） |
| **链接** | 解析符号引用，合并目标文件和库 → 可执行文件 |

> **ODR（One Definition Rule）**: 全局变量、非 inline 函数、类定义在整个程序中只能定义一次。`inline` 关键字（C++17 起）允许在多个编译单元中重复定义。

## 文档导航

* [类与对象](类与对象.md) — class、构造/析构、继承、访问控制、this 指针、static 成员、友元
* [虚函数与vtable](虚函数与vtable.md) — 虚函数机制、vtable/vptr 布局、纯虚函数、多继承、RTTI
* [智能指针](智能指针.md) — `unique_ptr`、`shared_ptr`、`weak_ptr`、循环引用、工厂函数
* [RAII与资源管理](RAII与资源管理.md) — RAII 原则、`lock_guard`、Rule of Five/Zero、自定义 RAII 封装
* [STL容器与算法](STL容器与算法.md) — 序列容器、关联容器、无序容器、迭代器、算法库、Lambda
* [Lambda与闭包](Lambda与闭包.md) — Lambda 语法、捕获模式、`std::function`、泛型 Lambda、性能
* [模板与泛型](模板与泛型.md) — 函数/类模板、特化、SFINAE、可变参模板、Concepts、TMP
* [移动语义](移动语义.md) — 左值/右值、右值引用、`std::move`、完美转发、RVO、Rule of Five

## 来源

* [C++ Reference](https://en.cppreference.com/w/)
* [ISO C++ Standard Committee](https://www.open-std.org/jtc1/sc22/wg21/)
* Bjarne Stroustrup, *The C++ Programming Language*, 4th Edition
