# 自己写一个简单的提供系统服务的demo程序
```C
#include <dbus/dbus.h>
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <unistd.h>

void handle_method_call(DBusConnection* conn, DBusMessage* msg) {
    if (dbus_message_is_method_call(msg, "com.example.SystemInterface", "TestMethod")) {
        DBusMessage* reply;
        DBusMessageIter args;
        const char* param = "Hello from System Bus";
        dbus_uint32_t serial = 0;

        reply = dbus_message_new_method_return(msg);
        dbus_message_iter_init_append(reply, &args);
        if (!dbus_message_iter_append_basic(&args, DBUS_TYPE_STRING, &param)) {
            fprintf(stderr, "Out Of Memory!\n");
            exit(1);
        }

        if (!dbus_connection_send(conn, reply, &serial)) {
            fprintf(stderr, "Out Of Memory!\n");
            exit(1);
        }

        dbus_connection_flush(conn);
        dbus_message_unref(reply);
    }
}

int main() {
    DBusError err;
    DBusConnection* conn;
    int ret;

    // Initialize the errors
    dbus_error_init(&err);

    // Connect to the system bus
    conn = dbus_connection_open_private("unix:path=/var/run/dbus/system_bus_socket", &err);
    if (dbus_error_is_set(&err)) {
        fprintf(stderr, "Connection Error (%s)\n", err.message);
        dbus_error_free(&err);
    }
    if (conn == NULL) {
        return 1;
    }

    if (!dbus_bus_register(conn, &err)) {
        fprintf(stderr, "Register Error (%s)\n", err.message);
        dbus_error_free(&err);
        return 1;
    }

    // Request a name on the bus
    ret = dbus_bus_request_name(conn, "com.example.SystemService", DBUS_NAME_FLAG_REPLACE_EXISTING, &err);
    if (dbus_error_is_set(&err)) {
        fprintf(stderr, "Name Error (%s)\n", err.message);
        dbus_error_free(&err);
    }
    if (ret != DBUS_REQUEST_NAME_REPLY_PRIMARY_OWNER) {
        fprintf(stderr, "Not Primary Owner (%d)\n", ret);
        return 1;
    }

    // Loop, waiting for method calls
    while (dbus_connection_read_write(conn, -1)) {
        DBusMessage* msg;

        msg = dbus_connection_pop_message(conn);

        if (msg == NULL) {
            sleep(1);
            continue;
        }

        handle_method_call(conn, msg);
        dbus_message_unref(msg);
    }

    return 0;
}

```

# 客户端程序请求获取提供服务的程序的进程id

```C
#include <dbus/dbus.h>
#include <stdio.h>
#include <stdlib.h>

void get_pid_for_service(const char* service_name) {
    DBusError err;
    DBusConnection* conn;
    DBusMessage* msg;
    DBusMessage* reply;
    dbus_uint32_t pid;

    // Initialize the error
    dbus_error_init(&err);

    // Connect to the system bus
    conn = dbus_bus_get(DBUS_BUS_SYSTEM, &err);
    if (dbus_error_is_set(&err)) {
        fprintf(stderr, "Connection Error (%s)\n", err.message);
        dbus_error_free(&err);
        return;
    }
    if (conn == NULL) {
        return;
    }

    // Create a new method call and check for errors
    msg = dbus_message_new_method_call("org.freedesktop.DBus",  // target for the method call
                                       "/org/freedesktop/DBus", // object to call on
                                       "org.freedesktop.DBus",  // interface to call on
                                       "GetConnectionUnixProcessID"); // method name
    if (msg == NULL) {
        fprintf(stderr, "Message Null\n");
        return;
    }

    // Append arguments
    DBusMessageIter args;
    dbus_message_iter_init_append(msg, &args);
    if (!dbus_message_iter_append_basic(&args, DBUS_TYPE_STRING, &service_name)) {
        fprintf(stderr, "Out Of Memory!\n");
        return;
    }

    // Send message and get a reply
    reply = dbus_connection_send_with_reply_and_block(conn, msg, -1, &err);
    dbus_message_unref(msg);

    if (dbus_error_is_set(&err)) {
        fprintf(stderr, "Error (%s)\n", err.message);
        dbus_error_free(&err);
        return;
    }

    // Read the reply
    if (!dbus_message_iter_init(reply, &args)) {
        fprintf(stderr, "Reply has no arguments!\n");
    } else if (DBUS_TYPE_UINT32 != dbus_message_iter_get_arg_type(&args)) {
        fprintf(stderr, "Argument is not uint32!\n");
    } else {
        dbus_message_iter_get_basic(&args, &pid);
        printf("The PID of service '%s' is %u\n", service_name, pid);
    }

    dbus_message_unref(reply);
    dbus_connection_unref(conn);
}

int main() {
    // Replace this with the actual service name or connection name
    const char* service_name = "com.example.SystemService";
    get_pid_for_service(service_name);
    return 0;
}
```