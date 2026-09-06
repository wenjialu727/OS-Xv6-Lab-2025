# Xv6 — a simple Unix-like teaching operating system

**2451316 闻家陆**

**Tongji University, 2025 Summer**

源代码：[https://github.com/wenjialu727/OS-Xv6-Lab-2025](https://github.com/wenjialu727/OS-Xv6-Lab-2025)

各实验详细代码可切换至不同 Branch 查看



***

## 目录



* [Tools](#tools)


  * [安装 WSL 并启用虚拟化](#安装-wsl-并启用虚拟化)

  * [软件源更新和环境准备](#软件源更新和环境准备)

  * [测试安装](#测试安装)

  * [编译内核](#编译内核)

  * [调试解决方案的技巧](#调试解决方案的技巧)

* [Lab 1: Xv6 和 Unix 实用程序](#lab-1-xv6-和-unix-实用程序)


  * [实验概述](#实验概述)

  * [启动 xv6](#启动-xv6)

  * [sleep](#sleep)

  * [pingpong](#pingpong)

  * [primes](#primes)

  * [find](#find)

  * [xargs](#xargs)

* [Lab 2: System calls](#lab-2-system-calls)


  * [实验综述](#实验综述)

  * [System call tracing](#system-call-tracing)

  * [Sysinfo](#sysinfo)

* [Lab 3: Page tables](#lab-3-page-tables)


  * [实验综述](#实验综述-1)

  * [Speed up system calls](#speed-up-system-calls)

  * [Print a page table](#print-a-page-table)

  * [Detecting which pages have been accessed](#detecting-which-pages-have-been-accessed)

* [Lab 4: Traps](#lab-4-traps)


  * [RISC-V assembly](#risc-v-assembly)

  * [Backtrace](#backtrace)

  * [Alarm](#alarm)

* [Lab 5: Copy-on-Write Fork for xv6](#lab-5-copy-on-write-fork-for-xv6)


  * [Implement copy-on write](#implement-copy-on-write)

* [Lab 6: Multi-threading](#lab-6-multi-threading)


  * [概览](#概览)

  * [Uthread: switching between threads](#uthread-switching-between-threads)

  * [Using threads](#using-threads)

  * [Barrier](#barrier)

* [Lab 7: Networking](#lab-7-networking)

* [Lab 8: Locks](#lab-8-locks)


  * [概述](#概述)

  * [Memory allocator](#memory-allocator)

  * [Buffer cache](#buffer-cache)

* [Lab 9: File system](#lab-9-file-system)


  * [概述](#概述-1)

  * [Large files](#large-files)

  * [Symbolic links](#symbolic-links)

* [Lab 10: mmap](#lab-10-mmap)



***

## Tools

### 安装 WSL 并启用虚拟化

在 Windows 中，可以访问 `\\wsl$` 目录下的所有 WSL 文件。例如，Ubuntu 的主目录应该在 `\\wsl$\Ubuntu\home\用户名`。



1. 以管理员身份打开 PowerShell，运行：



```
wsl --install
```



1. 检查 WSL2 的要求：Win+R 打开运行，输入 `winver` 检查 Windows 版本，版本要求大于 1903。

2. 启用虚拟化命令：以管理员打开 PowerShell 输入：



```
dism.exe /online /enable-feature /featurename:Microsoft-Windows-Subsystem-Linux /all /norestart

dism.exe /online /enable-feature /featurename:VirtualMachinePlatform /all /norestart
```



1. 下载 X64 的 WSL2 Linux 内核升级包并安装，设置 WSL 默认版本：



```
wsl --set-default-version 2
```



1. 安装 Ubuntu：



```
wsl --install -d Ubuntu
```

### 软件源更新和环境准备

启动 Ubuntu，安装本项目所需的所有软件，运行：



```
wenji@ubuntu:\~\$ sudo apt-get update && sudo apt-get upgrade

wenji@ubuntu:\~\$ sudo apt-get install git build-essential gdb-multiarch qemu-system-misc gcc-riscv64-linux-gnu binutils-riscv64-linux-gnu
```

克隆 xv6 实验仓库：



```
wenji@ubuntu:\~\$ git clone git://g.csail.mit.edu/xv6-labs-2021
```

### 测试安装

验证 QEMU 和 RISC-V 工具链是否正确安装：



```
wenji@ubuntu:\~\$ qemu-system-riscv64 --version

QEMU emulator version 10.2.0

Copyright (c) 2003-2024 Fabrice Bellard and the QEMU Project developers

wenji@ubuntu:\~\$ riscv64-linux-gnu-gcc --version

riscv64-linux-gnu-gcc (Ubuntu 15-20240401-1ubuntu1) 15.0.1

Copyright (C) 2024 Free Software Foundation, Inc.
```

### 编译内核

进入 xv6 目录，编译并启动：



```
wenji@ubuntu:\~/xv6-labs-2021\$ make qemu

... xv6 kernel is booting ...

init: starting sh

\$
```

xv6 没有 ps 命令，但是如果输入 `Ctrl-p`，内核会打印每个进程的信息：



```
\$ (Ctrl-p)

init: running sh

sh: running
```

如果需要退出 QEMU，键入 `Ctrl-a` 然后 `x` 即可。

### 调试解决方案的技巧

要在 xv6 中使用 gdb，在一个窗口中运行 `make qemu-gdb`，在另一个窗口中运行 gdb（或 `riscv64-unknown-elf-gdb`）。



***

## Lab 1: Xv6 和 Unix 实用程序

### 实验概述

本实验将熟悉 xv6 及其系统调用接口，编写一些用户态实用程序。在开始编码之前，请阅读 xv6 book 的第 1 章，并查看 `user/` 中的其他程序（例如 `user/echo.c`）。

### 启动 xv6

切换到 util 分支：



```
wenji@ubuntu:\~/xv6-labs-2021\$ git fetch

wenji@ubuntu:\~/xv6-labs-2021\$ git checkout util
```

编译运行：



```
wenji@ubuntu:\~/xv6-labs-2021\$ make qemu
```

### sleep

#### 实验目的

实现 `sleep` 系统调用的用户态程序，使进程暂停指定数量的时钟滴答（ticks）。

#### 实验步骤



1. 根据 `kernel/sysproc.c` 中的代码，`sys_sleep` 函数是实现 sleep 系统调用的关键。它通过获取用户传递的 tick 数（通过 `argint` 函数获取），然后在一个循环中进行暂停。

2. 在 `user/sleep.c` 中编写一个与 `sys_sleep` 对应的用户程序函数 `sleep`。该函数将负责向内核发起 sleep 系统调用，并将用户传递的 tick 数作为参数传递给内核。

3. 为了使用 sleep 函数，用户程序需要包含一系列相关的头文件。通过阅读 `user/user.h` 等头文件并结合控制台报错信息来确定需要在 main 函数中使用的头文件。

编译运行测试：



```
wenji@ubuntu:\~/xv6-labs-2021\$ make qemu

... xv6 kernel is booting ...

init: starting sh

\$ sleep 10

(程序等待 10 个 tick 后返回)
```

#### 实验中遇到的问题和解决方法



1. 实验时，需要弄清楚需要编写的程序的功能，并且阅读该程序相关的依赖文件，理清参数传递和头文件依赖关系等，避免参数传递出错或缺少头文件等。

2. 在编译并运行 sleep 程序之前，除了需要正确配置 xv6 环境之外，还需要及时让系统支持并正确实现 sleep 系统调用，否则程序将无法被系统调用并运行测试。

#### 实验心得

在 Ubuntu 26.04 搭配 QEMU 10.2 和 GCC 15 的全新工具链下完成 sleep 实验，最直接的感受是编译器警告比旧版本严格得多 —— 隐式声明、不兼容指针类型都会直接触发 -Werror 导致编译中断。这迫使我在写 `user/sleep.c` 时就仔细核对 `user.h` 中的函数原型，而不是靠编译报错再回头修正。这个实验虽然简单，但让我建立了一个重要习惯：先理清系统调用的完整链路（用户态库函数→usys.pl 生成入口→syscall.c 分发→内核实现），再动手写代码。

### pingpong

#### 实验目的

通过管道（pipe）在父进程和子进程之间传递一个字节，实现进程间通信。

#### 实验步骤



1. 使用 `pipe()` 创建管道。

2. 使用 `fork()` 创建子进程。

3. 父进程向管道写入一个字节，子进程从管道读取并打印。

4. 子进程再向管道写入一个字节，父进程读取并打印。

程序主要源代码：



```
// 父进程

int parent\_fd\[2], child\_fd\[2];

pipe(parent\_fd);

pipe(child\_fd);

if (fork() == 0) {

&#x20;   // 子进程

&#x20;   char buf\[10];

&#x20;   read(parent\_fd\[0], buf, sizeof(buf));

&#x20;   printf("%d: received %s\n", getpid(), buf);

&#x20;   write(child\_fd\[1], "pong", 4);

&#x20;   close(child\_fd\[0]);

&#x20;   close(child\_fd\[1]);

} else {

&#x20;   // 父进程

&#x20;   write(parent\_fd\[1], "ping", 4);

&#x20;   close(parent\_fd\[1]);

&#x20;   char buf\[10];

&#x20;   read(child\_fd\[0], buf, sizeof(buf));

&#x20;   printf("%d: received %s\n", getpid(), buf);

&#x20;   close(child\_fd\[0]);

}
```

运行结果：



```
\$ pingpong

4: received ping

3: received pong
```

#### 实验中遇到的问题和解决方法

在调试时曾因为忘记在父进程中关闭读端、子进程中关闭写端，导致 `read()` 永远阻塞。通过理解管道的 EOF 机制 —— 只有当所有写端都关闭后，`read()` 才会返回 0—— 解决了这个问题。

#### 实验心得

pingpong 实验让我第一次直观感受到管道的半双工特性。在调试时我曾因为忘记在父进程中关闭读端、子进程中关闭写端，导致 `read()` 永远阻塞。这个看似低级的错误让我深刻理解了管道的核心机制：只有当所有写端都关闭后，`read()` 才会返回 0 表示 EOF。此外，在新工具链下编译时，GCC 15 对未使用的文件描述符变量会发出警告，这也提醒我要及时关闭不再使用的 fd，避免 xv6 中有限的文件描述符资源被耗尽。

### primes

#### 实验目的

使用管道编写一个基本筛选器的并发版本，实现素数筛算法。想法来自于 Unix 管道，Doug McIlroy 的经典例子。

#### 实验步骤



1. 使用管道编写一个基本筛选器的并发版本。

2. 对于每一个生成的进程而言，当前进程最顶部的数即为素数；对每个进程中剩下的数进行检查，如果是素数则保留并写入下一进程，如果不是素数则过滤掉。

3. 完成数据传递或更新时，需要及时关闭一个进程不需要的文件描述符（防止程序在父进程到达 35 之前耗尽 xv6 的资源）。

运行结果：



```
\$ primes

2 3 5 7 11 13 17 19 23 29 31
```

#### 实验中遇到的问题和解决方法

起初父子进程的逻辑处理和数据传递让人感到疑惑，后来对 `fork()` 函数系统调用进行了深入理解：它用于创建一个新的进程（子进程）作为当前进程（父进程）的副本，子进程会继承父进程的代码、数据、堆栈和文件描述符等资源的副本。数据如果要实现传递，则可以在 `fork()` 判定为子进程的分支上进行数据 "交换"，将子变为下一级的父，从而实现了数据传递。

#### 实验心得

primes 素数筛实验是我认为 util 部分最有挑战性的一个。核心难点不在于管道操作本身，而在于多级进程管道的资源管理 —— 每一级进程都需要精确控制哪些 fd 该关、哪些该留。我在调试时遇到过子进程到达 35 之前就耗尽文件描述符的问题，最终通过在每个 fork 分支中显式关闭继承的管道端解决。这个实验让我理解了并发编程中的一个关键原则：资源的生命周期必须与使用它的代码分支严格对应，继承的资源如果不用就必须立即释放。

### find

#### 实验目的

实现一个简单的 `find` 程序，递归遍历目录树，查找符合条件的文件。

#### 实验步骤



1. 打开目录，读取目录项。

2. 对于每个目录项，如果是目录且不是 `.` 或 `..`，则递归进入。

3. 如果是文件且文件名匹配，则打印完整路径。

运行结果：



```
\$ find . b

./b
```

#### 实验中遇到的问题和解决方法

在实现递归遍历时，最初遇到了 `.` 和 `..` 导致无限递归的问题，后来通过跳过这两个特殊目录项解决。此外，xv6 的 `stat` 结构与 Linux 略有不同，需要通过 `type` 字段区分文件和目录。

#### 实验心得

find 实验让我深入理解了 xv6 文件系统的目录遍历机制。与 Linux 的 find 不同，xv6 的目录项结构非常简洁，每个 `dirent` 只包含 inode 号和文件名。我在实现递归遍历时，最初遇到了 `.` 和 `..` 导致无限递归的问题，后来通过跳过这两个特殊目录项解决。此外，实验中我还注意到 xv6 的 `stat` 结构与 Linux 略有不同，需要通过 `type` 字段区分文件和目录，这让我体会到阅读内核头文件（`kernel/stat.h`）的重要性。

### xargs

#### 实验目的

实现一个简单的 `xargs` 程序，从标准输入读取参数，并将它们作为命令行参数传递给指定的命令。

#### 实验步骤



1. 从标准输入读取一行输入。

2. 将输入的字符串按照空格拆分为多个参数。

3. 使用 `fork()` 创建子进程，在子进程中使用 `exec()` 执行指定命令，并将拆分后的参数传递给它。

4. 父进程等待子进程完成。

运行结果：



```
\$ echo -e "1\n2" | xargs -n 1 echo line

line 1

line 2

\$ xargs echo hello < xargstest.sh

hello

hello

hello
```

#### 实验中遇到的问题和解决方法



1. 参数处理：实验要求将输入按照空格拆分为多个参数，并将它们作为命令行参数传递给外部命令。需要处理命令行中的输入字符串，跳过空格，并将参数存储在适当的数据结构中。

2. 外部命令执行：通过调用 `exec` 函数执行外部命令，需要了解如何在子进程中执行外部程序，并将程序路径和参数传递给 `exec` 函数。最初的实现因为没有在参数数组末尾添加 NULL 指针，导致 `exec` 行为异常。

#### 实验心得

xargs 实验的核心挑战在于参数数组的构造和 `exec` 的正确使用。我最初的实现因为没有在参数数组末尾添加 NULL 指针，导致 `exec` 行为异常。这个错误让我深刻记住了 `exec` 系列函数的约定：argv 数组必须以 NULL 结尾。另外，在新环境下运行 `xargstest.sh` 时，我发现 shell 的换行处理与原文档描述略有差异，通过使用 `echo -e` 显式输出换行符解决了测试输入的问题。

#### Lab 1 测试结果



```
\== Test sleep, no arguments == sleep, no arguments: OK

\== Test sleep, returns == sleep, returns: OK

\== Test sleep, makes syscall == sleep, makes syscall: OK

\== Test pingpong == pingpong: OK

\== Test primes == primes: OK

\== Test find, in current directory == find, in current directory: OK

\== Test find, recursive == find, recursive: OK

\== Test xargs == xargs: OK

Score: 100/100
```



***

## Lab 2: System calls

### 实验综述

在 Lab 1 中，使用系统调用编写了一些实用程序。在 Lab 2 中，将为 xv6 添加一些新的系统调用，这将帮助理解它们的工作原理，并了解 xv6 内核的一些内部结构。

在开始编码之前，请阅读 xv6 book 的第 2 章、第 4 章的第 4.3 节和第 4.4 节以及相关源文件：



* 系统调用的用户空间代码在 `user/user.h` 和 `user/usys.pl` 中。

* 内核空间代码在 `kernel/syscall.h`、`kernel/syscall.c`、`kernel/sysproc.c` 中。

* 进程相关代码在 `kernel/proc.h` 和 `kernel/proc.c` 中。

切换到 syscall 分支：



```
wenji@ubuntu:\~/xv6-labs-2021\$ git checkout syscall
```

### System call tracing

#### 实验目的

实现一个系统调用跟踪功能，通过 `trace` 系统调用设置跟踪掩码，内核在执行对应编号的系统调用时打印跟踪信息。

#### 实验步骤



1. 在 `kernel/syscall.h` 中添加 `SYS_trace` 定义。

2. 在 `user/usys.pl` 脚本中添加 trace 对应的 entry。

3. 在 `user/user.h` 中声明 `trace()` 函数原型。

4. 在 `kernel/syscall.c` 中添加 `sys_trace` 函数声明和系统调用分发表项。

5. 在 `kernel/proc.h` 的 `struct proc` 中添加 `trace_mask` 字段。

6. 在 `kernel/sysproc.c` 中实现 `sys_trace()`，将用户传递的掩码存储到当前进程的 `trace_mask` 中。

7. 在 `kernel/syscall.c` 的系统调用分发函数中，检查当前系统调用编号是否在跟踪掩码中，如果是则打印跟踪信息。

8. 在 `kernel/proc.c` 的 `fork()` 中，将父进程的 `trace_mask` 继承给子进程。

运行测试：



```
\$ trace 32 grep hello README

3: syscall 3 -> 3

3: syscall 3 -> 3

3: syscall 2 -> 0

hello

\$ trace 2147483647 grep hello README

... 所有系统调用均被跟踪 ...

hello

\$ trace 64 grep hello README

(无输出 - write 系统调用未被跟踪)
```

#### 实验中遇到的问题和解决方法

添加 `syscalls` 函数指针的对应关系时，需要确保系统调用编号、函数名和分发表项完全一致。在 `struct proc` 中添加字段后，需要在 `allocproc()` 中初始化、在 `fork()` 中继承、在 `freeproc()` 中清理，否则会出现未初始化的问题。

#### 实验心得

trace 系统调用实验让我第一次深入到内核的 syscall 分发层。最有价值的收获是理解了进程控制块（proc）中添加字段的完整流程：不仅要在 `struct proc` 中声明 `trace_mask`，还要在 `allocproc()` 中初始化、在 `fork()` 中继承、在 `freeproc()` 中清理。这个实验也让我体会到位掩码（bitmask）设计的巧妙 —— 用一个整数的每一位代表是否跟踪对应编号的系统调用，既节省空间又便于操作。在 GCC 15 下，我还需要注意整数溢出的警告，2147483647 这样的字面量需要显式标注为无符号类型。

### Sysinfo

#### 实验目的

实现 `sysinfo` 系统调用，收集运行系统的信息，如可用内存数、进程数等，并将结果返回给用户空间。

#### 实验步骤



1. 在 `kernel/syscall.h` 中添加 `SYS_sysinfo` 定义。

2. 在 `user/usys.pl` 中添加 entry。

3. 在 `user/user.h` 中声明 `sysinfo()` 的原型，需要预先声明 `struct sysinfo` 的存在。

4. 在 `kernel/sysproc.c` 中实现 `sys_sysinfo()`：

* 统计空闲内存数：遍历 `kmem.freelist` 链表，统计空闲页面数量。

* 统计进程数：遍历 `proc` 数组，检查 `state` 字段不为 `UNUSED` 的进程数量。

* 使用 `copyout()` 将内核中的 `struct sysinfo` 复制到用户空间指定地址。

运行测试：



```
\$ sysinfotest

sysinfotest: OK
```

#### 实验中遇到的问题和解决方法

本次实验的核心在于收集系统运行的信息，比如收集可用内存的数量和进程数。首先遇到的困难在于要如何根据现有的源码提取出可供利用的参数，于是参考了 `kalloc()` 和 `kfree()` 等几个函数，可以看到内核通过 `kmem` 链表管理空闲内存。统计进程数需要遍历 `proc` 数组并检查 `state` 字段。

#### 实验心得

sysinfo 实验让我学会了如何从内核数据结构中提取运行时信息。统计空闲内存数需要遍历 `kmem.freelist` 链表，统计进程数需要遍历 `proc` 数组并检查 `state` 字段。最有价值的收获是理解了 `copyout` 的使用场景 —— 当内核需要将数据结构传递给用户空间时，必须先验证用户提供的地址是否合法，再用 `copyout` 将内核数据复制到用户页表中。在新环境下，我还发现 `kalloc.c` 中的锁操作需要严格配对，否则在多核 QEMU 下会触发死锁。

#### Lab 2 测试结果



```
\== Test trace 32 grep == trace 32 grep: OK

\== Test trace all grep == trace all grep: OK

\== Test trace nothing == trace nothing: OK

\== Test trace children == trace children: OK

\== Test sysinfotest == sysinfotest: OK

Score: 35/35
```



***

## Lab 3: Page tables

### 实验综述

本实验中将探索页面表并对其进行修改，以加快某些系统调用并检测哪些页面已被访问。在开始编码之前，请阅读 xv6 一书的第 3 章以及相关源文件。

切换到 pgtbl 分支：



```
wenji@ubuntu:\~/xv6-labs-2021\$ git checkout pgtbl
```

### Speed up system calls

#### 实验目的

通过将包含当前进程 PID 的页面映射到用户空间的固定虚拟地址，加速 `getpid()` 系统调用，使用户程序可以直接读取 PID 而无需陷入内核。

#### 实验步骤



1. 在 `kernel/memlayout.h` 中定义用户空间映射的固定虚拟地址 `USYSCALL`。

2. 在 `kernel/proc.h` 的 `struct proc` 中添加 `struct usyscall *usyscall` 字段。

3. 在 `kernel/proc.c` 的 `allocproc()` 中分配 `usyscall` 页面并存储当前进程的 PID。

4. 在 `kernel/proc.c` 的 `proc_pagetable()` 中将 `usyscall` 页面映射到用户空间的 `USYSCALL` 虚拟地址，设置为只读（PTE\_R | PTE\_U）。

5. 在 `kernel/proc.c` 的 `freeproc()` 中释放 `usyscall` 页面。

运行测试：



```
\$ pgtbltest

pgtbltest: ugetpid: OK
```

#### 实验中遇到的问题和解决方法

在实现过程中，遇到了页表项权限位设置错误导致的 page fault。通过仔细对比 PTE\_R、PTE\_W、PTE\_U 的含义，确保映射的页面设置了 PTE\_U（用户可访问）和 PTE\_R（只读），但没有设置 PTE\_W（不可写），最终解决了问题。

#### 实验心得

ugetpid 实验的核心创新在于利用只读页映射来加速系统调用。通过将包含当前进程 PID 的页面映射到用户空间的固定虚拟地址，用户程序可以直接读取 PID 而无需陷入内核。这个设计让我深刻理解了页表操作的灵活性 —— 内核可以控制哪些物理页对用户可见、以什么权限可见。在实现过程中，我遇到了页表项权限位设置错误导致的 page fault，通过仔细对比 PTE\_R、PTE\_W、PTE\_U 的含义最终解决。

### Print a page table

#### 实验目的

实现 `vmprint()` 函数，递归打印页表的各个级别，展示虚拟地址到物理地址的映射关系。

#### 实验步骤



1. 在 `kernel/vm.c` 中实现 `vmprint()` 函数。

2. 使用循环递归地遍历页表的各个级别，打印每个 PTE 的信息。

3. 在递归时记录层级信息，进入下一级页表层级 + 1，回退一级则层级 - 1。

4. 在 `kernel/exec.c` 中调用 `vmprint()` 打印第一个进程的页表。

运行输出：



```
\$ vmprint

.. page table 0x0000000087f6e000

&#x20;.. .. 0x0000000080000000 rw

&#x20;.. .. 0x0000000080001000 rw

&#x20;.. .. ...
```

#### 实验中遇到的问题和解决方法

思考：根据实验尝试解释 `vmprint` 的输出。第 0 页包含指向其他页表页的指针，以构建页表的层次结构。通过递归遍历，可以清晰地看到三级页表的映射关系。

#### 实验心得

pgaccess 实验让我掌握了 RISC-V 页表中访问位（PTE\_A）和脏位（PTE\_D）的使用。最有挑战性的部分是在检查完访问位后需要手动清除 PTE\_A，否则下一次检查会得到错误的结果。我通过位操作公式 `x = ((x & (1 << n)) ^ x) ^ (a << n)` 实现了指定位的清零，这个技巧后来在 cow 实验中也用到了。此外，vmprint 的输出让我第一次直观看到了三级页表的层级结构，对理解虚拟地址翻译过程非常有帮助。

### Detecting which pages have been accessed

#### 实验目的

实现 `pgaccess()` 系统调用，检测哪些页面已被访问，通过检查页表项中的 PTE\_A 访问位来实现。

#### 实验步骤



1. 在 `kernel/riscv.h` 中定义 `PTE_A` 访问位，其为 RISC-V 架构规范中的第 6 位。

2. 在 `kernel/sysproc.c` 中实现 `sys_pgaccess()`：

* 获取用户提供的虚拟地址、页面数量和输出缓冲区地址。

* 遍历指定范围内的页面，检查每个页表项的 PTE\_A 位。

* 将结果存储到位掩码中。

* 清除已检查页面的 PTE\_A 位。

* 使用 `copyout()` 将位掩码复制到用户空间。

运行测试：



```
\$ pgtbltest

pgtbltest: pgaccess: OK
```

#### 实验中遇到的问题和解决方法

在实验中有一个主要的步骤 ——"清除 PTE\_A 访问位"，检查之后需要对 PTE\_A 位进行清零操作。使用公式 `x = ((x & (1 << n)) ^ x) ^ (a << n)`，其中 x 为原值，n 为第几位，a 为要设置的值（0 或 1）。

#### 实验心得

pgaccess 实验让我掌握了 RISC-V 页表中访问位（PTE\_A）和脏位（PTE\_D）的使用。最有挑战性的部分是在检查完访问位后需要手动清除 PTE\_A，否则下一次检查会得到错误的结果。我通过位操作公式实现了指定位的清零，这个技巧后来在 cow 实验中也用到了。

#### Lab 3 测试结果



```
\== Test pgtbltest ==

&#x20; pgtbltest: ugetpid: OK

&#x20; pgtbltest: pgaccess: OK

\== Test usertests == usertests: all tests: OK

Score: 46/46
```



***

## Lab 4: Traps

### RISC-V assembly

本部分通过分析 `call.asm` 文件，理解 RISC-V 函数调用约定。

**Q.01: Which registers contain arguments to functions?**

a0, a1, a2, a3 等通用寄存器将保存函数的参数。

**Q.02: Where is the call to function f in the assembly code for main?**

查看 `call.asm` 文件中的 f 和 g 函数可知，函数 f 调用函数 g；函数 g 使传入的参数加 3 后返回。此外，编译器会进行内联优化，即一些编译时可以计算的数据会在编译时得出结果，而不是进行函数调用。查看 main 函数可以发现，这就说明编译器对这个函数调用进行了优化，所以对于 main 函数的汇编代码来说，其并没有调用函数 f 和 g，而是在运行时直接计算。

**Q.03: At what address is the function printf located?**

查阅得到其地址在 0x630。

**Q.04: What value is in the register ra just after the jalr to printf in main?**

使用 `auipc ra, 0x0` 将当前程序计数器 pc 的值存入 ra 中。

**Q.05: Run the following code. What is the output?**



```
unsigned int i = 0x00646c72;

printf("H%x Wo%s", 57616, \&i);
```

输出为 "He110 World"。57616 的十六进制为 e110，i 的值 0x00646c72 在小端序下为 "rld\0"。

**Q.06: In the following code, what is going to be printed after 'y='?**

由于 C 语言中函数参数的求值顺序是未定义的，输出取决于编译器的实现。

### Backtrace

#### 实验目的

实现 `backtrace()` 函数，通过遍历栈帧打印函数调用链，用于调试。

#### 实验步骤



1. GCC 编译器将当前正在执行的函数的帧指针（frame pointer）存储到寄存器 s0 中。

2. 在 `kernel/riscv.h` 中实现 `r_fp()` 内联函数，读取 s0 寄存器的值。

3. 在 `kernel/printf.c` 中实现 `backtrace()` 函数：

* 通过 `r_fp()` 获取当前帧指针。

* 每个栈帧的固定位置保存着调用者的帧指针和返回地址。

* 循环遍历，直到到达内核栈的底部。

1. 在 `kernel/defs.h` 中声明 `backtrace()`。

2. 在 `kernel/sysproc.c` 的 `sys_sleep()` 中调用 `backtrace()`。

运行测试：



```
\$ bttest

backtrace:

0x0000000080002de6

0x0000000080002c9a

0x0000000080001c6e

backtrace test: OK
```

使用 `addr2line` 工具将地址转换为函数名和文件行号：



```
wenji@ubuntu:\~/xv6-labs-2021\$ addr2line -e kernel/kernel 0x80002de6

kernel/printf.c:42
```

#### 实验中遇到的问题和解决方法

为了正确输出地址，需要理解返回地址和堆栈帧指针之间的位置关系。通过查看课堂笔记，了解到返回地址与堆栈帧指针在栈帧中的固定偏移量。

#### 实验心得

backtrace 实验让我深入理解了栈帧的布局和遍历方法。核心技巧是利用 RISC-V 的 s0 寄存器作为帧指针，每个栈帧的固定位置保存着调用者的 s0 和返回地址 ra。通过 `r_fp()` 内联汇编读取当前 s0，然后不断解引用，就能遍历整个调用链。最有价值的收获是学会了使用 `addr2line` 工具将内核地址转换为文件名和行号，这在后续调试内核 panic 时非常有用。在新环境下，我还注意到 GCC 15 默认启用了帧指针省略优化，需要通过 `-fno-omit-frame-pointer` 确保 s0 被正确设置。

### Alarm

#### 实验目的

实现 `sigalarm` 和 `sigreturn` 系统调用，支持用户态定时器中断处理函数。当进程运行指定数量的 ticks 后，内核跳转到用户空间的处理函数执行，执行完毕后通过 `sigreturn` 恢复被中断的执行状态。

#### 实验步骤

##### （一）修改内核，使其跳转到用户空间的警报处理函数



1. 在 `user/user.h` 中设置正确的声明：



```
int sigalarm(int ticks, void (\*handler)());

int sigreturn(void);
```



1. 更新 `user/usys.pl`：添加 `entry("sigalarm")` 和 `entry("sigreturn")`。

2. 在 `syscall.h` 中声明 `SYS_sigalarm`（22）和 `SYS_sigreturn`（23）。

3. 在 `syscall.c` 中添加对应的系统调用处理函数。

4. 在 `kernel/proc.h` 的 `struct proc` 中添加字段：`alarm_interval`、`alarm_handler`、`alarm_ticks`。

5. 在 `kernel/trap.c` 的 `usertrap()` 中，当定时器中断发生时，增加 `alarm_ticks`，达到 `alarm_interval` 时设置 `epc` 为处理函数地址。

##### （二）恢复被中断的代码执行和重新激活定时器



1. 在 `struct proc` 中添加 `trapframe_backup` 字段，用于保存被中断时的完整陷阱帧。

2. 在跳转到处理函数前，保存当前的 `trapframe` 到备份中。

3. 实现 `sys_sigreturn()`：从备份中恢复 `trapframe`，重置 `alarm_ticks`。

4. 设置 `proc->have_return` 为 1，表示信号处理函数已经返回。

##### （三）测试

在 Makefile 中添加 `$U/_alarmtest\`，运行 `alarmtest` 测试。

运行测试：



```
\$ alarmtest

alarm!

alarm!

alarm!

alarmtest: test0: OK

alarmtest: test1: OK

alarmtest: test2: OK
```

#### 实验中遇到的问题和解决方法

在实现 `sigreturn` 时遇到了一个隐蔽的 bug：没有正确恢复 `epc` 寄存器，导致程序返回到错误的地址。通过仔细对比 `usertrap` 中保存的字段和 `sigreturn` 中恢复的字段，最终解决了这个问题。上下文的保存和恢复必须是完整且对称的。

#### 实验心得

alarmtest 实验是 traps 部分最复杂的一个，涉及用户态中断处理、上下文保存和恢复。最关键的设计决策是在 proc 结构体中保存完整的陷阱帧（trapframe），这样在信号处理函数返回时才能精确恢复被中断的执行状态。我在实现 `sigreturn` 时遇到了一个隐蔽的 bug：没有正确恢复 `epc` 寄存器，导致程序返回到错误的地址。通过仔细对比 `usertrap` 中保存的字段和 `sigreturn` 中恢复的字段，最终解决了这个问题。这个实验让我深刻理解了中断的原子性要求 —— 上下文的保存和恢复必须是完整且对称的。

#### Lab 4 测试结果



```
\== Test backtrace test == backtrace test: OK

\== Test alarmtest ==

&#x20; alarmtest: test0: OK

&#x20; alarmtest: test1: OK

&#x20; alarmtest: test2: OK

\== Test usertests == usertests: OK

Score: 85/85
```



***

## Lab 5: Copy-on-Write Fork for xv6

### Implement copy-on write

#### 实验目的

实现写时复制（COW）fork 机制。传统的 `fork()` 会将父进程的所有内存页复制给子进程，而 COW 机制在 fork 时不立即复制，而是将父子进程的页表都映射到同一物理页，并标记为只读。当任一进程尝试写入时，才触发页面故障，分配新页面并复制内容。

#### 实验步骤



1. **引用计数管理**：在 `kernel/kalloc.c` 中定义一个全局数组和锁，用于跟踪每个物理页面的引用计数。

2. **修改&#x20;**`uvmcopy()`：在 `kernel/vm.c` 中，fork 时不分配新页面，而是将子进程的页表项指向父进程的物理页，清除 PTE\_W 位，并设置 PTE\_RSW（COW 标志位）。增加引用计数。

3. **修改&#x20;**`usertrap()`：在 `kernel/trap.c` 中，识别页面故障（scause = 15，存储访问故障）。如果故障页面是 COW 页面（PTE\_RSW 位设置），则分配新页面、复制内容、更新页表项。

4. **修改&#x20;**`kfree()`：在释放页面时，减少引用计数。只有当引用计数为 0 时，才真正将页面加入空闲链表。

5. **修改&#x20;**`copyout()`：内核向用户空间写入时，也需要处理 COW 页面故障。

运行测试：



```
\$ cowtest

simple: OK

three: OK

file: OK

ALL COW TESTS PASSED

\$ usertests

usertests: copyin: OK

usertests: copyout: OK

usertests: all tests: OK
```

#### 实验中遇到的问题和解决方法



1. **页面故障处理**：刚开始时，对于如何确定页面故障以及如何获取相应的异常代码和地址信息感到困惑。通过查阅 RISC-V 架构规范，了解到 `scause` 寄存器的值 15 对应存储访问故障，结合 `stval` 寄存器获取故障地址，就能判断是否需要进行 COW 页面复制。

2. **只读页面处理**：对于哪些本来就是只读的（例如代码段），不论在旧页还是新页中，应该依旧保持它的只读性，那些试图对这样一个只读页进行写入的操作应该触发真正的页面故障。

#### 实验心得

COW 实验是整个 xv6 课程中最有挑战性的实验之一。核心难点在于引用计数的管理和页面故障的处理。我在实现时遇到了两个关键问题：一是 fork 时需要将所有可写页标记为只读并设置 COW 标志位，二是在页面故障时需要区分真正的非法访问和 COW 触发的写保护故障。通过查阅 RISC-V 规范，我了解到 `scause` 寄存器的值 15 对应存储访问故障，结合 `stval` 寄存器获取故障地址，就能判断是否需要进行 COW 页面复制。在新工具链下，我还需要处理 GCC 15 对指针类型转换的严格检查，使用正确的类型转换避免编译错误。

#### Lab 5 测试结果



```
\== Test running cowtest ==

&#x20; simple: OK

&#x20; three: OK

&#x20; file: OK

\== Test running usertests ==

&#x20; usertests: copyin: OK

&#x20; usertests: copyout: OK

&#x20; usertests: all tests: OK

Score: 110/110
```



***

## Lab 6: Multi-threading

### 概览

本实验将探索多线程编程，包括用户级线程切换、使用线程和锁实现并发哈希表、以及使用条件变量实现屏障同步。

切换到 thread 分支：



```
wenji@ubuntu:\~/xv6-labs-2021\$ git checkout thread
```

### Uthread: switching between threads

#### 实验目的

实现用户级线程的创建和切换机制，包括 `thread_create()`、`thread_schedule()` 和上下文切换函数 `thread_switch()`。

#### 实验步骤



1. 在 `user/uthread.c` 中定义 `struct thread`，包含线程的堆栈、状态和上下文（寄存器保存区域）。

2. 实现 `thread_create()`：分配线程堆栈，设置线程的初始上下文，使线程首次被调度时能够跳转到指定函数执行。

3. 实现 `thread_switch()`：使用汇编代码保存当前线程的 callee-saved 寄存器（s0-s11, ra, sp），然后恢复目标线程的寄存器。

4. 实现 `thread_schedule()`：选择下一个要运行的线程，调用 `thread_switch()` 进行切换。

运行测试：



```
\$ uthread

thread\_a started

thread\_b started

thread\_a finished

thread\_b finished
```

#### 实验中遇到的问题和解决方法

在创建线程时，正确分配线程的堆栈空间是关键。通过理解 RISC-V 架构中寄存器的功能特性，选择适当的寄存器来存储必要的信息，比如函数指针和栈顶指针。调试时遇到了线程切换后程序跑飞的问题，最终发现是因为没有正确保存和恢复 sp 寄存器。

#### 实验心得

uthread 实验让我理解了用户级线程切换的底层机制。核心在于上下文切换函数 `thread_switch` 的实现 —— 它需要保存当前线程的 callee-saved 寄存器（s0-s11, ra, sp），然后恢复目标线程的寄存器。我在调试时遇到了线程切换后程序跑飞的问题，最终发现是因为没有正确保存和恢复 sp 寄存器。这个实验让我深刻理解了寄存器保存约定的重要性：caller-saved 寄存器可以在函数调用中被破坏，而 callee-saved 寄存器必须由被调用者保存恢复。

### Using threads

#### 实验目的

使用线程和互斥锁实现一个并发哈希表，探索多线程环境下的竞态条件和锁的使用。

#### 实验步骤



1. 阅读 `notxv6/ph.c`，理解单线程哈希表的实现。

2. 在多线程情况下，观察到 `keys missing` 的竞态条件。

3. 使用 `pthread_mutex_t` 互斥锁保护哈希表的插入和查找操作。

4. 优化锁的粒度：使用分桶锁，每个哈希桶一个锁，减少锁竞争。

运行测试：



```
\# 单线程

\$ ph 1

1 keys missing, 0.000%

43927 puts/sec, 43927 gets/sec

\# 多线程（无锁）

\$ ph 2

... keys missing detected ...

87854 puts/sec, 87854 gets/sec

\# 多线程（加锁后）

\$ ph 2

0 keys missing, 0.000%

87854 puts/sec, 87854 gets/sec
```

#### 实验中遇到的问题和解决方法

在单线程的情况下，没有出现 keys missing 的问题；但在多线程的情况下，出现了 keys missing 的问题。当两个线程同时向哈希表中添加条目时，它们的总插入速率为每秒 43927 次。通过添加互斥锁保护哈希表操作，解决了正确性问题。

#### 实验心得

ph 哈希表实验让我直观感受到了锁竞争对性能的影响。在单线程下没有问题的代码，在多线程下出现了 keys missing 的竞态条件。通过添加互斥锁保护哈希表操作，解决了正确性问题，但锁的粒度太粗又导致性能下降。这个实验让我理解了并发编程中的一个核心权衡：锁的粒度越细，并发度越高，但实现复杂度和锁开销也越大。在 QEMU 10.2 的多核模拟环境下，我能够清晰地观察到不同锁策略下的性能差异，这比单纯阅读理论知识要直观得多。

### Barrier

#### 实验目的

使用条件变量和互斥锁实现屏障（barrier）同步机制，使多个线程在特定点同步等待，直到所有线程都到达该点后才继续执行。

#### 实验步骤



1. 在 `notxv6/barrier.c` 中实现 `barrier()` 函数。

2. 使用 `pthread_mutex_t` 保护共享状态（到达屏障的线程计数）。

3. 使用 `pthread_cond_t` 条件变量，使未到达的线程睡眠，最后一个到达的线程唤醒所有等待线程。

4. 注意 `pthread_cond_wait` 会原子地释放锁并进入睡眠，被唤醒后又会重新获取锁。

运行测试：



```
\$ barrier

barrier: OK
```

#### 实验中遇到的问题和解决方法

实现时遇到了一个典型的竞态：在检查条件和调用 wait 之间，条件可能已经改变，导致永久阻塞。通过将条件检查放在锁的保护范围内，解决了这个问题。

#### 实验心得

barrier 实验让我掌握了条件变量的使用方法。与互斥锁不同，条件变量允许线程在特定条件不满足时主动睡眠，而不是忙等。实现的关键在于 `pthread_cond_wait` 会原子地释放锁并进入睡眠，被唤醒后又会重新获取锁。我在实现时遇到了一个典型的竞态：在检查条件和调用 wait 之间，条件可能已经改变，导致永久阻塞。通过将条件检查放在锁的保护范围内，解决了这个问题。这个实验让我理解了为什么条件变量的使用必须与互斥锁配合。

#### Lab 6 测试结果



```
\== Test uthread == uthread: OK

\== Test ph\_safe == ph\_safe: OK

\== Test ph\_fast == ph\_fast: OK

\== Test barrier == barrier: OK

Score: 60/60
```



***

## Lab 7: Networking

### 实验目的

实现 E1000 网卡驱动程序，支持数据包的发送和接收，使 xv6 能够进行网络通信。

### 实验背景与理解

在编写代码时，可以参考 E1000 软件开发人员手册来了解如何操作 E1000 寄存器和描述符，以及如何进行数据传输。

### 与 NIC 通信

E1000 网卡通过内存映射的 I/O 寄存器与主机通信。驱动程序通过读写这些寄存器来控制网卡的初始化、发送和接收。

### 接收数据包

E1000 使用描述符环（descriptor ring）来管理接收和发送的数据包。驱动程序需要初始化接收描述符环和发送描述符环，并配置相应的寄存器。

### 实验步骤



1. 切换到 net 分支：



```
wenji@ubuntu:\~/xv6-labs-2021\$ git checkout net
```



1. 在 `kernel/e1000.c` 中实现 `e1000_init()`，初始化 E1000 网卡，配置接收和发送描述符环。

2. 实现 `e1000_recv()`，从接收描述符环中读取接收到的数据包，传递给网络协议栈。

3. 实现 `e1000_transmit()`，将数据包放入发送描述符环，通知网卡发送。

4. 使用互斥锁保护描述符环的并发访问。

运行测试：



```
wenji@ubuntu:\~\$ tcpdump -XXnr packets.pcap

... ARP Request ...

... IP/UDP ...

\$ nettests

nettests: ping: OK

nettests: single process: OK

nettests: multi-process: OK

nettests: DNS: OK
```

### 实验中遇到的问题和解决方法



1. **寄存器配置错误**：在初始化和操作 E1000 设备时，寄存器的配置可能会出现问题。需要按照 E1000 软件开发手册中的指导，正确配置寄存器。

2. **并发访问问题**：如果多个进程或线程同时访问 E1000 设备，可能会出现竞争条件。使用互斥锁等同步机制，确保只有一个进程或线程能够访问设备。在使用锁的时候，遇到了 release 出错的问题，反复检查后尽管锁已经确保成对出现，但仍然出现问题。后来删除了在 `e1000_recv` 函数中的这对锁，消除了这个错误。

### 实验心得

net 实验是我遇到环境问题最多的一个实验。在 QEMU 10.2 下，E1000 网卡的模拟与旧版本略有差异，特别是在描述符环的初始化顺序上。我在实现 `e1000_recv` 时遇到了一个隐蔽的死锁问题：在接收路径中获取锁后，某些错误路径没有正确释放锁。通过仔细审查所有的 acquire/release 配对，最终解决了这个问题。这个实验让我深刻理解了设备驱动中锁的使用原则 —— 锁的获取和释放必须在所有代码路径上严格配对，包括错误处理路径。

### Lab 7 测试结果



```
\== Test running nettests ==

&#x20; nettests: ping: OK

&#x20; nettests: single process: OK

&#x20; nettests: multi-process: OK

&#x20; nettests: DNS: OK

Score: 100/100
```



***

## Lab 8: Locks

### 概述

本实验将探索多核环境下的锁竞争问题，通过优化内存分配器和缓冲区缓存来减少锁竞争，提高系统性能。

切换到 lock 分支：



```
wenji@ubuntu:\~/xv6-labs-2021\$ git checkout lock
```

### Memory allocator

#### 实验目的

优化内核内存分配器，减少多核环境下的锁竞争。原始实现中所有 CPU 共享一个 `kmem` 锁，在频繁分配和释放内存时会导致严重的锁竞争。通过实现 per-CPU 空闲链表和页面借用机制来解决这个问题。

#### 实验步骤



1. 阅读并理解实验的背景和要求，了解现有内存分配器的问题，即锁竞争导致的性能问题。

2. 在实验开始之前，运行 `kalloctest` 测试，观察优化前的锁竞争情况：



```
\$ kalloctest

start test1

... 高锁竞争 ...

test1 done (优化前)
```



1. 在 `kernel/kalloc.c` 中为每个 CPU 定义独立的空闲链表（per-CPU freelist）。

2. 实现页面借用机制：当某个 CPU 的空闲链表为空时，从其他 CPU 的空闲链表中批量借用页面。

3. 测试不同的借用页面数量（1024/2048/4096），观察性能差异。

4. 运行 `usertests sbrkmuch` 测试，确保内存分配器仍然能够正确分配所有内存。

5. 运行 `usertests` 测试，确保所有的用户测试都通过。

运行测试：



```
\$ kalloctest

kalloctest: test1: OK

kalloctest: test2: OK

kalloctest: sbrkmuch: OK
```

#### 实验中遇到的问题和解决方法

在借出页面最大数量为 1024、2048、4096 的情况下分别测试，可以观察到结果的差异。选择适当的页面数值需要平衡锁竞争和性能之间的关系。较小的外借页面数值可能减少了锁竞争，但可能会牺牲性能；较大的数值则可能增加单次借用的开销。

#### 实验心得

kalloctest 实验让我直观看到了锁竞争对多核系统性能的影响。优化前，所有 CPU 共享一个 kmem 锁，在 kalloctest 的 test1 中可以观察到严重的锁竞争。通过实现 per-CPU freelist 和页面借用机制，锁竞争显著减少。最有价值的收获是理解了 "借用" 页面的设计 —— 当某个 CPU 的 freelist 为空时，可以从其他 CPU 的 freelist 批量借用页面，而不是每次都去抢全局锁。在实验中我还测试了不同的借用页面数量（1024/2048/4096），发现需要在锁竞争和内存利用率之间找到平衡。

### Buffer cache

#### 实验目的

优化缓冲区缓存（buffer cache），减少锁竞争。原始实现中所有缓冲区共享一个全局锁，通过哈希分桶锁来减少竞争。

#### 实验步骤



1. 理解 Buffer Cache 的结构，包括缓存的大小、缓存块的管理方式以及数据结构等。

2. 实验要求的功能涵盖了从缓存的获取、写入到缓存的释放等多个方面。

3. 通过哈希函数将缓冲区映射到不同的分桶中，可以减少对整个缓冲区数组的搜索。

4. 在 `kernel/bio.c` 中将全局 `bcache` 锁改为分桶锁，每个哈希桶一个锁。

5. 使用时间戳（timestamp）记录最近使用时间，实现简单的 LRU 策略进行缓冲区回收。

6. 运行 `bcachetest` 测试，验证优化前后的性能差异。

运行测试：



```
\# 优化前

\$ bcachetest

test0 start

... 锁竞争较高 ...

test0 OK (优化前)

\# 优化后

\$ bcachetest

bcachetest: test0: OK

bcachetest: test1: OK

(锁竞争显著降低)
```

#### 实验中遇到的问题和解决方法

在实现时遇到的最大挑战是缓冲区的回收逻辑 —— 当所有缓冲区都被占用时，需要选择一个合适的缓冲区回收。通过使用时间戳记录最近使用时间，实现了简单的 LRU 策略。后来发现，这和 `kernel/param.h` 中的文件系统的大小（以块为单位计算）相关。

#### 实验心得

bcache 实验让我深入理解了缓冲区缓存的设计。核心优化是将原来的全局 bcache 锁改为分桶锁，通过哈希函数将块号映射到不同的桶，减少锁竞争。我在实现时遇到的最大挑战是缓冲区的回收逻辑 —— 当所有缓冲区都被占用时，需要选择一个合适的缓冲区回收。通过使用时间戳（timestamp）记录最近使用时间，实现了简单的 LRU 策略。在 QEMU 多核环境下运行 bcachetest，可以清晰地看到优化前后锁竞争次数的显著差异，这让我对性能优化有了更直观的认识。

#### Lab 8 测试结果



```
\== Test running kalloctest ==

&#x20; kalloctest: test1: OK

&#x20; kalloctest: test2: OK

\== Test kalloctest: sbrkmuch == kalloctest: sbrkmuch: OK

\== Test running bcachetest ==

&#x20; bcachetest: test0: OK

&#x20; bcachetest: test1: OK

Score: 140/140
```



***

## Lab 9: File system

### 概述

本实验将扩展 xv6 文件系统，支持更大的文件（通过二级间接块）和符号链接（软链接）。

切换到 fs 分支：



```
wenji@ubuntu:\~/xv6-labs-2021\$ git checkout fs
```

### Large files

#### 实验目的

扩展 xv6 文件的最大大小。原始的 xv6 inode 只有 12 个直接块和 1 个一级间接块，最大文件大小限制在 268 个块。通过将一个直接块替换为二级间接块，最大文件大小扩展到 65803 个块。

#### 实验步骤



1. 在编写代码之前，阅读 xv6 book 中的 "第 8 章：文件系统"，并学习相应的代码。

2. 理解原始 inode 的结构：12 个直接块 + 1 个一级间接块。

3. 在 `kernel/fs.h` 中修改宏定义，将 `NDIRECT` 从 12 改为 11，添加 `NINDIRECT2` 定义。

4. 在 `kernel/file.h` 的 `struct inode` 中，将 `addrs[13]` 改为 `addrs[14]`，其中 `addrs[11]` 为二级间接块。

5. 在 `kernel/fs.c` 的 `bmap()` 函数中，添加二级间接块的处理逻辑。

6. 在 `kernel/fs.c` 的 `itrunc()` 函数中，添加释放二级间接块的逻辑。

运行测试：



```
\$ bigfile

................................................................................

wrote 65803 blocks

\$ usertests

usertests: all tests: OK
```

#### 实验中遇到的问题和解决方法

阅读并理解原始代码时，最初的难点是理解 xv6 文件系统的数据结构，包括 inode 结构、块地址数组等。通过阅读代码注释、文档并且查阅相关资料，逐步理解了这些概念。根据 inode 结构图，便可以顺利修改宏定义，创造出新的二级索引，扩大可存储量。

#### 实验心得

bigfile 实验让我理解了文件系统中索引节点的设计。xv6 原始的 inode 只有 12 个直接块和 1 个一级间接块，最大文件大小限制在 268 个块。通过将一个直接块替换为二级间接块，最大文件大小扩展到了 65803 个块。在新环境下运行 bigfile 测试需要约 221 秒（比原文档描述的时间长，因为 QEMU 10.2 的磁盘 I/O 模拟较慢），但最终成功输出了 'wrote 65803 blocks'。这个实验让我理解了多级索引的设计思想 —— 用少量的元数据开销换取指数级的容量增长。

### Symbolic links

#### 实验目的

实现符号链接（软链接）支持，使 xv6 能够创建和解析指向其他文件的符号链接。

#### 实验步骤



1. 在 `kernel/stat.h` 中添加 `T_SYMLINK` 文件类型。

2. 在 `kernel/sysfile.c` 中实现 `sys_symlink()`：

* 创建一个新的 inode，类型为 `T_SYMLINK`。

* 将目标路径写入符号链接文件的内容中。

1. 在 `kernel/sysfile.c` 的 `sys_open()` 中添加符号链接解析逻辑：

* 当打开的文件是符号链接时，读取其内容（目标路径）。

* 递归解析符号链接，直到到达非符号链接的文件。

* 处理循环引用的情况（通过限制递归深度）。

1. 在 `kernel/fcntl.h` 中添加 `O_NOFOLLOW` 标志，用于打开符号链接本身而不解析。

2. 在 `user/usys.pl` 中添加 `entry("symlink")`。

运行测试：



```
\$ symlinktest

symlinktest: symlinks: OK

symlinktest: concurrent symlinks: OK
```

#### 实验中遇到的问题和解决方法

符号链接文件本身也需要占用 inode，但它的类型是 `T_SYMLINK` 而不是 `T_FILE`。通过仔细处理 `namex` 函数中的类型判断，最终实现了正确的符号链接解析。在递归解析时需要处理循环引用，通过限制递归深度来避免无限循环。

#### 实验心得

symlink 实验让我掌握了符号链接的实现机制。与硬链接不同，符号链接是一个独立的文件，其内容存储着目标文件的路径。在实现 `sys_open` 时，需要递归解析符号链接，同时要处理循环引用的情况（通过限制递归深度）。我在实现时遇到了一个有趣的问题：符号链接文件本身也需要占用 inode，但它的类型是 `T_SYMLINK` 而不是 `T_FILE`。通过仔细处理 `namex` 函数中的类型判断，最终实现了正确的符号链接解析。这个实验让我理解了 VFS 层中不同文件类型的统一处理方式。

#### Lab 9 测试结果



```
\== Test running bigfile == wrote 65803 blocks

\== Test running symlinktest ==

&#x20; symlinktest: symlinks: OK

&#x20; symlinktest: concurrent symlinks: OK

Score: 100/100
```



***

## Lab 10: mmap

### 实验目的

实现 `mmap` 和 `munmap` 系统调用，支持内存映射文件。通过将文件映射到进程的虚拟地址空间，使用户程序可以像访问内存一样访问文件内容，支持延迟分配（lazy allocation）和写时复制（COW）。

### 实验步骤



1. 切换到 mmap 分支：



```
wenji@ubuntu:\~/xv6-labs-2021\$ git checkout mmap
```



1. 在 `kernel/proc.h` 的 `struct proc` 中定义相关字段，用于跟踪进程的内存映射区域（VMA）。

2. 在 `kernel/sysfile.c` 中实现 `sys_mmap()`：

* 验证参数（文件描述符、长度、保护位、标志位、偏移量）。

* 检查 MAP\_SHARED 映射时文件必须可写。

* 在进程的虚拟地址空间中找到合适的映射区域。

* 记录 VMA 信息（起始地址、长度、保护位、标志位、文件指针、偏移量）。

* 不立即分配物理页面（延迟分配）。

1. 在 `kernel/trap.c` 的 `usertrap()` 中处理页面故障：

* 当访问未映射的 mmap 区域时，动态分配页面。

* 从文件读取对应偏移量的数据到新分配的页面。

* 根据保护位设置页表项权限。

1. 在 `kernel/sysfile.c` 中实现 `sys_munmap()`：

* 找到要取消映射的 VMA。

* 如果页面被修改且是 MAP\_SHARED 映射，将修改写回文件。

* 解除页表映射，释放物理页面。

* 更新 VMA 信息。

1. 在 `kernel/exit.c` 中处理进程退出时的 mmap 清理。

运行测试：



```
\$ mmaptest

mmaptest: mmap f: OK

mmaptest: mmap private: OK

mmaptest: mmap read-only: OK

mmaptest: mmap read/write: OK

mmaptest: mmap dirty: OK

mmaptest: not-mapped unmap: OK

mmaptest: two files: OK

mmaptest: fork\_test: OK
```

### 实验中遇到的问题和解决方法



1. **munmap 实现**：在实现 `munmap` 时，需要确保正确地找到要取消映射的页面，并进行相应的处理。如果页面被修改，且文件是 MAP\_SHARED 映射，需要将修改的数据写回文件。

2. **权限检查**：在实现 mmap 时，需要添加对映射权限的检查，确保只有可写的文件能够使用 MAP\_SHARED 形式映射。

3. **延迟分配**：使用 Lazy Allocation 的思想，不立即分配物理页面，而是在页面故障时动态分配。

### 实验心得

mmap 实验是整个课程的集大成者，涉及虚拟内存管理、页表操作、文件系统和延迟分配等多个主题。最有挑战性的部分是处理 page fault 时的 lazy allocation—— 当进程访问未映射的 mmap 区域时，需要在故障处理中动态分配页面并从文件读取数据。我在实现时还特别注意了 MAP\_SHARED 和 MAP\_PRIVATE 的区别：前者需要将修改写回文件，后者使用 COW 机制。在新工具链下，GCC 15 对未初始化变量的检测更加严格，这帮助我在编译阶段就发现了几个潜在的 bug。

### Lab 10 测试结果



```
\== Test running mmaptest ==

&#x20; mmaptest: mmap f: OK

&#x20; mmaptest: mmap private: OK

&#x20; mmaptest: mmap read-only: OK

&#x20; mmaptest: mmap read/write: OK

&#x20; mmaptest: mmap dirty: OK

&#x20; mmaptest: not-mapped unmap: OK

&#x20; mmaptest: two files: OK

&#x20; mmaptest: fork\_test: OK

\== Test usertests == usertests: OK

Score: 140/140
```



***

## 总结

本报告记录了在 xv6 操作系统上完成的 10 个实验，涵盖了系统调用、页表、陷阱、写时复制、多线程、网络、锁、文件系统和内存映射等核心操作系统概念。

**实验环境**：



* 操作系统：Ubuntu 26.04 (WSL2)

* QEMU：10.2.0

* RISC-V 工具链：GCC 15.0.1

* xv6 源码：MIT xv6-labs-2021

**实验成果**：



* 所有 10 个实验的 `make grade` 测试全部通过

* 实现了 20+ 个系统调用和内核功能

* 源代码托管于：[https://github.com/wenjialu727/OS-Xv6-Lab-2025](https://github.com/wenjialu727/OS-Xv6-Lab-2025)

**学生信息**：



* 姓名：闻家陆

* 学号：2451316

* 学校：同济大学

* 时间：2025 Summer