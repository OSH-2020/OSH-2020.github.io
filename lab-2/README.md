# Lab 2

在实验一中，我们使用了大量的命令行操作来完成一系列工作，如编译 Linux 内核、打包 initrd、运行 QEMU 等。所有的命令行都由一个叫做外壳（shell）的程序解释并执行。本实验我们将自己编写一个简单的 shell 并理解 Linux shell 程序的工作原理。

## 编写 Shell 程序

首先，大家可以将本页底部助教编写的一个示例程序命名为 `shell.c` 并尝试编译运行它：

```shell
gcc -o sh shell.c
```

以上命令会调用 GCC 编译器编译出一个名为 `sh` 的可执行文件，你可以继续输入 `./sh` 来运行它。

这是一个非常简陋的 shell，它会提示你输入命令，你可以输入 `cd` 来切换工作目录，输入 `pwd` 显示当前目录，输入 `export` 导出环境变量，输入 `exit` 退出，或者调用系统中有的其他命令来运行，例如 `ls`, `cat` 等。

你的任务是对这个 shell 程序进行修改升级（当然，你也可以选择自己从头编写——这不是强制要求的）。

**具体的任务将在 3 月 20 日周五晚上发布。**建议有兴趣的同学可以自己调研一下现有的 shell，如 Bash，以及 [POSIX 标准对 shell 的要求](https://pubs.opengroup.org/onlinepubs/009695399/utilities/xcu_chap02.html)。掌握在 Linux 命令行环境中编译运行 C 语言程序的技巧，可以参考 Linux 用户协会编写的 Linux 101 讲义中[有关编程的章节](https://101.ustclug.org/Ch07/)。

## 示例程序

```c
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>
#include <string.h>
#include <sys/wait.h>
#include <sys/types.h>

int main() {
    /* 输入的命令行 */
    char cmd[256];
    /* 命令行拆解成的各部分，以空指针结尾 */
    char *args[128];
    int i;
    while (1) {
        /* 提示符 */
        printf("# ");
        fflush(stdin);
        fgets(cmd, 256, stdin);
        /* 清理结尾的换行符 */
        for (i = 0; cmd[i] != '\n'; i++);
        cmd[i] = '\0';
        /* 拆解命令行 */
        args[0] = cmd;
        for (i = 0; *args[i]; i++)
            for (args[i+1] = args[i] + 1; *args[i+1]; args[i+1]++)
                if (*args[i+1] == ' ') {
                    *args[i+1] = '\0';
                    args[i+1]++;
                    break;
                }
        args[i] = NULL;

        /* 没有输入命令 */
        if (!args[0])
            continue;

        /* 内建命令 */
        if (strcmp(args[0], "cd") == 0) {
            if (args[1])
                chdir(args[1]);
            continue;
        }
        if (strcmp(args[0], "pwd") == 0) {
            char wd[4096];
            puts(getcwd(wd, 4096));
            continue;
        }
        if (strcmp(args[0], "export") == 0) {
            for (i = 1; args[i] != NULL; i++) {
                /*处理每个变量*/
                char *name = args[i];
                char *value = args[i] + 1;
                while (*value != '\0' && *value != '=')
                    value++;
                *value = '\0';
                value++;
                setenv(name, value, 1);
            }
            continue;
        }
        if (strcmp(args[0], "exit") == 0)
            return 0;

        /* 外部命令 */
        pid_t pid = fork();
        if (pid == 0) {
            /* 子进程 */
            execvp(args[0], args);
            /* execvp失败 */
            return 255;
        }
        /* 父进程 */
        wait(NULL);
    }
}
```
