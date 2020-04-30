# 限制容器中的 CPU、内存、进程数与 I/O 优先级

到这里，你可能已经完成了一个看起来还不错的容器。但我们还缺少最后一块拼图：如果容器中的进程疯狂占用系统资源的话，可能会让主机不可用（典型的例子是 fork 炸弹）。我们应该如何进行限制呢？

对于单进程的场合来说，`setrlimit()` 可以用来限制系统资源的占用，但是对于容器来说，使用 `setrlimit()` 就很捉襟见肘了。尽管 `setrlimit` 的文档声称 "A child process created via fork(2) inherits its parent's resource limits."，然而这不代表父进程和它的子进程们会共享相同的资源限制。仅仅是它们的资源限制的量是相同的。这意味着这样的情况：如果你限制容器主进程只能使用 2 GiB 内存，它创建了 2 个子进程，那么你的内存可能会被吃掉 6 GiB（而不是 2 GiB）。

对于需要限制一组进程资源的场合，使用 cgroup 更加合适。cgroup 在 Linux 2.6.24 被引入主线版本。

## 使用 cgroup

### 使用命令行工具体验

Cgroup 有自己的命令行工具，在 Ubuntu 中可以通过安装 `cgroup-tools` 软件包来获得。安装好后我们来使用 cgroup-tools 中的命令体验使用 cgroup 限制进程内存，这些命令中的一个或多个需要以 root 用户运行。

首先，在 memory 分类下创建一个新的子控制组：

```shell
cgcreate -g memory:test
```

在 test 中设置用户态内存限制为 16 MiB：

```shell
cgset -r memory.limit_in_bytes=16777216 test
```

在新创建的控制组中运行一个程序，例如 Python：

```shell
cgexec -g memory:test python3
```

然后你就可以尝试突破这个内存限制了，例如，在 64 位的系统中，尝试创建一个含有 400 万个元素的数组即可占用至少 32 MB 的内存：

```python
Python 3.8.2 (default, Mar 13 2020, 10:14:16)
[GCC 9.3.0] on linux
Type "help", "copyright", "credits" or "license" for more information.
>>> a = [0] * 4000000
```

有时候你会发现这个进程似乎还在运行，甚至还能使用更多内存，但是反应十分缓慢，这是因为我们只限制了**内存**，而超出限制的内存被内核**交换**（swap）进了硬盘。如果你没有遇到这个问题，可能是因为你的系统没有启用交换空间，此时你可以忽略下一步。

我们需要对这个子 cgroup 禁用 swap：

```shell
cgset -r memory.swappiness=0 test
```

再次尝试运行一个占用内存超过 16 MiB 的程序，你会发现它被杀掉了，这说明 cgroup 的资源限制起作用了。

```python
osh@ubuntu:~$ cgexec -g memory:test python3
Python 3.8.2 (default, Mar 13 2020, 10:14:16)
[GCC 9.3.0] on linux
Type "help", "copyright", "credits" or "license" for more information.
>>> a = [0] * 4000000
Killed
osh@ubuntu:~$ echo $?
130
osh@ubuntu:~$ 
```

测试完毕后，记得删除刚才创建的 cgroup：

```shell
cgdelete memory:test
```

### 底层细节

与 seccomp 和 capabilities 不同，cgroup 并没有专门的系统调用，而是使用文件系统的 API（也就是列目录和读写文件那些）来进行交互的。所有 cgroup 的结构都位于 `/sys/fs/cgroup` 下，该路径下的所有“文件”和其他所有用作与内核交互的接口的“文件”（如 /proc 和 /sys 下的）一样，读写它就是与内核中的 cgroup 控制器进行交互。在一个典型的 Linux 5.x 系统中，你可以在该目录下看到以下项目；

```text
blkio  cpu,cpuacct  cpuset   freezer  memory   net_cls,net_prio  perf_event  rdma     unified
cpu    cpuacct      devices  hugetlb  net_cls  net_prio          pids        systemd
```

其中一些项目的功能如下：

- blkio：即 Block I/O，控制块设备（硬盘等）操作
- cpu,cpuacct：处理器与其核算，控制 CPU 使用与分配
- devices：控制对设备的访问
- memory：控制对内存的使用
- pids：控制 PID 相关资源
- unified：这是第二代 cgroup 接口，与其他所有内容不同，在一个树中提供的所有的 cgroup 控制选项

或许你已经猜到了，创建一个新子 cgroup 就是在合适的位置进行 mkdir，而设置数值就是向其中的“文件”写入内容。下面我们抛弃 cgroup-tools，使用最普通的命令行工具重复一遍上面的工作。

在 memory 类别下创建一个名为 test 的 cgroup。请记得在上一节的最后一步中删除名为 test 的 cgroup，否则这里会出现冲突。

```shell
mkdir /sys/fs/cgroup/memory/test
```

在新创建的 cgroup 中设置内存与交换（swap）限制：

```shell
echo 16777216 > /sys/fs/cgroup/memory/test/memory.limit_in_bytes
echo 0 > /sys/fs/cgroup/memory/test/memory.swappiness
```

接下来的问题是，我们怎么模拟 cgexec(1) 呢？答案是新创建的目录中的 `cgroup.procs` 文件。读取它会获得属于该 cgroup 的所有进程的 PID，而将一个进程的 PID 写入这个文件，实际上就会将对应的进程移至该 cgroup 中。因此，为了将 python3 进程移入这个带有内存限制的 cgroup 中，我们首先要启动它，并获得它的 PID：

```python3
Python 3.8.2 (default, Mar 13 2020, 10:14:16)
[GCC 9.3.0] on linux
Type "help", "copyright", "credits" or "license" for more information.
>>> import os
>>> os.getpid()
888888
>>> 
```

在以上示例中，python3 进程的 PID 为 888888。现在新开一个终端，将这个数字写入正确的文件中；

```shell
echo 888888 >> /sys/fs/cgroup/memory/test/cgroup.procs
```

回到 python 中，尝试占用超过 16 MiB 的内存，观察 python 进程有没有被杀掉。

同样，实验结束后，我们要清理刚才创建的 cgroup：

```shell
rmdir /sys/fs/cgroup/memory/test
```

这里还有一个小问题需要注意，如果我们尝试删除一个子 cgroup 的时候还有进程被它控制怎么办？不难想到，把相关进程全部移出进对应的父 cgroup 即可：

```shell
cat /sys/fs/cgroup/memory/test/cgroup.procs > /sys/fs/cgroup/memory/cgroup.procs
```

## 实验要求

请为你的容器限制以下内容：

- 用户态内存上限（推荐值 64 MiB = 67108864）
- 内核内存上限（同样推荐 64 MiB，提示：内核内存称为 kmem）
- 禁用交换空间
- 设置 CPU 配额（`cpu.shares`）为 256
- 设置 PID 数量上限为 64
- 设置块设备 IO 权重为 50

同时，请在容器中挂载上面涉及到的 4 个 cgroup 结构，以使得容器中能够正常使用它们。

请确保当你的容器退出时你创建的子 cgroup 也能够正确删除。
