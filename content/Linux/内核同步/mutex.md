---
tags: [linux, mutex, kernel-locking]
type: entity
aliases:
  - Linux 互斥锁
  - mutex_lock
---

> Mutex（互斥锁）是 Linux 内核中进程上下文的标准睡眠锁。无法获取锁时，进程进入睡眠状态而非自旋，适合保护较长的临界区。

## 基本 API

```c
#include <linux/mutex.h>

DEFINE_MUTEX(my_mutex);  // 静态定义

// 使用
mutex_lock(&my_mutex);   // 获取锁，可能睡眠
// ... 临界区 ...
mutex_unlock(&my_mutex);

// 非阻塞尝试
if (mutex_trylock(&my_mutex)) { // 立即返回
    // ... 获取成功 ...
    mutex_unlock(&my_mutex);
} else {
    // ... 获取失败 ...
}
```

## Mutex 特性

| 特性 | 说明 |
|------|------|
| 所有者追踪 | `mutex_lock()` 记录当前所有者，用于调试 |
| 优先级继承 | 防止优先级反转 |
| 可递归 | ❌ 同一线程不能重复获取同一 mutex |
| 在中断上下文使用 | ❌ 会触发 `might_sleep()` 警告 |
| 约束 | 只能在进程上下文使用 |

## Mutex 在进程上下文示例

```c
#include <linux/mutex.h>
#include <linux/slab.h>
#include <linux/fs.h>
#include <linux/uaccess.h>

static DEFINE_MUTEX(device_mutex);
static char *device_buffer;
static size_t buffer_size = 1024;

ssize_t device_write(struct file *file, const char __user *buf,
                     size_t count, loff_t *offset)
{
    ssize_t ret;

    if (mutex_lock_interruptible(&device_mutex))  // 可被信号打断
        return -ERESTARTSYS;

    if (count > buffer_size) {
        ret = -ENOSPC;
        goto out;
    }

    if (copy_from_user(device_buffer, buf, count)) {
        ret = -EFAULT;
        goto out;
    }

    *offset = count;
    ret = count;

out:
    mutex_unlock(&device_mutex);
    return ret;
}
```

## 相关文档

- [内核同步](内核同步.md) — 同步机制全景对比
- [spinlock](spinlock.md) — 自旋锁，适用于短临界区
- [信号量](信号量.md) — 计数信号量

## 相关资料

- Linux 内核源码: `include/linux/mutex.h`、`kernel/locking/mutex.c`
- `Documentation/locking/mutex-design.rst`
