### I/O执行模型

在UNIX环境中，用户进程通过I/O系统调用进行I/O操作。根据Richard Stevens的“UNIX Network Programming Volume 1: The Sockets Networking”一书，I/O系统调用有五种不同的执行模型：

1. 阻塞IO（Blocking IO）
2. 非阻塞IO（Non-blocking IO）
3. 多路复用IO（IO Multiplexing）
4. 信号驱动IO（Signal Driven IO）
5. 异步IO（Asynchronous IO）

需要注意，阻塞与非阻塞关注的是进程的执行状态：
- 阻塞：进程执行系统调用后会被阻塞
- 非阻塞：进程执行系统调用后不会被阻塞

同步和异步关注的是消息通信机制：
- 同步：用户进程与操作系统（设备驱动）之间的操作是经过双方协调的，步调一致的
- 异步：用户进程与操作系统（设备驱动）之间并不需要协调，都可以随意进行各自的操作
### I/O操作的两个阶段

1. **等待数据准备好**（Waiting for the data to be ready）
2. **将数据从内核拷贝到用户进程**（Copying the data from the kernel to the process）

- 阻塞和非阻塞的区别在于内核数据还没准备好时，用户进程是否会阻塞（第一阶段是否阻塞）；

- 同步与异步的区别在于当数据从内核copy到用户空间时，用户进程是否会阻塞/参与（第二阶段是否阻塞）。

### 各种I/O模型的处理方式

#### 1. 阻塞IO（Blocking IO）

![[Pasted image 20240720230450.png]]

**特点**：
- 用户进程在两个阶段（等待数据和拷贝数据）都被阻塞。

**过程**：
1. 用户进程发出read系统调用。
2. 内核发现数据不在I/O缓冲区中，发起I/O操作并阻塞用户进程。
3. 数据准备好后，内核将数据拷贝到用户进程的缓冲区中并唤醒用户进程。
4. read系统调用完成。

#### 2. 非阻塞IO（Non-blocking IO）

![[Pasted image 20240720230537.png]]

**特点**：
- 用户进程在两个阶段都不会被阻塞，但需要主动轮询内核数据是否准备好。

**过程**：
1. 用户进程发出read系统调用。
2. 内核立即返回一个错误，指示数据未准备好。
3. 用户进程不断重复read操作。
4. 数据准备好后，内核将数据拷贝到用户进程的缓冲区中。
5. read系统调用完成。

使用系统调用 `fcntl( fd, F_SETFL, O_NONBLOCK )` 可以将对某文件句柄 `fd` 进行的读写访问设为非阻塞IO模型的读写访问。

#### 3. 多路复用IO（IO Multiplexing）

![[Pasted image 20240720231135.png]]

**特点**：
- 通过select或epoll系统调用同时处理多个文件或网络连接的I/O操作。

**过程**：
1. 用户进程发出select或epoll系统调用，关注多个文件句柄或socket。
2. 内核阻塞进程，直到某个文件句柄或socket有数据到达。(进程是被select/epoll阻塞的)
3. 用户进程发出read系统调用，内核将数据拷贝到用户进程的缓冲区中。
4. read系统调用完成。

#### 4. 信号驱动IO（Signal Driven IO）

![[Pasted image 20240720231505.png]]

**特点**：
- 采用信号机制，回调函数处理数据，开发和调试较为复杂。

**过程**：
1. 用户进程发出read系统调用并注册信号处理函数。
2. 数据准备好后，内核发送信号给用户进程。
3. 用户进程在信号处理函数中调用read系统调用，内核将数据拷贝到缓冲区中。
4. read系统调用完成。

#### 5. 异步IO（Asynchronous IO）

![[Pasted image 20240720231547.png]]

**特点**：
- 用户进程在两个阶段都不会被阻塞，内核处理数据并通知用户进程操作完成。

**过程**：
1. 用户进程发出async_read系统调用并立即返回。
2. 内核等待数据准备好，并将数据拷贝到用户进程的缓冲区中。
3. 数据拷贝完成后，内核通知用户进程I/O操作完成。

### 对比总结

- **阻塞IO**：用户进程在I/O操作的两个阶段都被阻塞。
- **非阻塞IO**：用户进程不会被阻塞，但需要不断轮询内核数据是否准备好。
- **多路复用IO**：单进程处理多个I/O操作，通过select或epoll进行事件驱动。
- **信号驱动IO**：通过信号机制和回调函数处理I/O操作。
- **异步IO**：用户进程在I/O操作的两个阶段都不会被阻塞，内核完成所有操作后通知用户进程。

### 阻塞与非阻塞、同步与异步

- **阻塞**：进程在I/O系统调用后会被阻塞，直到操作完成。
- **非阻塞**：进程在I/O系统调用后不会被阻塞，可以继续执行其他操作。
- **同步**：用户进程与操作系统（设备驱动）之间的操作是经过协调的，步调一致的。
- **异步**：用户进程与操作系统（设备驱动）之间的操作不需要协调，彼此独立进行。

**总结**：前四种模型（阻塞IO、非阻塞IO、多路复用IO、信号驱动IO）都属于同步IO，因为用户进程在数据拷贝阶段需要参与。异步IO则完全不需要用户进程参与，内核完成所有操作后通知用户进程。

### 补充说明

按照上面的分析，多路复用IO（IO multiplexing）确实可以被归类为阻塞(被select/epoll系统调用阻塞)+同步IO，那为什么多路复用IO和普通IO比起来还是很高效呢？

	`select/epoll` 的优势并不会导致单个文件或socket的I/O访问性能更好，而是在有很多个文件或socket的I/O访问情况下，其总体效率会高。

多路复用IO（IO multiplexing）之所以高效，主要是因为它在处理大量I/O操作时能够显著提高资源利用率和响应速度。以下是具体原因：

### 1. 减少了系统调用的开销

在没有多路复用的情况下，每个文件描述符（如socket）的I/O操作都需要一个独立的线程或进程来管理。每个线程或进程都需要通过阻塞的read或write系统调用来处理I/O，这样会导致大量的上下文切换和系统调用开销。

多路复用IO通过一个单独的系统调用（如select或epoll）来管理多个文件描述符，从而减少了系统调用的次数，降低了上下文切换的开销。

### 2. 更少的资源消耗

传统的阻塞IO模型通常需要为每个连接创建一个线程，这在高并发场景下会消耗大量的系统资源（CPU、内存）。线程数过多还会导致线程调度开销增大。

多路复用IO模型可以通过一个单独的线程或少量线程管理大量的文件描述符，显著减少了线程创建和调度的开销，提高了系统的资源利用率。

### 3. 更好的可扩展性

多路复用IO模型可以更好地应对大量连接和高并发请求。由于select或epoll系统调用可以同时监控多个文件描述符，当任何一个文件描述符有事件发生时，系统调用会返回所有有事件的描述符列表，用户进程可以一次性处理多个I/O操作，提高了整体的响应速度和吞吐量。

### 4. 更高的I/O效率

虽然select在处理大量文件描述符时会有性能瓶颈（因为每次调用都需要遍历所有文件描述符），但epoll解决了这个问题。epoll使用事件通知机制和回调函数，不需要遍历整个文件描述符集合，因此在大量并发连接场景下性能更高。

### 5. 非阻塞的I/O处理

多路复用IO模型通常配合非阻塞IO使用。非阻塞IO可以在数据未准备好时立即返回，不会阻塞用户进程，使得用户进程可以继续处理其他任务。这样，即使某些I/O操作需要等待，也不会阻塞整个进程。

### 6. 事件驱动编程

多路复用IO模型通常采用事件驱动编程模式。事件驱动模型使得程序可以在一个主循环中处理I/O事件，代码结构更加清晰，逻辑更加简单，有助于实现高效的并发处理。

### 总结

多路复用IO高效的原因在于它通过单个系统调用管理多个文件描述符，减少了系统调用次数和上下文切换开销，降低了资源消耗，提高了系统的资源利用率和I/O效率，特别是在高并发场景下，能够显著提升系统的可扩展性和响应速度。因此，多路复用IO模型在网络服务器和高并发I/O场景中被广泛使用。