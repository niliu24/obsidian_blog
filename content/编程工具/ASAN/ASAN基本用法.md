---
tags:
  - asan
  - address-sanitizer
  - debugging
  - memory
type: note
---

**AddressSanitizer（ASan）** 是 Google 开发的快速内存错误检测工具，集成在 GCC（4.8+）和 Clang（3.1+）中，可检测堆/栈/全局变量的内存越界、use-after-free、double-free 等错误，运行时开销约 2x，远低于 Valgrind。

## 工作原理

ASan 在编译时对内存访问指令插入检查代码，运行时将程序虚拟地址空间分为两块：

* **主内存（Main Memory）**：正常使用
* **影子内存（Shadow Memory）**：记录主内存每个字节的可访问状态

主内存每 8 字节对应影子内存 1 字节。影子值 0 表示对应 8 字节全部可访问，1-7 表示部分可访问，负数表示不可访问（如红区）。利用简单的映射公式 `Shadow = (Addr >> 3) + Offset` 即可快速查表。

同时 ASan 在栈变量和堆分配块周围插入 **红区（Redzone）**（已标记为不可访问的影子内存），一旦越界读写立即触发报告。红区不会被程序实际使用，仅在影子内存中标记为中毒（poisoned）。

## 安装

ASan 随编译器一起提供，通常无需额外安装。确认编译器支持：

```bash
gcc --version        # GCC >= 4.8
clang --version      # Clang >= 3.1
```

如果动态链接 `libasan.so`，GCC 可能需要安装：

```bash
# Ubuntu/Debian
sudo apt install libasan8

# CentOS/RHEL
sudo yum install libasan
```

优先用静态链接（`-static-libasan`），避免目标环境缺少动态库。

## 编译与运行

```bash
# 编译时加上 -fsanitize=address
gcc -fsanitize=address -g -O0 -o my_program my_program.c

# 直接运行
./my_program
```

* `-fsanitize=address`：启用 ASan
* `-g`：生成调试符号，报告中展示源码行列号
* `-O0` 或 `-O1`：建议较低优化级别以保持准确的行号映射。也可用 `-Og`（GCC）获取调试友好的基础优化
* 不要同时使用 `-fsanitize=address` 和 `-fsanitize=thread`（互斥）

## 检测类型

| 错误类型 | 说明 |
|---------|------|
| heap-buffer-overflow | 堆缓冲区越界读写 |
| stack-buffer-overflow | 栈缓冲区越界读写 |
| global-buffer-overflow | 全局缓冲区越界 |
| heap-use-after-free | 堆内存释放后使用 |
| stack-use-after-return | 函数返回后使用局部变量地址 |
| stack-use-after-scope | 离开作用域后使用变量 |
| double-free | 重复释放 |
| alloc-dealloc-mismatch | `new[]` 与 `delete` 不匹配 |
| memcpy-param-overlap | `memcpy` 的源和目标重叠 |
| initialization-order-fiasco | 跨翻译单元全局变量初始化顺序问题 |
| memory-leaks | 内存泄漏（LeakSanitizer，需启用） |

## 示例：常见内存错误

### 堆越界

```c
#include <stdlib.h>
int main() {
    int *p = (int *)malloc(sizeof(int) * 5);  // 分配 5 个 int
    p[5] = 42;                                // 越界写入
    free(p);
    return 0;
}
```

ASan 报错：

```
==12345==ERROR: AddressSanitizer: heap-buffer-overflow on address 0x...
WRITE of size 4 at 0x... thread T0
    #0 0x... in main heap_overflow.c:4
...
```

红区位于 `p[5]` 位置，写入被精准拦截。

### Use-after-free

```c
#include <stdlib.h>
int main() {
    int *p = (int *)malloc(sizeof(int) * 100);
    free(p);
    p[0] = 42;   // 释放后又使用
    return 0;
}
```

ASan 报错：

```
==12345==ERROR: AddressSanitizer: heap-use-after-free on address 0x...
WRITE of size 4 at 0x... thread T0
    #0 0x... in main uaf.c:5
```

`free` 后堆区域在影子内存中被标记为不可访问，再次访问立即检测到。

ASan 的堆内存默认放入隔离区（quarantine zone），不会立即归还给操作系统，因此 use-after-free 检测窗口比常规 malloc 长得多。

### 栈越界

```c
int main() {
    int arr[10];
    arr[10] = 42;   // 越界写入
    return 0;
}
```

栈变量之间插入红区，越界立即触发。

## ASAN_OPTIONS

通过环境变量 `ASAN_OPTIONS` 控制行为，多个选项用 `:` 分隔：

```bash
ASAN_OPTIONS="detect_leaks=1:abort_on_error=1:log_path=asan.log" ./my_program
```

| 选项 | 默认值 | 说明 |
|------|--------|------|
| `detect_leaks` | 1 (Linux) | 启用 LeakSanitizer（退出时检测内存泄漏） |
| `abort_on_error` | 0 | 检测到错误时调用 `abort()`（对获取 core dump 有用） |
| `halt_on_error` | 0 | 检测到首个错误后退出，不继续执行 |
| `log_path` | 空 | 将报告写入文件而不是 stderr（自动追加 PID） |
| `verbosity` | 0 | 日志详细级别（1 或 2） |
| `symbolize` | 1 | 符号化栈回溯 |
| `malloc_context_size` | 30 | 记录 malloc/free 调用栈的最大帧数 |
| `alloc_dealloc_mismatch` | 1 | 是否检测分配/释放不匹配 |
| `detect_stack_use_after_return` | 0 | 是否检测栈返回后使用。运行时开销较大（需 `-fsanitize-address-use-after-return=always` 编译） |
| `quarantine_size_mb` | 256 | 堆隔离区最大大小（MB），越大检测窗口越长 |
| `print_stats` | 0 | 退出时输出 ASan 内存使用统计 |

常用组合：

```bash
# 调试：输出到文件，首次错误即退出
ASAN_OPTIONS="log_path=asan.log:halt_on_error=1:abort_on_error=1" ./my_program

# 泄漏排查：输出详细符号化信息
ASAN_OPTIONS="detect_leaks=1:verbosity=1" ./my_program

# 性能测试：关掉隔离区
ASAN_OPTIONS="quarantine_size_mb=64" ./my_program
```

## 禁用特定类型的检测

在源代码中屏蔽特定函数或变量：

```c
// 禁止对某个函数内的内存访问做检查
__attribute__((no_sanitize("address")))
void hot_function(...) { ... }

// 禁止对某个变量的红区保护
__attribute__((no_sanitize("address")))
int large_buffer[1000000];

// GCC 也支持
__attribute__((no_sanitize_address))
void hot_function(...) { ... }
```

## 内存泄漏检测 (LeakSanitizer)

ASan 内置 LeakSanitizer（LSan），程序退出时报告内存泄漏：

```bash
ASAN_OPTIONS="detect_leaks=1" ./my_program
```

报错示例：

```
==12345==ERROR: LeakSanitizer: detected memory leaks
Direct leak of 40 byte(s) in 1 object(s) allocated from:
    #0 0x... in malloc
    #1 0x... in main leak.c:3
```

关闭泄漏检测（仅关注内存破坏类错误时）：

```bash
export ASAN_OPTIONS=detect_leaks=0
```

## 与其他工具配合

```bash
# ASan + GDB：出错时自动中断到调试器，检查现场
ASAN_OPTIONS="abort_on_error=1:halt_on_error=1" gdb --args ./my_program

# ASan + 栈保护：栈溢出双层防御
gcc -fsanitize=address -fstack-protector-strong -g my_program.c

# 从 core dump 中获取 ASan 报告
ASAN_OPTIONS="abort_on_error=1" ./my_program   # 生成 core dump
# 然后用 gdb 分析
```

## 注意事项

* ASan 与 TSan（ThreadSanitizer）不能同时使用，与 MSan（MemorySanitizer）也不能同时使用
* ASan 可和 UBSan（UndefinedBehaviorSanitizer）组合：`-fsanitize=address,undefined`
* 运行时内存开销约为原程序的 2-3 倍，大内存程序需注意资源限制
* 默认不使用栈返回后检测（`detect_stack_use_after_return=0`，有额外开销）
* 汇编手写的内存操作（如加密库的 SIMD 代码）无法被 ASan 检测
* 共享库（.so）也需用 `-fsanitize=address` 编译才能被检测。若主程序开启而 .so 未开启，主程序调用的库函数中的内存错误可能漏检
* 生产环境可考虑用 GWP-ASan（Android/Chromium 就地的 heap-only ASan），开销极低但检测面窄
