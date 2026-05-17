---
tags: [c++, stl, container, algorithm]
type: entity
---

> STL（Standard Template Library）是 C++ 标准库的核心组成部分，由容器（Containers）、算法（Algorithms）和迭代器（Iterators）三大组件构成。由 Alexander Stepanov 设计，1994 年纳入 C++ 标准。

## 容器分类

### 序列容器

```cpp
#include <vector>
#include <deque>
#include <list>
#include <forward_list>
#include <array>

std::vector<int> vec = {1, 2, 3};        // 动态数组，随机访问
std::deque<int> deq = {1, 2, 3};         // 双端队列，头尾插入快
std::list<int> lst = {1, 2, 3};          // 双向链表
std::forward_list<int> flst = {1, 2, 3}; // 单向链表（C++11）
std::array<int, 3> arr = {1, 2, 3};      // 固定大小数组（C++11）
```

### 关联容器（有序，红黑树）

```cpp
#include <set>
#include <map>
#include <multiset>
#include <multimap>

std::set<int> s = {3, 1, 2};              // 有序集合（元素唯一，O(log n)）
std::map<std::string, int> m = {{"a", 1}, {"b", 2}};  // 有序键值对
std::multiset<int> ms = {1, 1, 2};        // 允许重复
std::multimap<std::string, int> mm;        // 允许键重复
```

### 无序容器（哈希表，C++11）

```cpp
#include <unordered_set>
#include <unordered_map>

std::unordered_set<int> us = {3, 1, 2};            // O(1) 平均查找
std::unordered_map<std::string, int> um;            // O(1) 平均查找
um["hello"] = 42;
```

## 容器对比

| 容器 | 插入 | 删除 | 查找 | 随机访问 | 内存 |
|------|------|------|------|----------|------|
| `vector` | 尾 O(1)，中间 O(n) | 尾 O(1)，中间 O(n) | O(n) | **O(1)** | 连续 |
| `deque` | 头尾 O(1) | 头尾 O(1) | O(n) | O(1) | 分段连续 |
| `list` | O(1)（已知位置） | O(1) | O(n) | 不支持 | 节点链式 |
| `forward_list` | O(1)（头插入） | O(1)（头删除） | O(n) | 不支持 | 单向链式 |
| `set`/`map` | O(log n) | O(log n) | **O(log n)** | 不支持 | 红黑树 |
| `unordered_set/map` | **O(1)** 平均 | **O(1)** 平均 | **O(1)** 平均 | 不支持 | 哈希表 + 桶 |
| `array` | 不支持 | 不支持 | O(n) | **O(1)** | 栈上连续 |

## vector 深入

### 容量与大小

```cpp
std::vector<int> v;
v.reserve(100);                // 预留容量（capacity=100, size=0）
v.push_back(1);                // size=1, capacity=100
v.shrink_to_fit();             // 请求缩减 capacity 到 size（C++11，非强制的）

for (int i = 0; i < 10; ++i)
    v.push_back(i);
// size=10, capacity 可能大于 10
```

| 函数 | 说明 |
|------|------|
| `size()` | 当前元素数量 |
| `capacity()` | 当前分配的内存可容纳的元素数量 |
| `reserve(n)` | 预留至少 n 个元素的空间 |
| `resize(n)` | 调整大小为 n（多余元素默认构造，不足则删除） |
| `shrink_to_fit()` | 请求缩减 capacity 到 size |

### 增长因子

```cpp
std::vector<int> v;
for (int i = 0; i < 100; ++i) {
    std::cout << "size=" << v.size() << " capacity=" << v.capacity() << "\n";
    v.push_back(i);
}
```

不同实现的增长因子：GCC libstdc++ 为 **2** 倍，LLVM libc++ 为 **2** 倍，MSVC STL 为 **1.5** 倍。

增长过程：`capacity` 不足时 → 分配新内存块 → **移动**（C++11 起，原来是拷贝）所有元素到新块 → 释放旧块。每次 reallocation 使迭代器失效。

**最佳实践**: 如果可以预估元素数量，用 `reserve(n)` 避免多次 reallocation。

## map vs unordered_map

| 特性 | `map`（红黑树） | `unordered_map`（哈希表） |
|------|----------------|--------------------------|
| 元素顺序 | 按键升序 | 无序 |
| 查找复杂度 | O(log n) | O(1) 平均，O(n) 最坏 |
| 插入复杂度 | O(log n) | O(1) 平均 |
| 内存 | 每个节点 ~3 指针 | 数组 + 链表，内存更大 |
| 迭代器稳定性 | 插入不失效 | rehash 时全失效 |
| 需要 | `operator<` | `std::hash` + `operator==` |

```cpp
// map 保持有序
std::map<int, std::string> ordered = {{3, "three"}, {1, "one"}, {2, "two"}};
for (auto& [k, v] : ordered)                // 结构化绑定（C++17）
    std::cout << k << ":" << v << " ";      // 1:one 2:two 3:three

// unordered_map 无序
std::unordered_map<int, std::string> unordered = {{3, "three"}, {1, "one"}, {2, "two"}};
// 顺序不确定
```

## 迭代器

### 迭代器分类

```cpp
#include <iterator>

std::vector<int> v = {1, 2, 3, 4, 5};

// 正向迭代
for (auto it = v.begin(); it != v.end(); ++it)
    *it *= 2;

// 反向迭代
for (auto it = v.rbegin(); it != v.rend(); ++it)
    std::cout << *it << " ";    // 5 4 3 2 1

// 只读迭代（C++11 cbegin/cend）
for (auto it = v.cbegin(); it != v.cend(); ++it)
    ;  // *it = 42; — 编译错误

// 范围 for（C++11，内部使用迭代器）
for (auto& x : v)
    x *= 2;
```

| 迭代器类别 | 能力 | 示例容器 |
|-----------|------|----------|
| **Input** | `++`、`*`（读）、`==`/`!=` | `istream_iterator` |
| **Output** | `++`、`*`（写） | `ostream_iterator` |
| **Forward** | Input + 多次遍历 | `forward_list`、`unordered_set` |
| **Bidirectional** | Forward + `--` | `list`、`set`、`map` |
| **Random Access** | Bidirectional + `+n`、`-n`、`[]`、`<`/`>` | `vector`、`deque`、`array` |

### 迭代器失效规则

| 操作 | `vector` | `deque` | `list`/`set`/`map` |
|------|----------|---------|-------------------|
| 插入 | 插入点之后全失效（realloc 时全部失效） | 除首尾外全失效 | 不失效 |
| 删除 | 删除点之后全失效 | 删除点之后全失效 | 仅被删元素失效 |
| rehash | — | — | 全部失效 |

## 算法

### 常用算法

```cpp
#include <algorithm>
#include <numeric>

std::vector<int> v = {5, 2, 8, 1, 9, 3, 7, 4, 6};

// 排序
std::sort(v.begin(), v.end());                   // [1,2,3,4,5,6,7,8,9]
std::nth_element(v.begin(), v.begin() + 4, v.end());  // 第 5 小的元素到位

// 查找
auto it = std::find(v.begin(), v.end(), 7);      // O(n)
bool exists = std::binary_search(v.begin(), v.end(), 7);  // 需有序，O(log n)

// 变换
std::vector<int> squared(v.size());
std::transform(v.begin(), v.end(), squared.begin(),
               [](int x) { return x * x; });

// 累积
int sum = std::accumulate(v.begin(), v.end(), 0); // 36

// 遍历
std::for_each(v.begin(), v.end(), [](int x) {
    std::cout << x << " ";
});

// 删除-擦除惯用法
auto new_end = std::remove_if(v.begin(), v.end(), [](int x) { return x % 2 == 0; });
v.erase(new_end, v.end());                        // 删除所有偶数

// C++20: std::erase_if 简化
std::erase_if(v, [](int x) { return x % 2 == 0; });
```

### 算法复杂度速查

| 算法 | 复杂度 | 要求 |
|------|--------|------|
| `sort` | O(n log n) | RandomAccessIterator |
| `stable_sort` | O(n log n) | RandomAccessIterator，稳定排序 |
| `partial_sort` | O(n log k) | RandomAccessIterator |
| `nth_element` | O(n) 平均 | RandomAccessIterator |
| `lower_bound`/`upper_bound` | O(log n) | 有序区间 |
| `merge` | O(n) | 两个有序区间 |
| `unique` | O(n) | 相邻重复 |
| `remove_if` | O(n) | ForwardIterator |

## 算法与 Lambda

```cpp
struct Person {
    std::string name;
    int age;
};

std::vector<Person> people = {{"Alice", 30}, {"Bob", 25}, {"Charlie", 35}};

// 按年龄升序
std::sort(people.begin(), people.end(),
          [](const Person& a, const Person& b) {
              return a.age < b.age;
          });

// 查找第一个年龄 > 30 的人
auto it = std::find_if(people.begin(), people.end(),
                       [](const Person& p) { return p.age > 30; });

// 统计年龄 >= 30 的人数
int count = std::count_if(people.begin(), people.end(),
                          [](const Person& p) { return p.age >= 30; });
```

## 来源

* [cppreference.com — Containers library](https://en.cppreference.com/w/cpp/container)
* [cppreference.com — Algorithms library](https://en.cppreference.com/w/cpp/algorithm)
* [cppreference.com — Iterator library](https://en.cppreference.com/w/cpp/iterator)
* Bjarne Stroustrup, *The C++ Programming Language*, 4th Edition
