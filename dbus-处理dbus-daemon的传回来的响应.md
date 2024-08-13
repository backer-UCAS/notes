dbus客户端在向用户发送数据以后，dbus客户端处理dbus-daemon传会的回复有两种方式：阻塞、非阻塞：

# 1. 阻塞方式处理
### 客户端请求方法的过程


#### 步骤 1: 创建消息

首先，客户端需要创建一个 `DBusMessage` 对象，并设置消息类型为方法调用。

```c
DBusMessage *msg;
DBusMessageIter args;
dbus_uint32_t serial = 0;
char *param = "Hello, D-Bus!";

/* 创建方法调用消息 */
msg = dbus_message_new_method_call("com.example.Service", // 目标服务名称
                                   "/com/example/Object", // 目标对象路径
                                   "com.example.Interface", // 目标接口名称
                                   "MethodName"); // 方法名称

if (msg == NULL) {
    fprintf(stderr, "Message Null\n");
    exit(1);
}

/* 添加参数 */
dbus_message_iter_init_append(msg, &args);
if (!dbus_message_iter_append_basic(&args, DBUS_TYPE_STRING, &param)) {
    fprintf(stderr, "Out Of Memory!\n");
    exit(1);
}
```

#### 步骤 2: 发送消息

创建消息后，客户端需要通过 D-Bus 连接发送消息。

```c
DBusConnection *conn;
DBusError err;

/* 初始化错误 */
dbus_error_init(&err);

/* 连接到系统总线 */
conn = dbus_bus_get(DBUS_BUS_SYSTEM, &err);
if (dbus_error_is_set(&err)) {
    fprintf(stderr, "Connection Error (%s)\n", err.message);
    dbus_error_free(&err);
}
if (conn == NULL) {
    exit(1);
}

/* 发送消息并获取序列号 */
if (!dbus_connection_send(conn, msg, &serial)) {
    fprintf(stderr, "Out Of Memory!\n");
    exit(1);
}
dbus_connection_flush(conn);

printf("Request Sent\n");

/* 释放消息 */
dbus_message_unref(msg);
```

#### 步骤 3: 处理响应

客户端需要等待并处理服务端的响应。

```c
DBusMessage *reply;
DBusMessageIter reply_args;
char *reply_param;

/* 阻塞等待响应 */
reply = dbus_connection_pop_message(conn);
if (reply == NULL) {
    sleep(1);
    reply = dbus_connection_pop_message(conn);
}

/* 解析响应 */
if (dbus_message_iter_init(reply, &reply_args)) {
    if (DBUS_TYPE_STRING == dbus_message_iter_get_arg_type(&reply_args)) {
        dbus_message_iter_get_basic(&reply_args, &reply_param);
        printf("Response: %s\n", reply_param);
    } else {
        printf("Response Error: Type Mismatch\n");
    }
}

/* 释放响应消息 */
dbus_message_unref(reply);
```
# 2. 非阻塞方式处理
非阻塞处理通常涉及到注册一个回调函数，当有响应到达时，该回调函数会被调用。这样客户端可以继续执行其他任务，而不需要等待响应。

### 非阻塞处理响应示例


#### 步骤 1: 设置回调函数

首先，定义一个回调函数，当接收到响应时，这个函数会被调用。

```c
void handle_reply(DBusPendingCall *pending, void *user_data) {
    DBusMessage *reply;
    DBusMessageIter args;
    char *reply_param;

    // 从 pending call 中获取响应消息
    reply = dbus_pending_call_steal_reply(pending);
    if (reply == NULL) {
        fprintf(stderr, "Reply Null\n");
        return;
    }

    // 解析响应
    if (dbus_message_iter_init(reply, &args)) {
        if (DBUS_TYPE_STRING == dbus_message_iter_get_arg_type(&args)) {
            dbus_message_iter_get_basic(&args, &reply_param);
            printf("Response: %s\n", reply_param);
        } else {
            printf("Response Error: Type Mismatch\n");
        }
    }

    // 释放响应消息
    dbus_message_unref(reply);

    // 释放 pending call
    dbus_pending_call_unref(pending);
}
```

#### 步骤 2: 发送消息并设置回调

在发送消息时，创建一个 `DBusPendingCall` 对象，并设置回调函数。

```c
DBusMessage *msg;
DBusMessageIter args;
DBusPendingCall *pending;
dbus_uint32_t serial = 0;
char *param = "Hello, D-Bus!";

/* 创建方法调用消息 */
msg = dbus_message_new_method_call("com.example.Service", // 目标服务名称
                                   "/com/example/Object", // 目标对象路径
                                   "com.example.Interface", // 目标接口名称
                                   "MethodName"); // 方法名称

if (msg == NULL) {
    fprintf(stderr, "Message Null\n");
    exit(1);
}

/* 添加参数 */
dbus_message_iter_init_append(msg, &args);
if (!dbus_message_iter_append_basic(&args, DBUS_TYPE_STRING, &param)) {
    fprintf(stderr, "Out Of Memory!\n");
    exit(1);
}

/* 发送消息并获取 pending call */
if (!dbus_connection_send_with_reply(conn, msg, &pending, -1)) {
    fprintf(stderr, "Out Of Memory!\n");
    exit(1);
}
if (pending == NULL) {
    fprintf(stderr, "Pending Call Null\n");
    exit(1);
}
dbus_connection_flush(conn);

/* 设置回调函数 */
dbus_pending_call_set_notify(pending, handle_reply, NULL, NULL);

/* 释放消息 */
dbus_message_unref(msg);

printf("Request Sent\n");
```

#### 步骤 3: 事件循环

为了让回调函数被调用，需要运行事件循环，这样 D-Bus 库可以处理来自总线的事件。

```c
// dbus_connection_read_write_dispatch 主要作用包括以下几个方面：
//  1.	读取数据: 从底层套接字读取数据，并将其添加到连接的内部队列中。
//  2.	写入数据: 将待发送的数据写入到底层套接字。
//  3.	事件分发: 处理从底层读取的数据，并调用相应的回调函数。

/* 非阻塞事件循环 */
    while (1) {
        fd_set read_fds;
        int fd;
        struct timeval timeout = { 0, 100000 }; // 100 milliseconds

        FD_ZERO(&read_fds);
        fd = dbus_connection_get_unix_fd(conn);
        FD_SET(fd, &read_fds);

        int ret = select(fd + 1, &read_fds, NULL, NULL, &timeout);
        if (ret < 0) {
            perror("select");
            exit(1);
        }

        if (ret > 0 && FD_ISSET(fd, &read_fds)) {
            while (dbus_connection_read_write_dispatch(conn, 0)) {
                // 处理所有可用的事件
            }
        }

        // 处理其他非 D-Bus 任务
        // 如处理用户输入、更新界面等
    }

```

通过这种方式，客户端可以非阻塞地发送消息，并在响应到达时通过回调函数处理响应。这种机制允许客户端在等待响应的同时继续执行其他任务，提高了程序的并发性和响应性。

完整代码：
```C
#include <dbus/dbus.h>
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>
#include <sys/select.h>

/* 回调函数，当收到响应时被调用 */
void handle_reply(DBusPendingCall *pending, void *user_data) {
    DBusMessage *reply;
    DBusMessageIter args;
    char *reply_param;

    reply = dbus_pending_call_steal_reply(pending);
    if (reply == NULL) {
        fprintf(stderr, "Reply Null\n");
        return;
    }

    if (dbus_message_iter_init(reply, &args)) {
        if (DBUS_TYPE_STRING == dbus_message_iter_get_arg_type(&args)) {
            dbus_message_iter_get_basic(&args, &reply_param);
            printf("Response: %s\n", reply_param);
        } else {
            printf("Response Error: Type Mismatch\n");
        }
    }

    dbus_message_unref(reply);
    dbus_pending_call_unref(pending);
}

int main() {
    DBusConnection *conn;
    DBusError err;
    DBusMessage *msg;
    DBusMessageIter args;
    DBusPendingCall *pending;
    dbus_uint32_t serial = 0;
    char *param = "Hello, D-Bus!";

    /* 初始化错误对象 */
    dbus_error_init(&err);

    /* 连接到系统总线 */
    conn = dbus_bus_get(DBUS_BUS_SYSTEM, &err);
    if (dbus_error_is_set(&err)) {
        fprintf(stderr, "Connection Error (%s)\n", err.message);
        dbus_error_free(&err);
    }
    if (conn == NULL) {
        exit(1);
    }

    /* 创建方法调用消息 */
    msg = dbus_message_new_method_call("com.example.Service", // 目标服务名称
                                       "/com/example/Object", // 目标对象路径
                                       "com.example.Interface", // 目标接口名称
                                       "MethodName"); // 方法名称

    if (msg == NULL) {
        fprintf(stderr, "Message Null\n");
        exit(1);
    }

    /* 添加参数 */
    dbus_message_iter_init_append(msg, &args);
    if (!dbus_message_iter_append_basic(&args, DBUS_TYPE_STRING, &param)) {
        fprintf(stderr, "Out Of Memory!\n");
        exit(1);
    }

    /* 发送消息并获取 pending call */
    if (!dbus_connection_send_with_reply(conn, msg, &pending, -1)) {
        fprintf(stderr, "Out Of Memory!\n");
        exit(1);
    }
    if (pending == NULL) {
        fprintf(stderr, "Pending Call Null\n");
        exit(1);
    }
    dbus_connection_flush(conn);

    /* 设置回调函数 */
    dbus_pending_call_set_notify(pending, handle_reply, NULL, NULL);

    /* 释放消息 */
    dbus_message_unref(msg);

    printf("Request Sent\n");

    /* 非阻塞事件循环 */
    while (1) {
        fd_set read_fds;
        int fd;
        struct timeval timeout = { 0, 100000 }; // 100 milliseconds

        FD_ZERO(&read_fds);
        fd = dbus_connection_get_unix_fd(conn);
        FD_SET(fd, &read_fds);

        int ret = select(fd + 1, &read_fds, NULL, NULL, &timeout);
        if (ret < 0) {
            perror("select");
            exit(1);
        }

        if (ret > 0 && FD_ISSET(fd, &read_fds)) {
            while (dbus_connection_read_write_dispatch(conn, 0)) {
                // 处理所有可用的事件
            }
        }

        // 处理其他非 D-Bus 任务
        // 如处理用户输入、更新界面等
    }

    return 0;
}

```

#### 注意事项

1. **线程安全**: 确保在多线程环境中使用 D-Bus 时，适当的同步机制，以避免竞态条件。
2. **资源管理**: 确保正确地管理和释放资源，例如消息和 pending call 对象，以防内存泄漏。
3. **事件循环**: 需要保持事件循环运行，以便处理 D-Bus 消息和调用回调函数。
