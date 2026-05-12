---
title: "同步机制"
description: "命名你等待的通道，一次获取锁并从多个线程读取，限制无限期阻塞，并将第11章的互斥锁转化为可以维护的同步设计。"
partNumber: 3
partName: "并发与同步"
chapter: 12
lastUpdated: "2026-04-18"
status: "complete"
author: "Edson Brandi"
reviewer: "待定"
translator: "AI辅助翻译为简体中文"
estimatedReadTime: 195
language: "zh-CN"
---

# 同步机制

## 读者指南与学习目标

第11章结束时，我们的驱动程序首次在本书中变得*可验证地*并发。你有一个保护环形缓冲区的互斥锁、可扩展到多核的原子计数器、监视每次锁获取的`WITNESS`和`INVARIANTS`、运行在调试内核上的压力测试套件，以及任何未来维护者（包括未来的你）都可以阅读以理解并发故事的`LOCKING.md`文档。那是真正的进步。但这并不是故事的结局。

FreeBSD为你提供的同步工具包远大于第11章介绍的单个原语。你使用的互斥锁对许多情况来说是正确答案，但对其他几种情况来说也是错误答案。一个被二十个线程每秒扫描一万次的读多配置表需要的不是将读取序列化的互斥锁。一个应该在毫秒内响应Ctrl-C的阻塞读取需要的不是没有超时的`mtx_sleep(9)`。一个跨十几个等待者、每个等待一个条件的协调唤醒需要的比`&sc->cb`这样的匿名通道指针更具表现力。内核为每种情况都有原语，第12章正是我们要认识它们的地方。

本章为同步工具包的其他部分做了第11章为互斥锁所做的事。我们首先介绍每个原语及其动机，建立心智模型，在真实的FreeBSD源代码中验证它，将其应用于运行中的`myfirst`驱动程序，并在`WITNESS`内核上验证结果。到本章结束时，你将能够阅读真实的驱动程序，仅凭原语的选择就能识别其作者想要表达什么。你还将能够在自己的驱动程序中做出这些选择，而不会出于习惯而选错工具。

### 为什么本章值得独立成章

原则上，在第11章停下来是可能的。一个互斥锁、一个原子计数器和`mtx_sleep`涵盖了简单情况。FreeBSD树中的许多小型驱动程序不使用其他任何东西。

问题在于"许多小型驱动程序"并不是大多数bug存在的地方。人们维护时间最长的驱动程序是那些成长的驱动程序。一个USB设备驱动程序开始时很小，然后获得了控制通道，然后是用户空间可以在运行时更改的配置表，然后是有自己等待者的独立事件队列。这些添加中的每一个都暴露了"一个互斥锁保护一切"的限制。只知道互斥锁的驱动程序编写者最终要么误用它（持有时间过长的互斥锁阻塞整个子系统），要么绕过它（一堆忙等待循环、竞争性重试和"不应该发生但有时会发生"的全局标志）。本章教授的同步原语正是为了让那些变通方案远离代码而存在的。

每个原语都是线程之间不同形式的协议。互斥锁说*同一时间只有我们中的一个*。条件变量说*我会等待一个特定的变化，你会告诉我*。共享/独占锁说*我们中的许多人可以读；只有我们中的一个可以写*。定时睡眠说*如果花费太长时间，请放弃*。为每个工具适合的用途使用它的驱动程序读起来清晰，行为可预测，并且在原作者停止查看很久之后仍然可理解。对所有事情使用一个工具的驱动程序要么在性能上受损，要么在没人查看的地方隐藏bug。

因此，本章既是词汇章节，也是机制章节。我们确实介绍了API，我们也确实演练了代码。更深层次的目标是给你表达你想说的话的词汇。

### 第11章留给驱动程序的状态

快速检查一下，因为第12章直接建立在第11章交付物之上。如果以下任何内容缺失或感觉不确定，请在开始本章之前返回第11章。

- 你的`myfirst`驱动程序使用`WARNS=6`干净地编译。
- 它使用`MYFIRST_LOCK(sc)`、`MYFIRST_UNLOCK(sc)`和`MYFIRST_ASSERT(sc)`宏，这些宏展开为设备范围`sc->mtx`（一个`MTX_DEF`睡眠互斥锁）上的`mtx_lock`、`mtx_unlock`和`mtx_assert(MA_OWNED)`。
- cbuf、每个描述符的计数器、打开计数和活动描述符计数都由`sc->mtx`保护。
- 字节计数器`sc->bytes_read`和`sc->bytes_written`是`counter_u64_t`每CPU计数器；它们不需要互斥锁。
- 阻塞读写路径使用`mtx_sleep(&sc->cb, &sc->mtx, PCATCH, "myfrd"|"myfwr", 0)`作为它们的等待原语，使用`wakeup(&sc->cb)`作为匹配的唤醒。
- `INVARIANTS`和`WITNESS`在你的测试内核中启用；你已经构建并启动了它。
- 一个`LOCKING.md`文档伴随驱动程序，列出了每个共享字段、每个锁、每个等待通道和每个"故意不加锁"的决定及其理由。
- 第11章压力套件（`producer_consumer`、`mp_stress`、`mt_reader`、`lat_tester`）构建并干净地运行。

该驱动程序是本章的起点。我们不会抛弃它。我们将用更适合的原语替换一些原语，添加一个小的子系统让新原语有东西可以保护，并完成一个同步设计既更强大又更易读的驱动程序。

### 你将学到什么

当你结束本章时，你应该能够：

- 解释*同步*在内核级别意味着什么，并将其与*互斥*这个更窄的概念区分开来。
- 将FreeBSD同步工具包映射到一个小决策树：互斥锁、条件变量、共享/独占锁、读写锁、原子、带超时的睡眠。
- 用命名条件变量（`cv(9)`）替换匿名等待通道，并解释为什么该更改改善了正确性和可读性。
- 正确使用`cv_wait`、`cv_wait_sig`、`cv_wait_unlock`、`cv_signal`、`cv_broadcast`和`cv_broadcastpri`及其互锁互斥锁。
- 使用`cv_timedwait_sig`（或带有非零`timo`参数的`mtx_sleep`）通过超时限制阻塞操作，并设计超时触发时调用者的响应。
- 区分`EINTR`、`ERESTART`和`EWOULDBLOCK`，并决定等待失败时驱动程序应该返回哪个。
- 在`sx(9)`（可睡眠）和`rw(9)`（基于自旋）读写锁之间选择，理解每个对调用上下文施加的规则，并应用`sx_init`、`sx_xlock`、`sx_slock`、`sx_xunlock`、`sx_sunlock`、`sx_try_upgrade`和`sx_downgrade`。
- 设计一个多读者、单写者安排，为数据路径使用一个锁，为配置路径使用另一个锁，有文档化的锁顺序和`WITNESS`验证的不存在反转。
- 足够精确地阅读`WITNESS`警告，以识别确切的锁对和违规的源代码行。
- 使用内核调试器命令`show locks`、`show all locks`、`show witness`和`show lockchain`检查因同步bug而挂起的系统。
- 构建一个同时跨描述符、sysctl和定时等待测试驱动程序的压力工作负载，并阅读`lockstat(1)`的输出以找到竞争原语。
- 将驱动程序重构为同步故事被文档化、锁顺序明确、版本字符串反映新架构的形式。

这是一个相当长的列表。对于一个希望比作者更长久的驱动程序来说，其中没有任何一项是可选的。所有这些都建立在第11章留给你的东西之上。

### 本章不涵盖的内容

几个相关主题被有意推迟：

- **Callouts（`callout(9)`）。** 第13章介绍从内核时钟基础设施触发的定时工作。我们在这里只作为从驱动程序阻塞调用角度看到的睡眠-带-超时原语来涉及该主题；完整的callout API及其规则属于第13章。
- **Taskqueues（`taskqueue(9)`）。** 第16章介绍内核的延迟工作框架。几个驱动程序使用taskqueue来解耦阻塞线程和唤醒信号，但做好它需要自己的章节。
- **`epoch(9)`和读多无锁模式。** 特别是网络驱动程序使用`epoch(9)`让读者完全不获取锁地继续。该机制微妙，最好与网络驱动程序子系统一起在第6部分教授。
- **中断上下文同步。** 真实的硬件中断处理程序增加了另一层约束，关于你可以持有哪些锁以及哪些睡眠原语是合法的。第14章介绍中断处理程序，并从中断上下文内部重新审视同步规则。对于第12章，我们完全保持在进程和内核线程上下文中。
- **无锁数据结构。** `buf_ring(9)`和朋友们是热路径的有效工具，但它们值得仔细研究，并且需要特定的工作负载来回报其复杂性。第6部分（第28章）在本书中的驱动程序实际需要时介绍它们。
- **分布式和跨机器同步。** 超出范围。我们在本书中是一个单主机操作系统。

保持在这些界限内使本章专注于它能很好教授的内容。第12章的读者应该以对`cv(9)`、`sx(9)`和定时等待的自信控制结束，并对`rw(9)`和`epoch(9)`适合哪里有工作感；这种自信使后面章节在它们出现时可读。

### 预计时间投入

- **仅阅读**：约三小时。新词汇（条件变量、共享/独占锁、可睡眠性规则）需要时间吸收，即使API面很小。
- **阅读加上输入工作示例**：分两次会话六到八小时。驱动程序分四个小阶段演化；每个阶段添加一个原语。
- **阅读加上所有实验和挑战**：分三或四次会话十到十四小时，包括压力运行和`lockstat(1)`分析的时间。

如果你发现自己在第4节中间感到困惑，那是正常的。共享/独占区别即使对于熟悉互斥锁的读者来说也是真正的新内容，对数据路径使用`sx`的诱惑正是第5节存在要解决的诱惑。停下来，重读第4节的示例，在模型稳定后继续。

### 先决条件

在开始本章之前，确认：

- 你的驱动程序源代码与第11章阶段3（counter9）或阶段5（KASSERTs）树匹配。阶段5更受推荐，因为断言更快地捕获新bug。
- 你的实验机器运行FreeBSD 14.3，磁盘上有`/usr/src`并与运行中的内核匹配。
- 一个调试内核，启用了`INVARIANTS`、`WITNESS`、`WITNESS_SKIPSPIN`、`DDB`、`KDB`和`KDB_UNATTENDED`，已构建、安装并干净地启动。第11章参考部分"构建和启动调试内核"有配方。
- 你仔细阅读了第11章。互斥锁规则、睡眠-带-互斥锁规则、锁顺序纪律和`WITNESS`工作流程都是这里的假设知识。
- 你至少运行过一次第11章压力套件并看到它通过。

如果以上任何一项不稳定，现在修复它是比硬撑第12章并试图从移动的基础上调试更好的投资。

### 如何从本章获得最大收益

三个习惯会很快回报。

首先，保持`/usr/src/sys/kern/kern_condvar.c`、`/usr/src/sys/kern/kern_sx.c`和`/usr/src/sys/kern/kern_rwlock.c`有书签。每一个都很短、注释良好，是原语实际做什么的权威来源。本章中几次我们会让你看一个特定函数。花在那里的一分钟会使周围的段落更容易吸收。

> **关于行号的说明。** 当我们在本章后面指向特定函数时，打开文件并搜索符号，而不是跳转到数字行。`_cv_wait_sig`位于`/usr/src/sys/kern/kern_condvar.c`，`sleepq_signal`位于`/usr/src/sys/kern/subr_sleepqueue.c`，在编写时14.3树中；名称将延续到未来的点发布，但每个名称所在的行不会。持久的引用始终是符号。

其次，在`WITNESS`下运行你进行的每个代码更改。本章介绍的同步原语比互斥锁有更严格的规则。`WITNESS`是发现你违反了其中一个规则的最便宜方式。第11章参考部分"构建和启动调试内核"详细介绍了内核构建，如果你需要的话；现在不要跳过它。

第三，尽可能手动输入驱动程序更改。`examples/part-03/ch12-synchronization-mechanisms/`下的配套源代码是规范版本，但手动输入`cv_wait_sig(&sc->data_cv, &sc->mtx)`一次的肌肉记忆比阅读十次更有价值。本章增量显示更改；在你自己的驱动程序副本中镜像该增量节奏。

### 本章路线图

各节顺序如下：

1. 内核中的同步是什么，以及各种原语在小决策树中的位置。
2. 条件变量，匿名唤醒通道的更干净替代方案，以及`myfirst`阻塞路径的第一次重构。
3. 超时和可中断睡眠，包括信号处理以及`EINTR`、`ERESTART`和`EWOULDBLOCK`之间的选择。
4. `sx(9)`锁，它终于让我们能够表达"多读者，偶尔写者"而不序列化每个读者。
5. 驱动程序中的多读者、单写者场景：一个小的配置子系统、数据路径和配置路径之间的锁顺序，以及保持正确的`WITNESS`纪律。
6. 调试同步问题，包括`WITNESS`的仔细浏览、用于检查锁的内核调试器命令，以及最常见的死锁模式。
7. 在现实I/O模式下的压力测试，使用`lockstat(1)`、`dtrace(1)`和为新原语扩展的现有第11章测试器。
8. 重构和版本化驱动程序：干净的`LOCKING.md`、升级的版本字符串、更新的变更日志，以及验证整个内容的回归运行。

动手实验和挑战练习紧随其后，然后是故障排除参考、总结部分和通往第13章的桥梁。

如果这是你第一次阅读，请线性阅读并按顺序进行实验。如果你是重读，调试部分和重构部分独立存在，是很好的单次阅读材料。

## 第1节：内核中的同步是什么？

第11章使用*同步*、*锁定*、*互斥*和*协调*这些词时有些互换。当桌上唯一的原语是互斥锁时，这是可接受的，因为互斥锁将所有这些想法折叠成一个机制。随着第12章介绍的更广泛工具包，这些词开始意味着不同的事情，尽早把它们弄清楚可以防止以后很多混淆。

本节建立词汇。它还绘制了我们将在本章其余部分反复引用的小决策树，当我们问"我应该在这里使用哪个原语？"时。

### 同步意味着什么

**同步**是更广泛的想法：两个或多个并发执行线程协调它们对共享状态的访问、它们通过共享过程的进度，或它们相对于彼此的时序的任何机制。

三种协调风格涵盖了驱动程序需要的几乎所有东西：

**互斥**：一次最多一个线程在临界区内。互斥锁和独占锁提供这个。保证是结构性的：当你在里面时，没有其他人在。

**带限制写入的共享访问**：多个线程可以同时检查一个值，但想要更改它的线程必须等到所有人都出去，没有其他人在。共享/独占锁提供这个。保证是不对称的：读者容忍彼此；写者不容忍任何人。

**协调等待**：一个线程挂起直到某个条件变为真，另一个知道条件已变为真的线程唤醒等待者。条件变量和较旧的`mtx_sleep`/`wakeup`通道机制提供这个。保证是时间性的：等待者在等待时不消耗CPU；唤醒者不必知道谁在等待；内核处理会合。

驱动程序通常使用这三种。`myfirst`中的cbuf已经使用了两种：保护cbuf状态的互斥，以及当缓冲区为空时挂起读者的协调等待。第12章添加第三种（共享访问）并改进第二种（命名条件变量代替匿名通道）。

### 同步与锁定

很容易认为*同步*和*锁定*是同一个词。它们不是。

**锁定**是同步的一种技术。它是在共享对象上操作并授予或拒绝访问它的机制家族。互斥锁、sx锁、rw锁和lockmgr锁都是锁。

**同步**包括锁定，但也包括协调等待（条件变量、睡眠通道）、事件信号（信号量）和定时协调（callout、定时睡眠）。一个等待的线程在它被挂起时可能没有持有任何锁（事实上，对于`mtx_sleep`和`cv_wait`，锁在等待期间被释放），然而它正在参与与最终会唤醒它的线程的同步。

从这个区别得出的心智模型是有用的：锁定是*关于访问*；协调等待是*关于进度*。大多数非平凡的驱动程序代码混合两者。cbuf周围的互斥锁是锁定。当缓冲区为空时在`&sc->cb`上睡眠是协调等待。两者都是同步。单独任何一个都不够。

### 阻塞与自旋

两种基本形状在FreeBSD原语中反复出现。知道原语使用哪种形状是选择它们的一半战斗。

**阻塞原语**将竞争线程放在内核的睡眠队列上睡眠。睡眠线程不消耗CPU；当持有线程释放或等待的条件被信号通知时，它将再次变得可运行。阻塞原语适用于等待可能很长、线程处于睡眠合法的上下文、以及让CPU忙于紧密重试循环会损害整体吞吐量的情况。`MTX_DEF`互斥锁、`cv_wait`、`sx_xlock`、`sx_slock`和`mtx_sleep`都是阻塞的。

**自旋原语**保持线程在CPU上并在锁状态上忙等待，原子地重试直到持有者释放。它们仅适用于临界区非常短、线程不能合法睡眠（例如，在硬件中断过滤器内部）、或上下文切换的成本会使等待相形见绌的情况。`MTX_SPIN`互斥锁和`rw(9)`锁是自旋的。内核本身为调度器和中断机制的最底层使用自旋锁。

第12章主要停留在阻塞世界。我们的驱动程序在进程上下文中运行；它被允许睡眠；自旋的收益会是微不足道的。一个例外是当我们为完整性提到`rw(9)`作为`sx(9)`的兄弟时；对`rw(9)`的更深入处理属于驱动程序有真正理由使用它的章节。

### FreeBSD原语小地图

FreeBSD同步工具包比人们预期的要大。对于驱动程序工作，八个原语基本上承担了所有负载：

| 原语 | 头文件 | 行为 | 最适合 |
|---|---|---|---|
| `mtx(9)` (`MTX_DEF`) | `sys/mutex.h` | 睡眠互斥锁；一次一个所有者 | 大多数softc状态的默认锁 |
| `mtx(9)` (`MTX_SPIN`) | `sys/mutex.h` | 自旋互斥锁；禁用中断 | 中断上下文中的短临界区 |
| `cv(9)` | `sys/condvar.h` | 命名等待通道；与互斥锁配对 | 有多个不同条件的协调等待 |
| `sx(9)` | `sys/sx.h` | 睡眠模式共享/独占锁 | 进程上下文中的读多状态 |
| `rw(9)` | `sys/rwlock.h` | 自旋模式读写锁 | 中断或短临界区中的读多状态 |
| `rmlock(9)` | `sys/rmlock.h` | 读多锁；读便宜，写昂贵 | 热读路径与罕见配置更改 |
| `sema(9)` | `sys/sema.h` | 计数信号量 | 资源计数；驱动程序中很少需要 |
| `epoch(9)` | `sys/epoch.h` | 带延迟回收的读多同步 | 网络/存储驱动程序中的热读路径 |

除了第11章介绍的互斥锁外，我们在本章使用的还有`cv(9)`和`sx(9)`。`rw(9)`为上下文而提到。`rmlock(9)`、`sema(9)`和`epoch(9)`推迟到问题中的驱动程序实际证明它们合理的后面章节。

### 同一地图中的原子

严格来说，第11章涵盖的`atomic(9)`原语根本不是同步工具包的一部分。它们是*并发操作*：与锁组合但本身不提供阻塞、等待或信号通知的不可分割内存访问。它们以动力工具与手工具并列的方式坐在锁旁边：对特定工作有用，不是工具包其余部分的替代品。

我们只当单字读-修改-写是我们想要表达的正确形状时，才会在本章使用原子。对于其他一切，锁和条件变量值得它们的成本。

### 第一个决策树

当你面对一块共享状态并决定如何保护它时，按这个顺序处理问题。第一个产生明确答案的问题结束搜索。

1. **状态是需要单个读-修改-写的单个字吗？** 使用原子。（示例：代计数器、标志字。）
2. **状态有跨越多个字段的复合不变量，并且访问在进程上下文中吗？** 使用`MTX_DEF`互斥锁。（示例：cbuf头/使用对、队列头/尾对。）
3. **访问在中断上下文中，或者临界区必须禁用抢占吗？** 使用`MTX_SPIN`互斥锁。
4. **状态被多线程频繁读取但很少写入吗？** 对可睡眠调用者使用`sx(9)`（大多数驱动程序代码），或对可能在中断上下文中运行的短临界区使用`rw(9)`。
5. **你需要等待直到特定条件变为真（不仅仅是获取锁）吗？** 使用条件变量（`cv(9)`）与保护条件的互斥锁配对。较旧的`mtx_sleep`/`wakeup`通道机制是遗留替代品；新代码应该更喜欢`cv(9)`。
6. **你需要用墙上时间限制等待吗？** 使用定时变体（`cv_timedwait_sig`、带非零`timo`参数的`mtx_sleep`、`msleep_sbt(9)`）并设计调用者处理`EWOULDBLOCK`。

树压缩成简短口号：*原子用于字，互斥锁用于结构，sx用于读多，cv用于等待，定时用于有界等待，仅在必须时自旋*。

### 一个工作决策：每个`myfirst`状态在哪里

针对当前驱动程序中的每个状态片段遍历树。练习简短有用。

- cbuf索引和后备内存：复合不变量，进程上下文。使用`MTX_DEF`互斥锁。（这是第11章的选择。）
- `sc->bytes_read`、`sc->bytes_written`：高频计数器，很少读取。使用`counter(9)`每CPU计数器。（这是第11章迁移到的。）
- `sc->open_count`、`sc->active_fhs`：低频整数，在cbuf的同一互斥锁下很好。没有理由分开它们。
- `sc->is_attached`：一个标志，在处理程序入口经常读取，每次attach/detach写入一次。第11章设计作为优化在没有互斥锁的情况下读取它，每次睡眠后重新检查，并在互斥锁下写入它。
- 读取等待者阻塞的"缓冲区是否为空？"条件：一个协调等待。当前使用`mtx_sleep(&sc->cb, ...)`。第2节将用命名条件变量替换这个。
- 写入等待者阻塞的"缓冲区是否有空间？"条件：另一个协调等待，当前共享同一通道。第2节将给它自己的条件变量。
- 一个未来的配置子系统（在第5节添加）：每个I/O调用频繁读取，偶尔由sysctl处理程序写入。使用`sx(9)`。

注意树是如何工作的。我们不必为其中任何一个发明自定义设计；我们问了问题，正确的原语就落出来了。

### 现实世界类比：门、走廊和白板

给喜欢它们的读者一个小类比。想象一个研究实验室。

cbuf是一次只能一个人操作的精密仪器。实验室安装了一扇带单把钥匙的门。任何想使用仪器的人必须拿钥匙。当他们有钥匙时，没有其他人可以进入。那就是互斥锁。

实验室有一个状态白板，列出了仪器的当前校准。任何人可以随时阅读白板；他们不会干扰彼此。只有实验室经理更新白板，他们只在等所有人都走开后才这样做。那是共享/独占锁。

实验室有一个咖啡壶。想要咖啡但发现壶是空的人会在公告板上留言："我在休息室；有咖啡时叫醒我。"当有人煮新鲜一壶时，他们检查公告板并轻拍所有留言为"咖啡"的人的肩膀，不管他们多久前写的。那是条件变量。

留咖啡便条的同一人可能还会留第二张便条："但只等十五分钟；如果那时还没有咖啡，我就去食堂了。"那是定时等待。

实验室中的每个机制匹配一个真实的协调问题。它们都不是任何其他机制的替代品。内核中也是如此。

### 并排比较原语

有时在单个表中并排看到原语使选择立即变得明显。它们不同的属性是：它们是阻塞还是自旋、它们是否支持共享（多读者）访问、从持有者角度看它们是否可睡眠、它们是否支持优先级传播、它们是否可被信号中断，以及调用上下文是否可以包括睡眠操作。

| 属性 | `mtx(9) MTX_DEF` | `mtx(9) MTX_SPIN` | `sx(9)` | `rw(9)` | `cv(9)` |
|---|---|---|---|---|---|
| 竞争时行为 | 睡眠 | 自旋 | 睡眠 | 自旋 | 睡眠 |
| 多个持有者 | 否 | 否 | 是（共享） | 是（读） | n/a（等待者） |
| 持有时调用者可睡眠 | 是 | 否 | 是 | 否 | n/a |
| 优先级传播 | 是 | 否（中断禁用） | 否 | 是 | n/a |
| 可信号中断变体 | n/a | n/a | `_sig` | 否 | `_sig` |
| 有定时等待变体 | `mtx_sleep` w/ timo | n/a | n/a | n/a | `cv_timedwait` |
| 适合中断上下文 | 否 | 是 | 否 | 是（小心） | 否 |

两件事脱颖而出。首先，`cv(9)`的列并不真正适合相同的问题，因为cv不是锁；它是等待原语。我们把它包含在比较中，因为选择"我应该等待还是自旋？"本质上与"我应该阻塞在cv上还是自旋在`MTX_SPIN`上？"相同。其次，优先级传播列区分了`mtx(9)`和`rw(9)`与`sx(9)`。`sx(9)`不传播优先级，因为它的睡眠队列不支持它。实际上这只对实时工作负载重要；普通驱动程序不会注意到。

当你面对一块新状态时，用这个表作为快速查找。上面的决策树给你提问的*顺序*；表在你问了问题后给你*答案*。

### 关于信号量的说明

FreeBSD还有一个偶尔有用的计数信号量原语（`sema(9)`）。信号量是一个计数器；线程减少它（通过`sema_wait`或`sema_trywait`）并在计数器为零时阻塞；线程增加它（通过`sema_post`）并可能唤醒等待者。经典用途是有界资源计数：一个有最大长度的队列，生产者在队列满时阻塞，消费者在队列空时阻塞。

大多数看起来像信号量形状的驱动程序问题同样可以用互斥锁加条件变量解决。cv方法的优势是你可以为每个条件附加命名；信号量是匿名的。信号量的优势是等待和信号通知是原语本身的一部分，不需要单独的互锁。

本章不使用`sema(9)`。我们为完整性提到它；如果你在真实驱动程序源代码中遇到它，你现在知道它有什么形状。

### 第1节总结

同步比锁定更广泛，锁定比互斥更广泛，FreeBSD工具包为你可能遇到的每种协调问题形状提供不同的原语。单字更新用原子，复合不变量用互斥锁，读多状态用sx锁，协调等待用条件变量，有界等待用定时睡眠，自旋变体仅在调用上下文要求时使用。

那个决策树将指导我们在本章其余部分做出的每个选择。第2节从第一次重构开始：将第11章的匿名`&sc->cb`唤醒通道转变为一对命名条件变量。

## 第2节：条件变量与睡眠/唤醒

第11章留下的`myfirst`驱动程序有两个不同的条件阻塞I/O路径。读取者在`cbuf_used(&sc->cb) == 0`时睡眠，等待"数据已到达"。写入者在`cbuf_free(&sc->cb) == 0`时睡眠，等待"空间已出现"。两者当前睡眠在同一匿名通道`&sc->cb`上。两次唤醒在每次状态更改后调用`wakeup(&sc->cb)`，无论哪个条件触发了更改，都会唤醒每个睡眠者。

那个安排可行。它也是浪费的、不透明的，比需要更难推理。本节介绍条件变量（`cv(9)`），相同协调等待模式的更干净FreeBSD原语，并演练给每个条件自己的变量的重构。

### 为什么互斥锁加唤醒不够

第11章使用`mtx_sleep(chan, mtx, pri, wmesg, timo)`和`wakeup(chan)`来协调读写路径。这对有很大的简单性优点：任何指针都可以是通道，内核保持每个通道的等待者哈希表，正确通道上的`wakeup`找到他们所有人。

随着驱动程序增长，缺点出现。

**通道是匿名的。** 源代码的读取者看到`mtx_sleep(&sc->cb, &sc->mtx, PCATCH, "myfrd", 0)`，必须从上下文和wmesg字符串推断线程在等待什么条件。`&sc->cb`中没有任何东西说"数据可用"而不是"空间可用"或"设备已分离"。通道只是内核用作哈希键的指针；含义存在于约定中。

**多个条件共享一个通道。** 当`myfirst_write`完成写入时，它调用`wakeup(&sc->cb)`。那唤醒`&sc->cb`上的每个等待者，包括等待数据的读取者（正确）和等待空间的写入者（不正确；写入不释放空间，它消耗空间）。每个不需要的等待者在重新检查其条件后回到睡眠。这是小规模的*惊群问题*：唤醒正确但昂贵。

**如果粗心，丢失唤醒仍然可能。** 如果你曾经在检查条件和进入`mtx_sleep`之间释放互斥锁，唤醒可以在那个窗口中触发并被错过。第11章解释了`mtx_sleep`本身与锁释放是原子的，这关闭了窗口；但规则隐含在API中，在重构时容易违反。

**wmesg参数是唯一的标签。** 对着挂起进程运行`procstat -kk`的调试工程师看到`myfrd`，必须记住那意味着什么。字符串最多七个字符；它是提示，不是结构化描述。

条件变量解决了所有这四个。每个cv有一个名称（它的描述字符串可以从`dtrace`和`procstat`检查）。每个cv正好代表一个逻辑条件；`cv_signal`会影响哪些等待者没有问题。`cv_wait`原语通过将互斥锁作为参数来强制锁释放的原子性，所以误用难得多。等待者和唤醒者之间的关系在类型本身中表达：双方都引用同一个`struct cv`。

### 条件变量是什么

**条件变量**是一个内核对象，代表某些线程正在等待而其他线程最终会信号通知的逻辑条件。条件变量不存储条件；条件存在于你的驱动程序状态中，由你的互斥锁保护。条件变量是会合点：等待者排队的地方，唤醒者找到他们的地方。

数据结构很小，存在于`/usr/src/sys/sys/condvar.h`中：

```c
struct cv {
        const char      *cv_description;
        int              cv_waiters;
};
```

两个字段：一个用于调试的描述字符串，一个当前等待线程计数（当没有人在等待时跳过唤醒机制的优化）。

API也很小：

```c
void  cv_init(struct cv *cvp, const char *desc);
void  cv_destroy(struct cv *cvp);

void  cv_wait(struct cv *cvp, struct mtx *mtx);
int   cv_wait_sig(struct cv *cvp, struct mtx *mtx);
void  cv_wait_unlock(struct cv *cvp, struct mtx *mtx);
int   cv_timedwait(struct cv *cvp, struct mtx *mtx, int timo);
int   cv_timedwait_sig(struct cv *cvp, struct mtx *mtx, int timo);

void  cv_signal(struct cv *cvp);
void  cv_broadcast(struct cv *cvp);
void  cv_broadcastpri(struct cv *cvp, int pri);

const char *cv_wmesg(struct cv *cvp);
```

从开始就有几条规则和约定很重要。

`cv_init`在cv结构存在于内存中后、任何等待者或唤醒者能够到达它之前调用一次。匹配的`cv_destroy`在每个等待者要么已唤醒要么被强制离开队列后、cv结构被释放之前调用一次。这里的生命周期错误会导致与互斥锁生命周期错误相同类型的灾难性崩溃。

`cv_wait`及其变体必须在持有互锁互斥锁的情况下调用。在`cv_wait`内部，内核原子地释放互斥锁并将调用线程放在cv的等待队列上。当线程被唤醒时，互斥锁在`cv_wait`返回之前重新获取。从你的代码角度，互斥锁在调用前后都被持有；不同的线程不可能观察到间隙，即使间隙真的存在。这正是`mtx_sleep`提供的相同原子释放-睡眠契约。

`cv_signal`唤醒一个等待者，`cv_broadcast`唤醒所有等待者。`cv_signal`挑选哪个等待者值得仔细说明。`condvar(9)`手册页只承诺它解除阻塞"一个等待者"；它*不*承诺严格的FIFO顺序，你的代码不能依赖特定的顺序。当前FreeBSD 14.3实现实际做的，在`/usr/src/sys/kern/subr_sleepqueue.c`中的`sleepq_signal(9)`内部，是扫描cv的睡眠队列并挑选优先级最高的线程，以睡眠时间最长的线程打破平局。那是一个有用的心智模型，但把它当作实现细节而不是API保证。如果正确性取决于哪个线程接下来唤醒，你的设计可能是错的，应该使用不同的原语或显式队列。`cv_signal`和`cv_broadcast`通常都在持有互锁互斥锁的情况下调用，尽管规则更多是关于周围逻辑的正确性而不是原语本身：如果你在没有互锁的情况下调用`cv_signal`，新的等待者可能到达并错过信号。因此标准纪律是"持有互斥锁，更改状态，信号通知，释放互斥锁"。

`cv_wait_sig`如果线程被信号唤醒返回非零（通常是`EINTR`或`ERESTART`）；如果被`cv_signal`或`cv_broadcast`唤醒返回零。希望其阻塞I/O路径遵守Ctrl-C的驱动程序使用`cv_wait_sig`，而不是`cv_wait`。第3节深入探索信号处理规则。

`cv_wait_unlock`是罕见的变体，用于调用者希望在等待一侧释放互锁并且返回时不重新获取的情况。在等待完成后调用者与互锁没有进一步事务的拆卸序列中有用。驱动程序很少需要它；我们提到它是因为你会在FreeBSD树的几个地方看到它，本章不再进一步使用它。

`cv_timedwait`和`cv_timedwait_sig`添加以tick为单位的超时。如果超时在任何唤醒到达之前触发，它们返回`EWOULDBLOCK`。第3节解释如何用这些限制阻塞操作。

### 一个工作重构：向myfirst添加两个条件变量

第11章驱动程序对两个条件有一个匿名通道。本章阶段1将其分成两个命名条件变量：`data_cv`（"有数据可读"）和`room_cv`（"有空间可写"）。

向softc添加两个字段：

```c
struct myfirst_softc {
        /* ... 现有字段 ... */
        struct cv               data_cv;
        struct cv               room_cv;
        /* ... 现有字段 ... */
};
```

在attach和detach中初始化和销毁它们：

```c
static int
myfirst_attach(device_t dev)
{
        /* ... 现有设置 ... */
        cv_init(&sc->data_cv, "myfirst data");
        cv_init(&sc->room_cv, "myfirst room");
        /* ... attach的其余部分 ... */
}

static int
myfirst_detach(device_t dev)
{
        /* ... 已清除is_attached并唤醒睡眠者的现有拆卸 ... */
        cv_destroy(&sc->data_cv);
        cv_destroy(&sc->room_cv);
        /* ... detach的其余部分 ... */
}
```

一个小但重要的微妙之处：detach不能销毁仍有等待者的cv。第11章detach路径已经唤醒睡眠者并拒绝在`active_fhs > 0`时继续，这意味着当我们到达`cv_destroy`时，没有描述符打开，没有线程仍然可以在`cv_wait`内部。我们在销毁之前立即添加`cv_broadcast(&sc->data_cv)`和`cv_broadcast(&sc->room_cv)`作为双保险，以防任何后台路径真的潜入。

更新等待助手以使用新变量：

```c
static int
myfirst_wait_data(struct myfirst_softc *sc, int ioflag, ssize_t nbefore,
    struct uio *uio)
{
        int error;

        MYFIRST_ASSERT(sc);
        while (cbuf_used(&sc->cb) == 0) {
                if (uio->uio_resid != nbefore)
                        return (-1);
                if (ioflag & IO_NDELAY)
                        return (EAGAIN);
                error = cv_wait_sig(&sc->data_cv, &sc->mtx);
                if (error != 0)
                        return (error);
                if (!sc->is_attached)
                        return (ENXIO);
        }
        return (0);
}

static int
myfirst_wait_room(struct myfirst_softc *sc, int ioflag, ssize_t nbefore,
    struct uio *uio)
{
        int error;

        MYFIRST_ASSERT(sc);
        while (cbuf_free(&sc->cb) == 0) {
                if (uio->uio_resid != nbefore)
                        return (-1);
                if (ioflag & IO_NDELAY)
                        return (EAGAIN);
                error = cv_wait_sig(&sc->room_cv, &sc->mtx);
                if (error != 0)
                        return (error);
                if (!sc->is_attached)
                        return (ENXIO);
        }
        return (0);
}
```

三件事改变了，没有其他。`mtx_sleep(&sc->cb, &sc->mtx, PCATCH, "myfrd", 0)`变成了`cv_wait_sig(&sc->data_cv, &sc->mtx)`。写路径中的相应行变成了`cv_wait_sig(&sc->room_cv, &sc->mtx)`。wmesg字符串消失了（cv的描述字符串取而代之），通道现在是带名称的真实对象，`PCATCH`标志隐含在`_sig`后缀中。

更新唤醒者。成功读取后，不再唤醒`&sc->cb`上的所有人，只唤醒等待空间的写入者：

```c
got = myfirst_buf_read(sc, bounce, take);
fh->reads += got;
MYFIRST_UNLOCK(sc);

if (got > 0) {
        cv_signal(&sc->room_cv);
        selwakeup(&sc->wsel);
}
```

成功写入后，只唤醒等待数据的读取者：

```c
put = myfirst_buf_write(sc, bounce, want);
fh->writes += put;
MYFIRST_UNLOCK(sc);

if (put > 0) {
        cv_signal(&sc->data_cv);
        selwakeup(&sc->rsel);
}
```

两个改进一起出现。首先，成功读取不再唤醒其他读取者（立即回到睡眠的浪费唤醒）；只有写入者，他们实际上对释放的空间有用途，被唤醒。写端对称。其次，源代码现在是自解释的：`cv_signal(&sc->room_cv)`读作"现在有空间了"；读者不必记住`&sc->cb`意味着什么。

注意我们在信号之前添加了`if (got > 0)`和`if (put > 0)`保护。如果没有改变，唤醒睡眠者没有意义；空信号是良性的但跳过很便宜。这是小优化和澄清：信号在宣布状态更改，保护说明了这一点。

用`cv_signal`代替`cv_broadcast`：我们每次状态更改唤醒一个等待者，而不是所有。状态更改（读取释放一个字节，写入添加一个字节）足够一个等待者取得进展。如果多个等待者被阻塞，下一个信号将唤醒下一个。这是cv API鼓励的每事件对应关系。

### 何时信号与广播

`cv_signal`唤醒一个等待者。`cv_broadcast`唤醒所有。选择比人们预期的更重要。

使用`cv_signal`当：

- 状态更改是每事件更新（一个字节到达；一个描述符释放；一个数据包入队）。一个等待者取得进展就足够；下一个事件将唤醒下一个等待者。
- 所有等待者等价，他们中任何一个可以消耗更改。
- 唤醒无事可做的等待者的成本不可忽略（因为等待者立即重新检查条件并再次睡眠）。

使用`cv_broadcast`当：

- 状态更改是全局的，所有等待者需要知道（设备正在被分离；配置更改；缓冲区被重置）。
- 等待者不等价；每个可能在等待广播解决的稍有不同的子条件。
- 你想避免弄清楚哪些等待者子集可以继续的簿记，代价是唤醒一些会回到睡眠的等待者。

对于`myfirst`数据和空间条件，`cv_signal`是正确的调用。对于detach路径，`cv_broadcast`是正确的调用：detach必须唤醒每个阻塞线程，以便他们可以返回`ENXIO`并干净退出。

向detach添加广播：

```c
MYFIRST_LOCK(sc);
sc->is_attached = 0;
cv_broadcast(&sc->data_cv);
cv_broadcast(&sc->room_cv);
MYFIRST_UNLOCK(sc);
```

那替换了第11章的`wakeup(&sc->cb)`。两次广播唤醒每个可能正在睡眠的读取者和写入者；每个重新检查`is_attached`，看到它现在为零，返回`ENXIO`。

### 一个微妙的陷阱：没有互斥锁的cv_signal

标准纪律说"持有互斥锁，更改状态，信号通知，释放互斥锁"。你可能注意到我们的重构在释放互斥锁*之后*信号（`MYFIRST_UNLOCK(sc)`在`cv_signal`之前）。那是错的吗？

不是，原因值得理解。

纪律要防止的竞争是这样的：等待者检查条件（假），即将调用`cv_wait`，但已经释放了互斥锁。唤醒者现在更改状态，看到cv的等待队列中没有人（因为等待者还没有入队），跳过信号。等待者然后入队并永远睡眠。

`cv_wait`本身通过在释放互斥锁*之前*将等待者入队到cv来防止那个竞争。内核的内部cv队列锁在调用者的互斥锁仍然持有时获取，线程被添加到等待队列，只有那时调用者的互斥锁才被释放，线程被取消调度。该cv上的任何后续`cv_signal`，无论是否持有调用者的互斥锁，都会找到等待者并唤醒它。

因此在互斥锁下信号通知的纪律是防御性约定而不是严格要求。我们在简单情况下遵循它（因为更难出错），我们在互斥锁外信号通知是可测量改进的情况下放宽它（它让被唤醒的线程无需与信号发送者竞争就能获取互斥锁）。对于`myfirst`，在`MYFIRST_UNLOCK(sc)`之后信号通知从唤醒路径上节省了几个周期；为了安全，我们仍然注意不允许状态更改和信号之间存在状态可以被恢复的窗口。在我们的重构中，唯一可以恢复状态的线程也在互斥锁下操作，所以窗口是关闭的。

如果你不确定，在互斥锁下信号通知。这是更安全的默认，成本可以忽略不计。

### 验证重构

构建新驱动程序并在`WITNESS`内核上加载它。运行第11章压力套件。三件事应该发生：

- 所有测试以与之前相同的字节计数语义通过。
- `dmesg`是静默的。没有新警告。
- 对着睡眠读取者的`procstat -kk`现在在等待通道列中显示cv的描述。报告`wmesg`的工具截断到`WMESGLEN`（八个字符，定义在`/usr/src/sys/sys/user.h`中）；因此`"myfirst data"`的描述在`procstat`和`ps`中显示为`"myfirst "`。完整描述字符串对`dtrace`（直接读取`cv_description`）和源代码仍然可见。如果你希望截断形式更有信息量，选择更短的描述如`"mfdata"`和`"mfroom"`；本章保留更长、更可读的名称，因为dtrace和源代码使用完整字符串，那是你花费大部分调试时间的地方。

`lockstat(1)`将显示比旧的`wakeup`机制产生的唤醒更少的cv事件，因为每条件信号通知不会唤醒无关的线程。这是我们预期的吞吐量改进。

### 心智模型：cv_wait如何展开

对于从逐步图景中学习最好的读者，这里是线程调用`cv_wait_sig`然后被信号通知时的事件序列。

时间t=0：线程A在`myfirst_read`中。cbuf为空。

时间t=1：线程A调用`MYFIRST_LOCK(sc)`。互斥锁被获取。线程A现在是受`sc->mtx`保护的任何临界区中的唯一线程。

时间t=2：线程A进入等待助手。检查`cbuf_used(&sc->cb) == 0`为真。线程A调用`cv_wait_sig(&sc->data_cv, &sc->mtx)`。

时间t=3：在`cv_wait_sig`内部，内核为`data_cv`获取cv队列自旋锁，递增`data_cv.cv_waiters`，并原子地做两件事：释放`sc->mtx`并将线程A添加到cv的等待队列。线程A的状态变为"在data_cv上睡眠"。

时间t=4：线程A被取消调度。CPU运行其他线程。

时间t=5：线程B从另一个进程进入`myfirst_write`。线程B调用`MYFIRST_LOCK(sc)`。互斥锁当前空闲；线程B获取它。

时间t=6：线程B从用户空间读取（`uiomove`），将字节提交到cbuf，更新计数器。线程B调用`MYFIRST_UNLOCK(sc)`。

时间t=7：线程B调用`cv_signal(&sc->data_cv)`。内核获取cv队列自旋锁，在等待队列上找到线程A，递减`cv_waiters`，从队列中移除线程A，并将线程A标记为可运行。

时间t=8：调度器决定线程A是最高优先级的可运行线程（或几个之一；相等者优先FIFO）。线程A被调度到CPU上。

时间t=9：线程A在`cv_wait_sig`内恢复。函数重新获取`sc->mtx`（如果另一个线程现在持有互斥锁，这可能本身会阻塞；如果是，线程A被添加到互斥锁的等待列表）。线程A以返回值0（正常唤醒）从`cv_wait_sig`返回。

时间t=10：线程A在等待助手中继续。`while (cbuf_used(&sc->cb) == 0)`检查现在为假（线程B添加了字节）。循环退出。

时间t=11：线程A从cbuf读取并继续。

从图景中得出的三件事。首先，锁状态在每一步都是一致的。互斥锁要么被恰好一个线程持有，要么不被任何人持有；线程A对世界的看法在等待前后是相同的。其次，唤醒与实际调度解耦；线程B没有直接将CPU交给线程A。第三，在t=9和t=10之间存在一个窗口，线程A持有互斥锁，另一个写入者可能（如果它一直在等待）潜在地进一步填充缓冲区。那没问题；线程A的检查是在t=10时的cbuf状态，而不是在t=7时。

这个序列是规范的"等待、信号、唤醒、重新检查、继续"模式。本章中的每个cv使用都是它的一个实例。

### 看看kern_condvar.c

如果你有十分钟，打开`/usr/src/sys/kern/kern_condvar.c`并浏览。三个函数特别值得一看：

`cv_init`（文件顶部）：非常短。它只是初始化描述并将等待者计数归零。

`_cv_wait`（文件中部）：核心阻塞原语。它获取cv队列自旋锁，递增`cv_waiters`，释放调用者的互锁，调用睡眠队列机制来入队线程并让出，返回时递减`cv_waiters`并重新获取互锁。原子释放-睡眠由睡眠队列层执行，正是支持`mtx_sleep`的相同机制。cv在睡眠队列之上没有任何神奇；它是一个薄薄的命名接口。

`cv_signal`和`cv_broadcastpri`：每个都获取cv队列自旋锁，找到一个（或所有）等待者，并使用`sleepq_signal`或`sleepq_broadcast`唤醒他们。

要点：条件变量是`mtx_sleep`使用的相同睡眠队列原语之上的薄结构化层。它们不更慢；它们不更快；它们更清晰。

### 第2节总结

本节的重构给每个等待条件自己的对象、自己的名称、自己的等待者队列和自己的信号。驱动程序对用户空间行为相同，但现在读起来更诚实：`cv_signal(&sc->room_cv)`说"有空间"，这就是我们的意思。`WITNESS`纪律被保留；助手中的`mtx_assert`调用仍然成立；测试套件继续通过。我们已经上移了一级同步词汇表，没有丢失第11章构建的任何安全性。

第3节转向正交问题：*我们应该等多久？*。无限期阻塞对实现方便但对用户苛刻。定时等待和信号感知等待是行为良好的驱动程序响应世界的方式。



## 第3节：处理超时和可中断睡眠

阻塞原语默认是无限期的。一个在缓冲区为空时调用`cv_wait_sig`的读取者将睡眠直到有人在同一个cv上调用`cv_signal`（或`cv_broadcast`），或信号被传递给读取者的进程。从内核的角度看，"无限期"是一个完全可接受的答案。从用户的角度看，"无限期"是挂起。

本节是关于FreeBSD同步原语让你限制等待的两种方式：通过墙上时钟超时和通过信号中断。两者都简单易用，但两者都有关于返回值的惊人微妙的规则。我们从更容易的那个开始并逐步深入。

### 无限期睡眠会出什么问题

三个实际问题推动我们在驱动程序中使用定时和可中断睡眠。

**挂起的程序。** 用户在终端中运行`cat /dev/myfirst`。没有生产者。`cat`阻塞在`read(2)`中，后者阻塞在`myfirst_read`中，后者阻塞在`cv_wait_sig`中。用户按Ctrl-C。如果等待是可中断的（`_sig`变体），内核传递`EINTR`，用户取回他们的shell。如果不是（没有`_sig`的`cv_wait`），内核忽略信号，用户必须从另一个终端使用Ctrl-Z和`kill %1`。大多数用户不知道如何那样做。他们伸手去按重置按钮。

**停滞的进度。** 设备驱动程序等待一个永远不会到达的中断，因为硬件卡住了。驱动程序的I/O线程永远睡眠。整个系统慢慢填满阻塞在这个驱动程序上的进程。最终管理员注意到，但那时除了重启别无他法。有界的等待会更早捕获这个问题。

**糟糕的用户体验。** 网络协议期望在指定时间内有响应。存储操作期望在服务级别协议内有完成。两者都不被可以永远等待的原语很好地服务。驱动程序应该能够强制执行截止日期并在错过截止日期时返回干净的错误。

解决这些的FreeBSD原语是`cv_wait_sig`和`cv_timedwait_sig`，较旧的`mtx_sleep`和`tsleep`家族通过不同的形状提供相同的能力。我们已经在第2节中遇到了`cv_wait_sig`。这里我们更仔细地看它的返回值告诉我们什么以及如何添加显式超时。

### 三种等待结果

任何阻塞睡眠原语都可能因为三个原因之一返回：

1. **正常唤醒。** 其他线程调用了`cv_signal`、`cv_broadcast`或（在遗留API中）`wakeup`。这个线程正在等待的条件已经改变，线程应该重新检查它。
2. **信号被传递给进程。** 线程被要求放弃等待以便信号处理程序可以运行。驱动程序通常向用户空间返回`EINTR`，这也是睡眠原语的返回值。
3. **超时触发。** 线程带截止日期等待，截止日期在任何唤醒到达之前过期。睡眠原语返回`EWOULDBLOCK`。

驱动程序的工作是弄清楚发生了哪三种情况并适当地响应。

第一种情况是容易的。线程重新检查其条件（`cv_wait_sig`周围的`while`循环正是做这个的）；如果条件现在为真，循环结束，I/O继续；如果不是，线程再次睡眠。

第二种情况更有趣。内核不是向*线程*而是向*进程*传递信号。信号可以是严重的条件（`SIGTERM`、`SIGKILL`）或常规的（来自Ctrl-C的`SIGINT`，来自定时器的`SIGALRM`）。正在睡眠的线程需要迅速返回用户空间以便信号处理程序可以运行。约定是睡眠原语返回`EINTR`（中断的系统调用），驱动程序从其处理程序返回`EINTR`，内核要么重启系统调用（如果处理程序以`SA_RESTART`返回），要么向用户空间返回`EINTR`（如果不是）。

第三种情况是有界等待的情况。驱动程序通常将`EWOULDBLOCK`映射到`EAGAIN`（稍后重试）或更具体的错误（`ETIMEDOUT`，在适当的情况下）。

### EINTR、ERESTART和重启问题

在第2种情况中潜伏着一个微妙之处，在章节继续之前值得理解。

当`cv_wait_sig`被信号中断时，实际返回值是以下两件事之一：

- 如果信号的处置是"不重启系统调用"，则为`EINTR`。内核向用户空间返回`EINTR`，`read(2)`报告`-1`且`errno == EINTR`。如果用户程序想要重试，它负责这样做。
- 如果信号的处置是"重启系统调用"（`SA_RESTART`标志），则为`ERESTART`。内核透明地重新进入系统调用，等待再次发生。用户程序看不到中断。

驱动程序不应直接向用户空间返回`ERESTART`；它是系统调用层的内部哨兵。如果驱动程序从其处理程序返回`ERESTART`，系统调用层知道要重启。如果驱动程序返回`EINTR`，系统调用层向用户空间返回`EINTR`。

大多数驱动程序遵循的约定：直接传递`cv_wait_sig`的返回值。如果你得到`EINTR`，驱动程序返回`EINTR`。如果你得到`ERESTART`，驱动程序返回`ERESTART`。内核从那里接管。第11章驱动程序隐式地做了这件事；第2节中的第12章重构继续这样做：

```c
error = cv_wait_sig(&sc->data_cv, &sc->mtx);
if (error != 0)
        return (error);
```

直接返回`error`是正确的举动。第12章没有改变这个规则；它只是使规则在新API中可见。

### 向读取路径添加超时

现在是有界等待的情况。假设我们想让`myfirst_read`可选地等待最多某个可配置的持续时间，如果没有数据到达则返回`EAGAIN`。（我们使用`EAGAIN`而不是`ETIMEDOUT`，因为`EAGAIN`是"操作会阻塞；稍后重试"的常规UNIX答案。）

驱动程序需要三件事：

1. 超时的配置值（比如，以毫秒为单位）。零意味着"像以前一样无限期阻塞"。
2. 将超时转换为tick的方法，因为`cv_timedwait_sig`以其`timo`参数接受tick。
3. 正确处理三种结果的循环：正常唤醒、信号中断、超时。

向softc添加配置字段：

```c
int     read_timeout_ms;  /* 0 = no timeout */
```

在attach中初始化它：

```c
sc->read_timeout_ms = 0;
```

将其公开为sysctl：

```c
SYSCTL_ADD_INT(&sc->sysctl_ctx, SYSCTL_CHILDREN(sc->sysctl_tree),
    OID_AUTO, "read_timeout_ms", CTLFLAG_RW,
    &sc->read_timeout_ms, 0,
    "Read timeout in milliseconds (0 = block indefinitely)");
```

我们现在使用普通的`SYSCTL_ADD_INT`；该值是一个整数，在amd64上字级别的读取是原子的，稍微陈旧的值是可以接受的。（第5节将给我们一种更纪律化的方式来处理配置更改。）

更新等待助手：

```c
static int
myfirst_wait_data(struct myfirst_softc *sc, int ioflag, ssize_t nbefore,
    struct uio *uio)
{
        int error, timo;

        MYFIRST_ASSERT(sc);
        while (cbuf_used(&sc->cb) == 0) {
                if (uio->uio_resid != nbefore)
                        return (-1);
                if (ioflag & IO_NDELAY)
                        return (EAGAIN);

                timo = sc->read_timeout_ms;
                if (timo > 0) {
                        int ticks_total = (timo * hz + 999) / 1000;
                        error = cv_timedwait_sig(&sc->data_cv, &sc->mtx,
                            ticks_total);
                } else {
                        error = cv_wait_sig(&sc->data_cv, &sc->mtx);
                }
                if (error == EWOULDBLOCK)
                        return (EAGAIN);
                if (error != 0)
                        return (error);
                if (!sc->is_attached)
                        return (ENXIO);
        }
        return (0);
}
```

几个细节值得评论。

`(timo * hz + 999) / 1000`算术将毫秒转换为tick，向上取整。我们想要至少请求的等待，绝不少。1000 Hz内核上的1 ms超时变为1 tick。100 Hz内核上的1 ms超时变为1 tick（从0.1向上取整）。5500 ms超时在1000 Hz时变为5500 tick，或在100 Hz时变为550 tick。

`timo > 0`的分支在请求正超时时选择`cv_timedwait_sig`，在没有时选择`cv_wait_sig`（无超时）。我们总是可以用`timo = 0`调用`cv_timedwait_sig`，但cv API将`timo = 0`视为"无限期等待"，行为与`cv_wait_sig`相同。显式分支使意图对读者更清晰。

`EWOULDBLOCK -> EAGAIN`转换给用户空间常规的"重试"指示。得到`EAGAIN`的用户程序知道该做什么；得到`ETIMEDOUT`的用户程序必须学习新的错误代码。

睡眠后的`is_attached`重新检查仍然存在。即使有有界等待，设备可能在睡眠期间被分离；detach中的cv广播唤醒我们；超时本身不跳过睡眠后检查。

如果你想要有界写入，可以对`myfirst_wait_room`应用对称更改，使用单独的`write_timeout_ms` sysctl。配套源代码两者都做了。

### 验证超时

一个小的用户空间测试器确认新行为。将超时设置为100 ms，在没有生产者的情况下打开设备，然后读取。你应该看到`read(2)`在大约100 ms后返回`-1`且`errno == EAGAIN`，而不是永远阻塞。

```c
/* timeout_tester.c: confirm bounded reads. */
#include <err.h>
#include <errno.h>
#include <fcntl.h>
#include <stdio.h>
#include <string.h>
#include <sys/sysctl.h>
#include <sys/time.h>
#include <unistd.h>

#define DEVPATH "/dev/myfirst"

int
main(void)
{
        int timeout_ms = 100;
        size_t sz = sizeof(timeout_ms);

        if (sysctlbyname("dev.myfirst.0.read_timeout_ms",
            NULL, NULL, &timeout_ms, sz) != 0)
                err(1, "sysctlbyname set");

        int fd = open(DEVPATH, O_RDONLY);
        if (fd < 0)
                err(1, "open");

        char buf[1024];
        struct timeval t0, t1;
        gettimeofday(&t0, NULL);
        ssize_t n = read(fd, buf, sizeof(buf));
        gettimeofday(&t1, NULL);
        int saved = errno;

        long elapsed_ms = (t1.tv_sec - t0.tv_sec) * 1000 +
            (t1.tv_usec - t0.tv_usec) / 1000;
        printf("read returned %zd, errno=%d (%s) after %ld ms\n",
            n, saved, strerror(saved), elapsed_ms);

        close(fd);
        return (0);
}
```

在没有连接生产者的情况下运行。预期输出类似于：

```text
read returned -1, errno=35 (Resource temporarily unavailable) after 102 ms
```

FreeBSD上的Errno 35是`EAGAIN`。102 ms是100 ms超时加上几毫秒的调度抖动。

将sysctl重置为零（`sysctl -w dev.myfirst.0.read_timeout_ms=0`）并重新运行。现在`read(2)`阻塞直到你按Ctrl-C，此时它返回`-1`且`errno == EINTR`。可中断性（`_sig`后缀）和超时（`_timedwait`变体）是独立的能力。我们可以两者都没有、有其中之一、或两者都有，API在各自的开关上暴露每一个。

### 在EAGAIN和ETIMEDOUT之间选择

当超时触发时，驱动程序选择报告什么错误。两个合理的选择是`EAGAIN`和`ETIMEDOUT`。

`EAGAIN`（FreeBSD上的errno值35；符号值`EWOULDBLOCK`在`/usr/src/sys/sys/errno.h`中被`#define`为相同的数字）是"操作会阻塞"的常规UNIX答案。处理`O_NONBLOCK`的用户程序理解它。许多用户程序已经在`EAGAIN`上重试。为超时返回`EAGAIN`是一个安全的默认值；它对大多数调用者做正确的事情。

`ETIMEDOUT`（FreeBSD上的errno值60）更具体："操作有截止日期，截止日期已过期"。网络协议使用它；它意味着与"现在会阻塞"不同的东西。想要区分"还没有数据，重试"和"商定的截止日期后没有数据，放弃"的用户程序需要`ETIMEDOUT`。

对于`myfirst`，我们使用`EAGAIN`。驱动程序与调用者没有截止日期约定；超时是一种礼貌，不是保证。其他驱动程序可能做出不同的选择；两者都是合法的。

### 关于超时下公平性的说明

定时等待不改变cv的公平性故事。`cv_timedwait_sig`以`cv_wait_sig`使用的相同睡眠队列实现。当唤醒到达时，睡眠队列选择最高优先级的等待者（相等优先级中FIFO），无论每个等待者是否有待处理的超时。超时是每个等待者的看门狗；它不影响非超时等待者被唤醒的顺序。

实际后果：一个有50 ms超时的线程和一个没有超时的线程，都在同一个cv上等待，都将被`cv_signal`以睡眠队列选择的顺序唤醒。50 ms线程不会获得优先权。如果你需要定时等待者先被唤醒，你有一个不同的设计问题（优先级队列、每个优先级类别的单独cv），这超出了本章的范围。

对于`myfirst`，所有读取者都是等价的，缺乏基于超时的优先级是好的。

### 何时使用超时

超时不是免费的。每个定时等待设置一个callout，如果没有真正的唤醒先到达，它会触发cv的唤醒。callout有小的每tick成本，并增加了内核的整体callout压力。对每个阻塞调用使用超时的驱动程序比无限期阻塞的驱动程序创建更多的callout流量。

三条经验法则：

- 当调用者有真正的截止期限时使用超时（网络协议、硬件看门狗、用户可见的响应）。
- 当等待有一个"这不应该可能"的后备时使用超时。一个"以防万一"在通常在微秒内完成的等待上设置60秒超时的驱动程序是将超时用作健全性检查，不是截止期限。那没问题。
- 当等待自然是无限期时不使用超时（空闲设备上的`cat /dev/myfirst`应该阻塞直到数据到达或用户放弃；两者都可以，都不需要超时）。

本章中的`myfirst`驱动程序公开了一个每设备sysctl，让用户选择。零的默认值（无限期阻塞）对于伪设备是正确的默认值。真实驱动程序可能有更强的意见。

### 使用sbintime_t的子Tick精度

`cv_timedwait`和`cv_timedwait_sig`宏以其超时接受tick。FreeBSD 14.3上的一个tick通常是一毫秒（因为`hz=1000`是默认值），所以tick精度是毫秒精度。对于大多数驱动程序用例这足够了。网络和存储驱动程序偶尔想要微秒精度，`_sbt`（缩放二进制时间）变体是你获得它的方式。

相关的原语：

```c
int  cv_timedwait_sbt(struct cv *cvp, struct mtx *mtx,
         sbintime_t sbt, sbintime_t pr, int flags);
int  cv_timedwait_sig_sbt(struct cv *cvp, struct mtx *mtx,
         sbintime_t sbt, sbintime_t pr, int flags);
int  msleep_sbt(void *chan, struct mtx *mtx, int pri,
         const char *wmesg, sbintime_t sbt, sbintime_t pr, int flags);
```

`sbt`参数是超时，表示为`sbintime_t`（一个64位整数，高32位是秒，低32位是秒的二进制小数部分）。`pr`参数是精度：调度定时器时内核被允许多大的摆动（用于省电的定时器中断合并）。`flags`参数是`C_HARDCLOCK`、`C_ABSOLUTE`、`C_DIRECT_EXEC`等之一，控制定时器如何注册。

对于250微秒超时：

```c
sbintime_t sbt = 250 * SBT_1US;  /* 250 microseconds */
int err = cv_timedwait_sig_sbt(&sc->data_cv, &sc->mtx, sbt,
    SBT_1US, C_HARDCLOCK);
```

`SBT_1US`常量（定义在`/usr/src/sys/sys/time.h`中）是一微秒作为`sbintime_t`。乘以250给出250微秒。精度参数`SBT_1US`说"我对一微秒精度满意"；内核不会将此定时器与其他相隔超过1微秒的定时器合并。

对于5秒：

```c
sbintime_t sbt = 5 * SBT_1S;
int err = cv_timedwait_sig_sbt(&sc->data_cv, &sc->mtx, sbt,
    SBT_1MS, C_HARDCLOCK);
```

五秒等待，毫秒精度。内核可能合并最多1 ms。

对于大多数驱动程序代码，毫秒tick API（带有tick计数的`cv_timedwait_sig`）是正确的精度级别。当你有真正的理由时使用`_sbt`：有亚毫秒定时的网络协议、有微秒级看门狗的硬件控制器、睡眠本身贡献于结果的测量。

### cv_timedwait_sig内部发生了什么

概念上，`cv_timedwait_sig`做与`cv_wait_sig`相同的事情，但还调度一个callout，如果没有真正的信号先到达，它将触发cv的信号。实现位于`/usr/src/sys/kern/kern_condvar.c`中的`_cv_timedwait_sig_sbt`。三个观察值得记住。

首先，callout在持有互锁互斥锁时注册，然后线程睡眠。如果callout在线程睡眠时触发，内核将线程标记为带超时唤醒。线程以`EWOULDBLOCK`从睡眠返回。

其次，如果真正的`cv_signal`在超时之前到达，callout在线程唤醒时被取消。取消原则上是竞争的（callout可能在线程因真正原因唤醒后立即触发），但内核通过检查callout触发时线程是否仍在睡眠来处理这个问题；如果不是，callout是无操作。

第三，每个定时等待创建并拆除一个callout。在有数千个并发定时等待的系统上，callout机制成为可测量的成本。对于最多有几十个等待者的单个驱动程序，成本可以忽略不计。

这些细节不是你需要记忆的东西。然而，它们解释了为什么到处使用定时等待的驱动程序可能比使用单独看门狗线程的无限期等待在callout子系统中显示更多活动。如果你曾经想知道为什么你的驱动程序产生许多callout事件，定时等待是一个可能的原因。

### 第3节总结

有界等待和可中断等待是内核睡眠原语与其外部世界合作的两种方式。我们将两者都添加到了`myfirst`阻塞路径：`cv_wait_sig`已经在那里；`cv_timedwait_sig`是新添加的，由sysctl控制。用户空间测试确认Ctrl-C和100毫秒截止期限都产生预期的返回值；驱动程序分别报告`EINTR`和`EAGAIN`。

第4节转向完全不同的同步形状：共享/独占锁，其中许多线程可以同时读取，只有写入者必须等待轮次。



## 第4节：sx(9)锁：共享和独占访问

`myfirst`今天使用的互斥锁保护具有复合不变量的cbuf。那是该工作的正确原语。然而，并非每块状态都有复合不变量。有些状态被频繁读取、很少写入，并且从不跨越被读取的字段。对于该状态，通过互斥锁序列化每个读取者是浪费的序列化。读者-写者锁更适合那个形状。

本节介绍`sx(9)`，FreeBSD的可睡眠共享/独占锁。我们首先解释共享/独占意味着什么以及为什么它重要，然后演练API，然后简要讨论自旋模式的兄弟`rw(9)`，最后以区分两者并将每个放在正确上下文中的规则结束。

### 共享和独占意味着什么

**共享锁**（也称为*读锁*）同时允许多个持有者。以共享模式持有锁的线程被保证没有其他线程当前以*独占*模式持有它。共享持有者可以并发执行；他们看不到彼此。

**独占锁**（也称为*写锁*）一次恰好被一个线程持有。以独占模式持有锁的线程被保证没有其他线程以任何模式持有它。

锁可以在两个方向之间转换：

- **降级**：独占锁的持有者可以在不释放它的情况下将其转换为共享锁。转换是非阻塞的；紧接着，原始持有者仍然持有锁（现在是共享模式），其他读取者可以继续。
- **升级**：共享锁的持有者可以尝试将其转换为独占锁。如果其他共享持有者仍然存在，尝试可能失败。标准原语是`sx_try_upgrade`，它返回成功/失败而不是阻塞。

升级的不对称性（尝试，可能失败）反映了一个基本困难：如果多个共享持有者同时尝试升级，他们将互相死锁等待。非阻塞的`sx_try_upgrade`让一个成功而其他失败并必须释放并作为独占重新获取。

共享/独占锁是当访问模式是*多读取者，偶尔写入者*时的正确原语。FreeBSD内核中的例子包括sysctl的命名空间锁、内核模块的命名空间锁、文件系统的超级块锁，以及网络驱动程序中的许多配置状态锁。

### 为什么共享/独占在这里胜过普通互斥锁

想象一块驱动程序状态，"当前调试详细级别"，在每个I/O调用的开始读取以决定是否记录某些事件，可能每小时由sysctl更改一次。在第11章互斥锁设计下：

- 每个I/O调用获取互斥锁，读取详细级别，释放互斥锁。
- 每个I/O调用在互斥锁上与每个其他I/O调用的详细级别检查序列化。
- 互斥锁看到巨大的争用，即使没有人在争夺底层*状态*（每个人只是在读取）。

在`sx`设计下：

- 每个I/O调用以共享模式获取锁（在多核系统上便宜；快速路径减少到几次原子操作，没有调度器参与）。
- 多个I/O调用可以并发持有锁。他们不会互相阻塞。
- sysctl写入者偶尔以独占模式获取锁，短暂排除读取者。读取者一旦写入者释放就以共享持有者身份重试。

对于读取频繁的工作负载，差异是戏剧性的。互斥锁的序列化成本随核心数量增长；`sx`的共享模式成本保持恒定。

权衡：`sx_xlock`每次获取比`mtx_lock`更昂贵，因为锁在内部更复杂。对于只读取一次且读取者不争用的状态，`mtx`仍然更好。平衡点取决于工作负载，但经验法则是*当读取者众多且写入者稀有时使用sx；当访问模式对称或写入频繁时使用mtx*。

### sx(9) API

`sx(9)`函数位于`/usr/src/sys/sys/sx.h`和`/usr/src/sys/kern/kern_sx.c`。公共API很小。

```c
void  sx_init(struct sx *sx, const char *description);
void  sx_init_flags(struct sx *sx, const char *description, int opts);
void  sx_destroy(struct sx *sx);

void  sx_xlock(struct sx *sx);
int   sx_xlock_sig(struct sx *sx);
void  sx_xunlock(struct sx *sx);
int   sx_try_xlock(struct sx *sx);

void  sx_slock(struct sx *sx);
int   sx_slock_sig(struct sx *sx);
void  sx_sunlock(struct sx *sx);
int   sx_try_slock(struct sx *sx);

int   sx_try_upgrade(struct sx *sx);
void  sx_downgrade(struct sx *sx);

void  sx_unlock(struct sx *sx);  /* 多态：共享或独占 */
void  sx_assert(struct sx *sx, int what);

int   sx_xlocked(struct sx *sx);
struct thread *sx_xholder(struct sx *sx);
```

`_sig`变体可被信号中断；如果在等待时被信号通知，它们返回`EINTR`或`ERESTART`。非`_sig`变体不可中断地阻塞。跨长时间操作持有sx锁的驱动程序应该考虑`_sig`变体，原因与它们更喜欢`cv_wait_sig`而不是`cv_wait`相同：Ctrl-C应该能够释放等待。

`sx_init_flags`接受的标志包括：

- `SX_DUPOK`：允许同一线程多次获取锁（主要是`WITNESS`指令）。
- `SX_NOWITNESS`：不向`WITNESS`注册锁（很少使用；最好注册并记录任何例外）。
- `SX_RECURSE`：允许同一线程递归获取；只有当每次获取都匹配时锁才被释放。
- `SX_QUIET`、`SX_NOPROFILE`：关闭各种调试仪器。
- `SX_NEW`：声明内存是新的（跳过先前初始化检查）。

对于大多数驱动程序用例，没有标志的`sx_init(sx, "name")`是正确的默认值。

`sx_assert(sx, what)`检查锁状态，如果断言失败则在`INVARIANTS`下panic。`what`参数是以下之一：

- `SA_LOCKED`：锁被调用线程以某种模式持有。
- `SA_SLOCKED`：锁以共享模式被持有。
- `SA_XLOCKED`：锁被调用线程以独占模式持有。
- `SA_UNLOCKED`：锁未被调用线程持有。
- `SA_RECURSED`、`SA_NOTRECURSED`：匹配递归状态。

在期望特定锁状态的助手中自由使用`sx_assert`，就像第11章使用`mtx_assert`一样。

### 一个快速工作示例

假设我们有一个持有驱动程序配置的结构：

```c
struct myfirst_config {
        int     debug_level;
        int     soft_byte_limit;
        char    nickname[32];
};
```

对这些字段的大多数读取发生在数据路径上（每个`myfirst_read`和`myfirst_write`检查`debug_level`）。写入很少发生，来自sysctl处理程序。

向softc添加sx锁：

```c
struct sx               cfg_sx;
struct myfirst_config   cfg;
```

初始化和销毁：

```c
sx_init(&sc->cfg_sx, "myfirst cfg");
/* 在detach中： */
sx_destroy(&sc->cfg_sx);
```

在数据路径上读取：

```c
static bool
myfirst_debug_enabled(struct myfirst_softc *sc, int level)
{
        bool enabled;

        sx_slock(&sc->cfg_sx);
        enabled = (sc->cfg.debug_level >= level);
        sx_sunlock(&sc->cfg_sx);
        return (enabled);
}
```

从sysctl处理程序写入：

```c
static int
myfirst_sysctl_debug_level(SYSCTL_HANDLER_ARGS)
{
        struct myfirst_softc *sc = arg1;
        int new, error;

        sx_slock(&sc->cfg_sx);
        new = sc->cfg.debug_level;
        sx_sunlock(&sc->cfg_sx);

        error = sysctl_handle_int(oidp, &new, 0, req);
        if (error || req->newptr == NULL)
                return (error);

        if (new < 0 || new > 3)
                return (EINVAL);

        sx_xlock(&sc->cfg_sx);
        sc->cfg.debug_level = new;
        sx_xunlock(&sc->cfg_sx);
        return (0);
}
```

写入者中有三件事值得注意。

首先，我们在共享锁下读取*当前*值，以便sysctl框架可以在没有设置新值时填充要显示的值。我们可以不用锁读取它，但这样做会为（诚然很小的）`int`创建撕裂读取的可能性。共享锁便宜且明确。

其次，我们在通过`sysctl_handle_int`验证新值之前释放共享锁、验证范围，然后获取独占锁来提交。我们不能在这个路径中从共享升级到独占，因为`sx_try_upgrade`可能失败；以释放并重新获取的方式做更简单且正确。

第三，验证发生在独占锁之前，这意味着我们持有独占锁的时间最短。独占持有者排除所有读取者；我们要它尽快释放。

### Try-Upgrade和Downgrade

`sx_try_upgrade`是"我有一个共享锁，请给我一个独占锁而不必释放并重新获取"的乐观版本。成功时返回非零（锁现在是独占的），失败时返回零（锁仍然是共享的；另一个线程同时持有它共享，内核无法安全提升）。

模式：

```c
sx_slock(&sc->cfg_sx);
/* 做一些读取 */
if (need_to_modify) {
        if (sx_try_upgrade(&sc->cfg_sx)) {
                /* 现在独占；修改 */
                sx_downgrade(&sc->cfg_sx);
                /* 回到共享；继续读取 */
        } else {
                /* 升级失败；释放并重新获取 */
                sx_sunlock(&sc->cfg_sx);
                sx_xlock(&sc->cfg_sx);
                /* 现在独占但我们之前的视图可能已过时 */
                /* 重新验证并修改 */
                sx_downgrade(&sc->cfg_sx);
        }
}
sx_sunlock(&sc->cfg_sx);
```

`sx_downgrade`总是成功：独占持有者总是可以降级到共享而不阻塞，因为没有其他写入者可以存在（我们持有独占），现有的读取者都是在我们在持有时获取他们的共享锁的（他们不可能），所以他们也不能存在。

对于我们的`myfirst`配置，我们不需要升级/降级：读取和写入是分开的路径，sysctl处理程序愿意释放并重新获取。升级/降级在算法中最有用，其中同一线程读取、决定，然后有条件地修改，所有这些都在一个锁获取-释放周期内。

### 比较sx(9)与rw(9)

`rw(9)`是`sx(9)`的自旋模式兄弟。两者都实现共享/独占想法。它们在如何等待不可用锁方面不同。

`sx(9)`使用睡眠队列。无法立即获取锁的线程被放在睡眠队列上并让出。其他线程在CPU上运行。当锁变得可用时，内核唤醒最高优先级的等待者，然后它重试。

`rw(9)`使用turnstile，内核的基于自旋的支持优先级传播的原语。无法立即获取锁的线程自旋（短暂地），然后交给turnstile机制进行带优先级继承的阻塞。阻塞的方式不像`sx`那么容易放弃CPU。

实际差异：

- `sx(9)`在严格意义上是可睡眠的：持有`sx`锁允许你调用可能睡眠的函数（`uiomove`、`malloc(... M_WAITOK)`）。持有`rw(9)`锁*不*允许；`rw(9)`锁为了睡眠目的被视为自旋锁。
- `sx(9)`支持`_sig`变体用于信号可中断等待。`rw(9)`不支持。
- `sx(9)`通常适用于进程上下文中的代码；`rw(9)`更适用于临界区短且可能在中断上下文中运行的情况（尽管中断上下文的严格选择仍然是`MTX_SPIN`）。

对于`myfirst`，所有配置访问都来自进程上下文，临界区短但包括潜在睡眠的调用，信号中断是有用的功能。`sx(9)`是正确的选择。

一个在硬件中断处理程序内部读取配置的驱动程序将不得不使用`rw(9)`代替，因为`sx_slock`可能睡眠，而在中断中睡眠是非法的。我们在本书后面之前不会遇到这样的驱动程序。

### 可睡眠性规则，重访

第11章介绍了规则"不要跨睡眠操作持有不可睡眠的锁"。有了sx和cv在桌面上，规则需要小的细化。

完整的规则是：*你持有的锁决定临界区中哪些操作是合法的。*

- 持有`MTX_DEF`互斥锁：大多数操作是合法的。睡眠是被允许的（使用`mtx_sleep`、`cv_wait`）。`uiomove`、`copyin`、`copyout`和`malloc(M_WAITOK)`原则上是合法的，但应该避免以保持临界区短。驱动程序约定是在其中任何一个周围释放互斥锁。
- 持有`MTX_SPIN`互斥锁：很少操作是合法的。没有睡眠。没有`uiomove`。没有`malloc(M_WAITOK)`。临界区必须微小。
- 持有`sx(9)`锁（共享或独占）：像`MTX_DEF`。睡眠是被允许的。同样的"如果可以就在睡眠前释放"约定适用，但对睡眠的绝对禁止不存在。
- 持有`rw(9)`锁：像`MTX_SPIN`。没有睡眠。没有长阻塞调用。
- 持有`cv(9)`（即，当前在`cv_wait`内部）：底层互锁互斥锁被`cv_wait`原子释放；从"持有什么"的角度看，你什么都没持有。

这个细化说：`sx`是可睡眠的，`rw`不是。那是它们之间的操作性差异。根据你的临界区需要在哪条线的一侧来选择。

### 锁顺序和sx

`WITNESS`跨所有类别跟踪锁排序：互斥锁、sx锁和rw锁。如果你的驱动程序在持有互斥锁时获取`sx`锁，那建立了一个顺序：互斥锁第一，sx第二。任何路径的反向顺序是违规；`WITNESS`会警告。

对于`myfirst`阶段3（本节），我们将在某些路径中同时持有`sc->mtx`和`sc->cfg_sx`。我们必须显式声明顺序。

自然的顺序是*互斥锁在sx之前*。原因：数据路径为cbuf操作持有`sc->mtx`；如果它需要在临界区期间读取配置值，它会在仍然持有`sc->mtx`时获取`sc->cfg_sx`。反向（`cfg_sx`第一，`mtx`第二）也是可能的（一个想要更新配置并触发事件的sysctl写入者可以获取`cfg_sx`，然后`mtx`），但驱动程序应该选择一个顺序并记录它。

第5节详细说明这个设计并编纂规则。

### 看看kern_sx.c

如果你有几分钟，打开`/usr/src/sys/kern/kern_sx.c`并浏览。`sx_xlock`的快速路径是锁字上的一次比较并交换，与`mtx_lock`的快速路径形状完全相同。慢速路径（在`_sx_xlock_hard`中）将线程交给带优先级传播的睡眠队列。共享锁路径（`_sx_slock_int`）类似，但更新共享持有者计数而不是设置所有者。

对驱动程序编写者重要的是快速路径便宜，慢速路径正确，API与你已经知道的互斥锁API形状相同。如果你能使用`mtx_lock`，你就能使用`sx_xlock`；新词汇是共享模式操作和围绕它们的规则。

### rw(9)简短游览

我们已经多次提到`rw(9)`作为`sx(9)`的自旋模式兄弟。尽管我们的驱动程序不使用它，你会在真实的FreeBSD源代码中遇到它，所以简短的游览值得这几分钟。

API镜像`sx(9)`：

```c
void  rw_init(struct rwlock *rw, const char *name);
void  rw_destroy(struct rwlock *rw);

void  rw_wlock(struct rwlock *rw);
void  rw_wunlock(struct rwlock *rw);
int   rw_try_wlock(struct rwlock *rw);

void  rw_rlock(struct rwlock *rw);
void  rw_runlock(struct rwlock *rw);
int   rw_try_rlock(struct rwlock *rw);

int   rw_try_upgrade(struct rwlock *rw);
void  rw_downgrade(struct rwlock *rw);

void  rw_assert(struct rwlock *rw, int what);
```

与`sx(9)`的差异：

- 模式名称不同：`wlock`（写入/独占）和`rlock`（读取/共享）而不是`xlock`和`slock`。相同的想法，不同的词汇。
- 没有`_sig`变体。`rw(9)`不能被信号中断，因为它在turnstile上实现，不是睡眠队列。
- 持有任何`rw(9)`锁的线程不能睡眠。没有`cv_wait`、没有`mtx_sleep`、没有`uiomove`、没有`malloc(M_WAITOK)`。
- `rw(9)`支持优先级传播。等待被低优先级线程持有的独占锁的线程将提升持有者的优先级。这是`rw(9)`存在而不是仅仅作为`sx(9)`的薄包装的主要原因。

`rw_assert`标志是`RA_LOCKED`、`RA_RLOCKED`、`RA_WLOCKED`，加上与`sx_assert`相同的递归变体。

你会在FreeBSD树中的哪里看到`rw(9)`：

- 网络栈为几个读多表使用`rw(9)`（路由表、地址解析表）。读取访问发生在接收路径中，该路径在网络中断上下文中运行，睡眠是禁止的。
- VFS层为一些命名空间缓存使用它。
- 具有热读取路径和罕见配置更新的各种子系统。

对于我们的`myfirst`驱动程序，每个cfg访问都发生在进程上下文中，每个cfg写入者愿意在`sysctl_handle_*`周围释放锁（它会睡眠），我们受益于信号可中断性。`sx(9)`是正确的选择。如果你曾经需要从中断处理程序访问相同的配置（第14章将讨论这个），答案是切换到`rw(9)`并接受cfg写入者必须在无睡眠情况下完成所有工作的约束。

### 一个使用rw(9)的工作示例

为了使替代方案具体化，这里是cfg路径使用`rw(9)`时的样子。代码在结构上相同，除了API和缺乏信号可中断性：

```c
/* 在softc中： */
struct rwlock           cfg_rw;
struct myfirst_config   cfg;

/* 在attach中： */
rw_init(&sc->cfg_rw, "myfirst cfg");

/* 在detach中： */
rw_destroy(&sc->cfg_rw);

/* 读取路径： */
static int
myfirst_get_debug_level_rw(struct myfirst_softc *sc)
{
        int level;

        rw_rlock(&sc->cfg_rw);
        level = sc->cfg.debug_level;
        rw_runlock(&sc->cfg_rw);
        return (level);
}

/* 写入路径（sysctl处理程序）： */
static int
myfirst_sysctl_debug_level_rw(SYSCTL_HANDLER_ARGS)
{
        struct myfirst_softc *sc = arg1;
        int new, error;

        rw_rlock(&sc->cfg_rw);
        new = sc->cfg.debug_level;
        rw_runlock(&sc->cfg_rw);

        error = sysctl_handle_int(oidp, &new, 0, req);
        if (error || req->newptr == NULL)
                return (error);

        if (new < 0 || new > 3)
                return (EINVAL);

        rw_wlock(&sc->cfg_rw);
        sc->cfg.debug_level = new;
        rw_wunlock(&sc->cfg_rw);
        return (0);
}
```

两件事值得注意。首先，`sysctl_handle_int`在锁*外部*。在`rw(9)`临界区内调用它是非法的，因为`sysctl_handle_int`可能睡眠。这与我们为`sx(9)`版本使用的纪律相同，但对于`rw(9)`它是强制性的而不是仅仅建议性的。其次，读取路径看起来与`sx(9)`版本相同；只有函数名称改变了。那是对称API的重点：心智模型延续。

如果我们的驱动程序某天需要支持配置的中断上下文读取者（也许是一个想知道当前调试级别的硬件中断处理程序），这就是我们要做的更改。目前，`sx(9)`是正确的，我们坚持使用它。

### 第4节总结

`sx(9)`给了我们一种表达"多读取者，偶尔写入者"而不序列化每个读取者的方式。它是可睡眠的、信号感知的，并遵循与互斥锁相同的锁顺序纪律。`rw(9)`是它的不可睡眠兄弟，当临界区可能运行在睡眠非法的上下文中时有用；上面的工作示例显示了小的差异。我们为`myfirst`使用`sx(9)`，因为进程上下文和信号可中断性都是理想的。

第5节将新原语放在一起。我们向`myfirst`添加一个小的配置子系统，决定数据路径和配置路径之间的锁顺序，并根据`WITNESS`验证设计。



## 第5节：实现安全的多读取者、单写入者场景

前面三节孤立地介绍了原语。本节将它们组合成一个连贯的驱动程序设计。我们向`myfirst`添加一个小的配置子系统，给它自己的sx锁，解决与现有数据路径互斥锁的锁顺序，并根据`WITNESS`内核验证结果设计。

配置子系统故意很小。重点不是演示复杂功能；而是演示任何具有多个锁类别的驱动程序必须遵循的锁顺序纪律。

### 配置子系统

我们添加三个可配置参数：

- `debug_level`：0到3的整数。更高的值产生数据路径更详细的`dmesg`输出。
- `soft_byte_limit`：一个整数。如果非零，驱动程序拒绝将cbuf推高到此字节数以上的写入（它提前返回`EAGAIN`）。这是一个穷人版的流控旋钮。
- `nickname`：驱动程序在其日志行中打印的短字符串。用于在`dmesg`中区分多个驱动程序实例。

持有它们的结构：

```c
struct myfirst_config {
        int     debug_level;
        int     soft_byte_limit;
        char    nickname[32];
};
```

将其添加到softc，旁边是它的sx锁：

```c
struct myfirst_softc {
        /* ... 现有字段 ... */
        struct sx               cfg_sx;
        struct myfirst_config   cfg;
        /* ... 其余 ... */
};
```

初始化和销毁：

```c
/* 在attach中： */
sx_init(&sc->cfg_sx, "myfirst cfg");
sc->cfg.debug_level = 0;
sc->cfg.soft_byte_limit = 0;
strlcpy(sc->cfg.nickname, "myfirst", sizeof(sc->cfg.nickname));

/* 在detach中： */
sx_destroy(&sc->cfg_sx);
```

初始值在cdev创建之前设置，所以没有其他线程可以观察到半初始化的配置。

### 锁顺序决定

驱动程序现在有两个可以同时持有的锁类别：`sc->mtx`（cbuf和每softc状态）和`sc->cfg_sx`（配置）。当两者都需要时，我们必须决定哪个先获取。

要问的自然问题：

1. 每个路径最常持有哪个锁？数据路径不断持有`sc->mtx`（每个`myfirst_read`和`myfirst_write`进出它）。数据路径还想要读取`sc->cfg.debug_level`来决定是否记录；那是一个`sx_slock(&sc->cfg_sx)`。所以数据路径已经想要两者，顺序是*互斥锁第一，sx第二*。

2. 哪个路径持有cfg锁并可能想要数据锁？更新配置的sysctl处理程序获取`sx_xlock(&sc->cfg_sx)`。它需要`sc->mtx`吗？原则上，是的：一个重置字节计数器的sysctl处理程序会获取两者。最干净的设计是*不*从sx临界区内获取数据互斥锁；sysctl写入者暂存其工作，释放sx锁，然后如果需要则获取数据互斥锁。那保持顺序单调。

决定：**`sc->mtx`总是在`sc->cfg_sx`之前获取，当两者同时持有时。**

反向顺序被禁止。`WITNESS`将捕获任何违规。

我们在`LOCKING.md`中记录决定：

```markdown
## 锁顺序

sc->mtx -> sc->cfg_sx

持有sc->mtx的线程可以获取sc->cfg_sx（以共享或独占模式）。
持有sc->cfg_sx的线程不能获取sc->mtx。

理由：数据路径总是持有sc->mtx，可能需要在其临界区期间读取配置。
配置路径（sysctl写入者）不需要数据互斥锁；如果未来的功能
需要两者，它必须先获取sc->mtx。
```

### 在数据路径上读取配置

对配置最频繁的访问是数据路径检查`debug_level`来决定是否发出日志消息。我们将其包装在一个小助手中：

```c
static int
myfirst_get_debug_level(struct myfirst_softc *sc)
{
        int level;

        sx_slock(&sc->cfg_sx);
        level = sc->cfg.debug_level;
        sx_sunlock(&sc->cfg_sx);
        return (level);
}
```

注意这个助手只获取`sc->cfg_sx`，不是`sc->mtx`。那是故意的：助手不需要数据互斥锁来读取配置。如果它从已经持有`sc->mtx`的上下文中调用，锁顺序被满足（互斥锁第一，sx第二）。如果它从什么都没持有的上下文中调用，那也没问题。

一个感知调试的日志宏：

```c
#define MYFIRST_DBG(sc, level, fmt, ...) do {                          \
        if (myfirst_get_debug_level(sc) >= (level))                    \
                device_printf((sc)->dev, fmt, ##__VA_ARGS__);          \
} while (0)
```

在数据路径上使用它：

```c
MYFIRST_DBG(sc, 2, "read got %zu bytes\n", got);
```

共享锁获取是每次检查的成本。在多核机器上这是几次原子操作；读取者不互相争用。在单核机器上成本本质上为零（没有其他线程可以在写入的中间）。

### 读取软字节限制

软字节限制的相同模式，被`myfirst_write`用来决定是否拒绝：

```c
static int
myfirst_get_soft_byte_limit(struct myfirst_softc *sc)
{
        int limit;

        sx_slock(&sc->cfg_sx);
        limit = sc->cfg.soft_byte_limit;
        sx_sunlock(&sc->cfg_sx);
        return (limit);
}
```

在`myfirst_write`内部，在实际写入发生之前（注意此时循环中`want`尚未计算），限制检查使用`sizeof(bounce)`作为最坏情况代理：任何单次迭代最多写入一个弹跳缓冲区的字节数，所以当`cbuf_used + sizeof(bounce)`会超过限制时拒绝是一个保守的提前退出：

```c
int limit = myfirst_get_soft_byte_limit(sc);

MYFIRST_LOCK(sc);
if (limit > 0 && cbuf_used(&sc->cb) + sizeof(bounce) > (size_t)limit) {
        MYFIRST_UNLOCK(sc);
        return (uio->uio_resid != nbefore ? 0 : EAGAIN);
}
/* 继续到wait_room和迭代的其余部分 */
```

背靠背两次获取：cfg sx用于限制，mtx用于cbuf检查。注意我们*先*获取sx并在获取互斥锁之前释放它。我们原则上可以同时持有两者（cfg_sx然后mtx），但顺序会错；规则说互斥锁第一，sx第二。所以我们独立获取每一个。两次获取的微小成本是正确性的代价。

一个微妙的点：在cfg_sx释放和mtx获取之间，限制可能改变。那是可以接受的；限制是软提示，不是硬保证。如果sysctl写入者在我们两次获取之间提高了限制，我们仍然拒绝写入，用户会在第二次尝试时重试并成功。如果限制被降低而我们继续了一个新限制会拒绝的写入，没有危害，因为cbuf有自己的硬大小限制。

使用`sizeof(bounce)`而不是实际的`want`反映了另一个微妙的点：在循环的这个阶段，驱动程序还没有计算`want`（那需要知道cbuf当前有多少空间，这需要先持有互斥锁）。使用`sizeof(bounce)`作为最坏情况边界让检查可以在空间计算之前发生。配套源文件严格遵循这个模式。

### 更新配置：sysctl写入者

写入者一侧，作为可以读取和写入`debug_level`的sysctl公开：

```c
static int
myfirst_sysctl_debug_level(SYSCTL_HANDLER_ARGS)
{
        struct myfirst_softc *sc = arg1;
        int new, error;

        sx_slock(&sc->cfg_sx);
        new = sc->cfg.debug_level;
        sx_sunlock(&sc->cfg_sx);

        error = sysctl_handle_int(oidp, &new, 0, req);
        if (error || req->newptr == NULL)
                return (error);

        if (new < 0 || new > 3)
                return (EINVAL);

        sx_xlock(&sc->cfg_sx);
        sc->cfg.debug_level = new;
        sx_xunlock(&sc->cfg_sx);
        return (0);
}
```

遍历锁获取：

1. `sx_slock`读取当前值（以便sysctl框架可以在只读查询时返回它）。
2. 在调用`sysctl_handle_int`之前`sx_sunlock`，因为该函数可能在用户空间之间复制数据（可能睡眠），我们不想跨它持有sx锁。
3. 验证后，`sx_xlock`提交新值。
4. `sx_xunlock`释放。

我们在这个路径中从不持有`sc->mtx`。锁顺序规则平凡满足：这个路径从不同时持有两个锁。

在attach中注册sysctl：

```c
SYSCTL_ADD_PROC(&sc->sysctl_ctx, SYSCTL_CHILDREN(sc->sysctl_tree),
    OID_AUTO, "debug_level",
    CTLTYPE_INT | CTLFLAG_RW | CTLFLAG_MPSAFE,
    sc, 0, myfirst_sysctl_debug_level, "I",
    "Debug verbosity level (0-3)");
```

`CTLFLAG_MPSAFE`标志告诉sysctl框架我们的处理程序在不获取巨锁的情况下安全调用；我们是。这是新sysctl处理程序的现代默认值。

### 更新软字节限制

字节限制的相同形状：

```c
static int
myfirst_sysctl_soft_byte_limit(SYSCTL_HANDLER_ARGS)
{
        struct myfirst_softc *sc = arg1;
        int new, error;

        sx_slock(&sc->cfg_sx);
        new = sc->cfg.soft_byte_limit;
        sx_sunlock(&sc->cfg_sx);

        error = sysctl_handle_int(oidp, &new, 0, req);
        if (error || req->newptr == NULL)
                return (error);

        if (new < 0)
                return (EINVAL);

        sx_xlock(&sc->cfg_sx);
        sc->cfg.soft_byte_limit = new;
        sx_xunlock(&sc->cfg_sx);
        return (0);
}
```

对于昵称（一个字符串，所以sysctl处理程序略有不同）：

```c
static int
myfirst_sysctl_nickname(SYSCTL_HANDLER_ARGS)
{
        struct myfirst_softc *sc = arg1;
        char buf[sizeof(sc->cfg.nickname)];
        int error;

        sx_slock(&sc->cfg_sx);
        strlcpy(buf, sc->cfg.nickname, sizeof(buf));
        sx_sunlock(&sc->cfg_sx);

        error = sysctl_handle_string(oidp, buf, sizeof(buf), req);
        if (error || req->newptr == NULL)
                return (error);

        sx_xlock(&sc->cfg_sx);
        strlcpy(sc->cfg.nickname, buf, sizeof(sc->cfg.nickname));
        sx_xunlock(&sc->cfg_sx);
        return (0);
}
```

结构相同：共享锁读取、释放、通过sysctl框架验证、独占锁提交。字符串版本使用`strlcpy`以保证安全。

### 同时持有两个锁的单个操作

有时路径合法地需要两个锁。例如，假设我们添加一个sysctl重置cbuf并一次清除所有字节计数器。那个sysctl需要：

1. 独占cfg锁，如果它还要重置一些配置（比如，重置调试级别）。
2. 数据互斥锁来操作cbuf。

按照我们的锁顺序，我们先获取`sc->mtx`，然后`sc->cfg_sx`：

```c
static int
myfirst_sysctl_reset(SYSCTL_HANDLER_ARGS)
{
        struct myfirst_softc *sc = arg1;
        int reset = 0;
        int error;

        error = sysctl_handle_int(oidp, &reset, 0, req);
        if (error || req->newptr == NULL || reset != 1)
                return (error);

        MYFIRST_LOCK(sc);
        sx_xlock(&sc->cfg_sx);

        cbuf_reset(&sc->cb);
        sc->cfg.debug_level = 0;
        counter_u64_zero(sc->bytes_read);
        counter_u64_zero(sc->bytes_written);

        sx_xunlock(&sc->cfg_sx);
        MYFIRST_UNLOCK(sc);

        cv_broadcast(&sc->room_cv);  /* 现在有空间可用 */
        return (0);
}
```

获取顺序是`mtx`然后`sx`。释放顺序是相反的：`sx`第一，`mtx`第二。（释放必须反转获取顺序，以维护任何在中间观察锁状态的线程的锁顺序不变量。）

cv广播在两个锁都释放后发生。唤醒睡眠者不需要持有任何一个锁。

`cbuf_reset`是我们添加到cbuf模块的小助手：

```c
void
cbuf_reset(struct cbuf *cb)
{
        cb->cb_head = 0;
        cb->cb_used = 0;
}
```

它将索引归零但不触碰后备内存；在`cb_used`为零的那一刻内容变得无关紧要。

### 根据WITNESS验证

构建新驱动程序并在`WITNESS`内核上加载它。运行第11章压力套件加上一个新的测试器，在I/O发生时猛烈敲击sysctl：

```c
/* config_writer.c: continuously update config sysctls. */
#include <err.h>
#include <stdio.h>
#include <stdlib.h>
#include <sys/sysctl.h>
#include <unistd.h>

int
main(int argc, char **argv)
{
        int seconds = (argc > 1) ? atoi(argv[1]) : 30;
        time_t end = time(NULL) + seconds;
        int v = 0;

        while (time(NULL) < end) {
                v = (v + 1) % 4;
                if (sysctlbyname("dev.myfirst.0.debug_level",
                    NULL, NULL, &v, sizeof(v)) != 0)
                        warn("sysctl debug_level");

                int limit = (v == 0) ? 0 : 4096;
                if (sysctlbyname("dev.myfirst.0.soft_byte_limit",
                    NULL, NULL, &limit, sizeof(limit)) != 0)
                        warn("sysctl soft_byte_limit");

                usleep(10000);  /* 10 ms */
        }
        return (0);
}
```

在一个终端运行`mp_stress`，第二个终端`mt_reader`，第三个终端`config_writer`，全部同时运行。观察`dmesg`看警告。

三件事应该发生：

1. 所有测试以一致的字节计数通过。
2. 调试级别在`dmesg`中可见地改变（当级别高时，数据路径发出日志消息；当低时，它是安静的）。
3. `WITNESS`是静默的。没有报告锁顺序反转。

如果`WITNESS`确实报告了反转，那意味着某处违反了锁顺序。针对规则（互斥锁第一，sx第二）重新阅读受影响的代码路径并修复违规。

### 如果你弄反顺序会出什么问题

为了使规则具体，故意违反它。选择一个持有`sc->mtx`的现有路径，重写它以错误的顺序获取锁。例如，在`myfirst_read`中：

```c
/* 错误：这是我们想让WITNESS捕获的bug。 */
sx_slock(&sc->cfg_sx);   /* sx第一 */
MYFIRST_LOCK(sc);        /* mtx第二；反转全局顺序 */
/* ... */
MYFIRST_UNLOCK(sc);
sx_sunlock(&sc->cfg_sx);
```

构建，在`WITNESS`内核上加载，运行测试套件。`WITNESS`应该在使用此路径和任何其他做互斥锁然后sx的路径的第一次运行时触发：

```text
lock order reversal:
 1st 0xfffffe000a1b2c30 myfirst cfg (myfirst cfg, sx) @ ...:<line>
 2nd 0xfffffe000a1b2c50 myfirst0 (myfirst, sleep mutex) @ ...:<line>
lock order myfirst cfg -> myfirst0 attempted at ...
where myfirst0 -> myfirst cfg is established at ...
```

警告命名了两个锁、它们的地址、每次获取的源位置和先前建立的顺序。修复是将锁放回规范顺序；恢复更改，警告消失。

这就是`WITNESS`在做它的工作。一个有多个锁类别而没有`WITNESS`的驱动程序是一个等待没有人能重现的死锁的驱动程序。

### 一个稍大的模式：快照并应用

当两个锁都需要且操作在每个锁下都有工作要做时，一个常见模式是*快照并应用*。在其最简单的形式中，在实际传输大小已知之前：

```c
/* 阶段1：在sx下快照配置（以共享模式）。 */
sx_slock(&sc->cfg_sx);
int dbg = sc->cfg.debug_level;
int limit = sc->cfg.soft_byte_limit;
sx_sunlock(&sc->cfg_sx);

/* 阶段2：在mtx下做工作，使用快照。 */
MYFIRST_LOCK(sc);
if (limit > 0 && cbuf_used(&sc->cb) + sizeof(bounce) > (size_t)limit) {
        MYFIRST_UNLOCK(sc);
        return (EAGAIN);
}
/* ... cbuf操作 ... */
size_t actual = /* 在临界区内确定 */;
MYFIRST_UNLOCK(sc);

if (dbg >= 2)
        device_printf(sc->dev, "wrote %zu bytes\n", actual);
```

快照并应用模式保持每个锁持有时间最短，避免同时持有两个锁，并产生清晰的二阶段形状，易于推理。代价是快照在应用执行时可能稍微过时；实际上，过时是微秒级的，对几乎任何配置值都是可接受的。

如果过时对某个特定值不可接受（比如，关键安全标志），那么持有者必须原子地获取锁并遵循全局顺序。快照模式是默认的，不是定律。

### myfirst_sysctl_reset演练

重置sysctl是本章中唯一合法同时持有两个锁的路径。它值得仔细追踪，因为模式（`mtx`然后`sx`，两者以相反顺序释放，在两者释放后广播）是你每次必须持有两个锁时要模仿的模式。

当用户运行`sysctl -w dev.myfirst.0.reset=1`时，内核以`req->newptr`非空调用`myfirst_sysctl_reset`。处理程序：

1. 通过`sysctl_handle_int`读取新值。如果值不是1，不做任何事返回（只将`1`视为确认的重置请求）。
2. 获取`MYFIRST_LOCK(sc)`。数据互斥锁现在被持有。
3. 获取`sx_xlock(&sc->cfg_sx)`。cfg sx现在以独占模式被持有。两个锁都持有；锁顺序满足（互斥锁第一，sx第二）。
4. 调用`cbuf_reset(&sc->cb)`。缓冲区现在为空（`cb_used = 0`，`cb_head = 0`）。
5. 设置`sc->cfg.debug_level = 0`。配置现在处于其初始状态。
6. 对每个每CPU计数器调用`counter_u64_zero`。字节计数器现在为零。
7. 调用`sx_xunlock(&sc->cfg_sx)`。cfg sx被释放。现在只持有mtx。
8. 调用`MYFIRST_UNLOCK(sc)`。mtx被释放。没有持有锁。
9. 调用`cv_broadcast(&sc->room_cv)`。任何因满而阻塞的写入者被唤醒；他们将重新检查`cb_free`，发现它等于`cb_size`，然后继续。
10. 返回0。

关于此序列的三个观察。

锁获取遵循全局顺序；`WITNESS`满意。释放遵循相反顺序，这维护了任何观察序列的线程看到一致状态的不变量。

广播在两个锁都释放*之后*发生。在广播时持有任何一个锁会不必要地阻塞被唤醒的线程，当它们尝试获取它们需要的东西时；广播是即发即弃的，内核处理其余部分。

数据路径的重置（`cbuf_reset`）和配置的重置（`debug_level = 0`）相对于彼此是原子的。一个在重置后观察任何字段的线程看到每个字段的重置后值；没有线程可以观察到半重置状态。

如果我们想要添加更多重置操作（清除状态机、重置硬件寄存器），每个都适合这个模板。锁持有范围扩展以覆盖新字段；末尾的广播通知适当的cv。

### 第5节总结

驱动程序现在有两个锁类别：数据路径的互斥锁和配置的sx锁。顺序被记录（互斥锁第一，sx第二），顺序在运行时由`WITNESS`强制执行，我们使用的模式（常见情况的快照并应用，只需要一个锁的路径的单次获取，以及真正需要两者的路径的仔细排序的获取-释放）保持设计可审计。新sysctl让用户空间可以在运行时调整驱动程序的行为而不中断数据路径。

第5节更大的教训是添加第二个锁类别是一个真正的设计决定，不是一个透明的优化。成本是维护锁顺序所需的纪律、文档工作以及运行时验证两者的审计工作。好处是数据路径不再是状态唯一存在的地方；配置有自己的保护、自己的节奏和自己的sysctl接口。随着驱动程序增长，它们通常最终有三或四个锁类别，每个有自己的目的。本节的模式可以扩展。

第6节转向当你开始混合锁类别时总是出现的下一个问题：当同步bug出现时如何调试它？内核为你提供了几种工具来完成这项工作；本章就是介绍它们的。


## 第6节：调试同步问题

第11章介绍了基本的内核调试钩子：`INVARIANTS`、`WITNESS`、`KASSERT`和`mtx_assert`。我们用它们来验证互斥锁在正确的时机被持有，以及锁顺序规则被遵守。随着第12章将更广泛的同步工具包放到你手中，调试工具也扩展了。本节将演练你将最常使用的模式和工具：仔细阅读`WITNESS`警告、使用内核内调试器检查挂起的系统，以及识别条件变量和共享/独占锁最常见的故障模式。

### 同步Bug目录

六种故障形状涵盖了你在实践中将遇到的大多数情况。识别形状是诊断的一半。

**丢失唤醒。** 线程进入`cv_wait_sig`（或`mtx_sleep`）；它等待的条件变为真但从未调用`cv_signal`（或`wakeup`）。线程永远睡眠。原因：忘记在状态更改后调用`cv_signal`；信号通知了错误的cv；在状态实际更改之前就信号通知了；状态更改发生在不包含信号的路径中。

**虚假唤醒处理不当。** 线程被信号或其他瞬态事件唤醒，但它等待的条件仍然为假。如果周围代码没有循环并重新检查，线程在假设条件为真的情况下继续操作陈旧的状态。治疗：总是在`cv_wait`调用周围使用`while (!condition) cv_wait(...)`，绝不用`if (!condition) cv_wait(...)`。

**锁顺序反转。** 两个锁被两个路径以相反顺序获取。在争用下要么死锁，要么`WITNESS`捕获。治疗：在`LOCKING.md`中定义全局顺序并在各处遵循。

**过早销毁。** cv或sx在线程仍在等待它时被销毁。症状不可预测：panic、释放后使用、陈旧指针崩溃。治疗：确保每个等待者已被唤醒（并实际上从等待原语返回）后再调用`cv_destroy`或`sx_destroy`。detach路径在这里必须小心。

**持有不可睡眠锁时睡眠。** 持有`MTX_SPIN`互斥锁或`rw(9)`锁然后调用`cv_wait`、`mtx_sleep`、`uiomove`或`malloc(M_WAITOK)`。`WITNESS`捕获它；在非`WITNESS`内核上系统要么死锁要么panic。治疗：在睡眠操作前释放自旋/rw锁，或改为持有`MTX_DEF`互斥锁/`sx(9)`锁，两者都允许睡眠。

**detach与活动操作之间的竞争。** 一个描述符打开，线程在`cv_wait_sig`中，设备被告知要分离。detach路径必须唤醒睡眠者，必须等到睡眠者返回，必须保持cv（和互斥锁）活跃直到那发生。治疗：标准detach模式是设置"正在离开"标志，广播所有cv，等待`active_fhs`降为零，然后销毁原语。

一个存活足够久进入生产的驱动程序已经处理过这些中的每一个至少一次。本章的实验让你故意引发和解决其中几个，使模式变得熟悉。

### 仔细阅读WITNESS警告

`WITNESS`警告有三个有用的部分：警告文本、每次锁获取的源位置和先前建立的顺序。将它们分开。

锁顺序反转的典型警告：

```text
lock order reversal:
 1st 0xfffffe000a1b2c30 myfirst cfg (myfirst cfg, sx) @ /var/.../myfirst.c:120
 2nd 0xfffffe000a1b2c50 myfirst0 (myfirst, sleep mutex) @ /var/.../myfirst.c:240
lock order myfirst cfg -> myfirst0 attempted at /var/.../myfirst.c:241
where myfirst0 -> myfirst cfg is established at /var/.../myfirst.c:280
```

从上到下阅读：

- **`lock order reversal:`**：警告类别。其他类别包括`acquiring duplicate lock of same type`、`sleeping thread (pid N) owns a non-sleepable lock`、`WITNESS exceeded the recursion limit`。
- **`1st 0x... myfirst cfg`**：违规路径中第一个获取的锁。地址（`0x...`）、名称（`myfirst cfg`）、类型（`sx`）和源位置（`myfirst.c:120`）准确地告诉你哪个锁和在哪里。
- **`2nd 0x... myfirst0`**：第二个获取的锁。相同的字段集。
- **`lock order myfirst cfg -> myfirst0 attempted at ...`**：这段代码尝试使用的顺序。
- **`where myfirst0 -> myfirst cfg is established at ...`**：`WITNESS`先前观察并记录为规范的顺序，以及规范示例的源位置。

修复是以下两件事之一。要么新路径错了，应该遵循规范顺序（最常见），要么规范顺序本身错了，需要改变。选择哪个是判断调用；通常答案是修复新路径以匹配规范，因为规范可能是对的。

有时警告是误报：同一锁类型的两个不同对象，它们之间的顺序无关紧要，因为它们是独立的。`WITNESS`不总是知道这一点；锁初始化时的`LOR_DUPOK`标志告诉它跳过检查。我们不需要为`myfirst`这样做，但有每实例锁的真实驱动程序有时需要。

### 使用DDB检查挂起的系统

如果测试挂起且`dmesg`是静默的，罪魁祸首通常是错过唤醒或死锁。内核内调试器（DDB）让你在挂起时刻检查每个线程和每个锁的状态。

要在挂起系统上进入DDB，在控制台上按`Break`键（如果你在使用`cu`的串行控制台上，发送`~b`转义）。DDB用`db>`提示你。

同步调试最有用的命令：

- `show locks`：列出当前线程持有的锁（DDB进来的那个，通常是内核空闲线程；本身很少有用）。
- `show all locks`（别名`show alllocks`）：列出系统上每个线程持有的每个锁。这是你大多数时候想要的命令。
- `show witness`：转储整个`WITNESS`锁顺序图。详细但权威。
- `show sleepchain <thread>`：追踪特定线程涉及的锁和等待链。当你怀疑死锁循环时有用。
- `show lockchain <lock>`：从锁追踪到持有它的线程以及该线程持有的任何其他锁。
- `ps`：列出所有进程和线程及其状态。
- `bt <thread>`：特定线程的反向追踪。
- `continue`：离开DDB并恢复系统。仅在你没有做任何更改时使用。
- `panic`：强制panic使系统干净重启。如果`continue`不安全时使用。

挂起测试的工作流程：

1. 通过控制台中断进入DDB。
2. `show all locks`。注意任何持有锁的线程。
3. `ps`找到测试线程（在等待通道列中寻找`myfrd`、`myfwr`或`cv_w`）。
4. 对于每个有趣的线程，`bt <pid> <tid>`获取反向追踪。
5. 如果怀疑死锁，对每个等待线程`show sleepchain`。
6. 一旦你有足够的信息，`panic`重启。

DDB转录成为你的调试日志。保存它（DDB可以输出到控制台和内核消息缓冲区；在有`EARLY_AP_STARTUP`的调试内核上，你可以将输出重定向到串口）。

### 识别丢失唤醒

丢失唤醒是最常见的cv bug。症状是永远存在的挂起等待者；线程在`cv_wait_sig`中，没有东西唤醒它。

在DDB中检测：

```text
db> ps
... (找到挂起的线程，例如状态为"*myfirst data")
db> bt 1234 1235
... (反向追踪显示线程在cv_wait_sig内部)
db> show locks
... (线程没有持有锁；它在睡眠)
```

有问题的cv通过等待通道名称识别。如果通道名称是`myfirst data`，cv是`sc->data_cv`。现在问：谁应该调用`cv_signal(&sc->data_cv)`？搜索源代码：

```sh
$ grep -n 'cv_signal(&sc->data_cv\|cv_broadcast(&sc->data_cv' myfirst.c
```

对于每个调用点，弄清楚它是否应该触发以及它是否实际触发了。常见罪魁祸首：

- 信号在一个永远不为真的`if`内部。
- 信号在一个绕过它的`return`之后。
- 信号目标错误的cv（`cv_signal(&sc->room_cv)`而不是`data_cv`）。
- 状态更改没有实际改变条件（你递增了`cb_used`但消费者在检查`cb_free`，这是相同的逻辑状态但`cb_free == cb_size - cb_used`；那个没问题，但类似的错误计算可能隐藏问题）。

修复源代码，重建，重测。症状应该消失。

### 识别虚假唤醒

虚假唤醒是在条件仍然为假时到达的唤醒。原因包括信号（`cv_wait_sig`由于信号返回即使条件没有改变）和超时（`cv_timedwait_sig`由于定时器返回）。两者都是正常的；驱动程序必须处理它们。

检测：bug不是唤醒本身，而是处理它的失败。形状：

```c
/* 错误： */
if (cbuf_used(&sc->cb) == 0)
        cv_wait_sig(&sc->data_cv, &sc->mtx);
/* 现在读取cbuf假设有数据，但虚假唤醒可能
   在缓冲区仍然为空时把我们带到这里 */
got = cbuf_read(&sc->cb, bounce, take);
```

`cbuf_read`在这种情况下会返回零，传播到用户空间为零字节读取，`cat`和其他工具将其解释为EOF。用户看到一个不是真正文件结束的静默文件结束。

治疗：总是循环：

```c
/* 正确： */
while (cbuf_used(&sc->cb) == 0) {
        int error = cv_wait_sig(&sc->data_cv, &sc->mtx);
        if (error != 0)
                return (error);
        if (!sc->is_attached)
                return (ENXIO);
}
got = cbuf_read(&sc->cb, bounce, take);
```

第2节的`myfirst_wait_data`助手已经遵循这个模式。一般规则是：*永远不要在`cv_wait`周围使用`if`；总是使用`while`。*

### 识别锁顺序反转

我们之前看到了`WITNESS`警告。在非`WITNESS`内核上的替代是死锁。两个线程各持有一个锁并想要另一个；都无法继续。

在DDB中检测：

```text
db> show all locks
Process 1234 (test1) thread 0xfffffe...
shared sx myfirst cfg (myfirst cfg) ... locked @ ...:120
shared sx myfirst cfg (myfirst cfg) r = 1 ... locked @ ...:120

Process 5678 (test2) thread 0xfffffe...
exclusive sleep mutex myfirst0 (myfirst) r = 0 ... locked @ ...:240
```

然后对每个等待线程`show sleepchain`：

```text
db> show sleepchain 1234
Thread 1234 (pid X) blocked on lock myfirst0 owned by thread 5678
db> show sleepchain 5678
Thread 5678 (pid Y) blocked on lock myfirst cfg owned by thread 1234
```

那个循环就是死锁。每个线程都被对方持有的锁阻塞。修复是重新审视锁顺序；两个路径中有一个以错误的顺序获取。修复与`WITNESS`警告的相同：找到违规的获取并重新排序。

### 持有不可睡眠锁时睡眠

如果你的驱动程序使用`MTX_SPIN`互斥锁或`rw(9)`锁，并且在持有其中一个时某处调用`cv_wait`、`mtx_sleep`、`uiomove`或`malloc(M_WAITOK)`，`WITNESS`将触发：

```text
sleeping thread (pid 1234, tid 5678) owns a non-sleepable lock:
exclusive rw lock myfirst rw (myfirst rw) r = 0 ... locked @ ...:100
```

警告命名了锁和获取的位置。修复是在睡眠操作前释放锁。`myfirst`不使用`MTX_SPIN`或`rw(9)`，所以我们不会直接遇到这个问题；如果你在另一个驱动程序中重用这些模式，注意它。

### 过早销毁

cv或sx在线程仍在等待时被销毁会导致释放后使用风格的崩溃。症状通常是在`cv_destroy`或`sx_destroy`已经运行后在`cv_wait`或`sx_xlock`内部的panic，带有反向追踪。

第11章detach模式（在`active_fhs > 0`时拒绝detach）为大多数情况防止了这个问题。第12章驱动程序用`cv_broadcast`调用扩展了模式，在销毁之前：

```c
MYFIRST_LOCK(sc);
sc->is_attached = 0;
cv_broadcast(&sc->data_cv);
cv_broadcast(&sc->room_cv);
MYFIRST_UNLOCK(sc);

/* 现在等待任何正在睡眠的线程返回。
   上面的cv_broadcast唤醒它们；它们看到is_attached
   为假并返回ENXIO。它们在退出时释放互斥锁。 */

/* 一旦我们知道没有人在I/O路径中，销毁原语。
   按构造，当我们到达这里时active_fhs == 0，
   所以没有线程可以重新进入。 */
cv_destroy(&sc->data_cv);
cv_destroy(&sc->room_cv);
sx_destroy(&sc->cfg_sx);
mtx_destroy(&sc->mtx);
```

顺序很重要。互斥锁是cv的互锁（`cv_wait_sig`内部的线程在`sc->mtx`被释放给内核的情况下睡眠；在唤醒时，`cv_wait_sig`在返回之前重新获取`sc->mtx`）。如果我们先销毁`sc->mtx`然后`cv_destroy`，一个尚未完全唤醒的线程可能卡在内核内部试图重新获取一个我们刚刚拆除内存的互斥锁。先销毁cv保证到互斥销消失时没有线程仍然在`cv_wait_sig`内部。相同的推理适用于sx：阻塞在`sx_xlock`内部的线程不持有互斥锁，但其唤醒后重新获取路径如果sx和互斥锁被同时拆除可能会被顺序绊倒。以线程可能仍然持有或等待每个原语的相反顺序销毁：cv先（等待者排空），然后sx（没有读取者或写入者剩下），然后互斥锁（没有互锁伙伴剩下）。

### 有用的断言添加

在助手中散布`sx_assert`和`mtx_assert`调用来记录预期的锁状态。每个在生产时是免费的（`INVARIANTS`被编译出去），在调试内核上捕获新bug。

示例：

```c
static int
myfirst_get_debug_level(struct myfirst_softc *sc)
{
        int level;

        sx_slock(&sc->cfg_sx);
        sx_assert(&sc->cfg_sx, SA_SLOCKED);  /* 记录锁状态 */
        level = sc->cfg.debug_level;
        sx_sunlock(&sc->cfg_sx);
        return (level);
}
```

`sx_assert(&sc->cfg_sx, SA_SLOCKED)`在`sx_slock`之后技术上是冗余的（锁刚刚被获取），但它使意图对读者明显，并捕获重构错误（有人移动函数并忘记锁）。

一个更有用的模式：在*期望*调用者已获取锁的助手中的断言：

```c
static void
myfirst_apply_debug_level(struct myfirst_softc *sc, int level)
{
        sx_assert(&sc->cfg_sx, SA_XLOCKED);  /* 调用者必须持有xlock */
        sc->cfg.debug_level = level;
}
```

如果未来的调用站点在没有锁的情况下尝试使用这个助手，断言触发。函数的期望现在是可执行的，不只是文档化的。

### 使用dtrace和lockstat追踪锁活动

两个用户空间工具让你在不修改驱动程序的情况下观察锁行为。

`lockstat(1)`总结一段时间内的锁争用：

```sh
# lockstat -P sleep 10
```

这运行10秒并打印一个表，列出每个被争用的锁，带有持有时间和等待时间。对于在`mp_stress`下的`myfirst`，你应该看到`myfirst0`（设备互斥锁）和`myfirst cfg`（sx）在列表顶部。两者是否有值得担心的争用取决于工作负载；对于我们的伪设备，两者都不应该有。

`dtrace(1)`让你追踪特定事件。要看data cv上每次cv_signal：

```sh
# dtrace -n 'fbt::cv_signal:entry /args[0]->cv_description == "myfirst data"/ { stack(); }'
```

这每次信号被发送时打印内核堆栈追踪。对于精确定位哪个路径正在信号通知以及从哪里有用。

两个工具都有最小的开销，可以在生产内核以及调试内核上使用。

### 工作演练：丢失唤醒Bug

为了使诊断工作流程具体，演练一个假设的丢失唤醒bug，从症状到修复。

你对驱动程序做了一个微妙破坏`myfirst_write`中信号配对的更改。更改后，测试套件大部分通过，但`mt_reader`偶尔挂起。你可以通过运行`mt_reader`十到二十次来重现；一两次程序挂起在它的一个线程上。

**步骤1：用`procstat`确认症状。**

```sh
$ ps -ax | grep mt_reader
12345 ?? S+    mt_reader

$ procstat -kk 12345
  PID    TID COMM             TDNAME           KSTACK
12345 67890 mt_reader        -                mi_switch+0xc1 _cv_wait_sig+0xff
                                              myfirst_wait_data+0x4e
                                              myfirst_read+0x91
                                              dofileread+0x82
                                              sys_read+0xb5
```

线程在`_cv_wait_sig`中。查找它的等待通道：

```sh
$ ps -axHo pid,tid,wchan,command | grep mt_reader
12345 67890 myfirst         mt_reader
12345 67891 -               mt_reader
```

一个线程阻塞在`myfirst `上（`"myfirst data"`的八字符截断形式；见第2节关于`WMESGLEN`的说明）。其他已退出。所以cv `data_cv`有一个等待者，大概驱动程序在应该信号通知它时没有。

**步骤2：检查cbuf状态。**

```sh
$ sysctl dev.myfirst.0.stats.cb_used
dev.myfirst.0.stats.cb_used: 17
```

缓冲区有17个字节。读取者应该能够排空那些字节并返回它们。为什么读取者仍然在睡觉？

**步骤3：检查源代码。** 读取者在`cv_wait_sig(&sc->data_cv, &sc->mtx)`中。谁调用`cv_signal(&sc->data_cv)`？搜索：

```sh
$ grep -n 'cv_signal(&sc->data_cv\|cv_broadcast(&sc->data_cv' myfirst.c
180:        cv_signal(&sc->data_cv);
220:        cv_broadcast(&sc->data_cv);  /* 在detach中 */
```

两个调用者。相关的是第180行，在`myfirst_write`中。看它：

```c
put = myfirst_buf_write(sc, bounce, want);
fh->writes += put;
MYFIRST_UNLOCK(sc);

if (put > 0) {
        cv_signal(&sc->data_cv);
        selwakeup(&sc->rsel);
}
```

信号以`put > 0`为条件。它看起来正确。但之前引入的bug可能改变了其他东西。再往上看：

```c
MYFIRST_LOCK(sc);
error = myfirst_wait_room(sc, ioflag, nbefore, uio);
if (error != 0) {
        MYFIRST_UNLOCK(sc);
        return (error == -1 ? 0 : error);
}
room = cbuf_free(&sc->cb);
MYFIRST_UNLOCK(sc);

want = MIN((size_t)uio->uio_resid, sizeof(bounce));
want = MIN(want, room);
error = uiomove(bounce, want, uio);
if (error != 0)
        return (error);

MYFIRST_LOCK(sc);
put = myfirst_buf_write(sc, bounce, want);
```

这是bug。在`uiomove`之后，代码直接跳回cbuf写入。但如果`want`是针对陈旧`room`计算的呢？假设在`cbuf_free`调用和第二次`MYFIRST_LOCK`之间，另一个写入者添加了字节。第二次`myfirst_buf_write`可能以超过实际当前空间的`want`被调用。

在我们的情况中，`myfirst_buf_write`返回实际写入的字节数，可能少于`want`。我们正确更新`bytes_written`。但我们然后只在`put > 0`时信号`data_cv`。到目前为止还好。

但等等。仔细看有bug的行：想象引入的更改是将信号包装在不同的条件中：

```c
if (put == want) {  /* 错误：原来是 put > 0 */
        cv_signal(&sc->data_cv);
        selwakeup(&sc->rsel);
}
```

现在如果`put < want`（因为另一个写入者抢占了空间），我们不信号。字节被添加到了cbuf，但读取者没有被唤醒。当前在`cv_wait_sig`中的读取者将睡眠直到有人写入一个完整的缓冲区。

那就是丢失唤醒bug。修复是在`put > 0`时信号，不是在`put == want`时。应用修复，重建，重测。挂起消失。

**步骤4：防止回归。** 在唤醒站点添加一个记录契约的`KASSERT`：

```c
KASSERT(put <= want, ("myfirst_buf_write returned %zu > want=%zu",
    put, want));
if (put > 0) {
        cv_signal(&sc->data_cv);
        selwakeup(&sc->rsel);
}
```

KASSERT不捕获我们刚修复的bug（它由`put != want`触发，这是被允许的）。但它记录信号条件是"任何前进"，这是下一个维护者应该保留的规则。

这个演练是人为的；真实的bug更混乱。模式是真实的。症状 -> 仪器 -> 源代码检查 -> 假设 -> 修复 -> 回归防护。在本章的实验中练习它。

### 工作演练：锁顺序反转

另一个常见场景：`WITNESS`报告了一个你不立即理解的反转。

警告，简化：

```text
lock order reversal:
 1st 0xfffffe000a1b2c30 myfirst cfg (myfirst cfg, sx) @ myfirst.c:120
 2nd 0xfffffe000a1b2c50 myfirst0 (myfirst, sleep mutex) @ myfirst.c:240
lock order myfirst cfg -> myfirst0 attempted at myfirst.c:241
where myfirst0 -> myfirst cfg is established at myfirst.c:280
```

**步骤1：识别锁。** `myfirst cfg`是sx；`myfirst0`是设备互斥锁。

**步骤2：识别规范顺序。** 从警告：`myfirst0 -> myfirst cfg`。所以互斥锁第一，然后sx。

**步骤3：识别违规路径。** 警告说违规路径在`myfirst.c:241`，在那里它尝试在已经持有`myfirst cfg`时获取`myfirst0`。在源代码第241行打开并向上追踪找到cfg sx何时被获取（第120行，由警告的`1st`字段给出）。

**步骤4：决定修复。** 两个选项。要么重新排序违规路径以匹配规范顺序（先获取mtx，然后sx；这通常是更便宜的更改），要么接受违规路径有真正理由以相反顺序获取，在这种情况下规范顺序需要全局改变并更新`LOCKING.md`。

对于我们的`myfirst`，违规路径几乎肯定应该匹配规范。修复是通过快照并应用模式读取cfg值：在获取mtx之前释放sx。

**步骤5：验证。** 应用修复，重建，重跑触发警告的测试。`WITNESS`现在应该是静默的。如果不是，警告已转移到不同的路径，你有第二个违规要调查。

### WITNESS无法检测的常见错误

`WITNESS`在它检查的内容上是出色的，但不检查所有东西。它无法检测的三类bug：

**跨函数指针持有的锁。** 如果一个函数持有锁A并调用由用户控制配置提供的函数指针回调，`WITNESS`无法预测回调可能获取什么锁。相对于回调的锁顺序是未定义的。避免这个模式；如果必须使用它，记录任何回调可接受的锁状态。

**无锁字段上的竞争条件。** 故意不加锁访问的字段对`WITNESS`是不可见的。如果两个线程在该字段上竞争且竞争很重要，`WITNESS`不会警告。使用原子或适当的锁；永远不要因为没触发警告就假设无锁字段是安全的。

**不正确的保护。** 一个字段在写入路径上被互斥锁保护但在读取路径上没有互斥锁。间歇性撕裂读取结果。`WITNESS`不标记这个；第11章第3节的审计程序会。

所有这三者的治疗是编写`LOCKING.md`并保持其准确的纪律。`WITNESS`确认你声称持有的锁你实际上持有；文档确认你声称遵循的规则是设计意图的规则。

### 第6节总结

同步调试首先是词汇，然后是工具集。词汇是六种故障形状：丢失唤醒、虚假唤醒、锁顺序反转、过早销毁、持有不可睡眠锁时睡眠、detach与活动操作之间的竞争。每种都有可识别的签名和标准治疗。工具集是`WITNESS`、`INVARIANTS`、`KASSERT`、`sx_assert`、内核内调试器命令`show all locks`和`show sleepchain`，以及用户空间可观察性工具`lockstat(1)`和`dtrace(1)`。

第7节将这些工具在现实压力场景中投入使用。



## 第7节：使用现实I/O模式进行压力测试

第11章压力套件（`producer_consumer`、`mp_stress`、`mt_reader`、`lat_tester`）在简单的多线程和多进程工作负载下测试了数据路径。第12章驱动程序有新原语（cv、sx）和新代码路径（配置sysctl）。本节扩展测试以锻炼新表面，并展示如何阅读结果数据。

目标不是穷尽覆盖；而是*现实*覆盖。一个在类似真实生产流量的压力工作负载下存活的驱动程序是一个你可以信任的驱动程序。

### 现实是什么样的

真实驱动程序通常有：

- 多个生产者和消费者并发运行。
- 散布在各处的sysctl读取（监控工具、仪表板）。
- 偶尔的sysctl写入（管理员的配置更改）。
- 活动爆发与空闲期间交错。
- 混合优先级线程竞争CPU。

只测试这些轴之一的测试可能错过只在多个轴交互时出现的bug。例如，一个在数据路径操作中途以独占模式获取cfg sx的sysctl写入可能暴露纯I/O测试不会暴露的微妙排序问题。

### 复合工作负载

构建一个脚本同时运行三件事，持续固定时间：

```sh
#!/bin/sh
# 复合压力：I/O + sysctl读取者 + sysctl写入者。

DUR=60

(./mp_stress &) >/tmp/mp.log
(./mt_reader &) >/tmp/mt.log
(./config_writer $DUR &) >/tmp/cw.log

# 爆发sysctl读取者
for i in 1 2 3 4; do
    (while sleep 0.5; do
        sysctl -q dev.myfirst.0.stats >/dev/null
        sysctl -q dev.myfirst.0.debug_level >/dev/null
        sysctl -q dev.myfirst.0.soft_byte_limit >/dev/null
    done) &
done
SREAD_PIDS=$!

sleep $DUR

# 停止一切
pkill -f mp_stress
pkill -f mt_reader
pkill -f config_writer
kill $SREAD_PIDS 2>/dev/null

wait

echo "=== mp_stress ==="
cat /tmp/mp.log
echo "=== mt_reader ==="
cat /tmp/mt.log
echo "=== config_writer ==="
cat /tmp/cw.log
```

脚本运行一分钟。在这一分钟内，驱动程序看到：

- 两个写入者进程猛烈写入。
- 两个读取者进程猛烈读取。
- `mt_reader`中的四个pthread在单个描述符上猛烈读取。
- `config_writer`每10 ms切换调试级别和软字节限制。
- 四个shell循环每0.5秒读取sysctl。

在有`WITNESS`的调试内核下，这足够多的活动来捕获大多数锁排序和信号配对bug。运行它。如果它在没有panic、没有`WITNESS`警告、读取者和写入者字节计数一致的情况下完成，驱动程序通过了有意义的同步测试。

### 长时间变体

对于最微妙的bug，将复合运行一小时：

```sh
$ for i in $(seq 1 60); do
    ./composite_stress.sh
    echo "iteration $i complete"
    sleep 5
  done
```

60次一分钟测试的迭代给出一小时的累积覆盖。在一百万事件中出现一次的bug（这大致是一小时mp_stress在现代机器上产生的）通常在这次运行中浮出水面。

### 混合负载下的延迟

第11章`lat_tester`测量了没有其他负载时单次读取的延迟。在现实负载下，延迟讲述不同的故事：它包括等待互斥锁的时间、等待sx的时间和`cv_wait_sig`内的时间。

在`mp_stress`和`config_writer`运行时运行`lat_tester`。直方图应该显示比无负载情况更长的尾部。无争用操作几微秒，互斥锁短暂被其他线程持有时几十微秒，cv不得不实际睡眠等待数据时的毫秒级小尖峰。如果尾部延伸到秒，有问题。

### 阅读lockstat输出

`lockstat(1)`是测量锁争用的规范工具。在重度压力期间运行它：

```sh
# lockstat -P sleep 30 > /tmp/lockstat.out
```

`-P`标志包括自旋锁数据；没有它，只报告自适应锁。30意味着"采样30秒"。

输出按锁组织，有持有时间和等待时间统计。对于我们的驱动程序，寻找提及`myfirst0`（mtx）、`myfirst cfg`（sx）和cv（`myfirst data`、`myfirst room`）的行。

`myfirst`在典型压力下的健康结果：

- mtx被获取数百万次。每次获取的持有时间是几十纳秒。等待时间是偶尔的且很小。
- sx被获取数万次。大多数获取是共享的；少数独占获取对应sysctl写入。持有时间很低。
- cv以与I/O速率成比例的频率被信号通知和广播。每个cv上的等待计数对应读取者或写入者实际不得不阻塞的次数。

如果任何锁的等待时间占总时间的显著比例，那个锁被争用。修复是以下之一：更短的临界区、更细粒度的锁定，或不同的原语。

对于我们数据路径上单互斥锁设计的伪设备，mtx会在4-8个核心左右饱和，取决于cbuf操作的速度。这是预期的；我们没有为高核心数优化。本章的重点是正确性，不是吞吐量。

### 使用dtrace追踪

当特定事件需要可见性时，`dtrace`是正确的工具。示例：在10秒窗口内计数每个cv被信号通知的次数：

```sh
# dtrace -n 'fbt::cv_signal:entry { @[args[0]->cv_description] = count(); }' \
    -n 'fbt::cv_broadcastpri:entry { @[args[0]->cv_description] = count(); }' \
    -n 'tick-10sec { exit(0); }'
```

10秒后，dtrace打印一个表：

```text
 myfirst data           48512
 myfirst room           48317
 ...
```

如果工作负载对称（读写相等），`data_cv`和`room_cv`的数字应该大致相等。大的失衡意味着一方比另一方睡得更多，通常意味着流控问题。

另一个有用的一行程序：data cv上cv_wait延迟的直方图：

```sh
# dtrace -n '
fbt::_cv_wait_sig:entry /args[0]->cv_description == "myfirst data"/ {
    self->ts = timestamp;
}
fbt::_cv_wait_sig:return /self->ts/ {
    @ = quantize(timestamp - self->ts);
    self->ts = 0;
}
tick-10sec { exit(0); }
'
```

直方图显示线程在`_cv_wait_sig`内花费时间的分布。大多数应该很短（被迅速信号通知）。长尾部表示线程睡眠了很长时间，对空闲设备是正常的，但对繁忙设备是可疑的。

### 使用vmstat和top观察

对于更粗略的视图，在后台运行的`vmstat`和`top`提供上下文。

`vmstat 1`显示每秒统计：在用户、系统和空闲中花费的CPU时间；上下文切换；中断。在压力运行期间，`sy`（系统时间）应该上升；`cs`（上下文切换）也应该上升，因为cv信号通知。

`top -SH`（`-S`显示系统进程；`-H`显示单个线程）显示每线程CPU使用。在压力运行期间，测试线程应该是可见的。`WCHAN`列显示它们在等待什么；预期看到截断的cv描述（`myfirst `对`data_cv`和`room_cv`都是，因为尾随词被`WMESGLEN`截断）加上，对于仍在使用第11章匿名通道的任何线程，`&sc->cb`的地址打印为小数字字符串。

两者作为长时间压力运行的背景伴侣都是有用的。它们不产生结构化数据，但它们一目了然地确认事情正在发生。

### 观察Sysctl

压力期间的简单健全性检查：定期读取sysctl并验证它们合理。

```sh
$ while sleep 1; do
    sysctl dev.myfirst.0.stats.bytes_read \
           dev.myfirst.0.stats.bytes_written \
           dev.myfirst.0.stats.cb_used \
           dev.myfirst.0.debug_level \
           dev.myfirst.0.soft_byte_limit
  done
```

字节计数器应该单调递增。cb_used应该在某个范围内波动。配置应该随着`config_writer`更新而改变。

如果任何sysctl读取挂起（`sysctl`命令不返回），sysctl处理程序有同步问题。可能是持有的互斥锁阻塞了sysctl获取sx，或反之。从另一个终端使用`procstat -kk $$`查看挂起的shell在等待什么。

### 压力测试验收标准

驱动程序通过同步压力测试如果：

1. 复合脚本在没有panic的情况下完成。
2. `WITNESS`报告没有警告（`dmesg | grep -i witness | wc -l`返回零）。
3. 读取者和写入者的字节计数在彼此的1%以内（由于测试停止的时序，小的漂移是可接受的）。
4. `lockstat(1)`显示没有锁的等待时间超过总时间的5%。
5. `lat_tester`的延迟直方图显示空闲设备的第99百分位在一毫秒以下，或繁忙设备在配置的超时以下。
6. 重复运行（长时间循环）全部通过。

这些不是绝对阈值；它们是为本章示例服务的值。真实驱动程序可能有更严格或更宽松的边界，取决于工作负载。

### 详细解读lockstat输出

`lockstat(1)`产生的表在初次遇到时看起来令人生畏。对列的简短浏览揭开它们的神秘面纱。

被争用锁的典型行：

```text
Adaptive mutex spin: 1234 events in 30.000 seconds (41.13 events/sec)

------------------------------------------------------------------------
   Count   nsec     ----- Lock -----                       Hottest Caller
   1234     321     myfirst0                              myfirst_read+0x91
```

列的含义：

- `Count`：此类事件的数量（本例中为获取）。
- `nsec`：事件平均持续时间（此处为获取锁前平均自旋时间）。
- `Lock`：锁的名称。
- `Hottest Caller`：最常经历此事件的函数。

输出更下方：

```text
Adaptive mutex block: 47 events in 30.000 seconds (1.57 events/sec)

------------------------------------------------------------------------
   Count   nsec     ----- Lock -----                       Hottest Caller
     47   58432     myfirst0                              myfirst_read+0x91
```

"block"事件是自旋失败，线程不得不实际睡眠。平均睡眠时间为58微秒。那很高；意味着写入者在持有互斥锁期间做了应该是短临界区的事情。

合在一起，自旋事件（1234）和阻塞事件（47）告诉我们锁在30秒内被争用了1281次，其中96%的时间自旋成功。那是一个健康的模式：大多数争用是短暂的，只有罕见的长持有导致实际睡眠。

对于睡眠锁（sx、cv），列类似但事件分类不同：

```text
SX shared block: 2014 events in 30.000 seconds (67.13 events/sec)

------------------------------------------------------------------------
   Count   nsec     ----- Lock -----                       Hottest Caller
   2014    2105     myfirst cfg                            myfirst_get_debug_level+0x12
```

这表示：cfg sx上的共享等待者阻塞了2014次，平均等待2.1微秒，主要来自调试级别助手。有config写入者运行时，这是预期的。没有写入者时，它应该接近零。

阅读`lockstat`输出的关键技能是校准：知道你的工作负载预期什么数字。一个从未在负载下测量过的驱动程序是一个预期数字未知的驱动程序。用已知工作负载运行一次`lockstat`并保存输出作为基线。未来运行然后与基线比较；显著偏差是信号。

### 使用dtrace追踪特定代码路径

除了前面的cv计数和睡眠延迟示例，还有几个`dtrace`配方对第12章风格的驱动程序有用。

**每个cv每秒的cv等待计数：**

```sh
# dtrace -n '
fbt::_cv_wait_sig:entry { @[args[0]->cv_description] = count(); }
tick-1sec { printa(@); trunc(@); }'
```

打印cv等待的每秒计数，按cv名称分解。用于发现爆发。

**追踪哪个线程独占获取cfg sx：**

```sh
# dtrace -n '
fbt::_sx_xlock:entry /args[0]->lock_object.lo_name == "myfirst cfg"/ {
    printf("%s pid %d acquires cfg xlock\n", execname, pid);
    stack();
}'
```

用于确认唯一写入者是sysctl处理程序，不是其他意外路径。

**myfirst_read延迟直方图：**

```sh
# dtrace -n '
fbt::myfirst_read:entry { self->ts = timestamp; }
fbt::myfirst_read:return /self->ts/ {
    @ = quantize(timestamp - self->ts);
    self->ts = 0;
}
tick-30sec { exit(0); }'
```

与cv等待延迟直方图相同的模式，但在处理程序级别。包括在`cv_wait_sig`内花费的时间加上cbuf操作和uiomove内的时间。

这些配方是起点。内核函数的`dtrace`提供者（`fbt`）可以访问每个函数入口和返回；语言足够丰富来表达几乎任何聚合。

### 第7节总结

现实压力测试锻炼整个驱动程序，不只是一个路径。结合I/O、sysctl读取和sysctl写入的复合工作负载捕获纯I/O测试会错过的锁排序bug。`lockstat(1)`和`dtrace(1)`给你对锁和cv活动的可见性，而不修改驱动程序。一个在`WITNESS`内核上通过一小时复合压力套件的驱动程序是一个你可以有信心推进到下一章的驱动程序。

第8节以整理工作结束本章：文档传递、版本升级、回归测试和变更日志条目，告诉未来的你做了什么以及为什么。



## 第8节：重构和版本化你的同步驱动程序

驱动程序现在使用三个原语（`mtx`、`cv`、`sx`），有两个带文档记录顺序的锁类别，支持可中断和定时读取，并有一个小的配置子系统。剩余的工作是整理传递：清理源代码以提高清晰度、更新文档、升级版本、运行静态分析并验证回归测试通过。

本节涵盖每一项。没有一项是光鲜的。所有各项都是将工作驱动程序与可维护驱动程序区分开的东西。

### 清理源代码

经过一章的聚焦更改，源代码积累了一些值得整理的不一致。

**分组相关代码。** 将所有cv相关的助手移到彼此旁边（等待助手、信号调用、attach/detach中的cv_init/cv_destroy）。将所有sx相关的助手移到一起。编译器不关心排序，但读者关心。

**标准化宏词汇。** 第11章引入了`MYFIRST_LOCK`、`MYFIRST_UNLOCK`、`MYFIRST_ASSERT`。为sx添加对称集合：

```c
#define MYFIRST_CFG_SLOCK(sc)   sx_slock(&(sc)->cfg_sx)
#define MYFIRST_CFG_SUNLOCK(sc) sx_sunlock(&(sc)->cfg_sx)
#define MYFIRST_CFG_XLOCK(sc)   sx_xlock(&(sc)->cfg_sx)
#define MYFIRST_CFG_XUNLOCK(sc) sx_xunlock(&(sc)->cfg_sx)
#define MYFIRST_CFG_ASSERT_X(sc) sx_assert(&(sc)->cfg_sx, SA_XLOCKED)
#define MYFIRST_CFG_ASSERT_S(sc) sx_assert(&(sc)->cfg_sx, SA_SLOCKED)
```

现在驱动程序中的每个锁获取都通过宏。如果我们后来从`sx`切换到`rw`，更改在一个头文件中，而不是散布在源代码中。

**消除死代码。** 如果第11章的助手不再被调用（也许旧的wakeup通道已消失），删除它。死代码吸引混乱。

**注释不明显的部分。** 遵循锁顺序规则的每个锁获取值得一行注释。每个使用快照并应用的地方值得一个注释解释原因。锁定是驱动程序最微妙的部分；注释应该反映这一点。

### 更新LOCKING.md

第11章`LOCKING.md`记录了一个锁和一小集字段。第12章驱动程序有更多要说的。新版本：

```markdown
# myfirst锁定策略

版本 0.6-sync（第12章）。

## 概述

驱动程序使用三个同步原语：一个睡眠互斥锁
(sc->mtx)用于数据路径，一个sx锁(sc->cfg_sx)用于
配置子系统，和两个条件变量(sc->data_cv,
sc->room_cv)用于阻塞读取和写入。字节计数器使用
counter(9)每CPU计数器并自我保护。

## 此驱动程序拥有的锁

### sc->mtx (mutex(9), MTX_DEF)

保护：
- sc->cb（环形缓冲区的内部状态）
- sc->open_count, sc->active_fhs
- sc->is_attached（写入；处理程序入口处的读取可能不加锁
  作为优化，在每次睡眠后重新检查）

### 无锁普通整数

- sc->read_timeout_ms, sc->write_timeout_ms：普通int，不加锁
  访问。安全是因为对齐的int读写在FreeBSD支持的每个架构上
  都是原子的，值是建议性的；陈旧的读取只是为下次等待产生
  略微不同的超时。sysctl框架通过CTLFLAG_RW直接写入它们。

### sc->cfg_sx (sx(9))

保护：
- sc->cfg.debug_level
- sc->cfg.soft_byte_limit
- sc->cfg.nickname

共享模式：每个cfg字段的每次读取。
独占模式：每个cfg字段的每次写入。

### sc->data_cv (cv(9))

等待条件：cbuf中有数据可用。
互锁：sc->mtx。
信号通知者：成功cbuf写入后的myfirst_write。
广播者：myfirst_detach。
等待者：myfirst_wait_data中的myfirst_read。

### sc->room_cv (cv(9))

等待条件：cbuf中有空间可用。
互锁：sc->mtx。
信号通知者：成功cbuf读取后的myfirst_read，以及
重置cbuf后的myfirst_sysctl_reset。
广播者：myfirst_detach。
等待者：myfirst_wait_room中的myfirst_write。

## 无锁字段

- sc->bytes_read, sc->bytes_written：counter_u64_t。通过
  counter_u64_add更新；通过counter_u64_fetch读取。

## 锁顺序

sc->mtx -> sc->cfg_sx

持有sc->mtx的线程可以以任何模式获取sc->cfg_sx。
持有sc->cfg_sx的线程不能获取sc->mtx。

理由：数据路径总是持有sc->mtx，可能需要在其临界区期间
读取配置。配置路径（sysctl写入者）不需要数据互斥锁；
如果未来功能需要两者，它必须先获取sc->mtx。

## 锁定纪律

1. 使用MYFIRST_LOCK(sc)获取互斥锁，使用MYFIRST_UNLOCK(sc)释放。
2. 使用MYFIRST_CFG_SLOCK以共享模式获取sx，使用
   MYFIRST_CFG_XLOCK独占。使用匹配的解锁释放。
3. 使用cv_wait_sig（可中断）或cv_timedwait_sig（可中断+
   有界）在cv上等待。
4. 使用cv_signal（一个等待者）或cv_broadcast（所有
   等待者）信号通知cv。仅对影响所有等待者的状态更改
   使用cv_broadcast（detach、配置重置）。
5. 绝不跨uiomove(9)、copyin(9)、copyout(9)、
   selwakeup(9)或wakeup(9)持有sc->mtx。这些中的每一个
   可能睡眠或获取其他锁。cv_wait_sig是例外（它原子释放
   互锁）。
6. 绝不跨uiomove(9)等持有sc->cfg_sx，原因相同。
7. 所有cbuf_*调用必须在持有sc->mtx时发生（助手
   断言MA_OWNED）。
8. detach路径在sc->mtx下清除sc->is_attached，广播
   两个cv，并在active_fhs > 0时拒绝detach。

## 快照并应用模式

当路径需要sc->mtx和sc->cfg_sx两者时，它应该遵循
快照并应用模式：

  1. sx_slock(&sc->cfg_sx)；将cfg读入局部变量；
     sx_sunlock(&sc->cfg_sx)。
  2. MYFIRST_LOCK(sc)；使用快照做cbuf操作；
     MYFIRST_UNLOCK(sc)。

快照在使用时可能略微过时。对于建议性的配置值
（调试级别、软字节限制），这是可接受的。

## 已知的非锁定访问

### 处理程序入口处的sc->is_attached

无保护的普通读取。安全是因为：
- 陈旧的"true"在每次睡眠后通过
  if (!sc->is_attached) return (ENXIO)重新检查。
- 陈旧的"false"导致处理程序提前返回ENXIO，这也是
  它用新鲜的false会做的。

### sysctl读取时的sc->open_count, sc->active_fhs

无保护的普通加载。在amd64和arm64上安全（对齐的64位
加载是原子的）。在i386上可接受，因为撕裂读取如果曾经
发生，会产生一个没有正确性影响的单个错误统计。

## 等待通道

- sc->data_cv：数据已变得可用。
- sc->room_cv：空间已变得可用。

（第10章的遗留&sc->cb唤醒通道已在第12章中退役。）

## 历史

- 0.6-sync（第12章）：添加了cv通道，配置的sx，
  通过cv_timedwait_sig的有界读取。
- 0.5-kasserts（第11章，阶段5）：在整个cbuf助手和
  等待助手中添加了KASSERT调用。
- 0.5-counter9（第11章，阶段3）：字节计数器迁移到
  counter(9)。
- 0.5-concurrency（第11章，阶段2）：MYFIRST_LOCK/UNLOCK/ASSERT
  宏，显式锁定策略。
- 更早版本：见第10章/第11章历史。
```

该文档现在是驱动程序同步故事的权威描述。任何未来的更改在与代码更改相同的提交中更新文档。想要知道更改是否安全的审查者阅读与文档的diff，不是与代码的diff。

### 升级版本

更新版本字符串：

```c
#define MYFIRST_VERSION "0.6-sync"
```

在attach时打印它（attach中现有的`device_printf`行已经包含版本）：

```c
device_printf(dev,
    "Attached; version %s, node /dev/%s (alias /dev/myfirst), "
    "cbuf=%zu bytes\n",
    MYFIRST_VERSION, devtoname(sc->cdev), cbuf_size(&sc->cb));
```

更新变更日志：

```markdown
## 0.6-sync (第12章)

- 将匿名唤醒通道(&sc->cb)替换为两个命名条件变量
  (sc->data_cv, sc->room_cv)。
- 通过sc->read_timeout_ms sysctl添加了有界读取支持，
  底层使用cv_timedwait_sig。
- 添加了由sx锁(sc->cfg_sx)保护的小配置子系统(sc->cfg)。
- 为debug_level、soft_byte_limit和nickname添加了sysctl处理程序。
- 添加了myfirst_sysctl_reset，以规范顺序获取两个锁来清除
  cbuf并重置计数器。
- 使用新原语、锁顺序和快照并应用模式更新了LOCKING.md。
- 添加了与现有MYFIRST_*互斥锁宏对称的MYFIRST_CFG_*宏。
- 所有第11章测试继续通过；在userland/下添加了基于sysctl的新测试。
```

### 更新README

第11章README命名了驱动程序并描述了其功能。第12章README添加了新的：

```markdown
# myfirst

一个FreeBSD 14.3伪设备驱动程序，演示缓冲I/O、
并发和现代同步原语。作为书籍"FreeBSD Device Drivers: From First
Steps to Kernel Mastery"的运行示例开发。

## 状态

版本 0.6-sync（第12章）。

## 功能

- nexus0下的Newbus伪设备。
- 主设备节点在/dev/myfirst/0（别名：/dev/myfirst）。
- 环形缓冲区(cbuf)作为I/O缓冲区。
- 阻塞、非阻塞和定时读写。
- 通过d_poll和selinfo的poll(2)支持。
- 通过counter(9)的每CPU字节计数器。
- 单个睡眠互斥锁保护复合cbuf状态；见LOCKING.md。
- 两个命名条件变量(data_cv, room_cv)协调读写阻塞。
- sx锁保护运行时配置(debug_level, soft_byte_limit, nickname)。

## 配置

通过sysctl的三个运行时可调参数：

- dev.myfirst.<unit>.debug_level (0-3)：控制dmesg详细程度。
- dev.myfirst.<unit>.soft_byte_limit：拒绝会将cb_used推高到此
  阈值以上的写入（0 = 无限制）。
- dev.myfirst.<unit>.nickname：日志消息中使用的字符串。
- dev.myfirst.<unit>.read_timeout_ms：限制阻塞读取。

（最后一个是每实例的；详情见myfirst.4，待编写。）

## 构建和加载

    $ make
    # kldload ./myfirst.ko
    # dmesg | tail
    # ls -l /dev/myfirst
    # printf 'hello' > /dev/myfirst
    # cat /dev/myfirst
    # kldunload myfirst

## 测试

见../../userland/中的测试程序。第12章测试
包括config_writer（压力期间切换sysctl）和
timeout_tester（验证有界读取）。

## 许可证

BSD 2-Clause。见单个源文件中的SPDX头。
```

### 运行静态分析

对第12章驱动程序运行`clang --analyze`：

```sh
$ make WARNS=6 clean all
$ clang --analyze -D_KERNEL -I/usr/src/sys \
    -I/usr/src/sys/amd64/conf/GENERIC myfirst.c
```

分类输出。第11章以来的新警告应该是：

1. 误报（clang不理解锁定纪律）。记录每一个。
2. 真实bug。修复每一个。

驱动程序代码中常见的误报涉及clang无法看透的`sx_assert`和`mtx_assert`宏；分析器认为锁可能没有被持有，即使断言证明它被持有了。这些可以用`__assert_unreachable()`消除，或通过重构代码使锁状态对分析器更明显。

### 运行回归套件

第11章回归脚本自然扩展：

```sh
#!/bin/sh
# regression.sh：完整第12章回归。

set -eu

die() { echo "FAIL: $*" >&2; exit 1; }
ok()  { echo "PASS: $*"; }

[ $(id -u) -eq 0 ] || die "must run as root"
kldstat | grep -q myfirst && kldunload myfirst
[ -f ./myfirst.ko ] || die "myfirst.ko not built; run make first"

kldload ./myfirst.ko
trap 'kldunload myfirst 2>/dev/null || true' EXIT

sleep 1
[ -c /dev/myfirst ] || die "device node not created"
ok "load"

# 第7-10章测试。
printf 'hello' > /dev/myfirst || die "write failed"
cat /dev/myfirst >/tmp/out.$$
[ "$(cat /tmp/out.$$)" = "hello" ] || die "round-trip content mismatch"
rm -f /tmp/out.$$
ok "round-trip"

cd ../userland && make -s clean && make -s && cd -

../userland/producer_consumer || die "producer_consumer failed"
ok "producer_consumer"

../userland/mp_stress || die "mp_stress failed"
ok "mp_stress"

# 第12章特定测试。
../userland/timeout_tester || die "timeout_tester failed"
ok "timeout_tester"

../userland/config_writer 5 &
CW=$!
../userland/mt_reader || die "mt_reader (under config writer) failed"
wait $CW
ok "mt_reader under config writer"

sysctl dev.myfirst.0.stats >/dev/null || die "sysctl stats not accessible"
sysctl dev.myfirst.0.debug_level >/dev/null || die "sysctl debug_level not accessible"
sysctl dev.myfirst.0.soft_byte_limit >/dev/null || die "sysctl soft_byte_limit not accessible"
ok "sysctl"

# WITNESS检查。
WITNESS_HITS=$(dmesg | grep -ci "witness\|lor" || true)
if [ "$WITNESS_HITS" -gt 0 ]; then
    die "WITNESS warnings detected ($WITNESS_HITS lines)"
fi
ok "witness clean"

echo "ALL TESTS PASSED"
```

每次提交后的绿色运行是最低门槛。在`WITNESS`内核上长时间复合后的绿色运行是更高的门槛。

### 提交前检查清单

第11章检查清单为第12章增加了两个新项目：

1. 我是否用任何新锁、cv或顺序更改更新了`LOCKING.md`？
2. 我是否在`WITNESS`内核上运行了完整回归套件？
3. 我是否运行了至少30分钟的长时间复合压力？
4. 我是否运行了`clang --analyze`并分类了每个新警告？
5. 我是否为任何期望锁状态的新助手添加了`sx_assert`或`mtx_assert`？
6. 我是否升级了版本字符串并更新了`CHANGELOG.md`？
7. 我是否验证了测试套件构建并运行？
8. **（新）** 我是否检查了每个cv既有信号通知者又有文档记录的条件？
9. **（新）** 我是否检查了每个sx_xlock在每个代码路径（包括错误路径）上都有配对的sx_xunlock？

两个新项目捕获了第12章风格代码中最常见的bug。没有信号通知者的cv是死权重（等待者永远不会醒来）。错误路径上没有配对解锁的sx_xlock是一个安静的等待发生的死锁。

### 第8节总结

驱动程序现在不仅正确，而且可验证地正确、有良好文档记录并有版本控制。它有：

- 更新的`LOCKING.md`，描述三个原语、两个锁类别和一个规范锁顺序。
- 反映第12章工作的新版本字符串（0.6-sync）。
- 锻炼每个原语并验证`WITNESS`清洁度的回归脚本。
- 捕获第12章引入的两个新故障模式的提交前检查清单。

那是本章主要教学弧线的结束。实验和挑战紧随其后。



## 动手实验

这些实验通过直接动手经验巩固第12章概念。它们按从最不要求到最要求排序。每个设计为在单个实验会话中可完成。

### 实验前设置检查清单

在开始任何实验之前，确认以下四项。第11章检查清单适用；我们添加三个第12章特有的。

1. **调试内核运行中。** `sysctl kern.ident`报告带有`INVARIANTS`和`WITNESS`的内核。
2. **WITNESS活跃。** `sysctl debug.witness.watch`返回非零值。
3. **驱动程序源代码匹配第11章阶段5（kasserts）。** 从你的驱动程序目录，`make clean && make`应该干净编译。
4. **干净的dmesg。** 在第一个实验前`dmesg -c >/dev/null`一次。
5. **（新）配套userland已构建。** 从`examples/part-03/ch12-synchronization-mechanisms/userland/`，`make`应该产生`config_writer`和`timeout_tester`二进制文件。
6. **（新）第11章压力套件可用。** 实验重用第11章的`mp_stress`、`mt_reader`和`producer_consumer`。
7. **（新）阶段5的备份。** 在开始任何修改源代码的实验之前，将工作阶段5驱动程序复制到安全位置。几个实验故意引入需要干净恢复的bug。

### 实验12.1：用条件变量替换匿名唤醒通道

**目标。** 将第11章驱动程序从匿名通道`&sc->cb`上的`mtx_sleep`/`wakeup`转换为两个命名条件变量（`data_cv`和`room_cv`）。

**步骤。**

1. 将你的阶段5驱动程序复制到`examples/part-03/ch12-synchronization-mechanisms/stage1-cv-channels/`。
2. 向`struct myfirst_softc`添加`struct cv data_cv`和`struct cv room_cv`。
3. 在`myfirst_attach`中，调用`cv_init(&sc->data_cv, "myfirst data")`和`cv_init(&sc->room_cv, "myfirst room")`。将它们放在互斥锁init之后。
4. 在`myfirst_detach`中，在`mtx_destroy`之前，对每个cv调用`cv_broadcast`唤醒任何睡眠者，然后对每个调用`cv_destroy`。
5. 将`myfirst_wait_data`和`myfirst_wait_room`中的`mtx_sleep(&sc->cb, ...)`调用分别替换为`cv_wait_sig(&sc->data_cv, &sc->mtx)`和`cv_wait_sig(&sc->room_cv, &sc->mtx)`。
6. 将`myfirst_read`和`myfirst_write`中的`wakeup(&sc->cb)`调用分别替换为`cv_signal(&sc->room_cv)`和`cv_signal(&sc->data_cv)`。注意交换：成功读取释放空间（所以唤醒写入者）；成功写入产生数据（所以唤醒读取者）。
7. 构建、加载、运行第11章压力套件。

**验证。** 所有第11章测试通过。对着睡眠读取者的`procstat -kk`显示等待通道`myfirst `（`"myfirst data"`的截断形式；见第2节关于`WMESGLEN`的说明）。没有`WITNESS`警告。

**延伸目标。** 使用`dtrace`在`mp_stress`期间计数每个cv的信号。确认data_cv和room_cv之间的信号计数大致相等（因为读取和写入大致相等）。

### 实验12.2：添加有界读取

**目标.** 添加一个限制阻塞读取的`read_timeout_ms` sysctl。

**步骤.**

1. 将实验12.1复制到`stage2-bounded-read/`。
2. 向softc添加`int read_timeout_ms`字段。在attach中初始化为0。
3. 在`dev.myfirst.<unit>.read_timeout_ms`下注册一个`SYSCTL_ADD_INT`，带`CTLFLAG_RW`。
4. 修改`myfirst_wait_data`在`read_timeout_ms > 0`时使用`cv_timedwait_sig`，将毫秒转换为tick。将`EWOULDBLOCK`转换为`EAGAIN`。
5. 构建并加载。
6. 从`examples/part-03/ch12-synchronization-mechanisms/userland/`构建`timeout_tester`。
7. 将sysctl设置为100，运行`timeout_tester`，观察`read(2)`在大约100 ms后返回`EAGAIN`。
8. 将sysctl重置为0，再次运行`timeout_tester`。读取阻塞直到你Ctrl-C，返回`EINTR`。

**验证.** `timeout_tester`的输出在超时和信号中断两种情况下都符合预期。压力套件仍然通过。

**延伸目标.** 添加一个对称的`write_timeout_ms` sysctl并验证它在缓冲区满时限制写入。

### 实验12.3：添加sx保护的配置子系统

**目标.** 添加第5节的`cfg`结构和`cfg_sx`锁；将`debug_level`公开为sysctl。

**步骤.**

1. 将实验12.2复制到`stage3-sx-config/`。
2. 向softc添加`struct sx cfg_sx`和`struct myfirst_config cfg`。在attach中初始化（`sx_init(&sc->cfg_sx, "myfirst cfg")`；cfg字段的默认值）。在detach中销毁。
3. 按照快照并应用模式添加`myfirst_sysctl_debug_level`处理程序。注册它。
4. 添加一个通过`sx_slock`查询`sc->cfg.debug_level`的`MYFIRST_DBG(sc, level, fmt, ...)`宏。
5. 在读/写路径中散布一些`MYFIRST_DBG(sc, 1, ...)`调用来记录缓冲区变空或满时。
6. 构建并加载。
7. 运行`mp_stress`。确认没有日志垃圾（debug_level默认为0）。
8. `sysctl -w dev.myfirst.0.debug_level=2`并再次运行`mp_stress`。现在`dmesg`应该显示调试消息。
9. 将级别重置为0。

**验证.** 调试消息随sysctl更改出现和消失。切换期间没有`WITNESS`警告。

**延伸目标.** 添加`soft_byte_limit` sysctl。将其设置为1024并运行产生4096字节爆发的写入者；确认写入者提前看到`EAGAIN`。

### 实验12.4：使用DDB检查持有的锁

**目标.** 使用内核内调试器检查挂起的测试。

**步骤.**

1. 确保调试内核有`options DDB`和配置好的进入DDB的方式（通常是串行控制台上的`Ctrl-Alt-Esc`，或`Break`键）。
2. 加载实验12.3的驱动程序。
3. 在一个终端开始`cat /dev/myfirst`。它阻塞（没有生产者）。
4. 从控制台（或通过`sysctl debug.kdb.enter=1`），进入DDB。
5. 运行`show all locks`。注意任何持有锁的线程。
6. 运行`ps`。找到`cat`进程和`myfirst data`等待通道。
7. 对cat线程运行`bt <pid> <tid>`。确认反向追踪以`_cv_wait_sig`结束。
8. `continue`离开DDB。
9. 向cat发送`SIGINT`（Ctrl-C）。

**验证.** cat返回`EINTR`。没有panic。你有DDB会话的转录。

**延伸目标.** 在`mp_stress`同时运行时重复。比较`show all locks`输出：更多锁、更多活动，但相同形状。

### 实验12.5：检测故意的锁顺序反转

**目标.** 引入一个故意的LOR并观察`WITNESS`捕获它。

**步骤.**

1. 将实验12.3复制到一个临时目录`stage-lab12-5/`。不要原地修改实验12.3。
2. 添加一个违反锁顺序的路径。例如，在一个小的实验sysctl处理程序中：

   ```c
   /* 错误：sx第一，然后mtx，反转规范顺序。 */
   sx_xlock(&sc->cfg_sx);
   MYFIRST_LOCK(sc);
   /* 微小工作 */
   MYFIRST_UNLOCK(sc);
   sx_xunlock(&sc->cfg_sx);
   ```

3. 构建并在`WITNESS`内核上加载。
4. 运行`mp_stress`（通过数据路径使用规范顺序）并同时触发新sysctl。
5. 观察`dmesg`中的`lock order reversal`警告。
6. 记录警告文本。注意行号。
7. 删除临时目录；不要提交bug。

**验证.** `dmesg`显示命名两个锁和两个源位置的`lock order reversal`警告。

**延伸目标.** 仅从`WITNESS`输出确定规范顺序首次在哪里建立。在源代码中那一行打开并确认。

### 实验12.6：长时间运行复合压力

**目标.** 运行第7节的复合压力工作负载30分钟并验证干净。

**步骤.**

1. 启动调试内核。
2. 构建并加载`examples/part-03/ch12-synchronization-mechanisms/stage4-final/`。这是最终集成驱动程序（cv通道 + 有界读取 + sx保护的配置 + 重置sysctl）。第7节复合脚本触摸的所有sysctl都在这里存在。
3. 构建userland测试器。
4. 将第7节复合压力脚本保存为`composite_stress.sh`。
5. 将其包装在30分钟循环中：
   ```sh
   for i in $(seq 1 30); do
     ./composite_stress.sh
     echo "iteration $i done"
   done
   ```
6. 定期监控`dmesg`。
7. 完成后，检查：
   - `dmesg | grep -ci witness`返回0。
   - 所有循环迭代完成。
   - `vmstat -m | grep cbuf`显示预期的静态分配（没有增长）。

**验证.** 所有标准满足。驱动程序在调试内核上30分钟复合压力下存活，没有警告、panic或内存增长。

**延伸目标.** 在专用测试机器上运行相同循环24小时。在这个规模出现的bug是生产中代价最高的。

### 实验12.7：验证快照并应用模式在争用下成立

**目标.** 展示`myfirst_write`中的快照并应用模式正确处理对软字节限制的并发更新。

**步骤.**

1. 将软字节限制设置为一个小值：`sysctl -w dev.myfirst.0.soft_byte_limit=512`。
2. 用两个写入者和两个读取者启动`mp_stress`。
3. 从第三个终端，重复切换限制：`while sleep 0.1; do sysctl -w dev.myfirst.0.soft_byte_limit=$RANDOM; done`。
4. 观察写入者输出。一些写入会成功；其他会返回`EAGAIN`（在检查时刻限制低于当前cb_used）。
5. 观察`dmesg`看`WITNESS`警告。

**验证.** 没有`WITNESS`警告。`mp_stress`中的字节计数略低于正常（因为一些写入被拒绝），但总写入约等于总读取。

**延伸目标.** 修改`myfirst_write`通过在持有数据互斥锁时获取cfg sx来违反锁顺序规则。重新加载，运行相同测试。`WITNESS`应该在第一次同时使用两个路径的运行时触发。恢复更改。

### 实验12.8：使用lockstat分析

**目标.** 使用`lockstat(1)`在压力下描述被争用的锁。

**步骤.**

1. 在调试内核上加载实验12.3的驱动程序。
2. 在一个终端启动`mp_stress`。
3. 从另一个终端，运行`lockstat -P sleep 30 > /tmp/lockstat.out`。
4. 打开输出文件。找到`myfirst0`（mtx）和`myfirst cfg`（sx）的条目。
5. 注意：最大持有时间、平均持有时间、最大等待时间、平均等待时间和获取计数。
6. 在`config_writer`运行时重复。比较`myfirst cfg`数字。

**验证.** 数字匹配预期配置文件。互斥锁显示数百万次获取，持有时间短。sx显示数万次获取，大多数是共享的，持有时间非常短。

**延伸目标.** 修改驱动程序人为延长临界区（例如，在互斥锁内添加10 ms的`pause(9)`）。重新运行`lockstat`。观察争用尖峰。恢复修改。



## 挑战练习

挑战将第12章扩展到基线实验之外。每个都是可选的；每个设计为加深你的理解。

### 挑战1：使用sx_downgrade进行配置刷新

`myfirst_sysctl_debug_level`处理程序当前释放共享锁并重新获取独占锁。替代方案是获取共享，尝试`sx_try_upgrade`，修改后`sx_downgrade`。实现这个变体。比较争用下的行为。每个模式什么时候赢？

### 挑战2：使用cv_broadcast实现排空操作

添加一个"排空"cbuf的ioctl或sysctl：阻塞直到`cb_used == 0`，然后返回。实现应该在条件`cb_used > 0`上使用`cv_wait_sig(&sc->room_cv, ...)`的循环。验证排空后的`cv_broadcast(&sc->room_cv)`唤醒每个等待者，不只是一个。

### 挑战3：cv_wait延迟的dtrace脚本

编写一个`dtrace`脚本，产生线程在`data_cv`和`room_cv`上每个`cv_wait_sig`内花费多长时间的直方图。在`mp_stress`期间运行它。分布看起来怎么样？长尾部在哪里？

### 挑战4：用匿名通道替换cv

使用匿名通道上的`mtx_sleep`和`wakeup`重新实现数据和空间条件（回归到第11章设计）。运行测试。驱动程序应该仍然工作，但`procstat -kk`输出和`dtrace`查询变得更少信息。描述可读性差异。

### 挑战5：添加每描述符的read_timeout_ms

`read_timeout_ms` sysctl是每设备的。通过`ioctl(2)`添加每描述符超时：`MYFIRST_SET_READ_TIMEOUT(int ms)`在文件描述符上设置该描述符的超时。驱动程序代码变得更有趣，因为超时现在存在于`struct myfirst_fh`而不是`struct myfirst_softc`中。注意：每fh状态不与其他描述符共享（字段本身不需要锁），但超时的选择仍然影响等待助手。

### 挑战6：使用rw(9)代替sx(9)

将`sx_init`替换为`rw_init`，`sx_xlock`替换为`rw_wlock`等。运行测试。什么坏了？（提示：cfg路径可能包括睡眠操作；rw不可睡眠。）失败看起来什么样？`rw(9)`什么时候是正确的选择？

### 挑战7：实现多cv排空

驱动程序有两个cv。假设detach应该只在两个cv都有零等待者时才被认为是完成的。在detach中实现一个循环检查，直到`data_cv.cv_waiters == 0`和`room_cv.cv_waiters == 0`，检查之间短暂睡眠。（注意：从cv API外部直接访问`cv_waiters`是不可移植的；这是一个理解内部状态的练习。真实生产代码应该使用不同的机制。）

### 挑战8：锁顺序可视化

使用`dtrace`或`lockstat`生成`mp_stress`期间锁获取的图。节点是锁；边是"A的持有者在仍持有A时获取了B"。将图与你的`LOCKING.md`锁顺序比较。有没有你未预期的获取？

### 挑战9：睡眠通道比较

构建两个版本的驱动程序：一个使用cv（第12章默认），一个使用匿名通道上的遗留`mtx_sleep`/`wakeup`（第11章默认）。在两者上运行相同工作负载。测量：最大吞吐量、第99百分位延迟、`WITNESS`清洁度和源代码可读性。写一页报告。

### 挑战10：限制配置写入

第12章驱动程序允许随时配置写入。添加一个sysctl `cfg_write_cooldown_ms`限制配置更改的频率（例如，每100 ms最多一次写入）。用cfg结构中的时间戳字段和每个cfg sysctl处理程序中的检查实现。决定当冷却被违反时做什么：返回`EBUSY`、排队更改，还是静默合并。记录选择。



## 故障排除

此参考目录了你在完成第12章期间最可能遇到的bug。

### 症状：读取者永远挂起尽管数据已写入

**原因.** 丢失唤醒。写入者添加了字节但没有信号通知`data_cv`，或信号通知了错误的cv。

**修复.** 搜索源代码中每个添加字节的地方；确保调用了`cv_signal(&sc->data_cv)`。确认cv是等待者阻塞的那个。

### 症状：WITNESS警告sc->mtx和sc->cfg_sx之间的"lock order reversal"

**原因.** 一个路径以错误顺序获取了锁。规范顺序是互斥锁第一，sx第二。

**修复.** 找到违规路径（警告命名了行）。要么重新排序获取以匹配规范顺序，要么重构路径避免同时持有两个锁（快照并应用）。

### 症状：cv_timedwait_sig立即返回EWOULDBLOCK

**原因.** 以tick为单位的超时为零或负数。最可能是毫秒到tick的转换向下取整到零。

**修复.** 使用`(timo_ms * hz + 999) / 1000`公式向上取整到至少一个tick。验证`hz`是预期值（FreeBSD 14.3上典型为1000）。

### 症状：detach挂起

**原因.** 线程正在睡眠在尚未被广播的cv上，或detach正在等待`active_fhs > 0`降为零且有描述符打开。

**修复.** 确认detach在active_fhs检查之前广播两个cv。从单独终端使用`fstat | grep myfirst`找到任何持有设备打开的进程；杀死它。

### 症状：sysctl写入挂起

**原因.** sysctl处理程序正在等待一个被线程做阻塞事情持有的锁。最常见的是，cfg sx被慢的`sysctl_handle_string`以独占模式持有。

**修复.** 验证sysctl处理程序遵循快照并应用模式：共享获取、读取、释放；然后锁外`sysctl_handle_*`；然后独占锁提交。跨`sysctl_handle_*`持有锁是bug。

### 症状：sx_destroy因"lock still held"而panic

**原因.** `sx_destroy`在另一个线程仍持有锁或正在等待它时被调用。

**修复.** 确认detach在`active_fhs > 0`时拒绝继续。确认detach开始后没有内核线程或callout正在使用cfg sx。

### 症状：cv_signal或cv_broadcast没有唤醒可见的东西

**原因.** 信号时cv上没有人在等待。当等待队列为空时`cv_signal`和`cv_broadcast`都是无操作，唤醒侧的`dtrace`探针看不到后续活动。

**修复.** 无需修复；空唤醒是正确且无害的。如果你预期有等待者但没有，bug在上游：要么等待者从未到达`cv_wait_sig`，要么唤醒者目标错误的cv。通过`dtrace`确认信号在你意图的cv上触发，对着等待者`procstat -kk`确认它在哪里睡眠。

### 症状：read_timeout_ms设置为100产生200 ms延迟

**原因.** 内核的`hz`值低于预期。`+999`向上取整意味着100 Hz的`hz`上100 ms超时变为10 tick（100 ms），但如果`hz=10`它变为1 tick（100 ms）。不同的取整。

**修复.** 用`sysctl kern.clockrate`确认`hz`。对于更严格的超时，直接使用带`SBT_1MS * timo_ms`的`cv_timedwait_sig_sbt`避免tick取整。

### 症状：故意的错误锁顺序不产生WITNESS警告

**原因.** 要么有bug的路径未被测试使用，要么`WITNESS`在运行内核中未启用。

**修复.** 确认`sysctl debug.witness.watch`返回非零。确认违规路径运行（添加`device_printf`验证）。在`mp_stress`下运行测试以最大化bug浮出水面的机会。

### 症状：lockstat在数据互斥锁上显示巨大的等待时间

**原因.** 互斥锁被跨长操作持有。常见罪魁祸首：意外在临界区内的`uiomove`；持有锁时打印大字符串的调试`device_printf`。

**修复.** 审计临界区。将长操作移出。互斥锁应该被持有几十纳秒，不是微秒。

### 症状：第12章更改后mp_stress报告字节计数不匹配

**原因.** cv重构期间错过了唤醒。读取者在写入者的信号已传递后开始等待（信号时无等待者，信号丢失）。

**修复.** 验证等待助手在`cv_wait_sig`周围使用`while`，不是`if`。验证信号发生在状态更改之后，不是之前。

### 症状：timeout_tester显示长于配置超时的延迟

**原因.** 调度器延迟。内核在定时器触发后几毫秒调度了线程。这是正常的；预期几毫秒的抖动。

**修复.** 对于典型工作负载无需修复。对于实时工作负载，通过`rtprio(2)`提升线程优先级。

### 症状：当没有描述符打开时kldunload报告忙

**原因.** taskqueue或后台线程仍在使用驱动程序中的原语。（本章不应该发生，但值得知道。）

**修复.** 审计任何taskqueue、callout或kthread生成代码。detach必须在声明可以安全卸载之前排空或终止所有这些。

### 症状：cv_wait_sig立即唤醒并返回0

**原因.** 信号在等待设置时到达，或cv被刚好在等待发出之前运行的线程信号通知。实际上不是bug；`while`循环应该处理它。

**修复.** 确认周围的`while (!condition)`重新检查。循环将虚假唤醒变成无操作：重新检查，发现条件为假，再次睡眠。

### 症状：同一cv上的两个等待者以意外顺序被唤醒

**原因.** `cv_signal`唤醒一个由睡眠队列策略选择的等待者（最高优先级，相等中FIFO）。如果它们的优先级不同，它不按到达顺序唤醒它们。

**修复.** 通常无需修复；内核的选择是正确的。如果你需要严格的到达顺序唤醒，使用不同的设计（每等待者cv，或显式队列）。

### 症状：重读取者负载下sx_xlock花费几秒获取

**原因.** 许多共享持有者，每个释放缓慢，因为cfg sx在每个I/O上被获取和释放。写入者被持续不断的读取者滴流饿死。

**修复.** 内核使用`SX_LOCK_WRITE_SPINNER`标志在写入者开始等待后给予它们优先级；饿死是有界的但仍可产生可见延迟。如果延迟不可接受，重新设计以便写入者在静默窗口期间或在不同协议下发生。

### 症状：测试在非WITNESS内核上通过但在WITNESS上失败

**原因.** 几乎总是`WITNESS`检测到的真实bug。最常见：在违反全局顺序的路径中获取锁，但因为争用工作负载未被命中所以死锁尚未显现。

**修复.** 仔细阅读`WITNESS`警告。警告文本包括每个违规的源位置。修复违规；测试应该然后在两个内核上都通过。

### 症状：锁定宏在非调试内核上展开为空

**原因.** 这是设计如此。`mtx_assert`、`sx_assert`、`KASSERT`和`MYFIRST_ASSERT(sc)`（它展开为`mtx_assert`）在没有`INVARIANTS`时编译为空。断言在生产时免费，在开发时有信息。

**修复.** 无需修复。确认你的测试内核启用了`INVARIANTS`，断言将在被违反时触发。

### 症状：sysctl处理程序阻塞整个系统

**原因.** 跨慢操作持有锁的sysctl处理程序可以有效地序列化需要相同锁的其他每个操作。如果锁是设备的主互斥锁，每个I/O被阻塞直到sysctl返回。

**修复.** sysctl处理程序应该遵循与I/O处理程序相同的纪律：持有锁最短时间，在任何潜在慢操作前释放。快照并应用模式在这里同样有效。

### 症状：读取者在read_timeout_ms=0时仍然得到EAGAIN

**原因.** 读取返回`EAGAIN`是因为`O_NONBLOCK`（文件描述符以非阻塞方式打开，或`fcntl(2)`在其上设置了`O_NONBLOCK`）。驱动程序的`IO_NDELAY`检查不管超时sysctl如何都返回`EAGAIN`。

**修复.** 确认描述符是阻塞的：`fcntl(fd, F_GETFL)`应该返回没有`O_NONBLOCK`位的值。如果非阻塞是预期的，`EAGAIN`是正确的响应。

### 症状：成功测试后kldunload仍然短暂挂起

**原因.** detach路径正在等待飞行中的处理程序返回。每个在cv上睡眠的等待者必须唤醒（因为广播）、重新获取互斥锁、看到`!is_attached`、返回并退出内核。对于几个等待者这花费几毫秒。

**修复.** 通常无需修复；几毫秒的延迟是正常的。如果延迟更长，检查每个等待者是否确实有睡眠后`is_attached`检查。

### 症状：两个单独驱动程序实例都报告关于相同锁名称的WITNESS警告

**原因.** 两个实例都用相同名称初始化它们的锁（例如都是`myfirst0`）。`WITNESS`将相同名称的锁视为相同的逻辑锁，可能跨实例警告重复获取或发明的顺序问题。

**修复.** 用包含单元号的唯一名称初始化每个实例的锁，例如通过`device_get_nameunit(dev)`，它产生`myfirst0`、`myfirst1`等。我们的章节已经为设备互斥锁这样做了；对cv和sx也这样做。

### 症状：有多个等待者的cv花费很长时间广播

**原因.** `cv_broadcast`遍历等待队列，标记每个等待者为可运行。遍历在等待者数量上是O(n)。有数百个等待者时这成为可测量的成本。

**修复.** 广播本身对于正常工作负载很少是瓶颈；后续每个被唤醒线程尝试获取互锁时的惊群竞争导致可见暂停。如果你的驱动程序例行在同一个cv上有数百个等待者，重新考虑设计；每等待者cv或基于队列的方法可能扩展更好。



## 总结

第12章拿你在第11章构建的驱动程序并给了它更丰富的同步词汇表。第11章的单个互斥锁仍然在那里，做着相同的工作，有相同的规则。它周围现在有两个命名条件变量替换匿名唤醒通道，一个sx锁保护小但真实的配置子系统，以及一个让读取路径在用户期望时迅速返回的有界等待能力。两个锁类别之间的锁顺序被记录、由`WITNESS`强制执行，并由并发运行数据路径和配置路径的压力套件验证。

我们学会了将同步视为词汇表，不只是机制。条件变量说*我正在等待一个特定的变化；当它发生时告诉我*。共享锁说*我正在读取；不要让写入者进来*。定时等待说*如果花太长时间请放弃*。每个原语都是线程之间不同形式的协议，为每个协议使用正确的形状产生读起来像设计而不是与设计对抗的代码。

我们还学会了仔细调试同步。六种故障形状（丢失唤醒、虚假唤醒、锁顺序反转、过早销毁、持有不可睡眠锁时睡眠、detach与活动操作之间的竞争）涵盖了你在实践中将遇到的几乎每个bug。`WITNESS`在运行时捕获内核能检测的；内核内调试器让你检查挂起的系统；`lockstat(1)`和`dtrace(1)`给你不修改源代码的可见性。

我们以重构传递结束。驱动程序现在有记录的锁顺序、干净的`LOCKING.md`、升级的版本字符串、更新的变更日志，以及验证每个原语在每个支持的工作负载上的回归测试。那个基础设施扩展：当第13章添加定时器，第14章添加中断，文档模式吸收新原语而不会变得脆弱。

### 你现在应该能够做什么

在转向第13章之前，你应该拥有的能力的简短自我检查清单：

- 查看任何驱动程序中的共享状态并通过遍历决策树选择正确原语（原子、互斥锁、sx、rw、cv）。
- 用命名cv替换任何驱动程序中的任何匿名唤醒通道，并解释为什么那个更改是一种改进。
- 向任何等待路径添加有界阻塞原语，并解释何时使用`EAGAIN`、`EINTR`、`ERESTART`或`EWOULDBLOCK`。
- 设计带记录锁顺序的多读取者、单写入者子系统。
- 阅读WITNESS警告并仅从源位置识别违规锁对。
- 使用`show all locks`和`show sleepchain`在DDB中诊断挂起的系统。
- 运行复合压力工作负载并用`lockstat(1)`测量锁争用。
- 编写另一个开发者可以用作权威参考的`LOCKING.md`文档。

如果其中任何一项感觉不确定，第12章的实验是建立肌肉记忆的地方。没有一个需要超过几个小时；合在一起它们涵盖了本章引入的每个原语和每个模式。

### 三个结束提醒

第一个是*在提交前运行复合压力*。复合套件捕获单轴测试会错过的跨原语bug。在调试内核上三十分钟是为它产生的信心的小投资。

第二个是*保持锁顺序诚实*。你引入的每个新锁开始一个新问题：它在顺序中的位置在哪里？在编写代码之前在`LOCKING.md`中显式回答问题。得到错误答案的成本随驱动程序大小增长；在开始时写下它的成本是一分钟。

第三个是*信任原语并使用正确的一个*。内核的互斥锁、cv、sx和rw锁是几十年工程的结果。使用标志和原子标志推出自己的协调的诱惑是真实的，几乎总是误导的。选择命名你试图说什么的原语。代码会更短、更清晰、且可证明更正确。



## 参考：驱动程序的阶段进展

第12章在四个离散阶段演化驱动程序，每个都是`examples/part-03/ch12-synchronization-mechanisms/`下的自己的目录。进展镜像章节叙述；它让读者可以一次一个原语地构建驱动程序并看到每个添加贡献了什么。

### 阶段1：cv-channels

将匿名`&sc->cb`唤醒通道替换为两个命名条件变量（`data_cv`、`room_cv`）。等待助手使用`cv_wait_sig`代替`mtx_sleep`。信号通知者在与状态更改匹配的cv上使用`cv_signal`（或detach中的`cv_broadcast`）。

改变了什么：睡眠/唤醒机制。驱动程序从用户空间行为相同。

你可以验证什么：`procstat -kk`显示cv名称（`myfirst data`或`myfirst room`）而不是wmesg（`myfrd`）。`dtrace`可以附加到特定cv。吞吐量略高，因为每事件信号通知避免唤醒无关的等待者。

### 阶段2：bounded-read

添加一个通过`cv_timedwait_sig`限制阻塞读取的`read_timeout_ms` sysctl。对称的`write_timeout_ms`也是可能的。

改变了什么：读取路径现在可以在可配置超时后返回`EAGAIN`。零的默认值保留阶段1的无限期等待行为。

你可以验证什么：`timeout_tester`在大约配置的超时后报告`EAGAIN`。将超时设置为零恢复无限期等待。Ctrl-C在两种情况下仍然工作。

### 阶段3：sx-config

向softc添加一个`cfg`结构，由`sx_lock`（`cfg_sx`）保护。三个配置字段（`debug_level`、`soft_byte_limit`、`nickname`）作为sysctl公开。数据路径为日志发射咨询`debug_level`，为写入拒绝咨询`soft_byte_limit`。

改变了什么：驱动程序获得配置接口。`MYFIRST_DBG`宏咨询当前调试级别。会超过软限制的写入返回`EAGAIN`。

你可以验证什么：`sysctl -w dev.myfirst.0.debug_level=2`产生可见调试消息。设置`soft_byte_limit`导致写入在缓冲区达到限制后开始失败。`WITNESS`报告锁顺序（互斥锁第一，sx第二）并在压力下静默。

### 阶段4：final

带有所有三个原语的组合版本，加上`LOCKING.md`更新、到`0.6-sync`的版本升级，以及使用两个锁在一起的新`myfirst_sysctl_reset`。

改变了什么：集成。没有新原语。

你可以验证什么：回归套件通过；复合压力工作负载干净运行至少30分钟；`clang --analyze`静默。

这个四阶段进展是规范的第12章驱动程序。配套示例完全镜像阶段，以便读者可以编译和加载其中任何一个。



## 参考：从mtx_sleep迁移到cv

如果你正在处理一个使用遗留`mtx_sleep`/`wakeup`通道机制的现有驱动程序，迁移到`cv(9)`是机械的。一个简短的配方。

开始前的说明：遗留机制未被弃用，仍在FreeBSD树中广泛使用。许多驱动程序将无限期保持`mtx_sleep`，那完全正确。当多个不同条件共享单个通道时（惊群情况），或当你想要命名cv提供的`procstat`和`dtrace`可见性时，迁移是值得的。对于有单个条件和单个通道的驱动程序，迁移纯粹是美学的；如果你想要就为可读性做，如果不想就跳过。

### 步骤1：识别每个逻辑等待通道

阅读源代码。找到每个`mtx_sleep`调用。对于每个，问：这个线程在等待什么条件？

在第11章驱动程序中，有两个逻辑条件都使用`&sc->cb`：

- `myfirst_wait_data`：等待`cbuf_used > 0`。
- `myfirst_wait_room`：等待`cbuf_free > 0`。

两个条件；一个通道。迁移为每个分配自己的cv。

### 步骤2：向Softc添加cv字段

对于每个逻辑条件，添加一个`struct cv`字段。选择描述性名称：

```c
struct cv  data_cv;
struct cv  room_cv;
```

在attach中初始化（`cv_init`）并在detach中销毁（`cv_destroy`）。

### 步骤3：用cv_wait_sig替换mtx_sleep

对于每个`mtx_sleep`调用，用`cv_wait_sig`（或`cv_timedwait_sig`）替换：

```c
/* 之前： */
error = mtx_sleep(&sc->cb, &sc->mtx, PCATCH, "myfrd", 0);

/* 之后： */
error = cv_wait_sig(&sc->data_cv, &sc->mtx);
```

wmesg参数消失了（cv的描述字符串取代它）。`PCATCH`隐含在`_sig`后缀中。互锁参数相同。

### 步骤4：用cv_signal或cv_broadcast替换wakeup

对于每个`wakeup(&channel)`调用，决定应该唤醒一个等待者还是所有等待者。用`cv_signal`或`cv_broadcast`替换：

```c
/* 之前： */
wakeup(&sc->cb);  /* 读取者和写入者都在这个通道上 */

/* 之后： */
if (write_succeeded)
        cv_signal(&sc->data_cv);  /* 只有读取者关心新数据 */
if (read_succeeded)
        cv_signal(&sc->room_cv);  /* 只有写入者关心新空间 */
```

这也是添加cv API鼓励的每事件对应关系的时刻：只在状态实际更改时信号通知。

### 步骤5：更新detach路径

detach曾经在销毁状态前唤醒通道：

```c
/* 之前： */
sc->is_attached = 0;
wakeup(&sc->cb);

/* 之后： */
sc->is_attached = 0;
cv_broadcast(&sc->data_cv);
cv_broadcast(&sc->room_cv);
/* 稍后，所有等待者退出后： */
cv_destroy(&sc->data_cv);
cv_destroy(&sc->room_cv);
```

`cv_broadcast`确保每个等待者唤醒；等待后`is_attached`检查为每个返回`ENXIO`。

### 步骤6：更新LOCKING.md

记录每个新cv：其名称、其条件、其互锁、其信号通知者、其等待者。第12章驱动程序的`LOCKING.md`是模板。

### 步骤7：重新运行压力套件

迁移不应该改变可观察行为；只是内部机制。运行现有测试；它们应该通过。在`WITNESS`下运行；不应该出现新警告。

我们章节中的迁移是几百行源代码；上面的配方扩展到任何大小的驱动程序。好处是每个等待现在在`procstat`、`dtrace`和源代码中都有可见的名称。

### 迁移何时不值得

迁移花费几小时的重构努力、仔细重新运行测试套件和文档更新。好处是：

- 每个等待获得在`procstat`、`dtrace`和源代码中可见的名称。
- 唤醒变成每条件；惊群缩小。
- 唤醒通道不匹配在源代码中更容易发现。

对于有单个等待条件的小驱动程序，成本和好处大致抵消；遗留机制没问题。对于有两个或更多不同条件的驱动程序，迁移几乎总是值得的。对于由多个开发人员维护的驱动程序，可读性收益大。



## 参考：生产前审计检查清单

在将重度同步驱动程序从开发提升到生产之前执行的简短审计。每一项是一个问题；每一项应该可以自信地回答。

### 锁清单

- [ ] 我是否在`LOCKING.md`中列出了驱动程序拥有的每个锁？
- [ ] 对于每个锁，我是否命名了它保护什么？
- [ ] 对于每个锁，我是否命名了可以在其中获取它的上下文？
- [ ] 对于每个锁，我是否记录了其生命周期（在哪里创建、在哪里销毁）？

### 锁顺序

- [ ] 全局锁顺序是否记录在`LOCKING.md`中？
- [ ] 持有两个锁的每个代码路径是否遵循全局顺序？
- [ ] 我是否在压力下运行`WITNESS`至少30分钟并观察到没有顺序反转？
- [ ] 如果驱动程序有多个实例，我是否确认实例内排序与实例间排序一致？

### cv清单

- [ ] 我是否列出了驱动程序拥有的每个cv？
- [ ] 对于每个cv，我是否命名了它代表的条件？
- [ ] 对于每个cv，我是否命名了互锁互斥锁？
- [ ] 对于每个cv，我是否确认至少一个信号通知者和至少一个等待者？
- [ ] 对于每个cv，我是否确认detach中`cv_destroy`之前调用了`cv_broadcast`？

### 等待助手

- [ ] 每个`cv_wait`（或变体）是否在`while (!condition)`循环内？
- [ ] 每个等待助手是否在等待后重新检查`is_attached`？
- [ ] 每个等待助手是否返回合理的错误（`ENXIO`、`EINTR`、`EAGAIN`）？

### 信号通知站点

- [ ] 每个应该唤醒等待者的状态更改是否有相应的`cv_signal`或`cv_broadcast`？
- [ ] 只有一个等待者需要唤醒时是否使用`cv_signal`；所有都需要时才使用`cv_broadcast`？
- [ ] 信号通知站点是否用`if (state_changed)`保护以便跳过空信号？

### detach路径

- [ ] detach是否在`active_fhs > 0`时拒绝继续？
- [ ] detach是否在设备互斥锁下清除`is_attached`？
- [ ] detach是否在销毁前广播每个cv？
- [ ] 原语是否以相反获取顺序销毁（最内层锁先销毁）？

### 静态分析

- [ ] 是否运行了`clang --analyze`；新警告已分类？
- [ ] `WARNS=6`构建是否没有产生警告？
- [ ] 回归套件是否在`WITNESS`内核上运行；所有测试通过？

### 文档

- [ ] `LOCKING.md`是否与代码保持同步？
- [ ] 源代码中的版本字符串是否升级？
- [ ] `CHANGELOG.md`是否更新？
- [ ] `README.md`是否描述新功能及其sysctl？

通过此审计的驱动程序是一个你可以在负载下信任的驱动程序。



## 参考：睡眠通道卫生

遗留`mtx_sleep`/`wakeup`通道机制和现代`cv(9)` API都依赖于标识唤醒影响哪些等待者的*通道*。通道是内核睡眠队列哈希表的键。通道周围的错误是几个常见bug的来源。

几条卫生规则。

### 每个逻辑条件一个通道

如果你的驱动程序有两个不同的阻塞条件（比如，"数据可用"和"空间可用"），使用两个不同的通道。共享单个通道强制每次唤醒唤醒所有等待者；其中一些会立即回到睡眠，因为它们的条件仍然为假。性能成本是真实的；可读性成本也是真实的。

在我们章节中，这条规则体现为`data_cv`和`room_cv`是分开的cv。第11章驱动程序使用共享匿名通道`&sc->cb`并付出了惊群代价；第12章的拆分是治疗。

### 通道指针必须稳定

通道是一个地址。内核不解释它；它将其用作哈希键。地址在等待和信号之间不能改变。这通常自动发生（softc字段的地址在softc生命周期内稳定），但对临时缓冲区、栈分配结构或释放的内存要小心。

如果你看到在特定代码路径后挂起的等待，怀疑通道指针不匹配。信号通知者和等待者必须使用相同的地址。

### 通道指针必须对其用途唯一

如果同一地址用于两个不同目的（比如，数据可用通道和"完成"通道），一个目的的唤醒可能无意中唤醒另一个目的的等待者。为每个目的使用softc的不同字段作为通道，或使用cv（它有名称，是单独的对象）。

### 状态可能更改时唤醒者应该持有互锁

尽管`wakeup`和`cv_signal`不严格要求在互锁下调用，但这样做关闭了一个竞争窗口，其中等待者检查条件（假）、状态更改、唤醒触发（队列中无等待者），然后等待者入队并永远睡眠。在信号通知时持有互锁是安全的默认值；只在你证明状态不能恢复时放宽。

我们的第12章设计在释放互斥锁后信号通知，这是安全的，因为cv自己的互锁下入队契约关闭了cv的竞争（不是`wakeup`的）。对于`wakeup`，在互斥锁下信号通知。

### 无等待者的信号免费

无等待者通道上的`cv_signal`和`wakeup`什么都不做。不必要信号没有惩罚；成本本质上是获取和释放cv队列自旋锁的成本。不要出于优化恐惧避免信号；在状态更改时信号通知，即使它有时信号通知空。

### 无信号通知者的等待是Bug

从未被信号通知的等待是挂起。确保每个等待至少有一个匹配信号通知站点，匹配站点在产生等待状态的每个代码路径中都被到达。

这是最常见的cv bug。审计检查清单提出问题；代码审查期间提问的纪律捕获大多数情况。



## 参考：常见cv习语

你将最常使用的cv模式的快速查找集合。

### 等待条件

```c
mtx_lock(&mtx);
while (!condition)
        cv_wait_sig(&cv, &mtx);
/* 条件为真；做工作 */
mtx_unlock(&mtx);
```

`while`循环是必不可少的。允许虚假唤醒；信号中断等待；两者看起来都像从`cv_wait_sig`返回。每次返回后重新检查条件。

### 信号通知一个等待者

```c
mtx_lock(&mtx);
/* 更改状态 */
mtx_unlock(&mtx);
cv_signal(&cv);
```

解锁后`cv_signal`节省上下文切换（被唤醒线程不立即争用互斥锁）。当状态更改无歧义且没有并发路径可以恢复时可接受。

### 广播状态更改

```c
mtx_lock(&mtx);
state_changed_globally = true;
cv_broadcast(&cv);
mtx_unlock(&mtx);
```

当每个等待者需要知道更改时使用`cv_broadcast`。detach路径和配置重置是典型示例。

### 带超时等待

```c
mtx_lock(&mtx);
while (!condition) {
        int ticks = (ms * hz + 999) / 1000;
        int err = cv_timedwait_sig(&cv, &mtx, ticks);
        if (err == EWOULDBLOCK) {
                mtx_unlock(&mtx);
                return (EAGAIN);
        }
        if (err != 0) {
                mtx_unlock(&mtx);
                return (err);
        }
}
/* 做工作 */
mtx_unlock(&mtx);
```

将毫秒转换为tick，向上取整，显式处理三种返回情况（超时、信号、正常唤醒）。

### 带detach感知等待

```c
while (!condition) {
        int err = cv_wait_sig(&cv, &mtx);
        if (err != 0)
                return (err);
        if (!sc->is_attached)
                return (ENXIO);
}
```

等待后`is_attached`检查确保如果设备在我们睡眠时被分离我们干净退出。detach路径中的`cv_broadcast`使这工作。

### 销毁前排空等待者

```c
mtx_lock(&mtx);
sc->is_attached = 0;
cv_broadcast(&cv);
mtx_unlock(&mtx);
/* 等待者唤醒，看到!is_attached，返回ENXIO，退出 */
/* 到active_fhs == 0时，无等待者剩余 */
cv_destroy(&cv);
```

广播和`is_attached`重新检查的组合保证销毁时cv中没有等待者。



## 参考：常见sx习语

### 读多字段

```c
sx_slock(&sx);
value = field;
sx_sunlock(&sx);
```

在多核系统上便宜；多个读取者不争用。

### 带验证更新

```c
sx_slock(&sx);
old = field;
sx_sunlock(&sx);

/* 用sysctl_handle_*等验证可能的新值 */

sx_xlock(&sx);
field = new;
sx_xunlock(&sx);
```

两次获取-释放循环。读取用共享锁；写入用独占锁。在它们之间释放以便验证不持有任何一个锁。

### 跨两个锁的快照并应用

```c
sx_slock(&cfg_sx);
local = cfg.value;
sx_sunlock(&cfg_sx);

mtx_lock(&data_mtx);
/* 使用local而不一起持有任何一个锁 */
mtx_unlock(&data_mtx);
```

避免同时持有两个锁；放宽锁顺序约束。

### Try-Upgrade模式

```c
sx_slock(&sx);
if (need_modify) {
        if (sx_try_upgrade(&sx)) {
                /* 独占 */
                modify();
                sx_downgrade(&sx);
        } else {
                /* 释放，作为独占重新获取，重新验证 */
                sx_sunlock(&sx);
                sx_xlock(&sx);
                if (still_need_modify())
                        modify();
                sx_downgrade(&sx);
        }
}
sx_sunlock(&sx);
```

乐观升级。回退路径必须重新验证，因为世界在解锁窗口期间改变了。

### 断言持有

```c
sx_assert(&sx, SA_SLOCKED);  /* 共享 */
sx_assert(&sx, SA_XLOCKED);  /* 独占 */
sx_assert(&sx, SA_LOCKED);   /* 任一 */
```

在期望特定锁状态的助手的开头使用。



## 参考：同步原语决策表

紧凑查找表。

| 如果你需要... | 使用 |
|---|---|
| 原子更新单个字 | `atomic(9)` |
| 廉价更新每CPU计数器 | `counter(9)` |
| 在进程上下文中保护复合状态 | `mtx(9)` (`MTX_DEF`) |
| 在中断上下文中保护复合状态 | `mtx(9)` (`MTX_SPIN`) |
| 在进程上下文中保护读多状态 | `sx(9)` |
| 在睡眠被禁止处保护读多状态 | `rw(9)` |
| 等待特定条件变为真 | `cv(9)` (与`mtx(9)`或`sx(9)`配对) |
| 等待到截止期限 | `cv_timedwait_sig`，或`timo` > 0的`mtx_sleep` |
| Ctrl-C可中断的等待 | 任何等待原语的`_sig`变体 |
| 在未来特定时间运行代码 | `callout(9)` (第13章) |
| 推迟工作到工作者线程 | `taskqueue(9)` (第16章) |
| 完全无同步地并发读取 | `epoch(9)` (后面章节) |

如果两个原语都适合，使用更简单的那个。



## 参考：阅读kern_condvar.c和kern_sx.c

`/usr/src/sys/kern/`中的两个文件值得在你已经在驱动程序中使用cv和sx API后打开。

`/usr/src/sys/kern/kern_condvar.c`是cv实现。值得看的函数：

- `cv_init`：初始化。琐碎。
- `_cv_wait`和`_cv_wait_sig`：核心阻塞原语。每个获取cv队列自旋锁、递增等待者计数、释放互锁、将线程交给睡眠队列、让出，返回时重新获取互锁。"释放互锁、睡眠"的原子性由睡眠队列层提供。
- `_cv_timedwait_sbt`和`_cv_timedwait_sig_sbt`：定时变体。相同形状，带一个如果超时先触发则唤醒线程的callout。
- `cv_signal`：获取cv队列自旋锁，通过`sleepq_signal`信号通知一个等待者。
- `cv_broadcastpri`：以给定优先级信号通知所有等待者。

整个文件约400行。一下午阅读绰绰有余地理解它端到端。

`/usr/src/sys/kern/kern_sx.c`是sx实现。更大更密集，因为锁支持带完整优先级传播的共享和独占模式。值得看的函数：

- `sx_init_flags`：初始化。设置初始状态，向`WITNESS`注册。
- `_sx_xlock_hard`和`_sx_xunlock_hard`：独占操作的慢路径。快速路径在`sx.h`中内联。
- `_sx_slock_int`和`_sx_sunlock_int`：共享模式操作。共享计数通过原子比较并交换递增；如果锁被独占持有，线程阻塞。
- `sx_try_upgrade_int`和`sx_downgrade_int`：模式更改操作。

浏览。内部错综复杂，但面向公众的API行为如文档所述，源代码确认它。



## 参考：cv和sx常见错误

每个新原语都有一组初学者在被咬之前会犯的错误。简短目录。

### cv错误

**在`cv_wait`周围使用`if`而不是`while`。** 由于虚假唤醒，返回时条件可能不为真。总是循环。

**在detach中忘记广播。** 等待者永不唤醒，销毁时cv有残留等待者，内核可能panic。总是在`cv_destroy`之前`cv_broadcast`。

**信号通知错误的cv。** 当你意思是读取者时唤醒写入者（或反之）。重构时容易犯错。cv的名称是你的防御；如果`cv_signal(&sc->room_cv)`在调用点感觉不对，它可能就不对。

**当状态可能恢复时没有互锁信号通知。** 如果两个线程都可以修改状态，其中一个必须在信号通知时持有互锁，否则唤醒可能丢失。默认在互锁下信号通知；只在你证明状态不能恢复时放宽。

**错过等待后detach检查。** 由于detach中`cv_broadcast`而唤醒的等待者必须重新检查`is_attached`并返回`ENXIO`。如果检查缺失，等待者在设备不再存活时继续并崩溃。

**在持有多个锁时调用cv_wait。** 睡眠期间只释放互锁。其他锁保持持有。如果唤醒者需要那些锁，你有死锁。先释放其他锁。

### sx错误

**跨睡眠调用持有sx。** 在`sysctl_handle_*`、`uiomove`或`malloc(M_WAITOK)`之前释放。sx是可睡眠的，所以内核不会panic，但其他等待者会在持续时间被阻塞。

**共享获取然后xlock而不释放共享。** 在以共享模式持有同一sx时调用`sx_xlock`是死锁；调用将永远阻塞等待自己。使用`sx_try_upgrade`或释放并重新获取。

**忘记sx是可睡眠的。** 从睡眠非法的上下文（中断上下文、自旋锁内）调用`sx_xlock`会panic。对那些上下文使用`rw(9)`。

**跨长操作以共享模式持有sx。** 其他读取者可以继续，但sx写入者被无限期阻塞。如果操作长，释放共享锁，做工作，如果需要提交则重新获取。

**释放错误的模式。** 共享模式锁上`sx_xunlock`是bug；独占模式锁上`sx_sunlock`是bug。只在不知道你在哪种模式时使用`sx_unlock`（多态版本）（罕见）。

### 两者组合特有的错误

**以错误顺序获取。** 第12章驱动程序要求互斥锁第一，sx第二。反向顺序在负载下产生`WITNESS`警告。

**以错误顺序释放。** 获取mtx、获取sx、释放mtx、释放sx。释放顺序*必须*是获取顺序的反向：先释放sx，然后释放mtx。否则两个释放之间的观察者看到意外组合。

**在陈旧重要处快照并应用。** 该模式只在快照可以容忍小陈旧时正确。对必须是最新的值（安全标志、硬配额限制），快照并应用是错误的；你必须原子持有两个锁。

**忘记更新LOCKING.md。** 在不更新文档的情况下添加锁或更改顺序产生漂移。三个月后，没人记得规则是什么。在同一提交中更新文档。



## 参考：时间原语

内核如何表达时间的简短游览。在阅读或编写定时等待变体时有用。

内核有三个常用时间表示：

- `int` tick。遗留单位。`hz` tick等于一秒。FreeBSD 14.3上默认`hz`是1000，所以一个tick是一毫秒。`mtx_sleep`、`cv_timedwait`和`tsleep`都以tick接受超时。
- `sbintime_t`。64位有符号二进制定点表示：高32位是秒，低32位是秒的小数部分。单位常量在`/usr/src/sys/sys/time.h`中：`SBT_1S`、`SBT_1MS`、`SBT_1US`、`SBT_1NS`。较新的时间API（`msleep_sbt`、`cv_timedwait_sbt`、`callout_reset_sbt`）使用sbintime。
- `struct timespec`。POSIX秒和纳秒。用于用户空间边界；驱动程序内部很少需要。

`time.h`中的转换助手：

- `tick_sbt`：一个全局变量持有`1 / hz`作为sbintime，所以`tick_sbt * timo_in_ticks`给出等效sbintime。
- `nstosbt(ns)`、`ustosbt(us)`、`sbttous(sbt)`、`sbttons(sbt)`、`tstosbt(ts)`、`sbttots(ts)`：各种单位之间的显式转换。

`_sbt`时间API存在是因为`hz`粒度对某些用途太粗。有`hz=1000`，最小可表达超时是1 ms，超时对齐到tick边界。有sbintime，你可以表达100微秒并要求内核尽可能接近硬件定时器允许地调度唤醒。

对于第12章，我们在任何地方都使用基于tick的API，因为精度足够。参考在这里以便当亚毫秒精度重要时你知道去哪里找。

`_sbt`函数的`pr`参数值得一句话。它是调用者愿意接受的*精度*：内核可以为省电定时器合并添加多少摆动。`SBT_1S`的精度意味着"我不在乎我的5秒定时器是否晚到1秒触发；如果你可以将其与另一个定时器合并省电，请这样做"。`SBT_1NS`的精度意味着"尽可能接近截止期限触发"。对于驱动程序代码，`0`（无容限）或`SBT_1MS`（一毫秒容限）是典型值。

`flags`参数控制定时器如何注册。`C_HARDCLOCK`是最常见的：对齐到系统的hardclock中断以获得可预测的时序。`C_DIRECT_EXEC`在定时器中断中运行callout，而不是将其推迟到callout线程。`C_ABSOLUTE`将`sbt`解释为绝对时间而不是相对超时。我们在第12章中任何地方都使用`C_HARDCLOCK`。



## 参考：常见WITNESS警告解码

`WITNESS`产生几类警告。每类都有可识别的形状。

### "lock order reversal"

签名：命名"1st"和"2nd"锁的两行，加上"established at"行。我们在第6节中演练了诊断。

常见原因：以与先前观察顺序矛盾的顺序获取锁的路径。通过重新排序或重构修复。

### "duplicate lock of same name"

签名：关于获取与已持有锁相同`lo_name`的锁的警告。

常见原因：同一驱动程序的两个实例，每个有自己的锁，都有相同的名称。`WITNESS`保守假设相同类型的两个锁属于同一类别。通过用唯一名称初始化每个锁修复（例如，通过`device_get_nameunit(dev)`包含单元号），或在初始化时传递适当的每类"重复获取OK"标志：互斥锁用`MTX_DUPOK`，sx用`SX_DUPOK`，rwlock用`RW_DUPOK`。这些都展开到锁对象级别的`LO_DUPOK`位；你在驱动程序代码中写每类名称。

### "sleeping thread (pid N) owns a non-sleepable lock"

签名：线程在睡眠原语（`cv_wait`、`mtx_sleep`、`_sleep`）中，同时持有自旋互斥锁或rw锁。

常见原因：获取不可睡眠锁然后调用可能睡眠的东西的函数。先释放不可睡眠锁修复。

### "exclusive sleep mutex foo not owned at"

签名：线程尝试释放或断言它不持有的互斥锁。

常见原因：错误的互斥锁指针，或此代码路径上没有匹配锁定的解锁。通过追踪锁获取修复。

### "lock list reversal"

签名：类似于锁顺序反转但指示涉及两个以上锁的更复杂反转。

常见原因：一起违反全局顺序的获取链。通过简化获取模式修复；如果链真的必要，考虑设计是否应该使用更少的锁。

### "sleepable acquired while holding non-sleepable"

签名：线程尝试获取可睡眠锁（sx、mtx_def、lockmgr）同时持有不可睡眠的（mtx_spin、rw）。

常见原因：关于锁类别的混淆。通过将内部锁切换到可睡眠变体或重构以避免嵌套修复。

### 对警告采取行动

当`WITNESS`触发时，诱惑是抑制警告。抵制。警告意味着内核观察到了违反真实规则的真实情况。抑制隐藏bug；不修复它。

正确的响应，按偏好顺序：

1. 修复bug（重新排序锁、释放锁、重构代码）。
2. 解释为什么警告对此情况不正确并在源代码中使用适当的`_DUPOK`标志并带注释。
3. 如果两个都做不到，升级。在freebsd-hackers上问或开PR。没人能解释的`WITNESS`警告是某处的真实bug。



## 参考：锁类快速参考

到目前为止你见过的锁类别之间差异的紧凑查找。

| 属性 | `mtx_def` | `mtx_spin` | `sx` | `rw` | `rmlock` | `lockmgr` |
|---|---|---|---|---|---|---|
| 争用时睡眠 | 是 | 否（自旋） | 是 | 否（自旋） | 否（大多数） | 是 |
| 多个持有者 | 否 | 否 | 是（共享） | 是（读） | 是（读） | 是（共享） |
| 持有者可睡眠 | 是 | 否 | 是 | 否 | 否（读） | 是 |
| 优先级传播 | 是 | n/a | 否 | 是 | n/a | 是 |
| 信号可中断 | n/a | n/a | `_sig` | 否 | 否 | 是 |
| 支持递归 | 可选 | 是 | 可选 | 否 | 否 | 是 |
| WITNESS跟踪 | 是 | 是 | 是 | 是 | 是 | 是 |
| 最佳驱动程序用途 | 默认 | 中断上下文 | 读多 | 热读取路径 | 极热读取 | 文件系统 |

`rmlock(9)`和`lockmgr(9)`列出是为了完整性；本书深入涵盖`mtx`、`cv`、`sx`和`rw`，并将其他视为"已知存在，如果需要查阅手册页"。



## 参考：多原语驱动程序设计模式

三种模式在组合多个同步原语的驱动程序中重复出现。每个都值得一句话，以便你在野外认出它们。

### 模式：一个互斥锁，一个配置sx

第12章的`myfirst`驱动程序是这个模式。互斥锁保护数据路径，sx保护配置。锁顺序是互斥锁第一，sx第二。大多数简单驱动程序适合这个模式。

何时使用：数据路径是进程上下文、有复合不变量、偶尔读取配置。配置频繁读取、很少写入。

何时不使用：当数据路径在中断上下文中运行（互斥锁必须是`MTX_SPIN`，配置必须是`rw`），或当数据路径本身有受益于不同锁的子路径时。

### 模式：每队列锁配配置锁

有多个队列（每CPU一个、每消费者一个、每流一个）的驱动程序给每个队列自己的锁并使用单独的sx用于配置。锁顺序是每队列锁第一，配置sx第二。队列之间未定义锁顺序（你不应该一次持有两个队列）。

何时使用：高核心数，工作负载自然每队列分区。

何时不使用：工作负载对称且每队列锁没有帮助，或数据频繁跨越队列会强制顺序规则。

### 模式：每对象锁配容器锁

维护对象列表（设备、会话、描述符）的驱动程序给每个对象自己的锁并使用容器锁保护对象列表。遍历列表获取容器锁；修改对象获取那个对象的锁；两者都可以以容器第一、对象第二的顺序持有。

何时使用：列表操作和每对象操作都需要保护，有不同的生命周期。

何时不使用：单个互斥锁足够（小列表、不频繁操作）。

`myfirst`驱动程序还不需要这个模式；本书后面的驱动程序会。

### 模式：每CPU计数器配互斥保护尾部

这是第11章模式，第12章继承。热计数器（bytes_read、bytes_written）使用`counter(9)`每CPU存储。有复合不变量的cbuf使用单个互斥锁。两者独立；计数器更新不需要互斥锁；cbuf更新仍然需要。

何时使用：高频计数器位于有复合不变量的结构旁边。

何时不使用：计数器更新需要与结构更新一致（然后两者都需要同一个锁）。

### 模式：跨两个锁类别的快照并应用

任何时候路径需要两个锁类别，快照并应用模式将锁顺序约束减少到单个方向。从一个锁读取、释放、然后获取另一个。快照可能略微过时；对于建议性值，那是可接受的。

何时使用：被快照的值不是严格当前要求的；微秒级的陈旧可接受。

何时不使用：值是安全标志、硬预算限制或任何陈旧可能违反契约的东西。

`myfirst_write`路径对软字节限制使用这个：在cfg sx下快照限制、释放、获取数据互斥锁、检查限制对当前`cb_used`。组合操作不是原子的，但它是正确的，因为任何竞争导致错误答案是可容忍的错误答案（拒绝本可以放入的写入，或接受刚刚溢出的写入；两者都可恢复）。



## 参考：每个原语的先决条件

每个原语都有关于何时以及如何使用它的规则。违反规则是bug；规则在此列出以便快速参考。

### mtx(9) (MTX_DEF)

`mtx_init`的先决条件：
- `struct mtx`的内存存在且未被别名化。
- 互斥锁尚未初始化。

`mtx_lock`的先决条件：
- 互斥锁已初始化。
- 调用线程在进程上下文、内核线程上下文或callout-mpsafe上下文中。
- 调用线程尚未持有互斥锁（除非`MTX_RECURSE`）。
- 调用线程不持有任何自旋互斥锁。

`mtx_unlock`的先决条件：
- 调用线程持有互斥锁。

`mtx_destroy`的先决条件：
- 互斥锁已初始化。
- 没有线程持有互斥锁。
- 没有线程阻塞在互斥锁上。

### cv(9)

`cv_init`的先决条件：
- `struct cv`的内存存在。
- cv尚未初始化。

`cv_wait`和`cv_wait_sig`的先决条件：
- 互锁互斥锁被调用线程持有。
- 调用线程在睡眠合法的上下文中。

`cv_signal`和`cv_broadcast`的先决条件：
- cv已初始化。
- 惯例：互锁互斥锁被调用线程持有（非API严格要求，但是防御性的）。

`cv_destroy`的先决条件：
- cv已初始化。
- 没有线程阻塞在cv上（等待队列必须为空）。

### sx(9)

`sx_init`的先决条件：
- `struct sx`的内存存在。
- sx尚未初始化（除非使用`SX_NEW`）。

`sx_xlock`和`sx_xlock_sig`的先决条件：
- sx已初始化。
- 调用线程尚未独占持有sx（除非`SX_RECURSE`）。
- 调用线程在睡眠合法的上下文中。
- 调用线程不持有任何不可睡眠锁（无自旋互斥锁，无rw锁）。

`sx_slock`和`sx_slock_sig`的先决条件：
- 与`sx_xlock`相同，除了递归检查适用于共享模式。

`sx_xunlock`和`sx_sunlock`的先决条件：
- 调用线程以相应模式持有sx。

`sx_destroy`的先决条件：
- sx已初始化。
- 没有线程以任何模式持有sx。
- 没有线程阻塞在sx上。

### rw(9)

`rw_init`、`rw_destroy`的先决条件：与`sx_init`、`sx_destroy`形状相同。

`rw_wlock`和`rw_rlock`的先决条件：
- rw已初始化。
- 调用线程当前未以冲突模式持有rw。
- 调用线程*不*需要在可睡眠上下文中。rw锁本身不睡眠；然而，持有rw时调用线程*绝不能*调用任何可能睡眠的函数。

`rw_wunlock`和`rw_runlock`的先决条件：
- 调用线程以相应模式持有rw。

遵循这些先决条件是在`WITNESS`下干净运行多年的驱动程序与在第一个不常见代码路径上产生意外panic的驱动程序之间的区别。



## 参考：第12章自我评估

在转向第13章之前使用此评分标准确认你已内化第12章材料。每个问题应该可以在不重读章节的情况下回答。

### 概念问题

1. **命名三种主要的同步形式。** 互斥、带限制写入的共享访问、协调等待。

2. **为什么条件变量比匿名唤醒通道更可取？** 每个cv代表一个逻辑条件；信号不唤醒无关等待者；cv有在`procstat`和`dtrace`中可见的名称；API通过其类型强制原子释放并睡眠契约。

3. **cv_signal和cv_broadcast有什么区别？** `cv_signal`唤醒一个等待者（最高优先级，相等中FIFO）；`cv_broadcast`唤醒所有等待者。对每事件状态更改使用信号；对全局更改使用广播（detach、重置）。

4. **cv_wait_sig被信号中断时返回什么？** `EINTR`或`ERESTART`，取决于信号的重启处置。驱动程序直接传递值。

5. **sx和rw锁有什么区别？** `sx(9)`是可睡眠的；`rw(9)`不是。在临界区可能包括睡眠调用的进程上下文中使用`sx`；在临界区可能在中断上下文中运行或绝不能睡眠时使用`rw`。

6. **为什么sx_try_upgrade存在而不是无条件的sx_upgrade？** 因为两个同时持有者都尝试无条件升级会死锁。`try`变体当另一个共享持有者存在时返回失败，让调用者干净退避。

7. **什么是快照并应用模式，为什么有用？** 获取一个锁，将所需值读入局部变量，释放；然后获取不同的锁并使用局部值。避免同时持有两个锁，放宽锁顺序约束。当快照可以容忍小陈旧时可接受。

8. **第12章驱动程序中规范锁顺序是什么？** sc->mtx在sc->cfg_sx之前。记录在`LOCKING.md`中；由`WITNESS`强制执行。

### 代码阅读问题

打开你的第12章驱动程序源代码并验证：

1. 每个`cv_wait_sig`在`while (!condition)`循环内。
2. 每个cv至少有一个信号通知者和一个广播调用者（detach中的广播可接受）。
3. 每个`sx_xlock`在每个代码路径（包括错误返回）上都有匹配的`sx_xunlock`。
4. detach路径在销毁任何原语之前广播每个cv。
5. cfg sx在任何潜在睡眠调用（`sysctl_handle_*`、`uiomove`、`malloc(M_WAITOK)`）之前释放。
6. 锁顺序规则（互斥锁第一，sx第二）在持有两者的每个路径中遵循。

### 动手问题

1. 在`WITNESS`内核上加载第12章驱动程序并运行30分钟复合压力。有任何警告吗？如果是，调查。

2. 将`read_timeout_ms`设置为100并对空闲设备运行`read(2)`。调用返回什么？多久之后？

3. 在`mp_stress`运行时用`sysctl -w`在0和3之间切换`debug_level`。级别是否迅速生效？有什么东西坏了吗？

4. 使用`lockstat(1)`在配置写入者重度工作负载下测量sx锁上的争用。等待时间是多少？

5. 打开kern_condvar.c源代码并找到函数`cv_signal`。阅读它。你能用两句话描述它做什么吗？

如果五个动手问题都通过且概念问题容易，你的第12章工作是扎实的。



## 参考：同步进一步阅读

对于想要深入本章涵盖内容的读者：

### 手册页

- `mutex(9)`：互斥锁API（第11章完整涵盖；为完整性在此参考）。
- `condvar(9)`：条件变量API。
- `sx(9)`：共享/独占锁API。
- `rwlock(9)`：读/写锁API。
- `rmlock(9)`：读多锁API（高级）。
- `sema(9)`：计数信号量API（高级）。
- `epoch(9)`：延迟回收读多框架（高级；与网络驱动程序相关）。
- `locking(9)`：FreeBSD锁定原语概述。
- `lock(9)`：公共锁对象基础设施。
- `witness(4)`：WITNESS锁顺序检查器（第11章涵盖；本章重访）。
- `lockstat(1)`：锁分析用户空间工具。
- `dtrace(1)`：动态追踪框架，第15章更深入涵盖。

### 源文件

- `/usr/src/sys/kern/kern_condvar.c`：cv实现。
- `/usr/src/sys/kern/kern_sx.c`：sx实现。
- `/usr/src/sys/kern/kern_rwlock.c`：rw实现。
- `/usr/src/sys/kern/subr_sleepqueue.c`：cv和其他睡眠原语底层的睡眠队列机制。
- `/usr/src/sys/kern/subr_turnstile.c`：rw和其他优先级传播原语底层的turnstile机制。
- `/usr/src/sys/sys/condvar.h`、`/usr/src/sys/sys/sx.h`、`/usr/src/sys/sys/rwlock.h`：公共API头文件。
- `/usr/src/sys/sys/_lock.h`、`/usr/src/sys/sys/lock.h`：公共锁对象结构和类别注册。

### 外部材料

对于适用于任何操作系统的并发理论，Herlihy和Shavit的*The Art of Multiprocessor Programming*非常出色。对于FreeBSD特定的内核内部，McKusick等人的*The Design and Implementation of the FreeBSD Operating System*仍然是规范的教科书；关于锁定和调度的章节特别相关。

本书不要求阅读这两本书。当更深入学习时机到来时两者都有用。



## 展望未来：通往第13章的桥梁

第13章标题为*定时器和延迟工作*。其范围是从驱动程序角度看到的内核时间基础设施：如何在未来某个时间调度回调、如何干净地取消、如何处理可能与驱动程序其他代码路径并发运行的callout周围的规则，以及如何将定时器用于典型驱动程序模式如看门狗、延迟工作和定期轮询。

第12章以三种特定方式准备了基础。

首先，你已经知道如何带超时等待。第13章的`callout(9)`机制是从另一侧看到的相同想法：不是"在时间T唤醒我"，而是"在时间T运行这个函数"。callout周围的同步规则（callout在内核线程上运行，可以与你的其他代码竞争，必须在销毁前排空）建立在第12章为cv和sx建立的纪律上。

其次，你已经知道如何设计多原语驱动程序。第13章的callout向驱动程序添加了另一个执行上下文：callout处理程序与`myfirst_read`、`myfirst_write`和sysctl处理程序并发运行。这意味着callout处理程序参与锁顺序。你在第12章编写的`LOCKING.md`将用一个新条目吸收添加。

第三，你已经知道如何在负载下调试。第13章引入了一个新的bug类别（卸载时的callout竞争），它受益于第12章教授的相同`WITNESS`、`lockstat`和`dtrace`工作流程。

第13章将涵盖的具体主题包括：

- `callout(9)` API：`callout_init`、`callout_init_mtx`、`callout_reset`、`callout_stop`、`callout_drain`。
- 感知锁的callout（`callout_init_mtx`）以及为什么它是驱动程序代码的正确默认值。
- Callout重用：安全地多次调度同一个callout。
- 卸载竞争：`kldunload`后触发的callout如何可能使内核崩溃，以及如何用`callout_drain`防止它。
- 周期模式：看门狗、心跳、延迟收割器。
- `tick_sbt`和`sbintime_t`时间抽象，用于亚毫秒计时。
- 与`timeout(9)`的比较（较旧接口，不推荐新代码使用）。

你不需要向前读。第12章材料足够准备。带上你的第12章驱动程序、你的测试套件和你的`WITNESS`启用内核。第13章从第12章结束的地方开始。

一个小小的结束反思。你以一个互斥锁、一个匿名通道和对同步意味着什么的清晰想法开始本章。你以三个原语、一个记录的锁顺序、一个更丰富的词汇表和使用真实内核工具调试真实协调问题的经验结束。那个进展是本书第3部分的核心。从这里，第13章扩展驱动程序对*时间*的意识，第14章扩展其对*中断*的意识，第3部分的其余章节为你准备第4部分的硬件接触章节。

暂停片刻。你开始第3部分时的驱动程序只知道如何一次处理一个线程。你现在拥有的驱动程序跨两个锁类别协调多个线程，可以在运行时重新配置而不中断其数据路径，并尊重用户的信号和截止期限。那是真正的质的飞跃。然后翻页。

当你确实打开第13章时，你将看到的第一件事是`callout(9)`，内核的定时回调基础设施。你在这里为cv、sx和锁顺序感知设计学习的纪律直接转移。Callout只是另一个参与锁顺序的并发执行上下文；第12章的模式吸收它们而不会变得脆弱。同步词汇表相同；时间词汇表是新的。
