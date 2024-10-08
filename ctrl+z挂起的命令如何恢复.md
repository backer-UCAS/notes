在 Linux 和 Unix 系统中，当你使用 `Ctrl + Z` 停止一个前台运行的命令时，这个命令会被挂起并转到后台。要恢复这个被挂起的命令，可以使用 `fg` 命令将其带回前台继续运行。以下是详细步骤：

### 恢复被 `Ctrl + Z` 停止的命令

1. **查看当前的作业**：
   使用 `jobs` 命令查看所有挂起的作业。这个命令会列出当前终端会话中的所有后台作业及其状态。

   ```bash
   jobs
   ```

   输出示例：

   ```bash
   [1]+  Stopped                 some_command
   ```

2. **将作业恢复到前台**：
   使用 `fg` 命令将作业恢复到前台。你可以指定作业编号来恢复特定的作业。例如，要恢复作业编号为 1 的作业：

   ```bash
   fg %1
   ```

   如果只有一个挂起的作业，也可以直接使用 `fg` 命令而不指定编号：

   ```bash
   fg
   ```

### 示例

假设你在终端中运行了一个名为 `some_command` 的命令，然后按 `Ctrl + Z` 将其停止。你可以按照以下步骤恢复这个命令：

1. **停止命令**：

   ```bash
   some_command
   ^Z
   [1]+  Stopped                 some_command
   ```

2. **查看挂起的作业**：

   ```bash
   jobs
   ```

   输出：

   ```bash
   [1]+  Stopped                 some_command
   ```

3. **恢复作业到前台**：

   ```bash
   fg %1
   ```

   或者直接使用 `fg`（如果只有一个挂起的作业）：

   ```bash
   fg
   ```

这样，`some_command` 将恢复在前台继续运行。