---
tags: [cmake, build-system, dev-tools]
type: index
aliases:
  - CMake 构建系统
  - CMake 跨平台构建
---

> CMake 是一个**跨平台构建系统生成器**（meta-build system），本身不直接构建项目，而是根据 `CMakeLists.txt` 生成目标平台的构建文件（Makefiles、Ninja、Visual Studio 解决方案、Xcode 项目等）。

## 核心概念

| 概念 | 说明 |
|------|------|
| **Meta-Build** | CMake 生成底层构建系统的输入文件，而非直接编译代码 |
| **CMakeLists.txt** | 每个项目的**构建定义文件**，描述源文件、目标、依赖和安装规则 |
| **Build Directory** | 与源码目录分离的构建目录（out-of-source build），`cmake -B build` |
| **Target** | 构建的产物（可执行文件、库、自定义目标），是现代 CMake 的核心抽象 |
| **Generator** | 后端构建系统，如 Unix Makefiles、Ninja、Visual Studio |
| **Toolchain** | 编译器和工具链描述（通过 `CMAKE_TOOLCHAIN_FILE` 或 `-DCMAKE_CXX_COMPILER=g++`） |

### modern CMake vs legacy CMake

| 维度 | Legacy CMake（2.x - 3.0 前） | Modern CMake（3.0+） |
|------|-----------------------------|----------------------|
| **变量 vs 目标** | 以全局变量为中心（`include_directories`、`link_directories`、`link_libraries`） | 以**目标**为中心（`target_include_directories`、`target_link_libraries`），属性通过接口传播 |
| **作用域** | 目录级命令影响全局 | 精确控制每个目标的公开/私有/接口属性 |
| **Generator Expressions** | 有限支持 | `$<…>` 全方位支持，在建树时而非配置时求值 |
| **依赖传播** | 手动管理包含路径和链接库 | `PUBLIC/PRIVATE/INTERFACE` 自动传播传递依赖 |
| **包管理** | `find_package` 基础用法 | `FetchContent`、`CPM` 等现代依赖管理方式 |

## 命令行基础

```bash
# 配置阶段：指定源码目录(-S)和构建目录(-B)
cmake -S . -B build

# 指定生成器
cmake -S . -B build -G Ninja

# 设置变量
cmake -S . -B build -DCMAKE_BUILD_TYPE=Release -DBUILD_SHARED_LIBS=ON

# 构建阶段
cmake --build build --parallel 8

# 指定构建目标
cmake --build build --target install

# 安装
cmake --install build --prefix /usr/local

# 清理
cmake --build build --target clean
```

## CMake 工作流程

```
Source Tree (CMakeLists.txt, *.cpp, *.h)
    │
    ▼  cmake -S src -B build
Configure Phase (解析 CMakeLists.txt, 检查依赖, 生成构建规则)
    │
    ▼
Build Tree (build/build.ninja, build/Makefile, build/*.vcxproj)
    │
    ▼  cmake --build build
Build Phase (调用编译器/链接器)
    │
    ▼
Binary Output (可执行文件、库)
```

### 关键阶段

| 阶段 | 输入 | 输出 |
|------|------|------|
| **Configure** | `CMakeLists.txt`、工具链、缓存变量 | `CMakeCache.txt`、构建规则文件 |
| **Generate** | 构建规则文件 | `Makefile`/`build.ninja`/`.sln` 等 |
| **Build** | 构建规则文件、源码 | 目标文件、可执行文件、库 |
| **Install** | 构建产物、安装规则 | 安装到指定前缀目录 |

## CMakeLists.txt 层次结构

```
project-root/
├── CMakeLists.txt         # 顶层：project(), 全局配置
├── lib/
│   ├── CMakeLists.txt     # 子目录：add_library(my_lib ...)
│   └── ...
├── app/
│   ├── CMakeLists.txt     # 子目录：add_executable(my_app ...)
│   └── ...
└── tests/
    ├── CMakeLists.txt     # 子目录：add_test(...)
    └── ...
```

子目录通过 `add_subdirectory()` 引入，每个子目录的 `CMakeLists.txt` 继承父目录的作用域。

## 文档导航

* [CMake 基本语法](基本语法.md) — CMakeLists.txt 结构、变量、控制流、函数与宏、Generator Expressions、消息打印
* [CMake 目标与属性](目标与属性.md) — Target 类型、目标属性（PUBLIC/PRIVATE/INTERFACE）、现代 CMake 依赖管理、安装
* [CMake 常用模块](常用模块.md) — find_package、FetchContent、CTest、CPack、ExternalProject、实用模块

## 来源

* [CMake 官方文档](https://cmake.org/cmake/help/latest/)
* [Modern CMake by Craig Scott](https://cliutils.gitlab.io/modern-cmake/)
* [Effective Modern CMake](https://gist.github.com/mbinna/c61dbb39bca0e4fb7d1f73b0d66a4fd1)
* [CMake Tutorial — 官方教程](https://cmake.org/cmake/help/latest/guide/tutorial/index.html)
