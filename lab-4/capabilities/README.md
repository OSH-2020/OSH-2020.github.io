# 为你的容器缩减不必要的能力

在完成了以上几点要求，创建了容器之后，你可能会发现：尽管你实现了上文提到的 5 种命名空间隔离，你的「容器」中的进程（在 root 权限下）仍然可以做一些越界的事情，比如说：

- 修改主机的系统日期与时间
- 使用 `mknod` 创建设备文件，并且（与主机一样）任意读写（请不要在你的电脑上尝试！）
- 任意加载、卸载内核模块
- 通过 `/sys` 修改内核设置
- ……

本节 (capabilities) 与下一节 (seccomp) 的内容将关注缩减容器中程序的权限，安全地执行程序。

## 什么是能力 (capabilities)？

从 Linux 2.2 开始，capabilities 的概念出现，用于更加细分系统级别的权限。在此之前，需要使用 `root` 级别权限执行系统级任务的程序需要获取完整的 root 权限才行。但一部分程序只需要这些权限中很小的一部分（例如需要发送原始数据包的 `ping`），将它们以 root 权限运行带来了安全性上的风险：这些程序中的小缺陷，会导致攻击者获得完整的 root 权限。

在 capabilities 出现之后，这样的情况有了改善。Capabilities 将 root 权限进一步细分，只需要给程序授予必需的权限（“能力”），即使不以 root 用户运行，也可以完成任务，提高了系统的安全性。同样，对于以 root 用户执行的程序，也可以通过丢弃权限，减小攻击面。实际上，一个拥有全部 capabilities 的程序与真实的 root 用户几乎没有区别。

有关 capabilities 的列表等更多信息，可以到 `man 7 capabilities` 查看。

## 如何限制容器中进程的能力？

可以使用 libcap 库来进行限制,使用 `man libcap` 查看库函数与链接方式。

或者，你也可以使用 `libcap-ng` 库来进行限制（更简单）。使用方法可参见 <https://people.redhat.com/sgrubb/libcap-ng/>。

两个库可以分别用以下命令安装（Ubuntu）：

```shell
sudo apt install libcap-dev
sudo apt install libcap-ng-dev
```

如果你使用了 libcap，那么编译时需要加上链接参数 `-lcap`；同理，对于 libcap-ng，需要加上链接参数 `-lcap-ng`。

提示：你可能需要阅读 `man 7 capabilities` 了解 capability set 的相关资料。此外，`libcap` 附带一个命令行工具 `capsh`，可以帮助你检查能力设置是否正确。

## 我需要限制哪些能力？

在实验中，我们使用**白名单**的方式限制能力，即丢弃除白名单中能力以外的其他所有能力。白名单参考 [Docker 的默认配置](https://docs.docker.com/engine/reference/run/#runtime-privilege-and-linux-capabilities)如下：

| Capability | （中文翻译后的）描述 |
| :--------: | :------------------: |
| SETPCAP | 修改进程的「能力」 |
| MKNOD | 使用 mknod(2) 创建设备文件 |
| AUDIT\_WRITE | 向内核审计日志写入记录 |
| CHOWN | 任意修改文件的 UID 和 GID (参考 chown(2)) |
| NET\_RAW | 使用 packet socket，收发任意数据包 (参考 packet(7)) |
| DAC\_OVERRIDE | 绕过文件 rwx 权限检查 |
| FOWNER | 绕过要求进程的 UID 与文件 UID 一致的检查 |
| FSETID | 在文件修改时允许不清除 setuid 和 setgid 权限位 |
| KILL | 绕过发送信号时的权限检查 |
| SETGID | 任意修改进程的 GID 与辅助 GID 列表 |
| SETUID | 任意修改进程的 UID |
| NET\_BIND\_SERVICE | 将 TCP 或 UDP 套接字绑定到小于 1024 的端口（即「特权端口」） |
| SYS\_CHROOT | 使用 chroot(2) |
| SETFCAP | 设置文件的「能力」 |

在这份白名单的基础上，你还可以再删除一些 capabilities。如果你还删除了以上列表中的 capabilities，请在你仓库的 README 文件中注明。
