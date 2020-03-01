# Lab 1

实验一目标是带领同学们感受 OS，「以体验、观察、微调为主」。

由于 COVID-19，没有办法给同学们人手一块树莓派，所以还是在 x86-64 上做。

## 实验内容

- 安装 Linux 操作系统。
- 熟悉 Linux 的简单使用，搭建实验环境。
- 学习 Git 的简单使用。
- 熟悉 GitHub 的简单使用，搭建实验私有仓库。
- 编译 Linux 内核，并在 qemu 上跑起来。
- 编写 bare-metal 代码，并在 qemu 上跑起来。

## 实验教程部分

### 安装 Linux 操作系统

### 熟悉 Linux 的简单使用

### 搭建实验环境

你需要安装 `qemu`，以及其他一些开发需要的组件。以 Ubuntu 18.04 为例：

```shell
sudo apt install git \
  qemu-system-x86 build-essential flex bison bc \
  ncurses-dev libssl-dev libelf-dev \
  nasm
```

### 学习 Git 的简单使用

### 学习 GitHub 的简单使用，搭建实验私有仓库

你需要创建一个 GitHub 账号，并且创建一个**私有 (private) 的仓库**，并命名为 OSH-2020-Labs。

在这里仓库中，你需要将助教的账号 [OSH-2020-TA](https://github.com/OSH-2020-TA) 加入为你的 collaborator。发送邀请后，向三位助教之一发送你的姓名、学号、GitHub 账号名。

### 编译 Linux 内核并在 `qemu` 中测试

<https://www.kernel.org/> 上可以下载到 Linux 内核的源代码。此次实验，我们选择 Linux 5.4.22 的内核进行编译。

**请在 64 位的 Linux 环境中进行编译。**

下载 <https://cdn.kernel.org/pub/linux/kernel/v5.x/linux-5.4.22.tar.xz>，解压缩。

```shell
make defconfig
```

创建默认内核配置。

```shell
make menuconfig
```

可以对内核的详细选项进行配置。为了完成此次实验，你需要在这里对内核进行缩减。当然，对不熟悉 Linux 的同学我们不建议在第一次编译时这么做。

注意：为了成功运行之后的测试程序 3，在 `defconfig` 的基础上需要在 `menuconfig` 的时候开启 Device Drivers -> Graphics support -> Frame buffer Devices -> Support for frame buffer devices -> VESA VGA graphics support。

```shell
make bzImage
```

开始编译内核。如果你有多核心的处理器，可以加上 `-j` 参数以并行编译，加快速度。例如，对 4 核心的处理器：

```shell
make -j4 bzImage
```

编译内核需要比较长的时间。可以休息一下，做点别的事情。

在编译完成后，你可以使用 QEMU 测试效果。

```shell
qemu-system-x86_64 -kernel linux-5.4.22/arch/x86_64/boot/bzImage
```

如果 Linux 内核在屏幕上有输出，那么说明编译成功了。

### 构建初始内存盘 (initrd, init ram disk)

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

#### 测试程序

我们提供了三个静态链接的测试程序，在合适的环境下可以独立运行。它们的功能分别是：

- 程序 1: 调用 Linux 下的系统调用，每隔 1 秒显示一行信息。显示五次。
- 程序 2: 向串口输出一条信息。此程序依赖 `/dev/ttyS0` 设备文件。
- 程序 3: 在 TTY 中使用 Framebuffer 显示一张图片。需要 800x600 32ppm 的 VGA 设置。此程序依赖 `/dev/fb0` 设备文件。

你需要做的是：编写一个 `init` 程序，执行这三个程序。下载链接 [TBD]。具体的要求见实验要求。

为了能够得到正确的结果，你的 QEMU 命令需要将串口设置为「标准输入输出」(`-serial stdio`)，并且使用内核参数通知内核设置正确的图形分辨率与颜色位数 (`-append 'vga=0x343'`)。

```
qemu-system-x86_64 -kernel linux-5.4.22/arch/x86_64/boot/bzImage -initrd ramdisk/test3/initrd.cpio.gz -append 'vga=0x343' -serial stdio
```

#### 关于设备文件

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

##### 在 fakeroot 环境中打包 initrd

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

### 使用 BusyBox 构建 initrd（可选）

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

### 使用汇编语言，编写 x86 裸金属 (bare-metal) 程序

你可能需要使用的 x86 指令（不限制于此）：

```
mov (数据移动)
int (呼叫中断处理)
jmp (跳转)
cmp (比较操作数，修改 flags，以提供给条件跳转指令)
je 等一些条件跳转指令
lodsb (将寄存器 si 指向的字符串对应的字符加载到 al 中，并自增 si)
hlt (停机)
```

在传统的 PC 加载时，BIOS 会加载磁盘中的 MBR 块，并在 16 位实模式中执行其中的指令。一个简单的、向屏幕输出一行字符串的 MBR 程序如下：

```assembly
[BITS 16]                               ; 16 bits program
[ORG 0x7C00]                            ; starts from 0x7c00, where MBR lies in memory

mov si, OSH                             ; si points to string OSH
.print_str:
    lodsb                               ; load char to al
    cmp al, 0                           ; is it the end of the string?
    je .hlt                             ; if true, then halt the system
    mov ah, 0x0e                        ; if false, then set AH = 0x0e 
    int 0x10                            ; call BIOS interrupt procedure, print a char to screen
    jmp .print_str                      ; loop over to print all chars

.hlt:
    hlt

OSH db `Hello, OSH 2020 Lab1!`, 0       ; our string, null-terminated

TIMES 510 - ($ - $$) db 0               ; the size of MBR is 512 bytes, fill remaining bytes to 0
DW 0xAA55                               ; magic number, mark it as a valid bootloader to BIOS
```

此样例程序需要使用 `nasm` 汇编。

```
nasm example.asm
```

会生成 `example` 文件。这个文件可以直接被 `qemu` 加载。

```
qemu-system-x86_64 -hda example
```

为了完成附属的实验要求，你可能需要使用 8254 计时器。绝大多数的 BIOS 都将此计时器值映射到了内存中的 0x046c 位置。这个值会每秒更新 18.2 次。如果你需要使用这个计时器的话，需要打开中断 (`sti`)。

## 实验要求

本次实验满分 10 分。你需要完成：

- 关于 Git 与 GitHub 仓库：
  - 正确创建了仓库，助教可访问，且**仓库权限为私有 (private)**。（**必须完成**）
  - Git 记录能够清晰显示你的实验进度，没有出现以下的情况。（1 分）
    - 很大一部分的 commit 由 GitHub 网页版生成，即通过网页版文件上传的方式提交实验的文件。
    - 只有一个，或两三个 commit。
    - 上传了大量与实验要求无关的文件，没有设置 `.gitignore`。
- 关于 Linux 内核：
  - 正确编译了内核，并将编译出的 `arch/x86/boot/bzImage` 放在你的仓库中的 `lab1/linux/` 下面。（1 分）
  - 你裁剪了内核，裁剪得到的内核大小不超过 8 MiB (= 8388608 bytes)，并且它可以完成以下「关于 `initrd` 与 `init` 程序」中的全部任务。（1 分）
    - 如果你没有完成「关于 `initrd` 与 `init` 程序」中的全部任务，助教会使用自己的 `initrd` 进行测试。
  - 在上一条的基础上，裁剪得到的内核大小不超过 4 MiB (= 4194304 bytes)。（0.5 分）
- 关于 `initrd` 与 `init` 程序：
  - 构造了格式正确的 `initrd.cpio.gz` 文件，并将对应文件放在你的仓库中的 `lab1/linux/initrd.cpio.gz`。（0.5 分）
  - 使用你编译的内核和 `initrd.cpio.gz`，其中的 `/init` 能够成功执行助教提供的程序 1。（0.5 分）
  - 使用你编译的内核和 `initrd.cpio.gz`，其中的 `/init` 能够成功依次执行助教提供的程序 1、2、3，并且程序 3 执行完成后不出现内核恐慌 (Kernel Panic)。（1.5 分）
- 关于 x86 裸金属 MBR 程序：
  - 汇编源文件在仓库中的 `lab1/mbr/` 下，源文件汇编得到的二进制文件可以在 QEMU 环境下向屏幕输出文字。（1 分）
  - 汇编程序能够首先清空屏幕，之后每隔大约 1 秒（不需要非常精确），向屏幕上输出一行文字。（1.5 分）
- 思考题：
  - 在下面给出的思考题中任选 4 道并完成它们。（1.5 分）

**注意：你需要写实验报告，并且报告需要要点齐全。在要点齐全的情况下可以精简，可以描述你在完成实验时遇到的问题与解决方法，也可以提供建议等。如果对以上每个给分大项（除 Git 相关内容外）没有对应的实验报告，你可能不会得到这一部分的分数。**

### 思考题

1. 在使用 `make menuconfig` 调整 Linux 编译选项的时候，你会看到有些选项是使用 `[M]` 标记的。它们是什么意思？在你的 `init` 启动之前的整个流程中它们会被加载吗？如果不，在正常的 Linux 系统中，它们是怎么被加载的？
2. 在「构建 `initrd`」的教程中我们创建了一个示例的 init 程序。为什么屏幕上输出 "Hello, Linux!" 之后，Linux 内核就立刻 kernel panic 了？
3. 为什么我们编写 C 语言版本的 `init` 程序在编译时需要静态链接？我们能够在这里使用动态链接吗？
4. 在 Vlab 平台上，即使是真正的 root 也无法使用 `mknod` 等命令，查找资料，尝试简要说明为什么 fakeroot 环境却能够正常使用这些命令。
5. 在介绍 BusyBox 的一节，我们发现 `init` 程序可以是一段第一行是 `#!/bin/sh` 的 shell 脚本。尝试解释为什么它可以作为系统第一个启动的程序，并且说明这样做需要什么条件。
6. 我们的 MBR 样例程序使用了 BIOS 调用 (`int 0x10`) 以显示文本。Linux 的 `init` 程序也可以用这种办法输出文本吗？为什么？
7. MBR 能够用于编程的部分只有 510 字节，而目前的系统引导器（如 GRUB2）可以实现很多复杂的功能：如从不同的文件系统中读取文件、引导不同的系统启动、提供美观的引导选择界面等，用 510 字节显然是无法做到这些功能的。它们是怎么做到的？
8. 目前，越来越多的 PC 使用 UEFI 启动。请简述使用 UEFI 时系统启动的流程，并与传统 BIOS 的启动流程比较。

## 实验资料

助教提供的三个测试程序：[binaries.zip](binaries.zip)
