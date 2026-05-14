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
- 桩函数 `d_read` 和 `d_write` 处理程序检索每次打开的状态，可选地查看它，并立即返回：`d_read` 返回零字节（EOF），`d_write` 通过设置 `uio_resid = 0` 声称已消耗所有字节。

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
- 你理解 `struct cdev` 是什么以及它如何与 `cdevsw` 相关联。第八章已详细介绍了这些内容。

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
13. 从用户空间通过内核到你的处理程序的 `read(2)` 的逐步追踪，加上镜像的写入追踪。
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

这是我们的阶段 1 `d_read` 的样子。先不要输入它；我们将在实现部分完整演练源码。现在在这里看到它主要是为了锚定讨论。

在阅读代码之前，请停在一个将在本章余下几乎每个处理程序中重复出现的细节上。任何感知每次打开的处理程序的前四行都遵循固定的**样板模式**：

```c
struct myfirst_fh *fh;
int error;

error = devfs_get_cdevpriv((void **)&fh);
if (error != 0)
        return (error);
```

这个模式检索 `d_open` 通过 `devfs_set_cdevpriv(9)` 注册的每次描述符 `fh`，并将任何失败原封不动地传播回内核。你将在 `myfirst_read`、`myfirst_write`、`myfirst_ioctl`、`myfirst_poll` 和 `kqfilter` 辅助函数的顶部看到它。当后面的实验说"用通常的 `devfs_get_cdevpriv` 样板检索每次打开状态"时，它指的就是这个代码块，本章余下部分不会重新解释它。如果一个处理程序曾经重新排列了这些行，请将其视为一个危险信号：在此调用之前运行任何逻辑意味着处理程序尚不知道它正在为哪次打开服务。一个值得记住的微妙之处是 `sc == NULL` 存活检查位于这个样板*之后*，而不是之前，因为你需要安全地检索每次打开状态，即使设备正在被拆除。

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

将此协议应用于 `/usr/src/sys/dev/null/null.c` 中的 `zero_read`。参数名称是标准的。`uiomove` 调用传递内核指针 `zbuf`（指向 `zero_region`）和由 `ZERO_REGION_SIZE` 限制的长度。没有锁；数据是常量。处理程序可以返回的唯一 errno 是 `uiomove` 返回的任何值。没有状态转换；`/dev/zero` 是无状态的。

现在将同样的协议应用于阶段 3 的 `myfirst_write`。参数名称：标准的。`uiomove` 调用：内核指针 `sc->buf + bufhead + bufused`，长度 `MIN((size_t)uio->uio_resid, avail)`。锁：`sc->mtx` 在之前获取、在之后释放。Errno 返回值：`ENXIO`（设备已消失）、`ENOSPC`（缓冲区满）、通过 `uiomove` 产生的 `EFAULT` 或零。状态转换：`sc->bufused += towrite`、`sc->bytes_written += towrite`、`fh->writes += towrite`。

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

一个"只在正常路径上工作"的初学者驱动最终会崩溃内核。 I/O 处理的有趣部分不是正常路径的部分：零长度读取、部分写入、错误的用户指针、调用中途传递的信号、耗尽的缓冲区，以及这些情况的几十种变体。本节讨论常见情况和与之相关的 errno 值。

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

`__DECONST` 转换是 FreeBSD 中去除 `const` 的惯用法。`uiomove_frombuf` 接受非 `const` 的 `void *`，因为它准备在任一方向移动，但在这种上下文中我们知道方向是内核到用户（读取），所以我们知道内核缓冲区不会被修改。在这里去除 `const` 是安全的；使用普通的 `(void *)` 转换也可以，但不够自文档化。

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

`bufused` 是 `size_t`，无符号整数的 sysctl 宏在 32 位平台上是 `SYSCTL_ADD_UINT`，在 64 位平台上是 `SYSCTL_ADD_U64`。由于此驱动在典型实验室中针对 amd64 上的 FreeBSD 14.3，`SYSCTL_ADD_UINT` 就可以；即使内部类型是 `size_t`，该字段也将被呈现为 `unsigned int`。如果你针对 arm64 或其他 64 位平台，使用 `SYSCTL_ADD_U64` 并相应转换。

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

调整 `attach` 中的错误展开以包含缓冲区释放：

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

现在来看读取处理程序：

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

写入处理程序：

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

从用户态进行冒烟测试：

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

缓冲区因 `"hello\n"` 增长了 6 字节，然后因 `"more\n"` 又增长了 5 字节，总共 11 字节。`cat` 读回所有 11 字节。从一个新打开的描述符的第二次 `cat` 从偏移零开始并再次读取它们。

如果我们写入的数据超过缓冲区容量会发生什么？

```sh
% dd if=/dev/zero bs=1024 count=8 | sudo tee /dev/myfirst/0 > /dev/null
dd: stdout: No space left on device
tee: /dev/myfirst/0: No space left on device
8+0 records in
7+0 records out
```

`dd` 写入了 7 个 1024 字节的块，第 8 个失败了。`tee` 报告了错误。驱动接受到其限制后干净地返回了 `ENOSPC`。内核将 errno 值传回用户空间。

### 阶段 3：先进先出回显驱动

阶段 3 将缓冲区变为 FIFO。写入追加到尾部。读取从头部排空。当缓冲区为空时，读取返回零字节（空时 EOF）。当缓冲区满时，写入返回 `ENOSPC`。

缓冲区保持线性：没有环绕。在排空所有数据的读取之后，`bufused` 为零，下一次写入再次从 `sc->buf` 中的偏移零开始。这使记录保持最少，并将阶段集中在 I/O 方向变化而不是环形缓冲区机制上。

softc 再增加一个字段：

```c
struct myfirst_softc {
        /* ...existing fields... */

        size_t  bufhead;   /* index of next byte to read */
        size_t  bufused;   /* bytes in the buffer, from bufhead onward */

        /* ...remaining fields... */
};
```

`bufhead` 是仍然要读取的第一个字节的偏移。`bufused` 是从 `bufhead` 开始的有效字节数。不变量 `bufhead + bufused <= buflen` 总是成立。

在 `attach` 中重置两者：

```c
sc->bufhead = 0;
sc->bufused = 0;
```

新的读取处理程序：

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

新的写入处理程序（与阶段 2 大部分相同，但注意追加的位置）：

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

写入在 `sc->bufhead + sc->bufused` 处追加，而不是仅在 `sc->bufused` 处，因为随着读取排空，有效数据切片已经移动了。

冒烟测试：

```sh
% echo "one" | sudo tee /dev/myfirst/0 > /dev/null
% echo "two" | sudo tee -a /dev/myfirst/0 > /dev/null
% cat /dev/myfirst/0
one
two
% cat /dev/myfirst/0
%
```

第一次 `cat` 之后，缓冲区为空。第二次 `cat` 看不到数据并立即退出。

这就是阶段 3 的形状。驱动是一个小型、诚实、内存中的 FIFO。用户可以向其中推入字节、从中拉出字节，并通过 sysctl 观察计数器。这就是真正的 I/O，也是第十章构建的基础。



## 从用户空间到你的处理程序追踪 read(2)

在开始做实验之前，逐步仔细看看当用户程序在你其中一个节点上调用 `read(2)` 时到底发生了什么。理解这条路径是改变你阅读驱动代码方式的事情之一。你在源码树中看到的每个处理程序都位于下面描述的调用链底部；一旦你认识到这个链条，每个处理程序都会变得眼熟。

### 步骤 1：用户程序调用 read(2)

C 库的 `read` 包装器将调用简单转换为系统调用陷阱：它将文件描述符、缓冲区指针和计数放入适当的寄存器，并执行当前架构的陷阱指令。控制权转移到内核。

这部分与驱动无关。每个系统调用都是相同的。重要的是，内核现在代表用户进程执行，在内核的地址空间中，用户的寄存器已保存，进程的凭据通过 `curthread->td_ucred` 可见。

### 步骤 2：内核查找文件描述符

内核调用 `sys_read(2)`（位于 `/usr/src/sys/kern/sys_generic.c`），它验证参数，在调用进程的文件表中查找文件描述符，并获取对结果 `struct file` 的引用。

如果描述符未打开，调用在此处以 `EBADF` 失败。如果描述符已打开但不可读（例如，用户使用 `O_WRONLY` 打开了设备），调用同样以 `EBADF` 失败。驱动不参与；`sys_read` 强制执行访问模式。

### 步骤 3：通用文件操作向量分派

`struct file` 有一个文件类型标签（`f_type`）和一个文件操作向量（`f_ops`）。对于常规文件，向量分派到 VFS 层；对于套接字，它分派到套接字；对于通过 devfs 打开的设备，它分派到 `vn_read`，后者又调用文件背后 vnode 上的 vnode 操作 `VOP_READ`。

这听起来可能像是为间接而间接。实际上这是内核如何保持系统调用路径的其余部分对每种文件都相同的方式。驱动不需要了解这一层；devfs 和 VFS 最终会将调用传递给你的处理程序。

### 步骤 4：VFS 调用 devfs

vnode 的文件系统操作指向 devfs 对 vnode 接口的实现（`devfs_vnops`）。devfs vnode 上的 `VOP_READ` 调用 `devfs_read_f`，它查看 vnode 背后的 cdev，获取其上的线程计数引用（递增 `si_threadcount`），并调用 `cdevsw->d_read`。那就是你的函数。

这一步的两个细节对你的驱动有影响。

首先，**`si_threadcount` 递增是 `destroy_dev(9)` 用来知道你的处理程序是否活跃的方式**。当模块卸载并且 `destroy_dev` 运行时，它会等待直到每个处理程序的每次当前调用都返回。引用在你的 `d_read` 被调用之前递增，并在它返回之后释放。这个机制是你的驱动可以在用户正在进行 `read(2)` 时安全卸载的原因。

其次，**从 VFS 层的角度来看，调用是同步的**。VFS 调用你的处理程序，等待它返回，然后传播结果。你不需要做任何特殊的事情来参与这种同步；完成时从处理程序返回即可。

### 步骤 5：你的 d_read 处理程序运行

这就是我们整章所在的位置。处理程序：

- 接收一个 `struct cdev *dev`（被读取的节点）、一个 `struct uio *uio`（I/O 描述）和一个 `int ioflag`（来自文件表条目的标志）。
- 通过 `devfs_get_cdevpriv(9)` 检索每次打开的状态。
- 验证活跃性。
- 通过 `uiomove(9)` 传输字节。
- 返回零或 errno。

到目前为止，这个步骤应该没有任何神秘之处了。

### 步骤 6：内核展开并报告

`devfs_read_f` 看到你的返回值。如果为零，它从 `uio->uio_resid` 的减少量计算字节计数并返回该计数。如果非零，它将 errno 转换为系统调用的错误返回。VFS 的 `vn_read` 将结果向上传递给 `sys_read`。`sys_read` 将结果写入返回值寄存器。

控制权转移回用户空间。C 库的 `read` 包装器检查结果：正值作为 `read(2)` 的返回值返回；负值设置 `errno` 并返回 `-1`。

用户程序看到它期望的整数，其控制流继续。

### 步骤 7：引用计数展开

在返回途中，`devfs_read_f` 释放 cdev 上的线程计数引用。如果 `destroy_dev(9)` 一直在等待 `si_threadcount` 达到零，它现在可以继续拆除了。

这就是为什么整个链的结构如此仔细。每个引用都是配对的；每次递增都有匹配的递减；处理程序触及的每段状态要么由处理程序拥有，要么由 softc 拥有，要么由每次打开的 `fh` 拥有。如果这些不变量中的任何一个被破坏，卸载就会变得不安全。

### 为什么这个追踪对你重要

三个要点。

**第一个**：上述机制是你的处理程序不需要做任何奇特操作就能与模块卸载共存的原因。只要你在有限时间内从 `d_read` 返回，内核就能让你的驱动干净地卸载。这是第九章在驱动级别保持所有读取为非阻塞的部分原因。

**第二个**：`read(2)` 和你的处理程序之间的每一层都是由内核在你的代码运行之前设置的。用户的缓冲区是有效的（否则 `uiomove` 会报告 `EFAULT`），cdev 是活跃的（否则 devfs 会拒绝调用），访问模式与描述符兼容（否则 `sys_read` 会拒绝），进程的凭据是当前线程的。你可以专注于你的驱动工作并信任这些层。

**第三个**：当你在源码树中阅读一个不熟悉的驱动，其 `d_read` 看起来很奇怪时，你可以反向追踪调用链。谁调用了这个处理程序？他们准备了什么状态？我的处理程序在返回时承诺了什么不变量？调用链会告诉你。答案通常与 `myfirst` 的相同。

### 镜像：追踪 write(2)

写入遵循相同类型的链，是镜像的。完整的七步分解大部分会是对读取追踪的重复，只是替换了词语，所以下面的段落是刻意压缩的。

用户调用 `write(fd, buf, 1024)`。C 库陷入内核。`/usr/src/sys/kern/sys_generic.c` 中的 `sys_write(2)` 验证参数，查找描述符，并获取其 `struct file` 的引用。文件操作向量分派到 `vn_write`，后者调用 devfs vnode 上的 `VOP_WRITE`。`/usr/src/sys/fs/devfs/devfs_vnops.c` 中的 `devfs_write_f` 获取 cdev 上的线程计数引用，从 `fp->f_flag` 组合 `ioflag`，并使用描述调用者缓冲区的 uio 调用 `cdevsw->d_write`。

你的 `d_write` 处理程序运行。它通过 `devfs_get_cdevpriv(9)` 检索每次打开的状态，检查活跃性，获取驱动在缓冲区周围需要的任何锁，将传输长度限制到可用空间，并调用 `uiomove(9)` 将字节从用户空间复制到内核缓冲区。成功时，处理程序更新其记账并返回零。`devfs_write_f` 释放线程计数引用。`vn_write` 通过 `sys_write` 展开，后者从 `uio_resid` 的减少量计算字节计数并返回它。用户看到 `write(2)` 的返回值。

与读取链在实质上有三个方面不同。

**首先，内核在 `uiomove` 内部运行 `copyin` 而不是 `copyout`。** 相同的机制，相反的方向。故障处理是相同的：错误的用户指针返回 `EFAULT`，短复制使 `uio_resid` 与实际传输的内容保持一致，处理程序只需传播错误代码。

**其次，`ioflag` 以相同方式携带 `IO_NDELAY`，但驱动的解释不同。** 在读取上，非阻塞意味着"如果没有数据则返回 `EAGAIN`"。在写入上，非阻塞意味着"如果没有空间则返回 `EAGAIN`"。对称的条件，对称的 errno 值。

**第三，`atime` / `mtime` 更新是方向特定的。** `devfs_read_f` 在字节移动时标记 `si_atime`；`devfs_write_f` 在字节移动时标记 `si_mtime`（在某些路径中还有 `si_ctime`）。这些是 `stat(2)` 在节点上报告的内容，也是为什么 `ls -lu /dev/myfirst/0` 对于读取和写入显示不同的时间戳。你的驱动不管理这些字段；devfs 管理。

一旦你认识到读取和写入追踪是镜像的，你就已经内化了字符设备分派路径的大部分。从这里开始的每一章都会在同一个链的略微不同的槽位上添加钩子（`d_poll`、`d_kqfilter`、`d_ioctl`、`mmap` 路径）。链本身保持不变。



## 实用工作流：从 shell 测试你的驱动

基本系统工具是你首先也是最好的测试工具。本节是一个简短的实地指南，介绍在开发驱动时如何很好地使用它们。下面的命令对你来说都不陌生，但将它们用于驱动工作有一种值得明确学习的节奏。

### cat(1)：第一次检查

`cat` 从其参数读取并写入标准输出。对于提供静态消息或已排空缓冲区的驱动，`cat` 是查看读取路径产生什么的最快方式：

```sh
% cat /dev/myfirst/0
```

如果输出符合预期，读取路径就是活跃的。如果为空，要么你的驱动没有内容可以提供（检查 `sysctl dev.myfirst.0.stats.bufused`），要么你的处理程序在第一次调用时返回 EOF。如果输出乱码，要么你的缓冲区未初始化，要么你在 `bufused` 之外传递了字节。

`cat` 打开其参数一次并从中读取直到 EOF。每次 `read(2)` 都是对你 `d_read` 的独立调用。使用 `truss(1)` 查看 `cat` 进行了多少次调用：

```sh
% truss cat /dev/myfirst/0 2>&1 | grep read
```

输出显示每次 `read(2)` 及其参数和返回值。如果你预期一次读取却看到三次，这告诉了你关于缓冲区大小的信息；如果你预期三次读取却看到一次，你的处理程序在单次调用中传递了所有数据。

### echo(1) 和 printf(1)：简单写入

`echo` 是将已知字符串放入驱动写入路径的最快方式：

```sh
% echo "hello" | sudo tee /dev/myfirst/0 > /dev/null
```

有两点需要注意。首先，`echo` 默认追加换行符；你发送的字符串是 6 字节，不是 5 字节。在需要时使用 `echo -n` 抑制换行符。其次，`tee` 调用是为了解决权限问题：shell 重定向（`>`）以用户权限运行，所以 `sudo echo > /dev/myfirst/0` 无法打开节点。通过在 `sudo` 下运行的 `tee` 管道传递，可以避开这个问题。

`printf` 给你更多控制：

```sh
% printf 'abc' | sudo tee /dev/myfirst/0 > /dev/null
```

三个字节，没有换行符。使用 `printf '\x41\x42\x43'` 生成二进制模式。

### dd(1)：精确工具

对于需要特定字节计数或特定块大小的任何测试，`dd` 是正确的工具。`dd` 也是基本系统工具中少数在其摘要中报告短读取和短写入的工具之一，这使其对测试驱动行为特别有用：

```sh
% sudo dd if=/dev/urandom of=/dev/myfirst/0 bs=128 count=4
4+0 records in
4+0 records out
512 bytes transferred in 0.001234 secs (415000 bytes/sec)
```

`X+Y records in` / `X+Y records out` 计数器有精确含义：`X` 是完整块传输的次数，`Y` 是短传输的次数。一行显示 `0+4 records out` 意味着每个块只被部分接受。那是驱动在告诉你某些信息。

`dd` 还允许你以已知块大小读取：

```sh
% sudo dd if=/dev/myfirst/0 of=/tmp/dump bs=64 count=1
```

这恰好发出一个 64 字节的 `read(2)`。你的处理程序看到 `uio_resid = 64`；你用你有的内容响应；结果就是 `dd` 写入 `/tmp/dump` 的内容。

`iflag=fullblock` 标志告诉 `dd` 在短读取时循环，直到填满请求的块。当你想吸收驱动的所有输出而不因短读取默认行为丢失字节时很有用。

### od(1) 和 hexdump(1)：字节级检查

对于驱动测试，`od` 和 `hexdump` 让你看到驱动发出的确切字节：

```sh
% sudo dd if=/dev/myfirst/0 bs=32 count=1 | od -An -tx1z
  68 65 6c 6c 6f 0a                                 >hello.<
```

`-An` 标志抑制地址打印。`-tx1z` 以十六进制和 ASCII 显示字节。如果预期输出是文本，你在右侧看到它；如果是二进制，你在左侧看到十六进制。

这些工具在读取产生意外字节时变得不可或缺。"它看起来很奇怪"和"我能以十六进制看到每个字节"是非常不同的调试状态。

### sysctl(8) 和 dmesg(8)：内核的声音

你的驱动通过 `sysctl` 发布计数器，通过 `dmesg` 发布生命周期事件。两者都值得在每次测试时检查：

```sh
% sysctl dev.myfirst.0
% dmesg | tail -20
```

sysctl 输出是你对驱动当前状态的视图。`dmesg` 是你对驱动自启动以来（或自环形缓冲区环绕以来）历史的视图。

一个有用的习惯：每次测试后，都运行两者。如果数字不符合你的预期，你就快速缩小了 bug 的范围。

### fstat(1)：谁打开了描述符？

当你的驱动拒绝卸载时（"module busy"），问题是"现在谁打开了 `/dev/myfirst/0`？"。`fstat(1)` 回答它：

```sh
% fstat -p $(pgrep cat) /dev/myfirst/0
USER     CMD          PID   FD MOUNT      INUM MODE         SZ|DV R/W NAME
ebrandi  cat          1234    3 /dev         0 crw-rw----  myfirst/0  r /dev/myfirst/0
```

或者，使用 `fuser(8)`：

```sh
% sudo fuser /dev/myfirst/0
/dev/myfirst/0:         1234
```

两种工具都能命名持有描述符的进程。终止罪魁祸首（小心；不要终止任何你没启动的进程）后模块就可以卸载了。

### truss(1) 和 ktrace(1)：观察系统调用

对于你想检查其与驱动交互的用户程序，`truss` 显示每个系统调用及其返回值：

```sh
% truss ./rw_myfirst
open("/dev/myfirst/0",O_WRONLY,0666)             = 3 (0x3)
write(3,"round-trip test payload\n",24)          = 24 (0x18)
close(3)                                         = 0 (0x0)
...
```

`ktrace` 记录到文件，稍后用 `kdump` 打印；当你想捕获长时间运行程序的追踪时，它是正确的工具。

这两个工具不是驱动特定的，但它们是你从外部确认驱动正在产生用户程序将看到的结果的方式。

### 建议的测试节奏

对于章节的每个阶段，尝试这个循环：

1. 构建并加载。
2. 使用 `cat` 产生初始输出，目视确认。
3. 使用 `sysctl dev.myfirst.0` 查看计数器是否匹配。
4. 使用 `dmesg | tail` 查看生命周期事件。
5. 用 `echo` 或 `dd` 写入一些内容。
6. 读回内容。
7. 用更大的尺寸、边界尺寸和异常尺寸重复测试。
8. 卸载。

经过几次迭代后，这变得自动化且快速。正是这种节奏将驱动开发从苦差事变成了例行公事。

### 一个具体的 truss 演示

在 `truss(1)` 下运行用户态程序是查看它对你的驱动发出了什么系统调用以及内核产生了什么返回值的最快方式之一。以下是加载了阶段 3 驱动且缓冲区为空时的典型会话：

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

有几件事值得停下来注意。每一行显示一个系统调用、其参数及其以十进制和十六进制表示的返回值。`write` 调用接收了 29 字节，驱动接受了全部 29 字节（返回值与请求长度匹配）。`read` 调用接收了 255 字节空间的缓冲区，驱动产生了 29 字节的内容；一个短读取，用户程序显式接受。两次 `open` 调用都返回了 3，因为文件描述符 0、1 和 2 是标准流，第一个空闲描述符是 3。

如果你通过限制驱动来强制短写入，`truss` 将清楚地显示它：

```sh
% truss ./write_big 2>&1 | head
open("/dev/myfirst/0",O_WRONLY,00)               = 3 (0x3)
write(3,"<8192 bytes of data>",8192)             = 4096 (0x1000)
write(3,"<4096 bytes of data>",4096)             ERR#28 'No space left on device'
close(3)                                         = 0 (0x0)
```

第一次写入请求了 8192 字节并被接受了 4096 字节。第二次写入没有什么可说的，因为缓冲区已满；驱动返回了 `ENOSPC`，`truss` 将其呈现为 `ERR#28 'No space left on device'`。这是用户侧的视图；你的驱动侧对第一次调用返回零（`uio_resid` 递减到 4096），对第二次调用返回 `ENOSPC`。将 `truss` 看到的与你的 `device_printf` 输出进行比较是捕获驱动意图与内核报告之间不匹配的绝佳方式。

`truss -f` 跟踪派生，当你的测试工具生成工作进程时很有用。`truss -d` 为每行添加相对时间戳前缀；用于推理调用之间的延迟。两个标志都是小的投入；当你开始运行多进程压力测试时，回报会迅速累积。

### 关于 ktrace 的简要说明

`ktrace(1)` 是 `truss` 的更大的兄弟。它将二进制追踪记录到文件（默认为 `ktrace.out`），然后用 `kdump(1)` 格式化。它是以下情况的正确工具：

- 测试运行很长，你不想实时观看输出。
- 你想捕获对 `truss` 来说太细粒度的细节（系统调用时间、信号传递、namei 查找）。
- 你想稍后重放追踪，也许在不同的机器上。

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

对于第九章，`truss` 和 `ktrace` 之间的差异很小。默认使用 `truss`；当你需要更多细节或记录的追踪时使用 `ktrace`。

### 用 vmstat -m 观察内核内存

你的驱动通过 `malloc(9)` 以 `M_DEVBUF` 类型分配内核内存。FreeBSD 的 `vmstat -m` 揭示每个类型桶中有多少活跃分配。在驱动加载且空闲时运行它，然后在它有已分配缓冲区时再次运行，增加量将在 `devbuf` 行中可见：

```sh
% vmstat -m | head -1
         Type InUse MemUse HighUse Requests  Size(s)
% vmstat -m | grep devbuf
       devbuf   415   4120K       -    39852  16,32,64,128,256,512,1024,2048,...
```

`InUse` 列是此类型的当前活跃分配计数。`MemUse` 是当前使用的总大小。`HighUse` 是自启动以来的历史高水位标记。`Requests` 是选择此类型的 `malloc` 调用的生命周期计数。

加载阶段 2 驱动。`InUse` 增加一（4096 字节缓冲区），`MemUse` 增加约 4 KiB，`Requests` 递增。卸载。`InUse` 减少一；`MemUse` 减少那 4 KiB。如果不是这样，你就有内存泄漏，而 `vmstat -m` 刚刚告诉了你。

这是值得添加到测试节奏中的第二个可观测性通道。`sysctl` 显示驱动拥有的计数器。`dmesg` 显示驱动拥有的日志行。`vmstat -m` 显示内核拥有的分配计数，它捕获一类前两个无法看到的 bug（忘记释放）。

对于通过 `MALLOC_DEFINE(M_MYFIRST, "myfirst", ...)` 声明自己 malloc 类型的驱动，`vmstat -m | grep myfirst` 更好：它将你的驱动的分配从通用 `devbuf` 池中隔离出来。`myfirst` 在本章中为简单起见一直使用 `M_DEVBUF`，但在将驱动发布到本书实验环境之外之前，升级到专用类型是一个你可能想做的小更改。



## 可观测性：让你的驱动可读

一个做正确事情的驱动，如果你能从内核外部确认它在做正确的事情，就更有价值。本节是对本章一直在做的可观测性选择的简短思考，以及为什么这样做。

### 三个接口：sysctl、dmesg、用户态

你的驱动向操作员呈现三个界面：

- **sysctl** 用于实时计数器：操作员可以轮询的即时值。
- **dmesg (device_printf)** 用于生命周期事件：打开、关闭、错误、转换。
- **/dev** 节点用于数据路径：实际的字节。

每个都有不同的角色。sysctl 告诉操作员*现在什么是真的*。dmesg 告诉操作员*最近发生了什么变化*。`/dev` 是操作员实际使用的东西。

一个可观测性好的驱动刻意使用所有三个。一个可观测性最小的驱动只使用第三个，调试它需要调试器或大量猜测。

### Sysctl：计数器与状态

`myfirst` 通过 `dev.myfirst.0.stats` 下的 sysctl 树暴露计数器：

- `attach_ticks`：即时值（驱动附加的时间）。
- `open_count`：单调递增计数器（生命周期打开次数）。
- `active_fhs`：实时计数（当前描述符数）。
- `bytes_read`、`bytes_written`：单调递增计数器。
- `bufused`：实时值（当前缓冲区占用）。

单调递增计数器比实时值更容易推理，因为即使绝对值没有意义，它们的变化率也是有用的信息。看到 `bytes_read` 以 1 MB/s 增长的操作员已经学到了某些东西，即使 1 MB/s 在上下文之外没有意义。

实时值在状态对决策有影响时是必不可少的（`active_fhs > 0` 意味着卸载将失败）。优先选择单调递增计数器，在需要时使用实时值。

### dmesg：值得查看的事件

`device_printf(9)` 写入内核消息缓冲区，`dmesg` 显示它。每一行都值得恰好看到一次：将 dmesg 用于事件，而不是用于连续状态。

`myfirst` 记录的事件：

- 附加（每个实例一次）。
- 打开（每次打开一次）。
- 析构函数（每个描述符关闭一次）。
- 拆离（每个实例一次）。

即每个实例每次加载/卸载周期四行，加上每次打开/关闭对两行。舒适。

我们不记录的内容：

- 每次 `read` 或 `write` 调用。那会在任何真实工作负载下淹没 dmesg。
- 每次 sysctl 读取。那些是被动的。
- 每次成功传输。sysctl 计数器携带该信息，而且携带得更紧凑。

如果驱动需要记录每秒发生很多次的事情，通常的答案是用 `if (bootverbose)` 保护日志，这样在生产系统上它是静默的，但对使用 `boot -v` 启动的开发者可用。对于 `myfirst`，我们甚至不需要那样做。

### 过度日志记录的陷阱

一个记录每次操作的驱动是一个将重要事件隐藏在噪音海洋中的驱动。如果你的 dmesg 显示一万行 `read returned 0 bytes`，那行说 `buffer full, returning ENOSPC` 的消息就是不可见的。

保持日志稀疏。记录转换，而不是状态。每个实例记录一次，而不是每次调用记录一次。有疑问时，保持静默。

### 你稍后要添加的计数器

第十章及以后将通过以下内容扩展计数器树：

- `reads_blocked`、`writes_blocked`：不得不睡眠的调用计数（第十章）。
- `poll_waiters`：活跃 `poll(2)` 订阅者计数（第十章）。
- `drain_waits`、`overrun_events`：环形缓冲区诊断（第十章）。

每一个都是操作员可以查看以了解驱动正在做什么的又一个东西。模式是相同的：暴露计数器，保持机制静默，让操作员决定何时检查。

### 你的驱动在轻负载下的样子

一个具体的例子比抽象的建议更有用。加载阶段 3，从另一个终端用 `sysctl dev.myfirst.0.stats` 监控运行伴随的 `stress_rw` 程序几秒钟，你会看到类似这样的内容：

**`stress_rw` 启动前：**

```text
dev.myfirst.0.stats.attach_ticks: 12345678
dev.myfirst.0.stats.open_count: 0
dev.myfirst.0.stats.active_fhs: 0
dev.myfirst.0.stats.bytes_read: 0
dev.myfirst.0.stats.bytes_written: 0
dev.myfirst.0.stats.bufused: 0
```

零活动，一次附加，缓冲区为空。

**`stress_rw` 运行期间，使用 `watch -n 0.5 sysctl dev.myfirst.0.stats`：**

```text
dev.myfirst.0.stats.attach_ticks: 12345678
dev.myfirst.0.stats.open_count: 2
dev.myfirst.0.stats.active_fhs: 2
dev.myfirst.0.stats.bytes_read: 1358976
dev.myfirst.0.stats.bytes_written: 1359040
dev.myfirst.0.stats.bufused: 64
```

两个活跃描述符（写入者 + 读取者），计数器攀升，缓冲区持有 64 字节的在途数据。`bytes_written` 略微领先于 `bytes_read`，这正是你期望的：写入者产生了读取者尚未完全消耗的一块数据。差值等于 `bufused`。

**`stress_rw` 退出后：**

```text
dev.myfirst.0.stats.attach_ticks: 12345678
dev.myfirst.0.stats.open_count: 2
dev.myfirst.0.stats.active_fhs: 0
dev.myfirst.0.stats.bytes_read: 4800000
dev.myfirst.0.stats.bytes_written: 4800000
dev.myfirst.0.stats.bufused: 0
```

两个描述符都已关闭。生命周期打开次数为 2（累计）。活跃数为 0。`bytes_read` 等于 `bytes_written`；读取者已完全赶上。缓冲区为空。

三个特征值得注意。首先，`active_fhs` 始终跟踪活跃描述符；它是实时值，不是累计计数器。其次，当读取者跟上时，稳态下 `bytes_read == bytes_written`，加上 `bufused` 中的任何内容。第三，`open_count` 是一个永不减少的生命周期值；发现搅动的一个快速方式是观察它增长而 `active_fhs` 保持稳定。

一个在负载下行为可预测的驱动是你能自信操作的驱动。一旦计数器按照本段描述的方式排列，你就拥有了你的第一个真正的驱动，而不是一个玩具。



## 有符号、无符号和差一的危害

关于一类几乎比任何其他类型都导致了更多内核崩溃的 bug 的简短章节。它在 I/O 处理程序中尤其频繁出现。

### ssize_t 与 size_t

两种类型主导 I/O 代码：

- `size_t`：无符号，用于大小和计数。`sizeof(x)` 返回 `size_t`。`malloc(9)` 接受 `size_t`。`memcpy` 接受 `size_t`。
- `ssize_t`：有符号，用于值可能为负的情况（通常 -1 表示错误）。`read(2)` 和 `write(2)` 返回 `ssize_t`。`uio_resid` 是 `ssize_t`。

这两种类型在 FreeBSD 支持的每个平台上具有相同的宽度，但它们不会在没有警告的情况下在彼此之间静默转换，并且在算术下溢时行为非常不同。

`size_t` 值的减法如果会产生负数结果，则会回绕成一个巨大的正数，因为 `size_t` 是无符号的。例如：

```c
size_t avail = sc->buflen - sc->bufused;
```

如果 `sc->bufused` 大于 `sc->buflen`，`avail` 将是一个巨大的数字，下一次 `uiomove` 会尝试一个超过缓冲区末尾的传输。

防御手段就是不变量。在本章的每个缓冲区管理部分，我们维护 `sc->bufhead + sc->bufused <= sc->buflen`。只要这个不变量成立，`sc->buflen - (sc->bufhead + sc->bufused)` 就不可能下溢。

风险在于意外违反不变量的代码路径。一个恢复了已消耗值的重复释放；一个两次更新 `bufused` 的写入；写入者之间的竞争。这些是当 `avail` 看起来不对时要寻找的 bug。

### uio_resid 可以与无符号值比较

`uio_resid` 是 `ssize_t`。你的缓冲区大小是 `size_t`。像这样的代码：

```c
if (uio->uio_resid > sc->buflen) ...
```

将以有符号与无符号比较的方式编译。现代编译器对此发出警告；应该认真对待这个警告。

更安全的模式是显式转换：

```c
if ((size_t)uio->uio_resid > sc->buflen) ...
```

或者使用我们一直在用的 `MIN`：

```c
towrite = MIN((size_t)uio->uio_resid, avail);
```

这个转换是合理的，因为 `uio_resid` 在有效 uio 中被文档化为非负的（并且 `uiomove` 对其进行 `KASSERT`）。转换使编译器满意并使意图明确。

### 计数器中的差一错误

在错误检查的错误一侧更新的计数器是一个经典 bug：

```c
sc->bytes_read += towrite;          /* BAD: happens even on error */
error = uiomove(sc->buf, towrite, uio);
```

正确的形状是在成功后递增：

```c
error = uiomove(sc->buf, towrite, uio);
if (error == 0)
        sc->bytes_read += towrite;
```

这就是为什么我们在章节中用 `if (error == 0)` 保护每个计数器更新。代价是一行代码。好处是你的计数器与实际情况匹配。

### uio_offset - before 惯用法

当你想知道"`uiomove` 实际移动了多少字节？"时，最干净的方式是比较前后的 `uio_offset`：

```c
off_t before = uio->uio_offset;
error = uiomove_frombuf(sc->buf, sc->buflen, uio);
size_t moved = uio->uio_offset - before;
```

这对完整和短传输都有效。`moved` 是实际的字节计数，不管调用者请求了多少或有多少可用。

这个惯用法在运行时是免费的（两次减法），在代码中是明确的。当你的驱动想计数字节时使用它；替代方案——从 `uio_resid` 推断计数——需要知道原始请求大小，这需要更多簿记。



## 额外故障排除：边界情况

扩展前面的故障排除部分，以下是你第一次编写真正驱动时可能遇到的更多场景。

### "同一描述符上的第二次读取返回零"

对于静态消息驱动（阶段 1）是预期的：一旦 `uio_offset` 到达消息末尾，`uiomove_frombuf` 返回零。

对于 FIFO 驱动（阶段 3）是意外的：第一次读取排空了缓冲区，没有写入者重新填充它。调用者不应该在没有写入发生的情况下连续发出第二次读取。

要区分这两种情况，检查 `sysctl dev.myfirst.0.stats.bufused`。如果为零，缓冲区为空。如果非零但你仍然看到零字节，你有一个 bug。

### "驱动在缓冲区有数据时立即返回零字节"

读取处理程序走了错误的分支。常见原因：

- `bufused == 0` 检查放在了错误的位置。如果检查在每次打开状态检索之前运行，它可能在真正工作之前就短路了读取。
- 处理程序中较早处有一个意外的 `return 0;`（例如，之前实验留下的调试分支）。
- 错误路径上缺少 `mtx_unlock`，使后续每次调用永远阻塞在互斥锁上。症状：第二次调用挂起，而不是零字节返回；但值得检查。

### "我的 `uiomove_frombuf` 无论缓冲区如何都总是返回零"

两个常见原因：

- `buflen` 参数为零。如果 `buflen <= 0`，`uiomove_frombuf` 立即返回零。
- `uio_offset` 已经达到或超过 `buflen`。在这种情况下，`uiomove_frombuf` 返回零以发出 EOF 信号。

在入口处添加一个 `device_printf` 记录参数以确认你属于哪种情况。

### "缓冲区溢出到相邻内存"

你的算术是错误的。某处你调用了 `uiomove(sc->buf + X, N, uio)` 其中 `X + N > sc->buflen`。写入静默进行并损坏内核内存。

你的内核通常随后会崩溃，可能在一个完全不相关的子系统中。崩溃消息不会提及你的驱动；它会提及被破坏的堆邻居。

如果你怀疑这一点，用 `INVARIANTS` 和 `WITNESS`（在许多目标上还有 amd64 上的 KASAN）重新构建。这些内核特性比默认内核更早捕获缓冲区溢出。

### "从设备读取的进程永远挂起"

由于第九章不实现阻塞 I/O，这不应该在 `myfirst` 阶段 3 中发生。如果发生了，最可能的原因是进程在你尝试卸载驱动时持有一个文件描述符；`destroy_dev(9)` 正在等待 `si_threadcount` 达到零，而进程因某种原因停留在你的处理程序内部。

诊断方法：`ps auxH | grep <your-test>`；`gdb -p <pid>` 和 `bt`。栈应该揭示线程停在哪里。

如果你的阶段 3 处理程序意外睡眠（例如，因为你在提前实验第十章材料时添加了 `tsleep`），修复方法是移除睡眠。第九章的驱动不阻塞。

### "`kldunload` 提示 `kldunload: can't unload file: Device busy`"

描述符仍然打开的经典症状。使用 `fuser /dev/myfirst/0` 找到有问题的进程，关闭描述符或终止进程，然后重试。

### "我修改了驱动，`make` 编译成功但 `kldload` 因版本不匹配而失败"

你的构建环境与运行中的内核不匹配。检查：

```sh
% freebsd-version -k
14.3-RELEASE
% ls /usr/obj/usr/src/amd64.amd64/sys/GENERIC
```

如果 `/usr/src` 是不同版本的，你的头文件产生的模块会被内核拒绝。用匹配的源码重新构建。在实验虚拟机中，这通常意味着通过 `fetch` 或 `freebsd-update src-install` 将 `/usr/src` 与运行中的版本同步。

### "我看到通过设备写入的每个字节在 dmesg 中打印了两次"

你在热路径中有一个打印每次传输的 `device_printf`。移除它或用 `if (bootverbose)` 保护它。

同一个 bug 的更温和版本：打印每次传输长度的单行日志。对于小型测试工作负载看起来没问题；对于真正的用户工作负载，它会淹没 dmesg 并导致内核缓冲区中的时间戳压缩。

### "我的 `d_read` 被调用了但 `d_write` 没有被调用"

要么是用户程序从未在设备上调用 `write(2)`，要么是调用 `write(2)` 时描述符没有以写入方式打开（`O_RDONLY`）。检查两者。

另外：确认 `cdevsw.d_write` 被分配给了 `myfirst_write`。将其分配给 `myfirst_read` 的复制粘贴 bug 会导致两个方向都命中读取处理程序，产生可预见的混乱结果。



## 设计说明：为什么每个阶段停在它停的地方

一个简短的元节，解释第九章的三个阶段为什么有这些边界。这是值得明确的章节设计推理类型，因为这是你在设计自己的驱动时将应用的推理。

### 为什么存在阶段 1

阶段 1 是不是 `/dev/null` 的最小可能 `d_read`。它介绍了：

- `uiomove_frombuf(9)` 辅助函数，将固定缓冲区输出到用户空间的最简单方式。
- 每次描述符偏移处理。
- 使用 `uio_offset` 作为状态载体的模式。

阶段 1 不对写入做任何事情；第八章的桩函数就可以。

没有阶段 1，从桩函数到缓冲区读/写驱动的跳跃就太大了。阶段 1 让你用最少的代码确认读取处理程序已正确连接。其他一切都建立在那个确认之上。

### 为什么存在阶段 2

阶段 2 介绍了：

- 动态分配的内核缓冲区。
- 接受用户数据的写入路径。
- 遵守调用者偏移穿越累积缓冲区的读取路径。
- softc 互斥锁在 I/O 处理程序中的首次实际使用。

阶段 2 刻意不排空读取。缓冲区增长直到满；后续写入返回 `ENOSPC`。这让两个并发读取者可以确认它们各自有自己的 `uio_offset`，这是阶段 1 无法演示的属性（因为阶段 1 没有可写入的内容）。

### 为什么存在阶段 3

阶段 3 介绍了：

- 排空缓冲区的读取。
- 头指针和已用计数之间的协调。
- 大多数真正驱动近似使用的 FIFO 语义。

阶段 3 不会环绕。头指针和已用指针在缓冲区中向前移动，当缓冲区为空时折叠回开头。真正的环形缓冲区（头和尾在固定大小数组中环绕）属于第十章，因为它与阻塞读取和 `poll(2)` 自然配对：环形使稳态操作高效，而高效的稳态操作正是阻塞读取者所需要的。

### 为什么这里没有环形缓冲区

环形缓冲区比阶段 3 多五到十五行的额外簿记。现在添加它不会是大量的代码。推迟的原因是教学性的：两个概念（"I/O 路径语义"和"环形缓冲区机制"）对初学者来说各自都容易混淆，将它们分成两章让每章一次解决一堆困惑。

到第十章引入环形时，读者已经熟练掌握 I/O 路径。新的材料只是环形簿记。

### 为什么没有阻塞

阻塞是有用的，但它引入了 `msleep(9)`、条件变量、`d_purge` 拆卸钩子，以及围绕何时唤醒和唤醒什么的大量正确性问题。其中每一个都是实质性的主题。将它们混入第九章会使长度翻倍、清晰度减半。

第十章的第一节是"当你的驱动必须等待时"。这是一个自然的延续。

### 各阶段**不**试图成为什么

这些阶段不是硬件驱动的模拟。它们不模拟 DMA。它们不模拟中断。它们不假装成它们不是的东西：它们是锻炼 UNIX I/O 路径的内存中驱动。

这很重要，因为在本书后面，当我们编写真正的硬件驱动时，I/O 路径看起来会是相同的。硬件细节（字节从哪里来，字节到哪里去）会改变，但处理程序形状、uiomove 用法、errno 约定、计数器模式，所有这些都将从第九章中可以识别。

一个正确跨越用户/内核信任边界移动字节的驱动是任何真正驱动的 80%。第九章教你那 80%。



## 动手实验

下面的实验跟踪上面的三个阶段。每个实验都是一个检查点，证明你的驱动正在做文本刚才描述的事情。在开始之前完整阅读实验，并按顺序进行。

### 实验 9.1：构建并加载阶段 1

**目标：** 构建阶段 1 驱动，加载它，读取静态消息，并确认每次描述符的偏移处理。

**步骤：**

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
   你应该会看到第 8 章的 `open via /dev/myfirst/0 fh=...` 和 `per-open dtor fh=...` 行，以及消息正文已被读取的信息。
8. Unload:
   ```sh
   % sudo kldunload myfirst
   ```

**成功标准：**

- `cat` 打印消息。
- 用户态工具在第一次读取时显示 75 字节，第二次读取时显示 0 字节。
- `dmesg` 为每次 `./rw_myfirst read` 调用显示一次打开和一次析构。

**常见错误：**

- 忘记 `sizeof(myfirst_message) - 1` 中的 `-1`。消息将包含一个作为杂散字符出现在用户输出中的末尾 NUL 字节。
- 在 `sc == NULL` 检查之前没有调用 `devfs_get_cdevpriv`。本章的其余部分依赖于这个顺序；运行它来看看为什么这是正确的。
- 使用 `(void *)sc->message` 而不是 `__DECONST(void *, sc->message)`。两者在大多数编译器上都能工作；`__DECONST` 形式是约定并在某些编译器配置上抑制警告。

### 实验 9.2：用写入和读取练习阶段 2

**目标：** 构建阶段 2，从用户态推入数据，拉回出来，并观察 sysctl 计数器。

**步骤：**

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
   预期会出现短写错误。检查 `sysctl dev.myfirst.0.stats.bufused`；它应该是 4096（缓冲区大小）。
8. Confirm reads still deliver the content:
   ```sh
   % sudo cat /dev/myfirst/0 | od -An -c | head -3
   ```
9. Unload:
   ```sh
   % sudo kldunload myfirst
   ```

**成功标准：**

- 写入存入字节；读取传回它们。
- `bufused` 与自上次重置以来写入的字节数匹配。
- 当缓冲区填满时 `dd` 表现出短写入；驱动返回 `ENOSPC`。
- `dmesg` 为每个打开设备的进程显示打开和析构行。

**常见错误：**

- 忘记在 `detach` 中释放 `sc->buf`。驱动将无提示地卸载，但后续的内核内存泄漏检查（`vmstat -m | grep devbuf`）将显示漂移。
- 在调用 `uiomove` 时持有 softc 互斥锁，但不确定互斥锁是 `MTX_DEF` 而不是自旋锁。第七章的 `mtx_init(..., MTX_DEF)` 是正确的选择；不要更改它。
- 在 `attach` 中省略 `sc->bufused = 0` 重置。Newbus 为你将 softc 初始化为零，但使初始化显式是约定；它也使后续重构更不容易出错。

### 实验 9.3：阶段 3 FIFO 行为

**目标：** 构建阶段 3，从两个终端练习 FIFO 行为，并确认读取排空缓冲区。

**步骤：**

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
   预期没有输出。缓冲区是空的。
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
   预期两行会拼接在一起。两次写入在任一读取发生之前追加到了同一个缓冲区。
8. Inspect the counters:
   ```sh
   % sysctl dev.myfirst.0.stats
   ```
   `bufused` should be back to zero. `bytes_read` and `bytes_written` should match.
9. Unload:
   ```sh
   % sudo kldunload myfirst
   ```

**成功标准：**

- 写入追加到缓冲区；读取排空它。
- 缓冲区排空后的读取立即返回（空时 EOF）。
- 一旦读取者跟上，`bytes_read` 总是等于 `bytes_written`。

**常见错误：**

- 当 `bufused` 达到零时不重置 `bufhead = 0`。缓冲区将向 `sc->buf` 末尾"漂移"，在满之前很久就拒绝写入。
- 忘记在读取排空时更新 `bufhead`。驱动将重复读取相同的字节。
- 使用 `uio->uio_offset` 作为每次描述符的偏移。在 FIFO 中，偏移是共享的；每次描述符偏移没有意义，会使测试者困惑。

### 实验 9.4：使用 dd 测量传输行为

**目标：** 使用 `dd(1)` 生成已知大小的传输，读回结果，并检查计数器是否一致。

`dd` 是这里的工具选择，因为它让你控制块大小、块数量和短传输上的行为。

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
   驱动接受了 8192 字节请求中的 4096 字节（缓冲区大小），其余部分返回了短写入。
7. Alternatively, use `bs=4096` with `count=2`:
   ```sh
   % sudo dd if=/dev/urandom of=/dev/myfirst/0 bs=4096 count=2
   dd: /dev/myfirst/0: No space left on device
   1+0 records in
   0+0 records out
   4096 bytes transferred
   ```
   第一个 4096 字节的块完全成功；第二个块以 `ENOSPC` 失败。
8. Drain:
   ```sh
   % sudo dd if=/dev/myfirst/0 of=/tmp/out bs=4096 count=1
   % sudo kldunload myfirst
   ```

**成功标准：**

- `dd` 在每一步报告预期的字节计数。
- 驱动接受最多 4096 字节，对剩余部分返回 `ENOSPC`。
- `bufused` 在每次操作后跟踪缓冲区状态。

### 实验 9.5：一个小型往返 C 程序

**目标：** 编写一个短的用户态 C 程序，打开设备，写入已知字节，关闭描述符，再次打开，读回字节，并验证它们匹配。

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

**成功标准：**

- 程序打印 `round-trip OK: 24 bytes`。
- `dmesg` 显示一次写入的打开/析构对和一次读取的打开/析构对。

**常见错误：**

- 写入的字节少于载荷且未检查返回值。`write(2)` 可以返回短计数；你的测试必须处理它。
- 忘记 `O_WRONLY` 与 `O_RDONLY` 的区别。`open(2)` 根据节点的访问位强制执行模式；以错误模式打开返回 `EACCES`（或类似的错误）。
- 假设 `read(2)` 返回请求的计数。它可以返回更少；同样，调用者需要循环。

### 实验 9.6：检查二进制往返

**目标：** 通过推送随机字节并检查相同字节返回，确认驱动处理任意二进制数据，而不仅仅是文本。

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

**成功标准：**

- `cmp` 报告没有差异。
- 驱动保留输入的每一位。
- 没有字节顺序问题，没有"有帮助的"解释，没有意外的转换。

这个实验很短但很重要：它验证你的驱动是一个透明的字节存储，而不是一个意外特殊解释某些字节的文本过滤器。如果你在发送和接收的文件之间看到差异，你的传输路径中有 bug，可能是长度误算或缓冲区算术中的差一错误。

### 实验 9.7：端到端观察运行中的驱动

**目标：** 将 sysctl、dmesg、truss 和 vmstat 组合成对阶段 3 驱动在真实负载下的单一端到端观察。这个实验没有新代码；它是从"我写了驱动"到"我能看到它在做什么"的桥梁。

**步骤：**

1. With Stage 3 loaded fresh, open four terminals. Terminal A will run the driver load / unload cycles. Terminal B will monitor sysctl. Terminal C will tail dmesg. Terminal D will run a user workload.
2. **Terminal A:**
   ```sh
   % sudo kldload ./myfirst.ko
   % vmstat -m | grep devbuf
   ```
   注意 `devbuf` 行的 `InUse` 和 `MemUse` 值。
3. **Terminal B:**
   ```sh
   % watch -n 1 sysctl dev.myfirst.0.stats
   ```
4. **Terminal C:**
   ```sh
   % sudo dmesg -c > /dev/null
   % sudo dmesg -w
   ```
   `-c` 清除累积的消息；`-w` 监视新消息。
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
   观察终端 B。你应该会看到 `bufused` 在振荡，计数器在攀升，并且测试运行期间 `active_fhs` 达到了 2。
10. When the stress run finishes, in terminal B verify `active_fhs` is 0. In terminal A,
    ```sh
    % sudo kldunload myfirst
    % vmstat -m | grep devbuf
    ```
    `InUse` should have returned to its pre-load baseline. If it has not, your driver leaked an allocation and `vmstat -m` just told you.

**成功标准：**

- Sysctl 计数器与你运行的工作负载匹配。
- Dmesg 为每个描述符的打开/关闭显示一个打开/析构对。
- Truss 输出与你对程序所做操作的心理模型匹配。
- `vmstat -m | grep devbuf` 在卸载后回到其基线。
- 没有崩溃、没有警告、没有无法解释的计数器漂移。

**为什么这个实验重要：** 这是第一个同时锻炼完整可观测性工具链的实验。在生产中，出问题的信号几乎从不来自崩溃；它来自一个超出范围的计数器、一条没人预期的 `dmesg` 行，或一个与现实不匹配的 `vmstat -m` 读数。建立同时查看所有四个界面的习惯是区分"我写了一个驱动"和"我对一个驱动负责"的关键。



## 挑战练习

这些挑战扩展了材料而没有引入属于后续章节的主题。每一个都只使用我们已介绍的原语。在看伴随树之前先尝试它们；学习在于尝试，而不在于答案。

### 挑战 9.1：每次描述符的读取计数器

扩展阶段 2，使每次描述符的 `reads` 计数器通过 sysctl 暴露。计数器应该对每个活跃描述符可用，这意味着是每次 `fh` 的 sysctl 而不是每次 softc 的。

这个挑战比看起来更难：sysctl 在 softc 生命周期的已知点分配和释放，而每次描述符的结构只在其描述符存活期间存在。一个干净的解决方案在 `d_open` 中为每个 `fh` 注册一个 sysctl 节点，并在析构函数中取消注册。注意生命周期；sysctl 上下文必须在 `fh` 内存之前释放。

*提示：* `sysctl_ctx_init` 和 `sysctl_ctx_free` 是每次上下文的。你可以给每个 `fh` 自己的上下文，并在析构函数中释放它。

*替代方案：* 在 softc 中（在互斥锁下）保持一个 `fh` 指针链表，并通过按需遍历链表的自定义 sysctl 处理程序暴露它。这是 `/usr/src/sys/kern/tty_info.c` 用于每次进程统计的模式。

### 挑战 9.2：一个支持 readv(2) 的测试

编写一个用户程序，使用 `readv(2)` 从驱动读取到三个大小分别为 8、16 和 32 字节的独立缓冲区。确认驱动按顺序将字节传递到所有三个缓冲区。

内核和 `uiomove(9)` 已经处理 `readv(2)`；驱动不需要更改。这个挑战的目的是让你自己确信这个事实。

*提示：* `struct iovec iov[3] = {{buf1, 8}, {buf2, 16}, {buf3, 32}};`，然后 `readv(fd, iov, 3)`。返回值是所有三个缓冲区传递的总字节数；用户端不会修改各个 `iov_len` 值。

### 挑战 9.3：短写入演示

修改阶段 2 的 `myfirst_write` 使其每次调用最多接受 128 字节，不管 `uio_resid` 是多少。写入 1024 字节的用户程序应该每次看到 128 字节的短写入。

然后编写一个短测试程序，在单次 `write(2)` 调用中写入 1024 字节，观察短写入返回值，并循环直到所有 1024 字节都被接受。

值得思考的问题：

- `cat` 是否正确处理短写入？（是的。）
- `echo > /dev/myfirst/0 "..."` 是否正确处理它们？（通常通过 shell 中的 `printf` 处理，但有时不会；值得测试。）
- 如果你移除短写入行为并尝试超过缓冲区大小会发生什么？（在第一次 4096 字节写入后你会得到 `ENOSPC`。）

这个挑战教你区分"驱动做正确的事"和"用户程序假设驱动做什么"。

### 挑战 9.4：一个 ls -l 传感器

使驱动对读取的响应取决于设备本身的 `ls -l` 输出。即：每次读取产生设备节点的当前时间戳。

*提示：* `sc->cdev->si_ctime` 和 `sc->cdev->si_mtime` 是 cdev 上的 `struct timespec` 字段。你可以用 `printf` 格式化将它们转换为字符串，将字符串放入内核缓冲区，然后用 `uiomove_frombuf(9)` 输出。

*警告：* `si_ctime` / `si_mtime` 可能会在节点被触及时由 devfs 更新。观察当你 `touch /dev/myfirst/0` 并再次读取时会发生什么。

### 挑战 9.5：一个反向回显驱动

修改阶段 3，使每次读取以写入顺序的相反顺序返回字节。写入 `"hello"` 后跟读取应该产生 `"olleh"`。

这个挑战完全关于缓冲区簿记。`uiomove` 调用保持不变；你改变传递给它们的地址。

*提示：* 你可以在每次读取时反转缓冲区（昂贵），或者在写入端以相反顺序存储字节（更便宜）。两者都不是"正确"的答案；每个都有不同的正确性和并发属性。选择一个并在注释中论证它。

### 挑战 9.6：二进制往返

编写一个用户程序，向驱动写入一个 `struct timespec`，然后读回一个。比较两个结构。它们相等吗？它们应该相等，因为 `myfirst` 是一个透明字节存储。

扩展程序写入两个 `struct timespec` 值，然后 `lseek(fd, sizeof(struct timespec), SEEK_SET)` 并读取第二个。会发生什么？（线索：FIFO 不有意义地支持寻址。）

这个挑战说明了安全数据传输部分的"读取和写入携带字节，而不是类型"的观点。字节完美往返；类型信息则不会。

### 挑战 9.7：一个十六进制查看测试工具

编写一个短 shell 脚本，给定字节计数 N，用 `dd if=/dev/urandom bs=$N count=1` 生成 N 个随机字节，通过管道传入你的阶段 3 驱动，然后用 `dd if=/dev/myfirst/0 bs=$N count=1` 读回它们，并用 `cmp` 比较两个流。脚本应该对匹配的流报告成功，对不匹配的流报告类似 diff 的输出。用 N = 1, 2, 4, ..., 4096 运行它来覆盖小型、边界和容量填满的大小。

运行扫描时要回答的问题：

- 每个大小（包括 4096）是否都能干净地往返？
- 在 4097 时，驱动做什么？测试工具是否有意义地报告错误？
- 是否有任何大小使得 `cmp` 报告差异？如果有，根本原因是什么？

这个挑战奖励你在实用工作流部分组合工具：`dd` 用于精确传输，`cmp` 用于字节级验证，`sysctl` 用于计数器，shell 用于编排。像这样健壮的测试工具是每次重构驱动并想快速知道行为是否仍然正确时都会回报的习惯。

### 挑战 9.8：谁打开了描述符？

编写一个小型 C 程序，打开 `/dev/myfirst/0`，在 `pause()` 上阻塞（从而无限期持有描述符），运行直到收到 `SIGTERM`。在第二个终端中，运行 `fstat | grep myfirst` 然后运行 `fuser /dev/myfirst/0`。注意输出。现在尝试 `kldunload myfirst`。你得到什么错误？为什么？

现在用 `SIGTERM` 或普通 `kill` 终止持有者。观察 `dmesg` 中析构函数的触发。再次尝试 `kldunload`。它应该成功。

这个挑战很短，但它巩固了本章一个更微妙的不变量：当任何描述符在其某个 cdev 上打开时，驱动无法卸载，FreeBSD 给操作员提供了一套标准工具来找到持有者。下次真正的 `kldunload` 因 `EBUSY` 失败时，你将已经见过这种问题的形状。



## 常见错误故障排除

你可能犯的每个 `d_read` / `d_write` 错误都属于少数几个类别之一。本节是一个简短的实地指南。

### "我的驱动即使写入了数据也返回零字节"

这通常是两个 bug 之一。

**Bug 1**：你在成功的 `uiomove` 之后忘记更新 `bufused`（或等价物）。写入到达了，字节移动了，但驱动的状态从未反映到达。下一次读取看到 `bufused == 0` 并报告 EOF。

修复：总是在 `uiomove` 返回后在 `if (error == 0) { ... }` 内更新你的跟踪字段。

**Bug 2**：你在不合适的地方重置了 `bufused`（或 `bufhead`）。一个常见的模式是在 `d_open` 或 `d_close` 中添加重置行"为了整洁"。那会擦除前一个调用者写入的数据。

修复：仅在 `attach`（加载时）或 `detach`（卸载时）中重置驱动范围的状态。每次描述符的状态属于 `fh`，由 `malloc(M_ZERO)` 重置并由析构函数清理。

### "我的读取返回垃圾"

缓冲区未初始化。不带 `M_ZERO` 的 `malloc(9)` 返回内容未定义的内存块。如果你的 `d_read` 越过 `bufused`，或从尚未写入的偏移读取，你看到的字节是内核回收的任何内存的残留。

修复：在 `attach` 中总是传递 `M_ZERO` 给 `malloc`。总是将读取限制在当前高水位标记（`bufused`），而不是缓冲区的总大小（`buflen`）。

这个 bug 有一个更严重的变体。将未初始化的内核内存返回给用户空间的驱动刚刚将内核状态泄露到了用户空间。在开发中这是一个 bug；在生产中这是一个安全漏洞和一个 CVE。

### "内核在用户地址上发生页错误而崩溃"

你直接在用户指针上调用了 `memcpy` 或 `bcopy`，而不是通过 `uiomove` / `copyin` / `copyout`。访问出错，内核没有安装故障处理程序，结果是崩溃。

修复：永远不要直接解引用用户指针。通过 `uiomove(9)`（在处理程序中）或 `copyin(9)` / `copyout(9)`（在其他上下文中）路由。

### "驱动拒绝卸载"

你至少有一个文件描述符仍然打开。当 `active_fhs > 0` 时 `detach` 返回 `EBUSY`；在所有 `fh` 被销毁之前模块不会卸载。

修复：在用户态关闭描述符。如果后台进程持有它，终止进程（在确认它是你的之后；不要终止系统守护进程）。`fstat -p <pid>` 显示进程打开了哪些文件；`fuser /dev/myfirst/0` 显示哪些进程打开了节点。

第十章将为需要强制阻塞读取者退出的驱动引入 `destroy_dev_drain` 模式。第九章不阻塞，所以这个问题在正常操作中不会出现；当它出现时，是因为用户态在某个意外的地方持有描述符。

### "我的写入处理程序返回 EFAULT"

你的 `uiomove` 调用遇到了无效的用户地址。常见原因：

- 用户程序调用了 `write(fd, NULL, n)` 或 `write(fd, (void*)0xdeadbeef, n)`。
- 用户程序写入了一个它已释放的指针。
- 你意外地将内核指针作为目标传递给 `uiomove`。如果你手工构建了一个用于内核空间数据的 uio，然后将其传递给期望用户空间 uio 的处理程序，就会发生这种情况。由此产生的 `copyout` 看到一个实际上是内核地址的"用户"地址；取决于架构，你要么得到 `EFAULT`，要么得到微妙的损坏。

修复：检查 `uio->uio_segflg`。对于用户驱动的处理程序，它应该是 `UIO_USERSPACE`。如果你传递的是内核空间 uio，确保 `uio_segflg == UIO_SYSSPACE` 并且你的代码路径知道区别。

### "并发写入下我的计数器是错误的"

两个写入者在 `bufused` 上竞争。每个都读取当前值，添加到它，然后写回，第二个写入者用陈旧的值覆盖了第一个写入者的更新。

修复：在共享状态的每次读-修改-写周围获取 `sc->mtx`。第三部分将此作为一等主题；对于第九章，整个临界区周围的一个互斥锁就足够了。

### "sysctl 计数器不反映真实状态"

两个变体。

**变体 A**：计数器是 `size_t`，但 sysctl 宏是 `SYSCTL_ADD_U64`。在 32 位架构上，宏读取 8 字节而字段只有 4 字节宽；值的一半是垃圾。

修复：将 sysctl 宏与字段类型匹配。`size_t` 在 32 位平台上与 `SYSCTL_ADD_UINT` 配对，在 64 位平台上与 `SYSCTL_ADD_U64` 配对。为了可移植性，使用 `uint64_t` 作为计数器并在更新时转换。

**变体 B**：计数器从未更新，因为更新在 `if (error == 0)` 块内，而 `uiomove` 返回了非零错误。这实际上是正确的行为：你不应该计算你没有移动的字节。症状只在你想用计数器调试错误时看起来像 bug。

修复：添加一个 `error_count` 计数器，在每次非零返回时递增，独立于 `bytes_read` 和 `bytes_written`。对调试有用。

### "全新加载后的第一次读取返回零字节"

通常是有意的。在阶段 3 中，空缓冲区返回零字节。如果你期望阶段 1 的静态消息，检查你运行的是阶段 1 驱动，而不是更高版本。

如果是无意的，仔细检查 `attach` 是否按预期设置了 `sc->buf`、`sc->buflen` 和 `sc->message_len`。一个常见的 bug 是从阶段 1 复制粘贴附加代码到阶段 2 并保留了 `sc->message = ...` 赋值，然后它优先于 `malloc` 行。

### "构建失败，提示未知的 uiomove_frombuf 引用"

你忘记包含 `<sys/uio.h>`。将它添加到 `myfirst.c` 的顶部。

### "我的处理程序对一次 read(2) 被调用了两次"

几乎肯定不是。更可能的是：你的处理程序被调用了一次，`uio_iovcnt > 1`（一个 `readv(2)` 调用），在 `uiomove` 内部每个 iovec 条目依次被排空。`uiomove` 中的内部循环可能在处理程序的单次调用中进行多次 `copyout` 调用。

通过在 `d_read` 的入口和出口添加 `device_printf` 来验证。你应该看到每次用户空间 `read(2)` 调用有一个入口和一个出口，不管 iovec 计数是多少。



## 对比模式：正确与有问题的处理程序

上面的故障排除指南是被动的：它帮助处理已经出错的情况。本节是规范性的配套。每个条目展示编写处理程序某部分的一个看似合理但错误的方式，将其与正确的重写配对，并解释区别。提前学习对比是避免 bug 的最快方式。

仔细阅读每对。正确的版本是你应该采用的模式；有 bug 的版本是你在快速移动时自己的手可能产生的形状。在几个月后从野外识别出错误，值得今天花五分钟来内化差异。

### 对比 1：返回字节计数

**有 bug 的：**

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

**正确的：**

```c
static int
myfirst_read(struct cdev *dev, struct uio *uio, int ioflag)
{
        /* ... */
        return (uiomove_frombuf(sc->message, sc->message_len, uio));
}
```

**为什么重要。** 处理程序的返回值是 errno，不是计数。内核从 `uio->uio_resid` 的变化计算字节计数并报告给用户空间。非零正返回被解释为 errno；如果你返回 `sc->message_len`，调用者会收到一个非常奇怪的 `errno` 值。例如，返回 `75` 会表现为 `errno = 75`，在 FreeBSD 上恰好是 `EPROGMISMATCH`。这个 bug 既错误又对从用户侧查看它的任何人来说非常困惑。

规则简单且绝对：处理程序返回 errno 值，永远不返回计数。如果你想知道字节计数，从 uio 计算。

### 对比 2：处理零长度请求

**有 bug 的：**

```c
static int
myfirst_read(struct cdev *dev, struct uio *uio, int ioflag)
{
        if (uio->uio_resid == 0)
                return (EINVAL); /* BAD: zero-length is legal */
        /* ... */
}
```

**正确的：**

```c
static int
myfirst_read(struct cdev *dev, struct uio *uio, int ioflag)
{
        /* No special case. uiomove handles zero-resid cleanly. */
        return (uiomove_frombuf(sc->message, sc->message_len, uio));
}
```

**为什么重要。** `read(fd, buf, 0)` 调用是合法的 UNIX 操作。用 `EINVAL` 拒绝它的驱动会破坏使用零字节读取检查描述符状态的程序。如果 uio 没有东西要移动，`uiomove` 立即返回零；你的处理程序不需要特殊情况处理它。错误地特殊情况处理它比完全不特殊情况处理更糟糕。

### 对比 3：缓冲区容量计算

**有 bug 的：**

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

**正确的：**

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

**为什么重要。** 有 bug 的版本传递给 `uiomove` 的长度是 `uio_resid`，可能超过缓冲区的剩余容量。`uiomove` 不会移动超过 `uio_resid` 字节，但*目标*是 `sc->buf + sc->bufused`，数学计算不知道 `sc->buflen`。如果用户向 `bufused = 0` 的 4 KiB 缓冲区写入 8 KiB，处理程序将在 `sc->buf` 末尾之后写入 4 KiB。这是一个经典的内核堆溢出：崩溃不会立即发生，不会牵连你的驱动，可能在半秒后作为完全不相关子系统内的崩溃显现。

正确的版本将传输限制到 `avail`，保证指针算术留在缓冲区内。限制是一个 `MIN` 调用，而且它不是可选的。

### 对比 4：跨 uiomove 持有自旋锁

**有 bug 的：**

```c
mtx_lock_spin(&sc->spin);            /* BAD: spin lock, not a regular mutex */
error = uiomove(sc->buf + off, n, uio);
mtx_unlock_spin(&sc->spin);
return (error);
```

**正确的：**

```c
mtx_lock(&sc->mtx);                  /* MTX_DEF mutex */
error = uiomove(sc->buf + off, n, uio);
mtx_unlock(&sc->mtx);
return (error);
```

**为什么重要。** `uiomove(9)` 可能睡眠。当它调用 `copyin` 或 `copyout` 时，用户页面可能被换出，内核可能需要从磁盘换入，这需要等待 I/O。持有自旋锁（`MTX_SPIN`）时睡眠会死锁系统。如果启用了 `WITNESS`，FreeBSD 的 `WITNESS` 框架在第一次发生时会崩溃。在非 `WITNESS` 内核上，结果是静默的活锁。

规则很简单：自旋锁不能跨可能睡眠的函数持有，而 `uiomove` 可能睡眠。对 I/O 处理程序触及的 softc 状态使用 `MTX_DEF` 互斥锁（默认的，也是 `myfirst` 使用的）。

### 对比 5：在 d_open 中重置共享状态

**有 bug 的：**

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

**正确的：**

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

**为什么重要。** `d_open` 每个描述符运行一次。如果两个读取者打开设备，第二次打开将擦除第一次打开留下的所有内容。驱动范围的状态（`sc->bufused`、`sc->buf`、计数器）属于整个驱动，仅在 `attach` 和 `detach` 时重置。每次描述符的状态属于 `fh`，`malloc(M_ZERO)` 自动将其初始化为零。

在 `d_open` 中重置共享状态的驱动在单一打开者下看起来正常工作，当两个打开者出现时静默地损坏状态。这个 bug 在两个用户同时读取设备的那一天之前是不可见的。

### 对比 6：在知道结果之前记账

**有 bug 的：**

```c
sc->bytes_written += towrite;       /* BAD: count before success */
error = uiomove(sc->buf + tail, towrite, uio);
if (error == 0)
        sc->bufused += towrite;
```

**正确的：**

```c
error = uiomove(sc->buf + tail, towrite, uio);
if (error == 0) {
        sc->bufused += towrite;
        sc->bytes_written += towrite;
}
```

**为什么重要。** 如果 `uiomove` 中途失败，一些字节可能已移动而另一些没有。`sc->bytes_written` 计数器应该反映实际到达缓冲区的内容，而不是驱动尝试的内容。在知道结果之前更新计数器使计数器说谎。如果用户读取 sysctl 来诊断问题，他们看到的数字与现实不符。

规则：在 `if (error == 0)` 分支内更新计数器，这样成功是递增它们的唯一路径。这是小的代价换来大的正确性收益。

### 对比 7：直接解引用用户指针

**有 bug 的：**

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

**正确的：**

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

**为什么重要。** `memcpy` 假设两个指针都引用当前地址空间中可访问的内存。用户指针不是。取决于平台，在内核上下文中将用户指针传递给 `memcpy` 的结果从 `EFAULT` 等效故障（在启用了 SMAP 的 amd64 上）到静默数据损坏（在没有用户/内核分离的平台上）到彻底的内核崩溃。

`copyin` 和 `copyout` 是从内核上下文访问用户内存的唯一正确方式。它们安装故障处理程序、验证地址、安全地遍历页表，并在任何失败时返回 `EFAULT`。性能成本是几条额外的指令；正确性收益是"当有 bug 的用户程序运行时内核不会崩溃"。

### 对比 8：在 d_open 失败时泄漏每次打开的结构

**有 bug 的：**

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

**正确的：**

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

**为什么重要。** 当 `devfs_set_cdevpriv` 失败时，内核不会注册析构函数，所以析构函数永远不会在这个 `fh` 上运行。如果处理程序返回而不释放 `fh`，内存就泄漏了。在持续负载下，重复的 `d_open` 失败可以泄漏足够的内存使内核不稳定。

规则：在错误展开路径中，到目前为止所做的每个分配都必须被释放。第八章的读者已经在 attach 中见过这个模式；它同样适用于 `d_open`。

### 如何使用这个对比表

这八对不是详尽的列表。它们是我们早期学生驱动中最常见到的 bug，也是本章文本一直在试图帮你避免的 bug。现在通读它们一次。在你写本书之外的第一个真正驱动之前，再读一遍。

开发过程中的一个有用习惯：每当你完成一个处理程序，在脑海中对照对比表走一遍。处理程序返回计数了吗？它特殊情况处理零 resid 了吗？它有容量限制吗？互斥锁类型正确吗？它在 `d_open` 中重置共享状态了吗？它在失败时计入字节了吗？它直接解引用任何用户指针了吗？它在 `d_open` 失败时泄漏了吗？八个问题，五分钟。检查的代价很小；将这些 bug 中的一个发布到生产的成本很大。



## 第十章前的自我评估

第九章涵盖了很多内容。在你放下它之前，过一遍以下检查清单。如果任何项目让你犹豫，相关章节值得在继续之前重读。这不是测试；这是识别你的心理模型可能仍然薄弱的地方的快速方式。

**概念：**

- [ ] 我可以用一句话解释 `struct uio` 是用来做什么的。
- [ ] 我能说出 `struct uio` 中我的驱动最常读取的三个字段。
- [ ] 我可以解释为什么在 `d_read` 和 `d_write` 内部 `uiomove(9)` 优于 `copyin` / `copyout`。
- [ ] 我可以解释为什么跨用户/内核边界使用 `memcpy` 是不安全的。
- [ ] 我可以解释在驱动语境下 `ENXIO`、`EAGAIN`、`ENOSPC` 和 `EFAULT` 之间的区别。

**机制：**

- [ ] 我可以编写一个使用 `uiomove_frombuf(9)` 提供固定缓冲区的最小 `d_read` 处理程序。
- [ ] 我可以编写一个以正确的容量限制向内核缓冲区追加的最小 `d_write` 处理程序。
- [ ] 我知道在传输周围放置互斥锁获取和释放的位置。
- [ ] 我知道如何将 errno 从 `uiomove` 传播回用户空间。
- [ ] 我知道如何用 `uio_resid = 0` 标记写入为完全消耗。

**可观测性：**

- [ ] 我可以读取 `sysctl dev.myfirst.0.stats` 并解释每个计数器。
- [ ] 我可以用 `vmstat -m | grep devbuf` 发现内存泄漏。
- [ ] 我可以用 `truss(1)` 查看我的测试程序发出了什么系统调用。
- [ ] 我可以用 `fstat(1)` 或 `fuser(8)` 找到谁持有描述符。

**陷阱：**

- [ ] 我不会从 `d_read` / `d_write` 返回字节计数。
- [ ] 我不会用 `EINVAL` 拒绝零长度请求。
- [ ] 我不会在 `d_open` 内重置 `sc->bufused`。
- [ ] 我不会在 `uiomove` 调用期间持有自旋锁。

任何"否"都是一个信号，而不是判决。重读相关章节；在你的实验室运行一个小实验；回到检查清单。当每个复选框都打勾时，你就稳稳地为第十章做好了准备。



## 总结

你刚刚实现了使驱动活跃的入口点。 在第七章结束时你的驱动有了一个骨架。 在第八章结束时它有了一个形状良好的门。 现在，在第九章结束时，数据双向流过这扇门。

本章的核心教训比看起来更短。 你将编写的每个 `d_read` 都有相同的三行骨架：获取每次打开的状态、验证活跃性、调用 `uiomove`。 你将编写的每个 `d_write` 都有类似的骨架，加上一个额外的决定（我有多少空间？）和一个防止缓冲区溢出的钳制（`MIN(uio_resid, avail)`）。本章的其他一切都是上下文：为什么 `struct uio` 看起来是这个样子，为什么 `uiomove` 是唯一安全的移动器，为什么 errno 值重要，为什么计数器重要，为什么缓冲区必须在每个错误路径上被释放。

### 最重要的三个想法

**首先，`struct uio` 是你的驱动和内核 I/O 机制之间的契约。** 它携带你的处理程序需要知道的关于调用的所有信息：用户请求了什么、用户的内存在哪里、传输应该向哪个方向移动、已经取得了多少进展。你不需要记住所有七个字段。你需要识别 `uio_resid`（剩余工作）、`uio_offset`（位置，如果你关心的话）和 `uio_rw`（方向），你需要信任 `uiomove(9)` 处理其余的。

**其次，`uiomove(9)` 是用户内存和内核内存之间的边界。** 你的驱动在两者之间移动的所有东西都通过它（或通过其近亲：`uiomove_frombuf`、`copyin`、`copyout`）。这不是建议。跨信任边界的直接指针访问要么损坏内存要么泄露信息，内核没有廉价的方法在错误变成 CVE 之前捕获它。如果指针来自用户空间，通过内核的信任边界函数路由它。永远如此。

**第三，正确的处理程序通常是短的处理程序。** 如果你的 `d_read` 或 `d_write` 超过十五行，可能有问题。更长的处理程序要么复制了属于其他地方的逻辑（在缓冲区管理中、在每次打开状态设置中、在 sysctl 中），要么它们试图做一些驱动不应该在数据路径处理程序中做的事情（通常是属于 `d_ioctl` 的事情）。保持处理程序短。将它们调用的机制放入命名良好的辅助函数。你未来的自己会感谢你。

### 你结束本章时的驱动形状

你的阶段 3 `myfirst` 是一个小型、诚实、内存中的 FIFO。显著特征：

- 一个 4 KiB 内核缓冲区，在 `attach` 中分配，在 `detach` 中释放。
- 一个保护 `bufhead`、`bufused` 和相关计数器的每次实例互斥锁。
- 一个排空缓冲区并推进 `bufhead` 的 `d_read`，在缓冲区为空时折叠到零。
- 一个追加到缓冲区并在填满时返回 `ENOSPC` 的 `d_write`。
- 存储在 `struct myfirst_fh` 中的每次描述符计数器，在 `d_open` 中分配，在析构函数中释放。
- 暴露实时驱动状态的 sysctl 树。
- 干净的 `attach` 错误展开和干净的 `detach` 排序。

这个形状会在你在第四部分和第六部分读到的一半驱动中回来，是可识别的。它是一个通用模式，不是一次性的演示。

### 开始第十章前你应该练习什么

五个练习，大致按难度递增排列：

1. 从头开始重建所有三个阶段，不看伴随树。之后将你的结果与树比较；差异是你还需要内化的内容。
2. 在阶段 3 中引入一个故意 bug：当 `bufused` 达到零时忘记重置 `bufhead`。观察第二次大写入时发生什么。用代码的术语解释症状。
3. 添加一个暴露 `sc->buflen` 的 sysctl。使其只读。然后将其转换为可在加载时通过 `kenv` 或 `loader.conf` 设置并在 `attach` 中获取的可调参数。（第十章正式回顾可调参数；这是一个预览。）
4. 编写一个 shell 脚本，将已知长度的随机数据写入 `/dev/myfirst/0`，然后通过 `sha256` 读回。比较哈希值。当写入大小超过缓冲区时哈希值是否匹配？（不应该匹配；想想为什么。）
5. 在 `/usr/src/sys/dev` 下找一个同时实现 `d_read` 和 `d_write` 的驱动。阅读其处理程序。将它们映射到本章的模式。好的候选：`/usr/src/sys/dev/null/null.c`（你已经知道它）、`/usr/src/sys/dev/random/randomdev.c`、`/usr/src/sys/dev/speaker/spkr.c`。

### 展望第十章

第十章采用阶段 3 驱动并使其扩展。四个新能力出现：

- **环形缓冲区**取代线性缓冲区。写入和读取都可以持续发生，而不需要阶段 3 使用的显式折叠。
- **阻塞读取**到来。在空缓冲区上调用 `read(2)` 的读取者可以睡眠直到数据可用，而不是立即返回零字节。内核的 `msleep(9)` 是原语；`d_purge` 处理程序是拆卸安全网。
- **非阻塞 I/O**成为一等特性。`O_NONBLOCK` 用户在阻塞调用者会睡眠的地方得到 `EAGAIN`。
- **`poll(2)` 和 `kqueue(9)` 集成**。用户程序可以等待设备变得可读或可写，而不需要主动尝试操作。这是将设备集成到事件循环中的标准方式。

所有这四个都建立在你刚刚实现的相同 `d_read` / `d_write` 形状之上。你将扩展处理程序而不是重写它们，你已经就位的每次描述符状态将承载必要的簿记。前面的那一章（这一章）是 I/O 路径本身正确的章节。第十章是 I/O 路径变得高效的章节。

在关闭文件之前，最后一个安慰。本章的材料并不像第一次阅读时感觉的那么难。模式很小。想法是真实的，但它们是有限的，你刚刚针对工作代码练习了每一个。当你在源码树中阅读真正驱动的 `d_read` 或 `d_write` 时，你现在将能识别函数在做什么以及为什么。你不再是这个的初学者了。你是一个手握真正工具的学徒。



## 参考：本章使用的签名和辅助函数

本章依赖的声明、辅助函数和常量的综合参考。在编写驱动时将此页面加入书签；大多数初学者问题都可以在这些表格之一中找到答案。

### `d_read` 和 `d_write` 签名

来自 `/usr/src/sys/sys/conf.h`：

```c
typedef int d_read_t(struct cdev *dev, struct uio *uio, int ioflag);
typedef int d_write_t(struct cdev *dev, struct uio *uio, int ioflag);
```

返回值成功时为零，失败时为正 errno。字节计数从 `uio->uio_resid` 的变化计算，并作为 `read(2)` / `write(2)` 的返回值报告给用户空间。

### 规范的 `struct uio`

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

### `uio_seg` 和 `uio_rw` 枚举

来自 `/usr/src/sys/sys/_uio.h`：

```c
enum uio_rw  { UIO_READ, UIO_WRITE };
enum uio_seg { UIO_USERSPACE, UIO_SYSSPACE, UIO_NOCOPY };
```

### `uiomove` 系列

来自 `/usr/src/sys/sys/uio.h`：

```c
int uiomove(void *cp, int n, struct uio *uio);
int uiomove_frombuf(void *buf, int buflen, struct uio *uio);
int uiomove_fromphys(struct vm_page *ma[], vm_offset_t offset, int n,
                     struct uio *uio);
int uiomove_nofault(void *cp, int n, struct uio *uio);
int uiomove_object(struct vm_object *obj, off_t obj_size, struct uio *uio);
```

在初学者驱动代码中，只有 `uiomove` 和 `uiomove_frombuf` 是常见的。其他的支持特定的内核子系统（物理页 I/O、无页故障复制、VM 支持的对象），超出了本章的范围。

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

在控制路径（`d_ioctl`）中使用这些函数，当用户指针在 uio 抽象之外到达时。在 `d_read` 和 `d_write` 内部，优先使用 `uiomove`。

### 字符设备重要的 `ioflag` 位

来自 `/usr/src/sys/sys/vnode.h`：

```c
#define IO_NDELAY       0x0004  /* FNDELAY flag set in file table */
```

当描述符处于非阻塞模式时设置。你的 `d_read` 或 `d_write` 可以使用它来决定是阻塞（缺少标志）还是返回 `EAGAIN`（设置了标志）。大多数其他 `IO_*` 标志是文件系统级别的，与字符设备无关。

### 内存分配

来自 `/usr/src/sys/sys/malloc.h`：

```c
void *malloc(size_t size, struct malloc_type *type, int flags);
void  free(void *addr, struct malloc_type *type);
```

常见标志：`M_WAITOK`、`M_NOWAIT`、`M_ZERO`。驱动的常见类型：`M_DEVBUF`（通用）或通过 `MALLOC_DECLARE` / `MALLOC_DEFINE` 声明的驱动特定类型。

### 每次打开状态（第八章延续，在此使用）

来自 `/usr/src/sys/sys/conf.h`：

```c
int  devfs_set_cdevpriv(void *priv, d_priv_dtor_t *dtr);
int  devfs_get_cdevpriv(void **datap);
void devfs_clear_cdevpriv(void);
```

模式是：在 `d_open` 中分配，用 `devfs_set_cdevpriv` 注册，在每个后续处理程序中用 `devfs_get_cdevpriv` 检索，在 `devfs_set_cdevpriv` 注册的析构函数中清理。

### 本章使用的 Errno 值

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

### 有用的 `device_printf(9)` 模式

```c
device_printf(sc->dev, "open via %s fh=%p\n", devtoname(sc->cdev), fh);
device_printf(sc->dev, "write rejected: buffer full (used=%zu)\n",
    sc->bufused);
device_printf(sc->dev, "read delivered %zd bytes\n",
    (ssize_t)(before - uio->uio_offset));
```

这些是为可读性编写的。`dmesg` 中你需要解码的行可能是在重要时刻不会被阅读的行。

### 三个阶段一览

| Stage | `d_read`                                             | `d_write`                          |
|-------|------------------------------------------------------|------------------------------------|
| 1     | Serve fixed message via `uiomove_frombuf`            | Discard writes (like `/dev/null`)  |
| 2     | Serve buffer up to `bufused`                         | Append to buffer, `ENOSPC` if full |
| 3     | Drain buffer from `bufhead`, reset on empty          | Append at `bufhead + bufused`, `ENOSPC` if full |

阶段 3 是第十章构建的基础。

### 本章的综合文件列表

伴随文件位于 `examples/part-02/ch09-reading-and-writing/`：

- `stage1-static-message/`: Stage 1 driver source and Makefile.
- `stage2-readwrite/`: Stage 2 driver source and Makefile.
- `stage3-echo/`: Stage 3 driver source and Makefile.
- `userland/rw_myfirst.c`: small C program to exercise read and write round-trips.
- `userland/stress_rw.c`: multi-process stress test for Lab 9.3 and beyond.
- `README.md`: a short map of the companion tree.

每个阶段都是独立的；你可以构建、加载和练习它们中的任何一个而不需要构建其他的。Makefile 除了驱动名称（始终为 `myfirst`）和可选的调优标志外完全相同。



## 附录 A：深入查看 uiomove 的内部循环

对于想确切了解 `uiomove(9)` 做什么的读者，本附录演练 `uiomove_faultflag` 核心循环在 `/usr/src/sys/kern/subr_uio.c` 中的样子。你不需要阅读这个来编写驱动。它在这里是因为一次循环阅读将澄清你后续关于 uio 语义的每个问题。

### 设置

在入口处，函数有：

- A kernel pointer `cp` provided by the caller (your driver).
- An integer `n` provided by the caller (the max bytes to move).
- The uio provided by the kernel dispatch.
- A boolean `nofault` indicating whether page faults during the copy should be handled or fatal.

它对几个不变量进行健全性检查：方向是 `UIO_READ` 或 `UIO_WRITE`，当段是用户空间时拥有线程是当前线程，`uio_resid` 是非负的。任何违反都是一个 `KASSERT`，会使启用了 `INVARIANTS` 的内核崩溃。

### 主循环

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

每次迭代做一个工作单元：在当前 iovec 条目和内核缓冲区之间复制最多 `cnt` 字节（其中 `cnt` 是 `MIN(iov->iov_len, n)`）。方向由两个嵌套的 `switch` 语句选择。成功复制后，所有记账字段同步推进：iovec 条目缩小 `cnt`，uio 的 resid 缩小 `cnt`，uio 的偏移增长 `cnt`，内核指针 `cp` 推进 `cnt`，调用者的 `n` 缩小 `cnt`。

当 iovec 条目完全排空时（循环入口处 `cnt == 0`），函数推进到下一个条目。当调用者的 `n` 达到零或 uio 的 resid 达到零时，循环终止。

如果 `copyin` 或 `copyout` 返回非零，函数跳到 `out` 而不更新该迭代的字段，所以部分复制的记账是一致的：已复制的任何字节反映在 `uio_resid` 中，未复制的仍然待处理。

### 你应该带走什么

循环中产生三个对你的驱动代码重要的不变量。

- **你对 `uiomove(cp, n, uio)` 的调用最多移动 `MIN(n, uio->uio_resid)` 字节。** 没有办法请求超过 uio 有空间的内容；函数以较小的一方为上限。
- **在部分传输时，状态是一致的。** `uio_resid` 精确反映未移动的字节。你可以再进行一次调用，它会正确地继续。
- **故障处理在循环内部，而不是周围。** `copyin` / `copyout` 期间的故障为剩余部分返回 `EFAULT`；字段仍然一致。

这三个事实就是为什么我们一直回到的三行骨架（`uiomove`、检查错误、更新状态）是足够的。内核在循环内部做复杂的工作；你的驱动只需配合。



## 附录 B：为什么允许 read(fd, buf, 0)

一个关于频繁出现的问题的简短说明：为什么 UNIX 允许 `read(fd, buf, 0)` 或 `write(fd, buf, 0)` 调用？

有两个答案，两个都值得了解。

**实际答案**：零长度 I/O 是免费测试。想检查描述符是否处于合理状态的用户程序可以调用 `read(fd, NULL, 0)` 而不提交真正的传输。如果描述符坏了，调用返回错误。如果没问题，调用返回零，几乎不花费任何代价。

**语义答案**：UNIX I/O 接口一致地使用字节计数，特殊处理零比允许它更费事。`count == 0` 的调用是定义良好的无操作：内核不需要做任何事，可以立即返回零。替代方案——对零计数调用返回 `EINVAL`——将迫使每个动态计算计数的用户程序防范这种情况。那是那种为了没有好处而破坏几十年代码的改变。

驱动侧的后果，我们之前已经注意到：你的处理程序不能在零 `uio_resid` 上崩溃或报错。当你通过 `uiomove` 时，内核实际上为你处理了这种情况，如果没有东西要移动，它会立即返回零。

如果你曾经发现自己编写 `if (uio->uio_resid == 0) return (EINVAL);`，停下来。那是错误的答案。零计数 I/O 是有效的；返回零。



## 附录 C：/dev/zero 读取路径简短导览

作为结束分析，值得演练当用户程序在 `/dev/zero` 上调用 `read(2)` 时到底发生了什么。驱动是 `/usr/src/sys/dev/null/null.c`，处理程序是 `zero_read`。一旦你理解了这个路径，你就理解了第九章的一切。

### 从用户空间到内核分派

用户调用：

```c
ssize_t n = read(fd, buf, 1024);
```

C 库发起 `read` 系统调用。内核在调用进程的文件表中查找 `fd`，检索 `struct file`，识别其 vnode，将调用分派到 devfs。

devfs 识别与 vnode 关联的 cdev，获取其上的引用，并使用内核准备的 uio 调用其 `d_read` 函数指针（`zero_read`）。

### `zero_read` 内部

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

- 断言方向是正确的。好习惯；`KASSERT` 在生产内核中不花费任何代价。
- 将 `zbuf` 设置为指向 `zero_region`，一个大型预分配的零填充区域。
- 循环：当调用者需要更多字节时，确定传输大小（`uio_resid` 和零区域大小的最小值），调用 `uiomove`，累积任何错误。
- 返回。

### `uiomove` 内部

对于第一次迭代，`uiomove` 看到 `uio_resid = 1024`、`len = 1024`（因为 `ZERO_REGION_SIZE` 大得多）、`uio_segflg = UIO_USERSPACE`、`uio_rw = UIO_READ`。它选择 `copyout(zbuf, buf, 1024)`。内核执行复制，处理用户缓冲区上的任何页面故障。成功时，`uio_resid` 降到零，`uio_offset` 增长 1024，iovec 被完全消耗。

### 回溯调用栈

`uiomove` 返回零。`zero_read` 中的循环看到 `uio_resid == 0` 并退出。`zero_read` 返回零。

devfs 释放其在 cdev 上的引用。内核计算字节计数为 `1024 - 0 = 1024`。`read(2)` 向用户返回 1024。

用户的缓冲区现在持有 1024 个零字节。

### 这告诉你关于你自己驱动的什么

两个观察。

首先，`zero_read` 中的每个数据路径决策都是你现在也在做的决策。每次迭代移动多大的块；从哪个缓冲区读取；如何处理 `uiomove` 的错误。你的驱动的决策在细节上会有所不同（你的缓冲区不是预分配的零区域，你的块大小不是 `ZERO_REGION_SIZE`），但形状是相同的。

其次，`zero_read` 之上的所有东西是你不需要编写的内核机制。你实现处理程序，内核负责系统调用、文件描述符查找、VFS 分派、devfs 路由、引用计数和故障处理。这就是抽象的力量：你添加驱动的知识，其他一切都免费获得。

反面是，当你编写驱动时，你承诺与该机制*合作*。`uiomove` 和 devfs 依赖的每个不变量现在都是你要维护的责任。本章一直在一次一个地引导你了解这些不变量，通过构建三个小驱动，每个驱动锻炼不同的子集。

到目前为止，模式应该已经很熟悉了。



## 附录 D：用户侧常见的 read(2)/write(2) 返回值

用户程序与你的驱动通信时看到的内容的简短速查表。这不是驱动代码；这是信任边界另一侧的视图。偶尔阅读它是针对驱动做了行为良好的 UNIX 程序期望之外的事情时出现的微妙 bug 的最好预防。

### `read(2)`

- 正整数：那么多字节被放入调用者的缓冲区。少于请求计数意味着短读取；调用者循环。
- 零：文件结束。此描述符上不再会产生字节。调用者停止。
- `-1` 且 `errno = EAGAIN`：非阻塞模式，现在没有可用数据。调用者等待（通过 `select(2)` / `poll(2)` / `kqueue(2)`）并重试。
- `-1` 且 `errno = EINTR`：信号中断了读取。调用者通常重试，除非信号处理程序告诉它不要。
- `-1` 且 `errno = EFAULT`：缓冲区指针无效。调用者有 bug。
- `-1` 且 `errno = ENXIO`：设备已消失。调用者应该关闭描述符并放弃。
- `-1` 且 `errno = EIO`：设备报告了硬件错误。调用者可以重试或报告。

### `write(2)`

- 正整数：那么多字节被接受。少于提供的计数意味着短写入；调用者用剩余部分循环。
- 零：理论上可能，实践中很少见到。通常被视为零字节的短写入。
- `-1` 且 `errno = EAGAIN`：非阻塞模式，现在没有空间。调用者等待并重试。
- `-1` 且 `errno = ENOSPC`：永久没有空间。调用者要么停止写入，要么重新打开描述符。
- `-1` 且 `errno = EPIPE`：读取者关闭了。与管道类设备相关，与 `myfirst` 无关。
- `-1` 且 `errno = EFAULT`：缓冲区指针无效。
- `-1` 且 `errno = EINTR`：被信号中断。通常重试。

### 这对你的驱动意味着什么

两个要点。

首先，`EAGAIN` 是非阻塞调用者期望驱动说"现在没有数据/没有空间，稍后再来"的方式。看到 `EAGAIN` 的非阻塞调用者不将其视为错误；它等待唤醒（通常通过 `poll(2)`）并重试。第十章使这个机制为 `myfirst` 工作。

其次，`ENOSPC` 是驱动在写入时发出永久没有空间信号的方式。它与 `EAGAIN` 的区别在于调用者不期望重试很快成功。对于 `myfirst` 阶段 3，当缓冲区填满且没有读取者主动排空时我们使用 `ENOSPC`；第十章将在非阻塞读取者和写入者的相同条件之上叠加 `EAGAIN`。

在这里返回错误 errno 的驱动几乎与行为不当的驱动无法区分。正确的代价很小。错误的代价在几个月后以困惑的用户程序的形式出现。



## 附录 E：一页速查表

如果你在开始第十章之前只有五分钟，这是上面所有内容的一页版本。

**签名：**

```c
static int myfirst_read(struct cdev *dev, struct uio *uio, int ioflag);
static int myfirst_write(struct cdev *dev, struct uio *uio, int ioflag);
```

成功时返回零，失败时返回正 errno。永远不返回字节计数。

**读取的三行骨架：**

```c
error = devfs_get_cdevpriv((void **)&fh);
if (error) return error;
return uiomove_frombuf(sc->buf, sc->buflen, uio);
```

或者，对于动态缓冲区：

```c
mtx_lock(&sc->mtx);
toread = MIN((size_t)uio->uio_resid, sc->bufused);
error = uiomove(sc->buf + offset, toread, uio);
if (error == 0) { /* update state */ }
mtx_unlock(&sc->mtx);
return error;
```

**写入的三行骨架：**

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

**关于 uio 要记住的：**

- `uio_resid`: bytes still pending. `uiomove` decrements this.
- `uio_offset`: position, if meaningful. `uiomove` increments this.
- `uio_rw`: direction. Trust `uiomove` to use it.
- Everything else: do not touch.

**不要做什么：**

- Do not dereference user pointers directly.
- Do not use `memcpy` / `bcopy` between user and kernel.
- Do not return byte counts.
- Do not reset driver-wide state in `d_open` / `d_close`.
- Do not forget `M_ZERO` on `malloc(9)`.
- Do not hold a spin lock across `uiomove`.

**Errno 值：**

- `0`: success.
- `ENXIO`: device not ready.
- `ENOSPC`: buffer full (permanent).
- `EAGAIN`: would block (non-blocking).
- `EFAULT`: from `uiomove`, propagate.
- `EIO`: hardware error.

以上就是本章内容。



## 章节总结

本章构建了数据路径。 从第八章的桩函数开始，我们在三个阶段中实现了 `d_read` 和 `d_write`，每个阶段都是完整的可加载驱动。

- **Stage 1** 对静态内核字符串使用了 `uiomove_frombuf(9)`, 带有使两个并发读取者的进度独立的每次描述符偏移处理。
- **Stage 2** 引入了动态内核缓冲区、向其中追加的写入路径和从中提供服务的读取路径。 缓冲区在挂载时确定大小，满缓冲区以 `ENOSPC` 拒绝进一步写入。
- **Stage 3** 将缓冲区转变为先进先出队列。 读取从头部排空，写入向尾部追加，驱动在缓冲区清空时将 `bufhead` 折叠为零。

在此过程中我们逐字段剖析了 `struct uio`, 解释了为什么 `uiomove(9)` 是在读或写处理程序中跨越用户/内核信任边界的唯一合法方式, 并构建了一个良好行为的驱动使用的小型 errno 值词汇表: `ENXIO`, `EFAULT`, `ENOSPC`, `EAGAIN`, `EIO`. 我们走过了 `uiomove` 的内部循环，使其保证感觉是应得的而非神秘的。 最后我们有五个实验、六个挑战、一个故障排除指南和一份一页速查表。

阶段 3 驱动是通往第十章的路径点。 它正确地移动字节。 它尚未高效地移动字节: 空缓冲区立即返回零字节，满缓冲区立即返回 `ENOSPC`，没有阻塞，没有 `poll(2)` 集成，没有环形缓冲区。 第十章在我们刚刚绘制的形状基础上修复所有这些问题。

你刚学到的模式会重复。 `/usr/src/sys/dev` 中的每个字符设备 I/O 处理程序都构建在相同的三参数签名、相同的 `struct uio` 和相同的 `uiomove(9)` 原语之上。 驱动之间的差异在于它们如何准备数据，而不在于它们如何移动数据。 一旦你识别了移动机制，你打开的每个处理程序几乎立即变得可读。

你现在有足够的知识阅读 FreeBSD 源码树中的任何 `d_read` 或 `d_write` 并理解它在做什么。 那是一个重要的里程碑。在翻页之前花一分钟欣赏它。
