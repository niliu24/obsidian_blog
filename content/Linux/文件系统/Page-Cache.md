---
tags: [linux, page-cache]
type: entity
---

> Page Cache（页缓存）是 Linux 内核介于 VFS 与块设备之间的缓存层。它以物理内存页（通常 4 KB）为单位缓存磁盘上的文件数据，使重复读取操作在内存中完成、避免慢速磁盘 I/O，同时也暂存写入数据（脏页）并在后台批量写回磁盘。Page Cache 是文件 I/O 性能的基石，也是 Linux 内核内存管理的核心组成部分之一。

## 核心数据结构：address_space

VFS 通过 `address_space` 结构体将每个 inode 关联到一个 Page Cache 实例。该结构体定义在 `include/linux/fs.h` 中：

```c
struct address_space {
    struct inode            *host;          /* 关联的 inode */
    struct xarray           i_pages;        /* 页缓存基数树（原 radix tree） */
    atomic_t                i_mmap_writable; /* 可写 mmap 计数 */
    struct rb_root_cached   i_mmap;         /* 所有 mmap 映射的 vma 红黑树 */
    const struct address_space_operations *a_ops; /* 页面操作函数表 */
    unsigned long           nrpages;        /* 当前缓存的页面数 */
    unsigned long           writeback_index; /* 写回起始位置 */
    struct list_head        private_list;   /* 文件系统私有链表 */
};
```

关键字段解释：

| 字段 | 作用 |
|------|------|
| `host` | 指向拥有该 Page Cache 的 inode（每个 inode 最多一个 address_space） |
| `i_pages` | `struct xarray`，用于快速查找文件偏移对应的缓存页（内核 4.20+ 以 xarray 替代 radix tree） |
| `i_mmap` | 红黑树索引该文件被 mmap 到的所有虚拟内存区间 |
| `a_ops` | 定义如何读写页、是否将页标记为脏 |
| `nrpages` | 当前缓存中存活的页面数量 |

### address_space_operations

`a_ops` 定义了具体文件系统如何操作缓存页：

```c
struct address_space_operations {
    int (*readpage)(struct file *, struct page *);
    int (*writepage)(struct page *, struct writeback_control *);
    int (*readpages)(struct file *, struct address_space *, struct list_head *, unsigned);
    void (*dirty_folio)(struct address_space *, struct folio *);
    int (*write_begin)(struct file *, struct address_space *, loff_t, unsigned, struct page **, void **);
    int (*write_end)(struct file *, struct address_space *, loff_t, unsigned, unsigned, struct page *, void *);
    int (*releasepage)(struct address_space *, struct page *, gfp_t);
    int (*migratepage)(struct address_space *, struct page *, struct page *);
    int (*direct_IO)(struct kiocb *, struct iov_iter *);
};
```

## 缓冲 I/O 流程

读请求的默认路径（不带 `O_DIRECT`）：

```
用户态: read(fd, buf, 4096)
    │
    ▼
内核: sys_read() → vfs_read() → file->f_op->read_iter()
    │
    ▼
      generic_file_read_iter()  (通用文件读)
    │
    ▼
      filemap_read() → filemap_get_pages()
    │       │
    │       ├── filemap_get_read_batch()  ← 在 i_pages (xarray) 中查找页面
    │       │     │
    │       │     ├── **命中** → 直接使用缓存中的页面，调用 mark_page_accessed()
    │       │     │
    │       │     └── **未命中** (page fault) →
    │       │            │
    │       │            ├── page_cache_alloc()  — 分配一个新页
    │       │            ├── add_to_page_cache_lru() — 加入 xarray 和 LRU 链表
    │       │            └── a_ops->readpage()  — 调用具体文件系统从磁盘读入
    │       │                   │
    │       │                   └── submit_bio() → 通用块层 → 设备驱动 → 磁盘
    │       ▼
    └── copy_page_to_iter()  — 将内核页中的数据拷贝到用户缓冲区
```

**流程图简化版：**

```
read(fd, buf, N)
    ↓
page in cache? ──yes──→ copy_to_user(buf, page) → 返回
    │no
    ↓
alloc page → add to xarray → submit_bio(disk read)
    ↓
disk IRQ → unlock page → copy_to_user(buf, page) → 返回
```

## 写操作与脏页

写操作同样先修改缓存页，再异步写回磁盘：

```
用户态: write(fd, buf, 4096)
    │
    ▼
内核: sys_write() → vfs_write() → file->f_op->write_iter()
    │
    ▼
      generic_perform_write()
    │   │
    │   ├── a_ops->write_begin()
    │   │       │  — 确保目标页在 cache 中（不在则先读入）
    │   │       └— 返回可写的 page 指针
    │   │
    │   ├── copy_from_user(page, buf) — 数据写入内核页（此时页变为 **脏页**）
    │   │
    │   └── a_ops->write_end()
    │           — 释放锁，标记页面为脏 (SetPageDirty)
    ↓
返回用户态（数据还在内存中，并未落盘）
```

### 页的四种状态

| 状态 | 说明 |
|------|------|
| **Uptodate** | 页面内容与磁盘一致（已被读取） |
| **Dirty** | 页面内容已被修改，需要写回磁盘 |
| **Writeback** | 页面正在被写回磁盘（I/O 进行中） |
| **Locked** | 页面被加锁，禁止其他并发访问 |

## 写回机制 (Writeback)

脏页不会一直留在内存中，内核通过以下机制确保最终写回：

### 触发条件

1. **周期性写回** — `flusher` 线程（原 `pdflush`）周期性唤醒
   * `dirty_writeback_interval`（默认 5 秒）
   * `dirty_expire_interval`（默认 30 秒，脏页超过此时间的必须写回）
2. **内存压力** — `kswapd` 回收内存页时，将脏页加入写回队列
3. **显式调用** — `sync()` / `fsync()` / `fdatasync()` 等系统调用
4. **脏页比例超限**：
   * `dirty_background_ratio`（默认 10%）— 后台 flusher 开始写回
   * `dirty_ratio`（默认 20%）— 进程本身进入同步写回（阻塞）

### flusher 线程框架

```
flusher threads (每个 backing-dev 信息一个)
    │  per-BDI flusher:  /sys/class/bdi/<bdi>/...
    │
    ├── wb_workfn() — 工作队列回调
    │   │
    │   ├── wb_check_background_flush() — 后台比例触发
    │   ├── wb_check_old_data_flush()   — 超时过期
    │   └── wb_writeback() — 核心写回函数
    │       │
    │       ├── walk dirty inode list
    │       ├── for each dirty page: a_ops->writepage()
    │       └── submit_bio() → 块设备
```

### sync / fsync / fdatasync 对比

| 系统调用 | 数据落盘 | 元数据落盘 | 影响范围 | 备注 |
|----------|----------|------------|----------|------|
| `sync()` | 是 | 是 | **所有**文件 | 全局同步，写 `/proc/sys/vm/drop_caches` 时常见 |
| `fsync(fd)` | 是 | 是 | 指定文件的**全部**元数据 | 完整同步，代价高 |
| `fdatasync(fd)` | 是 | 仅文件大小等关键元数据（`mtime` 等可选） | 指定文件的**关键**元数据 | 更快，如需后续文件名不变则传 `st_mtime` |

**代码示例：sync 类型对比**

```c
#include <unistd.h>
#include <fcntl.h>
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <errno.h>

int main(int argc, char *argv[])
{
    const char *filepath = "/tmp/test_sync.txt";
    const char *data = "Hello, Page Cache! 数据写入了缓存，现在尝试落盘。\n";
    int fd;
    ssize_t ret;

    /* 打开文件（带 O_CREAT 和 O_TRUNC） */
    fd = open(filepath, O_WRONLY | O_CREAT | O_TRUNC, 0644);
    if (fd < 0) {
        perror("open");
        exit(EXIT_FAILURE);
    }

    /* 写入数据 —— 此时仅在 Page Cache 中 */
    ret = write(fd, data, strlen(data));
    if (ret < 0) {
        perror("write");
        close(fd);
        exit(EXIT_FAILURE);
    }
    printf("write: %zd bytes written (still in page cache)\n", ret);

    /* === 方式一：fsync — 同步数据和全部元数据 === */
    if (fsync(fd) == 0) {
        printf("fsync: data + all metadata flushed to disk\n");
    }

    /* 重新写入一些新数据用于演示 fdatasync */
    lseek(fd, 0, SEEK_END);
    ret = write(fd, "more data for fdatasync\n", 23);
    if (ret < 0) {
        perror("write2");
        close(fd);
        exit(EXIT_FAILURE);
    }

    /* === 方式二：fdatasync — 同步数据 + 关键元数据（如文件大小） === */
    if (fdatasync(fd) == 0) {
        printf("fdatasync: data + critical metadata flushed\n");
    }

    close(fd);

    /* === 方式三：sync — 全局同步所有文件到磁盘 === */
    sync();
    printf("sync: all dirty pages across the system flushed\n");

    return 0;
}
```

编译及运行验证：

```bash
gcc -o sync_demo sync_demo.c
./sync_demo
# 输出类似:
# write: 57 bytes written (still in page cache)
# fsync: data + all metadata flushed to disk
# fdatasync: data + critical metadata flushed
# sync: all dirty pages across the system flushed
```

## Direct I/O (O_DIRECT)

当打开文件时指定 `O_DIRECT` 标志，数据在内核态和用户态之间**直接传输**，完全绕过 Page Cache。

| 特性 | 缓冲 I/O（默认） | Direct I/O (O_DIRECT) |
|------|------------------|----------------------|
| 数据路径 | 用户 buf ↔ Page Cache ↔ 磁盘 | 用户 buf ↔ 磁盘 |
| CPU / 内存占用 | 一次额外拷贝，但可缓存重用 | 减少内存拷贝，但无缓存 |
| 对齐要求 | 无 | 缓冲区、偏移、大小需对齐到块大小（通常 512 B） |
| 适合场景 | 常规文件操作、小文件 | 数据库（自管理缓存）、大文件顺序读写 |
| 缓存一致性 | 内核自动维护 | 用户程序自行管理 |

### O_DIRECT 使用示例

```c
#include <unistd.h>
#include <fcntl.h>
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <errno.h>

int main(void)
{
    const char *filepath = "/tmp/direct_io_test.bin";
    char *buf;
    int fd;
    long blocksize;

    /* 获取块设备块大小（通常 512 或 4096） */
    blocksize = 4096; /* 也可用 fstat 或 BLKSSZGET ioctl 获取 */

    /* 分配对齐的内存：O_DIRECT 要求 buffer 地址对齐到块大小 */
    if (posix_memalign((void **)&buf, blocksize, blocksize) != 0) {
        perror("posix_memalign");
        exit(EXIT_FAILURE);
    }
    memset(buf, 0xAB, blocksize);

    /* 带 O_DIRECT 打开文件 */
    fd = open(filepath, O_WRONLY | O_CREAT | O_DIRECT | O_TRUNC, 0644);
    if (fd < 0) {
        perror("open with O_DIRECT");
        free(buf);
        exit(EXIT_FAILURE);
    }

    /* 写入：偏移、大小、buf 地址都必须是块大小的整数倍 */
    ssize_t ret = write(fd, buf, blocksize);
    if (ret < 0) {
        perror("O_DIRECT write");
        close(fd);
        free(buf);
        exit(EXIT_FAILURE);
    }
    printf("O_DIRECT write: wrote %zd bytes (no page cache)\n", ret);

    close(fd);
    free(buf);

    /* 验证：确认文件内容 */
    printf("file size: ");
    fflush(stdout);
    int r = system("ls -l /tmp/direct_io_test.bin | awk '{print $5}'");

    return 0;
}
```

编译运行：

```bash
gcc -o direct_io_demo direct_io_demo.c
./direct_io_demo
# O_DIRECT write: wrote 4096 bytes (no page cache)
# file size: 4096
```

## 内存映射 I/O (mmap)

`mmap()` 将文件的一段区域直接映射到进程的虚拟地址空间，进程通过指针读写即可完成文件 I/O，无需 `read()`/`write()` 系统调用。

```
mmap(file, offset, length)
    │
    ▼
do_mmap() → 创建 VMA (vm_area_struct)
    │   vma->vm_file = file
    │   vma->vm_ops = &generic_file_vm_ops
    ▼
首次访问映射地址 → 缺页异常
    │
    ▼
do_linear_fault() → filemap_fault()
    │
    ├── 在 address_space->i_pages 中查找
    ├── 未命中 → 分配新页 → a_ops->readpage() 从磁盘读入
    ├── 添加到页面缓存（和其他 buffered I/O 共享同一套缓存）
    └── 插入进程页表
```

mmap 与 Page Cache 的关系：

| 方面 | 说明 |
|------|------|
| 缓存共享 | mmap 和 read()/write() 共享同一份 Page Cache 中的页面 |
| 脏页处理 | 通过 `msync()` 将映射区脏页写回；`munmap()` 不保证落盘 |
| 性能优势 | 省去 `read()`/`write()` 的系统调用开销和内核↔用户间数据拷贝 |
| 写入 | `MAP_SHARED` 模式下写映射区直接修改页面（标记为脏），对端可见 |
| 限制 | 文件大小不能超过地址空间；映射粒度受 PAGE_SIZE 约束 |

## 典型监控命令

```bash
# 查看全局页缓存大小
grep "^Cached" /proc/meminfo

# 查看每个文件在 Page Cache 中的缓存情况 (内核 4.15+)
# 需要安装 linux-tools
sudo fincore /var/log/syslog

# 查看写回参数
sysctl vm.dirty_background_ratio
sysctl vm.dirty_ratio
sysctl vm.dirty_writeback_interval
sysctl vm.dirty_expire_interval

# 查看 per-BDI 脏页统计
cat /sys/class/bdi/*/dirty_bytes 2>/dev/null

# 使用 /proc/meminfo 查看脏页量
grep -E "^(Dirty|Writeback|WritebackTmp)" /proc/meminfo
```

## 相关资料

* Linux 内核源码: `mm/filemap.c`、`mm/page-writeback.c`、`fs/fs-writeback.c`
* `include/linux/fs.h` — `address_space` 结构体定义
* `include/linux/pagemap.h` — Page Cache 相关内联函数
* [Linux Kernel Documentation: Page Cache](https://www.kernel.org/doc/html/latest/filesystems/caching/page-cache.html)
* 《Linux Kernel Development》（第 3 版）第 15 章（Page Cache and Page Writeback）
* 《Understanding the Linux Kernel》（第 3 版）第 15 章（Page Cache）
* `man 2 sync`、`man 2 fsync`、`man 2 fdatasync`
* `man 2 mmap`
* `man 2 open`（见 `O_DIRECT` 标志说明）
