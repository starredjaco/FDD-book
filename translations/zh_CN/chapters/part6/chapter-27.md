---
title: "使用存储设备与VFS层"
description: "Developing storage 设备驱动程序 and VFS integration"
partNumber: 6
partName: "编写特定传输层驱动程序"
chapter: 27
lastUpdated: "2026-04-19"
status: "complete"
author: "Edson Brandi"
reviewer: "TBD"
translator: "AI辅助翻译为简体中文"
estimatedReadTime: 225
language: "zh-CN"
---

# 使用存储设备与VFS层

## 引言

在上一章中，我们仔细地走过了USB串行驱动程序的生命周期。我们从内核在总线上发现设备的那一刻开始，跟踪它经过探测和附加，进入其作为字符设备的活跃生命周期，最终在硬件被拔出时通过分离退出。那次完整的演练教会了我们特定传输层驱动程序如何在FreeBSD中生存。它们参与总线，暴露面向用户的抽象，并接受自己可能随时消失的事实，因为底层硬件是可移除的。

存储驱动程序生活在一个完全不同的世界里。硬件仍然是真实的，许多存储设备仍然可能被意外移除，但驱动程序的角色以一种重要的方式发生了转变。USB串行适配器一次为一个进程提供字节流。而存储设备提供的是一个块可寻址的、持久的、结构化的表面，文件系统就建立在它之上。当用户插入USB串行适配器时，他们可能会立即打开`/dev/cuaU0`并开始一个会话。当用户插入磁盘时，他们几乎从不会将其作为原始流来读取。他们会挂载它，从那一刻起，磁盘就消失在文件系统之后、消失在缓存之后、消失在虚拟文件系统层之后，以及消失在共享其文件的众多进程之后。

本章将教你这种安排在驱动程序一侧发生了什么。你将了解VFS层是什么，它与`devfs`有何不同，以及存储驱动程序如何插入GEOM框架而不是直接与VFS层对话。你将从零开始编写一个小型伪块设备，将其暴露为GEOM提供者，赋予它一个可用的后备存储，观察`newfs_ufs`对其进行格式化，挂载结果，在其上创建文件，干净地卸载它，并在不留下内核足迹的情况下分离它。到本章结束时，你将拥有一个可用的存储栈心智模型和一个练习我们讨论的每一层的具体示例驱动程序。

本章很长，因为主题是分层的。与字符驱动程序的主要交互单元是来自进程的单个`read`或`write`调用不同，存储驱动程序生活在一个框架链中。请求从进程出发，经过VFS，经过缓冲区缓存，经过文件系统，经过GEOM，然后才到达驱动程序。回复则沿相反方向返回。在编写任何真正的存储代码之前，理解这条链是至关重要的，在诊断仅在负载下或在卸载期间出现的细微故障时，这种理解同样重要。我们将缓慢地推进基础部分，然后逐渐引入更多的层。

与第26章一样，本章的目标不是交付一个生产级块驱动程序。目标是给你一个坚固、正确、可读的第一个块驱动程序，让你完全理解它。真正的生产级存储驱动程序——无论是SATA磁盘、NVMe驱动器、SCSI控制器、SD卡还是虚拟块设备——都建立在相同的模式之上。一旦基础清晰了，从伪设备到真实设备的步骤主要就是用与硬件寄存器和DMA引擎对话的代码替换后备存储，以及处理真实磁盘暴露的更丰富的错误和恢复面。

你还将看到存储驱动程序如何与读者已经从FreeBSD用户侧了解的工具交互。`mdconfig(8)`将作为我们驱动程序的近亲出现，因为内核的`md(4)` RAM磁盘正是我们要构建的东西。`newfs_ufs(8)`、`mount(8)`、`umount(8)`、`diskinfo(8)`、`gstat(8)`和`geom(8)`将成为验证工具，而不仅仅是其他人使用的工具。本章的结构确保在你完成时，你可以在对你的设备运行`dd`的同时查看`gstat -I 1`的输出，并带着理解来阅读它。

最后，关于我们不会涵盖的内容的一点说明。我们不会编写与物理存储控制器对话的真正总线驱动程序。我们不会讨论UFS、ZFS、FUSE或其他特定文件系统的内部机制——超出理解它们在边界处如何与块设备相遇所需的范围。我们不会涵盖DMA、PCIe、NVMe队列或SCSI命令集。所有这些主题都值得单独讨论，在相关的地方，它们将出现在涵盖特定总线和特定子系统的后续章节中。我们在这里要做的是给你一个完整的、自包含的块层体验，它代表了FreeBSD中所有存储驱动程序如何与内核集成。

慢慢来阅读本章。慢慢读，输入代码，启动模块，格式化它，挂载它，故意破坏它，观察会发生什么。存储栈奖励耐心，惩罚走捷径。你不是在比赛。

## 第6部分与第1至第5部分有何不同

在本章开始之前，先做一个简短的框架说明。第27章位于一个要求你改变一个特定习惯的部分中，而这种转变在一开始就明确命名会更容易接受。

第1至第5部分通过连续二十章构建了一个持续运行的驱动程序`myfirst`，每一章都在同一个源码树中增加一项规范。第26章通过`myfirst_usb`作为传输层的兄弟扩展了这个家族，这样迈向真实硬件的步骤不会同时也是迈向陌生源码的步骤。**从第27章开始，持续运行的`myfirst`驱动程序作为本书的主干暂停了。**第6部分转向新的、自包含的演示程序，以适应它所教授的每个子系统：第27章中的伪块设备用于存储，第28章中的伪网络接口用于网络。这些演示程序在精神上与`myfirst`平行，但在代码上是独立的，因为定义存储驱动程序或网络驱动程序的模式不适合`myfirst`所成长的字符设备模具。

**训练规范和教学形态保持不变。**每一章仍然引导你完成探测、附加、主数据路径、清理路径、实验、挑战练习、故障排除和通往下一章的桥梁。每一章仍然将示例建立在`/usr/src`下的真实FreeBSD源码之上。你在第25章及更早章节中建立的习惯——带标签的goto清理链、限速日志、`INVARIANTS`和`WITNESS`、生产就绪检查清单——无需修改即可延续。改变的是你面前的代码产物：一个小型、专注的驱动程序，其形状与所研究的子系统相匹配，而不是`myfirst`时间线中的又一个阶段。

这是一个经过深思熟虑的教学选择，而非范围上的偶然。存储驱动程序和网络驱动程序各自拥有自己的生命周期、自己的数据流、自己偏好的惯用法，以及自己要插入的框架。将它们作为全新的驱动程序来教授，而不是作为`myfirst`的进一步变体，可以保持对每个子系统的独特之处的关注。试图将`myfirst`延伸为块设备或网络接口的读者，很快就会得到一段关于存储或网络什么也教不了的代码。全新的演示程序是更清晰的路径，也是本部分所采取的路径。

第7部分回归累积式学习，但它不是恢复单一运行中的驱动程序，而是回顾你已经编写过的驱动程序（`myfirst`、`myfirst_usb`和第6部分的演示程序），并教授一旦驱动程序的第一个版本存在后就变得重要的面向生产力的主题：跨架构的可移植性、高级调试、性能调优、安全审查和对上游项目的贡献。累积构建的习惯将伴随你；改变的只是你面前的具体产物。

在第27章展开的过程中，请记住这个框架。如果在二十章相同源码树之后，从`myfirst`切换到新的伪块设备感觉有些突兀，这种反应是正常的，通常会很快过去，一般到第3节结束时就会消失。

## 读者指南：如何使用本章

本章被设计为FreeBSD内核存储方面的引导式课程。它是本书中较长的章节之一，因为主题是分层的，每一层都有自己的词汇、自己的关注点和自己的故障模式。你不需要急于求成。

如果你选择**仅阅读路径**，预计需要花费大约两到三个小时仔细阅读本章。你将获得关于VFS、缓冲区缓存、文件系统、GEOM和块设备边界如何组合在一起的清晰图景，并且你面前将有一个具体的驱动程序作为心智模型的锚点。这是使用本章的合理方式，尤其是在第一次阅读时。

如果你选择**阅读加实验路径**，计划在连续一到两个晚上花费四到六个小时，具体取决于你对第26章中内核模块的熟悉程度。你将构建驱动程序、格式化它、挂载它、在负载下观察它，并安全地拆解它。预计到本章结束时，`kldload`、`kldunload`、`newfs_ufs`和`mount`的操作将变得轻车熟路。

如果你选择**阅读加实验加挑战路径**，计划在一个周末或分散在一个月内的两个晚上。这些挑战在实际重要的方向上扩展了驱动程序：添加可选的刷新语义、用清零响应`BIO_DELETE`、支持多个单元、通过`disk_getattr`导出额外属性、以及干净地强制只读模式。每个挑战都是自包含的，只使用本章已经涵盖的内容。

无论你选择哪条路径，都不要跳过故障排除部分。存储bug从外部看往往很相似，通过症状识别它们的能力在实践中远比记住GEOM中每个函数的名字有用。故障排除材料放在接近末尾的位置以便于阅读，但你可能会在做实验时回过头来查阅它。

关于先决条件的说明。本章直接建立在第26章之上，因此你至少应该能够熟练编写小型内核模块、声明softc、分配和释放资源，以及走过加载和卸载路径。你还应该足够熟悉shell，能够在不停下来查找标志的情况下运行`kldload`、`kldstat`、`dmesg`、`mount`和`umount`。如果其中任何内容感觉不熟悉，值得在继续之前回顾第5、14和26章。

你应该在一个一次性的FreeBSD 14.3系统、虚拟机或不介意偶尔内核崩溃的分支上工作。如果你仔细按照文本操作，崩溃是不太可能的，但在开发笔记本电脑上犯错的代价远高于在可以回滚的VM快照上犯错的代价。我们之前说过这话，我们会继续说：内核工作在安全的地方进行时是安全的。

### 按节逐读

本章按循序渐进的方式组织。第1节介绍VFS。第2节对比`devfs`和VFS，并将我们的驱动程序定位在这种对比之中。第3节注册一个最小的伪块设备。第4节将其暴露为GEOM提供者。第5节实现真正的读写路径。第6节在顶部挂载文件系统。第7节赋予设备持久性。第8节教授安全的卸载和清理。第9节讨论重构、版本管理以及驱动程序增长时该怎么做。

你应该按顺序阅读它们。每一节都假设前面的章节内容在你脑海中仍然清晰，实验也是相互构建的。如果你跳到中间，内容会显得很奇怪。

### 输入代码

手动输入代码仍然是内化内核惯用法的最有效方式。`examples/part-06/ch27-storage-vfs/`下的配套文件是为了让你检查工作，而不是让你跳过输入。阅读代码和编写代码不是一回事。

### 打开FreeBSD源码树

你将被多次要求打开真正的FreeBSD源文件，而不仅仅是配套示例。感兴趣的文件包括`/usr/src/sys/geom/geom.h`、`/usr/src/sys/sys/bio.h`、`/usr/src/sys/geom/geom_disk.h`、`/usr/src/sys/dev/md/md.c`和`/usr/src/sys/geom/zero/g_zero.c`。每一个都是主要参考，本章的文字经常会引用它们。如果你还没有克隆或安装14.3源码树，现在是个好时机。

### 使用你的实验日志

在工作时保持第26章的实验日志打开。你会想记录`gstat -I 1`的输出、加载和卸载模块时`dmesg`发出的消息、格式化设备所需的时间，以及你看到的任何警告或崩溃。当你做笔记时，内核工作会变得容易得多，因为许多症状乍一看很相似，日志让你可以跨会话进行比较。

### 掌握节奏

如果你感觉在某一节中理解变得模糊，停下来。重新阅读它。在运行中的模块上尝试一个小实验。不要硬撑过尚未理解的小节。存储驱动程序比字符驱动程序更严厉地惩罚混乱，因为块层的混乱往往会变成上层的文件系统损坏，而文件系统损坏即使在一次性虚拟机中也需要时间和精力来修复。

## 如何从本章获得最大收益

本章的结构使得每一节都恰好在前一节的基础上增加一个新概念。为了充分利用这种结构，把本章当作研讨会而不是参考手册。你不是来这里找快速答案的。你是来这里构建正确的心智模型的。

### 按节学习

不要一口气从头到尾读完整个章节。读一节，然后暂停。尝试与之配套的实验或练习。查看相关的FreeBSD源码。在日志中写几行。然后再继续。内核中的存储编程是高度累积性的，跳过前进通常意味着你会因为两节之前解释过的原因而对下一个内容感到困惑。

### 保持驱动程序运行

一旦你在第3节加载了驱动程序，在阅读时尽量保持它处于加载状态。修改它，重新加载，用`gstat`探查它，对它运行`dd`，对它调用`diskinfo`。拥有一个活生生的、可观察的示例远比任何数量的阅读都更有价值。你会注意到没有任何章节能告诉你的事情，因为没有章节能向你展示真实的时序、真实的抖动或你特定设置中的真实边界情况。

### 查阅手册页

FreeBSD的手册页是教学材料的一部分，而不是额外的形式要求。手册的第9节是内核接口所在的地方。我们将多次引用`g_bio(9)`、`geom(4)`、`DEVICE_IDENTIFY(9)`、`disk(9)`、`bus_dma(9)`和`devstat(9)`等页面。与本章一起阅读它们。它们比你想象的要短，而且它们是由编写你正在使用的内核的同一社区编写的。

### 输入代码，然后修改它

当你从配套示例构建驱动程序时，先输入它。一旦它工作了，就开始修改它。重命名一个方法，观察构建失败。删除一个`if`分支，观察加载模块时发生什么。硬编码一个更小的介质大小，观察`newfs_ufs`的反应。内核代码通过有意的修改变得可理解，远比纯粹阅读更有效。

### 信任工具

FreeBSD为你提供了丰富的工具来检查存储栈：`geom`、`gstat`、`diskinfo`、`dd`、`mdconfig`、`dmesg`、`kldstat`、`sysctl`。使用它们。当出现问题时，第一步几乎从不是阅读更多源码。而是询问系统当前处于什么状态。`geom disk list`和`geom part show`通常比五分钟的grep更有信息量。

### 适当休息

内核工作在认知上是密集的。两到三个专注的小时通常比七个小时的冲刺更有效率。如果你发现自己犯了三次同样的打字错误，或者不看就复制粘贴，那就是你该站起来休息十分钟的信号。

建立了这些习惯后，让我们开始吧。

## 第1节：什么是虚拟文件系统层？

当进程在FreeBSD上打开文件时，它会带着路径调用`open(2)`。该路径可能解析为UFS上的文件、ZFS上的文件、远程挂载的NFS共享上的文件、`devfs`中的伪文件、`procfs`下的文件，甚至FUSE挂载的用户态文件系统中的文件。进程无法区分。进程收到一个文件描述符，然后像世界上只有一种文件一样进行读写。这种统一性并非偶然。这是虚拟文件系统层的工作。

### VFS解决的问题

在VFS出现之前，UNIX内核通常只知道如何与一种文件系统对话。如果你想要一个新的文件系统，你必须修改`open`、`read`、`write`、`stat`、`unlink`、`rename`以及每个涉及文件的系统调用的代码路径。这种方法在一段时间内有效，但无法扩展。新的文件系统到来了：用于远程访问的NFS，用于内存暂存空间的MFS，用于暴露进程状态的Procfs，用于CD-ROM介质的ISO 9660，用于互操作的FAT。每次添加都意味着在每个与文件相关的系统调用中出现新的分支。

Sun Microsystems在1980年代中期引入了虚拟文件系统架构，作为摆脱这种困境的方法。思路很简单。内核与一个单一抽象接口对话，该接口以通用文件对象上的通用操作来定义。每个具体的文件系统注册这些操作的实现，内核通过函数指针调用它们。当内核需要读取文件时，它不知道也不关心文件存储在UFS还是NFS还是ZFS上。它知道有一个带有`VOP_READ`方法的节点，然后调用该方法。

FreeBSD采用了这种架构，并在数十年间对其进行了显著扩展。结果是，向FreeBSD添加文件系统不再需要修改核心系统调用。文件系统是一个独立的内核模块，它向VFS注册一组操作，从那一刻起，VFS将正确的请求路由给它。

### VFS对象模型

VFS定义了三种主要对象。

第一种是**挂载点**，在内核中用`struct mount`表示。每个挂载的文件系统都有一个挂载点，它记录了文件系统附加在命名空间中的位置、它有什么标志，以及哪个文件系统代码负责它。

第二种是**vnode**，用`struct vnode`表示。vnode是内核对已挂载文件系统中单个文件或目录的句柄。它不是文件本身。它是内核对该文件的运行时表示，只要内核中有东西关心它就存在。每个进程打开的文件都有一个vnode。内核遍历的每个目录都有一个vnode。当没有任何东西持有对vnode的引用时，它就可以被回收，内核保留一个vnode池以避免小inode情况下的压力。

第三种是**vnode操作向量**，用`struct vop_vector`表示，它列出了每个文件系统必须在vnode上实现的操作。这些操作有`VOP_LOOKUP`、`VOP_READ`、`VOP_WRITE`、`VOP_CREATE`、`VOP_REMOVE`、`VOP_GETATTR`和`VOP_SETATTR`等名称。每个文件系统提供指向自己向量的指针，内核在需要对文件执行任何操作时通过这些向量调用操作。

这种设计的优雅之处在于，从内核的系统调用一侧看，只有抽象接口重要。系统调用层调用`VOP_READ(vp, uio, ioflag, cred)`，不关心`vp`属于UFS、ZFS、NFS还是tmpfs。从文件系统一侧看，也只有抽象接口重要。UFS实现vnode操作，永远不会看到系统调用代码。

### 存储驱动程序的位置

这是本章的关键问题。如果VFS是文件系统所在的地方，那么存储驱动程序在哪里？

答案是：不直接在VFS内部。存储驱动程序不实现`VOP_READ`。它实现的是一个看起来像磁盘的低得多层的抽象。然后文件系统位于其上，消费类似磁盘的抽象，将文件级操作转换为块级操作，并向下调用。

FreeBSD中进程和块设备之间的层链通常如下所示。

```text
       +------------------+
       |   user process   |
       +--------+---------+
                |
                |  read(fd, buf, n)
                v
       +--------+---------+
       |   system calls   |  sys_read, sys_write, sys_open, ...
       +--------+---------+
                |
                v
       +--------+---------+
       |       VFS        |  vfs_read, VOP_READ, vnode cache
       +--------+---------+
                |
                v
       +--------+---------+
       |    filesystem    |  UFS, ZFS, NFS, tmpfs, ...
       +--------+---------+
                |
                v
       +--------+---------+
       |   buffer cache   |  bufcache, bwrite, bread, getblk
       +--------+---------+
                |
                |  struct bio
                v
       +--------+---------+
       |      GEOM        |  classes, providers, consumers
       +--------+---------+
                |
                v
       +--------+---------+
       |  storage driver  |  disk_strategy, bio handler
       +--------+---------+
                |
                v
       +--------+---------+
       |    hardware      |  real disk, SSD, or memory buffer
       +------------------+
```

这个栈中的每一层都有各自的职责。VFS向系统调用隐藏文件系统的差异。文件系统将文件转换为块。缓冲区缓存在RAM中保存最近使用的块。GEOM通过变换、分区和镜像来路由块请求。存储驱动程序将块请求转换为实际的I/O。硬件执行具体工作。

在本章中，我们几乎所有的操作都发生在最底部的两层：GEOM和存储驱动程序。我们将在设备上挂载UFS时简要涉及文件系统层，而VFS仅在`mount(8)`调用它的意义上会涉及到。GEOM以上的层不是我们的代码。

### 内核源码中的VFS

如果你想直接查看VFS，入口点在`/usr/src/sys/kern/vfs_*.c`下。vnode层在`vfs_vnops.c`和`vfs_subr.c`中。挂载侧在`vfs_mount.c`中。vnode操作向量在`vfs_default.c`中定义和处理。UFS，本章的主要文件系统，位于`/usr/src/sys/ufs/ufs/`和`/usr/src/sys/ufs/ffs/`下。你不需要阅读其中任何一个来跟随本章。你应该知道它们在哪里，以便理解你即将编写的代码之上有什么。

### 这对我们的驱动程序意味着什么

因为VFS不是我们的直接调用者，我们不需要实现`VOP_`方法。我们需要实现的是文件系统最终调用到的块层接口。该接口由GEOM定义，对于类似磁盘的设备，由`g_disk`子系统定义。我们的驱动程序将暴露一个GEOM提供者。文件系统将消费它。I/O的流动将通过`struct bio`而不是`struct uio`，工作单元将是块而不是字节范围。

这也解释了为什么存储驱动程序很少像字符驱动程序那样直接与`cdevsw`或`make_dev`交互。磁盘的`/dev`节点是由GEOM创建的，而不是由驱动程序创建的。驱动程序向GEOM描述自己，GEOM发布一个提供者，该提供者随后以自动生成的名称出现在`/dev`中。

### VFS调用链实践

让我们追踪一下当用户运行`cat /mnt/myfs/hello.txt`时会发生什么，假设`/mnt/myfs`挂载在我们未来的块设备上。

首先，进程调用`open("/mnt/myfs/hello.txt", O_RDONLY)`。这会进入系统调用层的`sys_openat`，后者要求VFS解析路径。VFS逐个组件地遍历路径，在每个目录vnode上调用`VOP_LOOKUP`。当到达`myfs`时，它注意到vnode是一个挂载点并跨越到已挂载的文件系统中。它最终到达`hello.txt`的vnode并返回一个文件描述符。

第二步，进程调用`read(fd, buf, 64)`。这进入`sys_read`，后者调用`vn_read`，再调用vnode上的`VOP_READ`。UFS的`VOP_READ`实现查阅其inode，找出哪些磁盘块保存了请求的字节，然后向缓冲区缓存请求这些块。如果块未被缓存，缓冲区缓存调用`bread`，最终构建一个`struct bio`并将其交给GEOM。

第三步，GEOM查看文件系统正在消费的提供者。通过提供者和消费者的链条，`bio`最终到达最底层的提供者，也就是我们驱动程序的提供者。我们的策略函数接收`bio`，从后备存储中读取请求的字节，并调用`biodone`或`g_io_deliver`来完成请求。

第四步，回复沿相反方向返回。缓冲区缓存获取其数据，文件系统返回到`vn_read`，`vn_read`将数据复制到用户缓冲区中，`sys_read`返回。

除了最后一跳，所有这些代码都不是我们的。但理解整个链路能让你在编写最后一跳时做出合理的设计选择。

### 总结 Section 1

VFS是FreeBSD中统一文件系统的层。它位于系统调用接口和各种具体文件系统之间，提供了使文件无论存储在哪里看起来都相同的抽象。存储驱动程序不在VFS内部。它们位于栈的底部，远在VFS之下，在GEOM和缓冲区缓存之后。我们在本章的任务是编写一个正确参与该底层的驱动程序，并充分理解上层以避免在诊断问题时产生困惑。

在下一节中，我们将明确`devfs`和VFS之间的区别，因为该区别决定了你在考虑给定设备节点时适用哪种心智模型。

## 第2节：devfs与VFS

初学者常常认为`devfs`和虚拟文件系统层是同一事物的两个名称。它们不是。它们有关联，但扮演着非常不同的角色。尽早弄清这个区别可以在以后省去很多困惑，特别是在考虑存储驱动程序时，因为存储驱动程序横跨两者。

### 什么是devfs

`devfs`是一个文件系统。这听起来像是循环论证，但确实如此。`devfs`被实现为一个文件系统模块，向VFS注册，并在每个FreeBSD系统上挂载到`/dev`。当你在`/dev`下读取文件时，你是通过VFS进行读取的，VFS将请求交给`devfs`，`devfs`识别出你正在读取的"文件"实际上是一个内核设备节点，并将调用路由到相应的驱动程序。

`devfs`有几个特殊属性，使它区别于像UFS这样的普通文件系统。

第一，其内容不存储在磁盘上。`devfs`中的"文件"是由内核根据当前加载的驱动程序和当前存在的设备合成的。当驱动程序调用`make_dev(9)`创建`/dev/mybox`时，`devfs`将相应的节点添加到其视图中。当驱动程序用`destroy_dev(9)`销毁该设备时，`devfs`移除该节点。用户会实时看到`/dev/mybox`出现和消失。

第二，`devfs`节点的读路径和写路径不是文件数据路径。当你写入`/dev/myserial0`时，你不是在向存储的文件追加字节。你是在通过`cdevsw`调用驱动程序的`d_write`函数，该函数决定这些字节的含义。对于USB串行驱动程序，它们意味着要在传输线上发送的字节。对于像`/dev/null`这样的伪设备，它们意味着要丢弃的字节。

第三，`devfs`节点的元数据（如权限和所有权）由内核中的策略层管理，而不是由文件系统本身管理。`devfs_ruleset(8)`和`devd`框架配置该策略。

第四，`devfs`支持克隆，字符驱动程序如`pty`、`tun`和`bpf`利用克隆在进程打开节点时创建新的次设备。这就是`/dev/ptyp0`、`/dev/ptyp1`及其后继者按需产生的方式。

### 什么是VFS

正如我们在第1节中看到的，VFS是抽象的文件系统层。FreeBSD系统上的每个文件系统，包括`devfs`，都向VFS注册并通过VFS调用。VFS不是文件系统。它是文件系统插入的框架。

当你在UFS上打开文件时，链路是：系统调用 -> VFS -> UFS -> 缓冲区缓存 -> GEOM -> 驱动程序。当你在`devfs`中打开节点时，链路是：系统调用 -> VFS -> devfs -> 驱动程序。两者都经过VFS。只有UFS链路涉及GEOM。

### 存储驱动程序为何横跨两端

这就是存储驱动程序变得有趣的地方。

存储驱动程序暴露一个块设备，该块设备最终以`/dev`下的节点形式出现。例如，如果我们注册驱动程序并告知GEOM，`devfs`中可能会出现一个名为`/dev/myblk0`的节点。当用户写入`dd if=image.iso of=/dev/myblk0`时，他们正在通过`devfs`写入GEOM在我们磁盘之上提供的特殊字符接口。请求以BIO的形式通过GEOM流入我们的策略函数。

但当用户运行`newfs_ufs /dev/myblk0`然后`mount /dev/myblk0 /mnt`时，使用模式就改变了。内核现在在设备之上挂载UFS。当进程随后在`/mnt`下读取文件时，路径是：系统调用 -> VFS -> UFS -> 缓冲区缓存 -> GEOM -> 驱动程序。`devfs`中的`/dev/myblk0`节点甚至不参与热路径。UFS和缓冲区缓存直接与GEOM提供者对话。`devfs`节点本质上是工具用来引用设备的句柄，而不是正常操作期间文件数据流经的管道。

### 深入了解缓冲区缓存

在存储路径中，文件系统和GEOM之间是缓冲区缓存。我们已经提到过它好几次但没有停下来描述。现在让我们停下来，因为它解释了你在测试驱动程序时会观察到的几种行为。

缓冲区缓存是内核内存中固定大小缓冲区的池，每个缓冲区保存一个文件系统块。当文件系统读取一个块时，缓冲区缓存就会参与：文件系统向缓存请求该块，缓存要么返回命中（该块已在内存中），要么发出未命中（缓存分配一个缓冲区，通过GEOM向下调用获取数据，并在读取完成后返回该缓冲区）。当文件系统写入一个块时，相同的缓存路径以相反方式应用：写入填充一个缓冲区，该缓冲区被标记为脏，缓存安排在稍后的某个时间点进行回写。

缓冲区缓存就是为什么对同一文件数据的连续读取并不总是到达驱动程序的原因。第一次读取未命中，导致BIO传输到驱动程序。第二次读取命中缓存并立即返回。这是一个很好的性能特性。在你第一次调试驱动程序时，这可能会让人有些困惑，因为策略函数中的`printf`不会在每次用户空间读取时触发。

缓冲区缓存也是为什么写入看起来比底层驱动程序更快的原因。一个`dd if=/dev/zero of=/mnt/myblk/big bs=1m count=16`可能在几分之一秒内就看似完成了，因为写入进入了缓存，缓存将实际的BIO延迟了一段时间。文件系统在接下来的一两秒内向GEOM发出真正的写入。如果系统在此之前崩溃，磁盘上的文件是不完整的。`sync(2)`强制缓存刷新到底层设备。`fsync(2)`仅刷新与单个文件描述符关联的缓冲区。

缓冲区缓存与页缓存是不同的。FreeBSD两者都有，它们相互协作。页缓存保存支持内存映射文件和匿名内存的内存页。缓冲区缓存保存支持文件系统块操作的缓冲区。现代FreeBSD在许多数据路径上已经基本统一了它们，但这种区别仍然出现在源码树中，特别是在`bread`、`bwrite`、`getblk`和`brelse`周围，它们是接口的缓冲区缓存侧。

缓冲区缓存对我们的驱动程序有一个最重要的含义：我们几乎永远不会看到完全同步的BIO流量。当文件系统想要读取一个块时，一个BIO到达我们的策略函数；当文件系统想要写入一个块时，另一个BIO到达，但通常比触发它的写入系统调用晚一些。BIO也会在缓存刷新时以突发方式到达。这是正常的，你的驱动程序不能对BIO之间的时序或顺序做出假设，除非严格文档化的情况。每个BIO都是独立的请求。

### 读路径和写路径

让我们追踪一个贯穿整个链路的具体示例。

当用户运行`cat /mnt/myblk/hello.txt`时，shell运行`cat`，`cat`调用`open("/mnt/myblk/hello.txt", O_RDONLY)`。`open`进入`sys_openat`，后者交给VFS。VFS调用`namei`遍历路径。对于每个路径组件，VFS在当前目录的vnode上调用`VOP_LOOKUP`。当VFS到达`myblk`挂载点时，它跨越到UFS中，UFS遍历其目录结构来找到`hello.txt`。UFS返回该文件的vnode，VFS返回一个文件描述符。

然后用户调用`read(fd, buf, 64)`。`sys_read`调用`vn_read`，后者在vnode上调用`VOP_READ`。UFS的`VOP_READ`查阅inode以找到请求字节的块地址，然后在缓冲区缓存上调用`bread`来获取该块。缓冲区缓存要么返回命中，要么发出一个BIO。

如果是缓存未命中，缓冲区缓存分配一个新缓冲区，构建一个向底层GEOM提供者请求相关块的BIO，并将其发出。BIO向下通过GEOM，通过我们的策略函数，然后返回。当BIO完成时，缓冲区缓存解除等待中的`bread`调用的阻塞。然后UFS将请求的字节从缓冲区复制到用户的`buf`中。`read`返回。

对于写入，链路是对称的，但时序不同。UFS的`VOP_WRITE`调用`bread`或`getblk`来获取目标缓冲区，将用户数据复制到缓冲区中，将缓冲区标记为脏，然后调用`bdwrite`或`bawrite`来安排回写。用户的`write`调用在BIO向驱动程序发出之前很久就返回了。稍后，缓冲区缓存的同步线程拾取脏缓冲区并向驱动程序发出BIO_WRITE请求。

最终效果是，我们驱动程序的策略函数看到的是一系列BIO，这些BIO与用户空间读写流相关但不完全相同。缓冲区缓存是两者之间的中介。

换句话说，同一个存储驱动程序可以通过两种不同的方式到达。

1. **通过`/dev`的原始访问**：用户空间程序打开`/dev/myblk0`并发出`read(2)`或`write(2)`调用。这些调用通过`devfs`和GEOM字符接口，最终到达我们的策略函数。
2. **通过挂载的文件系统访问**：内核在设备上挂载文件系统。文件I/O流经VFS、文件系统、缓冲区缓存和GEOM。`devfs`不是这些请求的热路径的一部分。

两条路径在GEOM提供者处汇合，这就是为什么GEOM是存储驱动程序的正确抽象，尽管字符驱动程序通常更直接地与`devfs`打交道。

### 这种区分为何重要

这之所以重要有两个原因。

第一，它澄清了为什么我们不会为块驱动程序使用`make_dev`。`make_dev`是字符驱动程序想要在`/dev`下发布`cdevsw`时的正确调用。它对块设备来说是错误的调用，因为GEOM在我们发布提供者时会立即为我们创建`/dev`节点。如果你在存储驱动程序中调用`make_dev`，通常会得到两个竞争同一设备的`/dev`节点，其中一个未连接到GEOM拓扑，这会导致令人困惑的行为。

第二，这种区别解释了为什么内核有两套用于检查设备状态的工具。`devfs_ruleset(8)`、`devfs.rules`和每个节点的权限属于`devfs`。`geom(8)`、`gstat(8)`、`diskinfo(8)`和GEOM类树属于GEOM。当你诊断权限问题时，你查看`devfs`。当你诊断I/O问题时，你查看GEOM。

### 具体示例：/dev/null与/dev/ada0

比较你已经知道的两个示例。

`/dev/null`是一个经典的字符设备。它存在于`/dev`下是因为`devfs`创建了它。驱动程序是`null(4)`，其源码在`/usr/src/sys/dev/null/null.c`中。当你写入`/dev/null`时，`devfs`通过`cdevsw`将请求路由到`null`驱动程序的写入函数，该函数只是丢弃字节。没有GEOM，没有缓冲区缓存，没有文件系统。它是一个原始的`devfs`字符节点。

`/dev/ada0`是一个块设备。它也存在于`/dev`下。但该节点是由GEOM创建的，而不是由`ada`驱动程序中的直接`make_dev`调用创建的。当你从`/dev/ada0`读取原始字节时，这些字节通过GEOM的字符接口层流到`ada`驱动程序的策略函数中。当你在`/dev/ada0`上挂载UFS然后读取文件时，文件数据流经VFS、UFS、缓冲区缓存和GEOM，最终到达相同的策略函数，而不会为每个请求通过`devfs`。

`devfs`中的节点是相同的。使用模式不同。驱动程序必须处理两者。

### 我们将如何继续

我们不会在本章编写字符驱动程序。我们在第26章已经写过了。相反，我们将编写一个以磁盘形式向GEOM注册的驱动程序，并让GEOM为我们创建`/dev`节点。devfs集成将是自动的。

这是FreeBSD 14.3中块驱动程序的主流模式。你可以在`md(4)`、`ata(4)`、`nvme(4)`以及几乎每个其他存储驱动程序中看到它。它们每一个都向GEOM注册，每一个都接收`bio`请求，每一个都让GEOM处理`/dev`节点。

### 总结 Section 2

`devfs`和VFS是不同的层。`devfs`是挂载到`/dev`的文件系统，VFS是所有文件系统（包括`devfs`）插入的抽象框架。存储驱动程序与两者交互，但通过GEOM，GEOM负责创建`/dev`节点以及从原始访问路径和文件系统访问路径路由请求。在本章中，我们将使用GEOM作为入口点，让它代表我们管理`devfs`。

在下一节中，我们将开始构建驱动程序。我们将从向GEOM注册伪块设备所需的最少内容开始，暂不实现真正的I/O。一旦完成，我们将在后续小节中添加后备存储、`bio`处理程序和所有其他内容。

## 第3节：注册伪块设备

在本节中，我们将创建一个骨架驱动程序，向内核注册一个伪块设备。我们暂时不实现读取或写入。我们暂时不将其连接到后备存储。我们的目标更加温和但也更加重要：我们想确切了解让内核将我们的代码识别为存储驱动程序、为其发布`/dev`节点并让`geom(8)`等工具看到它需要什么。

一旦这个工作完成，我们之后添加的一切都将纯粹是增量的。注册本身是最令人感到神秘的步骤，也是驱动程序其余部分构建的基础。

### g_disk API

FreeBSD为存储驱动程序提供了一个称为`g_disk`的高级注册API。它位于`/usr/src/sys/geom/geom_disk.c`和`/usr/src/sys/geom/geom_disk.h`中。该API封装了较低层的GEOM类机制，暴露了更简单的接口，与磁盘驱动程序通常需要的相匹配。

使用`g_disk`使我们免于手工实现完整的`g_class`。使用`g_disk`时，我们分配一个`struct disk`，填充少量字段和回调指针，然后调用`disk_create`。该API负责构建GEOM类、创建geom、发布提供者、连接字符接口、启动devstat统计，以及使我们的设备通过`/dev`对用户空间可见。

并非每个存储驱动程序都使用`g_disk`。对其他提供者进行变换的GEOM类，如`g_nop`、`g_mirror`、`g_stripe`或`g_eli`，直接构建在较低层的`g_class`机制上，因为它们不是磁盘形状的。但对于任何看起来像磁盘的东西，当然也对于像我们这样的伪磁盘，`g_disk`是正确的起点。

你可以在`/usr/src/sys/geom/geom_disk.h`中看到公共结构。其形状大致如下，为清晰起见进行了缩略。

```c
struct disk {
    struct g_geom    *d_geom;
    struct devstat   *d_devstat;

    const char       *d_name;
    u_int            d_unit;

    disk_open_t      *d_open;
    disk_close_t     *d_close;
    disk_strategy_t  *d_strategy;
    disk_ioctl_t     *d_ioctl;
    disk_getattr_t   *d_getattr;
    disk_gone_t      *d_gone;

    u_int            d_sectorsize;
    off_t            d_mediasize;
    u_int            d_fwsectors;
    u_int            d_fwheads;
    u_int            d_maxsize;
    u_int            d_flags;

    void             *d_drv1;

    /* other fields elided */
};
```

这些字段分为三组。

**标识**：`d_name`是命名磁盘类的短字符串，如`"myblk"`，`d_unit`是区分多个实例的小整数。它们一起构成`/dev`节点名。`d_name = "myblk"`且`d_unit = 0`的驱动程序发布`/dev/myblk0`。

**回调**：`d_open`、`d_close`、`d_strategy`、`d_ioctl`、`d_getattr`和`d_gone`指针是内核将调用到我们驱动程序中的函数。其中只有`d_strategy`是严格要求的，因为它是处理实际I/O的函数。其他都是可选的，我们将在它们变得相关时讨论。

**几何参数**：`d_sectorsize`、`d_mediasize`、`d_fwsectors`、`d_fwheads`和`d_maxsize`描述磁盘的物理和逻辑形状。`d_sectorsize`是扇区大小（字节），通常为512或4096。`d_mediasize`是设备的总大小（字节）。`d_fwsectors`和`d_fwheads`是分区工具使用的建议性提示。`d_maxsize`是驱动程序可以接受的最大单个I/O，GEOM将使用它来拆分大请求。

**驱动程序状态**：`d_drv1`是驱动程序用于存放自己上下文的通用指针。它相当于Newbus世界中`device_get_softc(dev)`的最近等价物。

### 最小骨架

现在让我们勾画一个最小的骨架。我们将把它放在`examples/part-06/ch27-storage-vfs/myfirst_blk.c`中。这个初始版本几乎不做任何有用的事情。它注册一个磁盘，在每个操作上返回成功，并在卸载时干净地注销。但它足以出现在`/dev`中，在`geom disk list`中可见，并且可以被`newfs_ufs`或`fdisk`探测而不会导致内核崩溃。

```c
/*
 * myfirst_blk.c - a minimal pseudo block device driver.
 *
 * This driver registers a single pseudo disk called myblk0 with
 * the g_disk subsystem. It is intentionally not yet capable of
 * performing real I/O. Sections 4 and 5 of Chapter 27 will add
 * the BIO handler and the backing store.
 */

#include <sys/param.h>
#include <sys/systm.h>
#include <sys/kernel.h>
#include <sys/module.h>
#include <sys/malloc.h>
#include <sys/lock.h>
#include <sys/mutex.h>
#include <sys/bio.h>

#include <geom/geom.h>
#include <geom/geom_disk.h>

#define MYBLK_NAME       "myblk"
#define MYBLK_SECTOR     512
#define MYBLK_MEDIASIZE  (1024 * 1024)   /* 1 MiB to start */

struct myblk_softc {
    struct disk     *disk;
    struct mtx       lock;
    u_int            unit;
};

static MALLOC_DEFINE(M_MYBLK, "myblk", "myfirst_blk driver state");

static struct myblk_softc *myblk_unit0;

static void
myblk_strategy(struct bio *bp)
{

    /*
     * No real I/O yet. Mark every request as successful
     * but unimplemented so the caller does not hang.
     */
    bp->bio_error = ENXIO;
    bp->bio_flags |= BIO_ERROR;
    bp->bio_resid = bp->bio_bcount;
    biodone(bp);
}

static int
myblk_attach_unit(struct myblk_softc *sc)
{

    sc->disk = disk_alloc();
    sc->disk->d_name       = MYBLK_NAME;
    sc->disk->d_unit       = sc->unit;
    sc->disk->d_strategy   = myblk_strategy;
    sc->disk->d_sectorsize = MYBLK_SECTOR;
    sc->disk->d_mediasize  = MYBLK_MEDIASIZE;
    sc->disk->d_maxsize    = MAXPHYS;
    sc->disk->d_drv1       = sc;

    disk_create(sc->disk, DISK_VERSION);
    return (0);
}

static void
myblk_detach_unit(struct myblk_softc *sc)
{

    if (sc->disk != NULL) {
        disk_destroy(sc->disk);
        sc->disk = NULL;
    }
}

static int
myblk_loader(struct module *m, int what, void *arg)
{
    struct myblk_softc *sc;
    int error;

    switch (what) {
    case MOD_LOAD:
        sc = malloc(sizeof(*sc), M_MYBLK, M_WAITOK | M_ZERO);
        mtx_init(&sc->lock, "myblk lock", NULL, MTX_DEF);
        sc->unit = 0;
        error = myblk_attach_unit(sc);
        if (error != 0) {
            mtx_destroy(&sc->lock);
            free(sc, M_MYBLK);
            return (error);
        }
        myblk_unit0 = sc;
        printf("myblk: loaded, /dev/%s%u size=%jd bytes\n",
            MYBLK_NAME, sc->unit,
            (intmax_t)sc->disk->d_mediasize);
        return (0);

    case MOD_UNLOAD:
        sc = myblk_unit0;
        if (sc == NULL)
            return (0);
        myblk_detach_unit(sc);
        mtx_destroy(&sc->lock);
        free(sc, M_MYBLK);
        myblk_unit0 = NULL;
        printf("myblk: unloaded\n");
        return (0);

    default:
        return (EOPNOTSUPP);
    }
}

static moduledata_t myblk_mod = {
    "myblk",
    myblk_loader,
    NULL
};

DECLARE_MODULE(myblk, myblk_mod, SI_SUB_DRIVERS, SI_ORDER_MIDDLE);
MODULE_VERSION(myblk, 1);
```

花一点时间仔细阅读这段代码。只有少量的活动部件可见，但每一个都在做真正的工作。

`myblk_softc`结构是驱动程序的本地上下文。它保存了指向`struct disk`的指针、一个供未来使用的互斥锁和单元号。我们在模块加载时分配它，在卸载时释放它。

`myblk_strategy`函数是GEOM在`bio`指向我们的设备时将调用的回调函数。在这个第一个版本中，我们简单地对每个请求返回`ENXIO`失败。这不太礼貌，但作为占位符是正确的：内核不会阻塞等待我们，我们也不会在I/O未成功时假装成功了。在第5节中，我们将用可工作的处理程序替换它。

`myblk_attach_unit`函数分配一个`struct disk`，填充标识、回调和几何参数字段，然后用`disk_create`发布它。对`disk_create`的调用是实际产生`/dev`节点并在GEOM拓扑中注册磁盘的操作。

`myblk_detach_unit`函数逆转该过程。`disk_destroy`请求GEOM使提供者枯萎，取消任何待处理的I/O，并移除`/dev`节点。我们将`sc->disk`设置为`NULL`，以便后续的卸载尝试不会试图释放已经释放的结构，尽管在我们遵循的加载/卸载路径中这不可能发生。

模块加载器是你在第26章中看到的标准`moduledata_t`模板。在`MOD_LOAD`时，它分配softc并调用`myblk_attach_unit`。在`MOD_UNLOAD`时，它调用`myblk_detach_unit`，释放softc，然后返回。

有一行值得特别注意。

调用`disk_create(sc->disk, DISK_VERSION)`传递了磁盘结构的当前ABI版本。`DISK_VERSION`在`/usr/src/sys/geom/geom_disk.h`中定义，每当`g_disk` ABI不兼容地变更时递增。如果你针对错误的源码树编译驱动程序，内核将拒绝注册磁盘并打印诊断信息。这种版本控制机制允许内核在不静默破坏树外驱动程序的情况下演进。

你可能想知道为什么我们不使用`MODULE_DEPEND`来声明对`g_disk`的依赖。原因是`g_disk`在通常意义上不是一个可加载的内核模块。它是通过`/usr/src/sys/geom/geom_disk.c`中的`DECLARE_GEOM_CLASS(g_disk_class, g_disk)`在内核中声明的GEOM类，每当GEOM本身被编译进内核时它就存在。没有可以独立卸载或重新加载的单独`g_disk.ko`文件，`MODULE_DEPEND(myblk, g_disk, ...)`不会解析到一个真实的模块。我们调用的符号（`disk_alloc`、`disk_create`、`disk_destroy`）来自内核本身。

### Makefile

这个模块的Makefile与第26章中的几乎相同。

```make
# Makefile for myfirst_blk.
#
# Companion file for Chapter 27 of
# "FreeBSD Device Drivers: From First Steps to Kernel Mastery".

KMOD    = myblk
SRCS    = myfirst_blk.c

# Where the kernel build machinery lives.
.include <bsd.kmod.mk>
```

将此文件放在与`myfirst_blk.c`相同的目录中。运行`make`将构建`myblk.ko`。如果你将内核源码安装在常规位置，运行`make load`将加载它。运行`make unload`将卸载它。

### 加载和检查骨架

模块加载后，内核将创建一个伪磁盘和对应的`/dev`节点。让我们浏览一下你应该看到的内容。

```console
# kldload ./myblk.ko
# dmesg | tail -n 1
myblk: loaded, /dev/myblk0 size=1048576 bytes
# ls -l /dev/myblk0
crw-r-----  1 root  operator  0x8b Apr 19 18:04 /dev/myblk0
# diskinfo -v /dev/myblk0
/dev/myblk0
        512             # sectorsize
        1048576         # mediasize in bytes (1.0M)
        2048            # mediasize in sectors
        0               # stripesize
        0               # stripeoffset
        myblk0          # Disk ident.
```

权限字符串开头的`c`告诉我们GEOM创建了一个字符设备节点，这是现代内核在`/dev`下暴露面向块的设备的方式。设备主设备号，这里是`0x8b`，是动态分配的。

现在让我们看看GEOM拓扑。

```console
# geom disk list myblk0
Geom name: myblk0
Providers:
1. Name: myblk0
   Mediasize: 1048576 (1.0M)
   Sectorsize: 512
   Mode: r0w0e0
   descr: (null)
   ident: (null)
   rotationrate: unknown
   fwsectors: 0
   fwheads: 0
```

`Mode: r0w0e0`表示零个读取者、零个写入者、零个独占持有者。没有人正在使用磁盘。

现在尝试一些无害的操作。

```console
# dd if=/dev/myblk0 of=/dev/null bs=512 count=1
dd: /dev/myblk0: Device not configured
0+0 records in
0+0 records out
0 bytes transferred in 0.000123 secs (0 bytes/sec)
```

`Device not configured`错误是我们故意返回的`ENXIO`。我们的策略函数运行了，将BIO标记为失败，`dd`忠实地报告了失败。这是我们的驱动程序被内核块层代码到达的第一个真正证据。

尝试一个期望成功的读取来大声失败。

```console
# newfs_ufs /dev/myblk0
newfs: /dev/myblk0: read-only
# newfs_ufs -N /dev/myblk0
/dev/myblk0: 1.0MB (2048 sectors) block size 32768, fragment size 4096
        using 4 cylinder groups of 0.31MB, 10 blks, 40 inodes.
super-block backups (for fsck_ffs -b #) at:
192, 832, 1472, 2112
```

`-N`标志告诉`newfs`规划文件系统布局但不写入任何内容。我们可以看到它将我们的设备视为一个小型磁盘，有2048个512字节的扇区。这与我们声明的几何参数匹配。它实际上还没有写入任何内容，因为我们的策略函数仍然会失败，但规划是有效的。

最后，让我们干净地卸载模块。

```console
# kldunload myblk
# dmesg | tail -n 1
myblk: unloaded
# ls /dev/myblk0
ls: /dev/myblk0: No such file or directory
```

这就是骨架的完整生命周期。

### 为何这些失败是预期的

在这个阶段，任何实际尝试读取或写入数据的用户空间工具都会失败。这是正确的。我们的策略函数还不知道如何做任何事情，我们绝不能伪造成功。伪造成功会在文件系统试图读回它认为已经写入的内容时导致数据损坏。

内核和工具能够优雅地处理我们的失败这一事实证明块层正在正确地工作。一个`bio`传下来，驱动程序拒绝了它，错误传播回用户空间，没有人崩溃。这就是我们想要的行为。

### 各部分如何组合

在继续之前，让我们命名这些部件，以便我们以后可以毫无歧义地引用它们。

我们的**驱动程序模块**是`myblk.ko`。它是用户用`kldload`加载的东西。

我们的**softc**是`struct myblk_softc`。它保存驱动程序的本地状态。在第一个版本中恰好有一个实例。

我们的**磁盘**是由`disk_alloc`分配并用`disk_create`注册的`struct disk`。内核拥有它的内存。我们不直接释放它。我们通过调用`disk_destroy`请求内核释放它。

我们的**geom**是`g_disk`子系统代表我们创建的GEOM对象。我们在代码中不直接看到它。它作为我们提供者的父级存在于GEOM拓扑中。

我们的**提供者**是我们设备面向生产者的一面。它是其他GEOM类连接到我们时消费的东西。GEOM自动在`/dev`下为我们的提供者创建字符设备节点。

我们的**消费者**目前还是空的。还没有人连接到我们。消费者是位于我们之上的GEOM类（如分区层或文件系统的GEOM消费者）附加的方式。

我们的**/dev节点**是`/dev/myblk0`。它是一个活跃的句柄，用户空间工具可以使用它来发出原始I/O。当文件系统后来挂载到设备上时，它也会通过这个名称引用设备，即使热I/O路径不会为每个请求通过`devfs`。

### 总结 Section 3

我们构建了参与FreeBSD存储栈的最小可能驱动程序。它向`g_disk`子系统注册一个伪磁盘，通过GEOM发布`/dev`节点，接受BIO请求并礼貌地拒绝它们。它加载，它出现在`geom disk list`中，它卸载无泄漏。

在下一节中，我们将更直接地了解GEOM。我们将理解提供者到底是什么，消费者到底是什么，以及基于类的设计如何让分区、镜像、加密和压缩等变换与我们的驱动程序自由组合。这种理解将为我们进入第5节做好准备，在那里我们将用实际从后备存储提供读写的处理程序替换占位符策略函数。

## 第4节：暴露GEOM支持的提供者

上一节让我们用`g_disk`注册了一个磁盘，并接受了框架关于底层发生的事情的说法。这是一个合理的起点，对于许多驱动程序来说，这就是它们与GEOM的全部交互。但存储工作需要对所坐的层有深入的理解。当文件系统挂载失败时，当`gstat`显示请求堆积时，或者当`kldunload`阻塞的时间超过你的预期时，你会希望足够了解GEOM的词汇来提出正确的问题。

本节是从存储驱动程序角度对GEOM的导览。它不是详尽的参考。FreeBSD开发者手册中有整章专门介绍GEOM，我们不会重复。我们要做的是描述对驱动程序作者重要的概念和对象，并展示`g_disk`如何融入这幅图景。

### GEOM一页概览

GEOM是一个存储框架。它位于文件系统和与真实硬件对话的块驱动程序之间，按设计可组合。这种组合是其全部意义所在。

其理念是存储栈由小的变换构建而成。一个变换呈现原始磁盘。另一个变换将其拆分为分区。另一个变换将两个磁盘镜像为一个。另一个变换加密一个分区。另一个变换压缩一个文件系统。每个变换都是一小段代码，从上方接收I/O请求，对其进行处理，然后要么直接返回结果，要么将其传递给下一层。

在GEOM的词汇中，每个变换是一个**类**。类的每个实例是一个**geom**。每个geom有若干**提供者**（其输出）和若干**消费者**（其输入）。提供者面向上方的下一层。消费者面向下方的前一层。没有消费者的geom位于栈的底部：它必须自己产生I/O。没有提供者的geom位于栈的顶部：它必须终止I/O并将其传递到GEOM之外的某个地方，通常是文件系统或`devfs`字符设备。

请求从提供者通过栈流向消费者。回复沿相反方向流回。I/O的单位是`struct bio`，我们将在第5节详细研究。

### 组合的具体示例

假设你有一个1 TB的SATA SSD。内核的`ada(4)`驱动程序在SATA控制器上运行，发布一个名为`ada0`的磁盘提供者。那是一个底部没有消费者、顶部有一个提供者的geom。

你用`gpart`对SSD进行切片。`PART`类创建一个geom，其单个消费者附加到`ada0`，并发布多个提供者，每个分区一个：`ada0p1`、`ada0p2`、`ada0p3`等等。

你用`geli`加密`ada0p2`。`ELI`类创建一个geom，其单个消费者附加到`ada0p2`，并发布一个名为`ada0p2.eli`的提供者。

你在`ada0p2.eli`上挂载UFS。UFS打开该提供者，读取其超级块，并开始提供文件服务。

当进程读取文件时，请求从UFS出发，到`ada0p2.eli`，通过`geli` geom解密相关块，到`ada0p2`，通过`PART` geom偏移块地址，到`ada0`，`ada`驱动程序与SATA控制器通信。

UFS始终不知道其底层存储是加密的、分区的，甚至是一个物理磁盘。它只看到一个提供者。它下面的层可以简单也可以复杂，由管理员选择。

这种组合就是GEOM存在的原因。单个存储驱动程序只需要知道如何成为可靠的栈底I/O生产者。它上面的所有东西都是可重用的。

### 代码中的提供者和消费者

在内核中，提供者是`struct g_provider`，消费者是`struct g_consumer`。两者都在`/usr/src/sys/geom/geom.h`中定义。作为磁盘驱动程序作者，你几乎从不直接分配它们中的任何一个。`g_disk`在你调用`disk_create`时代表你分配一个提供者，而你从不需要消费者，因为磁盘驱动程序不附加到下面的任何东西。

你确实需要的是对它们含义的心智模型。

提供者是一个命名的、可寻址的、块可寻址的表面，可供读写。它有大小、扇区大小、名称和一些访问计数器。GEOM通过其字符设备集成在`/dev`中发布提供者，因此管理员可以通过名称引用它们。

消费者是从一个geom到另一个geom的提供者的通道。消费者是上层geom发出I/O请求的地方，也是上层geom注册访问权限的地方。当你在`ada0p2.eli`上挂载UFS时，挂载操作导致一个消费者被附加在UFS的GEOM钩子内，该消费者获取对`ada0p2.eli`提供者的访问权限。

### 访问权限

提供者有三个访问计数器：读取（`r`）、写入（`w`）和独占（`e`）。它们在`gstat`和`geom disk list`中可见，显示为`r0w0e0`或类似的格式。每个数字在消费者请求该类型的访问时递增，在消费者释放时递减。

独占访问是`mount`、`newfs`等管理工具在需要确保没有其他进程正在写入设备时获取的。独占计数为零意味着没有持有独占访问。独占计数大于零意味着提供者正忙。

访问计数不是琐碎的细节。它们是真正的同步工具。当你调用`disk_destroy`移除磁盘时，如果提供者仍有打开的用户，内核将拒绝销毁它，因为在已挂载的文件系统脚下销毁它将是灾难性的。这与`kldunload`在模块使用中时阻塞的机制相同，但它在GEOM层操作，比模块子系统高一级。

你可以实时观察访问计数器的变化。

```console
# geom disk list myblk0 | grep Mode
   Mode: r0w0e0
# dd if=/dev/myblk0 of=/dev/null bs=512 count=1 &
# geom disk list myblk0 | grep Mode
   Mode: r1w0e0
```

当`dd`完成时，模式返回到`r0w0e0`。

### BIO对象及其生命周期

GEOM中的工作单位是BIO，在`/usr/src/sys/sys/bio.h`中定义为`struct bio`。一个BIO代表一个I/O请求。它有一个命令（`bio_cmd`）、一个偏移量（`bio_offset`）、一个长度（`bio_length`）、一个数据指针（`bio_data`）、一个字节计数（`bio_bcount`）、一个剩余量（`bio_resid`）、一个错误（`bio_error`）、标志（`bio_flags`），以及我们将在需要时遇到的其他一些字段。

`bio_cmd`的值告诉驱动程序正在请求什么类型的I/O。最常见的值是`BIO_READ`、`BIO_WRITE`、`BIO_DELETE`、`BIO_GETATTR`和`BIO_FLUSH`。`BIO_READ`和`BIO_WRITE`如你所预期。`BIO_DELETE`要求驱动程序释放范围内的块，就像SSD上的`TRIM`或内存磁盘上的`mdconfig -d`所做的那样。`BIO_GETATTR`通过名称查询属性，是GEOM层发现分区类型、介质标签和其他元数据的方式。`BIO_FLUSH`要求驱动程序将未完成的写入提交到稳定存储。

BIO通过`g_io_request`从一个geom向下传递到下一个。当它到达栈底时，驱动程序的策略函数被调用。当驱动程序完成时，它通过调用`biodone`或在GEOM类级别调用`g_io_deliver`来完成BIO。完成调用将BIO沿栈向上释放。

`g_disk`驱动程序得到稍微简化的视图，因为`g_disk`基础设施将GEOM级别的BIO处理转换为`biodone`风格的完成方式。当你实现`d_strategy`时，你接收一个`struct bio`，最终必须调用`biodone(bp)`来完成它。你不直接调用`g_io_deliver`。框架来做。

### GEOM拓扑锁

GEOM有一个称为拓扑锁的全局锁。它保护geom、提供者和消费者树的修改。当提供者被创建或销毁时，当消费者被附加或分离时，当访问计数变化时，或当GEOM遍历树来路由请求时，都会获取拓扑锁。

拓扑锁在可能耗时的操作期间被持有，这对内核锁来说是不寻常的，所以GEOM通过称为事件队列的专用线程异步执行其大部分实际工作。当你在源码树中查看`g_class`定义时，`init`、`fini`、`access`和类似方法是在GEOM事件线程的上下文中调用的，而不是在触发操作的用户进程的上下文中。

对于使用`g_disk`的驱动程序，这在一个特定方面很重要。你不应该在调用GEOM级别函数时持有自己的驱动程序锁，因为GEOM可能在那些函数内部获取拓扑锁，错误的嵌套锁定顺序会导致死锁。`g_disk`编写得足够仔细，只要你遵循我们展示的模式，通常不需要考虑这一点。但这个事实值得了解。

### GEOM事件队列

GEOM在名为`g_event`的单一专用内核线程上处理许多事件。如果你在启用调试的情况下运行内核，你可以在`procstat -kk`中看到它。该线程从其队列中拾取事件并逐个处理。典型事件包括创建geom、销毁geom、附加消费者、分离消费者和重新品尝提供者。

一个实际的后果是，你从驱动程序采取的某些操作（如`disk_destroy`）不会在调用线程的上下文中同步发生。它们被排队等待事件线程处理，实际的销毁在稍后发生。`disk_destroy`正确处理了等待，所以在它返回时，磁盘已经消失了。但如果你在追踪一个微妙的排序bug，记住GEOM有自己的线程可能会有帮助。

### g_disk如何封装这一切

有了这些词汇，我们现在可以更精确地描述`g_disk`为我们做了什么。

当我们调用`disk_alloc`时，我们收到一个已经预初始化到足以填充的`struct disk`。我们设置名称、单元、回调和几何参数，然后调用`disk_create`。

`disk_create`通过事件队列为我们执行以下操作：

1. 如果该磁盘名称的GEOM类不存在，则创建一个，
2. 在该类下创建一个geom，
3. 创建与geom关联的提供者，
4. 设置devstat统计，使`iostat`和`gstat`有数据，
5. 连接GEOM的字符设备接口，使`/dev/<name><unit>`出现，
6. 安排BIO请求流入我们的`d_strategy`回调。

它还设置了一些可选行为。如果我们提供`d_ioctl`，内核将`/dev`节点上的用户空间`ioctl`调用路由到我们的函数。如果我们提供`d_getattr`，GEOM将`BIO_GETATTR`请求路由到它。如果我们提供`d_gone`，当我们驱动程序之外的东西决定磁盘已消失（如热插拔移除事件）时，内核会调用它。

在拆解侧，`disk_destroy`排队移除，等待所有待处理的I/O排干，释放提供者，销毁geom，并释放`struct disk`。我们不自己调用`free`释放磁盘。框架来做。

### 在哪里阅读源码

你现在有足够的词汇来直接从阅读`g_disk`源码中受益了。打开`/usr/src/sys/geom/geom_disk.c`并查找以下内容。

函数`disk_alloc`在文件的开头。它是一个简单的分配器，返回一个清零的`struct disk`。没什么戏剧性的。

函数`disk_create`更长。略读它并注意基于事件的方法：大部分实际工作是排队的，而不是内联执行的。同时注意对磁盘字段的健全性检查，它们能捕获忘记设置扇区大小、介质大小或策略函数的驱动程序。

函数`disk_destroy`同样是事件排队的。它用访问计数检查来保护拆解，因为销毁仍然打开的磁盘将是一个bug。

函数`g_disk_start`是内部策略函数。它验证BIO，更新devstat，并调用驱动程序的`d_strategy`。

花一点时间看看代码。你不需要理解每个分支。你确实需要识别整体形状：结构变更用事件，I/O用内联工作。这就是大多数基于GEOM的代码的形状。

### 比较md(4)和g_zero

两个真实的驱动程序是作为`g_disk`对照的良好阅读材料。第一个是`md(4)`驱动程序，在`/usr/src/sys/dev/md/md.c`中。这是一个内存磁盘驱动程序，同时使用`g_disk`和直接管理的GEOM结构。它是源码树中最全面的存储驱动程序示例，支持多种后备存储类型、调整大小、转储和许多其他功能。它是一个大文件，但它是我们正在构建的东西的最近亲。

第二个是`g_zero`，在`/usr/src/sys/geom/zero/g_zero.c`中。这是一个最小的GEOM类，读取总是返回清零的内存，写入被丢弃。它大约145行，直接使用较低层的`DECLARE_GEOM_CLASS` API而不是`g_disk`。它是一个很好的对照，因为它展示了没有任何磁盘特定装饰的GEOM类机制。当你想理解`g_disk`隐藏了什么时，阅读`g_zero`。

### 我们的驱动程序为何使用g_disk

你可能会问我们是否应该像`g_zero`那样直接在较低层的`g_class` API上构建驱动程序，以暴露更多机制。我们不会，有三个原因。

第一，`g_disk`是任何看起来像磁盘的东西的惯用选择，而我们的伪块设备就是如此。真正的FreeBSD驱动程序补丁的审查者会对在`g_disk`可用时直接使用`g_class`的驱动程序提出反对。

第二，`g_disk`免费为我们提供devstat集成、标准ioctl和`/dev`节点管理。手工重新实现这些将是对本章教学目标的重大干扰。

第三，第一个可工作的驱动程序越简单，就越容易推理。我们在接下来的几个小节中有很多代码要写。我们不需要在`g_disk`已经正确处理的类级别GEOM管道上花费篇幅。

话虽如此，如果你好奇，绝对应该阅读`g_zero.c`。它是一个小文件，揭示了`g_disk`抽象的机制。本节的总结将最后一次指向它。

### g_class详解

对于想要更多底层机制的读者，让我们走一遍`g_class`结构在代码中的样子，先不构建我们自己的。

以下内容（略有简化）摘自`/usr/src/sys/geom/zero/g_zero.c`。

```c
static struct g_class g_zero_class = {
    .name = G_ZERO_CLASS_NAME,
    .version = G_VERSION,
    .start = g_zero_start,
    .init = g_zero_init,
    .fini = g_zero_fini,
    .destroy_geom = g_zero_destroy_geom
};

DECLARE_GEOM_CLASS(g_zero_class, g_zero);
```

`.name`是类名，用于`geom -t`输出。`.version`必须与运行内核的`G_VERSION`匹配；版本不匹配在加载时被拒绝。`.start`是当BIO到达该类的提供者时调用的函数。`.init`在类首次实例化时调用，通常用于创建初始geom及其提供者。`.fini`是`.init`的拆解对应物。`.destroy_geom`在该类下的特定geom被移除时调用。

`DECLARE_GEOM_CLASS`是一个宏，展开为一个模块声明，在模块加载时将该类加载到内核中。它在单行后面隐藏了`moduledata_t`、`SYSINIT`和`g_modevent`的连接。

我们的驱动程序不直接使用`g_class`。`g_disk`为我们做了，它在底层声明的类是所有磁盘形状的驱动程序共享的通用`DISK`类。但理解这个结构是有用的，因为如果你以后要编写变换类（GEOM级别的加密、压缩或分区层），你将定义自己的`g_class`。

### BIO的生命周期详解

我们之前简要介绍了BIO的生命周期。这里更详细地说明，因为每个存储驱动程序的bug都会在某个时候触及这个生命周期。

BIO在驱动程序之上的某个地方产生。对于我们的驱动程序，最常见的来源是：

1. **文件系统的缓冲区缓存回写**。UFS在缓冲区上调用`bwrite`或`bawrite`，构建一个BIO并通过`g_io_request`将其交给GEOM。
2. **文件系统的缓冲区缓存读取**。UFS调用`bread`，检查缓存，在未命中时发出BIO。
3. **通过`/dev/myblk0`的原始访问**。程序在节点上调用`read(2)`或`write(2)`。`devfs`和GEOM的字符设备集成构建一个BIO并发出它。
4. **工具发出的操作**。`newfs_ufs`、`diskinfo`、`dd`和类似工具以与原始访问相同的方式发出BIO。

BIO构建后，通过GEOM的拓扑路由。沿途的每个消费者->提供者跳转可能会变换或验证BIO。对于简单的栈（我们的驱动程序没有中间geom），没有中间跳转；BIO到达我们的提供者并被分派到我们的策略函数。

在`g_disk`内部，策略函数之前有三个小的簿记步骤：

1. 一些健全性检查（例如，验证BIO的偏移量和长度在介质范围内）。
2. 调用`devstat_start_transaction_bio`开始为请求计时。
3. 调用驱动程序的`d_strategy`。

在完成时，`g_disk`拦截`biodone`调用，用`devstat_end_transaction_bio`记录结束时间，并将完成沿栈向上转发。

从驱动程序的角度看，唯一重要的是`d_strategy`被调用，并且每个BIO恰好调用一次`biodone`。其他一切都是管道。

### 错误传播

当BIO失败时，驱动程序将`bio_error`设置为一个`errno`值，并在`bio_flags`中设置`BIO_ERROR`标志。然后像平常一样调用`biodone`。

在驱动程序之上，GEOM的完成代码检查错误。如果设置了错误，错误沿栈向上传播。文件系统看到错误并决定怎么做；通常，元数据上的读取错误是致命的，文件系统向用户空间报告EIO。写入错误通常被延迟；文件系统可能重试，或者可能将关联的缓冲区标记为需要在下次同步时注意。

BIO路径中常见的`errno`值：

- `EIO`：通用I/O错误。内核假设设备遇到问题。
- `ENXIO`：设备未配置或已消失。
- `EOPNOTSUPP`：驱动程序不支持此操作。
- `EROFS`：介质是只读的。
- `ENOSPC`：没有可用空间。
- `EFAULT`：请求中的地址无效。在BIO路径中非常罕见。

对于我们的内存驱动程序，唯一应该出现的错误是边界检查错误（`EIO`）和未知命令错误（`EOPNOTSUPP`）。

### g_disk为你做的你看不到的事

我们提到过`g_disk`代表我们处理了几件事情。这里是更完整的列表。

- 如果`DISK`类型的GEOM类不存在，它会创建它，并在所有磁盘驱动程序之间共享该类。
- 当我们调用`disk_create`时，它在该类下创建一个geom。
- 它在geom上创建一个提供者并在`/dev`中发布。
- 它自动连接devstat统计。
- 它处理GEOM访问协议，将`/dev/myblk0`上的用户空间`open`和`close`调用转换为提供者访问计数变化。
- 它处理GEOM字符设备接口，将`/dev/myblk0`上的读写转换为到我们策略函数的BIO。
- 它处理BIO_GETATTR的默认情况（大多数属性有合理的默认值）。
- 它处理`disk_destroy`时的枯萎，等待飞行中的BIO。
- 它转发它自己不处理的ioctl的`d_ioctl`调用。

这些中的每一项都是如果你直接在`g_class`上构建就必须编写的代码。阅读`/usr/src/sys/geom/geom_disk.c`是感受`g_disk`为我们做了多少的好方法。

### 检查我们的提供者

让我们从第3节中取出我们的骨架驱动程序，加载它，并通过GEOM的眼睛检查它。

```console
# kldload ./myblk.ko
# geom disk list myblk0
Geom name: myblk0
Providers:
1. Name: myblk0
   Mediasize: 1048576 (1.0M)
   Sectorsize: 512
   Mode: r0w0e0
   descr: (null)
   ident: (null)
   rotationrate: unknown
   fwsectors: 0
   fwheads: 0
```

`geom disk list`只向我们显示`DISK`类的geom。这些geom每个都有一个提供者。我们还可以看到完整的类树。

```console
# geom -t | head -n 40
Geom        Class      Provider
ada0        DISK       ada0
 ada0p1     PART       ada0p1
 ada0p2     PART       ada0p2
 ada0p3     PART       ada0p3
myblk0      DISK       myblk0
```

我们的geom是真实磁盘的兄弟，还没有附加任何上层类。在后续小节中，我们将看到当文件系统附加时会发生什么。

```console
# geom stats myblk0
```

`geom stats`返回详细的性能计数器。在像我们这样空闲、未使用的设备上，所有计数器都是零。

```console
# gstat -I 1
dT: 1.002s  w: 1.000s
 L(q)  ops/s    r/s   kBps   ms/r    w/s   kBps   ms/w    %busy Name
    0      0      0      0    0.0      0      0    0.0    0.0| ada0
    0      0      0      0    0.0      0      0    0.0    0.0| myblk0
```

`gstat`是一个更紧凑的实时更新视图。我们将在后续小节中大量使用它。

### 总结 Section 4

GEOM是一个由类、geom、提供者和消费者组成的可组合块层框架。请求以`struct bio`对象流经它，带有`BIO_READ`、`BIO_WRITE`和少量其他命令。访问权限、拓扑锁定和事件驱动的结构管理是保持框架在负载下安全演进的机制。`g_disk`为磁盘形状的驱动程序封装了所有这些，给它们一个更友好的接口，几乎没有表达能力的损失。

我们的骨架驱动程序现在是一流的GEOM参与者，尽管它还不能做任何真正的I/O。在下一节中，我们将给它那个缺失的部分。我们将分配一个后备缓冲区，实现一个实际读写的策略函数，并观察内核的存储栈从原始访问和文件系统访问两个方向测试我们的代码。

## 第5节：实现基本读写

在第3节中，我们对每个BIO返回`ENXIO`。在第4节中，我们学到了足够多的关于GEOM的知识，确切地知道我们的策略函数接收什么样的请求以及它的义务是什么。在本节中，我们将用一个可工作的处理程序替换那个占位符，该处理程序针对内存后备存储读写真实的字节。到本节结束时，我们的驱动程序将通过`dd`服务流量，返回合理的数据，并在被`newfs_ufs`格式化后存活。

### 后备存储

我们目前的后备存储只是内核内存中的一个字节数组，大小与`d_mediasize`匹配。这是磁盘的最简单可能表示：一个扁平缓冲区。真正的存储驱动程序用硬件DMA、vnode支持的文件或交换支持的VM对象替换它，但扁平缓冲区足以无干扰地教授本章中的其他概念。

对于1 MiB，我们可以简单地用`malloc`分配缓冲区。对于更大的大小，我们需要不同的分配器，因为内核堆不能优雅地扩展到几十或几百兆字节的连续分配。`md(4)`通过使用逐页分配和自定义间接结构来避免大型内存磁盘的这个问题。我们目前还不需要那种复杂程度，但我们会在代码中注明这个限制。

让我们更新`myblk_softc`以包含后备存储。

```c
struct myblk_softc {
    struct disk     *disk;
    struct mtx       lock;
    u_int            unit;
    uint8_t         *backing;
    size_t           backing_size;
};
```

两个新字段：`backing`是我们分配的内核内存的指针，`backing_size`是我们分配的字节数。这些应该始终等于`d_mediasize`，但显式存储大小比通过`disk->d_mediasize`间接引用更干净。

现在，在`myblk_attach_unit`中，分配后备缓冲区。

```c
static int
myblk_attach_unit(struct myblk_softc *sc)
{

    sc->backing_size = MYBLK_MEDIASIZE;
    sc->backing = malloc(sc->backing_size, M_MYBLK, M_WAITOK | M_ZERO);

    sc->disk = disk_alloc();
    sc->disk->d_name       = MYBLK_NAME;
    sc->disk->d_unit       = sc->unit;
    sc->disk->d_strategy   = myblk_strategy;
    sc->disk->d_sectorsize = MYBLK_SECTOR;
    sc->disk->d_mediasize  = MYBLK_MEDIASIZE;
    sc->disk->d_maxsize    = MAXPHYS;
    sc->disk->d_drv1       = sc;

    disk_create(sc->disk, DISK_VERSION);
    return (0);
}
```

带有`M_WAITOK | M_ZERO`的`malloc`返回一个清零的缓冲区或睡眠直到有可用的。在健康的系统上，小分配不可能失败，这就是为什么我们不在这里检查返回值。如果我们要分配一个非常大的缓冲区，我们可能需要`M_NOWAIT`和显式错误处理，但对于1 MiB，`M_WAITOK`是惯用的选择。

`myblk_detach_unit`必须在销毁磁盘后释放后备存储。

```c
static void
myblk_detach_unit(struct myblk_softc *sc)
{

    if (sc->disk != NULL) {
        disk_destroy(sc->disk);
        sc->disk = NULL;
    }
    if (sc->backing != NULL) {
        free(sc->backing, M_MYBLK);
        sc->backing = NULL;
        sc->backing_size = 0;
    }
}
```

顺序很重要。我们首先销毁磁盘，这确保没有更多的BIO在飞行中。然后我们才释放后备缓冲区。如果我们先释放缓冲区，一个飞行中的BIO可能试图`memcpy`到或从一个不再引用我们内存的指针，内核将在下一次I/O时崩溃。

### 策略函数

现在是变更的核心。用实际服务BIO的函数替换占位符`myblk_strategy`。

```c
static void
myblk_strategy(struct bio *bp)
{
    struct myblk_softc *sc;
    off_t offset;
    size_t len;

    sc = bp->bio_disk->d_drv1;
    offset = bp->bio_offset;
    len = bp->bio_bcount;

    if (offset < 0 ||
        offset > sc->backing_size ||
        len > sc->backing_size - offset) {
        bp->bio_error = EIO;
        bp->bio_flags |= BIO_ERROR;
        bp->bio_resid = len;
        biodone(bp);
        return;
    }

    switch (bp->bio_cmd) {
    case BIO_READ:
        mtx_lock(&sc->lock);
        memcpy(bp->bio_data, sc->backing + offset, len);
        mtx_unlock(&sc->lock);
        bp->bio_resid = 0;
        break;

    case BIO_WRITE:
        mtx_lock(&sc->lock);
        memcpy(sc->backing + offset, bp->bio_data, len);
        mtx_unlock(&sc->lock);
        bp->bio_resid = 0;
        break;

    case BIO_DELETE:
        mtx_lock(&sc->lock);
        memset(sc->backing + offset, 0, len);
        mtx_unlock(&sc->lock);
        bp->bio_resid = 0;
        break;

    case BIO_FLUSH:
        /*
         * In-memory backing store is always "flushed".
         * Nothing to do.
         */
        bp->bio_resid = 0;
        break;

    default:
        bp->bio_error = EOPNOTSUPP;
        bp->bio_flags |= BIO_ERROR;
        bp->bio_resid = len;
        break;
    }

    biodone(bp);
}
```

让我们仔细阅读这段代码。它不是一个很长的函数，但每一行都在做一些重要的事情。

第一行找到我们的softc。GEOM在`bp->bio_disk`中给了我们一个指向磁盘的指针的BIO。我们在`disk_create`期间将softc存放在`d_drv1`中，所以我们从那里取回它。这是Newbus世界中`device_get_softc(dev)`在块驱动程序中的等价物。

第二对行提取请求的偏移量和长度。`bio_offset`是介质中的字节偏移量。`bio_bcount`是要传输的字节数。GEOM已经通过我们上面的任何层将文件级操作转换为线性字节范围。

随后的边界检查是防御性编程。GEOM通常不会向我们发送超过介质大小的请求，因为它代表我们拆分和验证BIO。但防御性驱动程序无论如何都会检查，因为静默接受的越界写入可能破坏内核内存，而且检查的成本是每个请求几条指令。我们还通过将明显的`offset + len > backing_size`检查重写为`len > backing_size - offset`来防止算术溢出，这不可能溢出，因为此时`offset <= backing_size`。

switch是真正工作发生的地方。每个BIO命令有自己的case。

`BIO_READ`从后备存储的`offset`处复制`len`字节到`bp->bio_data`。GEOM已经为我们分配了`bp->bio_data`，它将在BIO完成时释放。我们的工作只是填充它。

`BIO_WRITE`从`bp->bio_data`复制`len`字节到后备存储的`offset`处。与读取情况对称。

`BIO_DELETE`将范围清零。对于真实磁盘，`BIO_DELETE`是文件系统通知一定范围的块不再使用的方式，磁盘可以自由回收它。SSD用它来驱动TRIM。对于我们的内存驱动程序，没有什么可以回收的，但将范围清零是合理的响应，因为它反映了"数据已消失"的语义。

`BIO_FLUSH`是将未完成的写入提交到稳定存储的请求。我们的存储从不具有FLUSH会有帮助的那种易失性：每个`memcpy`已经按照发出的顺序对下一个`memcpy`可见。我们返回成功，无需做任何事情。

我们不认识的任何其他命令获得`EOPNOTSUPP`。我们上面的GEOM层将看到这个并相应地反应。

最后，`biodone(bp)`完成BIO。这不是可选的。每个进入策略函数的BIO必须恰好通过`biodone`一次离开，否则BIO将被泄漏，调用者将永远阻塞，你将很难诊断这个问题。

### bio_resid的作用

注意`bp->bio_resid`的处理。这个字段代表驱动程序完成后剩余要传输的字节数。当完整传输成功时，`bio_resid`为零。当传输完全失败时，`bio_resid`等于`bio_bcount`。当传输部分成功时，`bio_resid`是未成功传输的字节数。

我们的驱动程序要么传输全部，要么什么也不传输，所以我们将`bio_resid`设置为`0`（成功）或`len`（错误）。真正的硬件驱动程序可能在传输中途停止时将其设置为中间值。文件系统和用户空间工具使用`bio_resid`来计算实际移动了多少数据。

### 锁

我们在`memcpy`周围获取`sc->lock`。对于一次服务一个请求的内存驱动程序，锁并没有做太多可见的工作：内核的BIO调度使得真正的并发请求在我们的玩具设备上不太可能。但锁是良好的卫生习惯。GEOM不承诺你的策略函数会被串行调用，即使承诺了，将来为驱动程序添加异步工作线程的更改也无论如何都需要锁。现在添加比以后添加更便宜。

更复杂的驱动程序可能使用细粒度锁，或者可能使用依赖原子操作的MPSAFE方法。目前，`memcpy`周围的粗粒度互斥锁就可以了。它是正确的，容易推理，并且不会损害伪设备的性能。

### 重新构建和重新加载

更新源码并`kldunload`旧版本后，重新构建并重新加载。

```console
# make
cc -O2 -pipe -fno-strict-aliasing ...
# kldunload myblk
# kldload ./myblk.ko
# dmesg | tail -n 1
myblk: loaded, /dev/myblk0 size=1048576 bytes
```

现在让我们尝试一些真正的I/O。

```console
# dd if=/dev/zero of=/dev/myblk0 bs=4096 count=16
16+0 records in
16+0 records out
65536 bytes transferred in 0.001104 secs (59 MB/sec)
# dd if=/dev/myblk0 of=/dev/null bs=4096 count=16
16+0 records in
16+0 records out
65536 bytes transferred in 0.000512 secs (128 MB/sec)
```

我们写入了64 KiB的零并将其读回。你看到的速度取决于你的硬件和缓冲区缓存帮助了多少，但任何超过几MB/秒的速度对于第一次运行都是可以的。

```console
# dd if=/dev/random of=/dev/myblk0 bs=4096 count=16
16+0 records in
16+0 records out
65536 bytes transferred in 0.001233 secs (53 MB/sec)
# dd if=/dev/myblk0 of=pattern.bin bs=4096 count=16
16+0 records in
16+0 records out
# dd if=/dev/myblk0 of=pattern2.bin bs=4096 count=16
16+0 records in
16+0 records out
# cmp pattern.bin pattern2.bin
#
```

我们写入了随机数据，读回两次，确认两次读取返回相同的内容。我们的驱动程序现在是一个一致的存储。

### 负载下的快速观察

让我们运行一个短的压力测试并观察`gstat`。

在一个终端中：

```console
# while true; do dd if=/dev/urandom of=/dev/myblk0 bs=4096 \
    count=256 2>/dev/null; done
```

在另一个终端中：

```console
# gstat -I 1 -f myblk0
dT: 1.002s  w: 1.000s
 L(q)  ops/s    r/s   kBps   ms/r    w/s   kBps   ms/w    %busy Name
    0    251      0      0    0.0    251   1004    0.0    2.0| myblk0
```

大约每秒250次写入操作，每次4 KiB，大约1 MB/秒。延迟非常低，因为后备存储是RAM。对于真实磁盘，数字会非常不同，但你正在观察的结构是相同的。

在第一个终端上用`Ctrl-C`停止压力测试。

### 通过ioctl支持完善驱动程序

许多存储工具向设备发送ioctl来查询几何参数或发出命令。GEOM为我们处理常见的ioctl，但如果我们提供`d_ioctl`回调，内核会将未知的ioctl路由到我们的函数。目前我们不实现任何自定义ioctl。我们只注意到钩子存在。

```c
static int
myblk_ioctl(struct disk *d, u_long cmd, void *data, int flag,
    struct thread *td)
{

    (void)d; (void)data; (void)flag; (void)td;

    switch (cmd) {
    /* No custom ioctls yet. */
    default:
        return (ENOIOCTL);
    }
}
```

我们在调用`disk_create`之前通过赋值`sc->disk->d_ioctl = myblk_ioctl;`注册回调。从默认case返回`ENOIOCTL`告诉GEOM我们不处理该命令，并给它将请求传递给自己默认处理器的机会。

### 通过getattr支持完善驱动程序

GEOM使用`BIO_GETATTR`向存储设备请求命名属性。文件系统可能会询问`GEOM::rotation_rate`以了解它是否在旋转介质上。分区层可能会询问`GEOM::ident`以获取稳定标识符。`d_getattr`回调是让我们响应的钩子。

```c
static int
myblk_getattr(struct bio *bp)
{
    struct myblk_softc *sc;

    sc = bp->bio_disk->d_drv1;

    if (strcmp(bp->bio_attribute, "GEOM::ident") == 0) {
        if (bp->bio_length < sizeof("MYBLK0"))
            return (EFAULT);
        strlcpy(bp->bio_data, "MYBLK0", bp->bio_length);
        bp->bio_completed = strlen("MYBLK0") + 1;
        return (0);
    }

    /* Let g_disk fall back to default behaviour. */
    (void)sc;
    return (-1);
}
```

`d_getattr`的返回值约定值得停下来思考，因为它让许多首次阅读者绊倒。返回`0`并设置`bio_completed`告诉`g_disk`我们成功处理了属性。返回一个正的errno值（如太小的缓冲区的`EFAULT`）告诉`g_disk`我们处理了属性但操作失败了。返回`-1`告诉`g_disk`我们不认识该属性，它应该尝试其内建的默认处理器。这就是为什么我们在底部返回`-1`：我们希望`g_disk`代表我们回答标准属性如`GEOM::fwsectors`。对于我们的驱动程序，用短字符串响应`GEOM::ident`足以在`diskinfo -v`中显示。在`disk_create`之前用`sc->disk->d_getattr = myblk_getattr;`注册。

### 部分写入和短读取

我们的驱动程序实际上不会产生部分写入或短读取，因为后备存储在RAM中，每次传输要么完全成功要么完全失败。但对于真正的硬件驱动程序，部分传输是正常的：磁盘可能成功返回几个扇区然后在一个坏扇区上失败。BIO框架通过`bio_resid`支持这一点，驱动程序应该将`bio_resid`设置为未完成的字节数。

实际的指导是在调用`biodone`之前始终显式设置`bio_resid`。如果传输完全成功，将其设置为零。如果部分成功，设置为剩余量。如果完全失败，设置为`bio_bcount`。忘记设置`bio_resid`会在BIO分配时留下字段中的任何垃圾，这可能使调用者困惑。

### 常见错误 in Strategy Functions

在我们继续之前，让我们指出首次编写策略函数时出现的三个常见错误。

**忘记`biodone`。**策略函数的每条退出路径都必须对BIO调用`biodone(bp)`。如果你忘记了，BIO被泄漏，调用者挂起。这是"我的挂载挂起"问题最常见的单一来源。

**在`biodone`之间持锁。**`biodone`可能向上调用到GEOM或文件系统的完成处理器。这些处理器可能获取其他锁，或者可能需要获取你已经持有的锁，导致锁顺序反转和潜在死锁。最安全的模式是在调用`biodone`之前释放你的锁。我们的简单版本隐式地做到了这一点：`mtx_unlock`总是在switch内部，`biodone`在switch之后运行。

**从策略函数返回错误码。**`d_strategy`是一个`void`函数。错误通过在BIO上设置`bio_error`和`BIO_ERROR`标志来报告，而不是通过返回值。如果你正确声明了函数，编译器会捕获这个错误，但初学者有时会将其写为返回`int`，这会产生不应被忽略的编译器警告。

### Chained BIOs and BIO Hierarchies

BIO可以有一个子级。GEOM在变换类需要将请求拆分、组合或变换为一个或多个下游请求时使用它。例如，镜像类可能接收一个BIO_WRITE并发出两个子BIO，每个镜像成员一个。分区类可能接收一个BIO_READ并发出一个偏移量已移入底层提供者地址空间的单个子BIO。

父子关系记录在`bio_parent`中。当子级完成时，其错误由`biodone`传播到父级，`biodone`累积错误并在所有子级完成后交付父级。

我们的驱动程序不产生子BIO。它作为链的叶级接收它们。从驱动程序的角度看，每个BIO都是自包含的：它有偏移量、长度和数据缓冲区，我们的工作是服务它。

但如果你发现自己需要在驱动程序内部分割BIO（例如，如果请求跨越了后备存储以单独块处理的边界），你可以使用`g_clone_bio`创建子BIO，`g_io_request`分派它，`g_std_done`或自定义完成处理器来重组父级。该模式在内核中的多个地方可见，包括`g_mirror`和`g_raid`。

### 策略函数的线程上下文

策略函数在提交BIO的任何线程中运行。对于文件系统产生的BIO，通常是文件系统的同步线程或缓冲区缓存工作线程。对于直接的用户空间访问，是调用`/dev/myblk0`上`read`或`write`的用户线程。对于GEOM变换，可能是GEOM事件线程或类特定的工人线程。

这对你的驱动程序意味着`d_strategy`可以在许多不同的线程上下文中运行。你不能假设`curthread`属于任何特定进程，你不能长时间阻塞，否则调用的文件系统（或用户程序）将停滞。

如果你的策略函数需要做慢的事情（对vnode的I/O、等待硬件或复杂的锁定），正确的模式是将BIO排队到内部队列并让专门的工人线程处理它。这就是`md(4)`对所有后备类型所做的，因为vnode I/O（例如）可能任意长时间阻塞。

我们的驱动程序完全在内存中且只做`memcpy`，所以我们不需要工人线程。但理解这个模式对未来很重要。

### 实例演练：跨边界读取

假设一个文件系统发出一个偏移量100000、长度8192的BIO_READ。它跨越字节100000到108191。让我们追踪策略函数如何处理它。

1. `bp->bio_cmd`是`BIO_READ`。
2. `bp->bio_offset`是100000。
3. `bp->bio_bcount`是8192。
4. `bp->bio_data`指向一个内核缓冲区（或映射到内核的用户缓冲区），8192字节应该放入其中。

我们的代码计算`offset = 100000`和`len = 8192`。边界检查通过：`100000 + 8192 = 108192`，小于我们的`backing_size`32 MiB（33554432）。

switch进入`BIO_READ` case。我们获取锁，从`sc->backing + 100000`将8192字节`memcpy`到`bp->bio_data`中，然后释放锁。我们设置`bp->bio_resid = 0`表示完整传输。我们落入`biodone(bp)`，完成BIO。

文件系统收到完成通知，注意到错误为零，并使用这8192字节。读取完成。

现在假设偏移量是33554431，长度是2字节。即一个字节在后备存储内，一个字节超出末尾。

1. `offset = 33554431`。
2. `len = 2`。

边界检查：`offset > sc->backing_size`计算为`33554431 > 33554432`，为假。`len > sc->backing_size - offset`计算为`2 > 33554432 - 33554431`，即`2 > 1`，为真。检查失败，我们落入错误路径：设置`bio_error = EIO`，设置`BIO_ERROR`标志，设置`bio_resid = 2`，然后调用`biodone`。文件系统看到错误并处理它。

注意我们是如何使用减法来避免溢出风险的。如果我们写成`offset + len > sc->backing_size`，并且`offset`和`len`都接近`off_t`的最大值，加法可能回绕为一个小数字，检查会静默通过一个格式错误的请求。防御性边界检查总是重新排列算术以避免溢出。

### Devstat的副作用

使用`g_disk`的一个令人愉快的特性是devstat统计是自动的。我们服务的每个BIO都被`iostat`和`gstat`计数。不需要额外代码。

你可以在运行压力循环时在另一个终端中用`iostat -x 1`验证这一点。

```text
                        extended device statistics
device     r/s     w/s    kr/s    kw/s  ms/r  ms/w  ms/o  ms/t qlen  %b
ada0         0       2       0      48   0.0   0.1   0.0   0.1    0   0
myblk0       0     251       0    1004   0.0   0.0   0.0   0.0    0   2
```

如果我们的驱动程序构建在原始`g_class` API而不是`g_disk`上，我们将不得不自己连接devstat。这是`g_disk`免费给我们的小生活质量特性之一。

### 总结 Section 5

我们将占位符策略函数替换为可工作的处理程序。我们的驱动程序现在正确地对内存后备存储服务`BIO_READ`、`BIO_WRITE`、`BIO_DELETE`和`BIO_FLUSH`。它参与devstat，与`gstat`协作，并接受来自`dd`的真实流量。

在下一节中，我们将跨越从原始块访问到文件系统访问的边界。我们将用`newfs_ufs`格式化设备，挂载它，在其上创建文件，并观察当真正的文件系统位于提供者之上时请求路径如何变化。

## 第6节：在设备上挂载文件系统

到目前为止，我们的驱动程序通过原始访问进行测试：`dd`、`diskinfo`和类似工具将整个表面作为扁平字节范围进行读写。这是一种有价值的模式，但不是大多数存储设备生活的模式。现实生活中的存储设备为文件系统服务。本节将我们的驱动程序带到最后一英里：我们将格式化它，在上面挂载真正的文件系统，创建文件，并观察当文件系统出现时内核的块层管道如何路由请求。

这也是理论上的原始访问和文件系统访问之间的区别变得具体的第一节。理解这种差异并能够看到它在实践中运行，是存储驱动程序作者可以获得的最有用的洞察之一。

### 计划

我们将在本节中按顺序执行以下操作。

1. 将驱动程序的介质大小从1 MiB增加到足以容纳可用UFS文件系统的大小。
2. 构建并加载更新后的驱动程序。
3. 对设备运行`newfs_ufs`以创建文件系统。
4. 将文件系统挂载到临时目录上。
5. 创建一些文件并验证数据正确读回。
6. 卸载文件系统。
7. 重新加载模块并观察发生了什么。

到本节结束时，你将在自己的块驱动程序上看到一个完整的文件系统。

### 增加介质大小

UFS有一个最小的实际大小。你可以创建很小的UFS文件系统，但超级块、柱面组和inode表的开销在任何小于几兆字节的东西上都占据明显的空间比例。对于我们的目的，32 MiB是一个舒适的大小：它足够小，后备存储仍然适合普通的`malloc`，也足够大，UFS有呼吸的空间。

更新`myfirst_blk.c`顶部的尺寸定义。

```c
#define MYBLK_SECTOR     512
#define MYBLK_MEDIASIZE  (32 * 1024 * 1024)   /* 32 MiB */
```

重新构建。

```console
# make clean
# make
# kldunload myblk
# kldload ./myblk.ko
# diskinfo -v /dev/myblk0
/dev/myblk0
        512             # sectorsize
        33554432        # mediasize in bytes (32M)
        65536           # mediasize in sectors
        0               # stripesize
        0               # stripeoffset
```

32 MiB足够了。

### 使用newfs_ufs格式化

`newfs_ufs`是FreeBSD上的标准UFS格式化工具。它放置超级块、柱面组、根inode和UFS文件系统所需的所有其他结构。让我们在我们的设备上运行它。

```console
# newfs_ufs /dev/myblk0
/dev/myblk0: 32.0MB (65536 sectors) block size 32768, fragment size 4096
        using 4 cylinder groups of 8.00MB, 256 blks, 1280 inodes.
super-block backups (for fsck_ffs -b #) at:
192, 16576, 32960, 49344
```

幕后发生了几件事。

`newfs_ufs`为写入打开了`/dev/myblk0`，这导致GEOM访问计数增加。然后我们的策略函数收到了一连串写入：先是超级块，然后是柱面组，然后是空根目录，然后是几个备份超级块。每次写入都是一个BIO，每个BIO都由我们的驱动程序处理。

你可以通过读回几个字节来验证`newfs_ufs`确实写入了设备。

```console
# dd if=/dev/myblk0 bs=1 count=16 2>/dev/null | hexdump -C
00000000  00 00 00 00 00 00 00 00  00 00 00 00 00 00 00 00
```

UFS分区的前几个字节故意为零，因为超级块不在偏移零处：它在偏移65536（块128）处，以留出启动块和其他前导的空间。让我们看看那里。

```console
# dd if=/dev/myblk0 bs=512 count=2 skip=128 2>/dev/null | hexdump -C | head
00010000  00 00 00 00 00 00 00 00  00 00 00 00 00 00 00 00
00010010  80 00 00 00 80 00 00 00  a0 00 00 00 00 00 00 00
...
```

你现在应该看到非零字节了。那就是`newfs_ufs`放置在我们后备存储上的超级块。

### 挂载文件系统

创建一个挂载点并挂载文件系统。

```console
# mkdir -p /mnt/myblk
# mount /dev/myblk0 /mnt/myblk
# mount | grep myblk
/dev/myblk0 on /mnt/myblk (ufs, local)
# df -h /mnt/myblk
Filesystem    Size    Used   Avail Capacity  Mounted on
/dev/myblk0    31M    8.0K     28M     0%    /mnt/myblk
```

我们的伪设备现在是一个真正的文件系统了。观察GEOM访问计数。

```console
# geom disk list myblk0 | grep Mode
   Mode: r1w1e1
```

`r1w1e1`表示一个读取者、一个写入者、一个独占持有者。独占持有者是UFS：它已经告诉GEOM它是对设备写入的唯一权威，直到被卸载。

### 创建和读取文件

让我们实际使用文件系统。

```console
# echo "hello from myblk" > /mnt/myblk/hello.txt
# ls -l /mnt/myblk
total 4
-rw-r--r--  1 root  wheel  17 Apr 19 18:17 hello.txt
# cat /mnt/myblk/hello.txt
hello from myblk
```

注意刚才发生了什么。调用`echo "hello from myblk" > /mnt/myblk/hello.txt`通过系统调用层传到`sys_openat`，然后到VFS，然后到UFS，UFS打开根目录的inode，为`hello.txt`创建新inode，分配数据块，将17字节复制到缓冲区缓存中，并安排回写。缓冲区缓存最终向下调用GEOM，GEOM向下调用我们的策略函数，策略函数将那些字节复制到我们的后备存储中。

当你运行`cat`时，请求沿相同的栈向下传输。只不过，由于数据仍然在缓冲区缓存中（来自最近的写入），UFS实际上不需要从我们的设备读取。缓冲区缓存从RAM提供读取服务。如果你卸载并重新挂载，你会看到实际的读取。

```console
# umount /mnt/myblk
# mount /dev/myblk0 /mnt/myblk
# cat /mnt/myblk/hello.txt
hello from myblk
```

第二次`cat`可能确实导致了BIO_READ请求到达我们的驱动程序，因为卸载和重新挂载的循环使该文件系统的缓冲区缓存失效了。

### 观察流量

`gstat`实时显示BIO流量。打开另一个终端并运行`gstat -I 1 -f myblk0`。然后在第一个终端中，创建一个大文件。

```console
# dd if=/dev/zero of=/mnt/myblk/big bs=1m count=16
16+0 records in
16+0 records out
16777216 bytes transferred in 0.150 secs (112 MB/sec)
```

在`gstat`终端中，你应该看到写入的突发，可能分布在一两秒内，取决于缓冲区缓存刷新的速度。

```text
 L(q)  ops/s    r/s   kBps   ms/r    w/s   kBps   ms/w    %busy Name
    0    128      0      0    0.0    128  16384    0.0   12.0| myblk0
```

这些是UFS为填充文件而发出的4 KiB或32 KiB（取决于UFS的块大小）的写入。我们可以验证文件的存在。

```console
# ls -lh /mnt/myblk
total 16460
-rw-r--r--  1 root  wheel    16M Apr 19 18:19 big
-rw-r--r--  1 root  wheel    17B Apr 19 18:17 hello.txt
# du -ah /mnt/myblk
 16M    /mnt/myblk/big
4.5K    /mnt/myblk/hello.txt
 16M    /mnt/myblk
```

我们可以再次删除它来观察BIO_DELETE流量。

```console
# rm /mnt/myblk/big
```

UFS默认不发出`BIO_DELETE`，除非文件系统以`trim`选项挂载，所以在普通挂载上，你几乎不会在删除时看到BIO流量：UFS只是在自己的元数据中将块标记为空闲。要看到`BIO_DELETE`，我们需要以`-o trim`挂载，我们将在实验中简要介绍。

### 卸载

在卸载模块之前卸载文件系统。

```console
# umount /mnt/myblk
# geom disk list myblk0 | grep Mode
   Mode: r0w0e0
```

访问计数在UFS释放其独占持有时立即降回零。我们的驱动程序现在可以自由卸载或进一步操作。

### 在挂载状态下尝试卸载

如果你忘记`umount`并尝试卸载模块会怎样？

```console
# mount /dev/myblk0 /mnt/myblk
# kldunload myblk
kldunload: can't unload file: Device busy
```

内核拒绝了。`g_disk`子系统知道我们的提供者仍然有一个活跃的独占持有者，在持有者释放之前它不会让`disk_destroy`继续。这与我们在第26章中看到的保护活跃会话中USB串行设备的机制相同，只是提升到了GEOM层。

这是一个安全特性。在后备设备上挂载文件系统时卸载模块会导致内核在下一次BIO时崩溃：策略函数将不再存在，但UFS仍会尝试调用它。

先卸载，再卸载模块。

```console
# umount /mnt/myblk
# kldunload myblk
# kldstat | grep myblk
# 
```

干净。

### 我们的驱动程序上UFS的简要剖析

现在我们已经将UFS挂载到我们的设备上，值得停下来注意后备存储上实际有什么。UFS是一个文档完善的文件系统，在我们可以控制的设备上看到其结构就位是很有启发性的。

UFS文件系统的前65535字节保留给启动区。在我们的设备上，这些字节全为零，因为`newfs_ufs`默认不写入启动扇区。

偏移65536处是超级块。超级块是一个固定大小的结构，描述文件系统的几何参数：块大小、片段大小、柱面组数量、根inode的位置和许多其他不变量。`newfs_ufs`首先写入超级块，它还在可预测的偏移处写入备份副本，以防主副本损坏。

超级块之后是柱面组。每个柱面组保存inode、数据块和文件系统地址空间一部分的元数据。柱面组的数量和大小取决于文件系统大小。我们的32 MiB文件系统有四个柱面组，每个8 MiB。

每个柱面组内有inode块。每个inode是一个小结构（FreeBSD UFS2上为256字节），描述单个文件或目录：其类型、所有者、权限、时间戳、大小和数据的块地址。

最后，数据块本身保存文件内容。它们从柱面组的空闲块映射中分配。

当我们写入`"hello from myblk"`到`/mnt/myblk/hello.txt`时，内核大致执行了以下操作：

1. VFS请求UFS在根目录中创建新文件`hello.txt`。
2. UFS从根柱面组的inode表中分配了一个inode。
3. UFS更新根目录的inode以包含`hello.txt`的条目。
4. UFS为文件分配了一个数据块。
5. UFS将17字节的内容写入该数据块。
6. UFS将更新的inode写回。
7. UFS将更新的目录条目写回。
8. UFS更新了其内部簿记。

这些步骤中的每一个都转换为到我们驱动程序的一个或多个BIO。大多数是对元数据块的小写入。文件内容本身是一个BIO。UFS的Soft Updates特性对写入进行排序以确保崩溃一致性。

如果你想看到这些BIO的实际运行，可以在创建文件时运行实验7中的DTrace单行命令。你会在`echo`的时间附近看到一小阵写入。

### 挂载实际上是如何工作的

`mount(8)`命令是`mount(2)`系统调用的包装器。该系统调用接受一个文件系统类型、一个源设备和一个目标挂载点，然后请求内核执行挂载。

内核的响应是按类型找到相应的文件系统代码（UFS、ZFS、tmpfs等）并调用其挂载处理器，UFS的情况下是`/usr/src/sys/ufs/ffs/ffs_vfsops.c`中的`ufs_mount`。挂载处理器验证源设备，将其作为GEOM消费者打开，读取超级块，验证其格式正确，分配一个内存中的挂载结构，并将其安装到命名空间中。

从我们驱动程序的角度看，这些都不可见。我们看到的是一系列BIO：先是几次超级块读取，然后是UFS引导其内存状态所需的任何内容。挂载成功后，UFS在使用文件系统时按自己的时间表发出BIO。

如果挂载失败，UFS报告错误，内核的挂载代码进行清理。GEOM消费者被分离，访问计数下降，命名空间保持不变。我们的驱动程序不需要在挂载失败时做任何特殊处理。

### GEOM字符接口

在本章前面我们说通过`/dev/myblk0`的原始访问经过"GEOM的字符接口"。以下是更详细的含义。

GEOM为每个提供者发布一个字符设备。这与用`make_dev`创建的`cdev`不同；它是GEOM内部的一个专门路径，将提供者作为字符设备呈现给`devfs`。其代码位于`/usr/src/sys/geom/geom_dev.c`。

当用户程序打开`/dev/myblk0`时，`devfs`将`open`路由到GEOM的字符接口代码，该代码以请求的访问模式将一个消费者附加到我们的提供者。当程序写入时，GEOM的字符接口代码构建一个BIO并向我们的提供者发出，后者将其路由到我们的策略函数。当程序关闭文件描述符时，GEOM分离消费者，释放访问权限。

字符接口层在`struct uio`（用户空间I/O描述符）和`struct bio`（块层I/O描述符）之间转换。必要时，它将大型用户I/O拆分为多个BIO，尊重我们指定的`d_maxsize`。

所有这些对我们的驱动程序都是不可见的。我们只是接收BIO。但知道字符接口的存在有助于你理解为什么某些用户空间操作映射到某些BIO模式，以及为什么`d_maxsize`重要。

### 文件系统从块驱动程序中需要什么

现在我们已经在驱动程序上实际挂载了文件系统，我们可以更精确地描述文件系统从底层块驱动程序需要什么。

文件系统需要**正确的读写**。如果在偏移X处的写入后跟在偏移X处的读取，读取必须返回写入放在那里的内容，精度到扇区大小的粒度。我们通过`memcpy`进出后备存储来保证这一点。

文件系统需要**正确的边界**。块驱动程序不能接受超出介质大小的读取或写入。我们在策略函数中显式检查这一点。

文件系统需要**稳定的介质大小**。设备的大小在文件系统挂载后不能在文件系统脚下改变，因为文件系统元数据编码了假设固定大小的偏移和计数。我们的驱动程序保持介质大小不变。

文件系统需要**崩溃安全**，在底层存储提供的范围内。如果后备存储不丢失先前提交的写入，UFS可以从不干净的关机中恢复。我们的RAM后备驱动程序在重启时丢失所有内容，但至少在运行时是自洽的。在第7节中，我们将介绍持久性选项。

文件系统有时需要**刷新语义**。对`BIO_FLUSH`的调用应确保在返回之前所有先前发出的写入都是持久的。我们的RAM后备驱动程序平凡地满足这一点，因为它的路径中没有延迟回写。

最后，文件系统受益于**快速顺序访问**。这是一个服务质量问题而非正确性问题，但我们的驱动程序在这方面很好，因为`memcpy`很快。

### 原始访问与文件系统访问可视化

让我们将两条访问路径并排画出，以我们的实际驱动程序作为锚点。

```text
Raw access:                          Filesystem access:

  dd(1)                                cat(1)
   |                                    |
   v                                    v
  open("/dev/myblk0")                  open("/mnt/myblk/hello.txt")
   |                                    |
   v                                    v
  read(fd, ...)                        read(fd, ...)
   |                                    |
   v                                    v
  sys_read                             sys_read
   |                                    |
   v                                    v
  devfs                                VFS
   |                                    |
   v                                    v
  GEOM character                       UFS
  interface                            (VOP_READ, bmap)
   |                                    |
   |                                    v
   |                                   buffer cache
   |                                    |
   v                                    v
  GEOM topology  <--------------------- GEOM topology
   |                                    |
   v                                    v
  myblk_strategy (BIO_READ)            myblk_strategy (BIO_READ)
```

最后两跳是相同的。无论请求来自`dd`还是来自已挂载文件上的`cat`，我们的策略函数都以完全相同的方式被调用。这是位于块层的巨大优势：我们不需要区分两条路径。上层负责将文件级操作转换为块级操作，而我们只处理块。

### 使用DTrace观察请求路径

如果你想显式地看到请求路径，DTrace可以帮助。

```console
# dtrace -n 'fbt::myblk_strategy:entry { printf("cmd=%d off=%lld len=%u", \
    args[0]->bio_cmd, args[0]->bio_offset, args[0]->bio_bcount); }'
```

在探测运行时，在另一个终端中对已挂载的文件系统做一些操作并观察BIO的到来。你会看到读取以512字节到32 KiB的块到达，取决于UFS的块大小和你执行的操作。运行`dd if=/dev/zero of=/mnt/myblk/test bs=1m count=1`会产生一阵32 KiB的写入。

DTrace是FreeBSD提供的最强可观测性工具之一，它在存储工作中活跃起来是因为BIO路径被充分仪器化。我们将在后面的章节中更多地使用它，但即使像上面那样的单行命令也足以使抽象路径具体化。

### 总结 Section 6

我们的伪块设备现在扮演存储设备的完整角色：通过`dd`的原始访问，通过UFS的文件系统访问，以及与内核卸载保护的安全共存。我们在第5节编写的策略函数完全不需要改变就能让UFS工作，因为UFS和`dd`在它们下面共享相同的块层协议。

我们也看到了端到端的流程：VFS在顶部，UFS紧随其后，中间是缓冲区缓存，下面是GEOM，最底部是我们的驱动程序。这个流程对FreeBSD中的每个存储驱动程序都是相同的。你现在知道如何占据它的底部了。

在下一节中，我们将把注意力转向持久性。RAM后备设备便于测试，但在每次重新加载时都会丢失内容。我们将讨论使后备存储持久化的选项，每个选项带来什么权衡，以及如何将其中之一添加到我们的驱动程序中。

## 第7节：持久性与内存后备存储

我们的驱动程序在运行时是自洽的。如果你在偏移X处写入一个字节，稍后你可以在偏移X处读回它。如果你在已挂载的文件系统上创建一个文件，你可以再次读取它直到你卸载或卸载模块。这对于测试和短期工作负载已经很有用了。

然而，它不是持久的。卸载模块后，后备缓冲区被释放。重启机器后，每个字节都消失了。对于教学驱动程序来说，这可以说是一个特性：它重启后是干净的，不会跨运行累积状态，也不会静默损坏之前的会话。但理解使存储持久化的选项对于真正的驱动程序工作是至关重要的，因此本节遍历主要选择，然后展示如何向我们的驱动程序添加最简单的持久性。

### 持久性为何困难

存储持久性不仅仅是字节存储在哪里的问题。它涉及三个相互关联的属性。

**持久性**意味着一旦写入返回，数据在崩溃时是安全的。在硬件磁盘上，持久性通常与磁盘自身的缓存策略相关联：写入到达驱动器的内部缓冲区，然后是盘片，然后驱动器报告完成。`BIO_FLUSH`是给文件系统一种要求刷新到盘片语义的钩子。

**一致性**意味着在偏移X处的读取返回在偏移X处的最近写入，而不是某个更早或部分的版本。一致性通常由硬件或驱动程序中的仔细锁定提供。

**崩溃安全**意味着在不干净关机后，存储的状态是可用的。它要么反映所有提交的写入，要么反映它们的良好定义的前缀。UFS有SU+J（带日志的Soft Updates）来帮助从崩溃中恢复；ZFS使用写时复制和原子事务。所有这些都依赖于可预测地行为的块层。

对于教学驱动程序，我们不需要以完全的严谨性来解决所有三个问题。我们需要理解选择是什么，并选择一个适合我们目标的。

### 各种选择

有四种常见的伪块设备后备方式。

**内存后备（我们目前的选择）**。快速、简单，重新加载时丢失。实现为`malloc`分配的缓冲区。在几MiB以上扩展性差，因为它需要连续的内核内存。

**逐页内存后备**。`md(4)`在内部将其用于大型内存磁盘。不是一个大的缓冲区，驱动程序维护一个页大小分配的间接表并按需填充。这可以扩展到非常大的大小，避免在稀疏区域浪费内存，但更复杂。

**Vnode后备**。驱动程序在主机文件系统中打开一个文件并将其用作后备存储。`mdconfig -t vnode`是经典示例。读写经过主机的文件系统，以速度和对主机文件系统正确性的依赖为代价提供持久性。这是FreeBSD经常从嵌入内核的内存磁盘镜像启动的方式：内核加载镜像，将其呈现为`/dev/md0`，根文件系统在其上运行。

**交换后备**。驱动程序使用交换支持的VM对象作为后备存储。`mdconfig -t swap`使用这种方式。它只在交换是持久性的程度上提供跨重启的持久性，而在大多数系统上交换不是持久性的。但它提供了非常大的稀疏地址空间，而不在被触及之前消耗物理内存，这对于临时存储很有用。

在本章中，我们将坚持使用内存选项。它是最简单的，对实验足够了，并且干净地演示了每个其他概念。我们将讨论如何切换到vnode后备存储作为练习，并向那些想看到完整功能实现的人指出`md(4)`。

### 保存和恢复缓冲区

如果我们想让设备在重新加载后记住其内容，而不改变后备方式，我们可以在卸载时将缓冲区保存到文件中，在加载时恢复它。这不够优雅，但很直接，它清楚地说明了契约：驱动程序负责在第一个BIO到达之前将后备字节放入内存，并在最后一个BIO离开之前将它们刷新到安全的地方。

在我们的情况下，机制大致如下。

在模块加载时，在分配后备缓冲区之后但在调用`disk_create`之前，可选择将主机文件系统上的文件读入缓冲区。在模块卸载时，在`disk_destroy`完成后，可选择将缓冲区写回该文件。

从内核内部干净地执行此操作需要vnode API。内核提供`vn_open`、`vn_rdwr`和`vn_close`，它们共同让模块读取或写入主机文件系统中的路径。这些不是我们想随意使用的API，因为它们不是为驱动程序内部的高吞吐量I/O设计的，而且它们运行在挂载到该路径的任何文件系统上，这并不总是安全的。但对于加载和卸载时的一次性保存和恢复，它们是可以接受的。

出于教学目的，我们不会实现这个。持久化块设备内容的正确方式是使用真正的后备存储，而不是快照RAM缓冲区。但理解该技术有助于澄清契约。

### 与上层的契约

无论你的后备存储是什么，与上层的契约都是精确的。

**成功完成的BIO_WRITE必须对所有后续的BIO_READ请求可见**，无论中间有什么缓冲层。我们的内存驱动程序满足这一点，因为`memcpy`是可见的效果。

**成功完成的BIO_FLUSH必须使所有先前成功的BIO_WRITE请求变为持久的**。我们的内存驱动程序平凡地满足这一点，因为我们的`memcpy`和后备内存之间没有更低的层；在我们能提供的意义上，所有写入都是"持久的"。真正的磁盘驱动程序通常在响应`BIO_FLUSH`时向硬件发出缓存刷新命令。

**BIO_DELETE可以丢弃数据但不能损坏相邻块**。我们的内存驱动程序通过仅清零请求的范围来满足这一点。真正的SSD驱动程序可能为该范围发出TRIM；真正的HDD驱动程序通常没有DELETE的硬件支持，可以安全地忽略它。

**BIO_READ必须返回介质内容或错误；它不能返回未初始化的内存、来自不同事务的陈旧缓存数据或随机字节**。我们的内存驱动程序通过在分配时清零后备存储并仅通过策略函数写入来满足这一点。

如果你在设计新驱动程序时牢记这四条规则，你将避免困扰新存储驱动程序的几乎每一个正确性bug。

### md(4)的不同之处

内核的`md(4)`驱动程序是一个成熟的、多类型的内存磁盘驱动程序。它支持五种后备类型：malloc、preload、swap、vnode和null。每种类型都有自己的策略函数，知道如何为该后备类型服务请求。阅读`/usr/src/sys/dev/md/md.c`是本章有价值的后续，因为它展示了真正的驱动程序如何处理我们略过的所有情况。

`md(4)`做了一些我们没有做的具体事情。

`md(4)`为每个单元使用一个专用的工作线程。传入的BIO被排队在softc上，工作线程逐个出列并分派它们。这使得策略函数非常简单：只需入队和发信号。它还将阻塞工作隔离在工作线程中，这对vnode后备类型很重要，因为`vn_rdwr`可能阻塞。

`md(4)`一致使用`DEV_BSHIFT`（即`9`，表示512字节扇区），并使用整数算术而不是浮点来处理偏移。这是块层的标准做法。

`md(4)`有一个完整的ioctl配置接口。`mdconfig`工具通过`/dev/mdctl`上的ioctl与内核通信，驱动程序支持`MDIOCATTACH`、`MDIOCDETACH`、`MDIOCQUERY`和`MDIOCRESIZE`。我们没有实现任何类似的东西，因为对于我们的伪设备，配置是在编译时固定的。

`md(4)`使用`DISK_VERSION_06`，这是`g_disk` ABI的当前版本。我们的驱动程序通过`DISK_VERSION`宏做同样的事情。

如果你想看到一个生产质量的伪块设备，`md(4)`是规范的参考。我们正在构建的几乎所有东西，在真正的驱动程序中，最终都会随着时间推移而类似于`md(4)`的形状。

### 关于交换支持的内存的说明

有一种值得提及的技术，尽管我们不会在这里使用它，那就是交换支持的内存。不是用`malloc`分配的缓冲区，驱动程序可以分配一个`OBJT_SWAP`类型的VM对象并按需映射页面。页面由交换空间支持，这意味着它们可以在系统内存压力大时被换出，在被触及 时换入。这给你一个非常大的、稀疏的、按需的后备存储，热时行为像RAM，冷时行为像磁盘。

`md(4)`正是对其交换支持的内存磁盘使用了这种方法。交换VM对象作为后备存储，由内核的VM子系统为我们管理，驱动程序不需要预先分配连续的物理内存。`OBJT_SWAP`对象可以持有TB级的可寻址空间，而系统上只有GB级的RAM，因为该空间的大部分从未被触及。

如果你需要原型化一个大于几百MiB的块设备，交换支持的内存可能是正确的工具。它的VM API位于`/usr/src/sys/vm/swap_pager.c`。阅读它不是轻松的工作，但很有教育意义。

### 关于预加载镜像的说明

FreeBSD有一个称为**预加载模块**的机制。在启动期间，加载器不仅可以引入内核模块，还可以引入任意数据blob，这些通过`preload_fetch_addr`和`preload_fetch_size`对内核可用。`md(4)`使用它将预加载的文件系统镜像暴露为`/dev/md*`设备，这是FreeBSD可以完全从内存磁盘根启动的方式之一。

预加载镜像本身不是持久化机制。它们是与内核模块一起发布数据的一种方式。但它们经常用于嵌入式系统，其中根文件系统太珍贵而不能存储在可写存储上。

### 小扩展：仅在模块重新加载时持久化

我们不会向驱动程序添加真正的持久性，但现在是讨论后备存储在同一内核启动中存活模块卸载和重新加载实际需要什么的好时机。最天真的第一个想法（也是初学者很快会想到的）是将后备指针放在文件作用域的`static`变量中，并在卸载处理器中简单地不释放它。让我们看看为什么这不起作用以及什么才起作用。

考虑这个草拟：

```c
static uint8_t *myblk_persistent_backing;  /* wishful thinking */
static size_t   myblk_persistent_size;
```

直觉是，如果我们在首次附加时分配`myblk_persistent_backing`并拒绝在分离时释放它，随后的`kldload`将看到指针仍然设置并重用缓冲区。问题在于，这个图景忽略了KLD实际是如何加载和卸载的。当`kldunload`移除我们的模块时，内核回收模块的文本、数据和`.bss`段以及其映像的其余部分。我们的静态指针不会持久存在于某个稳定的位置；它与模块一起消失。当`kldload`随后带回模块时，内核分配一个新的`.bss`，将其清零，我们的指针从`NULL`开始。我们在上次附加时分配的`malloc`缓冲区仍然在内核堆中某处，但我们已经丢失了对它的每个句柄。我们泄漏了它。

`SYSUNINIT`也没有帮助，因为在KLD上下文中，它在`kldunload`时触发，而不是在某个稍后的"最终拆解"事件上。注册`SYSUNINIT`来释放缓冲区会在每次卸载时释放它，这正是我们不想要的。没有KLD级别的钩子意味着"模块文件真的、确实要从内存中永久移除"与普通的`kldunload`不同。

两种技术实际实现了跨卸载持久性，两者都被`md(4)`在生产中使用。第一种是**文件后备存储**。驱动程序不是分配内核堆缓冲区，而是使用vnode I/O API（`VOP_READ`、`VOP_WRITE`和通过`vn_open`获取的vnode引用）在现有文件系统上打开一个文件，并通过读取和写入该文件来服务BIO。卸载时，驱动程序关闭文件；下次加载时，它重新打开。持久性是真实的，因为它存在于状态独立于我们模块的文件系统中。这正是`md -t vnode -f /path/to/image.img`所做的，你可以在`/usr/src/sys/dev/md/md.c`中研究它。

第二种技术是**交换后备存储**。驱动程序分配一个`OBJT_SWAP`类型的VM对象，正如我们之前提到的，并按需从中映射页面。分页器位于比我们模块更高的内核级别，因此只要其他东西持有对它的引用，对象就可以比任何特定的`kldunload`存活更久。在实践中，`md(4)`将此用于交换支持的内存磁盘，它将对象的生命周期绑定到内核范围的列表而不是模块实例。

对于我们的教学驱动程序，我们不会实现任何一种技术。展示这个讨论的目的是确保你理解为什么明显的快捷方式不起作用，这样你就不会花一下午时间调试在`kldunload`后不断消失的缓冲区。如果你想实验真正的跨卸载持久性，仔细阅读`md.c`，特别是`mdstart_vnode`和`mdstart_swap`中的`MD_VNODE`和`MD_SWAP`分支，并注意后备对象是如何附加到每单元的`struct md_s`而不是模块范围的全局变量。这个结构选择正是让那些后端跨模块生命周期工作的原因。

### 草拟vnode后备的策略函数

为了让前面的讨论更具体，让我们草拟一个vnode后备的策略函数在代码级别是什么样子的。我们不会将这个放入教学驱动程序中。我们展示它是为了让你能看到"真正的"解决方案涉及什么，并在阅读`md.c`时能认出相同的形状。

其理念是，每单元的softc持有对vnode的引用，在附加时从管理员提供的路径获取。策略函数将每个BIO转换为正确偏移处的`vn_rdwr`调用，并根据结果完成BIO。

附加获取vnode：

```c
static int
myblk_vnode_attach(struct myblk_softc *sc, const char *path)
{
    struct nameidata nd;
    int flags, error;

    flags = FREAD | FWRITE;
    NDINIT(&nd, LOOKUP, FOLLOW, UIO_SYSSPACE, path);
    error = vn_open(&nd, &flags, 0, NULL);
    if (error != 0)
        return (error);
    NDFREE_PNBUF(&nd);
    VOP_UNLOCK(nd.ni_vp);
    sc->vp = nd.ni_vp;
    sc->vp_cred = curthread->td_ucred;
    crhold(sc->vp_cred);
    return (0);
}
```

`vn_open`查找路径并返回一个锁定的、已引用的vnode。然后我们释放锁，因为我们想在不阻塞其他操作的情况下持有引用，并将vnode指针挂在我们的softc上。我们还保留了用于后续I/O的凭据引用。

策略函数对vnode服务BIO：

```c
static void
myblk_vnode_strategy(struct bio *bp)
{
    struct myblk_softc *sc = bp->bio_disk->d_drv1;
    int error;

    switch (bp->bio_cmd) {
    case BIO_READ:
        error = vn_rdwr(UIO_READ, sc->vp, bp->bio_data,
            bp->bio_length, bp->bio_offset, UIO_SYSSPACE,
            IO_DIRECT, sc->vp_cred, NOCRED, NULL, curthread);
        break;
    case BIO_WRITE:
        error = vn_rdwr(UIO_WRITE, sc->vp, bp->bio_data,
            bp->bio_length, bp->bio_offset, UIO_SYSSPACE,
            IO_DIRECT | IO_SYNC, sc->vp_cred, NOCRED, NULL,
            curthread);
        break;
    case BIO_FLUSH:
        error = VOP_FSYNC(sc->vp, MNT_WAIT, curthread);
        break;
    case BIO_DELETE:
        /* Vnode-backed devices usually do not support punching
         * holes through BIO_DELETE without additional plumbing.
         */
        error = EOPNOTSUPP;
        break;
    default:
        error = EOPNOTSUPP;
        break;
    }

    if (error != 0) {
        bp->bio_error = error;
        bp->bio_flags |= BIO_ERROR;
        bp->bio_resid = bp->bio_bcount;
    } else {
        bp->bio_resid = 0;
    }
    biodone(bp);
}
```

注意switch的形状与我们RAM后备策略函数完全相同。唯一的区别是case分支做什么：不是对缓冲区进行`memcpy`，而是对vnode调用`vn_rdwr`。我们上面的框架，GEOM和缓冲区缓存，不知道也不关心我们选择了哪个后端。

分离释放vnode：

```c
static void
myblk_vnode_detach(struct myblk_softc *sc)
{

    if (sc->vp != NULL) {
        (void)vn_close(sc->vp, FREAD | FWRITE, sc->vp_cred,
            curthread);
        sc->vp = NULL;
    }
    if (sc->vp_cred != NULL) {
        crfree(sc->vp_cred);
        sc->vp_cred = NULL;
    }
}
```

`vn_close`释放vnode引用，如果这是最后一个引用，则允许vnode被回收。凭据以相同的方式引用计数。

为什么这给我们跨卸载持久性？因为我们关心的状态（即后备存储的内容）存在于真正文件系统上的文件中，其生命周期完全独立于我们的模块。当我们调用`kldunload`时，vnode引用被释放，文件关闭；其在磁盘上的内容由文件系统保留。当我们再次调用`kldload`并附加时，我们再次打开文件并从上次离开的地方继续。

剩余的微妙之处相当多。如果`vn_open`成功但后续的注册步骤失败，错误路径需要释放vnode。对`vn_rdwr`的调用可能睡眠，这意味着策略函数不能从不允许睡眠的上下文调用；在实践中，这就是为什么`md(4)`为vnode后备单元使用专用工作线程。读取文件可能与管理员修改它竞争，所以生产驱动程序通常采取措施检测并发的外部更改。`VOP_FSYNC`不是免费的，因此在刷新之前批量写入的快速路径是典型的。而且vnode生命周期本身受VFS自己的引用计数约束，这与其所含文件系统的卸载交互。

我们不会将此添加到教学驱动程序中，但当你阅读`/usr/src/sys/dev/md/md.c`中的`mdstart_vnode`时，你会认出这些问题中每一个都被仔细和明确地处理了。

### 总结 Section 7

持久性是一个分层概念。持久性、一致性和崩溃安全都是真正的存储设备必须提供的，不同的后备存储提供这些保证的不同子集。对于教学驱动程序，`malloc`分配的内存缓冲区是合理的选择，我们可以通过将缓冲区与每实例的softc分离来添加"存活模块重新加载"的语义，而不需要太多代码。

对于生产环境，技术变得更加精细：逐页分配、交换支持的VM对象、vnode支持的文件、专用工作线程、BIO_FLUSH协调和每个错误路径的仔细处理。`md(4)`是FreeBSD源码树中的规范示例，强烈建议阅读。

在下一节中，我们将详细关注拆解路径。我们将了解GEOM如何协调卸载、分离和清理；访问计数如何门控模块卸载路径；以及我们的驱动程序在拆解中途出现问题时应该如何表现。存储卸载bug是一些更棘手的内核bug类型，在这里的仔细关注将在你的驱动程序编写生涯的剩余时间里得到回报。

## 第8节：安全卸载与清理

存储驱动程序比字符驱动程序更加小心地处理其生命终结，因为风险更高。当字符驱动程序干净地卸载时，可能发生的最坏情况是一个打开的会话被拆除，可能有一些飞行中的字节丢失。当存储驱动程序在文件系统挂载在其上时卸载，可能发生的最坏情况是内核在下一次BIO时崩溃，用户留下一个在驱动程序消失时可能或可能未处于一致状态的文件系统镜像。

好消息是，如果你正确使用`g_disk`，内核的防御措施使灾难性情况几乎不可能发生。`kldunload`在GEOM访问计数非零时拒绝继续，这是我们在第6节中看到的主要安全网。但这不是唯一的关注点。本节详细遍历拆解路径，以便你知道该期望什么、该实现什么和该测试什么。

### 预期的拆卸序列

当用户想要移除存储驱动程序时，事件的名义序列如下。

1. 用户卸载挂载在设备上的每个文件系统。
2. 用户关闭任何以原始访问方式打开`/dev/myblk0`的程序。
3. 用户调用`kldunload`。
4. 模块卸载函数调用`disk_destroy`。
5. `disk_destroy`将提供者排队等待枯萎，枯萎在GEOM事件线程上运行。
6. 枯萎过程等待任何飞行中的BIO完成。
7. 提供者从GEOM拓扑中移除，`/dev`节点被销毁。
8. `disk_destroy`将控制返回给我们的卸载函数。
9. 我们的卸载函数释放softc和后备存储。
10. 内核卸载模块。

每个步骤都有自己的故障模式。让我们逐个遍历它们。

### Step 1: Un挂载

用户运行`umount /mnt/myblk`。VFS请求UFS刷新文件系统，这导致缓冲区缓存向GEOM发出任何待处理的写入，GEOM将它们路由到我们的驱动程序。我们的策略函数服务写入并调用`biodone`。缓冲区缓存报告成功；UFS处置其内存状态；VFS释放挂载点。UFS附加到我们提供者的消费者被分离。访问计数下降。

我们的驱动程序在此阶段不做任何特殊处理。我们继续按到达顺序处理BIO，直到UFS停止发出它们。

### 步骤2：关闭原始访问

用户确保没有程序持有打开的`/dev/myblk0`。如果有`dd`在运行，杀掉它。如果有shell通过`exec`打开了设备，关闭它。在释放每个打开句柄之前，访问计数将在`r`、`w`或`e`计数器中的至少一个上保持非零。

同样，我们的驱动程序不做任何特殊处理。对`/dev/myblk0`的`close(2)`调用通过`devfs`、通过GEOM的字符设备集成传播，并释放它们的访问权限。关闭不会发出BIO。

### 步骤3：kldunload

用户运行`kldunload myblk`。内核的模块子系统以`MOD_UNLOAD`调用我们的卸载函数。我们的卸载函数调用`myblk_detach_unit`，后者调用`disk_destroy`。

此时，我们的驱动程序即将停止存在。我们不能持有任何可能阻塞的锁，我们不能在我们自己的工作线程上阻塞（我们在本设计中没有任何工作线程），我们不能发出新的BIO。我们现在做的任何事情都不应该给内核带来新的工作。

### 步骤4：disk_destroy

`disk_destroy`是不可逆的点。阅读`/usr/src/sys/geom/geom_disk.c`中的源码发现它做三件事：

1. 它在磁盘上设置一个标志，表示销毁正在进行。
2. 它排队一个GEOM事件，该事件将实际拆解提供者。
3. 它等待事件完成。

在我们等待期间，GEOM事件线程拾取事件并遍历我们的geom。如果访问计数为零，事件继续。如果它们不为零，事件会以试图销毁仍有用户的磁盘的消息崩溃。

这就是步骤1和步骤2的重要性所在。如果你跳过它们并试图在文件系统挂载时卸载，崩溃就在这里发生。幸运的是，`g_disk`拒绝到达崩溃，因为模块子系统已经早些时候拒绝了卸载，但如果你绕过模块子系统并从其他上下文直接调用`disk_destroy`，这就是保护内核的检查。

### 步骤5到7：枯萎

GEOM枯萎过程是提供者从拓扑中移除的方式。它的工作方式是将提供者标记为枯萎、取消已排队但尚未交付的任何BIO、等待任何飞行中的BIO完成、从geom的提供者列表中移除提供者，然后从类中移除geom。`/dev`节点作为此过程的一部分被移除。

在枯萎期间，对于在枯萎开始之前飞行中的BIO，策略函数仍然可能被调用。我们的策略函数将正常处理它们，因为我们的驱动程序不知道也不关心枯萎正在进行。框架负责确保在不可逆点之后不会发出新的BIO。

如果我们的驱动程序有工作线程、队列或其他内部状态，我们需要小心地与枯萎协调。`md(4)`是一个这样做的驱动程序的好例子：它的工作线程监视关闭标志并在退出前排干其队列。由于我们的驱动程序完全是同步和单线程的，我们没有这个复杂性。

### 步骤8到9：释放资源

`disk_destroy` 返回后，磁盘已消失，提供者已消失，不会再有 BIO 到达。此时可以安全地释放后备存储并销毁互斥锁。

```c
static void
myblk_detach_unit(struct myblk_softc *sc)
{

    if (sc->disk != NULL) {
        disk_destroy(sc->disk);
        sc->disk = NULL;
    }
    if (sc->backing != NULL) {
        free(sc->backing, M_MYBLK);
        sc->backing = NULL;
        sc->backing_size = 0;
    }
}
```

然后我们的卸载函数销毁互斥锁并释放 softc。

```c
case MOD_UNLOAD:
    sc = myblk_unit0;
    if (sc == NULL)
        return (0);
    myblk_detach_unit(sc);
    mtx_destroy(&sc->lock);
    free(sc, M_MYBLK);
    myblk_unit0 = NULL;
    printf("myblk: unloaded\n");
    return (0);
```

### 步骤10：模块卸载

模块子系统卸载 `.ko` 文件。此时驱动程序已经消失。任何通过名称引用该模块的尝试都将失败，直到用户再次加载它。

### 可能出现的问题

正常路径是顺利的。让我们列举异常路径以及如何识别它们。

**`kldunload` 返回 `Device busy`**。文件系统仍然挂载着，或者某个程序仍然打开了原始设备。卸载并关闭，然后重试。这是最常见的失败，而且是良性的。

**`disk_destroy` 永不返回**。某个东西持有一个永远不会完成的 BIO，枯萎过程正在等待它。实际上，如果你的策略函数在某条路径上没有调用 `biodone`，就会发生这种情况。查看 `g_event` 线程的 `procstat -kk` 输出；如果它卡在 `g_waitfor_event` 中，你就有一个泄漏的 BIO。修复方法在策略函数中：确保每条路径恰好调用 `biodone` 一次。

**内核崩溃并显示 "g_disk: destroy with open count"**。你的驱动程序在提供者仍然有用户时调用了 `disk_destroy`。如果你只从模块卸载路径调用 `disk_destroy`，这不应该发生，因为模块子系统拒绝卸载繁忙的模块。但如果你响应其他事件调用 `disk_destroy`，你必须自己检查访问计数或容忍崩溃。

**内核崩溃并显示 "Freeing free memory"**。你的驱动程序试图释放 softc 或后备存储两次。检查你的分离路径是否存在竞争条件或提前退出后又继续执行到释放代码。

**内核崩溃并显示 "Page fault in kernel mode"**。某个东西正在解引用已释放的指针，最常见的是后备存储释放后仍有 BIO 在飞行中。修复方法是确保 `disk_destroy` 在释放策略函数触及的任何内容之前完成。

### d_gone回调

拆卸故事中还有一部分值得讨论。`d_gone` 回调在我们的驱动程序之外的其他东西决定磁盘应该消失时被调用。典型的例子是热插拔移除：用户拔出 USB 驱动器，USB 栈告诉存储驱动程序设备已消失，存储驱动程序希望尽可能优雅地告诉 GEOM 拆卸磁盘，即使 I/O 将开始失败。

我们的驱动程序是一个伪设备；它没有物理消失事件。但注册 `d_gone` 回调没有任何代价，并且使驱动程序在将来的扩展中更加健壮。

```c
static void
myblk_disk_gone(struct disk *dp)
{

    printf("myblk: disk_gone(%s%u)\n", dp->d_name, dp->d_unit);
}
```

在 `disk_create` 之前用 `sc->disk->d_gone = myblk_disk_gone;` 注册它。该函数在 `disk_gone` 被调用时由 `g_disk` 调用。你可以在开发过程中通过从测试路径调用 `disk_gone(sc->disk)` 来手动触发它；在伪驱动程序中你通常不会自己调用它。

注意 `disk_gone` 和 `disk_destroy` 之间的区别。`disk_gone` 表示"这个磁盘已经物理消失了；停止接受 I/O 并将提供者标记为返回错误"。`disk_destroy` 表示"从拓扑中移除这个磁盘并释放其资源"。在热拔出路径中，`disk_gone` 通常首先被调用（由总线驱动程序调用，当它注意到设备已消失时），`disk_destroy` 稍后被调用（由模块卸载或总线驱动程序的分离函数调用）。在两次调用之间，磁盘仍然存在于拓扑中，但所有 I/O 都会失败。我们的驱动程序不实现这种双阶段拆卸；例如 USB 大容量存储驱动程序则必须实现。

### 测试拆卸

拆卸 bug 通常不是通过仔细测试发现的，而是几个月后某个用户找到了触发它们的异常序列时偶然发现的。主动测试拆卸的成本要低得多。

以下是我建议在任何新存储驱动程序上运行的测试。

**基本卸载**。加载、格式化、挂载、卸载文件系统、卸载模块。验证 `dmesg` 显示我们的加载和卸载消息且没有其他内容。重复十次以捕获缓慢的泄漏。

**未卸载文件系统时的卸载**。加载、格式化、挂载。尝试卸载模块。验证卸载被拒绝。卸载文件系统，然后卸载模块。验证没有残留状态。

**负载下的卸载**。加载、格式化、挂载，启动 `dd if=/dev/urandom of=/mnt/myblk/stress bs=1m count=64`。在 `dd` 运行时，尝试卸载模块。验证卸载被拒绝。等待 `dd` 完成。卸载文件系统。卸载模块。验证清理干净。

**原始设备打开时的卸载**。加载。在另一个终端中，运行 `cat > /dev/myblk0` 以保持设备打开。尝试卸载模块。验证卸载被拒绝。终止 cat。卸载模块。验证清理干净。

**重载压力测试**。在紧凑循环中加载、卸载、加载、卸载一分钟。如果 `vmstat -m` 或 `zpool list` 开始显示泄漏，进行调查。

**损坏时的崩溃**。这个比较难：通过内核调试器钩子故意损坏模块状态，并验证驱动程序不会静默返回错误数据。实际上，很少有初学者这样做，教学驱动程序也不需要这样做。

如果所有这些都通过了，你就有了一个相当健壮的拆卸。每次更改涉及卸载路径的代码时都要继续测试。

### 幂等性原则

一个好的拆卸路径是幂等的：调用两次不会比调用一次更糟。这很重要，因为附加过程中的错误路径可能在所有内容设置完成之前就调用拆卸。

编写拆卸函数时，在尝试释放每个资源之前检查它是否确实被分配了。

```c
static void
myblk_detach_unit(struct myblk_softc *sc)
{

    if (sc == NULL)
        return;

    if (sc->disk != NULL) {
        disk_destroy(sc->disk);
        sc->disk = NULL;
    }
    if (sc->backing != NULL) {
        free(sc->backing, M_MYBLK);
        sc->backing = NULL;
        sc->backing_size = 0;
    }
}
```

释放指针后将它们设置为 `NULL` 是一个值得坚持的小习惯。它使双重释放错误在运行时变得明显（它们变成空操作而不是内存损坏），并且使拆卸函数幂等。

### 顺序与逆序

一个通用的拆卸准则：按分配的逆序释放资源。如果附加按 `A -> B -> C` 的顺序进行，分离应该按 `C -> B -> A` 的顺序进行。

在我们的驱动程序中，附加按 `malloc backing -> disk_alloc -> disk_create` 的顺序进行。所以分离按 `disk_destroy -> free backing` 的顺序进行。我们跳过释放磁盘，因为 `disk_destroy` 会替我们释放它。

这个模式是通用的。每个编写良好的拆卸函数都逆转分配顺序。当你看到一个分离函数按照与附加相同的顺序运行时，要怀疑存在 bug。

### MOD_QUIESCE事件

还有一个我们没有提到的第三个模块事件：`MOD_QUIESCE`。它在 `MOD_UNLOAD` 之前传递，给模块一个在驱动程序处于不安全卸载状态时拒绝卸载的机会。

对于大多数驱动程序，GEOM 访问计数检查已经足够，不需要实现 `MOD_QUIESCE`。但如果你的驱动程序有独立于 GEOM 的内部状态使卸载不安全（例如，必须刷新的缓存），`MOD_QUIESCE` 就是你通过返回错误来拒绝卸载的地方。

我们的驱动程序不实现 `MOD_QUIESCE`。默认行为是静默接受它，这对我们来说是正确的。

### 与未来工作线程的协调

如果你将来给驱动程序添加工作线程，拆卸契约就会改变。你必须：

1. 通知工作线程停止，通常通过在 softc 上设置一个标志。
2. 如果工作线程正在睡眠，唤醒它，通常用 `wakeup` 或 `cv_signal`。
3. 等待工作线程退出，通常通过 `kthread_exit` 可见的终止标志。
4. 然后才能调用 `disk_destroy`。
5. 释放 softc 和后备存储。

跳过这些步骤中的任何一个都会导致崩溃。常见的失败模式是工作线程在 softc 被释放后仍在访问 softc 状态的函数中睡眠。`md(4)` 仔细地处理了这个问题，如果你计划在自己的驱动程序中添加工作线程，值得阅读它的 worker 关闭代码。

### 错误情况下的清理

最后一个关注点：如果附加中途失败怎么办？假设 `disk_alloc` 成功了，但 `disk_create` 失败了。或者假设我们添加了验证扇区大小并在调用 `disk_create` 之前拒绝无效配置的代码。

处理这种情况的模式是"单一清理路径"。编写附加函数，使任何失败都跳转到一个清理标签，该标签按逆序展开到目前为止分配的所有内容。

```c
static int
myblk_attach_unit(struct myblk_softc *sc)
{
    int error;

    sc->backing_size = MYBLK_MEDIASIZE;
    sc->backing = malloc(sc->backing_size, M_MYBLK, M_WAITOK | M_ZERO);

    sc->disk = disk_alloc();
    sc->disk->d_name       = MYBLK_NAME;
    sc->disk->d_unit       = sc->unit;
    sc->disk->d_strategy   = myblk_strategy;
    sc->disk->d_ioctl      = myblk_ioctl;
    sc->disk->d_getattr    = myblk_getattr;
    sc->disk->d_gone       = myblk_disk_gone;
    sc->disk->d_sectorsize = MYBLK_SECTOR;
    sc->disk->d_mediasize  = MYBLK_MEDIASIZE;
    sc->disk->d_maxsize    = MAXPHYS;
    sc->disk->d_drv1       = sc;

    error = 0;  /* disk_create is void; no error path from it */
    disk_create(sc->disk, DISK_VERSION);
    return (error);

    /*
     * Future expansion: if we add a step that can fail between
     * disk_alloc and disk_create, use a cleanup label here.
     */
}
```

对于我们的驱动程序，`disk_alloc` 实际上不会失败（它使用 `M_WAITOK`），`disk_create` 是一个异步排队实际工作的 `void` 函数。所以附加路径实际上不会失败。但准备单一清理标签的模式对于变得更复杂的驱动程序来说是值得记住的。

### 总结 Section 8

存储驱动程序的安全卸载和清理归结为一小组准则：确保每个 BIO 都通过 `biodone` 完成、在完成回调期间不持有锁、只在提供者没有用户时才调用 `disk_destroy`、按分配的逆序释放资源、以及在负载下测试拆卸。`g_disk` 框架处理了大部分困难的部分；你的工作是避免破坏其不变量。

在下一节中，我们将从拆卸细节中退一步，讨论如何让存储驱动程序成长。我们将讨论重构、版本管理、如何干净地支持多个单元，以及当驱动程序变得超过单个源文件时该怎么做。这些是将教学驱动程序变成你可以长期持续演进的东西的习惯。

## 第9节：重构与版本管理

我们的驱动程序适合放在一个文件中，解决一个问题：它暴露一个由 RAM 支持的固定大小的伪磁盘。这是一个有用的教学起点，但不是大多数真实驱动程序所处的位置。一个真实的存储驱动程序会演进。它增长 ioctl 支持。它增长多单元支持。它增长可调参数。它拆分为多个源文件。它的磁盘上表示（如果有的话）经历格式变更。它积累了兼容性选择的历史。

本节是关于让驱动程序优雅成长的习惯。我们不会在这里添加大量新功能；配套的实验和挑战会做这些。我们要做的是调查任何存储驱动程序成熟时出现的重构和版本管理问题，并指出每种情况的 FreeBSD 惯用答案。

### 多单元支持

目前我们的驱动程序只支持一个实例，硬编码为 `myblk0`。如果你想要两个或三个伪磁盘，当前代码需要重复的 softc 和重复的磁盘注册。真实的驱动程序用可以容纳任意数量单元的数据结构来解决这个问题。

惯用的 FreeBSD 模式是一个由锁保护的全局列表。softc 按单元分配并链接到列表中。加载器时可调参数或 ioctl 驱动的调用决定何时创建新单元。单元号从 `unrhdr`（唯一编号范围）分配器分配。

A sketch:

```c
static struct mtx          myblk_list_lock;
static LIST_HEAD(, myblk_softc) myblk_list =
    LIST_HEAD_INITIALIZER(myblk_list);
static struct unrhdr      *myblk_unit_pool;

static int
myblk_create_unit(size_t mediasize, struct myblk_softc **scp)
{
    struct myblk_softc *sc;
    int unit;

    unit = alloc_unr(myblk_unit_pool);
    if (unit < 0)
        return (ENOMEM);

    sc = malloc(sizeof(*sc), M_MYBLK, M_WAITOK | M_ZERO);
    mtx_init(&sc->lock, "myblk unit", NULL, MTX_DEF);
    sc->unit = unit;
    sc->backing_size = mediasize;
    sc->backing = malloc(mediasize, M_MYBLK, M_WAITOK | M_ZERO);

    sc->disk = disk_alloc();
    sc->disk->d_name       = MYBLK_NAME;
    sc->disk->d_unit       = sc->unit;
    sc->disk->d_strategy   = myblk_strategy;
    sc->disk->d_sectorsize = MYBLK_SECTOR;
    sc->disk->d_mediasize  = mediasize;
    sc->disk->d_maxsize    = MAXPHYS;
    sc->disk->d_drv1       = sc;
    disk_create(sc->disk, DISK_VERSION);

    mtx_lock(&myblk_list_lock);
    LIST_INSERT_HEAD(&myblk_list, sc, link);
    mtx_unlock(&myblk_list_lock);

    *scp = sc;
    return (0);
}

static void
myblk_destroy_unit(struct myblk_softc *sc)
{

    mtx_lock(&myblk_list_lock);
    LIST_REMOVE(sc, link);
    mtx_unlock(&myblk_list_lock);

    disk_destroy(sc->disk);
    free(sc->backing, M_MYBLK);
    mtx_destroy(&sc->lock);
    free_unr(myblk_unit_pool, sc->unit);
    free(sc, M_MYBLK);
}
```

加载器一次性初始化单元池，然后各个单元可以独立创建和销毁。这与 `md(4)` 使用的模式非常接近。

我们还不会将本章的驱动程序重构为多单元，因为添加的代码会分散其他教学目标的注意力。但你应该知道这是驱动程序未来发展的方向。支持多个单元是真实驱动程序首先需要的扩展之一。

### 运行时配置的ioctl接口

有了多个单元，就需要在运行时配置它们。你不想每次想要第二个单元或不同大小时都编译一个新模块。答案是在控制设备上使用 ioctl。

`md(4)` 遵循这种模式。有一个单独的 `/dev/mdctl` 设备，`mdconfig(8)` 通过 ioctl 与它通信。`MDIOCATTACH` 创建一个具有指定大小和后备类型的新单元。`MDIOCDETACH` 销毁一个单元。`MDIOCQUERY` 读取单元的状态。`MDIOCRESIZE` 更改大小。

对于任何复杂度的驱动程序，这是值得投入的地方。通过宏进行编译时配置对玩具来说没问题。通过 ioctl 进行运行时配置是真正的管理员想要的。

如果你要将其添加到我们的驱动程序中，你会：

1. 使用 `make_dev` 为控制设备创建一个 `cdev`。
2. 在 cdev 上实现 `d_ioctl`，根据你定义的一小组 ioctl 号进行分发。
3. 编写一个发出 ioctl 的用户空间工具。

这是一个相当大的添加，这就是为什么我们在这里提到它但没有实现它。第28章及后续章节将重新讨论这种模式。

### 拆分源文件

在某个时候，驱动程序会超出单个文件的范围。FreeBSD 存储驱动程序通常的分解大致如下：

- `driver_name.c`：公共模块入口、ioctl 分发和附加/分离连接。
- `driver_name_bio.c`：策略函数和 BIO 路径。
- `driver_name_backing.c`：后备存储实现。
- `driver_name_util.c`：小型辅助函数、验证和调试打印。
- `driver_name.h`：声明 softc、枚举和函数原型的共享头文件。

Makefile 被更新以在 `SRCS` 中列出所有文件，构建系统处理其余部分。这是 `md(4)`、`ata(4)` 以及源代码树中大多数重要驱动程序的形态。

我们将为本章的驱动程序保持一个文件。但当挑战或你自己的扩展将其推过，比如说，500 行时，像上面那样的分解是正确的选择。想要具体示例的读者应该查看 `/usr/src/sys/dev/ata/`，它沿着清晰的界限将复杂的驱动程序拆分到多个文件中。

### 版本管理

存储驱动程序有几种版本管理需要关心。

**模块版本**，用 `MODULE_VERSION(myblk, 1)` 声明。这是一个单调递增的整数，其他模块或用户空间工具可以检查。每当你以无法从代码检测到的方式更改模块的外部行为时，就递增它。

**磁盘 ABI 版本**，编码在 `DISK_VERSION` 中。这是你的驱动程序编译时所针对的 `g_disk` 接口版本。如果内核的 `g_disk` 发生不兼容更改，它会递增版本，针对旧版本编译的驱动程序将注册失败。你不直接设置它；你通过 `disk_create` 传递 `DISK_VERSION` 宏，它会获取编译时在 `geom_disk.h` 中找到的任何版本。你应该针对你目标的内核重新编译驱动程序。

**磁盘上格式版本**，用于具有任何磁盘上元数据的驱动程序。如果你的驱动程序在保留扇区中写入魔术号和版本，你必须处理升级。我们的驱动程序没有磁盘上格式，所以这暂时不适用，但如果我们添加了适当的后备存储头，就需要了。

**Ioctl 号版本**。一旦你定义了 ioctl，它们的编号就是用户空间 ABI 的一部分。更改它们会破坏旧的用户空间工具。使用带有稳定魔术字母的 `_IO`、`_IOR`、`_IOW`、`_IOWR`，不要重新使用编号。

对于我们本章的驱动程序，目前我们关心的唯一版本是模块版本。但记住这四种版本管理可以避免以后的麻烦。

### 调试和可观测性辅助工具

随着驱动程序的增长，你会想比仅使用 `dmesg` 更丰富地观察其状态。有三个工具值得现在介绍。

**`sysctl` 节点**。FreeBSD 的 `sysctl(3)` 框架允许模块发布用户空间工具可以查询的只读或读写变量。你在选定的名称下创建一棵树并附加值到它。模式是标准的；大约十行代码就可以暴露已服务的 BIO 数量、读写的字节数和当前媒体大小。

```c
SYSCTL_NODE(_dev, OID_AUTO, myblk, CTLFLAG_RD, 0,
    "myblk driver parameters");
static u_long myblk_reads = 0;
SYSCTL_ULONG(_dev_myblk, OID_AUTO, reads, CTLFLAG_RD, &myblk_reads,
    0, "Number of BIO_READ requests serviced");
```

**Devstat**。我们已经通过 `g_disk` 使用了这个。它为 `iostat` 和 `gstat` 提供数据。不需要额外工作。

**DTrace 探针**。`SDT` 框架允许模块定义静态 DTrace 探针，当探针没有被监视时零开销。这些在 BIO 路径中特别有用，因为它们让你无需重新编译就能看到实时请求流。

```c
#include <sys/sdt.h>
SDT_PROVIDER_DECLARE(myblk);
SDT_PROBE_DEFINE3(myblk, , strategy, request,
    "int" /* cmd */, "off_t" /* offset */, "size_t" /* length */);

/* inside myblk_strategy: */
SDT_PROBE3(myblk, , strategy, request,
    bp->bio_cmd, bp->bio_offset, bp->bio_bcount);
```

然后你可以用 `dtrace -n 'myblk::strategy:request {...}'` 来监视。

对于本章的驱动程序，我们不会添加所有这些，但随着驱动程序的增长，这些是你应该使用的模式。

### 命名稳定性

一个容易被忽视的习惯：不要随意重命名东西。名称 `myblk` 出现在设备节点中、模块版本记录中、devstat 名称中，可能还出现在 sysctl 节点、DTrace 探针和文档中。重命名它会级联影响到所有这些。对于项目驱动程序，选择一个你可以永远使用的名称。`md`、`ada`、`nvd`、`zvol` 和其他存储驱动程序多年来一直保持它们的名称，因为重命名是对用户空间工具有 ABI 影响的更改。

### 保持教学驱动程序的简洁

本节中的所有内容都是你的驱动程序可能成长的方向。这些都不是本章教学驱动程序所必需的。我们指出这些方向，以便你在真实驱动程序源代码中看到它们时能够识别，并且当你扩展自己的驱动程序时不必从头发明这些模式。

配套的 `myfirst_blk.c` 在本章结束时仍是一个单文件。它的 README 记录了扩展点，挑战练习添加了其中一些。除此之外，你可以自由地继续扩展它，你做的每个扩展都将以某种形式使用这些模式。

### 设计模式简要回顾

到目前为止我们已经积累了足够多的模式，列出它们会有帮助。当你开始下一个存储驱动程序时，这些是你应该使用的模式。

**softc 模式。** 每个实例一个结构体来保存驱动程序需要的一切。由 `d_drv1` 指向。在回调内通过 `bp->bio_disk->d_drv1` 获取。

**附加/分离对。** 附加负责分配、初始化和注册。分离逆转这个序列。两者都必须是幂等的。

**switch-and-biodone 模式。** 每个策略函数根据 `bio_cmd` 进行分发，服务每个命令，设置 `bio_resid`，并恰好调用 `biodone` 一次。

**防御性边界检查。** 根据媒体大小验证偏移量和长度，使用减法避免溢出。

**粗粒度锁模式。** 热路径周围的一个互斥锁通常对教学驱动程序足够了。只在性能需要时才拆分它。

**逆序拆卸。** 按分配相反的顺序释放资源。

**释放后置空模式。** 释放指针后将其设置为 `NULL`。捕获双重释放。

**单一清理标签。** 在可能失败的附加函数中，所有失败都跳转到一个清理标签，展开到目前为止的状态。

**版本化 ABI。** 将 `DISK_VERSION` 传递给 `disk_create`。声明 `MODULE_VERSION`。对你依赖的每个内核模块使用 `MODULE_DEPEND`。

**延迟工作模式。** 必须阻塞的工作（如 vnode I/O）属于工作线程，而不是 `d_strategy`。

**可观测性优先习惯。** 在构建时添加 `printf`、`sysctl` 或 DTrace 探针。后期改造的可观测性比设计时就考虑的可观测性更难。

这些不是详尽无遗的，但它们是你最常使用的模式。每个模式都出现在我们驱动程序的某个地方，每个模式都出现在真实的 FreeBSD 存储代码中。

### 总结 Section 9

一个成熟的存储驱动程序以可预测的方向增长：多单元支持、通过 ioctl 进行运行时配置、多个源文件，以及它暴露的每个接口的稳定版本管理。这些都不必出现在第一个版本中。知道增长将发生在哪里可以让你做出以后不需要撤销的早期选择。

我们已经涵盖了本章要教授的每个概念。在动手实验之前，还有一个主题值得用专门的章节讨论，因为它作为驱动程序作者会回报你很多次：观察运行中的存储驱动程序。在下一节中，我们将看看 FreeBSD 为你提供的实时监视驱动程序和以有条理的方式测量其行为的工具。

## 第10节：可观测性与测量你的驱动程序

编写存储驱动程序主要是让结构正确的问题。一旦结构正确，驱动程序就会运行。但要保持结构正确，你必须能够在驱动程序运行时观察正在发生的事情。你会想知道每秒有多少BIO到达策略函数、每个花费多长时间、延迟分布如何、后备存储消耗了多少内存、是否有任何BIO正在被重试，以及是否有任何路径泄漏了完成。

FreeBSD为此提供了一套出色的工具，其中许多我们已经随意使用过。在本节中，我们将依次介绍最重要的工具，目标是让你在下一个奇怪的症状出现时能够轻松地拿起正确的工具。

### gstat

`gstat` 是首选工具。它实时更新每个提供者的 I/O 活动视图，向你展示 GEOM 层正在发生的具体情况。

```console
# gstat -I 1
dT: 1.002s  w: 1.000s
 L(q)  ops/s    r/s   kBps   ms/r    w/s   kBps   ms/w    %busy Name
    0    117      0      0    0.0    117    468    0.1    1.1| ada0
    0      0      0      0    0.0      0      0    0.0    0.0| myblk0
```

各列从左到右依次是：

- `L(q)`：队列长度。当前在此提供者上未完成的BIO数量。
- `ops/s`：每秒总操作数，无论方向。
- `r/s`：每秒读取数。
- `kBps`（读取）：以每秒千字节为单位的读取吞吐量。
- `ms/r`：平均读取延迟，以毫秒为单位。
- `w/s`：每秒写入数。
- `kBps`（写入）：以每秒千字节为单位的写入吞吐量。
- `ms/w`：平均写入延迟，以毫秒为单位。
- `%busy`：提供者非空闲的时间百分比。
- `Name`：提供者名称。

对于你刚构建的驱动程序，`gstat` 让你一眼就能看出内核是否正在向你的设备发送流量，以及你的驱动程序相对于真实磁盘的表现如何。如果数字与你期望的大不相同，你就有了调查的起点。

`gstat -p` 仅显示提供者（默认）。`gstat -c` 仅显示消费者，这对驱动程序调试不太有用。`gstat -f <regex>` 按名称过滤。`gstat -b` 逐屏批处理输出而不是原地刷新。

### iostat

`iostat` 具有更传统的风格，但提供相同的底层数据。当你想要文本日志而不是交互式显示时，它很有用。

```console
# iostat -x myblk0 1
                        extended device statistics
device     r/s     w/s    kr/s    kw/s  ms/r  ms/w  ms/o  ms/t qlen  %b
myblk0       0     128       0     512   0.0   0.1   0.0   0.1    0   2
myblk0       0     128       0     512   0.0   0.1   0.0   0.1    0   2
```

`iostat` 可以同时监视多个设备，并且可以重定向到日志文件以供后续分析。对于快速实时查看，`gstat` 通常更好。

### diskinfo

`diskinfo` 较少关注实时流量，更多关注静态属性。我们已经用它确认了我们的媒体大小。

```console
# diskinfo -v /dev/myblk0
/dev/myblk0
        512             # sectorsize
        33554432        # mediasize in bytes (32M)
        65536           # mediasize in sectors
        0               # stripesize
        0               # stripeoffset
        myblk0          # Disk ident.
```

`diskinfo -c` 运行计时测试，读取几百兆字节并报告持续速率。这对于一阶性能比较很有用。

```console
# diskinfo -c /dev/myblk0
/dev/myblk0
        512             # sectorsize
        33554432        # mediasize in bytes (32M)
        65536           # mediasize in sectors
        0               # stripesize
        0               # stripeoffset
        myblk0          # Disk ident.

I/O command overhead:
        time to read 10MB block      0.000234 sec       =    0.000 msec/sector
        time to read 20480 sectors   0.001189 sec       =    0.000 msec/sector
        calculated command overhead                     =    0.000 msec/sector

Seek times:
        Full stroke:      250 iter in   0.000080 sec =    0.000 msec
        Half stroke:      250 iter in   0.000085 sec =    0.000 msec
        Quarter stroke:   500 iter in   0.000172 sec =    0.000 msec
        Short forward:    400 iter in   0.000136 sec =    0.000 msec
        Short backward:   400 iter in   0.000137 sec =    0.000 msec
        Seq outer:       2048 iter in   0.000706 sec =    0.000 msec
        Seq inner:       2048 iter in   0.000701 sec =    0.000 msec

Transfer rates:
        outside:       102400 kbytes in  0.017823 sec =  5746 MB/sec
        inside:        102400 kbytes in  0.017684 sec =  5791 MB/sec
```

这些数字异常快，因为后备存储是 RAM。在真实磁盘上它们会看起来非常不同，跨设备比较数字通常是性能问题的第一步诊断。

### sysctl

`sysctl` 是内核将其内部变量暴露给用户空间的方式。许多子系统通过 `sysctl` 发布数据。你可以用以下命令浏览与存储相关的 sysctl：

```console
# sysctl -a | grep -i kern.geom
# sysctl -a | grep -i vfs
```

将你自己的 sysctl 树添加到驱动程序中，正如我们在第9节中讨论的，让你可以暴露驱动程序需要跟踪的任何指标，而无需定义新工具的繁文缛节。

### vmstat

`vmstat -m` 按 `MALLOC_DEFINE` 标签显示内存分配。我们的驱动程序使用 `M_MYBLK`，所以我们可以看到我们的驱动程序分配了多少内存。

```console
# vmstat -m | grep myblk
       myblk     1  32768K         -       12  32K,32M
```

各列是类型、分配数量、当前大小、保护请求、总请求和可能的大小。对于持有 32 MiB 后备存储的驱动程序，当前大小 32 MiB 正是我们期望的。如果它随时间增长而在卸载时没有相应的减少，我们就有了泄漏。

`vmstat -z` 显示区域分配器统计。许多与存储相关的状态存在于区域中（GEOM 提供者、BIO、磁盘结构），如果你怀疑 GEOM 级别的泄漏，`vmstat -z` 是查看的地方。

### procstat

`procstat` 显示每线程的内核栈。当某些东西卡住时，它是不可或缺的。

```console
# procstat -kk -t $(pgrep -x g_event)
  PID    TID COMM                TDNAME              KSTACK                       
    4 100038 geom                -                   mi_switch sleepq_switch ...
```

如果 `g_event` 线程正在睡眠，GEOM 层是空闲的。如果它卡在一个栈上有你的驱动程序名称的函数中，你就有一个没有完成的 BIO。

```console
# procstat -kk $(pgrep -x kldload)
```

如果 `kldload` 或 `kldunload` 卡住了，这会精确地显示在哪里。最常见的罪魁祸首是等待 BIO 排空的 `disk_destroy`。

### 块层的DTrace

我们在第6节和实验7中简要介绍了 DTrace。这里让我们深入一点，因为 DTrace 是理解实时存储行为的最有效工具。

函数边界跟踪（FBT）提供者允许你在几乎所有内核函数的入口和返回处放置探针。对于我们驱动程序的策略函数，探针名称是 `fbt::myblk_strategy:entry` 用于入口，`fbt::myblk_strategy:return` 用于返回。

一个按命令计数 BIO 的简单单行命令：

```console
# dtrace -n 'fbt::myblk_strategy:entry \
    { @c[args[0]->bio_cmd] = count(); }'
```

当你中断脚本（用 `Ctrl-C`）时，它按命令值打印计数。`BIO_READ` 是 1，`BIO_WRITE` 是 2，`BIO_DELETE` 是 3，`BIO_GETATTR` 是 4，`BIO_FLUSH` 是 5。（具体数字在 `/usr/src/sys/sys/bio.h` 中。）

延迟直方图：

```console
# dtrace -n '
fbt::myblk_strategy:entry { self->t = timestamp; }
fbt::myblk_strategy:return /self->t/ {
    @lat = quantize(timestamp - self->t);
    self->t = 0;
}'
```

这给你一个每次策略函数执行时间的对数刻度直方图。对于我们的内存驱动程序，大多数桶应该在数百纳秒范围内；对于内存驱动程序来说，毫秒范围内的任何东西都是可疑的。

I/O 大小的分布：

```console
# dtrace -n 'fbt::myblk_strategy:entry \
    { @sz = quantize(args[0]->bio_bcount); }'
```

这显示 BIO 大小的分布。对于 UFS 文件系统，你应该看到在 4 KiB、8 KiB、16 KiB 和 32 KiB 处有峰值。对于使用 `bs=1m` 的原始 `dd`，你应该看到在 1 MiB（或 `MAXPHYS` 上限，以较小者为准）处有峰值。

DTrace 的能力非凡。上面的单行命令只是冰山一角。如果你想深入了解，可以阅读 Sun 原始的 "DTrace Guide" 和 Brendan Gregg 的 "DTrace Book"。两者都比 FreeBSD 14.3 老，但基本原理仍然适用。

### kgdb和崩溃转储

当你的驱动程序崩溃时，FreeBSD 可以捕获崩溃转储。在 `/etc/rc.conf` 中配置转储设备（通常是 `dumpdev="AUTO"`）并用 `dumpon` 验证。

崩溃后，重新启动。`/var/crash/vmcore.last`（一个符号链接）指向最近的转储。`kgdb /boot/kernel/kernel /var/crash/vmcore.last` 打开转储以供检查。`kgdb` 中有用的命令：

- `bt`：崩溃线程的回溯。
- `info threads`：列出崩溃系统中的所有线程。
- `thread N` 然后 `bt`：线程 N 的回溯。
- `print *var`：检查变量。
- `list function`：显示函数周围的源代码。

如果你编译模块时包含了调试符号（大多数内核配置的默认值），`kgdb` 可以在你自己的代码中显示源代码级别的变量。一旦你习惯了，这是一种变革性的能力。

### ktrace

`ktrace` 是一个面向用户空间的工具，但当你想确切看到用户程序正在发出哪些系统调用时，它对存储调试很有用。如果 `newfs_ufs` 行为异常，你可以跟踪它：

```console
# ktrace -f /tmp/newfs.ktr newfs_ufs /dev/myblk0
# kdump /tmp/newfs.ktr | head -n 50
```

生成的跟踪显示系统调用的序列、参数和结果。对于存储工具，这精确地揭示了正在发出哪些 ioctl 以及正在打开哪些文件描述符。

### dmesg和内核日志

简单的 `dmesg` 通常是诊断问题最快的方式。我们的驱动程序在加载和卸载时打印到它。内核在许多其他事件中也打印到它，包括 GEOM 类创建、访问计数违规和系统恢复的崩溃。

专业提示：在每个实验会话开始时将 `dmesg -a` 重定向到文件。如果出了问题，你将有一个完整的日志。

```console
# dmesg -a > /tmp/session.log
# # ... work ...
# dmesg -a > /tmp/session-final.log
# diff /tmp/session.log /tmp/session-final.log
```

这给你一个精确的日志，记录内核在你的会话期间报告了什么。

### 简单的测量方法

以下是一个可以用来生成驱动程序一页性能概要的方法。

1. 加载驱动程序。
2. 运行 `diskinfo -c /dev/myblk0` 并记录三个传输速率数字。
3. 格式化设备并挂载它。
4. 在一个终端中，启动 `gstat -I 1 -f myblk0 -b` 并重定向到文件。
5. 在另一个终端中，运行 `dd if=/dev/zero of=/mnt/myblk/stress bs=1m count=128`。
6. `dd` 完成后停止 `gstat` 并保存日志。
7. 用 `awk` 解析日志以提取峰值 ops/s、峰值吞吐量和平均延迟。
8. 卸载文件系统并卸载模块。

这个方法可以扩展。对于真实驱动程序，你会自动化它，在块大小的矩阵上运行，并绘制结果。对于教学驱动程序，运行一两次就能让你对数字有感觉，并在未来的更改后有一个比较基准。

### 与md(4)的比较

最有用的练习之一是以与你的驱动程序相同的配置加载 `md(4)` 并进行比较。

```console
# mdconfig -a -t malloc -s 32m
md0
# diskinfo -c /dev/md0
```

数字可能在你驱动程序的一个小倍数范围内。如果它们非常不同，就有一些有趣的事情发生。通常的差异是：

- `md(4)` 使用工作线程接收来自策略函数的 BIO 并在单独的上下文中处理它们。这为每个 BIO 增加了一点延迟，但允许更高的并发性。
- `md(4)` 使用逐页后备，对于顺序 I/O 每字节稍慢，但可以扩展到更大的大小。
- `md(4)` 支持比我们的驱动程序更多的 BIO 命令和属性。

与 `md(4)` 比较是一种调试形式：如果你的驱动程序在相同工作负载上比 `md(4)` 慢得多或快得多，要么你做了一些不寻常的事情，要么你发现了一个值得理解的差异。

### 总结 Section 10

可观测性不是事后的想法。对于存储驱动程序，它是你保持方向的方式。`gstat`、`iostat`、`diskinfo`、`sysctl`、`vmstat`、`procstat` 和 DTrace 是你最常使用的工具。`kgdb` 和崩溃转储是当事情灾难性地出错时的后盾。

现在就学习这些工具，当驱动程序还简单的时候，因为当驱动程序变得复杂时，它们将是你使用的相同工具。一个能够观察运行中驱动程序的开发者比只能阅读源代码的开发者要有效得多。

我们现在已经涵盖了本章要教授的每个概念，加上可观测性和测量。在我们进入动手实验之前，让我们花一些时间阅读真实的 FreeBSD 源代码。接下来的案例研究将我们从源代码树中学到的一切锚定在实际代码中。

## 真实FreeBSD存储代码案例研究

阅读生产驱动程序源代码是内化模式最快的方式。在本节中，我们将遍历`/usr/src/sys/`中三个真实驱动程序的摘录，并附有注释指出每个摘录在做什么以及为什么。摘录故意简短；我们不会阅读每个驱动程序的每一行。我们会挑选重要的行。

打开文件并跟着文本一起阅读。关键是要让你看到我们驱动程序中的相同模式在不同的名称和不同的约束下重新出现在真实驱动程序中。

### 案例研究1：g_zero.c

`g_zero.c` 是源代码树中最简单的 GEOM 类。它是一个读取始终返回零、写入被丢弃的提供者，没有真正的后备存储，也没有真正的工作要做。它的目的是为你提供一个标准的"空磁盘"用于测试。它也是一个优秀的教学参考，因为它在不到 150 行中使用了完整的 `g_class` API。

让我们看看它的策略函数，叫做 `g_zero_start`。

```c
static void
g_zero_start(struct bio *bp)
{
    switch (bp->bio_cmd) {
    case BIO_READ:
        bzero(bp->bio_data, bp->bio_length);
        g_io_deliver(bp, 0);
        break;
    case BIO_WRITE:
        g_io_deliver(bp, 0);
        break;
    case BIO_GETATTR:
    default:
        g_io_deliver(bp, EOPNOTSUPP);
        break;
    }
}
```

三种行为，`BIO_GETATTR` 被故意折叠到 default 情况中。读取返回零。写入被静默接受。其他任何操作（包括属性查询）都得到 `EOPNOTSUPP`。真实的 `/usr/src/sys/geom/zero/g_zero.c` 还在成功写入路径中处理 `BIO_DELETE`；我们上面简化的摘录省略了该情况以便你能清楚地看到结构。注意调用的是 `g_io_deliver` 而不是 `biodone`。这是因为 `g_zero` 是一个类级别的 GEOM 模块，而不是 `g_disk` 模块。`g_io_deliver` 是类级别的完成调用；`biodone` 是 `g_disk` 的包装。

如果你并排重新阅读我们驱动程序的策略函数和这个，你会看到相同的结构：对 `bio_cmd` 的 switch、每个支持操作的一个 case、一个默认错误路径。我们的驱动程序有更多的 case 并且有真正的后备存储，但结构是相同的。

`g_zero` 向类注册的 `init` 函数也很小：

```c
static void
g_zero_init(struct g_class *mp)
{
    struct g_geom *gp;

    gp = g_new_geomf(mp, "gzero");
    gp->start = g_zero_start;
    gp->access = g_std_access;
    g_new_providerf(gp, "%s", gp->name);
    g_error_provider(g_provider_by_name(gp->name), 0);
}
```

当 `g_zero` 模块被加载时，这会运行。它在类下创建一个新的 geom，将 `start` 方法指向策略函数，使用标准访问处理器，并创建一个提供者。这就是暴露 `/dev/gzero` 所需的全部。

在我们的驱动程序中，当调用 `disk_create` 时，`g_disk` 做了所有这些的等价工作。你可以在这里再次看到 `g_disk` 抽象掉了什么。对于大多数磁盘驱动程序来说，这是一个好的交易；对于 `g_zero`，它不想要 `g_disk` 的磁盘特定功能，直接使用类 API 更合适。

### 案例研究2：md.c的malloc策略函数

`md(4)` 是一个具有多种后备类型的内存磁盘驱动程序。malloc 后备类型与我们的驱动程序最接近，其策略函数值得详细阅读。

以下是 `md(4)` 的工作线程为 `MD_MALLOC` 类型磁盘拾取 BIO 时发生情况的简化版本。（在真实的 `md(4)` 中，这是函数 `mdstart_malloc`。）

```c
static int
mdstart_malloc(struct md_s *sc, struct bio *bp)
{
    u_char *dst, *src;
    off_t offset;
    size_t resid, len;
    int error;

    error = 0;
    resid = bp->bio_length;
    offset = bp->bio_offset;

    switch (bp->bio_cmd) {
    case BIO_READ:
        /* find the page that contains offset */
        /* copy len bytes out of it */
        /* advance, repeat until resid == 0 */
        break;
    case BIO_WRITE:
        /* find the page that contains offset */
        /* allocate it if not allocated yet */
        /* copy len bytes into it */
        /* advance, repeat until resid == 0 */
        break;
    case BIO_DELETE:
        /* free pages in the range */
        break;
    }

    bp->bio_resid = 0;
    return (error);
}
```

与我们驱动程序的关键区别是逐页后备。`md(4)` 不分配一个大缓冲区。它按需分配 4 KiB 页面并通过 softc 内部的数据结构索引它们。好处是内存磁盘可以比单个连续 `malloc` 允许的大得多，稀疏区域（从未写入的）不消耗内存。

代价是每个 BIO 可能跨越多个页面，所以策略函数必须循环。每次迭代将 `len` 字节复制到当前页面，递减 `resid`，推进 `offset`，当 `resid` 达到零时退出或移动到下一个页面。

我们的驱动程序避免了这种复杂性，代价是只支持连续后备，这在几十兆字节以内没问题，但更远就不行了。

如果你想扩展我们的驱动程序以匹配 `md(4)` 的规模，逐页模式是你应该去的方向。一旦你有 `md(4)` 作为参考，这是很直接的。

### 案例研究3：md.c的模块加载路径

`md(4)` 另一个值得研究的部分是它如何引导其类并设置控制设备。

```c
static void
g_md_init(struct g_class *mp __unused)
{
    /*
     * Populate sc_list with pre-loaded memory disks
     * (preloaded kernel images, ramdisks from boot, etc.)
     */
    /* ... */

    /*
     * Create the control device /dev/mdctl.
     */
    status = make_dev_p(MAKEDEV_CHECKNAME | MAKEDEV_WAITOK,
        &status_dev, &mdctl_cdevsw, 0, UID_ROOT, GID_WHEEL,
        0600, MDCTL_NAME);
    /* ... */
}
```

`g_md_init` 函数在每次内核启动时运行一次，当 `md(4)` 类首次被实例化时。它处理加载器预加载到内存中的任何内存磁盘（以便内核可以从内存磁盘根启动），并创建控制设备 `/dev/mdctl`，`mdconfig` 稍后将通过它与驱动程序通信。

与我们的加载器比较，它是一个直接调用 `disk_create` 的简单 `moduledata_t`。`md(4)` 默认不创建任何内存磁盘。它只在响应预加载事件或控制设备上的 `MDIOCATTACH` ioctl 时创建它们。

这里的模式是可泛化的。如果你想要一个按需创建单元而不是在加载时创建的存储驱动程序，你会：

1. 注册类（或者对于基于 `g_disk` 的驱动程序，设置基础设施）。
2. 创建一个支持 ioctl 的带有 cdevsw 的控制设备。
3. 实现创建、销毁和查询 ioctl。
4. 编写一个与控制设备通信的用户空间工具。

`md(4)` 是典型的例子。其他驱动程序，如 `geli(4)` 和 `gmirror(4)`，使用略有不同的模式，因为它们是 GEOM 转换类而不是磁盘驱动程序，但整体结构相似。

### 案例研究4：真实存储驱动程序的 newbus 侧

作为对比，让我们简要看看真实硬件支持的存储驱动程序如何附加。例如 `ada(4)` 驱动程序是一个基于 CAM 的 ATA 驱动程序。它的附加路径不能直接作为单个函数看到，因为 CAM 在驱动程序和硬件之间进行中介，但链的末端看起来像这样（摘自 `/usr/src/sys/cam/ata/ata_da.c`）：

```c
static void
adaregister(struct cam_periph *periph, void *arg)
{
    struct ada_softc *softc;
    /* ... */

    softc->disk = disk_alloc();
    softc->disk->d_open = adaopen;
    softc->disk->d_close = adaclose;
    softc->disk->d_strategy = adastrategy;
    softc->disk->d_getattr = adagetattr;
    softc->disk->d_dump = adadump;
    softc->disk->d_gone = adadiskgonecb;
    softc->disk->d_name = "ada";
    softc->disk->d_drv1 = periph;
    /* ... */
    softc->disk->d_unit = periph->unit_number;
    /* ... */
    softc->disk->d_sectorsize = softc->params.secsize;
    softc->disk->d_mediasize = ...;
    /* ... */

    disk_create(softc->disk, DISK_VERSION);
    /* ... */
}
```

结构与我们完全相同：填充`struct disk`并调用`disk_create`。区别在于：

- `d_strategy`是`adastrategy`，它将BIO转换为ATA命令并通过CAM发给控制器。
- `d_dump`已实现，因为`ada(4)`支持内核崩溃转储。我们的驱动程序没有实现它。
- `d_sectorsize`和`d_mediasize`等字段来自硬件探测，而不是宏定义。

但从`g_disk`的角度看，`ada0`和我们的`myblk0`是同一种东西。两者都是磁盘。两者都接收BIO。两者都通过`biodone`完成。区别只在于字节实际去了哪里。

这就是`g_disk`提供的统一性。你的驱动程序可以选择任何后备技术，只要正确填充`struct disk`，它对内核的其余部分看起来就像任何其他磁盘一样。

### 案例研究要点

阅读这些摘录后，三个模式变得更加清晰。

第一，策略函数始终是对`bio_cmd`的switch。各个case不同，但switch始终存在。记住这个模式：传入BIO -> switch -> 每个命令一个case -> 完成。它是每个存储驱动程序的核心。

第二，`g_disk`驱动程序在注册层面结构完全相同。无论驱动程序是RAM磁盘还是真正的SATA驱动器，注册代码看起来都一样。区别在于BIO到达时发生了什么。

第三，更复杂的驱动程序将工作排队到专用线程。我们的驱动程序不会，因为它可以在任何线程中同步完成工作。执行慢速或阻塞工作的驱动程序必须排队，因为策略函数在调用者的线程上下文中运行。

掌握了这些模式，你现在可以阅读FreeBSD源码树中几乎任何存储驱动程序并理解其整体结构，即使有关硬件或子系统的具体细节需要进一步研究。

我们现在已经涵盖了本章要教授的每个概念，加上可观测性、测量和一些真实案例研究。在下一部分中，我们将通过动手实验将这些知识付诸实践。实验建立在你一直在编写的驱动程序和你一直在使用的技能之上，带你从最小可工作驱动程序经历持久性、挂载和清理场景。让我们开始吧。

## 动手实验

每个实验都是一个自包含的检查点。它们按顺序设计，但你可以在以后重新访问任何实验来练习特定技能。每个实验在`examples/part-06/ch27-storage-vfs/`下都有配套文件夹，其中包含参考实现和你手动输入代码时会产生的工作产物。

开始之前，确保本章的驱动程序能针对你的本地内核干净地构建。从示例树的全新检出开始：

```console
# cd examples/part-06/ch27-storage-vfs
# make
# ls myblk.ko
myblk.ko
```

如果这成功了，你就准备好了。如果没有，重新检查Makefile和第26章"你的构建环境"一节中的建议。

### 实验1：在运行中的系统上探索GEOM

**目标。**在接触任何代码之前建立对GEOM检查工具的熟悉度。

**你做什么。**

在你的FreeBSD 14.3系统上，运行以下命令并在实验日志中做笔记。

```console
# geom disk list
# geom part show
# geom -t | head -n 40
# gstat -I 1
# diskinfo -v /dev/ada0   # or whatever your primary disk is called
```

**你观察什么。**

识别每个`DISK`类的geom。对每个，记录其提供者名称、媒体大小、扇区大小和当前模式。注意哪些geom上面有分区层，哪些没有。如果你的系统有`geli`或`zfs`，注意类的链条。

**延伸问题。**你哪些geom目前有非零访问计数？哪些是空闲的？如果你尝试在每个上面运行`newfs_ufs`会发生什么？

**参考实现。**`examples/part-06/ch27-storage-vfs/lab01-explore-geom/README.md`包含建议的演练和典型系统的示例输出记录。

### 实验2：构建骨架驱动程序

**目标。**让第3节的骨架驱动程序在你的系统上编译和加载。

**你做什么。**

将`examples/part-06/ch27-storage-vfs/lab02-skeleton/myfirst_blk.c`及其`Makefile`复制到工作目录。构建它。

```console
# cp -r examples/part-06/ch27-storage-vfs/lab02-skeleton /tmp/myblk
# cd /tmp/myblk
# make
```

Load the module.

```console
# kldload ./myblk.ko
# dmesg | tail -n 2
# ls /dev/myblk0
# geom disk list myblk0
```

Unload it.

```console
# kldunload myblk
# ls /dev/myblk0
```

**你观察什么。**

确认内核打印了你的`myblk: loaded`消息。确认`/dev/myblk0`出现了。确认`geom disk list`报告了预期的媒体大小。确认卸载后节点消失了。

**延伸问题。**如果你在骨架驱动程序上尝试`newfs_ufs -N /dev/myblk0`会发生什么？你能读懂输出吗？为什么干运行会成功，即使真正的写入会失败？

### 实验3：实现BIO处理器

**目标。**将第5节的可工作策略函数添加到骨架驱动程序中。

**你做什么。**

从骨架开始，实现`myblk_strategy`，支持`BIO_READ`、`BIO_WRITE`、`BIO_DELETE`和`BIO_FLUSH`。在`myblk_attach_unit`中分配后备缓冲区，在`myblk_detach_unit`中释放它。

构建、加载和测试。

```console
# dd if=/dev/zero of=/dev/myblk0 bs=4096 count=16
# dd if=/dev/myblk0 of=/dev/null bs=4096 count=16
# dd if=/dev/random of=/dev/myblk0 bs=4096 count=16
# dd if=/dev/myblk0 of=/tmp/a bs=4096 count=16
# dd if=/dev/myblk0 of=/tmp/b bs=4096 count=16
# cmp /tmp/a /tmp/b
```

**你观察什么。**

最后的`cmp`必须成功且没有输出。如果它打印`differ: byte N`，你的策略函数存在竞争或返回陈旧数据。

**延伸问题。**在策略函数中放一个`printf`，报告`bio_cmd`、`bio_offset`和`bio_bcount`。运行`dd if=/dev/myblk0 of=/dev/null bs=1m count=1`并查看`dmesg`。`dd`实际发出了什么大小？你看到分片了吗？

**参考实现。**`examples/part-06/ch27-storage-vfs/lab03-bio-handler/myfirst_blk.c`。

### 实验4：增加大小并挂载UFS

**目标。**将后备存储增加到32 MiB并在设备上挂载UFS。

**你做什么。**

将`MYBLK_MEDIASIZE`改为`(32 * 1024 * 1024)`并重新构建。加载模块。格式化并挂载。

```console
# newfs_ufs /dev/myblk0
# mkdir -p /mnt/myblk
# mount /dev/myblk0 /mnt/myblk
# echo "hello" > /mnt/myblk/greeting.txt
# cat /mnt/myblk/greeting.txt
# umount /mnt/myblk
# mount /dev/myblk0 /mnt/myblk
# cat /mnt/myblk/greeting.txt
# umount /mnt/myblk
# kldunload myblk
```

**你观察什么。**

验证文件在卸载和重新挂载后仍然存在。验证`geom disk list`中的访问计数在卸载后为零。验证`kldunload`干净地成功。

**延伸问题。**在运行`dd if=/dev/zero of=/mnt/myblk/big bs=1m count=16`时观察`gstat -I 1`。你能看到写入以突发方式到达吗？单个BIO的大小是多少？提示：UFS在这个大小的文件系统上默认块大小通常是32 KiB。

**参考实现。**`examples/part-06/ch27-storage-vfs/lab04-mount-ufs/myfirst_blk.c`。

### 实验5：用md(4)观察真正的跨重载持久性

**目标。**通过使用`md(4)`的vnode模式作为对照，实验性地确认跨重载持久性需要外部后备，正如第7节所论证的。

**你做什么。**

首先，演示我们的RAM后备`myblk`在重载时会丢失其文件系统。加载、格式化、挂载、写入、卸载、卸载模块、重新加载、再次挂载，观察空的文件系统。

```console
# kldload ./myblk.ko
# newfs_ufs /dev/myblk0
# mount /dev/myblk0 /mnt/myblk
# echo "not persistent" > /mnt/myblk/token.txt
# umount /mnt/myblk
# kldunload myblk
# kldload ./myblk.ko
# mount /dev/myblk0 /mnt/myblk
# ls /mnt/myblk
```

`ls`应该显示一个空的或全新的UFS目录；`token.txt`消失了，因为模块卸载时后备缓冲区被内核回收了。

现在用`md(4)`的vnode后端做同样的序列，它使用磁盘上的真实文件：

```console
# truncate -s 64m /var/tmp/mdimage.img
# mdconfig -a -t vnode -f /var/tmp/mdimage.img -u 9
# newfs_ufs /dev/md9
# mount /dev/md9 /mnt/md
# echo "persistent" > /mnt/md/token.txt
# umount /mnt/md
# mdconfig -d -u 9
# mdconfig -a -t vnode -f /var/tmp/mdimage.img -u 9
# mount /dev/md9 /mnt/md
# cat /mnt/md/token.txt
persistent
```

**你观察什么。**

第一个序列丢失了文件；第二个序列保留了它。区别在于`md9`由磁盘上的真实文件支持，其状态独立于内核内部发生的事情而存在。这与`myblk0`形成对比，后者由`kldunload`时消失的内核堆支持。

**延伸问题。**阅读`/usr/src/sys/dev/md/md.c`中`mdstart_vnode`的`MD_VNODE`分支。识别vnode引用存储在哪里（提示：它存在于每单元的`struct md_s`上，而不是模块范围的全局变量）。用你自己的话解释为什么那个设计让后备存储能够跨越模块生命周期。

**参考实现。**`examples/part-06/ch27-storage-vfs/lab05-persistence/README.md`走过了两个序列及其诊断输出。

### 实验6：负载下的安全卸载

**目标。**验证拆卸路径正确处理活跃的文件系统。

**你做什么。**

加载模块、格式化、挂载。在一个终端中，启动压力循环。

```console
# while true; do dd if=/dev/urandom of=/mnt/myblk/stress bs=4k \
    count=512 2>/dev/null; sync; done
```

在另一个终端中，尝试卸载。

```console
# kldunload myblk
kldunload: can't unload file: Device busy
```

停止压力循环。卸载文件系统。卸载模块。

**你观察什么。**

初始卸载必须优雅地失败。卸载文件系统后，最终卸载必须成功。`dmesg`不能显示任何内核警告。

**延伸问题。**与其杀死压力循环，直接尝试`umount /mnt/myblk`。UFS在你写入进行中时允许卸载吗？错误是什么，意味着什么？

**参考实现。**`examples/part-06/ch27-storage-vfs/lab06-safe-unload/`包含执行上述序列并报告失败的测试脚本。

### 实验7：用DTrace观察BIO流量

**目标。**使用DTrace实时查看BIO路径。

**你做什么。**

在驱动程序加载且文件系统挂载的情况下，在一个终端中运行以下DTrace单行命令：

```console
# dtrace -n 'fbt::myblk_strategy:entry { \
    printf("cmd=%d off=%lld len=%u", \
        args[0]->bio_cmd, args[0]->bio_offset, \
        args[0]->bio_bcount); \
    @count[args[0]->bio_cmd] = count(); \
}'
```

在另一个终端中，在挂载的文件系统上创建和读取文件。

**你观察什么。**

注意你看到哪些BIO命令以及数量。注意典型的偏移量和长度。比较来自`dd`流量、`cp`流量和`tar`流量的模式。注意`cp`或`mv`如何根据缓冲区缓存决定刷新什么而产生非常不同的BIO模式。

**延伸问题。**在DTrace运行时发出`sync`。`sync`导致哪些BIO命令？`newfs_ufs`呢？

**参考实现。**`examples/part-06/ch27-storage-vfs/lab07-dtrace/README.md`包含示例DTrace输出和笔记。

### 实验8：添加getattr属性

**目标。**实现一个响应`GEOM::ident`的`d_getattr`回调。

**你做什么。**

将第5节中的`myblk_getattr`函数添加到驱动程序中，并在`disk_create`之前在磁盘上注册它。重新构建、重新加载并检查`diskinfo -v /dev/myblk0`。

**你观察什么。**

`ident`字段现在应该显示`MYBLK0`而不是`(null)`。

**延伸问题。**文件系统还可能查询哪些属性？查看`/usr/src/sys/geom/geom.h`中的命名属性如`GEOM::rotation_rate`。也尝试实现它。

**参考实现。**`examples/part-06/ch27-storage-vfs/lab08-getattr/myfirst_blk.c`。

### 实验9：探索md(4)进行比较

**目标。**阅读一个真实的FreeBSD存储驱动程序并识别我们使用过的模式。

**你做什么。**

打开`/usr/src/sys/dev/md/md.c`。这是一个长文件。不要试图阅读每一行。相反，找到并理解以下具体内容：

1. 文件顶部的`g_md_class`结构。
2. `struct md_s` softc。
3. 处理`MD_MALLOC`内存磁盘的BIO_READ和BIO_WRITE的`mdstart_malloc`函数。
4. `md_kthread`中的工作线程模式（或你版本中的等效实现）。
5. 按需创建新单元的`MDIOCATTACH` ioctl处理器。

将每项与我们驱动程序中的对应代码进行比较。

**你观察什么。**

发现差异。`md(4)`在哪里有我们没有的功能？我们的驱动程序在哪里以更简单的形式拥有相同的机制？你需要在哪里扩展我们的驱动程序以添加`md(4)`的功能之一？

**参考笔记。**`examples/part-06/ch27-storage-vfs/lab09-md-comparison/NOTES.md`包含FreeBSD 14.3的`md.c`相关章节的映射演练。

### 实验10：故意破坏它

**目标。**诱发已知故障模式，以便你在真实工作中能快速识别它们。

**你做什么。**

获取实验8中完成的驱动程序的干净副本。在单独的副本中（不要将破坏混在一起），逐一引入以下bug，重新构建、加载并观察。

**破坏1：忘记biodone。**注释掉`BIO_READ` case中的`biodone(bp)`调用。加载、挂载并对一个文件运行`cat`。`cat`将永远挂起。尝试用`Ctrl-C`杀死它；它可能不响应。对卡住的PID使用`procstat -kk`查看进程在哪里等待。这是经典的泄漏BIO症状。

**破坏2：在disk_destroy之前释放后备存储。**在`myblk_detach_unit`中，交换顺序使`free(sc->backing, ...)`在`disk_destroy(sc->disk)`之前。加载、格式化、挂载、卸载并尝试卸载模块。如果在卸载窗口期间没有BIO在飞行中，你会安然无恙。如果有任何BIO在飞行中（用运行中的`dd`确保这一点），你将因页面错误而崩溃。

**破坏3：跳过bio_resid。**从`BIO_READ` case中删除`bp->bio_resid = 0`行。加载、格式化、挂载并创建一个文件。读回它。根据`bio_resid`在分配时的垃圾内容，文件系统可能报告不正确的读取大小并可能记录错误。有时它能工作；有时不行。这是被遗忘的`bio_resid`的特征性间歇性故障。

**破坏4：差一的边界检查。**将边界检查从`offset > sc->backing_size`改为`offset >= sc->backing_size`。这拒绝了最后一个偏移量处的有效读取。加载、格式化、挂载。尝试写入一个延伸到最后一个块的文件。观察UFS是否注意到；`dd`是否注意到；报告了什么错误。

**你观察什么。**

在每种情况下，在你的日志中描述你观察到了什么、哪个工具揭示了问题（dmesg、`procstat`、`gstat`、panic跟踪），以及修复应该是什么。然后应用修复并确认正常操作。

**延伸问题。**什么命令序列可以可靠地重现每个故障？你能写一个确定性地触发破坏1或破坏2的shell脚本吗？

**参考笔记。**`examples/part-06/ch27-storage-vfs/lab10-break-on-purpose/BREAKAGES.md`包含每个故障模式的简短描述和测试脚本。

### 实验11：在不同块大小下测量

**目标。**理解BIO大小如何影响吞吐量。

**你做什么。**

在驱动程序加载且文件系统挂载的情况下，使用逐渐增大的块大小运行`dd`并计时每次运行。

```console
# for bs in 512 4096 32768 131072 524288 1048576; do
    rm -f /mnt/myblk/bench
    time dd if=/dev/zero of=/mnt/myblk/bench bs=$bs count=$((16*1024*1024/bs))
done
```

Record the throughput in each case.

**你观察什么。**

吞吐量应该随着块大小增加而增加，然后在`d_maxsize`（通常为128 KiB）附近达到平台。非常小的块大小将主要受每BIO开销支配。

**延伸问题。**在什么块大小下曲线明显达到平台？为什么？

### 实验12：两个进程的竞争测试

**目标。**观察驱动程序如何处理来自多个进程的同时访问。

**你做什么。**

在驱动程序加载且文件系统挂载的情况下，并行运行两个`dd`进程写入不同文件。

```console
# dd if=/dev/urandom of=/mnt/myblk/a bs=4k count=1024 &
# dd if=/dev/urandom of=/mnt/myblk/b bs=4k count=1024 &
# wait
```

Record the combined throughput.

**你观察什么。**

两个写入都应该完成且没有损坏。用每个文件的`md5`或`sha256`验证。合并吞吐量可能略低于两倍的单进程吞吐量，因为我们粗粒度互斥锁的锁竞争。

**延伸问题。**移除互斥锁是否影响吞吐量？是否导致损坏？为什么是或为什么不是？

### 关于实验纪律

每个实验都很小，没有一个是考试。如果你卡住了，参考实现就在那里供你比较。但不要把复制粘贴作为你的第一次尝试。复制不是技能。技能在于输入、阅读、诊断和验证。

保持你的日志打开。记录你运行了什么、看到了什么和什么让你惊讶。存储bug经常跨项目重复，你未来的自己会感谢现在的自己做的笔记。

## 挑战练习

挑战练习将驱动程序推得更远一些。每一个都限定在初学者可以凭借本章已涵盖的材料结合对FreeBSD源码的仔细阅读来完成的范围内。它们不是计时的。慢慢来。打开源码树。查阅手册页。有疑问时与`md(4)`比较。

下面的每个挑战在`examples/part-06/ch27-storage-vfs/`下都有桩文件夹，但没有提供参考解决方案。重点是自己解决它们。解决方案作为后续留给你与同行比较或发布在你的学习笔记中。

### 挑战1：暴露只读模式

添加一个模块加载时可调参数，让驱动程序以只读模式启动。在只读模式下，`BIO_WRITE`和`BIO_DELETE`应以`EROFS`失败。`newfs_ufs`应拒绝格式化设备，不带`-r`的`mount`应拒绝挂载。

提示。可调参数可以是绑定到静态变量的`sysctl_int`。`TUNABLE_INT`是另一种方式，仅在加载时使用。你的策略函数可以在分派写入之前检查该变量。记住，在文件系统挂载时运行时更改模式是数据损坏的根源；你可以禁止更改或记录可调参数仅在模块加载时生效。

### 挑战2：实现第二个单元

添加对恰好两个单元的支持：`myblk0`和`myblk1`。每个应该有自己的后备存储和自己的大小。不要尝试实现完全动态的单元分配；只需在模块加载器中硬编码两个softc和两个附加调用。

提示。将后备分配、磁盘分配和磁盘创建移入按单元号和大小参数化的`myblk_attach_unit`中，并从加载器中调用两次。确保分离路径遍历两个单元。

### 挑战3：用Sysctl计数器响应BIO_DELETE

扩展`BIO_DELETE`处理，同时递增一个`sysctl`计数器，报告已删除的总字节数。在运行`fstrim /mnt/myblk`或`dd`写入和覆盖文件时用`sysctl dev.myblk`验证。

提示。UFS默认不发出`BIO_DELETE`。要看到删除流量，用`-o trim`挂载。你可以用实验7中的DTrace单行命令验证trim流。

### 挑战4：响应BIO_GETATTR查询rotation_rate

扩展`myblk_getattr`以用`DISK_RR_NON_ROTATING`（在`/usr/src/sys/geom/geom_disk.h`中定义）回答`GEOM::rotation_rate`。用`gpart show`和`diskinfo -v`验证设备报告为非旋转。

提示。属性作为普通`u_int`返回。看看`md(4)`如何为类似属性处理`BIO_GETATTR`。

### 挑战5：调整设备大小

添加一个ioctl，让用户空间在未挂载任何东西时调整后备存储大小。如果文件系统已挂载，ioctl必须以`EBUSY`失败。如果调整大小成功，更新`d_mediasize`并通知GEOM，以便`diskinfo`报告新大小。

提示。查看`md(4)`的`MDIOCRESIZE`处理了解模式。这是一个非平凡的挑战；慢慢来，用一次性的文件系统测试。不要在你舍不得丢失的后备上尝试这个。

### 挑战6：写入计数器和速率显示

添加每秒写入字节计数器，通过`sysctl`暴露，以及一个小的用户空间shell脚本，每秒读取sysctl并打印人类可读的速率。这对测试有用，也给你提供将指标接入内核可观测性机制的经验。

提示。在计数器上使用`atomic_add_long`。shell脚本是一个`while true`循环中的单行命令。

### 挑战7：固定模式后备存储

实现一种后备存储模式，读取总是返回固定的字节模式，写入被静默丢弃。这类似于`g_zero`，但具有可配置的模式字节。当你不关心数据内容时，它对压力测试上面的层很有用。

提示。在策略函数内部分支到一个模式变量。正常模式保留内存中的后备，在模式模式下跳过`memcpy`。

### 挑战8：编写类似mdconfig的控制工具

编写一个小的用户空间程序，与驱动程序上的控制设备（你需要添加一个）通信，并可以在运行时创建、销毁和查询单元。程序应该接受类似`mdconfig`的命令行标志。

提示。这是一个相当大的挑战。从一个打印"hello"的单个ioctl开始，从那里构建。在你的控制设备上用cdevsw的`make_dev`，然后在该cdev上实现`d_ioctl`。

### 挑战9：在模拟崩溃中存活

添加一个模式，驱动程序静默丢弃每第N次写入（假装写入成功但实际上什么也不做）。用这个来测试UFS对丢失写入的弹性。

提示。这是一个危险的模式。只在一次性的文件系统上运行。你应该能用它重现有趣的`fsck_ffs`修复场景。准备好向自己解释为什么这个模式只在你可以从零重新生成的伪设备上是安全的。

### 挑战10：足够理解md(4)以至于能教授它

写一页关于`md(4)`如何响应`MDIOCATTACH`创建新单元的说明。涵盖ioctl路径、softc分配、特定于后备类型的初始化、`g_disk`连接和工作线程创建。这是一个阅读挑战而不是编码挑战，但它是对加深你对存储栈理解最有用的练习之一。

提示。`/usr/src/sys/dev/md/md.c`和`/usr/src/sbin/mdconfig/mdconfig.c`是两个要阅读的文件。注意`/usr/src/sys/sys/mdioctl.h`中的`struct md_ioctl`结构，因为那是用户空间和内核之间的ABI。

### 何时尝试挑战

你不需要做所有挑战。选择一两个你对其中某个东西感到好奇或你能想象以后会用到的东西。一个仔细完成的挑战比五个半完成的挑战更有价值。参考`md(4)`实现会在你想将自己的方法与生产驱动程序比较时始终在那里。

## 故障排除

存储驱动程序有一族特定的故障模式。有些在发生时就显而易见。其他则一开始是静默的，只有在重启后才变得明显，有时中间还会出现数据损坏。本节列出了你在学习本章和做实验时最可能看到的症状，以及常见原因和修复方法。当出现问题时将其用作参考，在开始之前至少通读一次，因为第二次识别故障模式要容易得多。

### `kldload`成功但没有/dev节点出现

**症状。**`kldload`返回零。`kldstat`显示模块已加载。但`/dev/myblk0`不存在。

**可能原因。**

- 你忘记调用`disk_create`。softc已分配，磁盘已分配，但磁盘未向GEOM注册。
- 你用`d_name`设置为空指针或空字符串调用了`disk_create`。
- 你用`d_mediasize`设置为零调用了`disk_create`。`g_disk`静默拒绝创建零大小的提供者。
- 你在填充字段之前调用了`disk_create`。框架在注册时捕获字段值，之后不会重新读取。

**修复。**用`dmesg`检查内核消息缓冲区。`g_disk`在拒绝注册时打印诊断信息。修复字段值并重新构建。

### `kldload`失败并显示"module version mismatch"

**症状。**加载模块报告`kldload: can't load ./myblk.ko: No such file or directory`或更明确的版本不匹配错误。

**可能原因。**

- 你针对与当前运行内核不同的内核编译了模块。
- 你自己修改了`DISK_VERSION`，你不应该这样做。
- 你忘记添加`MODULE_VERSION(myblk, 1)`。

**修复。**检查`uname -a`和你的构建选择的内核版本。针对运行中的内核重新编译。

### `diskinfo`打印错误的大小

**症状。**`diskinfo -v /dev/myblk0`打印的大小与`MYBLK_MEDIASIZE`不匹配。

**可能原因。**

- 你将`d_mediasize`设置为错误的表达式。一个常见的差一错误是将其设置为扇区计数而不是字节数。
- 你的`MYBLK_MEDIASIZE`定义为非`(size * 1024 * 1024)`的值，宏被以不同于你意图的方式解释。积极使用括号。

**修复。**在加载消息中打印大小并与`diskinfo -v`进行合理性检查。

### `newfs_ufs`失败并显示"Device not configured"

**症状。**`newfs_ufs /dev/myblk0`打印`newfs: /dev/myblk0: Device not configured`。

**可能原因。**

- 你的策略函数仍然是返回`ENXIO`的占位符。`ENXIO`被`errno`映射为`Device not configured`消息。

**修复。**实现第5节中的策略函数。

### `newfs_ufs`挂起

**症状。**`newfs_ufs /dev/myblk0`启动但从不完成。

**可能原因。**

- 你的策略函数在某条路径上没有调用`biodone`。`newfs_ufs`发出一个BIO，等待其完成，如果完成永远不会到来就会永远等待。
- 你的策略函数在某条路径上调用了两次`biodone`。第一次调用返回成功；第二次调用通常会导致崩溃，但在某些情况下BIO状态损坏足以导致挂起。

**修复。**审计你的策略函数。每个控制流路径必须恰好以一次`biodone(bp)`调用结束。一个有用的模式是在函数末尾使用单一退出点。

### `mount`失败并显示"bad super block"

**症状。**`mount /dev/myblk0 /mnt/myblk`报告`mount: /dev/myblk0: bad magic`。

**可能原因。**

- 你的策略函数在某些偏移量返回错误数据。超级块在偏移65536处，UFS仔细验证它。
- 你的边界检查拒绝了合法的读取。
- 你的`memcpy`从错误的地址复制（通常是偏移算术中的差一错误）。

**修复。**用`dd`向设备写入已知模式，然后用`dd`在不同偏移量读回并用`cmp`比较。如果模式正确往返，基本I/O是正确的。如果不正确，找到第一个出现偏差的偏移量并检查相应边界检查或地址算术处的代码。

### `kldunload`挂起

**症状。**`kldunload myblk`不返回。

**可能原因。**

- 一个BIO在飞行中，你的策略函数从未调用`biodone`。`disk_destroy`正在等待BIO完成。
- 你添加了工作线程，它在一个永远不会被唤醒的函数中睡眠。

**修复。**在另一个终端中运行`procstat -kk`。查看`g_event`线程的栈和你驱动程序线程的任何栈。如果它们卡在`sleep`或`waitfor`状态，你有一个泄漏的BIO或行为不当的工作线程。

### `kldunload`返回"Device busy"

**症状。**`kldunload myblk`报告`Device busy`并退出。

**可能原因。**

- 文件系统仍然挂载在`/dev/myblk0`上。
- 一个程序仍然以原始访问方式打开了`/dev/myblk0`。
- 之前终端会话的`dd`仍然在后台运行。

**修复。**运行`mount | grep myblk`检查活跃的挂载。运行`fuser /dev/myblk0`找到打开的句柄。卸载、关闭，然后卸载模块。

### 内核崩溃并显示"freeing free memory"

**症状。**内核崩溃，显示关于释放已释放内存的消息，栈跟踪经过你的驱动程序。

**可能原因。**

- 分离路径正在两次释放softc或后备存储。
- 一个工作线程在`disk_destroy`后仍然存活并试图访问已释放的状态。

**修复。**审查分离顺序。先销毁磁盘（它等待飞行中的BIO），然后释放后备存储，然后销毁互斥锁，然后释放softc。如果你添加了工作线程，确保它在任何`free`被调用之前已经退出。

### 内核崩溃并显示"vm_fault: kernel mode"

**症状。**内核在你的驱动程序内崩溃，出现页面错误，通常在策略函数或分离路径中。

**可能原因。**

- 你解引用了一个空指针或已释放的指针。最常见的情况是在`sc->backing`被释放后使用它。
- 你混淆了`bp->bio_data`和`bp->bio_disk`，从错误的指针读取。

**修复。**审计指针生命周期。如果后备存储在分离期间被释放，确保之后没有BIO还能到达策略函数。`disk_destroy` -> `free(backing)`是正确的顺序。

### `gstat`显示没有活动

**症状。**你正在对设备运行`dd`或`newfs_ufs`，但`gstat -f myblk0`显示零ops/s。

**可能原因。**

- 你在观察错误的设备。`gstat -f myblk0`使用正则表达式；确保你的设备名称匹配。
- 你的驱动程序使用了`gstat`正在过滤掉的自定义GEOM类名。

**修复。**不带过滤器运行`gstat`并查找你的设备。仔细检查名称字段。

### DELETE的"Operation not supported"

**症状。**带trim挂载失败或`fstrim`打印"Operation not supported"。

**可能原因。**

- 你的策略函数不处理`BIO_DELETE`并返回`EOPNOTSUPP`。
- 文件系统在挂载期间探测了`BIO_DELETE`支持并缓存了否定结果。

**修复。**在策略函数中实现`BIO_DELETE`，然后卸载并重新挂载。大多数文件系统只在挂载时探测。

### /dev/myblk0在kldload几秒后才出现

**症状。**`kldload`后，`ls /dev/myblk0`立即失败。片刻后，它成功。

**可能原因。**

- GEOM异步处理事件。`disk_create`将事件排队，直到事件线程拾取它才发布提供者。
- 在负载下的系统上，事件队列可能很慢。

**修复。**这是正常行为。如果你的脚本依赖于节点在`kldload`后立即可用，添加一个小延迟或轮询循环。

### 写入的数据可读但乱码

**症状。**写入后读取返回正确数量的字节但内容不同。

**可能原因。**

- 后备存储偏移算术中的差一错误。
- 一个并发的BIO与你期望的重叠，你的锁持有时间不够长。
- 策略函数在内核完成设置`bp->bio_data`之前从中读取（这对普通BIO极其不可能，但可能在解析属性的方式有bug时发生）。

**修复。**在策略函数中添加一个`printf`，记录`memcpy`前后前几个字节。用已知模式重复测试并查找不匹配。

### 后备存储未释放，每次重新加载内存增长

**症状。**`vmstat -m | grep myblk`显示分配字节在每个加载/卸载周期中增长。

**可能原因。**

- `MOD_UNLOAD`处理器返回时没有调用`myblk_detach_unit`，所以`free(sc->backing, M_MYBLK)`被跳过了。
- `MOD_UNLOAD`中的错误路径在到达释放之前提前返回。每个错误路径都需要释放，否则分配就会泄漏。
- 一个工作线程持有softc的引用，处理器在该引用存在时拒绝释放。

**修复。**审计`MOD_UNLOAD`路径。`vmstat -m`是一个粗糙但有效的工具。在释放路径中添加`printf`以确认它被执行。

### `gstat`显示非常高的队列长度

**症状。**`gstat -I 1`显示`L(q)`上升到数十或数百且从不返回零。

**可能原因。**

- 你的策略函数很慢或阻塞，导致BIO排队速度比处理速度快。
- 你添加了工作线程但它的调度频率比应该的低。
- 一个同步瓶颈（严重竞争的互斥锁）正在串行化工作。

**修复。**用DTrace分析策略函数在做什么。如果每个BIO的延迟增加了，调查原因。对于内存驱动程序，这几乎不应该发生；如果发生了，你可能在热路径中引入了`vn_rdwr`或其他阻塞调用。

### 策略函数被调用时bio_disk为NULL

**症状。**解引用`bp->bio_disk->d_drv1`时在策略函数中出现内核崩溃。

**可能原因。**

- BIO被驱动程序外部的代码错误合成。
- 你在错误的上下文中访问`bp->bio_disk`。在某些GEOM路径中，`bp->bio_disk`只在`g_disk`驱动程序的策略函数内部有效。

**修复。**如果你需要访问softc，在策略函数开始时进行。将指针缓存在局部变量中。不要从不同线程或延迟回调中访问`bp->bio_disk`。

### 重新加载后出现神秘的I/O错误

**症状。**`kldunload`和`kldload`后，读取在卸载之前有效的偏移量上返回EIO。

**可能原因。**

- 你正在使用文件后备或vnode后备的实验（来自实验5或你自己的修改），并且文件的大小或内容在两次加载之间被更改了。
- 保存的偏移量与新偏移量之间存在类型不匹配（例如，在两次加载之间更改了`d_sectorsize`）。
- `d_mediasize`已更改但底层文件仍反映旧的布局。

**修复。**确保后备文件和驱动程序的几何参数在大小和扇区布局上一致。如果你更改了`d_mediasize`或`d_sectorsize`，重新生成后备文件以匹配。对于没有更改的简单重新加载，RAM后备驱动程序上的缓冲区总是全新的，所以神秘的重新加载后EIO通常指向几何参数不匹配而不是数据丢失。

### 卸载后访问计数卡在非零

**症状。**`umount`后，`geom disk list`仍然显示非零访问计数。

**可能原因。**

- 一个程序仍然打开了原始设备。`fuser /dev/myblk0`会揭示它。
- 文件系统未干净卸载。检查`mount | grep myblk`看它是否仍然挂载。
- 一个残留的NFS客户端或类似的东西正在持有文件系统打开。对本地内存磁盘不太可能，但在共享系统上可能。

**修复。**找到并关闭打开的句柄。如果`umount`报告成功但访问计数保持不变，重启是最安全的恢复方式。

### 驱动程序已加载但在geom -t中不可见

**症状。**`kldstat`显示模块已加载，但`geom -t`不显示我们名称的任何geom。

**可能原因。**

- 加载器运行了但从未调用`disk_create`。
- `disk_create`被调用了但事件线程尚未运行。

**修复。**添加`printf`确认`disk_create`运行了。`kldload`后等待一两秒再检查，给事件线程一个机会。

### 第二次加载时崩溃

**症状。**加载模块一次成功。卸载成功。第二次加载时崩溃。

**可能原因。**

- `MOD_UNLOAD`处理器没有重置`MOD_LOAD`假设为新的所有状态。
- 一个静态指针持有对跨卸载边界已释放结构的引用；下一次加载看到悬空指针。
- 在第一次加载时注册的GEOM类未注销。

**修复。**将你的加载和卸载路径作为匹配对进行审计。加载时的每个分配都需要卸载时相应的释放，加载时写入的每个指针都需要卸载时清除。对于GEOM类，`DECLARE_GEOM_CLASS`为你处理注销，但如果你绕过它，你必须自己做这个工作。

### newfs_ufs中止并显示"File system too small"

**症状。**`newfs_ufs /dev/myblk0`中止并显示`newfs: /dev/myblk0: partition smaller than minimum UFS size`。

**可能原因。**

- `MYBLK_MEDIASIZE`对UFS的最小实际大小来说太小了。
- 你更改大小后忘记重新构建模块。

**修复。**确保媒体大小至少有几兆字节。UFS的绝对最小值大约为1 MiB，但实际最小值为4-8 MiB，舒适的最小值为32 MiB或更多。

### mount -o trim不触发BIO_DELETE

**症状。**用`-o trim`挂载成功，但`gstat`在重度删除期间不显示删除操作。

**可能原因。**

- UFS仅在某些模式上发出`BIO_DELETE`；它不会无条件地修剪每个释放的块。
- 你的驱动程序没有在其`d_flags`中声明`BIO_DELETE`支持。

**修复。**在`disk_create`之前设置`sc->disk->d_flags |= DISKFLAG_CANDELETE;`。这告诉GEOM和文件系统你的驱动程序支持`BIO_DELETE`并愿意处理它们。

### UFS抱怨"Fragment out of bounds"

**症状。**挂载后，UFS记录关于片段越界的错误，文件操作开始返回EIO。

**可能原因。**

- 你的驱动程序在某些偏移量返回错误数据，UFS读取了损坏的元数据块。
- 后备存储在其他测试期间被部分覆盖。
- 边界检查算术返回了不正确的范围。

**修复。**卸载、运行`fsck_ffs -y /dev/myblk0`修复，然后重新测试。如果错误在全新文件系统上重现，在策略函数中寻找偏移计算bug。

### 内核打印"interrupt storm"消息

**症状。**`dmesg`显示关于中断风暴的消息，系统响应性下降。

**可能原因。**

- 一个真正的硬件驱动程序（不是你的）行为不当。
- 你的驱动程序没问题；这是另一个子系统的问题。

**修复。**验证风暴与你的模块无关。如果是的话，问题几乎可以肯定在中断处理器中，而我们的伪驱动程序没有中断处理器。

### 关机时重启挂在卸载上

**症状。**关机时，系统在卸载时挂住，显示类似"Syncing disks, vnodes remaining..."的消息。

**可能原因。**

- 一个文件系统仍然挂载在你的设备上，你的驱动程序持有一个BIO。
- 一个同步线程卡在等待完成。

**修复。**确保你的驱动程序在系统关机前干净地卸载。一个健壮的方法是添加一个`shutdown_post_sync`事件处理器来卸载文件系统并卸载模块。对于开发，在发出`shutdown -r now`之前手动卸载并卸载模块。

### 一般建议

每当出现问题时，第一步是阅读`dmesg`并查找来自你自己printf和内核子系统的消息。第二步是运行`procstat -kk`并查看线程在做什么。第三步是查阅`gstat`、`geom disk list`和`geom -t`了解存储拓扑。这三个工具在几乎每种情况下都会告诉你大部分你需要的信息。

如果发生崩溃，FreeBSD会把你放进调试器。用`bt`捕获回溯，用`show registers`进行寄存器转储，然后用`reboot`重启。如果获取了崩溃转储，`kgdb`可以让你离线检查`/var/crash/vmcore.last`上的状态。保留崩溃转储，至少在开发环境中，当你追踪间歇性bug时会立即产生回报。

最重要的是，当某个东西失败时，尝试重现它。存储驱动程序中的间歇性bug几乎总是由有多少BIO在飞行中、它们花费多长时间以及调度器何时决定运行你的线程等时间差异引起的。如果你能找到可靠的重现方法，你就距离修复已经走了一大半。

## 总结

这是一章很长的内容。让我们花点时间退后一步，看看我们涵盖了什么。

我们从将存储驱动程序置于FreeBSD的分层架构中开始。虚拟文件系统层位于系统调用和文件系统之间，给每个文件系统一个共同的形状。`devfs`本身是一个文件系统，提供用户空间工具和管理员用来引用设备的`/dev`目录。存储驱动程序不在VFS内部。它们位于栈的底部，在GEOM之下，在缓冲区缓存之下，通过`struct bio`与内核的其余部分通信。

我们从零开始构建了一个可工作的伪块设备驱动程序。在第3节中，我们编写了用`g_disk`注册磁盘并发布`/dev`节点的骨架。在第4节中，我们探索了GEOM的类、geom、提供者和消费者的概念，并理解了拓扑如何组合以及访问计数如何在拆卸期间保持系统安全。在第5节中，我们实现了针对内存后备存储实际服务`BIO_READ`、`BIO_WRITE`、`BIO_DELETE`和`BIO_FLUSH`的策略函数。在第6节中，我们用`newfs_ufs`格式化设备，在其上挂载了真正的文件系统，并看到两条访问路径（原始和文件系统）在我们的策略函数中汇合。在第7节中，我们调查了持久性选项并添加了存活模块重载的简单技术。在第8节中，我们详细遍历了拆卸路径并学习了如何测试它。在第9节中，我们看了增长的驱动程序倾向于去的方向：多单元支持、ioctl表面、源文件拆分和稳定版本管理。

我们通过实验练习了驱动程序，通过挑战扩展了它。我们在故障排除部分收集了常见的故障模式。在整个过程中，我们将目光保持在真实的FreeBSD源码树上，因为本书的目标不是教授玩具内核代码，而是教授真正的代码。

你现在应该能够真正理解地阅读`md(4)`，而不仅仅是盯着它看。你应该能够阅读`g_zero.c`并识别它调用的每个函数。你应该能够通过症状诊断存储驱动程序bug的常见类别。你应该有一个可工作的、虽然简单的、你自己编写的伪块设备。

这是覆盖的大量内容。花点时间注意你已经走了多远。在第26章中，你知道如何编写字符驱动程序。现在你也可以编写块驱动程序了。这两章一起为你提供了FreeBSD中几乎所有其他类型驱动程序的基础，因为大多数驱动程序在与内核的其余部分相遇的边界上要么是面向字符的，要么是面向块的。

### 关键操作总结

为了快速回忆，以下是定义最小存储驱动程序的操作。

1. 包含正确的头文件：`sys/bio.h`、`geom/geom.h`、`geom/geom_disk.h`。
2. 用`disk_alloc`分配`struct disk`。
3. 填充`d_name`、`d_unit`、`d_strategy`、`d_sectorsize`、`d_mediasize`、`d_maxsize`和`d_drv1`。
4. 调用`disk_create(sc->disk, DISK_VERSION)`。
5. 在`d_strategy`中，对`bio_cmd`进行switch并服务请求。始终恰好调用`biodone`一次。
6. 在卸载路径中，在释放策略函数触及的任何内容之前调用`disk_destroy`。
7. 声明`MODULE_DEPEND`依赖于`g_disk`。
8. 除非有特定理由使用更小的值，否则对`d_maxsize`使用`MAXPHYS`。
9. 在负载下测试卸载路径。在文件系统挂载时测试。在原始`cat`持有设备打开时测试。
10. 出问题时阅读`dmesg`、`gstat`、`geom disk list`和`procstat -kk`。

这十个操作是你将编写的每个FreeBSD存储驱动程序的骨架。它们在`ada(4)`、`nvme(4)`、`mmcsd(4)`、`zvol(4)`和源码树中的每个其他驱动程序中以不同形式出现。一旦你看到这个模式，真实驱动程序之间的变化就不那么神秘了。

### 关于原始访问的提醒

即使文件系统已挂载，你的驱动程序仍然可以作为原始块设备访问。`/dev/myblk0`仍然是一个有效的句柄，`dd`、`diskinfo`、`gstat`和`dtrace`等工具可以使用它。两条访问路径通过GEOM的纪律共存：两条路径都发出BIO，两条路径都遵守访问计数，你的策略函数服务两者而不区分它们。这种统一性是GEOM给予存储驱动程序作者的巨大礼物。

### 关于安全的提醒

在共享系统上开发存储驱动程序是自找麻烦。使用虚拟机，或至少使用一个你可以重新安装的系统。保持救援镜像在手。备份你无法承受丢失的任何东西，包括你正在编写中的代码。本章的驱动程序行为良好，不应该损坏任何东西，但你将来编写的驱动程序可能不会，而且准备的成本与未准备的代价相比非常小。

### 在FreeBSD源码树中接下来看什么

如果你想在下一章之前继续探索存储，源码树中有三个领域值得仔细阅读。

- `/usr/src/sys/geom/`有GEOM框架本身，包括`g_class`、`g_disk`和许多变换类如`g_mirror`、`g_stripe`和`g_eli`。
- `/usr/src/sys/dev/md/md.c`是功能齐全的内存磁盘驱动程序，本章已经多次提到。
- `/usr/src/sys/ufs/`是UFS文件系统。不是驱动程序工作的必读内容，但它有助于了解你上面那层的样子。

阅读这些不是下一章的先决条件。这是对你自身成长的建议。

## 通向下一章的桥梁

在本章中，我们从零开始构建了一个存储驱动程序。流经其中的数据是系统内部的：写入文件的字节、从文件读取的字节、在缓冲区缓存中来回移动的超级块、柱面组和inode。没有字节离开过机器。驱动程序的整个世界是内核自己的内存和消费它的进程。

第28章将我们带入一个不同的世界。我们将编写一个网络接口驱动程序。网络驱动程序是传输驱动程序，就像第26章的USB串行驱动程序和本章的存储驱动程序一样，但它们的对话伙伴不是进程也不是文件系统。它是一个网络栈，工作单位不是字节范围也不是块，而是数据包。数据包是带有头部和有效载荷的结构化对象，驱动程序参与包括IP、ARP、ICMP、TCP、UDP和许多其他协议在内的栈。

你本章内化的模式将以不同的名称重新出现。你将看到`struct mbuf`代替`struct bio`。你将看到`ifnet`接口代替`g_disk`。你将看到`if_transmit`和`if_input`钩子代替`disk_strategy`。你将看到链接到内核网络栈中的网络接口对象代替GEOM提供者和消费者。角色是相同的：传输驱动程序从上面接收请求，向下传递，从下面接受响应，向上传递。

许多关注点也是相同的。锁。热拔出。分离时的资源清理。通过内核工具的可观测性。面对错误时的安全性。基础延续了下来。改变的是词汇、工作单元的结构以及一些特定工具。

在继续之前，短暂休息一下。卸载你的存储驱动程序。运行`kldstat`确认本章的任何东西都不再加载。合上你的实验日志。站起来。续杯咖啡。下一章将与本章一样充实，你需要一个清醒的头脑。

当你回来时，第28章将以与本章相同的方式开始：一个温和的引言和一幅清晰的路线图。在那里见。

## 快速参考

下面的表格旨在作为你在编写或调试存储驱动程序时快速查找名称、命令或路径的参考。它们不是本章前面完整解释的替代品。

### 关键头文件

| 头文件 | 定义 |
|--------|------|
| `sys/bio.h` | `struct bio`、`BIO_READ`、`BIO_WRITE`、`BIO_DELETE`、`BIO_FLUSH`、`BIO_GETATTR` |
| `geom/geom.h` | `struct g_class`、`struct g_geom`、`struct g_provider`、`struct g_consumer`、拓扑原语 |
| `geom/geom_disk.h` | `struct disk`、`DISK_VERSION`、`disk_alloc`、`disk_create`、`disk_destroy`、`disk_gone` |
| `sys/module.h` | `DECLARE_MODULE`、`MODULE_VERSION`、`MODULE_DEPEND` |
| `sys/malloc.h` | `MALLOC_DEFINE`、`malloc`、`free`、`M_WAITOK`、`M_NOWAIT`、`M_ZERO` |
| `sys/lock.h`、`sys/mutex.h` | `struct mtx`、`mtx_init`、`mtx_lock`、`mtx_unlock`、`mtx_destroy` |

### 关键结构

| 结构 | 角色 |
|------|------|
| `struct disk` | `g_disk`对磁盘的表示。由驱动程序填充，由框架拥有。 |
| `struct bio` | 一个I/O请求，在GEOM层之间传递并进入驱动程序的策略函数。 |
| `struct g_provider` | geom的面向生产者接口。文件系统和其他geom从提供者消费。 |
| `struct g_consumer` | 从一个geom到另一个geom提供者的连接。 |
| `struct g_geom` | `g_class`的一个实例。 |
| `struct g_class` | 创建geom的模板。定义`init`、`fini`、`start`、`access`等方法。 |

### 常见BIO命令

| 命令 | 含义 |
|------|------|
| `BIO_READ` | 从设备读取字节到缓冲区。 |
| `BIO_WRITE` | 从缓冲区写入字节到设备。 |
| `BIO_DELETE` | 丢弃一个范围的块。用于TRIM。 |
| `BIO_FLUSH` | 将未完成的写入提交到持久存储。 |
| `BIO_GETATTR` | 从提供者查询命名属性的值。 |
| `BIO_ZONE` | 分区块设备操作。不常用。 |

### 常见GEOM工具

| 工具 | 用途 |
|------|------|
| `geom disk list` | 列出已注册的磁盘及其提供者。 |
| `geom -t` | 以树形显示整个GEOM拓扑。 |
| `geom part show` | 显示分区geom及其提供者。 |
| `gstat` | 实时每提供者I/O统计。 |
| `diskinfo -v /dev/xxx` | 显示磁盘几何参数和属性。 |
| `iostat -x 1` | 实时每设备吞吐量和延迟。 |
| `dd if=... of=...` | 用于测试的原始块I/O。 |
| `newfs_ufs /dev/xxx` | 在设备上创建UFS文件系统。 |
| `mount /dev/xxx /mnt` | 挂载文件系统。 |
| `umount /mnt` | 卸载文件系统。 |
| `mdconfig` | 创建或销毁内存磁盘。 |
| `fuser` | 找到持有文件打开的进程。 |
| `procstat -kk` | 显示所有线程的内核栈跟踪。 |

### 关键回调类型定义

| 类型定义 | 用途 |
|---------|------|
| `disk_strategy_t` | 处理BIO。核心I/O函数。必需。 |
| `disk_open_t` | 当授予新访问时调用。可选。 |
| `disk_close_t` | 当释放访问时调用。可选。 |
| `disk_ioctl_t` | 处理`/dev`节点上的ioctl。可选。 |
| `disk_getattr_t` | 回答`BIO_GETATTR`查询。可选。 |
| `disk_gone_t` | 当磁盘被强制移除时通知驱动程序。可选。 |

### 文件和路径参考

| 路径 | 包含内容 |
|------|----------|
| `/usr/src/sys/geom/geom_disk.c` | `g_disk`实现。 |
| `/usr/src/sys/geom/geom_disk.h` | 公共`g_disk`接口。 |
| `/usr/src/sys/geom/geom.h` | 核心GEOM结构和函数。 |
| `/usr/src/sys/sys/bio.h` | `struct bio`定义。 |
| `/usr/src/sys/dev/md/md.c` | 参考内存磁盘驱动程序。 |
| `/usr/src/sys/geom/zero/g_zero.c` | 最小GEOM类，作为阅读参考很有用。 |
| `/usr/src/sys/ufs/ffs/ffs_vfsops.c` | UFS的挂载路径。如果你想看挂载在文件系统侧做了什么就阅读它。 |
| `/usr/src/share/man/man9/disk.9` | `disk(9)`手册页。 |
| `/usr/src/share/man/man9/g_bio.9` | `g_bio(9)`手册页。 |

### 常见磁盘标志

| 标志 | 含义 |
|------|------|
| `DISKFLAG_CANDELETE` | 驱动程序处理`BIO_DELETE`。 |
| `DISKFLAG_CANFLUSHCACHE` | 驱动程序处理`BIO_FLUSH`。 |
| `DISKFLAG_UNMAPPED_BIO` | 驱动程序接受未映射的BIO（高级）。 |
| `DISKFLAG_WRITE_PROTECT` | 设备是只读的。 |
| `DISKFLAG_DIRECT_COMPLETION` | 完成在任何上下文中都是安全的（高级）。 |

这些标志在`disk_create`之前设置在`sc->disk->d_flags`上。它们让内核对如何向你的驱动程序发出BIO做出更智能的选择。

### d_strategy的模式

以下是策略函数最常见的三种形状。

**模式1：同步、内存中。**我们的驱动程序使用这种方式。函数内联服务BIO并在调用`biodone`后返回。

```c
void strategy(struct bio *bp) {
    /* validate */
    switch (bp->bio_cmd) {
    case BIO_READ:  memcpy(bp->bio_data, ...); break;
    case BIO_WRITE: memcpy(..., bp->bio_data); break;
    }
    bp->bio_resid = 0;
    biodone(bp);
}
```

**模式2：排队到工作线程。**`md(4)`使用这种方式。函数将BIO附加到队列并唤醒工作线程。

```c
void strategy(struct bio *bp) {
    mtx_lock(&sc->lock);
    TAILQ_INSERT_TAIL(&sc->queue, bp, bio_queue);
    wakeup(&sc->queue);
    mtx_unlock(&sc->lock);
}
```

工作线程出队BIO，逐个服务它们（可能调用`vn_rdwr`或发出硬件命令），并用`biodone`完成每个。

**模式3：带中断完成的硬件DMA。**真正的硬件驱动程序使用这种方式。函数编程硬件、设置DMA并返回。稍后的中断处理器完成BIO。

```c
void strategy(struct bio *bp) {
    /* validate */
    program_hardware(bp);
    /* strategy returns, interrupt will call biodone eventually */
}
```

每种模式都有权衡。模式1最简单但不能阻塞。模式2处理阻塞工作但增加延迟。模式3是真正硬件所必需的，但需要中断处理，这增加了整个另一层复杂性。

我们的章节驱动程序使用模式1。`md(4)`使用模式2。`ada(4)`、`nvme(4)`等使用模式3。

### 最小注册序列

对于匆忙的驱动程序编写者，注册块设备的最小序列是：

```c
sc->disk = disk_alloc();
sc->disk->d_name       = "myblk";
sc->disk->d_unit       = sc->unit;
sc->disk->d_strategy   = myblk_strategy;
sc->disk->d_sectorsize = 512;
sc->disk->d_mediasize  = size_in_bytes;
sc->disk->d_maxsize    = MAXPHYS;
sc->disk->d_drv1       = sc;
disk_create(sc->disk, DISK_VERSION);
```

最小拆卸序列是：

```c
disk_destroy(sc->disk);
free(sc->backing, M_MYBLK);
mtx_destroy(&sc->lock);
free(sc, M_MYBLK);
```

## 术语表

**Access count（访问计数）。**GEOM提供者上三个计数器的元组，跟踪当前有多少读取者、写入者和独占持有者正在访问它。在`geom disk list`中显示为`rNwNeN`。

**Attach（附加）。**在Newbus意义上，驱动程序接管设备的步骤。在存储意义上，驱动程序调用`disk_create`向`g_disk`注册的步骤。这个词有重载；请根据上下文理解。

**Backing store（后备存储）。**存储设备字节实际存在的地方。对于我们的驱动程序，后备存储是内核内存中`malloc`分配的缓冲区。对于真正的磁盘，它是盘片或闪存。对于vnode模式的`md(4)`，它是主机文件系统中的文件。

**BIO。**一个`struct bio`。流经GEOM的I/O单位。

**BIO_DELETE。**要求驱动程序丢弃一个范围的块的BIO命令。用于SSD上的TRIM。

**BIO_FLUSH。**要求驱动程序在返回前使所有先前写入持久的BIO命令。

**BIO_GETATTR。**要求驱动程序返回命名属性值的BIO命令。

**BIO_READ。**要求驱动程序读取一个范围字节的BIO命令。

**BIO_WRITE。**要求驱动程序写入一个范围字节的BIO命令。

**Block device（块设备）。**以固定大小的块寻址的设备，具有可寻道的随机访问。在BSD中历史上与字符设备不同；在现代FreeBSD中，块和字符访问通过GEOM收敛，但心理上的区分仍然重要。

**Buffer cache（缓冲区缓存）。**将最近使用的文件系统块保存在RAM中的内核子系统。位于文件系统和GEOM之间。不要与页缓存混淆；它们在FreeBSD中相关但不同。

**Cache coherency（缓存一致性）。**读写以一致顺序看到彼此的属性。策略函数不能返回相对于同一偏移量上最近写入过时的数据。

**Cdev。**由`struct cdev`表示的字符设备节点。字符驱动程序用`make_dev`创建它们。块驱动程序通常不创建。

**Consumer（消费者）。**geom的面向输入的一侧。消费者附加到提供者并向其发出BIO。

**d_drv1。**`struct disk`中的通用指针，驱动程序在其中存储其私有上下文，通常是softc。

**d_mediasize。**设备的总大小（字节）。

**d_maxsize。**驱动程序可以接受的最大单个BIO。对伪设备通常为`MAXPHYS`。

**d_sectorsize。**扇区大小（字节）。通常为512或4096。

**d_strategy。**驱动程序的BIO处理函数。

**Devfs。**挂载在`/dev`的伪文件系统，为内核设备合成文件节点。

**Devstat。**内核的设备统计子系统，被`iostat`、`gstat`等使用。使用`g_disk`的存储驱动程序自动获得devstat集成。

**Disk_alloc。**分配`struct disk`。从不失败；内部使用`M_WAITOK`。

**Disk_create。**向`g_disk`注册填充好的`struct disk`。实际工作异步完成。

**Disk_destroy。**注销并销毁`struct disk`。等待飞行中的BIO完成。如果提供者仍有用户则崩溃。

**Disk_gone。**通知`g_disk`底层介质已消失。用于热拔出场景。与`disk_destroy`不同。

**DISK_VERSION。**`struct disk`接口的ABI版本。在`geom_disk.h`中定义，传递给`disk_create`。

**DTrace。**FreeBSD的动态跟踪工具。对观察BIO流量特别有用。

**Event thread（事件线程）。**GEOM用于处理拓扑事件（如创建和销毁geom）的单一内核线程。在`procstat`输出中通常称为`g_event`。

**Exclusive access（独占访问）。**提供者上禁止其他写入者的一种访问类型。文件系统在其挂载的设备上获取独占访问。

**Filesystem（文件系统）。**文件存储语义的具体实现，如UFS、ZFS、tmpfs或NFS。插入VFS。

**GEOM。**FreeBSD的可组合块层变换框架。类、geom、提供者和消费者是其主要对象。

**g_disk。**将磁盘形状的驱动程序包装为更简单API的GEOM子系统。我们的驱动程序使用它。

**g_event。**处理拓扑变更的GEOM事件线程。

**g_io_deliver。**在类级别用于完成BIO的函数。`g_disk`为我们调用它；我们的驱动程序调用`biodone`。

**g_io_request。**在类级别用于向下发出BIO的函数。仅在实现自己GEOM类的驱动程序中使用。

**Hotplug（热插拔）。**可以在不重启的情况下出现或消失的设备。

**Ioctl。**设备上区别于读写的控制操作。在存储路径中，`/dev/diskN`上的ioctl通过GEOM，可能由`g_disk`或驱动程序的`d_ioctl`处理。

**md(4)。**FreeBSD内存磁盘驱动程序。源码树中规范的伪块设备，推荐的阅读参考。

**Mount（挂载）。**将文件系统附加到命名空间中某个点的行为。调用VFS，VFS调用文件系统自己的挂载例程，后者通常打开一个GEOM提供者。

**Newbus。**FreeBSD的总线框架。用于字符和硬件驱动程序。我们的存储驱动程序不直接使用Newbus，因为它是伪设备；真正的存储驱动程序几乎总是使用它。

**Provider（提供者）。**geom的面向输出的一侧。其他geom或`/dev`节点消费提供者。

**Softc。**驱动程序的每实例状态结构。

**Strategy function（策略函数）。**驱动程序的BIO处理器。在`struct disk` API中称为`d_strategy`。

**Superblock（超级块）。**描述文件系统布局的小型磁盘上结构。UFS的在偏移65536处。

**Topology（拓扑）。**GEOM类、geom、提供者和消费者的树。受拓扑锁保护。

**Topology lock（拓扑锁）。**保护GEOM拓扑免受并发修改的全局锁。

**UFS。**Unix文件系统，FreeBSD的默认文件系统。位于`/usr/src/sys/ufs/`下。

**Unit（单元）。**驱动程序的一个编号实例。`myblk0`是`myblk`驱动程序的单元0。

**VFS。**虚拟文件系统层。位于系统调用和具体文件系统之间。

**Vnode。**内核对打开文件或目录的运行时句柄。存在于VFS内部。

**Withering（枯萎）。**GEOM从拓扑中移除提供者的过程。在事件线程上排队，等待飞行中的BIO，最终销毁提供者。

**Zone（区域）。**在VM子系统词汇中，通过UMA（通用内存分配器）分配的固定大小对象池。许多内核结构，包括BIO和GEOM提供者，从区域分配。

**BIO_ORDERED。**要求驱动程序仅在所有先前发出的BIO完成后才执行此BIO的BIO标志。用于写屏障。

**BIO_UNMAPPED。**指示`bio_data`不是映射的内核虚拟地址而是未映射页面列表的BIO标志。可以处理未映射数据的驱动程序应设置`DISKFLAG_UNMAPPED_BIO`。

**Direct completion（直接完成）。**在提交BIO的同一线程中完成BIO，不经过延迟回调。通常更快但不总是安全的。

**Drivers in Tree（树内驱动程序）。**生活在FreeBSD源码树中并作为标准内核构建一部分构建的驱动程序。与树外驱动程序相对，后者单独维护。

**Out-of-tree drivers（树外驱动程序）。**不属于FreeBSD源码树的驱动程序。这些需要针对匹配的内核编译，在内核ABI更改时可能需要更新。

**ABI。**应用二进制接口。允许两段编译代码互操作的函数调用、结构布局和类型大小约定的集合。`DISK_VERSION`是一种ABI标记。

**API。**应用编程接口。代码在源码级别使用的函数签名和类型的集合。与ABI不同：两个具有相同API的内核如果编译方式不同，可能有不同的ABI。

**KPI。**内核编程接口。FreeBSD对内核API的首选术语。对KPI稳定性的保证是有限的；始终针对你运行的内核重新编译。

**KLD。**内核可加载模块。我们产生的`.ko`文件。"KLD"代表Kernel Loadable Driver，虽然模块不一定是驱动程序。

**Module（模块）。**参见KLD。

**Taste（品尝）。**在GEOM词汇中，将提供者提供给所有类以便每个类决定是否附加到它的过程。新提供者出现时品尝自动发生。

**Retaste（重新品尝）。**强制GEOM再次品尝提供者，通常在其内容更改后。`geom provider retaste`对单个提供者触发此操作；`geom retaste`全局触发。

**Orphan（孤儿）。**在GEOM词汇中，底层存储已消失的提供者。孤儿由事件线程清理。

**Spoil（ spoil）。**与缓存失效相关的GEOM概念。如果提供者的内容以可能使缓存失效的方式更改，则称其已被spoiled。

**Bufobj。**将vnode（或GEOM消费者）与缓冲区缓存关联的内核对象。每个块设备和每个文件都有一个。

**bdev_strategy。**`d_strategy`的遗留同义词。现代代码直接使用`d_strategy`。

**Schedule（调度）。**将BIO放入内部队列以供稍后执行的行为。与"执行"不同。

**Plug/unplug。**在某些内核中，plug是BIO提交的批处理机制。FreeBSD没有plug/unplug；它立即交付BIO。

**Elevator（电梯）。**按偏移量排序BIO以减少磁盘寻道时间的BIO调度器。FreeBSD的GEOM不在GEOM层实现电梯；如果相关的话，这是块设备的责任。

**Superblock（超级块）。**文件系统的第一个元数据块。描述几何参数。对UFS在偏移65536处。

**Cylinder group（柱面组）。**UFS概念。文件系统被划分为区域，每个区域有自己的inode表和块分配位图。保持相关数据在旋转磁盘上物理接近，并限制单个坏区域可以造成的损害。

**Inode。**UFS（和POSIX）结构，描述一个文件：其模式、所有者、大小、时间戳和指向数据块的指针。文件名存在于目录条目中，不在inode中。

**Vop_vector。**文件系统向VFS提供的分发表，列出VFS知道如何询问的所有操作（打开、关闭、读、写、查找、重命名等）。VFS将这些作为间接函数指针调用。

**Devstat。**附加到设备上记录聚合I/O统计的内核结构。`iostat(8)`读取devstat数据；`g_disk`为其创建的每个磁盘分配并填充devstat结构。

**bp。**内核源码中BIO指针的简写。在策略函数和完成回调中几乎普遍使用。当你看到`struct bio *bp`时，读作"当前请求"。

**Bread。**缓冲区缓存函数，读取一个块，先查询缓存，仅在未命中时发出I/O。由文件系统使用，不由驱动程序使用。

**Bwrite。**缓冲区缓存函数，同步写入一个块。文件系统使用它；你的策略函数最终看到产生的BIO。

**Getblk。**缓冲区缓存函数，返回给定块的缓冲区，必要时分配它。由文件系统用作读取和写入的入口点。

**Bdwrite。**延迟缓冲区缓存写入。将缓冲区标记为脏但不立即发出I/O。稍后由syncer或缓冲区缓存压力写入。

**Bawrite。**异步缓冲区缓存写入。类似于`bwrite`但不等待完成。

**Syncer。**定期将脏缓冲区刷新到其后备设备的内核线程。干净地关闭文件系统需要syncer完成。

**Taskqueue。**在单独线程中运行回调的内核机制。当你的策略函数想延迟工作时很有用。在后面讨论中断处理器时会更深入地介绍。

**Callout。**在给定时间调度一次性或周期性回调的内核机制。在简单的存储驱动程序中不常用，但在实现超时的硬件驱动程序中非常常见。

**Witness。**检测锁顺序违规并打印警告的内核子系统。在调试内核中始终启用；节省数小时的调试时间。

**INVARIANTS。**添加运行时断言的内核编译选项。在调试内核中始终启用；在许多存储bug变成静默损坏之前捕获它们。

**Debug kernel（调试内核）。**用`INVARIANTS`、`WITNESS`和相关选项构建的内核。较慢但对驱动程序开发更安全。在实验工作中使用一个。

## 常见问题

### 我需要支持BIO_ORDERED吗？

对于同步服务BIO的伪设备，不需要。每个BIO在处理下一个之前完成，这自然地保留了顺序。对于异步驱动程序，你必须通过延迟后续BIO直到有序BIO完成来尊重`BIO_ORDERED`。

### d_maxsize和MAXPHYS之间有什么关系？

`d_maxsize`是你的驱动程序可以接受的最大BIO大小。`MAXPHYS`是BIO大小的编译时上限，在`/usr/src/sys/sys/param.h`中定义。在amd64和arm64等64位系统上，`MAXPHYS`为1 MiB；在32位系统上为128 KiB。FreeBSD 14.3还暴露了一个运行时可调参数`maxphys`，一些子系统通过`MAXPHYS`宏或`maxphys`变量查询它。设置`d_maxsize = MAXPHYS`接受内核愿意发出的任何大小。对大多数伪驱动程序来说这没问题。

### 我的驱动程序能向自身发出BIO吗？

技术上可以，但很少有道理。这种模式被GEOM变换类使用（它们从上面接收BIO并向下发出新BIO）。`g_disk`驱动程序位于栈的底部，没有向下的方向；如果你需要跨多个后备单元拆分工作，你可能需要工作线程而不是嵌套BIO。

### 为什么struct disk中有些字段使用u_int，有些使用off_t？

`u_int`用于适合32位的无符号整数大小（扇区大小、磁头数等）。`off_t`是有符号64位类型，用于可以超过32位的字节偏移和大小（媒体大小、请求偏移）。这个区别对大磁盘很重要；10 TB的媒体大小需要超过32位。

### disk_alloc在任何时候调用都安全吗？

`disk_alloc`使用`M_WAITOK`，在内存紧张时会睡眠。不要在持有自旋锁或你不能释放的互斥锁时调用它。在附加时、任何锁之外调用它。

### 如果我用相同的名称调用disk_create两次会怎样？

如果单元号不同，`disk_create`会愉快地创建多个同名的磁盘。如果名称和单元号都匹配，GEOM将拒绝第二次注册，产生的行为是实现定义的。避免这种情况。

### 策略函数可以睡眠吗？

技术上可以，但不应该。策略函数在调用者的线程上下文中运行，在那里睡眠会阻塞调用者。对于必须阻塞的工作，使用工作线程。

### 我怎么知道给定文件系统的所有BIO何时完成？

你通常不需要知道。`umount(2)`做这个工作：它刷新脏缓冲区、排干飞行中的BIO，只有在文件系统完全静默后才返回。`umount`返回后，不会有BIO再到达那个挂载点，除非有其他东西打开了设备。

### 我可以通过bio_caller1或类似字段在线程之间传递指针吗？

可以。`bio_caller1`和`bio_caller2`是不透明字段，供BIO的发布者存放完成处理器可以使用的上下文。只要你拥有BIO（你确实拥有，因为你发出了它），这些字段就是你的。`g_disk`驱动程序通常不需要它们，因为BIO从上面到达并通过调用`biodone`完成，`g_disk`处理回调路由。

### 我的驱动程序在笔记本上工作但在服务器上不行。为什么？

可能原因：不同的内核ABI（针对服务器的内核重新编译）、不同的`MAXPHYS`（在14.3系统上应该相同但检查一下）、加载了不同的GEOM类（不太可能但可能）、不同的内存大小（你的分配在较小的系统上可能失败）、不同的时钟速度（影响时序）。从比较`uname -a`和`sysctl -a | grep kern.maxphys`开始。

### /dev节点的名称实际上从哪里来？

来自你传递给`disk_create`的`struct disk`中的`d_name`和`d_unit`。GEOM将它们连接起来不加分隔符：`d_name = "myblk"`、`d_unit = 0`产生`/dev/myblk0`。如果你想要不同的约定，相应地设置`d_name`。名称和单元号之间没有分隔符。

### 我可以创建的最大单元数是多少？

受`d_unit`限制，它是`u_int`，理论上为2^32 - 1。在实践中，每单元的内存消耗和`/dev`命名空间的实际限制会在你达到那个数字之前很久就阻止你。

### 我可以在disk_create之后更改d_mediasize吗？

可以，但要小心。挂载在磁盘上的文件系统不会自动拾取更改；大多数需要卸载并重新挂载。`md(4)`支持`MDIOCRESIZE`，有向GEOM发出更改信号的基础设施，但这个模式不简单。

### 如果我忘记MODULE_DEPEND会怎样？

如果`g_disk`尚未加载，内核可能无法加载你的模块，或者如果`g_disk`碰巧内建在内核中，可能成功加载。始终显式声明`MODULE_DEPEND`以避免意外。

### 我的驱动程序中应该使用biodone还是g_io_deliver？

使用`biodone`。`g_disk`包装器提供了`d_strategy`风格的接口，其中正确的完成调用是`biodone`。如果你编写自己的`g_class`，你将改为调用`g_io_deliver`，但那是不同的路径和不同章节的复杂度。

### BIO_DELETE与TRIM和UNMAP有什么关系？

`BIO_DELETE`是内核内的抽象。对于SATA SSD，它映射到ATA TRIM命令；对于SCSI/SAS，映射到UNMAP；对于NVMe，映射到带释放位的Dataset Management。用户态通过`fstrim(8)`或UFS上的`-o trim`挂载选项触发它。我们的驱动程序可以自由地将其视为提示或通过清零内存来响应它，因为后备存储在RAM中。

### 为什么我的策略函数有时收到bio_length为零的BIO？

在正常操作中你不应该看到这个。如果发生了，将其视为防御性情况：不带错误调用`biodone(bp)`并返回。长度为零的BIO不是非法的，但表示上游有奇怪的东西。对发出代码提交PR是合理的。

### d_flags和bio_flags之间有什么区别？

`d_flags`是整个磁盘的静态配置，在注册时设置一次，描述驱动程序能做什么（处理DELETE、可以FLUSH、接受未映射BIO等）。`bio_flags`是单个BIO上的动态元数据，每个请求变化（有序、未映射、直接完成）。不要混淆它们。

### 我的驱动程序可以将自己表现为可移动介质吗？

可以，设置`DISKFLAG_CANDELETE`并考虑响应`disk_gone`来模拟弹出。像`camcontrol`和文件系统处理器这样的工具通常统一对待任何GEOM提供者，所以用户可见意义上的"可移动"在其他操作系统中不那么明显。

### 什么线程实际调用我的策略函数？

取决于情况。对于来自缓冲区缓存的同步提交，是调用`bwrite`或`bread`的线程。对于异步完成路径，通常是GEOM工作线程或缓冲区缓存的刷新线程。你的策略函数必须编写为容忍任何调用者。不要假设特定的线程标识或特定的优先级。

### 我怎么知道哪个进程导致了给定的BIO？

你通常无法知道，因为BIO可以被重新排序、合并、归并，并从不是原始请求者的后台线程发出。带有`io:::start`探测加上栈捕获的`dtrace`可以让你接近，但这是调查工作，不是常规的驱动程序责任。

### 我的驱动程序的两个不同单元号上可以同时挂载两个不同的文件系统吗？

可以，如果你实现了多单元支持。每个单元呈现自己的GEOM提供者。它们的后备存储是独立的。唯一共享的状态是你模块的全局变量和内核本身，所以两个挂载不会交互，除非你让它们交互。

### 我的驱动程序应该处理电源管理事件吗？

对于伪设备，不需要。对于真正的硬件驱动程序，需要：挂起和恢复事件通过Newbus作为方法调用传递，驱动程序必须在挂起时停止I/O，在恢复时重新验证设备状态。笔记本电脑上的存储驱动程序是挂起相关bug的常见来源，所以真正的驱动程序对此很认真。

### 选择512还是4096作为d_sectorsize有什么实际影响？

在现代文件系统上，几乎没有：UFS、ZFS和大多数其他FreeBSD文件系统都能愉快地使用两者。在驱动程序侧，较大的扇区大小减少了大传输的BIO数量。在工作负载侧，做O_DIRECT或对齐I/O的应用程序可能在乎。有疑问时，为新驱动程序选择4096；它匹配现代闪存并避免对齐惩罚。

### 如果我多次重新加载驱动程序，内存会泄漏吗？

只有在你有bug时才会。在我们的设计中，`MOD_UNLOAD`调用`myblk_detach_unit`，它释放后备存储和softc。持久性变体故意在重载间保留后备存储，但使用单一全局指针，所以没有泄漏；相同的内存被重用。如果`vmstat -m | grep myblk`在重载间攀升，请调查。

### 为什么`mount`有时在我的原始设备上成功但`newfs_ufs`失败？

`newfs_ufs`写入结构化元数据（超级块、柱面组、inode表）然后回读其中一些来验证。如果设备太小、静默损坏写入或仅在特定条件下返回错误，`newfs_ufs`会首先捕获它。`mount`在写入路径上不那么严格；它可以读入损坏的超级块并在之后产生奇怪的错误。成功的`newfs`是比成功的`mount`更强的正确性信号。

### 我怎么验证我的BIO_FLUSH实现确实使数据持久？

对于我们的内存驱动程序，持久性受主机电源约束：刷新没有实际用途，因为断电会带走一切。对于由持久存储支持的真正驱动程序，向底层介质发出刷新命令并在调用`biodone`之前确认完成就是契约。测试需要断电循环工具或模拟器；没有捷径。

### d_strategy内部正确的锁定规则是什么？

持有驱动程序的锁足够长时间来保护后备存储免受并发访问，并在调用`biodone`之前释放它。永远不要在调用另一个子系统时持有锁。永远不要在持有锁时调用`malloc(M_WAITOK)`。永远不要睡眠。如果你需要睡眠，在工作线程上调度工作并从工作线程调用`biodone`。

### 为什么BIO_FLUSH不像Linux上的写屏障那样是百分比容量的屏障？

FreeBSD的BIO_FLUSH是时间点屏障：当它完成时，所有先前发出的写入都是持久的。它不与特定范围或设备的百分比关联。驱动程序可以将它实现为严格屏障或机会性刷新，但最低契约是时间点保证。

### 有生成BIO流量帮助我测试的工具吗？

有。带有各种`bs=`值的`dd(1)`、来自ports的`fio(1)`、来自ports的`ioping(8)`，加上通常的嫌疑人：`newfs`、`tar`、`rsync`、`cp`。`diskinfo -t`运行一套基准读取，对粗略的吞吐量数字有用。`/usr/src/tools/regression/`下的测试工具也可以被适配。

## 本章未涵盖的内容

本章很长，但有几个相关主题我们有意留待以后。在这里命名它们有助于你规划未来的学习，并防止产生存储驱动程序止步于BIO处理器的错误印象。

**真正的硬件存储驱动程序**，如SATA、SAS和NVMe控制器的驱动程序，位于CAM之下，需要大量额外的机制：命令块分配、标记队列、热插拔事件处理、固件上传、SMART数据和错误恢复协议。我们通过`ada_da.c`摘录简要介绍了CAM世界，但没有深入探讨。第33到36章将讨论这些接口，你在本章阅读的`md(4)`驱动程序相比之下是一个刻意设置的小台阶。

**ZFS集成**是一个独立的世界。ZFS通过其vdev层消费GEOM提供者，但添加了写时复制语义、端到端校验和、池化存储和快照，这些是简单块驱动程序永远不需要知道的。如果你的驱动程序在UFS下工作，它几乎肯定在ZFS下也工作，但反过来不一定：ZFS对BIO路径的要求，特别是刷新和写入顺序，是不那么苛刻的文件系统跳过的。

**GEOM类编写**是比`g_disk`包装更大的主题。完整的类实现taste、start、access、attach、detach、dumpconf、destroy_geom和orphan方法。它还可以创建和销毁消费者、构建多层拓扑、通过`gctl`响应配置。一旦你决定深入研究，mirror、stripe和crypt类是好的起点。

**配额、ACL和扩展属性**是完全位于GEOM层之上的文件系统功能。它们对用户空间很重要但不涉及存储驱动程序。这是一个有用的澄清：驱动程序的工作止于BIO边界。

**跟踪和调试内核崩溃**值得单独一章。内核核心转储通过`dumpon(8)`配置的转储设备落地，用`kgdb(1)`或`crashinfo(8)`分析。如果你的驱动程序导致系统崩溃，能够加载核心文件并检查回溯是一项专业级技能，本章只是提及。

**高性能存储路径**使用未映射I/O、直接分派完成、CPU固定、NUMA感知分配和专用队列等特性。这些优化对每秒千兆字节的工作负载很重要，但对教学驱动程序无关。当你开始追逐微秒时，回到`/usr/src/sys/dev/nvme/`研究真正的专业人士是怎么做的。

**文件系统特定行为**差异很大。UFS请求一组BIO；ZFS请求不同的组；msdosfs和ext2fs又请求不同的东西。一个好的存储驱动程序是文件系统无关的，但在你的驱动程序上观察不同文件系统是建立直觉的绝佳方式。在你对UFS感到舒适后，尝试`msdosfs`、`ext2fs`和`tmpfs`进行对比。

**iSCSI和网络块设备**也以GEOM提供者的形式呈现自己，但它们由用户态控制守护程序创建并与网络栈通信。第28章开始使这些提供者成为可能的网络工作。

我们对存储路径的处理是刻意聚焦的。我们编写了一个文件系统接受为真实的驱动程序，我们理解了为什么以及如何被这样看待，我们跟踪了从`write(2)`到RAM的数据路径。这个基础足以使上述未探索的主题变得可读而非令人困惑。

## 最终反思

存储驱动程序有令人生畏的名声。本章应该已经用熟悉感取代了部分这种名声：BIO只是一个结构，策略函数只是一个分派器，GEOM只是一个图，`disk_create`只是一个注册调用。将存储工作从例行公事中提升的不是底层API——它们很紧凑——而是围绕它们积累的运维需求：性能、持久性、错误恢复和竞争下的正确性。

当你从伪设备转向真正的硬件时，这些需求不会消失。它们会倍增。但你已经有了理解它们的词汇。你知道BIO是什么，它从哪里来。你知道哪个线程调用你的代码以及它期望什么。你知道如何向GEOM注册、如何干净地注销，以及如何从`gstat`中的影子识别飞行中的请求。当你坐在真正的SATA控制器驱动程序前开始阅读时，你会识别出代码的形状，即使具体细节不同。

存储驱动程序编写的技艺，归根结底，是耐心的。你通过编写小驱动程序、阅读源码树、重现简单的实验、建立对看起来正确的东西什么时候实际上正确的直觉来学习。你刚完成的章节是这条旅程中的漫长一步。接下来的章节将再次迈步，每次朝着不同的方向。

## 延伸阅读

如果本章激发了你对存储内部机制的兴趣，以下是一些可以继续深入的地方。

**手册页**。`disk(9)`、`g_bio(9)`、`geom(4)`、`devfs(5)`、`ufs(5)`、`newfs(8)`、`mdconfig(8)`、`gstat(8)`、`diskinfo(8)`、`mount(2)`、`mount(8)`。按此顺序阅读。

**FreeBSD架构手册**。存储章节是本章的良好补充。

**Kirk McKusick等人，"The Design and Implementation of the FreeBSD Operating System"。**该书中关于文件系统的章节特别相关。

**DTrace书籍。**Brendan Gregg的"DTrace Book"是实用的参考；Sun的"Dynamic Tracing Guide"是原始教程。

**FreeBSD源码树。**`/usr/src/sys/geom/`、`/usr/src/sys/dev/md/`、`/usr/src/sys/ufs/`和`/usr/src/sys/cam/ata/`（`ata_da.c`在其中实现`ada`磁盘驱动程序）。本章讨论的每个模式都根植于该代码。

**邮件列表档案。**`freebsd-geom@`和`freebsd-fs@`是两个最相关的列表。阅读历史帖子是获取书籍未捕获的制度知识的最佳方式之一。

**GitHub镜像上的提交历史。**FreeBSD源码树有很长、注释良好的提交历史。对于你打开的任何文件，对其镜像运行`git log --follow`通常会揭示设计选择背后的原因、塑造当前代码的bug以及维护它的人。历史背景使现在的代码更容易阅读。

**FreeBSD开发者峰会论文集。**几次峰会都包含存储相关的会议。录音和幻灯片（如果有）非常适合了解最新技术和公开的设计辩论。

**阅读其他操作系统的存储栈。**一旦你了解了FreeBSD的存储路径，Linux的块层、Illumos的SD框架和macOS的IOKit存储类都变得可以理解了，而以前可能不是这样。具体API不同，但基本的形状——BIO或其等价物、上面是文件系统、下面是硬件——是通用的。

**内核代码的测试框架。**`kyua(1)`工具运行针对真实内核的回归测试。`/usr/tests/sys/geom/`树有编写良好的存储代码测试的例子；阅读它们既能建立测试直觉也能增强代码正确的信心。

**FreeBSD基金会博客文章。**基金会资助多个存储相关项目，并发布可读的摘要来补充源码树。

---

第27章结束。合上你的实验日志，确保你的驱动程序已卸载、挂载点已释放，在第28章之前休息一下。

你刚刚编写了一个存储驱动程序，在其上挂载了文件系统，跟踪了数据穿过缓冲区缓存、进入GEOM、通过你的策略函数然后返回。这是一项真正的成就。在翻页之前好好感受一下。
