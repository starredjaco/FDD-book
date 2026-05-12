---
title: "与内核集成"
description: "第24章将myfirst驱动程序从1.6-debug版本扩展到1.7-integration版本。本章教授驱动程序从一个独立的内核模块转变为FreeBSD内核成员的含义。本章解释了集成为何重要；驱动程序如何在devfs中生存，/dev节点如何出现、获取权限、重命名和消失；如何实现用户空间可以依赖的ioctl()接口，包括_IO/_IOR/_IOW/_IOWR编码和内核的自动copyin/copyout层；如何通过根植于dev.myfirst.N下的动态sysctl树公开驱动程序指标、计数器和可调参数；如何以介绍性层面思考通过ifnet(9)将驱动程序挂钩到网络栈，以if_tuntap.c作为参考；如何以介绍性层面思考通过cam_sim_alloc和xpt_bus_register挂钩到CAM存储子系统；如何组织注册、附加、拆卸和清理，以便集成的路径可以在压力下干净地加载和卸载；以及如何将驱动程序重构为可维护、版本化的包，以便后续章节可以继续扩展。该驱动程序获得了myfirst_ioctl.c、myfirst_ioctl.h、myfirst_sysctl.c和一个小型配套测试程序；获得了一个支持克隆的/dev/myfirst0节点，带有每个实例的sysctl子树；第24章结束后，该驱动程序成为其他软件可以以FreeBSD原生方式与之通信的驱动程序。"
partNumber: 5
partName: "调试、工具和实际实践"
chapter: 24
lastUpdated: "2026-04-19"
status: "complete"
author: "Edson Brandi"
reviewer: "TBD"
translator: "AI辅助翻译为简体中文"
estimatedReadTime: 210
language: "zh-CN"
---

# 与内核集成

## 读者指导与学习目标

第23章以一个能够自我解释的驱动程序结束。版本为`1.6-debug`的`myfirst`驱动程序知道如何通过`device_printf`记录结构化消息，如何通过运行时sysctl掩码控制详细输出，如何向DTrace公开静态探针点，以及如何留下操作员可以稍后阅读的记录。结合第22章添加的电源管理、第21章添加的DMA管道、第19章和第20章添加的中断机制，该驱动程序现在本身就是一个完整的单元：它可以启动、运行、与真实的PCI设备通信、在挂起和恢复后存活，并告诉开发者它正在做什么。

该驱动程序尚未做到的是像一个更广泛的内核成员那样行事。外部程序能够看到、控制或测量的`myfirst`内容仍然很少。模块加载时驱动程序只创建一个设备节点。用户空间工具无法要求驱动程序重置自身、在运行时翻转配置旋钮或读出计数器。系统的sysctl树中没有操作员可以提供给监控系统的指标。无法干净地为驱动程序提供多个实例。没有与网络栈、存储栈或任何用户通常从自己的程序中访问的内核子系统集成。在所有意义上，该驱动程序仍然独自站在角落里。第24章是将其带入房间的章节。

第24章以适合本书这一阶段的正确水平教授内核集成。读者将在本章学习集成实际上意味着什么、为什么它比最初看起来更重要、以及如何构建每个集成表面。本章从概念性叙述开始：一个工作的驱动程序与一个集成的驱动程序之间的区别，以及将集成作为事后想法的成本。然后，它将大部分时间花在典型FreeBSD驱动程序始终与之集成的四个接口上：设备文件系统`devfs`、用户控制的`ioctl(2)`通道、系统范围的`sysctl(8)`树，以及用于干净的附加、分离和模块卸载的内核生命周期钩子。在这四个之后是两个可选的小章节，一个用于硬件是网络设备的驱动程序，另一个用于硬件是存储控制器的驱动程序。两者都在概念层面介绍，以便读者在第6部分和第7部分中遇到它们时能够识别，两者都没有完全教授，因为每个最终都值得拥有自己的章节。然后本章退后一步，讨论所有这些表面的注册和拆卸规则：顺序很重要、失败路径很重要、压力下的边界情况很重要，一个正确实现集成接口但错误实现生命周期的驱动程序仍然是一个脆弱的驱动程序。最后，本章以重构结束，将新代码拆分到自己的文件中，将驱动程序升级到版本`1.7-integration`，更新版本横幅，并组织源代码树以供后续内容使用。

第5部分的主线在这里延续。第22章使驱动程序能够在电源状态更改后存活。第23章使驱动程序能够告诉你它在做什么。第24章使驱动程序能够自然地融入系统的其余部分，以便FreeBSD用户已经知道的工具和习惯可以毫无意外地延续到你的驱动程序中。第25章将通过教授维护规则继续这一主线，使驱动程序在演进过程中保持可读、可调和可扩展，第6部分将开始依赖于第5部分各章所建立的每个品质的传输特定章节。

### 为什么devfs、ioctl和sysctl集成值得单独一章

这里出现的一个问题是，连接`devfs`、`ioctl`和`sysctl`是否真的值得整整一章。驱动程序从很早的章节就已经有了一个单独的`cdev`节点。添加ioctl看起来很小。添加sysctl看起来更小。为什么要在一整章中分散这项工作，而每个接口看起来只需要几十行代码？

答案是，几十行的观点只是容易的部分。每个接口都有一组从API阅读中看不明显的约定和陷阱，而犯错的代价不是由开发者承担，而是由尝试监控驱动程序的操作员、尝试重置设备的用户、尝试在负载下加载和卸载模块的打包者，以及六个月后尝试扩展驱动程序的下一个开发者承担。第24章把这些时间花在这些约定和陷阱上，因为那就是价值所在。

本章赢得其位置的第一原因是**集成接口是其他所有东西到达驱动程序的方式**。跟随本书到这里的读者已经构建了一个做有趣工作的驱动程序，但目前只有内核本身知道如何要求驱动程序执行该工作。一旦驱动程序有了ioctl接口，shell脚本就可以驱动它。一旦驱动程序有了sysctl树，监控系统就可以监视它。一旦驱动程序创建了遵循标准约定的`/dev`节点，打包者就可以发布udev风格的规则，系统管理员可以为它编写`/etc/devfs.rules`，另一个驱动程序可以通过`vop_open`或`ifnet`层叠在其上。所有这些都不取决于驱动程序的用途；所有这些都取决于集成是否做得好。

第二原因是**集成选择会在生产故障模式中显现**。从错误的上下文调用`make_dev`的驱动程序可能会在模块加载时死锁。省略`_IO`、`_IOR`、`_IOW`、`_IOWR`规则的驱动程序强制每个调用者发明一个私有约定来决定谁在用户-内核边界之间复制什么，而这些调用者中至少有一个会弄错。忘记在分离时调用`sysctl_ctx_free`的驱动程序会泄漏OID，下一次使用相同名称的模块加载会失败并显示令人困惑的消息。在排空打开的文件句柄之前销毁其`cdev`的驱动程序会产生释放后使用的崩溃。第24章在每一项上花费了段落，因为每一项都是FreeBSD社区多年来不得不追踪的真实错误，学习规则的正确时机是在编写第一行集成代码之前，而不是之后。

第三原因是**集成代码是驱动程序的设计对作者以外的人可见的第一个地方**。在第24章之前，驱动程序是一个只有一个方法表和一个设备节点的黑盒。从第24章开始，驱动程序有一个公共表面。其sysctl的名称出现在监控图表中。其ioctl的编号出现在shell脚本和用户空间库中。其`/dev`节点的布局出现在包文档和管理员运行手册中。一旦公共表面存在，更改它就有代价。因此，本章注意教授使表面在驱动程序演进时保持稳定的约定。升级到`1.7-integration`也是驱动程序第一个具有真实公共面的版本；之前的所有内容都是内部里程碑。

第24章通过将这三个想法与`myfirst`驱动程序作为运行示例一起具体教授来赢得其位置。完成第24章的读者可以将任何FreeBSD驱动程序集成到标准系统接口中，知道每个集成表面的约定和陷阱，可以阅读另一个驱动程序的集成代码并识别什么是正常的什么是不寻常的，并且拥有其他软件最终可以与之通信的`myfirst`驱动程序。

### 第23章留给驱动程序的内容

在真正的工作开始之前进行简短的检查。第24章扩展了第23章末尾生成的驱动程序，标记为版本`1.6-debug`。如果以下任何项目不确定，请返回第23章并在开始本章之前修复它，因为集成主题假设调试规则已经存在，几个新的集成表面将使用它。

- 你的驱动程序可以干净地编译，并在`kldstat -v`中标识为`1.6-debug`。
- 驱动程序仍然执行`1.5-power`版本所做的一切：它附加到PCI（或模拟PCI）设备，分配MSI-X向量，运行DMA管道，并在`devctl suspend myfirst0`后跟`devctl resume myfirst0`后存活。
- 驱动程序在磁盘上有`myfirst_debug.c`和`myfirst_debug.h`对。头文件定义了`MYF_DBG_INIT`、`MYF_DBG_OPEN`、`MYF_DBG_IO`、`MYF_DBG_IOCTL`、`MYF_DBG_INTR`、`MYF_DBG_DMA`、`MYF_DBG_PWR`和`MYF_DBG_MEM`。`DPRINTF(sc, MASK, fmt, ...)`宏可以从驱动程序中的任何源文件使用。
- 驱动程序有三个名为`myfirst:::open`、`myfirst:::close`和`myfirst:::io`的SDT探针。简单的DTrace一行程序`dtrace -n 'myfirst::: { @[probename] = count(); }'`在设备被使用时返回计数。
- softc携带一个`uint32_t sc_debug`字段，`sysctl dev.myfirst.0.debug.mask`可以读取和写入它。
- 驱动程序在源代码旁边有一个`DEBUG.md`文档。`HARDWARE.md`、`LOCKING.md`、`SIMULATION.md`、`PCI.md`、`INTERRUPTS.md`、`MSIX.md`、`DMA.md`和`POWER.md`在你工作树中从早期章节开始也是最新的。
- 你的测试内核仍然启用了`INVARIANTS`、`WITNESS`、`WITNESS_SKIPSPIN`、`DDB`、`KDB`、`KDB_UNATTENDED`、`KDTRACE_HOOKS`和`DDB_CTF`。第24章的实验使用相同的内核。

这就是第24章扩展的驱动程序。添加的内容在代码行数上比第23章大，但在概念表面上更小。新的部分是：一个更丰富的`/dev/myfirst0`节点，可以按需克隆实例，一组在用户空间程序可以包含的公共头文件中定义的类型良好的ioctl，`dev.myfirst.N.`下的每个实例sysctl子树，公开少量指标和一个可写旋钮，将新代码拆分到`myfirst_ioctl.c`和`myfirst_sysctl.c`中并带有匹配头文件的重构，一个名为`myfirstctl`的小型配套用户空间程序，用于测试新接口，源代码旁边的`INTEGRATION.md`文档，更新的回归测试，以及版本升级到`1.7-integration`。

### 你将学到什么

完成本章后，你将能够：

- 用具体的FreeBSD术语解释内核集成意味着什么，区分独立的驱动程序与集成的驱动程序，并命名每个集成表面提供的特定用户可见的好处。
- 描述`devfs`是什么，它与旧的静态`/dev`方案有何不同，以及设备节点在它下面如何出现和消失。正确使用`make_dev`、`make_dev_s`、`make_dev_credf`和`destroy_dev`。为节点选择正确的标志集、所有权和模式。
- 使用现代的`D_VERSION`字段、最小回调集和可选回调（`d_kqfilter`、`d_mmap_single`、`d_purge`）初始化和填充`struct cdevsw`。
- 使用`cdev->si_drv1`和`si_drv2`字段附加每个节点的驱动程序状态，并从cdevsw回调内部读回该状态。
- 从单个驱动程序实例创建多个设备节点，并通过`dev_clone`事件处理程序在固定名称节点、索引节点和可克隆节点之间进行选择。
- 在创建时设置每个节点的权限和所有权，并在创建后通过`devfs.rules`调整它们，以便管理员可以在不重建驱动程序的情况下授予访问权限。
- 解释`ioctl(2)`是什么，内核如何使用`_IO`、`_IOR`、`IOW`和`_IOWR`编码ioctl命令，每个宏对数据流方向意味着什么，以及为什么正确编码对于32位和64位用户空间之间的可移植性很重要。
- 为驱动程序定义公共ioctl头文件，选择一个空闲的魔术字母，并记录每个命令，以便用户空间调用者可以跨版本依赖该接口。
- 实现一个`d_ioctl`处理程序，根据命令字分发，安全地执行每个命令的逻辑，并在每个失败路径上返回正确的errno。
- 阅读和理解内核对ioctl数据的自动`copyin`/`copyout`层，并识别驱动程序仍然必须自己复制内存的情况：可变长度负载、嵌入的用户指针以及布局需要显式对齐的结构。
- 解释`sysctl(9)`，区分静态OID和动态OID，并遍历`device_get_sysctl_ctx`和`device_get_sysctl_tree`模式，该模式给每个设备自己的子树。
- 使用`SYSCTL_ADD_UINT`和`SYSCTL_ADD_QUAD`添加只读计数器，使用适当的访问标志添加可写旋钮，并使用`SYSCTL_ADD_PROC`添加自定义过程OID，用于必须在读取时计算值的情况。
- 管理用户可以在`/boot/loader.conf`中使用`TUNABLE_INT_FETCH`设置的可调参数，并结合可调参数和sysctl，以便相同的配置旋钮可以在启动时设置或在运行时调整。
- 识别FreeBSD网络集成的介绍性形式：`if_alloc`、`if_initname`、`if_attach`、`bpfattach`、`if_detach`和`if_free`如何组合；概念层面上`if_t`是什么；驱动程序在更大的ifnet机制中扮演什么角色。理解第28章将深入讨论这一点。
- 识别FreeBSD存储集成的介绍性形式：CAM是什么，`cam_sim_alloc`、`xpt_bus_register`、`sim_action`回调和`xpt_done`如何组合；概念层面上CCB是什么；以及CAM为什么存在。理解第27章的存储驱动程序和第27章的GEOM材料将深入讨论这一点。
- 应用在重复`kldload`/`kldunload`下、在启动中途附加失败时、在用户仍持有打开的文件描述符时分离失败时、以及在底层设备真正意外移除时都健壮的注册和拆卸规则。
- 将积累了几个集成表面的驱动程序重构为可维护的结构：每个集成关注点一个单独的文件，一个用于用户空间的公共头文件，一个用于驱动程序内部使用的私有头文件，以及一个将所有部分编译成单个内核模块的更新构建系统。

列表很长，因为集成涉及多个子系统。每一项都是狭窄且可教学的。本章的工作是将它们组合成一个连贯的驱动程序。

### 本章不涵盖的内容

为了使第24章专注于集成规则，几个相邻主题被明确推迟。

- **ifnet网络驱动程序的完整实现**，包括发送和接收队列、通过`iflib(9)`的多队列协调、超出介绍性`bpfattach`调用的BPF集成、链路状态事件和完整的以太网驱动程序生命周期。第28章是专门的网络驱动程序章节，假设第24章的集成规则已经到位。
- **CAM存储驱动程序的完整实现**，包括目标模式、完整的CCB类型集、通过`xpt_setup_ccb`和`xpt_async`的异步通知，以及通过`disk_create`或GEOM的几何呈现。第27章深入涵盖存储栈。
- **GEOM集成**，包括提供者、消费者、类、`g_attach`、`g_detach`和GEOM事件机制。GEOM是具有自己约定的自己的子系统；第27章涵盖它。
- **基于`epoch(9)`的并发**，这是ifnet热路径的现代锁定模式。第24章仅在上下文中提及它。第28章（网络驱动程序）将与`iflib(9)`一起回顾它，其中需要实践中的epoch风格并发。
- **`mac(9)`（强制访问控制）集成**，它在集成表面周围添加策略钩子。MAC框架是一个专业主题，尚不适用于简单的`myfirst`驱动程序。
- **`vfs(9)`集成**，这是文件系统所做的。字符驱动程序不在`vop_open`或`vop_read`层与VFS交互；它与`cdevsw`和`devfs`交互。本章注意不要混淆两者。
- **通过`kobj(9)`和使用`INTERFACE`构建机制声明的自定义接口进行的跨驱动程序接口**。这些是网络和存储栈定义其内部契约的方式。它们在第7节中在上下文中提及，但深入处理属于后面的更高级章节。
- **新的`netlink(9)`接口**，最近的FreeBSD内核为某些网络管理流量公开。Netlink目前由路由子系统使用，而不是由单个设备驱动程序使用，教授它的正确位置是与网络章节一起。
- **通过`pr_protocol_init`的自定义协议模块**，这是用于新的传输协议而不是设备驱动程序。

保持在那些界限内使第24章成为关于驱动程序如何成为内核一部分的章节，而不是关于驱动程序可能最终触及的每个内核子系统的章节。

### 预计时间投入

- **仅阅读**：四到五个小时。第24章的想法主要是读者已经遇到过的概念扩展。新词汇（devfs、cdev、ioctl、sysctl、ifnet、CAM）从早期章节开始大多以名称熟悉；本章的工作是给它们每个一个具体的形状。
- **阅读加输入示例代码**：在两到三次会话中十到十二个小时。驱动程序依次通过三个集成表面（devfs、ioctl、sysctl）演进，每个都有自己的短阶段。每个阶段都很短且自包含；测试是花费时间的地方，因为集成表面最好通过编写驱动它们的小型用户空间程序来测试。
- **阅读加所有实验和挑战**：在三到四次会话中十五到十八个小时。实验包括可克隆devfs实验、使用`myfirstctl`的完整ioctl往返、sysctl驱动的计数器监控练习、有意破坏拆卸以暴露失败模式的清理规则实验，以及为想要预览第28章的读者准备的小型ifnet存根挑战。

第3节和第4节是新词汇最密集的部分。ioctl宏和sysctl回调签名是本章中唯一真正新的API；其余是组合。如果宏在第一次阅读时感觉不透明，那是正常的。停下来，在驱动程序上运行匹配练习，然后在形状稳定后回来。

### 先决条件

在开始本章之前，确认：

- 你的驱动程序源代码与第23章阶段3（`1.6-debug`）匹配。起点假设每个第23章原语：`DPRINTF`宏、SDT探针、调试掩码sysctl和`myfirst_debug.c`/`myfirst_debug.h`文件对。第24章构建在适当时候使用每一个的新集成代码。
- 你的实验机器运行FreeBSD 14.3，磁盘上有`/usr/src`并与运行的内核匹配。
- 已构建、安装并干净启动的调试内核，启用了`INVARIANTS`、`WITNESS`、`WITNESS_SKIPSPIN`、`DDB`、`KDB`、`KDB_UNATTENDED`、`KDTRACE_HOOKS`和`DDB_CTF`。
- `bhyve(8)`或`qemu-system-x86_64`可用，并且你在`1.6-debug`状态有一个可用的VM快照。第24章的实验包括针对清理规则部分的有意失败场景，快照使恢复变得廉价。
- 以下用户空间命令在你的路径中：`dmesg`、`sysctl`、`kldstat`、`kldload`、`kldunload`、`devctl`、`cc`、`make`、`dtrace`、`dd`、`head`、`cat`、`chown`、`chmod`和`truss`。第24章的实验轻度使用`truss`，即FreeBSD中相当于Linux `strace`的工具，以验证用户空间程序确实通过新ioctl到达驱动程序。
- 你习惯于针对FreeBSD `libc`头文件编写短C程序。本章通过一个名为`myfirstctl`的小程序介绍新ioctl的用户空间端。
- 对`git`的工作知识有帮助但不是必需的。本章建议你在阶段之间提交，以便驱动程序的每个版本都有可恢复的点。

如果以上任何项目不稳定，现在就修复它。集成代码平均比前面章节的内核模式工作危险性小，因为大多数失败模式在模块加载或用户空间调用时被捕获，而不是产生内核崩溃。但教训会累积：本章中的可克隆`make_dev`错误将在第28章中产生丑陋的诊断，当网络驱动程序也想要自己的节点时，本章中的sysctl OID泄漏将在第27章中产生令人困惑的模块加载失败，当存储驱动程序尝试注册已存在的名称时。

### 如何充分利用本章

五个习惯在本章比第5部分的任何先前章节都更有价值。

首先，将`/usr/src/sys/dev/null/null.c`、`/usr/src/sys/sys/conf.h`、`/usr/src/sys/sys/ioccom.h`和`/usr/src/sys/sys/sysctl.h`加入书签。第一个是FreeBSD源代码树中最短的非平凡字符驱动程序，是`cdevsw`/`make_dev`/`destroy_dev`模式的规范示例。第二个声明了`cdevsw`结构、`make_dev`系列和本章重复使用的`MAKEDEV_*`标志位。第三个定义了ioctl编码宏（`_IO`、`_IOR`、`_IOW`、`_IOWR`）并包含内核用于决定是否自动复制数据的`IOC_VOID`、`IOC_IN`、`IOC_OUT`和`IOC_INOUT`常量。第四个定义了sysctl OID宏、`SYSCTL_HANDLER_ARGS`调用约定以及静态和动态OID接口。这些文件都不长；最长的是几千行，其中大部分是注释。在相应部分开始时每个阅读一次是你能为流利度做的最有效的事情。

其次，保持三个真实驱动程序示例近在咫尺：`/usr/src/sys/dev/null/null.c`、`/usr/src/sys/net/if_tuntap.c`和`/usr/src/sys/dev/virtio/block/virtio_blk.c`。第一个是最小的cdevsw示例。第二个是第5节用于介绍ifnet的规范可克隆`dev_clone`示例。第三个说明了完整的基于`device_get_sysctl_ctx`的动态sysctl树和切换运行时旋钮的`SYSCTL_ADD_PROC`回调。第24章在适当时刻指向每一个。现在阅读它们一次，不尝试记忆，给本章的其余部分提供具体的锚点来悬挂其想法。

> **关于行号的说明。** 本章后面指向`null.c`、`if_tuntap.c`和`virtio_blk.c`的指针锚定在命名符号上：特定的`make_dev`调用、`SYSCTL_ADD_PROC`处理程序、特定的`cdevsw`。这些名称在未来的FreeBSD 14.x版本中延续。每个名称所在的具体行号不会。当散文引用某个位置时，打开文件并搜索符号，而不是滚动到数字。

第三，手动将每个代码更改输入`myfirst`驱动程序。集成代码是容易复制但以后很难记住的代码。手动输入`cdevsw`表、ioctl命令定义、sysctl树构造和用户空间`myfirstctl`程序建立了复制粘贴无法建立的那种熟悉度。目标不是拥有代码；目标是成为可以在二十分钟内当未来的错误需要时从头再次编写它的人。

第四，在每个阶段构建用户空间程序。本章的许多教训仅在用户端可见。内核是否正确复制了ioctl负载、sysctl是否可读但不可写、`/dev`节点是否具有你设置的权限、克隆是否产生可用的设备节点，所有这些问题都用`cat`、`dd`、`chmod`、`sysctl`、`truss`和小型`myfirstctl`配套程序回答。仅通过内核端计数器测试的驱动程序只被测试了一半。

第五，完成第4节后重读第23章的调试规则。第24章中的每个集成表面都包装了来自第23章的`DPRINTF`。每个ioctl路径触发`MYF_DBG_IOCTL`日志行。每个sysctl路径可以通过SDT机制观察。看到第23章的工具如何服务于第24章的接口加强了这两章，并为第25章做准备，其中相同的模式继续。

### 本章路线图

各节的顺序是：

1. **为什么集成很重要。** 概念性叙述。从独立模块到系统组件；将集成作为事后想法的代价；每个集成驱动程序最终需要的四个用户可见接口；本章介绍但未完成的可选子系统钩子。
2. **使用devfs和设备树。** 内核对`/dev`的看法。devfs是什么以及它如何与旧系统的静态设备表不同；`cdev`的生命周期；详细的`make_dev`及其系列；`cdevsw`结构及其回调；`si_drv1`/`si_drv2`字段；权限和所有权；通过`dev_clone`事件处理程序的可克隆节点。第24章驱动程序的第1阶段用干净的可克隆感知模式替换原始的临时节点创建。
3. **实现`ioctl()`支持。** 用户驱动的控制接口。ioctl是什么；`_IO`/`_IOR`/`_IOW`/`_IOWR`编码；内核的自动copyin/copyout层；如何选择魔术字母和数字；如何布局公共ioctl头文件；如何编写根据命令字分发的`d_ioctl`回调；常见陷阱（可变长度数据、嵌入指针、版本演进）。第2阶段添加`myfirst_ioctl.c`和`myfirst_ioctl.h`以及小型`myfirstctl`用户空间程序。
4. **通过`sysctl()`公开指标。** 监控和调整接口。sysctl是什么；静态与动态OID；`device_get_sysctl_ctx`/`device_get_sysctl_tree`模式；计数器、旋钮和过程回调；`SYSCTL_ADD_*`系列；`/boot/loader.conf`中的可调参数；访问控制和单位。第3阶段添加`myfirst_sysctl.c`和每个实例的指标子树。
5. **与网络子系统集成（可选）。** ifnet的简短概念性介绍。`if_t`是什么；`if_alloc`/`if_initname`/`if_attach`/`bpfattach`/`if_detach`/`if_free`大纲；`tun(4)`和`tap(4)`如何围绕它构建；驱动程序在更大的网络栈中扮演什么角色。本节故意简短；第28章是网络驱动程序章节。
6. **与CAM存储子系统集成（可选）。** CAM的简短概念性介绍。CAM是什么；SIM和CCB是什么；`cam_sim_alloc`/`xpt_bus_register`/`xpt_action`/`xpt_done`大纲；如何通过它公开小型只读内存磁盘。本节故意简短；第27章是存储驱动程序章节。
7. **注册、拆卸和清理规则。** 跨切主题。模块事件处理程序（`MOD_LOAD`、`MOD_UNLOAD`、`MOD_SHUTDOWN`）；带有部分清理的附加失败；用户仍持有打开句柄时的分离失败；失败时清理模式；集成表面之间的顺序；`bus_generic_attach`、`bus_generic_detach`和`device_delete_children`为你做什么；用于跨切注册的SYSINIT和EVENTHANDLER。
8. **重构和版本化集成驱动程序。** 清理。最终拆分为`myfirst.c`、`myfirst_debug.c`/`.h`、`myfirst_ioctl.c`/`.h`和`myfirst_sysctl.c`/`.h`；用于用户空间调用者的公共`myfirst.h`头文件；`INTEGRATION.md`文档；升级到`1.7-integration`；回归测试添加；提交和标记。

在八个部分之后是一组实践实验，端到端地练习每个集成表面，一组挑战练习，在不引入新基础的情况下扩展读者，针对大多数读者会遇到的症状的故障排除参考，结束第24章故事并开启第25章的总结，通往下一章的桥梁，以及通常的快速参考卡和词汇表。

如果这是你的第一遍，请按顺序阅读并按顺序进行实验。如果你正在回顾，第2节和第3节独立存在，适合单次阅读。第5节和第6节简短且概念性强，可以在第一遍时跳过而不会失去本章的主线，然后在开始第26章或第27章之前返回。

在技术工作开始之前的一个小注。本章经常要求读者编译一个小型用户空间程序，针对驱动程序运行它，观察结果，然后返回内核端阅读发生了什么。这种节奏是故意的。集成不是驱动程序单独的属性；它是驱动程序与系统其余部分之间关系的属性。用户空间程序很短，但它们是本章衡量每个集成表面是否真正工作的方式。

## 第1节：为什么集成很重要

在代码之前，先看框架。第1节清楚地说明了当驱动程序成为集成时会发生什么变化。跟随第4部分和第5部分前几章的读者已经构建了一个工作的驱动程序。说这个驱动程序尚未*集成*意味着什么，集成添加了哪些具体品质？

本节仔细而详尽地回答这个问题，因为本章的其余部分是实现这些品质。清楚知道每个集成表面*为什么*存在的读者会发现第2到第8节中的实现工作容易得多。跳过框架的读者将整章想知道为什么驱动程序需要`ioctl`，而内部sysctl可以做同样的工作；这个问题有真正的答案，第1节是答案所在。

### 从独立模块到系统组件

独立的驱动程序是一个内核模块，当内核调用时正确执行其工作，否则不妨碍。当前的`myfirst`驱动程序，版本为`1.6-debug`，正是那种模块。它有一个单独的`cdev`节点，没有公共ioctl，除了少量内部调试旋钮外没有发布的sysctl，与自身小角落之外的任何内核子系统没有关系，也没有期望任何用户空间程序会介入并告诉它该做什么。它工作，而且孤立地工作。

相比之下，系统组件是一个内核模块，其价值取决于它与系统其余部分的关系。同一个`myfirst`驱动程序，集成后，呈现一个具有适合其硬件角色的正确权限的`/dev/myfirst0`节点，公开用户程序可以用来重置设备或查询其状态的小型ioctl接口，发布监控系统可以抓取的每个实例sysctl树，如果硬件是网络或存储设备，则向适当的内核子系统注册自己，并在卸载时干净地清理。这些接口中的每一个都很小。它们共同构成了在一个开发者实验室机器上运行的驱动程序与随FreeBSD发布的驱动程序之间的区别。

从独立到集成的转变不是对驱动程序的一行更改。它是一系列有意识的决定，每个决定都扩展了驱动程序的公共表面，每个决定都会产生维护成本。在时刻看起来很小的决定，比如为ioctl选择魔术字母或sysctl OID的名称，会成为长期契约。2010年为其ioctl选择字母`M`的驱动程序今天仍然在其公共头文件中有这些编号，因为更改它们会破坏曾经调用它们的每个用户空间程序。

想象驱动程序的一种有用方法是想象两个接近驱动程序的读者。第一个读者是编写驱动程序的开发者：他们对它了如指掌，可以更改其中的任何内容，并且可以随时重建它。第二个读者是从包安装模块且从不阅读其源代码的系统管理员：他们只通过驱动程序的`/dev`节点、sysctl、ioctl和日志消息看到它。独立的驱动程序是为第一个读者设计的；集成的驱动程序是为两者设计的。

第24章教读者如何为第二个读者设计。这是本章所做的概念工作，第2到第8节中的代码工作是其实际实现。

### 常见集成目标

FreeBSD驱动程序通常与四个内核端表面集成，并且根据硬件，与两个子系统之一集成。在开始时清楚地命名目标有助于读者记住本章的结构。

第一个目标是**`devfs`**。每个字符驱动程序通过`make_dev`（或其变体之一）创建一个或多个`/dev`节点，并通过`destroy_dev`删除它们。这些节点的形状和命名是系统其余部分寻址驱动程序的方式。名为`/dev/myfirst0`的节点让管理员可以用`cat /dev/myfirst0`打开它，让脚本将其包含在`find /dev -name 'myfirst*'`中，并让内核本身将打开、读取、写入和ioctl调用分发到驱动程序中。第2节专门讨论devfs。

第二个目标是**`ioctl(2)`**。`read(2)`和`write(2)`系统调用移动字节；它们不控制驱动程序。任何不适合读/写数据流模型的控制操作都存在于`ioctl(2)`中。用户空间程序调用`ioctl(fd, MYF_RESET)`要求驱动程序重置其硬件，或`ioctl(fd, MYF_GET_STATS, &stats)`读出计数器快照。每个ioctl都是一个小型的、类型良好的入口点，带有编号、方向和负载。第3节专门讨论ioctl。

第三个目标是**`sysctl(8)`**。计数器、统计信息和可调参数存在于系统范围的sysctl树中，可以从用户空间使用`sysctl(8)`访问，从C使用`sysctlbyname(3)`访问。驱动程序将其OID放在`dev.<driver>.<unit>.<name>`下，以便`sysctl dev.myfirst.0`列出设备公开的每个指标和旋钮。sysctl接口是只读计数器和缓慢变化的旋钮的正确归宿；ioctl是快速动作和类型化数据的正确归宿。第4节专门讨论sysctl。

第四个目标是**内核的生命周期钩子**。模块事件处理程序（`MOD_LOAD`、`MOD_UNLOAD`、`MOD_SHUTDOWN`）、设备树的附加和分离方法，以及通过`EVENTHANDLER(9)`和`SYSINIT(9)`的跨切注册共同定义了驱动程序进入和离开内核时发生的事情。正确实现集成接口但错误实现生命周期的驱动程序会泄漏资源、在卸载时死锁，或在第三次`kldload`/`kldunload`循环时崩溃。第7节专门讨论生命周期规则。

此外，硬件是网络设备的驱动程序与**`ifnet`**子系统集成，硬件是存储设备的驱动程序与**`CAM`**子系统集成。两者的概念都足够大，深入处理需要自己的章节（ifnet在第28章，CAM和GEOM在第27章）。本章第5节和第6节以识别其形状和了解它们涉及何种工作所需的水平介绍它们。

每个目标都因为特定原因而值得命名。`devfs`是临时用户输入shell的名称。`ioctl`是程序需要向设备发出类型化动作的入口点。`sysctl`是监控工具查找数字的地方。生命周期钩子是打包者和系统管理员在`kldload`和`kldunload`时触及的地方。这四个共同涵盖了驱动程序对作者以外任何人重要的四个方面。

### 正确集成的好处

集成本身不是目的。它是本章将不断返回的一小部分实际成果的手段。

第一个成果是**监控**。计数器通过`sysctl`可见的驱动程序可以被`prometheus-fbsd-exporter`、Nagios检查、每分钟运行的小型shell脚本抓取。计数器仅通过`device_printf`到`dmesg`可见的驱动程序只能通过手动阅读日志文件来检查。两种操作现实非常不同，差异完全由驱动程序编写时的集成选择决定。

第二个成果是**管理**。公开名为`MYF_RESET`的ioctl的驱动程序让管理员将重置周期脚本化到维护窗口中。没有该接口的驱动程序强制管理员`kldunload`和`kldload`模块，这是一个更繁重的操作，会丢弃每个打开的文件描述符，并且可能在生产流量流经设备时不可接受。

第三个成果是**自动化**。发出具有可预测名称的格式良好的`/dev`节点的驱动程序让`devd(8)`对附加和分离事件做出反应，在热插拔时运行脚本，并将驱动程序集成到更大的系统启动、关闭和恢复流程中。发出不透明的单个节点且从不告诉任何人其生命周期的驱动程序无法在不诉诸`dmesg`日志抓取的情况下自动化，这是脆弱的。

第四个成果是**可重用性**。ioctl接口文档良好的驱动程序可以成为更高级库的基础。例如，`bsnmp`守护程序使用定义良好的内核接口通过SNMP公开驱动程序计数器，而不触及驱动程序源代码。第一次就正确设计接口的驱动程序无需进一步工作即可获得这些好处。

这四个成果（监控、管理、自动化、可重用性）是接下来所有内容的实际原因。本章的每一节都提供这四个成果之一的一部分，第8节中的结束重构是使整个包可以呈现给系统其余部分的内容。

### 依赖集成的小型系统工具之旅

在技术工作之前，一个有用的练习是查看正是因为驱动程序与上述内核子系统集成而存在的用户空间工具。这些工具都不适用于独立的驱动程序。它们都自动适用于集成的驱动程序。

`devinfo(8)`遍历内核的设备树并打印它找到的内容。它之所以工作，是因为树中的每个设备都通过newbus接口注册，每个设备都有名称、单元号和父设备。管理员运行`devinfo -v`并看到整个设备层次结构，包括`myfirst0`实例。

`sysctl(8)`读取和写入内核sysctl树。它之所以工作，是因为内核中的每个计数器和旋钮都可以通过OID层次结构访问，包括驱动程序通过`device_get_sysctl_tree`注册的OID。

`devctl(8)`让管理员操作单个设备：`devctl detach`、`devctl attach`、`devctl suspend`、`devctl resume`、`devctl rescan`。它之所以工作，是因为每个设备都实现了内核设备树机制期望的kobj方法。第22章已经使用了`devctl suspend myfirst0`和`devctl resume myfirst0`。

`devd(8)`监视内核的设备事件通道并运行脚本以响应附加、分离、热插拔和类似事件。它之所以工作，是因为内核为每个newbus操作发出结构化事件。遵循标准newbus模式的驱动程序自动对`devd`可见。

`ifconfig(8)`配置网络接口。它之所以工作，是因为每个网络驱动程序都向ifnet子系统注册并接受一组标准的ioctl（第5节介绍这一点）。

`camcontrol(8)`通过CAM控制SCSI和SATA设备。它之所以工作，是因为每个存储驱动程序都注册一个SIM并处理CCB（第6节介绍这一点）。

`gstat(8)`显示实时GEOM统计信息，`geom(8)`列出GEOM树，`top -H`显示每线程CPU使用率。这些工具中的每一个都依赖于驱动程序注册的特定集成表面。忽略这些表面的驱动程序得不到任何好处。

确认这一点的一个简单方法是在你的实验机器上运行以下练习：

```sh
# 游览设备树
devinfo -v | head -40

# 游览sysctl树，仅dev分支
sysctl dev | head -40

# 看看实时网络接口是什么样子
ifconfig -a

# 通过CAM看看存储是什么样子
camcontrol devlist

# 看看GEOM看到了什么
geom -t

# 观看设备事件通道几秒钟
sudo devd -d -f /dev/null &
DEVDPID=$!
sleep 5
kill $DEVDPID
```

每个命令之所以存在，是因为驱动程序集成。阅读输出并注意可见的内容中有多少来自做了集成工作的驱动程序。`myfirst`驱动程序在第24章开始时几乎不对该输出做出贡献。到第24章结束时，它将向`sysctl`贡献其`dev.myfirst.0`子树，向`devinfo -v`贡献其`myfirst0`设备，向文件系统贡献其`/dev/myfirst*`节点。每一步都很小。总体上是一个一次性实验室驱动程序与FreeBSD真实部分之间的区别。

### 本章中"可选"的含义

第5节（网络）和第6节（存储）被标记为可选。该标签需要一个仔细的定义。

可选并不意味着不重要。两个部分都将在本书后面成为必读内容，网络材料在第27章之前，存储材料在第26章和第28章之前。可选意味着对于将`myfirst` PCI驱动程序作为运行示例跟随的读者，本章中不会使用网络和存储钩子，因为`myfirst`不是网络设备，也不是存储设备。本章介绍这些钩子的概念形状，以便读者在它们出现时能够识别它们，并使第7节和第8节中的结构决策考虑到它们。

时间紧迫的第一遍读者可以跳过第5节和第6节。其他部分不依赖于它们。计划跟随第26章或第27章的读者应该阅读它们，因为它们介绍了这些章节将假设的词汇。

本章诚实地介绍了这些部分教授内容的深度。网络子系统在`/usr/src/sys/net`中有几千行源代码。CAM子系统在`/usr/src/sys/cam`中有几千行源代码。每个都花了多年时间设计，并且仍在演进。第24章以*这是形状*、*这是驱动程序通常进行的调用*、*这是使用每一个的真实驱动程序*的水平介绍它们。完整的机制属于其他地方。

### 集成之路上的陷阱

三个陷阱困住了大多数第一次集成者。

第一个陷阱是**临时添加集成表面**。一个在某个周二添加ioctl的驱动程序，因为开发者需要一种快速的方法来测试设备，下一周添加sysctl，因为开发者想看到计数器，下个月因为报告的错误添加另一个ioctl，最终得到一个在风格、命名、错误处理和文档方面不一致的公共表面。正确的模式是有意设计公共表面，使用一致的命名，并在编写实现之前在头文件中记录每个入口点。第3节和第8节回顾这个规则。

第二个陷阱是**在设备方法内混合关注点**。其`device_attach`在一个长函数中执行kobj工作、资源分配、devfs节点创建、sysctl树构造和面向用户空间的设置的驱动程序很快变得不可读。本章建议首先将这些关注点分离到辅助函数中，并在第8节中分离到单独的源文件中。第23章中的`myfirst_debug.c`和`myfirst_debug.h`对是朝这个方向迈出的第一步；本章中新的`myfirst_ioctl.c`和`myfirst_sysctl.c`文件继续这个模式。

第三个陷阱是**不从用户空间测试公共表面**。仅从内核端测试的驱动程序将通过每个内核端测试，但在真实用户空间程序调用它时仍然失败，因为开发者假设了在实践中不成立的关于调用约定的某些东西。因此，本章坚持在驱动程序有任何ioctl时立即构建小型`myfirstctl`配套程序，并通过`sysctl(8)`测试每个sysctl，而不是仅通过直接读取驱动程序内计数器。用户空间测试是唯一确认集成真正有效的测试。

这些陷阱不是FreeBSD独有的。它们出现在每个具有内核-用户边界的操作系统中。FreeBSD的工具使做正确的事情比大多数系统更容易，因为约定在手册页（`devfs(5)`、`ioctl(2)`、`sysctl(9)`、`style(9)`）中记录良好，并且内核本身附带数百个集成驱动程序供读者学习。本章依赖这些约定并在进行时指向匹配的真实驱动程序。

### 本章的思维模型

在进入第2节之前，在脑海中固定一幅图画很有帮助。到第24章结束时，驱动程序从外部看将是这样的：

```text
用户空间工具                          内核
+----------------------+                +----------------------+
| myfirstctl           |  ioctl(2)      | d_ioctl回调          |
| sysctl(8)            +--------------->| sysctl OID树         |
| cat /dev/myfirst0    |  read/write    | d_read, d_write      |
| chmod, chown         |  fileops       | devfs节点生命周期     |
| devinfo -v           |  newbus查询    | device_t myfirst0    |
| dtrace -n 'myfirst:::'|  SDT探针      | sc_debug, DPRINTF    |
+----------------------+                +----------------------+
```

左侧的每个条目都是真实FreeBSD用户已经知道的工具。右侧的每个条目都是本章教你编写的集成部分。它们之间的箭头是本章每一节实现的内容。

在技术工作开始时记住这幅图画。以下每一节的目的都是向驱动程序添加这些箭头之一，并使用使箭头在驱动程序增长时保持可靠的规则。

### 第1节总结

集成是使驱动程序从其自己的源代码之外可见、可控和可观察的规则。FreeBSD中的四个主要集成表面是devfs、ioctl、sysctl和内核生命周期钩子。两个可选的子系统钩子是用于网络设备的ifnet和用于存储设备的CAM。它们共同构成了驱动程序如何不再是一个开发者的项目而成为FreeBSD一部分的方式。本章的其余部分实现每个表面，以`myfirst`驱动程序为运行示例，版本从`1.6-debug`升级到`1.7-integration`作为可见里程碑。

在下一节中，我们将转向第一个也是最基础的集成表面：`devfs`和为每个字符驱动程序提供其`/dev`存在的设备文件系统。


## 第2节：使用devfs和设备树

每个字符驱动程序跨越的第一个集成表面是`devfs`，即设备文件系统。读者从最早的章节开始就一直在创建`/dev/myfirst0`，但对`make_dev`的调用一直被呈现为单行样板代码，没有太多解释。第2节填补了这个空白。它解释了devfs实际上是什么，遍历了`cdev`的生命周期，详细调查了`make_dev`的变体和`cdevsw`回调表，展示了如何通过`si_drv1`和`si_drv2`附加每个节点的状态，教授了想要按需每个实例一个节点的驱动程序的现代可克隆感知模式，并以展示管理员如何在不重建驱动程序的情况下调整权限和所有权结束。

### devfs是什么

`devfs`是一个虚拟文件系统，将内核的已注册字符设备集作为`/dev`下的文件树公开。它是虚拟的，与`procfs`和`tmpfs`是虚拟的意义相同：没有支持它的磁盘存储。`/dev`下的每个文件都是内核将`cdev`结构投射到文件系统命名空间中的结果。当用户空间程序调用`open("/dev/null", O_RDWR)`时，内核在devfs中查找路径，找到匹配的`cdev`，跟踪指向`cdevsw`表的指针，并通过它分发打开操作。

较旧的UNIX系统使用静态设备树。管理员运行像`MAKEDEV`这样的程序或直接编辑`/dev`，文件系统包含设备节点，无论相应的硬件是否存在。静态方法有两个众所周知的问题。首先，管理员必须提前知道哪些设备是可能的，并手动创建匹配的节点，使用正确的主次设备号。其次，文件系统包含系统实际上没有的硬件的孤立节点，这令人困惑。

FreeBSD的`devfs`，在早期的5.x版本系列中作为默认引入，用动态方案取代了静态方案。内核本身根据哪些驱动程序调用了`make_dev`来决定哪些设备节点存在。当驱动程序调用`make_dev`时，节点出现在`/dev`下。当驱动程序调用`destroy_dev`时，节点消失。管理员不再需要手动维护设备条目，也没有不存在硬件的孤立节点。

devfs引入的权衡是`/dev`节点的生命周期现在由内核而不是文件系统控制。在分离时未能删除其节点的驱动程序会使它可见，直到内核自己删除它（内核最终会删除，但不如驱动程序那么及时）。意外创建相同节点两次的驱动程序会因重复名称检查而产生内核崩溃。从错误的上下文创建节点的驱动程序可能会使内核死锁。本章教授避免每个这些问题的模式。

理解devfs的一个有用细节是，内核维护单个全局设备节点命名空间，`make_dev`注册到该命名空间中。管理员可以在jail或chroot内挂载额外的devfs实例；每个实例投射全局命名空间的过滤视图，通过`devfs.rules(8)`控制。驱动程序本身不需要知道这些投射。它只需注册其`cdev`一次，内核和规则系统一起决定哪些视图可以看到它。

### `cdev`的生命周期

每个`cdev`经历五个阶段。按名称了解这些阶段使本节的其余部分更容易跟随。

第一阶段是**注册**。驱动程序从`device_attach`（或从模块事件处理程序，取决于设备是总线附加的还是伪设备）调用`make_dev`（或其变体之一）。调用返回驱动程序存储在其softc中的`struct cdev *`。从这一刻起，节点在`/dev`下可见。

第二阶段是**使用**。用户空间程序可以打开节点并针对它调用read、write、ioctl、mmap、poll或kqueue。每个调用都通过驱动程序在注册时安装的匹配cdevsw回调进行。一个`cdev`在任何时刻都可能有多个打开的文件句柄，驱动程序的回调必须能够安全地与彼此并发调用以及与驱动程序自己的内部工作（中断、callout、taskqueue）并发调用。

第三阶段是**销毁请求**。驱动程序调用`destroy_dev`（或`destroy_dev_sched`用于异步变体）。节点立即从`/dev`取消链接，因此不能有新的打开成功。现有打开在此刻不会关闭。

第四阶段是**排空**。`destroy_dev`阻塞，直到cdev的每个打开的文件句柄都通过`d_close`，并且每个进行中的对驱动程序的调用都已返回。一旦`destroy_dev`返回，驱动程序的回调保证不会再被调用。

第五阶段是**释放**。一旦`destroy_dev`返回，驱动程序可以释放softc，释放cdev回调正在使用的任何资源，并卸载模块。`struct cdev *`本身由内核拥有，在其最后一个引用消失时由内核释放；驱动程序不释放它。

排空步骤是最常困住第一次编写驱动程序的开发者的步骤。一个天真的驱动程序执行"destroy_dev; free(sc);"的等效操作，然后一个持有的打开文件句柄调用cdevsw并解引用已释放的softc，这会导致内核崩溃。本章教授如何正确处理这个问题：在分离路径中的任何状态释放之前放置`destroy_dev`调用，并信任内核在destroy返回之前排空进行中的调用。

### `make_dev`系列

FreeBSD提供了几个`make_dev`变体，每个都有不同的选项组合。它们位于`/usr/src/sys/sys/conf.h`和`/usr/src/sys/fs/devfs/devfs_devs.c`中。本章介绍最有用的四个。

最简单的形式是**`make_dev`**本身：

```c
struct cdev *
make_dev(struct cdevsw *devsw, int unit, uid_t uid, gid_t gid,
    int perms, const char *fmt, ...);
```

此调用创建一个由`uid:gid`拥有、权限为`perms`、名称根据`printf`风格格式确定的节点。它在内部使用`M_WAITOK`，可能会睡眠，因此必须从可以睡眠的上下文调用（通常是`device_attach`或模块加载处理程序，绝不从中断处理程序）。它不会失败：如果无法分配内存，它会睡眠直到可以。读者从早期章节开始就一直在使用这种形式。

更丰富的形式是**`make_dev_credf`**：

```c
struct cdev *
make_dev_credf(int flags, struct cdevsw *devsw, int unit,
    struct ucred *cr, uid_t uid, gid_t gid, int mode,
    const char *fmt, ...);
```

此变体接受显式的`flags`参数和显式的凭据。凭据由MAC框架在检查是否可以用给定所有者创建设备时使用。标志选择`MAKEDEV_ETERNAL`（内核从不自动销毁此节点）和`MAKEDEV_ETERNAL_KLD`（相同，但仅允许在可加载模块内）等功能。`null(4)`驱动程序使用这种形式，正如第1节参考列表中引用的那样。

为新驱动程序推荐的形式是**`make_dev_s`**：

```c
int
make_dev_s(struct make_dev_args *args, struct cdev **cdev,
    const char *fmt, ...);
```

此变体接受参数结构而不是长参数列表。该结构在填充其字段之前用`make_dev_args_init(&args)`初始化。`make_dev_s`的优势是它可以失败而不是睡眠，失败通过返回值而不是通过睡眠报告。它还有一个用于`cdev *`的输出参数，这意味着调用者不需要记住哪个位置返回代表什么。新代码应该优先使用`make_dev_s`，因为失败路径更干净。

参数结构如下：

```c
struct make_dev_args {
    size_t        mda_size;
    int           mda_flags;
    struct cdevsw *mda_devsw;
    struct ucred  *mda_cr;
    uid_t         mda_uid;
    gid_t         mda_gid;
    int           mda_mode;
    int           mda_unit;
    void          *mda_si_drv1;
    void          *mda_si_drv2;
};
```

`mda_size`字段由内核用于检测ABI不匹配；`make_dev_args_init`正确设置它。`mda_si_drv1`和`mda_si_drv2`字段让驱动程序在创建时将两个自己的指针附加到cdev；本章使用`mda_si_drv1`附加指向softc的指针。

与大多数驱动程序相关的`MAKEDEV_*`标志位是：

| 标志                    | 含义                                                          |
|-------------------------|------------------------------------------------------------------|
| `MAKEDEV_REF`           | 返回的cdev已被引用；用`dev_rel`平衡。         |
| `MAKEDEV_NOWAIT`        | 不睡眠；如果调用需要睡眠则返回失败。    |
| `MAKEDEV_WAITOK`        | 调用可以睡眠（`make_dev`的默认值）。                     |
| `MAKEDEV_ETERNAL`       | 内核不自动销毁此节点。             |
| `MAKEDEV_ETERNAL_KLD`   | 与ETERNAL相同，但允许在可加载模块内。            |
| `MAKEDEV_CHECKNAME`     | 根据devfs字符集验证名称。                |

`make_dev_p`变体与`make_dev_s`类似，但采用位置参数列表。它较旧，仍然受支持，并被树中的一些驱动程序使用；新驱动程序可以忽略它，转而使用`make_dev_s`。

### `cdevsw`结构

`cdevsw`表是字符设备回调的分发表。读者在早期章节中已经安装了一个，但第2节逐字段检查它。

一个最小的现代cdevsw如下：

```c
static struct cdevsw myfirst_cdevsw = {
    .d_version = D_VERSION,
    .d_flags   = D_TRACKCLOSE,
    .d_name    = "myfirst",
    .d_open    = myfirst_open,
    .d_close   = myfirst_close,
    .d_read    = myfirst_read,
    .d_write   = myfirst_write,
    .d_ioctl   = myfirst_ioctl,
};
```

`d_version`必须设置为`D_VERSION`。内核使用此字段检测针对旧cdevsw布局构建的驱动程序。缺少或错误的`d_version`是令人困惑的模块加载失败的常见来源；始终显式设置它。

`d_flags`控制一组小的可选行为。最常见的标志是：

| 标志             | 含义                                                                |
|------------------|---------------------------------------------------------------------|
| `D_TRACKCLOSE`   | 在每个fd的最后一次关闭时调用`d_close`，而不是每次关闭。        |
| `D_NEEDGIANT`    | 在分发周围获取内核范围的Giant锁（现代代码中罕见）。  |
| `D_NEEDMINOR`    | 分配次设备号（旧版；今天很少需要）。                  |
| `D_MMAP_ANON`    | 驱动程序通过`dev_pager`支持匿名`mmap`。               |
| `D_DISK`         | cdev是类似磁盘设备的入口点。                     |
| `D_TTY`          | cdev是终端设备；影响线路规程路由。         |

对于`myfirst`驱动程序，`D_TRACKCLOSE`是唯一值得设置的标志。它使内核在每个文件描述符的最后一次关闭时精确调用`d_close`一次，而不是在每次关闭时调用。没有`D_TRACKCLOSE`，想要计数打开文件句柄的驱动程序必须处理同一个`d_close`被多次调用的情况，这很尴尬。

`d_name`是内核在某些诊断消息中使用的名称。按照惯例，它与驱动程序的名称相同。

回调字段是指向实现每个操作的函数的指针。驱动程序只需安装它实际支持的回调；缺少的回调默认为返回`ENODEV`或`EOPNOTSUPP`的安全存根。字符驱动程序最常见的组合是`d_open`、`d_close`、`d_read`、`d_write`和`d_ioctl`。支持轮询的驱动程序添加`d_poll`。支持kqueue的驱动程序添加`d_kqfilter`。将内存映射到用户空间的驱动程序添加`d_mmap`或`d_mmap_single`。模拟磁盘的驱动程序添加`d_strategy`。

`d_purge`回调罕见但值得了解。当cdev正在被销毁并且驱动程序应该释放任何挂起的I/O时，内核调用它。大多数驱动程序不需要它，因为它们的`d_close`已经处理了释放。

`d_open`回调签名是：

```c
int myfirst_open(struct cdev *dev, int oflags, int devtype, struct thread *td);
```

`dev`是被打开的cdev。`oflags`是open(2)标志的并集（`O_RDWR`、`O_NONBLOCK`等）。`devtype`携带设备类型，对字符驱动程序很少有用。`td`是执行打开操作的线程。回调成功时返回`0`，失败时返回errno。典型的模式将通过`dev->si_drv1`找到的softc指针存储到后续调用可以恢复的每次打开的私有结构中。

`d_close`签名是平行的：

```c
int myfirst_close(struct cdev *dev, int fflags, int devtype, struct thread *td);
```

`d_read`和`d_write`签名使用读者在第8章中遇到的`uio`机制：

```c
int myfirst_read(struct cdev *dev, struct uio *uio, int ioflag);
int myfirst_write(struct cdev *dev, struct uio *uio, int ioflag);
```

`d_ioctl`签名是：

```c
int myfirst_ioctl(struct cdev *dev, u_long cmd, caddr_t data,
    int fflag, struct thread *td);
```

`cmd`是ioctl命令字。`data`指向ioctl数据缓冲区（内核已经为`IOC_IN`命令复制了数据，并将在`IOC_OUT`命令时复制出去，如第3节将详细讨论）。`fflag`是打开调用的文件标志。`td`是调用线程。

### 通过`si_drv1`实现每个Cdev的状态

cdev是内核对设备的句柄，驱动程序几乎总是需要一种从cdev指针找到自己softc的方法。标准机制是`cdev->si_drv1`。驱动程序在创建cdev时设置此字段（要么在调用`make_dev`时，要么之后写入该字段），并在每个cdevsw回调中读取它。

该模式在attach中看起来像这样：

```c
sc->sc_cdev = make_dev(&myfirst_cdevsw, device_get_unit(dev),
    UID_ROOT, GID_WHEEL, 0660, "myfirst%d", device_get_unit(dev));
sc->sc_cdev->si_drv1 = sc;
```

在每个回调中像这样：

```c
static int
myfirst_open(struct cdev *dev, int oflags, int devtype, struct thread *td)
{
    struct myfirst_softc *sc = dev->si_drv1;

    DPRINTF(sc, MYF_DBG_OPEN, "open: pid=%d flags=%#x\n",
        td->td_proc->p_pid, oflags);
    /* ... open的其余部分 ... */
    return (0);
}
```

`si_drv2`是驱动程序可以随意使用的第二个指针。一些驱动程序将它用于每个实例的cookie；其他驱动程序不使用它，让它保持`NULL`。`myfirst`驱动程序只使用`si_drv1`。

`make_dev_s`变体更干净，因为它在创建时设置`si_drv1`：

```c
struct make_dev_args args;
make_dev_args_init(&args);
args.mda_devsw = &myfirst_cdevsw;
args.mda_uid = UID_ROOT;
args.mda_gid = GID_WHEEL;
args.mda_mode = 0660;
args.mda_si_drv1 = sc;
args.mda_unit = device_get_unit(dev);
error = make_dev_s(&args, &sc->sc_cdev, "myfirst%d", device_get_unit(dev));
if (error != 0) {
    device_printf(dev, "make_dev_s failed: %d\n", error);
    goto fail;
}
```

`make_dev_s`形式的额外优势是`si_drv1`在cdev在`/dev`中可见之前设置，这关闭了一个小但真实的竞争窗口，在这个窗口中，一个快速打开的程序可能调用`si_drv1`仍为`NULL`的cdevsw。新驱动程序应该优先使用这种形式。

### `null(4)`参考

树中最干净的小型cdevsw和`make_dev_credf`示例是`/usr/src/sys/dev/null/null.c`。相关的摘录短到可以一次阅读。cdevsw声明（分别用于`/dev/null`和`/dev/zero`）：

```c
static struct cdevsw null_cdevsw = {
    .d_version = D_VERSION,
    .d_read    = (d_read_t *)nullop,
    .d_write   = null_write,
    .d_ioctl   = null_ioctl,
    .d_name    = "null",
};
```

创建和销毁节点的模块事件处理程序：

```c
static int
null_modevent(module_t mod, int type, void *data)
{
    switch (type) {
    case MOD_LOAD:
        full_dev = make_dev_credf(MAKEDEV_ETERNAL_KLD, &full_cdevsw, 0,
            NULL, UID_ROOT, GID_WHEEL, 0666, "full");
        null_dev = make_dev_credf(MAKEDEV_ETERNAL_KLD, &null_cdevsw, 0,
            NULL, UID_ROOT, GID_WHEEL, 0666, "null");
        zero_dev = make_dev_credf(MAKEDEV_ETERNAL_KLD, &zero_cdevsw, 0,
            NULL, UID_ROOT, GID_WHEEL, 0666, "zero");
        break;
    case MOD_UNLOAD:
        destroy_dev(full_dev);
        destroy_dev(null_dev);
        destroy_dev(zero_dev);
        break;
    case MOD_SHUTDOWN:
        break;
    default:
        return (EOPNOTSUPP);
    }
    return (0);
}

DEV_MODULE(null, null_modevent, NULL);
MODULE_VERSION(null, 1);
```

有几个细节值得暂停。`MAKEDEV_ETERNAL_KLD`标志告诉内核，即使模块在异常情况下卸载，也不应静默地使cdev无效。`0666`模式意味着每个人都可以读写；这对`/dev/null`是正确的。单元号为`0`，因为每种只有一个。`MOD_LOAD`分支在模块加载时运行并创建节点；`MOD_UNLOAD`分支在模块卸载时运行并销毁它们；`MOD_SHUTDOWN`在系统关闭序列期间运行，这里不做任何事情，因为这些伪设备不需要关闭工作。

这是存在于模块范围而不是设备树范围的伪设备的规范形状。相比之下，`myfirst`驱动程序在`device_attach`中创建其cdev，因为cdev的生命周期与特定PCI设备的生命周期绑定，而不是与模块的生命周期绑定。两种模式不同但舒适地共存。

### 多节点和可克隆节点

单个驱动程序可以创建多个节点。三种模式很常见。

第一种是**固定名称节点**。驱动程序提前知道需要多少节点并用固定名称创建它们：`/dev/myfirst0`、`/dev/myfirst-status`、`/dev/myfirst-config`。当每个节点有不同的角色且数量已知时，这是正确的模式。

第二种是**索引节点**。驱动程序为每个单元创建一个节点，命名为`/dev/myfirstN`，其中`N`是单元号。当每个节点代表相同类型对象的单独实例（例如每个附加的PCI卡一个）时，这是正确的模式。

第三种是**可克隆节点**。驱动程序注册一个克隆处理程序，每当用户打开匹配模式的名称时按需创建新节点。`tun(4)`的读者打开`/dev/tun`并获得`/dev/tun0`；再次打开`/dev/tun`产生`/dev/tun1`；内核在每次打开时分配下一个空闲单元。对于用户希望通过简单打开来"创建"的伪设备，这是正确的模式。

克隆机制是`dev_clone`事件处理程序。它位于`/usr/src/sys/sys/conf.h`中：

```c
typedef void (*dev_clone_fn)(void *arg, struct ucred *cred, char *name,
    int namelen, struct cdev **result);

EVENTHANDLER_DECLARE(dev_clone, dev_clone_fn);
```

驱动程序用`EVENTHANDLER_REGISTER`注册处理程序，当`/dev`下的打开路径不匹配现有节点时，内核调用该处理程序。处理程序决定名称是否属于其驱动程序，如果是，则分配新单元，调用`make_dev`用该名称创建节点，并通过`result`参数存储生成的cdev指针。然后内核重新打开新创建的节点并继续用户的打开调用。

`tun(4)`驱动程序展示了这种模式。来自`/usr/src/sys/net/if_tuntap.c`：

```c
static eventhandler_tag clone_tag;

static int
tuntapmodevent(module_t mod, int type, void *data)
{
    switch (type) {
    case MOD_LOAD:
        clone_tag = EVENTHANDLER_REGISTER(dev_clone, tunclone, 0, 1000);
        if (clone_tag == NULL)
            return (ENOMEM);
        ...
        break;
    case MOD_UNLOAD:
        EVENTHANDLER_DEREGISTER(dev_clone, clone_tag);
        ...
    }
}

static void
tunclone(void *arg, struct ucred *cred, char *name, int namelen,
    struct cdev **dev)
{
    /* 如果*dev != NULL，另一个处理程序已经创建了cdev。 */
    if (*dev != NULL)
        return;

    /* 检查名称；如果匹配我们的模式，分配一个单元 */
    /* 并调用make_dev来填充*dev。 */
    ...
}
```

克隆处理程序适用一些规则。处理程序不能假设它是唯一注册的处理程序；多个子系统可以注册到`dev_clone`，每个处理程序在做任何工作之前必须检查`*dev`是否已经非NULL。处理程序在可以睡眠的上下文中运行，所以它可以直接调用`make_dev`。处理程序应该仔细验证名称，因为它由用户空间提供。

`myfirst`驱动程序最初使用索引节点模式（每个附加的PCI设备一个节点，`/dev/myfirst0`、`/dev/myfirst1`等），第2节的实验演示了添加克隆处理程序，以便用户可以打开`/dev/myfirst-clone`并按需获得新单元。克隆模式对于没有底层硬件的伪设备最有用；对于硬件支持的驱动程序，它很少需要。

### 权限、所有权和`devfs.rules`

`make_dev`接受节点的初始所有者UID、组GID和权限。这些值在创建时嵌入。它们从用户空间通过`ls -l /dev/myfirst0`可见，并决定哪些用户空间程序可以打开节点。

对于硬件支持的驱动程序，正确的默认值取决于角色。只有root可以访问的设备使用`UID_ROOT, GID_WHEEL, 0600`。任何管理用户都应该能够访问的设备使用`UID_ROOT, GID_OPERATOR, 0660`。任何用户都可以读但只有root可以写的设备使用`UID_ROOT, GID_WHEEL, 0644`。`myfirst`驱动程序默认使用`UID_ROOT, GID_WHEEL, 0660`；用户应该是root或通过`devfs.rules`被授予访问权限。

管理员可以在运行时通过`devfs.rules(8)`覆盖这些默认值。典型的规则文件如下：

```text
[localrules=10]
add path 'myfirst*' mode 0660
add path 'myfirst*' group operator
```

管理员通过添加到`/etc/rc.conf`来激活规则：

```text
devfs_system_ruleset="localrules"
```

在`service devfs restart`（或重启）之后，规则适用于新出现的设备节点。这种机制让管理员可以在不重建模块的情况下授予对驱动程序的访问权限，这是正确的责任分工：开发者选择安全的默认值，管理员在需要时放宽它们。

一个常见的错误是默认使设备全局可写，因为"这使测试更容易"。以`0666`权限发布控制硬件设备的驱动程序是安全问题。本章建议默认使用`0660`和`GID_WHEEL`，并在`INTEGRATION.md`中指导读者如果需要如何使用`devfs.rules`更改它。

### 综合运用：第1阶段驱动程序

第24章第1阶段驱动程序用现代的`make_dev_s`模式替换了原始的临时节点创建，在创建时设置`si_drv1`，使用`D_TRACKCLOSE`标志，并为第3节中要添加的ioctl回调做准备。以下是新的attach函数的相关摘录。手动输入；本章的全部意义在于从旧的临时形式到新的规则形式的仔细更改。

```c
static int
myfirst_attach(device_t dev)
{
    struct myfirst_softc *sc;
    struct make_dev_args args;
    int error;

    sc = device_get_softc(dev);
    sc->sc_dev = dev;

    /* ... 早期的attach工作：PCI资源、MSI-X、DMA、sysctl树
     * 存根、debug子树。参见第18-23章。 ... */

    /* 为/dev/myfirstN构建cdev。 */
    make_dev_args_init(&args);
    args.mda_devsw = &myfirst_cdevsw;
    args.mda_uid = UID_ROOT;
    args.mda_gid = GID_WHEEL;
    args.mda_mode = 0660;
    args.mda_si_drv1 = sc;
    args.mda_unit = device_get_unit(dev);
    error = make_dev_s(&args, &sc->sc_cdev, "myfirst%d",
        device_get_unit(dev));
    if (error != 0) {
        device_printf(dev, "make_dev_s failed: %d\n", error);
        DPRINTF(sc, MYF_DBG_INIT, "cdev creation failed (%d)\n", error);
        goto fail;
    }
    DPRINTF(sc, MYF_DBG_INIT, "cdev created at /dev/myfirst%d\n",
        device_get_unit(dev));

    /* ... attach的其余部分：注册callout、完成sysctl OID等。 */

    return (0);

fail:
    /* 以相反顺序展开早期资源。 */
    /* 参见第7节了解这里的规则。 */
    return (error);
}
```

对应的带有`D_TRACKCLOSE`的cdevsw：

```c
static d_open_t myfirst_open;
static d_close_t myfirst_close;
static d_read_t myfirst_read;
static d_write_t myfirst_write;
static d_ioctl_t myfirst_ioctl;

static struct cdevsw myfirst_cdevsw = {
    .d_version = D_VERSION,
    .d_flags   = D_TRACKCLOSE,
    .d_name    = "myfirst",
    .d_open    = myfirst_open,
    .d_close   = myfirst_close,
    .d_read    = myfirst_read,
    .d_write   = myfirst_write,
    .d_ioctl   = myfirst_ioctl,
};
```

对应的打开和关闭，带有`si_drv1`查找和每次打开计数器更新：

```c
static int
myfirst_open(struct cdev *dev, int oflags, int devtype, struct thread *td)
{
    struct myfirst_softc *sc = dev->si_drv1;

    SDT_PROBE2(myfirst, , , open, sc, oflags);

    mtx_lock(&sc->sc_mtx);
    sc->sc_open_count++;
    mtx_unlock(&sc->sc_mtx);

    DPRINTF(sc, MYF_DBG_OPEN,
        "open: pid=%d flags=%#x open_count=%u\n",
        td->td_proc->p_pid, oflags, sc->sc_open_count);
    return (0);
}

static int
myfirst_close(struct cdev *dev, int fflags, int devtype, struct thread *td)
{
    struct myfirst_softc *sc = dev->si_drv1;

    SDT_PROBE2(myfirst, , , close, sc, fflags);

    mtx_lock(&sc->sc_mtx);
    KASSERT(sc->sc_open_count > 0,
        ("myfirst_close: open_count underflow"));
    sc->sc_open_count--;
    mtx_unlock(&sc->sc_mtx);

    DPRINTF(sc, MYF_DBG_OPEN,
        "close: pid=%d flags=%#x open_count=%u\n",
        td->td_proc->p_pid, fflags, sc->sc_open_count);
    return (0);
}
```

以及匹配的detach更新，在任何softc状态释放之前带有`destroy_dev`：

```c
static int
myfirst_detach(device_t dev)
{
    struct myfirst_softc *sc = device_get_softc(dev);

    /* 当用户仍然打开设备时拒绝detach。 */
    mtx_lock(&sc->sc_mtx);
    if (sc->sc_open_count > 0) {
        mtx_unlock(&sc->sc_mtx);
        device_printf(dev, "detach refused: %u open(s) outstanding\n",
            sc->sc_open_count);
        return (EBUSY);
    }
    mtx_unlock(&sc->sc_mtx);

    /* 首先销毁cdev。内核在destroy_dev返回之前
     * 排空所有进行中的回调。 */
    if (sc->sc_cdev != NULL) {
        destroy_dev(sc->sc_cdev);
        sc->sc_cdev = NULL;
        DPRINTF(sc, MYF_DBG_INIT, "cdev destroyed\n");
    }

    /* ... detach的其余部分：拆除DMA、MSI-X、callout、sysctl ctx
     * 等，按与attach相反的顺序。参见第7节。 */

    return (0);
}
```

这是第1阶段的里程碑。驱动程序现在拥有一个干净的、现代的、对初学者友好的devfs入口。接下来的阶段将在此基础之上添加ioctl和sysctl。

### 具体演练：加载和检查第1阶段驱动程序

构建、加载并检查第1阶段驱动程序：

```sh
cd ~/myfirst-1.7-integration/stage1-devfs
make
sudo kldload ./myfirst.ko

# 确认设备存在并具有预期的属性。
ls -l /dev/myfirst0

# 读取其sysctl debug子树（仍来自第23章）。
sysctl dev.myfirst.0

# 用cat打开它以确认cdevsw read和close路径被触发。
sudo cat /dev/myfirst0 > /dev/null
dmesg | tail -20
```

`ls -l /dev/myfirst0`的预期输出：

```text
crw-rw----  1 root  wheel  0x71 Apr 19 16:30 /dev/myfirst0
```

`crw-rw----`表示字符设备，所有者读写，组读写，其他人无权限。所有者是root。组是wheel。次设备号是`0x71`（内核的分配；该值因系统而异）。

`dmesg`片段的预期输出：

```text
myfirst0: cdev created at /dev/myfirst0
myfirst0: open: pid=4321 flags=0x1 open_count=1
myfirst0: close: pid=4321 flags=0x1 open_count=0
```

（假设调试掩码设置了`MYF_DBG_INIT`和`MYF_DBG_OPEN`。如果掩码为零，这些行是静默的；用第23章的调试规则中的`sysctl dev.myfirst.0.debug.mask=0xFFFFFFFF`打开它们。）

如果打开和关闭行没有出现，检查调试掩码。如果打开行出现但关闭行没有，你忘记了`D_TRACKCLOSE`，内核在每个fd关闭时调用close而不是仅最后一次；要么打开`D_TRACKCLOSE`，要么预期每次打开会有多个关闭行。如果打开计数变为负数，你有一个真正的错误：在没有匹配的打开的情况下调用了关闭。

### devfs的常见错误

五个错误占了第一次集成者遇到的devfs问题的大部分。提前命名它们可以节省后续的调试会话。

第一个错误是**从不可睡眠的上下文调用`make_dev`**。`make_dev`可能会睡眠。如果从中断处理程序、从callout、从自旋锁内部或从任何禁止睡眠的上下文内调用它，内核会崩溃，`WITNESS`或`INVARIANTS`会抱怨在不可睡眠上下文中有可睡眠函数。修复方法是要么从`device_attach`或模块事件处理程序调用`make_dev`（两者都是安全的上下文），要么使用带`MAKEDEV_NOWAIT`的`make_dev_s`（这样它可能会失败，调用者必须处理失败）。

第二个错误是**忘记设置`si_drv1`**。cdevsw回调然后解引用空指针并导致内核崩溃。修复方法是在`make_dev`之后立即设置`si_drv1`（或者更好的是，使用`make_dev_s`并在args中设置`mda_si_drv1`，这关闭了创建和赋值之间的竞争窗口）。

第三个错误是**在释放softc之后调用`destroy_dev`**。当调用`destroy_dev`时，cdevsw回调可能仍在进行中；内核在`destroy_dev`返回之前排空它们。如果softc已经释放，回调会解引用垃圾数据。修复方法是以严格的顺序首先调用`destroy_dev`，然后释放softc。

第四个错误是**创建两个具有相同名称的cdev**。内核检查重复名称，第二次`make_dev`调用根据变体的不同会崩溃或返回错误。修复方法是从单元号组合名称，或使用克隆处理程序。

第五个错误是**不处理设备在detach时处于打开状态的情况**。天真的驱动程序只是调用`destroy_dev`并释放softc，这只在没有用户打开设备时有效。用`cat`保持`/dev/myfirst0`打开的用户会使这失败。修复方法是当`open_count > 0`时拒绝detach，或使用内核的`dev_ref`/`dev_rel`机制来协调。本章的模式是简单的拒绝（`return (EBUSY)`），因为它提供了最干净的用户可见错误。

### 多实例驱动程序的独特陷阱

创建多个cdev（每个附加设备一个或每个通道一个）的驱动程序会遇到一些额外的陷阱。

第一个是**部分失败时泄漏节点**。如果驱动程序创建三个cdev而第三个调用失败，它必须在返回之前销毁前两个。第7节的失败时清理模式是规范的解决方案。本节中的实验通过故意的失败注入演示了该模式。

第二个是**忘记每个cdev需要自己的`si_drv1`**。创建每个通道节点的驱动程序通常希望`si_drv1`指向通道而不是softc。cdevsw回调然后根据需要从通道回溯到softc。混淆两者会导致通道相互踩踏状态。

第三个是**无竞争的节点可见性**。在`make_dev`（或`make_dev_s`）返回和驱动程序完成attach的其余部分之间，快速的用户已经可以打开节点。驱动程序必须准备好处理在attach完全完成之前到达的打开。最简单的模式是将`make_dev`推迟到attach的最后，这样节点只在每件其他状态都准备好后才可见。`myfirst`驱动程序遵循这种模式。

这些陷阱对于像`myfirst`这样的单cdev驱动程序不常出现，但现在识别它们意味着读者在第27章（存储驱动程序可以有许多cdev，每个LUN一个）或第28章（网络驱动程序可以有许多cdev，每个命令通道一个）中出现时不会感到惊讶。

### 第2节总结

第2节使驱动程序的`/dev`存在成为了一等公民。cdev现在使用现代的`make_dev_s`模式创建，cdevsw已用`D_TRACKCLOSE`和对调试友好的回调集完全填充，每个cdev的状态通过`si_drv1`连接，detach路径干净地排空和销毁cdev。驱动程序仍然做与`1.6-debug`相同的工作，但现在通过一个管理员可以从内核外部chmod、chown、通过`ls`监视和推理的正确构建的`/dev`节点公开该工作。

在下一节中，我们将转向第二个集成表面：让程序告诉驱动程序该做什么的用户驱动控制接口。词汇是`ioctl(2)`、`_IO`、`_IOR`、`_IOW`、`_IOWR`和一个用户空间程序包含的小型公共头文件。

## 第3节：实现`ioctl()`支持

### `ioctl(2)`是什么，以及驱动程序为什么需要它

`read(2)`和`write(2)`系统调用非常适合在用户程序和驱动程序之间移动字节流。然而，它们不太适合控制。`read`无法在不重载返回字节含义的情况下询问驱动程序"你当前的状态是什么？"。`write`无法在不发明字节流内私有命令词汇的情况下要求驱动程序"请重置你的统计信息"。填补这一空白的系统调用是`ioctl(2)`，即输入/输出控制调用。

`ioctl`是命令的侧信道。用户空间的签名很简单：`int ioctl(int fd, unsigned long request, ...);`。第一个参数是文件描述符（在我们的例子中是打开的`/dev/myfirst0`）。第二个是一个数字请求代码，告诉驱动程序该做什么。第三个是一个可选的指向携带请求参数的结构的指针，可以是入站、出站或双向。内核将调用路由到支持该文件描述符的cdev的cdevsw中，进入`d_ioctl`指向的函数。驱动程序查看请求代码，执行相应的操作，并成功时返回0或失败时返回正的`errno`。

几乎所有向用户空间公开控制接口的驱动程序都使用`ioctl`。磁盘驱动程序使用`ioctl`报告扇区大小、分区表和转储目标。磁带驱动程序使用`ioctl`进行倒带、弹出和张紧命令。网络驱动程序使用`ioctl`进行媒体更改（`SIOCSIFFLAGS`）、MAC地址更新和总线探测命令。声音驱动程序使用`ioctl`进行采样率、通道数和缓冲区大小协商。该词汇如此普遍，学习一次就能解锁树中每个类别的驱动程序。

对于`myfirst`驱动程序，`ioctl`让我们添加没有干净字节表达的命令。我们可以让操作员查询内存中的消息长度而不必读取它。我们可以让操作员重置消息和打开计数器而不必写入特殊的标记。我们可以公开驱动程序的版本号，以便用户空间工具可以检测它们正在与之通信的API。每一个对操作员来说都是一行更改，对驱动程序来说是半页更改，每一个都是`ioctl`的教科书式适配。

本节遍历整个ioctl管道：请求代码的编码、内核的自动copyin和copyout、公共头文件的设计、分发器的实现、小型用户空间配套程序的构建以及最常见的陷阱。到本节结束时，驱动程序将处于版本`1.7-integration-stage-2`，并将支持四个ioctl命令：`MYFIRSTIOC_GETVER`、`MYFIRSTIOC_GETMSG`、`MYFIRSTIOC_SETMSG`和`MYFIRSTIOC_RESET`。

### `ioctl`编号如何编码

ioctl请求代码不是任意整数。它是一个打包的32位值，在固定位字段中编码四条信息，定义在`/usr/src/sys/sys/ioccom.h`中。头文件以显示布局的注释开始，在继续之前值得阅读。

```c
/*
 * Ioctl's have the command encoded in the lower word, and the size of
 * any in or out parameters in the upper word.  The high 3 bits of the
 * upper word are used to encode the in/out status of the parameter.
 *
 *       31 29 28                     16 15            8 7             0
 *      +---------------------------------------------------------------+
 *      | I/O | Parameter Length        | Command Group | Command       |
 *      +---------------------------------------------------------------+
 */
```

四个字段是：

**方向位**（第29到31位）告诉内核`ioctl`的第三个参数是纯出站（`IOC_OUT`，内核将结果复制回用户空间）、纯入站（`IOC_IN`，内核在分发器运行之前将用户数据复制到内核中）、双向（`IOC_INOUT`，两个方向）或不存在（`IOC_VOID`，请求不带数据参数）。内核使用这些位来决定自动执行什么`copyin`和`copyout`。驱动程序本身永远不必为正确编码的ioctl调用`copyin`或`copyout`。

**参数长度**（第16到28位）编码作为第三个参数传递的结构的大小（以字节为单位），上限为`IOCPARM_MAX = 8192`。内核使用此大小分配临时内核缓冲区，执行适当的`copyin`或`copyout`，并将缓冲区作为`caddr_t data`参数呈现给分发器。需要通过单个ioctl传递超过8192字节的驱动程序必须在较小的结构中嵌入指针（代价是自己进行`copyin`），或使用不同的机制如`mmap`或`read`。

**命令组**（第8到15位）是一个命名相关ioctl系列的单个字符。按照惯例，它是可打印ASCII字母之一，用于标识子系统。`'d'`用于GEOM磁盘ioctl（`DIOCGMEDIASIZE`、`DIOCGSECTORSIZE`）。`'i'`用于`if_ioctl`（`SIOCSIFFLAGS`）。`'t'`用于终端ioctl（`TIOCGPTN`）。读者应该选择一个尚未被驱动程序可能与之共存的任何东西占用的字母。对于`myfirst`驱动程序，我们将使用`'M'`。

**命令编号**（第0到7位）是一个标识组内特定ioctl的小整数。编号通常从1开始，随着命令的添加单调递增。重用编号是向后兼容性风险，因此退役命令的驱动程序应该保留编号而不是回收它。

`ioccom.h`中的宏为你构建这些编码。它们是正确构建ioctl编号的唯一方法：

```c
#define _IO(g,n)        _IOC(IOC_VOID, (g), (n), 0)
#define _IOR(g,n,t)     _IOC(IOC_OUT,  (g), (n), sizeof(t))
#define _IOW(g,n,t)     _IOC(IOC_IN,   (g), (n), sizeof(t))
#define _IOWR(g,n,t)    _IOC(IOC_INOUT,(g), (n), sizeof(t))
```

`_IO`声明一个不带参数的命令。`_IOR`声明一个将`t`大小的结果返回给用户空间的命令。`_IOW`声明一个从用户空间接受`t`大小参数的命令。`_IOWR`声明一个接受`t`大小参数并通过同一缓冲区写回`t`大小结果的命令。`t`是一个类型，而不是指针；宏使用`sizeof(t)`来计算长度字段。

几个现实世界的例子使模式具体化。来自`/usr/src/sys/sys/disk.h`：

```c
#define DIOCGSECTORSIZE _IOR('d', 128, u_int)
#define DIOCGMEDIASIZE  _IOR('d', 129, off_t)
```

这些命令是扇区大小（作为`u_int`返回）和媒体大小（作为`off_t`返回）的只读请求。组字母`'d'`和编号128和129为磁盘子系统保留。

驱动程序本身永远不必解码位布局。命令代码对分发器是不透明的，分发器将其与命名常量进行比较：

```c
switch (cmd) {
case MYFIRSTIOC_GETVER:
        ...
        break;
case MYFIRSTIOC_RESET:
        ...
        break;
default:
        return (ENOIOCTL);
}
```

内核在设置调用时使用位布局（分配缓冲区并执行copyin/copyout），用户空间通过宏隐式使用它。在这两点之间，请求代码只是一个标签。

### 选择组字母

组字母的选择很重要，因为冲突是静默的。选择相同组字母和相同命令编号的两个驱动程序，如果操作员混淆两者，将会看到原本发给另一个驱动程序的ioctl请求。内核不跨驱动程序强制唯一性，部分原因是没有中央权威分配字母，部分原因是大多数字母按传统保留而不是注册。

防御性方法是遵循这些约定：

仅在扩展现有子系统时使用**小写字母**（`'d'`、`'t'`、`'i'`），且你已经知道该子系统的字母。小写字母被基本驱动程序大量使用，很容易冲突。

为需要自己ioctl命名空间的新驱动程序使用**大写字母**（`'M'`、`'X'`、`'Q'`）。有26个大写字母，树中冲突少得多。

完全避免**数字**。它们按历史惯例为早期子系统保留，使用其中一个的新驱动程序对审查者来说会显得格格不入。

对于`myfirst`驱动程序，我们使用`'M'`。它是驱动程序名称的第一个字母，它是大写的（所以不会与任何基本子系统冲突），它使请求代码在堆栈跟踪和`ktrace`输出中自文档化：在组字段中带有`0x4d`（`'M'`的ASCII值）的ioctl编号的十六进制转储明确是`myfirst`命令。

### `d_ioctl`回调签名

`cdevsw->d_ioctl`指向的分发器函数具有`d_ioctl_t`类型，定义在`/usr/src/sys/sys/conf.h`中。签名是：

```c
typedef int d_ioctl_t(struct cdev *dev, u_long cmd, caddr_t data,
                      int fflag, struct thread *td);
```

五个参数值得慢慢阅读。

`dev`是支持用户调用`ioctl`的文件描述符的cdev。驱动程序使用`dev->si_drv1`恢复其softc。这与我们已经看到的每个cdevsw回调使用的模式相同。

`cmd`是请求代码，即用户作为第二个参数传递给`ioctl`的值。驱动程序将其与公共头文件中的命名常量进行比较。

`data`是第三个参数的内核本地副本。因为内核已经执行了copyin并将执行copyout（对于`_IOR`、`_IOW`和`_IOWR`请求），`data`始终是内核指针。驱动程序直接解引用它而不调用`copyin`。对于`_IO`请求，`data`未定义，不得解引用。

`fflag`是来自打开调用的文件标志：`FREAD`、`FWRITE`或两者。驱动程序可以使用`fflag`为特定命令强制只读或只写访问。例如，重置状态的命令可能需要`FWRITE`，否则返回`EBADF`。

`td`是调用线程。驱动程序可以使用`td`提取调用者的凭据（`td->td_ucred`）、执行特权检查（`priv_check_cred(td->td_ucred, PRIV_DRIVER, 0)`）或简单地记录调用者的pid。对于大多数命令，`td`未使用。

返回值成功时为0，失败时为正的`errno`值。特殊值`ENOIOCTL`（定义在`/usr/src/sys/sys/errno.h`中）告诉内核驱动程序不识别该命令，内核然后将命令路由到文件系统层的通用ioctl处理程序。对未知命令返回`EINVAL`而不是`ENOIOCTL`是一个微妙的错误：它告诉内核"我识别该命令但参数错误"，这抑制了通用回退。始终在默认情况下使用`ENOIOCTL`。

### 内核如何执行Copyin和Copyout

在分发器运行之前，内核检查编码在`cmd`中的方向位和参数长度。如果设置了`IOC_IN`，内核读取参数长度，分配该大小的临时内核缓冲区，将用户空间参数复制到其中（`copyin`），并将内核缓冲区作为`data`传递。如果设置了`IOC_OUT`，内核分配缓冲区，用（未初始化的）缓冲区作为`data`调用分发器，并在返回0时将缓冲区复制回用户空间（`copyout`）。如果两个位都设置了（`IOC_INOUT`），内核在分发器调用周围执行copyin和copyout。如果都没有设置（`IOC_VOID`），不分配缓冲区，`data`未定义。

这种自动化有两个值得记住的后果。

首先，分发器使用普通C解引用写入和读取`data`。驱动程序永远不为正确编码的ioctl调用`copyin`或`copyout`。这是正确设计的ioctl接口比伪造控制通道的写入和读取协议简单得多的原因之一。

其次，编码在`_IOW`或`_IOR`中的参数类型必须与分发器实际读取或写入的内容匹配。如果用户空间头文件声明`_IOR('M', 1, uint32_t)`，但分发器将`uint64_t`写入`*(uint64_t *)data`，分发器将溢出内核的4字节缓冲区并损坏相邻的栈内存，在启用`WITNESS`的构建下导致内核崩溃，在生产构建下静默损坏状态。头文件是契约；分发器必须逐字节遵守它。

对于带有嵌入指针的结构（包含指向单独缓冲区的`char *buf`的结构），内核无法复制该缓冲区，因为它不是结构的一部分。驱动程序必须自己对缓冲区内容进行`copyin`和`copyout`，而内核处理包装结构。这种模式用于可变长度数据，在下面的陷阱子节中介绍。

### 设计公共头文件

公开ioctl的驱动程序必须发布声明请求代码和数据结构的头文件。用户空间程序包含此头文件以构建正确的调用。头文件存在于内核模块之外：它是驱动程序与用户空间契约的一部分，必须可以在系统上安装（例如，安装到`/usr/local/include/myfirst/myfirst_ioctl.h`）。

约定是将头文件放在驱动程序源目录中，后缀为`.h`，与公共名称匹配。对于`myfirst`驱动程序，头文件是`myfirst_ioctl.h`。其职责很窄：声明ioctl编号、声明用作ioctl参数的结构、声明任何相关常量（如消息字段的最大长度），除此之外别无其他。它绝不能包含仅内核的头文件，绝不能声明仅内核的类型，并且必须在被用户空间程序包含时干净地编译。

以下是本章第2阶段驱动程序的完整头文件：

```c
/*
 * myfirst_ioctl.h - myfirst驱动程序的公共ioctl接口。
 *
 * 此头文件由内核模块和任何与驱动程序通信的用户空间
 * 程序包含。保持其自包含：不包含内核头文件、不包含内核类型、
 * 不包含拉取内核状态的内联函数。
 */

#ifndef _MYFIRST_IOCTL_H_
#define _MYFIRST_IOCTL_H_

#include <sys/ioccom.h>
#include <sys/types.h>

/*
 * 驱动程序内消息的最大长度，包括尾随的NUL。
 * 驱动程序在SETMSG上强制执行此限制；构建
 * 更大缓冲区的用户空间程序将看到EINVAL。
 */
#define MYFIRST_MSG_MAX 256

/*
 * 接口版本。当此头文件以非向后兼容的方式更改时递增。
 * 用户空间程序应该首先调用MYFIRSTIOC_GETVER，
 * 如果版本不符合预期则拒绝操作。
 */
#define MYFIRST_IOCTL_VERSION 1

/*
 * MYFIRSTIOC_GETVER - 返回驱动程序的接口版本。
 *
 *   ioctl(fd, MYFIRSTIOC_GETVER, &ver);   // ver = 1, 2, ...
 *
 * 不需要FREAD或FWRITE标志。
 */
#define MYFIRSTIOC_GETVER  _IOR('M', 1, uint32_t)

/*
 * MYFIRSTIOC_GETMSG - 将当前驱动程序内消息复制到
 * 调用者的缓冲区。缓冲区必须是MYFIRST_MSG_MAX字节；
 * 消息以NUL结尾。
 */
#define MYFIRSTIOC_GETMSG  _IOR('M', 2, char[MYFIRST_MSG_MAX])

/*
 * MYFIRSTIOC_SETMSG - 替换驱动程序内消息。缓冲区必须
 * 是MYFIRST_MSG_MAX字节；内核采用第一个NUL之前的前缀
 * 或MYFIRST_MSG_MAX - 1字节。
 *
 * 需要对文件描述符具有FWRITE权限。
 */
#define MYFIRSTIOC_SETMSG  _IOW('M', 3, char[MYFIRST_MSG_MAX])

/*
 * MYFIRSTIOC_RESET - 重置所有每个实例的计数器并清除
 * 消息。成功返回0。
 *
 * 需要对文件描述符具有FWRITE权限。
 */
#define MYFIRSTIOC_RESET   _IO('M', 4)

#endif /* _MYFIRST_IOCTL_H_ */
```

此头文件中有几个细节值得暂停。

使用`uint32_t`和`sys/types.h`（而不是`u_int32_t`和`sys/cdefs.h`）使头文件在FreeBSD基本系统和任何遵循POSIX的程序之间可移植。内核和用户空间在`uint32_t`的大小上一致，因此请求代码中编码的长度与分发器的数据视图匹配。

最大消息长度`MYFIRST_MSG_MAX = 256`远低于`IOCPARM_MAX = 8192`，因此内核将毫无怨言地复制该消息。需要移动更大消息的驱动程序要么提高限制（最高8192），要么切换到嵌入指针模式。

`MYFIRST_IOCTL_VERSION`常量为用户空间提供了一种检测API更改的方法。任何程序应该发出的第一个ioctl是`MYFIRSTIOC_GETVER`；如果返回的版本与程序编译时的版本不匹配，程序应该拒绝发出进一步的ioctl并打印清晰的错误。这是期望演进的驱动程序的标准做法。

参数类型`char[MYFIRST_MSG_MAX]`在`_IOR`和`_IOW`中不寻常但合法。宏采用`sizeof(t)`，而`sizeof(char[256]) == 256`，因此编码的长度恰好是数组大小。这是在公共ioctl头文件中表达固定大小缓冲区的最干净的方式。

### 实现分发器

有了头文件，分发器就是一个switch语句，读取命令代码，执行操作，并返回0（成功）或正的errno（失败）。分发器位于`myfirst_ioctl.c`中，这是第2阶段添加到驱动程序的新源文件。

完整的分发器：

```c
/*
 * myfirst_ioctl.c - myfirst驱动程序的ioctl分发器。
 *
 * myfirst_cdevsw中的d_ioctl回调指向myfirst_ioctl。
 * 每个命令的参数布局在myfirst_ioctl.h中记录，
 * 该文件与用户空间共享。
 */

#include <sys/param.h>
#include <sys/systm.h>
#include <sys/conf.h>
#include <sys/file.h>
#include <sys/lock.h>
#include <sys/mutex.h>
#include <sys/malloc.h>
#include <sys/proc.h>

#include "myfirst.h"
#include "myfirst_debug.h"
#include "myfirst_ioctl.h"

SDT_PROBE_DEFINE3(myfirst, , , ioctl,
    "struct myfirst_softc *", "u_long", "int");

int
myfirst_ioctl(struct cdev *dev, u_long cmd, caddr_t data, int fflag,
    struct thread *td)
{
        struct myfirst_softc *sc = dev->si_drv1;
        int error = 0;

        SDT_PROBE3(myfirst, , , ioctl, sc, cmd, fflag);
        DPRINTF(sc, MYF_DBG_IOCTL, "ioctl: cmd=0x%08lx fflag=0x%x\n",
            cmd, fflag);

        mtx_lock(&sc->sc_mtx);

        switch (cmd) {
        case MYFIRSTIOC_GETVER:
                *(uint32_t *)data = MYFIRST_IOCTL_VERSION;
                break;

        case MYFIRSTIOC_GETMSG:
                /*
                 * 将当前消息复制到调用者的缓冲区。
                 * 缓冲区是MYFIRST_MSG_MAX字节；我们总是发出
                 * 一个以NUL结尾的字符串。
                 */
                strlcpy((char *)data, sc->sc_msg, MYFIRST_MSG_MAX);
                break;

        case MYFIRSTIOC_SETMSG:
                if ((fflag & FWRITE) == 0) {
                        error = EBADF;
                        break;
                }
                /*
                 * 内核已将MYFIRST_MSG_MAX字节复制到
                 * data中。采用第一个NUL之前的前缀。
                 */
                strlcpy(sc->sc_msg, (const char *)data, MYFIRST_MSG_MAX);
                sc->sc_msglen = strlen(sc->sc_msg);
                DPRINTF(sc, MYF_DBG_IOCTL,
                    "SETMSG: new message is %zu bytes\n", sc->sc_msglen);
                break;

        case MYFIRSTIOC_RESET:
                if ((fflag & FWRITE) == 0) {
                        error = EBADF;
                        break;
                }
                sc->sc_open_count = 0;
                sc->sc_total_reads = 0;
                sc->sc_total_writes = 0;
                bzero(sc->sc_msg, sizeof(sc->sc_msg));
                sc->sc_msglen = 0;
                DPRINTF(sc, MYF_DBG_IOCTL,
                    "RESET: counters and message cleared\n");
                break;

        default:
                error = ENOIOCTL;
                break;
        }

        mtx_unlock(&sc->sc_mtx);
        return (error);
}
```

此分发器中融入了几个有规则的选择，在读者编写自己的分发器之前值得指出。

分发器在顶部获取softc互斥锁一次，在底部释放一次。每个命令都在互斥锁下运行。这防止`read`与`SETMSG`竞争（否则读取会看到半替换的消息缓冲区），并防止两个同时的`RESET`调用损坏计数器。互斥锁与第IV部分前面介绍的`sc->sc_mtx`相同；我们只是扩展其范围以涵盖ioctl序列化。

分发器在获取互斥锁后的第一个动作是单个SDT探针和单个`DPRINTF`。两者都报告命令和文件标志。SDT探针让DTrace脚本实时跟踪每个ioctl；`DPRINTF`让操作员打开`MYF_DBG_IOCTL`并通过`dmesg`观察相同的流程。两者都使用第23章引入的调试基础设施，没有新机制。

`MYFIRSTIOC_SETMSG`和`MYFIRSTIOC_RESET`路径在改变状态之前检查`fflag & FWRITE`。没有此检查，以只读方式打开设备的程序可以更改驱动程序的状态，这在某些驱动程序中是特权升级模式。检查返回`EBADF`（操作文件描述符错误）而不是`EPERM`（无权限），因为失败是关于文件的打开标志而不是关于用户的身份。

默认分支返回`ENOIOCTL`，从不返回`EINVAL`。这是上一小节的规则，这里重复是因为它是自制分发器中最常见的错误。

`GETMSG`和`SETMSG`中的`strlcpy`调用是FreeBSD内核中安全的字符串复制原语。它们保证NUL终止且永远不会溢出目标。在旧代码中相同的调用会是`strncpy`；`strlcpy`是现代首选的形式，也是`style(9)`推荐的。

### Softc添加

第2阶段用两个字段扩展softc并确认现有字段仍在使用：

```c
struct myfirst_softc {
        device_t        sc_dev;
        struct cdev    *sc_cdev;
        struct mtx      sc_mtx;

        /* 来自早期章节。 */
        uint32_t        sc_debug;
        u_int           sc_open_count;
        u_int           sc_total_reads;
        u_int           sc_total_writes;

        /* 第2阶段新增。 */
        char            sc_msg[MYFIRST_MSG_MAX];
        size_t          sc_msglen;
};
```

消息缓冲区是一个固定大小的数组，其大小与公共头文件匹配。内联存储它（而不是作为指向单独分配缓冲区的指针）使生命周期简单：缓冲区与softc一样长。没有要跟踪的`malloc`，也没有要忘记的`free`。

`myfirst_attach`中的初始化变为：

```c
strlcpy(sc->sc_msg, "Hello from myfirst", sizeof(sc->sc_msg));
sc->sc_msglen = strlen(sc->sc_msg);
```

驱动程序现在有一个默认问候语，它会一直保留，直到操作员通过`SETMSG`更改它，并且只有在每次加载时重新设置新值的情况下才能在`unload`/`load`循环后存活。（这与每个其他softc字段的生命周期相同；跨重启的持久性需要sysctl可调参数，这是第4节的主题。）

### 将分发器连接到`cdevsw`

第1阶段声明的cdevsw已经有一个等待填充的`.d_ioctl`槽位。第2阶段填充它：

```c
static struct cdevsw myfirst_cdevsw = {
        .d_version = D_VERSION,
        .d_flags   = D_TRACKCLOSE,
        .d_name    = "myfirst",
        .d_open    = myfirst_open,
        .d_close   = myfirst_close,
        .d_read    = myfirst_read,
        .d_write   = myfirst_write,
        .d_ioctl   = myfirst_ioctl,    /* 新增 */
};
```

内核在模块加载时读取此表一次。没有运行时注册步骤；cdevsw是驱动程序静态状态的一部分。

### 构建第2阶段驱动程序

第2阶段的`Makefile`必须包含新的源文件：

```make
KMOD=   myfirst
SRCS=   myfirst.c myfirst_debug.c myfirst_ioctl.c

CFLAGS+= -I${.CURDIR}

SYSDIR?= /usr/src/sys

.include <bsd.kmod.mk>
```

构建命令与第1阶段相同：

```console
$ make
$ sudo kldload ./myfirst.ko
$ ls -l /dev/myfirst0
crw-rw---- 1 root wheel 0x... <date> /dev/myfirst0
```

如果构建失败是因为找不到`myfirst_ioctl.h`，检查`CFLAGS`行是否包含`-I${.CURDIR}`。如果加载失败是因为有未解析的符号如`myfirst_ioctl`，检查`myfirst_ioctl.c`是否列在`SRCS`中，以及函数名是否与cdevsw条目匹配。

### `myfirstctl`用户空间配套程序

具有ioctl接口的驱动程序需要一个测试它的小型配套程序。没有它，操作员除了通过手写测试或通过`devctl(8)`的ioctl传递外，无法调用ioctl，这对日常使用来说很尴尬。

配套程序是`myfirstctl`，一个单文件C程序，在命令行上接受子命令并调用相应的ioctl。它故意很小（不到200行），只依赖于公共头文件。

```c
/*
 * myfirstctl.c - myfirst驱动程序ioctl的命令行前端。
 *
 * 构建:  cc -o myfirstctl myfirstctl.c
 * 用法:  myfirstctl get-version
 *        myfirstctl get-message
 *        myfirstctl set-message "<text>"
 *        myfirstctl reset
 */

#include <sys/types.h>
#include <sys/ioctl.h>

#include <err.h>
#include <fcntl.h>
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <unistd.h>

#include "myfirst_ioctl.h"

#define DEVPATH "/dev/myfirst0"

static void
usage(void)
{
        fprintf(stderr,
            "usage: myfirstctl get-version\n"
            "       myfirstctl get-message\n"
            "       myfirstctl set-message <text>\n"
            "       myfirstctl reset\n");
        exit(EX_USAGE);
}

int
main(int argc, char **argv)
{
        int fd, flags;
        const char *cmd;

        if (argc < 2)
                usage();
        cmd = argv[1];

        /*
         * SETMSG和RESET需要写访问；其他只需要读。
         * 用正确的标志打开设备，以便分发器
         * 不会返回EBADF。
         */
        if (strcmp(cmd, "set-message") == 0 ||
            strcmp(cmd, "reset") == 0)
                flags = O_RDWR;
        else
                flags = O_RDONLY;

        fd = open(DEVPATH, flags);
        if (fd < 0)
                err(EX_OSERR, "open %s", DEVPATH);

        if (strcmp(cmd, "get-version") == 0) {
                uint32_t ver;
                if (ioctl(fd, MYFIRSTIOC_GETVER, &ver) < 0)
                        err(EX_OSERR, "MYFIRSTIOC_GETVER");
                printf("driver ioctl version: %u\n", ver);
        } else if (strcmp(cmd, "get-message") == 0) {
                char buf[MYFIRST_MSG_MAX];
                if (ioctl(fd, MYFIRSTIOC_GETMSG, buf) < 0)
                        err(EX_OSERR, "MYFIRSTIOC_GETMSG");
                printf("%s\n", buf);
        } else if (strcmp(cmd, "set-message") == 0) {
                char buf[MYFIRST_MSG_MAX];
                if (argc < 3)
                        usage();
                strlcpy(buf, argv[2], sizeof(buf));
                if (ioctl(fd, MYFIRSTIOC_SETMSG, buf) < 0)
                        err(EX_OSERR, "MYFIRSTIOC_SETMSG");
        } else if (strcmp(cmd, "reset") == 0) {
                if (ioctl(fd, MYFIRSTIOC_RESET) < 0)
                        err(EX_OSERR, "MYFIRSTIOC_RESET");
        } else {
                usage();
        }

        close(fd);
        return (0);
}
```

有两个细节值得注意。

程序用请求操作所需的最低标志打开设备。`MYFIRSTIOC_GETVER`和`MYFIRSTIOC_GETMSG`用`O_RDONLY`工作正常，但`MYFIRSTIOC_SETMSG`和`MYFIRSTIOC_RESET`需要`O_RDWR`，因为分发器检查`fflag & FWRITE`。在没有适当组成员资格的情况下运行`myfirstctl set-message foo`的用户会从`open`看到"Permission denied"；具有成员资格但分发器仍然拒绝的用户会从`ioctl`看到"Bad file descriptor"。两种错误都是可理解的。

`MYFIRSTIOC_RESET`调用不传递第三个参数，因为宏`_IO`（没有`R`或`W`）声明了一个void-data ioctl。库的`ioctl(2)`是可变参数的，所以用两个参数调用它是合法的，但需要小心，因为额外的参数会被传递但被忽略。本书的约定是用恰好两个参数调用`_IO` ioctl，以使源代码中的void-data性质清晰。

典型的会话如下：

```console
$ myfirstctl get-version
driver ioctl version: 1
$ myfirstctl get-message
Hello from myfirst
$ myfirstctl set-message "drivers are fun"
$ myfirstctl get-message
drivers are fun
$ myfirstctl reset
$ myfirstctl get-message

$
```

`reset`之后，消息缓冲区为空，`myfirstctl get-message`打印一个空行。计数器也被重置，下一节的sysctl接口将让我们直接验证这一点。

### ioctl的常见陷阱

第一个陷阱是**头文件和分发器之间的类型大小不匹配**。如果头文件声明`_IOR('M', 1, uint32_t)`，但分发器将`uint64_t`写入`*(uint64_t *)data`，内核分配了一个4字节缓冲区，分发器向其中写入8字节。额外的4字节会损坏缓冲区旁边的任何东西（通常是其他ioctl参数或分发器的本地栈帧）。在`WITNESS`和`INVARIANTS`下，内核可能会捕获溢出并崩溃；在生产构建下，结果是静默损坏。修复方法是使头文件和分发器保持同步，理想情况下是通过在两个地方包含相同的头文件（本章的模式就是这样做的）。

第二个陷阱是**嵌入指针**。包含`char *buf; size_t len;`的结构无法通过单个`_IOW`安全传输。内核会复制该结构（指针和长度），但指针指向的缓冲区在用户空间，分发器不能直接解引用它。分发器必须调用`copyin(uap->buf, kbuf, uap->len)`来自己传输缓冲区内容。忘记这一步会导致分发器通过内核指针读取用户空间内存，内核的地址空间保护会将其作为错误捕获。修复方法是将缓冲区内联到结构中（本章用于消息字段的模式），或在分发器内添加显式的`copyin`/`copyout`调用。

第三个陷阱是**忘记正确处理`ENOIOCTL`**。对未知命令返回`EINVAL`的驱动程序抑制了内核的通用ioctl回退。用户可能会从本应静默传递到文件系统层的命令看到"Invalid argument"（例如用于非阻塞I/O提示的`FIONBIO`）。修复方法是在默认情况下使用`ENOIOCTL`作为返回值。

第四个陷阱是**更改现有ioctl的线路格式**。一旦程序针对声明为256字节缓冲区的`MYFIRSTIOC_SETMSG`编译，用512字节缓冲区重新编译驱动程序会破坏程序：请求代码中编码的长度更改，内核检测到不匹配（因为用户用新的512字节命令传递了256字节缓冲区），`ioctl`调用返回`ENOTTY`（"Inappropriate ioctl for device"）。修复方法是将现有ioctl保留原样，并在格式必须更改时定义带有新编号的新命令。`MYFIRST_IOCTL_VERSION`常量让用户空间程序在发出受影响的调用之前检测这种演进。

第五个陷阱是**在持有互斥锁的同时在分发器中进行慢速工作**。本节中的分发器在整个switch语句中持有`sc->sc_mtx`，这很好，因为每个命令都很快（一个memcpy、一个计数器重置、一个strlcpy）。需要执行可能需要几毫秒的硬件操作的真实驱动程序必须首先释放互斥锁，然后再重新获取它，或使用可睡眠锁。在`tsleep`或`msleep`期间持有不可睡眠的互斥锁会导致内核崩溃。

### 第3节总结

第3节完成了第二个集成表面：一个正确设计的ioctl接口。驱动程序现在通过`MYFIRSTIOC_GETVER`、`MYFIRSTIOC_GETMSG`、`MYFIRSTIOC_SETMSG`和`MYFIRSTIOC_RESET`公开四个命令。接口是自描述的（任何用户空间程序都可以调用`MYFIRSTIOC_GETVER`来检测API版本），编码是显式的（来自`/usr/src/sys/sys/ioccom.h`的`_IOR`/`_IOW`/`_IO`宏），内核根据位布局自动处理copyin和copyout。配套的`myfirstctl`程序演示了用户空间工具如何在不触及请求代码本身字节的情况下测试接口。

第2阶段的驱动程序里程碑是添加了`myfirst_ioctl.c`和`myfirst_ioctl.h`，它们都与第23章的调试基础设施干净地集成（`MYF_DBG_IOCTL`掩码位和`myfirst:::ioctl` SDT探针）。`Makefile`在`SRCS`中增加了一个条目，cdevsw增加了一个填充的回调。驱动程序中的其他一切都没有改变。

在第4节中，我们将转向第三个集成表面：管理员可以使用`sysctl(8)`从shell查询和调整的只读和读写旋钮。ioctl是已经打开设备的程序的正确通道，sysctl是想要在不打开任何东西的情况下检查或调整驱动程序状态的脚本或操作员的正确通道。这两个接口相辅相成；大多数生产驱动程序都提供两者。

## 第4节：通过`sysctl()`公开指标

### `sysctl`是什么，以及驱动程序为什么使用它

`sysctl(8)`是FreeBSD内核的分层名称服务。树中的每个名称映射到一条内核状态：一个常量、一个计数器、一个可调变量或一个按需产生值的函数指针。树的根是`kern.`、`vm.`、`hw.`、`net.`、`dev.`和少数其他顶级前缀。任何具有适当特权的程序都可以通过`sysctl(3)`库、`sysctl(8)`命令行工具或`sysctlbyname(3)`便捷接口读取和（对于可写节点）修改这些值。

对于驱动程序，相关的子树是`dev.<driver_name>.<unit>.*`。Newbus子系统为每个附加的设备自动创建此前缀。名为`myfirst`且附加了单元0的驱动程序免费获得前缀`dev.myfirst.0`，无需驱动程序代码。驱动程序唯一的工作是用命名OID（对象标识符）为它想要公开的值填充此前缀。

为什么通过sysctl而不是ioctl公开状态？两种机制回答不同的问题。ioctl是已经打开设备并想要发出命令的程序的正确通道。Sysctl是在shell提示符下想要在不打开任何东西的情况下检查或调整状态的操作员或脚本的正确通道。大多数生产驱动程序都提供两者：用于程序的ioctl接口和用于人类、脚本和监控工具的sysctl接口。

常见模式是sysctl公开：

* 自附加以来总结驱动程序活动的**计数器**
* 如版本号、硬件标识符和链路状态的**只读状态**
* 如调试掩码、队列深度和超时值的**读写可调参数**
* 从`/boot/loader.conf`读取的、在驱动程序附加之前的**启动时可调参数**

到本节结束时，`myfirst`驱动程序将在`dev.myfirst.0`下公开所有四个类别，并将从`/boot/loader.conf`读取其初始调试掩码。第3阶段的驱动程序里程碑是添加`myfirst_sysctl.c`和一个小型OID树。

### Sysctl命名空间

完整的sysctl名称看起来像一个点分路径。我们驱动程序的默认Newbus前缀是：

```text
dev.myfirst.0
```

在此前缀下，驱动程序可以添加任何它喜欢的内容。第3阶段的`myfirst`驱动程序将添加：

```text
dev.myfirst.0.%desc            "myfirst pseudo-device, integration version 1.7"
dev.myfirst.0.%driver          "myfirst"
dev.myfirst.0.%location        ""
dev.myfirst.0.%pnpinfo         ""
dev.myfirst.0.%parent          "nexus0"
dev.myfirst.0.version          "1.7-integration"
dev.myfirst.0.open_count       0
dev.myfirst.0.total_reads      0
dev.myfirst.0.total_writes     0
dev.myfirst.0.message          "Hello from myfirst"
dev.myfirst.0.debug.mask       0
dev.myfirst.0.debug.classes    "INIT(0x1) OPEN(0x2) IO(0x4) IOCTL(0x8) ..."
```

前五个名称（以`%`开头的）由Newbus自动添加，描述设备树关系。其余名称是驱动程序的贡献。其中，`version`、`open_count`、`total_reads`、`total_writes`和`debug.classes`是只读的；`message`和`debug.mask`是读写的。`debug`子树本身是一个节点，这意味着随着驱动程序的增长，它可以容纳更多的OID。

读者已经可以在加载了`myfirst`的系统上看到结果：

```console
$ sysctl dev.myfirst.0
dev.myfirst.0.debug.classes: INIT(0x1) OPEN(0x2) IO(0x4) IOCTL(0x8) INTR(0x10) DMA(0x20) PWR(0x40) MEM(0x80)
dev.myfirst.0.debug.mask: 0
dev.myfirst.0.message: Hello from myfirst
dev.myfirst.0.total_writes: 0
dev.myfirst.0.total_reads: 0
dev.myfirst.0.open_count: 0
dev.myfirst.0.version: 1.7-integration
dev.myfirst.0.%parent: nexus0
dev.myfirst.0.%pnpinfo:
dev.myfirst.0.%location:
dev.myfirst.0.%driver: myfirst
dev.myfirst.0.%desc: myfirst pseudo-device, integration version 1.7
```

行的顺序是OID创建顺序的逆序。（Newbus最后添加`%`前缀的名称，所以当sysctl反向遍历列表时，它们首先打印。这是外观上的，没有语义含义。）

### 静态OID与动态OID

Sysctl OID有两种风格。

**静态OID**在编译时用`SYSCTL_*`宏之一（`SYSCTL_INT`、`SYSCTL_STRING`、`SYSCTL_ULONG`等）声明。宏生成一个常量数据结构，链接器将其粘合到一个特殊的节中，内核在启动时将该节组装到全局树中。静态OID适用于在内核生命周期内存在的系统范围值：计时器滴答、调度器统计等。

**动态OID**在运行时用`SYSCTL_ADD_*`函数之一（`SYSCTL_ADD_INT`、`SYSCTL_ADD_STRING`、`SYSCTL_ADD_PROC`等）创建。该函数接受上下文、父节点、名称和指向底层数据的指针，并将新节点插入树中。动态OID适用于随设备来去的每个实例的值：驱动程序在`attach`中创建它们，在`detach`中拆除。

驱动程序代码几乎完全使用动态OID。驱动程序在内核编译时不存在；它在模块加载时出现，它拥有的任何sysctl子树必须在附加时构建并在分离时处置。Newbus框架为每个驱动程序提供了专门用于此目的的每个设备sysctl上下文和父OID：

```c
struct sysctl_ctx_list *ctx;
struct sysctl_oid *tree;
struct sysctl_oid_list *child;

ctx = device_get_sysctl_ctx(dev);
tree = device_get_sysctl_tree(dev);
child = SYSCTL_CHILDREN(tree);
```

`device_get_sysctl_ctx`返回每个设备的上下文。上下文跟踪驱动程序创建的每个OID，以便框架在驱动程序分离时可以一次性释放它们。驱动程序不必自己跟踪它们。

`device_get_sysctl_tree`返回每个设备的树节点，即对应于`dev.<driver>.<unit>`的OID。该树是在设备添加时由Newbus创建的。

`SYSCTL_CHILDREN(tree)`从树节点中提取子列表。这是驱动程序在后续`SYSCTL_ADD_*`调用中作为父参数传递的内容。

有了这三个句柄，驱动程序可以向其子树添加任意数量的OID：

```c
SYSCTL_ADD_UINT(ctx, child, OID_AUTO, "open_count",
    CTLFLAG_RD, &sc->sc_open_count, 0,
    "Number of times the device has been opened");
```

`SYSCTL_ADD_UINT`调用在名为`open_count`的父节点下添加一个无符号int OID，带有`CTLFLAG_RD`（只读），由`&sc->sc_open_count`支持，没有特殊的初始值，带有描述。描述是`sysctl -d dev.myfirst.0.open_count`将打印的内容。始终编写有用的描述；空的描述是文档空白。

读写整数的匹配调用除了标志外完全相同：

```c
SYSCTL_ADD_UINT(ctx, child, OID_AUTO, "debug_mask_simple",
    CTLFLAG_RW, &sc->sc_debug, 0,
    "Simple writable debug mask");
```

`CTLFLAG_RW`标志告诉内核允许特权用户（root或具有`PRIV_DRIVER`的进程）写入。

对于字符串，宏是`SYSCTL_ADD_STRING`：

```c
SYSCTL_ADD_STRING(ctx, child, OID_AUTO, "version",
    CTLFLAG_RD, sc->sc_version, 0,
    "Driver version string");
```

倒数第二个参数是指向保存字符串的缓冲区的指针，倒数第二个是缓冲区的大小（零表示只读字符串的无限制）。

### 处理程序支持的OID

一些OID需要比普通内存访问更多的逻辑。读取OID可能需要从多个softc字段计算值；写入OID可能需要验证新值并更新相关状态。这些OID使用处理程序函数和宏`SYSCTL_ADD_PROC`。

处理程序的签名是：

```c
static int handler(SYSCTL_HANDLER_ARGS);
```

`SYSCTL_HANDLER_ARGS`是一个展开为以下内容的宏：

```c
struct sysctl_oid *oidp, void *arg1, intptr_t arg2,
struct sysctl_req *req
```

`oidp`标识被访问的OID。`arg1`和`arg2`是创建OID时注册的用户提供的参数（通常`arg1`指向softc，`arg2`未使用或持有一个小常量）。`req`携带读/写上下文：`req->newptr`对写入操作是非NULL（并指向用户正在提供的新值），处理程序必须调用`SYSCTL_OUT(req, value, sizeof(value))`来在读取时返回值。

一个暴露计算值的典型处理程序：

```c
static int
myfirst_sysctl_message_len(SYSCTL_HANDLER_ARGS)
{
        struct myfirst_softc *sc = arg1;
        u_int len;

        mtx_lock(&sc->sc_mtx);
        len = (u_int)sc->sc_msglen;
        mtx_unlock(&sc->sc_mtx);

        return (sysctl_handle_int(oidp, &len, 0, req));
}
```

处理程序计算值（这里通过在互斥锁下复制消息长度），然后委托给`sysctl_handle_int`，后者处理读写的簿记，并（对于写入）用已经在新值`*ptr`中的值回调处理程序。处理程序-处理程序模式是惯用的；正确使用它可以避免为每个类型化处理程序重新实现copyin和copyout。

处理程序用`SYSCTL_ADD_PROC`注册：

```c
SYSCTL_ADD_PROC(ctx, child, OID_AUTO, "message_len",
    CTLTYPE_UINT | CTLFLAG_RD | CTLFLAG_MPSAFE,
    sc, 0, myfirst_sysctl_message_len, "IU",
    "Current length of the in-driver message");
```

三个参数值得注意。`CTLTYPE_UINT | CTLFLAG_RD | CTLFLAG_MPSAFE`是类型和标志字。`CTLTYPE_UINT`声明OID的外部类型（无符号int）；`CTLFLAG_RD`声明它只读；`CTLFLAG_MPSAFE`声明处理程序在不持有giant锁的情况下调用是安全的。`CTLFLAG_MPSAFE`标志对新代码是强制性的；没有它，内核仍然可以工作，但会在每次sysctl访问时获取giant锁，这会在任何sysctl访问上序列化整个系统。

第七个参数是格式字符串。`"IU"`声明一个无符号int（`I`表示整数，`U`表示无符号）。完整的集合记录在`/usr/src/sys/sys/sysctl.h`中：`"I"`表示int，`"IU"`表示uint，`"L"`表示long，`"LU"`表示ulong，`"Q"`表示int64，`"QU"`表示uint64，`"A"`表示字符串，`"S,structname"`表示不透明结构。`sysctl(8)`命令使用格式字符串来决定在不使用`-x`（原始十六进制标志）调用时如何打印值。

### `myfirst` Sysctl树

第3阶段的完整sysctl树在单个函数`myfirst_sysctl_attach`中构建，在cdev创建后从`myfirst_attach`调用。该函数足够短，可以端到端阅读：

```c
/*
 * myfirst_sysctl.c - myfirst驱动程序的sysctl树。
 *
 * 构建dev.myfirst.<unit>.*，包含version、counters、message和
 * debug子树（debug.mask、debug.classes）。
 */

#include <sys/param.h>
#include <sys/systm.h>
#include <sys/conf.h>
#include <sys/kernel.h>
#include <sys/lock.h>
#include <sys/malloc.h>
#include <sys/mutex.h>
#include <sys/sysctl.h>

#include "myfirst.h"
#include "myfirst_debug.h"

#define MYFIRST_VERSION "1.7-integration"

static int
myfirst_sysctl_message_len(SYSCTL_HANDLER_ARGS)
{
        struct myfirst_softc *sc = arg1;
        u_int len;

        mtx_lock(&sc->sc_mtx);
        len = (u_int)sc->sc_msglen;
        mtx_unlock(&sc->sc_mtx);

        return (sysctl_handle_int(oidp, &len, 0, req));
}

void
myfirst_sysctl_attach(struct myfirst_softc *sc)
{
        device_t dev = sc->sc_dev;
        struct sysctl_ctx_list *ctx;
        struct sysctl_oid *tree;
        struct sysctl_oid_list *child;
        struct sysctl_oid *debug_node;
        struct sysctl_oid_list *debug_child;

        ctx = device_get_sysctl_ctx(dev);
        tree = device_get_sysctl_tree(dev);
        child = SYSCTL_CHILDREN(tree);

        /* 只读：驱动程序版本。 */
        SYSCTL_ADD_STRING(ctx, child, OID_AUTO, "version",
            CTLFLAG_RD, MYFIRST_VERSION, 0,
            "Driver version string");

        /* 只读：计数器。 */
        SYSCTL_ADD_UINT(ctx, child, OID_AUTO, "open_count",
            CTLFLAG_RD, &sc->sc_open_count, 0,
            "Number of currently open file descriptors");

        SYSCTL_ADD_UINT(ctx, child, OID_AUTO, "total_reads",
            CTLFLAG_RD, &sc->sc_total_reads, 0,
            "Total read() calls since attach");

        SYSCTL_ADD_UINT(ctx, child, OID_AUTO, "total_writes",
            CTLFLAG_RD, &sc->sc_total_writes, 0,
            "Total write() calls since attach");

        /* 只读：消息缓冲区（不通过用户复制） */
        SYSCTL_ADD_STRING(ctx, child, OID_AUTO, "message",
            CTLFLAG_RD, sc->sc_msg, sizeof(sc->sc_msg),
            "Current in-driver message");

        /* 只读处理程序：消息长度，计算得出。 */
        SYSCTL_ADD_PROC(ctx, child, OID_AUTO, "message_len",
            CTLTYPE_UINT | CTLFLAG_RD | CTLFLAG_MPSAFE,
            sc, 0, myfirst_sysctl_message_len, "IU",
            "Current length of the in-driver message in bytes");

        /* 子树：debug.* */
        debug_node = SYSCTL_ADD_NODE(ctx, child, OID_AUTO, "debug",
            CTLFLAG_RD | CTLFLAG_MPSAFE, NULL,
            "Debug controls and class enumeration");
        debug_child = SYSCTL_CHILDREN(debug_node);

        SYSCTL_ADD_UINT(ctx, debug_child, OID_AUTO, "mask",
            CTLFLAG_RW | CTLFLAG_TUN, &sc->sc_debug, 0,
            "Bitmask of enabled debug classes");

        SYSCTL_ADD_STRING(ctx, debug_child, OID_AUTO, "classes",
            CTLFLAG_RD,
            "INIT(0x1) OPEN(0x2) IO(0x4) IOCTL(0x8) "
            "INTR(0x10) DMA(0x20) PWR(0x40) MEM(0x80)",
            0, "Names and bit values of debug classes");
}
```

三个细节值得仔细看看。

`version` OID由字符串常量（`MYFIRST_VERSION`）支持，而不是softc字段。只读字符串OID可以指向任何稳定的缓冲区；内核从不通过指针写入。这比携带版本的每个softc副本更安全更简单，并且即使驱动程序在附加过程中途失败，也使版本通过`sysctl`可见。

`message` OID直接指向softc的`sc_msg`字段，带有`CTLFLAG_RD`。调用`sysctl dev.myfirst.0.message`的读取者将获得当前值。因为OID是只读的，sysctl不会写入缓冲区，所以我们不需要写入处理程序。（此OID的读写版本将需要一个处理程序来验证输入；读写路径在第2阶段的ioctl接口中运行。）

`debug.mask` OID具有`CTLFLAG_RW | CTLFLAG_TUN`。`RW`标志允许特权用户写入。`TUN`标志告诉内核在`/boot/loader.conf`中查找匹配的可调参数并在OID可访问之前应用它。（我们将在下一小节中设置loader.conf钩子。）

### 将Sysctl连接到Attach和Detach

attach路径现在在cdev创建后调用sysctl构建器：

```c
static int
myfirst_attach(device_t dev)
{
        struct myfirst_softc *sc = device_get_softc(dev);
        struct make_dev_args args;
        int error;

        sc->sc_dev = dev;
        mtx_init(&sc->sc_mtx, "myfirst", NULL, MTX_DEF);
        strlcpy(sc->sc_msg, "Hello from myfirst", sizeof(sc->sc_msg));
        sc->sc_msglen = strlen(sc->sc_msg);

        make_dev_args_init(&args);
        args.mda_devsw = &myfirst_cdevsw;
        args.mda_uid = UID_ROOT;
        args.mda_gid = GID_WHEEL;
        args.mda_mode = 0660;
        args.mda_si_drv1 = sc;
        args.mda_unit = device_get_unit(dev);

        error = make_dev_s(&args, &sc->sc_cdev,
            "myfirst%d", device_get_unit(dev));
        if (error != 0) {
                mtx_destroy(&sc->sc_mtx);
                return (error);
        }

        myfirst_sysctl_attach(sc);

        DPRINTF(sc, MYF_DBG_INIT, "attach: cdev created and sysctl tree built\n");
        return (0);
}
```

detach路径不变：它不需要调用`myfirst_sysctl_detach`。Newbus框架拥有每个设备的sysctl上下文，并在设备分离时自动拆除它。驱动程序只需要清理它在框架之外分配的资源（cdev和互斥锁）。这是优先使用每个设备上下文而不是私有上下文的一个小但真实的原因之一。

### 通过`/boot/loader.conf`的启动时可调参数

驱动程序可以让操作员在启动时通过从loader环境读取值来配置其初始行为。加载器（`/boot/loader.efi`或`/boot/loader`）在内核启动之前解析`/boot/loader.conf`并将变量导出到内核可以查询的小型环境中。

读取loader变量的最简单方法是`TUNABLE_INT_FETCH`：

```c
TUNABLE_INT_FETCH("hw.myfirst.debug_mask_default", &sc->sc_debug);
```

第一个参数是loader变量名。第二个是指向目标的指针，如果变量不存在，它也是默认值。如果变量不存在，调用是静默的，否则写入解析的值。

该调用放在`myfirst_attach`中`myfirst_sysctl_attach`之前。到sysctl树构建时，`sc->sc_debug`已经具有loader提供的值（或编译时默认值），`dev.myfirst.0.debug.mask` OID反映了它。

驱动程序在`/boot/loader.conf`中的代表性条目如下：

```ini
myfirst_load="YES"
hw.myfirst.debug_mask_default="0x06"
```

第一行告诉加载器自动加载`myfirst.ko`。第二行将默认调试掩码设置为`MYF_DBG_OPEN | MYF_DBG_IO`。启动后，`sysctl dev.myfirst.0.debug.mask`报告`6`，操作员可以在运行时修改它而无需重启。

命名约定是宽松的，但遵循一些实践。将loader变量保持在`hw.<driver>.<knob>`下，因为`hw.`命名空间在运行时按约定是只读的，不会受到意外重命名的影响。当值是运行时可修改OID的初始值时，在变量名中使用`default`以使关系清晰。在驱动程序的手册页或本章的参考卡中记录每个loader变量。

### 将Sysctl与调试掩码结合

读者会回忆起第23章，驱动程序已经有一个`sc->sc_debug`字段和一个查询它的`DPRINTF`宏。第3阶段到位后，操作员现在可以从shell操作掩码：

```console
$ sysctl dev.myfirst.0.debug.mask
dev.myfirst.0.debug.mask: 0
$ sysctl dev.myfirst.0.debug.classes
dev.myfirst.0.debug.classes: INIT(0x1) OPEN(0x2) IO(0x4) IOCTL(0x8) ...
$ sudo sysctl dev.myfirst.0.debug.mask=0xff
dev.myfirst.0.debug.mask: 0 -> 255
$ # 现在驱动程序中的每个DPRINTF调用都会打印
```

`classes` OID正是为了让操作员不必记住位值而存在的。`sysctl`将名称和值一起打印，操作员可以从屏幕上复制十六进制值并粘贴到下一个命令中。

相同的机制扩展到驱动程序可能想要公开的任何其他旋钮。具有可调超时的驱动程序将添加：

```c
SYSCTL_ADD_UINT(ctx, child, OID_AUTO, "timeout_ms",
    CTLFLAG_RW | CTLFLAG_TUN, &sc->sc_timeout_ms, 0,
    "Operation timeout in milliseconds");
```

想要按实例启用/禁用功能的驱动程序将添加一个`bool`（用`SYSCTL_ADD_BOOL`声明，这是布尔标志的现代首选类型）或具有两个有效值（0和1）的int。

### 构建第3阶段驱动程序

第3阶段的`Makefile`列出了新的源文件：

```make
KMOD=   myfirst
SRCS=   myfirst.c myfirst_debug.c myfirst_ioctl.c myfirst_sysctl.c

CFLAGS+= -I${.CURDIR}

SYSDIR?= /usr/src/sys

.include <bsd.kmod.mk>
```

在`make`和`kldload`之后，操作员可以立即遍历树：

```console
$ sudo kldload ./myfirst.ko
$ sysctl -a dev.myfirst.0
dev.myfirst.0.debug.classes: INIT(0x1) OPEN(0x2) IO(0x4) IOCTL(0x8) ...
dev.myfirst.0.debug.mask: 0
dev.myfirst.0.message_len: 18
dev.myfirst.0.message: Hello from myfirst
dev.myfirst.0.total_writes: 0
dev.myfirst.0.total_reads: 0
dev.myfirst.0.open_count: 0
dev.myfirst.0.version: 1.7-integration
dev.myfirst.0.%parent: nexus0
dev.myfirst.0.%pnpinfo:
dev.myfirst.0.%location:
dev.myfirst.0.%driver: myfirst
dev.myfirst.0.%desc: myfirst pseudo-device, integration version 1.7
```

打开并读取设备应立即增加计数器：

```console
$ cat /dev/myfirst0
Hello from myfirst
$ sysctl dev.myfirst.0.total_reads
dev.myfirst.0.total_reads: 1
$ sysctl dev.myfirst.0.open_count
dev.myfirst.0.open_count: 0
```

`open_count`显示零，因为`cat`打开设备、读取并立即关闭；到`sysctl`运行时，计数已回到零。要看到非零值，请在另一个终端中保持设备打开：

```console
# 终端 1
$ exec 3< /dev/myfirst0

# 终端 2
$ sysctl dev.myfirst.0.open_count
dev.myfirst.0.open_count: 1

# 终端 1
$ exec 3<&-

# 终端 2
$ sysctl dev.myfirst.0.open_count
dev.myfirst.0.open_count: 0
```

Shell的`exec 3< /dev/myfirst0`在文件描述符3上打开设备并保持它打开，直到`exec 3<&-`关闭它。这是在不编写程序的情况下检查任何驱动程序打开计数指标的有用技术。

### Sysctl的常见陷阱

第一个陷阱是**忘记`CTLFLAG_MPSAFE`**。没有该标志，内核在OID的处理程序周围获取giant锁。对于只读整数，这是无害的；对于大量访问的OID，它会序列化整个内核，是延迟灾难。现代内核代码在各处使用`CTLFLAG_MPSAFE`；缺少该标志是代码早于细粒度锁定迁移的标志，应该审查其正确性。

第二个陷阱是**在驱动程序代码中使用静态OID**。`SYSCTL_INT`和`SYSCTL_STRING`宏（没有`_ADD_`前缀）声明静态OID并将它们放在内核启动时处理的特殊链接器节中。使用这些宏的可加载模块在模块加载时安装OID，但OID将引用编译时不存在的每个实例的字段，导致操作员读取它们时崩溃。修复方法是对所有驱动程序OID使用`SYSCTL_ADD_*`系列。

第三个陷阱是**泄漏每个驱动程序的上下文**。使用自己的`sysctl_ctx_init`和`sysctl_ctx_free`（而不是由`device_get_sysctl_ctx`返回的每个设备上下文）的驱动程序必须记住在detach中调用`sysctl_ctx_free`。忘记这一点会泄漏驱动程序创建的每个OID，并在操作员下次读取其中一个时导致内核崩溃。修复方法是尽可能使用每个设备上下文（框架自动清理）。

第四个陷阱是**将每个实例的状态放在进程共享的OID中**。想要在其所有实例之间共享可调参数的驱动程序可能会试图将其放在`kern.myfirst.foo`或`dev.myfirst.foo`下。后者看起来无害但会破坏：当第二个实例附加时，Newbus尝试创建`dev.myfirst.0.foo`和`dev.myfirst.1.foo`，而现有的`dev.myfirst.foo`（没有单元号）不再在范围内。修复方法是对共享可调参数使用`hw.myfirst.<knob>`，或对每个实例的状态使用每个实例的OID，但不要用相同的名称同时使用两者。

第五个陷阱是**更改OID的类型**。声明为`CTLTYPE_UINT`的OID不能在不使任何通过`sysctlbyname`调用它的用户空间程序失效的情况下更改其类型。如果用户传递了错误大小的缓冲区，内核返回`EINVAL`。修复方法是在版本之间保持类型稳定；如果需要不同的类型，定义一个新的OID名称并弃用旧的。

### 第4节总结

第4节添加了第三个集成表面：`dev.myfirst.0`下的sysctl树。驱动程序现在公开其版本、计数器、当前消息、调试掩码和类别枚举，都带有描述性帮助文本，全部使用Newbus提供的每个设备sysctl上下文构建。调试掩码可以通过`/boot/loader.conf`在启动时设置，并通过`sysctl(8)`在运行时调整。attach中的一小段代码构建了整个树；detach不做任何事情，因为框架自动清理。

第3阶段的驱动程序里程碑是添加`myfirst_sysctl.c`和对`myfirst_attach`的少量扩展。cdevsw、ioctl分发器、调试基础设施和驱动程序的其余部分都没有改变。sysctl树纯粹是附加的。

在第5节中，我们将看一个可选但具有说明性的集成目标：网络栈。大多数驱动程序永远不会成为网络驱动程序，但了解驱动程序如何注册`ifnet`并参与`if_*` API为读者提供了内核用于每个"带有注册接口的子系统"的模式示例。如果驱动程序不是网络驱动程序，读者可以阅读第5节作为背景，然后直接跳到第7节。

## 第5节：网络集成（可选）

### 为什么本节是可选的

`myfirst`驱动程序不是网络驱动程序，在本章也不会成为网络驱动程序。我们构建的cdevsw、ioctl和sysctl接口对它来说已经足够。跟随本章集成非网络驱动程序的读者可以安全地跳到第7节，不会失去任何本质内容。

然而，网络集成是一个更普遍原则的完美示例：许多FreeBSD子系统提供一个注册接口，将驱动程序变成更大框架的参与者。无论框架是网络、存储、USB还是声音，模式都是相同的：驱动程序分配一个框架定义的对象，填充回调，调用注册函数，从那一刻起接收来自框架的回调。即使不编写网络驱动程序，阅读本节也能为本书中的每个其他框架集成建立直觉。

本章使用网络栈作为示例有两个原因。首先，它是被最广泛理解的框架，所以词汇（`ifnet`、`if_attach`、`bpf`）连接到用户可见的命令如`ifconfig(8)`和`tcpdump(8)`。其次，网络注册接口足够小，可以端到端遍历而不失去读者。第6节随后展示了应用于CAM存储栈的相同模式。

### `ifnet`是什么

`ifnet`是网络栈的每个接口对象。它是网络中对应于我们在第2节中使用的`cdev`的东西。就像`cdev`代表`/dev`下的一个设备节点一样，`ifnet`代表`ifconfig`下的一个网络接口。`ifconfig -a`的每一行对应一个`ifnet`。

`ifnet`从网络栈外部是不透明的。驱动程序通过`if_t` typedef看到它，并通过访问器函数（`if_setflags`、`if_getmtu`、`if_settransmitfn`）操作它。不透明性是故意的：它让网络栈演进`ifnet`内部而不在每个版本中破坏每个驱动程序。新驱动程序应仅使用`if_t` API。

驱动程序中`ifnet`的生命周期是：

1. 用`if_alloc(IFT_<type>)` **分配**
2. 用`if_initname(ifp, "myif", unit)` **命名**
3. 为ioctl、transmit、init等**填充回调**
4. 用`if_attach(ifp)` **附加**，使接口可见
5. 用`bpfattach(ifp, ...)` **附加到BPF**，使`tcpdump`可以看到流量
6. ... 接口存活，接收流量，运行ioctl ...
7. 用`bpfdetach(ifp)` **从BPF分离**
8. 用`if_detach(ifp)` **分离**，从可见列表中移除
9. 用`if_free(ifp)` **释放**

生命周期几乎完全镜像cdev生命周期（分配、命名、附加、分离、释放），这并非巧合；网络栈和devfs都从相同的注册接口模式演进而来。

### 使用`disc(4)`的演示

树中最简单的`ifnet`驱动程序示例是`disc(4)`，即丢弃接口。`disc(4)`接受包并静默丢弃它们；其驱动程序代码因此主要是集成脚手架，没有协议逻辑来分散读者注意力。完整的驱动程序位于`/usr/src/sys/net/if_disc.c`。

相关函数是`disc_clone_create`，每当操作员运行`ifconfig disc create`时都会调用：

```c
static int
disc_clone_create(struct if_clone *ifc, int unit, caddr_t params)
{
        struct ifnet     *ifp;
        struct disc_softc *sc;

        sc = malloc(sizeof(struct disc_softc), M_DISC, M_WAITOK | M_ZERO);
        ifp = sc->sc_ifp = if_alloc(IFT_LOOP);
        ifp->if_softc = sc;
        if_initname(ifp, discname, unit);
        ifp->if_mtu = DSMTU;
        ifp->if_flags = IFF_LOOPBACK | IFF_MULTICAST;
        ifp->if_drv_flags = IFF_DRV_RUNNING;
        ifp->if_ioctl = discioctl;
        ifp->if_output = discoutput;
        ifp->if_hdrlen = 0;
        ifp->if_addrlen = 0;
        ifp->if_snd.ifq_maxlen = 20;
        if_attach(ifp);
        bpfattach(ifp, DLT_NULL, sizeof(u_int32_t));

        return (0);
}
```

逐步来看：

`malloc`使用`M_WAITOK | M_ZERO`分配驱动程序的softc。waitok标志是允许的，因为clone-create在可睡眠上下文中运行。zero标志将结构初始化为零，这让驱动程序假设任何未显式设置的字段为零或NULL。

`if_alloc(IFT_LOOP)`从网络栈的池分配`ifnet`。`IFT_LOOP`参数标识接口类型，栈将其用于SNMP风格的报告和一些默认行为。其他常见类型是`IFT_ETHER`（用于以太网驱动程序）和`IFT_TUNNEL`（用于隧道伪设备）。

`if_initname`设置用户可见的名称。`discname`是字符串`"disc"`，`unit`是克隆框架传入的单元号。它们一起形成`disc0`、`disc1`等。

接下来的几行填充回调和每个接口数据：MTU、标志、ioctl处理程序（`discioctl`）、输出函数（`discoutput`）、最大发送队列长度等。这是网络的相当于第2节中的`cdevsw`表；区别在于它被填充到每个接口对象而不是静态表中。

`if_attach(ifp)`使接口对用户空间可见。此调用返回后，`ifconfig disc0`工作，接口出现在`netstat -i`中，协议可以绑定到它。

`bpfattach(ifp, DLT_NULL, ...)`将接口附加到BPF（Berkeley Packet Filter）机制，这是`tcpdump`读取的内容。`DLT_NULL`声明链路层类型为"无链路层"，适用于环回。以太网驱动程序会调用`bpfattach(ifp, DLT_EN10MB, ETHER_HDR_LEN)`。没有`bpfattach`，`tcpdump`无法看到接口的流量，即使接口本身工作。

销毁路径以相反顺序镜像创建路径：

```c
static void
disc_clone_destroy(struct ifnet *ifp)
{
        struct disc_softc *sc;

        sc = ifp->if_softc;

        bpfdetach(ifp);
        if_detach(ifp);
        if_free(ifp);

        free(sc, M_DISC);
}
```

首先`bpfdetach`，因为`tcpdump`可能有引用。接下来`if_detach`，因为网络栈可能仍向接口排队流量；`if_detach`排空队列并从可见列表移除接口。最后`if_free`，因为`ifnet`可能仍被上层尚未完成清理的套接字引用；`if_free`推迟实际释放直到最后一个引用消失。

`disc(4)`驱动程序大约200行。真实的以太网驱动程序接近5000行，但集成样板（分配、initname、附加、bpfattach、分离、bpfdetach、释放）完全相同。额外的4800行是协议特定的细节：描述符环、中断处理程序、MAC地址管理、多播过滤器、统计、链路状态轮询等。每一个都有自己的模式，第28章详细介绍它们。这里涵盖的集成框架是它们所有的基础。

### 操作员如何看到结果

一旦`disc_clone_create`成功返回，操作员可以从shell操作接口：

```console
$ sudo ifconfig disc create
$ ifconfig disc0
disc0: flags=8049<UP,LOOPBACK,RUNNING,MULTICAST> metric 0 mtu 1500
$ sudo ifconfig disc0 inet 169.254.99.99/32
$ sudo tcpdump -i disc0 &
$ ping -c1 169.254.99.99
... ping输出 ...
$ sudo ifconfig disc destroy
```

这些命令中的每一个都触及集成的不同部分：

* `ifconfig disc create`调用`disc_clone_create`，构建`ifnet`并附加它。
* `ifconfig disc0`通过`if_t`访问器读取`ifnet`的标志和MTU。
* `ifconfig disc0 inet 169.254.99.99/32`用`SIOCAIFADDR`（添加地址的ioctl）调用`discioctl`。
* `tcpdump -i disc0`打开`bpfattach`创建的BPF tap。
* `ping -c1`发送一个包，它路由通过`discoutput`，被丢弃，永不返回。
* `ifconfig disc destroy`调用`disc_clone_destroy`，分离并释放。

整个集成在用户空间层面可见。底层协议机制都不需要更改来适应新驱动程序；网络栈的框架已经为它留有位置。

### 此模式推广到什么

相同的注册模式适用于许多其他子系统：

* **声音栈**（`sys/dev/sound`）使用`pcm_register`和`pcm_unregister`使声音设备可见。驱动程序填充缓冲区播放、混音器访问和通道配置的回调。
* **USB栈**（`sys/dev/usb`）使用`usb_attach`和`usb_detach`注册USB设备驱动程序。驱动程序填充传输设置、控制请求和断开连接的回调。
* **GEOM I/O框架**（`sys/geom`）使用`g_attach`和`g_detach`注册存储提供者和消费者。驱动程序填充I/O启动、完成和孤立的回调。
* **CAM SIM框架**（`sys/cam`）使用`cam_sim_alloc`和`xpt_bus_register`注册存储适配器。第6节更详细介绍这一点。
* **kobj方法分发系统**（我们在`device_method_t`背后已经看到）本身是一个注册框架：驱动程序声明方法表，kobj子系统通过它分发调用。

在每种情况下，步骤都是相同的：分配框架的对象，填充回调，调用注册函数，接收流量，干净地注销。词汇变化，但节奏不变。

### 第5节总结

第5节使用网络栈演示了注册风格的集成。驱动程序分配一个`ifnet`，命名它，填充回调，将其附加到栈，附加到BPF，接收流量，并在销毁时以相反顺序拆卸。模式很小且有界；协议机制位于其背后，是第27章的主题。

不编写网络驱动程序的读者即使不将其应用于`myfirst`也能从本节获得有用的回报：FreeBSD内核中的每个其他注册风格集成都遵循相同的形状。一旦分配-命名-填充-附加-流量-分离-释放的节奏被内化，第6节中的存储栈将一目了然。

在第6节中，我们将把相同的镜头应用于CAM存储栈。词汇变化（`cam_sim`、`xpt_bus_register`、`xpt_action`、CCB），但注册形状相同。

## 第6节：CAM存储集成（可选）

### 为什么本节是可选的

`myfirst`不是存储适配器，也不会成为存储适配器。集成非存储驱动程序的读者应该浏览本节了解词汇，注意与第5节镜像的注册形状，然后继续到第7节。

集成存储适配器（SCSI主机总线适配器、NVMe控制器、模拟虚拟存储控制器）的读者将在这里找到CAM期望驱动程序如何与它通信的基本骨架。完整的协议表面足够大，可以填满自己的章节，是第27章的主题；我们在这里涵盖的只是集成框架，精神上与用于网络的`if_alloc`/`if_attach`框架相同。

### CAM是什么

CAM（Common Access Method）是FreeBSD中设备驱动程序层之上的存储子系统。它拥有挂起的I/O请求队列、目标和逻辑单元号（LUN）的抽象概念、将请求发送到正确适配器的路径路由逻辑，以及将块I/O转换为协议特定命令的一组通用外设驱动程序（磁盘用`da(4)`、光盘用`cd(4)`、磁带用`sa(4)`）。驱动程序位于CAM之下，仅负责将命令发送到硬件并报告完成的适配器特定工作。

CAM使用的词汇很小但特定：

* **SIM**（SCSI Interface Module）是框架对存储适配器的视图。驱动程序用`cam_sim_alloc`分配一个，填充回调（action函数），并用`xpt_bus_register`注册它。SIM是存储栈中`ifnet`的类似物。
* **CCB**（CAM Control Block）是单个I/O请求。CAM通过action回调将CCB交给驱动程序；驱动程序检查CCB的`func_code`，执行请求的操作，填充结果，并用`xpt_done`将CCB返回给CAM。CCB是存储栈中`mbuf`的类似物，区别在于CCB同时携带请求和响应。
* **路径**将目标标识为`(bus, target, LUN)`三元组。驱动程序调用`xpt_create_path`构建一个可用于异步事件的路径。
* **XPT**（Transport Layer）是中央CAM分发机制。驱动程序调用`xpt_action`将CCB发送给CAM（或发给自己，用于自目标操作）；CAM最终为针对驱动程序总线的I/O CCB回调到驱动程序的action函数。

### 注册生命周期

对于单通道适配器，注册步骤是：

1. 用`cam_simq_alloc(maxq)`分配CAM设备队列。
2. 用`cam_sim_alloc(action, poll, "name", softc, unit, mtx, max_tagged, max_dev_transactions, devq)`分配SIM。
3. 锁定驱动程序的互斥锁。
4. 用`xpt_bus_register(sim, dev, 0)`注册SIM。
5. 创建驱动程序可用于事件的路径：`xpt_create_path(&path, NULL, cam_sim_path(sim), CAM_TARGET_WILDCARD, CAM_LUN_WILDCARD)`。
6. 解锁互斥锁。

清理以相反顺序运行：

1. 锁定驱动程序的互斥锁。
2. 用`xpt_free_path(path)`释放路径。
3. 用`xpt_bus_deregister(cam_sim_path(sim))`注销SIM。
4. 用`cam_sim_free(sim, TRUE)`释放SIM。`TRUE`参数告诉CAM也释放底层devq；如果驱动程序想要保留devq以重用，传`FALSE`。
5. 解锁互斥锁。

`/usr/src/sys/dev/ahci/ahci.c`中的`ahci(4)`驱动程序是一个很好的现实世界示例。其通道附加路径包括规范序列：

```c
ch->sim = cam_sim_alloc(ahciaction, ahcipoll, "ahcich", ch,
    device_get_unit(dev), (struct mtx *)&ch->mtx,
    (ch->quirks & AHCI_Q_NOCCS) ? 1 : min(2, ch->numslots),
    (ch->caps & AHCI_CAP_SNCQ) ? ch->numslots : 0,
    devq);
if (ch->sim == NULL) {
        cam_simq_free(devq);
        device_printf(dev, "unable to allocate sim\n");
        error = ENOMEM;
        goto err1;
}
if (xpt_bus_register(ch->sim, dev, 0) != CAM_SUCCESS) {
        device_printf(dev, "unable to register xpt bus\n");
        error = ENXIO;
        goto err2;
}
if (xpt_create_path(&ch->path, NULL, cam_sim_path(ch->sim),
    CAM_TARGET_WILDCARD, CAM_LUN_WILDCARD) != CAM_REQ_CMP) {
        device_printf(dev, "unable to create path\n");
        error = ENXIO;
        goto err3;
}
```

`goto`标签（`err1`、`err2`、`err3`）汇入一个单一的清理部分，展开迄今为止已分配的一切。这是FreeBSD驱动程序失败处理的标准模式，正是第7节将编码的规则。

### Action回调

action回调是CAM驱动程序的核心。其签名是`void action(struct cam_sim *sim, union ccb *ccb)`。驱动程序检查`ccb->ccb_h.func_code`并分发：

```c
static void
mydriver_action(struct cam_sim *sim, union ccb *ccb)
{
        struct mydriver_softc *sc;

        sc = cam_sim_softc(sim);

        switch (ccb->ccb_h.func_code) {
        case XPT_SCSI_IO:
                mydriver_start_io(sc, ccb);
                /* 完成是异步的；稍后调用xpt_done */
                return;

        case XPT_RESET_BUS:
                mydriver_reset_bus(sc);
                ccb->ccb_h.status = CAM_REQ_CMP;
                break;

        case XPT_PATH_INQ: {
                struct ccb_pathinq *cpi = &ccb->cpi;

                cpi->version_num = 1;
                cpi->hba_inquiry = PI_SDTR_ABLE | PI_TAG_ABLE;
                cpi->target_sprt = 0;
                cpi->hba_misc = PIM_NOBUSRESET | PIM_SEQSCAN;
                cpi->hba_eng_cnt = 0;
                cpi->max_target = 0;
                cpi->max_lun = 7;
                cpi->initiator_id = 7;
                strncpy(cpi->sim_vid, "FreeBSD", SIM_IDLEN);
                strncpy(cpi->hba_vid, "MyDriver", HBA_IDLEN);
                strncpy(cpi->dev_name, cam_sim_name(sim), DEV_IDLEN);
                cpi->unit_number = cam_sim_unit(sim);
                cpi->bus_id = cam_sim_bus(sim);
                cpi->ccb_h.status = CAM_REQ_CMP;
                break;
        }

        default:
                ccb->ccb_h.status = CAM_REQ_INVALID;
                break;
        }

        xpt_done(ccb);
}
```

三个分支演示了模式：

`XPT_SCSI_IO`是数据路径。驱动程序启动异步I/O（向硬件写入描述符、编程DMA等）并立即返回而不调用`xpt_done`。硬件在几毫秒后完成I/O，引发中断，中断处理程序计算结果，填充CCB的状态，然后才调用`xpt_done`。CAM不要求同步完成；驱动程序可以花费硬件花费的任何时间。

`XPT_RESET_BUS`是同步控制。驱动程序执行重置，设置`CAM_REQ_CMP`，并落入`xpt_done`。没有异步组件。

`XPT_PATH_INQ`是SIM的自我描述。CAM第一次探测SIM时，它发出`XPT_PATH_INQ`并读回总线特征：最大LUN、支持的标志、供应商标识符等。驱动程序填充结构并返回。没有正确的`XPT_PATH_INQ`响应，CAM无法探测SIM后面的目标，驱动程序看起来已注册但无效。

`default`分支对驱动程序未实现的任何功能代码返回`CAM_REQ_INVALID`。CAM对此是宽容的；它只是将请求视为不受支持，要么回退到通用实现，要么将错误暴露给外设驱动程序。

### 操作员如何看到结果

一旦带有CAM的驱动程序调用了`xpt_bus_register`，CAM探测总线，用户可见的结果是`camcontrol devlist`中的一个或多个条目：

```console
$ camcontrol devlist
<MyDriver Volume 1.0>             at scbus0 target 0 lun 0 (pass0,da0)
$ ls /dev/da0
/dev/da0
$ diskinfo /dev/da0
/dev/da0   512 ... ...
```

`/dev`下的`da0`设备是一个CAM外设驱动程序（`da(4)`），包装了CAM在SIM后面发现的LUN。操作员从不直接处理SIM；他们只看到每个块设备使用的标准`/dev/daN`接口。这就是使CAM成为如此高效的集成目标的原因：编写一个SIM，免费获得完整的磁盘风格I/O。

### 模式识别

到这时，读者应该看到与第5节中相同的形状：

| 步骤              | 网络          | CAM                     |
|-------------------|---------------|-------------------------|
| 分配对象          | `if_alloc`          | `cam_sim_alloc`         |
| 命名和配置        | `if_initname`，设置回调 | 隐含在`cam_sim_alloc`参数中 |
| 附加到框架        | `if_attach`        | `xpt_bus_register`      |
| 使可发现          | `bpfattach`         | `xpt_create_path`       |
| 接收流量          | `if_output`回调     | action回调              |
| 完成操作          | （同步）       | `xpt_done(ccb)`         |
| 分离              | `bpfdetach`、`if_detach` | `xpt_free_path`、`xpt_bus_deregister` |
| 释放              | `if_free`           | `cam_sim_free`          |

其他注册接口（声音用`pcm_register`、USB用`usb_attach`、GEOM用`g_attach`）遵循相同的列结构，使用它们自己的词汇。一旦读者看到这个表一次，每个后续集成就是查找名称的问题。

### 第6节总结

第6节概述了CAM SIM的注册接口。驱动程序用`cam_sim_alloc`分配SIM，用`xpt_bus_register`注册它，为事件创建路径，通过action回调接收I/O，用`xpt_done`完成I/O，并在分离时以相反顺序注销。我们在`ifnet`中看到的相同注册风格集成模式适用，词汇有明显的更改。

读者现在已经看到了几乎每个驱动程序需要的三个集成表面（devfs、ioctl、sysctl）和一些驱动程序需要的两个注册风格表面（网络、CAM）。在第7节中，我们将退后一步，编码将一切联系在一起的生命周期规则：attach中的注册顺序、detach中的拆卸顺序，以及区分能干净加载、运行和卸载的驱动程序与泄漏资源或在分离时崩溃的驱动程序的一小组模式。

## 第7节：注册、拆卸和清理规则

### 基本规则

通过多个框架（devfs用于`/dev`、sysctl用于可调参数、`ifnet`用于网络、CAM用于存储、callout用于定时器、taskqueue用于延迟工作等）与内核集成的驱动程序积累了一小组已分配的对象和已注册的回调。这些中的每一个都具有相同的属性：它必须以与创建相反的顺序释放。忘记这一点会将干净的detach变成内核崩溃、在模块卸载时泄漏资源，并在驱动程序不再拥有的子系统中散布悬空指针。

因此，集成的基本规则很简单，尽管干净地应用它需要小心：

> **每次成功的注册必须与一次注销配对。注销的顺序与注册的顺序相反。失败的注册必须在函数返回失败之前触发每个先前成功注册的注销。**

这单个句子描述了整个生命周期规则。本节的其余部分是如何应用它的引导之旅。

### 为什么是相反顺序

相反顺序的规则听起来是任意的；它不是。每次注册都是对框架的承诺，"从现在起直到我调用注销，你可以回调到我、依赖我的状态或交给我工作。"拥有对驱动程序回调并为其持有工作的框架不能在另一个框架仍然可以访问相同状态时安全地拆卸。

例如，假设驱动程序先注册一个callout，然后一个cdev，然后一个sysctl OID。cdev的`read`回调可能会查询callout更新的值；callout反过来可能会读取sysctl OID暴露的状态。如果detach首先拆卸callout，那么在cdev被拆卸时，来自用户空间的`read`可能试图查询callout本应保持刷新的值；该值现在已过时，读取返回无意义的数据。如果detach首先拆卸cdev，那么`read`就不可能再进入了，callout可以安全地取消。顺序很重要。

一般规则是：在拆卸依赖项之前，先拆卸可以调用你的东西。

对于大多数驱动程序，依赖链与创建顺序相同：

* cdev依赖于softc（cdev的回调解引用`si_drv1`）。
* sysctl OID依赖于softc（它们指向softc字段）。
* callout和taskqueue依赖于softc（它们接收softc指针作为参数）。
* 中断处理程序依赖于softc、锁和任何DMA标签。
* DMA标签和总线资源依赖于设备。

如果驱动程序按此顺序创建这些，它应该以完全相反的顺序销毁它们：首先是中断（它们可以随时触发），然后是callout和taskqueue（它们随时执行），然后是cdev（它们接收用户空间调用），然后是sysctl OID（框架自动清理这些），然后是DMA，然后是总线资源，然后是锁。softc本身是最后释放的东西。

### Attach中的`goto err1`模式

应用该规则最难的地方是在attach中，当部分失败可能使驱动程序处于半初始化状态时。FreeBSD的规范模式是`goto`标签链，每个标签代表到该点所需的清理：

```c
static int
myfirst_attach(device_t dev)
{
        struct myfirst_softc *sc = device_get_softc(dev);
        struct make_dev_args args;
        int error;

        sc->sc_dev = dev;
        mtx_init(&sc->sc_mtx, "myfirst", NULL, MTX_DEF);
        strlcpy(sc->sc_msg, "Hello from myfirst", sizeof(sc->sc_msg));
        sc->sc_msglen = strlen(sc->sc_msg);

        TUNABLE_INT_FETCH("hw.myfirst.debug_mask_default", &sc->sc_debug);

        make_dev_args_init(&args);
        args.mda_devsw = &myfirst_cdevsw;
        args.mda_uid = UID_ROOT;
        args.mda_gid = GID_WHEEL;
        args.mda_mode = 0660;
        args.mda_si_drv1 = sc;
        args.mda_unit = device_get_unit(dev);

        error = make_dev_s(&args, &sc->sc_cdev,
            "myfirst%d", device_get_unit(dev));
        if (error != 0)
                goto fail_mtx;

        myfirst_sysctl_attach(sc);

        DPRINTF(sc, MYF_DBG_INIT, "attach: stage 3 complete\n");
        return (0);

fail_mtx:
        mtx_destroy(&sc->sc_mtx);
        return (error);
}
```

这里只有一个错误标签，因为只有一个可能发生真正失败的点（`make_dev_s`调用）。更复杂的驱动程序每个注册步骤会有一个标签。按照约定，每个标签以失败的步骤命名（`fail_mtx`、`fail_cdev`、`fail_sysctl`），每个标签运行其**上方**每一步的清理。处理最后可能失败的标签的清理最长；处理第一个失败的标签的清理最短。

假设的硬件驱动程序的四阶段attach如下：

```c
static int
mydriver_attach(device_t dev)
{
        struct mydriver_softc *sc = device_get_softc(dev);
        int error;

        mtx_init(&sc->sc_mtx, "mydriver", NULL, MTX_DEF);

        error = bus_alloc_resource_any(...);
        if (error != 0)
                goto fail_mtx;

        error = bus_setup_intr(...);
        if (error != 0)
                goto fail_resource;

        error = make_dev_s(...);
        if (error != 0)
                goto fail_intr;

        return (0);

fail_intr:
        bus_teardown_intr(...);
fail_resource:
        bus_release_resource(...);
fail_mtx:
        mtx_destroy(&sc->sc_mtx);
        return (error);
}
```

标签从上到下阅读，与清理操作执行的顺序相同。任何步骤的失败跳转到匹配的标签并落入每个先前成功步骤的清理标签。这种模式如此常见，以至于阅读没有它的驱动程序代码令人不快；审查者期望看到它。

### Detach镜像

Detach应该是成功attach的精确镜像。attach中完成的每个注册在detach中必须有匹配的注销，以相反顺序：

```c
static int
myfirst_detach(device_t dev)
{
        struct myfirst_softc *sc = device_get_softc(dev);

        mtx_lock(&sc->sc_mtx);
        if (sc->sc_open_count > 0) {
                mtx_unlock(&sc->sc_mtx);
                return (EBUSY);
        }
        mtx_unlock(&sc->sc_mtx);

        DPRINTF(sc, MYF_DBG_INIT, "detach: tearing down\n");

        /*
         * destroy_dev排空所有进行中的cdevsw回调。在此
         * 调用返回后，不会有新的open/close/read/write/ioctl
         * 到达，也没有进行中的回调仍在运行。
         */
        destroy_dev(sc->sc_cdev);

        /*
         * 每个设备的sysctl上下文在detach成功返回后
         * 由框架自动拆除。这里不需要做什么。
         */

        mtx_destroy(&sc->sc_mtx);
        return (0);
}
```

detach首先在互斥锁下检查`open_count`；如果有人持有设备打开，detach拒绝（返回`EBUSY`），这样操作员得到清晰的错误而不是崩溃。检查之后，函数以相反顺序拆卸attach分配的一切：先是cdev，然后是sysctl（自动），然后是互斥锁。

早期的`EBUSY`返回是"软"分离模式。它将关闭设备的责任放在操作员身上：`kldunload myfirst`会失败，直到操作员运行`pkill cat`（或其他持有设备打开的程序）。另一种选择是"硬"模式，仅在关键资源被使用时拒绝分离，并接受普通文件描述符是内核排空的责任。硬模式更复杂（通常需要`dev_ref`和`dev_rel`），留作第27章CAM驱动程序部分的主题。

### 模块事件处理程序

到目前为止我们讨论了`attach`和`detach`，即Newbus在驱动程序实例被添加或移除时调用的每个设备生命周期钩子。还有一个每个模块的生命周期，由通过`DRIVER_MODULE`（或`MODULE_VERSION`加上`DECLARE_MODULE`）注册的函数控制。内核在`MOD_LOAD`、`MOD_UNLOAD`和`MOD_SHUTDOWN`时调用此函数。

对于大多数驱动程序，每个模块的钩子是未使用的。`DRIVER_MODULE`默认接受NULL事件处理程序，内核做正确的事情：在`MOD_LOAD`时，它将驱动程序添加到总线的驱动程序列表；在`MOD_UNLOAD`时，它遍历总线并分离每个实例。驱动程序作者只编写`attach`和`detach`。

然而，有些驱动程序确实需要模块级别的钩子。经典情况是驱动程序必须设置所有实例共享的全局资源（全局哈希表、全局互斥锁、全局事件处理程序）。钩子是：

```c
static int
myfirst_modevent(module_t mod, int what, void *arg)
{
        switch (what) {
        case MOD_LOAD:
                /* 分配全局状态 */
                return (0);
        case MOD_UNLOAD:
                /* 释放全局状态 */
                return (0);
        case MOD_SHUTDOWN:
                /* 即将关机；刷新任何重要内容 */
                return (0);
        default:
                return (EOPNOTSUPP);
        }
}

static moduledata_t myfirst_mod = {
        "myfirst", myfirst_modevent, NULL
};
DECLARE_MODULE(myfirst, myfirst_mod, SI_SUB_DRIVERS, SI_ORDER_ANY);
MODULE_VERSION(myfirst, 1);
```

本章中的`myfirst`驱动程序没有全局状态，因此不需要`modevent`。默认的`DRIVER_MODULE`机制就足够了。我们在这里提及钩子以便读者能在更大的驱动程序中识别它。

### `EVENTHANDLER`用于系统事件

一些驱动程序关心内核其他地方发生的事件：进程正在fork、系统正在关闭、网络正在改变状态等。`EVENTHANDLER`机制让驱动程序为命名事件注册回调：

```c
static eventhandler_tag myfirst_eh_tag;

static void
myfirst_shutdown_handler(void *arg, int howto)
{
        /* 系统关闭时调用 */
}

/* 在attach中： */
myfirst_eh_tag = EVENTHANDLER_REGISTER(shutdown_pre_sync,
    myfirst_shutdown_handler, sc, EVENTHANDLER_PRI_ANY);

/* 在detach中： */
EVENTHANDLER_DEREGISTER(shutdown_pre_sync, myfirst_eh_tag);
```

`shutdown_pre_sync`、`shutdown_post_sync`、`shutdown_final`和`vm_lowmem`事件名称是驱动程序中最常用的。每个都是文档化的钩子点，每个都有自己的语义关于驱动程序在回调内可以做什么（睡眠、分配内存、获取锁、与硬件通信）。

基本规则对事件处理程序与对其他一切完全一样：每次成功的`EVENTHANDLER_REGISTER`必须与`EVENTHANDLER_DEREGISTER`以相反顺序配对。忘记注销会在事件处理程序表中留下悬空的函数指针；模块卸载后事件下次触发时，内核将跳转到已释放的内存并崩溃。

### `SYSINIT`用于一次性内核初始化

最后一个值得了解的机制是`SYSINIT(9)`，内核的编译时注册一次性初始化机制。驱动程序代码中的`SYSINIT`声明：

```c
static void
myfirst_sysinit(void *arg __unused)
{
        /* 在内核启动早期运行一次 */
}
SYSINIT(myfirst_init, SI_SUB_DRIVERS, SI_ORDER_FIRST,
    myfirst_sysinit, NULL);
```

声明一个在内核初始化期间特定点运行的函数，在任何用户空间进程存在之前。`SYSINIT`在驱动程序代码中很少需要；函数在模块重新加载时不会重新运行，所以它不给驱动程序设置每次加载状态的机会。大多数认为需要`SYSINIT`的驱动程序实际上想要`MOD_LOAD`事件处理程序。

匹配的`SYSUNINIT(9)`声明：

```c
SYSUNINIT(myfirst_uninit, SI_SUB_DRIVERS, SI_ORDER_FIRST,
    myfirst_sysuninit, NULL);
```

声明一个在相应拆卸点运行的函数。声明的顺序很重要：`SI_SUB_DRIVERS`在`SI_SUB_VFS`之后但在`SI_SUB_KICK_SCHEDULER`之前运行，所以此级别的`SYSINIT`已经可以使用文件系统，但还不能调度进程。

### `bus_generic_detach`和`device_delete_children`

本身是总线的驱动程序（PCI到PCI桥驱动程序、USB集线器驱动程序、总线式虚拟驱动程序）附加了子设备。分离父设备必须首先以正确的顺序分离所有子设备。框架提供了两个辅助函数：

`bus_generic_detach(dev)`遍历设备的子设备并在每个上调用`device_detach`。如果每个子设备成功分离则返回0，如果任何子设备拒绝则返回第一个非零返回码。

`device_delete_children(dev)`调用`bus_generic_detach`然后为每个子设备调用`device_delete_child`，释放子设备结构。

总线式驱动程序的detach应该始终以这两个之一开始：

```c
static int
mybus_detach(device_t dev)
{
        int error;

        error = bus_generic_detach(dev);
        if (error != 0)
                return (error);

        /* 现在可以安全地拆除每总线状态 */
        ...
        return (0);
}
```

如果驱动程序在分离子设备之前拆除总线状态，子设备会发现其父设备的资源被从下面释放而崩溃。因此顺序是：先分离子设备（bus_generic_detach），然后拆除每总线状态。

### 综合运用

生命周期规则可以总结为每个驱动程序都应该通过的小清单：

1. **每次分配都有对应的释放。** 用attach中的`goto err`链和detach中的镜像顺序跟踪这一点。
2. **每次注册都有对应的注销。** 这同样适用于cdev、sysctl、callout、taskqueue、事件处理程序、中断处理程序、DMA标签和总线资源。
3. **拆卸的顺序与设置的顺序相反。** 违反此规则的驱动程序会泄漏、崩溃或两者兼有。
4. **当任何外部可见的资源仍被使用时，detach函数拒绝操作。** `EBUSY`是正确的返回码。
5. **detach函数从不释放softc；框架在detach成功返回后自动执行。**
6. **cdev用`destroy_dev`销毁（不是释放），`destroy_dev`阻塞直到进行中的回调返回。**
7. **每个设备的sysctl上下文自动拆除；驱动程序不为它调用`sysctl_ctx_free`。**
8. **总线式驱动程序首先用`bus_generic_detach`或`device_delete_children`分离子设备，然后拆除每总线状态。**
9. **失败的attach在返回失败代码之前展开每个前面的步骤。**
10. **内核从不看到半附加的驱动程序：attach要么完全成功，要么完全失败。**

第3阶段的`myfirst`驱动程序通过了此清单上的每一项；第9节中的实验让读者注入故意的失败以看到展开运行。

### 第7节总结

第7节编码了将每个前面部分联系在一起的生命周期规则。attach中的`goto err`链和detach中的相反顺序拆卸是读者从现在开始在每个编写的驱动程序中使用的两个模式。模块级别的钩子（`MOD_LOAD`、`MOD_UNLOAD`）、事件处理程序注册（`EVENTHANDLER_REGISTER`）和总线式分离（`bus_generic_detach`）是一些驱动程序需要的变体；对于像`myfirst`这样的单实例伪设备，基本的attach/detach对加上`goto err`链就足够了。

在第8节中，我们将退后一步到本章的另一个元主题：驱动程序如何从第II部分的版本`1.0`演进到第III部分的`1.5-channels`、第23章的`1.6-debug`，以及现在的`1.7-integration`，以及这种演进如何应该在源代码注释、`MODULE_VERSION`声明和sysctl `version` OID等用户可见的地方体现出来。读者将离开第24章，不仅拥有一个完全集成的驱动程序，还拥有驱动程序的版本号如何告诉读者期望什么的规则。

## 第8节：重构和版本化

### 驱动程序有历史

`myfirst`驱动程序不是完全成形的。它从第II部分作为`DRIVER_MODULE`工作原理的单文件演示开始，在第III部分中增长以支持多个实例和每通道状态，在第23章中获得了调试和跟踪基础设施，并在本章中获得了完整的集成表面。每一步都使源代码更大、更强大。

FreeBSD树中的驱动程序有类似的悠久历史。`null(4)`可追溯到1982年；其`cdevsw`已被重构至少三次以适应内核演进，但其用户可见的行为没有改变。`if_ethersubr.c`早于IPv6，其API在每个版本中都增长了新函数，而旧版函数保持不变。驱动程序维护的艺术部分在于知道如何在不破坏之前内容的情况下扩展驱动程序。

本节是一个短暂的停顿，讨论三个密切相关的规则：如何在驱动程序增长时重构它、如何表达它所处的版本，以及如何决定什么算作破坏性更改。本章的运行示例是从`1.6-debug`（第23章结尾）到`1.7-integration`（本章结尾）的过渡，但模式适用于任何驱动程序项目。

### 从一个文件到多个文件

第23章中的`myfirst`驱动程序是一个小但真实的源代码树：

```text
myfirst.c          /* probe、attach、detach、cdevsw、read、write */
myfirst.h          /* softc、函数声明 */
myfirst_debug.c    /* SDT提供者定义 */
myfirst_debug.h    /* DPRINTF、调试类别位 */
Makefile
```

本章第3阶段添加两个新的源文件：

```text
myfirst_ioctl.c    /* ioctl分发器 */
myfirst_ioctl.h    /* 用户空间的PUBLIC ioctl接口 */
myfirst_sysctl.c   /* sysctl OID构造 */
```

将每个新关注点拆分为自己的文件对的决定是故意的。一个单一的2000行`myfirst.c`可以编译、加载并工作，但它也更难阅读、更难测试、更难让共同维护者导航。按关注线（打开/关闭 vs ioctl vs sysctl vs 调试）拆分使每个文件适合一个屏幕，让读者一次理解一个关注点。

模式大致是：

* `<driver>.c`保存probe、attach、detach、cdevsw结构和少量cdevsw回调（打开、关闭、读取、写入）。
* `<driver>.h`保存softc、跨文件共享的函数声明和任何私有常量。**不**由用户空间包含。
* `<driver>_debug.c`和`<driver>_debug.h`保存SDT提供者、DPRINTF宏、调试类别枚举。**不**由用户空间包含。
* `<driver>_ioctl.c`保存ioctl分发器。`<driver>_ioctl.h`是**公共**头文件，仅包含`sys/types.h`和`sys/ioccom.h`，从用户空间代码包含是安全的。
* `<driver>_sysctl.c`保存sysctl OID构造。**不**由用户空间包含。

公共头文件和私有头文件之间的拆分很重要，有两个原因。首先，公共头文件必须在没有内核上下文（用户空间包含时`_KERNEL`未定义）的情况下干净地编译；拉入`sys/lock.h`和`sys/mutex.h`的头文件从用户空间构建会编译失败。其次，公共头文件是驱动程序与用户空间契约的一部分，必须可以安装到系统范围的位置，如`/usr/local/include/myfirst/myfirst_ioctl.h`。意外变为公共的私有头文件是维护陷阱：每个包含它的用户空间程序都固定了驱动程序的内部布局，未来的任何重构都会破坏它们。

本章中的`myfirst_ioctl.h`头文件是驱动程序唯一的公共头文件。它很小、自包含，并且只使用稳定的类型。

### 版本字符串、版本号和API版本

驱动程序携带三个不同的版本，每个意味着不同的东西。

**发布版本**是在`dmesg`中打印、通过`dev.<driver>.0.version`公开、并在对话和文档中使用的人类可读字符串。`myfirst`驱动程序使用点分字符串如`1.6-debug`和`1.7-integration`。格式是约定俗成的；重要的是字符串简短、描述性强且每个发布唯一。

**模块版本**是用`MODULE_VERSION(<name>, <integer>)`声明的整数。内核用它来强制模块之间的依赖关系。依赖于`myfirst`的模块声明`MODULE_DEPEND(other, myfirst, 1, 1, 1)`，其中三个整数是最小、首选和最大可接受版本。提升模块版本意味着"我破坏了与以前版本的兼容性；依赖我的模块必须重建。"

**API版本**是通过`MYFIRSTIOC_GETVER`公开并存储在`MYFIRST_IOCTL_VERSION`常量中的整数。用户空间程序用它来在发出可能失败的ioctl之前检测API漂移。提升API版本意味着"用户空间可见的接口以旧程序无法处理的方式更改。"

三个版本是独立的。同一个发布可能只提升API版本（因为添加了新的ioctl）而不提升模块版本（因为内核内依赖者不受影响）。反过来，改变导出的内核内数据结构布局的重构可能提升模块版本而不提升API版本，因为用户空间看不到变化。

对于`myfirst`，本章使用这些值：

```c
/* myfirst_sysctl.c */
#define MYFIRST_VERSION "1.7-integration"

/* myfirst.c */
MODULE_VERSION(myfirst, 1);

/* myfirst_ioctl.h */
#define MYFIRST_IOCTL_VERSION 1
```

发布是`1.7-integration`因为我们刚刚完成了集成工作。模块版本仍为`1`因为没有内核内依赖者存在。API版本为`1`因为这是公开ioctl的第一章；本章的第2阶段引入了接口，ioctl布局的任何未来更改都必须提升它。

### 何时提升每个版本

提升**发布版本**的规则是"每次驱动程序以操作员可能关心的方式更改时"。添加功能、更改默认行为、修复值得注意的错误都符合条件。发布版本是给人类看的；它应该足够频繁地更改，以便字段信息丰富。

提升**模块版本**的规则是"当驱动程序的内核内用户需要重新编译才能继续工作时"。添加新的内核内函数不是提升（旧的依赖者仍然工作）。删除函数或更改其签名是提升。重命名其他模块读取的结构字段是提升。不在内核中导出任何内容的驱动程序可以永远保持模块版本为1。

提升**API版本**的规则是"当现有的用户空间程序会误解驱动程序的响应或以不明显的方式失败时"。添加新的ioctl不是提升（旧程序不使用它）。更改现有ioctl参数结构的布局是提升。重新编号现有ioctl是提升。尚未向用户发布的驱动程序可以在设计接口时自由更改API版本；一旦第一个用户发布了，每次更改都是公共事件。

### 兼容性垫片

广泛发布的驱动程序会积累兼容性垫片。经典的形状是驱动程序永远支持的"版本1"ioctl，与取代它的"版本2"ioctl并存。使用v1接口的用户空间程序继续工作，使用v2的程序获得新行为，驱动程序携带两条代码路径。

垫片的代价是真实的。每个垫片都是需要测试、文档化和维护的代码。每个垫片也是一个覆盖的API，约束着未来的重构。有五个垫片的驱动程序比有一个垫片的驱动程序更难演进。

因此规则是提前仔细设计，使垫片罕见。三个习惯有帮助：

* **使用命名常量，而不是字面数字。** 使用`MYFIRSTIOC_SETMSG`而不是`0x802004d3`的程序在驱动程序重新编号ioctl时将继续工作，因为头文件和程序都针对新头文件重建。
* **优先添加性更改而不是修改性更改。** 当驱动程序需要暴露新字段时，添加新的ioctl而不是扩展现有结构。旧的ioctl保持其布局；新的携带额外信息。
* **为每个公共结构添加版本。** 与`MYFIRSTIOC_SETMSG_V1`配对的`struct myfirst_v1_args`现在是一个小的注释，以后是一个大的兼容性胜利。

本章中的`myfirst`太小，还没有任何垫片。本章对版本化的唯一让步是`MYFIRSTIOC_GETVER` ioctl，它为未来的维护者提供了一个干净的地方，在需要时添加垫片逻辑。

### 一个实际的重构：拆分`myfirst.c`

从第23章的第3阶段（debug）到本章第3阶段（sysctl）的过渡本身是一个小的重构。起始源有一个单文件的1000行`myfirst.c`和一个小型的`myfirst_debug.c`。结束源有同一个`myfirst.c`减少了约100行，加上两个吸收了新逻辑的新文件（`myfirst_ioctl.c`和`myfirst_sysctl.c`）。

重构步骤是：

1. 添加两个包含新逻辑的新文件。
2. 将新函数声明添加到`myfirst.h`以便cdevsw可以引用`myfirst_ioctl`。
3. 更新`myfirst.c`以从`attach`调用`myfirst_sysctl_attach(sc)`。
4. 更新`Makefile`以在`SRCS`中列出新文件。
5. 构建、加载、测试，并验证驱动程序仍然通过每个第23章的实验。
6. 将发布版本提升到`1.7-integration`。
7. 将`MYFIRSTIOC_GETVER`测试添加到本章的验证脚本。

每个步骤都足够小，可以自己审查。它们都没有触及现有的逻辑，这意味着重构不太可能在以前工作的代码中引入回归。这是添加性重构的规则：通过添加新文件和新声明向外增长驱动程序，保留现有代码，并在尘埃落定后提升版本。

更激进的重构（重命名函数、重新安排结构、更改cdevsw的标志集）将需要不同的规则：每次更改一个提交，每次更改后运行回归测试，并在版本提升中清楚记录重新安排了什么。广泛发布的驱动程序在每个发布中使用此规则；例如，树内的`if_em`驱动程序在FreeBSD的几乎每个小版本中都有多提交的重构，每个提交独立推出并单独测试。

### 三个树内驱动程序比较

FreeBSD源代码树中的三个驱动程序演示了复杂性光谱上三个点的源代码布局规则。将它们作为三元组阅读使模式可见。

`/usr/src/sys/dev/null/null.c`是最小的。它是一个200行的单源文件，带有一个`cdevsw`表、一组回调、没有单独的头文件、没有调试或sysctl机制。整个驱动程序适合三页打印纸。这是整个工作就是存在并吸收（或生成）字节的驱动程序的布局；集成仅在cdev层。

`/usr/src/sys/net/if_disc.c`是一个两文件的网络驱动程序：`if_disc.c`用于驱动程序代码和隐式的`if.h`用于框架。驱动程序向网络栈注册，但没有sysctl树、没有调试子树、没有公共ioctl头文件（它使用框架定义的标准`if_ioctl`集）。这是作为框架实例而非自己的东西的驱动程序的布局；框架定义表面，驱动程序填充槽位。

`/usr/src/sys/dev/ahci/ahci.c`是一个多文件驱动程序，具有用于AHCI核心、PCI附加粘合、设备树FDT附加粘合、机箱管理代码和总线特定逻辑的单独文件。每个文件专门用于一个关注点；中心文件超过5000行，但每个文件的大小是可管理的。这是扩展到真实生产驱动程序的布局：按关注点拆分、通过头文件粘合、使用文件边界作为重构单位。

本章中的`myfirst`驱动程序位于中间。第3阶段有五个源文件：`myfirst.c`（打开/关闭/读取/写入和cdevsw）、`myfirst.h`（softc、声明）、`myfirst_debug.c`和`myfirst_debug.h`（调试和SDT）、`myfirst_ioctl.c`和`myfirst_ioctl.h`（ioctl，后者是公共的）、以及`myfirst_sysctl.c`（sysctl树）。这足以演示按关注点拆分的模式，而没有五十个文件驱动程序的认知负担。需要进一步增长`myfirst`的读者有一个清晰的模板：为新关注点添加一对新文件，将源文件添加到`SRCS`，如果用户空间需要则将公共头文件添加到安装集，并更新`MYFIRST_VERSION`。

### 第8节总结

第8节闭合了本章另一个主题的循环：驱动程序的源代码布局、版本号和重构规则如何跟踪其演进。本章的驱动程序里程碑是`1.7-integration`，同时作为`MYFIRST_VERSION`中的发布字符串、模块版本`1`（因为没有内核内依赖者存在而未更改）和API版本`1`（因为这是第一次公开稳定ioctl接口的章节，所以首次设置）来表达。重构保持添加性，所以不需要垫片。

读者现在已经看到了完整的集成表面：第2到第4节涵盖了三个通用接口（devfs、ioctl、sysctl），第5节和第6节概述了注册风格的集成（网络、CAM），第7节编码了生命周期规则，第8节将整个框架构建为驱动程序版本号应跟踪的演进。本章的其余部分为读者提供了相同材料的动手实践。

### 综合运用：最终的Attach和Detach

在进入实验之前，值得在一个地方看到本章完整的attach和detach函数。它们将每个前面的部分联系在一起：第2节的cdev构建、第3节的ioctl连接、第4节的sysctl树、第7节的生命周期规则以及第8节的版本处理。

第3阶段的完整attach：

```c
static int
myfirst_attach(device_t dev)
{
        struct myfirst_softc *sc = device_get_softc(dev);
        struct make_dev_args args;
        int error;

        /* 1. 保存设备指针并初始化锁。 */
        sc->sc_dev = dev;
        mtx_init(&sc->sc_mtx, "myfirst", NULL, MTX_DEF);

        /* 2. 将驱动程序内状态初始化为默认值。 */
        strlcpy(sc->sc_msg, "Hello from myfirst", sizeof(sc->sc_msg));
        sc->sc_msglen = strlen(sc->sc_msg);
        sc->sc_open_count = 0;
        sc->sc_total_reads = 0;
        sc->sc_total_writes = 0;
        sc->sc_debug = 0;

        /* 3. 读取启动时可调参数的调试掩码。如果
         *    操作员在/boot/loader.conf中设置了
         *    hw.myfirst.debug_mask_default，sc_debug现在持有该值；
         *    否则sc_debug保持为零。
         */
        TUNABLE_INT_FETCH("hw.myfirst.debug_mask_default", &sc->sc_debug);

        /* 4. 构建cdev。args结构为我们提供了类型化的、
         *    可版本化的接口；mda_si_drv1以原子方式将
         *    每个cdev的指针连接到softc，关闭了创建与
         *    赋值之间的竞争窗口。
         */
        make_dev_args_init(&args);
        args.mda_devsw = &myfirst_cdevsw;
        args.mda_uid = UID_ROOT;
        args.mda_gid = GID_WHEEL;
        args.mda_mode = 0660;
        args.mda_si_drv1 = sc;
        args.mda_unit = device_get_unit(dev);

        error = make_dev_s(&args, &sc->sc_cdev,
            "myfirst%d", device_get_unit(dev));
        if (error != 0)
                goto fail_mtx;

        /* 5. 构建sysctl树。框架拥有每个设备的上下文，
         *    所以我们不需要自己跟踪或销毁它；
         *    下面的detach不调用sysctl_ctx_free。
         */
        myfirst_sysctl_attach(sc);

        DPRINTF(sc, MYF_DBG_INIT,
            "attach: stage 3 complete, version " MYFIRST_VERSION "\n");
        return (0);

fail_mtx:
        mtx_destroy(&sc->sc_mtx);
        return (error);
}
```

第3阶段的完整detach：

```c
static int
myfirst_detach(device_t dev)
{
        struct myfirst_softc *sc = device_get_softc(dev);

        /* 1. 如果有人持有设备打开则拒绝detach。本章的
         *    模式是简单的软拒绝；挑战3演示了更复杂的
         *    dev_ref/dev_rel模式，该模式排空进行中的
         *    引用而不是拒绝。
         */
        mtx_lock(&sc->sc_mtx);
        if (sc->sc_open_count > 0) {
                mtx_unlock(&sc->sc_mtx);
                return (EBUSY);
        }
        mtx_unlock(&sc->sc_mtx);

        DPRINTF(sc, MYF_DBG_INIT, "detach: tearing down\n");

        /* 2. 销毁cdev。destroy_dev阻塞直到每个
         *    进行中的cdevsw回调返回；此调用后，
         *    不会有新的open/close/read/write/ioctl到达。
         */
        destroy_dev(sc->sc_cdev);

        /* 3. 每个设备的sysctl上下文在detach成功返回后
         *    由框架自动拆除。这里不需要做什么。
         */

        /* 4. 销毁锁。现在安全是因为cdev已消失且
         *    没有其他代码路径可以获取它。
         */
        mtx_destroy(&sc->sc_mtx);

        return (0);
}
```

有两点值得最后的说明。

操作的顺序是attach的严格反向：attach中先锁，detach中最后销毁锁；attach中接近末尾创建cdev，detach中接近开头销毁cdev；attach中最后创建sysctl树，detach中最先（由框架自动）拆除sysctl树。这是第7节基本规则的具体形式。

detach中的拒绝模式（`if (open_count > 0)`检查）是本章为简单起见的选择。真实驱动程序可能需要更复杂的`dev_ref`/`dev_rel`机制来实现排空式分离；挑战3演示了该变体。对于`myfirst`，简单的拒绝为操作员提供了清晰的错误，并且足够了。

在第9节中，我们从解释转向实践。实验引导读者依次构建第1阶段、第2阶段和第3阶段的集成，每个都配有验证命令和预期输出。实验之后是挑战练习（第10节）、故障排除目录（第11节）以及结束本章的总结和桥梁。

## 动手实验

本节中的实验引导读者从新克隆的工作树到本章添加的每个集成表面。每个实验都足够小，可以在一次会话中完成，并配有确认更改的验证命令。按顺序运行实验；后面的实验建立在前面的实验之上。

`examples/part-05/ch24-integration/`下的配套文件包含三个分阶段的参考驱动程序（`stage1-devfs/`、`stage2-ioctl/`、`stage3-sysctl/`），与本章的里程碑匹配。实验假设读者要么从自己第23章结束的驱动程序（版本`1.6-debug`）开始，要么将适当的阶段目录复制到工作位置，在那里进行更改，并在卡住时参考匹配的分阶段目录。

每个实验使用真实的FreeBSD 14.3系统。虚拟机也可以；不要在生产主机上运行这些实验，因为如果驱动程序有错误，模块加载和卸载可能会挂起或崩溃系统。

### 实验1：构建和加载第1阶段驱动程序

**目标**：将驱动程序从第23章基线（`1.6-debug`）提升到本章第1阶段里程碑（在`/dev/myfirst0`下正确构建的cdev）。

**设置**：

从你自己的第23章结束工作树（版本`1.6-debug`的驱动程序）开始，或将第23章最后阶段的参考树复制到实验目录：

```console
$ cp -r ~/myfirst-1.6-debug ~/myfirst-lab1
$ cd ~/myfirst-lab1
$ ls
Makefile  myfirst.c  myfirst.h  myfirst_debug.c  myfirst_debug.h
```

如果你想与已迁移的第1阶段起始点（已应用`make_dev_s`）进行比较，请参考`examples/part-05/ch24-integration/stage1-devfs/`作为参考解决方案而非起始目录。

**步骤1**：打开`myfirst.c`并找到现有的`make_dev`调用。第23章的代码使用旧的单调用形式。用第2节中的`make_dev_args`形式替换它：

```c
struct make_dev_args args;
int error;

make_dev_args_init(&args);
args.mda_devsw = &myfirst_cdevsw;
args.mda_uid = UID_ROOT;
args.mda_gid = GID_WHEEL;
args.mda_mode = 0660;
args.mda_si_drv1 = sc;
args.mda_unit = device_get_unit(dev);

error = make_dev_s(&args, &sc->sc_cdev,
    "myfirst%d", device_get_unit(dev));
if (error != 0) {
        mtx_destroy(&sc->sc_mtx);
        return (error);
}
```

**步骤2**：将`D_TRACKCLOSE`添加到cdevsw标志（它应该已经有`D_VERSION`）：

```c
static struct cdevsw myfirst_cdevsw = {
        .d_version = D_VERSION,
        .d_flags   = D_TRACKCLOSE,
        .d_name    = "myfirst",
        .d_open    = myfirst_open,
        .d_close   = myfirst_close,
        .d_read    = myfirst_read,
        .d_write   = myfirst_write,
};
```

**步骤3**：确认`myfirst_open`和`myfirst_close`使用`dev->si_drv1`恢复softc：

```c
static int
myfirst_open(struct cdev *dev, int oflags, int devtype, struct thread *td)
{
        struct myfirst_softc *sc = dev->si_drv1;
        ...
}
```

**步骤4**：构建并加载：

```console
$ make
$ sudo kldload ./myfirst.ko
```

**验证**：

```console
$ ls -l /dev/myfirst0
crw-rw---- 1 root wheel 0x... <date> /dev/myfirst0
$ sudo cat /dev/myfirst0
Hello from myfirst
$ sudo kldstat | grep myfirst
N    1 0xffff... 1...    myfirst.ko
$ sudo dmesg | tail
... (来自MYF_DBG_INIT的调试消息)
```

如果`ls`显示错误的所有者、组或模式，重新检查`mda_uid`、`mda_gid`和`mda_mode`值。如果`cat`返回空字符串，检查`myfirst_read`是否从`sc->sc_msg`填充用户缓冲区。如果加载成功但设备未出现，检查cdevsw是否从`make_dev_args`引用。

**清理**：

```console
$ sudo kldunload myfirst
$ ls -l /dev/myfirst0
ls: /dev/myfirst0: No such file or directory
```

成功卸载会删除设备节点。如果卸载失败并显示"Device busy"，检查没有shell或程序持有设备打开（`fstat | grep myfirst0`）。

### 实验2：添加ioctl接口

**目标**：通过添加第3节的四个ioctl命令将驱动程序扩展到第2阶段。

**设置**：

```console
$ cp -r examples/part-05/ch24-integration/stage1-devfs ~/myfirst-lab2
$ cd ~/myfirst-lab2
```

**步骤1**：从第3节的模板创建`myfirst_ioctl.h`。将其放在与其他源文件相同的目录中。包含`sys/ioccom.h`和`sys/types.h`。定义`MYFIRST_MSG_MAX = 256`和四个ioctl编号。不要包含任何仅内核的头文件。

**步骤2**：从第3节的模板创建`myfirst_ioctl.c`。分发器是具有标准`d_ioctl_t`签名的单个函数`myfirst_ioctl`。

**步骤3**：将`myfirst_ioctl.c`添加到`Makefile`中的`SRCS`：

```make
SRCS=   myfirst.c myfirst_debug.c myfirst_ioctl.c
```

**步骤4**：更新cdevsw以将`.d_ioctl`指向新分发器：

```c
.d_ioctl = myfirst_ioctl,
```

**步骤5**：将`sc_msg`和`sc_msglen`添加到softc并在attach中初始化它们：

```c
strlcpy(sc->sc_msg, "Hello from myfirst", sizeof(sc->sc_msg));
sc->sc_msglen = strlen(sc->sc_msg);
```

**步骤6**：构建用户空间配套程序。将`myfirstctl.c`放在同一目录中并创建一个小型`Makefile.user`：

```make
CC?= cc
CFLAGS+= -Wall -Werror -I.

myfirstctl: myfirstctl.c myfirst_ioctl.h
        ${CC} ${CFLAGS} -o myfirstctl myfirstctl.c
```

（注意缩进必须是制表符，不是空格，以便`make`解析规则。）

构建内核模块和配套程序：

```console
$ make
$ make -f Makefile.user
$ sudo kldload ./myfirst.ko
```

**验证**：

```console
$ ./myfirstctl get-version
driver ioctl version: 1
$ ./myfirstctl get-message
Hello from myfirst
$ sudo ./myfirstctl set-message "drivers are fun"
$ ./myfirstctl get-message
drivers are fun
$ sudo ./myfirstctl reset
$ ./myfirstctl get-message

$
```

如果`set-message`返回"Permission denied"，问题是设备模式为`0660`而用户不在`wheel`组中。要么用`sudo`运行（如上面的验证命令所做），要么用`mda_gid`将设备组更改为用户所属的组并重新加载模块。

如果`set-message`返回"Bad file descriptor"，问题是`myfirstctl`以只读方式打开了设备。检查程序是否为`set-message`和`reset`选择`O_RDWR`。

如果任何ioctl返回"Inappropriate ioctl for device"，问题是`myfirst_ioctl.h`中编码的长度与分发器的数据视图不匹配。重新检查`_IOR`/`_IOW`宏和它们声明的结构的大小。

**清理**：

```console
$ sudo kldunload myfirst
```

### 实验3：添加Sysctl树

**目标**：通过添加第4节的sysctl OID并从`/boot/loader.conf`读取可调参数将驱动程序扩展到第3阶段。

**设置**：

```console
$ cp -r examples/part-05/ch24-integration/stage2-ioctl ~/myfirst-lab3
$ cd ~/myfirst-lab3
```

**步骤1**：从第4节的模板创建`myfirst_sysctl.c`。函数`myfirst_sysctl_attach(sc)`构建整个树。

**步骤2**：将`myfirst_sysctl.c`添加到`Makefile`中的`SRCS`。

**步骤3**：更新`myfirst_attach`以调用`TUNABLE_INT_FETCH`和`myfirst_sysctl_attach`：

```c
TUNABLE_INT_FETCH("hw.myfirst.debug_mask_default", &sc->sc_debug);

/* ... 在make_dev_s成功之后： */
myfirst_sysctl_attach(sc);
```

**步骤4**：构建并加载：

```console
$ make
$ sudo kldload ./myfirst.ko
```

**验证**：

```console
$ sysctl -a dev.myfirst.0
dev.myfirst.0.debug.classes: INIT(0x1) OPEN(0x2) IO(0x4) IOCTL(0x8) ...
dev.myfirst.0.debug.mask: 0
dev.myfirst.0.message_len: 18
dev.myfirst.0.message: Hello from myfirst
dev.myfirst.0.total_writes: 0
dev.myfirst.0.total_reads: 0
dev.myfirst.0.open_count: 0
dev.myfirst.0.version: 1.7-integration
```

打开设备一次并重新检查计数器：

```console
$ cat /dev/myfirst0
Hello from myfirst
$ sysctl dev.myfirst.0.total_reads
dev.myfirst.0.total_reads: 1
```

测试loader时可调参数。编辑`/boot/loader.conf`（先备份）：

```console
$ sudo cp /boot/loader.conf /boot/loader.conf.backup
$ sudo sh -c 'echo hw.myfirst.debug_mask_default=\"0x06\" >> /boot/loader.conf'
```

注意这仅在下次重启时生效，且仅在模块由loader加载（而不是启动后由`kldload`加载）时生效。对于不重启的交互式测试，在加载前设置值：

```console
$ sudo kenv hw.myfirst.debug_mask_default=0x06
$ sudo kldload ./myfirst.ko
$ sysctl dev.myfirst.0.debug.mask
dev.myfirst.0.debug.mask: 6
```

如果值是0而不是6，检查`TUNABLE_INT_FETCH`调用是否使用与`kenv`命令相同的字符串。调用必须在`myfirst_sysctl_attach`之前运行，以便值在OID创建时就位。

**清理**：

```console
$ sudo kldunload myfirst
$ sudo cp /boot/loader.conf.backup /boot/loader.conf
```

### 实验4：通过注入失败来演示生命周期

**目标**：通过故意使其中一个步骤失败来看到attach中的`goto err`链实际展开。

**设置**：

```console
$ cp -r examples/part-05/ch24-integration/stage3-sysctl ~/myfirst-lab4
$ cd ~/myfirst-lab4
```

**步骤1**：打开`myfirst.c`并找到`myfirst_attach`。在`make_dev_s`成功后立即插入故意失败：

```c
error = make_dev_s(&args, &sc->sc_cdev,
    "myfirst%d", device_get_unit(dev));
if (error != 0)
        goto fail_mtx;

/* 实验4的故意失败 */
device_printf(dev, "Lab 4: injected failure after make_dev_s\n");
error = ENXIO;
goto fail_cdev;

myfirst_sysctl_attach(sc);
return (0);

fail_cdev:
        destroy_dev(sc->sc_cdev);
fail_mtx:
        mtx_destroy(&sc->sc_mtx);
        return (error);
```

**步骤2**：构建并尝试加载：

```console
$ make
$ sudo kldload ./myfirst.ko
kldload: an error occurred while loading module myfirst. Please check dmesg(8) for more details.
$ sudo dmesg | tail
myfirst0: Lab 4: injected failure after make_dev_s
```

**验证**：

```console
$ ls /dev/myfirst0
ls: /dev/myfirst0: No such file or directory
$ kldstat | grep myfirst
$
```

cdev消失了（`goto fail_cdev`清理销毁了它），模块未加载，没有资源泄漏。如果cdev在失败后仍然存在，清理缺少`destroy_dev`调用。如果下次模块加载尝试内核崩溃，清理正在释放或销毁两次。

**附加**：将失败注入更改为在`make_dev_s`之前发生。清理链现在应该跳过`fail_cdev`标签，只运行`fail_mtx`。验证cdev从未创建且互斥锁被销毁：

```console
$ sudo kldload ./myfirst.ko
$ sudo dmesg | tail
... 没有实验4消息，因为它现在在make_dev_s之前运行 ...
```

**清理**：

在继续之前删除故意失败块。

### 实验5：用DTrace跟踪集成表面

**目标**：使用第23章的SDT探针实时跟踪ioctl、打开、关闭和读写流量。

**设置**：按实验3加载第3阶段驱动程序。

**步骤1**：验证探针对DTrace可见：

```console
$ sudo dtrace -l -P myfirst
   ID   PROVIDER      MODULE    FUNCTION   NAME
... id  myfirst       kernel    -          open
... id  myfirst       kernel    -          close
... id  myfirst       kernel    -          io
... id  myfirst       kernel    -          ioctl
```

如果列表为空，SDT探针未注册。检查`myfirst_debug.c`是否在`SRCS`中，以及`SDT_PROBE_DEFINE*`是否从那里调用。

**步骤2**：在一个终端中打开长时间运行的跟踪：

```console
$ sudo dtrace -n 'myfirst:::ioctl { printf("ioctl cmd=0x%x flags=0x%x", arg1, arg2); }'
dtrace: description 'myfirst:::ioctl' matched 1 probe
```

**步骤3**：在另一个终端中测试驱动程序：

```console
$ ./myfirstctl get-version
$ ./myfirstctl get-message
$ sudo ./myfirstctl set-message "Lab 5"
$ sudo ./myfirstctl reset
```

DTrace终端应显示每个ioctl一行，带有命令代码和文件标志。

**步骤4**：将多个探针组合到一个脚本中：

```console
$ sudo dtrace -n '
    myfirst:::open  { printf("open  pid=%d", pid); }
    myfirst:::close { printf("close pid=%d", pid); }
    myfirst:::io    { printf("io    pid=%d write=%d resid=%d", pid, arg1, arg2); }
    myfirst:::ioctl { printf("ioctl pid=%d cmd=0x%x", pid, arg1); }
'
```

在另一个终端中：

```console
$ cat /dev/myfirst0
$ ./myfirstctl get-version
$ echo "hello" | sudo tee /dev/myfirst0
```

DTrace输出现在显示完整的流量模式，包括每个操作周围的打开和关闭、内部的读取或写入，以及任何ioctl。这就是让SDT探针与cdevsw回调集成的价值：驱动程序暴露的每个集成表面也是DTrace的探测表面。

**清理**：

```console
^C
$ sudo kldunload myfirst
```

### 实验6：集成冒烟测试

**目标**：构建一个单一shell脚本，在一次运行中测试每个集成表面，并生成绿色/红色摘要，读者可以粘贴到错误报告或发布准备检查清单中。

冒烟测试是一个小型的、快速的端到端检查，确认驱动程序存活且每个表面都有响应。它不替代仔细的单元测试；它为读者提供了在投入更多时间之前五秒钟确认没有明显损坏的检查。真实驱动程序都有冒烟测试；本章建议从第一天起为每个新驱动程序添加一个。

**设置**：加载第3阶段驱动程序。

**步骤1**：在工作目录中创建`smoke.sh`：

```sh
#!/bin/sh
# smoke.sh - myfirst驱动程序的端到端冒烟测试。

set -u
fail=0

check() {
        if eval "$1"; then
                printf "  PASS  %s\n" "$2"
        else
                printf "  FAIL  %s\n" "$2"
                fail=$((fail + 1))
        fi
}

echo "=== myfirst integration smoke test ==="

# 1. 模块已加载。
check "kldstat | grep -q myfirst" "module is loaded"

# 2. /dev节点存在且具有正确的模式。
check "test -c /dev/myfirst0" "/dev/myfirst0 exists as a character device"
check "test \"\$(stat -f %Lp /dev/myfirst0)\" = \"660\"" "/dev/myfirst0 is mode 0660"

# 3. sysctl树存在。
check "sysctl -N dev.myfirst.0.version >/dev/null 2>&1" "version OID is present"
check "sysctl -N dev.myfirst.0.debug.mask >/dev/null 2>&1" "debug.mask OID is present"
check "sysctl -N dev.myfirst.0.open_count >/dev/null 2>&1" "open_count OID is present"

# 4. ioctl工作（需要构建myfirstctl）。
check "./myfirstctl get-version >/dev/null" "MYFIRSTIOC_GETVER returns success"
check "./myfirstctl get-message >/dev/null" "MYFIRSTIOC_GETMSG returns success"
check "sudo ./myfirstctl set-message smoke && [ \"\$(./myfirstctl get-message)\" = smoke ]" "MYFIRSTIOC_SETMSG round-trip works"
check "sudo ./myfirstctl reset && [ -z \"\$(./myfirstctl get-message)\" ]" "MYFIRSTIOC_RESET clears state"

# 5. 读写基本路径。
check "echo hello | sudo tee /dev/myfirst0 >/dev/null" "write to /dev/myfirst0 succeeds"
check "[ \"\$(cat /dev/myfirst0)\" = hello ]" "read returns the previously written message"

# 6. 计数器更新。
sudo ./myfirstctl reset >/dev/null
cat /dev/myfirst0 >/dev/null
check "[ \"\$(sysctl -n dev.myfirst.0.total_reads)\" = 1 ]" "total_reads incremented after one read"

# 7. SDT探针已注册。
check "sudo dtrace -l -P myfirst | grep -q open" "myfirst:::open SDT probe is visible"

echo "=== summary ==="
if [ $fail -eq 0 ]; then
        echo "ALL PASS"
        exit 0
else
        printf "%d FAIL\n" "$fail"
        exit 1
fi
```

**步骤2**：使其可执行并运行：

```console
$ chmod +x smoke.sh
$ ./smoke.sh
=== myfirst integration smoke test ===
  PASS  module is loaded
  PASS  /dev/myfirst0 exists as a character device
  PASS  /dev/myfirst0 is mode 0660
  PASS  version OID is present
  PASS  debug.mask OID is present
  PASS  open_count OID is present
  PASS  MYFIRSTIOC_GETVER returns success
  PASS  MYFIRSTIOC_GETMSG returns success
  PASS  MYFIRSTIOC_SETMSG round-trip works
  PASS  MYFIRSTIOC_RESET clears state
  PASS  write to /dev/myfirst0 succeeds
  PASS  read returns the previously written message
  PASS  total_reads incremented after one read
  PASS  myfirst:::open SDT probe is visible
=== summary ===
ALL PASS
```

如果任何检查失败，脚本的输出直接指向损坏的集成表面。"version OID is present"失败意味着sysctl构造未运行；"MYFIRSTIOC_GETVER"失败意味着ioctl分发器未正确连接；"total_reads incremented"失败意味着read回调未在互斥锁下增加计数器。

**验证**：在每次驱动程序更改后重新运行。提交前通过的冒烟测试是防止破坏基本流程的回归的最便宜保险。

### 实验7：在用户空间程序不停止的情况下重新加载

**目标**：确认驱动程序可以在另一个终端中用户空间程序持有打开的文件描述符时被卸载和重新加载。

此测试揭示了本章"软分离"模式旨在防止的生命周期错误。当用户持有设备打开时从detach返回`EBUSY`的驱动程序正在正确地保护自己；让detach成功然后在用户发出ioctl时崩溃的驱动程序是损坏的。

**设置**：加载第3阶段驱动程序。

**步骤1**（终端1）：用长时间运行的命令保持设备打开：

```console
$ sleep 3600 < /dev/myfirst0 &
$ jobs
[1]+ Running                 sleep 3600 < /dev/myfirst0 &
```

**步骤2**（终端2）：尝试卸载：

```console
$ sudo kldunload myfirst
kldunload: can't unload file: Device busy
```

这是预期的行为。本章的`myfirst_detach`检查`open_count > 0`并返回`EBUSY`，而不是在打开的文件描述符下拆除cdev。

**步骤3**（终端2）：验证设备从不同的shell仍然可用：

```console
$ ./myfirstctl get-version
driver ioctl version: 1
$ sysctl dev.myfirst.0.open_count
dev.myfirst.0.open_count: 1
```

打开计数反映了被持有的文件描述符。

**步骤4**（终端1）：释放文件描述符：

```console
$ kill %1
$ wait
```

**步骤5**（终端2）：现在卸载成功：

```console
$ sudo kldunload myfirst
$ sysctl dev.myfirst.0
sysctl: unknown oid 'dev.myfirst.0'
```

OID消失了，因为Newbus在detach成功返回后拆除了每个设备的sysctl上下文。

**验证**：卸载应该每次都成功而没有崩溃。如果步骤5中内核崩溃，原因几乎总是cdev的回调在`destroy_dev`返回时仍在进行中；检查cdevsw的`d_close`是否正确释放了它在`d_open`中获取的任何内容，并检查没有callout或taskqueue仍在调度。

一个附加扩展是编写一个小程序，打开设备，立即调用`MYFIRSTIOC_RESET`，然后循环调用`MYFIRSTIOC_GETVER`几秒钟。在循环运行时，尝试从另一个终端卸载。卸载应该仍然因`EBUSY`而失败；进行中的ioctl不应该损坏任何东西。

### 实验总结

七个实验引导读者完成了完整的集成表面、生命周期规则、冒烟测试和软分离契约。第1阶段添加了cdev；第2阶段添加了ioctl接口；第3阶段添加了sysctl树；生命周期实验（实验4）确认了展开；DTrace实验（实验5）确认了与第23章调试基础设施的集成；冒烟测试（实验6）给了读者可重用的验证脚本；重新加载实验（实验7）确认了软分离契约。

通过所有七个实验的驱动程序处于本章的里程碑版本`1.7-integration`，并为下一章的主题做好了准备。第10节中的挑战练习为读者提供了超越本章所涵盖内容的可选后续工作。

## 第 6 节：CAM 存储集成（可选）

### 为什么本节是可选的

`myfirst` 不是存储适配器，也不会成为存储适配器。集成非存储驱动程序的读者应该浏览本节以了解词汇，注意到与第 5 节镜像的注册形状，然后继续到第 7 节。

集成存储适配器（SCSI 主机总线适配器、NVMe 控制器、模拟虚拟存储控制器）的读者将在这里找到 CAM 期望驱动程序如何与其对话的骨架。完整的协议表面大到足以单独填满一章，是第 27 章的主题；我们在这里涵盖的只是集成框架，在精神上与网络使用的 `if_alloc`/`if_attach` 框架相同。

### CAM 是什么

CAM（Common Access Method，通用访问方法）是 FreeBSD 在设备驱动程序层之上的存储子系统。它拥有待处理 I/O 请求的队列、目标和逻辑单元号（LUN）的抽象概念、将请求发送到正确适配器的路径路由逻辑，以及将块 I/O 转换为协议特定命令的一组通用外围驱动程序（`da(4)` 用于磁盘、`cd(4)` 用于光盘、`sa(4)` 用于磁带）。驱动程序位于 CAM 之下，仅负责将命令发送到硬件和报告完成的适配器特定工作。

CAM 使用的词汇很小但很具体：

* **SIM**（SCSI Interface Module，SCSI 接口模块）是框架对存储适配器的视图。驱动程序用 `cam_sim_alloc` 分配一个 SIM，填充回调（action 函数），然后用 `xpt_bus_register` 注册它。SIM 是存储堆栈中 `ifnet` 的对应物。
* **CCB**（CAM Control Block，CAM 控制块）是单个 I/O 请求。CAM 通过 action 回调将 CCB 交给驱动程序；驱动程序检查 CCB 的 `func_code`，执行请求的操作，填充结果，然后用 `xpt_done` 将 CCB 返回给 CAM。CCB 是存储堆栈中 `mbuf` 的对应物，区别在于 CCB 同时承载请求和响应。
* **路径** 将目标标识为 `(bus, target, LUN)` 三元组。驱动程序调用 `xpt_create_path` 来构建可用于异步事件的路径。
* **XPT**（Transport Layer，传输层）是中央 CAM 分发机制。驱动程序调用 `xpt_action` 将 CCB 发送到 CAM（或发送到自身，用于自定向操作）；CAM 最终通过驱动程序的 action 函数回调到驱动程序中处理针对驱动程序总线的 I/O CCB。

### 注册生命周期

对于单通道适配器，注册步骤为：

1. 使用 `cam_simq_alloc(maxq)` 分配 CAM 设备队列。
2. 使用 `cam_sim_alloc(action, poll, "name", softc, unit, mtx, max_tagged, max_dev_transactions, devq)` 分配 SIM。
3. 锁定驱动程序的互斥锁。
4. 使用 `xpt_bus_register(sim, dev, 0)` 注册 SIM。
5. 创建驱动程序可用于事件的路径：`xpt_create_path(&path, NULL, cam_sim_path(sim), CAM_TARGET_WILDCARD, CAM_LUN_WILDCARD)`。
6. 解锁互斥锁。

清理以相反顺序运行：

1. 锁定驱动程序的互斥锁。
2. 使用 `xpt_free_path(path)` 释放路径。
3. 使用 `xpt_bus_deregister(cam_sim_path(sim))` 注销 SIM。
4. 使用 `cam_sim_free(sim, TRUE)` 释放 SIM。`TRUE` 参数告诉 CAM 同时释放底层 devq；如果驱动程序想保留 devq 以供重用，则传递 `FALSE`。
5. 解锁互斥锁。

`/usr/src/sys/dev/ahci/ahci.c` 中的 `ahci(4)` 驱动程序是一个很好的真实世界示例。其通道附加路径包含规范序列：

```c
ch->sim = cam_sim_alloc(ahciaction, ahcipoll, "ahcich", ch,
    device_get_unit(dev), (struct mtx *)&ch->mtx,
    (ch->quirks & AHCI_Q_NOCCS) ? 1 : min(2, ch->numslots),
    (ch->caps & AHCI_CAP_SNCQ) ? ch->numslots : 0,
    devq);
if (ch->sim == NULL) {
        cam_simq_free(devq);
        device_printf(dev, "unable to allocate sim\n");
        error = ENOMEM;
        goto err1;
}
if (xpt_bus_register(ch->sim, dev, 0) != CAM_SUCCESS) {
        device_printf(dev, "unable to register xpt bus\n");
        error = ENXIO;
        goto err2;
}
if (xpt_create_path(&ch->path, NULL, cam_sim_path(ch->sim),
    CAM_TARGET_WILDCARD, CAM_LUN_WILDCARD) != CAM_REQ_CMP) {
        device_printf(dev, "unable to create path\n");
        error = ENXIO;
        goto err3;
}
```

`goto` 标签（`err1`、`err2`、`err3`）进入单个清理部分，该部分释放到目前为止已分配的所有内容。这是用于失败处理的标准 FreeBSD 驱动程序模式，正是第 7 节将编纂的规范。

### Action 回调

action 回调是 CAM 驱动程序的核心。其签名为 `void action(struct cam_sim *sim, union ccb *ccb)`。驱动程序检查 `ccb->ccb_h.func_code` 并分发：

```c
static void
mydriver_action(struct cam_sim *sim, union ccb *ccb)
{
        struct mydriver_softc *sc;

        sc = cam_sim_softc(sim);

        switch (ccb->ccb_h.func_code) {
        case XPT_SCSI_IO:
                mydriver_start_io(sc, ccb);
                /* completion is asynchronous; xpt_done called later */
                return;

        case XPT_RESET_BUS:
                mydriver_reset_bus(sc);
                ccb->ccb_h.status = CAM_REQ_CMP;
                break;

        case XPT_PATH_INQ: {
                struct ccb_pathinq *cpi = &ccb->cpi;

                cpi->version_num = 1;
                cpi->hba_inquiry = PI_SDTR_ABLE | PI_TAG_ABLE;
                cpi->target_sprt = 0;
                cpi->hba_misc = PIM_NOBUSRESET | PIM_SEQSCAN;
                cpi->hba_eng_cnt = 0;
                cpi->max_target = 0;
                cpi->max_lun = 7;
                cpi->initiator_id = 7;
                strncpy(cpi->sim_vid, "FreeBSD", SIM_IDLEN);
                strncpy(cpi->hba_vid, "MyDriver", HBA_IDLEN);
                strncpy(cpi->dev_name, cam_sim_name(sim), DEV_IDLEN);
                cpi->unit_number = cam_sim_unit(sim);
                cpi->bus_id = cam_sim_bus(sim);
                cpi->ccb_h.status = CAM_REQ_CMP;
                break;
        }

        default:
                ccb->ccb_h.status = CAM_REQ_INVALID;
                break;
        }

        xpt_done(ccb);
}
```

三个分支说明了模式：

`XPT_SCSI_IO` 是数据路径。驱动程序启动异步 I/O（向硬件写入描述符、编程 DMA 等）并立即返回而不调用 `xpt_done`。硬件在几毫秒后完成 I/O，引发中断，中断处理程序计算结果，填充 CCB 的状态，然后才调用 `xpt_done`。CAM 不要求同步完成；驱动程序可以花费硬件所需的任何时间。

`XPT_RESET_BUS` 是同步控制。驱动程序执行重置，设置 `CAM_REQ_CMP`，并落入 `xpt_done`。没有异步组件。

`XPT_PATH_INQ` 是 SIM 的自描述。CAM 第一次探测 SIM 时发出 `XPT_PATH_INQ` 并读回总线特征：最大 LUN、支持的标志、供应商标识符等。驱动程序填充结构并返回。没有正确的 `XPT_PATH_INQ` 响应，CAM 无法探测 SIM 后面的目标，驱动程序看起来已注册但实际上是不活跃的。

`default` 分支对驱动程序未实现的任何功能代码返回 `CAM_REQ_INVALID`。CAM 对此是宽容的；它只是将请求视为不受支持，要么回退到通用实现，要么将错误传递给外围驱动程序。

### 操作员看到的结果

一旦承载 CAM 的驱动程序调用了 `xpt_bus_register`，CAM 就会探测总线，用户可见的结果是 `camcontrol devlist` 中的一个或多个条目：

```console
$ camcontrol devlist
<MyDriver Volume 1.0>             at scbus0 target 0 lun 0 (pass0,da0)
$ ls /dev/da0
/dev/da0
$ diskinfo /dev/da0
/dev/da0   512 ... ...
```

`/dev` 下的 `da0` 设备是一个 CAM 外围驱动程序（`da(4)`），封装了 CAM 在 SIM 后面发现的 LUN。操作员从不直接处理 SIM；他们只看到每个块设备使用的标准 `/dev/daN` 接口。这就是使 CAM 成为如此高效的集成目标的原因：编写一个 SIM，免费获得完整的磁盘式 I/O。

### 模式识别

到现在为止，读者应该看到我们在第 5 节中看到的相同形状：

| 步骤              | 网络          | CAM                     |
|-------------------|---------------------|-------------------------|
| 分配对象   | `if_alloc`          | `cam_sim_alloc`         |
| 命名和配置| `if_initname`，设置回调 | 隐含在 `cam_sim_alloc` 参数中 |
| 附加到框架| `if_attach`        | `xpt_bus_register`      |
| 使可发现 | `bpfattach`         | `xpt_create_path`       |
| 接收流量      | `if_output` 回调| action 回调         |
| 完成操作    | （同步）       | `xpt_done(ccb)`         |
| 分离            | `bpfdetach`，`if_detach` | `xpt_free_path`，`xpt_bus_deregister` |
| 释放              | `if_free`           | `cam_sim_free`          |

其他注册接口（声音的 `pcm_register`，USB 的 `usb_attach`，GEOM 的 `g_attach`）遵循相同的列结构，但使用自己的词汇。一旦读者看到这张表一次，每个后续集成就是查找名称的事情。

### 第 6 节总结

第 6 节概述了 CAM SIM 的注册接口。驱动程序用 `cam_sim_alloc` 分配 SIM，用 `xpt_bus_register` 注册它，为事件创建路径，通过 action 回调接收 I/O，用 `xpt_done` 完成 I/O，并在分离时以相反顺序注销。我们在 `ifnet` 中看到的相同注册风格集成模式适用，但词汇有明显变化。

读者现在已经看到了几乎每个驱动程序需要的三个集成表面（devfs、ioctl、sysctl）和某些驱动程序需要的两个注册风格表面（网络、CAM）。在第 7 节中，我们将退后一步，编纂将所有内容联系在一起的生命周期规范：附加中的注册顺序，分离中的拆卸顺序，以及区分干净加载、运行和卸载的驱动程序与在分离时泄漏资源或恐慌的驱动程序的一小组模式。

## 第 7 节：注册、拆卸和清理规范

### 基本规则

通过多个框架（用于 `/dev` 的 devfs、用于可调参数的 sysctl、用于网络的 `ifnet`、用于存储的 CAM、用于定时器的 callout、用于延迟工作的 taskqueue 等）与内核集成的驱动程序积累了一小组已分配对象和已注册回调。每个都有相同的属性：必须以与创建时相反的顺序释放。忘记这一点会将干净的分离变成内核恐慌，在模块卸载时泄漏资源，并在驱动程序不再拥有的子系统中散布悬空指针。

因此，集成的基本规则非常简单，尽管干净地应用它需要注意：

> **每个成功的注册必须与一个注销配对。注销的顺序与注册的顺序相反。失败的注册必须在函数返回失败之前触发每个先前成功注册的注销。**

这一句话描述了整个生命周期规范。本节的其余部分是如何应用它的导览。

### 为什么是相反顺序

相反顺序规则听起来是任意的；其实不然。每个注册都是对框架的承诺，即"从现在到我调用注销，您可以回调到我，依赖我的状态，或交给我工作"。有一个回调到驱动程序并为其持有工作的框架不能在另一个框架仍然访问相同状态时安全拆卸。

例如，假设驱动程序注册了一个 callout，然后是一个 cdev，然后是一个 sysctl OID。cdev 的 `read` 回调可能查询 callout 更新的值；callout 反过来可能读取 sysctl OID 暴露的状态。如果分离首先拆卸 callout，那么在拆卸 cdev 时，用户空间的 `read` 可能尝试查询 callout 应该保持刷新的值；该值现在已过时，read 返回无意义的内容。如果分离首先拆卸 cdev，那么 `read` 就没有方式再进入，callout 可以安全取消。顺序很重要。

一般规则是：在拆卸它依赖的内容之前，先拆卸能调用到您的东西。

对于大多数驱动程序，依赖链与创建顺序相同：

* cdev 依赖于 softc（cdev 的回调解引用 `si_drv1`）。
* sysctl OID 依赖于 softc（它们指向 softc 字段）。
* callout 和 taskqueue 依赖于 softc（它们接收 softc 指针作为参数）。
* 中断处理程序依赖于 softc、锁和任何 DMA 标签。
* DMA 标签和总线资源依赖于设备。

如果驱动程序按此顺序创建这些，它应该以完全相反的顺序销毁它们：首先是中断（它们可以随时触发），然后是 callout 和 taskqueue（它们随时执行），然后是 cdev（它们接收用户空间调用），然后是 sysctl OID（框架自动清理这些），然后是 DMA，然后是总线资源，然后是锁。softc 本身是最后释放的东西。

### 附加中的 `goto err1` 模式

应用规则最困难的地方是在附加中，当部分失败可能使驱动程序半初始化时。FreeBSD 的规范模式是 `goto` 标签链，每个标签代表到该点所需的清理：

```c
static int
myfirst_attach(device_t dev)
{
        struct myfirst_softc *sc = device_get_softc(dev);
        struct make_dev_args args;
        int error;

        sc->sc_dev = dev;
        mtx_init(&sc->sc_mtx, "myfirst", NULL, MTX_DEF);
        strlcpy(sc->sc_msg, "Hello from myfirst", sizeof(sc->sc_msg));
        sc->sc_msglen = strlen(sc->sc_msg);

        TUNABLE_INT_FETCH("hw.myfirst.debug_mask_default", &sc->sc_debug);

        make_dev_args_init(&args);
        args.mda_devsw = &myfirst_cdevsw;
        args.mda_uid = UID_ROOT;
        args.mda_gid = GID_WHEEL;
        args.mda_mode = 0660;
        args.mda_si_drv1 = sc;
        args.mda_unit = device_get_unit(dev);

        error = make_dev_s(&args, &sc->sc_cdev,
            "myfirst%d", device_get_unit(dev));
        if (error != 0)
                goto fail_mtx;

        myfirst_sysctl_attach(sc);

        DPRINTF(sc, MYF_DBG_INIT, "attach: stage 3 complete\n");
        return (0);

fail_mtx:
        mtx_destroy(&sc->sc_mtx);
        return (error);
}
```

这里只有一个错误标签是因为只有一个真正失败可能发生的点（`make_dev_s` 调用）。更复杂的驱动程序每个注册步骤都有一个标签。按照约定，每个标签以失败的步骤命名（`fail_mtx`、`fail_cdev`、`fail_sysctl`），每个标签运行函数中其 **上方** 每个步骤的清理。处理最后一个可能失败的标签的清理最长；处理第一个失败的标签最短。

假设硬件驱动程序的四阶段附加看起来像：

```c
static int
mydriver_attach(device_t dev)
{
        struct mydriver_softc *sc = device_get_softc(dev);
        int error;

        mtx_init(&sc->sc_mtx, "mydriver", NULL, MTX_DEF);

        error = bus_alloc_resource_any(...);
        if (error != 0)
                goto fail_mtx;

        error = bus_setup_intr(...);
        if (error != 0)
                goto fail_resource;

        error = make_dev_s(...);
        if (error != 0)
                goto fail_intr;

        return (0);

fail_intr:
        bus_teardown_intr(...);
fail_resource:
        bus_release_resource(...);
fail_mtx:
        mtx_destroy(&sc->sc_mtx);
        return (error);
}
```

标签从上到下读取的顺序与清理操作执行的顺序相同。任何步骤的失败跳转到匹配的标签并落入每个先前成功步骤的清理标签。这个模式如此常见，以至于阅读没有它的驱动程序代码会令人不快；审阅者期望看到它。

### 分离镜像

分离应该是成功附加的精确镜像。在附加中完成的每个注册必须在分离中有匹配的注销，以相反顺序：

```c
static int
myfirst_detach(device_t dev)
{
        struct myfirst_softc *sc = device_get_softc(dev);

        mtx_lock(&sc->sc_mtx);
        if (sc->sc_open_count > 0) {
                mtx_unlock(&sc->sc_mtx);
                return (EBUSY);
        }
        mtx_unlock(&sc->sc_mtx);

        DPRINTF(sc, MYF_DBG_INIT, "detach: tearing down\n");

        /*
         * destroy_dev drains any in-flight cdevsw callbacks. After
         * this call returns, no new open/close/read/write/ioctl can
         * arrive, and no in-flight callback is still running.
         */
        destroy_dev(sc->sc_cdev);

        /*
         * The per-device sysctl context is torn down automatically by
         * the framework after detach returns successfully. Nothing to
         * do here.
         */

        mtx_destroy(&sc->sc_mtx);
        return (0);
}
```

分离从在互斥锁下检查 `open_count` 开始；如果有人持有设备打开，分离拒绝（返回 `EBUSY`），以便操作员得到清晰的错误而不是恐慌。检查后，函数以相反顺序拆卸附加分配的所有内容：先是 cdev，然后是 sysctl（自动），然后是互斥锁。

早期的 `EBUSY` 返回是"软"分离模式。它将关闭设备的责任放在操作员身上：`kldunload myfirst` 将失败，直到操作员运行 `pkill cat`（或任何其他持有设备打开的进程）。替代方案是"硬"模式，仅当关键资源在使用中时拒绝分离，并接受普通文件描述符是内核排空的责任。硬模式更复杂（通常需要 `dev_ref` 和 `dev_rel`），留作第 27 章 CAM 驱动程序部分的主题。

### 模块事件处理程序

到目前为止，我们讨论了 `attach` 和 `detach`，即 Newbus 在添加或删除驱动程序实例时调用的每设备生命周期挂钩。还有每模块生命周期，由通过 `DRIVER_MODULE`（或 `MODULE_VERSION` 加 `DECLARE_MODULE`）注册的函数控制。内核在 `MOD_LOAD`、`MOD_UNLOAD` 和 `MOD_SHUTDOWN` 时调用此函数。

对于大多数驱动程序，每模块挂钩未使用。`DRIVER_MODULE` 默认接受 NULL 事件处理程序，内核做正确的事情：在 `MOD_LOAD` 时将驱动程序添加到总线的驱动程序列表中，在 `MOD_UNLOAD` 时遍历总线并分离每个实例。驱动程序作者只编写 `attach` 和 `detach`。

然而，一些驱动程序确实需要模块级挂钩。经典情况是驱动程序必须设置在所有实例之间共享的全局资源（全局哈希表、全局互斥锁、全局事件处理程序）。其挂钩为：

```c
static int
myfirst_modevent(module_t mod, int what, void *arg)
{
        switch (what) {
        case MOD_LOAD:
                /* allocate global state */
                return (0);
        case MOD_UNLOAD:
                /* free global state */
                return (0);
        case MOD_SHUTDOWN:
                /* about to power off; flush anything important */
                return (0);
        default:
                return (EOPNOTSUPP);
        }
}

static moduledata_t myfirst_mod = {
        "myfirst", myfirst_modevent, NULL
};
DECLARE_MODULE(myfirst, myfirst_mod, SI_SUB_DRIVERS, SI_ORDER_ANY);
MODULE_VERSION(myfirst, 1);
```

本章的 `myfirst` 驱动程序没有全局状态，因此不需要 `modevent`。默认的 `DRIVER_MODULE` 机制就足够了。我们在这里提到挂钩是为了让读者可以在更大的驱动程序中识别它。

### 用于系统事件的 `EVENTHANDLER`

一些驱动程序关心内核其他地方发生的事件：进程正在 fork、系统正在关闭、网络正在改变状态等。`EVENTHANDLER` 机制让驱动程序为命名事件注册回调：

```c
static eventhandler_tag myfirst_eh_tag;

static void
myfirst_shutdown_handler(void *arg, int howto)
{
        /* called when the system is shutting down */
}

/* In attach: */
myfirst_eh_tag = EVENTHANDLER_REGISTER(shutdown_pre_sync,
    myfirst_shutdown_handler, sc, EVENTHANDLER_PRI_ANY);

/* In detach: */
EVENTHANDLER_DEREGISTER(shutdown_pre_sync, myfirst_eh_tag);
```

`shutdown_pre_sync`、`shutdown_post_sync`、`shutdown_final` 和 `vm_lowmem` 事件名称是驱动程序中最常用的。每个都是有文档记录的挂钩点，每个都有关于驱动程序在回调内可以做什么（睡眠、分配内存、获取锁、与硬件通信）的自己的语义。

基本规则同样适用于事件处理程序：每个成功的 `EVENTHANDLER_REGISTER` 必须以相反顺序与 `EVENTHANDLER_DEREGISTER` 配对。忘记注销会在事件处理程序表中留下悬空的函数指针；模块卸载后下次事件触发时，内核将跳转到已释放的内存并恐慌。

### 用于一次性内核初始化的 `SYSINIT`

最后一个值得了解的机制是 `SYSINIT(9)`，内核的编译时注册一次性初始化机制。驱动程序代码中的 `SYSINIT` 声明：

```c
static void
myfirst_sysinit(void *arg __unused)
{
        /* runs once, very early at kernel boot */
}
SYSINIT(myfirst_init, SI_SUB_DRIVERS, SI_ORDER_FIRST,
    myfirst_sysinit, NULL);
```

声明一个在内核初始化期间特定点运行的函数，在任何用户空间进程存在之前。`SYSINIT` 在驱动程序代码中很少需要；模块重新加载时函数不会重新运行，因此它不给驱动程序设置每次加载状态的机会。大多数认为需要 `SYSINIT` 的驱动程序实际上需要 `MOD_LOAD` 事件处理程序。

匹配的 `SYSUNINIT(9)` 声明：

```c
SYSUNINIT(myfirst_uninit, SI_SUB_DRIVERS, SI_ORDER_FIRST,
    myfirst_sysuninit, NULL);
```

声明一个在相应拆卸点运行的函数。声明的顺序很重要：`SI_SUB_DRIVERS` 在 `SI_SUB_VFS` 之后但在 `SI_SUB_KICK_SCHEDULER` 之前运行，因此此级别的 `SYSINIT` 已经可以使用文件系统但不能调度进程。

### `bus_generic_detach` 和 `device_delete_children`

本身是总线的驱动程序（PCI-to-PCI 桥驱动程序、USB 集线器驱动程序、总线式虚拟驱动程序）有附加到它们的子设备。分离父设备必须首先以正确顺序分离所有子设备。框架提供两个辅助函数：

`bus_generic_detach(dev)` 遍历设备的子设备并对每个子设备调用 `device_detach`。如果每个子设备都成功分离，它返回 0，如果任何子设备拒绝，它返回第一个非零返回码。

`device_delete_children(dev)` 调用 `bus_generic_detach`，然后对每个子设备调用 `device_delete_child`，释放子设备结构。

总线式驱动程序的分离应始终以这两个之一开始：

```c
static int
mybus_detach(device_t dev)
{
        int error;

        error = bus_generic_detach(dev);
        if (error != 0)
                return (error);

        /* now safe to tear down per-bus state */
        ...
        return (0);
}
```

如果驱动程序在分离子设备之前拆卸其总线状态，子设备会发现其父设备的资源被释放并崩溃。因此顺序是：首先分离子设备（bus_generic_detach），然后拆卸每总线状态。

### 综合起来

生命周期规范可以总结为每个驱动程序都应通过的小型检查清单：

1. **每个分配都有对应的释放。** 通过附加中的 `goto err` 链和分离中的镜像顺序来跟踪这一点。
2. **每个注册都有对应的注销。** 这同样适用于 cdev、sysctl、callout、taskqueue、事件处理程序、中断处理程序、DMA 标签和总线资源。
3. **拆卸的顺序与设置的顺序相反。** 违反这一点的驱动程序将泄漏、恐慌或两者兼有。
4. **如果任何外部可见资源仍在使用中，分离函数拒绝操作。** `EBUSY` 是正确的返回码。
5. **分离函数从不释放 softc；框架在分离成功返回后自动执行。**
6. **cdev 用 `destroy_dev` 销毁（而不是释放），`destroy_dev` 阻塞直到进行中的回调返回。**
7. **每设备 sysctl 上下文自动拆卸；驱动程序不为其调用 `sysctl_ctx_free`。**
8. **总线式驱动程序首先用 `bus_generic_detach` 或 `device_delete_children` 分离子设备，然后拆卸每总线状态。**
9. **失败的附加在返回失败代码之前展开每个先前步骤。**
10. **内核从不看到半附加的驱动程序：附加要么完全成功，要么完全失败。**

阶段 3 的 `myfirst` 驱动程序通过此检查清单的每个项目；第 9 节的实验室让读者注入故意失败以查看展开过程。

### 第 7 节总结

第 7 节编纂了将之前每节联系在一起的生命周期规范。附加中的 `goto err` 链和分离中的反向顺序拆卸是读者从现在开始编写的每个驱动程序中都将使用的两个模式。模块级挂钩（`MOD_LOAD`、`MOD_UNLOAD`）、事件处理程序注册（`EVENTHANDLER_REGISTER`）和总线式分离（`bus_generic_detach`）是一些驱动程序需要的变体；对于像 `myfirst` 这样的单实例伪驱动程序，基本的附加/分离对加上 `goto err` 链就足够了。

在第 8 节中，我们将退后一步到本章的另一个元主题：驱动程序如何从第二部分的版本 `1.0` 演进到第三部分的 `1.5-channels`、第 23 章的 `1.6-debug`，以及现在的 `1.7-integration`，以及这种演进如何应在源代码注释、`MODULE_VERSION` 声明和用户可见位置（如 sysctl `version` OID）中可见。读者离开第 24 章时不仅拥有完全集成的驱动程序，还拥有驱动程序版本号告诉读者期望什么的规范。

## 第 8 节：重构和版本化

### 驱动程序有历史

`myfirst` 驱动程序并非完全成型。它在第二部分以演示 `DRIVER_MODULE` 如何工作的单文件开始，在第三部分增长以支持多个实例和每通道状态，在第 23 章获得调试和跟踪基础设施，并在本章获得完整的集成表面。每一步都使源代码更大、更有能力。

FreeBSD 树中的驱动程序有类似长的历史。`null(4)` 可追溯到 1982 年；其 `cdevsw` 至少重构了三次以适应内核演进，但其用户可见的行为未变。`if_ethersubr.c` 早于 IPv6，其 API 每个版本都在增长新函数，而旧函数保持不变。驱动程序维护的艺术部分在于知道如何在不破坏以前内容的情况下扩展驱动程序。

本节是一个简短的暂停，讨论三个密切相关的规范：如何在驱动程序增长时重构它，如何表达它所处的版本，以及如何决定什么算作破坏性更改。本章的工作示例是从 `1.6-debug`（第 23 章的结尾）到 `1.7-integration`（本章的结尾）的过渡，但模式适用于任何驱动程序项目。

### 从一个文件到多个文件

第 23 章中的 `myfirst` 驱动程序是一个小但真实的源代码树：

```text
myfirst.c          /* probe, attach, detach, cdevsw, read, write */
myfirst.h          /* softc, function declarations */
myfirst_debug.c    /* SDT provider definition */
myfirst_debug.h    /* DPRINTF, debug class bits */
Makefile
```

本章阶段 3 添加两个新源文件：

```text
myfirst_ioctl.c    /* ioctl dispatcher */
myfirst_ioctl.h    /* PUBLIC ioctl interface for user space */
myfirst_sysctl.c   /* sysctl OID construction */
```

将每个新关注点拆分到自己的文件对中的决定是故意的。单个 2000 行的 `myfirst.c` 可以编译、加载和工作，但它也更难阅读、更难测试、更难让共同维护者导航。沿着关注点线（打开/关闭 vs ioctl vs sysctl vs 调试）拆分使每个文件适合一个屏幕，让读者一次理解一个关注点。

模式大致如下：

* `<driver>.c` 包含 probe、attach、detach、cdevsw 结构和少量 cdevsw 回调（open、close、read、write）。
* `<driver>.h` 包含 softc、跨文件共享的函数声明和任何私有常量。**不**被用户空间包含。
* `<driver>_debug.c` 和 `<driver>_debug.h` 包含 SDT 提供者、DPRINTF 宏、调试类枚举。**不**被用户空间包含。
* `<driver>_ioctl.c` 包含 ioctl 分发器。`<driver>_ioctl.h` 是 **公共** 头文件，仅包含 `sys/types.h` 和 `sys/ioccom.h`，可以安全地从用户空间代码包含。
* `<driver>_sysctl.c` 包含 sysctl OID 构造。**不**被用户空间包含。

公共头文件和私有头文件之间的拆分很重要，原因有两个。首先，公共头文件必须在没有内核上下文的情况下干净编译（用户空间包含它们时未定义 `_KERNEL`）；拉入 `sys/lock.h` 和 `sys/mutex.h` 的头文件在用户空间构建中会编译失败。其次，公共头文件是驱动程序与用户空间契约的一部分，必须可安装到系统范围的位置，如 `/usr/local/include/myfirst/myfirst_ioctl.h`。意外成为公共的私有头文件是维护陷阱：包含它的每个用户空间程序都固定了驱动程序的内部布局，任何未来的重构都会破坏它们。

本章的 `myfirst_ioctl.h` 头文件是驱动程序唯一的公共头文件。它很小、自包含，只使用稳定类型。

### 版本字符串、版本号和 API 版本

驱动程序携带三个不同的版本，每个意味着不同的东西。

**发布版本** 是在 `dmesg` 中打印、通过 `dev.<driver>.0.version` 暴露、在对话和文档中使用的可读字符串。`myfirst` 驱动程序使用点分字符串，如 `1.6-debug` 和 `1.7-integration`。格式是约定的；重要的是字符串简短、描述性强且每个发布唯一。

**模块版本** 是用 `MODULE_VERSION(<name>, <integer>)` 声明的整数。内核使用它来强制模块之间的依赖关系。依赖 `myfirst` 的模块声明 `MODULE_DEPEND(other, myfirst, 1, 1, 1)`，其中三个整数是最小、首选和最大可接受版本。提升模块版本意味着"我破坏了与以前版本的兼容性；依赖我的模块必须重新构建。"

**API 版本** 是通过 `MYFIRSTIOC_GETVER` 暴露并存储在 `MYFIRST_IOCTL_VERSION` 常量中的整数。用户空间程序使用它在发出可能失败的 ioctl 之前检测 API 偏移。提升 API 版本意味着"用户空间可见的接口以旧程序无法处理的方式更改了。"

三个版本是独立的。同一发布可以仅提升 API 版本（因为添加了新 ioctl）而不提升模块版本（因为内核内依赖者不受影响）。反过来，更改导出的内核内数据结构布局的重构可以提升模块版本而不提升 API 版本，因为用户空间看不到变化。

对于 `myfirst`，本章使用这些值：

```c
/* myfirst_sysctl.c */
#define MYFIRST_VERSION "1.7-integration"

/* myfirst.c */
MODULE_VERSION(myfirst, 1);

/* myfirst_ioctl.h */
#define MYFIRST_IOCTL_VERSION 1
```

发布版本是 `1.7-integration`，因为我们刚刚落地了集成工作。模块版本保持 `1`，因为没有内核内依赖者存在。API 版本是 `1`，因为这是第一个暴露 ioctl 的章节；本章阶段 2 引入了接口，对 ioctl 布局的任何未来更改都必须提升它。

### 何时提升每个版本

提升 **发布版本** 的规则是"每次驱动程序以操作员可能关心的方式更改时。"添加功能、更改默认行为、修复值得注意的错误都符合条件。发布版本是给人类看的；它应该经常更改，使该字段信息丰富。

提升 **模块版本** 的规则是"当驱动程序的内核内用户需要重新编译才能继续工作时。"添加新的内核内函数不是提升（旧依赖者仍然工作）。删除函数或更改其签名是提升。重命名其他模块读取的结构字段是提升。不在内核中导出任何内容的驱动程序可以将模块版本永远保持为 1。

提升 **API 版本** 的规则是"当现有用户空间程序会误解驱动程序的响应或以不明显的方式失败时。"添加新 ioctl 不是提升（旧程序不使用它）。更改现有 ioctl 参数结构的布局是提升。重新编号现有 ioctl 是提升。尚未向用户发布的驱动程序可以在接口仍在设计时自由更改 API 版本；一旦第一个用户已经针对它发布，每次更改都是公共事件。

### 兼容性垫片

广泛发布的驱动程序积累了兼容性垫片。经典形状是驱动程序永远支持的"版本 1"ioctl，以及取代它的"版本 2"ioctl。使用 v1 接口的用户空间程序继续工作，使用 v2 的程序获得新行为，驱动程序承载两条代码路径。

垫片的成本是真实的。每个垫片都是需要测试、文档和维护的代码。每个垫片也是约束未来重构的覆盖 API。有五个垫片的驱动程序比有一个的更难演进。

因此，规范是预先仔细设计，使垫片罕见。三个习惯有帮助：

* **使用命名常量，而不是字面数字。** 使用 `MYFIRSTIOC_SETMSG` 而不是 `0x802004d3` 的程序在驱动程序重新编号 ioctl 时仍将继续工作，因为头文件和程序都针对新头文件重新构建。
* **优先添加性更改而非修改性更改。** 当驱动程序需要暴露新字段时，添加新 ioctl 而不是扩展现有结构。旧 ioctl 保持其布局；新 ioctl 承载额外信息。
* **为每个公共结构设置版本。** 与 `MYFIRSTIOC_SETMSG_V1` 配对的 `struct myfirst_v1_args` 现在是一个小注解，以后是一个大的兼容性收益。

本章的 `myfirst` 太小，还没有任何垫片。本章对版本控制的唯一让步是 `MYFIRSTIOC_GETVER` ioctl，它为未来的维护者在需要时提供了添加垫片逻辑的干净位置。

### 实践重构：拆分 `myfirst.c`

从第 23 章阶段 3（调试）到本章阶段 3（sysctl）的过渡本身就是一个小型重构。起始源代码有一个 1000 行的 `myfirst.c` 和一个小型 `myfirst_debug.c`。结束源代码有相同的 `myfirst.c` 缩小约 100 行，加上吸收新逻辑的两个新文件（`myfirst_ioctl.c` 和 `myfirst_sysctl.c`）。

重构步骤为：

1. 添加包含新逻辑的两个新文件。
2. 将新函数声明添加到 `myfirst.h`，以便 cdevsw 可以引用 `myfirst_ioctl`。
3. 更新 `myfirst.c` 以从 `attach` 调用 `myfirst_sysctl_attach(sc)`。
4. 更新 `Makefile` 以在 `SRCS` 中列出新文件。
5. 构建、加载、测试，并验证驱动程序仍然通过每个第 23 章实验室。
6. 将发布版本提升到 `1.7-integration`。
7. 将 `MYFIRSTIOC_GETVER` 测试添加到本章的验证脚本。

每个步骤都小到可以单独审阅。它们都不触及现有逻辑，这意味着重构不太可能在以前工作的代码中引入回归。这是添加性重构的规范：通过添加新文件和新声明向外增长驱动程序，保持现有代码不变，在尘埃落定时提升版本。

更激进的重构（重命名函数、重新排列结构、更改 cdevsw 的标志集）需要不同的规范：每次更改一个提交，每次之后运行回归测试，并在版本提升中清楚记录重新排列了什么。广泛发布的驱动程序在每个发布中使用此规范；例如，树内的 `if_em` 驱动程序在 FreeBSD 的几乎每个次要版本中都有多提交重构，每个提交独立推出并单独测试。

### 三个树内驱动程序的比较

FreeBSD 源代码树中的三个驱动程序说明了复杂度谱系上三个点的源代码布局规范。将它们作为一组阅读使模式可见。

`/usr/src/sys/dev/null/null.c` 是最小的。它是一个 200 行的单一源文件，有一个 `cdevsw` 表、一组回调、没有单独的头文件、没有调试或 sysctl 机制。整个驱动程序适合三页打印。这是整个工作就是存在并吸收（或生成）字节的驱动程序的布局；集成仅在 cdev 层。

`/usr/src/sys/net/if_disc.c` 是一个两文件网络驱动程序：`if_disc.c` 用于驱动程序代码和隐式的 `if.h` 用于框架。驱动程序向网络堆栈注册，但没有 sysctl 树、没有调试子树、没有公共 ioctl 头文件（它使用框架定义的标准 `if_ioctl` 集）。这是作为框架实例而不是自己的东西的驱动程序的布局；框架定义表面，驱动程序填充槽位。

`/usr/src/sys/dev/ahci/ahci.c` 是一个多文件驱动程序，有单独的 AHCI 核心、PCI 附加粘合、设备树 FDT 附加粘合、机箱管理代码和总线特定逻辑。每个文件专注于一个关注点；中央文件超过 5000 行，但每文件大小可管理。这是扩展到真实生产驱动程序的布局：按关注点拆分，通过头文件粘合，使用文件边界作为重构单位。

本章的 `myfirst` 驱动程序位于中间。阶段 3 有五个源文件：`myfirst.c`（打开/关闭/读取/写入和 cdevsw）、`myfirst.h`（softc、声明）、`myfirst_debug.c` 和 `myfirst_debug.h`（调试和 SDT）、`myfirst_ioctl.c` 和 `myfirst_ioctl.h`（ioctl，后者是公共的），以及 `myfirst_sysctl.c`（sysctl 树）。这足以演示按关注点拆分的模式，而没有五十文件驱动程序的认知开销。需要进一步增长 `myfirst` 的读者有清晰的模板：为新关注点添加新的文件对，将源文件添加到 `SRCS`，如果用户空间需要则将公共头文件添加到安装集，并更新 `MYFIRST_VERSION`。

### 第 8 节总结

第 8 节关闭了本章另一个主题的循环：驱动程序的源代码布局、版本号和重构规范如何跟踪其演进。本章的驱动程序里程碑是 `1.7-integration`，同时作为 `MYFIRST_VERSION` 中的发布字符串、模块版本 `1`（因为不存在内核内依赖者而未更改）和 API 版本 `1`（因为这是第一个暴露稳定 ioctl 接口的章节而首次设置）表达。重构保持添加性，因此不需要垫片。

读者现在已经看到了完整的集成表面：第 2 到 4 节涵盖了三个通用项（devfs、ioctl、sysctl），第 5 和 6 节概述了注册风格集成（网络、CAM），第 7 节编纂了生命周期规范，第 8 节将整个内容框架为驱动程序版本号应跟踪的演进。本章的其余部分为读者提供相同材料的动手实践。

### 综合起来：最终的附加和分离

在章节进入实验室之前，值得在一个地方看到本章完整的附加和分离函数。它们将之前每节联系在一起：第 2 节的 cdev 构造、第 3 节的 ioctl 连接、第 4 节的 sysctl 树、第 7 节的生命周期规范以及第 8 节的版本处理。

阶段 3 的完整附加：

```c
static int
myfirst_attach(device_t dev)
{
        struct myfirst_softc *sc = device_get_softc(dev);
        struct make_dev_args args;
        int error;

        /* 1. Stash the device pointer and initialise the lock. */
        sc->sc_dev = dev;
        mtx_init(&sc->sc_mtx, "myfirst", NULL, MTX_DEF);

        /* 2. Initialise the in-driver state to its defaults. */
        strlcpy(sc->sc_msg, "Hello from myfirst", sizeof(sc->sc_msg));
        sc->sc_msglen = strlen(sc->sc_msg);
        sc->sc_open_count = 0;
        sc->sc_total_reads = 0;
        sc->sc_total_writes = 0;
        sc->sc_debug = 0;

        /* 3. Read the boot-time tunable for the debug mask. If the
         *    operator set hw.myfirst.debug_mask_default in
         *    /boot/loader.conf, sc_debug now holds that value;
         *    otherwise sc_debug remains zero.
         */
        TUNABLE_INT_FETCH("hw.myfirst.debug_mask_default", &sc->sc_debug);

        /* 4. Construct the cdev. The args struct gives us a typed,
         *    versionable interface; mda_si_drv1 wires the per-cdev
         *    pointer to the softc atomically, closing the race window
         *    between creation and assignment.
         */
        make_dev_args_init(&args);
        args.mda_devsw = &myfirst_cdevsw;
        args.mda_uid = UID_ROOT;
        args.mda_gid = GID_WHEEL;
        args.mda_mode = 0660;
        args.mda_si_drv1 = sc;
        args.mda_unit = device_get_unit(dev);

        error = make_dev_s(&args, &sc->sc_cdev,
            "myfirst%d", device_get_unit(dev));
        if (error != 0)
                goto fail_mtx;

        /* 5. Build the sysctl tree. The framework owns the per-device
         *    context, so we do not need to track or destroy it
         *    ourselves; detach below does not call sysctl_ctx_free.
         */
        myfirst_sysctl_attach(sc);

        DPRINTF(sc, MYF_DBG_INIT,
            "attach: stage 3 complete, version " MYFIRST_VERSION "\n");
        return (0);

fail_mtx:
        mtx_destroy(&sc->sc_mtx);
        return (error);
}
```

阶段 3 的完整分离：

```c
static int
myfirst_detach(device_t dev)
{
        struct myfirst_softc *sc = device_get_softc(dev);

        /* 1. Refuse detach if anyone holds the device open. The
         *    chapter's pattern is the simple soft refusal; Challenge 3
         *    walks through the more elaborate dev_ref/dev_rel pattern
         *    that drains in-flight references rather than refusing.
         */
        mtx_lock(&sc->sc_mtx);
        if (sc->sc_open_count > 0) {
                mtx_unlock(&sc->sc_mtx);
                return (EBUSY);
        }
        mtx_unlock(&sc->sc_mtx);

        DPRINTF(sc, MYF_DBG_INIT, "detach: tearing down\n");

        /* 2. Destroy the cdev. destroy_dev blocks until every
         *    in-flight cdevsw callback returns; after this call,
         *    no new open/close/read/write/ioctl can arrive.
         */
        destroy_dev(sc->sc_cdev);

        /* 3. The per-device sysctl context is torn down automatically
         *    by the framework after detach returns successfully.
         *    Nothing to do here.
         */

        /* 4. Destroy the lock. Safe now because the cdev is gone and
         *    no other code path can take it.
         */
        mtx_destroy(&sc->sc_mtx);

        return (0);
}
```

有两件事值得最后注意。

操作顺序是附加的严格相反：附加中首先锁定，分离中最后销毁锁；附加中接近末尾创建 cdev，分离中接近开头销毁 cdev；附加中最后创建 sysctl 树，分离中首先（由框架自动）拆卸 sysctl 树。这是第 7 节基本规则的具体形式。

分离中的拒绝模式（`if (open_count > 0)` 检查）是本章为简单起见选择的。真实驱动程序可能需要更复杂的 `dev_ref`/`dev_rel` 机制来实现排空分离；挑战 3 遍历了该变体。对于 `myfirst`，简单的拒绝给操作员清晰的错误并且足够了。

在第 9 节中，我们从解释转向实践。实验室引导读者依次构建集成的阶段 1、阶段 2 和阶段 3，每个都有验证命令和预期输出。实验室之后是挑战练习（第 10 节）、故障排除目录（第 11 节）以及关闭章节的总结和桥梁。

## 挑战练习

下面的挑战是可选的，适用于想要将驱动程序推到本章里程碑之外的读者。每个挑战都有明确的目标、关于方法的几点提示，以及关于本章哪些部分包含相关材料的说明。没有哪个挑战有单一的正确答案；鼓励读者与审查者或引用的树内驱动程序比较他们的解决方案。

### 挑战1：添加可变长度Ioctl

**目标**：扩展ioctl接口，使用户空间程序可以传输大于`MYFIRSTIOC_SETMSG`使用的固定256字节的缓冲区。

本章的模式是固定大小的：`MYFIRSTIOC_SETMSG`声明`_IOW('M', 3, char[256])`，内核处理整个copyin。对于更大的缓冲区（比如高达1 MB），需要嵌入指针模式：

```c
struct myfirst_blob {
        size_t  len;
        char   *buf;    /* 用户空间指针 */
};
#define MYFIRSTIOC_SETBLOB _IOW('M', 5, struct myfirst_blob)
```

分发器必须调用`copyin`来传输指针引用的字节；结构本身通过自动copyin传递。提示：强制最大长度（1 MB是合理的）。用`malloc(M_TEMP, len, M_WAITOK)`分配临时内核缓冲区；不要在softc互斥锁内分配。返回前释放它。参考：第3节"ioctl的常见陷阱"，第二个陷阱。

一个附加扩展是添加`MYFIRSTIOC_GETBLOB`，以相同的可变长度格式复制当前消息；注意用户提供的缓冲区比消息短的情况，并决定是截断、返回`ENOMEM`还是写回所需长度。真实驱动程序（`SIOCGIFCAP`、`KIOCGRPC`）使用后一种模式。

### 挑战2：添加每次打开的计数器

**目标**：维护每个文件描述符的计数器（每个`/dev/myfirst0`打开一个数字），而不是我们现在拥有的每个实例计数器。

本章的`sc_open_count`跨所有打开聚合。每次打开的计数器让程序知道自己从描述符读取了多少。提示：使用`cdevsw->d_priv`附加每个fd的结构（包含计数器的`struct myfirst_fdpriv`）。在`myfirst_open`中分配结构，在`myfirst_close`中释放。框架在文件的`f_data`字段中给每个`cdev_priv`一个唯一指针；read和write回调然后可以通过`devfs_get_cdevpriv()`查找每个fd的结构。

参考：`/usr/src/sys/kern/kern_conf.c`中的`devfs_set_cdevpriv`和`devfs_get_cdevpriv`。该模式也被`/usr/src/sys/dev/random/random_harvestq.c`使用。

一个附加扩展是添加一个报告每个fd计数器总和的sysctl OID，并验证它始终等于现有的聚合计数器。差异表示某处缺少增量。

### 挑战3：使用`dev_ref`实现软分离

**目标**：用更干净的"排空到最后关闭，然后分离"模式替换本章的"打开时拒绝分离"模式。

本章的detach在用户持有设备打开时返回`EBUSY`。更优雅的模式使用`dev_ref`/`dev_rel`来计数未完成的引用，并等待计数在完成分离之前降到零。提示：在`myfirst_open`中获取`dev_ref`，在`myfirst_close`中释放它。在detach中，设置"即将消失"标志，然后调用`destroy_dev_drain`（或编写一个小循环，在`dev_refs > 0`时调用`tsleep`），然后调用`destroy_dev`。一旦计数降到零并销毁cdev，正常完成detach。

参考：`/usr/src/sys/kern/kern_conf.c`中的`dev_ref`机制；`/usr/src/sys/fs/cuse`是使用排空模式进行睡眠分离的真实驱动程序。

附加扩展是添加一个报告当前引用计数的sysctl OID，并验证它与打开计数匹配。

### 挑战4：替换静态魔术字母

**目标**：用不与树中其他任何东西冲突的名称替换`myfirst_ioctl.h`中硬编码的`'M'`魔术字母。

本章任意选择了`'M'`并警告了冲突的风险。更防御性的驱动程序使用更长的魔术标识符并从中构造ioctl编号。提示：定义`MYFIRST_IOC_GROUP = 0x83`（或任何未被其他驱动程序使用的字节）。然后`_IOC`宏采用该常量而不是字符字面量。在头文件中用注释记录选择，解释它是如何被选中的。

一个附加练习是在`/usr/src/sys`中grep `_IO[RW]?\\(.\\?'M'`，并生成使用`'M'`的所有现有用途的列表。（有几个，包括MIDI ioctl和其他；调查本身是有教育意义的。）

### 挑战5：为关机添加`EVENTHANDLER`

**目标**：使驱动程序在系统关闭时行为优雅。

本章的驱动程序没有关闭处理程序；如果在加载`myfirst`的情况下系统关闭，框架最终调用detach。更完善的驱动程序为`shutdown_pre_sync`注册`EVENTHANDLER`，以便它可以在文件系统变为只读之前刷新任何进行中的状态。

提示：在attach中用`EVENTHANDLER_REGISTER(shutdown_pre_sync, ...)`注册处理程序。处理程序在相应的关闭阶段被调用。在detach中用`EVENTHANDLER_DEREGISTER`注销。在处理程序内部，将驱动程序设置为静默状态（清除消息、归零计数器）；此时文件系统仍然可写，所以任何通过`printf`的用户反馈将在下次启动后落在`/var/log/messages`中。

参考：第7节"EVENTHANDLER用于系统事件"和`/usr/src/sys/sys/eventhandler.h`获取命名事件的完整列表。

### 挑战6：第二个每个驱动程序的Sysctl子树

**目标**：在`dev.myfirst.0`下添加第二个子树，公开每个线程的统计信息。

本章的树有一个`debug.`子树。完整的驱动程序可能还有一个`stats.`子树（用于按文件描述符细分的读/写统计信息）或`errors.`子树（用于错误计数器）。提示：使用`SYSCTL_ADD_NODE`创建新节点，然后在新节点的`SYSCTL_CHILDREN`下用`SYSCTL_ADD_*`填充它。该模式与现有的`debug.`子树完全相同，只是以不同的名称为根。

参考：第4节"The `myfirst` Sysctl Tree"中现有的`debug.`子树作为模型；`/usr/src/sys/dev/iicbus`中有几个使用多子树sysctl布局的驱动程序。

### 挑战7：跨模块依赖

**目标**：构建一个依赖于`myfirst`并使用其内核内API的第二个小型模块（`myfirst_logger`）。

本章的`myfirst`驱动程序不为内核内用户导出任何符号。添加一个调用`myfirst`的第二个模块练习`MODULE_DEPEND`机制。提示：在`myfirst.h`中声明一个承载符号的函数（也许是`int myfirst_get_message(int unit, char *buf, size_t len)`）并在`myfirst.c`中实现它。用`MODULE_DEPEND(myfirst_logger, myfirst, 1, 1, 1)`构建第二个模块，以便加载`myfirst_logger`时内核自动加载`myfirst`。

一个附加练习是将`myfirst`的模块版本提升到2，以非向后兼容的方式更改导出的内核内数据结构，并观察第二个模块在重新构建针对新版本之前无法加载。参考：第8节"版本字符串、版本号和API版本"。

### 结束挑战

七个挑战的范围从短（挑战4主要是重命名和注释）到实质性（挑战3需要阅读和理解`dev_ref`）。完成所有七个的读者将对本章仅概述的每个集成角落有动手熟悉感。完成任何一个的读者都会比仅有本章有更深的感觉。

## 故障排除

本章中的集成表面位于内核与系统其余部分之间的接缝处。接缝处的问题通常看起来像驱动程序错误，但实际上是缺少标志、头文件中的拼写错误或对谁拥有什么的误解的症状。下面的目录收集了最常见的症状、其可能的原因以及每个的修复方法。

### `kldload`后`/dev/myfirst0`未出现

首先检查模块是否成功加载：

```console
$ kldstat | grep myfirst
```

如果模块未列出，加载失败；查阅`dmesg`获取更具体的消息。最常见的原因是未解析的符号（通常是因为新源文件未在`SRCS`中）。

如果模块已列出但设备节点缺失，`myfirst_attach`内的`make_dev_s`调用可能失败了。在调用旁边添加`device_printf(dev, "make_dev_s returned %d\n", error)`并重试。返回非零的最常见原因是另一个驱动程序已经创建了`/dev/myfirst0`（内核不会静默覆盖现有节点），或者从不可睡眠的上下文用`MAKEDEV_NOWAIT`调用了`make_dev_s`。

一个更微妙的原因是`cdevsw->d_version`不等于`D_VERSION`。内核检查这一点并拒绝注册版本不匹配的cdevsw。修复方法是`static struct cdevsw myfirst_cdevsw = { .d_version = D_VERSION, ... };`，精确如此。

### `cat /dev/myfirst0`返回"Permission denied"

设备存在但用户无法打开它。本章的默认模式是`0660`，默认组是`wheel`。要么用`sudo`运行，要么将`mda_gid`更改为用户的组，或者将`mda_mode`更改为`0666`（后者对教学模块可以，但对生产驱动程序是糟糕的选择，因为任何本地用户都可以打开设备）。

### `ioctl`返回"Inappropriate ioctl for device"

内核返回了`ENOTTY`，这意味着它无法将请求代码匹配到任何cdevsw。两个常见原因是：

* 驱动程序的分发器对命令返回了`ENOIOCTL`。内核将`ENOIOCTL`转换为用户空间的`ENOTTY`。修复方法是在分发器的switch语句中为命令添加case。

* 请求代码中编码的长度与程序使用的实际缓冲区大小不匹配。这发生在头文件重构后，`_IOR`行被编辑但用户空间程序未针对新头文件重新编译。修复方法是针对当前头文件重新编译程序并针对相同源代码重建模块。

### `ioctl`返回"Bad file descriptor"

分发器返回了`EBADF`，这是本章用于"文件未用正确标志打开此命令"的模式。修复方法是对任何改变状态的命令用`O_RDWR`而不是`O_RDONLY`打开设备。`myfirstctl`配套程序已经这样做了；自定义程序可能没有。

### `sysctl dev.myfirst.0`显示树但读取返回"operation not supported"

这通常意味着sysctl OID是用陈旧或无效的处理程序指针添加的。如果读取立即返回`EOPNOTSUPP`（95），原因几乎总是OID用`CTLTYPE_OPAQUE`注册且处理程序未调用`SYSCTL_OUT`。修复方法是使用类型化的`SYSCTL_ADD_*`辅助函数之一（`SYSCTL_ADD_UINT`、`SYSCTL_ADD_STRING`、带有正确格式字符串的`SYSCTL_ADD_PROC`），以便框架知道读取时该做什么。

### `sysctl -w dev.myfirst.0.foo=value`失败并显示"permission denied"

OID可能创建时使用了`CTLFLAG_RD`（只读），而原意是可写变体`CTLFLAG_RW`。重新检查`SYSCTL_ADD_*`调用中的标志字并重建。

如果标志正确且失败持续，用户可能不是作为root运行。默认情况下Sysctl写入需要`PRIV_SYSCTL`特权；写入时使用`sudo`。

### `sysctl`挂起或导致死锁

OID处理程序正在获取giant锁（因为缺少`CTLFLAG_MPSAFE`），同时另一个线程持有giant锁并调用到驱动程序中。修复方法是将`CTLFLAG_MPSAFE`添加到每个OID的标志字中。现代内核假设到处都是MPSAFE；缺少该标志是一个代码审查问题。

一个更微妙的原因是处理程序获取softc互斥锁，同时另一个线程持有softc互斥锁并从sysctl读取。审查处理程序：它应该在互斥锁下计算值但在互斥锁外调用`sysctl_handle_*`。本章的`myfirst_sysctl_message_len`遵循此模式。

### `kldunload myfirst`失败并显示"Device busy"

detach拒绝了，因为某个用户持有设备打开。用`fstat | grep myfirst0`找到他们，要么请他们关闭，要么终止进程。他们释放设备后，卸载将成功。

如果`fstat`显示没有内容但卸载仍然失败，原因最可能是泄漏的`dev_ref`。重新检查驱动程序中获取`dev_ref`的每个代码路径是否也调用了`dev_rel`；特别是，`myfirst_open`内失败之前的任何错误路径必须释放在失败之前获取的任何引用。

### `kldunload myfirst`导致内核崩溃

驱动程序的detach正在销毁或释放内核仍在使用的东西。两个最常见的原因是：

* detach在销毁cdev之前释放了softc。cdev的回调可能仍在进行中；它们解引用`si_drv1`，获得垃圾数据，然后崩溃。修复方法是严格的顺序：首先是`destroy_dev`（它排空进行中的回调），然后是mutex_destroy，然后返回；框架释放softc。

* detach忘记注销事件处理程序。下一个事件在卸载后触发并跳转到已释放的内存。修复方法是为attach中完成的每个`EVENTHANDLER_REGISTER`调用`EVENTHANDLER_DEREGISTER`。

`dmesg`中的`Lock order reversal`和`WITNESS`消息对两种情况都是有用的诊断。带有"page fault while in kernel mode"和损坏的`%rip`值的崩溃是第二种模式；带有通过两个子系统的堆栈跟踪的"lock order reversal"崩溃是第一种。

### DTrace探针不可见

即使模块已加载，`dtrace -l -P myfirst`也返回空。原因几乎总是SDT探针在头文件中声明但未在任何地方定义。探针需要`SDT_PROBE_DECLARE`（在头文件中，消费者看到它们的地方）和`SDT_PROBE_DEFINE*`（在恰好一个源文件中，拥有探针存储的地方）。本章的模式将定义放在`myfirst_debug.c`中。如果该文件未在`SRCS`中，探针将不会被定义，DTrace将什么也看不到。

一个更微妙的原因是SDT探针在头文件中重命名但匹配的`SDT_PROBE_DEFINE*`未更新。构建仍然成功，因为两个声明引用不同的符号，但DTrace只看到定义的名称。审查头文件和源文件中的相同探针名称。

### sysctl树在卸载后仍然存在但下次sysctl时挂起

当驱动程序使用自己的sysctl上下文（而不是每个设备的上下文）并忘记在detach中调用`sysctl_ctx_free`时会发生这种情况。OID引用现在已释放的softc中的字段；下一次`sysctl`遍历解引用已释放的内存，内核要么崩溃要么返回垃圾数据。修复方法是切换到`device_get_sysctl_ctx`，框架自动清理它。

### 常规诊断清单

当出现问题时原因不明显，在求助于`kgdb`之前遍历这个简短的列表：

1. `kldstat | grep <driver>`：模块真的加载了吗？
2. `dmesg | tail`：有提到驱动程序的内核消息吗？
3. `ls -l /dev/<driver>0`：设备节点存在且具有预期的模式吗？
4. `sysctl dev.<driver>.0.%driver`：Newbus知道该设备吗？
5. `fstat | grep <driver>0`：有人持有设备打开吗？
6. `dtrace -l -P <driver>`：SDT探针已注册吗？
7. 重新阅读attach函数并检查每个步骤在detach中都有匹配的清理。

前六个命令只需十秒钟，可以排除大多数常见问题。第七个是慢的，但几乎总是前六个没有暴露的任何错误的最终答案。

### 常见问题解答

以下问题在集成工作中经常出现，本章以简短的FAQ结束。每个答案故意简洁；本章的相关部分有完整的讨论。

**Q1. 为什么同时使用ioctl和sysctl，它们似乎重叠？**

它们回答不同的问题。ioctl是已经打开设备并想要发出命令的程序的正确通道（请求状态、推送新状态、触发动作）。sysctl是在shell提示符下想要在不打开任何东西的情况下检查或调整状态的操作员或脚本的正确通道。相同的值可以通过两个接口公开，许多生产驱动程序正是这样做：一个用于程序的`MYFIRSTIOC_GETMSG`和一个用于人类的`dev.myfirst.0.message`。每个用户选择适合其上下文的通道。

**Q2. 什么时候应该使用mmap而不是read/write/ioctl？**

当数据量大、随机访问且自然存在于内存地址（帧缓冲区、DMA描述符环、内存映射寄存器空间）时使用`mmap`。当数据是顺序的、面向字节的且每次调用较小时使用`read`/`write`。对控制命令使用`ioctl`。这三种不是对立的；许多驱动程序公开所有三种（如控制台用的`vt(4)`）。

**Q3. 为什么本章使用`make_dev_s`而不是`make_dev`？**

`make_dev_s`是现代首选的形式。它返回显式错误而不是在重复名称时崩溃；它接受args结构以便可以在没有混乱的情况下添加新选项；它是大多数当前驱动程序使用的。旧的`make_dev`仍然工作，但不鼓励用于新代码。

**Q4. 我需要声明`D_TRACKCLOSE`吗？**

如果你的驱动程序的`d_close`应该只在文件描述符的最后一次关闭时调用（"关闭"的自然含义），你需要它。没有它，内核为每个重复描述符的每次关闭调用`d_close`，这会困扰大多数驱动程序。在任何新的cdevsw中设置它，除非你有特定的理由不这样做。

**Q5. 什么时候应该提升`MODULE_VERSION`？**

当驱动程序的内核内API以不兼容方式更改时。添加新的导出符号是可以的；重命名或删除它们是提升。更改公开可见结构的布局是提升。提升模块版本强制依赖者（`MODULE_DEPEND`消费者）重建。

**Q6. 什么时候应该在公共头文件中提升API版本常量？**

当用户可见接口以不兼容方式更改时。添加新的ioctl是可以的；更改现有ioctl参数结构的布局是提升。重新编号现有ioctl是提升。提升API版本让用户空间程序在发出调用之前检测不兼容性。

**Q7. 我应该在`myfirst_detach`中分离我的OID吗？**

不，如果你使用了`device_get_sysctl_ctx`（每个设备的上下文）就不需要。框架在成功detach后自动清理每个设备的上下文。只有在你使用`sysctl_ctx_init`创建自己的上下文时才需要显式清理。

**Q8. 为什么我的detach因"invalid memory access"而崩溃？**

几乎总是因为当驱动程序释放它们引用的东西时cdev的回调仍在进行中。修复方法是首先调用`destroy_dev(sc->sc_cdev)`；`destroy_dev`阻塞直到每个进行中的回调返回。它返回后，cdev消失了，没有新的回调可以到达。只有那时释放softc、释放锁等才是安全的。严格的顺序是不可商量的。

**Q9. `dev_ref`/`dev_rel`与`D_TRACKCLOSE`有什么区别？**

`D_TRACKCLOSE`是一个cdevsw标志，控制内核何时调用`d_close`：有它，只在最后一次关闭时；没有它，在每次关闭时。`dev_ref`/`dev_rel`是一个引用计数机制，让驱动程序推迟分离直到未完成的引用被释放。它们是不相关且互补的。本章在阶段1中使用`D_TRACKCLOSE`；挑战3演示`dev_ref`/`dev_rel`。

**Q10. 为什么我的sysctl写入返回EPERM即使我是root？**

三个可能的原因。（a）OID创建时只有`CTLFLAG_RD`；添加`CTLFLAG_RW`。（b）OID有`CTLFLAG_SECURE`且系统处于`securelevel > 0`；降低securelevel或移除标志。（c）用户实际上不是root而是在没有`allow.sysvipc`或类似的jail中；jail内的root对任意OID没有`PRIV_SYSCTL`。

**Q11. 我的sysctl处理程序在不应该的时候获取giant锁。我忘记了什么？**

标志字中的`CTLFLAG_MPSAFE`。没有它，内核在每次调用处理程序时获取giant锁。到处添加它；现代内核假设到处都是MPSAFE。

**Q12. 我应该将ioctl组字母命名大写还是小写？**

新驱动程序用大写。小写字母被基本子系统大量使用（`'d'`用于磁盘、`'i'`用于`if_ioctl`、`'t'`用于终端），冲突的机会是真实的。大写字母大部分是空闲的，新驱动程序应该选择其中一个。

**Q13. 我的ioctl返回`Inappropriate ioctl for device`，我不明白为什么。**

内核返回`ENOTTY`，因为要么（a）分发器对命令返回`ENOIOCTL`（为它添加case），要么（b）请求代码中编码的长度与用户传递的缓冲区不匹配（双方针对相同的头文件重新编译）。

**Q14. 我应该在内核中使用`strncpy`还是`strlcpy`？**

`strlcpy`。它保证NUL终止且永远不会溢出目标。`strncpy`两者都不做，是微妙错误的常见来源。FreeBSD `style(9)`手册页推荐所有新代码使用`strlcpy`。

**Q15. 我的模块加载但`dmesg`没有显示来自驱动程序的任何消息。有什么问题？**

驱动程序的调试掩码为零。本章的`DPRINTF`宏只在设置掩码位时打印。在加载前设置掩码（`kenv hw.myfirst.debug_mask_default=0xff`），或在加载后设置（`sysctl dev.myfirst.0.debug.mask=0xff`）。

**Q16. 为什么本章如此频繁地提到DTrace？**

因为它是FreeBSD内核中最有效的调试工具，第23章的调试基础设施旨在与之集成。SDT探针为操作员提供了进入每个集成表面的运行时tap，而无需重建驱动程序。公开命名良好的SDT探针的驱动程序比不公开的驱动程序更容易调试。

**Q17. 我可以将此驱动程序用作真实硬件驱动程序的模板吗？**

集成表面（cdev、ioctl、sysctl）直接转换。硬件特定的部分（资源分配、中断处理、DMA设置）来自第IV部分的第18到22章。真实的PCI驱动程序通常将第IV部分的结构模式与本章的集成模式结合，得出一个可发布的驱动程序。

**Q18. 如何在不重建驱动程序的情况下授予非root用户对`/dev/myfirst0`的访问权限？**

使用`devfs.rules(5)`。在`/etc/devfs.rules`下添加匹配设备名称并在运行时设置所有者、组或模式的规则文件。例如，让`operator`组读写`/dev/myfirst*`：

```text
[myfirst_rules=10]
add path 'myfirst*' mode 0660 group operator
```

在`/etc/rc.conf`中用`devfs_system_ruleset="myfirst_rules"`启用规则集并执行`service devfs restart`。驱动程序的`mda_uid`、`mda_gid`和`mda_mode`仍然在创建时设置默认值；`devfs.rules`让管理员在不触及源代码的情况下覆盖它们。

**Q19. 我的`SRCS`列表不断增长。那是问题吗？**

本身不是。内核模块`Makefile`中的`SRCS`行列出编译到模块中的每个源文件；随着新职责获得自己的文件而增长列表是正常和预期的。本章的第3阶段驱动程序已经有四个源文件（`myfirst.c`、`myfirst_debug.c`、`myfirst_ioctl.c`、`myfirst_sysctl.c`），第25章将添加更多。警告信号不是条目数量，而是缺乏结构：如果`SRCS`包含没有命名方案合并在一起的不相关文件，驱动程序已经超出了其布局，值得一次小的重构。第25章将此重构视为一等习惯。

**Q20. 我接下来应该做什么？**

阅读第25章（高级主题和实用技巧）将此集成驱动程序转变为一个*可维护的*驱动程序，如果你想动手实践可以完成本章的挑战，并查看本章参考卡中引用的树内驱动程序之一作为完整示例。`null(4)`驱动程序是最温和的入口；`if_em`以太网驱动程序是最完整的；`ahci(4)`存储驱动程序展示了CAM模式。选择最接近你想构建的，并从头到尾阅读它。

## 总结

本章将`myfirst`从一个工作的但孤立的模块带入了一个完全集成的FreeBSD驱动程序。主线是刻意的：每一节都添加了一个具体的集成表面，并以驱动程序比开始时更有用、更易发现结束。在`/dev/myfirst0`下正确构建的cdev、在公共头文件中定义的四个设计良好的ioctl、`dev.myfirst.0`下的自描述sysctl树、通过`/boot/loader.conf`的启动时可调参数、以及在加载/卸载循环后存活且不泄漏资源的干净生命周期。

技术里程碑如下：

* 第1阶段（第2节）用现代的`make_dev_args`形式替换了旧的`make_dev`调用，填充了`D_TRACKCLOSE`，通过`si_drv1`连接了每个cdev的状态，并遍历了从创建到排空到销毁的cdev生命周期。驱动程序的`/dev`存在成为了一等公民。

* 第2阶段（第3节）添加了`MYFIRSTIOC_GETVER`、`MYFIRSTIOC_GETMSG`、`MYFIRSTIOC_SETMSG`和`MYFIRSTIOC_RESET` ioctl以及匹配的`myfirst_ioctl.h`公共头文件。分发器重用了第23章的调试基础设施（`MYF_DBG_IOCTL`和`myfirst:::ioctl` SDT探针）。配套的`myfirstctl`用户空间程序演示了小型命令行工具如何测试每个ioctl而无需手动解码请求代码。

* 第3阶段（第4节）添加了`dev.myfirst.0.*` sysctl树，包括让操作员在运行时检查和修改调试掩码的`debug.`子树、报告集成发布的`version` OID、读写活动的计数器以及当前消息的字符串OID。启动时可调参数`hw.myfirst.debug_mask_default`让操作员在附加前预加载调试掩码。

* 第5节和第6节概述了应用于网络栈（`if_alloc`、`if_attach`、`bpfattach`）和CAM存储栈（`cam_sim_alloc`、`xpt_bus_register`、`xpt_action`）的相同注册风格集成。不构建网络或存储驱动程序的读者仍然获得了一个有用的模式：FreeBSD中的每个框架注册都使用相同的分配-命名-填充-附加-流量-分离-释放形状。

* 第7节编码了将一切联系在一起的生命周期规则：每次成功的注册必须以相反顺序与注销配对，失败的attach必须在返回之前展开每个先前的步骤。`goto err`链是此规则的规范编码。

* 第8节将本章框架构建为更长主线中的一个步骤：`myfirst`从第II部分的单文件演示开始，在第III部分和第IV部分中增长为多文件驱动程序，在第23章中获得了调试和跟踪，并在这里获得了集成表面。发布版本、模块版本和API版本各自跟踪该演进的不同方面；在正确的时间提升每个是长期存在驱动程序的版本规则。

本章的实验（第9节）引导读者完成每个里程碑，挑战（第10节）为有动力的读者提供了后续工作，故障排除目录（第11节）收集了最常见的症状和修复方法以供快速参考。

结果是一个读者可以带入下一章而没有任何未完成的集成工作等待困扰他们的驱动程序里程碑（`1.7-integration`）。本章的模式（cdev构建、ioctl设计、sysctl树、生命周期规则）也是第V部分其余部分和第VI部分、第VII部分大部分内容将假设读者知道的相同模式。

## 通往第25章的桥梁

第25章（高级主题和实用技巧）通过将本章的集成驱动程序转变为一个*可维护的*驱动程序来结束第5部分。第24章添加了让驱动程序与系统其余部分对话的接口，第25章教授了随着驱动程序吸收下一年的错误修复、可移植性更改和功能请求而保持这些接口稳定和可读的工程习惯。驱动程序从`1.7-integration`增长到`1.8-maintenance`；可见的添加是适度的，但其背后的规则是将存活一个开发周期的驱动程序与存活十年的驱动程序区分开来的东西。

从第24章到第25章的桥梁有四个具体部分。

首先，第25章引入的限速日志直接建立在第23章的`DPRINTF`宏和本章添加的集成表面之上。围绕`ppsratecheck(9)`构建的新`DLOG_RL`宏让驱动程序保持已经使用的相同调试类别，但在事件风暴期间不会淹没`dmesg`。规则很小：选择每秒限制，将其折叠到现有的调试调用站点，并审计少数无限制的`device_printf`可能循环运行的地方。

其次，本章构建的ioctl和sysctl路径将在第25章中审查一致的errno词汇。本章区分`EINVAL`与`ENXIO`、`ENOIOCTL`与`ENOTTY`、`EBUSY`与`EAGAIN`、`EPERM`与`EACCES`，以便每个集成表面在每个失败路径上返回正确的代码。读者遍历第3节编写的分发器和第4节编写的sysctl处理程序，并在返回错误的地方进行调整。

第三，第4节引入的启动时可调参数`hw.myfirst.debug_mask_default`将在第25章中推广为通过`TUNABLE_INT_FETCH`、`TUNABLE_LONG_FETCH`、`TUNABLE_BOOL_FETCH`和`TUNABLE_STR_FETCH`的小型但规则的可调参数词汇，与`CTLFLAG_TUN`下的可写sysctl协作。本章确定的相同`MYFIRST_VERSION`、`MODULE_VERSION`和`MYFIRST_IOCTL_VERSION`三元组将用`MYFIRSTIOC_GETCAPS` ioctl扩展，以便用户空间工具可以在运行时检测功能而无需试错。

第四，第7节引入的`goto err`链将从实验练习提升为驱动程序的生产清理模式，本章的重构将把Newbus附加逻辑和cdev回调移动到单独的文件（`myfirst_bus.c`和`myfirst_cdev.c`）以及用于新日志宏的`myfirst_log.c`。第25章还引入了`SYSINIT(9)`和`SYSUNINIT(9)`用于驱动程序范围的初始化，以及通过`EVENTHANDLER(9)`的`shutdown_pre_sync`事件处理程序，添加了两个更多的注册风格表面到本章已经教授的表面中。

带着集成词汇已经到位的信心继续阅读。第25章将此驱动程序并为长途做好准备；第6部分然后开始依赖于第5部分每个习惯的传输特定章节。

## 参考卡和词汇表

本章的剩余页面是一个紧凑的参考。它们设计为第一次通读，然后在读者需要查找内容时跳入。顺序是：重要宏、结构和标志的参考卡；集成词汇词汇表；以及随章发布的配套文件的简短目录。

由于篇幅限制，完整的快速参考表、词汇表和配套文件清单请参阅英文原版文档。以下是关键概念的简要总结：

### 关键宏和函数摘要

**cdev构建**：`make_dev_args_init()`、`make_dev_s()`、`destroy_dev()`

**ioctl编码**：`_IO()`、`_IOR()`、`_IOW()`、`_IOWR()`

**sysctl OID**：`SYSCTL_ADD_UINT()`、`SYSCTL_ADD_STRING()`、`SYSCTL_ADD_PROC()`、`SYSCTL_ADD_NODE()`

**网络集成**：`if_alloc()`、`if_initname()`、`if_attach()`、`bpfattach()`

**CAM集成**：`cam_sim_alloc()`、`xpt_bus_register()`、`xpt_done()`

### 核心词汇

- **cdev**：字符设备，内核的每个设备节点对象
- **cdevsw**：字符设备开关，回调的分发表
- **devfs**：设备文件系统，支持`/dev`的虚拟文件系统
- **ifnet**：网络栈的每个接口对象
- **CAM**：Common Access Method，FreeBSD的存储子系统
- **SIM**：SCSI Interface Module，CAM对存储适配器的视图
- **CCB**：CAM Control Block，单个I/O请求
- **OID**：对象标识符，sysctl树中的节点
- **SDT**：静态定义跟踪，DTrace使用的探针机制

### 读者自检

在翻页之前，完成本章的读者应该能够在不查阅文本的情况下回答以下问题。每个问题映射到介绍底层材料的章节。如果问题不熟悉，括号中列出的章节节是继续前进之前重温的正确位置。

1. `D_TRACKCLOSE`如何改变`d_close`的调用方式？（第2节）
2. 为什么`mda_si_drv1`优于在`make_dev`返回后赋值`si_drv1`？（第2节）
3. `_IOR('M', 1, uint32_t)`宏在结果请求代码中编码了什么？（第3节）
4. 为什么分发器的默认分支必须返回`ENOIOCTL`而不是`EINVAL`？（第3节）
5. 哪个内核函数拆除每个设备的sysctl上下文，它何时运行？（第4节）
6. `CTLFLAG_TUN`如何与`TUNABLE_INT_FETCH`协作来应用启动时的值？（第4节）
7. `MYFIRST_VERSION`字符串、`MODULE_VERSION`整数和`MYFIRST_IOCTL_VERSION`整数之间的区别是什么？（第8节）
8. 为什么attach中的清理链使用反向顺序的标记`goto`而不是嵌套的`if`语句？（第7节）
9. 本章的软分离模式与挑战3概述的`dev_ref`/`dev_rel`模式有何不同？（第7节和第10节）
10. 哪两个集成表面是几乎每个驱动程序都需要的，哪两个只是加入特定子系统的驱动程序才需要的？（第1、2、3、4、5、6节）

能毫不犹豫地回答大多数这些问题的读者已经内化了本章，为接下来的内容做好了准备。对两个以上犹豫不决的读者应该在处理第25章的维护规则之前重温相关章节。

### 最后的话

集成是将工作模块转变为可用驱动程序的东西。本章中的模式不是可选的润色；它们是操作员可以采用的驱动程序与操作员必须与之斗争的驱动程序之间的区别。掌握它们一次，随后的每个驱动程序都更容易构建、更容易维护、更容易发布。

下一章将这里引入的规则推广为一组维护习惯：限速日志、一致的errno词汇、可调参数和版本化、生产级清理，以及将驱动程序的生命周期扩展到简单加载和卸载之外的`SYSINIT`/`SYSUNINIT`/`EVENTHANDLER`机制。词汇变化，但节奏相同：注册、接收流量、干净注销。有了第24章的基础，第25章将感觉像是自然的延伸而不是新世界。

在翻页之前最后一个想法。本章中的集成表面故意在数量上很少。有devfs、ioctl和sysctl。有对ifnet和CAM等子系统的可选注册。有将它们联系在一起的生命周期规则。总共五个概念。

在第 9 节中，我们从解释转向实践。实验室引导读者依次构建集成的阶段 1、阶段 2 和阶段 3，每个都有验证命令和预期输出。实验室之后是挑战练习（第 10 节）、故障排除目录（第 11 节），以及关闭章节的总结和桥梁。

## 动手实验室

本节的实验室引导读者从 freshly cloned 的工作树通过本章添加的每个集成表面。每个实验室足够小，可以一次完成，并配有一个验证命令来确认更改。按顺序运行实验室；后面的实验室建立在早期的实验室之上。

`examples/part-05/ch24-integration/` 下的配套文件包含三个分阶段的参考驱动程序（`stage1-devfs/`、`stage2-ioctl/`、`stage3-sysctl/`），与本章的里程碑匹配。实验室假设读者从自己的第 23 章末尾驱动程序（版本 `1.6-debug`）开始，或者将适当的阶段目录复制到工作位置，在那里进行更改，并在遇到困难时参考匹配的分阶段目录。

每个实验室使用真实的 FreeBSD 14.3 系统。虚拟机可以；不要在生产主机上运行这些实验室，因为如果驱动程序有错误，模块加载和卸载可能会挂起或恐慌系统。

### 实验室 1：构建并加载阶段 1 驱动程序

**目标**：将驱动程序从第 23 章基线（`1.6-debug`）带到本章阶段 1 里程碑（在 `/dev/myfirst0` 下正确构建的 cdev）。

**设置**：

从您自己的第 23 章末尾工作树（版本为 `1.6-debug` 的驱动程序）开始，或将第 23 章最后阶段的参考树复制到实验室目录：

```console
$ cp -r ~/myfirst-1.6-debug ~/myfirst-lab1
$ cd ~/myfirst-lab1
$ ls
Makefile  myfirst.c  myfirst.h  myfirst_debug.c  myfirst_debug.h
```

如果您想与已经迁移的章节阶段 1 起点进行比较（已应用 `make_dev_s`），请将 `examples/part-05/ch24-integration/stage1-devfs/` 作为参考解决方案而不是起始目录进行咨询。

**步骤 1**：打开 `myfirst.c` 并找到现有的 `make_dev` 调用。第 23 章代码使用较旧的单调用形式。用第 2 节的 `make_dev_args` 形式替换它：

```c
struct make_dev_args args;
int error;

make_dev_args_init(&args);
args.mda_devsw = &myfirst_cdevsw;
args.mda_uid = UID_ROOT;
args.mda_gid = GID_WHEEL;
args.mda_mode = 0660;
args.mda_si_drv1 = sc;
args.mda_unit = device_get_unit(dev);

error = make_dev_s(&args, &sc->sc_cdev,
    "myfirst%d", device_get_unit(dev));
if (error != 0) {
        mtx_destroy(&sc->sc_mtx);
        return (error);
}
```

**步骤 2**：将 `D_TRACKCLOSE` 添加到 cdevsw 标志（它应该已经有 `D_VERSION`）：

```c
static struct cdevsw myfirst_cdevsw = {
        .d_version = D_VERSION,
        .d_flags   = D_TRACKCLOSE,
        .d_name    = "myfirst",
        .d_open    = myfirst_open,
        .d_close   = myfirst_close,
        .d_read    = myfirst_read,
        .d_write   = myfirst_write,
};
```

**步骤 3**：确认 `myfirst_open` 和 `myfirst_close` 使用 `dev->si_drv1` 恢复 softc：

```c
static int
myfirst_open(struct cdev *dev, int oflags, int devtype, struct thread *td)
{
        struct myfirst_softc *sc = dev->si_drv1;
        ...
}
```

**步骤 4**：构建并加载：

```console
$ make
$ sudo kldload ./myfirst.ko
```

**验证**：

```console
$ ls -l /dev/myfirst0
crw-rw---- 1 root wheel 0x... <date> /dev/myfirst0
$ sudo cat /dev/myfirst0
Hello from myfirst
$ sudo kldstat | grep myfirst
N    1 0xffff... 1...    myfirst.ko
$ sudo dmesg | tail
... (来自 MYF_DBG_INIT 的调试消息)
```

如果 `ls` 显示错误的所有者、组或模式，请重新检查 `mda_uid`、`mda_gid` 和 `mda_mode` 值。如果 `cat` 返回空字符串，请检查 `myfirst_read` 是否正在从 `sc->sc_msg` 填充用户缓冲区。如果加载成功但设备未出现，请检查 cdevsw 是否从 `make_dev_args` 引用。

**清理**：

```console
$ sudo kldunload myfirst
$ ls -l /dev/myfirst0
ls: /dev/myfirst0: No such file or directory
```

成功的卸载会删除设备节点。如果卸载失败并显示 `Device busy`，请检查没有 shell 或程序打开了设备（`fstat | grep myfirst0`）。

### 实验室 2：添加 ioctl 接口

**目标**：通过添加第 3 节的四个 ioctl 命令将驱动程序扩展到阶段 2。

**设置**：

```console
$ cp -r examples/part-05/ch24-integration/stage1-devfs ~/myfirst-lab2
$ cd ~/myfirst-lab2
```

**步骤 1**：从第 3 节的模板创建 `myfirst_ioctl.h`。将其放置在与其他源文件相同的目录中。包含 `sys/ioccom.h` 和 `sys/types.h`。定义 `MYFIRST_MSG_MAX = 256` 和四个 ioctl 编号。不要包含任何仅限内核的头文件。

**步骤 2**：从第 3 节的模板创建 `myfirst_ioctl.c`。分发器是单个函数 `myfirst_ioctl`，具有标准的 `d_ioctl_t` 签名。

**步骤 3**：在 `Makefile` 中将 `myfirst_ioctl.c` 添加到 `SRCS`：

```make
SRCS=   myfirst.c myfirst_debug.c myfirst_ioctl.c
```

**步骤 4**：更新 cdevsw 以将 `.d_ioctl` 指向新分发器：

```c
.d_ioctl = myfirst_ioctl,
```

**步骤 5**：将 `sc_msg` 和 `sc_msglen` 添加到 softc 并在附加中初始化它们：

```c
strlcpy(sc->sc_msg, "Hello from myfirst", sizeof(sc->sc_msg));
sc->sc_msglen = strlen(sc->sc_msg);
```

**步骤 6**：构建用户空间配套程序。将 `myfirstctl.c` 放在同一目录中，并创建一个小的 `Makefile.user`：

```make
CC?= cc
CFLAGS+= -Wall -Werror -I.

myfirstctl: myfirstctl.c myfirst_ioctl.h
        ${CC} ${CFLAGS} -o myfirstctl myfirstctl.c
```

（请注意，缩进必须是制表符，而不是空格，以便 `make` 解析规则。）

构建内核模块和配套程序：

```console
$ make
$ make -f Makefile.user
$ sudo kldload ./myfirst.ko
```

**验证**：

```console
$ ./myfirstctl get-version
driver ioctl version: 1
$ ./myfirstctl get-message
Hello from myfirst
$ sudo ./myfirstctl set-message "drivers are fun"
$ ./myfirstctl get-message
drivers are fun
$ sudo ./myfirstctl reset
$ ./myfirstctl get-message

$
```

如果 `set-message` 返回 `Permission denied`，问题是设备模式为 `0660`，用户不在 `wheel` 中。要么使用 `sudo` 运行（如上面的验证命令），要么将 `mda_gid` 更改为用户所属的组并重新加载模块。

如果 `set-message` 返回 `Bad file descriptor`，问题是 `myfirstctl` 以只读方式打开了设备。检查程序是否为 `set-message` 和 `reset` 选择 `O_RDWR`。

如果任何 ioctl 返回 `Inappropriate ioctl for device`，问题是 `myfirst_ioctl.h` 中编码的长度与分发器对数据的视图不匹配。重新检查 `_IOR`/`_IOW` 宏和它们声明的结构的大小。

**清理**：

```console
$ sudo kldunload myfirst
```

### 实验室 3：添加 Sysctl 树

**目标**：通过添加第 4 节的 sysctl OID 并从 `/boot/loader.conf` 读取可调参数，将驱动程序扩展到阶段 3。

**设置**：

```console
$ cp -r examples/part-05/ch24-integration/stage2-ioctl ~/myfirst-lab3
$ cd ~/myfirst-lab3
```

**步骤 1**：从第 4 节的模板创建 `myfirst_sysctl.c`。函数 `myfirst_sysctl_attach(sc)` 构建整个树。

**步骤 2**：在 `Makefile` 中将 `myfirst_sysctl.c` 添加到 `SRCS`。

**步骤 3**：更新 `myfirst_attach` 以调用 `TUNABLE_INT_FETCH` 和 `myfirst_sysctl_attach`：

```c
TUNABLE_INT_FETCH("hw.myfirst.debug_mask_default", &sc->sc_debug);

/* ... after make_dev_s succeeds: */
myfirst_sysctl_attach(sc);
```

**步骤 4**：构建并加载：

```console
$ make
$ sudo kldload ./myfirst.ko
```

**验证**：

```console
$ sysctl -a dev.myfirst.0
dev.myfirst.0.debug.classes: INIT(0x1) OPEN(0x2) IO(0x4) IOCTL(0x8) ...
dev.myfirst.0.debug.mask: 0
dev.myfirst.0.message_len: 18
dev.myfirst.0.message: Hello from myfirst
dev.myfirst.0.total_writes: 0
dev.myfirst.0.total_reads: 0
dev.myfirst.0.open_count: 0
dev.myfirst.0.version: 1.7-integration
```

打开设备一次并重新检查计数器：

```console
$ cat /dev/myfirst0
Hello from myfirst
$ sysctl dev.myfirst.0.total_reads
dev.myfirst.0.total_reads: 1
```

测试加载器时可调参数。编辑 `/boot/loader.conf`（先备份）：

```console
$ sudo cp /boot/loader.conf /boot/loader.conf.backup
$ sudo sh -c 'echo hw.myfirst.debug_mask_default=\"0x06\" >> /boot/loader.conf'
```

请注意，这仅在下次重新启动时生效，并且仅当模块由加载器加载（而不是启动后通过 `kldload` 加载）时。对于无需重新启动的交互式测试，请在加载之前设置值：

```console
$ sudo kenv hw.myfirst.debug_mask_default=0x06
$ sudo kldload ./myfirst.ko
$ sysctl dev.myfirst.0.debug.mask
dev.myfirst.0.debug.mask: 6
```

如果值是 0 而不是 6，请检查 `TUNABLE_INT_FETCH` 调用是否使用与 `kenv` 命令相同的字符串。该调用必须在 `myfirst_sysctl_attach` 之前运行，以便在创建 OID 时值已就位。

**清理**：

```console
$ sudo kldunload myfirst
$ sudo cp /boot/loader.conf.backup /boot/loader.conf
```

### 实验室 4：通过注入失败来演练生命周期

**目标**：通过故意失败其中一个步骤，查看附加中的 `goto err` 链实际展开。

**设置**：

```console
$ cp -r examples/part-05/ch24-integration/stage3-sysctl ~/myfirst-lab4
$ cd ~/myfirst-lab4
```

**步骤 1**：打开 `myfirst.c` 并找到 `myfirst_attach`。在 `make_dev_s` 成功后立即插入故意失败：

```c
error = make_dev_s(&args, &sc->sc_cdev,
    "myfirst%d", device_get_unit(dev));
if (error != 0)
        goto fail_mtx;

/* DELIBERATE FAILURE for Lab 4 */
device_printf(dev, "Lab 4: injected failure after make_dev_s\n");
error = ENXIO;
goto fail_cdev;

myfirst_sysctl_attach(sc);
return (0);

fail_cdev:
        destroy_dev(sc->sc_cdev);
fail_mtx:
        mtx_destroy(&sc->sc_mtx);
        return (error);
```

**步骤 2**：构建并尝试加载：

```console
$ make
$ sudo kldload ./myfirst.ko
kldload: an error occurred while loading module myfirst. Please check dmesg(8) for more details.
$ sudo dmesg | tail
myfirst0: Lab 4: injected failure after make_dev_s
```

**验证**：

```console
$ ls /dev/myfirst0
ls: /dev/myfirst0: No such file or directory
$ kldstat | grep myfirst
$
```

cdev 已消失（`goto fail_cdev` 清理销毁了它），模块未加载，没有资源泄漏。如果失败后 cdev 仍然存在，则清理缺少 `destroy_dev` 调用。如果下次尝试模块加载时内核恐慌，则清理正在释放或销毁某些东西两次。

**奖励**：将失败注入更改为在 `make_dev_s` 之前发生。清理链现在应该跳过 `fail_cdev` 标签并只运行 `fail_mtx`。验证 cdev 从未创建并且互斥锁已销毁：

```console
$ sudo kldload ./myfirst.ko
$ sudo dmesg | tail
... no Lab 4 message because it now runs before make_dev_s ...
```

**清理**：

在继续之前删除故意失败块。

### 实验室 5：使用 DTrace 追踪集成表面

**目标**：使用第 23 章的 SDT 探针实时追踪 ioctl、open、close 和 read 流量。

**设置**：阶段 3 驱动程序已加载，如实验室 3 所示。

**步骤 1**：验证探针对 DTrace 可见：

```console
$ sudo dtrace -l -P myfirst
   ID   PROVIDER      MODULE    FUNCTION   NAME
... id  myfirst       kernel    -          open
... id  myfirst       kernel    -          close
... id  myfirst       kernel    -          io
... id  myfirst       kernel    -          ioctl
```

如果列表为空，则 SDT 探针未注册。检查 `myfirst_debug.c` 是否在 `SRCS` 中，以及是否从中调用了 `SDT_PROBE_DEFINE*`。

**步骤 2**：在一个终端中打开长期运行的追踪：

```console
$ sudo dtrace -n 'myfirst:::ioctl { printf("ioctl cmd=0x%x flags=0x%x", arg1, arg2); }'
dtrace: description 'myfirst:::ioctl' matched 1 probe
```

**步骤 3**：在另一个终端中操作驱动程序：

```console
$ ./myfirstctl get-version
$ ./myfirstctl get-message
$ sudo ./myfirstctl set-message "Lab 5"
$ sudo ./myfirstctl reset
```

DTrace 终端应该为每个 ioctl 显示一行，带有命令代码和文件标志。

**步骤 4**：将多个探针组合到一个脚本中：

```console
$ sudo dtrace -n '
    myfirst:::open  { printf("open  pid=%d", pid); }
    myfirst:::close { printf("close pid=%d", pid); }
    myfirst:::io    { printf("io    pid=%d write=%d resid=%d", pid, arg1, arg2); }
    myfirst:::ioctl { printf("ioctl pid=%d cmd=0x%x", pid, arg1); }
'
```

在另一个终端中：

```console
$ cat /dev/myfirst0
$ ./myfirstctl get-version
$ echo "hello" | sudo tee /dev/myfirst0
```

DTrace 输出现在显示完整的流量模式，每次操作周围的 open 和 close、内部的 read 或 write 以及任何 ioctl。这就是让 SDT 探针与 cdevsw 回调集成的价值：驱动程序暴露的每个集成表面也是 DTrace 的探针表面。

**清理**：

```console
^C
$ sudo kldunload myfirst
```

### 实验室 6：集成冒烟测试

**目标**：构建一个单一的 shell 脚本，一次演练每个集成表面，并产生绿色/红色摘要，读者可以将其粘贴到错误报告或发布准备清单中。

冒烟测试是一个小型、快速、端到端的检查，确认驱动程序还活着并且每个表面都有响应。它不能替代仔细的单元测试；它让读者在投入更多时间之前进行五秒钟的确认，确保没有明显损坏。真实驱动程序有冒烟测试；本章建议从第一天起就为每个新驱动程序添加一个。

**设置**：阶段 3 驱动程序已加载。

**步骤 1**：在工作目录中创建 `smoke.sh`：

```sh
#!/bin/sh
# smoke.sh - myfirst 驱动程序的端到端冒烟测试。

set -u
fail=0

check() {
        if eval "$1"; then
                printf "  PASS  %s\n" "$2"
        else
                printf "  FAIL  %s\n" "$2"
                fail=$((fail + 1))
        fi
}

echo "=== myfirst integration smoke test ==="

# 1. 模块已加载。
check "kldstat | grep -q myfirst" "module is loaded"

# 2. /dev 节点以正确的模式存在。
check "test -c /dev/myfirst0" "/dev/myfirst0 exists as a character device"
check "test \"\$(stat -f %Lp /dev/myfirst0)\" = \"660\"" "/dev/myfirst0 is mode 0660"

# 3. Sysctl 树存在。
check "sysctl -N dev.myfirst.0.version >/dev/null 2>&1" "version OID is present"
check "sysctl -N dev.myfirst.0.debug.mask >/dev/null 2>&1" "debug.mask OID is present"
check "sysctl -N dev.myfirst.0.open_count >/dev/null 2>&1" "open_count OID is present"

# 4. Ioctls 工作（需要已构建 myfirstctl）。
check "./myfirstctl get-version >/dev/null" "MYFIRSTIOC_GETVER returns success"
check "./myfirstctl get-message >/dev/null" "MYFIRSTIOC_GETMSG returns success"
check "sudo ./myfirstctl set-message smoke && [ \"\$(./myfirstctl get-message)\" = smoke ]" "MYFIRSTIOC_SETMSG round-trip works"
check "sudo ./myfirstctl reset && [ -z \"\$(./myfirstctl get-message)\" ]" "MYFIRSTIOC_RESET clears state"

# 5. Read/write 基本路径。
check "echo hello | sudo tee /dev/myfirst0 >/dev/null" "write to /dev/myfirst0 succeeds"
check "[ \"\$(cat /dev/myfirst0)\" = hello ]" "read returns the previously written message"

# 6. 计数器更新。
sudo ./myfirstctl reset >/dev/null
cat /dev/myfirst0 >/dev/null
check "[ \"\$(sysctl -n dev.myfirst.0.total_reads)\" = 1 ]" "total_reads incremented after one read"

# 7. SDT 探针已注册。
check "sudo dtrace -l -P myfirst | grep -q open" "myfirst:::open SDT probe is visible"

echo "=== summary ==="
if [ $fail -eq 0 ]; then
        echo "ALL PASS"
        exit 0
else
        printf "%d FAIL\n" "$fail"
        exit 1
fi
```

**步骤 2**：使其可执行并运行：

```console
$ chmod +x smoke.sh
$ ./smoke.sh
=== myfirst integration smoke test ===
  PASS  module is loaded
  PASS  /dev/myfirst0 exists as a character device
  PASS  /dev/myfirst0 is mode 0660
  PASS  version OID is present
  PASS  debug.mask OID is present
  PASS  open_count OID is present
  PASS  MYFIRSTIOC_GETVER returns success
  PASS  MYFIRSTIOC_GETMSG returns success
  PASS  MYFIRSTIOC_SETMSG round-trip works
  PASS  MYFIRSTIOC_RESET clears state
  PASS  write to /dev/myfirst0 succeeds
  PASS  read returns the previously written message
  PASS  total_reads incremented after one read
  PASS  myfirst:::open SDT probe is visible
=== summary ===
ALL PASS
```

如果任何检查失败，脚本的输出直接指向损坏的集成表面。失败的 `version OID is present` 意味着 sysctl 构造未运行；失败的 `MYFIRSTIOC_GETVER` 意味着 ioctl 分发器未正确连接；失败的 `total_reads incremented` 意味着 read 回调没有在互斥锁下增加计数器。

**验证**：在每次更改驱动程序后重新运行。提交前通过的冒烟测试是防止回归破坏基本流程的最廉价保险。

### 实验室 7：无需重启用户空间程序即可重新加载

**目标**：确认当用户空间程序在另一个终端持有打开的文件描述符时，驱动程序可以卸载并重新加载。

此测试揭示了本章"软分离"模式应该防止的生命周期错误。当用户持有设备打开时返回 `EBUSY` 的驱动程序正在正确保护自己；让分离成功然后在用户发出 ioctl 时恐慌的驱动程序是损坏的。

**设置**：阶段 3 驱动程序已加载。

**步骤 1**（终端 1）：使用长期运行的命令保持设备打开：

```console
$ sleep 3600 < /dev/myfirst0 &
$ jobs
[1]+ Running                 sleep 3600 < /dev/myfirst0 &
```

**步骤 2**（终端 2）：尝试卸载：

```console
$ sudo kldunload myfirst
kldunload: can't unload file: Device busy
```

这是预期行为。本章的 `myfirst_detach` 检查 `open_count > 0` 并返回 `EBUSY`，而不是在打开的文件描述符下拆卸 cdev。

**步骤 3**（终端 2）：验证设备在另一个 shell 中仍然可用：

```console
$ ./myfirstctl get-version
driver ioctl version: 1
$ sysctl dev.myfirst.0.open_count
dev.myfirst.0.open_count: 1
```

打开计数反映了保持的文件描述符。

**步骤 4**（终端 1）：释放文件描述符：

```console
$ kill %1
$ wait
```

**步骤 5**（终端 2）：卸载现在成功：

```console
$ sudo kldunload myfirst
$ sysctl dev.myfirst.0
sysctl: unknown oid 'dev.myfirst.0'
```

OID 已消失，因为 Newbus 在分离成功返回后拆卸了每设备 sysctl 上下文。

**验证**：每次卸载都应该成功而不恐慌。如果内核在步骤 5 期间恐慌，原因几乎总是 cdev 的回调在 `destroy_dev` 返回时仍在运行；检查 cdevsw 的 `d_close` 是否正确释放在 `d_open` 中获取的任何内容，并检查没有 callout 或 taskqueue 仍在调度。

奖励扩展是编写一个打开设备、立即调用 `MYFIRSTIOC_RESET`，然后循环 `MYFIRSTIOC_GETVER` 几秒钟的小程序。当循环运行时，尝试从另一个终端卸载。卸载仍应失败并显示 `EBUSY`；进行中的 ioctl 不应该损坏任何内容。

### 实验室总结

七个实验室引导读者完成了完整的集成表面、生命周期规范、冒烟测试和软分离契约。阶段 1 添加了 cdev；阶段 2 添加了 ioctl 接口；阶段 3 添加了 sysctl 树；生命周期的实验室（实验室 4）确认了展开；DTrace 实验室（实验室 5）确认了与第 23 章调试基础设施的集成；冒烟测试（实验室 6）给了读者一个可重用的验证脚本；重新加载实验室（实验室 7）确认了软分离契约。

通过所有七个实验室的驱动程序处于本章的里程碑版本 `1.7-integration`，并为下一章的主题做好准备。第 10 节的挑战练习给读者可选的后续工作，将驱动程序扩展到本章涵盖的内容之外。

## 挑战练习

以下挑战是可选的，旨在为希望将驱动程序推到章节里程碑之外的读者准备。每个挑战都有明确的目标、关于方法的几点提示，以及关于章节中包含相关材料的注释。没有一个挑战有唯一的正确答案；鼓励读者将其解决方案与审阅者或引用为参考的树内驱动程序进行比较。

### 挑战 1：添加可变长度 Ioctl

**目标**：扩展 ioctl 接口，使户空间程序可以传输大于 `MYFIRSTIOC_SETMSG` 使用的固定 256 字节的缓冲区。

本章的模式是固定大小的：`MYFIRSTIOC_SETMSG` 声明 `_IOW('M', 3, char[256])`，内核处理整个 copyin。对于更大的缓冲区（比如高达 1 MB），需要嵌入式指针模式：

```c
struct myfirst_blob {
        size_t  len;
        char   *buf;    /* user-space pointer */
};
#define MYFIRSTIOC_SETBLOB _IOW('M', 5, struct myfirst_blob)
```

分发器必须调用 `copyin` 来传输指针引用的字节；结构本身像以前一样通过自动 copyin 传入。提示：强制执行最大长度（1 MB 是合理的）。使用 `malloc(M_TEMP, len, M_WAITOK)` 分配临时内核缓冲区；不要在 softc 互斥锁内分配它。在返回之前释放它。参考：第 3 节，"Common Pitfalls With ioctl"，第二个陷阱。

奖励扩展是添加 `MYFIRSTIOC_GETBLOB`，以相同的可变长度格式复制当前消息；注意用户提供的缓冲区比消息短的情况，决定是截断、返回 `ENOMEM` 还是回写所需长度。真实驱动程序（`SIOCGIFCAP`、`KIOCGRPC`）使用后一种模式。

### 挑战 2：添加每次打开的计数器

**目标**：维护每次文件描述符计数器（`/dev/myfirst0` 的每次打开一个数字），而不是我们现在只有每次实例的计数器。

本章的 `sc_open_count` 跨所有打开聚合。每次打开的计数器会让程序知道从自己的描述符读取了多少。提示：使用 `cdevsw->d_priv` 附加每次 fd 结构（包含计数器的 `struct myfirst_fdpriv`）。在 `myfirst_open` 中分配结构，在 `myfirst_close` 中释放它。框架在每个文件的 `f_data` 字段中为每个 `cdev_priv` 提供唯一的指针；read 和 write 回调然后可以通过 `devfs_get_cdevpriv()` 查找每次 fd 结构。

参考：`/usr/src/sys/kern/kern_conf.c` 中的 `devfs_set_cdevpriv` 和 `devfs_get_cdevpriv`。该模式也被 `/usr/src/sys/dev/random/random_harvestq.c` 使用。

奖励扩展是添加一个报告每次 fd 计数器总和的 sysctl OID，并验证它始终等于现有的聚合计数器。差异表示某处缺少增量。

### 挑战 3：使用 `dev_ref` 实现软分离

**目标**：用更干净的"排空到上次关闭，然后分离"模式替换本章的"如果打开则拒绝分离"模式。

本章的分离在用户持有设备打开时返回 `EBUSY`。更优雅的模式使用 `dev_ref`/`dev_rel` 计算未完成的引用，并在完成分离之前等待计数达到零。提示：在 `myfirst_open` 中获取 `dev_ref` 并在 `myfirst_close` 中释放它。在分离中，设置一个"即将离开"标志，然后调用 `destroy_dev_drain`（或编写一个调用 `tsleep` 的小循环，同时 `dev_refs > 0`）在调用 `destroy_dev` 之前。一旦计数达到零并且 cdev 被销毁，正常完成分离。

参考：`/usr/src/sys/kern/kern_conf.c` 中的 `dev_ref` 机制；`/usr/src/sys/fs/cuse` 是使用排空模式进行睡眠分离的真实驱动程序。

奖励扩展是添加一个报告当前引用计数的 sysctl OID，并验证它与打开计数匹配。

### 挑战 4：替换静态魔术字母

**目标**：将 `myfirst_ioctl.h` 中硬编码的 `'M'` 魔术字母替换为不会与树中其他任何内容冲突的名称。

本章任意选择 `'M'` 并警告冲突风险。更防御性的驱动程序使用更长的魔术标识符并从中构建 ioctl 编号。提示：定义 `MYFIRST_IOC_GROUP = 0x83`（或任何未被另一个驱动程序使用的字节）。`_IOC` 宏然后接受该常量而不是字符字面量。用注释在头文件中记录选择，解释它是如何选择的。

奖励是在 `/usr/src/sys` 中 grep `_IO[RW]?\\(.\\?'M'` 并生成 `'M'` 的每个现有用途的列表。（有几个，包括 `MIDI` ioctl 和其他；调查本身具有教育意义。）

### 挑战 5：添加用于关闭的 `EVENTHANDLER`

**目标**：使驱动程序在系统关闭时优雅地运行。

本章的驱动程序没有关闭处理程序；如果系统在加载 `myfirst` 的情况下关闭，框架最终调用分离。更完善的驱动程序注册 `shutdown_pre_sync` 的 `EVENTHANDLER`，以便它可以在文件系统变为只读之前刷新任何进行中的状态。

提示：在附加中使用 `EVENTHANDLER_REGISTER(shutdown_pre_sync, ...)` 注册处理程序。处理程序在相应的关闭阶段被调用。在分离中使用 `EVENTHANDLER_DEREGISTER` 注销。在处理程序内部，将驱动程序设置为静止状态（清除消息、清零计数器）；此时文件系统仍然可写，因此任何通过 `printf` 的用户反馈将在下次引导后落在 `/var/log/messages` 中。

参考：第 7 节，"EVENTHANDLER for System Events" 和 `/usr/src/sys/sys/eventhandler.h` 中的完整命名事件列表。

### 挑战 6：第二个每驱动程序 Sysctl 子树

**目标**：在 `dev.myfirst.0` 下添加第二个子树，暴露每线程统计信息。

本章的树有一个 `debug.` 子树。完整的驱动程序可能还有一个 `stats.` 子树（用于按文件描述符细分的读/写统计信息）或一个 `errors.` 子树（用于错误计数器）。提示：使用 `SYSCTL_ADD_NODE` 创建新节点，然后使用 `SYSCTL_ADD_*` 在新节点的 `SYSCTL_CHILDREN` 下填充它。该模式与 `debug.` 子树相同，只是根于不同的名称。

参考：第 4 节，"`myfirst` Sysctl Tree"，以现有的 `debug.` 子树为模型；`/usr/src/sys/dev/iicbus` 中有几个使用多子树 sysctl 布局的驱动程序。

### 挑战 7：跨模块依赖

**目标**：构建一个依赖 `myfirst` 并使用其内核内 API 的小型第二模块（`myfirst_logger`）。

本章的 `myfirst` 驱动程序不导出任何符号供内核内用户使用。添加调用 `myfirst` 的第二个模块练习 `MODULE_DEPEND` 机制。提示：在 `myfirst.h` 中声明承载符号的函数（可能是 `int myfirst_get_message(int unit, char *buf, size_t len)`）并在 `myfirst.c` 中实现它。使用 `MODULE_DEPEND(myfirst_logger, myfirst, 1, 1, 1)` 构建第二个模块，以便当 `myfirst_logger` 加载时内核自动加载 `myfirst`。

奖励是将 `myfirst` 的模块版本提升为 2，以不向后兼容的方式更改内核内 API，并观察第二个模块在重新构建针对新版本之前加载失败。参考：第 8 节，"Version Strings, Version Numbers, and the API Version"。

### 结束挑战

七个挑战从简短（挑战 4 主要是重命名和注释）到实质性的（挑战 3 需要阅读并理解 `dev_ref`）。完成所有七个的读者将对本章仅概述的每个集成角落有实际熟悉。完成任何单个挑战的读者将比仅靠本章获得更深的集成规范感。

## 故障排除

本章的集成表面位于内核与系统其余部分之间的接缝处。接缝处的问题通常看起来像驱动程序错误，但实际上是缺少标志、头文件中的拼写错误或对谁拥有什么的误解的症状。下面的目录收集了最常见的症状、可能的原因和每个的修复方法。

### `kldload` 后 `/dev/myfirst0` 未出现

首先检查模块是否成功加载：

```console
$ kldstat | grep myfirst
```

如果模块未列出，则加载失败；请参阅 `dmesg` 获取更具体的消息。最常见的原因是未解析的符号（通常因为新源文件不在 `SRCS` 中）。

如果模块已列出但设备节点缺失，则 `myfirst_attach` 内部的 `make_dev_s` 调用可能失败了。在调用旁边添加 `device_printf(dev, "make_dev_s returned %d\n", error)` 并重试。非零返回的最常见原因是另一个驱动程序已经创建了 `/dev/myfirst0`（内核不会静默覆盖现有节点），或者 `make_dev_s` 在不可睡眠的上下文中使用 `MAKEDEV_NOWAIT` 调用。

更微妙的原因是 `cdevsw->d_version` 不等于 `D_VERSION`。内核检查此值并拒绝注册版本不匹配的 cdevsw。修复方法是 `static struct cdevsw myfirst_cdevsw = { .d_version = D_VERSION, ... };` 准确无误。

### `cat /dev/myfirst0` 返回 "Permission denied"

设备存在但用户无法打开它。本章的默认模式是 `0660`，默认组是 `wheel`。要么使用 `sudo` 运行，将 `mda_gid` 更改为用户的组，或将 `mda_mode` 更改为 `0666`（后者对于教学模块可以，但对于生产驱动程序是不好的选择，因为任何本地用户都可以打开设备）。

### `ioctl` 返回 "Inappropriate ioctl for device"

内核返回了 `ENOTTY`，这意味着它无法将请求代码匹配到任何 cdevsw。两个常见原因是：

* 驱动程序的分发器为命令返回了 `ENOIOCTL`。内核将 `ENOIOCTL` 转换为用户空间的 `ENOTTY`。修复方法是在分发器的 switch 语句中为命令添加一个 case。

* 请求代码中编码的长度与程序使用的实际缓冲区大小不匹配。这在头文件重构后发生，其中 `_IOR` 行被编辑但用户空间程序未针对新头重新编译。修复方法是根据当前头重新编译程序并针对相同源重新构建模块。

### `ioctl` 返回 "Bad file descriptor"

分发器返回了 `EBADF`，这是本章对于"文件未以正确的标志打开以执行此命令"的模式。修复方法是为任何改变状态的命令使用 `O_RDWR` 而不是 `O_RDONLY` 打开设备。`myfirstctl` 配套程序已经这样做了；自定义程序可能没有。

### `sysctl dev.myfirst.0` 显示树但读取返回 "operation not supported"

这通常意味着 sysctl OID 使用陈旧或无效的处理程序指针添加。如果读取立即返回 `EOPNOTSUPP`（95），原因几乎总是 OID 使用 `CTLTYPE_OPAQUE` 注册，处理程序没有调用 `SYSCTL_OUT`。修复方法是使用类型化的 `SYSCTL_ADD_*` 辅助函数之一（`SYSCTL_ADD_UINT`、`SYSCTL_ADD_STRING`、带有正确格式字符串的 `SYSCTL_ADD_PROC`），以便框架知道在读取时要做什么。

### `sysctl -w dev.myfirst.0.foo=value` 失败并显示 "permission denied"

OID 可能使用 `CTLFLAG_RD`（只读）创建，而原本意图是可写变体 `CTLFLAG_RW`。重新检查 `SYSCTL_ADD_*` 调用中的标志字并重新构建。

如果标志正确但失败持续存在，用户可能未以 root 身份运行。Sysctl 写入默认需要 `PRIV_SYSCTL` 权限；使用 `sudo` 进行写入。

### `sysctl` 挂起或导致死锁

OID 处理程序在另一个线程持有 giant 锁并调用到驱动程序时同时获取 giant 锁（因为缺少 `CTLFLAG_MPSAFE`）。修复方法是在每个 OID 的标志字中添加 `CTLFLAG_MPSAFE`。现代内核假设到处都是 MPSAFE；缺少该标志是代码审查问题。

更微妙的原因是处理程序在另一个线程持有 softc 互斥锁并从 sysctl 读取时获取 softc 互斥锁。审计处理程序：它应该在互斥锁下计算值但在互斥锁外调用 `sysctl_handle_*`。本章的 `myfirst_sysctl_message_len` 遵循此模式。

### `kldunload myfirst` 失败并显示 "Device busy"

分离拒绝是因为某个用户持有设备打开。使用 `fstat | grep myfirst0` 找到他们，然后要求他们关闭它或杀死进程。在他们释放设备后，卸载将成功。

如果 `fstat` 什么也没显示但卸载仍然失败，原因很可能是泄漏的 `dev_ref`。重新检查驱动程序中每个获取 `dev_ref` 的代码路径是否也调用 `dev_rel`；特别是，`myfirst_open` 内部在失败之前获取引用的任何错误路径必须释放之前获取的任何引用。

### `kldunload myfirst` 导致内核恐慌

驱动程序的分离正在销毁或释放内核仍在使用的东西。两个最常见的原因是：

* 分离在销毁 cdev 之前释放了 softc。cdev 的回调可能仍在运行；它们解引用 `si_drv1`，获得垃圾，并恐慌。修复方法是严格的顺序：首先是 `destroy_dev`（它排空进行中的回调），然后是 mutex_destroy，然后返回；框架释放 softc。

* 分离忘记注销事件处理程序。下次事件在卸载后触发时会跳入已释放的内存。修复方法是为附加中完成的每个 `EVENTHANDLER_REGISTER` 调用 `EVENTHANDLER_DEREGISTER`。

`dmesg` 中的 `Lock order reversal` 和 `WITNESS` 消息对这两种情况都是有用的诊断。带有 `page fault while in kernel mode` 和损坏的 `%rip` 值的恐慌是第二种模式；带有 `lock order reversal` 和穿过两个子系统的堆栈跟踪的恐慌是第一种。

### DTrace 探针不可见

`dtrace -l -P myfirst` 返回空，即使模块已加载。原因几乎总是 SDT 探针在头文件中声明但未在任何地方定义。探针需要 `SDT_PROBE_DECLARE`（在头文件中，消费者看到它们）和 `SDT_PROBE_DEFINE*`（在正好一个源文件中，拥有探针存储）。本章的模式将定义放在 `myfirst_debug.c` 中。如果该文件不在 `SRCS` 中，探针将不会被定义，DTrace 什么也看不到。

更微妙的原因是 SDT 探针在头文件中重命名但匹配的 `SDT_PROBE_DEFINE*` 未更新。构建仍然成功，因为两个声明引用不同的符号，但 DTrace 只看到定义的名称。审计头文件和源文件以获取相同的探针名称。

### sysctl 树在卸载后存活但下次 Sysctl 挂起

当驱动程序使用自己的 sysctl 上下文（而不是每设备上下文）并忘记在分离中调用 `sysctl_ctx_free` 时会发生这种情况。OID 引用现已释放的 softc 中的字段；下次 `sysctl` 遍历解引用已释放的内存，内核要么恐慌要么返回垃圾。修复方法是切换到 `device_get_sysctl_ctx`，框架会自动清理它。

### 通用诊断检查清单

当出现问题且原因不明显时，在使用 `kgdb` 之前遍历这个简短列表：

1. `kldstat | grep <driver>`：模块实际加载了吗？
2. `dmesg | tail`：是否有任何内核消息提到驱动程序？
3. `ls -l /dev/<driver>0`：设备节点是否以预期模式存在？
4. `sysctl dev.<driver>.0.%driver`：Newbus 是否知道该设备？
5. `fstat | grep <driver>0`：是否有人持有设备打开？
6. `dtrace -l -P <driver>`：SDT 探针是否已注册？
7. 重读附加函数并检查每一步在分离中是否有匹配的清理。

前六个命令耗时十秒，排除了大部分常见问题。第七个是慢的，但几乎总是前六个没有发现的任何错误的最终答案。

### 常见问题解答

以下问题在集成工作中经常出现，本章以简短的 FAQ 结束。每个答案故意紧凑；章节的相关部分有完整讨论。

**问 1. 当 ioctl 和 sysctl 似乎重叠时为什么要使用两者？**

它们回答不同的问题。Ioctl 是用于已经打开设备并希望发出命令的程序（请求状态、推送新状态、触发操作）。Sysctl 是用于 shell 提示符下的操作员或希望在不打开任何东西的情况下检查或调整状态的脚本。相同的值可以通过两个接口公开，许多生产驱动程序正是这样做的：程序的 `MYFIRSTIOC_GETMSG` 和人类的 `dev.myfirst.0.message`。每个用户选择适合其上下文的通道。

**问 2. 我应该何时使用 mmap 而不是 read/write/ioctl？**

当数据很大、随机访问并且自然位于内存地址时使用 `mmap`（帧缓冲区、DMA 描述符环、内存映射寄存器空间）。当数据是顺序的、面向字节的且每次调用较小时使用 `read`/`write`。使用 `ioctl` 进行控制命令。这三者不是对立的；许多驱动程序暴露所有三个（如控制台的 `vt(4)`）。

**问 3. 为什么本章使用 `make_dev_s` 而不是 `make_dev`？**

`make_dev_s` 是现代首选形式。它返回显式错误而不是在重复名称上恐慌；它接受参数结构以便可以在不变动的情况下添加新选项；并且它是大多数当前驱动程序使用的。较旧的 `make_dev` 仍然工作，但不鼓励用于新代码。

**问 4. 我需要声明 `D_TRACKCLOSE` 吗？**

如果驱动程序的 `d_close` 应该只在文件描述符的最后一次关闭时调用（"close"的自然含义），则需要它。没有它，内核为每个重复的描述符的每次关闭调用 `d_close`，这会让大多数驱动程序感到意外。在任何新 cdevsw 中设置它，除非有特定原因不这样做。

**问 5. 我应该何时提升 `MODULE_VERSION`？**

当驱动程序的内核内 API 以不兼容方式更改时。添加新的导出符号可以；重命名或删除它们是提升。更改公开可见结构的布局是提升。提升模块版本会强制依赖者（`MODULE_DEPEND` 消费者）重新构建。

**问 6. 我应该何时提升公共头文件中的 API 版本常量？**

当用户可见接口以不兼容方式更改时。添加新 ioctl 可以；更改现有 ioctl 参数结构的布局是提升。重新编号现有 ioctl 是提升。提升 API 版本让用户空间程序在发出调用之前检测不兼容性。

**问 7. 我应该在 `myfirst_detach` 中分离我的 OID 吗？**

不，如果您使用了 `device_get_sysctl_ctx`（每设备上下文）。框架在成功分离返回后自动清理每设备上下文。只有在您使用 `sysctl_ctx_init` 创建自己的上下文时才需要显式清理。

**问 8. 为什么我的分离以 "invalid memory access" 恐慌？**

几乎总是因为 cdev 的回调在驱动程序释放它们引用的内容时仍在运行。修复方法是首先调用 `destroy_dev(sc->sc_cdev)`；`destroy_dev` 阻塞直到每个进行中的回调返回。在它返回后，cdev 消失，没有新的回调可以到达。只有这样才能安全释放 softc、释放锁等。严格的顺序是不可协商的。

**问 9. `dev_ref` / `dev_rel` 和 `D_TRACKCLOSE` 之间有什么区别？**

`D_TRACKCLOSE` 是控制内核何时调用 `d_close` 的 cdevsw 标志：有它时只在最后一次关闭；没有时在每次关闭时。`dev_ref`/`dev_rel` 是让驱动程序延迟分离直到未完成的引用被释放的引用计数机制。它们无关且互补。本章在阶段 1 中使用 `D_TRACKCLOSE`；挑战 3 演示 `dev_ref`/`dev_rel`。

**问 10. 为什么我的 sysctl 写入返回 EPERM 即使我是 root？**

三个可能的原因。(a) OID 仅使用 `CTLFLAG_RD` 创建；添加 `CTLFLAG_RW`。(b) OID 有 `CTLFLAG_SECURE` 且系统处于 `securelevel > 0`；降低安全级别或移除标志。(c) 用户实际上不是 root 但在没有 `allow.sysvipc` 或类似的 jail 中；jail 内的 root 对任意 OID 没有 `PRIV_SYSCTL`。

**问 11. 我的 sysctl 处理程序不应该获取 giant 锁时却获取了。我忘记了什么？**

标志字中的 `CTLFLAG_MPSAFE`。没有它，内核在每次调用处理程序时都获取 giant 锁。到处添加它；现代内核假设到处都是 MPSAFE。

**问 12. 我应该将我的 ioctl 组字母命名为大写还是小写？**

新驱动程序用大写。小写字母被基础子系统大量使用（`'d'` 用于磁盘、`'i'` 用于 `if_ioctl`、`'t'` 用于终端），碰撞风险是真实的。大写字母大多空闲，新驱动程序应该选择其中之一。

**问 13. 我的 ioctl 返回 `Inappropriate ioctl for device` 但我不明白为什么。**

内核返回 `ENOTTY` 是因为 (a) 分发器为命令返回了 `ENOIOCTL`（为其添加 case），或 (b) 请求代码中编码的长度与用户传递的缓冲区不匹配（针对相同头文件重新编译两侧）。

**问 14. 我应该在内核中使用 `strncpy` 还是 `strlcpy`？**

`strlcpy`。它保证 NUL 终止并永远不会溢出目标。`strncpy` 既不保证也不会溢出，是微妙错误的常见来源。FreeBSD `style(9)` 手册页建议所有新代码使用 `strlcpy`。

**问 15. 我的模块加载但 `dmesg` 没有显示来自驱动程序的任何消息。怎么了？**

驱动程序的调试掩码为零。本章的 `DPRINTF` 宏只在掩码位设置时打印。要么在加载前设置掩码（`kenv hw.myfirst.debug_mask_default=0xff`），要么在加载后设置（`sysctl dev.myfirst.0.debug.mask=0xff`）。

**问 16. 为什么本章如此频繁地提到 DTrace？**

因为它是 FreeBSD 内核中最富有成效的调试工具，并且因为第 23 章调试基础设施设计为与它集成。SDT 探针让操作员在不重新构建驱动程序的情况下运行时点击每个集成表面。暴露命名良好的 SDT 探针的驱动程序比不暴露的驱动程序更容易调试。

**问 17. 我可以将此驱动程序用作真实硬件驱动程序的模板吗？**

集成表面（cdev、ioctl、sysctl）直接转化。硬件特定的部分（资源分配、中断处理、DMA 设置）在第四部分的第 18 到 22 章中介绍。真实的 PCI 驱动程序通常将第四部分的结构模式与本章的集成模式结合，形成可发布的驱动程序。

**问 18. 如何在不重新构建驱动程序的情况下授予非 root 用户访问 `/dev/myfirst0` 的权限？**

使用 `devfs.rules(5)`。在 `/etc/devfs.rules` 下添加一个匹配设备名称并在运行时设置所有者、组或模式的规则文件。例如，让 `operator` 组读写 `/dev/myfirst*`：

```text
[myfirst_rules=10]
add path 'myfirst*' mode 0660 group operator
```

在 `/etc/rc.conf` 中使用 `devfs_system_ruleset="myfirst_rules"` 启用规则集，然后 `service devfs restart`。驱动程序的 `mda_uid`、`mda_gid` 和 `mda_mode` 仍然在创建时设置默认值；`devfs.rules` 让管理员在不触及源代码的情况下覆盖它们。

**问 19. 我的 `SRCS` 列表不断增长。这是问题吗？**

本身不是。内核模块 `Makefile` 中的 `SRCS` 行列出编译到模块中的每个源文件；随着新职责获得自己的文件而增长列表是正常和预期的。本章的阶段 3 驱动程序已经有四个源文件（`myfirst.c`、`myfirst_debug.c`、`myfirst_ioctl.c`、`myfirst_sysctl.c`），第 25 章将添加更多。警告信号不是条目数量而是缺乏结构：如果 `SRCS` 包含合并在一起的不相关文件且没有命名方案，驱动程序已经超出了其布局，值得进行小型重构。第 25 章将该重构视为一流习惯。

**问 20. 我接下来应该做什么？**

阅读第 25 章（高级主题和实际技巧），将这个集成的驱动程序变成一个 *可维护的* 驱动程序，如果您想要动手实践，请完成章节的挑战，并查看引用的树内驱动程序之一作为完整示例。`null(4)` 驱动程序是最温和的入口；`if_em` 以太网驱动程序是最完整的；`ahci(4)` 存储驱动程序显示 CAM 模式。选择最接近您想要构建的驱动程序并从头到尾阅读。

## 总结

本章将 `myfirst` 从工作的但独立的模块带入完全集成的 FreeBSD 驱动程序。故事线是刻意的：每一节添加一个具体的集成表面，并以更有用、更可发现的驱动程序结束。按顺序演练第 9 节实验室的读者现在在磁盘上有一个驱动程序，它暴露一个正确构建的 `/dev` 下的 cdev，四个设计良好的 ioctl，一个自描述的 `dev.myfirst.0` 下的 sysctl 树，一个通过 `/boot/loader.conf` 的启动时可调参数，以及一个干净的生命周期，可以在加载/卸载循环中存活而不泄漏资源。

沿途的技术里程碑是：

* 阶段 1（第 2 节）用现代 `make_dev_args` 形式替换了较旧的 `make_dev` 调用，填充了 `D_TRACKCLOSE`，连接了 `si_drv1` 用于每次 cdev 状态，并演练了从创建到排空到销毁的 cdev 生命周期。驱动程序的 `/dev` 存在变得一流。

* 阶段 2（第 3 节）添加了 `MYFIRSTIOC_GETVER`、`MYFIRSTIOC_GETMSG`、`MYFIRSTIOC_SETMSG` 和 `MYFIRSTIOC_RESET` ioctl 以及匹配的 `myfirst_ioctl.h` 公共头文件。分发器重用了第 23 章的调试基础设施（`MYF_DBG_IOCTL` 和 `myfirst:::ioctl` SDT 探针）。配套的 `myfirstctl` 用户空间程序演示了一个小型命令行工具如何演练每个 ioctl，而无需手动解码请求代码。

* 阶段 3（第 4 节）添加了 `dev.myfirst.0.*` sysctl 树，包括一个让操作员在运行时检查和修改调试掩码的 `debug.` 子树，一个报告集成版本的 `version` OID，读和写活动的计数器，以及当前消息的字符串 OID。启动时可调参数 `hw.myfirst.debug_mask_default` 让操作员在附加之前预加载调试掩码。

* 第 5 和 6 节概述了应用于网络堆栈（`if_alloc`、`if_attach`、`bpfattach`）和 CAM 存储堆栈（`cam_sim_alloc`、`xpt_bus_register`、`xpt_action`）的相同注册风格集成。不构建网络或存储驱动程序的读者仍然获得了一个有用的模式：FreeBSD 中的每个框架注册都使用相同的分配-命名-填充-附加-流量-分离-释放形状。

* 第 7 节编纂了将所有内容联系在一起的生命周期规范：每个成功的注册必须在反向顺序中与注销配对，失败的附加必须在返回失败代码之前展开每个先前步骤。

* 第 8 节将本章框架为更长故事线中的一步：`myfirst` 从第二部分的单文件演示开始，在第三部分和第四部分成长为多文件驱动程序，在第 23 章获得调试和跟踪，并在此获得集成表面。发布版本、模块版本和 API 版本各自跟踪该演进的不同方面；在正确的时机提升每一个是长期运行驱动程序的版本规范。

本章的实验室（第 9 节）引导读者完成每个里程碑，挑战（第 10 节）为有动力的读者提供后续工作，故障排除目录（第 11 节）收集了最常见的症状和修复方法以供快速参考。

结果是驱动程序里程碑（`1.7-integration`），读者可以带入下一章而没有未完成的集成工作等着咬他们。本章的模式（cdev 构建、ioctl 设计、sysctl 树、生命周期规范）也是第五部分其余部分和第六部分和第七部分的大部分将假设读者已经知道的模式。

## 通往第 25 章的桥梁

第 25 章（高级主题和实际技巧）通过将本章集成的驱动程序变成 *可维护的* 驱动程序来关闭第五部分。第 24 章添加了让驱动程序与系统其余部分对话的接口，第 25 章教授使这些接口在驱动程序吸收下一年的错误修复、可移植性更改和功能请求时保持稳定和可读的工程习惯。驱动程序从 `1.7-integration` 成长到 `1.8-maintenance`；可见的添加是适度的，但它们背后的规范是将从一个开发周期存活的驱动程序与存续十年的驱动程序区分开来的东西。

从第 24 章到第 25 章的桥梁有四个具体部分。

首先，第 25 章引入的速率限制日志直接建立在第 23 章的 `DPRINTF` 宏和本章添加的集成表面之上。一个围绕 `ppsratecheck(9)` 构建的新 `DLOG_RL` 宏让驱动程序保留它已经使用的相同调试类，但在事件风暴期间不会淹没 `dmesg`。规范很小：选择一个每秒限制，将其折叠到现有的调试调用点，并审计少数几个无限制的 `device_printf` 可能在循环中运行的地方。

其次，本章构建的 ioctl 和 sysctl 路径将在第 25 章中审计一致的 errno 词汇。本章区分 `EINVAL` 和 `ENXIO`、`ENOIOCTL` 和 `ENOTTY`、`EBUSY` 和 `EAGAIN`、`EPERM` 和 `EACCES`，以便每个集成表面在每个失败路径上返回正确的代码。读者演练第 3 节编写的分发器和第 4 节编写的 sysctl 处理器，并在返回错误的地方进行调整。

第三，第 4 节引入的启动时可调参数 `hw.myfirst.debug_mask_default` 将在第 25 章中通过 `TUNABLE_INT_FETCH`、`TUNABLE_LONG_FETCH`、`TUNABLE_BOOL_FETCH` 和 `TUNABLE_STR_FETCH` 概括为一个虽小但规范的可调参数词汇，与 `CTLFLAG_TUN` 下的可写 sysctl 合作。本章确定的相同的 `MYFIRST_VERSION`、`MODULE_VERSION` 和 `MYFIRST_IOCTL_VERSION` 三元组将通过 `MYFIRSTIOC_GETCAPS` ioctl 扩展，以便用户空间工具可以在运行时检测功能而无需试错。

第四，第 7 节引入的 `goto err` 链将从实验室练习提升为驱动程序的生产清理模式，章节的重构将把 Newbus 附加逻辑和 cdev 回调移到单独的文件（`myfirst_bus.c` 和 `myfirst_cdev.c`），旁边是用于新日志宏的 `myfirst_log.c`。第 25 章还引入 `SYSINIT(9)` 和 `SYSUNINIT(9)` 用于驱动程序范围的初始化和通过 `EVENTHANDLER(9)` 的 `shutdown_pre_sync` 事件处理器，为本章已经教授的添加了两个更多注册风格的表面。

带着集成词汇已经就位的信心继续阅读。第 25 章采用此驱动程序并使其为长期做准备；第六部分然后开始依赖第五部分建立的每个习惯的传输特定章节。

## 参考卡和词汇表

章节的其余页面是紧凑参考。它们设计为第一次直读，然后在读者需要查找内容时跳转。顺序是：重要宏、结构和标志的参考卡；集成词汇的词汇表；以及随章节附带的配套文件的简短目录。

### 快速参考：cdev 构建

| 函数 | 何时使用 |
|----------|-------------|
| `make_dev_args_init(args)` | 总是在 `make_dev_s` 之前；安全地清零 args 结构。 |
| `make_dev_s(args, &cdev, fmt, ...)` | 现代首选形式。返回 0 或 errno。 |
| `make_dev(devsw, unit, uid, gid, mode, fmt, ...)` | 较旧的单调用形式。不鼓励用于新代码。 |
| `make_dev_credf(flags, ...)` | 当您需要 `MAKEDEV_*` 标志位时。 |
| `destroy_dev(cdev)` | 总是在分离中；排空进行中的回调。 |
| `destroy_dev_drain(cdev)` | 当分离必须等待未完成的引用时。 |

### 快速参考：cdevsw 标志

| 标志 | 含义 |
|------|---------|
| `D_VERSION` | 必需；标识 cdevsw 布局版本。 |
| `D_TRACKCLOSE` | 仅在最后一次关闭时调用 `d_close`。推荐。 |
| `D_NEEDGIANT` | 在每次回调周围获取 giant 锁。不鼓励。 |
| `D_DISK` | Cdev 表示磁盘；使用 bio 而不是 uio 进行 I/O。 |
| `D_TTY` | Cdev 是终端；影响线路规程路由。 |
| `D_MMAP_ANON` | Cdev 支持匿名 mmap。 |
| `D_MEM` | Cdev 是 `/dev/mem` 类似；原始内存访问。 |

### 快速参考：`make_dev` 标志（`MAKEDEV_*`）

| 标志 | 含义 |
|------|---------|
| `MAKEDEV_REF` | 获取额外引用；调用者必须稍后 `dev_rel`。 |
| `MAKEDEV_NOWAIT` | 不要睡眠；如果没有内存则返回 `ENOMEM`。 |
| `MAKEDEV_WAITOK` | 允许睡眠（大多数调用者的默认值）。 |
| `MAKEDEV_ETERNAL` | Cdev 永远不会消失；某些优化适用。 |
| `MAKEDEV_ETERNAL_KLD` | 与 ETERNAL 类似，但仅在 kld 的生命周期内。 |
| `MAKEDEV_CHECKNAME` | 验证名称；如果太长则 `ENAMETOOLONG`。 |

### 快速参考：ioctl 编码宏

全部在 `/usr/src/sys/sys/ioccom.h` 中。

| 宏 | 方向 | 参数 |
|-------|-----------|----------|
| `_IO(g, n)` | none | none |
| `_IOR(g, n, t)` | out | 类型 `t`，大小 `sizeof(t)` |
| `_IOW(g, n, t)` | in | 类型 `t`，大小 `sizeof(t)` |
| `_IOWR(g, n, t)` | in 和 out | 类型 `t`，大小 `sizeof(t)` |
| `_IOWINT(g, n)` | none，但传递 int 的值 | int |

参数含义：

* `g`：组字母，约定为 `myfirst` 的 `'M'` 和类似。
* `n`：命令编号，在组内单调递增。
* `t`：参数类型，仅用于其 `sizeof`。

最大大小是 `IOCPARM_MAX = 8192` 字节。对于更大的传输，使用嵌入式指针模式（挑战 1）或不同的机制如 `mmap` 或 `read`/`write`。

### 快速参考：`d_ioctl_t` 签名

```c
int d_ioctl(struct cdev *dev, u_long cmd, caddr_t data,
            int fflag, struct thread *td);
```

| 参数 | 含义 |
|----------|---------|
| `dev` | 支持文件描述符的 cdev。使用 `dev->si_drv1` 获取 softc。 |
| `cmd` | 来自用户空间的请求代码。与命名常量比较。 |
| `data` | 带有用户数据的内核端缓冲区。直接解引用；不需要 `copyin`。 |
| `fflag` | 来自打开调用的文件标志（`FREAD`、`FWRITE`）。在改变之前检查。 |
| `td` | 调用线程。使用 `td->td_ucred` 获取凭据。 |

成功时返回 0，失败时返回正 errno，或未知命令时返回 `ENOIOCTL`。

### 快速参考：sysctl OID 宏

全部在 `/usr/src/sys/sys/sysctl.h` 中。

| 宏 | 添加 |
|-------|------|
| `SYSCTL_ADD_INT(ctx, parent, nbr, name, flags, ptr, val, descr)` | 有符号 int，由 `*ptr` 支持。 |
| `SYSCTL_ADD_UINT` | 无符号 int。 |
| `SYSCTL_ADD_LONG` / `SYSCTL_ADD_ULONG` | long / unsigned long。 |
| `SYSCTL_ADD_S64` / `SYSCTL_ADD_U64` | 64 位有符号 / 无符号。 |
| `SYSCTL_ADD_BOOL` | 布尔值（优于 int 0/1）。 |
| `SYSCTL_ADD_STRING(ctx, parent, nbr, name, flags, ptr, len, descr)` | NUL 终止字符串。 |
| `SYSCTL_ADD_NODE(ctx, parent, nbr, name, flags, handler, descr)` | 子树节点。 |
| `SYSCTL_ADD_PROC(ctx, parent, nbr, name, flags, arg1, arg2, handler, fmt, descr)` | 处理程序支持的 OID。 |

### 快速参考：sysctl 标志位

| 标志 | 含义 |
|------|---------|
| `CTLFLAG_RD` | 只读。 |
| `CTLFLAG_WR` | 只写（罕见）。 |
| `CTLFLAG_RW` | 读写。 |
| `CTLFLAG_TUN` | 加载器可调参数；在启动时从 `/boot/loader.conf` 读取。 |
| `CTLFLAG_MPSAFE` | 处理程序在没有 giant 锁的情况下调用是安全的。**新代码始终设置。** |
| `CTLFLAG_PRISON` | 在 jail 内可见。 |
| `CTLFLAG_VNET` | 每 VNET（虚拟化网络堆栈）。 |
| `CTLFLAG_DYN` | 动态 OID；由 `SYSCTL_ADD_*` 自动设置。 |
| `CTLFLAG_SECURE` | 当 `securelevel > 0` 时只读。 |

### 快速参考：sysctl 类型位

在 `SYSCTL_ADD_PROC` 和类似函数的标志字中 OR 运算。

| 标志 | 含义 |
|------|---------|
| `CTLTYPE_INT` / `CTLTYPE_UINT` | 有符号 / 无符号 int。 |
| `CTLTYPE_LONG` / `CTLTYPE_ULONG` | long / unsigned long。 |
| `CTLTYPE_S64` / `CTLTYPE_U64` | 64 位有符号 / 无符号。 |
| `CTLTYPE_STRING` | NUL 终止字符串。 |
| `CTLTYPE_OPAQUE` | 不透明 blob；新代码中很少使用。 |
| `CTLTYPE_NODE` | 子树节点。 |

### 快速参考：sysctl 处理程序格式字符串

由 `SYSCTL_ADD_PROC` 使用，告诉 `sysctl(8)` 如何打印值。

| 格式 | 类型 |
|--------|------|
| `"I"` | int |
| `"IU"` | unsigned int |
| `"L"` | long |
| `"LU"` | unsigned long |
| `"Q"` | int64 |
| `"QU"` | uint64 |
| `"A"` | NUL 终止字符串 |
| `"S,structname"` | 不透明结构（罕见） |

### 快速参考：sysctl 处理程序样板

```c
static int
my_handler(SYSCTL_HANDLER_ARGS)
{
        struct my_softc *sc = arg1;
        u_int val;

        /* 在互斥锁下将当前值读入 val。 */
        mtx_lock(&sc->sc_mtx);
        val = sc->sc_field;
        mtx_unlock(&sc->sc_mtx);

        /* 让框架进行读取或写入。 */
        return (sysctl_handle_int(oidp, &val, 0, req));
}
```

### 快速参考：每设备 Sysctl 上下文

```c
struct sysctl_ctx_list *ctx = device_get_sysctl_ctx(dev);
struct sysctl_oid      *tree = device_get_sysctl_tree(dev);
struct sysctl_oid_list *child = SYSCTL_CHILDREN(tree);
```

框架拥有 ctx；驱动程序不对其调用 `sysctl_ctx_free`。框架在成功分离后自动清理。

### 快速参考：加载器可调参数

```c
TUNABLE_INT_FETCH("hw.driver.knob", &sc->sc_knob);
TUNABLE_LONG_FETCH("hw.driver.knob", &sc->sc_knob);
TUNABLE_STR_FETCH("hw.driver.knob", buf, sizeof(buf));
```

第一个参数是加载器变量名。第二个是指向目标的指针，如果变量未设置，它也是默认值。

### 快速参考：ifnet 生命周期

| 函数 | 何时 |
|----------|-------------------|---------------------|
| `if_alloc(IFT_<type>)` | 分配 ifnet。 |
| `if_initname(ifp, name, unit)` | 设置用户可见名称（`ifconfig` 显示它）。 |
| `if_setflags(ifp, flags)` | 设置 `IFF_*` 标志。 |
| `if_setsoftc(ifp, sc)` | 附加驱动程序的 softc。 |
| `if_setioctlfn(ifp, fn)` | 设置 ioctl 处理程序。 |
| `if_settransmitfn(ifp, fn)` | 设置传输函数。 |
| `if_attach(ifp)` | 使接口可见。 |
| `bpfattach(ifp, dlt, hdrlen)` | 使流量对 BPF 可见。 |
| `bpfdetach(ifp)` | 反转 `bpfattach`。 |
| `if_detach(ifp)` | 反转 `if_attach`。 |
| `if_free(ifp)` | 释放 ifnet。 |

### 快速参考：CAM SIM 生命周期

| 函数 | 何时 |
|----------|-------------------|
| `cam_simq_alloc(maxq)` | 分配设备队列。 |
| `cam_sim_alloc(action, poll, name, sc, unit, mtx, max_tagged, max_dev_tx, devq)` | 分配 SIM。 |
| `xpt_bus_register(sim, dev, 0)` | 用 CAM 注册总线。 |
| `xpt_create_path(&path, NULL, cam_sim_path(sim), targ, lun)` | 为事件创建路径。 |
| `xpt_action(ccb)` | 向 CAM 发送 CCB。 |
| `xpt_done(ccb)` | 告诉 CAM 驱动程序完成了 CCB。 |
| `xpt_free_path(path)` | 反转 `xpt_create_path`。 |
| `xpt_bus_deregister(cam_sim_path(sim))` | 反转 `xpt_bus_register`。 |
| `cam_sim_free(sim, free_devq)` | 反转 `cam_sim_alloc`。传递 `TRUE` 也释放 devq。 |

### 快速参考：模块生命周期

```c
static moduledata_t mymod = {
        "myfirst",        /* name */
        myfirst_modevent, /* event handler, can be NULL */
        NULL              /* extra data, rarely used */
};
DECLARE_MODULE(myfirst, mymod, SI_SUB_DRIVERS, SI_ORDER_ANY);
MODULE_VERSION(myfirst, 1);
MODULE_DEPEND(myfirst, otherdriver, 1, 1, 1);
```

事件处理程序签名为 `int (*)(module_t mod, int what, void *arg)`。`what` 参数是 `MOD_LOAD`、`MOD_UNLOAD`、`MOD_QUIESCE` 或 `MOD_SHUTDOWN` 之一。成功返回 0 或正 errno。

### 快速参考：事件处理程序

```c
eventhandler_tag tag;

tag = EVENTHANDLER_REGISTER(event_name, callback,
    arg, EVENTHANDLER_PRI_ANY);

EVENTHANDLER_DEREGISTER(event_name, tag);
```

常见事件名称：`shutdown_pre_sync`、`shutdown_post_sync`、`shutdown_final`、`vm_lowmem`、`power_suspend_early`、`power_resume`。

### 快速参考：Errno 约定

| Errno | 何时返回 |
|-------|----------------|
| `0` | 成功。 |
| `EINVAL` | 参数被识别但无效。 |
| `EBADF` | 文件描述符未正确打开以进行操作。 |
| `EBUSY` | 资源正在使用中（通常从分离返回）。 |
| `ENOIOCTL` | ioctl 命令未知。**在 `d_ioctl` 的默认 case 中使用此值。** |
| `ENOTTY` | 内核对用户空间的 `ENOIOCTL` 翻译。 |
| `ENOMEM` | 分配失败。 |
| `EAGAIN` | 稍后重试（通常从非阻塞 I/O 返回）。 |
| `EPERM` | 调用者缺少必要权限。 |
| `EOPNOTSUPP` | 此驱动程序不支持该操作。 |
| `EFAULT` | 用户指针无效（由失败的 `copyin`/`copyout` 返回）。 |
| `ETIMEDOUT` | 等待超时。 |
| `EIO` | 来自硬件的通用 I/O 错误。 |

### 快速参考：调试类位（来自第 23 章）

| 位 | 名称 | 用于 |
|-----|------|----------|
| `0x01` | `MYF_DBG_INIT` | probe / attach / detach |
| `0x02` | `MYF_DBG_OPEN` | open / close lifecycle |
| `0x04` | `MYF_DBG_IO` | read / write paths |
| `0x08` | `MYF_DBG_IOCTL` | ioctl handling |
| `0x10` | `MYF_DBG_INTR` | interrupt handler |
| `0x20` | `MYF_DBG_DMA` | DMA mapping/sync |
| `0x40` | `MYF_DBG_PWR` | power-management events |
| `0x80` | `MYF_DBG_MEM` | malloc/free trace |
| `0xFFFFFFFF` | `MYF_DBG_ANY` | all classes |
| `0` | `MYF_DBG_NONE` | no logging |

掩码通过 `dev.<driver>.<unit>.debug.mask` 每实例设置，或在启动时通过 `/boot/loader.conf` 中的 `hw.<driver>.debug_mask_default` 全局设置。

### 集成词汇表

**API 版本**：通过驱动程序的 ioctl 接口暴露的整数（通常通过 `GETVER` ioctl），用户空间程序可以查询它以检测驱动程序公共接口的变化。仅当用户可见接口不兼容更改时才提升。

**bpfattach**：将 `ifnet` 挂钩到 BPF（Berkeley Packet Filter）机制以便 `tcpdump` 和类似工具可以观察其流量的函数。必须与 `bpfdetach` 配对。

**bus_generic_detach**：一个辅助函数，分离总线式驱动程序的每个子设备。在总线驱动程序的分离中作为第一步使用，以便在父设备拆卸自己的状态之前释放子设备。

**CAM**：Common Access Method，FreeBSD 在设备驱动程序层之上的存储子系统。拥有 I/O 队列、目标/LUN 抽象和将块 I/O 转换为协议特定命令的外围驱动程序（`da`、`cd`、`sa`）。

**CCB**：CAM Control Block。单个 I/O 请求结构化为标记联合；驱动程序检查 `ccb->ccb_h.func_code` 并相应分发。通过 `xpt_done` 完成。

**cdev**：字符设备。支持 `/dev` 下条目的内核每次设备节点对象。用 `make_dev_s` 创建，用 `destroy_dev` 销毁。

**cdevsw**：字符设备开关。内核对 cdev 的操作（`d_open`、`d_close`、`d_read`、`d_write`、`d_ioctl`、...）调用的静态回调表。

**copyin / copyout**：在用户空间和内核空间地址之间传输字节的函数。内核对正确编码的 ioctl 自动执行它们；驱动程序仅对嵌入式指针模式显式调用它们。

**CTLFLAG_MPSAFE**：一个 sysctl 标志，声明 OID 的处理程序在没有 giant 锁的情况下调用是安全的。新代码强制；没有它，内核在每次访问时获取 giant 锁。

**d_ioctl_t**：cdevsw 的 ioctl 回调的函数指针类型。签名：`int (*)(struct cdev *, u_long, caddr_t, int, struct thread *)`。

**d_priv**：通过 `devfs_set_cdevpriv` 附加的每次文件描述符私有指针。用于必须与一次打开关联而不是与驱动程序实例关联的状态。

**dev_ref / dev_rel**：一对增加和减少 cdev 引用计数的函数。用于协调分离与进行中的回调；参见挑战 3。

**devfs**：支持 `/dev` 的内核管理文件系统。驱动程序创建 cdev，devfs 使它们可见。

**devfs.rules(8)**：一种用于运行时 devfs 权限的配置机制。在编辑 `/etc/devfs.rules` 后通过 `service devfs restart` 应用。

**DTrace**：动态跟踪框架。驱动程序通过 SDT 宏暴露探测点；DTrace 脚本在运行时附加到它们。

**EVENTHANDLER_REGISTER**：为命名系统范围事件（`shutdown_pre_sync`、`vm_lowmem` 等）注册回调的机制。必须与 `EVENTHANDLER_DEREGISTER` 配对。

**ifnet**：网络堆栈的每次接口对象。cdev 的网络对应物。

**if_t**：网络堆栈用于 `ifnet` 的不透明 typedef。驱动程序通过访问器函数操作接口，而不是直接字段访问。

**IOC_VOID / IOC_IN / IOC_OUT / IOC_INOUT**：在 ioctl 请求代码中编码的四个方向位。内核使用它们决定执行什么 `copyin`/`copyout`。

**IOCPARM_MAX**：ioctl 参数结构的最大大小（8192 字节），如请求代码中编码的那样。更大的传输需要嵌入式指针模式。

**kldload / kldunload**：加载和卸载内核模块的用户空间工具。两者都调用相应的模块事件处理程序（`MOD_LOAD` 和 `MOD_UNLOAD`）。

**make_dev_args**：传递给 `make_dev_s` 以描述新 cdev 的结构。用 `make_dev_args_init` 初始化。

**make_dev_s**：现代创建 cdev 的首选函数。返回 0 或正 errno；成功时设置 `*cdev`。

**MAKEDEV_***：传递给 `make_dev_credf` 和类似函数的标志位。常见位：`MAKEDEV_REF`、`MAKEDEV_NOWAIT`、`MAKEDEV_ETERNAL_KLD`。

**MOD_LOAD / MOD_UNLOAD / MOD_SHUTDOWN**：传递给模块事件处理程序的事件。返回 0 确认或非零拒绝。

**MODULE_DEPEND**：声明模块对另一个模块的依赖关系的宏。内核使用版本参数（`min`、`pref`、`max`）强制兼容性。

**MODULE_VERSION**：声明模块版本号的宏。当内核内用户需要重新编译时提升。

**Newbus**：FreeBSD 的设备树框架。拥有 `device_t`、每设备 softc、每设备 sysctl 上下文，以及 probe/attach/detach 生命周期。

**OID**：对象标识符。sysctl 树中的节点。静态 OID 在编译时声明；动态 OID 在运行时用 `SYSCTL_ADD_*` 添加。

**路径（CAM）**：将目标标识为 `(bus, target, LUN)` 三元组。用 `xpt_create_path` 创建。

**公共头文件**：用户空间程序包含以与驱动程序对话的头文件。必须在未定义 `_KERNEL` 的情况下干净编译；仅使用稳定类型。

**注册框架**：为驱动程序暴露"分配-命名-填充-附加-流量-分离-释放"接口的 FreeBSD 子系统。示例：网络（`ifnet`）、存储（CAM）、声音、USB、GEOM。

**发布版本**：标识驱动程序发布的人可读字符串。通过 sysctl 作为 `dev.<driver>.<unit>.version` 暴露。

**SDT**：静态定义跟踪。内核用于 DTrace 可消费的编译时探测点的机制。

**si_drv1 / si_drv2**：`struct cdev` 中供驱动程序使用的两个私有指针字段。约定上 `si_drv1` 指向 softc。

**SIM**：SCSI Interface Module。CAM 对存储适配器的视图。用 `cam_sim_alloc` 分配，用 `xpt_bus_register` 注册。

**软分离**：一种分离模式，驱动程序等待未完成的引用降至零而不是立即拒绝分离。参见挑战 3。

**Softc**：软件上下文。驱动程序的每次实例状态。由 Newbus 分配并通过 `device_get_softc(dev)` 访问。

**SYSINIT**：编译时注册的一次性内核初始化函数。在引导期间特定阶段运行。驱动程序代码中很少需要。

**SYSCTL_HANDLER_ARGS**：一个宏，扩展为 sysctl 处理程序的标准参数列表：`oidp, arg1, arg2, req`。

**TUNABLE_INT_FETCH**：一个从加载器环境读取值并将其写入内核变量的函数。如果加载器变量不存在，变量保留其先前值。

**XPT**：CAM Transport Layer。中央 CAM 分发机制。驱动程序调用 `xpt_action` 向 CAM 发送 CCB；CAM 通过 SIM 的 action 函数回调到驱动程序中处理针对驱动程序总线的 I/O CCB。

### 配套文件清单

本章的配套文件位于书籍仓库的 `examples/part-05/ch24-integration/` 下。目录布局如下：

```text
examples/part-05/ch24-integration/
├── README.md
├── INTEGRATION.md
├── stage1-devfs/
│   ├── Makefile
│   ├── myfirst.c             (带有 make_dev_args)
│   └── README.md
├── stage2-ioctl/
│   ├── Makefile
│   ├── Makefile.user         (用于 myfirstctl)
│   ├── myfirst_ioctl.c
│   ├── myfirst_ioctl.h       (公共)
│   ├── myfirstctl.c
│   └── README.md
├── stage3-sysctl/
│   ├── Makefile
│   ├── myfirst.c
│   ├── myfirst_sysctl.c
│   └── README.md
└── labs/
    ├── lab24_1_stage1.sh     (实验室 1 的验证命令)
    ├── lab24_2_stage2.sh
    ├── lab24_3_stage3.sh
    ├── lab24_4_failure.sh
    ├── lab24_5_dtrace.sh
    ├── lab24_6_smoke.sh
    ├── lab24_7_reload.sh
    └── loader.conf.example
```

实验室 1 的起点是读者自己的第 23 章末尾驱动程序（`1.6-debug`）；`stage1-devfs/`、`stage2-ioctl/` 和 `stage3-sysctl/` 是读者在每个实验室完成后可以咨询的参考解决方案。实验室目录中有小型 shell 脚本，执行验证命令，读者可以调整用于自己的测试。

章节根目录的 `README.md` 描述如何使用目录、阶段的顺序以及分阶段树之间的关系。`INTEGRATION.md` 是更长的文档，将章节中的每个概念映射到它出现的文件。

### 章节的源代码树在真实 FreeBSD 中的位置

对于想要查阅章节引用的树内实现的读者，这里是最重要文件的简短索引：

| 概念 | 树内文件 |
|---------|--------------|
| ioctl 编码 | `/usr/src/sys/sys/ioccom.h` |
| cdevsw 定义 | `/usr/src/sys/sys/conf.h` |
| make_dev 系列 | `/usr/src/sys/kern/kern_conf.c` |
| sysctl 框架 | `/usr/src/sys/sys/sysctl.h`, `/usr/src/sys/kern/kern_sysctl.c` |
| ifnet API | `/usr/src/sys/net/if.h`, `/usr/src/sys/net/if.c` |
| ifnet 示例 | `/usr/src/sys/net/if_disc.c` |
| CAM SIM API | `/usr/src/sys/cam/cam_xpt.h`, `/usr/src/sys/cam/cam_sim.h` |
| CAM 示例 | `/usr/src/sys/dev/ahci/ahci.c` |
| EVENTHANDLER | `/usr/src/sys/sys/eventhandler.h` |
| 模块机制 | `/usr/src/sys/sys/module.h`, `/usr/src/sys/kern/kern_module.c` |
| TUNABLE | `/usr/src/sys/sys/sysctl.h`（搜索 `TUNABLE_INT_FETCH`） |
| SDT 探针 | `/usr/src/sys/sys/sdt.h`, `/usr/src/sys/cddl/dev/sdt/sdt.c` |
| `null(4)` 参考 | `/usr/src/sys/dev/null/null.c` |

与章节一起阅读这些文件是任何想要加深集成知识的读者的下一步。特别是 `null(4)` 驱动程序值得完整阅读；它足够小，可以一次消化，并演示了本章涵盖的几乎所有模式。

### 未纳入本章的内容

一些集成主题属于 FreeBSD 更广泛的工具箱，但在这里没有获得专门章节，要么因为它们以会分散初学者注意力的方式特定于子系统，要么因为它们在后面的章节中更全面地涵盖。在这里命名它们使章节对其范围保持诚实，并给读者一个前瞻性的地图。

第一个遗漏是 `geom(4)`。暴露块设备的驱动程序挂接到 GEOM 而不是 CAM。注册模式与 cdev 模式类似（分配 `g_geom`，填充 `g_class` 回调，调用 `g_attach`），但词汇差异大到将其混入章节会模糊存储与字符的区别。原始磁盘和伪磁盘目标的驱动程序位于这个附近；规范参考是 `/usr/src/sys/geom/geom_disk.c`。

第二个遗漏是 `usb(4)`。USB 驱动程序通过 `usb_attach` 和 USB 特定的方法表而不是直接通过 Newbus 向 USB 堆栈注册。集成表面（devfs、sysctl）一旦设备附加就相同，但上边缘由 USB 堆栈拥有。规范参考在 `/usr/src/sys/dev/usb/` 下。

第三个遗漏是 `iicbus(4)` 和 `spibus(4)`。与 I2C 或 SPI 外设通信的驱动程序作为总线驱动程序的子设备附加，并使用总线特定的传输例程。集成表面保持相同，但驱动现代 Arm SoC 的设备树和 FDT 集成添加了值得单独章节的词汇。第六部分在其适当的上下文中涵盖这些表面。

第四个遗漏是 `kqueue(2)` 和 `poll(2)` 集成。想要唤醒阻塞在 `select`、`poll` 或 `kqueue` 上的用户空间程序的字符驱动程序必须实现 `d_kqfilter`（以及可选的 `d_poll`），将 `selwakeup` 和 `KNOTE` 连接到数据路径，并提供一小组过滤操作。该机制并不难，但在概念上是基本 cdev 契约之上的一层；我们将在第 26 章返回讨论它。

今天需要这些表面之一的读者应该将本章的模式视为基础，并寻求上面命名的树内参考。规范（在附加时注册，在分离时注销，在改变回调时持有单个互斥锁，版本化公共表面）是相同的。

### 读者自检

在翻页之前，完成章节的读者应该能够在不查阅文本的情况下回答以下问题。每个问题映射到介绍基础材料的章节。如果问题不熟悉，在继续之前重新访问括号中列出的章节部分是正确的。

1. `D_TRACKCLOSE` 如何改变 `d_close` 的调用方式？（第 2 节）
2. 为什么 `mda_si_drv1` 比在 `make_dev` 返回后分配 `si_drv1` 更可取？（第 2 节）
3. `_IOR('M', 1, uint32_t)` 宏在生成的请求代码中编码了什么？（第 3 节）
4. 为什么分发器的默认分支必须返回 `ENOIOCTL` 而不是 `EINVAL`？（第 3 节）
5. 哪个内核函数拆卸每设备 sysctl 上下文，何时运行？（第 4 节）
6. `CTLFLAG_TUN` 如何与 `TUNABLE_INT_FETCH` 合作以应用启动时的值？（第 4 节）
7. `MYFIRST_VERSION` 字符串、`MODULE_VERSION` 整数和 `MYFIRST_IOCTL_VERSION` 整数之间有什么区别？（第 8 节）
8. 为什么附加中的清理链使用反向顺序的标记 `goto` 而不是嵌套的 `if` 语句？（第 7 节）
9. 本章的软分离模式与挑战 3 概述的 `dev_ref`/`dev_rel` 模式有何不同？（第 7 和 10 节）
10. 几乎每个驱动程序都需要哪两个集成表面，仅加入特定子系统的驱动程序才需要哪两个？（第 1、2、3、4、5、6 节）

不犹豫地回答大多数这些问题的读者已经内化了章节，并为接下来的内容做好了准备。在两个以上问题上犹豫的读者应该在解决第 25 章的维护规范之前重新访问相关部分。
