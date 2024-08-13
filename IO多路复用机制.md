# 为什么要有IO多路复用机制
`select` 和 `poll` 等 I/O 多路复用机制的存在是为了解决以下问题：

### 1. 同时处理多个 I/O 事件

在许多网络和 I/O 密集型应用中，单个进程或线程需要同时处理多个 I/O 事件。例如，一个服务器需要同时处理多个客户端连接。这种情况下，简单的阻塞 I/O 操作（即等待某个 I/O 操作完成再进行下一步）会导致效率低下，因为进程会在每个 I/O 操作上阻塞，无法同时处理多个事件。

### 2. 避免大量线程或进程

一个常见的替代方案是为每个 I/O 操作创建一个线程或进程。但是，这种方法有几个缺点：
- **资源消耗**：创建和管理大量线程或进程会消耗大量系统资源，尤其是内存和 CPU。
- **复杂性**：编写和调试多线程或多进程程序复杂性较高，容易引入并发错误，如死锁和竞争条件。
- **性能瓶颈**：大量线程或进程的上下文切换会带来性能瓶颈，降低系统的整体性能。

### 3. 提高效率和可扩展性

I/O 多路复用机制允许一个单独的进程或线程同时监视多个文件描述符，通过一个系统调用等待多个 I/O 事件的发生，从而提高效率和可扩展性。


下面将详细介绍传统的 `select` 和 `poll`，并对比 `epoll`，以便更好地理解它们的区别和适用场景。

### select

#### 概述

`select` 是一种用于监视多个文件描述符的 I/O 多路复用机制。它可以检测多个文件描述符上的事件，如可读、可写或异常条件。

#### 使用方法

`select` 的基本使用步骤如下：

1. 初始化文件描述符集。
2. 调用 `select` 函数等待事件。
3. 检查哪些文件描述符发生了事件。

#### 示例代码

```c
#include <sys/select.h>
#include <unistd.h>
#include <stdio.h>

int main() {
    fd_set readfds;
    struct timeval tv;
    int retval;

    FD_ZERO(&readfds); // 清空文件描述符集
    FD_SET(STDIN_FILENO, &readfds); // 将标准输入文件描述符添加到集合中

    tv.tv_sec = 5; // 设置超时时间为5秒
    tv.tv_usec = 0;

    retval = select(STDIN_FILENO + 1, &readfds, NULL, NULL, &tv);

    if (retval == -1) {
        perror("select()");
    } else if (retval) {
        printf("Data is available now.\n");
    } else {
        printf("No data within five seconds.\n");
    }

    return 0;
}
```

#### 特点

- 监视的文件描述符数目有限（通常为 1024）[select文件描述符限制.md]。
- 每次调用 `select` 时，必须重新初始化文件描述符集，效率较低。

### poll

#### 概述

`poll` 与 `select` 类似，但在一些方面做了改进。`poll` 使用一个结构体数组来描述要监视的文件描述符及其事件。

#### 使用方法

`poll` 的基本使用步骤如下：

1. 初始化 `pollfd` 结构体数组。
2. 调用 `poll` 函数等待事件。
3. 检查哪些文件描述符发生了事件。

#### 示例代码

```c
#include <poll.h>
#include <unistd.h>
#include <stdio.h>

int main() {
    struct pollfd fds[1];
    int retval;

    fds[0].fd = STDIN_FILENO; // 标准输入文件描述符
    fds[0].events = POLLIN; // 监视读事件

    retval = poll(fds, 1, 5000); // 5000 毫秒超时

    if (retval == -1) {
        perror("poll()");
    } else if (retval) {
        if (fds[0].revents & POLLIN) {
            printf("Data is available now.\n");
        }
    } else {
        printf("No data within five seconds.\n");
    }

    return 0;
}
```

#### 特点

- 监视的文件描述符数目理论上不受限制（仅受系统内存限制）。
- 每次调用 `poll` 时，不需要重新初始化数组，但性能仍然较低，因为每次都需要遍历整个数组。

### select 和 poll 的局限性

- **效率问题**：每次调用 `select` 或 `poll` 都需要遍历所有监视的文件描述符，导致性能较低，特别是在大量文件描述符的情况下。
- **可扩展性问题**：`select` 受文件描述符数量的限制（通常为 1024），而 `poll` 虽然不受此限制，但在大量文件描述符情况下性能较差。


### epoll

#### 概述

`epoll` 是 Linux 内核中的一种高效 I/O 多路复用机制，用于处理大量文件描述符的并发 I/O 操作。它相较于传统的 `select` 和 `poll` 具有更高的性能和扩展性，特别适用于高并发场景。

#### 使用方法

`epoll` 的基本使用步骤如下：

1. 创建一个 epoll 实例。
2. 使用 `epoll_ctl` 注册要监视的文件描述符和事件。
3. 调用 `epoll_wait` 等待事件，并处理就绪的文件描述符。

#### 示例代码

```c
#include <sys/epoll.h>
#include <fcntl.h>
#include <unistd.h>
#include <stdio.h>
#include <stdlib.h>

#define MAX_EVENTS 10

int main() {
    int epoll_fd = epoll_create1(0);
    if (epoll_fd == -1) {
        perror("epoll_create1");
        exit(EXIT_FAILURE);
    }

    int fd = open("example.txt", O_RDONLY);
    if (fd == -1) {
        perror("open");
        exit(EXIT_FAILURE);
    }

    // events：事件类型，可以是以下的一个或多个：
        //  EPOLLIN：表示对应的文件描述符可以读（包括对端套接字关闭）。
        //  EPOLLOUT：表示对应的文件描述符可以写。
        //  EPOLLET：设置边缘触发（Edge Triggered）模式。
        //  EPOLLPRI：表示对应的文件描述符有紧急数据可读（带外数据）。
    struct epoll_event event;
    event.events = EPOLLIN; // 监视读事件
    event.data.fd = fd;

    // 向 epoll 实例中添加一个新的文件描述符，并指定要监视的事件
    // op：操作类型，指明对 epoll 实例进行的操作。可以是以下三个值之一：
        //  EPOLL_CTL_ADD：注册新的 fd 到 epoll 实例中。
        //  EPOLL_CTL_MOD：修改已经注册的 fd 的事件。
        //  EPOLL_CTL_DEL：从 epoll 实例中删除 fd。
    if (epoll_ctl(epoll_fd, EPOLL_CTL_ADD, fd, &event) == -1) {
        perror("epoll_ctl");
        exit(EXIT_FAILURE);
    }

    struct epoll_event events[MAX_EVENTS];
    // 返回值是已准备好进行 I/O 操作的文件描述符数量。如果返回 -1，则表示调用失败，可以通过 errno 获取错误信息
    int nfds = epoll_wait(epoll_fd, events, MAX_EVENTS, -1);
    if (nfds == -1) {
        perror("epoll_wait");
        exit(EXIT_FAILURE);
    }

    for (int i = 0; i < nfds; i++) {
        if (events[i].events & EPOLLIN) {
            printf("File descriptor %d is ready to read\n", events[i].data.fd);
        }
    }

    close(fd);
    close(epoll_fd);
    return 0;
}
```

#### 特点

- **边沿触发 (Edge Triggered, ET) 和电平触发 (Level Triggered, LT)**：
  - **ET** 模式只在事件首次发生时通知，需要用户在事件触发后及时处理并清空事件。
  - **LT** 模式则每次都通知用户文件描述符的当前状态，直到事件被处理。
  
- **性能**：
  - `epoll` 使用事件通知机制，而非遍历所有描述符，因而在高并发场景下表现优异。
  - 内核通过红黑树和链表管理文件描述符，插入、删除操作时间复杂度为 O(1)。

- **内存效率**：
  - `epoll` 实例内存开销较小，即使监视大量文件描述符。

### epoll 的优势

- **事件驱动**：`epoll` 使用事件驱动机制，不需要每次遍历所有文件描述符，性能更高。
- **可扩展性**：`epoll` 支持大量文件描述符，适用于高并发场景。
- **内存效率**：`epoll` 使用红黑树和链表管理文件描述符，内存开销小，操作效率高。


## 一些概念
### 文件描述符就绪是啥意思？

文件描述符就绪是指一个文件描述符（file descriptor）达到了可以进行 I/O 操作的状态，而不会导致操作阻塞。这些操作通常是读取、写入或处理异常条件。具体来说，不同类型的就绪状态意味着：

1. **可读（Readable）**：如果一个文件描述符表示的资源有数据可以读取，这意味着你可以调用读取操作（如 `read` 或 `recv`）而不会阻塞。例如，一个套接字有新数据到达、一个管道中有数据、或一个文件到达了结尾（EOF）。

2. **可写（Writable）**：如果一个文件描述符表示的资源可以写入数据，这意味着你可以调用写入操作（如 `write` 或 `send`）而不会阻塞。例如，一个套接字缓冲区有空闲空间可以写入数据。

3. **异常条件（Exception Conditions）**：如果一个文件描述符表示的资源上有异常条件（如错误状态或带外数据），这意味着你需要处理这些条件。例如，一个带外数据到达的套接字。

### 示例场景

#### 网络服务器的示例

考虑一个简单的网络服务器，它使用 `select` 来处理多个客户端连接。服务器需要监视每个客户端套接字是否有数据可读或是否可以写入响应数据。

```c
#include <sys/select.h>
#include <unistd.h>
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <arpa/inet.h>

#define PORT 8080
#define MAX_CLIENTS 10

int main() {
    int server_fd, new_socket, client_socket[MAX_CLIENTS], max_sd, sd;
    struct sockaddr_in address;
    fd_set readfds;

    // 初始化客户端套接字列表
    for (int i = 0; i < MAX_CLIENTS; i++) {
        client_socket[i] = 0;
    }

    // 创建服务器套接字
    if ((server_fd = socket(AF_INET, SOCK_STREAM, 0)) == 0) {
        perror("socket failed");
        exit(EXIT_FAILURE);
    }

    address.sin_family = AF_INET;
    address.sin_addr.s_addr = INADDR_ANY;
    address.sin_port = htons(PORT);

    if (bind(server_fd, (struct sockaddr *)&address, sizeof(address)) < 0) {
        perror("bind failed");
        exit(EXIT_FAILURE);
    }

    if (listen(server_fd, 3) < 0) {
        perror("listen");
        exit(EXIT_FAILURE);
    }

    int addrlen = sizeof(address);

    while (1) {
        // 清空文件描述符集合
        FD_ZERO(&readfds);

        // 将服务器套接字添加到集合中
        FD_SET(server_fd, &readfds);
        max_sd = server_fd;

        // 添加所有客户端套接字到集合中
        for (int i = 0; i < MAX_CLIENTS; i++) {
            sd = client_socket[i];

            // 如果套接字有效，则添加到集合中
            if (sd > 0)
                FD_SET(sd, &readfds);

            // 找到当前最大的文件描述符
            if (sd > max_sd)
                max_sd = sd;
        }

        // 等待事件发生
        // 监听可读事件是否就绪
        // 返回值为：
            // select 返回正数时，表示已经准备好进行 I/O 操作的文件描述符的数量。
            // 返回 0 表示在指定的超时时间内没有任何文件描述符准备好进行 I/O 操作。
            // 返回 -1 表示发生了错误，可以通过 errno 获取具体的错误信息。
        int activity = select(max_sd + 1, &readfds, NULL, NULL, NULL);

        if ((activity < 0) && (errno != EINTR)) {
            printf("select error");
        }

        // 检查服务器套接字是否有新的连接
        // FD_ISSET 是一个宏，用于检查给定的文件描述符是否在文件描述符集合中被设置。
        if (FD_ISSET(server_fd, &readfds)) {
            // accept 是一个网络编程中的系统调用，用于从监听套接字中提取第一个连接请求，并为该连接请求创建一个新的套接字。这个新套接字用于与客户端进行通信
            // 当客户端连接到服务器时，服务器通过 accept 函数接受连接请求并创建一个新的套接字，这个套接字用于服务器与该客户端进行数据通信。这个新创建的套接字（我们称之为客户端套接字）是由服务器持有的。
            if ((new_socket = accept(server_fd, (struct sockaddr *)&address, (socklen_t *)&addrlen)) < 0) {
                perror("accept");
                exit(EXIT_FAILURE);
            }

            // 将新连接添加到客户端套接字列表中
            for (int i = 0; i < MAX_CLIENTS; i++) {
                if (client_socket[i] == 0) {
                    client_socket[i] = new_socket;
                    break;
                }
            }
        }

        // 检查客户端套接字是否有数据可读
        for (int i = 0; i < MAX_CLIENTS; i++) {
            sd = client_socket[i];

            if (FD_ISSET(sd, &readfds)) {
                char buffer[1024];
                int valread = read(sd, buffer, 1024);
                if (valread == 0) {
                    // 客户端断开连接
                    close(sd);
                    client_socket[i] = 0;
                } else {
                    // 处理数据
                    buffer[valread] = '\0';
                    printf("Received: %s\n", buffer);
                    send(sd, buffer, strlen(buffer), 0);
                }
            }
        }
    }

    return 0;
}
```

### 总结

文件描述符就绪意味着文件描述符代表的资源可以执行相应的 I/O 操作而不会导致调用进程阻塞。这是 `select`、`poll` 和 `epoll` 等 I/O 多路复用机制的核心概念，使得它们能够在单线程或少量线程中高效地处理大量并发 I/O 操作。
