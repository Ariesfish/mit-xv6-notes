# Lab1-启动PC

[XV6 JOS代码](https://github.com/Ariesfish/mit-xv6-lab)

```bash
% mkdir 6.828
% cd 6.828
% git clone https://github.com/Ariesfish/mit-xv6-lab.git lab
% cd lab
```

---

## 1.PC Bootstrap

### 从x86汇编语言开始

> **练习1**
>
> 熟悉x86汇编语言，注意Intel语法与Linux上使用的AT&T语法的差异

[PC Assembly Language Book](res/pcasm-book.pdf)

[Brennan's Guide to Inline Assembly](res/Inline-Assembly-with-DJGPP.pdf)

### 模拟x86

使用qemu模拟x86环境，利用gdb远程调试来学习引导过程

> 编译boot引导程序和kernel内核镜像

```bash
% cd lab
% make
+ as kern/entry.S
+ cc kern/entrypgdir.c
+ cc kern/init.c
+ cc kern/console.c
+ cc kern/monitor.c
+ cc kern/printf.c
+ cc kern/kdebug.c
+ cc lib/printfmt.c
+ cc lib/readline.c
+ cc lib/string.c
+ ld obj/kern/kernel
+ as boot/boot.S
+ cc -Os boot/main.c
+ ld boot/boot
boot block is 380 bytes (max 510)
+ mk obj/kern/kernel.img
```

注意: *如果出现 "undefined reference to `__udivdi3'" 之类的错误是由于未安装32-bit的gcc支持库*

> 启动QEMU

```bash
% make qemu # 带VGA显示界面，信息会同时显示在当前终端和QEMU的VGA显示界面
% make qemu-nox # 不带VGA显示界面
Booting from Hard Disk...
6828 decimal is XXX octal!
entering test_backtrace 5
entering test_backtrace 4
entering test_backtrace 3
entering test_backtrace 2
entering test_backtrace 1
entering test_backtrace 0
leaving test_backtrace 0
leaving test_backtrace 1
leaving test_backtrace 2
leaving test_backtrace 3
leaving test_backtrace 4
leaving test_backtrace 5
Welcome to the JOS kernel monitor!
Type 'help' for a list of commands.
K>
```

目前在 `K>` 提示符后仅可输入 `help` 和 `kerninfo` 两个命令查看信息，在终端按 `Ctrl+a x` 退出QEMU

```bash
K> help
help - Display this list of commands
kerninfo - Display information about the kernel
K> kerninfo
Special kernel symbols:
  _start                  0010000c (phys)
  entry  f010000c (virt)  0010000c (phys)
  etext  f01019e9 (virt)  f01019e9 (phys)
  edata  f0113060 (virt)  f0113060 (phys)
  end    f01136a0 (virt)  f01136a0 (phys)
Kernel executable memory footprint: 78KB
K>
```

### PC的物理地址空间

![物理内存地址空间](res/memory-space.png)

在最初的搭载16-bit Intel 8088 CPU的PC上，由于采用了20-bit的地址总线，所以能访问的最大物理地址空间为1M (2<sup>20</sup>)

### BIOS

> 首先在一个终端启动 qemu-gdb(或qemu-nox-gdb)

```bash
% make qemu-nox-gdb
```

> 然后在另一个终端启动GBD执行远程调试

```bash
% make gdb
gdb -n -x .gdbinit
GNU gdb (Ubuntu 8.1-0ubuntu3.2) 8.1.0.20180409-git
Copyright (C) 2018 Free Software Foundation, Inc.
License GPLv3+: GNU GPL version 3 or later <http://gnu.org/licenses/gpl.html>
This is free software: you are free to change and redistribute it.
There is NO WARRANTY, to the extent permitted by law.  Type "show copying"
and "show warranty" for details.
This GDB was configured as "x86_64-linux-gnu".
+ target remote localhost:26000
The target architecture is assumed to be i8086
[f000:fff0]    0xffff0: ljmp   $0xf000,$0xe05b
0x0000fff0 in ?? ()
+ symbol-file obj/kern/kernel
(gdb)
```

加电启动时，首先进入16-bit 8086的实模式(real-mode)，在实模式采用了分段的技术将逻辑地址转换为20-bit的物理地址空间

其中CS(Code Segment)寄存器保存了段的基地址(如0xf000)，IP(Instruction Pointer)或者通俗一点PC(Program Count)寄存器保存了偏移地址(如0xfff0)。实际的物理地址采用 `CS:IP = CS << 4 + IP` 算法转换而成，所以 `0xf000:0xfff0` 指向的实际地址为 `0xffff0`

在8086的实模式下最高可访问的物理地址为 `0xfffff`

BIOS会被加载到1M地址空间的最后 `64K字节` `0xf0000 ~ 0xfffff`，这里加载后的第一条命令 `0xffff0: ljmp $0xf000,$0xe05b` 即是跳转到BIOS前部的 `0xfe05b` 处开始执行

> **练习2**
>
> 通过GDB的si(Step Instruction)单步执行命令查看BIOS的执行过程

BIOS会设置一个中断描述符表，初始化VGA显示界面和PCI总线等，最后它会寻找可启动的设备(软盘、硬盘、光驱等)，然后读取Boot Loader

## 2.The Boot Loader

BIOS将512个字节扇区(光驱的话是2048个字节)的 `Boot Loader` 加载到物理内存的 `0x7c00` 地址，[这里](https://www.glamenv-septzen.net/en/view/6)一篇文章讲述了魔数 `0x7c00` 的由来

`Boot Loader` 由一个汇编源文件 `boot/boot.S` 和一个C源文件 `boot/main.c` 组成。开始执行后首先会将 CPU 从实模式切换到 32-bit 的保护模式(protected mode) 以使用更大的内存地址空间

- cli (clear interrupt, disable interrupts)
- cld (clear direction flag, string operations increment)
  - df: 方向标志位。在串处理指令中，控制每次操作后si，di的增减(df=0，每次操作后si、di递增; df=1，每次操作后si、di递减)

**Note-1**: *80286扩展到24-bit地址总线，最大可访问16M的物理地址空间；80386之后采用32-bit地址总线，最大可访问4G的物理地址空间。它们采用了不同于8086的分段机制*

**Note-2**: *在80286之后的CPU上，为了防止实模式下地址回折的问题，设计了A20线的与门开关，boot/boot.S中可以看到进入保护模式前开启A20的操作*

**Note-3**: 关于 inb/outb IO端口指令, 操作的[外部设备](http://bochs.sourceforge.net/techspec/PORTS.LST)

> **练习3**
>
> 学习更多的GDB命令，在 `0x7c00` 处设置断点，结合 `boot/boot.S` 汇编代码和反汇编之后 `obj/boot/boot.asm` 跟踪 `boot loader` 的加载过程
>
> 然后继续跟踪 boot/main.c 中的 bootmain() 函数以及 readsect() 函数，了解kernel的剩余扇区是如何加载的

**回答下列问题:**

> At what point does the processor start executing 32-bit code? What exactly causes the switch from 16- to 32-bit mode?

- lgdt gdtdesc: 加载全局描述符表
- cr0: control register。cr0包含了6个预定义标志，第0位是保护允许位PE(Protedted Enable), 用于启动保护模式, 如果 PE=1 则保护模式启动, 如果 PE=0 则在实模式下运行。
- ljmp $PROT_MODE_CSEG, $protcseg

从 `.code32 protcseg:` 开始执行32-bit代码，在此之前通过设置 `GDT(lgdtw 0x7c64)` 和将 `CR0` 控制寄存器的保护模式Flag设为有效来完成16-bit到32-bit的切换，此时CPU工作在保护模式下

PROT_MODE_CSEG=0x8, 指定了CS段选择符, 指向GDT中的段描述符。

> What is the last instruction of the boot loader executed, and what is the first instruction of the kernel it just loaded?

bool loader最后执行的指令是 `call *0x10018`(0x10000 + 0x18: ELFHDR->e_entry)，kernel被加载后第一个执行的指令 `0x10000c: movw $0x1234,0x472`，加载之前会做一个entry地址的转换

> How does the boot loader decide how many sectors it must read in order to fetch the entire kernel from disk? Where does it find this information?

从 [ELF](https://wiki.osdev.org/ELF) 头部中的 `ELFHDR + ELFHDR->e_phoff` 指向的 `Proghdr` 的 `p_memsz` 和 `p_offset` 中获得信息

### 加载内核

> **练习4**
>
> 学习C语言，特别是指针概念[pointer.c](res/pointers.c)

查看ELF文件各个section的信息

```bash
% objdump -h obj/kern/kernel
```

- .text: 程序的可执行代码
- .rodata: 只读数据，如字符串常量等
- .data: 已初始化的数据，如全局变量
- .bss: 存放未初始化的变量，但是ELF中只需要记录.bss的起始地址和长度，loader和程序必须自己将.bss段清零

查看程序头信息

```bash
% objdump -x obj/kern/kernel
Program Header:
    LOAD off    0x00001000 vaddr 0xf0100000 paddr 0x00100000 align 2**12
         filesz 0x0000759d memsz 0x0000759d flags r-x
    LOAD off    0x00009000 vaddr 0xf0108000 paddr 0x00108000 align 2**12
         filesz 0x0000b6a8 memsz 0x0000b6a8 flags rw-
   STACK off    0x00000000 vaddr 0x00000000 paddr 0x00000000 align 2**4
         filesz 0x00000000 memsz 0x00000000 flags rwx
```

- vaddr: 虚拟地址 0xf010000
- paddr: 物理地址 0x0010000
- memsz/filesz: 加载数据的大小

> **练习5**
>
> 在 `boot/Makefrag` 里 `-Ttext 0x7C00` 修改 boot loader 的加载地址，看看会发生什么

Boot loader会加载失败，最后产生 SIGTRAP 中断

```bash
% objdump -f obj/kern/kernel
kernel:     file format elf32-i386
architecture: i386, flags 0x00000112:
EXEC_P, HAS_SYMS, D_PAGED
start address 0x0010000c
```

使用 `objdump -f` 查看程序或共享库的起始地址

> **练习6**
>
> 在 boot loader 加载后以及加载 Kernel 时查看 0x100000 地址有什么变化

一开始时是 0x0000, 在加载 kernel 后其中有了内容, 调试时用 `x/8x 0x100000` 查看内存内容

## 3.The Kernel

### 使用虚拟地址解决位置依赖问题

前一节看到 kernel 的虚拟地址为 `0xf010000` 使用这样的高位地址目的是将低位的地址留给用户程序使用, 但某些操作系统没有这样的高地址, 内存管理单元会将 `0xf010000` 映射为低地址 `0x00100000` (1MB, 正好在 BIOS ROM 的加载位置之上)

256MB 内存的PC的物理地址为 `0x00000000 ~ 0x0fffffff`, 对应的虚拟地址是 `0xf0000000 ~ 0xffffffff`

该实验只需映射前4MB的物理内存就足以启动并运行内核。 静态初始化的页面目录和页表定义在 `kern/entrypgdir.c`

> kern/entry.S 设置了 CR0_PG 标志位, `PG` 是 `CR0` 的位31 分页 `Paging` 标志。当设置该位时即开启分页机制(虚拟地址将转为物理地址), 否则禁止分页机制(所有线性地址等同于物理地址)

 entry_pgdir 会将虚拟地址 `0xf0000000 ~ 0xf0400000` 映射为物理地址 `0x00000000 ~ 0x00400000` (分页机制下), 在非分页时同样将线性地址 `0x00000000 ~ 0x00400000` 也映射到物理地址 `0x00000000 ~ 0x00400000`. 由于没有设置中断处理, 所以在上述两个虚拟地址区间之外的地址会触发硬件异常, 引起 QEMU 转存系统状态并退出

> **练习7**
>
> 用gdb调试JOS, 设置断点在 `movl %eax, %cr0`, 在执行前后检查 0x00100000 和 0xf0100000 内存, 并理解内存地址映射

设置断点在 `0x100025: mov %eax,%cr0`, 执行前

```gdb
(gdb) x/8x 0xf0100000
0xf0100000: 0x00000000 0x00000000 0x00000000 0x00000000
0xf0100010: 0x00000000 0x00000000 0x00000000 0x00000000
(gdb) x/8x 0x00100000
0x100000: 0x1badb002 0x00000000 0xe4524ffe 0x7205c766
0x100010: 0x34000004 0x2000b812 0x220f0011 0xc0200fd8
```

执行后

```gdb
(gdb) x/8x 0xf0100000
0xf0100000: 0x1badb002 0x00000000 0xe4524ffe 0x7205c766
0xf0100010: 0x34000004 0x2000b812 0x220f0011 0xc0200fd8
(gdb) x/8x 0x00100000
0x100000: 0x1badb002 0x00000000 0xe4524ffe 0x7205c766
0x100010: 0x34000004 0x2000b812 0x220f0011 0xc0200fd8
```

这条语句的作用就是开启分页模式, 在执行这条语句之前, 所有的线性地址直接等于物理地址, 执行之后, 线性地址经过 `MMU` 的映射到物理地址

所以执行之前 `0x100000` 处为加载的内核代码, `0xf0100000` 处为空; 执行之后 `0x100000` 和 `0xf0100000` 都映射到物理内存 `0x100000`, 所以它们内容相同。

可以看到 `0xf0100000` 完成了 `0x00100000` 的映射, 下一条指令 `0x100028: mov $0xf010002f,%eax` 可以完成正常的取址操作

> `cr3` 是页目录基址寄存器, 保存页目录表的物理地址, 页目录表以4KB对齐，因此它的地址的低12位总为0

在修改cr0之前修改了cr3寄存器, 将页目录表地址 0x118000 写入了页目录寄存器. entry_pgdir 完成从虚拟地址 [KERNBASE, KERNBASE+4MB) 到物理地址 [0, 4MB) 的映射. 使得再读取 0xf0100000 起始的地址时自动映射到了 0~4M 的某个地址

> kernel.ld 链接脚本文件里指定了 kernel 链接的起始地址, 同时也指定了 boot loader 加载它的物理地址 `.text : AT(0x100000)`

没有不开启分页机制, 当访问高位地址时, 会出现内存访问错误, QEMU执行错误退出

### 格式化输出到控制台

在内核中没有 printf() 这些C标准库, 需要自己来实现IO操作

阅读 kern/printf.c, lib/printfmt.c, kern/console.c 源代码, 理清三者关系

> **练习8**
>
> 实现省略的 "%o" 模式打印八进制数的代码

直接实现在 lib/printfmt.c 中

**回答下列问题:**

> Explain the interface between printf.c and console.c. Specifically, what function does console.c export? How is this function used by printf.c?

- kern/printf.c 提供用户实际调用的接口 `cprintf`
- lib/printfmt.c 提供了函数接口 `vcprintf` 对输出进行格式化
- kern/console.c 提供了处理向屏幕上输出一个字符的底层细节的 `cputchar`, 它被 kern/printf.c 中的 `putch` 包装, 并作为回调函数接口传给 `vcprintf`, 所以 `vcprintf` 做格式化操作不需要关心底层如何操作字符

> Explain the following from console.c

见 console.c 注释

> Trace the execution of the following code step-by-step:

```c
int x = 1, y = 3, z = 4;
cprintf("x %d, y %x, z %d\n", x, y, z);
```

调试结果如下:

```gdb
=> 0xf0100a7a <cprintf>: push   %ebp
cprintf (fmt=0xf0101b12 "x %d, y %x, z %d\n") at kern/printf.c:27

(gdb) x/s 0xf0101b12
0xf0101b12: "x %d, y %x, z %d\n"

=> 0xf0100a43 <vcprintf>: push   %ebp
vcprintf (fmt=0xf0101b12 "x %d, y %x, z %d\n", ap=0xf010ffc4 "\001")
    at kern/printf.c:18

(gdb) x/3xw 0xf010ffc4
0xf010ffc4: 0x00000001 0x00000003 0x00000004
```

> In the call to cprintf(), to what does fmt point? To what does ap point?
>
> List (in order of execution) each call to cons_putc, va_arg, and vcprintf. For cons_putc, list its argument as well. For va_arg, list what ap points to before and after the call. For vcprintf list the values of its two arguments.

`fmt` 指向字符串常量地址 `0xf0101b12`, `ap` 指向第一个参数 `x` 的地址 `0xf010ffc4`, 因为C语言的调用规约时从右往左压栈的, 所以 y 和 z 参数的地址分别为 `0xf010ffc8` 和 `0xf010ffcc`, x 再往栈顶是前一个函数 `cprintf` 栈的 `eax` 的值, 然后是返回地址, 即下一条指令 `f0100101: add $0x1c,%esp`. 再然后是前一个函数栈的 `ebp` 的值-栈基址.

va_start函数根据当前函数栈的 `ebp` 计算出传入的参数的地址 %(ebp)-0xc 即是参数 x 的地址

> Run the following code.

```c
unsigned int i = 0x00646c72;
cprintf("H%x Wo%s", 57616, &i);
```

输出结果为 "He110 World", 如果是大端字节序, 需要将i的值颠倒一下(0x726c6400), 57616不需要调整, 会自动转成大端序

> In the following code, what is going to be printed after 'y='? (note: the answer is not a specific value.) Why does this happen?

```c
cprintf("x=%d y=%d", 3);
```

"y=" 是不确定的, 实际取的值是 x 参数在函数栈中的上一个地址处的值

> Let's say that GCC changed its calling convention so that it pushed arguments on the stack in declaration order, so that the last argument is pushed last. How would you have to change cprintf or its interface so that it would still be possible to pass it a variable number of arguments?

TODO

### 栈

> **练习9**
>
> Determine where the kernel initializes its stack, and exactly where in memory its stack is located. How does the kernel reserve space for its stack? And at which "end" of this reserved area is the stack pointer initialized to point to?

kern/entry.S 第76~77行 `movl $(bootstacktop),%esp` 初始化栈, 调试可以看到栈顶指针 `esp` 初始化为 `0xf0110000` `(0x100034: mov 0xf0110000,%esp)`

```asm
.data
###################################################################
# boot stack
###################################################################
    .p2align    PGSHIFT     # force page alignment
    .globl      bootstack
bootstack:
    .space      KSTKSIZE
    .globl      bootstacktop
bootstacktop:
```

设置栈的方法是在 `.data` 数据段预留一段空间, 定义在 `memlayout.h` 中 `KSTKSIZE (8*4096(PGSIZE))`, 大小为 32K. 栈的位置就是 `0xf0110000 ~ 0xf0118000`

> **练习10**
>
> To become familiar with the C calling conventions on the x86, find the address of the test_backtrace function in obj/kern/kernel.asm, set a breakpoint there, and examine what happens each time it gets called after the kernel starts. How many 32-bit words does each recursive nesting level of test_backtrace push on the stack, and what are those words?

