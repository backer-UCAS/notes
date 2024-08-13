```C
// TODO: 定义RulePool结构体
struct RulePool {
    /* 将非NULL的接口名称映射到非NULL的(DBusList **) */
    DBusHashTable *rules_by_iface;

    /* 不指定接口的BusMatchRules列表 */
    DBusList *rules_without_iface;
};
```

`RulePool`结构体用于管理与消息匹配相关的规则集。它包含两部分：一个哈希表`rules_by_iface`，用于存储按接口名称分类的规则列表；另一个是`rules_without_iface`，用于存储不指定接口的规则列表。这些规则用于决定消息应该发送给哪些接收者。

下面是一个简单的示例，演示如何初始化、添加规则以及使用`RulePool`：

```c
#include <dbus/dbus.h>

// 初始化RulePool
void rule_pool_init(RulePool *pool) {
    pool->rules_by_iface = _dbus_hash_table_new(DBUS_HASH_STRING, NULL, NULL);
    pool->rules_without_iface = NULL;
}

// 添加规则到RulePool
void rule_pool_add_rule(RulePool *pool, const char *interface, BusMatchRule *rule) {
    if (interface) {
        // 如果接口不为空，将规则添加到rules_by_iface
        DBusList **rules_list = _dbus_hash_table_lookup_string(pool->rules_by_iface, interface);
        if (!rules_list) {
            rules_list = dbus_new0(DBusList *, 1);
            _dbus_hash_table_insert_string(pool->rules_by_iface, interface, rules_list);
        }
        _dbus_list_append(rules_list, rule);
    } else {
        // 如果接口为空，将规则添加到rules_without_iface
        _dbus_list_append(&pool->rules_without_iface, rule);
    }
}

// 从RulePool中获取规则
DBusList* rule_pool_get_rules(RulePool *pool, const char *interface) {
    if (interface) {
        // 获取指定接口的规则列表
        return _dbus_hash_table_lookup_string(pool->rules_by_iface, interface);
    } else {
        // 获取不指定接口的规则列表
        return pool->rules_without_iface;
    }
}

// 清理RulePool
void rule_pool_clear(RulePool *pool) {
    // 清理rules_by_iface
    DBusHashIter iter;
    _dbus_hash_iter_init(pool->rules_by_iface, &iter);
    while (_dbus_hash_iter_next(&iter)) {
        DBusList **rules_list = _dbus_hash_iter_get_value(&iter);
        _dbus_list_clear(rules_list);
        dbus_free(rules_list);
    }
    _dbus_hash_table_unref(pool->rules_by_iface);

    // 清理rules_without_iface
    _dbus_list_clear(&pool->rules_without_iface);
}

// 示例使用
int main() {
    RulePool pool;
    rule_pool_init(&pool);

    BusMatchRule rule1, rule2;

    // 添加规则
    rule_pool_add_rule(&pool, "org.example.Interface1", &rule1);
    rule_pool_add_rule(&pool, NULL, &rule2);

    // 获取规则
    DBusList *rules = rule_pool_get_rules(&pool, "org.example.Interface1");
    DBusList *rules_no_iface = rule_pool_get_rules(&pool, NULL);

    // 清理RulePool
    rule_pool_clear(&pool);

    return 0;
}
```

在这个示例中：

1. `rule_pool_init` 函数初始化了一个 `RulePool` 实例。
2. `rule_pool_add_rule` 函数根据接口名称将规则添加到对应的列表中。如果接口名称为 `NULL`，则将规则添加到 `rules_without_iface` 列表中。
3. `rule_pool_get_rules` 函数根据接口名称获取对应的规则列表。
4. `rule_pool_clear` 函数清理 `RulePool` 中的所有规则和数据结构。

这个示例展示了如何使用 `RulePool` 结构体来管理和查询规则集

### 补充说明
#### 1. 规则列表是什么？怎么使用？
规则列表在 `DBus` 的上下文中是用于匹配和处理消息的规则集合。每个规则定义了特定条件，当一条消息满足这些条件时，规则所对应的操作就会被触发。这些规则主要用于决定消息应该发送给哪些接收者。

在 `DBus` 中，规则列表可以存储在 `DBusList` 结构中，每个列表节点包含一个 `BusMatchRule` 结构体。下面是一个简单的示例，展示了如何定义和使用规则列表：

### `BusMatchRule` 结构体定义

首先定义 `BusMatchRule` 结构体，它包含规则的条件：

```c
struct BusMatchRule {
    int refcount; /**< 引用计数 */
    DBusConnection *matches_go_to; /**< 规则的拥有者 */
    unsigned int flags; /**< 匹配规则的标志 */
    int message_type; /**< 消息类型 */
    char *interface; /**< 消息接口 */
    char *member; /**< 消息成员 */
    char *sender; /**< 消息发送者 */
    char *destination; /**< 消息接收者 */
    char *path; /**< 消息路径 */
    unsigned int *arg_lens; /**< 消息参数长度 */
    char **args; /**< 消息参数 */
    int args_len; /**< 消息参数的数量 */
};
```

### 初始化和添加规则

下面展示了如何初始化一个 `RulePool` 并添加规则：

```c
#include <dbus/dbus.h>

void rule_pool_init(RulePool *pool) {
    pool->rules_by_iface = _dbus_hash_table_new(DBUS_HASH_STRING, NULL, NULL);
    pool->rules_without_iface = NULL;
}

void rule_pool_add_rule(RulePool *pool, const char *interface, BusMatchRule *rule) {
    if (interface) {
        DBusList **rules_list = _dbus_hash_table_lookup_string(pool->rules_by_iface, interface);
        if (!rules_list) {
            rules_list = dbus_new0(DBusList *, 1);
            _dbus_hash_table_insert_string(pool->rules_by_iface, interface, rules_list);
        }
        _dbus_list_append(rules_list, rule);
    } else {
        _dbus_list_append(&pool->rules_without_iface, rule);
    }
}

DBusList* rule_pool_get_rules(RulePool *pool, const char *interface) {
    if (interface) {
        return _dbus_hash_table_lookup_string(pool->rules_by_iface, interface);
    } else {
        return pool->rules_without_iface;
    }
}

void rule_pool_clear(RulePool *pool) {
    DBusHashIter iter;
    _dbus_hash_iter_init(pool->rules_by_iface, &iter);
    while (_dbus_hash_iter_next(&iter)) {
        DBusList **rules_list = _dbus_hash_iter_get_value(&iter);
        _dbus_list_clear(rules_list);
        dbus_free(rules_list);
    }
    _dbus_hash_table_unref(pool->rules_by_iface);
    _dbus_list_clear(&pool->rules_without_iface);
}
```

### 示例使用

创建并使用 `RulePool` 和规则列表：

```c
int main() {
    RulePool pool;
    rule_pool_init(&pool);

    BusMatchRule rule1 = { "org.example.Interface1", "Member1", "Sender1" };
    BusMatchRule rule2 = { NULL, "Member2", "Sender2" };

    // 添加规则
    rule_pool_add_rule(&pool, rule1.interface, &rule1);
    rule_pool_add_rule(&pool, rule2.interface, &rule2);

    // 获取规则
    DBusList *rules = rule_pool_get_rules(&pool, "org.example.Interface1");
    DBusList *rules_no_iface = rule_pool_get_rules(&pool, NULL);

    // 遍历和使用规则
    DBusList *link;
    for (link = _dbus_list_get_first_link(rules); link != NULL; link = _dbus_list_get_next_link(rules, link)) {
        BusMatchRule *rule = link->data;
        // 根据规则执行相应的操作
    }

    // 清理RulePool
    rule_pool_clear(&pool);

    return 0;
}
```

### 规则的使用

当接收到一条消息时，可以根据消息的接口、成员、发送者等属性在规则列表中查找匹配的规则，并执行相应的处理逻辑。例如：

```c
void handle_message(DBusMessage *message, RulePool *pool) {
    const char *interface = dbus_message_get_interface(message);
    const char *member = dbus_message_get_member(message);
    const char *sender = dbus_message_get_sender(message);

    DBusList *rules = rule_pool_get_rules(pool, interface);
    DBusList *link;

    for (link = _dbus_list_get_first_link(rules); link != NULL; link = _dbus_list_get_next_link(rules, link)) {
        BusMatchRule *rule = link->data;
        if ((rule->member == NULL || strcmp(rule->member, member) == 0) &&
            (rule->sender == NULL || strcmp(rule->sender, sender) == 0)) {
            // 执行与规则匹配的操作
        }
    }

    // 处理不指定接口的规则
    rules = rule_pool_get_rules(pool, NULL);
    for (link = _dbus_list_get_first_link(rules); link != NULL; link = _dbus_list_get_next_link(rules, link)) {
        BusMatchRule *rule = link->data;
        if ((rule->member == NULL || strcmp(rule->member, member) == 0) &&
            (rule->sender == NULL || strcmp(rule->sender, sender) == 0)) {
            // 执行与规则匹配的操作
        }
    }
}
```

### 总结

规则列表用于存储和管理消息匹配规则，并根据这些规则决定消息应该发送给哪些接收者。通过使用 `RulePool` 结构体，可以方便地组织和查询这些规则。