---
title: "定时器与延迟工作"
description: "FreeBSD 驱动程序如何表达时间：使用 callout(9) 调度未来的工作，在文档化的锁下安全运行，以及在卸载时无竞争地拆除。"
partNumber: 3
partName: "并发与同步"
chapter: 13
lastUpdated: "2026-04-18"
status: "complete"
author: "Edson Brandi"
reviewer: "TBD"
translator: "AI辅助翻译为简体中文"
language: "zh-CN"
estimatedReadTime: 195
---

# 定时器与延迟工作

## 读者指南与学习目标

到目前为止，我们编写的每一行驱动程序代码都是*反应式*的：用户调用 `read(2)`，内核调用我们的处理程序，我们完成工作，我们返回。第 12 章的阻塞原语通过能够*等待*我们没有发起的事情扩展了该模型。但驱动程序本身从未主动接触世界。它没有办法说"从现在起 100 毫秒后，请执行这个"。它根本无法计数时间，除了在已经在系统调用内部时观察到时间流逝。

这在这里改变了。第 13 章将*时间*作为驱动程序中的一等概念引入。内核有一个完整的子系统，专门用于在未来的特定时刻运行你的代码，如果你要求的话可以重复运行，具有精确的锁处理规则和干净的拆除语义。它被称为 `callout(9)`，它小巧、规则且非常有用。在本章结束时，你的 `myfirst` 驱动程序将学会调度自己的工作，在不被推动的情况下主动作用于世界，并在卸载设备时安全地归还其调度工作。

### 为什么本章值得一席之地

你可以尝试伪造定时器。一个在循环中休眠的内核线程，每次醒来时调用 `cv_timedwait_sig` 并做工作，技术上就是一个定时器。一个每秒打开设备一次并触发 sysctl 的用户空间进程也是如此。两者都不是错误的，但与内核提供的相比都很笨拙，而且两者都创建了自己需要管理生存期的额外资源（内核线程、用户空间进程）。

`callout(9)` 在几乎每种你想"稍后"运行函数的情况下都是正确的答案。它构建在内核的硬件时钟基础设施上，静止时几乎不消耗资源，可以扩展到每个系统数千个待处理 callout，与 `WITNESS` 和 `INVARIANTS` 集成，并提供了关于如何与锁交互以及如何在拆除时排空待处理工作的清晰规则。`/usr/src/sys/dev/` 中的大多数驱动程序都使用它。一旦你了解了它，该模式可以转移到你将遇到的每种驱动程序：USB、网络、存储、看门狗、传感器，任何其物理世界中有时钟的东西。

不掌握 `callout(9)` 的代价很高。重新发明计时方式的驱动程序创建了一个其他人都不知道如何调试的私有子系统。正确使用 `callout(9)` 的驱动程序可以顺利接入内核现有的可观察性工具（`procstat`、`dtrace`、`lockstat`、`ddb`），并在卸载时行为可预测。本章在你第一次需要扩展别人编写的驱动程序时就物超所值。

### 第 12 章留下的驱动程序状态

简要回顾你应该站在哪里，因为第 13 章直接基于第 12 章的交付物。如果以下任何内容缺失或感觉不确定，请在开始本章之前返回第 12 章。

- 你的 `myfirst` 驱动程序干净编译，版本为 `0.6-sync`。
- 它使用 `MYFIRST_LOCK(sc)` / `MYFIRST_UNLOCK(sc)` 宏围绕 `sc->mtx`（数据路径互斥锁）。
- 它使用 `MYFIRST_CFG_SLOCK(sc)` / `MYFIRST_CFG_XLOCK(sc)` 围绕 `sc->cfg_sx`（配置 sx）。
- 它使用两个命名条件变量（`sc->data_cv`、`sc->room_cv`）进行阻塞读取和写入。
- 它通过 `cv_timedwait_sig` 和 `read_timeout_ms` sysctl 支持定时读取。
- 锁顺序 `sc->mtx -> sc->cfg_sx` 在 `LOCKING.md` 中有文档记录，并由 `WITNESS` 强制执行。
- 你的测试内核中启用了 `INVARIANTS` 和 `WITNESS`；你已经构建并启动了它。
- 第 12 章压力套件（第 11 章测试器加上 `timeout_tester` 和 `config_writer`）可以构建并干净运行。

那个驱动程序是我们在第 13 章中扩展的。我们将添加一个周期性 callout，然后是一个看门狗 callout，然后是一个可配置的 tick 源，最后通过重构和文档更新将它们整合。驱动程序的数据路径保持不变；新代码与现有原语并存。

### 你将学到什么

在进入下一章时，你将能够：

- 解释何时 callout 是正确的原语，何时内核线程、`cv_timedwait` 或用户空间助手更合适。
- 使用 `callout_init`、`callout_init_mtx`、`callout_init_rw` 或 `callout_init_rm` 初始化带有适当锁感知的 callout，并在驱动程序上下文的 lock-managed 和 `mpsafe` 变体之间进行选择。
- 使用 `callout_reset`（基于 tick）或 `callout_reset_sbt`（亚 tick 精度）调度一次性定时器，在适当的地方使用 `tick_sbt`、`SBT_1S`、`SBT_1MS`、`SBT_1US` 时间常量。
- 通过让回调重新武装自己来调度周期性定时器，使用能够经受 `callout_drain` 的正确模式。
- 在 `callout_reset` 和 `callout_schedule` 之间进行选择，并理解何时每个是正确的工具。
- 描述当你用锁指针初始化时 `callout(9)` 强制执行的锁契约：内核在函数运行之前获取该锁，在之后释放它（除非你设置了 `CALLOUT_RETURNUNLOCKED`），并相对于其他锁持有者串行化 callout。
- 读取和解释 callout 的 `c_iflags` 和 `c_flags` 字段，并正确使用 `callout_pending`、`callout_active` 和 `callout_deactivate`。
- 使用 `callout_stop` 在正常驱动程序代码中取消待处理 callout，使用 `callout_drain` 在拆除时等待正在飞行的回调完成。
- 识别卸载竞争（callout 在 `kldunload` 后触发并使内核崩溃），并描述标准治疗方法：在 detach 时排空，在设备安静之前拒绝 detach。
- 将 `is_attached` 模式（我们在第 12 章中为 cv 等待者构建的）应用于 callout 回调，以便在拆除期间触发的回调干净地返回而不重新调度。
- 构建检测卡住条件并对其采取行动的看门狗定时器。
- 构建忽略快速重复事件的防抖定时器。
- 构建向 cbuf 注入合成数据用于测试的周期性 tick 源。
- 针对包含定时器活动的长时间运行压力测试，使用 `WITNESS`、`lockstat(1)` 验证启用 callout 的驱动程序。
- 用"Callouts"部分扩展 `LOCKING.md`，命名每个 callout、其回调、其锁和其生存期。
- 将驱动程序重构为定时器代码分组、命名且明显安全可维护的形式。

### 本章不涵盖的内容

几个相关主题被有意推迟：

- **任务队列（`taskqueue(9)`）**。第 16 章介绍内核的通用延迟工作框架。Taskqueue 和 callout 是互补的：callout 在特定时间运行函数；taskqueue 在工作线程可以接收时尽快运行函数。许多驱动程序同时使用两者：callout 在正确时刻触发，callout 将任务入队，任务在允许休眠的进程上下文中运行实际工作。第 13 章为简单起见留在 callout 自己的回调内部；延迟工作模式属于第 16 章。
- **硬件中断处理程序**。第 14 章介绍中断。真正的驱动程序可能安装在没有进程上下文中运行的中断处理程序。`callout(9)` 在锁类周围的规则与中断处理程序的规则相似（你可能不能休眠），但框架不同。我们将在第 14 章重新审视定时器和中断的交互。
- **`epoch(9)`**。网络驱动程序使用的读多同步框架。超出第 13 章范围。
- **高分辨率事件调度**。内核公开 `sbintime_t` 和 `_sbt` 变体用于亚 tick 精度；我们简要触及基于 sbintime 的 callout API 变体，但事件定时器驱动程序（`/usr/src/sys/kern/kern_clocksource.c`）的完整故事属于内核内部书籍，而不是驱动程序书籍。
- **实时和截止时间调度**。超出范围。我们依赖通用调度器。
- **通过调度器 tick（`hardclock`）的周期性工作负载**。内核本身使用 `hardclock(9)` 进行系统范围的周期性工作；驱动程序不直接与 `hardclock` 交互。我们为上下文提到它。

保持在这些界限内使章节保持专注。第 13 章的读者应该以自信地掌握 `callout(9)` 和何时转向 `taskqueue(9)` 的工作感觉完成。第 14 章和第 16 章填充其余部分。

### 预计时间投入

- **仅阅读**：约三小时。API 表面很小，但锁和生存期规则值得仔细关注。
- **阅读加上输入实际示例**：六到八小时，分两次会话。驱动程序在四个小阶段中演进；每个阶段添加一个定时器模式。
- **阅读加上所有实验和挑战**：十到十四小时，分三到四次会话，包括定时器活动下的压力测试和 `lockstat` 测量时间。

如果你在第 5 节（锁上下文规则）中间感到困惑，那是正常的。callout 和锁之间的交互是 API 中最令人惊讶的部分，即使是有经验的内核程序员偶尔也会弄错。停下来，重新阅读第 5 节的实际示例，在模型稳定后继续。

### 先决条件

在开始本章之前，确认：

- 你的驱动程序源代码与第 12 章第 4 阶段（`stage4-final`）匹配。起点假设 cv 通道、有界读取、sx 保护的配置和 reset sysctl 都已就位。
- 你的实验机器运行 FreeBSD 14.3，磁盘上有 `/usr/src` 并且与运行内核匹配。
- 已经构建、安装并干净启动的带有 `INVARIANTS`、`WITNESS`、`WITNESS_SKIPSPIN`、`DDB`、`KDB` 和 `KDB_UNATTENDED` 的调试内核。
- 你仔细阅读了第 12 章。锁顺序纪律、cv 模式和快照-应用模式是这里的假设知识。
- 你至少运行过一次第 12 章综合压力套件并看到它干净通过。

如果以上任何一项不稳定，现在修复比从一个移动基础上推进第 13 章并尝试调试是更好的投资。

### 本章路线图

各节的顺序是：

1. 为什么在驱动程序中使用定时器。时间在真实驱动程序工作中何时进入画面，以及哪些模式映射到 `callout(9)` 而不是其他东西。
2. FreeBSD 的 `callout(9)` API 简介。结构、生命周期、四种初始化变体以及每种给你带来什么。
3. 调度一次性和重复事件。`callout_reset`、`callout_reset_sbt`、`callout_schedule` 和周期性回调重新武装模式。
4. 将定时器集成到驱动程序中。第一次重构：一个定期记录统计信息或触发合成事件的心跳 callout。
5. 处理定时器中的锁定和上下文。锁感知初始化、`CALLOUT_RETURNUNLOCKED` 和 `CALLOUT_SHAREDLOCK` 标志、callout 函数可以做什么和不可以做什么的规则。
6. 定时器清理和资源管理。卸载竞争、`callout_stop` 与 `callout_drain`、带定时器的标准 detach 模式。
7. 定时工作的用例和扩展。看门狗、防抖、周期性轮询、延迟重试、统计翻转，全部构架为你可以提升到其他驱动程序的小配方。
8. 重构和版本控制。干净的 `LOCKING.md` 扩展、提升的版本字符串、更新的更新日志和包括定时器相关测试的回归运行。

动手实验和挑战练习紧随其后，然后是故障排除参考、总结部分和通往第 14 章的桥梁。

如果这是你第一次阅读，请线性阅读并按顺序进行实验。如果你在重温，清理部分（第 6 节）和重构部分（第 8 节）可以独立阅读，适合单次阅读。



## 第 1 节：为什么在驱动程序中使用定时器

驱动程序所做的大部分工作都是反应式的。用户打开设备，open 处理程序运行。用户发起 `read(2)`，read 处理程序运行。信号到达，睡眠者被唤醒。内核将控制权交给驱动程序以响应世界所做的某事。每次调用都有明确的原因，一旦工作完成，驱动程序返回并等待下一个原因。

真实硬件并不总是配合这个模型。网卡可能需要每隔几秒发送一次心跳，即使没有其他事情发生，只是为了让另一端的交换机确信链路是活跃的。存储控制器可能需要每五百毫秒进行一次看门狗复位，否则它会假设主机已经离开并重置通道。USB 集线器轮询必须按时器进行，因为 USB 总线不会为驱动程序想要看到的那类状态变化产生中断。开发板上的按钮需要防抖，因为弹簧触点在用户只想要一次时会快速连续产生许多事件。重试暂时性故障的驱动程序应该退避，而不是紧密循环。

所有这些都是为将来调度代码的理由。内核为此有一个单一原语，`callout(9)`，第 13 章从头开始教授它。在进入 API 之前，本节设定概念舞台。我们看看"稍后"在驱动程序中意味着什么，"稍后"回调通常采用什么形式，以及 `callout(9)` 相对于驱动程序表达时间概念的其他方式处于什么位置。

### "稍后"的三种形态

想要在未来做某事的驱动程序代码属于三种形态之一。知道你处于哪种形态是选择正确原语的一半。

**一次性。**"从现在起 X 毫秒做一次这件事，然后忘记它。"示例：调度一个只有在接下来一秒内没有观察到活动时才会触发的看门狗超时；通过忽略随后五十毫秒内的所有按下来防抖按钮；推迟拆除步骤直到当前操作完成。回调运行一次，驱动程序不重新武装它。

**周期性。**"每隔 X 毫秒做一次这件事，直到我告诉你停止。"示例：轮询不产生中断的硬件寄存器；向对端发送心跳；刷新缓存值；采样传感器；轮换统计窗口。回调运行一次，然后为下一个间隔重新武装自己，并持续直到驱动程序停止它。

**有界等待。**"当条件 Y 变为真时做这件事，但如果 Y 在 X 毫秒内没有发生就放弃。"示例：等待硬件响应并超时；等待缓冲区排空或截止时间到达；允许 Ctrl-C 中断等待。我们在第 12 章用 `cv_timedwait_sig` 遇到了这种形态。驱动程序线程是等待的那一方，而不是回调。

`callout(9)` 是前两种形态的原语。第三种使用 `cv_timedwait_sig`（第 12 章）、带有非零 `timo` 的 `mtx_sleep`，或用于亚 tick 精度的 `_sbt` 变体之一。两者是互补的，不是替代品：许多驱动程序同时使用两者。有界等待挂起调用线程；callout 在延迟后在单独的上下文中运行。

### 真实世界的模式

简要浏览 `/usr/src/sys/dev/` 中驱动程序反复出现的模式。早期识别它们为你提供了本章其余部分的词汇。

**心跳。**每 N 毫秒触发一次并发出一些简单状态（计数器增量、日志行、网络包）的周期性 callout。用于调试和需要活跃信号的协议。

**看门狗。**在操作开始时调度的一次性 callout。如果操作正常完成，驱动程序取消 callout。如果操作挂起，callout 触发，驱动程序采取纠正措施（重置硬件、记录警告、杀死卡住的请求）。几乎每个存储和网络驱动程序至少有一个看门狗。

**防抖。**当事件到达时调度的一次性 callout。超时内的后续相同事件被忽略。当 callout 触发时，驱动程序对最近的事件采取行动。用于会抖动的硬件事件（机械开关、光学传感器）。

**轮询。**周期性读取硬件寄存器并根据值采取行动的 callout。当硬件不为驱动程序关心的事件产生中断，或中断太嘈杂而无法使用时使用。

**带退避的重试。**每次失败尝试后以增加的延迟调度的一次性 callout。第一次失败调度 10 毫秒重试；第二次调度 20 毫秒重试；依此类推。限制驱动程序在失败后打扰硬件的速率。

**统计翻转。**以固定间隔拍摄内部计数器快照、计算每间隔速率并存储在循环缓冲区中供以后检查的周期性 callout。

**延迟收割器。**在某些宽限期后完成拆除的一次性 callout。当对象因某些其他代码路径可能仍持有引用而无法立即释放时使用；callout 等待足够长的时间让这些引用排空，然后释放对象。

我们将在本章过程中在 `myfirst` 中实现前三个（心跳、看门狗、延迟 tick 源）。其他是相同的形态；一旦你知道了模式，变化就是机械的。

### 为什么不直接使用内核线程？

初学者的合理问题：为什么 `callout(9)` 是一个独立的 API？同样的效果能否由一个循环、睡眠并行动的内核线程实现？

原则上可以。在实践中，当 callout 可以胜任时，没有驱动程序应该使用内核线程。

内核线程是重量级资源。它有自己的栈（amd64 上通常为 16 KB）、自己的调度器条目、自己的优先级、自己的状态。为一个每秒只需要 10 微秒的周期性动作启动一个线程是浪费的：16 KB 内存加上调度器开销，只是为了唤醒、做微不足道的工作、再次睡眠。乘以许多驱动程序，内核最终会有数百个大部分空闲的线程。

callout 在静止时几乎不消耗资源。数据结构是几个指针和整数（参见 `/usr/src/sys/sys/_callout.h` 中的 `struct callout`）。没有线程，没有栈，没有调度器条目。内核的硬件时钟中断遍历 callout 轮并运行每个到期的 callout，然后返回。数千个待处理的 callout 在触发前几乎不消耗任何资源。

callout 也插入内核现有的可观察性工具。`dtrace`、`lockstat` 和 `procstat` 都理解 callout。自定义内核线程没有任何免费的东西；你必须自己检测它。

当然，例外情况是定时器需要做的工作确实很长，并且会受益于处于可睡眠的线程上下文中。callout 函数可能不会睡眠；如果你的工作需要睡眠，callout 的工作是*将工作入队*到 taskqueue 或唤醒可以安全执行它的内核线程。第 16 章涵盖该模式。对于第 13 章，callout 所做的工作是短小、非睡眠且锁感知的。

### 为什么不直接在循环中使用 `cv_timedwait`？

另一个合理的替代方案：一个在 `cv_timedwait_sig` 上循环的内核线程也会产生周期性行为。一个每秒打开设备一次并触发 sysctl 的用户空间助手也是如此。为什么用 callout？

内核线程的答案是上一小节的资源论点：callout 比线程便宜得多。

用户空间助手的答案是正确性：一个时序依赖于用户空间进程的驱动程序是一个在该进程崩溃、被换出或被无关工作负载拒绝 CPU 时会失败的驱动程序。驱动程序应该对自己的正确性自给自足，即使用户空间工具在上面提供额外功能。

有一种情况 `cv_timedwait_sig` 是正确的答案：当*调用线程本身*需要等待时。第 12 章的 `read_timeout_ms` sysctl 使用 `cv_timedwait_sig` 因为读取者是等待的一方；一旦数据到达或截止时间触发，它就有工作要做。callout 是错误的，因为读取者的 syscall 线程不可能是运行回调的那一方（回调在不同上下文中运行）。

当 syscall 线程等待时使用 `cv_timedwait_sig`。当必须在特定时间发生某些独立于任何 syscall 线程的事情时使用 `callout(9)`。两者可以在同一驱动程序中舒适地共存；第 13 章将以一个使用两者的驱动程序结束。

### 关于时间本身的简要说明

内核通过几种单位公开时间，每种都有自己的约定。我们在第 12 章遇到过它们；在深入研究 API 之前回顾一下有帮助。

- **`int` ticks。** 传统单位。`hz` ticks 等于一秒。FreeBSD 14.3 上的默认 `hz` 是 1000，所以一个 tick 是一毫秒。`callout_reset` 以 ticks 为单位接受其延迟。
- **`sbintime_t`。** 一个 64 位有符号二进制定点表示：高 32 位是秒，低 32 位是秒的小数部分。单位常量在 `/usr/src/sys/sys/time.h` 中：`SBT_1S`、`SBT_1MS`、`SBT_1US`、`SBT_1NS`。`callout_reset_sbt` 以 sbintime 为单位接受其延迟。
- **`tick_sbt`。** 一个全局变量，保存 `1 / hz` 作为 sbintime。当你有 tick 计数并想要等效的 sbintime 时很有用：`tick_sbt * timo_in_ticks`。
- **精度参数。** `callout_reset_sbt` 接受一个额外的 `precision` 参数。它告诉内核在调度时可接受多少摆动，这让 callout 子系统可以合并附近的定时器以实现电源效率。精度为零意味着"尽可能接近截止时间触发"。精度为 `SBT_1MS` 意味着"截止时间一毫秒内的任何位置都可以"。

对于大多数驱动程序工作，基于 tick 的 API 是正确的精度级别。我们在本章早期部分使用 `callout_reset`（ticks），只有当需要亚毫秒精度或我们想告诉内核关于可接受的偏差时才使用 `callout_reset_sbt`。

### 何时 Callout 是错误的工具

为了完整性，三种 `callout(9)` *不是*正确答案的情况。

- **工作需要睡眠。** Callout 函数在可能不睡眠的上下文中运行。如果工作涉及 `uiomove`、`copyin`、`malloc(M_WAITOK)` 或任何其他可能阻塞的调用，callout 必须将任务入队到 taskqueue 或唤醒可以在进程上下文中执行工作的内核线程。第 16 章。
- **工作需要因为缓存原因在特定 CPU 上运行。** `callout_reset_on` 让你将 callout 绑定到特定 CPU，这很有用，但如果要求是"在提交请求的同一 CPU 上运行"，答案可能是每 CPU 原语。我们简要触及 `callout_reset_on` 并推迟更深入的 CPU 亲和性讨论。
- **工作是事件驱动的，而不是时间驱动的。** 如果触发是"数据到达"而不是"100 毫秒已过"，你想要的是 cv 或 wakeup，而不是 callout。混合两者通常导致不必要的复杂性。

### 心智模型：Callout 轮

为了让前几小节的成本论点具体化，这里是内核实际为管理 callout 所做的事情。你不需要这个来正确使用 `callout(9)`，但知道它使后面的几个部分更容易理解。

内核为每个 CPU 维护一个 *callout 轮*。概念上，轮是一个桶的循环数组。每个桶对应一个小的时间范围。当你调用 `callout_reset(co, ticks_in_future, fn, arg)` 时，内核计算"现在加上 ticks_in_future"落入哪个桶，并将 callout 添加到该桶的列表中。算术是 `(current_tick + ticks_in_future) modulo wheel_size`。

周期性定时器中断（硬件时钟）在每个 tick 触发。中断处理程序增加全局 tick 计数器，查看每个 CPU 轮的当前桶，并遍历列表。对于每个已达到截止时间的 callout，内核将其从轮上拉下并内联运行回调（对于 `C_DIRECT_EXEC` callout）或将其交给 callout 处理线程。

此机制的三个属性对章节很重要。

首先，调度 callout 很便宜：本质上是"计算桶索引并将结构链接到列表中"。几次原子操作。没有分配。没有上下文切换。

其次，未调度的 callout 不消耗任何东西：它只是你的 softc 中某处的 `struct callout`。内核在你调用 `callout_reset` 之前不知道它。静止时没有每 callout 的开销。

第三，轮的粒度是一个 tick。1.7-tick 的延迟向上舍入到 2 ticks。`callout_reset_sbt` 的 `precision` 参数让你用精确性换取内核合并附近触发的自由，这是在有许多并发定时器的系统上的省电优化。对于驱动程序工作，默认精度几乎总是可以的。

实际实现中还有更多内容：用于缓存局部性的每 CPU 轮、当 callout 重新调度到不同 CPU 时的延迟迁移、对在定时器中断本身中运行的 `C_DIRECT_EXEC` callout 的特殊处理等。实现在 `/usr/src/sys/kern/kern_timeout.c` 中，如果你好奇的话。读一次是值得的；你不需要记住它。

### 内核中"现在"的含义

一个微小但反复出现的困惑：内核中有几个时间基准，它们测量不同的东西。

`ticks` 是一个全局变量，计算自启动以来的硬件时钟中断数。每个时钟 tick 增加一。它读取很快（一次内存加载），在典型的 `hz=1000` 下每隔几周回绕一次，是 `callout_reset` 使用的时间基准。始终将 callout 截止时间表示为"现在加上 N ticks"，这就是 `callout_reset(co, N, ...)` 所做的。

`time_uptime` 和 `time_second` 是 `time_t` 值，计算自启动以来或自纪元以来的秒数（分别）。精度较低；用于日志时间戳和人类可读的经过时间。

`sbinuptime()` 返回一个表示自启动以来秒和小数的 `sbintime_t`。这是 `callout_reset_sbt` 工作的时间基准。它不会回绕（嗯，它会在几百年后回绕）。

`getmicrouptime()` 和 `getnanouptime()` 是"现在"的粗略但快速访问器；它们可能落后一两个 tick。`microuptime()` 和 `nanouptime()` 精确但更昂贵（它们直接读取硬件定时器）。

对于做典型定时器工作的驱动程序，`ticks`（用于基于 tick 的 callout 工作）和 `getsbinuptime()`（用于基于 sbintime 的工作）是两个会用到的。我们在实验中使用它们而不加注释；如果你想知道它们来自哪里，这就是答案。

### 第 1 节总结

时间以三种形态进入驱动程序工作：一次性、周期性和有界等待。前两种正是 `callout(9)` 的用途；第三种是第 12 章 `cv_timedwait_sig` 的用途。真实世界的模式（心跳、看门狗、防抖、轮询、重试、翻转、延迟收割器）都是一次性或周期性 callout 的实例；识别它们让你在许多情况下重用相同的原语。

在 API 之下，内核维护一个每 CPU callout 轮，在静止时几乎不消耗资源，调度成本也很低。粒度是一个 tick（典型 FreeBSD 14.3 系统上为一毫秒）。实现可以毫不费力地处理每个 CPU 数千个待处理的 callout。

第 2 节介绍 API：callout 结构、四种初始化变体以及每个 callout 遵循的生命周期。


## 第 2 节：FreeBSD 的 `callout(9)` API 简介

`callout(9)` 与大多数同步原语一样，是精心实现之上的一个小型 API。数据结构很短，生命周期是规则的（初始化、调度、触发、停止或排空、销毁），规则足够明确，你可以通过阅读源代码来验证你的使用。本节遍历结构，命名初始化变体，并排列生命周期阶段，以便本章其余部分有可重用的词汇。

### Callout 结构

数据结构位于 `/usr/src/sys/sys/_callout.h`：

```c
struct callout {
        union {
                LIST_ENTRY(callout) le;
                SLIST_ENTRY(callout) sle;
                TAILQ_ENTRY(callout) tqe;
        } c_links;
        sbintime_t c_time;       /* ticks to the event */
        sbintime_t c_precision;  /* delta allowed wrt opt */
        void    *c_arg;          /* function argument */
        callout_func_t *c_func;  /* function to call */
        struct lock_object *c_lock;   /* lock to handle */
        short    c_flags;        /* User State */
        short    c_iflags;       /* Internal State */
        volatile int c_cpu;      /* CPU we're scheduled on */
};
```

每个 callout 一个结构，嵌入在 softc 或你需要它的任何其他地方。你直接接触的字段是：无。每次交互都通过 API 调用。你可以读取用于诊断目的的字段是 `c_flags`（通过 `callout_active` / `callout_pending`）和 `c_arg`（从外部很少有用）。

两个标志字段各值得一句话。

`c_iflags` 是内部的。内核在 callout 子系统自己的锁下设置和清除其中的位。这些位编码 callout 是否在轮上或处理列表上、是否待处理以及少量内部簿记状态。驱动程序代码使用 `callout_pending(c)` 读取它；其他不做什么。

`c_flags` 是外部的。调用者（你的驱动程序）应该管理其中的两个位：`CALLOUT_ACTIVE` 和 `CALLOUT_RETURNUNLOCKED`。活跃位用于跟踪"我已经要求调度此 callout 并且尚未取消它"。returnunlocked 位更改锁处理契约；我们将在第 5 节讨论。驱动程序代码通过 `callout_active(c)` 读取活跃位，通过 `callout_deactivate(c)` 清除它。

`c_lock` 字段值得单独一段。当你用 `callout_init_mtx`、`callout_init_rw` 或 `callout_init_rm` 初始化 callout 时，内核在这里记录锁指针。稍后，当 callout 触发时，内核在调用你的回调函数之前获取该锁并在回调返回后释放它（除非你特别要求否则）。这意味着你的回调运行起来就像调用者已经为你获取了锁一样。锁管理的 callout 几乎总是驱动程序代码想要的；我们将在第 5 节说更多。

### 回调函数签名

callout 的回调函数有一个参数：`void *`。内核传递你用 `callout_reset`（或其变体）注册的任何内容。函数返回 `void`。它的完整签名，来自 `/usr/src/sys/sys/_callout.h`：

```c
typedef void callout_func_t(void *);
```

约定：传递一个指向你的 softc 的指针（或回调需要的任何每实例状态）。回调的第一行将 void 指针转换回 struct 指针：

```c
static void
myfirst_heartbeat(void *arg)
{
        struct myfirst_softc *sc = arg;
        /* ... do timer work ... */
}
```

参数在注册时固定，在触发之间不会改变。如果你需要向回调传递变化的上下文，将它存储在回调可以通过 softc 找到的某个地方。

### 四种初始化变体

`callout(9)` 提供四种初始化 callout 的方式，通过内核在回调运行之前为你获取的锁类型（如果有）来区分：

```c
void  callout_init(struct callout *c, int mpsafe);

#define callout_init_mtx(c, mtx, flags) \
    _callout_init_lock((c), &(mtx)->lock_object, (flags))
#define callout_init_rw(c, rw, flags) \
    _callout_init_lock((c), &(rw)->lock_object, (flags))
#define callout_init_rm(c, rm, flags) \
    _callout_init_lock((c), &(rm)->lock_object, (flags))
```

`callout_init(c, mpsafe)` 是传统的、锁无感知的变体。`mpsafe` 参数现在命名不当；它实际上意味着"可以在不为我获取 Giant 的情况下运行"。对于任何现代驱动程序代码传递 `1`；只有在你确实希望内核在回调之前获取 Giant 时才传递 `0`（几乎从不，只存在于非常旧的代码路径中）。新驱动程序不应使用此变体。本章提到它是为了完整性，因为你会在旧代码中看到它。

`callout_init_mtx(c, mtx, flags)` 注册一个睡眠互斥锁（`MTX_DEF`）作为 callout 的锁。每次触发之前，内核获取互斥锁并在回调返回后释放它。这是你几乎在所有驱动程序代码中使用的变体。它与你已经在数据路径上的 `MTX_DEF` 互斥锁自然配对。

`callout_init_rw(c, rw, flags)` 注册一个 `rw(9)` 读写锁。除非你设置了 `CALLOUT_SHAREDLOCK`，内核获取写锁，在这种情况下它获取读锁。在驱动程序代码中不太常见；当回调需要读取某个读多状态且多个 callout 共享同一锁时有用。

`callout_init_rm(c, rm, flags)` 注册一个 `rmlock(9)`。专用的；用于具有热读路径且不应竞争的网络驱动程序。

对于 `myfirst` 驱动程序，我们添加的每个 callout 都将使用 `callout_init_mtx(&sc->some_co, &sc->mtx, 0)`。内核在回调运行之前获取 `sc->mtx`，回调可以操作 cbuf 和其他互斥锁保护的状态而无需自己获取锁，内核在之后释放 `sc->mtx`。模式是干净的，规则是明确的，如果你违反它们，`WITNESS` 会大喊。

### flags 参数

`_callout_init_lock` 的 flags 参数对于驱动程序代码是以下两个值之一：

- `0`：callout 的锁在回调之前获取并在之后释放。这是默认值，几乎总是正确的答案。
- `CALLOUT_RETURNUNLOCKED`：callout 的锁在回调之前获取。回调负责释放它（或者回调调用的某个东西可能已经释放了它）。当回调的最后动作是丢弃锁并做一些锁不能覆盖的事情时偶尔有用。
- `CALLOUT_SHAREDLOCK`：只对 `callout_init_rw` 和 `callout_init_rm` 有效。锁以共享模式而非独占模式获取。

对于第 13 章，我们到处使用 `0`。`CALLOUT_RETURNUNLOCKED` 在第 5 节为完整性而提及；本章不需要它。

### 五个生命周期阶段

每个 callout 遵循相同的五阶段生命周期。按名称了解这些阶段将使本章其余部分更容易阅读。

**阶段 1：已初始化。** `struct callout` 已用某个初始化变体初始化。它有一个锁关联（或 `mpsafe`）。它尚未被调度。在你告诉它之前不会触发任何东西。

**阶段 2：待处理。** 你已经调用了 `callout_reset` 或 `callout_reset_sbt`。内核已将 callout 放在其内部轮上并记录了它应该触发的时间。`callout_pending(c)` 返回 true。回调尚未运行。你可以通过调用 `callout_stop(c)` 来取消，这会将其从轮上移除。

**阶段 3：触发中。** 截止时间已到，内核正在运行回调。如果 callout 有注册的锁，内核已获取它。你的回调函数正在执行。在此阶段 `callout_active(c)` 为 true，`callout_pending(c)` 可能为 false（它已从轮上移除）。回调可以自由调用 `callout_reset` 来重新武装自己（这是周期性模式）。

**阶段 4：已完成。** 回调已返回。如果回调通过 `callout_reset` 重新武装，callout 回到阶段 2。否则它现在空闲：`callout_pending(c)` 为 false。如果内核为回调获取了锁，它已释放它。

**阶段 5：已销毁。** callout 的底层内存不再需要。没有 `callout_destroy` 函数；相反，你必须确保 callout 不待处理且不在触发中，然后释放包含它的结构。"等待 callout 安全地变为空闲"的标准工具是 `callout_drain`。第 6 节详细讨论这个。

循环是：初始化一次，在待处理和（触发+完成）之间交替任意次数，排空，释放。

### API 初步了解

我们还没有调度任何东西。让我们阅读四个最重要的调用，每个都有一行摘要：

```c
int  callout_reset(struct callout *c, int to_ticks,
                   void (*fn)(void *), void *arg);
int  callout_reset_sbt(struct callout *c, sbintime_t sbt,
                   sbintime_t prec, void (*fn)(void *), void *arg, int flags);
int  callout_stop(struct callout *c);
int  callout_drain(struct callout *c);
```

`callout_reset` 调度 callout。第一个参数是要调度的 callout。第二个是以 ticks 为单位的延迟（乘以 `hz` 来转换；在 FreeBSD 14.3 上，`hz=1000` 通常，所以一个 tick 是一毫秒）。第三个是回调函数。第四个是传递给回调的参数。如果 callout 之前待处理并被取消（所以新调度替换旧的），返回非零。

`callout_reset_sbt` 相同，但以 `sbintime_t` 接受延迟并接受精度和标志。用于亚 tick 精度或当你想告诉内核关于可接受的偏差时。大多数驱动程序使用 `callout_reset`，只在需要时才使用 `_sbt`。

`callout_stop` 取消待处理的 callout。如果 callout 待处理，它从轮上移除且永不触发。如果 callout 不待处理（已触发或从未调度），调用是空操作并返回零。关键的是：`callout_stop` *不*等待正在执行的回调完成。如果回调当前正在另一个 CPU 上执行，`callout_stop` 在回调返回之前返回。

`callout_drain` 是安全的拆除变体。它取消待处理的 callout，*并且*等待任何当前正在执行的回调返回后才自己返回。`callout_drain` 返回后，callout 保证空闲且不在任何地方运行。这是你在 detach 时调用的函数。第 6 节详细讲解为什么这很重要。

### 阅读源代码

如果你有十分钟，打开 `/usr/src/sys/sys/callout.h` 和 `/usr/src/sys/kern/kern_timeout.c` 并浏览。三件要看的东西：

头文件在不到 130 行中定义了公共 API。本章提到的每个函数都在那里声明。包装 `_callout_init_lock` 的宏清晰可见。

实现文件很长（FreeBSD 14.3 中约 1550 行），但函数名与 API 匹配。`callout_reset_sbt_on` 是核心调度函数；其他一切都是包装。`_callout_stop_safe` 是统一的停止并可能排空函数；`callout_stop` 和 `callout_drain` 是用不同标志调用它的宏。`callout_init` 和 `_callout_init_lock` 位于文件底部附近。

本章按名称而不是行号引用 FreeBSD 函数和表，因为行号在版本之间会漂移，而函数和符号名会保留。如果你需要 FreeBSD 14.3 中 `kern_timeout.c` 的大约行号：`callout_reset_sbt_on` 在 936 附近，`_callout_stop_safe` 在 1085 附近，`callout_init` 在 1347 附近。打开文件并跳转到符号；行号是你的编辑器报告的任何内容。

散布在源代码中的 KASSERT 是代码形式的规则。例如，`_callout_init_lock` 中"你不能给我一个可睡眠的锁"的断言强制执行 callout 可能不会阻塞在可能睡眠的锁上的规则。阅读这些断言建立对 API 保证其所说内容的信心。

### 一个演练的生命周期演练

将生命周期阶段放在时间线上使它们具体化。想象一个在 attach 时初始化、在 t=0 时启用、在 t=2.5 秒时禁用的心跳 callout。

- **t=-1s（attach 时间）**：驱动程序调用 `callout_init_mtx(&sc->heartbeat_co, &sc->mtx, 0)`。callout 现在处于阶段 1（已初始化）。`callout_pending(c)` 返回 false。内核知道 callout 的锁关联。
- **t=0s**：用户通过写入 sysctl 启用心跳。处理程序获取 `sc->mtx`，设置 `interval_ms = 1000`，并调用 `callout_reset(&sc->heartbeat_co, hz, myfirst_heartbeat, sc)`。callout 转换到阶段 2（待处理）。`callout_pending(c)` 返回 true。内核已将其放入对应 t+1 秒的轮桶中。
- **t=1s**：截止时间到达。内核将 callout 从轮上拉下（`callout_pending(c)` 变为 false）。内核获取 `sc->mtx`。内核调用 `myfirst_heartbeat(sc)`。callout 现在处于阶段 3（触发中）。回调运行，发出一条日志行，调用 `callout_reset` 来重新武装。重新武装将 callout 放回对应 t+2 秒的轮桶中。`callout_pending(c)` 再次为 true。回调返回。内核释放 `sc->mtx`。callout 现在回到阶段 2（待处理），等待下一次触发。
- **t=2s**：相同序列。回调触发，重新武装，callout 待处理直到 t+3 秒。
- **t=2.5s**：用户通过写入 sysctl 禁用心跳。处理程序获取 `sc->mtx`，设置 `interval_ms = 0`，并调用 `callout_stop(&sc->heartbeat_co)`。内核将 callout 从轮上移除。`callout_stop` 返回 1（它取消了一个待处理的 callout）。`callout_pending(c)` 变为 false。callout 现在回到阶段 1（已初始化但空闲）。
- **t=无穷（稍后，在 detach 时间）**：detach 路径调用 `callout_drain(&sc->heartbeat_co)`。callout 已经空闲；`callout_drain` 立即返回。驱动程序现在可以安全地释放周围的状态。

关于时间线注意三件事。

待处理到触发到待处理的循环只要回调重新武装就无限重复。迭代次数没有硬限制。

`callout_stop` 可以在任何点拦截待处理-触发-待处理循环。如果 callout 处于阶段 2（待处理），`callout_stop` 取消它。如果 callout 在另一个 CPU 上处于阶段 3（触发中），`callout_stop` *不*取消它（回调将运行到完成）；循环的下一次迭代不会发生，因为回调的重新武装条件（`interval_ms > 0`）现在为 false。

回调中的 `is_attached` 检查（我们将在第 4 节介绍）在拆除期间提供了类似的拦截点。如果回调在 detach 清除了 `is_attached` 之后触发，回调退出而不重新武装，下一次迭代不会发生。

这个时间线是驱动程序代码中 `callout(9)` 使用的整个形态。变化包括添加一次性模式（不重新武装）、看门狗模式（成功时取消）或防抖模式（仅在未待处理时调度）。生命周期阶段是相同的。

### 关于"活跃"与"待处理"

初学者有时混淆的两个相关概念。

`callout_pending(c)` 在 callout 在轮上等待触发时由内核设置。在 callout 触发（回调即将运行）或 `callout_stop` 取消它时由内核清除。

`callout_active(c)` 在 `callout_reset` 成功时由内核设置。由 `callout_deactivate`（你调用的函数）或 `callout_stop` 清除。关键是，内核在回调触发时*不*清除 `callout_active`。活跃位是一个标志，表示"我调度了这个 callout 并且没有主动取消它"；自那以后回调是否触发过是一个单独的问题。

callout 可以处于以下四种状态之一：

- 不活跃且不待处理：从未调度，或通过 `callout_stop` 取消，或在触发后通过 `callout_deactivate` 取消活跃。
- 活跃且待处理：已调度，在轮上，等待触发。
- 活跃且不待处理：已调度，已触发（或即将触发），回调尚未调用 `callout_deactivate`。
- 不活跃且待处理：罕见，但如果驱动程序在 callout 仍被调度时调用 `callout_deactivate` 则可能。大多数驱动程序从不达到此状态，因为它们只在回调内部调用 `callout_deactivate`，在待处理位已经被清除之后。

对于大多数驱动程序，你只需要 `callout_pending`（在防抖等模式中使用）。`active` 标志在想知道"我们是否调度了一个 callout，即使它已经运行了？"的代码中更重要。对于第 13 章，我们使用一次 `pending`，从不使用 `active`。

### 第 2 节总结

Callout 是小型结构，具有小型 API 和规则的生命周期。四种初始化变体选择内核为你获取的锁类型（或无）。你最常用的四个函数是 `callout_reset`、`callout_reset_sbt`、`callout_stop` 和 `callout_drain`。第 3 节将它们投入使用，调度一次性和周期性定时器并展示周期性重新武装模式的实际工作方式。


## 第 3 节：调度一次性和重复事件

`callout(9)` 中的定时器总是概念上一次性的。没有 `callout_reset_periodic` 函数。周期性行为是通过让回调在每次触发结束时重新武装自己来构建的。一次性和周期性模式都使用相同的 API 调用（`callout_reset`）；区别在于回调是否决定调度下一次触发。

本节通过可编译和运行的实际示例遍历两种模式。我们还没有将它们集成到 `myfirst` 中；那是第 4 节。这里我们专注于时序原语和你将使用的模式。

### 一次性模式

最简单的可能 callout：调度一个回调在未来一次性触发。

```c
static void
my_oneshot(void *arg)
{
        device_printf((device_t)arg, "one-shot fired\n");
}

void
schedule_a_one_shot(device_t dev, struct callout *co)
{
        callout_reset(co, hz / 10, my_oneshot, dev);
}
```

`hz / 10` 在 `hz=1000` 的系统上意味着"从现在起 100 毫秒"。回调接收我们注册的设备指针。它运行一次，打印，然后返回。callout 现在空闲。要再次运行它，你需要再次调用 `callout_reset`。

注意三件事。首先，回调的参数是我们传递给 `callout_reset` 的任何东西，无类型，用转换恢复。其次，回调发出一条日志行然后返回；它不重新调度。这是一次性模式。第三，我们使用 `hz / 10` 而不是硬编码值。始终以 `hz` 表示 callout 延迟，以便代码可移植到具有不同时钟速率的系统。

如果你想要 250 毫秒的延迟，你会写 `hz / 4`（或 `hz * 250 / 1000` 以清晰）。对于 5 秒的延迟，`hz * 5`。算术是整数；对于小数值，先乘后除以保持精度。

### 周期性模式

对于周期性行为，回调在结束时重新武装自己：

```c
static void
my_periodic(void *arg)
{
        struct myfirst_softc *sc = arg;
        device_printf(sc->dev, "tick\n");
        callout_reset(&sc->heartbeat_co, hz, my_periodic, sc);
}

void
start_periodic(struct myfirst_softc *sc)
{
        callout_reset(&sc->heartbeat_co, hz, my_periodic, sc);
}
```

第一次调用 `callout_reset`（在 `start_periodic` 中）将 callout 调度到从现在起一秒。当它触发时，`my_periodic` 运行，发出一条日志行，并重新武装到当前时刻一秒后。下一次触发发生，循环继续。要停止周期性触发，调用 `callout_stop(&sc->heartbeat_co)`（或在拆除时使用 `callout_drain`）。一旦 callout 被停止，`my_periodic` 将不会再次触发，直到再次调用 `start_periodic`。

三个微妙之处。

首先，重新武装发生在回调的*末尾*。如果回调的工作花费很长时间，下一次触发会被该工作延迟。触发之间的实际间隔大约是 `hz` ticks 加上回调花费的时间。对于大多数驱动程序使用情况，这是可以的。如果你需要精确的周期，使用 `callout_schedule` 或 `callout_reset_sbt` 配合计算的绝对截止时间。

其次，回调是在 callout 的锁已获取的情况下被调用的（我们将在第 5 节看到原因）。当回调调用 `callout_reset` 重新武装时，callout 子系统正确处理重新武装，即使它是在同一个 callout 的触发内部被调用的。内核的内部簿记正是为这种模式设计的。

第三，如果驱动程序正在被拆除的同时回调重新武装，你有一个竞争：重新武装将 callout 放回轮上，在取消/排空已经运行之后。第 6 节解释如何处理这个。简短的答案是：在 detach 时间，在互斥锁下在 softc 中设置一个"正在关闭"标志，然后 `callout_drain` callout。回调在入口处检查标志，如果看到标志被设置则返回而不重新武装。排空等待正在飞行的回调返回。

### `callout_schedule` 用于无需重复参数的重新武装

对于周期性 callout，回调每次用相同的函数和参数重新武装。`callout_reset` 要求你再次传递它们。`callout_schedule` 是一个便利函数，使用上次 `callout_reset` 的函数和参数：

```c
int  callout_schedule(struct callout *c, int to_ticks);
```

在周期性回调内部：

```c
static void
my_periodic(void *arg)
{
        struct myfirst_softc *sc = arg;
        device_printf(sc->dev, "tick\n");
        callout_schedule(&sc->heartbeat_co, hz);
}
```

内核使用它从上次 `callout_reset` 调用记住的函数指针和参数。更少的输入，代码读起来稍微干净。`callout_reset` 和 `callout_schedule` 都可以用于周期性模式；选择你喜欢的那个。

### 使用 `callout_reset_sbt` 的亚 Tick 精度

当你需要比一个 tick 更精细的精度，或你想告诉内核关于可接受的偏差时，使用 sbintime 变体：

```c
int  callout_reset_sbt(struct callout *c, sbintime_t sbt,
                       sbintime_t prec,
                       void (*fn)(void *), void *arg, int flags);
```

示例：调度一个 250 微秒的定时器：

```c
sbintime_t sbt = 250 * SBT_1US;
callout_reset_sbt(&sc->fast_co, sbt, SBT_1US,
    my_callback, sc, C_HARDCLOCK);
```

`prec` 参数是调用者愿意接受的精度。`SBT_1US` 表示"截止时间一微秒内的任何位置都可以"；内核可能会将此定时器与相差一微秒的其他定时器合并。`0` 表示"尽可能接近截止时间触发"。标志包括 `C_HARDCLOCK`（与系统时钟中断对齐，大多数情况的默认值）、`C_DIRECT_EXEC`（在定时器中断上下文中运行，只对自旋锁有用）、`C_ABSOLUTE`（将 `sbt` 解释为绝对时间而不是相对延迟）和 `C_PRECALC`（内部使用；不要设置它）。

对于本章，我们几乎到处使用 `callout_reset`（基于 tick）。`callout_reset_sbt` 为完整性而提及；实验部分有一个使用它的练习。

### 取消：`callout_stop`

要取消待处理的 callout，调用 `callout_stop`：

```c
int  callout_stop(struct callout *c);
```

如果 callout 待处理，内核将其从轮上移除并返回 1。如果 callout 不待处理（已触发或从未调度），调用是空操作并返回 0。

关键是：`callout_stop` *不*等待。如果回调当前正在另一个 CPU 上执行时调用 `callout_stop`，调用立即返回。回调继续在另一个 CPU 上运行并在它完成时完成。如果回调重新武装自己，callout 在 `callout_stop` 返回后回到轮上。

这意味着 `callout_stop` 是正常操作的正确工具（取消待处理的 callout 因为促成它的条件已解决），但它是拆除的*错误*工具（在释放周围状态之前你必须等待任何正在飞行的回调完成）。对于拆除，使用 `callout_drain`。第 6 节深入涵盖这个区别。

正常操作中的标准模式：

```c
/* Decided we don't need this watchdog any more */
if (callout_stop(&sc->watchdog_co)) {
        /* The callout was pending; we just cancelled it. */
        device_printf(sc->dev, "watchdog cancelled\n");
}
/* If callout_stop returned 0, the callout had already fired
   or was never scheduled; nothing to do. */
```

一个小注意点：在 `callout_stop` 返回 1 和下一条语句运行之间，没有其他线程可以重新武装 callout，因为我们持有保护周围状态的锁。没有锁，`callout_stop` 仍会正确取消，但返回值的含义会变得有竞争。

### 取消：`callout_drain`

`callout_drain` 是拆除安全的变体：

```c
int  callout_drain(struct callout *c);
```

像 `callout_stop`，它取消待处理的 callout。*不同于* `callout_stop`，如果回调当前正在另一个 CPU 上执行，`callout_drain` 在自己返回之前等待它返回。`callout_drain` 返回后，callout 保证空闲：不待处理，不触发中，并且（如果回调没有重新武装）它不会再次触发。

两个重要规则。

首先，`callout_drain` 的调用者*不得*持有 callout 的锁。如果 callout 当前正在执行（它已获取锁并正在运行回调），`callout_drain` 需要等待回调返回，这意味着回调需要释放锁，这意味着 `callout_drain` 的调用者不能持有它。持有锁会死锁。

其次，`callout_drain` 可能睡眠。线程在睡眠队列上等待回调完成。因此 `callout_drain` 只在允许睡眠的上下文中合法（进程上下文或内核线程；不是中断或自旋锁上下文）。

标准拆除模式：

```c
static int
myfirst_detach(device_t dev)
{
        struct myfirst_softc *sc = device_get_softc(dev);

        /* mark "going away" so a re-arming callback will not re-schedule */
        MYFIRST_LOCK(sc);
        sc->is_attached = 0;
        MYFIRST_UNLOCK(sc);

        /* drain the callout: cancel pending, wait for in-flight */
        callout_drain(&sc->heartbeat_co);
        callout_drain(&sc->watchdog_co);

        /* now safe to destroy other primitives and free state */
        /* ... */
}
```

第 6 节扩展此模式。

### `callout_pending` 和 `callout_active`

两个诊断访问器在你想知道 callout 处于什么状态时有用：

```c
int  callout_pending(const struct callout *c);
int  callout_active(const struct callout *c);
void callout_deactivate(struct callout *c);
```

`callout_pending(c)` 如果 callout 当前已调度并等待触发则返回非零。如果 callout 已触发（或从未被调度，或被取消）则返回 false。

`callout_active(c)` 如果自上次 `callout_deactivate` 以来在此 callout 上调用了 `callout_reset` 则返回非零。"活跃"位是*你*管理的。内核从不自己设置或清除它（有一个小例外：成功的 `callout_stop` 清除它）。约定是回调在开始时清除该位，驱动程序的其余部分在调度时设置它，想知道"我是否有一个待处理或刚触发的 callout？"的代码可以检查 `callout_active`。

对于大多数驱动程序工作你不需要任一访问器。我们提到它们是因为真实驱动程序源代码使用它们，你应该识别该模式。第 13 章 `myfirst` 驱动程序使用一次 `callout_pending`，在看门狗取消路径中；本章其余部分不需要它们。

### 实际示例：两阶段调度

将各个部分放在一起：一个小型实际示例，调度一个回调从现在起 100 毫秒触发，然后将其重新调度到 500 毫秒后，然后运行一次并停止。

```c
static int g_count = 0;
static struct callout g_co;
static struct mtx g_mtx;

static void
my_callback(void *arg)
{
        printf("callback fired (count=%d)\n", ++g_count);
        if (g_count == 1) {
                /* Reschedule for 500 ms later. */
                callout_reset(&g_co, hz / 2, my_callback, NULL);
        } else if (g_count == 2) {
                /* Done; do nothing, callout becomes idle. */
        }
}

void
start_test(void)
{
        mtx_init(&g_mtx, "test_co", NULL, MTX_DEF);
        callout_init_mtx(&g_co, &g_mtx, 0);
        callout_reset(&g_co, hz / 10, my_callback, NULL);
}

void
stop_test(void)
{
        callout_drain(&g_co);
        mtx_destroy(&g_mtx);
}
```

十行实质内容。回调根据计数决定是否重新武装。两次触发后，它停止重新武装，callout 变为空闲。`stop_test` 排空 callout（如有必要等待任何正在飞行的触发），然后销毁互斥锁。

此模式及其变化是驱动程序代码中 `callout(9)` 使用的整个形态。第 4 节将它放入 `myfirst` 并给它真正的工作。

### 第 3 节总结

Callout 使用 `callout_reset`（基于 tick）或 `callout_reset_sbt`（基于 sbintime）调度。一次性行为来自不重新武装的回调；周期性行为来自在结束时重新武装自己的回调。取消是正常操作用 `callout_stop`，拆除用 `callout_drain`。访问器 `callout_pending`、`callout_active` 和 `callout_deactivate` 用于诊断检查。

第 4 节采用本节的模式并将一个真实的 callout 集成到 `myfirst` 驱动程序中：一个定期记录统计信息行的心跳。


## 第 4 节：将定时器集成到驱动程序中

理论是舒适的；集成是粗糙边缘出现的地方。本节遍历将心跳 callout 添加到 `myfirst`。心跳每秒触发一次，记录一条简短的统计信息行，并重新武装自己。我们将看到 callout 如何与现有互斥锁集成，锁感知初始化如何消除一类竞争，第 12 章的 `is_attached` 标志如何在拆除期间保护回调，以及 `WITNESS` 如何确认设计正确。

将此视为本章驱动程序演进的第一阶段。在本节结束时，`myfirst` 驱动程序有了它的第一个定时器。

### 添加心跳 Callout

向 `struct myfirst_softc` 添加两个字段：

```c
struct myfirst_softc {
        /* ... existing fields ... */
        struct callout          heartbeat_co;
        int                     heartbeat_interval_ms;  /* 0 = disabled */
        /* ... rest ... */
};
```

`heartbeat_co` 是 callout 本身。`heartbeat_interval_ms` 是 sysctl 可调参数，让用户可以在运行时启用、禁用和调整心跳。值为零禁用心跳。正值是以毫秒为单位的间隔。

在 `myfirst_attach` 中初始化 callout。将调用放在互斥锁初始化之后、cdev 创建之前（这样 callout 可以调度但没有用户可以触发任何东西）：

```c
static int
myfirst_attach(device_t dev)
{
        /* ... existing setup ... */

        mtx_init(&sc->mtx, device_get_nameunit(dev), "myfirst", MTX_DEF);
        cv_init(&sc->data_cv, "myfirst data");
        cv_init(&sc->room_cv, "myfirst room");
        sx_init(&sc->cfg_sx, "myfirst cfg");
        callout_init_mtx(&sc->heartbeat_co, &sc->mtx, 0);

        /* ... rest of attach ... */
}
```

`callout_init_mtx(&sc->heartbeat_co, &sc->mtx, 0)` 注册 `sc->mtx` 作为 callout 的锁。从此时起，每当心跳 callout 触发，内核将在调用我们的回调之前获取 `sc->mtx` 并在回调返回后释放它。这正是我们想要的契约：回调可以自由操作 cbuf 状态和每 softc 字段而无需自己获取锁。

在 `myfirst_detach` 中排空 callout，在销毁原语之前：

```c
static int
myfirst_detach(device_t dev)
{
        struct myfirst_softc *sc = device_get_softc(dev);

        /* ... refuse detach while active_fhs > 0 ... */
        /* ... clear is_attached and broadcast cvs under sc->mtx ... */

        seldrain(&sc->rsel);
        seldrain(&sc->wsel);

        callout_drain(&sc->heartbeat_co);

        if (sc->cdev_alias != NULL) { destroy_dev(sc->cdev_alias); /* ... */ }
        /* ... rest of detach as before ... */
}
```

`callout_drain` 调用必须在 `is_attached` 清除和 cvs 广播之后（这样在排空期间触发的回调看到清除的标志），并且在回调可能触及的任何原语被销毁之前。清除的 `is_attached` 标志阻止回调重新调度；排空等待任何正在飞行的回调完成。`callout_drain` 返回后，没有回调可以运行且没有待处理的；其余的 detach 可以安全地释放状态。

### 心跳回调

现在回调本身：

```c
static void
myfirst_heartbeat(void *arg)
{
        struct myfirst_softc *sc = arg;
        size_t used;
        uint64_t br, bw;
        int interval;

        MYFIRST_ASSERT(sc);

        if (!sc->is_attached)
                return;  /* device going away; do not re-arm */

        used = cbuf_used(&sc->cb);
        br = counter_u64_fetch(sc->bytes_read);
        bw = counter_u64_fetch(sc->bytes_written);
        device_printf(sc->dev,
            "heartbeat: cb_used=%zu, bytes_read=%ju, bytes_written=%ju\n",
            used, (uintmax_t)br, (uintmax_t)bw);

        interval = sc->heartbeat_interval_ms;
        if (interval > 0)
                callout_reset(&sc->heartbeat_co,
                    (interval * hz + 999) / 1000,
                    myfirst_heartbeat, sc);
}
```

十行代码捕捉整个周期性心跳模式。让我们逐一查看。

`MYFIRST_ASSERT(sc)` 确认 `sc->mtx` 已持有。callout 用 `callout_init_mtx(&sc->heartbeat_co, &sc->mtx, 0)` 初始化，所以内核在调用我们之前获取了 `sc->mtx`；断言是一个健全性检查，捕获某人（可能是未来的维护者）意外地将初始化更改为 `callout_init` 而不注意的情况。

`if (!sc->is_attached) return;` 是拆除守卫。如果 detach 路径已清除 `is_attached`，我们立即退出，不做任何工作也不重新武装。detach 中的排空将看到 callout 空闲并干净地完成。

cbuf-used 和 counter 读取在锁下发生。我们调用 `cbuf_used`（期望 `sc->mtx` 持有）和 `counter_u64_fetch`（无锁且到处安全）。`device_printf` 调用可能昂贵但对于日志行是常规的；我们容忍成本因为它每秒至多发生一次。

末尾的重新武装使用 `heartbeat_interval_ms` 的当前值。如果用户已将其设置为零（禁用心跳），我们不重新武装，callout 变为空闲直到其他东西调度它。如果用户已更改间隔，下一次触发将使用新值。这是一个小但重要的功能：心跳的频率是动态可配置的，无需重启驱动程序。

`(interval * hz + 999) / 1000` 算术将毫秒转换为 ticks，向上舍入。与第 12 章有界等待相同的公式，同样的原因：永远不要向下舍入低于请求的持续时间。

### 从 Sysctl 启动心跳

用户通过向 `dev.myfirst.<unit>.heartbeat_interval_ms` 写入非零值来启用心跳。我们需要一个调度第一次触发的 sysctl 处理程序：

```c
static int
myfirst_sysctl_heartbeat_interval_ms(SYSCTL_HANDLER_ARGS)
{
        struct myfirst_softc *sc = arg1;
        int new, old, error;

        old = sc->heartbeat_interval_ms;
        new = old;
        error = sysctl_handle_int(oidp, &new, 0, req);
        if (error || req->newptr == NULL)
                return (error);

        if (new < 0)
                return (EINVAL);

        MYFIRST_LOCK(sc);
        sc->heartbeat_interval_ms = new;
        if (new > 0 && old == 0) {
                /* Enabling: schedule the first firing. */
                callout_reset(&sc->heartbeat_co,
                    (new * hz + 999) / 1000,
                    myfirst_heartbeat, sc);
        } else if (new == 0 && old > 0) {
                /* Disabling: cancel any pending heartbeat. */
                callout_stop(&sc->heartbeat_co);
        }
        MYFIRST_UNLOCK(sc);
        return (0);
}
```

处理程序：

1. 读取当前值（所以只读查询返回当前间隔）。
2. 让 `sysctl_handle_int` 验证并更新局部 `new` 变量。
3. 验证新值非负。
4. 获取 `sc->mtx` 以原子地提交更改，对抗任何竞争的 callout 活动。
5. 如果心跳被禁用而现在启用，调度第一次触发。
6. 如果心跳启用而现在禁用，取消待处理的 callout。
7. 释放锁并返回。

注意对称处理。如果用户快速切换心跳开关，处理程序每次都做正确的事情。回调中的重新武装不会在用户禁用后触发新的心跳（回调在重新武装前检查 `heartbeat_interval_ms`）。sysctl 的调度不会双重调度（回调只在 `interval_ms > 0` 时重新武装，sysctl 只在 `old == 0` 时调度）。

一个微妙点：callout 用 `sc->mtx` 作为其锁初始化，sysctl 处理程序在调用 `callout_reset` 之前获取 `sc->mtx`。内核也为回调获取 `sc->mtx`。这意味着 sysctl 处理程序和任何正在飞行的回调是串行化的：sysctl 等待如果回调当前运行中，回调不能在 sysctl 持有锁时运行。"用户禁用心跳时回调重新武装"的竞争被锁关闭。

在 attach 中注册 sysctl：

```c
SYSCTL_ADD_PROC(&sc->sysctl_ctx,
    SYSCTL_CHILDREN(device_get_sysctl_tree(dev)),
    OID_AUTO, "heartbeat_interval_ms",
    CTLTYPE_INT | CTLFLAG_RW | CTLFLAG_MPSAFE,
    sc, 0, myfirst_sysctl_heartbeat_interval_ms, "I",
    "Heartbeat interval in milliseconds (0 = disabled)");
```

并在 attach 中初始化 `heartbeat_interval_ms = 0` 以使心跳默认禁用。用户通过设置 sysctl 来启用；在此之前驱动程序是静默的。

### 验证重构

在 `WITNESS` 内核上构建并加载新驱动程序。三个测试：

**测试 1：心跳默认关闭。**

```sh
$ kldload ./myfirst.ko
$ dmesg | tail -3   # attach line shown; no heartbeat logs
$ sleep 5
$ dmesg | tail -3   # still no heartbeat logs
```

预期：attach 行，然后没有东西。心跳默认禁用。

**测试 2：心跳开启。**

```sh
$ sysctl -w dev.myfirst.0.heartbeat_interval_ms=1000
$ sleep 5
$ dmesg | tail -10
```

预期：大约五条心跳行，每秒一条：

```text
myfirst0: heartbeat: cb_used=0, bytes_read=0, bytes_written=0
myfirst0: heartbeat: cb_used=0, bytes_read=0, bytes_written=0
myfirst0: heartbeat: cb_used=0, bytes_read=0, bytes_written=0
```

**测试 3：心跳在负载下。**

在一个终端：

```sh
$ ../../part-02/ch10-handling-io-efficiently/userland/producer_consumer
```

当它运行时，在另一个终端观察 `dmesg`。心跳行现在应该显示非零字节计数：

```text
myfirst0: heartbeat: cb_used=0, bytes_read=1048576, bytes_written=1048576
```

**测试 4：干净禁用。**

```sh
$ sysctl -w dev.myfirst.0.heartbeat_interval_ms=0
$ sleep 5
$ dmesg | tail -3   # nothing new
```

心跳停止；不再发出行。

**测试 5：心跳活跃时 detach。**

```sh
$ sysctl -w dev.myfirst.0.heartbeat_interval_ms=1000
$ kldunload myfirst
```

预期：detach 成功。`myfirst_detach` 中的排空取消待处理的 callout 并等待任何正在飞行的触发完成。没有 `WITNESS` 警告，没有 panic。

如果这些测试中有任何失败，最可能的原因要么是（a）回调中缺少 `is_attached` 检查（所以回调在拆除期间重新武装且 `callout_drain` 永不返回），要么是（b）锁初始化错误（所以回调在没有预期互斥锁持有的情况下运行且 `MYFIRST_ASSERT` 触发）。

### 关于心跳开销的说明

1 秒心跳几乎免费：每秒一次 callout，三次 counter 读取，一条日志行，一次重新武装。总 CPU：每次触发微秒。内存：除了 softc 中已有的 `struct callout` 外为零。

1 毫秒心跳是另一回事。每秒一千条日志行会在几秒内饱和 `dmesg` 缓冲区并主导驱动程序的 CPU 使用。仅在确实快速且日志被调试级别门控时使用短间隔。

为了演示目的，`1000`（每秒一次）是合理的。对于真实世界心跳，合理范围可能是 100 毫秒到 10 秒。本章不强制最小值；用户的选择是他们自己的。

### 心智模型：心跳如何展开

内核和驱动程序在单次心跳触发期间做什么的分步画面。用于将生命周期词汇巩固为具体术语。

- **t=0**：用户运行 `sysctl -w dev.myfirst.0.heartbeat_interval_ms=1000`。
- **t=0+δ**：sysctl 处理程序运行。它读取当前值（0），验证新值（1000），并获取 `sc->mtx`。在锁内：它设置 `sc->heartbeat_interval_ms = 1000`。检测到 0 到非零的转换，它调用 `callout_reset(&sc->heartbeat_co, hz, myfirst_heartbeat, sc)`。内核计算"现在加上 1000 ticks"的轮桶并将 callout 链接入该桶。处理程序释放 `sc->mtx` 并返回 0。
- **t=0 到 t=1s**：内核做其他工作。callout 坐在轮上。
- **t=1s**：硬件时钟中断触发。callout 子系统遍历当前轮桶并找到 `sc->heartbeat_co` 等待。内核将其从轮上移除并分发它。
- **t=1s+δ**：一个 callout 处理线程醒来（或者，如果系统空闲，定时器中断本身运行回调）。内核获取 `sc->mtx`（如果另一个线程持有它可能短暂阻塞；对于我们典型的工作负载，锁是自由的）。一旦 `sc->mtx` 持有，内核调用 `myfirst_heartbeat(sc)`。
- **回调内部**：`MYFIRST_ASSERT(sc)` 确认锁已持有。`is_attached` 检查通过。回调读取 `cbuf_used`（锁持有；安全），读取每 CPU counters（无锁；总是安全），发出一条 `device_printf` 行。它检查 `sc->heartbeat_interval_ms`（1000）；因为它是正的，它调用 `callout_reset(&sc->heartbeat_co, hz, myfirst_heartbeat, sc)` 来调度下一次触发。内核将 callout 重链接入"现在加上 1000 ticks"的轮桶。回调返回。
- **t=1s+ε**：内核释放 `sc->mtx`。callout 现在回到轮上，等待 t=2s。
- **t=2s**：循环重复。

三个观察。

首先，内核代表你管理锁。你的回调运行就像某个看不见的调用者已为你获取了 `sc->mtx`。回调中没有 `mtx_lock`/`mtx_unlock`，因为内核处理它们。

其次，重新武装只是另一个 `callout_reset` 调用。它是被允许的，因为 callout 的锁已持有；内核的内部簿记处理"此 callout 当前正在触发，并且在其自己的回调内部被重新武装"的情况。

第三，触发之间的时间大约是 `hz` ticks，但略多于：回调的工作时间加上任何调度延迟加到间隔上。对于 1 秒心跳，漂移是微秒；对于 1 毫秒心跳可能是可测量的。如果精确周期重要，使用 `callout_reset_sbt` 并计算下一个截止时间为"上一个截止时间 + 间隔"，而不是"现在 + 间隔"。

### 用 dtrace 可视化定时器

一个有用的健全性检查：确认心跳以你配置的速率触发。

```sh
# dtrace -n 'fbt::myfirst_heartbeat:entry { @ = count(); } tick-1sec { printa(@); trunc(@); }'
```

这个 dtrace 一行脚本每秒计数 `myfirst_heartbeat` 被进入多少次。对于 `heartbeat_interval_ms=1000`，计数应该是每秒 1。对于 `heartbeat_interval_ms=100`，计数应该是 10。对于 `heartbeat_interval_ms=10`，计数应该是 100。

如果计数与预期值相差很大，配置没有生效。常见原因：sysctl 处理程序没有提交更改（处理程序中的 bug）、回调由于 `is_attached == 0` 提前退出（拆除流程中的 bug），或系统负载过高导致 callout 触发延迟并堆积。在正常操作中，计数应该稳定到每秒内一个计数。

更精细的 dtrace 配方：回调中花费时间的直方图。

```sh
# dtrace -n '
fbt::myfirst_heartbeat:entry { self->ts = timestamp; }
fbt::myfirst_heartbeat:return /self->ts/ {
    @ = quantize(timestamp - self->ts);
    self->ts = 0;
}
tick-30sec { exit(0); }'
```

每次回调通常花费几微秒（读取 counters 和发出日志行的时间）。如果直方图显示回调花费毫秒或更多，有问题；调查。

### 第 4 节总结

驱动程序有了它的第一个定时器。心跳 callout 周期性触发，记录统计信息行，并重新武装。`is_attached` 标志（在第 12 章为 cv 等待者引入）在这里扮演完全相同的角色：它让回调在设备正在拆除时干净退出。锁感知初始化（用 `sc->mtx` 的 `callout_init_mtx`）意味着回调在数据路径互斥锁持有时运行，内核为我们处理锁获取。

第 5 节更仔细地检查锁契约。契约是 callout API 中最重要的规则；正确使用它使其余部分容易，错误使用它产生难以发现的 bug。


## 第 5 节：处理定时器中的锁定和上下文

第 4 节使用了 `callout_init_mtx` 并信任内核在每次触发之前获取 `sc->mtx`。本节打开那个黑盒。我们看看内核用你注册的锁指针到底做了什么，你在回调内部可以假设什么保证，以及在触发期间你可以做什么和不可以做什么。

锁契约是 `callout(9)` 中最重要的规则。尊重它的驱动程序构造上正确。违反它的驱动程序产生难以重现甚至更难诊断的竞争。现在在这个小节上花时间；当模型稳固时，本章其余部分更容易。

### 内核在你的回调运行之前做了什么

当 callout 的截止时间到达时，内核的 callout 处理代码（在 `/usr/src/sys/kern/kern_timeout.c` 中）在轮上找到 callout 并准备触发它。准备取决于你注册的锁：

- **无锁（`callout_init` 带 `mpsafe=1`）。** 内核设置 `c_iflags`，将 callout 标记为不再待处理，并直接调用你的函数。你的函数必须做所有自己的锁定。
- **互斥锁（`callout_init_mtx` 带 `MTX_DEF` 互斥锁）。** 内核用 `mtx_lock` 获取互斥锁。如果互斥锁有竞争，触发线程阻塞直到可以获取。一旦互斥锁持有，内核调用你的函数。函数返回后，内核用 `mtx_unlock` 释放互斥锁（除非你设置了 `CALLOUT_RETURNUNLOCKED`）。
- **rw 锁（`callout_init_rw`）。** 与互斥锁情况相同，但用 `rw_wlock`（或 `rw_rlock` 如果你设置了 `CALLOUT_SHAREDLOCK`）。
- **rmlock（`callout_init_rm`）。** 与 rmlock 原语相同的形态。
- **Giant（`callout_init` 带 `mpsafe=0` 的默认值）。** 内核获取 Giant。新代码避免这个。

锁在触发线程的上下文中获取。从回调的角度来看，锁就是被持有的：与任何其他线程调用 `mtx_lock` 然后调用你的函数相同的不变量适用。

### 为什么锁由内核获取

一个自然的问题：为什么回调不自己获取锁？内核获取锁的模型有三个微妙的好处。

**它正确地与 `callout_drain` 配合。** 当 `callout_drain` 等待正在飞行的回调完成时，它必须知道回调当前是否正在运行。内核的 callout 子系统精确跟踪这一点，但只是因为它是获取锁并开始回调的代码。如果回调自己获取锁，子系统将不知道"回调当前被阻塞试图获取锁"和"回调已返回"之间的区别，干净的排空将不可能实现而不暴露内核私有状态。内核获取模型保持子系统牢牢控制触发时间线。

**它强制锁类规则。** 内核在注册时检查你提供的锁不是可睡眠的超过 callout 可以容忍的。可睡眠的 sx 锁或 lockmgr 锁将让回调调用 `cv_wait`，这在 callout 上下文中是非法的。初始化函数（`kern_timeout.c` 中的 `_callout_init_lock`）有断言：`KASSERT(lock == NULL || !(LOCK_CLASS(lock)->lc_flags & LC_SLEEPABLE), ...)` 来捕获这个。

**它串行化回调与 `callout_reset` 和 `callout_stop`。** 当回调触发时，锁已持有。当你从驱动程序代码调用 `callout_reset` 或 `callout_stop` 时，你必须持有相同的锁（内核检查）。因此取消/重新调度和触发是互斥的：在任何时刻，要么回调正在触发（锁由内核获取路径持有），要么驱动程序代码正在响应状态变化（锁由驱动程序代码持有）。它们永不并发运行。

第三个属性是使第 4 节心跳 sysctl 处理程序无竞争的原因。处理程序获取 `sc->mtx`，决定取消或调度，取消/调度原子完成，对抗任何正在飞行的回调。不需要特别预防措施；锁完成工作。

### 你在回调内部可以做什么

回调在注册的锁持有时运行。锁决定什么是合法的。

对于 `MTX_DEF` 互斥锁（我们的情况），规则与持有睡眠互斥锁的任何其他代码相同：

- 你可以读写互斥锁保护的任何状态。
- 你可以调用 `cbuf_*` 辅助函数和其他互斥锁内部操作。
- 你可以调用 `cv_signal` 和 `cv_broadcast`（cv API 不要求先放弃互锁）。
- 你可以在同一个 callout 上调用 `callout_reset`、`callout_stop` 或 `callout_pending`（重新武装、取消或检查）。
- 如果你持有它的锁（或它是 mpsafe 的），你可以在*不同*的 callout 上调用 `callout_reset`。

### 你在回调内部不可以做什么

相同的规则：持有互斥锁时不睡眠。

- 你**不可以**直接调用 `cv_wait`、`cv_wait_sig`、`mtx_sleep` 或任何其他睡眠原语。（互斥锁已持有；持有它睡眠将是 sleep-with-mutex 违规，`WITNESS` 会捕获。）
- 你**不可以**调用 `uiomove`、`copyin` 或 `copyout`（每个都可能睡眠）。
- 你**不可以**调用 `malloc(..., M_WAITOK)`。使用 `M_NOWAIT`，并对分配失败情况进行适当的错误处理。
- 你**不可以**调用 `selwakeup`（它获取自己的锁可能产生排序违规）。
- 你**不可以**调用任何可能睡眠的函数。

回调应该简短。几微秒的工作是典型的。如果你需要做长时间运行的事情，回调应该*将工作入队*到 taskqueue 或唤醒可以在进程上下文中执行的内核线程。第 16 章涵盖 taskqueue 模式。

### 如果你确实需要睡眠怎么办？

标准答案：不要在回调中睡眠。推迟工作。两种模式常见。

**模式 1：设置标志，发信号给内核线程。** 回调在 softc 中设置一个标志并发信号给 cv。一个内核线程（由 `kproc_create` 或 `kthread_add` 创建，都是后面章节的主题）在 cv 上等待；它醒来，在进程上下文中做长时间运行的工作，然后回去等待。回调简短；工作不受约束。

**模式 2：在 taskqueue 上入队任务。** 回调调用 `taskqueue_enqueue` 将工作推迟到 taskqueue 工作线程。工作线程在进程上下文中运行并可能睡眠。同样，回调简短；工作不受约束。第 16 章深入介绍这个。

对于第 13 章，我们保持所有定时器工作简短且锁友好；我们还不需要推迟。提到该模式让你知道这个选项存在。

### CALLOUT_RETURNUNLOCKED 标志

`CALLOUT_RETURNUNLOCKED` 更改锁契约。没有它，内核在调用回调之前获取锁并在回调返回后释放它。有了它，内核在调用回调之前获取锁，*回调*负责释放它（或回调可能调用了释放锁的东西）。

为什么你想要这个？两个原因。

**回调丢弃锁以做一些在锁下不能做的事情。** 例如，回调完成其锁定工作，丢弃锁，然后在 taskqueue 上入队任务。入队不需要锁，如果持有甚至可能违反排序。设置 `CALLOUT_RETURNUNLOCKED` 让你在自然的位置写释放。

**回调将锁交给另一个函数。** 如果回调调用一个获取锁所有权并负责释放它的辅助函数，`CALLOUT_RETURNUNLOCKED` 将交接文档化给 `WITNESS`，这样断言检查通过。

没有 `CALLOUT_RETURNUNLOCKED`，内核会在回调返回时断言锁仍由触发线程持有。标志告诉断言允许回调带着锁已释放的状态离开函数。

对于第 13 章，我们不需要 `CALLOUT_RETURNUNLOCKED`。我们所有的回调不获取额外锁，不释放锁，返回时与进入时锁状态相同。提到标志是为了让你在真实驱动程序源代码中能识别它。

### CALLOUT_SHAREDLOCK 标志

`CALLOUT_SHAREDLOCK` 只对 `callout_init_rw` 和 `callout_init_rm` 有效。它告诉内核在调用回调之前以共享（读）模式而非独占（写）模式获取锁。

当回调只读取状态且有许多共享同一锁的 callout 时使用。有了 `CALLOUT_SHAREDLOCK`，只要没有写者持有锁，多个回调可以并发运行。

对于第 13 章，我们使用 `callout_init_mtx` 配合 `MTX_DEF`，不存在共享模式。提到标志是为了完整性。

### "直接执行"模式

内核提供一种"直接"模式，callout 函数在定时器中断上下文本身中运行，而不是被推迟到线程。标志是 `C_DIRECT_EXEC`，传递给 `callout_reset_sbt`。它在 `/usr/src/sys/sys/callout.h` 中文档化，只对锁是自旋互斥锁（或根本没有锁）的 callout 有效。

直接执行很快（没有上下文切换，没有线程唤醒）但规则比普通 callout 上下文更严格：不睡眠（已经为真），不获取睡眠互斥锁，不调用可能睡眠的函数。函数在中断上下文中运行，带有所有这意味着的约束（第 14 章）。

对于第 13 章，我们从不使用 `C_DIRECT_EXEC`。我们的 callout 在那个程度上不是时间关键的。提到它是因为你会在一些硬件驱动程序中看到它（特别是具有热 RX 路径的网络驱动程序）。

### 实际示例：心跳中的锁契约

回顾第 4 节的心跳回调：

```c
static void
myfirst_heartbeat(void *arg)
{
        struct myfirst_softc *sc = arg;
        size_t used;
        uint64_t br, bw;
        int interval;

        MYFIRST_ASSERT(sc);

        if (!sc->is_attached)
                return;

        used = cbuf_used(&sc->cb);
        br = counter_u64_fetch(sc->bytes_read);
        bw = counter_u64_fetch(sc->bytes_written);
        device_printf(sc->dev,
            "heartbeat: cb_used=%zu, bytes_read=%ju, bytes_written=%ju\n",
            used, (uintmax_t)br, (uintmax_t)bw);

        interval = sc->heartbeat_interval_ms;
        if (interval > 0)
                callout_reset(&sc->heartbeat_co,
                    (interval * hz + 999) / 1000,
                    myfirst_heartbeat, sc);
}
```

遍历锁契约：

- callout 用 `callout_init_mtx(&sc->heartbeat_co, &sc->mtx, 0)` 初始化。内核在调用我们之前持有 `sc->mtx`。
- `MYFIRST_ASSERT(sc)` 确认 `sc->mtx` 已持有。健全性检查。
- `sc->is_attached` 在锁下读取。安全。
- `cbuf_used(&sc->cb)` 被调用。cbuf 辅助函数期望 `sc->mtx` 持有；我们持有它。
- `counter_u64_fetch(sc->bytes_read)` 被调用。`counter(9)` 是无锁的，到处安全。
- `device_printf` 被调用。`device_printf` 不获取我们的任何锁；在我们的互斥锁下是安全的。
- `sc->heartbeat_interval_ms` 在锁下读取。安全。
- `callout_reset` 被调用以重新武装。callout API 要求调用 `callout_reset` 时持有 callout 的锁；我们持有它。

回调中的每个操作都尊重锁契约。内核将在回调返回后释放 `sc->mtx`。

一个具体检查：回调*不*调用任何可能睡眠的东西。`device_printf` 不睡眠。`cbuf_used` 不睡眠。`counter_u64_fetch` 不睡眠。`callout_reset` 不睡眠。回调尊重互斥锁的不睡眠约定。

如果我们不小心添加了睡眠，`WITNESS` 会在调试内核上捕获它："sleeping thread (pid X) owns a non-sleepable lock"或类似。教训：信任内核强制执行规则；只要保持回调简短。

### 两个 Callout 共享锁时会发生什么

一个锁可以是许多 callout 的互锁。考虑：

```c
callout_init_mtx(&sc->heartbeat_co, &sc->mtx, 0);
callout_init_mtx(&sc->watchdog_co, &sc->mtx, 0);
callout_init_mtx(&sc->tick_source_co, &sc->mtx, 0);
```

三个 callout，都使用 `sc->mtx`。当其中任何一个触发时，内核获取 `sc->mtx` 并运行回调。当那个回调运行时，锁已持有；没有其他回调（或获取 `sc->mtx` 的其他线程）可以进行。

这是正确的模式：数据路径互斥锁保护所有每 softc 状态，任何需要读取或修改该状态的 callout 共享同一锁。串行化是自动的且免费的。

缺点：如果心跳回调很慢，它会延迟看门狗回调。保持回调简短。

### 回调当前正在触发时你调用 `callout_reset` 会怎样？

一个微妙但重要的问题：回调在一个 CPU 上执行到一半，你在另一个 CPU 上调用 `callout_reset` 重新调度它，会发生什么？

内核正确处理这个情况。让我们遍历一下。

回调在 CPU 0 上触发。它持有 `sc->mtx`（内核在调用之前获取了它）。在 CPU 1 上，你调用 `callout_reset(&sc->heartbeat_co, hz, fn, arg)`（也许是因为用户更改了间隔）。callout API 要求调用者持有 callout 使用的同一锁；你持有，在 CPU 1 上。

但 CPU 0 已经在回调内部，持有 `sc->mtx`。因此 CPU 1 不可能刚刚获取了它。要么 CPU 1 在 CPU 0 获取它之前很久就获取了锁（在这种情况下 CPU 0 当前被阻塞等待锁且不在回调中），要么 CPU 1 某种方式即将获取锁而 CPU 0 即将释放它。

内核通过它用于普通 `mtx_lock` 同步的相同机制正确处理这种情况。在任何给定时刻，`sc->mtx` 只有一个持有者。如果 CPU 0 正在触发，CPU 1 的 `callout_reset` 被阻塞等待锁。当 CPU 0 的回调完成并且内核释放锁时，CPU 1 获取锁并继续重新调度。callout 现在调度到新的截止时间。

如果回调在 CPU 0 释放锁之前重新武装了自己（周期性模式），callout 当前待处理。CPU 1 的 `callout_reset` 取消待处理并替换为新调度。返回值是 1（已取消）。

如果回调没有重新武装（一次性，或间隔为 0），callout 空闲。CPU 1 的 `callout_reset` 调度它。返回值是 0（没有取消之前的调度）。

无论哪种方式，结果都是正确的：`callout_reset` 返回后，callout 调度到新的截止时间，使用新的函数和参数。

### 回调当前正在触发时你调用 `callout_stop` 会怎样？

类似的问题：回调在 CPU 0 上触发，CPU 1 上的调用者想要取消。

CPU 1 调用 `callout_stop`。它需要持有 callout 的锁；它持有。CPU 0 在持有同一锁的同时触发回调；CPU 1 的锁获取被阻塞。当 CPU 0 的回调返回并释放锁时，CPU 1 获取它。

此时，回调可能已经重新武装（如果是周期性的）。`callout_stop` 取消待处理的调度。返回值是 1。

如果回调没有重新武装，callout 空闲。`callout_stop` 是空操作。返回值是 0。

`callout_stop` 返回后，callout 不会再次触发，除非其他东西调度它。重要的是，在 CPU 0 上运行的回调在 `callout_stop` 返回时*已经完成*；锁在整个持续时间内持有。所以 `callout_stop` 确实有效地等待了正在飞行的回调，但只是因为锁获取等待，不是因为 callout 子系统中的任何显式等待。

这就是为什么 `callout_stop` 在你持有锁的正常驱动程序操作中使用是安全的，以及为什么 `callout_drain` 只在你即将释放周围状态（在等待期间不能持有锁）时需要。

### 没有锁的上下文中的 `callout_stop`

如果你在没有持有 callout 锁的情况下调用 `callout_stop` 会怎样？内核的 `_callout_stop_safe` 函数会检测到缺少锁并断言（在 `INVARIANTS` 下）。在非 `INVARIANTS` 内核上，调用可能产生不正确的结果或竞争条件。

规则：当调用 `callout_stop` 或 `callout_reset` 时，你必须持有 callout 初始化时使用的同一锁。内核强制执行这个；违规是 `WITNESS` 警告或 `INVARIANTS` panic。

对于第 13 章，我们总是在从 sysctl 处理程序调用 `callout_reset` 或 `callout_stop` 时持有 `sc->mtx`。detach 路径是例外：它在调用 `callout_drain` 之前释放锁。`callout_drain` 不要求持有锁；事实上它要求*不*持有。

### 模式：条件重新武装

对于周期性 callout 的有用模式：只在某个条件为真时重新武装。在我们的心跳中：

```c
interval = sc->heartbeat_interval_ms;
if (interval > 0)
        callout_reset(&sc->heartbeat_co, ..., myfirst_heartbeat, sc);
```

条件重新武装给用户对周期性触发的精细控制。设置 `interval_ms = 0` 的用户在下一次触发时禁用心跳。回调退出而不重新武装；callout 变为空闲。

更精细的版本：根据活动以可变间隔重新武装。一个在缓冲区忙时更频繁触发、空闲时更少触发的心跳：

```c
if (cbuf_used(&sc->cb) > 0)
        interval = sc->heartbeat_busy_interval_ms;  /* short */
else
        interval = sc->heartbeat_idle_interval_ms;  /* long */

if (interval > 0)
        callout_reset(&sc->heartbeat_co, ..., myfirst_heartbeat, sc);
```

可变间隔让心跳自适应地采样设备。当活动高时，它频繁触发（快速捕捉状态变化）；当活动低时，它很少触发（节省 CPU 和日志空间）。

### 第 5 节总结

锁契约是 `callout(9)` 的核心。内核在每次触发之前获取注册的锁，运行你的回调，然后在之后释放锁。这串行化回调与锁的其他持有者并消除了一类否则需要显式处理的竞争。回调内部的规则与锁的正常规则相同：对于 `MTX_DEF` 互斥锁，不睡眠，不 `uiomove`，不 `malloc(M_WAITOK)`。回调应该简短；如果需要做长时间工作，推迟到 taskqueue（第 16 章）或内核线程。

重新调度和停止在回调在另一个 CPU 上触发时也能正确工作；锁获取机制确保原子性。条件重新武装模式（只在某个条件为真时重新武装）是给周期性 callout 提供优雅禁用路径的自然方式。

第 6 节处理所有这些的推论：在卸载时间，你必须在回调正在进行或待处理时不释放周围状态。`callout_drain` 是工具，卸载竞争是它解决的问题。


## 第 6 节：定时器清理和资源管理

每个 callout 都有一个销毁问题。在你决定移除驱动程序的时刻和你释放周围内存的时刻之间，你必须确保没有回调正在运行且没有回调被调度运行。如果回调在内存被释放后触发，内核崩溃。如果回调在你释放内存时正在运行，内核崩溃。崩溃是可靠的、立即的和致命的；它是那种挂起测试机器并难以调试的 bug，因为回溯指向已经释放的代码。

`callout(9)` 提供干净解决这个问题的工具：正常取消用 `callout_stop`，拆除用 `callout_drain`，以及罕见情况下你想在不阻塞的情况下调度清理用 `callout_async_drain`。本节遍历每个，精确命名卸载竞争，并展示安全驱动程序 detach 的标准模式。

### 卸载竞争

想象驱动程序作为第 13 章的第一阶段（心跳启用，调用 `kldunload`）。没有 `callout_drain`，序列可能是：

1. 用户运行 `kldunload myfirst`。
2. 内核调用 `myfirst_detach`。
3. `myfirst_detach` 清除 `is_attached`，广播 cvs，释放互斥锁，并调用 `mtx_destroy(&sc->mtx)`。
4. 驱动程序模块被卸载；包含 `sc->mtx`、`sc->heartbeat_co` 和 `myfirst_heartbeat` 代码的内存被释放。
5. 硬件时钟中断触发，callout 子系统遍历轮，找到 `sc->heartbeat_co`（仍在轮上因为我们从未取消它），并用 `sc` 作为参数调用 `myfirst_heartbeat`。
6. `myfirst_heartbeat` 不再在内存中。内核跳转到现在无效的地址。Panic。

竞争不是理论的。即使第 5 步在第 4 步之后微秒发生，内核仍然崩溃。窗口很小但非零。

解决方法是确保到第 4 步时，没有 callout 待处理且没有回调在飞行中。两个动作：

- **取消待处理的 callout。** 如果 callout 在轮上，移除它。`callout_stop` 做这个。
- **等待正在飞行的回调。** 如果回调当前在另一个 CPU 上运行，等待它返回。`callout_drain` 做这个。

`callout_drain` 两者都做：它取消待处理并等待正在飞行。这是你在 detach 时调用的。

### `callout_stop` vs `callout_drain`

区别在于调用是否等待。

`callout_stop`：取消待处理，立即返回。不等待正在飞行的回调。如果 callout 待处理并被取消返回 1；否则返回 0。

`callout_drain`：取消待处理，*并且*在它自己返回之前等待任何正在飞行的回调返回。如果 callout 待处理并被取消返回 1；否则返回 0。`callout_drain` 返回后，callout 保证空闲。

在正常驱动程序操作中使用 `callout_stop`，当你想要取消定时器因为促成它的条件已解决。看门狗用例：在操作开始时调度看门狗；当操作成功完成时取消它（用 `callout_stop`）。如果看门狗已经在另一个 CPU 上触发，`callout_stop` 返回，看门狗将运行到完成；这没问题，因为看门狗处理程序将看到操作已完成并什么都不做（或采取一些现在不必要但无害的恢复动作）。

在 detach 时使用 `callout_drain`，那里等待是防止卸载竞争所需的。不要在 detach 时使用 `callout_stop`；回调可能正在另一个 CPU 上运行，周围内存可能在它返回之前被释放。

### `callout_drain` 的两条关键规则

`callout_drain` 有两条容易违反的规则。

**规则 1：调用 `callout_drain` 时不要持有 callout 的锁。** 如果 callout 当前正在执行，回调持有锁（内核为回调获取了它）。`callout_drain` 等待回调返回；回调在它的工作完成时返回；工作包括锁被释放。如果 `callout_drain` 的调用者*也*持有锁，调用者会阻塞等待自己释放它。死锁。

**规则 2：`callout_drain` 可能睡眠。** 它在睡眠队列上等待正在飞行的回调完成。因此 `callout_drain` 只在允许睡眠的上下文中合法：进程上下文（典型 detach 路径）或内核线程上下文。不是中断上下文。不是持有自旋锁时。不是持有任何其他不可睡眠锁时。

这些规则一起暗示标准 detach 路径在调用 `callout_drain` 之前释放 `sc->mtx`（和任何其他不可睡眠锁）。本章的 detach 模式遵循这个：

```c
MYFIRST_LOCK(sc);
sc->is_attached = 0;
cv_broadcast(&sc->data_cv);
cv_broadcast(&sc->room_cv);
MYFIRST_UNLOCK(sc);    /* drop the mutex before draining */

seldrain(&sc->rsel);
seldrain(&sc->wsel);

callout_drain(&sc->heartbeat_co);   /* now safe to call */
```

互斥锁在清除 `is_attached` 后释放。`callout_drain` 运行时不持有互斥锁；它可以自由在睡眠队列上等待。在排空期间触发的任何回调看到 `is_attached == 0` 并退出而不重新武装。排空后，callout 空闲。

### `is_attached` 模式，重温

在第 12 章我们使用 `is_attached` 作为 cv 等待者的信号："设备正在离开；返回 ENXIO"。在第 13 章我们为 callout 使用相同目的："设备正在离开；不要重新武装"。

模式相同：

```c
static void
myfirst_some_callback(void *arg)
{
        struct myfirst_softc *sc = arg;

        MYFIRST_ASSERT(sc);

        if (!sc->is_attached)
                return;  /* device going away; do not re-arm */

        /* ... do the work ... */

        /* re-arm if periodic */
        if (some_condition)
                callout_reset(&sc->some_co, ticks, myfirst_some_callback, sc);
}
```

检查在顶部，任何工作之前。如果 `is_attached == 0`，回调立即退出，不做工作也不重新武装。detach 中的排空将看到 callout 空闲（没有待处理的触发）并干净完成。

一个微妙点：检查发生*在锁下*（内核为我们获取了它）。detach 路径清除 `is_attached` *在锁下*。所以回调总是看到 `is_attached` 的当前值；没有竞争。这是我们为 cv 等待者在第 12 章依赖的相同属性。

### 为什么不用 `callout_stop` 代替？

一个自然问题：不用 `callout_drain`，为什么不用 `callout_stop` 后跟某个手动等待？

`callout_drain` 的实现（在 `/usr/src/sys/kern/kern_timeout.c` 的 `_callout_stop_safe` 中）确切地做了那个，但在内核内部，它可以使用内部睡眠队列而不暴露它们。试图在驱动程序代码中做同样的事情是脆弱的：你需要知道回调当前是否正在运行，你无法从外部判断而不检查内核私有字段。

只需调用 `callout_drain`。这是 API 的目的。

### `callout_async_drain`

对于罕见情况你想在不阻塞的情况下排空，内核提供 `callout_async_drain`：

```c
#define callout_async_drain(c, d) _callout_stop_safe(c, 0, d)
```

它取消待处理并安排一个"排空完成"回调（`d` 函数指针）在正在飞行的回调完成时被调用。调用者不阻塞；控制立即返回。在不能睡眠但需要知道排空何时完成的上下文中有用。

对于本章的目的，`callout_async_drain` 是过度的。我们在阻塞可以的进程上下文中做 detach。提到它是因为你会在一些真实驱动程序源代码中看到它。

### 带定时器的标准 Detach 模式

将所有内容放在一起，带一个或多个 callout 的驱动程序的标准 detach 模式：

> **阅读此示例。** 下面的列表是规范 `callout(9)` 拆除序列的组合视图，从真实驱动程序如 `/usr/src/sys/dev/re/if_re.c`（其中 `callout_drain(&sc->rl_stat_callout)` 在 detach 时运行）和 `/usr/src/sys/dev/watchdog/watchdog.c`（其中两个 callout 依次排空）提炼而来。我们保持了阶段顺序、强制的 `callout_drain()` 调用和锁纪律完整；生产驱动程序添加真实 detach 函数与每一步交错的真实驱动程序特有的每设备簿记。列表命名的每个符号，从 `callout_drain` 到 `seldrain` 到 `mtx_destroy`，都是真实的 FreeBSD API；`myfirst_softc` 字段是本章演进的驱动程序。

```c
static int
myfirst_detach(device_t dev)
{
        struct myfirst_softc *sc = device_get_softc(dev);

        /* 1. Refuse detach if the device is in use. */
        MYFIRST_LOCK(sc);
        if (sc->active_fhs > 0) {
                MYFIRST_UNLOCK(sc);
                return (EBUSY);
        }

        /* 2. Mark the device as going away. */
        sc->is_attached = 0;
        cv_broadcast(&sc->data_cv);
        cv_broadcast(&sc->room_cv);
        MYFIRST_UNLOCK(sc);

        /* 3. Drain the selinfo readiness machinery. */
        seldrain(&sc->rsel);
        seldrain(&sc->wsel);

        /* 4. Drain every callout. Each takes its own line. */
        callout_drain(&sc->heartbeat_co);
        callout_drain(&sc->watchdog_co);
        callout_drain(&sc->tick_source_co);

        /* 5. Destroy cdevs (no new opens after this). */
        if (sc->cdev_alias != NULL) {
                destroy_dev(sc->cdev_alias);
                sc->cdev_alias = NULL;
        }
        if (sc->cdev != NULL) {
                destroy_dev(sc->cdev);
                sc->cdev = NULL;
        }

        /* 6. Free other resources. */
        sysctl_ctx_free(&sc->sysctl_ctx);
        cbuf_destroy(&sc->cb);
        counter_u64_free(sc->bytes_read);
        counter_u64_free(sc->bytes_written);

        /* 7. Destroy primitives in reverse acquisition order:
         *    cvs first, then sx, then mutex. */
        cv_destroy(&sc->data_cv);
        cv_destroy(&sc->room_cv);
        sx_destroy(&sc->cfg_sx);
        mtx_destroy(&sc->mtx);

        return (0);
}
```

七个阶段。每个都是硬性要求。让我们逐一遍历。

**阶段 1**：设备在使用中时拒绝 detach（`active_fhs > 0`）。没有这个，一个打开设备的用户可能在 detach 中间关闭他们的描述符，命中不再有有效状态的代码路径。

**阶段 2**：将设备标记为正在离开。`is_attached` 标志是每个阻塞或未来代码路径的信号，设备正在被移除。cv 广播唤醒任何 cv 等待者；它们重新检查 `is_attached` 并以 `ENXIO` 退出。锁在此阶段持有以使更改对任何刚刚进入处理程序的线程原子。

**阶段 3**：排空 `selinfo`。这确保 `selrecord(9)` 和 `selwakeup(9)` 调用者不再引用设备的 selinfo 结构。

**阶段 4**：排空每个 callout。每个 `callout_drain` 取消待处理并等待正在飞行。互斥锁在第一个排空之前释放（它在阶段 2 结束时释放）。阶段 4 之后，没有 callout 可以运行。

**阶段 5**：销毁 cdevs。在此之后，没有新的 `open(2)` 可以到达驱动程序。（刚好在之前潜入的那些已经在阶段 1 被拒绝了，但那是安全网。）

**阶段 6**：释放辅助资源（sysctl 上下文、cbuf、counters）。

**阶段 7**：以相反顺序销毁原语。顺序重要的原因与第 12 章讨论的相同：cv 使用互斥锁作为它们的互锁；如果我们先销毁互斥锁，一个正在释放互斥锁中间的回调会崩溃。

这很多。它也是每个有 callout 和原语的驱动程序必须做的。第 13 章的配套源代码（`stage4-final/myfirst.c`）完全遵循此模式。

### 关于内核模块卸载的说明

`kldunload myfirst` 通过内核的模块事件处理触发 detach 路径。`MOD_UNLOAD` 事件导致内核调用驱动程序的 detach 函数。如果 detach 函数返回错误（通常是 `EBUSY`），卸载失败，模块保持加载。

我们刚才遍历的标准模式在 `active_fhs > 0` 时返回 `EBUSY`。想要卸载驱动程序的用户必须先关闭每个打开的描述符。从 shell：

```sh
# List processes holding the device open.
$ fstat | grep myfirst
USER     CMD          PID    FD     ... NAME
root     cat        12345     3     ... /dev/myfirst
$ kill 12345
$ kldunload myfirst
```

这是传统的 UNIX 行为；期望用户在卸载之前关闭描述符。驱动程序强制执行它。

### 排空后初始化

一个微妙点：`callout_drain` 之后，callout 空闲但*不*处于与刚初始化的 callout 相同的状态。`c_func` 和 `c_arg` 字段仍指向最后一次回调和参数，以防后面的 `callout_schedule` 想要重用它们。内部标志已清除。

如果你想为不同目的（不同锁、不同回调签名）重用同一个 `struct callout`，你需要再次调用 `callout_init_mtx`（或变体之一）来重新初始化。在 detach 路径中，我们从不重新初始化；周围内存即将被释放。排空时的状态足够。

### 实际演练：在 DDB 中捕获卸载竞争

为了让卸载竞争变得直观，遍历一个粗心的驱动程序省略 `callout_drain` 并且下一次 callout 触发使内核崩溃时发生什么。

想象一个有缺陷的驱动程序，在 detach 中禁用心跳 sysctl 但不调用 `callout_drain`。detach 路径看起来像这样：

```c
static int
buggy_detach(device_t dev)
{
        struct myfirst_softc *sc = device_get_softc(dev);

        MYFIRST_LOCK(sc);
        sc->is_attached = 0;
        sc->heartbeat_interval_ms = 0;  /* hope the callback won't re-arm */
        MYFIRST_UNLOCK(sc);

        /* No callout_drain here! */

        destroy_dev(sc->cdev);
        mtx_destroy(&sc->mtx);
        return (0);
}
```

`is_attached = 0` 和 `heartbeat_interval_ms = 0` 旨在让回调退出而不重新武装。但：

- 回调可能已经执行到一半当 detach 开始时。锁由内核获取路径持有。detach 路径的 `MYFIRST_LOCK(sc)` 阻塞直到回调释放锁。一旦 detach 获取了锁，`is_attached` 和 `heartbeat_interval_ms` 被设置。detach 释放锁。到目前为止还好。
- *但是*：刚刚运行的回调在检查 `interval_ms` 之前已经进入重新武装路径。它调用 `callout_reset` 来调度下一次触发，使用刚刚清除的 `interval_ms` 值 0... 不，等等，回调重新读取 `sc->heartbeat_interval_ms`，看到 0，并不重新武装。好吧，那个情况是安全的。
- *或者*：回调干净完成，没有重新武装。callout 现在空闲。detach 路径继续。它销毁 `sc->mtx` 和周围状态。一切看起来正常。
- *然后*：回调的另一次调用开始触发。callout 不在轮上（没有重新武装），所以这不应该发生，对吧？

如果在不同 CPU 上有并发触发，这可能发生。想象：callout 在 CPU 0 和 CPU 1 上紧密相继触发。CPU 0 启动回调（获取锁）。CPU 1 进入触发路径，试图获取锁，被阻塞。CPU 0 完成回调并重新武装（将 callout 放回轮上以进行下一次触发）。CPU 0 释放锁。CPU 1 获取锁并运行回调。回调重新武装。CPU 1 释放锁。

现在假设 detach 路径在 CPU-0 释放和 CPU-1 获取之间运行。detach 获取锁（现在自由），清除标志，释放锁。CPU 1 获取锁并调用回调。回调重新读取标志，看到清除的值，并退出而不重新武装。好吧，仍然安全。

但现在考虑：detach 路径已销毁互斥锁。CPU 1 的回调执行完成。内核释放现在已销毁的互斥锁。释放操作作用于已释放的内存。Panic。

这就是卸载竞争。修复是直接但绝对要求的：在释放互斥锁之后、销毁原语之前调用 `callout_drain(&sc->heartbeat_co)`。排空等待所有正在飞行的回调（在任何 CPU 上）返回，在它自己返回之前。

带着排空遍历：

- Detach 获取锁，清除标志，释放锁。
- Detach 调用 `callout_drain(&sc->heartbeat_co)`。排空注意到任何正在飞行的回调并等待。
- 所有正在触发的回调干净返回（它们重新读取标志，退出而不重新武装）。
- 排空返回。
- Detach 销毁 cdev，然后销毁互斥锁。
- 此时没有回调可以运行。没有回调可以稍后触发，因为轮上没有 callout。

排空是安全网。跳过它产生一个 panic，可能不会在每次卸载时发生，但最终会在负载下发生。排空是强制性的。

### 在生产内核上忘记排空会发生什么

没有 `INVARIANTS` 或 `WITNESS` 的生产内核不会预先捕获卸载竞争。第一次 callout 在已释放模块的内存被重用后触发，内核读取垃圾指令，跳转到随机位置，并以随机字节碰巧产生的任何模式崩溃。崩溃回溯指向从来不是 bug 的代码；真正的 bug 是几秒前的过去，在没有排空的 detach 路径中。

这就是为什么标准建议是"在提升到生产之前在调试内核上测试"。`WITNESS` 捕获竞争的某些形式（它警告回调以意外方式持有不可睡眠锁被调用）；`INVARIANTS` 捕获其他一些（已销毁互斥锁的 `mtx_destroy`）。生产内核只看到 panic 和错误的回溯。

### `callout_drain` 返回什么

`callout_drain` 返回与 `callout_stop` 相同的值：如果待处理的 callout 被取消则返回 1，否则返回 0。调用者通常不看返回值；调用函数是为了它的副作用（等待正在飞行的回调完成）。

如果你想在特定代码路径完成后确保 callout 完全空闲，纪律是：调用 `callout_drain` 并忽略返回值。无论 callout 是否待处理，排空后它空闲。

### 多个 Callout 的 Detach 顺序

如果你的驱动程序有三个 callout（心跳、看门狗、tick 源）并且你依次 `callout_drain` 每个，总等待时间至多是任何单个正在飞行回调的最长时间（不是总和）。排空是独立的：每个等待自己的回调。它们可以有效并行运行，因为每个只阻塞在其特定 callout 上。

对于本章的伪设备，回调简短（微秒）。排空时间由睡眠队列上的唤醒成本主导，而不是回调工作。总共，所有三个排空在远少于一毫秒内完成，即使在负载下。

对于具有更长回调工作的驱动程序，等待时间可能更长。一个花费 10 毫秒的看门狗回调意味着最坏情况排空是 10 毫秒（如果你碰巧在它正在触发时调用 `callout_drain`）。大多数时候 callout 空闲且排空瞬间完成。无论哪种方式，排空是有界的；它不会无限循环。

### 同样的 Bug，不同的原语：Taskqueue 草图

"回调在 detach 后运行"的 bug 不是 callout 独有的。每个内核延迟工作原语都有相同的陷阱，标准答案始终是一个等待正在飞行回调完成的排空例程。一个简短的兄弟遍历说明了这一点，而不会把第 13 章带离主题。

假设一个驱动程序在 taskqueue 上入队工作而不是使用 callout。softc 持有一个 `struct task` 和一个 `struct taskqueue *`，驱动程序中的某个东西在需要工作时调用 `taskqueue_enqueue(sc->tq, &sc->work)`。现在想象一个有缺陷的 detach，清除了 `is_attached` 并拆除 softc 但忘记排空任务：

```c
static int
buggy_tq_detach(device_t dev)
{
        struct myfirst_softc *sc = device_get_softc(dev);

        MYFIRST_LOCK(sc);
        sc->is_attached = 0;
        MYFIRST_UNLOCK(sc);

        /* No taskqueue_drain here! */

        destroy_dev(sc->cdev);
        free(sc->buf, M_MYFIRST);
        mtx_destroy(&sc->mtx);
        return (0);
}
```

结果与 callout 情况形态相同。如果任务在 detach 运行时待处理，工作线程在 detach 已释放 `sc->buf` 并销毁 `sc->mtx` 后弹出它。任务处理程序解引用 `sc`，发现陈旧内存，要么读取垃圾，要么在第一个锁定操作时 panic。如果任务已经在另一个 CPU 上运行，工作线程在 detach 在它下面释放内存时仍在处理程序内部，结局相同。

修复在结构上与 `callout_drain` 相同：

```c
taskqueue_drain(sc->tq, &sc->work);
```

`taskqueue_drain(9)` 等待直到指定任务既不待处理也不在任何工作线程上当前执行。它返回后，该任务不能再次触发，除非有东西重新入队它，这正是 detach 试图通过首先清除 `is_attached` 来防止的。对于在同一个队列上使用许多任务的驱动程序，`taskqueue_drain_all(9)` 等待该 taskqueue 上当前排队或运行的每个任务，这是模块卸载路径中的通常调用，其中队列上的任何东西都不会被重新入队。

要点不是一条新规则，而是一条更宽的规则：内核中的任何延迟工作原语，无论是 `callout(9)`、`taskqueue(9)`，还是你将在第 6 部分遇到的网络栈 epoch 回调，都需要在它读取的内存被释放之前进行相应的排空。第 16 章深入遍历 `taskqueue(9)`，包括排空如何与任务入队排序交互；现在，记住心智模型是相同的。清除标志，释放锁，排空原语，销毁存储。这个词随原语变化，但模式的形态不变。

### 第 6 节总结

卸载竞争是真实的。`callout_drain` 是解决方法。标准 detach 模式是：忙时拒绝，在锁下清除 `is_attached`，广播 cvs，释放锁，排空 selinfo，排空每个 callout，销毁 cdevs，释放辅助资源，以相反顺序销毁原语。每个阶段都是必要的；跳过任何一个都会产生在负载下使内核崩溃的竞争。

第 7 节将框架用于真实的定时器用例：看门狗、防抖、周期性 tick 源。


## 第 7 节：定时工作的用例和扩展

第 4 节到第 6 节介绍了心跳 callout：周期性的、锁感知的、在拆除时排空的。相同的模式通过小的变化处理广泛的真实驱动程序问题。本节遍历我们添加到 `myfirst` 的三个更多 callout：一个检测缓冲区停滞的看门狗、一个注入合成事件的 tick 源，以及（简要地）许多硬件驱动程序中使用的防抖形态。连同心跳，这四个覆盖了实践中驱动程序定时器的绝大部分。

将本节视为配方集合。每个小节是一个自包含的模式，你可以提升到其他驱动程序。

### 模式 1：看门狗定时器

看门狗检测卡住的条件并对其采取行动。经典形态：在操作开始时调度一个 callout；如果操作成功完成，取消 callout；如果 callout 触发，假定操作卡住，驱动程序采取恢复措施。

对于 `myfirst`，一个有用的看门狗是"缓冲区在太长时间内没有进展"。如果 `cb_used > 0` 且值在 N 秒内没有变化，没有读取者在排空缓冲区。这是不正常的；我们将记录一条警告。

向 softc 添加字段：

```c
struct callout          watchdog_co;
int                     watchdog_interval_ms;   /* 0 = disabled */
size_t                  watchdog_last_used;
```

`watchdog_interval_ms` 是 sysctl 可调参数。`watchdog_last_used` 记录上一次 tick 的 `cbuf_used` 值；下一次 tick 进行比较。

在 attach 中初始化：

```c
callout_init_mtx(&sc->watchdog_co, &sc->mtx, 0);
sc->watchdog_interval_ms = 0;
sc->watchdog_last_used = 0;
```

在 detach 中排空：

```c
callout_drain(&sc->watchdog_co);
```

回调：

```c
static void
myfirst_watchdog(void *arg)
{
        struct myfirst_softc *sc = arg;
        size_t used;
        int interval;

        MYFIRST_ASSERT(sc);

        if (!sc->is_attached)
                return;

        used = cbuf_used(&sc->cb);
        if (used > 0 && used == sc->watchdog_last_used) {
                device_printf(sc->dev,
                    "watchdog: buffer has %zu bytes, no progress in last "
                    "interval; reader stuck?\n", used);
        }
        sc->watchdog_last_used = used;

        interval = sc->watchdog_interval_ms;
        if (interval > 0)
                callout_reset(&sc->watchdog_co,
                    (interval * hz + 999) / 1000,
                    myfirst_watchdog, sc);
}
```

结构与心跳镜像：断言，检查 `is_attached`，做工作，如果间隔非零则重新武装。这次的工作是停滞检查：比较当前 `cbuf_used` 与上次记录的；如果它们匹配且非零，没有进展。

sysctl 处理程序与心跳的对称：

```c
static int
myfirst_sysctl_watchdog_interval_ms(SYSCTL_HANDLER_ARGS)
{
        struct myfirst_softc *sc = arg1;
        int new, old, error;

        old = sc->watchdog_interval_ms;
        new = old;
        error = sysctl_handle_int(oidp, &new, 0, req);
        if (error || req->newptr == NULL)
                return (error);
        if (new < 0)
                return (EINVAL);

        MYFIRST_LOCK(sc);
        sc->watchdog_interval_ms = new;
        if (new > 0 && old == 0) {
                sc->watchdog_last_used = cbuf_used(&sc->cb);
                callout_reset(&sc->watchdog_co,
                    (new * hz + 999) / 1000,
                    myfirst_watchdog, sc);
        } else if (new == 0 && old > 0) {
                callout_stop(&sc->watchdog_co);
        }
        MYFIRST_UNLOCK(sc);
        return (0);
}
```

唯一的添加：启用时，我们将 `watchdog_last_used` 初始化为当前 `cbuf_used`，所以第一次比较有合理的基线。

测试：启用一个 2 秒间隔的看门狗，向缓冲区写入一些字节，不读取它们。两秒后，`dmesg` 应该显示看门狗警告。

```sh
$ sysctl -w dev.myfirst.0.watchdog_interval_ms=2000
$ printf 'hello' > /dev/myfirst
$ sleep 5
$ dmesg | tail
myfirst0: watchdog: buffer has 5 bytes, no progress in last interval; reader stuck?
myfirst0: watchdog: buffer has 5 bytes, no progress in last interval; reader stuck?
```

现在排空缓冲区：

```sh
$ cat /dev/myfirst
hello
```

看门狗停止警告，因为 `cbuf_used` 现在为零（比较 `used > 0` 失败）。

这是一个人为的看门狗。真实看门狗做得更多：重置硬件引擎，杀死卡住的请求，以监控工具可以 grep 的特定格式记录到内核环形缓冲区。形态是相同的：检测，行动，重新武装。

### 模式 2：用于合成事件的 Tick 源

tick 源是一个定期生成事件的 callout，就像硬件所做的那样。用于模拟某物的驱动程序或想要独立于用户空间活动的稳定测试工作负载的驱动程序。

对于 `myfirst`，tick 源可以定期向 cbuf 写入单个字节。启用心跳后，字节计数会明显上升而无需任何外部生产者。

添加字段：

```c
struct callout          tick_source_co;
int                     tick_source_interval_ms;  /* 0 = disabled */
char                    tick_source_byte;          /* the byte to write */
```

在 attach 中初始化：

```c
callout_init_mtx(&sc->tick_source_co, &sc->mtx, 0);
sc->tick_source_interval_ms = 0;
sc->tick_source_byte = 't';
```

在 detach 中排空：

```c
callout_drain(&sc->tick_source_co);
```

回调：

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
                        /* selwakeup omitted on purpose: it may sleep
                         * and we are inside a callout context with the
                         * mutex held. Defer to a taskqueue if real-time
                         * poll(2) wakeups are needed. */
                }
        }

        interval = sc->tick_source_interval_ms;
        if (interval > 0)
                callout_reset(&sc->tick_source_co,
                    (interval * hz + 999) / 1000,
                    myfirst_tick_source, sc);
}
```

结构与心跳相同。工作不同：向 cbuf 写入一个字节，递增 counter，发信号 `data_cv` 让任何读取者醒来。

注意从回调中有意省略 `selwakeup`。`selwakeup` 可能睡眠并可能获取其他锁，这在我们的互斥锁下是非法的。在持有互斥锁的 callout 上下文中调用它将是 `WITNESS` 违规。`cv_signal` 足以唤醒阻塞读取者；`poll(2)` 等待者不会实时被唤醒，但它们会在正常轮询间隔拾取下一个状态变化。对于需要从 callout 立即 `poll(2)` 唤醒的真实驱动程序，答案是将 `selwakeup` 推迟到 taskqueue（第 16 章）。对于第 13 章，省略它是可以接受的。

sysctl 处理程序启用和禁用，镜像其他：

```c
static int
myfirst_sysctl_tick_source_interval_ms(SYSCTL_HANDLER_ARGS)
{
        struct myfirst_softc *sc = arg1;
        int new, old, error;

        old = sc->tick_source_interval_ms;
        new = old;
        error = sysctl_handle_int(oidp, &new, 0, req);
        if (error || req->newptr == NULL)
                return (error);
        if (new < 0)
                return (EINVAL);

        MYFIRST_LOCK(sc);
        sc->tick_source_interval_ms = new;
        if (new > 0 && old == 0)
                callout_reset(&sc->tick_source_co,
                    (new * hz + 999) / 1000,
                    myfirst_tick_source, sc);
        else if (new == 0 && old > 0)
                callout_stop(&sc->tick_source_co);
        MYFIRST_UNLOCK(sc);
        return (0);
}
```

测试：

```sh
$ sysctl -w dev.myfirst.0.tick_source_interval_ms=100
$ cat /dev/myfirst
ttttttttttttttttttttttttttttt    # ten 't's per second
^C
$ sysctl -w dev.myfirst.0.tick_source_interval_ms=0
```

tick 源每秒产生十个 't' 字符，`cat` 读取并打印。通过将 sysctl 设置回零来禁用。

### 模式 3：防抖形态

防抖忽略快速重复的事件。形态：当事件到达时，检查"防抖定时器"是否已经待处理；如果是，忽略事件；如果不是，为 N 毫秒调度一个防抖定时器，当定时器触发时对事件采取行动。

对于 `myfirst`，我们没有硬件事件源，所以我们不会实现完整的防抖。形态，伪代码：

```c
static void
some_event_callback(struct myfirst_softc *sc)
{
        MYFIRST_LOCK(sc);
        sc->latest_event_time = ticks;
        if (!callout_pending(&sc->debounce_co)) {
                callout_reset(&sc->debounce_co,
                    DEBOUNCE_DURATION_TICKS,
                    myfirst_debounce_handler, sc);
        }
        MYFIRST_UNLOCK(sc);
}

static void
myfirst_debounce_handler(void *arg)
{
        struct myfirst_softc *sc = arg;

        MYFIRST_ASSERT(sc);
        if (!sc->is_attached)
                return;

        /* Act on the latest event seen. */
        process_event(sc, sc->latest_event_time);
        /* Do not re-arm; one-shot. */
}
```

当第一个事件到达时，防抖定时器被调度。后续事件更新记录的"最新事件时间"但不重新调度定时器（因为它仍然待处理）。当防抖定时器触发时，处理程序处理最新事件。处理程序返回后，定时器不再待处理；下一个事件将重新调度。

这是模式，不是周期性的。回调不重新武装。`some_event_callback` 中的 `callout_pending` 检查是门。

实验 13.5 将实现类似的防抖作为扩展练习。本章不将其添加到 `myfirst`，因为我们没有要防抖的硬件事件，但形态是值得记住的。

### 模式 4：带指数退避的重试

重试带退避形态：操作失败；在 N 毫秒后调度重试；如果重试也失败，在 2N 毫秒后调度下一次重试；依此类推，在上限处封顶。

对于 `myfirst`，没有操作以需要重试的方式失败。形态：

```c
struct callout          retry_co;
int                     retry_attempt;          /* 0, 1, 2, ... */
int                     retry_base_ms;          /* base interval */
int                     retry_max_attempts;     /* cap */

static void
some_operation_failed(struct myfirst_softc *sc)
{
        int next_delay_ms;

        MYFIRST_LOCK(sc);
        if (sc->retry_attempt < sc->retry_max_attempts) {
                next_delay_ms = sc->retry_base_ms * (1 << sc->retry_attempt);
                callout_reset(&sc->retry_co,
                    (next_delay_ms * hz + 999) / 1000,
                    myfirst_retry, sc);
                sc->retry_attempt++;
        } else {
                /* Give up. */
                device_printf(sc->dev, "retry: exhausted attempts; failing\n");
                some_failure_action(sc);
        }
        MYFIRST_UNLOCK(sc);
}

static void
myfirst_retry(void *arg)
{
        struct myfirst_softc *sc = arg;

        MYFIRST_ASSERT(sc);
        if (!sc->is_attached)
                return;

        if (some_operation(sc)) {
                /* success */
                sc->retry_attempt = 0;
        } else {
                /* failure: schedule next retry */
                some_operation_failed(sc);
        }
}
```

回调重试操作。成功重置尝试计数器。失败以指数增长的延迟调度下一次重试，上限为 `retry_max_attempts`。

此模式存在于许多真实驱动程序中，特别是处理暂时性硬件错误的存储和网络驱动程序。第 13 章不将其添加到 `myfirst`，因为我们没有失败要重试。形态在你的工具箱中。

### 模式 5：延迟收割器

延迟收割器是一个在宽限期后释放某物的一次性 callout。当对象因某些其他代码路径可能仍持有引用而无法立即释放时使用，但我们知道在某个时间过去后，所有引用都将排空。

形态，伪代码草图（`some_object` 类型代表你的驱动程序实际使用的任何延迟释放对象）：

```c
struct some_object {
        TAILQ_ENTRY(some_object) link;
        /* ... per-object fields ... */
};

TAILQ_HEAD(some_object_list, some_object);

struct myfirst_softc {
        /* ... existing fields ... */
        struct callout           reaper_co;
        struct some_object_list  pending_free;
        /* ... */
};

static void
schedule_free(struct myfirst_softc *sc, struct some_object *obj)
{
        MYFIRST_LOCK(sc);
        TAILQ_INSERT_TAIL(&sc->pending_free, obj, link);
        if (!callout_pending(&sc->reaper_co))
                callout_reset(&sc->reaper_co, hz, myfirst_reaper, sc);
        MYFIRST_UNLOCK(sc);
}

static void
myfirst_reaper(void *arg)
{
        struct myfirst_softc *sc = arg;
        struct some_object *obj, *tmp;

        MYFIRST_ASSERT(sc);
        if (!sc->is_attached)
                return;

        TAILQ_FOREACH_SAFE(obj, &sc->pending_free, link, tmp) {
                TAILQ_REMOVE(&sc->pending_free, obj, link);
                free(obj, M_DEVBUF);
        }

        /* Do not re-arm; new objects scheduled later will re-arm us. */
}
```

收割器每秒运行一次（或任何有意义的间隔），释放待处理列表上的所有东西，然后停止。新的调度添加到列表并仅在收割器当前不待处理时重新武装。

用于网络驱动程序，其中接收缓冲区不能立即释放，因为网络层仍有引用；缓冲区为收割器排队，收割器在宽限期后释放它。

`myfirst` 不需要此模式。它在你的工具箱中。

### 模式 6：轮询循环替换

一些硬件不为驱动程序关心的事件产生中断。典型示例：一个传感器有一个状态寄存器，驱动程序必须每几毫秒检查一次以了解新的读数。没有 callout，驱动程序要么自旋（浪费 CPU）要么运行一个睡眠并轮询的内核线程（浪费线程）。有了 callout，轮询循环是一个周期性回调，读取寄存器、采取适当行动并重新武装。

```c
static void
myfirst_poll(void *arg)
{
        struct myfirst_softc *sc = arg;
        uint32_t status;
        int interval;

        MYFIRST_ASSERT(sc);
        if (!sc->is_attached)
                return;

        status = bus_read_4(sc->res, REG_STATUS);   /* hypothetical */
        if (status & STATUS_DATA_READY) {
                /* Pull data from the device into the cbuf. */
                myfirst_drain_hardware(sc);
        }
        if (status & STATUS_ERROR) {
                /* Recover from the error. */
                myfirst_handle_error(sc);
        }

        interval = sc->poll_interval_ms;
        if (interval > 0)
                callout_reset(&sc->poll_co,
                    (interval * hz + 999) / 1000,
                    myfirst_poll, sc);
}
```

回调读取硬件寄存器（未在我们的伪设备中实现，但形态清晰），检查位，采取行动，并重新武装。间隔决定驱动程序检查的频率；更短意味着更响应但更多 CPU。真实轮询驱动程序通常在活跃时使用 1-10 毫秒间隔；空闲时更长。

关于 `bus_read_4` 的代码注释是对第 19 章的前向引用，该章介绍总线空间访问。对于第 13 章，将其视为演示模式的伪代码；轮询逻辑是重要的。

### 模式 7：统计窗口

一个定期拍摄内部计数器快照并计算每间隔速率的周期性 callout。用于监控；驱动程序可以回答"我当前每秒移动多少字节？"而无需用户手动采样。

```c
struct myfirst_stats_window {
        uint64_t        last_bytes_read;
        uint64_t        last_bytes_written;
        uint64_t        rate_bytes_read;       /* bytes/sec, latest interval */
        uint64_t        rate_bytes_written;
};

struct myfirst_softc {
        /* ... existing fields ... */
        struct callout                  stats_window_co;
        int                             stats_window_interval_ms;
        struct myfirst_stats_window     stats_window;
        /* ... */
};

static void
myfirst_stats_window(void *arg)
{
        struct myfirst_softc *sc = arg;
        uint64_t cur_br, cur_bw;
        int interval;

        MYFIRST_ASSERT(sc);
        if (!sc->is_attached)
                return;

        cur_br = counter_u64_fetch(sc->bytes_read);
        cur_bw = counter_u64_fetch(sc->bytes_written);
        interval = sc->stats_window_interval_ms;

        if (interval > 0) {
                /* bytes-per-second over this interval */
                sc->stats_window.rate_bytes_read = (cur_br -
                    sc->stats_window.last_bytes_read) * 1000 / interval;
                sc->stats_window.rate_bytes_written = (cur_bw -
                    sc->stats_window.last_bytes_written) * 1000 / interval;
        }

        sc->stats_window.last_bytes_read = cur_br;
        sc->stats_window.last_bytes_written = cur_bw;

        if (interval > 0)
                callout_reset(&sc->stats_window_co,
                    (interval * hz + 999) / 1000,
                    myfirst_stats_window, sc);
}
```

通过 sysctl 暴露速率。用户可以 `sysctl dev.myfirst.0.stats.rate_bytes_read` 并看到每间隔速率，实时计算，无需手动采样和差值。

此模式存在于许多对监控友好的驱动程序中。粒度（间隔）是可配置的；更长间隔平滑短期突发；更短间隔更快速响应。选择匹配用户想要测量的。

### 模式 8：定时状态刷新

定期刷新驱动程序其余部分读取的缓存值的周期性 callout。当底层值每次计算很昂贵但可以接受稍微陈旧时有用。

对于我们的 `myfirst`，我们没有昂贵的计算要缓存。形态，伪代码：

```c
static void
myfirst_refresh_status(void *arg)
{
        struct myfirst_softc *sc = arg;

        MYFIRST_ASSERT(sc);
        if (!sc->is_attached)
                return;

        sc->cached_status = expensive_compute(sc);
        callout_reset(&sc->refresh_co, hz, myfirst_refresh_status, sc);
}

/* Other code reads sc->cached_status freely; it may be up to 1s stale. */
```

用于计算昂贵的驱动程序（解析硬件状态表、与远程子系统通信），但消费者可以容忍陈旧值。回调定期运行并刷新；消费者获取缓存值。

`myfirst` 不需要此模式。它在你的工具箱中。

### 模式 9：周期性重置

一些硬件需要周期性重置（写入特定寄存器）以保持内部看门狗不触发。模式：

```c
static void
myfirst_periodic_reset(void *arg)
{
        struct myfirst_softc *sc = arg;

        MYFIRST_ASSERT(sc);
        if (!sc->is_attached)
                return;

        bus_write_4(sc->res, REG_KEEPALIVE, KEEPALIVE_VALUE);
        callout_reset(&sc->keepalive_co, hz / 2,
            myfirst_periodic_reset, sc);
}
```

硬件期望至少每秒一次的 keepalive 写入；我们每 500 毫秒发送一次以留有裕量。如果我们错过几次写入（系统负载、重新调度），硬件不会 panic。

用于存储控制器、网络控制器和嵌入式系统，其中设备有主机驱动程序必须满足的每侧看门狗。

### 组合模式

驱动程序通常同时使用多个 callout。`myfirst`（本章第 4 阶段）使用三个：心跳、看门狗、tick 源。每个有自己的 callout 和自己的 sysctl 可调参数。它们共享同一锁（`sc->mtx`），这意味着一次只有一个触发；串行化是自动的。

在更复杂的驱动程序中，你可能有个十个或二十个 callout，每个有特定目的。模式扩展：每个 callout 有自己的 struct callout、自己的回调、自己的 sysctl（如果面向用户）、以及 detach `callout_drain` 块中自己的行。本章的纪律（锁感知初始化、`is_attached` 检查、detach 时排空）适用于它们每一个。

### 第 7 节总结

九种模式覆盖了驱动程序定时器做的大部分工作：心跳、看门狗、防抖、带退避的重试、延迟收割器、统计翻转、轮询循环、统计窗口、定时状态刷新和周期性重置。每个都是周期性或一次性形态的小变化。第 4 到第 6 节的纪律（锁感知初始化、`is_attached` 检查、detach 时排空）统一适用。添加新定时器的驱动程序遵循相同的配方；表面积扩大而维护负担不增加。

第 8 节用整理工作关闭本章：文档、版本提升、回归测试、提交前检查清单。


## 第 8 节：重构和版本控制你的定时器增强驱动程序

驱动程序现在有三个 callout（心跳、看门狗、tick 源）、四个 sysctl（三个间隔加上现有配置），以及一个安全排空每个 callout 的 detach 路径。剩下的工作是整理工作：整理源代码以求清晰、更新文档、提升版本、运行静态分析，并验证回归套件通过。

本节遵循与第 11 章和第 12 章等效部分相同的形态。没有什么光鲜的。所有这些都是区分一次交付的驱动程序和随着增长保持工作的驱动程序的关键。

### 清理源代码

经过本章专注的添加后，三个小的重组值得做。

**将 callout 相关代码分组。** 将所有 callout 回调（`myfirst_heartbeat`、`myfirst_watchdog`、`myfirst_tick_source`）移到源文件的一个部分，在等待辅助函数之后、cdevsw 处理程序之前。将对应的 sysctl 处理程序移到它们旁边。编译器不在乎顺序；读者在乎。

**标准化宏词汇。** 添加一组小宏使 callout 操作在整个驱动程序中一致。现有的 `MYFIRST_LOCK` 和 `MYFIRST_CFG_*` 模式自然扩展：

```c
#define MYFIRST_CO_INIT(sc, co)  callout_init_mtx((co), &(sc)->mtx, 0)
#define MYFIRST_CO_DRAIN(co)     callout_drain((co))
```

`MYFIRST_CO_INIT` 宏显式接受 `sc`，这样它在任何函数中都工作，不只是那些局部变量名为 `sc` 碰巧在作用域中的函数。`MYFIRST_CO_DRAIN` 只需要 callout 本身，因为排空不需要 softc。

宏很薄，但它们文档化约定：驱动程序中的每个 callout 使用 `sc->mtx` 作为其锁并在 detach 时排空。添加 callout 的未来维护者看到宏并知道规则。

**注释 detach顺序。** detach 函数本身很短，但操作顺序是关键的。在每个阶段添加注释：

```c
static int
myfirst_detach(device_t dev)
{
        struct myfirst_softc *sc = device_get_softc(dev);

        /* Phase 1: refuse if in use. */
        MYFIRST_LOCK(sc);
        if (sc->active_fhs > 0) {
                MYFIRST_UNLOCK(sc);
                return (EBUSY);
        }

        /* Phase 2: signal "going away" to all waiters and callbacks. */
        sc->is_attached = 0;
        cv_broadcast(&sc->data_cv);
        cv_broadcast(&sc->room_cv);
        MYFIRST_UNLOCK(sc);

        /* Phase 3: drain selinfo. */
        seldrain(&sc->rsel);
        seldrain(&sc->wsel);

        /* Phase 4: drain every callout (no lock held; safe to sleep). */
        MYFIRST_CO_DRAIN(&sc->heartbeat_co);
        MYFIRST_CO_DRAIN(&sc->watchdog_co);
        MYFIRST_CO_DRAIN(&sc->tick_source_co);

        /* Phase 5: destroy cdevs (no new opens after this). */
        if (sc->cdev_alias != NULL) {
                destroy_dev(sc->cdev_alias);
                sc->cdev_alias = NULL;
        }
        if (sc->cdev != NULL) {
                destroy_dev(sc->cdev);
                sc->cdev = NULL;
        }

        /* Phase 6: free auxiliary resources. */
        sysctl_ctx_free(&sc->sysctl_ctx);
        cbuf_destroy(&sc->cb);
        counter_u64_free(sc->bytes_read);
        counter_u64_free(sc->bytes_written);

        /* Phase 7: destroy primitives in reverse order. */
        cv_destroy(&sc->data_cv);
        cv_destroy(&sc->room_cv);
        sx_destroy(&sc->cfg_sx);
        mtx_destroy(&sc->mtx);

        return (0);
}
```

在 attach 中，匹配的初始化使用两参数形式以便显式传递 `sc`：

```c
MYFIRST_CO_INIT(sc, &sc->heartbeat_co);
MYFIRST_CO_INIT(sc, &sc->watchdog_co);
MYFIRST_CO_INIT(sc, &sc->tick_source_co);
```

注释将函数从看似任意的调用序列转变为文档化的检查清单。

### 更新 LOCKING.md

第 12 章 `LOCKING.md` 文档化了三个原语、两个锁类和一个锁顺序。第 13 章添加三个 callout。要添加的新部分：

```markdown
## Callouts Owned by This Driver

### sc->heartbeat_co (callout(9), MYFIRST_CO_INIT)

Lock: sc->mtx (registered via callout_init_mtx).
Callback: myfirst_heartbeat.
Behaviour: periodic; re-arms itself at the end of each firing if
  sc->heartbeat_interval_ms > 0.
Started by: the heartbeat sysctl handler (transition 0 -> non-zero).
Stopped by: the heartbeat sysctl handler (transition non-zero -> 0)
  via callout_stop, and by myfirst_detach via callout_drain.
Lifetime: initialised in attach via MYFIRST_CO_INIT; drained in detach
  via MYFIRST_CO_DRAIN.

### sc->watchdog_co (callout(9), MYFIRST_CO_INIT)

Lock: sc->mtx.
Callback: myfirst_watchdog.
Behaviour: periodic; emits a warning if cb_used has not changed and
  is non-zero between firings.
Started/stopped: via the watchdog sysctl handler and detach, parallel
  to the heartbeat.

### sc->tick_source_co (callout(9), MYFIRST_CO_INIT)

Lock: sc->mtx.
Callback: myfirst_tick_source.
Behaviour: periodic; injects a single byte into the cbuf each firing
  if there is room.
Started/stopped: via the tick_source sysctl handler and detach,
  parallel to the heartbeat.

## Callout Discipline

1. Every callout uses sc->mtx as its lock via callout_init_mtx.
2. Every callout callback asserts MYFIRST_ASSERT(sc) at entry.
3. Every callout callback checks !sc->is_attached at entry and
   returns early without re-arming.
4. The detach path clears sc->is_attached under sc->mtx, broadcasts
   both cvs, drops the mutex, and then calls callout_drain on every
   callout.
5. callout_stop is used to cancel pending callouts in normal driver
   operation (sysctl handlers); callout_drain is used at detach.
6. NEVER call selwakeup, uiomove, copyin, copyout, malloc(M_WAITOK),
   or any sleeping primitive from a callout callback. The mutex is
   held during the callback, and these calls would violate the
   sleep-with-mutex rule.

## History (extended)

- 0.7-timers (Chapter 13): added heartbeat, watchdog, and tick-source
  callouts; documented callout discipline; standardised callout
  detach pattern.
- 0.6-sync (Chapter 12, Stage 4): combined version with cv channels,
  bounded reads, sx-protected configuration, reset sysctl.
- ... (earlier history as before) ...
```

将此添加到现有 `LOCKING.md` 而不是替换现有内容。新部分与现有的"Locks Owned by This Driver"、"Lock Order"、"Locking Discipline"等并存。

### 提升版本

更新版本字符串：

```c
#define MYFIRST_VERSION "0.7-timers"
```

更新更新日志条目：

```markdown
## 0.7-timers (Chapter 13)

- Added struct callout heartbeat_co, watchdog_co, tick_source_co
  to the softc.
- Added sysctls dev.myfirst.<unit>.heartbeat_interval_ms,
  watchdog_interval_ms, tick_source_interval_ms.
- Added callbacks myfirst_heartbeat, myfirst_watchdog,
  myfirst_tick_source, each lock-aware via callout_init_mtx.
- Updated detach to drain every callout under the documented
  seven-phase pattern.
- Added MYFIRST_CO_INIT and MYFIRST_CO_DRAIN macros for callout
  init and teardown.
- Updated LOCKING.md with a Callouts section and callout
  discipline rules.
- Updated regression script to include callout tests.
```

### 更新 README

README 中的两个新特性：

```markdown
## Features (additions)

- Callout-based heartbeat that periodically logs cbuf usage and
  byte counts.
- Callout-based watchdog that detects stalled buffer drainage.
- Callout-based tick source that injects synthetic data for testing.

## Configuration (additions)

- dev.myfirst.<unit>.heartbeat_interval_ms: periodic heartbeat
  in milliseconds (0 = disabled).
- dev.myfirst.<unit>.watchdog_interval_ms: watchdog interval in
  milliseconds (0 = disabled).
- dev.myfirst.<unit>.tick_source_interval_ms: tick-source interval
  in milliseconds (0 = disabled).
```

### 运行静态分析

对新代码运行 `clang --analyze`。确切的标志取决于你的内核配置；第 11 章回归部分使用的相同配方仍然有效，并增加了 clang 现在可以看透 callout 初始化宏扩展为函数调用的知识：

```sh
$ make WARNS=6 clean all
$ clang --analyze -D_KERNEL -DKLD_MODULE \
    -I/usr/src/sys -I/usr/src/sys/contrib/ck/include \
    -fno-builtin -nostdinc myfirst.c
```

像以前一样分类输出。callout 初始化宏周围可能出现一些假阳性（分析器不总是跟踪嵌入在 `_callout_init_lock` 中的锁关联）；文档化每个，以便下一个维护者不会重新分类。

### 运行回归套件

第 12 章回归脚本自然扩展。两个设计点在脚本之前值得注意：每个子测试用 `dmesg -c` 清除内核消息缓冲区，以便 `grep -c` 只计算*在那个子测试期间*产生的行；读取使用带固定 `count=` 的 `dd` 而不是 `cat`，以便意外的空缓冲区不会让脚本挂起。

```sh
#!/bin/sh
# regression.sh: full Chapter 13 regression.

set -eu

die() { echo "FAIL: $*" >&2; exit 1; }
ok()  { echo "PASS: $*"; }

[ $(id -u) -eq 0 ] || die "must run as root"
kldstat | grep -q myfirst && kldunload myfirst
[ -f ./myfirst.ko ] || die "myfirst.ko not built; run make first"

# Clear any stale dmesg contents so per-subtest greps are scoped.
dmesg -c >/dev/null

kldload ./myfirst.ko
trap 'kldunload myfirst 2>/dev/null || true' EXIT

sleep 1
[ -c /dev/myfirst ] || die "device node not created"
ok "load"

# Chapter 7-12 tests (abbreviated; see prior chapters' scripts).
printf 'hello' > /dev/myfirst || die "write failed"
# dd with bs and count avoids blocking if the buffer is shorter
# than expected; if the read returns short, the test still proceeds.
ROUND=$(dd if=/dev/myfirst bs=5 count=1 2>/dev/null)
[ "$ROUND" = "hello" ] || die "round-trip mismatch (got '$ROUND')"
ok "round-trip"

# Chapter 13-specific tests. Each subtest clears dmesg first so the
# subsequent grep counts only the lines produced during that test.

# Heartbeat enable/disable.
dmesg -c >/dev/null
sysctl -w dev.myfirst.0.heartbeat_interval_ms=100 >/dev/null
sleep 1
HB_LINES=$(dmesg | grep -c "heartbeat:" || true)
[ "$HB_LINES" -ge 5 ] || die "expected >=5 heartbeat lines, got $HB_LINES"
sysctl -w dev.myfirst.0.heartbeat_interval_ms=0 >/dev/null
ok "heartbeat enable/disable"

# Watchdog: enable, write, wait, expect warning, then drain via dd
# (not cat, which would block once the 7 bytes are gone).
dmesg -c >/dev/null
sysctl -w dev.myfirst.0.watchdog_interval_ms=500 >/dev/null
printf 'wd_test' > /dev/myfirst
sleep 2
WD_LINES=$(dmesg | grep -c "watchdog:" || true)
[ "$WD_LINES" -ge 1 ] || die "expected >=1 watchdog line, got $WD_LINES"
sysctl -w dev.myfirst.0.watchdog_interval_ms=0 >/dev/null
dd if=/dev/myfirst bs=7 count=1 of=/dev/null 2>/dev/null  # drain
ok "watchdog warns on stuck buffer"

# Tick source: enable, read, expect synthetic bytes.
dmesg -c >/dev/null
sysctl -w dev.myfirst.0.tick_source_interval_ms=50 >/dev/null
TS_BYTES=$(dd if=/dev/myfirst bs=1 count=10 2>/dev/null | wc -c | tr -d ' ')
[ "$TS_BYTES" -eq 10 ] || die "expected 10 tick bytes, got $TS_BYTES"
sysctl -w dev.myfirst.0.tick_source_interval_ms=0 >/dev/null
ok "tick source produces bytes"

# Detach with callouts active. The trap will not fire after the
# explicit unload because the unload succeeds.
sysctl -w dev.myfirst.0.heartbeat_interval_ms=100 >/dev/null
sysctl -w dev.myfirst.0.tick_source_interval_ms=100 >/dev/null
sleep 1  # allow each callout to fire at least a few times
dmesg -c >/dev/null
kldunload myfirst
trap - EXIT  # the driver is now unloaded
ok "detach with active callouts"

# WITNESS check. Confined to events since the unload above.
WITNESS_HITS=$(dmesg | grep -ci "witness\|lor" || true)
if [ "$WITNESS_HITS" -gt 0 ]; then
    die "WITNESS warnings detected ($WITNESS_HITS lines)"
fi
ok "witness clean"

echo "ALL TESTS PASSED"
```

关于可移植性和健壮性的几点说明。

`dmesg -c` 调用在子测试之间刷新内核消息缓冲区；在 FreeBSD 上 `dmesg -c` 在打印后清除缓冲区。没有这些，在心跳子测试后运行的测试可能会看到运行早期的心跳行并错误计数。

用 `dd` 代替 `cat` 进行往返和看门狗排空读取。`cat` 阻塞直到 EOF，字符设备永不返回；`dd` 在读取 `count=` 块后退出。驱动程序默认阻塞，所以在空缓冲区上过度渴望的 `cat` 会简单地挂起并破坏脚本。

detach 步骤最后不再调用 `kldload`，因为后面唯一的测试（`witness clean`）不需要加载驱动程序。在成功卸载后 `trap` 被清除，这样 EXIT 不会尝试卸载已经卸载的模块。

每次提交后绿色运行是最低标准。在 `WITNESS` 内核上长时间复合压力测试（第 12 章加上第 13 章 callout 活跃）后绿色运行是更高的标准。

### 提交前检查清单

第 12 章检查清单为第 13 章增加三个新项目：

1. 我是否用任何新 callout、间隔或 detach 更改更新了 `LOCKING.md`？
2. 我是否在 `WITNESS` 内核上运行了完整回归套件？
3. 我是否运行了至少 30 分钟启用所有定时器的长时间复合压力测试？
4. 我是否运行了 `clang --analyze` 并分类了每个新警告？
5. 我是否为每个新 callout 回调添加了 `MYFIRST_ASSERT(sc)` 和 `if (!sc->is_attached) return;`？
6. 我是否提升了版本字符串并更新了 `CHANGELOG.md`？
7. 我是否验证了测试套件构建并运行？
8. 我是否检查了每个 cv 都有信号发送者和文档化的条件？
9. 我是否检查了每个 sx_xlock 在每个代码路径上都有配对的 sx_xunlock？
10. **(新)** 我是否在 detach 路径中为每个新 callout 添加了 `MYFIRST_CO_DRAIN`？
11. **(新)** 我是否确认没有 callout 回调调用 `selwakeup`、`uiomove` 或任何睡眠原语？
12. **(新)** 我是否验证通过 sysctl 禁用 callout 实际上停止了周期性触发？

新项目捕捉最常见的第 13 章错误。初始化但从未排空的 callout 是等待发生的 kldunload 崩溃。调用睡眠函数的回调是等待发生的 `WITNESS` 警告。未能停止 callout 的 sysctl 是令人困惑的用户体验。

### 关于向后兼容性的说明

合理的担忧：第 13 章驱动程序添加三个新 sysctl。与 `myfirst` 交互的现有脚本（也许是第 12 章压力套件）会崩溃吗？

答案是不会，原因有二。

首先，新 sysctl 都默认禁用（间隔 = 0）。除非用户启用其中一个，否则驱动程序的行为不变。

其次，第 12 章 sysctl（`debug_level`、`soft_byte_limit`、`nickname`、`read_timeout_ms`、`write_timeout_ms`）和统计（`cb_used`、`cb_free`、`bytes_read`、`bytes_written`）不变。现有脚本读取和写入相同的值。第 13 章添加纯粹是增量的。

这是*非破坏性更改*的纪律：当你添加特性时，不要改变现有特性的含义。成本很小（在更改之前思考）；好处是现有用户看不到回归。

对于第 13 章，心跳、看门狗和 tick 源都是可选的。不知道第 13 章的用户看到与以前相同的驱动程序。阅读本章并启用一个定时器的用户获得新行为。两组都满意。

### 关于 Sysctl 命名的说明

本章使用像 `dev.myfirst.0.heartbeat_interval_ms` 的 sysctl 名称。`_ms` 后缀是有意的：它文档化单位。看到 `heartbeat_interval` 的用户可能合理猜测秒、毫秒或微秒；后缀消除歧义。

其他约定：

- `_count` 用于计数器（总是非负）。
- `_max`、`_min` 用于边界。
- `_threshold` 用于开关。
- `_ratio` 用于百分比或分数。

遵循这些约定使 sysctl 树自描述。检查 `sysctl dev.myfirst.0` 的用户可以从名称和单位猜测每个条目的含义。

### 第 8 节总结

驱动程序现在版本为 `0.7-timers`。它有：

- `LOCKING.md` 中文档化的 callout 纪律。
- callout 生命周期的标准化宏对（`MYFIRST_CO_INIT`、`MYFIRST_CO_DRAIN`）。
- 代码注释中文档化的七阶段 detach 模式。
- 测试每个 callout 的回归脚本。
- 捕获第 13 章特定失败模式的提交前检查清单。
- 三个新 sysctl，具有自描述名称和默认禁用的姿态。

那是本章主要教学弧的结束。实验和挑战紧随其后。


## 动手实验

这些实验通过直接的动手经验巩固第 13 章概念。它们按从最简单到最难的顺序排列。

### 实验前设置检查清单

在开始任何实验之前，确认：

1. **调试内核运行中。** `sysctl kern.ident` 报告带有 `INVARIANTS` 和 `WITNESS` 的内核。
2. **WITNESS 活跃。** `sysctl debug.witness.watch` 返回非零值。
3. **驱动程序源代码与第 12 章第 4 阶段匹配。** 第 13 章示例在此基础上构建。
4. **干净的 dmesg。** 在第一个实验之前 `dmesg -c >/dev/null` 一次。
5. **配套用户空间已构建。** 从 `examples/part-03/ch12-synchronization-mechanisms/userland/`，超时/配置测试器应该存在。
6. **第 4 阶段备份。** 在开始任何修改源代码的实验之前，将你的第 12 章第 4 阶段驱动程序复制到安全位置。

### 实验 13.1：添加心跳 Callout

**目标。** 通过添加心跳 callout 将你的第 12 章第 4 阶段驱动程序转换为第 13 章第 1 阶段驱动程序。

**步骤。**

1. 将你的第 4 阶段驱动程序复制到 `examples/part-03/ch13-timers-and-delayed-work/stage1-heartbeat/`。
2. 向 `struct myfirst_softc` 添加 `struct callout heartbeat_co` 和 `int heartbeat_interval_ms`。
3. 在 `myfirst_attach` 中，调用 `callout_init_mtx(&sc->heartbeat_co, &sc->mtx, 0)` 并初始化 `heartbeat_interval_ms = 0`。
4. 在 `myfirst_detach` 中，释放互斥锁并在销毁原语之前添加 `callout_drain(&sc->heartbeat_co);`。
5. 实现第 4 节所示的 `myfirst_heartbeat` 回调。
6. 实现 `myfirst_sysctl_heartbeat_interval_ms` 并注册它。
7. 在 `WITNESS` 内核上构建并加载。
8. 通过设置 sysctl 验证：`sysctl -w dev.myfirst.0.heartbeat_interval_ms=1000` 并观察 `dmesg` 中的心跳行。

**验证。** 启用时每秒出现一次心跳行。sysctl 设置为 0 时停止。即使启用心跳，detach 也成功。没有 `WITNESS` 警告。

**扩展目标。** 使用 `dtrace` 计数每秒心跳回调：

```sh
# dtrace -n 'fbt::myfirst_heartbeat:entry { @ = count(); } tick-1sec { printa(@); trunc(@); }'
```

计数应该与配置的间隔匹配（1000 毫秒为每秒 1）。

### 实验 13.2：添加看门狗 Callout

**目标.** 添加检测缓冲区停滞的看门狗 callout。

**步骤.**

1. 将实验 13.1 复制到 `stage2-watchdog/`。
2. 向 softc 添加 `struct callout watchdog_co`、`int watchdog_interval_ms`、`size_t watchdog_last_used`。
3. 在 attach/detach 中初始化和排空，与心跳相同。
4. 实现第 7 节的 `myfirst_watchdog` 和相应的 sysctl 处理程序。
5. 构建并加载。
6. 测试：启用 1 秒看门狗，写入一些字节，不排空，观察警告。

**验证.** 缓冲区有未消费字节时每秒出现看门狗警告。缓冲区排空后警告停止。

**扩展目标.** 让看门狗在警告消息中记录自上次更改以来的时间："no progress for X.Y seconds"。

### 实验 13.3：添加 Tick 源

**目标.** 添加向 cbuf 注入合成字节的 tick 源 callout。

**步骤.**

1. 将实验 13.2 复制到 `stage3-tick-source/`。
2. 向 softc 添加 `struct callout tick_source_co`、`int tick_source_interval_ms`、`char tick_source_byte`。
3. 像以前一样初始化和排空。
4. 实现第 7 节所示的 `myfirst_tick_source`。注意有意从回调中省略 `selwakeup`。
5. 实现 sysctl 处理程序。
6. 构建并加载。
7. 启用 100 毫秒 tick 源，用 `cat` 读取，观察合成字节。

**验证.** `cat /dev/myfirst` 每秒大约产生 10 个配置的 tick 字节（默认 `'t'`）。

**扩展目标.** 添加一个 sysctl 让用户在运行时更改 tick 字节。验证更改在下一次触发时立即生效。

### 实验 13.4：验证带活跃 Callout 的 Detach

**目标.** 确认即使所有三个 callout 正在触发，detach 也能正确工作。

**步骤.**

1. 加载第 3 阶段（tick 源）驱动程序。
2. 启用所有三个 callout：
   ```sh
   sysctl -w dev.myfirst.0.heartbeat_interval_ms=500
   sysctl -w dev.myfirst.0.watchdog_interval_ms=500
   sysctl -w dev.myfirst.0.tick_source_interval_ms=100
   ```
3. 确认 `dmesg` 中有活动。
4. 运行 `kldunload myfirst`。
5. 验证没有 panic、没有 `WITNESS` 警告、没有挂起。

**验证.** 卸载在几百毫秒内完成。`dmesg` 显示与卸载相关的没有警告。

**扩展目标.** 用 `time kldunload myfirst` 计时卸载。排空应该是时间的主要贡献者；预期几百毫秒取决于间隔。

### 实验 13.5：构建防抖定时器

**目标.** 实现防抖形态（`myfirst` 不使用，但有用的练习）。

**步骤.**

1. 为实验驱动程序创建临时目录。
2. 实现一个 sysctl `dev.myfirst.0.event_count`，每次写入时递增 1。（用户写入触发"事件"。）
3. 添加一个防抖 callout，在最近事件后 100 毫秒触发并打印窗口内看到的事件总数。
4. 测试：快速写入 sysctl 五次。观察在最后一次写入后 100 毫秒出现一条防抖日志行，报告计数。

**验证.** 多次快速事件只产生一条日志行，计数等于事件数。

### 实验 13.6：检测故意竞争

**目标.** 引入故意 bug（一个调用可睡眠东西的 callout 回调）并观察 `WITNESS` 捕获它。

**步骤.**

1. 在临时目录中，修改心跳回调调用可能睡眠的东西，如 `pause("test", hz / 100)`。
2. 在 `WITNESS` 内核上构建并加载。
3. 启用 1 秒间隔的心跳。
4. 观察 `dmesg` 中的警告："Sleeping on \"test\" with the following non-sleepable locks held: ..."或类似。
5. 撤销更改。

**验证.** `WITNESS` 产生命名睡眠操作和持有互斥锁的警告。警告包括源代码行。

### 实验 13.7：带定时器的长时间复合压力测试

**目标.** 在启用新的第 13 章 callout 的情况下运行第 12 章复合压力套件 30 分钟。

**步骤.**

1. 加载第 4 阶段驱动程序。
2. 以 100 毫秒间隔启用所有三个 callout。
3. 运行第 12 章复合压力脚本 30 分钟。
4. 完成后，检查：
   - `dmesg | grep -ci witness` 返回 0。
   - 所有循环迭代完成。
   - `vmstat -m | grep cbuf` 显示预期的静态分配。

**验证.** 所有标准满足；没有警告、没有 panic、没有内存增长。

### 实验 13.8：用 dtrace 分析 Callout 活动

**目标.** 使用 dtrace 观察 callout 触发模式。

**步骤.**

1. 加载第 4 阶段驱动程序。
2. 以 100 毫秒间隔启用所有三个 callout。
3. 运行一个 dtrace 一行脚本计数每秒每个回调的 callout 触发：
   ```sh
   # dtrace -n '
   fbt::myfirst_heartbeat:entry,
   fbt::myfirst_watchdog:entry,
   fbt::myfirst_tick_source:entry { @[probefunc] = count(); }
   tick-1sec { printa(@); trunc(@); }'
   ```
4. 观察每秒计数。

**验证.** 每个回调每秒大约触发 10 次（1000 毫秒 / 100 毫秒）。

**扩展目标.** 修改 dtrace 脚本报告每个回调内部花费的时间（使用 `quantize` 和 `timestamp`）。

### 实验 13.9：内联取消看门狗

**目标.** 让看门狗成为读取路径在成功时取消的一次性定时器，演示取消-on-进展模式。

**步骤.**

1. 将实验 13.4（`stage3-tick-source` 加心跳/看门狗）复制到临时目录。
2. 修改 `myfirst_watchdog` 为一次性：末尾不重新武装。
3. 每次成功写入后从 `myfirst_write` 调度看门狗。
4. 成功排空后从 `myfirst_read` 取消看门狗（使用 `callout_stop`）。
5. 测试：写入一些字节；不读取；观察看门狗警告触发一次。
6. 测试：写入一些字节；读取它们；观察没有警告（因为读取取消了看门狗）。

**验证.** 看门狗警告只在缓冲区未排空时触发。成功排空取消待处理的看门狗。

**扩展目标.** 添加一个计数器跟踪看门狗触发与取消的频率。通过 sysctl 暴露。比率是缓冲区排水的质量指标。

### 实验 13.10：从 Sysctl 处理程序内部调度

**目标.** 验证从 sysctl 处理程序调度 callout 产生正确的时序。

**步骤.**

1. 向第 4 阶段驱动程序添加 sysctl `dev.myfirst.0.schedule_oneshot_ms`。向它写入 N 调度一个一次性回调在 N 毫秒后触发。
2. 回调简单地记录"one-shot fired"。
3. 测试：写入 100 到 sysctl。观察大约 100 毫秒后的日志行。
4. 测试：写入 1000 到 sysctl。观察大约 1 秒后。
5. 测试：快速连续 5 次写入 1 到 sysctl。观察内核如何处理快速重新调度。

**验证.** 每次写入在大约配置的间隔产生一条日志行。快速写入要么调度新的触发（取消前一个），要么被合并；观察哪种情况。

**扩展目标.** 使用 `dtrace` 测量 sysctl 写入和实际触发之间的增量。直方图应该在配置间隔附近紧密。



## 挑战练习

挑战将第 13 章延伸到基线实验之外。每个都是可选的；每个都旨在加深你的理解。

### 挑战 1：亚毫秒 Tick 源

修改 tick 源 callout 使用带亚毫秒间隔（比如 250 微秒）的 `callout_reset_sbt`。测试它。心跳输出（记录 counters）会发生什么？`lockstat` 对数据互斥锁显示什么？

### 挑战 2：带自适应间隔的看门狗

让看门狗每次触发时减少间隔（麻烦的信号），看到正向进展时增加间隔。在合理值处封顶两端。

### 挑战 3：将 Selwakeup 推迟到 Taskqueue

tick 源省略 `selwakeup` 因为它不能从 callout 上下文调用。阅读 `taskqueue(9)`（第 16 章将深入介绍）并使用 taskqueue 将 `selwakeup` 推迟到工作线程。验证 `poll(2)` 等待者现在正确唤醒。

### 挑战 4：多 CPU Callout 分配

默认情况下，callout 在单个 CPU 上运行。使用 `callout_reset_on` 将三个 callout 绑定到不同的 CPU。使用 `dtrace` 验证绑定。讨论权衡。

### 挑战 5：限制最大间隔

向每个间隔 sysctl 添加验证以强制最小值（比如 10 毫秒）和最大值（比如 60000 毫秒）。低于最小值，用 `EINVAL` 拒绝。高于最大值，也拒绝。文档化选择。

### 挑战 6：基于 Callout 的读取超时

用基于 callout 的机制替换第 12 章基于 `cv_timedwait_sig` 的读取超时：当读取者开始阻塞时调度一次性 callout；callout 触发 `cv_signal` 在 data cv 上唤醒读取者。比较两种方法。

### 挑战 7：统计翻转

添加一个 callout 每 5 秒拍摄 `bytes_read` 和 `bytes_written` 快照，并将每间隔速率存储在循环缓冲区（与 cbuf 分开）中。通过 sysctl 暴露最新速率。

### 挑战 8：持有排空

实验验证在持有 callout 锁时调用 `callout_drain` 会死锁。写一个故意这样做的小驱动程序变体，用 DDB 观察死锁，并文档化症状。

### 挑战 9：重用 Callout 结构

在不同时间为两个不同回调使用同一个 `struct callout`：用回调 A 调度，等待它触发，用回调 B 调度。如果 A 仍然待处理时你用 B 的函数调用 `callout_reset` 会发生什么？写一个测试验证内核的行为。

### 挑战 10：基于 Callout 的 Hello-World 模块

写一个最小模块（不涉及 `myfirst`），除了安装一个每秒打印"tick"的单个 callout 外什么都不做。使用它作为你测试机器上 callout 子系统的健全性检查。

### 挑战 11：验证锁串行化

演示共享同一锁的两个 callout 被串行化。写一个带有两个 callout 的驱动程序；让每个回调短暂睡眠（如果必须的话用 `DELAY()`，因为 `DELAY()` 不睡眠但自旋）。通过 `dtrace` 确认回调永不重叠。

### 挑战 12：合并延迟

对 1 秒定时器使用带各种精度值（0、`SBT_1MS`、`SBT_1S`）的 `callout_reset_sbt`。使用 `dtrace` 测量实际触发时间。给定更多偏差时内核合并多少？合并何时减少 CPU 使用？

### 挑战 13：Callout 轮检查

内核通过 `kern.callout_stat` 和 `kern.callout_*` sysctl 暴露 callout 轮状态。在繁忙系统上读取它们。你能识别你的驱动程序调度的 callout 吗？

### 挑战 14：Callout 函数指针替换

用一个函数调度 callout。在它触发之前，用不同函数再次调度它。会发生什么？第二个函数替换第一个吗？用小实验文档化行为。

### 挑战 15：自适应心跳

让心跳在有最近活动（最后一秒内的写入）时更快触发，空闲时更慢。间隔范围从 100 毫秒（活跃）到 5 秒（空闲）。在压力工作负载下测试以验证它按预期适应。



## 故障排除

此参考目录编排了你在第 13 章工作期间最可能遇到的 bug。

### 症状：Callout 从不触发

**原因。** 要么间隔为零（callout 被禁用），要么 sysctl 处理程序没有实际调用 `callout_reset`。

**修复。** 检查 sysctl 处理程序逻辑。确认检测到 0 到非零的转换。在调用点添加 `device_printf` 以验证。

### 症状：kldunload 后不久 panic

**原因.** callout 在 detach 时未被排空。callout 在模块卸载后触发。

**修复.** 在 detach 路径中为每个 callout 添加 `callout_drain`。确认顺序：排空在清除 `is_attached` 之后，在销毁原语之前。

### 症状：WITNESS 警告 "sleeping thread (pid X) owns a non-sleepable lock"

**原因.** callout 回调在持有内核获取的互斥锁时调用了睡眠的东西（uiomove、copyin、malloc(M_WAITOK)、pause 或任何 cv_wait 变体）。

**修复.** 从回调中移除睡眠操作。如果工作需要睡眠，推迟到 taskqueue 或内核线程。

### 症状：心跳触发一次后不再触发

**原因.** 回调重新武装代码缺失或被变为 false 的条件保护。

**修复.** 检查回调末尾的重新武装。确认 `interval_ms > 0` 且对 `callout_reset` 的调用实际执行。

### 症状：Callout 比配置间隔更频繁触发

**原因.** 两条路径正在调度同一个 callout。sysctl 处理程序和回调都调用 `callout_reset`，或两个回调共享一个 callout 结构。

**修复.** 审计调用点。sysctl 处理程序应该只在 0 到非零转换时 `callout_reset`；回调只在自己末尾重新武装。

### 症状：Detach 挂起

**原因.** callout 回调在 `is_attached = 0` 和 `callout_drain` 之间重新武装了自己。排空现在等待回调完成；回调（在赋值生效之前检查 `is_attached`）没有退出。

**修复.** 确认 `is_attached = 0` 发生在 callout 锁的同一锁下。确认排空发生在赋值之后，不是之前。回调内部的检查必须看到清除的标志。

### 症状：WITNESS 警告关于 callout 锁的锁顺序问题

**原因.** callout 锁被不同路径以冲突顺序获取。

**修复.** callout 的锁是 `sc->mtx`。确认每个获取 `sc->mtx` 的路径遵循规范顺序（先 mtx，然后任何其他锁）。回调在 `sc->mtx` 已持有时运行；回调不能获取应该在 `sc->mtx` 之前获取的任何锁。

### 症状：Callout-Drain 永远睡眠

**原因.** 在持有 callout 锁时调用了 `callout_drain`。死锁：排空等待回调释放锁，回调在等待因为排空是锁持有者。

**修复.** 在调用 `callout_drain` 之前释放锁。标准 detach 模式做这个。

### 症状：回调运行但数据陈旧

**原因.** 回调使用触发前缓存的值。要么它将数据存储在变陈的局部变量中，要么解引用了被修改的结构。

**修复.** 回调在锁持有时运行。每次回调触发时重新读取字段；不要跨触发缓存。

### 症状：`procstat -kk` 显示没有线程在 callout 上等待

**原因.** Callout 没有关联线程。回调在内核线程上下文中运行（callout 子系统管理一个小池），但没有特定线程是"callout 的线程"，就像内核线程可能拥有等待条件那样。

**修复.** 无需；这是设计如此。要查看 callout 活动，使用 `dtrace` 或 `lockstat` 代替。

### 症状：`callout_reset` 意外返回 1

**原因.** callout 之前待处理并被此 `callout_reset` 取消。返回值是信息性的，不是错误。

**修复.** 无需；这是正常的。如果你关心之前的调度是否被覆盖，使用返回值。

### 症状：Sysctl 处理程序对有效输入报告 EINVAL

**原因.** 处理程序的验证拒绝了该值。常见原因：用户传递了验证正确拒绝的负数，或处理程序有过于严格的边界。

**修复.** 检查验证代码。确认用户输入符合文档约束。

### 症状：不同 callout 的两个回调并发运行并死锁

**原因.** 两个 callout 都绑定到同一个锁，所以它们不能并发运行。如果它们看起来死锁，检查任一回调是否获取了另一路径已经持有的另一个锁。

**修复.** 审计锁获取顺序。回调运行的线程持有 `sc->mtx`；如果它尝试获取 `sc->cfg_sx`，顺序必须是 mtx-然后-sx（这正是我们的规范顺序）。

### 症状：tick_source 产生错误的字节

**原因.** 回调在触发时读取 `tick_source_byte`。如果 sysctl 刚刚改变了它，回调可能看到旧值或新值，取决于时机。

**修复.** 这是正确行为；字节更改在下次触发时生效。如果需要立即生效，使用第 12 章的快照并应用模式。

### 症状：lockstat 显示心跳期间数据互斥锁持有异常长时间

**原因.** 心跳回调在锁持有时做了太多工作。

**修复.** 心跳只做 counter 读取和一条日志行；如果持有时间长，可能是 `device_printf`（它为消息缓冲区获取全局锁）。对于低开销心跳，用调试级别门控日志行。

### 症状：Sysctl 设置为 0 后心跳继续

**原因.** `callout_stop` 实际上没有取消，因为回调已经在运行。回调在检查新值之前重新武装了。

**修复.** 如果 sysctl 处理程序在更新 `interval_ms` 和调用 `callout_stop` 时持有 `sc->mtx`，竞争就会关闭。回调在同一个锁下运行；它不能在更新和停止之间运行。验证锁在正确的地方被持有。

### 症状：WITNESS 警告在 init 期间获取 callout 的锁

**原因.** attach 中的某些早期路径尚未建立锁顺序规则。添加 callout 的锁关联使 WITNESS 注意到不一致性。

**修复.** 将 `callout_init_mtx` 移到互斥锁初始化之后。顺序必须是：mtx_init，然后 callout_init_mtx。

### 症状：单个快速 callout 导致高 CPU 使用

**原因.** 一个 1 ms callout 即使做少量工作也会每秒触发 1000 次。如果每次触发需要 100 微秒，那就是一个 CPU 的 10%。

**修复.** 增加间隔。亚秒间隔应该仅在真正需要时使用。

### 症状：dtrace 找不到回调函数

**原因.** dtrace 的 `fbt` 提供者需要函数存在于内核符号表中。如果函数被内联或优化掉了，探测点不可用。

**修复.** 确认函数没有声明为 `static inline` 或以阻止外部链接的方式包装。标准的 `static void myfirst_heartbeat(void *arg)` 是可以的；dtrace 可以探测它。

### 症状：heartbeat_interval_ms 设置后读回为 0

**原因.** sysctl 处理程序更新局部副本但从未提交到 softc 字段，或字段在其他地方被覆盖。

**修复.** 确认处理程序在验证后返回前分配 `sc->heartbeat_interval_ms = new`。

### 症状：WITNESS 警告 "callout_init: lock has sleepable lock_class"

**原因.** 你用 `sx` 锁或其他可睡眠原语而不是 `MTX_DEF` 互斥锁调用了 `callout_init_mtx`。可睡眠锁被禁止作为 callout 互锁，因为 callout 在睡眠非法的上下文中运行。

**修复.** 使用带 `MTX_DEF` 互斥锁的 `callout_init_mtx`，或带 `rw(9)` 锁的 `callout_init_rw`，或带 `rmlock(9)` 的 `callout_init_rm`。不要使用 `sx`、`lockmgr` 或任何其他可睡眠锁。

### 症状：周期性 callout 漂移：每次触发比前一次略晚

**原因.** 重新武装是 `callout_reset(&co, hz, ..., ...)`，它调度"从现在起 1 秒"。每次触发的"现在"比前一次触发的截止时间略晚，所以实际间隔增长回调执行时间。

**修复.** 对于精确周期性，计算下一个截止时间为"上一个截止时间 + 间隔"，而不是"现在 + 间隔"。使用带 `C_ABSOLUTE` 的 `callout_reset_sbt` 和从原始调度计算的绝对 sbintime。

### 症状：压力测试导致 `callout_process` 中间歇性 panic

**原因.** 几乎肯定是卸载竞争或 callout 上不正确的锁关联。callout 子系统本身经过良好测试；这个级别的 bug 通常在调用代码中。

**修复.** 审计每个 callout 的初始化和排空。检查锁关联正确（没有可睡眠锁）。在 `INVARIANTS` 下运行以捕获不变量违规。



## 总结

第 13 章取你在第 12 章构建的驱动程序并赋予它按自己的时间表行动的能力。三个 callout 现在与现有原语并排：一个定期记录状态的心跳、一个检测停滞排水的看门狗、以及一个注入合成字节的 tick 源。每个都是锁感知的、在 detach 时排空的、通过 sysctl 可配置的、并在 `LOCKING.md` 中文档化的。驱动程序的数据路径不变；新代码纯粹是增量的。

我们学到 `callout(9)` 是小型的、规则的、与内核其余部分良好集成的。生命周期每次都是相同的五个阶段：初始化、调度、触发、完成、排空。锁契约每次都是相同的模型：内核在每次触发之前获取注册的锁并在之后释放它，串行化回调与锁的任何其他持有者。detach 模式每次都是相同的七个阶段：忙时拒绝、在锁下标记离开、释放锁、排空 selinfo、排空每个 callout、销毁 cdevs、释放状态、以相反顺序销毁原语。

我们还学到少量在驱动程序中反复出现的配方：心跳、看门狗、防抖、带退避的重试、延迟收割器、统计翻转。每个是周期性或一次性形态的小变化；一旦你知道模式，变化就是机械的。

进入下一章之前的四个提醒。

第一是*在 detach 时排空每个 callout*。卸载竞争是可靠的、立即的和致命的。解决是机械的：每个 callout 一个 `callout_drain`，在 `is_attached` 清除之后、原语销毁之前。没有借口跳过这个。

第二是*保持回调简短且锁感知*。回调在注册的锁持有时、在可能不睡眠的上下文中运行。像对待硬件中断处理程序一样对待它：做最少的，推迟其余的。如果工作需要睡眠，将其入队到 taskqueue（第 16 章）或唤醒内核线程。

第三是*使用 sysctl 使定时器行为可配置*。硬编码间隔是维护负担。让用户通过 `sysctl -w` 调整心跳、看门狗或 tick 源使驱动程序在你未预见的环境中有用。成本很小（每个旋钮一个 sysctl 处理程序），好处很大。

第四是*在每次代码更改的同一提交中更新 `LOCKING.md`*。文档偏离代码的驱动程序积累微妙 bug，因为没有人知道规则应该是什么。纪律是每次更改一分钟；好处是多年的干净维护。

这四个纪律一起产生与 FreeBSD 其余部分良好组合、经受长期维护、在负载下可预测行为的驱动程序。它们也是第 14 章将假设的纪律；本章的模式直接转移到中断处理程序。

### 你现在应该能够做什么

在进入第 14 章之前的简短自检清单：

- 为任何"等待直到 X 发生，或直到 Y 时间过去"的需求在 `callout(9)` 和 `cv_timedwait_sig` 之间做出选择。
- 用适合你驱动程序需求的锁变体初始化 callout。
- 使用 `callout_reset`（或 `callout_reset_sbt` 实现亚 tick 精度）调度一次性和周期性 callout。
- 在正常操作中用 `callout_stop` 取消 callout；在 detach 时用 `callout_drain` 排空。
- 编写尊重锁契约和不睡眠规则的 callout 回调。
- 使用 `is_attached` 模式使回调在拆除期间安全。
- 在 `LOCKING.md` 中文档化每个 callout，包括其锁、回调、生命周期。
- 识别卸载竞争并通过标准七阶段 detach 模式避免它。
- 根据需要构建看门狗、心跳、防抖和 tick 源模式。
- 使用 `dtrace` 验证 callout 触发速率、延迟和生命周期行为。
- 阅读真实驱动程序源代码（led、uart、网络驱动程序）并识别本章的模式在工作。



## 展望：通往第 14 章的桥梁

第 14 章标题为 *Taskqueues and Deferred Work*。它的范围是从驱动程序角度看到的内核延迟工作框架：如何将工作从一个不能安全运行它的上下文（callout 回调、中断处理程序、epoch 段落）移到一个可以运行它的上下文。

第 13 章以三种具体方式准备了基础。

首先，你已经知道 callout 回调在严格的上下文契约下运行：不睡眠、不获取可睡眠锁、不 `uiomove`、不 `copyin`、不 `copyout`、不 `selwakeup`（持有驱动程序互斥锁时）。你看到该契约在 `myfirst_tick_source` 中 `selwakeup` 被有意省略的那一行被强制执行，因为 callout 上下文不能合法地进行调用。第 14 章介绍 `taskqueue(9)`，这正是内核为这种交接提供的原语：callout 将任务入队，任务在线程上下文中运行，其中省略的调用是合法的。

其次，你已经知道排空-at-detach 纪律。`callout_drain` 确保回调在 detach 继续时不运行。任务有匹配的原语：`taskqueue_drain` 等待直到特定任务既不待处理也不运行。心智模型相同；排序增长一步（先 callout，然后任务，然后它们影响的所有东西）。

第三，你已经知道 `LOCKING.md` 作为活文档的形态。第 14 章用 Tasks 部分扩展它，命名每个任务、其回调、其生命周期及其在 detach 顺序中的位置。纪律相同；词汇稍微宽一些。

第 14 章将涵盖的具体主题：

- `taskqueue(9)` API：`struct task`、`TASK_INIT`、`taskqueue_create`、`taskqueue_start_threads`、`taskqueue_enqueue`、`taskqueue_drain`、`taskqueue_free`。
- 预定义的系统 taskqueue（`taskqueue_thread`、`taskqueue_swi`、`taskqueue_fast`、`taskqueue_bus`）以及何时私有 taskqueue 更好。
- 合并规则：当任务已经待处理时再次入队会发生什么。
- `struct timeout_task` 和 `taskqueue_enqueue_timeout` 用于延迟并调度的工作。
- 真实 FreeBSD 驱动程序中反复出现的模式，以及出错时的调试故事。

你不需要提前阅读。第 13 章是足够的准备。带上你的 `myfirst` 驱动程序（第 13 章第 4 阶段）、你的测试套件和你的 `WITNESS` 启用内核。第 14 章从第 13 章结束的地方开始。

一个简短的结束反思。你开始本章时有一个不能自行行动的驱动程序：每一行工作都由用户做的事情触发。你离开时有一个拥有内部时间的驱动程序，定期记录其状态，检测停滞排水，注入合成事件用于测试，并在模块卸载时干净地拆除所有这些基础设施。这是真正的质的飞跃，模式直接转移到第 4 部分将介绍的每种驱动程序。

花点时间。你在开始第 3 部分时的驱动程序知道如何一次处理一个线程。你现在拥有的驱动程序协调许多线程，支持可配置的定时工作，并无竞争地拆除。从这里，第 14 章添加*任务*，这是任何定时器回调需要触发 callout 不能安全执行的工作的驱动程序缺失的部分。然后翻页。

### 关于时间的最后附注

在第 14 章之前的最后一个想法。你花了两个章节学习同步（第 12 章）和一个章节学习时间（第 13 章）。两者深度相关：同步的核心是关于事件彼此*何时*发生，而时间是衡量这种关系的明确方式。锁序列化访问；cv 协调等待；callout 在截止时间触发。这三种是切分同一个底层问题的不同方式：独立的执行流如何就顺序达成一致？

第 14 章添加第四个部分：*上下文*。Callout 在精确时刻触发，但它触发的上下文（不睡眠、不可睡眠锁、无用户空间复制）比大多数真实工作需要的更窄。通过 `taskqueue(9)` 的延迟工作是从该窄上下文到线程上下文的桥梁，在那里全套内核操作是合法的。

模式转移。Callout 为其回调使用的锁感知初始化与你决定任务回调获取哪个锁时的形态相同。Callout 在分离时使用的排空模式是任务在关闭时使用的形态。"在此做少量工作，延迟其余"的纪律是第 14 章为你提供的遵循该纪律的具体工具。

所以当你到达第 14 章时，框架已经很熟悉。你将向驱动程序的工具包添加一个原语。它的规则与你现在知道的 callout 规则干净地组合。你构建的工具（LOCKING.md、七阶段 detach、断言并检查已附加模式）将在不变得脆弱的情况下吸收新原语。

这就是使本书第 3 部分作为一个单元工作的原因。每章向驱动程序对世界的感知添加一个维度（并发、同步、时间、延迟工作），每一章都建立在上一章的基础设施上。到第 3 部分结束时，你的驱动程序将为第 4 部分及其后面的真实硬件做好准备。


## 参考：故障排除症状详述

本节继续本章前面开始的故障排除参考，涵盖更多症状及其修复。

### 症状：Sysctl 设置 callout 间隔但行为不变

**原因。** Sysctl 处理程序更新局部副本但从未提交到 softc 字段，或字段在其他地方被覆盖。

**修复。** 确认处理程序在验证后、返回前赋值 `sc->heartbeat_interval_ms = new`。

### 症状：Detach 耗时数秒即使 callout 似乎空闲

**原因。** Callout 有长间隔（比如 30 秒）且当前待处理。`callout_drain` 等待下一次触发或显式取消。如果截止时间还很远，等待可能很长。

**修复。** `callout_drain` 实际上不等待截止时间；它取消待处理并在任何进行中的回调完成后返回。如果你的 detach 耗时数秒，那是其他地方有问题（回调真的花了那么长时间，或涉及不同的睡眠）。用 `dtrace` 检查 `_callout_stop_safe` 来调查。

### 症状：`callout_stop` 后 `callout_pending` 返回 true

**原因。** 竞争：另一个路径在 `callout_stop` 和你检查 `callout_pending` 之间调度了 callout。或者：callout 之前在另一个 CPU 上触发并刚刚重新武装。

**修复。** 在调用 `callout_stop` 和检查 `callout_pending` 时始终持有 callout 的锁。锁使操作原子化。

### 症状：驱动程序卸载很久后 callout 函数出现在 `dmesg` 中

**原因。** 卸载竞争。Callout 在 detach 销毁状态后触发。如果内核没有立即 panic，打印的行来自已释放回调的代码，在一个已经忘记原始模块的内核中执行。

**修复。** 如果你正确调用了 `callout_drain` 这不应该发生。如果发生了，你的 detach 路径有问题；检查每个 callout 确认每个都被排空。

### 症状：长时间暂停后多个 callout 一次全部触发

**原因。** 系统处于负载下（长时间运行的中断、卡住的 callout 处理线程）无法服务 callout 轮。当它恢复时，它快速连续处理所有延迟的 callout。

**修复。** 这在异常负载下是正常的。如果经常发生，调查为什么系统不能按时服务 callout。`dtrace -n 'callout-end'`（如果你的内核暴露 `callout` 提供者）显示实际触发时间。

### 症状：Callout 从不触发即使 `callout_pending` 返回 true

**原因。** 要么 callout 卡在一个离线的 CPU 上（罕见，但在 CPU 热插拔期间可能），要么系统的时钟中断在该 CPU 上不触发。

**修复。** 检查 `kern.hz` 和 `kern.eventtimer` sysctl。默认的 hz=1000 应该产生定期触发。如果 CPU 离线，callout 子系统将待处理的 callout 迁移到工作 CPU，但有一个窗口。对于大多数驱动程序，这不是真正的问题。

### 症状：`kern.callout.busy` 计数器在负载下增长

**原因。** Callout 子系统检测到回调耗时太长。每个"忙碌"事件是一个未在预期窗口内完成的回调。

**修复。** 用 `dtrace` 检查慢回调。长回调表示要么工作太多（拆分成多个 callout 或推迟到 taskqueue），要么锁竞争问题（回调在等待锁变得可用）。

### 症状：驱动程序日志显示"callout_drain detected migration"或类似

**原因。** Callout 绑定到特定 CPU（通过 `callout_reset_on`），绑定迁移与排空重叠。内核在内部解决此问题；日志消息是信息性的。

**修复。** 通常不需要。如果消息频繁，考虑是否真的需要每 CPU 绑定。

### 症状：`callout_reset_sbt` 给出意外的计时

**原因。** `precision` 参数太宽松：内核将你的 callout 与其他 callout 合并到一个比预期宽得多的窗口中。

**修复。** 将精度设置为更小的值（或 0 表示"尽可能接近截止时间触发"）。默认是 `tick_sbt`（一个 tick 的余量），这对大多数定时器工作没问题。

### 症状：电源管理事件后正常工作的 callout 停止触发

**原因。** 系统的时钟中断可能被重新配置（睡眠/唤醒期间事件定时器模式之间的转换）。Callout 子系统在此类转换后重新调度待处理的 callout，但计时可能略有偏差。

**修复。** 用 `dtrace` 验证 callout 的回调正在被调用。如果没有，callout 已被迁移或丢弃；从已知良好的代码路径重新调度。

### 症状：驱动程序中所有 callout 都在同一个 CPU 上触发

**原因。** 这是默认行为。Callout 绑定到调度它们的 CPU；如果你的所有 `callout_reset` 调用都在 CPU 0 上运行（因为用户的 syscall 被分发到那里），所有 callout 都在 CPU 0 上触发。

**修复。** 这对大多数驱动程序是正确的。如果你想要负载分配，使用 `callout_reset_on` 显式绑定到不同的 CPU。大多数驱动程序不需要这个；随着不同的 syscall 命中不同的 CPU，每 CPU 轮随时间自然平衡。

### 症状：`callout_drain` 返回但下一个 syscall 看到陈旧状态

**原因。** 回调完成并返回，但后续代码路径观察到了回调设置的状态。这是正确的行为，不是 bug。

**修复。** 无。排空只保证回调不再运行；回调所做的任何状态更改仍然有效。如果更改是不想要的，回调本不应该进行这些更改。

### 症状：回调中的重新武装静默失败

**原因。** 条件 `interval > 0` 为假，因为用户刚刚禁用了定时器。回调退出而不重新武装；callout 变为空闲。

**修复。** 这是正确的行为。如果你想知道回调何时拒绝重新武装，添加一个计数器或日志行。

### 症状：Callout 触发但 `device_printf` 无输出

**原因。** 驱动程序的 `dev` 字段为 NULL 或设备已被分离且 cdev 被销毁。`device_printf` 在这些状态下可能抑制输出。

**修复。** 添加显式的 `printf("%s: ...\n", device_get_nameunit(dev), ...)` 来绕过包装器。或通过 `KASSERT` 确认 `sc->dev` 有效。



## 参考：驱动程序阶段演进

第 13 章在四个不同阶段演进 `myfirst` 驱动程序，每个都是 `examples/part-03/ch13-timers-and-delayed-work/` 下的自己的目录。演进反映了章节的叙述；它让读者可以一次一个定时器地构建驱动程序并看到每个添加贡献了什么。

### 第 1 阶段：heartbeat

添加定期记录 cbuf 使用情况和字节计数的心跳 callout。新 sysctl `dev.myfirst.<unit>.heartbeat_interval_ms` 在运行时启用、禁用和调整心跳。

变化：一个新 callout、一个新回调、一个新 sysctl，以及 attach/detach 中相应的初始化/排空。

你可以验证：将 sysctl 设置为正值产生定期日志行；设置为 0 停止它们；即使在启用心跳的情况下 detach 也成功。

### 第 2 阶段：watchdog

添加检测停滞缓冲区排水的看门狗 callout。新 sysctl `dev.myfirst.<unit>.watchdog_interval_ms` 启用、禁用和调整间隔。

变化：一个新 callout、一个新回调、一个新 sysctl，以及相应的初始化/排空。

你可以验证：启用看门狗并写入字节（不读取它们）产生警告行；读取缓冲区停止警告。

### 第 3 阶段：tick-source

添加向 cbuf 注入合成字节的 tick 源 callout。新 sysctl `dev.myfirst.<unit>.tick_source_interval_ms` 启用、禁用和调整间隔。

变化：一个新 callout、一个新回调、一个新 sysctl，以及相应的初始化/排空。

你可以验证：启用 tick 源并从 `/dev/myfirst` 读取以配置速率产生字节。

### 第 4 阶段：final

结合了所有三个 callout 的完整驱动程序，加上 `LOCKING.md` 扩展、版本升级到 `0.7-timers`，以及标准化的 `MYFIRST_CO_INIT` 和 `MYFIRST_CO_DRAIN` 宏。

变化：整合。没有新原语。

你可以验证：回归套件通过；所有 callout 活跃时的长时间压力测试干净运行；`WITNESS` 无警告。

这个四阶段演进是规范的第 13 章驱动程序。配套示例完全反映了各阶段，读者可以编译和加载其中任何一个。



## 参考：真实看门狗剖析

真实的生产看门狗比本章的示例做得更多。简要介绍真实看门狗通常包含的内容，在编写或阅读驱动程序源代码时有用。

### 每请求跟踪

真实 I/O 看门狗单独跟踪每个待处理请求。看门狗回调遍历待处理请求列表，找出那些已经未完成太长时间的请求，并对每个采取行动。

```c
struct myfirst_request {
        TAILQ_ENTRY(myfirst_request) link;
        sbintime_t   submitted_sbt;
        int           op;
        /* ... 其他请求状态 ... */
};

TAILQ_HEAD(, myfirst_request) pending_requests;
```

看门狗遍历 `pending_requests`，计算每个的年龄，并对陈旧的采取行动。

### 基于阈值的行动

不同的年龄得到不同的行动。直到 T1，忽略（请求仍在工作）。T1 到 T2，记录警告。T2 到 T3，尝试软恢复（向请求发送重置）。超过 T3，硬恢复（重置通道，向用户失败请求）。

```c
age_sbt = now - req->submitted_sbt;
if (age_sbt > sc->watchdog_hard_sbt) {
        /* 硬恢复 */
} else if (age_sbt > sc->watchdog_soft_sbt) {
        /* 软恢复 */
} else if (age_sbt > sc->watchdog_warn_sbt) {
        /* 记录警告 */
}
```

### 统计

真实看门狗跟踪每个阈值被命中的频率、超过每个阈值的请求百分比等。统计信息作为 sysctl 暴露用于监控。

### 可配置阈值

每个阈值（T1、T2、T3）是一个 sysctl。不同的部署需要不同的边界；硬编码是错误的。

### 恢复日志

恢复行动记录到 dmesg，带有监控工具可以 grep 的可识别前缀。详细消息包含请求的身份、采取的行动，以及可能有助于诊断底层问题的任何内核状态。

### 与其他子系统的协调

硬恢复通常涉及与驱动程序其他部分的合作：I/O 层必须知道通道正在被重置、排队的请求必须重新排队或失败、驱动程序的"是否运行"状态必须更新。

对于第 13 章，我们的看门狗简单得多。它检测一种特定情况（cbuf 无进展），记录警告，并重新武装。这捕捉了基本模式。真实世界的看门狗增量添加上述各部分。



## 参考：周期性与事件驱动驱动程序架构

一个小架构题外话。有些驱动程序由事件主导（中断到达，驱动程序响应）。其他由轮询主导（驱动程序定期唤醒检查）。理解你的驱动程序是哪一种有助于选择原语。

### 事件驱动

在事件驱动设计中，驱动程序大部分时间空闲。活动由以下触发：

- 用户 syscall（`open`、`read`、`write`、`ioctl`）。
- 硬件中断（第 14 章）。
- 来自其他子系统的唤醒（cv 信号、taskqueue 运行）。

事件驱动设计中的 callout 通常是看门狗（跟踪一个事件，如果未发生则触发）和清理器（在事件后清理）。

`myfirst` 驱动程序最初是事件驱动的（读/写触发一切）。第 13 章添加了一些轮询风格的行为（心跳、tick 源）用于演示，但底层设计仍然是事件驱动的。

### 轮询驱动

在轮询驱动设计中，驱动程序定期唤醒做工作，无论是否有人在请求。这对于不为驱动程序关心的事件产生中断的硬件是合适的。

轮询驱动设计中的 callout 是驱动程序的心跳：每次触发，回调检查硬件并处理发现的任何内容。

轮询循环模式（第 7 节）是基本形态。真实轮询驱动用自适应间隔（忙时更快轮询，空闲时更慢）、错误计数（太多次失败轮询后放弃）等扩展它。

### 混合

大多数真实驱动程序是混合的：事件驱动大部分活动，但周期性 callout 捕获事件遗漏的内容（超时、慢轮询、统计）。本章的模式适用于任一侧；在哪里使用哪个是设计决策。

对于 `myfirst`，我们的混合使用：
- 用于主 I/O 的事件驱动 syscall 处理程序。
- 用于定期日志的心跳 callout。
- 用于卡住状态检测的看门狗 callout。
- 用于合成事件生成的可选 tick 源 callout。

真实驱动程序会有更多 callout，但形态相同。



## 收尾

第 13 章采用了你在第 12 章构建的驱动程序，并赋予它按自己的时间表行动的能力。三个 callout 现在位于现有原语旁边：定期记录状态的心跳、检测停滞排水的看门狗、注入合成字节的 tick 源。每个都是锁感知的、在分离时排空的、通过 sysctl 可配置的、并在 `LOCKING.md` 中记录的。驱动程序的数据路径未变；新代码纯粹是增量的。

我们学到 `callout(9)` 小巧、规则、与内核其余部分良好集成。生命周期每次都是相同的五个阶段：初始化、调度、触发、完成、排空。锁契约每次都是相同的模型：内核在每次触发前获取注册的锁并在之后释放，将回调与任何其他持有者序列化。分离模式每次都是相同的七个阶段：如果忙则拒绝、在锁下标记正在离开、释放锁、排空 selinfo、排空每个 callout、销毁 cdev、释放状态、以相反顺序销毁原语。

我们还学到了少数几个在驱动程序中反复出现的配方：心跳、看门狗、防抖、重试并退避、延迟清理器、统计滚动。每个都是周期性或一次性形态的小变化；一旦你知道模式，变化就是机械的。

转向第 14 章之前的四个结束提醒。

第一是*在分离时排空每个 callout*。卸载竞争是可靠的、立即的、致命的。治愈是机械的：每个 callout 一个 `callout_drain`，在清除 `is_attached` 之后、销毁原语之前。没有理由跳过这个。

第二是*保持回调短小且锁感知*。回调在持有注册的锁时运行，在可能不睡眠的上下文中。像对待硬件中断处理程序：做最少的，延迟其余的。如果工作需要睡眠，将其入队到 taskqueue（第 16 章）或唤醒内核线程。

第三是*使用 sysctl 使定时器行为可配置*。硬编码间隔是维护负担。让用户从 `sysctl -w` 调整心跳、看门狗或 tick 源使驱动程序在你未预期的环境中有用。成本很小（每个旋钮一个 sysctl 处理程序），收益很大。

第四是*在与任何代码更改相同的提交中更新 `LOCKING.md`*。文档与代码漂移的驱动程序会积累微妙的 bug，因为没有人知道规则应该是什么。纪律是每次更改一分钟；收益是多年的干净维护。

这四个纪律一起产生与 FreeBSD 其余部分良好组合、在长期维护中存活、在负载下行为可预测的驱动程序。它们也是第 14 章将假设的纪律；本章的模式直接转移到中断处理程序。

### 你现在应该能够做到的

转向第 14 章之前的简短自检清单：

- 为任何"等待 X 发生，或直到 Y 时间已过"的需求在 `callout(9)` 和 `cv_timedwait_sig` 之间选择。
- 用适合驱动程序需求的锁变体初始化 callout。
- 使用 `callout_reset`（或 `callout_reset_sbt` 用于亚 tick 精度）调度一次性 和周期性 callout。
- 在正常操作中用 `callout_stop` 取消 callout；在分离时用 `callout_drain` 排空。
- 编写尊重锁契约和不睡眠规则的 callout 回调。
- 使用 `is_attached` 模式使回调在拆除期间安全。
- 在 `LOCKING.md` 中文档化每个 callout，包括其锁、回调、生命周期。
- 识别卸载竞争并通过标准七阶段 detach 模式避免它。
- 根据需要构建看门狗、心跳、防抖和 tick 源模式。
- 使用 `dtrace` 验证 callout 触发速率、延迟和生命周期行为。
- 阅读真实驱动程序源代码（led、uart、网络驱动程序）并识别本章的模式在工作。

如果其中任何一项感觉不确定，本章的实验是建立肌肉记忆的地方。每个不需要超过一两小时；它们一起覆盖了本章介绍的每个原语和每个模式。

### 关于配套示例的说明

`examples/part-03/ch13-timers-and-delayed-work/` 下的配套源代码反映了章节的阶段。每个阶段建立在上一个之上，所以你可以编译和加载任何阶段来准确看到章节描述的驱动程序状态。

如果你喜欢手动输入更改（首次阅读推荐），使用章节的示例作为指南，配套源代码作为参考。如果你喜欢阅读完成的代码，配套源代码是规范的。

关于 `LOCKING.md` 文档的说明：章节文本解释 `LOCKING.md` 应该包含什么。实际文件在示例树中与源代码并排。在进行更改时保持两者同步；在与代码更改相同的提交中更新 `LOCKING.md` 的纪律是保持文档准确的最可靠方式。



## 参考：callout(9) 快速参考

日常查阅的紧凑 API 摘要。

### 初始化

```c
callout_init(&co, 1)                       /* mpsafe; 无锁 */
callout_init_mtx(&co, &mtx, 0)             /* 锁是 mtx（默认） */
callout_init_mtx(&co, &mtx, CALLOUT_RETURNUNLOCKED)
callout_init_rw(&co, &rw, 0)               /* 锁是 rw，排他 */
callout_init_rw(&co, &rw, CALLOUT_SHAREDLOCK)
callout_init_rm(&co, &rm, 0)               /* 锁是 rmlock */
```

### 调度

```c
callout_reset(&co, ticks, fn, arg)         /* 基于 tick 的延迟 */
callout_reset_sbt(&co, sbt, prec, fn, arg, flags)
callout_reset_on(&co, ticks, fn, arg, cpu) /* 绑定到 CPU */
callout_schedule(&co, ticks)               /* 重用上次 fn/arg */
```

### 取消

```c
callout_stop(&co)                          /* 取消; 不等待 */
callout_drain(&co)                         /* 取消 + 等待进行中 */
callout_async_drain(&co, drain_fn)         /* 异步排空 */
```

### 检查

```c
callout_pending(&co)                       /* callout 是否已调度? */
callout_active(&co)                        /* 用户管理的活动标志 */
callout_deactivate(&co)                    /* 清除活动标志 */
```

### 常用标志

```c
CALLOUT_RETURNUNLOCKED   /* 回调自己释放锁 */
CALLOUT_SHAREDLOCK       /* 以共享模式获取 rw/rm */
C_HARDCLOCK              /* 对齐 hardclock() */
C_DIRECT_EXEC            /* 在定时器中断上下文中运行 */
C_ABSOLUTE               /* sbt 是绝对时间 */
```



## 参考：标准 Detach 模式

带有 callout 的驱动程序的七阶段 detach 模式：

```c
static int
myfirst_detach(device_t dev)
{
        struct myfirst_softc *sc = device_get_softc(dev);

        /* 第 1 阶段: 如果在使用中则拒绝。 */
        MYFIRST_LOCK(sc);
        if (sc->active_fhs > 0) {
                MYFIRST_UNLOCK(sc);
                return (EBUSY);
        }

        /* 第 2 阶段: 标记正在离开; 广播 cv。 */
        sc->is_attached = 0;
        cv_broadcast(&sc->data_cv);
        cv_broadcast(&sc->room_cv);
        MYFIRST_UNLOCK(sc);

        /* 第 3 阶段: 排空 selinfo。 */
        seldrain(&sc->rsel);
        seldrain(&sc->wsel);

        /* 第 4 阶段: 排空每个 callout（不持有锁）。 */
        callout_drain(&sc->heartbeat_co);
        callout_drain(&sc->watchdog_co);
        callout_drain(&sc->tick_source_co);

        /* 第 5 阶段: 销毁 cdev（无新打开）。 */
        if (sc->cdev_alias != NULL) {
                destroy_dev(sc->cdev_alias);
                sc->cdev_alias = NULL;
        }
        if (sc->cdev != NULL) {
                destroy_dev(sc->cdev);
                sc->cdev = NULL;
        }

        /* 第 6 阶段: 释放辅助资源。 */
        sysctl_ctx_free(&sc->sysctl_ctx);
        cbuf_destroy(&sc->cb);
        counter_u64_free(sc->bytes_read);
        counter_u64_free(sc->bytes_written);

        /* 第 7 阶段: 以相反顺序销毁原语。 */
        cv_destroy(&sc->data_cv);
        cv_destroy(&sc->room_cv);
        sx_destroy(&sc->cfg_sx);
        mtx_destroy(&sc->mtx);

        return (0);
}
```

跳过任何阶段都会创建一类在负载下使内核崩溃的 bug。



## 参考：何时使用每种定时原语

紧凑的决策表。

| 需求 | 原语 |
|---|---|
| 在时间 T 的一次性回调 | `callout_reset` |
| 每 T tick 的周期性回调 | `callout_reset` 并重新武装 |
| 亚毫秒回调计时 | `callout_reset_sbt` |
| 将回调绑定到特定 CPU | `callout_reset_on` |
| 在定时器中断上下文中运行回调 | `callout_reset_sbt` 带 `C_DIRECT_EXEC` |
| 等待直到条件，有截止时间 | `cv_timedwait_sig`（第 12 章） |
| 等待直到条件，无截止时间 | `cv_wait_sig`（第 12 章） |
| 将工作推迟到工作线程 | `taskqueue_enqueue`（第 16 章） |
| 长时间运行的周期性工作 | 内核线程 + `cv_timedwait` |

前四个是 `callout(9)` 用例；其他使用其他原语。



## 参考：要避免的 Callout 错误

最常见错误的紧凑列表：

- **在分离时忘记 `callout_drain`。** 导致下次 callout 触发时 panic。
- **在持有 callout 的锁时调用 `callout_drain`。** 导致死锁。
- **从回调调用睡眠函数。** 导致 `WITNESS` 警告或 panic。
- **为新代码使用 `callout_init`（mpsafe=0）。** 获取 Giant；损害可扩展性。
- **在回调顶部忘记 `is_attached` 检查。** 分离可能与重新武装竞争且永不完成。
- **在两个回调之间共享 `struct callout`。** 混淆且很少是你想要的；使用两个结构。
- **在回调中硬编码间隔。** 用户无法调整行为。
- **未能验证 sysctl 输入。** 负数或荒谬的间隔产生令人惊讶的行为。
- **从回调调用 `selwakeup`。** 获取其他锁；可能产生顺序违规。
- **在分离时使用 `callout_stop`。** 不等待进行中；导致卸载竞争。



## 参考：阅读 kern_timeout.c

`/usr/src/sys/kern/kern_timeout.c` 中的两个函数值得打开一次。

`callout_reset_sbt_on` 是核心调度函数。每个其他 `callout_reset` 变体都是最终到达这里的包装器。函数处理"callout 当前正在运行"、"callout 待处理并正在重新调度"、"callout 需要迁移到不同 CPU"和"callout 是新的"的情况。复杂性是真实的；面向公众的行为是简单的。

`_callout_stop_safe` 是统一的停止并可能排空函数。`callout_stop` 和 `callout_drain` 都是用不同标志调用此函数的宏。`CS_DRAIN` 标志是触发等待进行中行为的那个。阅读此函数一次可准确了解排空如何与触发的回调交互。

该文件在 FreeBSD 14.3 中约 1550 行。你不需要阅读每一行。浏览函数名，找到上述两个函数，并仔细阅读每个。二十分钟的阅读给你实现的运作感觉。



## 参考：c_iflags 和 c_flags 字段

简短看看两个标志字段，在阅读直接检查它们的真实驱动程序源代码时有用。

`c_iflags`（内部标志，由内核设置）：

- `CALLOUT_PENDING`：callout 在轮上等待触发。通过 `callout_pending(c)` 读取。
- `CALLOUT_PROCESSED`：callout 在哪个列表上的内部记账。
- `CALLOUT_DIRECT`：如果使用了 `C_DIRECT_EXEC` 则设置。
- `CALLOUT_DFRMIGRATION`：延迟迁移到不同 CPU 期间设置。
- `CALLOUT_RETURNUNLOCKED`：如果锁处理契约设置为期望回调释放锁则设置。
- `CALLOUT_SHAREDLOCK`：如果 rw/rm 锁要以共享模式获取则设置。

`c_flags`（外部标志，由调用者管理）：

- `CALLOUT_ACTIVE`：用户管理的位。由内核在成功的 `callout_reset` 期间设置；由 `callout_deactivate` 或成功的 `callout_stop` 清除。驱动程序代码通过 `callout_active(c)` 读取。
- `CALLOUT_LOCAL_ALLOC`：已弃用；仅用于旧式 `timeout(9)` 风格。
- `CALLOUT_MPSAFE`：已弃用；改用 `callout_init_mtx`。

驱动程序代码只触及 `CALLOUT_ACTIVE`（通过 `callout_active` 和 `callout_deactivate`）和 `CALLOUT_PENDING`（通过 `callout_pending`）。其他都是内部的。



## 参考：第 13 章自测

在转向第 14 章之前，使用此评估标准。格式反映第 11 章和第 12 章：概念问题、代码阅读问题和动手问题。如果任何项目感觉不确定，相关章节名称在括号中。

问题不是详尽的；它们采样章节的核心思想。能够自信回答所有问题的读者已为下一章做好准备。在特定项目上挣扎的读者应在继续之前重新阅读相关章节。

### 概念问题

这些问题采样第 13 章词汇。能够不重新查阅章节回答所有问题的读者已内化材料。

1. **为什么使用 `callout(9)` 而不是内核线程做周期性工作？** Callout 在休息时基本上免费；线程有 16 KB 栈和调度器开销。对于短周期性工作，callout 是正确答案。

2. **`callout_stop` 和 `callout_drain` 有什么区别？** `callout_stop` 取消待处理并立即返回；`callout_drain` 取消待处理并等待任何进行中的回调完成。在正常操作中使用 `callout_stop`，在分离时使用 `callout_drain`。

3. **`callout_init_mtx` 的 `lock` 参数完成什么？** 内核在每次回调触发前获取该锁并在之后释放。回调在持有锁时运行。

4. **Callout 回调不能做什么？** 任何可能睡眠的事情，包括 `cv_wait`、`mtx_sleep`、`uiomove`、`copyin`、`copyout`、`malloc(M_WAITOK)`、`selwakeup`。

5. **为什么标准 detach 模式在调用 `callout_drain` 之前释放互斥锁？** `callout_drain` 可能睡眠等待进行中的回调。持有互斥锁睡眠是非法的。在排空前释放互斥锁是强制的。

6. **什么是卸载竞争？** Callout 在 `kldunload` 运行后触发，发现其函数和周围状态已释放。内核跳转到无效内存并 panic。

7. **什么是周期性回调重新武装模式？** 回调做它的工作并在末尾调用 `callout_reset` 调度下一次触发。

8. **为什么回调在做工作前检查 `is_attached`？** 分离清除 `is_attached`；如果回调在清除和 `callout_drain` 之间的短暂窗口期间触发，检查防止回调做依赖于正在拆除的状态的工作。

9. **如果在持有 callout 的锁时调用 `callout_drain` 会发生什么？** 死锁：排空等待回调释放锁，但回调无法释放排空者持有的锁。在调用 `callout_drain` 之前始终释放锁。

10. **`MYFIRST_CO_INIT` 和 `MYFIRST_CO_DRAIN` 的目的是什么？** 它们是 `callout_init_mtx` 和 `callout_drain` 的宏包装器，记录约定：每个 callout 使用 `sc->mtx` 并在分离时排空。通过宏标准化使新 callout 机械地添加且易于审查。

11. **为什么在持有互斥锁时从 callout 回调调用 `device_printf` 是安全的？** 它不获取任何驱动程序的锁且不睡眠；它用自己的内部锁定写入全局环形缓冲区。它是少数几个"从 callout 上下文调用安全"的输出函数之一。

12. **为 `hz` tick 调度 callout 与为 `tick_sbt * hz` 调度有什么区别？** 概念上没有；两者都代表一秒。第一个使用 `callout_reset`（基于 tick 的 API）；第二个使用 `callout_reset_sbt`（基于 sbintime 的 API）。选择匹配你所需精度的 API。

### 代码阅读问题

打开你的第 13 章驱动程序源代码并验证：

1. 每个 `callout_init_mtx` 在 detach 中配对了一个 `callout_drain`。
2. 每个回调以 `MYFIRST_ASSERT(sc)` 和 `if (!sc->is_attached) return;` 开始。
3. 没有回调调用 `selwakeup`、`uiomove`、`copyin`、`copyout`、`malloc(M_WAITOK)` 或 `cv_wait`。
4. 分离路径在调用 `callout_drain` 之前释放互斥锁。
5. 每个 callout 有一个 sysctl 允许用户启用、禁用或更改其间隔。
6. 每个 callout 的生命周期（attach 中初始化，detach 中排空）在 `LOCKING.md` 中记录。
7. 每个周期性回调仅在其间隔为正时重新武装。
8. 每个 sysctl 处理程序在调用 `callout_reset` 或 `callout_stop` 时持有互斥锁。

### 动手问题

这些都应该快速运行；如果任何失败，本章的相关实验会遍历设置。

1. 加载第 13 章驱动程序。用 1 秒间隔启用心跳。确认 dmesg 每秒显示一条日志行。

2. 用 1 秒间隔启用看门狗。写入一些字节。等待。确认警告出现。

3. 用 100 ms 间隔启用 tick 源。用 `cat` 读取。确认每秒 10 字节。

4. 启用所有三个 callout。运行 `kldunload myfirst`。验证无 panic，无警告。

5. 打开 `/usr/src/sys/kern/kern_timeout.c`。找到 `callout_reset_sbt_on`。阅读前 50 行。你能用两句话描述它做什么吗？

6. 使用 `dtrace` 确认心跳以预期速率触发。当 `heartbeat_interval_ms=200`，速率应该是每秒 5 次触发。

7. 修改看门狗以记录额外信息（例如，启动以来触发过多少次回调）。验证新字段出现在 dmesg 中。

8. 打开 `/usr/src/sys/dev/led/led.c`。找到对 `callout_init_mtx`、`callout_reset` 和 `callout_drain` 的调用。与本章的模式比较。有差异吗？

如果所有八个动手问题通过且概念问题容易，你的第 13 章工作是扎实的。你已为第 14 章做好准备。

关于节奏的说明：本书第 3 部分很密集。三个章节（11、12、13）关于同步相关主题是很多新词汇要吸收。如果本章的实验感觉容易，那是好迹象。如果感觉难，在开始第 14 章之前休息一两天；材料在短暂休息后会很好地复合，带着新鲜注意力开始第 14 章比疲惫地推进更好。

关于测试的说明：第 8 节的回归脚本覆盖基本功能。对于长期信心，在 `WITNESS` 内核上运行第 12 章的综合压力套件，所有三个第 13 章 callout 活跃，至少 30 分钟。干净的运行是声明驱动程序生产就绪之前要清除的门槛。较少则冒着卸载竞争或基本回归未捕获的微妙锁顺序问题的风险。

如果压力运行发现问题，本章前面的故障排除参考是第一站。大多数问题落入那里的症状模式之一。如果症状不匹配参考中的任何内容，下一步是内核调试器；故障排除部分的 DDB 配方帮你入门。

当你有干净的压力运行和干净的审查时，章节工作完成。驱动程序现在是第 4 阶段最终版，版本 0.7-timers，有记录的 callout 纪律和证明纪律在负载下保持的回归套件。花点时间欣赏。然后继续。



## 参考：真实 FreeBSD 驱动程序中的 Callout

简要介绍 `callout(9)` 如何在实际 FreeBSD 源代码中使用。你在本章学到的模式直接映射到这些驱动程序使用的模式。

### `/usr/src/sys/dev/led/led.c`

一个简单但有启发性的例子。`led(4)` 驱动程序让用户空间脚本在支持它们的硬件上闪烁 LED。驱动程序每 `hz / 10`（100 ms）调度一个 callout 来步进闪烁模式。

`led_timeout` 和 `led_state` 中的关键调用是：

```c
callout_reset(&led_ch, hz / 10, led_timeout, p);
```

Callout 在 `led_drvinit` 中初始化：

```c
callout_init_mtx(&led_ch, &led_mtx, 0);
```

周期性 callout，通过驱动程序的互斥锁锁感知。正是本章教授的模式。

### `/usr/src/sys/dev/uart/uart_core.c`

串口（UART）驱动程序在某些配置中使用 callout 来轮询不在字符接收时产生中断的硬件上的输入。模式相同：`callout_init_mtx`、周期性回调、在分离时排空。

### 网络驱动程序看门狗

大多数网络驱动程序（ixgbe、em、mlx5 等）在 attach 时安装一个看门狗 callout。看门狗每几秒触发一次；它检查硬件最近是否产生了中断，如果没有则重置芯片。回调短小、锁感知，如果需要做复杂的事情则延迟到驱动程序的其余部分（通常通过将任务入队到 taskqueue）。

### 存储驱动程序 I/O 超时

ATA、NVMe 和 SCSI 驱动程序使用 callout 作为 I/O 超时。当请求发送到硬件时，驱动程序为将来某个有界时间调度一个 callout。如果请求正常完成，驱动程序取消 callout。如果 callout 触发，驱动程序假设请求卡住并采取恢复行动（重置通道、重试、向用户失败请求）。

这是应用于每个请求操作而不是整个设备的看门狗模式（第 7 节）。

### USB 集线器轮询

USB 集线器驱动程序每几百毫秒（可配置）轮询集线器状态。轮询发现连接/断开的设备、端口状态更改和集线器不为其中断的传输完成。模式是轮询循环模式（第 7 节）。

### 这些驱动程序的不同之处

上述驱动程序使用第 13 章未覆盖的额外原语，特别是 taskqueue。它们中许多调度一个 callout，该 callout 不是做实际工作，而是将任务入队到 taskqueue。任务在进程上下文中运行并可能睡眠。第 16 章深入介绍此模式。

对于第 13 章，我们将所有工作保持在 callout 的回调内，锁感知且不睡眠。真实驱动程序通过延迟长工作扩展此模式；底层定时基础设施（callout）相同。



## 参考：比较 callout 与其他时间原语

比本章前面的决策表更详细的比较。

### `callout(9)` vs `cv_timedwait_sig`

在 syscall 层的相同原语是 `cv_timedwait_sig`（第 12 章）："等待直到条件 X 变为真，但在 T 毫秒后放弃"。调用者是等待的那方；cv 是被信号通知的那方。

与 `callout(9)` 比较：回调在时间 T 运行，无论是否有人在等待它。回调是行动的那方，它做自己的工作，可能信号通知其他东西。

两者的区别在于*谁等待*和*谁行动*。在 `cv_timedwait_sig` 中，syscall 线程既是等待者又是（唤醒后）行动者。在 `callout(9)` 中，syscall 线程不参与；独立上下文触发回调。

当 syscall 线程在等待完成后有工作要做时使用 `cv_timedwait_sig`。当需要在截止时间发生独立的事情时使用 `callout(9)`。

### `callout(9)` vs `taskqueue(9)`

`taskqueue(9)` 通过将函数入队到工作线程"尽快"运行它。没有时间延迟；工作在工作线程可以取起时尽快运行。

与 `callout(9)` 比较：函数在将来特定时间运行。

常见模式是组合两者：callout 在时间 T 触发，决定需要工作，并将任务入队。任务在进程上下文中运行并做实际工作（可能包括睡眠）。第 16 章将覆盖此组合。

### `callout(9)` vs 内核线程

内核线程可以循环并调用 `cv_timedwait_sig` 产生周期性行为。线程是重量级的：16 KB 栈、调度器条目、优先级分配。

与 `callout(9)` 比较：无线程；内核的定时器中断机制处理触发，小型 callout 处理池运行回调。

当工作真正长时间运行时使用内核线程（工作线程等待、做大量工作、再次等待）。当工作短小且只需要按计划触发时使用 callout。

### `callout(9)` vs 用户空间周期性 SIGALRM

用户空间进程可以安装 `SIGALRM` 处理程序并使用 `alarm(2)` 进行周期性行为。信号处理程序在进程中运行；它短小且受约束。

与 `callout(9)` 比较：内核端、锁感知、与驱动程序的其余部分集成。

用户空间闹钟适合用户空间代码。它们在驱动程序工作中没有角色；内核自己做自己的事。

### `callout(9)` vs 硬件定时器

一些硬件有自己的定时器寄存器（主机驱动程序编程的"GP 定时器"或"看门狗定时器"）。这些硬件定时器直接向主机触发中断。它们快速、精确，并绕过内核的 callout 子系统。

在以下情况使用硬件定时器：
- 硬件提供了一个且你有中断处理程序。
- 所需精度超过 `callout_reset_sbt` 能提供的。

在以下情况使用 `callout(9)`：
- 硬件没有可用的定时器用于你的目的。
- 内核能提供的精度（低至 `tick_sbt` 或带 `_sbt` 的亚 tick）足够。

对于我们的伪设备，不存在硬件；`callout(9)` 是正确的唯一选择。



## 参考：常用 Callout 词汇

本章使用的术语词汇表，在驱动程序源代码中遇到时有用。

**Callout**：`struct callout` 的实例；已调度或未调度的定时器。

**轮（Wheel）**：内核的每 CPU callout 桶数组，按截止时间组织。

**桶（Bucket）**：轮的一个元素，包含应该在很小时间范围内触发的 callout 列表。

**待处理（Pending）**：callout 在轮上等待触发的状态。

**活动（Active）**：用户管理的位，表示"我调度了这个 callout 且未主动取消它"；与待处理不同。

**触发（Firing）**：callout 的回调当前正在执行的状态。

**空闲（Idle）**：callout 已初始化但未待处理的状态；要么从未调度，要么已触发且未重新武装。

**排空（Drain）**：等待进行中的回调完成的操作（通常在分离时）；`callout_drain`。

**停止（Stop）**：不等待取消待处理 callout 的操作；`callout_stop`。

**直接执行（Direct execution）**：回调在定时器中断上下文本身运行的优化，用 `C_DIRECT_EXEC` 设置。

**迁移（Migration）**：内核将 callout 重定位到不同 CPU（通常因为原来绑定的 CPU 离线）。

**锁感知（Lock-aware）**：用锁变体之一（`callout_init_mtx`、`_rw` 或 `_rm`）初始化的 callout；内核为每次触发获取锁。

**Mpsafe**：旧术语，表示"可以在不获取 Giant 的情况下调用"；在现代使用中作为 `callout_init` 的 `mpsafe` 参数出现。

**重新武装（Re-arm）**：回调调度同一个 callout 下一次触发的动作。



## 参考：要避免的定时器反模式

看起来合理但错误的模式的简短目录。

**反模式 1：紧密循环轮询。** 一些驱动程序，特别是初学者编写的，忙等待硬件寄存器：`while (!(read_reg() & READY)) ; /* keep checking */`。这消耗 CPU 并产生负载下无响应的系统。基于 callout 的轮询模式是治愈：调度一个检查寄存器并重新武装的回调。

**反模式 2：硬编码间隔。** 到处硬编码"等待 100 ms"的驱动程序是难以调整的驱动程序。使间隔成为用户可以调整的 sysctl 或 softc 字段。

**反模式 3：分离时缺少排空。** 最常见的第 13 章错误。卸载竞争使内核崩溃。始终排空。

**反模式 4：回调中睡眠。** 回调在持有不可睡眠锁时运行；睡眠是禁止的。如果工作需要睡眠，延迟到内核线程或 taskqueue。

**反模式 5：为新代码使用 `callout_init`（旧式变体）。** 锁简单变体要求你在回调内部自己做所有锁定，这比让内核做更容易出错。为新代码使用 `callout_init_mtx`。

**反模式 6：在多个回调之间共享 `struct callout`。** `struct callout` 不是队列。如果你需要触发两个不同的回调，使用两个 `struct callout`。

**反模式 7：在持有 callout 的锁时调用 `callout_drain`。** 导致死锁。先释放锁。

**反模式 8：将同一个锁设置为多个不相关子系统的 callout 互锁。** 序列化可能产生令人惊讶的锁竞争。每个子系统通常应该有自己的锁；仅当工作真正相关时共享。

**反模式 9：在 `callout_drain` 后不重新初始化就重用 `struct callout`。** 排空后，callout 的内部状态被重置，但上次 `callout_reset` 的函数和参数仍然存在。如果你接着 `callout_schedule`，你会重用那些。这很微妙。为清晰起见，在重用前再次调用 `callout_init_mtx`。

**反模式 10：忘记 `callout_stop` 不等待。** 在正常操作中这是正确的；在分离时是错误的。为分离使用 `callout_drain`。

这些模式经常出现，值得记忆。避免所有十个的驱动程序会轻松得多。



## 参考：用 dtrace 跟踪 Callout

用于检查 callout 行为的 `dtrace` 配方简短集合。每个一两行；它们一起覆盖大多数诊断需求。

### 统计特定回调的触发次数

```sh
# dtrace -n 'fbt::myfirst_heartbeat:entry { @ = count(); } tick-1sec { printa(@); trunc(@); }'
```

心跳回调每秒运行多少次的计数。用于确认配置的速率。

### 回调耗时直方图

```sh
# dtrace -n '
fbt::myfirst_heartbeat:entry { self->ts = timestamp; }
fbt::myfirst_heartbeat:return /self->ts/ {
    @ = quantize(timestamp - self->ts);
    self->ts = 0;
}
tick-30sec { exit(0); }'
```

回调持续时间分布，以纳秒为单位。用于发现异常慢的触发。

### 跟踪所有 Callout 重置

```sh
# dtrace -n 'fbt::callout_reset_sbt_on:entry { printf("co=%p, fn=%p, arg=%p", arg0, arg3, arg4); }'
```

每个 `callout_reset`（及其变体）调用。用于确认哪些代码路径在调度 callout。

### 跟踪 Callout 排空

```sh
# dtrace -n 'fbt::_callout_stop_safe:entry /arg1 == 1/ { printf("drain co=%p", arg0); stack(); }'
```

每次对排空路径的调用（`flags == CS_DRAIN`）。用于确认分离调用对每个 callout 排空。

### 每 CPU Callout 活动

```sh
# dtrace -n 'fbt::callout_process:entry { @[cpu] = count(); } tick-1sec { printa(@); trunc(@); }'
```

每个 CPU 上 callout 处理调用的每秒计数。告诉你哪些 CPU 在做定时器工作。

### 识别慢 Callout

```sh
# dtrace -n '
fbt::callout_process:entry { self->ts = timestamp; }
fbt::callout_process:return /self->ts/ {
    @ = quantize(timestamp - self->ts);
    self->ts = 0;
}
tick-30sec { exit(0); }'
```

callout 处理循环耗时分布。长持续时间表示要么许多 callout 同时触发，要么单个回调慢。

### 综合诊断脚本

用于每秒读取：

```sh
# dtrace -n '
fbt::callout_reset_sbt_on:entry { @resets = count(); }
fbt::_callout_stop_safe:entry /arg1 == 1/ { @drains = count(); }
fbt::myfirst_heartbeat:entry { @hb = count(); }
fbt::myfirst_watchdog:entry { @wd = count(); }
fbt::myfirst_tick_source:entry { @ts = count(); }
tick-1sec {
    printa("resets=%@u drains=%@u hb=%@u wd=%@u ts=%@u\n",
        @resets, @drains, @hb, @wd, @ts);
    trunc(@resets); trunc(@drains);
    trunc(@hb); trunc(@wd); trunc(@ts);
}'
```

每秒一行压缩诊断。在开发期间用作健全性检查。



## 参考：从 DDB 检查 Callout 状态

当系统挂起且你需要从调试器检查 callout 状态时，几个 DDB 命令有帮助。

### `show callout <addr>`

如果你知道 callout 的地址，这显示其当前状态：待处理与否、调度的截止时间、回调函数指针、参数。当你知道要检查哪个 callout 时有用。

### `show callout_stat`

转储整体 callout 统计：多少已调度、启动以来多少触发、多少待处理。用于系统范围概览。

### `ps`

标准进程列表。callout 处理内的线程通常命名为 `clock` 或类似。它们通常在 `mi_switch` 或正在执行的回调中。

### `bt <thread>`

特定线程的反向跟踪。如果线程在 callout 回调内，反向跟踪显示调用链：底部的内核 callout 子系统，顶部的回调。这告诉你哪个回调正在运行。

### `show all locks`

如果 callout 的回调当前正在执行，反向跟踪会显示 `mtx_lock`（内核获取 callout 的锁）。`show all locks` 确认哪个锁被哪个线程持有。

### 综合：检查卡住的 Callout

```text
db> show all locks
... 显示线程 1234 持有 myfirst0 互斥锁

db> ps
... 1234 是 "myfirst_heartbeat"（或类似）

db> bt 1234
... 反向跟踪显示 _cv_wait 或类似；回调正在睡眠（它不应该！）
```

如果你看到这个，回调正在做非法的事情（持有不可睡眠锁时睡眠）。修复是从回调中移除睡眠操作。



## 参考：比较基于 Tick 和基于 SBT 的 API

两个 callout API（基于 tick 和基于 sbintime）值得并排比较。

### 基于 Tick 的 API

```c
callout_reset(&co, ticks, fn, arg);
callout_schedule(&co, ticks);
```

延迟以 tick 表示：时钟中断的整数计数。在 1000 Hz 内核上，一个 tick 是一毫秒。将秒乘以 `hz` 转换；例如，`5 * hz` 表示五秒，`hz / 10` 表示 100 ms。

优点：简单、知名、快速（无 sbintime 算术）。
缺点：精度限于一个 tick（典型 1 ms）；无法表达亚 tick 延迟。

用于：大多数 callout 工作。秒级间隔的看门狗、百毫秒间隔的心跳、几十毫秒间隔的周期性轮询。

### 基于 SBT 的 API

```c
callout_reset_sbt(&co, sbt, prec, fn, arg, flags);
callout_schedule_sbt(&co, sbt, prec, flags);
```

延迟是 `sbintime_t`：高精度二进制定点时间。使用 `SBT_1S`、`SBT_1MS`、`SBT_1US`、`SBT_1NS` 常量构造值。

优点：亚 tick 精度；显式精度/合并参数；显式绝对 vs 相对时间标志。
缺点：更多算术；必须理解 `sbintime_t`。

用于：需要亚毫秒精度的 callout（网络协议、有严格时序要求的硬件控制器）。大多数驱动程序工作不需要这个。

### 转换辅助

```c
sbintime_t  ticks_to_sbt = tick_sbt * timo_in_ticks;  /* tick_sbt 是全局变量 */
sbintime_t  ms_to_sbt = ms_value * SBT_1MS;
sbintime_t  us_to_sbt = us_value * SBT_1US;
```

`tick_sbt` 全局变量给你一个 tick 的 sbintime 等效值；乘以你的 tick 计数来转换。



## 参考：生产前 Callout 审计

在将使用 callout 的驱动程序从开发提升到生产之前执行的简短审计。每个项目是一个问题；每个都应该能自信回答。

### Callout 清单

- [ ] 我是否在 `LOCKING.md` 中列出了驱动程序拥有的每个 callout？
- [ ] 对于每个 callout，我是否命名了其回调函数？
- [ ] 对于每个 callout，我是否命名了它使用的锁（如果有）？
- [ ] 对于每个 callout，我是否记录了其生命周期（attach 中初始化，detach 中排空）？
- [ ] 对于每个 callout，我是否记录了其触发器（什么导致它被调度）？
- [ ] 对于每个 callout，我是否记录了它是重新武装（周期性）还是触发一次（一次性）？

### 初始化

- [ ] 每个 callout 初始化是否使用 `callout_init_mtx`（或 `_rw`/`_rm`）而不是裸 `callout_init`？
- [ ] 初始化是否在它引用的锁初始化之后调用？
- [ ] 锁类型是否正确（可睡眠上下文用睡眠互斥锁等）？

### 调度

- [ ] 每个 `callout_reset` 是否在持有适当锁的情况下发生？
- [ ] 间隔对回调所做的工作是否合理？
- [ ] 从毫秒到 tick 的转换是否正确（`(ms * hz + 999) / 1000` 用于向上取整）？
- [ ] 如果 callout 是周期性的，回调是否仅在记录的条件下重新武装？

### 回调卫生

- [ ] 每个回调是否以 `MYFIRST_ASSERT(sc)`（或等效）开始？
- [ ] 每个回调是否在做工作前检查 `is_attached`？
- [ ] 每个回调是否在 `is_attached == 0` 时提前退出？
- [ ] 回调是否避免睡眠操作（`uiomove`、`cv_wait`、`mtx_sleep`、`malloc(M_WAITOK)`、`selwakeup`）？
- [ ] 回调的总工作时间是否有界？

### 取消

- [ ] sysctl 处理程序是否使用 `callout_stop` 禁用定时器？
- [ ] sysctl 处理程序是否在调用 `callout_stop` 和 `callout_reset` 时持有锁？
- [ ] 是否有任何代码路径可能与 sysctl 处理程序竞争？

### 分离

- [ ] 分离路径是否在调用 `callout_drain` 之前释放互斥锁？
- [ ] 分离路径是否排空每个 callout？
- [ ] callout 是否在正确的阶段排空（在清除 `is_attached` 之后）？

### 文档

- [ ] 每个 callout 是否在 `LOCKING.md` 中记录？
- [ ] 纪律规则（锁感知、不睡眠、分离时排空）是否记录？
- [ ] callout 子系统是否在 README 中提及？
- [ ] 是否有让用户调整行为的暴露 sysctl？

### 测试

- [ ] 我是否在启用 `WITNESS` 的情况下运行了回归套件？
- [ ] 我是否测试了所有 callout 活跃时的分离？
- [ ] 我是否运行了长时间压力测试？
- [ ] 我是否使用 `dtrace` 验证触发速率匹配配置的间隔？

通过此审计的驱动程序是你可以在负载下信任的驱动程序。



## 参考：跨驱动程序标准化定时器

对于有多个 callout 的驱动程序，一致性比聪明更重要。简短纪律。

### 一个命名约定

选择一个约定并遵循它。章节的约定：

- Callout 结构命名为 `<purpose>_co`（例如 `heartbeat_co`、`watchdog_co`、`tick_source_co`）。
- 回调命名为 `myfirst_<purpose>`（例如 `myfirst_heartbeat`、`myfirst_watchdog`、`myfirst_tick_source`）。
- 间隔 sysctl 命名为 `<purpose>_interval_ms`（例如 `heartbeat_interval_ms`、`watchdog_interval_ms`、`tick_source_interval_ms`）。
- Sysctl 处理程序命名为 `myfirst_sysctl_<purpose>_interval_ms`。

新维护者可以按照约定添加新 callout 而不需要考虑名称。相反，代码审查立即捕获偏差。

### 一个初始化/排空模式

每个 callout 使用相同的初始化和排空：

```c
/* 在 attach 中： */
callout_init_mtx(&sc-><purpose>_co, &sc->mtx, 0);

/* 在 detach 中（释放互斥锁后）： */
callout_drain(&sc-><purpose>_co);
```

或者，使用宏：

```c
MYFIRST_CO_INIT(sc, &sc-><purpose>_co);
MYFIRST_CO_DRAIN(&sc-><purpose>_co);
```

宏在其定义中记录模式；调用点短小且统一。

### 一个 Sysctl 处理程序模式

每个间隔 sysctl 处理程序遵循相同结构：

```c
static int
myfirst_sysctl_<purpose>_interval_ms(SYSCTL_HANDLER_ARGS)
{
        struct myfirst_softc *sc = arg1;
        int new, old, error;

        old = sc-><purpose>_interval_ms;
        new = old;
        error = sysctl_handle_int(oidp, &new, 0, req);
        if (error || req->newptr == NULL)
                return (error);
        if (new < 0)
                return (EINVAL);

        MYFIRST_LOCK(sc);
        sc-><purpose>_interval_ms = new;
        if (new > 0 && old == 0) {
                /* 启用 */
                callout_reset(&sc-><purpose>_co,
                    (new * hz + 999) / 1000,
                    myfirst_<purpose>, sc);
        } else if (new == 0 && old > 0) {
                /* 禁用 */
                callout_stop(&sc-><purpose>_co);
        }
        MYFIRST_UNLOCK(sc);
        return (0);
}
```

处理程序的形态对每个间隔 sysctl 相同。添加新 sysctl 是机械的。

### 一个回调模式

每个周期性回调遵循相同结构：

```c
static void
myfirst_<purpose>(void *arg)
{
        struct myfirst_softc *sc = arg;
        int interval;

        MYFIRST_ASSERT(sc);
        if (!sc->is_attached)
                return;

        /* ... 做每次触发的工作 ... */

        interval = sc-><purpose>_interval_ms;
        if (interval > 0)
                callout_reset(&sc-><purpose>_co,
                    (interval * hz + 999) / 1000,
                    myfirst_<purpose>, sc);
}
```

断言、检查 `is_attached`、做工作、有条件重新武装。驱动程序中的每个回调都有此形态；偏差很显眼。

### 一个文档模式

每个 callout 在 `LOCKING.md` 中用相同字段记录：

- 使用的锁。
- 回调函数。
- 行为（周期性或一次性）。
- 由谁启动（哪个代码路径调度它）。
- 由谁停止（哪个代码路径停止它）。
- 生命周期（attach 中初始化，detach 中排空）。

新 callout 的文档是机械的。代码审查可以对照代码验证文档。

### 为什么要标准化

标准化有成本：新贡献者必须学习约定；偏差需要特殊原因。收益更大：

- 减少认知负担。知道模式的读者立即理解每个 callout。
- 更少错误。标准模式正确处理常见情况（锁获取、`is_attached` 检查、排空）；偏差更可能出错。
- 更容易审查。审查者可以扫描形态而不是阅读每一行。
- 更容易交接。未见过驱动程序的维护者可以按照现有模板添加新 callout。

标准化的成本在设计时支付一次。收益永远累积。总是值得。



## 参考：关于定时器的进一步阅读

对于想深入学习的读者：

### 手册页

- `callout(9)`：规范 API 参考。
- `timeout(9)`：旧接口（已弃用；为历史阅读而提及）。
- `microtime(9)`、`getmicrouptime(9)`、`getsbinuptime(9)`：callout 经常使用的时间读取原语。
- `eventtimers(4)`：驱动 callout 的事件定时器子系统。
- `kern.eventtimer`：暴露事件定时器状态的 sysctl 树。

### 源文件

- `/usr/src/sys/kern/kern_timeout.c`：callout 实现。
- `/usr/src/sys/kern/kern_clocksource.c`：事件定时器驱动层。
- `/usr/src/sys/sys/callout.h`、`/usr/src/sys/sys/_callout.h`：公共 API 和结构。
- `/usr/src/sys/sys/time.h`：sbintime 常量和转换宏。
- `/usr/src/sys/dev/led/led.c`：展示 callout 模式的小驱动程序。
- `/usr/src/sys/dev/uart/uart_core.c`：更详细的使用，包括不为输入中断的硬件的轮询回退。

### 按顺序阅读的手册页

对于刚接触 FreeBSD 时间子系统的读者，合理的阅读顺序：

1. `callout(9)`：规范 API 参考。
2. `time(9)`：单位和原语。
3. `eventtimers(4)`：驱动 callout 的事件定时器子系统。
4. `kern.eventtimer` 和 `kern.hz` sysctl：运行时控制。
5. `microuptime(9)`、`getmicrouptime(9)`：时间读取原语。
6. `kproc(9)`、`kthread(9)`：当你真正需要内核线程时。

每个建立在上一个之上；按顺序阅读需要几个小时，给你内核时间基础设施的扎实心智模型。

### 外部材料

*The Design and Implementation of the FreeBSD Operating System*（McKusick 等人）中关于定时器的章节覆盖定时器子系统的历史演进和当前设计背后的推理。作为背景有用；不要求。

FreeBSD 开发者邮件列表（`freebsd-hackers@`）偶尔讨论 callout 改进和边缘情况。在档案中搜索"callout"返回 API 如何演进的相关历史背景。

为了更深入理解内核如何在最低级别调度事件，`eventtimers(4)` 手册页和 `/usr/src/sys/kern/kern_clocksource.c` 下的源代码值得仔细阅读。它们低于本章级别（我们不直接与事件定时器交互），但它们解释了为什么 callout 子系统能提供它所提供的精度。

最后，真实驱动程序源代码。选择 `/usr/src/sys/dev/` 中任何使用 callout 的驱动程序（大多数都使用），阅读其 callout 相关代码，并与本章的模式比较。转换是直接的；你会立即识别形态。那种阅读将章节的抽象转化为工作知识。



## 参考：Callout 成本分析

关于 callout 实际成本的简短讨论，在决定间隔或设计高频率定时器时有用。

### 静态成本

未调度的 `struct callout` 除了结构大小（amd64 上约 80 字节）外没有成本。内核不知道它。它坐在你的 softc 中，什么都不做。

已调度但尚未触发的 `struct callout` 成本略高：内核已将其链接到轮桶中。链接条目花费几个字节。内核不轮询结构；只在相关桶到期时查看它。

硬件时钟中断（驱动轮）每秒触发 `hz` 次（通常 1000）。在空情况下（无 callout 到期）基本上零成本，在忙情况下与到期 callout 数量成正比。

### 每次触发成本

当 callout 触发时，内核大致做：

1. 遍历轮桶；找到 callout。桶中每个 callout 常数时间。
2. 获取 callout 的锁（如果有）。成本取决于竞争；通常纳秒级。
3. 调用回调函数。成本取决于回调。
4. 释放锁。微秒级。

对于典型的短回调（几微秒工作），每次触发成本由回调本身加锁获取主导。内核开销可忽略。

### 取消/排空成本

`callout_stop` 快：链表移除加原子标志更新。微秒级。

`callout_drain` 如果 callout 空闲则快（就像 `callout_stop`）。如果回调当前正在触发，排空通过睡眠队列机制等待；等待时间取决于回调需要多长时间。

### 实际影响

数百个待处理 callout：没问题。轮高效处理它们。

数千个待处理 callout：正常操作下仍然没问题。遍历几十个 callout 的轮桶是快的。

单个 1 Hz 触发的 callout：基本免费。千分之一的一个硬件中断遍历桶并找到 callout。

单个 1 kHz 触发的 callout：开始可测量。每秒一千次回调累积。如果回调需要 10 微秒，那是一个 CPU 的 1%。如果回调更重，更多。

10 kHz 或更快的 callout：可能是错误设计。考虑忙轮询或硬件定时器或专门机制。

### 与其他方法的比较

循环 `cv_timedwait` 并在每次唤醒做工作的内核线程成本：

- 内存：~16 KB 栈。
- 每次唤醒：调度器进入、上下文切换、回调、上下文切换回来。

对于 1 Hz 工作负载，内核线程成本（每秒一次唤醒）与 callout 成本大致相同。对于 1 kHz 工作负载，两者相似。对于 10 kHz 工作负载，两者都变昂贵；考虑你是否真的需要那个频率。

轮询 sysctl 的用户空间循环：

- 内存：整个用户进程（MB 级）。
- 每次轮询：syscall 往返、sysctl 处理程序调用、返回用户空间。

总是比内核 callout 更昂贵。仅在轮询逻辑真正属于用户空间（监控工具、外部探测）时合适。

### 何时担心成本

大多数驱动程序不需要。Callout 便宜；内核调优良好。仅在以下情况担心成本：

- 分析显示 callout 主导 CPU 使用。（用 `dtrace` 确认。）
- 你正在编写高频率驱动程序（有严格延迟要求的网络或存储）。
- 系统有数千个活跃 callout 且你想理解负载。

在所有其他情况下，自然地编写 callout 并信任内核处理负载。



## 参考：第 13 章词汇表

快速查找的术语词汇表。

**Callout**：`struct callout` 的实例；已调度或未调度的定时器。

**轮（Wheel）**：内核的每 CPU callout 桶数组，按截止时间组织。

**桶（Bucket）**：轮的一个元素，包含应该在很小时间范围内触发的 callout 列表。

**待处理（Pending）**：callout 在轮上等待触发的状态。

**活动（Active）**：用户管理的位，表示"我调度了这个 callout 且未主动取消它"；与待处理不同。

**触发（Firing）**：callout 的回调当前正在执行的状态。

**空闲（Idle）**：callout 已初始化但未待处理的状态；要么从未调度，要么已触发且未重新武装。

**排空（Drain）**：等待进行中的回调完成的操作（通常在分离时）；`callout_drain`。

**停止（Stop）**：不等待取消待处理 callout 的操作；`callout_stop`。

**直接执行（Direct execution）**：回调在定时器中断上下文本身运行的优化，用 `C_DIRECT_EXEC` 设置。

**迁移（Migration）**：内核将 callout 重定位到不同 CPU（通常因为原来绑定的 CPU 离线）。

**锁感知（Lock-aware）**：用锁变体之一初始化的 callout；内核为每次触发获取锁。

**Mpsafe**：旧术语，表示"可以在不获取 Giant 的情况下调用"。

**重新武装（Re-arm）**：回调调度同一个 callout 下一次触发的动作。


## 展望：通往第 14 章的桥梁

第 14 章标题为*任务队列与延迟工作*。其范围是从驱动程序视角看内核的延迟工作框架：如何将工作从一个无法安全运行它的上下文（callout 回调、中断处理程序、epoch 区段）移到一个可以运行它的上下文。

第 13 章以三种具体方式为此奠定了基础。

首先，你已经知道 callout 回调在严格的上下文约束下运行：不能睡眠、不能获取可睡眠锁、不能 `uiomove`、不能 `copyin`、不能 `copyout`，以及不能在持有驱动程序互斥锁时调用 `selwakeup`。你在 `myfirst_tick_source` 中看到了这个约束的实施，其中故意省略了 `selwakeup`，因为 callout 上下文不能合法地进行该调用。第 14 章介绍 `taskqueue(9)`，这正是内核为这种传递提供的原语：callout 将任务入队，任务在线程上下文中运行，在那里被省略的调用是合法的。

其次，你已经知道分离时排空的规则。`callout_drain` 确保分离进行时没有回调在运行。任务有一个匹配的原语：`taskqueue_drain` 等待直到特定任务既不待处理也不运行。心智模型是相同的；顺序增加一步（先 callout，再任务，然后是它们影响的所有内容）。

第三，你已经知道 `LOCKING.md` 作为活文档的形式。第 14 章扩展它，添加一个任务部分，命名每个任务、其回调、其生命周期及其在分离顺序中的位置。规则相同；词汇稍宽。

第 14 章将涵盖的具体主题：

- `taskqueue(9)` API：`struct task`、`TASK_INIT`、`taskqueue_create`、`taskqueue_start_threads`、`taskqueue_enqueue`、`taskqueue_drain`、`taskqueue_free`。
- 预定义的系统任务队列（`taskqueue_thread`、`taskqueue_swi`、`taskqueue_fast`、`taskqueue_bus`）以及何时使用私有任务队列更合适。
- 合并规则：当任务在已待处理时入队会发生什么。
- `struct timeout_task` 和 `taskqueue_enqueue_timeout` 用于延迟且已调度的工作。
- 在真实 FreeBSD 驱动程序中重复出现的模式，以及出错时的调试故事。

你不需要提前阅读。第 13 章已足够准备。带上你的 `myfirst` 驱动程序（第 13 章第 4 阶段）、你的测试工具包和启用 `WITNESS` 的内核。第 14 章从第 13 章结束的地方开始。

一个小小的结束语。你开始本章时的驱动程序无法自主行动：每一行工作都由用户做的事情触发。你离开时拥有一个具有内部时间的驱动程序，它周期性记录其状态，检测停滞的排水，注入合成事件用于测试，并在卸载模块时干净地拆除所有这些基础设施。这是一个真正的质的飞跃，这些模式直接迁移到第 4 部分将介绍的每种驱动程序。

暂停片刻。你开始第 3 部分时的驱动程序知道如何一次处理一个线程。你现在的驱动程序协调多个线程，支持可配置的定时工作，并无竞态地拆除。从这里开始，第 14 章添加*任务*，这是任何定时器回调需要触发 callout 无法安全执行的工作的驱动程序所缺少的部分。然后翻页。

### 关于时间的最后附注

在第 14 章之前的最后一个想法。你花了两章学习同步（第 12 章）和一章学习时间（第 13 章）。两者深度相关：同步的核心是关于事件彼此相对*何时*发生，而时间是它的显式度量。锁序列化访问；cv 协调等待；callout 在截止时间触发。这三者是切分同一个潜在问题的不同方式：独立的执行流如何就顺序达成一致？

第 14 章添加第四块：*上下文*。Callout 在精确时刻触发，但它触发的上下文（不能睡眠、不能获取可睡眠锁、不能进行用户空间复制）比大多数实际工作需要的要窄。通过 `taskqueue(9)` 的延迟工作是从那个窄上下文到线程上下文的桥梁，在那里完整的内核操作集是合法的。

模式迁移。Callout 为其回调使用的锁感知初始化形状与你决定任务回调获取哪个锁时应用的形状相同。Callout 在分离时使用的排空模式与任务在拆除时使用的形状相同。Callout 要求的"在这里少做，推迟其余"的规则是第 14 章给你一个具体工具来遵循的规则。

所以当你进入第 14 章时，框架已经很熟悉。你将向驱动程序工具包添加一个原语。它的规则与你现在知道的 callout 规则干净地组合。你构建的工具（LOCKING.md、七阶段分离、断言并检查已附加模式）将吸收新原语而不变得脆弱。

这就是使本书第 3 部分作为一个单元运作的原因。每一章为驱动程序对世界的感知增加一个维度（并发、同步、时间、延迟工作），每一章都建立在前一章的基础设施之上。到第 3 部分结束时，你的驱动程序将为第 4 部分及其背后的真实硬件做好准备。
