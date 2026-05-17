---
tags: [linux, strace, ptrace, debugging]
type: entity
aliases:
  - Linux strace
  - ptrace 系统调用跟踪
---

> `strace` 是 Linux 系统调用跟踪工具，通过 `ptrace` 系统调用拦截目标进程的每一次系统调用。它是调试程序行为、排查权限问题和理解系统工作原理的必备工具。

## strace 常用选项

```bash
# 跟踪命令
strace ls
strace -e open,read,write ls       # 只跟踪特定系统调用
strace -p 1234                     # 附加到运行中进程
strace -c ls                       # 统计系统调用次数和时间
strace -f ls                       # 跟踪子进程 (fork)
strace -t ls                       # 显示时间戳
strace -T ls                       # 显示每个系统调用耗时
strace -o trace.log ls             # 输出到文件
strace -s 1024 ls                  # 设置字符串打印长度
strace -k ls                       # 打印系统调用的内核调用栈
```

## strace 输出解析

```
$ strace -e trace=open,read,write ls ~/
open("/home/user/", O_RDONLY|O_NONBLOCK|O_DIRECTORY|O_CLOEXEC) = 3
getdents64(3, 0x7fff..., 32768)     = 520
getdents64(3, 0x7fff..., 32768)     = 0
close(3)                            = 0
write(1, "Documents\nDownloads\n", 20) = 20
```

每一行格式:

```
系统调用名称(参数...) = 返回值
```

| 字段 | 含义 | 示例 |
|------|------|------|
| `open` | 系统调用名称 | 也可能是 read/write/ioctl |
| `/home/user/` | 字符串参数 | 文件路径 |
| `O_RDONLY` | 标志位参数 | 内核展开为符号名 |
| `= 3` | 返回值 | 文件描述符、字节数或 -1 (错误) |
| `= 520` | 有效返回值 | 读取的字节数 |

## ptrace 基本原理

`strace` 的核心依赖是 `ptrace` 系统调用:

```c
// strace 的内部机制大致如下:
ptrace(PTRACE_TRACEME, 0, 0, 0);   // 被跟踪进程: 告诉内核将被跟踪
ptrace(PTRACE_ATTACH, pid, 0, 0);  // 跟踪者: 附加到目标进程

while (1) {
    int status;
    waitpid(pid, &status, 0);       // 等待被跟踪进程停止
    // 读取系统调用号和参数 (通过 PTRACE_PEEKUSER)
    // 打印系统调用信息
    ptrace(PTRACE_SYSCALL, pid, 0, 0); // 继续运行直到下一个 syscall
}
```

### ptrace 工作流程

```
跟踪者 (strace)                          被跟踪进程 (target)
      │                                        │
      ├─ ptrace(PTRACE_ATTACH, pid) ──────────►│
      │                                        │
      ├─ waitpid(pid) ◄── 进程停在 syscall ────┤
      │                                        │
      ├─ 读取寄存器 (PTRACE_PEEKUSER)           │
      │  (rax=syscall号, rdi=参数1...)         │
      │  打印系统调用信息                       │
      │                                        │
      ├─ ptrace(PTRACE_SYSCALL) ──────────────►│ 继续执行到 syscall 返回
      │                                        │
      ├─ waitpid(pid) ◄── 进程停在返回点 ──────┤
      │                                        │
      ├─ 读取返回值 (rax)                       │
      │  打印返回值                             │
      │                                        │
      └─ ptrace(PTRACE_SYSCALL) ──────────────►│ 继续执行到下一个 syscall
```

> `ptrace` 不仅用于 `strace`，也是 `gdb` 调试器的核心机制。`gdb` 通过 `PTRACE_PEEKDATA`/`PTRACE_POKEDATA` 读写被调试进程的内存和寄存器。

## 常见排查场景

```bash
# 排查程序启动失败 (权限问题)
strace -e open,stat,access /usr/bin/myapp

# 排查网络连接问题
strace -e trace=network curl https://example.com

# 排查慢系统调用
strace -c -T -p $(pidof mysqld)

# 跟踪信号发送
strace -e trace=signal kill -USR1 12345

# 查看子进程行为
strace -f -e clone,fork,execve make
```

## 相关文档

* [系统调用](系统调用.md) — 系统调用总览
* [调用约定](调用约定.md) — 系统调用调用约定与 VDSO

## 相关资料

* [Linux man pages — strace(1)](https://man7.org/linux/man-pages/man1/strace.1.html)
* [Linux man pages — ptrace(2)](https://man7.org/linux/man-pages/man2/ptrace.2.html)
* [How strace works — Julia Evans](https://jvns.ca/blog/2015/04/14/strace-zine/)
