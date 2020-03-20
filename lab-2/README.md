# Lab 2

在实验一中，我们使用了大量的命令行操作来完成一系列工作，如编译 Linux 内核、打包 initrd、运行 QEMU 等。所有的命令行都由一个叫做外壳（shell）的程序解释并执行。本实验我们将自己编写一个简单的 shell 并理解 Linux shell 程序的工作原理。

## 编写 Shell 程序

首先，大家可以将本页底部助教编写的一个示例程序命名为 `shell.c` 并尝试编译运行它：

```shell
gcc -o sh shell.c
```

以上命令会调用 GCC 编译器编译出一个名为 `sh` 的可执行文件，你可以继续输入 `./sh` 来运行它。

这是一个非常简陋的 shell，它会提示你输入命令，你可以输入 `cd` 来切换工作目录，输入 `pwd` 显示当前目录，输入 `export` 导出环境变量，输入 `exit` 退出，或者调用系统中有的其他命令来运行，例如 `ls`, `cat` 等。

你的任务是对这个 shell 程序进行修改升级（当然，你也可以选择自己从头编写——这不是强制要求的），并完成下面的任务。

### 更健壮（推荐，但不要求）

目前的示例程序非常脆弱，无法处理不良的输入（如 `cd /` 可以运行，但将中间的一个空格改成两个就不行），也不检查各种可能的错误（`chdir`、`fork`、`exec`等系统调用都可能出错）。建议你将它改得更健壮，以方便之后的进一步开发和调试。

### 支持管道

形如 `env | wc` 这样的命令利用了「管道」语法，将两条不同的命令对接在一起同时运行。`|` 的意思是将前面的命令 `env`（输出所有环境变量）的标准输出连接到后面命令 `wc`（统计行数）的标准输入（这样就能统计出环境变量的总数）。请你观察并学习这个语法的效果，为你的 `shell` 程序实现这一功能。

你可能要用到的函数：`pipe`、`close`、`dup`。你可以运行 `man 函数名` 来查看系统自带的文档，或者上网搜索更多信息。

### 支持基本的文件重定向

形如 `ls > out.txt` 会将 `ls` 命令的输出重定向到 `out.txt` 文件中，具体地说，会将 `out.txt` 关联到程序的标准输出，然后再运行相应的命令。

类似的，`ls >> out.txt` 会将输出追加（而不是覆盖）到 `out.txt` 文件，`cat < in.txt` 会将程序的标准输入重定向到文件 `in.txt`。

请为你的 `shell` 程序实现 `>`、`>>` 和 `<` 的功能。

你可能要用到的函数：`open`、`close`、`dup`。

### 处理 Ctrl-C 的按键

在使用 shell 的时候按下 Ctrl-C 可以丢弃当前输入到一半的命令，重新显示提示符并接受新的命令输入。当有程序运行时，按下 Ctrl-C 可以终结运行中的程序，立即回到 shell 开始新的命令输入（shell 没有随程序一起结束）。

例如（`^C` 表示遇到 Ctrl-C 的输入）：

```shell
$ echo Hello
Hello
$ echo Hello^C
$ sleep 9999  # 几秒之后
^C
$             # sleep 没有运行完
$ ^C
$ sh  # 这里新开了一个嵌套的 shell
$ ^C
$ exit
$ exit
```

请为你的 `shell` 实现对 Ctrl-C 的处理。

提示：当你正确处理第一种情况后（丢弃未输入完的命令），第二种情况（终结运行中的程序）并不需要你做任何工作。

你可能要用到的函数：`signal`、`waitpid`。

### 支持 Bash 风格的 TCP 重定向（选做）

在精简的 Linux 环境中（如 Docker 容器里），常常是没有 `nc` 命令用来进行原始的 TCP 网络通信的。Bash 和一些其他 shell 支持一种特殊的重定向语法：`/dev/tcp/<host>/<port>`。

通过查看 [Bash 的 man 文档][bash.1]，`REDIRECTION` 一节，当重定向目标是下面几种路径，且操作系统没有提供这个路径时，Bash 会自行处理它们：

```text
/dev/fd/<fd>
/dev/stdin
/dev/stdout
/dev/stderr
/dev/tcp/<host>/<port>
/dev/udp/<host>/<port>
```

阅读相关文档，模拟 Bash 的行为实现 `cmd > /dev/tcp/<host>/<port>` 和 `cmd < /dev/tcp/<host>/<port>` 的重定向。

方便起见，你只需要处理 `<host>` 是典型的 IPv4 地址（即 `a.b.c.d` 的形式，其中 abcd 均为 0\~255 之间的整数）且 `<port>` 为 1\~65535 之间的整数时的情况。

你可能要用到的函数：`socket`, `connect`

  [bash.1]: https://linux.die.net/man/1/bash

### 支持基于文件描述符的文件重定向、文件重定向组合（选做）

形如 `cmd 10> out.txt` 和 `cmd 20< in.txt` 以及 `cmd 10>&20 30< in.txt` 这样的命令会将打开文件描述符 10、20 和 30 并重定向到相应的文件。请自行查找资料，实现这些文件重定向。

```shell
cmd << EOF
this
output
EOF
```

上述命令会将字符串 `"this\noutput\n"` 作为标准输入重定向给 `cmd`。请实现这种重定向方式。

`cmd <<< text` 会将 `"text\n"` 作为标准输入重定向给 `cmd`。请实现这种重定向方式。

### 更多功能（选做）

我们一般使用的 shell 非常强大，你还可以自行了解下面这些语法的含义：

```shell
echo $PATH
A=1 env
alias ll='ls -l'
echo ~
echo ~root
(sleep 10; echo aha) &
if true; then ls; fi
```

请自行选择一个或多个功能并实现它们。

## 使用 strace 工具追踪系统调用

Linux 系统中有许多用于监控、追踪系统状态的工具，如下图所示。

![horror image](https://i.stack.imgur.com/ntC1q.png)

`strace` 是一个用于监控进程系统调用的程序，例如，在 Debian 系统中使用 `strace` 追踪 `true` 命令（一个什么都不做并返回 0 的命令），可以看到类似以下输出：

```c
execve("/usr/bin/true", ["true"], 0x7ffc07ef2ae0 /* 48 vars */) = 0
brk(NULL)                               = 0x55cc2c5dc000
arch_prctl(0x3001 /* ARCH_??? */, 0x7fff1b8ec3e0) = -1 EINVAL (Invalid argument)
access("/etc/ld.so.preload", R_OK)      = -1 ENOENT (No such file or directory)
openat(AT_FDCWD, "/etc/ld.so.cache", O_RDONLY|O_CLOEXEC) = 3
fstat(3, {st_mode=S_IFREG|0644, st_size=113477, ...}) = 0
mmap(NULL, 113477, PROT_READ, MAP_PRIVATE, 3, 0) = 0x7fcc2e447000
close(3)                                = 0
openat(AT_FDCWD, "/lib/x86_64-linux-gnu/libc.so.6", O_RDONLY|O_CLOEXEC) = 3
read(3, "\177ELF\2\1\1\3\0\0\0\0\0\0\0\0\3\0>\0\1\0\0\0\360r\2\0\0\0\0\0"..., 832) = 832
lseek(3, 64, SEEK_SET)                  = 64

/* 此处省略多行 */

exit_group(0)                           = ?
+++ exited with 0 +++
```

请使用 `strace` 工具追踪你编写的 shell，找出 3 个你代码里没有出现，但出现在 strace 的输出中的系统调用（`open`, `read`, `write` 除外）。查阅资料，**简单**说说它们的功能。

## 实验要求

请按照以下目录结构组织你的 GitHub 仓库：

```
(Git)                     // Git 仓库目录
├── lab2                  // 实验二根目录
│   ├── shell.c           // 你的 Shell 的源代码
│   ├── other.c           // （可选）更多代码文件
│   ├── Makefile          // （可选）你提供的 Makefile
│   └── README.md         // 实验报告
├── .gitignore            // 这两个文件在实验一的文档里说过了
└── README.md
```

本次实验满分 10 分。你需要完成：

- 按照上面的要求组织仓库结构，提交你的 shell 源代码
  - 如果你提交的内容无法正常编译，我们会尝试修复并酌情扣除一定分数
  - 实验一中对于 Git 工具的使用要求仍然适用于本实验，即当出现以下情况时，我们会酌情扣除一定分数
    - 很大一部分的 commit 由 GitHub 网页版生成，即通过网页版文件上传的方式提交实验的文件
    - 只有寥寥无几的 commit
    - 上传了大量与实验要求无关的文件，没有设置 `.gitignore`
- 你的 shell 能够正确处理含有 1 个管道的命令，如 `ls | grep hello`
  - 你的 shell 能够正确处理含有多个管道的命令，如 `ls | cat -n | grep hello`
- 你的 shell 支持 `>`, `>>`, `<` 重定向
- 你的 shell 在遇到 Ctrl-C 时能丢弃已经输入一半的命令行，显示 `# ` 提示符并重新接受输入
- 以上必做项目全部完成可以获得 7 分。对于额外的选做项目，由助教评估确定分数，最高 4 分
  - 完成 shell 获得加分后，总分不超过 9 分
- 按照「strace 工具」一节的实验要求有效地描述了 3 个系统调用（1 分）

### 关于实验报告

尽管本实验除了「strace 工具」一节以外对实验报告并无要求，但是我们仍然推荐你在 `README.md` 中写少量内容，例如

- 你的 shell 实现可能与系统中的 `sh`（或助教期望的表现）有所不同，简要介绍这些潜在的区别，以免产生误会，导致不必要的扣分。
- 你完成了一些选做项目，也可以简单介绍，方便助教进行更准确的评估

本实验的主要内容为 shell 程序的编写，因此不必花费太多工夫在实验报告上。

### 关于选做项目

如果你按照本文档要求实现了上述三个功能，你将获得至少 7 分。如果你想获得更高的分数，请参考标记为「选做」的几个小节中介绍的 Linux shell 的常见功能并实现（不限制为本文档列出的功能，见下）。

每一项额外功能都会由助教讨论评估，但通常单个项目不会超过 2 分。

我们鼓励进行与操作系统相关的实验探究，因此过度脱离主题的项目可能不会获得加分，例如：

- 过于简单的内置命令，如 `:` (colon), `true`, `false`, `help` 等
- 严重偏离 shell 的基本功能的项目，例如你[模仿 Zsh](https://github.com/johan/zsh/blob/master/Functions/Misc/tetris) 为你的 shell 内置了一个俄罗斯方块游戏

    ![Zsh Tetris](https://i.redd.it/blfzzmopc7j41.png)

作为一个参考基准，GNU Bash 具有的功能大部分都会被认可。

## 其他说明

本实验可以使用 C 或 C++ 语言完成，如果需要使用其他语言请先询问助教。

本实验可以使用 libc, libstdc++, libm 等 C/C++ 语言常用库。如果你愿意，你也可以使用 readline 和 ncurses 等 Linux 程序常用库。使用此处没有列出的库前请询问助教。

### 关于 Makefile 的解释

Make 是一种自动化复杂项目编译过程的工具，你可以自行了解它的用法。

[这里](https://ibugone.com/p/16)有一篇博客（英文）简单介绍了使用 Make 进行编译自动化的好处与方式。

本实验中，如果你提供了 `Makefile` 文件，我们将使用它来编译你提交的程序；否则，我们将编译 `lab2/` 目录下所有的 `*.c` 和 `*.cpp` 文件。在任何情况下，你也可以在 `README.md` 中说明编译与运行相关的注意事项。

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
