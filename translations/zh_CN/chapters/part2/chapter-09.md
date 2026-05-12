---
title: "设备的读取和写入"
description: "d_read 和 d_write 如何通过 uio 和 uiomove 安全地在用户空间和内核之间移动字节。"
partNumber: 2
partName: "构建你的第一个驱动"
chapter: 9
lastUpdated: "2026-04-17"
status: "complete"
author: "Edson Brandi"
reviewer: "TBD"
translator: "AI辅助翻译为简体中文"
language: "zh-CN"
estimatedReadTime: 195
---

# 设备的读取和写入

## 读者指南与学习目标

第七章教会了你让驱动站立起来。 第八章教会了你该驱动如何通过 `/dev` 与用户态会面。 你在上一章结束时得到的驱动作为 Newbus 设备挂载、创建 `/dev/myfirst/0`、在 `/dev/myfirst` 携带别名、分配每次打开的状态、干净地记录日志，并且拆离时不泄漏。 这些部分每一个都很重要，但没有一个实际移动了字节。

本章是字节开始移动的地方。

当用户程序在你的设备节点之一上调用 `read(2)` 或 `write(2)` 时，内核必须在用户地址空间和驱动内存之间传递真实数据。 那次传输不是简单的 `memcpy`。 它跨越了一个信任边界。 用户传递的缓冲区指针可能无效。 缓冲区可能不是全部驻留的。 长度可能是零，或者巨大，或者是分散-聚集列表的一部分。 用户可能在 jail 中，可能有挂起的信号，可能用 `O_NONBLOCK` 读取，可能已通过管道重定向结果。 你的驱动不需要孤立地理解每一个这些情况，但它确实需要与解决所有情况的单一内核抽象协作。 那个抽象是 `struct uio`，使用它的主要工具是 `uiomove(9)`。

本章是我们最终实现第七章留作桩函数的 `d_read` 和 `d_write` 入口点的地方。 在此过程中，我们将仔细查看内核如何描述 I/O 请求、为什么本书自第五章以来一直说"不要直接触及用户指针"，以及如何塑造驱动使部分传输、未对齐缓冲区、被信号中断的读取和短写入都表现出经典 UNIX 文件的方式。

### 为什么本章值得单独设立

写一个只说"调用 `uiomove`"然后继续的短章节会很诱人。 那会给读者留下一个通过最简单测试然后在二十种微妙方式上失败的驱动。 本章之所以有这个长度是因为 I/O 是初学者驱动最常出错的地方，他们出错的地方不是代码看起来有风险的地方。 错误通常在返回值中、在 `uio_resid` 的处理中、在零长度传输的处理中、在驱动从 `msleep(9)` 醒来因为进程被杀死时发生什么中、在部分读取应该向哪个方向排空中。

一个在这些细节上出错的驱动干净地编译、通过单次 `cat /dev/myfirst`，然后在真正程序开始向其推送字节时产生损坏的数据。 那是消耗数天的错误。 本章的目标是在源头阻止那类错误。

### 第八章结束时驱动的状态

在第八章结束时，你的 `myfirst` 驱动具有以下形状。值得检查一下，因为第九章直接在其之上构建：

- 单个 Newbus 子设备，在 `device_identify` 中创建，注册在 `nexus0` 下。
- 由 Newbus 分配并在 `attach` 中初始化的 `struct myfirst_softc`。
- 以设备命名的互斥锁，用于保护 softc 计数器。
- `dev.myfirst.0.stats` 下的 sysctl 树，暴露 `attach_ticks`、`open_count`、`active_fhs` 和 `bytes_read`。
- `/dev/myfirst/0` 处的主 cdev，所有权为 `root:operator`，模式为 `0660`。
- `/dev/myfirst` 处指向主设备的别名 cdev。
- 每次 `open(2)` 分配的 `struct myfirst_fh`，通过 `devfs_set_cdevpriv(9)` 注册，由每个描述符恰好触发一次的析构函数释放。
- 桩函数 `d_read` 和 `d_write` 处理程序检索每次打开的状态，可选地查看它，并立即返回: `d_read` returns zero bytes (EOF), `d_write` claims to have consumed every byte by setting `uio_resid = 0`.

第九章将这些桩函数变成真正的。 驱动的外部形状变化不大。 新读者仍然应该看到 `/dev/myfirst/0`，仍然看到别名，仍然看到 sysctl。 变化的是 `cat /dev/myfirst/0` 现在会产生输出，`echo hello > /dev/myfirst/0` 现在会将文本存储到驱动内存中，第二次 `cat` 会读回第一次写入的确切内容。 到本章结束时，你的驱动将是一个小型的、纪律严明的内存缓冲区，你可以向其中推送字节并从中拉取字节。 它还不会是带有阻塞读取的循环缓冲区；那是第十章的工作。 它将是一个正确移动字节的驱动。

### 你将学到什么

完成本章后，你将能够：

- 解释 `read(2)` 和 `write(2)` 如何从用户空间通过 devfs 流入你的 `cdevsw` 处理程序。
- 读写 `struct uio` 的字段而无需记忆它们。
- 使用 `uiomove(9)` 在任一方向的内核缓冲区和调用者缓冲区之间传输字节。
- 当内核缓冲区有固定大小且你想要自动偏移记账时，使用 `uiomove_frombuf(9)`。
- 决定何时使用 `copyin(9)` 或 `copyout(9)` 而不是 `uiomove(9)`。
- 为短传输、空传输、文件结束和被中断的读取返回正确的字节计数。
- 为驱动读取或写入可能采用的每个错误路径选择适当的 errno 值。
- 设计一个内部缓冲区，驱动从 `d_write` 填充并从 `d_read` 排空。
- 识别和修复最常见的 `d_read` 和 `d_write` bug。
- 从基本系统工具（`cat`、`echo`、`dd`、`od`、`hexdump`）和一个小型 C 程序测试驱动。

### 你将构建什么

你将从第八章结束时的 `myfirst` 驱动通过三个增量阶段。

1. **阶段 1，静态消息读取器。** `d_read` 返回固定内核空间字符串的内容。每次打开从偏移零开始，读取完消息。这是设备读取的"hello world"，但带有正确的偏移处理。
2. **阶段 2，写一次/读多次缓冲区。** 驱动拥有一个固定大小的内核缓冲区。`d_write` 向其中追加。`d_read` 返回迄今为止写入的内容，从一个记住每个读取者已消耗多远的每次描述符偏移开始。两个并发读取者仍然独立看到自己的进度。
3. **阶段 3，小型回显驱动。** 同一个缓冲区，现在用作先进先出存储。每次 `write(2)` 向尾部追加字节。每次 `read(2)` 从头部移除字节。双进程测试脚本在一个终端写入，在另一个终端读取回显的数据。这是到第十章的交接点，我们将在那里围绕真正的循环缓冲区重建相同的驱动，添加部分 I/O 和非阻塞支持，并连接 `poll(2)` 和 `kqueue(9)`。

所有三个阶段都能编译、加载并行为可预测。 你将从 `cat`、`echo` 和一个名为 `rw_myfirst.c` 的小型用户态程序测试每一个，该程序测试 `cat` 自己无法到达的边缘情况。

### 本章不涵盖的内容

有几个涉及 `read` 和 `write` 的主题被刻意推迟：

- **循环缓冲区和环绕**：第十章实现真正的环形缓冲区。这里的阶段 3 使用简单的线性缓冲区，这样我们可以保持专注于 I/O 路径本身。
- **阻塞读取和 `poll(2)`**：第十章介绍基于 `msleep(9)` 的阻塞和 `d_poll` 处理程序。本章在驱动级别保持所有读取非阻塞；空缓冲区产生立即的零字节读取。
- **`ioctl(2)`**：第十五章构建 `d_ioctl`。我们只在读者需要理解为什么某些控制路径属于那里而不是 `write` 时才触及它。
- **硬件寄存器和 DMA**：第四部分处理总线资源、`bus_space(9)` 和 DMA。我们在本章读取和写入的内存是从 `M_DEVBUF` 用 `malloc(9)` 分配的普通内核堆。
- **负载下的并发正确性**：第三部分专门讨论竞态条件、锁定和验证。我们在竞态会损坏阶段 3 缓冲区的地方采取互斥锁保护措施，但更深入的讨论被推迟。

坚守这些界限是我们保持章节诚实的方式。一个漂移到 `ioctl`、DMA 和 `kqueue` 的初学者章节是一个什么都教不好的初学者章节。

### 预计时间投入

- **仅阅读**：大约一小时。
- **阅读加上输入三个阶段**：大约三小时，包括每个阶段几次加载/卸载循环。
- **阅读加上所有实验和挑战**：五到七小时，分两到三次会话。

在开始时给自己一个全新的实验启动。不要急。阶段故意很小，真正的价值来自每次更改后观看 `dmesg`、观看 `sysctl` 和从用户态探测设备。

### 前提条件

在开始本章之前，确认：

- 你有一个等效于 `examples/part-02/ch08-working-with-device-files/stage2-perhandle/` 下第八章阶段 2 源码的工作 `myfirst` 驱动。如果你还没有到达第八章结尾，在此暂停并回来。
- 你的实验机器运行带有匹配 `/usr/src` 的 FreeBSD 14.3。
- 你已经阅读过第四章关于指针、结构和内存布局的讨论，以及第五章关于内核空间习惯和安全性的讨论。
- You understand what a `struct cdev` is and how it is related to a `cdevsw`. Chapter 8 covered this in detail.

如果你对这些中的任何一个不确定，本章的其余部分会比它需要的更难。先重温相关部分。

### 如何从本章获得最大收益

三个习惯立刻见效。

首先，在第二个终端中保持打开 `/usr/src/sys/dev/null/null.c`。它是树中最短、最干净、最可读的 `d_read` 和 `d_write` 示例。本章介绍的每个理念都出现在 `null.c` 的五十行或更少中。真正的 FreeBSD 驱动就是教科书；本书是阅读指南。

其次，保持打开 `/usr/src/sys/sys/uio.h` 和 `/usr/src/sys/sys/_uio.h`。那里的声明简短且稳定。现在阅读它们一次，这样当章节提到 `uio_iov`、`uio_iovcnt`、`uio_offset` 和 `uio_resid` 时，你不必仅信任文字。

第三，在更改之间重建并在下次更改前从用户态确认行为。这是将编写驱动与编写关于驱动的文字区分开来的习惯。你将在每个检查点运行 `cat`、`echo`、`dd`、`stat`、`sysctl`、`dmesg` 和一个短 C 程序。不要跳过它们。本章正在教你识别的失败模式只有在运行代码时才会变得可见。

### 本章路线图

各节按顺序排列：

1. 完整 I/O 路径的可视化地图，从用户空间的 `read(2)` 到你处理程序内的 `uiomove(9)`。
2. UNIX 中 `read` 和 `write` 意味着什么的简短复习，以及对驱动编写者具体意味着什么。
3. `d_read` 的剖析：其签名、被要求做什么、被要求返回什么。
4. `d_write` 的剖析：`d_read` 的镜像，加上一些仅适用于写入方向的细节。
5. 不熟悉处理程序的阅读协议，然后是第二次真实驱动巡览（`mem(4)`）以展示不同的形状。
6. `ioflag` 参数：它从哪里来、哪些位重要、为什么第九章大多忽略它。
7. 仔细看看 `struct uio`——内核的 I/O 描述对象，逐字段，包括通过一次调用的同一 uio 的三个快照。
8. `uiomove(9)` 及其同伴——实际移动字节的函数。
9. `copyin(9)` 和 `copyout(9)`：何时使用它们，何时不用它们而使用 `uiomove`。加上关于结构化数据的警示案例研究。
10. 内部缓冲区：静态、动态和固定大小。如何选择一个、如何安全地拥有它、你应该识别的内核辅助函数。
11. 错误处理：对 I/O 重要的 errno 值、如何发出文件结束信号、以及如何思考部分传输。
12. 三阶段 `myfirst` 实现，包括驱动源码。
13. A step-by-step trace of `read(2)` from user space through the kernel down to your handler, plus a mirrored write trace.
14. 测试的实用工作流：`cat`、`echo`、`dd`、`truss`、`ktrace`，以及将它们变成开发节奏的纪律。
15. 可观测性：sysctl、dmesg 和 `vmstat -m`，带有驱动在轻负载下的具体快照。
16. 有符号、无符号和差一的危害——简短但高价值的章节。
17. 本章材料最可能产生的错误的故障排除说明，以及正确与有问题的处理程序模式的对比表。
18. 动手实验（七个）带你通过每个阶段并巩固可观测性工作流。
19. 挑战练习（八个）延伸模式。
20. 总结和通往第十章的桥梁。

如果你是第一次阅读本章，请线性阅读并在遇到实验时做。如果你复习材料以巩固，末尾的参考风格章节可以独立阅读。



## I/O 路径的可视化地图

在文字深入之前，一张图值得铭记。 下图是 `read(2)` 调用从用户程序到你的驱动再返回调用者的路径。 每个框是你在 `/usr/src/sys/` 下可以找到的真实内核代码片段。 每个箭头是真实的函数调用。 没有一个是比喻。

```text
                         user space
      +----------------------------------------------+
      |   user program                               |
      |                                              |
      |     n = read(fd, buf, 1024);                 |
      |            |                                 |
      |            v                                 |
      |     libc read() wrapper                      |
      |     (syscall trap instruction)               |
      +-------------|--------------------------------+
                    |
     ==============| kernel trust boundary |===============
                    |
                    v
      +----------------------------------------------+
      |  sys_read()                                   |
      |  /usr/src/sys/kern/sys_generic.c              |
      |  - lookup fd in file table                    |
      |  - fget(fd) -> struct file *                  |
      |  - build a uio around buf, count              |
      +-------------|--------------------------------+
                    |
                    v
      +----------------------------------------------+
      |  struct file ops -> vn_read                   |
      |  /usr/src/sys/kern/vfs_vnops.c                |
      +-------------|--------------------------------+
                    |
                    v
      +----------------------------------------------+
      |  devfs_read_f()                               |
      |  /usr/src/sys/fs/devfs/devfs_vnops.c          |
      |  - devfs_fp_check -> cdev + cdevsw            |
      |  - acquire thread-count ref                   |
      |  - compose ioflag from f_flag                 |
      |  - call cdevsw->d_read(dev, uio, ioflag)      |
      +-------------|--------------------------------+
                    |
                    v
      +----------------------------------------------+
      |  YOUR HANDLER (myfirst_read)                  |
      |  - devfs_get_cdevpriv(&fh)                    |
      |  - verify is_attached                         |
      |  - call uiomove(9) to transfer bytes          |
      |            |                                  |
      |            v                                  |
      |     +-----------------------------------+     |
      |     |  uiomove_faultflag()              |     |
      |     |  /usr/src/sys/kern/subr_uio.c     |     |
      |     |  - for each iovec entry           |     |
      |     |    copyout(kaddr, uaddr, n)  ===> |====|====> user's buf
      |     |    decrement uio_resid            |     |
      |     |    advance uio_offset             |     |
      |     +-----------------------------------+     |
      |  - return 0 or an errno                       |
      +-------------|--------------------------------+
                    |
                    v
      +----------------------------------------------+
      |  devfs_read_f continues                       |
      |  - release thread-count ref                   |
      |  - update atime if bytes moved                |
      +-------------|--------------------------------+
                    |
                    v
      +----------------------------------------------+
      |  sys_read finalises                           |
      |  - compute count = orig_resid - uio_resid     |
      |  - return to userland                         |
      +-------------|--------------------------------+
                    |
     ==============| kernel trust boundary |===============
                    |
                    v
      +----------------------------------------------+
      |   user program sees the return value         |
      |   in n                                        |
      +----------------------------------------------+
```

这张图中有几个特征值得确认，因为它们在整章中反复出现。

**信任边界恰好被跨越两次。** 一次在向下时（用户通过系统调用陷阱进入内核），一次在向上时（内核将控制权返回给用户空间）。 中间的一切都是仅限内核的执行。 你的处理程序完全在内核内运行，在内核栈上，用户的寄存器已被保存到一边。

**你的处理程序是驱动知识进入路径的唯一地方。** 它上面的一切是对树中每个字符设备完全相同工作的内核机制。 它下面的一切是 `uiomove` 和 `copyout`，也是内核机制。 你的处理程序是计算"这次读取应该产生什么字节？"这个答案的唯一函数。

**用户的缓冲区永远不会被你的驱动直接触及。** 它是由 `uiomove` 内部的 `copyout` 触及的。 你的驱动向 `uiomove` 传递一个内核指针，而 `uiomove` 是唯一代表你解引用用户指针的代码。 这就是作为代码绘制的信任边界形状：用户内存只能通过知道如何安全访问的唯一 API 访问。

**每一步在返回路上都有一个匹配的步骤。** devfs 获取的线程计数引用在你处理程序返回后被释放；检查 uio 的状态以计算字节计数；控制通过每一层展开并返回用户空间。 理解这种对称性是让引用计数感觉自然而非任意的。

打印这张图或在纸上画出来。 当你在书中稍后阅读不熟悉的驱动时，请回头参考它。 你将学习的每个 `d_read` 或 `d_write` 都恰好位于调用链中的这个点。 驱动之间的差异在于处理程序；处理程序周围的路径是恒定的。

对于 `d_write`，图是镜像的。 `devfs_write_f` 分派到 `cdevsw->d_write`，你的处理程序以另一个方向调用 `uiomove(9)`，`uiomove` 调用 `copyin` 而不是 `copyout`，内核展开回到 `write(2)`。 图中的每个箭头都有一个镜像；上面列出的每个属性也适用于写入。



## UNIX 中的设备：快速回顾

在开始写代码之前值得花十分钟复习。 第六章在概念层面介绍了 UNIX I/O 模型；第七章将其付诸实践；第八章使设备文件接口变得整洁。 这三个章节都有理由不深入讨论 `read(2)` 和 `write(2)` 本身的行为，因为这些章节中的驱动不承载真实数据。 现在我们承载了，一个紧凑的复习为接下来的一切搭好了舞台。

### 设备与文件的区别是什么？

从外部看，它们看起来完全相同。 两者都用 `open(2)` 打开。 两者都用 `read(2)` 读取和 `write(2)` 写入。 两者都用 `close(2)` 关闭。 在常规文件上工作的用户程序几乎总是在设备文件上工作而不需要修改源代码，因为用户空间 API 不区分它们。

从内部看，有真正的区别，驱动作者需要内化它们。

常规文件有后备存储，通常是由文件系统管理的磁盘上的字节。 内核决定何时预读、何时缓存、何时刷新。 数据有持久的身份；读取文件零字节位置的两个程序看到相同的字节。 在文件大小内寻址是廉价且无限制的。

设备文件在文件系统意义上没有后备存储。 当用户程序从中读取时，驱动决定产生什么字节。 当用户程序向其写入时，驱动决定如何处理它们。 数据的身份是你的驱动定义的。 从同一设备读取的两个程序不一定看到相同的字节；取决于驱动，它们可能看到相同的字节，可能看到单个流的不相交的两半，或者可能看到完全独立的流。 寻址可能有意义，或者无意义，或者被主动禁止。

对你的 `d_read` 和 `d_write` 处理程序的实际后果是 **驱动是此设备上 `read` 和 `write` 含义的权威定义**。 内核将向你传递一个 I/O 请求；它不会告诉你如何处理它。 UNIX 程序期望的约定——字节流、一致的返回值、诚实的错误代码、以零字节返回表示文件结束——是你的驱动必须有意识地遵守的约定。 内核不强制执行它们。

### UNIX 如何将设备视为数据流

"流"这个词值得明确界定，因为它出现在每一次 UNIX I/O 的讨论中，并且根据上下文至少有三种不同的含义。

对我们的目的来说，流是**按顺序传递的字节序列**。调用者和驱动都不知道总长度。任何一方都可以在任何时候停止。序列可能有自然终点（已被完全读取的文件），也可能无限延续（终端、网络套接字、传感器）。无论哪种情况，规则都是相同的：读取者请求一定数量的字节，写入者请求一定数量的字节被接受，内核报告实际移动了多少字节。

流除了数据传输本身之外没有副作用。如果你的驱动需要暴露控制接口——改变配置、重置状态或协商参数的方式——那个接口不应该在 `read` 和 `write` 中。控制接口是 `ioctl(2)`，在第25章涵盖。不要在数据流中走私控制命令。这会让你的驱动更难使用、更难测试、更难演进。

流在每次调用中是单向的。`read(2)` 将字节从驱动移动到用户。`write(2)` 将字节从用户移动到驱动。单个系统调用永远不会同时做两件事。如果你需要双向行为（例如请求-响应模式），你通过写入后跟读取来实现，并在驱动内部进行所需的任何协调。

### 顺序访问与随机访问

大多数驱动产生顺序流：字节按照到达的顺序输出，`lseek(2)` 要么没有实际效果，要么被拒绝。终端、串口、数据包捕获设备、日志流，所有这些都是顺序的。

少数驱动是随机访问的：调用者可以通过 `lseek(2)` 寻址任何字节，相同的偏移总是读取相同的数据。内存磁盘驱动、`/dev/mem` 和少数其他驱动符合这个模型。在大多数方面，它们看起来更像常规文件而不是设备。

驱动作者选择驱动在这个范围上的位置。你的 `myfirst` 驱动在本章大部分内容中位于顺序端，但有一个细微差别：每个打开的描述符都有自己的读取偏移，因此两个并发读取的进程从流中的不同点开始。这是大多数小型字符设备使用的折衷方案。它给每个读取者提供一致的已消费内容视图，而不需要驱动承担真正的随机访问契约。

这个选择在代码中体现在两个地方：

- **你的 `d_read` 更新 `uio->uio_offset`**（由 `uiomove(9)` 为你完成）当且仅当偏移对你有意义时。对于偏移没有意义的真正顺序设备，该值被忽略。
- **你的驱动要么遵守要么忽略传入的 `uio->uio_offset`** 在每次读取开始时。顺序驱动忽略它并从当前位置提供服务。随机访问驱动将其视为线性空间中的地址。

对于三阶段的 `myfirst`，我们将把 `uio->uio_offset` 视为该描述符在流中位置的每次调用快照，并相应更新我们的内部计数器。

### read() 和 write() 在设备驱动中的角色

在内核中，设备文件上的 `read(2)` 和 `write(2)` 最终调用你的 `cdevsw->d_read` 和 `cdevsw->d_write` 函数指针。系统调用和你的函数之间的一切都是 devfs 和 VFS 机制；你的函数返回后的一切都是内核将结果传回用户态。你的处理程序是计算"这次调用会发生什么？"这个驱动特定答案的唯一地方。

处理程序的工作在抽象上并不复杂：

1. 查看请求。要求你提供或交给你多少字节？
2. 移动字节。使用 `uiomove(9)` 在内核缓冲区和用户缓冲区之间传输数据。
3. 返回结果。成功返回零（`uio_resid` 相应更新），失败返回 errno 值。

使处理程序变得不简单的是，步骤 2 是用户内存和内核内存之间的信任边界，与用户内存的每次交互都必须能够安全地应对行为不当或恶意的用户程序。这就是 `uiomove(9)` 存在的原因。你不需要编写安全逻辑；内核来做，只要你通过正确的 API 请求。

### 字符设备与块设备再探

第八章指出，FreeBSD 多年来一直没有向用户态提供块特殊设备节点。存储驱动存在于 GEOM 中，并以字符设备的形式发布。就本章而言，字符设备是我们唯一关心的形式。

实际后果是，本章的所有内容都适用于你在第2到第4部分可能编写的每个驱动。`d_read` 和 `d_write` 是入口点。`struct uio` 是载体。`uiomove(9)` 是移动器。当我们到达第6部分并查看 GEOM 支持的存储驱动时，它们的数据路径看起来会有所不同，但仍然由我们正在学习的相同原语构建。

### 练习：分类你的 FreeBSD 系统上的真实设备

在本章其余部分深入代码之前，在你的实验机上花五分钟。打开一个终端并浏览 `/dev`：

```sh
% ls /dev
% ls -l /dev/null /dev/zero /dev/random /dev/urandom /dev/console
```

对于你看到的每个节点，问自己三个问题：

1. 它是顺序访问还是随机访问？
2. 如果我对它执行 `cat`，它应该产生任何字节吗？什么字节？
3. 如果我执行 `echo something >` 向其写入，应该看到什么变化？在哪里？

试几个：

```sh
% head -c 16 /dev/zero | od -An -tx1
% head -c 16 /dev/random | od -An -tx1
% echo "hello" > /dev/null
% echo $?
```

注意 `/dev/zero` 是取之不尽的，`/dev/random` 传递不可预测的字节，`/dev/null` 静默吞噬写入并返回成功，这三个设备在有用意义上都不可寻址。这些行为不是偶然的。它们是这些驱动的 `d_read` 和 `d_write` 处理程序，做的正是我们即将学习的内容。

如果你打开 `/usr/src/sys/dev/null/null.c` 并查看 `null_write`，你会看到一行实现：`uio->uio_resid = 0; return 0;`。那是一个功能完整的 `write` 处理程序。驱动宣布"我消耗了所有字节；没有错误"。那是 FreeBSD 中最小的有意义的写入实现，到本章结束时，你将能够毫不犹豫地编写它以及许多更大的处理程序。



## d_read() 的剖析

你的驱动的读取路径从 devfs 将调用分派到 `cdevsw->d_read` 时开始。签名是固定的，在 `/usr/src/sys/sys/conf.h` 中声明：

```c
typedef int d_read_t(struct cdev *dev, struct uio *uio, int ioflag);
```

FreeBSD 源码树中的每个 `d_read` 函数都恰好具有这种形状。三个参数是调用的完整描述：

- `dev` 是表示被打开设备节点的 `struct cdev *`。在处理每个实例多个 cdev 的驱动中，它告诉你调用发生在哪个 cdev 上。在 `myfirst` 中，主设备及其别名通过同一处理程序分派，两者都通过 `dev->si_drv1` 解析到相同的底层 softc。
- `uio` 是描述 I/O 请求的 `struct uio *`：用户提供了什么缓冲区、它们有多大、读取应该从流中的哪个位置开始、还有多少字节需要移动。我们将在下一节详细分析它。
- `ioflag` 是在 `/usr/src/sys/sys/vnode.h` 中定义的标志位掩码。对非阻塞 I/O 重要的是 `IO_NDELAY`，当用户使用 `O_NONBLOCK` 打开描述符（或稍后通过 `fcntl(F_SETFL, ...)` 传递 `O_NONBLOCK`）时设置。还有一些与基于 vnode 的文件系统 I/O 相关的标志，但对于字符设备驱动，你通常只需检查 `IO_NDELAY`。

返回值是 errno 风格的整数：成功时为零，失败时为正 errno 代码。它**不是**字节计数。内核通过查看调用期间 `uio_resid` 减少了多少来计算字节计数，并将该值作为 `read(2)` 的返回值报告给用户空间。这种反转是本章需要内化的两三件最重要的事情之一。`d_read` 返回错误代码；传输的字节数隐含在 uio 中。

### d_read 被要求做什么

简化为一句话，任务是：**从设备产生最多 `uio->uio_resid` 个字节，通过 `uiomove(9)` 将它们传递到 `uio` 描述的任何缓冲区中，并返回零**。

这句话有几个推论值得明确说明。

函数可以产生比请求更少的字节。短读取是合法且预期的。一个请求 4096 字节但只收到 17 字节的用户程序不会将其视为错误；它将其视为"驱动现在只有 17 字节可以提供"。这个数字对调用者可见，因为 `uiomove(9)` 在移动字节时将 `uio_resid` 减少了 17。

函数可以产生零字节。零字节读取是 UNIX 报告文件结束的方式。如果你的驱动没有更多数据可以提供，也不会再有更多数据，返回零并保持 `uio_resid` 不变。调用者看到一个零字节的 `read(2)`，就知道流结束了。

函数不能产生比请求更多的字节。`uiomove(9)` 为你强制执行这一点；它在单次调用中不会移动超过 `MIN(uio_resid, n)` 字节。如果你在单个 `d_read` 内重复调用 `uiomove`，确保你的循环也尊重 `uio_resid`。

函数在失败时必须返回 errno。成功时返回值为零。非零返回值被内核解释为错误；内核通过 `errno` 将它们传播到用户空间。常见值有 `ENXIO`、`EFAULT`、`EIO`、`EINTR` 和 `EAGAIN`。我们将在错误处理部分逐一讲解。

函数可以睡眠。`d_read` 在进程上下文（调用者的上下文）中运行，因此 `msleep(9)` 及相关函数是合法的。这是驱动实现等待数据的阻塞读取的方式。我们在本章不会使用 `msleep(9)`（第十章正式介绍它），但值得知道你有阻塞的权利。

### d_read **不被**要求做什么

处理程序明确不负责的事项简表，因为内核或 devfs 会处理它们：

- **定位用户内存**。`uio` 已经描述了目标缓冲区。你的处理程序不需要查找页表或验证地址。
- **检查权限**。用户的凭据已由 `open(2)` 验证；当 `d_read` 运行时，调用者已被允许从此描述符读取。
- **为调用者计数字节**。内核从 `uio_resid` 计算字节计数。你永远不返回字节计数。
- **强制执行全局大小限制**。内核已将 `uio_resid` 限制为系统能够处理的值。

每一个在某些时候都是一种诱惑。抵制它们所有。每一个都是处理程序可能引入微妙错误的地方，而正确使用 `uiomove` 可以从根本上避免这些错误。

### 第一个真正的 d_read

这是 FreeBSD 源码树中最小的有用 `d_read`。它是 `/usr/src/sys/dev/null/null.c` 中的 `zero_read` 函数，也是 `/dev/zero` 产生无限零字节流的方式：

```c
static int
zero_read(struct cdev *dev __unused, struct uio *uio, int flags __unused)
{
        void *zbuf;
        ssize_t len;
        int error = 0;

        zbuf = __DECONST(void *, zero_region);
        while (uio->uio_resid > 0 && error == 0) {
                len = uio->uio_resid;
                if (len > ZERO_REGION_SIZE)
                        len = ZERO_REGION_SIZE;
                error = uiomove(zbuf, len, uio);
        }
        return (error);
}
```

暂停一下思考。循环体有三行。终止条件有两个：要么 `uio_resid` 达到零（我们传输了调用者请求的所有内容），要么 `uiomove` 返回错误。每次迭代移动零填充区域中请求有空间容纳的那么多。函数返回最后的错误代码，如果传输干净完成则为零。

循环是必要的，因为零区域是有限的：单次 `uiomove` 调用不能从中移动任意多字节，所以循环分块传输。对于源数据适合放入适当大小的单个内核缓冲区的驱动，循环退化为单次调用。`myfirst` 的阶段 1 将恰好是这种形状。

还要注意函数**没有**做什么。它不看 `uio_offset`。它不关心读取在某个想象流中的哪个位置开始；每次读取 `/dev/zero` 都产生零字节。它不检查 cdev。它不检查标志。它只做一件事，并使用一个 API 完成那件事。

这就是模型。你的 `d_read` 通常看起来像是那个循环的某种变体。

### 变体：uiomove_frombuf

当你的源数据是固定大小的内核缓冲区，并且你希望驱动的行为像由该缓冲区支持的文件时，辅助函数 `uiomove_frombuf(9)` 为你做偏移算术。

其声明，来自 `/usr/src/sys/sys/uio.h`：

```c
int uiomove_frombuf(void *buf, int buflen, struct uio *uio);
```

其实现，来自 `/usr/src/sys/kern/subr_uio.c`，足够短可以复现：

```c
int
uiomove_frombuf(void *buf, int buflen, struct uio *uio)
{
        size_t offset, n;

        if (uio->uio_offset < 0 || uio->uio_resid < 0 ||
            (offset = uio->uio_offset) != uio->uio_offset)
                return (EINVAL);
        if (buflen <= 0 || offset >= buflen)
                return (0);
        if ((n = buflen - offset) > IOSIZE_MAX)
                return (EINVAL);
        return (uiomove((char *)buf + offset, n, uio));
}
```

仔细阅读，因为行为是精确的。函数接受一个指向大小为 `buflen` 的内核缓冲区的指针 `buf`，参考 `uio->uio_offset`，然后：

- 如果偏移为负或其他无意义值，返回 `EINVAL`。
- 如果偏移超出缓冲区末尾，返回零而不移动任何字节。这是文件结束：调用者将看到零字节读取。
- 否则，以指向 `buf` 中当前偏移处的指针和等于缓冲区剩余尾部长度的长度调用 `uiomove(9)`。

函数不循环；`uiomove` 将移动 `uio_resid` 有空间容纳的那么多字节，并相应减少 `uio_resid`。驱动事后不需要触及 `uio_offset`，因为 `uiomove` 会做。

如果你的驱动将固定缓冲区暴露为可读文件，一行 `d_read` 就足够了：

```c
static int
myfirst_read(struct cdev *dev, struct uio *uio, int ioflag)
{
        struct myfirst_softc *sc = dev->si_drv1;
        return (uiomove_frombuf(sc->buf, sc->buflen, uio));
}
```

本章的阶段 1 正好使用这种模式，只是做了一个小调整来跟踪每次描述符的读取偏移，这样两个并发读取者能看到各自的进度。

### 实际使用中的签名：myfirst_read 阶段 1

Here is what our Stage 1 `d_read` will look like. Do not type it in yet; we will walk through the full source in the implementation section. Seeing it here and now is mostly to anchor the discussion.

Before you read the code, pause on one detail that will recur in almost every handler for the rest of this chapter. The first four lines of any per-open-aware handler follow a fixed **boilerplate pattern**:

```c
struct myfirst_fh *fh;
int error;

error = devfs_get_cdevpriv((void **)&fh);
if (error != 0)
        return (error);
```

This pattern retrieves the per-descriptor `fh` that `d_open` registered through `devfs_set_cdevpriv(9)`, and it propagates any failure back to the kernel unchanged. You will see it at the top of `myfirst_read`, `myfirst_write`, `myfirst_ioctl`, `myfirst_poll`, and the `kqfilter` helpers. When a later lab says "retrieve the per-open state with the usual `devfs_get_cdevpriv` boilerplate", this is the block it refers to, and the rest of the chapter will not re-explain it. If a handler ever re-orders these lines, treat that as a red flag: running any logic before this call means the handler does not yet know which open it is serving. The one subtlety worth remembering is that the `sc == NULL` liveness check comes *after* this boilerplate, not before, because you need the per-open state retrieved safely even on a device that is being torn down.

```c
static int
myfirst_read(struct cdev *dev, struct uio *uio, int ioflag)
{
        struct myfirst_softc *sc = dev->si_drv1;
        struct myfirst_fh *fh;
        off_t before;
        int error;

        error = devfs_get_cdevpriv((void **)&fh);
        if (error != 0)
                return (error);

        if (sc == NULL || !sc->is_attached)
                return (ENXIO);

        before = uio->uio_offset;
        error = uiomove_frombuf(__DECONST(void *, sc->message),
            sc->message_len, uio);
        if (error == 0)
                fh->reads += (uio->uio_offset - before);
        fh->read_off = uio->uio_offset;
        return (error);
}
```

在继续之前有几个值得注意的要点。函数通过 `devfs_get_cdevpriv(9)` 获取每次打开的结构，检查 softc 是否活跃，然后将真正的工作交给 `uiomove_frombuf`。我们在入口处将 `uio->uio_offset` 快照到本地 `before`，这样在调用之后我们可以计算内核刚刚移动的字节数为 `uio->uio_offset - before`。该增量被记录到每次描述符的计数器中。对 `fh->read_off` 的结束赋值记住流位置，以便驱动的其余部分稍后可以报告它。

如果驱动没有数据可以提供，`uiomove_frombuf` 返回零且 `uio_resid` 不变，这是文件结束的报告方式。如果 `uiomove` 内部发生错误，我们通过返回错误代码将其传播回去。这个处理程序不需要直接使用 `copyin` 或 `copyout`。传输的安全性由 `uiomove` 代表我们处理。

### 在源码树中阅读 d_read

完成本节后的一个好阅读练习是在 `/usr/src/sys/dev` 中 grep `d_read`，看看其他驱动在其中做什么。你会发现三种反复出现的形状：

- **从固定缓冲区读取的驱动。** 它们使用 `uiomove_frombuf(9)` 或手工编写的等价物，一次调用，完成。`/usr/src/sys/fs/pseudofs/pseudofs_vnops.c` 广泛使用该辅助函数；字符设备的模式是相同的。
- **从动态缓冲区读取的驱动。** 它们获取内部锁，快照有多少数据可用，以该长度调用 `uiomove(9)`，释放锁，然后返回。我们将在阶段 2 构建其中一个。
- **从阻塞源读取的驱动。** 它们检查数据是否可用，如果不可用，要么在条件变量上睡眠（阻塞模式），要么返回 `EAGAIN`（非阻塞模式）。这是第十章的领域。

所有三种形状共享相同的四行骨架：如果你使用每次打开状态就获取它，验证活跃性，调用 `uiomove`（或其变体），返回错误代码。区别在于它们如何准备缓冲区，而不在于它们如何传输。



## d_write() 的剖析

写入处理程序是读取处理程序的镜像，边缘处有几个小区别。签名来自 `/usr/src/sys/sys/conf.h`：

```c
typedef int d_write_t(struct cdev *dev, struct uio *uio, int ioflag);
```

形状是相同的。三个参数携带相同的含义。返回值是 errno，成功时为零。字节计数仍然从 `uio_resid` 计算：内核查看调用期间 `uio_resid` 减少了多少，并将其作为 `write(2)` 的返回值报告。

### d_write 被要求做什么

再说一句话：**从用户消耗最多 `uio->uio_resid` 个字节，通过 `uiomove(9)` 将它们传递到驱动存储数据的任何地方，并返回零**。

推论几乎与读取完全相同，但有两个显著区别：

- 短写入是合法但不常见的。接受比提供字节更少的驱动必须更新 `uio_resid` 以反映实际情况，内核将向用户空间报告部分计数。大多数行为良好的用户程序会循环并重试剩余部分；许多则不会。经验法则是：接受你能接受的一切，如果你不能再接受更多，对非阻塞调用者返回 `EAGAIN`，对阻塞调用者（最终）睡眠。
- 零字节写入不是文件结束。它只是移动了零字节的写入。`d_write` 没有 EOF 概念；只有读取才有。想要拒绝写入的驱动返回非零 errno。

错误方面最常见的返回值是当驱动缓冲区满时的 `ENOSPC`（设备上没有剩余空间），`uiomove` 内部发生指针相关故障时的 `EFAULT`，以及作为通用硬件错误的 `EIO`。强制执行每次写入长度限制的驱动可以对超过限制的写入返回 `EINVAL` 或 `EMSGSIZE`；我们将在本章稍后讨论选择哪个。

### d_write **不被**要求做什么

与 `d_read` 相同的列表：它不定位用户内存、不检查权限、不为调用者计数字节、不强制执行系统范围限制。内核处理所有这四个。

专门针对写入的一个补充：**不要假设传入的数据是空终止或其他结构化的**。用户可能写入任意字节。如果你的驱动期望结构化输入，它必须防御性地解析它。如果你的驱动期望二进制数据，它必须处理与任何自然边界不对齐的写入。`write(2)` 是字节流，不是消息队列。第25章的 `ioctl` 路径是结构化、成帧命令所属的地方。

### 第一个真正的 d_write

源码树中最简单的非平凡 `d_write` 是 `/usr/src/sys/dev/null/null.c` 中的 `null_write`：

```c
static int
null_write(struct cdev *dev __unused, struct uio *uio, int flags __unused)
{
        uio->uio_resid = 0;

        return (0);
}
```

两行。处理程序通过将 `uio_resid` 设置为零来告诉内核"我消耗了所有字节"，并返回成功。内核将原始请求长度报告给用户空间作为写入的字节数。`/dev/null` 实际上不对字节做任何事；这就是 `/dev/null` 的全部意义。但这种模式很有启发性：**设置 `uio_resid = 0` 是将写入标记为完全消耗的最短方式**，这正是如果我们给它一个目的地，`uiomove(9)` 会做的事情。

一个稍有趣味的案例是 `full_write`，同样在 `null.c` 中：

```c
static int
full_write(struct cdev *dev __unused, struct uio *uio __unused, int flags __unused)
{
        return (ENOSPC);
}
```

这是 `/dev/full` 的后端，一个永远满的设备。每次写入都以 `ENOSPC` 失败，调用者看到相应的 `errno` 值。处理程序不触及 `uio_resid`；内核看到没有字节移动，并报告返回值 -1 和 `errno = ENOSPC`。

这两个处理程序一起说明了写入端的两个极端：接受一切，或拒绝一切。真正的驱动介于两者之间，决定可以接受多少提供的字节并将这些字节存储在某处。

### 实际存储数据的写入

这是我们在本章末尾将实现的写入处理程序的形状。同样，先不要输入；这只是预览以便定位。

```c
static int
myfirst_write(struct cdev *dev, struct uio *uio, int ioflag)
{
        struct myfirst_softc *sc = dev->si_drv1;
        struct myfirst_fh *fh;
        size_t avail, towrite;
        int error;

        error = devfs_get_cdevpriv((void **)&fh);
        if (error != 0)
                return (error);

        if (sc == NULL || !sc->is_attached)
                return (ENXIO);

        mtx_lock(&sc->mtx);
        avail = sc->buflen - sc->bufused;
        if (avail == 0) {
                mtx_unlock(&sc->mtx);
                return (ENOSPC);
        }
        towrite = MIN((size_t)uio->uio_resid, avail);
        error = uiomove(sc->buf + sc->bufused, towrite, uio);
        if (error == 0) {
                sc->bufused += towrite;
                fh->writes += towrite;
        }
        mtx_unlock(&sc->mtx);
        return (error);
}
```

处理程序锁定 softc 互斥锁，检查剩余多少缓冲区空间，将传输限制为适合的数量，以该长度调用 `uiomove(9)`，并通过推进 `bufused` 来记录成功传输。如果缓冲区已满，它返回 `ENOSPC` 以通知调用者。处理程序为处理并发或部分写入所做的一切都体现在锁和限制的组合中。

注意 `uiomove` 本身是在**持有互斥锁的情况下调用的**。只要互斥锁是普通的 `MTX_DEF` 互斥锁（如 `myfirst` 的那样），并且调用上下文是可以睡眠的常规内核线程，这就没问题。`uiomove` 在向用户内存复制或从用户内存复制时可能会发生页面故障，页面故障可能需要内核睡眠等待磁盘读取。持有 `MTX_DEF` 互斥锁睡眠是合法的；持有自旋锁（`MTX_SPIN`）睡眠则是 bug。第三部分正式涵盖锁定规则；现在，相信你在第七章选择的锁类型。

### 与 d_read 的对称性

从驱动的角度来看，读取和写入几乎相同。数据向相反方向流动，`uio->uio_rw` 字段告诉 `uiomove` 向哪个方向移动字节。在驱动侧，你传递相同的参数：指向内核内存的指针、长度和 uio。在用户侧，`uiomove` 要么从内核缓冲区复制出（对于读取），要么复制入（对于写入）。你很少需要考虑方向；`uio_rw` 已经设置好了。

两个处理程序之间变化的是**意图**。读取是驱动产生数据的机会。写入是驱动消耗数据的机会。每个处理程序中的代码知道它在扮演什么角色并做适当的记录：读取者跟踪它已传递了多少，写入者跟踪它已存储了多少。

### 在源码树中阅读 d_write

阅读本节后，花几分钟用 `grep d_write /usr/src/sys/dev | head -20` 看看其他驱动做什么。出现三种形状：

- **丢弃写入的驱动**。通常一行：设置 `uio_resid = 0` 并返回零。`null` 驱动的 `null_write` 是原型。
- **存储写入的驱动**。它们加锁、检查容量、调用 `uiomove(9)`、记录、解锁并返回。我们的阶段 3 处理程序就是这种形状。
- **将写入转发到硬件的驱动**。它们从 uio 中提取数据，将其暂存到 DMA 缓冲区或硬件拥有的环形中，然后触发硬件。这种形状在第四部分之前超出范围；`uiomove` 的机制是相同的，但目标是 DMA 映射区域而非 `malloc` 分配的缓冲区。

每个真正的驱动都符合这三种之一。首先累积或重塑数据，然后转发到某处的驱动倾向于结合形状 2 和形状 3，但原语是相同的。

### 在实际中阅读不熟悉的 d_read 或 d_write

像本章这样的章节最有用的地方是帮助你阅读他人的代码，而不仅仅是你自己的。当你探索 FreeBSD 源码树时，你会遇到看起来一点也不像 `null_write` 或 `zero_read` 的处理程序。形状仍然存在；装饰会有所不同。这里有一个小的阅读协议可以消除猜测。

**第一步：找到返回类型和参数名称。** 每个 `d_read_t` 和 `d_write_t` 都接受相同的三个参数。如果处理程序从 `dev`、`uio` 和 `ioflag` 重命名了它们，注意作者选择了什么（`cdev`、`u`、`flags` 都很常见）。阅读时记住这些名称。

**第二步：找到 `uiomove` 调用（或相关函数）。** 从那里向后追踪以理解传递给它的是什么内核指针和什么长度。这对是处理程序的核心。`uiomove` 调用之前的所有内容都在准备指针和长度；之后的所有内容都在记录。

**第三步：找到锁获取和释放。** 在 `uiomove` 之前获取锁并在之后释放的处理程序正在与其他处理程序串行化。没有锁的处理程序要么在只读数据上操作，要么使用某种其他同步原语（条件变量、引用计数、读取锁）。指出是哪种。

**第四步：找到 errno 返回。** 列出处理程序可能产生的 errno 值。如果列表很短且每个值都有明显的触发条件，处理程序写得很好。如果列表很长或不透明，作者可能留下了一些松散的结尾。

**第五步：找到状态转换。** 处理程序增加什么计数器？它触及什么每次句柄字段？这些转换是驱动的行为签名，通常是驱动之间差异最大的部分。

Apply this protocol to `zero_read` in `/usr/src/sys/dev/null/null.c`. The argument names are the standard ones. The `uiomove` call hands the kernel pointer `zbuf` (pointing at `zero_region`) and a length clamped by `ZERO_REGION_SIZE`. There is no lock; the data is constant. The only errno the handler can return is whatever `uiomove` returned. There are no state transitions; `/dev/zero` is stateless.

Now apply the same protocol to `myfirst_write` at Stage 3. Argument names: standard. `uiomove` call: kernel pointer `sc->buf + bufhead + bufused`, length `MIN((size_t)uio->uio_resid, avail)`. Lock: `sc->mtx` taken before and released after. Errno returns: `ENXIO` (device gone), `ENOSPC` (buffer full), `EFAULT` via `uiomove`, or zero. State transitions: `sc->bufused += towrite`, `sc->bytes_written += towrite`, `fh->writes += towrite`.

两个驱动，同样的协议，两个关于处理程序功能的连贯描述。一旦你应用这种阅读习惯五六次，不熟悉的处理程序就不再看起来陌生。

### 如果你不设置 d_read 或 d_write 会怎样？

初学者有时会好奇的一个细节：如果你的 `cdevsw` 没有设置 `.d_read` 或 `.d_write` 会怎样？简短的回答是内核替换一个默认值，根据哪个特定的槽为空以及设置了哪些其他 `d_flags`，返回 `ENODEV` 或表现得像无操作。详细的回答值得了解，因为真正的驱动确实有意使用默认值，当它们想要表达"此设备不执行读取"或"写入被静默丢弃"时。

看看 `/usr/src/sys/dev/null/null.c` 如何连接其三个驱动：

```c
static struct cdevsw null_cdevsw = {
        .d_version =    D_VERSION,
        .d_read =       (d_read_t *)nullop,
        .d_write =      null_write,
        .d_ioctl =      null_ioctl,
        .d_name =       "null",
};
```

`.d_read` 被设置为内核辅助函数 `nullop`，转换为 `d_read_t *`。`nullop` 是一个通用的"什么都不做，返回零"函数，在 `/usr/src/sys/sys/systm.h` 中声明并在 `/usr/src/sys/kern/kern_conf.c` 中定义；它不接受参数并返回零。它在内核中用于任何方法槽需要无害默认值的地方。转换有效是因为 `d_read_t` 期望一个返回 `int` 的函数，而 `nullop` 的 `int (*)(void)` 形状足够接近，cdevsw 分派可以无意外地调用它。

对于 `/dev/null`，`(d_read_t *)nullop` 意味着"每次读取永远返回零字节"。`cat /dev/null` 的用户看到立即的 EOF。这与安装 `zero_read` 以产生无限零字节流的 `/dev/zero` 不同。两个驱动之间的对比是两种默认读取行为的对比，两者都只是 `cdevsw` 中的一行。

如果你完全省略 `.d_read` 和 `.d_write`，内核用返回 `ENODEV` 的默认值填充它们。当设备确实不支持数据传输时，这是正确的选择；调用者看到清晰的错误而不是静默的成功。但对于应该静默接受写入或产生零字节读取的设备，将槽设置为 `(d_read_t *)nullop` 是惯用的 FreeBSD 做法。

**实用规则：** 刻意决定。要么实现处理程序（用于真实行为），要么将其设置为 `(d_read_t *)nullop` / `(d_write_t *)nullop`（用于无害默认值），要么完全保留未设置（用于 `ENODEV`）。树中的每个真正的驱动都有意选择这三者之一，这个选择对用户可见。

### 第二个真实驱动：mem(4) 如何对两个方向使用一个处理程序

`null.c` 是典型的最小示例。在我们继续之前，一个稍丰富的例子值得一看，因为它演示了你将在树中经常遇到的模式：**一个同时服务于 `d_read` 和 `d_write` 的单一处理程序**，依靠 `uio->uio_rw` 来区分两个方向。

该驱动是 `mem(4)`，它暴露 `/dev/mem` 和 `/dev/kmem`。公共部分位于 `/usr/src/sys/dev/mem/memdev.c`，架构特定的读写逻辑位于 `/usr/src/sys/<arch>/<arch>/mem.c`。在 amd64 上，文件是 `/usr/src/sys/amd64/amd64/mem.c`，函数是 `memrw`。

首先看 `cdevsw`：

```c
static struct cdevsw mem_cdevsw = {
        .d_version =    D_VERSION,
        .d_flags =      D_MEM,
        .d_open =       memopen,
        .d_read =       memrw,
        .d_write =      memrw,
        .d_ioctl =      memioctl,
        .d_mmap =       memmmap,
        .d_name =       "mem",
};
```

`.d_read` 和 `.d_write` 都指向同一个函数。这是合法的，因为 `d_read_t` 和 `d_write_t` 的 typedef 是相同的（都是 `int (*)(struct cdev *, struct uio *, int)`），所以单个函数可以满足两者。诀窍是在处理程序内部读取 `uio->uio_rw` 来决定移动的方向。

`memrw` 的简化草图如下：

```c
int
memrw(struct cdev *dev, struct uio *uio, int flags)
{
        struct iovec *iov;
        /* ... locals ... */
        ssize_t orig_resid;
        int error;

        error = 0;
        orig_resid = uio->uio_resid;
        while (uio->uio_resid > 0 && error == 0) {
                iov = uio->uio_iov;
                if (iov->iov_len == 0) {
                        uio->uio_iov++;
                        uio->uio_iovcnt--;
                        continue;
                }
                /* compute a page-bounded chunk size into c */
                /* ... direction-independent mapping logic ... */
                error = uiomove(kernel_pointer, c, uio);
        }
        /*
         * Don't return error if any byte was written.  Read and write
         * can return error only if no i/o was performed.
         */
        if (uio->uio_resid != orig_resid)
                error = 0;
        return (error);
}
```

这个草图中有三个想法可以推广到你自己的驱动。

**首先，当每个字节的工作相同时，两个方向共用一个处理程序可以节省代码。** `memrw` 中的映射逻辑将用户空间偏移解析为一块内核可访问的内存；你是在从该内存读取还是向其写入，稍后由查看 `uio->uio_rw` 的 `uiomove` 决定。你以一个必须清楚其所在方向的单一函数为代价，避免了近乎相同的读写对重复。如果两个方向几乎不共享任何内容，写两个函数；如果它们几乎共享所有内容，合并它们。

**其次，`memrw` 自己遍历 iovec。** 与将整个传输在一两次调用中交给 `uiomove` 的 `myfirst` 不同，`memrw` 显式遍历 iovec 条目，以便它可以将每个请求的偏移映射到内核内存，然后在映射区域上调用 `uiomove`。这是当你的驱动传递给 `uiomove` 的*内核指针*依赖于正在服务的偏移时使用的模式。它不如 `myfirst` 风格常见，但当传输的每个块对应于驱动后备存储的不同部分时，它是正确的形状。

**第三，注意末尾的 orig_resid 技巧。** 处理程序在入口保存 `uio_resid`，然后在循环之后检查是否有任何内容移动。如果移动了，它返回零（成功），即使后来发生了错误，因为 UNIX 约定要求具有非零字节计数的读取或写入将该计数返回给调用者，而不是使整个调用失败。这是"部分成功"惯用法：如果移动了任何字节，报告字节计数；只在完全没有移动字节时才失败。

你的 `myfirst` 处理程序不需要这个惯用法，因为它们恰好调用一次 `uiomove`。如果 `uiomove` 成功，一切移动了；如果失败，没有移动任何内容（从驱动的记账角度来看）。当你的处理程序循环且循环可能被 `uiomove` 的错误中途中断时，orig_resid 惯用法才重要。记住这个模式；当你的驱动从多个源提供数据时，你将在后面的章节中使用它。

### 为什么这次详解值得绕道

两个驱动。两种非常不同的后备存储。一个原语。在 `null.c` 中，`zero_read` 服务一个预分配的零区域；在 `memrw` 中，处理程序服务按需映射的物理内存。代码在中间看起来不同，因为中间是驱动独特知识所在的地方。两端看起来相同：两个函数都接受 uio，都在 `uio_resid` 上循环，都调用 `uiomove(9)` 进行实际传输，都在成功时返回零或 errno。

这种一致性就是重点。树中每个字符设备的读写都遵循这个形状。一旦你识别了它，你就可以打开 `/usr/src/sys/dev` 下的任何不熟悉的驱动，并自信地阅读处理程序：你还不理解的部分总是在中间，而不是在两端。



## 理解 ioflag 参数

`d_read` 和 `d_write` 都接收第三个参数，本章其余部分几乎没有使用过。本节简短但有用，解释 `ioflag` 是什么、它从哪里来，以及字符设备驱动何时应该实际查看它。

### ioflag 从哪里来

每当进程在 devfs 节点上执行 `read(2)` 或 `write(2)` 时，内核在调用你的处理程序之前从当前文件描述符标志组合 `ioflag` 值。组合在 devfs 本身中，在 `/usr/src/sys/fs/devfs/devfs_vnops.c`。`devfs_read_f` 中的相关行是：

```c
ioflag = fp->f_flag & (O_NONBLOCK | O_DIRECT);
if (ioflag & O_DIRECT)
        ioflag |= IO_DIRECT;
```

`devfs_write_f` 中的模式是镜像的。内核取出文件表 `f_flag` 字中对 I/O 有意义的位，掩码出来，并将该子集作为 `ioflag` 传递。

这很重要，有两个原因。首先，它意味着你的驱动接收的 `ioflag` 是一个*快照*。如果用户程序在两次 `read(2)` 调用之间改变了非阻塞设置（通过 `fcntl(F_SETFL, O_NONBLOCK)`），每次调用都会携带自己最新的 `ioflag`。你不需要缓存状态或监视变化；内核在每次分派时重新派生值。

其次，它意味着你可能期望看到的大多数常量永远不会到达你的处理程序。像 `O_APPEND`、`O_TRUNC`、`O_CLOEXEC` 和各种 `O_EXLOCK` 风格的标志属于文件系统和文件表层。它们不影响字符设备 I/O，也不会被转发。

### 重要的标志位

`IO_*` 标志在 `/usr/src/sys/sys/vnode.h` 中声明。对于字符设备驱动，只有一小部分值得记住：

```c
#define	IO_UNIT		0x0001		/* do I/O as atomic unit */
#define	IO_APPEND	0x0002		/* append write to end */
#define	IO_NDELAY	0x0004		/* FNDELAY flag set in file table */
#define	IO_DIRECT	0x0010		/* attempt to bypass buffer cache */
```

其中，**只有 `IO_NDELAY` 和 `IO_DIRECT` 被组合到你的处理程序接收的 `ioflag` 中**。前三个位用于文件系统 I/O。检查 `IO_UNIT` 或 `IO_APPEND` 的字符设备驱动查看的值将始终为零。

`IO_NDELAY` 是常见情况。当描述符处于非阻塞模式时设置。实现阻塞读取的驱动（第十章）使用此位来决定是睡眠还是返回 `EAGAIN`。第九章的驱动不睡眠于任何东西，所以该位仅供信息参考，但后面的章节依赖它。

`IO_DIRECT` 是一个提示，表示用户程序使用 `O_DIRECT` 打开了描述符，要求内核尽可能绕过缓冲区缓存。对于简单的字符驱动，它几乎总是不相关的。与存储相关的驱动可以选择遵守它；大多数不会。

注意数值一致性：`/usr/src/sys/sys/fcntl.h` 中的 `O_NONBLOCK` 的值为 `0x0004`，`/usr/src/sys/sys/vnode.h` 中的 `IO_NDELAY` 具有相同的值。这不是巧合。`IO_*` 定义上方的头文件注释明确指出 `IO_NDELAY` 和 `IO_DIRECT` 与相应的 `fcntl(2)` 位对齐，这样 devfs 不需要转换。你的驱动可以以任一方式检查位并得到相同的答案。

### 检查 ioflag 的处理程序

这是非阻塞感知的读取处理程序在骨架级别的样子。我们在第九章不会使用这种形状，因为我们从不睡眠，但现在学习它会使第十章的介绍更快。

```c
static int
myfirst_read_nb(struct cdev *dev, struct uio *uio, int ioflag)
{
        struct myfirst_softc *sc = dev->si_drv1;
        int error;

        mtx_lock(&sc->mtx);
        while (sc->bufused == 0) {
                if (ioflag & IO_NDELAY) {
                        mtx_unlock(&sc->mtx);
                        return (EAGAIN);
                }
                /* ... would msleep(9) here in Chapter 10 ... */
        }
        /* ... drain buffer, uiomove, unlock, return ... */
        error = 0;
        mtx_unlock(&sc->mtx);
        return (error);
}
```

对 `IO_NDELAY` 的分支是处理程序关于阻塞的唯一决定。函数中的其他一切都是普通 I/O 代码。这种狭窄性是 `ioflag` 作为单个整数的原因之一：驱动对标志位的响应通常是处理程序顶部附近的一个 `if` 语句，而不是庞大的状态机。

### 第九章各阶段对 ioflag 的处理

阶段 1、阶段 2 和阶段 3 **不**检查 `ioflag`。它们不能阻塞，所以非阻塞位没有意义；它们不关心 `IO_DIRECT`。参数出现在它们的处理程序签名中是因为类型定义要求它，而且它被静默忽略。

当被忽略的行为明显正确时，静默忽略参数不是 bug。使用 `O_NONBLOCK` 打开我们描述符之一的读取者将看到与未使用的读取者相同的行为：两次调用都不睡眠，所以标志没有可观察的效果。第十章是我们将通过标志的地方。

### 一个小型调试辅助

如果你好奇测试期间 `ioflag` 包含什么，入口处的一个 `device_printf` 就会告诉你：

```c
device_printf(sc->dev, "d_read: ioflag=0x%x resid=%zd offset=%jd\n",
    ioflag, (ssize_t)uio->uio_resid, (intmax_t)uio->uio_offset);
```

加载驱动，运行 `cat /dev/myfirst/0`，观察十六进制值。然后运行一个在读取之前使用 `fcntl(fd, F_SETFL, O_NONBLOCK)` 的小程序，观察差异。当你第一次在脑中使机制变真实时，这是一个有启发性的两分钟绕道。

### 源码树中的 ioflag

在 `/usr/src/sys/dev` 中搜索 `IO_NDELAY`，你会发现几十个匹配。几乎每一个都是相同的模式：检查位，如果设置且驱动没有东西可提供则返回 `EAGAIN`，否则睡眠。这种一致性是刻意的。FreeBSD 驱动以相同方式对待非阻塞 I/O，无论它们是伪设备、TTY 线路、USB 端点还是 GEOM 支持的存储，这种一致性是为什么为一种设备编写的用户程序可以干净地移植到另一种的部分原因。



## 深入理解 struct uio

`struct uio` 是内核对 I/O 请求的表示。它被传递给每次 `d_read` 和 `d_write` 调用。每次成功的 `uiomove(9)` 调用都会修改它。你将遇到的每个驱动作者都曾在某个时刻盯着它的字段，想知道哪些可以信任。本节是我们使结构不那么神秘的地方。

### 声明

来自 `/usr/src/sys/sys/uio.h`：

```c
struct uio {
        struct  iovec *uio_iov;         /* scatter/gather list */
        int     uio_iovcnt;             /* length of scatter/gather list */
        off_t   uio_offset;             /* offset in target object */
        ssize_t uio_resid;              /* remaining bytes to process */
        enum    uio_seg uio_segflg;     /* address space */
        enum    uio_rw uio_rw;          /* operation */
        struct  thread *uio_td;         /* owner */
};
```

七个字段。每个都有特定的用途，并且只有一个函数 `uiomove(9)` 协调使用所有这些字段。你的驱动将直接读取其中一些字段；有几个字段你永远不会触及。

### uio_iov 和 uio_iovcnt：分散-聚集列表

单个 `read(2)` 或 `write(2)` 操作一个连续的用户缓冲区。相关的 `readv(2)` 和 `writev(2)` 操作一个缓冲区列表（一个 "iovec"）。内核将两种情况统一表示为 `iovec` 条目列表，简单情况使用长度为一的列表。

`uio_iov` 指向该列表的第一个条目。`uio_iovcnt` 是条目数。每个条目是一个 `struct iovec`，在 `/usr/src/sys/sys/_iovec.h` 中声明：

```c
struct iovec {
        void    *iov_base;
        size_t   iov_len;
};
```

`iov_base` 是指向用户内存（对于 `UIO_USERSPACE` uio）或内核内存（对于 `UIO_SYSSPACE` uio）的指针。`iov_len` 是该条目中剩余的字节数。

你几乎永远不会直接触及这些字段。`uiomove(9)` 为你遍历 iovec 列表，在移动字节时消耗条目，并使列表与剩余传输保持一致。如果你的驱动手动触及 `uio_iov` 或 `uio_iovcnt`，你要么在编写一个非常不寻常的驱动，要么做错了什么。传统的模式是：让 `uiomove` 管理 iovec，读取其他字段以理解请求的状态。

### uio_offset：目标中的偏移

对于常规文件的读取或写入，`uio_offset` 是 I/O 发生在文件中的位置。内核在字节移动时递增它，因此顺序 `read(2)` 自然地在文件中前进。

对于设备文件，`uio_offset` 的含义由驱动定义。真正顺序且没有位置概念的设备将忽略传入的值，并让传出值反映 `uiomove` 做了什么。由固定缓冲区支持的设备将把偏移视为该缓冲区中的地址并遵守它。

`uiomove(9)` 与 `uio_resid` 同步更新 `uio_offset`：对于它移动的每个字节，它将 `uio_resid` 减一并将 `uio_offset` 增一。如果你的驱动每个处理程序调用一次 `uiomove`，你很少需要手动读取 `uio_offset`。如果你的驱动多次调用 `uiomove`，或者如果它使用偏移索引到自己的缓冲区，`uiomove_frombuf(9)` 是你想要的辅助函数。

### uio_resid：剩余字节

`uio_resid` 是仍然需要移动的字节数。在 `d_read` 开始时，它是用户请求的总长度。在成功传输结束时，它是没有移动的内容；内核从原始长度中减去它以产生 `read(2)` 的返回值。

两个有符号算术陷阱值得指出。首先，`uio_resid` 是 `ssize_t`，是有符号的。负值是非法的（`uiomove` 在调试内核中会对其 `KASSERT`），但要注意不要通过粗心的算术意外构造出负值。其次，`uio_resid` 在调用开始时可能为零。当用户程序调用 `read(fd, buf, 0)` 或 `write(fd, buf, 0)` 时会发生这种情况。你的处理程序不能将零视为"没有用户意图"，然后继续对可能未初始化的缓冲区执行 I/O。安全的模式是尽早检查零并返回零（或对于写入，接受零并返回零）。`uiomove` 干净地处理这种情况：它立即返回零而不触及任何内容。所以在实践中"尽早检查"通常是多余的；重要的是你不要*假设*它是非零的。

### uio_segflg：缓冲区所在位置

这个字段说明 iovec 指针指向哪里：用户空间（`UIO_USERSPACE`）、内核空间（`UIO_SYSSPACE`）或直接对象映射（`UIO_NOCOPY`）。枚举在 `/usr/src/sys/sys/_uio.h` 中：

```c
enum uio_seg {
        UIO_USERSPACE,          /* from user data space */
        UIO_SYSSPACE,           /* from system space */
        UIO_NOCOPY              /* don't copy, already in object */
};
```

对于代表用户系统调用调用的 `d_read` 或 `d_write`，`uio_segflg` 是 `UIO_USERSPACE`。`uiomove(9)` 读取该字段并选择正确的传输原语：`copyin` / `copyout` 用于用户空间段，`bcopy` 用于内核空间段。你的驱动不需要对此进行分支；`uiomove` 为你做。

你偶尔会看到手工构建内核模式 uio 的代码，通常是为了重用一个接受 uio 但从内核缓冲区提供服务的函数。该代码将 `uio_segflg` 设置为 `UIO_SYSSPACE`。这是合法且有用的，我们将在实验中简要遇到它。不要将其与用户空间 uio 混淆：安全属性非常不同。

### uio_rw：方向

传输方向。枚举在同一头文件中：

```c
enum uio_rw {
        UIO_READ,
        UIO_WRITE
};
```

对于 `d_read` 处理程序，`uio_rw` 是 `UIO_READ`。对于 `d_write` 处理程序，`uio_rw` 是 `UIO_WRITE`。该字段告诉 `uiomove` 是复制内核->用户（读取）还是用户->内核（写入）。一些处理程序将其作为健全性检查进行断言：

```c
KASSERT(uio->uio_rw == UIO_READ,
    ("Can't be in %s for write", __func__));
```

那个断言来自 `/usr/src/sys/dev/null/null.c` 中的 `zero_read`。这是文档化不变量的廉价方式。你的驱动不需要这样的断言就能正确，但它们在开发期间可以是有用的安全网。

### uio_td：拥有线程

调用者的 `struct thread *`。对于代表系统调用构建的 uio，这是发起系统调用的线程。一些内核 API 需要线程指针；使用 `uio->uio_td` 而不是 `curthread` 在 uio 被传递时保持关联显式。

在直接的 `d_read` 或 `d_write` 中，你很少需要 `uio_td`。如果你的驱动想要在调用中途检查调用者的凭据（超出 `open(2)` 已验证的内容），它就变得有用。这不太常见。

### 图示：read(fd, buf, 1024) 调用期间 uio 发生了什么

演练一次 `read(2)` 有助于巩固字段如何移动。假设用户程序调用：

```c
ssize_t n = read(fd, buf, 1024);
```

内核在到达你的 `d_read` 时构建的 uio 大致如下：

- `uio_iov` 指向一个单条目列表。
- 该条目的 `iov_base = buf`（用户的缓冲区）和 `iov_len = 1024`。
- `uio_iovcnt = 1`.
- `uio_offset = <当前文件指针所在位置>`。对于新打开的可寻址设备，为零。
- `uio_resid = 1024`。
- `uio_segflg = UIO_USERSPACE`。
- `uio_rw = UIO_READ`。
- `uio_td = <调用线程>`。

你的处理程序调用，比如 `uiomove(sc->buf, 300, uio)`。在 `uiomove` 内部，内核：

- 取第一个 iovec 条目。
- 确定 300 小于 1024，所以它将移动 300 字节。
- 调用 `copyout(sc->buf, buf, 300)`。
- 将 `iov_len` 减 300，变为 724。
- 将 `iov_base` 推进 300，变为 `buf + 300`。
- 将 `uio_resid` 减 300，变为 724。
- 将 `uio_offset` 增 300。

你的处理程序返回零。内核计算字节计数为 `1024 - 724 = 300` 并从 `read(2)` 返回 300。用户在 `buf[0..299]` 中看到 300 字节，并知道要么再次调用 `read(2)` 获取剩余内容，要么继续使用已有的内容。

这就是 `uiomove` 按顺序做的一切。没有魔法。

### readv(2) 有何不同

如果用户使用三个 iovec 条目调用 `readv(fd, iov, 3)`，`d_read` 开始时的 uio 将有 `uio_iovcnt = 3`，`uio_iov` 指向三个条目的列表，`uio_resid` 等于它们长度的总和。你的处理程序进行一次 `uiomove` 调用（或在循环中多次调用），`uiomove` 为你遍历列表。驱动代码是相同的。

这是 uio 抽象的隐秘好处之一：分散-聚集读取和写入是免费的。你的驱动是为单个缓冲区编写的；它已经处理了多缓冲区请求。

### 使用部分消耗的 uio 重新进入处理程序

一个偶尔让初学者绊倒的约定：**单个 `d_read` 或 `d_write` 调用可以进行多次 `uiomove` 调用**。每次调用缩小 `uio_resid` 并推进 `uio_iov`。uio 在调用之间保持一致。如果你的处理程序第一次 `uiomove` 移动了 128 字节，下一次移动了 256 字节，内核只看到一次传输了 384 字节的单个处理程序调用。

你**不应该**做的是跨处理程序调用保存 uio 指针并尝试稍后恢复它。uio 在产生它的分派期间有效。在分派之间，它指向的内存（包括 iovec 数组）可能无效。如果你需要排队请求以供后续处理，你将必要数据从 uio 复制出来（到你自己的内核缓冲区）并使用你自己的队列。

### 你的驱动需要读取什么和应该避免什么

简短速查表，按使用频率递减排列：

| Field          | Read it?    | Write it?                         |
|----------------|-------------|-----------------------------------|
| `uio_resid`    | Yes, often  | Only to mark a transfer consumed (e.g., `uio_resid = 0`) |
| `uio_offset`   | Yes, if you honour it | No, let `uiomove` update it |
| `uio_rw`       | Occasionally, for KASSERTs | No |
| `uio_segflg`   | Rarely       | No, unless building a kernel-mode uio |
| `uio_td`       | Rarely       | No |
| `uio_iov`      | Almost never | Never |
| `uio_iovcnt`   | Almost never | Never |

如果初学者级别的驱动写入 `uio_iov` 或 `uio_iovcnt`，说明出了严重的问题。如果它写入 `uio_resid` 而不是 `uio_resid = 0` "我消耗了一切"的技巧，说明稍有偏差。如果它读取前三行，它就在正常路径上。

### uio 字段的实际使用

一旦你看到处理程序实际使用了它，这一切就不那么令人生畏了。本章中的 myfirst 阶段检查 `uio_resid`（用于限制传输），偶尔读取 `uio_offset`（用于知道读取者在哪里），并将其余一切交给 `uiomove`。辅助函数做真正的工作，驱动代码保持精简。

### 单个 uio 的生命周期：三个快照

为了巩固逐字段的讨论，值得演练一下 uio 在其生命周期中三个点的状态：你的处理程序被调用的时刻、部分 `uiomove` 之后的时刻、以及你的处理程序即将返回的时刻。每个快照捕获的是同一个 uio，所以你可以准确看到字段如何演变。

示例是对一个驱动的 `read(fd, buf, 1024)` 调用，该驱动的读取处理程序每次调用将提供 300 字节。

**快照 1：`d_read` 入口处。**

```text
uio_iov     -> [ { iov_base = buf,       iov_len = 1024 } ]
uio_iovcnt  =  1
uio_offset  =  0        (this is the first read on the descriptor)
uio_resid   =  1024
uio_segflg  =  UIO_USERSPACE
uio_rw      =  UIO_READ
uio_td      =  <calling thread>
```

uio 描述了一个完整的请求。用户请求 1024 字节，缓冲区在用户空间，方向是读取，偏移为零。这就是内核传递给你处理程序的内容。

**快照 2：`uiomove(sc->buf, 300, uio)` 成功返回后。**

```text
uio_iov     -> [ { iov_base = buf + 300, iov_len =  724 } ]
uio_iovcnt  =  1
uio_offset  =  300
uio_resid   =  724
uio_segflg  =  UIO_USERSPACE    (unchanged)
uio_rw      =  UIO_READ         (unchanged)
uio_td      =  <calling thread> (unchanged)
```

四个字段同步变化。`iov_base` 前进了 300，所以下一次传输将把字节放在刚刚写入的字节之后。`iov_len` 缩小了 300，因为 iovec 条目现在只描述剩余的 724 字节。`uio_offset` 增加了 300，因为 300 字节的流位置移动了。`uio_resid` 缩小了 300，因为 300 字节的工作已完成。

三个字段保持不变：`uio_segflg`、`uio_rw` 和 `uio_td` 描述请求的*形状*，这在传输中途不会改变。如果你的处理程序需要检查其中任何一个，它可以在 `uiomove` 之前或之后检查并得到相同的答案。

**快照 3：`d_read` 返回之前。**

假设处理程序在提供 300 字节后决定没有更多数据，于是返回零而不再次调用 `uiomove`。

```text
uio_iov     -> [ { iov_base = buf + 300, iov_len =  724 } ]
uio_iovcnt  =  1
uio_offset  =  300
uio_resid   =  724
uio_segflg  =  UIO_USERSPACE
uio_rw      =  UIO_READ
uio_td      =  <calling thread>
```

与快照 2 相同。处理程序没有触及任何东西；它只是返回了。内核将看到 `uio_resid = 724` 对比开始的 `uio_resid = 1024`，并计算 `1024 - 724 = 300`，它将其作为 `read(2)` 的结果返回给用户空间。调用者看到返回值 300，知道驱动产生了 300 字节。

如果处理程序循环调用 `uiomove` 直到 `uio_resid` 达到零，返回时的快照将变为 `uio_resid = 0`，内核将向用户空间返回 1024（完整传输）。如果处理程序调用了 `uiomove` 并收到错误，`uio_resid` 将反映故障发生前的部分进度，处理程序将返回 errno。

### 这个心智模型给你带来什么

三个观察从快照中得出，值得明确命名。

**首先，uio_resid 是契约。** 你的处理程序返回时 `uio_resid` 中的任何值，内核都会信任。如果它比入口处小，说明移动了一些字节；差值就是字节计数。如果它没有改变，说明没有移动任何内容；返回值将是零（EOF）或 errno（取决于你的处理程序返回了什么）。

**其次，uiomove 是你应该依赖的唯一东西来递减 uio_resid。** 手动从 `uio_resid` 减去的驱动几乎肯定做错了；内核的故障处理、iovec 遍历和偏移更新都内置于 `uiomove` 代码路径中。设置 `uio_resid = 0` 是唯一的例外，被 `null.c` 的 `null_write` 等驱动用来表示"假装所有字节都被消耗了"。

**第三，uio 是临时空间。** uio 不是长期存在的对象。它每个系统调用创建一次，随着 `uiomove` 消耗它而衰减，并在你的处理程序返回时被丢弃。保存 uio 指针以供后续使用是一个等待触发的生命周期 bug。如果你的驱动需要在当前调用之外使用 uio 中的数据，它会将字节复制到自己的存储中（这就是 `d_write` 所做的：它通过 `uiomove` 将字节复制到 `sc->buf` 中，在 uio 中不留任何内容以供后续使用）。

这三个事实是本章其余内容构建的基础。如果你内化了它们，uio 机制的其余部分就不再神秘。



## 安全数据传输：uiomove、copyin、copyout

前面的章节描述了 `struct uio` 并将 `uiomove(9)` 命名为移动字节的函数。本节解释为什么该函数存在、它在底层做什么，以及驱动何时应该直接使用 `copyin(9)` 或 `copyout(9)`。

### 为什么直接内存访问不安全

用户进程有自己的虚拟地址空间。当进程使用缓冲区指针调用 `read(2)` 时，该指针是进程地址空间中的虚拟地址。它可能引用存在于物理 RAM 中的内存页，或已被换出的页，或根本没有映射的页。它甚至可能是用户程序故意伪造的指针，试图使内核崩溃。

从内核的角度来看，用户的地址空间不是直接可寻址的。内核有自己的地址空间；传递给内核的用户指针作为内核指针没有意义。即使内核可以通过页表机制解析用户指针，直接使用它也是危险的：页面可能故障、内存保护可能错误、地址可能落在进程映射区域之外，或者指针可能被构造为指向内核内存以试图泄露或破坏它。

换言之，直接内存访问不是内核免费获得的功能。它是一种必须谨慎行使的特权，每次访问都必须通过知道如何处理故障、检查保护并保持用户和内核地址空间分离的函数。

在 FreeBSD 上，这些函数是 `copyin(9)`（用户到内核）、`copyout(9)`（内核到用户）和 `uiomove(9)`（任一方向，由 uio 驱动）。

### copyin(9) 和 copyout(9) 做什么

来自 `/usr/src/sys/sys/systm.h`：

```c
int copyin(const void * __restrict udaddr,
           void * __restrict kaddr, size_t len);

int copyout(const void * __restrict kaddr,
            void * __restrict udaddr, size_t len);
```

`copyin` 接受用户空间指针、内核空间指针和长度。它从用户复制 `len` 字节到内核。`copyout` 相反：内核指针、用户指针、长度。从内核复制到用户。

两个函数都验证用户地址、必要时调入用户页面、执行复制并捕获任何发生的故障。它们成功时返回零，如果用户地址无效或复制失败则返回 `EFAULT`。它们从不静默地破坏内存；它们要么完成复制，要么报告错误。

这两个原语是所有用户/内核内存传输构建的基础。当 uio 在用户空间时，`uiomove(9)` 在底层调用它们。`fubyte(9)`、`subyte(9)` 和其他一些便利函数使用它们。它们是内核信任为信任边界的函数。

### uiomove(9) 做什么

`uiomove(9)` 是 `copyin` / `copyout` 的包装器，理解 uio 结构。它的实现很短，值得阅读；位于 `/usr/src/sys/kern/subr_uio.c`。

大致上，算法是：

1. 健全性检查 uio：方向有效、resid 非负、如果段是用户空间则拥有线程是当前线程。
2. 循环：当调用者请求更多字节（`n > 0`）且 uio 仍有空间（`uio->uio_resid > 0`），消耗下一个 iovec 条目。
3. 对于每个 iovec 条目，计算要移动多少字节（条目长度、调用者剩余计数和 uio 的 resid 的最小值），并根据 `uio_rw` 和 `uio_segflg` 调用 `copyin` 或 `copyout`（对于用户空间段）或 `bcopy`（对于内核空间段）。
4. 随着字节移动推进 iovec 的 `iov_base` 和 `iov_len`；递减 `uio_resid`，递增 `uio_offset`。
5. 如果任何复制失败，跳出并返回错误。

函数成功时返回零，失败时返回 errno 代码。最常见的失败是来自错误用户指针的 `EFAULT`。

`uiomove` 的关键属性是它是**你的驱动应该用来通过 uio 移动字节的唯一函数**。不是 `bcopy`，不是 `memcpy`，不是 `copyout`。uio 携带 `uiomove` 选择正确原语所需的信息，驱动不需要猜疑。

### 何时使用哪个

在实践中，分工是直接的。

在 `d_read` 和 `d_write` 处理程序中使用 `uiomove(9)`，只要 uio 描述传输。这是绝大多数情况。

当你有来自 uio 之外的用户指针时，直接使用 `copyin(9)` 和 `copyout(9)`。示例：

- 在 `d_ioctl` 处理程序内部，用于携带用户空间指针作为参数的控制命令（第25章）。
- 在通过你自己构建的机制（而非 uio）接受用户提供数据的内核线程内部。
- 当读取或写入一小块固定大小的用户内存，而这块内存不是系统调用的主题时。

在 `d_read` 或 `d_write` 内部**不要**使用 `copyin` 或 `copyout` 从 uio 的 iovec 获取数据。始终通过 `uiomove`。iovec 不保证是单个连续缓冲区，即使它是，你的驱动也没有越过 uio 抽象直接触及它的业务。

### 快速参考表

| 情况                                         | 首选工具         |
|---------------------------------------------------|------------------------|
| 通过 uio 传输字节（读取或写入）  | `uiomove(9)`           |
| 通过 uio 传输字节，带有固定内核缓冲区和自动偏移 | `uiomove_frombuf(9)` |
| 读取不由 uio 携带的已知用户指针 | `copyin(9)`            |
| 写入到不由 uio 携带的已知用户指针 | `copyout(9)`         |
| 从用户空间读取空终止字符串  | `copyinstr(9)`         |
| 从用户空间读取单个字节             | `fubyte(9)`            |
| 向用户空间写入单个字节               | `subyte(9)`            |

`fubyte` 和 `subyte` 是小众的；大多数驱动从不使用它们。它们在这里列出是为了识别。`copyinstr` 偶尔在接受用户字符串的控制路径中有用；我们不会在本章中使用它。

### 为什么不能用简单的 memcpy？

初学者有时会问"我可以直接转换用户指针并用 `memcpy` 复制字节吗？"答案是不加限定的不行，值得理解为什么。

`memcpy` 假设两个指针都指向当前地址空间中可访问的内存。用户指针不保证指向可访问的内存。在硬件级别分离用户和内核指针的架构上（例如 amd64 上的 SMAP），CPU 将拒绝访问。在共享地址空间的架构上，指针可能仍然无效，或者可能指向已被换出的页面，或者可能指向内核被禁止触及的页面。这些情况中没有一种可以在普通 `memcpy` 内安全处理；由此产生的故障要么会导致系统崩溃，要么会跨信任边界泄露信息。

内核原语 `copyin` 和 `copyout` 的存在正是为了正确处理这些情况。它们在访问之前安装故障处理程序，所以错误的用户指针返回 `EFAULT` 而不是崩溃。它们遵守 SMAP 和类似的保护。它们可以等待页面被换入。这些都不是可选的，也不是你的驱动应该复制的东西。

实用规则：如果指针来自用户空间，通过 `copyin` / `copyout` / `uiomove` 路由它。不要直接解引用它。不要通过它 `memcpy`。不要将它传递给任何会通过它 `memcpy` 的函数。如果你停在抽象边界，内核给你一个稳定的、安全的、文档良好的接口。如果你越过它，你将永远拥有每一个 bug。

### 故障时发生什么

一些具体的：当用户指针错误时 `uiomove` 实际做什么？

内核在复制之前安装故障处理程序，通常通过其陷阱处理程序表。当 CPU 在用户访问上发生故障时，故障处理程序注意到故障指令位于 `copyin` 或 `copyout` 代码路径内，跳到失败返回路径，并返回 `EFAULT`。没有崩溃。没有数据损坏。`uiomove` 的调用者看到非零返回值，将其传播给 `d_read` 或 `d_write` 的调用者，系统调用以 `errno = EFAULT` 返回用户态。

驱动不需要做任何特殊的事情来配合这个机制。它只需要检查 `uiomove` 的返回值并传播错误。我们将在本章的每个处理程序中这样做。

### 对齐和类型安全

另一个值得指出的微妙之处。用户的缓冲区是字节流。它不携带类型信息。如果你的驱动将 `struct` 放入缓冲区，用户将其拉出，用户得到的是字节；这些字节可能与调用者架构上的 `struct` 访问正确对齐，也可能不对齐。

对于 `myfirst`，这个问题不会出现，因为字节是任意用户文本。对于想要导出结构化数据的驱动，约定是要求用户在解释字节之前将其 memcpy 到对齐的本地结构中，或者在数据格式中包含显式对齐和版本协商。`ioctl(2)` 避免了这个问题，因为它的数据布局是 `IOCTL` 命令号的一部分；`read` 和 `write` 没有这种奢侈。

这是将结构化数据绑定到 `read`/`write` 上既诱人又错误的地方之一。如果你的驱动想要将类型化数据交给用户态，`ioctl` 接口或外部 RPC 机制是正确的工具。`read` 和 `write` 携带字节。这就是承诺，承诺使它们可移植。

### 一个小型计算示例

假设用户程序向你的设备写入四个整数：

```c
int buf[4] = { 1, 2, 3, 4 };
write(fd, buf, sizeof(buf));
```

在驱动的 `d_write` 中，uio 看起来像：:

- 一个 iovec 条目，`iov_base = <用户 buf[0] 的地址>`，`iov_len = 16`.
- `uio_resid = 16`，`uio_offset = 0`，`uio_segflg = UIO_USERSPACE`，`uio_rw = UIO_WRITE`.

一个天真的处理程序可能会调用 `uiomove(sc->intbuf, 16, uio)`，其中 `sc->intbuf` 是 `int sc->intbuf[4];`。`uiomove` 会发出一个复制 16 字节的 `copyin`。成功时，`sc->intbuf` 将保存调用程序字节顺序的四个整数。

但注意：如果驱动跨架构使用，用户可能以完全不同 CPU 的字节顺序写入了那些整数。用户可能在驱动使用 `int` 的地方使用了 `int32_t`。用户可能以不同方式填充了结构。对于 `myfirst`，这些都不重要，因为我们将数据视为不透明字节。对于通过 `read`/`write` 暴露结构化数据的驱动，这些问题会迅速倍增，这也是大多数真正的驱动要么使用 `ioctl` 处理结构化载荷，要么在文档中声明显式线格式（字节顺序、字段宽度、对齐）的原因。

教训：`uiomove` 移动字节。它不知道也不关心类型。你的驱动必须决定这些字节意味着什么。

### 迷你案例研究：当结构体往返出错时

为了使"字节不是类型"这一点具体化，让我们演练一个看似合理但错误的尝试，即通过 `read(2)` 作为类型化结构暴露内核计数器。

假设你的驱动维护一组计数器：

```c
struct myfirst_stats {
        uint64_t reads;
        uint64_t writes;
        uint64_t errors;
        uint32_t flags;
};
```

并且乐观地假设，你通过 `d_read` 暴露它们：

```c
static int
stats_read(struct cdev *dev, struct uio *uio, int ioflag)
{
        struct myfirst_softc *sc = dev->si_drv1;
        struct myfirst_stats snap;

        mtx_lock(&sc->mtx);
        snap.reads  = sc->stat_reads;
        snap.writes = sc->stat_writes;
        snap.errors = sc->stat_errors;
        snap.flags  = sc->stat_flags;
        mtx_unlock(&sc->mtx);

        return (uiomove(&snap, sizeof(snap), uio));
}
```

乍一看这没问题。字节到达用户空间。读取者可以将缓冲区转换为 `struct myfirst_stats` 并查看字段。作者在 amd64 上测试它，看到正确的值，发布驱动。

三个问题正在那里等待。

**问题 1：结构体填充。** `struct myfirst_stats` 的布局取决于编译器和架构。在 amd64 上使用默认 ABI，`uint64_t` 是 8 字节对齐的，所以结构体是 `reads` 的 8 字节，`writes` 的 8 字节，`errors` 的 8 字节，`flags` 的 4 字节，加上 4 字节的尾部填充将大小四舍五入为 32。用户程序必须声明一个具有*相同*填充的结构体才能正确读取字段。使用 `#pragma pack(1)` 重新声明结构体或使用不同编译器版本的用户程序将错误解析字节并在 `errors` 中看到垃圾。

**问题 2：字节顺序。** amd64 机器以小端存储 `uint64_t`。在同一架构上运行的用户程序正确解码。在大端机器上远程运行的用户程序，通过网络管道读取字节，看到的是字节交换的整数。驱动没有选择线上字节顺序，所以格式偶然地依赖于 CPU。

**问题 3：快照的原子性。** 读取者可能在 `mtx_unlock` 释放互斥锁之后、内核将控制权返回给调用者之前通过 `uiomove` 提取字节。在这两个时刻之间，字段 `snap.reads`、`snap.writes` 等已经捕获在栈本地 `snap` 中，所以*那*部分是没问题的。但示例足够小以至于 bug 没有出现；更大的快照可能跨越多个互斥锁获取而表现出撕裂读取。

**修复不是在结构体布局上"更加努力"。** 修复是停止使用 `read(2)` 处理结构化数据。存在两个更好的选项：

- **`sysctl`**：本章一直在使用它。各个计数器作为已知类型的命名节点暴露。用户侧的 `sysctl(3)` 直接返回整数；没有结构体布局，没有填充，没有字节顺序。
- **`d_ioctl`**：第25章正确构建了 `ioctl`。对于这个用例，一个具有明确定义请求结构的 `ioctl` 会是合适的，而 `_IOR` / `_IOW` 宏文档化大小和方向。

`read(2)` 接口承诺"驱动定义的字节流"；仅此而已。如果你尊重承诺，你的驱动是可移植的、可测试的，并且能够抵抗静默的布局漂移。如果你通过暴露类型化结构打破承诺，你将继承网络协议花费数十年学习绕过的每一个 ABI 陷阱。

对于本章中的 `myfirst`，我们从未遇到这个问题，因为我们只推送和拉取不透明的字节流。案例研究的重点是帮助你在有人递给你一个已经在犯错的驱动之前识别错误的形状。

### 本节总结

- 在 `d_read` 和 `d_write` 内部使用 `uiomove(9)`。它读取 uio，选择正确的原语，并为你处理用户/内核故障。
- 当你想要自动偏移到缓冲区算术时使用 `uiomove_frombuf(9)`。
- 仅当你在 uio 上下文之外有用户指针时才使用 `copyin(9)` 和 `copyout(9)`，通常在 `d_ioctl` 中。
- 不要直接解引用用户指针。永远不要。
- 检查返回值。任何复制都可能因 `EFAULT` 而失败，你的处理程序必须传播错误。

这些规则很短，但它们涵盖了初学者驱动几乎犯下的每一个 I/O 安全错误。



## 管理驱动中的内部缓冲区

读取和写入处理程序是驱动 I/O 路径的可见表面。在它们背后，驱动必须在某处存储数据。本节是关于存储如何设计、分配、保护和清理的，在本章需要的初学者友好级别。第十章将把缓冲区扩展为真正的环形并使其在负载下并发安全；我们在这里故意不涉及。

### 为什么需要缓冲区

缓冲区是 I/O 调用之间的临时存储。驱动至少因三个原因使用它：

1. **速率匹配。** 生产者和消费者不会同时到达。写入可以存入稍后读取将取走的字节。
2. **请求重塑。** 用户可能以与驱动产生数据不一致的单位读取。缓冲区吸收这种不匹配。
3. **隔离。** 驱动缓冲区内的字节是内核数据。它们不是用户指针、不是 DMA 地址、不在分散-聚集列表中。内核缓冲区中的一切都可以由驱动安全统一地寻址。

对于 `myfirst`，缓冲区是一小块 RAM 内存储。`d_write` 向其中写入；`d_read` 从其中读取。缓冲区是驱动的状态。uio 机制是移动字节进出它的管道。

### 静态与动态分配

关于缓冲区位置存在两种合理的设计。

**静态分配**将缓冲区放在 softc 结构内部或作为模块级数组：

```c
struct myfirst_softc {
        ...
        char buf[4096];
        size_t bufused;
        ...
};
```

优点：分配永不失败，大小明确，生命周期轻松绑定到 softc。缺点：大小在编译时固定；如果以后想让它可调，你需要重构。

**动态分配**使用 `malloc(9)` 从 `M_*` 桶分配：

```c
sc->buf = malloc(sc->buflen, M_DEVBUF, M_WAITOK | M_ZERO);
```

优点：大小可以在 attach 时从 sysctl 或可调参数选择；如果小心可以调整大小。缺点：分配可能失败（`M_WAITOK` 相关性较小，`M_NOWAIT` 更相关）；驱动拥有另一个释放路径。

对于小缓冲区，在 softc 内部的静态分配是最简单的选择，这也是第七章隐式使用的选择，依赖 Newbus 分配整个 softc。第九章将使用动态分配，因为缓冲区足够大，把它放在 softc 中有点浪费，而且动态路径是你在本书后面会反复使用的模式。

### malloc(9) 调用

内核的 `malloc(9)` 接受三个参数：大小、malloc 类型（内核用于记账和调试的标签）和标志字。常见形状：

```c
sc->buf = malloc(sc->buflen, M_DEVBUF, M_WAITOK | M_ZERO);
```

`M_DEVBUF` 是通用的"设备缓冲区" malloc 类型，在树中定义，适用于不值得拥有专用类型的驱动私有数据。如果你的驱动增长到足以拥有自己的标签，你可以用 `MALLOC_DECLARE(M_MYFIRST)` 和 `MALLOC_DEFINE(M_MYFIRST, "myfirst", "myfirst driver data")` 声明一个，并使用 `M_MYFIRST` 代替。目前，`M_DEVBUF` 就可以。

此阶段最相关的标志位：

- `M_WAITOK`：分配睡眠等待内存是可以的。在 attach 上下文中，这几乎总是正确的选择。
- `M_NOWAIT`：不要睡眠；如果内存紧张则返回 `NULL`。当你处于不能睡眠的上下文（中断处理程序、非可睡眠锁内部）时需要。
- `M_ZERO`：返回前将内存清零。根据情况与 `M_WAITOK` 或 `M_NOWAIT` 配对。

在 FreeBSD 14.3 上，带有 `M_WAITOK` 而没有 `M_NOWAIT` 的调用保证返回有效指针。内核会睡眠并在需要时可能触发回收，但在实践中不会返回 `NULL`。尽管如此，检查 `NULL` 是防御性的且成本为零；我们会这样做。

### 对应的 free(9) 调用

每个 `malloc(9)` 都有匹配的 `free(9)`。签名是：

```c
free(sc->buf, M_DEVBUF);
```

传递给 `free` 的 malloc 类型必须与同一指针传递给 `malloc` 的类型匹配。传递不同的类型会损坏内核的记账，这是 `INVARIANTS` 启用的内核在运行时捕获的错误之一。

`free` 放在哪里取决于 `malloc` 在哪里：attach 分配，detach 释放。如果 attach 中途失败，错误展开路径释放在失败之前分配的所有内容。我们在第七章见过这个模式；我们将在这里重用它。

### 缓冲区大小

选择缓冲区大小是一个设计选择。对于课堂驱动，任何小大小都可以。一些指导原则：

- **小**（几百字节到几千字节）：适合演示。易于推理。大于缓冲区的用户工作负载会快速观察到 `ENOSPC` 或短读取；这是教学特性，不是 bug。
- **页大小**（4096 字节）：常见、合理的默认值。内存分配免费页对齐，许多工具将 4 KiB 视为自然单位。
- **更大**（几千字节到兆字节）：适用于期望缓冲大量数据的驱动。记住内核内存不是无限的；每次打开分配兆字节的失控驱动可能使系统不稳定。

对于 `myfirst` 阶段 2，我们将使用 4096 字节缓冲区。它足够大，可以容纳合理的测试（一段文本，几个整数），也足够小，`ENOSPC` 行为易于从 shell 触发。

### 缓冲区溢出

管理自己缓冲区的驱动中最常见的 bug 是写入超过缓冲区末尾。这个 bug 在内核空间绝对是致命的。越界缓冲区的用户空间程序可能损坏自己的堆；这样做的内核模块可能损坏另一个子系统的内存，崩溃（或更糟，静默的错误行为）可能出现在远离 bug 的地方。

防御是算术纪律。每次你的代码要在大小为 `S` 的缓冲区中从偏移 `O` 开始写入 `N` 字节时，在写入之前验证 `O + N <= S`。在阶段 3 的处理程序中，表达式 `towrite = MIN((size_t)uio->uio_resid, avail)` 正是那个检查：`towrite` 被限制为 `avail`，其中 `avail` 是 `sc->buflen - sc->bufused`。无法超过 `sc->buflen`。

一个相关的 bug 是有符号与无符号混淆。`uio_resid` 是 `ssize_t`；`sc->bufused` 是 `size_t`。不小心混合它们可能产生一个负值，当转换为 `size_t` 时会回绕，后果是灾难性的。`MIN` 宏和显式的 `(size_t)` 转换值得它们给代码添加的少量噪音。

### 加锁注意事项

如果你的驱动可以同时从多个用户上下文进入，缓冲区需要一个锁。两个同时写入者可能在 `bufused` 上竞争；两个同时读取者可能在读取偏移上竞争；一个写入者和一个读取者可能以损坏两者的方式交错它们的状态更新。

在 `myfirst` 中，我们从第七章开始携带的 `struct mtx mtx` 字段是我们要使用的锁。它是一个普通的 `MTX_DEF` 互斥锁，这意味着它可以跨 `uiomove` 调用持有（`uiomove` 可能因页面故障而睡眠）。我们将在每次更新 `bufused` 时以及将字节传入或传出共享缓冲区的 `uiomove` 期间持有它。

第三部分更深入地讨论锁定策略。现在，规则是：**保护任何可能同时被多个处理程序触及的字段**。在阶段 3 中，那是 `sc->buf` 和 `sc->bufused`。你的每次打开 `fh` 是每次描述符的；它不需要相同的锁，因为在我们将练习的情况下，两个处理程序不能同时为同一描述符运行。

### 循环缓冲区预览

第十章构建了一个真正的环形缓冲区：一个 `head` 和 `tail` 指针相互追逐的固定大小缓冲区。它与我们在第九章使用的线性缓冲区有两个不同：

1. 它不需要在使用之间重置。指针环绕；缓冲区就地重用。
2. 它可以在稳态下支持流式传输。线性缓冲区填满后拒绝写入；环形缓冲区维护一个移动的最近数据窗口。

本章的阶段 3 *不*实现环形。它实现了一个线性缓冲区，`d_write` 向其中追加，`d_read` 从其中排空。当缓冲区满时，`d_write` 返回 `ENOSPC`；当它为空时，`d_read` 返回零字节。这足以正确实现 I/O 路径，而不需要环形的额外簿记。第十章在相同的处理程序形状之上添加了那些簿记。

### 关于每次描述符 fh 的线程安全说明

你的驱动在 `d_open` 中分配的 `struct myfirst_fh` 是每次描述符的。本章练习的场景中，同一描述符的两个处理程序不能并发执行（内核通过文件描述符机制为常见情况序列化每次文件操作），所以 `fh` 内部的字段不需要自己的锁。*不同*描述符的两个处理程序确实并发运行，但它们触及不同的 `fh` 结构。

这是一个令人欣慰的不变量，但不是绝对的。安排将 `fh` 指针传递给与系统调用并行运行的内核线程的驱动必须添加自己的同步。我们在本章不会那样做；目前只要你只从被给予描述符的处理程序内部触及 `fh`，`fh` 就是安全的。

### 你应该认识的内核辅助函数

在继续之前，值得指出本章一直在使用的一小部分辅助宏和函数。它们在标准 FreeBSD 头文件中定义，初学者有时复制粘贴使用它们的代码而不知道它们来自哪里或适用什么约束。

`MIN(a, b)` 和 `MAX(a, b)` 通过 `<sys/libkern.h>` 在内核代码中可用，`<sys/libkern.h>` 由 `<sys/systm.h>` 传递引入。它们最多对每个参数求值两次，所以 `MIN(count++, limit)` 是一个 bug：`count` 会递增两次。编写良好的驱动避免在 `MIN`/`MAX` 参数中产生副作用。

```c
towrite = MIN((size_t)uio->uio_resid, avail);
```

显式的 `(size_t)` 转换是模式的一部分，不是风格装饰。`uio_resid` 是 `ssize_t`，是有符号的；`avail` 是 `size_t`，是无符号的。没有转换，编译器为比较选择一种类型，现代编译器在同一 `MIN` / `MAX` 中遇到有符号和无符号时会发出警告。转换使意图明确：我们已经检查过 `uio_resid` 是非负的（内核保证），所以转换是安全的。

`howmany(x, d)`，定义在 `<sys/param.h>` 中，计算 `(x + d - 1) / d`。当你需要向上取整除法时使用它。分配页面来保存字节计数的驱动通常写：

```c
npages = howmany(buflen, PAGE_SIZE);
```

`rounddown(x, y)` 和 `roundup(x, y)` 将 `x` 向下或向上对齐到 `y` 的最近倍数。`roundup2` 和 `rounddown2` 是更快的变体，只在 `y` 是 2 的幂时工作。这些是驱动如何页对齐缓冲区或块对齐偏移的。

`__DECONST(type, ptr)` 在没有编译器警告的情况下移除 `const`。这是告诉编译器"我知道这个指针声明为 `const`，但我已经验证我调用的函数不会修改数据，所以请停止抱怨"的礼貌方式。在 `null.c` 的 `zero_read` 中围绕 `zero_region` 使用了它；我们在阶段 1 的 `myfirst_read` 中使用了它。比普通的 `(void *)` 转换更可取，因为它表达了意图。

`curthread` 是一个架构特定的宏（通过每 CPU 寄存器解析），指向当前执行的线程。当 uio 来自系统调用时，`uio->uio_td` 通常等于 `curthread`；在这种上下文中两者可以互换，但 uio 携带的值更自文档化。

`bootverbose` 是一个整数，如果内核使用 `-v` 启动或操作员通过 sysctl 切换了它，则设置为非零。用 `if (bootverbose)` 保护冗长的日志行是 FreeBSD 的惯用法，用于按需可见但默认静默的调试日志。

在其他驱动中遇到这些辅助函数时识别它们，可以缩短阅读不熟悉代码所需的时间。它们都不是异类的；它们都是内核贡献者应该能够不查就能阅读的标准词汇。



## 错误处理和边界情况

一个"只在正常路径上工作"的初学者驱动最终会崩溃内核。 I/O 处理的有趣部分不是正常路径的部分: 零长度读取、部分写入、错误的用户指针、调用中途传递的信号、耗尽的缓冲区, and several dozen variations of those. 本节讨论常见情况和与之相关的 errno 值。

### I/O 中重要的 errno 值

FreeBSD 有一个大的 errno 空间。只有少数在驱动 I/O 路径中频繁出现；学好它们比浏览整个列表更有用。

`0`：成功。当传输干净完成时返回此值。字节计数隐含在 `uio_resid` 中。

`ENXIO`（"设备未配置"）：操作无法继续，因为设备未处于可用状态。如果 softc 缺失、`is_attached` 为 false 或驱动被告知关闭，从 `d_open`、`d_read` 或 `d_write` 返回此值。这是惯用的"cdev 存在但后备设备不存在"错误。

`EFAULT`（"错误地址"）：用户指针无效。你很少直接返回此值；`uiomove(9)` 在 `copyin`/`copyout` 失败时代表你返回它。通过返回 `uiomove` 产生的任何错误来传播它。

`EINVAL`（"无效参数"）：某些参数无意义。对于读取或写入，这通常是越界偏移（如果你的驱动遵守偏移）或格式错误的请求。避免将其作为通用错误使用。

`EAGAIN`（"资源暂时不可用"）：操作会阻塞，但设置了 `O_NONBLOCK`。对于没有数据的 `d_read`，这是非阻塞模式下的正确答案。对于没有空间的 `d_write`，同理。我们将在阶段 3 处理它。

`EINTR`（"中断的系统调用"）：线程在驱动内部阻塞时传递了信号。如果睡眠被信号中断，你的 `d_read` 可能返回 `EINTR`。然后内核要么透明地重试系统调用（取决于 `SA_RESTART` 标志），要么以 `errno = EINTR` 返回用户态。我们将在第十章看到 `EINTR` 处理；第九章不阻塞，因此不产生 `EINTR`。

`EIO`（"输入/输出错误"）：硬件错误的通用错误。当你的驱动与真实硬件通信且硬件报告故障时使用。在 `myfirst` 中很少见，因为它没有硬件。

`ENOSPC`（"设备上没有剩余空间"）：驱动的缓冲区已满，无法接受更多数据。写入时没有空间的正确响应。阶段 3 返回此值。

`EPIPE`（"断开的管道"）：当对端关闭时管道类驱动使用。与 `myfirst` 无关。

`ERANGE`、`EOVERFLOW`、`EMSGSIZE`：字符驱动中不太常见；当内核或驱动想要说"你请求的数字超出范围"时出现。我们不会在本章使用它们。

### 读取中的文件结束

按照约定，返回零字节的读取（因为 `uiomove` 没有移动任何内容且你的处理程序返回零）被调用者解释为文件结束。Shell、`cat`、`head`、`tail`、`dd` 和大多数其他基本系统工具都依赖这个约定。

对你的 `d_read` 的含义：当你的驱动没有更多内容可以提供时，返回零。不要返回 errno。`uio_resid` 应该仍然具有原始值，因为没有移动任何字节。

在 `myfirst` 的阶段 1 和阶段 2 中，EOF 发生在每次描述符的读取偏移到达缓冲区长度时。`uiomove_frombuf` 在这种情况下自然返回零，所以我们不需要特殊的代码路径。

在阶段 3 中，`d_read` 排空 `d_write` 已追加的缓冲区，EOF 行为更微妙："现在没有数据"与"永远不会有更多数据"不同。我们将把"现在没有数据"报告为零字节读取。一个深思熟虑的用户程序可能会将其解释为 EOF 并停止；一个不够深思熟虑的程序会循环并再次调用 `read(2)`。第十章引入了正确的阻塞读取或 `poll(2)` 策略，允许用户程序在不旋转的情况下等待更多数据。

### 零长度读取和写入

零长度请求（`read(fd, buf, 0)` 或 `write(fd, buf, 0)`）是合法的。它意味着"不做任何事，但告诉我你是否可以做某事"。内核为你处理大部分分派：如果 `uio_resid` 在条目处为零，任何 `uiomove` 调用都是无操作，你的处理程序返回零。调用者看到零字节传输且没有错误。

两个微妙之处。首先，不要将 `uio_resid == 0` 视为错误条件。它不是。它是一个合法的请求。其次，不要假设 `uio_resid == 0` 意味着文件结束；它只是意味着调用者请求了零字节。EOF 是关于驱动耗尽数据，而不是关于调用者请求没有数据。

### 短传输

短读取是返回少于请求字节数的读取。短写入是消耗少于提供字节数的写入。两者在 UNIX I/O 中都是合法且预期的；编写良好的用户程序通过循环处理它们。

你的驱动是传输多少的权威决定者。`uiomove` 系列函数在单次调用中最多传输 `MIN(user_request, driver_offer)` 字节。如果你的代码调用 `uiomove(buf, 128, uio)` 而用户请求了 1024，内核传输 128 并在 `uio_resid` 中留下 896。调用者从 `read(2)` 看到返回 128 字节。

不在短 I/O 上循环的行为不良的用户程序会丢失字节。那不是你驱动的问题；UNIX 自 1971 年以来就是这样。行为良好的驱动是返回诚实字节计数（通过 `uio_resid`）和可预测的 errno 值的驱动，即使在部分传输发生时也是如此。

### 处理来自 uiomove 的 EFAULT

当 `uiomove(9)` 返回非零错误时，最常见的值是 `EFAULT`。当你看到它时，内核已经：

- 在复制周围安装了故障处理程序。
- 观察到故障。
- 展开了部分复制。
- 将 `EFAULT` 返回给 `uiomove` 的调用者。

你的处理程序有两个选项来响应：

1. **传播错误**。从 `d_read` / `d_write` 返回 `EFAULT`（或任何返回的 errno）。这是最简单且几乎总是正确的。
2. **调整驱动状态并返回成功**。如果在故障之前移动了一些字节，`uio_resid` 可能已经减少了。内核将向用户空间报告该部分成功。你可能想要更新反映传输进行到多远的任何驱动侧计数器。

在实践中，选项 1 是通用答案，除非你有特定理由做更多。选项 2 增加了很少值得的复杂性。

### 针对用户输入的防御性编程

用户写入你设备的每个字节都是不受信任的。这听起来很戏剧性；这也是字面上的事实。将用户写入解析为结构并解引用该结构中指针的内核模块是一个具有简单任意内存写入漏洞的内核模块。

经验法则：**将缓冲区中的字节视为任意数据，而不是类型化结构，除非你有意选择了一个在每个边界验证的线格式**。对于 `myfirst` 这很容易，因为我们从不解释字节；它们是载荷。对于暴露结构化写入接口的驱动（例如，让用户通过写入配置行为的驱动），防御性路径是：

- 根据期望的消息大小验证写入的长度。
- 将字节复制到内核空间结构（而不是用户指针）。
- 在对其操作之前验证该结构的每个字段。
- 不要在驱动内存储用户指针以供后续使用。

这些规则比听起来更容易遵循，但它们很容易在没有注意到的情况下被打破。第25章在我们查看 `ioctl` 设计时重新审视它们。目前，标准很低：你的 `myfirst` 驱动应该通过 `uiomove` 复制字节，而不是解释它们。

### 记录错误与静默失败

当处理程序返回 errno 时，错误作为失败系统调用的 `errno` 值传播到用户空间。大多数用户程序会在那里看到并报告它。有些会吞掉它。

对于驱动开发，将重要错误记录到 `dmesg` 也有帮助。`device_printf(9)` 是正确的工具，因为它用 Newbus 设备名标记每一行，这样你就可以知道哪个实例产生了消息。阶段 3 的示例：

```c
if (avail == 0) {
        mtx_unlock(&sc->mtx);
        if (bootverbose)
                device_printf(sc->dev, "write rejected: buffer full\n");
        return (ENOSPC);
}
```

`if (bootverbose)` 保护是 FreeBSD 中冗长日志的常见惯用法：它只在内核使用 `-v` 标志启动或 `bootverbose` sysctl 被设置时打印，这保持生产日志安静，同时仍然给开发者提供查看细节的方式。

不要在每次调用时记录每个错误；那会产生日志垃圾，使真正的问题更难找到。记录条件的首次出现，或定期记录，或仅在 `bootverbose` 下记录。选择取决于驱动。对于 `myfirst`，每次转换（缓冲区空、缓冲区满）一条日志就够了。

### 可预测性和用户友好性

编写驱动的初学者通常专注于使正常路径快速。更有经验的驱动作者专注于使错误路径可预测。区别在于：当操作员运行你的驱动并出现故障时，errno 值、日志消息和用户空间反应需要组成一个清晰的故事。如果 `read(2)` 返回 `-1` 且 `errno = EIO` 而日志是静默的，操作员无从下手。如果日志说 "myfirst0: read failed, device detached" 并且用户得到 `ENXIO`，故事不言自明。

以此为目标准。返回正确的 errno。记录一次底层原因。使部分传输诚实。永远不要静默丢弃数据。

### 约定简表

| Situation in d_read                             | Return        |
|-------------------------------------------------|---------------|
| No data to deliver, more might arrive later     | `0` with `uio_resid` unchanged |
| No data, never will be any more (EOF)           | `0` with `uio_resid` unchanged |
| Some data delivered, some not                   | `0` with `uio_resid` reflecting remainder |
| Full delivery                                   | `0` with `uio_resid = 0` |
| User pointer invalid                            | `EFAULT` (from `uiomove`) |
| Device not ready / detaching                    | `ENXIO` |
| Non-blocking, would block                       | `EAGAIN` |
| Hardware error                                  | `EIO` |

| Situation in d_write                            | Return        |
|-------------------------------------------------|---------------|
| Full acceptance                                 | `0` with `uio_resid = 0` |
| Partial acceptance                              | `0` with `uio_resid` reflecting remainder |
| No room, would block                            | `EAGAIN` (non-blocking) or sleep (blocking) |
| No room, permanent                              | `ENOSPC` |
| Invalid pointer                                 | `EFAULT` (from `uiomove`) |
| Device not ready                                | `ENXIO` |
| Hardware error                                  | `EIO` |

两张表都是有意的简短。大多数驱动总共只使用四五个 errno 值。错误故事越干净，你的驱动就越好用。



## 演进你的驱动：三个阶段

理论就绪后，我们转向代码。本节通过 `myfirst` 的三个阶段进行讲解，每个阶段都很小，每个都是一个完整的驱动，可以加载、运行并演示特定的 I/O 模式。

阶段被设计为相互构建：

- **阶段 1** 添加一个提供固定内核空间消息的读取路径。这是最简单的 `myfirst_read`。
- **阶段 2** 添加一个将用户数据存入内核缓冲区的写入路径，以及一个从同一缓冲区读取的读取路径。缓冲区在 attach 时确定大小，不会环绕。
- **阶段 3** 将阶段 2 变为先进先出缓冲区，使写入追加、读取排空，驱动可以提供连续（但有限）的流。

所有三个阶段都从第八章阶段 2 的源码开始。构建系统（`Makefile`）不变。`attach` 和 `detach` 处理程序在每个阶段略微增长。`cdevsw` 的形状、Newbus 方法、每次打开 `fh` 的管道和 sysctl 树保持不变。你将把大部分时间花在 `d_read` 和 `d_write` 上。

### 阶段 1：静态消息读取器

阶段 1 驱动在内核内存中保存固定消息并将其提供给读取者。`d_read` 使用 `uiomove_frombuf(9)` 传递消息。`d_write` 保持桩函数：它返回成功但不消耗任何字节。此阶段是从第八章桩函数到真正读取者的桥梁；它在尽可能小的上下文中介绍 `uiomove_frombuf`。

向 softc 添加一对字段来保存消息及其长度，并向 `fh` 添加每次描述符偏移：

```c
struct myfirst_softc {
        /* ...existing Chapter 8 fields... */

        const char *message;
        size_t      message_len;
};

struct myfirst_fh {
        struct myfirst_softc *sc;
        uint64_t              reads;
        uint64_t              writes;
        off_t                 read_off;
};
```

在 `myfirst_attach` 中，初始化消息：

```c
static const char myfirst_message[] =
    "Hello from myfirst.\n"
    "This is your first real read path.\n"
    "Chapter 9, Stage 1.\n";

sc->message = myfirst_message;
sc->message_len = sizeof(myfirst_message) - 1;
```

注意 `- 1`：我们不想将终止 NUL 字节提供给用户空间。文本文件末尾不带 NUL，行为类似的设备也不应该带。

新的 `myfirst_read`：

```c
static int
myfirst_read(struct cdev *dev, struct uio *uio, int ioflag)
{
        struct myfirst_softc *sc = dev->si_drv1;
        struct myfirst_fh *fh;
        off_t before;
        int error;

        error = devfs_get_cdevpriv((void **)&fh);
        if (error != 0)
                return (error);

        if (sc == NULL || !sc->is_attached)
                return (ENXIO);

        before = uio->uio_offset;
        error = uiomove_frombuf(__DECONST(void *, sc->message),
            sc->message_len, uio);
        if (error == 0)
                fh->reads += (uio->uio_offset - before);
        fh->read_off = uio->uio_offset;
        return (error);
}
```

有两个细节值得停下来思考。

首先，`uio->uio_offset` 是流中的每次描述符位置。内核在调用之间维护它，随着 `uiomove_frombuf` 移动字节而前进。新打开描述符上的第一次 `read(2)` 从偏移零开始；每次后续 `read(2)` 从上一次结束的地方开始。当偏移达到 `sc->message_len` 时，`uiomove_frombuf` 返回零而不移动任何字节，调用者看到 EOF。

其次，`before` 在条目处捕获 `uio->uio_offset`，以便我们可以计算移动了多少字节。`uiomove_frombuf` 返回后，差异就是传输大小，我们将其添加到每次描述符 `reads` 计数器。这是第八章中 `fh->reads` 字段最终发挥作用的地方。

The `__DECONST` cast is a FreeBSD idiom for casting away `const`. `uiomove_frombuf` takes a non-`const` `void *` because it is prepared to move in either direction, but in this context we know the direction is kernel-to-user (a read), so we know the kernel buffer will not be modified. Stripping the `const` here is safe; using a plain `(void *)` cast would work as well but is less self-documenting.

`myfirst_write` 在阶段 1 保持第八章留下的样子：

```c
static int
myfirst_write(struct cdev *dev, struct uio *uio, int ioflag)
{
        struct myfirst_fh *fh;
        int error;

        error = devfs_get_cdevpriv((void **)&fh);
        if (error != 0)
                return (error);

        (void)fh;
        uio->uio_resid = 0;
        return (0);
}
```

写入被接受并丢弃，即 `/dev/null` 形状。阶段 2 将改变这一点。

构建并加载。从用户态快速冒烟测试：

```sh
% cat /dev/myfirst/0
Hello from myfirst.
This is your first real read path.
Chapter 9, Stage 1.
%
```

从同一描述符的第二次读取返回 EOF，因为偏移已经超过消息末尾：

```sh
% cat /dev/myfirst/0 /dev/myfirst/0
Hello from myfirst.
This is your first real read path.
Chapter 9, Stage 1.
Hello from myfirst.
This is your first real read path.
Chapter 9, Stage 1.
```

等等：`cat` 读取消息两次。那是因为 `cat` 两次打开文件（每个参数一次），每次打开获得一个带有自己 `uio_offset` 的新描述符。如果你想验证两次打开确实看到独立的偏移，从一个小的 C 程序打开设备并从同一描述符多次读取：

```c
#include <fcntl.h>
#include <unistd.h>
#include <stdio.h>

int
main(void)
{
        int fd = open("/dev/myfirst/0", O_RDONLY);
        if (fd < 0) { perror("open"); return 1; }
        char buf[64];
        ssize_t n;
        while ((n = read(fd, buf, sizeof(buf))) > 0) {
                fwrite(buf, 1, n, stdout);
        }
        close(fd);
        return 0;
}
```

第一次 `read(2)` 返回消息；第二次返回零（EOF）；程序退出。这确认了 `uio_offset` 是每次描述符维护的。

阶段 1 故意很短。它介绍了三个想法（`uiomove_frombuf` 辅助函数、每次描述符偏移、`__DECONST` 惯用法）而不会让读者负担过重。本章其余部分在此基础上构建。

### 阶段 2：写一次/读多次缓冲区

阶段 2 扩展驱动以接受写入。驱动在 attach 时分配内核缓冲区，写入存入其中，读取从中传递。没有环绕：一旦缓冲区填满，后续写入返回 `ENOSPC`。读取看到迄今为止写入的内容，从自己的每次描述符偏移开始。

`myfirst_softc` 的形状增长了几个字段：

```c
struct myfirst_softc {
        /* ...existing Chapter 8 fields... */

        char    *buf;
        size_t   buflen;
        size_t   bufused;

        uint64_t bytes_read;
        uint64_t bytes_written;
};
```

`buf` 是 `malloc(9)` 返回的指针。`buflen` 是它的大小，为简单起见是编译时常量；你以后可以使其可调。`bufused` 是高水位标记：迄今为止写入的字节数。

两个新的 sysctl 节点用于可观测性：

```c
SYSCTL_ADD_U64(&sc->sysctl_ctx, SYSCTL_CHILDREN(sc->sysctl_tree),
    OID_AUTO, "bytes_written", CTLFLAG_RD,
    &sc->bytes_written, 0, "Total bytes written into the buffer");

SYSCTL_ADD_UINT(&sc->sysctl_ctx, SYSCTL_CHILDREN(sc->sysctl_tree),
    OID_AUTO, "bufused", CTLFLAG_RD,
    &sc->bufused, 0, "Current byte count in the buffer");
```

`bufused` is a `size_t`, and the sysctl macro for unsigned integer is `SYSCTL_ADD_UINT` on 32-bit platforms or `SYSCTL_ADD_U64` on 64-bit platforms. Since this driver targets FreeBSD 14.3 on amd64 in the typical lab, `SYSCTL_ADD_UINT` is fine; the field will be presented as an `unsigned int` even though the internal type is `size_t`. If you target arm64 or another 64-bit platform, use `SYSCTL_ADD_U64` and cast accordingly.

在 `attach` 中分配缓冲区：

```c
#define MYFIRST_BUFSIZE 4096

sc->buflen = MYFIRST_BUFSIZE;
sc->buf = malloc(sc->buflen, M_DEVBUF, M_WAITOK | M_ZERO);
if (sc->buf == NULL) {
        error = ENOMEM;
        goto fail_mtx;
}
sc->bufused = 0;
```

在 `detach` 中释放它：

```c
if (sc->buf != NULL) {
        free(sc->buf, M_DEVBUF);
        sc->buf = NULL;
}
```

Adjust the error-unwind in `attach` to include the buffer free:

```c
fail_dev:
        if (sc->cdev_alias != NULL) {
                destroy_dev(sc->cdev_alias);
                sc->cdev_alias = NULL;
        }
        destroy_dev(sc->cdev);
        sysctl_ctx_free(&sc->sysctl_ctx);
        free(sc->buf, M_DEVBUF);
        sc->buf = NULL;
fail_mtx:
        mtx_destroy(&sc->mtx);
        sc->is_attached = 0;
        return (error);
```

Now the read handler:

```c
static int
myfirst_read(struct cdev *dev, struct uio *uio, int ioflag)
{
        struct myfirst_softc *sc = dev->si_drv1;
        struct myfirst_fh *fh;
        off_t before;
        size_t have;
        int error;

        error = devfs_get_cdevpriv((void **)&fh);
        if (error != 0)
                return (error);
        if (sc == NULL || !sc->is_attached)
                return (ENXIO);

        mtx_lock(&sc->mtx);
        have = sc->bufused;
        before = uio->uio_offset;
        error = uiomove_frombuf(sc->buf, have, uio);
        if (error == 0) {
                sc->bytes_read += (uio->uio_offset - before);
                fh->reads += (uio->uio_offset - before);
        }
        fh->read_off = uio->uio_offset;
        mtx_unlock(&sc->mtx);
        return (error);
}
```

读取处理程序获取互斥锁以一致地读取 `bufused`，然后以当前高水位标记作为有效缓冲区大小调用 `uiomove_frombuf`。在任何写入之前运行的读取者将看到 `have = 0`，`uiomove_frombuf` 将返回零，调用者将其解释为 EOF。在一些写入之后运行的读取者将看到当前 `bufused` 并接收最多那么多字节。

The write handler:

```c
static int
myfirst_write(struct cdev *dev, struct uio *uio, int ioflag)
{
        struct myfirst_softc *sc = dev->si_drv1;
        struct myfirst_fh *fh;
        size_t avail, towrite;
        int error;

        error = devfs_get_cdevpriv((void **)&fh);
        if (error != 0)
                return (error);
        if (sc == NULL || !sc->is_attached)
                return (ENXIO);

        mtx_lock(&sc->mtx);
        avail = sc->buflen - sc->bufused;
        if (avail == 0) {
                mtx_unlock(&sc->mtx);
                return (ENOSPC);
        }
        towrite = MIN((size_t)uio->uio_resid, avail);
        error = uiomove(sc->buf + sc->bufused, towrite, uio);
        if (error == 0) {
                sc->bufused += towrite;
                sc->bytes_written += towrite;
                fh->writes += towrite;
        }
        mtx_unlock(&sc->mtx);
        return (error);
}
```

注意限制：`towrite = MIN(uio->uio_resid, avail)`。如果用户请求写入 8 KiB 而我们有 512 字节空间，我们接受 512 字节并让内核向用户空间报告 512 的短写入。行为良好的调用者会用剩余字节循环；行为不太好的调用者会丢失多余部分。那是调用者的责任；驱动已经诚实地完成了它的部分。

Smoke-test from userland:

```sh
% sudo kldload ./myfirst.ko
% echo "hello" | sudo tee /dev/myfirst/0 > /dev/null
% cat /dev/myfirst/0
hello
% echo "more" | sudo tee -a /dev/myfirst/0 > /dev/null
% cat /dev/myfirst/0
hello
more
% sysctl dev.myfirst.0.stats.bufused
dev.myfirst.0.stats.bufused: 11
%
```

The buffer grew by 6 bytes for `"hello\n"`, then by 5 more for `"more\n"`, yielding 11 bytes. `cat` reads all 11 bytes back. A second `cat` from a fresh open starts at offset zero and reads them again.

What happens if we write more than the buffer can hold?

```sh
% dd if=/dev/zero bs=1024 count=8 | sudo tee /dev/myfirst/0 > /dev/null
dd: stdout: No space left on device
tee: /dev/myfirst/0: No space left on device
8+0 records in
7+0 records out
```

`dd` wrote 7 blocks of 1024 bytes before the 8th one failed. `tee` reports the error. The driver accepted up to its limit and then returned `ENOSPC` cleanly. The kernel carried the errno value back to user space.

### 阶段 3：先进先出回显驱动

阶段 3 将缓冲区变为 FIFO。写入追加到尾部。读取从头部排空。当缓冲区为空时，读取返回零字节（空时 EOF）。当缓冲区满时，写入返回 `ENOSPC`。

缓冲区保持线性：没有环绕。在排空所有数据的读取之后，`bufused` 为零，下一次写入再次从 `sc->buf` 中的偏移零开始。这使记录保持最少，并将阶段集中在 I/O 方向变化而不是环形缓冲区机制上。

The softc gains one more field:

```c
struct myfirst_softc {
        /* ...existing fields... */

        size_t  bufhead;   /* index of next byte to read */
        size_t  bufused;   /* bytes in the buffer, from bufhead onward */

        /* ...remaining fields... */
};
```

`bufhead` 是仍然要读取的第一个字节的偏移。`bufused` 是从 `bufhead` 开始的有效字节数。不变量 `bufhead + bufused <= buflen` 总是成立。

Reset both in `attach`:

```c
sc->bufhead = 0;
sc->bufused = 0;
```

New read handler:

```c
static int
myfirst_read(struct cdev *dev, struct uio *uio, int ioflag)
{
        struct myfirst_softc *sc = dev->si_drv1;
        struct myfirst_fh *fh;
        size_t toread;
        int error;

        error = devfs_get_cdevpriv((void **)&fh);
        if (error != 0)
                return (error);
        if (sc == NULL || !sc->is_attached)
                return (ENXIO);

        mtx_lock(&sc->mtx);
        if (sc->bufused == 0) {
                mtx_unlock(&sc->mtx);
                return (0); /* EOF-on-empty */
        }
        toread = MIN((size_t)uio->uio_resid, sc->bufused);
        error = uiomove(sc->buf + sc->bufhead, toread, uio);
        if (error == 0) {
                sc->bufhead += toread;
                sc->bufused -= toread;
                sc->bytes_read += toread;
                fh->reads += toread;
                if (sc->bufused == 0)
                        sc->bufhead = 0;
        }
        mtx_unlock(&sc->mtx);
        return (error);
}
```

有一些细节与阶段 2 不同。读取不再遵守 `uio->uio_offset`；对于每个描述符看到相同流且流在消耗时消失的 FIFO，每次描述符偏移没有意义。当 `bufused` 达到零时，我们将 `bufhead` 重置为零，这保持下一次写入在缓冲区开头对齐，避免将数据推向末尾。

这种"空时折叠"技巧不是环形缓冲区，但对于教学 FIFO 足够接近。额外的重新对齐步骤是 `O(1)`；几乎不花费任何代价。

New write handler (mostly unchanged from Stage 2, but note where it appends):

```c
static int
myfirst_write(struct cdev *dev, struct uio *uio, int ioflag)
{
        struct myfirst_softc *sc = dev->si_drv1;
        struct myfirst_fh *fh;
        size_t avail, tail, towrite;
        int error;

        error = devfs_get_cdevpriv((void **)&fh);
        if (error != 0)
                return (error);
        if (sc == NULL || !sc->is_attached)
                return (ENXIO);

        mtx_lock(&sc->mtx);
        tail = sc->bufhead + sc->bufused;
        avail = sc->buflen - tail;
        if (avail == 0) {
                mtx_unlock(&sc->mtx);
                return (ENOSPC);
        }
        towrite = MIN((size_t)uio->uio_resid, avail);
        error = uiomove(sc->buf + tail, towrite, uio);
        if (error == 0) {
                sc->bufused += towrite;
                sc->bytes_written += towrite;
                fh->writes += towrite;
        }
        mtx_unlock(&sc->mtx);
        return (error);
}
```

The write appends at `sc->bufhead + sc->bufused`, not at `sc->bufused` alone, because the valid data slice has moved as reads drained it.

Smoke-test:

```sh
% echo "one" | sudo tee /dev/myfirst/0 > /dev/null
% echo "two" | sudo tee -a /dev/myfirst/0 > /dev/null
% cat /dev/myfirst/0
one
two
% cat /dev/myfirst/0
%
```

After the first `cat`, the buffer is empty. The second `cat` sees no data and exits immediately.

This is the Stage 3 shape. The driver is a small, honest, in-memory FIFO. Users can push bytes in, pull them out, and observe the counters from sysctl. That is real I/O, and it is the waypoint Chapter 10 builds from.



## 从用户空间到你的处理程序追踪 read(2)

Before you start working through the labs, take a step-by-step look at exactly what happens when a user program calls `read(2)` on one of your nodes. Understanding this path is one of those things that changes how you read driver code. Every handler you see in the tree is sitting at the bottom of the call chain described below; once you recognise the chain, every handler starts to look familiar.

### 步骤 1：用户程序调用 read(2)

The C library's `read` wrapper is a thin translation of the call into a system-call trap: it places the file descriptor, the buffer pointer, and the count into the appropriate registers and executes the trap instruction for the current architecture. Control transfers to the kernel.

This part has nothing to do with drivers. It is the same for every syscall. What matters is that the kernel is now executing on behalf of the user process, in the kernel's address space, with the user's registers saved and the process's credentials visible through `curthread->td_ucred`.

### 步骤 2：内核查找文件描述符

The kernel calls `sys_read(2)` (in `/usr/src/sys/kern/sys_generic.c`), which validates the arguments, looks up the file descriptor in the calling process's file table, and acquires a reference on the resulting `struct file`.

If the descriptor is not open, the call fails here with `EBADF`. If the descriptor is open but is not readable (for instance, the user opened the device with `O_WRONLY`), the call fails with `EBADF` as well. The driver is not involved; `sys_read` enforces the access mode.

### 步骤 3：通用文件操作向量分派

The `struct file` has a file-type tag (`f_type`) and a file-operations vector (`f_ops`). For a regular file the vector dispatches to the VFS layer; for a socket it dispatches to sockets; for a device opened through devfs, it dispatches to `vn_read`, which in turn calls the vnode operation `VOP_READ` on the vnode behind the file.

This may sound like indirection for its own sake. It is actually how the kernel keeps the rest of the syscall path identical for every kind of file. Drivers do not need to know about this layer; devfs and VFS hand the call to your handler eventually.

### 步骤 4：VFS 调用 devfs

The vnode's filesystem ops point to devfs's implementation of the vnode interface (`devfs_vnops`). `VOP_READ` on a devfs vnode calls `devfs_read_f`, which looks at the cdev behind the vnode, acquires a thread-count reference on it (incrementing `si_threadcount`), and calls `cdevsw->d_read`. That is your function.

Two details from this step carry implications for your driver.

First, **the `si_threadcount` increment is what `destroy_dev(9)` uses to know your handler is active**. When a module unloads and `destroy_dev` runs, it waits until every current invocation of every handler returns. The reference is incremented before your `d_read` is called and released after it returns. The mechanism is why your driver can be safely unloaded while a user is in the middle of `read(2)`.

Second, **the call is synchronous from the VFS layer's point of view**. VFS calls your handler, waits for it to return, and then propagates the result. You do not need to do anything special to participate in this synchronisation; just return from your handler when you are done.

### 步骤 5：你的 d_read 处理程序运行

This is where we have been all chapter. The handler:

- Receives a `struct cdev *dev` (the node being read), a `struct uio *uio` (the I/O description), and an `int ioflag` (flags from the file-table entry).
- Retrieves per-open state via `devfs_get_cdevpriv(9)`.
- Verifies liveness.
- Transfers bytes through `uiomove(9)`.
- Returns zero or an errno.

Nothing about this step should be mysterious by now.

### 步骤 6：内核展开并报告

`devfs_read_f` sees your return value. If zero, it computes the byte count from the decrease in `uio->uio_resid` and returns that count. If non-zero, it converts the errno into the syscall's error return. VFS's `vn_read` passes the result upward to `sys_read`. `sys_read` writes the result into the return-value register.

Control transfers back to user space. The C library's `read` wrapper examines the result: a positive value is returned as the return value of `read(2)`; a negative value sets `errno` and returns `-1`.

The user program sees the integer it expected, and its control flow continues.

### 步骤 7：引用计数展开

On the way out, `devfs_read_f` releases the thread-count reference on the cdev. If `destroy_dev(9)` had been waiting for `si_threadcount` to reach zero, it may now proceed with the tear-down.

This is why the whole chain is structured as carefully as it is. Every reference is paired; every increment has a matching decrement; every piece of state the handler touches is either owned by the handler, owned by the softc, or owned by the per-open `fh`. If any of those invariants breaks, unload becomes unsafe.

### 为什么这个追踪对你重要

Three takeaways.

**The first**: the mechanism above is why your handler does not need to do anything exotic to coexist with module unload. Provided you return from `d_read` in finite time, the kernel will let your driver unload cleanly. This is part of why Chapter 9 keeps all reads non-blocking at the driver level.

**The second**: every layer between `read(2)` and your handler is set up by the kernel before your code runs. The user's buffer is valid (or `uiomove` will report `EFAULT`), the cdev is alive (or devfs would have refused the call), the access mode is compatible with the descriptor (or `sys_read` would have refused), and the process's credentials are the current thread's. You can focus on your driver's job and trust the layers.

**The third**: when you read an unfamiliar driver in the tree and its `d_read` looks weird, you can walk the chain in reverse. Who called this handler? What state did they prepare? What invariants does my handler promise on return? The chain tells you. The answers are usually the same as they are for `myfirst`.

### 镜像：追踪 write(2)

A write follows the same kind of chain, mirrored. A full seven-step breakdown would be mostly a restatement of the read trace with words substituted, so the paragraph below is deliberately compressed.

The user calls `write(fd, buf, 1024)`. The C library traps into the kernel. `sys_write(2)` in `/usr/src/sys/kern/sys_generic.c` validates arguments, looks up the descriptor, and acquires a reference on its `struct file`. The file-ops vector dispatches to `vn_write`, which calls `VOP_WRITE` on the devfs vnode. `devfs_write_f` in `/usr/src/sys/fs/devfs/devfs_vnops.c` acquires the thread-count reference on the cdev, composes the `ioflag` from `fp->f_flag`, and calls `cdevsw->d_write` with the uio describing the caller's buffer.

Your `d_write` handler runs. It retrieves per-open state via `devfs_get_cdevpriv(9)`, checks liveness, takes whatever lock the driver needs around the buffer, clamps the transfer length to whatever space is available, and calls `uiomove(9)` to copy bytes from user space into the kernel buffer. On success, the handler updates its bookkeeping and returns zero. `devfs_write_f` releases the thread-count reference. `vn_write` unwinds through `sys_write`, which computes the byte count from the decrease in `uio_resid` and returns it. The user sees the return value of `write(2)`.

Three things differ from the read chain in substantive ways.

**First, the kernel runs `copyin` inside `uiomove` instead of `copyout`.** Same mechanism, opposite direction. The fault handling is identical: a bad user pointer returns `EFAULT`, a short copy leaves `uio_resid` consistent with whatever did transfer, and the handler just propagates the error code.

**Second, `ioflag` carries `IO_NDELAY` in the same way, but the driver's interpretation is different.** On a read, non-blocking means "return `EAGAIN` if there is no data". On a write, non-blocking means "return `EAGAIN` if there is no space". Symmetric conditions, symmetric errno values.

**Third, the `atime` / `mtime` updates are direction-specific.** `devfs_read_f` stamps `si_atime` if bytes moved; `devfs_write_f` stamps `si_mtime` (and `si_ctime` in some paths) if bytes moved. These are what `stat(2)` on the node reports, and why `ls -lu /dev/myfirst/0` shows different timestamps for reads versus writes. Your driver does not manage these fields; devfs does.

Once you recognise the read and write traces as mirror images, you have internalised most of the character-device dispatch path. Every chapter from here on will add hooks (a `d_poll`, a `d_kqfilter`, a `d_ioctl`, an `mmap` path) that sit on the same chain at slightly different slots. The chain itself stays constant.



## 实用工作流：从 shell 测试你的驱动

The base-system tools are your first and best test harness. This section is a short field guide to using them well on a driver you are developing. None of the commands below are new to you, but using them for driver work has a rhythm worth learning explicitly.

### cat(1)：第一次检查

`cat` reads from its arguments and writes to standard output. For a driver that serves a static message or a drained buffer, `cat` is the fastest way to see what the read path produces:

```sh
% cat /dev/myfirst/0
```

If the output is what you expect, the read path is alive. If it is empty, either your driver has nothing to deliver (check `sysctl dev.myfirst.0.stats.bufused`) or your handler is returning EOF on the first call. If the output is garbled, either your buffer is uninitialised or you are handing out bytes past `bufused`.

`cat` opens its argument once and reads from it until EOF. Every `read(2)` is a separate call into your `d_read`. Use `truss(1)` to see how many calls `cat` makes:

```sh
% truss cat /dev/myfirst/0 2>&1 | grep read
```

The output shows each `read(2)` with its arguments and return value. If you expected one read and see three, that tells you about your buffer sizing; if you expected three reads and see one, your handler delivered all the data in a single call.

### echo(1) 和 printf(1)：简单写入

`echo` is the quickest way to get a known string into your driver's write path:

```sh
% echo "hello" | sudo tee /dev/myfirst/0 > /dev/null
```

Two things to notice. First, `echo` appends a newline by default; the string you sent is six bytes, not five. Use `echo -n` to suppress the newline when that matters. Second, the `tee` invocation is there to solve a permission problem: shell redirection (`>`) runs with the user's privileges, so a `sudo echo > /dev/myfirst/0` fails to open the node. Piping through `tee`, which runs under `sudo`, sidesteps that.

`printf` gives you more control:

```sh
% printf 'abc' | sudo tee /dev/myfirst/0 > /dev/null
```

Three bytes, no newline. Use `printf '\x41\x42\x43'` for binary patterns.

### dd(1)：精确工具

For any test that needs a specific byte count or a specific block size, `dd` is the right tool. `dd` is also one of the only base-system tools that reports short reads and short writes in its summary, which makes it uniquely useful for testing driver behaviour:

```sh
% sudo dd if=/dev/urandom of=/dev/myfirst/0 bs=128 count=4
4+0 records in
4+0 records out
512 bytes transferred in 0.001234 secs (415000 bytes/sec)
```

The `X+Y records in` / `X+Y records out` counters have a precise meaning: `X` is the number of full-block transfers, `Y` is the number of short transfers. A line reading `0+4 records out` means every block was accepted only partially. That is a driver telling you something.

`dd` also lets you read with a known block size:

```sh
% sudo dd if=/dev/myfirst/0 of=/tmp/dump bs=64 count=1
```

This issues exactly one `read(2)` for 64 bytes. Your handler sees `uio_resid = 64`; you respond with whatever you have; the result is what `dd` writes to `/tmp/dump`.

The `iflag=fullblock` flag tells `dd` to loop on short reads until it has filled the requested block. Useful when you want to soak all of the driver's output without losing bytes to the short-read default.

### od(1) 和 hexdump(1)：字节级检查

For driver testing, `od` and `hexdump` let you see the exact bytes your driver emitted:

```sh
% sudo dd if=/dev/myfirst/0 bs=32 count=1 | od -An -tx1z
  68 65 6c 6c 6f 0a                                 >hello.<
```

The `-An` flag suppresses address printing. `-tx1z` shows bytes in hex and ASCII. If the expected output is text, you see it on the right; if it is binary, you see the hex on the left.

These tools become essential when a read produces unexpected bytes. "It looks weird" and "I can see every byte in hex" are very different debugging states.

### sysctl(8) 和 dmesg(8)：内核的声音

Your driver publishes counters through `sysctl` and lifecycle events through `dmesg`. Both are worth checking during every test:

```sh
% sysctl dev.myfirst.0
% dmesg | tail -20
```

The sysctl output is your view into the driver's state right now. `dmesg` is your view into the driver's history since boot (or since the ring buffer wrapped).

A useful habit: after every test, run both. If the numbers do not match your expectation, you have narrowed down the bug quickly.

### fstat(1)：谁打开了描述符？

When your driver refuses to unload ("module busy"), the question is "who has `/dev/myfirst/0` open right now?". `fstat(1)` answers it:

```sh
% fstat -p $(pgrep cat) /dev/myfirst/0
USER     CMD          PID   FD MOUNT      INUM MODE         SZ|DV R/W NAME
ebrandi  cat          1234    3 /dev         0 crw-rw----  myfirst/0  r /dev/myfirst/0
```

Alternatively, `fuser(8)`:

```sh
% sudo fuser /dev/myfirst/0
/dev/myfirst/0:         1234
```

Either tool names the processes holding the descriptor. Kill the culprit (carefully; do not kill anything you did not start) and the module will unload.

### truss(1) 和 ktrace(1)：观察系统调用

For a user program whose interaction with your driver you want to inspect, `truss` shows every syscall and its return value:

```sh
% truss ./rw_myfirst
open("/dev/myfirst/0",O_WRONLY,0666)             = 3 (0x3)
write(3,"round-trip test payload\n",24)          = 24 (0x18)
close(3)                                         = 0 (0x0)
...
```

`ktrace` records to a file that `kdump` prints later; it is the right tool when you want to capture a trace of a long-running program.

These two tools are not driver-specific, but they are how you confirm from the outside that your driver is producing the results a user program will see.

### 建议的测试节奏

For each stage of the chapter, try this loop:

1. Build and load.
2. `cat` to produce initial output, confirm by eye.
3. `sysctl dev.myfirst.0` to see counters match.
4. `dmesg | tail` to see lifecycle events.
5. Write something with `echo` or `dd`.
6. Read it back.
7. Repeat with a larger size, a boundary size, and a pathological size.
8. Unload.

After a couple of iterations this becomes automatic and fast. It is the kind of rhythm that turns driver development from a slog into a routine.

### 一个具体的 truss 演示

Running a userland program under `truss(1)` is one of the fastest ways to see exactly what syscalls it makes to your driver and what return values the kernel produces. Here is a typical session with the Stage 3 driver loaded and empty:

```sh
% truss ./rw_myfirst rt 2>&1
open("/dev/myfirst/0",O_WRONLY,00)               = 3 (0x3)
write(3,"round-trip test payload, 24b\n",29)     = 29 (0x1d)
close(3)                                         = 0 (0x0)
open("/dev/myfirst/0",O_RDONLY,00)               = 3 (0x3)
read(3,"round-trip test payload, 24b\n",255) = 29 (0x1d)
close(3)                                         = 0 (0x0)
exit(0x0)
```

A few things are worth pausing on. Each line shows a single syscall, its arguments, and its return value in both decimal and hex. The `write` call received 29 bytes and the driver accepted all 29 (the return value matches the request length). The `read` call received a buffer of 255 bytes of room and the driver produced 29 bytes of content; a short read, which the user program explicitly accepts. Both `open` calls returned 3, because file descriptors 0, 1, and 2 are standard streams and the first free descriptor is 3.

If you force a short write by limiting the driver, `truss` will show it plainly:

```sh
% truss ./write_big 2>&1 | head
open("/dev/myfirst/0",O_WRONLY,00)               = 3 (0x3)
write(3,"<8192 bytes of data>",8192)             = 4096 (0x1000)
write(3,"<4096 bytes of data>",4096)             ERR#28 'No space left on device'
close(3)                                         = 0 (0x0)
```

The first write requested 8192 bytes and was accepted for 4096. The second write had nothing to say because the buffer is full; the driver returned `ENOSPC`, which `truss` rendered as `ERR#28 'No space left on device'`. This is the view from the user side; your driver side was returning zero (with `uio_resid` decremented to 4096) for the first call and `ENOSPC` for the second. Comparing what `truss` sees against what your `device_printf` says is an excellent way to catch mismatches between the driver's intent and the kernel's reporting.

`truss -f` follows forks, which is useful when your test harness spawns worker processes. `truss -d` prefixes each line with a relative timestamp; useful for reasoning about latency between calls. Both flags are small investments; the rewards add up quickly when you start running multi-process stress tests.

### 关于 ktrace 的简要说明

`ktrace(1)` is `truss`'s bigger sibling. It records a binary trace to a file (`ktrace.out` by default) which you then format with `kdump(1)`. It is the right tool when:

- The test run is long and you do not want to watch output live.
- You want to capture detail that is too fine-grained for `truss` (syscall timing, signal delivery, namei lookups).
- You want to replay a trace later, perhaps on a different machine.

A typical session:

```sh
% sudo ktrace -i ./stress_rw -s 5
% sudo kdump | head -40
  2345 stress_rw CALL  open(0x800123456,0x1<O_WRONLY>)
  2345 stress_rw NAMI  "/dev/myfirst/0"
  2345 stress_rw RET   open 3
  2345 stress_rw CALL  write(0x3,0x800123500,0x40)
  2345 stress_rw RET   write 64
  2345 stress_rw CALL  write(0x3,0x800123500,0x40)
  2345 stress_rw RET   write 64
...
```

For Chapter 9 the difference between `truss` and `ktrace` is small. Use `truss` as the default; reach for `ktrace` when you need more detail or a recorded trace.

### 用 vmstat -m 观察内核内存

Your driver allocates kernel memory through `malloc(9)` with the `M_DEVBUF` type. FreeBSD's `vmstat -m` reveals how many allocations are active in each type bucket. Run it while your driver is loaded and idle, then again while it has a buffer allocated, and the increase will be visible in the `devbuf` row:

```sh
% vmstat -m | head -1
         Type InUse MemUse HighUse Requests  Size(s)
% vmstat -m | grep devbuf
       devbuf   415   4120K       -    39852  16,32,64,128,256,512,1024,2048,...
```

The `InUse` column is the current count of live allocations of this type. `MemUse` is the total size currently in use. `HighUse` is the all-time high-water mark since boot. `Requests` is the lifetime count of `malloc` calls that selected this type.

Load the Stage 2 driver. `InUse` goes up by one (the 4096-byte buffer), `MemUse` goes up by approximately 4 KiB, and `Requests` increments. Unload. `InUse` goes down by one; `MemUse` goes down by the 4 KiB. If it does not, you have a memory leak, and `vmstat -m` just told you so.

This is the second observability channel worth adding to your test rhythm. `sysctl` shows driver-owned counters. `dmesg` shows driver-owned log lines. `vmstat -m` shows kernel-owned allocation counts, and it catches a class of bug (forgot to free) that the first two cannot see.

For a driver that declares its own malloc type via `MALLOC_DEFINE(M_MYFIRST, "myfirst", ...)`, `vmstat -m | grep myfirst` is even better: it isolates your driver's allocations from the generic `devbuf` pool. `myfirst` stays with `M_DEVBUF` throughout this chapter for simplicity, but upgrading to a dedicated type is a small change you may want to make before shipping a driver outside the book's lab environment.



## 可观测性：让你的驱动可读

A driver that does the right thing is worth more if you can confirm, from outside the kernel, that it is doing the right thing. This section is a short meditation on the observability choices this chapter has been making, and why.

### 三个接口：sysctl、dmesg、用户态

Your driver presents three surfaces to the operator:

- **sysctl** for live counters: point-in-time values the operator can poll.
- **dmesg (device_printf)** for lifecycle events: open, close, errors, transitions.
- **/dev** nodes for the data path: the actual bytes.

Each has a distinct role. sysctl tells the operator *what is true right now*. dmesg tells the operator *what changed recently*. `/dev` is the thing the operator is actually using.

A well-observed driver uses all three, deliberately. A minimally-observed driver uses only the third, and debugging it requires either a debugger or a lot of guessing.

### Sysctl：计数器与状态

`myfirst` exposes counters through the sysctl tree under `dev.myfirst.0.stats`:

- `attach_ticks`: a point-in-time value (when the driver attached).
- `open_count`: a monotonically-increasing counter (lifetime opens).
- `active_fhs`: a live count (current descriptors).
- `bytes_read`, `bytes_written`: monotonically-increasing counters.
- `bufused`: a live value (current buffer occupancy).

Monotonically-increasing counters are easier to reason about than live values, because their rate of change is informative even when the absolute value is not. An operator who sees `bytes_read` increasing at 1 MB/s has learned something even if 1 MB/s is meaningless out of context.

Live values are essential when the state matters for decisions (`active_fhs > 0` means unload will fail). Choose monotonically-increasing counters first, live values when you need them.

### dmesg：值得查看的事件

`device_printf(9)` writes to the kernel message buffer, which `dmesg` shows. Every line is worth seeing exactly once: use dmesg for events, not for continuous status.

The events `myfirst` logs:

- Attach (once per instance).
- Open (once per open).
- Destructor (once per descriptor close).
- Detach (once per instance).

That is four lines per instance per load/unload cycle, plus two lines per open/close pair. Comfortable.

What we do not log:

- Every `read` or `write` call. That would flood dmesg in any real workload.
- Every sysctl read. Those are passive.
- Every successful transfer. The sysctl counters carry that information, and they carry it more compactly.

If a driver needs to log something that happens many times a second, the usual answer is to guard the logging with `if (bootverbose)`, so it is silent on production systems but available to developers who boot with `boot -v`. For `myfirst` we do not need even that.

### 过度日志记录的陷阱

A driver that logs every operation is a driver that hides its important events in a sea of noise. If your dmesg shows ten thousand lines of `read returned 0 bytes`, the line that says `buffer full, returning ENOSPC` is invisible.

Keep logs sparse. Log transitions, not states. Log once per instance, not once per call. When in doubt, silence.

### 你稍后要添加的计数器

Chapters 10 and beyond will extend the counter tree with:

- `reads_blocked`, `writes_blocked`: count of calls that had to sleep (Chapter 10).
- `poll_waiters`: count of active `poll(2)` subscribers (Chapter 10).
- `drain_waits`, `overrun_events`: ring-buffer diagnostics (Chapter 10).

Each one is one more thing an operator can look at to understand what the driver is doing. The pattern is the same: expose the counters, keep the mechanism silent, let the operator decide when to inspect.

### 你的驱动在轻负载下的样子

A concrete example is more useful than abstract advice. Load Stage 3, run the companion `stress_rw` program for a few seconds with `sysctl dev.myfirst.0.stats` watching from another terminal, and you see something like this:

**Before `stress_rw` starts:**

```text
dev.myfirst.0.stats.attach_ticks: 12345678
dev.myfirst.0.stats.open_count: 0
dev.myfirst.0.stats.active_fhs: 0
dev.myfirst.0.stats.bytes_read: 0
dev.myfirst.0.stats.bytes_written: 0
dev.myfirst.0.stats.bufused: 0
```

Zero activity, one attach, buffer empty.

**During `stress_rw`, with `watch -n 0.5 sysctl dev.myfirst.0.stats`:**

```text
dev.myfirst.0.stats.attach_ticks: 12345678
dev.myfirst.0.stats.open_count: 2
dev.myfirst.0.stats.active_fhs: 2
dev.myfirst.0.stats.bytes_read: 1358976
dev.myfirst.0.stats.bytes_written: 1359040
dev.myfirst.0.stats.bufused: 64
```

Two active descriptors (writer + reader), counters climbing, buffer holding 64 bytes of in-flight data. `bytes_written` is slightly ahead of `bytes_read`, which is exactly what you would expect: the writer produced a chunk the reader has not quite consumed yet. The difference equals `bufused`.

**After `stress_rw` exits:**

```text
dev.myfirst.0.stats.attach_ticks: 12345678
dev.myfirst.0.stats.open_count: 2
dev.myfirst.0.stats.active_fhs: 0
dev.myfirst.0.stats.bytes_read: 4800000
dev.myfirst.0.stats.bytes_written: 4800000
dev.myfirst.0.stats.bufused: 0
```

Both descriptors closed. Lifetime opens is 2 (cumulative). Active is 0. `bytes_read` equals `bytes_written`; the reader caught up fully. Buffer is empty.

Three signatures to notice. First, `active_fhs` always tracks live descriptors; it is a live value, not a cumulative counter. Second, `bytes_read == bytes_written` at steady state when the reader is keeping up, plus whatever is sitting in `bufused`. Third, the `open_count` is a lifetime value that never decreases; a quick way to spot churn is to watch it grow while `active_fhs` stays stable.

A driver that behaves predictably under load is a driver you can operate with confidence. Once the counters line up the way this paragraph describes, you have your first real driver, not a toy.



## 有符号、无符号和差一的危害

A short section on a class of bug that has caused more kernel panics than almost any other. It shows up especially often in I/O handlers.

### ssize_t 与 size_t

Two types dominate I/O code:

- `size_t`: unsigned, used for sizes and counts. `sizeof(x)` returns `size_t`. `malloc(9)` takes `size_t`. `memcpy` takes `size_t`.
- `ssize_t`: signed, used when a value could be negative (usually -1 for error). `read(2)` and `write(2)` return `ssize_t`. `uio_resid` is `ssize_t`.

The two types have the same width on every platform FreeBSD supports, but they do not silently convert between each other without warnings, and they behave very differently when arithmetic underflows.

A subtraction of `size_t` values that would produce a negative result instead wraps around to a huge positive value, because `size_t` is unsigned. For example:

```c
size_t avail = sc->buflen - sc->bufused;
```

If `sc->bufused` is larger than `sc->buflen`, `avail` is an enormous number, and the next `uiomove` attempts a transfer that blows past the end of the buffer.

The defence is the invariant. In every buffer-management section of the chapter, we maintain `sc->bufhead + sc->bufused <= sc->buflen`. As long as that invariant holds, `sc->buflen - (sc->bufhead + sc->bufused)` cannot underflow.

The risk is in code paths that violate the invariant accidentally. A double-free that restores an already-consumed value; a write that updates `bufused` twice; a race between writers. Those are the bugs to hunt for when `avail` ever looks wrong.

### uio_resid 可以与无符号值比较

`uio_resid` is `ssize_t`. Your buffer sizes are `size_t`. Code like this:

```c
if (uio->uio_resid > sc->buflen) ...
```

Will be compiled with a signed-vs-unsigned comparison. Modern compilers warn about this; the warning should be taken seriously.

The safer pattern is to cast explicitly:

```c
if ((size_t)uio->uio_resid > sc->buflen) ...
```

Or to use `MIN`, which we have been using:

```c
towrite = MIN((size_t)uio->uio_resid, avail);
```

The cast is defensible because `uio_resid` is documented to be non-negative in valid uios (and `uiomove` `KASSERT`s on it). The cast makes the compiler happy and makes the intent explicit.

### 计数器中的差一错误

A counter updated on the wrong side of an error check is a classic bug:

```c
sc->bytes_read += towrite;          /* BAD: happens even on error */
error = uiomove(sc->buf, towrite, uio);
```

The correct shape is to increment after success:

```c
error = uiomove(sc->buf, towrite, uio);
if (error == 0)
        sc->bytes_read += towrite;
```

This is why we have `if (error == 0)` guarding every counter update in the chapter. The cost is one line of code. The benefit is that your counters match reality.

### uio_offset - before 惯用法

When you want to know "how many bytes did `uiomove` actually move?", the cleanest way is to compare `uio_offset` before and after:

```c
off_t before = uio->uio_offset;
error = uiomove_frombuf(sc->buf, sc->buflen, uio);
size_t moved = uio->uio_offset - before;
```

This works for both full and short transfers. `moved` is the actual byte count, regardless of what the caller asked for or how much was available.

The idiom is free at runtime (two subtractions) and unambiguous in code. Use it when your driver wants to count bytes; the alternative, inferring the count from `uio_resid`, requires knowing the original request size, which is more bookkeeping.



## 额外故障排除：边界情况

Expanding on the earlier troubleshooting section, here are a few more scenarios you are likely to hit the first time you write a real driver.

### "The second read on the same descriptor returns zero"

Expected for a static-message driver (Stage 1): once `uio_offset` reaches the end of the message, `uiomove_frombuf` returns zero.

Unexpected for a FIFO driver (Stage 3): the first read drained the buffer, and no writer has refilled it. The caller should not be issuing a second read back-to-back without a write happening in between.

To distinguish the two cases, check `sysctl dev.myfirst.0.stats.bufused`. If it is zero, the buffer is empty. If it is non-zero and you still see zero bytes, you have a bug.

### "The driver returns zero bytes immediately when the buffer has data"

The read handler is taking the wrong branch. Common causes:

- A `bufused == 0` check placed in the wrong spot. If the check runs before the per-open state retrieval, it might short-circuit the read before the real work.
- An accidental `return 0;` earlier in the handler (for example, in a debug branch left from a previous experiment).
- A missing `mtx_unlock` on an error path, making every subsequent call block on the mutex forever. Symptom: the second call hangs, not a zero-byte return; but it is worth checking.

### "My `uiomove_frombuf` always returns zero regardless of the buffer"

Two common causes:

- The `buflen` argument is zero. `uiomove_frombuf` returns zero immediately if `buflen <= 0`.
- `uio_offset` is already at or past `buflen`. `uiomove_frombuf` returns zero to signal EOF in that case.

Add a `device_printf` logging the arguments at entry to confirm which case you are in.

### "The buffer overflows into adjacent memory"

Your arithmetic is wrong. Somewhere you are calling `uiomove(sc->buf + X, N, uio)` where `X + N > sc->buflen`. The write proceeds silently and corrupts kernel memory.

Your kernel will usually panic shortly thereafter, possibly in a completely unrelated subsystem. The panic message will not mention your driver; it will mention whichever heap neighbour got clobbered.

If you suspect this, rebuild with `INVARIANTS` and `WITNESS` (and on many targets, KASAN on amd64). These kernel features catch buffer overruns much earlier than the default kernel does.

### "A process reading from the device hangs forever"

Since Chapter 9 does not implement blocking I/O, this should not happen with `myfirst` Stage 3. If it does, the most likely cause is the process holding a file descriptor while you tried to unload the driver; `destroy_dev(9)` is waiting for `si_threadcount` to reach zero, and the process is sitting inside your handler for some reason.

To diagnose: `ps auxH | grep <your-test>`; `gdb -p <pid>` and `bt`. The stack should reveal where the thread is parked.

If your Stage 3 handler accidentally sleeps (for instance, because you added a `tsleep` while experimenting with Chapter 10 material early), the fix is to remove the sleep. Chapter 9's driver does not block.

### "`kldunload` says `kldunload: can't unload file: Device busy`"

Classic symptom of a descriptor still open. Use `fuser /dev/myfirst/0` to find the offending process, close the descriptor or kill the process, and retry.

### "I modified the driver and `make` compiles but `kldload` fails with version mismatch"

Your build environment does not match your running kernel. Check:

```sh
% freebsd-version -k
14.3-RELEASE
% ls /usr/obj/usr/src/amd64.amd64/sys/GENERIC
```

If `/usr/src` is for a different release, your headers produce a module that the kernel refuses. Rebuild against the matching sources. In a lab VM this usually means syncing `/usr/src` with the running release via `fetch` or `freebsd-update src-install`.

### "I see every byte written through the device printed twice in dmesg"

You have a `device_printf` inside the hot path that prints every transfer. Remove it or guard it with `if (bootverbose)`.

A milder version of the same bug: a single-line log that prints the length of every transfer. For small test workloads that looks fine; for a real user workload it will bury dmesg and cause timestamp compression in the kernel buffer.

### "My `d_read` is called but my `d_write` is not"

Either the user program never calls `write(2)` on the device, or it calls `write(2)` with the descriptor not opened for writing (`O_RDONLY`). Check both.

Also: confirm that `cdevsw.d_write` is assigned to `myfirst_write`. A copy-paste bug that assigns it to `myfirst_read` results in both directions hitting the read handler, with predictably confusing results.



## 设计说明：为什么每个阶段停在它停的地方

A short meta-section on why Chapter 9's three stages have the boundaries they have. This is the kind of chapter-design reasoning that is worth making explicit, because it is the reasoning you will apply when you design your own drivers.

### 为什么存在阶段 1

Stage 1 is the smallest possible `d_read` that is not `/dev/null`. It introduces:

- The `uiomove_frombuf(9)` helper, the easiest way to get a fixed buffer out to user space.
- Per-descriptor offset handling.
- The pattern of using `uio_offset` as the state carrier.

Stage 1 does not do anything with writes; the Chapter 8 stub is fine.

Without Stage 1, the jump from stubs to a buffered read/write driver is too large. Stage 1 lets you confirm, with minimal code, that the read handler is wired up correctly. Everything else builds on that confirmation.

### 为什么存在阶段 2

Stage 2 introduces:

- A dynamically-allocated kernel buffer.
- A write path that accepts user data.
- A read path that honours the caller's offset across the accumulating buffer.
- The first realistic use of the softc mutex in an I/O handler.

Stage 2 deliberately does not drain reads. The buffer grows until full; subsequent writes return `ENOSPC`. This lets two concurrent readers confirm that they each have their own `uio_offset`, which is the property Stage 1 couldn't demonstrate (because Stage 1 had nothing to write).

### 为什么存在阶段 3

Stage 3 introduces:

- Reads that drain the buffer.
- The coordination between a head pointer and a used count.
- The FIFO semantics that most real drivers approximate.

Stage 3 does not wrap around. The head and used pointers walk forward through the buffer and the buffer collapses to the beginning when empty. A proper ring buffer (with head and tail wrapping around a fixed-size array) belongs in Chapter 10 because it pairs naturally with blocking reads and `poll(2)`: a ring makes steady-state operation efficient, and efficient steady-state operation is exactly what a blocking reader needs.

### 为什么这里没有环形缓冲区

A ring buffer is five to fifteen lines of additional bookkeeping beyond what Stage 3 does. Adding it now would not be a large amount of code. The reason it is deferred is pedagogical: the two concepts ("I/O path semantics" and "ring buffer mechanics") are independently confusing to a beginner, and splitting them into two chapters lets each chapter address one pile of confusion at a time.

By the time Chapter 10 introduces the ring, the reader is fluent in the I/O path. The new material is only the ring bookkeeping.

### 为什么没有阻塞

Blocking is useful, but it introduces `msleep(9)`, condition variables, the `d_purge` teardown hook, and a thicket of correctness issues around what to wake and when. Each of those is a substantial topic. Mixing them into Chapter 9 would double its length and halve its clarity.

Chapter 10's first section is "when your driver has to wait". It is a natural follow-on.

### 各阶段**不**试图成为什么

The stages are not a simulation of a hardware driver. They do not mimic DMA. They do not simulate interrupts. They do not pretend to be anything other than what they are: in-memory drivers that exercise the UNIX I/O path.

This matters because later in the book, when we write actual hardware drivers, the I/O path will look identical. The hardware specifics (where the bytes come from, where the bytes go to) will change, but the handler shape, the uiomove usage, the errno conventions, the counter patterns, all of these will be recognisable from Chapter 9.

A driver that moves bytes correctly across the user/kernel trust boundary is 80% of any real driver. Chapter 9 teaches that 80%.



## 动手实验

The labs below track the three stages above. Each lab is a checkpoint that proves your driver is doing the thing the text just described. Read the lab fully before starting, and do them in order.

### 实验 9.1：构建并加载阶段 1

**Goal:** Build the Stage 1 driver, load it, read the static message, and confirm per-descriptor offset handling.

**Steps:**

1. Start from the companion tree: `cp -r examples/part-02/ch09-reading-and-writing/stage1-static-message ~/drivers/ch09-stage1`. Alternatively, modify your Chapter 8 stage 2 driver according to the Stage 1 walkthrough above.
2. Change into the directory and build:
   ```sh
   % cd ~/drivers/ch09-stage1
   % make
   ```
3. Load the module:
   ```sh
   % sudo kldload ./myfirst.ko
   ```
4. Confirm the device is present:
   ```sh
   % ls -l /dev/myfirst/0
   crw-rw----  1 root  operator ... /dev/myfirst/0
   ```
5. Read the message:
   ```sh
   % cat /dev/myfirst/0
   Hello from myfirst.
   This is your first real read path.
   Chapter 9, Stage 1.
   ```
6. Build the `rw_myfirst.c` userland tool from the companion tree and run it in "read twice" mode:
   ```sh
   % cc -o rw_myfirst rw_myfirst.c
   % ./rw_myfirst read
   [read 1] 75 bytes:
   Hello from myfirst.
   This is your first real read path.
   Chapter 9, Stage 1.
   [read 2] 0 bytes (EOF)
   ```
7. Confirm the per-descriptor counter:
   ```sh
   % dmesg | tail -5
   ```
   You should see the `open via /dev/myfirst/0 fh=...` and `per-open dtor fh=...` lines from Chapter 8, plus the message body was read.
8. Unload:
   ```sh
   % sudo kldunload myfirst
   ```

**Success criteria:**

- `cat` prints the message.
- The userland tool shows 75 bytes on the first read and 0 bytes on the second.
- `dmesg` shows one open and one destructor per `./rw_myfirst read` invocation.

**Common mistakes:**

- Forgetting the `-1` on `sizeof(myfirst_message) - 1`. The message will include a trailing NUL byte that appears as a stray character in user output.
- Not calling `devfs_get_cdevpriv` before the `sc == NULL` check. The rest of the chapter depends on this order; run it to see why it is the right one.
- Using `(void *)sc->message` instead of `__DECONST(void *, sc->message)`. Both work on most compilers; the `__DECONST` form is the convention and suppresses a warning on some compiler configurations.

### 实验 9.2：用写入和读取练习阶段 2

**Goal:** Build Stage 2, push data in from userland, pull it back out, and observe the sysctl counters.

**Steps:**

1. From the companion tree: `cp -r examples/part-02/ch09-reading-and-writing/stage2-readwrite ~/drivers/ch09-stage2`.
2. Build and load:
   ```sh
   % cd ~/drivers/ch09-stage2
   % make
   % sudo kldload ./myfirst.ko
   ```
3. Check the initial state:
   ```sh
   % sysctl dev.myfirst.0.stats
   dev.myfirst.0.stats.attach_ticks: ...
   dev.myfirst.0.stats.open_count: 0
   dev.myfirst.0.stats.active_fhs: 0
   dev.myfirst.0.stats.bytes_read: 0
   dev.myfirst.0.stats.bytes_written: 0
   dev.myfirst.0.stats.bufused: 0
   ```
4. Write a line of text:
   ```sh
   % echo "the quick brown fox" | sudo tee /dev/myfirst/0 > /dev/null
   ```
5. Read it back:
   ```sh
   % cat /dev/myfirst/0
   the quick brown fox
   ```
6. Observe the counters:
   ```sh
   % sysctl dev.myfirst.0.stats.bufused
   dev.myfirst.0.stats.bufused: 20
   % sysctl dev.myfirst.0.stats.bytes_written
   dev.myfirst.0.stats.bytes_written: 20
   % sysctl dev.myfirst.0.stats.bytes_read
   dev.myfirst.0.stats.bytes_read: 20
   ```
7. Trigger `ENOSPC`:
   ```sh
   % dd if=/dev/zero bs=1024 count=8 | sudo tee /dev/myfirst/0 > /dev/null
   ```
   Expect a short-write error. Inspect `sysctl dev.myfirst.0.stats.bufused`; it should be 4096 (the buffer size).
8. Confirm reads still deliver the content:
   ```sh
   % sudo cat /dev/myfirst/0 | od -An -c | head -3
   ```
9. Unload:
   ```sh
   % sudo kldunload myfirst
   ```

**Success criteria:**

- Writes deposit bytes; reads deliver them back.
- `bufused` matches the number of bytes written since the last reset.
- `dd` exhibits a short write when the buffer fills; the driver returns `ENOSPC`.
- `dmesg` shows open and destructor lines for every process that opened the device.

**Common mistakes:**

- Forgetting to free `sc->buf` in `detach`. The driver will unload without complaint, but a subsequent kernel memory leak check (`vmstat -m | grep devbuf`) will show drift.
- Holding the softc mutex while calling `uiomove`, without being sure the mutex is an `MTX_DEF` and not a spin lock. Chapter 7's `mtx_init(..., MTX_DEF)` is the right choice; do not change it.
- Omitting the `sc->bufused = 0` reset in `attach`. `Newbus` initialises softc to zero for you, but making the initialisation explicit is the convention; it also makes a later refactor less error-prone.

### 实验 9.3：阶段 3 FIFO 行为

**Goal:** Build Stage 3, exercise FIFO behaviour from two terminals, and confirm that reads drain the buffer.

**Steps:**

1. From the companion tree: `cp -r examples/part-02/ch09-reading-and-writing/stage3-echo ~/drivers/ch09-stage3`.
2. Build and load:
   ```sh
   % cd ~/drivers/ch09-stage3
   % make
   % sudo kldload ./myfirst.ko
   ```
3. In terminal A, write some bytes:
   ```sh
   % echo "message A" | sudo tee /dev/myfirst/0 > /dev/null
   ```
4. In terminal B, read them:
   ```sh
   % cat /dev/myfirst/0
   message A
   ```
5. Read again in terminal B:
   ```sh
   % cat /dev/myfirst/0
   ```
   Expect no output. The buffer is empty.
6. In terminal A, write two lines in rapid succession:
   ```sh
   % echo "first" | sudo tee /dev/myfirst/0 > /dev/null
   % echo "second" | sudo tee /dev/myfirst/0 > /dev/null
   ```
7. In terminal B, read:
   ```sh
   % cat /dev/myfirst/0
   first
   second
   ```
   Expect the two lines concatenated. Both writes appended to the same buffer before either read happened.
8. Inspect the counters:
   ```sh
   % sysctl dev.myfirst.0.stats
   ```
   `bufused` should be back to zero. `bytes_read` and `bytes_written` should match.
9. Unload:
   ```sh
   % sudo kldunload myfirst
   ```

**Success criteria:**

- Writes append to the buffer; reads drain it.
- A read after the buffer is drained returns immediately (EOF-on-empty).
- `bytes_read` always equals `bytes_written` once the reader has caught up.

**Common mistakes:**

- Not resetting `bufhead = 0` when `bufused` reaches zero. The buffer will "drift" toward the end of `sc->buf` and refuse writes long before it is full.
- Forgetting to update `bufhead` as reads drain. The driver will read the same bytes repeatedly.
- Using `uio->uio_offset` as a per-descriptor offset. In a FIFO, offsets are shared; a per-descriptor offset does not make sense and will confuse testers.

### 实验 9.4：使用 dd 测量传输行为

**Goal:** Use `dd(1)` to generate known-size transfers, read the results back, and check that the counters agree.

`dd` is the tool of choice here because it lets you control the block size, the number of blocks, and the behaviour on short transfers.

1. Reload the Stage 3 driver fresh:
   ```sh
   % sudo kldunload myfirst; sudo kldload ./myfirst.ko
   ```
2. Write 512 bytes in a single block:
   ```sh
   % sudo dd if=/dev/urandom of=/dev/myfirst/0 bs=512 count=1
   1+0 records in
   1+0 records out
   512 bytes transferred
   ```
3. Observe `bufused = 512`:
   ```sh
   % sysctl dev.myfirst.0.stats.bufused
   dev.myfirst.0.stats.bufused: 512
   ```
4. Read them back with matching block size:
   ```sh
   % sudo dd if=/dev/myfirst/0 of=/tmp/out bs=512 count=1
   1+0 records in
   1+0 records out
   512 bytes transferred
   ```
5. Check that the FIFO is now empty:
   ```sh
   % sysctl dev.myfirst.0.stats.bufused
   dev.myfirst.0.stats.bufused: 0
   ```
6. Write 8192 bytes in one big block:
   ```sh
   % sudo dd if=/dev/urandom of=/dev/myfirst/0 bs=8192 count=1
   dd: /dev/myfirst/0: No space left on device
   0+0 records in
   0+0 records out
   0 bytes transferred
   ```
   The driver accepted 4096 bytes (the buffer size) of the 8192 requested and returned a short write for the rest.
7. Alternatively, use `bs=4096` with `count=2`:
   ```sh
   % sudo dd if=/dev/urandom of=/dev/myfirst/0 bs=4096 count=2
   dd: /dev/myfirst/0: No space left on device
   1+0 records in
   0+0 records out
   4096 bytes transferred
   ```
   The first block of 4096 bytes succeeded in full; the second block failed with `ENOSPC`.
8. Drain:
   ```sh
   % sudo dd if=/dev/myfirst/0 of=/tmp/out bs=4096 count=1
   % sudo kldunload myfirst
   ```

**Success criteria:**

- `dd` reports the expected byte counts at each step.
- The driver accepts up to 4096 bytes and refuses the rest with `ENOSPC`.
- `bufused` tracks the buffer state after every operation.

### 实验 9.5：一个小型往返 C 程序

**Goal:** Write a short userland C program that opens the device, writes known bytes, closes the descriptor, opens it again, reads the bytes back, and verifies they match.

1. Save the following as `rw_myfirst.c` in `~/drivers/ch09-stage3`:

```c
#include <fcntl.h>
#include <unistd.h>
#include <string.h>
#include <stdio.h>

static const char payload[] = "round-trip test payload\n";

int
main(void)
{
        int fd;
        ssize_t n;

        fd = open("/dev/myfirst/0", O_WRONLY);
        if (fd < 0) { perror("open W"); return 1; }
        n = write(fd, payload, sizeof(payload) - 1);
        if (n != (ssize_t)(sizeof(payload) - 1)) {
                fprintf(stderr, "short write: %zd\n", n);
                return 2;
        }
        close(fd);

        char buf[128] = {0};
        fd = open("/dev/myfirst/0", O_RDONLY);
        if (fd < 0) { perror("open R"); return 3; }
        n = read(fd, buf, sizeof(buf) - 1);
        if (n < 0) { perror("read"); return 4; }
        close(fd);

        if ((size_t)n != sizeof(payload) - 1 ||
            memcmp(buf, payload, n) != 0) {
                fprintf(stderr, "mismatch: wrote %zu, read %zd\n",
                    sizeof(payload) - 1, n);
                return 5;
        }

        printf("round-trip OK: %zd bytes\n", n);
        return 0;
}
```

2. Build and run:
   ```sh
   % cc -o rw_myfirst rw_myfirst.c
   % sudo ./rw_myfirst
   round-trip OK: 24 bytes
   ```
3. Inspect `dmesg` to see the two opens and two destructors.

**Success criteria:**

- The program prints `round-trip OK: 24 bytes`.
- `dmesg` shows one open/destructor pair for the write and one for the read.

**Common mistakes:**

- Writing fewer bytes than the payload and not checking the return value. `write(2)` can return a short count; your test must handle it.
- Forgetting `O_WRONLY` vs `O_RDONLY`. `open(2)` enforces the mode against the access bits of the node; opening with the wrong mode returns `EACCES` (or similar).
- Assuming `read(2)` returns the requested count. It can return less; again, the caller loops.

### 实验 9.6：检查二进制往返

**Goal:** Confirm that the driver handles arbitrary binary data, not only text, by pushing random bytes through and checking that the same bytes come back.

1. With Stage 3 loaded and empty, write 256 random bytes:
   ```sh
   % sudo dd if=/dev/urandom of=/tmp/sent bs=256 count=1
   % sudo dd if=/tmp/sent of=/dev/myfirst/0 bs=256 count=1
   ```
2. Read the same number of bytes back:
   ```sh
   % sudo dd if=/dev/myfirst/0 of=/tmp/received bs=256 count=1
   ```
3. Compare:
   ```sh
   % cmp /tmp/sent /tmp/received && echo MATCH
   MATCH
   ```
4. Inspect both files byte-for-byte:
   ```sh
   % od -An -tx1 /tmp/sent | head -2
   % od -An -tx1 /tmp/received | head -2
   ```
5. Try a pathological pattern: all zeros, all `0xff`, then a file full of a single byte. Confirm every pattern round-trips exactly.

**Success criteria:**

- `cmp` reports no differences.
- The driver preserves every bit of the input.
- No byte-ordering, no "helpful" interpretation, no surprise transformations.

This lab is short but important: it verifies that your driver is a transparent byte store, not a text filter that accidentally interprets some bytes specially. If you ever see differences between the sent and received files, you have a bug in the transfer path, probably a length miscount or an off-by-one in the buffer arithmetic.

### 实验 9.7：端到端观察运行中的驱动

**Goal:** Combine sysctl, dmesg, truss, and vmstat into a single end-to-end observation of the Stage 3 driver under real load. This lab has no new code; it is the bridge from "I wrote the driver" to "I can see what it is doing".

**Steps:**

1. With Stage 3 loaded fresh, open four terminals. Terminal A will run the driver load / unload cycles. Terminal B will monitor sysctl. Terminal C will tail dmesg. Terminal D will run a user workload.
2. **Terminal A:**
   ```sh
   % sudo kldload ./myfirst.ko
   % vmstat -m | grep devbuf
   ```
   Note the `devbuf` row's `InUse` and `MemUse` values.
3. **Terminal B:**
   ```sh
   % watch -n 1 sysctl dev.myfirst.0.stats
   ```
4. **Terminal C:**
   ```sh
   % sudo dmesg -c > /dev/null
   % sudo dmesg -w
   ```
   The `-c` clears accumulated messages; the `-w` watches for new ones.
5. **Terminal D:**
   ```sh
   % cd examples/part-02/ch09-reading-and-writing/userland
   % make
   % sudo truss ./rw_myfirst rt 2>&1 | tail -10
   ```
6. Check terminal B: you should see `open_count` increment by 2 (one for the write, one for the read), `active_fhs` return to 0, and `bytes_read == bytes_written`.
7. Check terminal C: you should see two open lines and two destructor lines from `device_printf`.
8. In terminal A, run `vmstat -m | grep devbuf` again. `InUse` and `MemUse` should have decreased back to their pre-load values plus whatever the driver itself allocated (typically just the 4 KiB buffer and the softc).
9. **Stress run:** in terminal D,
   ```sh
   % sudo ./stress_rw -s 5
   ```
   Watch terminal B. You should see `bufused` oscillate, counters climb, and `active_fhs` hit 2 while the test runs.
10. When the stress run finishes, in terminal B verify `active_fhs` is 0. In terminal A,
    ```sh
    % sudo kldunload myfirst
    % vmstat -m | grep devbuf
    ```
    `InUse` should have returned to its pre-load baseline. If it has not, your driver leaked an allocation and `vmstat -m` just told you.

**Success criteria:**

- Sysctl counters match the workload you ran.
- Dmesg shows one open/destructor pair per descriptor open/close.
- Truss output matches your mental model of what the program did.
- `vmstat -m | grep devbuf` returns to its baseline after unload.
- No panics, no warnings, no unexplained counter drift.

**Why this lab matters:** this is the first lab that exercises the full observability toolchain at once. In production, the signal that something is wrong almost never comes from a crash; it comes from a counter that has drifted out of bounds, a `dmesg` line nobody expected, or a `vmstat -m` reading that does not match reality. Building the habit of looking at all four surfaces together is what separates "I wrote a driver" from "I am responsible for a driver".



## 挑战练习

These challenges stretch the material without introducing topics that belong to later chapters. Each one uses only the primitives we have introduced. Try them before looking at the companion tree; the learning is in the attempt, not the answer.

### 挑战 9.1：每次描述符的读取计数器

Extend Stage 2 so that the per-descriptor `reads` counter is exposed via a sysctl. The counter should be available per active descriptor, which means a per-`fh` sysctl rather than a per-softc one.

This challenge is harder than it looks: sysctls are allocated and freed at known points in the softc lifecycle, and the per-descriptor structure lives only as long as its descriptor. A clean solution registers a sysctl node per `fh` in `d_open` and unregisters it in the destructor. Be careful about lifetimes; the sysctl context must be freed before the `fh` memory.

*Hint:* `sysctl_ctx_init` and `sysctl_ctx_free` are per-context. You can give each `fh` its own context, and free it in the destructor.

*Alternative:* keep a linked list of `fh` pointers in the softc (under the mutex) and expose it through a custom sysctl handler that walks the list on demand. This is the pattern `/usr/src/sys/kern/tty_info.c` uses for per-process stats.

### 挑战 9.2：一个支持 readv(2) 的测试

Write a user program that uses `readv(2)` to read from the driver into three separate buffers of sizes 8, 16, and 32 bytes. Confirm that the driver delivers bytes into all three buffers in sequence.

The kernel and `uiomove(9)` already handle `readv(2)`; the driver does not need changes. The purpose of this challenge is to convince yourself of that fact.

*Hint:* `struct iovec iov[3] = {{buf1, 8}, {buf2, 16}, {buf3, 32}};`, then `readv(fd, iov, 3)`. The return value is the total bytes delivered across all three buffers; the individual `iov_len` values are not modified on the user side.

### 挑战 9.3：短写入演示

Modify Stage 2's `myfirst_write` to accept at most 128 bytes per call, regardless of `uio_resid`. A user program that writes 1024 bytes should see a short write of 128 every time.

Then write a short test program that writes 1024 bytes in a single `write(2)` call, observes the short-write return value, and loops until all 1024 bytes have been accepted.

Questions to think through:

- Does `cat` handle short writes correctly? (Yes.)
- Does `echo > /dev/myfirst/0 "..."` handle them correctly? (Usually, via `printf` in the shell, but sometimes not; worth testing.)
- What happens if you remove the short-write behaviour and try to exceed the buffer size? (You get `ENOSPC` after the first 4096-byte write.)

This challenge teaches you to separate "the driver does the right thing" from "user programs assume what drivers do".

### 挑战 9.4：一个 ls -l 传感器

Make the driver's response to a read depend on the `ls -l` output of the device itself. That is: every read produces the current timestamp of the device node.

*Hint:* `sc->cdev->si_ctime` and `sc->cdev->si_mtime` are `struct timespec` fields on the cdev. You can convert them to a string with `printf` formatting, place the string in a kernel buffer, and `uiomove_frombuf(9)` it out.

*Warning:* `si_ctime` / `si_mtime` may be updated by devfs as nodes are touched. Observe what happens when you `touch /dev/myfirst/0` and read again.

### 挑战 9.5：一个反向回显驱动

Modify Stage 3 so that every read returns the bytes in reverse order from how they were written. A write of `"hello"` followed by a read should produce `"olleh"`.

This challenge is entirely about buffer bookkeeping. The `uiomove` calls stay the same; you change the addresses you hand to them.

*Hint:* You can either reverse the buffer on every read (expensive) or store bytes in reverse order on the write side (cheaper). Neither is the "right" answer; each has different correctness and concurrency properties. Pick one and argue for it in a comment.

### 挑战 9.6：二进制往返

Write a user program that writes a `struct timespec` to the driver, then reads one back. Compare the two structures. Are they equal? They should be, because `myfirst` is a transparent byte store.

Extend the program to write two `struct timespec` values, then `lseek(fd, sizeof(struct timespec), SEEK_SET)` and read the second one. What happens? (Clue: the FIFO does not support seeks meaningfully.)

This challenge illustrates the "read and write carry bytes, not types" point from the safe-data-transfer section. The bytes round-trip perfectly; the type information does not.

### 挑战 9.7：一个十六进制查看测试工具

Write a short shell script that, given a byte count N, generates N random bytes with `dd if=/dev/urandom bs=$N count=1`, pipes them into your Stage 3 driver, then reads them back with `dd if=/dev/myfirst/0 bs=$N count=1`, and compares the two streams with `cmp`. The script should report success for matching streams and diff-like output for mismatching streams. Run it with N = 1, 2, 4, ..., 4096 to sweep small, boundary, and capacity-filling sizes.

Questions to answer as you run the sweep:

- Does every size round-trip cleanly up to and including 4096?
- At 4097, what does the driver do? Does the test harness report the error meaningfully?
- Is there any size at which `cmp` reports a difference? If so, what was the underlying cause?

This challenge rewards combining the tools in the Practical Workflow section: `dd` for precise transfers, `cmp` for byte-level verification, `sysctl` for counters, and the shell for orchestration. A robust test harness like this is the kind of habit that pays for itself every time you refactor a driver and want to know quickly whether the behaviour is still right.

### 挑战 9.8：谁打开了描述符？

Write a small C program that opens `/dev/myfirst/0`, blocks on `pause()` (so it holds the descriptor indefinitely), and runs until `SIGTERM`. In a second terminal, run `fstat | grep myfirst` and then `fuser /dev/myfirst/0`. Note the output. Now try to `kldunload myfirst`. What error do you get? Why?

Now kill the holder with `SIGTERM` or plain `kill`. Observe the destructor fire in `dmesg`. Try `kldunload` again. It should succeed.

This challenge is short, but it grounds one of the chapter's subtler invariants: a driver cannot unload while any descriptor is open on one of its cdevs, and FreeBSD gives operators a standard set of tools to find the holder. The next time a real-world `kldunload` fails with `EBUSY`, you will have seen the shape of the problem before.



## 常见错误故障排除

Every `d_read` / `d_write` mistake you are likely to make falls into one of a small number of categories. This section is a short field guide.

### "My driver returns zero bytes even though I wrote data"

This is usually one of two bugs.

**Bug 1**: You forgot to update `bufused` (or equivalent) after the successful `uiomove`. The write arrived, the bytes moved, but the driver's state never reflected the arrival. The next read sees `bufused == 0` and reports EOF.

Fix: always update your tracking fields inside `if (error == 0) { ... }` after `uiomove` returns.

**Bug 2**: You reset `bufused` (or `bufhead`) somewhere inappropriate. A common pattern is adding a reset line inside `d_open` or `d_close` "for cleanliness". That wipes out the data the previous caller wrote.

Fix: reset driver-wide state only in `attach` (at load) or `detach` (at unload). Per-descriptor state belongs in `fh`, reset by `malloc(M_ZERO)` and cleaned up by the destructor.

### "My reads return garbage"

The buffer is uninitialised. `malloc(9)` without `M_ZERO` returns a block of memory whose contents are undefined. If your `d_read` reaches past `bufused`, or reads from offsets that have not been written, the bytes you see are leftovers from whatever memory the kernel recycled.

Fix: always pass `M_ZERO` to `malloc` in `attach`. Always clamp reads to the current high-water mark (`bufused`), not to the buffer's total size (`buflen`).

There is a more serious variant of this bug. A driver that returns uninitialised kernel memory to user space has just leaked kernel state into user space. In production that is a security hole. In development it is a bug; in production it is a CVE.

### "The kernel panics with a pagefault on a user address"

You called `memcpy` or `bcopy` directly on a user pointer instead of going through `uiomove` / `copyin` / `copyout`. The access faulted, the kernel had no fault handler installed, and the result was a panic.

Fix: never dereference a user pointer directly. Route through `uiomove(9)` (in handlers) or `copyin(9)` / `copyout(9)` (in other contexts).

### "The driver refuses to unload"

You have at least one file descriptor still open. `detach` returns `EBUSY` when `active_fhs > 0`; the module will not unload until every `fh` has been destroyed.

Fix: close the descriptor in userland. If a background process is holding it, kill the process (after confirming it is yours; do not kill system daemons). `fstat -p <pid>` shows which files a process has open; `fuser /dev/myfirst/0` shows which processes have the node open.

Chapter 10 will introduce `destroy_dev_drain` patterns for drivers that need to coerce a blocked reader to exit. Chapter 9 does not block, so this issue does not arise in normal operation; when it arises, it is because userland is holding the descriptor somewhere unexpected.

### "My write handler returns EFAULT"

Your `uiomove` call hit an invalid user address. The common causes:

- A user program called `write(fd, NULL, n)` or `write(fd, (void*)0xdeadbeef, n)`.
- A user program wrote a pointer it had freed.
- You accidentally passed a kernel pointer as the destination to `uiomove`. This can happen if you build a uio by hand for kernel-space data and then pass it to a handler expecting a user-space uio. The resulting `copyout` sees a "user" address that is actually a kernel address; depending on the architecture, you either get `EFAULT` or a subtle corruption.

Fix: check `uio->uio_segflg`. For user-driven handlers, it should be `UIO_USERSPACE`. If you are passing around a kernel-space uio, make sure `uio_segflg == UIO_SYSSPACE` and that your code paths know the difference.

### "My counters are wrong under concurrent writes"

Two writers raced on `bufused`. Each read the current value, added to it, and wrote back, and the second writer overwrote the first writer's update with a stale value.

Fix: take `sc->mtx` around every read-modify-write of shared state. Part 3 makes this a first-class topic; for Chapter 9, a single mutex around the whole critical section is enough.

### "sysctl counters do not reflect the real state"

Two variants.

**Variant A**: the counter is a `size_t`, but the sysctl macro is `SYSCTL_ADD_U64`. On 32-bit architectures, the macro reads 8 bytes where the field is only 4 bytes wide; half the value is junk.

Fix: match the sysctl macro to the field type. `size_t` pairs with `SYSCTL_ADD_UINT` on 32-bit platforms and `SYSCTL_ADD_U64` on 64-bit platforms. To be portable, use `uint64_t` for counters and cast when updating.

**Variant B**: the counter is never updated because the update is inside the `if (error == 0)` block and `uiomove` returned a non-zero error. That is actually correct behaviour: you should not count bytes you did not move. The symptom only looks like a bug if you are trying to use the counter to debug the error.

Fix: add an `error_count` counter that ticks on every non-zero return, independently of `bytes_read` and `bytes_written`. Useful for debugging.

### "The first read after a fresh load returns zero bytes"

Usually intentional. In Stage 3, an empty buffer returns zero bytes. If you expected the static message from Stage 1, check that you are running the Stage 1 driver, not a later one.

If it is unintentional, double-check that `attach` is setting `sc->buf`, `sc->buflen`, and `sc->message_len` as expected. A common bug is copy-pasting the attach code from Stage 1 into Stage 2 and leaving the `sc->message = ...` assignment in place, which then takes precedence over the `malloc` line.

### "The build fails with unknown reference to uiomove_frombuf"

You forgot to include `<sys/uio.h>`. Add it to the top of `myfirst.c`.

### "My handler is called twice for one read(2)"

It almost certainly is not. What is more likely: your handler is being called once with `uio_iovcnt > 1` (a `readv(2)` call), and inside `uiomove` each iovec entry is being drained in turn. The internal loop in `uiomove` may make multiple `copyout` calls in what is a single invocation of your handler.

Verify by adding a `device_printf` at entry and exit of your `d_read`. You should see one entry and one exit per user-space `read(2)` call, regardless of iovec count.



## 对比模式：正确与有问题的处理程序

The troubleshooting guide above is reactive: it helps when something has already gone wrong. This section is the prescriptive companion. Each entry shows a plausible but wrong way to write part of a handler, pairs it with the correct rewrite, and explains the distinction. Studying the contrasts in advance is the fastest way to avoid the bugs in the first place.

Read each pair carefully. The correct version is the pattern you should reach for; the buggy version is the shape your own hands may produce when you are moving fast. Recognising the mistake in the wild, months from now, is worth the five minutes it takes to internalise the difference today.

### 对比 1：返回字节计数

**Buggy:**

```c
static int
myfirst_read(struct cdev *dev, struct uio *uio, int ioflag)
{
        /* ... */
        error = uiomove_frombuf(sc->message, sc->message_len, uio);
        if (error)
                return (error);
        return (sc->message_len); /* BAD: returning a count */
}
```

**Correct:**

```c
static int
myfirst_read(struct cdev *dev, struct uio *uio, int ioflag)
{
        /* ... */
        return (uiomove_frombuf(sc->message, sc->message_len, uio));
}
```

**Why it matters.** The handler's return value is an errno, not a count. The kernel computes the byte count from the change in `uio->uio_resid` and reports it to user space. A non-zero positive return is interpreted as an errno; if you returned `sc->message_len`, the caller would receive a very strange `errno` value. For example, returning `75` would manifest as `errno = 75`, which on FreeBSD happens to be `EPROGMISMATCH`. The bug is both wrong and deeply confusing to anybody looking at it from the user side.

The rule is simple and absolute: handlers return errno values, never counts. If you want to know the byte count, compute it from the uio.

### 对比 2：处理零长度请求

**Buggy:**

```c
static int
myfirst_read(struct cdev *dev, struct uio *uio, int ioflag)
{
        if (uio->uio_resid == 0)
                return (EINVAL); /* BAD: zero-length is legal */
        /* ... */
}
```

**Correct:**

```c
static int
myfirst_read(struct cdev *dev, struct uio *uio, int ioflag)
{
        /* No special case. uiomove handles zero-resid cleanly. */
        return (uiomove_frombuf(sc->message, sc->message_len, uio));
}
```

**Why it matters.** A `read(fd, buf, 0)` call is legal UNIX. A driver that rejects it with `EINVAL` breaks programs that use zero-byte reads to check descriptor state. `uiomove` returns zero immediately if the uio has nothing to move; your handler does not need to special-case it. Special-casing it wrong is worse than not special-casing it at all.

### 对比 3：缓冲区容量计算

**Buggy:**

```c
mtx_lock(&sc->mtx);
avail = sc->buflen - sc->bufused;
towrite = uio->uio_resid;            /* BAD: no clamp */
error = uiomove(sc->buf + sc->bufused, towrite, uio);
if (error == 0)
        sc->bufused += towrite;
mtx_unlock(&sc->mtx);
return (error);
```

**Correct:**

```c
mtx_lock(&sc->mtx);
avail = sc->buflen - sc->bufused;
if (avail == 0) {
        mtx_unlock(&sc->mtx);
        return (ENOSPC);
}
towrite = MIN((size_t)uio->uio_resid, avail);
error = uiomove(sc->buf + sc->bufused, towrite, uio);
if (error == 0)
        sc->bufused += towrite;
mtx_unlock(&sc->mtx);
return (error);
```

**Why it matters.** The buggy version hands `uiomove` a length of `uio_resid`, which may exceed the buffer's remaining capacity. `uiomove` will not move more than `uio_resid` bytes, but the *destination* is `sc->buf + sc->bufused`, and the math does not know about `sc->buflen`. If the user writes 8 KiB into a 4 KiB buffer with `bufused = 0`, the handler will write 4 KiB past the end of `sc->buf`. That is a classic kernel heap overflow: the crash will not be immediate, will not implicate your driver, and may reveal itself as a panic inside a completely unrelated subsystem half a second later.

The correct version clamps the transfer to `avail`, guaranteeing that the pointer arithmetic stays inside the buffer. The clamp is one `MIN` call, and it is not optional.

### 对比 4：跨 uiomove 持有自旋锁

**Buggy:**

```c
mtx_lock_spin(&sc->spin);            /* BAD: spin lock, not a regular mutex */
error = uiomove(sc->buf + off, n, uio);
mtx_unlock_spin(&sc->spin);
return (error);
```

**Correct:**

```c
mtx_lock(&sc->mtx);                  /* MTX_DEF mutex */
error = uiomove(sc->buf + off, n, uio);
mtx_unlock(&sc->mtx);
return (error);
```

**Why it matters.** `uiomove(9)` may sleep. When it calls `copyin` or `copyout`, the user page may be paged out, and the kernel may need to page it in from disk, which requires waiting on I/O. A sleep while holding a spin lock (`MTX_SPIN`) deadlocks the system. FreeBSD's `WITNESS` framework panics on this the first time it happens, if `WITNESS` is enabled. On a non-`WITNESS` kernel the result is silent livelock.

The rule is straightforward: spin locks cannot be held across functions that may sleep, and `uiomove` may sleep. Use an `MTX_DEF` mutex (the default, and the one `myfirst` uses) for softc state that is touched by I/O handlers.

### 对比 5：在 d_open 中重置共享状态

**Buggy:**

```c
static int
myfirst_open(struct cdev *dev, int oflags, int devtype, struct thread *td)
{
        struct myfirst_softc *sc = dev->si_drv1;
        /* ... */
        mtx_lock(&sc->mtx);
        sc->bufused = 0;                 /* BAD: wipes other readers' data */
        sc->bufhead = 0;
        mtx_unlock(&sc->mtx);
        /* ... */
}
```

**Correct:**

```c
static int
myfirst_open(struct cdev *dev, int oflags, int devtype, struct thread *td)
{
        struct myfirst_softc *sc = dev->si_drv1;
        struct myfirst_fh *fh;
        /* ... no shared-state reset ... */
        fh = malloc(sizeof(*fh), M_DEVBUF, M_WAITOK | M_ZERO);
        /* fh starts zeroed, which is correct per-descriptor state */
        /* ... register fh with devfs_set_cdevpriv, bump counters ... */
}
```

**Why it matters.** `d_open` runs once per descriptor. If two readers open the device, the second open will wipe whatever the first open left behind. Driver-wide state (`sc->bufused`, `sc->buf`, counters) belongs to the whole driver, and is reset only at `attach` and `detach`. Per-descriptor state belongs in `fh`, which `malloc(M_ZERO)` initialises to zeros automatically.

A driver that resets shared state in `d_open` looks like it works under a single opener and silently corrupts state when two openers appear. The bug is invisible until the day two users read the device at once.

### 对比 6：在知道结果之前记账

**Buggy:**

```c
sc->bytes_written += towrite;       /* BAD: count before success */
error = uiomove(sc->buf + tail, towrite, uio);
if (error == 0)
        sc->bufused += towrite;
```

**Correct:**

```c
error = uiomove(sc->buf + tail, towrite, uio);
if (error == 0) {
        sc->bufused += towrite;
        sc->bytes_written += towrite;
}
```

**Why it matters.** If `uiomove` fails part-way through, some bytes may have moved and some may not. The `sc->bytes_written` counter should reflect what actually reached the buffer, not what the driver attempted. Updating counters before the outcome is known makes the counters lie. If a user reads the sysctl to diagnose a problem, they see numbers that do not correspond to reality.

The rule: update counters inside the `if (error == 0)` branch, so success is the only path that increments them. This is a small cost for a large correctness benefit.

### 对比 7：直接解引用用户指针

**Buggy:**

```c
/* Imagine the driver somehow gets a user pointer, maybe through ioctl. */
static int
handle_user_string(void *user_ptr)
{
        char buf[128];
        memcpy(buf, user_ptr, 128);     /* BAD: user pointer in memcpy */
        /* ... */
}
```

**Correct:**

```c
static int
handle_user_string(void *user_ptr)
{
        char buf[128];
        int error;

        error = copyin(user_ptr, buf, sizeof(buf));
        if (error != 0)
                return (error);
        /* ... */
}
```

**Why it matters.** `memcpy` assumes both pointers refer to memory accessible in the current address space. A user pointer does not. Depending on the platform, the result of passing a user pointer to `memcpy` in kernel context ranges from an `EFAULT`-equivalent fault (on amd64 with SMAP enabled) to silent data corruption (on platforms without user/kernel separation) to an outright kernel panic.

`copyin` and `copyout` are the one-and-only correct way to access user memory from kernel context. They install a fault handler, validate the address, walk page tables safely, and return `EFAULT` on any failure. The performance cost is a few extra instructions; the correctness benefit is "the kernel does not panic when a buggy user program is running".

### 对比 8：在 d_open 失败时泄漏每次打开的结构

**Buggy:**

```c
static int
myfirst_open(struct cdev *dev, int oflags, int devtype, struct thread *td)
{
        struct myfirst_fh *fh;
        int error;

        fh = malloc(sizeof(*fh), M_DEVBUF, M_WAITOK | M_ZERO);
        /* ... set fields ... */
        error = devfs_set_cdevpriv(fh, myfirst_fh_dtor);
        if (error != 0)
                return (error);         /* BAD: fh is leaked */
        return (0);
}
```

**Correct:**

```c
static int
myfirst_open(struct cdev *dev, int oflags, int devtype, struct thread *td)
{
        struct myfirst_fh *fh;
        int error;

        fh = malloc(sizeof(*fh), M_DEVBUF, M_WAITOK | M_ZERO);
        /* ... set fields ... */
        error = devfs_set_cdevpriv(fh, myfirst_fh_dtor);
        if (error != 0) {
                free(fh, M_DEVBUF);     /* free before returning */
                return (error);
        }
        return (0);
}
```

**Why it matters.** When `devfs_set_cdevpriv` fails, the kernel does not register the destructor, so the destructor will never run on this `fh`. If the handler returns without freeing `fh`, the memory is leaked. Under steady load, repeated `d_open` failures can leak enough memory to destabilise the kernel.

The rule: in error-unwind paths, every allocation made so far must be freed. The Chapter 8 reader has seen this pattern for attach; it applies equally to `d_open`.

### 如何使用这个对比表

These eight pairs are not an exhaustive list. They are the bugs we have seen most often in early student drivers, and the bugs the chapter's text has been trying to help you avoid. Read through them once now. Before you write your first real driver outside this book, read them again.

A useful habit while developing: whenever you finish a handler, walk it against the contrast table mentally. Does the handler return a count? Does it special-case zero-resid? Does it have a capacity clamp? Is the mutex type right? Does it reset shared state in `d_open`? Does it account for bytes on failure? Does it dereference any user pointer directly? Does it leak on `d_open` failure? Eight questions, five minutes. The price of the check is small; the cost of shipping one of these bugs into production is large.



## 第十章前的自我评估

Chapter 9 has covered a lot of ground. Before you put it down, run through the following checklist. If any item makes you hesitate, the relevant section is worth re-reading before moving on. This is not a test; it is a quick way to identify the spots where your mental model may still be thin.

**Concepts:**

- [ ] I can explain in one sentence what `struct uio` is for.
- [ ] I can name the three fields of `struct uio` my driver reads most often.
- [ ] I can explain why `uiomove(9)` is preferred over `copyin` / `copyout` inside `d_read` and `d_write`.
- [ ] I can explain why `memcpy` across the user / kernel boundary is unsafe.
- [ ] I can explain the difference between `ENXIO`, `EAGAIN`, `ENOSPC`, and `EFAULT` in driver terms.

**Mechanics:**

- [ ] I can write a minimal `d_read` handler that serves a fixed buffer using `uiomove_frombuf(9)`.
- [ ] I can write a minimal `d_write` handler that appends to a kernel buffer with a correct capacity clamp.
- [ ] I know where to put the mutex acquire and release around the transfer.
- [ ] I know how to propagate an errno from `uiomove` back to user space.
- [ ] I know how to mark a write as fully consumed with `uio_resid = 0`.

**Observability:**

- [ ] I can read `sysctl dev.myfirst.0.stats` and interpret each counter.
- [ ] I can spot a memory leak with `vmstat -m | grep devbuf`.
- [ ] I can use `truss(1)` to see what syscalls my test program makes.
- [ ] I can use `fstat(1)` or `fuser(8)` to find who is holding a descriptor.

**Traps:**

- [ ] I would not return a byte count from `d_read` / `d_write`.
- [ ] I would not reject a zero-length request with `EINVAL`.
- [ ] I would not reset `sc->bufused` inside `d_open`.
- [ ] I would not hold a spin lock across a `uiomove` call.

Any "no" here is a signal, not a verdict. Re-read the relevant section; run a small experiment in your lab; come back to the checklist. By the time every box is ticked, you are solidly ready for Chapter 10.



## 总结

你刚刚实现了使驱动活跃的入口点。 在第七章结束时你的驱动有了一个骨架。 在第八章结束时它有了一个形状良好的门。 现在，在第九章结束时，数据双向流过这扇门。

本章的核心教训比看起来更短。 你将编写的每个 `d_read` 都有相同的三行骨架：获取每次打开的状态、验证活跃性、调用 `uiomove`。 你将编写的每个 `d_write` 都有类似的骨架，加上一个额外的决定（我有多少空间？）和一个防止缓冲区溢出的钳制（`MIN(uio_resid, avail)`）。 Everything else in the chapter is context: why `struct uio` looks the way it does, why `uiomove` is the only safe mover, why errno values matter, why counters matter, why the buffer has to be freed on every error path.

### 最重要的三个想法

**First, `struct uio` is the contract between your driver and the kernel's I/O machinery.** It carries everything your handler needs to know about a call: what the user asked for, where the user's memory is, what direction the transfer should move, and how much progress has been made. You do not need to memorise all seven fields. You need to recognise `uio_resid` (the remaining work), `uio_offset` (the position, if you care), and `uio_rw` (the direction), and you need to trust `uiomove(9)` with the rest.

**Second, `uiomove(9)` is the boundary between user memory and kernel memory.** Everything your driver ever moves between the two passes through it (or through one of its close relatives: `uiomove_frombuf`, `copyin`, `copyout`). This is not a suggestion. Direct pointer access across the trust boundary either corrupts memory or leaks information, and the kernel has no cheap way to catch the mistake before it becomes a CVE. If a pointer came from user space, route it through the kernel's trust-boundary functions. Always.

**Third, a correct handler is usually a short one.** If your `d_read` or `d_write` is longer than fifteen lines, something is probably wrong. Longer handlers either duplicate logic that belongs elsewhere (in the buffer management, in the per-open state setup, in the sysctls), or they are trying to do something the driver should not be doing in a data-path handler (typically, something that belongs in `d_ioctl`). Keep the handlers short. Put the machinery they call into well-named helper functions. Your future self will thank you.

### 你结束本章时的驱动形状

Your Stage 3 `myfirst` is a small, honest, in-memory FIFO. The salient features:

- A 4 KiB kernel buffer, allocated in `attach` and freed in `detach`.
- A per-instance mutex guarding `bufhead`, `bufused`, and the associated counters.
- A `d_read` that drains the buffer and advances `bufhead`, collapsing to zero when the buffer empties.
- A `d_write` that appends to the buffer and returns `ENOSPC` when it fills.
- Per-descriptor counters stored in `struct myfirst_fh`, allocated in `d_open`, freed in the destructor.
- A sysctl tree exposing the live driver state.
- Clean `attach` error unwind and clean `detach` ordering.

That shape will come back, recognisable, in half the drivers you will read in Part 4 and Part 6. It is a general pattern, not a one-off demo.

### 开始第十章前你应该练习什么

Five exercises, in rough order of increasing challenge:

1. Rebuild all three stages from scratch, without looking at the companion tree. Compare your result to the tree afterward; the differences are what you have left to internalise.
2. Introduce an intentional bug in Stage 3: forget to reset `bufhead` when `bufused` reaches zero. Observe what happens on the second big write. Explain the symptom in terms of the code.
3. Add a sysctl that exposes `sc->buflen`. Make it read-only. Then convert it into a tunable that can be set at load time via `kenv` or `loader.conf` and picked up in `attach`. (Chapter 10 revisits tunables formally; this is a preview.)
4. Write a shell script that writes random data of a known length to `/dev/myfirst/0` and then reads it back through `sha256`. Compare the hashes. Do the hashes match even when the write size exceeds the buffer? (They should not; think about why.)
5. Find a driver under `/usr/src/sys/dev` that implements both `d_read` and `d_write`. Read its handlers. Map them against the patterns in this chapter. Good candidates: `/usr/src/sys/dev/null/null.c` (you already know it), `/usr/src/sys/dev/random/randomdev.c`, `/usr/src/sys/dev/speaker/spkr.c`.

### 展望第十章

Chapter 10 takes the Stage 3 driver and makes it scale. Four new capabilities show up:

- **A circular buffer** replaces the linear buffer. Writes and reads can both happen continuously without the explicit collapse that Stage 3 uses.
- **Blocking reads** arrive. A reader that calls `read(2)` on an empty buffer can sleep until data is available, rather than returning zero bytes immediately. The kernel's `msleep(9)` is the primitive; the `d_purge` handler is the teardown safety net.
- **Non-blocking I/O** becomes a first-class feature. `O_NONBLOCK` users get `EAGAIN` where a blocking caller would sleep.
- **`poll(2)` and `kqueue(9)` integration**. A user program can wait for the device to become readable or writable without actively attempting the operation. This is the standard way to integrate a device into an event loop.

All four of these build on the same `d_read` / `d_write` shapes you just implemented. You will extend the handlers rather than rewriting them, and the per-descriptor state you have in place will carry the necessary bookkeeping. The chapter before that (this one) is the one where the I/O path itself is correct. Chapter 10 is where the I/O path becomes efficient.

Before you close the file, a last reassurance. The material in this chapter is not as difficult as it may feel on a first read. The pattern is small. The ideas are real, but they are finite, and you have just exercised every one of them against working code. When you read a real driver's `d_read` or `d_write` in the tree, you will now recognise what the function is doing and why. You are not a beginner at this any more. You are an apprentice with a real tool in your hands.



## 参考：本章使用的签名和辅助函数

A consolidated reference for the declarations, helpers, and constants the chapter leans on. Keep this page bookmarked while you write drivers; most beginner questions are a lookup in one of these tables.

### `d_read` and `d_write` Signatures

From `/usr/src/sys/sys/conf.h`:

```c
typedef int d_read_t(struct cdev *dev, struct uio *uio, int ioflag);
typedef int d_write_t(struct cdev *dev, struct uio *uio, int ioflag);
```

The return value is zero on success, a positive errno on failure. The byte count is computed from the change in `uio->uio_resid` and reported to user space as the return value of `read(2)` / `write(2)`.

### The Canonical `struct uio`

来自 `/usr/src/sys/sys/uio.h`：

```c
struct uio {
        struct  iovec *uio_iov;         /* scatter/gather list */
        int     uio_iovcnt;             /* length of scatter/gather list */
        off_t   uio_offset;             /* offset in target object */
        ssize_t uio_resid;              /* remaining bytes to process */
        enum    uio_seg uio_segflg;     /* address space */
        enum    uio_rw uio_rw;          /* operation */
        struct  thread *uio_td;         /* owner */
};
```

### The `uio_seg` and `uio_rw` Enumerations

From `/usr/src/sys/sys/_uio.h`:

```c
enum uio_rw  { UIO_READ, UIO_WRITE };
enum uio_seg { UIO_USERSPACE, UIO_SYSSPACE, UIO_NOCOPY };
```

### `uiomove` Family

来自 `/usr/src/sys/sys/uio.h`：

```c
int uiomove(void *cp, int n, struct uio *uio);
int uiomove_frombuf(void *buf, int buflen, struct uio *uio);
int uiomove_fromphys(struct vm_page *ma[], vm_offset_t offset, int n,
                     struct uio *uio);
int uiomove_nofault(void *cp, int n, struct uio *uio);
int uiomove_object(struct vm_object *obj, off_t obj_size, struct uio *uio);
```

In beginner driver code, only `uiomove` and `uiomove_frombuf` are common. The others support specific kernel subsystems (physical-page I/O, page-fault-free copies, VM-backed objects) and are out of scope for this chapter.

### `copyin` and `copyout`

来自 `/usr/src/sys/sys/systm.h`：

```c
int copyin(const void * __restrict udaddr,
           void * __restrict kaddr, size_t len);
int copyout(const void * __restrict kaddr,
            void * __restrict udaddr, size_t len);
int copyinstr(const void * __restrict udaddr,
              void * __restrict kaddr, size_t len,
              size_t * __restrict lencopied);
```

Use these in control paths (`d_ioctl`) where a user pointer arrives outside the uio abstraction. Inside `d_read` and `d_write`, prefer `uiomove`.

### `ioflag` Bits That Matter for Character Devices

From `/usr/src/sys/sys/vnode.h`:

```c
#define IO_NDELAY       0x0004  /* FNDELAY flag set in file table */
```

Set when the descriptor is in non-blocking mode. Your `d_read` or `d_write` can use this to decide whether to block (missing flag) or return `EAGAIN` (flag set). Most of the other `IO_*` flags are filesystem-level and irrelevant to character devices.

### Memory Allocation

From `/usr/src/sys/sys/malloc.h`:

```c
void *malloc(size_t size, struct malloc_type *type, int flags);
void  free(void *addr, struct malloc_type *type);
```

Common flags: `M_WAITOK`, `M_NOWAIT`, `M_ZERO`. Common types for drivers: `M_DEVBUF` (generic) or a driver-specific type declared via `MALLOC_DECLARE` / `MALLOC_DEFINE`.

### Per-Open State (Chapter 8 carryover, used here)

From `/usr/src/sys/sys/conf.h`:

```c
int  devfs_set_cdevpriv(void *priv, d_priv_dtor_t *dtr);
int  devfs_get_cdevpriv(void **datap);
void devfs_clear_cdevpriv(void);
```

The pattern is: allocate in `d_open`, register with `devfs_set_cdevpriv`, retrieve in every later handler with `devfs_get_cdevpriv`, clean up in the destructor that `devfs_set_cdevpriv` registered.

### Errno Values Used in This Chapter

| Errno         | Meaning in a driver context                                |
|---------------|------------------------------------------------------------|
| `0`           | Success.                                                    |
| `ENXIO`       | Device not configured (softc missing, not attached).        |
| `EFAULT`      | Bad user address. Usually propagated from `uiomove`.        |
| `EIO`         | Input / output error. Hardware issue.                       |
| `ENOSPC`      | No space left on device. Buffer full.                       |
| `EAGAIN`      | Would block; relevant in non-blocking mode (Chapter 10).    |
| `EINVAL`      | Invalid argument.                                           |
| `EACCES`      | Permission denied at `open(2)`.                             |
| `EPIPE`       | Broken pipe. Not used by `myfirst`.                         |

### Helpful `device_printf(9)` Patterns

```c
device_printf(sc->dev, "open via %s fh=%p\n", devtoname(sc->cdev), fh);
device_printf(sc->dev, "write rejected: buffer full (used=%zu)\n",
    sc->bufused);
device_printf(sc->dev, "read delivered %zd bytes\n",
    (ssize_t)(before - uio->uio_offset));
```

These are written for readability. A line in `dmesg` you have to decode is a line that probably will not be read when it matters.

### The Three Stages at a Glance

| Stage | `d_read`                                             | `d_write`                          |
|-------|------------------------------------------------------|------------------------------------|
| 1     | Serve fixed message via `uiomove_frombuf`            | Discard writes (like `/dev/null`)  |
| 2     | Serve buffer up to `bufused`                         | Append to buffer, `ENOSPC` if full |
| 3     | Drain buffer from `bufhead`, reset on empty          | Append at `bufhead + bufused`, `ENOSPC` if full |

Stage 3 is the foundation Chapter 10 builds on.

### Consolidated File List for the Chapter

Companion files under `examples/part-02/ch09-reading-and-writing/`:

- `stage1-static-message/`: Stage 1 driver source and Makefile.
- `stage2-readwrite/`: Stage 2 driver source and Makefile.
- `stage3-echo/`: Stage 3 driver source and Makefile.
- `userland/rw_myfirst.c`: small C program to exercise read and write round-trips.
- `userland/stress_rw.c`: multi-process stress test for Lab 9.3 and beyond.
- `README.md`: a short map of the companion tree.

Each stage is independent; you can build, load, and exercise any of them without building the others. The Makefiles are identical except for the driver name (always `myfirst`) and optional tuning flags.



## 附录 A：深入查看 uiomove 的内部循环

For readers who want to see exactly what `uiomove(9)` does, this appendix walks through the core loop of `uiomove_faultflag` as it appears in `/usr/src/sys/kern/subr_uio.c`. You do not need to read this to write a driver. It is here because one reading of the loop will clarify every later question you have about uio semantics.

### The Setup

At entry, the function has:

- A kernel pointer `cp` provided by the caller (your driver).
- An integer `n` provided by the caller (the max bytes to move).
- The uio provided by the kernel dispatch.
- A boolean `nofault` indicating whether page faults during the copy should be handled or fatal.

It sanity-checks a few invariants: the direction is `UIO_READ` or `UIO_WRITE`, the owning thread is the current thread when the segment is user-space, and `uio_resid` is non-negative. Any violation is a `KASSERT` and will panic a kernel with `INVARIANTS` enabled.

### The Main Loop

```c
while (n > 0 && uio->uio_resid) {
        iov = uio->uio_iov;
        cnt = iov->iov_len;
        if (cnt == 0) {
                uio->uio_iov++;
                uio->uio_iovcnt--;
                continue;
        }
        if (cnt > n)
                cnt = n;

        switch (uio->uio_segflg) {
        case UIO_USERSPACE:
                switch (uio->uio_rw) {
                case UIO_READ:
                        error = copyout(cp, iov->iov_base, cnt);
                        break;
                case UIO_WRITE:
                        error = copyin(iov->iov_base, cp, cnt);
                        break;
                }
                if (error)
                        goto out;
                break;

        case UIO_SYSSPACE:
                switch (uio->uio_rw) {
                case UIO_READ:
                        bcopy(cp, iov->iov_base, cnt);
                        break;
                case UIO_WRITE:
                        bcopy(iov->iov_base, cp, cnt);
                        break;
                }
                break;
        case UIO_NOCOPY:
                break;
        }
        iov->iov_base = (char *)iov->iov_base + cnt;
        iov->iov_len -= cnt;
        uio->uio_resid -= cnt;
        uio->uio_offset += cnt;
        cp = (char *)cp + cnt;
        n -= cnt;
}
```

Each iteration does one unit of work: copy up to `cnt` bytes (where `cnt` is `MIN(iov->iov_len, n)`) between the current iovec entry and the kernel buffer. The direction is chosen by the two nested `switch` statements. After a successful copy, all the accounting fields advance in lockstep: the iovec entry shrinks by `cnt`, the uio's resid shrinks by `cnt`, the uio's offset grows by `cnt`, the kernel pointer `cp` advances by `cnt`, and the caller's `n` shrinks by `cnt`.

When an iovec entry is fully drained (`cnt == 0` at loop entry), the function advances to the next entry. When the caller's `n` reaches zero or the uio's resid reaches zero, the loop terminates.

If `copyin` or `copyout` returns non-zero, the function jumps to `out` without updating the fields for that iteration, so the partial-copy accounting is consistent: whatever bytes did copy are reflected in `uio_resid`, whatever did not copy is still pending.

### What You Should Take Away

Three invariants fall out of the loop that matter for your driver code.

- **Your call to `uiomove(cp, n, uio)` moves at most `MIN(n, uio->uio_resid)` bytes.** There is no way to ask for more than the uio has room for; the function caps at whichever side is smaller.
- **On a partial transfer, the state is consistent.** `uio_resid` reflects exactly the bytes that did not move. You can make another call and it will pick up correctly.
- **The fault handling is inside the loop, not around it.** A fault during a `copyin` / `copyout` returns `EFAULT` for the remainder; the fields are still consistent.

These three facts are why the three-line spine we keep returning to (`uiomove`, check error, update state) is sufficient. The kernel is doing the complicated work inside the loop; your driver just has to cooperate.



## 附录 B：为什么允许 read(fd, buf, 0)

A short note on a question that comes up frequently: why does UNIX allow a `read(fd, buf, 0)` or `write(fd, buf, 0)` call at all?

There are two answers, and both are worth knowing.

**The practical answer**: zero-length I/O is a free test. A user program that wants to check whether a descriptor is in a reasonable state can call `read(fd, NULL, 0)` without committing to a real transfer. If the descriptor is broken, the call returns an error. If it is fine, the call returns zero and costs almost nothing.

**The semantic answer**: the UNIX I/O interface uses byte counts consistently, and special-casing zero is more work than allowing it. A call with `count == 0` is a well-defined no-op: the kernel has to do nothing, and can return zero immediately. The alternative, returning `EINVAL` for zero-count calls, would force every user program that computed a count dynamically to guard against the case. That is the kind of change that breaks decades of code for no benefit.

The driver-side consequence, which we noted earlier: your handler must not panic or error on a zero `uio_resid`. The kernel effectively handles the case for you when you go through `uiomove`, which returns zero immediately if there is nothing to move.

If you ever find yourself writing `if (uio->uio_resid == 0) return (EINVAL);` in a driver, stop. That is the wrong answer. Zero-count I/O is valid; return zero.



## 附录 C：/dev/zero 读取路径简短导览

As a closing piece of analysis, it is worth walking through exactly what happens when a user program calls `read(2)` on `/dev/zero`. The driver is `/usr/src/sys/dev/null/null.c` and the handler is `zero_read`. Once you understand this path, you understand everything in Chapter 9.

### From User Space to Kernel Dispatch

The user calls:

```c
ssize_t n = read(fd, buf, 1024);
```

The C library makes the `read` syscall. The kernel looks up `fd` in the calling process's file table, retrieves the `struct file`, identifies its vnode, dispatches the call into devfs.

devfs identifies the cdev associated with the vnode, acquires a reference on it, and calls its `d_read` function pointer (`zero_read`) with the uio the kernel prepared.

### Inside `zero_read`

```c
static int
zero_read(struct cdev *dev __unused, struct uio *uio, int flags __unused)
{
        void *zbuf;
        ssize_t len;
        int error = 0;

        KASSERT(uio->uio_rw == UIO_READ,
            ("Can't be in %s for write", __func__));
        zbuf = __DECONST(void *, zero_region);
        while (uio->uio_resid > 0 && error == 0) {
                len = uio->uio_resid;
                if (len > ZERO_REGION_SIZE)
                        len = ZERO_REGION_SIZE;
                error = uiomove(zbuf, len, uio);
        }
        return (error);
}
```

- Assert that the direction is correct. Good practice; a `KASSERT` costs nothing in production kernels.
- Set `zbuf` to point at `zero_region`, a large pre-allocated zero-filled area.
- Loop: while the caller wants more bytes, determine the transfer size (min of `uio_resid` and the zero region's size), call `uiomove`, accumulate any error.
- Return.

### Inside `uiomove`

For the first iteration, `uiomove` sees `uio_resid = 1024`, `len = 1024` (since `ZERO_REGION_SIZE` is much larger), `uio_segflg = UIO_USERSPACE`, `uio_rw = UIO_READ`. It selects `copyout(zbuf, buf, 1024)`. The kernel performs the copy, handling any page fault on the user buffer. On success, `uio_resid` drops to zero, `uio_offset` grows by 1024, and the iovec is fully consumed.

### Back Up the Stack

`uiomove` returns zero. The loop in `zero_read` sees `uio_resid == 0` and exits. `zero_read` returns zero.

devfs releases its reference on the cdev. The kernel computes the byte count as `1024 - 0 = 1024`. `read(2)` returns 1024 to the user.

The user's buffer now holds 1024 zero bytes.

### What This Tells You About Your Own Driver

Two observations.

First, every data-path decision in `zero_read` is one you are now making too. How large of a chunk to move per iteration; which buffer to read from; how to handle the error from `uiomove`. Your driver's decisions will differ in the specifics (your buffer is not a pre-allocated zero region, your chunk size is not `ZERO_REGION_SIZE`), but the shape is identical.

Second, everything above `zero_read` is kernel machinery you do not have to write. You implement the handler, and the kernel takes care of the syscall, the file-descriptor lookup, the VFS dispatch, the devfs routing, the reference counting, and the fault handling. That is the power of the abstraction: you add your driver's knowledge, and everything else comes for free.

The flip side is that when you write a driver, you are committing to *cooperating* with that machinery. Every invariant that `uiomove` and devfs rely on is now your responsibility to uphold. The chapter has been walking you through those invariants one at a time, by building three small drivers that each exercise a different subset.

By now, the pattern should be familiar.



## 附录 D：用户侧常见的 read(2)/write(2) 返回值

A short cheat sheet for what a user program sees when it talks to your driver. This is not driver code; it is the view from the other side of the trust boundary. Reading it occasionally is the best inoculation against the subtle bugs that arise when the driver does something other than what a well-behaved UNIX program expects.

### `read(2)`

- A positive integer: that many bytes were placed into the caller's buffer. Less than the requested count means a short read; the caller loops.
- Zero: end of file. No more bytes will ever be produced on this descriptor. The caller stops.
- `-1` with `errno = EAGAIN`: non-blocking mode, no data available right now. The caller waits (via `select(2)` / `poll(2)` / `kqueue(2)`) and tries again.
- `-1` with `errno = EINTR`: a signal interrupted the read. The caller usually retries unless the signal handler tells it not to.
- `-1` with `errno = EFAULT`: the buffer pointer was invalid. The caller has a bug.
- `-1` with `errno = ENXIO`: the device is gone. The caller should close the descriptor and give up.
- `-1` with `errno = EIO`: the device reported a hardware error. The caller may retry or report.

### `write(2)`

- A positive integer: that many bytes were accepted. Less than the offered count means a short write; the caller loops with the remainder.
- Zero: theoretically possible, rarely seen in practice. Usually treated the same as a short write of zero bytes.
- `-1` with `errno = EAGAIN`: non-blocking mode, no space right now. The caller waits and retries.
- `-1` with `errno = ENOSPC`: permanently no space. The caller either stops writing or reopens the descriptor.
- `-1` with `errno = EPIPE`: the reader closed. Relevant for pipe-like devices, not for `myfirst`.
- `-1` with `errno = EFAULT`: the buffer pointer was invalid.
- `-1` with `errno = EINTR`: interrupted by a signal. Usually retried.

### What This Means for Your Driver

Two takeaways.

First, `EAGAIN` is how non-blocking callers expect a driver to say "no data / no room right now, come back later". A non-blocking caller that sees `EAGAIN` does not treat it as an error; it waits for a wake-up (usually via `poll(2)`) and retries. Chapter 10 makes this mechanism work for `myfirst`.

Second, `ENOSPC` is how a driver signals a permanent out-of-room condition on a write. It differs from `EAGAIN` in that the caller does not expect retries to succeed soon. For `myfirst` Stage 3 we use `ENOSPC` when the buffer fills and there is no reader actively draining; Chapter 10 will layer `EAGAIN` on top of the same condition for non-blocking readers and writers.

A driver that returns the wrong errno here is almost indistinguishable from a driver that is misbehaving. The cost of getting it right is tiny. The cost of getting it wrong shows up in confused user programs months later.



## 附录 E：一页速查表

If you only have five minutes before starting Chapter 10, here is the one-page version of everything above.

**The signatures:**

```c
static int myfirst_read(struct cdev *dev, struct uio *uio, int ioflag);
static int myfirst_write(struct cdev *dev, struct uio *uio, int ioflag);
```

Return zero on success, a positive errno on failure. Never return a byte count.

**The three-line spine for reads:**

```c
error = devfs_get_cdevpriv((void **)&fh);
if (error) return error;
return uiomove_frombuf(sc->buf, sc->buflen, uio);
```

Or, for a dynamic buffer:

```c
mtx_lock(&sc->mtx);
toread = MIN((size_t)uio->uio_resid, sc->bufused);
error = uiomove(sc->buf + offset, toread, uio);
if (error == 0) { /* update state */ }
mtx_unlock(&sc->mtx);
return error;
```

**The three-line spine for writes:**

```c
mtx_lock(&sc->mtx);
avail = sc->buflen - (sc->bufhead + sc->bufused);
if (avail == 0) { mtx_unlock(&sc->mtx); return ENOSPC; }
towrite = MIN((size_t)uio->uio_resid, avail);
error = uiomove(sc->buf + sc->bufhead + sc->bufused, towrite, uio);
if (error == 0) { sc->bufused += towrite; }
mtx_unlock(&sc->mtx);
return error;
```

**What to remember about uio:**

- `uio_resid`: bytes still pending. `uiomove` decrements this.
- `uio_offset`: position, if meaningful. `uiomove` increments this.
- `uio_rw`: direction. Trust `uiomove` to use it.
- Everything else: do not touch.

**What not to do:**

- Do not dereference user pointers directly.
- Do not use `memcpy` / `bcopy` between user and kernel.
- Do not return byte counts.
- Do not reset driver-wide state in `d_open` / `d_close`.
- Do not forget `M_ZERO` on `malloc(9)`.
- Do not hold a spin lock across `uiomove`.

**Errno values:**

- `0`: success.
- `ENXIO`: device not ready.
- `ENOSPC`: buffer full (permanent).
- `EAGAIN`: would block (non-blocking).
- `EFAULT`: from `uiomove`, propagate.
- `EIO`: hardware error.

That is the chapter.



## 章节总结

本章构建了数据路径。 从第八章的桩函数开始，我们在三个阶段中实现了 `d_read` 和 `d_write`，每个阶段都是完整的可加载驱动。

- **Stage 1** 对静态内核字符串使用了 `uiomove_frombuf(9)`, 带有使两个并发读取者的进度独立的每次描述符偏移处理。
- **Stage 2** 引入了动态内核缓冲区、向其中追加的写入路径和从中提供服务的读取路径。 缓冲区在挂载时确定大小，满缓冲区以 `ENOSPC` 拒绝进一步写入。
- **Stage 3** 将缓冲区转变为先进先出队列。 读取从头部排空，写入向尾部追加，驱动在缓冲区清空时将 `bufhead` 折叠为零。

在此过程中我们逐字段剖析了 `struct uio`, 解释了为什么 `uiomove(9)` 是在读或写处理程序中跨越用户/内核信任边界的唯一合法方式, 并构建了一个良好行为的驱动使用的小型 errno 值词汇表: `ENXIO`, `EFAULT`, `ENOSPC`, `EAGAIN`, `EIO`. 我们走过了 `uiomove` 的内部循环，使其保证感觉是应得的而非神秘的。 最后我们有五个实验、六个挑战、一个故障排除指南和一份一页速查表。

阶段 3 驱动是通往第十章的路径点。 它正确地移动字节。 它尚未高效地移动字节: 空缓冲区立即返回零字节，满缓冲区立即返回 `ENOSPC`，没有阻塞，没有 `poll(2)` 集成，没有环形缓冲区。 第十章在我们刚刚绘制的形状基础上修复所有这些问题。

你刚学到的模式会重复。 `/usr/src/sys/dev` 中的每个字符设备 I/O 处理程序都构建在相同的三参数签名、相同的 `struct uio` 和相同的 `uiomove(9)` 原语之上。 驱动之间的差异在于它们如何准备数据，而不在于它们如何移动数据。 一旦你识别了移动机制，你打开的每个处理程序几乎立即变得可读。

你现在有足够的知识阅读 FreeBSD 源码树中的任何 `d_read` 或 `d_write` 并理解它在做什么。 那是一个重要的里程碑。在翻页之前花一分钟欣赏它。
