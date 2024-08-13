```C
typedef struct service_state {
    DBusConnection *connection;
} service_state;
```

```C
struct DBusConnection {
    DBusAtomic refcount; /**< Reference count. 用于管理 DBusConnection 对象的生命周期。 */

    DBusRMutex *mutex; /**< Lock on the entire DBusConnection 整个 DBusConnection 对象的锁，用于保护并发访问。 */

    DBusCMutex *dispatch_mutex; /**< Protects dispatch_acquired 保护 dispatch_acquired 字段的互斥锁。 */
    DBusCondVar
            *dispatch_cond; /**< Notify when dispatch_acquired is available dispatch_acquired 可用时通知的条件变量。 */
    DBusCMutex *io_path_mutex; /**< Protects io_path_acquired 保护 io_path_acquired 字段的互斥锁。 */
    DBusCondVar *io_path_cond; /**< Notify when io_path_acquired is available io_path_acquired 可用时通知的条件变量。 */

    DBusList *outgoing_messages; /**< Queue of messages we need to send, send the end of the list first. 需要发送的消息队列，末尾的消息最先发送。 */
    DBusList *incoming_messages; /**< Queue of messages we have received, end of the list received most recently. 已接收的消息队列，末尾的消息是最近接收的。 */
    DBusList *expired_messages; /**< Messages that will be released when we next unlock. 下次解锁时释放的过期消息队列。 */

    DBusMessage *message_borrowed; /**< Filled in if the first incoming message has been borrowed;
                                  *   dispatch_acquired will be set by the borrower
                                  *   如果第一个接收的消息被借用了，则该字段会被填充；借用者会设置 dispatch_acquired。
                                  */

    int n_outgoing; /**< Length of outgoing queue. 发送队列的长度。 */
    int n_incoming; /**< Length of incoming queue. 接收队列的长度。 */

    DBusCounter *outgoing_counter; /**< Counts size of outgoing messages. 统计发送消息的大小。 */

    DBusTransport *transport; /**< Object that sends/receives messages over network. 发送和接收网络消息的对象。 */
    DBusWatchList *watches; /**< Stores active watches. 存储活跃的监视器列表。 */
    DBusTimeoutList *timeouts; /**< Stores active timeouts. 存储活跃的超时列表。 */
    // 这个过滤器列表是什么时候被初始化的呢？
    // TODO:每个自定义服务的过滤器是啥时候设置的呢？
    DBusList *filter_list; /**< List of filters. 过滤器列表。 */

    DBusRMutex *
            slot_mutex; /**< Lock on slot_list so overall connection lock need not be taken 锁定 slot_list 的锁，以便无需获取整个连接锁。 */
    DBusDataSlotList slot_list; /**< Data stored by allocated integer ID 通过分配的整数 ID 存储的数据。 */

    DBusHashTable *
            pending_replies; /**< Hash of message serials to #DBusPendingCall. 将消息序列号映射到 DBusPendingCall 的哈希表。 */
    // 用于存储当前等待回复的消息的哈希表,key是message的序列号,value是

    dbus_uint32_t
            client_serial; /**< Client serial. Increments each time a message is sent 客户端序列号，每次发送消息时递增。 */
    DBusList *disconnect_message_link; /**< Preallocated list node for queueing the disconnection message 预分配的用于排队断开消息的列表节点。 */

    DBusWakeupMainFunction wakeup_main_function; /**< Function to wake up the mainloop 唤醒主循环的函数。 */
    void *wakeup_main_data; /**< Application data for wakeup_main_function 唤醒主循环函数的应用数据。 */
    DBusFreeFunction free_wakeup_main_data; /**< free wakeup_main_data 释放唤醒主循环数据的函数。 */

    DBusDispatchStatusFunction
            dispatch_status_function; /**< Function on dispatch status changes 调度状态更改时调用的函数。 */
    void *dispatch_status_data; /**< Application data for dispatch_status_function 调度状态函数的应用数据。 */
    DBusFreeFunction free_dispatch_status_data; /**< free dispatch_status_data 释放调度状态数据的函数。 */

    DBusDispatchStatus
            last_dispatch_status; /**< The last dispatch status we reported to the application 上一次报告给应用程序的调度状态。 */

    DBusObjectTree
            *objects; /**< Object path handlers registered with this connection 在此连接中注册的对象路径处理程序。 */

    char *server_guid; /**< GUID of server if we are in shared_connections, #NULL if server GUID is unknown or connection is private 如果在共享连接中，这是服务器的 GUID；如果服务器 GUID 未知或连接是私有的，则为 NULL。 */

    /* These two MUST be bools and not bitfields, because they are protected by a separate lock
   * from connection->mutex and all bitfields in a word have to be read/written together.
   * So you can't have a different lock for different bitfields in the same word.
   * 这两个必须是布尔值而不是位域，因为它们由与 connection->mutex 不同的锁保护，并且同一个字中的所有位域必须一起读/写。
   * 因此，您不能在同一个字中的不同位域上使用不同的锁。
   */
    dbus_bool_t
            dispatch_acquired; /**< Someone has dispatch path (can drain incoming queue) 表示是否有调度路径（可以处理接收队列中的消息）。 */
    dbus_bool_t
            io_path_acquired; /**< Someone has transport io path (can use the transport to read/write messages) 表示是否有传输 IO 路径（可以使用传输来读写消息）。 */

    unsigned int
            shareable : 1; /**< #TRUE if libdbus owns a reference to the connection and can return it from dbus_connection_open() more than once 如果为 TRUE，则 libdbus 拥有该连接的引用，并且可以多次从 dbus_connection_open() 返回。 */

    unsigned int
            exit_on_disconnect : 1; /**< If #TRUE, exit after handling disconnect signal 如果为 TRUE，则在处理断开信号后退出。 */

    unsigned int
            builtin_filters_enabled : 1; /**< If #TRUE, handle org.freedesktop.DBus.Peer messages automatically, whether they have a bus name or not 如果为 TRUE，则无论是否有总线名称，自动处理 org.freedesktop.DBus.Peer 消息。 */

    unsigned int
            route_peer_messages : 1; /**< If #TRUE, if org.freedesktop.DBus.Peer messages have a bus name, don't handle them automatically 如果为 TRUE，如果 org.freedesktop.DBus.Peer 消息有总线名称，则不自动处理它们。 */

    unsigned int disconnected_message_arrived : 1; /**< We popped or are dispatching the disconnected message.
                                                    * if the disconnect_message_link is NULL then we queued it, but
                                                    * this flag is whether it got to the head of the queue.
                                                    * 表示是否弹出或正在调度断开消息。如果 disconnect_message_link 为 NULL，则表示它已排队，但此标志表示是否到达队列头部。
                                                    */
    unsigned int disconnected_message_processed : 1; /**< We did our default handling of the disconnected message,
                                                    * such as closing the connection.
                                                    * 表示是否已默认处理断开消息，例如关闭连接。
                                                    */

#ifndef DBUS_DISABLE_CHECKS
    unsigned int have_connection_lock : 1; /**< Used to check locking 用于检查锁定状态。 */
#endif

#if defined(DBUS_ENABLE_CHECKS) || defined(DBUS_ENABLE_ASSERT)
    int generation; /**< _dbus_current_generation that should correspond to this connection 对应于当前连接的 _dbus_current_generation。 */
#endif
};
```

	