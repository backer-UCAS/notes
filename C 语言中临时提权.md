
是的，`setuid(0)` 需要进程以 `root` 权限运行。要安全且有效地在非 `root` 用户环境中执行需要 `root` 权限的命令，有几种方法可以考虑。以下是一个通过 `setuid` 程序实现临时提升权限的更完整和安全的方法。

### 方案：使用`setuid` 辅助程序

#### 步骤一：创建`setuid` 辅助程序

编写一个 C 程序 `run_with_privilege.c`，用于以 `root` 权限执行传递给它的命令。

```c
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>
#include <sys/types.h>
#include <sys/wait.h>

int main(int argc, char *argv[]) {
    if (argc < 2) {
        fprintf(stderr, "Usage: %s <command> [args...]\n", argv[0]);
        return 1;
    }

    // 提升权限到 root
    if (setuid(0) != 0) {
        perror("setuid");
        return 1;
    }

    // 执行传递的命令
	// 这里可能是0，可能是1，如果是shell运行，就是1，如果是C语言exec调用这个可执行文件，就是0
    execvp(argv[1], &argv[1]);

    // 如果 execvp 失败，输出错误并返回
    perror("execvp");
    return 1;
}
```

#### 步骤二：编译并设置 `setuid` 权限

1. 编译程序：

   ```sh
   gcc -o run_with_privilege run_with_privilege.c
   ```

2. 设置文件所有者为 `root` 并设置 `setuid` 位：

   ```sh
   sudo chown root:root run_with_privilege
   sudo chmod u+s run_with_privilege
   ```

   设置完毕后，可以确认文件权限：

   ```sh
   ls -l run_with_privilege
   ```

   应该看到 `s` 位设置成功：

   ```sh
   -rwsr-xr-x 1 root root 123456 运行日期和时间 run_with_privilege
   ```

#### 步骤三：在 `dbus-daemon` 中调用 `setuid` 辅助程序

在 `dbus-daemon` 的源代码中调用 `run_with_privilege` 来执行 `criu` 和 `lsof` 命令。例如：

```c
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>
#include <sys/types.h>
#include <sys/wait.h>

int execute_with_privilege(const char *cmd, char *const argv[]) {
    pid_t pid = fork();

    if (pid == -1) {
        perror("fork");
        return -1;
    }

    if (pid == 0) {
        // 子进程: 调用 run_with_privilege 来执行命令
        execvp("./run_with_privilege", argv);
        perror("execvp");
        exit(EXIT_FAILURE);
    }

    // 父进程: 等待子进程完成
    int status;
    if (waitpid(pid, &status, 0) == -1) {
        perror("waitpid");
        return -1;
    }

    if (WIFEXITED(status)) {
        return WEXITSTATUS(status);
    } else {
        return -1;
    }
}

void perform_criu_and_lsof(pid_t pid) {
    char pid_str[16];
    snprintf(pid_str, sizeof(pid_str), "%d", pid);

    // 准备执行 criu 命令
    char *criu_args[] = {"./run_with_privilege", "/usr/sbin/criu", "dump", "-t", pid_str, "-D", "/path/to/dump", "--shell-job", NULL};
    int criu_ret = execute_with_privilege("./run_with_privilege", criu_args);
    if (criu_ret != 0) {
        fprintf(stderr, "Failed to execute criu: %d\n", criu_ret);
    }

    // 准备执行 lsof 命令
    char *lsof_args[] = {"./run_with_privilege", "/usr/bin/lsof", "-p", pid_str, NULL};
    int lsof_ret = execute_with_privilege("./run_with_privilege", lsof_args);
    if (lsof_ret != 0) {
        fprintf(stderr, "Failed to execute lsof: %d\n", lsof_ret);
    }
}
```

### 示例用法

假设你在 `dbus-daemon` 的某个逻辑中需要调用 `perform_criu_and_lsof`：

```c
int main() {
    pid_t pid = 1234; // 替换为实际的 PID
    perform_criu_and_lsof(pid);

    // dbus-daemon 其他逻辑
    return 0;
}
```

### 总结

- **`setuid` 辅助程序**：通过创建一个 `setuid` 辅助程序来执行需要 `root` 权限的命令，从而避免在 `dbus-daemon` 中直接设置 `setuid`。
- **安全性**：这种方法限制了权限提升的范围，确保只有特定命令以 `root` 权限运行，降低安全风险。
- **灵活性**：可以通过 `execvp` 动态执行传递的命令和参数，适用于多种场景。