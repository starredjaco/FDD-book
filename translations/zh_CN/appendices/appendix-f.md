---
title: "基准测试工具和结果"
description: "可重现的基准测试工具，包含可工作的源代码和代表性测量结果，用于第 15、28、33 和 34 章中的性能声明。"
appendix: "F"
lastUpdated: "2026-04-21"
status: "complete"
author: "Edson Brandi"
reviewer: "TBD"
translator: "AI辅助翻译为简体中文"
language: "zh-CN"
estimatedReadTime: 35
---

# 附录 F：基准测试工具和结果

## 如何使用本附录

本书中的几章提出了性能声明。第 15 章给出了 `mtx_lock`、`sx_slock`、条件变量和信号量的数量级计时。第 28 章指出，当网络驱动程序转换为 `iflib(9)` 时，驱动程序代码通常减少 30% 到 50%。第 33 章讨论了时间计数器源的成本层次，TSC 在一端，HPET 在另一端，ACPI-fast 在中间。第 34 章描述了启用 `INVARIANTS` 或 `WITNESS` 的调试内核的运行时成本，以及附加了活动 DTrace 脚本的内核的成本。这些声明中的每一个都在文本中用"在典型的 FreeBSD 14.3-amd64 硬件上"、"在我们的实验室环境中"或"数量级"等短语限定。限定词存在是因为绝对数字取决于特定机器、特定工作负载和构建内核的特定编译器。

本附录的存在是为了让这些限定词建立在可重现的基础上。对于每类声明，`examples/appendices/appendix-f-benchmarks/` 下的配套树包含一个可工作的工具，读者可以构建、运行和扩展。在工具可移植且不需要硬件访问的地方，本附录还报告它在已知机器上产生的测量结果，以便读者有具体的数字可供比较。在工具需要无法假设在任何给定读者机器上存在的特定内核配置的地方，仅提供工具本身以及重现它的清晰说明。

目的不是用权威数字替换第 15 章或第 34 章的声明。它是让读者看到声明是如何得出的，让他们在自己的硬件上验证声明，并使结果诚实地说明什么随环境变化，什么不变化。

### 本附录如何组织

本附录有五个基准测试部分，每个都有相同的内部结构。

- **测量什么。**一段话描述章节中的声明和工具测量的量。
- **工具。**配套文件的文件系统位置、使用的编程语言和方法的简短描述。
- **如何重现。**读者运行的确切命令或命令序列。
- **代表性结果。**测量值，或"仅工具，未捕获结果"，当作者未运行工具时。
- **硬件范围。**结果预期可概括的机器范围，以及已知不可概括的范围。

五个部分按顺序是时间计数器读取成本、同步原语延迟、iflib 驱动程序代码大小减少，以及 DTrace-with-INVARIANTS-and-WITNESS 开销。关于调度器唤醒延迟的最后一节指向现有的第 33 章脚本而非引入新脚本，因为那里的脚本已经是工具。

## 硬件和软件设置

在介绍基准测试之前，关于范围的一句话。本附录中的数字和其中引用的代表性结果来自两种测量。

第一种是**可移植测量**：计算源代码行数、读取确定性工具的输出或以其他方式仅依赖于 FreeBSD 源代码树和工作编译器的任何内容。这些测量在具有相同源代码检出的任何主机上产生相同结果。第 4 节中的 iflib 代码大小比较是附录中唯一的可移植测量，其结果可以在 `/usr/src` 同步到 FreeBSD 14.3-RELEASE 源代码标签的任何机器上精确重现。

第二种是**硬件相关测量**：计时内核路径、系统调用或硬件寄存器读取的任何内容。这些测量取决于 CPU、内存层次结构和内核配置。对于本附录中的每个硬件相关基准测试，都提供了工具，重现步骤是精确的，并且仅当作者实际在已知机器上运行过工具时才引用代表性结果。在作者没有运行的地方，附录明确说明并为读者留空结果表。

更公平的说法是本附录是**可运行的野外指南**。章节引用有限定数字；附录向你展示如何在面前硬件上测量相同量，以及如果你的硬件属于与章节心目中的硬件相同的大类（"现代 amd64"、"当前服役中的 Intel 和 AMD 代"）可以预期什么。

### 适用于每节的注意事项

一些注意事项适用于全篇，一次性命名比每节重复更简单。

所有硬件相关工具测量大循环上的平均值。平均值隐藏尾部行为。P99 延迟可能比同一路径上的均值高一个数量级，特别是对于涉及调度器唤醒的任何内容。任何严肃的生产性能声明都需要分布测量，而非单个数字；本附录关于均值，因为那是章节声明所指的。

在虚拟化下运行这些工具的读者应该预期比裸机明显更嘈杂的结果。例如，虚拟化 TSC 可能由管理程序以每次读取增加数百纳秒的方式合成。第 33 章成本层次在虚拟化下定性仍然成立，但绝对数字会移动。

最后，这些工具都不旨在用于生产内核。特别是同步原语 kmod 生成在紧密空操作循环中运行的内核线程；在开发机器上加载几秒钟并卸载是安全的，但不应该在繁忙的服务器上加载。

## 时间计数器读取成本

### 测量什么

第 33 章描述了 FreeBSD 中时间计数器源的三路成本层次：TSC 读取便宜，ACPI-fast 中等昂贵，HPET 昂贵。声明用"在当前服役中的 Intel 和 AMD 代"限定，并单独用"在典型的 FreeBSD 14.3-amd64 硬件上"限定。本节的工具测量一次 `clock_gettime(CLOCK_MONOTONIC)` 调用的平均成本，内核通过 `kern.timecounter.hardware` 当前选择的任何源解析它，并单独测量裸 `rdtsc` 指令作为底限。

测量的量是每次调用的纳秒，平均超过一千万次迭代。底层内核路径是 `/usr/src/sys/kern/kern_tc.c` 中的 `sbinuptime()`，它读取当前时间计数器的 `tc_get_timecount` 方法，缩放它，并将结果作为 `sbintime_t` 返回。

### 工具

工具生活在 `examples/appendices/appendix-f-benchmarks/timecounter/` 下，由三部分组成。

`tc_bench.c` 是一个小型用户空间程序，在紧密循环中调用 `clock_gettime(CLOCK_MONOTONIC)` 并报告每次调用的平均纳秒。它在启动时读取 `kern.timecounter.hardware` 并打印当前源名称，以便每次运行自文档化。

`rdtsc_bench.c` 是配套的用户空间程序，使用内核在其自己的 `/usr/src/sys/amd64/include/cpufunc.h` 中的 `rdtsc()` 包装器中使用的内联汇编模式直接读取 `rdtsc` 指令。其输出是指令本身的成本，没有任何内核开销。

`run_tc_bench.sh` 是仅限 root 的 shell 包装器，它读取 `kern.timecounter.choice`（当前内核可用源列表），遍历每个条目，将 `kern.timecounter.hardware` 设置为该源，运行 `tc_bench`，并在退出时恢复原始设置。结果是每个时间计数器源一行的表，准备好比较。

### 如何重现

构建两个用户空间程序：

```console
$ cd examples/appendices/appendix-f-benchmarks/timecounter
$ make
```

运行轮换（需要 root 来翻转 sysctl）：

```console
# sh run_tc_bench.sh
```

或仅直接 TSC 底限：

```console
$ ./rdtsc_bench
```

### 代表性结果

仅工具，未捕获结果。工具已编译并审查其逻辑，但作者在编写本附录时未在参考机器上运行它。在典型 FreeBSD 14.3-amd64 硬件上运行工具的读者应该预期 TSC 列报告低两位数纳秒的值，ACPI-fast 列报告高几倍的值，HPET 列（如果可用且未在固件中禁用）报告再高一个数量级的值。绝对数字将随 CPU 代、电源状态以及 `clock_gettime` 是由快速 gettime 路径服务还是落入完整系统调用而变化。

### 硬件范围

三个时间计数器源按成本排序自 2000 年代中期不变 TSC 成为标准以来在 amd64 代间一直稳定。不同 CPU 厂商或不同微架构代的读者将看到不同的绝对数字但相同的排序。在 ARM64 上，没有通常意义上的 HPET 或 ACPI-fast，相关比较是通用定时器计数寄存器和包装它的软件路径之间；工具仍将运行，但表中只出现一个条目。如果读者机器上的 `kern.timecounter.choice` 显示单个源，这本身是有用的数据点：系统固件已限制选择，无法轮换。

另见第 33 章获取周围上下文，特别是关于 `sbinuptime()` 的部分和关于驱动程序代码为何应避免直接读取 `rdtsc()` 的讨论。

## 同步原语延迟

### 测量什么

第 15 章展示了 FreeBSD 同步原语的近似每操作成本表：原子操作在一两纳秒，无竞争 `mtx_lock` 在几十纳秒，无竞争 `sx_slock` 和 `sx_xlock` 稍高，`cv_wait`/`sema_wait` 在微秒级因为它们总是涉及完整调度器唤醒。表中的数字被描述为"典型 FreeBSD 14.3 amd64 硬件上的数量级估计"，并附带说明它们"可能跨 CPU 代变化两倍或更多"。本节的工具直接测量表的每一行。

测量的量是：

- 无竞争互斥锁上每 `mtx_lock` / `mtx_unlock` 对的纳秒。
- 无竞争 sx 上每 `sx_slock` / `sx_sunlock` 对的纳秒。
- 无竞争 sx 上每 `sx_xlock` / `sx_xunlock` 对的纳秒。
- 两个内核线程之间一次性 `cv_signal` / `cv_wait` 往返的纳秒。
- 两个内核线程之间一次性 `sema_post` / `sema_wait` 往返的纳秒。

### 工具

工具生活在 `examples/appendices/appendix-f-benchmarks/sync/` 下，是单个可加载内核模块 `sync_bench.ko`。模块在 `debug.sync_bench.` 下暴露五个仅写 sysctl（每个基准测试一个）和五个报告每个基准测试最近结果的只读 sysctl。

每个基准测试使用 `/usr/src/sys/kern/kern_tc.c` 中的 `sbinuptime()` 获取时间戳来计时固定次数的迭代。互斥锁、sx_slock 和 sx_xlock 基准测试完全在调用线程中运行，仅练习无竞争快速路径。cv 和 sema 基准测试生成一个工作者 kproc，与主线程签署乒乓协议；因此每次迭代在每个方向包括一次唤醒和一次上下文切换，这正是第 15 章"唤醒延迟"列测量的内容。

模块使用本书其他章节中的 kmod 模式，用 `DECLARE_MODULE` 声明并通过 `SI_SUB_KLD / SI_ORDER_ANY` 初始化。源代码是 `mtx_lock` 的 `/usr/src/sys/kern/kern_mutex.c`、`sx_slock` / `sx_xlock` 的 `/usr/src/sys/kern/kern_sx.c`、条件变量的 `/usr/src/sys/kern/kern_condvar.c` 和计数信号量的 `/usr/src/sys/kern/kern_sema.c`。

### 如何重现

构建模块：

```console
$ cd examples/appendices/appendix-f-benchmarks/sync
$ make
```

加载并驱动它：

```console
# kldload ./sync_bench.ko
# sh run_sync_bench.sh
# kldunload sync_bench
```

`run_sync_bench.sh` 按顺序运行每个基准测试并打印小表；也可以通过向相应的 `debug.sync_bench.run_*` sysctl 写入 `1` 然后读取 `debug.sync_bench.last_ns_*` 直接触发单个基准测试。

### 代表性结果

仅工具，未捕获结果。kmod 已针对 FreeBSD 14.3 头文件编写，其逻辑已对照第 15 章表审查，但作者在编写本附录时未在参考机器上加载和运行它。在典型 FreeBSD 14.3-amd64 硬件上运行工具的读者应该预期无竞争互斥锁和 sx 数字落在低两位数纳秒，cv 和 sema 往返数字落在低微秒，因为这些路径两次穿过调度器。任何数字高两倍以上的读者应该查看调度器亲和性、CPU 频率缩放以及主机是否在额外负载下。

### 硬件范围

第 15 章表的"数量级"限定词是故意的。无竞争锁成本跟踪当前缓存行上一两个原子比较和交换操作的成本，它随 CPU 代和缓存拓扑变化。往返唤醒成本跟踪调度器延迟，变化更大：具有为低延迟调优的 `kern.sched.preempt_thresh` 的专用 CPU 服务器可以显示亚微秒往返，而繁忙的多租户机器可以看到几十微秒。工具报告每个基准测试的单个均值；需要分布的读者应该扩展模块以捕获分位数，或在运行内核上使用 DTrace `lockstat` 探针替代。

另见第 15 章获取概念框架和关于每个原语何时适当的指导。

## iflib 驱动程序代码大小减少

### 测量什么

第 28 章声称，在迄今已转换为 `iflib(9)` 的驱动程序上，驱动程序代码通常比等效的普通 ifnet 实现减少 30% 到 50%。与本附录中的其他基准测试不同，这不是硬件测量。它是源代码测量：现代 NIC 驱动程序在 `iflib(9)` 下需要多少行 C 源代码与其在没有它时需要多少行。

测量的量是驱动程序主源文件的行数，分为三个数字：原始 `wc -l` 总计、非空行数和非空非注释行数（对于数量级比较足够接近"代码行数"真值的近似）。

### 工具

工具生活在 `examples/appendices/appendix-f-benchmarks/iflib/` 下，是一组可移植的 shell 脚本。

`count_driver_lines.sh` 接受一个驱动程序源文件并报告所有三个数字。注释剥离是一个理解 `/* ... */`（包括多行）和 `// ... EOL` 形式的简单 `awk` 过程；它不是完整的 C 解析器，但足够准确有用。

`compare_iflib_corpus.sh` 是主驱动程序。它遍历两个精选语料库并产生比较表：

- iflib 语料库：`/usr/src/sys/dev/e1000/if_em.c`、`/usr/src/sys/dev/ixgbe/if_ix.c`、`/usr/src/sys/dev/igc/if_igc.c`、`/usr/src/sys/dev/vmware/vmxnet3/if_vmx.c`。
- 普通 ifnet 语料库：`/usr/src/sys/dev/re/if_re.c`、`/usr/src/sys/dev/bge/if_bge.c`、`/usr/src/sys/dev/fxp/if_fxp.c`。

iflib 驱动程序通过搜索 `IFDI_` 方法回调（特征性 iflib 接口点）选择；普通 ifnet 驱动程序被选择以跨越相当范围的硬件类别。两个列表都是脚本顶部的变量，读者可以编辑。

`git_conversion_delta.sh` 是供拥有带历史的完整 FreeBSD Git 克隆的读者使用的第三个脚本。它找到将命名驱动程序转换为 iflib 的提交（通过搜索接触文件并在日志中提及"iflib"的提交）并报告该提交的行数增量。转换前后的 diff 是直接测量第 28 章声明的唯一方式；跨驱动程序比较是依赖驱动程序复杂性大致相当的代理，这是强假设。

### 如何重现

在任何 FreeBSD 14.3 源代码检出上：

```console
$ cd examples/appendices/appendix-f-benchmarks/iflib
$ sh compare_iflib_corpus.sh /usr/src
```

对于前后测量，在完整 FreeBSD Git 克隆上：

```console
$ sh git_conversion_delta.sh /path/to/freebsd-src.git if_em.c
```

### 代表性结果

针对 FreeBSD 14.3-RELEASE 源代码树捕获，`compare_iflib_corpus.sh` 产生以下摘要：

```text
=== iflib ===
  if_em.c  raw=5694  nonblank=5044  code=4232
  if_ix.c  raw=5168  nonblank=4519  code=3573
  if_igc.c raw=3305  nonblank=2835  code=2305
  if_vmx.c raw=2544  nonblank=2145  code=1832
  corpus=iflib drivers=4 avg_code=2985

=== plain-ifnet ===
  if_re.c  raw=4151  nonblank=3693  code=3037
  if_bge.c raw=6839  nonblank=6055  code=4990
  if_fxp.c raw=3245  nonblank=2943  code=2228
  corpus=plain-ifnet drivers=3 avg_code=3418

=== summary ===
  iflib avg code lines:       2985
  plain-ifnet avg code lines: 3418
  delta:                      433
  reduction:                  12%
```

跨语料库减少约 12% 小于第 28 章的 30% 到 50% 声明，这正是脚本底部的警告所警告的。章节声明是每驱动程序前后数字：相同硬件转换为 iflib 从 N 行变为 (0.5-0.7) 倍 N 行。跨驱动程序比较是不同的事情：它比较具有不同功能集和不同怪癖计数的不同硬件。跨驱动程序减少设定下限（跨语料库*有一些*平均大小减少），第 28 章引用的每驱动程序减少设定上限（单个转换提交显示更大数字）。拥有 Git 克隆的读者可以使用 `git_conversion_delta.sh` 直接验证每驱动程序数字。

### 固定的每驱动程序测量

为将第 28 章的范围锚定到具体数字，我们在 2026-04-21 针对提交 `4fd3548cada3`（"ixgbe(4): Convert driver to use iflib"，由 erj@FreeBSD.org 于 2017-12-20 编写）运行了每提交比较。该提交是树中最干净的每驱动程序转换：它既未合并驱动程序也未在同一修订中更改功能。提交两侧存在的四个驱动程序特定文件（`if_ix.c`、`if_ixv.c`、`ix_txrx.c` 和 `ixgbe.h`）从 10,606 原始行（7,093 非空非注释）降至转换提交本身的 7,600 原始行（5,074 非空非注释），代码行数减少 28%。将比较限制到核心 PF 文件（`if_ix.c` 和 `ix_txrx.c`）将结果收紧到 32%，在第 28 章的 30% 到 50% 范围内。28% 数字是要带走的标题数字，较窄的 32% 数字显示多少残余是共享头文件和 VF 兄弟代码而非框架节省的驱动程序逻辑。

更大但代表性较差的数据点是更早的 em/e1000 转换提交 `efab05d61248`（"Migrate e1000 to the IFLIB framework"，2017-01-10），它在 e1000 类驱动程序代码上实现约 70% 减少：组合驱动程序源从 13,188 降至 3,920 非空非注释行。该提交将 `if_em.c`、`if_igb.c` 和 `if_lem.c` 折叠为单个基于 iflib 的 `if_em.c` 加上新的 `em_txrx.c`，因此测量减少混合了框架节省和三个相关驱动程序合并为一个，不应读作典型每驱动程序数字。综合起来，ixgbe 和 e1000 数据点从下方和上方限定第 28 章的 30% 到 50% 范围：干净的单驱动程序转换落在或略低于下边缘，驱动程序合并转换超过上边缘。

### 硬件范围

此基准测试不依赖于硬件。结果是特定修订 FreeBSD 源代码树的函数，在具有相同检出的任何机器上应该相同。使用不同 FreeBSD 分支（15-CURRENT、较旧发布版）的读者将看到不同绝对数字，因为树在演进。

另见第 28 章获取关于 `ifnet(9)` 和 `iflib(9)` 如何嵌入其中的周围讨论。

## DTrace、INVARIANTS 和 WITNESS 开销

### 测量什么

第 34 章对调试内核做出两个性能声明：

- 繁忙的 `INVARIANTS` 内核运行比发布内核大致慢 5% 到 20%，在分配密集工作负载上有时更多，这是典型 FreeBSD 14.3-amd64 硬件上的粗略数量级数字。
- `WITNESS` 在每次锁获取和释放时添加簿记；在我们的实验室环境中，在运行锁密集工作负载的繁忙内核上，开销可能接近 20%。

它还在 DTrace 上下文中提到，活动 DTrace 脚本添加与它们触发多少探针和每个探针做多少工作成比例的开销。

本节工具测量的量是完成固定工作负载的墙上时钟时间。工具不试图直接计算百分比；它提供两个定义良好的工作负载和一致的输出格式，并期望读者针对每种内核条件运行套件一次并在它们之间计算比率。

### 工具

工具生活在 `examples/appendices/appendix-f-benchmarks/dtrace/` 下，有四个部分。

`workload_syscalls.c` 是一个用户空间程序，执行包含四个便宜系统调用（`getpid`、`getuid`、`gettimeofday`、`clock_gettime`）的紧密循环的一百万次迭代。此工作负载夸大了系统调用进入和退出路径的成本，其中 `INVARIANTS` 断言和 `WITNESS` 锁跟踪最常触发。

`workload_locks.c` 是一个用户空间程序，生成四个线程并在一小旋转互斥锁集上每线程执行一千万个 `pthread_mutex_lock` / `pthread_mutex_unlock` 对。用户空间互斥锁在竞争时落入内核的 `umtx(2)` 路径，因此此工作负载练习 `WITNESS` 仪表化的锁密集竞争路径。

`dtrace_overhead.d` 是一个最小的 DTrace 脚本，在每个系统调用进入和返回时触发但不为每个探针打印任何内容。在 `workload_syscalls` 运行期间附加它测量让 DTrace 探针框架主动仪表化系统调用路径的成本。

`run_overhead_suite.sh` 运行每个工作负载一次，捕获 `uname` 和 `sysctl kern.conftxt`，并写入带标签的报告。期望读者运行套件四次：在基础 `GENERIC` 内核上、在 `INVARIANTS` 内核上、在 `WITNESS` 内核上，以及在其中任何一个上并在另一个终端中附加 `dtrace_overhead.d`。比较四个报告给出第 34 章声明的百分比。

周围内核源代码是 WITNESS 的 `/usr/src/sys/kern/subr_witness.c`、`INVARIANTS` 启用的断言宏的 `/usr/src/sys/sys/proc.h` 及其兄弟，以及探针框架的 `/usr/src/sys/cddl/dev/` 下的 DTrace 提供者源代码。

### 如何重现

针对每种内核条件：

1. 引导到测试中的内核。
2. 构建工作负载：

   ```console
   $ cd examples/appendices/appendix-f-benchmarks/dtrace
   $ make
   ```

3. 运行套件：

   ```console
   # sh run_overhead_suite.sh > result-<label>.txt
   ```

4. 对于 DTrace 条件，在运行套件前在另一个终端启动脚本：

   ```console
   # dtrace -q -s dtrace_overhead.d
   ```

四次运行后，并排比较 `result-*.txt` 文件。比率 `INVARIANTS_ns / base_ns`、`WITNESS_ns / base_ns` 和 `dtrace_ns / base_ns` 是第 34 章所指的开销数字。

### 代表性结果

仅工具，未捕获结果。工作负载已在参考系统上编译并审查其逻辑与第 34 章声明对照，但作者在编写本附录时未构建三个比较内核来捕获端到端数字。在典型 FreeBSD 14.3-amd64 硬件上运行四内核比较的读者应该预期 `INVARIANTS` 在 `workload_syscalls` 上落在 5% 到 20% 范围内，在 `workload_locks` 上稍高，因为锁路径是 `INVARIANTS` 下最热的代码。`WITNESS` 应该落在纯系统调用工作负载上低于 `INVARIANTS`（其中很少访问新锁顺序），在 `workload_locks` 上更高，接近章节命名的 20% 数字。DTrace 列应该在 `workload_locks` 上很小（热路径中未触发相关探针），在 `workload_syscalls` 上非平凡（每次迭代触发两个探针）。

### 硬件范围

比率跨硬件比绝对数字更稳定。不同 CPU 上的读者将看到基线工作负载的不同墙上时间，但每内核百分比开销应该保持在紧密范围内，因为它由调试内核每次操作做多少额外工作决定，而非每次操作有多快。惊喜的最常见来源是虚拟化（其中系统调用路径有稀释百分比的额外每调用管理程序开销）、激进电源管理（其中 CPU 频率跨运行变化）和 SMT（其中每核数字不同于每逻辑 CPU 数字）。工具不试图控制其中任何一个；想要生产质量数字的读者需要将测试固定到单个 CPU、禁用频率缩放并多次运行套件。

另见第 34 章获取调试内核的周围处理以及关于何时值得启用 `INVARIANTS`、`WITNESS` 和 DTrace 中每一个的讨论。

## 调度器唤醒延迟

第 33 章包含一个使用 `sched:::wakeup` 和 `sched:::on-cpu` 测量调度器唤醒延迟的 DTrace 片段。该片段已经是工具；本节只会重复它。对第 33 章命名的亚微秒空闲系统数字和低两位数微秒竞争系统数字感兴趣的读者应该完全按照那里印刷的方式在读者硬件上使用脚本。DTrace 提供者源代码在 `/usr/src/sys/cddl/dev/dtrace/`。

如果读者想比较有和没有 `WITNESS` 的唤醒延迟，应用相同脚本；只需引导两个内核并在每个上运行它。

## 总结：使用工具而不被误导

本附录中的工具存在是因为第 15、28、33 和 34 章中的每个数字都是数量级声明，数量级声明值得可重现性。几个习惯使工具实际上有用而非虚假信心的来源。

首先，多次运行每个基准测试。单次运行结果由启动噪声、预热效应和机器上运行的其他任何东西的干扰主导。工具脚本报告单个数字；运行三到五次并取中位数，或在信任结果前扩展它们以自动聚合运行。

其次，保持条件诚实。如果想比较 `INVARIANTS` 与基础内核，用相同编译器、相同 `CFLAGS` 和相同 `GENERIC` 或 `GENERIC-NODEBUG` 基线构建两个内核。如果想比较两个时间计数器源，在相同内核的相同引导上运行两个比较；运行间重新引导改变太多变量。

第三，抵制将数字视为通用的诱惑。在安静办公室中 4 核笔记本电脑上的测量不是嘈杂机架中 64 核生产服务器上的测量。工具测量面前机器上的内容；第 15 到 34 章的限定短语（"在典型 14.3-amd64 硬件上"、"在我们的实验室环境中"）对相同限制是诚实的。工具使限定短语可测试而不假装它变成通用的。

第四，优先比率而非绝对值。`WITNESS` 代价 20% 的声明比 `WITNESS` 每锁代价 400 纳秒的声明稳定得多。前者存活于会消灭后者的 CPU 更改、编译器更改和内核版本更改。当你在硬件上运行工具时，你应该保留百分比开销；绝对数字只是产生它的算术。

最后，扩展工具而非盲目信任它。这里的每个脚本都小到可以端到端阅读；每个 kmod 都在三百行以下。如果章节声明对你重要，打开工具，验证它测量你期望它测量的内容，如果工作负载需要不同东西则修改它，并运行修改版本。工具是起点，不是终点线。

第 15、28、33 和 34 章中的限定短语和本附录中的可运行工具是同一规则的两半。章节对什么是变化的诚实；附录向你展示如何自己测量变化。
