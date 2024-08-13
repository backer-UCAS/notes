```C
typedef struct {
    const char *name;
    const MessageHandler *message_handlers;
    const char *extra_introspection;
    InterfaceFlags flags;
    const PropertyHandler *property_handlers;
} InterfaceHandler;
```
