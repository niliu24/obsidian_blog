---
tags: [data-structure, avl, balanced-tree]
type: entity
aliases:
  - AVL 树
  - 高度平衡二叉搜索树
---

> AVL 树是最早发明的自平衡二叉搜索树，通过维护**任意节点左右子树高度差 ≤ 1**（平衡因子）来保证 O(log n) 操作。每次插入/删除后检查平衡因子，一旦失衡，通过对应的旋转恢复平衡。

## 平衡因子

```
平衡因子 BF = 左子树高度 - 右子树高度

BF = 0:  左右等高 (平衡)
BF = ±1: 高度差 1 (可接受)
BF = ±2: 失衡，需旋转
```

## 四种旋转

| 失衡模式 | 条件 | 矫正操作 |
|----------|------|----------|
| **LL** | BF(node)=2 且 BF(node.left)≥0 | 右旋 (Right Rotate) |
| **RR** | BF(node)=-2 且 BF(node.right)≤0 | 左旋 (Left Rotate) |
| **LR** | BF(node)=2 且 BF(node.left)<0 | 左旋左子 + 右旋自身 |
| **RL** | BF(node)=-2 且 BF(node.right)>0 | 右旋右子 + 左旋自身 |

### LL 失衡 — 右旋

```
       z (BF=2)                 y
      / \                      / \
     y   T4    ───右旋──→     x   z
    / \          ←──左旋──   / \ / \
   x   T3                   T1 T2 T3 T4
  / \
 T1 T2
```

```
算法 RIGHT_ROTATE(z):
    y ← z.left
    T3 ← y.right

    y.right ← z
    z.left ← T3

    更新高度(z)
    更新高度(y)
    返回 y  // 新的子树根
```

### RR 失衡 — 左旋

```
     z (BF=-2)               y
    / \                      / \
   T1  y      ───左旋──→    z   x
      / \    ←──右旋──     / \ / \
     T2  x                T1 T2 T3 T4
        / \
       T3 T4
```

```
算法 LEFT_ROTATE(z):
    y ← z.right
    T2 ← y.left

    y.left ← z
    z.right ← T2

    更新高度(z)
    更新高度(y)
    返回 y
```

### LR 失衡 — 先左旋左子，再右旋自身

```
      z (BF=2)               z (BF=2)              x
     / \                     / \                   / \
    y  T4    ──左旋y──→     x  T4   ──右旋z──→   y   z
   / \                     / \                   / \ / \
  T1  x                    y  T3               T1 T2 T3 T4
     / \                  / \
    T2 T3                T1 T2
```

```
算法 LEFT_RIGHT_ROTATE(z):
    z.left ← LEFT_ROTATE(z.left)
    返回 RIGHT_ROTATE(z)
```

### RL 失衡 — 先右旋右子，再左旋自身

```
     z (BF=-2)             z (BF=-2)                 x
    / \                    / \                       / \
   T1  y      ──右旋y──→  T1  x     ──左旋z──→     z   y
      / \                     / \                 / \ / \
     x  T4                   T2  y              T1 T2 T3 T4
    / \                        / \
   T2 T3                      T3 T4
```

```
算法 RIGHT_LEFT_ROTATE(z):
    z.right ← RIGHT_ROTATE(z.right)
    返回 LEFT_ROTATE(z)
```

## 插入 — 伪代码

```
算法 AVL_INSERT(node, key):
    如果 node == NULL: 返回 NEW_NODE(key)

    如果 key < node.key:
        node.left ← AVL_INSERT(node.left, key)
    否则如果 key > node.key:
        node.right ← AVL_INSERT(node.right, key)
    否则:
        返回 node  // key 已存在

    更新高度(node)
    BF ← 左高(node) - 右高(node)

    // LL
    如果 BF > 1 且 key < node.left.key:
        返回 RIGHT_ROTATE(node)
    // RR
    如果 BF < -1 且 key > node.right.key:
        返回 LEFT_ROTATE(node)
    // LR
    如果 BF > 1 且 key > node.left.key:
        node.left ← LEFT_ROTATE(node.left)
        返回 RIGHT_ROTATE(node)
    // RL
    如果 BF < -1 且 key < node.right.key:
        node.right ← RIGHT_ROTATE(node.right)
        返回 LEFT_ROTATE(node)

    返回 node
```

## 删除 — 伪代码

```
算法 AVL_REMOVE(node, key):
    如果 node == NULL: 返回 NULL

    如果 key < node.key:
        node.left ← AVL_REMOVE(node.left, key)
    否则如果 key > node.key:
        node.right ← AVL_REMOVE(node.right, key)
    否则:  // 找到待删除节点
        如果 node.left == NULL 或 node.right == NULL:
            temp ← node.left ? node.left : node.right
            如果 temp == NULL:  // 叶子节点
                node ← NULL
            否则:  // 单子节点
                node ← temp
        否则:  // 双子节点
            succ ← GET_MIN(node.right)
            node.key ← succ.key
            node.right ← AVL_REMOVE(node.right, succ.key)

    如果 node == NULL: 返回 NULL

    更新高度(node)
    BF ← 左高(node) - 右高(node)

    // 四种失衡修正 (与插入相同，但条件基于 BF 而非 key)
    如果 BF > 1:
        如果 左高(node.left) >= 0:  返回 RIGHT_ROTATE(node)
        否则:                      返回 LEFT_RIGHT_ROTATE(node)
    如果 BF < -1:
        如果 右高(node.right) <= 0: 返回 LEFT_ROTATE(node)
        否则:                      返回 RIGHT_LEFT_ROTATE(node)

    返回 node
```

## AVL vs 红黑树

| 特性 | AVL | 红黑树 |
|------|-----|--------|
| 平衡度 | **严格** (高度差 ≤ 1) | **近似** (≤ 2×) |
| 查找速度 | 略快 (更矮) | 略慢 |
| 插入/删除开销 | 旋转较多 (O(log n) 次) | 旋转较少 (最多 3 次) |
| 适用场景 | 读多写少 (数据库索引) | 读写均衡 (通用) |

## 相关文档

- [二叉树基础](二叉树基础.md) — 二叉树遍历与属性
- [二叉搜索树](二叉搜索树.md) — AVL 的底层是 BST
- [红黑树](红黑树.md) — 更常用的近似平衡树

## 来源

- CLRS 第 13 章 (练习)
- Sedgewick & Wayne, *Algorithms*, 第 3.3 节
- Adelson-Velsky & Landis, "An algorithm for the organization of information", 1962
