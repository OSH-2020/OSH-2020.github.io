# 编译 Linux 内核并在 QEMU 中测试

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