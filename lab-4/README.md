# Lab 4

**本实验文档尚未完稿**，仅供有兴趣的同学提前预习实验四。

**容器**是近年来非常热门的一个概念。它通过操作系统内核提供的隔离技术，实现轻量级的虚拟化环境。目前，它在软件的开发、部署等方面有着非常广泛的应用。

最常见的几种容器，如 Docker，都充分利用了多种 Linux 的特性，因此毫不夸张地说，容器是属于 Linux 的。本实验中我们将动手编写一个容器实现，理解 Linux 中的容器技术。

## 准备工作

你需要一个 C 编译器和最基本的 C 语言库，相信你已经在前几个实验中使用熟练了，因此略过。

Linux 容器的典型实现使用了至少五种特性：[命名空间（namespaces）][namespaces.7]、[`pivot_root`][pivot_root.2]、[能力（capabilities）][capabilities.7]、[SecComp（Secure Computing）][seccomp.2]和[Cgroup（Control Groups）][cgroups.7]

  [namespaces.7]: http://man7.org/linux/man-pages/man7/namespaces.7.html
  [pivot_root.2]: http://man7.org/linux/man-pages/man2/pivot_root.2.html
  [capabilities.7]: http://man7.org/linux/man-pages/man7/capabilities.7.html
  [seccomp.2]: http://man7.org/linux/man-pages/man2/seccomp.2.html
  [cgroups.7]: http://man7.org/linux/man-pages/man7/cgroups.7.html

其中，命名空间使用 `clone(2)` 和 `unshare(2)` 的 flags 来完成，cgroup 使用文件系统的 API（即通过创建删除目录和读写文件的方式与 cgroup 系统交互），而 seccomp 和 capabilities 使用额外的库（`libseccomp` 和 `libcap`）来与内核交互。

为了能够使用相关功能，你需要安装额外的开发时依赖文件。对于 Ubuntu 和 Debian 系统，使用

```shell
sudo apt install libseccomp-dev libcap-dev
```

即可完成安装。

在阅读本次实验文档前，你需要理解如 fork(2) 这样的表达方式。你可以参考 Linux 用户协会的 Linux 101 讲义附录中的[相关章节](https://101.lug.ustc.edu.cn/Appendix/man/#man-sections)来进行初步的了解。

## 实验内容

- 构建容器的根文件系统（rootfs）
- 使用 clone(2) 代替 fork(2)，并隔离命名空间
- 在容器中使用 mount(2) 与 mknod(2) 挂载必要的文件系统结构
- 使用 pivot\_root(2) 替代 chroot(2) 完成容器内根文件系统的切换
- 用 libcap 为容器缩减不必要的能力（capabilities）
- 使用 libseccomp 对容器中的系统调用进行白名单过滤
- 使用 cgroup 限制容器中的 CPU、内存、进程数与 I/O 优先级

## 实验要求

请按照以下目录结构组织你的 GitHub 仓库：

```
(Git)                     // Git 仓库目录
├── lab4                  // 实验四根目录
│   ├── *.c               // 你的容器源代码
│   ├── Makefile          // （可选）你提供的 Makefile
│   └── README.md         // 简要的说明
├── .gitignore            // 这两个文件在实验一的文档里说过了
└── README.md
```

本次实验满分为 10 分，你需要完成：

- 使用 Linux 命名空间隔离子进程的特性（不要求隔离网络命名空间与用户命名空间）
- 正确挂载容器内的 `/proc`, `/sys`, `/dev`，并在 `/dev` 下创建必要的节点
- 使用 pivot\_root(2) 替代 chroot(2) 完成容器内根文件系统的切换
- 为容器中的进程移除一些能力（drop capabilities）
- 使用 SecComp 限制容器中能够进行的系统调用
- 使用 cgroup 限制容器中的系统资源使用

与实验二和三相似，本实验不要求实验报告，但是推荐简单描述你的实现，并对你使用的「奇技淫巧」简单介绍，以免对助教造成误会等。

本次实验满分为 10 分，但根据以上内容，实际可获得的分数上限为*（待定）*分，超出 10 分的部分将减半计入实验总分。

例如，若你根据以上标准获得了 16 分，那么本次实验计入总分的分数为 13 分，这将等价于本课程总评成绩的 6.5 分。
