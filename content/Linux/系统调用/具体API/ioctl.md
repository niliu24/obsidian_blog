---
tags: [linux, syscall, ioctl]
type: entity
aliases:
  - ioctl 系统调用
  - ioctl
---

> `ioctl`（Input/Output Control）是 Linux 中最强大的系统调用之一，用于对设备执行**超出标准 read/write 语义**的控制操作。它提供了泛化的命令通道——一个系统调用号承载了成千上万种设备特定操作。ioctl 常用于配置设备参数、查询设备状态、触发特殊操作等。

## 函数原型

```c
#include <sys/ioctl.h>

int ioctl(int fd, unsigned long request, ...);
```

| 参数 | 说明 |
|------|------|
| `fd` | 打开的文件描述符（设备文件、socket、普通文件等） |
| `request` | 设备相关的命令码（cmd），由宏构造，编码了方向和参数大小 |
| `…` | 可选参数，通常是一个指针，类型由 `request` 决定 |

返回值：成功返回 0，失败返回 -1 并设置 `errno`。

## 命令码 (request) 的构造

ioctl 命令码是一个 32 位整数，由 4 个字段拼接而成：

```
┌────────┬───────┬─────┬──────┬──────────────────┐
│  bits  │ 31-30 │29-16│ 15-8 │      7-0         │
├────────┼───────┼─────┼──────┼──────────────────┤
│ 含义   │ 方向  │ 大小 │ 类型  │ 序号             │
│ 宏     │ _IOC_ │ _IOC│ _IOC_│ _IOC_NR          │
│        │ DIR   │SIZE │ TYPE │                  │
└────────┴───────┴─────┴──────┴──────────────────┘
```

| 字段 | 位数 | 含义 | 取值范围 |
|------|------|------|---------|
| 方向 | 2 bit | 数据传输方向（读/写/读写/无） | `_IOC_NONE`, `_IOC_READ`, `_IOC_WRITE`, `_IOC_READ\|_IOC_WRITE` |
| 大小 | 14 bit | 第三个参数的字节数 | `sizeof(arg_type)` |
| 类型 | 8 bit | 魔数（Magic Number），区分不同设备 | 0x00-0xFF |
| 序号 | 8 bit | 该设备内的命令序号 | 0x00-0xFF |

### 构造宏

```c
#include <linux/ioctl.h>

// 定义命令码
#define MYDEV_IOC_MAGIC  'k'

// 无参数命令
#define MYDEV_RESET      _IO(MYDEV_IOC_MAGIC, 0)

// 写参数（用户→内核）
#define MYDEV_SET_CONFIG  _IOW(MYDEV_IOC_MAGIC, 1, struct mydev_config)

// 读参数（内核→用户）
#define MYDEV_GET_CONFIG  _IOR(MYDEV_IOC_MAGIC, 2, struct mydev_config)

// 读写参数
#define MYDEV_XCHG_DATA   _IOWR(MYDEV_IOC_MAGIC, 3, struct mydev_data)
```

| 宏 | 方向 | 用途 |
|----|------|------|
| `_IO(type, nr)` | 无 | 纯命令，无数据交换 |
| `_IOR(type, nr, datatype)` | 读 | 内核 → 用户 |
| `_IOW(type, nr, datatype)` | 写 | 用户 → 内核 |
| `_IOWR(type, nr, datatype)` | 读写 | 双向数据交换 |

## 驱动端实现

```c
#include <linux/ioctl.h>
#include <linux/uaccess.h>

struct mydev_config {
    int mode;
    int timeout_ms;
    char name[32];
};

#define MYDEV_IOC_MAGIC  'k'
#define MYDEV_SET_CONFIG  _IOW(MYDEV_IOC_MAGIC, 1, struct mydev_config)
#define MYDEV_GET_CONFIG  _IOR(MYDEV_IOC_MAGIC, 2, struct mydev_config)
#define MYDEV_RESET       _IO(MYDEV_IOC_MAGIC, 0)
#define MYDEV_IOC_MAXNR   2  // 最大命令序号

static long mydev_ioctl(struct file *file, unsigned int cmd, unsigned long arg)
{
    struct mydev_config config;

    // 验证魔数和命令范围
    if (_IOC_TYPE(cmd) != MYDEV_IOC_MAGIC)
        return -ENOTTY;
    if (_IOC_NR(cmd) > MYDEV_IOC_MAXNR)
        return -ENOTTY;

    switch (cmd) {
    case MYDEV_RESET:
        // 执行重置操作
        pr_info("mydev: reset\n");
        break;

    case MYDEV_SET_CONFIG:
        // 从用户空间拷贝数据
        if (copy_from_user(&config, (void __user *)arg, sizeof(config)))
            return -EFAULT;
        pr_info("mydev: set mode=%d, timeout=%d, name=%s\n",
                config.mode, config.timeout_ms, config.name);
        break;

    case MYDEV_GET_CONFIG:
        // 填充数据并返回给用户空间
        config.mode = 1;
        config.timeout_ms = 5000;
        strscpy(config.name, "mydev0", sizeof(config.name));
        if (copy_to_user((void __user *)arg, &config, sizeof(config)))
            return -EFAULT;
        break;

    default:
        return -ENOTTY; // "Not a typewriter" — 不支持的 ioctl 命令
    }
    return 0;
}
```

## 用户空间调用

```c
#include <sys/ioctl.h>
#include <fcntl.h>
#include <stdio.h>

int main()
{
    int fd = open("/dev/mydev", O_RDWR);
    if (fd < 0) {
        perror("open");
        return 1;
    }

    // 无参数命令
    ioctl(fd, MYDEV_RESET);

    // 写参数
    struct mydev_config cfg = { .mode = 2, .timeout_ms = 3000, .name = "test" };
    if (ioctl(fd, MYDEV_SET_CONFIG, &cfg) < 0)
        perror("ioctl SET_CONFIG");

    // 读参数
    if (ioctl(fd, MYDEV_GET_CONFIG, &cfg) == 0)
        printf("mode=%d, timeout=%d, name=%s\n",
               cfg.mode, cfg.timeout_ms, cfg.name);

    close(fd);
    return 0;
}
```

## 常见 ioctl 场景

| 设备类型 | 典型命令 | 用途 |
|----------|---------|------|
| **终端 (TTY)** | `TCGETS` / `TCSETS` | 获取/设置终端属性 (termios) |
| **Socket** | `FIONREAD` / `SIOCGIFADDR` | 查询可读字节数 / 获取网卡 IP |
| **磁盘** | `HDIO_GETGEO` / `BLKGETSIZE` | 获取磁盘几何信息 / 块设备大小 |
| **显卡 (DRM)** | `DRM_IOCTL_*` | GPU 命令提交、显存管理 |
| **输入设备** | `EVIOCGNAME` / `EVIOCGABS` | 获取设备名称 / 绝对值信息 |
| **NVMe** | `NVME_IOCTL_*` | NVMe 命令提交 |
| **非标准文件** | `FS_IOC_GETFLAGS` / `FICLONE` | 获取/设置 inode 标志 / 文件克隆 |

## ioctl 的优缺点

| 优点 | 缺点 |
|------|------|
| 统一的命令通道，驱动可自由扩展 | 命令码无类型安全（编译期不检查） |
| 支持双向数据传输 | 命令码冲突风险（魔数需全局协调） |
| 一个系统调用覆盖所有设备 | 用户空间需额外的头文件才能知道命令码 |
| 避免创建大量专用系统调用 | 复杂数据结构版本兼容性难处理 |

> 现代内核中，很多 ioctl 已迁移到 sysfs 属性文件或 netlink 接口，以获得更好的可发现性和类型安全。但对于性能要求高的操作（批量命令提交），ioctl 仍是首选。

## 相关文档

* [系统调用](../系统调用.md) — 系统调用总览
* [设备驱动](../../设备驱动/设备驱动.md) — ioctl 在字符设备驱动中的应用
* [mmap](mmap.md) — 另一种绕过 read/write 的数据交换机制

## 相关资料

* [Linux man pages — ioctl(2)](https://man7.org/linux/man-pages/man2/ioctl.2.html)
* [Linux kernel — ioctl based interfaces](https://www.kernel.org/doc/html/latest/driver-api/ioctl.html)
* [Linux Device Drivers, 3rd Edition — Ch. 6](https://lwn.net/Kernel/LDD3/)
