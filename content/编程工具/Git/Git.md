---
tags: [dev-tools, git, version-control]
type: index
---

> Git 是 Linus Torvalds 开发的分布式版本控制系统。每个开发者的本地仓库都是一个完整的版本库，包含全部历史记录，不依赖中央服务器即可进行提交、分支、查看历史等操作。

## 核心概念

Git 将数据视为快照流而非差异流。每次提交保存的是整个项目的完整快照，未变化的文件仅保留引用。

```
工作目录 (Working Directory)
    │  git add
    ▼
暂存区 (Staging Area / Index)
    │  git commit
    ▼
本地仓库 (Local Repository)
    │  git push
    ▼
远程仓库 (Remote Repository)
```

### 核心对象

| 对象 | 说明 |
|------|------|
| **Blob** | 文件内容的快照，以 SHA-1 哈希寻址 |
| **Tree** | 目录的快照，记录文件名→Blob/Tree 的映射 |
| **Commit** | 一次提交，包含 Tree 指针、父提交指针、作者/提交者信息、提交说明 |
| **Tag** | 对特定 Commit 的命名引用（轻量标签 vs 附注标签） |
| **Branch** | 指向某个 Commit 的可移动指针 |

## 文档导航

- [Git 基本用法](基本用法.md) — 仓库初始化/克隆、暂存/提交、分支/合并、远程操作、历史查看、撤销操作

## 来源

- [Pro Git Book](https://git-scm.com/book/en/v2)
