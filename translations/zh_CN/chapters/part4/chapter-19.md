---
title: "处理中断"
description: "第19章将第18章的PCI驱动程序转变为中断感知驱动程序。它讲解中断是什么、FreeBSD如何建模和路由中断、驱动程序如何通过bus_setup_intr(9)声明IRQ资源并注册处理程序、如何在快速过滤器和延迟ithread之间分配工作、如何使用FILTER_STRAY和FILTER_HANDLED安全处理共享IRQ、如何在不使用真实IRQ事件的情况下模拟中断进行测试，以及在detach中如何拆除处理程序。驱动程序从1.1-pci发展到1.2-intr，新增了一个中断专用文件，并为第20章的MSI和MSI-X做好准备。"
partNumber: 4
partName: "硬件与平台级集成"
chapter: 19
lastUpdated: "2026-04-19"
status: "complete"
author: "Edson Brandi"
reviewer: "TBD"
translator: "AI辅助翻译为简体中文"
language: "zh-CN"
estimatedReadTime: 210
---

# 处理中断

## 读者指南与学习目标

第18章结束时，驱动程序终于与真实的PCI硬件建立了联系。`myfirst`模块版本`1.1-pci`通过供应商ID和设备ID探测PCI设备，作为`pci0`的proper newbus子设备进行attach，通过`bus_alloc_resource_any(9)`配合`SYS_RES_MEMORY`和`RF_ACTIVE`声明设备的BAR，将BAR交给第16章的访问器层以便`CSR_READ_4`和`CSR_WRITE_4`读写真实的硅片，创建每个实例的cdev，并在detach时严格按照相反顺序拆除所有内容。第17章的模拟仍在源码树中但在PCI路径上不运行；其callout保持静默，因此无法写入真实设备的寄存器。

驱动程序目前尚未做的是响应设备。迄今为止的每一次寄存器访问都由驱动程序主动发起：用户空间的`read`或`write`到达cdev，cdev处理函数获取`sc->mtx`，访问器读取或写入BAR，控制返回用户空间。如果设备本身有话要说，比如"我的接收队列有数据包"、"命令已完成"或"温度阈值被越过"，驱动程序无从得知。驱动程序轮询；它不倾听。

这正是第19章要解决的问题。真实设备通过**中断**CPU来对话。总线架构将信号从设备传递到中断控制器，中断控制器将信号分发到CPU，CPU从当前工作中短暂偏离，驱动程序注册的处理程序运行几微秒。处理程序的任务很小：弄清楚设备想要什么，在设备端确认中断，在此处安全地完成少量工作，然后将其余工作交给一个可以阻塞、睡眠或获取慢锁的线程。"移交"是现代中断纪律的第二半；第一半是进入处理程序本身。

第19章的范围正是核心中断路径：中断在硬件层面是什么、FreeBSD如何在内核中建模它们、驱动程序如何通过`bus_setup_intr(9)`分配IRQ资源并注册处理程序、快速过滤器处理程序与延迟ithread处理程序之间的分离如何运作、`INTR_MPSAFE`标志对驱动程序有何约束、当真实事件不易产生时如何模拟中断进行测试、如何在其他驱动程序也监听的共享IRQ线上正确行为、以及如何在detach中拆除所有这些内容而不泄漏资源或执行过时的处理程序。本章暂不涉及MSI和MSI-X，它们属于第20章；这些机制建立在读者在此处编写的核心处理程序之上，同时教授两者会稀释两者。

第19章有意回避了中断工作自然会触及的一些领域。MSI和MSI-X、每向量处理程序、中断合并、每队列中断路由在第20章。DMA以及中断与DMA描述符环之间的交互在第20章和第21章。NUMA平台上的高级中断亲和策略有简短讨论，但深入处理属于第20章及后续章节。平台特定的中断路由（arm64上的GICv3、x86上的APIC、嵌入式目标上的NVIC）仅提及词汇；本书关注的是隐藏这些差异的驱动可见API。第19章保持在它能良好覆盖的范围内，在话题值得单独成章的地方明确移交。

第19章教授的过滤器加ithread模型并不孤立。第16章给了驱动程序寄存器访问的词汇。第17章教它像设备一样思考。第18章将其介绍给真实的PCI设备。第19章给了它耳朵。第20章和第21章将给它腿：直接内存访问，使设备无需驱动程序介入即可访问RAM。每章添加一层。每层依赖之前的层。第19章是驱动程序停止轮询开始倾听的地方，第3部分建立的纪律让倾听保持诚实。

### 为什么中断处理值得单独一章

这里浮现的一个疑问是`bus_setup_intr(9)`和过滤器加ithread模型是否真的值得整整一章。第17章的模拟使用callout产生自主状态变化；第18章的驱动程序在真实PCI上运行但完全忽略中断线。我们能不能继续通过callout轮询并避开这个话题？

两个原因。

第一个是性能。每秒轮询设备十次的callout在无事发生时浪费CPU时间，并错过轮询之间发生的事件。真实设备每毫秒可能产生多个事件；100毫秒的轮询间隔几乎错过所有事件。中断反转了成本：无事发生时不消耗CPU，事件发生后几微秒内处理程序运行。FreeBSD中每个严肃的驱动程序都使用中断，原因相同；轮询的驱动程序是有特殊理由的驱动程序。

第二个是正确性。某些设备要求驱动程序在严格的时间窗口内响应。网卡接收FIFO在几微秒内填满；如果驱动程序不排空它，网卡丢弃数据包。串口发送FIFO以线路速率排空；如果驱动程序不重新填充它，发送器饥饿。任何间隔足够长以节省成本的轮询，也是足够短以错过截止时间的间隔。中断是让驱动程序在不让CPU全职燃烧的情况下满足实时设备要求的唯一机制。

本章也因教授一种远超PCI的纪律而赢得其位置。FreeBSD中断模型（过滤器加ithread、`INTR_MPSAFE`、`bus_setup_intr(9)`、detach中的清理拆除）是USB驱动程序使用的同一模型，是SDIO驱动程序使用的同一模型，是virtio驱动程序使用的同一模型，也是arm64 SoC驱动程序使用的同一模型。理解第19章模型的读者可以理解阅读任何FreeBSD驱动程序的中断处理程序。这种通用性是即使不从事PCI工作的读者也值得仔细阅读本章的原因。

### 第18章为驱动程序留下的起点

在继续之前做一个简短的检查点。第19章扩展第18章阶段4结束时产生的驱动程序，标记为版本`1.1-pci`。如果以下任何项目感觉不确定，请在开始本章之前返回第18章。

- 您的驱动程序编译干净，在`kldstat -v`中标识为`1.1-pci`。
- 在暴露virtio-rnd设备（供应商`0x1af4`，设备`0x1005`）的bhyve或QEMU guest上，驱动程序通过`myfirst_pci_probe`和`myfirst_pci_attach`进行attach，打印其banner，将BAR 0声明为`SYS_RES_MEMORY`配合`RF_ACTIVE`，遍历PCI能力列表，并创建`/dev/myfirst0`。
- softc持有BAR资源指针（`sc->bar_res`）、资源ID（`sc->bar_rid`）和`pci_attached`标志。
- detach路径销毁cdev、静默活动callout和任务、分离硬件层、释放BAR，并反初始化softc。
- 第18章的完整回归脚本通过：attach、exercise cdev、detach、unload，无泄漏。
- `HARDWARE.md`、`LOCKING.md`、`SIMULATION.md`和`PCI.md`是当前的。
- `INVARIANTS`、`WITNESS`、`WITNESS_SKIPSPIN`、`DDB`、`KDB`和`KDB_UNATTENDED`在您的测试内核中启用。

那个驱动程序是第19章扩展的基础。新增内容在数量上同样适中：一个新文件（`myfirst_intr.c`）、一个新头文件（`myfirst_intr.h`）、少量新的softc字段（`irq_res`、`irq_rid`、`intr_cookie`、一两个计数器）、中断文件中的三个新函数（setup、teardown、过滤器处理程序）、一个用于模拟中断的sysctl、版本号提升到`1.2-intr`、以及简短的`INTERRUPTS.md`文档。心智模型的改变同样比行数暗示的更大：驱动程序终于有了两个控制线程而不是一个，保持它们不相互踩踏的纪律是新的。

### 您将学到什么

到本章结束时，您将能够：

- 解释中断在硬件层面是什么，边沿触发和电平触发信号的区别，以及CPU的中断处理流程如何从设备到达驱动程序的处理程序。
- 描述FreeBSD如何表示中断事件：中断事件（`intr_event`）是什么、中断线程（`ithread`）是什么、过滤器处理程序是什么，以及过滤器与ithread之间的分离为何重要。
- 阅读`vmstat -i`和`devinfo -v`的输出，定位您系统正在处理的中断、它们的计数以及绑定到每个中断的驱动程序。
- 通过`bus_alloc_resource_any(9)`配合`SYS_RES_IRQ`分配IRQ资源，在传统PCI线上使用`rid = 0`（第20章中使用非零RID用于MSI和MSI-X向量）。
- 通过`bus_setup_intr(9)`注册中断处理程序，选择过滤器处理程序（`driver_filter_t`）、ithread处理程序（`driver_intr_t`）或过滤器加ithread组合，并为设备的工作类别选择正确的`INTR_TYPE_*`标志。
- 编写一个最小过滤器处理程序，读取设备的状态寄存器，在设备端确认中断，适当地返回`FILTER_HANDLED`或`FILTER_STRAY`，并与内核的中断机制协作。
- 知道在过滤器中什么是安全的（仅自旋锁，无`malloc`，无睡眠，无阻塞锁）以及ithread放宽了什么（睡眠互斥锁、条件变量、`malloc(M_WAITOK)`），以及为什么存在这些约束。
- 仅在您真正需要时设置`INTR_MPSAFE`，并理解该标志对驱动程序的承诺（自己的同步、无隐式Giant获取、可在任何CPU上并发运行的权利）。
- 将延迟工作从过滤器处理程序交给taskqueue任务或ithread，保持小量紧急工作在过滤器中完成而大量工作在线程上下文中完成的纪律。
- 通过sysctl模拟中断，在正常锁定规则下直接调用处理程序，使读者可以演练处理程序的状态机而无需真实IRQ触发。
- 正确处理共享中断线：首先读取设备的INTR_STATUS寄存器，判断此中断是否属于我们的设备，如果不属于则返回`FILTER_STRAY`，避免窃取其他驱动程序的工作。
- 在detach中通过`bus_teardown_intr(9)`拆除中断处理程序，然后通过`bus_release_resource(9)`释放IRQ，并构造detach路径使中断无法对已释放的状态触发。
- 认识什么是中断风暴，知道FreeBSD的`hw.intr_storm_threshold`机制如何检测风暴，并理解常见的设备端原因（未能清除INTR_STATUS、边沿触发线被误配置为电平触发）。
- 当亲和性重要时通过`bus_bind_intr(9)`将中断绑定到特定CPU，并通过`bus_describe_intr(9)`向`devinfo -v`描述中断，使操作者可以看到哪个处理程序在哪个CPU上。
- 将中断相关代码拆分到自己的文件，更新模块的`SRCS`行，将驱动程序标记为`1.2-intr`，并生成简短的`INTERRUPTS.md`文档描述处理程序的行为和延迟工作纪律。

列表很长；每个项目范围狭窄。本章的意义在于组合。

### 本章不涵盖的内容

几个相邻话题明确推迟，以保持第19章专注。

- **MSI和MSI-X。**`pci_alloc_msi(9)`、`pci_alloc_msix(9)`、向量分配、每向量处理程序和MSI-X表布局在第20章。第19章针对用`rid = 0`分配的传统PCI INTx线；词汇可以迁移，但每向量机制不能。
- **DMA。**`bus_dma(9)`标签、scatter-gather列表、bounce缓冲区、DMA描述符周围的缓存一致性，以及中断如何信号描述符环传输完成在第20章和第21章。第19章的处理程序读取BAR寄存器并决定做什么；它不触及DMA。
- **每队列多队列网络。**现代NIC有独立的接收和发送队列，有独立的MSI-X向量和中断处理程序。`iflib(9)`框架构建于此；`em(4)`、`ix(4)`和`ixl(4)`使用它。第19章的驱动程序有一个中断；第20章开始发展多队列故事。
- **NUMA硬件上的深度中断亲和。**`bus_bind_intr`有介绍；将中断绑定到靠近设备PCIe根端口的CPU的精细策略留给后续关于可扩展性的章节。
- **围绕中断的驱动程序暂停和恢复。**`bus_suspend_intr(9)`和`bus_resume_intr(9)`存在；为完整性提及但在第19章驱动程序中不演练。
- **实时中断优先级操作。**FreeBSD的`intr_priority(9)`和`INTR_TYPE_*`标志影响ithread优先级，但本书将优先级系统视为高级话题章节之外的黑盒。
- **纯软件中断（SWI）。**`swi_add(9)`创建驱动程序可以从任意上下文调度的纯软件中断。本章在讨论延迟工作时提及SWI，但首选的现代模式（taskqueue）以更少的陷阱覆盖相同用例。

保持在这些界限内使第19章成为关于核心中断处理的章节。词汇是可以迁移的；后续特定章节将词汇应用于MSI/MSI-X、DMA和多队列设计。

### 预估时间投入

- **仅阅读**：四到五小时。中断模型概念上不大但需要仔细阅读，特别是过滤器与ithread周围以及过滤器内的安全规则。
- **阅读加输入工作示例**：两到三次会话共十到十二小时。驱动程序分四个阶段演进；每个阶段是对第18章代码基础的小而真实的扩展。
- **阅读加所有实验和挑战**：四到五次会话共十六到二十小时，包括搭建bhyve实验室（如果第18章的设置尚未就位）、阅读`if_em.c`的中断路径和`if_mgb.c`的过滤器处理程序、以及针对模拟中断路径和（在可能的情况下）真实中断路径运行第19章回归测试。

第3、4和6节是最密集的。如果过滤器与ithread分离在第一次阅读时感觉陌生，这是正常的。停下来，重读第3节的决策树，在形状稳定后继续。

### 前提条件

在开始本章之前，确认：

- 您的驱动程序源码匹配第18章阶段4（`1.1-pci`）。起点假设第16章硬件层、第17章模拟后端、第18章PCI attach、完整`CSR_*`访问器家族、同步头文件以及第3部分引入的每个原语。
- 您的实验室机器运行FreeBSD 14.3，磁盘上有`/usr/src`并匹配运行内核。
- 已构建、安装并干净启动具有`INVARIANTS`、`WITNESS`、`WITNESS_SKIPSPIN`、`DDB`、`KDB`和`KDB_UNATTENDED`的调试内核。
- `bhyve(8)`或`qemu-system-x86_64`可用，第18章的实验室环境（带有`-s 4:0,virtio-rnd`的virtio-rnd设备的FreeBSD guest）可按需重现。
- `devinfo(8)`、`vmstat(8)`和`pciconf(8)`工具在您的路径中。三者都在基本系统中。

如果上述任何项目不稳固，现在修复它而不是勉强推进第19章并试图从移动的基础上推理。中断bug常表现为负载下的内核崩溃或微妙损坏；调试内核的`WITNESS`特别能早期捕获常见的锁错误类别。

### 如何充分利用本章

四个习惯会快速回报。

首先，将`/usr/src/sys/sys/bus.h`和`/usr/src/sys/kern/kern_intr.c`加入书签。第一个文件定义您将在每个处理程序中使用的`driver_filter_t`、`driver_intr_t`、`INTR_TYPE_*`、`INTR_MPSAFE`和`FILTER_*`返回值。第二个文件是内核的中断事件机制：接收低级IRQ、分发到过滤器、唤醒ithread并检测中断风暴的代码。您不需要深入阅读`kern_intr.c`，但粗略浏览前一千行一次，能给您描绘"设备断言IRQ 19"和"您的过滤器被调用"之间发生的事情。

其次，在实验室主机和guest上运行`vmstat -i`，并在阅读时保持输出在终端中打开。第2节和第3节介绍的每个概念（每处理程序计数、每CPU亲和、中断命名约定）都在该输出中可见。一个凝视过自己机器`vmstat -i`的读者会发现中断路由更不那么抽象。

第三，手动输入更改并运行每个阶段。中断代码是小错误变成沉默bug的地方。忘记`FILTER_HANDLED`使您的处理程序技术上非法；忘记`INTR_MPSAFE`在您的处理程序周围悄悄获取Giant；忘记清除INTR_STATUS在五毫秒后产生中断风暴。手动输入每行，每次`kldload`后检查`dmesg`输出，并在迭代之间观察`vmstat -i`，这是在错误便宜时捕获它们的方式。

第四，在阅读第4节后阅读`/usr/src/sys/dev/mgb/if_mgb.c`（查找`mgb_legacy_intr`和`mgb_admin_intr`）。`mgb(4)`是Microchip LAN743x千兆以太网控制器的驱动程序。其中断路径是过滤器加ithread设计的干净、可读示例，复杂性处于第19章教授的水平。七百行的仔细阅读会在第4部分的其余部分产生回报。

### 本章路线图

各节顺序如下：

1. **什么是中断？** 硬件画面：中断是什么、边沿与电平触发、CPU的分发流程，以及中断到达时驱动程序必须做的最少事情。概念基础。
2. **FreeBSD中的中断。** 内核如何表示中断事件、ithread是什么、中断如何通过`vmstat -i`和`devinfo -v`计数和显示，以及从IRQ线到驱动程序处理程序发生了什么。
3. **注册中断处理程序。** 驱动程序编写的代码：`bus_alloc_resource_any(9)`配合`SYS_RES_IRQ`、`bus_setup_intr(9)`、`INTR_TYPE_*`标志、`INTR_MPSAFE`、`bus_describe_intr(9)`。第19章驱动程序的第一个阶段（`1.2-intr-stage1`）。
4. **编写真实中断处理程序。** 过滤器处理程序的形状：读取INTR_STATUS、判断所有权、确认设备、返回正确的`FILTER_*`值。ithread处理程序的形状：获取睡眠锁、延迟到taskqueue、做慢工作。阶段2（`1.2-intr-stage2`）。
5. **使用模拟中断进行测试。** 一个sysctl在正常锁定规则下同步调用处理程序，使读者可以在无需真实IRQ的情况下演练处理程序。阶段3（`1.2-intr-stage3`）。
6. **处理共享中断。** 为什么`RF_SHAREABLE`在传统PCI线上重要，过滤器处理程序如何必须判断所有权相对于同一IRQ上的其他处理程序，以及如何避免饥饿。无阶段提升；这是一种纪律，不是新代码产物。
7. **清理中断资源。** 先`bus_teardown_intr(9)`，然后`bus_release_resource(9)`。detach排序现在多了两步，部分失败级联多了一个标签。
8. **重构和版本化您的中断就绪驱动程序。** 最终拆分到`myfirst_intr.c`、新`INTERRUPTS.md`、版本提升到`1.2-intr`、以及回归通过。阶段4。

八个节之后是动手实验、挑战练习、故障排除参考、收尾总结结束第19章故事并开启第20章，以及通往第20章的桥梁。本章末尾的参考和速查材料旨在在您阅读第20和21章时重读；第19章的词汇是两者依赖的基础。

如果这是您的第一次阅读，请按顺序线性阅读并按顺序做实验。如果您是重温，第3和4节独立，适合单次阅读。

## 第1节：什么是中断？

在驱动程序代码之前，先看硬件画面。第1节在CPU和总线层面教授中断是什么，不涉及任何FreeBSD特定词汇。理解第1节的读者可以将内核的中断路径作为具体对象而不是模糊抽象来阅读本章其余部分。回报是后续每一节都更容易。

一句话的总结，您可以在本章其余部分随身携带：中断是设备打断CPU当前工作、短时间运行驱动程序处理程序、然后让CPU回到原来做的事情的方式。其余一切都是围绕这句话的机制。

### 中断解决的问题

CPU按顺序运行指令流。每条指令完成，程序计数器前进，下一条指令运行，以此类推。如果不受打扰，CPU会执行一个程序直到该程序完成，然后另一个，以此类推，从不注意其指令流之外发生的事情。

这不是计算机的工作方式。按下半秒的键盘产生四五个独立事件；网络数据包在前一个几微秒后到达；磁盘完成读取、风扇控制器越过温度阈值、传感器值更新、定时器过期。每个事件都在CPU直接控制之外发生，发生在CPU未选择的时间。CPU必须注意到。

一种注意到的方式是轮询。CPU可以定期查看设备的状态寄存器。如果状态寄存器说"我有数据"，CPU读取数据。如果状态寄存器说"我什么都没有"，CPU继续。轮询对事件罕见、可预测、不时间敏感的设备有效。对其他所有设备效果不佳。每百毫秒轮询一次的键盘感觉迟钝。每毫秒轮询一次的网卡仍然错过大部分数据包。而且轮询消耗的CPU时间与轮询速率成正比，即使在无事发生时也是如此。

另一种注意到的方式是让设备告诉CPU。这就是中断。设备在导线上发出信号或通过总线发送消息。CPU打断当前工作，记住它在哪里，运行一小段代码询问设备发生了什么，适当响应，然后恢复被打断的工作。编写那段"小段代码"的纪律是第19章其余部分教授的内容。

### 硬件中断实际是什么

物理上，硬件中断始于导线上的信号（或更常见于现代系统，总线上的消息）。当操作系统需要知道的某事发生时，设备断言信号。例子包括：

- 当数据包到达并停留在其接收FIFO时，网卡断言其IRQ线。
- 当字节到达接收器，或发送FIFO低于阈值时，串口UART断言其IRQ线。
- 当命令队列条目完成时，SATA控制器断言其IRQ线。
- 当程序间隔过期时，定时器芯片断言其IRQ线。
- 当越过程序阈值时，温度传感器断言其IRQ线。

断言是设备说"有件事我需要您知道"的方式。CPU和操作系统必须准备好响应。从"信号断言"到"处理程序被调用"的路径经过中断控制器、CPU的中断分发机制和内核的中断事件机制。第2节走完整条路径；本小节保持在硬件层面。

关于信号本身的一些有用事实。

首先，**中断线通常共享**。CPU有少量中断输入，传统PC上常为16到24个，通过APIC和GIC的现代平台更多。系统通常设备多于中断输入，所以多个设备共享单条线。当共享线上中断触发时，每个设备可能是源的驱动程序都必须检查：这是我的中断吗？如果不是，返回"杂散"指示；如果是，处理它。第6节覆盖共享中断协议。

其次，**中断信号有两种风格**。边沿触发信号意味着中断由导线上的转换（低到高，或高到低）信号。电平触发信号意味着中断由将导线保持在特定电平（高或低）来信号，只要中断待处理就保持。两种风味有不同的操作后果，下一小节探讨。

第三，**中断相对于CPU是异步的**。CPU不知道设备何时会发出信号。驱动程序的处理程序必须容忍在驱动程序自己工作的任何点被调用，并必须适当地与其自己的非中断代码同步。第11章的锁定纪律是驱动程序用来做到这一点的方式。

第四，**中断本身基本上不携带信息**。导线说"发生了某事"；不说发生了什么。驱动程序通过读取设备的状态寄存器发现发生了什么。单个IRQ线可以报告许多不同事件（接收数据就绪、发送FIFO空、链路状态改变、错误等），解码状态位并决定做什么是驱动程序的工作。

### 边沿触发与电平触发

区别值得理解，因为它解释了为什么某些bug产生中断风暴、某些bug产生静默丢弃的中断、某些bug产生卡住的系统。

**边沿触发**中断在信号转换时触发一次。设备将导线拉低（对于低电平有效线）；中断控制器注意到转换；中断排队给CPU。如果设备继续将导线保持低电平，不触发额外中断，因为信号没有转换，只是继续被断言。要触发新中断，设备必须释放导线然后再次断言。

边沿触发中断高效。中断控制器只需跟踪转换，不是持续信号。缺点是脆弱：如果中断在控制器不观看时触发（例如因为另一个中断正在处理），转换可能被错过。现代中断控制器排队边沿触发中断以避免大部分这种情况，但风险是真实的，某些驱动程序（或某些设备bug）产生偶尔丢弃事件的边沿触发设置。

**电平触发**中断在信号被断言期间持续触发。只要设备将导线保持在断言电平，中断控制器报告中断。当设备释放导线时，中断控制器停止报告。CPU看到中断，驱动程序的处理程序运行，处理程序读取设备状态并清除待处理条件，设备停止断言信号，中断控制器停止报告。如果处理程序未能清除待处理条件，信号保持断言，中断控制器继续报告，驱动程序的处理程序立即再次被调用，在消耗CPU的循环中。这就是经典的**中断风暴**。

电平触发中断健壮。只要设备有要报告的，操作系统就会知道；没有可以错过事件的窗口。代价是有bug的驱动程序可以产生风暴。FreeBSD有风暴检测以缓解此问题（本章后面的附录*中断风暴检测深入*覆盖它）；其他操作系统有类似保护。常见经验法则：电平触发是更安全的默认，PCI的传统INTx线因该原因是电平触发。

区别对驱动程序作者在几个特定地方重要：

- 在电平触发线上，从处理程序返回前未能清除设备INTR_STATUS寄存器的驱动程序会产生中断风暴。在边沿触发线上，相同bug产生丢失中断而不是风暴。
- 正确读写INTR_STATUS的驱动程序在两种类型上都工作，无需特殊知识。
- 直接操作中断控制器触发模式的驱动程序（罕见；主要是遗留）必须理解区别。

对于第19章的PCI驱动程序，传统路径上的信号是电平触发INTx。在MSI和MSI-X（第20章）上，信号是消息基础的，不直接对应边沿或电平，但驱动程序模式相同：读状态、确认设备、返回。

### CPU中断处理流程，简化版

当设备的IRQ线被断言时，逐步发生了什么？现代x86系统上的简化跟踪：

1. 设备在总线上断言其IRQ（或对于启用MSI的PCIe发送MSI包）。
2. 系统的中断控制器（x86上的APIC，arm64上的GIC）接收信号并根据配置的亲和确定哪个CPU应该处理它。在多CPU系统上这是一个可引导的决策。
3. 选定CPU的中断硬件检测到待处理中断。在完成当前指令前，CPU保存足够状态（程序计数器、标志寄存器和少数其他字段）以便稍后返回被打断的工作。
4. CPU跳转到其中断描述符表中的向量。此向量的条目是一小段内核代码，称为**陷阱stub**，转换到超级用户模式，保存被打断线程的寄存器集，并调用内核的中断分发代码。
5. 内核的中断分发代码找到与此IRQ关联的`intr_event`（这是第2节覆盖的FreeBSD结构）并调用附着于其的驱动程序处理程序。
6. 驱动程序的过滤器处理程序运行。它读取设备状态寄存器，判断发生了什么类型事件，写入设备INTR_STATUS寄存器以确认事件（使设备停止断言线，对于电平触发），并返回一个值告诉内核接下来做什么。
7. 如果过滤器返回`FILTER_SCHEDULE_THREAD`，内核调度与此中断关联的ithread。ithread是内核线程，唤醒、运行驱动程序的二级处理程序、回到睡眠。
8. 所有处理程序运行后，内核向中断控制器发送中断结束（EOI）信号，重新使能IRQ线。
9. CPU从中断返回。被打断线程的寄存器集恢复，线程在到达中断时即将执行的指令处恢复。

对于简单处理程序，第3到9步在现代硬件上需要几微秒。整个流程对被打断线程不可见：不是设计为中断安全的代码（比如用户空间的浮点计算）在打断前后都正确运行，因为CPU在整个序列周围保存和恢复其状态。

从驱动程序作者的角度，第1到5步是内核的关注；第6和7步是驱动程序代码运行的地方。驱动程序的处理程序必须快（第8步的EOI等待它），必须不睡眠（被打断线程持有CPU资源），必须不获取可能间接阻塞被打断线程的锁。第2节将以FreeBSD术语精确说明这些约束；第1章现已建立了心智模型。

### 中断到达时驱动程序必须做什么

驱动程序在中断上的义务数量不大但细节不小：

1. **识别原因。**读取设备中断状态寄存器。如果没有位被设置（设备没有待处理中断），这是共享IRQ的虚假调用；返回`FILTER_STRAY`并让内核尝试线上下一个处理程序。
2. **在设备端确认中断。**将状态位写回（典型地是写1到每位，因为大多数INTR_STATUS寄存器是RW1C）使设备去断言线，中断控制器可以重新使能IRQ。并非每个设备都要求确认在过滤器内做，但在此处做是安全默认；电平触发风暴故事依赖于及时确认。
3. **决定做什么工作。**读取足够设备以决定。这是接收事件？发送完成？错误？链路改变？状态位告诉您。
4. **做紧急小工作。**更新计数器。从FIFO复制字节到队列。切换控制位。任何可以在微秒内做而无需获取睡眠锁的事情都符合此处。
5. **延迟大量工作。**如果事件触发长操作（处理接收数据包、解码数据流、向用户空间发送命令），调度ithread或taskqueue任务并返回。延迟工作在线程上下文运行，那里可以获取睡眠锁、分配内存、慢慢做。
6. **返回适当的FILTER_*值。**`FILTER_HANDLED`表示中断完全处理；不需要ithread。`FILTER_SCHEDULE_THREAD`表示ithread应该运行。`FILTER_STRAY`表示中断不是该驱动程序的。这三个值是内核用来分发进一步工作的词汇。

在每次中断正确做这六件事的驱动程序有第19章其余部分教授的形状。跳过其中任何一个的驱动程序有bug。

### 真实世界例子

第19章词汇将覆盖的事件简短巡览。

**按键。**扫描码到达时PS/2键盘控制器触发中断。驱动程序读取扫描码，传递给键盘子系统，确认。整个处理程序在几微秒内运行；taskqueue通常不需要。

**网络数据包。**数据包在接收队列累积时NIC触发中断。驱动程序的过滤器读取状态寄存器确认接收事件，调度ithread，返回。ithread遍历描述符环、构造`mbuf`数据包、向上传递给网络栈。过滤器与ithread之间的分离在此重要，因为栈处理足够慢，在过滤器中运行它会将中断窗口延长太远。

**传感器读数。**新测量就绪时I2C连接的温度传感器触发中断。驱动程序读取值、更新sysctl缓存、可选唤醒任何待处理用户空间读者、确认。简单快速。

**串口。**接收或发送FIFO空条件上UART触发中断。驱动程序排空或重新填充FIFO、更新循环缓冲区、确认。在高波特率下，这可以每秒发生数万次，所以处理程序必须紧凑。

**磁盘完成。**排队命令完成时SATA或NVMe控制器触发中断。驱动程序遍历完成队列、将每个完成匹配到待处理I/O请求、唤醒等待线程、确认。匹配和唤醒有时在过滤器与ithread之间分离。

这些设备每个都以相同方式触及第19章词汇：过滤器读状态、过滤器判断发生了什么、过滤器确认、过滤器处理或延迟。具体寄存器布局不同；模式不变。

### 快速练习：在实验室主机上找到中断驱动的设备

在移动到第2节之前，一个简短练习使硬件画面具体化。

在您的实验室主机上，运行：

```sh
vmstat -i
```

输出是启动以来中断源及其计数列表。每行大致如下：

```text
interrupt                          total       rate
cpu0:timer                      1234567        123
cpu1:timer                      1234568        123
irq9: acpi0                          42          0
irq19: uhci0+                     12345         12
irq21: ahci0                      98765         99
irq23: em0                       123456        123
```

从您自己的输出选三行。对于每个，识别：

- 中断名（IRQ号和驱动设置描述的混合）。
- 总计数（启动以来中断触发多少次）。
- 速率（每秒中断；高速率表示设备繁忙）。

十秒后第二次运行`vmstat -i`。比较计数。哪些中断活跃计数？哪些基本空闲？

现在用`devinfo -v`匹配中断到设备：

```sh
devinfo -v | grep -B 2 irq
```

每个匹配显示声明IRQ的设备。与`vmstat -i`输出交叉检查以查看哪个驱动程序由每条线服务。

在您阅读第2节时保持此输出打开。例子中的`em0`条目是Intel以太网控制器；如果您使用带FreeBSD的Intel基础系统，`em0`或`igc0`或`ix0`可能在运行第19章教授相同模式的版本。运行FreeBSD 14.3的现代NUC显示一两个 dozen中断源；服务器显示更多。您实际拥有的系统比任何图表更有趣凝视。

### 中断简史

中断是计算机架构中最古老的想法之一。原始PDP-1在1961年支持它们，作为I/O设备信号CPU而无需CPU轮询的方式。IBM 704在差不多同一时间有它们。早期分时系统使用中断来驱动调度的时钟tick和每个I/O完成。

1970年代和1980年代，个人计算机继承了模式。原始IBM PC使用8259可编程中断控制器（PIC），支持八条IRQ线；PC/AT通过级联两个PIC扩展到十五条可用线。x86指令集添加了特定中断处理指令（`CLI`、`STI`、`INT`、`IRET`），至今以扩展形式持续。

PCI引入了设备通过配置空间广播其中断的概念（第18章讨论的`INTLINE`和`INTPIN`字段）。PCIe添加了MSI和MSI-X，用内存写消息替换物理IRQ线。三者共存于现代系统；第19章的传统INTx是三者中最老且唯一共享线的。

操作系统与其一起演进。早期Unix在内核中是单体和单线程的；中断抢占正在运行的任何东西。现代内核（包括FreeBSD）有细粒度锁定、每CPU数据结构和基于ithread的延迟分发。第19章教授的处理程序纪律是那种演进的提炼：过滤器快、任务慢、默认MP安全、可共享、可调试。

知道历史不是写驱动程序必须的。但词汇（IRQ、PIC、EOI、INTx）来自历史特定点，知道词汇来源的驱动程序作者会发现更少领域神秘。

### 第1节收尾

中断是设备打断CPU当前工作、运行一小段驱动程序代码、让CPU恢复的方式。机制经过中断控制器、CPU中断分发、内核中断事件机制、最后驱动程序的处理程序。边沿触发和电平触发信号有不同的操作后果，最显眼的是电平触发线未被正确确认时形成中断风暴。

驱动程序的处理程序有六个义务：识别原因、确认设备、决定工作、做紧急部分、延迟大量部分、返回正确的`FILTER_*`值。每个在隔离时很小，聚合时要求高；第19章其余部分是关于以FreeBSD术语正确做每个。

第2节现在走FreeBSD内核中断模型：`intr_event`是什么、ithread是什么、`vmstat -i`和`devinfo -v`如何暴露内核的中断视图、模型对驱动程序处理程序施加什么约束。

## 第2节：FreeBSD中的中断

第1节建立了硬件模型。第2节介绍软件模型。内核中断机制是中断控制器和驱动程序处理程序之间的层；清晰理解它是将硬件模型转化为驱动可写概念的关键。完成第2节的读者应该能用通俗英语回答三个问题：中断触发时运行什么、什么作为延迟工作稍后运行、驱动程序必须承诺什么以使两者安全发生。

### FreeBSD中断模型一图概览

驱动程序的处理程序不是孤立运行的。它运行在使中断处理有序且可调试的内核对象小生态内。生态有三个值得预先命名的部分。

第一个是**中断事件**，由`/usr/src/sys/sys/interrupt.h`中的`struct intr_event`表示（分发代码在`/usr/src/sys/kern/kern_intr.c`）。每个IRQ线（或第20章世界的每个MSI向量）存在一个`intr_event`。它是中央协调器：持有处理程序列表（驱动程序的过滤函数和ithread函数）、人类可读名称、标志、用于风暴检测的循环计数器（`ie_count`）、警告消息速率限制器（`ie_warntm`）和CPU绑定。中断控制器向内核报告IRQ时，内核查找对应的`intr_event`并遍历其处理程序列表。杂散中断全局计数，不按事件计数；它们通过`vmstat -i`的单独会计和内核日志消息而非事件上的字段显现。

第二个是**中断处理程序**，由`struct intr_handler`表示。每个注册在`intr_event`上的处理程序存在一个`intr_handler`。单条IRQ线可以有许多处理程序（共享该线的每个驱动程序一个）。处理程序携带驱动程序提供的过滤函数（如果有）、驱动程序提供的ithread函数（如果有）、`INTR_*`标志（最重要的是`INTR_MPSAFE`）以及内核为驱动程序保留的cookie指针。

第三个是**中断线程**，通常称为**ithread**，由`struct intr_thread`表示。与事件和处理程序不同，ithread是真实内核线程，有自己的栈、自己的调度优先级和自己的`proc`结构。当过滤器返回`FILTER_SCHEDULE_THREAD`（或驱动程序注册了仅ithread处理程序而没有过滤器），ithread被调度运行。ithread然后在可使用常规睡眠互斥锁和睡眠的线程上下文中调用驱动程序的处理函数。

三者共同产生FreeBSD已使用十多年的经典两阶段中断模式：快速过滤器在主中断上下文运行做紧急工作，线程上下文处理程序稍后运行做慢工作。第19章的驱动程序将使用两个阶段。

### IRQ分配与路由

现代x86系统有不止一个中断控制器。遗留8259 PIC已被本地APIC（每个CPU一个，处理每CPU中断如本地定时器）和I/O APIC（从I/O架构接收IRQ并路由到CPU的共享单元）取代。在arm64上，等效的是通用中断控制器（GIC），带有每CPU重分发器和共享分发器。嵌入式目标有少数其他控制器。FreeBSD在`intr_pic(9)`接口后抽象这些；驱动程序作者很少直接与中断控制器交互。

驱动程序看到的是IRQ号（在传统PCI路径上，BIOS在配置空间分配的号）或向量索引（在MSI和MSI-X路径上）。驱动程序按该号请求IRQ资源，内核分配`struct resource *`，驱动程序将资源交给`bus_setup_intr(9)`附着处理程序。内核完成将处理程序连接到正确`intr_event`、配置中断控制器将IRQ路由到CPU、使能线路的工作。

从驱动程序作者角度，IRQ路由通常是黑盒。内核处理它；驱动程序看到句柄和处理程序。一个例外：在有多CPU和多设备的平台上，**亲和性**重要。在远离设备的CPU上触发中断产生缓存未命中和跨插槽流量；在靠近设备的CPU上触发中断更便宜。`bus_bind_intr(9)`让驱动程序请求特定CPU；操作者使用`cpuset -x <irq> -l <cpu>`在运行时覆盖亲和性，使用`cpuset -g -x <irq>`查询它。本章后面的附录*中断CPU亲和深入*更详细覆盖两条路径。

### SYS_RES_IRQ：中断资源

第18章介绍了三种资源类型：`SYS_RES_MEMORY`用于内存映射BAR、`SYS_RES_IOPORT`用于I/O端口BAR、以及（顺便提及）`SYS_RES_IRQ`用于中断。第3节将首次使用第三种。词汇与BAR相同：

```c
int rid = 0;                  /* 传统PCI INTx */
struct resource *irq_res;

irq_res = bus_alloc_resource_any(dev, SYS_RES_IRQ, &rid,
    RF_SHAREABLE | RF_ACTIVE);
```

关于此分配有三件事值得注意。

首先，**`rid = 0`** 是传统PCI INTx线的约定。PCI总线驱动程序将第零个IRQ资源视为设备的遗留中断，从配置空间的`PCIR_INTLINE`字段设置。对于MSI和MSI-X（第20章），rid是1、2、3等，对应分配的向量。

其次，**`RF_SHAREABLE`** 请求内核允许IRQ线与其他驱动程序共享。在传统PCI上这是常见情况：一条物理线可以服务多个设备。没有`RF_SHAREABLE`，如果另一个驱动程序已在该线上持有处理程序，分配会失败。传递`RF_SHAREABLE`不意味着您的驱动程序必须处理杂散中断；意味着必须容忍它们。第6节正是关于那种容忍。

第三，**`RF_ACTIVE`** 一步激活资源，与BAR分配一样。没有它，驱动程序需要单独调用`bus_activate_resource(9)`。第19章总是使用`RF_ACTIVE`。

成功时，返回的`struct resource *`是IRQ的句柄。不是IRQ号；内核不暴露它。驱动程序将句柄传递给`bus_setup_intr(9)`、`bus_teardown_intr(9)`和`bus_release_resource(9)`。

### 过滤器处理程序与ithread处理程序

这是第2节也是本章的概念核心。内化过滤器与ithread区别的读者可以理解阅读每个FreeBSD驱动程序的中断代码。

**过滤器处理程序** 是驱动程序注册的在主中断上下文运行的C函数。主中断上下文意味着：CPU直接从它正在做的事情跳转，保存了最少状态，过滤器运行时被打断线程的上下文仍部分保留。具体地：

- 过滤器不能睡眠。没有线程可以阻塞；内核正在分发中断。
- 过滤器不能获取睡眠互斥锁（`mtx(9)`默认是睡眠式自适应互斥锁，可能短暂自旋但最终会睡眠）。自旋互斥锁（`mtx_init`用`MTX_SPIN`）安全。
- 过滤器不能用`M_WAITOK`分配内存；可以用`M_NOWAIT`，可能失败。
- 过滤器不能调用使用以上任何的代码。

过滤器应该快（微秒），做紧急工作（读状态、确认、更新计数器），返回。内核返回值约定是：

- `FILTER_HANDLED`：中断是我的，已完全处理，不需要ithread。
- `FILTER_SCHEDULE_THREAD`：中断是我的，部分工作已做，调度ithread做剩余工作。
- `FILTER_STRAY`：中断不是我的；尝试此线上下一个处理程序。

驱动程序也可以指定`FILTER_HANDLED | FILTER_SCHEDULE_THREAD`表示"我处理了部分"和"调度线程做更多"。

**ithread处理程序** 是在线程上下文运行的另一C函数。线程在过滤器返回`FILTER_SCHEDULE_THREAD`后由内核调度，或者如果处理程序注册为仅ithread（没有过滤器），内核分发中断时自动调度ithread。

在ithread上下文，约束大大放宽：

- ithread可以短暂睡眠在互斥锁或条件变量上。
- ithread可以使用`malloc(M_WAITOK)`。
- ithread可以调用大多数使用可睡眠锁的内核API。
- ithread仍不能任意长时间睡眠（它是实时式线程），但可以做正常驱动工作。

分离让驱动程序将紧急短工作（在过滤器中）与较慢大量工作（在ithread中）分开。网络驱动程序的过滤器可能读取状态寄存器并确认；其ithread遍历接收描述符环并向上传递数据包给栈。磁盘驱动程序的过滤器可能记录哪些完成发生并确认；其ithread将完成匹配到待处理请求并唤醒等待线程。

### 何时仅使用过滤器

当中断上的每项工作都可以在主中断上下文完成时，驱动程序使用仅过滤器。例子：

- **最小测试驱动程序**，每次中断仅递增计数器，别无其他。
- **简单传感器驱动程序**，读取一个寄存器、缓存值、唤醒sysctl读取器。（如果`selwakeup`或条件变量广播需要睡眠互斥锁，则移至过滤器加ithread。）
- **定时器驱动程序**，其工作是tick某些内核内部计数器。

第19章阶段1的驱动程序仅使用过滤器：读取INTR_STATUS、确认、更新计数器、返回`FILTER_HANDLED`。这足以证明处理程序已连接。

### 何时使用过滤器加ithread

当中断需要紧急小工作后跟较慢大量工作时，驱动程序使用过滤器加ithread。例子：

- **NIC驱动程序。** 过滤器确认并标记哪些队列有事件。Ithread遍历描述符环、构建mbuf、向上传递数据包。
- **磁盘控制器。** 过滤器读取完成状态并确认。Ithread将完成匹配到I/O请求并唤醒等待者。
- **USB主控制器。** 过滤器读取状态并确认。Ithread遍历传输描述符列表并完成任何待处理URB。

第19章阶段2的驱动程序在添加模拟"工作请求"事件时移至过滤器加ithread；过滤器记录事件，延迟工作者（通过taskqueue；第14章原语）处理工作。

### 何时仅使用ithread

当每项工作必须在线程上下文完成时，驱动程序使用仅ithread。这不太常见；通常原因是驱动程序需要每次中断获取睡眠互斥锁，无法在主上下文做任何有用的事。

注册仅ithread处理程序简单：为`bus_setup_intr(9)`的过滤器参数传递`NULL`。中断触发时内核调度ithread。

第19章驱动程序不使用仅ithread；过滤器总是运行便宜。

### INTR_MPSAFE：标志承诺什么

`INTR_MPSAFE`是`bus_setup_intr(9)`标志参数中的一位。设置它向内核承诺两件事：

1. 您的处理程序自己做同步。内核不会在其周围获取Giant锁。
2. 您的处理程序可在多CPU上并发安全运行（对于被多CPU共享的处理程序，发生在MSI-X场景和某些PIC配置中）。

如果您**没有**设置`INTR_MPSAFE`，内核在调用您的处理程序前获取Giant。这是旧BSD默认值，为依赖Giant隐式保护的前SMP驱动程序保留向后兼容性。现代驱动程序总是设置`INTR_MPSAFE`。

未能设置`INTR_MPSAFE`有可见症状：`kldload`时的`dmesg` banner包含类似`myfirst0: [GIANT-LOCKED]`的行。这是内核告诉您Giant正在您的处理程序周围获取。在生产系统上，它通过单个锁串行化每个中断，这对可扩展性是灾难性的。该行是`bus_setup_intr`故意的唠叨，帮助您注意。

实际上仍依赖Giant时设置`INTR_MPSAFE`也是bug，但更安静。内核不会获取Giant，所以任何曾经被Giant串行化的代码路径不再串行化。竞争条件出现在以前没有的地方。修复不是删除`INTR_MPSAFE`（那会掩盖bug）；修复是向处理程序及其触及的代码添加正确锁定。

第19章驱动程序总是设置`INTR_MPSAFE`并依赖现有`sc->mtx`进行同步。第11章纪律延续。

### INTR_TYPE_*标志

除`INTR_MPSAFE`外，`bus_setup_intr`接受暗示中断类别的类别标志：

- `INTR_TYPE_TTY`：tty和串口设备。
- `INTR_TYPE_BIO`：块I/O（磁盘、CD-ROM）。
- `INTR_TYPE_NET`：网络。
- `INTR_TYPE_CAM`：SCSI（CAM框架）。
- `INTR_TYPE_MISC`：杂项。
- `INTR_TYPE_CLK`：时钟和定时器中断。
- `INTR_TYPE_AV`：音频和视频。

类别影响ithread调度优先级。历史上，每个类别有不同优先级；现代FreeBSD中，只有`INTR_TYPE_CLK`获得提升优先级，其余大致相等。类别仍值得正确设置，因为它流经`devinfo -v`和`vmstat -i`输出，使中断自我标识。

对于第19章驱动程序，`INTR_TYPE_MISC`适当，因为演示目标不适合任何更具体类别。第20章将使用`INTR_TYPE_NET`，一旦驱动程序开始在实验中针对NIC。

### 共享与独占中断

在传统PCI上，多个设备可以共享单条INTx线。内核用两个资源标志跟踪：

- `RF_SHAREABLE`：此驱动程序愿意与其他驱动程序共享线路。
- `RF_SHAREABLE`缺席：此驱动程序想要线路独占；如果另一个驱动程序已持有则分配失败。

想要共享中断的驱动程序在其`bus_alloc_resource_any`调用中使用`RF_SHAREABLE | RF_ACTIVE`。想要独占访问的驱动程序（可能出于延迟原因）单独使用`RF_ACTIVE`，但请求在拥挤系统上可能失败。

内核从不阻止驱动程序共享；它阻止另一个驱动程序加入如果一个驱动程序请求独占。在现代带MSI-X的PCIe上，共享不太常见，因为每个设备有自己的消息信号向量。

第19章驱动程序设置`RF_SHAREABLE`，因为bhyve中virtio-rnd可能或可能不与其他bhyve模拟设备共享线路，取决于插槽拓扑。可共享是安全默认。

通过`bus_setup_intr`标志字段传递的`INTR_EXCL`标志（不要与资源分配标志混淆）是相关但不同的概念：它请求总线在中断事件级别给处理程序独占访问。传统PCI驱动程序很少需要它。某些总线驱动程序内部使用它。对于第19章驱动程序，我们不设置`INTR_EXCL`。

### vmstat -i显示什么

`vmstat -i`打印内核中断计数器。每行对应一个`intr_event`。列是：

- **interrupt**：人类可读标识符。对于硬件中断，名称派生自IRQ号和驱动程序描述。当使用MSI-X向量时出现`devinfo -v`风格名称（如`em0:rx 0`）。
- **total**：启动以来此中断触发次数。
- **rate**：每秒中断率，在最近窗口上平均。

一些解释说明。对于空闲设备快速增长的总计列是红旗（中断风暴）。对于应该处理流量的设备为零的速率列表明处理程序未正确连接。当多个设备共享传统INTx线时，`vmstat -i`每个`intr_event`（每个IRQ源）显示一行，该线上的驱动程序名称是第一个注册处理程序的描述；共享该线的其他驱动程序没有自己的行。当设备有自己的MSI或MSI-X向量时，每个向量是其自己的`intr_event`，每个有自己的行。每CPU中断如本地定时器显示为不同的每CPU行（`cpu0:timer`、`cpu1:timer`），因为内核为它们每个CPU创建一个事件。

内核通过`sysctl hw.intrcnt`和`sysctl hw.intrnames`暴露相同计数器，这是`vmstat -i`格式化的原始数据。驱动程序作者很少直接读取这些；`vmstat -i`是友好视图。

### devinfo -v关于中断显示什么

`devinfo -v`遍历newbus树并打印每个设备及其资源。对于有中断的PCI驱动程序，资源列表在`memory:`条目旁边包含`irq:`条目：

```text
myfirst0
    pnpinfo vendor=0x1af4 device=0x1005 ...
    resources:
        memory: 0xc1000000-0xc100001f
        irq: 19
```

`irq:`后的数字是内核IRQ标识符。在x86上它常是I/O APIC引脚号；在arm64上是GIC向量；具体含义是平台特定的，但数字在同一系统重启间稳定。

将`irq: 19`匹配到`vmstat -i`的`irq19: `条目确认驱动程序附着到预期中断线。

对于MSI-X中断（第20章），每个向量有自己的`irq:`条目，`devinfo -v`单独列出。

### 简单中断路径图

放在一起，这是从设备到驱动程序发生的事情：

```text
  设备        IRQ线          中断          CPU        intr_event         处理程序
 --------     -----------       -控制器-      --------   --------------      ---------
   |              |                  |                |             |                 |
   | 断言        |                  |                |             |                 |
   | IRQ线       | 信号             |                |             |                 |
   |------------>|                  |                |             |                 |
   |             | 锁存             |                |             |                 |
   |             |----------------->|                |             |                 |
   |             |                  | 转向CPU        |             |                 |
   |             |                  |--------------->|             |                 |
   |             |                  |                | 保存状态    |                 |
   |             |                  |                | 跳转向量    |                 |
   |             |                  |                |             | 查找            |
   |             |                  |                |------------>|                 |
   |             |                  |                |             | 每个            |
   |             |                  |                |             | 处理程序        |
   |             |                  |                |             |---------------->|
   |             |                  |                |             |                 | 过滤器运行
   |             |                  |                |             |<----------------|
   |             |                  |                |             | FILTER_HANDLED  |
   |             |                  |                |             | 或              |
   |             |                  |                |             | FILTER_SCHEDULE |
   |             |                  |                | EOI         |                 |
   |             |                  |<---------------|             |                 |
   |             |                  |                | 恢复        |                 |
   |             |                  |                | 状态        |                 |
   |             |                  |                | 恢复线程                        |
   |             |                  |                |             | ithread唤醒     |
   |             |                  |                |             | (如已调度)       |
   |             |                  |                |             |                 | ithread运行
   |             |                  |                |             |                 | 慢工作
```

图表略去几个细节（中断合并、为ithread栈交换被打断线程的栈、ithread自己的调度），但捕获形状。过滤器在中断上下文运行，ithread（如已调度）稍后在线程上下文运行，EOI在过滤器完成后发生，被打断线程在CPU空闲后恢复。

### 处理程序可能做什么的约束

过滤器处理程序可以和不可以做的简短汇总列表。这是第19章最常参考的列表；标记以供回访。

**过滤器处理程序可以：**

- 通过访问器层读写设备寄存器。
- 获取自旋互斥锁（用`MTX_SPIN`初始化的`struct mtx`）。
- 读取仅由自旋锁保护的softc字段。
- 调用内核原子操作（`atomic_add_int`等）。
- 调用`taskqueue_enqueue(9)`调度线程上下文工作。
- 如果上下文允许（大多数允许），调用`wakeup_one(9)`唤醒在通道上睡眠的线程。
- 返回`FILTER_HANDLED`、`FILTER_SCHEDULE_THREAD`、`FILTER_STRAY`或组合。

**过滤器处理程序不可以：**

- 获取睡眠互斥锁（默认初始化的`struct mtx`、`struct sx`、`struct rwlock`）。
- 调用任何可能睡眠的函数：`malloc(M_WAITOK)`、`tsleep`、`pause`、`cv_wait`等。
- 获取Giant。
- 调用可能间接做以上任何的代码。
- 花费长时间（微秒可以；毫秒是bug）。

**ithread处理程序可以：**

- 过滤器处理程序可以的一切，加上：
- 获取睡眠互斥锁、sx锁、rwlock。
- 调用`malloc(M_WAITOK)`。
- 调用`cv_wait`、`tsleep`、`pause`和其他阻塞原语。
- 花更长时间（几十或几百微秒正常）。
- 做有界工作且完成时间不可预测。

**ithread处理程序不应该：**

- 任意长时间睡眠。ithread有假设响应性的调度优先级；睡眠秒级的处理程序饥饿同一ithread上的其他工作。
- 阻塞ithread等待无界的各种外部事件。

第19章的过滤器严格遵守第一个列表；任何违规是调试内核常能捕获的bug。

### 关于每CPU与共享ithread的说明

对于传统PCI INTx线，内核通常每个`intr_event`分配一个ithread，在该事件上的任何处理程序间共享。对于MSI-X（第20章），每个向量有自己的ithread。当多个处理程序需要在同一IRQ上并发运行时差异重要：在共享ithread上，它们串行化；在独立MSI-X向量上，它们可以并行运行。

第19章驱动程序使用传统PCI。一个IRQ、一个ithread（如果有ithread）、一个延迟工作队列。串行化通常是单设备驱动程序想要的。

### 第2节收尾

FreeBSD中断模型围绕三个对象：`intr_event`（每IRQ线或MSI向量一个）、`intr_handler`（该事件上每个注册驱动程序一个）、ithread（每事件一个，处理程序间共享）。驱动程序通过`bus_setup_intr(9)`注册过滤函数、ithread函数或两者，并通过flags参数承诺`INTR_MPSAFE`合规。内核在主中断上下文分发过滤器处理程序，在过滤器返回后调度ithread。

过滤器处理程序的约束严格（无睡眠、无睡眠锁、无慢调用）；ithread处理程序的约束相比之下宽松。共享PCI INTx线允许一条IRQ上有许多驱动程序，所以过滤器必须判断中断是否属于其设备，如果不是则返回`FILTER_STRAY`。`vmstat -i`和`devinfo -v`暴露内核视图，使操作者和驱动程序作者可以看到正在发生什么。

第3节是驱动程序终于根据此模型编写代码的地方。它用`SYS_RES_IRQ`分配IRQ资源、通过`bus_setup_intr(9)`注册过滤器处理程序、设置`INTR_MPSAFE`，并在每次调用时记录短消息。阶段1是驱动程序第一次真正被中断打断。
## 第3节：注册中断处理程序

第1节和第2节建立了硬件和内核模型。第3节让驱动程序开始工作。任务范围窄：扩展第18章的attach路径，使其在分配BAR并遍历能力列表后，还分配IRQ资源、注册过滤器处理程序、设置`INTR_MPSAFE`。detach路径反向镜像增长：先拆除处理程序，然后释放IRQ。到第3节结束时，驱动程序处于版本`1.2-intr-stage1`，每次IRQ线被断言时触发小型计数器递增过滤器处理程序。

### 阶段1产生什么

阶段1的处理程序刻意最小化。驱动程序需要在每个形式意义上都正确（返回正确`FILTER_*`值、遵守"无睡眠"规则、`INTR_MPSAFE`）但尚不做任何真实工作的过滤器。目标是在引入状态解码和延迟工作复杂性之前证明处理程序已正确连接。

阶段1处理程序的行为：

1. 获取自旋安全计数器锁（在我们的例子中是简单原子操作）。
2. 递增softc中的计数器。
3. 返回`FILTER_HANDLED`。

就这些。无状态寄存器读取、无确认、无延迟工作。计数器让读者观察处理程序是否触发，以及触发频率。`dmesg`输出默认静默；计数通过阶段暴露的sysctl可见。

第4节添加真实工作（状态解码、确认、ithread调度）。第5节添加模拟中断用于测试。第6节扩展过滤器以通过检查中断是否真正属于我们的设备来成为共享IRQ安全。但脚手架是此阶段的贡献，需要在任何其他东西落在上面之前正确。

### IRQ资源分配

attach的第一行新代码，紧跟BAR分配之后：

```c
sc->irq_rid = 0;   /* 传统PCI INTx */
sc->irq_res = bus_alloc_resource_any(dev, SYS_RES_IRQ, &sc->irq_rid,
    RF_SHAREABLE | RF_ACTIVE);
if (sc->irq_res == NULL) {
    device_printf(dev, "cannot allocate IRQ\n");
    error = ENXIO;
    goto fail_hw;
}
```

几点值得注意。

首先，`rid = 0`是传统PCI INTx约定。每个PCI设备有单条遗留IRQ线，通过配置空间的`PCIR_INTLINE`和`PCIR_INTPIN`字段广播；PCI总线驱动程序将其作为资源rid 0暴露。第20章将对MSI和MSI-X向量使用非零rid，但第19章驱动程序使用遗留路径。

其次，如果内核选择了与请求不同的rid，`rid`变量会被`bus_alloc_resource_any`更新。对于`rid = 0`，内核总是返回`rid = 0`，所以更新是空操作，但模式与第18章BAR分配一致。

第三，`RF_SHAREABLE | RF_ACTIVE`是标准标志集。`RF_SHAREABLE`允许内核将我们的处理程序放在与其他驱动程序共享的`intr_event`上。`RF_ACTIVE`一步激活资源。

第四，分配可能失败。真实系统上最常见原因是设备的PCI配置空间中断字段为零（固件未向设备路由中断）。在带virtio-rnd设备的bhyve上，分配通常成功；在某些旧QEMU配置用`intx=off`时可能失败。如果分配失败，attach路径通过goto级联展开。

### 在Softc中存储资源

softc获得三个新字段：

```c
struct myfirst_softc {
    /* ... 现有字段 ... */

    /* 第19章中断字段。 */
    struct resource	*irq_res;
    int		 irq_rid;
    void		*intr_cookie;     /* 用于bus_teardown_intr */
    uint64_t	 intr_count;      /* 处理程序调用计数 */
};
```

`irq_res`是声称IRQ资源的句柄。`irq_rid`是资源ID（用于匹配释放调用）。`intr_cookie`是`bus_setup_intr(9)`返回和`bus_teardown_intr(9)`消耗的不透明cookie；它标识特定处理程序以便内核稍后可以干净地删除它。`intr_count`是阶段1处理程序每次调用递增的诊断计数器。

三个字段与第18章添加的三个BAR字段（`bar_res`、`bar_rid`、`pci_attached`）平行。平行并非偶然：每个资源类获得句柄、ID和驱动程序需要的任何簿记。

### 过滤器处理程序的签名

驱动程序的过滤器是签名为的函数：

```c
static int myfirst_intr_filter(void *arg);
```

参数是驱动程序传递给`bus_setup_intr`的`arg`参数的任何指针；按约定，是指向驱动程序softc的指针。返回值是`FILTER_STRAY`、`FILTER_HANDLED`和`FILTER_SCHEDULE_THREAD`的按位或，如第2节所述。

阶段1实现：

```c
static int
myfirst_intr_filter(void *arg)
{
    struct myfirst_softc *sc = arg;

    atomic_add_64(&sc->intr_count, 1);
    return (FILTER_HANDLED);
}
```

一行真实工作。计数器原子递增，因为处理程序可能同时在多CPU上运行（MSI-X场景）或与通过sysctl读取计数器的非中断代码并行。睡眠锁在过滤器中是错误的；原子操作是此处安全的轻量原语。

返回值是`FILTER_HANDLED`因为我们没有ithread工作要做，也没有理由返回`FILTER_STRAY`（第6节添加杂散检查；阶段1假设IRQ是我们的）。

### 通过bus_setup_intr注册处理程序

IRQ分配后，驱动程序调用`bus_setup_intr(9)`：

```c
error = bus_setup_intr(dev, sc->irq_res,
    INTR_TYPE_MISC | INTR_MPSAFE,
    myfirst_intr_filter, NULL, sc,
    &sc->intr_cookie);
if (error != 0) {
    device_printf(dev, "bus_setup_intr failed (%d)\n", error);
    goto fail_release_irq;
}
```

七个参数：

1. **`dev`**：设备句柄。
2. **`sc->irq_res`**：我们刚分配的IRQ资源。
3. **`INTR_TYPE_MISC | INTR_MPSAFE`**：标志。`INTR_TYPE_MISC`分类中断（第2节）。`INTR_MPSAFE`承诺处理程序自己做同步。
4. **`myfirst_intr_filter`**：我们的过滤器处理程序。非NULL。
5. **`NULL`**：ithread处理程序。NULL因为阶段1仅使用过滤器。
6. **`sc`**：传递给两个处理程序的参数。
7. **`&sc->intr_cookie`**：内核存储稍后拆除用的cookie的输出参数。

返回值成功为0，失败为errno。此时失败罕见；最常见原因是中断控制器或平台特定限制。

成功的`bus_setup_intr`结合下面的`device_printf`在驱动程序加载时在`dmesg`中产生短banner：

```text
myfirst0: attached filter handler on IRQ resource
```

IRQ号本身不在此行；`devinfo -v`和`vmstat -i`显示它（IRQ号取决于guest配置）。如果您看到额外的`myfirst0: [GIANT-LOCKED]`行，您的flags参数缺少`INTR_MPSAFE`，内核正在警告Giant正在处理程序周围获取；修复它。

### 向devinfo描述处理程序

可选但推荐的步骤。`bus_describe_intr(9)`让驱动程序向处理程序附加人类可读名称，`devinfo -v`和内核诊断将使用：

```c
bus_describe_intr(dev, sc->irq_res, sc->intr_cookie, "legacy");
```

此调用后，`vmstat -i`显示处理程序行为`irq19: myfirst0:legacy`而不是简单的`irq19: myfirst0`。后缀是驱动程序提供的名称。对于第19章单中断驱动程序，后缀主要是装饰性；对于第20章有多个向量的MSI-X驱动程序，区分`rx0`、`rx1`、`tx0`、`admin`等变得至关重要。

### 扩展的Attach级联

将新片段放入第18章阶段3 attach：

```c
static int
myfirst_pci_attach(device_t dev)
{
    struct myfirst_softc *sc = device_get_softc(dev);
    int error, capreg;

    sc->dev = dev;
    sc->unit = device_get_unit(dev);
    error = myfirst_init_softc(sc);
    if (error != 0)
        return (error);

    /* 步骤1：分配BAR0。 */
    sc->bar_rid = PCIR_BAR(0);
    sc->bar_res = bus_alloc_resource_any(dev, SYS_RES_MEMORY,
        &sc->bar_rid, RF_ACTIVE);
    if (sc->bar_res == NULL) {
        device_printf(dev, "cannot allocate BAR0\n");
        error = ENXIO;
        goto fail_softc;
    }

    /* 步骤2：遍历PCI能力（信息性）。 */
    if (pci_find_cap(dev, PCIY_EXPRESS, &capreg) == 0)
        device_printf(dev, "PCIe capability at 0x%x\n", capreg);
    if (pci_find_cap(dev, PCIY_MSI, &capreg) == 0)
        device_printf(dev, "MSI capability at 0x%x\n", capreg);
    if (pci_find_cap(dev, PCIY_MSIX, &capreg) == 0)
        device_printf(dev, "MSI-X capability at 0x%x\n", capreg);

    /* 步骤3：将硬件层附着到BAR。 */
    error = myfirst_hw_attach_pci(sc, sc->bar_res,
        rman_get_size(sc->bar_res));
    if (error != 0)
        goto fail_release_bar;

    /* 步骤4：分配IRQ。 */
    sc->irq_rid = 0;
    sc->irq_res = bus_alloc_resource_any(dev, SYS_RES_IRQ,
        &sc->irq_rid, RF_SHAREABLE | RF_ACTIVE);
    if (sc->irq_res == NULL) {
        device_printf(dev, "cannot allocate IRQ\n");
        error = ENXIO;
        goto fail_hw;
    }

    /* 步骤5：注册过滤器处理程序。 */
    error = bus_setup_intr(dev, sc->irq_res,
        INTR_TYPE_MISC | INTR_MPSAFE,
        myfirst_intr_filter, NULL, sc,
        &sc->intr_cookie);
    if (error != 0) {
        device_printf(dev, "bus_setup_intr failed (%d)\n", error);
        goto fail_release_irq;
    }
    bus_describe_intr(dev, sc->irq_res, sc->intr_cookie, "legacy");
    device_printf(dev, "attached filter handler on IRQ resource\n");

    /* 步骤6：创建cdev。 */
    sc->cdev = make_dev(&myfirst_cdevsw, sc->unit, UID_ROOT,
        GID_WHEEL, 0600, "myfirst%d", sc->unit);
    if (sc->cdev == NULL) {
        error = ENXIO;
        goto fail_teardown_intr;
    }
    sc->cdev->si_drv1 = sc;

    /* 步骤7：从BAR读取诊断字。 */
    MYFIRST_LOCK(sc);
    sc->bar_first_word = CSR_READ_4(sc, 0x00);
    MYFIRST_UNLOCK(sc);
    device_printf(dev, "BAR[0x00] = 0x%08x\n", sc->bar_first_word);

    sc->pci_attached = true;
    return (0);

fail_teardown_intr:
    bus_teardown_intr(dev, sc->irq_res, sc->intr_cookie);
    sc->intr_cookie = NULL;
fail_release_irq:
    bus_release_resource(dev, SYS_RES_IRQ, sc->irq_rid, sc->irq_res);
    sc->irq_res = NULL;
fail_hw:
    myfirst_hw_detach(sc);
fail_release_bar:
    bus_release_resource(dev, SYS_RES_MEMORY, sc->bar_rid, sc->bar_res);
    sc->bar_res = NULL;
fail_softc:
    myfirst_deinit_softc(sc);
    return (error);
}
```

attach序列现在有七步而不是五步。两个新goto标签（`fail_teardown_intr`、`fail_release_irq`）扩展级联。模式与第18章相同：每步撤销它之前的一步，向下链到softc init。

### 扩展的Detach

detach路径镜像attach，中断拆除位于cdev拆除和硬件层分离之间：

```c
static int
myfirst_pci_detach(device_t dev)
{
    struct myfirst_softc *sc = device_get_softc(dev);

    if (myfirst_is_busy(sc))
        return (EBUSY);

    sc->pci_attached = false;

    /* 销毁cdev使无新用户空间访问开始。 */
    if (sc->cdev != NULL) {
        destroy_dev(sc->cdev);
        sc->cdev = NULL;
    }

    /* 在任何它依赖的东西之前拆除中断处理程序。 */
    if (sc->intr_cookie != NULL) {
        bus_teardown_intr(dev, sc->irq_res, sc->intr_cookie);
        sc->intr_cookie = NULL;
    }

    /* 静默callout和任务（包括第17章模拟如已附着；
     * 包括任何延迟taskqueue工作）。 */
    myfirst_quiesce(sc);

    /* 释放第17章模拟如已附着。 */
    if (sc->sim != NULL)
        myfirst_sim_detach(sc);

    /* 分离硬件层。 */
    myfirst_hw_detach(sc);

    /* 释放IRQ资源。 */
    if (sc->irq_res != NULL) {
        bus_release_resource(dev, SYS_RES_IRQ, sc->irq_rid,
            sc->irq_res);
        sc->irq_res = NULL;
    }

    /* 释放BAR。 */
    if (sc->bar_res != NULL) {
        bus_release_resource(dev, SYS_RES_MEMORY, sc->bar_rid,
            sc->bar_res);
        sc->bar_res = NULL;
    }

    myfirst_deinit_softc(sc);

    device_printf(dev, "detached\n");
    return (0);
}
```

与第18章两个变化：`bus_teardown_intr`调用和`bus_release_resource(..., SYS_RES_IRQ, ...)`调用。顺序重要。`bus_teardown_intr`必须在处理程序读取或写入的任何东西被释放之前发生；特别是，在`myfirst_hw_detach`（释放`sc->hw`）之前。`bus_teardown_intr`返回后，内核保证处理程序不在运行且不会被再次调用；驱动程序然后可以释放处理程序触及的任何东西。

释放IRQ资源发生在拆除和硬件层分离之后。硬件分离和BAR释放之间的确切位置是判断调用：BAR和IRQ不相互依赖，所以任一顺序都可以。第19章驱动程序先释放IRQ，因为这是attach顺序的反向（attach在BAR后分配IRQ；detach在BAR前释放它）。

第7节更多说明顺序。

### 中断计数器的sysctl

小型诊断：暴露`intr_count`字段的sysctl使读者可以观看计数器增长：

```c
SYSCTL_ADD_U64(&sc->sysctl_ctx,
    SYSCTL_CHILDREN(sc->sysctl_tree), OID_AUTO, "intr_count",
    CTLFLAG_RD, &sc->intr_count, 0,
    "Number of times the interrupt filter has run");
```

加载后，`sysctl dev.myfirst.0.intr_count`返回当前计数。对于没有中断触发的virtio-rnd设备（设备尚未有东西要信号），计数保持为零。第5节的模拟中断将在无需真实IRQ事件的情况下驱动计数增长。

`sysctl`可被任何用户读取（`CTLFLAG_RD`使其在sysctl级别世界可读；sysctl MIB上的文件权限在其他地方设置）。访问通过：

```sh
sysctl dev.myfirst.0.intr_count
```

### 阶段1证明了什么

在带virtio-rnd设备的bhyve guest上加载阶段1驱动程序产生：

```text
myfirst0: <Red Hat Virtio entropy source (myfirst demo target)> ... on pci0
myfirst0: attaching: vendor=0x1af4 device=0x1005 revid=0x00
myfirst0: BAR0 allocated: 0x20 bytes at 0xc1000000
myfirst0: hardware layer attached to BAR: 32 bytes
myfirst0: attached filter handler on IRQ resource
myfirst0: BAR[0x00] = 0x10010000
```

`vmstat -i | grep myfirst`显示内核创建的中断事件：

```text
irq19: myfirst0:legacy              0          0
```

（IRQ号和速率取决于环境。）

初始计数为零，因为virtio-rnd设备尚未产生中断（我们还未编程它产生）。驱动程序已正确连接，处理程序已注册，过滤器准备触发。阶段1的工作完成。

### 阶段1未做什么

几件事刻意从阶段1缺席，出现在后续阶段：

- **状态寄存器读取。** 过滤器不读取设备的INTR_STATUS；只是递增计数器。第4节添加状态读取。
- **确认。** 过滤器不写入INTR_STATUS确认。在设备实际触发的电平触发线上，这是bug。在我们bhyve上的virtio-rnd目标上，设备不触发，所以缺席不可见。第4节添加确认并解释为何重要。
- **ithread处理程序。** 尚无延迟工作。第4节引入基于taskqueue的延迟路径并连接过滤器调度它。
- **模拟中断。** 没有方式让处理程序在没有来自设备真实IRQ的情况下触发。第5节添加在驱动程序正常锁规则下直接调用过滤器的sysctl。
- **共享IRQ纪律。** 过滤器假设每个中断属于我们的设备。第6节添加对共享线设备的`FILTER_STRAY`检查。

这些是第4、5和6节的话题。阶段1刻意不完整；每个后续节添加阶段1省略的特定内容。

### 此阶段常见错误

初学者在阶段1遇到的陷阱简短列表。

**忘记`INTR_MPSAFE`。** 处理程序被Giant包装。可扩展性消失。`dmesg`打印`[GIANT-LOCKED]`。修复：向flags参数添加`INTR_MPSAFE`。

**向过滤器传递错误参数。** C函数指针挑剔；传递`&sc`而不是`sc`产生过滤器然后错误解引用的双重指针。结果通常是内核崩溃。修复：`bus_setup_intr`中的`arg`是`sc`；过滤器接收相同值作为`void *`。

**从过滤器返回0。** 返回值是`FILTER_*`值的按位或。零是"无标志"，非法（内核要求至少一个`FILTER_STRAY`、`FILTER_HANDLED`或`FILTER_SCHEDULE_THREAD`）。调试内核在此断言。修复：返回`FILTER_HANDLED`。

**在过滤器中使用睡眠锁。** 过滤器获取`sc->mtx`（常规睡眠互斥锁）。`WITNESS`抱怨；调试内核崩溃。修复：使用原子操作，或将工作移至ithread。

**在消耗cookie前拆除IRQ。** 在`bus_teardown_intr`前对IRQ调用`bus_release_resource`是bug：资源没了，但处理程序仍注册在其上。下一次中断触发，内核解引用已释放状态。修复：总是先`bus_teardown_intr`。

**rid不匹配。** 传递给`bus_release_resource`的rid必须匹配`bus_alloc_resource_any`返回的rid（或最初传入的rid，对于`rid = 0`）。不匹配常表现为"Resource not found"或内核消息。修复：在softc中与资源句柄一起存储rid。

**忘记在拆除前排空待处理延迟工作。** 这更多适用于阶段2，但值得在此标记：如果过滤器已调度taskqueue项目，项目必须在softc消失前完成。释放IRQ但留下待处理taskqueue项目的拆除产生项目运行时的释放后使用。

### 检查点：阶段1工作

在第4节前，确认阶段1就位：

- `kldstat -v | grep myfirst`显示驱动程序版本`1.2-intr-stage1`。
- `dmesg | grep myfirst`显示包含`attached filter handler on IRQ resource`的attach banner。
- 无`[GIANT-LOCKED]`警告。
- `devinfo -v | grep -A 5 myfirst`显示BAR和IRQ资源。
- `vmstat -i | grep myfirst`显示处理程序行。
- `sysctl dev.myfirst.0.intr_count`返回`0`（或小数字，取决于设备是否碰巧中断）。
- `kldunload myfirst`干净运行；无崩溃、无警告。

如果任何步骤失败，返回相关小节。失败诊断方式与第18章失败相同：检查`dmesg`的banner，检查`devinfo -v`的资源，检查`WITNESS`输出的锁顺序问题。

### 第3节收尾

注册中断处理程序是三个新调用（`bus_alloc_resource_any`配合`SYS_RES_IRQ`、`bus_setup_intr`、`bus_describe_intr`）、三个新softc字段（`irq_res`、`irq_rid`、`intr_cookie`）和一个新计数器（`intr_count`）。attach级联增长两个标签；detach路径增长`bus_teardown_intr`调用。处理程序本身是一行原子递增返回`FILTER_HANDLED`。

阶段1的意义不是处理程序做的工作。意义是处理程序已正确注册、`INTR_MPSAFE`已设置、中断触发时计数器递增、拆除在卸载时干净运行。每个后续阶段建立在此脚手架之上；现在正确是贯穿本章剩余部分的投资回报。

第4节使处理程序做真实工作：读取INTR_STATUS、判断做什么、确认设备、延迟大量工作到taskqueue。这是真实中断处理程序的核心，是第4部分其余部分最重要的内容。

## 第4节：编写真实中断处理程序

阶段1证明了处理程序连接正确。阶段2使处理程序做真实驱动程序过滤器做的工作。第4节的结构是仔细走一遍第17章模拟的硬件模型，过滤器现在读取并确认设备的`INTR_STATUS`寄存器、根据哪些位被设置做出决策、内联处理紧急小工作、延迟大量工作到taskqueue。到第4节结束时，驱动程序处于版本`1.2-intr-stage2`，有过滤器加任务管道，行为像小型真实驱动程序。

### 寄存器画面

快速回顾第17章模拟的中断寄存器布局（完整细节见`HARDWARE.md`）。偏移`0x14`持有`INTR_STATUS`，一个32位寄存器，定义这些位：

- `MYFIRST_INTR_DATA_AV`（`0x00000001`）：数据可用事件已发生。
- `MYFIRST_INTR_ERROR`（`0x00000002`）：错误条件已检测。
- `MYFIRST_INTR_COMPLETE`（`0x00000004`）：命令已完成。

寄存器是"写一清除"（RW1C）语义：向位写1清除该位；写0保持不变。这是标准PCI中断状态约定，也是第19章处理程序期望的。

偏移`0x10`持有`INTR_MASK`，一个并行寄存器，控制`INTR_STATUS`的哪些位实际断言IRQ线。在`INTR_MASK`中设置位启用该中断类；清除它禁用。驱动程序在attach时设置`INTR_MASK`以启用想要接收的中断。

第17章模拟可以自主驱动这些位。第18章PCI驱动程序运行在真实virtio-rnd BAR上，那里的偏移意指不同东西（virtio遗留配置，不是第17章寄存器映射）。第4节针对第17章语义编写处理程序；第5节展示如何在没有真实IRQ事件的情况下演练处理程序；virtio-rnd设备不实现此寄存器布局，所以在bhyve实验室中处理程序主要通过模拟中断路径演练。

这是教学目标的诚实限制。将驱动程序适配到实现第17章风格寄存器的真实设备的读者会看到处理程序在真实中断上直接触发。对于bhyve virtio-rnd目标，第5节的sysctl触发处理程序是在实践中演练阶段2过滤器的方式。

### 阶段2的过滤器处理程序

阶段2过滤器读取`INTR_STATUS`、判断发生了什么、确认处理的位、要么内联做紧急工作要么调度任务做大量工作。

```c
int
myfirst_intr_filter(void *arg)
{
    struct myfirst_softc *sc = arg;
    uint32_t status;
    int rv = 0;

    /*
     * 读取原始状态。过滤器在主中断上下文运行，
     * 不能获取sc->mtx（睡眠互斥锁），所以访问通过
     * 断言正确上下文的专门访问器。我们使用本地、
     * 自旋安全的helper用于BAR访问；阶段2使用小内联
     * 而不是锁断言的CSR_READ_4宏。
     */
    status = bus_read_4(sc->bar_res, MYFIRST_REG_INTR_STATUS);
    if (status == 0)
        return (FILTER_STRAY);

    atomic_add_64(&sc->intr_count, 1);

    /* 处理DATA_AV位：仅小紧急工作。 */
    if (status & MYFIRST_INTR_DATA_AV) {
        atomic_add_64(&sc->intr_data_av_count, 1);
        bus_write_4(sc->bar_res, MYFIRST_REG_INTR_STATUS,
            MYFIRST_INTR_DATA_AV);
        taskqueue_enqueue(sc->intr_tq, &sc->intr_data_task);
        rv |= FILTER_HANDLED;
    }

    /* 处理ERROR位：记录并确认。 */
    if (status & MYFIRST_INTR_ERROR) {
        atomic_add_64(&sc->intr_error_count, 1);
        bus_write_4(sc->bar_res, MYFIRST_REG_INTR_STATUS,
            MYFIRST_INTR_ERROR);
        rv |= FILTER_HANDLED;
    }

    /* 处理COMPLETE位：唤醒任何待处理等待者。 */
    if (status & MYFIRST_INTR_COMPLETE) {
        atomic_add_64(&sc->intr_complete_count, 1);
        bus_write_4(sc->bar_res, MYFIRST_REG_INTR_STATUS,
            MYFIRST_INTR_COMPLETE);
        rv |= FILTER_HANDLED;
    }

    /* 如果我们没识别任何位，这不是我们的中断。 */
    if (rv == 0)
        return (FILTER_STRAY);

    return (rv);
}
```

几件事值得仔细阅读。

**原始访问。** 过滤器直接使用`bus_read_4`和`bus_write_4`（较新的基于资源的访问器），不是第16章的`CSR_READ_4`和`CSR_WRITE_4`宏。原因是微妙的。第16章宏通过`MYFIRST_ASSERT`获取`sc->mtx`，这是睡眠互斥锁。过滤器绝不能获取睡眠互斥锁。正确方法是直接使用原始`bus_space`访问器（如所示）或引入不断言锁要求的并行CSR宏家族。第8节的重构引入`ICSR_READ_4`和`ICSR_WRITE_4`（"I"表示中断上下文）使区别明确；阶段2使用原始访问器。

**早期杂散检查。** 状态为零表示没有位被设置；这是来自另一驱动程序的共享IRQ调用。返回`FILTER_STRAY`让内核尝试线上下一个处理程序。检查也是对真实硬件竞争的防御：如果中断控制器断言线但设备已清除状态（在我们读取它时），我们不应声称中断。

**每位处理。** 每个感兴趣的位被检查、计数、确认。顺序不重要（位独立），但结构是常规的：每位一个`if`。

**确认。** 将位写回`INTR_STATUS`清除它（RW1C）。这是使中断线去断言的原因。在电平触发线上未能确认产生中断风暴。

**taskqueue入队。** `DATA_AV`位触发延迟工作。过滤器入队任务；taskqueue的工作线程稍后在线程上下文运行任务，那里可以获取睡眠锁做慢工作。从过滤器调用入队是安全的（taskqueue内部为此路径使用自旋锁）。

**最终返回值。** 我们识别的每位的`FILTER_HANDLED`按位或，如果没有匹配则`FILTER_STRAY`。如果我们有ithread工作，我们会OR进`FILTER_SCHEDULE_THREAD`；但阶段2使用taskqueue而不是ithread，所以返回值只是`FILTER_HANDLED`。

### 为什么用taskqueue而不是ithread？

FreeBSD允许驱动程序通过`bus_setup_intr(9)`的第五个参数注册ithread处理程序。为什么阶段2用taskqueue代替？

两个原因。

首先，taskqueue更灵活。ithread与特定`intr_event`绑定；它在过滤器后运行驱动程序的ithread函数。taskqueue让驱动程序从任意上下文（过滤器、ithread、其他任务、用户空间ioctl路径）调度任务，并在共享工作线程上运行。对于第19章驱动程序，通过模拟中断和真实中断演练处理程序，taskqueue是更统一的延迟工作原语。

其次，taskqueue将优先级与中断类型分离。ithread优先级派生自`INTR_TYPE_*`；taskqueue优先级由`taskqueue_start_threads(9)`控制。对于希望延迟工作处于中断类别暗示的不同优先级的驱动程序，taskqueue给予那种控制。

真实FreeBSD驱动程序使用两种模式。具有简单即发即忘中断的简单驱动程序常用ithread（更少代码）。具有更丰富延迟工作模式的驱动程序使用taskqueue。`iflib(9)`框架使用一种混合。

第19章教授taskqueue模式因为它与书其余部分组合更好。第17章已有taskqueue；第14章引入模式；延迟工作纪律是全书范围的主题。

### 延迟工作任务

过滤器在看到`DATA_AV`时入队了`sc->intr_data_task`。该任务是：

```c
static void
myfirst_intr_data_task_fn(void *arg, int npending)
{
    struct myfirst_softc *sc = arg;

    MYFIRST_LOCK(sc);

    /*
     * 数据可用事件已触发。通过第16章访问器读取
     * 设备数据寄存器（隐式获取sc->mtx），更新
     * 驱动程序状态，并唤醒任何等待读者。
     */
    uint32_t data = CSR_READ_4(sc, MYFIRST_REG_DATA_OUT);
    sc->intr_last_data = data;
    sc->intr_task_invocations++;

    /* 唤醒任何在数据就绪条件上睡眠的线程。 */
    cv_broadcast(&sc->data_cv);

    MYFIRST_UNLOCK(sc);
}
```

几个显著属性。

**任务在线程上下文运行。** 可以获取`sc->mtx`、使用`cv_broadcast`、调用`malloc(M_WAITOK)`、做慢工作。

**任务遵守第11章锁定纪律。** 获取互斥锁；CSR访问使用标准第16章宏；条件变量广播使用第12章原语。

**任务的参数是softc。** 与过滤器相同。一个微妙含义：任务不能假设驱动程序未被分离。如果detach在过滤器入队任务后但在任务运行前触发，任务可能对已释放softc执行。第7节覆盖防止此问题的纪律（释放前排空）。

**`npending`参数** 是自上次运行以来任务被入队的次数。对于大多数驱动程序，这作为合并提示有用：如果`npending`是5，设备信号了五个数据就绪事件，都合并为一次运行。阶段2的任务忽略它；更大驱动程序用它调整批量操作大小。

### 声明和初始化任务

softc获得任务相关字段：

```c
struct myfirst_softc {
    /* ... 现有字段 ... */

    /* 第19章中断相关字段。 */
    struct resource		*irq_res;
    int			 irq_rid;
    void			*intr_cookie;
    uint64_t		 intr_count;
    uint64_t		 intr_data_av_count;
    uint64_t		 intr_error_count;
    uint64_t		 intr_complete_count;
    uint64_t		 intr_task_invocations;
    uint32_t		 intr_last_data;

    struct taskqueue	*intr_tq;
    struct task		 intr_data_task;
};
```

在`myfirst_init_softc`（或init路径）中：

```c
TASK_INIT(&sc->intr_data_task, 0, myfirst_intr_data_task_fn, sc);
sc->intr_tq = taskqueue_create("myfirst_intr", M_WAITOK,
    taskqueue_thread_enqueue, &sc->intr_tq);
taskqueue_start_threads(&sc->intr_tq, 1, PI_NET,
    "myfirst intr taskq");
```

taskqueue用一个优先级`PI_NET`（中断优先级；见`/usr/src/sys/sys/priority.h`）的工作线程创建。名称`"myfirst intr taskq"`出现在`top -H`中供诊断。创建时的`M_WAITOK`可以，因为`myfirst_init_softc`在attach上下文运行，任何中断触发之前。

### 在设备端启用中断

一个常被遗忘的细节：设备本身必须被告知交付中断。对于第17章模拟的寄存器布局，通过在`INTR_MASK`寄存器中设置位完成：

```c
/* 在附着硬件层后，启用我们关心的中断。 */
MYFIRST_LOCK(sc);
CSR_WRITE_4(sc, MYFIRST_REG_INTR_MASK,
    MYFIRST_INTR_DATA_AV | MYFIRST_INTR_ERROR |
    MYFIRST_INTR_COMPLETE);
MYFIRST_UNLOCK(sc);
```

`INTR_MASK`寄存器控制`INTR_STATUS`的哪些位实际断言IRQ线。没有它，设备可能内部设置`INTR_STATUS`位但从不发出线，所以处理程序从不触发。设置所有三位启用所有三个中断类。

这是教学目标的另一个诚实限制。virtio-rnd遗留BAR上的偏移`0x10`根本不是中断掩码寄存器。在遗留virtio布局中（见`/usr/src/sys/dev/virtio/pci/virtio_pci_legacy_var.h`），起始于`0x10`的双字由三个小字段共享：`0x10`处的`queue_notify`（16位）、`0x12`处的`device_status`（8位）、`0x13`处的`isr_status`（8位）。在该偏移写入我们的`DATA_AV | ERROR | COMPLETE`模式（`0x00000007`）向`queue_notify`写入`0x0007`（通知设备没有的virtqueue索引）并向`device_status`写入`0x00`（virtio规范定义为**设备复位**）。向`device_status`写零是virtio驱动程序在重新初始化前复位设备的方式。

因此，所写的`CSR_WRITE_4(sc, MYFIRST_REG_INTR_MASK, ...)`调用在bhyve virtio-rnd目标上**安全但无意义**：它复位设备的virtio状态机（我们的驱动程序本来就没用），从不启用任何真实中断，因为第17章`INTR_MASK`寄存器在该设备上不存在。如果计划在bhyve上向读者演示，请在代码中保留写入以与真实第17章兼容设备保持连续，依赖第5节的模拟中断sysctl进行测试而不是期望真实IRQ事件。将驱动程序适配到匹配第17章寄存器映射的真实设备的读者会看到掩码写入起作用。

### 在detach时禁用中断

detach时对称步骤：

```c
/* 在拆除前禁用设备端所有中断。 */
MYFIRST_LOCK(sc);
if (sc->hw != NULL)
    CSR_WRITE_4(sc, MYFIRST_REG_INTR_MASK, 0);
MYFIRST_UNLOCK(sc);
```

此写入在`bus_teardown_intr`之前发生，使设备在处理程序被移除前停止断言线。对`sc->hw == NULL`的保护防止硬件层失败的局部附着情况；如果硬件未附着则跳过禁用。

### 工作流程

当`DATA_AV`事件触发时（在实际实现第17章语义的设备上），具体跟踪：

1. 设备设置`INTR_STATUS.DATA_AV`。因为`INTR_MASK.DATA_AV`被设置，设备断言其IRQ线。
2. 中断控制器将IRQ路由到CPU。
3. CPU接受中断并跳转到内核分发代码。
4. 内核找到我们IRQ的`intr_event`并调用`myfirst_intr_filter`。
5. 过滤器读取`INTR_STATUS`、看到`DATA_AV`、递增计数器、向`INTR_STATUS`写回`DATA_AV`（清除它）、入队`intr_data_task`、返回`FILTER_HANDLED`。
6. 设备去断言其IRQ线（因为`INTR_STATUS.DATA_AV`现已清除）。
7. 内核发送EOI、返回被打断线程。
8. 几毫秒后，taskqueue工作线程唤醒、运行`myfirst_intr_data_task_fn`、读取`DATA_OUT`、更新softc、广播条件变量。
9. 任何在条件变量上等待的线程唤醒并继续。

第1到7步在现代硬件上对于简单处理程序需要几微秒。第8步可能需要几百微秒或更多，这就是为什么在线程上下文。分离是让中断路径保持快的原因。

对于bhyve virtio-rnd目标，第1到6步不发生（设备不匹配第17章寄存器布局）。第4到9步仍可通过第5节模拟中断路径演练。

### 内联紧急工作与延迟

决定什么进入过滤器与任务的有用方式：过滤器处理必须**每次中断**做的事，任务处理必须**每个事件**做的事。

每次中断（过滤器）：
- 读取`INTR_STATUS`识别事件。
- 在设备端确认事件（写回`INTR_STATUS`）。
- 更新计数器。
- 做单个调度决策（入队任务）。

每个事件（任务）：
- 从设备寄存器或DMA缓冲区读取数据。
- 更新驱动程序内部状态机。
- 唤醒等待线程。
- 向上传递数据给网络栈、存储栈或cdev队列。
- 处理需要慢恢复的错误。

经验法则：如果过滤器花费超过一百个CPU周期真实工作（不计寄存器访问，其本身便宜），可能做了太多。

### FILTER_SCHEDULE_THREAD与taskqueue

读者可能问：何时使用`FILTER_SCHEDULE_THREAD`而不是taskqueue？

使用`FILTER_SCHEDULE_THREAD`当：
- 您希望内核的每事件ithread（每个`intr_event`一个）运行慢工作。
- 您只需要从过滤器调度工作。
- 您希望调度优先级跟随中断的`INTR_TYPE_*`。

使用taskqueue当：
- 您希望从多条路径（过滤器、ioctl、sysctl、基于睡眠的超时）调度相同工作。
- 您希望在多个设备间共享工作线程。
- 您希望通过`taskqueue_start_threads`显式控制优先级。

对于第19章驱动程序，taskqueue是更干净选择，因为第5节将从模拟中断路径调度相同任务。ithread从那里无法到达。

### 当taskqueue本身是错误答案时

警告。taskqueue适合短延迟工作。不适合长时间运行操作。如果驱动程序需要运行状态机几秒，或阻塞等待USB传输，或处理大缓冲区链，专用工作线程更好。taskqueue工作线程跨任务共享；阻塞很长时间的单个任务延迟其后的每个其他任务。

第19章任务运行微秒级。taskqueue没问题。第20章带每队列接收处理的MSI-X驱动程序可能需要每队列工作线程。第21章DMA驱动的大容量传输可能需要专用线程。每章为其工作负载选择正确原语；第19章使用适合的最简单原语。

### 此阶段常见错误

简短列表。

**读取INTR_STATUS而不确认。** 处理程序读取、判断、返回而不写回。在电平触发线上，设备持续断言；处理程序立即再次触发；风暴。修复：确认您处理的每个位。

**确认太多位。** 粗心处理程序每次调用向`INTR_STATUS`写`0xffffffff`以"清除所有位"。这同时清除了处理程序未处理的事件，丢弃数据或混淆状态机。修复：仅确认实际处理的位。

**在过滤器中获取睡眠锁。** `MYFIRST_LOCK(sc)`获取`sc->mtx`，是睡眠互斥锁。在过滤器中这是bug；`WITNESS`崩溃。修复：在过滤器中使用原子操作，仅在线程上下文的任务中获取睡眠互斥锁。

**在softc被拆除后调度任务。** 如果任务从过滤器调度但过滤器在detach部分拆除后运行，任务对陈旧状态运行。修复：第7节覆盖排序。简短地：`bus_teardown_intr`必须在硬件层释放前发生，`taskqueue_drain`必须在taskqueue释放前发生。

**直接在过滤器中使用`CSR_READ_4`/`CSR_WRITE_4`。** 如果第16章访问器断言持有`sc->mtx`（调试内核上确实如此），过滤器崩溃。修复：使用原始`bus_read_4`/`bus_write_4`或引入并行中断安全CSR宏集。第8节用`ICSR_READ_4`处理。

**在`TASK_INIT`前入队任务。** 在`TASK_INIT`前入队的任务有损坏函数指针。任务首次运行跳转到垃圾。修复：在启用中断前的attach路径中初始化任务。

**忘记在设备端启用中断。** 处理程序已注册，`bus_setup_intr`成功；`vmstat -i`仍显示零触发。问题是设备的`INTR_MASK`寄存器仍为零（或任何复位后值），所以设备从不断言线。修复：在attach期间写入`INTR_MASK`。

**忘记在detach时禁用中断。** 处理程序已被拆除但设备仍在断言线。内核最终抱怨杂散中断，或更糟，共享该线的另一驱动看到神秘活动。修复：在`bus_teardown_intr`前清除`INTR_MASK`。

### 阶段2输出：成功样子

在产生中断的真实设备上加载阶段2后，`dmesg`显示：

```text
myfirst0: <Red Hat Virtio entropy source (myfirst demo target)> ... on pci0
myfirst0: attaching: vendor=0x1af4 device=0x1005 revid=0x00
myfirst0: BAR0 allocated: 0x20 bytes at 0xc1000000
myfirst0: hardware layer attached to BAR: 32 bytes
myfirst0: attached filter handler on IRQ resource
myfirst0: interrupts enabled (mask=0x7)
myfirst0: BAR[0x00] = 0x10010000
```

`interrupts enabled`行是新的。它确认驱动程序已写入`INTR_MASK`。

在产生中断的真实设备上，`sysctl dev.myfirst.0.intr_count`会递增。在bhyve virtio-rnd目标上，计数保持为零，因为设备不触发我们期望的中断。第5节的模拟中断路径是从那里演练处理程序的方式。

### 第4节收尾

真实中断处理程序读取`INTR_STATUS`识别原因、处理每个感兴趣的位、通过写回`INTR_STATUS`确认处理的位、返回正确`FILTER_*`值组合。紧急工作（寄存器访问、计数器更新、确认）在过滤器中发生。慢工作（通过获取`sc->mtx`的第16章访问器读取数据、条件变量广播、用户空间通知）在过滤器入队的taskqueue任务中发生。

过滤器短（典型设备二十到四十行真实代码）。任务也短（十到三十行）。组合使驱动程序功能化：过滤器以中断速率处理中断；任务以线程速率处理事件；分离保持中断窗口紧凑，延迟工作自由阻塞。

第5节是让读者在bhyve目标上演练此机制的章节，那里真实IRQ路径不匹配第17章寄存器语义。它添加在驱动程序正常锁规则下调用过滤器的sysctl，让读者随意触发模拟中断，确认计数器、任务和条件变量广播都按设计行为。

## 第5节：使用模拟中断进行测试

第4节的过滤器和任务是真实驱动程序代码。它们已准备好在匹配第17章寄存器布局的设备上处理真实中断。第19章实验室目标带来的问题是，我们拥有的设备（bhyve下的virtio-rnd）不匹配该布局；向virtio-rnd BAR写入第17章中断掩码位有定义但不相关的效果，从virtio-rnd BAR读取第17章中断状态位返回与模拟语义无关的virtio特定值。在此目标上，过滤器如果触发，会看到垃圾。

第5节通过教读者模拟中断解决此问题。核心想法简单：暴露一个sysctl，写入时在驱动程序正常锁规则下直接调用过滤器处理程序，完全像内核从真实中断调用它那样。过滤器读取`INTR_STATUS`寄存器（读者也通过另一个sysctl或通过仅模拟构建的第17章模拟后端写入），做出与真实中断相同的决策，并端到端驱动完整管道。

### 为什么模拟值得一个节

完成第17章的读者可能合理地问：整个第17章模拟不是已经是模拟中断的方式吗？是也不是。

第17章模拟了**自主设备**。其callout按自己的时间表改变寄存器值，其命令callout在驱动程序写入`CTRL.GO`时触发，其故障注入框架使模拟设备行为不端。第17章的驱动程序是仅模拟驱动程序；没有`bus_setup_intr`因为没有真实总线。

第19章不同。驱动程序现在有在真实IRQ线上注册的真实`bus_setup_intr`处理程序。第17章的callout不参与；在PCI构建上第17章模拟不运行。我们想要的是直接触发**过滤器处理程序**的方式，具有真实中断产生的确切锁语义，以便我们可以在不依赖实际产生正确中断的设备的情况下验证第4节的过滤器和任务管道。

做到这一点的最干净方式（也是许多FreeBSD驱动程序为类似目的使用的方式）是直接调用过滤器函数的sysctl写入。过滤器在调用者上下文（sysctl写入来源的线程上下文）运行，但过滤器的代码不关心外层上下文，只要内部锁纪律正确。原子递增、BAR读取、BAR写入、`taskqueue_enqueue`：所有这些在线程上下文也工作。模拟调用演练内核在真实中断上演练的相同代码路径。

有一个微妙区别。在真实中断上，内核安排同一`intr_event`上的过滤器在一个CPU上串行运行。sysctl触发的模拟调用没有那个保证；另一个线程可能同时在调用过滤器。对于第19章驱动程序这没问题，因为过滤器的状态由原子操作保护（不是内核每IRQ单CPU保证）。对于依赖隐式单CPU串行化的驱动程序，通过sysctl模拟不会是忠实测试。教训是：使用原子和自旋锁的`INTR_MPSAFE`驱动程序干净地转换为模拟。

### 模拟中断sysctl

机制是调用过滤器的只写sysctl：

```c
static int
myfirst_intr_simulate_sysctl(SYSCTL_HANDLER_ARGS)
{
    struct myfirst_softc *sc = arg1;
    uint32_t mask;
    int error;

    mask = 0;
    error = sysctl_handle_int(oidp, &mask, 0, req);
    if (error != 0 || req->newptr == NULL)
        return (error);

    /*
     * "mask"是调用者想要假装设备已设置的INTR_STATUS位。
     * 在真实寄存器中设置它们，然后直接调用过滤器。
     */
    MYFIRST_LOCK(sc);
    if (sc->hw == NULL) {
        MYFIRST_UNLOCK(sc);
        return (ENODEV);
    }
    bus_write_4(sc->bar_res, MYFIRST_REG_INTR_STATUS, mask);
    MYFIRST_UNLOCK(sc);

    /* 直接调用过滤器。 */
    (void)myfirst_intr_filter(sc);

    return (0);
}
```

以及`myfirst_intr_add_sysctls`中的sysctl声明：

```c
SYSCTL_ADD_PROC(&sc->sysctl_ctx,
    SYSCTL_CHILDREN(sc->sysctl_tree), OID_AUTO, "intr_simulate",
    CTLTYPE_UINT | CTLFLAG_WR | CTLFLAG_MPSAFE,
    sc, 0, myfirst_intr_simulate_sysctl, "IU",
    "Simulate an interrupt by setting INTR_STATUS bits and "
    "invoking the filter");
```

写入`dev.myfirst.0.intr_simulate`导致处理程序以指定INTR_STATUS位运行。

### 演练模拟

一旦sysctl就位，读者可以从用户空间驱动完整管道：

```sh
# 模拟DATA_AV事件。
sudo sysctl dev.myfirst.0.intr_simulate=1

# 检查计数器。
sysctl dev.myfirst.0.intr_count
sysctl dev.myfirst.0.intr_data_av_count
sysctl dev.myfirst.0.intr_task_invocations

# 模拟ERROR事件。
sudo sysctl dev.myfirst.0.intr_simulate=2

# 模拟COMPLETE事件。
sudo sysctl dev.myfirst.0.intr_simulate=4

# 同时模拟全部三个。
sudo sysctl dev.myfirst.0.intr_simulate=7
```

第一次调用递增`intr_count`（过滤器触发）、`intr_data_av_count`（DATA_AV位被识别）和最终`intr_task_invocations`（taskqueue任务运行）。第二次递增`intr_count`和`intr_error_count`。第三次递增`intr_count`和`intr_complete_count`。第四次命中所有三个。

读者可以验证完整管道：

```sh
# 循环观察计数器。
while true; do
    sudo sysctl dev.myfirst.0.intr_simulate=1
    sleep 0.5
    sysctl dev.myfirst.0 | grep intr_
done
```

计数器以预期速率向前递增。驱动程序表现得像真实中断在到达。

### 为什么这不是玩具

可能认为此模拟路径是教学产物。不是。许多真实驱动程序保留类似路径用于诊断目的。原因值得命名：

**回归测试。** 模拟中断路径让CI管道无需真实硬件即可演练处理程序。第17章为模拟设备行为做了相同论证；第5节为模拟中断路径做相同论证。

**故障注入。** 模拟中断sysctl让测试注入特定`INTR_STATUS`模式以演练错误处理代码。驱动程序对`INTR_STATUS = ERROR | COMPLETE`（两位同时设置）的响应难以通过真实硬件触发；设置两位并调用处理程序的sysctl使其容易。

**开发者生产力。** 驱动程序作者调试处理程序逻辑时，按需触发处理程序的sysctl极其有用。`dtrace -n 'fbt::myfirst_intr_filter:entry'`结合`sudo sysctl dev.myfirst.0.intr_simulate=1`给出按需单步查看处理程序。

**带来新硬件。** 驱动程序作者常有尚未正确产生中断的原型设备。模拟中断路径让驱动程序上层在硬件工作前被测试，意味着驱动程序和硬件可以并行而不是串行开发。

**教学。** 对于本书目的，模拟路径使过滤器和任务在不自然产生期望中断的实验室目标上可观察。读者可以看到管道工作，即使硬件不配合。

### 模拟路径中的锁定

值得想通的一个细节。sysctl在持有`sc->mtx`时写入`INTR_STATUS`。过滤器处理程序，通过内核真实中断路径调用时，运行时没有持有`sc->mtx`（过滤器直接使用`bus_read_4`/`bus_write_4`，不是锁断言的CSR宏）。通过sysctl调用时，调用上下文是什么？

sysctl处理程序在线程上下文运行。`MYFIRST_LOCK(sc)`获取睡眠互斥锁。锁获取和释放之间，线程持有互斥锁。然后锁被释放，调用`myfirst_intr_filter(sc)`。过滤器不获取锁，仅使用原子和`bus_read_4`/`bus_write_4`、入队任务、返回。整个序列安全。

在持有`sc->mtx`时调用过滤器安全吗？实际上是的：过滤器不尝试获取同一互斥锁，过滤器运行在持有锁不非法的上下文（线程上下文）。但过滤器被设计为上下文无关的；持有睡眠锁调用它会使该合同模糊。sysctl在调用过滤器前释放锁以求清晰。

### 使用第17章模拟产生中断

值得提及的补充技术。第17章的模拟后端如果已附着，按自己的时间表产生自主状态变化。特别是其产生`DATA_AV`的callout设置`INTR_STATUS.DATA_AV`。在仅模拟构建（编译时定义`MYFIRST_SIMULATION_ONLY`）上，模拟是活动的，callout触发，第17章驱动甚至可以从callout本身调用过滤器。

第19章不改变第17章在仅模拟构建上的行为。想看到过滤器由第17章模拟驱动的读者可以用`-DMYFIRST_SIMULATION_ONLY`构建、加载模块并观察callout设置`INTR_STATUS`位。第5节sysctl触发路径在两种构建上仍可用。

在PCI构建上，第17章模拟未附着（第18章的纪律），所以第17章callout不运行。模拟中断路径是PCI构建上驱动过滤器的唯一方式。

### 扩展sysctl以按速率调度

对负载测试有用的扩展：通过callout周期性调度模拟中断的sysctl。callout每N毫秒触发，在`INTR_STATUS`中设置位，并调用过滤器。读者可以调整速率并观察负载下的管道。

```c
static void
myfirst_intr_sim_callout_fn(void *arg)
{
    struct myfirst_softc *sc = arg;

    MYFIRST_LOCK(sc);
    if (sc->intr_sim_period_ms > 0 && sc->hw != NULL) {
        bus_write_4(sc->bar_res, MYFIRST_REG_INTR_STATUS,
            MYFIRST_INTR_DATA_AV);
        MYFIRST_UNLOCK(sc);
        (void)myfirst_intr_filter(sc);
        MYFIRST_LOCK(sc);
        callout_reset_sbt(&sc->intr_sim_callout,
            SBT_1MS * sc->intr_sim_period_ms, 0,
            myfirst_intr_sim_callout_fn, sc, 0);
    }
    MYFIRST_UNLOCK(sc);
}
```

只要`intr_sim_period_ms`非零，callout就重新调度自己。sysctl暴露周期：

```sh
# 每100毫秒触发模拟中断。
sudo sysctl hw.myfirst.intr_sim_period_ms=100

# 停止模拟。
sudo sysctl hw.myfirst.intr_sim_period_ms=0
```

观察计数器以预期速率增长：

```sh
sleep 10
sysctl dev.myfirst.0.intr_count
```

十秒的100毫秒周期后，计数器应读取约100。如果读取少得多，过滤器或任务是瓶颈（在此规模不太可能；更关注高速率测试）。如果读取多得多，有其他东西在从其他地方触发过滤器。

### 模拟未捕获什么

技术的诚实限制。

**并发触发。** sysctl在每次写入串行化模拟中断为一次一个。真实中断路径可以在不同CPU上看到背靠背两个中断，sysctl测试不会产生。对于压力测试并发性，生成多个线程每个写sysctl的单独测试更有效。

**中断控制器行为。** 模拟完全绕过中断控制器。依赖EOI定时、掩码或风暴检测的测试不能通过此方式驱动。

**CPU亲和。** 模拟过滤器在sysctl写入线程所在的CPU运行。真实中断在亲和配置选择的CPU触发。每CPU行为测试需要真实中断或其他机制。

**与真实中断路径的争用。** 如果真实中断也在触发（可能因为设备实际产生了一些），模拟路径可能与真实路径竞争。原子计数器正确处理；更复杂的共享状态可能不行。

这些是限制，不是致命问题。对于大多数第19章测试，模拟路径足够。对于高级压力测试，需要额外技术（rt线程、多CPU调用、真实硬件）。

### 观察任务运行

值得暴露的诊断。任务的`intr_task_invocations`计数器在任务每次运行时递增。读者可以将其与`intr_data_av_count`比较以检查taskqueue是否跟上：

```sh
sudo sysctl dev.myfirst.0.intr_simulate=1    # 触发DATA_AV
sleep 0.1
sysctl dev.myfirst.0.intr_data_av_count       # 应为1
sysctl dev.myfirst.0.intr_task_invocations    # 也应为1
```

如果任务计数器滞后于DATA_AV计数器，taskqueue工作线程积压。在此规模不应发生；在更高速率（每秒数千）可能发生。

更敏感的探针：添加用户空间程序等待的`cv_signal`路径。sysctl触发模拟中断；过滤器入队任务；任务更新`sc->intr_last_data`并广播；等待条件变量的用户空间线程（通过cdev的`read`）醒来。从sysctl写入到唤醒的往返延迟大致是驱动程序中断到用户空间的延迟，一个有用的数字。

### 与第17章故障框架集成

值得注意的想法。第17章的故障注入框架（`FAULT_MASK`和`FAULT_PROB`寄存器）应用于命令，不应用于中断。第19章可以通过添加"下一次中断时故障"选项扩展框架：使下一次过滤器调用跳过确认的sysctl，导致电平触发线上产生风暴。

这是可选扩展。挑战练习提及它；章节正文不要求它。

### 第5节收尾

模拟中断是简单但有效的技术。sysctl写入`INTR_STATUS`、直接调用过滤器、过滤器驱动完整管道：计数器更新、确认、任务入队、任务执行。技术让驱动程序在不自然产生期望中断的实验室目标上端到端演练，且在生产驱动程序中保留用于回归测试和诊断访问的成本很低。

第6节是核心中断处理的最后一个概念部分。它覆盖共享中断：当多个驱动程序在同一IRQ线上监听时发生什么、过滤器处理程序必须如何判断中断是否属于其设备、以及`FILTER_STRAY`在实践中的含义。

## 第6节：处理共享中断

第4节阶段2过滤器已经有共享IRQ的正确返回值形状：`INTR_STATUS`为零时返回`FILTER_STRAY`。第6节探讨为什么该检查是整个纪律、处理程序出错时出了什么问题、以及何时值得设置`RF_SHAREABLE`标志。

### 为什么要共享IRQ？

两个原因。

首先，**硬件约束**。经典PC架构有16条硬件IRQ线；I/O APIC在许多芯片组上扩展到24条。有30个设备的系统必然有些共享线路。现代x86系统短缺不那么严重（数百向量），但在传统PCI和许多arm64 SoC上共享是正常的。

其次，**驱动可移植性**。正确处理共享中断的驱动程序也正确处理独占中断（共享路径是超集）。假设独占中断的驱动程序在硬件改变或另一驱动程序到达同一线路时出问题。为共享情况编写基本上不花成本且面向未来。

在启用MSI或MSI-X的PCIe上（第20章），每个设备有自己的向量，共享很少需要。但即使在那里，正确处理杂散中断（通过返回`FILTER_STRAY`）的驱动程序比不处理的更好。纪律可以迁移。

### 共享IRQ上的流程

当共享IRQ触发时，内核按注册顺序遍历附着在`intr_event`上的过滤器处理程序列表。每个过滤器运行、检查中断是否属于其设备、相应返回：

- 如果过滤器声称中断（返回`FILTER_HANDLED`或`FILTER_SCHEDULE_THREAD`），内核继续下一个过滤器（如果有）并聚合结果。在现代内核上，返回`FILTER_HANDLED`的过滤器不停止后续过滤器运行；内核总是遍历整个列表。
- 如果过滤器返回`FILTER_STRAY`，内核尝试下一个过滤器。

所有过滤器运行后，如果有过滤器声称中断，内核在中断控制器确认并返回。如果所有过滤器返回`FILTER_STRAY`，内核递增杂散中断计数器；如果杂散计数超过阈值，内核禁用IRQ（激烈的最后手段）。

当中断实际属于其设备时返回`FILTER_STRAY`的过滤器是bug：线保持断言（电平触发），风暴机制启动，设备得不到服务。当中断不属于其设备时返回`FILTER_HANDLED`的过滤器也是bug：另一驱动程序的中断被标记为已服务，其处理程序从不运行，其数据停留在FIFO，用户的网络或磁盘停止工作。

纪律是基于设备状态精确判断所有权，返回正确值。

### INTR_STATUS测试

判断所有权的标准方式是读取告诉我们中断是否待处理的设备寄存器。在有每设备INTR_STATUS寄存器的设备上，问题是"INTR_STATUS中有位设置吗？"如果是，中断是我的。如果不是，不是。

第17章寄存器布局使这容易：

```c
status = bus_read_4(sc->bar_res, MYFIRST_REG_INTR_STATUS);
if (status == 0)
    return (FILTER_STRAY);
```

这正是阶段2过滤器已做的。模式健壮：如果状态寄存器读为零，此设备无待处理事件，所以中断不是我们的。

微妙细节：INTR_STATUS读取必须发生在任何可能掩码或重置位的状态改变之前。在中途状态读取INTR_STATUS没问题（寄存器反映设备当前视图）；先写其他寄存器然后读取INTR_STATUS可能错过写入无意清除的位。

### "是我的吗？"在真实硬件上看起来像什么

INTR_STATUS测试是教科书式的，因为第17章寄存器布局是教科书式的。真实设备有各种风味。

**有干净INTR_STATUS的设备。** 大多数现代设备有读为零时明确说"不是我的"的寄存器。第19章驱动程序过滤器形状直接适用。

**有总是设置的位的设备。** 某些设备有跨中断保持设置的待处理中断位（等待驱动程序重置）。过滤器必须掩码这些或对照每中断类掩码检查。第17章寄存器布局避免了这种复杂；真实驱动偶尔面对。

**根本没有INTR_STATUS的设备。** 少数旧设备要求驱动读取单独的寄存器序列（或从状态寄存器推断）以判断中断是否待处理。这些驱动更复杂；过滤器可能需要获取自旋锁并读取几个寄存器。FreeBSD源码在少数嵌入式驱动中有示例。

**有全局INTR_STATUS和每源寄存器的设备。** NIC上常见模式：顶层寄存器报告哪个队列有待处理事件，每队列寄存器包含事件详情。过滤器读取顶层寄存器判断所有权；ithread或任务读取每队列寄存器处理事件。

第19章驱动使用第一种风味。其他风味的纪律相同：读寄存器、判断。

### 正确返回FILTER_STRAY

规则简单：如果过滤器没有将任何位识别为属于它处理的类，返回`FILTER_STRAY`。

```c
if (rv == 0)
    return (FILTER_STRAY);
```

变量`rv`从每个识别的位累积`FILTER_HANDLED`。如果没有位被识别，`rv`为零，过滤器除了`FILTER_STRAY`别无返回。

微妙推论：识别了一些位但没识别其他位的过滤器为识别的位返回`FILTER_HANDLED`，不为未识别的位返回`FILTER_STRAY`。在`INTR_MASK`中设置驱动程序不处理的位是驱动bug；内核帮不了。

有趣的边界情况：INTR_STATUS中设置了位但驱动不识别它（可能是新设备修订添加了驱动代码之前的位）。驱动有两个选项：

1. 忽略位。不确认它。让它保持设置。在电平触发线上这产生风暴，因为位永远断言线。不好。

2. 不做任何工作确认位。写回INTR_STATUS。设备停止为该位断言，无风暴，但事件丢失。在重要事件上这是功能bug；在诊断事件上可能可接受。

推荐模式是选项2加日志消息：确认未识别位、以降低速率记录（避免位持续断言时日志泛洪）、继续。这使驱动对新硬件修订健壮，代价是可能丢失未知事件信息。

```c
uint32_t unknown = status & ~(MYFIRST_INTR_DATA_AV |
    MYFIRST_INTR_ERROR | MYFIRST_INTR_COMPLETE);
if (unknown != 0) {
    atomic_add_64(&sc->intr_unknown_count, 1);
    bus_write_4(sc->bar_res, MYFIRST_REG_INTR_STATUS, unknown);
    rv |= FILTER_HANDLED;
}
```

此代码片段不在阶段2过滤器中；它是阶段3或以后的有用扩展。

### 如果多个驱动共享IRQ会发生什么

具体场景。假设bhyve客户机中的virtio-rnd设备与AHCI控制器共享IRQ 19。两个驱动都注册了处理程序。IRQ 19上到达中断。

内核按注册顺序遍历处理程序列表。假设AHCI先注册，所以其过滤器先运行：

1. AHCI过滤器：读取其INTR_STATUS，看到位设置（AHCI有待处理I/O），确认，返回`FILTER_HANDLED`。
2. `myfirst`过滤器：读取其INTR_STATUS，读到零，返回`FILTER_STRAY`。

内核看到"至少一个FILTER_HANDLED"，不将中断标记为杂散。

现在反过来。virtio-rnd设备有事件：

1. AHCI过滤器：读取其INTR_STATUS，看到零，返回`FILTER_STRAY`。
2. `myfirst`过滤器：读取其INTR_STATUS，看到`DATA_AV`，确认，返回`FILTER_HANDLED`。

内核看到一个`FILTER_HANDLED`就满意了。

关键属性是每个过滤器只检查自己的设备。没有过滤器假设中断是自己的；每个从自己设备状态决定。

### 如果驱动弄错了会发生什么

一个损坏的AHCI过滤器如果每次触发都返回`FILTER_HANDLED`（不检查状态）会认领我们的`myfirst`中断。`myfirst`过滤器永远不会运行，`DATA_AV`永远不会被确认，线路会风暴。

修复不在`myfirst`侧；在AHCI侧。实际上，所有主要FreeBSD驱动程序检查都正确，因为代码已审计和测试多年。教训是共享IRQ协议需要合作：线上每个驱动必须正确检查自己的状态。

针对单个损坏驱动的保护是`hw.intr_storm_threshold`。当内核检测到连续中断都被标记为杂散（或都返回`FILTER_HANDLED`而实际上没有设备有工作）时，最终会掩码线路。这是检测机制，不是预防机制。

### 与非共享驱动共存

用`RF_SHAREABLE`分配IRQ的驱动可以与不共享分配的驱动共存，只要内核能满足两个请求。如果我们的`myfirst`驱动先用`RF_SHAREABLE`分配，然后AHCI尝试独占分配，AHCI的分配会失败（线路已被可能不独占的驱动持有）。如果AHCI先不共享分配，我们的`myfirst`分配（带`RF_SHAREABLE`）会失败。

实际上，现代驱动几乎总是使用`RF_SHAREABLE`。遗留驱动偶尔省略它；如果读者的驱动因中断分配冲突无法加载，修复通常是向分配添加`RF_SHAREABLE`。

独占分配适用于：

- 有严格延迟要求且不能容忍线上其他处理程序的驱动。
- 因特定内核原因使用`INTR_EXCL`的驱动。
- 一些在共享IRQ支持成熟之前编写的遗留驱动。

对于第19章驱动程序，`RF_SHAREABLE`是默认且永不错误。

### bhyve Virtio IRQ拓扑

关于第19章实验室环境的实际细节。bhyve模拟器根据插槽的INTx引脚将每个模拟PCI设备映射到IRQ线。同一插槽上不同功能的多个设备共享一条线；不同插槽通常有不同线。插槽4功能0的virtio-rnd设备有自己的引脚。

实际上，在只有少量模拟设备的bhyve客户机上，每个设备通常有自己的IRQ线（无共享）。在非共享线上用`RF_SHAREABLE`分配的`myfirst`驱动行为与非可共享分配相同；标志无害。

要在bhyve中故意测试共享IRQ行为，读者可以将多个virtio设备堆叠到同一插槽（不同功能），强制它们共享一条线。这是高级操作，对基础第19章实验不必要。

### 饥饿担忧

共享IRQ线有潜在的饥饿问题：在过滤器中花费太长时间的单一驱动可以延迟线上每个其他驱动。每个过滤器看到其设备状态在慢过滤器持续期间"不变"，事件可能未被检测地累积。

纪律与第4节覆盖的相同：过滤器必须快。数十或数百微秒的实际工作通常是行为良好的过滤器所做的最大值；任何更慢的移到任务。做长工作的过滤器不仅饿死自己驱动的上层，还饿死线上每个其他驱动。

在MSI-X（第20章）上，每个向量有自己的`intr_event`，所以饥饿担忧对使用MSI-X的特定驱动对消失。但纪律仍适用：花费一毫秒的过滤器在延迟上伤害每个后续中断。

### 误报与防御性处理

状态寄存器检查的一个有用属性是它天然容忍来自内核侧的误报。偶尔，中断控制器在没有设备实际断言时报告杂散中断（线上噪声、边沿触发与掩码之间的竞争、平台特定怪癖）。内核分发，过滤器读取INTR_STATUS，它是零，过滤器返回`FILTER_STRAY`，内核继续。

这对驱动是无操作。杂散中断计数上升；其他无变化。

一些驱动添加速率限制的日志消息使杂散中断可见。合理默认是仅在速率超过阈值时记录：

```c
static struct timeval last_stray_log;
static int stray_rate_limit = 5;  /* 每秒消息数 */
if (rv == 0) {
    if (ppsratecheck(&last_stray_log, &stray_rate_limit, 1))
        device_printf(sc->dev, "spurious interrupt\n");
    return (FILTER_STRAY);
}
```

`ppsratecheck(9)`工具限制消息速率。没有它，正在风暴的线会用相同消息泛洪`dmesg`。

第19章驱动在阶段2过滤器中不包括速率限制日志；它在挑战练习中添加。

### 当过滤器应处理而任务不应运行时

一个思想实验。想象过滤器识别`ERROR`但不识别`DATA_AV`。过滤器处理`ERROR`（确认、递增计数器）并返回`FILTER_HANDLED`。无任务入队。设备满意；线解除断言。

但`INTR_STATUS.DATA_AV`可能仍设置，因为过滤器未确认它（过滤器未将位识别为驱动处理的类）。在电平触发线上，设备持续为`DATA_AV`断言，新中断触发，循环重复。

这是"未知位风暴"问题的一个版本。修复是确认驱动愿意看到的每个位，即使驱动对某些不做任何事。将`INTR_MASK`设置为仅驱动处理的位是预防措施；在过滤器中确认未识别位是防御措施。

### 第6节收尾

共享中断是传统PCI的常见情况，在现代硬件上仍是正确假设编写。共享线上的过滤器必须检查中断是否属于其设备（通常通过读取设备INTR_STATUS寄存器）、处理识别的位、确认那些位、如果什么都没识别则返回`FILTER_STRAY`。纪律在代码上小但在可靠性上大：正确的驱动与其线上每个行为良好的驱动共存，分配中的`RF_SHAREABLE`是唯一额外代码行。

第7节是拆除节。它短：先`bus_teardown_intr`再`bus_release_resource`、在任务触及的任何东西释放前排空taskqueue、清除`INTR_MASK`使设备停止断言、验证计数器合理。但顺序严格，第19章detach路径恰好多了这些步骤。

## 第7节：清理中断资源

attach路径获得三个新操作（分配IRQ、注册处理程序、在设备端启用中断）；detach路径必须严格逆序撤销每个。第7节简短因为模式现已熟悉；但顺序以之前节未触及的特定方式重要，此处的错误产生调试内核非常善于捕获也非常善于使诊断困惑的内核崩溃。

### 必要的顺序

从最特定到最通用，第19章阶段2的detach序列是：

1. **忙时拒绝。** `myfirst_is_busy(sc)`在cdev打开或命令进行中时返回true。
2. **标记为不再attached**使拒绝用户空间路径开始。
3. **销毁cdev**使无新用户空间访问开始。
4. **在设备端禁用中断。** 清除`INTR_MASK`使设备停止断言。
5. **拆除中断处理程序。** `bus_teardown_intr(9)`对`irq_res`用保存的cookie。此调用返回后，内核保证过滤器不会再次运行。
6. **排空taskqueue。** `taskqueue_drain(9)`等待任何待处理任务完成并阻止新任务开始。
7. **销毁taskqueue。** `taskqueue_free(9)`关闭工作线程。
8. **静默第17章模拟callout**如`sc->sim`非NULL。
9. **分离第17章模拟**如已附着。
10. **分离硬件层**使`sc->hw`被释放。
11. **释放IRQ资源**用`bus_release_resource(9)`。
12. **释放BAR**用`bus_release_resource(9)`。
13. **反初始化softc。**

十三个步骤。每个做一件事。危险在顺序。

### 为什么在bus_teardown_intr前在设备端禁用

在拆除处理程序前清除`INTR_MASK`是防御性的。如果我们先拆除处理程序，设备待处理中断可能触发而无处理程序；内核标记为杂散并最终禁用线。先清除`INTR_MASK`停止设备断言，然后拆除移除处理程序，中间无中断可触发。

对于MSI-X（第20章），逻辑略有不同因为每个向量独立。但原则迁移：在移除处理程序前停止源。

在真实硬件上此窗口是微秒级；期间杂散中断罕见。在bhyve上事件率低，基本不发生。但仔细的驱动无论如何关闭窗口，因为仔细的驱动是您想在生产中阅读的。

### 为什么bus_teardown_intr在释放资源前

`bus_teardown_intr`从`intr_event`移除驱动程序的处理程序。返回后，内核保证过滤器不会再次运行。但IRQ资源（`struct resource *`）仍有效；内核未释放它。`bus_release_resource`是释放它的。

如果先释放资源，内核围绕`intr_event`的内部簿记会看到注册在不再存在资源上的处理程序。取决于定时，这产生要么`bus_release_resource`期间立即失败（内核检测到处理程序仍附着），要么稍后线路尝试触发时延迟问题。

安全顺序总是先`bus_teardown_intr`。`bus_setup_intr(9)`手册页明确说明。

### 为什么在释放Softc前排空taskqueue

过滤器可能入队了尚未运行的任务。任务的函数指针存储在`struct task`中，参数指针是softc。如果我们在任务运行前释放softc，任务会解引用已释放指针并崩溃。

`taskqueue_drain(9)`对特定任务等待该任务完成并阻止该任务未来入队运行。在`&sc->intr_data_task`上调用`taskqueue_drain`恰好是正确的：等待数据可用任务完成。

`taskqueue_drain`返回后，无任务运行进行中。softc可以安全释放。

常见错误：用`taskqueue_drain(tq, &task)`排空单个任务与用`taskqueue_drain_all(tq)`排空整个taskqueue不同。对于同一taskqueue上有多个任务的驱动，每个任务需要自己的排空，或`taskqueue_drain_all`作为组处理它们。

对于第19章驱动，有一个任务，单个`taskqueue_drain`足够。

### 为什么bus_teardown_intr在taskqueue_drain前

过滤器在`INTR_MASK`被清除和`bus_teardown_intr`返回之间仍可能入队任务。如果我们在拆除处理程序前排空taskqueue，仍在运行的过滤器可能在排空后入队任务，排空的保证被违反。

正确顺序是：清除`INTR_MASK`（停止新中断）、拆除处理程序（停止过滤器再次运行）、排空taskqueue（停止任何先前入队任务运行）。每步缩小可触及状态的代码路径集。

### 清理代码

将顺序放入第19章阶段2detach：

```c
static int
myfirst_pci_detach(device_t dev)
{
    struct myfirst_softc *sc = device_get_softc(dev);

    if (myfirst_is_busy(sc))
        return (EBUSY);

    sc->pci_attached = false;

    /* 销毁cdev使无新用户空间访问开始。 */
    if (sc->cdev != NULL) {
        destroy_dev(sc->cdev);
        sc->cdev = NULL;
    }

    /* 在设备端禁用中断。 */
    MYFIRST_LOCK(sc);
    if (sc->hw != NULL && sc->bar_res != NULL)
        bus_write_4(sc->bar_res, MYFIRST_REG_INTR_MASK, 0);
    MYFIRST_UNLOCK(sc);

    /* 拆除中断处理程序。 */
    if (sc->intr_cookie != NULL) {
        bus_teardown_intr(dev, sc->irq_res, sc->intr_cookie);
        sc->intr_cookie = NULL;
    }

    /* 排空并销毁中断taskqueue。 */
    if (sc->intr_tq != NULL) {
        taskqueue_drain(sc->intr_tq, &sc->intr_data_task);
        taskqueue_free(sc->intr_tq);
        sc->intr_tq = NULL;
    }

    /* 静默第17章callout（如模拟已附着）。 */
    myfirst_quiesce(sc);

    /* 分离第17章模拟如已附着。 */
    if (sc->sim != NULL)
        myfirst_sim_detach(sc);

    /* 分离硬件层。 */
    myfirst_hw_detach(sc);

    /* 释放IRQ资源。 */
    if (sc->irq_res != NULL) {
        bus_release_resource(dev, SYS_RES_IRQ, sc->irq_rid,
            sc->irq_res);
        sc->irq_res = NULL;
    }

    /* 释放BAR。 */
    if (sc->bar_res != NULL) {
        bus_release_resource(dev, SYS_RES_MEMORY, sc->bar_rid,
            sc->bar_res);
        sc->bar_res = NULL;
    }

    myfirst_deinit_softc(sc);

    device_printf(dev, "detached\n");
    return (0);
}
```

十三个不同动作，每个简单。代码比早期阶段长仅因为每个新能力添加自己的拆除步骤。

### 处理部分Attach失败

第3节attach路径的goto级联有每个分配步骤的标签。随着阶段2注册中断处理程序，级联多增长一个：

```c
fail_teardown_intr:
    MYFIRST_LOCK(sc);
    if (sc->hw != NULL && sc->bar_res != NULL)
        bus_write_4(sc->bar_res, MYFIRST_REG_INTR_MASK, 0);
    MYFIRST_UNLOCK(sc);
    bus_teardown_intr(dev, sc->irq_res, sc->intr_cookie);
    sc->intr_cookie = NULL;
fail_release_irq:
    bus_release_resource(dev, SYS_RES_IRQ, sc->irq_rid, sc->irq_res);
    sc->irq_res = NULL;
fail_hw:
    myfirst_hw_detach(sc);
fail_release_bar:
    bus_release_resource(dev, SYS_RES_MEMORY, sc->bar_rid, sc->bar_res);
    sc->bar_res = NULL;
fail_softc:
    myfirst_deinit_softc(sc);
    return (error);
```

每个级联标签撤销其之前成功的步骤。失败的`bus_setup_intr`跳转到`fail_release_irq`（跳过拆除因为处理程序未注册）。失败的`make_dev`（cdev创建）跳转到`fail_teardown_intr`（在释放IRQ前拆除处理程序）。

taskqueue在`myfirst_init_softc`中初始化，在`myfirst_deinit_softc`中销毁，所以级联不需要显式处理它；任何到达`fail_softc`的标签都通过反初始化清理taskqueue。

### 验证拆除

`kldunload myfirst`后内核应处于干净状态。具体检查：

- `kldstat -v | grep myfirst`无返回（模块已卸载）。
- `devinfo -v | grep myfirst`无返回（设备已分离）。
- `vmstat -i | grep myfirst`无返回（中断事件已清理）。
- `vmstat -m | grep myfirst`无返回或显示零`InUse`（malloc类型已排空）。
- `dmesg | tail`显示分离横幅且无警告或崩溃。

任何这些失败都是bug。最常见失败是`vmstat -i`显示陈旧条目；这通常意味着`bus_teardown_intr`未被调用。第二常见是`vmstat -m`显示活跃分配；这通常意味着任务被入队未排空，或模拟被附着未分离。

### 处理"处理程序在Detach期间触发"的情况

值得想通的微妙情况。假设真实中断在cdev销毁和`INTR_MASK`写入之间在共享IRQ线上触发。另一个驱动设备在断言，我们的过滤器运行（因为线共享），我们的过滤器读取`INTR_STATUS`（在我们设备上为零），返回`FILTER_STRAY`。无状态触及，无任务入队。

假设中断来自我们的设备。我们的`INTR_STATUS`有位设置。过滤器识别它、确认、入队任务、返回。任务入队针对尚未排空的taskqueue。任务稍后运行、获取`sc->mtx`、通过硬件层读取`DATA_OUT`（仍然附着因为我们还未调用`myfirst_hw_detach`）。全部安全。

假设中断在`INTR_MASK = 0`之后但`bus_teardown_intr`之前到达。设备已停止为我们清除的位断言，但已在途的中断（在中断控制器中排队）仍可运行过滤器。过滤器读取`INTR_STATUS`、看到零（因为掩码写入领先设备内部状态）、返回`FILTER_STRAY`。中断被计为杂散；内核忽略它。

假设中断在`bus_teardown_intr`之后到达。处理程序已消失。内核的杂散中断记账注意到。足够多杂散后，内核禁用线路。这是`INTR_MASK = 0`步骤设计要防止的场景；如果掩码先清除，无杂散可累积。

代码路径都是防御性的。调试内核断言捕获常见错误。遵循第19章顺序的驱动可靠地干净拆除。

### 跳过拆除会出什么问题

产生具体症状的几个场景。

**处理程序未拆除，资源已释放。** `kldunload`在IRQ上调用`bus_release_resource`而无`bus_teardown_intr`。内核检测到正在释放的资源上有活跃处理程序并以"releasing allocated IRQ with active handler"消息崩溃。调试内核在此可靠。

**处理程序已拆除，taskqueue未排空。** 任务在过滤器中入队，过滤器最后调用恰好发生在拆除之前，任务尚未运行。驱动释放`sc`（通过softc反初始化）并卸载。任务工作线程醒来、运行任务函数、解引用已释放softc、以空指针或释放后使用故障崩溃。调试内核的`WITNESS`或`MEMGUARD`可能捕获它；如无，崩溃在任务函数首次内存访问时。

**Taskqueue已排空，未释放。** `taskqueue_drain`成功，但`taskqueue_free`被跳过。taskqueue工作线程继续运行（空闲）。`vmstat -m`显示分配。不是功能bug，但是跨加载卸载循环累积的泄漏。

**模拟callout未静默。** 如果第17章模拟已附着（在仅模拟构建上），其callout在运行。不静默，它们在detach释放寄存器块后触发，并访问垃圾。`WITNESS`或`MEMGUARD`捕获因命中而异；有时普通空指针解引用是症状。

**INTR_MASK未清除。** 真实中断在detach开始后触发。过滤器（短暂地，直到拆除）处理它们；拆除后，它们是内核最终禁用线路的杂散。线路禁用状态在`vmstat -i`中可见（增长杂散计数）和`dmesg`中（内核警告）。

每个都可通过修复拆除顺序恢复。第19章代码设置正确；危险在于修改顺序的读者。

### 拆除健全性测试

读者编写detach代码后可运行的简单健全性测试：

```sh
# 加载。
sudo kldload ./myfirst.ko

# 触发几个模拟中断，确保任务运行。
for i in 1 2 3 4 5; do
    sudo sysctl dev.myfirst.0.intr_simulate=1
done
sleep 1
sysctl dev.myfirst.0.intr_task_invocations  # 应为5

# 卸载。
sudo kldunload myfirst

# 检查无泄漏。
vmstat -m | grep myfirst  # 应为空
devinfo -v | grep myfirst   # 应为空
vmstat -i | grep myfirst    # 应为空
```

在循环中运行此序列（shell循环中二十次迭代）是合理的回归测试：任何泄漏累积、任何崩溃显现、任何失败模式变得可见。

### 第7节收尾

清理中断资源是detach路径中的六个小操作：禁用`INTR_MASK`、拆除处理程序、排空并释放taskqueue、分离硬件层、释放IRQ、释放BAR。每个操作恰好撤销一个attach路径操作。顺序是attach的逆。taskqueue排空是过滤器加任务驱动特有的新关注点；跳过它的驱动有等待下次加载卸载循环的释放后使用bug。

第8节是整理节：将中断代码拆分到自己的文件、提升版本到`1.2-intr`、写`INTERRUPTS.md`、运行回归通过。驱动程序在第7节后功能完整；第8节使其可维护。

## 第8节：重构和版本化您的中断就绪驱动程序

中断处理程序工作了。第8节是整理节。它将中断代码拆分到自己的文件、更新模块元数据、添加新`INTERRUPTS.md`文档、引入小组中断上下文CSR宏使过滤器可以在没有锁断言宏的情况下访问寄存器、提升版本到`1.2-intr`、运行回归通过。

到达此处的读者可能再次被诱惑跳过此节。这与第18章第8节警告的诱惑相同，同样的拒绝：中断代码混入PCI文件、过滤器临时使用原始`bus_read_4`、taskqueue设置分散在三个文件中的驱动程序变得难以扩展。第20章添加MSI和MSI-X；第21章添加DMA。两者都建立在第19章中断代码上。现在干净的结构节省两者的工作量。

### 最终文件布局

在第19章结束时，驱动程序由以下文件组成：

```text
myfirst.c         - 主驱动：softc、cdev、模块事件、数据路径。
myfirst.h         - 共享声明：softc、锁宏、原型。
myfirst_hw.c      - 第16章硬件访问层：CSR_*访问器、
                     访问日志、sysctl处理程序。
myfirst_hw_pci.c  - 第18章硬件层扩展：myfirst_hw_attach_pci。
myfirst_hw.h      - 寄存器映射和访问器声明。
myfirst_sim.c     - 第17章模拟后端。
myfirst_sim.h     - 第17章模拟接口。
myfirst_pci.c     - 第18章PCI attach：probe、attach、detach、
                     DRIVER_MODULE、MODULE_DEPEND、ID表。
myfirst_pci.h     - 第18章PCI声明。
myfirst_intr.c    - 第19章中断处理程序：过滤器、任务、setup、teardown。
myfirst_intr.h    - 第19章中断接口。
myfirst_sync.h    - 第3部分同步原语。
cbuf.c / cbuf.h   - 第10章循环缓冲区。
Makefile          - kmod构建。
HARDWARE.md       - 第16/17章寄存器映射。
LOCKING.md        - 第15章起锁定纪律。
SIMULATION.md     - 第17章模拟。
PCI.md            - 第18章PCI支持。
INTERRUPTS.md     - 第19章中断处理。
```

`myfirst_intr.c`和`myfirst_intr.h`是新的。`INTERRUPTS.md`是新的。其他每个文件要么之前存在要么略有扩展（softc获得字段；PCI attach调用`myfirst_intr.c`）。

经验法则保持：每个文件一个职责。`myfirst_intr.c`拥有中断处理程序、延迟任务和模拟中断sysctl。`myfirst_pci.c`拥有PCI attach但委托中断设置和拆除给`myfirst_intr.c`导出的函数。

### 最终Makefile

```makefile
# 第19章 myfirst 驱动程序的 Makefile。

KMOD=  myfirst
SRCS=  myfirst.c \
       myfirst_hw.c myfirst_hw_pci.c \
       myfirst_sim.c \
       myfirst_pci.c \
       myfirst_intr.c \
       cbuf.c

CFLAGS+= -DMYFIRST_VERSION_STRING=\"1.2-intr\"

# CFLAGS+= -DMYFIRST_SIMULATION_ONLY
# CFLAGS+= -DMYFIRST_PCI_ONLY

.include <bsd.kmod.mk>
```

SRCS列表中多了一个源文件；版本字符串提升；其余不变。

### 版本字符串

`1.1-pci`到`1.2-intr`。提升反映驱动程序获得了重要新能力（中断处理）而不改变任何用户可见接口（cdev仍做之前做的事）。次版本提升适当。

后续章节继续：第20章MSI和MSI-X工作后`1.3-msi`；第20和21章添加DMA后`1.4-dma`。每次次版本反映一个重要能力添加。

### myfirst_intr.h头文件

头文件向驱动程序其余部分导出中断层的公共接口：

```c
#ifndef _MYFIRST_INTR_H_
#define _MYFIRST_INTR_H_

#include <sys/types.h>
#include <sys/taskqueue.h>

struct myfirst_softc;

/* 中断设置和拆除，从PCI attach路径调用。 */
int  myfirst_intr_setup(struct myfirst_softc *sc);
void myfirst_intr_teardown(struct myfirst_softc *sc);

/* 注册中断层特定的sysctl节点。 */
void myfirst_intr_add_sysctls(struct myfirst_softc *sc);

/* 中断上下文访问器宏。这些不获取sc->mtx，
 * 因此在过滤器中安全。它们不是其他上下文中
 * CSR_READ_4 / CSR_WRITE_4 的替代品。 */
#define ICSR_READ_4(sc, off) \
	bus_read_4((sc)->bar_res, (off))
#define ICSR_WRITE_4(sc, off, val) \
	bus_write_4((sc)->bar_res, (off), (val))

#endif /* _MYFIRST_INTR_H_ */
```

公共API是三个函数（`myfirst_intr_setup`、`myfirst_intr_teardown`、`myfirst_intr_add_sysctls`）和两个访问器宏（`ICSR_READ_4`、`ICSR_WRITE_4`）。"I"前缀代表"interrupt-context"；这些宏不获取`sc->mtx`，所以在过滤器中安全。

### myfirst_intr.c文件

完整文件在配套示例树中；以下是核心结构：

```c
#include <sys/param.h>
#include <sys/systm.h>
#include <sys/kernel.h>
#include <sys/bus.h>
#include <sys/lock.h>
#include <sys/mutex.h>
#include <sys/sysctl.h>
#include <sys/taskqueue.h>
#include <sys/rman.h>

#include <machine/bus.h>
#include <machine/resource.h>

#include "myfirst.h"
#include "myfirst_hw.h"
#include "myfirst_intr.h"

/* 数据可用事件的延迟任务。 */
static void myfirst_intr_data_task_fn(void *arg, int npending);

/* 过滤器处理程序。导出以便模拟中断sysctl可以
 * 直接调用它。 */
int myfirst_intr_filter(void *arg);

int
myfirst_intr_setup(struct myfirst_softc *sc)
{
	int error;

	TASK_INIT(&sc->intr_data_task, 0, myfirst_intr_data_task_fn, sc);
	sc->intr_tq = taskqueue_create("myfirst_intr", M_WAITOK,
	    taskqueue_thread_enqueue, &sc->intr_tq);
	taskqueue_start_threads(&sc->intr_tq, 1, PI_NET,
	    "myfirst intr taskq");

	sc->irq_rid = 0;
	sc->irq_res = bus_alloc_resource_any(sc->dev, SYS_RES_IRQ,
	    &sc->irq_rid, RF_SHAREABLE | RF_ACTIVE);
	if (sc->irq_res == NULL)
		return (ENXIO);

	error = bus_setup_intr(sc->dev, sc->irq_res,
	    INTR_TYPE_MISC | INTR_MPSAFE,
	    myfirst_intr_filter, NULL, sc,
	    &sc->intr_cookie);
	if (error != 0) {
		bus_release_resource(sc->dev, SYS_RES_IRQ, sc->irq_rid,
		    sc->irq_res);
		sc->irq_res = NULL;
		return (error);
	}

	bus_describe_intr(sc->dev, sc->irq_res, sc->intr_cookie, "legacy");

	/* 在设备端启用我们关心的中断。 */
	MYFIRST_LOCK(sc);
	if (sc->hw != NULL)
		CSR_WRITE_4(sc, MYFIRST_REG_INTR_MASK,
		    MYFIRST_INTR_DATA_AV | MYFIRST_INTR_ERROR |
		    MYFIRST_INTR_COMPLETE);
	MYFIRST_UNLOCK(sc);

	return (0);
}

void
myfirst_intr_teardown(struct myfirst_softc *sc)
{
	/* 在设备端禁用中断。 */
	MYFIRST_LOCK(sc);
	if (sc->hw != NULL && sc->bar_res != NULL)
		CSR_WRITE_4(sc, MYFIRST_REG_INTR_MASK, 0);
	MYFIRST_UNLOCK(sc);

	/* 拆除处理程序。 */
	if (sc->intr_cookie != NULL) {
		bus_teardown_intr(sc->dev, sc->irq_res, sc->intr_cookie);
		sc->intr_cookie = NULL;
	}

	/* 排空并销毁taskqueue。 */
	if (sc->intr_tq != NULL) {
		taskqueue_drain(sc->intr_tq, &sc->intr_data_task);
		taskqueue_free(sc->intr_tq);
		sc->intr_tq = NULL;
	}

	/* 释放IRQ资源。 */
	if (sc->irq_res != NULL) {
		bus_release_resource(sc->dev, SYS_RES_IRQ, sc->irq_rid,
		    sc->irq_res);
		sc->irq_res = NULL;
	}
}

int
myfirst_intr_filter(void *arg)
{
	/* ... 如第4节所述 ... */
}

static void
myfirst_intr_data_task_fn(void *arg, int npending)
{
	/* ... 如第4节所述 ... */
}

void
myfirst_intr_add_sysctls(struct myfirst_softc *sc)
{
	/* ... 计数器和intr_simulate sysctl ... */
}
```

文件在阶段4约250行。`myfirst_pci.c`相应缩短：中断分配和设置移出。

### 重构后的PCI Attach

将中断代码移入`myfirst_intr.c`后，`myfirst_pci_attach`变为：

```c
static int
myfirst_pci_attach(device_t dev)
{
	struct myfirst_softc *sc = device_get_softc(dev);
	int error;

	sc->dev = dev;
	sc->unit = device_get_unit(dev);
	error = myfirst_init_softc(sc);
	if (error != 0)
		return (error);

	/* 步骤1：分配BAR 0。 */
	sc->bar_rid = PCIR_BAR(0);
	sc->bar_res = bus_alloc_resource_any(dev, SYS_RES_MEMORY,
	    &sc->bar_rid, RF_ACTIVE);
	if (sc->bar_res == NULL) {
		device_printf(dev, "cannot allocate BAR0\n");
		error = ENXIO;
		goto fail_softc;
	}

	/* 步骤2：附着硬件层。 */
	error = myfirst_hw_attach_pci(sc, sc->bar_res,
	    rman_get_size(sc->bar_res));
	if (error != 0)
		goto fail_release_bar;

	/* 步骤3：设置中断。 */
	error = myfirst_intr_setup(sc);
	if (error != 0) {
		device_printf(dev, "interrupt setup failed (%d)\n", error);
		goto fail_hw;
	}

	/* 步骤4：创建cdev。 */
	sc->cdev = make_dev(&myfirst_cdevsw, sc->unit, UID_ROOT,
	    GID_WHEEL, 0600, "myfirst%d", sc->unit);
	if (sc->cdev == NULL) {
		error = ENXIO;
		goto fail_intr;
	}
	sc->cdev->si_drv1 = sc;

	/* 步骤5：注册sysctl。 */
	myfirst_intr_add_sysctls(sc);

	sc->pci_attached = true;
	return (0);

fail_intr:
	myfirst_intr_teardown(sc);
fail_hw:
	myfirst_hw_detach(sc);
fail_release_bar:
	bus_release_resource(dev, SYS_RES_MEMORY, sc->bar_rid, sc->bar_res);
	sc->bar_res = NULL;
fail_softc:
	myfirst_deinit_softc(sc);
	return (error);
}
```

PCI attach更短了；中断细节隐藏在`myfirst_intr_setup`后面。goto级联是四个标签而不是六个（中断特定标签移入了`myfirst_intr.c`）。

### 重构后的Detach

```c
static int
myfirst_pci_detach(device_t dev)
{
	struct myfirst_softc *sc = device_get_softc(dev);

	if (myfirst_is_busy(sc))
		return (EBUSY);

	sc->pci_attached = false;

	if (sc->cdev != NULL) {
		destroy_dev(sc->cdev);
		sc->cdev = NULL;
	}

	myfirst_intr_teardown(sc);

	if (sc->sim != NULL)
		myfirst_sim_detach(sc);

	myfirst_hw_detach(sc);

	if (sc->bar_res != NULL) {
		bus_release_resource(dev, SYS_RES_MEMORY, sc->bar_rid,
		    sc->bar_res);
		sc->bar_res = NULL;
	}

	myfirst_deinit_softc(sc);

	device_printf(dev, "detached\n");
	return (0);
}
```

中断特定的拆除是对`myfirst_intr_teardown`的一次调用，封装了掩码清除、拆除、排空和资源释放步骤。

### INTERRUPTS.md文档

新文档位于驱动程序源码旁边。其角色是向未来读者描述驱动程序的中断处理，无需他们阅读`myfirst_intr.c`：

```markdown
# myfirst 驱动程序中的中断处理

## 分配与设置

驱动程序通过 `bus_alloc_resource_any(9)` 配合 `SYS_RES_IRQ`、`rid = 0`、
`RF_SHAREABLE | RF_ACTIVE` 分配单个遗留PCI IRQ。过滤器处理程序通过
`bus_setup_intr(9)` 配合 `INTR_TYPE_MISC | INTR_MPSAFE` 注册。名为
"myfirst_intr" 的taskqueue创建，有一个工作线程，优先级为 `PI_NET`。

设置成功后，`INTR_MASK` 写入 `DATA_AV | ERROR | COMPLETE`，
使设备为这三个事件类断言线路。

## 过滤器处理程序

`myfirst_intr_filter(sc)` 读取 `INTR_STATUS`。如果为零，返回
`FILTER_STRAY`（共享IRQ防御）。否则检查三个识别位中的每一个，
原子递增每比特计数器，将比特写回 `INTR_STATUS` 以确认设备，
并（对于 `DATA_AV`）在taskqueue上入队 `intr_data_task`。

如果识别了任何位，过滤器返回 `FILTER_HANDLED`，否则返回
`FILTER_STRAY`。

## 延迟任务

`myfirst_intr_data_task_fn(sc, npending)` 在taskqueue工作线程的
线程上下文中运行。它获取 `sc->mtx`，读取 `DATA_OUT`，将值存储在
`sc->intr_last_data` 中，广播 `sc->data_cv` 以唤醒待处理读者，
并释放锁。

## 模拟中断 sysctl

`dev.myfirst.N.intr_simulate` 是只写的；向其写入位掩码会设置
`INTR_STATUS` 中的对应位并直接调用 `myfirst_intr_filter`。
这无需真实IRQ事件即可演练完整管道。

## 拆除

`myfirst_intr_teardown(sc)` 在detach期间运行。它清除
`INTR_MASK`，调用 `bus_teardown_intr`，排空并销毁taskqueue，
并释放IRQ资源。顺序严格：
先清除掩码再拆除（使杂散不累积），先拆除再排空（使无新任务入队发生），
先排空再释放（使无任务对已释放状态运行）。

## 中断上下文访问器宏

由于过滤器在主中断上下文运行，不能获取 `sc->mtx`。`myfirst_intr.h` 中
的两个宏隐藏了原始 `bus_read_4`/`bus_write_4` 调用而不断言任何锁：
`ICSR_READ_4` 和 `ICSR_WRITE_4`。仅在睡眠锁非法的上下文中使用。

## 已知限制

- 仅处理遗留PCI INTx线。MSI和MSI-X在第20章。
- 过滤器通过原子操作合并每比特计数器；任务以单一优先级运行。
  每队列或每优先级设计是后续章节的话题。
- 中断风暴检测由内核管理（`hw.intr_storm_threshold`）；
  驱动程序不实现自己的风暴缓解。
- 第17章模拟callout在PCI构建上不活跃；
  模拟中断sysctl是在bhyve实验室目标上驱动管道的方式。
```

五分钟阅读；对中断层形状的清晰画面。

### 回归通过

第19章回归是第18章的超集：

1. 编译干净。`make`成功；无警告。
2. 加载。`kldload ./myfirst.ko`成功；`dmesg`显示attach序列。
3. 附着到真实PCI设备。`devinfo -v`显示BAR和IRQ。
4. 无`[GIANT-LOCKED]`警告。
5. `vmstat -i | grep myfirst`显示`intr_event`。
6. `sysctl dev.myfirst.0.intr_count`从零开始。
7. 模拟中断。`sudo sysctl dev.myfirst.0.intr_simulate=1`；计数器递增；任务运行。
8. 速率测试。将`intr_sim_period_ms`设为100；10秒后检查计数器。
9. 分离。`devctl detach myfirst0`；`dmesg`显示干净detach。
10. 重新附着。`devctl attach pci0:0:4:0`；完整attach周期运行。
11. 卸载。`kldunload myfirst`；`vmstat -m | grep myfirst`显示零活跃分配；`vmstat -i | grep myfirst`无返回。

运行完整回归每次迭代需要一两分钟。在循环中运行二十次的CI作业是那种能捕获第20和21章扩展引入回归的防护。

### 重构完成了什么

第19章代码比没有重构少一个文件；一个新文档存在；版本号前进了一位。驱动程序是可辨认的FreeBSD风格，结构与`/usr/src/sys/dev/`中的生产驱动程序平行，准备好接受第20章的MSI-X机制和第21章的DMA机制而无需再次重组。

### 第8节收尾

重构遵循第16到18章建立的相同形状。新文件拥有新职责。新头文件导出公共接口。新文档解释行为。版本提升；回归通过；驱动保持可维护。无戏剧性；一点整理；一个可以构建的干净代码基础。

第19章的教学体完成。实验、挑战、故障排除、收尾和通往第20章的桥梁紧随其后。

## 收尾

第19章给了驱动程序耳朵。开始时，`myfirst`版本`1.1-pci`附着在真实PCI设备上但不倾听它：驱动程序采取的每个动作都由用户空间发起，设备自己的异步事件（如果有）未被注意。结束时，`myfirst`版本`1.2-intr`有连接到设备IRQ线的过滤器处理程序、在线程上下文处理大量工作的延迟任务管道、用于在实验室目标上测试的模拟中断路径、与同线上其他驱动共存的共享IRQ纪律、以正确顺序释放每个资源的干净拆除、以及新`myfirst_intr.c`文件加`INTERRUPTS.md`文档。

转换经历了八个节。第1节在硬件层面介绍中断，覆盖边沿触发和电平触发信号、CPU分发流程、以及驱动程序处理程序有六个义务。第2节介绍FreeBSD内核模型：`intr_event`、`intr_handler`、ithread、过滤器加ithread分离、`INTR_MPSAFE`、以及过滤器上下文约束。第3节编写最小过滤器和attach/detach连接。第4节用状态解码、每位确认和基于taskqueue延迟工作扩展过滤器。第5节添加模拟中断sysctl让读者无需真实IRQ事件即可演练管道。第6节编纂共享IRQ纪律：检查所有权、正确返回`FILTER_STRAY`、防御性处理未识别位。第7节整合拆除：在设备端掩码、拆除处理程序、排空taskqueue、释放资源。第8节将一切重构为可维护布局。

第19章没做的是MSI、MSI-X或DMA。驱动程序的中断路径是单个遗留IRQ；数据路径不使用DMA；延迟工作是单个taskqueue任务。第20章引入MSI和MSI-X（多个向量、每向量过滤器、更丰富的中断路由）。第20和21章引入DMA以及中断与DMA描述符环之间的交互。

第19章完成的是两个控制线程之间的分离。驱动程序的过滤器短、在主中断上下文运行、处理紧急每次中断工作。驱动程序的延迟任务较长、在线程上下文运行、处理大量每事件工作。保持它们协作的纪律（过滤器用原子、任务用睡眠锁、严格拆除顺序）是每个后续章节中断代码假设的纪律。

文件布局已增长：`myfirst.c`、`myfirst_hw.c`、`myfirst_hw_pci.c`、`myfirst_hw.h`、`myfirst_sim.c`、`myfirst_sim.h`、`myfirst_pci.c`、`myfirst_pci.h`、`myfirst_intr.c`、`myfirst_intr.h`、`myfirst_sync.h`、`cbuf.c`、`cbuf.h`、`myfirst.h`。文档已增长：`HARDWARE.md`、`LOCKING.md`、`SIMULATION.md`、`PCI.md`、`INTERRUPTS.md`。测试套件已增长：模拟中断管道、阶段4回归脚本、一些挑战练习让读者持续练习。

### 第20章之前的反思

下一章前的暂停。第19章教授了过滤器加任务模式、`INTR_MPSAFE`承诺、中断上下文约束和共享IRQ纪律。您在此练习的模式（读状态、确认、延迟工作、返回正确的`FILTER_*`、干净拆除）是每个FreeBSD中断处理程序使用的模式。第20章将在其上叠加MSI-X；第21章将在其上叠加DMA。两章都不替换第19章模式；都构建于它们之上。

值得提出的第二个观察。第17章模拟、第18章真实PCI attach和第19章中断处理的组合现在在架构意义上是完整驱动。理解三层的读者可以打开任何FreeBSD PCI驱动并识别部分：寄存器映射、PCI attach、中断过滤器。具体不同；结构恒定。该识别是使书籍投资在整个FreeBSD源码树回报的原因。

第三观察：第16章访问器层的回报持续。`CSR_*`宏在第19章未改变；`ICSR_*`宏为过滤器上下文使用添加，但它们调用相同底层`bus_read_4`和`bus_write_4`。抽象已三次回报：对抗第17章模拟后端、对抗第18章真实PCI BAR、对抗第19章过滤器上下文。在自己驱动中构建类似访问器层的读者会发现相同红利。

### 如果卡住了怎么办

三个建议。

首先，专注于模拟中断路径。如果`sudo sysctl dev.myfirst.0.intr_simulate=1`使计数器跳动并任务运行，管道在工作。章节的其他每块在装饰管道意义上可选，但如果管道失败，整章不工作，第5节是诊断的正确地方。

其次，打开`/usr/src/sys/dev/mgb/if_mgb.c`并缓慢重读`mgb_legacy_intr`函数。它约六十行过滤器代码。每行映射到第19章概念。完成章节后读一次应感觉是熟悉领域。

第三，首次通过跳过挑战。实验为第19章节奏校准；挑战假设章节材料稳固。如果现在感觉遥不可及，第20章后回来。

第19章目标是给驱动一种倾听其设备的方式。如果有了，第20章MSI-X机制成为特化而非全新话题，第21章DMA成为将描述符完成连接到已有中断路径的事情。

## 通往第20章的桥梁

第20章标题为*高级中断处理*。其范围是第19章刻意未采取的特化：MSI（消息信号中断）和MSI-X，现代PCIe中断机制，用作为内存写入交付的每设备（或每队列）向量取代遗留INTx线。

第19章以四种具体方式准备了基础。

首先，**您有一个正常工作的过滤器处理程序**。第19章过滤器读取状态、处理位、确认、延迟。第20章过滤器类似，但每向量复制：每个MSI-X向量有自己的过滤器，每个处理设备事件的特定子集。

其次，**您理解attach/detach级联**。第19章增长级联两个标签（`fail_release_irq`、`fail_teardown_intr`）。第20章进一步增长：每向量一对标签。模式不变；数量变化。

第三，**您有中断拆除纪律**。第20章重用第19章顺序：在设备端清除中断、每向量`bus_teardown_intr`、每向量`bus_release_resource`。每向量的性质添加小循环；顺序相同。

第四，**您有暴露MSI-X的实验室环境**。在带`virtio-rng-pci`的QEMU上，MSI-X可用；在带`virtio-rnd`的bhyve上，仅暴露遗留INTx。第20章实验可能需要切换到QEMU或更丰富模拟的bhyve设备以演练MSI-X路径。

第20章将覆盖的具体话题：

- 为什么MSI和MSI-X是对遗留INTx的改进。
- MSI与MSI-X的区别（单向量与向量表）。
- `pci_alloc_msi(9)`、`pci_alloc_msix(9)`：分配向量。
- `pci_msi_count(9)`、`pci_msix_count(9)`：查询能力。
- `pci_release_msi(9)`：拆除对应物。
- 多向量中断处理程序：每队列过滤器。
- MSI-X表布局及如何到达特定条目。
- 跨向量的CPU亲和用于NUMA感知。
- 中断合并：当设备支持时减少中断速率。
- MSI-X与iflib的交互（现代网络驱动框架）。
- 将`myfirst`驱动从第19章遗留路径迁移到MSI-X路径，对不支持MSI-X的设备回退到遗留。

不需要提前阅读。第19章准备充分。带上您的`myfirst`驱动`1.2-intr`、`LOCKING.md`、`INTERRUPTS.md`、启用`WITNESS`的内核和回归脚本。第20章从第19章结束的地方开始。

词汇是您的；结构是您的；纪律是您的。第20章向三者添加精度。



## 参考：第19章快速参考卡

第19章引入的词汇、API、宏和过程的紧凑摘要。

### 词汇

- **中断 (Interrupt)**：异步硬件信号事件。
- **IRQ (中断请求)**：中断线的标识符。
- **边沿触发 (Edge-triggered)**：由电平转换信号触发；每次转换一个中断。
- **电平触发 (Level-triggered)**：由保持的电平信号触发；电平保持期间持续触发中断。
- **intr_event**：FreeBSD 内核中代表一个中断源的结构。
- **ithread**：FreeBSD 运行延迟中断处理程序的内核线程。
- **过滤器处理程序 (filter handler)**：在主中断上下文中运行的函数。
- **ithread 处理程序 (ithread handler)**：过滤器之后在线程上下文中运行的函数。
- **FILTER_HANDLED**：过滤器已处理中断；不需要 ithread。
- **FILTER_SCHEDULE_THREAD**：过滤器已部分处理；运行 ithread。
- **FILTER_STRAY**：中断不属于此驱动程序。
- **INTR_MPSAFE**：承诺处理程序自行进行同步的标志。
- **INTR_TYPE_*** (TTY, BIO, NET, CAM, MISC, CLK, AV)：处理程序类别提示。
- **INTR_EXCL**：独占中断。

### 核心 API

- `bus_alloc_resource_any(dev, SYS_RES_IRQ, &rid, flags)`：声明 IRQ。
- `bus_release_resource(dev, SYS_RES_IRQ, rid, res)`：释放 IRQ。
- `bus_setup_intr(dev, res, flags, filter, ihand, arg, &cookie)`：注册处理程序。
- `bus_teardown_intr(dev, res, cookie)`：注销处理程序。
- `bus_describe_intr(dev, res, cookie, "name")`：为工具命名处理程序。
- `bus_bind_intr(dev, res, cpu)`：将中断路由到 CPU。
- `pci_msi_count(dev)`、`pci_msix_count(dev)`（第20章）。
- `pci_alloc_msi(dev, &count)`、`pci_alloc_msix(dev, &count)`（第20章）。
- `pci_release_msi(dev)`（第20章）。
- `taskqueue_create("name", M_WAITOK, taskqueue_thread_enqueue, &tq)`：创建 taskqueue。
- `taskqueue_start_threads(&tq, n, PI_pri, "thread name")`：启动工作线程。
- `taskqueue_enqueue(tq, &task)`：入队任务。
- `taskqueue_drain(tq, &task)`：等待任务完成，阻止新入队。
- `taskqueue_free(tq)`：释放 taskqueue。
- `TASK_INIT(&task, pri, fn, arg)`：初始化任务。

### 核心宏

- `FILTER_HANDLED`、`FILTER_STRAY`、`FILTER_SCHEDULE_THREAD`。
- `INTR_TYPE_TTY`、`INTR_TYPE_BIO`、`INTR_TYPE_NET`、`INTR_TYPE_CAM`、`INTR_TYPE_MISC`、`INTR_TYPE_CLK`、`INTR_TYPE_AV`。
- `INTR_MPSAFE`、`INTR_EXCL`。
- `RF_SHAREABLE`、`RF_ACTIVE`。
- `SYS_RES_IRQ`。

### 常用过程

**分配传统 PCI 中断并注册过滤器处理程序：**

1. `sc->irq_rid = 0;`
2. `sc->irq_res = bus_alloc_resource_any(dev, SYS_RES_IRQ, &sc->irq_rid, RF_SHAREABLE | RF_ACTIVE);`
3. `bus_setup_intr(dev, sc->irq_res, INTR_TYPE_MISC | INTR_MPSAFE, filter, NULL, sc, &sc->intr_cookie);`
4. `bus_describe_intr(dev, sc->irq_res, sc->intr_cookie, "name");`

**拆除中断处理程序：**

1. 在设备端禁用中断（清除 `INTR_MASK`）。
2. `bus_teardown_intr(dev, sc->irq_res, sc->intr_cookie);`
3. `taskqueue_drain(sc->intr_tq, &sc->intr_data_task);`
4. `taskqueue_free(sc->intr_tq);`
5. `bus_release_resource(dev, SYS_RES_IRQ, sc->irq_rid, sc->irq_res);`

**编写过滤器处理程序：**

1. 读取 `INTR_STATUS`；如果为零，返回 `FILTER_STRAY`。
2. 对于每个识别的位，递增计数器，通过写回确认，并有选择地入队任务。
3. 返回 `FILTER_HANDLED`（或 `FILTER_SCHEDULE_THREAD`），如果没有识别任何内容则返回 `FILTER_STRAY`。

### 常用命令

- `vmstat -i`：列出带计数的中断源。
- `devinfo -v`：列出设备及其 IRQ 资源。
- `sysctl hw.intrcnt` 和 `sysctl hw.intrnames`：原始计数器。
- `sysctl hw.intr_storm_threshold`：启用内核风暴检测。
- `cpuset -g`：查询中断 CPU 亲和性（平台特定）。
- `sudo sysctl dev.myfirst.0.intr_simulate=1`：触发模拟中断。

### 需要保持书签的文件

- `/usr/src/sys/sys/bus.h`：`driver_filter_t`、`driver_intr_t`、`FILTER_*`、`INTR_*`。
- `/usr/src/sys/kern/kern_intr.c`：内核的中断事件机制。
- `/usr/src/sys/sys/taskqueue.h`：taskqueue API。
- `/usr/src/sys/dev/mgb/if_mgb.c`：可读的过滤器加任务示例。
- `/usr/src/sys/dev/ath/if_ath_pci.c`：最小的仅 ithread 中断设置。



## 参考：第4部分对比表

第4部分每章的紧凑摘要，包括其位置、添加内容和前提假设。对跳入或回溯该部分的读者有用。

| 主题 | 第16章 | 第17章 | 第18章 | 第19章 | 第20章（预览） | 第21章（预览） |
|------|--------|--------|--------|--------|----------------|----------------|
| BAR 访问 | 用 malloc 模拟 | 用模拟层扩展 | 真实 PCI BAR | 相同 | 相同 | 相同 |
| 第17章模拟 | 不适用 | 引入 | PCI 上不活跃 | PCI 上不活跃 | PCI 上不活跃 | PCI 上不活跃 |
| PCI attach | 不适用 | 不适用 | 引入 | 相同 + IRQ | MSI-X 选项 | 添加 DMA 初始化 |
| 中断处理 | 不适用 | 不适用 | 不适用 | 引入 | MSI-X 每向量 | 完成驱动 |
| DMA | 不适用 | 不适用 | 不适用 | 不适用 | 预览 | 引入 |
| 版本 | 0.9-mmio | 1.0-simulated | 1.1-pci | 1.2-intr | 1.3-msi | 1.4-dma |
| 新文件 | `myfirst_hw.c` | `myfirst_sim.c` | `myfirst_pci.c` | `myfirst_intr.c` | `myfirst_msix.c` | `myfirst_dma.c` |
| 关键纪律 | 访问器抽象 | 伪设备 | Newbus attach | 过滤器/任务分离 | 每向量处理程序 | DMA 映射 |

该表使书的累积结构一目了然。理解给定主题行的读者可以预测第19章的工作如何融入更大的图景。



## 参考：第19章 FreeBSD 手册页

第19章材料最有用的手册页列表。在 FreeBSD 系统上使用 `man 9 <name>`（内核 API）或 `man 4 <name>`（子系统概述）打开每个页面。

### 内核 API 手册页

- **`bus_setup_intr(9)`**：注册中断处理程序。
- **`bus_teardown_intr(9)`**：拆除处理程序。
- **`bus_bind_intr(9)`**：绑定到 CPU。
- **`bus_describe_intr(9)`**：标记处理程序。
- **`bus_alloc_resource(9)`**：资源分配（通用）。
- **`bus_release_resource(9)`**：资源释放。
- **`atomic(9)`**：原子操作，包括 `atomic_add_64`。
- **`taskqueue(9)`**：taskqueue 原语。
- **`ppsratecheck(9)`**：速率限制日志辅助函数。
- **`swi_add(9)`**：软件中断（作为替代方案提及）。
- **`intr_event(9)`**：中断事件机制（如果存在；某些 API 是内部的）。

### 设备子系统手册页

- **`pci(4)`**：PCI 子系统。
- **`vmstat(8)`**：`vmstat -i` 用于观察中断。
- **`devinfo(8)`**：设备树和资源。
- **`devctl(8)`**：运行时设备控制。
- **`sysctl(8)`**：读写 sysctl。
- **`dtrace(1)`**：动态追踪。

其中大多数已在章节正文中引用。这个汇总列表供想要一个位置找到它们的读者使用。



## 参考：驱动程序记忆短语

几句总结第19章纪律的格言。适用于阅读和代码审查。

- **"读取、确认、延迟、返回。"** 过滤器做的四件事。
- **"如果没有识别任何内容则返回 FILTER_STRAY。"** 共享 IRQ 协议。
- **"拆除前先掩码；释放前先拆除。"** detach 顺序。
- **"过滤器上下文仅限自旋锁。"** 无睡眠锁规则。
- **"每次入队在释放前都需要排空。"** taskqueue 生命周期。
- **"一个过滤器，一个设备，一个状态。"** 保持每设备代码清醒的隔离。
- **"如果 WITNESS 崩溃，相信它。"** 调试内核捕获微妙的错误。
- **"先 PROD，后中断。"** 在启用处理程序之前编程设备（`INTR_MASK`）。
- **"过滤器中做小的；任务中做大的。"** 工作量纪律。
- **"风暴检测是安全网，不是设计工具。"** 不要依赖内核的节流。

这些都不是完整的规范。每条都是展开为章节详细处理的紧凑提醒。



## 参考：第19章术语词汇表

**ack (确认)**：写回 INTR_STATUS 以清除待处理位并取消 IRQ 线断言的操作。

**driver_filter_t**：过滤器处理函数的 C 类型定义：`int f(void *)`。

**driver_intr_t**：ithread 处理函数的 C 类型定义：`void f(void *)`。

**边沿触发 (edge-triggered)**：由电平转换信号的中断信号模式。

**FILTER_HANDLED**：过滤器返回值，表示"此中断已处理；不需要 ithread"。

**FILTER_SCHEDULE_THREAD**：返回值，表示"调度 ithread 运行"。

**FILTER_STRAY**：返回值，表示"此中断不属于此驱动程序"。

**过滤器处理程序 (filter handler)**：在主中断上下文中运行的 C 函数。

**Giant**：旧式单一全局内核锁；现代驱动通过设置 INTR_MPSAFE 避免它。

**IE (中断事件)**：`intr_event` 的简称。

**INTR_MPSAFE**：承诺处理程序自行进行同步且在没有 Giant 的情况下安全的标志。

**INTR_STATUS**：跟踪待处理中断原因的设备寄存器（RW1C）。

**INTR_MASK**：启用特定中断类别的设备寄存器。

**intr_event**：代表一个中断源的内核结构。

**ithread**：内核中断线程；在线程上下文中运行延迟处理程序。

**电平触发 (level-triggered)**：电平保持期间持续触发中断的中断信号模式。

**MSI**：消息信号中断；PCIe 机制（第20章）。

**MSI-X**：MSI 的更丰富变体，带有向量表（第20章）。

**主中断上下文 (primary interrupt context)**：过滤器处理程序的上下文；无睡眠，无睡眠锁。

**PCIR_INTLINE / PCIR_INTPIN**：指定传统 IRQ 线和引脚的 PCI 配置空间字段。

**RF_ACTIVE**：资源分配标志；一步激活资源。

**RF_SHAREABLE**：资源分配标志；允许与其他驱动程序共享资源。

**杂散中断 (stray interrupt)**：没有过滤器返回声明的中断；由内核单独计数。

**风暴 (storm)**：电平触发中断因驱动程序未确认而持续触发的情况。

**SYS_RES_IRQ**：中断的资源类型。

**taskqueue**：用于在线程上下文中运行延迟工作的内核原语。

**陷阱存根 (trap stub)**：CPU 接受中断向量时运行的一小段内核代码。

**EOI (中断结束)**：发送到中断控制器以重新武装 IRQ 线的信号。



## 参考：关于中断处理理念的结束语

一段值得在实验后返回阅读的结束语。

中断处理程序的工作不是做设备的工作。设备的工作（处理数据包、完成 I/O、读取传感器）由驱动程序的其余部分完成，在线程上下文中，在驱动程序完整的锁集合保护下。处理程序的工作范围更窄：注意到设备有话要说，确认设备以便对话可以继续，调度稍后将发生的实际工作，并足够快地返回以使 CPU 可以释放给被中断的线程或下一个中断。

编写了第19章驱动程序的读者已经编写了一个中断处理程序。它很小。驱动程序的其余部分才是让它有用的部分。第20章将把处理程序特化为 MSI-X 上的每向量工作。第21章将把任务特化为遍历 DMA 描述符环。这两个都是扩展，而不是替换。第19章的处理程序是两者构建的骨架。

第19章教授的技能不是"如何为 virtio-rnd 设备处理中断"。它是"如何在主上下文和线程上下文之间分配工作，如何尊重过滤器的约束，如何干净地拆除，以及如何与共享线上的其他驱动程序合作"。其中每一项都是可迁移的技能。FreeBSD 源码树中的每个驱动程序都使用了其中一些；大多数驱动程序使用了全部。

对于这位读者以及本书未来的读者，第19章的过滤器和任务是 `myfirst` 驱动程序架构的永久组成部分。每个后续章节都假设它们。每个后续章节都扩展它们。驱动程序的整体复杂度将增长，但中断路径将保持第19章创造的样子：一段狭窄、快速、正确排序的代码，它让出位置以便驱动程序的其余部分可以完成工作。


## 一起阅读真实驱动程序：mgb(4)的中断路径

在实验之前，简短地走一遍使用第19章教授的相同过滤器加任务模式的真实驱动程序。`/usr/src/sys/dev/mgb/if_mgb.c` 是Microchip LAN743x千兆以太网控制器的FreeBSD驱动程序。它可读性强，是生产级质量的，其中断处理大约是第19章词汇覆盖的复杂度水平。

本节遍历`mgb_legacy_intr`和设置代码的中断相关部分，并标记每部分对应的第19章概念。

### 传统过滤器

`mgb(4)`传统IRQ路径的过滤器处理程序：

```c
int
mgb_legacy_intr(void *xsc)
{
	struct mgb_softc *sc;
	if_softc_ctx_t scctx;
	uint32_t intr_sts, intr_en;
	int qidx;

	sc = xsc;
	scctx = iflib_get_softc_ctx(sc->ctx);

	intr_sts = CSR_READ_REG(sc, MGB_INTR_STS);
	intr_en = CSR_READ_REG(sc, MGB_INTR_ENBL_SET);
	intr_sts &= intr_en;

	/* TODO: shouldn't continue if suspended */
	if ((intr_sts & MGB_INTR_STS_ANY) == 0)
		return (FILTER_STRAY);

	if ((intr_sts &  MGB_INTR_STS_TEST) != 0) {
		sc->isr_test_flag = true;
		CSR_WRITE_REG(sc, MGB_INTR_STS, MGB_INTR_STS_TEST);
		return (FILTER_HANDLED);
	}
	if ((intr_sts & MGB_INTR_STS_RX_ANY) != 0) {
		for (qidx = 0; qidx < scctx->isc_nrxqsets; qidx++) {
			if ((intr_sts & MGB_INTR_STS_RX(qidx))){
				iflib_rx_intr_deferred(sc->ctx, qidx);
			}
		}
		return (FILTER_HANDLED);
	}
	if ((intr_sts & MGB_INTR_STS_TX_ANY) != 0) {
		for (qidx = 0; qidx < scctx->isc_ntxqsets; qidx++) {
			if ((intr_sts & MGB_INTR_STS_RX(qidx))) {
				CSR_WRITE_REG(sc, MGB_INTR_ENBL_CLR,
				    MGB_INTR_STS_TX(qidx));
				CSR_WRITE_REG(sc, MGB_INTR_STS,
				    MGB_INTR_STS_TX(qidx));
				iflib_tx_intr_deferred(sc->ctx, qidx);
			}
		}
		return (FILTER_HANDLED);
	}

	return (FILTER_SCHEDULE_THREAD);
}
```

逐步分析。过滤器读取两个寄存器（`INTR_STS`和`INTR_ENBL_SET`），将它们AND运算得到已启用中断的待处理子集，并检查是否有位被设置。如果没有，返回`FILTER_STRAY`，这是第19章共享IRQ的规则。

对于每类中断（测试、接收、发送），过滤器确认`INTR_STS`中的相关位（通过写回它们）并调度延迟处理。`iflib_rx_intr_deferred`是iflib框架调度接收队列工作的方式；概念上它与第19章的`taskqueue_enqueue`相同。

值得注意的一点：测试中断处理程序写入`INTR_STS`但也设置了一个标志（`sc->isr_test_flag = true`）。这是驱动程序向用户空间代码（通过sysctl或ioctl）发送信号表示测试中断已触发的方式。第19章的等价物是`intr_count`计数器。

最后的返回是`FILTER_SCHEDULE_THREAD`。如果没有任何特定位类别匹配但`MGB_INTR_STS_ANY`匹配了，就会触发这个。ithread处理剩余情况。第19章的驱动程序没有这个特定的穿透逻辑，因为它没有注册ithread；`mgb(4)`有。

### mgb过滤器教给我们什么

三点教训直接适用于第19章的过滤器：

1. **读取-AND-掩码。** `intr_sts & intr_en`确保过滤器只报告实际被启用的中断。设备内部可能报告驱动程序已掩码掉的事件；AND运算将它们过滤掉。
2. **每位的确认。** 每个位类别被单独确认（通过写回特定位）。过滤器不会写`0xffffffff`；它只写入处理的位。
3. **每队列延迟工作。** 每个接收队列和发送队列都有自己的延迟路径。第19章更简单的驱动程序有一个任务；`mgb(4)`的多队列驱动程序有很多。

### mgb中的中断设置

在`if_mgb.c`中搜索`bus_setup_intr`可以看到几个调用点，一个用于传统IRQ路径，一个用于每个MSI-X向量：

```c
if (bus_setup_intr(sc->dev, sc->irq[0], INTR_TYPE_NET | INTR_MPSAFE,
    mgb_legacy_intr, NULL, sc, &sc->irq_tag[0]) != 0) {
	/* ... */
}
```

模式完全就是第19章的：过滤器处理程序，无ithread，`INTR_MPSAFE`，softc作为参数，返回cookie。唯一的区别是`INTR_TYPE_NET`而不是`INTR_TYPE_MISC`（驱动程序针对网络）。

### mgb中的中断拆除

拆除模式分布在`iflib(9)`的助手中，它们为驱动程序处理排空和释放。iflib框架之外的定制驱动程序会显式执行拆除；第19章的驱动程序就是显式执行的。

### 这次遍历教给我们什么

`mgb(4)`的中断路径不是玩具。它是生产级的实现，使用第19章驱动程序遵循的相同模式。能够理解并阅读`mgb_legacy_intr`的读者已经内化了第19章的词汇。该文件是免费提供的；阅读周围代码（attach路径、ithread、iflib集成）会进一步加深理解。

在`mgb(4)`之后值得阅读的：`/usr/src/sys/dev/e1000/em_txrx.c`用于MSI-X多向量模式（第20章材料），`/usr/src/sys/dev/usb/controller/xhci_pci.c`用于USB主控制器的中断路径（第21章+），以及`/usr/src/sys/dev/ahci/ahci_pci.c`用于存储控制器的中断路径。



## 深入理解中断上下文

第2节列出了过滤器中可以和不可以做什么。本节深入一级，解释每个规则存在的原因。理解"为什么"的读者可以推理不熟悉的约束（新锁、新内核API）并预测它们是否是过滤器安全的。

### 栈情况

当CPU接受中断时，它保存被打断线程的寄存器并切换到内核中断栈。中断栈很小（几KB，与平台相关），是每CPU的，并在该CPU上的所有中断之间共享。过滤器就在这个栈上运行。

两个推论：

首先，过滤器的栈空间有限。在栈上分配大数组（几百字节或更多）的过滤器可能溢出中断栈。症状通常是panic，有时是栈旁边内存的无声损坏。规则是：过滤器的栈预算很小。大数组属于任务。

其次，栈在所有中断之间共享。如果过滤器睡眠（理论上可以，虽然不能），它会占用栈；同一CPU上的其他中断无法重用它。即使允许睡眠，也不是免费的。小栈约束是过滤器必须短小的原因之一。

### 为什么不能使用睡眠锁

睡眠互斥锁（默认的`mtx(9)`）可能阻塞：如果另一个线程持有该互斥锁，`mtx_lock`让调用线程在互斥锁上睡眠。在过滤器的上下文中：

- 没有传统意义上的调用"线程"。被打断的线程暂停在指令中间；过滤器是CPU上的内核侧偏移。
- 从中断中睡眠会停顿CPU：中断栈被占用，其他中断无法在此CPU上运行，调度器在没有工作内核状态的情况下无法轻易调度不同的线程。

内核原则上可以处理这种情况（有些内核确实这样做）。FreeBSD的设计是禁止它。该禁令由调试内核上的`WITNESS`强制执行：任何在中断上下文中获取睡眠锁的尝试都会产生立即的panic。

自旋互斥锁（`MTX_SPIN`互斥锁）是安全的，因为它们不睡眠；它们只是自旋。获取自旋互斥锁的过滤器没问题。

### 为什么不能使用睡眠malloc

`malloc(M_WAITOK)`调用VM页面分配器，如果系统内存不足可能会睡眠。与锁相同的问题：调用者不能被暂停。`malloc(M_NOWAIT)`是替代方案；它可能失败，但从不对眠。

在过滤器中，唯一安全的选择是`M_NOWAIT`、UMA区域（有自己有界分配器），或预分配的缓冲区。第19章的驱动程序完全不在过滤器中分配内存；过滤器需要的所有内存都在softc中，在attach期间预分配。

### 为什么不能使用条件变量

`cv_wait`和`cv_timedwait`睡眠。过滤器不能睡眠。`cv_signal`和`cv_broadcast`不睡眠，但它们在大多数用法中内部获取睡眠互斥锁；使用它们的过滤器必须小心。第19章任务处理`cv_broadcast`；过滤器只是入队任务。

### 为什么过滤器不能重入

内核的中断分发在过滤器运行时禁用该CPU上的后续中断（在大多数平台上；某些架构使用优先级级别代替）。这意味着过滤器不能递归触发自己，即使其设备在执行期间断言线路。任何此类断言都会被排队并在过滤器返回后触发。

一个推论：过滤器不需要内部重入保护。单个`sc->intr_count++`从过滤器的角度对同一CPU上的同时过滤器调用是安全的。它仍可能与其他代码（任务、用户空间读取）竞争，这就是为什么使用原子操作，但过滤器不会与自己竞争。

### 为什么原子操作是安全的

FreeBSD中的原子操作实现为CPU指令，根据定义是原子的。它们不获取锁；它们不睡眠；它们不阻塞。它们在每个上下文中都是安全的，包括过滤器。

第19章广泛使用`atomic_add_64`：用于中断计数器、每个位的计数器以及任务调用计数。操作便宜（几个周期）且可预测（不涉及调度）。

### 为什么ithread获得更多宽松度

ithread在线程上下文中运行。它有：

- 自己的栈（正常的内核线程栈，比中断栈大得多）。
- 自己的调度优先级（提升的，但仍是正常的线程）。
- 如果调度器决定，可以睡眠的能力。

它的约束是通常的线程上下文规则：按顺序持有睡眠锁（防止死锁），避免在调用无界代码时持有锁，如果分配可能失败则使用`M_WAITOK`，等等。第13和15章的规则适用。

ithread的提升优先级意味着它不应该任意长时间阻塞。阻塞微秒（短暂的互斥锁竞争）没问题；阻塞数秒会使系统上的每个其他ithread饥饿。

### 为什么taskqueue线程有更多宽松度

taskqueue的工作线程是普通的内核线程，通常在正常优先级（或驱动程序在`taskqueue_start_threads`中指定的优先级）。它可以睡眠，可以阻塞在任何睡眠锁上，可以任意分配。它是三者中最灵活的。

权衡是taskqueue工作不如ithread工作及时。taskqueue的工作线程可能不会立即运行；由调度器决定。对于延迟关键的工作，ithread更好；对于大量工作，taskqueue更简单。

第19章的驱动程序使用taskqueue，因为它做的大量工作（读取`DATA_OUT`、更新softc、广播条件变量）不是延迟关键的。第20章和21章的驱动程序可能根据其工作负载以不同方式选择ithread或taskqueue。

### 中断上下文如何与锁定交互

第3部分引入的锁定规则仍然适用，但有一个新增：知道在每个上下文中可以获取哪些锁。

**过滤器上下文。** 只有自旋锁、原子操作、无锁算法。没有睡眠互斥锁、sx锁、rwlock，或没有`MTX_SPIN`初始化的`mtx`。

**ithread上下文。** 所有锁类型。遵守项目的锁顺序，如`LOCKING.md`所定义。

**taskqueue工作线程上下文。** 所有锁类型。遵守项目的锁顺序。如果需要可以任意睡眠（虽然驱动程序作者不应该这样做）。

**一般线程上下文（cdev open/read/write/ioctl、sysctl处理程序）。** 所有锁类型。遵守项目的锁顺序。

对于第19章的驱动程序，过滤器不获取锁（使用原子操作），任务通过`MYFIRST_LOCK`获取`sc->mtx`（睡眠互斥锁），sysctl处理程序也获取`sc->mtx`。规则被遵守。

### "Giant"脚注

较旧的BSD使用称为Giant的单一全局锁来串行化整个内核。当FreeBSD在1990年代末和2000年代初引入SMPng（细粒度锁定）时，内核的大部分被转换，但一些传统路径仍持有Giant。未设置`INTR_MPSAFE`的驱动程序会自动在它们的处理程序周围获取Giant；`WITNESS`可能会抱怨涉及Giant的锁顺序问题。

第19章的驱动程序设置`INTR_MPSAFE`并不触及Giant。现代FreeBSD驱动程序约定不鼓励驱动程序代码中的Giant。这个脚注存在是因为如果在`kern_intr.c`中搜索"Giant"，读者会找到引用；它们是向后兼容性产物。



## 深入理解中断CPU亲和

第2节简要介绍的CPU亲和附录。深入处理属于第20章（当有多个MSI-X向量可用时）和后续可扩展性章节；第19章的覆盖是一个起点。

### 亲和意味着什么

中断亲和是允许中断触发的CPU集合。对于单CPU系统，亲和是简单的（一个CPU）。对于多CPU系统，亲和变得有趣：将中断路由到特定CPU（而不是让中断控制器选择）可以改善缓存局部性、减少跨插槽流量，并与线程放置对齐。

在x86上，I/O APIC每个IRQ有一个可编程的目标字段；内核使用它来路由IRQ。在arm64上，GIC有类似的设施。FreeBSD的`bus_bind_intr(9)`是配置特定IRQ资源亲和的可移植API。

### 默认行为

没有显式绑定，FreeBSD使用轮询或特定平台的算法在CPU之间分散中断。对于像第19章这样的单中断驱动程序，这通常意味着中断在引导时内核决定的任何CPU上触发。当前亲和通过`cpuset -g -x <irq>`可见；特定IRQ的每CPU触发细分不是`vmstat -i`默认输出的一部分（它将`intr_event`的所有触发聚合为一个计数），但在平台支持时可以从内核工具重建。

对于许多驱动程序，默认值是好的。中断率足够低，亲和不重要，或者工作足够短，跨CPU成本可忽略。第19章的驱动程序属于这一类。

### 当亲和重要时

三种驱动程序作者想要显式亲和的场景：

1. **高中断率。** 处理十吉比特流量的NIC每秒触发数万次中断。在CPU之间移动中断工作变成真实成本。将每个接收队列的MSI-X向量绑定到特定CPU保持其缓存行热度。
2. **NUMA局部性。** 在多插槽系统上，设备的PCIe根复合体物理连接到一个插槽。来自该设备的中断在靠近根复合体的同一NUMA节点中的CPU上处理更便宜。放置对延迟和吞吐量都重要。
3. **实时约束.** 需要特定CPU上低延迟响应的系统（用于实时应用）可能会将常规中断钉扎远离这些CPU。`bus_bind_intr`让驱动程序参与这种分区。

### bus_bind_intr API

函数签名：

```c
int bus_bind_intr(device_t dev, struct resource *r, int cpu);
```

`cpu`是范围0到`mp_ncpus - 1`的整数CPU ID。成功时，中断被路由到该CPU。失败时，函数返回errno（最常见的是`EINVAL`，如果平台不支持重新绑定或CPU无效）。

该调用在`bus_setup_intr`之后：

```c
error = bus_setup_intr(dev, irq_res, flags, filter, ihand, arg,
    &cookie);
if (error == 0)
	bus_bind_intr(dev, irq_res, preferred_cpu);
```

第19章的驱动程序不绑定其中断。挑战练习添加了一个sysctl让操作者设置首选CPU。

### 内核的CPU集抽象

更复杂的API：`bus_get_cpus(9)`让驱动程序查询哪些CPU被视为设备的"局部"，对于想要在设备局部NUMA节点的CPU子集中分散中断的多队列驱动程序很有用。`/usr/src/sys/sys/bus.h`中的`LOCAL_CPUS`和`INTR_CPUS` cpusets暴露此信息。

第20章的MSI-X工作将使用`bus_get_cpus(9)`将每队列中断放在设备局部NUMA节点的不同CPU上。第19章的单中断驱动程序不需要这种复杂性。

### 观察亲和

`cpuset -g -x <irq>`命令显示IRQ的当前CPU掩码。对于多CPU系统上的`myfirst`驱动程序，从`devinfo -v | grep -A 5 myfirst0`获取IRQ号，使用`cpuset -l 1 -x <irq>`将中断绑定到（比如说）CPU 1，并用`cpuset -g -x <irq>`确认。

细节与平台相关。在x86上，I/O APIC（或MSI路由）实现请求；在arm64上，GIC重分发器实现。某些架构拒绝重新绑定并返回错误；合作的驱动程序的`bus_bind_intr`调用将此视为非致命提示。



## 深入理解中断风暴检测

FreeBSD内核有内置保护机制，防止特定故障模式：因为驱动程序无法确认而持续触发的电平触发IRQ。该保护称为中断风暴检测，在`/usr/src/sys/kern/kern_intr.c`中实现，由单个sysctl控制。

### hw.intr_storm_threshold Sysctl

```c
static int intr_storm_threshold = 0;
SYSCTL_INT(_hw, OID_AUTO, intr_storm_threshold, CTLFLAG_RWTUN,
    &intr_storm_threshold, 0,
    "Number of consecutive interrupts before storm protection is enabled");
```

默认值为零（风暴检测禁用）。将sysctl设置为正值启用检测：如果`intr_event`在没有任何其他中断发生在同一CPU上的情况下连续传递超过N次中断，内核假设是风暴并节流该事件。

节流意味着内核在再次运行处理程序之前暂停（通过`pause("istorm", 1)`）。暂停是单个时钟tick，在大多数系统上约一毫秒。效果是限制风暴源消耗CPU的速率。

### 何时启用检测

默认关闭是生产设置。启用风暴检测意味着内核在认为发生风暴时暂停中断；如果检测错误（例如，高速率合法中断，比如10吉比特NIC），暂停是性能bug。

对于驱动程序开发，启用风暴检测是有用的：过滤器中忘记确认会产生中断风暴，内核会检测到并节流它（并记录到`dmesg`）。没有检测，风暴会永远消耗一个CPU；有了检测，风暴可见并被节流。

合理的开发时设置是`hw.intr_storm_threshold=1000`。同一事件上一千次连续中断而不交错对于合法流量是不寻常的，能可靠标记风暴。

### 风暴的样子

在`dmesg`中：

```text
interrupt storm detected on "irq19: myfirst0:legacy"; throttling interrupt source
```

以速率限制间隔重复（默认每秒一次，由内核风暴代码内的`ppsratecheck`控制）。中断源被命名；驱动程序可以从名称识别。

内核不会永久禁用该线路；它只是控制处理程序的节奏。风暴结束后（可能因为驱动程序被卸载或设备停止断言），处理程序以全速恢复。

### 驱动程序侧风暴缓解

驱动程序可以实现自己的风暴缓解。经典技术是：

1. 在滑动窗口中计数中断。
2. 如果速率超过阈值，通过`INTR_MASK`屏蔽设备的中断并调度任务稍后重新启用。
3. 在任务中，检查设备，清除导致风暴的原因，然后重新启用。

这比内核默认更激进。大多数驱动程序不实现它。第19章驱动程序不实现；内核的阈值对于章节练习的场景足够。

### 与共享IRQ的关系

在共享IRQ线路上，一个驱动程序的风暴可能干扰另一个驱动程序的合法中断。内核的风暴检测是每事件的，不是每处理程序的，所以如果一个驱动程序的处理程序缓慢或错误，整个事件被节流。这是编写正确过滤器的有力论据：风暴影响不限于有bug的驱动程序。



## 过滤器与ithread选择的思维模型

初学者常在选择仅过滤器、过滤器加ithread、过滤器加taskqueue、仅ithread之间挣扎。本节提供在大多数情况下有帮助的决策框架，基于驱动程序作者可以针对其特定设备回答的问题。

### 四个问题

关于中断的工作问这些问题：

1. **每部分工作都可以在主上下文中完成吗？** 如果是（所有状态访问通过自旋锁或原子操作；所有确认通过BAR写入；无睡眠），仅过滤器是最干净的选择。
2. **是否有任何部分工作需要睡眠锁或条件变量广播？** 如果是，大量工作必须在线程上下文中完成。选择在于ithread和taskqueue。
3. **延迟工作是否从中断以外的任何地方调度？** 如果是（sysctl处理程序、ioctl、定时器callout、其他任务），taskqueue更好。相同工作可以从任何上下文调度。
4. **延迟工作是否对中断的优先级类敏感？** 如果是（您希望`INTR_TYPE_NET`的ithread优先级用于网络工作），注册ithread处理程序。ithread继承中断的优先级；taskqueue在其工作线程创建时获得任何优先级。

### 应用框架

**仅过滤器适合：**
- 计数器递增演示驱动程序。
- 仅读取设备寄存器并通过原子传递值的驱动程序。
- 数据很少产生并被直接读取的非常简单的传感器。

**过滤器加ithread适合：**
- 延迟工作仅在中断时有意义的简单驱动程序。
- 受益于中断优先级类的驱动程序。
- 想要内核管理的ithread而没有taskqueue额外机制的驱动程序。

**过滤器加taskqueue适合：**
- 相同延迟工作可由多个源触发（中断、sysctl、ioctl）的驱动程序。
- 想要合并中断的驱动程序（taskqueue的`npending`计数告诉自上次运行以来发生了多少次入队）。
- 想要特定工作线程数或优先级独立于中断类别的驱动程序。
- 第19章的目标情况：`myfirst`驱动程序从过滤器和模拟中断sysctl调度相同的任务。

**仅ithread适合：**
- 没有紧急工作且每个动作都需要睡眠锁的驱动程序。
- 过滤器会是平凡的（只是"调度线程"）的驱动程序；注册无过滤器和让内核调度ithread节省一次函数调用。

### 实例：假设存储驱动程序

假设您正在为小型存储控制器编写驱动程序。设备有一条IRQ线。当I/O完成时，它设置`INTR_STATUS.COMPLETION`并在完成队列寄存器中列出完成的命令ID。

决策：

- **每部分工作都可以在主上下文中完成吗？** 不。唤醒发起I/O的线程需要条件变量广播，这需要线程的锁。过滤器不能持有该锁。
- **哪种延迟机制？** 完成处理工作仅由中断调度，所以过滤器加ithread是干净的。优先级类是`INTR_TYPE_BIO`，ithread继承它。
- **最终设计。** 过滤器读取`INTR_STATUS`，将完成的命令ID提取到每中断上下文队列中，确认，返回`FILTER_SCHEDULE_THREAD`。ithread遍历每上下文队列，将命令ID匹配到待处理请求，唤醒每个请求的线程。

### 实例：假设网络驱动程序

一个有四个MSI-X向量（两个接收队列、两个发送队列）的NIC。每个向量有自己的过滤器。

决策：

- **过滤器工作？** 每队列：确认、标记哪些队列有事件。
- **延迟工作？** 每队列：遍历描述符环、构建mbuf、传递到栈。
- **多个源？** 正常操作仅由中断；轮询模式（用于高负载卸载）是第二个源。taskqueue更好：过滤器和轮询模式定时器都可以入队。
- **优先级？** `INTR_TYPE_NET`，与taskqueue工作线程的`PI_NET`优先级匹配。
- **最终设计。** 每向量过滤器入队每队列任务后返回`FILTER_HANDLED`。每个接收队列一个taskqueue，各一个工作线程。taskqueue配置为优先级`PI_NET`。

### 实例：第19章驱动程序

一条IRQ线，简单事件类型，基于taskqueue的延迟工作。

- **过滤器工作：** 读取`INTR_STATUS`，每位的计数器，确认，为DATA_AV入队任务。
- **延迟工作：** 读取`DATA_OUT`，更新softc，广播`data_cv`。
- **多个源？** 过滤器和模拟中断sysctl都需要任务。taskqueue是正确的。
- **优先级？** `PI_NET`是合理的默认值，即使驱动程序不是NIC；模拟框架期望响应性。

### 何时重新审视决策

决策不是永久的。以仅过滤器开始的驱动程序可能在获得新能力时增加任务；以taskqueue开始的驱动程序可能在taskqueue的额外灵活性不需要时移至ithread。重构通常很小（半小时的代码移动）。

框架帮助您避免明显错误的初始选择。细节是驱动程序作者根据特定设备做出的判断调用。



## 锁顺序与中断路径

第3部分的锁规则引入了驱动程序有固定锁顺序的想法：`sc->mtx -> sc->cfg_sx -> sc->stats_cache_sx`。第19章没有添加新锁，但确实添加了触及现有锁的新上下文。本小节检查第19章添加是否遵守现有顺序。

### 过滤器不获取锁

过滤器读取`INTR_STATUS`，更新原子计数器，入队任务。没有获取睡眠锁。过滤器对`INTR_STATUS`的访问使用`ICSR_READ_4`和`ICSR_WRITE_4`，它们不断言任何锁。因此过滤器不参与锁顺序；它是无锁的。

这是最简单的可能选择。更复杂的过滤器可能使用自旋锁（保护小的共享数据结构）；第19章的过滤器比那更简单。

### 任务获取sc->mtx

任务的函数`myfirst_intr_data_task_fn`获取`sc->mtx`（通过`MYFIRST_LOCK`），做其工作，释放。它不获取任何其他锁。因此任务通过不引入任何新锁获取模式来遵守现有锁顺序。

### 模拟中断sysctl获取并释放sc->mtx

sysctl处理程序获取`sc->mtx`以设置`INTR_STATUS`，释放锁，然后调用过滤器。这不是锁顺序违规，因为过滤器不获取锁；没有新边被添加到锁图。

### Attach和Detach路径

attach路径短暂获取`sc->mtx`以设置`INTR_MASK`并执行初始诊断读取。它不在`bus_setup_intr`期间持有锁（这可能原则上调用持有自己锁的其他内核部分；`bus_setup_intr`记录为可锁定，意味着调用者可以不持有自己的锁）。detach路径类似地短暂持有`sc->mtx`在`INTR_MASK`清除周围，然后在调用`bus_teardown_intr`之前释放。

### 微妙的顺序问题：bus_teardown_intr可能阻塞

值得指出的细节。`bus_teardown_intr`等待任何进行中的过滤器或ithread调用完成后返回。如果驱动程序持有过滤器需要的锁（比如，过滤器短暂获取的自旋锁），`bus_teardown_intr`可能永远阻塞，因为过滤器无法运行完成。

第19章的过滤器不获取自旋锁，所以此问题是学术性的。但在过滤器中使用自旋锁的驱动程序必须小心：在调用`bus_teardown_intr`时绝不持有过滤器的自旋锁。

### WITNESS与中断路径

调试内核的`WITNESS`跟踪每个上下文的锁顺序，包括过滤器。获取自旋锁的过滤器在`WITNESS`的图中创建顺序边。如果任何线程上下文代码在持有另一个自旋锁时获取相同自旋锁，`WITNESS`标记潜在死锁。

对于第19章驱动程序，没有边被添加。`WITNESS`是静默的。

### 在LOCKING.md中记录什么

良好驱动程序的`LOCKING.md`清楚记录锁顺序。第19章的添加很小：

- 过滤器不获取锁（仅原子操作）。
- 任务获取`sc->mtx`（现有顺序的叶子）。
- 模拟中断sysctl短暂获取`sc->mtx`以设置状态，释放，然后调用过滤器（在任何锁之外）。

`LOCKING.md`中的短段落记录这些事实。顺序本身不变。



## 可观察性：第19章向操作者暴露什么

关于中断的章节间接地也是关于可观察性的。驱动程序的用户（系统操作者或调试问题的驱动程序作者）想要看到驱动程序在做什么。第19章通过计数器和模拟中断sysctl暴露适度的可观察性；本小节整合什么是可见的以及如何看到。

### 计数器套件

第4阶段后，驱动程序暴露这些只读sysctl：

- `dev.myfirst.N.intr_count`：过滤器总调用次数。
- `dev.myfirst.N.intr_data_av_count`：DATA_AV事件。
- `dev.myfirst.N.intr_error_count`：ERROR事件。
- `dev.myfirst.N.intr_complete_count`：COMPLETE事件。
- `dev.myfirst.N.intr_task_invocations`：任务运行次数。
- `dev.myfirst.N.intr_last_data`：任务最近读取的DATA_OUT。

计数器给出中断活动的简明视图。随时间观察它们（通过`watch sysctl dev.myfirst.0`或shell循环）实时显示驱动程序的活动。

### 可写Sysctl

- `dev.myfirst.N.intr_simulate`：写入位掩码以模拟中断。

（第19章驱动程序只暴露这一个用于中断的可写sysctl。挑战练习添加`intr_sim_period_ms`用于基于速率的模拟和`intr_cpu`用于亲和。）

### 内核级视图

`vmstat -i`和`devinfo -v`已经展示内核的视图：

- `vmstat -i`展示`intr_event`的总数和速率。
- `devinfo -v`展示设备的IRQ资源。

这些不是`myfirst`特有的；它们对每个驱动程序都可用。学习阅读它们是通用FreeBSD操作者技能的一部分。

### 关联视图

尝试诊断问题的操作者可能会交叉检查计数器：

```sh
# 内核对分发给我们处理程序的中断计数。
vmstat -i | grep myfirst

# 驱动程序对过滤器被调用次数的计数。
sysctl dev.myfirst.0.intr_count
```

如果这些数字匹配，内核路径和驱动程序过滤器一致。如果内核计数超过驱动程序计数，一些中断正在被处理但未被识别（可能被同一线路上的另一个处理程序）。如果驱动程序计数超过内核计数，有问题（驱动程序正在计算内核未传递的调用；如果最近触发，模拟中断sysctl是最可能的原因）。

加载卸载循环中一到两次差异是正常的（卸载周围的计时）。持续增长的差异表明有bug。

### DTrace

内核的`fbt`提供者让您跟踪任何内核函数的进入和退出，包括`myfirst_intr_filter`：

```sh
sudo dtrace -n 'fbt::myfirst_intr_filter:entry { @[probefunc] = count(); }'
```

打印DTrace看到的过滤器调用次数。与`intr_count`交叉检查。

更有趣的是，DTrace脚本可以聚合每次调用的计时：

```sh
sudo dtrace -n '
fbt::myfirst_intr_filter:entry { self->t = timestamp; }
fbt::myfirst_intr_filter:return /self->t/ {
    @["filter_ns"] = quantize(timestamp - self->t);
    self->t = 0;
}'
```

输出是过滤器执行时间的纳秒直方图。健康过滤器花费在几百纳秒到个位数微秒之间；任何更高的都是bug或极其慢的设备。

### ktrace和kgdb

对于深度调试，`ktrace`可以跟踪系统调用活动；`kgdb`可以检查panic的内核核心转储。第19章不直接使用这些，但驱动程序在中断路径panic的读者会需要它们。



## 动手实验

每个实验建立在前一个之上，对应章节的一个阶段。完成全部五个的读者拥有完整的中断感知驱动程序、模拟中断管道和验证一切的回归脚本。

时间预算假设读者已阅读相关小节。

### 实验1：探索系统的中断源

时间：三十分钟。

目标：建立对系统正在处理什么中断以及什么速率的直觉。

步骤：

1. 运行`vmstat -i > /tmp/intr_before.txt`。
2. 做一些让系统活跃三十秒的事情：运行`dd if=/dev/urandom of=/dev/null bs=1m count=1000`，或打开浏览器页面（在有图形会话的系统上），或从另一台主机scp文件。
3. 运行`vmstat -i > /tmp/intr_after.txt`。
4. 用`diff`计算差异：

```sh
paste /tmp/intr_before.txt /tmp/intr_after.txt
```

5. 对于每个改变的源，注意：
   - 中断名。
   - 前后的计数。
   - 推断的三十秒内速率。
6. 选一个源，用`devinfo -v`或`pciconf -lv`识别其驱动程序。

预期观察：

- 定时器中断（`cpu0:timer`等）高且稳定，每CPU一个。
- 网络中断（`em0`、`igc0`等）在`dd`或`scp`活动期间高，其他时候接近零。
- 存储中断（`ahci0`、`nvme0`等）在磁盘活动期间高，其他时候低。
- 某些中断从不改变；那些是测试期间安静的设备。

本实验是关于读取现实。无代码。回报是每个后续实验的`vmstat -i`输出都是熟悉的领域。

### 实验2：阶段1，注册并触发处理程序

时间：两到三小时。

目标：向第18章的驱动程序添加中断分配、过滤器注册和清理。版本目标：`1.2-intr-stage1`。

步骤：

1. 从第18章阶段4开始，将驱动程序源码复制到新的工作目录。
2. 编辑`myfirst.h`，添加四个softc字段（`irq_res`、`irq_rid`、`intr_cookie`、`intr_count`）。
3. 在`myfirst_pci.c`中，添加最小过滤器处理程序（`atomic_add_64`；返回`FILTER_HANDLED`）。
4. 扩展attach路径，加入IRQ分配、`bus_setup_intr`和`bus_describe_intr`调用。添加相应的goto标签。
5. 扩展detach路径，加入`bus_teardown_intr`和`bus_release_resource`用于IRQ。
6. 添加只读sysctl `dev.myfirst.N.intr_count`。
7. 提升版本字符串到`1.2-intr-stage1`。
8. 编译：`make clean && make`。
9. 在bhyve guest上加载。检查：
   - `dmesg`无`[GIANT-LOCKED]`警告。
   - `devinfo -v | grep -A 5 myfirst0`同时显示`memory:`和`irq:`。
   - `vmstat -i | grep myfirst`显示处理程序。
   - `sysctl dev.myfirst.0.intr_count`返回合理值（如果设备安静则为零）。
10. 卸载。检查`vmstat -m | grep myfirst`显示零活跃分配。

常见失败：

- 缺少`INTR_MPSAFE`：检查`dmesg`中的`[GIANT-LOCKED]`。
- 错误的`rid`值：`bus_alloc_resource_any`返回NULL。确认`sc->irq_rid = 0`。
- 过滤器中的睡眠锁：`WITNESS` panic。
- 缺少拆除：`kldunload` panic或调试内核抱怨活跃的处理程序。

### 实验3：阶段2，真实过滤器和延迟任务

时间：三到四小时。

目标：扩展过滤器以读取INTR_STATUS、确认并入队延迟任务。版本目标：`1.2-intr-stage2`。

步骤：

1. 从实验2开始，向softc添加每位的计数器（`intr_data_av_count`、`intr_error_count`、`intr_complete_count`、`intr_task_invocations`、`intr_last_data`）。
2. 添加taskqueue（`intr_tq`）和任务（`intr_data_task`）字段。
3. 在`myfirst_init_softc`中，初始化任务并创建taskqueue。
4. 在`myfirst_deinit_softc`中，排空任务，释放taskqueue。
5. 重写过滤器，读取`INTR_STATUS`，检查每个位，确认，为`DATA_AV`入队任务，返回正确的`FILTER_*`值。
6. 编写任务函数（`myfirst_intr_data_task_fn`），读取`DATA_OUT`，更新softc，广播条件变量。
7. 在attach路径中，过滤器注册后，在设备端启用`INTR_MASK`。
8. 在detach路径中，`bus_teardown_intr`之前，禁用`INTR_MASK`。
9. 为新计数器添加只读sysctl。
10. 提升版本到`1.2-intr-stage2`。
11. 编译、加载、验证基本接线（同实验2）。

为了观察，加载后等待几秒：如果设备产生匹配我们位布局的真实中断，计数器会递增。在bhyve virtio-rnd目标上，没有正确类型的真实中断到达；通过进入实验4来验证计数器。

### 实验4：阶段3，通过sysctl模拟中断

时间：两到三小时。

目标：添加`intr_simulate` sysctl，用它来驱动管道。版本目标：`1.2-intr-stage3`。

步骤：

1. 从实验3开始，添加`intr_simulate` sysctl处理程序（第5节中的那个）。
2. 在`myfirst_init_softc`或sysctl设置中注册它。
3. 编译、加载。
4. 模拟单个`DATA_AV`事件：

```sh
sudo sysctl dev.myfirst.0.intr_simulate=1
sleep 0.1
sysctl dev.myfirst.0.intr_count
sysctl dev.myfirst.0.intr_data_av_count
sysctl dev.myfirst.0.intr_task_invocations
```

所有三个计数器应显示1。

5. 在循环中模拟十个`DATA_AV`事件：

```sh
for i in 1 2 3 4 5 6 7 8 9 10; do
    sudo sysctl dev.myfirst.0.intr_simulate=1
done
sleep 0.5
sysctl dev.myfirst.0.intr_task_invocations
```

任务计数应接近10（如果taskqueue将多次入队合并为单次运行可能会少；每次运行只记录一次调用但`npending`会更大）。

6. 一起模拟所有三个位：

```sh
sudo sysctl dev.myfirst.0.intr_simulate=7
```

所有三个每位的计数器递增。

7. 检查`intr_error_count`和`intr_complete_count`正确递增：

```sh
sudo sysctl dev.myfirst.0.intr_simulate=2  # ERROR
sudo sysctl dev.myfirst.0.intr_simulate=4  # COMPLETE
sysctl dev.myfirst.0 | grep intr_
```

8. 实现可选的基于速率的callout（`intr_sim_period_ms`），验证速率：

```sh
sudo sysctl hw.myfirst.intr_sim_period_ms=100
sleep 10
sysctl dev.myfirst.0.intr_count  # 大约100
sudo sysctl hw.myfirst.intr_sim_period_ms=0
```

### 实验5：阶段4，重构、回归、版本

时间：三到四小时。

目标：将中断代码移入`myfirst_intr.c`/`.h`，引入`ICSR_*`宏，编写`INTERRUPTS.md`，运行回归。版本目标：`1.2-intr`。

步骤：

1. 从实验4开始，创建`myfirst_intr.c`和`myfirst_intr.h`。
2. 将过滤器、任务、设置、拆除和sysctl注册移入`myfirst_intr.c`。
3. 向`myfirst_intr.h`添加`ICSR_READ_4`和`ICSR_WRITE_4`宏。
4. 更新过滤器以使用`ICSR_READ_4`/`ICSR_WRITE_4`而非原始`bus_read_4`/`bus_write_4`。
5. 在`myfirst_pci.c`中，用对`myfirst_intr_setup`和`myfirst_intr_teardown`的调用替换内联中断代码。
6. 更新`Makefile`，将`myfirst_intr.c`添加到SRCS。提升版本到`1.2-intr`。
7. 编写`INTERRUPTS.md`文档描述中断处理程序的设计。
8. 编译。
9. 运行完整回归脚本（十个attach/detach/unload循环并检查计数器；见配套示例）。
10. 确认：无警告、无泄漏、计数器符合预期。

预期结果：

- 版本`1.2-intr`的驱动程序与阶段3行为相同，但文件结构更清晰。
- `myfirst_pci.c`缩短50-80行。
- `myfirst_intr.c`约200-300行。
- 回归脚本连续通过十次。



## 挑战练习

挑战是可选的。每个建立在一个实验之上，以章节未采取的方向扩展驱动程序。它们巩固章节材料，是为第20章做准备的练习。

### 挑战1：添加过滤器加ithread处理程序

重写阶段2的过滤器，使其返回`FILTER_SCHEDULE_THREAD`而非入队taskqueue任务。通过`bus_setup_intr(9)`的第五个参数注册ithread处理程序，执行任务所做的同样工作。比较两种方法。

此练习是内化基于ithread的延迟工作与基于taskqueue的延迟工作之间区别的方式。完成后，读者应能说出每种何时适合。

### 挑战2：实现驱动程序侧风暴缓解

添加一个计数器，跟踪当前毫秒内处理的中断数。如果计数超过阈值（比如10000），屏蔽设备的中断并调度任务10毫秒后重新启用。

此练习证明驱动程序侧缓解是可能的，并展示为什么内核默认（什么都不做）通常就可以了。

### 挑战3：将中断绑定到特定CPU

添加sysctl `dev.myfirst.N.intr_cpu`，接受CPU ID。写入时，调用`bus_bind_intr(9)`将中断路由到该CPU。用`cpuset -g`或`vmstat -i`中的每CPU计数验证。

此练习引入CPU亲和API并展示选择如何在系统级工具中可见。

### 挑战4：扩展带每类型速率的模拟中断

修改`intr_sim_period_ms` callout，接受位掩码指示模拟哪个事件类，不只是`DATA_AV`。读者应能以不同速率模拟交替的`ERROR`和`COMPLETE`事件。

此练习测试读者对阶段2过滤器每位处理的理解。

### 挑战5：添加速率限制的杂散日志

实现第6节提到的基于`ppsratecheck(9)`的杂散中断日志。验证当驱动程序收到杂散时日志以预期速率出现（可以通过在设备产生事件时禁用`INTR_MASK`来诱发杂散，或通过用零状态手动调用过滤器）。

### 挑战6：实现MSI分配（第20章预览）

向attach路径添加代码，首先尝试`pci_alloc_msi(9)`，如果MSI不可用则回退到传统IRQ。过滤器保持不变。这是第20章的预览；现在做让读者熟悉MSI分配API。

注意在bhyve virtio-rnd目标上，MSI通常不可用（bhyve的传统virtio传输使用INTx）。QEMU的`virtio-rng-pci`暴露MSI-X；您可能希望为此挑战切换实验室到QEMU。

### 挑战7：编写延迟测试

使用模拟中断路径测量驱动程序的中断到用户空间延迟。用户空间程序打开`/dev/myfirst0`，发起在条件变量上睡眠的`read(2)`；第二个程序写入`intr_simulate` sysctl，启动墙钟计时器；第一个程序的`read`返回，停止计时器。多次迭代后绘制分布。

此练习让读者接触驱动程序延迟路径的性能测量。典型延迟在良好调优系统上是几十微秒。

### 挑战8：故意共享IRQ

如果您有配置了同一插槽不同功能上有多个设备的bhyve guest，故意强制它们共享IRQ。加载两个驱动程序（其他设备的基系统驱动程序；我们用于virtio-rnd的`myfirst`）。用`vmstat -i`验证它们共享线路。观察任一个触发时的行为。

此实验是对共享IRQ正确性的最清晰演示。搞错第6节规则的驱动程序会在这里行为异常。



## 故障排除和常见错误

整合了中断特有的故障模式、症状和修复列表。作为您可以返回参考的参考。

### "驱动程序加载但没有中断被计数"

症状：`kldload`成功，`dmesh`显示attach横幅，但`sysctl dev.myfirst.0.intr_count`无限保持为零。

可能原因：

1. 设备未产生中断。在bhyve virtio-rnd目标上这是正常的，因为设备不产生第17章风格的事件。使用模拟中断sysctl来驱动管道。
2. `INTR_MASK`未设置。处理程序已注册，但设备未断言线路因为掩码为零。检查attach路径中的`CSR_WRITE_4(sc, MYFIRST_REG_INTR_MASK, ...)`调用。
3. 设备被其他方式掩码。检查命令寄存器中的`PCIM_CMD_INTxDIS`（中断禁用位）；如果设置了，清除它。
4. 分配了错误的IRQ。`rid = 0`应该产生设备的传统INTx。检查`devinfo -v | grep -A 5 myfirst0`显示`irq:`条目。

### "dmesh中出现GIANT-LOCKED警告"

症状：`kldload`后的`dmesh`显示`myfirst0: [GIANT-LOCKED]`。

原因：`INTR_MPSAFE`未传递给`bus_setup_intr`的flags参数。

修复：向flags添加`INTR_MPSAFE`。确认过滤器仅使用自旋安全操作（原子、自旋互斥锁）。确认softc中的锁规则允许MP安全操作。

### "过滤器中出现内核panic"

症状：内核panic，其回溯显示`myfirst_intr_filter`。

可能原因：

1. `sc`为NULL或陈旧。检查传递给`bus_setup_intr`的参数；应为`sc`，而非`&sc`。
2. 正在获取睡眠锁。`WITNESS`在此上panic。修复是移除睡眠锁或将工作移至taskqueue。
3. 过滤器对已释放的softc调用。这通常意味着detach在释放状态之前未拆除处理程序。检查detach顺序。
4. `sc->bar_res`为NULL。attach部分失败展开与过滤器运行之间的竞争。用检查保护过滤器的首次访问。

### "任务运行但访问已释放状态"

症状：任务函数中的内核panic，回溯显示`myfirst_intr_data_task_fn`。

原因：任务在detach期间或就在detach之前入队，detach在任务运行之前释放了softc。

修复：向detach路径添加`taskqueue_drain`，在释放任务触及的任何东西之前。见第7节。

### "INTR_STATUS位持续触发；检测到风暴"

症状：`dmesh`显示`interrupt storm detected`。

原因：过滤器未正确确认`INTR_STATUS`。可能性：

1. 过滤器根本不写`INTR_STATUS`。添加写入。
2. 过滤器写入错误的值。写入处理的具体位，而非`0`或`0xffffffff`。
3. 过滤器仅处理部分位；未识别的位保持设置并继续断言。要么识别所有位，要么显式确认未识别的位，或在`INTR_MASK`中取消设置它们。

### "模拟中断不运行任务"

症状：`sudo sysctl dev.myfirst.0.intr_simulate=1`递增`intr_count`但不递增`intr_task_invocations`。

可能原因：

1. 模拟的位与过滤器查找的不匹配。阶段2过滤器为`DATA_AV`（位0x1）入队任务。写入`2`或`4`设置ERROR或COMPLETE；那些不入队。写入`1`或`7`。
2. 任务函数未注册。检查`myfirst_init_softc`中的`TASK_INIT`。
3. taskqueue未创建。检查`myfirst_init_softc`中的`taskqueue_create`和`taskqueue_start_threads`。

### "kldunload失败并显示设备忙碌"

症状：`kldunload myfirst`失败并显示`Device busy`。

原因：同第18章。用户空间进程已打开cdev；进行中的命令未完成；驱动程序的忙碌检查有bug。添加`fstat /dev/myfirst0`查看谁打开了它。

### "卸载后vmstat -m显示活跃分配"

症状：`vmstat -m | grep myfirst`在`kldunload`后返回非零`InUse`。

可能原因：

1. taskqueue未排空。检查detach路径中的`taskqueue_drain`。
2. 模拟后端已附着（仅模拟构建）且未分离。检查detach路径中的`myfirst_sim_detach`。
3. `myfirst_init_softc` / `myfirst_deinit_softc`中的泄漏。检查每个分配都有匹配的释放。

### "处理程序在错误的CPU上触发"

症状：`cpuset -g`显示中断在CPU X上触发；读者希望它在CPU Y上。

原因：未调用`bus_bind_intr`，或用错误的CPU参数调用。

修复：添加sysctl让操作者设置所需CPU并调用`bus_bind_intr`。见挑战3。

### "INTR_MASK写入有意外的副作用"

症状：在bhyve virtio-rnd目标上，向偏移0x10（第17章`INTR_MASK`偏移）写入导致设备状态意外改变。

原因：第17章寄存器布局不匹配virtio-rnd。virtio-rnd上的偏移0x10是`queue_notify`，不是`INTR_MASK`。

修复：这是目标不匹配，不是驱动程序bug。章节承认了该问题。对于具有第17章布局的真实设备，写入是正确的。对于bhyve教学目标，写入无害（它通知一个空闲的virtqueue）但无意义。

### "dmesh中出现杂散中断消息"

症状：`dmesh`周期性显示IRQ线路上杂散中断的消息。

可能原因：

1. 处理程序在detach期间未正确在设备处掩码`INTR_MASK`（电平触发杂散）。
2. 设备正在产生驱动程序未启用的中断。检查`INTR_MASK`设置。
3. 共享线路的另一个驱动程序返回错误的`FILTER_*`值。这是那个驱动程序的bug，不是我们的。

### "处理程序在多个CPU上同时被调用"

症状：原子计数器非单调递增，暗示同时的过滤器调用。

原因：在MSI-X（第20章）上，同一处理程序可在不同CPU上同时运行。这是设计使然。对于传统IRQ这很少见但在某些配置中可能。

修复：确保所有过滤器状态访问是原子或自旋锁保护的。第19章驱动程序全程使用`atomic_add_64`；无需更改。

### "bus_setup_intr返回EINVAL"

症状：`bus_setup_intr`的返回值为`EINVAL`，驱动程序无法加载。

可能原因：

1. `filter`和`ihand`参数都是`NULL`。至少一个必须非NULL；否则内核没有可调用的东西。
2. flags参数中省略了`INTR_TYPE_*`标志。必须恰好设置一个类别。
3. IRQ资源未以`RF_ACTIVE`分配。未激活的资源不能附着处理程序。
4. flags参数包含互斥的位（罕见；驱动程序作者需要发明这个）。

修复：阅读`bus_setup_intr(9)`手册页；常见情况是缺少过滤器或ithread参数或缺少类别标志。

### "bus_setup_intr返回EEXIST"

症状：后续加载时`bus_setup_intr`返回`EEXIST`。

原因：IRQ线路已经附着了独占处理程序。要么此驱动程序之前已加载且未正确拆除，要么另一个驱动程序已独占声明该线路。

修复：首先，尝试卸载任何先前的实例（`kldunload myfirst`）。如果问题持续，检查`devinfo -v`当前使用该IRQ的任何驱动程序。

### "调试内核在taskqueue_drain时panic"

症状：调试内核中`taskqueue_drain` panic。

可能原因：

1. taskqueue从未创建。`sc->intr_tq`为NULL。检查`myfirst_init_softc`。
2. taskqueue已被释放。检查拆除路径中的双重释放。
3. 从未调用`TASK_INIT`。任务函数指针是垃圾。

修复：确保`TASK_INIT`在`taskqueue_enqueue`运行前运行；确保`taskqueue_free`最多运行一次。

### "过滤器被调用但INTR_STATUS读取为0xffffffff"

症状：过滤器运行，读取`INTR_STATUS`，看到`0xffffffff`。

可能原因：

1. 设备无响应（可能bhyve guest已死亡或设备被热拔出）。
2. BAR映射错误。检查attach路径。
3. PCI错误已使设备进入瘫痪状态。

修复：如果设备活着，读取返回真实状态位。如果`0xffffffff`，说明其他东西有问题。过滤器仍应返回`FILTER_STRAY`（因为0xffffffff不太可能是合法状态值；与设备数据手册交叉检查有效位组合）。

### "中断被计数但设备无进展"

症状：`intr_count`递增，但设备的操作（数据传输、任务完成等）不推进。

可能原因：

1. 过滤器确认每个位但任务不运行。检查`intr_task_invocations`；如果为零，`taskqueue_enqueue`路径断了。
2. 任务运行但未唤醒等待者。检查任务中的`cv_broadcast`。
3. 设备正在发信号表示异常情况。检查`INTR_STATUS`内容（通过sysctl路径读取，或在DDB中）。

修复：在任务中添加日志（通过`device_printf`）；检查任务的逻辑是否匹配设备的实际行为。

### "kldunload挂起"

症状：`kldunload myfirst`不返回。无panic，无输出。

可能原因：

1. `bus_teardown_intr`阻塞等待进行中的处理程序（过滤器或ithread）完成。处理程序卡住。
2. `taskqueue_drain`阻塞等待卡住的任务。
3. detach函数等待从未广播的条件变量。

修复：如果系统其他方面响应，切换到DDB（按下NMI键或输入`sysctl debug.kdb.enter=1`）并`ps`查找卡住的线程。回溯通常精确定位卡住的函数。

### "过滤器中出现未对齐内存访问"

症状：在对齐敏感的架构（arm64、MIPS、SPARC）上内核panic，回溯指向过滤器。

原因：过滤器正在未对齐的偏移处读写寄存器。PCI BAR读写需要自然对齐（32位读写4字节对齐，16位读写2字节对齐）。

修复：在4字节对齐的偏移处使用`bus_read_4` / `bus_write_4`。第17章的寄存器映射全部是4字节对齐的。

### "来自过滤器的device_printf减慢系统"

症状：向过滤器添加`device_printf`调用使系统在高中断率下明显滞后。

原因：`device_printf`获取锁并进行格式化打印。每秒一万次中断时，开销可测量。

修复：在生产测试前移除过滤器中的调试打印。改用计数器和DTrace来观察。

### "驱动程序通过所有测试但在负载下行为异常"

症状：单线程测试通过，但多用多进程并发负载测试触发偶尔错误或状态损坏。

可能原因：

1. 过滤器与任务之间的竞争条件。过滤器设置任务读取的标志；任务更新过滤器读取的状态。没有适当同步，一个可能错过另一个的更新。
2. 任务与另一个线程上下文路径（cdev处理程序、sysctl）之间的竞争。任务持有`sc->mtx`；另一路径也应如此。
3. 无锁的复合操作中使用的原子变量。单独的`atomic_add_64`是原子的；`atomic_load_64`后跟计算后跟`atomic_store_64`作为序列不是原子的。

修复：审查锁定规则。`WITNESS`不捕获纯原子变量竞争；仔细的代码审查可以。在启用`INVARIANTS`的严重负载下运行并观察断言失败。

### "vmstat -i显示我不拥有的线路上有许多杂散"

症状：共享线路上的驱动程序看到线路的杂散计数器稳定增长。

可能原因：

1. 线路上的另一个驱动程序错误地返回`FILTER_STRAY`（中断是它的但它声称不是）。
2. 线路上的一个设备正在发信号表示驱动程序不确认的事件，产生幻像杂散。
3. 硬件噪声或触发模式配置错误。

修复：修复通常在于返回`FILTER_STRAY`错误的任何驱动程序。您自己的驱动程序的行为只要其状态寄存器检查正确就是正确的。



## 高级可观察性：与DTrace集成

FreeBSD的DTrace可以在多个级别观察中断路径。本小节展示一些有用的DTrace单行脚本，驱动程序作者在开发期间可以使用。

### 计算过滤器调用次数

```sh
sudo dtrace -n '
fbt::myfirst_intr_filter:entry { @invocations = count(); }'
```

显示自DTrace启动以来过滤器被调用的总次数。与`sysctl dev.myfirst.0.intr_count`比较；它们应该一致。

### 测量过滤器延迟

```sh
sudo dtrace -n '
fbt::myfirst_intr_filter:entry { self->ts = vtimestamp; }
fbt::myfirst_intr_filter:return /self->ts/ {
    @["filter_ns"] = quantize(vtimestamp - self->ts);
    self->ts = 0;
}'
```

`vtimestamp`测量CPU时间（非墙钟），所以直方图是过滤器的真实CPU时间。健康过滤器在几百纳秒到个位数微秒范围内。

### 观察任务队列

```sh
sudo dtrace -n '
fbt::myfirst_intr_data_task_fn:entry {
    @["task_runs"] = count();
    self->ts = vtimestamp;
}
fbt::myfirst_intr_data_task_fn:return /self->ts/ {
    @["task_ns"] = quantize(vtimestamp - self->ts);
    self->ts = 0;
}'
```

显示任务的调用计数和每次调用的执行时间。任务通常比过滤器慢一个数量级（因为它持有睡眠锁并做更多工作）。

### 关联过滤器和任务

```sh
sudo dtrace -n '
fbt::myfirst_intr_filter:entry /!self->in_filter/ {
    self->in_filter = 1;
    self->filter_start = vtimestamp;
    @["filter_enters"] = count();
}
fbt::myfirst_intr_filter:return /self->in_filter/ {
    self->in_filter = 0;
}
fbt::myfirst_intr_data_task_fn:entry {
    @["task_starts"] = count();
}'
```

如果`filter_enters`是100而`task_starts`是80，一些过滤器调用未调度任务（因为事件是ERROR或COMPLETE，不是DATA_AV）。

### 跟踪Taskqueue调度决策

taskqueue基础设施也有DTrace探针；可以观察任务如何入队以及工作线程何时运行：

```sh
sudo dtrace -n '
fbt::taskqueue_enqueue:entry /arg0 == $${tq_addr}/ {
    @["enqueues"] = count();
}'
```

其中`$${tq_addr}`是`sc->intr_tq`的数字地址，可通过`kldstat` / `kgdb`组合获得。这种级别的细节对于第19章驱动程序通常是杀鸡用牛刀。

### DTrace与模拟中断路径

模拟中断与真实中断可区分，因为模拟路径经过sysctl处理程序：

```sh
sudo dtrace -n '
fbt::myfirst_intr_simulate_sysctl:entry { @["simulate"] = count(); }
fbt::myfirst_intr_filter:entry { @["filter"] = count(); }'
```

两个计数之间的差异是真实中断的数量（不紧跟sysctl调用的过滤器调用）。



## 详细遍历：阶段2端到端

为了让第19章驱动程序具体化，这里是通过sysctl模拟`DATA_AV`事件时发生什么的完整逐步遍历。

### 序列

1. 用户运行`sudo sysctl dev.myfirst.0.intr_simulate=1`。
2. 内核的sysctl机制将写入路由到`myfirst_intr_simulate_sysctl`。
3. 处理程序解析值（1），通过`MYFIRST_LOCK`获取`sc->mtx`。
4. 处理程序将`1`写入BAR中的`INTR_STATUS`。
5. 处理程序释放`sc->mtx`。
6. 处理程序直接调用`myfirst_intr_filter(sc)`。
7. 过滤器通过`ICSR_READ_4`读取`INTR_STATUS`。值为`1`（DATA_AV）。
8. 过滤器原子递增`intr_count`。
9. 过滤器看到设置了DATA_AV位，递增`intr_data_av_count`。
10. 过滤器通过`ICSR_WRITE_4`将`1`写回`INTR_STATUS`来确认。
11. 过滤器通过`taskqueue_enqueue`在`intr_tq`上入队`intr_data_task`。
12. 过滤器返回`FILTER_HANDLED`。
13. sysctl处理程序向内核的sysctl层返回0。
14. 用户的`sysctl`命令返回成功。

同时，在taskqueue中：

15. taskqueue的工作线程（被`taskqueue_enqueue`唤醒）调度。
16. 工作线程调用`myfirst_intr_data_task_fn(sc, 1)`。
17. 任务获取`sc->mtx`。
18. 任务通过`CSR_READ_4`读取`DATA_OUT`。
19. 任务将值存储在`sc->intr_last_data`中。
20. 任务递增`intr_task_invocations`。
21. 任务广播`sc->data_cv`（此例中无等待者）。
22. 任务释放`sc->mtx`。
23. 工作线程回到等待更多工作。

步骤1-14花费微秒；步骤15-23根据调度花费几十到几百微秒。

### 计数器显示什么

一次模拟中断后：

```text
dev.myfirst.0.intr_count: 1
dev.myfirst.0.intr_data_av_count: 1
dev.myfirst.0.intr_error_count: 0
dev.myfirst.0.intr_complete_count: 0
dev.myfirst.0.intr_task_invocations: 1
```

如果`intr_task_invocations`仍为0，任务尚未运行（通常因为sysctl在工作线程调度之前返回）。短暂`sleep 0.01`足够。

### dmesh显示什么

默认情况下，无。阶段4驱动程序不啰嗦。想看过滤器触发的读者可以添加`device_printf`调用调试，但生产质量驱动程序通常不在每次中断时打印。

### vmstat -i显示什么

`vmstat -i | grep myfirst`显示`intr_event`的总计数。这仅计算内核向我们过滤器传递的真实中断。通过sysctl传递的模拟中断不经过内核的中断分发器，所以不出现在`vmstat -i`计数中。

这是有用的区别：sysctl传递的模拟是补充机制，非替代。真实中断仍计数；模拟的不计数。

### 用打印语句跟踪

为了快速调试，向过滤器和任务添加`device_printf`调用给出实时画面：

```c
/* 在过滤器中，暂时用于调试： */
device_printf(sc->dev, "filter: status=0x%x\n", status);

/* 在任务中： */
device_printf(sc->dev, "task: data=0x%x npending=%d\n",
    data, npending);
```

这产生类似如下的`dmesh`输出：

```text
myfirst0: filter: status=0x1
myfirst0: task: data=0xdeadbeef npending=1
```

生产前移除这些打印；高中断率时打印代价昂贵。



## 真实FreeBSD驱动程序的模式

紧凑巡视`/usr/src/sys/dev/`中反复出现的中断模式。每个模式是来自真实驱动程序（稍微重写以提高可读性）的具体代码片段，并附带重要性说明。第19章后阅读这些可巩固词汇。

### 模式：快速过滤器加慢任务

来自`/usr/src/sys/dev/mgb/if_mgb.c`：

```c
int
mgb_legacy_intr(void *xsc)
{
	struct mgb_softc *sc = xsc;
	uint32_t intr_sts = CSR_READ_REG(sc, MGB_INTR_STS);
	uint32_t intr_en = CSR_READ_REG(sc, MGB_INTR_ENBL_SET);

	intr_sts &= intr_en;
	if ((intr_sts & MGB_INTR_STS_ANY) == 0)
		return (FILTER_STRAY);

	/* Acknowledge and defer per-queue work. */
	if ((intr_sts & MGB_INTR_STS_RX_ANY) != 0) {
		for (int qidx = 0; qidx < scctx->isc_nrxqsets; qidx++) {
			if (intr_sts & MGB_INTR_STS_RX(qidx))
				iflib_rx_intr_deferred(sc->ctx, qidx);
		}
		return (FILTER_HANDLED);
	}
	return (FILTER_SCHEDULE_THREAD);
}
```

为什么重要：过滤器短，延迟工作是每队列的，共享IRQ规则被维持。第19章的过滤器遵循相同形状。

### 模式：仅ithread处理程序

来自`/usr/src/sys/dev/ath/if_ath_pci.c`：

```c
bus_setup_intr(dev, psc->sc_irq,
    INTR_TYPE_NET | INTR_MPSAFE,
    NULL, ath_intr, sc, &psc->sc_ih);
```

过滤器参数为`NULL`；`ath_intr`是ithread处理程序。内核在每次中断时调度`ath_intr`，没有中间过滤器。

为什么重要：有时所有工作需要线程上下文。为过滤器参数注册NULL比写一个只返回`FILTER_SCHEDULE_THREAD`的平凡过滤器更简单。

### 模式：INTR_EXCL用于独占访问

某些驱动程序需要独占访问中断线路：

```c
bus_setup_intr(dev, irq,
    INTR_TYPE_BIO | INTR_MPSAFE | INTR_EXCL,
    NULL, driver_intr, sc, &cookie);
```

为什么重要：在罕见情况下，驱动程序需要线路仅为自己（处理程序对作为唯一监听者的假设已固化）。`INTR_EXCL`要求内核拒绝同一事件上的其他驱动程序。

### 模式：短调试日志

某些驱动程序有可选的详细模式，记录每次过滤器调用：

```c
if (sc->sc_debug > 0)
	device_printf(sc->sc_dev, "interrupt: status=0x%x\n", status);
```

为什么重要：开发中的驱动程序受益于日志；生产驱动程序希望日志抑制。sysctl（`dev.driver.N.debug`）切换模式。

### 模式：绑定到特定CPU

知道其拓扑的驱动程序将中断绑定到局部CPU：

```c
/* After bus_setup_intr: */
error = bus_bind_intr(dev, irq, local_cpu);
if (error != 0)
	device_printf(dev, "bus_bind_intr: %d\n", error);
/* Non-fatal: some platforms do not support binding. */
```

为什么重要：NUMA局部的处理程序更快。费心绑定的驱动程序在多插槽系统上产生更好的可扩展性故事。

### 模式：为诊断描述处理程序

每个驱动程序应调用`bus_describe_intr`：

```c
bus_describe_intr(dev, irq, cookie, "rx-%d", queue_id);
```

为什么重要：`vmstat -i`和`devinfo -v`使用描述来区分共享事件上的处理程序。有N个队列和N个MSI-X向量的驱动程序有N个`bus_describe_intr`调用。

### 模式：在Detach前静默

```c
mtx_lock(&sc->mtx);
sc->shutting_down = true;
mtx_unlock(&sc->mtx);

/* Let the interrupt handler drain. */
bus_teardown_intr(dev, sc->irq_res, sc->intr_cookie);
```

为什么重要：`shutting_down`标志给处理程序一个快速退出路径（处理程序在正常工作前检查该标志）。`bus_teardown_intr`是最终排空，但该标志让排空更快。

第19章驱动程序使用`sc->pci_attached`达到类似目的。



## 参考：常见错误速查表

紧凑的中断特定错误及其单行修复列表。审查自己驱动程序时用作检查清单很有用。

1. **无INTR_MPSAFE。** 修复：`flags = INTR_TYPE_MISC | INTR_MPSAFE`。
2. **过滤器中的睡眠锁。** 修复：使用原子操作或自旋互斥锁。
3. **缺少确认。** 修复：`bus_write_4(res, INTR_STATUS, bits_handled);`。
4. **确认太多位。** 修复：只写回您处理的位。
5. **缺少`FILTER_STRAY`返回。** 修复：如果状态为零或未识别，返回`FILTER_STRAY`。
6. **缺少`FILTER_HANDLED`返回。** 修复：`rv |= FILTER_HANDLED;` 对于每个识别的位。
7. **任务使用陈旧softc。** 修复：向detach添加`taskqueue_drain`。
8. **缺少`bus_teardown_intr`。** 修复：在`bus_release_resource(SYS_RES_IRQ, ...)`之前。
9. **在detach时缺少`INTR_MASK = 0`。** 修复：在拆除之前清除掩码。
10. **缺少`taskqueue_drain`。** 修复：在释放softc状态之前排空。
11. **错误的过滤器返回值。** 修复：必须是`FILTER_HANDLED`、`FILTER_STRAY`、`FILTER_SCHEDULE_THREAD`或其按位或。
12. **在`TASK_INIT`之前入队任务。** 修复：在attach中初始化任务。
13. **在attach时未设置`INTR_MASK`。** 修复：写入您想要启用的位。
14. **传统IRQ的错误rid。** 修复：使用`rid = 0`。
15. **错误的资源类型。** 修复：中断使用`SYS_RES_IRQ`。
16. **在共享线路上缺少`RF_SHAREABLE`。** 修复：在分配中包含该标志。
17. **在`bus_setup_intr`期间持有sc->mtx。** 修复：在调用之前释放锁。
18. **在`bus_teardown_intr`期间持有自旋锁。** 修复：拆除时绝不持有过滤器的自旋锁。
19. **任务仍在入队时taskqueue被销毁。** 修复：`taskqueue_free`之前`taskqueue_drain`。
20. **缺少`bus_describe_intr`调用。** 修复：在`bus_setup_intr`之后添加它以获得诊断清晰度。



## 参考：ithread和Taskqueue优先级

第19章代码为taskqueue使用`PI_NET`。FreeBSD在`/usr/src/sys/sys/priority.h`中定义了几个优先级常量。简化视图：

```text
PI_REALTIME  = PRI_MIN_ITHD + 0   (最高ithread优先级)
PI_INTR      = PRI_MIN_ITHD + 4   (通用"硬件中断"级别)
PI_AV        = PI_INTR            (音频/视频)
PI_NET       = PI_INTR            (网络)
PI_DISK      = PI_INTR            (块存储)
PI_TTY       = PI_INTR            (终端/串口)
PI_DULL      = PI_INTR            (低优先级硬件ithread)
PI_SOFT      = PRI_MIN_ITHD + 8   (软中断)
PI_SOFTCLOCK = PI_SOFT            (软时钟)
PI_SWI(c)    = PI_SOFT            (每类别SWI)
```

读者查看此列表会注意到，大多数"硬件中断"别名（`PI_AV`、`PI_NET`、`PI_DISK`、`PI_TTY`、`PI_DULL`）解析为相同数值`PI_INTR`。`priority.h`中该块顶部的注释明确说明了原因："大多数硬件中断线程以相同优先级运行，但如果它们运行完整时间片可以衰减到更低优先级"。类别名称存在是因为每个名称在调用点读起来自然，而非数值优先级不同。

只有`PI_REALTIME`（略高于`PI_INTR`）和`PI_SOFT`（低于`PI_INTR`）实际上与通用硬件中断级别不同。

ithread的优先级来自`INTR_TYPE_*`标志；taskqueue的优先级显式设置。向`taskqueue_start_threads`传递`PI_NET`使工作线程处于与网络ithread相同的名义级别，这是与网络速率中断处理协作工作的正确选择。存储驱动程序应传递`PI_DISK`；低优先级后台驱动程序传递`PI_DULL`。因为常量都映射为相同数值，名称实际上可以互换用于正确性。它们对可读性和任何未来数值区别变真实的内核仍很重要。



## 参考：/usr/src/sys/kern/kern_intr.c 短途浏览

对`bus_setup_intr(9)`和`bus_teardown_intr(9)`幕后发生的事情好奇的读者可以打开`/usr/src/sys/kern/kern_intr.c`。该文件约1800行，有明显的节：

- **intr_event管理**（`intr_event_create`、`intr_event_destroy`）：`intr_event`结构的顶层创建和清理。
- **处理程序管理**（`intr_event_add_handler`、`intr_event_remove_handler`）：`bus_setup_intr`和`bus_teardown_intr`调用的底层操作。
- **分发**（`intr_event_handle`、`intr_event_schedule_thread`）：中断触发时实际运行的代码。
- **风暴检测**（`intr_event_handle`）：`intr_storm_threshold`逻辑。
- **ithread创建和调度**（`ithread_create`、`ithread_loop`、`ithread_update`）：每事件ithread机制。
- **SWI（软件中断）管理**（`swi_add`、`swi_sched`、`swi_remove`）：软件中断。

读者不需要理解整个文件就能写驱动。浏览顶级函数列表和阅读`intr_event_handle`（分发函数）的注释是值得的半小时。

### kern_intr.c中的关键函数

| 函数 | 目的 |
|------|------|
| `intr_event_create` | 分配新的`intr_event`。 |
| `intr_event_destroy` | 释放`intr_event`。 |
| `intr_event_add_handler` | 附着过滤器/ithread处理程序。 |
| `intr_event_remove_handler` | 分离处理程序。 |
| `intr_event_handle` | 分发：每次中断时调用。 |
| `intr_event_schedule_thread` | 唤醒ithread。 |
| `ithread_loop` | ithread的主体。 |
| `swi_add` | 注册软件中断。 |
| `swi_sched` | 调度软件中断。 |

暴露给驱动程序的BUS_*函数（`bus_setup_intr`、`bus_teardown_intr`、`bus_bind_intr`、`bus_describe_intr`）在平台特定的总线驱动钩子之后调用这些内核内部函数。




## 总结

第19章给了驱动程序耳朵。在开始时，`myfirst`版本`1.1-pci`已附着到真实PCI设备但不监听它：驱动程序采取的每个动作都由用户空间发起，设备自身的异步事件（如有）未被注意。在结束时，`myfirst`版本`1.2-intr`有一个过滤器处理程序连接到设备的IRQ线路、一个在线程上下文中处理大量工作的延迟任务管道、一个用于实验室目标测试的模拟中断路径、一个与同一线路上其他驱动程序共存的共享IRQ纪律、一个按正确顺序释放每个资源的干净拆除，以及新文件`myfirst_intr.c`和`INTERRUPTS.md`文档。

过渡经过八个节。第1节从硬件层面介绍中断，涵盖边沿触发和电平触发信号、CPU的分发流程，以及驱动程序处理程序的六个义务。第2节介绍FreeBSD的内核模型：`intr_event`、`intr_handler`、ithread、过滤器加ithread分离、`INTR_MPSAFE`，以及过滤器上下文的约束。第3节编写最小过滤器和attach/detach接线。第4节用状态解码、按位确认和基于taskqueue的延迟工作扩展过滤器。第5节添加模拟中断sysctl，让读者能在无真实IRQ事件时演练管道。第6节编纂共享IRQ纪律：检查所有权、正确返回`FILTER_STRAY`、防御性处理未识别位。第7节整合拆除：在设备处屏蔽、拆除处理程序、排空taskqueue、释放资源。第8节将整个内容重构为可维护的布局。

第19章未做的是MSI、MSI-X或DMA。驱动程序的中断路径是单条传统IRQ；数据路径不使用DMA；延迟工作是单个taskqueue任务。第20章介绍MSI和MSI-X（多向量、每向量过滤器、更丰富的中断路由）。第20和21章介绍DMA以及中断与DMA描述符环的交互。

第19章完成的是两个控制线程之间的分离。驱动程序的过滤器短小，在主中断上下文中运行，处理紧急的每中断工作。驱动程序的延迟任务较长，在线程上下文中运行，处理大量的每事件工作。保持它们协作的纪律（过滤器状态用原子操作、任务状态用睡眠锁、拆除的严格顺序）是每章后续中断代码假设的纪律。

文件布局已增长：`myfirst.c`、`myfirst_hw.c`、`myfirst_hw_pci.c`、`myfirst_hw.h`、`myfirst_sim.c`、`myfirst_sim.h`、`myfirst_pci.c`、`myfirst_pci.h`、`myfirst_intr.c`、`myfirst_intr.h`、`myfirst_sync.h`、`cbuf.c`、`cbuf.h`、`myfirst.h`。文档已增长：`HARDWARE.md`、`LOCKING.md`、`SIMULATION.md`、`PCI.md`、`INTERRUPTS.md`。测试套件已增长：模拟中断管道、阶段4回归脚本、少量挑战练习供读者实践。

### 第20章前的反思

下一章前暂停一下。第19章教授了过滤器加任务模式、`INTR_MPSAFE`承诺、中断上下文约束和共享IRQ纪律。您在此练习的模式（读状态、确认、延迟工作、返回正确的`FILTER_*`、干净拆除）是每个FreeBSD中断处理程序使用的模式。第20章将MSI-X叠加其上；第21章将DMA叠加其上。两章都不替换第19章模式；都建立在它们之上。

值得做出的第二个观察。第17章模拟、第18章真实PCI附着和第19章中断处理的组合现在架构意义上是完整的驱动程序。理解这三层的读者可以打开任何FreeBSD PCI驱动程序并识别各部分：寄存器映射、PCI附着、中断过滤器。具体细节不同；结构恒定。这种识别是本书投资在整个FreeBSD源码树获得回报的原因。

第三个观察：第16章访问器层的回报继续。`CSR_*`宏在第19章未改变；`ICSR_*`宏为过滤器上下文使用而添加，但它们调用相同的底层`bus_read_4`和`bus_write_4`。抽象现已回报三次：针对第17章模拟后端、第18章真实PCI BAR和第19章过滤器上下文。在自己驱动程序中构建类似访问器层的读者将发现相同的红利。

### 卡住时怎么办

三个建议。

首先，专注于模拟中断路径。如果`sudo sysctl dev.myfirst.0.intr_simulate=1`使计数器跳动并使任务运行，管道就在工作。章节的其他每部分在装饰管道的意义上是可选的，但如果管道失败，整章就不工作，第5节是诊断的正确位置。

其次，打开`/usr/src/sys/dev/mgb/if_mgb.c`并缓慢重读`mgb_legacy_intr`函数。它约六十行过滤器代码。每行映射到第19章概念。完成章节后阅读一次应该感觉像熟悉领域。

第三，首次通过时跳过挑战。实验室按第19章节奏校准；挑战假设章节内容扎实。如果现在感觉难以企及，第20章后再回来。

第19章的目标是给驱动程序一种监听其设备的方式。如果已做到，第20章的MSI-X机制成为专业化而非全新主题，第21章的DMA成为将描述符完成连接到您已有中断路径的问题。



## 通向第20章的桥梁

第20章标题为*高级中断处理*。其范围是第19章故意未采用的专业化：MSI（消息信号中断）和MSI-X，取代传统INTx线路的现代PCIe中断机制，以作为内存写入传递的每设备（或每队列）向量。

第19章以四种具体方式准备了基础。

首先，**您有一个运作的过滤器处理程序**。第19章过滤器读状态、处理位、确认并延迟。第20章过滤器看起来类似，但按向量复制：每个MSI-X向量有自己的过滤器，每个处理设备事件的特定子集。

其次，**您理解attach/detach级联**。第19章通过两个标签（`fail_release_irq`、`fail_teardown_intr`）增长级联。第20章进一步增长：每向量一对标签。模式不变；计数变。

第三，**您有中断拆除纪律**。第20章重用第19章顺序：清除设备中断、每个向量`bus_teardown_intr`、每个IRQ资源`bus_release_resource`。每向量性质添加小循环；顺序相同。

第四，**您有暴露MSI-X的实验室环境**。在带`virtio-rng-pci`的QEMU上，MSI-X可用；在带`virtio-rnd`的bhyve上，仅暴露传统INTx。第20章实验室可能需要切换到QEMU或更丰富模拟的bhyve设备来演练MSI-X路径。

第20章将涵盖的具体主题：

- 为什么MSI和MSI-X是对传统INTx的改进。
- MSI与MSI-X的区别（单向量 vs 向量表）。
- `pci_alloc_msi(9)`、`pci_alloc_msix(9)`：分配向量。
- `pci_msi_count(9)`、`pci_msix_count(9)`：查询能力。
- `pci_release_msi(9)`：拆除对应物。
- 多向量中断处理程序：每队列过滤器。
- MSI-X表布局及如何到达特定条目。
- 跨向量的CPU亲和性用于NUMA感知。
- 中断合并：设备支持时减少中断率。
- MSI-X与iflib（现代网络驱动框架）的交互。
- 将`myfirst`驱动程序从第19章传统路径迁移到MSI-X路径，为不支持MSI-X的设备回退到传统。

您不需要向前读。第19章是足够准备。带上您的`myfirst`驱动程序`1.2-intr`、您的`LOCKING.md`、您的`INTERRUPTS.md`、您的启用`WITNESS`内核和您的回归脚本。第20章从第19章结束处开始。

第21章再向前一章；值得简要前向指针。DMA将引入与中断的另一交互：发信号"描述符环条目N完成"的完成中断。第19章教授的过滤器加任务纪律延续；任务的工作现在涉及遍历描述符环而非读取单个寄存器。

词汇是您的；结构是您的；纪律是您的。第20章为这三者添加精度。



## 参考：第19章快速参考卡

第19章引入的词汇、API、宏和过程的紧凑总结。

### 词汇

- **Interrupt（中断）**：异步硬件信号事件。
- **IRQ（中断请求）**：中断线路的标识符。
- **Edge-triggered（边沿触发）**：由跃迁信号；每次跃迁一次中断。
- **Level-triggered（电平触发）**：由保持电平信号；电平保持时中断触发。
- **intr_event**：FreeBSD的一个中断源的内核结构。
- **ithread**：运行延迟中断处理程序的FreeBSD内核线程。
- **filter handler（过滤器处理程序）**：在主中断上下文中运行的函数。
- **ithread handler（ithread处理程序）**：在过滤器之后的线程上下文中运行的函数。
- **FILTER_HANDLED**：过滤器处理了中断；不需要ithread。
- **FILTER_SCHEDULE_THREAD**：过滤器部分处理；运行ithread。
- **FILTER_STRAY**：中断不是给此驱动程序的。
- **INTR_MPSAFE**：承诺处理程序自己做同步的标志。
- **INTR_TYPE_***（TTY、BIO、NET、CAM、MISC、CLK、AV）：处理程序类别提示。
- **INTR_EXCL**：独占中断。

### 基本API

- `bus_alloc_resource_any(dev, SYS_RES_IRQ, &rid, flags)`：声明IRQ。
- `bus_release_resource(dev, SYS_RES_IRQ, rid, res)`：释放IRQ。
- `bus_setup_intr(dev, res, flags, filter, ihand, arg, &cookie)`：注册处理程序。
- `bus_teardown_intr(dev, res, cookie)`：注销处理程序。
- `bus_describe_intr(dev, res, cookie, "name")`：为工具命名处理程序。
- `bus_bind_intr(dev, res, cpu)`：将中断路由到CPU。
- `pci_msi_count(dev)`、`pci_msix_count(dev)`（第20章）。
- `pci_alloc_msi(dev, &count)`、`pci_alloc_msix(dev, &count)`（第20章）。
- `pci_release_msi(dev)`（第20章）。
- `taskqueue_create("name", M_WAITOK, taskqueue_thread_enqueue, &tq)`：创建taskqueue。
- `taskqueue_start_threads(&tq, n, PI_pri, "thread name")`：启动工作线程。
- `taskqueue_enqueue(tq, &task)`：入队任务。
- `taskqueue_drain(tq, &task)`：等待任务完成，阻止新入队。
- `taskqueue_free(tq)`：释放taskqueue。
- `TASK_INIT(&task, pri, fn, arg)`：初始化任务。

### 基本宏

- `FILTER_HANDLED`、`FILTER_STRAY`、`FILTER_SCHEDULE_THREAD`。
- `INTR_TYPE_TTY`、`INTR_TYPE_BIO`、`INTR_TYPE_NET`、`INTR_TYPE_CAM`、`INTR_TYPE_MISC`、`INTR_TYPE_CLK`、`INTR_TYPE_AV`。
- `INTR_MPSAFE`、`INTR_EXCL`。
- `RF_SHAREABLE`、`RF_ACTIVE`。
- `SYS_RES_IRQ`。

### 常见过程

**分配传统PCI中断并注册过滤器处理程序：**

1. `sc->irq_rid = 0;`
2. `sc->irq_res = bus_alloc_resource_any(dev, SYS_RES_IRQ, &sc->irq_rid, RF_SHAREABLE | RF_ACTIVE);`
3. `bus_setup_intr(dev, sc->irq_res, INTR_TYPE_MISC | INTR_MPSAFE, filter, NULL, sc, &sc->intr_cookie);`
4. `bus_describe_intr(dev, sc->irq_res, sc->intr_cookie, "name");`

**拆除中断处理程序：**

1. 在设备处禁用中断（清除`INTR_MASK`）。
2. `bus_teardown_intr(dev, sc->irq_res, sc->intr_cookie);`
3. `taskqueue_drain(sc->intr_tq, &sc->intr_data_task);`
4. `taskqueue_free(sc->intr_tq);`
5. `bus_release_resource(dev, SYS_RES_IRQ, sc->irq_rid, sc->irq_res);`

**编写过滤器处理程序：**

1. 读`INTR_STATUS`；如为零，返回`FILTER_STRAY`。
2. 对每个识别位，递增计数器，通过写回确认，并可选入队任务。
3. 返回`FILTER_HANDLED`（或`FILTER_SCHEDULE_THREAD`），或如无识别返回`FILTER_STRAY`。

### 有用命令

- `vmstat -i`：列出中断源及计数。
- `devinfo -v`：列出设备及其IRQ资源。
- `sysctl hw.intrcnt`和`sysctl hw.intrnames`：原始计数器。
- `sysctl hw.intr_storm_threshold`：启用内核风暴检测。
- `cpuset -g`：查询中断CPU亲和性（平台特定）。
- `sudo sysctl dev.myfirst.0.intr_simulate=1`：触发模拟中断。

### 要保持书签的文件

- `/usr/src/sys/sys/bus.h`：`driver_filter_t`、`driver_intr_t`、`FILTER_*`、`INTR_*`。
- `/usr/src/sys/kern/kern_intr.c`：内核的中断事件机制。
- `/usr/src/sys/sys/taskqueue.h`：taskqueue API。
- `/usr/src/sys/dev/mgb/if_mgb.c`：可读的过滤器加任务示例。
- `/usr/src/sys/dev/ath/if_ath_pci.c`：最小ithread-only中断设置。



## 参考：第四部分对比表

第四部分每章适合位置、添加内容和假设内容的紧凑总结。对在部分中跳跃或回顾的读者有用。

| 主题 | 第16章 | 第17章 | 第18章 | 第19章 | 第20章（预览）| 第21章（预览）|
|------|--------|--------|--------|--------|----------------|----------------|
| BAR访问 | 用malloc模拟 | 用模拟层扩展 | 真实PCI BAR | 相同 | 相同 | 相同 |
| 第17章模拟 | 不适用 | 引入 | PCI上不活跃 | PCI上不活跃 | PCI上不活跃 | PCI上不活跃 |
| PCI附着 | 不适用 | 不适用 | 引入 | 相同+IRQ | MSI-X选项 | DMA初始化添加 |
| 中断处理 | 不适用 | 不适用 | 不适用 | 引入 | MSI-X每向量 | 完成驱动 |
| DMA | 不适用 | 不适用 | 不适用 | 不适用 | 预览 | 引入 |
| 版本 | 0.9-mmio | 1.0-simulated | 1.1-pci | 1.2-intr | 1.3-msi | 1.4-dma |
| 新文件 | `myfirst_hw.c` | `myfirst_sim.c` | `myfirst_pci.c` | `myfirst_intr.c` | `myfirst_msix.c` | `myfirst_dma.c` |
| 关键纪律 | 访问器抽象 | 假设备 | Newbus附着 | 过滤器/任务分离 | 每向量处理程序 | DMA映射 |

表格使本书累积结构一目了然。理解给定主题行的读者可以预测第19章工作如何适应大局。



## 参考：第19章FreeBSD手册页

第19章材料最有用的手册页列表。在FreeBSD系统上用`man 9 <name>`（内核API）或`man 4 <name>`（子系统概述）打开每个。

### 内核API手册页

- **`bus_setup_intr(9)`**：注册中断处理程序。
- **`bus_teardown_intr(9)`**：拆除处理程序。
- **`bus_bind_intr(9)`**：绑定到CPU。
- **`bus_describe_intr(9)`**：标记处理程序。
- **`bus_alloc_resource(9)`**：资源分配（通用）。
- **`bus_release_resource(9)`**：资源释放。
- **`atomic(9)`**：原子操作包括`atomic_add_64`。
- **`taskqueue(9)`**：taskqueue原语。
- **`ppsratecheck(9)`**：速率限制日志助手。
- **`swi_add(9)`**：软件中断（作为替代提及）。
- **`intr_event(9)`**：中断事件机制（如存在；某些API是内部的）。

### 设备子系统手册页

- **`pci(4)`**：PCI子系统。
- **`vmstat(8)`**：`vmstat -i`用于观察中断。
- **`devinfo(8)`**：设备树和资源。
- **`devctl(8)`**：运行时设备控制。
- **`sysctl(8)`**：读写sysctl。
- **`dtrace(1)`**：动态追踪。

大多数已在章节正文中引用。此汇总列表供想要单一位置查找它们的读者。



## 参考：驱动程序记忆短语

总结第19章纪律的几句格言。阅读和代码审查有用。

- **"读、确认、延迟、返回。"** 过滤器做的四件事。
- **"如果未识别任何内容则FILTER_STRAY。"** 共享IRQ协议。
- **"拆除前屏蔽；释放前拆除。"** detach顺序。
- **"过滤器上下文仅限自旋锁。"** 无睡眠锁规则。
- **"每次入队释放前需排空。"** taskqueue生命周期。
- **"一个过滤器、一个设备、一个状态。"** 保持每设备代码清晰的隔离。
- **"如果WITNESS崩溃，相信它。"** 调试内核捕获细微错误。
- **"先PROD，后中断。"** 在启用处理程序前编程设备（`INTR_MASK`）。
- **"过滤器中做小；任务中做大。"** 工作大小纪律。
- **"风暴检测是安全网，不是设计工具。"** 不要依赖内核的节流。

这些都不是完整规范。每个是紧凑提醒，可展开为章节的详细处理。



## 参考：第19章术语表

**ack（acknowledge，确认）**：写回INTR_STATUS以清除待定位并取消IRQ线路断言的操作。

**driver_filter_t**：过滤器处理程序函数的C类型定义：`int f(void *)`。

**driver_intr_t**：ithread处理程序函数的C类型定义：`void f(void *)`。

**edge-triggered（边沿触发）**：中断由电平跃迁信号的中断信号模式。

**FILTER_HANDLED**：过滤器返回值，意为"此中断已处理；不需要ithread"。

**FILTER_SCHEDULE_THREAD**：返回值，意为"调度ithread运行"。

**FILTER_STRAY**：返回值，意为"此中断不是给此驱动程序的"。

**filter handler（过滤器处理程序）**：在主中断上下文中运行的C函数。

**Giant**：遗留单一全局内核锁；现代驱动程序通过设置INTR_MPSAFE避免它。

**IE（interrupt event）**：`intr_event`的简称。

**INTR_MPSAFE**：承诺处理程序自己做同步且无Giant安全的标志。

**INTR_STATUS**：跟踪待决中断原因的设备寄存器（RW1C）。

**INTR_MASK**：启用特定中断类的设备寄存器。

**intr_event**：表示一个中断源的内核结构。

**ithread**：内核中断线程；在线程上下文中运行延迟处理程序。

**level-triggered（电平触发）**：电平保持时中断触发的中断信号模式。

**MSI**：消息信号中断；PCIe机制（第20章）。

**MSI-X**：带向量表的MSI更丰富变体（第20章）。

**primary interrupt context（主中断上下文）**：过滤器处理程序的上下文；无睡眠、无睡眠锁。

**PCIR_INTLINE / PCIR_INTPIN**：指定传统IRQ线路和引脚的PCI配置空间字段。

**RF_ACTIVE**：资源分配标志；一步激活资源。

**RF_SHAREABLE**：资源分配标志；允许与其他驱动程序共享资源。

**stray interrupt（迷途中断）**：无过滤器返回声明的中断；由内核单独计数。

**storm（风暴）**：电平触发中断因驱动程序未确认而持续触发的情况。

**SYS_RES_IRQ**：中断的资源类型。

**taskqueue**：在线程上下文中运行延迟工作的内核原语。

**trap stub**：CPU取中断向量时运行的小片内核代码。

**EOI（End of Interrupt，中断结束）**：发送给中断控制器以重新装备IRQ线路的信号。



## 参考：中断处理哲学结语

一段结束章节的话，实验室后值得回顾。

中断处理程序的工作不是做设备的工作。设备的工作（处理数据包、完成I/O、读取传感器）由驱动程序的其余部分完成，在线程上下文中，在驱动程序全套锁下。处理程序的工作更窄：注意设备有话要说，确认设备使对话继续，调度稍后发生的真正工作，并足够快返回使CPU空闲用于被中断线程或下一个中断。

编写了第19章驱动程序的读者已编写了一个中断处理程序。它很小。驱动程序的其余部分使其有用。第20章将处理程序专业化为MSI-X上的每向量工作。第21章将任务专业化为遍历DMA描述符环。每个都是扩展，非替换。第19章处理程序是两者构建的骨架。

第19章教授的技能不是"如何为virtio-rnd设备处理中断"。它是"如何在主上下文和线程上下文之间分离工作、如何尊重过滤器约束、如何干净拆除、以及如何与共享线路上的其他驱动程序合作"。每个是可转移技能。FreeBSD树中的每个驱动程序都使用其中一些；大多数驱动程序使用全部。

对于此读者和本书未来读者，第19章过滤器和任务是`myfirst`驱动程序架构的永久部分。每章后续都假设它们。每章后续都扩展它们。驱动程序的整体复杂性将增长，但中断路径将保持第19章所做：狭窄、快速、正确排序的代码，让开路以便驱动程序其余部分完成工作。

