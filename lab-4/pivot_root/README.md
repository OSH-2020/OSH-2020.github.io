# 切换容器的根文件系统

在第一部分中，我们比较了 chroot 与 systemd-nspawn 两种工具。

**请确保你在前一步实验中按要求完成了对挂载命名空间和 PID 命名空间的隔离**（推荐按说明隔离 5 个命名空间，但本步骤最少需要这两个），否则可能会对你的操作系统产生一些影响。这也是给出的样例代码只用了 chroot(2) 的原因。

## 使用 [pivot\_root(2) 系统调用][pivot_root.2]

查看 pivot\_root(2) 的 man 文档，可以知道它的原型如下：

```c
int pivot_root(const char *new_root, const char *put_old);
```

同时注意到带有下划线的 Note 就在下面一行：

```text
Note: There is no glibc wrapper for this system call; see NOTES.
```

因此，我们面对的第一个问题是，不能直接使用 `pivot_root(new_root, put_old)`，而是需要使用它的系统调用号和 [syscall(2)][syscall.2]。请自行查阅资料解决这个问题。

### 准备 pivot\_root 的环境

在 pivot\_root(2) 的 DESCRIPTION 一节有写明 *The following restrictions apply*，我们比较关心其中的第二个：

> *new\_root* and *put\_old* must not be on the same filesystem as the current root.

由于我们的容器的根文件系统就在一个目录下，在这种情况下，这实际上是不可能的。实验表明，这里要求的并不是物理上或逻辑上不同的“文件系统”，而是不在同一个挂载点下的目录树，因此我们只需通过 bind mount 的方式将我们的容器的根文件系统挂载到一个临时的地方，然后就可以 pivot\_root 过去了。

创建一个临时的目录用于 bind mount 十分简单，例如你可以使用 `/tmp/lab4`，但是多次运行的时候可能会发生冲突，因此这里推荐使用 [mkdtemp(3)][mkdtemp.3] 创建含有随机字符串的临时文件夹用于后续工作。

mkdtemp(3) 接受一个字符串参数，且末尾有至少 6 个大写字母 `X`。当其运行成功时，会将字符串末尾的 X 替换为随机的字母和数字，并以这个新的名字创建一个目录。

例如，下面就是一种 mkdtemp(3) 的用法：

```c
char tmpdir[] = "/tmp/lab4-XXXXXX";
mkdtemp(tmpdir);
```

接下来我们需要使用 mount(2) 将容器的根文件系统 bind mount 至新创建的目录。关于 mount(2) 的用法请参阅[相关章节](../mounts/README.md)。在 shell 中，你将会使用 `mount -o bind <src> <dst>` 这样的命令进行操作。

**注意**：bind mount 会覆盖（隐藏）挂载点原有的文件，同时 bind mount 不是递归的（即原路径下的其他挂载点不会跟着一起 bind 过去），因此如果你已经在上一步完成了对 `/proc` 等容器内特殊文件系统的挂载，你需要将相关操作移动到 bind mount 之后进行。

### 进行 pivot\_root

一切准备就绪后，就可以为容器切换根文件系统了。切换的操作比较简单，直接调用 pivot\_root(2) 即可：

```c
char oldrootdir[] = "";
sprintf(oldrootdir, "%s/oldroot", tmpdir);
pivot_root(tmpdir, oldrootdir);
```

若 `pivot_root` 返回值为零，那么就切换成功了，原来环境的根文件系统现在位于 `/oldroot`，我们将在下面的小节讨论它。

这里对 pivot\_root(2) 的第二个参数做一些补充说明。

在 pivot\_root(2) 的 DESCRIPTION 一节有写明 *The following restrictions apply*，刚才我们关注了其中的第二个，现在我们要关注第三个：

> *put\_old* must be underneath *new\_root*, that is, adding a nonzero number of `/..` to the string pointed to by *put\_old* must yield the same directory as *new\_root*.

这里的要求是 `put_old` 必须是 `new_root` 的子目录，所以它们都是进行 pivot\_root **之前**的路径，而在 pivot\_root 之后，原先的根文件系统就位于 `put_old - new_root` 的位置了。（这里的字符串减法仅作示意，理解意思即可）

## 隐藏主机的根文件系统

为容器切换根文件系统后，主机的根文件系统就位于容器中的 `/oldroot` 了。这时候你可以观察 `mount` 命令的输出，应该能看到其中有下面这样一行：

```text
/dev/sda1 on /oldroot type ext4 (rw,relatime,errors=remount-ro)
```

同时你也应该注意到了，挂载路径开头就是 `/`（根目录），而不是像 `/tmp/lab4-tkytql` 这样在主机上的前缀，这说明 pivot\_root 运行成功了。

容器的一大特点就是与主机隔离，因此**将主机的文件系统暴露在容器中显然是不能接受的**，所以接下来我们就要将 `/oldroot` 从容器中隐藏起来。

现在主机的根文件系统现在刚好是一个挂载点，我们可以直接将其卸载。你可能会迫不及待地想要运行下面的命令：

```shell
umount /oldroot
```

这时候你会失望地看到下面的输出：

```text
umount: /oldroot: target is busy.
```

这是因为主机上还有进程在使用这个挂载点下的文件等（这是显然的），因此不能直接卸载这个路径。

Linux 提供了一种方式，不直接卸载整个挂载点，而是将挂载点从当前的目录树中脱离（detach），此时所有程序都不再能够通过这个路径来访问该挂载点下的内容，但已经打开的文件描述符等则不受影响。这被称作「懒惰卸载」（lazy unmounting），对应的命令是 `umount -l <path>`。请自行查阅资料（如 [umount(8)][umount.8] 等），结合使用 strace 工具找出合适的系统调用来完成该操作（将主机的根文件系统从当前目录树结构中脱离）。正确隐藏主机的根文件系统后，`/oldroot` 应该为一个空目录，此时你可以将它删除（使用 `rmdir("/oldroot")` 即可）

作为一个提示，本段落所述内容非常简单，只需要调用两个系统调用，其中第二个是 `rmdir(2)`；

## 从主机上隐藏容器的根文件系统

在前面的步骤中，我们为了使用 pivot\_root(2)，将本已存在于系统目录结构中的容器的根文件系统通过 bind mount 的方式又挂载到了另一个地方，这从某种程度上在主机的目录树中增加了混乱。

这一步的要求与上一步完全一致，但方向恰好相反，将前面的步骤中创建的 bind mount **挂载点**用相同的方式从主机中隐藏（包括 rmdir 移除挂载目录）。你不需要将 bind mount 的源目录隐藏（事实上这个要求可能是不合理的）。

如果你在前面实现的挂载点隔离是正确的，那么只需要移除 bind mount 的临时目录即可。

如果你需要一种在父进程和子进程之间通信的方法，请仔细回忆实验二中要求实现的功能，思考一下有没有在这里用得上的办法。


  [pivot_root.2]: http://man7.org/linux/man-pages/man2/pivot_root.2.html
  [syscall.2]: http://man7.org/linux/man-pages/man2/syscall.2.html
  [mkdtemp.3]: http://man7.org/linux/man-pages/man3/mkdtemp.3.html
  [umount.8]: http://man7.org/linux/man-pages/man8/syscall.8.html
