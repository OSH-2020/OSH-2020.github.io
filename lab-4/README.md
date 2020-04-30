# Lab 4

**容器**是近年来非常热门的一个概念。它通过操作系统内核提供的隔离技术，实现轻量级的虚拟化环境。目前，它在软件的开发、部署等方面有着非常广泛的应用。

最常见的几种容器，如 Docker，都充分利用了多种 Linux 的特性，因此毫不夸张地说，容器是属于 Linux 的。本实验中我们将动手编写一个容器实现，理解 Linux 中的容器技术。

本实验由于与操作系统深度结合，因此无法在 vlab 平台上进行，请同学们在自己的虚拟机上完成。以下资源可以使用：

- [VMware Workstation 15.5 软件](https://vlab.ustc.edu.cn/downloads/VMware-workstation-full-15.5.0-14665864.exe)
- [Linux 101 提供的 Xubuntu 虚拟机镜像](https://101.lug.ustc.edu.cn/Ch01/#get-vm-images)

## 准备工作

你需要一个 C 编译器和最基本的 C 语言库，相信你已经在前几个实验中使用熟练了，因此略过。

Linux 容器的典型实现使用了至少五种特性：[命名空间（namespaces）][namespaces.7]、[`pivot_root`][pivot_root.2]、[能力（capabilities）][capabilities.7]、[SecComp（Secure Computing）][seccomp.2]和[Cgroup（Control Groups）][cgroups.7]

  [namespaces.7]: http://man7.org/linux/man-pages/man7/namespaces.7.html
  [pivot_root.2]: http://man7.org/linux/man-pages/man2/pivot_root.2.html
  [capabilities.7]: http://man7.org/linux/man-pages/man7/capabilities.7.html
  [seccomp.2]: http://man7.org/linux/man-pages/man2/seccomp.2.html
  [cgroups.7]: http://man7.org/linux/man-pages/man7/cgroups.7.html

其中，命名空间使用 clone(2) 和 unshare(2) 的 flags 来完成，cgroup 使用文件系统的 API（即通过创建删除目录和读写文件的方式与 cgroup 系统交互），而 seccomp 和 capabilities 使用原生的系统调用（该系统调用非常复杂，因此我们使用相关库来完成这两项功能）。

在阅读本次实验文档前，你需要理解如 fork(2) 这样的表达方式。你可以参考 Linux 用户协会的 Linux 101 讲义附录中的[相关章节](https://101.lug.ustc.edu.cn/Appendix/man/#man-sections)来进行初步的了解。

一个重要的提示是，实验二中学习使用的 strace 工具在本实验中非常有帮助。

## 实验内容

- [构建容器的根文件系统（rootfs）](rootfs/README.md)

在你准备好 rootfs 之后，可以编译运行该页面底部的示例程序体验“光杆容器”。下面的各项任务是对其进行修改升级（你也可以选择自己从头编写一个）。由于下面几个项目中有互相依赖的任务，或者是进行功能上的限制，因此请尽量按顺序完成。

- [使用 clone(2) 代替 fork(2)，并隔离命名空间](namespaces/README.md)
- [在容器中使用 mount(2) 与 mknod(2) 挂载必要的文件系统结构](mounts/README.md)
- [使用 pivot\_root(2) 替代 chroot(2) 完成容器内根文件系统的切换](pivot_root/README.md)
- [使用 libcap 为容器缩减不必要的能力（capabilities）](capabilities/README.md)
- [使用 libseccomp 对容器中的系统调用进行白名单过滤](seccomp/README.md)
- [使用 cgroup 限制容器中的 CPU、内存、进程数与 I/O 优先级](cgroup/README.md)

## 实验要求

请按照以下目录结构组织你的 GitHub 仓库：

```
(Git)                     // Git 仓库目录
├── lab4                  // 实验四根目录
│   ├── *.c               // 你的容器源代码
│   ├── *.cpp             // 你的容器源代码
│   ├── *.go              // 你的容器源代码
│   ├── *.rs              // 你的容器源代码
│   ├── Makefile          // （可选）你提供的 Makefile
│   └── README.md         // 简要的说明
├── .gitignore            // 这两个文件在实验一的文档里说过了
└── README.md
```

本次实验满分为 10 分，你需要完成：

- 使用 Linux 命名空间隔离子进程的特性（不要求隔离网络命名空间、用户命名空间与时间命名空间）（1 分）
- 正确挂载容器内的 `/proc`, `/sys`, `/tmp`, `/dev`，并在 `/dev` 下创建必要的节点（3 分）
- 使用 pivot\_root(2) 替代 chroot(2) 完成容器内根文件系统的切换，并从主机上隐藏容器的挂载点（4 分）
- 为容器中的进程移除一些能力（drop capabilities）（2 分）
- 使用 SecComp 限制容器中能够进行的系统调用（2 分）
- 使用 cgroup 限制容器中的系统资源使用，并在容器中按要求挂载四个 cgroup 控制器（2 分）
- 完成思考题（2 分）

本实验可以使用 C / C++ / Go / Rust 语言完成，推荐使用 C 或 C++ 语言。使用其他编程语言前请询问助教。方便起见，你的程序应当支持下面的命令行参数：

```shell
./lab4 <rootfs> <command> [args...]
```

即 `argv[1]` 表示 rootfs 的路径（没有 / 结尾），`argv[2]` 开始表示作为容器中的 PID 1 运行的程序及参数，可以假设 argc ≥ 3。

除了运行容器中的第一个程序（即容器中的 PID 1，应由命令行给出）之外，你的程序不应该调用其他程序来完成任何功能。你可以将「调用其他程序」理解为「除了运行目标命令之外的 execve 系统调用」。请注意，一些库函数（如 system(3) 和 popen(3) 等）都会调用额外的程序，这些调用过程通常包含了 fork+execve 的系统调用。

与实验二和三相似，本实验不要求实验报告，但是推荐简单描述你的实现，并对你使用的「奇技淫巧」简单介绍，以免对助教造成误会等。

本次实验满分为 10 分，但根据以上内容，实际可获得的分数上限为 **16** 分，超出 10 分的部分将减半计入实验总分。

例如，若你根据以上标准获得了 16 分，那么本次实验计入总分的分数为 13 分，这将等价于本课程总评成绩的 6.5 分。

## 思考题

1. 用于限制进程能够进行的系统调用的 seccomp 模块实际使用的系统调用是哪个？用于控制进程能力的 capabilities 实际使用的系统调用是哪个？尝试说明为什么本文最上面认为「该系统调用非常复杂」。
2. 当你用 cgroup 限制了容器中的 CPU 与内存等资源后，容器中的所有进程都不能够超额使用资源，但是诸如 htop 等「任务管理器」类的工具仍然会显示主机上的全部 CPU 和内存（尽管无法使用）。查找资料，说明原因，尝试**提出**一种解决方案，使任务管理器一类的程序能够正确显示被限制后的可用 CPU 和内存（不要求实现）。
