# 使用汇编语言，编写 x86 裸金属 (bare-metal) 程序

请先阅读、搜索资料，熟悉 x86 汇编指令。

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

```asm
[BITS 16]                               ; 16 bits program
[ORG 0x7C00]                            ; starts from 0x7c00, where MBR lies in memory

mov si, OSH                             ; si points to string OSH
print_str:
    lodsb                               ; load char to al
    cmp al, 0                           ; is it the end of the string?
    je halt                             ; if true, then halt the system
    mov ah, 0x0e                        ; if false, then set AH = 0x0e 
    int 0x10                            ; call BIOS interrupt procedure, print a char to screen
    jmp print_str                      ; loop over to print all chars

halt:
    hlt

OSH db `Hello, OSH 2020 Lab1!`, 0       ; our string, null-terminated

TIMES 510 - ($ - $$) db 0               ; the size of MBR is 512 bytes, fill remaining bytes to 0
DW 0xAA55                               ; magic number, mark it as a valid bootloader to BIOS
```

（3/8 更新：样例程序有小幅修改，修复了在较老版本的 nasm 中无法汇编的问题。）

此样例程序需要使用 `nasm` 汇编。

```
nasm example.asm
```

会生成 `example` 文件。这个文件可以直接被 `qemu` 加载。

```
qemu-system-x86_64 -hda example
```

为了完成附属的实验要求，你可能需要使用 8254 计时器。绝大多数的 BIOS 都将此计时器值映射到了内存中的 0x046c 位置。这个值会每秒更新 18.2 次。如果你需要使用这个计时器的话，需要打开中断 (`sti`)。

## 参考资料

1. [x86 Assembly Guide](https://www.cs.virginia.edu/~evans/cs216/guides/x86.html)
2. [NASM Tutorial](https://cs.lmu.edu/~ray/notes/nasmtutorial/)
