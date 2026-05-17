---
tags: [c++, raii, resource-management]
type: entity
---

> RAII（Resource Acquisition Is Initialization，资源获取即初始化）是 C++ 最核心的资源管理惯用法：将资源的生命周期绑定到对象的生命周期——构造函数获取资源，析构函数释放资源。由 Bjarne Stroustrup 提出，是 C++ 没有 GC 仍能安全管理内存的关键。

## RAII 原则

```cpp
class LockGuard {                    // RAII 封装
    std::mutex& mtx;
public:
    explicit LockGuard(std::mutex& m)
        : mtx(m) { mtx.lock(); }    // 构造函数：获取资源
    ~LockGuard() { mtx.unlock(); }  // 析构函数：释放资源
    // 禁止拷贝
    LockGuard(const LockGuard&) = delete;
    LockGuard& operator=(const LockGuard&) = delete;
};

std::mutex m;
{
    LockGuard guard(m);              // 锁定
    // 临界区...
    if (error) throw std::runtime_error("!");  // 异常安全！guard 析构时仍会 unlock
}                                       // unlock （无论在正常退出还是异常退出）
```

**RAII 的关键优势**:

* **异常安全**：栈展开时析构函数一定被调用
* **无泄漏**：资源生命周期与作用域绑定
* **确定性释放**：离开作用域即释放，比 GC 的不可预测收集更可控

### 标准 RAII 封装

| 资源类型 | C++ 标准封装 |
|----------|-------------|
| 堆内存 | `unique_ptr`、`shared_ptr`、`vector`、`string` |
| 互斥锁 | `lock_guard`、`unique_lock`、`scoped_lock`（C++17） |
| 文件 | `fstream`（析构时自动 close） |
| 线程 | `thread`（析构时 join 或 detach） |
| 套接字 | 无标准封装，推荐自定义 RAII 类 |

## lock_guard / unique_lock / scoped_lock

```cpp
#include <mutex>

std::mutex m1, m2;

// lock_guard: 最简单的 RAII 锁包装，不可手动 unlock
{
    std::lock_guard<std::mutex> lock(m1);
    // 临界区...
}   // 自动 unlock

// unique_lock: 更灵活，支持手动 unlock/lock、延迟锁定、尝试锁定
{
    std::unique_lock<std::mutex> lock(m1, std::defer_lock);  // 延迟锁定
    lock.lock();     // 手动锁定
    // ...
    lock.unlock();   // 提前解锁
    lock.lock();     // 重新锁定
}   // 自动 unlock

// scoped_lock (C++17): 可同时锁定多个互斥锁，避免死锁
{
    std::scoped_lock locks(m1, m2);   // 一个锁搞定两个 mutex
    // 同时持有 m1 和 m2...
}   // 按相反顺序自动 unlock
```

| 特性 | `lock_guard` | `unique_lock` | `scoped_lock` |
|------|-------------|--------------|---------------|
| 构造函数锁定 | 是 | 是（也可延迟） | 是 |
| 手动 unlock | 否 | 是 | 否 |
| 条件变量配合 | 否 | 是 | 否 |
| 多锁死锁避免 | 否 | 否 | 是 |
| 开销 | 最小 | 略大（维护锁定状态） | 同 lock_guard |

## 文件句柄 RAII 封装

```cpp
class FileHandle {
    FILE* fp;
public:
    explicit FileHandle(const char* filename, const char* mode)
        : fp(fopen(filename, mode)) {
        if (!fp) throw std::runtime_error("Failed to open file");
    }

    ~FileHandle() {
        if (fp) {
            fclose(fp);
            fp = nullptr;
        }
    }

    // 禁止拷贝
    FileHandle(const FileHandle&) = delete;
    FileHandle& operator=(const FileHandle&) = delete;

    // 移动支持
    FileHandle(FileHandle&& other) noexcept : fp(other.fp) {
        other.fp = nullptr;
    }
    FileHandle& operator=(FileHandle&& other) noexcept {
        if (this != &other) {
            if (fp) fclose(fp);
            fp = other.fp;
            other.fp = nullptr;
        }
        return *this;
    }

    // 读写操作
    void write(const char* data) {
        fputs(data, fp);
    }

    void close() {
        if (fp) { fclose(fp); fp = nullptr; }
    }
};

// 使用
{
    FileHandle fh("log.txt", "w");
    fh.write("Hello, RAII\n");
    // 自动 close
}
```

## Rule of Five（五法则）

C++11 引入移动语义后，如果需要自定义析构函数、拷贝构造函数或拷贝赋值运算符中的**任何一个**，通常需要自定义**全部五个**：

```cpp
class StringBuffer {
    char* data;
    size_t size;
public:
    // 1. 析构函数
    ~StringBuffer() { delete[] data; }

    // 2. 拷贝构造函数
    StringBuffer(const StringBuffer& other)
        : data(new char[other.size]), size(other.size) {
        std::copy(other.data, other.data + size, data);
    }

    // 3. 拷贝赋值运算符
    StringBuffer& operator=(const StringBuffer& other) {
        if (this != &other) {               // 自赋值检查
            auto* newData = new char[other.size];  // 先分配再释放，异常安全
            delete[] data;
            data = newData;
            size = other.size;
            std::copy(other.data, other.data + size, data);
        }
        return *this;
    }

    // 4. 移动构造函数
    StringBuffer(StringBuffer&& other) noexcept
        : data(other.data), size(other.size) {
        other.data = nullptr;
        other.size = 0;
    }

    // 5. 移动赋值运算符
    StringBuffer& operator=(StringBuffer&& other) noexcept {
        if (this != &other) {
            delete[] data;
            data = other.data;
            size = other.size;
            other.data = nullptr;
            other.size = 0;
        }
        return *this;
    }
};
```

### 拷贝并交换惯用法（Copy-and-Swap）

```cpp
class StringBuffer {
    // ... 构造、析构、拷贝构造、移动构造同上
    friend void swap(StringBuffer& a, StringBuffer& b) noexcept {
        using std::swap;
        swap(a.data, b.data);
        swap(a.size, b.size);
    }
    // 统一赋值运算符
    StringBuffer& operator=(StringBuffer other) noexcept {  // pass by value!
        swap(*this, other);   // 交换*this 与临时对象
        return *this;         // 临时对象析构，释放旧资源
    }
    // 一个 operator= 同时处理拷贝和移动 !
};
```

## Rule of Zero（零法则）

最佳实践：尽量使用标准库的 RAII 类型（`vector`、`string`、`unique_ptr` 等），让它们管理资源，这样你的类就不需要自定义任何特殊成员函数：

```cpp
class Person {                          // Rule of Zero
    std::string name;                   // string 自己管理内存
    std::vector<int> scores;            // vector 自己管理内存
    DatabaseConnection conn;             // 自定义 RAII 类管理连接
public:
    Person(const std::string& n) : name(n) {}
    // 编译器生成的析构、拷贝、移动都正确！
};
```

**Rule of Zero > Rule of Five**：如果所有成员都是 RAII 类型，编译器生成的默认特殊成员函数就是正确的。

## 来源

* [cppreference.com — RAII](https://en.cppreference.com/w/cpp/language/raii)
* [cppreference.com — lock_guard](https://en.cppreference.com/w/cpp/thread/lock_guard)
* [cppreference.com — Rule of Three/Five/Zero](https://en.cppreference.com/w/cpp/language/rule_of_three)
* Herb Sutter, *Exceptional C++*
* Bjarne Stroustrup, *The C++ Programming Language*, 4th Edition
