下面对dbus工作流程进行分析，分为两个方面：dbus-daemon如何管理dbus服务进程注册的服务；dbus-daemon如何将dbus客户进程申请的服务转发给dbus服务进程的；

# 1. dbus-daemon如何管理dbus服务进程注册的服务

dbus-daemon和dbus服务进程之间建立的就是普通的socket连接，建立连接的过程中会注册一系列的针对这个连接的回调函数

# 2. dbus-daemon处理dbus客户进程的申请

dbus-daemon收到dbus客户进程的消息以后，提取出service name，然后根据当前server中以后的过滤规则，将消息转发给指定的服务进程