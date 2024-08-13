DBusPendingCall结构可以用于在发送带有期望回复的消息（如方法调用）时，跟踪这些消息并管理它们的回复。

### 示例代码

以下示例展示了如何使用 `pending_replies` 字段来发送一个需要回复的消息，并处理其回复：

```c
// 创建并发送一个方法调用消息
DBusMessage *msg = dbus_message_new_method_call("com.example.Service", "/com/example/Object", "com.example.Interface", "SomeMethod");
DBusPendingCall *pending;

// 发送消息并期待回复
// 	dbus_connection_send_with_reply：发送消息并期待回复。参数包括：
	//  connection：当前的DBus连接。
	//  msg：要发送的消息对象。
	//  &pending：用于存储等待回复的DBusPendingCall对象的地址。
	//  -1：超时时间（-1表示使用默认超时）。
	//  返回值：如果发送消息成功并期待回复，返回TRUE，否则返回FALSE。
if (dbus_connection_send_with_reply(connection, msg, &pending, -1)) {
    // dbus_pending_call_set_notify：设置一个回调函数，当收到回复时调用
        //  pending：等待回复的DBusPendingCall对象。
        //  reply_handler：回调函数，用于处理回复消息。
        //  NULL：用户数据，传递给回调函数。
        //  NULL：用户数据的释放函数。
    dbus_pending_call_set_notify(pending, reply_handler, NULL, NULL);
}

// 释放消息
dbus_message_unref(msg);

// 回复处理函数
void reply_handler(DBusPendingCall *pending, void *user_data) {
    DBusMessage *reply;
    
    reply = dbus_pending_call_steal_reply(pending);
    if (reply) {
        // 处理回复消息
        dbus_message_unref(reply);
    }
    
    dbus_pending_call_unref(pending);
}
```


#### 1. 创建并发送一个方法调用消息

```c
// 创建并发送一个方法调用消息
DBusMessage *msg = dbus_message_new_method_call("com.example.Service", "/com/example/Object", "com.example.Interface", "SomeMethod");
DBusPendingCall *pending;
```

- `dbus_message_new_method_call`：创建一个新的方法调用消息。参数包括：
  - `"com.example.Service"`：目标服务的总线名称。
  - `"/com/example/Object"`：目标对象路径。
  - `"com.example.Interface"`：目标接口名称。
  - `"SomeMethod"`：要调用的方法名称。
- `DBusMessage *msg`：指向新创建的消息对象的指针。
- `DBusPendingCall *pending`：用于存储等待回复的`DBusPendingCall`对象的指针。

#### 2. 发送消息并期待回复

```c
// 发送消息并期待回复
if (dbus_connection_send_with_reply(connection, msg, &pending, -1)) {
    dbus_pending_call_set_notify(pending, reply_handler, NULL, NULL);
}
```

- `dbus_connection_send_with_reply`：发送消息并期待回复。参数包括：
  - `connection`：当前的DBus连接。
  - `msg`：要发送的消息对象。
  - `&pending`：用于存储等待回复的`DBusPendingCall`对象的地址。
  - `-1`：超时时间（-1表示使用默认超时）。
- 返回值：如果发送消息成功并期待回复，返回`TRUE`，否则返回`FALSE`。

如果发送成功，`pending`将指向一个新的`DBusPendingCall`对象，该对象用于跟踪此消息的回复。

- `dbus_pending_call_set_notify`：设置一个回调函数，当收到回复时调用。参数包括：
  - `pending`：等待回复的`DBusPendingCall`对象。
  - `reply_handler`：回调函数，用于处理回复消息。
  - `NULL`：用户数据，传递给回调函数。
  - `NULL`：用户数据的释放函数。

#### 3. 释放消息

```c
// 释放消息
dbus_message_unref(msg);
```

- `dbus_message_unref`：释放消息对象，减少其引用计数。如果引用计数为0，则销毁消息对象。

#### 4. 回复处理函数

```c
// 回复处理函数
void reply_handler(DBusPendingCall *pending, void *user_data) {
    DBusMessage *reply;
    
    reply = dbus_pending_call_steal_reply(pending);
    if (reply) {
        // 处理回复消息
        dbus_message_unref(reply);
    }
    
    dbus_pending_call_unref(pending);
}
```

- `reply_handler`：当收到回复消息时调用的回调函数。参数包括：
  - `DBusPendingCall *pending`：等待回复的`DBusPendingCall`对象。
  - `void *user_data`：用户数据，此处为`NULL`。

在回调函数中：

1. `dbus_pending_call_steal_reply`：从`pending`中获取回复消息。这个函数会返回一个指向回复消息的指针，并且将回复消息从`pending`中移除，使得`pending`对象不再跟踪该回复消息。
2. 如果`reply`不为`NULL`，则处理回复消息。
3. 使用`dbus_message_unref`释放回复消息，减少其引用计数。
4. 使用`dbus_pending_call_unref`释放`pending`对象，减少其引用计数。

