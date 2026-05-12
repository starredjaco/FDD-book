---
title: "导航 FreeBSD 内核内部"
description: "围绕驱动程序工作的 FreeBSD 内核子系统的导航导向地图，包含帮助读者快速定位的结构、源代码树位置和驱动程序接触点。"
appendix: "E"
lastUpdated: "2026-04-20"
status: "complete"
author: "Edson Brandi"
reviewer: "TBD"
translator: "AI辅助翻译为简体中文"
language: "zh-CN"
estimatedReadTime: 40
---

# 附录 E：导航 FreeBSD 内核内部

## 如何使用本附录

主要章节教你从第一个 `printf("hello")` 模块到具有 DMA 和中断的工作 PCI 驱动程序来构建 FreeBSD 设备驱动程序。在该进展之下是一个拥有许多活动部件的大型内核，本书无法从头教授每个部分而不失去你实际尝试做的事情的主线。大多数时候你不需要知道内核的每个角落。你只需要知道你在哪里、当前代码行正在接触哪个子系统、当你停下来查看时哪个结构给你答案，以及 `/usr/src` 中证据的位置。

本附录就是那张地图。它不试图从第一原理教授每个子系统。它选取驱动程序作者最常遇到的七个子系统，并为每个提供简短版本：它的用途、哪些结构重要、你的驱动程序可能跨越哪些 API、在哪里打开文件查看，以及接下来读什么。你可以把它视为放在章节旁边的野外指南，而不是它们的替代品。

### 你将在这里找到什么

每个子系统都以相同的小模式覆盖，这样你可以浏览一个并知道在下一个中看向哪里。

- **子系统用于什么。**一段话说明子系统的职责。
- **为什么驱动程序作者应该关心。**你的代码遇到该子系统的具体原因。
- **关键结构、接口或概念。**真正重要的名称简短列表。
- **典型驱动程序接触点。**驱动程序调用、注册或从子系统接收回调的具体位置。
- **在 `/usr/src` 中查看的位置。**首先值得打开的两三个文件。
- **接下来阅读的手册页和文件。**当你想要更多深度时的下一步。
- **常见初学者困惑。**消耗人们时间的误解。
- **本书在哪里教授这个。**回引用在上下文中使用该子系统的章节。

并非每个条目都需要每个标签，也没有条目试图详尽无遗。目的是模式识别，而非完整的子系统手册。

### 本附录不是什么

它不是 API 参考。附录 A 是 API 参考，深入到每个调用的标志、生命周期阶段和注意事项。当问题是*这个函数做什么*或*哪个标志正确*时，附录 A 是查看的地方。

它也不是概念教程。附录 D 涵盖操作系统心智模型（内核与用户空间、驱动程序类型、引导到 init 路径），附录 C 涵盖硬件模型（物理内存、MMIO、中断、DMA），附录 B 涵盖算法模式（`<sys/queue.h>`、环形缓冲区、状态机、展开阶梯）。如果你想回答的问题是"什么是进程"、"什么是 BAR"或"应该使用哪个列表宏"，其中一个附录是正确的目的地。

它也不是完整的子系统参考。VFS、VM 或网络栈的完整游览本身就需要一本书。你在这里得到的是驱动程序作者实际遇到的每个子系统的百分之十，按照驱动程序作者遇到的顺序。

## 读者指南

使用本附录的三种方式，每种需要不同的阅读策略。

如果你正在**阅读主要章节**，请在第二个窗口中保持附录打开。当第 5 章介绍内核内存分配器时，浏览这里的内存子系统部分以查看这些分配器相对于 UMA 和 VM 系统的位置。当第 6 章遍历 `device_t`、softc 和 probe/attach 生命周期时，驱动程序基础设施部分向你展示这些类型如何嵌入 Newbus 层。当第 24 章讨论 `SYSINIT`、`eventhandler(9)` 和任务队列时，引导和模块系统以及内核服务部分各用一页给你周围上下文。

如果你正在**阅读不熟悉的内核代码**，请将附录视为翻译器。当你在函数签名中看到 `struct mbuf` 时，跳到网络子系统部分。当你看到 `struct bio` 时，跳到文件和 VFS。当你看到 `kobj_class_t` 或 `device_method_t` 时，跳到驱动程序基础设施。探索期间的目标不是掌握子系统，只是命名它。

如果你正在**设计新驱动程序**，请在开始前扫描你的驱动程序将接触的子系统。小型外设的字符驱动程序将依赖驱动程序基础设施和文件和 VFS。网络驱动程序将添加网络子系统。存储驱动程序将添加 GEOM 和缓冲区缓存层。嵌入式板上的早期引导驱动程序将添加引导和模块系统部分。知道你将接触哪些子系统有助于你在写一行代码前猜测正确的头文件和正确的章节。

一些约定适用于全篇：

- 源代码路径以面向书籍的形式显示，`/usr/src/sys/...`，与标准 FreeBSD 系统上的布局匹配。你可以在你的实验机器上打开它们中的任何一个。
- 手册页以通常的 FreeBSD 风格引用。面向内核的页面生活在第 9 节：`kthread(9)`、`malloc(9)`、`uma(9)`、`bus_space(9)`、`eventhandler(9)`。用户空间接口生活在第 2 或 3 节，在相关时提及。
- 当条目指向阅读示例时，文件是初学者可以在一次阅读中阅读的文件。存在也使用每个模式的更大文件；仅当它们是权威参考时才提及。

有了这些，我们从整个内核的一页纸定位开始，然后逐一深入子系统。

## 本附录与附录 A 的区别

驱动程序作者在工作期间最终会咨询两种非常不同的参考。一种回答*我需要的函数或标志的确切名称是什么*的问题。那是附录 A。另一种回答*我在哪个子系统中，这部分放在哪里*的问题。那是本附录。

具体地说，区别像这样表现。当你想知道 `malloc(9)` 的签名、`M_WAITOK` 与 `M_NOWAIT` 的含义，以及打开哪个手册页时，那是附录 A。当你想知道 `malloc(9)` 是 UMA 之上的薄便利层，而 UMA 又建立在 `vm_page_t` 层之上，后者又依赖于每架构 `pmap(9)` 时，那是本附录。

两个附录都引用真实源代码路径和真实手册页。分离是故意的。将 API 查找与子系统地图分开使每个都足够短以实际使用。如果这里的条目开始看起来像附录 A，它就偏离了角色，正确的做法是改为阅读附录 A。

## 主要子系统的地图

在进入任何子系统之前，整个内核的形状值得命名。FreeBSD 内核很大，但驱动程序作者遇到的部分适合一小组家族。下面的图是最简单的诚实图片。

```text
+-----------------------------------------------------------------+
|                            用户空间                           |
|     应用程序、守护进程、shell、工具、库         |
+-----------------------------------------------------------------+
                               |
                      系统调用陷阱（边界）
                               |
+-----------------------------------------------------------------+
|                           内核空间                          |
|                                                                 |
|   +-----------------------+   +-----------------------------+   |
|   |   VFS / devfs / GEOM  |   |        网络栈        |
|   |  struct vnode, buf,   |   |   struct mbuf, socket,      |   |
|   |  bio, vop_vector      |   |   ifnet, route, VNET        |   |
|   +-----------------------+   +-----------------------------+   |
|                 \                     /                         |
|                  \                   /                          |
|                 驱动程序基础设施 (Newbus)                  |
|           device_t, driver_t, devclass_t, softc, kobj           |
|           bus_alloc_resource, bus_space, bus_dma                |
|                               |                                 |
|      进程/线程子系统  |  内存 / VM 子系统          |
|      struct proc, thread       |  vm_map, vm_object, vm_page    |
|      ULE 调度器, kthreads   |  pmap, UMA, pagers             |
|                               |                                 |
|         引导和模块系统 (SYSINIT, KLD, modules)          |
|         内核服务 (eventhandler, taskqueue, callout)      |
+-----------------------------------------------------------------+
                               |
                      硬件 I/O 边界
                               |
+-----------------------------------------------------------------+
|                             硬件                            |
|     MMIO 寄存器、中断控制器、支持 DMA 的内存   |
+-----------------------------------------------------------------+
```

上面标记的每个框都有下面的一节。中间的驱动程序基础设施框是每个驱动程序开始的地方。顶部的两个框是驱动程序向内核其余部分发布的子系统入口点（左侧为字符或存储，右侧为网络）。中间行的两个框是每个驱动程序依赖的水平服务。底部的框是使内核首先运行起来的管道。

大多数驱动程序只详细接触这些框中的三四个。附录的组织使你可以只阅读你的驱动程序实际使用的那些。

## 进程和线程子系统

### 子系统用于什么

进程和线程子系统管理 FreeBSD 内部的每个执行单元。它拥有描述运行程序的数据结构、决定哪个线程接下来在哪个 CPU 上运行的调度器、创建和销毁内核线程的机制，以及管理线程如何阻塞、睡眠或被抢占的规则。每行内核代码，包括你的驱动程序，都由某个线程执行，子系统强制的规则是对你的驱动程序允许做什么的直接约束。

### 为什么驱动程序作者应该关心

三个实际原因。首先，你的代码运行的上下文（中断过滤器、ithread、任务队列工作者、来自用户空间的系统调用线程、你生成的专用内核线程）决定你是否可以睡眠、可以用 `M_WAITOK` 分配内存或可以持睡眠锁。其次，任何需要后台工作的驱动程序（轮询循环、恢复看门狗、延迟命令处理器）将创建内核线程或 kproc 来承载它。第三，任何查看调用者进程凭据的驱动程序（例如，用于 `d_ioctl` 中的安全检查）将访问进程结构。

### 关键结构、接口或概念

- **`struct proc`** 是每进程描述符。它记录进程 ID、凭据、文件描述符表、信号状态、地址空间和属于该进程的线程列表。
- **`struct thread`** 是每线程描述符。它记录线程 ID、优先级、运行状态、保存的寄存器上下文、指向其拥有的 `struct proc` 的指针，以及它当前持有的锁。FreeBSD 内核线程也由 `struct thread` 描述；它只是没有用户空间端。
- **ULE 调度器**是 FreeBSD 默认的多处理器调度器。它将线程分配给 CPU，实现优先级类（实时、分时、空闲），并尊重交互性和亲和性提示。从驱动程序作者的角度，ULE 最重要的因素是每当释放锁、睡眠结束或中断完成时，它运行下一个线程；你不能跨此类事件控制 CPU。
- **`kthread_add(9)`** 在现有内核进程内创建新内核线程。当你想要一个与现有 kproc 共享状态的轻量级工作者时使用它（例如，驱动程序特定 kproc 内的额外工作者线程）。
- **`kproc_create(9)`** 创建新内核进程，它带有自己的 `struct proc` 和一个初始线程。当你想要 `ps -axH` 将显示为不同名称的独立顶层工作者时使用它（例如，`g_event`、`usb`、`bufdaemon`）。

### 典型驱动程序接触点

- 中断处理器和 `bus_setup_intr(9)` 回调在中断框架创建的内核线程上下文中运行。
- 需要长期运行后台工作的驱动程序从其 `attach` 路径调用 `kproc_create(9)` 或 `kthread_add(9)`，并从 `detach` 加入线程。
- 代表用户进程采取行动的驱动程序读取 `curthread` 或 `td->td_proc` 来查看凭据、进程 ID 或根目录进行验证。
- 在条件上睡眠的驱动程序使用睡眠原语，它记录睡眠线程并让出给调度器直到被唤醒。

### 在 `/usr/src` 中查看的位置

- `/usr/src/sys/sys/proc.h` 定义 `struct proc` 和 `struct thread` 以及在它们之间导航的宏。
- `/usr/src/sys/sys/kthread.h` 声明内核线程创建 API。
- `/usr/src/sys/kern/kern_kthread.c` 包含实现。
- `/usr/src/sys/kern/sched_ule.c` 是 ULE 调度器源代码。

### 接下来阅读的手册页和文件

`kthread(9)`、`kproc(9)`、`curthread(9)`、`proc(9)`，以及头文件 `/usr/src/sys/sys/proc.h`。如果你想看拥有内核线程的驱动程序，`/usr/src/sys/dev/random/random_harvestq.c` 是一个可读的示例。

### 常见初学者困惑

最常见的陷阱是假设运行你的驱动程序代码的线程是驱动程序拥有的线程。它不是。大多数时候它是通过系统调用进入内核的用户线程，或是中断框架为你创建的 ithread。你的驱动程序只拥有它显式创建的线程。另一个反复出现的陷阱是从与设备无关的上下文（例如中断 ithread）访问 `curthread->td_proc`；你找到的进程不是请求操作的进程。

### 本书在哪里教授这个

第 5 章介绍内核执行上下文和睡眠与原子的区别。第 11 章在并发变得真实时回到它。第 14 章使用任务队列作为将工作卸载到安全可睡眠上下文的方式。第 24 章展示驱动程序内的完整 kproc 生命周期。

### 进一步阅读

- **本书中**：第 11 章（驱动程序中的并发），第 14 章（任务队列和延迟工作），第 24 章（与内核集成）。
- **手册页**：`kthread(9)`、`kproc(9)`、`scheduler(9)`。
- **外部**：McKusick, Neville-Neil, 和 Watson, *The Design and Implementation of the FreeBSD Operating System*（第 2 版），进程和线程管理章节。

## 内存子系统

### 子系统用于什么

虚拟内存（VM）子系统管理内核可寻址的每个字节内存。它拥有从虚拟地址到物理页面的映射、向进程和内核分配页面、在压力下回收页面的分页策略，以及从磁盘、设备或零交出页面的页面后备存储。分配内存、通过 `mmap` 向用户空间公开内存或进行 DMA 的驱动程序都在与 VM 子系统交互，无论它是否命名它。

### 为什么驱动程序作者应该关心

四个实际原因。首先，每个内核分配直接或间接通过该子系统。其次，任何向用户空间导出设备或软件缓冲区内存映射视图的驱动程序通过 VM 分页器这样做。第三，DMA 涉及物理地址，只有 VM 子系统知道虚拟内核地址如何转换为它们。第四，子系统定义分配的睡眠规则：`M_WAITOK` 可能走 VM 的页面回收路径，这是你不能从中断过滤器做的事情。

### 关键结构、接口或概念

- **`vm_map_t`** 表示属于一个地址空间的连续虚拟地址映射集合。内核有自己的 `vm_map_t`，每个用户进程有一个。驱动程序几乎从不直接遍历 `vm_map_t`；更高级的 API 为它们做遍历。
- **`vm_object_t`** 表示后备存储：可以映射到 `vm_map_t` 的一组页面。对象按产生其页面的分页器类型化（匿名、vnode 支持、交换支持、设备支持）。
- **`vm_page_t`** 表示一个物理 RAM 页面，连同其当前状态（固定、活动、非活动、空闲）和它当前属于的对象。系统中的所有物理内存由 `vm_page_t` 记录数组跟踪。
- **分页器层** 是产生页面数据的可插拔策略集合。对驱动程序作者最重要的三个是交换分页器（匿名内存）、vnode 分页器（文件支持的内存）和设备分页器（内容由驱动程序产生的内存）。当驱动程序实现 `d_mmap` 或 `d_mmap_single` 时，它正在发布设备分页器的一个切片。
- **`pmap(9)`** 是机器相关的页表管理器。它知道如何为当前 CPU 架构将虚拟地址转换为物理地址。驱动程序很少直接调用 `pmap`。查看物理视图的可移植方式是通过 `bus_dma(9)`（用于 DMA）或 `bus_space(9)`（用于 MMIO 寄存器）。
- **UMA** 是 FreeBSD 的固定大小对象 slab 分配器，具有每 CPU 缓存以避免快速路径中的锁定。`malloc(9)` 本身为常见大小在 UMA 上实现。每秒分配和释放数百万相同小对象（网络描述符、每请求上下文）的驱动程序用 `uma_zcreate` 创建自己的 UMA 区并重用对象，而不是走通用分配器。

### 典型驱动程序接触点

- 用于常规控制平面内存的 `malloc(9)`、`free(9)`、`contigmalloc(9)`。
- 用于高速固定大小对象的 `uma_zcreate(9)`、`uma_zalloc(9)`、`uma_zfree(9)`。
- 用于 DMA 可用内存的 `bus_dmamem_alloc(9)` 和 `bus_dma(9)` 接口的其余部分；这是 VM 物理侧的驱动程序面向包装器。
- `cdevsw` 方法表中的 `d_mmap(9)` 或 `d_mmap_single(9)` 用于向用户空间发布硬件内存的设备分页器视图。
- `vm_page_wire(9)` 和 `vm_page_unwire(9)` 仅在驱动程序需要在长时间运行的 I/O 中固定用户缓冲区页面的罕见情况下使用。

### 在 `/usr/src` 中查看的位置

- `/usr/src/sys/vm/vm.h` 声明 `vm_map_t`、`vm_object_t` 和 `vm_page_t` typedef。
- `/usr/src/sys/vm/vm_map.h`、`/usr/src/sys/vm/vm_object.h` 和 `/usr/src/sys/vm/vm_page.h` 保存完整类型定义。
- `/usr/src/sys/vm/swap_pager.c`、`/usr/src/sys/vm/vnode_pager.c` 和 `/usr/src/sys/vm/device_pager.c` 是与驱动程序最相关的三个分页器。
- `/usr/src/sys/vm/uma.h` 是 UMA 公共接口；`/usr/src/sys/vm/uma_core.c` 是实现。
- `/usr/src/sys/vm/pmap.h` 是机器无关 pmap 接口；机器特定侧生活在 `/usr/src/sys/amd64/amd64/pmap.c`、`/usr/src/sys/arm64/arm64/pmap.c` 和每个架构的类似文件下。

### 接下来阅读的手册页和文件

`malloc(9)`、`uma(9)`、`contigmalloc(9)`、`bus_dma(9)`、`pmap(9)`，以及头文件 `/usr/src/sys/vm/uma.h`。对于发布设备分页器的可读驱动程序，检查 `/usr/src/sys/dev/drm2/` 或 `/usr/src/sys/dev/fb/` 下的帧缓冲区代码。

### 常见初学者困惑

两个陷阱。首先，混淆支持 DMA 的驱动程序看到的三种指针风格：内核虚拟地址（你的指针解引用的内容）、物理地址（内存控制器看到的）和总线地址（设备看到的，可能通过 IOMMU）。`bus_dma(9)` 正是为了保持这些分离而存在。其次，假设 `bus_dmamem_alloc(9)` 分配是通用内存分配；它是具有你传入的标签规定的更严格对齐、边界和段规则的专用分配。

### 本书在哪里教授这个

第 5 章介绍内核内存和分配器标志。第 10 章在读/写路径中重访缓冲区。第 17 章介绍用于 MMIO 访问的 `bus_space`。第 21 章是完整的 DMA 章节，是总线与物理区别变得具体的地方。

### 进一步阅读

- **本书中**：第 5 章（理解用于 FreeBSD 内核编程的 C），第 21 章（DMA 和高速数据传输）。
- **手册页**：`malloc(9)`、`uma(9)`、`contigmalloc(9)`、`bus_dma(9)`、`pmap(9)`。
- **外部**：McKusick, Neville-Neil, 和 Watson, *The Design and Implementation of the FreeBSD Operating System*（第 2 版），内存管理章节。

## 文件和 VFS 子系统

### 子系统用于什么

文件和 VFS（虚拟文件系统）子系统拥有用户空间通过 `open(2)`、`read(2)`、`write(2)`、`ioctl(2)`、`mmap(2)` 和总体文件系统层次结构看到的一切。它通过 vnode 操作向量将操作分派给正确的文件系统，管理位于文件系统和存储驱动程序之间的缓冲区缓存，并托管让存储驱动程序组合成栈的 GEOM 框架。对于驱动程序作者，该子系统要么是主要入口点（如果你编写字符或存储驱动程序），要么是你乐意让别人担心的安静中间层（如果你编写网络或嵌入式驱动程序）。

### 为什么驱动程序作者应该关心

三个实际原因。首先，每个字符驱动程序通过 `cdevsw` 和 devfs 创建的 `/dev` 节点向 VFS 发布自己。其次，每个存储驱动程序插入 VFS 和 GEOM 层在其上组装的栈底部，你接收的工作单元是 `struct bio`，而非用户指针。第三，即使是非存储驱动程序，如果向用户空间发布其内存，也可能需要理解 vnode 和 `mmap`。

### 关键结构、接口或概念

- **`struct vnode`** 是内核的抽象文件或设备。它携带类型（常规文件、目录、字符设备、块设备、命名管道、套接字、符号链接）、指向其文件系统 `vop_vector` 的指针、它属于的挂载点、引用计数和锁。用户空间中的每个文件描述符最终解析为 vnode。
- **`struct vop_vector`** 是 vnode 操作分派表：每个操作一个指针（`VOP_LOOKUP`、`VOP_READ`、`VOP_WRITE`、`VOP_IOCTL` 等几十个），由文件系统或 devfs 实现。向量在概念上在 `/usr/src/sys/sys/vnode.h` 中声明，从 `/usr/src/sys/kern/vnode_if.src` 中的操作列表生成。
- **GEOM 框架** 是 FreeBSD 的可堆叠存储层。GEOM *提供者*是存储表面；*消费者*是附加到提供者的东西。存储硬件驱动程序注册为提供者；`g_part`、`g_mirror` 或文件系统等类作为消费者附加。拓扑图是动态的，在运行时通过 `gpart show`、`geom disk list` 和 `sysctl kern.geom` 可见。
- **devfs** 是填充 `/dev` 的伪文件系统。当你的字符驱动程序调用 `make_dev_s(9)` 时，devfs 分配一个 vnode 支持的条目，将 VFS 操作转发到你的 `cdevsw` 回调。devfs 是 `/dev` 路径上 `open(2)` 和你驱动程序中 `d_open` 之间的单一层。
- **`struct buf`** 是旧块设备路径和在其上层叠的文件系统使用的传统缓冲区缓存描述符。它仍然重要，因为许多文件系统在 `buf_strategy()` 将它们汇入 GEOM 之前通过 `buf` 对象驱动 I/O。
- **`struct bio`** 是流经 GEOM 的现代每操作描述符。GEOM 中的每个块读或写是一个带有命令（`BIO_READ`、`BIO_WRITE`、`BIO_FLUSH`、`BIO_DELETE`）、范围、缓冲区指针和完成回调的 `bio`。你的存储驱动程序在其启动例程上接收 `bio`，并在完成它们时调用 `biodone()`（或 GEOM 等效物）。

### 典型驱动程序接触点

- 字符驱动程序填充带有回调（`d_open`、`d_close`、`d_read`、`d_write`、`d_ioctl`，可选 `d_poll`、`d_mmap`）的 `struct cdevsw` 并调用 `make_dev_s(9)` 将其附加到 `/dev`。
- 存储驱动程序注册 GEOM 类，实现接受 `bio` 的启动例程，并在完成时调用 `g_io_deliver()`。
- 想要作为文件可见的驱动程序（例如用于读取遥测）可以公开字符设备，其 `d_read` 复制驱动程序数据。
- 向用户空间发布设备内存的驱动程序实现 `d_mmap` 或 `d_mmap_single` 以交回设备分页器对象。

### 在 `/usr/src` 中查看的位置

- `/usr/src/sys/sys/vnode.h` 声明 `struct vnode` 和 vnode 操作管道。
- `/usr/src/sys/kern/vnode_if.src` 是内核中每个 VOP 的真理来源；阅读它以查看操作列表和锁定协议。
- `/usr/src/sys/fs/devfs/` 保存 devfs 实现；`devfs_devs.c` 和 `devfs_vnops.c` 是可读的入口点。
- `/usr/src/sys/geom/geom.h` 声明提供者、消费者和 GEOM 类接口。
- `/usr/src/sys/sys/buf.h` 和 `/usr/src/sys/sys/bio.h` 声明块 I/O 结构。
- `/usr/src/sys/dev/null/null.c` 是树中最简单的字符驱动程序，是正确的首次阅读。

### 接下来阅读的手册页和文件

`vnode(9)`、`VOP_LOOKUP(9)` 和 VOP 家族的其余部分、`devfs(4)`、`devfs(5)`、`cdev(9)`、`make_dev(9)`、`g_attach(9)`、`geom(4)`，以及头文件 `/usr/src/sys/sys/bio.h`。

### 常见初学者困惑

三个陷阱。首先，期望字符驱动程序处理 `struct buf` 或 `struct bio`。它不处理；那些生活在存储路径中。字符驱动程序在其 `d_read` 和 `d_write` 回调上看到 `struct uio`，仅此而已。其次，期望存储驱动程序自己创建 `/dev` 节点。在现代 FreeBSD 中，GEOM 层为块设备创建 `/dev` 条目；你的存储驱动程序向 GEOM 注册，devfs 在 GEOM 的另一侧做其余的事情。第三，假设 vnode 和 cdev 是同一对象。它们不是。vnode 是打开文件的 VFS 侧句柄；cdev 是驱动程序侧身份。`/dev/foo` 上的 `open(2)` 产生一个 vnode，其操作转发到你的 `cdevsw`。

### 本书在哪里教授这个

第 7 章编写第一个字符驱动程序和第一个 `cdevsw`。第 8 章遍历 `make_dev_s(9)` 和 devfs 节点创建。第 9 章将 `d_read` 和 `d_write` 连接到 `uio`。第 27 章是存储章节，介绍 `struct bio`、GEOM 提供者和消费者，以及缓冲区缓存。

### 进一步阅读

- **本书中**：第 7 章（编写你的第一个驱动程序），第 8 章（使用设备文件），第 27 章（使用存储设备和 VFS 层）。
- **手册页**：`vnode(9)`、`make_dev(9)`、`devfs(5)`、`geom(4)`、`g_bio(9)`。
- **外部**：McKusick, Neville-Neil, 和 Watson, *The Design and Implementation of the FreeBSD Operating System*（第 2 版），I/O 系统和本地文件系统章节。

## 网络子系统

### 子系统用于什么

网络子系统移动数据包。它拥有表示飞行中数据包的数据结构（`mbuf` 和朋友）、向栈其余部分表示网络设备的每接口状态（`ifnet`）、决定数据包应该去哪里的路由表、用户空间看到的套接字层，以及让多个独立网络栈在单个内核中共存的 VNET 基础设施。网络驱动程序是该栈的底层：它在接收时向上传递数据包到栈，在发送时栈向下传递数据包到驱动程序。

### 为什么驱动程序作者应该关心

两个原因。如果你编写网络驱动程序，你接触的几乎每个字节都是下面命名的结构之一的一个字段，你的代码形状由它们强制执行的协议设定。如果你编写任何其他类型的驱动程序，你仍然受益于在代码中看到 `struct mbuf` 和 `struct ifnet` 时识别它们，因为它们出现在许多相邻的子系统中（数据包过滤器、负载平衡助手、虚拟接口）。

### 关键结构、接口或概念

- **`struct mbuf`** 是数据包片段。数据包表示为 mbuf 链，由 `m_next` 链接用于单个数据包，由 `m_nextpkt` 链接用于队列中的连续数据包。mbuf 携带小头部和要么小的内联数据区，要么指向外部存储集群的指针。该设计优化用于廉价地前置头部。
- **`struct m_tag`** 是附加到 mbuf 的可扩展元数据标签。它让栈和驱动程序向数据包附加类型化信息（例如，硬件发送校验和卸载、接收端缩放哈希、过滤器决策）而不扩大 mbuf 本身。
- **`ifnet`**（在现代 API 中拼写为 `if_t`）是每接口描述符。它携带接口名称和索引、标志、MTU、发送函数（`if_transmit`）、栈增加的计数器，以及让更高层向驱动程序传递数据包的钩子。
- **VNET** 是每虚拟网络栈容器。当编译 `VIMAGE` 时，启用 VNET 的每个 jail 有自己的路由表、自己的接口集和自己的协议控制块。网络驱动程序必须感知 VNET：它们使用 `VNET_DEFINE` 和 `VNET_FOREACH`，以便每 VNET 状态生活在正确的位置。
- **路由** 是为出站数据包选择下一跳的子系统。它拥有转发信息库（FIB），一个每 VNET 的路由基数树。驱动程序很少直接与路由交互；栈在到达驱动程序之前已经选择了接口。
- **套接字层** 是 `socket(2)` 系统调用族的内核侧。对于驱动程序作者，相关事实是套接字最终产生对 `ifnet` 的调用，后者产生对你的驱动程序的调用。你不自己实现套接字。

### 典型驱动程序接触点

- 驱动程序在 `attach` 中分配并填充 `ifnet`，注册发送函数，并调用 `ether_ifattach(9)` 或 `if_attach(9)` 向栈宣布自己。
- 驱动程序的发送函数接收 mbuf 链，写描述符，触发硬件，并返回。
- 接收路径处理中断或轮询，将接收的字节包装在 mbuf 中，并在接口上调用 `if_input(9)` 将它们推上栈。
- 在分离时，驱动程序在释放其资源前调用 `ether_ifdetach(9)` 或 `if_detach(9)`。
- 如果驱动程序需要在对等体出现或消失时做出反应，它注册 `ifnet_arrival_event` 或 `ifnet_departure_event`（参见 `/usr/src/sys/net/if_var.h` 的声明）。

### 在 `/usr/src` 中查看的位置

- `/usr/src/sys/sys/mbuf.h` 声明 `struct mbuf` 和 `struct m_tag`。
- `/usr/src/sys/net/if.h` 声明 `if_t` 和公共接口 API。
- `/usr/src/sys/net/if_var.h` 声明接口事件事件处理器和内部状态。
- `/usr/src/sys/net/if_private.h` 包含栈内部使用的完整 `struct ifnet` 定义。
- `/usr/src/sys/net/vnet.h` 声明 VNET 基础设施。
- `/usr/src/sys/net/route.h` 和 `/usr/src/sys/net/route/` 保存路由表。
- `/usr/src/sys/sys/socketvar.h` 声明 `struct socket`。

### 接下来阅读的手册页和文件

`mbuf(9)`、`ifnet(9)`、`ether_ifattach(9)`、`vnet(9)`、`route(4)`、`socket(9)`。对于小型、可读的真实网络驱动程序，`/usr/src/sys/net/if_tuntap.c` 是权威阅读示例。

### 常见初学者困惑

两个陷阱。首先，期望网络驱动程序通过 `/dev` 发布自己。它不；它通过 `ifnet` 发布自己并作为 `bge0`、`em0`、`igb0` 等可见，而非通过 devfs。其次，在将 mbuf 交给栈后仍持有它。一旦你调用 `if_input` 或从 `if_transmit` 返回，mbuf 不再属于你；此后使用它会静默破坏栈。

### 本书在哪里教授这个

第 28 章是完整的网络驱动程序章节，是细节的正确位置。第 11 章和第 14 章提供接收路径所需的锁定和延迟工作规则。第 24 章在驱动程序集成层面覆盖 `ifnet_arrival_event` 和相关事件钩子。

### 进一步阅读

- **本书中**：第 28 章（编写网络驱动程序），第 11 章（驱动程序中的并发），第 14 章（任务队列和延迟工作）。
- **手册页**：`mbuf(9)`、`ifnet(9)`、`vnet(9)`、`socket(9)`。
- **外部**：McKusick, Neville-Neil, 和 Watson, *The Design and Implementation of the FreeBSD Operating System*（第 2 版），网络子系统章节。

## 驱动程序基础设施 (Newbus)

### 子系统用于什么

Newbus 是 FreeBSD 的驱动程序框架。它拥有系统中的设备树，通过探测将驱动程序匹配到设备，管理每个附加的生命周期，将资源分配路由到正确的总线，并提供面向对象的分派，让总线可以覆盖和扩展彼此的行为。树中的每个字符驱动程序、存储驱动程序、网络驱动程序和嵌入式驱动程序都是 Newbus 参与者。如果本附录中的其他子系统是房间，Newbus 就是连接它们的走廊。

### 为什么驱动程序作者应该关心

本质上没有不带 Newbus 的 FreeBSD 驱动程序。你首先遇到的类型（`device_t`、softc、`driver_t`、`devclass_t`），你在 `attach` 中到达的 API（`bus_alloc_resource_any`、`bus_setup_intr`），你包装整个驱动程序的宏（`DRIVER_MODULE`、`DEVMETHOD`、`DEVMETHOD_END`）都属于这里。学会导航 Newbus 与学会导航 FreeBSD 驱动程序源代码是同一回事。

### 关键结构、接口或概念

- **`device_t`** 是 Newbus 设备树中节点的不透明句柄。你在 `probe` 和 `attach` 中接收一个，传递给几乎每个总线 API，并用它通过 `device_get_softc(9)` 获取 softc。
- **`driver_t`** 是驱动程序的描述符：其名称、其方法表和其 softc 的大小。你为驱动程序构造一个并将其交给 `DRIVER_MODULE(9)`，后者在父总线名称下注册它。
- **`devclass_t`** 是每驱动程序类的注册表：驱动程序附加的 `device_t` 实例的集合。它是内核给每个实例一个单元号的方式。
- **`kobj(9)`** 是 Newbus 下面的面向对象机制。方法表、方法分派以及总线继承另一个总线方法的能力都是 kobj 特性。作为驱动程序作者你使用 `DEVMETHOD` 宏，它们展开成 kobj 元数据；你很少直接调用 kobj 原语。
- **softc** 是驱动程序的每实例状态，由内核在分配 `device_t` 时分配。内核知道你的 softc 有多大，因为你在 `driver_t` 中告诉了它。`device_get_softc(9)` 给你返回指向它的指针。
- **`bus_alloc_resource(9)`** 和朋友代表你的驱动程序从父总线分配内存窗口、I/O 端口和中断线。它们是可移植地获取设备资源的方式，无需关心它坐在哪条总线上。

### 典型驱动程序接触点

- 声明一个带有 `DEVMETHOD(device_probe, ...)`、`DEVMETHOD(device_attach, ...)`、`DEVMETHOD(device_detach, ...)` 的 `device_method_t` 数组，以 `DEVMETHOD_END` 终止。
- 声明一个带有驱动程序名称、方法和 `sizeof(struct mydev_softc)` 的 `driver_t`。
- 使用 `DRIVER_MODULE(mydev, pci, mydev_driver, ...)`（或 `usbus`、`iicbus`、`spibus`、`simplebus`、`acpi`、`nexus`）在父总线下注册驱动程序。
- 在 `probe` 中，决定此设备是否属于你，如果是，返回 `BUS_PROBE_DEFAULT`（或更弱/更强的值）和描述。
- 在 `attach` 中，用 `bus_alloc_resource_any(9)` 分配资源，用 `bus_space(9)` 映射寄存器，用 `bus_setup_intr(9)` 设置中断，然后才向内核其余部分公开自己。
- 在 `detach` 中，按相反顺序撤销一切。

### 在 `/usr/src` 中查看的位置

- `/usr/src/sys/sys/bus.h` 声明 `device_t`、`driver_t`、`devclass_t`、`DEVMETHOD`、`DEVMETHOD_END`、`DRIVER_MODULE`、`bus_alloc_resource_any`、`bus_setup_intr` 和大部分其余内容。
- `/usr/src/sys/sys/kobj.h` 声明方法分派机制。
- `/usr/src/sys/kern/subr_bus.c` 保存 Newbus 实现。
- `/usr/src/sys/kern/subr_kobj.c` 保存 kobj 实现。
- `/usr/src/sys/dev/null/null.c` 和 `/usr/src/sys/dev/led/led.c` 是你可以在一次阅读中阅读的非常小的真实驱动程序。

### 接下来阅读的手册页和文件

`device(9)`、`driver(9)`、`DEVMETHOD(9)`、`DRIVER_MODULE(9)`、`bus_alloc_resource(9)`、`bus_setup_intr(9)`、`kobj(9)`，以及 `devinfo(8)` 查看运行中的 Newbus 树。

### 常见初学者困惑

两个陷阱。首先，认为 `device_t` 和 softc 是同一对象。`device_t` 是 Newbus 句柄；softc 是你的驱动程序的私有状态。你通过 `device_get_softc(9)` 从 `device_t` 获取 softc。其次，忘记 `DRIVER_MODULE` 的第二个参数是父总线名称。用 `DRIVER_MODULE(..., pci, ...)` 声明的驱动程序只能在 PCI 总线下附加，无论其他地方存在多少类 PCI 板。如果驱动程序必须在多个总线下附加（例如，同时作为 PCI 和 ACPI 出现的芯片），你注册两次。

### 本书在哪里教授这个

第 6 章是驱动程序解剖的完整章节，是上述一切的权威教学位置。第 7 章针对这些 API 编写第一个工作驱动程序。第 18 章为 PCI 扩展图景。第 24 章在内核集成成为主题时回到 `DRIVER_MODULE`、`MODULE_VERSION` 和 `MODULE_DEPEND`。

### 进一步阅读

- **本书中**：第 6 章（FreeBSD 驱动程序解剖），第 7 章（编写你的第一个驱动程序），第 18 章（编写 PCI 驱动程序）。
- **手册页**：`device(9)`、`driver(9)`、`DRIVER_MODULE(9)`、`bus_alloc_resource(9)`、`bus_setup_intr(9)`、`kobj(9)`、`rman(9)`。

## 引导和模块系统

### 子系统用于什么

引导和模块系统是内核如何进入内存、如何在任何东西运行之前初始化其依赖的数百个子系统，以及如何将未编译进内核的代码（可加载模块）引入、连接并最终移除。从驱动程序作者的角度，子系统定义你的初始化代码相对于内核其余部分何时运行，以及你的模块级 `MOD_LOAD` 和 `MOD_UNLOAD` 事件如何与内核的内部初始化顺序交互。

### 为什么驱动程序作者应该关心

三个原因。首先，如果你的驱动程序可以作为模块加载，它可能运行在子系统顺序与你预期不同的内核上，你需要声明你依赖什么。其次，如果你的驱动程序必须早期运行（例如，控制台驱动程序或引导时存储驱动程序），你必须理解 `SYSINIT(9)` 子系统 ID，以便你的代码在正确的槽位运行。第三，即使是普通驱动程序也依赖模块系统注册自己、声明 ABI 兼容性，以及在缺少依赖时干净地失败。

### 关键结构、接口或概念

- **引导序列** 遵循固定弧线：加载器从磁盘读取内核，将控制权交给内核入口点，后者设置早期 CPU 状态然后调用 `mi_startup()`。`mi_startup()` 遍历排序后的 `SYSINIT` 条目列表，按顺序调用每个。当列表耗尽时，内核有足够的服务启动 `init(8)` 作为用户进程 1。
- **`SYSINIT(9)`** 是注册函数在内核初始化的特定阶段被调用的宏。每个条目有子系统 ID（`SI_SUB_*`，粗排序）和子系统内顺序（`SI_ORDER_*`，细排序）。合法子系统 ID 的完整列表在 `/usr/src/sys/sys/kernel.h`，值得浏览一次。`SYSUNINIT(9)` 是匹配的拆除。
- **模块加载** 由 KLD 框架驱动。`kldload(8)` 调用链接器，后者重定位模块，针对运行内核解析其符号，并用 `MOD_LOAD` 调用模块的事件处理器。匹配的 `MOD_UNLOAD` 在模块移除时运行。驱动程序很少手写模块事件处理器；`DRIVER_MODULE(9)` 为你产生一个。
- **`MODULE_DEPEND(9)`** 声明你的模块需要另一个模块（`usb`、`miibus`、`pci`、`iflib`）存在，以及在哪个版本范围。内核拒绝加载你的模块如果缺少依赖。
- **`MODULE_VERSION(9)`** 声明你的模块导出的 ABI 版本，以便其他模块可以用 `MODULE_DEPEND` 依赖它。

### 典型驱动程序接触点

- `DRIVER_MODULE(mydev, pci, mydev_driver, ...)` 发出一个模块事件处理器，在 `MOD_LOAD` 上注册驱动程序并在 `MOD_UNLOAD` 上注销它。
- `MODULE_VERSION(mydev, 1);` 公布你的模块的 ABI 版本。
- `MODULE_DEPEND(mydev, pci, 1, 1, 1);` 声明 pci 依赖。
- 必须在 Newbus 可用前运行的驱动程序使用 `SYSINIT(9)` 在早期子系统 ID 注册一次性设置钩子。
- 在最后可能时刻连接拆除钩子的驱动程序使用带有匹配顺序的 `SYSUNINIT(9)`。

### 在 `/usr/src` 中查看的位置

- `/usr/src/sys/sys/kernel.h` 定义 `SYSINIT`、`SYSUNINIT`、`SI_SUB_*` 和 `SI_ORDER_*`。
- `/usr/src/sys/kern/init_main.c` 包含 `mi_startup()` 和遍历 SYSINIT 列表的行走。
- `/usr/src/sys/sys/module.h` 声明 `MODULE_VERSION` 和 `MODULE_DEPEND`。
- `/usr/src/sys/sys/linker.h` 和 `/usr/src/sys/kern/kern_linker.c` 实现 KLD 链接器。
- `/usr/src/stand/` 保存加载器和引导时代码。（在较旧的 FreeBSD 发布版中，这生活在 `/usr/src/sys/boot/`；FreeBSD 14 将其完全托管在 `/usr/src/stand/`。）

### 接下来阅读的手册页和文件

`SYSINIT(9)`、`kld(9)`、`kldload(9)`、`kldload(8)`、`kldstat(8)`、`module(9)`、`MODULE_VERSION(9)` 和 `MODULE_DEPEND(9)`。对于简短的真实 `SYSINIT` 示例，查看 `/usr/src/sys/dev/random/random_harvestq.c` 的顶部附近。

### 常见初学者困惑

两个陷阱。首先，假设 `MOD_LOAD` 是你的 `attach` 函数运行的时刻。它不是。`MOD_LOAD` 是你的*驱动程序*注册到 Newbus 的时刻；`attach` 稍后运行，每设备一次，每当总线提供匹配的子设备。其次，将 `SYSINIT` 级别视为任意的。每个 `SI_SUB_*` 对应内核启动的定义良好阶段，在错误阶段注册你的钩子要么使其运行太早（缺少内核一半），要么太晚（在你关心的事件通过后）。

### 本书在哪里教授这个

第 6 章作为驱动程序解剖的一部分介绍 `DRIVER_MODULE`、`MODULE_VERSION` 和 `MODULE_DEPEND`。第 24 章覆盖内核集成主题，包括 `SYSINIT`、子系统 ID 和模块拆除顺序。第 32 章回到嵌入式平台的引导时关注。

### 进一步阅读

- **本书中**：第 24 章（与内核集成），第 32 章（设备树和嵌入式开发）。
- **手册页**：`SYSINIT(9)`、`module(9)`、`MODULE_VERSION(9)`、`MODULE_DEPEND(9)`、`kldload(8)`、`kldstat(8)`。

## 内核服务

### 子系统用于什么

内核附带一组不绑定任何特定子系统但在驱动程序中反复出现的通用服务：事件通知、延迟工作队列、定时回调和订阅钩子。它们中没有教你如何编写驱动程序，但它们都出现在真实驱动程序代码中，识别它们会加速每次代码阅读会话。本节收集你可能遇到的一小部分。

### 为什么驱动程序作者应该关心

驱动程序经常需要对系统范围事件做出反应（关机、内存不足、接口到达、根文件系统挂载），或需要在传递事件的上下文之外做工作（远离中断过滤器、远离自旋锁临界区）。下面的内核服务是对这两种需求的标准 FreeBSD 答案。使用它们意味着你的驱动程序与系统其余部分干净集成；重新实现它们意味着你最终会与期望你的钩子存在的子系统冲突。

### 关键结构、接口或概念

- **`eventhandler(9)`** 是内核事件的发布/订阅系统。发布者用 `EVENTHANDLER_DECLARE` 声明事件，订阅者用 `EVENTHANDLER_REGISTER` 注册，用 `EVENTHANDLER_INVOKE` 调用会扇出到每个订阅者。`/usr/src/sys/sys/eventhandler.h` 中定义的标准事件标签包括 `shutdown_pre_sync`、`shutdown_post_sync`、`shutdown_final`、`vm_lowmem` 和 `mountroot`；接口事件（`ifnet_arrival_event`、`ifnet_departure_event`）在 `/usr/src/sys/net/if_var.h` 中声明。驱动程序使用这些来清理、释放内存、在兄弟接口出现时做出反应，或延迟早期工作直到根文件系统可用。
- **`taskqueue(9)`** 是延迟工作项的队列。驱动程序从不能睡眠的上下文（例如中断过滤器）将任务入队，任务稍后在允许多睡眠和阻塞的专用工作者线程上运行。内核附带一小组系统范围的任务队列（`taskqueue_swi`、`taskqueue_thread`、`taskqueue_fast`）并让你创建自己的。
- **分组任务队列（`gtaskqueue`）** 扩展 `taskqueue` 带 CPU 亲和性和重平衡；它们在 `iflib` 和高速网络栈中大量使用。声明生活在 `/usr/src/sys/sys/gtaskqueue.h`。
- **`callout(9)`** 是内核的一次性和周期性定时器。驱动程序用未来截止时间武装 callout 并在截止时间到达时接收回调。`callout(9)` 替换几乎每个驱动程序可能编写的临时"睡眠 N 个滴答"循环。
- **`hooks(9)` 风格子系统扩展点。** 许多 FreeBSD 子系统发布行为类似事件处理器但特定于子系统的注册 API（例如，数据包过滤器用 `pfil(9)` 注册；磁盘驱动程序可以用 `disk(9)` 事件注册）。这些不是统一的接口，但模式相同：子系统在定义良好的时刻调用的回调列表。

### 典型驱动程序接触点

- `attach` 中的 `EVENTHANDLER_REGISTER(shutdown_pre_sync, mydev_shutdown, softc, SHUTDOWN_PRI_DEFAULT);` 以便驱动程序在重启前刷新硬件；`detach` 中的 `EVENTHANDLER_DEREGISTER`。（关闭钩子的三个标准优先级常量是 `SHUTDOWN_PRI_FIRST`、`SHUTDOWN_PRI_DEFAULT` 和 `SHUTDOWN_PRI_LAST`，在 `/usr/src/sys/sys/eventhandler.h` 中声明。）
- `attach` 中的 `taskqueue_create("mydev", M_WAITOK, ...); taskqueue_start_threads(...);` 创建每设备工作者；`detach` 中的 `taskqueue_drain_all` 和 `taskqueue_free`。
- `attach` 中的 `callout_init_mtx(&sc->sc_watchdog, &sc->sc_mtx, 0)` 武装看门狗；`detach` 中的 `callout_drain`。
- 分组任务队列在 `iflib` 网络驱动程序中最可见；典型的独立驱动程序很少直接到达它们。

### 在 `/usr/src` 中查看的位置

- `/usr/src/sys/sys/eventhandler.h` 和 `/usr/src/sys/kern/subr_eventhandler.c` 用于事件处理器。
- `/usr/src/sys/sys/taskqueue.h` 和 `/usr/src/sys/kern/subr_taskqueue.c` 用于任务队列。
- `/usr/src/sys/sys/gtaskqueue.h` 和 `/usr/src/sys/kern/subr_gtaskqueue.c` 用于分组任务队列。
- `/usr/src/sys/sys/callout.h` 和 `/usr/src/sys/kern/kern_timeout.c` 用于 callout。

### 接下来阅读的手册页和文件

`eventhandler(9)`、`taskqueue(9)`、`callout(9)`，以及头文件 `/usr/src/sys/sys/eventhandler.h`。查看 `/usr/src/sys/dev/random/random_harvestq.c` 获取干净使用 `SYSINIT` 和专用 kproc 的驱动程序；即使它本身不练习 `taskqueue(9)` 或 `callout(9)`，阅读关于内核服务时它是好的配套。

### 常见初学者困惑

一个重要陷阱：忘记注册是契约的一半。每个 `EVENTHANDLER_REGISTER` 需要在匹配生命周期时刻的 `EVENTHANDLER_DEREGISTER`，每个 `taskqueue_create` 需要 `taskqueue_free`，每个武装的 `callout` 在其内存释放前需要 `callout_drain`。泄漏的注册保留指向释放内存的悬空指针；事件的下次调用将使内核在与你的驱动程序无关的子系统中崩溃。

### 本书在哪里教授这个

第 13 章介绍 `callout(9)`。第 14 章是任务队列章节。第 24 章是内核集成章节，在上下文中覆盖 `eventhandler(9)` 和 SYSINIT/模块合作。

### 进一步阅读

- **本书中**：第 13 章（定时器和延迟工作），第 14 章（任务队列和延迟工作），第 24 章（与内核集成）。
- **手册页**：`eventhandler(9)`、`taskqueue(9)`、`callout(9)`。

## 交叉引用：结构及其子系统

下面的表是将不熟悉类型转换为已知子系统的最快方式。当你在阅读驱动程序源代码、遇到不认识的结构名并想知道打开本附录的哪一节时使用它。

| 结构或类型         | 子系统                     | 声明位置                                     |
| :------------------------ | :---------------------------- | :------------------------------------------------- |
| `struct proc`, `thread`   | 进程和线程            | `/usr/src/sys/sys/proc.h`                          |
| `vm_map_t`                | 内存 (VM)                   | `/usr/src/sys/vm/vm.h` 和 `/usr/src/sys/vm/vm_map.h` |
| `vm_object_t`             | 内存 (VM)                   | `/usr/src/sys/vm/vm.h` 和 `/usr/src/sys/vm/vm_object.h` |
| `vm_page_t`               | 内存 (VM)                   | `/usr/src/sys/vm/vm.h` 和 `/usr/src/sys/vm/vm_page.h` |
| `uma_zone_t`              | 内存 (VM)                   | `/usr/src/sys/vm/uma.h`                            |
| `struct vnode`            | 文件和 VFS                  | `/usr/src/sys/sys/vnode.h`                         |
| `struct vop_vector`       | 文件和 VFS                  | 从 `/usr/src/sys/kern/vnode_if.src` 生成    |
| `struct buf`              | 文件和 VFS                  | `/usr/src/sys/sys/buf.h`                           |
| `struct bio`              | 文件和 VFS (GEOM)           | `/usr/src/sys/sys/bio.h`                           |
| `struct g_provider`       | 文件和 VFS (GEOM)           | `/usr/src/sys/geom/geom.h`                         |
| `struct cdev`             | 文件和 VFS (devfs)          | `/usr/src/sys/sys/conf.h`                          |
| `struct cdevsw`           | 文件和 VFS (devfs)          | `/usr/src/sys/sys/conf.h`                          |
| `struct mbuf`, `m_tag`    | 网络                       | `/usr/src/sys/sys/mbuf.h`                          |
| `if_t`, `struct ifnet`    | 网络                       | `/usr/src/sys/net/if.h`, `/usr/src/sys/net/if_private.h` |
| `struct socket`           | 网络                       | `/usr/src/sys/sys/socketvar.h`                     |
| `device_t`                | 驱动程序基础设施         | `/usr/src/sys/sys/bus.h`                           |
| `driver_t`, `devclass_t`  | 驱动程序基础设施         | `/usr/src/sys/sys/bus.h`                           |
| `device_method_t`         | 驱动程序基础设施 (kobj)  | `/usr/src/sys/sys/bus.h` (kobj 在 `sys/kobj.h`)    |
| `struct resource`         | 驱动程序基础设施         | `/usr/src/sys/sys/rman.h`                          |
| `SYSINIT`, `SI_SUB_*`     | 引导和模块               | `/usr/src/sys/sys/kernel.h`                        |
| `MODULE_VERSION`, `MODULE_DEPEND` | 引导和模块       | `/usr/src/sys/sys/module.h`                        |
| `eventhandler_tag`        | 内核服务               | `/usr/src/sys/sys/eventhandler.h`                  |
| `struct taskqueue`        | 内核服务               | `/usr/src/sys/sys/taskqueue.h`                     |
| `struct callout`          | 内核服务               | `/usr/src/sys/sys/callout.h`                       |

当类型不在表中时，在 `/usr/src/sys/sys/` 或 `/usr/src/sys/<subsystem>/` 中搜索其声明；定义附近的注释通常直接命名子系统。

## 源代码树导航检查表

FreeBSD 源代码树按职责组织，一旦你知道模式，你几乎可以猜测任何东西生活的位置。下面的列表是将"在树的哪里"转换为"打开这个文件"的五个快速问题。

### 当你有结构名时

1. 它是低级原语（`proc`、`thread`、`vnode`、`buf`、`bio`、`mbuf`、`callout`、`taskqueue`、`eventhandler`）？首先看 `/usr/src/sys/sys/`。
2. 它是 VM 类型（`vm_*`、`uma_*`）？看 `/usr/src/sys/vm/`。
3. 它是网络类型（`ifnet`、`if_*`、`m_tag`、`route`、`socket`、`vnet`）？看 `/usr/src/sys/net/`、`/usr/src/sys/netinet/` 或 `/usr/src/sys/netinet6/`。
4. 它是设备或总线类型（`device_t`、`driver_t`、`resource`、`rman`、`pci_*`、`usbus_*`）？看 `/usr/src/sys/sys/bus.h`、`/usr/src/sys/sys/rman.h` 或 `/usr/src/sys/dev/` 下的匹配总线目录。
5. 它是完全其他东西？`grep -r 'struct NAME {' /usr/src/sys/sys/ /usr/src/sys/kern/ /usr/src/sys/vm/ /usr/src/sys/net/` 通常一次通过就能找到它。

### 当你有函数名时

1. 如果名称以 `vm_` 开头，它生活在 `/usr/src/sys/vm/`。
2. 如果以 `bus_`、`device_`、`driver_`、`devclass_`、`resource_` 开头，它生活在 `/usr/src/sys/kern/subr_bus.c`、`/usr/src/sys/kern/subr_rman.c` 或特定于总线的目录之一。
3. 如果以 `vfs_`、`vn_` 或 `VOP_` 前缀开头，它生活在 `/usr/src/sys/kern/vfs_*.c` 或 `/usr/src/sys/fs/` 下的文件系统之一。
4. 如果以 `g_` 开头，它是 GEOM；看 `/usr/src/sys/geom/`。
5. 如果以 `if_`、`ether_` 或 `in_` 开头，它是网络；看 `/usr/src/sys/net/` 或 `/usr/src/sys/netinet/`。
6. 如果以 `kthread_`、`kproc_`、`sched_` 或 `proc_` 开头，它是 `/usr/src/sys/kern/` 下的进程/线程子系统。
7. 如果以 `uma_` 或 `malloc` 开头，它是内存；看 `/usr/src/sys/vm/uma_core.c` 或 `/usr/src/sys/kern/kern_malloc.c`。
8. 当什么都不匹配时，`grep -rl '\bFUNC_NAME\s*(' /usr/src/sys/` 较慢但详尽。

### 当你有宏名时

1. `SYSINIT`、`SYSUNINIT`、`SI_SUB_*`、`SI_ORDER_*`：`/usr/src/sys/sys/kernel.h`。
2. `DRIVER_MODULE`、`DEVMETHOD`、`DEVMETHOD_END`、`MODULE_VERSION`、`MODULE_DEPEND`：`/usr/src/sys/sys/bus.h` 和 `/usr/src/sys/sys/module.h`。
3. `EVENTHANDLER_*`：`/usr/src/sys/sys/eventhandler.h`。
4. `VNET_*`、`CURVNET_*`：`/usr/src/sys/net/vnet.h`。
5. `TAILQ_*`、`LIST_*`、`STAILQ_*`、`SLIST_*`：`/usr/src/sys/sys/queue.h`。
6. `VOP_*`：从 `/usr/src/sys/kern/vnode_if.src` 生成，一旦内核构建在 `sys/vnode_if.h` 中可见。

### 当你有子系统问题时

1. 什么初始化内核以及按什么顺序？`/usr/src/sys/kern/init_main.c`。
2. 树包含哪些驱动程序？`ls /usr/src/sys/dev/` 及其子目录。
3. 网络栈入口点在哪里？`/usr/src/sys/net/if.c`、`/usr/src/sys/netinet/` 及其兄弟。
4. 特定系统调用如何到达驱动程序？从 `/usr/src/sys/kern/syscalls.master` 开始，跟随分派器进入相关 VFS 或套接字代码，并继续阅读直到分派落在 `cdevsw`、`vop_vector` 或 `ifnet`。

## 手册页和源代码阅读行程

跨内核的模式识别来自阅读它，而不仅仅是阅读关于它。覆盖本附录中子系统的自学计划可能看起来像这样：

1. `intro(9)` 加上 `/usr/src/sys/sys/` 文件名遍历，总共十五分钟。
2. `kthread(9)`、`kproc(9)` 和 `/usr/src/sys/sys/proc.h`。
3. `malloc(9)`、`uma(9)`、`bus_dma(9)` 和 `/usr/src/sys/vm/uma.h`。
4. `vnode(9)`、`cdev(9)`、`make_dev(9)`、`devfs(4)` 和 `/usr/src/sys/dev/null/null.c`。
5. `mbuf(9)`、`ifnet(9)`、`ether_ifattach(9)` 和 `/usr/src/sys/net/if_tuntap.c`。
6. `device(9)`、`DRIVER_MODULE(9)`、`bus_alloc_resource(9)` 和 `/usr/src/sys/dev/led/led.c`。
7. `SYSINIT(9)`、`kld(9)`、`module(9)` 和 `/usr/src/sys/kern/init_main.c` 的顶部。
8. `eventhandler(9)`、`taskqueue(9)`、`callout(9)` 和 `/usr/src/sys/dev/random/random_harvestq.c`。

`examples/appendices/appendix-e-navigating-freebsd-kernel-internals/` 中的配套文件以你可以打印、注释并保存在机器旁边的形式收集相同的行程。

## 总结：如何安全地继续探索内核

探索内核源代码树可能感觉无止境，很容易通过十个子系统追逐一个有趣的线程而失去一个周末。一小套习惯使探索保持高效。

带着特定问题在短会话中阅读。"`bus_setup_intr` 底层实际做什么"是一个好的会话。"阅读 VM"不是。

保持地图在视野中。当你从驱动程序跳到 VFS 时，提醒自己你现在在 VFS 中，VFS 的规则适用。当你返回驱动程序时，提醒自己 VFS 停在函数边界。每个子系统都有自己的不变量和自己的锁定规则，它们很少延续。

写下你发现的。像"`subr_bus.c` 中的 `bus_alloc_resource_any` 通过 kobj 分派调用 `BUS_ALLOC_RESOURCE`，PCI 总线方法在 `pci.c` 中实现"这样的简短笔记比一下午被动阅读更有价值。本附录及其配套文件正是为了给你这种笔记的锚点。

使用安全轨道。`/usr/src/sys/dev/null/null.c` 和 `/usr/src/sys/dev/led/led.c` 很小。`/usr/src/sys/net/if_tuntap.c` 小到可以在一次阅读中阅读。`/usr/src/sys/dev/random/random_harvestq.c` 使用真实内核服务而不隐藏在抽象层后面。每当子系统感觉太大而无法直接接近时，从这些开始。

记住目标不是背诵内核。它是建立足够的模式识别，使下次你打开不熟悉的驱动程序或新子系统时，结构、函数和源代码路径感觉像你已经走过的街区。本附录，连同附录 A 到 D 和在上下文中教授各部分的章节，旨在使那种感觉更早到达。

当这里的地图不够时，本书够了。当本书不够时，源代码够了。而源代码已经坐在你的 FreeBSD 机器上，等待被阅读。
