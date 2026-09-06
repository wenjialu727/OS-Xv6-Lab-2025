# Xv6 实验答辩操作速查手册

> 学生：闻家陆 2451316 | 2025 Summer | Tongji University
> 仓库：
>
> [https://github.com/wenjialu727/OS-Xv6-Lab-2025](https://github.com/wenjialu727/OS-Xv6-Lab-2025)



***

## 通用操作

### 启动 xv6



```
cd \~/xv6-labs-2021

git checkout <分支名>

make qemu
```

### 退出 xv6

按 `Ctrl+A`，松开后按 `x`

### 完整测试（退出 xv6 后）



```
make grade
```



***

## Lab 1: util（实用程序）

**分支**：`util`

### 演示命令



```
\$ sleep 10          # 停顿10秒

\$ pingpong          # 输出: 5: received ping / 4: received pong

\$ primes            # 输出素数 2,3,5,...,31

\$ find . README     # 输出: ./README

\$ sh < xargstest.sh # 输出三个 hello
```

### 核心讲解



* **sleep**：调用 `sleep()` 系统调用，理解用户态→内核态切换

* **pingpong**：父子进程通过两个管道双向通信

* **primes**：管道素数筛，每个进程过滤一个素数的倍数，体现 Unix 哲学

* **find**：递归遍历目录，理解目录也是文件（inode + 文件名对）

* **xargs**：从 stdin 读取参数，拼接后 `exec()` 执行

### make grade



```
Score: 100/100
```



***

## Lab 2: syscall（系统调用）

**分支**：`syscall`

### 演示命令



```
\$ trace 32 grep hello README        # 只跟踪 read（掩码32=1<<5）

\$ trace 2147483647 grep hello README # 跟踪所有系统调用

\$ sysinfotest                       # 输出: sysinfotest: OK
```

### 预期输出



```
\# trace 32:

3: read -> 1023

3: read -> 968

3: read -> 235

3: read -> 0

\# trace all:

4: trace -> 0

4: exec -> 3

4: open -> 3

4: read -> 1023

...

4: close -> 0
```

### 核心讲解



* **trace**：在 `proc` 结构体加 `trace_mask`，`fork()` 时继承，`syscall()` 入口检查掩码并打印

* **sysinfo**：遍历 `kmem.freelist` 统计空闲内存，遍历 `proc[]` 统计进程数，用 `copyout()` 拷贝到用户态

### make grade



```
Score: 35/35
```



***

## Lab 3: pgtbl（页表）

**分支**：`pgtbl`

### 演示命令



```
\$ pgtbltest
```

### 预期输出



```
ugetpid\_test starting

ugetpid\_test: OK

pgaccess\_test starting

pgaccess\_test: OK

pgtbltest: all tests succeeded
```

> vmprint 在启动时自动打印 init 进程的三级页表

### 核心讲解



* **ugetpid**：在 `USYSCALL` 页面映射 `struct usyscall`（含 pid），用户态直接读取，无需系统调用

* **vmprint**：递归打印三级页表（L2→L1→L0），`..` 表示层级缩进

* **pgaccess**：检查 PTE 的 `PTE_A`（访问位），返回位图，检测后清除 A 位

### make grade



```
Score: 46/46
```



***

## Lab 4: traps（中断与陷阱）

**分支**：`traps`

### 演示命令



```
\$ bttest        # 栈回溯

\$ alarmtest     # 定时信号
```

### 预期输出



```
\# bttest:

backtrace:

0x00000000800030c0

0x0000000080002f0c

0x0000000080002a0c

\# alarmtest:

test0 start

...alarm!

test0 passed

test1 start

...alarm! (10次)

test1 passed

test2 start

...alarm!

test2 passed
```

### 核心讲解



* **RISC-V assembly**：a0-a7 传参，a0-a1 返回值，t0-t6 调用者保存，s0-s11 被调用者保存

* **backtrace**：通过 `s0`（帧指针）链表回溯，`-8(s0)` 是返回地址，`-16(s0)` 是上一级 s0

* **alarm**：定时器中断（`which_dev==2`）时计数，达到间隔后修改 `epc` 跳转到 handler，`sigreturn()` 恢复寄存器

### make grade



```
Score: 85/85
```



***

## Lab 5: cow（写时复制）

**分支**：`cow`

### 演示命令



```
\$ cowtest
```

### 预期输出



```
simple: ok

simple: ok

three: ok

three: ok

three: ok

file: ok

ALL COW TESTS PASSED
```

### 核心讲解



* **核心思想**：`fork()` 时不复制页面，父子共享物理页面（只读），写入时触发 page fault 才复制

* **引用计数**：`kalloc.c` 中维护 `refcount[]`，`kfree()` 减到 0 才真正释放

* **page fault 处理**：`scause==15`（store fault）时，分配新页、复制内容、更新页表为可写、原页引用计数减 1

* **PTE\_COW**：用保留位标记 COW 页面，区分普通只读页

### make grade



```
Score: 110/110
```



***

## Lab 6: thread（多线程）

**分支**：`thread`

### 演示命令

**xv6 内**：



```
\$ uthread    # 用户级线程切换
```

**退出 xv6，WSL 终端**：



```
make ph && ./ph 1 && ./ph 2    # 哈希表细粒度锁性能对比

make barrier && ./barrier 2    # 条件变量屏障
```

### 预期输出



```
\# uthread: 三个线程交替打印 0-99

thread\_c 0

thread\_a 0

thread\_b 0

...

thread\_c: exit after 100

\# ph 1 vs ph 2:

单线程: 20455 puts/s

双线程: 44347 puts/s (加速2.17倍)

\# barrier:

OK; passed
```

### 核心讲解



* **uthread**：用户级线程，每个线程有独立栈，`thread_switch` 汇编保存 / 恢复 s0-s11、ra、sp

* **ph**：全局锁→每桶一把锁（`NBUCKET` 把锁），减少锁竞争，双线程性能提升 2 倍以上

* **barrier**：`pthread_cond_wait()` 自动释放 mutex 并等待，`pthread_cond_broadcast()` 唤醒所有线程，用 while 循环防虚假唤醒

### make grade



```
Score: 60/60
```



***

## Lab 7: net（网络）

**分支**：`net`

### 演示命令（需要两个终端）

**终端 A（xv6）**：



```
make qemu

\$ nettests
```

**终端 B（WSL）**：



```
cd \~/xv6-labs-2021

make server
```

### 预期输出



```
testing ping: OK

testing single-process pings: OK

testing multi-process pings: OK

testing DNS

DNS arecord for pdos.csail.mit.edu. is 128.52.129.126

DNS OK

all tests passed.
```

### 核心讲解



* **e1000 驱动**：TX/RX 描述符环（DMA），发送时写描述符，接收时网卡触发中断

* **发送路径**：应用→套接字→协议栈→`e1000_transmit()`→TX 描述符→网卡 DMA→网络

* **接收路径**：网络→网卡 DMA→RX 描述符→中断→`e1000_intr()`→协议栈→套接字→应用

* **协议栈**：Ethernet→ARP→IPv4→UDP/TCP→DNS

### 注意

`make grade` 可能因 QEMU 网络端口超时而失败，属环境问题，手动运行 `nettests` 全部通过即可。



***

## Lab 8: lock（锁优化）

**分支**：`lock`

### 演示命令



```
\$ kalloctest

\$ bcachetest
```

### 预期输出



```
\# kalloctest:

test1 OK

test2 OK (free pages: 32487/32768)

\# bcachetest:

test0: OK

test1: OK
```

### 核心讲解（答辩亮点：锁竞争数据）



* **per-CPU 内存分配器**：每个 CPU 独立 freelist，空闲时从其他 CPU 批量借用页面，消除全局 `kmem.lock` 竞争

* **分桶缓冲区缓存**：`blockno % NBUCKET` 映射到不同桶，每桶一把锁，不同桶互不阻塞

* **优化效果**：`bcache.bucket` 锁的 `test-and-set` 次数为 **0**（几乎无自旋等待），对比 `virtio_disk` 锁有 95 万次自旋

### make grade



```
Score: 140/140
```



***

## Lab 9: fs（文件系统）

**分支**：`fs`

### 演示命令



```
\$ bigfile        # 约需3-4分钟

\$ symlinktest
```

### 预期输出



```
\# bigfile:

................................................................................

wrote 65803 blocks

bigfile done; ok

\# symlinktest:

test symlinks: ok

test concurrent symlinks: ok
```

### 核心讲解



* **大文件**：11 直接块 + 1 一级间接块 + 1 二级间接块 = 11 + 256 + 256×256 = **65803 块**（原 268 块）

* **二级间接块**：inode.addrs \[11]→一级间接块→二级间接块→数据块，两级指针

* **符号链接**：`T_SYMLINK` 类型文件，内容存目标路径；`open()` 时递归解析，限制深度防循环；`O_NOFOLLOW` 打开链接本身

### make grade



```
Score: 100/100
```

> bigfile 在 QEMU 软件模拟下约 3-4 分钟，属正常现象。



***

## Lab 10: mmap（内存映射）

**分支**：`mmap`

### 演示命令



```
\$ mmaptest
```

### 预期输出



```
mmap\_test starting

test mmap f: OK

test mmap private: OK

test mmap read-only: OK

test mmap read/write: OK

test mmap dirty: OK

test not-mapped unmap: OK

test mmap two files: OK

mmap\_test: ALL OK

fork\_test starting

fork\_test OK

mmaptest: all tests succeeded
```

### 核心讲解



* **VMA 管理**：每个进程维护 `vmas[NVMA]` 数组，记录地址、长度、权限、标志、文件、偏移

* **延迟分配**：`mmap()` 不分配物理页，访问时触发 page fault，分配页面并从文件读取

* **MAP\_SHARED vs MAP\_PRIVATE**：SHARED 修改写回文件；PRIVATE 用 COW，修改不影响原文件

* **munmap**：解除映射，SHARED 映射的脏页写回文件，释放物理页

* **fork 集成**：子进程继承 VMA，共享物理页面（COW）

### make grade



```
Score: 140/140
```



***

## 答辩技巧

### 被问到某个实验时



1. 先 `git checkout <分支> && make qemu` 启动

2. 运行对应测试程序展示结果

3. 结合代码讲解核心实现（说文件名和关键函数）

4. 退出后 `make grade` 展示分数

### 重点准备的实验（最可能被问）



| 优先级 | 实验          | 原因                                |
| --- | ----------- | --------------------------------- |
| ⭐⭐⭐ | Lab 5 cow   | 写时复制是经典考点，涉及 page fault、引用计数      |
| ⭐⭐⭐ | Lab 8 lock  | 锁优化有直观数据对比（test-and-set 从 95 万→0） |
| ⭐⭐⭐ | Lab 10 mmap | 集大成者，VMA + 延迟分配 + COW             |
| ⭐⭐  | Lab 4 traps | backtrace 涉及栈帧结构，alarm 涉及中断处理     |
| ⭐⭐  | Lab 3 pgtbl | 三级页表是操作系统核心概念                     |
| ⭐   | Lab 1 util  | primes 管道素数筛，体现 Unix 哲学           |

### 常见问题回答要点



* **为什么用 COW？** → 减少 fork 开销，避免 exec 前的无意义复制

* **per-CPU 分配器有什么问题？** → 内存碎片化，某个 CPU 可能内存不足，需要页面借用机制

* **mmap 和普通 read/write 有什么区别？** → mmap 直接映射文件到内存，减少数据拷贝，利用 OS 的页缓存和预读

* **符号链接和硬链接的区别？** → 硬链接共享 inode，符号链接是独立文件存目标路径