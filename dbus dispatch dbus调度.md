```C
static void _dbus_connection_acquire_dispatch(DBusConnection *connection)
{
    HAVE_LOCK_CHECK(connection);  // 检查是否已持有连接锁

    _dbus_connection_ref_unlocked(connection);  // 增加连接引用计数
    CONNECTION_UNLOCK(connection);  // 解锁连接

    _dbus_verbose("locking dispatch_mutex\n");
    _dbus_cmutex_lock(connection->dispatch_mutex);  // 锁定调度互斥量

    while (connection->dispatch_acquired) {
        _dbus_verbose("waiting for dispatch to be acquirable\n");
        _dbus_condvar_wait(connection->dispatch_cond, connection->dispatch_mutex);  // 等待调度可用
    }

    _dbus_assert(!connection->dispatch_acquired);  // 断言调度未被获取

    connection->dispatch_acquired = TRUE;  // 获取调度

    _dbus_verbose("unlocking dispatch_mutex\n");
    _dbus_cmutex_unlock(connection->dispatch_mutex);  // 解锁调度互斥量

    CONNECTION_LOCK(connection);  // 重新锁定连接
    _dbus_connection_unref_unlocked(connection);  // 减少连接引用计数
}
```

