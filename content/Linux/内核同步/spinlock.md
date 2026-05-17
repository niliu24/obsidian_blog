---
tags: [linux, spinlock, kernel-locking]
type: entity
aliases:
  - Linux 自旋锁
  - spin_lock
---

> 自旋锁（Spinlock）是 Linux 内核中最底层的锁机制。当无法获取锁时，CPU 原地循环（自旋）等待而非睡眠，适合保护极短的临界区。

## 基本 API

```c
#include <linux/spinlock.h>

DEFINE_SPINLOCK(my_lock);  // 静态定义

// 基本使用
spin_lock(&my_lock);       // 自旋等待
// ... 临界区 ...
spin_unlock(&my_lock);

// 在中断上下文中使用时必须禁用本地中断
unsigned long flags;
spin_lock_irqsave(&my_lock, flags);   // 保存当前中断状态 + 关中断 + 获取锁
// ... 临界区 ...
spin_unlock_irqrestore(&my_lock, flags);  // 释放锁 + 恢复中断状态

// spin_lock_irqsave/spin_unlock_irqrestore 是最安全的变体
// 适合在中断处理程序 + 进程上下文混合访问的场景
```

## 变体对比

| API | 关闭本地中断 | 保存标志 | 适用场景 |
|-----|-------------|---------|----------|
| `spin_lock()` | ❌ | ❌ | 只有进程上下文竞争 |
| `spin_lock_irq()` | ✅ | ❌ | 进程 + 中断上下文，但已知中断状态 |
| `spin_lock_irqsave()` | ✅ | ✅ | 进程 + 中断上下文，推荐首选 |
| `spin_lock_bh()` | 关闭软中断 | ❌ | 进程 + 软中断上下文 |

## IRQ 中使用 spinlock 示例

```c
static DEFINE_SPINLOCK(dev_lock);
static struct device_data shared_data;

// 进程上下文
ssize_t dev_write(struct file *file, const char __user *buf,
                  size_t count, loff_t *offset)
{
    unsigned long flags;

    spin_lock_irqsave(&dev_lock, flags);
    // 修改共享数据
    shared_data.value = some_value;
    spin_unlock_irqrestore(&dev_lock, flags);

    return count;
}

// 中断上下文 - 也访问 shared_data
static irqreturn_t dev_irq_handler(int irq, void *dev_id)
{
    unsigned long flags;

    // 注意: 如果确定此 handler 执行时本地中断已关闭，也可用 spin_lock()
    // 但使用 _irqsave 更安全
    spin_lock_irqsave(&dev_lock, flags);
    // 读取/修改 shared_data
    shared_data.counter++;
    spin_unlock_irqrestore(&dev_lock, flags);

    return IRQ_HANDLED;
}
```

## raw_spinlock

`raw_spinlock` 是 spinlock 在 PREEMPT_RT 内核中的不变底层实现。在非 RT 内核中 `spinlock` 就是 `raw_spinlock`，但在 RT 内核中，`spinlock` 可被优先级继承的 `rt_mutex` 替代，而 `raw_spinlock` 始终是自旋锁。中断处理程序中的关键路径必须使用 `raw_spinlock`。

## 相关文档

- [内核同步](内核同步.md) — 同步机制全景对比
- [mutex](mutex.md) — 可睡眠的互斥锁
- [读写锁](读写锁.md) — 读多写少场景的 rwlock
- [原子操作](原子操作.md) — 无锁整数操作

## 相关资料

- Linux 内核源码: `include/linux/spinlock.h`、`kernel/locking/spinlock.c`
- `Documentation/locking/spinlocks.rst`
