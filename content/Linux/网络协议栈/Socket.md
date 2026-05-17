---
tags: [linux, socket, network]
type: entity
---

> Socket 是 Linux 网络编程的核心抽象，它将复杂的网络协议封装为文件描述符，使应用程序可以通过统一的 read/write 接口进行网络通信。理解 socket 的内部结构、API 流程和事件驱动模型是高性能网络编程的基石。

## Socket 类型

| 类型 | 宏定义 | 传输层 | 特点 |
|------|--------|--------|------|
| 流式套接字 | `SOCK_STREAM` | TCP | 面向连接、可靠、有序、无边界 |
| 数据报套接字 | `SOCK_DGRAM` | UDP | 无连接、不可靠、有边界 |
| 原始套接字 | `SOCK_RAW` | IP 层 | 直接操作 IP 包，绕过传输层 |
| 顺序包套接字 | `SOCK_SEQPACKET` | SCTP | 面向连接、可靠、有边界 |

```c
// 创建不同类型的 socket
int tcp_sock = socket(AF_INET, SOCK_STREAM, 0);    // TCP
int udp_sock = socket(AF_INET, SOCK_DGRAM, 0);     // UDP
int raw_sock = socket(AF_INET, SOCK_RAW, IPPROTO_TCP); // RAW
```

### 协议族 (Address Family)

| 协议族 | 说明 | 地址结构 |
|--------|------|----------|
| `AF_INET` | IPv4 | `struct sockaddr_in` |
| `AF_INET6` | IPv6 | `struct sockaddr_in6` |
| `AF_UNIX` | Unix 域套接字 | `struct sockaddr_un` |
| `AF_PACKET` | 链路层包 | `struct sockaddr_ll` |

## 核心数据结构

### struct socket — Socket 通用层

对应 VFS 层面，每个 socket 对应一个 `struct inode` 和 `struct file`。

```c
struct socket {
    socket_state            state;      // SS_FREE / SS_UNCONNECTED / SS_CONNECTED...
    short                   type;       // SOCK_STREAM / SOCK_DGRAM / SOCK_RAW...
    unsigned long           flags;      // SOCK_NONBLOCK, SOCK_CLOEXEC
    struct proto_ops       *ops;        // 协议操作函数表 (inet_stream_ops / inet_dgram_ops)
    struct sock             *sk;        // 指向传输层 sock 结构
    struct file             *file;      // VFS 文件对象
};
```

### struct sock — 传输层控制块

协议层面的核心结构，体积庞大 (~2000 字节)，包含连接状态、接收/发送缓冲区、拥塞控制、选项等。

```c
struct sock {
    struct sk_buff_head     sk_receive_queue;   // 接收队列
    struct sk_buff_head     sk_write_queue;     // 发送队列
    u32                     sk_daddr;           // 远端 IP 地址
    u32                     sk_rcv_saddr;       // 本地 IP 地址
    u16                     sk_dport;           // 远端端口
    u16                     sk_num;             // 本地端口
    unsigned int            sk_err;             // 错误码
    int                     sk_rcvbuf;          // 接收缓冲区大小
    int                     sk_sndbuf;          // 发送缓冲区大小
    struct socket           *sk_socket;         // 反向指向 socket
    unsigned char           sk_state;           // TCP 状态 (TCP_ESTABLISHED 等)
    struct proto            *sk_prot;           // 协议操作表 (tcp_prot / udp_prot)
    wait_queue_head_t       sk_wq;              // 等待队列 (用于阻塞 I/O)
    struct tcp_sock         *sk_tcp;            // TCP 扩展字段 (仅 TCP)
};
```

### struct sk_buff — 报文缓冲区

网络协议栈中最核心的数据结构，贯穿整个协议栈。每个网络包在内存中以 `sk_buff` 形式存在，各层协议通过调整指针来添加/剥离头部，避免数据拷贝。

```c
struct sk_buff {
    struct sk_buff          *next;      // 链表指针
    struct sk_buff          *prev;

    /* 协议层指针 */
    union {
        struct tcphdr       *h;         // 传输层头部 (已废弃，使用下面透明联合体)
        struct udphdr       *nh;
        struct ethhdr       *mac;
    };
    struct net_device       *dev;       // 关联的网络设备

    /* 数据区指针 */
    unsigned char           *head;      // 已分配内存起点
    unsigned char           *data;      // 实际数据起点
    unsigned char           *tail;      // 实际数据终点
    unsigned char           *end;       // 已分配内存终点

    unsigned int            len;        // 数据长度 (data 到 tail)
    unsigned int            truesize;   // 总分配大小 (含 sk_buff 本身)

    /* 管理字段 */
    atomic_t                users;      // 引用计数
    unsigned short          protocol;   // L3 协议类型 (htons(ETH_P_IP))

    /* 校验和 */
    __wsum                  csum;       // 校验和
};

/* 数据区结构示意图:
 *
 * | head ←────────────────→ end |  ← 整个分配的内存
 *        | data ←────→ tail |   ← 实际有效数据
 *
 * 协议栈中各层可以:
 *   skb_push(skb, header_len)  → data 指针前移 (添加头部)
 *   skb_put(skb, data_len)     → tail 指针后移 (添加数据)
 *   skb_pull(skb, header_len)  → data 指针后移 (剥离头部)
 *   skb_trim(skb, new_len)     → tail 指针前移 (移除尾部)
 */
```

#### sk_buff 数据操作 API

| 函数 | 操作 | 指针变化 | 典型场景 |
|------|------|----------|----------|
| `skb_put(skb, len)` | 添加数据 | tail += len | 驱动写入接收数据 |
| `skb_push(skb, len)` | 添加头部 | data -= len | 协议层添加头部 |
| `skb_pull(skb, len)` | 剥离头部 | data += len | 协议层解析头部 |
| `skb_reserve(skb, len)` | 预留头部空间 | data += len, tail += len | 分配时预留头部 |
| `skb_trim(skb, len)` | 截断数据 | tail = data + len | 去除尾部填充 |
| `skb_clone(skb, gfp)` | 浅拷贝 | 共享数据区 | 广播、多播 |
| `pskb_copy(skb, gfp)` | 拷贝头部 | 独立头部 + 共享数据 | 修改头部时 |
| `skb_copy(skb, gfp)` | 深拷贝 | 完全独立 | 需要修改数据时 |

## Socket API 流程

### 服务端流程

```
socket()  → 创建 socket 结构
    │
bind()    → 绑定本地地址 (IP + 端口)
    │
listen()  → 将 socket 设为被动监听状态
    │         创建全连接队列和半连接队列
    │
accept()  → 从全连接队列取出连接
    │         返回新的 socket fd
    │
recv() / read()   ←→  send() / write()
    │
close()
```

### 客户端流程

```
socket()  → 创建 socket
    │
connect() → 向服务端发起连接
    │         TCP 三次握手
    │
send() / write()   ←→  recv() / read()
    │
close()
```

### 关键系统调用详解

#### socket()

```c
int socket(int domain, int type, int protocol);
// domain:   AF_INET / AF_INET6 / AF_UNIX / AF_PACKET
// type:     SOCK_STREAM / SOCK_DGRAM / SOCK_RAW / SOCK_NONBLOCK
// protocol: 通常传 0 (由 type 决定), SOCK_RAW 时指定 IPPROTO_xxx
// return:   文件描述符 (>=0) 或 -1 (errno 设置)
```

内核流程：分配 `struct socket` → 分配 `struct sock` → 分配文件描述符 → 关联 VFS inode。

#### bind()

```c
int bind(int sockfd, const struct sockaddr *addr, socklen_t addrlen);

struct sockaddr_in addr = {
    .sin_family = AF_INET,
    .sin_port   = htons(8080),           // 端口 (网络字节序)
    .sin_addr   = .s_addr = INADDR_ANY,  // 绑定所有网卡
};
```

#### listen()

```c
int listen(int sockfd, int backlog);
// backlog: 全连接队列的最大长度
```

内核创建两个队列：

* **SYN 队列** (半连接队列)：收到 SYN 但未完成三次握手的连接
* **Accept 队列** (全连接队列)：已完成三次握手，等待 accept() 取走的连接

#### accept()

```c
int accept(int sockfd, struct sockaddr *addr, socklen_t *addrlen);
// return: 新的已连接 socket fd，如果 addr 非 NULL 则同时返回客户端地址
```

内核从全连接队列取出一个已完成握手的连接，创建新的 socket fd。

#### connect()

```c
int connect(int sockfd, const struct sockaddr *addr, socklen_t addrlen);
```

对于 TCP：触发三次握手，默认阻塞直到连接建立或超时。对于 UDP：仅记录远端地址，不发送任何包。

### 阻塞与非阻塞

| 模式 | 设置方式 | send() 缓冲区满 | recv() 无数据 | connect() 未完成 |
|------|----------|----------------|---------------|------------------|
| 阻塞 | 默认 | 阻塞直到可写 | 阻塞直到有数据 | 阻塞直到握手完成 |
| 非阻塞 | `O_NONBLOCK` | 返回 `EAGAIN` | 返回 `EAGAIN` | 返回 `EINPROGRESS` |

## TCP 状态机

```
CLOSED
  │
  │ 被动 open: listen()          主动 open: connect() → SYN_SENT
  │                                    │ 收到 SYN+ACK (三次握手)
  ▼                                    ▼
LISTEN  ←───────────────────  SYN_SENT
  │                                    │
  │ 收到 SYN → SYN_RCVD                │
  │    │ 发送 SYN+ACK                  │
  │    ▼                               │
  │ SYN_RCVD                           │
  │    │ 收到 ACK (三次握手完成)         │
  │    ▼                               ▼
  └─────────────→ ESTABLISHED ←────────┘
                        │
              ┌────────┴────────┐
              │                 │
        主动关闭:            被动关闭:
        close() →           recv() 返回 0 →
        FIN_WAIT1           CLOSE_WAIT
              │                 │
        收到 ACK →          close() →
        FIN_WAIT2           LAST_ACK
              │                 │
        收到 FIN →          收到 ACK →
        TIME_WAIT           CLOSED
              │
        2MSL 超时 →
        CLOSED
```

### 关键状态说明

| 状态 | 含义 | 说明 |
|------|------|------|
| `LISTEN` | 监听中 | 服务端等待客户端连接 |
| `SYN_SENT` | 发送 SYN | 客户端主动发起连接 |
| `SYN_RCVD` | 收到 SYN | 服务端收到 SYN，等待 ACK |
| `ESTABLISHED` | 连接已建立 | 正常数据传输状态 |
| `FIN_WAIT1` | 主动关闭第一步 | 发送 FIN 后等待 ACK |
| `FIN_WAIT2` | 主动关闭第二步 | 收到 ACK 后等待对端 FIN |
| `TIME_WAIT` | 等待 2MSL | 确保对端收到最后 ACK |
| `CLOSE_WAIT` | 被动关闭 | 收到 FIN 后等待本地 close() |
| `LAST_ACK` | 被动关闭最后一步 | 发送 FIN 后等待 ACK |
| `CLOSED` | 连接关闭 | 无连接状态 |

> `ss -tna` 可以查看所有 TCP 连接状态。大量 `TIME_WAIT` 或 `CLOSE_WAIT` 通常表明应用程序设计问题。

## 高并发模型：epoll

传统多线程/多进程模型在 C10K 问题前失效，epoll 是 Linux 上解决高并发 I/O 的最佳方案。

### epoll 核心 API

```c
int epoll_create1(int flags);
// flags: 0 或 EPOLL_CLOEXEC
// return: epoll 文件描述符

int epoll_ctl(int epfd, int op, int fd, struct epoll_event *event);
// op:     EPOLL_CTL_ADD / EPOLL_CTL_MOD / EPOLL_CTL_DEL
// event:  注册的事件和用户数据

int epoll_wait(int epfd, struct epoll_event *events,
               int maxevents, int timeout);
// timeout: -1 = 阻塞, 0 = 立即返回, >0 = 超时毫秒
// return:  就绪事件数量
```

### 事件类型

| 事件宏 | 含义 |
|--------|------|
| `EPOLLIN` | 数据可读 |
| `EPOLLOUT` | 缓冲区可写 |
| `EPOLLRDHUP` | 对端关闭连接 |
| `EPOLLPRI` | 紧急数据可读 |
| `EPOLLERR` | 发生错误 |
| `EPOLLHUP` | 挂起 |
| `EPOLLET` | 边沿触发模式 |
| `EPOLLONESHOT` | 一次性触发 |

### 水平触发 vs 边沿触发

| 模式 | 简称 | 行为 | 场景 |
|------|------|------|------|
| **Level-Triggered** | LT | 只要 fd 有数据可读，`epoll_wait` 就一直返回 | 简单、不易漏事件，但可能重复通知 |
| **Edge-Triggered** | ET | 仅当状态发生变化时通知一次 | 高性能，必须循环 read 直到 EAGAIN |

### epoll 内部实现

epoll 在内核中使用三个关键数据结构：

1. **红黑树** (rbtree) — 存储所有注册的 fd，支持快速增删改查
2. **就绪链表** (rdllist) — 存储有事件发生的 fd，`epoll_wait` 直接从链表取数据
3. **回调机制** — 每个 fd 注册回调函数 `ep_poll_callback`，当 fd 有事件时，内核将 fd 加入就绪链表并唤醒等待进程

```
epoll fd
    │
    ├── rbtree: 所有注册的 fd (快速查找 O(logN))
    │     │
    │     ├── [fd=3, events=EPOLLIN]
    │     ├── [fd=4, events=EPOLLIN|EPOLLOUT]
    │     ├── [fd=5, events=EPOLLIN]
    │     └── ...
    │
    └── rdllist: 有事件待处理的 fd (双向链表)
          │
          ├── [fd=3]  (数据已到达)
          └── [fd=5]  (数据已到达)
```

## 代码示例

### 示例 1：TCP Echo 服务端 (阻塞 I/O)

```c
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <unistd.h>
#include <sys/socket.h>
#include <netinet/in.h>
#include <arpa/inet.h>

#define PORT 8080
#define BUFFER_SIZE 1024

int main() {
    int server_fd, client_fd;
    struct sockaddr_in addr;
    socklen_t addr_len = sizeof(addr);
    char buffer[BUFFER_SIZE];

    // 1. 创建 socket
    server_fd = socket(AF_INET, SOCK_STREAM, 0);
    if (server_fd < 0) {
        perror("socket");
        exit(EXIT_FAILURE);
    }

    // 2. 设置 socket 选项 (允许地址重用)
    int opt = 1;
    setsockopt(server_fd, SOL_SOCKET, SO_REUSEADDR, &opt, sizeof(opt));

    // 3. bind
    addr.sin_family = AF_INET;
    addr.sin_port = htons(PORT);
    addr.sin_addr.s_addr = INADDR_ANY;
    if (bind(server_fd, (struct sockaddr *)&addr, sizeof(addr)) < 0) {
        perror("bind");
        close(server_fd);
        exit(EXIT_FAILURE);
    }

    // 4. listen
    if (listen(server_fd, 5) < 0) {
        perror("listen");
        close(server_fd);
        exit(EXIT_FAILURE);
    }
    printf("Echo server listening on port %d\n", PORT);

    // 5. accept 循环
    while (1) {
        client_fd = accept(server_fd, (struct sockaddr *)&addr, &addr_len);
        if (client_fd < 0) {
            perror("accept");
            continue;
        }

        char *client_ip = inet_ntoa(addr.sin_addr);
        printf("New connection from %s:%d\n", client_ip, ntohs(addr.sin_port));

        // 6. echo 循环 (单线程/串行)
        ssize_t n;
        while ((n = read(client_fd, buffer, sizeof(buffer) - 1)) > 0) {
            buffer[n] = '\0';
            printf("Received: %s", buffer);
            write(client_fd, buffer, n);  // echo back
        }

        printf("Connection closed: %s:%d\n", client_ip, ntohs(addr.sin_port));
        close(client_fd);
    }

    close(server_fd);
    return 0;
}
```

### 示例 2：TCP Echo 客户端

```c
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <unistd.h>
#include <sys/socket.h>
#include <netinet/in.h>
#include <arpa/inet.h>

#define PORT 8080
#define BUFFER_SIZE 1024

int main() {
    int sock_fd;
    struct sockaddr_in server_addr;
    char buffer[BUFFER_SIZE];

    // 1. 创建 socket
    sock_fd = socket(AF_INET, SOCK_STREAM, 0);
    if (sock_fd < 0) {
        perror("socket");
        exit(EXIT_FAILURE);
    }

    // 2. connect
    server_addr.sin_family = AF_INET;
    server_addr.sin_port = htons(PORT);
    inet_pton(AF_INET, "127.0.0.1", &server_addr.sin_addr);

    if (connect(sock_fd, (struct sockaddr *)&server_addr, sizeof(server_addr)) < 0) {
        perror("connect");
        close(sock_fd);
        exit(EXIT_FAILURE);
    }
    printf("Connected to server\n");

    // 3. 发送/接收循环
    while (1) {
        printf("> ");
        if (fgets(buffer, sizeof(buffer), stdin) == NULL)
            break;

        send(sock_fd, buffer, strlen(buffer), 0);

        ssize_t n = recv(sock_fd, buffer, sizeof(buffer) - 1, 0);
        if (n <= 0)
            break;

        buffer[n] = '\0';
        printf("Echo: %s", buffer);
    }

    close(sock_fd);
    return 0;
}
```

### 示例 3：epoll 高并发 Echo 服务端 (边沿触发)

```c
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <unistd.h>
#include <fcntl.h>
#include <sys/epoll.h>
#include <sys/socket.h>
#include <netinet/in.h>

#define PORT 8080
#define MAX_EVENTS 1024
#define BUFFER_SIZE 4096

/* 将 fd 设为非阻塞 */
static int set_nonblocking(int fd) {
    int flags = fcntl(fd, F_GETFL, 0);
    if (flags < 0) return -1;
    return fcntl(fd, F_SETFL, flags | O_NONBLOCK);
}

int main() {
    int server_fd, epoll_fd, nfds;
    struct sockaddr_in addr;
    struct epoll_event ev, events[MAX_EVENTS];

    // 1. 创建 server socket
    server_fd = socket(AF_INET, SOCK_STREAM | SOCK_NONBLOCK, 0);
    if (server_fd < 0) { perror("socket"); exit(EXIT_FAILURE); }

    int opt = 1;
    setsockopt(server_fd, SOL_SOCKET, SO_REUSEADDR, &opt, sizeof(opt));

    addr.sin_family = AF_INET;
    addr.sin_port = htons(PORT);
    addr.sin_addr.s_addr = INADDR_ANY;
    if (bind(server_fd, (struct sockaddr *)&addr, sizeof(addr)) < 0)
        { perror("bind"); exit(EXIT_FAILURE); }

    if (listen(server_fd, 128) < 0)
        { perror("listen"); exit(EXIT_FAILURE); }

    // 2. 创建 epoll fd
    epoll_fd = epoll_create1(0);
    if (epoll_fd < 0) { perror("epoll_create1"); exit(EXIT_FAILURE); }

    // 3. 注册 server_fd (监听 accept 事件)
    ev.events = EPOLLIN | EPOLLET;  // 边沿触发
    ev.data.fd = server_fd;
    epoll_ctl(epoll_fd, EPOLL_CTL_ADD, server_fd, &ev);

    printf("epoll echo server listening on port %d\n", PORT);

    // 4. 事件循环
    while (1) {
        nfds = epoll_wait(epoll_fd, events, MAX_EVENTS, -1);
        if (nfds < 0) { perror("epoll_wait"); break; }

        for (int i = 0; i < nfds; i++) {
            // 4a. 新连接到达
            if (events[i].data.fd == server_fd) {
                struct sockaddr_in client_addr;
                socklen_t client_len = sizeof(client_addr);

                /* ET 模式必须循环 accept 直到 EAGAIN */
                int client_fd;
                while ((client_fd = accept(server_fd,
                        (struct sockaddr *)&client_addr, &client_len)) >= 0) {
                    set_nonblocking(client_fd);

                    ev.events = EPOLLIN | EPOLLET | EPOLLRDHUP;
                    ev.data.fd = client_fd;
                    epoll_ctl(epoll_fd, EPOLL_CTL_ADD, client_fd, &ev);
                }
                if (client_fd < 0 && errno != EAGAIN && errno != EWOULDBLOCK) {
                    perror("accept");
                }
            }
            // 4b. 客户端数据到达或断开
            else {
                int fd = events[i].data.fd;
                char buffer[BUFFER_SIZE];

                if (events[i].events & (EPOLLRDHUP | EPOLLHUP | EPOLLERR)) {
                    /* 连接断开或出错 */
                    close(fd);
                    continue;
                }

                /* ET 模式必须循环 read 直到 EAGAIN */
                ssize_t n;
                int err = 0;
                while (1) {
                    n = read(fd, buffer, sizeof(buffer));
                    if (n > 0) {
                        write(fd, buffer, n);  // echo back
                    } else if (n == 0) {
                        /* 对端关闭 */
                        err = 1;
                        break;
                    } else {
                        if (errno == EAGAIN || errno == EWOULDBLOCK)
                            break;  // 数据已读完
                        /* 真正的错误 */
                        err = 1;
                        break;
                    }
                }

                if (err) {
                    epoll_ctl(epoll_fd, EPOLL_CTL_DEL, fd, NULL);
                    close(fd);
                }
            }
        }
    }

    close(server_fd);
    close(epoll_fd);
    return 0;
}
```

### epoll 示例编译与运行

```bash
# 编译
gcc -o echo_epoll_server echo_epoll_server.c

# 运行服务端
./echo_epoll_server

# 在另一个终端使用 telnet 或 nc 测试
telnet 127.0.0.1 8080
# 或
nc 127.0.0.1 8080
```

## 相关资料

* Linux 内核源码: `net/socket.c`, `net/ipv4/tcp.c`, `net/core/sk_buff.c`
* `man 7 socket` / `man 7 tcp` / `man 7 epoll`
* [网络协议栈](网络协议栈.md) — 网络协议栈整体架构
* `ss -tuln` / `ss -tna` — socket 与连接状态查询
* [epoll 内核实现分析](epoll.md) — epoll 红黑树与回调机制详解
* [I/O 多路复用](io-multiplexing.md) — select/poll/epoll 对比
