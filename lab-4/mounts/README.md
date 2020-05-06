# 为容器补全文件系统

在实验一中我们知道，仅有一个 `init` 程序的文件系统也可以启动一个完整的 Linux 系统，但是显然这样的 Linux 系统是没有什么实用价值的——它运行不了哪怕是稍微典型一些的 Linux 程序或 shell 命令，这是因为仅有一个 init 程序的文件系统缺乏大部分程序需要的目录结构，如 `/proc` 和 `/sys` 等。

## 使用 mount 命令

[mount(8)][mount.8] 是 Linux 中常用的挂载文件系统的命令，当什么参数都没有时，它列出全部已挂载的文件系统。

### 查看当前系统挂载点

在终端中运行 `mount` 命令，可以看到类似这样的输出：

```text
sysfs on /sys type sysfs (rw,nosuid,nodev,noexec,relatime)
proc on /proc type proc (rw,nosuid,nodev,noexec,relatime)
udev on /dev type devtmpfs (rw,nosuid,relatime,size=32852896k,nr_inodes=8213224,mode=755)
devpts on /dev/pts type devpts (rw,nosuid,noexec,relatime,gid=5,mode=620,ptmxmode=000)
tmpfs on /run type tmpfs (rw,nosuid,noexec,relatime,size=6576800k,mode=755)
none on /tmp type tmpfs (rw,nosuid,nodev,noatime)
/dev/nvme0n1p2 on / type ext4 (rw,relatime,errors=remount-ro)
securityfs on /sys/kernel/security type securityfs (rw,nosuid,nodev,noexec,relatime)
tmpfs on /dev/shm type tmpfs (rw,nosuid,nodev)
tmpfs on /run/lock type tmpfs (rw,nosuid,nodev,noexec,relatime,size=5120k)
tmpfs on /sys/fs/cgroup type tmpfs (ro,nosuid,nodev,noexec,mode=755)
cgroup2 on /sys/fs/cgroup/unified type cgroup2 (rw,nosuid,nodev,noexec,relatime)
cgroup on /sys/fs/cgroup/systemd type cgroup (rw,nosuid,nodev,noexec,relatime,xattr,name=systemd)
pstore on /sys/fs/pstore type pstore (rw,nosuid,nodev,noexec,relatime)
efivarfs on /sys/firmware/efi/efivars type efivarfs (rw,nosuid,nodev,noexec,relatime)
bpf on /sys/fs/bpf type bpf (rw,nosuid,nodev,noexec,relatime,mode=700)
```

mount 的每一行输出格式如下：

```text
{源} on {挂载点} type {类型} ({挂载参数},{挂载参数},...)
```

例如，对于上面的第一行输出，

```text
sysfs on /sys type sysfs (rw,nosuid,nodev,noexec,relatime)
```

表示源 `sysfs` 以 `sysfs` 类型挂载到了 `/sys` 路径，参数为读写（`rw`）、无视 setuid 标志（`nosuid`）、不允许设备文件（`nodev`）、不允许执行程序（`noexec`）以及使用相对的访问时间（`relatime`）。

通常情况下，文件系统需要真实设备的支持（如 `/dev/sda` 或 `/dev/nvme0n1p2`），但是对于一些类型的文件系统（如 tmpfs），挂载源是不重要的，甚至可以使用任意字符串。

### 挂载一个文件系统

使用 `mount` 命令挂载一个 tmpfs 到某个路径下的操作为：

```shell
mkdir -p /mnt/mytmpfs
mount -t tmpfs none /mnt/mytmpfs
```

mount 命令的详细用法可以在 [mount(8) 的手册][mount.8]中查看。对于上面的第二行命令，对应的是下面这个格式：

```text
mount [-fnrsvw] [-t fstype] [-o options] device dir
```

### 卸载文件系统

卸载文件系统可以使用 umount(8)，例如

```shell
umount /mnt/mytmpfs
```

即可完成对刚才挂载的目录的卸载操作。

同样，umount(8) 也有一些参数，例如 `-l`（lazy）可以将挂载点从目录树结构中卸载，而不影响正在使用该文件系统结构的进程（正常情况下，卸载一个文件系统前不能有进程在占用这个文件系统）。

## 使用 mount 系统调用挂载文件系统

查看 [mount(2) 的手册][mount.2]，可以发现其原型如下：

```text
#include <sys/mount.h>

int mount(const char *source, const char *target,
          const char *filesystemtype, unsigned long mountflags,
          const void *data);
```

不难猜到，mount(2) 的前两个参数对应 mount(8) 的最后两个参数，而 mountflags 则对应**部分**挂载选项，如只读 / 可写，以及 `nosuid`, `nodev`, `noexec` 等选项。请自己查阅资料，了解最后一个参数 `data` 的意义。

与 mount(2) 类似，umount(2) 及 umount2(2) 两个系统调用可以用于卸载文件系统。请自行查阅手册了解它们的用法。

## 隔离挂载点

请确保你在第一步完成了对挂载命名空间（CLONE\_NEWNS）的隔离，再进行这一步操作。

隔离了挂载命名空间后，为了保证所有对现有挂载点的修改不会传播（propagate）到主机中，首先需要将 rootfs 递归地重新挂载为私有（提示：`mount --make-rprivate`），然后再进行容器内的挂载操作等。

## 实验要求

请使用 mount(2) 为你的容器在 `/dev`, `/proc`, `/sys` 和 `/tmp` 位置各挂载一个合适的文件系统，并在 `/sys/fs/cgroup` 下挂载指定的四类 cgroup 控制器（见 [cgroup 一节](../cgroup/README.md)）。

注意**将 `/sys` 挂载为只读**（这是 systemd 的容器界面的一个要求，这里也作为本实验的要求）。

你可以自行决定是否为其他路径挂载恰当的文件系统（如 `/run`），或从主机系统以 bind 方式挂载一些目录。

下一节的 pivot\_root 后你需要使用 umount(2) 从容器中卸载主机系统的根文件系统，以将主机从容器中“隐藏”。这一步操作将在下一节中详细讨论。

另外，在下一节 pivot\_root 的过程中，你可能需要调整部分本节中已完成的 monut 操作的顺序。


  [mount.2]: http://man7.org/linux/man-pages/man2/mount.2.html
  [mount.8]: http://man7.org/linux/man-pages/man8/mount.8.html
