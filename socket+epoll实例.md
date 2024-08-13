下面是一个使用 `epoll` 实现的简单客户端和服务器示例，展示如何使用 `epoll` 进行高效的 I/O 多路复用。客户端和服务器通过套接字进行通信。

### 服务器代码

服务器将监听一个端口，接受多个客户端的连接，并处理客户端发送的数据。

```c
// server.c
#include <sys/epoll.h>
#include <netinet/in.h>
#include <fcntl.h>
#include <unistd.h>
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <errno.h>

#define PORT 8080
#define MAX_EVENTS 10
#define BUFFER_SIZE 1024

// [非阻塞套接字.md]
void set_nonblocking(int fd) {
    int flags = fcntl(fd, F_GETFL, 0);
    fcntl(fd, F_SETFL, flags | O_NONBLOCK);
}

int main() {
    int server_fd, new_socket, epoll_fd;
    struct sockaddr_in address;
    struct epoll_event event, events[MAX_EVENTS];
    int addrlen = sizeof(address);

    // 创建服务器套接字
    if ((server_fd = socket(AF_INET, SOCK_STREAM, 0)) == 0) {
        perror("socket failed");
        exit(EXIT_FAILURE);
    }

    // 设置服务器地址
    address.sin_family = AF_INET;
    address.sin_addr.s_addr = INADDR_ANY;
    address.sin_port = htons(PORT);

    // 绑定套接字到地址和端口
    if (bind(server_fd, (struct sockaddr *)&address, sizeof(address)) < 0) {
        perror("bind failed");
        exit(EXIT_FAILURE);
    }

    // 监听端口
    // listen 函数告诉操作系统这个套接字将用于接受连接请求，并且可以有多少个连接请求可以排队等待被接受。
    if (listen(server_fd, 10) < 0) {
        perror("listen");
        exit(EXIT_FAILURE);
    }

    // 创建 epoll 实例
    if ((epoll_fd = epoll_create1(0)) == -1) {
        perror("epoll_create1");
        exit(EXIT_FAILURE);
    }

    // 设置服务器套接字为非阻塞模式
    // 将 server_fd 对应的服务器套接字设置为非阻塞模式。具体来说，这意味着任何对该套接字的 I/O 操作（如 accept、read、write 等）在不能立即完成时，不会使调用线程阻塞，而是立即返回，并设置相应的错误码。
    set_nonblocking(server_fd);

    // 将服务器套接字添加到 epoll 实例中
    event.events = EPOLLIN;
    event.data.fd = server_fd;
    if (epoll_ctl(epoll_fd, EPOLL_CTL_ADD, server_fd, &event) == -1) {
        perror("epoll_ctl: server_fd");
        exit(EXIT_FAILURE);
    }

    while (1) {
        // events：就绪的事件
        int nfds = epoll_wait(epoll_fd, events, MAX_EVENTS, -1);
        if (nfds == -1) {
            perror("epoll_wait");
            exit(EXIT_FAILURE);
        }

        for (int n = 0; n < nfds; ++n) {
            // 在 epoll 服务器代码中，这样的判断用于区分不同类型的事件：服务器套接字上的事件和客户端套接字上的事件。
            // 	服务器套接字 (server_fd)：
                //  服务器套接字用于监听新的客户端连接请求。当有新的客户端连接请求到达时，服务器套接字会变为可读状态。
            //  客户端套接字：
                //  每个已连接的客户端都会有一个对应的套接字。客户端套接字用于与客户端进行数据通信。当有数据可读或可写时，客户端套接字会变为可读或可写状态
            if (events[n].data.fd == server_fd) {
                // 处理新连接
                // accept用于取出第一个连接
                while ((new_socket = accept(server_fd, (struct sockaddr *)&address, (socklen_t *)&addrlen)) > 0) {
                    set_nonblocking(new_socket);
                    event.events = EPOLLIN | EPOLLET;
                    event.data.fd = new_socket;
                    if (epoll_ctl(epoll_fd, EPOLL_CTL_ADD, new_socket, &event) == -1) {
                        perror("epoll_ctl: new_socket");
                        exit(EXIT_FAILURE);
                    }
                    printf("New connection on socket %d\n", new_socket);
                }
                if (errno != EAGAIN && errno != EWOULDBLOCK) {
                    perror("accept");
                    exit(EXIT_FAILURE);
                }
            } else {
                // 处理客户端数据
                int client_fd = events[n].data.fd;
                char buffer[BUFFER_SIZE];
                int bytes_read = read(client_fd, buffer, sizeof(buffer));
                if (bytes_read > 0) {
                    buffer[bytes_read] = '\0';
                    printf("Received: %s\n", buffer);
                    // 回送数据给客户端
                    write(client_fd, buffer, bytes_read);
                } else if (bytes_read == 0) {
                    // 客户端关闭连接
                    close(client_fd);
                    printf("Connection closed on socket %d\n", client_fd);
                } else {
                    perror("read");
                }
            }
        }
    }

    close(server_fd);
    close(epoll_fd);
    return 0;
}
```

### 客户端代码

客户端将连接服务器，发送消息并接收回显的消息。

```c
// client.c
#include <sys/types.h>
#include <sys/socket.h>
#include <netinet/in.h>
#include <arpa/inet.h>
#include <unistd.h>
#include <stdio.h>
#include <stdlib.h>
#include <string.h>

#define PORT 8080
#define SERVER_IP "127.0.0.1"
#define BUFFER_SIZE 1024

int main() {
    int sock = 0;
    struct sockaddr_in serv_addr;
    char *message = "Hello from client";
    char buffer[BUFFER_SIZE] = {0};

    // 创建客户端套接字
    if ((sock = socket(AF_INET, SOCK_STREAM, 0)) < 0) {
        perror("Socket creation error");
        return -1;
    }

    serv_addr.sin_family = AF_INET;
    serv_addr.sin_port = htons(PORT);

    // 转换 IP 地址
    if (inet_pton(AF_INET, SERVER_IP, &serv_addr.sin_addr) <= 0) {
        perror("Invalid address / Address not supported");
        return -1;
    }

    // 连接服务器
    if (connect(sock, (struct sockaddr *)&serv_addr, sizeof(serv_addr)) < 0) {
        perror("Connection failed");
        return -1;
    }

    // 发送消息
    send(sock, message, strlen(message), 0);
    printf("Message sent\n");

    // 接收服务器回显的消息
    int valread = read(sock, buffer, BUFFER_SIZE);
    if (valread > 0) {
        buffer[valread] = '\0';
        printf("Received from server: %s\n", buffer);
    }

    // 关闭套接字
    close(sock);
    return 0;
}
```

### 编译和运行

```makefile
# Makefile

# 编译器
CC = gcc

# 编译选项
CFLAGS = -Wall -O2

# 可执行文件
TARGETS = server client

# 源文件
SRCS = $(wildcard *.c)

# 目标文件
OBJS = $(SRCS:.c=.o)

# 生成所有目标
all: $(TARGETS)

# 生成服务器
server: server.o
	$(CC) $(CFLAGS) -o $@ $^

# 生成客户端
client: client.o
	$(CC) $(CFLAGS) -o $@ $^

# 生成目标文件
%.o: %.c
	$(CC) $(CFLAGS) -c $< -o $@

# 清理
clean:
	rm -f $(OBJS) $(TARGETS)

# 伪目标
.PHONY: all clean

```

编译
```bash
make
```

3. 运行服务器：
```sh
./server
```

4. 运行客户端：
```sh
./client
```

### 解释

- **服务器代码**：
  - 创建服务器套接字并绑定到指定端口。
  - 使用 `epoll` 进行 I/O 多路复用，处理多个客户端连接。
  - 服务器套接字和每个客户端套接字都设置为非阻塞模式。
  - 使用 `epoll_wait` 等待事件，处理新连接和客户端数据。

- **客户端代码**：
  - 创建客户端套接字并连接到服务器。
  - 发送消息到服务器并接收回显的消息。
  - 关闭套接字。

通过这种方式，服务器能够高效地处理多个客户端连接，同时保证每个客户端的交互都是非阻塞的。

### 一些说明
```C
int nfds = epoll_wait(epoll_fd, events, MAX_EVENTS, -1);
```
events中的IO事件一定是准备好的，理论上来说，是一定可以立刻进行read/write的，但是还是需要设置成阻塞，因为：

**1. 防止单个I/O操作阻塞整个事件循环**

即使epoll_wait返回的事件表示某个文件描述符已经准备好进行read或write操作，实际的I/O操作仍可能因为各种原因而无法立即完成。例如：
• 读操作：即使文件描述符可读，read操作可能读取到部分数据后就被阻塞，因为没有更多的数据可用。
• 写操作：即使文件描述符可写，write操作可能写入部分数据后就被阻塞，因为缓冲区已满。

如果文件描述符是阻塞的，那么这些I/O操作会阻塞整个事件循环，导致其他已经准备好的I/O事件得不到及时处理，从而降低系统的整体性能。

**2. 提高I/O处理的灵活性和效率**

设置文件描述符为非阻塞模式，可以使程序在无法立即完成I/O操作时立即返回，让事件循环继续处理其他事件。这种方式确保了即使某些I/O操作需要等待，事件循环仍能继续运行，不会因为单个阻塞的I/O操作而停顿。

**3. 避免死锁和僵局**

在复杂的网络应用中，阻塞的I/O操作容易导致死锁或僵局。例如，当一个线程在等待一个阻塞的读操作完成，而另一个线程在等待一个阻塞的写操作完成时，可能会出现两个线程相互等待对方释放资源的情况。非阻塞模式可以有效避免这种问题。