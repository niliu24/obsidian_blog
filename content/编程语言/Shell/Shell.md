---
tags: [shell, bash, programming-language]
type: index
---

## Shell

Shell 既是一个命令解释器（Command Interpreter），也是一门脚本编程语言。它运行在操作系统内核之上，为用户提供与系统交互的接口。在 Linux/Unix 系统中，Shell 是最核心的用户界面，也是 DevOps、系统管理、自动化运维的基石。

### 常见 Shell

| Shell | 路径 | 特点 |
|-------|------|------|
| **Bash** | `/bin/bash` | 最广泛使用的 Shell，Linux 默认，POSIX 兼容，功能丰富 |
| **Zsh** | `/bin/zsh` | Bash 超集，插件系统强大，macOS 默认，Oh My Zsh 生态 |
| **Fish** | `/usr/bin/fish` | 开箱即用，语法高亮，自动补全智能，不兼容 POSIX |
| **sh** | `/bin/sh` | POSIX 标准 Shell，最小化功能，通常指向 dash 或 bash |
| **dash** | `/bin/dash` | Debian/Ubuntu 的 `/bin/sh`，执行速度快，无交互功能 |

Bash（Bourne Again Shell）是事实上的脚本编程标准，绝大多数 Linux 发行版将其作为默认 Shell。本文所有示例基于 Bash。

### Shell 的角色

* **命令解释器**：接收用户输入的命令，解析后交给操作系统执行
* **脚本语言**：支持变量、条件判断、循环、函数、算术运算等编程范式
* **进程管理**：启动、停止、后台运行、信号控制
* **I/O 重定向**：管道、文件描述符、输入输出流控制
* **文本处理**：与 grep/sed/awk 等工具配合，组成强大的文本处理管线
* **系统管理**：批量操作、定时任务、日志处理、配置管理

### 知识地图

* [基本语法与变量](基本语法与变量.md) — Shebang、变量声明与使用、环境变量、特殊变量、命令替换、引号机制
* [流程控制](流程控制.md) — if/elif/else、test 条件测试、for/while/until 循环、case 模式匹配、break/continue
* [函数与脚本](函数与脚本.md) — 函数定义与传参、return 与返回值、局部变量、脚本调试、信号处理、编写健壮脚本
* [文本处理](文本处理.md) — grep/sed/awk/cut/sort/uniq/wc/tr/xargs 的使用与管道组合
* [常用命令速查](常用命令速查.md) — 文件操作、文本查看、系统信息、网络工具、进程管理、归档压缩、搜索命令速查表
