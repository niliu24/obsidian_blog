---
tags: [c++, vtable, polymorphism]
type: entity
---

> 虚函数（Virtual Function）是 C++ 实现**动态多态**的核心机制。通过虚函数表（vtable）和虚指针（vptr），在运行时确定调用哪个函数——即**晚绑定**（Late Binding）。

## 虚函数机制

### 基本概念

```cpp
class Base {
public:
    virtual void speak() const {      // virtual 关键字
        std::cout << "Base\n";
    }
    virtual ~Base() = default;        // 基类析构函数应为虚函数
};

class Derived : public Base {
public:
    void speak() const override {     // override（C++11）明确标注覆盖
        std::cout << "Derived\n";
    }
};

void talk(const Base& obj) {
    obj.speak();                      // 动态绑定：运行时决定调用的版本
}

// 使用
Derived d;
talk(d);                              // 输出 "Derived"（而非 "Base"）
Base b;
talk(b);                              // 输出 "Base"
```

### 虚函数表（vtable）与虚指针（vptr）

每个**含有虚函数的类**有一个 vtable（编译期生成的静态数组），存储该类所有虚函数的函数指针。每个**对象**有一个隐藏的 vptr，指向其所属类的 vtable。

```
内存布局（64位系统）:

Base 对象:                Derived 对象:
┌──────────────┐          ┌──────────────┐
│ vptr  ───────┼────┐     │ vptr  ───────┼────┐
│ 其他成员     │    │     │ 其他成员     │    │
└──────────────┘    │     └──────────────┘    │
                    │                         │
Base vtable:        │     Derived vtable:     │
┌──────────────┐    │     ┌──────────────┐    │
│ &Base::speak │◄───┘     │ &Derived::speak │◄─┘
│ &Base::~Base │          │ &Derived::~Base  │
│ (type_info)  │          │ (type_info)      │
└──────────────┘          └──────────────┘
```

**调用过程**（以 `obj.speak()` 为例）:

1. 通过 `obj` 的 vptr 找到所属类的 vtable
2. 在 vtable 中定位 `speak` 对应的槽位
3. 通过函数指针间接调用

### 性能开销

| 开销项 | 说明 |
|--------|------|
| **vtable 查找** | 多一次指针解引用（通常 1-2 条指令） |
| **间接调用** | 无法内联（inline），阻止编译器优化 |
| **难以分支预测** | 间接跳转可能导致分支预测失败 |
| **空间开销** | 每个对象多一个 vptr（8 字节，64 位），每个类多一个 vtable |

在 HPC 场景中，**虚函数调用**的延迟大约是普通函数调用的 2-3 倍。热点路径上应避免虚函数，改用模板或 `if constexpr`。

## 纯虚函数与抽象类

```cpp
class Shape {                          // 抽象类（不能实例化）
public:
    virtual double area() const = 0;   // 纯虚函数
    virtual ~Shape() = default;
};

class Circle : public Shape {
    double r;
public:
    explicit Circle(double radius) : r(radius) {}
    double area() const override {     // 必须实现，否则 Circle 也是抽象类
        return 3.14159 * r * r;
    }
};

// Shape s;                          // 编译错误：不能实例化抽象类
Shape* s = new Circle(5.0);
s->area();                            // 正确：通过指针调用
delete s;
```

## override 与 final（C++11）

```cpp
class Base {
    virtual void f();
    virtual void g() const;
};

class Derived : public Base {
    void f() override;                // 正确：覆盖 Base::f
    // void g() override;             // 编译错误：const 不匹配，不是覆盖
    // void h() override;             // 编译错误：基类无此虚函数
};

class FinalClass final : public Base { // final 类：不可被继承
public:
    void f() override final;           // final 函数：不可再被覆盖
};

// class Derived2 : public FinalClass {};  // 编译错误
```

| 关键字 | 用途 |
|--------|------|
| `override` | 明确标注要覆盖基类虚函数，编译器检查签名匹配 |
| `final` | 禁止派生类进一步覆盖该虚函数，或禁止类被继承 |

## 虚析构函数

```cpp
class Base {
public:
    ~Base() { std::cout << "~Base()\n"; }   // 非虚！
};

class Derived : public Base {
    int* data = new int[100];
public:
    ~Derived() { delete[] data; std::cout << "~Derived()\n"; }
};

Base* p = new Derived();
delete p;         // 未定义行为！Base 析构函数非虚，Derived 析构函数不会被调用
                  // → 内存泄漏！
```

**规则**: 如果类有虚函数，析构函数必须是虚函数。或者，如果类被设计为基类，析构函数应为虚函数。

## RTTI（Run-Time Type Information）

### typeid

```cpp
#include <typeinfo>

Base* b = new Derived();
std::cout << typeid(*b).name();   // 输出 "Derived"（vtable 中存储 type_info）
std::cout << typeid(b).name();    // 输出 "Base*"（指针本身的静态类型）
```

### dynamic_cast

```cpp
Base* b = new Derived();
Derived* d = dynamic_cast<Derived*>(b);  // 向下转型
if (d) {
    // 转型成功：b 确实是 Derived 类型
}

Base base;
// Derived& ref = dynamic_cast<Derived&>(base);  // std::bad_cast 异常
```

| 转换 | 说明 |
|------|------|
| `static_cast<D*>(b)` | 编译期检查，不验证运行时类型 |
| `dynamic_cast<D*>(b)` | 运行时检查 vtable，失败返回 nullptr |
| `dynamic_cast<D&>(b)` | 失败抛出 `std::bad_cast` |

> **性能**: `dynamic_cast` 需要遍历继承链，在深继承中可能较慢。尽量避免在热点路径使用。

## 多继承下的 vtable 布局

```cpp
class A {
public:
    virtual void fa();
    int a;
};

class B {
public:
    virtual void fb();
    int b;
};

class C : public A, public B {
public:
    void fa() override;
    void fb() override;
    void fc();
};
```

多继承下，C 对象包含**两个 vptr**（A 子对象和 B 子对象各一个）：

```
C 对象内存布局:
┌──────────────────┐
│ vptr_A  ─────────┼────→ A vtable 部分（用于 C::fa）
│ a                │
│ vptr_B  ─────────┼────→ B vtable 部分（用于 C::fb, 含 thunk）
│ b                │
│ (C 自身成员)     │
└──────────────────┘

A 兼容 vtable:          B 兼容 vtable:
┌────────────────┐      ┌────────────────┐
│ &C::fa         │      │ thunk to C::fb │ ← 调整 this 指针偏移
│ &A::~A (或C)   │      │ &A::~B (或 thunk) │
└────────────────┘      └────────────────┘
```

**Thunk 机制**: 当通过 B 指针调用 `C::fb()` 时，`this` 指针需要从 B 子对象的起始位置调整到 C 对象整体的起始位置（`this -= sizeof(A)`），这就是 thunk 小段汇编代码做的事情。

## 来源

* [cppreference.com — Virtual functions](https://en.cppreference.com/w/cpp/language/virtual)
* [cppreference.com — dynamic_cast](https://en.cppreference.com/w/cpp/language/dynamic_cast)
* [Itanium C++ ABI — vtable layout](https://itanium-cxx-abi.github.io/cxx-abi/abi.html#vtable-layout)
* [Stanley B. Lippman, *Inside the C++ Object Model*](https://www.informit.com/store/inside-the-c-plus-plus-object-model-9780201834543)
