---
tags: [c++, lambda, closure]
type: entity
---

> Lambda 表达式（C++11）是 C++ 对函数式编程范式的核心支持，允许就地定义匿名函数对象。编译器为每个 Lambda 生成一个**唯一的闭包类型**（Closure Type），是 `operator()` 的匿名类实例。

## Lambda 语法

```
[捕获](参数列表) -> 返回类型 { 函数体 }

[capture](params) -> ret { body }
```

```cpp
// 最简形式
auto greet = [] { std::cout << "Hello\n"; };
greet();                              // 无捕获，无参数，默认返回 void

// 带参数和返回值
auto add = [](int a, int b) -> int { return a + b; };
// 返回类型可自动推导
auto add2 = [](int a, int b) { return a + b; };

// 使用
int result = add(3, 4);               // 7
```

## 捕获模式

### 按值捕获 [=]

```cpp
int x = 10, y = 20;
auto by_value = [=] {                  // 捕获所有外部变量的副本
    return x + y;                      // 只能读，不能写（默认）
};
// by_value 内部存储了 x 和 y 的拷贝（闭包中为 const 成员）
```

### 按引用捕获 [&]

```cpp
int x = 10;
auto by_ref = [&] {                    // 捕获所有外部变量的引用
    x = 42;                            // 可以修改外部变量
};
by_ref();
// x 现在是 42
```

### 混合捕获

```cpp
int a = 1, b = 2, c = 3;
auto mixed = [=, &b] {                 // a/c 按值，b 按引用
    b += a + c;                        // b 可修改
};
```

### 显式捕获（C++11）

```cpp
int x = 42;
auto explicit_capture = [x] {          // 仅捕获 x（按值）
    return x * 2;
};

std::string msg = "Hello";
auto ref_capture = [&msg] {            // 仅捕获 msg（按引用）
    msg += " World";
};
```

### 初始化捕获（Init Capture，C++14）

```cpp
// 捕获表达式的值，支持移动语义
auto move_only = [ptr = std::make_unique<int>(42)] {
    return *ptr;
};
// ptr 是闭包类型的成员，在捕获时通过移动构造创建

// 捕获局部变量的重命名版本
int x = 10;
auto init_capture = [y = x + 1] {      // y 是 x+1 的拷贝
    return y;                          // 11
};
```

### 捕获 *this（C++17）

```cpp
struct Server {
    int value = 42;
    auto getLambda() {
        // C++11/14: 捕获 this 指针
        auto old_way = [this] { return value; };
        // C++17: 捕获 *this 的副本（拷贝整个对象）
        auto safe_way = [*this] { return value; };
        return safe_way;
    }
};
```

> 捕获 `[this]` 捕获的是指针，如果 Lambda 生命周期超过对象，则悬垂指针。捕获 `[*this]` 捕获对象的副本，完全安全。

## mutable Lambda

默认按值捕获的变量在 Lambda 中是 `const` 的，不可修改：

```cpp
int counter = 0;

// auto inc = [=] { return ++counter; };   // 编译错误！counter 是 const

auto inc = [=]() mutable {                  // mutable 允许修改按值捕获的副本
    return ++counter;
};

inc();  // 1（修改的是 Lambda 内部 counter 的副本）
inc();  // 2
// counter 仍然是 0（外部的 counter 未改变）
```

## 泛型 Lambda（C++14）

```cpp
// 编译器生成模板化的 operator()
auto generic = [](auto a, auto b) {
    return a + b;
};

std::cout << generic(3, 4) << "\n";           // 7（int + int）
std::cout << generic(3.14, 2.86) << "\n";     // 6.0（double + double）
std::cout << generic(std::string("Hello "), "World") << "\n"; // Hello World

// 模板参数也可显式
auto pair = [](auto&& a, auto&& b) -> std::pair<decltype(a), decltype(b)> {
    return {std::forward<decltype(a)>(a), std::forward<decltype(b)>(b)};
};
```

## Lambda 与 std::function

```cpp
#include <functional>

// 类型擦除：存储任意可调用对象
std::function<int(int, int)> func;

func = [](int a, int b) { return a + b; };    // Lambda
func = std::plus<int>{};                       // 函数对象
func = [](auto... xs) { return (xs + ...); };  // 泛型 Lambda

// 开销：std::function 有虚函数调用等效开销（类型擦除 + 堆分配可能）
// 性能敏感场景用 auto 或模板参数
```

| 方式 | 存储 | 调用开销 | 适用场景 |
|------|------|----------|----------|
| `auto` | 确切闭包类型 | 零开销内联（可） | 单个固定 Lambda |
| `std::function` | 类型擦除 | 间接调用，不可内联 | 运行时多态回调 |
| 模板参数 `F` | 确切闭包类型 | 零开销内联 | 高阶函数参数 |

```cpp
template <typename F>
void repeat(int n, F fn) {                     // 模板参数：不类型擦除
    for (int i = 0; i < n; ++i) fn(i);
}

void repeat_fn(int n, std::function<void(int)> fn) {  // 类型擦除
    for (int i = 0; i < n; ++i) fn(i);
}

// 性能对比：模板版本通常更快（可内联）
auto square = [](int x) { std::cout << x*x << " "; };
repeat(5, square);                            // auto 推导为 Lambda 闭包类型
repeat_fn(5, square);                          // std::function 类型擦除
```

## Lambda 与函数对象

编译器为每个 Lambda 生成一个唯一的匿名类（闭包类型）：

```cpp
// 源代码
auto add = [](int a, int b) { return a + b; };

// 编译器近似生成
class __lambda_1 {
public:
    inline int operator()(int a, int b) const {
        return a + b;
    }
};
auto add = __lambda_1{};

/////

int x = 10;
auto capture = [x](int y) { return x + y; };

// 编译器近似生成
class __lambda_2 {
    int x;                                   // 捕获的变量变成成员变量
public:
    __lambda_2(int x_) : x(x_) {}
    inline int operator()(int y) const {
        return x + y;
    }
};
auto capture = __lambda_2{x};
```

## 立即调用 Lambda（IILE）

```cpp
const int value = []() -> int {
    // 复杂的初始化逻辑
    int sum = 0;
    for (int i = 0; i < 10; ++i) sum += i * i;
    return sum;
}();   // 立即执行：Lambda 定义后紧跟 ()

// 等价于 const 计算结果，但避免了中间变量污染作用域
```

## 性能分析

* **内联潜力**: 非捕获 Lambda 可转换为普通函数指针，编译器可完全内联
* **捕获开销**: 按值捕获的对象存储在闭包中（栈上），无额外分配
* **std::function 代价**: 每次调用至少有一次间接跳转，小 Lambda 可能触发堆分配（小对象优化可避免）

## 总结示例

```cpp
#include <vector>
#include <algorithm>
#include <iostream>

int main() {
    std::vector<int> v = {1, 2, 3, 4, 5, 6, 7, 8};

    // C++11: Lambda 配合算法
    std::for_each(v.begin(), v.end(), [](int& x) { x *= 2; });

    // C++14: 泛型 Lambda + 初始化捕获
    auto multiplier = [factor = 3](auto&& container) {
        std::for_each(container.begin(), container.end(),
                      [factor](auto& x) { x *= factor; });
    };

    // C++17: Constexpr Lambda
    constexpr auto factorial = [](int n) {
        int result = 1;
        for (int i = 2; i <= n; ++i) result *= i;
        return result;
    };
    static_assert(factorial(5) == 120);      // 编译期计算

    // C++20: 模板 Lambda
    auto lambda = []<typename T>(std::vector<T> v) {
        return v.size();
    };
}
```

## 来源

* [cppreference.com — Lambda expressions](https://en.cppreference.com/w/cpp/language/lambda)
* [cppreference.com — std::function](https://en.cppreference.com/w/cpp/utility/functional/function)
* Herb Sutter, *Lambda Expressions and Closures: C++11 vs C++14*
* [Bartek's coding blog — everything about C++ lambdas](https://www.cppstories.com/2021/lambdas-story/)
