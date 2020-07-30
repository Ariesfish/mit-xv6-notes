# 环境准备

## 安装编译工具链

- Ubuntu 18.04.4

从仓库安装的工具链即可满足要求，可以省去自己编译工具链的时间。

> 安装编译工具和GDB

```bash
% sudo apt-get install -y build-essential gdb git
```

- GCC: v7.5.0
- GDB: v8.1.0
- Binutils: v2.30
- Git: v2.17.1

> 安装32-bit gcc支持库

```bash
% sudo apt-get install gcc-multilib
```

> 检查编译环境对x86的支持

```bash
% objdump -i
```

显示支持 `elf32-i386`

```bash
% gcc -m32 -print-libgcc-file-name
```

在我的环境上是 `/usr/lib/gcc/x86_64-linux-gnu/7/32/libgcc.a`

## 编译QEMU

MIT给QEMU(v2.3.0)打过了补丁，为了更好地进行实验，编译QEMU模拟器

> 安装依赖库

```bash
% sudo apt-get install -y libsdl1.2-dev libtool-bin libglib2.0-dev libz3-dev libpixman-1-dev
```

> 下载、编译并安装QEMU

```bash
% git clone https://github.com/mit-pdos/6.828-qemu.git qemu
% cd qemu
% ./configure --disable-kvm --disable-werror --prefix=/usr/local --target-list="i386-softmmu x86_64-softmmu"
% make
% sudo make install
```

> 查看QEMU版本

```bash
% qemu-system-i386 --version
QEMU emulator version 2.3.0
```
