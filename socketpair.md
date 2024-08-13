`socketpair` 函数用于创建一对无名的、相互连接的套接字，称为套接字对（socket pair）。这些套接字可以用于在同一进程或不同进程之间进行双向通信。套接字对的两个端点是对等的，可以同时进行读写操作。

### 函数原型

```c
int socketpair(int domain, int type, int protocol, int sv[2]);
```

### 参数解释

- **`domain`**：协议族，指定通信域。对于本地进程间通信，通常使用 `AF_UNIX`（也称为 `AF_LOCAL`）。
- **`type`**：套接字类型，指定通信语义。常用的类型有：
  - `SOCK_STREAM`：提供有序的、可靠的、全双工的字节流通信。
  - `SOCK_DGRAM`：提供不可靠的、固定最大长度的消息通信。
  - `SOCK_SEQPACKET`：提供有序的、可靠的、固定最大长度的数据包通信。
  可以组合其他选项，如 `SOCK_CLOEXEC`，用于在执行 `exec` 系列函数时自动关闭文件描述符。
- **`protocol`**：通常为 0，表示使用默认协议。
- **`sv`**：整数数组，大小为 2，用于存储创建的两个文件描述符。

### 返回值

- 成功时返回 0，并在 `sv` 数组中保存创建的两个文件描述符。
- 失败时返回 -1，并设置 `errno` 以指示错误类型。

### 示例

在你的代码中，调用 `socketpair` 的具体方式如下：

```c
int fds[2];
retval = socketpair(AF_UNIX, SOCK_STREAM | SOCK_CLOEXEC, 0, fds);
```

### 解释

1. **`AF_UNIX`**：
   - 指定创建用于本地进程间通信的套接字。

2. **`SOCK_STREAM | SOCK_CLOEXEC`**：
   - `SOCK_STREAM`：创建一个字节流套接字，提供有序的、可靠的、全双工的通信。
   - `SOCK_CLOEXEC`：在执行 `exec` 系列函数时自动关闭这些文件描述符，确保子进程不会继承这些文件描述符。

3. **`0`**：
   - 使用默认协议。

4. **`fds`**：
   - 用于存储创建的两个文件描述符。

### 返回值处理

- **成功**：
  - 如果 `retval` 大于等于 0，表示 `socketpair` 调用成功，`fds[0]` 和 `fds[1]` 将包含两个相互连接的套接字文件描述符。
- **失败**：
  - 如果 `retval` 小于 0，表示调用失败，可以检查 `errno` 以获取错误类型。

### 错误处理

常见的错误包括：

- `EINVAL`：参数无效。例如，组合的类型标志不支持。
- `EMFILE`：进程已达到打开文件描述符的最大数量。
- `ENFILE`：系统已达到打开文件描述符的最大数量。
- `ENOMEM`：内存不足。
- `EPROTONOSUPPORT`：指定的协议不受支持。

### 使用示例

以下是一个简单的示例，展示如何使用 `socketpair` 创建一个套接字对，并进行通信：

```c
#include <sys/types.h>
#include <sys/socket.h>
#include <stdio.h>
#include <unistd.h>
#include <string.h>
#include <errno.h>

int main() {
    int fds[2];
    char buffer[256];
    int retval;

    retval = socketpair(AF_UNIX, SOCK_STREAM | SOCK_CLOEXEC, 0, fds);
    if (retval < 0) {
        perror("socketpair");
        return 1;
    }

    if (fork() == 0) {  // 子进程
        close(fds[0]);  // 关闭子进程中的父端
        write(fds[1], "Hello from child", 17);
        close(fds[1]);
    } else {  // 父进程
        close(fds[1]);  // 关闭父进程中的子端
        read(fds[0], buffer, sizeof(buffer));
        printf("Parent received: %s\n", buffer);
        close(fds[0]);
    }

    return 0;
}
```

### 总结

`socketpair` 函数用于创建一对相互连接的套接字，通常用于同一主机上进程间通信。它提供了一种高效的双向通信机制。在 D-Bus 中，使用 `socketpair` 可以实现进程间的重载通知等功能。
