---
title: "FreeBSD 内核 API 参考"
description: "本书驱动开发章节中使用的 FreeBSD 内核 API、宏、数据结构及手册页系列的实用查询参考。"
appendix: "A"
lastUpdated: "2026-04-20"
status: "complete"
author: "Edson Brandi"
reviewer: "TBD"
translator: "AI辅助翻译为简体中文"
language: "zh-CN"
estimatedReadTime: 45
---

# 附录 A：FreeBSD 内核 API 参考

## 如何使用本附录

本附录是本书在 FreeBSD 驱动程序中教授的所有内容的配套查询表。主要章节循序渐进地构建每个 API，在可工作的驱动程序中展示它们，并解释其背后的思维模型。本附录是您在编码、调试或阅读他人驱动程序时保持打开的简明、可扫描的对应内容。

它被特意设计为参考而非教程。它不试图从零开始教授任何子系统。每个条目都假设您已经在书中某处遇到过该 API，或者您愿意在使用前阅读手册页。条目为您提供的是导航词汇：API 的用途、真正重要的少数名称、您可能犯的错误、在驱动程序生命周期中的典型位置，以及详细讲解它的章节。如果条目称职，您可以在一分钟内回答四个问题：

1. 我需要哪个 API 系列来解决眼前的问题？
2. 我想要的函数、宏或类型的确切名称是什么？
3. 在信任它之前我应该检查什么注意事项？
4. 我接下来应该打开哪个手册页或章节？

这就是所有承诺。下面的每个细节都根据 FreeBSD 14.3 源代码树和 `man 9` 中相应的手册页进行了验证。当某个区别很重要但推迟到本书其他部分时，条目会向前指引用户，而不是假装在此处解决它。

### 条目组织方式

本附录按 API 解决的问题进行分组，而不是按字母顺序。驱动程序很少孤立地使用某个名称。它使用整个系列：内存及其标志、锁及其睡眠规则、callout 及其取消故事。将这些系列保持在一起使附录对实际查询任务更有用。

在每个系列中，每个条目都遵循相同的简短模式：

- **用途。** API 的用途，一两句话。
- **驱动程序中的典型用途。** 驱动程序何时使用它。
- **关键名称。** 您实际调用或声明的函数、宏、标志或类型。
- **头文件。** 声明所在的位置。
- **注意事项。** 导致真正错误的少数几个错误。
- **生命周期阶段。** API 通常出现在 probe、attach、正常运行或 detach 中的哪个位置。
- **手册页。** 接下来要阅读的 `man 9` 条目。
- **本书讲解位置。** 完整上下文的章节参考。

扫描时请记住这个模式。如果您只需要标志的名称，查看**关键名称**。如果您只需要手册页，查看**手册页**。如果您忘记了 API 存在的原因，阅读**用途**然后停止。

### 本附录不是什么

本附录不是 `man 9` 的替代品，不是本书教学章节的替代品，也不是阅读 `/usr/src/sys/dev/` 下真实驱动程序的替代品。它故意保持简短。权威参考仍然是手册页；权威思维模型仍然是介绍该 API 的章节；权威真理仍然是源代码树。本附录帮助您快速找到这三者。

它也不涵盖每个内核接口。内核很大，完整的参考会重复属于附录 E（FreeBSD 内部结构和内核参考）或专注于第 16 章（访问硬件）、第 19 章（处理中断）和第 20 章（高级中断处理）等章节的内容。这里的目标是覆盖驱动程序作者在日常工作中实际使用的 API，以及驱动程序作者实际需要的详细程度。

## 读者指南

您可以以三种不同的方式使用本附录，每种方式都需要不同的阅读策略。

如果您**正在编写新代码**，请将其视为检查清单。选择与您的问题匹配的系列，浏览条目，记下关键名称，然后跳转到手册页或章节了解详情。时间投入：每次查询一到两分钟。

如果您**正在调试**，请将其视为假设地图。当驱动程序行为异常时，错误几乎总是作者忽略的注意事项：在可睡眠拷贝期间持有的互斥锁、已停止但未排空的 callout、在中断拆除之前释放的资源。每个条目的**注意事项**行是这些假设所在的位置。按顺序阅读它们，询问您的驱动程序是否遵守每一个。

如果您**正在阅读不熟悉的驱动程序**，请将其视为翻译器。当您看到不认识的函数或宏时，在本附录中找到其系列，阅读**用途**，然后继续。完整的理解可以稍后从章节或手册页中获得。探索期间的目标是保持前进并形成驱动程序正在做什么的初始思维模型。

全文使用的几个约定：

- 所有源代码路径以面向书籍的形式显示，`/usr/src/sys/...`，匹配标准 FreeBSD 系统上的布局。
- 手册页以通常的 FreeBSD 风格引用：`mtx(9)` 表示手册的第 9 节。您可以使用例如 `man 9 mtx` 阅读其中任何一个。
- 当系列没有专门的手册页时，条目会说明并指向最接近的可用文档。
- 当本书将主题推迟到后面的章节或附录 E 时，条目会向前指引而不是在此处编造细节。

考虑到这一点，我们可以正式开始本附录。第一个系列是内存：驱动程序从哪里获得它们需要的字节，如何归还它们，以及哪些标志控制整个过程的行为。

## 内存分配

每个驱动程序都分配内存，每次分配都带有关于何时可以阻塞、内存在物理上位于何处以及如何归还的规则。内核提供三个主要分配器：`malloc(9)` 用于通用分配，`uma(9)` 用于高频固定大小对象，`contigmalloc(9)` 用于硬件可以寻址的物理连续范围。下面您将看到每个分配器的概览，以及它们共享的小型标志词汇表。

### `malloc(9)` / `free(9)` / `realloc(9)`

**用途。** 通用内核内存分配器。为您提供任意大小的字节缓冲区，由 `malloc_type` 标记，以便您稍后可以使用 `vmstat -m` 对其进行统计。

**驱动程序中的典型用途。** Softc 分配、小型可变大小缓冲区、临时暂存空间，以及固定大小区域会过度杀戮的任何情况。

**关键名称。**

- `void *malloc(size_t size, struct malloc_type *type, int flags);`
- `void free(void *addr, struct malloc_type *type);`
- `void *realloc(void *addr, size_t size, struct malloc_type *type, int flags);`
- `MALLOC_DEFINE(M_FOO, "foo", "description for vmstat -m");`
- `MALLOC_DECLARE(M_FOO);` 用于头文件中的使用。

**头文件。** `/usr/src/sys/sys/malloc.h`。

**重要标志。**

- `M_WAITOK`：调用者可以阻塞直到内存可用。分配将成功或内核将 panic。
- `M_NOWAIT`：调用者不得阻塞。分配可能返回 `NULL`。使用 `M_NOWAIT` 时始终检查 `NULL`。
- `M_ZERO`：在返回之前将内存清零。与任一等待标志结合使用。
- `M_NODUMP`：从崩溃转储中排除分配。

**注意事项。**

- 在持有自旋互斥锁、在中断过滤器中或在任何不能睡眠的上下文中不得使用 `M_WAITOK`。
- `M_NOWAIT` 调用者必须检查返回值。未能处理 `NULL` 是审查中最常见的驱动程序崩溃之一。
- 切勿混合分配器系列。`malloc(9)` 返回的内存必须由 `free(9)` 释放；`uma_zfree(9)` 和 `contigfree(9)` 不可互换。
- `struct malloc_type` 指针必须在 `malloc` 和相应的 `free` 之间匹配。

**生命周期阶段。** 最常见于 `attach`（softc、缓冲区）和 `detach`（释放）。较小的分配可以出现在正常 I/O 路径中，只要上下文允许所选标志。

**手册页。** `malloc(9)`。

**本书讲解位置。** 在第 5 章中与内核特定的 C 习语一起介绍；在第 7 章中当您的第一个驱动程序分配其 softc 时使用；在第 10 章中当 I/O 缓冲区变得真实时重新讨论；在第 11 章中当分配标志必须遵守锁定规则时再次讨论。

### `uma(9)` 区域

**用途。** 固定大小对象缓存，针对频繁、统一和性能敏感的分配进行了高度优化。重用对象而不是重复访问通用分配器。

**驱动程序中的典型用途。** 网络 mbuf 类结构、每包状态、每请求描述符，以及每秒分配和释放数百万个相同小对象的任何情况。

**关键名称。**

- `uma_zone_t uma_zcreate(const char *name, size_t size, uma_ctor, uma_dtor, uma_init, uma_fini, int align, uint32_t flags);`
- `void uma_zdestroy(uma_zone_t zone);`
- `void *uma_zalloc(uma_zone_t zone, int flags);`
- `void uma_zfree(uma_zone_t zone, void *item);`

**头文件。** `/usr/src/sys/vm/uma.h`。

**标志。**

- 分配时的 `M_WAITOK`、`M_NOWAIT`、`M_ZERO`，含义与 `malloc(9)` 相同。
- 创建时标志如 `UMA_ZONE_ZINIT`、`UMA_ZONE_NOFREE`、`UMA_ZONE_CONTIG` 和对齐提示（`UMA_ALIGN_CACHE`、`UMA_ALIGN_PTR` 等）针对特定工作负载调整行为。

**注意事项。**

- 区域必须在使用前创建，必须在模块卸载前销毁。在 `detach` 中忘记 `uma_zdestroy` 会泄漏整个区域。
- 构造函数和析构函数分别在分配和释放时运行，而不是在区域创建和销毁时；使用 `init` 和 `fini` 回调进行每 slab 一次的工作。
- 创建区域代价高昂。每个模块每个对象类型创建一个，而不是每个实例一个。
- 没有专门的 `uma(9)` 手册页。权威参考是头文件和 `/usr/src/sys/` 下的现有用户。

**生命周期阶段。** `uma_zcreate` 在模块加载或早期 attach 中；`uma_zalloc` 和 `uma_zfree` 在 I/O 路径中；`uma_zdestroy` 在模块卸载中。

**手册页。** 没有专门的手册页。阅读 `/usr/src/sys/vm/uma.h` 并查看 `/usr/src/sys/kern/kern_mbuf.c` 和 `/usr/src/sys/net/netisr.c` 了解实际使用。

**本书讲解位置。** 在第 7 章中简要提及作为 `malloc(9)` 的替代方案；在第 28 章（网络）和第 33 章（性能调优）中当高速率驱动程序需要它时重新讨论。

### `contigmalloc(9)` / `contigfree(9)`

**用途。** 在指定地址窗口内分配物理连续的内存范围。当硬件必须在没有 IOMMU 的情况下 DMA 到内存并因此需要连续物理页面时需要。

**驱动程序中的典型用途。** 不能分散-聚集的设备的 DMA 缓冲区，并且仅在确认 `bus_dma(9)` 不是更好的选择之后。

**关键名称。**

- `void *contigmalloc(unsigned long size, struct malloc_type *type, int flags, vm_paddr_t low, vm_paddr_t high, unsigned long alignment, vm_paddr_t boundary);`
- `void contigfree(void *addr, unsigned long size, struct malloc_type *type);`

**头文件。** `/usr/src/sys/sys/malloc.h`。

**注意事项。**

- 启动后的碎片化使大型连续分配失败。不要假设成功。
- 对于几乎所有现代硬件，优先使用 `bus_dma(9)` 框架。它以可移植的方式处理标签、映射、弹跳和对齐。
- `contigmalloc` 分配是稀缺的系统资源；尽快释放它们。

**生命周期阶段。** 通常在 `attach` 中；在 `detach` 中释放。

**手册页。** `contigmalloc(9)`。

**本书讲解位置。** 在第 21 章中当 DMA 首次成为真正关注时与 `bus_dma(9)` 一起提及。

### 分配标志速查表

| 标志         | 含义                                                |
| :----------- | :----------------------------------------------------- |
| `M_WAITOK`   | 调用者可以阻塞直到内存可用。           |
| `M_NOWAIT`   | 调用者不得阻塞；失败时返回 `NULL`。     |
| `M_ZERO`     | 返回前将分配清零。                  |
| `M_NODUMP`   | 从崩溃转储中排除分配。              |

仅在允许睡眠的地方使用 `M_WAITOK`。有疑问时，安全的答案是 `M_NOWAIT` 加上 `NULL` 检查。

## 同步原语

如果内存是驱动程序的原材料，同步就是防止两个执行上下文同时破坏它的纪律。FreeBSD 为您提供了一个小型、设计良好的工具包。下面的名称是您最常遇到的。完整的教学在第 11、12 和 15 章，中断上下文细微差别在第 19 和 20 章；本附录收集词汇。

### `mtx(9)`：互斥锁

**用途。** 默认的内核互斥原语。线程获取锁，进入临界区，并释放锁。

**驱动程序中的典型用途。** 保护 softc 字段、环形缓冲区、引用计数和临界区短且不睡眠的任何共享状态。

**关键名称。**

- `void mtx_init(struct mtx *m, const char *name, const char *type, int opts);`
- `void mtx_destroy(struct mtx *m);`
- `mtx_lock(m)`, `mtx_unlock(m)`, `mtx_trylock(m)`.
- `mtx_assert(m, MA_OWNED | MA_NOTOWNED | MA_RECURSED | MA_NOTRECURSED);` 用于不变量。
- 互斥锁睡眠辅助：`msleep(9)` 和 `mtx_sleep(9)`。

**头文件。** `/usr/src/sys/sys/mutex.h`。

**选项。**

- `MTX_DEF`：默认的可在竞争时睡眠的互斥锁。几乎用于所有情况。
- `MTX_SPIN`：纯自旋锁。中断过滤器上下文和其他无法阻塞的地方需要。规则更严格。
- `MTX_RECURSE`：允许同一线程多次获取锁。谨慎使用；它通常隐藏设计错误。
- `MTX_NEW`：强制 `mtx_init` 将锁视为新创建的。与 `WITNESS` 一起使用。

**注意事项。**

- 在持有 `MTX_DEF` 或 `MTX_SPIN` 互斥锁时切勿睡眠。`uiomove(9)`、`copyin(9)`、`copyout(9)`、`malloc(9, M_WAITOK)` 和大多数总线原语都可以睡眠。仔细审计。
- 始终将 `mtx_init` 与 `mtx_destroy` 配对。忘记销毁会泄漏内部状态并惹恼 `WITNESS`。
- 锁顺序很重要。一旦内核看到您在锁 B 之前获取锁 A，如果您曾经颠倒该对，它将警告。提前规划您的锁层次结构。
- `MTX_SPIN` 禁用抢占；尽可能短地持有它。

**生命周期阶段。** `mtx_init` 在 `attach` 中；`mtx_destroy` 在 `detach` 中。锁定和解锁操作在两者之间的任何位置。

**手册页。** `mutex(9)`、`mtx_pool(9)`、`msleep(9)`。

**本书讲解位置。** 第一次处理在第 11 章，在第 12 章中深化锁顺序和 `WITNESS` 纪律，并在第 19 章中重新讨论中断安全变体（`MTX_SPIN`）。

### `sx(9)`：可睡眠共享-独占锁

**用途。** 读者-写者锁，其中读者或写者都可以阻塞。当多个读者常见、写者罕见且临界区可能睡眠时使用。

**驱动程序中的典型用途。** 被许多路径读取并很少修改的配置状态。不用于快速路径数据。

**关键名称。**

- `void sx_init(struct sx *sx, const char *desc);`
- `void sx_destroy(struct sx *sx);`
- `sx_slock(sx)`, `sx_sunlock(sx)` 用于共享访问。
- `sx_xlock(sx)`, `sx_xunlock(sx)` 用于独占访问。
- `sx_try_slock`, `sx_try_xlock`, `sx_upgrade`, `sx_downgrade`.
- `sx_assert(sx, SA_SLOCKED | SA_XLOCKED | SA_LOCKED | SA_UNLOCKED);`

**头文件。** `/usr/src/sys/sys/sx.h`。

**注意事项。**

- `sx` 允许在临界区内部睡眠，与 `mtx` 不同。这种灵活性是整个目的；确保您确实需要它。
- `sx` 锁比互斥锁更昂贵。不要默认使用它们。
- 避免在同一个锁顺序中混合 `sx` 和 `mtx`，而不仔细考虑其含义。

**生命周期阶段。** `sx_init` 在 `attach` 中；`sx_destroy` 在 `detach` 中。

**手册页。** `sx(9)`。

**本书讲解位置。** 第 12 章。

### `rmlock(9)`：读多锁

**用途。** 极快的读者路径，较慢的写者路径。读者之间不竞争。设计用于每次操作都读取但很少写入的数据。

**驱动程序中的典型用途。** 类路由表、快速路径中使用的配置状态、写入开销可接受因为写入罕见的结构。

**关键名称。**

- `void rm_init(struct rmlock *rm, const char *name);`
- `void rm_destroy(struct rmlock *rm);`
- `rm_rlock(rm, tracker)`, `rm_runlock(rm, tracker)`.
- `rm_wlock(rm)`, `rm_wunlock(rm)`.

**头文件。** `/usr/src/sys/sys/rmlock.h`。

**注意事项。**

- 每个读者需要自己的 `struct rm_priotracker`，通常在栈上。不要共享一个。
- 读者不得睡眠，除非锁是用 `RM_SLEEPABLE` 初始化的。
- 写者路径很重；如果写入频繁，`sx` 或 `mtx` 是更好的选择。

**生命周期阶段。** `rm_init` 在 `attach` 中；`rm_destroy` 在 `detach` 中。

**手册页。** `rmlock(9)`。

**本书讲解位置。** 在第 12 章中简要介绍，并在出现读多模式的后面章节中使用。

### `cv(9)` / `condvar(9)`：条件变量

**用途。** 命名等待通道。一个或多个线程睡眠，直到另一个线程发出信号表示它们等待的条件已变为真。

**驱动程序中的典型用途。** 等待缓冲区排空、硬件完成命令或特定状态转换。当您希望等待原因明确时使用，而不是裸 `wakeup(9)` 通道。

**关键名称。**

- `void cv_init(struct cv *cv, const char *desc);`
- `void cv_destroy(struct cv *cv);`
- `cv_wait(cv, mtx)`, `cv_wait_sig(cv, mtx)`, `cv_wait_unlock(cv, mtx)`.
- `cv_timedwait(cv, mtx, timo)`, `cv_timedwait_sig(cv, mtx, timo)`.
- `cv_signal(cv)`, `cv_broadcast(cv)`, `cv_broadcastpri(cv, pri)`.

**头文件。** `/usr/src/sys/sys/condvar.h`。

**注意事项。**

- 传递给 `cv_wait` 的互斥锁必须由调用者持有；`cv_wait` 在睡眠时释放它并在返回时重新获取它。
- `cv_wait` 返回后始终重新检查谓词。虚假唤醒和信号是可能的。
- `cv_signal` 唤醒一个等待者；`cv_broadcast` 唤醒所有。根据设计选择，而不是本能。

**生命周期阶段。** `cv_init` 在 `attach` 中；`cv_destroy` 在 `detach` 中。

**手册页。** `condvar(9)`。

**本书讲解位置。** 第 12 章，可中断和定时等待在第 15 章中重新讨论。

### `sema(9)`：计数信号量

**用途。** 具有 `wait` 和 `post` 操作的计数信号量。比互斥锁或条件变量更少见。

**驱动程序中的典型用途。** 必须跟踪计数资源的生产者-消费者模式，例如固定池的命令槽。

**关键名称。**

- `void sema_init(struct sema *sema, int value, const char *desc);`
- `void sema_destroy(struct sema *sema);`
- `sema_wait(sema)`, `sema_trywait(sema)`, `sema_timedwait(sema, timo)`.
- `sema_post(sema)`.

**头文件。** `/usr/src/sys/sys/sema.h`。

**注意事项。**

- 信号量适合计数。对于一线程在临界区中的模式，改用 `mtx`。
- `sema_wait` 可能因信号而提前返回；检查返回值。

**手册页。** `sema(9)`。

**本书讲解位置。** 第 15 章，作为高级同步工具包的一部分。

### `atomic(9)`：原子操作

**用途。** 单字、无中断的读-修改-写操作。比任何锁都快，表达能力严格受限。

**驱动程序中的典型用途。** 计数器、标志和比较并交换模式，其中临界区适合一个整数。

**关键名称。**

- `atomic_add_int`, `atomic_subtract_int`, `atomic_set_int`, `atomic_clear_int`.
- `atomic_load_int`, `atomic_store_int`，以及获取和释放变体。
- `atomic_cmpset_int`, `atomic_fcmpset_int` 用于比较并交换。
- 宽度变体：`_8`、`_16`、`_32`、`_64` 和指针大小的 `_ptr`。
- 屏障辅助：`atomic_thread_fence_acq()`、`atomic_thread_fence_rel()`、`atomic_thread_fence_acq_rel()`。

**头文件。** `/usr/src/sys/sys/atomic_common.h` 以及 `machine/atomic.h` 用于架构特定部分。

**注意事项。**

- 原子操作给您一个字的互斥。跨越两个字段的任何不变量仍然需要锁。
- 内存顺序很重要。普通操作是宽松的；当一个访问必须在另一个之前或之后变得可见时，使用 `_acq`、`_rel` 和 `_acq_rel` 变体。
- 对于很少读取的每 CPU 计数器，`counter(9)` 扩展性更好。

**生命周期阶段。** 任何。足够便宜可以在中断过滤器中使用。

**手册页。** `atomic(9)`。

**本书讲解位置。** 第 11 章，`counter(9)` 在旁边介绍用于每 CPU 模式。

### `epoch(9)`：读多无锁段

**用途。** 为读者远多于写者且延迟必须最小的数据结构提供轻量级读者保护。写者等待所有当前读者离开后再释放内存。

**驱动程序中的典型用途。** 网络栈快速路径、高性能驱动程序中的读多查找表。不是通用原语。

**关键名称。**

- `epoch_t epoch_alloc(const char *name, int flags);`
- `void epoch_free(epoch_t epoch);`
- `epoch_enter(epoch)`, `epoch_exit(epoch)`.
- `epoch_wait(epoch)` 用于写者阻塞直到读者排空。
- `NET_EPOCH_ENTER(et)` 和 `NET_EPOCH_EXIT(et)` 包装器用于网络栈。

**头文件。** `/usr/src/sys/sys/epoch.h`。

**注意事项。**

- 读者在 epoch 段内不得阻塞、睡眠或调用任何这样做的函数。
- 受保护内存的释放必须推迟到 `epoch_wait` 返回。
- Epoch 段是最后的手段工具，不是默认原语。首先选择锁。

**手册页。** `epoch(9)`。

**本书讲解位置。** 在第 12 章中简要介绍；仅在后面章节中实际驱动程序需要时深入使用。

### 锁决策速查表

| 您想要...                                        | 使用                  |
| :---------------------------------------------------- | :------------------------- |
| 保护短的、不睡眠的临界区        | `mtx(9)` with `MTX_DEF`    |
| 在中断过滤器中保护状态                  | `mtx(9)` with `MTX_SPIN`   |
| 允许多个读者、罕见写者、可能睡眠          | `sx(9)`                    |
| 允许多个读者、罕见写者、读者不睡眠 | `rmlock(9)`                |
| 睡眠直到命名条件成立                   | `cv(9)` with a mutex       |
| 递增或比较并换单个字          | `atomic(9)`                |
| 读多数据的无锁读者路径           | `epoch(9)`                 |

当一行不明显匹配问题时，第 11、12 和 15 章中的完整讨论是解决它的地方。

## 延迟执行和定时器

驱动程序通常需要稍后运行工作、定期运行或从可以睡眠的上下文运行。内核为此提供三个工具：`callout(9)` 用于单次和周期性定时器，`taskqueue(9)` 用于可能睡眠的延迟工作，`kthread(9)` 或 `kproc(9)` 用于长时间运行的后台线程。它们在某些情况下重叠；经验法则是 callout 从定时器中断上下文运行（快、不睡眠），taskqueue 在工作线程中运行（可以睡眠、可以获取可睡眠锁），kthread 是您拥有的整个线程。

### `callout(9)`：内核定时器

**用途。** 安排函数在时间延迟后运行。回调默认在软中断上下文中运行，不得睡眠。

**驱动程序中的典型用途。** 看门狗定时器、轮询间隔、重试延迟、空闲超时。

**关键名称。**

- `void callout_init(struct callout *c, int mpsafe);` 加上 `callout_init_mtx` 和 `callout_init_rm`。
- `int callout_reset(struct callout *c, int ticks, void (*func)(void *), void *arg);`
- `int callout_stop(struct callout *c);`
- `int callout_drain(struct callout *c);`
- `int callout_pending(struct callout *c);`, `callout_active(struct callout *c);`

**头文件。** `/usr/src/sys/sys/callout.h`。

**注意事项。**

- `callout_stop` 不等待正在运行的回调。在 `detach` 中释放 softc 之前使用 `callout_drain`。
- 即使您认为已取消它，callout 也可能触发，如果定时器已经分发。用标志保护回调或使用 `_mtx` 和 `_rm` 变体将取消与您的锁集成。
- 运行无 tick 内核意味着 tick 是抽象的。用 `hz` 转换实时或使用 `callout_reset_sbt` 获得亚秒精度。

**生命周期阶段。** `callout_init` 在 `attach` 中；`callout_drain` 在 `detach` 中；`callout_reset` 每当需要设置下次触发时间时。

**手册页。** `callout(9)`。

**本书讲解位置。** 第 13 章。

### `taskqueue(9)`：工作线程中的延迟工作

**用途。** 将工作从不能睡眠或不应长时间持有锁的上下文移交给工作线程。在同一 taskqueue 上排队的任务按顺序运行。

**驱动程序中的典型用途。** 中断后处理、硬件命令完成处理程序、可能需要分配内存或获取可睡眠锁的重置和恢复路径。

**关键名称。**

- `struct taskqueue *taskqueue_create(const char *name, int mflags, taskqueue_enqueue_fn, void *context);`
- `void taskqueue_free(struct taskqueue *queue);`
- `TASK_INIT(struct task *t, int priority, task_fn_t *func, void *context);`
- `int taskqueue_enqueue(struct taskqueue *queue, struct task *task);`
- `void taskqueue_drain(struct taskqueue *queue, struct task *task);`
- `void taskqueue_drain_all(struct taskqueue *queue);`
- 全局队列如 `taskqueue_thread`、`taskqueue_swi`、`taskqueue_fast`。

**头文件。** `/usr/src/sys/sys/taskqueue.h`。

**注意事项。**

- 在同一任务运行之前两次排队同一任务按设计是无操作。如果您每次需要新请求，那没问题；如果您期望两次运行，使用不同的任务。
- `taskqueue_drain` 等待任务完成；在释放任务使用的任何内容之前调用它。
- 私有 taskqueue 便宜但不免费。重用全局 taskqueue（`taskqueue_thread`、`taskqueue_fast`），除非您有理由拥有一个。

**生命周期阶段。** `taskqueue_create`（如果私有）和 `TASK_INIT` 在 `attach` 中；`taskqueue_drain` 和 `taskqueue_free` 在 `detach` 中。

**手册页。** `taskqueue(9)`。

**本书讲解位置。** 第 14 章。

### `kthread(9)` 和 `kproc(9)`：内核线程和进程

**用途。** 创建运行您函数的专用内核线程或进程。当工作负载是长时间运行的、需要自己的调度策略或需要明确可寻址时有用。

**驱动程序中的典型用途。** 罕见。大多数驱动程序工作由 taskqueue 或 callout 更好地服务。内核线程出现在具有真正长时间运行循环的子系统中，例如内务守护进程。

**关键名称。**

- `int kthread_add(void (*func)(void *), void *arg, struct proc *p, struct thread **td, int flags, int pages, const char *fmt, ...);`
- `int kproc_create(void (*func)(void *), void *arg, struct proc **procp, int flags, int pages, const char *fmt, ...);`
- `void kthread_exit(void);`
- `kproc_exit`, `kproc_suspend_check`.

**头文件。** `/usr/src/sys/sys/kthread.h`。

**注意事项。**

- 创建线程比排队任务更重。除非工作负载真正长时间运行，否则优先使用 `taskqueue(9)`。
- 干净地关闭 kthread 需要合作：设置停止标志、唤醒线程并等待它退出。忘记任何步骤都会在模块卸载时泄漏线程。
- kthread 必须通过调用 `kthread_exit` 退出，而不是通过返回。

**手册页。** `kthread(9)`、`kproc(9)`。

**本书讲解位置。** 在第 14 章中作为比 taskqueue 更重的替代方案提及。

### 延迟工作决策速查表

| 您需要...                                                   | 使用         |
| :--------------------------------------------------------------- | :---------------- |
| 延迟后触发函数、简短、不睡眠              | `callout(9)`      |
| 延迟可能睡眠或获取可睡眠锁的工作               | `taskqueue(9)`    |
| 运行持久的后台循环                                 | `kthread(9)`      |
| 将短期周期性轮询转换为真实中断             | 见第 19 章    |

## 总线和资源管理

总线层是驱动程序与硬件相遇的地方。Newbus 将驱动程序介绍给内核；`rman(9)` 分发代表 MMIO 区域、I/O 端口和中断的资源；`bus_space(9)` 可移植地访问它们；`bus_dma(9)` 让设备安全地 DMA。

### Newbus：`DRIVER_MODULE`、`DEVMETHOD` 和相关宏

**用途。** 向内核注册驱动程序、将其绑定到设备类、声明内核应调用的入口点以及发布版本和依赖信息。

**驱动程序中的典型用途。** 每个拥有设备的内核模块。这是将一堆 C 代码变成 `kldload` 可以附加到硬件的东西的脚手架。

**关键名称。**

- `DRIVER_MODULE(name, bus, driver, devclass, evh, evharg);`
- `MODULE_VERSION(name, version);`
- `MODULE_DEPEND(name, busname, vmin, vpref, vmax);`
- `DEVMETHOD(method, function)` 和 `DEVMETHOD_END` 用于方法表。
- `device_method_t` 条目如 `device_probe`、`device_attach`、`device_detach`、`device_shutdown`、`device_suspend`、`device_resume`。
- 类型：`device_t`、`devclass_t`、`driver_t`。

**头文件。** `/usr/src/sys/sys/module.h` 和 `/usr/src/sys/sys/bus.h`。

**注意事项。**

- `DRIVER_MODULE` 展开为模块事件处理程序；除非您确切知道原因，否则不要手动声明自己的 `module_event_t` 表。
- `MODULE_DEPEND` 是您让加载器引入先决条件的方式。忘记它会在加载时产生丑陋的符号解析失败。
- `DEVMETHOD_END` 终止方法表。没有它，内核将越过末尾。
- `device_t` 是不透明的；使用访问器如 `device_get_softc`、`device_get_parent`、`device_get_name` 和 `device_printf`。

**生命周期阶段。** 仅声明。宏展开为在 `kldload` 和 `kldunload` 上运行的模块初始化和模块终结粘合代码。

**手册页。** `DRIVER_MODULE(9)`、`MODULE_VERSION(9)`、`MODULE_DEPEND(9)`、`module(9)`、`DEVICE_PROBE(9)`、`DEVICE_ATTACH(9)`、`DEVICE_DETACH(9)`。

**本书讲解位置。** 第 7 章完整处理，第 6 章首次勾勒剖析。

### `devclass(9)` 和设备访问器

**用途。** `devclass_t` 将同一驱动程序的实例分组，以便内核可以找到它们、编号它们并遍历它们。在驱动程序中，您主要使用访问器，而不是直接使用 devclass。

**关键名称。**

- `device_t device_get_parent(device_t dev);`
- `void *device_get_softc(device_t dev);`
- `int device_get_unit(device_t dev);`
- `const char *device_get_nameunit(device_t dev);`
- `int device_printf(device_t dev, const char *fmt, ...);`
- `devclass_find`, `devclass_get_device`, `devclass_get_devices`, `devclass_get_count` 当您真正需要遍历类时。

**头文件。** `/usr/src/sys/sys/bus.h`。

**注意事项。**

- `device_get_softc` 假设 softc 是通过驱动程序结构注册的。自己滚动 `device_t` 到状态的映射几乎总是错误的。
- 驱动程序中直接操作 devclass 很罕见。如果您发现自己需要它，检查问题是否属于总线级接口。

**手册页。** `devclass(9)`、`device(9)`、`device_get_softc(9)`、`device_printf(9)`。

**本书讲解位置。** 第 6 章和第 7 章。

### `rman(9)`：资源管理器

**用途。** MMIO 区域、I/O 端口、中断号和 DMA 通道的统一视图。您的驱动程序按类型和 RID 请求资源，并返回带有有用访问器的 `struct resource *`。

**关键名称。**

- `struct resource *bus_alloc_resource(device_t dev, int type, int *rid, rman_res_t start, rman_res_t end, rman_res_t count, u_int flags);`
- `struct resource *bus_alloc_resource_any(device_t dev, int type, int *rid, u_int flags);`
- `int bus_release_resource(device_t dev, int type, int rid, struct resource *r);`
- `int bus_activate_resource(device_t dev, int type, int rid, struct resource *r);`
- `int bus_deactivate_resource(device_t dev, int type, int rid, struct resource *r);`
- `rman_res_t rman_get_start(struct resource *r);`, `rman_get_end`, `rman_get_size`.
- `bus_space_tag_t rman_get_bustag(struct resource *r);`, `rman_get_bushandle`.
- 资源类型：`SYS_RES_MEMORY`、`SYS_RES_IOPORT`、`SYS_RES_IRQ`、`SYS_RES_DRQ`。
- 标志：`RF_ACTIVE`、`RF_SHAREABLE`。

**头文件。** `/usr/src/sys/sys/rman.h`。

**注意事项。**

- `rid` 参数是指针，可能被分配器重写。传递真实变量的地址。
- 在 `detach` 中以分配的反向顺序释放每个分配的资源。泄漏资源几乎总是会破坏下一次 attach。
- `RF_ACTIVE` 是常见情况。不要忘记它，否则您将获得无法与 `bus_space(9)` 一起使用的句柄。
- 始终检查返回值。在有怪癖的硬件上分配失败很常见。

**生命周期阶段。** `attach` 中分配；`detach` 中释放。如果驱动程序有特殊需求，`bus_activate_resource` 和 `bus_deactivate_resource` 可以单独管理激活。

**手册页。** `rman(9)`、`bus_alloc_resource(9)`、`bus_release_resource(9)`、`bus_activate_resource(9)`。

**本书讲解位置。** 第 16 章。

### `bus_space(9)`：可移植寄存器访问

**用途。** 通过 `(tag, handle, offset)` 三元组读取和写入设备寄存器，隐藏底层访问是内存映射、基于端口、大端、小端还是索引。

**驱动程序中的典型用途。** 每个 MMIO 或 I/O 端口访问。不要自己解引用 `rman_get_virtual`；使用 `bus_space`。

**关键名称。**

- 类型：`bus_space_tag_t`、`bus_space_handle_t`。
- 读取：`bus_space_read_1(tag, handle, offset)`, `_2`, `_4`, `_8`.
- 写入：`bus_space_write_1(tag, handle, offset, value)`, `_2`, `_4`, `_8`.
- 多寄存器辅助：`bus_space_read_multi_N`、`bus_space_write_multi_N`、`bus_space_read_region_N`、`bus_space_write_region_N`。
- 屏障：`bus_space_barrier(tag, handle, offset, length, flags)` 使用 `BUS_SPACE_BARRIER_READ` 和 `BUS_SPACE_BARRIER_WRITE`。

**头文件。** `/usr/src/sys/sys/bus.h`，机器特定细节在 `machine/bus.h` 中。

**注意事项。**

- 切勿通过原始指针访问设备寄存器。可移植性和调试都依赖于 `bus_space`。
- 屏障不是自动的。当两个写入必须按顺序发生时，在它们之间插入 `bus_space_barrier`。
- `bus_space_read_N` 或 `bus_space_write_N` 中使用的宽度必须匹配寄存器的自然大小。不匹配会在某些架构上导致静默损坏。

**生命周期阶段。** 驱动程序与设备对话的任何时间。

**手册页。** `bus_space(9)`。

**本书讲解位置。** 第 16 章。

### `bus_dma(9)`：可移植 DMA

**用途。** 用标签描述 DMA 约束、通过映射加载缓冲区，让框架处理对齐、弹跳和一致性。任何移动数据的严肃设备都需要。

**关键名称。**

- `int bus_dma_tag_create(bus_dma_tag_t parent, bus_size_t alignment, bus_addr_t boundary, bus_addr_t lowaddr, bus_addr_t highaddr, bus_dma_filter_t *filtfunc, void *filtfuncarg, bus_size_t maxsize, int nsegments, bus_size_t maxsegsz, int flags, bus_dma_lock_t *lockfunc, void *lockfuncarg, bus_dma_tag_t *dmat);`
- `int bus_dma_tag_destroy(bus_dma_tag_t dmat);`
- `int bus_dmamap_create(bus_dma_tag_t dmat, int flags, bus_dmamap_t *mapp);`
- `int bus_dmamap_destroy(bus_dma_tag_t dmat, bus_dmamap_t map);`
- `int bus_dmamap_load(bus_dma_tag_t dmat, bus_dmamap_t map, void *buf, bus_size_t buflen, bus_dmamap_callback_t *callback, void *arg, int flags);`
- `void bus_dmamap_unload(bus_dma_tag_t dmat, bus_dmamap_t map);`
- `void bus_dmamap_sync(bus_dma_tag_t dmat, bus_dmamap_t map, bus_dmasync_op_t op);`
- `int bus_dmamem_alloc(bus_dma_tag_t dmat, void **vaddr, int flags, bus_dmamap_t *mapp);`
- `void bus_dmamem_free(bus_dma_tag_t dmat, void *vaddr, bus_dmamap_t map);`
- 标志：`BUS_DMA_WAITOK`、`BUS_DMA_NOWAIT`、`BUS_DMA_ALLOCNOW`、`BUS_DMA_COHERENT`、`BUS_DMA_ZERO`。
- 同步操作：`BUS_DMASYNC_PREREAD`、`BUS_DMASYNC_POSTREAD`、`BUS_DMASYNC_PREWRITE`、`BUS_DMASYNC_POSTWRITE`。

**头文件。** `/usr/src/sys/sys/bus_dma.h`。

**注意事项。**

- 标签形成树。子标签继承父标签约束；按正确顺序创建它们。
- `bus_dmamap_load` 可能异步完成。始终使用回调，即使对于同步缓冲区。
- `bus_dmamap_sync` 不是装饰。没有正确的同步方向，缓存和设备内存将不一致。
- 在有 IOMMU 的平台上，框架会做正确的事情。不要仅仅因为您的开发硬件是一致的而跳过它。

**生命周期阶段。** `attach` 中标签创建和映射设置；I/O 路径中加载和同步；`detach` 中卸载和销毁。

**手册页。** `bus_dma(9)`。

**本书讲解位置。** 第 21 章。

### 中断设置

**用途。** 将过滤器或处理程序附加到 IRQ 资源，以便内核可以将中断传递给驱动程序。

**关键名称。**

- `int bus_setup_intr(device_t dev, struct resource *r, int flags, driver_filter_t *filter, driver_intr_t *handler, void *arg, void **cookiep);`
- `int bus_teardown_intr(device_t dev, struct resource *r, void *cookie);`
- 标志：`INTR_TYPE_NET`、`INTR_TYPE_BIO`、`INTR_TYPE_TTY`、`INTR_TYPE_MISC`、`INTR_MPSAFE`、`INTR_EXCL`。

**注意事项。**

- 当快速路径决策便宜且驱动程序可以遵守过滤器上下文限制（不睡眠、无可睡眠锁）时提供过滤器。当工作需要线程时提供处理程序。
- `INTR_MPSAFE` 对于新驱动程序是强制性的。没有它，内核在 Giant 锁上序列化处理程序，这几乎总是错误的。
- 在释放资源之前拆除。顺序是：`bus_teardown_intr`，然后 `bus_release_resource`。

**生命周期阶段。** `bus_setup_intr` 在 `attach` 末尾，在 softc 其余部分准备好之后；`bus_teardown_intr` 在 `detach` 开始时，在任何资源释放之前。

**手册页。** `BUS_SETUP_INTR(9)`、`bus_alloc_resource(9)` 用于资源方面。

**本书讲解位置。** 第 19 章，高级模式在第 20 章。

## 设备节点和字符设备 I/O

一旦绑定硬件，驱动程序最常通过 `/dev/` 中的设备节点向用户空间公开自己。下面的 API 构建和拆除这些节点，相关的开关表声明内核应该如何分发 `read`、`write`、`ioctl` 和 `poll`。

### `make_dev_s(9)` 和 `destroy_dev(9)`

**用途。** 在 `/dev/` 下创建新设备节点，连接到包含内核将调用的函数指针的 `cdevsw`。

**关键名称。**

- `int make_dev_s(struct make_dev_args *args, struct cdev **cdev, const char *fmt, ...);`
- `void destroy_dev(struct cdev *cdev);`
- `struct make_dev_args` 中的字段：`mda_si_drv1`、`mda_devsw`、`mda_uid`、`mda_gid`、`mda_mode`、`mda_flags`、`mda_unit`。
- 旧式辅助 `make_dev(struct cdevsw *, int unit, uid_t, gid_t, int mode, const char *fmt, ...)` 仍然存在，但在新代码中首选 `make_dev_s`。

**头文件。** `/usr/src/sys/sys/conf.h`。

**注意事项。**

- 始终使用 `make_dev_s`。较旧的 `make_dev` 吞掉错误，不让您设置所有参数。
- 将 `mda_si_drv1` 设置为 softc，以便 cdev 携带指向驱动程序状态的指针，无需单独查找。
- `destroy_dev` 在返回之前等待所有活动线程离开 cdev，使得之后释放 softc 是安全的。

**生命周期阶段。** `make_dev_s` 在 `attach` 末尾；`destroy_dev` 在 `detach` 开始时，在拆除任何支持状态之前。

**手册页。** `make_dev(9)`。

**本书讲解位置。** 第 8 章。

### `cdevsw`：字符设备开关表

**用途。** 声明进程打开、读取、写入或以其他方式与设备节点交互时内核应调用的入口点。

**关键名称。**

- `struct cdevsw` 字段：`d_version`、`d_flags`、`d_name`、`d_open`、`d_close`、`d_read`、`d_write`、`d_ioctl`、`d_poll`、`d_kqfilter`、`d_mmap`、`d_mmap_single`。
- `d_version` 必须是 `D_VERSION`。
- 常见标志：`D_NEEDGIANT`（旧式）、`D_TRACKCLOSE`、`D_MEM`。

**头文件。** `/usr/src/sys/sys/conf.h`。

**注意事项。**

- 始终设置 `d_version = D_VERSION`。内核拒绝附加缺少或过时版本的开关表。
- `d_flags` 默认为零对于现代 MPSAFE 驱动程序是可以的。除非您真正需要，否则不要添加 `D_NEEDGIANT`。
- 未使用的条目可以保留为 `NULL`；内核替换默认值。不要将它们指向什么都不做的存根。

**生命周期阶段。** 在模块作用域静态声明。由 `struct make_dev_args` 引用。

**手册页。** `make_dev(9)` 在 `make_dev_s` 的上下文中涵盖该结构。

**本书讲解位置。** 第 8 章。

### `ioctl(9)` 分发

**用途。** 向设备提供带外命令，由数字命令和参数缓冲区寻址。

**关键名称。**

- 入口点：`d_ioctl_t` 签名为 `int (*)(struct cdev *, u_long cmd, caddr_t data, int fflag, struct thread *td);`
- 命令编码宏：`_IO`、`_IOR`、`_IOW`、`_IOWR`。
- 拷贝辅助：`copyin(9)` 和 `copyout(9)` 用于携带指针的 ioctl。

**注意事项。**

- 使用 `_IOR`、`_IOW` 或 `_IOWR` 声明命令。它们编码大小和方向，这对跨架构兼容性很重要。
- 在对命令参数采取行动之前验证它们。ioctl 是信任边界。
- 切勿直接解引用用户指针。使用 `copyin(9)` 和 `copyout(9)`。

**生命周期阶段。** 正常操作。

**手册页。** `ioctl(9)`（概念）；入口点与 `cdevsw` 一起记录在 `make_dev(9)` 中。

**本书讲解位置。** 第 24 章（第 3 节），进一步模式在第 25 章。

### `devfs_set_cdevpriv(9)`：每次打开状态

**用途。** 将驱动程序私有指针附加到打开的文件描述符。当最后一次关闭发生时，指针由回调释放。

**关键名称。**

- `int devfs_set_cdevpriv(void *priv, d_priv_dtor_t *dtor);`
- `int devfs_get_cdevpriv(void **datap);`
- `void devfs_clear_cdevpriv(void);`

**头文件。** `/usr/src/sys/sys/conf.h`。

**注意事项。**

- 每次打开状态是每次描述符设置、游标或待处理事务的正确工具。不要在 softc 中存储每次打开状态。
- 析构函数在最后一次关闭的上下文中运行。保持简短和非阻塞。

**手册页。** `devfs_set_cdevpriv(9)`。

**本书讲解位置。** 第 8 章。

## 进程和用户空间交互

用户空间不能信任内核地址，内核不能信任在没有小心的情况下跟随用户空间指针。下面的 API 安全地跨越信任边界。

### `copyin(9)`、`copyout(9)`、`copyinstr(9)`

**用途。** 在内核和用户地址空间之间移动字节并进行地址验证。这些是从内核代码触摸用户指针的唯一安全方式。

**关键名称。**

- `int copyin(const void *uaddr, void *kaddr, size_t len);`
- `int copyout(const void *kaddr, void *uaddr, size_t len);`
- `int copyinstr(const void *uaddr, void *kaddr, size_t len, size_t *done);`
- 相关：`fueword`、`fuword`、`subyte`、`suword`，记录在 `fetch(9)` 和 `store(9)` 下。

**头文件。** `/usr/src/sys/sys/systm.h`。

**注意事项。**

- 这三个都可能睡眠。在持有不可睡眠互斥锁时不要调用它们。
- 它们在错误地址上返回 `EFAULT`，而不是零。始终检查返回值。
- `copyinstr` 通过其 `done` 参数区分截断和成功；不要忽略它。

**生命周期阶段。** `d_ioctl`、`d_read`、`d_write` 和用户空间是源或目标的任何其他地方。

**手册页。** `copy(9)`、`fetch(9)`、`store(9)`。

**本书讲解位置。** 第 9 章。

### `uio(9)`：读写 I/O 描述符

**用途。** 内核自己对 I/O 请求的描述。隐藏用户和内核缓冲区之间的差异、分散-聚集和连续传输之间的差异以及读写方向之间的差异。

**关键名称。**

- `int uiomove(void *cp, int n, struct uio *uio);`
- `int uiomove_nofault(void *cp, int n, struct uio *uio);`
- `struct uio` 中的字段：`uio_iov`、`uio_iovcnt`、`uio_offset`、`uio_resid`、`uio_segflg`、`uio_rw`、`uio_td`。
- 段标志：`UIO_USERSPACE`、`UIO_SYSSPACE`、`UIO_NOCOPY`。
- 方向：`UIO_READ`、`UIO_WRITE`。

**头文件。** `/usr/src/sys/sys/uio.h`。

**注意事项。**

- 在 `d_read` 和 `d_write` 入口点中使用 `uiomove`。即使当用户空间缓冲区是简单的连续区域时，它也是正确的工具。
- `uiomove` 可能睡眠。在调用它之前放下不可睡眠互斥锁。
- `uiomove` 返回后，`uio_resid` 已更新。不要并行维护自己的字节计数；从 `uio_resid` 读取它。

**生命周期阶段。** 正常 I/O。

**手册页。** `uio(9)`。

**本书讲解位置。** 第 9 章。

### `proc(9)` 和驱动程序的线程上下文

**用途。** 访问调用线程及其进程，主要用于凭据检查、信号状态和诊断打印。

**关键名称。**

- `curthread`, `curproc`, `curthread->td_proc`.
- `struct ucred *cred = curthread->td_ucred;`
- `int priv_check(struct thread *td, int priv);`
- `pid_t pid = curproc->p_pid;`

**头文件。** `/usr/src/sys/sys/proc.h`。

**注意事项。**

- 直接使用进程内部很少见。当您需要它时，通常是为了凭据检查，应该通过 `priv_check(9)` 进行。
- 不要跨睡眠存储 `curthread`。重新进入驱动程序的线程可能是不同的线程。

**手册页。** 没有单独的页面；见 `priv(9)` 和 `proc(9)`。

**本书讲解位置。** 第 9 章和第 24 章中当 ioctl 处理程序需要凭据时引用。

## 可观察性和通知

无法被观察的驱动程序是无法被信任的驱动程序。内核为用户空间查看驱动程序状态、订阅事件和等待就绪提供了几种方式。下面的 API 是最常见的。

### `sysctl(9)`：读写配置节点

**用途。** 在层次结构名称下发布驱动程序状态和可调参数，以便 `sysctl(8)` 和监控脚本等工具可以读取或修改它们。

**关键名称。**

- 静态声明：`SYSCTL_NODE`、`SYSCTL_INT`、`SYSCTL_LONG`、`SYSCTL_STRING`、`SYSCTL_PROC`、`SYSCTL_OPAQUE`。
- 动态上下文 API：`sysctl_ctx_init`、`sysctl_ctx_free`、`SYSCTL_ADD_NODE`、`SYSCTL_ADD_INT`、`SYSCTL_ADD_PROC`。
- 处理程序辅助：`sysctl_handle_int`、`sysctl_handle_long`、`sysctl_handle_string`。
- 访问标志：`CTLFLAG_RD`、`CTLFLAG_RW`、`CTLFLAG_TUN`、`CTLFLAG_STATS`、`CTLTYPE_INT`、`CTLTYPE_STRING`。

**头文件。** `/usr/src/sys/sys/sysctl.h`。

**注意事项。**

- 对于绑定到特定设备实例的任何内容，使用动态 API。`device_get_sysctl_ctx` 和 `device_get_sysctl_tree` 为您提供正确的上下文。
- 处理程序在用户上下文中运行。它们可能睡眠并可能失败。
- 谨慎发布可调参数。每个旋钮都是与未来用户的契约。

**生命周期阶段。** 静态声明是模块范围的。动态声明在 `attach` 中创建，并通过上下文在 `detach` 中自动销毁。

**手册页。** `sysctl(9)`、`sysctl_add_oid(9)`、`sysctl_ctx_init(9)`。

**本书讲解位置。** 在第 7 章中介绍，在第 24 章（第 4 节）中当驱动程序开始向用户空间公开指标时深入处理。

### `eventhandler(9)`：内核内发布-订阅

**用途。** 注册内核范围的事件，如挂载、卸载、低内存和关机。内核响应调用注册的回调。

**关键名称。**

- `EVENTHANDLER_DECLARE(name, type_t);`
- `eventhandler_tag EVENTHANDLER_REGISTER(name, func, arg, priority);`
- `void EVENTHANDLER_DEREGISTER(name, tag);`
- `void EVENTHANDLER_INVOKE(name, ...);`
- 优先级常量：`EVENTHANDLER_PRI_FIRST`、`EVENTHANDLER_PRI_ANY`、`EVENTHANDLER_PRI_LAST`。

**头文件。** `/usr/src/sys/sys/eventhandler.h`。

**注意事项。**

- 处理程序同步运行。保持它们简短。
- 始终在模块卸载前注销。当事件触发时，悬空处理程序会 panic。

**手册页。** `EVENTHANDLER(9)`。

**本书讲解位置。** 第 24 章中当驱动程序与关机和低内存事件等内核范围通知集成时引用。

### `poll(2)` 和 `kqueue(2)`：就绪通知

**用途。** 让用户空间等待驱动程序拥有的就绪事件。`poll(2)` 是较旧的接口；`kqueue(2)` 是具有更丰富过滤器的现代接口。

**关键名称。**

- `poll` 的入口点：`int (*d_poll)(struct cdev *, int events, struct thread *);`
- `kqueue` 的入口点：`int (*d_kqfilter)(struct cdev *, struct knote *);`
- 等待列表管理：`struct selinfo`、`selrecord(struct thread *td, struct selinfo *sip)`、`selwakeup(struct selinfo *sip)`。
- kqueue 支持：`struct knote`、`knote_enqueue`、`knlist_init_mtx`、`knlist_add`、`knlist_remove`。
- 事件位：`poll` 的 `POLLIN`、`POLLOUT`、`POLLERR`、`POLLHUP`；`kqueue` 的 `EVFILT_READ`、`EVFILT_WRITE`。

**头文件。** `/usr/src/sys/sys/selinfo.h`、`/usr/src/sys/sys/event.h`、`/usr/src/sys/sys/poll.h`。

**注意事项。**

- 当没有事件就绪时 `d_poll` 必须调用 `selrecord`，当它们就绪时报告当前就绪状态。
- `selwakeup` 必须在不持有可能针对调度器反转的任何互斥锁的情况下调用。这是常见的锁顺序错误。
- `kqueue` 支持更丰富但也需要更多代码。当驱动程序已经有干净的 `poll` 路径时，将其扩展到 `kqueue` 通常是对的下一步，而不是重写。

**生命周期阶段。** `attach` 中设置；`detach` 中拆除；`d_poll` 或 `d_kqfilter` 中实际分发。

**手册页。** `selrecord(9)`、`kqueue(9)` 和用户空间页面 `poll(2)` 和 `kqueue(2)`。

**本书讲解位置。** 第 10 章完整介绍 `poll(2)` 集成；`kqueue(2)` 在那里引用并在第 35 章中深入探讨。

## 诊断、日志和跟踪

驱动程序正确性不仅存在于代码中。它存在于观察、断言和跟踪的能力中。下面的 API 是您让驱动程序讲述关于自身真相的方式。

### `log(9)` 和 `printf(9)`

**用途。** 向内核日志发送消息，以便它们出现在 `dmesg` 和 `/var/log/messages` 中。

**关键名称。**

- `void log(int level, const char *fmt, ...);`
- 标准内核 `printf` 系列：`printf`、`vprintf`、`uprintf`、`tprintf`。
- 每设备辅助：`device_printf(device_t dev, const char *fmt, ...);`
- 来自 `syslog.h` 的优先级常量：`LOG_EMERG`、`LOG_ALERT`、`LOG_CRIT`、`LOG_ERR`、`LOG_WARNING`、`LOG_NOTICE`、`LOG_INFO`、`LOG_DEBUG`。

**头文件。** `/usr/src/sys/sys/systm.h`、`/usr/src/sys/sys/syslog.h`。

**注意事项。**

- 不要在 I/O 快速路径上以 `LOG_INFO` 记录日志。它会淹没控制台并掩盖真正的问题。
- `device_printf` 自动添加设备名称前缀，使日志易于过滤。优先于裸 `printf`。
- 对每个不同的事件类记录一次，而不是每个包一次。

**生命周期阶段。** 任何。

**手册页。** `printf(9)`。

**本书讲解位置。** 第 23 章。

### `KASSERT(9)`：内核断言

**用途。** 声明必须为真的不变量。当内核使用 `INVARIANTS` 构建时，违反的断言会 panic 并显示描述性消息。没有 `INVARIANTS`，断言编译消失。

**关键名称。**

- `KASSERT(expression, (format, args...));`
- `MPASS(expression);` 用于更简单的无消息断言。
- `CTASSERT(expression);` 用于常量的编译时断言。

**头文件。** `/usr/src/sys/sys/kassert.h`，由 `/usr/src/sys/sys/systm.h` 传递包含。

**注意事项。**

- 表达式必须便宜且无副作用。编译器不会将其优化到位；您编写不变量。
- 消息是带括号的 `printf` 参数列表。包含足够的上下文以仅从 panic 诊断失败。
- 使用 `KASSERT` 表示指示程序员错误的条件，而不是正常运行时条件。

**生命周期阶段。** 必须记录和强制执行不变量的任何地方。

**手册页。** `KASSERT(9)`。

**本书讲解位置。** 第 23 章介绍 `INVARIANTS` 和断言使用；第 34 章第 2 节深入处理 `KASSERT` 和诊断宏。

### `WITNESS`：锁顺序验证器

**用途。** 跟踪每个线程获取锁的顺序并在后续线程颠倒先前观察到的顺序时警告的内核选项。

**关键名称。**

- 内置于 `mtx(9)`、`sx(9)`、`rm(9)` 和锁定宏。不需要单独的 API 调用。
- 内核选项：`WITNESS`、`WITNESS_SKIPSPIN`、`WITNESS_COUNT`。
- 与 `WITNESS` 合作的断言：`mtx_assert`、`sx_assert`、`rm_assert`。

**注意事项。**

- `WITNESS` 是调试选项。构建调试内核以启用它；对于生产来说太昂贵了。
- 警告不是噪音。如果 `WITNESS` 抱怨，就有错误。
- 锁顺序警告引用传递给 `mtx_init`、`sx_init` 等的锁名称。给每个锁一个有意义的名称。

**手册页。** 没有单独的页面。见 `lock(9)` 和 `locking(9)`。

**本书讲解位置。** 第 12 章（第 6 节），在第 23 章中加强。

### `ktr(9)`：内核跟踪设施

**用途。** 内核内部事件跟踪的低开销环形缓冲区。`ktr` 记录由宏发出，可以用 `ktrdump(8)` 转储。

**关键名称。**

- `CTR0(class, fmt)`, `CTR1(class, fmt, a1)`, 直到 `CTR6` 参数数量递增。
- 跟踪类：`KTR_GEN`、`KTR_NET`、`KTR_DEV` 和 `sys/ktr_class.h` 中的许多其他。
- 内核选项：带每类掩码的 `KTR`。

**头文件。** `/usr/src/sys/sys/ktr.h`。

**注意事项。**

- `ktr` 必须在内核构建时启用；检查配置中的 `KTR`。
- 每条记录很小。不要尝试记录整个结构。
- 对于面向用户的诊断，`dtrace(1)` 通常是更好的答案。

**手册页。** `ktr(9)`。

**本书讲解位置。** 第 23 章。

### DTrace 静态探针和主要提供者

**用途。** 静态和动态跟踪基础设施，让用户空间附加到运行内核中的探测点而无需重新编译。

**关键名称。**

- 静态定义跟踪：`SDT_PROVIDER_DECLARE`、`SDT_PROVIDER_DEFINE`、`SDT_PROBE_DECLARE`、`SDT_PROBE_DEFINE`、`SDT_PROBE`。
- FreeBSD 上的常见提供者：`sched`、`proc`、`io`、`vfs`、`fbt`（函数边界跟踪）、`sdt`。
- 头文件：`/usr/src/sys/sys/sdt.h`、`/usr/src/sys/cddl/dev/dtrace/...`。

**注意事项。**

- `fbt` 不需要对驱动程序的更改，但 `sdt` 探针为您提供命名的、稳定的点，这些点在未来重构中幸存。
- 禁用的探针成本可忽略不计。不要担心添加几个。
- DTrace 脚本本身是用户空间代码；驱动程序只定义脚本可以附加的探针点。

**手册页。** `SDT(9)`、`dtrace(1)`、`dtrace(8)`。

**本书讲解位置。** 第 23 章。

## 按驱动程序生命周期阶段交叉引用

相同的 API 出现在驱动程序生命周期的不同阶段。下表是一个快速反向索引：当您编写特定阶段时，这里有通常属于那里的系列。

### 模块加载

- `MODULE_VERSION`、`MODULE_DEPEND`、`DEV_MODULE`（如果模块是纯 cdev）。
- 静态 `MALLOC_DEFINE`、`SYSCTL_NODE`、`SDT_PROVIDER_DEFINE` 声明。
- 必须在任何设备附加之前幸存的事件处理程序注册。

### 探测

- `device_get_parent`、`device_get_nameunit`、`device_printf`。
- 返回值：`BUS_PROBE_DEFAULT`、`BUS_PROBE_GENERIC`、`BUS_PROBE_SPECIFIC`、`BUS_PROBE_LOW_PRIORITY`、不匹配时的 `ENXIO`。

### 附加

- `device_get_softc`、`malloc(9)` 用于 softc 字段、`MALLOC_DEFINE` 标签。
- 锁初始化：`mtx_init`、`sx_init`、`rm_init`、`cv_init`、`sema_init`。
- 资源分配：`bus_alloc_resource` 或 `bus_alloc_resource_any`。
- 通过 `rman_get_bustag` 和 `rman_get_bushandle` 设置 `bus_space`。
- DMA 脚手架：`bus_dma_tag_create`、`bus_dmamap_create`。
- `callout_init`、`TASK_INIT`、需要时创建 taskqueue。
- 中断设置：`bus_setup_intr`。
- 设备节点创建：`make_dev_s`。
- sysctl 树：`device_get_sysctl_ctx`、`SYSCTL_ADD_*`。
- `uma_zcreate` 用于高频对象。
- 绑定到此驱动程序的事件处理程序注册。

### 正常操作

- `d_open`、`d_close`、`d_read`、`d_write`、`d_ioctl`、`d_poll`、`d_kqfilter`。
- `uiomove`、`copyin`、`copyout`、`copyinstr`。
- `bus_space_read_*`、`bus_space_write_*`、`bus_space_barrier`。
- `bus_dmamap_load`、`bus_dmamap_sync`、`bus_dmamap_unload`。
- 锁定：`mtx_lock`、`mtx_unlock`、`sx_slock`、`sx_xlock`、`cv_wait`、`cv_signal`、`atomic_*`。
- 延迟工作：`callout_reset`、`taskqueue_enqueue`。
- 诊断：`device_printf`、`log`、`KASSERT`、`SDT_PROBE`。

### 分离

- 以反向附加顺序拆除。
- 在释放任何资源之前 `bus_teardown_intr`。
- 在拆除其引用的 softc 字段之前 `destroy_dev`。
- 在释放 callout 结构之前 `callout_drain`。
- 对于私有 taskqueue：`taskqueue_drain_all` 和 `taskqueue_free`。
- `bus_dmamap_unload`、`bus_dmamap_destroy`、`bus_dma_tag_destroy`。
- 对于 attach 中分配的每个资源 `bus_release_resource`。
- `cv_destroy`、`sx_destroy`、`mtx_destroy`、`rm_destroy`、`sema_destroy`。
- 对于驱动程序拥有的每个区域 `uma_zdestroy`。
- 事件处理程序注销。
- 任何分配内容的最终 `free` 或 `contigfree`。

### 模块卸载

- 验证没有设备实例仍然附加。Newbus 通常处理这个，但防御性 `DRIVER_MODULE` 事件处理程序应该在状态保留时拒绝卸载。

## 快速参考检查清单

这些检查清单旨在五分钟或更短时间内阅读。它们不取代章节中的教学；它们提醒您有经验的驱动程序作者不再忘记的事情。

### 锁定纪律检查清单

- softc 中的每个共享字段正好有一个锁保护它，在字段附近的注释中记录。
- 没有互斥锁在 `uiomove`、`copyin`、`copyout`、`malloc(9, M_WAITOK)` 或 `bus_alloc_resource` 期间持有。
- 锁顺序在文件顶部的注释中声明并在各处遵守。
- `mtx_assert` 或 `sx_assert` 出现在需要在入口时持有特定锁的函数上。
- `WITNESS` 在开发内核中启用，其警告被视为错误。
- 每个 `mtx_init` 都有匹配的 `mtx_destroy`，每种锁类型都一样。

### 资源生命周期检查清单

- `bus_setup_intr` 是 `attach` 中的最后一件事；`bus_teardown_intr` 是 `detach` 中的第一件事。
- 每个分配的资源在 `detach` 中以反向顺序有匹配的释放。
- 在它指向的结构被释放之前调用 `callout_drain`。
- 在任务结构或其参数被释放之前调用 `taskqueue_drain_all` 或 `taskqueue_drain`。
- 在 `mda_si_drv1` 引用的 softc 字段被拆除之前调用 `destroy_dev`。

### 用户空间安全检查清单

- 没有用户指针被直接解引用。每个跨边界访问都通过 `copyin`、`copyout`、`copyinstr` 或 `uiomove` 进行。
- 来自拷贝辅助的所有返回值都被检查。`EFAULT` 被传播而不是被忽略。
- `_IOR`、`_IOW` 和 `_IOWR` 用于 ioctl 命令号。
- Ioctl 处理程序在对参数采取行动之前验证它们。
- 当操作是特权操作时，使用 `priv_check(9)` 检查凭据。

### 诊断覆盖检查清单

- 每个不应该被采用的主要分支都带有 `KASSERT`。
- 日志使用 `device_printf` 获取实例上下文。
- 至少有一个 DTrace SDT 探针标记主 I/O 路径的入口。
- `sysctl` 在稳定的、记录的树中公开驱动程序的计数器。
- 驱动程序在被认为完成之前已在 `INVARIANTS` 和 `WITNESS` 下构建和运行。

## 结语

本附录是参考，不是章节。您使用得越多，它就越有用。在您编写、调试或阅读驱动程序代码时将其保持在身边，每当您想要快速提醒关于您几乎记得的标志、手册页或注意事项时转向它。

三个关于随时间获得最大收益的建议。

首先，将**手册页**行视为您只记得一半的任何 API 的权威下一步。第 9 节中的手册页随源代码树维护；它们老化得很好。打开其中任何一个没有任何成本，每次都有回报。

其次，将**注意事项**行视为调试伴侣。大多数驱动程序错误不是未知的未知。它们是作者在时间压力下跳过的记录注意事项。当您卡住时，阅读问题区域触及的每个 API 的注意事项。这不光鲜但有效。

第三，当您发现缺少的条目或要更正的内容时，写下来。本附录随着驱动程序的改进而改进。FreeBSD 内核是活的，参考也是活的。如果出现新原语或旧原语退休，匹配现实的附录是您真正会信任的。

从这里您可以跳向几个方向。附录 E 涵盖 FreeBSD 内部结构和子系统行为，深度是本参考故意避免的。附录 B 收集在整个内核中重复出现的算法和系统编程模式。附录 C 为总线和 DMA 系列依赖的硬件概念奠定基础。主书中的每一章仍然有您可以阅读的源代码、您可以运行的实验以及您可以通过打开 `/usr/src/` 并查看真实内容来回答的问题。

好的参考资料是安静的。它在您工作时保持距离，在您需要时它就在那里。这就是本附录意在为您的 FreeBSD 驱动程序编写生涯其余部分扮演的角色。
