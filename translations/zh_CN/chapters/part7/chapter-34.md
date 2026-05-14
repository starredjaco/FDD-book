---
title: "高级调试技术"
description: "针对复杂驱动问题的先进调试方法"
partNumber: 7
partName: "精通主题：特殊场景与边缘情况"
chapter: 34
lastUpdated: "2026-04-20"
status: "complete"
author: "Edson Brandi"
reviewer: "TBD"
translator: "TBD"
estimatedReadTime: 135
language: "zh-CN"
---

# 高级调试技术

## 引言

在前一章中，我们学习了如何测量驱动程序做了什么以及它的执行速度。我们观察性能计数器的增长，运行 DTrace 聚合来发现热点路径，并使用 `pmcstat` 查看哪些指令真正消耗了 CPU 周期。测量为我们提供了一种语言，用来询问驱动程序是否按照我们期望的方式运行。

调试提出的是一个不同的问题。它问的不是"这有多快？"而是"为什么这是错的？"性能问题通常产生的是运行缓慢但仍在运行的代码。正确性问题则可能导致崩溃、死锁、静默数据损坏、驱动程序拒绝卸载、指针解引用到垃圾数据，或者一把不知为何无人持有的锁。这些正是让经验丰富的内核工程师深吸一口气，然后去寻找更好工具的那类 bug。

FreeBSD 为我们提供了这些工具。它们涵盖的范围从驻留在内核中、在 bug 发生瞬间就能捕获它的极小极快断言，到对已经不再运行的机器进行完整的崩溃转储事后分析。还有运行时几乎零开销的轻量级跟踪环形缓冲区、能展开整个调用图的重型跟踪器，以及在开发期间可以替换进来、将微妙的释放后使用（use-after-free）bug 转化为立即可诊断崩溃的内存分配器。一个装备齐全的驱动程序作者会学会为正确的 bug 选择正确的工具，而不是盯着 `printf` 输出期盼顿悟。

本章的目标就是教你这套工具集。我们将首先理解何时应该使用高级调试、何时更简单的方法就足够了。然后我们将学习内核断言宏、panic 路径，以及如何使用 `kgdb` 离线读取和分析崩溃转储。我们将构建一个对调试友好的内核，使这些工具在需要时真正可用，学习如何使用 DTrace 和 `ktrace` 跟踪驱动程序行为，最后研究如何使用 `memguard(9)`、`redzone` 和保护页来追踪内存泄漏和无效访问。我们将以生产系统上的调试纪律作为结束——在那里每个操作都有后果，并简要研究如何在严重故障后重构驱动程序，使其对下一次故障更具韧性。

在本章中，我们将使用一个名为 `bugdemo` 的配套驱动程序。它是一个伪设备，包含蓄意的、可控的 bug，我们可以通过简单的 `ioctl(2)` 调用触发这些 bug，然后用本章教授的每种技术来追踪它们。我们所做的一切都不涉及真实硬件，因此即使我们故意让内核崩溃，实验环境也是安全的。

到本章结束时，你将能够为驱动程序添加防御性断言、构建调试内核、捕获崩溃转储、在 `kgdb` 中打开它、使用 DTrace 和 `ktrace` 跟踪实时行为、使用 `memguard(9)` 及相关工具捕获内存误用，并安全地将所有这些规范应用在其他人依赖该机器运行的系统上。

## 读者指南：如何使用本章

本章位于本书第七部分，与性能调优、异步 I/O 等其他精通主题并列。本章假设你已经编写过至少一个简单的字符设备驱动程序，理解加载和卸载的生命周期，并且已经使用过 `sysctl`、`counter(9)` 和 DTrace（在第 33 章中介绍的水平）。如果其中任何内容感觉不确定，快速回顾第 8 章到第 14 章以及第 33 章将在本章中为你带来数倍的回报。

### 与第23章一起阅读本章

本章特意从第 23 章结束的地方接续。第 23 章"调试与跟踪"介绍了基础知识：如何思考 bug、如何使用 `printf`、如何使用 `dmesg` 和内核日志、如何阅读简单的 panic、如何启用 DTrace 探针，以及如何从一开始就让驱动程序更容易观察。它紧密围绕新驱动程序作者所需的日常调试习惯。

第 23 章还以明确的交接作为结尾。它标明了对崩溃转储进行深度 `kgdb` 脚本编写和活内核断点工作流将留给后续更高级的章节。那后续的章节就是本章。你正在阅读的是一对章节的后半部分。如果第 23 章是急救箱，第 34 章就是完整的临床工具箱。

在实践中，这意味着两件事。首先，我们不会重新解释第 23 章已经涵盖的基础知识；我们假设你已经熟悉 `printf`、基本的 panic 阅读和入门级 DTrace。如果其中任何一个让你觉得不稳妥，请先重读第 23 章的相关章节，因为高级材料是直接建立在那些习惯之上的。其次，当这里的技术在第 23 章中有更简单的对应方法时（例如，`kgdb` 中基本的 `bt` 比遍历 `struct thread` 字段更简单），我们会指向第 23 章的版本，然后展示为什么高级版本值得其额外的复杂性。

将这两章视为一个完整的弧线。第 23 章教你如何注意到出了问题并进行初步查看。第 34 章教你如何详细重建 bug 发生时内核正在做什么，即使是在一台已经不再运行的机器上。

本章内容是累积的。每个部分都在 `bugdemo` 驱动程序上增加一层，因此实验最好按顺序阅读。你可以先浏览以供参考，但如果这是你第一次接触内核调试工具，按顺序完成实验将建立我们期望的思维模型。

你不需要任何特殊硬件。一台适中的 FreeBSD 14.3 虚拟机就足以完成本章中的每个实验。对于实验 3 和实验 4，你需要配置崩溃转储设备（本章会逐步介绍），对于实验 5，你需要在内核中启用 DTrace。这两者在普通的 FreeBSD 安装中都是标准的。

本章中的一些技术会故意使内核崩溃。这在开发机器上是安全的，也是学习过程的预期部分。但在其他人依赖不间断服务的生产机器上是不安全的。本章的最后一节专门讨论这个区别，因为知道何时不该使用工具的纪律与知道如何使用工具同样重要。

## 如何从本章获得最大收益

本章围绕一个反复出现的模式组织。首先我们解释一种技术是什么，然后解释它为什么存在以及它要捕获什么类型的 bug，然后我们将其落实到真实的 FreeBSD 源代码中，让你看到这个想法在内核中的位置，最后我们通过一个小实验将其应用到 `bugdemo` 驱动程序上。阅读和实验相结合是最有效的方法。实验被特意设计得足够小，每个只需几分钟即可运行。

一些习惯会让工作更顺畅。保持一个终端打开到 `/usr/src/`，这样每当本章引用真实代码时你就可以查看。本书通过观察真实的 FreeBSD 实践来教学，而不是通过编造的伪代码，通过亲眼确认 `KASSERT` 确实定义在本章所说的位置，或者 `memguard(9)` 确实拥有我们描述的 API，你将建立更强的直觉。

保持第二个终端打开到你的测试虚拟机，在那里你将加载 `bugdemo` 驱动程序、触发 bug 并观察输出。如果可以给虚拟机连接串行控制台，请这样做。串行控制台是在机器重启前捕获 panic 消息尾部最可靠的方式，我们将在多个实验中使用它。

最后，保持预期的校准。内核 bug 通常不是它们最初看起来的样子。释放后使用可能首先表现为不相关子系统中的随机数据损坏。死锁可能首先看起来像是一个缓慢的系统调用。本章教授的最有价值的技能之一是耐心：在形成理论之前收集证据，在承诺修复之前确认理论。工具有所帮助，但纪律才是区分快速 bug 猎杀和漫长排查的关键。

带着这些预期，让我们开始讨论何时高级调试才是对问题的正确回应。

## 1. 何时以及为何需要高级调试

驱动程序中的大多数 bug 无需借助崩溃转储或跟踪框架即可解决。仔细阅读代码、添加一个额外的 `printf`、重新检查函数的返回值、查看 `dmesg`——这些方法加在一起就能解决驱动程序作者遇到的大部分缺陷。如果你能看到问题、能低成本地重现它、并且能在脑中保持相关代码的清晰印象，那么最简单的工具就是正确的工具。

高级调试是为那些无法通过上述方法解决的 bug 而存在的。它是我们在以下情况下才会动用的工具箱：问题很罕见、问题的表现远离其根源、只在特定的时序下才会出现、驱动程序挂起而不是崩溃、或者症状是数据损坏而不是功能失败。这些 bug 有一个共同特征：它们需要你无法通过阅读代码轻松收集的证据，并且需要对内核执行过程的控制权——而这是普通用户进程所不具备的。

### 需要比printf更多工具的Bug

第一类需要高级工具的 bug 是会销毁自身成因证据的 bug。释放后使用（use-after-free）是典型例子。驱动程序释放了一个对象，然后一些后续代码——可能在不同的函数或不同的线程中——读取或写入了那块内存。等到崩溃发生时，释放操作早已完成，内存已被重新用于无关的用途，崩溃点的回溯指向的是受害者而非罪魁祸首。在崩溃点添加 `printf` 会忠实地打印出它看到的乱码，但不会告诉你谁释放了内存或何时释放的。

第二类是只在并发条件下才出现的 bug。两个线程竞争一把锁。其中一个以错误的顺序获取了锁，与另一个以相反顺序获取相同锁的线程发生死锁。系统安静下来，bug 没有在控制台留下任何消息。在加锁路径中添加 `printf` 调用往往会干扰时序，恰好使 bug 消失——这是 Heisenbug 爱好者熟悉的令人沮丧的特性。FreeBSD 通过 `WITNESS` 提供的静态锁序检查之所以存在，正是因为这类 bug 很难通过其他方式发现。

第三类是在用户空间完全无法观察的 bug。驱动程序在某条代码路径上损坏了一个内核数据结构，其后果在许多分钟后才在不相关的子系统中显现。触发损坏的进程在出问题时早已消失。将原因与结果关联起来的唯一方法是在 panic 的瞬间捕获完整的内核状态并用 `kgdb` 离线遍历，或者用 DTrace 持续跟踪内核，让可疑事件留下痕迹。

第四类是只出现在你无法附加调试器的硬件上、或无法直接插桩的生产配置中的 bug。驱动程序运行在客户的机器上，每周崩溃一次，没有人希望你把开发工作站物理连接到上面。这种情况下的工具就是崩溃转储：在 panic 时写入磁盘的内核内存快照，可以被转移到安全环境中进行分析。`dumpon(8)` 配置转储的写入位置，`savecore(8)` 在重启后取回转储，`kgdb` 离线读取。

这些 bug 类别中的每一个在 FreeBSD 调试工具箱中都有对应的工具。本章其余部分将逐一介绍它们。本节开头的目的是设定预期：我们不是要学习一种取代 `printf` 的单一技术，而是要学习一族技术，每一种适用于特定类型的困难。

### 高级工具的成本

高级调试不是免费的。我们将学习的每一种技术都带有构建时成本、运行时成本和纪律成本的某种组合。

构建时成本最容易描述。`INVARIANTS` 和 `WITNESS` 会使内核变慢，因为它们添加了生产内核会跳过的检查。`DEBUG_MEMGUARD` 会使某些分配大幅变慢，因为它用完整的页面映射替换分配并在释放时取消映射。使用 `makeoptions DEBUG=-g` 构建的调试内核比发布版内核大好几倍，因为每个函数都携带完整的调试信息。在开发机器上这些成本无关紧要，因为正确性的价值比速度高出几个数量级。但在生产环境中它们全部举足轻重。

运行时成本适用于你在运行中的内核里启用的工具。已禁用的 DTrace 探针基本上不花什么成本，但启用的探针在每次命中被插桩函数时仍会执行。`ktr(9)` 条目非常便宜但并非免费。一个详细的跟踪会话可以产生足够填满磁盘的日志输出。`kdb` 会话会暂停整个内核，这对有人正在使用的机器来说是一场灾难。每种工具都有一个运行时预算，本章的纪律之一就是了解那个预算是多少。

纪律成本最难量化，但也最容易低估。高级调试需要耐心、认真的记录，以及与不完整信息共处的意愿。它要求你抵抗在理解底层缺陷之前就修补可见症状的冲动。发生在模块 X 中的崩溃几乎从来不意味着 bug 就在模块 X 中。学会在形成理论之前先收集证据的读者，在学习本章时会比急于尽快提交修复的读者轻松得多。

### 决策框架

考虑到这些成本，以下是一个选择工具的简单决策框架。如果 bug 容易重现且原因可能在附近代码中可见，从阅读代码和策略性地放置 `printf` 或 `log(9)` 语句开始。如果 bug 只在负载或并发条件下出现，启用 `INVARIANTS` 和 `WITNESS` 并重新构建。如果 bug 产生了 panic，捕获转储并在 `kgdb` 中打开它。如果 bug 涉及内存损坏，对可疑的分配类型启用 `DEBUG_MEMGUARD`。如果 bug 表现为静默的错误行为而非崩溃，添加 SDT 探针并用 DTrace 观察它们。如果你需要理解中断处理程序中事件之间的时序，使用 `ktr(9)`。如果 bug 出现在生产机器上，在做任何事情之前先阅读第 7 节。

本章其余部分将深入讲解每种技术。我们即将认识的 `bugdemo` 驱动程序为我们提供了一个安全的地方来应用每一种技术，有已知的 bug 可以追踪，有已知的答案可以寻找。

### 认识bugdemo驱动

`bugdemo` 驱动程序是一个小型伪设备，我们将它作为本章的实验对象。它没有硬件需要驱动。它在 `/dev/bugdemo` 暴露一个设备节点，并接受少量 `ioctl(2)` 命令来故意触发不同类别的 bug：空指针解引用、`WITNESS` 可以捕获的无锁访问、释放后使用、内存泄漏、自旋锁内的无限循环等。每个 ioctl 都由一个 sysctl 开关控制，这样驱动程序可以在开发系统上安全加载而不会意外触发任何东西。

我们将在实验 1 中正式介绍这个驱动程序，届时我们已经掌握了断言宏。现在请记住，我们学习的每一种技术都可以在 `bugdemo` 上演示，有已知的起点和已知的答案。在受控环境中重现 bug——这种纪律本身就是本章旨在教授的最重要技能之一。

现在我们已准备好正式开始工具箱的学习，从在 bug 发生瞬间就能捕获它们的断言宏开始。

## 2. 使用KASSERT、panic及相关宏

用户空间中的防御性编程通常围绕运行时检查和谨慎的错误处理展开。内核中的防御性编程还多了一个工具：断言宏。断言声明了一个在给定位置必须为真的条件。如果条件为假，说明出现了严重问题，最安全的响应是立即停止内核，防止错误状态有机会扩散。断言是 FreeBSD 提供的最便宜、最有效的调试工具，它们属于每一个严肃的驱动程序。

我们将从两个最重要的宏 `KASSERT(9)` 和 `panic(9)` 开始，看看一些有用的伴生宏，然后讨论各自的适用场景。

> **关于行号的说明。** 当本章引用 `kassert.h`、`kern_shutdown.c` 或 `cdefs.h` 中的代码时，定位标记始终是宏名或函数名。`KASSERT`、`kassert_panic`、`panic` 和 `__dead2` 在每一个 FreeBSD 14.x 源码树中都可以通过这些名字找到，即使它们周围的行已经移动。你后面将看到的示例回溯引用了 `kern_shutdown.c:400` 这样的 `file:line` 对，反映的是撰写时 14.3 源码树的情况，在新更新的系统上不会逐行匹配。请用 grep 搜索符号名而不是滚动到行号。

### KASSERT：在生产环境中消失的检查

`KASSERT` 是内核中对应于用户空间 `assert()` 宏的等价物，但更智能。它接受一个条件和一个消息。如果条件为假，内核以该消息触发 panic。如果内核在编译时没有启用 `INVARIANTS` 选项，整个检查在编译时就被消除，运行时不花费任何成本。

这个宏位于 `/usr/src/sys/sys/kassert.h`。在 FreeBSD 14.3 源码树中它看起来是这样的：

```c
#if (defined(_KERNEL) && defined(INVARIANTS)) || defined(_STANDALONE)
#define KASSERT(exp,msg) do {                                           \
        if (__predict_false(!(exp)))                                    \
                kassert_panic msg;                                      \
} while (0)
#else /* !(KERNEL && INVARIANTS) && !STANDALONE */
#define KASSERT(exp,msg) do { \
} while (0)
#endif /* KERNEL && INVARIANTS */
```

这个定义中有四个细节值得仔细审视。

第一，宏的定义方式取决于是否设置了 `INVARIANTS`。如果没有设置，`KASSERT` 展开为一个空的 `do { } while (0)` 块，编译器会将其完全优化掉。没有 `INVARIANTS` 的发布版内核无论驱动程序中包含多少 `KASSERT` 调用，都不承担任何运行时成本。这正是让我们在开发时可以慷慨地编写断言而无需担心生产性能的属性。`_STANDALONE` 分支让同一个宏也能在引导加载程序中工作，那里可能没有 `INVARIANTS` 但检查仍然是需要的。

第二，`__predict_false` 提示告诉编译器该条件几乎总是为真。这改善了公共路径的代码生成，因为编译器会安排分支使得热路径不需要跳转。定义 `__predict_false` 是保持调试内核可用性的小性能纪律之一。

第三，失败断言的主体调用的是 `kassert_panic` 而非 `panic`。这是一个实现细节，目的是让断言消息更容易解析，但当你看到实际的 panic 消息时它很重要：`KASSERT` 失败会产生一个独特的前缀，我们后面会认出它。

第四，注意 `msg` 参数是用双层括号传递的。这是因为宏将其直接传递给 `kassert_panic`，后者具有 `printf` 风格的签名。在实践中你这样写：

```c
KASSERT(ptr != NULL, ("ptr must not be NULL in %s", __func__));
```

外层括号属于宏。内层括号是 `kassert_panic` 的参数列表。初学者的常见错误是只写一层括号 `KASSERT(ptr != NULL, "ptr is NULL")`，这无法编译。双层括号是一种纪律，提醒我们失败的断言将像 `printf` 一样格式化。

### INVARIANTS and INVARIANT_SUPPORT

`INVARIANTS` 是控制 `KASSERT` 是否生效的内核构建选项。调试内核会启用它。FreeBSD 14.3 附带的 `GENERIC-DEBUG` 配置通过包含 `std.debug` 来启用它，你可以在 `/usr/src/sys/conf/std.debug` 中看到。发布版 `GENERIC` 内核不启用它。

还有一个相关选项叫做 `INVARIANT_SUPPORT`。`INVARIANT_SUPPORT` 编译进断言可能调用的函数，但不使它们成为强制性的。这允许用 `INVARIANTS` 构建的可加载内核模块加载到没有用 `INVARIANTS` 构建的内核中，只要该内核有 `INVARIANT_SUPPORT`。对于驱动程序作者来说，实际意义在于：如果你用 `INVARIANTS` 构建模块，确保你要加载到的内核至少有 `INVARIANT_SUPPORT`。`GENERIC-DEBUG` 内核两者都有，这是我们推荐在整个开发过程中使用它的原因之一。

### MPASS：带默认消息的KASSERT

为每个断言都写一条消息可能很繁琐，特别是对于简单的不变式。FreeBSD 提供了 `MPASS` 作为 `KASSERT(expr, ("Assertion expr failed at file:line"))` 的简写：

```c
#define MPASS(ex)               MPASS4(ex, #ex, __FILE__, __LINE__)
#define MPASS2(ex, what)        MPASS4(ex, what, __FILE__, __LINE__)
#define MPASS3(ex, file, line)  MPASS4(ex, #ex, file, line)
#define MPASS4(ex, what, file, line)                                    \
        KASSERT((ex), ("Assertion %s failed at %s:%d", what, file, line))
```

四种形式允许你自定义消息、文件，或两者兼有。最简单的形式 `MPASS(ptr != NULL)` 会自动将表达式字符串化并嵌入位置。当消息可以简明扼要时，`MPASS` 在源代码中产生的视觉杂乱更少。当消息需要未来读者会感谢的上下文时，优先使用带书面消息的 `KASSERT`。

一个合理的经验法则是：`MPASS` 用于不应该发生的内部不变式，且表达式的含义不言自明。`KASSERT` 用于失败模式值得描述性消息的条件。

### CTASSERT：编译时断言

有时你想检查的条件可以在编译时决定。例如 `sizeof(struct foo) == 64` 或 `MY_CONST >= 8`。对于这些情况，FreeBSD 提供了 `CTASSERT`，同样位于 `/usr/src/sys/sys/kassert.h`：

```c
#define CTASSERT(x)     _Static_assert(x, "compile-time assertion failed")
```

`CTASSERT` 使用 C11 的 `_Static_assert`。如果条件为假，它会产生编译时错误，并且没有任何运行时成本，因为没有运行时参与。这是结构体布局检查的理想工具——这些检查必须成立才能保证驱动程序正确。

内核中的典型用法是保护结构体免受意外的大小变更：

```c
struct bugdemo_command {
        uint32_t        op;
        uint32_t        flags;
        uint64_t        arg;
};

CTASSERT(sizeof(struct bugdemo_command) == 16);
```

如果后来有人添加了字段却没有调整大小注释或仔细重新排列，构建会立即中断。这远比在运行时才发现结构体增长了、ioctl 不再匹配用户空间期望要好得多。

### panic：无条件停止

`KASSERT` 是条件检查，而 `panic` 是无条件版本。当你判定继续执行比停止更糟糕时调用它：

```c
void panic(const char *, ...) __dead2 __printflike(1, 2);
```

声明位于 `/usr/src/sys/sys/kassert.h`，实现在 `/usr/src/sys/kern/kern_shutdown.c`。`__dead2` 属性告诉编译器 `panic` 不会返回，这让它在下游产生更好的代码。`__printflike(1, 2)` 属性告诉编译器第一个参数是 `printf` 风格的格式字符串，这样编译器可以对格式与其参数进行类型检查。

什么时候应该直接使用 `panic` 而不是 `KASSERT`？三种常见情况。第一，当情况极其灾难性，即使在发布版内核中也没有安全的继续路径。例如，在 `attach` 期间分配 soft 上下文失败——如果驱动程序已经部分注册了，可能是 `panic` 而非优雅清理更合适。第二，当你希望消息即使在非调试构建中也出现，因为事件指示了用户必须知道的硬件或配置失败。第三，作为早期开发期间的占位符，确保不可达路径确实不可达，然后在代码成熟后将 `panic` 替换为 `KASSERT(0, ...)`。

`/usr/src/sys/dev/` 中的一些驱动程序很少使用 `panic`。阅读几个例子会让你对基调有所感觉：`panic` 消息说的是类似"控制器返回了一个不可能的状态"或"我们到达了状态机声称不可能出现的情况"。它不是对 I/O 错误的正常响应。它是对一个已被严重破坏的不变式的响应——破坏到驱动程序不能被信任继续运行。

### __predict_false and __predict_true

我们在 `KASSERT` 定义中看到了 `__predict_false`。这两个宏定义在 `/usr/src/sys/sys/cdefs.h`，是给分支预测器的编译时提示：

```c
#if __GNUC_PREREQ__(3, 0)
#define __predict_true(exp)     __builtin_expect((exp), 1)
#define __predict_false(exp)    __builtin_expect((exp), 0)
#else
#define __predict_true(exp)     (exp)
#define __predict_false(exp)    (exp)
#endif
```

它们不改变表达式的语义。它们只告诉编译器哪种结果更可能，这会影响编译器的代码布局。在热路径中，用 `__predict_true` 包裹一个可能为真的条件可以改善缓存行为；用 `__predict_false` 包裹一个可能为假的条件则让错误处理代码不落在快速路径上。

使用这些宏的第一条规则是正确。如果你预测错了，你会使代码变慢而不是变快。第二条规则是只在确实有影响的热路径中使用它们。对于大多数驱动程序代码，编译器的默认启发式就够了，在代码中添加预测杂乱无章，弊大于利。

### 断言在驱动中的位置

有了这些宏，你实际上应该把断言放在哪里？在 FreeBSD 驱动程序中，几种模式已被证明是有用的。

第一种是在函数入口处，用于非平凡的前置条件。一个期望在持有特定锁时被调用的驱动程序函数是完美的候选：

```c
static void
bugdemo_process(struct bugdemo_softc *sc, struct bugdemo_command *cmd)
{
        BUGDEMO_LOCK_ASSERT(sc);
        KASSERT(cmd != NULL, ("cmd must not be NULL"));
        KASSERT(cmd->op < BUGDEMO_OP_MAX,
            ("cmd->op %u out of range", cmd->op));
        /* ... */
}
```

`BUGDEMO_LOCK_ASSERT` 是许多驱动程序采用的宏约定，它封装了 `mtx_assert(9)` 或 `sx_assert(9)` 调用。这种模式——每个子系统有自己的 `_ASSERT` 宏来检查自己的锁——在大型驱动程序中扩展性很好。

第二种模式是在状态转换处。如果驱动程序状态机有四个有效状态，而 `attach` 路径应该只在 `INIT` 状态下运行，那么在 `attach` 顶部的断言将捕获任何未来破坏该不变式的重构：

```c
KASSERT(sc->state == BUGDEMO_STATE_INIT,
    ("attach called in state %d", sc->state));
```

第三种模式是在微妙的算术运算之后。如果一次计算应该产生一个已知范围内的值，检查它：

```c
idx = (offset / PAGE_SIZE) & (SC_NRING - 1);
KASSERT(idx < SC_NRING, ("idx %u out of range", idx));
```

这在环形缓冲区代码中特别有价值，因为生产者和消费者之间的差一错误可能导致静默的数据损坏。

第四种模式是用于可能为 NULL 但不应为 NULL 的指针。如果函数接收一个只有非零时才有效的指针参数，在函数顶部的一个 `KASSERT(ptr != NULL, ...)` 可以捕获多年的未来误用。

### 何时不应使用断言

断言不能替代错误处理。规则是：`KASSERT` 检查程序员保证的事情，而不是环境保证的事情。如果使用 `M_NOWAIT` 的内存分配在内存压力下可能失败，你不应该断言它成功了。你应该检查返回值并处理失败。如果用户空间程序传递了一个比你预期更大的结构体，你应该返回 `EINVAL`，而不是 `KASSERT(0)`。断言用于内部一致性，不用于外部输入。

另一种反模式是对只在某些配置中成立的条件使用断言。`KASSERT(some_sysctl == default)` 是错误的，如果 some_sysctl 是用户可调的，因为断言在任何调过它的系统上都会失败。应该显式检查配置并处理它，或者只在假设实际成立的分支内断言。

一种更微妙的反模式是将断言用作文档。"这是它的工作方式，而且最好保持这样"是一种诱人的 `KASSERT` 用法，但如果断言只是今天成立而明天可能合理改变，你就为某个不记得你承诺的人创造了一个未来的 bug。最好留下一个说明假设的注释，让代码演进。断言应该捕获永久性的不变式，而不是临时的实现选择。

### 一个小型真实世界示例

让我们看看这些想法在真实 FreeBSD 代码中的应用。打开 `/usr/src/sys/dev/null/null.c`，查看 read 处理程序附近的典型检查。该驱动程序极其简单，因此断言很少，但 `/usr/src/sys/dev/` 中的许多驱动程序大量使用 `KASSERT`。更丰富的示例可以浏览 `/usr/src/sys/dev/uart/uart_bus_pci.c` 或 `/usr/src/sys/dev/mii/mii.c`，那里的函数入口断言捕获了未持有期望锁的调用者。

整个源码树中这种模式的一致性并非偶然。它反映了一种文化期望：驱动程序将在代码中而非仅在注释中表达其不变式。当你在自己的驱动程序中采用同样的习惯时，你就融入了这种文化。你的驱动程序将更容易移植、更容易审查，在最终出问题时也更容易调试。

### 快速示例：为 bugdemo 添加断言

让我们为一设想的 `bugdemo` 驱动程序添加一组断言。假设我们有一个包含互斥锁、状态字段和计数器的 softc 结构，以及一个接受 `struct bugdemo_command` 的 `ioctl` 处理程序。

```c
static int
bugdemo_ioctl(struct cdev *dev, u_long cmd, caddr_t data, int fflag,
    struct thread *td)
{
        struct bugdemo_softc *sc = dev->si_drv1;
        struct bugdemo_command *bcmd = (struct bugdemo_command *)data;

        KASSERT(sc != NULL, ("bugdemo: softc missing"));
        KASSERT(sc->state == BUGDEMO_STATE_READY,
            ("bugdemo: ioctl in state %d", sc->state));

        switch (cmd) {
        case BUGDEMO_TRIGGER:
                KASSERT(bcmd->op < BUGDEMO_OP_MAX,
                    ("bugdemo: op %u out of range", bcmd->op));
                BUGDEMO_LOCK(sc);
                bugdemo_process(sc, bcmd);
                BUGDEMO_UNLOCK(sc);
                return (0);
        default:
                return (ENOTTY);
        }
}
```

四个断言，每一个捕获一类不同的未来 bug。第一个检查驱动程序的私有指针是否实际设置，这在 `make_dev(9)` 被误传 `NULL` 时很容易忘记。第二个检查驱动程序状态，如果有人添加了可以在 `attach` 完成前到达 `ioctl` 的代码路径，它就会触发。第三个检查用户提供的输入是否在范围内，不过在生上下文中，这个特定的检查也应作为返回错误的真正输入验证来完成，因为 `ioctl` 是公共接口。第四个（此处未显示但由 `bugdemo_process` 暗示）断言锁已被持有。

这几行代码表达了大量不变式。在调试内核中，它们会在 bug 发生的瞬间捕获真正的 bug。在发布版内核中，它们完全消失。这就是 `KASSERT` 提供的契约，接受它是驱动程序作者能培养的最佳习惯之一。

有了这个基础，我们可以继续讨论断言实际触发时会发生什么，这将我们带到了 panic 路径和崩溃转储。

## 3. 分析崩溃与崩溃转储

当 `KASSERT` 失败或 `panic` 被调用时，内核会执行一系列明确定义的步骤。理解这些步骤是理解崩溃的第一部分。第二部分是了解内核留下了什么痕迹，以及事后如何读取它们。本节将逐一讲解。

### 崩溃时会发生什么

panic 是内核对不可恢复错误的受控关闭。具体序列取决于构建选项，但 FreeBSD 14.3 内核中的典型 panic 过程如下。

首先，`panic()` 或 `kassert_panic()` 被调用并传入一条消息。消息被格式化并写入系统日志。如果连接了串行控制台，消息会立即出现在那里。如果只有图形控制台可用，消息会显示在屏幕上，但在机器重启前通常没有足够的时间读取长跟踪信息，这是我们推荐在本章中使用串行或虚拟控制台的原因之一。

其次，内核捕获恐慌线程的回溯。你将在控制台上看到一系列带有偏移量的函数名。回溯是 panic 产生的最有价值的信息，因为它告诉你导致失败的调用链。自顶向下阅读，它显示调用 `panic` 的函数、调用该函数的函数，以此类推，直到入口点。

第三，如果内核在构建时启用了 `KDB` 并且有 `DDB` 等后端，内核进入调试器。`DDB` 是内核内调试器。它直接在控制台上接受命令：`bt` 显示回溯，`show registers` 转储寄存器状态，`show proc` 显示进程信息等。我们将在第 4 节中简要使用 `DDB`。如果 `KDB` 未启用，或内核配置为在 panic 时跳过调试器，内核继续下一步。

第四，如果配置了转储设备，内核向其写入转储。转储是内核内存的全部内容，或至少标记为可转储的部分，序列化到转储设备上。这就是 `savecore(8)` 在重启后将取回的崩溃转储。

第五，内核重启机器，除非它被要求停在调试器中。重启后，系统启动时 `savecore(8)` 运行并将转储写入 `/var/crash/vmcore.N`，同时生成一份文本摘要。现在你有了离线分析崩溃所需的一切。

整个过程从几分之一秒到几分钟不等，取决于内核大小、转储设备速度和系统配置。在开发虚拟机上，将几百兆字节的内核转储到虚拟磁盘通常只需要几秒钟。

### 阅读崩溃消息

FreeBSD 14.3 中的 panic 消息看起来像这样：

```text
panic: bugdemo: softc missing
cpuid = 0
time = 1745188102
KDB: stack backtrace:
db_trace_self_wrapper() at db_trace_self_wrapper+0x2b
vpanic() at vpanic+0x182
panic() at panic+0x43
bugdemo_ioctl() at bugdemo_ioctl+0x24
devfs_ioctl() at devfs_ioctl+0xc2
VOP_IOCTL_APV() at VOP_IOCTL_APV+0x3f
vn_ioctl() at vn_ioctl+0xdc
devfs_ioctl_f() at devfs_ioctl_f+0x1a
kern_ioctl() at kern_ioctl+0x284
sys_ioctl() at sys_ioctl+0x12f
amd64_syscall() at amd64_syscall+0x111
fast_syscall_common() at fast_syscall_common+0xf8
--- syscall (54, FreeBSD ELF64, sys_ioctl), rip = ..., rsp = ...
```

从上到下阅读。第一行是 panic 消息本身。`cpuid` 和 `time` 行是元数据，对调试很少有用，但偶尔有助于比对多个日志。`KDB: stack backtrace:` 行标记跟踪的开始。

前几帧是 panic 基础设施本身：`db_trace_self_wrapper`、`vpanic`、`panic`。这些在每次 panic 中都会出现，可以跳过。第一个有意义的帧是 `bugdemo_ioctl`，这是我们驱动程序调用 `panic` 的地方。下面的帧是到达 `bugdemo_ioctl` 的路径：`devfs_ioctl`、`vn_ioctl`、`kern_ioctl`、`sys_ioctl`、`amd64_syscall`。这告诉我们 panic 发生在 ioctl 系统调用期间，这已经是一个有用的线索。最后一行显示系统调用号（54，即 `ioctl`）和入口处的指令指针。

偏移量（`+0x24`、`+0xc2`）是每个函数中的字节偏移。它们本身不是人类可读的，但它们让 `kgdb` 在调试内核可用时能解析到确切的源代码行。

记下这类消息，或捕获串行控制台日志，是 panic 发生时你应该做的第一件事。如果机器重启太快来不及阅读，请配置串行控制台或文本模式虚拟控制台以保留历史记录。

### 配置转储设备

要让 `savecore(8)` 有内容可以取回，内核必须知道把转储写到哪里。FreeBSD 称之为转储设备，`dumpon(8)` 是配置它的工具。

有两种常见的设置方式。最简单的是使用交换分区。在安装过程中，`bsdinstall` 通常会创建一个足够大的交换分区来容纳内核内存，FreeBSD 14.3 会在你启用相关选项后自动将其配置为转储设备。你可以用以下命令检查：

```console
# dumpon -l
/dev/da0p3
```

如果该命令列出了你的交换设备，就可以了。如果它说没有配置转储设备，你可以手动设置：

```console
# dumpon /dev/da0p3
```

要使其在重启后持久化，将其放入 `/etc/rc.conf`：

```sh
dumpdev="/dev/da0p3"
dumpon_flags=""
```

你可以在 `/usr/src/libexec/rc/rc.conf` 中看到这些变量的默认值，这是基本系统中所有默认 rc.conf 值的权威来源。用 grep 搜索 `dumpdev=` 和 `dumpon_flags=` 即可找到相关块。

另一种方法是使用现代 FreeBSD 引入的文件后备转储。这避免了专门为转储划分磁盘分区的需要。具体语法见 `dumpon(8)`；简而言之，你可以将 `dumpon` 指向文件系统上的一个文件，内核在 panic 时会转储到其中。文件后备转储对于不想重新分区的开发虚拟机很方便。

第二个 rc.conf 变量控制 `savecore(8)` 将取回的转储放在哪里：

```sh
dumpdir="/var/crash"
savecore_enable="YES"
savecore_flags="-m 10"
```

`-m 10` 参数只保留最近的十个转储，这是一个合理的默认值。如果你在追踪一个罕见的 bug，增大这个数字；如果磁盘空间紧张，减小它。`savecore(8)` 在启动期间从 `/etc/rc.d/savecore` 运行，在大多数服务启动之前，所以你的转储在其他任何东西触及 `/var` 之前就已保存。

### 在内核中启用转储

要让内核愿意写入转储，它必须用正确的选项构建。在 FreeBSD 14.3 中，`GENERIC` 内核已经配置了框架组件。如果你查看 `/usr/src/sys/amd64/conf/GENERIC` 文件顶部附近，会看到类似这样的内容：

```text
options         KDB
options         KDB_TRACE
options         EKCD
options         DDB_CTF
```

`KDB` 是内核调试器框架。`KDB_TRACE` 在 panic 时启用自动栈跟踪。`EKCD` 启用加密内核崩溃转储，当转储包含敏感数据时很有用。`DDB_CTF` 告诉构建系统为调试器包含 CTF 类型信息。这些选项共同提供了一个功能完备的转储内核。

注意 `GENERIC` 中*没有*什么：`options DDB` 和 `options GDB` 本身。`KDB` 框架在那里，但内核内调试器后端（`DDB`）和远程 GDB 桩（`GDB`）是由 `std.debug` 添加的，`GENERIC-DEBUG` 包含了它。普通的 `GENERIC` 内核仍然会在 panic 时写入转储，但如果你在运行的系统上进入控制台，不会有 `DDB` 提示符迎接你。

如果你在构建自己的内核，要么显式添加后端，要么更简单地从 `GENERIC-DEBUG` 开始，它启用了后端以及本章其余部分需要的调试选项。`GENERIC-DEBUG` 位于 `/usr/src/sys/amd64/conf/GENERIC-DEBUG`，只有两行：

```text
include GENERIC
include "std.debug"
```

`/usr/src/sys/conf/std.debug` 中的 `std.debug` 文件添加了 `DDB`、`GDB`、`INVARIANTS`、`INVARIANT_SUPPORT`、`WITNESS`、`WITNESS_SKIPSPIN`、`MALLOC_DEBUG_MAXZONES=8`、`ALT_BREAK_TO_DEBUGGER`、`DEADLKRES`、`BUF_TRACKING`、`FULL_BUF_TRACKING`、`QUEUE_MACRO_DEBUG_TRASH` 以及一些子系统特定的调试标志。注意 `DDB` 和 `GDB` 本身来自 `std.debug` 而非 `GENERIC`；发布版内核启用 `KDB` 和 `KDB_TRACE` 但不包含后端，除非你选择启用。这是推荐的驱动程序开发调试内核，除非另有说明，本章其余部分将假定使用此内核。

### 使用 savecore 取回转储

panic 并重启后，`savecore(8)` 在启动序列早期运行。当你获得 shell 提示符时，转储已经在 `/var/crash/` 中了：

```console
# ls -l /var/crash/
total 524288
-rw-------  1 root  wheel         1 Apr 20 14:23 bounds
-rw-r--r--  1 root  wheel         5 Apr 20 14:23 minfree
-rw-------  1 root  wheel  11534336 Apr 20 14:23 info.0
-rw-------  1 root  wheel  11534336 Apr 20 14:23 info.last
-rw-------  1 root  wheel  524288000 Apr 20 14:23 vmcore.0
-rw-------  1 root  wheel  524288000 Apr 20 14:23 vmcore.last
```

`vmcore.N` 文件是转储本身。`info.N` 文件是 panic 的文本摘要，包括 panic 消息、回溯和内核版本。始终先读 `info.N`。如果消息和回溯足以识别 bug，你可能不需要进一步分析。

一些常见问题需要注意。如果 `ls` 只显示 `bounds` 和 `minfree`，说明尚未捕获转储。这通常意味着转储设备未配置或内核在重启前未能写入。检查 `dumpon -l` 并重新触发 panic。如果 `savecore` 记录了校验和不匹配的消息，转储被截断了，通常表示转储设备太小。如果机器从未干净地 panic 而只是简单重启，内核可能没有启用 `KDB`，因此没有转储机制可调用。

`info.N` 文件很短，可以完整阅读。它包含内核版本、panic 字符串和内核在 panic 时捕获的回溯。在 FreeBSD 14.3 上它看起来像这样：

```text
Dump header from device: /dev/da0p3
  Architecture: amd64
  Architecture Version: 2
  Dump Length: 524288000
  Blocksize: 512
  Compression: none
  Dumptime: 2026-04-20 14:22:34 -0300
  Hostname: devbox
  Magic: FreeBSD Kernel Dump
  Version String: FreeBSD 14.3-RELEASE #0: ...
  Panic String: panic: bugdemo: softc missing
  Dump Parity: 3142...
  Bounds: 0
  Dump Status: good
```

如果 `Dump Status` 为 `good`，转储可用。如果为 `bad`，转储被截断或校验失败。

### 使用kgdb打开转储

有了转储后，下一步是用 `kgdb` 打开它。`kgdb` 是 FreeBSD 版本的 `gdb`，专门用于内核映像。它需要三样东西：产生转储的内核映像、包含符号的调试内核映像以及转储文件本身。在大多数系统上，这三者都在可预测的位置：

- 运行中的内核：`/boot/kernel/kernel`
- 带完整符号的调试内核：`/usr/lib/debug/boot/kernel/kernel.debug`
- 转储文件：`/var/crash/vmcore.N`

最简单的调用方式是：

```console
# kgdb /boot/kernel/kernel /var/crash/vmcore.0
```

或等价地：

```console
# kgdb /usr/lib/debug/boot/kernel/kernel.debug /var/crash/vmcore.0
```

`kgdb` 是一个带有内核特定调整的普通 GDB 会话。如果你的内核是用 `makeoptions DEBUG=-g` 构建的（`GENERIC-DEBUG` 就是这样做的），调试符号会被包含在内，`kgdb` 将能够将每个帧解析到源代码。

当 `kgdb` 启动时，它会自动运行一些命令：

```console
(kgdb) bt
#0  __curthread () at /usr/src/sys/amd64/include/pcpu_aux.h:57
#1  doadump (textdump=...) at /usr/src/sys/kern/kern_shutdown.c:400
#2  0xffffffff80b6cf77 in kern_reboot (howto=260)
    at /usr/src/sys/kern/kern_shutdown.c:487
#3  0xffffffff80b6d472 in vpanic (fmt=..., ap=...)
    at /usr/src/sys/kern/kern_shutdown.c:920
#4  0xffffffff80b6d2c3 in panic (fmt=...)
    at /usr/src/sys/kern/kern_shutdown.c:844
#5  0xffffffff83e01234 in bugdemo_ioctl (dev=..., cmd=..., data=..., fflag=..., td=...)
    at /usr/src/sys/modules/bugdemo/bugdemo.c:142
...
```

顶部帧是 panic 基础设施。有意义的帧是第 5 帧，`bugdemo_ioctl`，位于 `bugdemo.c:142`。要跳转到该帧：

```console
(kgdb) frame 5
#5  0xffffffff83e01234 in bugdemo_ioctl (dev=..., cmd=...)
    at /usr/src/sys/modules/bugdemo/bugdemo.c:142
142         KASSERT(sc != NULL, ("bugdemo: softc missing"));
```

`kgdb` 打印源代码行。从这里你可以用 `info locals` 检查局部变量，用 `print sc` 直接查看 `sc`，或用 `list` 列出周围的源代码：

```console
(kgdb) print sc
$1 = (struct bugdemo_softc *) 0x0
```

这告诉我们 `sc` 确实是 NULL，确认了 panic 消息。现在我们可以追查为什么，这通常意味着沿栈向上查找 `sc` 应该在哪里被设置：

```console
(kgdb) frame 6
```

等等。`frame N`、`print VAR`、`list` 的序列是 `kgdb` 分析的基本功。这是任何 gdb 用户与任何崩溃程序进行的相同对话，适配到了内核。

### 有用的kgdb命令

除了 `bt` 和 `frame`，少量命令即可覆盖大多数调试会话。

- `info threads` 列出转储系统中的所有线程。在现代内核中这可能有数百个条目。每个都有编号和状态。
- `thread N` 切换到特定线程，就好像该线程是 panic 的那个。当发生死锁且 panic 线程不是持有问题锁的线程时，这是必不可少的。
- `bt full` 打印带每帧局部变量的回溯。这通常是查看 panic 涉及函数状态的最快方式。
- `info locals` 显示当前帧中的局部变量。
- `print *SOMETHING` 解引用指针并打印其指向结构的内容。
- `list` 显示当前行周围的源代码；`list FUNC` 按名称显示函数的源代码。

还有很多，记录在 `gdb(1)` 中，但这些是驱动程序作者最常使用的命令。

### 在转储中遍历struct thread

panic 回溯回答了"崩溃在哪里发生？"但很少回答"谁在做什么？"。内核在 `struct thread` 中保存了每个活动线程的密集记录，一旦转储在 `kgdb` 中打开，我们就可以直接读取该记录。对驱动程序作者来说，价值是具体的：`struct thread` 的字段告诉你内核崩溃时该线程正在执行什么任务、它在等待什么锁、它属于哪个进程，以及 panic 发生时它是否仍在你的代码中。

`struct thread` 定义在 `/usr/src/sys/sys/proc.h` 中。它是一个很大的结构，因此我们不必阅读每个字段，而是聚焦于对驱动程序调试最重要的小部分字段。在 `kgdb` 中查看这些字段的最快方式是取当前线程并解引用它：

```console
(kgdb) print *(struct thread *)curthread
```

在 panic 的 CPU 上 `curthread` 已经是正确的，但你也可以从 `info threads` 列表中到达特定线程。`kgdb` 为每个线程顺序编号。知道编号后，`thread N` 切换上下文，然后用 `print *$td`（或者如果有原始地址，用 `print *(struct thread *)0xADDR`）打印结构。

需要了解的字段如下。`td_name` 是线程的简短可读名称，通常由 `kthread_add(9)` 或生成它的用户空间程序设置。当驱动程序创建自己的内核线程时，这就是显示出来的名称。`td_tid` 是内核分配的数字线程标识符；用户空间中的 `ps -H` 显示相同的数字。`td_proc` 是指向所属进程的指针，这使我们能够访问更大的 `struct proc` 获取更多上下文。`td_flags` 携带 `TDF_*` 位字段，记录调度器和调试器状态；定义位于 `/usr/src/sys/sys/proc.h` 中结构旁边，许多 panic 可以通过读取这些位来部分解释。`td_lock` 是当前保护此线程调度器状态的自旋互斥锁。在运行的内核中它几乎总是 CPU 本地锁；在转储中，`td_lock` 指向意外地址是某物损坏了此线程调度器视图的强烈暗示。

当 panic 涉及睡眠或等待时，另外两个字段是决定性的。`td_wchan` 是"等待通道"，线程正在其上睡眠的内核地址。`td_wmesg` 是描述原因的简短人类可读字符串（例如 `"biord"` 表示等待 buf 读取的线程，或 `"select"` 表示在 `select(2)` 内部的线程）。如果 panic 时有线程正在睡眠，这两个字段告诉你每个线程在等待什么。`td_state` 是 TDS_* 状态值（定义在 `struct thread` 正下方）；它告诉你线程在崩溃瞬间是正在运行、可运行还是被抑制的。

对于锁 bug，`td_locks` 计算线程当前持有的非自旋锁数量，`td_lockname` 记录线程当前阻塞的锁的名称（如果有的话）。如果一个线程在 panic 时 `td_locks` 非零，说明该线程在崩溃时持有一个或多个睡眠锁：当 panic 消息是 `mutex not owned` 或 `Lock (sleep mutex) ... is not sleepable` 时这是有用的上下文。

一个提取这些字段的简短 `kgdb` 会话可能看起来像这样：

```console
(kgdb) thread 42
[Switching to thread 42 ...]
(kgdb) set $td = curthread
(kgdb) print $td->td_name
$2 = "bugdemo_worker"
(kgdb) print $td->td_tid
$3 = 100472
(kgdb) print $td->td_state
$4 = TDS_RUNNING
(kgdb) print $td->td_wmesg
$5 = 0x0
(kgdb) print $td->td_locks
$6 = 1
(kgdb) print $td->td_proc->p_pid
$7 = 0
(kgdb) print $td->td_proc->p_comm
$8 = "kernel"
```

解读如下：线程 42 是一个名为 `bugdemo_worker` 的内核线程，在 panic 发生时正在运行，没有在睡眠等待任何东西（`td_wmesg` 为 NULL），并且它仍持有一个睡眠锁。所属进程是 pid 为 0、命令名为 `kernel` 的内核进程，这是纯内核线程的预期持有者。有趣的事实是 `td_locks == 1`，因为它告诉我们该线程在 panic 时持有一把锁；后续用 DDB 中的 `show alllocks` 或在文件锁相关时用 `show lockedvnods` 可以精确定位是哪把锁。

### 在转储中遍历struct proc

每个线程属于一个 `struct proc`，定义在 `/usr/src/sys/sys/proc.h` 中 `struct thread` 旁边。`struct proc` 携带进程范围的上下文：身份、凭证、地址空间、打开的文件、父进程关系。对于驱动程序 bug，其中一些字段特别有用。

`p_pid` 是进程标识符，与用户空间中 `ps` 看到的数字相同。`p_comm` 是进程命令名，截断为 `MAXCOMLEN` 字节。它们一起告诉你哪个用户空间进程触发了 panic 的内核路径。`p_state` 是 PRS_* 进程状态，让你区分新 fork 的进程、运行中的进程和僵尸进程。`p_numthreads` 告诉你这个进程有多少线程；对于一个调用了你的驱动的多线程用户程序，这个计数可能令人惊讶。`p_flag` 持有 P_* 标志位，编码了跟踪、记账和单线程等属性；`/usr/src/sys/sys/proc.h` 在标志块附近记录了每个位。

三个指针为你提供更大的上下文。`p_ucred` 引用进程凭证，当 panic 可能与你驱动程序执行的权限检查有关时很有用。`p_vmspace` 指向地址空间，当 panic 涉及一个原来属于意外进程的用户指针时很重要。`p_pptr` 指向父进程；用 `p_pptr->p_pptr` 沿此链遍历最终到达 `initproc`，即每个用户空间进程的祖先。

在 `kgdb` 中从线程到其进程的简短遍历如下：

```console
(kgdb) set $p = curthread->td_proc
(kgdb) print $p->p_pid
$9 = 3418
(kgdb) print $p->p_comm
$10 = "devctl"
(kgdb) print $p->p_state
$11 = PRS_NORMAL
(kgdb) print $p->p_numthreads
$12 = 4
(kgdb) print $p->p_flag
$13 = 536871424
```

现在我们知道 panic 发生在一个 pid 为 3418 的用户空间 `devctl` 进程运行时，该进程有四个线程，其标志位通过 `/usr/src/sys/sys/proc.h` 中的 P_* 常量解码后会告诉我们它是否正在被跟踪、被记账或正处于 exec 中间。标志整数本身看起来是不透明的，但在 `kgdb` 中你可以通过类型转换或使用 `info macro P_TRACED` 让枚举般的 P_* 宏进行解码。

对于暴露字符设备的驱动程序，`p_fd` 也值得了解。它指向调用你驱动程序的进程的文件描述符表，在高级会话中你可以遍历它来找出调用是从哪个描述符进来的。这通常超出了首轮崩溃分析的需要，但这个机制值得记住，以应对那些依赖于用户空间如何打开设备的罕见 bug。

通过 `struct thread` 和 `struct proc`，你可以从一个初看只有 panic 消息和回溯的转储中重建出惊人的大量上下文。代价是仔细阅读一次 `/usr/src/sys/sys/proc.h`；之后，同样的词汇在你余下职业生涯的每次调试会话中都可使用。

### 对运行中的内核使用kgdb

到目前为止我们一直将 `kgdb` 作为事后分析工具对待：打开转储、离线探索、按自己的节奏思考。`kgdb` 还有第二种模式，它通过 `/dev/mem` 而非保存的转储附加到运行中的内核。这种模式功能强大，但也是整个调试工具箱中最容易被误用的工具，所以我们将在明确警告下讨论它。

调用方式与事后形式几乎相同，只是"核心"是 `/dev/mem`：

```console
# kgdb /boot/kernel/kernel /dev/mem
```

实际发生的是 `kgdb` 使用 libkvm 库通过 `/dev/mem` 读取内核内存。该接口记录在 `/usr/src/lib/libkvm/kvm_open.3` 中，它明确区分了"核心"参数可以是 `savecore(8)` 产生的文件或 `/dev/mem`，在后一种情况下目标是当前运行的内核。

这确实有用。你可以检查全局变量、遍历锁图、查看进行中的 I/O、确认你刚设置的 sysctl 是否生效。你无需重启、无需中断服务、无需重现崩溃就能做到这些。在托管长时间运行测试的开发系统上，这通常是回答"驱动程序现在到底在做什么？"的最快方式。

风险是真实存在的。首先，内核在你读取时仍在运行。数据结构在你手下变化。你开始遍历的链表可能在中途被移除一个条目；你打印的计数器可能在你请求和 `kgdb` 打印之间被递增；你跟随的指针可能在解引用完成前被重新赋值。与转储不同，你在读取一个移动中的目标，有时你会看到瞬时不一致的状态。

其次，对运行中内核使用 `kgdb` 在实际使用中严格是只读的。你可以读取内存、打印结构、遍历数据，但你绝不能通过此路径写入内核内存。libkvm 接口不提供锁或屏障，不协调的写入会与内核本身竞争。将通过 `/dev/mem` 的每个操作视为检查，而非修改。如果你想改变运行中的内核状态，使用 `sysctl(8)` 或 `sysctl(3)`，或加载模块，或从控制台使用 DDB。这些机制被设计为与内核其余部分协调；通过 `/dev/mem` 的原始写入则不是。

第三，干扰并非为零。通过 `/dev/mem` 读取会产生 TLB 流量，对大型结构来说成本是可测量的。如果你同时在做性能分析，请相应地归因噪声。

最后，访问 `/dev/mem` 需要 root 权限，原因显而易见：能读取 `/dev/mem` 的任何东西都能读取内核曾经持有的任何秘密。在生产系统上，限制这种访问是一个安全问题，关于谁可以在运行中的内核上运行 `kgdb` 的策略应该反映这一点。

考虑到这些警告，指导原则是明确的。对于任何你想从容进行、想与同事共享状态、或一致性很重要的会话，优先使用崩溃转储。对于快速、只读地查看运行中系统的小问题，且重启成本很高时，优先使用实时 `kgdb` 会话。如有疑问，用 `sysctl debug.kdb.panic=1`（如果系统可消耗）或 `dumpon` 和一个有意触发事件来获取转储，然后在冻结的快照上做分析。快照明天还在；运行中的内核不会。

### 关于符号和模块的说明

当 panic 的驱动程序是可加载模块时，`kgdb` 还需要该模块的调试信息。如果模块在 `/boot/modules/bugdemo.ko` 并且是用 `DEBUG_FLAGS=-g` 构建的，调试符号就嵌入在内。`kgdb` 在解析该模块中的帧时会自动加载它们。

如果模块位于非标准位置，你可能需要告诉 `kgdb` 在哪里找到其调试信息：

```console
(kgdb) add-symbol-file /path/to/bugdemo.ko.debug ADDRESS
```

其中 `ADDRESS` 是模块的加载地址，你可以在 `kldstat(8)` 输出中找到。实际上这很少需要，因为现代 FreeBSD 系统上 `kgdb` 默认会查找正确的位置。

你确实需要避免的是混合内核。如果运行中的内核和调试内核来自不同的构建，符号将不匹配，`kgdb` 会显示令人困惑或错误的信息。从同一源码树重建两者，或保持匹配对。在开发系统上这通常不是问题，因为你同时构建和安装两者。

### 关于转储的结束语

崩溃转储之所以有价值，是因为它保存了 panic 瞬间的内核状态。与运行中的系统不同——每次读取都会扰动状态——转储是一个冻结的快照。你可以随意检查它，明天再回来，与同事分享，或将状态与源代码进行对比。即使你已经转向其他 bug，来自有趣故障的转储也值得保留，因为它通常是那个确切事件序列的唯一记录。

有了 panic 机制和转储分析的基础，我们可以转向让调试真正舒适的内核配置选择。这是第 4 节的主题。

## 4. 构建友好的调试内核环境

到目前为止我们学到的所有内容都依赖于启用了正确的内核选项。标准的 `GENERIC` 内核是生产配置。它针对速度进行了优化，不发布调试信息，也不包含能捕获许多驱动程序 bug 的检查。对于本章的工作，我们需要相反的东西：一个虽然慢但彻底的内核，携带完整的调试符号，主动寻找 bug 而不是信任驱动程序会正常行为。FreeBSD 称之为 `GENERIC-DEBUG`，设置它是本节的主题。

我们将逐步介绍构建和安装调试内核，然后详细查看每个有趣的选项，包括调试器后端（`DDB`、`GDB`）、不变式检查（`INVARIANTS`、`WITNESS`）、内存调试器（`DEBUG_MEMGUARD`、`DEBUG_REDZONE`），以及让你从键盘进入调试器的控制台控制。

### 构建GENERIC-DEBUG

在有 `/usr/src/` 的 FreeBSD 14.3 系统上，构建调试内核是一个三步操作。从 `/usr/src/` 开始：

```console
# make buildkernel KERNCONF=GENERIC-DEBUG
# make installkernel KERNCONF=GENERIC-DEBUG
# reboot
```

`buildkernel` 步骤比发布版构建花费更长时间，因为需要生成调试信息并编译更多的检查。在适中的四核虚拟机上通常需要二十到三十分钟。`installkernel` 将结果放入 `/boot/kernel/` 并将之前的内核保留在 `/boot/kernel.old/`，这是新内核无法启动时的安全网。

重启后你可以用 `uname -v` 确认运行的内核：

```console
# uname -v
FreeBSD 14.3-RELEASE-p2 #0: ...
```

`#0` 表示这是一个本地构建的内核。你还可以通过读取 `sysctl debug` 条目来确认调试选项处于活动状态，我们稍后会回来讨论这一点。

### GENERIC-DEBUG启用了什么

正如我们在第 3 节中看到的，`GENERIC-DEBUG` 是一个精简配置，简单地包含 `GENERIC` 和 `std.debug`。有趣的内容在 `std.debug` 中，值得完整阅读，因为它记录了内核对好的调试选项的看法。在 FreeBSD 14.3 源码树中，该文件位于 `/usr/src/sys/conf/std.debug`，核心选项如下：

```text
options         BUF_TRACKING
options         DDB
options         FULL_BUF_TRACKING
options         GDB
options         DEADLKRES
options         INVARIANTS
options         INVARIANT_SUPPORT
options         QUEUE_MACRO_DEBUG_TRASH
options         WITNESS
options         WITNESS_SKIPSPIN
options         MALLOC_DEBUG_MAXZONES=8
options         VERBOSE_SYSINIT=0
options         ALT_BREAK_TO_DEBUGGER
```

还有一些网络、USB、HID 和 CAM 的子系统特定调试标志，我们不需要深入讨论。让我们依次查看每个与驱动相关的选项。

注意 `std.debug` *不*包含的一件事：`makeoptions DEBUG=-g`。那行在 `GENERIC` 本身中，位于 `/usr/src/sys/amd64/conf/GENERIC` 顶部附近。发布版 `GENERIC` 内核已经用 `-g` 构建，因为发布工程过程希望即使 `INVARIANTS` 和 `WITNESS` 关闭时调试信息也可用。`GENERIC-DEBUG` 通过其 `include "GENERIC"` 继承了这一点。

### makeoptions DEBUG=-g

这将 `-g` 传递给编译器处理每个内核文件，产生一个带有完整 DWARF 调试信息的内核。`kgdb` 使用这些调试信息将地址映射回源代码行。没有 `-g`，`kgdb` 仍然可以显示函数名，但无法显示崩溃发生的源代码行，`print someVariable` 变成 `print *(char *)0xffffffff...`，没有符号名。

代价是内核二进制文件更大。在 amd64 上，调试版 `GENERIC-DEBUG` 内核是非调试 `GENERIC` 内核大小的数倍。对于开发虚拟机这不重要。对于生产系统，通常是将调试信息保留在单独的文件中（`/usr/lib/debug/boot/kernel/kernel.debug`），而运行中的内核是剥离过的。

### INVARIANTS 和 INVARIANT_SUPPORT

我们在第 2 节中见过它们。`INVARIANTS` 激活 `KASSERT` 和散布在内核中的许多其他运行时检查。`/usr/src/sys/` 中的函数有 `#ifdef INVARIANTS` 块来检查诸如"此链表格式是否正确"、"此指针是否指向有效区域"或"此引用计数是否非零"之类的东西。启用 `INVARIANTS` 后，这些检查在运行时触发。没有它，它们被编译出去。

这些检查消耗 CPU 周期。粗略估计，在典型的 FreeBSD 14.3-amd64 硬件上，繁忙的 `INVARIANTS` 内核比发布版内核大约慢百分之五到二十，在分配密集的工作负载上有时更多。这就是为什么 `INVARIANTS` 不在 `GENERIC` 中启用的原因。对于驱动程序开发，为了它捕获的 bug，这个开销是值得的。参见附录 F 获取在你自己硬件上测量此比例的可重现工作负载。

`INVARIANT_SUPPORT` 编译进断言调用的辅助例程，但不激活基础内核代码中的断言。如前所述，它允许用 `INVARIANTS` 构建的模块加载到没有 `INVARIANTS` 的内核中。你几乎总是需要两者。

### WITNESS：锁顺序验证器

`WITNESS` 是 FreeBSD 兵器库中最有效的调试工具之一。它跟踪内核中的每一次锁操作和每一个锁依赖，如果它看到可能导致死锁的锁序就会发出警告。因为死锁是一类极难通过其他方式捕获的 bug，`WITNESS` 对于任何获取多个锁的驱动程序都是不可或缺的。

`WITNESS` 的工作方式值得理解。每次线程获取锁时，`WITNESS` 记录该线程已经持有哪些其他锁。从这些观察中它构建一个锁序图："锁 A 在锁 B 之前被持有过"等等。如果图中有环，那就是一个潜在的死锁，`WITNESS` 在控制台上打印带有违规获取回溯的警告。

输出看起来像这样：

```text
lock order reversal:
 1st 0xfffff80003abc000 bugdemo_sc_mutex (bugdemo_sc_mutex) @ /usr/src/sys/modules/bugdemo/bugdemo.c:203
 2nd 0xfffff80003def000 sysctl_lock (sysctl_lock) @ /usr/src/sys/kern/kern_sysctl.c:1842
stack backtrace:
 #0 kdb_backtrace+0x71
 #1 witness_checkorder+0xc95
 #2 __mtx_lock_flags+0x8f
 ...
```

解读如下：你的驱动程序的 `bugdemo_sc_mutex` 先被获取，然后后来观察到另一个线程先获取 `sysctl_lock` 再获取 `bugdemo_sc_mutex`。这是一个潜在的死锁，因为足够的并发活动可能使两个线程互相等待。修复方法始终相同：在所有获取两把锁的路径上建立一致的锁序，并坚持它。

`WITNESS` 并不便宜。它为每次锁获取和释放添加簿记。在我们的实验环境中，在运行锁密集工作负载的繁忙内核上，开销可接近百分之二十；确切数字取决于工作负载的锁量。但它发现的 bug 是那种漏过去会摧毁生产运行时间的 bug，所以投资在开发中是值得的。参见附录 F 获取隔离此开销与基线内核的可重现工作负载。

`WITNESS_SKIPSPIN` 关闭自旋互斥锁上的 `WITNESS`。自旋锁通常寿命短且对性能关键，所以检查它们在最要紧的地方增加了开销。默认是检查它们，但 `std.debug` 禁用了该检查以保持内核可用。如果你专门追踪自旋锁 bug，可以重新启用它。

### 一个实战竞态条件演练：bugdemo 中的锁序 Bug

抽象地阅读 `WITNESS` 是一回事；在你编写的驱动程序中捕获一个真实的锁序反转是另一回事。本小节将完成一个完整循环：我们在 `bugdemo` 中引入一个故意的排序 bug，在 `GENERIC-DEBUG` 内核上运行它，阅读 `WITNESS` 报告，然后修复 bug。演练很短，但这个模式在你将来调试的每个死锁中都会重复。

假设我们的 `bugdemo` 驱动程序随着功能增加而拥有了两把锁。`sc_mtx` 保护每个单元的状态，`cfg_mtx` 保护跨单元共享的配置数据。驱动程序的大部分代码已经按照"先状态、后配置"的顺序获取它们，这是一个合理的选择，作者在 `bugdemo_ioctl` 和读写入口点中遵循了这一点。但最近的一个 sysctl 处理程序写得匆忙，先获取了配置锁来验证值，然后才去获取状态锁来应用它。源代码中，相关的两段摘录如下：

```c
/* bugdemo_ioctl: established ordering, state then config */
mtx_lock(&sc->sc_mtx);
/* inspect per-unit state */
mtx_lock(&cfg_mtx);
/* adjust shared config */
mtx_unlock(&cfg_mtx);
mtx_unlock(&sc->sc_mtx);
```

```c
/* bugdemo_sysctl_set: new path, config then state */
mtx_lock(&cfg_mtx);
/* validate new value */
mtx_lock(&sc->sc_mtx);
/* propagate into per-unit state */
mtx_unlock(&sc->sc_mtx);
mtx_unlock(&cfg_mtx);
```

两条路径单独看都没问题。问题在于它们合在一起形成了一个环。如果线程 A 进入 `bugdemo_ioctl` 并获取 `sc_mtx`，而线程 B 同时进入 `bugdemo_sysctl_set` 并获取 `cfg_mtx`，每个线程现在都在等待对方持有的锁。这就是经典的 AB-BA 死锁。它不一定在每次运行中都触发；取决于时序。`WITNESS` 就是那个拒绝等待罕见生产故障来发现它的工具。

在 `GENERIC-DEBUG` 内核上，只要两种顺序都被观察到，反转就会被捕获，即使尚未发生实际死锁。控制台消息有特定的格式。使用 `/usr/src/sys/kern/subr_witness.c` 中 `witness_output` 打印的格式，它会为每个涉及的锁打印指针、锁名、见证名、锁类和源位置，真实的报告如下：

```text
lock order reversal:
 1st 0xfffff80012345000 bugdemo sc_mtx (bugdemo sc_mtx, sleep mutex) @ /usr/src/sys/modules/bugdemo/bugdemo.c:412
 2nd 0xfffff80012346000 bugdemo cfg_mtx (bugdemo cfg_mtx, sleep mutex) @ /usr/src/sys/modules/bugdemo/bugdemo.c:417
lock order bugdemo cfg_mtx -> bugdemo sc_mtx established at:
 #0 witness_checkorder+0xc95
 #1 __mtx_lock_flags+0x8f
 #2 bugdemo_sysctl_set+0x7a
 #3 sysctl_root_handler_locked+0x9c
 ...
stack backtrace:
 #0 kdb_backtrace+0x71
 #1 witness_checkorder+0xc95
 #2 __mtx_lock_flags+0x8f
 #3 bugdemo_ioctl+0xd4
 ...
```

每个 `1st` 和 `2nd` 行携带四条信息。指针（`0xfffff80012345000`）是锁对象在内核内存中的地址。第一个字符串是实例名称，在锁初始化时设置。括号中的两个字符串是锁类的 `WITNESS` 名称和锁类本身，在此例中是 `sleep mutex`。路径和行号是锁沿此反转顺序最后被获取的位置。`lock order ... established at:` 后面的块显示了首次教会 `WITNESS` 此（现在被违反的）顺序的早期回溯，最终的 `stack backtrace` 显示了违反它的当前调用路径。

读完所有这些，诊断是即刻的。驱动程序在其正常路径中建立了 `sc_mtx -> cfg_mtx` 的顺序，而 `bugdemo_sysctl_set` 刚刚采用了 `cfg_mtx -> sc_mtx` 的顺序。两条路径都是我们的。修复是选择一种顺序（这里是已建立的顺序），并重写违规路径以匹配：

```c
/* bugdemo_sysctl_set: corrected to follow house ordering */
mtx_lock(&sc->sc_mtx);
mtx_lock(&cfg_mtx);
/* validate new value and propagate in one atomic window */
mtx_unlock(&cfg_mtx);
mtx_unlock(&sc->sc_mtx);
```

如果锁定区域需要更窄，一种常见的模式是在 `sc_mtx` 下读取状态、释放它、在无锁状态下验证，然后按照既定顺序重新获取以应用更改。无论哪种方式，顺序都在驱动级别固定，而非在调用站点。一个有用的习惯是在锁声明附近的注释中记录顺序，这样未来的贡献者就不必重新发现它。

修复后，在同一调试内核上重建 `bugdemo` 并重新运行触发测试不会产生更多 `WITNESS` 输出。如果反转再次出现，`WITNESS` 还支持在 DDB 中用 `show all_locks` 交互式查询图，即使没有完整的反转报告也能显示当前状态；对于更深入的内省，`/usr/src/sys/kern/subr_witness.c` 的源代码是簿记和报告格式的权威解释。

### 通过 lockstat(1) 看同一个 Bug

`WITNESS` 告诉你顺序是错误的。它不告诉你每把锁实际上被竞争的频率、每次获取等待多长时间、或哪些调用者对某把锁施加的压力最大。这些是关于竞争而非正确性的问题，`lockstat(1)` 就是回答它们的工具。

`lockstat(1)` 是一个基于 DTrace 的内核锁分析器。它通过在锁原语的入口和出口点插桩来工作，报告摘要信息，包括自适应互斥锁上的旋转时间、sx 锁上的睡眠时间，以及被请求时的持有时间。经典的调用方式是 `lockstat sleep N`，它收集 N 秒的数据然后打印摘要。

如果我们在施加压力于两条路径的工作负载下运行有 bug 的 `bugdemo`（一个打开多个单元节点并同时紧凑循环操作 sysctl 的用户空间小程序），并用 `lockstat` 分析五秒，FreeBSD 系统上的输出大致如下：

```console
# lockstat sleep 5

Adaptive mutex spin: 7314 events in 5.018 seconds (1458 events/sec)

Count indv cuml rcnt     nsec Lock                   Caller
-------------------------------------------------------------------------------
3612  49%  49% 0.00     4172 bugdemo sc_mtx         bugdemo_ioctl+0xd4
2894  40%  89% 0.00     3908 bugdemo cfg_mtx        bugdemo_sysctl_set+0x7a
 412   6%  95% 0.00     1205 bugdemo sc_mtx         bugdemo_read+0x2f
 220   3%  98% 0.00      902 bugdemo cfg_mtx        bugdemo_ioctl+0xe6
 176   2% 100% 0.00      511 Giant                  sysctl_root_handler_locked+0x4d
-------------------------------------------------------------------------------

Adaptive mutex block: 22 events in 5.018 seconds (4 events/sec)

Count indv cuml rcnt     nsec Lock                   Caller
-------------------------------------------------------------------------------
  14  63%  63% 0.00   184012 bugdemo sc_mtx         bugdemo_sysctl_set+0x8b
   8  36% 100% 0.00    41877 bugdemo cfg_mtx        bugdemo_ioctl+0xe6
-------------------------------------------------------------------------------
```

每个表遵循相同的列约定：`Count` 是此类事件的观测数量，`indv` 是此类事件在该类别中的百分比，`cuml` 是累计百分比，`rcnt` 是平均引用计数（互斥锁始终为 1），`nsec` 是以纳秒为单位的平均持续时间，最后两列标识锁实例和调用者。标题行 `Adaptive mutex spin` 表示通过短时间旋转解决的竞争；`Adaptive mutex block` 表示实际迫使线程在互斥锁上睡眠的竞争。这些标题和列布局是标准的 `lockstat` 输出；格式记录在 `/usr/src/cddl/contrib/opensolaris/cmd/lockstat/lockstat.1` 中，该手册页末尾有实际示例。

有两点值得注意。首先，`bugdemo sc_mtx` 和 `bugdemo cfg_mtx` 都出现在表的两个方向中：sysctl 路径在 `sc_mtx` 上阻塞（阻塞表第 1 行），ioctl 路径在 `cfg_mtx` 上阻塞（第 2 行）。这就是同一排序 bug 的竞争特征，从另一面看到的。`WITNESS` 告诉我们顺序不安全；`lockstat` 告诉我们在此工作负载下不安全的顺序也在消耗真实时间。

其次，应用前一子节的修复后，`lockstat` 变成了一个验证工具：用相同工作负载重新运行，`Adaptive mutex block` 表应该大幅缩小，因为两条路径之间的相互等待消失了。如果不缩小，说明我们修复了顺序但创建了纯粹的竞争问题，下一步是缩小临界区而非改变顺序。

有用的 `lockstat` 选项（除了默认）包括 `-H` 监视持有事件（锁被持有多长时间，而不仅仅是竞争）、`-D N` 每个表只显示前 N 行、`-s 8` 每行包含八帧栈跟踪、`-f FUNC` 过滤单个函数。对于驱动程序工作，在运行针对性测试时执行 `lockstat -H -s 8 sleep 10` 是一个极其高效的默认选择。

### 结合阅读 WITNESS 和 lockstat

`WITNESS` 和 `lockstat` 是互补的。`WITNESS` 是正确性工具：它检测最终会产生死锁的 bug，无论当前工作负载是否恰好命中它们。`lockstat` 是性能工具：它量化当前流量在多大程度上涉及每把锁以及流量等待多长时间。同一驱动路径经常出现在两者中，两个视图合在一起通常是决定性的。

当驱动程序的锁超过第一个时，一个有用的纪律是将两种工具都纳入常规。在开发期间运行 `GENERIC-DEBUG`，这样 `WITNESS` 在每个新代码路径执行的瞬间就能看到它。定期在现实工作负载上运行 `lockstat`，查看是否有任何锁正在成为瓶颈，即使其顺序是正确的。一把通过 `WITNESS` 且在 `lockstat` 中显示低竞争的锁，是你基本可以不用再担心的锁。一把通过 `WITNESS` 但在 `lockstat` 输出中占主导地位的锁，是等待重构的性能问题，而非正确性 bug。一把未通过 `WITNESS` 的锁，无论 `lockstat` 怎么说，都是一个 bug。

带着这个框架，我们可以继续查看其他能暴露不同 bug 类别的调试内核选项。

### MALLOC_DEBUG_MAXZONES

FreeBSD 的内核内存分配器（`malloc(9)`）将类似的分组合并到区域中以加快速度。`MALLOC_DEBUG_MAXZONES=8` 增加了 `malloc` 使用的区域数量，将分配分散到更多不同的内存区域。实际效果是释放后使用和无效释放 bug 更可能落在与原始分配不同的区域中，使它们更容易被检测到。

这是一个低成本的选项。它在调试内核中始终启用。

### ALT_BREAK_TO_DEBUGGER and BREAK_TO_DEBUGGER

这两个选项控制用户如何从控制台进入内核调试器。`BREAK_TO_DEBUGGER` 启用传统的 `Ctrl-Alt-Esc` 或串行 BREAK 序列。`ALT_BREAK_TO_DEBUGGER` 启用替代序列，输入为 `CR ~ Ctrl-B`，这在网络控制台（ssh、virtio_console 等）上发送真正的 BREAK 不方便时很有用。

`GENERIC` 附带 `BREAK_TO_DEBUGGER` 启用。`GENERIC-DEBUG` 额外添加 `ALT_BREAK_TO_DEBUGGER`。如果你在串行控制台上，任一序列都可以将你送入 `DDB`。在 `DDB` 中你可以检查内核状态、设置断点，以及选择继续执行或 panic。

这在开发期间是一个重要的便利。一个挂起系统而不产生 panic 的驱动程序可以通过命令式进入调试器来调查。

### DEADLKRES: The Deadlock Detector

`DEADLKRES` 启用死锁解析器，它是一个定期检查线程是否在不可中断等待中卡住过久的线程。如果发现这样的线程，它打印警告并可选择 panic。它补充了 `WITNESS`，捕获 `WITNESS` 未预测到的死锁，这在锁图无法静态遍历时发生（例如，当锁通过通用锁定 API 按地址获取时）。

`DEADLKRES` 在实践中有一些误报，特别是对于在重负载下长时间运行的文件系统 I/O 操作。阅读警告并判断它是否是真正的死锁，是本章正在教授的调试技能的一部分。

### BUF_TRACKING

`BUF_TRACKING` 记录缓冲区缓存中每个缓冲区操作的简短历史。当发现损坏时，可以打印缓冲区的历史，显示哪些代码路径以何种顺序触及了它。这对存储驱动程序 bug 很有用，但在其他驱动程序中较少需要。

### QUEUE_MACRO_DEBUG_TRASH

`queue(3)` 宏（`LIST_`、`TAILQ_`、`STAILQ_` 等）在内核中被广泛用于链表。当元素从列表中移除时，通常的行为是保持元素的指针不变。`QUEUE_MACRO_DEBUG_TRASH` 用可识别的垃圾值覆盖它们。之后任何试图解引用这些指针的尝试都会以可识别的方式崩溃，而不是静默地损坏列表。

这是一个便宜的选项，能捕获一类非常常见的 bug：在释放元素之前忘记将其从列表中移除，然后发现列表被损坏。

### Memory Debuggers: DEBUG_MEMGUARD and DEBUG_REDZONE

另外两个值得关注的选项是 `DEBUG_MEMGUARD` 和 `DEBUG_REDZONE`。它们不属于 `std.debug`，但通常用于内存调试会话。

`DEBUG_MEMGUARD` 是一个专用分配器，可以替换特定 `malloc(9)` 类型的分配。它的想法很简单：不从 slab 中返回一块内存，而是返回由专用页面支持的内存。当内存被释放时，页面不归还到池中；它们被取消映射，因此任何后续访问都会触发页面错误。分配周围的页面也被标记为不可访问，因此任何超出分配边界的读取或写入也会触发错误。这将释放后使用、缓冲区越界和缓冲区下溢 bug 从静默的损坏者转变为立即 panic，并带有直接指向误用点的回溯。

代价是每次分配现在至少消耗一个完整的虚拟内存页面（加上管理开销），每次释放都会消耗一个未映射的页面。因此，`memguard(9)` 通常一次只对一个 malloc 类型启用。

相关的头文件是 `/usr/src/sys/vm/memguard.h`，配置出现在 `/usr/src/sys/conf/NOTES` 中 `options DEBUG_MEMGUARD` 行。我们将在第 6 节详细使用 `memguard(9)`。

`DEBUG_REDZONE` 是一个更轻量的内存调试器，在每个分配前后放置保护字节。当分配被释放时，检查保护字节，如果被修改就报告损坏。它不捕获释放后使用，但非常擅长捕获缓冲区越界和下溢。参见 `/usr/src/sys/conf/NOTES` 中的 `options DEBUG_REDZONE` 行获取配置。

`DEBUG_MEMGUARD` 和 `DEBUG_REDZONE` 都消耗内存。对于开发虚拟机上的调试内核，两者通常都启用。对于大型生产服务器，两者都不启用。

### KDB, DDB, and GDB Together

我们在本章中多次引用了这三个选项。让我们澄清它们的区别，因为它让许多初学者感到困惑。

`KDB` 是内核调试器框架。它是管道。它定义了内核其余部分在 panic 或进入调试器事件发生时调用的入口点。它还为后端定义了接口。

`DDB` 和 `GDB` 是两个这样的后端。`DDB` 是内核内交互式调试器。当你触发 `KDB_ENTER` 且 `DDB` 是选定的后端时，你被投入控制台上的交互式提示符。`DDB` 有一小组命令：`bt`、`show`、`print`、`break`、`step`、`continue` 等等。它很原始但自包含：不需要其他机器。

`GDB` 是远程后端。当你触发 `KDB_ENTER` 且 `GDB` 是选定的后端时，内核等待远程 GDB 客户端通过串行线或网络连接附加。客户端在另一台机器上运行并通过称为 GDB 远程串行协议的协议发送命令。这更加灵活，因为客户端有完整的 `gdb`，但它需要第二台机器（或另一个虚拟机）和两者之间的连接。

在实践中，你启用两个后端并在运行时切换。`sysctl debug.kdb.current_backend` 命名活动后端。`sysctl debug.kdb.supported_backends` 列出所有编译进的后端。你可以根据想要的会话类型将 `debug.kdb.current_backend` 设置为 `ddb` 或 `gdb`。这是一个有用的便利，因为编译两者的开销与其灵活性的好处相比可以忽略不计。

`GENERIC` 中的 KDB 支持足以应对大多数 panic。我们将在第 7 节讨论远程调试时使用 `GDB`。

### KDB_UNATTENDED

还有一个值得一提的选项是 `KDB_UNATTENDED`。它使内核在 panic 时跳过进入调试器，直接进行转储和重启。在没有人在控制台旁的生产系统中，这是一个合理的默认值；等待一个永远不会到来的调试器交互毫无意义。在开发中，你通常想要相反的行为：在 panic 后留在 `DDB` 中，这样你可以在状态因重启而丢失之前进行调查。通过运行时 `sysctl debug.debugger_on_panic` 或内核配置中的 `options KDB_UNATTENDED` 设置此选项。

### CTF and Debug Info Paths

调试环境的最后一个部分是 CTF，即紧凑 C 类型格式。CTF 是 DTrace 用来理解内核结构的类型信息的压缩表示。`GENERIC` 包含 `options DDB_CTF`，告诉构建系统为内核生成 CTF 信息。在调试内核上，CTF 信息让 DTrace 能用名称而非十六进制偏移量打印结构字段，使其输出实用性大幅提升。

你可以用 `ctfdump` 确认 CTF 存在：

```console
# ctfdump -t /boot/kernel/kernel | head
```

如果这产生输出，内核有 CTF。如果没有，要么构建没有包含 `DDB_CTF`，要么 CTF 生成工具（`ctfconvert`）未安装。在 FreeBSD 14.3 中两者都是标准的。

对于模块，你需要在环境中设置 `WITH_CTF=1`（或传递给 `make`）来获取模块的 CTF 信息。这就是让 DTrace 理解你驱动程序定义的结构的方式。

### Confirming Your Debug Kernel

当你首次启动调试内核时，花一分钟验证你关心的选项确实处于活动状态。有用的 sysctl：

```console
# sysctl debug.kdb.current_backend
debug.kdb.current_backend: ddb
# sysctl debug.kdb.supported_backends
debug.kdb.supported_backends: ddb gdb
# sysctl debug.debugger_on_panic
debug.debugger_on_panic: 1
# sysctl debug.ddb.
debug.ddb.capture.inprogress: 0
debug.ddb.capture.bufsize: 0
...
```

如果这些打印出合理的值，你的调试内核就配置好了。如果 `debug.kdb.supported_backends` 只列出 `ddb` 但你期望有 `gdb`，你的配置有问题。回去检查 `options GDB` 是否在你的内核配置或 `std.debug` 中。

### Running on Top of the Debug Kernel

在调试内核运行后，本章其余部分的技术就变得可用了。`KASSERT` 真正触发。`WITNESS` 真正抱怨锁序。`DDB` 在你按下进入调试器序列时就在那里。崩溃转储包含 `kgdb` 可以用来显示源代码行的完整调试信息。你从一个信任驱动程序的内核转向了一个主动帮助你证明驱动程序正确的内核。

在调试内核上进行驱动程序开发的一个虽小但有意义的后果是，你会更早地看到驱动程序中的 bug，在它们到达现场之前，而且当它们确实出现时你修复起来更容易。始终在调试内核上开发的纪律——即使你只是在写简单代码——是将随意的业余驱动程序与值得信赖的严肃驱动程序区分开的习惯之一。

有了环境设置，我们可以转向下一类工具：跟踪。与捕获失败的断言不同，跟踪记录发生了什么，这样即使它没有崩溃你也能理解 bug 的形状。这是第 5 节的主题。

## 5. 追踪驱动行为：DTrace、ktrace和ktr(9)

断言捕获出错的地方。跟踪显示正在发生什么。当驱动程序在不崩溃的情况下行为异常，或当你需要理解多个线程之间事件的精确顺序时，跟踪通常是正确的工具。FreeBSD 为内核代码提供了三种互补的跟踪工具：DTrace、`ktrace(1)` 和 `ktr(9)`。每一种有不同的最佳适用场景，驱动程序作者应该知道何时使用哪种。

第 33 章将 DTrace 作为性能测量工具进行了介绍。在这里我们将它作为正确性调试工具重新审视，因为同一个能聚合热函数的框架也能在内核中跟踪 bug。我们还将认识 `ktr(9)`，轻量级内核内跟踪环，以及 `ktrace(1)`，它从用户空间跟踪系统调用。

### DTrace 用于正确性调试

DTrace 是 FreeBSD 的生产级动态跟踪框架。它通过让你将小程序附加到内核各处的探针点来工作。探针是代码中可被插桩的命名点。当探针触发时，脚本运行。如果脚本有有用的东西要记录，它就记录；如果没有，探针基本上是免费的。

第 33 章使用带 `profile` 提供者的 DTrace 进行 CPU 采样。在本章中，我们将使用不同的提供者用于不同目的：`fbt`（函数边界跟踪）跟踪函数的进入和退出、`sdt`（静态定义跟踪）在驱动程序中显式放置的探针点触发、`syscall` 观察用户-内核转换。

让我们逐一来看。

### fbt提供者

`fbt` 提供者为内核中的每个函数入口和退出提供了一个探针。要列出驱动程序中的所有 fbt 探针：

```console
# dtrace -l -P fbt -m bugdemo
```

每个函数产生两个探针，一个 `entry` 和一个 `return`。你可以为任何一个附加动作。调试新 bug 时常见的第一步是简单地查看哪些函数被调用：

```console
dtrace -n 'fbt::bugdemo_*:entry { printf("%s\n", probefunc); }'
```

这打印了 `bugdemo` 模块中任何函数的每次进入，显示了它们的调用顺序。如果你怀疑某个特定函数是否被到达或未被到达，这个一行命令会立即告诉你。

要更深入地查看，你还可以记录参数。`fbt` 探针参数就是函数的参数，可通过 `arg0`、`arg1` 等访问：

```console
dtrace -n 'fbt::bugdemo_ioctl:entry { printf("cmd=0x%lx\n", arg1); }'
```

这里 `arg1` 是 `bugdemo_ioctl` 的第二个参数，即 `ioctl` 命令号。你可以实时观察 ioctl 调用流。

退出探针让你看到返回值：

```console
dtrace -n 'fbt::bugdemo_ioctl:return { printf("rv=%d\n", arg1); }'
```

在返回探针上，`arg1` 是返回值。一连串的 `rv=0` 确认成功。突然出现的 `rv=22`（即 `EINVAL`）告诉你驱动程序拒绝了一个调用。通过结合入口和返回探针，你可以将每个调用与其结果匹配。

### SDT探针：静态定义追踪

`fbt` 很灵活，但给你的是函数边界而非语义事件。如果你想要一个在函数内部特定点触发的探针，代表一个特定事件，你就使用 SDT。SDT 探针在代码中显式放置。禁用时它们几乎不花什么成本，启用时产生你想要的确切信息。

在 FreeBSD 14.3 中，SDT 探针使用 `/usr/src/sys/sys/sdt.h` 中的宏定义。关键宏如下：

```c
SDT_PROVIDER_DEFINE(bugdemo);

SDT_PROBE_DEFINE2(bugdemo, , , cmd__start,
    "struct bugdemo_softc *", "int");

SDT_PROBE_DEFINE3(bugdemo, , , cmd__done,
    "struct bugdemo_softc *", "int", "int");
```

命名约定是 `provider:module:function:name`。开头的 `bugdemo` 是提供者。两个空字符串是模块和函数，我们为驱动程序级探针留空。末尾的名称标识探针。探针名称中的双下划线约定是 DTrace 的惯用法，在用户面对的名称中变成短划线。

`SDT_PROBE_DEFINE` 上的数字后缀表示探针接受多少个参数。字符串参数是这些参数的 C 类型名称，DTrace 用于显示。

要在驱动程序中触发探针：

```c
static void
bugdemo_process(struct bugdemo_softc *sc, struct bugdemo_command *cmd)
{
        SDT_PROBE2(bugdemo, , , cmd__start, sc, cmd->op);

        /* ... actual work ... */

        SDT_PROBE3(bugdemo, , , cmd__done, sc, cmd->op, error);
}
```

`SDT_PROBE2` 和 `SDT_PROBE3` 用给定的参数触发相应的探针。

现在在 DTrace 中你可以监视这些探针：

```console
dtrace -n 'sdt:bugdemo::cmd-start { printf("op=%d\n", arg1); }'
```

注意 `cmd-start` 中的短划线：DTrace 将名称中的双下划线转换为探针规范中的短划线。`arg0` 是 softc，`arg1` 是 op。

SDT 探针对于状态转换特别有用。如果你的驱动程序有三个状态而你想要跟踪序列，在每个转换处定义探针并对其进行聚合：

```console
dtrace -n 'sdt:bugdemo::state-change { @[arg1, arg2] = count(); }'
```

这计算每对 (from_state, to_state) 出现的频率，给出工作负载期间状态机行为的分布。

### 使用DTrace追踪Bug

考虑一个场景。`bugdemo` 驱动程序有时向用户空间返回 `EIO`，但你无法从用户空间判断是哪条代码路径产生了该错误。使用 DTrace，你可以从返回值追溯到来源：

```console
dtrace -n '
fbt::bugdemo_ioctl:return
/arg1 == 5/
{
        stack();
}
'
```

`arg1 == 5` 检查返回值 5，即 `EIO`。当返回匹配时，`stack()` 打印返回点的内核栈跟踪。这告诉你确切是哪条代码路径返回了该错误。

更复杂的版本记录开始时间和持续时间：

```console
dtrace -n '
fbt::bugdemo_ioctl:entry
{
        self->start = timestamp;
}

fbt::bugdemo_ioctl:return
/self->start != 0/
{
        @latency["bugdemo_ioctl", probefunc] = quantize(timestamp - self->start);
        self->start = 0;
}
'
```

这产生 ioctl 的延迟分布，当 bug 表现为异常延迟时很有用。`self->` 表示法是 DTrace 的线程本地存储，作用域限于当前线程。

这些脚本不是完整的程序；它们是你迭代的小型观察。"添加探针、运行工作负载、阅读输出、改进探针"的循环是 DTrace 的优势之一。一个完整的调试会话可能在 bug 的形状变得清晰之前经历十几种脚本变体。

### 理解ktrace(1)

`ktrace(1)` 是另一种不同的工具。它跟踪用户空间进程发出的系统调用及其参数和返回值。它不关注内核的内部行为；它关注的是用户空间与内核之间的接口。当用户空间工具在使用驱动程序时出现了奇怪的行为，`ktrace(1)` 通常是最先应该使用的工具，因为它准确显示了进程向内核请求了什么。

要跟踪一个程序：

```console
# ktrace -t cnsi ./test_bugdemo
# kdump
```

`ktrace` 写入一个二进制跟踪文件（默认为 `ktrace.out`），`kdump` 将其渲染为人类可读的文本。`-t` 标志选择跟踪什么：`c` 表示系统调用、`n` 表示 namei（路径名查找）、`s` 表示信号、`i` 表示 ioctl。对于驱动程序调试，`i` 是最直接有用的。

示例输出：

```text
  5890 test_bugdemo CALL  ioctl(0x3,BUGDEMO_TRIGGER,0x7fffffffe0c0)
  5890 test_bugdemo RET   ioctl 0
  5890 test_bugdemo CALL  read(0x3,0x7fffffffe0d0,0x100)
  5890 test_bugdemo RET   read 32/0x20
```

进程进行了两个系统调用。一个在文件描述符 3 上的 ioctl，命令为 `BUGDEMO_TRIGGER`，成功了。一个在同一 fd 上的 read 返回了 32 字节。如果测试失败，跟踪告诉你内核被请求了什么以及它返回了什么。

注意 `ktrace(1)` 不显示内部内核行为。为此你需要 DTrace 或 `ktr(9)`。但 `ktrace(1)` 是查看用户空间交互的规范方式，与 DTrace 结合使用可以给出完整的图景。

`ktrace(1)` 还可以附加到运行中的进程：

```console
# ktrace -p PID
```

并可以分离：

```console
# ktrace -C
```

对于被长时间运行的守护进程使用的驱动程序，这比在 `ktrace` 下重启守护进程更实用。

### ktr(9): Lightweight In-Kernel Tracing

`ktr(9)` 是 FreeBSD 的内核内跟踪环。它是一个代码可以低成本写入的跟踪条目环形缓冲区。每个条目包括时间戳、CPU 编号、线程指针、格式字符串和最多六个参数。环的大小由 `KTR_ENTRIES` 内核配置选项决定，其内容可以从 `DDB` 或用户空间转储。

当你需要关于时序或顺序的非常细粒度信息时，`ktr(9)` 是正确的工具，特别是在中断上下文中 `printf` 太慢的情况下。因为每个条目很小且写入是无锁的，`ktr(9)` 可以用在热路径中而不扭曲你试图观察的行为。

宏定义在 `/usr/src/sys/sys/ktr.h` 中。常用的有 `CTR0` 到 `CTR6`，按格式字符串后面跟多少个参数而不同。每个宏的第一个参数是类掩码，然后是格式字符串，然后是值：

```c
#include <sys/ktr.h>

static void
bugdemo_process(struct bugdemo_softc *sc, struct bugdemo_command *cmd)
{
        int error;

        CTR2(KTR_DEV, "bugdemo_process: sc=%p op=%d", sc, cmd->op);
        /* ... */
        CTR1(KTR_DEV, "bugdemo_process: done rv=%d", error);
}
```

`CTR2` 向跟踪环写入一个双参数条目。`KTR_DEV` 是类掩码：内核在运行时根据 `debug.ktr.mask` 决定某类的条目是否被记录。编译时，`KTR_COMPILE`（实际编译进去的类集合）控制哪些调用被发射。不在 `KTR_COMPILE` 中的类完全消失，因此你可以将调用永久保留在源代码中而不在类被禁用时为它们付费。

类定义在 `/usr/src/sys/sys/ktr_class.h` 中。常见的包括 `KTR_GEN`（通用）、`KTR_DEV`（设备驱动程序）、`KTR_NET`（网络）等等。对于驱动程序，你通常选择 `KTR_DEV`，或在较大的子系统中，在现有位旁边定义一个新位。

要启用和查看跟踪环：

```console
# sysctl debug.ktr.mask=0x4          # enable KTR_DEV (bit 0x04)
# sysctl debug.ktr.entries
```

并用以下命令转储：

```console
# ktrdump
```

`ktrdump(8)` 通过 `/dev/kmem` 读取内核的跟踪缓冲区并格式化它。输出是按时间排序的条目列表，包含时间戳、CPU、线程和消息。

`ktr(9)` 的美妙之处在于它的低开销。一个跟踪条目本质上是一小撮内存写入。你可以将它们留在代码中，编译进调试内核，在需要时在运行时启用。它们在中断处理程序调试中特别有价值，`printf` 在那里会增加毫秒级的延迟并实际改变被测量的行为。

### 何时使用哪种工具

有了三种跟踪工具，问题是首先该使用哪种。

当 bug 是关于内核在做什么时使用 DTrace，当你需要跨多个事件聚合时、需要过滤时、或探针可以动态放置时使用它。DTrace 是三者中功能最强的，但它需要一个运行中的内核和合理的探针触发频率。

当 bug 是关于用户空间在向内核请求什么时使用 `ktrace(1)`，当症状是错误的返回值或不符合预期的系统调用序列时使用它。`ktrace(1)` 简单、快速，立即显示用户-内核边界。

当你需要尽可能低的开销时、当你跟踪的代码在中断处理程序中时、或当你想要可以在生产环境中以最小风险开启的持久跟踪点时使用 `ktr(9)`。`ktr(9)` 是三者中最原始的，但也是最耐用的。

在实践中，调试会话通常使用两种或三种工具。你可能先用 `ktrace(1)` 查看系统调用序列，然后添加 DTrace 探针缩小哪个驱动函数行为异常，然后添加 `ktr(9)` 条目来确定中断路径中的时序。每种工具回答不同的问题，完整的图景通常需要全部三种。

### Tracing and Production

关于生产环境的一句简短说明。DTrace 在大多数配置中是生产安全的；它的设计明确包含了防止无限循环和防止坏探针崩溃内核的安全措施。你可以在繁忙的生产服务器上运行 DTrace 而不会使其宕机。`ktr(9)` 也是生产安全的，但要注意启用详细类会消耗 CPU。`ktrace(1)` 写入文件，如果不加检查可以无限增长；使用时请注意大小限制。

将这些与崩溃转储、`DDB` 和 `memguard(9)` 做对比，后者都是仅用于开发的工具。这个区别很重要，因为第 7 节将回到在生产机器上什么是安全的问题。现在请记住，跟踪是我们拥有的最轻量级的技术之一，这就是为什么它通常是诊断运行中问题的正确第一步。

有了跟踪工具在手，我们可以转向跟踪和断言倾向于遗漏的 bug：那些在不产生明显症状的情况下损坏状态的内存 bug，直到很久之后才显现。这是第 6 节的领域。

## 6. 查找内存泄漏与无效内存访问

内存 bug 是驱动程序作者面临的最棘手的 bug。它们很少在发生时就能被发现。它们静静地损坏状态，在多次运行中累积，然后在很久以后以看似与原始缺陷完全无关的方式显现。释放后使用可能表现为不同子系统中损坏的结构。缓冲区越界可能覆盖下一个分配，在几分钟后才显示为伪造的字段值。小的泄漏可能经过数天耗尽内存，直到内核最终拒绝分配，系统锁定。

FreeBSD 为这些 bug 提供了一系列工具：`memguard(9)` 用于释放后使用和释放后修改检测，`redzone` 用于缓冲区越界，VM 层的保护页，以及暴露内核内存分配器状态的 sysctl。配合使用，它们可以将一类几乎不可能找到的 bug 转化为一类在误用时立即崩溃的 bug。

### 理解内核内存分配器

要有效使用这些工具，我们需要对内核如何分配内存有一个粗略的心智模型。FreeBSD 有两个主要的分配器，都在 `/usr/src/sys/kern/` 中：

`kern_malloc.c` 实现了 `malloc(9)`，即通用分配器。它是 UMA（通用内存分配器）的薄封装，带有按 malloc 类型的记账。每次分配都计入一个 `struct malloc_type`（通常用 `MALLOC_DEFINE(9)` 声明），这让内核能跟踪每个子系统使用了多少内存。

`subr_vmem.c` 和 `uma_core.c` 实现了较低的层次。UMA 是一个 slab 分配器：它维护每 CPU 缓存和中央 slab，所以大多数分配非常快且无竞争。当驱动程序调用 `malloc(9)` 或 `uma_zalloc(9)` 时，实际发生的事情取决于大小、区域配置和缓存状态。

对于调试而言，实际后果是损坏的分配可能根据其落点的不同而呈现不同的外观。同一个 bug 在不同的内核或不同的负载下可能产生不同的症状，仅仅是因为底层内存布局不同。

### sysctl vm 和 kern.malloc：观察分配状态

在动用内存调试器之前，一个有用的第一步是查看分配器的实时状态。两个 sysctl 特别有用：

```console
# sysctl vm.uma
# sysctl kern.malloc
```

第一个命令转储 UMA 的每区域统计：已分配多少项、空闲多少项、发生了多少次失败、每个区域使用了多少页。输出很长，但可以文本搜索。如果你怀疑某个特定驱动类型有泄漏，在输出中找到它的区域并观察其增长。

第二个命令转储 `malloc(9)` 的每类型统计。每个条目显示类型名称、请求数、已分配量和高水位标记。在负载下运行驱动程序并比较前后结果，是一种不需要特殊工具的简单泄漏检测技术：

```console
# sysctl kern.malloc | grep bugdemo
bugdemo:
        inuse = 0
        memuse = 0K
```

运行一个负载，再次查询，然后比较。如果 `inuse` 上升且在负载结束后不下降，说明有东西在泄漏。

相关的 `vmstat(8)` 命令有一个 `-m` 标志，以更紧凑的形式呈现相同的 `malloc(9)` 状态：

```console
# vmstat -m | head
         Type InUse MemUse HighUse Requests  Size(s)
          acl     0     0K       -        0  16,32,64,128,256,1024
         amd6     4    64K       -        4  16384
        bpf_i     0     0K       -        2
        ...
```

用于在负载期间进行持续监控：

```console
# vmstat -m | grep -E 'bugdemo|Type'
```

可以给你提供单个类型占用情况的周期性快照。

### memguard(9)：查找释放后使用

`memguard(9)` 是一个特殊的分配器，可以替换特定类型的 `malloc(9)`。其想法很简单：不从 slab 中返回一块内存，而是返回由专用页面支持的内存。当内存被释放时，页面不归还到池中；它们被取消映射，因此任何后续访问都会触发页面错误。分配周围的页面也被标记为不可访问，因此任何超出分配边界的读取或写入也会触发错误。这将释放后使用、缓冲区越界和缓冲区下溢 bug 从静默的损坏者转变为立即 panic，并带有直接指向误用点的回溯。

代价是每次分配现在至少消耗一个完整的虚拟内存页面（加上管理开销），每次释放都会消耗一个未映射的页面。因此，`memguard(9)` 通常一次只对一个 malloc 类型启用。

配置涉及两个步骤。首先，内核必须用 `options DEBUG_MEMGUARD` 构建，`std.debug` 默认不启用此选项。你需要将其添加到内核配置中：

```text
include "std.debug"
options DEBUG_MEMGUARD
```

然后重新构建。

第二步，在运行时告诉 `memguard` 保护哪个 malloc 类型：

```console
# sysctl vm.memguard.desc=bugdemo
```

从那一刻起，类型为 `bugdemo` 的每次分配都通过 `memguard` 进行。注意类型字符串要与驱动源代码中传给 `MALLOC_DEFINE(9)` 的名称匹配。此处的拼写错误会静默地什么都不做。

你也可以使用 `vm.memguard.desc=*` 来保护所有内容，但如前所述，这很昂贵。对于有针对性的 bug 狩猎，只保护你怀疑的类型。

### memguard 实战会话

假设 `bugdemo` 有一个释放后使用的 bug：驱动程序在 ioctl 完成时释放了一个缓冲区，但随后一个中断处理程序稍后又从同一缓冲区读取。没有 `memguard` 时，读取通常成功，因为 slab 分配器尚未重用该内存，或者它返回了一些碰巧替换了缓冲区的无关数据。驱动程序获得了看似合理但错误的输出，这会损坏某些后续状态，然后在很久以后才表现为一个微妙的 bug。

当为该驱动的 malloc 类型启用 `memguard` 后，相同的事件序列会在中断处理程序解引用已释放指针的瞬间触发页面错误。该错误产生一个带有经过中断处理程序的回溯的 panic。panic 消息将错误地址标识为 `memguard` 区域内部，转储上的 `kgdb` 会准确显示哪个函数解引用了已释放的内存。

将该 bug 在没有 `memguard` 时需要数天的侦探工作与此对比，你就会理解为什么这个工具如此有价值。

### redzone：缓冲区越界检测

`memguard` 是重量级的。对于更窄的缓冲区越界和下溢情况，FreeBSD 提供了 `DEBUG_REDZONE`，一个更轻量的调试器，在每个分配前后添加几个保护字节。当分配被释放时，检查保护字节，如果被修改了，`redzone` 报告损坏，包括分配时的栈信息。

`DEBUG_REDZONE` 添加到内核配置中：

```text
options DEBUG_REDZONE
```

与 `memguard` 不同，它一旦编译进去就始终活跃，并适用于所有分配。它的开销是内存而非时间：每个分配增加几个字节。

`redzone` 不捕获释放后使用，因为它保护的内存仍在原始分配之内。它确实能捕获越出预期缓冲区的写入，这是驱动程序中从用户提供的尺寸计算偏移量时常见的一类 bug。

### VM 层中的保护页

第三种机制，独立于 `memguard` 和 `redzone`，是在关键内核分配周围使用保护页。VM 系统支持分配一个在前后放置不可访问页面的内存区域。内核线程栈使用这种机制：每个栈下面的页面是未映射的，因此失控的递归会触发页面错误而不是覆盖相邻分配。

分配栈式对象的驱动程序可以使用带正确标志的 `kmem_malloc(9)`，或通过 `vm_map_find(9)` 手动设置保护页。实际上，驱动代码很少直接这样做；这种机制更常被管理自己内存区域的子系统使用。但了解这种能力是有用的，因为你可能会在内核消息中看到它并想要理解它的含义。

### 实践中的泄漏检测

泄漏是最安静的一类内存 bug。它们不产生崩溃、不产生页面错误、不产生断言失败。唯一的症状是内存使用量随时间增长。FreeBSD 提供了几种查找它们的工具。

第一种，正如我们看到的，是 `kern.malloc`。之前拍一张快照，运行负载，之后拍一张快照，寻找 `inuse` 增长且未缩小的类型。这对驱动程序泄漏来说虽粗糙但有效。

第二种是在驱动程序中添加计数器。如果每次分配递增一个 `counter(9)`，每次释放递减它，那么卸载时残留的正值告诉你驱动程序泄漏了东西。配套的 sysctl 暴露计数器以供检查：

```c
static counter_u64_t bugdemo_inflight;

/* in attach: */
bugdemo_inflight = counter_u64_alloc(M_WAITOK);

/* in allocation path: */
counter_u64_add(bugdemo_inflight, 1);

/* in free path: */
counter_u64_add(bugdemo_inflight, -1);

/* in unload: */
KASSERT(counter_u64_fetch(bugdemo_inflight) == 0,
    ("bugdemo: %ld buffers leaked at unload",
     (long)counter_u64_fetch(bugdemo_inflight)));
```

这种显式计算进行中分配的惯用法在任何拥有对象池的子系统中都很有用。卸载时的断言在任何东西泄漏时触发，在你注意到泄漏的瞬间就给你一个即时报告，而不是数小时之后。

第三种工具是 DTrace。如果你知道哪个 malloc 类型在泄漏但不知道为什么，一个 DTrace 脚本可以跟踪每次分配和每次释放，按栈跟踪累积差异：

```console
dtrace -n '
fbt::malloc:entry
/arg1 == (uint64_t)&M_BUGDEMO/
{
        @allocs[stack()] = count();
}

fbt::free:entry
/arg1 == (uint64_t)&M_BUGDEMO/
{
        @frees[stack()] = count();
}
'
```

在运行一个负载后，比较两个聚合通常能揭示一个分配但从不释放的代码路径。栈跟踪直接指向有问题的调用点。

### 当内存 Bug 隐藏时

有时内存 bug 不匹配这些模式中的任何一种。症状是在不相关子系统中的 panic，回溯看起来不可能。审查时驱动程序看起来没问题；分配和释放看起来是平衡的。然而内核不断崩溃，消息是关于损坏的链表或无效指针。

这些情况下的常见原因是驱动程序写过了缓冲区末尾，进入下一个分配。下一个分配属于别的子系统；你的越界写入静默地损坏了那个子系统的数据。崩溃发生在另一个子系统下次触及它被损坏的数据时，这可能很快，也可能很久以后。

对于这类 bug，诊断方法是启用 `DEBUG_REDZONE` 并观察警告。当 `redzone` 报告保护字节被修改时，它为分配打印的栈跟踪就是那个被讨论的分配，而越界写入的代码就是当时正在写入该分配的代码。`redzone` 的报告告诉你 bug 的两端。

另一个技巧是启用带较大 N 值的 `MALLOC_DEBUG_MAXZONES=N`。这会将分配分散到更多区域，使驱动的分配不太可能与不相关的子系统共享区域。如果症状随更多区域而消失或变化，这是 bug 涉及跨区域损坏的强烈暗示。

### 在内存 Bug 上使用 DDB

当内核因内存 bug 而 panic 时，进入 `DDB` 可以帮助缩小原因范围。有用的 `DDB` 命令包括：

- `show malloc` 转储 `malloc(9)` 状态。
- `show uma` 转储 UMA 区域状态。
- `show vmochk` 对 VM 对象树运行一致性检查。
- `show allpcpu` 显示每 CPU 状态。

这些命令产生的输出有助于将崩溃与崩溃瞬间分配器的状态关联起来。它们不能替代 `kgdb` 分析，但在你已经处于 `DDB` 中时可以更快地查阅。

### 内存调试器的现实检验

`memguard`、`redzone` 及其相关工具是有效的。但它们也具有破坏性。它们改变分配器行为、减慢内核速度，有些还大量消耗内存。在生产环境中让它们持续开启不是好主意。

正确的用法是有针对性的。当 bug 出现时，启用适当的调试器，重现 bug，捕获证据，然后关闭调试器。你大部分驱动程序开发都在带有 `INVARIANTS` 和 `WITNESS` 但不带 `DEBUG_MEMGUARD` 的内核上进行。`DEBUG_MEMGUARD` 在你积极追捕内存 bug 时拿出来，完成时收起来。

最后一点考虑。某些内存调试器，特别是 `memguard`，会以可能掩盖 bug 的方式改变分配器的可观察行为。如果驱动程序依赖两个分配在内存中相邻（它永远不应该这样做，但有时会作为意外不变式），`memguard` 会打破这种依赖并使 bug 消失。这并不意味着 bug 被修复了；它意味着 bug 现在是潜在的。始终在修复后不使用 `memguard` 重新测试，以确保修复是真实的而非调试器存在的假象。

### 内存部分总结

内存 bug 是驱动代码中的静默杀手。找到它们的耐心建立在一小套聚焦的工具之上。`memguard(9)` 直接捕获释放后使用和缓冲区越界。`redzone` 以更低的开销捕获越界。`kern.malloc` 和 UMA sysctl 暴露正常代码无法看到的分配器状态。在驱动程序自身中计算进行中分配的纪律在卸载时捕获泄漏。把这些放在一起，一类过去需要数天才能找到的 bug 可以在几分钟内自我暴露。

主要技术工具现已涵盖，我们可以转向安全使用它们的纪律，特别是在其他人正在关注的系统上。这是第 7 节的内容。

## 7. 安全调试实践

本章学到的每一种工具都有成本，而每种成本都有其可接受的上下文。开发虚拟机上的调试内核是为尽早捕获 bug 付出的小代价。生产服务器上同样的调试内核则是一场慢动作灾难。知道在什么上下文中选择什么工具，是将胜任的驱动程序作者与危险的驱动程序作者区分开来的因素之一。

本节汇集了让你远离麻烦的实践：安全使用每种工具的惯例、你即将犯错的信号，以及在风险很高时帮助你有纪律地工作的心态。

### 开发环境与生产环境的区分

安全调试中最重要的区分是开发系统和生产系统之间的区分。开发系统上你可以随意使内核崩溃；生产系统上则不行。

在开发系统上，本章的一切都是可以做的。故意触发 panic。启用 `DEBUG_MEMGUARD`。反复加载和卸载驱动程序。将 `kgdb` 附加到运行的内核上。运行收集数兆字节数据的 DTrace 脚本。最坏的情况不过是重启虚拟机，这只消几秒钟。

在生产系统上，则采取相反的态度。除非你有具体、有针对性的理由，否则不启用调试选项。不加载实验性驱动程序。不运行可能使探针框架不稳定的 DTrace 脚本。不在运行的系统上闯入 `DDB`。每次干预之前都要清楚回答"如果出了问题怎么办？"

保持这两种环境分离的纪律是避免意外破坏生产的最有效方法。准备一台开发虚拟机，将生产内核放在不同的分区上，永远不要混淆两者。

### 在生产环境中安全的操作

令人惊讶的是，调试工具包中相当多的部分在生产环境中实际上是安全的，只要谨慎使用。以下是不完整的列表。

DTrace 脚本在生产环境中通常是安全的。DTrace 框架在设计上明确包含了安全保证：探针动作不能无限循环、不能分配任意内存、不能在没有进入明确定义恢复路径的情况下解引用任意指针。你可以在繁忙的服务器上运行 DTrace 聚合而不会使其宕机。注意事项是极高频探针会消耗大量 CPU（网络驱动程序中每个包触发一次的探针不太可能是免费的），而且如果不限速，DTrace 输出可能填满文件系统空间。

`ktrace(1)` 对特定进程是安全的，不过它会写入一个无限增长的文件。设置大小限制或监视文件大小。

`ktr(9)` 在相关类已编译进去时是安全的。通过 `sysctl debug.ktr.mask=` 启用类是安全的。编译新类需要重建内核，这是开发活动。

读取 sysctl 始终是安全的。`kern.malloc`、`vm.uma`、`hw.ncpu`、`debug.kdb.*` 以及所有其他 sysctl 暴露状态而不改变任何东西。一个有问题驱动的生产系统可以仅通过 sysctl 就进行大量探查。

### 在生产环境中不安全的操作

更短的列表，但很重要。

Panic 是不安全的。故意崩溃一台生产服务器只有在服务器已经不可恢复且转储是理解原因的最佳途径时才可接受。`sysctl debug.kdb.panic=1` 触发立即 panic 和转储。不要轻易这样做。

在生产控制台上闯入 `DDB` 是不安全的。当你在 `DDB` 中时整个内核停止。用户进程冻结。网络连接超时。实时工作停止。除非替代方案更糟（通常在灾难性崩溃期间是这种情况），否则不要在生产环境中进入 `DDB`。

对所有分配类型启用 `DEBUG_MEMGUARD` 是不安全的。内存使用量膨胀。性能急剧下降。内存密集型工作负载可能完全失败。如果你绝对必须在使用中环境使用 `memguard`，将范围限制为一次一个 malloc 类型并监控内存使用。

可加载内核模块是有风险的。加载或卸载模块会触及内核状态。有 bug 的模块可能在加载时、卸载时或其间任何时间使内核崩溃。在生产环境上，只加载在开发环境中针对相同内核测试过的模块。

过于激进的 DTrace 脚本可能使系统不稳定。记录栈跟踪的聚合会产生内存压力。有副作用的探针可能与工作负载以意想不到的方式交互。运行 DTrace 脚本时设置明确的时间限制，并在让它们持续运行之前仔细审查聚合。

### 在生产系统上捕获证据

当生产环境中出问题且 bug 罕见或在开发环境中难以重现时，挑战在于在不破坏运行服务的情况下捕获足够的证据来诊断问题。几种策略有所帮助。

首先，从被动观察开始。`sysctl`、`vmstat -m`、`netstat -m`、`dmesg` 以及各种 `-s` 系统统计命令可以在系统运行时执行且几乎不花任何成本。如果 bug 在这些报告中产生可见症状，定期捕获报告。

其次，使用有严格边界的 DTrace。运行六十秒然后退出的脚本产生一个快照而不留下持续风险。聚合特别适合这种风格：它们在一个时间窗口内收集统计、打印结果然后停止。

第三，如果需要崩溃转储但系统尚未崩溃，最安全的方法是等待崩溃发生。现代转储机制设计为在 panic 瞬间捕获内核状态；手动触发的转储只有在你知道系统已经不可恢复时才有用。

第四，当崩溃确实发生时，在转储上工作，而非运行中的系统。重启到新内核恢复服务，而转储仍然可供闲暇时离线分析。"快速重启，稍后分析"的纪律在生产硬件上通常是正确的权衡。

### 使用 log(9) 代替 printf 进行诊断

贯穿本章我们一直将 `printf` 用作内核端日志记录的简写，这也是教科书中的常见呈现方式。在生产系统上你应该优先使用 `log(9)`，它通过 `syslogd(8)` 设施写入而非直接写入控制台。原因是实际的：控制台输出是无缓冲且缓慢的，`log(9)` 是限速且缓冲的，`log(9)` 最终进入 `/var/log/messages`，日志分析工具可以使用。

API 位于 `/usr/src/sys/sys/syslog.h` 和 `/usr/src/sys/kern/subr_prf.c`。用法：

```c
#include <sys/syslog.h>

log(LOG_WARNING, "bugdemo: unexpected state %d\n", sc->state);
```

优先级（`LOG_DEBUG`、`LOG_INFO`、`LOG_NOTICE`、`LOG_WARNING`、`LOG_ERR`、`LOG_CRIT`、`LOG_ALERT`、`LOG_EMERG`）让 `syslogd` 以不同方式路由消息。

一种常见的扩展是限速日志，这样一个有问题的驱动程序不会以每秒数百万条的速度淹没 `/var/log/messages`。FreeBSD 提供了 `ratecheck(9)` 原语，你可以将其包装在自己的 `log` 调用周围：

```c
#include <sys/time.h>

static struct timeval lastlog;
static struct timeval interval = { 5, 0 };   /* 5 seconds */

if (ratecheck(&lastlog, &interval))
        log(LOG_WARNING, "bugdemo: error (rate-limited)\n");
```

`ratecheck(9)` 每个时间间隔返回一次非零值，在间隔之间抑制重复日志。这种技术对于可能反复观察同一错误的任何驱动程序来说都是必不可少的。

### 不要在集群中混合调试和发布版内核

一个微妙的陷阱是在生产集群中运行混合的调试和发布版内核。直觉是调试内核在 bug 出现时能提供更好的诊断。现实是调试内核的性能明显低于发布版内核，有不同的内存使用，并可能表现出不同的时序。如果 bug 对这些因素敏感（许多并发 bug 正是如此），运行混合内核保证你的重现环境不匹配生产环境。

正确的做法是统一的：要么整个集群运行发布版内核（你在开发硬件上调试），要么整个集群运行调试内核（你接受开销）。混合部署仅在非常受控的实验中才是第三种选择。

### 制定恢复计划

在执行任何有风险的调试操作之前，了解你的恢复计划。如果系统挂起，你将如何恢复？是否有可以发出硬件重置的 IPMI 接口？是否有另一位管理员可以在需要时循环电源？多少数据丢失是可接受的？

一个好的恢复计划是两步。首先，快速让系统恢复运行。其次，捕获证据（转储、日志）供离线分析。这两步通常涉及不同的人或不同的时间尺度，提前想清楚这两步可以防止事到临头时的慌乱。

### 保持调试日志

在追踪一个棘手的 bug 时，书面记录是无价的。每条记录应包含：

- 你在测试什么假设。
- 你采取了什么行动。
- 你观察到了什么结果。
- 结果排除了什么或确认了什么。

这听起来学究气，但确实有用。漫长的调试会话涉及数十个微假设，失去哪些已经测试过的跟踪会浪费大量时间。书面记录在你周末回来继续追查 bug 时、或将它交给同事时也有帮助。

对于跨多个系统的驱动程序 bug，共享记录（bug 追踪器、wiki 或内部工单）更有价值。每个接触该 bug 的人都能看到其他人已经尝试了什么，没有人重复运行同一个实验。

### 在你自己的驱动程序上练习

一个长期有效的习惯是保留一个故意有 bug 的驱动程序版本用于练习。每次你在实际工作中发现一个有趣的 bug，将它的变体添加到练习驱动程序中。然后定期用新的眼光运行练习驱动程序，确保你仍然能用本章的工具找到那些 bug。这会建立肌肉记忆，在真正的时间压力下 bug 出现时是无价的。

我们贯穿本章一直在使用的 `bugdemo` 驱动程序就是这样一个练习驱动程序的起点。派生它，添加你自己的 bug，用它来保持敏锐。

### 知道何时停下来

安全调试智慧的最后一部分是知道何时停下来。并非每个 bug 都需要追查到最后一条指令。如果一个 bug 很罕见、有变通方法、且找到根因的成本以天计，有时有理由记录变通方法然后继续前进。这是一个判断，而非规则，但做出这种判断的能力是专业成熟度的一部分。

相反的错误（过早宣布胜利，接受不解决底层缺陷的表面修复）也很常见。症状是 bug 不断以新形式回来。当一个"修复"不能产生稳定的结果时，说明更深层的东西出了问题，需要更多调查。

在这两个极端之间是健康的区域，你投入与 bug 重要性成正比的时间。内核开发回报耐心，但也回报务实。本章的工具旨在使投入高效，而非让每个 bug 都成为耗时数天的研究项目。

有了安全实践的建立，我们可以转向本章最后一个主要话题：在一次发现严重问题的调试会话之后该做什么，以及如何使驱动程序对下一次类似问题更具韧性。

## 8. 调试会话后的重构：恢复与韧性

来之不易的调试胜利不是工作的结束。找到 bug 是找到证据。真正的问题是：证据告诉了我们关于驱动程序的什么，驱动程序应该如何改变以回应？

一种常见的失败模式是修补即时症状然后继续。补丁让测试通过、让崩溃停止、让损坏消失。但首先允许 bug 进入的底层弱点仍然在那里，潜伏着。周围代码的下一次微妙变化，或下一个新环境，会发现同样的弱点并产生下一个 bug。

本节就是关于抵制这种失败模式的。我们将走一小套技术，用调试结果来加强驱动程序，而不仅仅是修复特定的 bug。

### 将Bug视为消息

每个 bug 都携带着关于设计的信息。释放后使用说的是"驱动程序对这个缓冲区的所有权模型不清晰"。死锁说的是"驱动程序的锁序没有显式记录或执行"。内存泄漏说的是"驱动程序对这个对象的生命周期管理不完整"。`attach` 中的 panic 说的是"驱动程序在初始化期间的错误恢复很弱"。竞态条件说的是"驱动程序对线程上下文的假设不够严格"。

当你找到一个 bug 时，花几分钟问问这个 bug 在告诉你关于设计的什么。具体的缺陷通常是更广泛模式的症状，理解模式使未来的 bug 更容易预防。

### 加强不变量

对 bug 的一种具体回应是添加能更早捕获它的断言。如果 bug 是释放后使用，在确认缓冲区使用时仍然有效的路径上添加 `KASSERT`。如果违反了锁序，在违反发生的点添加 `mtx_assert(9)`。如果结构字段被损坏，在其对齐上添加 `CTASSERT` 或对其值添加运行时检查。

目标不是用断言复制每个检查，而是将每个 bug 转化为一两个新的不变式，使同一类 bug 在未来不可能发生。随着时间推移，驱动程序积累了一套反映其实际行为的防御性检查，以代码而非以你脑海中的方式记录。

### 记录所有权模型

另一种常见回应是澄清文档。许多 bug 的产生是因为资源所有权（谁分配了它、谁负责释放它、何时可以安全访问）是隐式的。写几行注释显式声明所有权规则使规则对下一个读者可见，并且经常迫使你面对规则实际上并不一致的情况。

例如，这样的注释：

```c
/*
 * bugdemo_buffer is owned by the softc from attach until detach.
 * It may be accessed from any thread that holds sc->sc_lock.
 * It must not be accessed in interrupt context because the lock
 * is a regular mutex, not a spin mutex.
 */
struct bugdemo_buffer *sc_buffer;
```

这条注释不是装饰性的。它是驱动程序将执行的不变式声明。如果未来的 bug 违反了这些不变式，注释就是理解出了什么问题的参考点。

### 收窄 API 表面

第三种回应是收窄 API。如果 bug 是因为函数在不应该被调用的上下文中被调用了，能否将函数设为私有，使它只在安全的上下文中被调用？如果某个状态通过不应该存在的路径到达了，能否使该状态不可达？

原则是驱动程序的每个外部入口点都是 bug 的攻击面。通过将函数设为内部函数、将状态隐藏在访问器后面、将相关操作合并为原子事务来减少表面，使驱动程序更难被误用。

这不是关于教条式的极简主义。这是关于认识到表面积与 bug 风险成正比，许多 bug 可以通过不让暴露被误用的东西来预防。

### 加固卸载路径

卸载是驱动程序中经常未被充分加固的路径。`attach` 路径通常测试良好；卸载路径通常则不然。这是 bug 的主要来源：一个在长期运行中完美工作的驱动程序可能在 `kldunload` 时崩溃。

好的卸载路径满足几个不变式。在 `attach` 中分配的每个对象都被释放。驱动程序产生的每个线程都已退出。每个定时器都已取消。每个 callout 都已排干。每个 taskqueue 都已完成其待处理工作。每个设备节点在支持它的内存被释放之前都被销毁。

在卸载路径出现 bug 后，对照此清单审计整个卸载函数。每一项都是驱动程序应维护的不变式，违反是常见的。

### 韧性驱动的形态

把这些习惯放在一起，一个有韧性的驱动程序是什么样的？几个特征很突出。

它的加锁是显式的。每个共享数据结构都由一个命名的锁保护，访问该结构的每个函数要么有一个断言说锁已被持有，要么有一个文档说明为什么不需要锁。锁序在每次多锁获取处的注释中记录。`WITNESS` 在正常操作中不产生警告。

它的错误处理是完整的。每次分配都有对应的释放。每个 `attach` 都有完整的 `detach`。代码中的每条路径在失败时都能自行清理。部分状态不会残留。驱动程序不会卡在半初始化或半拆除的状态。

它的不变量以代码表达。前置条件在函数入口用 `KASSERT` 检查。结构不变量在编译时用 `CTASSERT` 检查。状态转换用显式检查验证。

它的可观测性是内置的。计数器暴露分配和错误率。SDT 探针在关键事件触发。sysctl 暴露足够的状态，使操作员无需调试器就能检查驱动程序。驱动程序告诉你它在做什么。

它的错误消息是有用的。`log(9)` 消息包含子系统名称、特定错误和足够的上下文来定位问题。它们是限速的。它们不会记录训练操作员忽略它们的虚假警告。

这些特征不是免费的。它们需要时间来实现和纪律来维护。但一旦驱动程序拥有它们，未来 bug 的成本就会急剧下降，因为 bug 被更早捕获、更容易诊断、更确定地修复。

### 重访 bugdemo 驱动程序

到本章实验结束时，我们将对 `bugdemo` 驱动程序应用许多这样的想法。最初只有少数几个故意破坏的代码路径，通过迭代，逐渐成长为一个在每个关键点有断言、每个操作有计数器、每个有趣事件有 SDT 探针、卸载路径经得起审查的驱动程序。这个轨迹是刻意设计的，用于映射真实驱动程序的成熟轨迹。

### 关闭重构循环

关于重构的最后一个想法。每次你因 bug 而修改驱动程序，你都在承担修改引入新 bug 的小风险。这种风险不可避免但可控。几种做法有帮助。

首先，隔离变更。做出解决根因的最小修改，并将其与外观性变更分开提交。如果出现回归，归因很容易。

其次，添加测试。如果 bug 是由特定的 ioctl 序列触发的，添加一个运行该序列并验证正确结果的小测试程序。将测试保留在你的仓库中。随每个 bug 增长的测试套件会随着时间推移成为一项资产。

第三，运行现有测试。如果驱动程序有任何自动化测试，在修复后运行它们。令人惊讶的是，即使测试套件很小，这种方式也能捕获很多回归。

第四，记录教训。在调试日志或提交消息中，简要记录 bug 揭示的驱动程序设计问题。这条记录是给未来自己的礼物，你以后会遇到类似的模式。

有了这些习惯，调试变成了一个发现的循环而非一连串的救火。每个 bug 都教授一些东西，每个教训都加强驱动程序，每个加强的驱动程序都变得更容易使用。本章的工具就是使这个循环转动的手段。

概念性内容已经讲完，我们可以进入动手实验部分，在那里我们将本章的每种技术应用到 `bugdemo` 驱动程序上，看每一种产生具体的结果。

## 动手实验

本节中的每个实验都是独立的，但建立在前一个之上。它们使用 `bugdemo` 驱动程序，其配套源代码位于 `examples/part-07/ch34-advanced-debugging/`。

开始之前，确保你有一个可以安全崩溃内核的开发用 FreeBSD 14.3 虚拟机，一份位于 `/usr/src/` 的 FreeBSD 源码树副本，以及能够附加保留跨重启输出的串行或虚拟控制台。

### 实验 1：为 bugdemo 添加断言

在本实验中，我们构建 `bugdemo` 的第一个版本并添加捕获内部不一致性的断言。目标是看到 `KASSERT`、`MPASS` 和 `CTASSERT` 在实践中工作。

**步骤 1：构建并加载基准驱动程序。**

基准驱动程序位于 `examples/part-07/ch34-advanced-debugging/lab01-kassert/`。它是一个最小的伪设备，有一个 ioctl 在被指示时触发 bug。从实验目录执行：

```console
$ make
$ sudo kldload ./bugdemo.ko
$ ls -l /dev/bugdemo
```

如果设备节点出现，驱动程序加载成功。

**步骤 2：运行测试工具确认驱动正常工作。**

实验还包含一个小型用户空间程序 `bugdemo_test`，它打开设备并发出 ioctl：

```console
$ ./bugdemo_test hello
$ ./bugdemo_test noop
```

两者都应返回成功。未触发任何 bug 时，驱动程序行为正确。

**步骤 3：检查源代码中的断言。**

打开 `bugdemo.c`，找到 `bugdemo_process` 函数。你会看到类似这样的代码：

```c
static void
bugdemo_process(struct bugdemo_softc *sc, struct bugdemo_command *cmd)
{
        KASSERT(sc != NULL, ("bugdemo: softc missing"));
        KASSERT(cmd != NULL, ("bugdemo: cmd missing"));
        KASSERT(cmd->op < BUGDEMO_OP_MAX,
            ("bugdemo: op %u out of range", cmd->op));
        MPASS(sc->state == BUGDEMO_STATE_READY);
        /* ... */
}
```

每个断言记录一个不变式。如果其中任何一个触发，内核以标识被违反不变式的消息 panic。

**步骤 4：触发一个断言。**

驱动程序有一个名为 `BUGDEMO_FORCE_BAD_OP` 的 ioctl，它在调用 `bugdemo_process` 之前故意将 `cmd->op` 设置为超出范围的值：

```console
$ ./bugdemo_test force-bad-op
```

在调试内核上，这产生立即的 panic：

```text
panic: bugdemo: op 255 out of range
```

系统重启。在发布版内核（无 `INVARIANTS`）上，`KASSERT` 被编译出去，驱动程序带着超出范围的值继续执行。这种差异正是开发期间使用调试内核的价值。

**步骤 5：确认断言在正确的行触发。**

重启后，如果捕获了转储，用 `kgdb` 打开它：

```console
# kgdb /boot/kernel/kernel /var/crash/vmcore.last
(kgdb) bt
```

回溯将显示 `bugdemo_process`，`frame N` 到那个条目将显示断言行。这就是端到端链条：断言触发、内核 panic、转储捕获状态、kgdb 标识代码。

**步骤 6：添加你自己的断言。**

修改驱动程序添加一个断言，在特定代码路径中计数器不为零。重建、重新加载、触发使计数器为零的情况。观察你的断言如期触发。

**本实验教授的内容。** `KASSERT` 宏是一个活的检查，而非理论的。它触发、它 panic、它标识代码。添加断言的纪律由测试它们应该触发时就触发的纪律支撑。

### 实验 2：使用 kgdb 捕获和分析 Panic

在本实验中我们专注于事后分析工作流。从一个干净的调试内核开始，我们触发 panic、捕获转储并用 `kgdb` 遍历它。

**步骤 1：确认转储设备已配置。**

在虚拟机上运行：

```console
# dumpon -l
```

如果输出显示设备路径（通常是交换分区），你就准备好了。如果没有，配置一个：

```console
# dumpon /dev/ada0p3        # replace with your swap partition
# echo 'dumpdev="/dev/ada0p3"' >> /etc/rc.conf
```

**步骤 2：确认调试内核正在运行。**

```console
# uname -v
# sysctl debug.debugger_on_panic
```

`debug.debugger_on_panic` 应为 `0` 或 `1`，取决于你是否想在转储之前暂停在调试器。对于自动化实验工作，`0` 更方便；对于交互式探索，`1` 更有教学意义。

```console
# sysctl debug.debugger_on_panic=0
```

**步骤 3：加载 bugdemo 并触发 panic。**

```console
# kldload ./bugdemo.ko
# ./bugdemo_test null-softc
panic: bugdemo: softc missing
Dumping ...
Rebooting ...
```

panic 消息、转储通知和重启都出现在控制台上。在带虚拟磁盘的虚拟机上转储写入需要几秒钟。

**步骤 4：重启后，检查保存的转储。**

```console
# ls /var/crash/
bounds  info.0  info.last  minfree  vmcore.0  vmcore.last
# cat /var/crash/info.0
```

`info.0` 文件汇总了 panic：内核版本、消息和转储前捕获的初始回溯。

**步骤 5：在 kgdb 中打开转储。**

```console
# kgdb /boot/kernel/kernel /var/crash/vmcore.0
```

`kgdb` 自动运行回溯。识别在 `bugdemo_ioctl` 或 `bugdemo_process` 内部的帧。切换到它：

```console
(kgdb) frame 5
(kgdb) list
(kgdb) info locals
(kgdb) print sc
```

观察 `sc` 为 NULL，确认 panic 消息。

**步骤 6：探索相邻状态。**

从 `kgdb` 中，检查触发 panic 的进程：

```console
(kgdb) info threads
(kgdb) thread N       # where N is the panicking thread
(kgdb) proc          # driver-specific helper for process state
```

`proc` 是一个特定于内核的命令，打印当前进程。通过这些命令和 `bt`，你可以构建 panic 上下文的完整图景。

**步骤 7：退出 kgdb。**

```console
(kgdb) quit
```

转储保留在磁盘上；你可以随时重新打开它。

**本实验教授的内容。** panic、转储和离线分析的完整循环是常规操作，并不神秘。开发虚拟机应该能在不到一分钟内完成这个循环。纪律是在遇到第一个真正 bug 之前就练习它，这样当你需要它时你不需要匆忙学习工具。

### 实验 3：构建 GENERIC-DEBUG 并确认选项激活

本实验是关于内核配置而非代码。目标是走完构建、安装和验证调试内核的完整流程。

**步骤 1：从一个干净的 `/usr/src/` 开始。**

如果你有源码树，更新它。如果没有，安装一个：

```console
# git clone --depth 1 -b releng/14.3 https://git.freebsd.org/src.git /usr/src
```

**步骤 2：查看现有的 GENERIC-DEBUG 配置。**

```console
$ ls /usr/src/sys/amd64/conf/GENERIC*
$ cat /usr/src/sys/amd64/conf/GENERIC-DEBUG
```

注意它只有两行：`include GENERIC` 和 `include "std.debug"`。接下来查看 `std.debug`：

```console
$ cat /usr/src/sys/conf/std.debug
```

确认我们讨论过的选项：`INVARIANTS`、`INVARIANT_SUPPORT`、`WITNESS` 及其他。

**步骤 3：构建内核。**

```console
# cd /usr/src
# make buildkernel KERNCONF=GENERIC-DEBUG
```

在适中的虚拟机上这需要二十到四十分钟。构建产生详细输出；如果因错误停止，调查并重试。

**步骤 4：安装内核。**

```console
# make installkernel KERNCONF=GENERIC-DEBUG
# ls -l /boot/kernel/kernel /boot/kernel.old/kernel
```

之前的内核保留在 `/boot/kernel.old/` 中作为恢复选项。

**步骤 5：重启到新内核。**

```console
# shutdown -r now
```

重启后，确认：

```console
$ uname -v
$ sysctl debug.kdb.current_backend
$ sysctl debug.kdb.supported_backends
```

后端应列出 `ddb` 和 `gdb` 两者。

**步骤 6：确认 INVARIANTS 已激活。**

构建并加载实验 1 的 `bugdemo.ko`，然后像实验 1 那样触发超出范围的 op。在调试内核上，panic 触发。在发布版内核上，不会触发。这个往返确认 `INVARIANTS` 确实被编译进去了。

**步骤 7：确认 WITNESS 已激活。**

实验 3 变体的 `bugdemo` 有一个故意的锁序反转，由特定的 ioctl 触发。加载它，运行触发测试，在控制台上观察 `WITNESS` 警告：

```text
lock order reversal:
 ...
```

不产生 panic，只有警告。这是预期行为：`WITNESS` 检测潜在的死锁并报告它们，而不强制系统失败。

**步骤 8：如果新内核无法启动则恢复。**

如果你的新内核因任何原因无法启动，FreeBSD 启动加载器提供恢复选项。从加载器菜单中，选择"Boot Kernel"然后选择"kernel.old"。你之前的内核启动，你可以从容地调查调试内核的故障。

**本实验教授的内容。** 构建调试内核不是神秘的操作。它是用不同选项重新构建加重启。风险是可预测的：较长的构建时间、较大的二进制文件，以及需要保留之前的内核作为后备。

### 实验 4：使用 DTrace 和 ktrace 追踪 bugdemo

本实验练习我们学过的三种追踪工具：DTrace `fbt` 探针、DTrace SDT 探针和 `ktrace(1)`。

**步骤 1：加载带 SDT 探针的 bugdemo 变体。**

`lab04-tracing` 变体的 bugdemo 在关键点定义了 SDT 探针：

```c
SDT_PROVIDER_DEFINE(bugdemo);
SDT_PROBE_DEFINE2(bugdemo, , , cmd__start, "struct bugdemo_softc *", "int");
SDT_PROBE_DEFINE3(bugdemo, , , cmd__done, "struct bugdemo_softc *", "int", "int");
```

加载它：

```console
# kldload ./bugdemo.ko
```

**步骤 2：列出探针。**

```console
# dtrace -l -P sdt -n 'bugdemo:::*'
```

你应该看到 `cmd-start` 和 `cmd-done` 探针列出。

**步骤 3：观察探针触发。**

在一个终端中：

```console
# dtrace -n 'sdt:bugdemo::cmd-start { printf("op=%d\n", arg1); }'
```

在另一个终端中：

```console
$ ./bugdemo_test noop
$ ./bugdemo_test hello
```

第一个终端显示每个探针及其 op 值触发。

**步骤 4：测量每次操作的延迟。**

```console
# dtrace -n '
sdt:bugdemo::cmd-start
{
        self->start = timestamp;
}

sdt:bugdemo::cmd-done
/self->start != 0/
{
        @by_op[arg1] = quantize(timestamp - self->start);
        self->start = 0;
}
'
```

运行大量 ioctl 的负载，然后 Ctrl-C 中止 DTrace。打印聚合结果，显示每种操作的延迟直方图。

**步骤 5：使用 fbt 追踪入口。**

```console
# dtrace -n 'fbt::bugdemo_*:entry { printf("%s\n", probefunc); }'
```

从用户空间触发一些 ioctl。DTrace 终端显示每次入口，给你驱动程序流程的实时视图。

**步骤 6：使用 ktrace 追踪用户空间侧。**

```console
$ ktrace -t ci ./bugdemo_test hello
$ kdump
```

观察 kdump 输出中可见的 ioctl 调用。

**步骤 7：结合 ktrace 和 DTrace。**

在一个终端运行 DTrace，监视 SDT 探针，同时在另一个终端对用户空间测试运行 ktrace。两个输出一起阅读，给出从用户空间到内核再返回的交互完整图景。

**本实验教授的内容。** 追踪不是单一工具；它是一个家族。DTrace 是最丰富的，`ktrace(1)` 是查看用户-内核边界最简单的方式，结合它们能给出最完整的视图。

### 实验 5：使用 memguard 捕获释放后使用

本实验走完一个真实的内存调试场景。`lab05-memguard` 变体的 `bugdemo` 包含一个故意的释放后使用 bug：在特定 ioctl 序列下，驱动程序释放了一个缓冲区，然后从 callout 中读取它。

**步骤 1：构建带 DEBUG_MEMGUARD 的内核。**

将 `options DEBUG_MEMGUARD` 添加到你的 `GENERIC-DEBUG` 配置中，或创建新配置：

```text
include GENERIC
include "std.debug"
options DEBUG_MEMGUARD
```

像实验 3 那样重新构建和安装。

**步骤 2：加载实验 5 的 bugdemo 并启用 memguard。**

```console
# kldload ./bugdemo.ko
# sysctl vm.memguard.desc=bugdemo
```

第二个命令告诉 `memguard(9)` 保护所有 malloc 类型为 `bugdemo` 的分配。确切的类型名称来自驱动程序的 `MALLOC_DEFINE` 调用。

**步骤 3：触发释放后使用。**

```console
$ ./bugdemo_test use-after-free
```

用户空间调用快速返回。稍后（当 callout 触发时），内核在 callout 例程内以页面错误 panic：

```text
Fatal trap 12: page fault while in kernel mode
fault virtual address = 0xfffff80002abcdef
...
KDB: stack backtrace:
db_trace_self_wrapper()
...
bugdemo_callout()
...
```

`memguard(9)` 将一个静默的释放后使用转变为立即的页面错误。回溯直接指向 `bugdemo_callout`。

**步骤 4：用 kgdb 分析转储。**

```console
# kgdb /boot/kernel/kernel /var/crash/vmcore.last
(kgdb) bt
(kgdb) frame N      # into bugdemo_callout
(kgdb) list
(kgdb) print buffer
```

源代码行显示从 `buffer` 的读取，而 `buffer` 是一个已释放的 `memguard` 保护的地址。`kgdb` 将其打印为一个不再映射的地址。

**步骤 5：修复 bug 并验证。**

修复是在释放缓冲区之前取消 callout。相应地修改驱动程序源代码，重建、重新加载并运行相同的测试。panic 不再触发。在验证期间保持 `memguard` 启用，然后禁用它并重新测试：

```console
# sysctl vm.memguard.desc=
```

两次运行都应成功。如果发布版模式运行（不带 `memguard`）仍然失败，说明 bug 未完全修复。

**步骤 6：计算进行中的分配。**

实验还展示了一种替代技术：计算进行中的分配。在驱动程序中添加一个 `counter(9)`，分配时递增，释放时递减。卸载时断言计数器为零：

```c
KASSERT(counter_u64_fetch(bugdemo_inflight) == 0,
    ("bugdemo: leaked %ld buffers",
     (long)counter_u64_fetch(bugdemo_inflight)));
```

在未先释放所有缓冲区的情况下卸载，观察断言触发。

**本实验教授的内容。** `memguard(9)` 是针对特定类别 bug 的特定工具。当它适用时，它将困难 bug 变成简单 bug。知道何时使用它是实用技能。

### 实验 6：使用 GDB 桩进行远程调试

本实验演示通过虚拟串行端口进行远程调试。假设你使用的是 bhyve 或 QEMU，串行控制台暴露给主机。

**步骤 1：在内核中配置 KDB 和 GDB。**

两者应该已经在 `GENERIC-DEBUG` 中。用以下命令确认：

```console
# sysctl debug.kdb.supported_backends
```

**步骤 2：在虚拟机中配置串行控制台。**

在 bhyve 中，向启动命令添加 `-l com1,stdio` 或等价参数。在 QEMU 中，使用 `-serial stdio` 或 `-serial pty`。目标是拥有一个可从主机访问的虚拟串行端口。

**步骤 3：在虚拟机中，切换到 GDB 后端。**

```console
# sysctl debug.kdb.current_backend=gdb
```

**步骤 4：在虚拟机中，进入调试器。**

在串行控制台上发送进入调试器的序列，或触发 panic：

```console
# sysctl debug.kdb.enter=1
```

内核停止。串行控制台显示：

```text
KDB: enter: sysctl debug.kdb.enter
[ thread pid 500 tid 100012 ]
Stopped at     kdb_enter+0x37: movq  $0,kdb_why
gdb>
```

**步骤 5：在主机上，附加 kgdb。**

```console
$ kgdb /boot/kernel/kernel
(kgdb) target remote /dev/ttyXX    # the host-side serial device
```

主机的 `kgdb` 通过串行线连接到内核。你现在可以在运行的内核上运行完整的 `kgdb` 命令：`bt`、`info threads`、`print`、`set variable` 等等。

**步骤 6：设置断点。**

```console
(kgdb) break bugdemo_ioctl
(kgdb) continue
```

虚拟机恢复运行。在虚拟机中运行 `./bugdemo_test hello`。断点触发，主机上的 `kgdb` 显示状态。

**步骤 7：干净地分离。**

```console
(kgdb) detach
(kgdb) quit
```

在虚拟机中，内核恢复运行。

**本实验教授的内容。** 远程调试是一个专业但有价值的工具。当你需要对运行中的内核进行实时检查时最有用，特别是对于难以作为转储捕获的间歇性 bug。

## 挑战练习

以下挑战建立在实验之上。它们被设计为开放式的：有多种有效的方法，重点是为每个 bug 练习选择正确的工具。

### 挑战 1：找到静默 Bug

`lab-challenges/silent-bug` 变体的 `bugdemo` 包含一个不产生崩溃也不产生错误的 bug。相反，一个计数器在特定 ioctl 序列后有时报告错误的值。你的任务：

1. 编写一个重现该 bug 的测试程序。
2. 使用 DTrace 缩小哪个函数产生了错误的计数器值。
3. 修复 bug 并验证 DTrace 特征消失。

提示：bug 是缺少内存屏障，而非缺少锁。症状是缓存一致性，而非竞争。

### 挑战 2：猎捕泄漏

`lab-challenges/leaky-driver` 变体在每次行使特定 ioctl 路径时泄漏一个对象。你的任务：

1. 使用 `vmstat -m` 在负载前后确认泄漏。
2. 使用 DTrace 记录泄漏对象类型的每次分配和释放，按栈聚合。
3. 标识分配但未释放的代码路径。
4. 向驱动程序添加基于 `counter(9)` 的进行中检查，并验证它在错误路径被采用时触发。

### 挑战 3：诊断死锁

`lab-challenges/deadlock` 变体有时在两个 ioctl 并发运行时挂起。你的任务：

1. 重现挂起。
2. 用 `kgdb` 附加到挂起的内核（或进入 `DDB`）。
3. 对每个卡住的线程使用 `info threads` 和 `bt` 标识锁序。
4. 确定修复方案（重排锁序，或消除其中一个）。

### 挑战 4：阅读真实的 Panic

加载一个你没写过的内核模块（例如某个 USB 类驱动或文件系统模块）。通过从用户空间发送畸形输入故意触发一次不良交互。当它 panic（或未能 panic）时，写下：

1. 导致症状的确切序列。
2. 观察到的回溯或错误。
3. 该模块是否有能更早捕获问题的断言。
4. 一个关于加强该模块不变量的建议。

### 挑战 5：构建你自己的 bugdemo 变体

创建一个 `bugdemo` 的新变体，包含你在真实代码中遇到过的 bug。编写一个确定性地触发该 bug 的测试程序。然后，使用本章技术的任何子集，从头诊断该 bug。写下你学到的内容。重点是练习将"我认出这个模式"转化为可重现的教学材料。

## 常见问题故障排除

即使最好的工具在实践中也会遇到问题。本节收集你最可能遇到的问题及其解决方法。

### 转储未被捕获

panic 后，`/var/crash/` 只显示 `bounds` 和 `minfree`，没有 `vmcore.N`。可能的原因：

- **未配置转储设备。** 正常启动后运行 `dumpon -l`。如果报告"no dump device configured"，用 `dumpon /dev/DEVICE` 设置一个并在 `/etc/rc.conf` 中用 `dumpdev=` 持久化。
- **转储设备太小。** 转储需要等于内核内存加一些开销的空间。1GB 的交换分区无法容纳 8GB 内存机器的转储。扩大转储设备或使用压缩转储（`dumpon -z`）。
- **savecore 被禁用。** 检查 `/etc/rc.conf` 中的 `savecore_enable="NO"`。改为 `YES` 并重启。
- **崩溃太严重。** 如果 panic 本身阻止转储机制运行，你可能完全看不到输出。这种情况下串行控制台对于至少捕获 panic 消息是必不可少的。

### kgdb 说"没有符号"

打开转储时，`kgdb` 打印"no debugging symbols found"或类似信息。可能的原因：

- **内核构建时没有 `-g`。** 调试内核通过 `makeoptions DEBUG=-g` 自动包含 `-g`。发布版内核不包含。要么构建调试内核，要么安装调试符号包（如果可用）。
- **内核和转储不匹配。** 如果转储来自与 `kgdb` 加载的不同的内核，符号将不匹配。使用 panic 时正在运行的精确内核二进制文件。
- **模块符号缺失。** 如果 panic 发生在没有用 `-g` 构建的模块内，`kgdb` 为该模块显示没有源代码行的地址。用 `DEBUG_FLAGS=-g` 重建模块。

### DDB 冻结系统

进入 `DDB` 有意地停止内核。这是设计如此，但在类生产系统上可能看起来像挂起。如果你在 `DDB` 中并想恢复：

- `continue` 退出 `DDB` 并返回内核。
- `reset` 立即重启。
- `call doadump` 强制转储然后重启。

如果你不小心进入了 `DDB`，`continue` 几乎总是正确的操作。

### 模块拒绝卸载

`kldunload bugdemo` 返回 `Device busy`。原因：

- **打开的文件描述符。** 某些东西仍然持有 `/dev/bugdemo` 打开。使用 `fstat | grep bugdemo` 找到进程并关闭它们。
- **引用计数。** 另一个模块引用了这个模块。先卸载那个模块。
- **待处理工作。** 一个 callout 或 taskqueue 仍在计划中。等待它排干，或让驱动程序在卸载路径中显式取消和排干。
- **卡住的线程。** 驱动程序产生的内核线程尚未退出。在卸载时从驱动程序内部终止它。

### memguard 不起作用

设置了 `vm.memguard.desc=bugdemo`，但 memguard 似乎没有捕获任何 bug。原因：

- **类型名称错误。** `vm.memguard.desc` 必须与传给 `MALLOC_DEFINE(9)` 的类型完全匹配。如果你设置了 `vm.memguard.desc=BugDemo` 但驱动使用 `MALLOC_DEFINE(..., "bugdemo", ...)`，名称不匹配。
- **内核未用 `DEBUG_MEMGUARD` 构建。** sysctl 节点只在选项编译进来时存在。检查 `sysctl vm.memguard.waste` 或类似的；如果返回"unknown oid"，该功能未编译进来。
- **未遍历分配路径。** 如果 bug 所在的代码路径实际上不使用被保护的类型，`memguard` 无法捕获它。用 `vmstat -m` 确认分配类型。

### DTrace 说"探针不存在"

```text
dtrace: invalid probe specifier sdt:bugdemo::cmd-start: probe does not exist
```

原因：

- **模块未加载。** SDT 探针由提供它们的模块定义。如果模块未加载，探针不存在。
- **探针名称不匹配。** 源代码中的名称使用双下划线（`cmd__start`），但 DTrace 使用单短划线（`cmd-start`）。这是转换规则；下划线形式出现在 C 中，短划线形式出现在 DTrace 中。
- **提供者未定义。** 如果 `SDT_PROVIDER_DEFINE(bugdemo)` 缺失或与 `SDT_PROBE_DEFINE` 在不同文件中，探针将不存在。

### 内核构建因符号冲突失败

当用不寻常的选项组合构建内核时，你可能看到类似"multiple definition of X"的链接错误。原因：

- **选项冲突。** 某些选项互斥。查看 `/usr/src/sys/conf/NOTES` 中的选项文档。
- **过期对象。** 旧的构建产物可能干扰新的构建。尝试在内核构建目录中运行 `make cleandir && make cleandir`。
- **源码树不一致。** `/usr/src/` 的部分更新可能使头文件和源码不同步。运行完整的 `svnlite update` 或 `git pull` 然后重试。

### 系统启动到旧内核

`installkernel` 后重启，`uname -v` 显示旧内核。原因：

- **启动条目未更新。** 默认是 `kernel`，指向当前内核。如果你用 `KERNCONF=GENERIC-DEBUG` 安装但没有干净地运行 `make installkernel`，旧二进制可能仍在原位。检查 `/boot/kernel/kernel` 的时间戳。
- **加载器中选择了错误的内核。** FreeBSD 加载器菜单有"Boot Kernel"选项，可以在可用内核之间选择。选择正确的，或在 `/boot/loader.conf` 中设置 `kernel="kernel"`。
- **启动分区未更改。** 在某些系统上启动分区是独立的，需要手动复制。检查你是否安装到了正确的分区。

### WITNESS 报告误报

有时 `WITNESS` 警告一个你知道是安全的锁序。可能的原因和回应：

- **顺序确实不安全但在实践中无害。** `WITNESS` 报告潜在的死锁，而非实际的死锁。锁图中从未被并发行使的环仍然是等待发生的 bug。重构加锁。
- **锁按地址获取。** 按指针加锁的通用代码可能产生取决于运行时数据而非静态结构的顺序。参见 `witness(4)` 了解如何使用 `witness_skipspin` 或手动覆盖来抑制特定顺序。
- **同类型的多个锁。** 获取同一锁类的两个实例始终是潜在问题。如果你需要将它们视为单独的类，使用带不同类型名称的 `mtx_init(9)`。

## 综合运用：一次调试会话演练

在结束之前，让我们走完一个使用我们学过的几种技术的完整调试会话。场景是虚构但真实的：一个驱动程序有时以误导性的错误失败，我们从首发症状追踪到根因。

### 症状

一位用户报告他们的程序有时从 `/dev/bugdemo` 的 ioctl 得到 `EBUSY`。程序总是以相同方式调用 ioctl，大多数时候工作正常。只有在重负载下 `EBUSY` 才会出现，而且不一致。

### 步骤 1：收集证据

第一步是在不干扰现象的情况下观察它。我们在 `ktrace(1)` 下运行用户程序以确认症状：

```console
$ ktrace -t ci ./user_program
$ kdump | grep ioctl
```

输出确认特定的 ioctl 有时返回 `EBUSY`。没有其他用户空间调用行为异常。这告诉我们 bug 在内核对该 ioctl 的处理中，而非用户程序的逻辑中。

### 步骤 2：形成假设

错误代码 `EBUSY` 通常表示资源冲突。阅读驱动程序源代码，我们发现 `EBUSY` 在内部标志指示前一个操作仍在进行时返回。该标志由完成操作的 callout 清除。

我们的假设是：在重负载下，callout 被延迟到足够长，以至于新的 ioctl 在前一个完成之前到达。驱动程序未设计为序列化此类请求，因此它拒绝了新的请求。

### 步骤 3：用 DTrace 测试假设

我们编写一个 DTrace 脚本，记录连续 ioctl 之间的延迟和每次入口时忙碌标志的状态：

```console
dtrace -n '
fbt::bugdemo_ioctl:entry
{
        self->ts = timestamp;
}

fbt::bugdemo_ioctl:return
/self->ts != 0/
{
        @[pid, self->result] = lquantize(timestamp - self->ts, 0, 1000000, 10000);
        self->ts = 0;
}
'
```

在负载下运行用户程序，我们观察到 `EBUSY` 返回几乎专门发生在前一个 ioctl 完成超过 50 微秒且 callout 尚未触发时。这证实了假设。

### 步骤 4：用 SDT 探针确认

我们在忙碌标志操作周围添加 SDT 探针并观察它们：

```console
dtrace -n '
sdt:bugdemo::set-busy
{
        printf("%lld set busy\n", timestamp);
}

sdt:bugdemo::clear-busy
{
        printf("%lld clear busy\n", timestamp);
}

sdt:bugdemo::reject-busy
{
        printf("%lld reject busy\n", timestamp);
}
'
```

跟踪显示了一个清晰的模式：设置、拒绝、拒绝、拒绝、清除、设置、清除。清除来得晚，因为 callout 在与其他工作竞争共享的 taskqueue。

### 步骤 5：确定修复方案

有了收集到的证据，修复方案很清楚。要么驱动程序需要序列化传入的 ioctl 而非拒绝它们（队列或等待），要么需要同步完成前一个操作而非通过 callout。

我们选择排队方法，因为它保留了 callout 的好处。驱动程序累积待处理请求并在 callout 触发时分发它们。轻负载下没有任何变化。重负载下，请求等待而非失败。

### 步骤 6：实现并验证

我们修改驱动程序。在原始负载下运行用户程序。`EBUSY` 不再出现。DTrace 延迟分布现在显示反映排队延迟的尾部，这对该驱动的用例是可接受的。

我们还在驱动的 malloc 类型上启用 `DEBUG_MEMGUARD` 并运行负载一段时间，以确保排队代码没有引入内存 bug。没有页面错误触发。

最后，我们运行完整的测试套件。一切通过。修复以解释根因而非仅仅症状的描述性消息提交。

### 本次会话的教训

有两点值得注意。

第一，我们使用的工具相对轻量。不需要崩溃转储。不需要进入 `DDB`。bug 是通过被动观察、DTrace 和仔细阅读来诊断的。对于许多驱动程序 bug，这就是会话的形态：不是戏剧性的 panic，而是系统性的假设缩小。

第二，修复解决了根因而非症状。表面修复可能是提高 callout 的 taskqueue 优先级。那会降低 bug 的频率但不消除它。更原则性的修复将驱动程序的契约从"忙碌时拒绝"改为"排队并服务"。这就是我们在第 8 节讨论的重构心态：每个 bug 都是关于设计的信息。

## 值得了解的附加技术

本章涵盖了 FreeBSD 调试工具包的核心。少数附加技术不适合放在主线叙述中，但值得一提，因为你最终会遇到它们。

### witness_checkorder 与手动列表

`WITNESS` 可以被调优。在 `/usr/src/sys/kern/subr_witness.c` 中有一个已知良好锁序的表，内核能识别它。在构建与由现有锁锁定的子系统交互的驱动程序时，将驱动程序自己的锁添加到此表让 `WITNESS` 能验证跨驱动程序和子系统的组合顺序。

这在小型驱动程序中很少需要，但对与多个子系统深度交互的驱动程序变得有用。

### sysctl debug.ktr

除了启用和禁用 `ktr(9)` 类，还有额外的控制：

- `debug.ktr.clear=1` 清除缓冲区。
- `debug.ktr.verbose=1` 实时将跟踪条目发送到控制台，除了环之外。
- `debug.ktr.stamp=1` 为每个条目添加时间戳。

这些组合在你想要观看实时跟踪而不反复运行 `ktrdump(8)` 时特别有用。

### 超越 bt 的 DDB 命令

`DDB` 有丰富的命令集，文档稀疏。有几个对驱动程序作者特别有用：

- `show all procs` 列出每个进程。
- `show lockedvnods` 显示当前锁定的 vnode（对存储驱动程序 bug 有用）。
- `show mount` 显示挂载的文件系统。
- `show registers` 转储 CPU 寄存器。
- `break FUNC` 设置断点。
- `step` 和 `next` 推进一条指令或一行。
- `watch` 在地址上设置观察点。

`DDB` 中的 `help` 命令列出所有可用命令。阅读一次列表是发现你不知道的功能的有用方式。

### 内核选项 KDB_TRACE

`KDB_TRACE` 使内核在每次 panic 时打印栈跟踪，即使操作员不与调试器交互。这在无人值守控制台的自动化测试中有用。它已在 `GENERIC` 中。

### EKCD：加密内核崩溃转储

如果内核转储包含敏感数据（进程内存、凭证、密钥），内核可以在转储时加密它们。`EKCD` 选项启用此功能。公钥在运行时用 `dumpon -k` 加载；匹配的私钥在 `savecore` 时用于解密。

这在转储可能通过不受信通道传输的生产系统上很重要。在开发虚拟机上不重要。

### 轻量级调试输出：bootverbose

另一个低开销选项是 `bootverbose`。在加载器中设置 `boot_verbose` 或在 sysctl 中设置 `bootverbose=1` 会导致许多内核子系统在启动时打印额外的诊断信息。如果你的驱动程序尚未到达 DTrace 适用的阶段，`bootverbose` 可以帮助你在 `attach` 期间看到驱动程序在做什么。

让你自己的驱动程序遵循 `bootverbose` 的方式是在探测或 attach 代码中检查 `bootverbose`：

```c
if (bootverbose)
        device_printf(dev, "detailed attach info: ...\n");
```

这是 `/usr/src/sys/dev/` 驱动程序中的成熟模式。

## 深入了解 DDB

内核内调试器 `DDB` 值得比我们到目前为止给予的更多关注。许多驱动程序作者只在 panic 意外将他们投入其中时才被动使用 `DDB`。通过一点练习，`DDB` 也是可以主动进入的有用工具，用于交互式检查运行的内核。

### 进入 DDB

有几种进入 `DDB` 的方式。我们已经见过其中一些：

- 通过 panic，如果 `debug.debugger_on_panic` 非零。
- 通过串行 BREAK（或键盘控制台上的 `Ctrl-Alt-Esc`），当编译了 `BREAK_TO_DEBUGGER` 时。
- 通过替代序列 `CR ~ Ctrl-B`，当编译了 `ALT_BREAK_TO_DEBUGGER` 时。
- 通过编程方式，使用 `sysctl debug.kdb.enter=1`。
- 从代码中，通过调用 `kdb_enter(9)`。

在开发中，编程方式进入最方便。你可以在脚本中在特定点投入 `DDB` 而不等 panic。

### DDB 提示符和命令

一旦进入，`DDB` 呈现提示符。标准提示符就是 `db>`。输入命令后按回车。`DDB` 有命令历史（在串行控制台上按上箭头）和许多命令名的 tab 补全。

一个有用的第一个命令是 `help`，列出命令类别。`help show` 列出许多 `show` 子命令。大多数探索通过 `show` 完成。

### 遍历线程

`DDB` 中最常见的诊断任务是遍历特定线程。从 `ps` 开始，列出所有进程：

```console
db> ps
  pid  ppid  pgrp  uid  state  wmesg   wchan    cmd
    0     0     0    0  RL     (swapper) [...] swapper
    1     0     1    0  SLs    wait     [...] init
  ...
  500   499   500    0  SL     nanslp   [...] user_program
```

选择感兴趣的线程。在 `DDB` 中，通过 `show thread` 命令切换线程：

```console
db> show thread 100012
  Thread 100012 at 0xfffffe00...
  ...
db> bt
```

这遍历该特定线程的栈。内核死锁调查通常涉及遍历每个卡住的线程，看它在等待什么。

### 检查结构

如果内核用 `DDB_CTF` 构建，`DDB` 可以解引用指针并打印结构字段。示例：

```console
db> show proc 500
db> show malloc
db> show uma
```

每个命令打印相关内核状态的格式化视图。`show malloc` 给出 malloc 类型及其当前分配的表。`show uma` 对 UMA 区域做同样的。`show proc` 详细显示特定进程。

### 设置断点

`DDB` 支持断点。`break FUNC` 在函数入口设置断点。`continue` 恢复执行。当断点触发时，内核返回 `DDB`，你可以检查该点的状态。

这是使 `DDB` 成为真正调试器而非仅是崩溃检查器的机制。使用断点你可以在特定代码位置暂停内核、检查参数，并决定是否继续。

注意事项是在 `DDB` 中暂停的内核确实是暂停的。当你在 `DDB` 中时，没有其他线程运行。在网络服务器上，每个客户端超时。在桌面上，GUI 冻结。对于本地开发虚拟机调试，这没问题。对于任何远程或共享使用，不行。

### 脚本化 DDB

`DDB` 支持简单的脚本功能。你可以定义命名脚本，执行一系列 `DDB` 命令。`script kdb.enter.panic=bt; show registers; show proc` 使这三个命令在每次因 panic 进入调试器时自动运行。这对无人值守的转储有用：脚本化输出出现在控制台和转储中，为你提供信息而无需交互式会话。

脚本存储在内核内存中，可以在启动时通过 `/boot/loader.conf` 或运行时通过 `sysctl` 调用配置。参见 `ddb(4)` 获取确切语法。

### 退出 DDB

完成后，`continue` 退出 `DDB`，内核恢复运行。`reset` 重启。`call doadump` 强制转储并重启。`call panic` 故意触发 panic（当你想从当前状态获取转储但不是通过 panic 到达 `DDB` 时有用）。

对于在虚拟机上练习的开发者，`continue` 是要记住的命令。它让内核恢复运行，让你继续工作。

### DDB 与 kgdb：何时使用哪个

`DDB` 和 `kgdb` 有重叠但不可互换。

当内核正在运行（或暂停在特定事件上）而你想要到处看看时使用 `DDB`。`DDB` 在内核内部运行，直接访问内核内存和线程。它是快速状态检查、设置断点和在特定事件上停止的正确工具。

当机器重启后在崩溃转储上使用 `kgdb`。`kgdb` 没有访问运行系统线程的能力，但它有完整的 gdb 功能用于离线分析：命令历史、源代码浏览、Python 脚本等等。

对于你不能重启的运行中内核，`KDB` 的 GDB 桩后端弥合了差距：内核暂停，另一台机器上的 `kgdb` 通过串行线附加，在实时状态上获得完整的 gdb 功能。这是最强大的组合，但需要两台机器（或虚拟机）。

## 实战演练示例：跟踪空指针

为了将工具整合在一起，让我们再走一个演练示例。症状是：`bugdemo` 偶尔以 `page fault: supervisor read instruction` 和经过 `bugdemo_read` 的回溯 panic。panic 地址很低，暗示空指针解引用。

### 步骤 1：捕获转储

panic 后，我们确认转储已保存：

```console
# ls -l /var/crash/
```

并打开它：

```console
# kgdb /boot/kernel/kernel /var/crash/vmcore.last
```

### 步骤 2：阅读回溯

```console
(kgdb) bt
#0  __curthread ()
#1  doadump (textdump=0) at /usr/src/sys/kern/kern_shutdown.c
#2  db_fncall_generic at /usr/src/sys/ddb/db_command.c
...
#8  bugdemo_read (dev=..., uio=..., ioflag=0)
    at /usr/src/sys/modules/bugdemo/bugdemo.c:185
```

有趣的帧是 8，`bugdemo_read`。第 185 行的代码是：

```c
sc = dev->si_drv1;
amt = MIN(uio->uio_resid, sc->buflen);
```

### 步骤 3：检查变量

```console
(kgdb) frame 8
(kgdb) print sc
$1 = (struct bugdemo_softc *) 0x0
(kgdb) print dev->si_drv1
$2 = (void *) 0x0
```

`si_drv1` 在设备上为 NULL。这是 `make_dev(9)` 设置的私有指针；它应该在 attach 期间被设置。

### 步骤 4：回溯

```console
(kgdb) print *dev
```

我们看到设备结构。名称字段显示"bugdemo"，标志看起来合理，但 `si_drv1` 是 NULL。有东西清除了它。

### 步骤 5：形成假设

在源代码中，`si_drv1` 设置一次，在 `attach` 中，在每个 `read`、`write` 和 `ioctl` 处理程序中读取。它从未被显式清除。然而，在卸载路径中，设备用 `destroy_dev(9)` 销毁，它在待处理处理程序完成之前就返回了。如果卸载开始时有一个 `read` 正在进行，设备可能被部分销毁。

### 步骤 6：添加断言

在 `bugdemo_read` 顶部的 `KASSERT` 捕获这种情况：

```c
KASSERT(sc != NULL, ("bugdemo_read: no softc"));
```

有了这个断言，下一次 panic 给我们相同的信息而不需要遍历转储。我们也立即知道条件是真实的，而非随机损坏。

### 步骤 7：修复 Bug

真正的修复是让卸载路径在销毁设备之前等待待处理处理程序。FreeBSD 为此目的提供了 `destroy_dev_drain(9)`。使用它：

```c
destroy_dev_drain(sc->dev);
```

确保在 softc 被释放时没有 read 或 write 正在进行。

### 步骤 8：验证

加载修复后的驱动程序。并发运行 read 和 unload。panic 不再重现。`KASSERT` 留在代码中作为未来重构的安全网。

### 要点

这个工作流（捕获、阅读、检查、假设、验证）是大多数高效调试会话的形态。每个工具扮演一个小而具体的角色。纪律是在行动之前收集证据，并留下断言作为未来的见证。

## 从第一天起让驱动可观测

贯穿本章的一个主题是添加调试基础设施的最佳时机是在你需要它之前。设计时就考虑可观测性的驱动程序比只为速度设计的驱动程序更容易调试。

一些具体的习惯支持这一点。

### 为每个分配器类型命名

`MALLOC_DEFINE(9)` 需要一个短名称和一个长名称。短名称是出现在 `vmstat -m` 输出和 `memguard(9)` 目标中的名称。选择一个描述性的、驱动程序唯一的名称，使后续诊断更容易。永远不要在不相关的子系统之间共享 malloc 类型；工具无法区分它们。

### 计数重要事件

驱动程序中的每个主要事件（open、close、read、write、中断、错误、状态转换）都是 `counter(9)` 的候选。计数器很便宜，它们随时间累积，并通过 sysctl 暴露。拥有良好计数器的驱动程序无需任何额外工具就能回答大多数"这东西在做什么"的问题。

### 声明 SDT 探针

每个状态转换都是 SDT 探针的候选。与断言或计数器不同，探针在禁用时没有成本。将它们保留在源代码中，伴随驱动程序的整个生命周期是净收益：当 bug 出现时，DTrace 可以看到事件流而无需重建。

### 使用一致的日志消息

`log(9)` 消息应遵循一致的格式。标识驱动程序的前缀、特定的错误代码或状态，以及足够的上下文来定位问题，这些是基本要素。避免在日志消息中耍小聪明；时间压力下的读者想知道发生了什么，而非欣赏你的文采。

### 提供有用的 sysctl

每个内部标志、每个计数器、每个配置值都应通过 sysctl 暴露，除非有特定原因不这样做。需要调试你驱动程序的人会感谢你；从不需要调试你驱动程序的人不为这种暴露付出任何代价。

### 边写代码边写断言

添加 `KASSERT` 的最佳时机是在不变式在你脑海中最清晰的时候，即你编写代码时。之后再回去撒断言效果较差，因为你已经忘记了一些不变式，并将其他一些合理化为"显而易见"。

### 暴露状态机的状态

每个非平凡的驱动程序都有状态机。通过 sysctl 暴露当前状态、在每个转换处添加 SDT 探针、以及为每个状态设置计数器，使状态机对人和工具都可见。这对异步驱动程序特别重要，这是下一章的主题。

### 测试卸载路径

未充分加固的卸载路径是经典的崩溃来源。在开发中，编写一个测试，加载驱动程序、短暂使用它、然后卸载它，在各种条件下重复执行。如果驱动程序无法承受一百次加载/卸载循环，它就有 bug。

这些习惯在开发中花费一点时间，但在调试中多次回本。有纪律的驱动程序作者把它们全部应用，即使对看起来太简单而用不上的驱动程序也是如此。

## 实用阅读清单

本章中的每种工具在其自己的手册页或源文件中都有更完整的文档。这是好消息：你不必把整个工具箱装在脑子里。当 bug 将你引向特定子系统时，打开正确的手册页或源文件几乎总是比任何章节都走得更远。以下列表按照你可能需要它们的顺序，汇集了对本材料最重要的参考。

`witness(4)` 手册页是当 `GENERIC-DEBUG` 开始打印锁序反转而你想要确切理解输出含义、哪些 `sysctl` 控制改变行为、以及哪些计数器可以检查时首先要阅读的。它记录了 `debug.witness.*` sysctl、`show all_locks` DDB 命令，以及 `WITNESS` 簿记的一般方法。对于实际实现，`/usr/src/sys/kern/subr_witness.c` 是权威来源。阅读它维护的结构和产生输出的函数（那些产生你之前在本章中看到的"1st ... 2nd ..."行的函数）消除了 `WITNESS` 报告中大部分的神秘感。该文件很长，但顶部注释和产生输出的函数加在一起涵盖了驱动程序作者需要了解的大部分内容。

对于锁性能分析，`lockstat(1)` 记录在 `/usr/src/cddl/contrib/opensolaris/cmd/lockstat/lockstat.1` 中。手册页末尾有几个演练示例，其输出格式与你将在自己系统上看到的匹配，这使得它在你第一次在真实负载上尝试 `-H -s 8` 时是一个有用的参考。因为 `lockstat(1)` 是基于 DTrace 的，`dtrace(1)` 手册页是它的天然伴侣；你可以用原始 D 表达相同的查询，如果你需要 `lockstat` 命令行标志不提供的灵活性。

对于内核调试器工作，`ddb(4)` 完整记录了内核内调试器，包括每个内建命令、每个脚本钩子，以及每种进入调试器的方式。有疑问时，在使用你之前未尝试过的 DDB 命令之前阅读此页面。对于离线事后分析，你安装的 FreeBSD 系统上的 `kgdb(1)` 记录了在标准 `gdb` 之上的内核特定扩展。底层访问层在 libkvm 中，记录在 `/usr/src/lib/libkvm/kvm_open.3`，它解释了你在第 3 节中遇到的转储和运行中内核两种模式。

两个较小的指针值得保留在你的阅读队列中。第一个是 `/usr/src/share/examples/witness/lockgraphs.sh`，一个随基本系统发布的小型 shell 脚本，演示了如何将 `WITNESS` 累积的锁序图转换为可视化图表。在真实驱动程序上运行一次就能给你一张你的锁相对于内核其余部分锁层次结构的位置图，这可能令人惊讶。第二个是 FreeBSD 内核源码树本身：阅读 `/usr/src/sys/kern/kern_shutdown.c`（panic 和转储路径）和 `/usr/src/sys/kern/kern_mutex.c`（`WITNESS` 插桩的互斥锁实现）将整个调试工作流建立在实际实现它的代码之上。

在源码树之外，FreeBSD 开发者手册和 FreeBSD 架构手册都包含关于内核调试的较长文章。两者都随任何 FreeBSD 系统上的文档集发布，并与源代码同步更新。它们值得浏览一次，即使你不从头读到尾，因为它们为你以后在自己的调试会话中会识别的模式赋予了名称。

关于选择参考的最后一点。手册页比博客文章老化得更慢，源代码注释比手册页老化得更慢。当两个参考不一致时，信任源代码，然后是手册页，然后是手册，然后是其他一切。这个层次结构数十年来很好地服务了 FreeBSD 开发者，这个习惯将在本书的其余部分和你的职业生涯中为你服务。

## 总结

高级调试是一门耐心的手艺。本章中的每种工具之所以存在，是因为某个人在某处遇到了无法通过其他方式找到的 bug。`KASSERT` 存在是因为只存在于程序员脑海中的不变式不是不变式。`kgdb` 和崩溃转储存在是因为有些 bug 会摧毁产生它们的机器。`DDB` 存在是因为冻结的内核无法通过其他任何渠道解释自己。`WITNESS` 存在是因为死锁在生产中是灾难性的而且在事后不可能调试。`memguard(9)` 存在是因为静默的内存损坏曾是最难的一类 bug，直到有人构建了一个让它变得响亮的工具。

这些工具都不能替代理解。调试器不能告诉你驱动程序应该做什么。崩溃转储不能告诉你正确的加锁纪律。DTrace 不能推断你的设计。工具是乐器；你是演奏者。音乐是你正在构建的驱动程序的形态。

使这门手艺成功的习惯是微小而不引人注目的。在调试内核上开发。为你能表述的每个不变式添加断言。常规地捕获转储，这样你可以毫无仪式地打开它们。在追查困难的东西时保持日志。当某个机制困扰你时阅读 FreeBSD 源代码。选择能回答你问题的最轻量工具，只在更轻量的工具不足时才升级到更重的工具。

调试也是一门社交手艺。一个花了你一天才找到的 bug，写清楚后可以为另一个作者节省一周。好的提交消息、详细的测试用例、以及对什么有效什么无效的诚实记录，都是对公共实践的贡献。FreeBSD 项目对 bug 报告的历史耐心、在提交日志中捕获根因的习惯，以及数十年来在驱动程序中一致使用 `KASSERT` 和 `WITNESS`，都源于将 bug 狩猎视为共同责任的集体习惯。

你现在有了参与其中的工具包。加载调试内核，在 `/usr/src/sys/dev/` 中选一个你感兴趣的驱动程序，用调试器的眼睛阅读它。不变式在哪里？断言在哪里？加锁纪律在哪里？bug 可能藏在哪里，什么工具能捕获它？这个练习磨砺本书其余部分一直在建立的直觉。

在下一章中，我们将把正确性放在一边，看看驱动程序如何处理异步 I/O 和事件驱动的工作：驱动程序一次为多个用户服务而不阻塞的模式，以及使这种设计成为可能的内核设施。你在这里获得的调试技能将在那个领域很好地为你服务，因为异步代码正是微妙并发 bug 倾向栖息的地方。一个有坚实断言、经 `WITNESS` 验证的干净锁序、以及一组 SDT 探针来跟踪其事件流的驱动程序，也是一个当其工作分散在回调、定时器和内核线程之间时更容易推理的驱动程序。

## 通往第35章的桥梁：异步I/O与事件处理

第 35 章从本章结束的地方接续。同步代码容易推理：一个调用到达，驱动程序做它的工作，调用返回。异步代码则不然：回调在不可预测的时间触发，事件乱序到达，驱动程序必须管理跨越多个线程上下文持久存在的状态。

异步驱动程序的复杂性正是本章工具所擅长的那类复杂性。有 bug 的同步驱动程序可能在可预测的位置崩溃。有 bug 的异步驱动程序可能在数小时后在一个与原始误行为没有明显联系的回调中崩溃。在每个回调入口对状态使用 `KASSERT` 可以早早捕获这类 bug。在每个事件转换上使用 DTrace 探针使序列可见。`WITNESS` 检测当多个异步路径需要协调时自然产生的死锁。

在下一章中，我们将认识 FreeBSD 中异步工作的构建块：`callout(9)` 用于延迟定时器、`taskqueue(9)` 用于后台工作、`kqueue(9)` 用于事件通知，以及正确使用它们的模式。我们将构建一个为许多并发用户服务而不阻塞的驱动程序，并将运用本章的调试技术来控制那种复杂性。

到完成第 35 章时，你将拥有完整的同步和异步工具包：能高效处理流量、扩展到多用户、在并发下维护正确性、以及在出问题时能被调试的驱动程序。这种组合正是在生产环境中存活下来的驱动程序所需要的。

第 35 章见。
