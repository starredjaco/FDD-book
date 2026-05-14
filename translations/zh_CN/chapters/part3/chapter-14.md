---
title: "任务队列与延迟工作"
description: "FreeBSD驱动程序如何将工作从无法睡眠的上下文移至可以睡眠的线程：从定时器和中断安全地入队任务、构建私有任务队列、合并突发工作、在分离时干净地清空、并调试结果。"
partNumber: 3
partName: "并发与同步"
chapter: 14
lastUpdated: "2026-04-18"
status: "complete"
author: "Edson Brandi"
reviewer: "TBD"
translator: "TBD"
estimatedReadTime: 195
language: "zh-CN"
---

# 任务队列与延迟工作

## 读者指南与学习成果

在第13章结束时，你的 `myfirst` 驱动获得了一种微小但真实的内部时间感。它可以用 `callout(9)` 调度周期性工作，发出心跳，用看门狗检测排放停滞，用 tick 源注入合成字节。每个回调都遵循严格的纪律：获取已注册的互斥锁，检查 `is_attached`，做短小有界的工作，可能重新武装，释放互斥锁。正是这种纪律使定时器安全，也正是这种纪律使它们变得狭窄。

第14章直面这种狭窄性。callout 回调运行在一个不能睡眠的上下文中。它不能调用 `uiomove(9)`，不能调用 `copyin(9)`，不能获取可睡眠的 `sx(9)` 锁，不能用 `M_WAITOK` 分配内存，不能在持有睡眠互斥锁时调用 `selwakeup(9)`。如果定时器想要触发的工作需要其中任何一种，定时器就必须将工作移交出去。同样的限制也适用于你将在第四部分遇到的中断处理程序，以及整个内核中出现的其他几种受限上下文。内核为此提供了一个单一原语：`taskqueue(9)`。

任务队列最简单的形式是一个由小型工作项组成的队列，配以一个或多个消费该队列的内核线程。你的受限上下文将任务入队；任务队列的线程被唤醒并在进程上下文中运行任务的回调，那里适用普通的内核规则。任务可以睡眠，可以自由分配内存，可以触碰可睡眠的锁。任务队列子系统还知道如何合并突发入队、取消待处理的工作、在拆除时等待进行中的工作，以及在特定的未来时刻调度任务。所有这些都在一个小的 API 表面上实现，并且所有这些正是使用 callout 或中断的驱动程序所需要的。

本章以第13章讲解 `callout(9)` 的同样细致程度来教授 `taskqueue(9)`。我们从问题的形态开始，逐步介绍 API，然后通过四个阶段演进 `myfirst` 驱动，将基于任务的延迟工作添加到现有的定时器基础设施上。到本章结束时，驱动将使用一个私有任务队列，将所有不能在 callout 或中断上下文中运行的工作移出这些上下文，并在分离时干净地拆除任务队列，不会泄漏过时任务、唤醒死线程或损坏任何东西。

### 为什么本章有其独立的价值

你可以假装任务队列不存在。你的 callout 可以尝试内联执行延迟工作，接受第一次 `WITNESS` 检测到持有自旋锁时睡眠而导致内核崩溃的后果，并希望没有人会在调试内核上加载你的驱动。这不是一个真正的选项，我们不会拿它当回事。本章的目的是给你一个诚实的替代方案，内核其余部分实际使用的那个。

你也可以用 `kproc_create(9)` 和自定义条件变量来搭建自己的延迟工作框架。这在技术上是可行的，偶尔也不可避免，但几乎总是错误的首选。自定义线程比任务更重量级，而且它缺少使用共享框架时免费获得的可观察性。`ps(1)`、`procstat(1)`、`dtrace(1)`、`ktr(4)`、`wchan` 跟踪和 `ddb(4)` 都理解任务队列线程。除非你自己为一次性的辅助线程添加插桩，否则它们不会理解你的辅助线程。

在几乎每个驱动程序需要将工作移出受限上下文的情况下，任务队列都是正确的答案。不了解它们的代价高于学习它们的代价，而学习它们的代价是适中的：API 比 `callout(9)` 更小，规则是规则的，习语可以直接跨驱动程序转移。一旦心智模型到位，你会开始在 `/usr/src/sys/dev/` 下的几乎每个驱动程序中识别出这种模式。

### 第13章结束后驱动的状态

在继续之前快速检查一下。第14章扩展的是第13章第4阶段结束时产生的驱动程序，不是任何更早的阶段。如果以下任何一项让你感到不确定，请在开始本章之前返回第13章。

- 你的 `myfirst` 驱动编译干净，版本标识为 `0.7-timers`。
- 它在 softc 中声明了三个 callout：`heartbeat_co`、`watchdog_co` 和 `tick_source_co`。
- 每个 callout 在 `myfirst_attach` 中用 `callout_init_mtx(&co, &sc->mtx, 0)` 初始化，并在 `is_attached` 被清除后用 `callout_drain` 排空。
- 每个 callout 有一个间隔 sysctl（`heartbeat_interval_ms`、`watchdog_interval_ms`、`tick_source_interval_ms`），默认为零（禁用），并在其处理程序中反映启用/禁用转换。
- 分离路径按文档记录的顺序运行：在 `active_fhs` 上拒绝，清除 `is_attached`，广播两个 cv，排空 `selinfo`，排空所有 callout，销毁设备，释放 sysctl，销毁 cbuf、计数器、cv、sx 和互斥锁。
- 你的 `LOCKING.md` 有一个 Callouts 节，命名每个 callout、其回调、其锁及其生命周期。
- 第13章压力测试套件（第12章测试器加第13章定时器测试）在启用 `WITNESS` 和 `INVARIANTS` 时构建并干净运行。

那个驱动就是我们要扩展的形态。第14章不会重构任何这些结构。它在 softc 中添加一个新列，一个新的初始化调用，一个新的拆除调用，以及对三个 callout 回调和一个或两个其他地方的少量更改，这些地方驱动程序会受益于将工作移出受限上下文。

### 你将学到什么

到本章结束时你将能够：

- 解释为什么某些工作不能在 callout 回调或中断处理程序中完成，并识别强制将工作移交给不同上下文的操作。
- 描述任务队列的三要素：`struct task` 的队列、消费该队列的线程（或小型线程池），以及将两者联系在一起的入队/调度策略。
- 使用 `TASK_INIT(&sc->foo_task, 0, myfirst_foo_task, sc)` 初始化任务，理解每个参数的含义，并将调用放在 attach 的正确阶段。
- 从 callout 回调、sysctl 处理程序、读或写路径，或任何其他延迟工作是正确答案的驱动程序代码中使用 `taskqueue_enqueue(tq, &sc->foo_task)` 将任务入队。
- 在预定义的系统任务队列（`taskqueue_thread`、`taskqueue_swi`、`taskqueue_swi_giant`、`taskqueue_fast`、`taskqueue_bus`）与你用 `taskqueue_create` 创建并用 `taskqueue_start_threads` 启动的私有任务队列之间做出选择。
- 理解合并契约：当任务在已待处理时被入队，内核递增 `ta_pending` 而不是将其链接两次，回调获得最终的待处理计数以便进行批量处理。
- 使用 `struct timeout_task` 变体配合 `taskqueue_enqueue_timeout` 在特定的未来时刻调度任务，并用 `taskqueue_drain_timeout` 正确排空它。
- 在精细的关闭步骤周围阻塞和解阻塞任务队列，以及当你需要保证队列中没有任务正在运行时静默任务队列。
- 在分离时排空驱动拥有的每个任务，以正确的顺序，不会与你已经排空的 callout 和 cv 发生死锁。
- 在驱动源码中分离定时器代码和任务代码的关注点，使新读者只需查看文件就能判断哪些工作在哪个上下文中运行。
- 认识并应用使用 `epoch(9)` 和组任务队列实现无锁读路径的网络驱动模式，达到知道何时使用它们何时不用的水平。
- 使用 `procstat -t`、`ps ax`、`dtrace -l` 和 `ktr(4)` 调试使用任务队列的驱动，并解释每个工具向你展示的内容。
- 将驱动标记为版本 `0.8-taskqueues`，并在 `LOCKING.md` 中记录延迟策略，使下一个继承驱动的人能够阅读它。

这是一个很长的列表。大多数条目相互构建，因此章节内的递进就是自然的路径。

### 本章不涵盖的内容

几个相邻主题被明确推迟，以保持第14章的专注。

- **中断处理程序作为主要主题。** 第四部分介绍 `bus_setup_intr(9)` 以及 `FILTER` 和 `ITHREAD` 处理程序之间的分离。第14章在解释延迟工作的重要性时提到中断上下文，它教授的模式可以直接从 callout 转移到真正的中断处理程序，但中断 API 本身是第四部分的工作。
- **完整的条件变量和信号量故事。** 第15章用计数信号量、信号可中断阻塞和协调定时器、任务和用户线程的跨组件握手来扩展同步词汇。第14章按原样使用现有的 cv 基础设施，不添加超出任务队列本身带来的新同步原语。
- **组任务队列和 iflib 的深入覆盖。** `taskqgroup` 系列存在，本章解释何时它是正确答案，但完整的故事属于第六部分的网络驱动（第28章）。这里的介绍有意保持轻量。
- **硬件驱动的 DMA 完成路径。** 任务队列是在中断发出完成信号后完成 DMA 传输的自然场所，我们提到了这种模式，但 DMA 缓冲区管理的机制等到总线空间和 DMA 章节再讲。
- **工作循环、每 CPU 轮询的内核线程和高级调度器钩子。** 这些是内核延迟工作领域的真实组成部分，但它们是专用的，驱动程序很少触及。当它们重要时，需要它们的章节会引入它们。

保持在这些界限内使章节的心智模型保持一致。第14章给你一个精心教授的好工具。后续章节给你相邻的工具和证明使用它们合理的真实硬件上下文。

### 预计时间投入

- **仅阅读**：约三小时。API 表面比 `callout(9)` 小，但任务队列与驱动其余锁定故事的交互需要一点时间来消化。
- **阅读加输入工作示例**：两次会话共六至八小时。驱动分四个阶段演进；每个阶段大约改变一个关注点。
- **阅读加所有实验和挑战**：三到四次会话共十至十四小时，包括用 `procstat`、`dtrace` 和压力工作负载观察任务队列线程所需的时间。

如果你发现第4节开头的排序规则令人困惑，那是正常的。带有 callout、cv、sel 处理程序以及现在任务的分离序列有四个必须正确组合的部分。我们将遍历一次顺序，陈述它，论证它，然后重用它。

### 先决条件

在开始本章之前，请确认：

- 你的驱动源码匹配第13章第4阶段（`stage4-final`）。起始点假设有三个 callout、三个间隔 sysctl、每个回调中的 `is_attached` 纪律，以及文档记录的分离顺序。
- 你的实验机器运行 FreeBSD 14.3，磁盘上有 `/usr/src` 并与运行中的内核匹配。本章的几个源码引用是你应该实际打开和阅读的内容。
- 一个启用了 `INVARIANTS`、`WITNESS`、`WITNESS_SKIPSPIN`、`DDB`、`KDB` 和 `KDB_UNATTENDED` 的调试内核已构建、安装并干净启动。
- 第13章让你感到舒适。锁感知的 callout、回调中的 `is_attached` 纪律和分离排序是这里的假设知识。
- 你至少运行过一次第13章的压力测试套件，每个定时器都已启用，并看到它干净通过。

如果以上任何一项不够扎实，现在修复它是比勉强推进第14章并试图在不稳固的基础上调试更好的投资。第14章的模式专门设计为与第13章的模式组合；从一个不太正确的第13章驱动开始会使第14章的每一步都更难。

### 如何从本章获得最大收益

三个习惯会很快带来回报。

首先，将 `/usr/src/sys/kern/subr_taskqueue.c` 和 `/usr/src/sys/sys/taskqueue.h` 加入书签。头文件很短，大约两百行，是 API 的权威总结。实现文件大约一千行，注释完善，仔细阅读 `taskqueue_run_locked` 在你第一次需要推理任务的 `pending` 计数实际含义时会物有所值。现在花十分钟读头文件，日后省十小时的信心。

其次，在 `WITNESS` 下运行每一次代码更改。任务队列子系统有自己的锁（自旋互斥锁或睡眠互斥锁，取决于队列是用 `taskqueue_create` 还是 `taskqueue_create_fast` 创建的），它以 `WITNESS` 理解的方式与你的驱动锁交互。任务回调中错位的锁获取正是 `WITNESS` 在调试内核上立即捕获而在生产内核上静默损坏的那种 bug。在通过调试内核之前不要在生产内核上运行第14章的代码。

第三，手动输入更改。`examples/part-03/ch14-taskqueues-and-deferred-work/` 下的配套源码是权威版本，但肌肉记忆比阅读更有价值。本章引入小的增量编辑；在你自己的驱动副本中镜像这种小步节奏。当测试环境在某个阶段通过时，提交那个版本然后继续；当某一步中断时，上一个提交就是你的恢复点。

### 本章路线图

各节顺序如下：

1. 为什么在驱动中使用延迟工作。问题的形态：在 callout、中断和其他受限上下文中不能做什么；迫使移交的真实世界案例。
2. `taskqueue(9)` 简介。结构、API、预定义队列，以及与 callout 的比较。
3. 从定时器或模拟中断延迟工作。第一次重构，第1阶段：添加一个由 callout 入队的单一任务。
4. 任务队列设置与清理。第2阶段：创建私有任务队列，连接分离序列，并对照 `WITNESS` 审计结果。
5. 工作的优先级与合并。第3阶段：故意使用 `ta_pending` 合并行为进行批处理，引入 `taskqueue_enqueue_timeout` 用于调度任务，并讨论优先级。
6. 使用任务队列的真实模式。在真实 FreeBSD 驱动中反复出现的模式之旅，以可直接用于你自己代码的小型方案呈现。
7. 调试任务队列。工具、常见错误，以及一个在真实场景上引导式破坏-修复练习。
8. 重构与版本化。第4阶段：将驱动整合为一个连贯的整体，将版本提升到 `0.8-taskqueues`，并扩展 `LOCKING.md`。

在八个主要节之后，我们以轻松的介绍级别覆盖 `epoch(9)`、组任务队列和每 CPU 任务队列，然后是动手实验、挑战练习、故障排除参考、收尾部分，以及通往第15章的桥梁。

如果是第一次阅读，请线性阅读并按顺序做实验。如果是复习，第4、6和8节可以独立阅读，适合单次阅读。



## 第1节：为什么在驱动中使用延迟工作？

第13章结束时，驱动的 callout 完成了回调可以安全完成的每项工作。心跳打印一行状态报告。看门狗记录单个计数并可选地打印警告。tick 源向循环缓冲区写入单个字节并通知条件变量。每个回调花费微秒，在这几微秒内持有互斥锁，然后返回。这就是 callout 契约的最佳状态：小型、可预测、锁感知、低成本。

真实的驱动工作并不总是能适配这个契约。有些任务希望以注意到需求的定时器相同的节奏运行，但它们想做定时器不能安全做的事情。其他任务由不同的受限上下文触发（例如中断处理程序，或网络栈中的过滤例程），但有同样的"不能在这里做"的问题。本节概述问题的形态：在受限上下文中不能做什么、驱动想要延迟什么样的工作，以及内核提供了哪些将工作转移到可以实际运行的地方的选项。

### 重温定时调用契约

简短地重读 callout 规则，因为任务队列的故事完全关于 callout 不能做的事情。

`callout(9)` 回调以两种模式之一运行。默认模式是从 callout 线程调度：该 CPU 的专用 callout 线程在硬件时钟边界上唤醒，遍历 callout 轮盘，找到截止时间已到的回调，并逐个调用它们。另一种模式 `C_DIRECT_EXEC` 直接在硬件时钟中断处理程序内部运行回调。你的驱动很少选择替代模式；几乎所有驱动都使用默认模式。

在两种模式下，回调都持有 callout 注册的锁运行（对于 `callout_init_mtx` 系列），并且不能跨越某些上下文边界。它不能睡眠。睡眠意味着调用任何可以取消调度线程并无限期等待条件的原语。`mtx_sleep`、`cv_wait`、`msleep`、`sx_slock`、`malloc(..., M_WAITOK)`、`uiomove`、`copyin` 和 `copyout` 在其慢路径上都会睡眠。`selwakeup(9)` 本身不睡眠，但它获取每 selinfo 的互斥锁，这可能是 callout 运行上下文中错误的互斥锁，标准做法是在没有驱动互斥锁持有的情况下调用它。这些调用都不属于 callout 回调内部。

这些在内核层面是硬性规则。`INVARIANTS` 和 `WITNESS` 在运行时捕获许多违规。其中一些会静默地损坏内核，以后很难调试。在所有情况下，想要获得这些操作效果的驱动必须从允许它的上下文进行调用。那个上下文就是任务队列提供的上下文。

本节的其余部分从不同角度扩展同一观察：驱动想要延迟什么样的工作、为什么受限上下文值得这些约束、以及哪些 FreeBSD 设施竞争这项工作。

### 受限上下文，不仅是定时调用

callout 是 `myfirst` 风格驱动遇到的第一个受限上下文，但它不是唯一的。内核中有其他几个地方运行的代码不能睡眠或不能做某些类型的分配。想要从其中任何一个采取行动的驱动都面临同样的"延迟它"的决定。

**硬件中断过滤器。** 当真实设备引发中断时，内核在接收中断的 CPU 上同步运行过滤例程。过滤器不能睡眠，不能获取睡眠互斥锁，不能调用内核大多数常规 API。它们通常被拆分为一个微型过滤器（读状态寄存器，判断中断是否属于我们）在硬件上下文中运行，加上一个关联的 ithread（中断线程）在完整线程上下文中运行真正的工作。当第四部分引入 `bus_setup_intr(9)` 时，我们将遇到精确的过滤器/ithread 分割，但结构性的教训现在就很清楚：中断过滤器是另一个必须将工作移交到其他地方的位置。

**网络包输入路径。** `ifnet(9)` 接收路径的某些部分在 `epoch(9)` 保护下运行，限制了安全的锁获取和睡眠操作类型。网络驱动在想要做属于进程上下文的非平凡工作时频繁入队任务。

**`taskqueue_fast` 和 `taskqueue_swi` 回调。** 即使你已经在任务回调内部，如果任务运行在自旋互斥锁支持的队列（`taskqueue_fast`）或软件中断队列（`taskqueue_swi`）上，同样的禁止睡眠规则与原始上下文一样适用。在默认的 `taskqueue_thread` 上的任务回调没有这种限制；它们在完整线程上下文中运行，可以自由睡眠。这个区别很重要，我们将在第2节回到这个话题。

**`epoch(9)` 读段。** 由 `epoch_enter()` 和 `epoch_exit()` 括起来的代码路径不能睡眠。网络驱动大量使用这种模式使读路径无锁；写侧工作被延迟到 epoch 外部。第14章在后面的"附加主题"部分以介绍级别覆盖 epoch。

所有这些上下文的共同线索是，周围环境的某些东西禁止线程上下文操作。"某些东西"各不相同（自旋锁、过滤器上下文、epoch 段、软件中断调度），但补救措施相同：入队一个任务，由不在受限上下文中的线程稍后运行。

### 实际延迟工作的原因

简要浏览驱动从受限上下文中推出的工作类型。现在识别这些形态会为第6节将要展开的模式提供词汇。

**非平凡的 `selwakeup(9)`。** `selwakeup` 是内核的通知所有 select/poll 等待者的调用。传统做法是在没有驱动互斥锁持有的情况下调用它，并且绝不在持有自旋锁的上下文中调用。callout 回调持有互斥锁；中断过滤器什么都没持有，但本身就处于不好的位置。想要从这些上下文通知 poller 的驱动通常入队一个任务，其唯一工作就是调用 `selwakeup`。

**硬件事件后的 `copyin` 和 `copyout`。** 中断发出 DMA 传输完成信号后，驱动可能想要将数据复制到或来自先前通过 ioctl 注册的用户空间缓冲区地址。`copyin` 和 `copyout` 在中断上下文中都不合法。驱动调度一个任务，其回调在进程上下文中执行复制。

**需要可睡眠锁的重新配置。** 驱动的配置通常由 `sx(9)` 锁保护，它可以睡眠。callout 或中断不能直接获取可睡眠锁。如果定时器驱动的决策意味着配置更改，定时器入队一个任务；任务获取 sx 锁并执行更改。

**退避后重试失败操作。** 硬件操作有时会瞬态失败。合理的响应是等待某个间隔然后重试。中断处理程序不能阻塞；它入队一个 `timeout_task`，延迟等于退避间隔。超时任务稍后在线程上下文中触发，重试操作，如果再次失败则用更长的延迟重新调度自己。

**记录非平凡事件。** 内核 `printf(9)` 对奇怪上下文出奇地容忍，但 `log(9)` 和其伙伴不是。想要从中断上下文发出多行诊断的驱动在处理程序中只写最少的内容（一个标志、一个计数器递增）并调度一个任务稍后做真正的日志。

**排空或重新配置长硬件队列。** 检测到队头阻塞的网络驱动可能想要遍历其发送环，释放已完成的描述符，并重置每描述符状态。工作有界但不平凡。在中断路径中内联做这件事会在不好的上下文中独占 CPU。在任务中做它允许中断立即返回，真正的工作在线程上发生。

**延迟拆除。** 当驱动在某个对象仍有未完成引用时分离，驱动不能立即释放该对象。一个常见模式：将释放延迟到一个任务，该任务在引用计数已知为零后，或在足够长的宽限期内任何进行中的引用都已排空后触发。

所有这些情况共享相同结构：受限上下文检测到需求，可能记录少量状态，然后入队一个任务。任务稍后在线程上下文中运行，做真正的工作，并可选地重新入队自己或调度后续操作。

### 轮询与延迟执行

此时一个合理的问题是：如果受限上下文不能做工作，为什么不安排工作在受限上下文之外的某个地方发生？为什么不拥有一个专用内核线程轮询"有什么事要做"，并在看到需要行动的状态时唤醒？

这实际上就是任务队列做的事。任务队列线程在有工作可用前一直睡眠，被唤醒后处理它。"任务队列"与"手工制作的轮询线程"之间的区别在于，任务队列框架为你解决了周围的后勤问题。入队是单个原子操作。任务结构直接持有回调和上下文，所以你不必设计"工作队列条目"类型。冗余入队的合并是自动的。排空任务是单个调用。拆除是单个调用。通过标准工具的可观察性免费获得。

手工制作的轮询线程可以做同样的工作，在极端情况下它是正确的选择（例如，工作有硬实时约束，或者它是需要专用优先级的子系统的一部分）。对于普通的驱动工作，越过 `taskqueue(9)` 几乎总是一个错误。

一个独立但相关的问题：为什么不每个延迟操作都启动一个新的内核线程？这极其昂贵：创建线程需要时间，分配完整的内核栈，并将新线程交给调度器。对于重复发生的工作，合理的设计是重用线程，这正是任务队列提供的。对于只发生一次的工作，你可以用 `kproc_create(9)` 并让新线程完成后退出，但即使那样，带有 `taskqueue_drain` 的任务队列通常更简单且成本相近。

### FreeBSD 的解决方案

内核为延迟工作提供了一小组设施。第14章专注于其中之一（`taskqueue(9)`），并在驱动编写者需要了解何时使用其他设施的适当细节水平上提及它们。现在做一个简要概览；后续各节在相关时展开每个设施。

**`taskqueue(9)`。** 一个由 `struct task` 条目组成的队列和一个或多个消费该队列的内核线程（或软件中断上下文）。延迟驱动工作的主导选择。本章深入覆盖。

**`epoch(9)`。** 一种无锁读同步机制，网络驱动用于允许读者在无锁情况下遍历共享数据结构。写入者通过 `epoch_call` 或 `epoch_wait` 延迟清理。不是通用驱动的通用延迟工作机制，但足够重要，本章稍后介绍，以便你在网络驱动代码中看到时能认出它。

**组任务队列。** 任务队列的可扩展变体，其中一组相关任务共享每 CPU 工作线程池。网络驱动大量使用；大多数其他驱动不用。本章稍后介绍。

**`kproc_create(9)` / `kthread_add(9)`。** 直接创建内核线程。当延迟工作是不适合"短任务"形态的长运行循环，以及工作值得拥有专用优先级或 CPU 亲和性时有用。对于简单延迟几乎总是过度杀伤；任务队列是首选。

**通过 `swi_add(9)` 的专用 SWI（软件中断）处理程序。** 一种注册在软件中断上下文中运行的函数的方式。系统任务队列（`taskqueue_swi`、`taskqueue_swi_giant`、`taskqueue_fast`）构建在此机制之上。驱动代码很少直接调用 `swi_add`；任务队列层是正确的抽象。

**callout 本身，重新调度为"从现在起零秒后"。** 一种行不通的模式：你不能通过调度另一个 callout 来"逃离" callout 上下文，因为下一个 callout 仍然在 callout 上下文中运行。认识到这是一个死胡同本身是有用的。callout 调度时刻；任务队列提供上下文。

在第14章的其余部分，除非我们另有说明，"延迟到任务"或"入队任务"意味着"将 `struct task` 入队到 `struct taskqueue`"。

### 何时延迟是错误答案

延迟是一种工具，不是默认。有几种情况从原地做工作中受益而不是延迟。

**工作确实很短且对当前上下文安全。** 从 callout 中用 `device_printf(9)` 记录一行统计数据是没问题的。递增计数器也是。通知 cv 也是。将这些琐碎操作延迟到任务比直接做成本更高。只在工作确实不属于当前上下文时才延迟。

**时序重要且延迟引入方差。** 任务不会立即运行。它在任务队列线程下次被调度时运行，根据系统负载可能是微秒或毫秒之后。如果工作有严格的时序要求（例如在截止时间内确认硬件事件），延迟可能错过截止时间。对于这类工作，你需要更快的机制（硬件级完成、`C_DIRECT_EXEC` callout 或 SWI）或不同的设计。

**延迟会无益地增加一跳。** 如果中断处理程序的唯一工作已经在中断上下文中安全完成，添加任务往返会使延迟加倍而不改善任何东西。只延迟工作中需要延迟的部分。

**工作需要特定线程。** 如果工作需要作为特定用户进程运行（例如，使用该进程的文件描述符表），通用任务队列线程是错误的地方。这种情况在驱动中很少见，但它存在。

对于其他所有情况，通过任务队列延迟是正确答案，本章的其余部分是关于如何做好它。

### 一个实战示例：为什么 Tick 源不能唤醒 Poller

来自第13章驱动的一个具体例子，值得放慢速度来看，因为这是第14章做出真正更改的第一个地方。

第13章的 `tick_source` 回调，来自 `stage4-final/myfirst.c`，看起来是这样的：

```c
static void
myfirst_tick_source(void *arg)
{
        struct myfirst_softc *sc = arg;
        size_t put;
        int interval;

        MYFIRST_ASSERT(sc);
        if (!sc->is_attached)
                return;

        if (cbuf_free(&sc->cb) > 0) {
                put = cbuf_write(&sc->cb, &sc->tick_source_byte, 1);
                if (put > 0) {
                        counter_u64_add(sc->bytes_written, put);
                        cv_signal(&sc->data_cv);
                        /* selwakeup omitted: cannot be called from a
                         * callout callback while sc->mtx is held. */
                }
        }
        ...
}
```

底部那个注释不是假设的。`selwakeup(9)` 进入每 selinfo 的互斥锁并可能调用 kqueue 子系统，这在持有不同驱动互斥锁的 callout 回调中是不安全的。因此，等待 `/dev/myfirst` 可读性的 `select(2)`/`poll(2)` 用户程序在 tick 源存入字节时不会收到通知。程序只在其他路径调用 `selwakeup` 时才醒来，例如当另一个线程的 `write(2)` 到达时。

这是第13章驱动中的一个真实 bug。我们在第13章中没有修复它，因为修复它需要一个我们尚未引入的原语。第14章引入该原语并修复了这个 bug。

修复很小。在 softc 中添加一个 `struct task`。在 attach 中初始化它。不是从 tick_source 回调中省略 `selwakeup`，而是入队该任务；任务在线程上下文中运行，没有驱动互斥锁持有，安全地调用 `selwakeup`。在分离时，在 `is_attached` 被清除后、释放 selinfo 之前排空该任务。

我们将在第3节中逐步完成该更改的每一步。现在的要点是更改是机械的，其必要性不是虚构的。第14章的第一个真正工作就是给你这种 bug 所需的工具。

### 一个小型心智模型

一个有用的图景，提供一次，后面会引用。

把你的驱动想象为由两种代码组成。第一种是因为有人请求而运行的代码：`read(2)` 处理程序、`write(2)` 处理程序、`ioctl(2)` 处理程序、sysctl 处理程序、open 和 close 处理程序。这些代码在线程上下文中运行，遵守普通规则，可以睡眠、分配内存和触碰任何锁。称之为"线程上下文代码"。

第二种是因为时间或硬件说了什么而运行的代码：callout 回调、中断过滤器、epoch 保护的读。这些代码在受限上下文中运行，规则更窄，必须保持工作短小且不可睡眠。称之为"边缘上下文代码"。

大多数真实工作属于线程上下文代码。边缘上下文代码实际需要做的大多是：注意到边缘，记录少量状态，然后将工作交给线程上下文代码。任务队列就是那个交接。任务回调在线程上下文代码中运行，因为任务队列的线程处于线程上下文。回调做的一切遵循普通规则。

这个心智模型让你可以将每个后续节阅读为单一思想的变体：边缘上下文代码检测，线程上下文代码行动，任务队列是两者之间的接缝。一旦你这样看待驱动，本章的其余部分就是工程细节。

### 第1节总结

有些工作必须在受限上下文（callout、中断过滤器、epoch 段）中运行。这些上下文的规则禁止睡眠、大量分配、可睡眠锁获取和几种其他常见操作。有真实职责的驱动频繁需要做恰恰这些操作来响应在受限上下文中到达的事件。补救措施是入队一个任务，让工作线程在规则允许的上下文中做真正的工作。

内核将这种模式暴露为 `taskqueue(9)`。API 很小，习语是规则的，该工具与你已经知道的 callout 和同步原语干净地组合。第2节介绍这个原语。



## 第2节：`taskqueue(9)` 简介

`taskqueue(9)` 与内核大多数成熟的子系统一样，是一个精心实现之上的小型 API。数据结构很短，生命周期是规则的（初始化、入队、运行、排空、释放），规则足够明确，你可以通过阅读源码来验证用法。本节逐步介绍结构，命名 API，列出内核免费提供的预定义队列，并将任务队列与第13章的 callout 进行比较，以便你看到各自何时是正确的工具。

### 任务结构

该数据结构位于 `/usr/src/sys/sys/_task.h`：

```c
typedef void task_fn_t(void *context, int pending);

struct task {
        STAILQ_ENTRY(task) ta_link;     /* (q) link for queue */
        uint16_t ta_pending;            /* (q) count times queued */
        uint8_t  ta_priority;           /* (c) Priority */
        uint8_t  ta_flags;              /* (c) Flags */
        task_fn_t *ta_func;             /* (c) task handler */
        void    *ta_context;            /* (c) argument for handler */
};
```

字段分为两组。`(q)` 字段由任务队列在其自己的内部锁下管理；驱动代码不直接触碰它们。`(c)` 字段在初始化后为常量；驱动代码通过初始化器设置一次，之后不再修改。

`ta_link` 是任务入队时使用的列表链接。任务空闲时未使用。

`ta_pending` 是合计计数器。任务第一次入队时从零变为一，任务被放入列表。如果在回调运行前再次入队，计数器简单递增，任务仍在列表中只出现一次。当回调最终运行时，最终的待处理计数作为第二个参数传递给回调，计数器重置为零。关于 `ta_pending` 你能犯的最大错误是假设任务入队 N 次就会运行 N 次；它不会。它将运行一次，回调将知道它被入队了 N 次。第5节详细讨论设计含义。

`ta_priority` 在单个队列内排序任务。更高优先级的任务在更低优先级的任务之前运行。对于大多数驱动，值为零（普通优先级），队列实际上是 FIFO。

`ta_flags` 是一个小位域。内核用它来记录任务当前是否已入队，以及对于网络任务，任务是否应该在网络 epoch 内运行。驱动代码在 `TASK_INIT` 或 `NET_TASK_INIT` 设置后不触碰它。

`ta_func` 是回调函数。其签名为 `void (*)(void *context, int pending)`。第一个参数是你在初始化时存储在 `ta_context` 中的任何内容；第二个是合计计数。

`ta_context` 是回调的参数。对于设备驱动任务，这几乎总是 softc 指针。

该结构在 amd64 上为 32 字节，加减填充。你为每个延迟工作模式在 softc 中嵌入一个。有三个延迟路径的驱动有三个 `struct task` 成员。

### 初始化任务

标准的初始化宏是 `TASK_INIT`，位于 `/usr/src/sys/sys/taskqueue.h`：

```c
#define TASK_INIT_FLAGS(task, priority, func, context, flags) do {      \
        (task)->ta_pending = 0;                                         \
        (task)->ta_priority = (priority);                               \
        (task)->ta_flags = (flags);                                     \
        (task)->ta_func = (func);                                       \
        (task)->ta_context = (context);                                 \
} while (0)

#define TASK_INIT(t, p, f, c)    TASK_INIT_FLAGS(t, p, f, c, 0)
```

从驱动的 attach 例程中调用通常如下：

```c
TASK_INIT(&sc->selwake_task, 0, myfirst_selwake_task, sc);
```

参数读作："初始化此任务，普通优先级零，在触发时运行 `myfirst_selwake_task(sc, pending)`"。这就是整个初始化仪式。没有对应的"销毁"调用；任务在回调完成后变为空闲，在周围 softc 释放时超出作用域。

对于网络路径任务有一个变体 `NET_TASK_INIT`，它设置 `TASK_NETWORK` 标志，使任务队列知道在 `net_epoch_preempt` epoch 内运行回调：

```c
#define NET_TASK_INIT(t, p, f, c) TASK_INIT_FLAGS(t, p, f, c, TASK_NETWORK)
```

除非你在编写网络驱动，否则 `TASK_INIT` 就是你使用的。第14章全程使用 `TASK_INIT`，只在"附加主题"部分回到 `NET_TASK_INIT`。

### 从驱动的角度看任务队列结构

从驱动的角度看，任务队列是一个 `struct taskqueue *`。该指针要么是一个预定义的全局指针（`taskqueue_thread`、`taskqueue_swi`、`taskqueue_bus` 等），要么是驱动用 `taskqueue_create` 创建并存储在其 softc 中的。两种情况下指针都是不透明的。所有交互都通过 API 调用进行。我们在本章中唯一关心的内部细节是任务队列持有自己的锁，它在入队和工作线程从列表中拉取任务时获取该锁。

为了完整性，定义（来自 `/usr/src/sys/kern/subr_taskqueue.c`）：

```c
struct taskqueue {
        STAILQ_HEAD(, task)     tq_queue;
        LIST_HEAD(, taskqueue_busy) tq_active;
        struct task            *tq_hint;
        u_int                   tq_seq;
        int                     tq_callouts;
        struct mtx_padalign     tq_mutex;
        taskqueue_enqueue_fn    tq_enqueue;
        void                   *tq_context;
        char                   *tq_name;
        struct thread         **tq_threads;
        int                     tq_tcount;
        int                     tq_spin;
        int                     tq_flags;
        ...
};
```

`tq_queue` 是待处理任务列表。`tq_active` 记录当前正在运行的任务，排空逻辑用它来等待完成。`tq_mutex` 是任务队列自己的锁。`tq_threads` 是工作线程数组，大小为 `tq_tcount`。`tq_spin` 记录互斥锁是自旋互斥锁（用于通过 `taskqueue_create_fast` 创建的任务队列）还是睡眠互斥锁（用于通过 `taskqueue_create` 创建的任务队列）。`tq_flags` 记录关闭状态。

你不从驱动代码中触碰任何这些字段。这里展示一次是为了让本节其余部分的 API 调用有一个具体的参照物。本章的其余部分将任务队列视为不透明的。

### API逐一介绍

公共函数声明在 `/usr/src/sys/sys/taskqueue.h` 中。驱动通常使用不到一打。我们现在按用途分组逐步介绍重要的函数。

**创建和销毁任务队列。**

```c
struct taskqueue *taskqueue_create(const char *name, int mflags,
    taskqueue_enqueue_fn enqueue, void *context);

struct taskqueue *taskqueue_create_fast(const char *name, int mflags,
    taskqueue_enqueue_fn enqueue, void *context);

int taskqueue_start_threads(struct taskqueue **tqp, int count, int pri,
    const char *name, ...);

void taskqueue_free(struct taskqueue *queue);
```

`taskqueue_create` 创建一个内部使用睡眠互斥锁的任务队列。在上面入队的任务运行在睡眠合法的上下文中（假设它们通过 `taskqueue_thread_enqueue` 和 `taskqueue_start_threads` 调度）。这是几乎所有驱动任务队列的正确选择。

`taskqueue_create_fast` 创建一个内部使用自旋互斥锁的任务队列。只有在你打算从睡眠互斥锁会出错的上下文入队时才需要（例如，从自旋互斥锁内部或过滤器中断中）。驱动代码很少需要这个；预定义的 `taskqueue_fast` 存在于需要的情况。

`enqueue` 回调在任务被添加到原本为空的队列时由任务队列层调用，是层"唤醒"消费者的方式。对于由内核线程服务的队列，入队函数是 `taskqueue_thread_enqueue`，由内核提供。对于由软件中断服务的队列，内核提供 `taskqueue_swi_enqueue`。驱动代码几乎总是传递 `taskqueue_thread_enqueue`。

`context` 参数被传回给入队回调。使用 `taskqueue_thread_enqueue` 时，约定是传递 `&your_taskqueue_pointer`，以便函数能找到它要唤醒的任务队列。第14章的示例字面遵循此约定。

`taskqueue_start_threads` 创建 `count` 个运行 `taskqueue_thread_loop` 调度器的内核线程，每个在队列上睡眠直到任务到达。`pri` 参数是线程的优先级。`PWAIT`（定义在 `/usr/src/sys/sys/priority.h`，数值为 76）是驱动任务队列的普通选择；网络驱动通常传递 `PI_NET`（数值为 4）以在中断邻近优先级运行。第14章的工作线程使用 `PWAIT`。

`taskqueue_free` 关闭任务队列。它排空所有待处理和运行中的任务，终止工作线程，并释放内部状态。它必须在没有未排空的待处理任务时调用；它返回后，`struct taskqueue *` 无效且不得使用。

**初始化任务。** 如上所示的 `TASK_INIT`。没有对应的"销毁"，因为任务结构由调用者拥有。

**入队任务。**

```c
int taskqueue_enqueue(struct taskqueue *queue, struct task *task);
int taskqueue_enqueue_flags(struct taskqueue *queue, struct task *task,
    int flags);
int taskqueue_enqueue_timeout(struct taskqueue *queue,
    struct timeout_task *timeout_task, int ticks);
int taskqueue_enqueue_timeout_sbt(struct taskqueue *queue,
    struct timeout_task *timeout_task, sbintime_t sbt, sbintime_t pr,
    int flags);
```

`taskqueue_enqueue` 是主力。它将任务链接到队列并唤醒工作线程。如果任务已经待处理，它递增 `ta_pending` 并返回。成功返回零；很少失败。

`taskqueue_enqueue_flags` 相同，带有可选标志：

- `TASKQUEUE_FAIL_IF_PENDING` 使入队在任务已待处理时返回 `EEXIST` 而不是合并。
- `TASKQUEUE_FAIL_IF_CANCELING` 使入队在任务当前正在取消时返回 `EAGAIN`。

默认的 `taskqueue_enqueue` 静默合并；标志变体让你在重要时检测这种情况。

`taskqueue_enqueue_timeout` 调度一个 `struct timeout_task` 在给定的 tick 数后触发。在幕后它使用一个内部 `callout`，其回调在延迟到期时将底层任务入队到任务队列。`sbt` 变体接受 sbintime 用于亚 tick 精度。

**取消任务。**

```c
int taskqueue_cancel(struct taskqueue *queue, struct task *task,
    u_int *pendp);
int taskqueue_cancel_timeout(struct taskqueue *queue,
    struct timeout_task *timeout_task, u_int *pendp);
```

`taskqueue_cancel` 如果任务尚未开始运行则将其从队列中移除，如果指针非 NULL 则将之前的待处理计数写入 `*pendp`。如果任务当前正在运行，函数返回 `EBUSY` 并且不等待；如果需要等待，你必须跟进 `taskqueue_drain`。

`taskqueue_cancel_timeout` 对超时任务相同。

**排空任务。**

```c
void taskqueue_drain(struct taskqueue *queue, struct task *task);
void taskqueue_drain_timeout(struct taskqueue *queue,
    struct timeout_task *timeout_task);
void taskqueue_drain_all(struct taskqueue *queue);
```

`taskqueue_drain(tq, task)` 阻塞直到给定任务不再待处理且不再运行。如果任务待处理，排空等待它运行并完成。如果任务正在运行，排空等待当前调用返回。如果任务空闲，排空立即返回。这是你在分离时为驱动拥有的每个任务使用的调用。

`taskqueue_drain_timeout` 对超时任务相同。

`taskqueue_drain_all` 排空任务队列中的每个任务和每个超时任务。当你拥有私有任务队列并想在释放它之前确保它完全安静时有用。`taskqueue_free` 本身在内部做等效工作，所以 `taskqueue_drain_all` 在 `taskqueue_free` 之前不是严格必需的，但当你想在不确定是否销毁任务队列的情况下静默它时很有用。

**阻塞和解阻塞。**

```c
void taskqueue_block(struct taskqueue *queue);
void taskqueue_unblock(struct taskqueue *queue);
void taskqueue_quiesce(struct taskqueue *queue);
```

`taskqueue_block` 停止队列运行新任务。已运行的任务完成；新入队的任务累积但直到调用 `taskqueue_unblock` 才运行。这对在精细过渡期间临时冻结队列而不拆除它很有用。

`taskqueue_quiesce` 等待当前运行的任务（如果有）完成并等待队列中没有待处理任务。等同于"排空一切但不销毁"。在队列运行时安全调用。

**成员检查。**

```c
int taskqueue_member(struct taskqueue *queue, struct thread *td);
```

如果给定线程是任务队列的工作线程之一则返回 true。在任务回调内部当你想要根据"我是否运行在自己的任务队列上"来分支时有用，尽管更常见的习语是用 `curthread` 与存储的线程指针比较。

这就是驱动通常使用的全部 API。还有一些不太常用的函数（`taskqueue_set_callback` 用于初始化/关闭钩子，`taskqueue_poll_is_busy` 用于轮询式检查），但大多数驱动从不触碰它们。

### 预定义任务队列

内核为不需要私有任务队列的驱动提供了一小组预配置任务队列。它们在 `/usr/src/sys/sys/taskqueue.h` 中用 `TASKQUEUE_DECLARE` 声明，展开为一个 extern 指针。驱动通过名称使用它们：

```c
TASKQUEUE_DECLARE(thread);
TASKQUEUE_DECLARE(swi);
TASKQUEUE_DECLARE(swi_giant);
TASKQUEUE_DECLARE(fast);
TASKQUEUE_DECLARE(bus);
```

**`taskqueue_thread`** 是通用线程上下文队列。一个内核线程，优先级 `PWAIT`。线程名在 `ps` 中显示为 `thread taskq`。对任何想要完整线程上下文且不需要特殊属性的任务安全。最容易使用的预定义队列；如果你不确定需要哪个队列，这是一个非常合理的第一选择。

**`taskqueue_swi`** 由软件中断处理程序调度，不是内核线程。此队列上的任务运行时没有驱动互斥锁持有但在 SWI 上下文中，仍有限制（不能睡眠）。适用于想要在入队后快速运行的非睡眠短工作，没有唤醒内核线程的调度延迟。驱动使用不常见。

**`taskqueue_swi_giant`** 与 `taskqueue_swi` 相同但运行时持有历史遗留的 `Giant` 锁。新代码中基本不使用。仅为完整性提及。

**`taskqueue_fast`** 是自旋互斥锁支持的软件中断队列，用于必须从睡眠互斥锁会出错的上下文入队的任务（例如，从另一个自旋互斥锁内部）。任务队列本身使用自旋互斥锁作为其内部列表，因此入队在任何上下文中都合法。但任务回调运行在 SWI 上下文中，仍有禁止睡眠的限制。驱动使用罕见；需要入队工作的过滤器中断上下文通常使用 `taskqueue_fast` 或更常见地使用私有 `taskqueue_create_fast` 队列。

**`taskqueue_bus`** 是 `newbus(9)` 设备事件（热插拔插入、移除、子总线通知）的专用队列。普通驱动不在该队列上入队。

对于像 `myfirst` 这样的驱动，现实的选择是 `taskqueue_thread`（共享队列）或你在分离时拥有并拆除的私有任务队列。第4节讨论权衡；重构的第1阶段为简单起见使用 `taskqueue_thread`，第2阶段转移到私有队列。

### 任务队列与定时调用的比较

一个简短的并排比较，因为新读者最先问这个问题。

| Property | `callout(9)` | `taskqueue(9)` |
|---|---|---|
| Fires at | A specific time | As soon as a worker thread picks it up |
| Callback context | Callout thread (default) or hardclock IRQ (`C_DIRECT_EXEC`) | Kernel thread (for `taskqueue_thread`, private queues) or SWI (for `taskqueue_swi`, `taskqueue_fast`) |
| May sleep | No | Yes, for thread-backed queues; no, for SWI-backed queues |
| May acquire sleepable locks | No | Yes, for thread-backed queues |
| May call `uiomove`, `copyin`, `copyout` | No | Yes, for thread-backed queues |
| Coalesces redundant submissions | No, each reset replaces the previous deadline | Yes, `ta_pending` increments |
| Cancellable before firing | `callout_stop(co)` | `taskqueue_cancel(tq, task, &pendp)` |
| Waits for in-flight callback | `callout_drain(co)` | `taskqueue_drain(tq, task)` |
| Periodic | Callback reschedules itself | No; enqueue again from somewhere else, or use a callout to enqueue |
| Scheduled for the future | `callout_reset(co, ticks, ...)` | `taskqueue_enqueue_timeout(tq, tt, ticks)` |
| Cost per firing | Microseconds | Microseconds plus thread wake (can be larger under load) |

该表格说明了分工。当你需要在特定时刻触发且工作对 callout 上下文安全时，callout 是正确的原语。当你需要线程上下文工作并愿意接受任务队列引入的任何调度延迟时，任务队列是正确的原语。许多驱动两者一起使用：callout 在截止时间触发，callout 入队一个任务，任务在线程上下文中做真正的工作。

### 任务队列与私有内核线程的比较

本章欠你的另一个比较，因为问"为什么不直接创建一个内核线程"的读者值得一个直接的回答。

用 `kproc_create(9)` 创建的内核线程是一个完整的调度实体：自己的栈（amd64 上通常 16 KB）、自己的优先级、自己的 `proc` 条目、自己的状态。想要运行"每秒做 X"循环的驱动可以创建这样的线程并用 `kproc_kthread_add` 加 `cv_timedwait` 来循环。代码可以工作，但成本超过工作通常应得的。一个有一个大部分时间空闲并在入队时唤醒的线程的任务队列，每个待处理工作项成本更低，也更容易拆除。

`kproc_create` 有合理的情况。一个有自己的调优（优先级、CPU 亲和性、进程组）的长运行子系统是一种。一个确实需要自己的线程以获得可观察性的周期性工作是另一种。驱动的延迟工作模式几乎从来不是。使用任务队列直到特定需求迫使你做其他事情。

### 入队已待处理规则

一个值得提前指出的规则，因为它是 API 新手最常感到惊讶的来源：任务不能被待处理两次。如果你在 `sc->t` 已经待处理时调用 `taskqueue_enqueue(tq, &sc->t)`，内核递增 `sc->t.ta_pending` 并返回成功，不会第二次链接任务。

这有两个含义。首先，你的回调将运行一次，不是两次，即使你入队了两次。其次，回调接收的 `pending` 参数是回调被调度前任务被入队的次数；你的回调可以使用该计数来批量处理累积的工作。

如果你想让回调对 N 次入队运行 N 次，单个任务是错误的模型。使用 N 个独立任务，或向驱动拥有的队列中入队一个哨兵并在回调中处理每个哨兵。几乎总是合并行为是你想要的；第5节逐步讲解如何有意利用它。

### 端到端最小示例

一个 hello-world 任务，为了具体。如果你在一个临时模块中输入并加载它，你将在 `dmesg` 中看到 `device_printf` 行：

```c
#include <sys/param.h>
#include <sys/kernel.h>
#include <sys/module.h>
#include <sys/systm.h>
#include <sys/taskqueue.h>

static struct task example_task;

static void
example_task_fn(void *context, int pending)
{
        printf("example_task_fn: pending=%d\n", pending);
}

static int
example_modevent(module_t m, int event, void *arg)
{
        int error = 0;

        switch (event) {
        case MOD_LOAD:
                TASK_INIT(&example_task, 0, example_task_fn, NULL);
                taskqueue_enqueue(taskqueue_thread, &example_task);
                break;
        case MOD_UNLOAD:
                taskqueue_drain(taskqueue_thread, &example_task);
                break;
        default:
                error = EOPNOTSUPP;
                break;
        }
        return (error);
}

static moduledata_t example_mod = {
        "example_task", example_modevent, NULL
};
DECLARE_MODULE(example_task, example_mod, SI_SUB_DRIVERS, SI_ORDER_MIDDLE);
MODULE_VERSION(example_task, 1);
```

该模块做了五件事，每件一行。加载时，`TASK_INIT` 准备任务结构。`taskqueue_enqueue` 请求共享的 `taskqueue_thread` 运行回调。回调打印一条消息。卸载时，`taskqueue_drain` 如果回调尚未完成则等待它完成。整个生命周期很紧凑。

如果你输入并加载它，`dmesg` 显示：

```text
example_task_fn: pending=1
```

`pending=1` 反映了任务在回调触发前被入队了一次。

现在尝试一个合并演示：将 `MOD_LOAD` 改为连续入队任务五次，然后添加一个短暂暂停让任务队列线程有机会唤醒：

```c
for (int i = 0; i < 5; i++)
        taskqueue_enqueue(taskqueue_thread, &example_task);
pause("example", hz / 10);
```

再次运行，`dmesg` 显示：

```text
example_task_fn: pending=5
```

一次调用，待处理为五。这就是合并规则的实际效果。

这足以让下一节中实战重构变得有意义。本章的其余部分将相同结构扩展到真正的 `myfirst` 驱动，将临时模块替换为四个集成阶段，添加拆除、添加私有任务队列，并逐步讲解调试故事。

### 第2节总结

`struct task` 持有回调和其上下文。`struct taskqueue` 管理此类任务的队列和一个或多个消费它们的线程（或 SWI 上下文）。API 很小：创建、启动线程、入队（可选带延迟）、取消、排空、释放、阻塞、解阻塞、静默。内核提供了少数预定义队列，每个驱动都可以使用而不必创建自己的。入队已待处理规则将冗余提交合并为单次调用，其待处理计数是最终总计。

第3节将这些工具带到 `myfirst` 驱动中，并在一直在静默跳过 `selwakeup` 的 `tick_source` callout 下放入第一个任务。修复很小；心智模型是重要的部分。



## 第3节：从定时器或模拟中断延迟工作

第13章留给 `myfirst` 驱动三个都严格遵守 callout 契约的 callout。它们都没有尝试做不属于 callout 回调的事情。特别是 `tick_source` 回调省略了真实驱动在新字节出现在缓冲区时想要调用的 `selwakeup`，文件中甚至带有一条注释说明了这一点。第14章移除了这个省略。

第3节是第一次实战重构。它引入第14章驱动的第1阶段：驱动获得一个 `struct task`、一个任务回调、一个从 `tick_source` 发出的入队，以及在分离时的排空。私有任务队列的工作留到第4节；第1阶段我们使用共享的 `taskqueue_thread`。先使用共享队列保持第一步小且将变更隔离到延迟工作模式本身。

### 一句话概述变更

当 `tick_source` 刚向循环缓冲区存入一个字节时，不再静默省略 `selwakeup`，而是入队一个任务，其回调在线程上下文中运行 `selwakeup`。

这就是全部更改。其他一切都是围绕的设置。

### softc添加

向 `struct myfirst_softc` 添加两个成员：

```c
struct task             selwake_task;
int                     selwake_pending_drops;
```

`selwake_task` 是我们将要入队的任务。`selwake_pending_drops` 是一个调试计数器，每当任务将两次或更多入队合并为一次触发时递增；"入队调用次数"与"回调调用次数"之间的差异告诉我们 tick 源产生数据的速度超过任务队列线程排空速度的频率。这纯粹是诊断性的；如果你愿意可以省略它，但看到实际合并计数在行动中是有价值的。

添加一个只读 sysctl 以便我们可以从用户空间观察计数器而无需调试构建：

```c
SYSCTL_ADD_INT(&sc->sysctl_ctx, SYSCTL_CHILDREN(sc->sysctl_tree),
    OID_AUTO, "selwake_pending_drops", CTLFLAG_RD,
    &sc->selwake_pending_drops, 0,
    "Times selwake_task coalesced two or more enqueues into one firing");
```

放置位置仅在它必须在 `sc->sysctl_tree` 创建之后、函数返回成功之前才有意义；第13章的 attach 序列已经有正确的结构，因此添加自然地与其他统计数据放在一起。

### 任务回调

添加一个函数：

```c
static void
myfirst_selwake_task(void *arg, int pending)
{
        struct myfirst_softc *sc = arg;

        if (pending > 1) {
                MYFIRST_LOCK(sc);
                sc->selwake_pending_drops++;
                MYFIRST_UNLOCK(sc);
        }

        /*
         * No driver mutex held. Safe to call selwakeup(9) here.
         */
        selwakeup(&sc->rsel);
}
```

有几件事值得注意。

回调通过 `arg` 指针获取 softc，与 callout 回调完全相同。它不需要在顶部使用 `MYFIRST_ASSERT`，因为任务回调运行时没有持有任何驱动锁；任务队列框架不会为你持有锁。这与第13章的 callout 锁感知模式不同，值得停下来思考。用 `callout_init_mtx(&co, &sc->mtx, 0)` 初始化的 callout 在 `sc->mtx` 持有的情况下运行。任务永远不会。在任务回调内部，如果你想触碰互斥锁保护的状态，你自行获取互斥锁，做工作，释放它，然后继续。

回调在互斥锁下有条件地更新 `selwake_pending_drops`。条件 `pending > 1` 意味着"此回调正在处理至少两个合并的入队"。在互斥锁下递增计数器是快速且安全的；无条件这样做会使常见情况（pending == 1，无合并）不必要地支付锁成本。

`selwakeup(&sc->rsel)` 调用本身就是我们在这里的原因。它在没有任何驱动锁持有的情况下运行，这正是 `selwakeup` 想要的，并且它在线程上下文中运行，这是 `selwakeup` 要求的。第13章的 bug 已修复。

回调不检查 `is_attached`。它不需要。分离路径在释放 selinfo 之前排空任务；当 `is_attached` 为零时，任务回调保证不在运行，`selwakeup` 将看到有效状态。排空顺序是使省略安全的原因，这就是为什么我们在第4节中如此仔细地讨论顺序。

### `tick_source` 的编辑

将 `tick_source` 回调从：

```c
static void
myfirst_tick_source(void *arg)
{
        struct myfirst_softc *sc = arg;
        size_t put;
        int interval;

        MYFIRST_ASSERT(sc);
        if (!sc->is_attached)
                return;

        if (cbuf_free(&sc->cb) > 0) {
                put = cbuf_write(&sc->cb, &sc->tick_source_byte, 1);
                if (put > 0) {
                        counter_u64_add(sc->bytes_written, put);
                        cv_signal(&sc->data_cv);
                        /* selwakeup omitted: cannot be called from a
                         * callout callback while sc->mtx is held. */
                }
        }

        interval = sc->tick_source_interval_ms;
        if (interval > 0)
                callout_reset(&sc->tick_source_co,
                    (interval * hz + 999) / 1000,
                    myfirst_tick_source, sc);
}
```

改为：

```c
static void
myfirst_tick_source(void *arg)
{
        struct myfirst_softc *sc = arg;
        size_t put;
        int interval;
        bool wake_sel = false;

        MYFIRST_ASSERT(sc);
        if (!sc->is_attached)
                return;

        if (cbuf_free(&sc->cb) > 0) {
                put = cbuf_write(&sc->cb, &sc->tick_source_byte, 1);
                if (put > 0) {
                        counter_u64_add(sc->bytes_written, put);
                        cv_signal(&sc->data_cv);
                        wake_sel = true;
                }
        }

        if (wake_sel)
                taskqueue_enqueue(taskqueue_thread, &sc->selwake_task);

        interval = sc->tick_source_interval_ms;
        if (interval > 0)
                callout_reset(&sc->tick_source_co,
                    (interval * hz + 999) / 1000,
                    myfirst_tick_source, sc);
}
```

两处编辑。一个局部 `wake_sel` 标志记录是否写入了字节；`taskqueue_enqueue` 调用发生在 cbuf 工作之后。关于"selwakeup 省略"的注释变得过时并被移除。

为什么使用标志而不是在 `if (put > 0)` 块中内联调用 `taskqueue_enqueue`？因为 `taskqueue_enqueue` 在持有 `sc->mtx` 时是安全的（它获取自己的内部互斥锁；任务队列自己的互斥锁与 `sc->mtx` 之间没有锁序问题），但保持互斥锁持有段紧凑并用局部变量命名入队原因是良好的卫生习惯。带标志的版本更易读，如果后续阶段添加更多应触发唤醒的条件也更容易扩展。

`taskqueue_enqueue` 在持有 `sc->mtx` 的 callout 回调中调用真的安全吗？是的。任务队列使用自己的内部互斥锁（`tq_mutex`），它完全独立于 `sc->mtx`；两者之间没有建立锁序，所以 `WITNESS` 没有什么可抱怨的。我们将在本节末尾的实验中验证这一点。供将来参考，`/usr/src/sys/kern/subr_taskqueue.c` 中的相关保证是 `taskqueue_enqueue` 获取 `TQ_LOCK(tq)`（对于 `taskqueue_create` 是睡眠互斥锁，对于 `taskqueue_create_fast` 是自旋互斥锁），执行列表操作，然后释放锁。不睡眠，不递归进入调用者的锁，无跨锁依赖。

### 附加变更

在 `myfirst_attach` 中，在现有 callout 初始化之后添加一行：

```c
TASK_INIT(&sc->selwake_task, 0, myfirst_selwake_task, sc);
```

把它放在 callout 初始化调用旁边。概念分组（"这里是我们准备驱动延迟工作原语的地方"）使文件更容易浏览。

在与其他计数器清零的同一代码块中将 `selwake_pending_drops` 初始化为零：

```c
sc->selwake_pending_drops = 0;
```

### 分离变更

这是阶段的关键部分。第13章的分离序列简化如下：

1. 如果 `active_fhs > 0` 则拒绝 detach。
2. 清除 `is_attached`。
3. 广播 `data_cv` 和 `room_cv`。
4. 通过 `seldrain` 排空 `rsel` 和 `wsel`。
5. 排空三个 callout。
6. 销毁设备、释放 sysctl、销毁 cbuf、释放计数器、销毁 cv、销毁 sx、销毁 mtx。

第14章第1阶段添加一步：在 callout 排空（步骤5）和 `seldrain` 调用（步骤4）之间排空 `selwake_task`。实际上，排序的微妙之处比这更仔细。让我们仔细想想。

`selwake_task` 回调调用 `selwakeup(&sc->rsel)`。如果 `sc->rsel` 正在被并发排空，回调可能产生竞争。规则是：确保在调用 `seldrain` 之前保证任务回调不在运行。这意味着 `taskqueue_drain(taskqueue_thread, &sc->selwake_task)` 必须在 `seldrain(&sc->rsel)` 之前发生。

然而，在我们排空 callout 之前，任务仍然可以被进行中的 callout 回调入队。如果我们先排空任务再排空 callout，进行中的 callout 可能在我们排空它之后重新入队任务，重新入队的任务将尝试在 `seldrain` 之后运行。

唯一安全的顺序是：先排空 callout（保证不会再有入队发生），然后排空任务（保证最后一次入队已完成），然后调用 `seldrain`。但我们也必须在排空 callout 之前清除 `is_attached`，以便进行中的回调提前退出而不是重新武装。

综合起来，第1阶段的分离顺序是：

1. 如果 `active_fhs > 0` 则拒绝 detach。
2. 清除 `is_attached`（在互斥锁保护下）。
3. 广播 `data_cv` 和 `room_cv`（先释放互斥锁）。
4. 排空三个 callout（不持有互斥锁；`callout_drain` 可能睡眠）。
5. 排空 `selwake_task`（不持有互斥锁；`taskqueue_drain` 可能睡眠）。
6. 通过 `seldrain` 排空 `rsel` 和 `wsel`。
7. 销毁设备、释放 sysctl、销毁 cbuf、释放计数器、销毁 cv、销毁 sx、销毁 mtx。

步骤4和5是新的排序约束。先 callout，第二任务，第三 sel。在调试内核上违反此顺序通常会触发 `seldrain` 内部的断言；在生产内核上这是一个等待发生的释放后使用。

`myfirst_detach` 中的代码变为：

```c
/* Chapter 13: drain every callout. No lock held; safe to sleep. */
MYFIRST_CO_DRAIN(&sc->heartbeat_co);
MYFIRST_CO_DRAIN(&sc->watchdog_co);
MYFIRST_CO_DRAIN(&sc->tick_source_co);

/* Chapter 14: drain every task. No lock held; safe to sleep. */
taskqueue_drain(taskqueue_thread, &sc->selwake_task);

seldrain(&sc->rsel);
seldrain(&sc->wsel);
```

两行代码加一条注释。排序在源码中可见。

### Makefile

无更改。`bsd.kmod.mk` 从系统树中获取任务队列 API 头文件；第1阶段不需要额外的源文件。

### 构建与加载

此时你的工作副本应该有：

- 两个新的 softc 成员（`selwake_task`、`selwake_pending_drops`）。
- `myfirst_selwake_task` 函数。
- 对 `myfirst_tick_source` 的编辑。
- attach 中的 `TASK_INIT` 调用和计数器清零。
- detach 中的 `taskqueue_drain` 调用。
- 新的 `selwake_pending_drops` sysctl。

从第1阶段目录构建：

```text
# cd /path/to/examples/part-03/ch14-taskqueues-and-deferred-work/stage1-first-task
# make clean && make
```

Load:

```text
# kldload ./myfirst.ko
```

Verify:

```text
# kldstat | grep myfirst
 7    1 0xffffffff82f30000    ... myfirst.ko
# sysctl dev.myfirst.0
dev.myfirst.0.stats.selwake_pending_drops: 0
...
```

### 观察修复效果

要观察第1阶段在做它的工作，在设备上启动一个 `poll(2)` 等待者并让 tick 源生成数据。一个简单的 poller 位于 `examples/part-03/ch14-taskqueues-and-deferred-work/labs/poll_waiter.c`：

```c
#include <stdio.h>
#include <unistd.h>
#include <fcntl.h>
#include <poll.h>
#include <err.h>

int
main(int argc, char **argv)
{
        int fd, n;
        struct pollfd pfd;
        char c;

        fd = open("/dev/myfirst", O_RDONLY);
        if (fd < 0)
                err(1, "open");
        pfd.fd = fd;
        pfd.events = POLLIN;

        for (;;) {
                n = poll(&pfd, 1, -1);
                if (n < 0) {
                        if (errno == EINTR)
                                continue;
                        err(1, "poll");
                }
                if (pfd.revents & POLLIN) {
                        n = read(fd, &c, 1);
                        if (n > 0)
                                write(STDOUT_FILENO, &c, 1);
                }
        }
}
```

用 `cc poll_waiter.c -o poll_waiter` 编译（无需特殊库）。在一个终端运行它：

```text
# ./poll_waiter
```

在第二个终端中，以较慢的节奏启用 tick 源使输出容易观察：

```text
# sysctl dev.myfirst.0.tick_source_interval_ms=500
```

没有第1阶段修复的第13章驱动，会让 `poll_waiter` 卡住。读取字节会累积在缓冲区中，但 `poll(2)` 永远不会返回，因为 `selwakeup` 从未被调用。你什么也看不到。

第1阶段驱动通过任务确实调用了 `selwakeup`。你应该看到 `t` 字符每半秒出现在 `poll_waiter` 终端中。当你停止测试时，`poll_waiter` 通过 `Ctrl-C` 干净退出。

现在加速 tick 源来给任务队列施加压力：

```text
# sysctl dev.myfirst.0.tick_source_interval_ms=1
```

你应该看到 `t` 字符的连续流。检查合计计数器：

```text
# sysctl dev.myfirst.0.stats.selwake_pending_drops
dev.myfirst.0.stats.selwake_pending_drops: <some number, growing slowly>
```

这个数字是任务回调处理了大于一的待处理计数的次数。在轻度负载的机器上它可能保持很小（任务队列线程足够快地醒来以单独处理每次入队）。在争用下数字会增长，你可以直接观察合并行为。

如果计数器即使在负载下也保持为零，说明机器足够快，每次入队在下一次到来之前就被排空了。这不是 bug；这表明合并存在但未触发。第5节引入一个故意的工作负载来强制合并。

### 卸载

停止 tick 源：

```text
# sysctl dev.myfirst.0.tick_source_interval_ms=0
```

用 `Ctrl-C` 关闭 `poll_waiter`。卸载：

```text
# kldunload myfirst
```

卸载应该干净。如果以 `EBUSY` 失败，你仍然在某处有打开的描述符；关闭它然后下次 `kldunload` 应该成功。

如果卸载挂起，分离路径中的某样东西被阻塞。最可能的原因是 `taskqueue_drain` 在等待一个无法完成的任务。那将表明有 bug，调试部分（第7节）展示了如何识别它。对于正常流程，卸载在毫秒内完成。

### 我们刚才做了什么

第4节扩展之前的简短总结。

第1阶段向驱动添加了一个任务，在 attach 中初始化它，从 callout 回调入队它，在分离时以正确顺序排空它，并观察了合并的实际效果。任务运行在共享的 `taskqueue_thread` 上；它与系统中其他也使用它的每个驱动共享该队列。对于低速率工作负载这完全没问题。对于最终会在任务中做大量工作的驱动，或想要将其任务处理延迟与系统正在做的其他事情隔离的驱动，私有任务队列是正确答案。第4节采取那一步。

### 需要避免的常见错误

初学者在编写第一个任务时犯的错误简表。每个都咬过真实的驱动；每个都有一个简单的规则来防止它。

**忘记在分离时排空。** 如果你入队任务但不排空它们，进行中的任务可能在 softc 被释放后运行，内核在任务回调中对已释放内存的解引用时崩溃。总是在释放任务触碰的任何东西之前排空驱动拥有的每个任务。

**相对于任务使用的状态以错误顺序排空。** 我们上面讨论的任务然后 sel 的顺序是一个特定情况。通用规则：排空每个入队的生产者，然后排空任务，然后释放任务使用的状态。违反顺序是竞争，即使竞争很少见。

**假设任务在入队后立即运行。** 它不会。任务队列线程在入队时被唤醒，然后由调度器决定何时运行它。在负载下这可能是毫秒级。假设零延迟的驱动在负载下会出问题。

**假设任务每次入队运行一次。** 它不会。合并折叠冗余提交。如果你需要"每个事件恰好一次"语义，你需要 softc 内部的每事件状态（例如工作项队列），而不是每个事件一个任务。

**在任务回调中以错误顺序获取驱动锁。** 任务回调是普通的线程上下文代码。它遵守驱动已建立的锁序。如果驱动的顺序是 `sc->mtx -> sc->cfg_sx`，任务回调必须先获取互斥锁再获取 sx。违反此顺序与在其他任何地方一样是 `WITNESS` 错误。

**从过滤器中断上下文内部在没有快速任务队列的情况下使用 `taskqueue_enqueue`。** `taskqueue_enqueue(taskqueue_thread, ...)` 获取任务队列内部锁上的睡眠互斥锁。这在过滤器中断上下文中是非法的。过滤器中断必须入队到 `taskqueue_fast` 或 `taskqueue_create_fast` 队列。callout 回调不触及此限制，因为它们在线程上下文中运行；这个问题是过滤器中断特有的。第四部分在引入 `bus_setup_intr` 时会重新讨论这一点。

这些错误中的每一个都可以通过审查、`WITNESS` 或精心编写的压力测试来捕获。前两个特别是那种在第一次负载下分离之前看起来都正常的 bug。

### 第3节总结

`myfirst` 驱动现在有一个任务。它使用该任务将 `selwakeup` 从 callout 回调中移到线程上下文，修复了第13章的一个真实 bug。任务在 attach 中初始化，从 `tick_source` 回调入队，并在分离时以相对于 callout 和 selinfo 排空的正确顺序排空。

共享的 `taskqueue_thread` 是我们使用的第一个任务队列，因为它已经存在。对于将要增长更多任务和更多职责的驱动，私有任务队列提供更好的隔离和更干净的拆除故事。第4节创建那个私有任务队列。



## 第4节：任务队列设置与清理

第1阶段使用了共享的 `taskqueue_thread`。那个选择保持了第一次更改的小型化：一个任务、一个入队、一个排空，以及一个需要遵守的分离顺序。第2阶段创建一个由驱动拥有的私有任务队列。代码量的变更很小，但它带来了几个在驱动成长后变得重要的属性。

本节教授重构的第2阶段，逐步讲解私有任务队列的设置和拆除，仔细审计分离顺序，最后以一个你可以在编写的每个任务队列驱动上重用的投产前检查表结束。

### 为什么使用私有任务队列

使用私有任务队列的三个原因。

第一，**隔离**。私有任务队列的线程只运行你驱动的任务。如果系统中其他驱动在 `taskqueue_thread` 上行为不当（例如，在任务回调中阻塞太久），你驱动的任务不受影响。反之，如果你的驱动行为不当，不当行为被限制在范围内。

第二，**可观察性**。`procstat -t` 和 `ps ax` 显示每个任务队列线程及其独特的名称。私有队列很容易发现：它以你给它的名称（按约定是 `myfirst taskq`）出现。共享的 `taskqueue_thread` 仅显示为 `thread taskq`，与所有其他驱动共享。

第三，**拆除是自包含的**。当你分离时，你排空并释放自己的任务队列。你不必推理其他驱动是否有一个你的排空可能等待的待处理任务。（你实际上不会在共享队列上等待其他驱动的任务，但"我们拥有自己的拆除"的心智模型更容易推理。）

成本很小。一个任务队列和一个内核线程，在 attach 时创建，在分离时拆除。几页内存和几个调度器条目。在任何现实系统上都不可衡量。

对于最终会有多个任务的驱动，私有任务队列是正确的默认选择。对于只有单个琐碎任务在罕见代码路径上的驱动，共享队列就可以了。`myfirst` 是前者：我们已有一个任务，本章还会添加更多。

### softc添加

向 `struct myfirst_softc` 添加一个成员：

```c
struct taskqueue       *tq;
```

第2阶段没有其他 softc 更改。

### 在 attach 中创建任务队列

在 `myfirst_attach` 中，在互斥锁/cv/sx 初始化和 callout 初始化之间，添加：

```c
sc->tq = taskqueue_create("myfirst taskq", M_WAITOK,
    taskqueue_thread_enqueue, &sc->tq);
if (sc->tq == NULL) {
        error = ENOMEM;
        goto fail_sx;
}
error = taskqueue_start_threads(&sc->tq, 1, PWAIT,
    "%s taskq", device_get_nameunit(dev));
if (error != 0)
        goto fail_tq;
```

该调用读作：创建一个名为 "myfirst taskq" 的任务队列，用 `M_WAITOK` 分配所以分配不会失败（我们在 attach 中，这是一个可睡眠上下文），使用 `taskqueue_thread_enqueue` 作为调度器以便队列由内核线程服务，并传递 `&sc->tq` 作为上下文以便调度器能找到队列。

名称 `"myfirst taskq"` 是在 `procstat -t` 中显示的可读标签。第14章示例中单队列驱动的约定是 `"<driver> taskq"`；有多个队列的驱动应使用更具体的名称如 `"myfirst rx taskq"` 和 `"myfirst tx taskq"`。

`taskqueue_start_threads` 创建工作线程。第一个参数是 `&sc->tq`，一个双重指针以便函数能找到任务队列。第二个参数是线程数量；`myfirst` 使用一个线程。有大量、可并行工作的驱动可能使用更多。第三个参数是优先级；`PWAIT` 是普通选择，等同于预定义 `taskqueue_thread` 使用的优先级。可变名称是每个线程名称的格式字符串；`device_get_nameunit(dev)` 给出每实例名称，以便多个 `myfirst` 实例有可区分的线程。

失败路径值得注意。如果 `taskqueue_create` 返回 NULL（用 `M_WAITOK` 通常不会，但要防御性处理），我们跳转到 `fail_sx`。如果 `taskqueue_start_threads` 失败，我们跳转到 `fail_tq`，它必须在继续其他清理之前调用 `taskqueue_free`。第14章第2阶段源码（见示例树）有正确顺序的标签。

### 更新入队调用点

每个 `taskqueue_enqueue(taskqueue_thread, ...)` 调用变为 `taskqueue_enqueue(sc->tq, ...)`。排空也一样：`taskqueue_drain(taskqueue_thread, ...)` 变为 `taskqueue_drain(sc->tq, ...)`。

第1阶段之后，驱动有两个这样的调用点：`myfirst_tick_source` 中的入队和 `myfirst_detach` 中的排空。两者都在一次搜索替换中更改。

### 拆除顺序

分离顺序增加了两行。第2阶段的完整顺序是：

1. 如果 `active_fhs > 0` 则拒绝分离。
2. 在互斥锁下清除 `is_attached`，广播两个 cv，释放互斥锁。
3. 排空三个 callout。
4. 在私有任务队列上排空 `selwake_task`。
5. 通过 `seldrain` 排空 `rsel` 和 `wsel`。
6. 用 `taskqueue_free` 释放私有任务队列。
7. 销毁设备，释放 sysctl，销毁 cbuf，释放计数器，销毁 cv，销毁 sx，销毁互斥锁。

新步骤是 4（在第1阶段已存在，现在指向 `sc->tq`）和 6（在第2阶段新增）。

一个自然的问题：如果步骤 6 会排空一切，我们还需要步骤 4 的显式 `taskqueue_drain` 吗？技术上不需要。`taskqueue_free` 在销毁队列之前排空所有待处理任务。但保留显式排空有两个好处。第一，它使顺序显式：你看到任务排空发生在 `seldrain` 之前，这是我们关心的顺序。第二，它将"等待此特定任务完成"的问题与"拆除整个队列"的问题分开。如果后续阶段在同一队列上添加更多任务，每个都有自己的显式排空，代码告诉读者正在发生什么。

`myfirst_detach` 中的相关代码：

```c
/* Chapter 13: drain every callout. No lock held; safe to sleep. */
MYFIRST_CO_DRAIN(&sc->heartbeat_co);
MYFIRST_CO_DRAIN(&sc->watchdog_co);
MYFIRST_CO_DRAIN(&sc->tick_source_co);

/* Chapter 14 Stage 1: drain every task. */
taskqueue_drain(sc->tq, &sc->selwake_task);

seldrain(&sc->rsel);
seldrain(&sc->wsel);

/* Chapter 14 Stage 2: destroy the private taskqueue. */
taskqueue_free(sc->tq);
sc->tq = NULL;
```

释放后将 `sc->tq` 设为 `NULL` 是防御性的：后来尝试使用已释放指针的 bug 会在调用点解引用 `NULL` 并崩溃，而不是损坏无关内存。它不花任何代价，偶尔能省下一个下午的调试。

### attach 失败路径

仔细遍历 attach 失败路径。第13章的 attach 有 cbuf 和互斥锁失败路径的标签。第2阶段添加任务队列相关标签：

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
        cv_init(&sc->data_cv, "myfirst data");
        cv_init(&sc->room_cv, "myfirst room");
        sx_init(&sc->cfg_sx, "myfirst cfg");

        sc->tq = taskqueue_create("myfirst taskq", M_WAITOK,
            taskqueue_thread_enqueue, &sc->tq);
        if (sc->tq == NULL) {
                error = ENOMEM;
                goto fail_sx;
        }
        error = taskqueue_start_threads(&sc->tq, 1, PWAIT,
            "%s taskq", device_get_nameunit(dev));
        if (error != 0)
                goto fail_tq;

        MYFIRST_CO_INIT(sc, &sc->heartbeat_co);
        MYFIRST_CO_INIT(sc, &sc->watchdog_co);
        MYFIRST_CO_INIT(sc, &sc->tick_source_co);

        TASK_INIT(&sc->selwake_task, 0, myfirst_selwake_task, sc);

        /* ... rest of attach as in Chapter 13 ... */

        return (0);

fail_cb:
        cbuf_destroy(&sc->cb);
fail_tq:
        taskqueue_free(sc->tq);
fail_sx:
        cv_destroy(&sc->data_cv);
        cv_destroy(&sc->room_cv);
        sx_destroy(&sc->cfg_sx);
        mtx_destroy(&sc->mtx);
        sc->is_attached = 0;
        return (error);
}
```

失败标签链式：`fail_cb` 调用 `cbuf_destroy` 然后落入 `fail_tq`，它调用 `taskqueue_free` 然后落入 `fail_sx`，后者销毁 cv、sx 和互斥锁。每个标签撤销到对应初始化调用成功点为止的所有内容。如果 `taskqueue_start_threads` 失败，我们直接跳转到 `fail_tq`（任务队列已分配但没有线程；`taskqueue_free` 仍然正确处理，因为刚创建且未启动的任务队列没有线程需要回收）。

还要注意：`TASK_INIT` 没有失败模式（它是一个设置字段的宏），也不需要对应的销毁。任务在 `taskqueue_drain` 被调用后变为空闲，其存储随 softc 一起回收。

### 线程命名约定

`taskqueue_start_threads` 接受一个格式字符串和可变参数列表，因此每个线程都有自己的名称。命名约定对可调试性有实际影响，所以一段简短的约定说明是值得的。

我们使用的格式字符串是 `"%s taskq"`，参数为 `device_get_nameunit(dev)`。对于第一个 `myfirst` 实例，线程显示为 `myfirst0 taskq`。对于第二个实例显示为 `myfirst1 taskq`。这使线程在 `procstat -t` 和 `ps ax` 中可识别。

有多个私有队列的驱动应选择区分队列的名称：

```c
taskqueue_start_threads(&sc->tx_tq, 1, PWAIT,
    "%s tx", device_get_nameunit(dev));
taskqueue_start_threads(&sc->rx_tq, 1, PWAIT,
    "%s rx", device_get_nameunit(dev));
```

网络驱动通常更具体地命名每队列线程（`"%s tx%d"` 加队列索引），以便 `procstat -t` 显示每个硬件队列的专用工作线程。

### 选择线程数量

大多数驱动创建单线程私有任务队列。一个工作线程意味着任务顺序运行，这简化了锁定故事：在任务回调内部，你可以假设同一回调没有其他调用在并发运行，无需任何显式排他。

有多个硬件通道需要并行处理的驱动可能在同一任务队列上创建多个工作线程。任务队列保证单个任务最多在一个线程上同时运行（这就是 `tq_active` 跟踪的），但同一队列上的不同任务可以在不同线程上并行运行。对于 `myfirst`，单线程配置是正确的。

多线程任务队列对锁争用有影响：同一队列上的两个工作线程，各自运行不同的任务，可能争用同一驱动互斥锁。如果工作负载天然可并行，多线程队列可以加速。如果工作负载无论如何都在驱动互斥锁上串行化，多线程只增加复杂性而无收益。对于第一个任务队列，单线程是正确的默认选择。

### 选择线程优先级

`taskqueue_start_threads` 的 `pri` 参数是线程的调度优先级。在第14章的示例中我们使用 `PWAIT`。实践中的选项有：

- `PWAIT`（数值 76）：普通驱动优先级，等同于 `taskqueue_thread` 的优先级。
- `PI_NET`（数值 4）：网络相关优先级，被许多以太网驱动使用。
- `PI_DISK`：历史常量；属于 `PRI_MIN_KERN` 范围。被存储驱动使用。
- `PRI_MIN_KERN`（数值 48）：通用内核线程优先级，当以上常量都不适用时使用。

对于任务工作不对延迟敏感的驱动，`PWAIT` 就够了。对于即使在负载下也必须及时运行任务回调的驱动，有时将优先级提高到接近中断线程是合理的。`myfirst` 使用 `PWAIT`。

如果你正在编写驱动但不确定约定期望什么优先级，可以查看 `/usr/src/sys/dev/` 中同类驱动的做法。使用 taskqueue 的存储驱动可能使用 `PRI_MIN_KERN` 或 `PI_DISK`；网络驱动可能使用 `PI_NET`。对现有驱动进行模式匹配比自己编造优先级更好。

### 实例源码阅读：`ale(4)`

一个使用本节所授确切模式的真实驱动。来自 `/usr/src/sys/dev/ale/if_ale.c`：

```c
/* Create local taskq. */
sc->ale_tq = taskqueue_create_fast("ale_taskq", M_WAITOK,
    taskqueue_thread_enqueue, &sc->ale_tq);
taskqueue_start_threads(&sc->ale_tq, 1, PI_NET, "%s taskq",
    device_get_nameunit(sc->ale_dev));
```

`ale` 以太网驱动创建了一个快速 taskqueue（`taskqueue_create_fast`），使用自旋互斥锁，因为它需要能从过滤器中断处理程序中入队。它以 `PI_NET` 优先级运行一个线程，使用每单元命名约定。其形状与我们在 `myfirst` 中使用的完全相同，只是快速 vs 普通的选择和优先级反映了驱动的上下文。

同一文件中匹配的拆卸路径：

```c
taskqueue_drain(sc->ale_tq, &sc->ale_int_task);
/* ... */
taskqueue_free(sc->ale_tq);
```

对特定任务调用 `taskqueue_drain`，然后对队列调用 `taskqueue_free`。与我们使用的惯用法相同。

阅读 `ale(4)` 的设置和拆卸一次是值得的。它是一个真实的驱动，做真实的工作，使用你即将在自己驱动中编写的模式。`/usr/src/sys/dev/` 下每个使用 taskqueue 的驱动都有非常相似的形状。

### 回归测试第13章行为

第2阶段不能破坏第13章建立的任何东西。在继续之前，用加载了第2阶段驱动的系统重新运行第13章的压力测试套件：

```text
# cd /path/to/examples/part-03/ch13-timers-and-delayed-work/stage4-final
# ./test-all.sh
```

测试应该像第13章结束时那样完全通过。如果不通过，回归出在第2阶段更改的某处；回滚到第2阶段之前的源码并找出差异。常见原因是遗漏了入队调用点的更新（某个入队仍指向 `taskqueue_thread` 而非 `sc->tq`）。这些代码编译没问题因为 API 相同；但会在运行时产生无关的 bug。

### 观察私有任务队列

加载第2阶段驱动后，`procstat -t` 显示新线程：

```text
# procstat -t | grep myfirst
  <PID> <THREAD>      0 100 myfirst0 taskq      sleep   -      -   0:00
```

名称 `myfirst0 taskq` 是我们在 `taskqueue_start_threads` 中请求的每实例线程名。状态为 `sleep` 是因为线程在等待任务时被阻塞。wchan 为空是因为线程在自己的条件变量上睡眠，`procstat` 在不同版本中可能显示不同。

启用 tick 源并再次观察：

```text
# sysctl dev.myfirst.0.tick_source_interval_ms=100
# procstat -t | grep myfirst
  <PID> <THREAD>      0 100 myfirst0 taskq      run     -      -   0:00
```

偶尔你可能捕捉到线程在 `run` 状态处理任务。大部分时间它处于 `sleep`。两种状态都是正常的。

`ps ax` 显示同样的线程：

```text
# ps ax | grep 'myfirst.*taskq'
   50  -  IL      0:00.00 [myfirst0 taskq]
```

方括号表示这是一个内核线程。该线程在驱动 attach 期间始终存在；在 detach 时消失。

### 第2阶段的预生产检查清单

一份简短的检查清单，在宣布第2阶段完成之前逐一过一遍。每一项都是一个问题；每一项都应该有信心地回答。

- [ ] `attach` 是否在任何可能入队的代码之前创建 taskqueue？
- [ ] `attach` 是否在任何期望任务实际运行的代码之前启动至少一个工作线程？
- [ ] `attach` 是否有失败标签，在后续初始化失败时 `taskqueue_free` 队列？
- [ ] `detach` 是否在释放任务触及的任何状态之前排空驱动拥有的每个任务？
- [ ] `detach` 是否在每个任务排空之后、销毁互斥锁之前调用 `taskqueue_free`？
- [ ] `detach` 是否在 free 之后设置 `sc->tq = NULL`，以保持防御性清晰？
- [ ] taskqueue 的线程优先级是否经过有意选择，理由与驱动的类型匹配？
- [ ] taskqueue 的线程名称是否足够有信息量，使 `procstat -t` 输出有用？
- [ ] 每个 `taskqueue_enqueue` 调用是否指向 `sc->tq`，而非 `taskqueue_thread`（除非入队因特定原因确实在共享路径上）？
- [ ] 每个 `taskqueue_drain` 调用是否在同一队列上用同一任务匹配一个 `taskqueue_enqueue`？

能干净地回答每一项的驱动就是一个正确处理其私有 taskqueue 的驱动。无法回答的驱动可能离 detach 时的释放后使用只有一步之遥。

### 第2阶段的常见错误

初学者在添加私有 taskqueue 时常犯的三个错误。每一个都可以通过一个好习惯来避免。

**在入队代码之后创建 taskqueue。** 如果入队发生在 `taskqueue_create` 返回之前，入队会解引用一个 `NULL` 指针。始终将 `taskqueue_create` 放在 attach 的早期，在任何可能触发入队的代码之前。

**忘记 `taskqueue_start_threads`。** 一个没有工作线程的 taskqueue 是一个接受入队但从不运行回调的队列。任务会静默堆积。如果你觉得"我的任务从不触发"，检查你是否调用了 `taskqueue_start_threads`。

**在未清除 `is_attached` 的情况下调用 `taskqueue_free`。** 如果 taskqueue 在 callout 回调仍在运行且可能入队时被释放，callout 的入队会在已释放的 taskqueue 上崩溃。始终先清除 `is_attached`，排空 callout，排空任务，然后释放。顺序正是使其安全的关键。

第7节将在一个故意错误排序的驱动上实时演练这些错误。目前的规则是：遵循本节的顺序，taskqueue 生命周期就会正确。

### 第4节总结

驱动现在拥有自己的 taskqueue。一个线程，一个名称，一个生命周期。Attach 创建它并启动工作线程；detach 排空任务、释放 taskqueue 并回滚。相对于 callout 和 selinfo 的顺序得到了遵守。`procstat -t` 以可识别的名称显示线程。驱动在其延迟工作的故事中是自包含的。

第5节将迈出下一步：我们有意识地利用合并行为进行批处理，引入 `timeout_task` 变体用于计划任务，并讨论当队列持有多种任务类型时优先级如何适用。



## 第5节：工作的优先级与合并

你驱动的每个任务都进入一个队列。队列有一个决定任务运行顺序和同一任务入队两次时发生什么的策略。第5节使该策略显式化，然后展示如何有意利用合并契约进行批处理，最后引入 `timeout_task` 作为任务队列侧的 callout 对应物。

本节思想密集但代码短小。第3阶段的两个驱动更改是同一任务队列上的一个新任务和一个驱动周期性批量写入的超时任务。价值在于你内化的规则。

### 优先级排序规则

`struct task` 中的 `ta_priority` 字段在单个队列内排序任务。更高优先级的任务在更低优先级的任务之前运行。优先级为 5 的任务在优先级为 0 的任务之后入队但在优先级为 0 的任务之前运行，即使优先级为 0 的任务先入队。

优先级是一个小无符号整数（`uint8_t`，范围 0-255）。大多数驱动所有任务使用优先级 0，此时队列实际上是 FIFO。有真正不同紧急程度任务的驱动可以分配不同优先级让任务队列重排序。

一个快速示例。假设驱动有两个任务：从硬件错误恢复的 `reset_task` 和汇总累积统计的 `stats_task`。如果两者在短窗口内被入队，重置应先运行。给 `reset_task` 优先级 10 和 `stats_task` 优先级 0 即可实现。重置任务即使最后入队也先运行。

谨慎使用优先级。有十种不同任务和十个不同优先级的驱动比有十种任务都以入队顺序运行的驱动更难推理。优先级用于真正的差异化，不是为了美观排序。

### 重述合并规则

从第2节回顾，值得再说一次：如果任务在已经待处理时被入队，内核递增 `ta_pending` 并且不会第二次链接任务。回调运行一次，第二个参数中有待处理计数。

精确代码，来自 `/usr/src/sys/kern/subr_taskqueue.c`：

```c
if (task->ta_pending) {
        if (__predict_false((flags & TASKQUEUE_FAIL_IF_PENDING) != 0)) {
                TQ_UNLOCK(queue);
                return (EEXIST);
        }
        if (task->ta_pending < USHRT_MAX)
                task->ta_pending++;
        TQ_UNLOCK(queue);
        return (0);
}
```

计数器在 `USHRT_MAX`（65535）处饱和，这是合计计数可以达到的硬上限。超过它后，重复入队从计数器的角度看会丢失，尽管它们仍然返回成功。实际上没人会触及那个上限，因为积压 65535 次的任务有更深层次的问题。

合并规则有三个你在设计中需要考虑的后果。

第一，**一个任务最多处理"每次调度器唤醒一次运行"**。如果你的工作模型需要"每个事件一次回调"，单个任务是错误的。你需要每事件状态。

第二，**回调必须能够在一次触发中处理多个事件**。将回调写成 `pending` 总是 1 是一个只在负载下才显现的 bug。有意使用 pending 参数，或者构建回调使其处理驱动拥有队列中的内容直到队列为空。

第三，**你可以利用合并进行批处理**。如果生产者每次事件入队任务一次，消费者每次触发排空一批，系统自然收敛到消费者能维持的速率。在轻负载下合并从不触发（一个事件，一次触发）。在重负载下合并将突发折叠为带有更大批次的单次触发。行为是自调节的。

### 一个有意的批处理模式：第3阶段

第3阶段向驱动添加第二个任务：一个 `bulk_writer_task`，在单次触发中写入固定数量的 tick 字节到缓冲区，由定期入队任务的 callout 驱动。这种模式是人为的（真正的驱动只会用更快的 tick 源），但它是有意批处理的最简单演示。

softc 添加：

```c
struct task             bulk_writer_task;
int                     bulk_writer_batch;      /* bytes per firing */
```

默认 `bulk_writer_batch` 为零（禁用）。一个 sysctl 暴露它以便调优。

回调：

```c
static void
myfirst_bulk_writer_task(void *arg, int pending)
{
        struct myfirst_softc *sc = arg;
        int batch, written;
        char buf[64];

        MYFIRST_LOCK(sc);
        batch = sc->bulk_writer_batch;
        MYFIRST_UNLOCK(sc);

        if (batch <= 0)
                return;

        batch = MIN(batch, (int)sizeof(buf));
        memset(buf, 'B', batch);

        MYFIRST_LOCK(sc);
        written = (int)cbuf_write(&sc->cb, buf, batch);
        if (written > 0) {
                counter_u64_add(sc->bytes_written, written);
                cv_signal(&sc->data_cv);
        }
        MYFIRST_UNLOCK(sc);

        if (written > 0)
                selwakeup(&sc->rsel);
}
```

几点说明。

回调获取 `sc->mtx`，读取批次大小，释放。获取和释放两次没问题；中间的工作（memset）不需要锁。第二次获取包装实际的 cbuf 操作和计数器更新。selwakeup 在没有锁持有的情况下发生，一如既往。

`pending` 参数在此简单回调中未使用。对于不同的批处理设计，`pending` 会告诉回调任务被入队了多少次因此累积了多少工作。这里的批处理策略是"每次触发总是精确写入 `bulk_writer_batch` 字节，不管入队多少次"，所以 `pending` 不参与。

回调不检查 `is_attached`。它不需要。分离在释放任务触碰的任何东西之前排空任务，`sc->mtx` 保护 `sc->cb` 直到排空完成。

### 实际合并的效果

为了有意演示合并，第3阶段添加一个 sysctl `bulk_writer_flood`，其写入者尝试在紧密循环中入队 `bulk_writer_task` 一千次：

```c
static int
myfirst_sysctl_bulk_writer_flood(SYSCTL_HANDLER_ARGS)
{
        struct myfirst_softc *sc = arg1;
        int flood = 0;
        int error, i;

        error = sysctl_handle_int(oidp, &flood, 0, req);
        if (error || req->newptr == NULL)
                return (error);
        if (flood < 1 || flood > 10000)
                return (EINVAL);

        for (i = 0; i < flood; i++)
                taskqueue_enqueue(sc->tq, &sc->bulk_writer_task);
        return (0);
}
```

运行它：

```text
# sysctl dev.myfirst.0.bulk_writer_batch=32
# sysctl dev.myfirst.0.bulk_writer_flood=1000
```

紧接着观察字节计数。没有合并的话，一千次入队每次 32 字节会产生 32000 字节。有了合并，实际数字是一次 32 字节的触发，因为一千次入队折叠为一个待处理任务。驱动的 `bytes_written` 计数器应增加 32，不是 32000。

这就是合并契约按设计工作的效果。生产者请求一千次任务运行；任务队列只交付了一次。回调的单次触发反映了所有一千个请求，但执行了批处理策略指定的固定工作量。

### 使用 `pending` 进行自适应批处理

一个更复杂的模式使用 `pending` 参数来适应批处理大小到队列深度。假设驱动想要每次触发写入 `pending` 字节：每次入队一个字节，折叠为合并运行。回调变为：

```c
static void
myfirst_adaptive_task(void *arg, int pending)
{
        struct myfirst_softc *sc = arg;
        char buf[64];
        int n;

        n = MIN(pending, (int)sizeof(buf));
        memset(buf, 'A', n);

        MYFIRST_LOCK(sc);
        (void)cbuf_write(&sc->cb, buf, n);
        counter_u64_add(sc->bytes_written, n);
        cv_signal(&sc->data_cv);
        MYFIRST_UNLOCK(sc);

        selwakeup(&sc->rsel);
}
```

回调写入 `pending` 字节（上限缓冲区大小）。在低负载时，`pending` 为 1，回调写入一个字节。在高负载时，`pending` 是回调开始时刻的队列深度，回调在一次遍历中写入那么多字节。批处理随负载自然缩放。

这种设计在每次入队对应一个想要一个工作单元的真实事件、且批处理是性能优化而非语义变更时有用。网络驱动的"发送完成"处理程序是经典例子：每个发送的包产生一个入队任务的中断；任务的工作是回收已完成的描述符；在高包速率下，多个中断折叠为单次任务触发，一次回收多个描述符。

我们不会在第3阶段的 `myfirst` 中添加自适应批处理任务，因为固定批次版本已经演示了合并。自适应模式值得在实际驱动工作中记住；你将读到的真实 FreeBSD 驱动经常使用它。

### 入队标志

`taskqueue_enqueue_flags` 用两个标志位扩展 `taskqueue_enqueue`：

- `TASKQUEUE_FAIL_IF_PENDING`：如果任务已经待处理，返回 `EEXIST` 而不是合并。
- `TASKQUEUE_FAIL_IF_CANCELING`：如果任务当前正在被取消，返回 `EAGAIN` 而不是等待。

`TASKQUEUE_FAIL_IF_PENDING` 在你想知道入队是否实际产生了新的待处理状态时有用，用于记账或调试。计算"此任务被入队了多少次"的驱动可以使用该标志，在冗余调用上得到 `EEXIST`，只计算非冗余入队。

`TASKQUEUE_FAIL_IF_CANCELING` 在关闭期间有用。如果你正在拆除驱动且某个代码路径会入队任务，你可以传递该标志并检查 `EAGAIN` 以避免重新添加正在取消过程中的任务。大多数驱动在实践中不需要这个；`is_attached` 检查通常处理等效条件。

`myfirst` 中没有使用任何一个标志。两者都存在，有特定需求的驱动可以使用它们。对于普通工作，普通的 `taskqueue_enqueue` 是正确的。

### `timeout_task` 变体

有时你想要一个任务在特定延迟后触发。callout 是自然的首选原语，但如果延迟回调想要做的工作需要线程上下文，你需要任务的上下文而不是 callout 的。内核为这种情况提供了 `struct timeout_task`。

`timeout_task` 定义在 `/usr/src/sys/sys/_task.h`：

```c
struct timeout_task {
        struct taskqueue *q;
        struct task t;
        struct callout c;
        int    f;
};
```

该结构包装了一个 `struct task`、一个 `struct callout` 和一个内部标志。当你用 `taskqueue_enqueue_timeout` 调度超时任务时，内核启动 callout；当 callout 触发时，其回调将底层任务入队到任务队列。任务随后在线程上下文中运行，拥有所有常规保证。

初始化使用 `TIMEOUT_TASK_INIT`：

```c
TIMEOUT_TASK_INIT(queue, timeout_task, priority, func, context);
```

该宏展开为函数调用 `_timeout_task_init`，它用适当的链接初始化任务和 callout。你必须在初始化时传递任务队列，因为 callout 被设置为在特定的队列上入队。

调度使用 `taskqueue_enqueue_timeout(tq, &tt, ticks)`：

```c
int taskqueue_enqueue_timeout(struct taskqueue *queue,
    struct timeout_task *timeout_task, int ticks);
```

`ticks` 参数与 `callout_reset` 使用相同约定：`hz` tick 等于一秒。

排空使用 `taskqueue_drain_timeout(tq, &tt)`，它等待 callout 到期（或如果仍待处理则取消它），然后等待底层任务完成。排空是一次调用，但它处理 callout 和任务两个阶段。

取消使用 `taskqueue_cancel_timeout(tq, &tt, &pendp)`：

```c
int taskqueue_cancel_timeout(struct taskqueue *queue,
    struct timeout_task *timeout_task, u_int *pendp);
```

如果超时被干净取消则返回零，如果任务当前正在运行则返回 `EBUSY`。在 `EBUSY` 情况下，你通常需要跟进 `taskqueue_drain_timeout`。

### 第3阶段的超时任务：延迟重置

第3阶段向驱动添加一个超时任务：一个延迟重置，在重置 sysctl 被写入 `reset_delay_ms` 毫秒后触发。现有重置 sysctl 同步运行；延迟变体将重置调度到稍后。对测试和重置不应该在当前 IO 排空之前发生的情况有用。

softc 添加：

```c
struct timeout_task     reset_delayed_task;
int                     reset_delay_ms;
```

attach 中的初始化：

```c
TIMEOUT_TASK_INIT(sc->tq, &sc->reset_delayed_task, 0,
    myfirst_reset_delayed_task, sc);
sc->reset_delay_ms = 0;
```

`TIMEOUT_TASK_INIT` 以任务队列作为第一个参数，因为 timeout_task 内部的 callout 需要知道触发时在哪个队列上入队。

回调：

```c
static void
myfirst_reset_delayed_task(void *arg, int pending)
{
        struct myfirst_softc *sc = arg;

        MYFIRST_LOCK(sc);
        MYFIRST_CFG_XLOCK(sc);

        cbuf_reset(&sc->cb);
        sc->cfg.debug_level = 0;
        counter_u64_zero(sc->bytes_read);
        counter_u64_zero(sc->bytes_written);

        MYFIRST_CFG_XUNLOCK(sc);
        MYFIRST_UNLOCK(sc);

        cv_broadcast(&sc->room_cv);
        device_printf(sc->dev, "delayed reset fired (pending=%d)\n", pending);
}
```

与第13章的同步重置逻辑相同，但在任务上下文中。它可以获取可睡眠的 `cfg_sx` 而没有 callout 会面临的复杂性。`pending` 计数用于诊断目的被记录。

武装延迟重置的 sysctl 处理程序：

```c
static int
myfirst_sysctl_reset_delayed(SYSCTL_HANDLER_ARGS)
{
        struct myfirst_softc *sc = arg1;
        int ms = 0;
        int error;

        error = sysctl_handle_int(oidp, &ms, 0, req);
        if (error || req->newptr == NULL)
                return (error);
        if (ms < 0)
                return (EINVAL);
        if (ms == 0) {
                (void)taskqueue_cancel_timeout(sc->tq,
                    &sc->reset_delayed_task, NULL);
                return (0);
        }

        sc->reset_delay_ms = ms;
        taskqueue_enqueue_timeout(sc->tq, &sc->reset_delayed_task,
            (ms * hz + 999) / 1000);
        return (0);
}
```

写入零取消待处理的延迟重置。任何正值调度任务在给定毫秒数后触发。tick 转换 `(ms * hz + 999) / 1000` 与我们用于 callout 的相同向上取整转换。

分离路径排空超时任务：

```c
taskqueue_drain_timeout(sc->tq, &sc->reset_delayed_task);
```

排空的位置与普通任务排空相同：在 callout 排空之后，`is_attached` 清除之后，`seldrain` 之前和 `taskqueue_free` 之前。

### 观察延迟重置

加载第3阶段后，将延迟重置武装为未来三秒：

```text
# sysctl dev.myfirst.0.reset_delayed=3000
```

三秒后 `dmesg` 显示：

```text
myfirst0: delayed reset fired (pending=1)
```

`pending=1` 确认超时任务触发了一次。现在快速连续地武装它多次：

```text
# sysctl dev.myfirst.0.reset_delayed=1000
# sysctl dev.myfirst.0.reset_delayed=1000
# sysctl dev.myfirst.0.reset_delayed=1000
```

一秒后，只有一个重置触发。`dmesg` 显示：

```text
myfirst0: delayed reset fired (pending=1)
```

为什么只有一次触发？因为 `taskqueue_enqueue_timeout` 的行为与 `callout_reset` 一致：武装一个待处理的超时任务替换之前的截止时间。三次连续武装产生一次调度触发。如果我们使用 `callout_reset` 对普通 callout 做同样的事情，行为相同。

### 何时使用 `timeout_task` 与 callout 加任务

当你想要线程上下文中的延迟操作且延迟是主要参数时，超时任务是正确的原语。当你想要延迟操作且延迟是实现细节时（例如，延迟每次动态重新计算），普通 callout 加任务入队是正确的原语。两者都可以工作。

两种模式在源码中有稍微不同的形态：

```c
/* timeout_task pattern */
TIMEOUT_TASK_INIT(tq, &tt, 0, fn, ctx);
...
taskqueue_enqueue_timeout(tq, &tt, ticks);
...
taskqueue_drain_timeout(tq, &tt);
```

```c
/* callout + task pattern */
callout_init_mtx(&co, &sc->mtx, 0);
TASK_INIT(&t, 0, fn, ctx);
...
callout_reset(&co, ticks, myfirst_co_fn, sc);
/* in the callout callback: taskqueue_enqueue(tq, &t); */
...
callout_drain(&co);
taskqueue_drain(tq, &t);
```

timeout_task 版本更短因为内核已经为你打包了模式。callout+任务版本更灵活因为 callout 回调可以动态决定是否入队任务（例如，基于调度时不存在的状态条件）。

对于 `myfirst` 的延迟重置，timeout_task 是正确选择，因为触发决策在调度时做出（sysctl 写入者请求了它）且中间没有任何东西改变那个决策。

### 跨任务种类的优先级排序

在同一任务队列上有多个任务的驱动可以使用优先级来排序它们。对于 `myfirst` 我们不需要这个；所有任务优先级相同。但当需要时这个模式值得理解。

假设我们有一个必须排在其他待处理任务之前的 `high_priority_reset_task`。我们会用大于零的优先级初始化它：

```c
TASK_INIT(&sc->high_priority_reset_task, 10,
    myfirst_high_priority_reset_task, sc);
```

并正常入队：

```c
taskqueue_enqueue(sc->tq, &sc->high_priority_reset_task);
```

如果队列有几个待处理任务，包括新的和几个优先级为 0 的任务，新的会因其更高优先级而先运行。优先级是任务的属性（在初始化时设置），不是入队的属性（在每次调用时设置）；如果一个任务有时紧急有时不紧急，你需要两个不同优先级的两个任务结构，而不是一个你重新调整的任务。

### 关于公平性的说明

有单个工作线程的任务队列严格按优先级顺序运行任务，平局按入队顺序打破。有多个工作线程的任务队列可以并行运行多个任务；优先级仍然排序列表，但并行工作线程在边际可能不按严格顺序分派任务。对大多数驱动这不重要。

如果需要严格公平性或严格优先级排序，单个工作线程是正确选择。如果以偶尔乱序为代价的吞吐量可接受，多个工作线程没问题。`myfirst` 使用单个工作线程。

### 第5节总结

第3阶段添加了一个有意的批处理任务和一个超时任务。批处理任务通过将一千次入队折叠为单次触发来演示合并；超时任务演示了线程上下文中的延迟执行。两者共享第2阶段的私有任务队列，两者在分离时以既定顺序排空，两者都遵守驱动其余部分使用的锁定纪律。

优先级和合并规则现在已显式化。任务的优先级在队列内对其排序；任务的 `ta_pending` 计数器将冗余入队折叠为单次触发，其 `pending` 参数携带总计。

第6节从 `myfirst` 重构中退后，审视在真实 FreeBSD 驱动中出现的模式。心智模型在积累；驱动直到第8节才再次更改。



## 第6节：使用任务队列的实际模式

到目前为止，第14章通过三个阶段开发了一个单一驱动。真实的 FreeBSD 驱动以少数几种重复形态使用任务队列。本节编目这些模式，展示每种模式在 `/usr/src/sys/dev/` 中出现的位置，并解释何时使用哪种。识别这些模式将阅读驱动源码从猜谜变成词汇练习。

每种模式以一个小方案呈现：问题、解决它的任务队列形态、代码草图，以及你可以阅读生产版本的真实驱动参考。

### 模式1：从边缘上下文延迟日志或通知

**问题。** 一个边缘上下文回调（callout、中断过滤器、epoch 段）检测到一个应该产生日志消息或通知用户空间的情况。日志调用对边缘上下文太重：`selwakeup`、`log(9)`、`kqueue_user_event`，或一个持有边缘上下文不能承受的锁的多行 `printf`。

**解决方案。** 每种情况一个 `struct task`，在 attach 中初始化，回调在线程上下文中执行重调用。边缘上下文回调在 softc 状态中记录情况（一个标志、一个计数器、一小块数据），入队任务，然后返回。任务在线程上下文中运行，从 softc 状态读取情况，执行调用，清除情况。

**代码草图。**

```c
struct my_softc {
        struct task log_task;
        int         log_flags;
        struct mtx  mtx;
        ...
};

#define MY_LOG_UNDERRUN  0x01
#define MY_LOG_OVERRUN   0x02

static void
my_log_task(void *arg, int pending)
{
        struct my_softc *sc = arg;
        int flags;

        mtx_lock(&sc->mtx);
        flags = sc->log_flags;
        sc->log_flags = 0;
        mtx_unlock(&sc->mtx);

        if (flags & MY_LOG_UNDERRUN)
                log(LOG_WARNING, "%s: buffer underrun\n",
                    device_get_nameunit(sc->dev));
        if (flags & MY_LOG_OVERRUN)
                log(LOG_WARNING, "%s: buffer overrun\n",
                    device_get_nameunit(sc->dev));
}

/* In an interrupt or callout callback: */
if (some_condition) {
        sc->log_flags |= MY_LOG_UNDERRUN;
        taskqueue_enqueue(sc->tq, &sc->log_task);
}
```

标志字段让边缘上下文在任务运行前累积多个不同情况。当任务触发时，它快照标志，清除它们，并按情况发出一行日志。合并将重复相同情况的入队折叠为一次回调调用，这正是你想要的日志垃圾防护。

**真实示例。** `/usr/src/sys/dev/ale/if_ale.c` 使用中断任务（`sc->ale_int_task`）处理过滤器中断的延迟工作，包括想要记录或通知的情况。

### 模式2：延迟重置或重新配置

**问题。** 驱动检测到需要硬件重置或配置更改的情况，但重置不应该立即发生。延迟的原因包括"给进行中的 IO 一个完成的机会"、"将多个原因合并为一次重置"或"限制重置以避免重置风暴"。

**解决方案。** 一个 `struct timeout_task`（或一个 `struct callout` 配对一个 `struct task`）。检测器以选定的延迟入队超时任务。如果情况在延迟到期前清除，检测器取消超时任务。如果情况持续，任务在线程上下文中触发并执行重置。

**代码草图。** 与 `myfirst` 第3阶段延迟重置任务形态相同。唯一的变体是检测器通常在"需要重置"状态改变时取消待处理任务，因此重置只在情况持续了完整延迟后才发生。

**真实示例。** 许多存储和网络驱动使用此模式进行恢复。`/usr/src/sys/dev/bge/if_bge.c` Broadcom 驱动在物理层事件后使用超时任务进行链路状态重新评估。

### 模式3：中断后处理（过滤器加任务分割）

**问题。** 硬件中断到达。中断过滤器的工作是判断"这是我们的中断吗，硬件真的需要关注吗"。过滤器必须快速运行且不能睡眠。实际处理（读寄存器、服务完成队列、可能 `copyout` 结果到用户空间）不属于过滤器。

**解决方案。** 两级分割。过滤器处理程序同步运行，读取状态寄存器，判断中断是否属于我们，如果是则入队一个任务。任务在线程上下文中运行并执行真正的工作。这是 `bus_setup_intr(9)` 原生支持的标准过滤器加 ithread 分割，但任务队列变体在驱动想要对延迟上下文有更多控制时有用。

**代码草图。**

```c
static int
my_intr_filter(void *arg)
{
        struct my_softc *sc = arg;
        uint32_t status;

        status = CSR_READ_4(sc, STATUS_REG);
        if (status == 0)
                return (FILTER_STRAY);

        /* Mask further interrupts from the hardware. */
        CSR_WRITE_4(sc, INTR_MASK_REG, 0);

        taskqueue_enqueue(sc->tq, &sc->intr_task);
        return (FILTER_HANDLED);
}

static void
my_intr_task(void *arg, int pending)
{
        struct my_softc *sc = arg;

        mtx_lock(&sc->mtx);
        my_process_completions(sc);
        mtx_unlock(&sc->mtx);

        /* Unmask interrupts again. */
        CSR_WRITE_4(sc, INTR_MASK_REG, ALL_INTERRUPTS);
}
```

几个微妙之处。过滤器在入队任务之前在硬件层面屏蔽中断，因此硬件在任务待处理时不会持续触发。任务在线程上下文中运行，处理完成，并在最后重新启用中断。合并将多个中断折叠为单次任务触发；屏蔽防止硬件无限制触发。第四部分将逐步讲解真实中断设置；这里展示的模式是该章为你准备的形态。

**真实示例。** `/usr/src/sys/dev/ale/if_ale.c`、`/usr/src/sys/dev/age/if_age.c` 和大多数以太网驱动使用此模式或其接近变体。

### 模式4：硬件完成后的异步 `copyin`/`copyout`

**问题。** 驱动有一个排队的用户空间请求，提供了输入或输出数据的地址。硬件完成以中断到达。驱动必须在用户空间和内核缓冲区之间复制数据来完成请求。`copyin` 和 `copyout` 在慢路径上睡眠，所以它们不能在中断上下文中运行。

**解决方案。** 中断路径记录请求标识符并入队一个任务。任务在线程上下文中运行，从存储的请求状态中识别用户空间地址，执行 `copyin` 或 `copyout`，并唤醒等待的用户线程。

**代码草图。**

```c
struct my_request {
        struct task finish_task;
        struct proc *proc;
        void *uaddr;
        void *kaddr;
        size_t len;
        int done;
        struct cv cv;
        /* ... */
};

static void
my_finish_task(void *arg, int pending)
{
        struct my_request *req = arg;

        (void)copyout(req->kaddr, req->uaddr, req->len);

        mtx_lock(&req->sc->mtx);
        req->done = 1;
        cv_broadcast(&req->cv);
        mtx_unlock(&req->sc->mtx);
}

/* In the interrupt task: */
taskqueue_enqueue(sc->tq, &req->finish_task);
```

用户空间线程在提交请求后等待 `req->cv`；当任务标记 `done` 并广播时它醒来。

**真实示例。** 实现带有大数据传输 ioctl 的字符设备驱动有时使用此模式。`/usr/src/sys/dev/usb/` 中的 USB 批量传输完成频繁通过任务延迟用户空间数据复制。

### 模式5：瞬态失败后的退避重试

**问题。** 硬件操作失败，但失败已知是瞬态的。驱动想在退避间隔后重试，重复失败时增加退避。

**解决方案。** 一个 `struct timeout_task`，在每次失败时用增加的延迟重新武装。任务回调执行重试；成功时驱动清除退避；失败时任务用更大的延迟重新入队。

**代码草图。**

```c
struct my_softc {
        struct timeout_task retry_task;
        int retry_interval_ms;
        int retry_attempts;
        /* ... */
};

static void
my_retry_task(void *arg, int pending)
{
        struct my_softc *sc = arg;
        int err;

        err = my_attempt_operation(sc);
        if (err == 0) {
                sc->retry_attempts = 0;
                sc->retry_interval_ms = 10;
                return;
        }

        sc->retry_attempts++;
        if (sc->retry_attempts > MAX_RETRIES) {
                device_printf(sc->dev, "giving up after %d attempts\n",
                    sc->retry_attempts);
                return;
        }

        sc->retry_interval_ms = MIN(sc->retry_interval_ms * 2, 5000);
        taskqueue_enqueue_timeout(sc->tq, &sc->retry_task,
            (sc->retry_interval_ms * hz + 999) / 1000);
}
```

初始间隔为 10 毫秒，每次失败翻倍，上限 5 秒，有最大尝试次数。重试持续触发直到成功或放弃。单独的代码路径可以在激励它的条件改变时取消重试（用 `taskqueue_cancel_timeout`）。

**真实示例。** `/usr/src/sys/dev/iwm/if_iwm.c` 和其他无线驱动使用超时任务进行固件加载重试和链路重新校准。

### 模式6：延迟拆除

**问题。** 驱动内部的一个对象必须被释放，但其他代码路径可能仍然持有引用。立即释放会是释放后使用。驱动需要稍后释放，在引用已知消失后。

**解决方案。** 一个 `struct task`，其回调释放对象。想要释放对象的代码路径入队该任务；任务在线程上下文中运行，在任何未完成引用都有机会完成后。

在更复杂的形式中，该模式使用引用计数：任务递减引用计数，只在计数归零时释放对象。在更简单的形式中，任务队列的 FIFO 排序足够了：所有更早入队的任务在拆除任务运行前完成，因此如果引用总是在任务内获取，它们在拆除任务触发时都已消失。

**代码草图。**

```c
static void
my_free_task(void *arg, int pending)
{
        struct my_object *obj = arg;

        /* All earlier tasks on this queue have completed. */
        free(obj, M_DEVBUF);
}

/* When we want to free the object: */
static struct task free_task;
TASK_INIT(&free_task, 0, my_free_task, obj);
taskqueue_enqueue(sc->tq, &free_task);
```

注意：`struct task` 本身必须存活到回调触发，这意味着要么将其嵌入对象中（并释放包含结构），要么单独分配它。

**真实示例。** `/usr/src/sys/dev/usb/usb_hub.c` 在 USB 设备被移除但仍被上层栈驱动使用时使用延迟拆除。

### 模式7：定时统计滚动

**问题。** 驱动维护必须在定期边界滚动为每间隔速率的累积统计。滚动涉及快照计数器、计算增量并将结果存储在环形缓冲区中。这可以在定时器回调中完成，但计算触碰由可睡眠锁保护的数据结构。

**解决方案。** 一个周期性 `timeout_task` 在线程上下文中处理滚动。任务在每次触发结束时重新入队自己以备下一个间隔。

这实际上是"基于任务队列的 callout"。它比普通 callout 稍重因为它搭载了一个 callout 加任务组合，但它可以做普通 callout 不能做的事情。只在单独的 callout 不够时才有用。

**代码草图。**

```c
static void
my_stats_rollover_task(void *arg, int pending)
{
        struct my_softc *sc = arg;

        sx_xlock(&sc->stats_sx);
        my_rollover_stats(sc);
        sx_xunlock(&sc->stats_sx);

        taskqueue_enqueue_timeout(sc->tq, &sc->stats_task, hz);
}
```

末尾的自重新入队保持任务每秒触发一次。想要停止滚动的控制路径取消超时任务。

**真实示例。** 几个网络驱动对工作需要可睡眠锁的看门狗邻近定时器使用此模式。

### 模式8：精细配置期间的 `taskqueue_block`

**问题。** 驱动正在执行不能被任务执行中断的配置更改。在配置中间触发的任务可能观察到不一致的状态。

**解决方案。** 在配置更改前 `taskqueue_block(sc->tq)`；配置更改后 `taskqueue_unblock(sc->tq)`。被阻塞时，新入队累积但不分派任务。已运行的任务（如果有）在阻塞生效前自然完成。

**代码草图。**

```c
taskqueue_block(sc->tq);
/* ... reconfigure ... */
taskqueue_unblock(sc->tq);
```

`taskqueue_block` 很快。它不排空运行中的任务；它只阻止新任务的分派。为了获得没有任务正在运行的保证，你将它与 `taskqueue_quiesce` 结合使用：

```c
taskqueue_block(sc->tq);
taskqueue_quiesce(sc->tq);
/* ... reconfigure ... */
taskqueue_unblock(sc->tq);
```

`taskqueue_quiesce` 等待当前运行的任务完成并等待待处理队列为空。与 `block` 结合，你获得了没有任务在运行且没有任务会启动的保证，直到你解除阻塞。

**真实示例。** 一些以太网驱动在接口状态转换（链路启动、链路断开、媒体更改）期间使用此模式。

### 模式9：子系统边界的 `taskqueue_drain_all`

**问题。** 复杂子系统想要在特定点完全安静。所有待处理任务，包括可能被其他待处理任务入队的任务，必须在子系统继续之前完成。

**解决方案。** `taskqueue_drain_all(tq)` 排空队列中的每个任务，等待每个进行中的任务完成，在队列安静时返回。

`taskqueue_drain_all` 不是分离时每任务 `taskqueue_drain` 的替代品（因为队列可能有来自其他路径不应被排空的任务），但对于你想要"一切都完成了，句号"的内部同步点很有用。

**真实示例。** `/usr/src/sys/dev/wg/if_wg.c` 在对等端清理期间在其每对等任务队列上使用 `taskqueue_drain_all`。

### 模式10：仿真级合成事件生成

**问题。** 在测试期间，驱动想要生成演练完整事件处理路径的合成事件。直接函数调用会绕过调度器、错过竞争条件，并且不会给任务队列机制施加压力。真实硬件事件在测试台上当然不可用。

**解决方案。** 一个入队任务的 sysctl 处理程序。任务回调调用真实事件会调用的相同驱动例程。因为任务通过任务队列，合成事件具有与真实事件相同的执行形态：它在线程上下文中运行，观察相同的锁定，并通过相同的合并。

这正是 `myfirst` 的 `bulk_writer_flood` sysctl 做的。该模式可以转移到任何想要自测延迟工作路径而无需真实硬件生成触发事件的驱动。

### 真实驱动精选

上述模式不是为本章发明的。`/usr/src/sys/dev/` 的简短之旅，你应该自己探索，建议顺序：

- **`/usr/src/sys/dev/ale/if_ale.c`**：一个小型、可读的以太网驱动，使用私有任务队列、过滤器加任务分割和单个中断任务。好的第一阅读。
- **`/usr/src/sys/dev/age/if_age.c`**：类似模式，略有不同的驱动系列。两者都阅读可以强化模式。
- **`/usr/src/sys/dev/bge/if_bge.c`**：更大的以太网驱动，有多个任务（中断任务、链路任务、重置任务）。展示多个任务如何在一个队列上组合。
- **`/usr/src/sys/dev/usb/usb_process.c`**：USB 的专用每设备进程队列（`usb_proc_*`）。演示子系统如何为自己的领域包装任务风格的延迟工作。
- **`/usr/src/sys/dev/wg/if_wg.c`**：WireGuard 使用组任务队列进行每对等端加密。进阶阅读，但在基本模式理解后有用。
- **`/usr/src/sys/dev/iwm/if_iwm.c`**：带有多个超时任务用于校准、扫描和固件管理的无线驱动。
- **`/usr/src/sys/kern/subr_taskqueue.c`**：实现本身。阅读 `taskqueue_run_locked` 一次让其他一切变得具体。

花二十分钟阅读其中任何一个文件相当于省去一小时的章节解释。一旦你知道要寻找什么，模式一眼就能看出来。

### 第6节总结

同一个小型 API 组合成一个大型模式家族。延迟日志、过滤器加任务中断分割、异步 `copyin`/`copyout`、退避重试、延迟拆除、统计滚动、重配置期间阻塞、子系统边界排空所有、合成事件生成：每种模式都是"边缘检测、任务行动"的变体，每当你在编写或阅读的驱动符合该形态时都很有用。

第7节转向同一硬币的另一面：当模式出错时，你如何看到？FreeBSD 提供了哪些工具来检查任务队列状态，这些工具帮助你诊断哪些常见 bug？



## 第7节：调试任务队列

大多数任务队列代码很短。常见 bug 并不微妙：从不触发的任务、触发太频繁的任务、softc 释放后触发的任务、与驱动互斥锁的死锁、以及在分离序列中错误位置的排空。本节命名这些 bug，展示如何观察它们，并在 `myfirst` 上逐步进行一个故意的破坏-修复以便你可以在面前有东西的情况下练习调试工作流程。

### 工具

你将使用的工具简短概述。

**`procstat -t`**：列出每个内核线程及其名称、优先级、状态和等待通道。私有任务队列的工作线程显示为 `<name> taskq`，其中 `<name>` 是你传递给 `taskqueue_start_threads` 的名称。卡在非平凡等待通道中的线程是线索：通道名称通常告诉你线程在等待什么。

**`ps ax`**：等同于 `procstat -t` 显示的大多数内容，输出不太针对任务队列。内核线程名出现在方括号中。

**`sysctl dev.<driver>`**：驱动自己的 sysctl 树。如果你添加了像 `selwake_pending_drops` 这样的计数器，其值在这里可见。诊断 sysctl 是最便宜的可观察性形式；在"此路径触发频率是否重要"可能重要时随时添加它们。

**`dtrace(1)`**：内核跟踪框架。任务队列活动可通过 `taskqueue_enqueue` 和 `taskqueue_run_locked` 上的 FBT（函数边界跟踪）探测来跟踪。一个简短的 D 脚本可以计数入队、测量入队到调度之间的延迟等。

**`ktr(4)`**：内核事件跟踪器。在调试内核中编译时启用，提供可在崩溃后转储或实时检查的内核事件环形缓冲区。对事后分析有用。

**`ddb(4)`**：内核内调试器。断点、栈跟踪、内存检查。通过 `kgdb` 在内核崩溃后可达，或在构建了 KDB 的内核上通过 `sysctl debug.kdb.enter=1` 交互式可达。

**`INVARIANTS` 和 `WITNESS`**：编译时断言和锁序检查器。不是你调用的工具，而是第一道防线。调试内核在你第一次命中时就捕获大多数任务队列 bug。

第14章实验明确练习 `procstat -t`、`sysctl` 和 `dtrace`。`ktr` 和 `ddb` 为完整性提及。

### 常见Bug 1：任务从不运行

**症状。** 你从 sysctl 处理程序或 callout 回调入队任务；任务回调的 `device_printf` 从不出现；驱动其他方面似乎工作正常。

**可能原因。** `taskqueue_start_threads` 未被调用，或你入队到的任务队列指针为 `NULL`。

**如何检查。**

```text
# procstat -t | grep myfirst
```

如果没有 `myfirst taskq` 线程列出，任务队列要么不存在，要么没有线程。检查 attach 路径：`taskqueue_create` 是否被调用了？它的返回值是否被存储了？之后是否调用了 `taskqueue_start_threads`？

```text
# dtrace -n 'fbt::taskqueue_enqueue:entry /arg0 != 0/ { @[stack()] = count(); }'
```

如果栈跟踪显示在驱动的任务队列上入队，任务正在被提交。如果什么也没显示，应该入队的代码路径没有到达。回溯并找出原因。

### 常见Bug 2：任务运行太频繁

**症状。** 任务回调做的工作比预期多，或驱动记录了奇怪的计数。

**可能原因。** 回调不遵守 `pending` 参数，或回调无条件地自入队，所以一旦启动就永远循环。

**如何检查。** 在回调中添加计数器：

```c
static void
myfirst_task(void *arg, int pending)
{
        struct myfirst_softc *sc = arg;
        static int invocations;

        atomic_add_int(&invocations, 1);
        if ((invocations % 1000) == 0)
                device_printf(sc->dev, "task invocations=%d\n", invocations);
        /* ... */
}
```

如果计数器增长快于预期触发率，任务正在自循环或合并没有发生。检查入队调用点。

### 常见Bug 3：任务回调中的释放后使用

**症状。** 内核崩溃，栈跟踪结束于你的任务回调中访问 softc 状态的位置。崩溃可能在分离期间或之后不久发生。

**可能原因。** 分离路径在排空任务之前释放了 softc（或任务触碰的某些东西）。来自 callout 或其他边缘上下文的尾随入队在排空后触发，任务对已释放状态运行。

**如何检查。** 根据第4节的顺序审查分离路径。具体地：

1. `is_attached` 必须在 callout 排空前清除，以便 callout 回调提前退出而不再入队。
2. callout 必须在任务排空前排空，以便排空任务后不再有入队发生。
3. 任务必须在它们触碰的状态被释放前排空。
4. `taskqueue_free` 必须在队列上所有任务排空后调用。

其中任何不匹配都是潜在的释放后使用。

调试内核通过 `cbuf_*`、`cv_*` 和 `mtx_*` 例程中的 `INVARIANTS` 断言捕获许多此类情况。在启用 `WITNESS` 的情况下在负载下运行分离路径；bug 通常会立即浮出。

### 常见Bug 4：任务与驱动互斥锁间的死锁

**症状。** 对设备的 `read(2)` 或 `write(2)` 永远挂起。任务回调卡在锁等待中。驱动互斥锁由不同线程持有。

**可能原因。** 任务回调尝试获取入队任务的线程已持有的锁，创建了循环。例如：

- 线程 A 持有 `sc->mtx` 并调用一个入队任务的函数。
- 任务回调在做其工作前获取 `sc->mtx`。
- 任务无法继续因为线程 A 仍持有互斥锁。
- 线程 A 等待任务完成。

"线程 A 等待任务完成"部分不符合 `myfirst` 架构（驱动不从互斥锁持有路径内部显式等待任务），但它是其他驱动中的常见形态。通过不在持有它们需要的锁时排空任务来避免它。

**如何检查。**

```text
# procstat -kk <pid of stuck read/write thread>
# procstat -kk <pid of taskqueue thread>
```

比较栈跟踪。如果一个在任务回调中显示 `mtx_lock`/`sx_xlock`，另一个在持有同一锁的位置显示 `msleep_sbt`/`sleepqueue`，你就有死锁。

### 常见Bug 5：排空永远挂起

**症状。** 分离挂起，`kldunload` 不返回。任务队列线程卡在某处。

**可能原因。** 任务回调在等待一个无法满足的条件因为排空路径阻塞了生产者。或者任务回调在等待分离路径持有的锁。

**如何检查。**

```text
# procstat -kk <pid of kldunload>
# procstat -kk <pid of taskqueue thread>
```

排空在 `taskqueue_drain` 中，处于 `msleep`。任务在某个等待中。识别等待通道；名称通常告诉你任务阻塞在什么上。如果任务阻塞在分离路径持有的东西上，设计有循环。

一个常见的具体情况：任务回调调用 `seldrain`，分离路径也调用 `seldrain`，两者冲突。通过确保 `seldrain` 在分离路径中只调用一次、在任务排空之后来避免。

### 破坏-修复练习

一个故意的 bug 和修复演练。第1阶段驱动是正确的；我们修改它以引入上述每个 bug，观察症状，然后修复它。

#### 损坏变体1：缺少 `taskqueue_start_threads`

从 attach 中移除 `taskqueue_start_threads` 调用。重新构建、加载、启用 tick 源，并运行 `poll_waiter`。你会观察到：`poll_waiter` 中没有数据出现，即使 `sysctl dev.myfirst.0.tick_source_interval_ms` 已设置。

检查 `procstat -t`：

```text
# procstat -t | grep myfirst
```

没有 `myfirst taskq` 线程出现。任务队列存在（你创建了它）但没有工作线程。入队的 `selwake_task` 永远停在队列上。

修复：把 `taskqueue_start_threads` 调用放回去。重新构建。确认线程出现在 `procstat -t` 中且 `poll_waiter` 看到数据。

#### 损坏变体2：以错误顺序排空

将 detach 中的 `taskqueue_drain` 调用移到 callout 排空之前：

```c
/* WRONG ORDER: */
taskqueue_drain(sc->tq, &sc->selwake_task);
MYFIRST_CO_DRAIN(&sc->heartbeat_co);
MYFIRST_CO_DRAIN(&sc->watchdog_co);
MYFIRST_CO_DRAIN(&sc->tick_source_co);
seldrain(&sc->rsel);
seldrain(&sc->wsel);
```

重新构建、加载、以高速率启用 tick 源，让数据流几秒钟，然后卸载。大多数时候卸载成功。偶尔，卸载会在 `selwakeup` 被调用时崩溃，在 `seldrain` 之后。竞争很少但真实存在。

问题：`taskqueue_drain` 返回了，但随后一个进行中的 `tick_source` callout 触发了（它还没被排空）并重新入队了任务。新任务在 `seldrain` 运行后触发，并尝试对一个已排空的 selinfo 调用 `selwakeup`。

修复：恢复正确顺序（先 callout，然后任务，然后 `seldrain`）。重新构建，验证竞争在相同压力下消失。

#### 损坏变体3：任务回调持有互斥锁太久

将 `myfirst_selwake_task` 改为在 `selwakeup` 期间持有互斥锁：

```c
static void
myfirst_selwake_task(void *arg, int pending)
{
        struct myfirst_softc *sc = arg;

        MYFIRST_LOCK(sc);        /* WRONG: holds mutex across selwakeup */
        selwakeup(&sc->rsel);
        MYFIRST_UNLOCK(sc);
}
```

重新构建。在调试内核下加载。启用 tick 源。几秒内内核崩溃，`WITNESS` 抱怨锁序（或某些配置中 `selwakeup` 本身的断言失败）。

问题：`selwakeup` 获取一个不在驱动文档记录锁序中的锁。`WITNESS` 注意到并抱怨。

修复：正确的 `myfirst_selwake_task` 在没有驱动互斥锁持有的情况下调用 `selwakeup`。恢复它，重新构建，验证没有 WITNESS 警告。

#### 损坏变体4：忘记在 detach 中排空任务

从 detach 中移除 `taskqueue_drain(sc->tq, &sc->selwake_task)` 行。重新构建。加载，以高速率启用 tick 源，运行 `poll_waiter`，然后立即卸载驱动。

大多数时候卸载完成。偶尔，卸载时进行中的任务对 selinfo 已被排空和释放的 softc 运行。症状通常是内核崩溃或后来作为不相关崩溃出现的内存损坏。

修复：恢复排空。重新构建，验证在负载下重复加载-卸载是稳定的。

#### 损坏变体5：错误的任务队列指针

一个微妙的第2阶段 bug。转移到私有任务队列后，忘记更新 detach 中的 `taskqueue_drain` 调用。它仍然指向 `taskqueue_thread`：

```c
/* WRONG: enqueue on sc->tq but drain on taskqueue_thread */
taskqueue_enqueue(sc->tq, &sc->selwake_task);
/* ... in detach ... */
taskqueue_drain(taskqueue_thread, &sc->selwake_task);
```

重新构建。加载，启用 tick 源，运行等待者，卸载。卸载通常无错完成，但 `taskqueue_drain(taskqueue_thread, ...)` 实际上不会等待运行在 `sc->tq` 上的任务。如果任务在 detach 继续时进行中，就会发生释放后使用。

修复：在同一任务队列指针上匹配入队和排空。重新构建，测试。

### 一个 DTrace 单行命令

对任何使用任务队列的驱动有用的单行命令。它测量系统上每个任务的入队到调度延迟：

```text
# dtrace -n '
  fbt::taskqueue_enqueue:entry { self->t = timestamp; }
  fbt::taskqueue_run_locked:entry /self->t/ {
        @[execname] = quantize(timestamp - self->t);
        self->t = 0;
  }
'
```

输出是每个进程的入队到调度延迟分布。在你的驱动产生任务时运行它，然后按 Ctrl-C 查看量化直方图。轻度负载机器上的典型结果：几十微秒。在负载下：毫秒级。如果你看到秒级，说明有问题。

第二个有用的单行命令测量任务回调持续时间：

```text
# dtrace -n '
  fbt::taskqueue_run_locked:entry { self->t = timestamp; }
  fbt::taskqueue_run_locked:return /self->t/ {
        @[execname] = quantize(timestamp - self->t);
        self->t = 0;
  }
'
```

相同结构，不同计时。告诉你每次 `taskqueue_run_locked` 调用花费多长时间（即回调持续时间加上一个小的固定开销）。

### 要添加的诊断 Sysctl

对任何使用任务队列的驱动有用的计数器，成本最小，诊断价值高。

```c
int enqueues;           /* Total enqueues attempted. */
int pending_drops;      /* Enqueues that coalesced. */
int callback_runs;      /* Total callback invocations. */
int largest_pending;    /* Peak pending count observed. */
```

在入队路径和回调中更新计数器：

```c
static void
myfirst_task(void *arg, int pending)
{
        struct myfirst_softc *sc = arg;

        sc->callback_runs++;
        if (pending > sc->largest_pending)
                sc->largest_pending = pending;
        if (pending > 1)
                sc->pending_drops += pending - 1;
        /* ... */
}

/* Enqueue site: */
sc->enqueues++;
taskqueue_enqueue(sc->tq, &sc->task);
```

将每个暴露为只读 sysctl。正常负载下，`enqueues == callback_runs + pending_drops`。`largest_pending` 告诉你最糟糕的合并时刻；如果它增长，任务队列正在落后于生产者。

这些计数器每次入队花费少量原子添加。在任何现实工作负载上成本不可衡量。诊断价值是实质性的。

### 调试内核义务

值得重复的提醒：在启用了 `INVARIANTS`、`WITNESS`、`WITNESS_SKIPSPIN`、`DDB`、`KDB` 和 `KDB_UNATTENDED` 的内核下运行第14章的每一次更改。大多数在生产内核上难以找到的任务队列 bug 在调试内核上立即被捕获。运行调试内核的成本是小的性能损失和稍大的构建；不运行的成本是每当出问题时的一个下午的调试。

### 第7节总结

调试任务队列是一项小技能，与你已有的调试 callout、互斥锁和 cv 的工具组合。`procstat -t` 和 `ps ax` 显示线程。`sysctl` 暴露诊断计数器。`dtrace` 测量入队到调度延迟和回调持续时间。`WITNESS` 在运行时捕获锁序违规。常见 bug（任务从不运行、错误排空顺序、回调中错误锁纪律、遗忘排空）每一个都可以通过检查表和调试内核捕获。

第8节将第14章的工作整合到第4阶段，最终驱动中。我们扩展 `LOCKING.md`，提升版本字符串，并根据完整的压力测试审计驱动。



## 第8节：重构任务队列驱动程序并更新版本

第4阶段是整合阶段。它不添加超出第3阶段建立的新功能；它锐化代码组织，更新文档，提升版本，并运行完整的回归扫描。如果第1到第3阶段是你构建驱动的阶段，第4阶段是你交付的阶段。

本节逐步讲解整合过程。驱动源码被统一为单一的、结构良好的文件；`LOCKING.md` 获得一个 Tasks 节；版本字符串提升到 `0.8-taskqueues`；最终回归通过确认每个第12章和第13章行为在新的第14章添加旁仍然正确工作。

### 文件组织

本章不将驱动拆分为多个 `.c` 文件。`myfirst.c` 保持为单一翻译单元，增加一项责任（任务）分组在相应 callout 旁边。如果驱动变得大得多，自然的拆分将是 `myfirst_timers.c` 用于 callout 代码和 `myfirst_tasks.c` 用于任务代码，共享声明在 `myfirst.h` 中。对于当前大小，单文件更容易阅读。

在 `myfirst.c` 内部，第4阶段的组织是：

1. 头文件和全局宏。
2. Softc 结构。
3. 文件句柄结构。
4. cdevsw 声明。
5. 缓冲区辅助函数。
6. 条件变量等待辅助函数。
7. Sysctl 处理函数，分组：
   - 配置 sysctl（调试级别、软字节限制、昵称）。
   - 定时器间隔 sysctl（心跳、看门狗、tick 源）。
   - 任务 sysctl（延迟重置、批量写入批次、批量写入洪泛）。
   - 只读统计 sysctl。
8. Callout 回调。
9. 任务回调。
10. Cdev 处理函数（open、close、read、write、poll、句柄析构函数）。
11. 设备方法（identify、probe、attach、detach）。
12. 模块胶水代码（驱动、`DRIVER_MODULE`、版本）。

文件顶部的一块注释列出主要节，以便新读者可以不用 grep 就跳到正确区域。每个节内部，顺序是先建立的：心跳在 watchdog 之前，watchdog 在 tick 源之前，对于大纲中共享该顺序的 callout。

### `LOCKING.md` 更新

第13章的 `LOCKING.md` 有互斥锁、cv、sx 和 callout 的节。第14章添加一个 Tasks 节。

```markdown
## Tasks

The driver owns one private taskqueue (`sc->tq`) and three tasks:

- `selwake_task` (plain): calls `selwakeup(&sc->rsel)`. Enqueued from
  `myfirst_tick_source` when a byte is written. Drained at detach after
  callouts are drained and before `seldrain`.
- `bulk_writer_task` (plain): writes a configured number of bytes to the
  cbuf, signals `data_cv`, calls `selwakeup(&sc->rsel)`. Enqueued from
  sysctl handlers and from the tick_source callback when
  `bulk_writer_batch` is non-zero. Drained at detach after callouts.
- `reset_delayed_task` (timeout_task): performs a delayed reset of the
  cbuf, counters, and configuration. Enqueued by the
  `reset_delayed` sysctl. Drained at detach.

The taskqueue is created in `myfirst_attach` with `taskqueue_create`
and one worker thread started at `PWAIT` priority via
`taskqueue_start_threads`. It is freed in `myfirst_detach` via
`taskqueue_free` after every task has been drained.

All task callbacks run in thread context. Each callback acquires
`sc->mtx` explicitly if it needs state protected by the mutex; the
taskqueue framework does not acquire driver locks automatically.

All task callbacks call `selwakeup(9)` (when they call it at all) with
no driver lock held. The rule is the same as for the `myfirst_read` /
`myfirst_write` paths: drop the mutex before `selwakeup`.

## Detach Ordering

The detach sequence is:

1. Refuse detach if `sc->active_fhs > 0` (EBUSY).
2. Clear `sc->is_attached` under `sc->mtx`.
3. Broadcast `data_cv` and `room_cv`.
4. Release `sc->mtx`.
5. Drain `heartbeat_co`, `watchdog_co`, `tick_source_co`.
6. Drain `selwake_task`, `bulk_writer_task`, `reset_delayed_task`
   (the last via `taskqueue_drain_timeout`).
7. `seldrain(&sc->rsel)`, `seldrain(&sc->wsel)`.
8. `taskqueue_free(sc->tq)`.
9. Destroy cdev and cdev alias.
10. Free sysctl context.
11. Destroy cbuf, free counters.
12. Destroy `data_cv`, `room_cv`, `cfg_sx`, `mtx`.

Violating the order risks use-after-free in task callbacks, selinfo
accesses after drain, or taskqueue teardown while a task is still
running.
```

更新明确说明了排序因为排序是可能出错的主要地方。一个从你那里继承驱动并想要添加新任务的读者会发现现有的纪律已被清楚写明。

### 版本提升

源码中的版本字符串从 `0.7-timers` 移到 `0.8-taskqueues`：

```c
#define MYFIRST_VERSION "0.8-taskqueues"
```

驱动的探测字符串更新：

```c
device_set_desc(dev, "My First FreeBSD Driver (Chapter 14 Stage 4)");
```

版本通过 `hw.myfirst.version` sysctl 可见，这在第12章中建立。

### 最终回归通过

第4阶段必须通过第1到第3阶段通过的每个测试，加上第12章和第13章的测试套件。一个紧凑的通过顺序：

1. **干净构建** 在调试内核下（`make clean && make`）。
2. **加载** 用 `kldload ./myfirst.ko`。
3. **第11章单元测试**：基本读、写、打开、关闭、重置。
4. **第12章同步测试**：有界阻塞读、有界阻塞写、超时读、sx 保护的配置、分离时 cv 广播。
5. **第13章定时器测试**：心跳以配置速率触发、看门狗检测排放停滞、tick 源注入字节。
6. **第14章任务测试**：
   - `poll_waiter` 在 tick 源活跃时看到数据。
   - `selwake_pending_drops` 计数器在负载下增长。
   - `bulk_writer_flood` 触发合并为单次回调。
   - `reset_delayed` 在配置的延迟后触发。
   - 重新武装 `reset_delayed` 替换截止时间（只触发一次）。
7. **负载下分离**：tick 源设为 1 ms，`poll_waiter` 运行中，`bulk_writer_flood` 发出洪水，然后立即卸载。应该干净。
8. **WITNESS 通过**：以上每个测试，`dmesg` 中没有 `WITNESS` 警告。
9. **lockstat 通过**：在 `lockstat -s 5` 下运行测试套件来测量锁争用。任务队列的内部互斥锁应该只短暂出现。

每个测试应该通过。如果任何失败，原因几乎肯定是第3阶段和第4阶段之间引入的回归，不是预先存在的问题；第1-3阶段在开始第4阶段之前各自独立验证。

### 保持文档同步

第14章应反映在三个地方：

- `myfirst.c` 的文件顶部注释。更新"锁定策略"块以提及任务队列。
- `LOCKING.md`。按前一小节更新。
- `examples/part-03/ch14-taskqueues-and-deferred-work/` 下的每章 `README.md`。描述每个阶段的交付物及如何构建它们。

更新文档感觉像是开销。它不是。明年的读者（通常是你未来的自己）依赖文档来重建设计。现在写，当设计还清晰时，比以后写成本低一个数量级。

### 最终审计

在关闭第4阶段之前，运行一个简短的审计。

- [ ] 每个调用排空是否在 每个 任务排空之前发生？
- [ ] 每个任务排空是否在 `seldrain` 之前发生？
- [ ] `taskqueue_free` 是否在所有任务排空之后发生？
- [ ] attach 失败路径是否在后续初始化步骤失败时 `taskqueue_free` 队列？
- [ ] 每个入队调用点是否指向正确的 taskqueue 指针（私有的，而非共享的）？
- [ ] 每个排空调用点是否与其入队调用点匹配（同一 taskqueue、同一任务）？
- [ ] 每个任务回调是否不包含"我每次入队恰好运行一次"的假设？
- [ ] 每个任务回调是否不包含"我进入时持有驱动锁"的假设？
- [ ] `LOCKING.md` 是否列出了每个任务及其回调、生命周期和入队路径？
- [ ] 版本字符串是否反映了新阶段？

通过此审计的驱动，加上第13章第7节的审计，是一个你可以有信心交给其他工程师的驱动。

### 第8节总结

第4阶段是整合。驱动代码被组织好了。`LOCKING.md` 是最新的。版本字符串反映了新能力。完整的回归套件在调试内核下通过。审计检查表是干净的。

`myfirst` 驱动走了很长的路。它在第10章开始时是一个单次打开的字符设备，通过循环缓冲区移动字节。第11章给了它并发访问。第12章给了它有界阻塞、cv 通道和 sx 保护的配置。第13章给了它 callout 用于周期性和看门狗工作。第14章给了它延迟工作，这是边缘上下文和线程上下文之间的桥梁，也是驱动最终将面对真实硬件中断所缺少的那一块。

本章的其余部分稍微扩展了视野。附加主题节以介绍级别介绍了 `epoch(9)`、组任务队列和每 CPU 任务队列。动手实验整合了第3节到第8节的材料。挑战练习延伸读者。故障排除参考将常见问题汇集在一个地方。然后是收尾和通往第15章的桥梁。



## Additional Topics: `epoch(9)`, Grouptaskqueues, and Per-CPU Taskqueues

第14章的主体教授了典型驱动需要的 `taskqueue(9)` 模式。三个相邻主题值得为最终会编写或阅读网络驱动的读者提及，或驱动规模增长到简单私有任务队列不够时。每个主题在"知道何时使用它"的级别介绍。完整的机制属于后续章节，特别是第六部分覆盖网络驱动的章节（第28章）。

### `epoch(9)` 一页概述

`epoch(9)` 是一种无锁读同步机制。其目的是允许多个读者并发遍历共享数据结构，无需获取任何排他锁，同时保证数据结构不会在它们脚下消失。

其形态是这样的。读取共享数据的代码用 `epoch_enter(epoch)` 进入一个 "epoch 段"，用 `epoch_exit(epoch)` 离开。在段内部，读者可以自由解引用指针。想要更改或释放共享对象的写入者不直接这样做；相反，它们要么调用 `epoch_wait(epoch)` 阻塞直到所有当前读者离开了 epoch 段，要么通过 `epoch_call(epoch, cb, ctx)` 注册一个回调，在所有当前读者离开后异步运行。

好处是可扩展性。读者不支付原子操作成本；它们只在进入和退出时记录线程本地状态。写入者支付同步成本，但写入相比读取很少，所以分摊成本很低。对于被多个线程遍历且偶尔更改的数据结构，`epoch(9)` 大幅优于读写锁。

代价是纪律。epoch 段内部的代码不能睡眠、不能获取可睡眠锁、不能调用可能做这两者的函数。使用 `epoch_wait` 的写入者阻塞直到所有当前读者离开，这意味着写入者可能在等待很多读者。

网络驱动大量使用 `epoch(9)`。`net_epoch_preempt` epoch 保护网络状态的读取（ifnet 列表、路由条目、接口标志）。包输入路径进入 epoch、遍历状态、退出 epoch。想要移除接口的写入者通过 `NET_EPOCH_CALL` 延迟释放，释放在类似任务队列的机制上在每个读者完成后发生。

对于任务队列的连接：当任务用 `NET_TASK_INIT` 而不是 `TASK_INIT` 初始化时，任务队列在 `net_epoch_preempt` epoch 内运行回调。任务回调因此可以遍历网络状态而无需显式进入 epoch。来自 `/usr/src/sys/kern/subr_taskqueue.c` 的实现：

```c
if (!in_net_epoch && TASK_IS_NET(task)) {
        in_net_epoch = true;
        NET_EPOCH_ENTER(et);
} else if (in_net_epoch && !TASK_IS_NET(task)) {
        NET_EPOCH_EXIT(et);
        in_net_epoch = false;
}
task->ta_func(task->ta_context, pending);
```

taskqueue 分发器注意到 `TASK_NETWORK` 标志并根据需要在回调周围进入或退出 epoch。连续的网络任务共享单个 epoch 进入，这是框架免费提供的一个小优化。

对于 `myfirst`，这不相关。驱动不触碰网络状态。但如果你后来编写网络驱动或阅读网络驱动代码，`NET_TASK_INIT` 和 `TASK_IS_NET` 是告诉你任务感知 epoch 的宏。

### 组任务队列一页概述

组任务队列是任务队列的可扩展泛化。基本想法：不是单一队列配单一（或小型）工作线程池，而是将任务分配到许多每 CPU 队列，每个由自己的工作线程服务。"组任务"是绑定到其中一个队列的任务。

头文件是 `/usr/src/sys/sys/gtaskqueue.h`：

```c
#define GROUPTASK_INIT(gtask, priority, func, context)   \
    GTASK_INIT(&(gtask)->gt_task, 0, priority, func, context)

#define GROUPTASK_ENQUEUE(gtask)                         \
    grouptaskqueue_enqueue((gtask)->gt_taskqueue, &(gtask)->gt_task)

void    taskqgroup_attach(struct taskqgroup *qgroup,
            struct grouptask *grptask, void *uniq, device_t dev,
            struct resource *irq, const char *name);
int     taskqgroup_attach_cpu(struct taskqgroup *qgroup,
            struct grouptask *grptask, void *uniq, int cpu, device_t dev,
            struct resource *irq, const char *name);
void    taskqgroup_detach(struct taskqgroup *qgroup, struct grouptask *gtask);
```

使用组任务的驱动在 attach 时做以下事情：

1. 用 `GROUPTASK_INIT` 初始化每个组任务。
2. 用 `taskqgroup_attach` 或 `taskqgroup_attach_cpu` 将每个组任务附加到 `taskqgroup`。附加将组任务分配到特定的每 CPU 队列和工作线程。
3. 在事件时，用 `GROUPTASK_ENQUEUE` 入队。
4. 在分离时，`taskqgroup_detach` 解除组任务的关联。

为什么使用组任务而不是普通任务？两个原因。

第一，**随 CPU 数量的可扩展性**。单线程任务队列在多个 CPU 上并发入队时是瓶颈。任务队列的内部互斥锁成为争用点。有每 CPU 队列的组任务队列让每个 CPU 在自己的队列上入队而无跨 CPU 争用。

第二，**缓存局部性**。当中断在 CPU N 上触发并将在 CPU N 上绑定的组任务入队时，任务在看到中断的同一 CPU 上运行。任务的数据已经在那个 CPU 的缓存中。对于高速率网络驱动这是实质性的性能提升。

代价是复杂性。组任务队列需要更多设置、更多拆除、更多考虑任务属于哪个队列。对大多数驱动这个代价不值得。对于每秒处理数百万包的高端以太网驱动，代价物有所值。

`myfirst` 不使用组任务。它不会受益。我们提到它们以便当你阅读 `/usr/src/sys/dev/wg/if_wg.c` 或 `/usr/src/sys/net/iflib.c` 这样的驱动时，宏看起来很熟悉。

### 每 CPU 任务队列一页概述

每 CPU 任务队列是组任务队列想法的简单版本：每个 CPU 一个任务队列，每个有自己的工作线程。驱动创建 N 个任务队列（每个 CPU 一个），用 `taskqueue_start_threads_cpuset` 将每个绑定到特定 CPU，并根据驱动想要的任何局部性规则将任务分派到适当的队列。

关键原语是 `taskqueue_start_threads_cpuset`：

```c
int taskqueue_start_threads_cpuset(struct taskqueue **tqp, int count,
    int pri, cpuset_t *mask, const char *name, ...);
```

它类似于 `taskqueue_start_threads` 但有一个描述线程可以运行在哪些 CPU 上的 `cpuset_t`。对于单 CPU 绑定，掩码恰好设置一位。对于多 CPU 灵活性，掩码有多个位。

使用每 CPU 任务队列的驱动通常维护一个按 CPU 索引的任务队列指针数组：

```c
struct my_softc {
        struct taskqueue *per_cpu_tq[MAXCPU];
        ...
};

for (int i = 0; i < mp_ncpus; i++) {
        CPU_SETOF(i, &mask);
        sc->per_cpu_tq[i] = taskqueue_create("per_cpu", M_WAITOK,
            taskqueue_thread_enqueue, &sc->per_cpu_tq[i]);
        taskqueue_start_threads_cpuset(&sc->per_cpu_tq[i], 1, PWAIT,
            &mask, "%s cpu%d", device_get_nameunit(sc->dev), i);
}
```

在入队时，选择对应当前 CPU 的队列：

```c
int cpu = curcpu;
taskqueue_enqueue(sc->per_cpu_tq[cpu], &task);
```

好处与组任务相同，没有组任务框架：工作留在产生它的 CPU 上，CPU 本地争用被消除，缓存保持热。代价是驱动管理自己的每 CPU 数据结构。

对于 `myfirst` 这是过度杀伤。对于事件率超过每秒数万个事件的驱动，每 CPU 任务队列值得考虑。组任务队列更通用，通常在可扩展性故事重要时更受青睐；每 CPU 任务队列是更轻量的替代方案。

### 何时使用哪种工具

一个简短的决策树。

- **低速率，线程上下文，共享队列即可**：使用 `taskqueue_thread`。最简单。
- **低速率，线程上下文，隔离重要**：私有任务队列，用 `taskqueue_create` 和 `taskqueue_start_threads`。`myfirst` 使用的。
- **高速率，争用是瓶颈**：每 CPU 任务队列或组任务队列。从每 CPU 开始；如果需要额外可扩展性特性则使用组任务。
- **网络路径数据**：`NET_TASK_INIT` 和组任务队列，遵循网络驱动中的模式。
- **过滤器中断上下文，必须不睡眠入队**：`taskqueue_create_fast` 或 `taskqueue_fast`，因为过滤器中断不能使用睡眠互斥锁。

你编写或阅读的大多数驱动将适合前两行之一。其余是需要其章节逐步讲解的专门情况。

### 附加主题总结

`epoch(9)`、组任务队列和每 CPU 任务队列是任务队列的扩展故事。它们共享与基本 API 相同的心智模型：从生产者入队、在工作线程上调度、遵守锁纪律、在关闭时排空。区别在于有多少队列以及任务如何在它们之间调度。对于大多数驱动程序基本 API 足够；这些高级变体存在于不够时。

本章现在移动到动手实验。



## 动手实验

实验将本章材料整合为四个实践练习。每个实验使用你通过第 1 到第 4 阶段演进的驱动程序，加上 `examples/part-03/ch14-taskqueues-and-deferred-work/labs/` 下提供的一些小型用户空间辅助工具。

每个实验分配一个会话。如果时间有限，实验 1 和 2 最重要；实验 3 和 4 值得做但更复杂。

### 实验 1：观察任务队列工作线程

**目标。** 确认你的驱动程序的私有任务队列有一个工作线程，线程在没有工作时睡眠，线程在工作入队时唤醒并运行回调。

**设置。** 加载第 2 阶段驱动程序（或第 4 阶段，都可以）。确保没有其他使用任务队列的进程在冲击系统；系统越安静，观察越容易。

**步骤。**

1. 运行 `procstat -t | grep myfirst`。记录显示的 PID 和 TID。线程应该处于 `sleep` 状态。
2. 运行 `sysctl dev.myfirst.0.heartbeat_interval_ms=1000`。等待几秒。
3. 再次运行 `procstat -t | grep myfirst`。线程可能在心跳触发期间短暂显示 `run` 状态；大部分时间仍为 `sleep` 因为心跳不入队任务。确认你看到的是这样。注意心跳运行在 callout 线程中，不是驱动程序的任务队列线程。
4. 运行 `sysctl dev.myfirst.0.tick_source_interval_ms=100`。等待几秒。
5. 再次运行 `procstat -t | grep myfirst`。线程现在应该在 `sleep` 和 `run` 之间震荡，因为 tick 源每秒入队任务十次。
6. 用 `sysctl dev.myfirst.0.tick_source_interval_ms=0` 停止 tick 源。确认线程返回永久 `sleep`。
7. 停止心跳。卸载驱动程序。确认线程从 `procstat -t` 消失。

**预期结果。** 你直接观察了线程的生命周期：在 attach 时创建，空闲时睡眠，调度时运行，在 detach 时销毁。这个观察比产生它的两页解释更有价值。

### 实验 2：在负载下测量合并

**目标。** 产生足够压力触发合并的工作负载，然后使用 `selwake_pending_drops` sysctl 测量合并率。

**设置。** 加载第 4 阶段驱动程序。按第 3 节编译 `poll_waiter`。

**步骤。**

1. 在一个终端启动 `poll_waiter`：`./poll_waiter > /dev/null`。重定向到 `/dev/null` 防止终端成为瓶颈。
2. 在第二个终端，将 tick 源设为快速率：`sysctl dev.myfirst.0.tick_source_interval_ms=1`。
3. 等待十秒。
4. 读取 `sysctl dev.myfirst.0.stats.selwake_pending_drops`。记录值。
5. 再等十秒并再次读取。计算每秒速率。
6. 增加 tick 源速率看合并是否增加：最小 tick 源间隔是 1 ms，但你可以结合 bulk_writer_flood sysctl 产生更突发性的负载：
   ```text
   # for i in $(seq 1 100); do sysctl dev.myfirst.0.bulk_writer_flood=1000; done
   ```
7. 在冲击后读取 `selwake_pending_drops`。

**预期结果。** 数字随时间增长，在更突发负载下更多，在稳定负载下更少。如果即使在激进负载下数字保持为零，任务队列线程足够快跟上；这是好状态，不是 bug。

**变体。** 用调试内核（启用 `WITNESS`）运行相同工作负载，观察 `dmesg` 是否显示任何 `WITNESS` 警告。不应该有。

### 实验 3：验证分离顺序

**目标。** 确认分离路径在释放任务触及的状态之前正确排空任务。故意引入第 7 节的 bug（`seldrain` 后任务排空）并观察竞争。

**设置。** 从第 4 阶段开始。制作 `myfirst.c` 的工作副本。

**步骤。**

1. 在你的工作副本中，重新排序 `myfirst_detach` 中的排空使 `seldrain` 在 `taskqueue_drain` 之前：
   ```c
   /* 错误顺序： */
   MYFIRST_CO_DRAIN(&sc->heartbeat_co);
   MYFIRST_CO_DRAIN(&sc->watchdog_co);
   MYFIRST_CO_DRAIN(&sc->tick_source_co);
   seldrain(&sc->rsel);
   seldrain(&sc->wsel);
   taskqueue_drain(sc->tq, &sc->selwake_task);
   /* ... 其余 ... */
   ```
   这是故意错误的。
2. 用错误顺序重新构建。
3. 加载驱动程序。以 1 ms 启用 tick 源。运行 `poll_waiter`。
4. 数据流动几秒后，卸载驱动程序：`kldunload myfirst`。
5. 大多数时候卸载成功。偶尔，特别是在负载下，内核崩溃。崩溃栈通常包括从 `myfirst_selwake_task` 调用的 `selwakeup`，在 `seldrain` 运行后。
6. 恢复正确顺序。重新构建。运行相同压力并多次重复卸载。
7. 确认正确顺序从不崩溃。

**预期结果。** 你直接体验了竞争。教训是"通常工作"不是"工作"。正确顺序是你保持的不变量，即使错误顺序在随意测试中似乎工作。

**注意。** 在生产内核上崩溃可能不发生；内存损坏可以隐藏直到其他东西崩溃。总是在启用 `INVARIANTS` 和 `WITNESS` 的调试内核上运行此类实验。

### 实验 4：合并与自适应批处理

**目标。** 构建一个使用 `pending` 参数驱动自适应批处理的小型修改，并将其行为与第 3 阶段的固定批次 bulk_writer_task 比较。

**设置。** 从第 4 阶段开始。

**步骤。**

1. 向驱动程序添加新任务：`adaptive_writer_task`。其回调写入 `pending` 字节（上限 64）到缓冲区。使用第 5 节的模式。
2. 添加按需入队 `adaptive_writer_task` 的 sysctl：
   ```c
   static int
   myfirst_sysctl_adaptive_enqueue(SYSCTL_HANDLER_ARGS)
   {
           struct myfirst_softc *sc = arg1;
           int n = 0, i, error;

           error = sysctl_handle_int(oidp, &n, 0, req);
           if (error || req->newptr == NULL)
                   return (error);
           for (i = 0; i < n; i++)
                   taskqueue_enqueue(sc->tq, &sc->adaptive_writer_task);
           return (0);
   }
   ```
3. 在 attach 中初始化任务，在 detach 中排空。
4. 重新构建，加载。
5. 通过 sysctl 发出 1000 次入队：`sysctl dev.myfirst.0.adaptive_enqueue=1000`。
6. 读取 `sysctl dev.myfirst.0.stats.bytes_written`。观察写入了多少字节。
7. 与第 3 阶段的 `bulk_writer_flood` 和 `bulk_writer_batch=1` 比较。固定批次会写入 1 字节（合并为一次触发）。自适应批次写入任何 `pending` 是多少，上限 64。

**预期结果。** 自适应任务在突发负载下写入更多字节，因为它使用内核已计算的合并信息。对于每次事件工作应该与事件计数成比例的工作负载，此模式优于固定批次大小。

**变体。** 添加记录见过的最大 `pending` 值的计数器。作为 sysctl 暴露。在压力下，你会看到峰值 pending 随负载增加而增长。



## 挑战练习

挑战是可选的延伸。它们将本章建立的模式推到正文未覆盖的领域。慢慢来；它们旨在巩固理解，不是引入新材料。

### 挑战 1：每文件句柄任务

修改驱动程序使每个打开的文件句柄有自己的任务。任务的工作，入队时，是发出标识句柄的日志行。编写同时入队每个句柄任务的 sysctl。

提示：
- `myfirst_open` 中分配的 `struct myfirst_fh` 是每句柄任务的自然归宿。
- 在 `myfirst_open` 中 `malloc` 后初始化任务。
- 在 `myfirst_fh_dtor` 中 `free` 前排空任务。
- 入队"每个句柄的任务"需要打开句柄列表。`devfs_set_cdevpriv` 不维护此列表；你必须在 softc 中构建一个，用互斥锁保护。

预期结果：演示比驱动程序更细粒度的任务所有权。挑战测试你对生命周期顺序的理解。

### 挑战 2：两级任务流水线

添加两个任务的流水线。任务 A 从 `write(2)` 处理程序接收数据，转换它（为简单起见，将每个字节大写），并入队任务 B。任务 B 将转换后的数据写入辅助缓冲区并通知等待者。

提示：
- 转换工作发生在任务回调中，在线程上下文中。`write(2)` 处理程序不应该阻塞等待转换。
- 你需要一个小型的待处理转换队列，用互斥锁保护。
- 任务 A 从队列拉取，转换，并带每项状态入队任务 B。或者，任务 B 在每次 A 入队时运行一次并处理队列中的任何东西。

预期结果：任务队列如何形成流水线的心智模型，每个阶段在自己的调用中运行。这是复杂驱动程序分割工作的方式。

### 挑战 3：优先级驱动的任务排序

向驱动程序添加两个不同优先级的任务。`urgent_task` 优先级为 10 并打印 "URGENT"。`normal_task` 优先级为 0 并打印 "normal"。编写入队两个任务的 sysctl 处理程序，normal 先，urgent 后。

预期结果：`dmesg` 输出显示 `URGENT` 在 `normal` 之前，确认优先级在队列内覆盖入队顺序。

### 挑战 4：阻塞重配置

实现使用 `taskqueue_block` 和 `taskqueue_quiesce` 的重配置路径。路径应该：

1. 阻塞任务队列。
2. 静默（等待运行任务完成）。
3. 执行重配置（比如，调整循环缓冲区大小）。
4. 解除阻塞。

用 `dtrace` 验证在重配置窗口期间没有任务运行。

预期结果：体验 `taskqueue_block` 和 `taskqueue_quiesce`，以及理解这些原语何时适用。

### 挑战 5：多线程任务队列

修改第 4 阶段使用多线程私有任务队列（比如，四个工作线程而不是一个）。运行实验 2 的合并测试。观察有什么变化。

预期结果：在负载下，合并率下降因为多个工作线程更快排空队列。在很轻负载下，没有可见变化。挑战展示任务队列配置如何在不同工作负载间权衡。

### 挑战 6：使用超时任务实现看门狗

用 `timeout_task` 而不是普通 callout 重新实现第 13 章看门狗。每次看门狗触发用配置的间隔重新入队自己。"kick" 操作（另一个 sysctl，也许是 `watchdog_kick`）取消并重新入队超时任务以重置定时器。

预期结果：理解 `timeout_task` 原语如何能为周期性工作替换 callout，以及何时每个更可取。（答案：当工作需要线程上下文时用 timeout_task；否则用 callout。）

### 挑战 7：加载真实驱动程序并阅读其代码

选择第 6 节列出的驱动程序之一（`/usr/src/sys/dev/ale/if_ale.c`、`/usr/src/sys/dev/age/if_age.c`、`/usr/src/sys/dev/bge/if_bge.c` 或 `/usr/src/sys/dev/iwm/if_iwm.c`）。阅读其任务队列使用。识别：

- 驱动程序拥有哪些任务。
- 每个任务在哪里初始化。
- 每个任务在哪里入队。
- 每个任务在哪里排空。
- 驱动程序使用 `taskqueue_create` 还是 `taskqueue_create_fast`。
- 驱动程序使用什么线程优先级。

编写驱动程序如何使用任务队列 API 的简短总结（一页左右）。保留作为参考。

预期结果：阅读真实驱动程序将模式识别从抽象转为具体。在用一个驱动程序做一次后，阅读下一个会显著更快。



## 故障排除参考

症状和补救措施的扁平参考列表，用于 bug 出现时需要快速答案的时刻。将此参考与每节内的常见错误列表配对；它们一起覆盖大多数真实问题。

### 任务从不运行

- **你是否在 `taskqueue_create` 后调用了 `taskqueue_start_threads`？** 无线程，队列接受入队但从不分发它们。
- **入队时任务队列指针是否为 `NULL`？** 检查 attach 路径；如果你没有检查返回值，`taskqueue_create` 可能静默失败。
- **入队触发时驱动程序的 `is_attached` 是否为 false？** 某些代码路径（如第 13 章的 callout 回调）如果 `is_attached` 为 false 会提前退出；如果退出发生在入队前，任务不运行。
- **任务队列是否通过 `taskqueue_block` 阻塞？** 如果是，它接受入队但不分发。解除阻塞。

### 你预期一次但任务运行两次

- **任务是否自我重新入队？** 调用 `taskqueue_enqueue` 自己的任务回调会无限循环，除非回调在某些条件下提前退出。
- **是否有不同代码路径也在入队？** 检查任务的每个 `taskqueue_enqueue` 调用点。两个来源入队会在某些时序下产生预期的双重运行。

### 你预期两次或更多但任务运行一次

- **你的入队是否合并了？** 入队已待处理规则折叠冗余提交。如果你需要精确的每次事件语义，使用单独任务或每事件队列。
- **`pending` 参数是否报告为大于一？** 如果是，框架合并了。

### 任务回调中内核崩溃

- **回调是否访问已释放状态？** 任务回调中崩溃的最常见原因是释放后使用。检查分离顺序：每个入队生产者必须在任务排空前排空；任务触及的状态必须在任务排空前不释放。
- **回调是否持有不应该持有的锁？** `WITNESS` 捕获大多数这些。在调试内核下运行并阅读 `dmesg`。
- **回调是否在持有驱动程序互斥锁时调用 `selwakeup`？** 不要。`selwakeup` 获取自己的锁，不应该在持有不相关的驱动程序锁时调用。

### 分离挂起

- **`taskqueue_drain` 是否在等待无法完成的任务？** 用 `procstat -kk` 检查任务队列工作线程的状态。如果它在等待分离路径持有的东西，设计有循环。
- **`taskqueue_free` 是否在等待仍在入队的任务？** 检查 `is_attached`：如果 callout 仍在运行并仍在入队，排空不会终止。确保 callout 先排空。

### `kldunload` 立即返回 EBUSY

- **是否有文件描述符仍打开？** `myfirst_detach` 中的分离路径如果 `active_fhs > 0` 会以 `EBUSY` 拒绝。关闭任何打开的描述符并重试。

### 合并计数保持为零

- **工作负载是否太轻？** 合并只在生产者超越消费者时发生。在轻负载机器上这很少发生。
- **你的测量是否正确？** 合并在回调中计数，不是在入队路径。检查你的计数器逻辑。
- **任务队列是否多线程？** 更多线程意味着更快消费，更少合并。

### 私有任务队列的线程不出现在 `procstat -t`

- **`taskqueue_start_threads` 是否返回零？** 如果返回错误，线程未创建。检查返回值。
- **驱动程序是否实际加载？** `kldstat` 确认。
- **线程名称是否与预期不同？** `taskqueue_start_threads` 的格式字符串控制名称；确保你在 grep 正确的东西。

### 任务和驱动程序互斥锁间死锁

- **任务回调是否获取不同线程在等待任务时持有的锁？** 那是教科书式死锁形态。通过将任务入队移到锁持有部分之外，或重构等待使其不阻塞任务来打破它。

### `taskqueue_enqueue` 以 `EEXIST` 失败

- **你传递了 `TASKQUEUE_FAIL_IF_PENDING` 且任务已待处理。** 失败是有意的；检查标志是否是你想要的。

### `taskqueue_enqueue_timeout` 似乎不触发

- **任务队列是否阻塞？** 阻塞队列也不分发超时任务。
- **tick 计数是否合理？** 零 tick 计数立即触发，但非整数毫秒到 tick 转换可能产生意外长延迟。用 `(ms * hz + 999) / 1000` 做向上取整。
- **超时任务是否已被 `taskqueue_cancel_timeout` 取消？** 如果是，重新入队。

### 重新设置 `timeout_task` 不替换截止时间

- **每次 `taskqueue_enqueue_timeout` 替换待处理的截止时间。** 如果你的驱动程序多次调用它但只有第一次似乎生效，你可能有顺序问题：你确定后续调用发生了吗？

### WITNESS 抱怨涉及 `tq_mutex` 的锁顺序

- **任务队列的内部互斥锁正在进入你的驱动程序锁顺序。** 通常因为任务回调获取驱动程序锁，而其他代码路径先获取该驱动程序锁然后入队。
- **解决方案通常是先入队再获取驱动程序锁，或重构代码使两个锁从不在同一线程以错误顺序持有。**

### `procstat -kk` 显示任务队列线程在锁上睡眠

- **任务回调阻塞在可睡眠锁上。** 从等待通道识别锁。检查该锁的持有者是否也在等待什么；如果是，你有依赖链。

### 任务回调慢

- **用 `dtrace` 分析。** 第 7 节的单行命令测量回调持续时间。
- **回调是否在长操作期间持有锁？** 将长操作移到锁外。
- **回调是否执行同步 IO？** 那属于 `read(2)` / `write(2)` / `ioctl(2)` 处理程序，不是任务队列回调，除非 IO 确实是任务的重点。

### 引导期间任务队列死锁

- **你是否从 `SI_SUB_TASKQ` 之前运行的 `SI_SUB` 入队任务？** 预定义任务队列在 `SI_SUB_TASKQ` 初始化。更早的 `SI_SUB` 处理程序不能入队到它们。



## 收尾

第 14 章深入讲授了一个原语。原语是 `taskqueue(9)`。其目的是将工作从不能做工作的上下文移到能做的上下文。其 API 很小：初始化任务，入队，排空，完成后释放队列。心智模型同样小：边缘上下文检测，线程上下文任务行动。

`myfirst` 驱动程序优雅地吸收了新机制，因为每个之前的章节都准备了脚手架。第 11 章给了它并发。第 12 章给了它 cv 通道和 sx 配置。第 13 章给了它 callout 和分离时排空纪律。第 14 章添加任务作为相同形态的第五个原语：在 attach 中初始化，在 detach 中排空，遵守已建立的锁规则，并与之前的组合。驱动程序现在版本 `0.8-taskqueues`，有三个任务和一个私有任务队列，在负载下干净拆除。

在这些具体更改之下，本章提出了几个更大的观点。每个简短回顾。

**延迟工作是边缘上下文和线程上下文之间的桥梁。** Callout、中断过滤器和 epoch 段落都面临相同约束：它们不能做需要睡眠或可睡眠锁获取的工作。任务队列通过接受边缘上下文的小提交并在线程上运行真正工作来统一解决问题。

**任务队列框架处理后勤所以你不必。** 分配、内部队列锁定、分发、合并、取消、排空：每一个都由框架处理。你的驱动程序提供回调和入队点。其余是 attach 中的简短设置和 detach 中的简短拆除。

**合并是特性，不是 bug。** 任务将冗余入队合并为单次触发，其 `pending` 参数携带计数。这让事件突发折叠为单次回调调用，几乎总是你想要的性能。需要每次事件调用的设计需要每事件任务或每事件队列，不是多次入队的一个任务。

**分离顺序是本章添加的最大新纪律。** 先 callout，第二任务，第三 selinfo，最后 taskqueue_free。违反顺序是竞争，可能在安静测试中不出现而在负载下出现。`LOCKING.md` 文档是你写下顺序的地方；遵循它是你避免竞争的地方。

**真实驱动程序都使用相同的少数模式。** 边缘上下文延迟日志；过滤器加任务中断分割；异步 `copyin`/`copyout`；重试并退避；延迟拆除；调度滚动；重配置期间阻塞。每一个都是边缘检测、任务行动形态的变体。阅读 `/usr/src/sys/dev/` 是深度吸收模式的最快方式。

**任务队列故事可扩展。** `epoch(9)`、组任务队列和每 CPU 任务队列处理简单私有任务队列不能的可扩展性情况。它们共享基本 API 的心智模型；区别在于队列数量、分发策略和工作线程周围的脚手架。对大多数驱动程序基本 API 足够；对于高端情况，高级变体在你需要时存在。

### 第 15 章前的反思

你开始第 14 章时带着可以通过时间行动（callout）但其行动受限于 callout 允许内容的驱动程序。你离开时带着可以通过时间行动并通过移交给工作线程的工作来行动。这两者组合覆盖驱动程序需要的几乎每种延迟行动。第三部分是安全协调生产者和消费者的同步，这部分是第 15 章发展的。

心智模型是累积的。第 12 章引入 cvs、mutexes 和 sx。第 13 章引入 callout 和不可睡眠上下文。第 14 章引入任务和延迟到线程上下文。第 15 章将引入高级协调原语（信号量、`cv_timedwait_sig`、跨组件握手）将早期部分组合成更丰富模式。每章添加小原语及其配套纪律。

驱动程序的累积形态在 `LOCKING.md` 中可见。第 10 章驱动程序没有 `LOCKING.md`。第 11 章驱动程序有单段落。第 14 章驱动程序有多页文档，有 mutex、cvs、sx、callout 和 tasks 的节，加上命名每个排空步骤正确顺序的分离顺序节。那份文档是你带进每个未来章节的人工品。当第 15 章添加信号量，`LOCKING.md` 增长信号量节。当第四部分添加中断，它增长中断节。驱动程序的生命周期是其 `LOCKING.md`。

### 第二个反思：纪律

本章希望你内化的习惯高于其他：驱动程序中的每个新原语在 `LOCKING.md` 中获得条目，在 attach/detach 中有析构对，在记录的分离顺序中有位置。跳过任何那些创造等待发生的 bug。第一次将驱动程序交给别人时纪律就会回报。

反之亦然：每次阅读别人的驱动程序时，先看他们的 `LOCKING.md`。如果缺失，阅读 attach 和 detach 函数从代码重建顺序。如果你看到 attach 中的原语在 detach 中没有对应排空，那是 bug。如果你看到排空没有明确前置，那可能是顺序错误。写作和阅读的纪律相同。

### 关于简单性的简短说明

任务队列看起来简单。确实。API 很小，模式规则，习语跨驱动程序转移。简单性是刻意的；这是使 API 在实践中可用的东西。同样的简单性也使规则不可协商：跳过的规则产生难以调试的竞争。遵循纪律，任务队列对你保持简单。即兴发挥，它们不会。

### 如果你卡住了该怎么办

如果驱动程序中的某些东西不按预期行为，按顺序检查第 7 节的故障排除参考。检查匹配你症状的第一项。如果没有匹配项，重读第 4 节（设置和清理）并根据 `LOCKING.md` 审计你的分离顺序。如果顺序正确，用 `dtrace` 跟踪入队路径看预期事件是否发生。

如果驱动程序崩溃，在崩溃转储上用 `gdb`。`bt` 显示栈。包含你任务回调的栈是好的起点；与第 7 节的模式比较。

如果一切都失败，重读你在前一节为挑战 7 选的真实驱动程序。有时在你自己代码中似乎困惑的模式在别人写的驱动程序中自然读取。模式是通用的；你阅读的驱动程序不特殊。



## 通往第15章的桥梁

第 15 章标题为*更多同步：条件、信号量和协调*。其范围是高级协调原语，将你现在拥有的 mutexes、cvs、sx locks、callouts 和 tasks 组合成更复杂模式。

第 14 章以四种具体方式准备了基础。

第一，**你现在在驱动程序中有工作的生产者/消费者对**。`tick_source` callout（和其他第 14 章入队点）是生产者；任务队列线程是消费者。第 12 章的 cv 通道是另一个生产者/消费者对：`write(2)` 生产，`read(2)` 消费。第 15 章推广模式并添加处理更复杂版本的相同形态的原语（计数信号量、可中断 cv 等待）。

第二，**你知道分离时排空的纪律**。你添加的每个原语都有对应排空。第 15 章引入信号量，有它们自己的"排空"模式（释放所有等待者，然后销毁），纪律直接转移。

第三，**你知道如何思考上下文边界**。Callout 上下文、任务上下文、syscall 上下文：每个有自己的规则，你的驱动程序设计尊重它们。第 15 章添加信号可中断等待，增加用户交互上下文到混合。"我在哪个上下文，我在这里能做什么"的习惯转移。

第四，**你的 `LOCKING.md` 处于"每个原语一节，末尾加顺序节"的节奏**。第 15 章将添加信号量节和可能的协调节。结构已建立；只有内容变化。

第 15 章将覆盖的具体主题：

- 通过 `sema(9)` API 的计数信号量和二进制信号量。
- `cv_timedwait_sig` 和信号可中断阻塞。
- 通过完全一般形式的 `sx(9)` 的读者-写者模式。
- 协调定时器、任务和用户线程的跨组件握手。
- 用于无锁协调的状态标志和内存屏障（介绍级别）。
- 并发测试工具：压力脚本、故障注入、竞争重放。

你不需要提前阅读。第 14 章是足够准备。带着你的第 14 章第 4 阶段的 `myfirst` 驱动程序、你的 `LOCKING.md`、你启用 `WITNESS` 的内核和你的测试套件。第 15 章从第 14 章结束处开始。

一个简短的结束反思。你开始第三部分时的驱动程序一次理解一个 syscall。你现在拥有的驱动程序有三个 callout、三个任务、两个 cvs、两个锁和完整的分离顺序。它处理并发读者和写者、定时事件、跨上下文边界的延迟工作、合并的事件突发和负载下的干净拆除。它有记录的锁故事和验证的回归套件。它开始看起来像真实的 FreeBSD 驱动程序。

第 15 章通过添加让部分组合成更丰富模式的协调原语结束第三部分。然后第四部分开始：硬件和平台级集成。真实中断。真实内存映射寄存器。可能失败、行为不当或拒绝合作的真实硬件。你通过第三部分建立的纪律是将带你通过的东西。

在继续之前花点时间。从第 13 章到第 14 章的跳跃是质变的：驱动程序获得了将工作延迟到线程上下文的能力，你沿途学习的模式（分离顺序、合并、边缘/线程心智模型）是你将在随后的每章中重用的模式。从这里，第 15 章巩固同步故事，第四部分开始硬件故事。你做的工作没有丢失；它在复合。

### 关于内核形态的最后旁白

第 15 章前的最后一个想法。你现在遇到了内核的五个同步和延迟原语：互斥锁、条件变量、sx lock、callout 和任务队列。每个存在是因为更早、更简单的原语不能解决相同问题。互斥锁不能表达"等待条件"；那是 cvs 做的。可睡眠互斥锁不能在睡眠期间持有；那是 sx locks 允许的。Callout 不能运行需要睡眠的工作；那是任务队列允许的。

模式在内核中可识别。每个同步原语存在是因为早期原语的特定差距。阅读内核代码时，你经常可以通过问其邻居不能做什么来猜测为什么选择特定原语。文件描述符不能遭受释放后使用，因为引用计数原语防止它。网络包不能在读者遍历列表时释放，因为 epoch 原语防止它。任务不能在分离后运行，因为排空原语防止它。

内核是这些原语的目录，每个刻意的，每个对特定问题类的响应。你的驱动程序随着增长积累自己的目录。第 14 章添加了列表中的一项。第 15 章添加几项。第四部分开始面向硬件的目录。从这里开始原语倍增，但纪律形态不变。定义问题，选择原语，干净初始化和拆除，记录顺序，在负载下验证。

那就是工艺。本书其余部分带你通过它。


## 参考：预生产任务队列审计

在将使用任务队列的驱动从开发提升到生产之前要执行的简短审计。每个项目是一个问题；每个应该能够自信地回答。

### 任务清单

- [ ] 我是否在 `LOCKING.md` 中列出了驱动拥有的每个任务？
- [ ] 对于每个任务，我是否命名了其回调函数？
- [ ] 对于每个任务，我是否文档记录了其生命周期（attach 中初始化，分离中排空）？
- [ ] 对于每个任务，我是否文档记录了其触发器（什么导致它被入队）？
- [ ] 对于每个任务，我是否文档记录了它是自重新入队还是每次外部触发运行一次？
- [ ] 对于每个超时任务，我是否命名了它被调度的间隔和取消路径？

### 任务队列清单

- [ ] 任务队列是私有队列还是预定义的？选择是否有理由？
- [ ] 如果是私有的，attach 是否在任何可能入队的代码之前调用 `taskqueue_create`（或 `taskqueue_create_fast`）？
- [ ] 如果是私有的，attach 是否在任何期望回调触发的代码之前调用 `taskqueue_start_threads`？
- [ ] 工作线程数量是否适合工作负载？
- [ ] 工作线程优先级是否适合工作负载？
- [ ] 工作线程名称是否足够信息丰富使 `procstat -t` 输出有用？

### 初始化

- [ ] 每个 `TASK_INIT` 是否在 softc 清零后、任务可以被入队前发生？
- [ ] 每个 `TIMEOUT_TASK_INIT` 是否引用了正确的任务队列和有效的回调？
- [ ] attach 是否通过回退较早初始化来处理 `taskqueue_create` 失败？
- [ ] attach 是否通过释放任务队列来处理 `taskqueue_start_threads` 失败？

### 入队点

- [ ] 每个入队点是否指向正确的任务队列指针？
- [ ] 从边缘上下文（callout、中断过滤器）的每次入队是否在入队前确认任务队列存在？
- [ ] 入队调用在它发生的上下文中是否安全（例如，不是在自旋互斥锁内部如果任务队列是 `taskqueue_create`）？
- [ ] 合并行为在每个入队点是否是有意的？

### 回调卫生

- [ ] 每个回调是否有正确的签名 `(void *context, int pending)`？
- [ ] 每个回调是否在需要的地方显式获取驱动锁？
- [ ] 每个回调是否在调用 `selwakeup`、`log` 或其他获取无关锁的函数之前释放驱动锁？
- [ ] 每个回调是否在 `M_WAITOK` 安全的地方避免 `M_NOWAIT` 分配？
- [ ] 回调的总工作时间是否有界？

### 取消

- [ ] 每个 `taskqueue_cancel` / `taskqueue_cancel_timeout` 调用是否在取消竞争重要时在正确的互斥锁下发生？
- [ ] 取消返回 `EBUSY` 的情况是否被处理（通常通过后续排空）？

### 分离

- [ ] 分离是否在排空 callout 前清除 `is_attached`？
- [ ] 分离是否在排空任何任务前排空每个 callout？
- [ ] 分离是否在调用 `seldrain` 前排空每个任务？
- [ ] 分离是否在 `taskqueue_free` 前调用 `seldrain`？
- [ ] 分离是否在销毁互斥锁前调用 `taskqueue_free`？
- [ ] 分离是否在释放后将 `sc->tq` 设为 `NULL`？

### 文档

- [ ] 每个任务是否文档记录在 `LOCKING.md` 中？
- [ ] 纪律规则（入队安全、回调锁定、排空顺序）是否被文档记录？
- [ ] 任务队列子系统是否在 README 中提及？
- [ ] 是否有暴露的 sysctl 让用户观察行为？

### 测试

- [ ] 我是否在启用 `WITNESS` 的情况下运行了回归套件？
- [ ] 我是否在所有任务进行中时测试了分离？
- [ ] 我是否在高入队率下运行了长持续时间压力测试？
- [ ] 我是否用 `dtrace` 验证了入队到调度延迟在预期范围内？
- [ ] 我是否在负载下用 `procstat -kk` 确认任务队列线程没有卡住？

通过此审计的驱动是你在负载下可以信任的驱动。



## 参考：在驱动中标准化任务

对于有几个任务的驱动，一致性比巧妙更重要。一个简短的纪律。

### 一个命名约定

选择一个约定并遵循它。本章的约定：

- 任务结构命名为 `<purpose>_task`（例如，`selwake_task`、`bulk_writer_task`）。
- 超时任务结构命名为 `<purpose>_delayed_task`（例如，`reset_delayed_task`）。
- 回调命名为 `myfirst_<purpose>_task`（例如，`myfirst_selwake_task`、`myfirst_bulk_writer_task`）。
- 入队任务的 sysctl（如果有）命名为 `<purpose>_enqueue` 或批量变体的 `<purpose>_flood`。
- 配置任务的 sysctl（如果有）命名为 `<purpose>_<parameter>`（例如，`bulk_writer_batch`）。

新的维护者可以按照约定添加新任务而不用想名字。反之，代码审查立即捕获偏差。

### 一个初始化/排空模式

每个任务使用相同的初始化和排空：

```c
/* In attach, after taskqueue_start_threads: */
TASK_INIT(&sc-><purpose>_task, 0, myfirst_<purpose>_task, sc);

/* In detach, after callout drains, before seldrain: */
taskqueue_drain(sc->tq, &sc-><purpose>_task);
```

对于超时任务：

```c
/* In attach: */
TIMEOUT_TASK_INIT(sc->tq, &sc-><purpose>_delayed_task, 0,
    myfirst_<purpose>_delayed_task, sc);

/* In detach: */
taskqueue_drain_timeout(sc->tq, &sc-><purpose>_delayed_task);
```

调用点简短且统一。审查者可以扫描模式并立即标记偏差。

### 一个回调模式

每个任务回调遵循相同结构：

```c
static void
myfirst_<purpose>_task(void *arg, int pending)
{
        struct myfirst_softc *sc = arg;

        /* Optional: record coalescing for diagnostics. */
        if (pending > 1) {
                MYFIRST_LOCK(sc);
                sc-><purpose>_drops += pending - 1;
                MYFIRST_UNLOCK(sc);
        }

        /* ... do the work, acquiring locks as needed ... */
}
```

可选的合并记录通过 sysctl 使合并行为可见。如果任务很少合并或计数器没有用，可以去掉它。

### 一个文档模式

每个任务在 `LOCKING.md` 中以相同字段文档记录：

- 任务名称和种类（普通或 timeout_task）。
- 回调函数。
- 哪些代码路径入队它。
- 哪些代码路径取消它（如果有）。
- 在分离时在哪里排空它。
- 回调获取什么锁（如果有）。
- 为什么此工作被延迟（即，为什么它不能在入队上下文中运行）。

新任务的文档是机械的。代码审查可以对照代码验证文档。

### 为什么要标准化

标准化有成本：新贡献者必须学习约定；偏差需要特殊理由。但好处更大：

- 减少认知负担。了解模式的读者立即理解每个任务。
- 更少错误。标准模式正确处理常见情况（attach 中初始化、分离中排空、selwakeup 前释放锁）；偏差更可能是错误的。
- 更容易审查。审查者可以扫描形态而不是逐行阅读。
- 更容易交接。未见过驱动的维护者可以按照现有模板添加新任务。

标准化的成本在设计时支付一次。好处永远累积。总是值得的。



## Reference: Further Reading on Taskqueues

供希望深入学习的读者参考。

### 手册页

- `taskqueue(9)`：权威 API 参考。
- `epoch(9)`：epoch 同步框架，与网络任务相关。
- `callout(9)`：相伴原语；`timeout_task` 建立在它之上。
- `swi_add(9)`：`taskqueue_swi` 及相关队列使用的软件中断注册接口。
- `kproc(9)`、`kthread(9)`：直接创建内核线程的接口，适用于 taskqueue 不够用的场景。

### 源代码文件

- `/usr/src/sys/kern/subr_taskqueue.c`：taskqueue 的实现。仔细阅读 `taskqueue_run_locked`；它是该子系统的核心。
- `/usr/src/sys/sys/taskqueue.h`、`/usr/src/sys/sys/_task.h`：公共 API 和数据结构。
- `/usr/src/sys/kern/subr_gtaskqueue.c`、`/usr/src/sys/sys/gtaskqueue.h`：grouptaskqueue 层。
- `/usr/src/sys/sys/epoch.h`、`/usr/src/sys/kern/subr_epoch.c`：epoch 框架。
- `/usr/src/sys/dev/ale/if_ale.c`：一个使用 taskqueue 的简洁以太网驱动。
- `/usr/src/sys/dev/bge/if_bge.c`：一个使用多个任务的较大以太网驱动。
- `/usr/src/sys/dev/wg/if_wg.c`：WireGuard 的 grouptaskqueue 用法。
- `/usr/src/sys/dev/iwm/if_iwm.c`：一个使用 timeout task 的无线驱动。
- `/usr/src/sys/dev/usb/usb_process.c`：USB 专用的每设备进程队列（`usb_proc_*`）。

### 手册页阅读顺序

对于初学 FreeBSD 延迟工作子系统的读者：

1. `taskqueue(9)`：权威 API。
2. `epoch(9)`：无锁读同步框架。
3. `callout(9)`：兄弟定时执行原语。
4. `swi_add(9)`：部分 taskqueue 底层的软件中断层。
5. `kthread(9)`：直接创建线程的替代方案。

每个手册页都建立在前一个的基础之上；按顺序阅读只需几个小时，就能对内核的延迟工作基础设施建立扎实的心智模型。

### 外部资料

*The Design and Implementation of the FreeBSD Operating System*（McKusick 等著）中关于同步的章节涵盖了延迟工作子系统的历史演变。作为背景阅读很有用，但非必需。

FreeBSD 开发者邮件列表（`freebsd-hackers@`）偶尔会讨论 taskqueue 的改进和边缘情况。在归档中搜索 "taskqueue" 可以找到相关的历史背景。

若要深入理解网络栈对 `epoch(9)` 和 grouptaskqueue 的使用，`iflib(9)` 框架文档和 `/usr/src/sys/net/iflib.c` 下的源代码值得一读。它们超出了本章的范围，但解释了现代网络驱动之所以如此架构的原因。

最后，真实的驱动源代码。在 `/usr/src/sys/dev/` 中挑选任何使用 taskqueue 的驱动（大多数都使用），阅读其 taskqueue 相关代码，并与本章的模式进行比较。这种对应是直接的；你会立刻认出那些模式。这种阅读能将本章的抽象概念转化为实际的工作知识。



## Reference: Taskqueue Cost Analysis

简要讨论 taskqueue 的实际开销，有助于决定是否延迟执行或是否创建私有队列。

### 静态开销

一个未入队的 `struct task` 除了结构体本身的大小（amd64 上为 32 字节）之外没有额外开销。内核并不感知它的存在。它静静地待在你的 softc 中，什么也不做。

一个已分配但空闲的 `struct taskqueue` 的开销：
- taskqueue 结构体本身（几百字节）。
- 一个或多个工作线程（amd64 上每个 16 KB 栈，外加调度器状态）。
- 空闲时没有每次入队的开销。

### 每次入队的开销

当你调用 `taskqueue_enqueue(tq, &task)` 时，内核执行以下操作：

1. 获取 taskqueue 的内部互斥锁。微秒级。
2. 检查任务是否已处于待处理状态。常数时间。
3. 如果未待处理，链接到列表中并唤醒工作线程（通过 `wakeup`）。常数时间加一个调度事件。
4. 如果已待处理，递增 `ta_pending`。单次算术运算。
5. 释放互斥锁。

在无竞争的队列上，总开销为微秒级。在存在竞争时，互斥锁获取可能需要更长时间，但框架使用填充对齐的互斥锁来最小化伪共享，且互斥锁很少被持有超过几条指令的时间。

### 每次分发的开销

当工作线程被唤醒并运行 `taskqueue_run_locked` 时，每个任务的开销为：

1. 步进到队列头部。常数时间。
2. 取下任务。常数时间。
3. 记录待处理计数，重置它。常数时间。
4. 释放互斥锁。
5. 进入所需的 epoch（对于网络任务）。
6. 调用回调函数。开销取决于回调。
7. 如果进入了 epoch 则退出。
8. 为下一次迭代重新获取互斥锁。

对于一个典型的短回调（微秒级的工作），每次分发的开销主要由回调本身加上一次互斥锁往返和一次唤醒往返构成。

### 取消/排空时的开销

`taskqueue_cancel` 很快：获取互斥锁、如果待处理则从列表中移除、释放互斥锁。微秒级。

`taskqueue_drain` 在任务空闲时很快。如果任务待处理，排空会等待它运行并完成；持续时间取决于队列深度和回调时长。如果任务正在运行，排空会等待当前调用返回。

`taskqueue_drain_all` 开销更大：它必须等待队列中的每个任务。持续时间与剩余总工作量成正比。

`taskqueue_free` 会排空队列、终止线程并释放状态。线程终止涉及通知每个线程退出并等待其完成当前任务。微秒到毫秒级，取决于队列深度。

### 实际影响

一些实践要点。

**单线程 taskqueue 很便宜。** 每个实例的开销是几百字节加一个 16 KB 的线程栈。在任何实际系统中，这都可以忽略不计。

**共享 taskqueue 每个驱动更便宜但存在竞争。** `taskqueue_thread` 被每个未创建自己队列的驱动使用。在高负载下它成为串行瓶颈。对于有显著任务流量的驱动，私有队列可以避免竞争。

**多线程 taskqueue 用内存换取并行性。** 四个线程就是四个 16 KB 的栈加四个调度器条目。在工作负载天然并行时值得；在工作负载在单个驱动互斥锁上串行化时则浪费。

**合并是免费的性能提升。** 当入队速度超过 taskqueue 的分发速度时，合并将突发请求折叠为单次触发。驱动为 `pending` 计数所隐含的任何工作量支付一次回调调用。

### 与其他方法的比较

使用 `kproc_create` 创建并由驱动管理的内核线程的开销：
- 16 KB 栈加调度器条目（与 taskqueue 工作线程相同）。
- 没有内置的入队/分发框架：驱动需要自行实现队列和唤醒。
- 没有内置的合并或取消功能。

对于符合任务模型（入队、分发、排空）的工作，taskqueue 始终是正确的选择。对于不符合的工作（一个有自己节奏的长时间循环），`kproc_create` 线程可能更合适。

一个入队任务的 callout 结合了两种原语的开销。当工作既需要特定截止时间又需要线程上下文时值得使用。

### 何时需要担心开销

大多数驱动不需要。Taskqueue 很便宜；内核经过了良好的调优。只在以下情况担心开销：

- 性能分析显示 taskqueue 操作主导了 CPU 使用。（用 `dtrace` 确认。）
- 你正在编写高速率驱动（每秒数千事件或更多），且 taskqueue 是串行化点。
- 系统有许多驱动竞争 `taskqueue_thread`，且竞争可被测量。

在所有其他情况下，自然地编写 taskqueue 代码，相信内核能处理负载。



## Reference: The Task Coalescing Semantics, Precisely

合并（coalescing）是最常让新手感到困惑的特性。精确地陈述其语义并配合实例，值得单独作为一个参考小节。

### 规则

当对一个已经处于待处理状态（`task->ta_pending > 0`）的任务调用 `taskqueue_enqueue(tq, &task)` 时，内核递增 `task->ta_pending` 并返回成功。任务不会被第二次链接到队列上。当回调最终运行时，它恰好运行一次，并将累积的 `ta_pending` 值作为第二个参数传入（该字段在回调被调用前重置为零）。

这条规则有一些值得指出的边界情况。

**上限。** `ta_pending` 是一个 `uint16_t`。它在 `USHRT_MAX`（65535）处饱和。超过该点的入队仍然返回成功，但计数器不再增长。在实践中，达到 65535 次合并入队是一个设计问题，而非性能问题。

**`TASKQUEUE_FAIL_IF_PENDING` 标志。** 如果向 `taskqueue_enqueue_flags` 传入此标志，函数返回 `EEXIST` 而不是合并。当你想知道入队是否产生了新的待处理状态时很有用。

**时序。** 合并发生在入队时刻。如果入队 A 和入队 B 都在任务待处理期间发生，两者都会合并。如果入队 A 导致任务开始运行，而入队 B 在回调执行期间发生，入队 B 使任务再次变为待处理（pending=1），回调将在当前调用返回后再次被调用。第二次调用看到 `pending=1`，因为只有 B 被累积了。第一次和第二次调用都会发生；没有入队会丢失。

**优先级。** 如果同一队列上有两个不同的任务待处理，且其中一个优先级更高，则高优先级的先运行，不管入队顺序如何。对于单个任务，优先级不是影响因素；给定任务的所有调用按顺序运行。

### 实例解析

**示例 1：简单单次入队。**

```c
taskqueue_enqueue(tq, &task);
/* Worker fires the callback. */
/* Callback sees pending == 1. */
```

**示例 2：分发前的合并入队。**

```c
taskqueue_enqueue(tq, &task);
taskqueue_enqueue(tq, &task);
taskqueue_enqueue(tq, &task);
/* (Worker has not yet woken up.) */
/* Worker fires the callback. */
/* Callback sees pending == 3. */
```

**示例 3：回调执行期间的入队。**

```c
taskqueue_enqueue(tq, &task);
/* Callback starts; pending is reset to 0. */
/* While callback is running: */
taskqueue_enqueue(tq, &task);
/* Callback finishes its first invocation. */
/* Worker notices pending == 1; fires callback again. */
/* Second callback invocation sees pending == 1. */
```

**示例 4：分发前的取消。**

```c
taskqueue_enqueue(tq, &task);
taskqueue_enqueue(tq, &task);
/* Cancel: */
taskqueue_cancel(tq, &task, &pendp);
/* pendp == 2; callback does not run. */
```

**示例 5：执行期间的取消。**

```c
taskqueue_enqueue(tq, &task);
/* Callback starts. */
/* During callback: */
taskqueue_cancel(tq, &task, &pendp);
/* Returns EBUSY; pending (if any future enqueues came in) may or may not be zeroed. */
/* The currently executing invocation completes; the cancellation affects only future runs. */
```

### 设计影响

从上述规则可以得出几个设计影响。

**回调必须对 `pending` 具有幂等性。** 编写假设 `pending==1` 的回调在高负载下会出问题。始终有意识地使用 `pending`，要么循环 `pending` 次，要么做一次处理累积状态的遍历。

**不要用"回调调用次数"作为事件计数。** 使用每次调用中的 `pending` 值求和。或者更好的方式，使用一个按事件的状态结构（softc 内部的队列）让回调去排空。

**合并将按事件的工作变为按突发的工作。** 一个每次调用做 O(1) 工作并丢弃 `pending` 的回调，无论入队速率如何都处理相同数量的工作。这对"通知等待者"类工作通常没问题；但对"处理每个事件"类工作则不对。

**合并让你可以自由地从边缘上下文入队。** 一个每毫秒触发的 callout 可以每毫秒入队一个任务；如果回调需要 10 毫秒运行，九次入队会合并为每次回调的一次调用。系统自然地收敛到回调所能维持的吞吐量。



## Reference: The Taskqueue State Diagram

一个简短的单任务状态图，作为推理生命周期的辅助工具。

```text
        +-----------+
        |   IDLE    |
        | pending=0 |
        +-----+-----+
              |
              | taskqueue_enqueue
              v
        +-----------+           +--------+
        |  PENDING  | <--- enq--|  any   |
        | pending>=1|          +--------+
        +-----+-----+
              |
              | worker picks up
              v
        +-----------+
        |  RUNNING  |
        | (callback |
        | executing)|
        +-----+-----+
              |
              | callback returns
              v
        +-----------+
        |   IDLE    |
        | pending=0 |
        +-----------+
```

一个任务始终处于以下三种状态之一：IDLE（空闲）、PENDING（待处理）或 RUNNING（运行中）。

**IDLE。** 不在任何队列上。`ta_pending == 0`。入队操作将其转移到 PENDING。

**PENDING。** 在 taskqueue 的待处理列表上。`ta_pending >= 1`。合并操作递增 `ta_pending` 但不离开 PENDING 状态。取消操作将其移回 IDLE。

**RUNNING。** 在 `tq_active` 中，回调正在执行。`ta_pending` 已被重置为零，回调已收到先前的值。新的入队会转换回 PENDING（因此回调返回后，工作线程会再次触发它）。在此状态下取消返回 `EBUSY`。

所有状态转换都由 `tq_mutex` 串行化。在任何时刻，内核都能告诉你任务处于哪个状态，且转换是原子的。

`taskqueue_drain(tq, &task)` 等待直到任务处于 IDLE 状态且没有新的入队到达后才返回。这是排空操作提供的精确保证。



## Reference: Observability Cheat Sheet

调试时的快速参考。

### 列出所有 Taskqueue 线程

```text
# procstat -t | grep taskq
```

### 用 DTrace 列出所有任务提交速率

```text
# dtrace -n 'fbt::taskqueue_enqueue:entry { @[(caddr_t)arg1] = count(); }' -c 'sleep 10'
```

该脚本在十秒内统计每个任务指针的入队次数。任务指针可以通过 `addr2line` 或 `kgdb` 映射回驱动。

### 测量分发延迟

```text
# dtrace -n '
  fbt::taskqueue_enqueue:entry { self->t = timestamp; }
  fbt::taskqueue_run_locked:entry /self->t/ {
        @[execname] = quantize(timestamp - self->t);
        self->t = 0;
  }
' -c 'sleep 10'
```

### 测量回调持续时间

```text
# dtrace -n '
  fbt::taskqueue_run_locked:entry { self->t = timestamp; }
  fbt::taskqueue_run_locked:return /self->t/ {
        @[execname] = quantize(timestamp - self->t);
        self->t = 0;
  }
' -c 'sleep 10'
```

### Taskqueue 线程卡住时的栈跟踪

```text
# procstat -kk <pid>
```

### ddb 中的活跃任务

在 `ddb` 提示符下：

```text
db> show taskqueues
```

列出每个 taskqueue、其活跃任务（如果有）和待处理队列。

### 驱动应提供的 Sysctl 旋钮

对于驱动拥有的每个任务，考虑暴露：

- `<purpose>_enqueues`：尝试的入队总数。
- `<purpose>_coalesced`：合并发生的次数。
- `<purpose>_runs`：回调调用总数。
- `<purpose>_largest_pending`：峰值待处理计数。

正常条件下：`enqueues == runs + coalesced`。合并条件下：`runs < enqueues`。无负载时：`largest_pending == 1`。高负载时：`largest_pending` 增长。

这些计数器将不透明的驱动行为转化为可读的 sysctl 显示。开销只是几次原子加法；价值很高。



## Reference: A Minimal Working Task Template

供复制和修改之用。每个部分都已在正文中介绍过；模板将它们组装成一个可直接使用的骨架。

```c
#include <sys/param.h>
#include <sys/kernel.h>
#include <sys/module.h>
#include <sys/systm.h>
#include <sys/bus.h>
#include <sys/taskqueue.h>
#include <sys/mutex.h>
#include <sys/lock.h>

struct example_softc {
        device_t          dev;
        struct mtx        mtx;
        struct taskqueue *tq;
        struct task       work_task;
        int               is_attached;
};

static void
example_work_task(void *arg, int pending)
{
        struct example_softc *sc = arg;

        mtx_lock(&sc->mtx);
        /* ... do work under the mutex if state protection is needed ... */
        mtx_unlock(&sc->mtx);

        /* ... do lock-free work or calls like selwakeup here ... */
}

static int
example_attach(device_t dev)
{
        struct example_softc *sc = device_get_softc(dev);
        int error;

        sc->dev = dev;
        mtx_init(&sc->mtx, device_get_nameunit(dev), "example", MTX_DEF);

        sc->tq = taskqueue_create("example taskq", M_WAITOK,
            taskqueue_thread_enqueue, &sc->tq);
        if (sc->tq == NULL) {
                error = ENOMEM;
                goto fail_mtx;
        }
        error = taskqueue_start_threads(&sc->tq, 1, PWAIT,
            "%s taskq", device_get_nameunit(dev));
        if (error != 0)
                goto fail_tq;

        TASK_INIT(&sc->work_task, 0, example_work_task, sc);
        sc->is_attached = 1;
        return (0);

fail_tq:
        taskqueue_free(sc->tq);
fail_mtx:
        mtx_destroy(&sc->mtx);
        return (error);
}

static int
example_detach(device_t dev)
{
        struct example_softc *sc = device_get_softc(dev);

        mtx_lock(&sc->mtx);
        sc->is_attached = 0;
        mtx_unlock(&sc->mtx);

        taskqueue_drain(sc->tq, &sc->work_task);
        taskqueue_free(sc->tq);
        mtx_destroy(&sc->mtx);
        return (0);
}

/* Elsewhere, a code path that wants to defer work: */
static void
example_trigger_work(struct example_softc *sc)
{
        if (sc->is_attached)
                taskqueue_enqueue(sc->tq, &sc->work_task);
}
```

每个要素都是必不可少的。删掉任何一个都会重新引入本章警告过的 bug。



## Reference: Comparison With Linux Workqueues

供来自 Linux 内核开发的读者参考的简短比较。两个系统解决相同的问题；差异在于命名、粒度和默认值。

### 命名

| 概念 | FreeBSD | Linux |
|---|---|---|
| 延迟工作单元 | `struct task` | `struct work_struct` |
| 队列 | `struct taskqueue` | `struct workqueue_struct` |
| 共享队列 | `taskqueue_thread` | `system_wq` |
| 无绑定队列 | `taskqueue_thread`（多线程） | `system_unbound_wq` |
| 创建队列 | `taskqueue_create` | `alloc_workqueue` |
| 入队 | `taskqueue_enqueue` | `queue_work` |
| 延迟入队 | `taskqueue_enqueue_timeout` | `queue_delayed_work` |
| 等待工作完成 | `taskqueue_drain` | `flush_work` |
| 销毁队列 | `taskqueue_free` | `destroy_workqueue` |
| 优先级 | `ta_priority` | `WQ_HIGHPRI` 标志 |
| 合并行为 | 自动，暴露 `pending` 计数 | `work_pending` 检查，无计数 |

### 语义差异

**合并可见性。** FreeBSD 将待处理计数暴露给回调；Linux 不暴露。Linux 回调知道工作被触发了，但不知道被请求了多少次。

**超时任务 vs 延迟工作。** FreeBSD 的 `timeout_task` 嵌入了一个 callout；Linux 的 `delayed_work` 嵌入了一个 `timer_list`。从用户角度看两者行为相同。

**Grouptaskqueue vs percpu 工作队列。** FreeBSD 的 `taskqgroup` 是显式且独立的；Linux 的 `alloc_workqueue(..., WQ_UNBOUND | WQ_CPU_INTENSIVE)` 有类似的语义但旋钮不同。

**Epoch 集成。** FreeBSD 有 `NET_TASK_INIT` 用于在网络 epoch 内运行的任务；Linux 没有直接对应物（RCU 框架类似但不完全相同）。

从 Linux 移植到 FreeBSD（或反之）的驱动通常可以几乎一一对应地翻译延迟工作模式。结构差异更多在周围的 API（设备注册、内存分配、锁定）而非 taskqueue 本身。



## Reference: When Not To Use a Taskqueue

简短列出其他原语更合适的场景。

**工作有严格的时序要求。** Taskqueue 会增加调度延迟。对于微秒级截止时间，`C_DIRECT_EXEC` callout 或 `taskqueue_swi` 更快。对于纳秒级截止时间，所有延迟工作机制都不够快；工作需要内联执行。

**工作是一次性清理，没有关联的生产者。** 在拆卸路径中的简单 `free` 不需要 taskqueue；直接调用即可。为延迟而延迟没有任何价值。

**工作必须在高于 `PWAIT` 的特定调度器优先级运行。** 如果工作是真正的高优先级（实时驱动、中断阈值任务），使用 `kthread_add` 配合显式优先级，而不是通用 taskqueue。

**工作需要通用工作线程无法提供的特定线程上下文。** 任务在没有特定用户进程上下文的内核线程中运行。需要特定用户凭据、文件描述符表或地址空间的工作必须在该进程内完成，而不是在任务中。

**驱动只有一个任务且很少运行。** 一个使用 `cv_timedwait` 循环的 `kthread_add` 可能比完整的 taskqueue 设置更清晰。自行判断；对于三个或更多任务，taskqueue 几乎总是更清晰。

对于其他所有情况，使用 taskqueue。默认选择是"使用 `taskqueue(9)`"；例外情况是狭窄的。



## Reference: A Worked Reading of `subr_taskqueue.c`

再来一次阅读练习，因为理解实现能让 API 的行为变得可预测。

该文件是 `/usr/src/sys/kern/subr_taskqueue.c`。其结构简要如下：

**`struct taskqueue`。** 在文件顶部附近定义。包含待处理队列（`tq_queue`）、活跃任务列表（`tq_active`）、内部互斥锁（`tq_mutex`）、入队回调（`tq_enqueue`）、工作线程（`tq_threads`）和标志位。

**`TQ_LOCK` / `TQ_UNLOCK` 宏。** 紧跟在结构体定义之后。获取互斥锁（自旋或睡眠，取决于 `tq_spin`）。

**`taskqueue_create` 和 `_taskqueue_create`。** 分配结构体，初始化互斥锁（MTX_DEF 或 MTX_SPIN），返回。

**`taskqueue_enqueue` 和 `taskqueue_enqueue_flags`。** 获取互斥锁，检查 `task->ta_pending`，合并或链接，唤醒工作线程（通过 `enqueue` 回调），释放互斥锁。

**`taskqueue_enqueue_timeout`。** 调度内部 callout；callout 的回调稍后会调用底层任务的 `taskqueue_enqueue`。

**`taskqueue_cancel` 和 `taskqueue_cancel_timeout`。** 如果待处理则从队列中移除；如果正在运行则返回 `EBUSY`。

**`taskqueue_drain` 及其变体。** 在任务空闲且未待处理的条件上 `msleep`。

**`taskqueue_run_locked`。** 子系统的核心。在循环中：从待处理队列取出一个任务，记录 `ta_pending`，清零，移至活跃列表，释放互斥锁，可选进入网络 epoch，调用回调，重新获取互斥锁，通知排空等待者。循环直到队列为空。

**`taskqueue_thread_loop`。** 工作线程的主循环。获取 taskqueue 互斥锁，如果队列为空则等待工作（`msleep`），工作到来时调用 `taskqueue_run_locked`，循环。

**`taskqueue_free`。** 设置"排空中"标志，唤醒每个工作线程，等待每个工作线程退出，排空剩余任务，释放结构体。

这段阅读以函数名而非行号引用每个函数，因为行号在不同 FreeBSD 版本间会变化，而符号名不会。如果你想在 FreeBSD 14.3 中找到 `subr_taskqueue.c` 的大致位置，主要入口点大约在以下行：`_taskqueue_create` 141、`taskqueue_create` 178、`taskqueue_free` 217、`taskqueue_enqueue_flags` 305、`taskqueue_enqueue` 317、`taskqueue_enqueue_timeout` 382、`taskqueue_run_locked` 485、`taskqueue_cancel` 579、`taskqueue_cancel_timeout` 591、`taskqueue_drain` 612、`taskqueue_thread_loop` 820。将这些数字视为滚动提示；打开文件后跳转到符号名即可。

阅读这些函数一次是值得的投资。第 14 章所教授的关于 API 行为的一切都可以在实现中看到。



## 最终导览：五种常见形状

五种常见形状涵盖了 FreeBSD 源码树中大多数 taskqueue 的使用方式。识别它们能将阅读驱动源码从逐行解析变为模式匹配。

### 形状 A：独立任务

一个任务，从一个地方入队，在 detach 时排空。最简单。用于需要延迟恰好一种工作的驱动。

```c
TASK_INIT(&sc->task, 0, sc_task, sc);
/* ... */
taskqueue_enqueue(sc->tq, &sc->task);
/* ... */
taskqueue_drain(sc->tq, &sc->task);
```

### 形状 B：过滤器加任务分离

中断过滤器做最少的工作，入队一个任务处理剩余部分。

```c
static int
sc_filter(void *arg)
{
        struct sc *sc = arg;
        taskqueue_enqueue(sc->tq, &sc->intr_task);
        return (FILTER_HANDLED);
}
```

### 形状 C：Callout 驱动的周期性任务

Callout 周期性触发，入队一个任务执行实际工作。

```c
static void
sc_periodic_callout(void *arg)
{
        struct sc *sc = arg;
        taskqueue_enqueue(sc->tq, &sc->periodic_task);
        callout_reset(&sc->co, hz, sc_periodic_callout, sc);
}
```

### 形状 D：超时任务

`timeout_task` 用于线程上下文中的延迟工作。

```c
TIMEOUT_TASK_INIT(sc->tq, &sc->delayed, 0, sc_delayed, sc);
/* ... */
taskqueue_enqueue_timeout(sc->tq, &sc->delayed, delay_ticks);
/* ... */
taskqueue_drain_timeout(sc->tq, &sc->delayed);
```

### 形状 E：自重新入队的任务

一个任务从自己的回调中再次调度自身。

```c
static void
sc_self(void *arg, int pending)
{
        struct sc *sc = arg;
        /* work */
        if (sc->keep_running)
                taskqueue_enqueue_timeout(sc->tq, &sc->self_tt, hz);
}
```

你阅读的每个驱动都会使用这五种形状的某种组合。一旦熟悉了它们，剩下的就只是实现细节了。



## 总结：本章交付了什么

简短的清单，供完成全章阅读后想要压缩版本的读者参考。

**引入的概念。**

- 延迟工作作为从边缘上下文到线程上下文的桥梁。
- `struct task` / `struct timeout_task` 数据结构及其生命周期。
- Taskqueue 作为队列加工作线程的组合。
- 私有 taskqueue 与预定义 taskqueue 以及各自的适用场景。
- 通过 `ta_pending` 和 `pending` 参数实现的合并。
- 队列内的优先级排序。
- `block`/`unblock`/`quiesce`/`drain_all` 原语。
- 包含 callout、任务、selinfo 和 taskqueue 拆卸的 detach 顺序。
- 通过 `procstat`、`dtrace`、sysctl 计数器和 `WITNESS` 进行调试。
- 对 `epoch(9)`、grouptaskqueue 和每 CPU taskqueue 的初步接触。

**驱动变更。**

- 阶段 1：一个从 `tick_source` 入队、在 detach 时排空的任务。
- 阶段 2：驱动拥有的私有 taskqueue。
- 阶段 3：展示有意合并的批量写入任务，用于延迟重置的 `timeout_task`。
- 阶段 4：整合，版本升级至 `0.8-taskqueues`，完整回归测试。

**文档变更。**

- `LOCKING.md` 中的 Tasks 节。
- 枚举每个排空步骤的 detach 顺序节。
- 每任务的文档，列出回调、生命周期、入队路径和取消路径。

**编目的模式。**

- 延迟日志。
- 延迟重置。
- 过滤器加任务的中断分离。
- 异步 `copyin`/`copyout`。
- 带退避的重试。
- 延迟拆卸。
- 统计滚动。
- 重配置期间阻塞。
- 子系统边界的排空全部。
- 合成事件生成。

**使用的调试工具。**

- `procstat -t` 用于 taskqueue 线程状态。
- `ps ax` 用于内核线程清单。
- `sysctl dev.<driver>` 用于驱动暴露的计数器。
- `dtrace` 用于入队延迟和回调持续时间。
- `procstat -kk` 用于卡住线程的诊断。
- `WITNESS` 和 `INVARIANTS` 作为调试内核的安全网。

**交付物。**

- `content/chapters/part3/chapter-14.md`（本文件）。
- `examples/part-03/ch14-taskqueues-and-deferred-work/stage1-first-task/`。
- `examples/part-03/ch14-taskqueues-and-deferred-work/stage2-private-taskqueue/`。
- `examples/part-03/ch14-taskqueues-and-deferred-work/stage3-coalescing/`。
- `examples/part-03/ch14-taskqueues-and-deferred-work/stage4-final/`。
- `examples/part-03/ch14-taskqueues-and-deferred-work/labs/`，包含 `poll_waiter.c` 和小型辅助脚本。
- `examples/part-03/ch14-taskqueues-and-deferred-work/LOCKING.md`，包含 Tasks 节。
- `examples/part-03/ch14-taskqueues-and-deferred-work/README.md`，包含每个阶段的构建和测试说明。

第 14 章到此结束。第 15 章继续同步的故事。


## Reference: Reading `taskqueue_run_locked` Line By Line

taskqueue 子系统的核心是 `/usr/src/sys/kern/subr_taskqueue.c` 中 `taskqueue_run_locked` 内的一个短循环。慢慢地读一遍，每当你需要推理子系统的行为时都会有所回报。以下是一段带注解的通读。

该函数从工作线程的主循环 `taskqueue_thread_loop` 中调用，调用时持有 taskqueue 互斥锁。它的任务是处理每个待处理任务，在回调周围释放互斥锁，并在队列为空时带着互斥锁仍被持有而返回。

```c
static void
taskqueue_run_locked(struct taskqueue *queue)
{
        struct epoch_tracker et;
        struct taskqueue_busy tb;
        struct task *task;
        bool in_net_epoch;
        int pending;

        KASSERT(queue != NULL, ("tq is NULL"));
        TQ_ASSERT_LOCKED(queue);
        tb.tb_running = NULL;
        LIST_INSERT_HEAD(&queue->tq_active, &tb, tb_link);
        in_net_epoch = false;
```

函数首先断言互斥锁已被持有，并将一个局部的 `taskqueue_busy` 结构插入活跃列表。`tb` 结构代表 `taskqueue_run_locked` 的本次调用；后续代码用它来跟踪本次调用当前正在运行什么。`in_net_epoch` 标志跟踪我们当前是否处于网络 epoch 内，这样当连续的任务都是网络标记的时，就不会冗余地进入 epoch。

```c
        while ((task = STAILQ_FIRST(&queue->tq_queue)) != NULL) {
                STAILQ_REMOVE_HEAD(&queue->tq_queue, ta_link);
                if (queue->tq_hint == task)
                        queue->tq_hint = NULL;
                pending = task->ta_pending;
                task->ta_pending = 0;
                tb.tb_running = task;
                tb.tb_seq = ++queue->tq_seq;
                tb.tb_canceling = false;
                TQ_UNLOCK(queue);
```

主循环。从待处理队列头部取出任务。将待处理计数快照到局部变量中，将字段重置为零（这样回调期间到达的新入队从零开始递增）。将任务记录在 `tb` 结构中，以便排空调用者能看到正在运行什么。递增序列计数器用于过时排空检测。释放互斥锁。

注意从这里到下一个 `TQ_LOCK` 之间，互斥锁未被持有。这是回调运行的窗口；内核的其余部分可以入队更多任务（会合并或排队）、排空其他任务（会看到 `tb.tb_running == task` 并等待），或做自己的事情。

```c
                KASSERT(task->ta_func != NULL, ("task->ta_func is NULL"));
                if (!in_net_epoch && TASK_IS_NET(task)) {
                        in_net_epoch = true;
                        NET_EPOCH_ENTER(et);
                } else if (in_net_epoch && !TASK_IS_NET(task)) {
                        NET_EPOCH_EXIT(et);
                        in_net_epoch = false;
                }
                task->ta_func(task->ta_context, pending);

                TQ_LOCK(queue);
                wakeup(task);
        }
        if (in_net_epoch)
                NET_EPOCH_EXIT(et);
        LIST_REMOVE(&tb, tb_link);
}
```

Epoch 簿记：如果此任务是网络标记的且我们尚未进入 epoch，则进入网络 epoch；如果我们为较早的任务进入了 epoch 但此任务不是网络标记的，则退出 epoch。这让连续的网络任务共享单个 epoch 进入，这是框架免费提供的优化。

用上下文和待处理计数调用回调。重新获取互斥锁。唤醒任何等待此特定任务的排空调用者。循环。

循环结束后，如果我们仍在网络 epoch 中，退出它。从活跃列表中移除 `tb` 结构。

从阅读这个函数得出的七个观察。

**观察 1。** 互斥锁在回调运行期间恰好不被持有。没有 taskqueue 内部代码与回调同时运行；如果回调需要毫秒，taskqueue 互斥锁就空闲毫秒。

**观察 2。** `ta_pending` 在回调运行之前而非之后被重置。回调期间的新入队使任务再次变为待处理（pending=1）。回调返回后，循环看到新的待处理，将其取出，并以 pending=1 再次运行回调。没有入队会丢失。

**观察 3。** 传给回调的 `pending` 值是任务从队列中取出时刻的计数，而非入队调用发生时刻的计数。如果回调期间有入队到达，它们不计入本次调用的 `pending`；它们计入下一次调用的 `pending`。

**观察 4。** 循环底部的唤醒会唤醒在任务地址上睡眠的排空调用者。排空使用 `msleep(&task, &tq->mutex, ...)`，等待任务不在队列上且当前未运行。这里的唤醒就是使排空终止的原因。

**观察 5。** 序列计数器 `tq_seq` 和 `tb.tb_seq` 允许 drain-all 检测在排空开始后是否有新任务被添加。没有序列号，drain-all 会与新入队竞争。

**观察 6。** `tb.tb_canceling` 是 `taskqueue_cancel` 设置的标志，用于告诉等待者"此任务当前正在被取消"；其目的是让并发的 cancel/drain 调用协调。我们在正文中没有讨论它，因为大多数驱动永远不会遇到它。

**观察 7。** 多个工作线程可以同时处于 `taskqueue_run_locked` 内部，每个分发不同的任务。`tq_active` 列表保存所有它们的 `tb` 结构。同一队列上的不同任务并行运行；同一任务不能与自身并行运行，因为同一时刻只有一个工作线程将其取出。

这些观察合在一起精确描述了 taskqueue 保证什么和不保证什么。本章前面描述的每个行为都是这个短循环的结果。



## Reference: A Walkthrough of `taskqueue_drain`

同样有启发，同样简短。来自 `/usr/src/sys/kern/subr_taskqueue.c`，大致如下：

```c
void
taskqueue_drain(struct taskqueue *queue, struct task *task)
{
        if (!queue->tq_spin)
                WITNESS_WARN(WARN_GIANTOK | WARN_SLEEPOK, NULL, ...);

        TQ_LOCK(queue);
        while (task->ta_pending != 0 || task_is_running(queue, task))
                TQ_SLEEP(queue, task, "taskqueue_drain");
        TQ_UNLOCK(queue);
}
```

函数获取 taskqueue 互斥锁，然后循环直到任务既非待处理也非运行中。每次迭代在任务地址上睡眠；`taskqueue_run_locked` 底部的每次唤醒都会唤醒排空者重新检查。

`task_is_running(queue, task)` 遍历活跃列表（`tq_active`），如果任何 `tb.tb_running == task` 则返回 true。它是工作线程数的 O(N)，但对于大多数驱动 N 为 1，所以是 O(1)。

函数在睡眠期间不持有锁；`TQ_SLEEP`（展开为 `msleep` 或 `msleep_spin`）在睡眠期间释放互斥锁并在唤醒时重新获取，这是标准的条件变量模式。

从阅读 `taskqueue_drain` 得出的观察。

**观察 1。** 排空是一个条件变量等待，使用任务指针作为唤醒通道。唤醒来自 `taskqueue_run_locked` 底部的 `wakeup(task)`。

**观察 2。** 排空不会阻止新入队发生。如果在排空等待期间任务被再次入队，排空将继续等待直到那次新入队触发并完成。这就是为什么 detach 纪律要求在排空目标任务之前先排空每个生产者（callout、其他任务、中断处理程序）。

**观察 3。** 对空闲任务（从未入队，或已入队并已完成）的排空立即返回。在 detach 中无条件调用排空是安全的。

**观察 4。** 排空在初始检查之前和睡眠之前持有 taskqueue 互斥锁，这意味着排空不会以错过新待处理任务的方式与入队竞争。如果入队在检查和睡眠之间到达，`ta_pending` 变为非零，排空循环重新迭代。

**观察 5。** 顶部的 `WITNESS_WARN` 断言调用者处于可以合法睡眠的上下文中。如果你尝试从不能睡眠的上下文（例如 callout 回调）调用 `taskqueue_drain`，`WITNESS` 会发出警告。

两个互补函数是 `taskqueue_cancel`（如果待处理则从队列中移除任务，如果正在运行则返回 `EBUSY`）和 `taskqueue_drain_timeout`（还会取消嵌入的 callout）。阅读它们的实现一次是值得的；它们都很短。



## Reference: The Lifecycle As Seen From the Softc

为了完整性，再来一个视角。同样的信息，但以 softc 而非 API 为中心来组织。

在 **attach 时**，softc 获得：

- 一个 taskqueue 指针（`sc->tq`），由 `taskqueue_create` 创建并由 `taskqueue_start_threads` 填充。
- 一个或多个任务结构（`sc->foo_task`），由 `TASK_INIT` 或 `TIMEOUT_TASK_INIT` 初始化。
- 用于可观察性的计数器和标志（可选但推荐）。

在 **运行时**，softc 的 taskqueue 状态为：

- `sc->tq` 是一个不透明指针；驱动从不读取其字段。
- `sc->foo_task` 在任何时刻可能处于 IDLE、PENDING 或 RUNNING 状态。
- taskqueue 的工作线程大部分时间在睡眠，入队时被唤醒，运行回调，再次睡眠。

在 **detach 时**，softc 按以下顺序拆卸：

1. 在互斥锁保护下清除 `sc->is_attached`，广播条件变量，释放互斥锁。
2. 排空每个 callout。
3. 排空每个任务。
4. 排空 selinfo。
5. 释放 taskqueue。
6. 销毁 cdev 及其别名。
7. 释放 sysctl 上下文。
8. 销毁 cbuf 和计数器。
9. 销毁条件变量、sx 锁、互斥锁。

`taskqueue_free` 之后，`sc->tq` 变为无效。任务排空之后，`sc->foo_task` 结构处于空闲状态，其存储可以随 softc 一起回收。

softc 的生命周期由设备的 attach/detach 决定。任务不能比其 softc 活得更久。detach 时的排空保证了这一属性。



## Reference: A Glossary of Terms

供快速查阅。

**任务（Task）。** `struct task` 的实例；一个打包了回调和上下文、用于入队到 taskqueue 的单元。

**Taskqueue。** `struct taskqueue` 的实例；一个待处理任务队列与一个或多个工作线程的配对。

**超时任务（Timeout task）。** `struct timeout_task` 的实例；一个任务加上一个内部 callout，用于未来某个时间点的计划工作。

**入队（Enqueue）。** 将任务添加到 taskqueue。如果任务已处于待处理状态，则递增其待处理计数。

**排空（Drain）。** 等待直到任务既非待处理也非运行中。

**分发（Dispatch）。** taskqueue 工作线程从待处理列表取出任务并运行其回调的动作。

**合并（Coalesce）。** 将冗余的入队折叠为单次待处理状态递增，而非两个列表条目。

**待处理计数（Pending count）。** `ta_pending` 的值，表示此任务上累积了多少次合并入队。

**空闲任务（Idle task）。** 既非待处理也非运行中的任务。`ta_pending == 0` 且没有工作线程持有它。

**工作线程（Worker thread）。** 一个内核线程（通常每个 taskqueue 一个），其职责是等待工作并运行任务回调。

**边缘上下文（Edge context）。** 受限的上下文（callout、中断过滤器、epoch 区段），其中某些操作不被允许。

**线程上下文（Thread context）。** 普通的内核线程上下文，允许睡眠、可睡眠锁获取和所有标准操作。

**Detach 顺序（Detach ordering）。** 在设备 detach 时排空和释放原语的顺序，使得没有原语在仍有东西引用它时被释放。

**排空竞争（Drain race）。** 一个 bug，即原语在回调或处理程序仍可能运行时被释放，由不正确的 detach 顺序导致。

**待处理丢弃计数器（Pending-drop counter）。** 当回调的 `pending` 参数大于一时递增的诊断计数器，表明发生了合并。

**私有 taskqueue（Private taskqueue）。** 由驱动拥有的 taskqueue，随 attach/detach 创建和释放，不与其他驱动共享。

**共享 taskqueue（Shared taskqueue）。** 内核提供的 taskqueue（`taskqueue_thread`、`taskqueue_swi` 等），由多个驱动同时使用。

**快速 taskqueue（Fast taskqueue）。** 使用 `taskqueue_create_fast` 创建的 taskqueue，内部使用自旋互斥锁，从过滤器中断上下文入队是安全的。

**Grouptaskqueue。** 一种可扩展变体，任务分布在每 CPU 的队列上。由高速网络驱动使用。

**Epoch。** 一种无锁读同步机制。`net_epoch_preempt` epoch 保护网络状态。



第 14 章到此结束。下一章将继续讲述同步的故事。

