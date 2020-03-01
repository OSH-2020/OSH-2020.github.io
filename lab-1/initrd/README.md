# 构建初始内存盘 (initrd, init ram disk)

在 Linux 启动时，需要首先加载 `initrd` 进行初始化的操作。以下操作可以构建一个最小化的 `initrd`。我们首先创建一个 C 程序，代码如下：

```c
#include <stdio.h>

int main() {
    printf("Hello, Linux!\n");
    return 0;
}
```

保存为 `init.c`。之后编译，**静态链接**为可执行程序。

```shell
gcc -static init.c -o init
```

创建一个新的目录用于放置文件。在新的目录下打包 `initrd`：

```shell
find . | cpio --quiet -H newc -o | gzip -9 -n > ../initrd.cpio.gz
```

这会在目录外创建压缩后的 `initrd.cpio.gz` 内存盘。同样，我们使用 QEMU 测试效果。

```shell
qemu-system-x86_64 -kernel linux-5.4.22/arch/x86_64/boot/bzImage -initrd ramdisk/test3/initrd.cpio.gz
```

当你看到屏幕上出现 "Hello, Linux!" 的时候，就成功了。

## 测试程序

我们提供了三个静态链接的测试程序，在合适的环境下可以独立运行。它们的功能分别是：

- 程序 1: 调用 Linux 下的系统调用，每隔 1 秒显示一行信息。显示五次。
- 程序 2: 向串口输出一条信息。此程序依赖 `/dev/ttyS0` 设备文件。
- 程序 3: 在 TTY 中使用 Framebuffer 显示一张图片。需要 800x600 32ppm 的 VGA 设置。此程序依赖 `/dev/fb0` 设备文件。

你需要做的是：编写一个 `init` 程序，执行这三个程序。下载链接 [TBD]。具体的要求见实验要求。

为了能够得到正确的结果，你的 QEMU 命令需要将串口设置为「标准输入输出」(`-serial stdio`)，并且使用内核参数通知内核设置正确的图形分辨率与颜色位数 (`-append 'vga=0x343'`)。

```
qemu-system-x86_64 -kernel linux-5.4.22/arch/x86_64/boot/bzImage -initrd ramdisk/test3/initrd.cpio.gz -append 'vga=0x343' -serial stdio
```

## 关于设备文件

`/dev/ttyS0` 和 `/dev/fb0` 不是普通的文件。它们是设备文件，需要使用 `mknod` 创建。

在 Linux 中主要有两类设备文件：块设备文件（有缓冲）、字节设备文件（无缓冲）。

- 块设备 (b)：以块为单位与硬件设备传输文件。对它的写入操作会被缓存，由内核在合适的时候发送给硬件。
- 字节设备 (c)：以字节为单位与硬件设备传输文件。

在创建时，还需要提供主设备号 (Major) 和次设备号 (Minor)。设备文件一般位于 `/dev/`，可以使用 `ls -l` 查看它们的信息。

```
$ ls -l /dev/null
crw-rw-rw- 1 root root 1, 3 Feb 14 20:27 /dev/null
$ ls -l /dev/sda
brw-rw---- 1 root disk 8, 0 Feb 14 20:27 /dev/sda
```

这里可以看到，空设备 (`/dev/null`) 为字节设备 (c)，主设备号为 1，次设备号为 3。第一块硬盘对应的设备 (`/dev/sda`) 为块设备 (b)，主设备号为 8，次设备号为 0。

为了能够在你的 `init` 执行的时候为程序 2 与程序 3 准备好这两个设备文件，你需要做的是，在 `init` 中调用 `mknod` 程序或者调用 `mknod` 系统调用生成这两个文件。已知 `/dev/ttyS0` 为字节设备文件，主设备号为 4，次设备号为 64；`/dev/fb0` 为字节设备文件，主设备号为 29，次设备号为 0。

你可以使用 `man 1 mknod` 查看 `mknod` 程序的帮助，使用 `man 2 mknod` 查看 `mknod` 系统调用的帮助。**这两者都需要最高用户权限**。

以下是一个在 C 语言中使用 `mknod()` 系统调用，在当前目录创建空设备文件 (null) 的示例：

```c
#include <stdio.h>
#include <sys/types.h>
#include <sys/stat.h>
#include <fcntl.h>
#include <unistd.h>
#include <sys/sysmacros.h>

int main() {
    if (mknod("./null", S_IFCHR | S_IRUSR | S_IWUSR, makedev(1, 3)) == -1) {
        perror("mknod() failed");
    }
    return 0;
}
```

```shell
$ gcc mknod_example.c -o mknod_example
$ sudo ./mknod_example
$ ls -l null
crw------- 1 root root 1, 3 Mar  1 17:10 null
```

当然，你也可以先使用 `mknod` 创建设备文件，然后再打包 `initrd`。但是需要注意两点：

- Vlab 平台不支持用户直接使用 `mknod`，即使是 `root`，但可以在 fakeroot 环境中进行操作（见下方）。
- 输入错误的主、次设备号，创建的设备文件可能会指向错误的硬件。进行写入操作可能会损坏对应的硬件，或使你的文件丢失。当然，在 `init` 中执行操作的话，在 QEMU 虚拟环境下就不需要担心这一点。

## 在 fakeroot 环境中打包 initrd

fakeroot 将建立一个虚拟的 `root` 环境，使 `mknod` 得以在 Vlab 平台使用。

fakeroot 的使用非常简单，只需要在加入设备文件及打包前输入：

```shell
fakeroot
```

即可以虚拟的 `root` 身份操作，输入：

```shell
mknod dev/ttyS0 c 4 64
mknod dev/fb c 29 0
find . | cpio --quiet -H newc -o | gzip -9 -n > ../initrd.cpio.gz
```

这样便可以打包好 `initrd`，再输入：

```shell
exit
```

即可退出虚拟 `root` 环境。

## 使用 BusyBox 构建 initrd（可选）

BusyBox 是一个将许多常用 Unix 命令行工具打包进一个二进制文件的项目，在嵌入式系统等存储空间有限的环境中非常常见。你可以选择使用 BusyBox 提供的工具来编写你的 init 程序，并打包 initrd。

方便起见，你可以直接从这里下载已编译好的 BusyBox: <https://www.busybox.net/downloads/binaries/1.30.0-i686/busybox>

BusyBox 的特点之一是，如果你使用某个支持的别名来运行它，它就会像那条命令一样工作，例如：

```shell
$ wget -qO busybox "https://www.busybox.net/downloads/binaries/1.30.0-i686/busybox"
$ chmod 755 busybox
$ mv busybox ls
$ ./ls
```

你会发现 BusyBox 此时的功能就和 `ls` 命令一模一样。

创建符号链接是最常见的以别名使用 BusyBox 的方式，下面的示范就以 BusyBox 为基础构建了一个 initrd，并使用 BusyBox 内置的 shell 作为一个可交互的 `init`。

```shell
$ mkdir -p rootfs/bin/
$ 把 busybox 文件放进 rootfs/bin/
$ cd rootfs/bin/
$ for item in $(./busybox --list)
> do
>   ln -s busybox $item
> done
```

然后向 `rootfs/init` 中写入以下内容：

```shell
#!/bin/sh

/bin/sh
```

给这个 `init` 文件加上执行权限，并按照前面的教程所述，将 `rootfs/` 目录打包为 `initrd.cpio.gz`，使用 QEMU 启动，你就能在自己编译的 Linux 系统中使用 shell 来探索了。

如果你希望在进入 shell 之前运行额外的命令，例如使用 `mknod` 创建必要的设备文件，你可以将它们写在 `init` 文件中，这些命令就会被依次执行。