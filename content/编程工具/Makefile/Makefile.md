---
tags: [makefile, build-system]
type: index
---

> Makefile 是 Unix/Linux 下最基础的构建工具，通过定义目标（target）、依赖（prerequisites）和命令（recipe）来描述文件间的依赖关系和构建顺序。Make 根据文件时间戳自动判断哪些目标需要重新构建。

## 核心概念

```
目标(target): 依赖1 依赖2 ...
	命令1
	命令2
```

- **目标** — 通常是要生成的文件名（如可执行文件、目标文件），也可以是伪目标 `.PHONY`
- **依赖** — 目标生成所依赖的文件列表，依赖变化则目标需要重新构建
- **命令** — Shell 命令，必须以 Tab 开头
- **规则** — 目标 + 依赖 + 命令构成一条规则

Make 通过依赖图的拓扑排序确定构建顺序，仅重新编译发生变化的源文件，避免全量重新编译。

## 文档导航

- [基本语法](基本语法.md) — 规则/变量/伪目标/通配符/自动变量/条件判断/内置规则
- [函数](函数.md) — 文本处理函数（subst/patsubst/filter/strip）、文件名函数（dir/notdir/suffix）、流程控制函数（foreach/if/call）
- [多级目录管理](多级目录管理.md) — 递归 make、非递归 make（include 模式）、子目录模块化组织

## 来源

- [GNU Make Manual](https://www.gnu.org/software/make/manual/)
