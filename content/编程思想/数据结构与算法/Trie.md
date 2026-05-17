---
tags: [data-structure, trie, prefix-tree]
type: entity
aliases:
  - Trie
  - 前缀树
  - 字典树
---

> Trie（前缀树 / 字典树）是一种 N 叉树结构，用于高效存储和检索字符串集合。每个节点代表一个前缀，从根到某个节点的路径对应一个字符串。Trie 的查找时间仅取决于字符串长度，与集合大小**无关**。

## 核心性质

- 根节点为空字符串
- 每条边代表一个字符
- 从根到某个节点的路径对应该节点的前缀
- 节点标记 `is_end` 区分完整单词和前缀

## 结构示意

```
插入 "apple", "app", "or":

     (root)
     /    \
    a      o
    |       \
    p        r(*)
    |
    p(*)
    |
    l
    |
    e(*)

(*) = is_end = true，表示该节点是某个插入单词的结尾
```

## 复杂度

| 操作 | 复杂度 | 说明 |
|------|--------|------|
| 插入 | O(L) | L = 字符串长度 |
| 精确查找 | O(L) | 与集合中已有字符串数量无关 |
| 前缀查找 | O(L + k) | k = 匹配前缀的字符串数 |
| 删除 | O(L) | 需清理不再使用的节点 |
| 全部键遍历 | O(N·L) | 遍历所有存储的字符串 |

## 插入 — 伪代码

```
算法 TRIE_INSERT(root, word):
    node ← root
    对于 word 中的每个字符 ch:
        如果 node.children[ch] == NULL:
            node.children[ch] ← NEW_TRIE_NODE()
        node ← node.children[ch]
    node.is_end ← TRUE
```

## 查找 — 伪代码

```
算法 TRIE_SEARCH(root, word):
    node ← root
    对于 word 中的每个字符 ch:
        如果 node.children[ch] == NULL: 返回 FALSE
        node ← node.children[ch]
    返回 node.is_end  // 必须是完整单词，而非仅前缀

算法 TRIE_STARTS_WITH(root, prefix):
    node ← root
    对于 prefix 中的每个字符 ch:
        如果 node.children[ch] == NULL: 返回 FALSE
        node ← node.children[ch]
    返回 TRUE  // 前缀存在即可
```

## 删除 — 伪代码

```
算法 TRIE_REMOVE(root, word, depth=0):
    如果 depth == WORD_LENGTH:
        如果 root.is_end:
            root.is_end ← FALSE
        返回 root 的所有子节点都为空  // 返回是否可安全删除

    ch ← word[depth]
    如果 root.children[ch] == NULL: 返回 FALSE

    should_delete_child ← TRIE_REMOVE(root.children[ch], word, depth + 1)

    如果 should_delete_child:
        delete root.children[ch]
        root.children[ch] ← NULL
        返回 root 的所有子节点都为空 且 root.is_end == FALSE

    返回 FALSE
```

## 经典应用

| 应用 | 说明 |
|------|------|
| **自动补全 / 搜索建议** | 输入前缀 → 遍历子树收集所有 completions |
| **拼写检查** | word 不在 Trie 中 → 标记为拼写错误 |
| **IP 路由 (最长前缀匹配)** | 将 IP 地址的每一位插入 Trie，查找最深的匹配前缀 |
| **XOR 最大值** | 二进制 Trie：对每个数，尽量走相反的 bit 路径 |
| **单词游戏 (Boggle)** | 在棋盘上搜索时快速判断当前前缀是否为有效单词 |
| **KMP 的 Aho-Corasick** | 多模式同时匹配的 Trie + failure link |

### 自动补全 — 伪代码

```
算法 AUTOCOMPLETE(root, prefix):
    // 1. 定位到前缀尾节点
    node ← root
    对于 prefix 中的每个字符 ch:
        如果 node.children[ch] == NULL: 返回 []
        node ← node.children[ch]

    // 2. 从该节点 DFS 收集所有完整单词
    result ← []
    DFS_COLLECT(node, prefix, result)
    返回 result

算法 DFS_COLLECT(node, prefix, result):
    如果 node.is_end:
        result.append(prefix)
    对于 node.children 中的每个 (ch, child):
        DFS_COLLECT(child, prefix + ch, result)
```

### 异或最大值 — 伪代码

```
算法 MAX_XOR(root, nums):
    max_xor ← 0
    对于 nums 中的每个数 x:
        node ← root
        cur_xor ← 0
        对于 bit 从高位到低位:
            bit ← (x >> b) & 1
            如果 node.children[1 - bit] != NULL:
                cur_xor ← cur_xor | (1 << b)
                node ← node.children[1 - bit]
            否则:
                node ← node.children[bit]
        max_xor ← max(max_xor, cur_xor)
    返回 max_xor
```

## 相关文档

- [二叉树基础](二叉树基础.md) — 二叉树遍历与属性
- [图](图.md) — Aho-Corasick 中的 failure link 类似图的边
- [查找算法](查找算法.md) — Trie 是字符串的另一种查找结构

## 来源

- CLRS 第 15 章 (问题)
- Sedgewick & Wayne, *Algorithms*, 第 5.2 节
