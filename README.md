# Xv6 操作系统实验

**闻家陆 2451316**
Tongji University, 2025 Summer

仓库地址：https://github.com/wenjialu727/OS-Xv6-Lab-2025

## 关于

本项目是 MIT 6.S081 课程的 xv6 操作系统实验，在 RISC-V 架构下完成了 10 个实验，涵盖操作系统的核心机制：从用户态实用程序到系统调用、内存管理、中断处理、文件系统和网络驱动。

实验环境：WSL2 Ubuntu + QEMU + riscv64 GCC

## 仓库文件

| 文件 | 说明 |
|---|---|
| `OS_实验报告.md` | 完整实验报告（Markdown 版），包含 10 个实验的目的、步骤、问题、心得和测试结果 |
| `OS_实验报告_导航版.docx` | 实验报告 Word 版，带封面、目录域和标题导航层级 |
| `Xv6答辩操作速查手册.md` | 答辩演示操作手册，包含 10 个实验的启动命令、预期输出、讲解要点和答辩技巧 |

## 实验分支

每个实验对应一个独立分支，切换分支即可查看对应实验的完整代码：

| 分支 | 实验内容 | 满分 |
|---|---|---|
| `util` | 用户态实用程序（sleep/pingpong/primes/find/xargs） | 100 |
| `syscall` | 系统调用（trace/sysinfo） | 35 |
| `pgtbl` | 页表（ugetpid/vmprint/pgaccess） | 46 |
| `traps` | 中断与陷阱（backtrace/sigalarm） | 85 |
| `cow` | 写时复制 | 110 |
| `thread` | 多线程（用户级线程/细粒度锁/屏障） | 60 |
| `net` | 网络驱动（e1000） | 100 |
| `lock` | 锁优化（per-CPU分配器/分桶缓存） | 70 |
| `fs` | 文件系统（大文件/符号链接） | 100 |
| `mmap` | 内存映射文件 | 140 |

## 快速开始

```bash
# 克隆仓库
git clone https://github.com/wenjialu727/OS-Xv6-Lab-2025.git
cd OS-Xv6-Lab-2025

# 切换到对应实验分支
git checkout util

# 编译并运行 xv6
make qemu

# 运行测试（退出 xv6 后）
make grade
```

**退出 QEMU**：按 `Ctrl+A`，松开后按 `x`

## 实验报告摘要

10 个实验按照操作系统的核心层次递进：
- **Lab 1-2**：用户态与系统调用接口
- **Lab 3-5**：内存管理（页表、中断、写时复制）
- **Lab 6**：多线程与并发
- **Lab 7**：网络驱动与协议栈
- **Lab 8**：锁与性能优化
- **Lab 9-10**：文件系统与内存映射

详细内容请阅读 `OS_实验报告.md`。
