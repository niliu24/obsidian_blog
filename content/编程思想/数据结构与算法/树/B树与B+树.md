---
tags: [data-structure, b-tree, b-plus-tree]
type: entity
aliases:
  - B-Tree
  - B+Tree
---

> B 树是面向磁盘存储设计的平衡多路搜索树。每个节点可包含数十到数百个 key，节点大小对齐磁盘页（通常 4KB-16KB），通过**降低树高**来减少磁盘 I/O 次数。B+ 树是 B 树的变体，将全部数据存储在叶子节点并串成链表，是 MySQL InnoDB 和 PostgreSQL 的默认索引结构。

## 为什么需要 B 树

对于内存中的平衡二叉树，O(log₂ n) 非常高效。但面对磁盘：

```
磁盘寻道延迟 ~5-10ms ≈ 10,000,000 CPU 周期
内存随机访问 ~100ns ≈ 100 CPU 周期

B 树策略: 用 CPU 换 I/O — 每个节点做更多计算，换取更少的磁盘访问
```

| 参数 | 二叉树 | B 树 (t=100) |
|------|--------|-------------|
| n=10⁶ 时高度 | ~20 | ~3 |
| 磁盘 I/O 次数 | ~20 | ~3 |
| 节点内比较次数 | 1 | ~200 (O(log t)) |
| 总时间 | ~200ms (磁盘 I/O 主导) | ~30ms |

## B 树定义 (M 阶)

| 性质 | 条件 |
|------|------|
| 节点内 key | 最多 M-1 个 |
| 内部节点子节点数 | 最少 ⌈M/2⌉ 个 (根节点除外) |
| 内部节点子节点数 = key 数 + 1 | `c₀, k₁, c₁, k₂, c₂, ..., kₙ, cₙ` |
| 所有叶子节点在同一层 | 保证 O(log n) 查找 |
| 节点内 key 有序 | 支持二分查找 |

## B 树查找 — 伪代码

```
算法 B_TREE_SEARCH(node, key):
    // 在节点内二分查找第一个 ≥ key 的位置
    i ← LOWER_BOUND(node.keys, key)

    如果 i < node.nkeys 且 node.keys[i] == key:
        返回 (node, i)  // 找到

    如果 node 是叶子节点: 返回 NULL  // 未找到

    // 递归查询子节点
    DISK_READ(node.children[i])
    返回 B_TREE_SEARCH(node.children[i], key)
```

## B 树插入 — 伪代码

```
算法 B_TREE_INSERT(tree, key):
    root ← tree.root

    如果 root.nkeys == M - 1:  // 根节点满，需分裂
        new_root ← NEW_NODE()
        new_root.is_leaf ← FALSE
        new_root.children[0] ← root
        B_TREE_SPLIT_CHILD(new_root, 0)
        tree.root ← new_root

    B_TREE_INSERT_NON_FULL(tree.root, key)

算法 B_TREE_INSERT_NON_FULL(node, key):
    i ← node.nkeys - 1

    如果 node 是叶子节点:
        // 向后搬移 key，插入新 key
        当 i >= 0 且 key < node.keys[i]:
            node.keys[i+1] ← node.keys[i]
            i ← i - 1
        node.keys[i+1] ← key
        node.nkeys ← node.nkeys + 1

    否则:  // 内部节点，递归插入到子节点
        当 i >= 0 且 key < node.keys[i]:
            i ← i - 1
        i ← i + 1
        如果 node.children[i].nkeys == M - 1:
            B_TREE_SPLIT_CHILD(node, i)
            如果 key > node.keys[i]:
                i ← i + 1
        B_TREE_INSERT_NON_FULL(node.children[i], key)

算法 B_TREE_SPLIT_CHILD(parent, i):
    // 分裂父节点的第 i 个子节点 (该子节点已满)
    child ← parent.children[i]
    median_key ← child.keys[t-1]  // t = ⌈M/2⌉

    // 创建新兄弟节点
    sibling ← NEW_NODE()
    sibling.is_leaf ← child.is_leaf
    sibling.nkeys ← t - 1

    // 复制 child 的右半部分到 sibling
    对于 j 从 0 到 t-2:
        sibling.keys[j] ← child.keys[t + j]
    如果 child 不是叶子:
        对于 j 从 0 到 t-1:
            sibling.children[j] ← child.children[t + j]

    child.nkeys ← t - 1

    // 在父节点中插入分隔 key 和新兄弟指针
    对于 j 从 parent.nkeys 向下到 i+1:
        parent.keys[j] ← parent.keys[j-1]
        parent.children[j+1] ← parent.children[j]
    parent.keys[i] ← median_key
    parent.children[i+1] ← sibling
    parent.nkeys ← parent.nkeys + 1
```

### 插入例子 (M=5, t=3)

```
插入 25 号点前后的分裂:

              [10|20|30]                 ← item; item full
                   |  (split)                  
                   |
            ┌──────┴──────┐
           ▼               ▼
          [10]            [30]
           |                |
      [5|6|8|12|...]   [25|26|28|...]     ← sibling; child
    
    parent: [10|20|30] → split → [10| · |30]
                                 children: [10] [new_sibling]
    
    插入 25 到新 sibling: [10|25|30]
```

## B+ 树

B+ 树与 B 树的关键区别：

| 特性 | B 树 | B+ 树 |
|------|-----|------|
| 数据存储 | **所有节点**都存数据 | **只有叶子节点**存数据 |
| 内部节点内容 | key + data + 子节点指针 | **仅 key + 子节点指针** (更紧凑) |
| 叶子节点关系 | 无链表，彼此独立 | **叶子节点之间形成有序链表** |
| 范围查询 | 需回溯 (低效) | O(log n + k) (链表顺序遍历) |
| 重复 key | 唯一 | 内部节点 key 可能在叶子节点重复出现 |
| 应用 | 早期 RDBMS | MySQL InnoDB, PostgreSQL, 文件系统 |

```
B+ 树结构:
              [5|10]              ← 内部节点 (仅 key)
              /   |   \
         [1|3] [6|8] [12|15]      ← 内部节点
         /| \   / | \  /|  \
       [1][3] [5][6][8] [10][12][15]    ← 叶子节点 (存全部数据)
       ↕ ↕ ↕  ↕  ↕  ↕   ↕  ↕  ↕       ← 双向链表
```

## B+ 树范围查询 — 伪代码

```
算法 B_PLUS_RANGE_QUERY(tree, low, high):
    // 1. 找到 >= low 的最左叶子节点
    node ← B_TREE_SEARCH_LEAF(tree.root, low)
    key_index ← LOWER_BOUND(node.keys, low)

    // 2. 沿链表顺序遍历叶子节点
    result ← []
    当 node != NULL:
        当 key_index < node.nkeys 且 node.keys[key_index] <= high:
            result.append(node.data[key_index])
            key_index ← key_index + 1
        如果 key_index >= node.nkeys 或 node.keys[key_index] > high:
            如果 node.keys[key_index] > high: 跳出循环
            // 读取下一个叶子节点
            node ← node.next_leaf  // DISK_READ
            key_index ← 0
        否则:
            跳出循环
    返回 result
```

## 相关文档

- [二叉搜索树](二叉搜索树.md) — BST 是 B 树 (t=1) 的特例
- [红黑树](红黑树.md) — 另一种广泛使用的平衡树
- [二叉树基础](二叉树基础.md) — 树的遍历和属性基础

## 来源

- CLRS 第 18 章
- [MySQL InnoDB Clustered Index](https://dev.mysql.com/doc/refman/8.0/en/innodb-index.html)
- [PostgreSQL B-Tree Index](https://www.postgresql.org/docs/current/btree.html)
