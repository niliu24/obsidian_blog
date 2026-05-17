---
tags: [linux, rcu, kernel-locking, lock-free]
type: entity
aliases:
  - Linux RCU 机制
  - Read-Copy-Update
---

> RCU（Read-Copy-Update）是一种适用于**读多写少**场景的无锁同步机制。读者不需要获取锁，写者通过发布/订阅和宽限期（Grace Period）实现安全更新。RCU 广泛应用于内核的链表、路由表、文件系统缓存等读密集型数据结构。

## 核心思想

```
读者: rcu_read_lock() → 读取数据 → rcu_read_unlock()
写者: 1. 分配新数据结构并复制旧数据
     2. 更新指针 (rcu_assign_pointer)
     3. 等待所有读者完成 (synchronize_rcu / call_rcu)
     4. 释放旧数据
```

## RCU API

| API | 说明 |
|-----|------|
| `rcu_read_lock()` | 进入 RCU 读端临界区（不允许睡眠、阻塞） |
| `rcu_read_unlock()` | 退出 RCU 读端临界区 |
| `rcu_dereference()` | 读取受 RCU 保护的指针 |
| `rcu_assign_pointer()` | 发布更新后的指针 |
| `synchronize_rcu()` | 等待宽限期结束（阻塞，可睡眠） |
| `call_rcu()` | 注册回调，宽限期结束后执行（非阻塞） |
| `rcu_barrier()` | 等待所有已注册的 call_rcu 回调执行完毕 |

## 宽限期 (Grace Period)

宽限期是 RCU 的核心概念：从一个写者更新指针开始，到所有可能看到旧数据的读者都完成 `rcu_read_unlock()` 为止。RCU 保证宽限期结束后释放旧数据是安全的。

```
写者更新指针                    宽限期结束
      │                             │
      ├─── Grace Period ────────────┤
      │                             │
读者:  [rcu_read_lock ... rcu_read_unlock]  ← 可能看到旧数据
时间: ─────────────────────────────────────────────────────►
```

## RCU 示例: 保护只读链表

```c
#include <linux/rcupdate.h>
#include <linux/slab.h>

struct my_data {
    struct list_head list;
    int key;
    int value;
    struct rcu_head rcu_head; // 用于 call_rcu 延迟释放
};

static LIST_HEAD(data_list);

// 读者 — 无需锁
void read_entries(void)
{
    struct my_data *entry;

    rcu_read_lock();
    list_for_each_entry_rcu(entry, &data_list, list) {
        // 读取 entry->key, entry->value
        pr_info("key=%d, value=%d\n", entry->key, entry->value);
    }
    rcu_read_unlock();
}

// 写者 — 插入新节点
void add_entry(int key, int value)
{
    struct my_data *new_entry = kmalloc(sizeof(*new_entry), GFP_KERNEL);
    new_entry->key = key;
    new_entry->value = value;

    rcu_read_lock();      // 保护 list_add_rcu 的内部一致性
    list_add_rcu(&new_entry->list, &data_list);
    rcu_read_unlock();
}

// 写者 — 安全删除节点
static void free_entry_rcu(struct rcu_head *head)
{
    struct my_data *entry = container_of(head, struct my_data, rcu_head);
    kfree(entry);
}

void remove_entry(int key)
{
    struct my_data *entry;

    rcu_read_lock();
    list_for_each_entry_rcu(entry, &data_list, list) {
        if (entry->key == key) {
            list_del_rcu(&entry->list);
            rcu_read_unlock();
            // 等待所有读者退出后释放
            call_rcu(&entry->rcu_head, free_entry_rcu);
            return;
        }
    }
    rcu_read_unlock();
}
```

## 相关文档

- [内核同步](内核同步.md) — 同步机制全景对比
- [spinlock](spinlock.md) — 自旋锁
- [内存屏障](内存屏障.md) — RCU 依赖内存屏障保证顺序

## 相关资料

- Linux 内核源码: `include/linux/rcupdate.h`、`kernel/rcu/`
- [What is RCU, Fundamentally? — LWN.net](https://lwn.net/Articles/262464/)
- `Documentation/RCU/`
