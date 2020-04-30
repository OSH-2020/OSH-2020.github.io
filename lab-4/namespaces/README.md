# 隔离容器的命名空间

命名空间（namespaces）是 Linux 内核的一个特性，将部分系统资源隔离，使得不同命名空间中的进程不能互相看见其他命名空间中的同种资源。

命名空间的概念最早出现在 2002 年的 Linux 2.4.19，当时唯一的命名空间是挂载命名空间（mount namespaces），由于当时的开发者并没有想到以后会有各种各样的命名空间被加入 Linux，所以挂载命名空间就使用了 `CLONE_NEWNS`（clone new namespace）这样一个平凡的 flag。

目前最新的 Linux 内核（&gt;= 5.6）一共支持八种命名空间：

- `CLONE_NEWNS`: 挂载命名空间（mount namespaces），隔离挂载点等信息，子挂载命名空间的挂载不会向上传递到父挂载命名空间，是 Linux 内核历史上第一个命名空间的概念。（Kernel 2.4.19, 2002）
- `CLONE_NEWUTS`: Unix 主机命名空间（**UTS** namespaces, **UNIX Time-Sharing**），隔离主机名与域名等信息，不同的 UTS 命名空间可以拥有不同的主机名，在网络上呈现为多个主机。（Kernel 2.6.19, 2006）
- `CLONE_NEWIPC`: 进程间通信命名空间（**IPC** namespaces, **Inter-Process Communication**），隔离 System V IPC，不同 IPC 命名空间中的进程不能使用传统的 System V 风格的进程间通信方式，如共享内存（SHM）等。（Kernel 2.6.19, 2006）
- `CLONE_NEWPID`: 进程 ID 命名空间（PID namespaces），隔离进程的 PID 空间，不同的 PID 命名空间中的 PID 可以重复，互不影响。（Kernel 2.6.24, 2008）
- `CLONE_NEWNET`: 网络命名空间（network namespaces），虚拟化一个完整的网络栈，每个网络栈拥有一套完整的网络资源，包括网络设备（interfaces）、路由表与防火墙等。与其他命名空间不同，网络命名空间**没有**层次结构，所有的网络命名空间互相独立，每个进程只能属于一个网络命名空间，并且网络命名空间在没有进程属于它的时候**不会**自动消失。（Kernel 2.6.29, 2009）
- `CLONE_NEWUSER`: 用户命名空间（user namespaces），隔离用户与组信息，子用户命名空间中的每个用户和组（UID / GID）均映射到父用户命名空间中的一个用户和组，提供一种更好的权限隔离方式。通过将容器中的 root 用户映射到主机上的一个非特权用户，可以提升容器的安全性，这也是 LXC / LXD 实现「非特权容器」的方法。（Kernel 3.8, 2013）
- `CLONE_NEWCGROUP`: Cgroup 命名空间，类似 chroot，隔离 cgroup 层次结构，子命名空间看到的根 cgroup 结构实际上是父命名空间的一个子树。（Kernel 4.6, 2016）
- `CLONE_NEWTIME`: 系统时间命名空间，与 UTS 命名空间类似，允许不同的进程看到不同的系统时间。（Kernel 5.6, 2020）

本实验要求为你的容器隔离 NEWNS, NEWUTS, NEWIPC, NEWPID 和 NEWCGROUP 五种命名空间。由于 NEWNET 和 NEWUSER 需要不少附加工作和额外的知识基础，因此本实验不对这两种命名空间作任何要求，当然，有兴趣的同学可以自行探究（不建议）。NEWTIME 由于是在 3 月 30 日才正式发布的 Linux 5.6 内核中新增的，相关支持还不完善，因此本实验也不要求。

## 使用 clone(2) 替代 fork(2) 创建子进程

Linux 中隔离命名空间的系统调用有两个，[unshare(2)][unshare.2] 与 [clone(2)][clone.2]。前者*多数时候*用于在当前进程中创建新命名空间，且对不同命名空间（clone flags）的处理方式略有不同，容易搞混，因此我们使用 clone 系统调用在创建子进程的同时直接将子进程的各种命名空间隔离。

!!! question "你知道吗？"

    几个知名容器，如 Docker 和 [Singularity][singularity] 等所使用的 Go 语言 `os/exec` 包中的 `Command` 使用 clone 而不是 fork 作为支持该功能的系统调用。

查看 [clone 系统调用的手册][clone.2]，可以看到其原型如下：

```c
int clone(
  int (*fn)(void *),
  void *child_stack,
  int flags,
  void *arg,
  ...
  /*
    pid_t *ptid,
    void *newtls,
    pid_t *ctid
  */
);
```

本实验中，我们只关心前四个必要的参数，这里简要介绍一下：

- `fn`: 与 [fork(2)][fork.2] 不同的是，clone 出来的子进程并不从当前位置继续，而是从给出的 `fn` 函数开始运行，此时的 `fn` 函数就相当于子进程的 main 函数，若 `fn` 函数返回了（或调用了 `exit`），那么子进程就退出了，`fn` 的返回值（或传入 `exit` 的参数）就作为子进程的返回值。
- `child_stack`: 同样与 fork 系统调用不同的是，clone 不仅能创建子进程，也可以在当前进程中创建额外的线程，因此 clone 不保证创建的新程序段一定是复制的或全新的，所以当创建的是子进程的时候，需要手动为子进程指定栈的开始位置。本文下方有一个使用 clone 代替 fork 的简单的示例程序可供参考，另外 clone 系统调用的手册中 Example 一节也有一个稍微长一点的示例。
- `flags`: 这是使用 clone 来隔离命名空间的关键，通过向 flags 参数传入上面所述的 `CLONE_` 开头的标志中的一个或多个的按位或（bitwise OR），可以将 clone 出来的子进程直接放入新的命名空间中。

  与 fork 不同的是，clone 出来的子进程默认情况下不能够直接 wait，需要在 flags 中使用按位或传入 `SIGCHLD` 后才能够 wait。详情可以参考本文后面的示例程序。

- `arg`: 注意到 clone 第一个参数的原型是 `int (*)(void *)`，这是一个函数指针，指向的函数接受一个 `void *` 类型的参数。这个参数 `arg` 就是直接传入子进程的，用于单向传递信息。
- 最后三个参数写在一个块注释里，表示它们是**条件参数**，仅当前面的参数满足特定条件时才需要设定。**本实验不涉及这几个条件参数。**

  顺便一提，[open(2)][open.2] 系统调用也有一个可选参数（参数 3 `mode_t mode`），该参数存在与否的判断与上面说的方法是一样的。

  [unshare.2]: http://man7.org/linux/man-pages/man2/unshare.2.html
  [clone.2]: http://man7.org/linux/man-pages/man2/clone.2.html
  [fork.2]: http://man7.org/linux/man-pages/man2/fork.2.html
  [open.2]: http://man7.org/linux/man-pages/man2/open.2.html

  [singularity]: https://en.wikipedia.org/wiki/Singularity_(software)

### 使用 mmap(2) 为子进程分配栈空间

上面提到了，由于 clone 的特殊性，需要手动为子进程指定栈位置。一个兼容性好的做法是，使用 [mmap(2) 系统调用][mmap.2]分配一段地址用作子进程的栈。

mmap 函数的原型和参数可以从手册中查到。对于此处的用途，我们关心以下参数的值：

- `prot` 参数应当为 `PROT_READ | PROT_WRITE`，显然子进程需要读写这块内存
- `flags` 参数应当包含 `MAP_PRIVATE | MAP_ANONYMOUS | MAP_STACK`，因为栈是进程私有的（private），并且不以任何文件为基础（anonymous），最后一个参数 `MAP_STACK` 是显然的
- `fd` 应为 -1，具体请见 mmap 的 man 文档中关于 `MAP_ANONYMOUS` 的解释

由于在我们的实验环境（amd64 架构）中，栈是反向增长的，栈底的内存地址在数值上是最大的，因此在将 mmap 返回的内存地址用作栈的起始位置前，应该加上它的大小，这样实际传入的指针指向这块内存区域尾部后面，才能够正确用作子进程的栈。所以你可以在下面的示例程序中看到如下代码：

```c
void *child_stack_start = child_stack + STACK_SIZE;
```

  [mmap.2]: http://man7.org/linux/man-pages/man2/mmap.2.html

## 将子进程隔离至新的命名空间中

这一步非常简单，只需要在调用 clone 时的 flags 参数中通过按位或的方式添加一个或多个本文最开头给出的 clone flags，此时生成的子进程就在新的命名空间中了。例如：

```diff
-int ch = clone(child, child_stack_start, SIGCHLD, name);
+int ch = clone(child, child_stack_start, CLONE_NEWPID | SIGCHLD, name);
```

## 参考资料

- [Separation Anxiety: A Tutorial for Isolating Your System with Linux Namespaces](https://www.toptal.com/linux/separation-anxiety-isolating-your-system-with-linux-namespaces)

下面几个链接是在线版的 man 文档，与终端里的 `man 7 <title>` 在内容上没有任何区别，建议按需阅读。

- [namespaces(7)](http://man7.org/linux/man-pages/man7/namespaces.7.html)
- [cgroup\_namespaces(7)](http://man7.org/linux/man-pages/man7/cgroup_namespaces.7.html)
- [ipc\_namespaces(7)](http://man7.org/linux/man-pages/man7/ipc_namespaces.7.html)
- [mount\_namespaces(7)](http://man7.org/linux/man-pages/man7/mount_namespaces.7.html)
- [pid\_namespaces(7)](http://man7.org/linux/man-pages/man7/pid_namespaces.7.html)
- [uts\_namespaces(7)](http://man7.org/linux/man-pages/man7/uts_namespaces.7.html)

## clone 的示例程序

```c
#define _GNU_SOURCE    // Required for enabling clone(2)
#include <stdio.h>
#include <sched.h>     // For clone(2)
#include <signal.h>    // For SIGCHLD constant
#include <sys/mman.h>  // For mmap(2)
#include <sys/types.h> // For wait(2)
#include <sys/wait.h>  // For wait(2)

int child(void *arg) {
    printf("My name is %s\n", (char *)arg);
    return 3;
}

int main() {
    char name[] = "child";

#define STACK_SIZE (1024 * 1024) // 1 MiB
    void *child_stack = mmap(NULL, STACK_SIZE,
                             PROT_READ | PROT_WRITE,
                             MAP_PRIVATE | MAP_ANONYMOUS | MAP_STACK,
                             -1, 0);
    // Assume stack grows downwards
    void *child_stack_start = child_stack + STACK_SIZE;

    int ch = clone(child, child_stack_start, SIGCHLD, name);
    int status;
    wait(&status);
    printf("Child exited with code %d\n", WEXITSTATUS(status));
    return 0;
}
```
