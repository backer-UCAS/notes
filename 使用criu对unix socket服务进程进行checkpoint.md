首先使用 `busctl list`命令获取到提供服务的进程的pid

然后使用 `sudo lsof -p pid | grep unix` 获取到inode号

然后进行checkpoint
```shell
sudo criu dump -D /tmp/criu/service/ -t 2490 -o dump.log -v4 --external unix[110712] --shell-job
```

• -D /tmp/criu/service/：指定存储镜像的目录。
• -t 2490：指定进程 ID。
• -o dump.log：指定日志文件。
• -v4：指定详细日志级别。
• --external unix[110712]：处理外部 Unix 套接字。
• --shell-job：用于处理附加到 shell 终端的任务。


## 使用criu对socket程序进行checkpoint和restore时的困难

在CRIU中处理外部UNIX套接字时，如果一个套接字在没有其对等端（peer）的情况下被转储，那么该套接字就是外部的。在这种情况下，对等端可能还保留着一些状态信息。这种套接字的一个典型例子是D-Bus套接字。

### D-Bus 套接字的特殊性
D-Bus是一种消息总线系统，提供应用程序之间的通信机制。D-Bus守护进程（daemon）知道哪些事件被某个套接字订阅。因此，当我们试图转储或恢复D-Bus套接字时，需要特别处理，以确保在恢复后D-Bus连接的状态和事件订阅能够正确恢复。

### CRIU 插件处理
CRIU提供了插件机制来处理这些外部UNIX套接字。插件是共享库，在转储或恢复之前加载。对于D-Bus套接字，可以使用以下回调函数：

1. **转储回调函数**：
   ```c
   int cr_plugin_dump_unix_sk(int sk, int id);
   ```
   - **参数**：
     - `sk`：要转储的套接字。
     - `id`：套接字的标识符。
   - **功能**：该回调函数接收一个套接字，将其状态和必要信息转储。

2. **恢复回调函数**：
   ```c
   int cr_plugin_restore_unix_sk(int id);
   ```
   - **参数**：
     - `id`：套接字的标识符。
   - **返回值**：恢复的套接字。
   - **功能**：该回调函数根据标识符返回一个恢复的套接字。

### 详细解释
- **转储过程**：
  在转储过程中，当CRIU遇到一个外部UNIX套接字时，会调用`cr_plugin_dump_unix_sk`函数。该函数接收需要转储的套接字及其标识符，并保存该套接字的所有必要状态和信息，使得在恢复时可以准确地重建该套接字。

- **恢复过程**：
  在恢复过程中，当CRIU需要恢复一个外部UNIX套接字时，会调用`cr_plugin_restore_unix_sk`函数。该函数使用之前转储时的标识符，恢复套接字的状态和连接，使其能够继续在恢复后的环境中正常工作。

### 关键点
- **外部UNIX套接字**：如果套接字在没有其对等端的情况下被转储，则视为外部套接字。
- **D-Bus套接字**：D-Bus守护进程知道哪些事件被某个套接字订阅，因此在转储和恢复时需要确保这些信息的正确处理。
- **插件机制**：使用CRIU插件机制，通过定义转储和恢复回调函数来处理外部UNIX套接字的状态保存和恢复。

通过正确实现并调用这些回调函数，可以确保在使用CRIU转储和恢复进程时，D-Bus套接字能够正确保存其状态并在恢复后继续正常工作。