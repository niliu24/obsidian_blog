---
tags: [linux, syscall, mmap, memory-mapping]
type: entity
aliases:
  - mmap 系统调用
  - 内存映射
---

> `mmap` 是 Linux 中最关键的系统调用之一。它将文件或设备的内存直接映射到进程的虚拟地址空间，使进程可以用指针直接访问数据，**绕过了 read/write 的内核缓冲区拷贝**。mmap 不仅是高性能 I/O 的基石，也是共享内存、动态库加载和 malloc 底层实现的核心机制。

## 函数原型

```c
#include <sys/mman.h>

void *mmap(void *addr, size_t length, int prot, int flags,
           int fd, off_t offset);
```

| 参数 | 说明 |
|------|------|
| `addr` | 建议的映射起始地址（通常为 NULL，由内核选择） |
| `length` | 映射的长度（字节），会被向上取整到页大小 |
| `prot` | 内存保护标志（`PROT_READ`, `PROT_WRITE`, `PROT_EXEC`, `PROT_NONE`） |
| `flags` | 映射类型和选项（`MAP_SHARED`, `MAP_PRIVATE`, `MAP_ANONYMOUS` 等） |
| `fd` | 要映射的文件描述符（匿名映射时为 -1） |
| `offset` | 文件中的偏移（必须是页大小的整数倍） |

返回值：成功返回映射区域的起始地址，失败返回 `MAP_FAILED` 并设置 `errno`。

## 核心标志

| 标志              | 说明                                           |
| --------------- | -------------------------------------------- |
| `MAP_SHARED`    | 共享映射：对映射的写入会回写到文件，对其他进程可见                    |
| `MAP_PRIVATE`   | 私有映射 (COW)：写入触发写时拷贝，不影响原文件和其他进程              |
| `MAP_ANONYMOUS` | 匿名映射：不关联文件，fd 应为 -1，分配零初始化内存                 |
| `MAP_FIXED`     | 固定地址映射（需谨慎，会替换已有映射）                          |
| `MAP_POPULATE`  | 预填充页表，减少后续缺页                                 |
| `MAP_LOCKED`    | 锁定映射页面，防止被换出 (类似 mlock)                      |
| `MAP_32BIT`     | 将映射限制在低 2GB 地址空间 (仅 x86_64)，供需要 32 位指针的旧代码使用 |

## 典型使用场景

### 1. 文件映射 I/O

绕过 Page Cache 的双重缓冲，减少一次内存拷贝：

```
传统 read(): 磁盘 → Page Cache → 用户缓冲区 (2 次拷贝)
mmap:        磁盘 → Page Cache ← (映射为) → 用户可直接访问 (1 次拷贝)
```

```c
#include <sys/mman.h>
#include <sys/stat.h>
#include <fcntl.h>
#include <unistd.h>
#include <stdio.h>

int main()
{
    int fd = open("large_file.bin", O_RDONLY);
    if (fd < 0) { perror("open"); return 1; }

    struct stat st;
    fstat(fd, &st);
    size_t file_size = st.st_size;

    // 将整个文件映射到内存
    char *data = mmap(NULL, file_size, PROT_READ, MAP_PRIVATE, fd, 0);
    if (data == MAP_FAILED) {
        perror("mmap");
        return 1;
    }

    // 直接通过指针访问文件内容，无需 read()
    for (size_t i = 0; i < file_size; i++) {
        if (data[i] == '\n')
            printf("Found newline at offset %zu\n", i);
    }

    munmap(data, file_size);
    close(fd);
    return 0;
}
```

### 2. 匿名映射 (malloc 底层)

```c
// 匿名映射 — glibc malloc 对大块内存 (>=128KB) 使用 mmap
void *buf = mmap(NULL, 1024 * 1024, PROT_READ | PROT_WRITE,
                 MAP_PRIVATE | MAP_ANONYMOUS, -1, 0);
if (buf == MAP_FAILED) {
    perror("mmap");
    return 1;
}

// buf 即普通内存，可自由读写
memset(buf, 0, 1024 * 1024);

munmap(buf, 1024 * 1024);
```

### 3. 共享内存 (进程间通信)

```c
// 进程 A 和 B 打开同一个文件，使用 MAP_SHARED 映射
int fd = shm_open("/my_shm", O_CREAT | O_RDWR, 0600);
ftruncate(fd, 4096);

int *shared = mmap(NULL, 4096, PROT_READ | PROT_WRITE,
                   MAP_SHARED, fd, 0);

// 进程 A 写入
shared[0] = 42;

// 进程 B (mmap 同一 shm 文件后):
printf("shared[0] = %d\n", shared[0]); // 读到 42
```

### 4. mmap vs read 性能对比

| 操作 | read/write | mmap |
|------|-----------|------|
| 内存拷贝次数 | 2 次 (Page Cache ↔ 用户缓冲区) | 0-1 次 (直接操作 Page Cache) |
| 系统调用频率 | 每次读写 | 仅 mmap/munmap 时 |
| 随机访问 | 需要 seek + read | 直接指针偏移 |
| 延迟 | 每次系统调用 ~1μs | 缺页时 ~10μs (仅首次)，后续 ~0 |
| 适用数据量 | 小量/流式 | 大文件、随机访问 |

## 驱动端实现 (mmap 文件操作)

设备驱动可以实现 `mmap` 让用户空间直接映射设备内存（如 GPU 显存、DMA 缓冲区）：

```c
#include <linux/mm.h>
#include <linux/dma-mapping.h>

static int mydev_mmap(struct file *file, struct vm_area_struct *vma)
{
    struct mydev *dev = file->private_data;
    unsigned long pfn = page_to_pfn(virt_to_page(dev->dma_buffer));
    size_t size = vma->vm_end - vma->vm_start;

    // 将设备 DMA 缓冲区的物理页映射到用户空间
    if (remap_pfn_range(vma, vma->vm_start, pfn, size, vma->vm_page_prot))
        return -EAGAIN;

    pr_info("mydev: mmap %zu bytes to user\n", size);
    return 0;
}

static struct file_operations mydev_fops = {
    .mmap = mydev_mmap,
    // ...
};
```

## 相关系统调用

| 系统调用 | 用途 |
|----------|------|
| `munmap(addr, length)` | 解除内存映射 |
| `mprotect(addr, length, prot)` | 修改映射区域的保护属性 |
| `msync(addr, length, flags)` | 将 MAP_SHARED 映射的修改同步回文件 |
| `mlock(addr, length)` | 锁定内存页防止换出 |
| `madvise(addr, length, advice)` | 向内核提供使用建议 (预读、DONTNEED 等) |
| `mremap(old, old_size, new_size, flags)` | 调整已有映射的大小 |

## 相关文档

* [系统调用](../系统调用.md) — 系统调用总览
* [页缓存](../../文件系统/Page-Cache.md) — mmap 文件映射依赖 Page Cache
* [缺页处理](../../内存管理/缺页处理.md) — mmap 的惰性物理页分配
* [进程地址空间](../../内存管理/进程地址空间.md) — mmap 在地址空间中的布局
* [ioctl](ioctl.md) — 另一种设备交互方式

## 相关资料

* [Linux man pages — mmap(2)](https://man7.org/linux/man-pages/man2/mmap.2.html)
* [Linux kernel — Memory Management](https://www.kernel.org/doc/html/latest/admin-guide/mm/)
* [The Linux Programming Interface (Kerrisk) — Ch. 49](https://man7.org/tlpi/)
