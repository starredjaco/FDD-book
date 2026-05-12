---
title: "高效处理输入与输出"
description: "将线性内核缓冲区转变为真正的循环队列：部分 I/O、非阻塞读写、mmap，以及为安全并发奠定基础。"
partNumber: 2
partName: "构建你的第一个驱动程序"
chapter: 10
lastUpdated: "2026-04-18"
status: "complete"
author: "Edson Brandi"
reviewer: "TBD"
translator: "AI辅助翻译为简体中文"
language: "zh-CN"
estimatedReadTime: 210
---

# 高效处理输入与输出

## 读者指南与学习目标

第九章以一个小而诚实的驱动结束。你的 `myfirst` 模块作为 Newbus 设备挂载，在 `/dev/myfirst/0` 创建带有 `/dev/myfirst` 别名的设备，分配每次打开的状态，通过 `uiomove(9)` 传输字节，并维护一个简单的先进先出内核缓冲区。你可以向其中写入数据、读回相同的数据、看到 `sysctl dev.myfirst.0.stats` 中的字节计数器上升，并观察每次描述符关闭时每次打开的析构函数运行。那是一个完整的、可加载的、基于源码的驱动，你在第九章构建的三个阶段是第十章即将构建的基础。

然而，阶段 3 的 FIFO 仍然有一些令人不满意的地方。它正确地移动字节，但没有很好地利用缓冲区。一旦 `bufhead` 向前移动，它留下的空间就会浪费，直到缓冲区排空且 `bufhead` 折回到零。一个稳定的生产者和一个匹配的消费者可以在任何一方真正耗尽工作之前很久就耗尽容量。满缓冲区立即返回 `ENOSPC`，即使读取者距离排空一半只有一毫秒。空缓冲区返回零字节作为伪文件结束，即使调用者准备好等待。这些行为对于教学检查点来说都没有错，但没有一个能扩展。

本章是我们使 I/O 路径高效的地方。

真正的驱动在其他工作的后台移动字节。到达当前数据末尾的读取者可能想要阻塞直到更多数据到达。发现缓冲区满的写入者可能想要阻塞直到读取者腾出了空间。非阻塞调用者想要清晰的 `EAGAIN` 而不是礼貌的虚构。像 `cat` 或 `dd` 这样的工具想要以匹配自己缓冲区大小的块读写，而不是驱动内部约束的大小。内核希望驱动在不丢失字节、不损坏共享状态、不积累初学者驱动经常积累的索引和边界情况混乱的情况下完成所有这些。

真正的驱动实现这一点的方式是通过少数纪律严明的模式。**循环缓冲区**取代线性阶段 3 缓冲区并使整个容量保持可用。**部分 I/O** 让 `read(2)` 和 `write(2)` 返回请求中实际可用的部分，而不是全有或全无。**非阻塞模式**让编写良好的调用者询问"是否有数据了？"而不必提交睡眠。仔细的**重构**将缓冲区从 softc 内部的一组临时字段转换为一个小型命名抽象，第十一章可以用真正的同步原语来保护它。

所有这些都牢牢地位于字符设备领域。我们仍在编写伪设备，仍然通过 `struct uio` 读写，仍然用 `kldload(8)` 和 `kldunload(8)` 加载和卸载模块。变化的是内核缓冲区和用户程序之间数据平面的*形状*，以及驱动做出的承诺的质量。

### 为什么本章值得单独设立

这些材料可以走捷径。许多教程用十行展示循环缓冲区，在某处撒上 `mtx_sleep`，然后宣布胜利。那种方法产生的代码通过一个测试，然后在负载下产生神秘的 bug。错误通常不在读写处理程序本身。它们在于循环缓冲区如何环绕，当活跃字节跨越缓冲区物理末尾时如何调用 `uiomove(9)`，如何解释 `IO_NDELAY`，`selrecord(9)` 和 `selwakeup(9)` 如何与从不调用 `poll(2)` 的非阻塞调用者组合，以及部分写入应该如何报告给本身在循环的用户空间程序。

本章逐一解决这些细节。结果是一个你可以用 `dd` 压力测试、用生产者-消费者对轰炸、通过 `sysctl` 检查、并交给第11章作为并发工作稳定基础的驱动。

### 第九章结束时驱动的状态

检查点你应该从中工作的状态。如果你的源码树和实验机器匹配此大纲，第10章的一切都会顺利落地。如果不匹配，返回并将阶段 3 调整为以下形状后再继续。

- `nexus0` 下的一个 Newbus 子设备，由 `device_identify` 创建。
- 为 FIFO 缓冲区加统计信息大小设计的 `struct myfirst_softc`。
- 以设备命名的互斥锁 `sc->mtx`，保护 softc 计数器和缓冲区索引。
- `dev.myfirst.0.stats` 处的 sysctl 树，暴露 `attach_ticks`、`open_count`、`active_fhs`、`bytes_read`、`bytes_written` 以及当前的 `bufhead`、`bufused` 和 `buflen`。
- `/dev/myfirst/0` 处的主 cdev，所有权为 `root:operator`，模式为 `0660`。
- `/dev/myfirst` 处指向主设备的别名 cdev。
- 通过 `devfs_set_cdevpriv(9)` 的每次打开状态和 `myfirst_fh_dtor` 析构函数。
- `MYFIRST_BUFSIZE` 字节的线性 FIFO 缓冲区，`bufhead` 在空时折回到零。
- `d_read` 在 `bufused == 0` 时返回零字节（这是我们在这一阶段对 EOF 的近似）。
- `d_write` 在尾部到达 `buflen` 时返回 `ENOSPC`。

第10章接受该驱动并用真正的循环缓冲区替换线性 FIFO。然后它扩展 `d_read` 和 `d_write` 处理程序以正确支持部分 I/O，添加带有 `EAGAIN` 的 `O_NONBLOCK` 感知路径，连接 `d_poll` 处理程序使 `select(2)` 和 `poll(2)` 开始工作，最后将缓冲区逻辑拉入一个准备好加锁的命名抽象。

### 你将学到什么

完成本章后，你将能够：

- 用简单的语言解释缓冲给驱动带来了什么以及它在哪里开始有害。
- 在内核空间中设计和实现固定大小的面向字节的循环缓冲区。
- 正确地推理环绕：检测它、跨环绕分割传输，并在这样做时保持 `uio` 记账诚实。
- 将该循环缓冲区集成到不断演进的 `myfirst` 驱动中，而不退化任何早期行为。
- 以经典 UNIX 程序期望的方式处理部分读取和部分写入。
- 在你的读写处理程序中解释和遵守 `IO_NDELAY`。
- 实现 `d_poll` 使 `select(2)` 和 `poll(2)` 能对 `myfirst` 工作。
- 使用 `dd(1)`、`cat(1)`、`hexdump(1)` 和一个小型生产者-消费者对从用户空间压力测试缓冲 I/O。
- 识别当前驱动包含的读-修改-写危险，并做出重构，让第11章在不重构代码的情况下引入真正的锁。
- 阅读 `d_mmap(9)` 并理解字符设备驱动何时想让用户空间直接映射缓冲区，何时确实不应该。
- 以区分真正节省和口号的方式讨论零拷贝。
- 在驱动中识别预读和写合并模式，并描述为什么它们对吞吐量重要。

### 你将构建什么

你将从第9章的阶段 3 驱动通过四个主要阶段，加上一个添加内存映射支持的短小可选第五阶段。

1. **阶段 1，独立循环缓冲区。** 在触及内核之前，你将在用户态构建 `cbuf.c` 和 `cbuf.h`，编写少量小测试，确认环绕、空和满的行为符合你的心智模型。这是你完全可以在用户态开发的章节的唯一部分，当驱动开始以本来可以被三行单元测试捕获的方式失败时，它会物有所值。
2. **阶段 2，循环缓冲区驱动。** 你将把循环缓冲区拼接进 `myfirst`，使 `d_read` 和 `d_write` 现在驱动新的抽象。`bufhead` 变为 `cb_head`，`bufused` 变为 `cb_used`，字段位于小型 `struct cbuf` 内，环绕算术在一处可见。用户空间可见的行为尚未改变，但驱动在稳定负载下立即表现更好。
3. **阶段 3，部分 I/O 和非阻塞支持。** 你将扩展处理程序以正确支持部分读写，将 `IO_NDELAY` 解释为 `EAGAIN`，并引入使用 `mtx_sleep(9)` 和 `wakeup(9)` 的阻塞读取路径。驱动现在以低延迟奖励礼貌的调用者，以阻塞语义奖励耐心的调用者。
4. **阶段 4，poll 感知和重构就绪。** 你将添加 `d_poll` 处理程序，连接 `struct selinfo`，并通过一组紧密的辅助函数重构所有缓冲区访问，以便第11章可以插入真正的锁策略。
5. **阶段 5，内存映射（可选）。** 你将添加一个小型 `d_mmap` 处理程序，以便用户空间可以通过 `mmap(2)` 读取缓冲区。这个阶段与第8节的补充主题和章节末尾的匹配实验一起探索。你可以在首次阅读时跳过它而不丢失线索；它从不同角度重新审视相同的缓冲区。

你将使用基本系统工具、你在阶段 1 编译的小型 `cb_test` 用户态程序以及两个新辅助工具来练习每个阶段：`rw_myfirst_nb`（非阻塞测试器）和 `producer_consumer`（基于 fork 的负载工具）。每个阶段位于 `examples/part-02/ch10-handling-io-efficiently/` 下专用目录中，那里的 README 反映了章节的检查点。

### 本章不涵盖的内容

值得明确指出我们在这里*不会*尝试做什么。这些主题的更深入讨论属于后面的章节，现在拖入它们会模糊本章的课程。

- **真正的并发正确性。** 第10章使用 softc 中已经存在的互斥锁，并使用 `mtx_sleep(9)` 以该互斥锁作为睡眠互锁参数。这在构造上是安全的。但本章不探索竞态条件的完整空间，也不对锁类别进行分类，也不教读者如何证明一块共享状态被正确保护。那是第11章的工作，也是本章最后一节被称为"重构并为并发做准备"而不是"驱动中的并发"的原因。
- **`ioctl(2)`。** 驱动仍然没有实现 `d_ioctl`。一些清理原语（刷新缓冲区、查询其填充级别）很适合放在 `ioctl` 下，但第25章才是合适的地方。
- **`kqueue(2)`。** 本章为 `select(2)` 和 `poll(2)` 实现 `d_poll`。伴随的 `d_kqfilter` 处理程序，连同 `knlist` 机制和 `EVFILT_READ` 过滤器，稍后与 `taskqueue` 驱动的驱动一起介绍。
- **硬件 mmap。** 我们将构建一个最小化的 `d_mmap` 处理程序，让用户空间将预分配的内核缓冲区映射为只读页面，我们将讨论设计决策以及这种模式能做和不能做的事情。我们不会涉足 `bus_space(9)`、`bus_dmamap_create(9)` 或 `dev_pager` 机制；那些是第4部分和第5部分的内容。
- **具有真正反压的真实设备。** 这里的反压模型是"缓冲区有固定容量，满时阻塞或返回 `EAGAIN`"。存储和网络驱动有更丰富的模型（水位线、信用、BIO 队列、mbuf 链）。这些细节属于它们自己的章节。

将章节保持在这些界限内是我们保持其诚实的方式。你将学到的材料足以编写一个体面的伪设备，并能够自信地阅读树中大多数字符设备驱动。

### 预计时间投入

- **仅阅读**：大约九十分钟，如果在图表处暂停可能两小时。
- **阅读加上输入四个阶段**：四到六小时，分为至少两次会话，中间重启一两次。
- **阅读加上所有实验和挑战**：八到十二小时，分三次会话。挑战确实比主要实验更丰富，它们奖励耐心的工作。

与第9章一样，在开始时给自己一个全新的实验启动。不要急于完成四个阶段。序列的价值在于观察驱动的行为变化，一次一个模式，当你添加每个能力时。

### 前提条件

在开始本章之前，确认：

- 你的驱动源码匹配 `examples/part-02/ch09-reading-and-writing/stage3-echo/` 下第9章阶段 3 的示例。如果不匹配，在这里停下并先将其调整到该形状。第10章假设它已经如此。
- 你的实验机器运行带有匹配 `/usr/src` 的 FreeBSD 14.3。 The APIs and file layouts you will see in this chapter align with that release.
- 你仔细阅读了第9章，包括附录 E（一页速查表）。那里的读写"三行骨架"正是我们即将扩展的内容。
- 你能够熟练加载和卸载自己的模块、观察 `dmesg`、读取 `sysctl dev.myfirst.0.stats`，以及在测试让你惊讶时读取 `truss(1)` 或 `ktrace(1)` 的输出。

如果这些中有任何不稳固的，现在修复它们比继续本章更能利用时间。

### 如何从本章获得最大收益

三个习惯保持有用。

首先，在第二个终端中保持打开 `/usr/src/sys/dev/evdev/cdev.c`。它是树中最干净的字符设备示例之一，实现了环形缓冲区、在缓冲区空时阻塞调用者、遵守 `O_NONBLOCK`，并通过 `wakeup(9)` 和 `selinfo` 机制唤醒睡眠者。我们将多次指向它。

其次，保持标记 `/usr/src/sys/kern/subr_uio.c`。第9章演练了这个文件的 `uiomove` 内部；当缓冲区环绕迫使我们分割传输时，我们将重新访问它。阅读真实代码强化正确的心智模型。

第三，偶尔在 `truss(1)` 下运行测试，而不仅仅在普通 shell 下。追踪系统调用返回值是区分正在遵守部分 I/O 的驱动和静默丢弃字节的驱动的最快方式。

### 本章路线图

各节按顺序排列：

1. 什么是缓冲 I/O 以及为何它重要。桶和管道、非缓冲与缓冲模式，以及各自在驱动中的适用场景。
2. 创建循环缓冲区。数据结构、不变量、环绕算术，以及一个你将在放入内核之前测试的独立用户态实现。
3. 将该循环缓冲区集成到 `myfirst` 中。`d_read` 和 `d_write` 如何变化、如何处理环绕周围的分割传输，以及如何记录缓冲区状态使调试不需要猜测。
4. 部分读取和部分写入。它们是什么、为什么它们是正确的 UNIX 行为、如何通过 `uio_resid` 报告它们，以及你必须避免的边界情况。
5. 非阻塞 I/O。`IO_NDELAY` 标志、它与 `O_NONBLOCK` 的关系、`EAGAIN` 约定，以及使用 `mtx_sleep(9)` 和 `wakeup(9)` 的简单阻塞读取路径的设计。
6. 从用户空间测试缓冲 I/O。综合测试套件：`dd`、`cat`、`hexdump`、`truss`、小型非阻塞测试器和在 `/dev/myfirst` 上 fork 读取者和写入者的生产者-消费者工具。
7. 重构并为并发做准备。当前代码哪里有风险、如何将缓冲区重构为辅助函数、以及你想交给第11章的形状。
8. 三个补充主题：`d_mmap(9)` 作为最小化的内核内存映射（可选的阶段 5）、伪设备的零拷贝考虑，以及真正高吞吐量驱动使用的模式（读取侧的预读、写入侧的写合并）。
9. 动手实验，一组你应该能够直接对驱动完成的具体练习。
10. 挑战练习，在不引入全新基础的情况下延伸相同技能。
11. 本章模式倾向于产生的 bug 类别的故障排除说明。
12. 总结以及通往第11章的桥梁。

如果你是第一次阅读，请线性阅读并按顺序做实验。如果你复习材料以巩固，每个编号的补充主题和故障排除部分都可以独立阅读。

## 第1节：什么是缓冲 I/O 以及为何它重要

每个在用户空间和硬件侧或内核侧数据源之间移动字节的驱动都必须决定这些字节在中间*驻留*在哪里。第9章的阶段 3 驱动已经隐式做出了这个决定。已写入但尚未读取的 `bufused` 字节位于内核缓冲区中。读取者从头部拉取它们；写入者在尾部追加新的。从这个意义上说，`myfirst` 驱动已经是缓冲的。

本章变化的不是你是否有缓冲区。而是缓冲区的*形状*如何、你一次能保持多少容量*可用*、以及驱动对调用者关于缓冲区行为做出*什么承诺*。在查看数据结构之前，值得在概念层面暂停思考非缓冲和缓冲 I/O 之间的差异。这种对比将指导本章其余部分要求你做出的每个决定。

### 简明的定义

在最简单的框架中，**非缓冲 I/O** 意味着每次 `read(2)` 或 `write(2)` 调用都直接触及底层源或汇。没有吸收突发流量的中间存储，没有生产者可以留下字节供消费者稍后提取的地方，没有解耦字节产生速率和消耗速率的方法。每次调用都一路到底。

**缓冲 I/O** 相反，在生产者和消费者之间放置一小块内存区域。写入者将字节放入缓冲区；读取者提取它们。只要缓冲区有空闲空间，写入者就不需要等待。只要缓冲区有活跃字节，读取者就不需要等待。缓冲区吸收两方之间的短期不匹配。

这听起来像是一个小区别，但在驱动代码中，它通常是负载下能工作与不能工作之间的区别。

值得在一个小但重要的细节上暂停。内核本身在驱动上下方多个层次进行缓冲。C 库的 `stdio` 在写入到达 `write(2)` 系统调用之前缓冲它们。VFS 路径在缓冲区缓存中缓冲常规文件的 I/O。第7部分的磁盘驱动将在 BIO 和队列级别缓冲。当本章说"缓冲 I/O"时，它指的是驱动*内部*的缓冲区，在面向用户的读写处理程序和驱动代表的任何数据源或汇之间。我们不是在争论缓冲是否存在；我们是在决定在哪里再放一个缓冲区，以及它应该做什么。

### 两个具体图景

首先想象一个非缓冲伪设备。想象一个 `d_write` 立即将每个字节交给上游消耗代码的驱动。如果消费者忙碌，写入者等待。如果消费者快速，写入者飞速通过。系统中没有弹性。任何一方的突发直接转化为另一方的压力。

现在想象一个缓冲伪设备。相同的 `d_write` 将字节存入一个小型缓冲区。消费者以自己的节奏提取字节。短暂的写入突发可以瞬间完成，因为缓冲区吸收了它们。消费者短暂的暂停不会使写入者停滞，因为缓冲区保持了积压。双方都感觉自己运行顺畅，即使它们的速率在每一刻都不完全匹配。

缓冲情况是大多数有用驱动在实践中看起来的样子。它不是魔法；缓冲区是有限的，一旦填满，生产者必须等待或退避。但它给了系统一个容忍正常变化的地方，这种容忍是使吞吐量可预测的原因。

### 桶与管道

一个有用的类比是用桶运水和通过管道运水之间的区别。

当你用桶运水时，每次传输都是离散事件。你走到井边，装满桶，走回来，倒空桶，再走。生产者（井）和消费者（水池）通过你的双臂和步行速度紧密耦合。如果你绊倒，系统停止。如果水池忙碌，你在那里等待。如果井忙碌，你在那里等待。每次交接要求双方同时准备好。

管道用一段管子取代了这种耦合。水从井端进入，从水池端流出。管道在任何时刻都保持一定量的水在运输中。只要管道有空间，生产者就可以泵水。只要管道中有水，消费者就可以排水。它们的时间表不再需要匹配。它们只需要*平均*匹配。

驱动缓冲区正是那个管道。它是一个有限的储库，解耦写入者速率和读取者速率，只要两个速率平均到缓冲区容量可以吸收的某个值。桶模型对应非缓冲 I/O。管道模型对应缓冲 I/O。两者在不同情况下都有效，驱动编写者的工作是知道构建哪一个。

### 性能：系统调用和上下文切换

驱动内部缓冲的性能优势是真实但间接的。它们来自三个方面。

第一个方面是**系统调用开销**。每次 `read(2)` 或 `write(2)` 都是从用户空间到内核空间再返回的转换。该转换在现代处理器上很便宜，但不是免费的。用一千字节调用一次 `write(2)` 的写入者支付一次转换。用一字节调用一千次 `write(2)` 的写入者支付一千次转换。如果驱动内部缓冲，调用者可以舒适地发出更大的读写，每次系统调用的开销成为总成本的更小部分。

第二个方面是**上下文切换减少**。因为现在没有可用内容而不得不等待的调用，通常导致调用线程被挂起并调度另一个线程。每次挂起和恢复都比系统调用更昂贵。缓冲区吸收否则会迫使睡眠的短暂不匹配，双方的线程继续运行。

第三个方面是**批处理机会**。知道有数千字节准备发送的驱动有时可以在一次操作中将整批数据交到下游，而逐字节处理的驱动必须为每次传输做相同的设置和拆卸工作。我们在本章的伪设备中不会直接看到这一点，但它是我们稍后在章节中看到的读取合并和写入合并模式的底层论据。

这些优势都不应该盲目应用。缓冲也增加了延迟，因为一个字节可能在缓冲区中停留一段时间后才被消费者注意到。它增加了内存成本。它引入了一对必须在并发访问下保持一致的索引。它迫使做出一组设计决策：缓冲区满时做什么（阻塞？丢弃？覆盖？）以及缓冲区空时做什么（阻塞？短读？发信号表示文件结束？）。本章在这些决策上花费大量时间是有原因的。

### 设备驱动中的缓冲数据传输

缓冲到底在设备驱动的哪里产生回报？

最清晰的情况是数据源产生突发的驱动。串口驱动在 UART 芯片发出中断时接收字符；如果消费者当前没有读取，这些字符需要一个地方存放直到它读取。键盘驱动在中断处理程序中收集按键事件，并以应用程序愿意读取的任何速率将它们交给用户空间。网络驱动在 DMA 缓冲区中组装数据包并尽快将其馈送到协议栈。在每种情况下，驱动都需要一个地方在数据到达时刻和可以被交付时刻之间保存传入数据。

镜像情况是数据汇吸收突发的驱动。图形驱动可以排队命令直到 GPU 准备好处理它们。打印机驱动可以接受文档并以打印机速度慢慢输出。存储驱动可以收集写入请求并让电梯算法为磁盘重新排序。同样，驱动需要一个地方在用户写入时刻和设备准备好时刻之间保存传出数据。

我们在本书中构建的伪设备位于这两种模式的中间。两端都没有真正的硬件，但数据路径的*形状*反映了真正驱动的做法。当你写入 `/dev/myfirst` 时，字节落在驱动拥有的缓冲区中。当你读取时，字节从同一缓冲区出来。一旦缓冲区是循环的且 I/O 处理程序知道如何支持部分传输，你可以从一个终端用 `dd if=/dev/zero of=/dev/myfirst bs=1m count=10` 压力测试驱动，从另一个终端用 `dd if=/dev/myfirst of=/dev/null bs=4k`，驱动将像真正的字符设备在类似负载下的行为一样。

### 何时在驱动中使用缓冲 I/O

几乎每个驱动都需要某种形式的缓冲。有趣的问题不是是否缓冲，而是以什么*粒度*和什么*反压模型*。

粒度是关于缓冲区大小相对于两侧速率的关系。太小的缓冲区不断填满并迫使写入者等待，违背了目的。太大的缓冲区隐藏问题太久，让内存无限制增长，并增加最坏情况延迟。正确的大小取决于缓冲区的用途：交互式键盘驱动的缓冲区只需要保存少量最近事件；网络驱动的缓冲区可能在峰值负载时保存数千个数据包。

反压是关于缓冲区填满（或排空）时且调用模式与驱动期望不匹配时做什么。有三种常见策略，每种适用于不同场景。

第一种是**阻塞**。当缓冲区满时，写入者等待。当缓冲区空时，读取者等待。这是经典的 UNIX 语义，是终端设备、管道和大多数通用伪设备的正确默认选择。我们将在第5节实现阻塞读取（和可选的阻塞写入）。

第二种是**丢弃**。当缓冲区满时，写入者丢弃字节（或标记溢出事件）并继续。当缓冲区空时，读取者看到零字节并继续。这是某些实时和高速率场景的正确默认选择，等待比丢失数据造成更多损害。但丢失必须是可观察的，否则驱动会从用户的角度静默损坏流。

第三种是**覆盖**。当缓冲区满时，写入者用新数据覆盖最旧的数据。当缓冲区空时，读取者看到零字节。这是最近事件循环日志的正确默认选择：类似 `dmesg(8)` 的历史，最近字节总是以最旧字节为代价保留。

本章中的驱动对阻塞模式调用者使用**阻塞**，对非阻塞模式调用者使用 **EAGAIN**，没有覆盖路径。这是 FreeBSD 源码树中最常见的模式，也是最容易推理的。另外两种策略在后面的章节中当它们的用例自然出现时引入。

### 非缓冲驱动代价的初览

值得具体说明为什么第9章的阶段 3 驱动在规模化时已经开始有害。

假设你正在运行一个高速率向驱动写入 64 字节块的 `dd`，以及一个并行从中读取 64 字节块的 `dd`。使用 4096 字节的阶段 3 缓冲区，在写入者遇到 `ENOSPC` 并停止之前，你最多有 64 个在途块。如果读取者因任何原因暂停（目标缓冲区页面故障、被调度器抢占、被移动到不同 CPU），写入者立即停止。一旦读取者恢复并排空单个块，写入者可以再多放入一个。总吞吐量是两半以锁步方式实现的最小值，加上用户空间程序不应该看到的持续 `ENOSPC` 错误流。

相同大小的循环缓冲区容纳相同数量的在途块，但从不浪费尾随容量。遇到满缓冲区的非阻塞写入者干净地收到 `EAGAIN`（传统的"稍后重试"信号）而不是 `ENOSPC`（传统的"此设备已无空间"），所以像 `dd` 这样的工具可以决定是重试还是退避。遇到满缓冲区的阻塞写入者在一个清晰的条件变量上睡眠，并在读取者释放空间的瞬间被唤醒。这些变化每一个都很小。它们一起使驱动感觉响应灵敏而不是脆弱。

### 我们要去哪里

你现在有了概念基础。本章的其余部分将把它转化为代码。第2节讲解数据结构，带有图表和你在信任内核之前可以测试的用户态实现。第3节将实现移入驱动，替换线性 FIFO。第4节和第5节扩展 I/O 路径，使部分传输和非阻塞语义以用户期望的方式工作。第6节构建你将在第2部分其余和第3部分大部分中使用的测试工具。第7节为定义第11章的锁工作准备代码。

顺序很重要。每一节都假设前一节的更改已到位。如果你跳过，你会进入不编译或以令人惊讶的方式行为的代码。像往常一样，慢路径就是快路径。

### Wrapping Up Section 1

我们命名了非缓冲和缓冲 I/O 之间的区别，并命名了各自的成本和收益。我们选择了一个可以不断回顾的类比（桶与管道）。我们讨论了缓冲在驱动代码中的回报、常见的反压策略，以及我们为本章其余部分承诺的策略。我们为驱动整个数据结构搭好了舞台：循环缓冲区。

如果你还不确定你的驱动应该使用哪种反压策略，那没关系。我们将构建的默认值——"在内核中阻塞，在外部返回 `EAGAIN`"——是通用伪设备的安全和传统选择，如果你以后需要不同的策略，你将有一个干净的代码形状可以重新审视。我们即将使那个缓冲区成为现实。

## 第2节：创建循环缓冲区

循环缓冲区是那种比我们现在使用的操作系统更古老的数据结构之一。它出现在串口芯片、音频采样队列、网络接收路径、键盘事件队列、追踪缓冲区、`dmesg(8)`、`printf(3)` 库以及几乎所有一段代码想要留下字节供另一段代码稍后提取的地方。结构简单。实现简短。初学者在其中犯的 bug 是可预测的。我们将用一次、在用户态、小心地构建它，然后在第3节中将验证过的版本带入驱动。

### What a Circular Buffer Is

线性缓冲区是可能工作的最简单的东西：一块内存加上一个"下一个空闲字节"索引。你从头开始写入，到达末尾时停止。一旦填满，你要么增长它、复制它，要么停止接受新数据。

循环缓冲区（也称为环形缓冲区）是相同的内存区域，但索引行为有所不同。有两个索引：*头部*，指向下一个要读取的字节，和*尾部*，指向下一个要写入的字节。当任一索引到达底层内存末尾时，它回绕到开头。缓冲区被视为其第一个字节与最后一个字节相邻，形成闭环。

两个派生计数对正确使用结构很重要。*活跃*字节数（当前存储了多少）是读取者关心的。*空闲*字节数（有多少容量未使用）是写入者关心的。两个计数都可以从头部和尾部加总容量通过一小段算术推导出来。

视觉上，当缓冲区部分填满且活跃区域没有环绕时，结构如下所示：

```text
  +---+---+---+---+---+---+---+---+
  | _ | _ | A | B | C | D | _ | _ |
  +---+---+---+---+---+---+---+---+
            ^               ^
           head           tail

  capacity = 8, used = 4, free = 4
```

足够多的写入后，尾部追上底层内存末尾并回绕到开头。现在活跃区域本身环绕了：

```text
  +---+---+---+---+---+---+---+---+
  | F | G | _ | _ | _ | _ | D | E |
  +---+---+---+---+---+---+---+---+
        ^               ^
       tail           head

  capacity = 8, used = 4, free = 4
  live region: head -> end of buffer, then start of buffer -> tail
```

"活跃区域环绕"情况是初学者容易出错的地方。简单的 `bcopy` 将缓冲区视为线性；你复制的字节不是你想要的字节。正确处理这种情况的方法是以*两块*执行传输：从 `head` 到缓冲区末尾，然后从缓冲区开头到 `tail`。我们将在下面的辅助函数中精确编写这个模式。

### 管理读写指针

头部和尾部指针（我们将称它们为索引，因为它们是固定大小数组中的整数偏移量）遵循简单规则。

当你读取 `n` 字节时，头部前进 `n`，对容量取模。当你写入 `n` 字节时，尾部前进 `n`，对容量取模。读取移除字节；活跃计数下降 `n`。写入添加字节；活跃计数增加 `n`。

有趣的问题是如何检测两个边界条件：空和满。仅有 `head` 和 `tail`，结构本身*几乎*足够。如果 `head == tail`，缓冲区可能是空的（没有存储字节）或满的（所有容量已存储）。仅从索引来看，两种状态看起来相同。实现以三种方式之一解决歧义。

第一种方式是保持一个单独的活跃字节**计数**。有了 `used`，`used == 0` 明确为空，`used == capacity` 明确为满。结构稍大但代码简短明显。这是我们在本章中使用的设计。

第二种方式是**总是留一个字节不使用**。使用这个规则，`head == tail` 总是表示空，缓冲区满的条件是 `(tail + 1) % capacity == head`。结构少一个字节，代码不需要 `used` 字段，但每次传输都涉及容易出错的差一。这是某些经典嵌入式代码中使用的设计；它没问题，但在我们的场景中没有真正的优势。

第三种方式是使用**单调递增的索引**，从不减少对容量取模，并将活跃计数计算为 `tail - head`。环绕则是你如何索引数组的函数（`tail % capacity`），而不是如何推进指针。这是 FreeBSD 内核中 `buf_ring(9)` 使用的设计，它使用 32 位计数器并信任环绕行为。它很优雅，但使原子操作和调试复杂化。我们不会使用它；显式的 `used` 字段是教学驱动的正确权衡。

### 检测缓冲区满和缓冲区空

有了显式的 `used` 计数，边界检查变得简单：

- **空**：`cb->cb_used == 0`。没有可读字节。
- **满**：`cb->cb_used == cb->cb_size`。没有可写空间。

头部和尾部索引的算术也很直接：

- 读取 `n` 字节后：`cb->cb_head = (cb->cb_head + n) % cb->cb_size; cb->cb_used -= n;`
- 写入 `n` 字节后：`cb->cb_used += n;`（尾部在需要时计算为 `(cb->cb_head + cb->cb_used) % cb->cb_size`）

我们将保持尾部隐式，从 `head` 和 `used` 推导。一些实现显式跟踪尾部。只要你一致地坚持，任何选择都可以。`tail` 隐式时，我们从不需要在单次操作中更新两个索引，这消除了整整一类 bug。

两个辅助派生量将反复出现：

- `cb_free(cb) = cb->cb_size - cb->cb_used`：缓冲区满之前还能写入多少字节。
- `cb_tail(cb) = (cb->cb_head + cb->cb_used) % cb->cb_size`：下一次写入应该放在哪里。

两者都是头部、已用计数和容量的纯函数。它们没有副作用，可以随时安全调用。

### 分配固定大小的循环缓冲区

缓冲区需要三个状态部分和一个后备内存块。以下是我们将使用的结构：

```c
struct cbuf {
        char    *cb_data;       /* backing storage, cb_size bytes */
        size_t   cb_size;       /* total capacity, in bytes */
        size_t   cb_head;       /* index of next byte to read */
        size_t   cb_used;       /* count of live bytes */
};
```

三个生命周期函数覆盖基础功能：

```c
int   cbuf_init(struct cbuf *cb, size_t size);
void  cbuf_destroy(struct cbuf *cb);
void  cbuf_reset(struct cbuf *cb);
```

`cbuf_init` 分配后备存储，初始化索引，成功时返回零，失败时返回正的 errno。`cbuf_destroy` 释放后备存储并将结构清零。`cbuf_reset` 清空缓冲区但不释放内存；两个索引回到零。

三个访问器函数给其余代码提供所需的边界信息：

```c
size_t cbuf_used(const struct cbuf *cb);
size_t cbuf_free(const struct cbuf *cb);
size_t cbuf_size(const struct cbuf *cb);
```

这些是微小的内联友好函数。它们不锁定任何东西；期望调用者持有更大系统所需的任何同步。（在本章的阶段 4，驱动的互斥锁将提供该同步。）

两个有趣的函数是字节移动原语：

```c
size_t cbuf_write(struct cbuf *cb, const void *src, size_t n);
size_t cbuf_read(struct cbuf *cb, void *dst, size_t n);
```

`cbuf_write` 从 `src` 复制最多 `n` 字节到缓冲区并返回实际复制的数量。`cbuf_read` 从缓冲区复制最多 `n` 字节到 `dst` 并返回实际复制的数量。两个函数内部处理环绕情况。调用者提供连续的源或目标；缓冲区在活跃区域或空闲区域跨越底层存储末尾时负责分割传输。

那个签名值得注意一下。注意函数返回 `size_t`，不是 `int`。它们报告进度，不是错误。返回少于请求的字节数*不是*错误条件；它是表达缓冲区满（对于写入）或空（对于读取）的正确方式。这反映了 `read(2)` 和 `write(2)` 本身的工作方式：小于请求的正返回值是"部分传输"，不是失败。我们将在第4节依赖这一点，当我们使驱动正确支持部分 I/O 时。

### 环绕详解

环绕逻辑很短，但值得仔细跟踪一个例子。假设缓冲区容量为 8，头部为 6，已用为 4。活跃字节存储在位置 6、7、0、1。

```text
  +---+---+---+---+---+---+---+---+
  | C | D | _ | _ | _ | _ | A | B |
  +---+---+---+---+---+---+---+---+
        ^               ^
       tail           head
  capacity = 8, used = 4, head = 6, tail = (6+4)%8 = 2
```

现在调用者通过 `cbuf_read` 请求 3 字节。函数执行以下操作：

1. 计算 `n = MIN(3, used) = MIN(3, 4) = 3`。调用者最多获得 3 字节。
2. 计算 `first = MIN(n, capacity - head) = MIN(3, 8 - 6) = 2`。这是从头部开始的连续块。
3. 从 `cb_data + 6` 复制 `first = 2` 字节到 `dst`。那是 A 和 B。
4. 计算 `second = n - first = 1`。这是必须从缓冲区开头来的传输部分。
5. 从 `cb_data + 0` 复制 `second = 1` 字节到 `dst + 2`。那是 C。
6. 推进 `cb_head = (6 + 3) % 8 = 1`。将 `cb_used` 减 3，留下 1。

调用者的目标现在持有 A、B、C。缓冲区状态将 D 作为唯一的活跃字节，`head = 1` 和 `used = 1`。下次读取将从位置 1 返回 D。

相同的逻辑适用于 `cbuf_write`，`tail` 取代 `head` 的角色。函数计算 `tail = (head + used) % capacity`，然后 `first = MIN(n, capacity - tail)`，从 `src` 复制 `first` 字节到 `cb_data + tail`，然后从 `src + first` 复制剩余部分到 `cb_data + 0`，并将 `cb_used` 推进总写入量。

两个函数中恰好有一个步骤会环绕。要么目标环绕（写入中），要么源环绕（读取中），但绝不会两者同时。这是使实现可管理的关键属性：缓冲区的环绕是内部数据的属性，不是调用者数据的属性，所以调用者的源和目标总是被视为普通的连续内存。

### Avoiding Overwrites and Data Loss

初学者常犯的一个错误是在缓冲区填满时让 `cbuf_write` 覆盖旧数据，理论是"更新的数据更重要"。这有时是正确的策略，正如我们在第1节中指出的，但它必须是*刻意的*设计选择，且对调用者可见，而不是状态的静默变更。传统的默认值是 `cbuf_write` 返回它实际写入的字节数，调用者应该查看返回值。

`cbuf_read` 也是一样：当缓冲区为空时，`cbuf_read` 返回零。调用者应该将零解释为"现在没有可用字节"，而不是错误。将该信号与驱动的 `EAGAIN` 或阻塞睡眠耦合是 I/O 处理程序的工作，不是缓冲区本身的工作。

如果你想要一个具有覆盖语义的循环缓冲区（例如 `dmesg` 风格的日志），最干净的方法是写一个单独的 `cbuf_overwrite` 函数并保持 `cbuf_write` 严格。两个不同的名字意味着两个不同的意图，未来的代码读者不必猜测哪种行为生效。

### Implementing It in Userland

学习这个结构的正确方式是在用户态输入一次并通过一些小测试运行它，然后才让内核信任它。相同的源码可以几乎不变地移入内核模块，除了分配和释放调用。

下面是用户态源码。它位于 `examples/part-02/ch10-handling-io-efficiently/cbuf-userland/`。

`cbuf.h`:

```c
/* cbuf.h: a fixed-size byte-oriented circular buffer. */
#ifndef CBUF_H
#define CBUF_H

#include <stddef.h>

struct cbuf {
        char    *cb_data;
        size_t   cb_size;
        size_t   cb_head;
        size_t   cb_used;
};

int     cbuf_init(struct cbuf *cb, size_t size);
void    cbuf_destroy(struct cbuf *cb);
void    cbuf_reset(struct cbuf *cb);

size_t  cbuf_size(const struct cbuf *cb);
size_t  cbuf_used(const struct cbuf *cb);
size_t  cbuf_free(const struct cbuf *cb);

size_t  cbuf_write(struct cbuf *cb, const void *src, size_t n);
size_t  cbuf_read(struct cbuf *cb, void *dst, size_t n);

#endif /* CBUF_H */
```

`cbuf.c`:

```c
/* cbuf.c: userland implementation of the byte-oriented ring buffer. */
#include "cbuf.h"

#include <errno.h>
#include <stdlib.h>
#include <string.h>

#ifndef MIN
#define MIN(a, b) (((a) < (b)) ? (a) : (b))
#endif

int
cbuf_init(struct cbuf *cb, size_t size)
{
        if (cb == NULL || size == 0)
                return (EINVAL);
        cb->cb_data = malloc(size);
        if (cb->cb_data == NULL)
                return (ENOMEM);
        cb->cb_size = size;
        cb->cb_head = 0;
        cb->cb_used = 0;
        return (0);
}

void
cbuf_destroy(struct cbuf *cb)
{
        if (cb == NULL)
                return;
        free(cb->cb_data);
        cb->cb_data = NULL;
        cb->cb_size = 0;
        cb->cb_head = 0;
        cb->cb_used = 0;
}

void
cbuf_reset(struct cbuf *cb)
{
        if (cb == NULL)
                return;
        cb->cb_head = 0;
        cb->cb_used = 0;
}

size_t
cbuf_size(const struct cbuf *cb)
{
        return (cb->cb_size);
}

size_t
cbuf_used(const struct cbuf *cb)
{
        return (cb->cb_used);
}

size_t
cbuf_free(const struct cbuf *cb)
{
        return (cb->cb_size - cb->cb_used);
}

size_t
cbuf_write(struct cbuf *cb, const void *src, size_t n)
{
        size_t avail, tail, first, second;

        avail = cbuf_free(cb);
        if (n > avail)
                n = avail;
        if (n == 0)
                return (0);

        tail = (cb->cb_head + cb->cb_used) % cb->cb_size;
        first = MIN(n, cb->cb_size - tail);
        memcpy(cb->cb_data + tail, src, first);
        second = n - first;
        if (second > 0)
                memcpy(cb->cb_data, (const char *)src + first, second);

        cb->cb_used += n;
        return (n);
}

size_t
cbuf_read(struct cbuf *cb, void *dst, size_t n)
{
        size_t first, second;

        if (n > cb->cb_used)
                n = cb->cb_used;
        if (n == 0)
                return (0);

        first = MIN(n, cb->cb_size - cb->cb_head);
        memcpy(dst, cb->cb_data + cb->cb_head, first);
        second = n - first;
        if (second > 0)
                memcpy((char *)dst + first, cb->cb_data, second);

        cb->cb_head = (cb->cb_head + n) % cb->cb_size;
        cb->cb_used -= n;
        return (n);
}
```

这段代码中有两件事值得注意。

首先，`cbuf_write` 和 `cbuf_read` 在进行任何复制*之前*将 `n` 限制为可用空间或活跃数据。这是部分传输语义的关键：函数乐意做比要求更少的工作，并确切告诉调用者做了多少。没有"缓冲区满"的错误路径，因为那不是错误。

其次，第二次 `memcpy` 周围的 `second > 0` 保护不是严格必要的（`memcpy(dst, src, 0)` 是良好定义的，什么也不做），但它使环绕推理一目了然。未来的读者可以看出第二次复制是有条件的，环绕情况已被处理。

### 一个小型测试程序

配套的 `cb_test.c` 用一组小但有意义的用例练习该结构。它足够短可以完整阅读：

```c
/* cb_test.c: simple sanity tests for the cbuf userland implementation. */
#include "cbuf.h"

#include <stdio.h>
#include <stdlib.h>
#include <string.h>

#define CHECK(cond, msg) \
        do { if (!(cond)) { fprintf(stderr, "FAIL: %s\n", msg); exit(1); } } while (0)

static void
test_basic(void)
{
        struct cbuf cb;
        char in[8] = "ABCDEFGH";
        char out[8] = {0};
        size_t n;

        CHECK(cbuf_init(&cb, 8) == 0, "init");
        CHECK(cbuf_used(&cb) == 0, "init used");
        CHECK(cbuf_free(&cb) == 8, "init free");

        n = cbuf_write(&cb, in, 4);
        CHECK(n == 4, "write 4");
        CHECK(cbuf_used(&cb) == 4, "used after write 4");

        n = cbuf_read(&cb, out, 2);
        CHECK(n == 2, "read 2");
        CHECK(memcmp(out, "AB", 2) == 0, "AB content");
        CHECK(cbuf_used(&cb) == 2, "used after read 2");

        cbuf_destroy(&cb);
        printf("test_basic OK\n");
}

static void
test_wrap(void)
{
        struct cbuf cb;
        char in[8] = "ABCDEFGH";
        char out[8] = {0};
        size_t n;

        CHECK(cbuf_init(&cb, 8) == 0, "init");

        /* Push head forward by writing and reading 6 bytes. */
        n = cbuf_write(&cb, in, 6);
        CHECK(n == 6, "write 6");
        n = cbuf_read(&cb, out, 6);
        CHECK(n == 6, "read 6");

        /* Now write 6 more, which should wrap. */
        n = cbuf_write(&cb, in, 6);
        CHECK(n == 6, "write 6 after wrap");
        CHECK(cbuf_used(&cb) == 6, "used after wrap write");

        /* Read all of it back; should return ABCDEF. */
        memset(out, 0, sizeof(out));
        n = cbuf_read(&cb, out, 6);
        CHECK(n == 6, "read 6 after wrap");
        CHECK(memcmp(out, "ABCDEF", 6) == 0, "content after wrap");
        CHECK(cbuf_used(&cb) == 0, "empty after drain");

        cbuf_destroy(&cb);
        printf("test_wrap OK\n");
}

static void
test_partial(void)
{
        struct cbuf cb;
        char in[8] = "12345678";
        char out[8] = {0};
        size_t n;

        CHECK(cbuf_init(&cb, 4) == 0, "init small");

        n = cbuf_write(&cb, in, 8);
        CHECK(n == 4, "write clamps to free space");
        CHECK(cbuf_used(&cb) == 4, "buffer full");

        n = cbuf_read(&cb, out, 8);
        CHECK(n == 4, "read clamps to live data");
        CHECK(memcmp(out, "1234", 4) == 0, "content of partial");
        CHECK(cbuf_used(&cb) == 0, "buffer empty after partial drain");

        cbuf_destroy(&cb);
        printf("test_partial OK\n");
}

int
main(void)
{
        test_basic();
        test_wrap();
        test_partial();
        printf("all tests OK\n");
        return (0);
}
```

编译并运行：

```sh
$ cc -Wall -Wextra -o cb_test cbuf.c cb_test.c
$ ./cb_test
test_basic OK
test_wrap OK
test_partial OK
all tests OK
```

三个测试覆盖了重要的情况：基本写/读循环、环绕缓冲区末尾的写入、以及触及容量边界的部分传输。它们不是穷尽的，但足以捕获最常见的实现错误。章节末尾的挑战练习要求你扩展它们。

### 为什么首先在用户态构建

当你知道驱动才是重要的时，在用户态写缓冲区可能感觉像是绕道。三个原因使这个绕道值得。

首先，内核是一个调试的恶劣环境。内核缓冲区中的 bug 可能锁定机器、导致内核崩溃，或静默损坏不相关的状态。相同代码在用户态中的 bug 只是一个打印友好消息的失败测试。

其次，这个缓冲区的内核侧和用户态侧实现几乎相同。唯一的区别是分配原语（`malloc(9)` 配合 `M_DEVBUF` 和 `M_WAITOK | M_ZERO` 对比 libc `malloc(3)`）和释放原语（`free(9)` 对比 libc `free(3)`）。一旦用户态版本正确，内核版本几乎是复制粘贴加小调整。

第三，独立构建一次缓冲区迫使你在平静的条件下思考它的 API。当你准备好将它拼接进驱动时，你已经知道 `cbuf_write` 返回什么、`cbuf_read` 返回什么、`cbuf_used` 意味着什么、环绕应该如何工作。这些都不需要在一个内核会话中途重新学习。

### 仍可能出错的事情

即使有上面的辅助函数，有几个错误值得现在标记，这样你就不会在第3节中犯它们。

第一个是**忘记将请求限制为可用空间**。如果 `cbuf_write` 在 `free = 30` 的缓冲区上以 `n = 100` 被调用，函数返回 30，不是 100。调用者必须检查返回值并相应行动。驱动的 `d_write` 将通过将 `uio_resid` 保持在未消耗量来将其转换为部分写入。我们将在第4节中非常明确地讨论这一点。

第二个是**忘记 `cbuf_used` 和 `cbuf_free` 可以在两次检查之间改变**。在单线程用户态测试中这是不可能的。在内核中，如果没有持有锁，不同的线程可以在任意两次函数调用之间修改缓冲区。第3节在所有缓冲区访问周围持有 softc 互斥锁；第7节解释原因。

第三个是**混淆索引**。一些实现显式跟踪尾部并隐式跟踪计数。另一些则相反。两者都可以。在单个缓冲区上混合两者不行。选择一个并坚持下去。我们选择"头部和已用"；尾部总是推导出来的。

第四个是**索引本身的整数回绕**。使用 `size_t` 和几千字节的缓冲区，索引永远不会超过 `cb_size`，`(cb_head + n) % cb_size` 总是良好定义的。如果你将此代码扩展到大于 `SIZE_MAX / 2` 的缓冲区，这就不再成立；你需要 64 位索引和显式模算术。对于 4 KB 或 64 KB 缓冲区的伪设备，基本结构绰绰有余。

### 第2节总结

你现在有了一个干净的、经过测试的、面向字节的循环缓冲区。它将请求限制为可用空间，报告实际传输大小，并在环绕唯一有意义的地方处理环绕：缓冲区内部。用户态测试给你一个小证据，证明实现行为与图表所说的一致。

第3节将这段代码带入内核。形状几乎相同；分配和同步改变。到下一节结束时，你的驱动的 `d_read` 和 `d_write` 将调用 `cbuf_read` 和 `cbuf_write` 而不是做自己的算术，以前内联在 `myfirst.c` 中的逻辑将有一个名字。

## 第3节：将循环缓冲区集成到驱动中

`cbuf` 的用户态实现与你即将放入内核的代码相同。几乎。有三个小改动：分配器、释放器，以及内核要求而用户态不要求的偏执级别。拼接后，驱动的读写处理程序大幅缩减，环绕算术从 `myfirst.c` 消失到它所属的辅助函数中。

本节仔细讲解拼接。我们将从缓冲区的内核侧变体开始，然后转向 `myfirst.c` 内部的集成更改，最后看看如何添加一些 sysctl 旋钮，使驱动的内部状态在调试时可见。

### 将 cbuf 移入内核

内核侧头文件 `cbuf.h` 与用户态的相同：

```c
#ifndef CBUF_H
#define CBUF_H

#include <sys/types.h>

struct cbuf {
        char    *cb_data;
        size_t   cb_size;
        size_t   cb_head;
        size_t   cb_used;
};

int     cbuf_init(struct cbuf *cb, size_t size);
void    cbuf_destroy(struct cbuf *cb);
void    cbuf_reset(struct cbuf *cb);

size_t  cbuf_size(const struct cbuf *cb);
size_t  cbuf_used(const struct cbuf *cb);
size_t  cbuf_free(const struct cbuf *cb);

size_t  cbuf_write(struct cbuf *cb, const void *src, size_t n);
size_t  cbuf_read(struct cbuf *cb, void *dst, size_t n);

#endif /* CBUF_H */
```

内核侧 `cbuf.c` 几乎是用户态文件的副本，有两个替换。`malloc(3)` 变为 `M_DEVBUF` 的 `malloc(9)`，带 `M_WAITOK | M_ZERO` 标志。`free(3)` 变为 `M_DEVBUF` 的 `free(9)`。`memcpy(3)` 调用在内核上下文中保持有效：内核有自己的 `memcpy` 和 `bcopy` 符号。以下是完整的内核版本：

```c
#include <sys/param.h>
#include <sys/kernel.h>
#include <sys/systm.h>
#include <sys/malloc.h>

#include "cbuf.h"

MALLOC_DEFINE(M_CBUF, "cbuf", "Chapter 10 circular buffer");

int
cbuf_init(struct cbuf *cb, size_t size)
{
        if (cb == NULL || size == 0)
                return (EINVAL);
        cb->cb_data = malloc(size, M_CBUF, M_WAITOK | M_ZERO);
        cb->cb_size = size;
        cb->cb_head = 0;
        cb->cb_used = 0;
        return (0);
}

void
cbuf_destroy(struct cbuf *cb)
{
        if (cb == NULL || cb->cb_data == NULL)
                return;
        free(cb->cb_data, M_CBUF);
        cb->cb_data = NULL;
        cb->cb_size = 0;
        cb->cb_head = 0;
        cb->cb_used = 0;
}

void
cbuf_reset(struct cbuf *cb)
{
        if (cb == NULL)
                return;
        cb->cb_head = 0;
        cb->cb_used = 0;
}

size_t
cbuf_size(const struct cbuf *cb)
{
        return (cb->cb_size);
}

size_t
cbuf_used(const struct cbuf *cb)
{
        return (cb->cb_used);
}

size_t
cbuf_free(const struct cbuf *cb)
{
        return (cb->cb_size - cb->cb_used);
}

size_t
cbuf_write(struct cbuf *cb, const void *src, size_t n)
{
        size_t avail, tail, first, second;

        avail = cbuf_free(cb);
        if (n > avail)
                n = avail;
        if (n == 0)
                return (0);

        tail = (cb->cb_head + cb->cb_used) % cb->cb_size;
        first = MIN(n, cb->cb_size - tail);
        memcpy(cb->cb_data + tail, src, first);
        second = n - first;
        if (second > 0)
                memcpy(cb->cb_data, (const char *)src + first, second);

        cb->cb_used += n;
        return (n);
}

size_t
cbuf_read(struct cbuf *cb, void *dst, size_t n)
{
        size_t first, second;

        if (n > cb->cb_used)
                n = cb->cb_used;
        if (n == 0)
                return (0);

        first = MIN(n, cb->cb_size - cb->cb_head);
        memcpy(dst, cb->cb_data + cb->cb_head, first);
        second = n - first;
        if (second > 0)
                memcpy((char *)dst + first, cb->cb_data, second);

        cb->cb_head = (cb->cb_head + n) % cb->cb_size;
        cb->cb_used -= n;
        return (n);
}
```

三点值得简要评论。

第一是 `MALLOC_DEFINE(M_CBUF, "cbuf", ...)`。这为缓冲区的分配声明了一个私有内存标签，这样 `vmstat -m` 可以单独显示 cbuf 代码使用了多少内存，与驱动的其余部分分开。我们在 `cbuf.c` 中声明它一次，与模块的其余部分有内部链接。驱动的 softc 仍然使用 `M_DEVBUF`。两个标签可以共存；它们是簿记标签，不是池。

第二是 `M_WAITOK` 标志。因为我们从不从中断上下文调用 `cbuf_init`（我们从 `myfirst_attach` 调用它，它在模块加载期间以普通内核线程上下文运行），如果系统暂时内存不足，等待内存是安全的。使用 `M_WAITOK`，`malloc(9)` 不会返回 `NULL`；如果分配无法继续，它会睡眠直到可以。因此我们不需要测试结果是否为 `NULL`。如果我们想要从禁止睡眠的上下文调用 `cbuf_init`，我们需要切换到 `M_NOWAIT` 并处理可能的 `NULL`。对于第10章的目的，`M_WAITOK` 是正确的选择。

第三是**内核 `cbuf` 不加锁**。它是一个纯数据结构。锁策略是*调用者*的责任。在 `myfirst.c` 内部，我们将在每次调用 `cbuf` 时持有 `sc->mtx`。这保持了抽象的小型化，并给第11章一个干净的重构目标。

### myfirst.c 中的变化

在你的编辑器中调出第9章的阶段 3 文件。集成涉及以下更改：

1. 将四个缓冲区相关的 softc 字段（`buf`、`buflen`、`bufhead`、`bufused`）替换为单个 `struct cbuf cb` 成员。
2. 从 `myfirst.c` 中移除 `MYFIRST_BUFSIZE` 宏（我们保留它但放在单个头文件中以避免重复）。
3. 在 `myfirst_attach` 中用 `cbuf_init` 初始化缓冲区。
4. 在 `myfirst_detach` 和 attach 失败路径中用 `cbuf_destroy` 拆除它。
5. 重写 `myfirst_read` 以对栈驻留弹跳缓冲区调用 `cbuf_read`，然后 `uiomove` 弹跳缓冲区出去。
6. 重写 `myfirst_write` 以将 `uiomove` 到栈驻留弹跳缓冲区，然后 `cbuf_write` 到环形中。

最后两个更改在查看代码之前值得简短讨论。为什么用弹跳缓冲区？为什么不直接对 cbuf 存储调用 `uiomove`？

答案是 `uiomove` 不理解环绕。它期望连续的目标（读取）或连续的源（写入）。如果循环缓冲区的活跃区域环绕了，调用 `uiomove(cb->cb_data + cb->cb_head, n, uio)` 会复制超出底层内存末尾到下一个分配的内容中。那是一个等待发生的堆损坏 bug。存在两种安全的形状；你可以选择任一种。

第一种安全形状是调用 `uiomove` *两次*，环绕的每一侧一次。驱动计算 `cb->cb_data + cb->cb_head` 处可用的连续块，为该块调用 `uiomove`，然后为 `cb->cb_data + 0` 处的环绕部分再次调用 `uiomove`。这很高效因为没有额外复制。它也更复杂且更难正确实现；驱动必须在两次 `uiomove` 调用之间做 `uio_resid` 的部分记账，中间的任何取消（信号、页面故障）使缓冲区处于部分排空状态。

第二种安全形状是使用内核侧的**弹跳缓冲区**：栈上的一个小型临时缓冲区，仅在 I/O 调用期间存在。驱动用 `cbuf_read` 将字节从 cbuf 读入弹跳缓冲区，然后 `uiomove` 弹跳缓冲区到用户空间。在写入侧，它从用户空间 `uiomove` 到弹跳缓冲区，然后 `cbuf_write` 弹跳缓冲区到 cbuf 中。成本是每个块多一次内核内复制；好处是简单、错误处理的局部性，以及将所有环绕感知逻辑放在 cbuf 中它所属的地方的能力。

弹跳缓冲区方法是我们在本章中将使用的。这与 `evdev/cdev.c` 等驱动使用的方法相同（在每客户端环形和栈驻留 `event` 结构之间用 `bcopy`，然后 `uiomove` 结构到用户空间）。栈驻留弹跳很小（256 或 512 字节就足够了），循环根据用户传输大小需求运行多次，每次迭代在 `uiomove` 失败时独立可重启。性能成本对于除极高吞吐量硬件驱动之外的一切都可以忽略不计，即使在那里，权衡通常也值得可读性的提升。

### 第2阶段驱动程序：重构的处理程序

以下是拼接后驱动的相关部分。完整源码在 `examples/part-02/ch10-handling-io-efficiently/stage2-circular/myfirst.c`。我们将内联展示 I/O 处理程序，然后演练变化的内容。

```c
#define MYFIRST_BUFSIZE         4096
#define MYFIRST_BOUNCE          256

struct myfirst_softc {
        device_t                dev;
        int                     unit;

        struct mtx              mtx;

        uint64_t                attach_ticks;
        uint64_t                open_count;
        uint64_t                bytes_read;
        uint64_t                bytes_written;

        int                     active_fhs;
        int                     is_attached;

        struct cbuf             cb;

        struct cdev            *cdev;
        struct cdev            *cdev_alias;

        struct sysctl_ctx_list  sysctl_ctx;
        struct sysctl_oid      *sysctl_tree;
};

static int
myfirst_read(struct cdev *dev, struct uio *uio, int ioflag)
{
        struct myfirst_softc *sc = dev->si_drv1;
        struct myfirst_fh *fh;
        char bounce[MYFIRST_BOUNCE];
        size_t take, got;
        int error;

        error = devfs_get_cdevpriv((void **)&fh);
        if (error != 0)
                return (error);
        if (sc == NULL || !sc->is_attached)
                return (ENXIO);

        while (uio->uio_resid > 0) {
                mtx_lock(&sc->mtx);
                take = MIN((size_t)uio->uio_resid, sizeof(bounce));
                got = cbuf_read(&sc->cb, bounce, take);
                if (got == 0) {
                        mtx_unlock(&sc->mtx);
                        break;          /* empty: short read or EOF */
                }
                sc->bytes_read += got;
                fh->reads += got;
                mtx_unlock(&sc->mtx);

                error = uiomove(bounce, got, uio);
                if (error != 0)
                        return (error);
        }
        return (0);
}

static int
myfirst_write(struct cdev *dev, struct uio *uio, int ioflag)
{
        struct myfirst_softc *sc = dev->si_drv1;
        struct myfirst_fh *fh;
        char bounce[MYFIRST_BOUNCE];
        size_t want, put, room;
        int error;

        error = devfs_get_cdevpriv((void **)&fh);
        if (error != 0)
                return (error);
        if (sc == NULL || !sc->is_attached)
                return (ENXIO);

        while (uio->uio_resid > 0) {
                mtx_lock(&sc->mtx);
                room = cbuf_free(&sc->cb);
                mtx_unlock(&sc->mtx);
                if (room == 0)
                        break;          /* full: short write */

                want = MIN((size_t)uio->uio_resid, sizeof(bounce));
                want = MIN(want, room);
                error = uiomove(bounce, want, uio);
                if (error != 0)
                        return (error);

                mtx_lock(&sc->mtx);
                put = cbuf_write(&sc->cb, bounce, want);
                sc->bytes_written += put;
                fh->writes += put;
                mtx_unlock(&sc->mtx);

                /*
                 * cbuf_write may store less than 'want' if another
                 * writer slipped in between our snapshot of 'room'
                 * and our cbuf_write call and consumed some of the
                 * space we had sized ourselves against.  With a single
                 * writer that cannot happen and put == want always.
                 * We still handle it defensively: a serious driver
                 * would reserve space up front to avoid losing bytes,
                 * and Chapter 11 will revisit this with proper
                 * multi-writer synchronization.
                 */
                if (put < want) {
                        /*
                         * The 'want - put' bytes we copied into 'bounce'
                         * with uiomove have already left the caller's
                         * uio and cannot be pushed back.  Record the
                         * loss by breaking out of the loop; the kernel
                         * will report the bytes actually stored via
                         * uio_resid.  This path is only reachable under
                         * concurrent writers, which the design here
                         * does not yet handle.
                         */
                        break;
                }
        }
        return (0);
}
```

与第9章阶段 3 相比有几处变化。

第一个变化是**循环**。两个处理程序现在循环直到 `uio_resid` 达到零或缓冲区无法满足下一次迭代。每次迭代最多移动 `sizeof(bounce)` 字节，即栈弹跳的大小。对于小请求，循环运行一次。对于大请求，它根据需要运行多次。这就是使部分 I/O 干净工作的方式：处理程序在缓冲区到达边界时自然产生短读取或写入。

第二个变化是**所有缓冲区访问都由 `mtx_lock`/`mtx_unlock` 包围**。`cbuf` 数据结构不知道锁的存在；驱动提供它。我们在每个 `cbuf_*` 调用和每个字节计数器更新周围持有锁。我们*不*跨 `uiomove(9)` 持有锁。跨 `uiomove` 持有互斥锁是 FreeBSD 中的真正 bug：`uiomove` 可能因页面故障而睡眠，持有互斥锁睡眠是 sleep-with-mutex 崩溃。第9章的演练讨论了这一点；我们现在通过将 cbuf 访问（在锁下）与 uiomove（无锁）分离来实施该规则。

第三个变化是**读取处理程序在缓冲区为空时返回 0**，可能已经传输了一些字节之后。旧阶段 3 的行为在这一层是相同的。变化的是*下一*节使读取可以阻塞，再下一节为非阻塞调用者添加 `EAGAIN` 路径。这里的结构是两个扩展的基础。

第四个变化是**写入处理程序支持部分写入**。当 `cbuf_free(&sc->cb)` 返回零时，循环退出，处理程序返回 0，`uio_resid` 反映未消耗的字节。用户空间 `write(2)` 调用将看到短写入计数，这是传统 UNIX 表示"我接受了你这么多的字节；请稍后用剩余部分再调用我"的方式。第4节详细讨论为什么这很重要以及如何编写处理它的用户代码。

### 更新 attach 和 detach

生命周期变化很小但真实：

```c
static int
myfirst_attach(device_t dev)
{
        struct myfirst_softc *sc;
        struct make_dev_args args;
        int error;

        sc = device_get_softc(dev);
        sc->dev = dev;
        sc->unit = device_get_unit(dev);

        mtx_init(&sc->mtx, device_get_nameunit(dev), "myfirst", MTX_DEF);

        sc->attach_ticks = ticks;
        sc->is_attached = 1;
        sc->active_fhs = 0;
        sc->open_count = 0;
        sc->bytes_read = 0;
        sc->bytes_written = 0;

        error = cbuf_init(&sc->cb, MYFIRST_BUFSIZE);
        if (error != 0)
                goto fail_mtx;

        make_dev_args_init(&args);
        args.mda_devsw = &myfirst_cdevsw;
        args.mda_uid = UID_ROOT;
        args.mda_gid = GID_OPERATOR;
        args.mda_mode = 0660;
        args.mda_si_drv1 = sc;

        error = make_dev_s(&args, &sc->cdev, "myfirst/%d", sc->unit);
        if (error != 0)
                goto fail_cb;

        sc->cdev_alias = make_dev_alias(sc->cdev, "myfirst");
        if (sc->cdev_alias == NULL)
                device_printf(dev, "failed to create /dev/myfirst alias\n");

        sysctl_ctx_init(&sc->sysctl_ctx);
        sc->sysctl_tree = SYSCTL_ADD_NODE(&sc->sysctl_ctx,
            SYSCTL_CHILDREN(device_get_sysctl_tree(dev)),
            OID_AUTO, "stats", CTLFLAG_RD | CTLFLAG_MPSAFE, 0,
            "Driver statistics");

        SYSCTL_ADD_U64(&sc->sysctl_ctx, SYSCTL_CHILDREN(sc->sysctl_tree),
            OID_AUTO, "attach_ticks", CTLFLAG_RD,
            &sc->attach_ticks, 0, "Tick count when driver attached");
        SYSCTL_ADD_U64(&sc->sysctl_ctx, SYSCTL_CHILDREN(sc->sysctl_tree),
            OID_AUTO, "open_count", CTLFLAG_RD,
            &sc->open_count, 0, "Lifetime number of opens");
        SYSCTL_ADD_INT(&sc->sysctl_ctx, SYSCTL_CHILDREN(sc->sysctl_tree),
            OID_AUTO, "active_fhs", CTLFLAG_RD,
            &sc->active_fhs, 0, "Currently open descriptors");
        SYSCTL_ADD_U64(&sc->sysctl_ctx, SYSCTL_CHILDREN(sc->sysctl_tree),
            OID_AUTO, "bytes_read", CTLFLAG_RD,
            &sc->bytes_read, 0, "Total bytes drained from the FIFO");
        SYSCTL_ADD_U64(&sc->sysctl_ctx, SYSCTL_CHILDREN(sc->sysctl_tree),
            OID_AUTO, "bytes_written", CTLFLAG_RD,
            &sc->bytes_written, 0, "Total bytes appended to the FIFO");
        SYSCTL_ADD_PROC(&sc->sysctl_ctx, SYSCTL_CHILDREN(sc->sysctl_tree),
            OID_AUTO, "cb_used",
            CTLTYPE_UINT | CTLFLAG_RD | CTLFLAG_MPSAFE,
            sc, 0, myfirst_sysctl_cb_used, "IU",
            "Live bytes currently held in the circular buffer");
        SYSCTL_ADD_PROC(&sc->sysctl_ctx, SYSCTL_CHILDREN(sc->sysctl_tree),
            OID_AUTO, "cb_free",
            CTLTYPE_UINT | CTLFLAG_RD | CTLFLAG_MPSAFE,
            sc, 0, myfirst_sysctl_cb_free, "IU",
            "Free bytes available in the circular buffer");
        SYSCTL_ADD_UINT(&sc->sysctl_ctx, SYSCTL_CHILDREN(sc->sysctl_tree),
            OID_AUTO, "cb_size", CTLFLAG_RD,
            (unsigned int *)&sc->cb.cb_size, 0,
            "Capacity of the circular buffer");

        device_printf(dev,
            "Attached; node /dev/%s (alias /dev/myfirst), cbuf=%zu bytes\n",
            devtoname(sc->cdev), cbuf_size(&sc->cb));
        return (0);

fail_cb:
        cbuf_destroy(&sc->cb);
fail_mtx:
        mtx_destroy(&sc->mtx);
        sc->is_attached = 0;
        return (error);
}

static int
myfirst_detach(device_t dev)
{
        struct myfirst_softc *sc;

        sc = device_get_softc(dev);

        mtx_lock(&sc->mtx);
        if (sc->active_fhs > 0) {
                mtx_unlock(&sc->mtx);
                device_printf(dev,
                    "Cannot detach: %d open descriptor(s)\n",
                    sc->active_fhs);
                return (EBUSY);
        }
        mtx_unlock(&sc->mtx);

        if (sc->cdev_alias != NULL) {
                destroy_dev(sc->cdev_alias);
                sc->cdev_alias = NULL;
        }
        if (sc->cdev != NULL) {
                destroy_dev(sc->cdev);
                sc->cdev = NULL;
        }
        sysctl_ctx_free(&sc->sysctl_ctx);
        cbuf_destroy(&sc->cb);
        mtx_destroy(&sc->mtx);
        sc->is_attached = 0;
        return (0);
}
```

两个新的 sysctl 处理程序很短：

```c
static int
myfirst_sysctl_cb_used(SYSCTL_HANDLER_ARGS)
{
        struct myfirst_softc *sc = arg1;
        unsigned int val;

        mtx_lock(&sc->mtx);
        val = (unsigned int)cbuf_used(&sc->cb);
        mtx_unlock(&sc->mtx);
        return (sysctl_handle_int(oidp, &val, 0, req));
}

static int
myfirst_sysctl_cb_free(SYSCTL_HANDLER_ARGS)
{
        struct myfirst_softc *sc = arg1;
        unsigned int val;

        mtx_lock(&sc->mtx);
        val = (unsigned int)cbuf_free(&sc->cb);
        mtx_unlock(&sc->mtx);
        return (sysctl_handle_int(oidp, &val, 0, req));
}
```

这些处理程序存在是因为我们想要在用户读取 `sysctl dev.myfirst.0.stats.cb_used` 时获得缓冲区状态的*一致*快照。直接读取字段（阶段 3 对 `bufused` 的做法）是有竞争的：并发写入可能在 `sysctl(8)` 读取时正在修改它，产生撕裂值。处理程序在读取周围持有互斥锁，所以用户看到的值至少是*自洽的*（它代表某一时刻的缓冲区状态，而不是半更新）。当然，缓冲区可以在处理程序释放锁后立即改变；那没问题，因为到 `sysctl(8)` 格式化并打印数字时，缓冲区通常已经改变了。我们防止的是读取部分修改的字段，而不是陈旧读取。

### 记录缓冲区状态以供调试

当驱动行为异常时，第一个问题几乎总是"缓冲区在做什么？"向 I/O 处理程序添加少量 `device_printf` 流量，在 sysctl 控制的调试标志后面，使这个问题容易回答。以下是模式：

```c
static int myfirst_debug = 0;
SYSCTL_INT(_dev_myfirst, OID_AUTO, debug, CTLFLAG_RW,
    &myfirst_debug, 0, "Verbose I/O tracing for the myfirst driver");

#define MYFIRST_DBG(sc, fmt, ...) do {                                  \
        if (myfirst_debug)                                              \
                device_printf((sc)->dev, fmt, ##__VA_ARGS__);           \
} while (0)
```

然后在 I/O 处理程序中，在成功的 `cbuf_read` 之后调用 `MYFIRST_DBG(sc, "read got=%zu used=%zu free=%zu\n", got, cbuf_used(&sc->cb), cbuf_free(&sc->cb));`。`myfirst_debug` 设置为 0 时，宏退化为无操作，生产路径不受影响。`sysctl dev.myfirst.debug=1` 时，每次传输向 `dmesg` 打印一行跟踪，这在驱动做你不理解的事情时是无价的。

对你发出多少跟踪要有礼貌。每次传输一条日志行就可以了。每传输一个字节一条日志行会在几秒钟内融化 `dmesg` 的环形缓冲区，并且会改变驱动的时序足以隐藏一些 bug。上面的模式在每次 `cbuf_read` 或 `cbuf_write` 调用时记录一次，即每次循环迭代一次，即每块最多 256 字节一次。这大约是正确的粒度。

最后，记住在生产环境中加载驱动之前设置 `myfirst_debug = 0`。这行作为开发辅助存在，不是永久特性。

### 为什么 cbuf 有自己的内存标签

当你在 FreeBSD 系统上运行 `vmstat -m` 时，你会看到一个长长的内存标签列表以及每个标签当前持有的内存量。标签是基本的可观测性工具：如果内核某处内存泄漏，计数持续增长的标签告诉你去哪里找。我们给 `cbuf` 自己的标签（`M_CBUF`），使其分配与驱动其余部分的分配分开可见。

要查看效果，加载阶段 2 驱动并运行：

```sh
$ vmstat -m | head -1
         Type InUse MemUse Requests  Size(s)
$ vmstat -m | grep -E '(^\s+Type|cbuf|myfirst)'
         Type InUse MemUse Requests  Size(s)
         cbuf     1      4K        1  4096
```

这四千字节对应 `cbuf_init` 为 `sc->cb.cb_data` 做的单次 4 KB 分配。卸载驱动，计数回落到零。如果在任何时候计数在没有相应驱动挂载的情况下上升，你就有 `cbuf_init` 或 `cbuf_destroy` 中的泄漏。这种回归否则会在数小时后系统内存耗尽之前不可见。

### 对齐传输和环绕传输的快速跟踪

为了使环绕行为真实，让我们跟踪阶段 2 驱动中的两次写入。假设缓冲区开始时为空，容量为 4096，用户调用 `write(fd, buf, 100)` 随后调用 `write(fd, buf2, 100)`。

第一次写入通过 `myfirst_write`：

1. `uio_resid = 100`, `cbuf_free = 4096`, `room = 4096`.
2. Loop iteration 1: `want = MIN(100, 256, 4096) = 100`. `uiomove` copies 100 bytes from user space into `bounce`. `cbuf_write(&sc->cb, bounce, 100)` returns 100, advances `cb_used` to 100, leaves `cb_head = 0`. The implicit tail is now 100.
3. `uio_resid = 0`. The loop exits. The handler returns 0. The user sees a write count of 100.

缓冲区状态为：`cb_data[0..99]` 保存数据，`cb_head = 0`，`cb_used = 100`，`cb_size = 4096`。

现在第二次写入到达。在它之前，假设读取者已消耗 80 字节，留下 `cb_head = 80`，`cb_used = 20`。隐式尾部在位置 100。`myfirst_write` 运行：

1. `uio_resid = 100`, `cbuf_free = 4076`, `room = 4076`.
2. Loop iteration 1: `want = MIN(100, 256, 4076) = 100`. `uiomove` copies 100 bytes from user space into `bounce`. `cbuf_write(&sc->cb, bounce, 100)` advances the implicit tail from 100 to 200, sets `cb_used = 120`, returns 100.
3. `uio_resid = 0`. The handler returns 0. The user sees a write count of 100.

两次传输都是"对齐的"，因为都没有跨越底层缓冲区末尾。现在想象一个更晚的状态，`cb_head = 4000` 且 `cb_used = 80`。活跃字节占据位置 4000..4079，隐式尾部在 4080。容量为 4096。空闲空间是 4016 字节，但它跨越环绕分割：4080 之后连续 16 字节，然后从位置 0 连续 4000 字节。

用户调用 `write(fd, buf, 64)`：

1. `uio_resid = 64`, `cbuf_free = 4016`, `room = 4016`.
2. Loop iteration 1: `want = MIN(64, 256, 4016) = 64`. `uiomove` copies 64 bytes into `bounce`. `cbuf_write(&sc->cb, bounce, 64)` runs:
   - `tail = (4000 + 80) % 4096 = 4080`.
   - `first = MIN(64, 4096 - 4080) = 16`. Copies 16 bytes from `bounce + 0` to `cb_data + 4080`.
   - `second = 64 - 16 = 48`. Copies 48 bytes from `bounce + 16` to `cb_data + 0`.
   - `cb_used += 64`, becoming 144.
3. `uio_resid = 0`. Handler returns 0.

环绕在 `cbuf_write` 内部处理，对驱动不可见。这就是将抽象放在自己文件中的全部意义。`myfirst.c` 源码没有环绕算术；环绕在 `cbuf.c` 中，可以独立测试。

### 用户看到什么

拼接后，用户空间程序无法辨别缓冲区是什么形状。`cat /dev/myfirst` 仍然按顺序打印已写入的内容。`echo hello > /dev/myfirst` 仍然存储 `hello` 以供稍后读取。`sysctl dev.myfirst.0.stats` 中的字节计数器仍然每个字节递增一。新的 `cb_used` 和 `cb_free` sysctl 暴露缓冲区状态，但数据路径与第9章阶段 3 逐字节相同。

不同的是*在负载下*发生的事情。使用线性 FIFO，持续的写入者最终会看到 `ENOSPC`，即使读取者正在积极消耗字节，因为 `bufhead` 只有在 `bufused` 达到零时才折回零。使用循环缓冲区，只要读取者跟上，写入者就无限期地继续，因为空闲空间和活跃字节可以占据底层内存内任何位置组合。缓冲区的全部容量现在真正可用了。

你将在第6节中清楚地看到这种差异，当我们对新驱动运行 `dd` 并将吞吐量数字与第9章阶段 3 比较时。现在，先相信它，完成拼接。

### 处理带活跃数据的拆离

有一个拆离时的微妙之处值得讨论。集成循环缓冲区后，当用户运行 `kldunload myfirst` 时缓冲区可能持有数据。第9章的拆离拒绝在任何描述符打开时卸载；该检查仍然适用。然而，如果缓冲区非空但没有打开的描述符，它不会拒绝卸载。应该吗？

传统的答案是不。缓冲区是瞬态资源。如果当前没有人读取设备，缓冲区中的字节不会被读取；用户通过关闭所有描述符隐式接受了它们的丢失。拆离路径只是将缓冲区与所有其他内容一起释放。如果你想在卸载期间保留字节（例如到文件中），那将是一个特性，而不是错误修复，它属于用户空间，不属于驱动。

因此我们对拆离生命周期不做更改。`cbuf_destroy` 被无条件调用；字节与后备内存一起释放。

### 第3节总结

驱动现在使用真正的循环缓冲区。环绕逻辑位于一个小型抽象中，拥有自己的头文件、源文件、内存标签和用户态测试程序。`myfirst.c` 中的 I/O 处理程序比第9章阶段 3 更简单，棘手的算术不再分散在其中。

你此时拥有的东西仍然不能优雅地处理部分读写。如果用户调用 `read(fd, buf, 4096)` 而缓冲区持有 100 字节，循环将恰好执行一次，传输 100 字节，并返回零，`uio_resid` 反映未消耗的部分。那是正确的行为，但关于用户应该期望什么、`read(2)` 返回什么以及编写良好的调用者如何循环的*文字*是第4节的内容。我们还将解决当缓冲区为空且调用者愿意等待时 `d_read` 应该做什么的问题，这是进入第5节非阻塞 I/O 的大门。

## 第4节：通过部分读写改进驱动行为

上一节的阶段 2 驱动已经正确实现了部分读写，几乎是偶然的。`myfirst_read` 和 `myfirst_write` 中的循环在循环缓冲区无法满足下一次迭代时退出，将 `uio->uio_resid` 保留为请求中未消耗的部分。内核将用户可见的字节计数计算为原始请求大小减去该剩余值。`read(2)` 和 `write(2)` 然后将该数字返回给用户空间。

我们没有做的是*清楚地思考*这些部分传输从信任边界两侧意味着什么。本节进行这种思考。到本节结束时，你将知道哪些用户空间程序正确处理短读写、哪些不处理、你的驱动在什么都没有时应该报告什么、以及罕见的零字节传输意味着什么。

### UNIX 中"部分"的含义

`read(2)` 返回三种之一：

- 一个小于或等于请求计数的*正整数*：那么多字节已放入调用者的缓冲区。
- *零*：文件结束。此描述符上永远不会再产生更多字节；调用者应该关闭。
- `-1`：发生错误；调用者检查 `errno` 来决定做什么。

第一种情况是部分传输所在。"完整"读取精确返回请求的计数。"部分"读取返回较少的字节。UNIX 一直允许部分读取，任何调用 `read(2)` 并假设获得了完整请求计数的程序都是错误的。健壮的程序总是查看返回值并循环直到获得所需的，或接受部分结果并继续。

`write(2)` 遵循相同的形状：

- 一个小于或等于请求计数的*正整数*：那么多字节已被内核接受。
- 有时是*零*（实践中很少见；通常被视为零字节的短写入）。
- `-1`：发生错误。

短写入意味着"我接受了你这么多字节；请用剩余的尾部再调用我。"健壮的生产者总是循环直到提供了整个载荷。

### 为什么驱动应该拥抱部分传输

使驱动总是满足整个请求很诱人，即使它必须内部循环或等待。一些驱动在特殊情况下这样做（考虑 `null` 驱动的读取，它内部循环传递 `ZERO_REGION_SIZE` 字节的块，直到调用者的请求耗尽）。然而，对于大多数驱动，拥抱部分传输是正确的设计选择，有几个原因。

第一个原因是**响应性**。请求 4096 字节但获得 100 字节的读取者有 100 字节的工作可以立即开始做，而不是等待可能永远不会到达的另外 3996 字节。内核不必猜测调用者愿意等待多久。

第二个原因是**公平性**。如果 `myfirst_read` 在内部循环直到满足整个请求，单个贪婪的读取者可以无限期地持有缓冲区的互斥锁，饿死每个想要访问驱动的其他线程。一旦无法取得进展就返回的处理程序让内核调度器在竞争线程之间保持公平。

第三个原因是**面对信号的正确性**。一直在等待的读取者可能收到信号（例如用户按 Ctrl-C 发出的 `SIGINT`）。内核需要有机会传递该信号，这通常意味着从当前系统调用返回。无限循环的处理程序从不给内核那个机会，用户的 `kill -INT` 被延迟或丢失。

第四个原因是**与 `select(2)` / `poll(2)` 的组合**。使用这些就绪原语的程序显式假设部分传输语义。它们期望被告知"数据就绪"，然后循环 `read(2)` 直到描述符返回零或 `EAGAIN`。总是返回完整请求计数的驱动会破坏轮询模型。

由于所有这些原因，第3节中 `myfirst` 驱动的循环被设计为对缓冲区的可用数据进行单次传递，传输能传的，然后返回。下次调用者想要更多时，它再次调用 `read(2)`。这是传统的 UNIX 形状。

### 报告准确的字节计数

驱动报告部分传输的机制是 `uio->uio_resid`。内核在调用 `d_read` 或 `d_write` 之前将其设置为请求的计数。处理程序负责在传输字节时递减它。`uiomove(9)` 自动递减它。当处理程序返回时，内核将字节计数计算为 `original_resid - uio->uio_resid` 并返回给用户空间。

这意味着处理程序必须一致地做两件事：

1. 使用 `uiomove(9)`（或其同伴之一，`uiomove_frombuf(9)`、`uiomove_nofault(9)`）执行跨越信任边界的每个字节移动。这是保持 `uio_resid` 诚实的方式。
2. 当它做了能做的所有事情时返回零，不管 `uio_resid` 现在是零还是某个正数。

返回*正*字节计数的处理程序是错误的。内核忽略正返回值；字节计数从 `uio_resid` 计算。返回正整数将是静默浪费的。返回*负*数或不在 `errno.h` 中的任何值的处理程序是未定义行为。

这个错误的一个常见且危险的变体是在缓冲区为空时返回 `EAGAIN`，而且在同一调用中早些时候已经传输了一些字节。用户空间 `read(2)` 会看到 `-1`/`EAGAIN`，用户缓冲区中的字节将被静默视为未涉及。正确的模式是：如果处理程序已经传输了任何字节，它返回 0 并让部分计数自行说明；只有在传输了*零*字节时才能返回 `EAGAIN`。第5节将在我们添加非阻塞支持时编纂这个规则。

### 数据结束：`d_read` 何时应该返回零？

UNIX 的"零意味着 EOF"规则对伪设备有一个有趣的后果。常规文件有确定的末尾：当 `read(2)` 到达时，内核返回零。字符设备通常没有确定的末尾。串行线路、键盘、网络设备、倒带超过介质末尾的磁带：这些中的每一个在特殊情况下*可能*返回零，但在正常操作中，"现在没有可用数据"与"永远不会再有数据"不同。

然而，每当缓冲区为空就返回零的天真 `myfirst_read` 从调用者的角度看，与文件结束处的常规文件无法区分。`cat /dev/myfirst` 会看到零字节，将其视为 EOF 并退出。那不是我们想要的。我们希望读取者等到更多字节到达，或者根据文件描述符的模式被告知"现在没有字节，但稍后重试"。

两种策略是常见的。

第一种策略是**默认阻塞**。`myfirst_read` 在缓冲区为空时在睡眠队列上等待，写入者在添加字节时唤醒队列。只有当某个条件发出真正文件结束的信号时（设备已被移除、写入者已显式关闭），读取才返回零。这是大多数伪设备和大多数 TTY 风格设备的做法。它匹配 `cat` 对终端将随用户输入交付行的期望。

第二种策略是**对非阻塞调用者立即返回 `EAGAIN`**。如果描述符是用 `O_NONBLOCK` 打开的（或用户稍后用 `fcntl(2)` 设置了标志），`myfirst_read` 返回 `-1`/`EAGAIN` 而不是阻塞。这让事件循环程序可以使用 `select(2)`、`poll(2)` 或 `kqueue(2)` 来多路复用许多描述符，而不必提交等待任何一个。

第5节将实现两种策略。阻塞路径是默认的；当 `ioflag` 中设置了 `IO_NDELAY` 时，非阻塞路径激活。目前，在阶段 2，驱动在空时仍然返回零，与第9章相同。那是临时状态；当数据路径随时可能消失时，用户空间中没有任何东西是稳定的。

### 写入侧的反压

"现在没有数据"的镜像就是"现在没有空间"。当缓冲区满且写入者要求添加更多字节时，驱动必须选择说什么。

第9章的阶段 3 驱动返回 `ENOSPC`，这是"设备空间已用完，永久性"的传统信号。在第9章中这是一个合理的选择，因为线性 FIFO 在缓冲区完全排空之前确实无法接受更多数据。然而，使用循环缓冲区，"满"是瞬态的：写入者只需要等到读取者消耗了一些东西。因此在稳定状态下正确的返回*不是* `ENOSPC`；它要么是阻塞睡眠直到空间出现，要么是对非阻塞调用者返回 `EAGAIN`。

阶段 2 的实现已经正确处理了部分写入情况：当缓冲区在传输中途填满时，循环退出，用户看到小于请求的写入计数。它*没有*做的是当缓冲区在调用*开始时*就满了时做正确的事情：它返回 0 且没有传输字节，内核将其转换为 `write(2)` 返回零。`write(2)` 返回零在技术上是合法的，但是一个奇怪的事情，大多数用户程序会将其视为错误或永远循环等待它变为非零。

传统的修复，同样，取决于模式。阻塞写入者应该睡眠直到空间可用；非阻塞写入者应该收到 `EAGAIN`。我们将在第5节实现两者。阶段 2 循环的结构对两种情况已经正确；缺少的是当第一次迭代*没有*取得进展时做什么的选择。

### 零长度读取和写入

零长度读取或写入是完全合法的调用。`read(fd, buf, 0)` 和 `write(fd, buf, 0)` 是有效的系统调用；它们明确存在，以便程序可以在不提交传输的情况下验证文件描述符。内核以 `uio->uio_resid == 0` 将它们向下传递给驱动。

在这种情况下你的处理程序不能崩溃、报错或循环。阶段 2 驱动自然地做了正确的事：`while (uio->uio_resid > 0)` 循环从不执行，处理程序返回 0 且 `uio_resid` 仍为 0。用户看到 `read(2)` 或 `write(2)` 返回零。调用零长度 I/O 进行描述符验证的程序得到了它们期望的结果。

在处理程序开头添加"请求是否为空？"的早期返回时要小心。它们看起来像小的优化，但它们引入了容易出错的分支。第9章速查表的规则适用：`if (uio->uio_resid == 0) return (EINVAL);` 是一个 bug。

### 用户空间的循环演练

观察用户程序如何处理部分传输是内化契约的最佳方式。以下是使用惯用 UNIX 风格编写的小型读取器：

```c
static int
read_all(int fd, void *buf, size_t want)
{
        char *p = buf;
        size_t left = want;
        ssize_t n;

        while (left > 0) {
                n = read(fd, p, left);
                if (n < 0) {
                        if (errno == EINTR)
                                continue;
                        return (-1);
                }
                if (n == 0)
                        break;          /* EOF */
                p += n;
                left -= n;
        }
        return (int)(want - left);
}
```

`read_all` 持续调用 `read(2)` 直到获得所有 `want` 字节，或看到文件结束，或看到真正的错误。短读取被透明地吸收。来自信号的 `EINTR` 导致重试。函数返回实际获得的字节数。

正确编写的 `write_all` 是镜像：

```c
static int
write_all(int fd, const void *buf, size_t have)
{
        const char *p = buf;
        size_t left = have;
        ssize_t n;

        while (left > 0) {
                n = write(fd, p, left);
                if (n < 0) {
                        if (errno == EINTR)
                                continue;
                        return (-1);
                }
                if (n == 0)
                        break;          /* unexpected; treat as error */
                p += n;
                left -= n;
        }
        return (int)(have - left);
}
```

`write_all` 反复调用 `write(2)` 直到整个载荷被内核接受。短写入被透明地吸收。`EINTR` 导致重试。函数返回接受的字节数。

两个辅助函数属于同一文件（或共享工具头文件），因为它们几乎总是一起使用。它们简短、健壮，使与你的驱动通信的用户空间代码在驱动进行部分传输时也能正确行为。我们将在第6节你构建的测试程序中使用两者。

### `cat`、`dd` 和朋友实际做什么

你一直用来测试驱动的基本系统工具各自以不同方式处理短读写。了解每个做什么值得，这样你就可以解释你看到的东西。

`cat(1)` 使用 `MAXBSIZE`（FreeBSD 14.3 上为 16 KB）的缓冲区读取，并在循环中写入它获得的任何内容。源描述符的短读取被吸收；`cat` 只需再进行一次 `read(2)` 调用。目标描述符的短写入也被吸收；`cat` 在后续调用中写入未消耗的尾部。对 `cat` 来说，传输的大小不重要；它只是持续移动字节直到在源上看到文件结束。

`dd(1)` 更严格。它以 `bs=` 字节（默认 512）的块读取，并以相同的块大小写入它接收到的内容。关键是，`dd` 默认*不*在短读取上循环。如果 `read(2)` 在 `bs=4096` 时返回 100 字节，`dd` 写入一个 100 字节的块并递增其短读计数器。你最终看到的输出（`X+Y records in / X+Y records out`）分为完整记录（`X`）和短记录（`Y`）。总字节计数是重要的；分割告诉你源是否在产生短读取。

有一个 `dd` 标志 `iflag=fullblock`，使它像 `cat` 那样在源上循环。当你想要在没有短读噪音的情况下测试吞吐量时使用它：`dd if=/dev/myfirst of=/dev/null bs=4k iflag=fullblock`。没有该标志，你将看到每次短读分割的记录。

`hexdump(1)` 默认一次读取一个字节，但可以被告知读取更大的块。它不关心源的短读取。

`truss(1)` 追踪每个系统调用，包括每个返回的字节计数。在 `truss` 下运行生产者或消费者是查看你的驱动返回什么字节计数的最直接方式。如果你运行 `truss -f -t read,write cat /dev/myfirst`，输出将确切告诉你每次 `read(2)` 返回了多少字节，你可以将其与 `sysctl` 中的 `cb_used` 关联。

### 部分传输代码中的常见错误

以下是初学者驱动代码中最常出现的错误。每个都有相同的形状：处理程序做了在单个测试用例中看起来合理但在负载下静默错误行为的事情。

**错误 1：从 `d_read` 或 `d_write` 返回字节计数。** 执行 `return ((int)nbytes);` 而不是 `return (0);` 的处理程序是错误的。内核忽略正值（因为正返回不是有效的 errno 值）并从 `uio_resid` 计算字节计数。返回 `nbytes` 并*同时*用 `uiomove` 做正确事情的处理程序偶然地工作；返回 `nbytes` 并跳过 `uiomove` 步骤的处理程序静默损坏数据。不要发明你自己的返回约定。

**错误 2：部分传输后返回 `EAGAIN`。** 已经从 `uio` 消耗了一些字节然后因为不再有可用字节而返回 `EAGAIN` 的处理程序静默丢弃了用户已经获得的字节。正确的规则是：如果你传输了任何字节，返回 0；只有在你传输了零字节时才能返回像 `EAGAIN` 这样的 errno。

**错误 3：拒绝零长度传输。** 如上所述，`read(fd, buf, 0)` 和 `write(fd, buf, 0)` 是合法的。在零 `uio_resid` 上返回 `EINVAL` 的处理程序会破坏使用零长度 I/O 进行描述符验证的程序。

**错误 4：缓冲区为空时在处理程序内部循环。** 在内核内部旋转等待数据出现的处理程序阻塞了调用线程*以及*每个想要获取相同锁的线程。等待的正确机制是 `mtx_sleep(9)` 或 `cv_wait(9)`，而不是忙循环。第5节涵盖这一点。

**错误 5：跨 `uiomove` 持有缓冲区互斥锁。** 这是初学者驱动代码中最常见的单一 bug。`uiomove` 可能因页面故障而睡眠。持有非可睡眠互斥锁睡眠在 `INVARIANTS` 启用的内核上是 `KASSERT` 崩溃，在 `WITNESS` 启用的内核上是 `WITNESS` 警告；在没有两者之一构建的生产内核上，相同的模式仍然可能在页面故障尝试换入用户页面时死锁机器或静默损坏状态。无论哪种方式，行为都是错误的，测试内核应该在生产之前捕获它。阶段 2 处理程序在调用 `uiomove` 之前小心地释放互斥锁。在你编写的每个新处理程序中重复该模式。

**错误 6：不遵守用户的信号。** 不将 `PCATCH` 传递给 `mtx_sleep(9)` 或 `tsleep(9)` 的阻塞处理程序不能被信号中断。用户的 Ctrl-C 被静默忽略，只有 `kill -9` 才能释放线程。总是允许信号中断等待，并总是干净地处理由此产生的 `EINTR`。

**错误 7：在失败后信任 `uio->uio_resid`。** 当 `uiomove` 返回非零错误时（例如，因为用户空间缓冲区无效的 `EFAULT`），`uio_resid` 可能被部分递减或完全递减，取决于故障发生在传输的哪个位置。约定是：传播错误，不重试，接受用户看到的字节计数可能包括故障到达之前的一些字节。这在实践中很少见，用户得到 `EFAULT` 加上让他们恢复的字节计数。

### 具体示例：观察部分读取

为了使其真实，加载阶段 2 驱动，向其中写入几百字节，并观察小型读取器分块收集它们. 驱动加载后:

```sh
$ printf 'aaaaaaaaaaaaaaaaaaaa' > /dev/myfirst              # 20 bytes
$ printf 'bbbbbbbbbbbbbbbbbbbb' > /dev/myfirst              # 20 more
$ sysctl dev.myfirst.0.stats.cb_used
dev.myfirst.0.stats.cb_used: 40
```

缓冲区保存 40 字节. 现在运行小型读取器，由 `truss` 追踪:

```c
/* shortreader.c */
#include <fcntl.h>
#include <stdio.h>
#include <unistd.h>

int
main(void)
{
        int fd = open("/dev/myfirst", O_RDONLY);
        char buf[1024];
        ssize_t n;

        n = read(fd, buf, sizeof(buf));
        printf("read 1: %zd\n", n);
        n = read(fd, buf, sizeof(buf));
        printf("read 2: %zd\n", n);
        close(fd);
        return (0);
}
```

```sh
$ cc -o shortreader shortreader.c
$ truss -t read,write ./shortreader
... read(3, ...) = 40 (0x28)
read 1: 40
... read(3, ...) = 0 (0x0)
read 2: 0
```

第一次 `read(2)` 返回了 40，即使用户请求了 1024. 这是部分读取，它是正确的. 第二次 `read(2)` 返回 0，因为缓冲区为空. 在阶段 2 中，零是"现在没有数据"的替代；在阶段 3（我们添加阻塞后），第二次读取将睡眠直到更多数据到达.

现在用更紧的缓冲区做同样的操作，以在更大的传输上看到部分读取:

```sh
$ dd if=/dev/zero bs=1m count=8 | dd of=/dev/myfirst bs=4096 2>/tmp/dd-w &
$ dd if=/dev/myfirst of=/dev/null bs=512 2>/tmp/dd-r
```

当写入者以比读取者消耗 512 字节块更快的速度产生 4096 字节块时，缓冲区填满. 写入者的 `write(2)` 调用开始返回短计数，`dd` 将每次短调用记录为部分记录. 读取者持续每次读取 512. 当你停止两个进程时，查看 `/tmp/dd-w` 中的 `records out` 行和 `/tmp/dd-r` 中的 `records in` 行；每行的第二个数字是短记录计数.

这是健康的行为. 驱动正在做 UNIX 设备应该做的事：让每一方以自己的速度进行，诚实地报告部分传输，并在没有什么可等待时从不阻塞. 没有部分传输语义，写入者会遇到 `ENOSPC`（第9章的行为），`dd` 会停止.

### 第4节总结

驱动的读写处理程序现在正确地具有部分传输感知. 我们没有更改第3节的代码；我们只是使行为显式化并建立了讨论它所需的词汇. 你知道当只有部分字节可用时 `read(2)` 和 `write(2)` 返回什么, 你知道如何编写处理这些返回的用户空间循环, 你知道哪些基本系统工具优雅地处理部分传输，哪些需要标志.

仍然缺失的是缓冲区*完全*为空（对于读取）或*完全*满（对于写入）时的正确行为. 阶段 2 驱动仍然返回零或无进展地停止；那是我们即将添加的更正确行为的替代. 第5节介绍非阻塞 I/O、阻塞睡眠路径和 `EAGAIN`. 在那之后，驱动将在填充状态和调用者模式的所有组合下正确行为.

## 第5节：实现非阻塞 I/O

到目前为止，当调用者请求无法立即满足的传输时，驱动一直做两件事之一. 它返回零（在读取时，模仿文件结束）或在中途停止且没有传输字节（在写入时，告诉用户"接受了零字节"）. 这两种行为都不是真正的字符设备应该做的. 本节用两种正确行为替换两者: a blocking wait for the default case, and a clean `EAGAIN` for callers that opened the descriptor non-blocking.

在触及驱动之前，让我们确保理解"非阻塞"在信任边界两侧意味着什么 from each side of the trust boundary. That vocabulary is what ties the implementation together.

### 什么是非阻塞 I/O

**阻塞**描述符是 `read(2)` 和 `write(2)` 允许睡眠的描述符. 如果驱动没有可用数据，`read(2)` 等待；如果驱动没有可用空间，`write(2)` 等待. 调用线程被挂起，可能很长时间，直到能取得进展. 这是 UNIX 中每个文件描述符的默认行为.

**非阻塞**描述符是 `read(2)` 和 `write(2)` 必须*从不*睡眠的描述符. 如果驱动现在没有数据，`read(2)` 返回 `-1` 且 `errno = EAGAIN`. 如果驱动现在没有空间，`write(2)` 返回 `-1` 且 `errno = EAGAIN`. 调用者应该做其他事情 (通常是调用 `select(2)`、`poll(2)` 或 `kqueue(2)` 来发现描述符何时就绪) 然后再试一次.

开启或关闭非阻塞模式的每次描述符标志是 `O_NONBLOCK`. 程序可以在 `open(2)` 时设置它 (`open(path, O_RDONLY | O_NONBLOCK)`) 或稍后用 `fcntl(2)` 设置 (`fcntl(fd, F_SETFL, O_NONBLOCK)`). 标志存在于描述符的 `f_flag` 字段中, 它对文件结构私有；驱动不直接看到标志.

驱动*确实*看到的是 `d_read` 和 `d_write` 的 `ioflag` 参数. devfs 层将描述符的标志转换为 `ioflag` 的位 处理程序可以检查. 具体来说:

- 当描述符有 `O_NONBLOCK` 时设置 `IO_NDELAY`.
- 当描述符有 `O_DIRECT` 时设置 `IO_DIRECT`.
- 当描述符有 `O_FSYNC` 时在 `d_write` 上设置 `IO_SYNC`.

转换比看起来更简单. /usr/src/sys/fs/devfs/devfs_vnops.c 中的 `CTASSERT` 声明 `O_NONBLOCK == IO_NDELAY`. 位值的选择使两个名称可互换, 你可以根据哪个约定更清晰写 `(ioflag & IO_NDELAY)` 或 `(ioflag & O_NONBLOCK)`. 两者都有效. FreeBSD 源码树更常使用 `IO_NDELAY`，所以我们遵循它.

### 非阻塞行为何时有用

非阻塞模式是使事件驱动程序成为可能的底层机制. 没有它，想要从多个描述符读取的单线程必须选择一个、阻塞在上面、并忽略其他直到它醒来. 有了它，单线程可以测试多个描述符的就绪状态, 处理就绪的那个，并循环回去而不在任何单个描述符上提交睡眠.

三个常见程序严重依赖此模式. 经典事件循环（`libevent`、`libev` 或 FreeBSD 中现在标准的基于 `kqueue` 的模式）只是在 `kevent(2)` 中等待事件、分派它、然后循环. 网络守护程序（`nginx`、`haproxy`）使用相同的形状在每线程中管理数千个连接. 实时应用（音频处理、工业控制）需要有界的最坏情况延迟，不能承受长时间阻塞.

想要与这些程序良好配合的驱动必须正确实现非阻塞模式. 返回错误的 errno、在设置 `IO_NDELAY` 时睡眠、或在状态改变时忘记通知 `poll(2)`，每个都会产生难以诊断的 bug.

### IO_NDELAY 标志：它如何流向驱动

追踪一次流程，这样你知道标志从哪里来. 用户在设置了 `O_NONBLOCK` 的描述符上调用 `read(fd, buf, n)`. Inside the kernel:

1. `sys_read` 查找文件描述符并找到 `fp->f_flag` 包含 `O_NONBLOCK` 的 `struct file`.
2. `vn_read` 或（对于字符设备）`devfs_read_f` 通过屏蔽 `fp->f_flag` 中驱动关心的位来组装 `ioflag`. 具体来说，它计算 `ioflag = fp->f_flag & (O_NONBLOCK | O_DIRECT);`.
3. 计算出的 `ioflag` 传递给驱动的 `d_read`.

从驱动的角度来看，转换已完成: `ioflag & IO_NDELAY` 为真当且仅当调用者想要非阻塞语义. 缺失的位意味着需要时阻塞. 额外的位意味着非阻塞并在需要时返回 EAGAIN.

在写入侧相同的模式适用. `devfs_write_f` 计算 `ioflag = fp->f_flag & (O_NONBLOCK | O_DIRECT | O_FSYNC);` 并传入. 写入处理程序的检查是对称的: `ioflag & IO_NDELAY` 是"不要阻塞"

### EAGAIN 约定

当驱动的处理程序决定它无法取得进展且调用者是非阻塞的，它返回 `EAGAIN`. 内核的通用层在用户级别将其传递为 `-1` / `errno = EAGAIN`. 用户应该将 `EAGAIN` 视为"此描述符未就绪；等待或稍后重试"，而不是传统意义上的错误.

关于 `EAGAIN` 的两个细节值得记住.

首先，`EAGAIN` 和 `EWOULDBLOCK` 在 FreeBSD 中是相同的值. 它们是单个 errno 的两个名称. 一些较旧的手册页在套接字相关上下文中使用 `EWOULDBLOCK`，在文件相关上下文中使用 `EAGAIN`; 兼容性紧密，驱动代码中任一名称都可接受. FreeBSD 源码树几乎只在驱动中使用 `EAGAIN`.

其次，`EAGAIN` 必须只在处理程序传输了*零*字节时返回. 如果处理程序已通过 `uiomove` 移动了一些字节，然后想停止因为现在没有更多可以移动，它必须返回 0（不是 `EAGAIN`）. 内核将从 `uio_resid` 计算部分字节计数并将其传递给用户. 用户的后续调用然后会看到 `EAGAIN`，因为缓冲区仍然为空. 规则是：`EAGAIN` 意味着"此调用完全没有进展"；部分传输意味着"有进展，但少于请求，现在你需要为剩余部分重试"

这正是第4节介绍的规则. 这里我们在代码中实现它.

### 阻塞路径：mtx_sleep(9) 和 wakeup(9)

阻塞路径是没有 `O_NONBLOCK` 的描述符的默认行为. 当缓冲区为空时，读取者睡眠；当写入者添加字节时，它唤醒读取者. FreeBSD 提供了一对与互斥锁组合的原语.

`mtx_sleep(void *chan, struct mtx *mtx, int priority, const char *wmesg, sbintime_t timo)` 将调用线程在"通道" `chan`（用作键的任意地址）上睡眠，原子地释放 `mtx`. 当线程醒来时，它在返回前重新获取 `mtx`. `priority` 参数可以包含 `PCATCH` 以允许信号传递中断睡眠, 而 `wmesg` 是在 `ps -AxH` 和类似工具中显示的短人类可读名称. `timo` 参数指定最大睡眠时间；零意味着无超时.

`wakeup(void *chan)` 唤醒*所有*在 `chan` 上睡眠的线程. `wakeup_one(void *chan)` 只唤醒一个. 对于单读取者驱动，`wakeup` 没问题；对于我们想将一块工作交给一个读取者的多读取者驱动，`wakeup_one` 通常是正确的. 对于 `myfirst` 我们将使用 `wakeup`，因为我们可能同时有生产者和消费者在等待，我们想确保两者都不被饿死.

两者之间的契约是睡眠者必须持有互斥锁、检查条件、并调用 `mtx_sleep`，*不*在中间释放互斥锁. `mtx_sleep` 原子地释放锁并睡眠；当它返回时，锁被重新获取，睡眠者必须重新检查条件（虚假唤醒是可能的；并发线程可能已取走我们等待的字节）. 模式是经典的 `while (condition) mtx_sleep(...)` 循环.

我们驱动中最小的阻塞读取如下所示:

```c
mtx_lock(&sc->mtx);
while (cbuf_used(&sc->cb) == 0) {
        if (ioflag & IO_NDELAY) {
                mtx_unlock(&sc->mtx);
                return (EAGAIN);
        }
        error = mtx_sleep(&sc->cb, &sc->mtx, PCATCH,
            "myfrd", 0);
        if (error != 0) {
                mtx_unlock(&sc->mtx);
                return (error);
        }
        if (!sc->is_attached) {
                mtx_unlock(&sc->mtx);
                return (ENXIO);
        }
}
/* ... now proceed to read from the cbuf ... */
```

四点值得评论.

第一个是 **while 循环中的条件**. 我们检查 `cbuf_used(&sc->cb) == 0`. 只要它为真，我们就睡眠. while 检查是必要的：`mtx_sleep` 可以因"数据出现"以外的原因返回（信号、超时、虚假唤醒，或其他线程在我们之前消耗了数据）. 每次从 `mtx_sleep` 返回后，我们必须重新检查.

第二个是 **EAGAIN 路径**. 如果调用者是非阻塞的且缓冲区为空，我们释放锁并返回 `EAGAIN` 而不睡眠. 检查必须在 `mtx_sleep` *之前*发生，而不是之后；否则我们会睡眠、醒来，然后发现调用者一直是非阻塞的.

第三个是 **PCATCH**. 有了 `PCATCH`，如果信号被传递，`mtx_sleep` 可以返回 `EINTR` 或 `ERESTART`. 将返回传播给用户是 `PCATCH` 的全部目的：我们希望用户的 Ctrl-C 实际中断读取. 没有 `PCATCH`，`SIGINT` 被保持直到睡眠因其他原因完成，用户得到长时间的、无法解释的挂起.

第四个是 **拆离检查**. `mtx_sleep` 返回后，可能 `myfirst_detach` 已开始且 `sc->is_attached` 现在为零. 我们检查并在如果是时返回 `ENXIO`. 这防止读取针对部分拆除的驱动继续进行. 拆离代码路径必须在拆除互斥锁之前调用 `wakeup(&sc->cb)` 来释放任何睡眠者；我们将在下面添加该调用.

### The Writer Side

写入路径是镜像:

```c
mtx_lock(&sc->mtx);
while (cbuf_free(&sc->cb) == 0) {
        if (ioflag & IO_NDELAY) {
                mtx_unlock(&sc->mtx);
                return (EAGAIN);
        }
        error = mtx_sleep(&sc->cb, &sc->mtx, PCATCH,
            "myfwr", 0);
        if (error != 0) {
                mtx_unlock(&sc->mtx);
                return (error);
        }
        if (!sc->is_attached) {
                mtx_unlock(&sc->mtx);
                return (ENXIO);
        }
}
/* ... now proceed to write into the cbuf ... */
```

同样的四点适用：在 `while` 循环中检查条件、在睡眠前处理 `IO_NDELAY`、传递 `PCATCH`、在睡眠后重新检查 `is_attached`。注意两个睡眠者使用相同的"通道"（`&sc->cb`）。这是刻意的。当读取者从缓冲区传输字节时，它调用 `wakeup(&sc->cb)` 解除任何等待空间的写入者阻塞。当写入者向缓冲区传输字节时，它调用 `wakeup(&sc->cb)` 解除任何等待数据的读取者阻塞。单个唤醒"此缓冲区上所有内容"的通道简单且正确。

一些驱动使用两个单独的通道（一个用于读取者，一个用于写入者），这样读取者的 `wakeup` 只打扰写入者，反之亦然。当你有许多读取者或许多写入者时，这是一个有效的优化。对于预期用途是一个生产者和一个消费者的伪设备，单个通道既简单又足够。

### 完整的阶段 3 处理程序

将非阻塞检查放入阶段 2 处理程序得到阶段 3. 完整形状如下:

```c
static int
myfirst_read(struct cdev *dev, struct uio *uio, int ioflag)
{
        struct myfirst_softc *sc = dev->si_drv1;
        struct myfirst_fh *fh;
        char bounce[MYFIRST_BOUNCE];
        size_t take, got;
        ssize_t nbefore;
        int error;

        error = devfs_get_cdevpriv((void **)&fh);
        if (error != 0)
                return (error);
        if (sc == NULL || !sc->is_attached)
                return (ENXIO);

        nbefore = uio->uio_resid;

        while (uio->uio_resid > 0) {
                mtx_lock(&sc->mtx);
                while (cbuf_used(&sc->cb) == 0) {
                        if (uio->uio_resid != nbefore) {
                                /*
                                 * We already transferred some bytes
                                 * in an earlier iteration; report
                                 * success now rather than block further.
                                 */
                                mtx_unlock(&sc->mtx);
                                return (0);
                        }
                        if (ioflag & IO_NDELAY) {
                                mtx_unlock(&sc->mtx);
                                return (EAGAIN);
                        }
                        error = mtx_sleep(&sc->cb, &sc->mtx, PCATCH,
                            "myfrd", 0);
                        if (error != 0) {
                                mtx_unlock(&sc->mtx);
                                return (error);
                        }
                        if (!sc->is_attached) {
                                mtx_unlock(&sc->mtx);
                                return (ENXIO);
                        }
                }
                take = MIN((size_t)uio->uio_resid, sizeof(bounce));
                got = cbuf_read(&sc->cb, bounce, take);
                sc->bytes_read += got;
                fh->reads += got;
                mtx_unlock(&sc->mtx);

                wakeup(&sc->cb);        /* space may have freed for writers */

                error = uiomove(bounce, got, uio);
                if (error != 0)
                        return (error);
        }
        return (0);
}

static int
myfirst_write(struct cdev *dev, struct uio *uio, int ioflag)
{
        struct myfirst_softc *sc = dev->si_drv1;
        struct myfirst_fh *fh;
        char bounce[MYFIRST_BOUNCE];
        size_t want, put, room;
        ssize_t nbefore;
        int error;

        error = devfs_get_cdevpriv((void **)&fh);
        if (error != 0)
                return (error);
        if (sc == NULL || !sc->is_attached)
                return (ENXIO);

        nbefore = uio->uio_resid;

        while (uio->uio_resid > 0) {
                mtx_lock(&sc->mtx);
                while ((room = cbuf_free(&sc->cb)) == 0) {
                        if (uio->uio_resid != nbefore) {
                                mtx_unlock(&sc->mtx);
                                return (0);
                        }
                        if (ioflag & IO_NDELAY) {
                                mtx_unlock(&sc->mtx);
                                return (EAGAIN);
                        }
                        error = mtx_sleep(&sc->cb, &sc->mtx, PCATCH,
                            "myfwr", 0);
                        if (error != 0) {
                                mtx_unlock(&sc->mtx);
                                return (error);
                        }
                        if (!sc->is_attached) {
                                mtx_unlock(&sc->mtx);
                                return (ENXIO);
                        }
                }
                mtx_unlock(&sc->mtx);

                want = MIN((size_t)uio->uio_resid, sizeof(bounce));
                want = MIN(want, room);
                error = uiomove(bounce, want, uio);
                if (error != 0)
                        return (error);

                mtx_lock(&sc->mtx);
                put = cbuf_write(&sc->cb, bounce, want);
                sc->bytes_written += put;
                fh->writes += put;
                mtx_unlock(&sc->mtx);

                wakeup(&sc->cb);        /* data may have appeared for readers */
        }
        return (0);
}
```

这段代码中的三个模式值得仔细研究.

第一个是内循环顶部的 **"已传输任何字节？"** 测试. 如果任何先前迭代传输了数据，`uio->uio_resid != nbefore` 为真. 当该条件成立且缓冲区现在为空（读取）或满（写入）时，我们立即返回 0 而不是阻塞. 内核将向用户空间报告部分传输，下一次调用将决定是阻塞还是返回 `EAGAIN`. 这是第4节规则在代码中的形式: 已取得进展的处理程序必须返回 0，不是 `EAGAIN`，也不是更深的阻塞.

第二个是 **进展后的 `wakeup`**. 当读取者排空字节时，空间已释放；写入者可能正在等待空间，我们唤醒它. 当写入者添加字节时，数据已出现；读取者可能正在等待数据，我们唤醒它. 每个状态变化都配有一个 `wakeup`. 缺失 `wakeup` 导致线程永远睡眠（或直到定时器触发，如果存在）；虚假的 `wakeup` 调用无害，因为 while 循环重新检查条件.

第三个是 **`mtx_unlock` 和 `uiomove` 的顺序**. 处理程序在操作 cbuf 时持有锁，然后在调用 `uiomove` *之前*释放锁. `uiomove` 可能睡眠；在互斥锁下睡眠是一个 bug. 还要注意在写入侧，处理程序在持有锁时快照 `room`，使用该快照来调整弹跳大小，并在 `uiomove` 前释放锁. 如果并发线程在处理程序从用户空间复制时修改了缓冲区，后续的 `cbuf_write` 可能存储少于 `want` 的字节（`cbuf_write` 中的钳位确保安全）. 在我们当前的单写入者设计中，此竞争从不触发，但代码免费处理它.

### 在拆离时唤醒睡眠者

我们还需要教会 `myfirst_detach` 释放任何睡眠者. The pattern is:

```c
static int
myfirst_detach(device_t dev)
{
        struct myfirst_softc *sc = device_get_softc(dev);

        mtx_lock(&sc->mtx);
        if (sc->active_fhs > 0) {
                mtx_unlock(&sc->mtx);
                device_printf(dev,
                    "Cannot detach: %d open descriptor(s)\n",
                    sc->active_fhs);
                return (EBUSY);
        }
        sc->is_attached = 0;
        wakeup(&sc->cb);                /* release any sleepers */
        mtx_unlock(&sc->mtx);

        /* ... destroy_dev, cbuf_destroy, mtx_destroy, sysctl_ctx_free ... */
        return (0);
}
```

这段代码中的两个细节是第10章特有的.

第一个是我们在调用 `wakeup` *之前*设置 `is_attached = 0`. 现在醒来的睡眠者将在阻塞循环中看到标志并返回 `ENXIO`；尚未睡眠的睡眠者将看到标志并返回 `ENXIO` 而不睡眠. 在 `wakeup` 之后设置标志将允许竞争，睡眠者重新获取锁、发现条件仍为真（缓冲区为空）、并回到睡眠，*而*拆离正在等待拆除互斥锁.

第二个是拆离检查 `active_fhs > 0` 并拒绝在任何描述符打开时继续. 这与第9章的检查相同. 这意味着睡眠者总是持有一个打开的描述符，这意味着拆离不会与睡眠者并发运行. `wakeup` 调用作为双重保险检查存在: 如果未来的重构允许在描述符仍打开时拆离，睡眠者不会被卡住.

### 为 select(2) 和 poll(2) 添加 d_poll

收到 `EAGAIN` 的非阻塞调用者需要某种方式在描述符就绪时被通知. `select(2)` 和 `poll(2)` 是经典机制；`kqueue(2)` 是现代机制. 我们将在这里实现经典的两个，并将 `kqueue` 留给第11章（`d_kqfilter` 和 `knlist` 基础设施属于那里）.

`d_poll` 处理程序形状简单:

```c
static int
myfirst_poll(struct cdev *dev, int events, struct thread *td)
{
        struct myfirst_softc *sc = dev->si_drv1;
        int revents = 0;

        mtx_lock(&sc->mtx);
        if (events & (POLLIN | POLLRDNORM)) {
                if (cbuf_used(&sc->cb) > 0)
                        revents |= events & (POLLIN | POLLRDNORM);
                else
                        selrecord(td, &sc->rsel);
        }
        if (events & (POLLOUT | POLLWRNORM)) {
                if (cbuf_free(&sc->cb) > 0)
                        revents |= events & (POLLOUT | POLLWRNORM);
                else
                        selrecord(td, &sc->wsel);
        }
        mtx_unlock(&sc->mtx);
        return (revents);
}
```

`d_poll` 接收用户感兴趣的事件，必须返回当前就绪的子集. 对于 `POLLIN`/`POLLRDNORM`（可读），如果缓冲区有任何字节，我们返回就绪. 对于 `POLLOUT`/`POLLWRNORM`（可写），如果缓冲区有任何空闲空间，我们返回就绪. 如果都不就绪，我们调用 `selrecord(td, &sc->rsel)` 或 `selrecord(td, &sc->wsel)` 注册调用线程，以便稍后唤醒它.

softc 中需要两个新字段：`struct selinfo rsel;` 和 `struct selinfo wsel;`. `selinfo` 是内核对等待 `select(2)`/`poll(2)` 的每次条件记录. 它在 `/usr/src/sys/sys/selinfo.h` 中声明.

每当缓冲区从空变为非空或从满变为非满时，读写处理程序需要匹配的 `selwakeup(9)` 调用. `selwakeup(9)` 是普通形式；FreeBSD 14.3 还暴露 `selwakeuppri(9)`，它以指定优先级唤醒注册的线程，通常用于想要延迟敏感唤醒的网络和存储代码. 对于通用伪设备，普通 `selwakeup` 是正确的默认选择. 我们将调用添加在 `wakeup(&sc->cb)` 调用旁边:

```c
/* 在 myfirst_read 中，成功 cbuf_read 后：*/
mtx_unlock(&sc->mtx);
wakeup(&sc->cb);
selwakeup(&sc->wsel);   /* 空间现在对写入者可用 */

/* 在 myfirst_write 中，成功 cbuf_write 后：*/
mtx_unlock(&sc->mtx);
wakeup(&sc->cb);
selwakeup(&sc->rsel);   /* 数据现在对读取者可用 */
```

如果你计划稍后支持 `kqueue(2)`，attach 用 `knlist_init_mtx(&sc->rsel.si_note, &sc->mtx);` 和 `knlist_init_mtx(&sc->wsel.si_note, &sc->mtx);` 初始化 `selinfo` 字段. 对于纯 `select(2)`/`poll(2)` 支持，`selinfo` 结构由 softc 分配零初始化，不需要进一步设置.

拆离必须在释放 softc 之前调用 `seldrain(&sc->rsel);` 和 `seldrain(&sc->wsel);`，拆除任何残留的选择记录.

将 `.d_poll = myfirst_poll,` 添加到 `myfirst_cdevsw` 初始化器，驱动的 `select(2)`/`poll(2)` 故事完成.

### 非阻塞调用者如何使用这一切

将各部分组合在一起，这是针对 `myfirst` 的良好编写的非阻塞读取器:

```c
int fd = open("/dev/myfirst", O_RDONLY | O_NONBLOCK);
char buf[1024];
ssize_t n;
struct pollfd pfd = { .fd = fd, .events = POLLIN };

for (;;) {
        n = read(fd, buf, sizeof(buf));
        if (n > 0) {
                /* got some bytes; process them */
        } else if (n == 0) {
                /* EOF; our driver never reaches this case yet */
                break;
        } else if (errno == EAGAIN) {
                /* no data; wait for readiness */
                poll(&pfd, 1, -1);
        } else if (errno == EINTR) {
                /* signal; retry */
        } else {
                perror("read");
                break;
        }
}
close(fd);
```

循环读取直到获得数据或 `EAGAIN`. 在 `EAGAIN` 时，它调用 `poll(2)` 等待直到内核报告描述符可读，然后循环回去. 当 `myfirst_write` 运行成功 `cbuf_write` 后的 `selwakeup(&sc->rsel)` 调用时，将报告 `POLLIN` 事件. 驱动的 `d_poll` 是内核的 `select/poll` 机制和缓冲区状态之间的桥梁.

这是事件驱动 UNIX I/O 的典型形状，你的驱动现在正确地参与其中.

### 关于 O_NONBLOCK 和 select/poll 组合的说明

理解 `select(2)` / `poll(2)` 和 `O_NONBLOCK` 如何交互是值得的. 传统规则是程序同时使用两者: 它用 `poll` 注册描述符然后从中读取. 单独使用任一有效但较少见.

如果程序使用 `O_NONBLOCK` 而没有 `poll`，它将忙等待. 每次 `EAGAIN` 时，它必须在重试前 `sleep` 或 `usleep`，浪费周期没有好理由. 这几乎总是错误的，但它工作.

如果程序使用 `poll` 而没有 `O_NONBLOCK`，`poll` 报告就绪然后 `read(2)` 进行阻塞调用. 阻塞调用在正常情况下几乎立即完成，因为条件刚刚报告就绪. 然而，在罕见情况下，内核状态在 `poll` 返回和 `read` 调用之间改变（例如另一个线程排空了缓冲区），`read` 将无限阻塞. 这是一个微妙的 bug，大多数事件驱动库通过始终将 `poll` 与 `O_NONBLOCK` 组合来防御它.

`myfirst` 驱动正确支持两种模式. 良好编写的程序组合两者；不太谨慎的程序将在简单情况下工作并具有上述边界情况.

### 观察阻塞路径的实际运行

加载阶段 3 驱动并运行快速实验:

```sh
$ kldload ./myfirst.ko
$ cat /dev/myfirst &
[1] 12345
```

`cat` 现在在 `myfirst_read` 内部阻塞，在 `&sc->cb` 上睡眠. 你可以用 `ps` 确认:

```sh
$ ps -AxH | grep cat
12345  -  S+    0:00.00 myfrd
```

`S+` 状态表示进程正在睡眠，`wmesg` 列显示 `myfrd`，这正是我们传递给 `mtx_sleep` 的字符串. 现在从另一个终端写入驱动:

```sh
$ echo hello > /dev/myfirst
```

`cat` 醒来、读取 `hello`、并打印它然后再次阻塞，或（如果写入者关闭了设备）到达文件结束并退出. 在我们当前阶段 3 中没有"写入者已关闭"机制，所以 `cat` 在打印后再次阻塞. 在其终端使用 Ctrl-C 中断它:

```sh
$ kill -INT %1
```

因为我们将 `PCATCH` 传递给 `mtx_sleep`，信号唤醒睡眠者，它返回 `EINTR`，传播给 `cat` 作为失败的 `read(2)`. `cat` 看到它、注意到信号、并干净地退出.

这是整个阻塞路径的实际运行. 没有神秘的事情发生；每一部分在源码和 `ps` 中可见.

### 阻塞路径中的常见错误

此材料中有两个错误特别常见.

**错误 1：在返回 EAGAIN 前忘记释放互斥锁。** 上面的代码在睡眠循环中每次 `return` 前显式解锁. 如果你忘记其中一个解锁，后续尝试获取互斥锁将恐慌或死锁. `WITNESS` 内核将在实验室环境中立即捕获此问题.

**错误 2：应该使用 mtx_sleep(9) 时使用了 tsleep(9)。** `tsleep` 不接受互斥锁参数；它假设调用者不持有任何互锁. 在使用 `mtx_sleep` 的驱动中，互斥锁与睡眠原子地释放；使用 `tsleep`，你必须自己释放互斥锁然后在醒来后重新获取，引入竞争窗口，生产者可以在你回到睡眠队列之前添加数据并调用 `wakeup`. `mtx_sleep` 对于持有互斥锁并想在释放时睡眠的每种情况都是正确的原语.

**错误 3：不处理 PCATCH 返回值。** 带 `PCATCH` 的 `mtx_sleep` 可以返回 `0`、`EINTR`、`ERESTART` 或 `EWOULDBLOCK`（对于超时）. 在驱动代码中，传统做法是不进一步检查就返回 `error`；当进程的信号处置允许时，内核知道如何将 `ERESTART` 转换为系统调用重启. 只在 `error == 0` 时检查值并返回 `0` 是上面阶段 3 代码中的模式.

**错误 4：对 mtx_sleep 和 wakeup 使用不同的"通道"。** 睡眠者使用 `&sc->cb` 作为通道；唤醒者必须使用完全相同的地址. 常见错误是一处使用 `sc`（softc 指针）而另一处使用 `&sc->cb`. 睡眠者永远不会醒来，直到超时触发或不同的 wakeup 恰好匹配. 仔细检查每个 `mtx_sleep` / `wakeup` 对使用相同的通道.

### 第5节总结

驱动现在正确处理阻塞和非阻塞调用者. 阻塞读取者在空缓冲区上睡眠并在写入者存入数据时醒来. 非阻塞读取者在空缓冲区上立即收到 `EAGAIN`. 对称对应用于写入者. `select(2)` 和 `poll(2)` 通过 `d_poll` 和 `selinfo` 机制得到支持，良好行为的事件循环程序现在可以将 `/dev/myfirst` 与其他描述符多路复用. 拆离在拆除驱动之前释放任何睡眠者.

你构建的是一个行为良好的字符设备. 它高效移动字节、与内核的就绪和睡眠原语合作、并遵守 UNIX I/O 的用户面对约定. 本章其余部分要做的是严格测试它（第6节）、为并发工作重构它（第7节）、并探索三个补充主题，这些主题经常与此材料一起出现在真实驱动中（`d_mmap`、零拷贝思考、预读和写合并的吞吐量模式）.

## 第6节：用用户程序测试缓冲 I/O

驱动只有你对它运行的测试一样可靠. 第7章到第9章建立了一个小型测试套件（一个短的 `rw_myfirst` 练习器，加上 `cat`、`echo`、`dd` 和 `hexdump`）. 第10章推进该套件，因为驱动现在表现的新行为（阻塞、非阻塞、部分 I/O、环绕）只在现实负载下出现. 本节构建三个新的用户空间工具，并逐步介绍你可以在每个阶段后完成的综合测试计划.

本节的工具位于 `examples/part-02/ch10-handling-io-efficiently/userland/`. 它们故意很小. 最长的在 150 行以下. 每个都存在以练习驱动现在应该处理的特定模式，每个都产生你可以读取和验证的输出.

### 我们将构建的三个工具

`rw_myfirst_nb.c` 是非阻塞测试器. 它用 `O_NONBLOCK` 打开设备、发出读取、期望 `EAGAIN`、写入一些字节、发出另一次读取、期望接收它们、并报告每一步的一行摘要. 这是端到端练习非阻塞路径的最小工具.

`producer_consumer.c` 是基于 fork 的负载工具. 它生成一个子进程以可配置的速率向驱动写入随机字节，而父进程读取并验证完整性. 目的是在真实并发负载下练习循环缓冲区的环绕和阻塞路径.

`stress_rw.c`（从第9章版本演变）是单进程压力测试器，运行一系列（块大小、传输计数）组合并打印聚合时间和字节计数统计. 目的是捕获单个交互测试不会揭示的性能悬崖.

所有三个都用我们将在最后展示的短 Makefile 编译.

### 更新 rw_myfirst 以处理更大输入

第9章的现有 `rw_myfirst` 处理文本大小的传输很好，但不以容量压力测试缓冲区. 一个简单的扩展让它接受命令行上的大小参数:

```c
/* rw_myfirst_v2.c: an incremental improvement on Chapter 9's tester. */
#include <sys/types.h>
#include <err.h>
#include <errno.h>
#include <fcntl.h>
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <unistd.h>

#define DEVPATH "/dev/myfirst"

static int
do_fill(size_t bytes)
{
        int fd = open(DEVPATH, O_WRONLY);
        if (fd < 0)
                err(1, "open %s", DEVPATH);

        char *buf = malloc(bytes);
        if (buf == NULL)
                err(1, "malloc %zu", bytes);
        for (size_t i = 0; i < bytes; i++)
                buf[i] = (char)('A' + (i % 26));

        size_t left = bytes;
        ssize_t n;
        const char *p = buf;
        while (left > 0) {
                n = write(fd, p, left);
                if (n < 0) {
                        if (errno == EINTR)
                                continue;
                        warn("write at %zu left", left);
                        break;
                }
                p += n;
                left -= n;
        }
        size_t wrote = bytes - left;
        printf("fill: wrote %zu of %zu\n", wrote, bytes);
        free(buf);
        close(fd);
        return (0);
}

static int
do_drain(size_t bytes)
{
        int fd = open(DEVPATH, O_RDONLY);
        if (fd < 0)
                err(1, "open %s", DEVPATH);

        char *buf = malloc(bytes);
        if (buf == NULL)
                err(1, "malloc %zu", bytes);

        size_t left = bytes;
        ssize_t n;
        char *p = buf;
        while (left > 0) {
                n = read(fd, p, left);
                if (n < 0) {
                        if (errno == EINTR)
                                continue;
                        warn("read at %zu left", left);
                        break;
                }
                if (n == 0) {
                        printf("drain: EOF at %zu left\n", left);
                        break;
                }
                p += n;
                left -= n;
        }
        size_t got = bytes - left;
        printf("drain: read %zu of %zu\n", got, bytes);
        free(buf);
        close(fd);
        return (0);
}

int
main(int argc, char *argv[])
{
        if (argc != 3) {
                fprintf(stderr, "usage: %s fill|drain BYTES\n", argv[0]);
                return (1);
        }
        size_t bytes = strtoul(argv[2], NULL, 0);
        if (strcmp(argv[1], "fill") == 0)
                return (do_fill(bytes));
        if (strcmp(argv[1], "drain") == 0)
                return (do_drain(bytes));
        fprintf(stderr, "unknown mode: %s\n", argv[1]);
        return (1);
}
```

有了这个工具，你可以用现实大小驱动驱动. For example:

```sh
$ ./rw_myfirst_v2 fill 4096
fill: wrote 4096 of 4096
$ sysctl dev.myfirst.0.stats.cb_used
dev.myfirst.0.stats.cb_used: 4096
$ ./rw_myfirst_v2 drain 4096
drain: read 4096 of 4096
$ sysctl dev.myfirst.0.stats.cb_used
dev.myfirst.0.stats.cb_used: 0
```

现在尝试填充缓冲区超过其容量，并观察每个阶段发生什么.

### 为什么往返测试重要

你编写的每个严肃测试都应该有*往返*组件: 向驱动写入已知模式、读回、并比较. 模式重要，因为如果你写"Hello, world!"十次，你无法分辨缓冲区是得到了 140 字节的"Hello, world!"还是 130 或 150 或某种奇怪的交错. 唯一的每位置模式（如上面的 `'A' + (i % 26)`）让你一目了然地发现错位、缺失字节和重复字节.

往返测试对循环缓冲区特别重要，因为环绕算术是初学者代码出错的地方. 推到底层存储末尾之外的写入和从开始之前提取的读取是你最想捕获的两种失败模式. 两者都表现为"我读的字节不是我写的字节"，往返测试立即使它们可见.

### 构建 rw_myfirst_nb

这是非阻塞测试器. 它比前一个文件稍长，但仍足够短可以一次读完.

```c
/* rw_myfirst_nb.c: non-blocking behaviour tester for /dev/myfirst. */
#include <sys/types.h>
#include <err.h>
#include <errno.h>
#include <fcntl.h>
#include <poll.h>
#include <stdio.h>
#include <string.h>
#include <unistd.h>

#define DEVPATH "/dev/myfirst"

int
main(void)
{
        int fd, error;
        ssize_t n;
        char rbuf[128];
        struct pollfd pfd;

        fd = open(DEVPATH, O_RDWR | O_NONBLOCK);
        if (fd < 0)
                err(1, "open %s", DEVPATH);

        /* Expect EAGAIN when the buffer is empty. */
        n = read(fd, rbuf, sizeof(rbuf));
        if (n < 0 && errno == EAGAIN)
                printf("step 1: empty-read returned EAGAIN (expected)\n");
        else
                printf("step 1: UNEXPECTED read returned %zd errno=%d\n", n, errno);

        /* poll(POLLIN) with timeout 0 should show not-readable. */
        pfd.fd = fd;
        pfd.events = POLLIN;
        pfd.revents = 0;
        error = poll(&pfd, 1, 0);
        printf("step 2: poll(POLLIN, 0) = %d revents=0x%x\n",
            error, pfd.revents);

        /* Write some bytes. */
        n = write(fd, "hello world\n", 12);
        printf("step 3: wrote %zd bytes\n", n);

        /* poll(POLLIN) should now show readable. */
        pfd.events = POLLIN;
        pfd.revents = 0;
        error = poll(&pfd, 1, 0);
        printf("step 4: poll(POLLIN, 0) = %d revents=0x%x\n",
            error, pfd.revents);

        /* Non-blocking read should now succeed. */
        memset(rbuf, 0, sizeof(rbuf));
        n = read(fd, rbuf, sizeof(rbuf));
        if (n > 0) {
                rbuf[n] = '\0';
                printf("step 5: read %zd bytes: %s", n, rbuf);
        } else
                printf("step 5: UNEXPECTED read returned %zd errno=%d\n",
                    n, errno);

        close(fd);
        return (0);
}
```

对阶段 3（非阻塞支持）的预期输出是:

```text
step 1: empty-read returned EAGAIN (expected)
step 2: poll(POLLIN, 0) = 0 revents=0x0
step 3: wrote 12 bytes
step 4: poll(POLLIN, 0) = 1 revents=0x41
step 5: read 12 bytes: hello world
```

第4步的 `0x41` 是 `POLLIN | POLLRDNORM`，正是我们的 `d_poll` 处理程序在缓冲区有活跃字节时设置的.

如果第1步失败（即 `read(2)` 返回 `0` 而不是 `-1`/`EAGAIN`），你的驱动仍在运行阶段 2 语义. 回去在处理程序中添加 `IO_NDELAY` 检查.

如果第2步以 `revents != 0` 成功，你的 `d_poll` 错误地在空缓冲区上报告可读. 检查 `myfirst_poll` 中的条件.

如果第4步返回零（即 `poll(2)` 没有发现描述符可读），你的 `d_poll` 没有正确反映缓冲区状态，或者写入路径中缺少 `selwakeup` 调用.

这是三个最常见的非阻塞 bug. 测试器在不到五十行输出中捕获所有这些.

### 构建 producer_consumer.c

这是基于 fork 的负载工具. 形状很简单：fork 一个写入的子进程、让父进程读取、并将出来的与进去的比较.

```c
/* producer_consumer.c: a two-process load test for /dev/myfirst. */
#include <sys/types.h>
#include <sys/wait.h>
#include <err.h>
#include <errno.h>
#include <fcntl.h>
#include <signal.h>
#include <stdint.h>
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <unistd.h>

#define DEVPATH         "/dev/myfirst"
#define TOTAL_BYTES     (1024 * 1024)
#define BLOCK           4096

static uint32_t
checksum(const char *p, size_t n)
{
        uint32_t s = 0;
        for (size_t i = 0; i < n; i++)
                s = s * 31u + (uint8_t)p[i];
        return (s);
}

static int
do_writer(void)
{
        int fd = open(DEVPATH, O_WRONLY);
        if (fd < 0)
                err(1, "writer: open");

        char *buf = malloc(BLOCK);
        if (buf == NULL)
                err(1, "writer: malloc");

        size_t written = 0;
        uint32_t sum = 0;
        while (written < TOTAL_BYTES) {
                size_t left = TOTAL_BYTES - written;
                size_t block = left < BLOCK ? left : BLOCK;
                for (size_t i = 0; i < block; i++)
                        buf[i] = (char)((written + i) & 0xff);
                sum += checksum(buf, block);

                const char *p = buf;
                size_t remain = block;
                while (remain > 0) {
                        ssize_t n = write(fd, p, remain);
                        if (n < 0) {
                                if (errno == EINTR)
                                        continue;
                                warn("writer: write");
                                close(fd);
                                return (1);
                        }
                        p += n;
                        remain -= n;
                }
                written += block;
        }

        printf("writer: %zu bytes, checksum 0x%08x\n", written, sum);
        close(fd);
        free(buf);
        return (0);
}

static int
do_reader(void)
{
        int fd = open(DEVPATH, O_RDONLY);
        if (fd < 0)
                err(1, "reader: open");

        char *buf = malloc(BLOCK);
        if (buf == NULL)
                err(1, "reader: malloc");

        size_t got = 0;
        uint32_t sum = 0;
        int mismatches = 0;
        while (got < TOTAL_BYTES) {
                ssize_t n = read(fd, buf, BLOCK);
                if (n < 0) {
                        if (errno == EINTR)
                                continue;
                        warn("reader: read");
                        break;
                }
                if (n == 0) {
                        /* Only reached if driver signals EOF. */
                        printf("reader: EOF at %zu\n", got);
                        break;
                }
                for (ssize_t i = 0; i < n; i++) {
                        if ((uint8_t)buf[i] != (uint8_t)((got + i) & 0xff))
                                mismatches++;
                }
                sum += checksum(buf, n);
                got += n;
        }

        printf("reader: %zu bytes, checksum 0x%08x, mismatches %d\n",
            got, sum, mismatches);
        close(fd);
        free(buf);
        return (mismatches == 0 ? 0 : 2);
}

int
main(void)
{
        pid_t pid = fork();
        if (pid < 0)
                err(1, "fork");
        if (pid == 0) {
                /* child: writer */
                _exit(do_writer());
        }
        /* parent: reader */
        int rc = do_reader();
        int status;
        waitpid(pid, &status, 0);
        int wexit = WIFEXITED(status) ? WEXITSTATUS(status) : -1;
        printf("exit: reader=%d writer=%d\n", rc, wexit);
        return (rc || wexit);
}
```

测试对第3或第4阶段效果最好. 对阶段 2（无阻塞），写入者将收到短写入，读取者偶尔会看到零字节读取，传输的总字节数可能少于 `TOTAL_BYTES`. 对阶段 3，双方在正确的时间阻塞和解除阻塞，测试运行到完成，两个校验和匹配.

成功的运行如下所示:

```sh
$ ./producer_consumer
writer: 1048576 bytes, checksum 0x12345678
reader: 1048576 bytes, checksum 0x12345678, mismatches 0
exit: reader=0 writer=0
```

不匹配是致命的. 如果写入者的校验和与读取者的匹配但不匹配非零，这意味着字节在往返过程中位置漂移（可能是环绕 bug）. 如果校验和不同，字节丢失或重复（可能是加锁 bug）. 如果测试永远挂起，阻塞路径的条件从未变为真（可能缺少 `wakeup`）.

### 使用 dd(1) 进行容量测试

基本系统 `dd(1)` 是在不编写任何新代码的情况下通过驱动推送容量的最快方式. 一些模式特别有用.

**模式 1：仅写入者。** 在读取者跟上时向驱动推送大量数据.

```sh
$ dd if=/dev/myfirst of=/dev/null bs=4k &
$ dd if=/dev/zero of=/dev/myfirst bs=4k count=100000
```

这产生 400 MB 的流量通过驱动. 观察 `sysctl dev.myfirst.0.stats.bytes_written` 增长，并将其与 `bytes_read` 比较；差异大致是缓冲区填充级别.

**模式 2：速率限制。** 一些测试想以稳定速率而不是最大吞吐量压力测试驱动. 使用 `rate` 或 GNU `pv(1)` 工具（作为 `ports/sysutils/pv` 可用）来限制:

```sh
$ pv -L 10m < /dev/zero | dd of=/dev/myfirst bs=4k
```

这将写入速率限制在 10 MB/s. 较慢的速率让你在 `sysctl` 中观察缓冲区的填充级别，并看到当速率接近消费者的速率时阻塞路径启动.

**模式 3：完整块。** 如第4节所述，默认 `dd` 不会在短读取上循环. 使用 `iflag=fullblock` 使其这样做:

```sh
$ dd if=/dev/myfirst of=/tmp/out bs=4k count=100 iflag=fullblock
```

没有 `iflag=fullblock`，输出文件可能因短读取而比请求的 400 KB 短.

### 使用 hexdump(1) 验证内容

`hexdump(1)` 是验证驱动传递内容的正确工具. 如果你写入已知字节序列并想确认它完整返回，`hexdump` 会显示给你.

```sh
$ printf 'ABCDEFGH' > /dev/myfirst
$ hexdump -C /dev/myfirst
00000000  41 42 43 44 45 46 47 48                           |ABCDEFGH|
$
```

`hexdump -C` 输出是规范的"这里是字节及其 ASCII 解释"格式. 当驱动发出基于文本的工具无法显示的二进制数据时，它特别有用.

### 使用 truss(1) 查看系统调用流量

`truss(1)` 追踪进程进行的系统调用. 在 `truss` 下运行测试向你显示每次 `read(2)` 和 `write(2)` 确切返回了什么，包括部分传输和错误代码.

```sh
$ truss -t read,write -o /tmp/trace ./rw_myfirst_nb
$ head /tmp/trace
read(3,0x7fffffffeca0,128)                       ERR#35 'Resource temporarily unavailable'
write(3,"hello world\n",12)                      = 12 (0xc)
read(3,0x7fffffffeca0,128)                       = 12 (0xc)
...
```

ERR#35 是 `EAGAIN`. 看到它确认非阻塞路径正在启动. 在 `truss` 下运行 `producer_consumer` 非常清楚地显示短写入和短读取的模式；它是调试缓冲区大小问题的良好诊断.

相关工具是 `ktrace(1)` / `kdump(1)`，它产生更详细和解码的追踪，代价是更冗长. 两者对这个级别的工作都适用.

### 使用 sysctl(8) 实时观察状态

sysctl 树 `dev.myfirst.0.stats.*` 是驱动的实时状态. 在测试期间实时观察它告诉你很多关于驱动正在做什么.

```sh
$ while true; do
    clear
    sysctl dev.myfirst.0.stats | egrep 'cb_|bytes_'
    sleep 1
  done
```

在一个终端运行此命令，同时在另一个终端运行测试. 你将看到 `cb_used` 在写入者领先时上升，在读取者跟上时下降，并在某个稳态水平附近振荡. 字节计数器只增加. 停滞的测试显示为冻结的计数器.

### 使用 vmstat -m 观察内存

如果你怀疑泄漏（可能你在 `attach` 的错误路径中忘记了 `cbuf_destroy`），`vmstat -m` 会显示它:

```sh
$ vmstat -m | grep cbuf
         cbuf     1      4K        1  4096
```

After `kldunload`:

```sh
$ vmstat -m | grep cbuf
$
```

当驱动卸载时，标签应该完全消失. 如果计数非零，某处仍持有分配. 这是你想立即捕获的回归类型；它会随时间静默恶化.

### 构建测试套件

这是同时构建所有用户态测试程序的 Makefile. 将其放在 `examples/part-02/ch10-handling-io-efficiently/userland/`:

```make
# Makefile for Chapter 10 userland testers.

PROGS= rw_myfirst_v2 rw_myfirst_nb producer_consumer stress_rw cb_test

.PHONY: all
all: ${PROGS}

CFLAGS?= -O2 -Wall -Wextra -Wno-unused-parameter

rw_myfirst_v2: rw_myfirst_v2.c
	${CC} ${CFLAGS} -o $@ $<

rw_myfirst_nb: rw_myfirst_nb.c
	${CC} ${CFLAGS} -o $@ $<

producer_consumer: producer_consumer.c
	${CC} ${CFLAGS} -o $@ $<

stress_rw: stress_rw.c
	${CC} ${CFLAGS} -o $@ $<

cb_test: ../cbuf-userland/cbuf.c ../cbuf-userland/cb_test.c
	${CC} ${CFLAGS} -I../cbuf-userland -o $@ $^

.PHONY: clean
clean:
	rm -f ${PROGS}
```

运行 `make` 构建所有四个工具. `make cb_test` 只构建独立的 `cbuf` 测试. 保持两个用户态目录（`cbuf-userland/` 用于缓冲区，`userland/` 用于驱动测试器）分开；第一个是后续阶段的先决条件，单独构建它反映了我们在章节中介绍它们的顺序.

### 综合测试计划

有了工具，这是你可以对驱动每个阶段运行的测试计划. 每次通过在加载相应的 `myfirst.ko` 后运行.

**阶段 2（循环缓冲区，无阻塞）：**

1. `./rw_myfirst_v2 fill 4096; sysctl dev.myfirst.0.stats.cb_used` 应该报告 4096.
2. `./rw_myfirst_v2 fill 4097` 应该显示短写入（写了 4096 of 4097）.
3. `./rw_myfirst_v2 drain 2048; sysctl dev.myfirst.0.stats.cb_used` 应该报告 2048.
4. `./rw_myfirst_v2 fill 2048; sysctl dev.myfirst.0.stats.cb_used` 应该报告 4096，但 `cb_head` 应该非零（证明环绕工作）.
5. `dd if=/dev/myfirst of=/dev/null bs=4k`：应该排空 4096 字节然后返回零.
6. `producer_consumer` 且 `TOTAL_BYTES = 8192`：应该成功完成.

**阶段 3（阻塞和非阻塞支持）：**

1. `cat /dev/myfirst &` 应该阻塞.
2. `echo hi > /dev/myfirst` 应该在 `cat` 终端产生输出.
3. `kill -INT %1` 应该干净地解除 `cat` 阻塞.
4. `./rw_myfirst_nb` 应该打印上面的六行输出.
5. `producer_consumer` 且 `TOTAL_BYTES = 1048576`：应该完成且无不匹配和匹配的校验和.

**阶段 4（poll 支持，重构的辅助函数）：**

所有阶段 3 测试，加上：

1. `./rw_myfirst_nb` 第4步应该显示 `revents=0x41`（POLLIN|POLLRDNORM）.
2. 一个小程序打开一个只读非阻塞描述符、用超时 -1 的 `poll(POLLIN)` 注册它、并从同一进程在第二个描述符上调用 `write`：`poll` 应该快速返回并设置 `POLLIN`.
3. `dd if=/dev/zero of=/dev/myfirst bs=1m count=10 &` 配合 `dd if=/dev/myfirst of=/dev/null bs=4k`：应该无错误移动 10 MB，大约在较慢方所花的时间内.

这个计划绝不是详尽的. 章节后面的实验部分给你更深入的序列. 但这些是冒烟测试：在每次非平凡更改后运行它们，如果它们通过，你没有破坏任何基本的东西.

### 测试失败时的调试

当测试失败时，检查序列通常是:

1. **`dmesg | tail -100`**：检查内核警告、恐慌或你自己的 `device_printf` 输出. 如果内核抱怨锁定违规或 `witness` 警告，问题在这里可见，在你做任何其他事情之前.
2. **`sysctl dev.myfirst.0.stats`**：将当前值与应有的值比较. 如果 `cb_used` 非零但没有人持有打开的描述符，关闭路径出了问题.
3. **`truss -t read,write,poll -f`**：在 `truss` 下运行失败的测试器并查看系统调用返回. 虚假的 `EAGAIN`（或其缺失）立即显示.
4. **`ktrace`**：如果 `truss` 不够，`ktrace -di ./test; kdump -f ktrace.out` 给出包括信号在内的更深视图.
5. **向驱动添加 `device_printf`**：在每个处理程序的顶部和底部撒上一行追踪，然后重现测试. 这是后备，有时是查看用户侧工具未捕获的时刻驱动在做什么的唯一方式.

对最后一步要小心. 每个 `device_printf` 都通过内核的日志缓冲区，它本身是有限的循环缓冲区. 将 `device_printf` 放入每个字节都运行的 `cbuf_write` 函数将融化日志. 从每次 I/O 调用一行日志开始，仅在需要时增加.

### 第6节总结

你现在有一个可以练习驱动承诺的每个非平凡行为的测试套件. `rw_myfirst_v2` 覆盖大小读取和写入以及往返正确性. `rw_myfirst_nb` 覆盖非阻塞路径和 `poll(2)` 契约. `producer_consumer` 覆盖并发双方负载和内容验证. `dd`、`cat`、`hexdump`、`truss`、`sysctl` 和 `vmstat -m` 一起提供驱动内部状态的可观测性.

这些工具都不是新的或奇异的. 它们是标准的 FreeBSD 基本系统工具，以及你可以在一个下午输入的短代码. 组合足以在到达他人之手前捕获大多数驱动 bug. 下一节采用你刚刚完成测试的驱动，并为其代码形状准备第11章的并发工作.

## 第7节：重构并为并发做准备

驱动工作. 它缓冲、阻塞、正确报告 poll，第6节的用户态测试确认字节在现实负载下正确流动. 我们还没有做的是为第11章将做的工作塑造代码. 本节是桥梁：它识别当前源码中需要从真正并发角度关注的地方，将缓冲区访问重构为一组紧密的辅助函数，并最终使驱动尽可能诚实地对待自己的状态.

我们在这里不引入新的锁定原语. 第11章将详细探索该材料，包括单个互斥锁的替代方案（可睡眠锁、sx 锁、rwlock、无锁模式）、验证工具（`WITNESS`、`INVARIANTS`）以及中断上下文、睡眠和锁排序的规则. 我们在第7节做的是使代码的*形状*使得那些工具可以在时机到来时干净地应用.

### 识别潜在竞态条件

驱动代码中的"竞态条件"是代码正确性取决于两个线程执行顺序的任何地方，其中顺序不受驱动中任何东西强制. 阶段 4 驱动有正确的*机制*（互斥锁、睡眠通道、通过 `mtx_sleep` 的睡眠持互斥锁语义）且 I/O 处理程序尊重它. 但仍有一些地方值得仔细审计.

让我们遍历数据结构，对每个共享字段问，"谁读它、谁写它、什么保护访问？"

**`sc->cb`（循环缓冲区）。** 由 `myfirst_read` 读取、由 `myfirst_write` 写入、由 `myfirst_poll` 读取、由两个 sysctl 处理程序（`cb_used` 和 `cb_free`）读取、由 `myfirst_detach` 读取（通过 `cbuf_destroy` 隐式）. 在它被触及的任何地方都由 `sc->mtx` 保护. *看起来安全。*

**`sc->bytes_read`, `sc->bytes_written`。** 由两个 I/O 处理程序在 `sc->mtx` 下更新. 由 sysctl 通过 `SYSCTL_ADD_U64` 直接读取（没有处理程序插入）. sysctl 读取在大多数架构上是单个 64 位加载，这在某些 32 位平台上是撕裂读取风险，但在 amd64 和 arm64 上是原子的. *大部分安全；见下面的撕裂读取说明。*

**`sc->open_count`, `sc->active_fhs`.** Updated under `sc->mtx`. Read by sysctl directly. Same torn-read consideration.

**`sc->is_attached`.** Read by every handler at entry, set by attach (without lock, before `make_dev`), cleared by detach (under lock). The unlocked write at attach time is safe because no one else can see the device yet. The locked clear at detach time is correctly ordered with the wakeup. *看起来安全。*

**`sc->cdev`, `sc->cdev_alias`.** Set by attach, cleared by detach. Once attach is done, these are stable for the lifetime of the device. The handlers reach the softc through `dev->si_drv1` (set during attach) and never dereference these directly during I/O. *Safe by construction.*

**`sc->rsel`, `sc->wsel`.** The `selinfo` machinery is internally locked (it uses the kernel's `selspinlock` and per-mutex `knlist` if you initialise one). For pure `select(2)`/`poll(2)` use, the `selrecord` and `selwakeup` calls handle their own concurrency. *Safe.*

**`sc->open_count` and friends, again.** The torn-read note above is worth being explicit about. On 32-bit platforms (i386, armv7), a 64-bit field can be split across two memory operations, and a concurrent write can yield a read that contains the high half of one value and the low half of another (a "torn read"). The chapter is targeting amd64 where this is not an issue, but it is the kind of thing a real driver should think about. The fix, if needed, is to add a sysctl handler (like the `cb_used` one) that takes the mutex around the load.

上面的审计给出了健康证明. 更大的重构机会不是竞态条件而是*代码形状*：缓冲区逻辑与 I/O 逻辑混合的地方、辅助函数可以澄清意图的地方、以及第11章可以在不触及 I/O 处理程序的情况下引入新锁类别的地方.

### 重构：将缓冲区访问拉入辅助函数

阶段 3 / 阶段 4 处理程序包含大量内联的锁定和记账. 让我们将其提取为一小组辅助函数. 目标是双重的：I/O 处理程序变得明显正确，第11章可以在不触及 `myfirst_read` 或 `myfirst_write` 的情况下将不同的锁定策略替换到辅助函数中.

定义以下辅助函数，都在 `myfirst.c` 中（或者如果你想要更清晰的分离，可以在新文件 `myfirst_buf.c` 中）:

```c
/* Read up to "n" bytes from the cbuf into "dst".  Returns count moved. */
static size_t
myfirst_buf_read(struct myfirst_softc *sc, void *dst, size_t n)
{
        size_t got;

        mtx_assert(&sc->mtx, MA_OWNED);
        got = cbuf_read(&sc->cb, dst, n);
        sc->bytes_read += got;
        return (got);
}

/* Write up to "n" bytes from "src" into the cbuf.  Returns count moved. */
static size_t
myfirst_buf_write(struct myfirst_softc *sc, const void *src, size_t n)
{
        size_t put;

        mtx_assert(&sc->mtx, MA_OWNED);
        put = cbuf_write(&sc->cb, src, n);
        sc->bytes_written += put;
        return (put);
}

/* Wait, with PCATCH, until the cbuf is non-empty or the device tears down. */
static int
myfirst_wait_data(struct myfirst_softc *sc, int ioflag, ssize_t nbefore,
    struct uio *uio)
{
        int error;

        mtx_assert(&sc->mtx, MA_OWNED);
        while (cbuf_used(&sc->cb) == 0) {
                if (uio->uio_resid != nbefore)
                        return (-1);            /* signal caller to break */
                if (ioflag & IO_NDELAY)
                        return (EAGAIN);
                error = mtx_sleep(&sc->cb, &sc->mtx, PCATCH, "myfrd", 0);
                if (error != 0)
                        return (error);
                if (!sc->is_attached)
                        return (ENXIO);
        }
        return (0);
}

/* Wait, with PCATCH, until the cbuf has free space or the device tears down. */
static int
myfirst_wait_room(struct myfirst_softc *sc, int ioflag, ssize_t nbefore,
    struct uio *uio)
{
        int error;

        mtx_assert(&sc->mtx, MA_OWNED);
        while (cbuf_free(&sc->cb) == 0) {
                if (uio->uio_resid != nbefore)
                        return (-1);            /* signal caller to break */
                if (ioflag & IO_NDELAY)
                        return (EAGAIN);
                error = mtx_sleep(&sc->cb, &sc->mtx, PCATCH, "myfwr", 0);
                if (error != 0)
                        return (error);
                if (!sc->is_attached)
                        return (ENXIO);
        }
        return (0);
}
```

`mtx_assert(&sc->mtx, MA_OWNED)` 调用是一个微小但有价值的安全网. 如果未来的调用者在调用这些辅助函数之一前忘记获取锁，断言触发（在 `WITNESS` 内核中）. 一旦你信任辅助函数，你可以停止在调用点考虑锁.

四个辅助函数一起覆盖 I/O 处理程序从缓冲区抽象需要的一切：读取字节、写入字节、等待数据、等待空间. 每个辅助函数通过引用获取互斥锁并断言它被持有. 它们都不锁定或解锁.

有了定义的辅助函数，I/O 处理程序大幅缩小:

```c
static int
myfirst_read(struct cdev *dev, struct uio *uio, int ioflag)
{
        struct myfirst_softc *sc = dev->si_drv1;
        struct myfirst_fh *fh;
        char bounce[MYFIRST_BOUNCE];
        size_t take, got;
        ssize_t nbefore;
        int error;

        error = devfs_get_cdevpriv((void **)&fh);
        if (error != 0)
                return (error);
        if (sc == NULL || !sc->is_attached)
                return (ENXIO);

        nbefore = uio->uio_resid;
        while (uio->uio_resid > 0) {
                mtx_lock(&sc->mtx);
                error = myfirst_wait_data(sc, ioflag, nbefore, uio);
                if (error != 0) {
                        mtx_unlock(&sc->mtx);
                        return (error == -1 ? 0 : error);
                }
                take = MIN((size_t)uio->uio_resid, sizeof(bounce));
                got = myfirst_buf_read(sc, bounce, take);
                fh->reads += got;
                mtx_unlock(&sc->mtx);

                wakeup(&sc->cb);
                selwakeup(&sc->wsel);

                error = uiomove(bounce, got, uio);
                if (error != 0)
                        return (error);
        }
        return (0);
}

static int
myfirst_write(struct cdev *dev, struct uio *uio, int ioflag)
{
        struct myfirst_softc *sc = dev->si_drv1;
        struct myfirst_fh *fh;
        char bounce[MYFIRST_BOUNCE];
        size_t want, put, room;
        ssize_t nbefore;
        int error;

        error = devfs_get_cdevpriv((void **)&fh);
        if (error != 0)
                return (error);
        if (sc == NULL || !sc->is_attached)
                return (ENXIO);

        nbefore = uio->uio_resid;
        while (uio->uio_resid > 0) {
                mtx_lock(&sc->mtx);
                error = myfirst_wait_room(sc, ioflag, nbefore, uio);
                if (error != 0) {
                        mtx_unlock(&sc->mtx);
                        return (error == -1 ? 0 : error);
                }
                room = cbuf_free(&sc->cb);
                mtx_unlock(&sc->mtx);

                want = MIN((size_t)uio->uio_resid, sizeof(bounce));
                want = MIN(want, room);
                error = uiomove(bounce, want, uio);
                if (error != 0)
                        return (error);

                mtx_lock(&sc->mtx);
                put = myfirst_buf_write(sc, bounce, want);
                fh->writes += put;
                mtx_unlock(&sc->mtx);

                wakeup(&sc->cb);
                selwakeup(&sc->rsel);
        }
        return (0);
}
```

每个 I/O 处理程序现在以相同的顺序做三件事：获取锁、向辅助函数询问状态、释放锁、做复制、再次获取锁、更新状态、释放、唤醒. 模式足够清晰，未来的读者可以一目了然地验证锁定纪律.

等待辅助函数返回的错误代码 `-1` 是一个小约定："没有错误要报告，但循环应该中断，调用者应该返回 0" 使用 `-1`（不是有效的 errno）使约定明显，而不添加第三个输出参数. 它对驱动是本地的，永远不会转义到用户空间.

### 文档化锁定策略

这种大小的驱动受益于文件顶部附近解释锁定纪律的一小段注释. 注释是为下一个阅读代码的人准备的，也是三个月后的你. 在 `struct myfirst_softc` 声明附近添加这个:

```c
/*
 * Locking strategy.
 *
 * sc->mtx protects:
 *   - sc->cb (the circular buffer's internal state)
 *   - sc->bytes_read, sc->bytes_written
 *   - sc->open_count, sc->active_fhs
 *   - sc->is_attached
 *
 * Locking discipline:
 *   - The mutex is acquired with mtx_lock and released with mtx_unlock.
 *   - mtx_sleep(&sc->cb, &sc->mtx, PCATCH, ...) is used to block while
 *     waiting on buffer state.  wakeup(&sc->cb) is the matching call.
 *   - The mutex is NEVER held across uiomove(9), copyin(9), or copyout(9),
 *     all of which may sleep.
 *   - The mutex is held when calling cbuf_*() helpers; the cbuf module is
 *     intentionally lock-free by itself and relies on the caller for safety.
 *   - selwakeup(9) and wakeup(9) are called with the mutex DROPPED, after
 *     the state change that warrants the wake.
 */
```

That comment is enough for Chapter 11 to either follow the same convention or to deliberately change it. A driver that explains its own rules makes future maintenance easier; a driver that does not explain its rules leaves every future reader to infer them from the source, which is slow and error-prone.

### Splitting `cbuf` Out of `myfirst.c`

In Stage 2 and Stage 3, the `cbuf` source lived alongside `myfirst.c` in the same module directory but in its own `.c` file. The Makefile is updated to compile both:

```make
KMOD=    myfirst
SRCS=    myfirst.c cbuf.c
SRCS+=   device_if.h bus_if.h

.include <bsd.kmod.mk>
```

两个小细节值得注意。

The first is that `cbuf.c` declares its own `MALLOC_DEFINE`. Each `MALLOC_DEFINE` for the same tag in the same module would be a duplicate definition; we therefore put the declaration in exactly one source file (`cbuf.c`) and an `extern` declaration in `cbuf.h` if needed. In our setup, the tag is local to `cbuf.c` and no external use is needed.

The second is that `cbuf.c` does not need any of the `myfirst` headers. It is a self-contained library that the driver happens to use. If you ever wanted to share `cbuf` with a second driver, you could pull it out into its own KLD or into `/usr/src/sys/sys/cbuf.h` and `/usr/src/sys/kern/subr_cbuf.c` (a hypothetical placement). The discipline of keeping `cbuf` self-contained makes that possible.

### Naming Conventions

A small but useful pattern: name your buffer-related fields and functions consistently. We have used `sc->cb` for the buffer, `cbuf_*` for buffer functions, `myfirst_buf_*` for the driver's wrappers. The pattern lets a reader scan the code and instantly know whether a function is touching the raw buffer (`cbuf_*`) or going through the locked driver wrappers (`myfirst_buf_*`).

Avoid mixing styles. Calling the buffer `sc->ring` in some places and `sc->cb` in others, or `cbuf_get` and `cbuf_read`, makes the code harder to skim. Pick one set of names and use them throughout.

### Defending Against Buffer-Size Surprises

The `MYFIRST_BUFSIZE` macro determines the capacity of the ring. Right now it is hard-coded to 4096. There is nothing wrong with that, but a `sysctl` knob (read-only) that exposes the value, plus a `module_param`-style override at module load time, would make the driver more usable in tests without needing to recompile.

这是使用 `TUNABLE_INT` 的加载时覆盖模式：

```c
static int myfirst_bufsize = MYFIRST_BUFSIZE;
TUNABLE_INT("hw.myfirst.bufsize", &myfirst_bufsize);
SYSCTL_INT(_hw_myfirst, OID_AUTO, bufsize, CTLFLAG_RDTUN,
    &myfirst_bufsize, 0, "Default buffer size for new myfirst attaches");
```

`TUNABLE_INT` reads the value from the kernel environment at boot or `kldload` time. A user can set it from the loader prompt (`set hw.myfirst.bufsize=8192`) or by running `kenv hw.myfirst.bufsize=8192` before `kldload`. The `CTLFLAG_RDTUN` flag indicates "read-only at runtime, but tunable at load time." After load, `sysctl hw.myfirst.bufsize` shows the chosen value.

然后在 `myfirst_attach` 中，在 `cbuf_init` 调用中使用 `myfirst_bufsize` 而不是 `MYFIRST_BUFSIZE`。 The change is small but useful: now you can experiment with different buffer sizes without rebuilding the module.

### Goals for the Next Milestone

第11章带驱动去的地方：

- The single mutex you have today protects everything. Chapter 11 will discuss whether a single lock is the right design under heavy contention, whether sleepable locks (`sx_*`) would be more appropriate, and how to reason about lock ordering when multiple subsystems get involved.
- The blocking path uses `mtx_sleep`, which is the right primitive for this kind of work. Chapter 11 will introduce `cv_wait(9)` (condition variables) as a more structured alternative for some patterns, and discuss when each is preferable.
- The wake-up strategy uses `wakeup(9)` (wake everyone). Chapter 11 will discuss `wakeup_one(9)` and the thundering-herd problem, and when each is appropriate.
- The cbuf is intentionally not thread-safe by itself. Chapter 11 will revisit this decision and discuss the tradeoffs of building locking *into* the data structure versus leaving it to the caller.
- The detach path's "wait for descriptors to close" rule is conservative. Chapter 11 will discuss alternative strategies (forced revocation, reference counting at the cdev level, the `destroy_dev_drain(9)` mechanism) for drivers that need to detach despite open descriptors.

You do not need to know any of this material yet. The point is that the current code's *shape* is what makes those topics approachable in Chapter 11. You can swap the mutex for an `sx` lock without touching the helpers' signatures. You can swap `wakeup` for `wakeup_one` with one-line changes. You can introduce a per-reader sleep channel without restructuring the I/O handlers. The refactor pays off as soon as you start asking the next chapter's questions.

### A Reading Order for the Next Chapter

When you start Chapter 11, three files in `/usr/src/sys` will repay careful reading.

`/usr/src/sys/kern/subr_sleepqueue.c` is where `mtx_sleep`, `tsleep`, and `wakeup` are implemented. Read it once for context. The implementation is more elaborate than the man pages suggest, but the core of it (chan-keyed sleep queues, atomic dequeue on wake) is straightforward.

`/usr/src/sys/sys/sx.h` and `/usr/src/sys/kern/kern_sx.c` together explain the sleepable-shared-exclusive lock. We mentioned `sx` above as an alternative to `mtx`; reading the actual implementation is the best way to understand the tradeoffs.

`/usr/src/sys/sys/condvar.h` and `/usr/src/sys/kern/kern_condvar.c` document the `cv_wait` family of condition-variable primitives. Like `mtx_sleep`, they build on the kernel's sleep-queue machinery in `subr_sleepqueue.c`, but they expose a distinct structured API where each wait point has its own named `struct cv` instead of an arbitrary address as the channel. Chapter 11 will explain when to prefer each, and why a dedicated `struct cv` is often the cleaner choice for a well-defined wait condition.

These are not required reading; they are the next step on a long path you are clearly already on.

### 第7节总结

The driver is now in the shape Chapter 11 wants. The buffer abstraction is in its own file, exercised in userland, and called from the driver through a small set of locked wrappers. The locking strategy is documented in a comment that names exactly what the mutex protects and what the rules are. The blocking path is correct, the non-blocking path is correct, the poll path is correct, and the detach path correctly waits for and wakes any sleepers.

Most of what you do in Chapter 11 will be additive to this base, not a rewrite of it. The patterns we have built (lock around state changes, sleep with the mutex as interlock, wake on every transition) are the same patterns the rest of the kernel uses. The vocabulary is the same, the primitives are the same, the discipline is the same. You are close to being able to read most character-device drivers in the tree without help.

Before we move on to the chapter's supplementary topics and the labs, take a moment to look at your own source. The Stage 4 driver should be roughly 500 lines of code (`myfirst.c`) plus about 110 lines of `cbuf.c` and 20 lines of `cbuf.h`. The total is small, the layering is clean, and almost every line is doing something specific. That density is what well-shaped driver code looks like.

## Section 8: Three Supplementary Topics

This section covers three topics that often appear alongside buffered I/O in the real world. Each one is large enough to fill an entire chapter on its own; we are not going to do that. Instead we are going to introduce each at the level a reader of this book needs in order to recognise the pattern, talk about it sensibly, and know where to look when the time comes to use it. The deeper treatments come later, in the chapters where each topic is the main subject.

The three topics are: `d_mmap(9)` for letting user space map a kernel buffer; zero-copy considerations and what they really mean; and the readahead and write-coalescing patterns used by high-throughput drivers.

### Topic 1: `d_mmap(9)` and Mapping a Kernel Buffer

`d_mmap(9)` is the character-device callback that the kernel invokes when a user-space program calls `mmap(2)` on `/dev/myfirst`. The handler's job is to translate a *file offset* into a *physical address* the VM system can map into the user's process. The signature is:

```c
typedef int d_mmap_t(struct cdev *dev, vm_ooffset_t offset, vm_paddr_t *paddr,
                     int nprot, vm_memattr_t *memattr);
```

对于用户想要映射的每个页大小块，内核调用 `d_mmap`，`offset` 设置为设备内的字节偏移. 处理程序计算相应页面的物理地址并通过 `*paddr` 存储. 它还可以通过 `*memattr` 调整内存属性（缓存、写合并等）. 返回非零错误代码告诉内核"此偏移无法映射"；返回 `0` 表示成功.

我们在这里介绍 `d_mmap` 的原因是它是缓冲 I/O 的轻量级兄弟. 有了 `read(2)` 和 `write(2)`，每次调用时每个字节都跨越信任边界复制. 有了 `mmap(2)` 后跟直接内存访问，字节对用户空间可见，无需任何显式复制. 用户空间程序读取或写入映射区域，就像它是普通内存一样，内核缓冲区是用户看到的相同字节.

此模式对于小而重要的一类设备很有吸引力. 帧缓冲区、DMA 映射的设备缓冲区、共享内存事件队列：每一个都受益于直接映射，这样用户代码可以操作字节而无需进入内核. /usr/src/sys/dev/mem/memdev.c` 中的经典示例（每个 `arch` 目录下有架构特定的 `memmmap` 函数）映射 `/dev/mem`，以便特权用户进程可以读取或写入物理内存页面.

对于像我们的学习驱动，目标更温和：让 `mmap(2)` 看到 `read(2)` 和 `write(2)` 使用的相同循环缓冲区. 用户可以然后读取缓冲区而不经过系统调用路径. 我们不会扩展驱动以支持通过 `mmap` 的写入（这需要仔细处理缓存一致性和与系统调用路径的并发更新），但只读映射是一个有用的功能.

#### 最小化的 d_mmap 实现

实现很短:

```c
static int
myfirst_mmap(struct cdev *dev, vm_ooffset_t offset, vm_paddr_t *paddr,
    int nprot, vm_memattr_t *memattr)
{
        struct myfirst_softc *sc = dev->si_drv1;

        if (sc == NULL || !sc->is_attached)
                return (ENXIO);
        if ((nprot & VM_PROT_WRITE) != 0)
                return (EACCES);
        if (offset >= sc->cb.cb_size)
                return (-1);
        *paddr = vtophys((char *)sc->cb.cb_data + (offset & ~PAGE_MASK));
        return (0);
}
```

将 `.d_mmap = myfirst_mmap,` 添加到 `cdevsw`. 处理程序按顺序做四件事.

首先，它检查设备是否仍然挂载. 在已拆除驱动上持有 `mmap` 的用户应该看到 `ENXIO`，而不是内核恐慌.

其次，它拒绝写入映射. 允许 `PROT_WRITE` 将让用户空间与读写处理程序并发修改缓冲区，这将与 cbuf 的不变量竞争. 只读映射足以满足我们的学习目的；想要可写映射的真正驱动必须做更多工作来保持 cbuf 一致.

第三，它限制偏移. 用户可以请求 `offset = 1 << 30`，远超缓冲区末尾；处理程序返回 `-1` 拒绝. （返回 `-1` 告诉内核"此偏移没有有效地址"；内核将此视为可映射区域的末尾。）

第四，它用 `vtophys(9)` 计算物理地址. `vtophys` 将内核虚拟地址转换为单个页面的相应物理地址. 缓冲区用 `malloc(9)` 分配，返回*虚拟*连续内存；对于适合一个页面的分配（我们在 4 KB 页机器上的 `MYFIRST_BUFSIZE` 4096 字节），这也是物理连续的，一次 `vtophys` 就够了. 对于更大的缓冲区，每个页面必须单独查找，因为 `malloc(9)` 不承诺跨页物理连续. 表达式 `(offset & ~PAGE_MASK)` 将调用者的偏移向下舍入到页面边界，以便在正确的页面基址上调用 `vtophys`；内核然后负责应用用户 `mmap` 调用中的页内偏移. 缓冲区可能跨越多个页面的生产驱动应该逐页遍历分配，或在真正需要物理连续时切换到 `contigmalloc(9)`.

#### 警告和限制

此最小化实现有几个重要警告适用.

`vtophys` 仅当分配的每个页面在物理内存中连续时，才对 `malloc(9)` 分配的内存工作. 小分配（一页以下）总是连续的. `malloc(9)` 进行的大分配是*虚拟*连续但不一定物理连续；处理程序需要计算每页物理地址而不是假设线性. 对于第10章的 4 KB 缓冲区（适合单个页面），简单形式工作.

对于真正大的缓冲区，正确的原语是 `contigmalloc(9)`（连续物理内存）或 `dev_pager_*` 函数提供自定义分页器. 两者都属于稍后我们正确讨论 VM 细节的章节.

映射是只读的. `PROT_WRITE` 请求将以 `EACCES` 失败. 允许写入需要一种在 cbuf 索引改变时失效用户映射的方法（对于循环缓冲区不切实际），或完全不同的设计，用户的写入直接驱动缓冲区. 两者都不适合学习章节.

最后，映射 cbuf *不*让用户空间以 `read` 的方式看到一致的字节流. 映射显示*原始*底层内存，包括活跃区域外的字节（可能陈旧或为零），并忽略 head/used 索引. 从映射读取的用户需要查阅 `sysctl dev.myfirst.0.stats.cb_used` 和 `cb_used` 来知道活跃区域从哪里开始和结束. 这是刻意的：`mmap` 是暴露原始内存的低级机制，任何结构化解释必须分层在上面.

#### 小型 mmap 测试器

映射缓冲区并遍历它的用户态程序如下：

```c
/* mmap_myfirst.c: map the myfirst buffer read-only and dump it. */
#include <sys/mman.h>
#include <fcntl.h>
#include <stdio.h>
#include <string.h>
#include <unistd.h>

#define DEVPATH "/dev/myfirst"
#define BUFSIZE 4096

int
main(void)
{
        int fd = open(DEVPATH, O_RDONLY);
        if (fd < 0) { perror("open"); return (1); }

        char *map = mmap(NULL, BUFSIZE, PROT_READ, MAP_SHARED, fd, 0);
        if (map == MAP_FAILED) { perror("mmap"); close(fd); return (1); }

        printf("first 64 bytes:\n");
        for (int i = 0; i < 64; i++)
                printf(" %02x", (unsigned char)map[i]);
        putchar('\n');

        munmap(map, BUFSIZE);
        close(fd);
        return (0);
}
```

写入一些字节后运行它：

```sh
$ printf 'ABCDEFGHIJKL' > /dev/myfirst
$ ./mmap_myfirst
first 64 bytes:
 41 42 43 44 45 46 47 48 49 4a 4b 4c 00 00 00 00 ...
```

前十二字节是 `A`、`B`、...、`L`，正是写入的内容. 剩余字节为零，因为 `cbuf_init` 将后备内存零填充，且我们尚未写入偏移 12 之后的内容. 这是基本机制.

#### 你实际上何时会使用 d_mmap

大多数伪设备不需要 `d_mmap`. 系统调用路径快速、简单且被充分理解，每页额外 `read(2)` 的成本对于低速率数据可以忽略不计. 当以下情况之一适用时使用 `d_mmap`：

- 数据以非常高速率（图形或高端 I/O 中每秒千兆字节）产生到缓冲区中，每字节的系统调用开销开始占主导.
- 用户空间想要查看或处理大缓冲区中的特定位置而不复制整个内容.
- 驱动代表硬件，其寄存器或 DMA 区域可作为内存寻址（例如，GPU 的命令 FIFO）.

对于我们的伪设备，`d_mmap` 主要是一个学习练习. Building it teaches you the call signature, the relationship to the VM system, and the `vtophys`/`contigmalloc` distinction. 真正的生产用途在你编写需要吞吐量的驱动时出现.

### 主题 2：零拷贝考虑

"零拷贝"是系统性能讨论中最过度使用的词之一. 严格来说，它意味着"操作期间没有数据在内存位置之间复制" 该定义太严格而不实用：即使从设备到内存的 DMA，技术上也是复制. 在实践中，"零拷贝"是"字节不作为 I/O 路径中显式复制指令的一部分经过 CPU 缓存"的简写

对于像 `myfirst` 这样的字符设备，问题是你是否可以避免读写处理程序中的 `uiomove(9)` 复制. 对于我们构建的模式，答案是"不，尝试这样做通常是错误的" 原因如下.

`uiomove(9)` 每次传输从内核到用户（或用户到内核）做一次复制. 这是每次 `read(2)` 或 `write(2)` 调用一组字节移动. CPU 将源拉入缓存、从缓存写入目标、然后继续其工作. 在现代硬件上，此复制很快：L1 缓存行是 64 字节，CPU 可以每秒流式传输数十千兆字节的内存复制，每字节成本在个位数纳秒内.

要消除该复制，你必须找到另一种方式使字节对用户空间可见. 两种主要机制是 `mmap(2)`（我们刚刚讨论过）和共享内存原语（`shm_open(3)`、带有 `MSG_PEEK` 的套接字、sendfile）. 它们都有自己的成本：页表更新、TLB 刷新、多 CPU 系统上的 IPI 流量、映射时无法将源内存用于其他任何用途. 对于中小传输，`uiomove` 比替代方案*更快*，因为替代方案的设置成本占主导.

零拷贝确实有回报的真实情况. 将传入数据包 DMA 到 mbuf 并将 mbuf 交给协议栈的网络驱动避免了否则会与 DMA 本身一样昂贵的复制. 使用 `bus_dmamap_load(9)` 从用户空间缓冲区（在 `vslock` 后）设置 DMA 传输的存储驱动避免了否则会占 I/O 成本主导地位的两次复制. 高吞吐量图形驱动可能将 GPU 命令缓冲区直接映射到渲染进程以避免每帧复制. 所有这些都是真正的胜利.

然而，对于数据不来自真实硬件的伪设备，收益是虚幻的. "保存的"复制只是重新排列字节存储位置；成本出现在其他地方（页表更新、用户触及直接从内核直写的页面时的缓存未命中、两个 CPU 触及相同共享页面时的竞争）. 阶段 4 驱动每个弹跳缓冲区大小的块做一次 `uiomove`；这大约是每 256 字节一次复制，完全在单核可以维持的吞吐量范围内.

如果你发现自己优化伪设备的复制，首先值得问两个问题.

第一个问题是复制是否实际上是瓶颈. 在 `dtrace` 或 `pmcstat` 下运行驱动并测量周期去向. 如果 `uiomove` 不在前三，优化它不会有可测量的差异. 此类代码中最常见的瓶颈是锁竞争（一个 CPU 等待另一个释放互斥锁）、系统调用开销（许多小系统调用而不是较少的大系统调用）、以及唤醒睡眠者的成本（每次 `wakeup` 都是睡眠队列遍历）. 所有这些都比复制本身提供更大的收益.

第二个问题是驱动的*用户*是否真的想要零拷贝语义. 调用 `read(2)` 的用户要求内核给他们字节的一份副本. 他们不是要求指向内核字节的指针. 切换到映射改变契约；用户必须知道映射、显式管理它、并理解缓存一致性规则. 这是用户必须选择加入的权衡，不是透明的改进.

正确的框架是：零拷贝是一种具有特定成本和特定收益的技术. 当收益明显超过成本时使用它，而不是在此之前. 对于大多数驱动，尤其是伪设备，带有 `uiomove` 的系统调用路径是正确的选择.

### 主题 3：预读和写合并

第三个主题是关于吞吐量. 当驱动支持稳定高速字节流时，两种模式变得重要：读取侧的**预读**和写入侧的**写合并**. 两者都是关于每次系统调用做更多工作，两者都减少 I/O 路径的每字节开销.

#### 预读

预读是获取比用户当前请求的更多数据的做法，假设他们接下来会请求它. 常规文件读取经常在 VFS 层触发预读：当内核注意到进程已读取几个顺序块时，它开始在后台读取下一个块，以便下一次 `read(2)` 发现它们已在内存中. 用户在后续读取中看到更低的延迟.

对于伪设备，VFS 层的预读不直接适用（没有底层文件）. 然而，*驱动*可以通过提前请求数据源产生来做自己的预读形式. 想象一个包装慢数据源（硬件传感器、远程服务）的驱动. 当用户读取时，驱动从源拉取数据. 用户再次读取；驱动拉取更多. 有了预读，驱动可能在用户第一次读取时从源拉取一个*块*的数据，将额外字节存储在 cbuf 中，并直接从 cbuf 服务后续读取而不返回源.

这正是 `myfirst` 驱动在精神上已经做的. cbuf *就是*预读缓冲区. 写入存入数据，读取消耗它，读取者不必等待写入者写入每个单独字节. 更广泛的教训是，驱动中有缓冲区在结构上是与预读相同的模式：它让消费者找到已准备好的数据.

当你针对真实源构建驱动时，预读逻辑通常存在于监视 `cbuf_used` 并在计数降至阈值以下时触发从源获取的内核线程或 callout 中. 阈值是*低水位线*；获取在计数达到*高水位线*时停止. cbuf 成为源的突发速率和消费者的突发速率之间的缓冲区，内核线程使其保持适当充满.

#### 写合并

写合并是镜像模式. 将数据沉入慢目的地（硬件寄存器、远程服务）的驱动可能将几个小写入合并为单个大写入，减少目的地的每次写入开销. 用户的 `write(2)` 调用将字节存入 cbuf；内核线程或 callout 从 cbuf 读取并以更大的块写入目的地.

合并当目的地有高每次操作开销时特别有用. 考虑一个与芯片通信的驱动，其命令结构期望每次写入有头部、载荷和尾部：一次 1024 字节的写入到芯片可能比一千次 1 字节的写入快二十倍，因为每次写入开销在小尺寸时占主导. 驱动通过在 cbuf 中收集字节并以更大的块刷新来合并.

*何时*刷新的决定是困难的部分. 两种常见策略存在：**阈值时刷新**（当 `cbuf_used` 超过高水位线时刷新）和**超时时刷新**（自第一个字节到达后固定延迟后刷新）. 大多数真正的驱动使用组合：当任一条件满足时刷新. `callout(9)`（内核的延迟执行原语）是调度超时的自然方式. 第13章详细涵盖 `callout`；目前，概念点是合并是每字节延迟（更差，因为字节在缓冲区中）和每次操作吞吐量（更好，因为目的地看到较少的较大写入）之间的刻意权衡.

#### 这些模式如何应用于 myfirst

`myfirst` 驱动不需要任一模式的显式，因为它没有真正的源或汇. cbuf 已经提供写入者和读取者之间的耦合，唯一的"刷新"是读取者调用 `read(2)` 时发生的自然刷新. 但了解模式有两个原因有用.

首先，当你在 `/usr/src/sys/dev/` 中读取驱动代码时，你会反复看到这些模式. 网络驱动通过队列合并 TX 写入. 音频驱动通过提前为消费者获取 DMA 块来做预读. 块设备驱动使用 BIO 层通过扇区相邻合并 I/O 请求. 识别模式让你在不丢失情节的情况下浏览一千行驱动代码.

其次，当你开始在第4部分及以后编写真正的硬件驱动时，你需要决定是否以及如何将这些模式应用于你的驱动. 第10章的工作给你*基础底座*（带有正确锁定和阻塞语义的循环缓冲区）. 添加预读意味着启动内核线程来填充它. 添加合并意味着在定时器或阈值上刷新它. 基础底座相同；策略不同.

### 第8节总结

这三个主题（`d_mmap`、零拷贝、预读/合并）是驱动开发中常见的后续对话. 它们都不是第10章本身的主题，但每个都建立在你刚刚建立的缓冲区抽象和 I/O 机制之上.

`d_mmap` 向缓冲区添加补充路径：除了 `read(2)` 和 `write(2)`，用户空间现在可以直接查看字节. 零拷贝是解释 `d_mmap` 在某些情况下重要而在其他情况下过度的框架. 预读和写合并是将缓冲驱动转变为高吞吐量驱动的模式.

本章接下来的部分回到你当前的驱动：巩固四个阶段的动手实验、延伸你理解的挑战练习、以及此材料最可能产生的 bug 的故障排除部分.

## 动手实验

下面的实验带你通过本章的四个阶段，之间有具体的检查点. 每个实验对应一个你可以用第6节测试套件验证的里程碑. 它们设计为按顺序进行；后续实验假设早期实验已完成.

一个一般说明：在每个实验会话开始时，做一个 `kldunload myfirst`（如果之前的模块仍加载）和新的 `kldload ./myfirst.ko`. 观察 `dmesg | tail` 的挂载消息. 如果挂载失败，其余实验将以令人困惑的方式失败；先修复挂载.

### 实验 1：独立的循环缓冲区

**目标：** 构建并验证用户态 `cbuf` 实现. 此实验完全在用户空间；没有内核模块涉及.

**步骤：**

1. 如果不存在，创建目录 `examples/part-02/ch10-handling-io-efficiently/cbuf-userland/`.
2. 完全按第2节所示输入 `cbuf.h` 和 `cbuf.c`. 抵制从书中浏览源码的诱惑；输入它迫使你注意到每一行.
3. 从第2节输入 `cb_test.c`.
4. 用 `cc -Wall -Wextra -o cb_test cbuf.c cb_test.c` 构建.
5. 运行 `./cb_test`。你应该看到三个"OK"行和最终的"all tests OK".

**检查点问题：**

- 当缓冲区已满时 `cbuf_write(&cb, src, n)` 返回什么？
- 当缓冲区已空时 `cbuf_read(&cb, dst, n)` 返回什么？
- 在 `cbuf_init(&cb, 4)` 和 `cbuf_write(&cb, "ABCDE", 5)` 后，`cbuf_used(&cb)` 是什么？`cb.cb_data`（位置 0..3）的内容是什么？

如果你无法从自己的代码回答这些，重读第2节并追踪源码.

**延伸目标：** 添加第四个测试 `test_alternation`，写入一个字节、读回、写入另一个字节、读回，如此进行 100 次迭代. 这捕获现有测试不捕获的 `cbuf_read` 中的差一错误.

### 实验 2：阶段 2 驱动（循环缓冲区集成）

**目标：** 将验证的 `cbuf` 移入内核并替换第9章的阶段 3 线性 FIFO.

**步骤：**

1. Create `examples/part-02/ch10-handling-io-efficiently/stage2-circular/`.
2. Copy `cbuf.h` from your userland directory into the new directory.
3. Type the kernel-side `cbuf.c` from Section 3 (this is the `MALLOC_DEFINE`-using version).
4. Copy `myfirst.c` from `examples/part-02/ch09-reading-and-writing/stage3-echo/` into the new directory.
5. Modify `myfirst.c` to use the cbuf abstraction. The changes are:
   - Add `#include "cbuf.h"` near the top.
   - Replace `char *buf; size_t buflen, bufhead, bufused;` with `struct cbuf cb;` in the softc.
   - Update `myfirst_attach` to call `cbuf_init(&sc->cb, MYFIRST_BUFSIZE)`. Update the failure path to call `cbuf_destroy`.
   - Update `myfirst_detach` to call `cbuf_destroy(&sc->cb)`.
   - Replace `myfirst_read` and `myfirst_write` with the loop-and-bounce versions from Section 3.
   - Update the sysctl handlers as in Section 3 (use the `myfirst_sysctl_cb_used` and `myfirst_sysctl_cb_free` helpers).
6. Update the `Makefile` to build both source files: `SRCS= myfirst.c cbuf.c device_if.h bus_if.h`.
7. Build with `make`. Fix any compilation errors.
8. Load with `kldload ./myfirst.ko` and verify with `dmesg | tail`.

**Verification:**

```sh
$ printf 'helloworld' > /dev/myfirst
$ sysctl dev.myfirst.0.stats.cb_used
dev.myfirst.0.stats.cb_used: 10
$ cat /dev/myfirst
helloworld
$ sysctl dev.myfirst.0.stats.cb_used
dev.myfirst.0.stats.cb_used: 0
```

**Stretch goal 1:** write enough bytes to wrap the buffer (write 3000 bytes, read 2000, write 2000 again). Verify that `cb_head` is non-zero in `sysctl` and that the data still comes back correctly.

**Stretch goal 2:** add a sysctl-controlled debug flag (`myfirst_debug`) and a `MYFIRST_DBG` macro (Section 3 shows the pattern). Use it to log every successful `cbuf_read` and `cbuf_write` in the I/O handlers. Set the flag with `sysctl dev.myfirst.debug=1` and watch `dmesg`.

### Lab 3: The Stage 3 Driver (Blocking and Non-Blocking)

**Goal:** Add blocking-on-empty, blocking-on-full, and `EAGAIN` for non-blocking callers.

**步骤：**

1. Create `examples/part-02/ch10-handling-io-efficiently/stage3-blocking/` and copy your Stage 2 source into it.
2. Modify `myfirst_read` to add the inner sleep loop (Section 5). The new shape includes the `nbefore = uio->uio_resid` snapshot, the `mtx_sleep` call, and the `wakeup(&sc->cb)` after a successful read.
3. Modify `myfirst_write` to add the symmetric sleep loop and the matching `wakeup(&sc->cb)`.
4. Update `myfirst_detach` to set `sc->is_attached = 0` *before* calling `wakeup(&sc->cb)`, all under the mutex.
5. Build, load, and verify.

**Verification:**

```sh
$ cat /dev/myfirst &
[1] 12345
$ ps -AxH -o pid,wchan,command | grep cat
12345 myfrd  cat /dev/myfirst
$ echo hi > /dev/myfirst
hi
[after the cat consumes "hi", it blocks again]
$ kill -INT %1
[1]    Interrupt: 2
```

**Verification of `EAGAIN`:**

```sh
$ ./rw_myfirst_nb       # from the userland directory
step 1: empty-read returned EAGAIN (expected)
step 2: poll(POLLIN, 0) = 0 revents=0x0
...
```

If step 1 still says `read returned 0`, your `IO_NDELAY` check in `myfirst_read` is missing or wrong.

**Stretch goal 1:** open two `cat` processes against `/dev/myfirst` simultaneously. Write 100 bytes from a third terminal. Both `cat`s should wake up; one will get the bytes (whichever wins the race for the lock), the other will block again. You can verify the assignment by tagging each `cat` with a different output stream: `cat /dev/myfirst > /tmp/a &` and `cat /dev/myfirst > /tmp/b &`, then `cmp /tmp/a /tmp/b` (one will be empty).

**Stretch goal 2:** time how long `cat /dev/myfirst` takes to wake up after a write, using `time(1)`. The wake-up latency should be in the low microseconds; if it is in milliseconds, something is buffering between the write and the wake (or your machine is heavily loaded).

### Lab 4: The Stage 4 Driver (Poll Support and Refactor)

**Goal:** Add `d_poll`, refactor buffer access into helpers, and document the locking strategy.

**步骤：**

1. Create `examples/part-02/ch10-handling-io-efficiently/stage4-poll-refactor/` and copy your Stage 3 source.
2. Add `struct selinfo rsel; struct selinfo wsel;` to the softc.
3. Implement `myfirst_poll` as in Section 5.
4. Add `selwakeup(&sc->wsel)` after the read's successful `cbuf_read`, and `selwakeup(&sc->rsel)` after the write's successful `cbuf_write`.
5. Add `seldrain(&sc->rsel); seldrain(&sc->wsel);` to detach.
6. Add `.d_poll = myfirst_poll,` to the `cdevsw`.
7. Refactor the I/O handlers to use the four helpers from Section 7 (`myfirst_buf_read`, `myfirst_buf_write`, `myfirst_wait_data`, `myfirst_wait_room`).
8. Add the locking-strategy comment from Section 7.
9. Build, load, and verify.

**Verification:**

```sh
$ ./rw_myfirst_nb
step 1: empty-read returned EAGAIN (expected)
step 2: poll(POLLIN, 0) = 0 revents=0x0
step 3: wrote 12 bytes
step 4: poll(POLLIN, 0) = 1 revents=0x41
step 5: read 12 bytes: hello world
```

The key change from Lab 3 is that step 4 should now return `1` (not `0`), with `revents=0x41` (POLLIN | POLLRDNORM). If it still returns 0, your `selwakeup` call is missing from the write path or the `myfirst_poll` handler is wrong.

**Stretch goal 1:** run `producer_consumer` with `TOTAL_BYTES = 8 * 1024 * 1024` (8 MB) and verify that the test completes with no mismatches. The producer is generating bytes faster than the consumer is reading them, so the buffer should fill up and trigger the blocking path repeatedly. Watch `sysctl dev.myfirst.0.stats.cb_used` in another terminal; it should oscillate.

**Stretch goal 2:** run two `producer_consumer`s in parallel against the same device. The two writers will compete for buffer space; the two readers will compete for bytes. Each pair should still see consistent checksums, but the *interleaving* of bytes will be unpredictable. This shows that the driver is single-stream per device, not per descriptor; if you need per-descriptor streams, that is a different driver design.

### Lab 5: Memory Mapping

**Goal:** Add `d_mmap` so user space can map the cbuf read-only.

**步骤：**

1. Create `examples/part-02/ch10-handling-io-efficiently/stage5-mmap/` and copy your Stage 4 source.
2. Add `myfirst_mmap` from Section 8 to the source.
3. 将 `.d_mmap = myfirst_mmap,` 添加到 `cdevsw`.
4. Build, load, and verify.

**Verification:**

```sh
$ printf 'ABCDEFGHIJKL' > /dev/myfirst
$ ./mmap_myfirst       # from the userland directory
first 64 bytes:
 41 42 43 44 45 46 47 48 49 4a 4b 4c 00 00 00 ...
```

The first twelve bytes are the bytes you wrote.

**Stretch goal 1:** write a small program that maps the buffer and reads bytes from `offset = sc->cb_size - 32` (i.e. the last 32 bytes). Verify that the program does not crash. Then write enough bytes to push the buffer head into the wrap region and read from the same offset. The contents will be different, because the *raw* bytes in memory are not the same as the *live* bytes from the cbuf's perspective.

**Stretch goal 2:** try to map the buffer with `PROT_WRITE`. Your program should see `mmap` fail with `EACCES`, because the driver refuses writable mappings.

### Lab 6: Stress and Long-Running Tests

**Goal:** Run the driver under sustained load for at least an hour without errors.

**步骤：**

1. Set up four parallel test processes:
   - `dd if=/dev/zero of=/dev/myfirst bs=4k 2>/dev/null &`
   - `dd if=/dev/myfirst of=/dev/null bs=4k 2>/dev/null &`
   - `./producer_consumer`
   - A loop that polls `sysctl dev.myfirst.0.stats` every 5 seconds.
2. Let the test run for at least an hour.
3. Check `dmesg` for any kernel warnings, panics, or `WITNESS` complaints. Check `vmstat -m | grep cbuf` to confirm no leak. Verify that `producer_consumer` reports zero mismatches.

**Verification:** No kernel warnings. No memory growth in `vmstat`. `producer_consumer` returns 0.

**Stretch goal:** run the same test under a `WITNESS`-enabled debug kernel. The kernel will be slower but will catch any locking-discipline violations. If your driver is correct, no warnings should appear.

### Lab 7: Deliberate Failures

**Goal:** Break the driver in three specific ways and observe what happens. This lab teaches you to recognise the failure modes you most want to avoid.

**Steps for failure 1: hold the lock across `uiomove`.**

1. Edit your Stage 4 driver. In `myfirst_read`, comment out the `mtx_unlock(&sc->mtx)` that comes before `uiomove(bounce, got, uio)`.
2. Add a matching `mtx_unlock` after the `uiomove` so the code still compiles.
3. Build and load on a `WITNESS`-enabled kernel.
4. Run a single `cat /dev/myfirst` and write some bytes from another terminal.

**What you should observe:** A `WITNESS` warning in `dmesg` complaining about "sleeping with mutex held". The system may continue running but the warning is the bug.

**Cleanup:** restore the original code.

**Steps for failure 2: forget the `wakeup` after a write.**

1. In `myfirst_write`, comment out `wakeup(&sc->cb)`.
2. Build and load.
3. Run `cat /dev/myfirst &` and `echo hi > /dev/myfirst`.

**What you should observe:** The `cat` does not wake up. It will sit in `myfrd` state forever (or until you interrupt it with Ctrl-C).

**Cleanup:** restore the wakeup. Verify that `cat` now wakes up immediately.

**Steps for failure 3: missing `PCATCH`.**

1. In `myfirst_wait_data`, change `PCATCH` to `0` in the `mtx_sleep` call.
2. Build and load.
3. Run `cat /dev/myfirst &` and try `kill -INT %1`.

**What you should observe:** The `cat` does not respond to Ctrl-C until you write some bytes to wake it. With `PCATCH`, the signal would interrupt the sleep immediately.

**Cleanup:** restore `PCATCH`. Verify that `kill -INT` works as expected.

These three failures are the most common driver bugs in this chapter's territory. Doing them deliberately, once, is the best way to recognise them when they happen accidentally.

### Lab 8: Reading Real FreeBSD Drivers

**Goal:** Read three character-device drivers in `/usr/src/sys/dev/` and identify how each implements its buffer, sleep, and poll patterns.

**步骤：**

1. Read `/usr/src/sys/dev/evdev/cdev.c`. Identify:
   - Where the per-client ring buffer is allocated.
   - Where the read handler blocks (look for `mtx_sleep`).
   - How `EVDEV_CLIENT_EMPTYQ` is implemented.
   - How `kqueue` is set up alongside `select/poll` (we have not done `kqueue` yet; just notice the calls to `knlist_*`).
2. Read `/usr/src/sys/dev/random/randomdev.c`. Identify:
   - Where `randomdev_poll` is defined.
   - How it handles a not-yet-seeded random device.
3. Read `/usr/src/sys/dev/null/null.c`. Identify:
   - How `zero_read` loops over `uio_resid`.
   - Why there is no buffer, no sleep, and no poll handler.

**检查点问题：**

- Why does `evdev`'s read handler use `mtx_sleep` while `null`'s does not?
- What would `randomdev`'s poll handler return if called while the device is unseeded?
- How does `evdev` detect that a client has been disconnected (revoked)?

The point of this lab is not to memorise these drivers. It is to confirm that the patterns you have built in `myfirst` are the same patterns the kernel uses elsewhere. By the end of the lab you should feel that the rest of `dev/` is largely *legible* now, where it might have looked impenetrable two chapters ago.

## 挑战练习

The labs above ensure you have a working driver and a working test kit. The challenges below are stretch exercises. Each one extends the chapter's material in a useful direction, and each one rewards careful work. Take your time; some of them are more involved than they look.

### Challenge 1: Add a Tunable Buffer Size

The `MYFIRST_BUFSIZE` macro hard-codes the buffer at 4 KB. Make it configurable.

- Add a `TUNABLE_INT("hw.myfirst.bufsize", &myfirst_bufsize)` and a matching `SYSCTL_INT(_hw_myfirst, OID_AUTO, bufsize, ...)` so the user can set the buffer size at module load time.
- Use the value in `myfirst_attach` to size the cbuf.
- Validate the value (reject zero, reject sizes larger than 1 MB, fall back to a sensible default if the input is bad).
- Verify with `kenv hw.myfirst.bufsize=8192; kldload ./myfirst.ko; sysctl dev.myfirst.0.stats.cb_size`.

**Stretch:** make the buffer size *runtime-tunable* via `sysctl`. This is harder than the load-time tunable because it requires safely reallocating the cbuf while the device may be in use; you will need to drain or copy existing bytes, take and release the lock at the right moments, and decide what to do with sleeping callers. (Hint: it may be easier to require that all descriptors be closed before allowing a runtime resize.)

### Challenge 2: Implement Overwrite Semantics as an Optional Mode

Add an `ioctl(2)` (or, simpler for now, a `sysctl`) that switches the buffer between "block on full" mode (the default) and "overwrite oldest on full" mode. In overwrite mode, `myfirst_write` always succeeds: when `cbuf_free` is zero, the driver advances `cb_head` to make room and then writes the new bytes.

- Add a `cbuf_overwrite` function alongside `cbuf_write` that implements the overwrite semantics. Do not modify `cbuf_write`; the two should be siblings.
- Add a sysctl `dev.myfirst.0.overwrite_mode` (read-write integer, 0 or 1).
- In `myfirst_write`, dispatch to `cbuf_overwrite` if the flag is set.
- Test with a small writer that produces bytes faster than the reader consumes; in overwrite mode, the reader should see the most recent bytes only, while in normal mode the writer blocks.

**Stretch:** add a counter for the number of bytes overwritten (lost). Expose it as a sysctl so the user can see how much data has been dropped.

### Challenge 3: Per-Reader Position

The current driver has one shared read position (`cb_head`). When two readers consume bytes, each `read(2)` call drains some bytes from the buffer; the two readers split the stream between them. Some drivers want the opposite: each reader should see *every* byte, so two readers each get the full stream independently.

This is a substantial refactor:

- Maintain a per-descriptor read position in `myfirst_fh`.
- Track the global "earliest live byte" across all descriptors. The cbuf's effective `head` becomes `min(per_fh_head)`.
- `myfirst_read` advances only the per-descriptor position; `cbuf_read` is replaced by a per-fh equivalent.
- A new descriptor opened mid-stream sees only bytes written after its open.
- When the buffer is "full" depends on the slowest descriptor; you need backpressure logic that accounts for laggards.

This challenge is harder than it sounds; it is essentially building a multicast pipe. Try it only if you have time to think through the locking carefully.

### Challenge 4: Implement `d_kqfilter`

Add `kqueue(2)` support alongside the `d_poll` you already have.

- Implement a `myfirst_kqfilter` function dispatched from `cdevsw->d_kqfilter`.
- For `EVFILT_READ`, register a filter that becomes ready when `cbuf_used > 0`.
- For `EVFILT_WRITE`, register a filter that becomes ready when `cbuf_free > 0`.
- Use `knlist_add(9)` and `knlist_remove(9)` to manage the per-filter list.
- Trigger `KNOTE_LOCKED(...)` from the I/O handlers when the buffer transitions.
- Test with a small `kqueue(2)` user program that opens the device, registers `EVFILT_READ`, calls `kevent(2)`, and reports when the descriptor becomes readable.

This challenge is the natural extension of Stage 4. It also previews the `kqueue` material that Chapter 11 will discuss in more depth alongside concurrency.

### Challenge 5: Per-CPU Counters

The `bytes_read` and `bytes_written` counters are updated under the mutex. Under heavy multi-CPU load, this can become a contention point. FreeBSD's `counter(9)` API provides per-CPU counters that can be incremented without a lock and summed for read access.

- Replace `sc->bytes_read` and `sc->bytes_written` with `counter_u64_t` instances.
- Allocate them with `counter_u64_alloc(M_WAITOK)` in attach; free them with `counter_u64_free` in detach.
- Use `counter_u64_add(counter, n)` to increment.
- Use `counter_u64_fetch(counter)` (with a sysctl handler) to read.

**Stretch:** measure the difference. Run `producer_consumer` against the old and new versions and compare wall-clock time. With a small test the difference will be invisible; with a heavily threaded test (multiple producers and consumers) the per-CPU version should be measurably faster.

### Challenge 6: A Hardware-Style Interrupt Simulator

Real driver buffers are usually filled by an interrupt handler, not by a `write(2)` syscall. Simulate this:

- Use `callout(9)` (Chapter 13 covers it; you can read ahead) to run a callback every 100 ms.
- The callback writes a small piece of data into the cbuf (for example, the current time as a string).
- The user reads from `/dev/myfirst` and sees a stream of timestamped lines.

This challenge previews Chapter 13's deferred-execution material and shows how the same buffer abstraction supports either a syscall-driven producer or a kernel-thread-driven producer.

### Challenge 7: A Logging Buffer with `dmesg`-Style Behaviour

Build a second character device, `/dev/myfirst_log`, that uses an overwrite-mode cbuf to keep a circular log of recent driver events. Every `MYFIRST_DBG` macro call would write into this log instead of (or in addition to) calling `device_printf`.

- Use a separate `struct cbuf` in the softc.
- Provide a way for the kernel side to push lines into the log (`myfirst_log_printf(sc, fmt, ...)`).
- The user can `cat /dev/myfirst_log` to see the recent N lines.
- A new line that overflows the buffer evicts the oldest line, not just the oldest byte (this requires line-aware eviction logic).

This challenge introduces a fairly common driver pattern (a private debug log) and gives you practice with a second, independently designed buffer use case in the same module.

### Challenge 8: Performance Measurement

Build a measurement harness that times the throughput of the driver across the four stages.

- Write a small C program that opens the device, writes 100 MB of data, and times the operation.
- Mirror it with a reader that drains 100 MB and times itself.
- Run the pair against Stage 2, Stage 3, and Stage 4 of the chapter, and produce a small table of throughput numbers.
- Identify which stage is slowest and explain why.

The expected answer is "Stage 3 is slower than Stage 2 because of the extra `wakeup` and `selwakeup` calls per iteration; Stage 4 is similar to Stage 3 within measurement noise". But the actual numbers are interesting and may surprise you, depending on your CPU, memory bandwidth, and system load.

**Stretch:** profile the driver under load with `pmcstat(8)` and identify the top three functions by CPU time. If `uiomove` is in the top three, you have validated the discussion in Section 8 about zero-copy. If `mtx_lock` is in the top three, you have a contention problem that Chapter 11's locking material will address.

### Challenge 9: Cross-Reading Real Drivers

Pick three drivers in `/usr/src/sys/dev/` that you have not read before. For each one, identify:

- Where the buffer is allocated and freed.
- Whether it is a circular buffer, a queue, or a different shape.
- What protects it (mutex, sx, lock-free, none).
- How `read` and `write` handlers consume from or produce into it.
- How `select`/`poll`/`kqueue` integrate with the buffer state changes.

Suggested starting points: `/usr/src/sys/dev/iicbus/iiconf.c` (different category but uses some of the same primitives) and `/usr/src/sys/fs/cuse/cuse.c` (a driver that exposes its buffer to user space). You will see variations on the same themes you have just built.

### Challenge 10: Document Your Driver

Write a one-page README in your `examples/part-02/ch10-handling-io-efficiently/stage4-poll-refactor/` directory. The README should cover:

- What the driver does.
- How to build it (`make`).
- How to load and unload (`kldload`, `kldunload`).
- The user-space interface: device path, mode, reader/writer expectations, blocking behaviour.
- What the sysctls expose.
- How to enable debug logging.
- A reference to the chapter that produced it.

Documentation is the part of driver work most often skipped. A driver that only its author understands is a maintenance liability. Even a one-page README that explains the basics makes the difference between code that survives a hand-off and code that does not.

## Troubleshooting and Common Mistakes

Most of the bugs that appear in this chapter's territory cluster into a small number of categories. The list below catalogues the categories, the symptoms each one produces, and the fix. Read it once before you work through the labs; come back to it when something goes wrong.

### Symptom: `cat /dev/myfirst` blocks forever, even after `echo` writes data

**Cause.** The write handler is not calling `wakeup(&sc->cb)` after a successful `cbuf_write`. The reader is asleep on the channel `&sc->cb`; without a matching `wakeup`, it will never return.

**Fix.** Add `wakeup(&sc->cb)` after every state-changing operation that might unblock a waiter. In `myfirst_write`, that means after the `cbuf_write` call. In `myfirst_read`, that means after the `cbuf_read` call (which may unblock a waiting writer).

**How to verify.** Run `ps -AxH -o pid,wchan,command | grep cat`. If the `wchan` column shows `myfrd` (or whatever wmesg you used), the reader is sleeping. The channel address you slept on must match the channel address you wake.

### Symptom: Data corruption under heavy load

**Cause.** Almost always a wrap-around bug or a missing lock around the cbuf access. Either the cbuf's internal arithmetic is wrong, or two threads are touching it concurrently without synchronization.

**Fix.** Re-read the cbuf source carefully. Run the userland `cb_test` against your current `cbuf.c` (compile it directly with `cc`). If the userland tests pass, the problem is in the driver's locking, not in the cbuf. Check that every `cbuf_*` call is bracketed by `mtx_lock` and `mtx_unlock`. Use `INVARIANTS` and `WITNESS` in your kernel config to catch violations.

**How to verify.** Run `producer_consumer` with a known checksum. If the checksums match but mismatches are reported, the data is being reordered (a wrap-around bug). If the checksums differ, bytes are being lost or duplicated (a locking bug).

### Symptom: Kernel panic with "sleeping with mutex held"

**Cause.** You called `uiomove(9)`, `copyin(9)`, `copyout(9)`, or another sleeping function while holding `sc->mtx`. The sleeping function tried to fault on user memory, and the page-fault handler tried to sleep, but holding a non-sleepable mutex during a sleep is forbidden.

**Fix.** Release the mutex before any call that might sleep. The Stage 4 handlers do this carefully: lock to access the cbuf, unlock to call `uiomove`, lock again to update state.

**How to verify.** A `WITNESS`-enabled kernel will print a warning before panicking. The warning identifies the mutex and the sleeping function. The first time this happens, copy the message into a debug log so you can find the call site.

### Symptom: `EAGAIN` is returned even when data is available

**Cause.** The handler is checking the wrong flag, or it is checking the flag in the wrong place in the loop. Two common variants: checking `ioflag & O_RDONLY` instead of `ioflag & IO_NDELAY`, or returning `EAGAIN` after some bytes have already been transferred (which is the rule from Section 4 you must not break).

**Fix.** Re-read Section 5's handler code carefully. The `EAGAIN` path is inside the inner `while (cbuf_used(&sc->cb) == 0)` loop, after the `nbefore` check, only when `ioflag & IO_NDELAY` is non-zero.

**How to verify.** Run `rw_myfirst_nb`. Step 5 should successfully read the bytes. If it shows `EAGAIN`, the bug is at one of the two locations above.

### Symptom: A write succeeds but a subsequent read gets fewer bytes

**Cause.** The byte counter is being updated incorrectly, or the cbuf is being modified outside the handlers. A specific failure mode: counting `want` bytes as written when `cbuf_write` only stored `put` bytes (a race in Stage 2 between the cbuf_free check and the cbuf_write call, even though it is not exercised in single-writer use).

**Fix.** Look at the `bytes_written += put` line in `myfirst_write`; it must use the actual return value of `cbuf_write`, not the requested size. Compare `sc->bytes_written` and `sc->bytes_read` over time; they should differ by at most `cbuf_size`.

**How to verify.** Add a log line: `device_printf(dev, "wrote %zu of %zu\n", put, want);`. If `put != want` ever appears in `dmesg`, you have found the discrepancy.

### Symptom: `kldunload` returns `EBUSY`

**Cause.** Some descriptor is still open against the device. The detach refuses to proceed when `active_fhs > 0`.

**Fix.** Find the process that holds the descriptor open and close it. `fstat | grep myfirst` lists the offending processes. `kill` them if necessary.

**How to verify.** After closing all descriptors (or killing the offending processes), `sysctl dev.myfirst.0.stats.active_fhs` should drop to zero. `kldunload myfirst` should then succeed.

### Symptom: Memory growth in `vmstat -m | grep cbuf`

**Cause.** The driver is allocating without freeing. Either the attach failure path forgot to call `cbuf_destroy`, or the detach path forgot, or there is more than one cbuf being allocated per attach.

**Fix.** Audit every code path that calls `cbuf_init`. Each call must be matched by exactly one `cbuf_destroy` call before the surrounding context goes away. The standard idiom is to put `cbuf_init` near the top of `attach` and `cbuf_destroy` near the bottom of `detach`, with the failure-path `goto fail_*` chain calling `cbuf_destroy` if attach fails after `cbuf_init`.

**How to verify.** `kldload` and `kldunload` the module several times. `vmstat -m | grep cbuf` should show `0` after each `kldunload`.

### Symptom: `select(2)` or `poll(2)` does not wake up

**Cause.** The driver is missing a `selwakeup` call when state changes. Either the read path forgot to call `selwakeup(&sc->wsel)` after draining bytes, or the write path forgot to call `selwakeup(&sc->rsel)` after adding bytes.

**Fix.** The pattern: every state change that might make a previously-not-ready condition into a ready condition must be paired with a `selwakeup` call. Drain bytes -> `selwakeup(&sc->wsel)`. Add bytes -> `selwakeup(&sc->rsel)`.

**How to verify.** Run `rw_myfirst_nb`. Step 4 should show `revents=0x41`. If it shows `revents=0x0`, your `selwakeup` is missing or the `myfirst_poll` handler is not setting `revents` correctly.

### Symptom: `truss` shows `EINVAL` for zero-byte reads

**Cause.** Your handler is rejecting zero-byte reads with `EINVAL`. As discussed in Section 4, zero-byte reads and writes are legal and the handler must not error on them.

**Fix.** Remove any `if (uio->uio_resid == 0) return (EINVAL);` early return at the top of `myfirst_read` or `myfirst_write`.

**How to verify.** A program that calls `read(fd, NULL, 0)` should see the call return `0`, not `-1` with `EINVAL`.

### Symptom: `ps` shows the reader stuck in a sleep state called something different

**Cause.** Your `mtx_sleep` is being called with a different `wmesg` than you expected. Two common variants: typo (`mfyrd` instead of `myfrd`), or the same handler is being called from a code path where the wait reason is actually different.

**Fix.** Standardise the `wmesg` strings. `myfrd` for "myfirst read", `myfwr` for "myfirst write". A unique short string per wait point makes `ps -AxH` immediately informative.

**How to verify.** `ps -AxH` should show `myfrd` for sleeping readers and `myfwr` for sleeping writers.

### Symptom: A signal does not interrupt a blocked read

**Cause.** The `mtx_sleep` is being called without `PCATCH`. Without `PCATCH`, signals are deferred until the sleep ends for some other reason.

**Fix.** Always pass `PCATCH` for user-driven sleeps. The exception is sleeps that should be uninterruptible (kernel internal logic that must not be cancelled by a signal). For `myfirst_read` and `myfirst_write`, both are user-driven and both should pass `PCATCH`.

**How to verify.** `cat /dev/myfirst &` followed by `kill -INT %1` should cause `cat` to exit. If `cat` does not exit until you also write to the device (or send `kill -9`), `PCATCH` is missing.

### Symptom: Compiler warns about a missing prototype for `cbuf_read`

**Cause.** The driver source uses `cbuf_read` but does not include `cbuf.h`.

**Fix.** Add `#include "cbuf.h"` near the top of `myfirst.c`. The file path is relative to the source directory, so as long as both files are in the same directory the include will resolve.

**How to verify.** A clean build with no warnings.

### Symptom: `make` complains about missing `bus_if.h` or `device_if.h`

**Cause.** The Makefile is missing the standard `SRCS+= device_if.h bus_if.h` line that pulls in the auto-generated kobj headers for Newbus.

**Fix.** Use the Makefile from Section 3.

**How to verify.** `make clean && make` should succeed without missing-header errors.

### Symptom: `kldload` fails with `Exec format error`

**Cause.** The .ko was built against a different kernel than the one currently running. This typically happens when you reboot into a different kernel without rebuilding, or when you copy a .ko from one machine to another with different kernel sources.

**Fix.** `make clean && make` against the running kernel's `/usr/src`.

**How to verify.** `uname -a` should match the kernel version that built the .ko. Check `dmesg` after the failed `kldload` for more details.

### Symptom: The driver reports correct data but loses it after several runs

**Cause.** The cbuf is not being reset between attaches, or the softc is not being zero-initialised. With `M_ZERO` on the `malloc(9)` call (and the `cbuf_init` call zero-initializing its own state), this should not happen, but a partial fix that misses one of these can leave stale state.

**Fix.** Audit `myfirst_attach` to make sure every field of the softc is initialised explicitly. Use `M_ZERO` on the `malloc(9)` call that allocates the softc (Newbus does this automatically with `device_get_softc`, but verify). Use `cbuf_init` to set the cbuf's indices to zero.

**How to verify.** `kldload`, write some data, `kldunload`, `kldload` again. The new attach should report `cb_used = 0`.

### Symptom: `producer_consumer` reports a small number of mismatches under heavy load

**Cause.** A subtle locking bug, often related to the order of `wakeup` calls and the inner sleep loop's re-check. The classic symptom: under contention, occasionally a thread wakes up and consumes bytes that another thread had thought were still available.

**Fix.** Verify that every `mtx_sleep` is in a `while` (not `if`) loop, and that the loop re-checks the condition after waking. The wakeup is a *hint*, not a guarantee; a woken thread might find the condition false again because another thread got there first.

**How to verify.** `producer_consumer` should report zero mismatches across multiple runs. A mismatch count that varies between runs suggests a race; a mismatch count that is always exactly N suggests an off-by-one bug.

### General Advice for Debugging

Three habits make driver debugging much faster.

The first is to have a `WITNESS`-enabled kernel ready. `WITNESS` catches lock-ordering violations and "sleeping with mutex held" bugs that a production kernel would silently allow. The performance overhead is significant, so run `WITNESS` in your lab environment, not in production.

The second is to add `device_printf` log lines liberally during development, then remove them or guard them behind `myfirst_debug` before committing. The log buffer is finite, so do not log per-byte; one line per I/O call is the right granularity.

The third is to compile with `-Wall -Wextra` and treat warnings as bugs. The kernel build system passes a lot of warning flags by default; pay attention to them. Almost every warning is the kernel telling you about a real or potential bug.

When all else fails, sit back and trace through the code path on paper. A driver this size is small enough to fit on a single sheet. Ninety percent of the time, drawing the call graph and the lock acquisitions in order shows you the bug.

## Quick Reference: Patterns and Primitives

This reference is the chapter's material reduced to fast-lookup form. Use it after you have read the chapter; it is a reminder, not a tutorial.

### The Circular Buffer API

```c
struct cbuf {
        char    *cb_data;       /* backing storage */
        size_t   cb_size;       /* total capacity */
        size_t   cb_head;       /* next byte to read */
        size_t   cb_used;       /* live byte count */
};

int     cbuf_init(struct cbuf *cb, size_t size);
void    cbuf_destroy(struct cbuf *cb);
void    cbuf_reset(struct cbuf *cb);
size_t  cbuf_size(const struct cbuf *cb);
size_t  cbuf_used(const struct cbuf *cb);
size_t  cbuf_free(const struct cbuf *cb);
size_t  cbuf_write(struct cbuf *cb, const void *src, size_t n);
size_t  cbuf_read(struct cbuf *cb, void *dst, size_t n);
```

Rules:
- The cbuf does not lock; the caller is responsible.
- `cbuf_write` and `cbuf_read` clamp `n` to available space or live data and return the actual count.
- `cbuf_used` and `cbuf_free` return current state under the assumption that the caller holds whatever lock protects the cbuf.

### The Driver-Level Helpers

```c
size_t  myfirst_buf_read(struct myfirst_softc *sc, void *dst, size_t n);
size_t  myfirst_buf_write(struct myfirst_softc *sc, const void *src, size_t n);
int     myfirst_wait_data(struct myfirst_softc *sc, int ioflag, ssize_t nbefore,
            struct uio *uio);
int     myfirst_wait_room(struct myfirst_softc *sc, int ioflag, ssize_t nbefore,
            struct uio *uio);
```

Rules:
- All four helpers assert that `sc->mtx` is held with `mtx_assert(MA_OWNED)`.
- The wait helpers return `-1` to mean "break the outer loop, return 0 to user space".
- The wait helpers return `EAGAIN`, `EINTR`, `ERESTART`, or `ENXIO` for the corresponding conditions.

### The Read Handler Spine

```c
nbefore = uio->uio_resid;
while (uio->uio_resid > 0) {
        mtx_lock(&sc->mtx);
        error = myfirst_wait_data(sc, ioflag, nbefore, uio);
        if (error != 0) {
                mtx_unlock(&sc->mtx);
                return (error == -1 ? 0 : error);
        }
        take = MIN((size_t)uio->uio_resid, sizeof(bounce));
        got = myfirst_buf_read(sc, bounce, take);
        fh->reads += got;
        mtx_unlock(&sc->mtx);

        wakeup(&sc->cb);
        selwakeup(&sc->wsel);

        error = uiomove(bounce, got, uio);
        if (error != 0)
                return (error);
}
return (0);
```

### The Write Handler Spine

```c
nbefore = uio->uio_resid;
while (uio->uio_resid > 0) {
        mtx_lock(&sc->mtx);
        error = myfirst_wait_room(sc, ioflag, nbefore, uio);
        if (error != 0) {
                mtx_unlock(&sc->mtx);
                return (error == -1 ? 0 : error);
        }
        room = cbuf_free(&sc->cb);
        mtx_unlock(&sc->mtx);

        want = MIN((size_t)uio->uio_resid, sizeof(bounce));
        want = MIN(want, room);
        error = uiomove(bounce, want, uio);
        if (error != 0)
                return (error);

        mtx_lock(&sc->mtx);
        put = myfirst_buf_write(sc, bounce, want);
        fh->writes += put;
        mtx_unlock(&sc->mtx);

        wakeup(&sc->cb);
        selwakeup(&sc->rsel);
}
return (0);
```

### The Sleep Pattern

```c
mtx_lock(&sc->mtx);
while (CONDITION) {
        if (uio->uio_resid != nbefore)
                break_with_zero;
        if (ioflag & IO_NDELAY)
                return (EAGAIN);
        error = mtx_sleep(CHANNEL, &sc->mtx, PCATCH, "wmesg", 0);
        if (error != 0)
                return (error);
        if (!sc->is_attached)
                return (ENXIO);
}
/* condition is false now; act on the buffer */
```

Rules:
- Use `while`, not `if`, around the condition.
- Always pass `PCATCH` for user-driven sleeps.
- Always re-check the condition after `mtx_sleep` returns.
- Always check `is_attached` after waking, in case detach is pending.

### The Wake Pattern

```c
/* After a state change that might unblock a sleeper: */
wakeup(CHANNEL);
selwakeup(SELINFO);
```

Rules:
- The channel must match the channel passed to `mtx_sleep`.
- The selinfo must be the one that `selrecord` registered against.
- Use `wakeup` (wake all) for shared waiters; `wakeup_one` for single-handoff patterns.
- Spurious wakeups are safe; missing wakeups are bugs.

### The `d_poll` Handler

```c
static int
myfirst_poll(struct cdev *dev, int events, struct thread *td)
{
        struct myfirst_softc *sc = dev->si_drv1;
        int revents = 0;

        mtx_lock(&sc->mtx);
        if (events & (POLLIN | POLLRDNORM)) {
                if (cbuf_used(&sc->cb) > 0)
                        revents |= events & (POLLIN | POLLRDNORM);
                else
                        selrecord(td, &sc->rsel);
        }
        if (events & (POLLOUT | POLLWRNORM)) {
                if (cbuf_free(&sc->cb) > 0)
                        revents |= events & (POLLOUT | POLLWRNORM);
                else
                        selrecord(td, &sc->wsel);
        }
        mtx_unlock(&sc->mtx);
        return (revents);
}
```

### The `d_mmap` Handler

```c
static int
myfirst_mmap(struct cdev *dev, vm_ooffset_t offset, vm_paddr_t *paddr,
    int nprot, vm_memattr_t *memattr)
{
        struct myfirst_softc *sc = dev->si_drv1;

        if (sc == NULL || !sc->is_attached)
                return (ENXIO);
        if ((nprot & VM_PROT_WRITE) != 0)
                return (EACCES);
        if (offset >= sc->cb.cb_size)
                return (-1);
        *paddr = vtophys((char *)sc->cb.cb_data + (offset & ~PAGE_MASK));
        return (0);
}
```

### The `cdevsw`

```c
static struct cdevsw myfirst_cdevsw = {
        .d_version =    D_VERSION,
        .d_open =       myfirst_open,
        .d_close =      myfirst_close,
        .d_read =       myfirst_read,
        .d_write =      myfirst_write,
        .d_poll =       myfirst_poll,
        .d_mmap =       myfirst_mmap,
        .d_name =       "myfirst",
};
```

### Errno Values for I/O

| Errno     | Meaning                                          |
|-----------|--------------------------------------------------|
| `0`       | Success                                          |
| `EAGAIN`  | Would block; retry later                         |
| `EFAULT`  | Bad user pointer (from `uiomove`)                |
| `EINTR`   | Interrupted by a signal                          |
| `ENXIO`   | Device not present or torn down                  |
| `EIO`     | Hardware error                                   |
| `ENOSPC`  | Permanent out-of-space (block-on-full preferred) |
| `EACCES`  | Forbidden access mode                            |
| `EBUSY`   | Device is open or otherwise locked               |

### `ioflag` Bits

| Bit            | Source flag    | Meaning                                |
|----------------|----------------|----------------------------------------|
| `IO_NDELAY`    | `O_NONBLOCK`   | Caller is non-blocking                 |
| `IO_DIRECT`    | `O_DIRECT`     | Bypass caching where possible          |
| `IO_SYNC`      | `O_FSYNC`      | (write only) Synchronous semantics     |

`O_NONBLOCK == IO_NDELAY` per the kernel's CTASSERT.

### `poll(2)` Events

| Event        | Meaning                                          |
|--------------|--------------------------------------------------|
| `POLLIN`     | Readable: bytes are available                    |
| `POLLRDNORM` | Same as POLLIN for character devices             |
| `POLLOUT`    | Writable: space is available                     |
| `POLLWRNORM` | Same as POLLOUT for character devices            |
| `POLLERR`    | Error condition                                  |
| `POLLHUP`    | Hangup (peer closed)                             |
| `POLLNVAL`   | Invalid file descriptor                          |

A driver typically handles `POLLIN | POLLRDNORM` for read readiness and `POLLOUT | POLLWRNORM` for write readiness. The other events are usually set by the kernel, not the driver.

### Memory Allocator Reference

| Call                                        | When to use                          |
|---------------------------------------------|--------------------------------------|
| `malloc(n, M_DEVBUF, M_WAITOK \| M_ZERO)`   | Normal allocation, can sleep         |
| `malloc(n, M_DEVBUF, M_NOWAIT \| M_ZERO)`   | Cannot sleep (interrupt context)     |
| `free(p, M_DEVBUF)`                         | Free memory allocated above          |
| `MALLOC_DEFINE(M_TAG, "name", "desc")`      | Declare a private memory tag         |
| `contigmalloc(n, M_TAG, M_WAITOK, ...)`     | Physically contiguous allocation     |

### Sleep / Wake Reference

| Call                                                              | When to use                              |
|-------------------------------------------------------------------|------------------------------------------|
| `mtx_sleep(chan, mtx, PCATCH, "msg", 0)`                          | Sleep with mutex interlock               |
| `tsleep(chan, PCATCH \| pri, "msg", timo)`                        | Sleep without mutex (rare in drivers)    |
| `cv_wait(&cv, &mtx)`                                              | Sleep on a condition variable            |
| `wakeup(chan)`                                                    | Wake all sleepers on channel             |
| `wakeup_one(chan)`                                                | Wake one sleeper (for single-handoff)    |

### Lock Reference

| Call                                                | When to use                             |
|-----------------------------------------------------|-----------------------------------------|
| `mtx_init(&mtx, "name", "type", MTX_DEF)`           | Initialise a sleepable spin/sleep mutex |
| `mtx_destroy(&mtx)`                                 | Destroy at detach                       |
| `mtx_lock(&mtx)`, `mtx_unlock(&mtx)`                | Acquire / release                       |
| `mtx_assert(&mtx, MA_OWNED)`                        | Assert lock is held (debug)             |

### Test Tools Reference

| Tool                  | Use                                          |
|-----------------------|----------------------------------------------|
| `cat`, `echo`         | Quick smoke tests                            |
| `dd`                  | Volume tests, partial-transfer observation   |
| `hexdump -C`          | Verify byte content                          |
| `truss -t read,write` | Trace syscall returns                        |
| `ktrace`              | Detailed trace including signals             |
| `sysctl dev.myfirst.0.stats` | Live driver state                     |
| `vmstat -m`           | Memory tag accounting                        |
| `ps -AxH`             | Find sleeping threads and their wmesg        |
| `dmesg | tail`        | Driver-emitted log lines and kernel warnings |

### Driver Lifecycle Summary

```text
kldload
    -> myfirst_identify   (optional in this driver: creates the child)
    -> myfirst_probe      (returns BUS_PROBE_DEFAULT)
    -> myfirst_attach     (allocates softc, cbuf, cdev, sysctl, mutex)

steady state
    -> myfirst_open       (allocates per-fh state)
    -> myfirst_read       (drains cbuf via bounce + uiomove)
    -> myfirst_write      (fills cbuf via uiomove + bounce)
    -> myfirst_poll       (reports POLLIN/POLLOUT readiness)
    -> myfirst_close      (per-fh dtor releases per-fh state)

kldunload
    -> myfirst_detach     (refuses if any descriptor open)
    -> wakeup releases sleepers
    -> destroy_dev
    -> cbuf_destroy
    -> sysctl_ctx_free
    -> mtx_destroy
```

### File Layout Summary

```text
examples/part-02/ch10-handling-io-efficiently/
    README.md
    cbuf-userland/
        cbuf.h
        cbuf.c
        cb_test.c
        Makefile
    stage2-circular/
        cbuf.h
        cbuf.c
        myfirst.c
        Makefile
    stage3-blocking/
        cbuf.h
        cbuf.c
        myfirst.c
        Makefile
    stage4-poll-refactor/
        cbuf.h
        cbuf.c
        myfirst.c
        Makefile
    stage5-mmap/
        cbuf.h
        cbuf.c
        myfirst.c
        Makefile
    userland/
        rw_myfirst_v2.c
        rw_myfirst_nb.c
        producer_consumer.c
        stress_rw.c
        mmap_myfirst.c
        Makefile
```

Each stage directory is independent; you can `make` and `kldload` any of them without touching the others. The userland tools are shared across all stages.

### A Mental Model in One Paragraph

The driver owns a circular buffer, protected by a single mutex. A reader holds the mutex while transferring bytes out of the buffer into a stack-resident bounce, releases the mutex, copies the bounce into user space with `uiomove`, wakes any waiting writers, and loops until the user's request is satisfied or the buffer is empty. A writer mirrors this: holds the mutex, copies user bytes into a bounce, releases the mutex, copies the bounce into the buffer, wakes any waiting readers, and loops. When either the buffer is empty (for reads) or full (for writes), the handler either sleeps with the mutex as interlock (default mode) or returns `EAGAIN` (non-blocking mode). `select(2)` and `poll(2)` integration is provided through `selrecord` (in `d_poll`) and `selwakeup` (in the I/O handlers). The detach path waits for all descriptors to close and then frees everything.

That paragraph fits in your head. Everything else in this chapter is the careful elaboration of how to make each piece of it work.

## Appendix: A Source-Reading Walkthrough of `evdev/cdev.c`

The chapter has pointed at `/usr/src/sys/dev/evdev/cdev.c` several times as the cleanest example in the tree of a character device that does what `myfirst` now does: a per-client ring buffer, blocking reads, non-blocking support, `select`/`poll`/`kqueue` integration. Reading that file once, with the patterns from this chapter in hand, is the fastest way to confirm that the kernel really does work the way the chapter has been describing. This appendix walks through the relevant pieces.

The goal is *not* to teach `evdev`. It is to use `evdev` as an exhibit. By the end of this walkthrough you should feel that what you built in `myfirst` is the same shape as what the kernel uses for real input devices. The differences are in the details (the protocol, the structures, the layered driver stack), not in the underlying patterns.

### What `evdev` Is

`evdev` is FreeBSD's port of the Linux event-device interface. It exposes input devices (keyboards, mice, touchscreens) through `/dev/input/eventN` nodes that user-space programs (X servers, Wayland compositors, console handlers) read from to get a stream of input events. Each event is a fixed-size structure with a timestamp, a type, a code, and a value.

The driver layer that interests us is the per-client cdev. When a process opens `/dev/input/event0`, the kernel creates a `struct evdev_client` for that descriptor, attaches it to the underlying device, and uses it as the per-open buffer. Reads pull events out of the buffer; writes push events into it (for some devices); `select`/`poll`/`kqueue` report when events are available.

That description should sound very familiar by now. It is the same architecture as `myfirst` Stage 4, with three differences: the buffer is per-descriptor rather than per-device; the unit of transfer is a fixed-size structure rather than a byte; and the driver participates in a larger framework of input handling.

### The Per-Client State

Open `/usr/src/sys/dev/evdev/evdev_private.h` (the file is short; you can read the relevant parts in a couple of minutes). The key structure is `struct evdev_client`:

```c
struct evdev_client {
        struct evdev_dev *      ec_evdev;
        struct mtx              ec_buffer_mtx;
        size_t                  ec_buffer_size;
        size_t                  ec_buffer_head;
        size_t                  ec_buffer_tail;
        size_t                  ec_buffer_ready;
        ...
        bool                    ec_blocked;
        bool                    ec_revoked;
        ...
        struct selinfo          ec_selp;
        struct sigio *          ec_sigio;
        ...
        struct input_event      ec_buffer[];
};
```

Compare this against your `myfirst` softc:

- `ec_evdev` is `evdev`'s analogue of `dev->si_drv1` (a back-pointer from the per-client state to the device-wide state).
- `ec_buffer_mtx` is the per-client mutex; `myfirst`'s `sc->mtx` is per-device.
- `ec_buffer_size`, `ec_buffer_head`, `ec_buffer_tail`, `ec_buffer_ready` are the circular-buffer indices. Notice that `evdev` uses an explicit `tail` instead of a derived one; the code is slightly different but the structure is the same.
- `ec_blocked` is a hint flag for the wakeup logic.
- `ec_revoked` flags a forcibly-disconnected client; this is the equivalent of `is_attached` in `myfirst`.
- `ec_selp` is the `selinfo` for `select`/`poll`/`kqueue` support, exactly like `sc->rsel` and `sc->wsel` in your driver (combined here because evdev only does read-readiness; there is no concept of "write would block").
- `ec_buffer[]` is the flexible-array-member that holds the actual events.

The patterns are the same. The naming is different.

### The Read Handler

Open `/usr/src/sys/dev/evdev/cdev.c` and find `evdev_read`:

```c
static int
evdev_read(struct cdev *dev, struct uio *uio, int ioflag)
{
        struct evdev_dev *evdev = dev->si_drv1;
        struct evdev_client *client;
        ...
        ret = devfs_get_cdevpriv((void **)&client);
        if (ret != 0)
                return (ret);

        debugf(client, "read %zd bytes by thread %d", uio->uio_resid,
            uio->uio_td->td_tid);

        if (client->ec_revoked)
                return (ENODEV);

        ...
        if (uio->uio_resid != 0 && uio->uio_resid < evsize)
                return (EINVAL);

        remaining = uio->uio_resid / evsize;

        EVDEV_CLIENT_LOCKQ(client);

        if (EVDEV_CLIENT_EMPTYQ(client)) {
                if (ioflag & O_NONBLOCK)
                        ret = EWOULDBLOCK;
                else {
                        if (remaining != 0) {
                                client->ec_blocked = true;
                                ret = mtx_sleep(client, &client->ec_buffer_mtx,
                                    PCATCH, "evread", 0);
                                if (ret == 0 && client->ec_revoked)
                                        ret = ENODEV;
                        }
                }
        }

        while (ret == 0 && !EVDEV_CLIENT_EMPTYQ(client) && remaining > 0) {
                head = client->ec_buffer + client->ec_buffer_head;
                ...
                bcopy(head, &event.t, evsize);

                client->ec_buffer_head =
                    (client->ec_buffer_head + 1) % client->ec_buffer_size;
                remaining--;

                EVDEV_CLIENT_UNLOCKQ(client);
                ret = uiomove(&event, evsize, uio);
                EVDEV_CLIENT_LOCKQ(client);
        }

        EVDEV_CLIENT_UNLOCKQ(client);

        return (ret);
}
```

Walk through it slowly.

The handler retrieves the per-client state with `devfs_get_cdevpriv`, exactly as `myfirst_read` does. The `ec_revoked` check is `evdev`'s equivalent of `myfirst`'s `is_attached` check, except that `evdev` returns `ENODEV` rather than `ENXIO` (both are valid choices for "the device is gone").

The handler then validates that the requested transfer size is a multiple of an event-record size (because partial events make no sense to deliver). This is a layer above what `myfirst` does, and it is specific to event-stream devices.

Then, exactly as Section 5 described, the handler enters a *check-wait-recheck* loop. If the buffer is empty (`EVDEV_CLIENT_EMPTYQ(client)`), the handler either returns `EWOULDBLOCK` (the same value as `EAGAIN`) for non-blocking callers, or sleeps with `mtx_sleep` and `PCATCH`. The sleep channel is `client` itself; the mutex interlock is `&client->ec_buffer_mtx`. When the sleep returns, the handler re-checks `ec_revoked`, returning `ENODEV` if the client has been disconnected during the sleep.

After the wait, the handler enters the transfer loop. It picks an event off the buffer (with `bcopy` into a stack-resident `event` variable), advances `ec_buffer_head` modulo the buffer size, releases the mutex, and calls `uiomove` to push the event to user space. Then it reacquires the mutex and continues until either the buffer empties or the user's request is satisfied.

This is the bounce-buffer pattern from Section 3, with `event` playing the role of `bounce`. The cbuf operation is `bcopy(head, &event.t, evsize)` (single-event copy out of the ring) followed by `uiomove(&event, evsize, uio)` (transfer to user). The mutex is held only across the cbuf operation, never across `uiomove`. This is exactly the rule we made operational in `myfirst`.

### The Wakeup

Find `evdev_notify_event` (the function that gets called when a new event is delivered into a client's buffer):

```c
void
evdev_notify_event(struct evdev_client *client)
{

        EVDEV_CLIENT_LOCKQ_ASSERT(client);

        if (client->ec_blocked) {
                client->ec_blocked = false;
                wakeup(client);
        }
        if (client->ec_selected) {
                client->ec_selected = false;
                selwakeup(&client->ec_selp);
        }

        KNOTE_LOCKED(&client->ec_selp.si_note, 0);
}
```

There is the wake-up. `wakeup(client)` matches the `mtx_sleep(client, ...)` from `evdev_read`. `selwakeup(&client->ec_selp)` matches the `selrecord` we will look at in a moment. `KNOTE_LOCKED` is the `kqueue` analogue of `selwakeup`; we have not built that yet (it is Chapter 11 territory) but the pattern is the same.

The `ec_blocked` flag is an optimisation: if no client is currently sleeping, the wakeup is skipped. This is a small but useful optimisation. `myfirst` does not have it because the cost is negligible in our use case, but you could add the same check trivially.

### The Poll

Find `evdev_poll`:

```c
static int
evdev_poll(struct cdev *dev, int events, struct thread *td)
{
        struct evdev_client *client;
        int revents = 0;
        int ret;

        ret = devfs_get_cdevpriv((void **)&client);
        if (ret != 0)
                return (POLLNVAL);

        if (events & (POLLIN | POLLRDNORM)) {
                EVDEV_CLIENT_LOCKQ(client);
                if (!EVDEV_CLIENT_EMPTYQ(client))
                        revents = events & (POLLIN | POLLRDNORM);
                else {
                        client->ec_selected = true;
                        selrecord(td, &client->ec_selp);
                }
                EVDEV_CLIENT_UNLOCKQ(client);
        }

        return (revents);
}
```

This is essentially identical to the `myfirst_poll` from Section 5, with two differences. `evdev` only handles `POLLIN`; there is no `POLLOUT` because input events are unidirectional. And `evdev` returns `POLLNVAL` if `devfs_get_cdevpriv` fails, which is the conventional "this descriptor is invalid" response (compared to `myfirst`'s simpler return-zero approach).

The pattern is the one Section 5 introduced: check the condition, return ready if it is true, register with `selrecord` if it is not. The `ec_selected` flag is again a wakeup-elision optimisation; you can ignore it for understanding.

### The kqfilter

Find `evdev_kqfilter`:

```c
static int
evdev_kqfilter(struct cdev *dev, struct knote *kn)
{
        struct evdev_client *client;
        int ret;

        ret = devfs_get_cdevpriv((void **)&client);
        if (ret != 0)
                return (ret);

        switch(kn->kn_filter) {
        case EVFILT_READ:
                kn->kn_fop = &evdev_cdev_filterops;
                break;
        default:
                return(EINVAL);
        }
        kn->kn_hook = (caddr_t)client;

        knlist_add(&client->ec_selp.si_note, kn, 0);

        return (0);
}
```

This is the `kqueue` registration handler. It is one of Chapter 11's topics; we are showing it here only to point out that the `selinfo`'s `si_note` field is what `kqueue` hooks into. The `selrecord`/`selwakeup` machinery for `select`/`poll`, and the `knlist_add`/`KNOTE_LOCKED` machinery for `kqueue`, share the same `selinfo` structure. That sharing is what lets a single set of state-change calls (in `evdev_notify_event`) wake all three readiness-notification paths at once.

When we extend `myfirst` with `kqueue` support in Chapter 11, the changes will fit into roughly the same pattern: a `myfirst_kqfilter` handler that registers against `&sc->rsel.si_note`, a `KNOTE_LOCKED(&sc->rsel.si_note, 0)` call alongside each `selwakeup(&sc->rsel)`. The substrate is here already.

### What the Walkthrough Confirms

Three things should be clear now.

The first is that the *patterns* you have been building are not invented for this book. They are the patterns the kernel uses, in a real driver that ships with FreeBSD and is exercised by every keyboard and mouse user every day. You can read this code, recognise what each section is doing, and explain it. That is a real skill that pays off in every later chapter.

The second is that the *details* differ between drivers. `evdev` uses an explicit `tail` index. It uses fixed-size event records instead of bytes. It has per-client buffers instead of per-device. It uses `bcopy` instead of a cbuf abstraction. None of these differences invalidate the underlying pattern; they are choices about how to specialise the pattern for a specific use case.

The third is that *you can read more*. With a couple of hours and a coffee, you can work through `/usr/src/sys/dev/uart/uart_core.c` or `/usr/src/sys/dev/snp/snp.c`. Each one will look different at first glance, but the buffer, the locking, the sleep / wake, the poll: those will be familiar. The chapter has handed you a vocabulary; the kernel source is where you exercise it.

### A Short Reading Plan

If you want to build the habit of reading kernel source as part of your driver development workflow, here is a short plan. Spend an hour a week, for three weeks, on the following.

Week one: re-read `/usr/src/sys/dev/null/null.c` and `/usr/src/sys/dev/evdev/cdev.c`. Compare them. The first is the simplest possible character device; the second is a competent buffered one. Note exactly which features each file has and why.

Week two: read `/usr/src/sys/dev/random/randomdev.c`. It is bigger than `evdev` but uses the same patterns, with the addition of an entropy-collection layer underneath. Note how `randomdev_read` differs from `evdev_read` and `myfirst_read`, and why.

Week three: pick a driver in `/usr/src/sys/dev/` that interests you (a USB driver, a network driver, a storage driver). Read the part of it that handles user-space I/O. By now the patterns should be familiar enough that the unfamiliar parts (bus binding, hardware register access, DMA setup) stand out as the *new* things to learn, not as obstacles to understanding the I/O path.

After three weeks of this rhythm, you will have read more driver code than most professional kernel developers do in a typical month. The investment compounds.

## 章节总结

This chapter took the in-kernel buffer from Chapter 9, which was a linear FIFO that wasted half its capacity, and turned it into a real circular buffer with proper partial-I/O semantics, blocking and non-blocking modes, and `poll(2)` integration. The driver you finish the chapter with is meaningfully better than the one you started with, in four specific ways.

It uses its full capacity. The circular buffer keeps the entire allocation in play. A steady writer and a matching reader can keep the buffer at any fill level indefinitely; the wrap-around is invisible to the I/O handlers because it lives inside the cbuf abstraction.

It honours partial transfers. Both `myfirst_read` and `myfirst_write` loop until they cannot make progress, then return zero with `uio_resid` reflecting the unconsumed portion. A user-space caller that loops on `read(2)` or `write(2)` will see correct UNIX semantics. A caller that does not loop will still see correct counts; the driver does not silently throw bytes away.

It blocks correctly. A reader that finds the buffer empty sleeps on a clear channel, with the mutex released atomically; a writer that adds bytes wakes the sleeper. The same pattern works in the other direction. Signals are honoured through `PCATCH`, so the user's Ctrl-C interrupts a blocked reader within microseconds.

It supports non-blocking mode. A descriptor opened with `O_NONBLOCK` (or with the flag set later via `fcntl(2)`) sees `EAGAIN` instead of a sleep. A `d_poll` handler reports `POLLIN` and `POLLOUT` correctly based on the buffer's state, and `selrecord(9)` plus `selwakeup(9)` ensure that `select(2)` and `poll(2)` callers wake when readiness changes.

Each of those capabilities is built up in a numbered section, each with code that compiles, loads, and behaves predictably. The chapter's stages (a userland buffer, a kernel splice, a blocking-aware version, a poll-aware refactor, a memory-mapped variant) form a clear progression that matches the order in which a beginner naturally encounters these concerns.

Along the way we covered three supplementary topics that often appear next to buffered I/O in real drivers: `d_mmap(9)`, the patterns and limitations of zero-copy thinking, and the readahead and write-coalescing patterns used by high-throughput drivers. None of these is a Chapter 10 topic in its own right, but each one builds naturally on the buffer abstraction you have just put in place.

We exercised the driver with five new user-space test programs (`rw_myfirst_v2`, `rw_myfirst_nb`, `producer_consumer`, `stress_rw`, `mmap_myfirst`) plus the standard base-system tools (`dd`, `cat`, `hexdump`, `truss`, `sysctl`, `vmstat`). The combination is enough to catch most of the bugs this material typically produces. The Troubleshooting section catalogues those bugs with their symptoms and fixes; keep it bookmarked.

Finally, we refactored the buffer access into a small set of helpers (`myfirst_buf_read`, `myfirst_buf_write`, `myfirst_wait_data`, `myfirst_wait_room`), wrote a one-paragraph locking-strategy comment near the top of the source, and put the cbuf into its own file with its own `MALLOC_DEFINE`. The driver's source is now in the shape that Chapter 11 wants: clear locking discipline, narrow abstractions, no surprises.

## 总结

The buffered driver you have just finished is the foundation for everything that follows in Part 3. The shape of its I/O path, the discipline of its locking, the way it sleeps and wakes, the way it composes with `poll(2)`: these are not Chapter 10 patterns. They are *the* patterns the kernel uses, and once you recognise them in your own code, you will recognise them in every character-device driver in `/usr/src/sys/dev/`.

It is worth taking a moment to feel that shift. When you started Chapter 7, the inside of a driver was probably an opaque set of names and signatures. By the end of Chapter 8 you had a sense of the lifecycle. By the end of Chapter 9 you had a sense of the data path. Now, at the end of Chapter 10, you have a sense of how a driver behaves under load: how it shapes itself around concurrent callers, how it manages a finite resource (the buffer), how it cooperates with the kernel's readiness primitives, how it gets out of its own way so that user-space programs can do real work against it.

Most of what you have built will carry through to Chapter 11. The mutex, the buffer, the helpers, the locking strategy, the test kit, the lab discipline: all of it remains. What changes in Chapter 11 is the *depth* of the questions you ask about each piece. Why one mutex and not two? Why `wakeup` and not `wakeup_one`? Why `mtx_sleep` and not `cv_wait`? What guarantees does the kernel make about when a sleeper wakes, and what guarantees does it not make? How do you prove a piece of code is correct under concurrency, rather than hope?

Chapter 11 takes those questions seriously. It introduces `WITNESS` and `INVARIANTS` as the kernel's verification tools, walks through the lock classes, and discusses the patterns that turn merely-working concurrency into provably-correct concurrency. It will be a substantial chapter, but the substrate is what you have just built.

Three closing reminders before you move on.

The first is to *commit your code*. Whatever version-control system you use, save the four stage directories as a snapshot. The next chapter's first lab will copy your Stage 4 source and modify it; you do not want to lose the working baseline.

The second is to *try the labs*. Reading driver code teaches you the patterns; writing driver code teaches you the discipline. The labs in this chapter are short on purpose. Even the long ones can be done in a single sitting. The combination of "I built this" and "I broke it on purpose to see what happens" is what the chapter is designed to produce.

The third is to *trust the slow path*. The chapter has been deliberately careful, deliberately patient, deliberately repetitive in places. Driver work rewards that style. The bugs that hurt are the ones that look like they could not possibly happen. The defence against them is to be slow, careful, and methodical, even when the code seems simple. The reader who slows down with each step finishes Chapter 11 ready for Chapter 12; the reader who rushes finishes Chapter 11 with a kernel panic and a lost afternoon.

You are doing well. Keep going.

## Part 2 Checkpoint

Before you cross into Part 3, pause and check that the ground beneath your feet is solid. Part 2 has carried you from "what is a module" to "a multi-stage pseudo-driver that serves real readers and writers under load." The next part will put that driver on a much heavier scale, so the foundation needs to be firm.

By now you should be comfortable doing each of the following without hunting for the answer:

- Writing, compiling, loading, and unloading a kernel module against a running kernel, and reading `dmesg` to confirm the lifecycle.
- Building a Newbus skeleton with `device_probe`, `device_attach`, and `device_detach`, backed by a per-unit softc allocated with `device_get_softc`.
- Exposing a `/dev` node through a `cdevsw` with working `d_open`, `d_close`, `d_read`, `d_write`, and `d_ioctl` handlers, and verifying that `devfs` cleans the node up on unload.
- Managing a circular buffer whose state is protected by a mutex, whose readers can block with `mtx_sleep` and be woken by `wakeup`, and whose readiness is advertised through `selrecord` and `selwakeup`.
- Walking a deliberate failure through the attach path and watching every allocation unwind in reverse order.

If any of those feels unsteady, the labs that anchor them are worth a second pass. A targeted review list:

- Build, load, and unload discipline: Lab 7.2 (Build, Load, and Verify Lifecycle) and Lab 7.4 (Simulate Attach Failure and Verify Unwinding).
- `cdevsw` hygiene and `devfs` nodes: Lab 8.1 (Structured Name and Tighter Permissions) and Lab 8.5 (Two-Node Driver).
- Data path and round-trip behaviour: Lab 9.2 (Exercise Stage 2 with Writes and Reads) and Lab 9.3 (Stage 3 FIFO Behaviour).
- The Chapter 10 core sequence: Lab 2 (Stage 2 Circular Buffer), Lab 3 (Stage 3 Blocking and Non-Blocking), and Lab 4 (Stage 4 Poll Support and Refactor).

Part 3 will assume that all of the above is muscle memory, not a lookup. 具体来说, Chapter 11 will expect:

- A working Stage 4 `myfirst` that loads, unloads, and survives concurrent readers and writers without corruption.
- Familiarity with `mtx_sleep`/`wakeup` and the `selrecord`/`selwakeup` pair as the kernel's basic blocking and readiness primitives, since Part 3 will compare and contrast them with `cv(9)`, `sx(9)`, and `sema(9)`.
- A kernel built with `INVARIANTS` and `WITNESS`, since every Part 3 chapter leans on both from the first section onward.

If those three items hold, you are ready to turn the page. If one wobbles, fix that first. A quiet hour now saves a bewildering afternoon later.

## Looking Ahead: Bridge to Chapter 11

Chapter 11 is titled "Concurrency in Drivers." Its job is to take the driver you have just finished and look at it through the lens of concurrency: not the casual "it works under modest load" sense we have used so far, but the rigorous "I can prove this is correct under any interleaving" sense.

The bridge is built on three observations from Chapter 10's work.

First, you already have a single mutex protecting all shared state. That is the simplest non-trivial concurrency design a driver can have, and it is the right starting point for understanding the more elaborate alternatives. Chapter 11 will use your driver as a test case for asking when one mutex is enough, when it is not, and what to do when it is not.

Second, you already have a sleep / wake pattern that uses the mutex as an interlock. `mtx_sleep` and `wakeup` are the building blocks of every blocking primitive in the kernel. Chapter 11 will introduce condition variables (`cv_*`) as a more structured alternative, and will explain when each is appropriate.

Third, you already have a buffer abstraction that is intentionally not thread-safe by itself. The cbuf relies on the caller to provide locking. Chapter 11 will discuss the spectrum from "data structure provides no locking" (your cbuf) through "data structure provides internal locking" (some kernel primitives) to "data structure is lock-free" (`buf_ring(9)`, `epoch(9)`-based readers). Each end of the spectrum has uses; understanding when to choose which is part of becoming a driver writer.

Specific topics Chapter 11 will cover include:

- The five FreeBSD lock classes (`mtx`, `sx`, `rw`, `rm`, `lockmgr`) and when each is appropriate.
- Lock ordering and how to use `WITNESS` to verify it.
- The interaction between locks and interrupt context.
- Condition variables and when to prefer them over `mtx_sleep`.
- Reader/writer locks and their use cases.
- The `epoch(9)` framework for read-mostly data structures.
- Atomic operations (`atomic_*`) and when they obviate the need for locks.
- A walk-through of common concurrency bugs (lost wakeups, lock-order reversals, ABA, double-free under contention).

You do not need to read ahead to start Chapter 11. Everything in this chapter is sufficient preparation. Bring your Stage 4 driver, your test kit, and your `WITNESS`-enabled kernel; the next chapter starts where this one ended.

A small farewell from this chapter: you have just turned a beginner driver into a respectable one. The bytes that flow through `/dev/myfirst` are now flowing the way bytes flow through every other character device on the system. The patterns are right, the locking is right, the user-space contracts are honoured. The driver is yours to extend, to specialise, and to use as a baseline for whatever real device comes next. Take a moment to enjoy that, and then turn the page.
