---
tags: [linux, vfs]
type: entity
---

> VFS（Virtual File System Switch，虚拟文件系统交换层）是 Linux 内核为实现多种文件系统共存而设计的抽象层。它定义了一套标准接口模型，所有具体文件系统（ext4、xfs、btrfs、tmpfs 等）只要实现这些接口，就能无缝接入内核的文件访问框架，用户进程无需关心底层是磁盘还是内存。

## 四个核心对象

VFS 基于四个核心内核对象构建，它们定义在 `include/linux/fs.h` 中：

| 对象 | 内核结构体 | 作用 | 生命周期 |
|------|-----------|------|----------|
| **超级块** | `super_block` | 描述已挂载文件系统的元数据（块大小、最大文件大小、挂载选项等） | 挂载时创建，卸载时销毁 |
| **索引节点** | `inode` | 描述文件元数据（权限、大小、时间戳、数据块位置），**不**包含文件名 | 首次访问时读入内存，回收时释放 |
| **目录项** | `dentry` | 描述文件名与 inode 的映射关系，构造目录树结构；缓存在 dcache 中 | 路径查找时创建，内存压力下回收 |
| **文件** | `file` | 描述一个打开的文件实例（当前读写偏移、访问模式、`file_operations` 指针） | `open()` 时创建，`close()` 时销毁 |

### 超级块 (super_block)

超级块定义一个挂载的文件系统的全局信息：

```c
struct super_block {
    dev_t                   s_dev;              /* 块设备编号 */
    unsigned long           s_blocksize;        /* 块大小（字节） */
    struct file_system_type *s_type;            /* 文件系统类型 */
    const struct super_operations *s_op;        /* 超级块操作函数表 */
    struct list_head        s_inodes;           /* 该文件系统所有 inode 链表 */
    struct list_head        s_dirty;            /* 脏 inode 链表 */
    void                    *s_fs_info;         /* 指向具体文件系统私有数据 */
};
```

### 索引节点 (inode)

inode 存的是**文件的元数据**，不包含文件名（文件名在 dentry 中）：

```c
struct inode {
    umode_t             i_mode;     /* 文件类型和权限 */
    kuid_t              i_uid;      /* 用户 ID */
    kgid_t              i_gid;      /* 组 ID */
    loff_t              i_size;     /* 文件大小（字节） */
    struct timespec64    i_atime;    /* 最后访问时间 */
    struct timespec64    i_mtime;    /* 最后修改时间 */
    struct timespec64    i_ctime;    /* 状态变更时间 */
    loff_t              i_blocks;   /* 已分配的 512 字节块数 */
    struct address_space *i_mapping; /* 关联的 address_space（Page Cache） */
    const struct inode_operations *i_op;
    const struct file_operations  *i_fop; /* 默认文件操作 */
};
```

### 目录项 (dentry)

dentry 是路径名和 inode 之间的桥梁，存储在 dcache 中加速路径查找：

```c
struct dentry {
    struct qstr             d_name;     /* 目录项名称 */
    struct inode            *d_inode;   /* 关联的 inode */
    struct dentry           *d_parent;  /* 父目录项 */
    struct list_head        d_child;    /* 兄弟链表 */
    const struct dentry_operations *d_op;
};
```

### 文件对象 (file)

文件对象表示进程打开的一个文件实例，是 VFS 面向进程的接口：

```c
struct file {
    struct path             f_path;         /* 包含 dentry 和 mount 信息 */
    struct inode            *f_inode;       /* 指向实际 inode */
    const struct file_operations *f_op;     /* 文件操作函数表 */
    loff_t                  f_pos;          /* 当前读写偏移 */
    unsigned int            f_flags;        /* 打开标志（O_RDONLY/O_WRONLY/O_RDWR） */
    fmode_t                 f_mode;         /* 访问模式 */
    void                    *private_data;  /* 具体文件系统私有数据 */
};
```

## 关键操作函数表

### file_operations

每个文件对象关联一个 `file_operations` 实例，定义该文件支持的操作：

```c
struct file_operations {
    loff_t (*llseek)   (struct file *, loff_t, int);
    ssize_t (*read)    (struct file *, char __user *, size_t, loff_t *);
    ssize_t (*write)   (struct file *, const char __user *, size_t, loff_t *);
    ssize_t (*read_iter) (struct kiocb *, struct iov_iter *);
    ssize_t (*write_iter)(struct kiocb *, struct iov_iter *);
    int (*mmap)        (struct file *, struct vm_area_struct *);
    unsigned long (*mmap_supported_flags)(struct file *);
    int (*open)        (struct inode *, struct file *);
    int (*release)     (struct inode *, struct file *);
    int (*fsync)       (struct file *, loff_t, loff_t, int datasync);
    long (*unlocked_ioctl)(struct file *, unsigned int, unsigned long);
    int (*flush)       (struct file *, fl_owner_t id);
    __poll_t (*poll)   (struct file *, struct poll_table_struct *);
};
```

### inode_operations

```c
struct inode_operations {
    struct dentry * (*lookup) (struct inode *, struct dentry *, unsigned int);
    int (*create)   (struct inode *, struct dentry *, umode_t, bool);
    int (*link)     (struct dentry *, struct inode *, struct dentry *);
    int (*unlink)   (struct inode *, struct dentry *);
    int (*mkdir)    (struct inode *, struct dentry *, umode_t);
    int (*rmdir)    (struct inode *, struct dentry *);
    int (*rename)   (struct inode *, struct dentry *, struct inode *, struct dentry *, unsigned int);
    int (*setattr)  (struct dentry *, struct iattr *);
    int (*getattr)  (const struct path *, struct kstat *, u32, unsigned int);
    ssize_t (*listxattr)(struct dentry *, char *, size_t);
};
```

## 路径查找流程

当用户调用 `open("/home/user/file.txt", O_RDONLY)` 时，内核的路径查找流程如下：

```
用户态: fd = open("/home/user/file.txt", O_RDONLY)
    │
    ▼
内核态: sys_open() → do_sys_open() → do_filp_open()
    │
    ▼
path_openat(dfd, "/home/user/file.txt", &op)
    │
    ├── link_path_walk() — 逐分量查找路径
    │   │   │  "/" → 从根 dentry 开始 (root of mount tree)
    │   │   │  "home" → dcache 中查找 → miss → 调用父 inode->lookup() 读目录
    │   │   │  "user" → dcache 中查找 → ...
    │   │   │  "file.txt" → dcache 中查找 → ...
    │   │   ▼
    │   └── 到达目标 dentry / 最后一个分量
    │
    ├── do_open() — 执行具体文件系统的 open 操作
    │   │  dentry->d_inode->i_fop->open() (或 def_file_open())
    │   ▼
    └── 返回 struct file → 分配到当前进程 fdtable → 返回 fd
```

关键点：

* **dcache 加速**：名字解析结果缓存在 dcache 中，后续同路径查找不经过具体文件系统的 `lookup()`
* **符号链接处理**：遇到 symlink 时，`link_path_walk()` 会跟踪到目标路径（有限制次数，默认 40 跳）
* **挂载点穿越**：dentry 的 `d_op->d_automount()` 或挂载点在查找过程中自动穿越

## 代码示例：注册自定义 file_operations 的内核模块

以下示例演示如何编写一个简单的内核模块，暴露一个 `/proc/hello_vfs` 文件，每次读取返回 "Hello, VFS!\n"：

```c
#include <linux/module.h>
#include <linux/kernel.h>
#include <linux/proc_fs.h>
#include <linux/uaccess.h>
#include <linux/fs.h>
#include <linux/slab.h>

#define PROC_NAME "hello_vfs"

static struct proc_dir_entry *proc_entry;

/* read 操作实现 */
static ssize_t hello_read(struct file *file, char __user *ubuf,
                          size_t count, loff_t *ppos)
{
    const char *msg = "Hello, VFS!\n";
    size_t len = strlen(msg);

    if (*ppos >= len)
        return 0;                   /* EOF */

    if (count > len - *ppos)
        count = len - *ppos;

    if (copy_to_user(ubuf, msg + *ppos, count))
        return -EFAULT;

    *ppos += count;
    return count;
}

/* write 操作实现：简单忽略写内容 */
static ssize_t hello_write(struct file *file, const char __user *ubuf,
                           size_t count, loff_t *ppos)
{
    /* 什么都不做，只是确认收到数据 */
    pr_info("hello_vfs: received %zu bytes\n", count);
    return count;
}

/* file_operations 实例 */
static const struct file_operations hello_fops = {
    .owner  = THIS_MODULE,
    .read   = hello_read,
    .write  = hello_write,
    .llseek = default_llseek,
};

static int __init hello_init(void)
{
    proc_entry = proc_create(PROC_NAME, 0444, NULL, &hello_fops);
    if (!proc_entry) {
        pr_err("hello_vfs: failed to create /proc/%s\n", PROC_NAME);
        return -ENOMEM;
    }
    pr_info("hello_vfs: /proc/%s created\n", PROC_NAME);
    return 0;
}

static void __exit hello_exit(void)
{
    remove_proc_entry(PROC_NAME, NULL);
    pr_info("hello_vfs: /proc/%s removed\n", PROC_NAME);
}

module_init(hello_init);
module_exit(hello_exit);

MODULE_LICENSE("GPL");
MODULE_AUTHOR("Your Name");
MODULE_DESCRIPTION("A simple VFS file_operations demo module");
```

编译方法（Makefile）：

```makefile
obj-m += hello_vfs.o

all:
	make -C /lib/modules/$(shell uname -r)/build M=$(PWD) modules

clean:
	make -C /lib/modules/$(shell uname -r)/build M=$(PWD) clean
```

测试步骤：

```bash
# 编译并加载模块
make
sudo insmod hello_vfs.ko

# 读取 proc 文件
cat /proc/hello_vfs
# 输出: Hello, VFS!

# 写入数据
echo "test data" > /proc/hello_vfs
# 内核日志: hello_vfs: received 10 bytes

# 卸载模块
sudo rmmod hello_vfs
```

## 相关资料

* Linux 内核源码: `fs/` 目录；`include/linux/fs.h` 中 VFS 核心结构体定义
* [Linux Kernel Documentation: VFS](https://www.kernel.org/doc/html/latest/filesystems/vfs.html)
* 《Linux Kernel Development》（第 3 版）第 12–15 章
* 《Understanding the Linux Kernel》（第 3 版）第 12 章
* `man 7 vfs` — Linux VFS 接口说明
