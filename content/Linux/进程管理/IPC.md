---
tags: [linux, ipc]
type: entity
aliases:
  - Linux 进程间通信
  - IPC 机制
---

> Linux 提供多种进程间通信 (IPC) 机制，从历史悠久的管道和 System V IPC 到现代的事件驱动机制，覆盖不同场景下的数据传输、同步和事件通知需求。

## IPC 方法概览

| 方法 | 数据量 | 速度 | 是否需要同步 | 典型场景 |
|------|--------|------|-------------|----------|
| **管道 (Pipe)** | 小 - 中 | 快（内核 buffer） | 否（阻塞读写） | 父子进程通信，shell 管道 `|` |
| **命名管道 (FIFO)** | 小 - 中 | 快 | 否 | 无关进程间通信 |
| **信号 (Signal)** | 极少（仅信号编号） | 很快 | 否（异步处理） | 事件通知、进程控制（kill） |
| **共享内存 (shm)** | 极大 | 最快（直接内存访问） | **需要**（互斥锁/信号量） | 大数据量传输、共享数据结构 |
| **消息队列 (msg)** | 小 - 中 | 较快 | 否（内核同步） | 消息传递、任务分发 |
| **信号量 (sem)** | 0（仅计数） | 快 | 本身是同步原语 | 保护共享资源 |
| **Unix 域套接字** | 中 - 大 | 很快 | 否 | 本地 C/S 架构、传递文件描述符 |
| **eventfd** | 极简（8 字节） | 很快 | 否 | 事件通知、epoll 配合 |
| **D-Bus** | 中 | 中 | 否 | 桌面环境 IPC、服务总线 |

## 管道 (Pipe)

### 匿名管道

```c
#include <unistd.h>
#include <stdio.h>
#include <sys/wait.h>

int main() {
    int pipefd[2];
    char buf[128];

    if (pipe(pipefd) == -1) {
        perror("pipe");
        return 1;
    }

    pid_t pid = fork();
    if (pid == 0) {
        // 子进程：写入管道
        close(pipefd[0]);                // 关闭读端
        write(pipefd[1], "Hello from child", 17);
        close(pipefd[1]);
        return 0;
    }

    // 父进程：读取管道
    close(pipefd[1]);                    // 关闭写端
    ssize_t n = read(pipefd[0], buf, sizeof(buf) - 1);
    buf[n] = '\0';
    printf("Parent received: %s\n", buf);
    close(pipefd[0]);

    wait(NULL);
    return 0;
}
```

管道通信模型：

```
     写入端            内核缓冲区           读取端
  ┌────────┐        ┌──────────────┐        ┌────────┐
  │ write  │───────►│  环形 buffer │───────►│  read  │
  │ fd[1]  │        │  (PIPE_BUF)  │        │ fd[0]  │
  └────────┘        └──────────────┘        └────────┘
```

* 半双工：数据单向流动（一端写、一端读）
* 内核缓冲区大小：默认 65536 字节（Linux 5.8+，可通过 `fcntl(fd, F_SETPIPE_SZ)` 修改）
* `PIPE_BUF` 保证写原子性：POSIX 要求至少 4096 字节
* 管道满时 `write()` 阻塞；管道空时 `read()` 阻塞

### 命名管道 (FIFO)

无名管道只能用于有亲缘关系的进程；FIFO 通过文件系统中的特殊文件实现无关进程通信。

```bash
# 创建 FIFO
mkfifo /tmp/myfifo

# 终端 1: 读取
cat /tmp/myfifo

# 终端 2: 写入
echo "hello" > /tmp/myfifo
```

```c
#include <sys/stat.h>

// 在 C 程序中创建 FIFO
mkfifo("/tmp/myfifo", 0644);
```

* FIFO 在文件系统中表现为一个特殊文件（`p` 类型）
* 以 `O_RDONLY` 打开会**阻塞**直到另一端以 `O_WRONLY` 打开
* 以 `O_RDWR` 打开时不会阻塞（不常用）
* 配合 `open()` 的 `O_NONBLOCK` 标志实现非阻塞模式

## 信号 (Signal)

| 信号 | 编号 | 默认行为 | 说明 |
|------|------|----------|------|
| `SIGKILL` | 9 | 终止 | 不可捕获、不可忽略 |
| `SIGSTOP` | 19 | 暂停 | 不可捕获、不可忽略 |
| `SIGTERM` | 15 | 终止 | 可捕获，优雅终止 |
| `SIGINT` | 2 | 终止 | Ctrl+C，可捕获 |
| `SIGQUIT` | 3 | 终止 + Core dump | Ctrl+\ |
| `SIGUSR1` | 10 | 终止 | 用户自定义 |
| `SIGUSR2` | 12 | 终止 | 用户自定义 |
| `SIGCHLD` | 17 | 忽略 | 子进程状态变化 |
| `SIGPIPE` | 13 | 终止 | 写入无读者管道 |
| `SIGALRM` | 14 | 终止 | 定时器到期 |
| `SIGSEGV` | 11 | 终止 + Core dump | 段错误 |

```c
#include <signal.h>
#include <stdio.h>
#include <unistd.h>

void handler(int sig) {
    // 注意: 信号处理函数只能调用 async-signal-safe 函数
    write(STDOUT_FILENO, "Caught SIGUSR1\n", 15);
}

int main() {
    struct sigaction sa = {0};
    sa.sa_handler = handler;
    sigemptyset(&sa.sa_mask);
    sa.sa_flags = 0;

    if (sigaction(SIGUSR1, &sa, NULL) == -1) {
        perror("sigaction");
        return 1;
    }

    printf("PID: %d. Send SIGUSR1 to test: kill -USR1 %d\n", getpid(), getpid());
    pause();  // 等待信号
    return 0;
}
```

### 信号发送

```c
#include <signal.h>

int kill(pid_t pid, int sig);    // 发送信号给进程
int raise(int sig);               // 发送信号给自己
int sigqueue(pid_t pid, int sig, const union sigval value);  // 附带数据
```

## 共享内存 (Shared Memory)

Linux 提供两种共享内存 API：System V (`shmget`/`shmat`) 和 POSIX (`shm_open`/`mmap`)。

### POSIX 共享内存

```c
#include <stdio.h>
#include <stdlib.h>
#include <sys/mman.h>
#include <sys/stat.h>
#include <fcntl.h>
#include <unistd.h>
#include <string.h>

#define SHM_NAME "/myshm"
#define SHM_SIZE 4096

// 生产者进程
void producer() {
    int fd = shm_open(SHM_NAME, O_CREAT | O_RDWR, 0666);
    ftruncate(fd, SHM_SIZE);

    void *ptr = mmap(NULL, SHM_SIZE, PROT_READ | PROT_WRITE,
                     MAP_SHARED, fd, 0);
    close(fd);

    const char *msg = "Hello from producer!";
    memcpy(ptr, msg, strlen(msg) + 1);
    printf("Producer: wrote '%s'\n", msg);

    munmap(ptr, SHM_SIZE);
}

// 消费者进程
void consumer() {
    int fd = shm_open(SHM_NAME, O_RDONLY, 0666);
    if (fd == -1) {
        perror("shm_open (run producer first)");
        return;
    }

    void *ptr = mmap(NULL, SHM_SIZE, PROT_READ, MAP_SHARED, fd, 0);
    close(fd);

    printf("Consumer: read '%s'\n", (char *)ptr);

    munmap(ptr, SHM_SIZE);
    shm_unlink(SHM_NAME);  // 删除共享内存对象
}

int main(int argc, char *argv[]) {
    if (argc > 1 && strcmp(argv[1], "consumer") == 0) {
        consumer();
    } else {
        producer();
    }
    return 0;
}
```

```
       进程 A                         进程 B
   ┌──────────────┐              ┌──────────────┐
   │  mmap MAP_SHARED            │  mmap MAP_SHARED
   │       │                     │       │
   │       ▼                     │       ▼
   │  虚拟地址 X                 │  虚拟地址 Y
   └───────┬─────────────────────┴───────┬──────┘
           │                             │
           │     物理内存 (同一页框)       │
           └─────────────┬───────────────┘
                         ▼
                ┌──────────────────┐
                │  物理页帧 (Page)  │
                └──────────────────┘
```

> 共享内存需要配合**互斥锁**或**信号量**使用，否则存在竞态条件。

### 相关操作

| 函数 | 用途 |
|------|------|
| `shm_open()` | 创建/打开 POSIX 共享内存对象 |
| `shm_unlink()` | 删除共享内存对象 |
| `ftruncate()` | 设置共享内存大小 |
| `mmap(MAP_SHARED)` | 映射共享内存到进程地址空间 |
| `munmap()` | 解除映射 |

## 消息队列 (Message Queue)

```
       发送方                       接收方
   ┌──────────┐                ┌──────────┐
   │ mq_send  │────► 内核 MQ ────► mq_receive │
   └──────────┘     ┌──────┐   └──────────┘
                    │ msg1 │
                    │ msg2 │
                    │ msg3 │
                    └──────┘
```

```c
#include <fcntl.h>
#include <sys/stat.h>
#include <mqueue.h>
#include <stdio.h>
#include <string.h>

#define QUEUE_NAME "/myqueue"

// 发送者
void sender() {
    mqd_t mq = mq_open(QUEUE_NAME, O_CREAT | O_WRONLY, 0666, NULL);
    if (mq == (mqd_t)-1) {
        perror("mq_open");
        return;
    }

    const char *msg = "Hello via message queue!";
    if (mq_send(mq, msg, strlen(msg) + 1, 0) == -1) {
        perror("mq_send");
    }

    mq_close(mq);
}

// 接收者
void receiver() {
    struct mq_attr attr;
    mqd_t mq = mq_open(QUEUE_NAME, O_RDONLY);
    if (mq == (mqd_t)-1) {
        perror("mq_open (run sender first)");
        return;
    }

    mq_getattr(mq, &attr);
    char *buf = malloc(attr.mq_msgsize);
    unsigned int priority;

    if (mq_receive(mq, buf, attr.mq_msgsize, &priority) != -1) {
        printf("Received (prio %u): %s\n", priority, buf);
    }

    free(buf);
    mq_close(mq);
    mq_unlink(QUEUE_NAME);
}

int main(int argc, char *argv[]) {
    if (argc > 1 && strcmp(argv[1], "receiver") == 0) {
        receiver();
    } else {
        sender();
    }
    return 0;
}
```

编译: `gcc mqueue.c -lrt -o mqueue`

### POSIX 消息队列属性

| 属性 | 说明 |
|------|------|
| `mq_maxmsg` | 队列最大消息数（默认 10） |
| `mq_msgsize` | 每条消息最大字节数（默认 8192） |
| `mq_curmsgs` | 当前消息数 |
| `mq_flags` | 阻塞/非阻塞标志 |

> System V 消息队列 (`msgget`/`msgsnd`/`msgrcv`) 是更老的接口，POSIX 版本更推荐使用。

## 信号量 (Semaphore)

```c
#include <semaphore.h>
#include <fcntl.h>
#include <stdio.h>
#include <sys/mman.h>
#include <unistd.h>

#define SEM_NAME "/mysem"

// 初始化信号量（在共享内存中使用）
sem_t *init_sem() {
    sem_t *sem = sem_open(SEM_NAME, O_CREAT, 0666, 1);  // 初值 1
    if (sem == SEM_FAILED) {
        perror("sem_open");
        return NULL;
    }
    return sem;
}

void critical_section(sem_t *sem, int id) {
    sem_wait(sem);   // P 操作：semaphore--
    printf("Thread/Process %d: entered critical section\n", id);
    sleep(1);
    printf("Thread/Process %d: leaving critical section\n", id);
    sem_post(sem);   // V 操作：semaphore++
}

int main() {
    sem_t *sem = init_sem();
    if (!sem) return 1;

    pid_t pid = fork();
    if (pid == 0) {
        critical_section(sem, 1);
    } else {
        critical_section(sem, 2);
    }

    sem_close(sem);
    sem_unlink(SEM_NAME);
    return 0;
}
```

编译: `gcc sem_demo.c -lpthread -o sem_demo`

| 函数 | 操作 | 说明 |
|------|------|------|
| `sem_open()` | — | 创建/打开命名信号量 |
| `sem_init()` | — | 初始化匿名信号量（通常配合共享内存） |
| `sem_wait()` | P (down) | 信号量减 1，若为 0 则阻塞 |
| `sem_trywait()` | P (尝试) | 非阻塞版，失败返回 EAGAIN |
| `sem_timedwait()` | P (带超时) | 指定等待上限 |
| `sem_post()` | V (up) | 信号量加 1，唤醒等待者 |
| `sem_getvalue()` | — | 获取当前信号量值 |
| `sem_destroy()` | — | 销毁匿名信号量 |
| `sem_close()` | — | 关闭命名信号量 |
| `sem_unlink()` | — | 删除命名信号量 |

## Unix 域套接字

用于同一主机上进程间通信，比 TCP 环回更高效（不经过网络协议栈）。

```c
#include <sys/socket.h>
#include <sys/un.h>
#include <stdio.h>
#include <string.h>
#include <unistd.h>

void server() {
    int sfd = socket(AF_UNIX, SOCK_STREAM, 0);
    struct sockaddr_un addr = {0};
    addr.sun_family = AF_UNIX;
    strcpy(addr.sun_path, "/tmp/unix_socket");

    unlink("/tmp/unix_socket");
    bind(sfd, (struct sockaddr *)&addr, sizeof(addr));
    listen(sfd, 1);

    int cfd = accept(sfd, NULL, NULL);
    char buf[256];
    ssize_t n = read(cfd, buf, sizeof(buf) - 1);
    buf[n] = '\0';
    printf("Server received: %s\n", buf);

    write(cfd, "Hello from server", 17);
    close(cfd);
    close(sfd);
    unlink("/tmp/unix_socket");
}

void client() {
    int sfd = socket(AF_UNIX, SOCK_STREAM, 0);
    struct sockaddr_un addr = {0};
    addr.sun_family = AF_UNIX;
    strcpy(addr.sun_path, "/tmp/unix_socket");

    if (connect(sfd, (struct sockaddr *)&addr, sizeof(addr)) == -1) {
        perror("connect (run server first)");
        return;
    }

    write(sfd, "Hello from client", 17);
    char buf[256];
    ssize_t n = read(sfd, buf, sizeof(buf) - 1);
    buf[n] = '\0';
    printf("Client received: %s\n", buf);
    close(sfd);
}

int main() {
    // Run server and client in separate terminals, or
    // use fork() to test both.
    return 0;
}
```

### socketpair —— 双向管道

```c
#include <sys/socket.h>
#include <stdio.h>
#include <unistd.h>

int main() {
    int sv[2];  // sv[0] 和 sv[1] 都是可读可写的

    if (socketpair(AF_UNIX, SOCK_STREAM, 0, sv) == -1) {
        perror("socketpair");
        return 1;
    }

    pid_t pid = fork();
    if (pid == 0) {
        close(sv[0]);
        write(sv[1], "child->parent", 13);
        close(sv[1]);
    } else {
        close(sv[1]);
        char buf[128];
        ssize_t n = read(sv[0], buf, sizeof(buf) - 1);
        buf[n] = '\0';
        printf("Parent: %s\n", buf);
        close(sv[0]);
    }
    return 0;
}
```

### Unix 域套接字传递文件描述符

通过 `sendmsg()` / `recvmsg()` 配合 `SCM_RIGHTS` 辅助数据，可以将文件描述符从一个进程传递到另一个进程——这是其他 IPC 机制无法做到的。

## eventfd

轻量级事件通知机制，适用于 epoll 驱动的异步编程。

```c
#include <sys/eventfd.h>
#include <stdio.h>
#include <unistd.h>

int main() {
    int efd = eventfd(0, EFD_NONBLOCK);  // 初始计数 0
    if (efd == -1) {
        perror("eventfd");
        return 1;
    }

    // 写入 eventfd（计数 +1）
    uint64_t val = 1;
    write(efd, &val, sizeof(val));

    // 读取 eventfd（获取计数并重置为 0）
    uint64_t result;
    read(efd, &result, sizeof(result));
    printf("eventfd value: %lu\n", result);

    close(efd);
    return 0;
}
```

* 内核维护一个 64 位计数器
* `write()` 将值加到计数器（累加）
* `read()` 读取并清零（若为阻塞模式且计数为 0 则阻塞）
* 非常适合作为 epoll 的事件源

## IPC 性能对比

| 机制 | 延迟 (近似) | 最大带宽 | 典型消息大小 |
|------|------------|---------|-------------|
| pipe | ~0.5-2 μs | ~5-10 GB/s | ≤ 64 KB |
| Unix socket (AF_UNIX) | ~1-3 μs | ~4-8 GB/s | 任意 |
| shared memory | ~0.1-0.3 μs | ~50+ GB/s | 任意 |
| message queue | ~2-5 μs | ~1 GB/s | ≤ 8 KB |
| signal | ~0.5-1 μs | N/A | 仅信号编号 |
| TCP loopback | ~5-15 μs | ~20-40 GB/s | 任意 |

> 共享内存是速度最快的 IPC，但需要显式同步；Unix 域套接字是功能最丰富的选择（支持 fd 传递、双向通信）。

## 来源

* [Linux man pages — pipe(2), fifo(7), signal(7), shm_open(3), mq_overview(7), sem_overview(7), unix(7), eventfd(2)](https://man7.org/linux/man-pages/)
* [Linux Kernel source — ipc/*.c](https://elixir.bootlin.com/linux/latest/source/ipc/)
* [The Linux Programming Interface — Michael Kerrisk](https://man7.org/tlpi/)
* [POSIX IPC Overview — GNU C Library Manual](https://www.gnu.org/software/libc/manual/html_node/IPC.html)
