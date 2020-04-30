# 对容器中的系统调用进行白名单过滤

Linux 的 seccomp (seccomp-bpf) 特性，可以帮助我们检查、过滤程序的系统调用，提高容器的安全性。

## 什么是 seccomp-bpf?

### seccomp

如果需要限制程序可以执行的系统调用，传统的方式是使用 ptrace。但是，（在中期报告的时候大家已经知道），这种方式的性能并不高，每次执行系统调用，都会带来很大的额外开销。

seccomp 是 "secure computing mode" 的缩写。在 Linux 2.6.12 中，seccomp 被引入 Linux 内核。在程序进入 seccomp 模式后，它只能执行 `_exit()`, `sigreturn()`, `read()`, `write()` 这四种系统调用，如果尝试执行其它的系统调用，程序会被 SIGKILL 信号杀死。显然，程序一旦进入 seccomp 模式，就无法恢复到原先的状态了，这就保证了 seccomp 模式下程序的安全。

### seccomp-bpf

但是，seccomp 的问题是：它的管理太严格了，且无法自定义。在 seccomp 中的程序，甚至连 `malloc()`（需要使用 `mmap()` 等系统调用）都无法执行，这对于一个面向应用程序的容器来说显然是不能接受的。于是，seccomp-bpf 被引入。

BPF ("Berkeley Packet Filter") 是一个在内核态用于过滤网络数据包的机制，内核中的解释器会解析 BPF 指令，在 BPF「虚拟机」上执行。它在这里也被用于与 seccomp 结合，过滤系统调用。

好消息是，这里我们不需要手写 BPF 指令代码。使用 libseccomp 库就可以方便地使用 seccomp-bpf 了。相关的开发文件可以使用 `apt install libseccomp-dev` 安装，同时在编译链接时需要加上 `-lseccomp` 参数。

你可能会使用到以下库函数（不限于此）：

```c
scmp_filter_ctx seccomp_init(uint32_t def_action);
int seccomp_rule_add(scmp_filter_ctx ctx, uint32_t action, int syscall, unsigned int arg_cnt, ...);
int seccomp_load(scmp_filter_ctx ctx);
void seccomp_release(scmp_filter_ctx ctx);
```

请查询它们的文档，了解使用方法。

### 我需要过滤哪些系统调用？

与上一节中的「能力」一样，我们使用**白名单**的方式来限制。你可以参考 [Docker 的默认配置](https://github.com/moby/moby/blob/f6a5ccf492e8eab969ffad8404117806b4a15a35/profiles/seccomp/default.json) 来配置 seccomp-bpf。

建议：

- 通过第 54 行 "names" 数组中的所有系统调用
    - 如果你在编译时遇到了问题，可以放心地从列表中删除 `io_uring` 开头的系统调用
- 通过 `arch_prctl` 系统调用（它出现在第 524 行的数组中，是 x86 平台特有的调用）
- 通过 `modify_ldt` 系统调用（它出现在第 539 行的数组中，是 x86 平台特有的调用）
- 通过第 586 行 "names" 数组中的所有系统调用（这些系统调用是进行一系列系统维护操作需要用到的）

你可以在保证程序正常运行的情况下修改白名单。安全性不是我们考核的核心。
