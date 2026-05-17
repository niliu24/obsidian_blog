---
tags: [linux, epoll, io-multiplexing, network]
type: entity
aliases:
  - epoll
  - IO 多路复用
---

> epoll 是 Linux 内核提供的高性能 I/O 多路复用机制。它解决了 select/poll 在大量文件描述符下 O(n) 扫描的性能瓶颈，通过内核事件驱动以 O(1) 的方式返回就绪事件。epoll 是现代高性能服务器（Nginx、Redis、Go netpoller）的核心组件。

## epoll vs select/poll

| 特性 | select | poll | epoll |
|------|--------|------|-------|
| **监控方式** | 内核全量遍历 FD 集合 | 同 select | 事件驱动（回调通知） |
| **复杂度** | O(n) | O(n) | O(1)（就绪事件数） |
| **最大 FD 数** | 1024 (FD_SETSIZE) | 无限制 | 无限制 |
| **内核态数据结构** | 位图 | 链表 | 红黑树 + 就绪链表 |
| **入-出参复用** | 是（每次重填） | 是 | 否（事件独立存储） |
| **边缘触发** | 不支持 | 不支持 | 支持 (EPOLLET) |

## epoll 核心数据结构

```
epoll 实例 (struct eventpoll)
    │
    ├─ interest list (红黑树)
    │  └─ 存储所有注册的 fd 及其关注事件
    │     └─ struct epitem (每个 fd 一个)
    │        └─ callback: ep_poll_callback
    │
    └─ ready list (双向链表)
       └─ 存储所有就绪的 epitem
          └─ 从 epoll_wait 返回后清空
```

## epoll API

### 创建 epoll 实例

```c
#include <sys/epoll.h>

int epfd = epoll_create1(0);
// epoll_create1(EPOLL_CLOEXEC) — 推荐，避免 fd 泄漏
```

### 注册/修改/删除事件

```c
struct epoll_event ev;
ev.events = EPOLLIN;          // 关注读事件
ev.data.fd = listen_fd;       // 用户数据（fd 或指针）

epoll_ctl(epfd, EPOLL_CTL_ADD, listen_fd, &ev);  // 注册
epoll_ctl(epfd, EPOLL_CTL_MOD, conn_fd, &ev);     // 修改
epoll_ctl(epfd, EPOLL_CTL_DEL, conn_fd, NULL);    // 删除
```

### 等待事件

```c
struct epoll_event events[MAX_EVENTS];

int nfds = epoll_wait(epfd, events, MAX_EVENTS, timeout_ms);
for (int i = 0; i < nfds; i++) {
    if (events[i].events & EPOLLIN) {
        int fd = events[i].data.fd;
        // 处理 fd 上的可读事件
    }
}
```

## 触发模式

| 模式 | 标志 | 行为 | 适用场景 |
|------|------|------|----------|
| **Level-Triggered (LT)** | 默认 | 只要 fd 就绪，epoll_wait 每次返回 | 简单可靠，应用层无要求 |
| **Edge-Triggered (ET)** | `EPOLLET` | fd 变为就绪时仅通知一次 | 高性能，必须非阻塞 + 读写直到 EAGAIN |

### ET 模式下的正确读写

```c
// ET 模式下必须循环读取直到 EAGAIN
while (1) {
    ssize_t n = read(fd, buf, sizeof(buf));
    if (n > 0) {
        // 处理数据
    } else if (n == 0) {
        // 对端关闭连接
        epoll_ctl(epfd, EPOLL_CTL_DEL, fd, NULL);
        close(fd);
        break;
    } else {
        if (errno == EAGAIN || errno == EWOULDBLOCK) {
            // 数据读取完毕
            break;
        }
        // 真正的错误
        epoll_ctl(epfd, EPOLL_CTL_DEL, fd, NULL);
        close(fd);
        break;
    }
}
```

## epoll 事件类型

| 事件 | 说明 |
|------|------|
| `EPOLLIN` | fd 可读（数据到达 / 连接可 accept） |
| `EPOLLOUT` | fd 可写 |
| `EPOLLERR` | fd 发生错误（自动监控，无需注册） |
| `EPOLLHUP` | fd 挂起（对端关闭）（自动监控） |
| `EPOLLRDHUP` | 对端关闭写入端（半关闭检测） |
| `EPOLLONESHOT` | 事件触发后自动从 interest list 移除，需重新注册 |

## 完整示例：epoll echo 服务器

```c
#include <sys/epoll.h>
#include <sys/socket.h>
#include <netinet/in.h>
#include <unistd.h>
#include <stdio.h>
#include <fcntl.h>
#include <errno.h>

#define MAX_EVENTS 1024
#define LISTEN_PORT 8080

int main()
{
    int listen_fd = socket(AF_INET, SOCK_STREAM, 0);
    int flags = fcntl(listen_fd, F_GETFL, 0);
    fcntl(listen_fd, F_SETFL, flags | O_NONBLOCK);

    struct sockaddr_in addr = {
        .sin_family = AF_INET,
        .sin_port   = htons(LISTEN_PORT),
        .sin_addr.s_addr = INADDR_ANY,
    };
    bind(listen_fd, (struct sockaddr*)&addr, sizeof(addr));
    listen(listen_fd, SOMAXCONN);

    int epfd = epoll_create1(0);
    struct epoll_event ev;
    ev.events = EPOLLIN;
    ev.data.fd = listen_fd;
    epoll_ctl(epfd, EPOLL_CTL_ADD, listen_fd, &ev);

    struct epoll_event events[MAX_EVENTS];

    while (1) {
        int nfds = epoll_wait(epfd, events, MAX_EVENTS, -1);
        for (int i = 0; i < nfds; i++) {
            int fd = events[i].data.fd;

            if (fd == listen_fd) {
                // 新连接
                int conn = accept(listen_fd, NULL, NULL);
                int flags = fcntl(conn, F_GETFL, 0);
                fcntl(conn, F_SETFL, flags | O_NONBLOCK);
                ev.events = EPOLLIN | EPOLLET;
                ev.data.fd = conn;
                epoll_ctl(epfd, EPOLL_CTL_ADD, conn, &ev);
            } else {
                // 客户端数据 — ET 模式循环读
                char buf[4096];
                while (1) {
                    ssize_t n = read(fd, buf, sizeof(buf));
                    if (n > 0) {
                        write(fd, buf, n); // echo back
                    } else if (n == 0) {
                        epoll_ctl(epfd, EPOLL_CTL_DEL, fd, NULL);
                        close(fd);
                        break;
                    } else if (errno == EAGAIN) {
                        break;
                    } else {
                        epoll_ctl(epfd, EPOLL_CTL_DEL, fd, NULL);
                        close(fd);
                        break;
                    }
                }
            }
        }
    }
}
```

## 相关文档

- [网络协议栈](../网络协议栈.md) — epoll 在内核网络栈中的位置
- [Socket](Socket.md) — epoll 管理的 fd 通常是 socket
- [中断处理](../../中断处理/中断处理.md) — epoll 的事件通知依赖内核的软中断机制

## 相关资料

- [Linux man pages — epoll(7)](https://man7.org/linux/man-pages/man7/epoll.7.html)
- [Linux kernel source: fs/eventpoll.c](https://elixir.bootlin.com/linux/latest/source/fs/eventpoll.c)
- [The Linux Programming Interface (Kerrisk) — Ch. 63](https://man7.org/tlpi/)
