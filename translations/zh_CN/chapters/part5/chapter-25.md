---
title: "高级主题与实用技巧"
description: "第25章作为第5部分的收官之作，教授将一个可工作的、已集成的FreeBSD驱动程序转变为健壮、可维护的内核软件所需的工程习惯。内容涵盖：限速内核日志与日志礼仪；用于read、write、ioctl、sysctl和生命周期回调的规范化errno值与返回约定；通过/boot/loader.conf可调参数和可写sysctl进行驱动配置；ioctl、sysctl和用户可见行为的版本控制与兼容性策略；使用标签化goto清理模式处理失败路径中的资源管理；将驱动模块化为逻辑分离的源文件；使用MODULE_DEPEND、MODULE_PNP_INFO和合理的打包为生产使用准备驱动；以及扩展驱动生命周期至简单MOD_LOAD和MOD_UNLOAD之外的SYSINIT/SYSUNINIT/EVENTHANDLER机制。myfirst驱动从1.7-integration版本升级到1.8-maintenance版本：新增myfirst_log.c和myfirst_log.h，包含基于ppsratecheck的DLOG_RL宏；将myfirst_cdev.c和myfirst_bus.c分离，使cdev回调与Newbus attach机制分开；添加MAINTENANCE.md文档；添加shutdown_pre_sync事件处理器；添加MYFIRSTIOC_GETCAPS ioctl让用户空间协商特性位；以及版本升级的回归测试脚本。本章完成后，第5部分即告结束：驱动程序仍可理解，无需重新编译即可调优，现在可以承受未来一年的维护而不会变得难以阅读。"
partNumber: 5
partName: "调试、工具与实践"
chapter: 25
lastUpdated: "2026-04-19"
status: "complete"
author: "Edson Brandi"
reviewer: "TBD"
translator: "AI辅助翻译为简体中文"
estimatedReadTime: 225
language: "zh-CN"
---

# 高级主题与实用技巧

## 读者指南与学习目标

第24章以一个系统其余部分可以与之通信的驱动程序结束。`myfirst`驱动在`1.7-integration`版本时拥有一个通过`make_dev_s`创建的干净的`/dev/myfirst0`节点、一个在内核与用户空间之间共享的公共ioctl头文件、位于`dev.myfirst.0`下的每实例sysctl子树、用于调试掩码的启动时可调参数，以及一个遵守`ENOIOCTL`回退规则的ioctl分发器，以便`FIONBIO`等内核辅助功能仍能正确到达cdev层。驱动程序可以编译、加载、在压力下运行，并在重复的`kldload`和`kldunload`循环中存活，而不会泄漏OID或cdev节点。在任何可观察的意义上，驱动程序都能正常工作。

第25章讨论的是一个*能工作*的驱动程序与一个*可维护*的驱动程序之间的区别。这两个品质并不相同，差异会慢慢显现。一个能工作的驱动程序通过第一轮测试，干净地附加到其硬件，然后投入使用。一个可维护的驱动程序也能做到这些，然后在接下来的一年里吸收错误修复、功能添加、可移植性更改、新硬件修订版和内核API变更，而不会在自身重压下慢慢崩溃。前者给开发者带来美好的一天，后者给驱动程序带来美好的十年。

第25章是第5部分的收官章节。第23章教授可观测性，第24章教授集成，第25章教授能够长期保持这两种品质的工程习惯。第6部分紧接着开始介绍特定传输介质的驱动程序（第26章USB，第27章存储，第28章网络及之后），这些章节都将假设本章介绍的规范。如果没有限速日志，USB热插拔风暴会填满消息缓冲区。如果没有一致的错误约定，存储驱动程序与其在CAM中的外设会对`EBUSY`的含义产生分歧。如果没有加载器可调参数，一个具有次优默认队列深度的网络驱动程序在生产机器上无法在不重新编译的情况下进行调优。如果没有版本控制规范，两个月前添加的新字段会被为此版本驱动程序编写的用户空间工具静默误解。每个习惯都很小，但它们共同使驱动程序成为FreeBSD的长久组成部分，而不是短命的实验室实验。

本章的运行示例仍然是`myfirst`驱动程序。在本章开始时，它处于`1.7-integration`版本。在本章结束时，它将达到`1.8-maintenance`版本，被拆分为更多文件，记录日志时不会淹没消息缓冲区，从一致的词汇表返回错误，可从`/boot/loader.conf`配置，附带解释持续维护契约的`MAINTENANCE.md`文档，通过`devctl`通道发布事件，并通过`EVENTHANDLER(9)`挂钩到内核的关机和低内存事件。这些新增内容都不需要新的硬件知识，但都需要更严谨的规范。

第5部分在此以保持驱动程序在成长过程中保持一致性的习惯作结。第22章使驱动程序能够承受电源状态变化。第23章使驱动程序能够告诉你它在做什么。第24章使驱动程序能够融入系统其余部分。第25章使驱动程序在演进过程中保持所有这些品质。第26章将开启第6部分，将这些品质应用于真实传输介质——通用串行总线，日志或失败路径处理中的每一个捷径都会被USB流量的速度和多样性所暴露。

### 为什么维护规范值得专门设一章

在继续之前，值得停下来思考一下限速日志、errno词汇表和加载器可调参数是否真的值得一整章。前面的章节已经教授了那么多。添加一个`ppsratecheck(9)`支持的日志宏看起来很小。规范化错误代码看起来更小。既然每个习惯看起来只有几行代码，为什么要用一整章来展开呢？

答案是每个习惯都很小，但缺少每个习惯的代价很大。一个不限速记录日志的驱动程序在实验室里很好，但在生产环境中，当一根松动的电缆每秒触发一万次重新枚举时就是灾难。一个在应该返回`ENXIO`时返回`EINVAL`、在应该返回`ENOIOCTL`时返回`ENXIO`的驱动程序，当作者作为唯一调用者时很好，但当第二个开发者编写第一个用户空间辅助程序时就是一个等待发生的错误报告。一个将每个配置默认值设为编译时常量的驱动程序，对于一个人来说很好，但对于一个在具有不同工作负载的多台生产机器上维护同一模块的团队来说则不可行。第25章花时间讨论这些习惯，因为价值不是在实验室中衡量的，而是在每个习惯所减少的两年维护成本中衡量的。

本章有资格设立的第一原因是**这些习惯塑造了驱动程序代码库随时间增长的样子**。跟随第23章和第24章的读者已经看到驱动程序拆分为多个文件：`myfirst.c`、`myfirst_debug.c`、`myfirst_ioctl.c`、`myfirst_sysctl.c`。那是小规模的模块化，一次一个接口。第25章以大多数读者尚未提出的问题重新审视模块化：*一旦驱动程序有十几个文件和三个开发者，可维护的源代码布局是什么样的？*本章以Newbus附加层、cdev层、ioctl层、sysctl层和日志层之间有意识的分离来回答这个问题，然后利用这种分离来支持本章教授的其他习惯。

第二个原因是**这些习惯决定了驱动程序是否可以在生产环境中调试**。一个审慎记录日志并返回信息性错误的驱动程序，给操作员足够的信息来提交有用的错误报告。一个记录太多或太少、或发明自己的errno约定的驱动程序，迫使操作员仅从症状推断，开发者最终会盲目地追踪间歇性问题。第23章的调试工具包很有效，但它依赖于驱动程序的配合。这种配合在这里建立。

第三个原因是**这些习惯使驱动程序在不破坏调用者的情况下可扩展**。第24章的`myfirst_ioctl.h`头文件已经是驱动程序与用户空间之间的契约。第25章教授读者如何演进该契约，添加旧用户空间程序可以安全忽略的新ioctl，在不破坏管理员脚本的情况下弃用过时的sysctl，并以外部消费者可以在运行时检查的方式升级驱动程序的版本。没有这些习惯，驱动程序的第一个v2版本会迫使每个调用者重写。有了它们，驱动程序可以添加功能长达十年，同时仍然运行驱动程序首次发布第一周编译的用户空间辅助程序。

第25章通过将这三个思想结合在一起，以`myfirst`驱动程序作为运行示例来具体教授，从而赢得了它的位置。完成第25章的读者可以使任何FreeBSD驱动程序为长期维护做好准备，可以阅读其他驱动程序的生产强化模式并识别哪些是原则性的哪些是临时的，可以与现有用户空间工具协商兼容性，并拥有一个明显准备好开始第6部分的`1.8-maintenance`版本的`myfirst`驱动程序。

### 第24章为驱动程序留下的状态

简要回顾您应该达到的位置。第25章扩展了第24章第3阶段结束时生成的驱动程序，标记为`1.7-integration`版本。如果以下任何一项不确定，请返回第24章并在开始本章之前修复它，因为新材料假设每个第24章原语都在工作。

- 您的驱动程序干净地编译，并在`kldstat -v`的输出中标识自己为`1.7-integration`。
- `kldload`后存在`/dev/myfirst0`节点，具有`root:wheel`所有权和`0660`模式，并在`kldunload`时干净地消失。
- 模块导出四个ioctl：`MYFIRSTIOC_GETVER`、`MYFIRSTIOC_GETMSG`、`MYFIRSTIOC_SETMSG`和`MYFIRSTIOC_RESET`。第24章中的小型`myfirstctl`用户空间程序执行每一个并全部返回成功。
- sysctl子树`dev.myfirst.0`至少列出`version`、`open_count`、`total_reads`、`total_writes`、`message`、`message_len`、`debug.mask`和`debug.classes`。
- `sysctl dev.myfirst.0.debug.mask=0xff`启用每个调试类，驱动程序随后的日志输出显示预期的标签。
- 放置在`/boot/loader.conf`中的启动时可调参数`hw.myfirst.debug_mask_default`在附加之前应用并设置sysctl的初始值。
- 在循环中重复`kldload`和`kldunload`一分钟不会留下残留的OID、孤立的cdev或泄漏的内存（由`vmstat -m | grep myfirst`报告）。
- 您的工作树包含来自前面章节的`HARDWARE.md`、`LOCKING.md`、`SIMULATION.md`、`PCI.md`、`INTERRUPTS.md`、`MSIX.md`、`DMA.md`、`POWER.md`、`DEBUG.md`和`INTEGRATION.md`。
- 您的测试内核启用了`INVARIANTS`、`WITNESS`、`WITNESS_SKIPSPIN`、`DDB`、`KDB`、`KDB_UNATTENDED`、`KDTRACE_HOOKS`和`DDB_CTF`。第25章的实验室像第24章一样强烈依赖`WITNESS`和`INVARIANTS`。

那就是第25章扩展的驱动程序。新增内容在代码行数上比第5部分的任何前一章都少，但在概念面上更大。新部分包括：围绕`ppsratecheck(9)`构建的`myfirst_log.c`和`myfirst_log.h`对、`myfirst_attach`中的标签化goto清理链、整个分发器中精炼的错误词汇表、用于驱动程序范围初始化的一对`SYSINIT`/`SYSUNINIT`挂钩、`shutdown_pre_sync`事件处理器、让用户空间查询特性位的新`MYFIRSTIOC_GETCAPS` ioctl、将Newbus附加从`myfirst.c`分离到`myfirst_bus.c`并将cdev回调分离到`myfirst_cdev.c`的适度重构、解释版本升级策略的`MAINTENANCE.md`文档、更新的回归测试脚本，以及升级到`1.8-maintenance`。

### 您将学到什么

在本章结束时，您将能够：

- 解释为什么不受限制的内核日志是生产环境的隐患，描述`ppsratecheck(9)`如何限制每秒事件数，编写与第23章调试掩码配合的限速日志宏，并识别三类需要不同节流策略的日志消息。
- 审计驱动程序的`read`、`write`、`ioctl`、`open`、`close`、`attach`、`detach`和sysctl处理程序路径的正确errno使用。区分`EINVAL`与`ENXIO`、`ENOIOCTL`与`ENOTTY`、`EBUSY`与`EAGAIN`、`EPERM`与`EACCES`、`EIO`与`EFAULT`，并知道何时使用每个是正确的返回值。
- 通过`TUNABLE_INT_FETCH`、`TUNABLE_LONG_FETCH`、`TUNABLE_BOOL_FETCH`和`TUNABLE_STR_FETCH`添加加载器可调参数，并将它们与可写sysctl结合，以便一个旋钮可以在启动时设置或在运行时调整。理解`CTLFLAG_TUN`如何与可调参数获取器配合。
- 将配置暴露为一个小的、有文档记录的接口，而不是一堆临时的环境变量。有纪律地在每驱动程序和每实例可调参数之间做出选择。记录每个可调参数的单位、范围和默认值。
- 使用稳定的三重拆分为驱动程序的用户可见接口建立版本：`dev.myfirst.0.version`中的人类可读发布字符串、内核模块依赖机制使用的整数`MODULE_VERSION`，以及嵌入公共头文件的线格式整数`MYFIRST_IOCTL_VERSION`。
- 在不破坏旧调用者的情况下向现有公共头文件添加新ioctl，以正确的弃用期弃用过时的ioctl，并通过`MYFIRSTIOC_GETCAPS`提供能力位掩码，以便用户空间程序无需试错即可检测特性可用性。
- 使用`goto fail;`模式构建驱动程序的附加和分离路径，以便每个分配恰好有一个清理点，每个清理按分配的相反顺序运行，部分附加永远不会留下分离路径不会释放的资源。
- 沿责任线而不是文件大小线将驱动程序拆分为逻辑源文件。在单个大文件、小型主题聚焦文件集合和完整子系统树之间做出选择，并知道每种何时合适。
- 使用`MODULE_DEPEND`、`MODULE_PNP_INFO`、接受`MOD_QUIESCE`的行为良好的`modevent`处理器（当驱动程序可以干净地暂停时）、同时安装模块及其文档的小型构建系统，以及通过`devctl_notify`发布驱动程序事件的`devd(8)`就绪模式，为生产使用准备驱动程序。
- 使用`SYSINIT(9)`和`SYSUNINIT(9)`在特定内核子系统阶段挂钩驱动程序范围的设置和拆卸，并理解模块事件处理器与子系统级初始化挂钩之间的区别。
- 通过`EVENTHANDLER(9)`在知名内核事件上注册和注销回调：`shutdown_pre_sync`、`shutdown_post_sync`、`shutdown_final`、`vm_lowmem`、`power_suspend_early`和`power_resume`。知道如何选择优先级以及如何保证在分离时注销。

列表很长，因为维护规范同时触及许多小的接口。每一项都是狭窄且可教授的。本章的工作是让它们成为习惯。

### 本章不涵盖的内容

几个相关主题被明确推迟，以便第25章保持在适合完成第5部分的读者的维护规范的正确水平上。

- **特定传输介质的生产模式**，如USB热插拔风暴、SATA链路状态事件和以太网介质变化处理，属于第6部分，那里每个传输介质都会被完整教授。第25章教授*通用*习惯；第26章及之后将其专门应用于USB。
- **完整的测试框架设计**，包括跨多个内核配置和故障注入场景运行的回归测试工具，属于第26、27和28章的硬件无关测试部分。第25章只是向现有回归脚本添加一行；它不引入整个工具。
- **`fail(9)`和`fail_point(9)`**，内核的错误注入设施，推迟到第28章，与最常使用它们的存储驱动程序工作一起介绍。
- **持续集成、包签名和分发**是发布驱动程序的项目的运维关注点，而不是驱动程序源代码本身的关注点。本章只说足够的关于打包的内容，使驱动程序可重现。
- **`MAC(9)`（强制访问控制）挂钩**是一个专门的框架，最好在后面的安全专题章节中介绍。
- **`kbi(9)`稳定性和ABI冻结**是FreeBSD项目做出的发布工程决策，而不是驱动程序作者做出的。本章注意内核导出函数的ABI含义，但不深入涵盖发布工程。
- **`capsicum(4)`能力模式集成**用于用户空间辅助程序是用户空间安全的主题，而不是驱动程序本身的主题。本章的`myfirstctl`仍然是传统的UNIX工具。
- **高级并发模式**，如`epoch(9)`、读多写少锁和无锁队列。这些只是顺便提及；驱动程序的单个softc互斥锁在这个阶段仍然足够。

保持在这些界限内使第25章成为关于*维护规范*的章节，而不是关于高级内核开发者可能在高级内核问题上使用的每种技术的章节。

### 预计时间投入

- **仅阅读**：三到四个小时。第25章的概念比第24章的轻，大部分词汇现在已熟悉。本章的工作是将熟悉的原语转化为规范。
- **阅读加输入示例**：在两到三次会话中花费八到十小时。驱动程序通过四个短阶段演进（限速日志、错误审计、可调参数和版本规范、SYSINIT和EVENTHANDLER），每个都比单个第24章阶段小。第6节的重构涉及多个文件，但更改的代码很少；大部分工作是将现有代码移动到新位置。
- **阅读加所有实验室和挑战**：在三到四次会话中花费十二到十五小时。实验室包括日志泛滥重现和修复、使用`truss`的errno审计、用不同`/boot/loader.conf`值启动两次VM的可调参数实验室、故意附加失败实验室（锻炼`goto fail;`链中的每个标签）、确认回调确实在正确时刻运行的`shutdown_pre_sync`实验室，以及将所有内容联系在一起的回归脚本演练。

第5节（失败路径管理）是新增规范而非新词汇中最密集的。`goto fail;`模式本身是机械的；技巧在于阅读真正的FreeBSD附加函数并将每个分配视为新标签的候选。如果模式在第一次通过时感觉机械，那就是它已成为习惯的信号。

### 先决条件

在开始本章之前，确认：

- 您的驱动程序源代码与第24章第3阶段（`1.7-integration`）匹配。假设每个第24章原语都在工作：基于`make_dev_s`的cdev创建、`myfirst_ioctl.c`分发器、`myfirst_sysctl.c`树构建、`MYFIRST_VERSION`、`MODULE_VERSION`和`MYFIRST_IOCTL_VERSION`三重，以及每设备`sysctl_ctx_free`释放模式。
- 您的实验机器运行FreeBSD 14.3，磁盘上有`/usr/src`并与运行的内核匹配。
- 已构建、安装并正常启动启用了`INVARIANTS`、`WITNESS`、`WITNESS_SKIPSPIN`、`DDB`、`KDB`、`KDB_UNATTENDED`、`KDTRACE_HOOKS`和`DDB_CTF`的调试内核。
- `1.7-integration`状态的快照保存在您的VM中。第25章的实验室包括故意附加失败场景，快照使恢复成本很低。
- 以下用户空间命令在您的路径中：`dmesg`、`sysctl`、`kldstat`、`kldload`、`kldunload`、`devctl`、`devd`、`cc`、`make`、`dtrace`、`truss`、`ktrace`、`kdump`和`procstat`。
- 您习惯于编辑`/boot/loader.conf`并重新启动VM以获取新的可调参数。
- 您已构建第24章的`myfirstctl`配套程序并且它可以工作。

如果以上任何一项不稳定，现在就修复它。在一个已经遵守前面章节规则的驱动程序上学习维护规范，比在一个仍有早期阶段未解决问题的驱动程序上学习要容易。

### 如何从本章获得最大收益

五个习惯在第5部分的任何前一章中都能获得更大回报。

首先，在浏览器标签页或终端窗格中保持四个简短的手册页文件：`ppsratecheck(9)`、`style(9)`、`sysctl(9)`和`module(9)`。第一个是限速检查API的规范文档。第二个是FreeBSD的编码风格。第三个解释sysctl框架。第四个是模块事件处理器契约。它们都不长；每章开始时浏览一次值得，当正文说"详情请参阅手册页"时再回头查阅。

其次，保持三个真正的驱动程序在手边。`/usr/src/sys/dev/mmc/mmcsd.c`展示了在生产中使用`ppsratecheck`节流设备printf。`/usr/src/sys/dev/virtio/block/virtio_blk.c`展示了其附加路径中干净的`goto fail;`链和生产质量的可调参数集。`/usr/src/sys/dev/e1000/em_txrx.c`展示了复杂驱动程序如何将日志、可调参数和分发拆分到多个文件中。第25章在适当的时刻指向每一个；现在阅读一次为本章其余部分提供具体的锚点。

> **关于行号的说明。**当本章指向`mmcsd.c`、`virtio_blk.c`或`em_txrx.c`中的特定位置时，指针是一个命名符号，而不是数字行。`ppsratecheck`、`virtio_blk_attach`中的`goto fail;`标签和`TUNABLE_*_FETCH`调用在未来树修订中仍然可以通过这些名称找到，即使它们周围的行在移动。您稍后将在本章中看到的审计示例使用`file:line`表示法纯粹作为示例工具输出，并带有相同的健康警告。

第三，手动将每个更改输入到`myfirst`驱动程序中。第25章中的添加是开发者在一年维护工作后会反射性地进行的更改类型。现在输入它们建立反射；粘贴它们跳过了课程。

第四，在第3节的可调参数材料之后，至少用新的`/boot/loader.conf`设置重启一次VM，并观察驱动程序在附加期间获取它。可调参数是那些在看到真实值从引导加载程序通过内核流入您的softc之前感觉很抽象的功能之一。两次重启和一个`sysctl`命令就是全部所需。

第五，当关于`goto fail;`的章节要求您在`myfirst_attach`中引入故意失败时，实际去做。在附加中间注入单个`return (ENOMEM);`并观察清理链正确展开是将模式内化的最佳方式。本章建议一个特定的注入位置，回归脚本确认清理确实运行了。

### 章节路线图

各节按顺序为：

1. **限速与日志礼仪。**为什么不受控制的内核日志是生产环境的隐患；驱动程序日志消息的三类（生命周期、错误、调试）；`ppsratecheck(9)`和`ratecheck(9)`作为FreeBSD对日志泛滥的应对；与第23章调试掩码配合的限速`DLOG_RL`宏；`log(9)`优先级级别及其与`device_printf`和`printf`的关系；内核消息缓冲区实际消耗什么以及如何不随意花费它。
2. **错误报告与返回约定。**为什么errno规范是与每个调用者的契约；驱动程序例行使用的小型内核errno词汇表；何时使用每个是合适的；`ENOIOCTL`与`ENOTTY`以及为什么驱动程序绝不能从ioctl默认值返回`EINVAL`；sysctl处理程序返回代码；模块事件处理程序返回代码；读者从现在开始可以应用于每个驱动程序的检查清单。
3. **通过加载器可调参数和sysctl进行驱动配置。**`/boot/loader.conf`可调参数与运行时sysctl的区别；`TUNABLE_*_FETCH`系列和`CTLFLAG_TUN`标志；每驱动程序与每实例可调参数；如何记录可调参数以便操作员可以信任它；一个工作的实验室，用可调参数在三个不同位置启动VM并观察驱动程序sysctl树中的效果。
4. **版本控制与兼容性策略。**三重版本拆分（`MODULE_VERSION`整数、`MYFIRST_VERSION`人类字符串、`MYFIRST_IOCTL_VERSION`线格式整数）；每一个如何使用；如何在不破坏旧调用者的情况下添加新ioctl；如何弃用过时的ioctl；`MYFIRSTIOC_GETCAPS`和能力位掩码思想；驱动程序如何在没有专用内核标志的情况下优雅地弃用sysctl OID；`MODULE_DEPEND`如何强制依赖模块的最低版本。
5. **管理失败路径中的资源。**`myfirst_attach`中的失败时清理问题；`goto fail;`模式以及为什么线性展开胜过嵌套`if`链；标签命名约定（`fail_mtx`、`fail_cdev`、`fail_sysctl`）；常见错误（成功后穿透、缺少标签、添加资源而不添加其清理）；减少重复的小型辅助函数规范；测试整个链的故意失败实验室。
6. **模块化与关注点分离。**沿责任轴将驱动程序拆分为文件；字符驱动程序的规范拆分（`myfirst.c`、`myfirst_bus.c`、`myfirst_cdev.c`、`myfirst_ioctl.c`、`myfirst_sysctl.c`、`myfirst_debug.c`、`myfirst_log.c`）；公共与私有头文件；如何组织`Makefile`使所有这些文件构建为一个`.ko`；模块化何时有帮助以及何时成为障碍；开发团队如何使用拆分来减少合并冲突。
7. **为生产使用做准备。**`MODULE_DEPEND`和依赖强制执行；用于自动加载的`MODULE_PNP_INFO`；`MOD_QUIESCE`和暂停后卸载契约；安装模块及其文档的构建系统模式；响应驱动程序事件的`devd(8)`规则；书面说明驱动程序维护契约的小型`MAINTENANCE.md`文档。
8. **SYSINIT、SYSUNINIT和EVENTHANDLER。**超越`MOD_LOAD`和`MOD_UNLOAD`的内核更广泛生命周期机制；带有子系统ID和顺序常量的`SYSINIT(9)`和`SYSUNINIT(9)`；每个的真实FreeBSD示例；用于跨切面通知的`EVENTHANDLER(9)`（`shutdown_pre_sync`、`vm_lowmem`、`power_resume`）；如何干净地注册和注销；驱动程序如何使用所有三种机制而不陷入过度工程。

八个部分之后是一组动手实验室，锻炼每个规范，一组挑战练习在不引入新基础的情况下扩展读者，针对大多数读者会遇到的症状的故障排除参考，总结第25章故事并开启第26章的总结，通往下一章的桥梁，快速参考卡和词汇表。

如果是第一次阅读，请线性阅读并按顺序进行实验室。如果是复习，第1节和第5节独立存在，适合单次阅读。第8节是本章结尾的简短概念性总结；它轻依赖于第7节的生产使用材料，很容易留到第二次阅读。

在技术工作开始前的一个小说明。第25章是第5部分的最后一章。其添加比第24章的小，但它们触及驱动程序中几乎每个文件。预计花更多时间重读自己早期的代码而不是编写新代码。这也是维护规范。耐心重读的驱动程序是您可以自信更改的驱动程序；自信更改的驱动程序是您可以保持活力的驱动程序。

## 第1节：限速与日志礼仪

本章教授的第一个规范是不说太多话的规范。第24章结束时的`myfirst`驱动程序在附加、分离、客户端打开或关闭设备、读取或写入跨越边界、分发ioctl以及调整调试掩码时记录日志。每一行日志都是在有充分理由的情况下引入的，当单个事件发生时每一行都很有用。第24章的日志行都没有考虑到的是，当同一事件每秒触发十万次时会发生什么。

本节解释为什么这个问题比看起来更重要，介绍在压力下表现不同的三类驱动程序日志消息，教授FreeBSD限速检查原语（`ratecheck(9)`和`ppsratecheck(9)`），并展示如何在它们之上构建一个与第23章现有调试掩码机制配合的小型、规范的宏。在本节结束时，`myfirst`驱动程序拥有`myfirst_log.c`和`myfirst_log.h`对，其消息缓冲区在压力下不再变成噪音。

### 不受限制日志的问题

内核消息写入便宜，携带昂贵。`device_printf(dev, "something happened\n")`是单个函数调用，在现代CPU上几十纳秒，它几乎立即返回。成本不在调用中；成本在于之后发生在字节上的一切。格式化的字符串被复制到内核消息缓冲区，这是一个在启动时固定大小的内核内存圆形区域。如果控制台已连接（在VM中通常是串口，具有有限的比特率），它将被传递到控制台设备。如果驱动程序使用该路径，它通过`log(9)`路径发送到用户空间运行的syslog守护进程，然后通过`newsyslog(8)`发送到磁盘上的`/var/log/messages`。每一步都有成本，每一步在驱动程序写入行时都是同步的。

当驱动程序写入一行时，这些都不重要。当驱动程序在一秒钟内写入一百万行时，一切都重要。内核消息缓冲区填满，最旧的消息在任何人读取之前就被覆盖。通常以115200波特运行的控制台跟不上，无法追赶，这反过来将压力反馈到写入行的内核路径，即您的驱动程序的快速路径。syslog守护进程每秒唤醒、工作并睡眠多次，从其他进程窃取周期。`/var/log/messages`所在的磁盘以可预测的速率填满，一个每秒记录一万行的驱动程序可以在一个下午填满一个合理大小的分区。

这些症状都不是由驱动程序逻辑中的错误引起的。它们是由驱动程序的*日志量*引起的，而日志量又是由驱动程序在每个事件上触发合理的日志行引起的。合理的日志行只要事件罕见就没问题。当事件变得常见时，它们就成为隐患。日志礼仪的整个技艺在于，在您写入日志行的时刻，知道其背后的事件是罕见还是常见，并编写代码以便不受控制的重复不会将罕见事件日志变成常见事件日志。

来自真实驱动程序的具体示例说明了这一点。考虑一个通知其驱动程序可恢复队列满条件的PCIe SSD控制器。在健康系统上，这种情况足够罕见，记录每次发生都是有用的。在病态系统上，它可能每秒发生数百次，直到有人更换硬件。如果驱动程序每次都写入一行，消息缓冲区将填满几乎相同的行，该次启动的所有早期消息都将被覆盖并丢失，试图通过阅读`dmesg`诊断问题的操作员只能看到泛滥的最后一页。硬件的实际行为被驱动程序对其的反应所掩盖。限速日志行会显示前几次发生、速率，然后是定期提醒；`dmesg`中较早的上下文会存活；操作员会有一些东西可以处理。

这个教训可以推广。正确的日志规范不是"少记录"，也不是"多记录"，而是"以无论底层事件触发频率如何都保持有用的速率记录"。本节其余部分具体教授该规范。

### 驱动程序日志消息的三个类别

在选择正确的节流策略之前，命名驱动程序通常发出的三类日志消息会有所帮助。每个类别都有不同的节流故事。

第一类是**生命周期事件**。这些是标记附加、分离、挂起、恢复、模块加载和模块卸载的消息。它们每个生命周期转换发生一次，通常在模块生命周期内几次。不需要节流；量自然很低。限速生命周期消息将是一个错误，因为它会隐藏重要的状态转换。

第二类是**错误和警告消息**。这些是报告驱动程序认为有问题的事情的消息。按照构造，这些应该是罕见的；如果警告每秒触发一百次，警告正在告诉您关于底层事件速率的信息，该信息即使事件重复也值得保留。错误和警告消息从限速中受益匪浅，但速率限制应该至少在每次突发中保留一条消息，并使*速率*本身可见。

第三类是**调试和跟踪消息**。这些是第23章`DPRINTF`宏下的消息。当调试掩码打开时它们故意冗长，当掩码关闭时它们静默。在发出点节流它们会增加已经低信噪比路径的噪音；更好的规范是在掩码关闭时避免发出它们，这已经是现有`DPRINTF`所做的。调试和跟踪消息不需要额外的限速，但它们需要用户能够用单个`sysctl`命令完全关闭它们。现有的第23章管道已经提供了这一点。

命名了三个类别后，本节其余部分专注于第二个类别。生命周期消息按原样就好。调试消息由现有掩码处理。错误和警告消息是真正需要规范的地方。

### 介绍`ratecheck`和`ppsratecheck`

FreeBSD的内核提供两个密切相关的原语用于限速输出。两者都位于`/usr/src/sys/kern/kern_time.c`，并在`/usr/src/sys/sys/time.h`中声明。

`ratecheck(struct timeval *lasttime, const struct timeval *mininterval)`是两者中较简单的一个。调用者持有一个`struct timeval`记住事件上次触发的时间，以及允许打印之间的最小间隔。每次调用时，`ratecheck`将当前时间与`*lasttime`比较，如果经过了`mininterval`，它更新`*lasttime`并返回1。否则返回0。调用代码只在返回值为1时打印。结果是对打印速率的简单下限：每个`mininterval`最多一次打印。

`ppsratecheck(struct timeval *lasttime, int *curpps, int maxpps)`是驱动程序中更常用的形式。其名称是最初为其编写的每秒脉冲遥测用例的遗留。内核源代码通过`/usr/src/sys/sys/time.h`中的`#define`公开它：

```c
int    eventratecheck(struct timeval *, int *, int);
#define ppsratecheck(t, c, m) eventratecheck(t, c, m)
```

调用接受一个指向时间戳的指针、一个指向当前一秒窗口中事件计数器的指针，以及每秒允许的最大事件数。每次调用时，如果秒尚未翻转，计数器递增。如果计数器超过`maxpps`，函数返回0，调用者抑制其输出。当新的一秒开始时，计数器重置为1，函数返回1，允许新的一秒中有一次打印。`maxpps == -1`的特殊值完全禁用限速（对调试路径有用）。

两个原语都很便宜：一个比较和一个算术更新，没有锁。两者都可以安全地从驱动程序当前调用`device_printf`的任何上下文调用，包括中断处理程序，只要它们访问的存储在该上下文中是稳定的。在实践中，驱动程序将`struct timeval`和计数器保持在softc内，由保护日志站点的相同锁保护，或者在方便的地方使用每CPU状态。

FreeBSD树中的一个简短示例展示了实际使用的模式。MMC SD卡驱动程序`/usr/src/sys/dev/mmc/mmcsd.c`对写入错误投诉进行限速，以便坏卡不会淹没日志：

```c
if (ppsratecheck(&sc->log_time, &sc->log_count, LOG_PPS))
        device_printf(dev, "Error indicated: %d %s\n",
            err, mmcsd_errmsg(err));
```

驱动程序将`log_time`和`log_count`存储在其softc中，选择一个合理的`LOG_PPS`（通常为5到10），并将`device_printf`调用包装在限速检查中。任何一秒内的前几次错误产生日志行；同一秒内的接下来的几百次什么也不产生。

这就是整个想法。本节后面的所有内容都是关于以更多的结构、更多的规范和更少的重复做同样的事情。

### 一个简单的限速日志宏

目标是驱动程序可以在任何可能重复的错误或警告路径中使用的宏，而不是裸的`device_printf`。宏应该：

1. 当超过限速时静默丢弃输出。
2. 允许每个调用站点或至少每个类别有不同的速率。
3. 与第23章现有调试掩码机制配合，以便调试输出仍由掩码控制而不是由限速器控制。
4. 如果驱动程序选择，在非调试构建中编译掉，没有运行时成本。

最小实现如下。在新的`myfirst_log.h`中：

```c
#ifndef _MYFIRST_LOG_H_
#define _MYFIRST_LOG_H_

#include <sys/time.h>

struct myfirst_ratelimit {
        struct timeval rl_lasttime;
        int            rl_curpps;
};

/*
 * Default rate for warning messages: at most 10 per second per call
 * site.  Chosen to keep the log readable under a burst while still
 * showing the rate itself.
 */
#define MYF_RL_DEFAULT_PPS  10

/*
 * DLOG_RL - rate-limited device_printf.
 *
 * rlp must point at a per-call-site struct myfirst_ratelimit stored in
 * the driver (typically in the softc).  pps is the maximum allowed
 * prints per second.  The remaining arguments match device_printf.
 */
#define DLOG_RL(sc, rlp, pps, fmt, ...) do {                            \
        if (ppsratecheck(&(rlp)->rl_lasttime, &(rlp)->rl_curpps, pps))  \
                device_printf((sc)->sc_dev, fmt, ##__VA_ARGS__);        \
} while (0)

#endif /* _MYFIRST_LOG_H_ */
```

在softc中，保留一个或多个限速结构：

```c
struct myfirst_softc {
        /* ... existing fields ... */
        struct myfirst_ratelimit sc_rl_ioerr;
        struct myfirst_ratelimit sc_rl_short;
};
```

在每个错误站点，用`DLOG_RL`替换裸的`device_printf`：

```c
/* Old:
 * device_printf(sc->sc_dev, "I/O error on read, ENXIO\n");
 */
DLOG_RL(sc, &sc->sc_rl_ioerr, MYF_RL_DEFAULT_PPS,
    "I/O error on read, ENXIO\n");
```

宏在`do { ... } while (0)`块内使用逗号运算符，以便它适合任何语句适合的地方，包括在`if`和`else`体内而无需花括号。`ppsratecheck`调用开销很低；当超过限速时，`device_printf`根本不会被调用。当未超过限速时，行为与直接`device_printf`相同。

一个重要的小点：每个调用站点应该有自己的`struct myfirst_ratelimit`。在多个不相关调用站点之间共享一个结构意味着每秒第一个触发的路径会抑制其余路径在该秒的其余时间。在有少量罕见但可能错误的驱动程序中，为每个类别保留一个限速结构，以类别命名，并一致地使用它。

### 与第23章调试掩码配合

限速宏解决了错误和警告的情况。调试情况已经有了第23章自己的机制：

```c
DPRINTF(sc, MYF_DBG_IO, "read: %zu bytes requested\n", uio->uio_resid);
```

`DPRINTF`宏在`sc_debug`中相应位清除时扩展为无，所以在静默掩码（`mask = 0`）下的调试输出没有运行时成本。不需要对调试输出进行限速：操作员在想要看到时打开它，在不想要时关闭它。如果操作员在繁忙设备上打开`MYF_DBG_IO`并看到输出泛滥，那是预期的行为；他们想要泛滥。限速宏和调试宏服务于不同的目的，不应合并。

两者确实相遇的地方是偶尔的概念上是警告但开发者希望能够完全静默的日志行。对于这些，正确的模式是用调试位门控`DLOG_RL`调用：

```c
if ((sc->sc_debug & MYF_DBG_IO) != 0)
        DLOG_RL(sc, &sc->sc_rl_short, MYF_RL_DEFAULT_PPS,
            "short read: %d bytes\n", n);
```

限速在调试掩码下触发，输出既是选择性加入又是有界的。这是少数模式；大多数警告应该在限速下无条件触发，大多数调试打印应该在没有限速的情况下由掩码门控。

### `log(9)`优先级级别

第三个日志原语值得一提：`log(9)`。与总是通过内核消息缓冲区路由的`device_printf`不同，`log`通过带有syslog优先级的syslog路径路由。函数位于`/usr/src/sys/kern/subr_prf.c`，接受来自`/usr/src/sys/sys/syslog.h`的优先级：

```c
void log(int level, const char *fmt, ...);
```

常见优先级：`LOG_EMERG`（0）用于系统不可用的情况，`LOG_ALERT`（1）用于需要立即行动，`LOG_CRIT`（2）用于严重情况，`LOG_ERR`（3）用于错误情况，`LOG_WARNING`（4）用于警告，`LOG_NOTICE`（5）用于值得注意但正常的情况，`LOG_INFO`（6）用于信息性消息，`LOG_DEBUG`（7）用于调试级别消息。使用`log(LOG_WARNING, ...)`而不是`device_printf`的驱动程序可以将其警告路径通过`syslog.conf(5)`过滤到单独的日志文件，而驱动程序作者不必做其他任何事情。

权衡是`log(9)`不会在消息前添加设备名称。使用`log`的驱动程序必须手动将设备名称格式化到消息中，这很冗长。因此，大多数FreeBSD驱动程序更喜欢用`device_printf`处理操作员打算用`dmesg`阅读的特定于驱动程序的消息，在这个阶段不使用`log`。

实用指南：当消息*关于此设备*时使用`device_printf`。当消息*关于跨切面条件*、syslog基础设施是正确的查看位置时使用`log(9)`，例如认证事件或策略违规。驱动程序代码很少需要第二种。

### 内核消息缓冲区及其成本

在本节结束前还有一个技术细节。内核消息缓冲区（`msgbuf`）是内核内部的固定大小圆形缓冲区，在启动时分配。其大小由`kern.msgbufsize`可调参数控制，在amd64上默认为96 KiB，可以在`/boot/loader.conf`中提高。每个`printf`、`device_printf`和`log`调用都通过缓冲区路由。当缓冲区填满时，最旧的消息被覆盖。缓冲区的内容是`dmesg`打印的内容。

两个实际后果随之而来。首先，短消息的泛滥可以驱逐操作员需要的早期消息。一行说"hello"使用几十字节；96 KiB的缓冲区可能容纳三千行这样的行；每秒打印一万行的循环在不到半秒内驱逐整个启动日志。其次，格式化消息不是免费的。`printf`风格格式化消耗CPU，在中断处理程序或热路径内部，该成本直接显示在延迟数字中。限速宏有助于第一个后果。第二个是为什么调试消息由掩码门控：掩码为零的`DPRINTF`在运行时编译为空语句，跳过格式化和存储。

增加`kern.msgbufsize`是对反复丢失启动日志的机器的合理响应，但它不是限速的替代品。更大的缓冲区只是在泛滥驱逐旧消息之前购买更多空间；限速减少泛滥本身。两者都值得做。`/boot/loader.conf`中的`kern.msgbufsize=262144`是操作员在生产机器上的常见选择。这不是第25章的动作，因为驱动程序无法在运行时更改缓冲区大小。

### 工作示例：`myfirst`读取路径

将各部分放在一起，考虑现有的`myfirst_read`回调。第24章的一个简化版本如下所示：

```c
static int
myfirst_read(struct cdev *cdev, struct uio *uio, int ioflag)
{
        struct myfirst_softc *sc = cdev->si_drv1;
        int error = 0;

        mtx_lock(&sc->sc_mtx);
        if (uio->uio_resid == 0) {
                device_printf(sc->sc_dev, "read: empty request\n");
                goto out;
        }
        /* copy bytes into user space, update counters ... */
out:
        mtx_unlock(&sc->sc_mtx);
        return (error);
}
```

该代码有一个潜在的限速泛滥问题。在压力下，有缺陷或恶意的用户空间程序可以在紧密循环中调用`read(fd, buf, 0)`并用"empty request"行填满消息缓冲区。该事件不是驱动程序中的错误；这是一种奇怪但合法的syscall模式。记录它本身就是边缘情况，但如果驱动程序记录它，日志行必须进行限速。

重构后，同一路径如下所示：

```c
static int
myfirst_read(struct cdev *cdev, struct uio *uio, int ioflag)
{
        struct myfirst_softc *sc = cdev->si_drv1;
        int error = 0;

        mtx_lock(&sc->sc_mtx);
        if (uio->uio_resid == 0) {
                DLOG_RL(sc, &sc->sc_rl_short, MYF_RL_DEFAULT_PPS,
                    "read: empty request\n");
                goto out;
        }
        /* copy bytes into user space, update counters ... */
out:
        mtx_unlock(&sc->sc_mtx);
        return (error);
}
```

更改是三行。效果是日志不再可能泛滥，任何一秒中的第一次发生仍然为操作员产生一行以注意。softc获得一个`struct myfirst_ratelimit sc_rl_short`字段；没有其他代码移动。

对每个错误或警告路径中的每个`device_printf`应用相同的转换，为每个类别保留一个`struct myfirst_ratelimit`，驱动程序就进行了限速。差异是机械的；规范是使差异在第一时间成为可能的原因。

### 常见错误及如何避免

首次应用限速时有三个常见错误。每个一旦知道要找什么就很容易发现。

第一个错误是**在不相关调用站点之间共享单个限速结构**。如果站点A和站点B都使用`sc->sc_rl_generic`，站点A的突发会在该秒的其余时间使站点B静默，操作员只能看到一个类别。正确的规范是每个逻辑类别一个限速结构。每个驱动程序两到三个类别是常见的；十个类别是驱动程序记录太多类事件的迹象。

第二个错误是**对生命周期消息进行限速**。驱动程序加载并打印横幅。该横幅触发一次。用`ppsratecheck`包装它会无益地添加噪音，在不幸的秒边界上的第二次加载期间它可能完全跳过横幅。将限速保留给实际可能重复的消息。

第三个错误是**忘记限速计数器位于softc中**。在附加完成之前或分离开始之后触发的调用站点可能会触及限速结构尚未初始化（或已被`bzero(sc, sizeof(*sc))`清零）的softc。`struct timeval`和`int`都是值类型；零初始化的结构对于第一次调用是可以的，因为`ppsratecheck`正确处理`lasttime == 0`的情况。但后来持有垃圾的未初始化堆分配则不可以，因为`lasttime`字段可能持有一个大值，使代码认为上次事件发生在遥远的未来，随后的每次调用都返回0，直到内核时钟通过那个未来时间，这可能永远不会发生。修复是确保softc被零初始化，在`myfirst`中它已经是（newbus用`MALLOC(... M_ZERO)`分配softc）。用`M_NOWAIT`而不带`M_ZERO`分配自己状态的驱动程序必须显式调用`bzero`。

### 何时不进行限速

限速是可以频繁触发的路径的规范。一些路径不可能。`KASSERT`失败会使内核恐慌，所以限速恐慌前消息是浪费精力。中止模块加载的错误会结束加载，所以消息只能出现一次。附加时的`device_printf`每个实例最多触发一次。对于所有这些，裸的`device_printf`是正确的，额外的包装是混乱。

一个有用的经验法则：如果调用站点在`attach`完成后运行且在`detach`运行之前，并且如果事件可以由驱动程序外部的某事引起（行为不当的用户空间程序、不稳定设备、压力大的内核），那么就限速。否则，不要。

### `myfirst`驱动程序现在包含什么

本节之后，`myfirst`驱动程序的工作树有两个新文件：

```text
myfirst_log.h   - DLOG_RL宏和struct myfirst_ratelimit定义
myfirst_log.c   - 任何非平凡的限速检查辅助函数（目前为空）
```

`myfirst.h`头文件仍然保存softc。softc获得两到三个`struct myfirst_ratelimit`字段，以使用它们的调用站点类别命名。`read`、`write`和`ioctl`路径将错误站点的裸`device_printf`调用替换为`DLOG_RL`。附加、分离、打开和关闭路径保留其裸`device_printf`调用，因为这些是生命周期消息，不会重复。

`Makefile`获得一行：

```makefile
SRCS= myfirst.c myfirst_debug.c myfirst_ioctl.c myfirst_sysctl.c \
      myfirst_log.c
```

模块构建、加载，在一般情况下行为与之前完全相同。在压力下，驱动程序不再淹没消息缓冲区。这就是第1节的全部贡献。

### 第1节总结

没有规范地记录日志的驱动程序是在实验室中优雅失败、在生产环境中大声失败的驱动程序。FreeBSD限速检查原语`ratecheck(9)`和`ppsratecheck(9)`小到可以在一小时内理解，有效到可以在驱动程序的余生中偿还其成本。与第23章现有调试掩码机制结合，它们为`myfirst`驱动程序提供了干净的三重日志故事：生命周期消息通过普通`device_printf`，错误和警告消息通过`DLOG_RL`，调试消息通过掩码下的`DPRINTF`。

在下一节中，我们将从驱动程序说什么转向它返回什么。日志行是给操作员的；errno是给调用者的。一个对操作员说对但对调用者说错的驱动程序仍然是一个损坏的驱动程序。

## 第2节：错误报告与返回约定

本章教授的第二个规范是返回正确errno的规范。errno是一个小数字。可能的errno集合在`/usr/src/sys/sys/errno.h`中定义，在撰写本文时FreeBSD定义的errno不到一百个。一个粗心选择返回哪个errno的驱动程序在当下看起来很好，因为调用者主要只检查返回值是否为非零，任何非零值都会通过该测试。几个月后，当第一个用户空间辅助程序试图区分调用失败的原因时，情况就变得不那么好了，驱动程序的errno选择结果是不一致的。本节教授驱动程序例行使用的小型errno词汇表，展示如何在它们之间选择，并演练`myfirst`驱动程序现有路径的审计。

### 为什么errno规范很重要

驱动程序的errno返回是与每个调用者的契约。用户空间程序通过`strerror(3)`和直接比较（`if (errno == EBUSY)`）使用errno。调用驱动程序回调的内核代码使用返回值决定下一步做什么：返回`EBUSY`的`d_open`导致内核以`EBUSY`失败`open(2)`系统调用；返回`ENOIOCTL`的`d_ioctl`导致内核回退到通用ioctl层；返回非零值的`device_attach`导致Newbus回滚附加并放弃设备。这些消费者中的每一个都期望特定值意味着特定的事情。在预期`ENXIO`的地方返回`EINVAL`的驱动程序不一定会失败；通常它只是误导，误导的errno会显示为驱动程序作者永远看不到的某处的令人困惑的诊断。

规范很便宜。忽略它的成本随时间复合。从一开始就选择好errno的驱动程序是产生准确手册页、准确用户空间辅助程序错误消息和准确错误报告的驱动程序。对errno粗心的驱动程序开始产生在许多地方略错的`strerror`输出，用户空间生态系统继承了这种粗心。

### 小型词汇表

完整的errno列表很长。典型字符驱动程序使用的子集很短。下表是FreeBSD驱动程序中最常需要的词汇表，按何时使用每个分组。

| Errno | 数值 | 何时返回 |
|-------|------|---------|
| `0` | 0 | 成功。唯一的非错误返回。 |
| `EPERM` | 1 | 调用者缺乏执行请求操作的权限，即使调用本身格式良好。例如：非root用户请求特权ioctl。 |
| `ENOENT` | 2 | 请求的对象不存在。例如：按名称或ID查找未找到任何内容。 |
| `EIO` | 5 | 来自硬件的通用I/O错误。当硬件返回失败且没有更具体的errno时使用。 |
| `ENXIO` | 6 | 设备已消失、分离或以其他方式不可达。例如：底层设备已被移除的文件描述符上的ioctl。与`ENOENT`不同：对象曾经存在，现在已消失。 |
| `EBADF` | 9 | 文件描述符未正确打开以进行操作。例如：在以只读方式打开的文件描述符上进行`MYFIRSTIOC_SETMSG`调用。 |
| `ENOMEM` | 12 | 分配失败。用于`malloc(M_NOWAIT)`失败等。 |
| `EACCES` | 13 | 调用者在文件系统级别缺乏权限。与`EPERM`不同：`EACCES`关于文件权限，`EPERM`关于特权。 |
| `EFAULT` | 14 | 用户指针无效。由失败的`copyin`或`copyout`返回。驱动程序应该原封不动地转发`copyin`/`copyout`失败。 |
| `EBUSY` | 16 | 资源正在使用中。用于因为客户端仍然保持设备打开而无法进行的`detach`，或无法等待的类似互斥锁的获取尝试。 |
| `EINVAL` | 22 | 参数已识别但无效。当驱动程序理解请求但输入格式错误时使用。 |
| `EAGAIN` | 35 | 稍后重试。当操作会阻塞时从非阻塞I/O返回，或可能在重试时成功的分配失败返回。 |
| `EOPNOTSUPP` | 45 | 此驱动程序不支持该操作。当调用格式良好但驱动程序没有处理它的代码时使用。 |
| `ETIMEDOUT` | 60 | 等待超时。用于在驱动程序超时预算内未完成的硬件命令。 |
| `ENOIOCTL` | -3 | ioctl命令此驱动程序未知。**在`d_ioctl`的默认情况下使用此值；内核将其转换为用户空间的`ENOTTY`。** |
| `ENOSPC` | 28 | 没有剩余空间，无论是在设备上、缓冲区中还是内部表中。 |

此表中有三对以容易混淆而闻名：`EPERM`与`EACCES`、`ENOENT`与`ENXIO`、`EINVAL`与`EOPNOTSUPP`。每对都值得依次看一看。

`EPERM`与`EACCES`。`EPERM`关于特权：调用者没有足够的特权来执行操作。`EACCES`关于权限：文件系统ACL或模式位禁止访问。当节点模式为`0600 root:wheel`时，尝试写入`/dev/myfirst0`的非root用户在驱动程序被咨询之前从内核获得`EACCES`。尝试调用驱动程序因调用者不在特定jail中而拒绝的特权ioctl的root用户从驱动程序获得`EPERM`。区别很重要，因为管理员的补救措施不同：`EACCES`要求管理员调整设备权限，而`EPERM`要求管理员调整调用者的特权。

`ENOENT`与`ENXIO`。`ENOENT`是*没有这样的对象*。`ENXIO`是*对象已消失，或设备不可达*。在查找驱动程序内部表时，当请求的键不存在时，`ENOENT`是正确答案。在对已被分离或已发出意外移除条件的设备进行操作时，`ENXIO`是正确答案。区别很重要，因为运维工具对它们的处理不同：`ENOENT`建议调用者给出了错误的键；`ENXIO`建议设备需要重新附加。

`EINVAL`与`EOPNOTSUPP`。`EINVAL`是*我理解你问什么但参数错误*。`EOPNOTSUPP`是*我不支持你问什么*。带有太长缓冲区的`MYFIRSTIOC_SETMSG`调用是`EINVAL`。对驱动程序从未实现的模式进行的`MYFIRSTIOC_SETMODE`调用是`EOPNOTSUPP`。区别很重要，因为`EOPNOTSUPP`告诉调用者使用不同的方法，而`EINVAL`告诉调用者修复参数并重试。

第四个混淆值得单独一段：`ENOIOCTL`与`ENOTTY`。`ENOIOCTL`是为内核内部的ioctl代码路径定义的负值（`-3`）。驱动程序的`d_ioctl`默认情况返回`ENOIOCTL`告诉内核"我不认识这个命令；请回退到通用层"。通用层处理`FIONBIO`、`FIOASYNC`、`FIOGETOWN`、`FIOSETOWN`和类似的跨设备ioctl。如果通用层也不认识该命令，它将`ENOIOCTL`转换为`ENOTTY`（正数25）传递给用户空间。常见的错误是从`d_ioctl`开关的默认情况返回`EINVAL`，这会完全抑制通用回退。第24章的驱动程序已经正确返回`ENOIOCTL`；第25章的审计确认它并检查驱动程序中每个其他errno是否有类似问题。

### Ioctl分发器审计

第一次审计通过针对`myfirst_ioctl.c`。switch语句中的每个case最多产生一个非零返回。审计查看每个并询问返回的errno是否正确。

Case `MYFIRSTIOC_GETVER`：成功时返回0，永不失败。无需审计。

Case `MYFIRSTIOC_GETMSG`：成功时返回0。当前代码不因`fflag`而拒绝，因为消息是公共的。那是设计选择，不是错误。如果驱动程序想要将`GETMSG`限制为读取者（即要求`FREAD`），它会在fflag检查时返回`EBADF`，与`SETMSG`和`RESET`情况一致。

Case `MYFIRSTIOC_SETMSG`：当文件描述符缺少`FWRITE`时返回`EBADF`，这是正确的。第二个审计问题是当输入不是NUL结尾时会发生什么：内核中的`strlcpy`容忍它（复制到`MYFIRST_MSG_MAX - 1`并终止），所以驱动程序不需要检查。第三个问题是在复制之前是否应该验证长度。内核的自动`copyin`已经强制执行了嵌入ioctl编码的固定长度，所以没有用户空间缓冲区需要验证；值在`data`中并且已经被复制了。

Case `MYFIRSTIOC_RESET`：当文件描述符缺少`FWRITE`时返回`EBADF`。第25章审计提出第二个问题：重置应该是特权的吗？让任何写入者调用`RESET`并归零统计数据的驱动程序暴露了轻微的拒绝服务面。简单的修复是在执行重置之前检查`priv_check(td, PRIV_DRIVER)`：

```c
case MYFIRSTIOC_RESET:
        if ((fflag & FWRITE) == 0) {
                error = EBADF;
                break;
        }
        error = priv_check(td, PRIV_DRIVER);
        if (error != 0)
                break;
        /* ... existing reset body ... */
        break;
```

如果`priv_check`失败，errno是`EPERM`（内核返回`EPERM`而不是`EACCES`，因为检查是关于特权，而不是文件系统权限）。以root运行的`myfirstctl`程序看到0；以`_myfirst`用户运行的非root程序看到`EPERM`。

Default case：返回`ENOIOCTL`，这是正确的。不要动它。

### 读取和写入路径审计

第二次审计通过针对读取和写入回调。

对于`myfirst_read`，当前代码成功时返回0，`uiomove`失败时返回`EFAULT`，`uio_resid == 0`时返回0。空请求返回0是标准的UNIX行为（零字节`read`是允许的并返回0字节），是正确的。无需更改errno。

对于`myfirst_write`，同样，成功时返回0，`uiomove`失败时返回`EFAULT`，零字节写入时返回0。正确。

两个回调都不需要`EIO`：驱动程序此时不做硬件I/O，所以没有硬件故障需要传播。驱动真实硬件的未来版本会在硬件指示传输级故障时从读取或写入回调返回`EIO`。现在添加该返回是过早的；这是第28章存储工作将具体处理的事情。

### 打开和关闭路径审计

打开回调目前无条件返回0。审计问题是它是否应该失败。三种失败模式是传统的：设备是独占打开的并且已经有用户（`EBUSY`），设备已断电且当前不接受打开（`ENXIO`），或驱动程序正在被分离（`ENXIO`）。简单的`myfirst`驱动程序不强制独占打开，它总是接受打开，除非在分离期间。在分离期间，内核在分离返回之前销毁cdev，所以在`destroy_dev`开始后到达的任何打开在驱动程序的`d_open`被调用之前就被内核本身拒绝了。因此`myfirst`驱动程序不需要显式的`ENXIO`逻辑。让打开回调返回0是正确的。

关闭回调无条件返回0。这是正确的。`d_close`返回非零的唯一可能原因是在关闭期间有硬件操作失败；由于`myfirst`驱动程序不进行此类操作，0是正确的返回。

### 附加和分离路径审计

附加和分离是Newbus调用的回调。它们的返回值告诉Newbus是回滚还是继续。

`myfirst_attach`的非零返回意味着"附加失败；请回滚"。附加中的每个错误路径必须返回一个正errno。当前代码返回来自`make_dev_s`的`error`值，失败时为正；这是正确的。本章第5节的添加将引入更多带有标签goto的错误路径；每个都将使用失败步骤的正确errno（分配失败的`ENOMEM`，资源分配失败的`ENXIO`等）。

`myfirst_detach`的非零返回意味着"现在无法分离；请保持设备附加"。当前代码在`sc_open_count > 0`时返回`EBUSY`，这是正确的。Newbus将分离返回的`EBUSY`转换为具有相同errno的`devctl detach`失败，这是正确的用户可见行为。

模块事件处理器（`myfirst_modevent`）返回非零以拒绝事件。因为某个设备实例仍在使用而无法进行的`MOD_UNLOAD`返回`EBUSY`。因为健全性检查失败而无法进行的`MOD_LOAD`返回适当的errno（`ENOMEM`、`EINVAL`等）。当前代码是正确的。

### Sysctl处理程序审计

Sysctl处理程序有自己的errno约定。第24章的驱动程序有一个自定义处理程序`myfirst_sysctl_message_len`。其主体为：

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

处理程序使用`sysctl_handle_int`读取其输入，成功时返回0，失败时返回正errno。处理程序原封不动地转发该errno，这是正确的。无需审计更改。

写入（而不是只读取）的sysctl处理程序应该检查`req->newptr`以区分读取和写入，并如果对只读OID尝试写入则返回`EPERM`。现有的`debug.mask` OID用`CTLFLAG_RW`声明，所以内核自动允许写入；处理程序不需要权限检查，因为OID已经被sysctl MIB权限限制为root。第25章的驱动程序在这个阶段不添加更多自定义sysctl处理程序。

### 错误路径消息

返回正确的errno是契约的一半。发出正确的日志消息是另一半。规范将第1节的限速日志与第2节的errno词汇表结合。警告路径如下所示：

```c
if (input_too_large) {
        DLOG_RL(sc, &sc->sc_rl_inval, MYF_RL_DEFAULT_PPS,
            "ioctl: SETMSG buffer too large (%zu > %d)\n",
            length, MYFIRST_MSG_MAX);
        error = EINVAL;
        break;
}
```

三个属性使这成为一个好的错误路径。首先，日志消息命名调用（"ioctl: SETMSG"）、原因（"buffer too large"）和涉及的数值。其次，返回的errno是`EINVAL`，这是"我理解但参数错误"的正确值。第三，整个路径是限速的，所以有缺陷的用户空间程序在紧密循环中调用ioctl不能淹没消息缓冲区。

一个坏的错误路径如下所示：

```c
if (input_too_large) {
        device_printf(sc->sc_dev, "ioctl failed\n");
        return (-1);
}
```

三个属性使这成为一个坏的错误路径。日志消息没有信息量："ioctl failed"没有说出调用者不知道的任何事情。返回值是`-1`，这不是有效的内核errno。而且日志行没有限速，所以行为不当的调用者可以用噪音填满消息缓冲区。

好的路径需要九行，坏的路径需要三行，这是一个好的交易。错误日志行只在有问题时打印；当它确实打印时多花几秒钟使其有信息量是值得的。

### 模块事件处理程序约定

模块事件处理器有自己的errno约定。处理程序签名为：

```c
static int
myfirst_modevent(module_t mod, int what, void *arg)
{
        switch (what) {
        case MOD_LOAD:
                /* Driver-wide init. */
                return (0);
        case MOD_UNLOAD:
                /* Driver-wide teardown. */
                return (0);
        case MOD_QUIESCE:
                /* Pause and prepare for unload. */
                return (0);
        case MOD_SHUTDOWN:
                /* System shutting down. */
                return (0);
        default:
                return (EOPNOTSUPP);
        }
}
```

每个case成功时返回0，或返回正errno以拒绝事件。每个case的特定errno：

- `MOD_LOAD`：如果全局分配失败返回`ENOMEM`，如果驱动程序与当前内核不兼容返回`ENXIO`，如果可调参数值超出范围返回`EINVAL`。
- `MOD_UNLOAD`：如果驱动程序现在无法卸载因为某个实例仍在使用返回`EBUSY`。内核尊重这一点并保持模块加载。
- `MOD_QUIESCE`：如果驱动程序无法暂停返回`EBUSY`。不支持停顿的驱动程序只需从此case返回0，因为停顿是一个可选功能，返回成功在拥有没有飞行中工作的意义上说"我已暂停"。
- `MOD_SHUTDOWN`：很少失败；除非驱动程序有特定理由反对关机，否则返回0。想要刷新持久状态的驱动程序使用`shutdown_pre_sync`上的`EVENTHANDLER`而不是拒绝`MOD_SHUTDOWN`。
- 默认case返回`EOPNOTSUPP`以指示驱动程序不认识事件类型。这不是错误；这是说"我不实现此事件"的标准方式。

### Errno检查清单

作为本节的结束，读者可以对自己编写的任何驱动程序运行检查清单。每一项都是一个答案应该是是的问题。

1. 回调中的每个非零返回是来自`errno.h`的正errno，除了`d_ioctl`可能返回`ENOIOCTL`（负值）。
2. `copyin`和`copyout`失败原封不动地传播其errno（通常是`EFAULT`）。
3. `d_ioctl`的默认case返回`ENOIOCTL`，而不是`EINVAL`。
4. 如果设备仍在使用，`d_detach`返回`EBUSY`，而不是`ENXIO`或其他值。
5. 如果底层硬件已消失或驱动程序正在被分离，`d_open`返回`ENXIO`，而不是`EIO`。
6. 如果文件描述符缺少`FWRITE`，`d_write`返回`EBADF`，而不是`EPERM`。
7. 每个错误路径记录命名调用、原因和相关值的消息，使用限速宏。
8. 没有错误路径记录*并且*返回通用errno。如果驱动程序有足够的上下文记录具体原因，它就有足够的上下文返回具体errno。
9. 驱动程序一致地区分`EINVAL`（参数错误）和`EOPNOTSUPP`（特性缺失）。
10. 驱动程序一致地区分`ENOENT`（没有这样的键）和`ENXIO`（设备不可达）。

通过此检查清单的驱动程序具有一致的errno接口，该接口小到足以使手册页可以列出驱动程序返回的每个errno并准确说明每个何时发生。

### 第2节总结

Errno是一个小型词汇表和一个契约。对任一方面的粗心都会显示为用户空间中的令人困惑的行为；对两者的规范显示为准确的诊断和更短的错误报告。与第1节的限速日志结合，`myfirst`驱动程序现在对操作员（通过日志行）和调用者（通过errno）都谨慎地交流。

在下一节中，我们将看看驱动程序欠契约的第三个受众：通过`/boot/loader.conf`和`sysctl`配置驱动程序的管理员。配置是第三种契约，对它的规范是驱动程序如何在不需重建的情况下跨工作负载保持有用。

## 第3节：通过加载器可调参数和sysctl进行驱动配置

第三个规范是外化决策的规范。任何驱动程序都有某人可能合理希望在不重建模块的情况下更改的值：超时、重试计数、内部缓冲区大小、详细级别、特性开关。将这些值烘焙到源代码中的驱动程序强制每个更改通过完整的编译、安装和重启循环。将它们公开为加载器可调参数和sysctl的驱动程序让操作员通过单个编辑或单个命令在启动时或运行时调整行为。提供旋钮的成本很小；不提供它们的成本由操作员支付。

本节教授FreeBSD外化配置的两种机制：加载器可调参数（从`/boot/loader.conf`读取并在内核到达`attach`之前应用）和sysctl（通过`sysctl(8)`在运行时读写）。它解释它们如何通过`CTLFLAG_TUN`标志配合，展示如何在每驱动程序和每实例可调参数之间选择，演练`TUNABLE_*_FETCH`系列，并以一个简短的实验室结束，其中`myfirst`驱动程序获得三个新可调参数，读者用每个启动VM。

### 可调参数与Sysctl的区别

对于操作员来说，可调参数和sysctl看起来很相似。两者都是命名空间中的字符串，如`hw.myfirst.debug_mask_default`或`dev.myfirst.0.debug.mask`。两者都接受操作员设置的值。两者最终都在内核内存中。它们在何时和如何上有所不同。

**可调参数**是在引导加载程序环境中设置的变量。引导加载程序（`loader(8)`）读取`/boot/loader.conf`，将其`key=value`对收集到环境中，并在内核启动时将该环境传递给内核。内核通过`getenv(9)`系列和`TUNABLE_*_FETCH`宏公开此环境。可调参数在启动期间读取，通常在相应驱动程序附加之前。它们不能在运行时更改（更改`/boot/loader.conf`需要重启才能生效）。它们适用于在`attach`运行之前必须知道的值：静态分配表的大小、控制哪些代码路径编译到附加路径的特性标志、调试掩码的初始值。

**sysctl**是内核分层配置树中的变量，在运行时通过`sysctl(2)`系统调用和`sysctl(8)`工具访问。sysctl可以是只读（`CTLFLAG_RD`）、读写（`CTLFLAG_RW`）或root可写只读（各种标志组合）。它们适用于在驱动程序附加后更改有意义的值：详细级别、节流速率、计数器重置命令、可写状态旋钮。

有用的特性是两种机制可以共享一个变量。用`CTLFLAG_TUN`声明的sysctl告诉内核在启动时读取同名的可调参数并将其值用作sysctl的初始值。然后操作员可以在运行时调整sysctl，可调参数作为默认值在重启后保留。`myfirst`驱动程序已经为其调试掩码使用了此模式：`debug.mask`是`CTLFLAG_RW | CTLFLAG_TUN` sysctl，`hw.myfirst.debug_mask_default`是`/boot/loader.conf`中匹配的可调参数。第3节将该模式推广到驱动程序想要公开的每个配置旋钮。

### `TUNABLE_*_FETCH`系列

FreeBSD提供了一系列宏用于从引导加载程序环境读取可调参数。每个宏读取命名可调参数，将其解析为正确的C类型，并存储结果。如果未设置可调参数，变量保留其现有值；因此调用者必须在调用获取宏之前将变量初始化为正确的默认值。

这些宏在`/usr/src/sys/sys/kernel.h`中声明：

```c
TUNABLE_INT_FETCH(path, pval)        /* int */
TUNABLE_LONG_FETCH(path, pval)       /* long */
TUNABLE_ULONG_FETCH(path, pval)      /* unsigned long */
TUNABLE_INT64_FETCH(path, pval)      /* int64_t */
TUNABLE_UINT64_FETCH(path, pval)     /* uint64_t */
TUNABLE_BOOL_FETCH(path, pval)       /* bool */
TUNABLE_STR_FETCH(path, pval, size)  /* char buffer of given size */
```

每个都扩展为匹配的`getenv_*`调用。例如，对于`TUNABLE_INT_FETCH`，扩展是`getenv_int(path, pval)`，它读取引导加载程序环境并将值解析为整数。

路径是一个字符串，按照惯例采用每驱动程序可调参数的`hw.<driver>.<knob>`形式和每实例可调参数的`hw.<driver>.<unit>.<knob>`形式。`hw.`前缀是硬件相关可调参数的惯例；其他前缀（`kern.`、`net.`）存在于不同子系统，但在驱动程序代码中较少见。

`myfirst`驱动程序的一个工作示例展示了该模式：

```c
static int
myfirst_attach(device_t dev)
{
        struct myfirst_softc *sc = device_get_softc(dev);
        int error;

        /* Initialise defaults. */
        sc->sc_debug = 0;
        sc->sc_timeout_sec = 30;
        sc->sc_max_retries = 3;

        /* Read tunables.  The variables keep their default values if
         * the tunables are not set. */
        TUNABLE_INT_FETCH("hw.myfirst.debug_mask_default", &sc->sc_debug);
        TUNABLE_INT_FETCH("hw.myfirst.timeout_sec", &sc->sc_timeout_sec);
        TUNABLE_INT_FETCH("hw.myfirst.max_retries", &sc->sc_max_retries);

        /* ... rest of attach ... */
}
```

操作员在`/boot/loader.conf`中设置可调参数：

```ini
hw.myfirst.debug_mask_default="0xff"
hw.myfirst.timeout_sec="15"
hw.myfirst.max_retries="5"
```

重启后，`myfirst`的每个实例都以`sc_debug=0xff`、`sc_timeout_sec=15`、`sc_max_retries=5`附加。无需重建；值位于驱动程序源代码之外。

### 每驱动程序与每实例可调参数

可能多次附加的驱动程序需要做出决定：其可调参数是应用于每个实例，还是每个实例有自己的？

每驱动程序形式使用`hw.myfirst.debug_mask_default`形式的路径。`myfirst`的每个实例在附加时读取这个单个变量，因此所有实例以相同的默认值开始。这是较简单的形式，当可调参数在每个实例上含义相同时是正确的。

每实例形式使用`hw.myfirst.0.debug_mask_default`形式的路径，其中`0`是单元号。每个实例读取自己的变量，因此实例0和实例1可以有不同的默认值。当每个实例背后的硬件可能合理需要不同配置时，这是正确的形式，例如同一系统上具有不同工作负载的两个PCI适配器。

该决定是设计选择，而不是正确性问题。大多数驱动程序对大多数可调参数使用每驱动程序形式，将每实例形式保留给少数实际重要的每实例配置情况。对于`myfirst`，一个虚构的伪设备，每驱动程序是每个可调参数的正确默认值。因此第25章驱动程序添加三个每驱动程序可调参数（`timeout_sec`、`max_retries`、`log_ratelimit_pps`）并保留现有的每驱动程序`debug_mask_default`。

如果驱动程序需要，结合两种形式的模式是先读取每驱动程序可调参数作为基线，然后读取每实例可调参数作为覆盖：

```c
int defval = 30;

TUNABLE_INT_FETCH("hw.myfirst.timeout_sec", &defval);
sc->sc_timeout_sec = defval;
TUNABLE_INT_FETCH_UNIT("hw.myfirst", unit, "timeout_sec",
    &sc->sc_timeout_sec);
```

FreeBSD没有开箱即用的`TUNABLE_INT_FETCH_UNIT`宏；需要此功能的驱动程序必须使用`snprintf`组合路径，然后手动调用`getenv_int`。工作量很小但需求很少，所以`myfirst`不会那样做。

### `CTLFLAG_TUN`标志

外化故事的另一半是可调参数本身只在启动时读取。要使相同的值在运行时可调整，驱动程序用`CTLFLAG_TUN`声明匹配的sysctl：

```c
SYSCTL_ADD_UINT(ctx, child, OID_AUTO, "debug.mask",
    CTLFLAG_RW | CTLFLAG_TUN,
    &sc->sc_debug, 0,
    "Bitmask of enabled debug classes");
```

`CTLFLAG_TUN`告诉内核此sysctl的初始值应该从引导加载程序环境中同名变量获取，使用OID名称作为键。匹配是文本的和自动的；驱动程序不需要单独调用`TUNABLE_INT_FETCH`。

关于`CTLFLAG_TUN`何时生效有一个微妙的规则。该标志适用于OID的*初始*值，在创建sysctl时从环境读取。如果驱动程序在创建sysctl之前显式调用`TUNABLE_INT_FETCH`，显式获取优先，`CTLFLAG_TUN`实际上是多余的。如果驱动程序不调用`TUNABLE_INT_FETCH`而仅依赖`CTLFLAG_TUN`，sysctl的初始值自动来自环境。

在实践中，`myfirst`驱动程序为清晰起见同时使用两种机制。附加中的显式`TUNABLE_INT_FETCH`使驱动程序的意图在源代码中可见；sysctl上的`CTLFLAG_TUN`为操作员在sysctl文档中提供清晰的提示，即OID尊重加载器可调参数。单独任一机制都可以工作；同时使用两者是在可读性上有回报的小型重复。

### 将可调参数声明为静态sysctl

对于不属于特定实例的驱动程序范围sysctl，FreeBSD提供将sysctl绑定到静态变量并在一个声明中从环境读取其默认值的编译时宏。规范形式：

```c
SYSCTL_NODE(_hw, OID_AUTO, myfirst, CTLFLAG_RW, NULL,
    "myfirst pseudo-driver");

static int myfirst_verbose = 0;
SYSCTL_INT(_hw_myfirst, OID_AUTO, verbose,
    CTLFLAG_RWTUN, &myfirst_verbose, 0,
    "Enable verbose driver logging");
```

`SYSCTL_NODE`声明一个新的父节点`hw.myfirst`。`SYSCTL_INT`声明一个整数OID `hw.myfirst.verbose`，带有`CTLFLAG_RWTUN`（结合`CTLFLAG_RW`和`CTLFLAG_TUN`）。`myfirst_verbose`变量是驱动程序的全局详细级别。操作员在`/boot/loader.conf`中设置`hw.myfirst.verbose=1`以在启动时启用详细输出，或运行`sysctl hw.myfirst.verbose=1`在运行时切换它。

静态声明适用于驱动程序范围状态。每实例状态（`sc_debug`、计数器）继续位于`dev.myfirst.<unit>.*`下，并通过`device_get_sysctl_ctx`动态声明。

### 关于`SYSCTL_INT`与`SYSCTL_ADD_INT`的小注

静态形式`SYSCTL_INT(parent, OID_AUTO, ...)`是编译时声明。动态形式`SYSCTL_ADD_INT(ctx, list, OID_AUTO, ...)`是运行时调用。两者都产生sysctl OID。静态形式适用于其存在不依赖于附加硬件的驱动程序范围sysctl。动态形式适用于在附加时创建并在分离时销毁的每实例sysctl。

初学者常见的错误是对驱动程序范围sysctl使用动态形式，这可以工作，但需要一个必须在`MOD_LOAD`时初始化并在`MOD_UNLOAD`时释放的驱动程序范围`sysctl_ctx_list`。静态形式避免了所有这些：sysctl从模块加载那一刻存在直到卸载那一刻，内核自动处理注册和注销。

### 记录可调参数

操作员不知道的可调参数是不会被使用的可调参数。规范是在三个地方记录驱动程序公开的每个可调参数。

首先，源代码中可调参数的声明应该包含一行描述字符串。对于`SYSCTL_ADD_UINT`等，最后一个参数是描述：

```c
SYSCTL_ADD_UINT(ctx, child, OID_AUTO, "timeout_sec",
    CTLFLAG_RW | CTLFLAG_TUN,
    &sc->sc_timeout_sec, 0,
    "Timeout in seconds for hardware commands (default 30, min 1, max 3600)");
```

描述字符串是`sysctl -d`在操作员请求文档时打印的内容。好的描述命名单位、默认值和可接受范围。

其次，驱动程序的`MAINTENANCE.md`（在第7节介绍）应该列出每个可调参数，每个有一段。该段落解释可调参数做什么、何时更改它、默认值是什么以及设置它有什么副作用。

第三，驱动程序的手册页（通常是`myfirst(4)`）应该在`LOADER TUNABLES`部分列出每个可调参数，在`SYSCTL VARIABLES`部分列出每个sysctl。`myfirst`驱动程序还没有手册页；本章将手册页视为后续关注点。在此期间，`MAINTENANCE.md`文档承载完整的文档。

### 工作示例：`hw.myfirst.timeout_sec`

`myfirst`驱动程序在这个阶段没有真实硬件，但本章介绍一个未来章节将使用的虚构`timeout_sec`旋钮。完整的迷你工作流程为：

1. 在`myfirst.h`中，将字段添加到softc：
   ```c
   struct myfirst_softc {
           /* ... existing fields ... */
           int   sc_timeout_sec;
   };
   ```

2. 在`myfirst_bus.c`（第6节介绍的新文件，保存附加和分离）中，初始化默认值并读取可调参数：
   ```c
   sc->sc_timeout_sec = 30;
   TUNABLE_INT_FETCH("hw.myfirst.timeout_sec", &sc->sc_timeout_sec);
   ```

3. 在`myfirst_sysctl.c`中，将旋钮公开为运行时sysctl：
   ```c
   SYSCTL_ADD_INT(ctx, child, OID_AUTO, "timeout_sec",
       CTLFLAG_RW | CTLFLAG_TUN,
       &sc->sc_timeout_sec, 0,
       "Timeout in seconds for hardware commands");
   ```

4. 在`MAINTENANCE.md`中，记录可调参数：
   ```
   hw.myfirst.timeout_sec
       Timeout in seconds for hardware commands.  Default 30.
       Acceptable range 1 through 3600.  Values below 1 are
       clamped to 1; values above 3600 are clamped to 3600.
       Adjustable at run time via sysctl dev.myfirst.<unit>.
       timeout_sec.
   ```

5. 在回归脚本中，添加一行验证可调参数获取其默认值：
   ```
   [ "$(sysctl -n dev.myfirst.0.timeout_sec)" = "30" ] || fail
   ```

驱动程序现在有一个操作员可以通过`/boot/loader.conf`在启动时设置、可以通过`sysctl`在运行时调整、并可以在`MAINTENANCE.md`中找到文档的超时旋钮。引入新的可配置值的每个未来章节都将遵循相同的五步工作流程。

### 范围检查和验证

操作员可以设置任何值的可调参数是可以设置为超出范围值的可调参数，无论是意外（`/boot/loader.conf`中的拼写错误）还是误导性的调优尝试。驱动程序必须验证它读取的值并将其限制或拒绝。

对于在启动时用`TUNABLE_INT_FETCH`读取的可调参数，验证内联进行：

```c
sc->sc_timeout_sec = 30;
TUNABLE_INT_FETCH("hw.myfirst.timeout_sec", &sc->sc_timeout_sec);
if (sc->sc_timeout_sec < 1 || sc->sc_timeout_sec > 3600) {
        device_printf(dev,
            "tunable hw.myfirst.timeout_sec out of range (%d), "
            "clamping to default 30\n",
            sc->sc_timeout_sec);
        sc->sc_timeout_sec = 30;
}
```

对于具有运行时写入支持的sysctl，验证在处理程序中进行。简单的`CTLFLAG_RW` sysctl接受任何int值；要拒绝超出范围的写入，驱动程序声明自定义处理程序：

```c
static int
myfirst_sysctl_timeout(SYSCTL_HANDLER_ARGS)
{
        struct myfirst_softc *sc = arg1;
        int v;
        int error;

        v = sc->sc_timeout_sec;
        error = sysctl_handle_int(oidp, &v, 0, req);
        if (error != 0 || req->newptr == NULL)
                return (error);
        if (v < 1 || v > 3600)
                return (EINVAL);
        sc->sc_timeout_sec = v;
        return (0);
}
```

处理程序读取当前值，调用`sysctl_handle_int`执行实际I/O，并仅在值在范围内时应用新值。写入0或7200向操作员返回`EINVAL`而不更改sysctl的值。这是正确的行为：操作员获得写入被拒绝的明确反馈。

`myfirst`驱动程序在这个阶段不验证其整数sysctl，因为它们都不能有意义的超出范围（调试掩码是位掩码，任何32位值都是合法掩码）。引入超时、重试计数和缓冲区大小的未来驱动程序将一致地使用自定义处理程序模式。

### 何时公开可调参数何时保留内部

公开可调参数是一个承诺。一旦操作员在`/boot/loader.conf`中设置`hw.myfirst.timeout_sec=15`，驱动程序就承诺该旋钮的含义不会在后续版本中更改。移除可调参数会破坏生产部署。静默更改其解释会破坏得更严重。

正确的规范是仅当以下三者都为真时才将值公开为可调参数：

1. 该值具有运维用例。某人可能合理需要在实际部署上更改它。
2. 合理值的范围已知。驱动程序可以在`MAINTENANCE.md`中记录它。
3. 在驱动程序生命周期内支持该旋钮的成本值得它提供的运维价值。

对自己提出这三个问题并对三者都回答是的驱动程序公开了一个小型的、有目的的可调参数集。因为"操作员可能想要调优它"而将每个内部常量公开为可调参数的驱动程序最终得到一个没人能记录也没人能在其完整范围内测试的 sprawling 配置接口。

对于`myfirst`，初始可调参数集故意很小：`debug_mask_default`、`timeout_sec`、`max_retries`、`log_ratelimit_pps`。每个都有清晰的运维案例、清晰的默认值和清晰的范围。驱动程序不是试图将softc中的每个int字段公开为可调参数；它试图公开操作员可能实际想要触摸的那些。

### 关于字符串`CTLFLAG_RWTUN`的警告说明

`TUNABLE_STR_FETCH`宏从引导加载程序环境将字符串读取到固定大小的缓冲区中。匹配的sysctl标志，`SYSCTL_STRING`上的`CTLFLAG_RWTUN`，可以工作，但它有一个陷阱：字符串的存储必须是静态缓冲区，而不是softc中的每实例`char[]`字段。写入softc字段的字符串sysctl可能在sysctl框架在softc释放之前未注销OID的情况下比softc存活更久，这会导致释放后使用错误。

更安全的模式是将字符串公开为只读，并通过在锁下将新值复制到softc的自定义处理程序处理写入。`myfirst`驱动程序遵循此模式：`dev.myfirst.0.message`仅用`CTLFLAG_RD`公开，写入通过`MYFIRSTIOC_SETMSG` ioctl进行。ioctl路径获取softc互斥锁，复制新值，并解锁；sysctl OID没有生命周期问题。

字符串可调参数和sysctl对某些驱动程序足够有用，值得小心处理，但第25章驱动程序不需要它们。该原则值得命名，因为该陷阱稍后会在真实驱动程序中出现。

### 可调参数与内核模块：它们位于何处

关于加载器环境的两个小但重要的细节值得命名。

首先，`/boot/loader.conf`中的可调参数从内核启动的那一刻起就适用。它对任何调用`TUNABLE_*_FETCH`或具有`CTLFLAG_TUN` sysctl的模块都可用，即使模块不是在启动时加载的。稍后用`kldload`加载的模块仍然看到可调参数的值。这很方便：操作员设置一次可调参数并忘记它，直到模块加载。

其次，可调参数从环境读取但不能写回。在运行时更改`hw.myfirst.timeout_sec`（用`kenv`）不会影响任何已经读取它的驱动程序；softc中的变量才是重要的，而不是环境。要在运行时更改值，操作员使用匹配的sysctl。

这两个细节一起解释了为什么`CTLFLAG_TUN`是大多数配置旋钮的正确形状：可调参数设置启动默认值，sysctl处理运行时调整，操作员的工具包（`/boot/loader.conf`加`sysctl(8)`）按预期工作。

### 第3节总结

配置是与操作员的对话。通过可调参数和sysctl外化正确值的驱动程序可以在不需重建的情况下调优；将每个值隐藏在源代码中的驱动程序强制每次更改都重建。`TUNABLE_*_FETCH`系列和`CTLFLAG_TUN`标志一起覆盖启动时和运行时调整，每驱动程序与每实例的选择使驱动程序适应其运维现实。`myfirst`驱动程序现在除了现有的`debug_mask_default`外还有三个新可调参数，每个都有记录的范围和匹配的sysctl。

在下一节中，我们将从驱动程序公开什么转向驱动程序如何演进。今天有效的配置旋钮必须在驱动程序更改时明天仍然有效。版本控制规范是保持该承诺的方式。

## 第4节：版本控制与兼容性策略

第四个规范是演进而不破坏的规范。`myfirst`驱动程序公开的每个公共接口，`/dev/myfirst0`节点、ioctl接口、sysctl树、可调参数集，都是与某人的契约。静默更改其中任何一个含义的更改是破坏性更改，躲过开发者注意的破坏性更改是真实世界驱动程序错误 disproportionate 来源。本节教授如何有意识地版本化驱动程序的公共接口，以便更改对调用者可见，旧调用者在驱动程序添加新特性时继续工作。

本章为`myfirst`驱动程序使用三个不同的版本号。每个都有特定用途。混淆它们是值得在扎根之前避免的混淆来源。

### 三个版本号

`myfirst`驱动程序有三个版本标识符，在第23、24和25章中引入。每个位于不同位置，因不同原因更改。

第一个是**人类可读发布字符串**。对于`myfirst`，这是`MYFIRST_VERSION`，在`myfirst_sysctl.c`中定义并通过`dev.myfirst.0.version` sysctl公开。其当前值为`"1.8-maintenance"`。发布字符串是给人类的：运行`sysctl dev.myfirst.0.version`的操作员看到一个简短标签，标识驱动程序历史的这个特定检查点。发布字符串不由程序解析；由人阅读。它在驱动程序达到作者想要标记的新里程碑时更改，在本书中是在每章结束时。

第二个是**内核模块版本整数**。这是`MODULE_VERSION(myfirst, N)`，其中`N`是内核依赖机制使用的整数。声明`MODULE_DEPEND(other, myfirst, 1, 18, 18)`的另一个模块要求`myfirst`存在且版本为18或以上（且小于或等于18，在此声明中意味着恰好18）。模块版本整数仅当模块的内核内调用者需要重新编译时更改，例如共享符号的签名更改时。对于不公开公共内核符号的驱动程序（如`myfirst`），模块版本号主要是象征性的；本章在每个里程碑提升它以保持读者心智模型与三个版本标识符一致。

第三个是**ioctl接口版本整数**。对于`myfirst`，这是`myfirst_ioctl.h`中的`MYFIRST_IOCTL_VERSION`。其当前值为1。当ioctl头文件以针对前一版本编译的旧用户空间程序会误解的方式更改时，ioctl接口版本更改。重新编号的ioctl命令、更改的有效负载布局、更改的现有ioctl语义：这些中的每一个都是对ioctl接口的破坏性更改，必须提升版本。添加新ioctl命令、在不重新解释现有字段的情况下在有效负载末尾扩展字段、添加不影响旧命令的特性：这些是兼容更改，不需要提升。

一个简单的经验法则保持三者分离。发布字符串是操作员读取的。模块版本整数是其他模块检查的。ioctl版本整数是用户空间程序检查的。每个按自己的时间表移动。

### 为什么用户需要查询版本

通过ioctl与驱动程序通信的用户空间程序有一个问题。头文件`myfirst_ioctl.h`定义了一组命令、布局和版本1语义。驱动程序的新版本可能添加命令、更改布局或更改语义。当用户空间程序在具有比编译时所用版本更新或更旧的驱动程序的系统上运行时，除非询问，否则无法知道驱动程序的实际版本。

解决方案是目的仅为返回驱动程序ioctl版本的ioctl。`myfirst`驱动程序已经有一个：`MYFIRSTIOC_GETVER`，定义为`_IOR('M', 1, uint32_t)`。用户空间程序在打开设备后立即调用此ioctl，将返回的版本与编译时版本比较，并决定是否可以安全继续。

用户空间的模式：

```c
#include "myfirst_ioctl.h"

int fd = open("/dev/myfirst0", O_RDWR);
uint32_t ver;
if (ioctl(fd, MYFIRSTIOC_GETVER, &ver) < 0)
        err(1, "getver");
if (ver != MYFIRST_IOCTL_VERSION)
        errx(1, "driver version %u, tool expects %u",
            ver, MYFIRST_IOCTL_VERSION);
```

如果版本不匹配，工具拒绝运行。这是一种可能的策略。更宽容的策略可能允许工具针对更新的驱动程序运行（如果驱动程序的新ioctl是旧的ioctl的超集），并允许工具针对旧驱动程序运行（如果工具可以回退到旧的命令集）。更严格的策略可能要求精确匹配。工具的作者根据值得花多少精力在向后兼容性上在其中选择。

### 在不破坏旧调用者的情况下添加新ioctl

常见情况是向驱动程序添加新特性，这通常意味着添加新ioctl。只要遵循两条规则，规范就是直接的。

首先，**不要重用现有的ioctl号**。每个ioctl命令都有由`_IO`、`_IOR`、`_IOW`或`_IOWR`编码的唯一`(magic, number)`对。`myfirst_ioctl.h`中的当前分配：

```c
#define MYFIRSTIOC_GETVER   _IOR('M', 1, uint32_t)
#define MYFIRSTIOC_GETMSG   _IOR('M', 2, char[MYFIRST_MSG_MAX])
#define MYFIRSTIOC_SETMSG   _IOW('M', 3, char[MYFIRST_MSG_MAX])
#define MYFIRSTIOC_RESET    _IO('M', 4)
```

新ioctl在同一魔术字母下取下一个可用号：`MYFIRSTIOC_GETCAPS = _IOR('M', 5, uint32_t)`。号5以前未使用，不能与旧程序编译的二进制冲突。针对没有`GETCAPS`的版本编译的旧程序根本不发送该ioctl，因此旧程序不受添加影响。

其次，**对于纯添加不要提升`MYFIRST_IOCTL_VERSION`**。不更改旧命令含义的新ioctl是兼容更改。从未听说过新ioctl的旧用户空间程序仍然说相同的语言；版本整数应该保持不变。为每次添加提升版本会强制每个调用者在驱动程序获得新命令时重建，这违背了版本控制的目的。

用不同语义替换现有命令的新ioctl确实需要提升。如果驱动程序添加具有新布局的`MYFIRSTIOC_SETMSG_V2`并弃用`MYFIRSTIOC_SETMSG`，调用已弃用命令的旧程序会看到更改的行为（驱动程序可能返回`ENOIOCTL`或行为不同）。这是破坏性更改，提升信号它。

### 弃用过时的ioctl

弃用是礼貌管理的移除形式。当命令要被移除时，驱动程序宣布意图，在过渡期保持命令工作，并在后续版本中移除它。典型的弃用序列：

- 版本N：在`MAINTENANCE.md`中宣布弃用。命令仍然工作。
- 版本N+1：命令工作但每次使用时记录限速警告。用户看到警告并知道要迁移。
- 版本N+2：命令返回`EOPNOTSUPP`并记录限速错误。大多数用户已迁移；少数未迁移的被强制迁移。
- 版本N+3：命令从头文件移除。仍然引用它的程序不再编译。

过渡期应该以发布次数（通常一到两个主要版本）而不是日历时间衡量。保持弃用契约可预测的驱动程序为消费者提供稳定的目标。

对于本章中的`myfirst`，尚无命令被弃用。本章为将来引入该模式。同样的规范适用于sysctl树：OID处理程序中的限速警告告诉操作员名称即将消失，`MAINTENANCE.md`中的注释记录计划的移除日期。

### 能力位掩码模式

对于跨多个发布演进的驱动程序，单个版本整数告诉调用者他们正在与哪个版本对话，但不告诉该版本支持哪些具体特性。特性丰富的驱动程序受益于更细粒度的机制：能力位掩码。

想法很简单。驱动程序在`myfirst_ioctl.h`中定义一组能力位：

```c
#define MYF_CAP_RESET       (1U << 0)
#define MYF_CAP_GETMSG      (1U << 1)
#define MYF_CAP_SETMSG      (1U << 2)
#define MYF_CAP_TIMEOUT     (1U << 3)
#define MYF_CAP_MAXRETRIES  (1U << 4)
```

新ioctl `MYFIRSTIOC_GETCAPS`返回一个`uint32_t`，设置了此驱动程序实际支持特性的位：

```c
#define MYFIRSTIOC_GETCAPS  _IOR('M', 5, uint32_t)
```

在内核中：

```c
case MYFIRSTIOC_GETCAPS:
        *(uint32_t *)data = MYF_CAP_RESET | MYF_CAP_GETMSG |
            MYF_CAP_SETMSG;
        break;
```

在用户空间：

```c
uint32_t caps;
ioctl(fd, MYFIRSTIOC_GETCAPS, &caps);
if (caps & MYF_CAP_TIMEOUT)
        set_timeout(fd, 60);
else
        warnx("driver does not support timeout configuration");
```

能力位掩码允许用户空间程序无需试错即可发现特性。如果调用者想知道特性是否存在，它检查位；如果位已设置，调用者知道驱动程序支持该特性和相关ioctl。未定义该位的旧驱动程序不会假装支持从未听说过的特性。

该模式随着驱动程序增长而扩展良好。每个发布为新特性添加新位。已弃用的特性保留其位作为未使用；为新模式回收位会是破坏性更改。位掩码本身是`uint32_t`，在需要添加第二个字之前给驱动程序32个特性。如果驱动程序达到32个特性，添加第二个字是兼容更改（新位在新字段中，因此只读取第一个字的旧程序看到相同的位）。

第25章将`MYFIRSTIOC_GETCAPS`添加到`myfirst`驱动程序，设置了三个位：`MYF_CAP_RESET`、`MYF_CAP_GETMSG`和`MYF_CAP_SETMSG`。`myfirstctl`用户空间程序被扩展以在启动时查询能力并拒绝调用不支持的特性。

### Sysctl弃用

FreeBSD不在sysctl树上提供专用的`CTLFLAG_DEPRECATED`标志。相关标志`CTLFLAG_SKIP`在`/usr/src/sys/sys/sysctl.h`中定义，将OID隐藏在默认列表之外（如果显式命名仍可读），但它主要用于非弃用目的。因此，礼貌地弃用sysctl OID的方法是用一个既做预期工作又*在*前几次触碰OID时记录限速警告的处理程序替换它。

```c
static int
myfirst_sysctl_old_counter(SYSCTL_HANDLER_ARGS)
{
        struct myfirst_softc *sc = arg1;

        DLOG_RL(sc, &sc->sc_rl_deprecated, MYF_RL_DEFAULT_PPS,
            "sysctl dev.myfirst.%d.old_counter is deprecated; "
            "use new_counter instead\n",
            device_get_unit(sc->sc_dev));
        return (sysctl_handle_int(oidp, &sc->sc_old_counter, 0, req));
}
```

操作员在前几次读取OID时在`dmesg`中看到警告，这是迁移的强烈提示。sysctl仍然工作，因此显式引用它的脚本在过渡期间不会破坏。在一两个发布后，OID本身被移除。`MAINTENANCE.md`中的注释记录意图和目标发布。

对于`myfirst`，尚无sysctl被弃用。第25章驱动程序在文档中引入该模式并保留供将来使用。

### 用户可见行为更改

并非每个破坏性更改都是重命名或重新编号。有时驱动程序保持相同的ioctl、相同的sysctl、相同的可调参数，并静默更改操作做什么。以前只归零计数器但现在也清除消息的`MYFIRSTIOC_RESET`是行为更改。以前报告总写入字节数但现在报告千字节数的sysctl是行为更改。以前是绝对值现在是乘数的可调参数是行为更改。

行为更改是最难捕捉的破坏性更改，因为它们不会出现在头文件差异或sysctl列表中。规范是在`MAINTENANCE.md`的"变更日志"部分下记录每个行为更改，当ioctl语义更改时提升ioctl接口版本整数，并在描述字符串本身中宣布sysctl语义更改。

行为更改的好模式是引入新的命名命令或新sysctl而不是重新定义现有命令或sysctl。`MYFIRSTIOC_RESET`保留旧语义。`MYFIRSTIOC_RESET_ALL`是具有新语义的新命令。旧命令最终被弃用。成本是过渡期稍大的公共接口；好处是没有调用者被静默行为更改破坏。

### `MODULE_DEPEND`和模块间兼容性

`MODULE_DEPEND`宏声明一个模块依赖另一个模块并需要特定的版本范围：

```c
MODULE_DEPEND(myfirst, dependency, 1, 2, 3);
```

三个整数是`myfirst`兼容的`dependency`的最低、首选和最高版本。如果`dependency`不存在或超出范围，内核拒绝加载`myfirst`。

对于不发布内核内符号的驱动程序，`MODULE_DEPEND`最常用于依赖标准子系统模块：

```c
MODULE_DEPEND(myfirst_usb, usb, 1, 1, 1);
```

这声明`myfirst`的USB版本恰好需要版本1的USB栈。子系统模块的版本号由子系统作者管理；驱动程序作者在子系统的头文件（对于USB是`/usr/src/sys/dev/usb/usbdi.h`）或已经依赖它的另一个驱动程序中找到当前值。

对于第25章结束时的`myfirst`，不需要`MODULE_DEPEND`，因为伪驱动程序不需要子系统。当驱动程序变成USB附加版本时，第26章USB章节将添加第一个真正的`MODULE_DEPEND`。

### 工作示例：1.7到1.8过渡

第25章驱动程序在本章结束时提升三个版本标识符：

- `MYFIRST_VERSION`：从`"1.7-integration"`到`"1.8-maintenance"`。
- `MODULE_VERSION(myfirst, N)`：从17到18。
- `MYFIRST_IOCTL_VERSION`：保持为1，因为本章的ioctl添加是纯添加（新命令，无移除，无语义更改）。

`GETCAPS` ioctl以先前未使用的命令号5添加。针对第24章版本头文件编译的旧`myfirstctl`二进制文件不知道`GETCAPS`并不发送它；它们继续不变地工作。针对第25章头文件编译的新`myfirstctl`二进制文件在启动时查询`GETCAPS`并相应行为。

`MAINTENANCE.md`文档获得1.8的变更日志条目：

```text
## 1.8-maintenance

- Added MYFIRSTIOC_GETCAPS (command 5) returning a capability
  bitmask.  Compatible with all earlier user-space programs.
- Added tunables hw.myfirst.timeout_sec, hw.myfirst.max_retries,
  hw.myfirst.log_ratelimit_pps.  Each has a matching writable
  sysctl under dev.myfirst.<unit>.
- Added rate-limited logging through ppsratecheck(9).
- No breaking changes from 1.7.
```

阅读`MAINTENANCE.md`的驱动程序用户可以一眼看到更改了什么，并可以评估是否需要更新工具。不阅读`MAINTENANCE.md`的用户仍然可以在运行时查询能力并以编程方式发现新特性。

### 版本控制中的常见错误

首次应用版本控制规范时，三个错误很常见。每个都值得命名。

第一个错误是**重用ioctl号**。曾经分配并后来弃用的号保持弃用。新命令取下一个可用号，而不是已弃用命令的号。重用号静默破坏编译了旧含义的旧调用者；编译器无法检测冲突，因为已弃用命令的头文件已被移除。

第二个错误是**为每次更改提升版本整数**。如果每个补丁都提升`MYFIRST_IOCTL_VERSION`，用户空间工具必须不断重建或版本检查失败。整数应该只为真正的破坏性更改提升。纯添加保留它不变。

第三个错误是**将发布字符串视为语义版本**。发布字符串是给人类的；它可以是任何东西。模块版本整数和ioctl版本整数由程序解析，应该遵循规范（单调递增，仅为特定原因提升）。混淆两者导致令人困惑的版本号。

### 第4节总结

版本控制是演进而不破坏的规范。保持三个版本标识符独立、ioctl添加兼容、弃用宣布、能力位准确的驱动程序为其调用者在驱动程序的长寿命期间提供稳定目标。`myfirst`驱动程序现在有一个工作的`GETCAPS` ioctl、`MAINTENANCE.md`中记录的弃用策略，以及每个因自己的原因更改的三个版本标识符。未来开发者添加特性或弃用命令所需的一切都已到位。

在下一节中，我们将从驱动程序的公共接口转向其私有资源规范。在附加失败时崩溃的驱动程序是无法从任何错误中恢复的驱动程序。标签化goto模式是FreeBSD驱动程序如何使每个分配可逆的方式。


## 第5节：管理失败路径中的资源

每个附加例程都是一个有序的获取序列。它分配一个锁，创建一个cdev，在设备上挂载sysctl树，也许注册一个事件处理器或定时器，在更复杂的驱动程序中它分配总线资源、映射I/O窗口、附加中断并设置DMA。每个获取都可能失败。在失败之前成功的每个获取必须按相反顺序释放，否则内核泄漏内存、泄漏锁、泄漏cdev，最坏情况下保持设备节点活着但内部有陈旧指针。

`myfirst`驱动程序自第17章以来一直在一次一节地增长其附加路径。附加开始时很小：一个锁和一个cdev。第24章添加了sysctl树。第25章即将添加限速状态、可调参数获取的默认值和一两个计数器。这些资源获取的顺序现在对清理路径很重要。每个新获取必须知道它在展开排序中属于哪里，展开本身必须结构化以便下周添加新资源不会迫使重写附加函数。

第20章非正式地介绍了该模式；本节给它一个名称、一个词汇表和一个足够强大以存活第25章`myfirst_attach`完整形态的规范。

### 问题：嵌套`if`路径不可扩展

附加例程的朴素形态是嵌套`if`语句的阶梯。每个成功条件包含下一步。每个失败返回。问题是每个失败必须展开前几步已经完成的工作，展开代码在梯子的每一层都重复：

```c
/*
 * Naive attach.  DO NOT WRITE DRIVERS THIS WAY.  This example shows
 * how the nested-if pattern forces duplicated cleanup at every level
 * and why it becomes unmaintainable as soon as a fourth resource is
 * added to the chain.
 */
static int
myfirst_attach_bad(device_t dev)
{
	struct myfirst_softc *sc = device_get_softc(dev);
	struct make_dev_args args;
	int error;

	sc->sc_dev = dev;
	mtx_init(&sc->sc_mtx, "myfirst", NULL, MTX_DEF);

	make_dev_args_init(&args);
	args.mda_devsw   = &myfirst_cdevsw;
	args.mda_uid     = UID_ROOT;
	args.mda_gid     = GID_WHEEL;
	args.mda_mode    = 0660;
	args.mda_si_drv1 = sc;
	args.mda_unit    = device_get_unit(dev);

	error = make_dev_s(&args, &sc->sc_cdev, "myfirst%d",
	    device_get_unit(dev));
	if (error == 0) {
		myfirst_sysctl_attach(sc);
		if (myfirst_log_attach(sc) == 0) {
			/* all resources held; we succeeded */
			return (0);
		} else {
			/* log allocation failed: undo sysctl and cdev */
			/* but wait, sysctl is owned by Newbus, so skip it */
			destroy_dev(sc->sc_cdev);
			mtx_destroy(&sc->sc_mtx);
			return (ENOMEM);
		}
	} else {
		mtx_destroy(&sc->sc_mtx);
		return (error);
	}
}
```

即使在这个小示例中，展开逻辑也出现在两个不同的地方，读者必须阅读分支以知道每个点获取了哪些资源，添加第四个资源迫使另一层嵌套和另一个重复的清理块。真正的驱动程序有七到八个资源。像`/usr/src/sys/dev/e1000/if_em.c`中的`if_em`这样的驱动程序有十几个以上。嵌套`if`在那里不是选项。

嵌套模式的失败模式不是理论上的。旧FreeBSD驱动程序中的常见错误模式是在清理分支之一中缺少`mtx_destroy`或缺少`bus_release_resource`：一个分支销毁了锁，另一个忘记了。每个分支都是犯错的机会，错误只在特定失败触发时才出现，这通常意味着直到客户报告设备附加失败时出现恐慌才会被发现。

### `goto fail;`模式

FreeBSD对嵌套清理问题的答案是标签化goto模式。附加函数被写成线性的获取序列。每个可能失败的获取后面跟着一个测试，要么在成功时穿透，要么在失败时跳转到清理标签。清理标签从最多获取到最少获取排序。每个标签释放在该点持有的资源，然后穿透到下一个标签。函数以成功时单个`return (0)`和清理链底部的单个`return (error)`结束：

```c
static int
myfirst_attach(device_t dev)
{
	struct myfirst_softc *sc = device_get_softc(dev);
	struct make_dev_args args;
	int error;

	/* Resource 1: softc basics.  Cannot fail. */
	sc->sc_dev = dev;

	/* Resource 2: the lock.  Cannot fail on DEF mutex. */
	mtx_init(&sc->sc_mtx, "myfirst", NULL, MTX_DEF);

	/* Resource 3: the cdev.  Can fail. */
	make_dev_args_init(&args);
	args.mda_devsw   = &myfirst_cdevsw;
	args.mda_uid     = UID_ROOT;
	args.mda_gid     = GID_WHEEL;
	args.mda_mode    = 0660;
	args.mda_si_drv1 = sc;
	args.mda_unit    = device_get_unit(dev);

	error = make_dev_s(&args, &sc->sc_cdev, "myfirst%d",
	    device_get_unit(dev));
	if (error != 0)
		goto fail_mtx;

	/* Resource 4: the sysctl tree.  Cannot fail (Newbus owns it). */
	myfirst_sysctl_attach(sc);

	/* Resource 5: the log state.  Can fail. */
	error = myfirst_log_attach(sc);
	if (error != 0)
		goto fail_cdev;

	/* All resources held.  Announce and return. */
	DPRINTF(sc, MYF_DBG_INIT,
	    "attach: version 1.8-maintenance ready\n");
	return (0);

fail_cdev:
	destroy_dev(sc->sc_cdev);
fail_mtx:
	mtx_destroy(&sc->sc_mtx);
	return (error);
}
```

从上到下阅读函数。每一步都是资源获取。每个失败检查是一个两行块：如果获取失败，跳转到以先前获取资源命名的标签。底部的标签按相反顺序释放资源并穿透到下一个标签。最后的`return (error)`返回来自哪个获取失败的errno。

这种形状可以扩展。添加第六个资源意味着在顶部添加一个获取块、在底部添加一个`goto`目标、和一行清理代码。没有嵌套，没有重复，没有分支树。管理附加路径的相同规则管理附加路径的每个未来添加：获取、测试、跳转到前一个标签、按相反顺序释放。

### 为什么线性展开是正确的形状

标签化goto模式的价值不仅仅是风格上的。它直接映射到附加序列是资源栈的结构属性，清理是该栈上的弹出操作。

栈有三个易于陈述和易于违反的属性。首先，资源按获取的相反顺序释放。其次，失败的获取不会将资源添加到栈中，因此清理从前一个获取的资源开始，而不是从刚刚失败的获取开始。第三，栈上的每个资源恰好释放一次：不是零次，不是两次。

这些属性中的每一个在`goto fail;`模式中都有可见的关联。清理标签在文件中以与获取相反的顺序出现：最后获取的清理标签在清理链顶部。失败的获取跳转到以前一个获取命名的标签，而不是以自身命名；标签的名称字面上就是现在必须撤销的资源名称。因为每个标签穿透到下一个，每个资源在恰好一个标签中出现，所以每个资源在每个失败路径上恰好释放一次。

栈规范是使模式健壮的原因。如果读者想要审计清理路径的正确性，他们不必阅读分支。他们只需计算标签数、计算获取数并比较。

### 标签命名约定

FreeBSD驱动程序中的标签传统上以`fail_`开头，后跟即将撤销的资源名称。资源名称与softc中字段的名称或调用以获取它的函数名称匹配。整个树中常见的模式：

- `fail_mtx`撤销`mtx_init`
- `fail_sx`撤销`sx_init`
- `fail_cdev`撤销`make_dev_s`
- `fail_ires`撤销用于IRQ的`bus_alloc_resource`
- `fail_mres`撤销用于内存窗口的`bus_alloc_resource`
- `fail_intr`撤销`bus_setup_intr`
- `fail_dma_tag`撤销`bus_dma_tag_create`
- `fail_log`撤销驱动程序私有的分配（`myfirst`中的限速块）

一些旧驱动程序使用编号标签（`fail1`、`fail2`、`fail3`）。编号标签是合法的但较差：在序列中间添加资源迫使重新编号插入点之后的每个标签，标签号不告诉读者哪个资源正在被清理。命名标签优雅地经受插入并自我文档化。

无论驱动程序选择什么约定，它应该在所有文件中保持一致。`myfirst`从本章起每个附加函数都使用`fail_<resource>`约定。

### 穿透规则

每个清理链必须遵守的单一规则是每个清理标签穿透到下一个。链中间的散落`return`或缺少标签跳过了本应释放的资源的清理。编译器不会警告任一错误。

考虑如果开发者编辑清理链并意外写入以下内容会发生什么：

```c
fail_cdev:
	destroy_dev(sc->sc_cdev);
	return (error);          /* BUG: skips mtx_destroy. */
fail_mtx:
	mtx_destroy(&sc->sc_mtx);
	return (error);
```

第一个`return`阻止`mtx_destroy`在`fail_cdev`路径上运行。锁被泄漏。内核的见证代码不会抱怨，因为泄漏的锁永远不会再次获取。泄漏持续到机器重启。在正常操作中不可见，只在驱动程序反复附加和失败的系统（例如热插拔设备）上显示为缓慢的内存膨胀。

防止此类错误的方法是在底部写入带有单个`return`的清理链，没有中间返回。中间的标签仅包含其资源的清理调用。穿透是默认和预期的行为：

```c
fail_cdev:
	destroy_dev(sc->sc_cdev);
fail_mtx:
	mtx_destroy(&sc->sc_mtx);
	return (error);
```

审计链的读者将其读作一个简单列表：销毁cdev、销毁锁、返回。没有分支需要跟踪，添加标签意味着添加单行清理代码和可选的单个新目标。

### 成功路径看起来像什么

附加函数以单个`return (0)`成功，紧接在第一个清理标签之前。这是每个获取都成功且不需要清理的点。`return (0)`在视觉上将获取链与清理链分开：它上面的所有内容都是获取，它下面的所有内容都是清理。

一些驱动程序忘记这种分离，从最后一个获取穿透到第一个清理标签，释放它们刚获取的资源。散落缺少`return (0)`是产生此错误的最简单方式：

```c
	/* Resource N: the final acquisition. */
	...

	/* Forgot to put a return here. */

fail_cdev:
	destroy_dev(sc->sc_cdev);
```

没有`return (0)`，控制在每次成功附加后穿透到`fail_cdev`，在成功路径上销毁cdev。驱动程序然后报告附加失败，因为`error`为零且内核看到成功返回，但它刚创建的cdev已消失。结果是一个出现几秒后就消失的设备节点。调试这需要注意到附加消息打印了但设备不响应；在繁忙日志中这不是一个容易发现的错误。

防御是规范。每个附加函数以其获取链结束，在独立一行上有`return (0);`，后跟一个空行，再后跟清理标签。没有例外。像`igor`这样的代码检查器或审查者的眼睛在形状始终相同时快速发现违规。

### 当获取不能失败时

一些获取不能失败。默认风格互斥锁的`mtx_init`不能返回错误。`sx_init`不能。`callout_init_mtx`不能。`SYSCTL_ADD_*`调用不能返回驱动程序被期望检查的错误（那里的失败是内核内部问题，不是驱动程序问题）。

对于不能失败的获取，没有goto。获取后面跟着下一步，没有测试。获取的清理标签仍然是必需的，因为如果后面的获取失败，清理链必须释放资源：

```c
	mtx_init(&sc->sc_mtx, "myfirst", NULL, MTX_DEF);

	error = make_dev_s(&args, &sc->sc_cdev, ...);
	if (error != 0)
		goto fail_mtx;       /* undoes the lock. */
```

`fail_mtx`存在，即使`mtx_init`本身没有失败路径，因为如果它下面的任何东西失败，锁仍然需要被销毁。

该模式成立：每个获取的资源都有一个标签，无论其获取是否可以失败。

### 减少重复的辅助函数

当几个获取共享相同形状（分配、检查、出错时goto）时，将它们隐藏在辅助函数后面很诱人。辅助函数的工作是整合获取和检查；调用者只看到单个`if (error != 0) goto fail_X;`行。只要辅助函数遵循相同的规范，这就可以：失败时它不释放部分获取的东西，它返回有意义的errno以便调用者的goto目标可以依赖它。

在`myfirst`中，第5节的配套示例引入了一个名为`myfirst_log_attach`的辅助函数，它分配限速状态、初始化其字段，并在成功时返回0或在失败时返回非零errno。附加函数用一行调用它：

```c
	error = myfirst_log_attach(sc);
	if (error != 0)
		goto fail_cdev;
```

辅助函数内部遵循相同的模式。如果它分配两个资源而第二个失败，辅助函数在返回之前展开第一个。调用者将辅助函数视为单个原子获取：它要么完全成功，要么完全失败，调用者永远不必担心辅助函数的中间状态。

然而，过于急切简化的辅助函数会破坏模式。分配资源并将其存储到softc中的辅助函数是可以的。分配资源、存储到softc中，并且在出错时也释放它的辅助函数是不行的：调用者的清理标签也会尝试释放它，导致双重释放。规则是获取辅助函数要么成功并将资源留在softc中，要么失败并使softc不变。它们不会半成功。

### 分离作为附加的镜像

分离例程是成功附加的清理链。它必须释放附加获取的完全相同的资源，按相反顺序。分离函数的形状是移除标签和删除获取的清理链：

```c
static int
myfirst_detach(device_t dev)
{
	struct myfirst_softc *sc = device_get_softc(dev);

	/* Check for busy first. */
	mtx_lock(&sc->sc_mtx);
	if (sc->sc_open_count > 0) {
		mtx_unlock(&sc->sc_mtx);
		return (EBUSY);
	}
	mtx_unlock(&sc->sc_mtx);

	/* Release resources in the reverse order of attach. */
	myfirst_log_detach(sc);
	destroy_dev(sc->sc_cdev);
	/* Sysctl is cleaned up by Newbus after detach returns. */
	mtx_destroy(&sc->sc_mtx);

	return (0);
}
```

与附加函数并排阅读，对应关系是精确的。附加中命名的每个资源在分离中都有释放。添加到附加的每个新获取在分离中有匹配的添加。审计向驱动程序添加新资源的补丁的审查者应该能够在差异中找到两个添加，一个在附加链中，一个在分离链中；仅添加到附加的差异是不完整的。

在触碰附加链时有用的规范是在相邻编辑器缓冲区中打开分离函数，并在添加获取后立即添加释放。这是确保两个函数保持同步的最简单方式：它们作为单个操作一起编辑。

### 用于测试的故意失败注入

清理链只有当每个标签可达时才是正确的。确保的唯一方式是故意触发每个失败路径并观察驱动程序之后干净地卸载。等待真实硬件失败来锻炼这些路径不是策略：大多数路径在现实生活中从未被锻炼。

用于此类测试的工具是故意失败注入。开发者在附加链中间添加临时`goto`或临时早期返回，并确认当注入失败触发时驱动程序的资源全部释放。

`myfirst`的最小模式：

```c
#ifdef MYFIRST_DEBUG_INJECT_FAIL_CDEV
	error = ENOMEM;
	goto fail_cdev;
#endif
```

用`-DMYFIRST_DEBUG_INJECT_FAIL_CDEV`编译驱动程序并加载它。附加返回`ENOMEM`。`kldstat`显示没有残留。`dmesg`显示附加失败，没有内核关于泄漏锁或泄漏资源的抱怨。卸载模块，移除定义，重新编译，驱动程序恢复正常。

依次为每个标签做一次：

1. 在锁初始化之后注入失败。确认只释放了锁。
2. 在cdev创建之后注入失败。确认cdev和锁被释放。
3. 在sysctl树构建之后注入失败。确认cdev和锁被释放，sysctl OID消失。
4. 在日志状态初始化之后注入失败。确认到该点获取的每个资源都被释放。

如果任何注入留下残留，清理链有错误。修复错误，重新运行注入，继续。

这第一次令人不舒服，之后令人安心。每个失败路径都被锻炼过一次的驱动程序是其失败路径随着代码演进将继续工作的驱动程序。从未锻炼过失败路径的驱动程序是有潜在错误的驱动程序，这些错误会在最糟糕的时刻出现。

配套示例`ex05-failure-injection/`位于`examples/part-05/ch25-advanced/`下，包含一个`myfirst_attach`版本，每个失败注入站点由注释的`#define`标记。本章末尾的实验室依次演练每个注入。

### 第5节完整的`myfirst_attach`

将第5节的所有内容与第25章的添加（日志状态、可调参数获取、能力位掩码）放在一起，最终附加函数如下所示：

```c
static int
myfirst_attach(device_t dev)
{
	struct myfirst_softc *sc = device_get_softc(dev);
	struct make_dev_args args;
	int error;

	/*
	 * Stage 1: softc basics.  Cannot fail.  Recorded for consistency;
	 * no cleanup label is needed because no resource is held yet.
	 */
	sc->sc_dev = dev;

	/*
	 * Stage 2: lock.  Cannot fail on MTX_DEF, but needs a label
	 * because anything below this line can fail and must release it.
	 */
	mtx_init(&sc->sc_mtx, "myfirst", NULL, MTX_DEF);

	/*
	 * Stage 3: pre-populate the softc with defaults, then allow
	 * boot-time tunables to override.  No allocations here, so no
	 * cleanup is needed.  Defaults come from the Section 3 tunable
	 * set.
	 */
	strlcpy(sc->sc_msg, "Hello from myfirst", sizeof(sc->sc_msg));
	sc->sc_msglen = strlen(sc->sc_msg);
	sc->sc_open_count = 0;
	sc->sc_total_reads = 0;
	sc->sc_total_writes = 0;
	sc->sc_debug = 0;
	sc->sc_timeout_sec = 5;
	sc->sc_max_retries = 3;
	sc->sc_log_pps = MYF_RL_DEFAULT_PPS;

	TUNABLE_INT_FETCH("hw.myfirst.debug_mask_default",
	    &sc->sc_debug);
	TUNABLE_INT_FETCH("hw.myfirst.timeout_sec",
	    &sc->sc_timeout_sec);
	TUNABLE_INT_FETCH("hw.myfirst.max_retries",
	    &sc->sc_max_retries);
	TUNABLE_INT_FETCH("hw.myfirst.log_ratelimit_pps",
	    &sc->sc_log_pps);

	/*
	 * Stage 4: cdev.  Can fail.  On failure, release the lock and
	 * return the error from make_dev_s.
	 */
	make_dev_args_init(&args);
	args.mda_devsw   = &myfirst_cdevsw;
	args.mda_uid     = UID_ROOT;
	args.mda_gid     = GID_WHEEL;
	args.mda_mode    = 0660;
	args.mda_si_drv1 = sc;
	args.mda_unit    = device_get_unit(dev);

	error = make_dev_s(&args, &sc->sc_cdev, "myfirst%d",
	    device_get_unit(dev));
	if (error != 0)
		goto fail_mtx;

	/*
	 * Stage 5: sysctl tree.  Cannot fail.  The framework owns the
	 * context, so no cleanup label is required specifically for it.
	 */
	myfirst_sysctl_attach(sc);

	/*
	 * Stage 6: rate-limit and counter state.  Can fail if memory
	 * allocation fails.  On failure, release the cdev and the lock.
	 */
	error = myfirst_log_attach(sc);
	if (error != 0)
		goto fail_cdev;

	DPRINTF(sc, MYF_DBG_INIT,
	    "attach: version 1.8-maintenance complete\n");
	return (0);

fail_cdev:
	destroy_dev(sc->sc_cdev);
fail_mtx:
	mtx_destroy(&sc->sc_mtx);
	return (error);
}
```

每个资源都已记录。每个失败路径都是线性的。函数在从获取到清理的过渡处有单个成功返回，在清理链底部有单个失败返回。下一章添加第七个资源是三行操作：一个新获取块、一个新标签、一行新清理代码。

### 失败路径中的常见错误

一些失败路径错误值得一次性命名，以便在别人的代码或审查中出现时可以识别。

第一个错误是**缺少标签**。开发者添加新的资源获取但忘记添加其清理标签。编译器不会警告；链从外部看起来很好；但在新获取之后的失败时，下面所有的清理都被跳过。规则是每个获取都有一个标签。即使获取不能失败，它仍然需要一个标签，以便后面的获取可以到达它。

第二个错误是**双重释放资源**。开发者在辅助函数中添加了本地清理，忘记调用者的清理标签也会释放资源。辅助函数释放一次，调用者再释放一次，内核要么恐慌（对于内存），要么见证代码抱怨（对于锁）。规则是只有一方拥有每个资源的清理。如果辅助函数获取资源并将其存储到softc中，辅助函数不会代替调用者清理它；它要么成功，要么使softc不变。

第三个错误是**依赖`NULL`测试**。开发者写这样的清理链：

```c
fail_cdev:
	if (sc->sc_cdev != NULL)
		destroy_dev(sc->sc_cdev);
fail_mtx:
	if (sc->sc_mtx_initialised)
		mtx_destroy(&sc->sc_mtx);
```

逻辑是：如果资源未实际获取则跳过清理。意图是防御性的；效果是隐藏错误。如果`NULL`检查存在是因为清理可能在资源未获取的状态下到达，链是错误的：goto目标应该是不同的标签。正确的行为是使清理标签在资源实际获取之前不可达。在任一状态下都可达的标签是获取顺序混乱的症状，`NULL`检查只是掩盖了它。

第四个错误是**将`goto`用于非错误流**。附加函数中的`goto`严格用于失败路径。在某些非错误条件下跳过获取链的一部分的`goto`违反了线性清理不变式：清理链假设每个标签对应一个已获取的资源，绕过获取的`goto`打破该假设。如果需要条件获取，在获取本身周围使用`if`，而不是在获取周围使用`goto`。

### 第5节总结

附加和分离是将驱动程序固定到内核的接缝。正确的附加是线性的获取栈；正确的分离是反向弹出的栈。标签化goto模式是FreeBSD驱动程序如何在不购买其他操作系统机制（C++析构函数、Go defer、Rust Drop）的情况下在C中编码该栈。它不迷人，但可以扩展：有十几个资源的驱动程序读起来和有两个的完全一样，添加新资源的规则始终相同。

`myfirst`附加函数现在有四个失败标签，以及获取、成功返回和清理的干净分离。第26章添加的每个新资源都将适合这种形状。

在下一节中，我们将从任何单个函数退后一步，看看增长的驱动程序如何跨文件分布。包含每个函数的`myfirst.c`已经承载了我们八章；是时候将其拆分为聚焦单元，以便驱动程序的结构在文件级别可见。


## 第6节：模块化与关注点分离

到第24章结束时，`myfirst`驱动程序已经超出了单个源文件可以舒适容纳的范围。文件形态是`myfirst.c`加上`myfirst_debug.c`、`myfirst_ioctl.c`和`myfirst_sysctl.c`；`myfirst.c`仍然承载cdevsw、读/写回调、打开/关闭回调、附加和分离例程以及模块粘合代码。那对教学来说没问题，因为每次添加都落在小到读者可以记住的文件中。但对于具有ioctl接口、sysctl树、调试框架、限速日志辅助函数、能力位掩码、版本控制规范和标签化清理附加例程的驱动程序来说，已经不再合适了。包含那么多内容的文件变得难以阅读、难以比较、难以交给新贡献者。

第6节是关于另一个方向。它不引入新行为；第5节结束时存在的每个函数在第6节结束时仍然存在。改变的是文件布局和各部分之间的边界线。目标是驱动程序的结构可以从`ls`理解，其单个文件各自回答一个问题。

### 为什么要拆分文件

自包含驱动程序的诱惑是将所有东西保持在一个文件中。单个`myfirst.c`容易定位、容易搜索、容易复制到tarball。拆分感觉像官僚主义。当驱动程序跨过三个阈值之一时，拆分的论据就出现了。

第一个阈值是**理解**。打开`myfirst.c`的读者应该能在几秒钟内找到他们要找的东西。一个有八个不相关职责的1200行文件很难导航；读者必须滚过cdevsw找到sysctl，滚过sysctl找到ioctl，滚过ioctl找到附加例程。每次切换主题时，他们必须重新加载心智上下文。有了单独的文件，主题就是文件名：`myfirst_ioctl.c`是关于ioctl的，`myfirst_sysctl.c`是关于sysctl的，`myfirst.c`是关于生命周期的。

第二个阈值是**独立性**。两个不相关的更改不应修改同一文件。当一个开发者添加sysctl而另一个开发者添加ioctl时，他们的补丁不应竞争`myfirst.c`的相同行。小型、聚焦的文件允许两个更改并行落地，没有合并冲突，一个更改中的错误意外触碰另一个的风险。

第三个阈值是**可测试性和可重用性**。驱动程序的日志基础设施、其ioctl分发和其sysctl树通常对同一项目中的多个驱动程序有用。将它们保持在具有干净接口的单独文件中使它们成为以后共享的候选。位于单个文件中的驱动程序无法轻松共享任何东西；提取意味着复制和手动重命名，这是一个容易出错的操作。

第25章结束时的`myfirst`已经跨过了所有三个阈值。拆分文件是保持驱动程序健康以应对接下来十章的维护行为。

### `myfirst`的文件布局

建议的布局是最终第25章示例目录中Makefile使用的布局：

```text
myfirst.h          - 公共类型和常量（softc、SRB、状态位）。
myfirst.c          - 模块粘合、cdevsw、devclass、模块事件。
myfirst_bus.c      - Newbus方法和device_identify。
myfirst_cdev.c     - 打开/关闭/读取/写入回调；无ioctl。
myfirst_ioctl.h    - ioctl命令号和有效负载结构。
myfirst_ioctl.c    - myfirst_ioctl开关和辅助函数。
myfirst_sysctl.c   - myfirst_sysctl_attach和处理程序。
myfirst_debug.h    - DPRINTF/DLOG/DLOG_RL宏和类位。
myfirst_debug.c    - 调试类枚举（如果有任何非内联的）。
myfirst_log.h      - 限速状态结构。
myfirst_log.c      - myfirst_log_attach/detach和辅助函数。
```

七个`.c`文件和四个`.h`文件。每个`.c`文件有一个由文件名命名的主题。头文件声明跨文件边界的接口。没有文件导入另一个文件的内部；每个跨文件引用都通过头文件。

乍看之下，这似乎比驱动程序需要的文件多。其实不然。每个文件都有特定的职责，与其配套的头文件是几十行声明。累积大小与单文件版本相同；结构明显更清晰。

### 单一职责规则

管理拆分的规则是单一职责规则：每个文件回答关于驱动程序的一个问题。

- `myfirst.c`回答：这个模块如何附加到内核并将其各部分连接起来？
- `myfirst_bus.c`回答：Newbus如何发现和实例化我的驱动程序？
- `myfirst_cdev.c`回答：驱动程序如何服务打开/关闭/读取/写入？
- `myfirst_ioctl.c`回答：驱动程序如何处理其头文件声明的命令？
- `myfirst_sysctl.c`回答：驱动程序如何向`sysctl(8)`公开其状态？
- `myfirst_debug.c`回答：调试消息如何分类和限速？
- `myfirst_log.c`回答：限速状态如何初始化和释放？

更改是否属于给定文件的测试是答案测试。如果更改不回答文件的问题，它属于其他地方。新sysctl不属于`myfirst_ioctl.c`；新ioctl不属于`myfirst_sysctl.c`；新读回调变体不属于`myfirst.c`。规则是显式的，应用它的审查者拒绝将东西放在错误文件中的补丁。

将规则应用于现有第24章形态得到第25章形态。

### 公共与私有头文件

头文件承载文件之间的接口。拆分`.c`文件的驱动程序必须为每个声明决定它是属于公共头文件还是私有头文件。

**公共头文件**包含对多个`.c`文件可见的类型和常量。`myfirst.h`是驱动程序的主要公共头文件。它声明：

- `struct myfirst_softc`定义（每个`.c`文件都需要它）。
- 在多个文件中出现的常量（调试类位、softc字段大小）。
- 跨文件边界调用的函数的原型（`myfirst_sysctl_attach`、`myfirst_log_attach`、`myfirst_log_ratelimited_printf`、`myfirst_ioctl`）。

**私有头文件**承载仅一个`.c`文件需要的声明。`myfirst_ioctl.h`是规范示例。它声明命令号和有效负载结构；`myfirst_ioctl.c`和用户空间调用者需要它们，但没有其他内核内文件需要。将它们放在`myfirst.h`中会将线格式泄漏到每个翻译单元。

区别很重要，因为每个公共声明都是驱动程序必须遵守的契约。`myfirst.h`中改变大小的类型会破坏包含`myfirst.h`的每个文件。`myfirst_ioctl.h`中改变大小的类型只破坏`myfirst_ioctl.c`和针对它编译的用户空间工具。

对于第25章结束时的`myfirst`，公共头文件`myfirst.h`如下所示（裁剪到与本节相关的声明）：

```c
/*
 * myfirst.h - public types and constants for the myfirst driver.
 *
 * Types and prototypes declared here are visible to every .c file in
 * the driver.  Keep this header small.  Wire-format declarations live
 * in myfirst_ioctl.h.  Debug macros live in myfirst_debug.h.  Rate-
 * limit state lives in myfirst_log.h.
 */

#ifndef _MYFIRST_H_
#define _MYFIRST_H_

#include <sys/types.h>
#include <sys/lock.h>
#include <sys/mutex.h>
#include <sys/conf.h>

#include "myfirst_log.h"

struct myfirst_softc {
	device_t       sc_dev;
	struct mtx     sc_mtx;
	struct cdev   *sc_cdev;

	char           sc_msg[MYFIRST_MSG_MAX];
	size_t         sc_msglen;

	u_int          sc_open_count;
	u_int          sc_total_reads;
	u_int          sc_total_writes;

	u_int          sc_debug;
	u_int          sc_timeout_sec;
	u_int          sc_max_retries;
	u_int          sc_log_pps;

	struct myfirst_ratelimit sc_rl_generic;
	struct myfirst_ratelimit sc_rl_io;
	struct myfirst_ratelimit sc_rl_intr;
};

#define MYFIRST_MSG_MAX  256

/* Sysctl tree. */
void myfirst_sysctl_attach(struct myfirst_softc *);

/* Rate-limit state. */
int  myfirst_log_attach(struct myfirst_softc *);
void myfirst_log_detach(struct myfirst_softc *);

/* Ioctl dispatch. */
struct thread;
int  myfirst_ioctl(struct cdev *, u_long, caddr_t, int, struct thread *);

#endif /* _MYFIRST_H_ */
```

`myfirst.h`中没有任何内容引用线格式常量、调试类位或限速结构内部。softc按值包含三个限速字段，所以`myfirst.h`必须包含`myfirst_log.h`，但`struct myfirst_ratelimit`的内部位于`myfirst_log.h`中，不在这里公开。

### 拆分后`myfirst.c`的剖析

拆分后的`myfirst.c`是驱动程序中最短的`.c`文件。它包含cdevsw表、模块事件处理器、设备类声明和附加/分离例程。所有其他职责都已移到别处：

```c
/*
 * myfirst.c - module glue and cdev wiring for the myfirst driver.
 *
 * This file owns the cdevsw table, the devclass, the attach and
 * detach routines, and the MODULE_VERSION declaration.  The cdev
 * callbacks themselves live in myfirst_cdev.c.  The ioctl dispatch
 * lives in myfirst_ioctl.c.  The sysctl tree lives in
 * myfirst_sysctl.c.  The rate-limit infrastructure lives in
 * myfirst_log.c.
 */

#include <sys/param.h>
#include <sys/systm.h>
#include <sys/conf.h>
#include <sys/kernel.h>
#include <sys/lock.h>
#include <sys/module.h>
#include <sys/mutex.h>

#include "myfirst.h"
#include "myfirst_debug.h"
#include "myfirst_ioctl.h"

MODULE_VERSION(myfirst, 18);

extern d_open_t    myfirst_open;
extern d_close_t   myfirst_close;
extern d_read_t    myfirst_read;
extern d_write_t   myfirst_write;

struct cdevsw myfirst_cdevsw = {
	.d_version = D_VERSION,
	.d_name    = "myfirst",
	.d_open    = myfirst_open,
	.d_close   = myfirst_close,
	.d_read    = myfirst_read,
	.d_write   = myfirst_write,
	.d_ioctl   = myfirst_ioctl,
};

static int
myfirst_attach(device_t dev)
{
	/* Section 5's labelled-cleanup attach goes here. */
	...
}

static int
myfirst_detach(device_t dev)
{
	/* Section 5's mirror-of-attach detach goes here. */
	...
}

static device_method_t myfirst_methods[] = {
	DEVMETHOD(device_probe,   myfirst_probe),
	DEVMETHOD(device_attach,  myfirst_attach),
	DEVMETHOD(device_detach,  myfirst_detach),
	DEVMETHOD_END
};

static driver_t myfirst_driver = {
	"myfirst",
	myfirst_methods,
	sizeof(struct myfirst_softc),
};

DRIVER_MODULE(myfirst, nexus, myfirst_driver, 0, 0);
```

文件有一个工作：在内核级别连接驱动程序的各部分。它有几百行；驱动程序中所有其他文件都更小。

### `myfirst_cdev.c`：字符设备回调

打开、关闭、读取和写入回调是我们早在第18章编写的第一批代码。从那时起它们已经成长。将它们提取到`myfirst_cdev.c`使它们在一起并远离`myfirst.c`：

```c
/*
 * myfirst_cdev.c - character-device callbacks for the myfirst driver.
 *
 * The open/close/read/write callbacks all operate on the softc that
 * make_dev_s installed as si_drv1.  The ioctl dispatch is in
 * myfirst_ioctl.c; this file intentionally does not handle ioctls.
 */

#include <sys/param.h>
#include <sys/systm.h>
#include <sys/conf.h>
#include <sys/uio.h>
#include <sys/lock.h>
#include <sys/mutex.h>

#include "myfirst.h"
#include "myfirst_debug.h"

int
myfirst_open(struct cdev *dev, int oflags, int devtype, struct thread *td)
{
	struct myfirst_softc *sc = dev->si_drv1;

	mtx_lock(&sc->sc_mtx);
	sc->sc_open_count++;
	mtx_unlock(&sc->sc_mtx);

	DPRINTF(sc, MYF_DBG_OPEN, "open: count %u\n", sc->sc_open_count);
	return (0);
}

/* close, read, write follow the same pattern. */
```

每个回调以`sc = dev->si_drv1`（`make_dev_args`设置的在每cdev指针）开始，并操作softc。除了公共头文件外没有跨文件耦合。

### `myfirst_ioctl.c`：命令开关

`myfirst_ioctl.c`自第22章以来就在自己的文件中。第25章的添加是`MYFIRSTIOC_GETCAPS`处理程序：

```c
int
myfirst_ioctl(struct cdev *dev, u_long cmd, caddr_t data, int flag,
    struct thread *td)
{
	struct myfirst_softc *sc = dev->si_drv1;
	int error = 0;

	switch (cmd) {
	case MYFIRSTIOC_GETVER:
		*(int *)data = MYFIRST_IOCTL_VERSION;
		break;
	case MYFIRSTIOC_RESET:
		mtx_lock(&sc->sc_mtx);
		sc->sc_total_reads  = 0;
		sc->sc_total_writes = 0;
		mtx_unlock(&sc->sc_mtx);
		break;
	case MYFIRSTIOC_GETMSG:
		mtx_lock(&sc->sc_mtx);
		strlcpy((char *)data, sc->sc_msg, MYFIRST_MSG_MAX);
		mtx_unlock(&sc->sc_mtx);
		break;
	case MYFIRSTIOC_SETMSG:
		mtx_lock(&sc->sc_mtx);
		strlcpy(sc->sc_msg, (const char *)data, MYFIRST_MSG_MAX);
		sc->sc_msglen = strlen(sc->sc_msg);
		mtx_unlock(&sc->sc_mtx);
		break;
	case MYFIRSTIOC_GETCAPS:
		*(uint32_t *)data = MYF_CAP_RESET | MYF_CAP_GETMSG |
		                    MYF_CAP_SETMSG;
		break;
	default:
		error = ENOIOCTL;
		break;
	}
	return (error);
}
```

开关是驱动程序的整个公共ioctl接口。添加命令意味着添加case；弃用一个意味着删除case并在`myfirst_ioctl.h`中弃用常量。

### `myfirst_log.h`和`myfirst_log.c`：限速日志

第1节介绍了限速日志宏`DLOG_RL`和它跟踪的`struct myfirst_ratelimit`状态。限速状态在第1节中留在softc中嵌入，因为抽象尚未被提取。第6节是提取它的正确时机：限速代码小到值得收集在一个地方，通用到其他驱动程序可能想要它。

`myfirst_log.h`包含状态定义：

```c
#ifndef _MYFIRST_LOG_H_
#define _MYFIRST_LOG_H_

#include <sys/time.h>

struct myfirst_ratelimit {
	struct timeval rl_lasttime;
	int            rl_curpps;
};

#define MYF_RL_DEFAULT_PPS  10

#endif /* _MYFIRST_LOG_H_ */
```

`myfirst_log.c`包含附加和分离辅助函数：

```c
#include <sys/param.h>
#include <sys/systm.h>
#include <sys/lock.h>
#include <sys/mutex.h>

#include "myfirst.h"
#include "myfirst_debug.h"

int
myfirst_log_attach(struct myfirst_softc *sc)
{
	/*
	 * The rate-limit state is embedded by value in the softc, so
	 * there is no allocation to do.  This function exists so that
	 * the attach chain has a named label for logging in case a
	 * future version needs per-class allocations.
	 */
	bzero(&sc->sc_rl_generic, sizeof(sc->sc_rl_generic));
	bzero(&sc->sc_rl_io,      sizeof(sc->sc_rl_io));
	bzero(&sc->sc_rl_intr,    sizeof(sc->sc_rl_intr));

	return (0);
}

void
myfirst_log_detach(struct myfirst_softc *sc)
{
	/* Nothing to release; the state is embedded in the softc. */
	(void)sc;
}
```

今天`myfirst_log_attach`不做分配；它将限速字段清零并返回。明天，如果驱动程序需要动态的每类计数器数组，分配适合这里，附加链不需要更改。这是在绝对必要之前提取辅助函数的价值：形状已为增长做好准备。

这里头文件大小很重要。`myfirst_log.h`不到20行。20行的头文件在任何地方包含都很便宜，阅读便宜，保持同步便宜。如果`myfirst_log.h`增长到200行，从每个`.c`文件包含它的成本将开始显示在编译时间和审查摩擦中；那时下一步是再次拆分它。

### 更新的Makefile

拆分驱动程序的Makefile列出每个`.c`文件：

```makefile
# Makefile for the myfirst driver - Chapter 25 (1.8-maintenance).
#
# Chapter 25 splits the driver into subject-matter files.  Each file
# answers a single question; the Makefile lists them in alphabetical
# order after myfirst.c (which carries the module glue) so the
# reader sees the main file first.

KMOD=	myfirst
SRCS=	myfirst.c myfirst_cdev.c myfirst_debug.c myfirst_ioctl.c \
	myfirst_log.c myfirst_sysctl.c

CFLAGS+=	-I${.CURDIR}

SYSDIR?=	/usr/src/sys

.include <bsd.kmod.mk>
```

`SRCS`列出六个`.c`文件，每个主题一个。添加第七个是一行更改。内核构建系统自动获取`SRCS`中的每个文件；没有手动链接步骤，没有需要维护的makefile依赖树。

### 在哪里画每个文件边界

拆分驱动程序最难的部分不是决定拆分；而是决定边界在哪里。大多数拆分经历三个阶段，这些阶段适用于任何驱动程序，不仅仅是`myfirst`。

**阶段一**是平面文件。所有东西都在`driver.c`中。这对驱动程序的前300行来说是正确的形状。更早拆分创造的摩擦多于节省。

**阶段二**是主题拆分。ioctl分发进入`driver_ioctl.c`，sysctl树进入`driver_sysctl.c`，调试基础设施进入`driver_debug.c`。每个文件以它处理的主题命名。这是`myfirst`自第24章以来的位置。

**阶段三**是子系统拆分。随着驱动程序增长，一个主题变得大于单个文件。ioctl文件拆分为`driver_ioctl.c`（分发）和`driver_ioctl_rw.c`（读/写有效负载辅助函数）。sysctl文件类似拆分。这是功能完整的驱动程序最终达到的位置，通常在第三或第四个主要版本。

第25章结束时的`myfirst`稳固地处于阶段二。阶段三目前还不合理，第26章在将伪驱动程序拆分为USB附加变体并将`myfirst_core.c`留作主题无关核心时会重置时钟。今天预先拆分到阶段三没有价值。

从阶段二移动到阶段三的经验法则是：当单个主题文件超过1,000行时，或者当同一主题文件的两个不相关更改导致合并冲突时，该主题就准备好拆分了。

### 包含图和构建排序

一旦驱动程序被拆分为多个文件，包含图就很重要。循环包含在C中不是硬错误，但它是混乱的依赖结构的迹象，会困惑读者。正确的形状是有向无环头文件图，以`myfirst.h`为根，带有像`myfirst_ioctl.h`和`myfirst_log.h`这样的叶子头文件。

`myfirst.h`是最宽的头文件。它声明softc和其他每个文件使用的原型。它包含`myfirst_log.h`，因为softc按值有限速字段。

`myfirst_debug.h`是叶子。它声明`DPRINTF`宏系列和类位。每个`.c`文件直接或间接包含它。它不被`myfirst.h`包含，因为`myfirst.h`不应该强制调试宏进入任何不想要它们的调用者。

`myfirst_ioctl.h`是叶子。它声明命令号、有效负载结构和线格式版本整数。它被`myfirst_ioctl.c`（及其用户空间对应物`myfirstctl.c`）包含。

没有头文件包含除公共内核头文件和驱动程序自己的头文件以外的文件。没有`.c`文件包含另一个`.c`文件。包含图很浅，容易画图。

### 拆分的成本

拆分文件有实际成本。每次拆分添加一个头文件，每个头文件都必须维护。签名更改的函数必须在`.c`和`.h`中更新，更改必须传播到包含该头文件的每个其他`.c`文件。有十二个文件的驱动程序编译比只有一个文件的驱动程序稍慢，因为每个`.c`都必须包含几个头文件，预处理器必须解析它们。

这些成本是真实的但很小。它们比没有人愿意触碰的单体文件的成本小得多。规则是在不拆分的成本超过维护边界的成本时拆分。对于第25章结束时的`myfirst`，阈值已经被跨过。

### 拆分真实驱动程序的实用步骤

拆分文件是例行重构，但如果粗心进行，例行重构仍然可能引入错误。拆分内核内驱动程序的实用步骤是：

1. **识别主题。**从上到下阅读单体文件并按主题（cdev、ioctl、sysctl、调试、生命周期）对其函数分组。在一张纸上或注释块中写下分组。

2. **创建空文件。**将新的`.c`文件及其头文件添加到源代码树。编译一次以确保构建系统看到它们。

3. **一次移动一个主题。**将ioctl函数移动到`driver_ioctl.c`。将它们的声明移动到`driver_ioctl.h`。更新`driver.c`以`#include "driver_ioctl.h"`。编译。通过其测试矩阵运行驱动程序。

4. **为每个主题拆分提交。**每次主题移动是单个提交。提交日志写道："myfirst: split ioctl dispatch into driver_ioctl.c"。审查者可以清楚地看到移动；`git blame`在新文件中显示同一提交的相同行。

5. **验证包含图。**在所有主题移动后，使用`-Wunused-variable`和`-Wmissing-prototypes`编译以捕获应该有但没有原型的函数。在构建的模块上使用`nm`确认没有应该是`static`的符号被导出。

6. **重新测试。**运行完整的驱动程序测试矩阵。拆分文件不应该改变行为；如果测试开始失败，拆分引入了错误。

第25章结束时`myfirst`的步骤完全遵循这些。`examples/part-05/ch25-advanced/`下的最终目录是结果。

### 拆分时的常见错误

首次拆分驱动程序时，几个错误很常见。注意它们可以缩短学习曲线。

第一个错误是**将声明放在错误的头文件中**。如果`myfirst.h`声明了只在`myfirst_ioctl.c`中调用的函数，其他每个翻译单元都在为解析不需要的声明付出代价。如果`myfirst_ioctl.h`声明了在`myfirst_ioctl.c`和`myfirst_cdev.c`中都调用的函数，两个消费者通过ioctl头文件耦合，ioctl头文件中的任何更改都重建两个文件。修复是将跨切面声明放在`myfirst.h`中，主题特定声明放在主题特定头文件中。

第二个错误是**忘记在应该是文件范围的函数上加`static`**。只在`myfirst_sysctl.c`内使用的函数应该声明为`static`。没有`static`，函数从目标文件导出，这意味着另一个文件可能意外调用它，原始文件中的任何后续重命名都变成ABI更改。`static`规范防止了整个类的问题。

第三个错误是**循环包含**。如果`myfirst_ioctl.h`包含`myfirst.h`，而`myfirst.h`包含`myfirst_ioctl.h`，驱动程序可以编译（感谢包含保护），但依赖图是错误的。对任一文件的每次编辑现在都触发包含任一文件的所有内容的重建。修复是决定哪个头文件在图中更高并删除反向引用。

第四个错误是**将主题重新引入错误的文件**。拆分六个月后，有人通过编辑`myfirst.c`添加新ioctl，因为那是ioctl以前所在的地方。单一职责规则必须由审查者强制执行。将新ioctl放在`myfirst.c`中的补丁被拒绝，并带有指向`myfirst_ioctl.c`的注释。

### 第6节总结

每个文件回答一个问题的驱动程序是第一天就可以交给新贡献者的驱动程序。他们阅读文件名，选择一个主题，开始编辑恰好一个文件。`myfirst`驱动程序已经跨过了那个阈值。六个`.c`文件加上它们的头文件保存了驱动程序自第17章以来增长的所有函数，每个文件以其功能命名。

在下一节中，我们将从内部组织转向外部准备。准备好生产的驱动程序具有在安装到开发者不拥有的机器之前必须满足的一小部分属性。第25章的生产就绪检查清单列出了这些属性并引导`myfirst`通过每一个。


## 第3节：配置：可调参数和Sysctl

第三种规范是将决策外部化的规范。任何驱动程序都有一些值，有人可能合理地希望在不重新构建模块的情况下更改它们：超时、重试次数、内部缓冲区大小、详细级别、功能开关。将这些值硬编码到源代码中的驱动程序强制每次更改都要经历完整的编译、安装和重新启动周期。将它们作为加载器可调参数和sysctl公开的驱动程序允许操作员通过一次编辑或一条命令在启动时或运行时调整行为。提供这些旋钮的成本很小；不提供它们的成本由操作员承担。

本节教授两种用于外部化配置的FreeBSD机制：加载器可调参数（从`/boot/loader.conf`读取并在内核到达`attach`之前应用）和sysctl（在运行时通过`sysctl(8)`读取和写入）。它解释了它们如何通过`CTLFLAG_TUN`标志协作，展示了如何在每驱动程序和每实例可调参数之间选择，演示了`TUNABLE_*_FETCH`系列，并以一个简短的实验结束，在该实验中`myfirst`驱动程序获得三个新的可调参数，读者在每个实验中引导虚拟机。

### 可调参数与Sysctl的区别

可调参数和sysctl对操作员来说看起来很相似。两者都是命名空间中的字符串，如`hw.myfirst.debug_mask_default`或`dev.myfirst.0.debug.mask`。两者都接受操作员设置的值。两者最终都进入内核内存。它们在何时以及如何上有所不同。

**可调参数**是在引导加载程序环境中设置的变量。引导加载程序（`loader(8)`）读取`/boot/loader.conf`，将其`key=value`对收集到一个环境中，并在内核启动时将该环境交给内核。内核通过`getenv(9)`系列和`TUNABLE_*_FETCH`宏公开此环境。可调参数在引导期间读取，通常在相应驱动程序附加之前。它们不能在运行时更改（更改`/boot/loader.conf`需要重新启动才能生效）。它们适用于必须在`attach`运行之前知道的值：静态分配表的大小、控制哪些代码路径被编译到附加路径的功能标志、调试掩码的初始值。

**Sysctl**是内核层次配置树中的变量，可在运行时通过`sysctl(2)`系统调用和`sysctl(8)`工具访问。Sysctl可以是只读（`CTLFLAG_RD`）、读写（`CTLFLAG_RW`）或只读根可写（各种标志组合）。它们适用于在驱动程序附加后更改才有意义的值：详细级别、节流速率、计数器重置命令、可写状态旋钮。

有用的特性是这两种机制可以共享一个变量。用`CTLFLAG_TUN`声明的sysctl告诉内核在引导时读取同名可调参数并将其值用作sysctl的初始值。然后操作员可以在运行时调整sysctl，可调参数在重新启动后作为默认值持久存在。`myfirst`驱动程序已经为其调试掩码使用了这种模式：`debug.mask`是一个`CTLFLAG_RW | CTLFLAG_TUN` sysctl，而`hw.myfirst.debug_mask_default`是`/boot/loader.conf`中的匹配可调参数。第3节将该模式推广到驱动程序想要公开的每个配置旋钮。

### `TUNABLE_*_FETCH`系列

FreeBSD提供了一系列宏，用于从引导加载程序环境读取可调参数。每个宏读取命名可调参数，将其解析为正确的C类型，并存储结果。如果未设置可调参数，变量保持其现有值；因此调用者必须在调用fetch宏之前将变量初始化为正确的默认值。

这些宏在`/usr/src/sys/sys/kernel.h`中声明：

```c
TUNABLE_INT_FETCH(path, pval)        /* int */
TUNABLE_LONG_FETCH(path, pval)       /* long */
TUNABLE_ULONG_FETCH(path, pval)      /* unsigned long */
TUNABLE_INT64_FETCH(path, pval)      /* int64_t */
TUNABLE_UINT64_FETCH(path, pval)     /* uint64_t */
TUNABLE_BOOL_FETCH(path, pval)       /* bool */
TUNABLE_STR_FETCH(path, pval, size)  /* 给定大小的char缓冲区 */
```

每个宏展开为匹配的`getenv_*`调用。例如，`TUNABLE_INT_FETCH`展开为`getenv_int(path, pval)`，它读取引导加载程序环境并将该值解析为整数。

路径是一个字符串，约定形式为`hw.<driver>.<knob>`用于每驱动程序可调参数，`hw.<driver>.<unit>.<knob>`用于每实例可调参数。`hw.`前缀是硬件相关可调参数的约定；其他前缀（`kern.`、`net.`）存在于不同的子系统，但在驱动程序代码中较少见。

来自`myfirst`驱动程序的一个工作示例展示了该模式：

```c
static int
myfirst_attach(device_t dev)
{
        struct myfirst_softc *sc = device_get_softc(dev);
        int error;

        /* 初始化默认值。 */
        sc->sc_debug = 0;
        sc->sc_timeout_sec = 30;
        sc->sc_max_retries = 3;

        /* 读取可调参数。如果可调参数未设置，
         * 变量保持其默认值。 */
        TUNABLE_INT_FETCH("hw.myfirst.debug_mask_default", &sc->sc_debug);
        TUNABLE_INT_FETCH("hw.myfirst.timeout_sec", &sc->sc_timeout_sec);
        TUNABLE_INT_FETCH("hw.myfirst.max_retries", &sc->sc_max_retries);

        /* ... attach的其余部分 ... */
}
```

操作员在`/boot/loader.conf`中设置可调参数：

```ini
hw.myfirst.debug_mask_default="0xff"
hw.myfirst.timeout_sec="15"
hw.myfirst.max_retries="5"
```

重新启动后，`myfirst`的每个实例以`sc_debug=0xff`、`sc_timeout_sec=15`、`sc_max_retries=5`附加。不需要重新构建；这些值存在于驱动程序源代码之外。

### 每驱动程序与每实例可调参数

可能多次附加的驱动程序需要做出决定：其可调参数应该应用于每个实例，还是每个实例应该有自己的？

每驱动程序形式使用`hw.myfirst.debug_mask_default`形式的路径。`myfirst`的每个实例在附加时读取这个单一变量，因此所有实例以相同的默认值开始。这是更简单的形式，当可调参数在每个实例上具有相同的含义时是正确的。

每实例形式使用`hw.myfirst.0.debug_mask_default`形式的路径，其中`0`是单元号。每个实例读取自己的变量，因此实例0和实例1可以有不同的默认值。当每个实例背后的硬件可能合理地需要不同的配置时，这是正确的形式，例如同一系统上的两个具有不同工作负载的PCI适配器。

这是一个设计选择，而不是正确性问题。大多数驱动程序对大多数可调参数使用每驱动程序形式，每实例形式仅保留给少数确实需要每实例配置的情况。对于`myfirst`（一个虚构的伪设备），每驱动程序是每个可调参数的正确默认值。因此第25章驱动程序添加三个每驱动程序可调参数（`timeout_sec`、`max_retries`、`log_ratelimit_pps`）并保留现有的每驱动程序`debug_mask_default`。

如果驱动程序需要，一种结合两种形式的模式是首先读取每驱动程序可调参数作为基线，然后读取每实例可调参数作为覆盖：

```c
int defval = 30;

TUNABLE_INT_FETCH("hw.myfirst.timeout_sec", &defval);
sc->sc_timeout_sec = defval;
TUNABLE_INT_FETCH_UNIT("hw.myfirst", unit, "timeout_sec",
    &sc->sc_timeout_sec);
```

FreeBSD没有开箱即用的`TUNABLE_INT_FETCH_UNIT`宏；需要此功能的驱动程序必须使用`snprintf`组合路径，然后手动调用`getenv_int`。工作量很小但需求很少，因此`myfirst`不会这样做。

### `CTLFLAG_TUN`标志

外部化故事的另一半是可调参数本身只在引导时读取。要使相同的值在运行时可调整，驱动程序用`CTLFLAG_TUN`声明匹配的sysctl：

```c
SYSCTL_ADD_UINT(ctx, child, OID_AUTO, "debug.mask",
    CTLFLAG_RW | CTLFLAG_TUN,
    &sc->sc_debug, 0,
    "已启用调试类别的位掩码");
```

`CTLFLAG_TUN`告诉内核此sysctl的初始值应从同名的引导加载程序环境变量中获取，使用OID名称作为键。匹配是文本的和自动的；驱动程序不需要单独调用`TUNABLE_INT_FETCH`。

关于何时遵守`CTLFLAG_TUN`有一个微妙的规则。该标志适用于OID的*初始*值，该值在创建sysctl时从环境中读取。如果驱动程序在创建sysctl之前显式调用`TUNABLE_INT_FETCH`，显式fetch优先，`CTLFLAG_TUN`实际上是多余的。如果驱动程序不调用`TUNABLE_INT_FETCH`而仅依赖`CTLFLAG_TUN`，sysctl的初始值会自动来自环境。

在实践中，`myfirst`驱动程序为了清晰起见同时使用两种机制。attach中显式的`TUNABLE_INT_FETCH`使驱动程序的意图在源代码中可见；sysctl上的`CTLFLAG_TUN`为操作员在sysctl文档中提供了一个清晰的提示，即OID遵循加载器可调参数。仅其中任何一种机制都可以工作；同时使用两者是一个小的重复，但在可读性方面是值得的。

### 将可调参数声明为静态Sysctl

对于不属于特定实例的驱动程序范围sysctl，FreeBSD提供了编译时宏，这些宏将sysctl绑定到静态变量并在一个声明中从环境读取其默认值。规范形式：

```c
SYSCTL_NODE(_hw, OID_AUTO, myfirst, CTLFLAG_RW, NULL,
    "myfirst伪驱动程序");

static int myfirst_verbose = 0;
SYSCTL_INT(_hw_myfirst, OID_AUTO, verbose,
    CTLFLAG_RWTUN, &myfirst_verbose, 0,
    "启用详细驱动程序日志记录");
```

`SYSCTL_NODE`声明一个新的父节点`hw.myfirst`。`SYSCTL_INT`声明一个整数OID `hw.myfirst.verbose`，带有`CTLFLAG_RWTUN`（它组合了`CTLFLAG_RW`和`CTLFLAG_TUN`）。`myfirst_verbose`变量是驱动程序的全局详细级别。操作员在`/boot/loader.conf`中设置`hw.myfirst.verbose=1`以在引导时启用详细输出，或运行`sysctl hw.myfirst.verbose=1`在运行时切换它。

静态声明适用于驱动程序范围的状态。每实例状态（`sc_debug`、计数器）继续存在于`dev.myfirst.<unit>.*`下，并通过`device_get_sysctl_ctx`动态声明。

### 关于`SYSCTL_INT`与`SYSCTL_ADD_INT`的小注

静态形式`SYSCTL_INT(parent, OID_AUTO, ...)`是编译时声明。动态形式`SYSCTL_ADD_INT(ctx, list, OID_AUTO, ...)`是运行时调用。两者都产生sysctl OID。静态形式适用于存在不依赖于附加硬件的驱动程序范围sysctl。动态形式适用于在attach中创建并在detach时销毁的每实例sysctl。

初学者的一个常见错误是对驱动程序范围sysctl使用动态形式，这虽然可行，但需要一个驱动程序范围的`sysctl_ctx_list`，必须在`MOD_LOAD`时初始化并在`MOD_UNLOAD`时释放。静态形式避免了所有这些：sysctl从模块加载的那一刻存在直到卸载的那一刻，内核自动处理注册和注销。

### 文档化可调参数

操作员不知道的可调参数是不会被使用的可调参数。规范是在三个地方文档化驱动程序公开的每个可调参数。

首先，可调参数在源代码中的声明应该包含一行描述字符串。对于`SYSCTL_ADD_UINT`等，最后一个参数是描述：

```c
SYSCTL_ADD_UINT(ctx, child, OID_AUTO, "timeout_sec",
    CTLFLAG_RW | CTLFLAG_TUN,
    &sc->sc_timeout_sec, 0,
    "硬件命令的超时秒数（默认30，最小1，最大3600）");
```

描述字符串是`sysctl -d`在操作员请求文档时打印的内容。好的描述命名单位、默认值和可接受范围。

其次，驱动程序的`MAINTENANCE.md`（在第7节中介绍）应该列出每个可调参数，每个一个段落。段落解释可调参数的作用、何时更改它、默认值是什么以及设置它有什么副作用。

第三，驱动程序的手册页（通常是`myfirst(4)`）应该在`LOADER TUNABLES`部分下列出每个可调参数，在`SYSCTL VARIABLES`部分下列出每个sysctl。`myfirst`驱动程序还没有手册页；本章将手册页视为后续关注事项。`MAINTENANCE.md`文档在此期间承载完整的文档。

### 工作示例：`hw.myfirst.timeout_sec`

`myfirst`驱动程序在这个阶段还没有真正的硬件，但本章引入了一个未来章节将使用的虚构`timeout_sec`旋钮。完整的小工作流程是：

1. 在`myfirst.h`中，将字段添加到softc：
   ```c
   struct myfirst_softc {
           /* ... 现有字段 ... */
           int   sc_timeout_sec;
   };
   ```

2. 在`myfirst_bus.c`（在第6节中引入的新文件，保存attach和detach）中，初始化默认值并读取可调参数：
   ```c
   sc->sc_timeout_sec = 30;
   TUNABLE_INT_FETCH("hw.myfirst.timeout_sec", &sc->sc_timeout_sec);
   ```

3. 在`myfirst_sysctl.c`中，将旋钮公开为运行时sysctl：
   ```c
   SYSCTL_ADD_INT(ctx, child, OID_AUTO, "timeout_sec",
       CTLFLAG_RW | CTLFLAG_TUN,
       &sc->sc_timeout_sec, 0,
       "硬件命令的超时秒数");
   ```

4. 在`MAINTENANCE.md`中，文档化可调参数：
   ```
   hw.myfirst.timeout_sec
       硬件命令的超时秒数。默认30。
       可接受范围1到3600。低于1的值被
       钳制到1；高于3600的值被钳制到3600。
       可通过sysctl dev.myfirst.<unit>.
       timeout_sec在运行时调整。
   ```

5. 在回归脚本中，添加一行验证可调参数选取其默认值：
   ```
   [ "$(sysctl -n dev.myfirst.0.timeout_sec)" = "30" ] || fail
   ```

驱动程序现在有一个超时旋钮，操作员可以在引导时通过`/boot/loader.conf`设置，可以在运行时通过`sysctl`调整，并可以在`MAINTENANCE.md`中找到文档。未来引入新的可配置值的每一章都将遵循相同的五步工作流程。

### 范围检查和验证

操作员可以设置为任何值的可调参数是可以设置为超出范围值的可调参数，无论是意外（`/boot/loader.conf`中的拼写错误）还是 misguided 的调优尝试。驱动程序必须验证它读取的值并钳制或拒绝它。

对于在引导时用`TUNABLE_INT_FETCH`读取的可调参数，验证内联发生：

```c
sc->sc_timeout_sec = 30;
TUNABLE_INT_FETCH("hw.myfirst.timeout_sec", &sc->sc_timeout_sec);
if (sc->sc_timeout_sec < 1 || sc->sc_timeout_sec > 3600) {
        device_printf(dev,
            "可调参数hw.myfirst.timeout_sec超出范围（%d），"
            "钳制到默认值30\n",
            sc->sc_timeout_sec);
        sc->sc_timeout_sec = 30;
}
```

对于具有运行时写入支持的sysctl，验证发生在处理程序中。一个简单的`CTLFLAG_RW` sysctl对int变量接受任何int；要拒绝超出范围的写入，驱动程序声明一个自定义处理程序：

```c
static int
myfirst_sysctl_timeout(SYSCTL_HANDLER_ARGS)
{
        struct myfirst_softc *sc = arg1;
        int v;
        int error;

        v = sc->sc_timeout_sec;
        error = sysctl_handle_int(oidp, &v, 0, req);
        if (error != 0 || req->newptr == NULL)
                return (error);
        if (v < 1 || v > 3600)
                return (EINVAL);
        sc->sc_timeout_sec = v;
        return (0);
}
```

处理程序读取当前值，调用`sysctl_handle_int`执行实际的I/O，并仅在值在范围内时应用新值。写入0或7200向操作员返回`EINVAL`，而不更改sysctl的值。这是正确的行为：操作员获得清晰的反馈，表明写入被拒绝。

`myfirst`驱动程序在这个阶段不验证其整数sysctl，因为它们都不可能超出范围（调试掩码是一个位掩码，任何32位值都是合法的掩码）。未来引入超时、重试次数和缓冲区大小的驱动程序将一致地使用自定义处理程序模式。

### 何时公开可调参数，何时保持内部

公开可调参数是一个承诺。一旦操作员在`/boot/loader.conf`中设置了`hw.myfirst.timeout_sec=15`，驱动程序就做出了承诺，即该旋钮的含义在以后的版本中不会改变。删除可调参数会破坏生产部署。静默更改其解释会更糟糕地破坏它们。

正确的规范是仅当满足以下所有三个条件时才将值公开为可调参数：

1. 该值具有操作用例。有人可能合理地需要在真实部署中更改它。
2. 合理值的范围是已知的。驱动程序可以在`MAINTENANCE.md`中文档化它。
3. 在驱动程序的生命周期内支持该旋钮的成本值得其提供的操作价值。

对自己问这三个问题并对所有三个回答是的驱动程序公开了一个小的、有目的的可调参数集。因为"操作员可能想调整它"而将每个内部常量公开为可调参数的驱动程序最终得到一个庞大的配置表面，没有人可以文档化，也没有人可以在其完整范围内测试。

对于`myfirst`，初始可调参数集故意很小：`debug_mask_default`、`timeout_sec`、`max_retries`、`log_ratelimit_pps`。每个都有清晰的操作案例、清晰的默认值和清晰的范围。驱动程序不是试图将softc中的每个int字段公开为可调参数；它试图公开操作员可能实际想要触摸的那些。

### 关于字符串`CTLFLAG_RWTUN`的警告说明

`TUNABLE_STR_FETCH`宏从引导加载程序环境读取字符串到固定大小的缓冲区。匹配的sysctl标志，`SYSCTL_STRING`上的`CTLFLAG_RWTUN`，可以工作，但有一个陷阱：字符串的存储必须是静态缓冲区，而不是softc中的每实例`char[]`字段。写入softc字段的字符串sysctl可能在sysctl框架释放softc之前没有注销OID的情况下比softc存活更久，这会导致释放后使用错误。

更安全的模式是将字符串公开为只读，并通过自定义处理程序处理写入，该处理程序在锁下将新值复制到softc中。`myfirst`驱动程序遵循此模式：`dev.myfirst.0.message`仅以`CTLFLAG_RD`公开，写入通过`MYFIRSTIOC_SETMSG` ioctl进行。ioctl路径获取softc互斥锁，复制新值，然后解锁；sysctl OID没有生命周期问题。

字符串可调参数和sysctl足够有用，一些驱动程序值得小心处理，但第25章驱动程序不需要它们。这个原则值得命名，因为这个陷阱在真实驱动程序中稍后会暴露出来。

### 可调参数与内核模块：它们在哪里

关于加载器环境的两个小但重要的细节值得命名。

首先，`/boot/loader.conf`中的可调参数从内核启动的那一刻起适用。它对任何调用`TUNABLE_*_FETCH`或具有`CTLFLAG_TUN` sysctl的模块都可用，即使该模块在引导时没有加载。稍后用`kldload`加载的模块仍然看到可调参数的值。这很方便：操作员设置一次可调参数，然后忘记它，直到模块加载。

其次，可调参数从环境中读取但不能写回。在运行时更改`hw.myfirst.timeout_sec`（用`kenv`）不影响任何已经读取它的驱动程序；softc中的变量才是重要的，而不是环境。要在运行时更改值，操作员使用匹配的sysctl。

这两个细节一起解释了为什么`CTLFLAG_TUN`是大多数配置旋钮的正确形状：可调参数设置引导默认值，sysctl处理运行时调整，操作员的工具包（`/boot/loader.conf`加`sysctl(8)`）按预期工作。

### 第3节总结

配置是与操作员的对话。通过可调参数和sysctl公开正确值的驱动程序可以在不重新构建的情况下调优；将每个值隐藏在源代码中的驱动程序强制每次更改都要重新构建。`TUNABLE_*_FETCH`系列和`CTLFLAG_TUN`标志一起覆盖引导时和运行时调整，每驱动程序与每实例的选择使驱动程序适应其操作现实。`myfirst`驱动程序现在除了现有的`debug_mask_default`外还有三个新可调参数，每个都有文档化的范围和匹配的sysctl。

在下一节中，我们从驱动程序公开的内容转向驱动程序如何演进。今天有效的配置旋钮必须在明天驱动程序更改时仍然有效。版本控制规范是保持该承诺的方式。

## 第4节：版本控制和兼容性策略

第四种规范是在不破坏的情况下演进的规范。`myfirst`驱动程序公开的每个表面——`/dev/myfirst0`节点、ioctl接口、sysctl树、可调参数集——都是与某人的契约。静默更改其中任何一个含义的更改是破坏性更改，而溜过开发者注意的破坏性更改是大量真实世界驱动程序错误的来源。本节教授如何有意识地版本化驱动程序的公共表面，以便调用者可以看到更改，旧调用者在驱动程序添加新功能时继续工作。

本章为`myfirst`驱动程序使用三个不同的版本号。每个都有特定的目的。在扎根之前将它们混淆是值得避免的困惑来源。

### 三个版本号

`myfirst`驱动程序有三个版本标识符，在第23、24和25章中引入。每个位于不同的位置，因不同的原因而更改。

第一个是**人类可读的发布字符串**。对于`myfirst`，这是`MYFIRST_VERSION`，定义在`myfirst_sysctl.c`中，通过`dev.myfirst.0.version` sysctl公开。其当前值是`"1.8-maintenance"`。发布字符串是给人类看的：运行`sysctl dev.myfirst.0.version`的操作员看到一个短标签，标识驱动程序历史的这个特定检查点。发布字符串不被程序解析；它被人阅读。它在驱动程序达到作者想要标记的新里程碑时更改，在本书中是在每章结束时。

第二个是**内核模块版本整数**。这是`MODULE_VERSION(myfirst, N)`，其中`N`是内核依赖机制使用的整数。声明`MODULE_DEPEND(other, myfirst, 1, 18, 18)`的另一个模块要求`myfirst`在版本18或以上存在（并且低于或等于18，在此声明中意味着恰好18）。模块版本整数仅在模块的内核内调用者需要重新编译时更改，例如当共享符号的签名更改时。对于不公开公共内核符号的驱动程序（如`myfirst`），模块版本号主要是象征性的；本章在每个里程碑提升它，以保持读者的心智模型在三个版本标识符之间对齐。

第三个是**ioctl接口版本整数**。对于`myfirst`，这是`myfirst_ioctl.h`中的`MYFIRST_IOCTL_VERSION`。其当前值是1。当ioctl头文件以针对先前版本编译的旧用户空间程序会误解的方式更改时，ioctl接口版本会更改。重新编号的ioctl命令、更改的有效载荷布局、现有ioctl的更改语义：这些中的每一个都是对ioctl接口的破坏性更改，必须提升版本。添加新的ioctl命令、在末尾扩展有效载荷而不重新解释现有字段、添加不影响旧命令的功能：这些是兼容的更改，不需要提升。

一个简单的经验法则可以保持三个直截了当。发布字符串是操作员读取的内容。模块版本整数是其他模块检查的内容。ioctl版本整数是用户空间程序检查的内容。每个都有自己的时间表。

### 为什么用户需要查询版本

通过ioctl与驱动程序对话的用户空间程序有一个问题。头文件`myfirst_ioctl.h`定义了一组命令、布局和版本1语义。新版本的驱动程序可能添加命令、更改布局或更改语义。当用户空间程序在具有比其编译版本更新或更旧的驱动程序的系统上运行时，除非它询问，否则它无法知道驱动程序的实际版本。

解决方案是一个ioctl，其唯一目的是返回驱动程序的ioctl版本。`myfirst`驱动程序已经有一个：`MYFIRSTIOC_GETVER`，定义为`_IOR('M', 1, uint32_t)`。用户空间程序在打开设备后立即调用此ioctl，将返回的版本与其编译的版本进行比较，并决定是否可以安全地进行。

用户空间的模式：

```c
#include "myfirst_ioctl.h"

int fd = open("/dev/myfirst0", O_RDWR);
uint32_t ver;
if (ioctl(fd, MYFIRSTIOC_GETVER, &ver) < 0)
        err(1, "getver");
if (ver != MYFIRST_IOCTL_VERSION)
        errx(1, "驱动程序版本%u，工具期望%u",
            ver, MYFIRST_IOCTL_VERSION);
```

如果版本不匹配，工具拒绝运行。这是一种可能的策略。更宽容的策略将允许工具针对较新的驱动程序运行，如果驱动程序的新ioctl是旧ioctl的超集，并且如果工具可以回退到较旧的命令集，则允许工具针对较旧的驱动程序运行。更严格的策略将要求完全匹配。工具的作者根据在向后兼容性上花费多少努力来选择这些策略。

### 添加新Ioctl而不破坏旧调用者

常见情况是向驱动程序添加新功能，这通常意味着添加新ioctl。只要遵循两条规则，规范就是直截了当的。

首先，**不要重用现有的ioctl号**。每个ioctl命令都有一个由`_IO`、`_IOR`、`_IOW`或`_IOWR`编码的唯一`(magic, number)`对。`myfirst_ioctl.h`中的当前分配：

```c
#define MYFIRSTIOC_GETVER   _IOR('M', 1, uint32_t)
#define MYFIRSTIOC_GETMSG   _IOR('M', 2, char[MYFIRST_MSG_MAX])
#define MYFIRSTIOC_SETMSG   _IOW('M', 3, char[MYFIRST_MSG_MAX])
#define MYFIRSTIOC_RESET    _IO('M', 4)
```

新ioctl获取相同魔术字母下的下一个可用号码：`MYFIRSTIOC_GETCAPS = _IOR('M', 5, uint32_t)`。号码5以前没有被使用过，不能与旧程序的编译二进制冲突。在没有`GETCAPS`的版本中编译的旧程序根本不发送该ioctl，因此旧程序不受添加的影响。

其次，**不要为纯添加提升`MYFIRST_IOCTL_VERSION`**。不更改旧ioctl含义的新ioctl是兼容的更改。从未听说过新ioctl的旧用户空间程序仍然说相同的语言；版本整数应该保持不变。为每次添加提升版本将强制每个调用者在驱动程序获得新命令时重新构建，这违背了版本控制的目的。

用不同语义替换现有ioctl的新ioctl确实需要提升。如果驱动程序添加`MYFIRSTIOC_SETMSG_V2`，具有新布局并退役`MYFIRSTIOC_SETMSG`，则调用退役命令的旧程序会看到更改的行为（驱动程序可能返回`ENOIOCTL`或可能表现不同）。那是破坏性更改，提升信号它。

### 退役已弃用的Ioctl

退役是礼貌管理的移除形式。当要移除命令时，驱动程序宣布意图，在过渡期内保持命令工作，并在以后的版本中移除它。典型的弃用序列：

- 版本N：在`MAINTENANCE.md`中宣布弃用。命令仍然工作。
- 版本N+1：命令工作但每次使用时记录限速警告。用户看到警告并知道要迁移。
- 版本N+2：命令返回`EOPNOTSUPP`并记录限速错误。大多数用户现在已经迁移；少数没有的用户被迫迁移。
- 版本N+3：命令从头文件中移除。仍然引用它的程序不再编译。

过渡期应该以发布（通常是一两个主要版本）而不是日历时间来衡量。保持弃用契约可预测的驱动程序为消费者提供一个稳定的目标来瞄准。

对于本章中的`myfirst`，还没有命令被弃用。本章为未来引入该模式。相同的规范适用于sysctl树：OID处理程序中的限速警告告诉操作员该名称即将被淘汰，`MAINTENANCE.md`中的注释记录计划的移除日期。

### 能力位掩码模式

对于在多个版本中演进的驱动程序，单个版本整数告诉调用者它们在与哪个版本对话，但不告诉该版本具体支持哪些功能。功能丰富的驱动程序受益于更细粒度的机制：能力位掩码。

想法很简单。驱动程序在`myfirst_ioctl.h`中定义一组能力位：

```c
#define MYF_CAP_RESET       (1U << 0)
#define MYF_CAP_GETMSG      (1U << 1)
#define MYF_CAP_SETMSG      (1U << 2)
#define MYF_CAP_TIMEOUT     (1U << 3)
#define MYF_CAP_MAXRETRIES  (1U << 4)
```

一个新的ioctl `MYFIRSTIOC_GETCAPS`返回一个`uint32_t`，其中设置了该驱动程序实际支持的功能位：

```c
#define MYFIRSTIOC_GETCAPS  _IOR('M', 5, uint32_t)
```

在内核中：

```c
case MYFIRSTIOC_GETCAPS:
        *(uint32_t *)data = MYF_CAP_RESET | MYF_CAP_GETMSG |
            MYF_CAP_SETMSG;
        break;
```

在用户空间：

```c
uint32_t caps;
ioctl(fd, MYFIRSTIOC_GETCAPS, &caps);
if (caps & MYF_CAP_TIMEOUT)
        set_timeout(fd, 60);
else
        warnx("驱动程序不支持超时配置");
```

能力位掩码允许用户空间程序发现功能而无需试错。如果调用者想知道某个功能是否存在，它检查该位；如果设置了该位，调用者就知道驱动程序支持该功能和相关的ioctl。未定义该位的旧驱动程序不会假装支持它从未听说过的功能。

该模式随着驱动程序的增长而很好地扩展。每个发布为新功能添加新位。退役的功能保留其位作为未使用保留；为新的含义回收位将是破坏性更改。位掩码本身是一个`uint32_t`，在需要添加第二个字之前给驱动程序32个功能。如果驱动程序达到32个功能，添加第二个字是兼容的更改（新位在新字段中，因此只读取第一个字的旧程序看到相同的位）。

第25章将`MYFIRSTIOC_GETCAPS`添加到`myfirst`驱动程序，设置了三个位：`MYF_CAP_RESET`、`MYF_CAP_GETMSG`和`MYF_CAP_SETMSG`。`myfirstctl`用户空间程序被扩展为在启动时查询能力并拒绝调用不支持的功能。

### Sysctl弃用

FreeBSD不提供sysctl树上的专用`CTLFLAG_DEPRECATED`标志。相关的标志`CTLFLAG_SKIP`，定义在`/usr/src/sys/sys/sysctl.h`中，将OID从默认列表中隐藏（如果显式命名它仍然可读），但它主要用于退休以外的目的。因此，退役sysctl OID的礼貌方式是用一个处理程序替换它，该处理程序执行预期的工作*并*在前几次触摸OID时记录限速警告。

```c
static int
myfirst_sysctl_old_counter(SYSCTL_HANDLER_ARGS)
{
        struct myfirst_softc *sc = arg1;

        DLOG_RL(sc, &sc->sc_rl_deprecated, MYF_RL_DEFAULT_PPS,
            "sysctl dev.myfirst.%d.old_counter已弃用；"
            "请改用new_counter\n",
            device_get_unit(sc->sc_dev));
        return (sysctl_handle_int(oidp, &sc->sc_old_counter, 0, req));
}
```

操作员在前几次读取OID时在`dmesg`中看到警告，这是迁移的强烈提示。sysctl仍然工作，因此显式引用它的脚本在过渡期间不会破坏。一两个发布后，OID本身被移除。`MAINTENANCE.md`中的注释记录意图和目标发布。

对于`myfirst`，还没有sysctl被弃用。第25章驱动程序在文档中引入该模式，并为未来使用做好准备。

### 用户可见的行为更改

并非每个破坏性更改都是重命名或重新编号。有时驱动程序保持相同的ioctl、相同的sysctl、相同的可调参数，并静默更改操作的作用。过去只清零计数器的`MYFIRSTIOC_RESET`现在也清除消息是行为更改。过去报告总写入字节数的sysctl现在报告千字节数是行为更改。过去是绝对值现在成为乘数的可调参数是行为更改。

行为更改是最难捕捉的破坏性更改，因为它们不会显示在头文件或sysctl列表的diff中。规范是在`MAINTENANCE.md`的"变更日志"部分下文档化每个行为更改，在ioctl语义更改时提升ioctl接口版本整数，并在描述字符串本身中宣布sysctl语义更改。

行为更改的一个好模式是引入新的命名命令或新sysctl而不是重新定义现有的。`MYFIRSTIOC_RESET`保持旧语义。`MYFIRSTIOC_RESET_ALL`是具有新语义的新命令。旧命令最终被弃用。成本是过渡期稍大的公共表面；好处是没有调用者被静默行为更改破坏。

### `MODULE_DEPEND`和模块间兼容性

`MODULE_DEPEND`宏声明一个模块依赖于另一个模块并需要特定的版本范围：

```c
MODULE_DEPEND(myfirst, dependency, 1, 2, 3);
```

三个整数是`myfirst`兼容的`dependency`的最小、首选和最大版本。如果`dependency`不存在或超出范围，内核拒绝加载`myfirst`。

对于不发布内核内符号的驱动程序，`MODULE_DEPEND`最常用于依赖标准子系统模块：

```c
MODULE_DEPEND(myfirst_usb, usb, 1, 1, 1);
```

这声明`myfirst`的USB版本恰好需要版本1的USB栈。子系统模块的版本号由子系统作者管理；驱动程序作者在子系统的头文件（对于USB，`/usr/src/sys/dev/usb/usbdi.h`）或已经依赖它的另一个驱动程序中找到当前值。

对于第25章末尾的`myfirst`，不需要`MODULE_DEPEND`，因为伪驱动程序不需要子系统。第26章USB章节将在驱动程序转换为USB附加版本时添加第一个真正的`MODULE_DEPEND`。

### 工作示例：1.7到1.8过渡

第25章驱动程序在章节末尾提升三个版本标识符：

- `MYFIRST_VERSION`：从`"1.7-integration"`到`"1.8-maintenance"`。
- `MODULE_VERSION(myfirst, N)`：从17到18。
- `MYFIRST_IOCTL_VERSION`：保持为1，因为本章的ioctl添加是纯添加（新命令，无移除，无语义更改）。

`GETCAPS` ioctl用命令号5添加，之前未使用。针对第24章版本头文件编译的旧`myfirstctl`二进制文件不知道`GETCAPS`也不发送它；它们继续不变地工作。针对第25章头文件编译的新`myfirstctl`二进制文件在启动时查询`GETCAPS`并相应地行为。

`MAINTENANCE.md`文档为1.8获得一个变更日志条目：

```text
## 1.8-maintenance

- 添加了MYFIRSTIOC_GETCAPS（命令5）返回能力
  位掩码。与所有早期用户空间程序兼容。
- 添加了可调参数hw.myfirst.timeout_sec、hw.myfirst.max_retries、
  hw.myfirst.log_ratelimit_pps。每个都有匹配的可写
  sysctl在dev.myfirst.<unit>下。
- 通过ppsratecheck(9)添加了限速日志记录。
- 从1.7没有破坏性更改。
```

阅读`MAINTENANCE.md`的驱动程序用户一眼就能看到更改了什么，并可以评估他们是否需要更新他们的工具。不阅读`MAINTENANCE.md`的用户仍然可以在运行时查询能力并以编程方式发现新功能。

### 版本控制中的常见错误

首次应用版本规范时会出现三个错误。每个都值得命名。

第一个错误是**重用ioctl号**。曾经分配并后来退役的号码保持退役。新命令获取下一个可用号码，而不是退役命令的号码。重用号码会静默破坏编译了旧含义的旧调用者；编译器无法检测冲突，因为退役命令的头文件已被移除。

第二个错误是**为每次更改提升版本整数**。如果每个补丁都提升`MYFIRST_IOCTL_VERSION`，用户空间工具必须不断重建或版本检查失败。整数应该仅对真正的破坏性更改提升。纯添加让它保持不变。

第三个错误是**将发布字符串视为语义版本**。发布字符串是给人类看的；它可以是任何东西。模块版本整数和ioctl版本整数被程序解析，应该遵循规范（单调递增，仅因特定原因提升）。混淆两者导致混乱的版本号。

### 第4节总结

版本控制是在不破坏的情况下演进的规范。保持三个版本标识符独立、ioctl添加兼容、弃用宣布、能力位准确的驱动程序为调用者在驱动程序的长期生命周期内提供一个稳定的目标。`myfirst`驱动程序现在有一个工作的`GETCAPS` ioctl、`MAINTENANCE.md`中文档化的弃用策略，以及三个各自因自己的原因更改的版本标识符。未来开发者添加功能或退役命令所需的一切都已就位。

在下一节中，我们从驱动程序的公共表面转向其私有资源规范。在附加失败时崩溃的驱动程序是一个无法从任何错误中恢复的驱动程序。标签化goto模式是FreeBSD驱动程序如何使每个分配可逆的方式。


## 第7节：为生产使用做准备

在您的工作站上能工作的驱动程序不是准备好生产的驱动程序。生产是当代码安装在您不拥有的硬件上、由您永远不会见到的操作员引导、并期望在重启之间的几个月或几年中可预测运行时面临的条件集合。"在我这里能用"和"准备好发布"之间的距离以习惯衡量，而不是以特性衡量。本节命名这些习惯。

`myfirst`在第25章形态下功能已经与伪驱动程序将要达到的一样完整。剩余的工作不是添加功能，而是加固边缘，以便驱动程序在它无法控制的环境中生存。

### 生产就绪的心态

心态转变是：驱动程序在开发时隐式做出的每个决定必须在生产时显式做出。可调参数有默认值的地方，默认值必须是正确的默认值。sysctl可写入的地方，凌晨3点紧张操作员写入的后果必须是安全的。日志消息可能触发的地方，消息在没有开发者帮助的情况下必须有用。模块依赖另一个模块的地方，必须声明依赖关系，以便加载器不会以错误的顺序加载它们。

生产就绪不是一次性行动；它是贯穿每个决定的态度。几乎准备好生产的驱动程序通常有一两个具体差距：没有文档的可调参数、每微秒触发的日志消息、假设没人使用设备的分离路径。生产就绪的规范是找到那些具体差距并逐个关闭它们，直到驱动程序的行为在开发者不站在旁边的机器上是可预测的。

### 声明模块依赖

第一个生产习惯是显式说明模块需要什么。如果`myfirst`调用位于另一个内核模块中的函数，内核的模块加载器需要在调用之前知道依赖关系，否则内核加载`myfirst`并在第一次使用依赖项时恐慌。

机制是`MODULE_DEPEND`。第4节将其作为兼容性工具介绍；在生产中，它也是正确性工具。没有对其真实依赖的`MODULE_DEPEND`的驱动程序在大多数引导排序中偶然工作，在其他排序中神秘失败。对每个真实依赖都有`MODULE_DEPEND`的驱动程序要么正确加载，要么以清晰的错误消息拒绝加载。

对于伪驱动程序`myfirst`，目前没有真正的依赖；驱动程序只使用内核核心的符号，内核核心始终存在。第26章的USB变体将添加第一个真正的`MODULE_DEPEND`：

```c
MODULE_DEPEND(myfirst_usb, usb, 1, 1, 1);
```

三个版本号是`myfirst_usb`兼容的USB栈版本的最低、首选和最高。在加载时，内核根据此范围检查安装的USB栈版本，如果USB栈缺失或超出范围则拒绝加载`myfirst_usb`。

生产习惯是：发布前，grep驱动程序调用的每个符号，并确认每个符号要么位于内核核心，要么位于驱动程序声明依赖的模块中。缺少`MODULE_DEPEND`在引导排序改变之前可以工作，然后驱动程序在生产硬件上恐慌。

### 发布PNP信息

对于硬件驱动程序，内核的模块加载器咨询每个模块的PNP元数据来决定哪个驱动程序处理哪个设备。不发布PNP信息的USB驱动程序在手动加载时可以工作，在引导加载程序尝试为新插入的设备自动加载驱动程序时失败。修复是`MODULE_PNP_INFO`，驱动程序用它声明它处理的供应商/产品标识符：

```c
MODULE_PNP_INFO("U16:vendor;U16:product", uhub, myfirst_usb,
    myfirst_pnp_table, nitems(myfirst_pnp_table));
```

第一个字符串描述PNP表条目的格式。`uhub`是总线名；`myfirst_usb`是驱动程序名；`myfirst_pnp_table`是结构体静态数组，每个驱动程序处理的设备一个。

第25章的`myfirst`仍然是伪驱动程序，没有硬件可匹配。`MODULE_PNP_INFO`在第26章随第一个真实硬件附加投入使用。对于第25章，生产习惯只是知道该宏存在并在硬件到来时计划使用它。

### `MOD_QUIESCE`事件

内核模块事件处理器被调用四个事件之一：`MOD_LOAD`、`MOD_UNLOAD`、`MOD_SHUTDOWN`、`MOD_QUIESCE`。大多数驱动程序显式处理`MOD_LOAD`和`MOD_UNLOAD`，内核合成其他两个。对于生产驱动程序，`MOD_QUIESCE`值得关注。

`MOD_QUIESCE`是内核的问题"你现在能被卸载吗？"它在`MOD_UNLOAD`之前触发，给驱动程序干净拒绝的机会。正在操作中途的驱动程序（未完成的DMA传输、打开的文件描述符、待处理的定时器）可以从`MOD_QUIESCE`返回非零errno以拒绝卸载；内核然后不会继续到`MOD_UNLOAD`。

对于`myfirst`，停顿检查已经内置到`myfirst_detach`中：如果`sc_open_count > 0`，分离返回`EBUSY`。内核的模块加载器将该`EBUSY`传播回`kldunload(8)`，操作员看到"module myfirst is busy"。检查在正确的位置，但单独考虑`MOD_QUIESCE`而不是`MOD_UNLOAD`的规范值得命名：`MOD_QUIESCE`是"你卸载安全吗？"的问题，`MOD_UNLOAD`是"继续卸载"的命令。一些驱动程序有在`MOD_QUIESCE`中检查安全但在`MOD_UNLOAD`中获取不安全的状态；将它们分开让驱动程序可以回答问题而没有副作用。

### 发出`devctl_notify`事件

长时间运行的生产系统由`devd(8)`等守护进程监控，它们监视设备到达、离开和状态变化。内核用来通知`devd`的机制是`devctl_notify(9)`：驱动程序发出结构化事件，`devd`读取事件，`devd`采取配置的操作（运行脚本、记录消息、通知操作员）。

原型为：

```c
void devctl_notify(const char *system, const char *subsystem,
    const char *type, const char *data);
```

- `system`是顶级类别，如`"DEVFS"`、`"ACPI"`或驱动程序特定的标签。
- `subsystem`是驱动程序或子系统名称。
- `type`是短事件名称。
- `data`是守护进程解析的可选结构化数据（键=值对）。

对于`myfirst`，一个有用的生产事件是"驱动程序内消息被重写"：

```c
devctl_notify("myfirst", device_get_nameunit(sc->sc_dev),
    "MSG_CHANGED", NULL);
```

操作员通过`ioctl(fd, MYFIRSTIOC_SETMSG, buf)`写入新消息后，驱动程序发出`MSG_CHANGED`事件。`devd`规则可以匹配事件，例如发送syslog条目或通知监控守护进程：

```text
notify 0 {
    match "system"    "myfirst";
    match "type"      "MSG_CHANGED";
    action "logger -t myfirst 'message changed on $subsystem'";
};
```

这里的习惯是为驱动程序中每个有趣的事件询问：操作员是否可能想要对其做出反应？如果答案是肯定的，用精心选择的名称发出`devctl_notify`。下游工具可以基于事件构建，驱动程序不必知道这些工具是什么。

### 编写`MAINTENANCE.md`

每个生产驱动程序都应该有一个维护文件，用通俗语言描述驱动程序做什么、接受什么可调参数、公开什么sysctl、处理什么ioctl、发出什么事件以及版本历史是什么。文件位于仓库中的源代码旁边；由操作员、新开发者、安全审查员和六个月后的作者阅读。

`MAINTENANCE.md`的具体骨架：

```text
# myfirst

A demonstration character driver that carries the book's running
example.  This file is the operator-facing reference.

## Overview

myfirst registers a pseudo-device at /dev/myfirst0 and serves a
read-write message buffer, a set of ioctls, a sysctl tree, and a
configurable debug-class logger.

## Tunables

- hw.myfirst.debug_mask_default (int, default 0)
    Initial value of dev.myfirst.<unit>.debug.mask.
- hw.myfirst.timeout_sec (int, default 5)
    Initial value of dev.myfirst.<unit>.timeout_sec.
- hw.myfirst.max_retries (int, default 3)
    Initial value of dev.myfirst.<unit>.max_retries.
- hw.myfirst.log_ratelimit_pps (int, default 10)
    Initial rate-limit ceiling (prints per second per class).

## Sysctls

All sysctls live under dev.myfirst.<unit>.

Read-only: version, open_count, total_reads, total_writes,
message, message_len.

Read-write: debug.mask (mirror of debug_mask_default), timeout_sec,
max_retries, log_ratelimit_pps.

## Ioctls

Defined in myfirst_ioctl.h.  Command magic 'M'.

- MYFIRSTIOC_GETVER (0): returns MYFIRST_IOCTL_VERSION.
- MYFIRSTIOC_RESET  (1): zeros read/write counters.
- MYFIRSTIOC_GETMSG (2): reads the in-driver message.
- MYFIRSTIOC_SETMSG (3): writes the in-driver message.
- MYFIRSTIOC_GETCAPS (5): returns MYF_CAP_* bitmask.

Command 4 was reserved during Chapter 23 draft work and retired
before release.  Do not reuse the number.

## Events

Emitted through devctl_notify(9).

- system=myfirst subsystem=<unit> type=MSG_CHANGED
    The operator-visible message was rewritten.

## Version History

See Change Log below.

## Change Log

### 1.8-maintenance
- Added MYFIRSTIOC_GETCAPS (command 5).
- Added tunables for timeout_sec, max_retries, log_ratelimit_pps.
- Added rate-limited logging via ppsratecheck(9).
- Added devctl_notify for MSG_CHANGED.
- No breaking changes from 1.7.

### 1.7-integration
- First end-to-end integration of ioctl, sysctl, debug.
- Introduced MYFIRSTIOC_{GETVER,RESET,GETMSG,SETMSG}.

### 1.6-debug
- Added DPRINTF framework and SDT probes.
```

该文件并不华丽。它是一个随每次版本升级保持更新的参考，是操作员的单一事实来源。

生产习惯是：对驱动程序可见接口的每个更改（新可调参数、新sysctl、新ioctl、新事件、行为更改）在`MAINTENANCE.md`中有对应条目。文件永远不会落后于代码。`MAINTENANCE.md`过时的驱动程序是用户在猜测的驱动程序；`MAINTENANCE.md`当前的驱动程序是用户可以自助的驱动程序。

### `devd`规则集

`devd(8)`规则告诉守护进程如何对内核事件做出反应。对于`myfirst`的生产部署，最小的规则集应确保重要事件到达操作员：

```console
# /etc/devd/myfirst.conf
#
# devd rules for the myfirst driver.  Drop this file into
# /etc/devd/ and restart devd(8) for the rules to take effect.

notify 0 {
    match "system"    "myfirst";
    match "type"      "MSG_CHANGED";
    action "logger -t myfirst 'message changed on $subsystem'";
};

# Future: match attach/detach events once Chapter 26's USB variant
# starts emitting them.
```

文件很短。它声明一条规则，匹配特定事件，采取特定行动。在生产中，这样的文件增长以匹配更多事件，触发更多行动，在某些部署中通知监视驱动程序异常的监控系统。

在驱动程序的仓库中包含草稿`devd.conf`使操作员易于采用。他们复制文件，调整行动，驱动程序的事件在第一天就集成到站点的监控中。

### 日志：支持工程师的朋友

生产驱动程序的日志消息由无法访问源代码且无法按需重现问题的支持工程师阅读。使日志消息对支持工程师有用的规则与使日志消息对开发者有用的规则不同。

阅读自己日志消息的开发者可以依赖支持工程师没有的上下文。支持工程师不能问"哪个附加？"或"哪个设备？"或"触发时`error`是什么？"答案必须已经在消息中。

生产习惯是审计驱动程序中的每个日志消息并问三个问题：

1. **消息是否命名其设备？** `device_printf(dev, ...)`在输出前加上设备名称单元；裸`printf`不会。每条不是来自`MOD_LOAD`（那里还没有设备）的消息应该是`device_printf`。

2. **消息是否包含相关的数字上下文？** "Failed to allocate"没有用。"Failed to allocate: error 12 (ENOMEM)"有用。"Failed to allocate a timer: error 12"更好。

3. **消息是否以适当的速率出现？** 第1节涵盖限速。最后一遍是确保每个可能在循环中触发的消息要么被限速，要么证明是一次性的。

满足这三个问题的日志消息带着足够信息到达支持工程师以提交有用的错误报告。任一问题失败的消息浪费操作员的时间和开发者的时间。

### 优雅处理总线附加/分离

生产驱动程序，特别是热插拔驱动程序，必须在不泄漏的情况下处理重复的附加和分离循环。第5节标签化清理模式规范是答案的一部分；另一部分是确认重复附加/分离实际上有效。本章末尾的实验室演练一个回归脚本，它连续加载、卸载和重新加载驱动程序100次，并验证模块的内存占用没有增长。

通过100次循环测试的驱动程序是将在生产硬件上存活一个月热插拔事件的驱动程序。100次循环测试失败的驱动程序有泄漏，随着时间的推移会表现为缓慢的内存增长或内核耗尽某些有限资源（sysctl OID、cdev次设备号、devclass条目）。

测试运行简单，价值不成比例。将其作为驱动程序发布前检查清单的一部分。

### 处理意外的操作员操作

操作员会犯错。他们在测试程序正在从`/dev/myfirst0`读取时运行`kldunload myfirst`。他们将`dev.myfirst.0.debug.mask`设置为一次启用每个类的值。他们复制`MAINTENANCE.md`并跳过可调参数部分。生产驱动程序必须容忍这些操作而不崩溃、不损坏状态、不使系统处于损坏配置。

对于每个公开的接口，生产习惯是问：我能想象的最糟糕的操作员操作序列是什么，驱动程序能否存活？

- 在文件描述符打开时`kldunload`：`myfirst_detach`返回`EBUSY`。操作员看到"module busy"。驱动程序不变。
- 可写sysctl被设置为超出范围的值：sysctl处理程序限制值或返回`EINVAL`。驱动程序内部状态不变。
- 带有长于缓冲区的消息的`MYFIRSTIOC_SETMSG`：`strlcpy`截断。复制正确；截断在`message_len`中可见。
- 并发一对`MYFIRSTIOC_SETMSG`调用：softc互斥锁串行化它们。第二个运行的获胜；两者都成功。

如果这些操作中的任何一个产生崩溃、损坏或不一致状态，驱动程序就没有准备好生产。修复总是相同的：添加缺少的保护，重新开始测试，并添加记录不变式的注释。

### 生产就绪检查清单

本节的习惯适合开发者发布前可以遍历的简短检查清单：

```text
myfirst production readiness
----------------------------

[  ] MODULE_DEPEND declared for every real dependency.
[  ] MODULE_PNP_INFO declared if the driver binds to hardware.
[  ] MOD_QUIESCE answers "can you unload?" without side effects.
[  ] devctl_notify emitted for operator-relevant events.
[  ] MAINTENANCE.md current: tunables, sysctls, ioctls, events.
[  ] devd.conf snippet included with the driver.
[  ] Every log message is device_printf, includes errno,
     and is rate-limited if it can fire in a loop.
[  ] attach/detach survives 100 load/unload cycles.
[  ] sysctls reject out-of-range values.
[  ] ioctl payload is bounds-checked.
[  ] Failure paths exercised via deliberate injection.
[  ] Versioning discipline: MYFIRST_VERSION, MODULE_VERSION,
     MYFIRST_IOCTL_VERSION each bumped for their own reason.
```

列表故意简短。十二项，大多数已由前面章节介绍的习惯解决。勾选每个框的驱动程序已准备好由永远不会遇见您的人安装。

### `myfirst`驱动程序涵盖什么

在第25章结束时对`myfirst`运行检查清单给出以下状态。

`MODULE_DEPEND`不是必需的，因为驱动程序没有子系统依赖；这在`MAINTENANCE.md`中显式注明。

`MODULE_PNP_INFO`不是必需的，因为驱动程序不绑定到硬件；这也记录在`MAINTENANCE.md`中。

`MOD_QUIESCE`由`myfirst_detach`中的`sc_open_count`检查回答；此版本没有添加专用`MOD_QUIESCE`处理程序，因为语义相同。

`devctl_notify`在`MYFIRSTIOC_SETMSG`上发出，事件类型为`MSG_CHANGED`。

`MAINTENANCE.md`在示例目录中发布，包含可调参数、sysctl、ioctl、事件和1.8-maintenance的变更日志条目。

`devd.conf`片段与`MAINTENANCE.md`一起发布，演示单个`MSG_CHANGED`规则。

每条日志消息通过`device_printf`（或包装`device_printf`的`DPRINTF`）发出；在热路径上触发的每条消息都包装在`DLOG_RL`中。

附加/分离回归脚本（见实验室）运行100次循环而不增长内核内存占用。

`timeout_sec`、`max_retries`和`log_ratelimit_pps`的sysctl各自在其处理程序中拒绝超出范围的值。

ioctl有效负载在结构级别由内核的ioctl框架边界检查（`_IOR`、`_IOW`、`_IOWR`声明精确大小），在字符串长度重要的驱动程序内部也进行检查。

失败注入点在示例中由条件`#ifdef`标记；每个标签在开发中至少到达过一次。

版本标识符各有自己的规则：字符串升级，模块整数升级，ioctl整数不变，因为添加是向后兼容的。

十二项检查，十二项结果。驱动程序准备好进入下一章。

### 第7节总结

生产是将有趣代码与可发布代码分开的安静标准。这里命名的规范并不华丽；它们是使驱动程序在远离编写它的开发者那里部署时保持工作的具体事情。`myfirst`已经经历了五章教学内容，现在佩戴着让它在书本之外存活的马具。

在下一节中，我们将转向两个让驱动程序在特定生命周期点运行代码而无需手动连接的内核基础设施：用于启动时初始化的`SYSINIT(9)`和用于运行时通知的`EVENTHANDLER(9)`。这些是本书在将所有内容应用于真实总线之前的第26章之前介绍的最后两件FreeBSD工具包。


## 第5节：管理失败路径中的资源

每个attach例程都是一系列有序的获取。它分配一个锁、创建一个cdev、在设备上挂起sysctl树、可能注册一个事件处理程序或定时器，对于更复杂的驱动程序，它分配总线资源、映射I/O窗口、附加中断、设置DMA。每个获取都可能失败。在失败之前成功的每个获取都必须以相反的顺序释放，否则内核泄漏内存、泄漏锁、泄漏cdev，在最坏的情况下保持设备节点存活，其中包含陈旧的指针。

`myfirst`驱动程序自第17章以来一直在逐节增长其附加路径。附加开始很小：一个锁和一个cdev。第24章添加了sysctl树。第25章即将添加限速状态、可调参数fetch的默认值和一个或两个计数器。这些资源获取的顺序现在对清理路径很重要。每个新获取都必须知道它在展开顺序中属于哪里，展开本身必须结构化，以便下周添加新资源不会强制重写attach函数。

第20章非正式地介绍了该模式；本节给它一个名称、词汇和足够强的规范，以经受完整的第25章形状的`myfirst_attach`。

### 问题：嵌套`if`路径不可扩展

attach例程的朴素形状是嵌套`if`语句的阶梯。每个成功条件包含下一步。每个失败返回。问题是每个失败都必须展开之前步骤已经做的任何事情，展开代码在阶梯的每一级都重复：

```c
/*
 * 朴素attach。不要以这种方式编写驱动程序。此示例展示
 * 嵌套if模式如何迫使在每一级重复清理
 * 以及为什么一旦第四个资源被添加到链中
 * 它就变得不可维护。
 */
static int
myfirst_attach_bad(device_t dev)
{
	struct myfirst_softc *sc = device_get_softc(dev);
	struct make_dev_args args;
	int error;

	sc->sc_dev = dev;
	mtx_init(&sc->sc_mtx, "myfirst", NULL, MTX_DEF);

	make_dev_args_init(&args);
	args.mda_devsw   = &myfirst_cdevsw;
	args.mda_uid     = UID_ROOT;
	args.mda_gid     = GID_WHEEL;
	args.mda_mode    = 0660;
	args.mda_si_drv1 = sc;
	args.mda_unit    = device_get_unit(dev);

	error = make_dev_s(&args, &sc->sc_cdev, "myfirst%d",
	    device_get_unit(dev));
	if (error == 0) {
		myfirst_sysctl_attach(sc);
		if (myfirst_log_attach(sc) == 0) {
			/* 所有资源已持有；我们成功 */
			return (0);
		} else {
			/* 日志分配失败：撤销sysctl和cdev */
			/* 但等等，sysctl由Newbus拥有，所以跳过它 */
			destroy_dev(sc->sc_cdev);
			mtx_destroy(&sc->sc_mtx);
			return (ENOMEM);
		}
	} else {
		mtx_destroy(&sc->sc_mtx);
		return (error);
	}
}
```

即使在这个小示例中，展开逻辑出现在两个不同的地方，读者必须阅读分支才能知道在每个点获取了哪些资源，添加第四个资源会强制另一级嵌套和另一个重复的清理块。真实驱动程序有七八个资源。像`if_em`在`/usr/src/sys/dev/e1000/if_em.c`的驱动程序有十几个以上。在那里嵌套`if`不是一个选项。

嵌套模式的失败模式不是理论上的。旧FreeBSD驱动程序中的一个常见错误模式是在清理分支之一中缺少`mtx_destroy`或缺少`bus_release_resource`：一个分支销毁了锁，另一个忘记了。每个分支都是犯错的机会，错误只有在那个特定失败触发时才显示，这意味着它通常直到客户报告设备未能附加的机器上的panic才显示。

### `goto fail;`模式

FreeBSD对嵌套清理问题的答案是标签化goto模式。attach函数写成获取的线性序列。每个可能失败的获取后面跟着一个测试，要么成功时继续执行，要么失败时跳转到清理标签。清理标签从最多获取到最少获取排序。每个标签释放在该点持有的资源，然后继续到下一个标签。函数以成功时的单个`return (0)`和清理链底部的单个`return (error)`结束：

```c
static int
myfirst_attach(device_t dev)
{
	struct myfirst_softc *sc = device_get_softc(dev);
	struct make_dev_args args;
	int error;

	/* 资源1：softc基础。不能失败。 */
	sc->sc_dev = dev;

	/* 资源2：锁。DEF互斥锁不能失败。 */
	mtx_init(&sc->sc_mtx, "myfirst", NULL, MTX_DEF);

	/* 资源3：cdev。可能失败。 */
	make_dev_args_init(&args);
	args.mda_devsw   = &myfirst_cdevsw;
	args.mda_uid     = UID_ROOT;
	args.mda_gid     = GID_WHEEL;
	args.mda_mode    = 0660;
	args.mda_si_drv1 = sc;
	args.mda_unit    = device_get_unit(dev);

	error = make_dev_s(&args, &sc->sc_cdev, "myfirst%d",
	    device_get_unit(dev));
	if (error != 0)
		goto fail_mtx;

	/* 资源4：sysctl树。不能失败（Newbus拥有它）。 */
	myfirst_sysctl_attach(sc);

	/* 资源5：日志状态。可能失败。 */
	error = myfirst_log_attach(sc);
	if (error != 0)
		goto fail_cdev;

	/* 所有资源已持有。宣布并返回。 */
	DPRINTF(sc, MYF_DBG_INIT,
	    "attach: 版本1.8-maintenance就绪\n");
	return (0);

fail_cdev:
	destroy_dev(sc->sc_cdev);
fail_mtx:
	mtx_destroy(&sc->sc_mtx);
	return (error);
}
```

从头到尾阅读函数。每一步都是一个资源获取。每个失败检查是一个两行块：如果获取失败，跳转到以前获取资源命名的标签。底部的标签以相反顺序释放资源并继续到下一个标签。最后的`return (error)`返回失败的获取的errno。

这种形状可扩展。添加第六个资源意味着在顶部添加一个获取块，在底部添加一个`goto`目标，和一行清理代码。没有嵌套，没有重复，没有分支树。管辖附加路径的相同规则管辖附加路径的每个未来添加：获取、测试、跳转到上一个标签、以相反顺序释放。

### 为什么线性展开是正确的形状

标签化goto模式的价值不仅仅是风格上的。它直接映射到结构属性，即附加序列是一个资源栈，清理是该栈上的弹出操作。

栈有三个易于陈述但易于违反的属性。首先，资源以获取相反的顺序释放。其次，失败的获取不向栈添加资源，因此清理从以前获取的资源开始，而不是从刚刚失败的资源开始。第三，栈上的每个资源恰好释放一次：不是零次，不是两次。

这些属性中的每一个在`goto fail;`模式中都有一个可见的相关性。清理标签以获取相反的顺序出现在文件中：最后获取的清理标签位于清理链顶部。失败的获取跳转到以前获取命名的标签，而不是自己；标签的名称字面上是现在必须撤销的资源名称。而且因为每个标签继续到下一个，每个资源恰好出现在一个标签中，每个资源在每个失败路径上恰好释放一次。

栈规范使模式健壮。如果读者想要审计清理路径的正确性，他们不必阅读分支。他们只需计算标签，计算获取，并比较。

### 标签命名约定

FreeBSD驱动程序中的标签传统上以`fail_`开头，后跟即将撤销的资源名称。资源名称与softc中的字段名称或调用以获取它的函数名称匹配。树中常见的模式：

- `fail_mtx`撤销`mtx_init`
- `fail_sx`撤销`sx_init`
- `fail_cdev`撤销`make_dev_s`
- `fail_ires`撤销IRQ的`bus_alloc_resource`
- `fail_mres`撤销内存窗口的`bus_alloc_resource`
- `fail_intr`撤销`bus_setup_intr`
- `fail_dma_tag`撤销`bus_dma_tag_create`
- `fail_log`撤销驱动程序私有分配（`myfirst`中的限速块）

一些旧驱动程序使用编号标签（`fail1`、`fail2`、`fail3`）。编号标签是合法的但较差：在序列中间添加资源会强制重新编号插入点之后的每个标签，标签号不告诉读者正在清理哪个资源。命名标签优雅地经受插入并自我文档化。

无论驱动程序选择什么约定，它应该在其所有文件中保持一致。`myfirst`从本章开始为每个attach函数使用`fail_<resource>`约定。

### 继续规则

每个清理链必须遵守的单一规则是每个清理标签继续到下一个。链中间的孤立`return`或缺少的标签会跳过本应释放的资源的清理。编译器不警告任一错误。

考虑如果开发人员编辑清理链并意外写入此内容会发生什么：

```c
fail_cdev:
	destroy_dev(sc->sc_cdev);
	return (error);          /* BUG：跳过mtx_destroy。 */
fail_mtx:
	mtx_destroy(&sc->sc_mtx);
	return (error);
```

第一个`return`阻止`mtx_destroy`在`fail_cdev`路径上运行。锁被泄漏。内核的witness代码不会抱怨，因为泄漏的锁从未再次获取。泄漏持续到机器重新启动。它在正常操作中不可见，仅在驱动程序反复附加和失败的系统上显示为缓慢的内存膨胀（例如，热插拔设备）。

防止这种错误的方法是在底部用单个`return`编写清理链，中间没有中间返回。中间的标签仅包含其资源的清理调用。继续是默认和预期的行为：

```c
fail_cdev:
	destroy_dev(sc->sc_cdev);
fail_mtx:
	mtx_destroy(&sc->sc_mtx);
	return (error);
```

审计链的读者将其读作简单的列表：销毁cdev、销毁锁、返回。没有要跟随的分支，添加标签意味着添加一行清理代码和可选的单个新目标。

### 成功路径看起来像什么

attach函数以单个`return (0)`成功，放置在第一个清理标签之前。这是每个获取都已成功且不需要清理的点。`return (0)`在视觉上将获取链与清理链分开：它上面的所有内容都是获取，它下面的所有内容都是清理。

一些驱动程序忘记此分离，从最后获取继续到第一个清理标签，释放它们刚刚获取的资源。缺少孤立的`return (0)`是产生此错误的最简单方式：

```c
	/* 资源N：最后的获取。 */
	...

	/* 忘记在这里放一个return。 */

fail_cdev:
	destroy_dev(sc->sc_cdev);
```

没有`return (0)`，控制从每个成功的attach继续到`fail_cdev`，在成功路径上销毁cdev。驱动程序然后报告attach失败，因为`error`为零，内核看到成功的返回，但它刚刚创建的cdev已消失。结果是几秒钟后消失的设备节点。调试这需要注意到附加消息打印但设备不响应；在繁忙的日志中不是一个容易发现的错误。

防御是规范。每个attach函数以单独一行上的`return (0);`结束其获取链，后跟空行，后跟清理标签。没有例外。像`igor`这样的linter或审阅者的眼睛在形状总是相同时快速捕捉违规。

### 当获取不能失败时

某些获取不能失败。默认样式互斥锁的`mtx_init`不能返回错误。`sx_init`不能。`callout_init_mtx`不能。`SYSCTL_ADD_*`调用不能返回驱动程序预期检查的错误（那里的失败是内核内部问题，不是驱动程序问题）。

对于不能失败的获取，没有goto。获取后面是下一步，没有测试。获取的清理标签仍然需要，因为如果后来的获取失败，清理链必须释放资源：

```c
	mtx_init(&sc->sc_mtx, "myfirst", NULL, MTX_DEF);

	error = make_dev_s(&args, &sc->sc_cdev, ...);
	if (error != 0)
		goto fail_mtx;       /* 撤销锁。 */
```

`fail_mtx`存在，即使`mtx_init`本身没有失败路径，因为如果下面的任何东西失败，锁仍然需要销毁。

模式成立：每个获取的资源都有一个标签，无论其获取是否可能失败。

### 减少重复的助手

当几个获取共享相同的形状（分配、检查、错误时goto）时，将它们隐藏在助手函数中是诱人的。助手的工作是合并获取和检查；调用者只看到一行`if (error != 0) goto fail_X;`。这没问题，只要助手遵循相同的规范：失败时，它不释放它部分获取的任何东西，并返回有意义的errno，以便调用者的goto目标可以依赖它。

在`myfirst`中，第5节的伴随示例引入了一个名为`myfirst_log_attach`的助手，它分配限速状态，初始化其字段，成功时返回0，失败时返回非零errno。attach函数用一行调用它：

```c
	error = myfirst_log_attach(sc);
	if (error != 0)
		goto fail_cdev;
```

助手本身内部遵循相同的模式。如果它分配两个资源而第二个失败，助手在返回之前展开第一个。调用者将助手视为单个原子获取：它要么完全成功，要么完全失败，调用者永远不必担心助手的中间状态。

然而，过于急于简化的助手会破坏模式。分配资源并将其存储到softc的助手是好的。分配资源、将其存储到softc、并在错误时也释放它的助手是不好的：调用者的清理标签也会尝试释放它，导致双重释放。规则是获取助手要么成功并将资源留在softc中，要么失败并保持softc不变。它们不会半成功。

### Detach作为Attach的镜像

detach例程是成功attach的清理链。它必须以相反顺序释放恰好attach获取的资源。detach函数的形状是移除了标签并删除了获取的清理链的形状：

```c
static int
myfirst_detach(device_t dev)
{
	struct myfirst_softc *sc = device_get_softc(dev);

	/* 首先检查是否繁忙。 */
	mtx_lock(&sc->sc_mtx);
	if (sc->sc_open_count > 0) {
		mtx_unlock(&sc->sc_mtx);
		return (EBUSY);
	}
	mtx_unlock(&sc->sc_mtx);

	/* 以attach相反的顺序释放资源。 */
	myfirst_log_detach(sc);
	destroy_dev(sc->sc_cdev);
	/* Sysctl由Newbus在detach返回后清理。 */
	mtx_destroy(&sc->sc_mtx);

	return (0);
}
```

与attach函数并排阅读，对应关系是精确的。attach中命名的每个资源在detach中都有释放。添加到attach的每个新获取在detach中都有匹配的添加。审阅审计添加新资源到驱动程序的补丁应该能够在diff中找到两个添加，一个在attach链中，一个在detach链中；仅添加到attach的diff是不完整的。

触及attach链时一个有用的规范是在相邻的编辑器缓冲区中打开detach函数，并在添加获取后立即添加释放。这是确保两个函数保持同步的最简单方法：它们作为一个单一操作一起编辑。

### 用于测试的故意失败注入

清理链仅在每个标签可到达时才正确。唯一确定的方法是故意触发每个失败路径并观察驱动程序之后干净地卸载。等待真实硬件失败来测试路径不是一个策略：大多数路径在真实生活中从未被测试。

用于这种测试的工具是故意失败注入。开发人员在中途向attach链添加临时的`goto`或临时的早期返回，并确认当注入失败触发时驱动程序的资源都被释放。

`myfirst`的最小模式：

```c
#ifdef MYFIRST_DEBUG_INJECT_FAIL_CDEV
	error = ENOMEM;
	goto fail_cdev;
#endif
```

用`-DMYFIRST_DEBUG_INJECT_FAIL_CDEV`编译驱动程序并加载它。attach返回`ENOMEM`。`kldstat`显示没有残留。`dmesg`显示附加失败，没有关于泄漏锁或泄漏资源的内核投诉。卸载模块，移除define，重新编译，驱动程序恢复正常。

每个标签这样做一次，依次：

1. 在锁初始化后立即注入失败。确认只有锁被释放。
2. 在cdev创建后立即注入失败。确认cdev和锁被释放。
3. 在sysctl树构建后立即注入失败。确认cdev和锁被释放，sysctl OID消失。
4. 在日志状态初始化后立即注入失败。确认到那点获取的每个资源都被释放。

如果任何注入留下残留，清理链有错误。修复错误，重新运行注入，继续。

这第一次是不舒服的工作，之后是令人欣慰的。每个失败路径都被测试过一次的驱动程序是失败路径将在代码演进时继续工作的驱动程序。失败路径从未被测试的驱动程序是有将在最糟糕的时刻出现的潜在错误的驱动程序。

`examples/part-05/ch25-advanced/`下的伴随示例`ex05-failure-injection/`包含一个版本的`myfirst_attach`，每个失败注入点由注释的`#define`标记。章节末尾的实验依次通过每个注入。

### 第25章的完整`myfirst_attach`

将第5节的所有内容与第25章添加（日志状态、可调参数fetch、能力位掩码）放在一起，最终attach函数看起来像这样：

```c
static int
myfirst_attach(device_t dev)
{
	struct myfirst_softc *sc = device_get_softc(dev);
	struct make_dev_args args;
	int error;

	/*
	 * 阶段1：softc基础。不能失败。为一致性记录；
	 * 不需要清理标签，因为还没有持有资源。
	 */
	sc->sc_dev = dev;

	/*
	 * 阶段2：锁。MTX_DEF不能失败，但需要一个标签，
	 * 因为这行下面的任何东西都可能失败，必须释放它。
	 */
	mtx_init(&sc->sc_mtx, "myfirst", NULL, MTX_DEF);

	/*
	 * 阶段3：用默认值预填充softc，然后允许
	 * 引导时可调参数覆盖。这里没有分配，
	 * 所以不需要清理。默认值来自第3节
	 * 可调参数集。
	 */
	strlcpy(sc->sc_msg, "Hello from myfirst", sizeof(sc->sc_msg));
	sc->sc_msglen = strlen(sc->sc_msg);
	sc->sc_open_count = 0;
	sc->sc_total_reads = 0;
	sc->sc_total_writes = 0;
	sc->sc_debug = 0;
	sc->sc_timeout_sec = 5;
	sc->sc_max_retries = 3;
	sc->sc_log_pps = MYF_RL_DEFAULT_PPS;

	TUNABLE_INT_FETCH("hw.myfirst.debug_mask_default",
	    &sc->sc_debug);
	TUNABLE_INT_FETCH("hw.myfirst.timeout_sec",
	    &sc->sc_timeout_sec);
	TUNABLE_INT_FETCH("hw.myfirst.max_retries",
	    &sc->sc_max_retries);
	TUNABLE_INT_FETCH("hw.myfirst.log_ratelimit_pps",
	    &sc->sc_log_pps);

	/*
	 * 阶段4：cdev。可能失败。失败时，
	 * 释放锁并返回make_dev_s的错误。
	 */
	make_dev_args_init(&args);
	args.mda_devsw   = &myfirst_cdevsw;
	args.mda_uid     = UID_ROOT;
	args.mda_gid     = GID_WHEEL;
	args.mda_mode    = 0660;
	args.mda_si_drv1 = sc;
	args.mda_unit    = device_get_unit(dev);

	error = make_dev_s(&args, &sc->sc_cdev, "myfirst%d",
	    device_get_unit(dev));
	if (error != 0)
		goto fail_mtx;

	/*
	 * 阶段5：sysctl树。不能失败。框架拥有
	 * 上下文，所以不需要特定的清理标签。
	 */
	myfirst_sysctl_attach(sc);

	/*
	 * 阶段6：限速和计数器状态。如果内存
	 * 分配失败可能失败。失败时，
	 * 释放cdev和锁。
	 */
	error = myfirst_log_attach(sc);
	if (error != 0)
		goto fail_cdev;

	DPRINTF(sc, MYF_DBG_INIT,
	    "attach: 版本1.8-maintenance完成\n");
	return (0);

fail_cdev:
	destroy_dev(sc->sc_cdev);
fail_mtx:
	mtx_destroy(&sc->sc_mtx);
	return (error);
}
```

每个资源都被计算。每个失败路径都是线性的。函数在从获取到清理的过渡处有单个成功返回，在清理链底部有单个失败返回。下一章添加第七个资源是三行操作：一个新获取块、一个新标签、一个新清理行。

### 失败路径中的常见错误

几个失败路径错误值得命名一次，以便在别人的代码或审阅中出现时可以识别。

第一个错误是**缺少标签**。开发人员添加新的资源获取但忘记添加其清理标签。编译器不警告；链从外面看起来很好；但在新获取后的失败时，下面的所有内容的清理被跳过。规则是每个获取都有一个标签。即使获取不能失败，它仍然需要一个标签，以便后来的获取可以到达它。

第二个错误是**两次释放资源**。开发人员在助手内部添加本地清理，忘记调用者的清理标签也释放资源。助手释放一次，调用者再次释放，要么内核panic（对于内存），要么witness代码抱怨（对于锁）。规则是只有一个方拥有每个资源的清理。如果助手获取资源并将其存储到softc，助手不为调用者清理它；它要么成功，要么保持softc不变。

第三个错误是**依赖`NULL`测试**。开发人员编写这样的清理链：

```c
fail_cdev:
	if (sc->sc_cdev != NULL)
		destroy_dev(sc->sc_cdev);
fail_mtx:
	if (sc->sc_mtx_initialised)
		mtx_destroy(&sc->sc_mtx);
```

逻辑是：如果资源实际上没有获取，跳过清理。意图是防御性的；效果是隐藏错误。如果`NULL`检查存在是因为清理可能在资源未获取的状态下到达，链是错误的：goto目标应该是不同的标签。正确的行为是使清理标签不可到达，除非资源实际上被获取。可以在任一状态下到达的标签是混乱获取顺序的症状，`NULL`检查只是掩盖它。

第四个错误是**为非错误流程使用`goto`**。attach函数中的`goto`严格用于失败路径。在某些非错误条件下跳过获取链的一部分的`goto`违反线性清理不变式：清理链假设每个标签对应一个已被获取的资源，跳过获取的`goto`破坏该假设。如果需要条件获取，使用围绕获取本身的`if`，而不是围绕它的`goto`。

### 第5节总结

Attach和detach是将驱动程序保持到内核的缝合。正确的attach是获取的线性栈；正确的detach是反向弹出的栈。标签化goto模式是FreeBSD驱动程序如何在C中编码该栈，而不购买其他操作系统的内核（C++析构函数、Go defer、Rust Drop）。它不华丽，它可扩展：有十几个资源的驱动程序读起来恰好像有两个的驱动程序一样，添加新资源的规则总是相同的。

`myfirst`attach函数现在有四个失败标签，以及获取、成功返回和清理之间的清晰分离。第26章添加的每个新资源都将适合这种形状。

在下一节中，我们退后一步，从任何单个函数，看看增长的驱动程序如何分布在文件中。承载每个函数的一个`myfirst.c`已经承载我们八章；现在是将其拆分为专注单元的时候，以便驱动程序的结构在文件级别可见。


## 第6节：模块化和关注点分离

到第24章结束时，`myfirst`驱动程序已经超出了一个源文件可以舒适承载的范围。文件形状是`myfirst.c`加上`myfirst_debug.c`、`myfirst_ioctl.c`和`myfirst_sysctl.c`；`myfirst.c`仍然承载cdevsw、读/写回调、打开/关闭回调、attach和detach例程以及模块粘合代码。这对于教学来说是好的，因为每个添加都落在一个足够小的文件中，读者可以牢记在脑海中。对于一个拥有ioctl表面、sysctl树、调试框架、限速日志助手、能力位掩码、版本控制规范和标签清理attach例程的驱动程序来说，这不再是好的了。有那么多的文件会变得痛苦地阅读、痛苦地diff、痛苦地移交给新的贡献者。

第6节是关于另一个方向。它不引入新行为；第5节结束时存在的每个函数在第6节结束时仍然存在。改变的是文件布局和各个部分之间的边界线。目标是驱动程序的结构你可以从`ls`中理解，其各个文件各自回答一个单一问题。

### 为什么要拆分文件

一个自包含驱动程序的诱惑是将所有东西保持在一个文件中。单个`myfirst.c`易于定位、易于grep、易于复制到tarball。拆分感觉像是官僚主义。拆分的论点出现在驱动程序跨越三个阈值之一时。

第一个阈值是**理解**。打开`myfirst.c`的读者应该能在几秒钟内找到他们要找的东西。一个1200行有八个不相关职责的文件难以导航；读者必须滚动过cdevsw才能找到sysctl，滚动过sysctl才能找到ioctl，滚动过ioctl才能找到attach例程。每次切换主题，他们必须重新加载心智上下文。使用单独的文件，主题是文件名：`myfirst_ioctl.c`是关于ioctl的，`myfirst_sysctl.c`是关于sysctl的，`myfirst.c`是关于生命周期的。

第二个阈值是**独立性**。两个不相关的更改不应该修改同一个文件。当一个开发人员添加sysctl而另一个开发人员添加ioctl时，他们的补丁不应该竞争`myfirst.c`的相同行。小的、专注的文件让两个更改并行落地，没有合并冲突，没有错误在一个更改中意外触及另一个更改的风险。

第三个阈值是**可测试性和可重用性**。驱动程序的日志基础设施、其ioctl分发和其sysctl树通常对同一项目内的多个驱动程序有用。将它们保持在具有清晰接口的单独文件中使它们成为以后共享的候选者。存在于单个文件中的驱动程序无法轻易共享任何东西；提取意味着复制和手动重命名，这是一个容易出错的操作。

第25章结束时的`myfirst`已经跨越了所有三个阈值。拆分文件是保持驱动程序在接下来的十章中健康的维护行为。

### `myfirst`的文件布局

提议的布局是第25章最终示例目录中Makefile使用的：

```text
myfirst.h          - 公共类型和常量（softc、SRB、状态位）。
myfirst.c          - 模块粘合、cdevsw、devclass、模块事件。
myfirst_bus.c      - Newbus方法和device_identify。
myfirst_cdev.c     - open/close/read/write回调；无ioctl。
myfirst_ioctl.h    - ioctl命令号和有效载荷结构。
myfirst_ioctl.c    - myfirst_ioctl switch和助手。
myfirst_sysctl.c   - myfirst_sysctl_attach和处理程序。
myfirst_debug.h    - DPRINTF/DLOG/DLOG_RL宏和类别位。
myfirst_debug.c    - 调试类别枚举（如果有任何超出行的）。
myfirst_log.h      - 限速状态结构。
myfirst_log.c      - myfirst_log_attach/detach和助手。
```

七个`.c`文件和四个`.h`文件。每个`.c`文件有一个由其文件名命名的主题。头文件声明跨文件边界的接口。没有文件导入另一个文件的内部；每个跨文件引用都通过头文件。

乍看之下，这看起来比驱动程序需要的文件多。不是的。每个文件有一个特定的职责，与之配套的头文件是一到三打声明的行。累积大小与单文件版本相同；结构明显更清晰。

### 单一职责规则

管辖拆分的规则是单一职责规则：每个文件回答关于驱动程序的一个问题。

- `myfirst.c`回答：这个模块如何附加到内核并连接其各个部分？
- `myfirst_bus.c`回答：Newbus如何发现和实例化我的驱动程序？
- `myfirst_cdev.c`回答：驱动程序如何服务于open/close/read/write？
- `myfirst_ioctl.c`回答：驱动程序如何处理其头文件声明的命令？
- `myfirst_sysctl.c`回答：驱动程序如何将其状态暴露给`sysctl(8)`？
- `myfirst_debug.c`回答：调试消息如何分类和限速？
- `myfirst_log.c`回答：限速状态如何初始化和释放？

一个更改是否属于给定文件的测试是答案测试。如果更改不回答文件的问题，它属于别处。新sysctl不属于`myfirst_ioctl.c`；新ioctl不属于`myfirst_sysctl.c`；新读回调变体不属于`myfirst.c`。规则是显式的，应用它的审阅者拒绝将东西放在错误文件中的补丁。

将规则应用于现有的第24章形状给出第25章形状。

### 公共与私有头文件

头文件承载文件之间的接口。拆分其`.c`文件的驱动程序必须决定，对于每个声明，它属于公共头文件还是私有头文件。

**公共头文件**包含对多个`.c`文件可见的类型和常量。`myfirst.h`是驱动程序的主要公共头文件。它声明：

- `struct myfirst_softc`定义（每个`.c`文件都需要它）。
- 出现在多个文件中的常量（调试类别位、softc字段大小）。
- 跨文件边界调用的函数原型（`myfirst_sysctl_attach`、`myfirst_log_attach`、`myfirst_log_ratelimited_printf`、`myfirst_ioctl`）。

**私有头文件**承载仅被一个`.c`文件需要的声明。`myfirst_ioctl.h`是规范示例。它声明命令号和有效载荷结构；它们被`myfirst_ioctl.c`和用户空间调用者需要，但没有其他内核内文件需要它们。将它们放在`myfirst.h`中会将线格式泄漏到每个翻译单元。

区别很重要，因为每个公共声明是驱动程序必须遵守的契约。`myfirst.h`中改变大小的类型会破坏包含`myfirst.h`的每个文件。`myfirst_ioctl.h`中改变大小的类型仅破坏`myfirst_ioctl.c`和针对它编译的用户空间工具。

对于第25章末尾的`myfirst`，公共头文件`myfirst.h`看起来像这样（修剪到与本节相关的声明）：

```c
/*
 * myfirst.h - myfirst驱动程序的公共类型和常量。
 *
 * 这里声明的类型和原型对驱动程序中的每个.c文件
 * 可见。保持此头文件小。线格式声明在
 * myfirst_ioctl.h中。调试宏在myfirst_debug.h中。限速
 * 状态在myfirst_log.h中。
 */

#ifndef _MYFIRST_H_
#define _MYFIRST_H_

#include <sys/types.h>
#include <sys/lock.h>
#include <sys/mutex.h>
#include <sys/conf.h>

#include "myfirst_log.h"

struct myfirst_softc {
	device_t       sc_dev;
	struct mtx     sc_mtx;
	struct cdev   *sc_cdev;

	char           sc_msg[MYFIRST_MSG_MAX];
	size_t         sc_msglen;

	u_int          sc_open_count;
	u_int          sc_total_reads;
	u_int          sc_total_writes;

	u_int          sc_debug;
	u_int          sc_timeout_sec;
	u_int          sc_max_retries;
	u_int          sc_log_pps;

	struct myfirst_ratelimit sc_rl_generic;
	struct myfirst_ratelimit sc_rl_io;
	struct myfirst_ratelimit sc_rl_intr;
};

#define MYFIRST_MSG_MAX  256

/* Sysctl树。 */
void myfirst_sysctl_attach(struct myfirst_softc *);

/* 限速状态。 */
int  myfirst_log_attach(struct myfirst_softc *);
void myfirst_log_detach(struct myfirst_softc *);

/* Ioctl分发。 */
struct thread;
int  myfirst_ioctl(struct cdev *, u_long, caddr_t, int, struct thread *);

#endif /* _MYFIRST_H_ */
```

`myfirst.h`中没有任何东西引用线格式常量、调试类别位或限速结构内部。softc按值包含三个限速字段，因此`myfirst.h`必须包含`myfirst_log.h`，但`struct myfirst_ratelimit`的内部在`myfirst_log.h`中，不在这里暴露。

### 拆分后`myfirst.c`的解剖

拆分后的`myfirst.c`是驱动程序中最短的`.c`文件。它包含cdevsw表、模块事件处理程序、设备类声明和attach/detach例程。每个其他职责已移到别处：

```c
/*
 * myfirst.c - myfirst驱动程序的模块粘合和cdev接线。
 *
 * 此文件拥有cdevsw表、devclass、attach和
 * detach例程以及MODULE_VERSION声明。cdev
 * 回调本身在myfirst_cdev.c中。ioctl分发
 * 在myfirst_ioctl.c中。sysctl树在
 * myfirst_sysctl.c中。限速基础设施在
 * myfirst_log.c中。
 */

#include <sys/param.h>
#include <sys/systm.h>
#include <sys/conf.h>
#include <sys/kernel.h>
#include <sys/lock.h>
#include <sys/module.h>
#include <sys/mutex.h>

#include "myfirst.h"
#include "myfirst_debug.h"
#include "myfirst_ioctl.h"

MODULE_VERSION(myfirst, 18);

extern d_open_t    myfirst_open;
extern d_close_t   myfirst_close;
extern d_read_t    myfirst_read;
extern d_write_t   myfirst_write;

struct cdevsw myfirst_cdevsw = {
	.d_version = D_VERSION,
	.d_name    = "myfirst",
	.d_open    = myfirst_open,
	.d_close   = myfirst_close,
	.d_read    = myfirst_read,
	.d_write   = myfirst_write,
	.d_ioctl   = myfirst_ioctl,
};

static int
myfirst_attach(device_t dev)
{
	/* 第5节的标签清理attach在这里。 */
	...
}

static int
myfirst_detach(device_t dev)
{
	/* 第5节的attach镜像detach在这里。 */
	...
}

static device_method_t myfirst_methods[] = {
	DEVMETHOD(device_probe,   myfirst_probe),
	DEVMETHOD(device_attach,  myfirst_attach),
	DEVMETHOD(device_detach,  myfirst_detach),
	DEVMETHOD_END
};

static driver_t myfirst_driver = {
	"myfirst",
	myfirst_methods,
	sizeof(struct myfirst_softc),
};

DRIVER_MODULE(myfirst, nexus, myfirst_driver, 0, 0);
```

文件有一个工作：在内核级别连接驱动程序的各个部分。它只有几百行；驱动程序中的每个其他文件都更小。

### `myfirst_cdev.c`：字符设备回调

打开、关闭、读取和写入回调是我们在第18章中编写的第一个代码。从那时起它们已经增长。将它们提取到`myfirst_cdev.c`使它们在一起并远离`myfirst.c`：

```c
/*
 * myfirst_cdev.c - myfirst驱动程序的字符设备回调。
 *
 * open/close/read/write回调都操作于make_dev_s
 * 作为si_drv1安装的softc。ioctl分发在
 * myfirst_ioctl.c中；此文件故意不处理ioctl。
 */

#include <sys/param.h>
#include <sys/systm.h>
#include <sys/conf.h>
#include <sys/uio.h>
#include <sys/lock.h>
#include <sys/mutex.h>

#include "myfirst.h"
#include "myfirst_debug.h"

int
myfirst_open(struct cdev *dev, int oflags, int devtype, struct thread *td)
{
	struct myfirst_softc *sc = dev->si_drv1;

	mtx_lock(&sc->sc_mtx);
	sc->sc_open_count++;
	mtx_unlock(&sc->sc_mtx);

	DPRINTF(sc, MYF_DBG_OPEN, "open: 计数%u\n", sc->sc_open_count);
	return (0);
}

/* close、read、write遵循相同的模式。 */
```

每个回调以`sc = dev->si_drv1`开始（`make_dev_args`设置的每cdev指针）并操作于softc。没有跨文件耦合超出公共头文件。

### `myfirst_ioctl.c`：命令Switch

`myfirst_ioctl.c`自第22章以来一直在自己的文件中。第25章添加的是`MYFIRSTIOC_GETCAPS`处理程序：

```c
int
myfirst_ioctl(struct cdev *dev, u_long cmd, caddr_t data, int flag,
    struct thread *td)
{
	struct myfirst_softc *sc = dev->si_drv1;
	int error = 0;

	switch (cmd) {
	case MYFIRSTIOC_GETVER:
		*(int *)data = MYFIRST_IOCTL_VERSION;
		break;
	case MYFIRSTIOC_RESET:
		mtx_lock(&sc->sc_mtx);
		sc->sc_total_reads  = 0;
		sc->sc_total_writes = 0;
		mtx_unlock(&sc->sc_mtx);
		break;
	case MYFIRSTIOC_GETMSG:
		mtx_lock(&sc->sc_mtx);
		strlcpy((char *)data, sc->sc_msg, MYFIRST_MSG_MAX);
		mtx_unlock(&sc->sc_mtx);
		break;
	case MYFIRSTIOC_SETMSG:
		mtx_lock(&sc->sc_mtx);
		strlcpy(sc->sc_msg, (const char *)data, MYFIRST_MSG_MAX);
		sc->sc_msglen = strlen(sc->sc_msg);
		mtx_unlock(&sc->sc_mtx);
		break;
	case MYFIRSTIOC_GETCAPS:
		*(uint32_t *)data = MYF_CAP_RESET | MYF_CAP_GETMSG |
		                    MYF_CAP_SETMSG;
		break;
	default:
		error = ENOIOCTL;
		break;
	}
	return (error);
}
```

switch是驱动程序的整个公共ioctl表面。添加命令意味着添加case；退役一个意味着删除case并在`myfirst_ioctl.h`中弃用常量。

### `myfirst_log.h`和`myfirst_log.c`：限速日志记录

第1节介绍了限速日志宏`DLOG_RL`和它跟踪的`struct myfirst_ratelimit`状态。限速状态在第1节中嵌入在softc中，因为抽象尚未被提取出来。第6节是提取它的正确时机：限速代码足够小，值得收集在一个地方，足够通用，其他驱动程序可能需要它。

`myfirst_log.h`包含状态定义：

```c
#ifndef _MYFIRST_LOG_H_
#define _MYFIRST_LOG_H_

#include <sys/time.h>

struct myfirst_ratelimit {
	struct timeval rl_lasttime;
	int            rl_curpps;
};

#define MYF_RL_DEFAULT_PPS  10

#endif /* _MYFIRST_LOG_H_ */
```

`myfirst_log.c`包含attach和detach助手：

```c
#include <sys/param.h>
#include <sys/systm.h>
#include <sys/lock.h>
#include <sys/mutex.h>

#include "myfirst.h"
#include "myfirst_debug.h"

int
myfirst_log_attach(struct myfirst_softc *sc)
{
	/*
	 * 限速状态按值嵌入在softc中，所以
	 * 没有分配要做。此函数存在是为了
	 * attach链有一个命名标签，以便将来
	 * 版本需要每类别分配时有日志记录。
	 */
	bzero(&sc->sc_rl_generic, sizeof(sc->sc_rl_generic));
	bzero(&sc->sc_rl_io,      sizeof(sc->sc_rl_io));
	bzero(&sc->sc_rl_intr,    sizeof(sc->sc_rl_intr));

	return (0);
}

void
myfirst_log_detach(struct myfirst_softc *sc)
{
	/* 没有什么可释放的；状态嵌入在softc中。 */
	(void)sc;
}
```

今天`myfirst_log_attach`不做分配；它将限速字段置零并返回。明天，如果驱动程序需要每类别计数器的动态数组，分配适合这里，attach链不必改变。这是在严格必要之前提取助手的价值：形状准备好增长。

这里的头文件大小很重要。`myfirst_log.h`不到20行。20行的头文件在任何地方包含都很便宜，易于阅读，易于保持同步。如果`myfirst_log.h`增长到200行，从每个`.c`文件包含它的成本将开始显示在编译时间和审阅摩擦中；那时下一步是再次拆分它。

### 更新的Makefile

拆分驱动程序的Makefile列出每个`.c`文件：

```makefile
# myfirst驱动程序的Makefile - 第25章（1.8-maintenance）。
#
# 第25章将驱动程序拆分为主题文件。每个文件
# 回答一个单一问题；Makefile在myfirst.c之后
# 按字母顺序列出它们（它承载模块粘合），
# 以便读者首先看到主文件。

KMOD=	myfirst
SRCS=	myfirst.c myfirst_cdev.c myfirst_debug.c myfirst_ioctl.c \
	myfirst_log.c myfirst_sysctl.c

CFLAGS+=	-I${.CURDIR}

SYSDIR?=	/usr/src/sys

.include <bsd.kmod.mk>
```

`SRCS`列出六个`.c`文件，每个主题一个。添加第七个是一行更改。内核构建系统自动获取`SRCS`中的每个文件；没有手动链接步骤，没有makefile依赖树要维护。

### 在哪里画每个文件边界

拆分驱动程序最难的部分不是决定拆分；是决定边界在哪里。大多数拆分经历三个阶段，这些阶段适用于任何驱动程序，不仅仅是`myfirst`。

**第一阶段**是平坦文件。所有东西都在`driver.c`中。这对于驱动程序的前300行是正确的形状。更早拆分产生的摩擦多于节省。

**第二阶段**是主题拆分。ioctl分发进入`driver_ioctl.c`，sysctl树进入`driver_sysctl.c`，调试基础设施进入`driver_debug.c`。每个文件以其处理的主题命名。这是`myfirst`自第24章以来的位置。

**第三阶段**是子系统拆分。随着驱动程序的增长，一个主题变得大于单个文件。ioctl文件拆分为`driver_ioctl.c`（分发）和`driver_ioctl_rw.c`（读/写有效载荷助手）。sysctl文件类似拆分。这是全功能驱动程序最终的位置，通常在第三或第四个主要版本中。

第25章末尾的`myfirst`牢牢处于第二阶段。第三阶段尚不需要，第26章将在伪驱动程序拆分为USB附加变体并将`myfirst_core.c`保留为主题无关核心时重置时钟。今天预先拆分到第三阶段没有价值。

从第二阶段移动到第三阶段的经验法则是：当单个主题文件超过1000行，或当同一主题文件的两个不相关更改导致合并冲突时，该主题准备好拆分。

### Include图和构建顺序

一旦驱动程序被拆分为几个文件，include图就很重要。循环include在C中不是硬错误，但它是依赖结构混乱的迹象，会困惑读者。正确的形状是一个有向无环的头文件图，以`myfirst.h`为根，叶子头文件如`myfirst_ioctl.h`和`myfirst_log.h`。

`myfirst.h`是最宽的头文件。它声明softc和每个其他文件使用的原型。它包含`myfirst_log.h`，因为softc按值有limit字段。

`myfirst_debug.h`是叶子。它声明`DPRINTF`宏系列和类别位。它被每个`.c`文件直接或间接包含。它不被`myfirst.h`包含，因为`myfirst.h`不应强制调试宏进入任何不想使用它们的调用者。

`myfirst_ioctl.h`是叶子。它声明命令号、有效载荷结构和线格式版本整数。它被`myfirst_ioctl.c`（及其用户空间对应物`myfirstctl.c`）包含。

没有头文件包含公共内核头文件和驱动程序自己的头文件以外的文件。没有`.c`文件包含另一个`.c`文件。Include图很浅，易于绘制。

### 拆分的成本

拆分文件有真实成本。每次拆分添加一个头文件，每个头文件必须维护。签名更改的函数必须在`.c`和`.h`中更新，更改必须传播到包含头文件的每个其他`.c`文件。有十二个文件的驱动程序编译比只有一个的驱动程序稍慢，因为每个`.c`必须包含几个头文件，预处理器必须解析它们。

这些成本是真实的但很小。它们比没人想触及的单体文件的成本小得多。规则是当不拆分的成本超过维护边界的成本时拆分。对于第25章末尾的`myfirst`，阈值已跨越。

### 拆分真实驱动程序的实用过程

拆分文件是一个常规重构，但如果不小心进行，常规重构仍然可能引入错误。拆分树内驱动程序的实用过程是：

1. **识别主题。** 从头到尾阅读单体文件，按主题（cdev、ioctl、sysctl、调试、生命周期）分组其函数。在纸上或注释块中写下分组。

2. **创建空文件。** 将新`.c`文件及其头文件添加到源树。编译一次以确保构建系统看到它们。

3. **一次移动一个主题。** 将ioctl函数移动到`driver_ioctl.c`。将其声明移动到`driver_ioctl.h`。更新`driver.c`以`#include "driver_ioctl.h"`。编译。在测试矩阵中运行驱动程序。

4. **提交每个主题拆分。** 每个主题移动是单个提交。提交日志读："myfirst：将ioctl分发拆分为driver_ioctl.c"。审阅者可以清楚地看到移动；`git blame`在新文件中显示同一提交的同一行。

5. **验证Include图。** 在所有主题移动后，用`-Wunused-variable`和`-Wmissing-prototypes`编译，以捕获应该有原型但没有的函数。使用`nm`在构建的模块上确认没有应该是`static`的符号被导出。

6. **重新测试。** 运行完整的驱动程序测试矩阵。拆分文件不应改变行为；如果测试开始失败，拆分引入了错误。

第25章末尾的`myfirst`的过程完全遵循这些步骤。`examples/part-05/ch25-advanced/`下的最终目录是结果。

### 拆分时的常见错误

第一次拆分驱动程序时会出现一些错误。注意它们可以缩短学习曲线。

第一个错误是**将声明放在错误的头文件中**。如果`myfirst.h`声明一个仅从`myfirst_ioctl.c`调用的函数，每个其他翻译单元都在付费解析它不需要的声明。如果`myfirst_ioctl.h`声明一个从`myfirst_ioctl.c`和`myfirst_cdev.c`都调用的函数，两个消费者通过ioctl头文件耦合，ioctl头文件的任何更改都会重建两个文件。修复是将跨领域声明放在`myfirst.h`中，主题特定声明放在主题特定头文件中。

第二个错误是**忘记应该是文件范围的函数上的`static`**。仅在`myfirst_sysctl.c`内部使用的函数应该声明为`static`。没有`static`，函数从对象文件导出，这意味着另一个文件可能意外调用它，原始文件中的任何后续重命名都成为ABI更改。`static`规范防止这整个类问题。

第三个错误是**循环include**。如果`myfirst_ioctl.h`包含`myfirst.h`，而`myfirst.h`包含`myfirst_ioctl.h`，驱动程序可以编译（多亏include守卫）但依赖图是错误的。对任一文件的编辑现在触发包含任一文件的每个东西的重建。修复是决定哪个头文件位于图中更高并移除反向引用。

第四个错误是**将主题重新引入错误的文件**。拆分六个月后，有人通过编辑`myfirst.c`添加新ioctl，因为那是ioctts曾经所在的地方。单一职责规则必须由审阅者强制执行。将新ioctl放在`myfirst.c`中的补丁被拒绝，并带有指向`myfirst_ioctl.c`的注释。

### 第6节总结

每个文件回答一个问题的驱动程序是一个你可以在第一天就交给新贡献者的驱动程序。他们阅读文件名，选择一个主题，然后开始编辑恰好一个文件。`myfirst`驱动程序已经跨越该阈值。六个`.c`文件及其头文件承载驱动程序自第17章以来增长的每个函数，每个文件以其所做的事情命名。

在下一节中，我们从内部组织转向外部准备。准备好生产的驱动程序在可以安装在开发者不拥有的机器上之前，必须满足一小组属性。第25章的生产准备就绪检查表命名这些属性，并让`myfirst`通过每一个。


## 第8节：SYSINIT、SYSUNINIT和EVENTHANDLER

驱动程序的附加和分离例程处理实例化和拆卸之间发生的一切，但有一些事情驱动程序可能需要做，它们落在这个窗口之外。有些代码必须在任何设备实例化之前运行：加载启动时可调参数、初始化子系统范围的锁、设置第一个`attach`将消耗的池。其他代码必须响应非设备特定的系统范围事件：系统范围的挂起、低内存条件、即将同步文件系统并关闭的关闭。

FreeBSD内核为这些情况提供两种机制。`SYSINIT(9)`注册一个函数在特定启动阶段运行，其配套`SYSUNINIT(9)`注册一个清理函数在模块卸载时运行。`EVENTHANDLER(9)`注册一个回调在内核触发命名事件时运行。

两种机制自FreeBSD最早发布以来就可用。它们是无聊的基础设施；这就是它们的价值。正确使用它们的驱动程序可以对完整的内核生命周期做出反应，而无需编写任何手动注册代码。忽略它们的驱动程序要么错过了时机，要么重新发明了同一设施的更差版本。

### 为什么内核需要启动时排序

FreeBSD内核以精确的顺序启动。内存管理在任何分配器可用之前启动。可调参数在驱动程序可以读取它们之前解析。锁在任何东西被允许获取它们之前初始化。文件系统仅在它们所在的设备被探测之后挂载。这些依赖中的每一个都必须遵守，否则内核在`init(8)`启动之前恐慌。

强制排序的机制是`SYSINIT(9)`。`SYSINIT`宏声明给定函数应该在给定子系统ID以给定顺序常量运行。内核的启动序列收集运行配置中的每个`SYSINIT`，按（子系统、顺序）排序，并按该顺序调用它们。内核启动后加载的模块仍然遵守其`SYSINIT`声明：加载器在模块附加时按相同的排序顺序调用它们。

从驱动程序的角度来看，`SYSINIT`是说"在启动序列的那个点做这件事，我不在乎还有哪些其他代码也在那里注册"的方式。内核处理排序；驱动程序编写回调。

### 子系统ID空间

子系统ID在`/usr/src/sys/sys/kernel.h`中定义。常量具有描述性名称和反映其排序的数值。驱动程序选择与其回调目的对应的子系统：

- `SI_SUB_TUNABLES` (0x0700000)：评估启动时可调参数。这是`TUNABLE_INT_FETCH`及其兄弟运行的地方。使用可调参数的代码必须在此点之后运行。
- `SI_SUB_KLD` (0x2000000)：可加载内核模块设置。早期模块基础设施在这里运行。
- `SI_SUB_SMP` (0x2900000)：启动应用处理器。
- `SI_SUB_DRIVERS` (0x3100000)：让驱动程序初始化。这是大多数用户空间驱动程序如果需要任何设备附加之前运行的早期代码时注册的子系统。
- `SI_SUB_CONFIGURE` (0x3800000)：配置设备。在这个子系统结束时，每个编译入的驱动程序都有机会附加。

`kernel.h`中有超过一百个子系统ID。上面这些是字符设备驱动程序最常交互的。数值排序以便"较小的数字"意味着"启动较早"。

在子系统内，顺序常量给出细粒度排序：

- `SI_ORDER_FIRST` (0x0)：在同一子系统中大多数其他代码之前运行。
- `SI_ORDER_SECOND`、`SI_ORDER_THIRD`：显式逐步排序。
- `SI_ORDER_MIDDLE` (0x1000000)：在中间运行。大多数驱动程序级别的`SYSINIT`使用这个或下面的一个。
- `SI_ORDER_ANY` (0xfffffff)：最后运行。内核不保证`SI_ORDER_ANY`条目之间的任何特定顺序。

驱动程序作者选择使其回调在其先决条件之后、其依赖者之前运行的最低顺序。对于大多数目的，`SI_ORDER_MIDDLE`是正确的。

### 驱动程序何时需要`SYSINIT`

大多数字符设备驱动程序根本不需要`SYSINIT`。`DRIVER_MODULE`已经向Newbus注册驱动程序；驱动程序的`device_attach`方法在匹配的设备出现时运行。这对于每个实例的工作来说已经足够。

`SYSINIT`用于非每个实例的工作。驱动程序可能注册`SYSINIT`的原因列表：

- **初始化全局池**，驱动程序的每个实例将从中提取。池存在一次；它不属于任何单个softc。
- **向期望调用者在使用之前注册的内核子系统注册**。例如，想要接收`vm_lowmem`事件的驱动程序应该提前注册，以便第一次低内存事件不会错过它。
- **解析比单个`TUNABLE_INT_FETCH`需要更多工作的复杂可调参数**。可调参数解析代码在`SI_SUB_TUNABLES`期间运行并填充每个实例代码稍后咨询的全局结构。
- **在第一个调用者可以使用之前自测**加密原语或子系统初始化器。

对于`myfirst`，这些目前都不适用。驱动程序是每个实例的，其可调参数很简单，它不使用需要预注册的子系统。第25章介绍`SYSINIT`不是`myfirst`需要，而是读者应该熟悉该宏并理解未来的更改何时会需要它。

### `SYSINIT`声明的形状

宏签名为：

```c
SYSINIT(uniquifier, subsystem, order, func, ident);
```

- `uniquifier`是将`SYSINIT`符号绑定到此声明的C标识符。它在其他地方不出现。约定是使用匹配子系统或函数的短名称。
- `subsystem`是`SI_SUB_*`常量。
- `order`是`SI_ORDER_*`常量。
- `func`是签名为`void (*)(void *)`的函数指针。
- `ident`是传递给`func`的单个参数。对于大多数用途，它是`NULL`。

匹配的清理宏是：

```c
SYSUNINIT(uniquifier, subsystem, order, func, ident);
```

`SYSUNINIT`注册清理函数。它在模块卸载时以`SYSINIT`声明的相反顺序运行。对于编译入内核（不是模块）的代码，`SYSUNINIT`永远不会触发，因为内核永远不会卸载；但声明仍然有用，因为将驱动程序编译为模块会锻炼清理路径。

### `myfirst`的`SYSINIT`工作示例

考虑对`myfirst`的假设增强：每个实例可以从中提取的全局、驱动程序范围的预分配日志缓冲区池。池每个模块加载初始化一次，每个模块卸载销毁一次。每个实例附加和分离不直接触碰池；它们只从中取用和归还缓冲区。

`SYSINIT`声明如下所示：

```c
#include <sys/kernel.h>

static struct myfirst_log_pool {
	struct mtx       lp_mtx;
	/* ... per-pool state ... */
} myfirst_log_pool;

static void
myfirst_log_pool_init(void *unused __unused)
{
	mtx_init(&myfirst_log_pool.lp_mtx, "myfirst log pool",
	    NULL, MTX_DEF);
	/* Allocate pool entries. */
}

static void
myfirst_log_pool_fini(void *unused __unused)
{
	/* Release pool entries. */
	mtx_destroy(&myfirst_log_pool.lp_mtx);
}

SYSINIT(myfirst_log_pool,  SI_SUB_DRIVERS, SI_ORDER_MIDDLE,
    myfirst_log_pool_init, NULL);
SYSUNINIT(myfirst_log_pool, SI_SUB_DRIVERS, SI_ORDER_MIDDLE,
    myfirst_log_pool_fini, NULL);
```

当加载`myfirst`时，内核对`SYSINIT`条目排序并在`SI_SUB_DRIVERS`阶段调用`myfirst_log_pool_init`。之后运行的第一个`myfirst_attach`发现池已准备好。当卸载模块时，`myfirst_log_pool_fini`在每个实例分离后运行，给池一个释放其资源的机会。

这是一个教学用的草图；`myfirst`在实际发布的第25章代码中并没有真正使用全局池。最终编写需要它的驱动程序的读者会在这里找到该模式。

### `SYSINIT`和`DRIVER_MODULE`之间的排序

`DRIVER_MODULE`本身在底层实现为`SYSINIT`。它在特定的子系统阶段向Newbus注册驱动程序，Newbus自己的`SYSINIT`之后探测和附加设备。因此，驱动程序的自定义`SYSINIT`可以通过选择正确的子系统和顺序相对于`DRIVER_MODULE`排序。

经验法则：

- `SI_SUB_DRIVERS`的`SYSINIT`带`SI_ORDER_FIRST`在`DRIVER_MODULE`注册之前运行。
- `SI_SUB_CONFIGURE`的`SYSINIT`带`SI_ORDER_MIDDLE`在大多数设备附加之后但最终配置步骤之前运行。

对于附加依赖的全局池，`SI_SUB_DRIVERS`带`SI_ORDER_MIDDLE`通常正确：池在`DRIVER_MODULE`的设备开始附加之前初始化（因为`SI_SUB_DRIVERS`早于`SI_SUB_CONFIGURE`），顺序常量使其远离最早的钩子。

### `EVENTHANDLER`：响应运行时事件

`SYSINIT`在已知的启动阶段触发一次。`EVENTHANDLER`在特定系统事件发生时触发零次或多次。这两种机制是表亲；它们互补。

内核定义了许多命名事件。每个事件都有固定的回调签名和触发它的固定情况。关心事件的驱动程序注册回调；内核每次事件触发时调用回调；驱动程序在分离时注销回调。

一些常用事件：

- `shutdown_pre_sync`：系统即将同步文件系统。有内存缓存的驱动程序在这里刷新它们。
- `shutdown_post_sync`：系统已完成同步文件系统。需要知道"文件系统安静"的驱动程序在这里挂钩。
- `shutdown_final`：系统即将停止或重启。必须保存硬件状态的驱动程序在这里做。
- `vm_lowmem`：虚拟内存子系统压力。有自己的缓存的驱动程序应该释放一些内存。
- `power_suspend_early`、`power_suspend`、`power_resume`：挂起/恢复生命周期。
- `dev_clone`：设备克隆事件，由按需出现的伪设备使用。

列表不是固定的；新事件随着内核增长而添加。上面这些是一般驱动程序最常考虑的。

### `EVENTHANDLER`注册的形状

模式有三部分：声明具有正确签名的处理函数，在附加时注册，在分离时注销。注册返回一个不透明标签；注销需要该标签。

对于`shutdown_pre_sync`，处理程序签名为：

```c
void (*handler)(void *arg, int howto);
```

`arg`是驱动程序传递给注册的任何指针；通常是softc。`howto`是关闭标志（`RB_HALT`、`RB_REBOOT`等）。

`myfirst`的最小关闭处理程序：

```c
#include <sys/eventhandler.h>

static eventhandler_tag myfirst_shutdown_tag;

static void
myfirst_shutdown(void *arg, int howto)
{
	struct myfirst_softc *sc = arg;

	mtx_lock(&sc->sc_mtx);
	DPRINTF(sc, MYF_DBG_INIT, "shutdown: howto=0x%x\n", howto);
	/* Flush any pending state here. */
	mtx_unlock(&sc->sc_mtx);
}
```

注册发生在`myfirst_attach`内（或从中调用的辅助函数中）：

```c
myfirst_shutdown_tag = EVENTHANDLER_REGISTER(shutdown_pre_sync,
    myfirst_shutdown, sc, SHUTDOWN_PRI_DEFAULT);
```

注销发生在`myfirst_detach`内：

```c
EVENTHANDLER_DEREGISTER(shutdown_pre_sync, myfirst_shutdown_tag);
```

注销是强制性的。分离但不注销的驱动程序在内核的事件列表中留下悬空的回调指针。当内核下次触发事件时，它调用不再映射的内存区域，系统恐慌。

存储在`myfirst_shutdown_tag`中的标签是将注册绑定到注销的原因。对于具有单个实例的驱动程序，像上面那样的静态变量可以工作。对于具有多个实例的驱动程序，标签应该位于softc中，以便每个实例的注销引用自己的标签。

### 附加链中的`EVENTHANDLER`

因为注册和注销是对称的，它们干净地嵌入第5节的标签化清理模式。注册变成获取；其失败模式是"注册是否返回错误？"（可能在低内存条件下失败）；其清理是`EVENTHANDLER_DEREGISTER`。

感知`EVENTHANDLER`的`myfirst`的更新附加片段：

```c
	/* Stage 7: shutdown handler. */
	sc->sc_shutdown_tag = EVENTHANDLER_REGISTER(shutdown_pre_sync,
	    myfirst_shutdown, sc, SHUTDOWN_PRI_DEFAULT);
	if (sc->sc_shutdown_tag == NULL) {
		error = ENOMEM;
		goto fail_log;
	}

	return (0);

fail_log:
	myfirst_log_detach(sc);
fail_cdev:
	destroy_dev(sc->sc_cdev);
fail_mtx:
	mtx_destroy(&sc->sc_mtx);
	return (error);
```

以及匹配的分离，注销在最前（获取的反序）：

```c
	EVENTHANDLER_DEREGISTER(shutdown_pre_sync, sc->sc_shutdown_tag);
	myfirst_log_detach(sc);
	destroy_dev(sc->sc_cdev);
	mtx_destroy(&sc->sc_mtx);
```

`sc->sc_shutdown_tag`位于softc中。将其存储在那里很重要：注销需要知道要移除哪个具体注册，每个softc存储保持驱动程序的两个实例独立。

### 优先级：`SHUTDOWN_PRI_*`

在单个事件内，回调按优先级顺序调用。优先级是`EVENTHANDLER_REGISTER`的第四个参数。对于关闭事件，常见常量是：

- `SHUTDOWN_PRI_FIRST`：在大多数其他处理程序之前运行。
- `SHUTDOWN_PRI_DEFAULT`：以默认顺序运行。
- `SHUTDOWN_PRI_LAST`：在其他处理程序之后运行。

硬件需要在文件系统刷新之前安静的驱动程序可能用`SHUTDOWN_PRI_FIRST`注册。状态依赖于文件系统已经刷新的驱动程序（实践中不太可能）可能用`SHUTDOWN_PRI_LAST`注册。大多数驱动程序使用`SHUTDOWN_PRI_DEFAULT`，不考虑优先级。

其他事件存在类似的优先级常量（`EVENTHANDLER_PRI_FIRST`、`EVENTHANDLER_PRI_ANY`、`EVENTHANDLER_PRI_LAST`）。

### 何时使用`vm_lowmem`

`vm_lowmem`是VM子系统在空闲内存降至阈值以下时触发的事件。维护自己缓存的驱动程序（例如预分配块池）可以响应释放一些回内核。

处理程序以单个参数（触发事件的子系统ID）调用。有缓存的驱动程序的最小处理程序：

```c
static void
myfirst_lowmem(void *arg, int unused __unused)
{
	struct myfirst_softc *sc = arg;

	mtx_lock(&sc->sc_mtx);
	/* Release some entries from the cache. */
	mtx_unlock(&sc->sc_mtx);
}
```

注册看起来与关闭的相同，只是事件名称不同：

```c
sc->sc_lowmem_tag = EVENTHANDLER_REGISTER(vm_lowmem,
    myfirst_lowmem, sc, EVENTHANDLER_PRI_ANY);
```

不维护缓存的驱动程序不应该为`vm_lowmem`注册。这样做的成本不是零：内核在每个低内存事件上调用每个注册的处理程序，无操作的处理程序给该调用链增加延迟。

对于`myfirst`，没有缓存，所以不使用`vm_lowmem`。为即将编写需要它的驱动程序的读者引入该模式。

### `power_suspend_early`和`power_resume`

挂起/恢复是敏感的生命周期。在`power_suspend_early`和`power_resume`之间，驱动程序的设备预期是静止的：无I/O、无中断、无状态转换。有必须在挂起前保存和恢复后恢复的硬件状态的驱动程序为两个事件注册处理程序。

对于不管理硬件的字符设备驱动程序，这些事件通常不适用。对于总线附加的驱动程序（PCI、USB、SPI），总线层处理大部分挂起/恢复簿记，驱动程序只需在其`device_method_t`表中提供`device_suspend`和`device_resume`方法。`EVENTHANDLER`方法用于想对系统范围挂起做出反应而无需总线附加的驱动程序。

第26章将在`myfirst`变成USB驱动程序时重新审视挂起/恢复；那时总线层的机制是首选。

### 模块事件处理器

与`SYSINIT`和`EVENTHANDLER`相关的是模块事件处理器：内核为`MOD_LOAD`、`MOD_UNLOAD`、`MOD_QUIESCE`和`MOD_SHUTDOWN`调用的回调。大多数驱动程序不覆盖它；`DRIVER_MODULE`提供调用`device_probe`和`device_attach`的默认实现。

需要在模块加载时自定义行为（超出`SYSINIT`能做的）的驱动程序可以提供自己的处理器：

```c
static int
myfirst_modevent(module_t mod, int what, void *arg)
{
	switch (what) {
	case MOD_LOAD:
		/* Custom load behaviour. */
		return (0);
	case MOD_UNLOAD:
		/* Custom unload behaviour. */
		return (0);
	case MOD_QUIESCE:
		/* Can we be unloaded?  Return errno if not. */
		return (0);
	case MOD_SHUTDOWN:
		/* Shutdown notification; usually no-op. */
		return (0);
	default:
		return (EOPNOTSUPP);
	}
}
```

处理器通过`moduledata_t`结构而不是`DRIVER_MODULE`连接。两种方法对于给定模块名是互斥的；驱动程序选择其中之一。

对于大多数驱动程序，`DRIVER_MODULE`的默认是正确的，模块事件处理器不自定义。`myfirst`全程使用`DRIVER_MODULE`。

### 注销规范

使用`EVENTHANDLER`时最重要的单一规则是：注册一次，注销一次，分别在附加和分离中。违反规则时会出现两种失败模式。

第一种失败模式是**遗漏注销**。分离运行，标签未注销，内核的事件列表仍然指向softc的处理程序，下一个事件触发到已释放的内存。恐慌发生在原因很远的地方，因为下一个事件可能在分离后几分钟或几小时才触发。

修复是机械的：附加中的每个`EVENTHANDLER_REGISTER`在分离中获得匹配的`EVENTHANDLER_DEREGISTER`。第5节的标签化清理模式使这变得容易：注册是带标签的获取，清理链按相反顺序注销。

第二种失败模式是**双重注册**。两次注册同一处理程序的驱动程序在内核事件列表中有两个条目；分离一次只移除其中一个。内核然后有一个指向刚离开的softc的陈旧条目。

修复也是机械的：每个附加恰好注册一次。不要在从多个地方调用的辅助函数中注册；不要响应第一个事件懒惰注册。

### 第8节总结

`SYSINIT`和`EVENTHANDLER`是内核让驱动程序参与自身附加/分离窗口之外的生命周期的方式。`SYSINIT`在特定启动阶段运行代码；`EVENTHANDLER`响应命名内核事件运行代码。它们一起覆盖每个设备代码不够的情况，驱动程序必须作为整体与系统交互。

第25章结束时的`myfirst`使用`EVENTHANDLER_REGISTER`作为演示`shutdown_pre_sync`处理程序；注册、注销和标签化清理形态都已到位。`SYSINIT`被介绍但未使用，因为`myfirst`今天没有全局池。模式已种下；当未来章节的驱动程序确实需要它们时，读者会立即认出它们。

第8节完成后，本章设定教授的每种机制都已在驱动程序中。章节剩余材料通过动手实验室、挑战练习和出错时的故障排除参考应用这些机制。

---

## 动手实验室（概要）

本节的实验室在真实的FreeBSD 14.3系统上锻炼本章的机制。每个实验室都有特定的可衡量结果；运行实验室后，您应该能够说明您看到了什么以及它意味着什么。实验室假设您有`examples/part-05/ch25-advanced/`配套目录。

**实验室1：重现日志泛滥** - 展示不限速的`device_printf`与限速的`DLOG_RL`在热循环中触发时的区别。

**实验室2：使用`truss`进行errno审计** - 使用`truss(1)`查看驱动程序返回不同errno值时报告的内容，校准关于哪个代码路径返回哪个errno的直觉。

**实验室3：可调参数重启行为** - 验证加载器可调参数在模块首次加载时实际更改驱动程序的初始状态。

**实验室4：故意附加失败注入** - 验证清理链中的每个标签都可达，失败时不泄漏资源。

**实验室5：`shutdown_pre_sync`处理程序** - 确认注册的关闭处理程序在实际关闭期间触发，观察其相对于文件系统同步的排序。

**实验室6：100次循环回归脚本** - 运行持续的加载/卸载循环以捕捉仅在重复附加/分离下出现的泄漏。

**实验室7：用户空间能力发现** - 确认用户空间程序可以在运行时发现驱动程序的能力并相应行为。

**实验室8：sysctl范围验证** - 确认驱动程序公开的每个可写sysctl拒绝超出范围的值并在拒绝时保持内部状态不变。

**实验室9：驱动程序日志消息审计** - 清点驱动程序中的每条日志消息并确认每条都遵循第1节和第2节的规范。

**实验室10：多版本兼容性矩阵** - 确认能力发现模式允许单个用户空间程序与三个不同版本的驱动程序一起工作。

---

## 故障排除指南（概要）

当本章的机制表现异常时，症状通常是间接的：一条静默丢失的日志行、一个因错误原因拒绝加载的驱动程序、一个未出现的sysctl、一个未调用处理程序的重启。此参考将常见症状映射到最可能负责的机制，以及驱动程序源代码中首先查看的地方。

常见症状包括：
- `kldload`返回"Exec format error"（模块ABI不匹配）
- `kldload`报告"module busy"（前一个实例仍有打开的文件描述符）
- 新sysctl未出现在树中（上下文/树指针错误）
- 可调参数似乎被忽略（名称拼写错误或在模块加载后设置）
- 日志消息应该触发但未出现在`dmesg`中（调试类位未设置或限速存储问题）
- 分离时带有"witness"警告恐慌（锁被销毁时仍被持有）
- 模块卸载时带有悬空指针恐慌（未调用`EVENTHANDLER_DEREGISTER`）

---

## 快速参考（概要）

快速参考是第25章介绍的宏、标志和函数的单页摘要。熟悉材料后在键盘上使用。

关键内容包括：
- 限速日志宏`DLOG_RL`
- errno词汇表（常用errno及其用途）
- 可调参数系列宏
- sysctl标志摘要
- 版本标识符
- 能力位定义
- 标签化清理骨架
- 模块化驱动程序的文件布局
- 生产检查清单
- SYSINIT子系统ID
- EVENTHANDLER骨架

---

## 真实世界驱动程序演练（概要）

本章目前为止在`myfirst`伪驱动程序上构建其规范。本节将镜头转向，看看相同的规范如何出现在作为FreeBSD 14.3一部分发布的驱动程序中。每个演练从`/usr/src`中的真实源文件开始，命名正在运行的第25章模式，并指向模式可见的行。

阅读的真实驱动程序示例包括：
- `mmcsd(4)`：热路径上的限速错误日志
- `uftdi(4)`：模块依赖和PNP元数据
- `iscsi(4)`：在`attach`中注册的关闭处理程序
- `ufs_dirhash`：`vm_lowmem`上的缓存驱逐
- `if_vtnet`：特定故障周围的限速日志

---

## 总结

第25章是第5部分最长的章节，其长度是有意的。每一节都引入了将工作驱动程序转变为可维护驱动程序的规范。没有一个是华丽的；每一个都是在驱动程序被作者以外的人使用时保持其工作的特定习惯。

本章以限速日志开始。不能刷屏控制台的驱动程序是日志消息值得阅读的驱动程序。`ppsratecheck(9)`和`DLOG_RL`宏使规范机械化：将限速状态放在softc中，用宏包装热路径消息，让内核做每秒簿记。日志下游的一切都受益，因为您可以阅读的日志是可以从中调试的日志。

第二节命名errno值并区分它们。`ENOTTY`不是`EINVAL`；`EPERM`不是`EACCES`；`ENOIOCTL`是驱动程序不认识ioctl命令并希望内核在放弃之前尝试另一个处理程序的特殊信号。了解词汇表将模糊的错误报告转化为精确的报告，精确的错误报告更快到达根本原因。

第三节将配置视为一等关注。可调参数是启动时初始值；sysctl是运行时句柄。两者通过`TUNABLE_*_FETCH`和`CTLFLAG_TUN`配合。公开恰好正确的可调参数和sysctl的驱动程序给操作员足够的控制来解决自己的问题，而无需修改源代码。

第四节命名驱动程序需要的三个版本标识符（发布字符串、模块整数、ioctl线格式整数），并指出它们因不同原因更改。`GETCAPS` ioctl给用户空间一个可以在能力随时间添加和移除时存活的运行时发现机制。

第五节命名标签化goto模式并使其成为附加和分离的无条件规范。每个资源获得一个标签；每个标签穿透到下一个；单个`return (0)`将获取链与清理链分开。模式从两个资源扩展到十几个而不改变形状。

第六节将`myfirst`拆分为主题文件。每个文件一个关注点，决定新代码放在哪里的单一职责规则，以及承载每个文件需要的声明的小型公共头文件。

第七节将焦点向外转。生产就绪检查清单涵盖模块依赖、PNP元数据、`MOD_QUIESCE`、`devctl_notify`、`MAINTENANCE.md`、`devd`规则、日志消息质量、100次循环回归测试、输入验证、失败路径锻炼和版本控制规范。

第八节以`SYSINIT(9)`和`EVENTHANDLER(9)`闭环。需要参与自身附加/分离窗口之外内核生命周期的驱动程序有干净的机制来做。`myfirst`作为演示注册`shutdown_pre_sync`；其他驱动程序注册`vm_lowmem`、挂起/恢复或自定义事件。

驱动程序已从带有单个消息缓冲区的单个文件成长为模块化的、可观测的、有版本控制的、限速的和生产就绪的伪设备。FreeBSD字符驱动程序编写的机制现在掌握在读者手中。

尚未涵盖的是硬件世界。到目前为止，每一章都使用伪驱动程序；字符设备支持软件缓冲区和一组计数器。实际与硬件对话的驱动程序必须分配总线资源、映射I/O窗口、附加中断处理程序、编程DMA引擎，并经受真实设备的特定失败模式。第26章开始该工作。

---

## 第5部分检查点

第5部分涵盖调试、跟踪和将仅仅能编译的驱动程序与团队可以维护的驱动程序分开的工程实践。在第6部分通过将`myfirst`附加到真实传输介质改变地形之前，确认第5部分的习惯在您的手指中而不仅仅在笔记中。

第5部分结束后，您应该能够舒适地完成以下每项：
- 使用正确的工具调查驱动程序行为
- 阅读恐慌消息并提取正确的线索
- 构建和运行第24章集成栈`myfirst`
- 逐项应用第25章生产就绪检查清单
- 当驱动程序必须参与自身附加/分离窗口之外的内核生命周期时使用`SYSINIT(9)`和`EVENTHANDLER(9)`

如果这些都感觉不稳固，重新访问的实验室是：
- 调试工具实践：实验室23.1、23.2、23.4、23.5
- 集成和可观测性：第24章实验室1、4、5、6
- 生产规范：第25章实验室3、4、6、10

第6部分期望以下作为基线：
- 信心阅读恐慌并跟踪到根本原因
- 通过第25章生产检查清单的`myfirst`
- 意识到第6部分改变运行示例模式

如果这些成立，您就为第6部分准备好了。如果一个仍然不稳定，修复是一个精心选择的实验室，而不是匆忙向前。

---

## 通往第26章的桥梁

第26章将`myfirst`驱动程序向外转。驱动程序将附加到真实总线并服务真实设备，而不是服务RAM中的缓冲区。本书的第一个硬件目标是USB子系统：USB无处不在、文档齐全，并且具有比PCI或ISA更容易开始的干净内核接口。

您在第25章建立的习惯不变地延续。附加中的标签化goto模式扩展到总线资源分配。模块化文件布局自然扩展。生产检查清单添加两项。真正新的是驱动程序与USB子系统之间的接口。

在开始第26章之前，暂停并确认第25章材料扎实。实验室旨在运行；运行它们是找到章节中未落地部分的最佳方式。如果您已运行实验室1到7且挑战练习清晰，您就为硬件准备好了。

---

## 词汇表

**附加链。** 驱动程序`device_attach`方法中有序的资源获取序列。每个可能失败的获取后面跟着一个goto到撤销先前获取资源的标签。

**能力位掩码。** 由`MYFIRSTIOC_GETCAPS`返回的32位（或64位）整数，每个可选特性一位。让用户空间在运行时查询驱动程序支持哪些特性。

**清理链。** 驱动程序`device_attach`方法底部的有序标签序列。每个标签释放一个资源并穿透到下一个。获取顺序的反序。

**`CTLFLAG_SKIP`。** 从默认`sysctl(8)`列表中隐藏OID的sysctl标志。显式给出全名时OID仍可读。

**`CTLFLAG_RDTUN` / `CTLFLAG_RWTUN`。** 分别是`CTLFLAG_RD | CTLFLAG_TUN`和`CTLFLAG_RW | CTLFLAG_TUN`的简写。声明与加载器可调参数配合的sysctl。

**`devctl_notify`。** 发出`devd(8)`可读的结构化事件的内核函数。让驱动程序通知用户空间守护进程有趣的状态变化。

**`DLOG_RL`。** 在给定每秒消息速率上限日志消息的`ppsratecheck`和`device_printf`上的宏包装器。

**Errno。** 表示特定失败模式的小正整数。FreeBSD的errno表位于`/usr/src/sys/sys/errno.h`。

**事件处理器。** 用`EVENTHANDLER_REGISTER`注册的回调，每当内核触发命名事件时运行。用`EVENTHANDLER_DEREGISTER`注销。

**`EVENTHANDLER(9)`。** FreeBSD的通用事件通知框架。定义事件、让子系统发布它们、让驱动程序订阅。

**失败注入。** 导致代码路径失败以锻炼其清理的故意测试技术。通常实现为由`#ifdef`保护的条件`return`。

**标签化goto模式。** 参见"附加链"和"清理链"。FreeBSD对使用`goto label;`进行线性展开而不是嵌套`if`的附加和分离的习惯形状。

**`MOD_QUIESCE`。** 询问"你现在能被卸载吗？"的模块事件。驱动程序返回`EBUSY`拒绝；返回`0`接受。

**`MODULE_DEPEND`。** 声明对另一个内核模块依赖的宏。内核强制加载顺序和版本兼容性。

**`MODULE_PNP_INFO`。** 发布驱动程序处理的供应商/产品标识符的宏。内核使用元数据在匹配硬件出现时自动加载驱动程序。

**`MODULE_VERSION`。** 声明模块版本整数的宏。被`MODULE_DEPEND`用于兼容性检查。

**`myfirst_ratelimit`。** 保存每类限速状态（`lasttime`和`curpps`）的结构。必须位于softc中，而不是栈上。

**`MYFIRST_VERSION`。** 驱动程序的人类可读发布字符串，例如`"1.8-maintenance"`。通过`dev.myfirst.<unit>.version`公开。

**`MYFIRST_IOCTL_VERSION`。** 驱动程序的ioctl线格式版本整数。由`MYFIRSTIOC_GETVER`返回。仅为破坏性更改提升。

**Pps。** 每秒事件数。以pps表示的限速上限（例如10 pps = 每秒10条消息）。

**`ppsratecheck(9)`。** FreeBSD限速原语。接受`struct timeval`、`int *`和pps上限；返回非零允许事件。

**生产就绪。** 满足第7节检查清单的驱动程序：声明的依赖、有文档的接口、限速日志、锻炼的失败路径、通过100次循环测试。

**`SHUTDOWN_PRI_*`。** 传递给关闭事件的`EVENTHANDLER_REGISTER`的优先级常量。`FIRST`早运行；`LAST`晚运行；`DEFAULT`在中间运行。

**`SI_SUB_*`。** `SYSINIT`的子系统标识符。数值按启动顺序排序。常见常量：`SI_SUB_TUNABLES`、`SI_SUB_DRIVERS`、`SI_SUB_CONFIGURE`。

**`SI_ORDER_*`。** 子系统内`SYSINIT`的顺序常量。`FIRST`先运行；`MIDDLE`在中间运行；`ANY`最后运行（`ANY`条目之间无保证顺序）。

**单一职责规则。** 每个源文件回答关于驱动程序的一个问题。违规意味着新ioctl渗入`myfirst_sysctl.c`或新sysctl渗入`myfirst_ioctl.c`。

**Softc。** 每设备状态结构。Newbus分配它、清零它并通过`device_get_softc`将其交给`device_attach`。

**`sysctl(8)`。** 运行时参数的用户空间命令和内核接口。节点名称位于固定层次结构下（`kern.*`、`hw.*`、`net.*`、`dev.*`、`vm.*`、`debug.*`等）。

**`SYSINIT(9)`。** FreeBSD的启动时初始化宏。注册一个函数在内核启动或模块加载期间的特定子系统和顺序运行。

**`SYSUNINIT(9)`。** `SYSINIT`的配套。注册一个清理函数在模块卸载时以`SYSINIT`相反顺序运行。

**标签（事件处理器）。** 由`EVENTHANDLER_REGISTER`返回并被`EVENTHANDLER_DEREGISTER`消耗的不透明值。必须存储（通常在softc中）以便注销可以定位注册。

**可调参数。** 在启动或模块加载时从内核环境解析并通过`TUNABLE_*_FETCH`使用的值。设置初始值；位于`hw.`、`kern.`或`debug.`级别。

**`TUNABLE_INT_FETCH`、`_LONG_FETCH`、`_BOOL_FETCH`、`_STR_FETCH`。** 从内核环境读取可调参数并填充变量的宏系列。可调参数缺失时静默。

**版本拆分。** 使用三个独立版本标识符（发布字符串、模块整数、ioctl整数）的做法，每个因不同原因更改。

**`vm_lowmem`事件。** 虚拟内存子系统压力时触发的`EVENTHANDLER`事件。有缓存的驱动程序可以释放一些内存回去。

**线格式。** 跨用户/内核边界传递的数据布局。ioctl线格式由`_IOR`、`_IOW`、`_IOWR`声明和有效负载结构确定。线格式更改是破坏性更改，需要提升`MYFIRST_IOCTL_VERSION`。


## 第7节：为生产使用做准备

在你的工作站上工作的驱动程序不是准备好生产的驱动程序。生产是你的代码被安装在你不拥有的硬件上、由你永远不会见到的操作员引导、并预期在重新启动之间的几个月或几年内可预测行为时面临的条件集。"对我来说可以工作"和"准备好交付"之间的距离是用习惯衡量的，而不是用功能。本节命名这些习惯。

第25章形状的`myfirst`功能完整，正如伪驱动程序将要完成的那样。剩余的工作不是添加功能，而是加固边缘，以便驱动程序能够在它无法控制的环境中生存。

### 生产准备就绪的心态

心态转变是这样的：驱动程序在开发时间隐式做出的每个决定在生产时间都必须显式做出。在可调参数有默认值的地方，默认值必须是正确的默认值。在sysctl可写的地方，凌晨3点惊吓的操作员写入的后果必须是安全的。在日志消息可能触发的地方，消息必须在没有开发者帮助的情况下有用。在模块依赖于另一个模块的地方，依赖必须声明，以便加载器不会以错误的顺序加载它们。

生产准备就绪不是一次性行动；它是一种贯穿每个决定的态度。几乎准备好生产的驱动程序通常有一两个具体的差距：一个没有文档的可调参数、每微秒触发的日志消息、假设没有人正在使用设备的detach路径。生产准备就绪的规范是找到这些具体的差距并逐个关闭它们，直到驱动程序的行为在开发者不站在前面的机器上是可预测的。

### 声明模块依赖

第一个生产习惯是对模块需要什么显式说明。如果`myfirst`调用存在于另一个内核模块中的函数，内核的模块加载器需要在调用之前知道依赖关系，否则内核加载`myfirst`并在第一次使用依赖时panic。

机制是`MODULE_DEPEND`。第4节将其作为兼容性工具引入；在生产中，它也是一个正确性工具。没有在其真实依赖上使用`MODULE_DEPEND`的驱动程序在大多数引导顺序中偶然工作，在其他顺序中神秘失败。在真实依赖上对每个使用`MODULE_DEPEND`的驱动程序要么正确加载，要么拒绝加载并带有清晰的错误消息。

对于伪驱动程序`myfirst`，还没有真正的依赖；驱动程序仅使用内核核心中的符号，它始终存在。第26章USB变体将添加第一个真正的`MODULE_DEPEND`：

```c
MODULE_DEPEND(myfirst_usb, usb, 1, 1, 1);
```

三个版本号是`myfirst_usb`兼容的USB栈版本的最小、首选和最大版本。在加载时，内核根据此范围检查安装的USB栈版本，如果USB栈缺失或超出范围则拒绝加载`myfirst_usb`。

生产习惯是：在交付之前，grep驱动程序中它调用的每个符号，并确认每个符号要么存在于内核核心中，要么存在于驱动程序声明依赖的模块中。缺少的`MODULE_DEPEND`在引导顺序更改之前工作，然后驱动程序在生产硬件上panic。

### 发布PNP信息

对于硬件驱动程序，内核的模块加载器参考每个模块的PNP元数据来决定哪个驱动程序处理哪个设备。不发布PNP信息的USB驱动程序在手动加载时工作，在引导加载程序尝试为新插入的设备自动加载驱动程序时失败。修复是`MODULE_PNP_INFO`，驱动程序用它声明它处理的供应商/产品标识符：

```c
MODULE_PNP_INFO("U16:vendor;U16:product", uhub, myfirst_usb,
    myfirst_pnp_table, nitems(myfirst_pnp_table));
```

第一个字符串描述PNP表条目的格式。`uhub`是总线名；`myfirst_usb`是驱动程序名；`myfirst_pnp_table`是结构体静态数组，驱动程序处理的每个设备一个。

第25章的`myfirst`仍然是伪驱动程序，没有硬件可匹配。`MODULE_PNP_INFO`在第26章第一个真实硬件附加时发挥作用。对于第25章，生产习惯只是知道宏存在，并在硬件到达时为其计划。

### `MOD_QUIESCE`事件

内核模块事件处理程序被调用时有四个事件之一：`MOD_LOAD`、`MOD_UNLOAD`、`MOD_SHUTDOWN`、`MOD_QUIESCE`。大多数驱动程序显式处理`MOD_LOAD`和`MOD_UNLOAD`，内核合成其他两个。对于生产驱动程序，`MOD_QUIESCE`值得关注。

`MOD_QUIESCE`是内核的问题"你现在可以被卸载吗？"它在`MOD_UNLOAD`之前触发，给驱动程序一个干净拒绝的机会。正在操作的驱动程序（未完成的DMA传输、打开的文件描述符、待处理的定时器）可以从`MOD_QUIESCE`返回非零errno以拒绝卸载；内核然后不继续到`MOD_UNLOAD`。

对于`myfirst`，quiesce检查已经内置在`myfirst_detach`中：如果`sc_open_count > 0`，detach返回`EBUSY`。内核的模块加载器将该`EBUSY`传播回`kldunload(8)`，操作员看到"module myfirst is busy"。检查在正确的位置，但分开考虑`MOD_QUIESCE`和`MOD_UNLOAD`的规范值得命名：`MOD_QUIESCE`是"你安全可以卸载吗？"问题，`MOD_UNLOAD`是"继续卸载"命令。一些驱动程序有在`MOD_QUIESCE`中检查安全但在`MOD_UNLOAD`中获取不安全的状态；拆分它们让驱动程序回答问题而没有副作用。

### 发出`devctl_notify`事件

长时间运行的生产系统由像`devd(8)`这样的守护进程监控，它们监视设备到达、离开和状态更改。内核用来通知`devd`的机制是`devctl_notify(9)`：驱动程序发出结构化事件，`devd`读取事件，`devd`采取配置的操作（运行脚本、记录消息、通知操作员）。

原型是：

```c
void devctl_notify(const char *system, const char *subsystem,
    const char *type, const char *data);
```

- `system`是顶级类别，如`"DEVFS"`、`"ACPI"`或驱动程序特定的标签。
- `subsystem`是驱动程序或子系统名称。
- `type`是短事件名。
- `data`是可选的结构化数据（键值对），供守护进程解析。

对于`myfirst`，一个有用的生产事件是"驱动程序内的消息被重写"：

```c
devctl_notify("myfirst", device_get_nameunit(sc->sc_dev),
    "MSG_CHANGED", NULL);
```

在操作员通过`ioctl(fd, MYFIRSTIOC_SETMSG, buf)`写入新消息后，驱动程序发出`MSG_CHANGED`事件。`devd`规则可以匹配事件，例如，发送syslog条目或通知监控守护进程：

```text
notify 0 {
    match "system"    "myfirst";
    match "type"      "MSG_CHANGED";
    action "logger -t myfirst '$subsystem上的消息已更改'";
};
```

这里的生产习惯是，对于驱动程序中的每个有趣事件，问操作员是否可能想对其做出反应。如果答案为是，发出一个名称选择良好的`devctl_notify`。下游工具可以在事件上构建，驱动程序不必知道这些工具是什么。

### 编写`MAINTENANCE.md`

每个生产驱动程序都应该有一个维护文件，用简单的英语描述驱动程序做什么、接受什么可调参数、公开什么sysctl、处理什么ioctl、发出什么事件以及版本历史是什么。文件存储在存储库中的源代码旁边；它由操作员、新开发人员、安全审阅者和六个月后的作者阅读。

`MAINTENANCE.md`的具体骨架：

```text
# myfirst

一个承载本书运行示例的演示字符驱动程序。此文件是面向操作员的参考。

## 概述

myfirst在/dev/myfirst0注册一个伪设备，服务于一个
读写消息缓冲区、一组ioctl、一个sysctl树和一个
可配置的调试类别记录器。

## 可调参数

- hw.myfirst.debug_mask_default (int, 默认 0)
    dev.myfirst.<unit>.debug.mask的初始值。
- hw.myfirst.timeout_sec (int, 默认 5)
    dev.myfirst.<unit>.timeout_sec的初始值。
- hw.myfirst.max_retries (int, 默认 3)
    dev.myfirst.<unit>.max_retries的初始值。
- hw.myfirst.log_ratelimit_pps (int, 默认 10)
    初始限速上限（每类每秒打印数）。

## Sysctl

所有sysctl位于dev.myfirst.<unit>下。

只读：version、open_count、total_reads、total_writes、
message、message_len。

读写：debug.mask（debug_mask_default的镜像）、timeout_sec、
max_retries、log_ratelimit_pps。

## Ioctl

定义在myfirst_ioctl.h中。命令魔术'M'。

- MYFIRSTIOC_GETVER (0)：返回MYFIRST_IOCTL_VERSION。
- MYFIRSTIOC_RESET  (1)：清零读/写计数器。
- MYFIRSTIOC_GETMSG (2)：读取驱动程序内消息。
- MYFIRSTIOC_SETMSG (3)：写入驱动程序内消息。
- MYFIRSTIOC_GETCAPS (5)：返回MYF_CAP_*位掩码。

命令4在第23章草案工作期间保留，发布前
退役。不要重用该号码。

## 事件

通过devctl_notify(9)发出。

- system=myfirst subsystem=<unit> type=MSG_CHANGED
    操作员可见的消息被重写。

## 版本历史

见下面的变更日志。

## 变更日志

### 1.8-maintenance
- 添加了MYFIRSTIOC_GETCAPS（命令5）。
- 添加了timeout_sec、max_retries、log_ratelimit_pps的可调参数。
- 通过ppsratecheck(9)添加了限速日志记录。
- 为MSG_CHANGED添加了devctl_notify。
- 从1.7没有破坏性更改。

### 1.7-integration
- ioctl、sysctl、调试的首次端到端集成。
- 引入了MYFIRSTIOC_{GETVER,RESET,GETMSG,SETMSG}。

### 1.6-debug
- 添加了DPRINTF框架和SDT探针。
```

文件没有任何华丽的东西。它是一个参考，随着每次版本提升而保持最新，作为操作员的单一真理来源。

生产习惯是：对驱动程序可见表面的每个更改（新可调参数、新sysctl、新ioctl、新事件、行为更改）在`MAINTENANCE.md`中有相应的条目。文件永远不会落后于代码。`MAINTENANCE.md`过时的驱动程序是其用户在猜测的驱动程序；`MAINTENANCE.md`当前的驱动程序是其用户可以自助的驱动程序。

### `devd`规则集

`devd(8)`规则告诉守护进程如何对内核事件做出反应。对于`myfirst`的生产部署，最小的规则集将确保重要事件到达操作员：

```console
# /etc/devd/myfirst.conf
#
# myfirst驱动程序的devd规则。将此文件放入
# /etc/devd/并重启devd(8)以使规则生效。

notify 0 {
    match "system"    "myfirst";
    match "type"      "MSG_CHANGED";
    action "logger -t myfirst '$subsystem上的消息已更改'";
};

# 未来：一旦第26章的USB变体
# 开始发出它们，匹配attach/detach事件。
```

文件很短。它声明一个规则，匹配特定事件，采取特定操作。在生产中，这样的文件增长以匹配更多事件，触发更多操作，在一些部署中通知监视驱动程序异常的监控系统。

在驱动程序的存储库中包含草稿`devd.conf`使操作员易于采用。他们复制文件，调整操作，驱动程序的事件在第一天就集成到站点的监控中。

### 日志：支持工程师的朋友

生产驱动程序的日志消息由无法访问源代码且无法按需重现问题的支持工程师阅读。使日志消息对支持工程师有用的规则与使日志消息对开发人员有用的规则不同。

阅读自己日志消息的开发人员可以依赖支持工程师没有的上下文。支持工程师不能问"哪个attach？"或"哪个设备？"或"当这触发时`error`是什么？"答案必须已经在消息中。

生产习惯是审计驱动程序中的每个日志消息并问三个问题：

1. **消息命名其设备了吗？** `device_printf(dev, ...)`在输出前加上设备nameunit；裸`printf`不。每个不是来自`MOD_LOAD`（那里还没有设备）的消息应该是`device_printf`。

2. **消息包含相关的数字上下文吗？** "分配失败"没有用。"分配失败：错误12 (ENOMEM)"有用。"分配定时器失败：错误12"更好。

3. **消息以适当的速率出现吗？** 第1节涵盖了限速。最后的检查是确保每个可以在循环中触发的消息要么被限速，要么明显是一次性的。

满足这三个问题的日志消息带着足够的信息到达支持工程师，可以提交有用的错误报告。失败其中任何一个的日志消息浪费操作员的时间和开发人员的时间。

### 优雅地处理总线Attach/Detach

生产驱动程序，特别是热插拔驱动程序，必须在不泄漏的情况下处理重复的attach和detach循环。第5节标签清理模式的规范是答案的一部分；另一部分是确认重复的attach/detach实际上有效。章节末尾的实验通过一个回归脚本，该脚本连续加载、卸载和重新加载驱动程序100次，并验证模块的内存占用不会增长。

通过100次循环测试的驱动程序是在生产硬件上存活一个月热插拔事件的驱动程序。未通过100次循环测试的驱动程序有泄漏，随着时间的推移会表现为缓慢的内存增长或内核运行出某些有界资源（sysctl OID、cdev次设备号、devclass条目）。

测试运行简单，但价值不成比例。将其作为驱动程序发布前检查表的一部分。

### 处理意外的操作员操作

操作员会犯错。当测试程序正在从`/dev/myfirst0`读取时，他们运行`kldunload myfirst`。他们将`dev.myfirst.0.debug.mask`设置为同时启用每个类别的值。他们复制`MAINTENANCE.md`并跳过可调参数的部分。生产驱动程序必须容忍这些操作而不崩溃、不破坏状态、不将系统留在损坏的配置中。

对于每个暴露的接口，生产习惯是问：我能想象的最糟糕的操作员操作序列是什么，驱动程序能存活吗？

- 文件描述符打开时`kldunload`：`myfirst_detach`返回`EBUSY`。操作员看到"module busy"。驱动程序不变。
- 可写sysctl设置为超出范围的值：sysctl处理程序钳制值或返回`EINVAL`。驱动程序的内部状态不变。
- 消息长于缓冲区的`MYFIRSTIOC_SETMSG`：`strlcpy`截断。副本正确；截断在`message_len`中可见。
- 一对并发的`MYFIRSTIOC_SETMSG`调用：softc互斥锁序列化它们。第二个运行的获胜；两个都成功。

如果这些操作中的任何一个产生崩溃、破坏或不一致状态，驱动程序没有为生产做好准备。修复总是相同的：添加缺少的保护，重新启动测试，并添加注释记录不变式。

### 生产准备就绪检查表

本节的习惯适合一个开发人员在交付前可以遍历的短检查表：

```text
myfirst生产准备就绪
----------------------------

[  ] MODULE_DEPEND为每个真实依赖声明。
[  ] MODULE_PNP_INFO如果驱动程序绑定到硬件则声明。
[  ] MOD_QUIESCE没有副作用地回答"你可以卸载吗？"。
[  ] devctl_notify为操作员相关事件发出。
[  ] MAINTENANCE.md当前：可调参数、sysctl、ioctl、事件。
[  ] devd.conf片段与驱动程序一起包含。
[  ] 每个日志消息是device_printf，包含errno，
     如果可以在循环中触发则被限速。
[  ] attach/detach存活100次加载/卸载循环。
[  ] sysctl拒绝超出范围的值。
[  ] ioctl有效载荷被边界检查。
[  ] 通过故意注入测试失败路径。
[  ] 版本控制规范：MYFIRST_VERSION、MODULE_VERSION、
     MYFIRST_IOCTL_VERSION各自因自己的原因提升。
```

列表故意简短。十二项，其中大多数已被早期章节介绍的习惯处理。勾选每个框的驱动程序准备好由永远不会见你的人安装。

### `myfirst`驱动程序覆盖的内容

在第25章结束时通过检查表运行`myfirst`给出以下状态。

`MODULE_DEPEND`不需要，因为驱动程序没有子系统依赖；这在`MAINTENANCE.md`中显式注明。

`MODULE_PNP_INFO`不需要，因为驱动程序不绑定到硬件；这也在`MAINTENANCE.md`中注明。

`MOD_QUIESCE`由`myfirst_detach`中的`sc_open_count`检查回答；此版本未添加专用的`MOD_QUIESCE`处理程序，因为语义相同。

`devctl_notify`在`MYFIRSTIOC_SETMSG`上以事件类型`MSG_CHANGED`发出。

`MAINTENANCE.md`在示例目录中提供，包含可调参数、sysctl、ioctl、事件和1.8-maintenance的变更日志条目。

`devd.conf`片段与`MAINTENANCE.md`一起提供，演示单个`MSG_CHANGED`规则。

每个日志消息通过`device_printf`（或`DPRINTF`，它包装`device_printf`）发出；在热路径上触发的每个消息都包装在`DLOG_RL`中。

attach/detach回归脚本（见实验）运行100次循环而不增长内核的内存占用。

`timeout_sec`、`max_retries`和`log_ratelimit_pps`的sysctl在其处理程序中各自拒绝超出范围的值。

ioctl有效载荷在结构级别由内核的ioctl框架（`_IOR`、`_IOW`、`_IOWR`声明精确大小）边界检查，在字符串长度重要的驱动程序内部边界检查。

失败注入点在示例中由条件的`#ifdef`标记；每个标签在开发中至少到达一次。

版本标识符各自有自己的规则：字符串提升、模块整数提升、ioctl整数不变，因为添加是向后兼容的。

十二个检查，十二个结果。驱动程序准备好迎接下一章。

### 第7节总结

生产是将有趣代码与可交付代码分开的安静标准。这里命名的规范不华丽；它们是当代码部署在远离编写它的开发人员的地方时保持驱动程序工作的具体内容。`myfirst`经历了五章的教学内容，现在戴着让它在书外生存的马具。

在下一节中，我们转向两个内核基础设施，它们让驱动程序在特定的生命周期点运行代码而无需手动接线：`SYSINIT(9)`用于引导时初始化，`EVENTHANDLER(9)`用于运行时通知。这些是第26章将本书学到的所有内容应用于真实总线之前的FreeBSD工具包的最后两块。


## 第8节：SYSINIT、SYSUNINIT和EVENTHANDLER

驱动程序的attach和detach例程处理在实例化和拆卸之间发生的一切，但有些事情驱动程序可能需要在那个窗口之外做。有些代码必须在任何设备实例化之前运行：加载引导时可调参数、初始化子系统范围的锁、设置第一个`attach`将消耗的池。其他代码必须响应不是设备特定的系统范围事件：系统范围的挂起、低内存条件、即将同步文件系统并关机的关闭。

FreeBSD内核为这些情况提供了两种机制。`SYSINIT(9)`注册一个函数在特定的引导阶段运行，其伴随者`SYSUNINIT(9)`注册一个清理函数在模块卸载时运行。`EVENTHANDLER(9)`注册一个回调在内核触发命名事件时运行。

两种机制都自FreeBSD最早发布以来就可用。它们是无聊的基础设施；这就是它们的价值。正确使用它们的驱动程序可以对完整的内核生命周期做出反应，而无需编写一行手动注册代码。忽略它们的驱动程序要么错过提示，要么重新发明同一设施的更差版本。

### 为什么内核需要引导时排序

FreeBSD内核以精确的顺序引导。内存管理在任何分配器可用之前启动。可调参数在驱动程序可以读取它们之前解析。锁在任何东西被允许获取它们之前初始化。文件系统仅在它们所在的设备被探测之后挂载。这些依赖中的每一个都必须遵守，否则内核在`init(8)`启动之前panic。

强制排序的机制是`SYSINIT(9)`。`SYSINIT`宏声明一个给定函数应该在给定子系统ID和给定顺序常量下运行。内核的引导序列收集运行配置中的每个`SYSINIT`，按（子系统、顺序）排序它们，并以该序列调用它们。内核引导后加载的模块仍然遵守其`SYSINIT`声明：加载器在模块附加时以相同的排序顺序调用它们。

从驱动程序的角度来看，`SYSINIT`是一种说法"在引导序列的那一点做这件事，我不在乎还有哪些其他代码也在那一点注册"的方式。内核处理排序；驱动程序编写回调。

### 子系统ID空间

子系统ID在`/usr/src/sys/sys/kernel.h`中定义。常量具有描述性名称和反映其排序的数值。驱动程序选择与其回调目的对应的子系统：

- `SI_SUB_TUNABLES` (0x0700000)：评估引导时可调参数。这是`TUNABLE_INT_FETCH`及其兄弟运行的地方。消费可调参数的代码必须在此点之后运行。
- `SI_SUB_KLD` (0x2000000)：可加载内核模块设置。早期模块基础设施在这里运行。
- `SI_SUB_SMP` (0x2900000)：启动应用处理器。
- `SI_SUB_DRIVERS` (0x3100000)：让驱动程序初始化。这是大多数用户空间驱动程序在需要任何设备附加之前的早期代码时注册的子系统。
- `SI_SUB_CONFIGURE` (0x3800000)：配置设备。到此子系统结束时，每个编译进内核的驱动程序都有机会附加。

`kernel.h`中有一百多个子系统ID。上面的是字符设备驱动程序最常交互的。数值排序以便"较小的数字"意味着"引导中较早"。

在子系统内部，顺序常量给出细粒度排序：

- `SI_ORDER_FIRST` (0x0)：在同一子系统中的大多数其他代码之前运行。
- `SI_ORDER_SECOND`、`SI_ORDER_THIRD`：显式的逐步排序。
- `SI_ORDER_MIDDLE` (0x1000000)：在中间运行。大多数驱动程序级别的`SYSINIT`使用这个或下面的。
- `SI_ORDER_ANY` (0xfffffff)：最后运行。内核不保证`SI_ORDER_ANY`条目之间的任何特定顺序。

驱动程序作者选择使回调在其前置条件之后、在其依赖者之前运行的最低顺序。对于大多数目的，`SI_ORDER_MIDDLE`是正确的。

### 驱动程序何时需要`SYSINIT`

大多数字符设备驱动程序完全不需要`SYSINIT`。`DRIVER_MODULE`已经向Newbus注册了驱动程序；驱动程序的`device_attach`方法在匹配的设备出现时运行。这对每实例的工作来说足够了。

`SYSINIT`用于不是每实例的工作。驱动程序可能注册`SYSINIT`的原因列表：

- **初始化一个全局池**，驱动程序的每个实例将从其中提取。池存在一次；它不属于任何一个softc。
- **向一个内核子系统注册**，该子系统预期调用者在使用它之前注册。例如，想要接收`vm_lowmem`事件的驱动程序需要早期注册，以便第一个低内存事件不会错过它。
- **解析一个复杂可调参数**，需要比单个`TUNABLE_INT_FETCH`更多的工作。可调参数解析代码在`SI_SUB_TUNABLES`期间运行，并填充一个全局结构，每实例代码稍后参考。
- **自检**一个加密原语或子系统初始化器，在第一个调用者可以使用它之前。

对于`myfirst`，这些今天都不适用。驱动程序是每实例的，其可调参数很简单，它不使用需要预注册的子系统。第25章引入`SYSINIT`不是因为`myfirst`需要一个，而是因为读者应该熟悉该宏，并理解未来何时会需要它。

### `SYSINIT`声明的形状

宏签名是：

```c
SYSINIT(uniquifier, subsystem, order, func, ident);
```

- `uniquifier`是一个C标识符，将`SYSINIT`符号绑定到此声明。它不出现在其他任何地方。约定是使用与子系统或函数匹配的短名称。
- `subsystem`是`SI_SUB_*`常量。
- `order`是`SI_ORDER_*`常量。
- `func`是一个函数指针，签名为`void (*)(void *)`。
- `ident`是传递给`func`的单个参数。对于大多数用途，它是`NULL`。

匹配的清理宏是：

```c
SYSUNINIT(uniquifier, subsystem, order, func, ident);
```

`SYSUNINIT`注册一个清理函数。它在模块卸载时以`SYSINIT`声明的相反顺序运行。对于编译到内核中（不是模块）的代码，`SYSUNINIT`永远不会触发，因为内核永远不会卸载；但声明仍然有用，因为将驱动程序编译为模块会测试清理路径。

### `myfirst`的`SYSINIT`工作示例

考虑对`myfirst`的一个假设增强：一个全局的、驱动程序范围的预分配日志缓冲区池，每个实例可以从中提取。池每次模块加载初始化一次，每次模块卸载销毁一次。每实例的attach和detach不直接触及池；它们只从池中取走和归还缓冲区。

`SYSINIT`声明看起来像这样：

```c
#include <sys/kernel.h>

static struct myfirst_log_pool {
	struct mtx       lp_mtx;
	/* ... 每池状态 ... */
} myfirst_log_pool;

static void
myfirst_log_pool_init(void *unused __unused)
{
	mtx_init(&myfirst_log_pool.lp_mtx, "myfirst log pool",
	    NULL, MTX_DEF);
	/* 分配池条目。 */
}

static void
myfirst_log_pool_fini(void *unused __unused)
{
	/* 释放池条目。 */
	mtx_destroy(&myfirst_log_pool.lp_mtx);
}

SYSINIT(myfirst_log_pool,  SI_SUB_DRIVERS, SI_ORDER_MIDDLE,
    myfirst_log_pool_init, NULL);
SYSUNINIT(myfirst_log_pool, SI_SUB_DRIVERS, SI_ORDER_MIDDLE,
    myfirst_log_pool_fini, NULL);
```

当`myfirst`被加载时，内核排序`SYSINIT`条目并在`SI_SUB_DRIVERS`阶段调用`myfirst_log_pool_init`。之后运行的第一个`myfirst_attach`发现池已准备好。当模块被卸载时，`myfirst_log_pool_fini`在每个实例被分离后运行，给池一个机会释放其资源。

这是一个用于教学目的的草图；`myfirst`在交付的第25章代码中实际上不使用全局池。最终编写需要它的驱动程序的读者会在这里找到该模式。

### `SYSINIT`和`DRIVER_MODULE`之间的排序

`DRIVER_MODULE`本身在底层作为`SYSINIT`实现。它在特定的子系统阶段向Newbus注册驱动程序，Newbus自己的`SYSINIT`随后探测和附加设备。驱动程序的自定义`SYSINIT`因此可以通过选择正确的子系统和顺序相对于`DRIVER_MODULE`排序。

经验法则：

- `SI_SUB_DRIVERS`以`SI_ORDER_FIRST`的`SYSINIT`在`DRIVER_MODULE`的注册之前运行。
- `SI_SUB_CONFIGURE`以`SI_ORDER_MIDDLE`的`SYSINIT`在大多数设备附加之后但在最终配置步骤之前运行。

对于attach依赖的全局池，`SI_SUB_DRIVERS`以`SI_ORDER_MIDDLE`通常是正确的：池在`DRIVER_MODULE`的设备开始附加之前初始化（因为`SI_SUB_DRIVERS`早于`SI_SUB_CONFIGURE`），顺序常量使其远离最早的钩子。

### `EVENTHANDLER`：响应运行时事件

`SYSINIT`在已知的引导阶段触发一次。`EVENTHANDLER`在特定系统事件发生时触发零次或多次。两种机制是表亲；它们互补。

内核定义了许多命名事件。每个事件有固定的回调签名和固定的触发环境。关心事件的驱动程序注册回调；内核每次事件触发时调用回调；驱动程序在detach时注销回调。

一些常用的事件：

- `shutdown_pre_sync`：系统即将同步文件系统。具有内存缓存的驱动程序在这里刷新它们。
- `shutdown_post_sync`：系统已完成同步文件系统。需要知道"文件系统安静"的驱动程序在这里挂钩。
- `shutdown_final`：系统即将停止或重新启动。需要保存硬件状态的驱动程序在这里做。
- `vm_lowmem`：虚拟内存子系统承受压力。有自己的缓存的驱动程序应该释放一些内存回去。
- `power_suspend_early`、`power_suspend`、`power_resume`：挂起/恢复生命周期。
- `dev_clone`：设备克隆事件，用于按需出现的伪设备。

列表不是固定的；随着内核的增长会添加新事件。上面的是通用驱动程序最常考虑的事件。

### `EVENTHANDLER`注册的形状

模式有三个部分：用正确的签名声明处理函数，在attach时注册它，在detach时注销它。注册返回一个不透明的标签；注销需要该标签。

对于`shutdown_pre_sync`，处理程序签名是：

```c
void (*handler)(void *arg, int howto);
```

`arg`是驱动程序传递给注册的任何指针；通常是softc。`howto`是关闭标志（`RB_HALT`、`RB_REBOOT`等）。

`myfirst`的最小关闭处理程序：

```c
#include <sys/eventhandler.h>

static eventhandler_tag myfirst_shutdown_tag;

static void
myfirst_shutdown(void *arg, int howto)
{
	struct myfirst_softc *sc = arg;

	mtx_lock(&sc->sc_mtx);
	DPRINTF(sc, MYF_DBG_INIT, "shutdown: howto=0x%x\n", howto);
	/* 在这里刷新任何待处理状态。 */
	mtx_unlock(&sc->sc_mtx);
}
```

注册发生在`myfirst_attach`内部（或从其调用的助手中）：

```c
myfirst_shutdown_tag = EVENTHANDLER_REGISTER(shutdown_pre_sync,
    myfirst_shutdown, sc, SHUTDOWN_PRI_DEFAULT);
```

注销发生在`myfirst_detach`内部：

```c
EVENTHANDLER_DEREGISTER(shutdown_pre_sync, myfirst_shutdown_tag);
```

注销是强制的。未经注销就分离的驱动程序在内核事件列表中留下悬空的回调指针。当内核下次触发事件时，它调用不再映射的内存区域，系统panic。

存储在`myfirst_shutdown_tag`中的标签是将注册绑定到注销的东西。对于具有单个实例的驱动程序，像上面这样的静态变量可以工作。对于具有多个实例的驱动程序，标签应该存储在softc中，以便每个实例的注销引用自己的标签。

### Attach链中的`EVENTHANDLER`

因为注册和注销是对称的，它们干净地适合第5节的标签清理模式。注册成为获取；其失败模式是"注册是否返回错误？"（在低内存条件下可能失败）；其清理是`EVENTHANDLER_DEREGISTER`。

一个感知`EVENTHANDLER`的`myfirst`的更新attach片段：

```c
	/* 阶段7：关闭处理程序。 */
	sc->sc_shutdown_tag = EVENTHANDLER_REGISTER(shutdown_pre_sync,
	    myfirst_shutdown, sc, SHUTDOWN_PRI_DEFAULT);
	if (sc->sc_shutdown_tag == NULL) {
		error = ENOMEM;
		goto fail_log;
	}

	return (0);

fail_log:
	myfirst_log_detach(sc);
fail_cdev:
	destroy_dev(sc->sc_cdev);
fail_mtx:
	mtx_destroy(&sc->sc_mtx);
	return (error);
```

以及匹配的detach，注销在前（获取的相反顺序）：

```c
	EVENTHANDLER_DEREGISTER(shutdown_pre_sync, sc->sc_shutdown_tag);
	myfirst_log_detach(sc);
	destroy_dev(sc->sc_cdev);
	mtx_destroy(&sc->sc_mtx);
```

`sc->sc_shutdown_tag`存储在softc中。将其存储在那里很重要：注销需要知道要移除哪个具体的注册，每softc存储保持驱动程序的两个实例独立。

### 优先级：`SHUTDOWN_PRI_*`

在单个事件中，回调按优先级顺序调用。优先级是`EVENTHANDLER_REGISTER`的第四个参数。对于关闭事件，常见的常量是：

- `SHUTDOWN_PRI_FIRST`：在大多数其他处理程序之前运行。
- `SHUTDOWN_PRI_DEFAULT`：以默认顺序运行。
- `SHUTDOWN_PRI_LAST`：在其他处理程序之后运行。

具有需要在文件系统刷新之前安静化的硬件的驱动程序可能以`SHUTDOWN_PRI_FIRST`注册。状态依赖于文件系统已经刷新的驱动程序（在实践中不太可能）可能以`SHUTDOWN_PRI_LAST`注册。大多数驱动程序使用`SHUTDOWN_PRI_DEFAULT`，不考虑优先级。

其他事件存在类似的优先级常量（`EVENTHANDLER_PRI_FIRST`、`EVENTHANDLER_PRI_ANY`、`EVENTHANDLER_PRI_LAST`）。

### 何时使用`vm_lowmem`

`vm_lowmem`是VM子系统在空闲内存降至阈值以下时触发的事件。维护自己的缓存（例如预分配块池）的驱动程序可以响应释放一些回内核。

处理程序用一个参数（触发事件的子系统ID）调用。具有缓存的驱动程序的最小处理程序：

```c
static void
myfirst_lowmem(void *arg, int unused __unused)
{
	struct myfirst_softc *sc = arg;

	mtx_lock(&sc->sc_mtx);
	/* 从缓存中释放一些条目。 */
	mtx_unlock(&sc->sc_mtx);
}
```

注册看起来与关闭注册相同，但事件名不同：

```c
sc->sc_lowmem_tag = EVENTHANDLER_REGISTER(vm_lowmem,
    myfirst_lowmem, sc, EVENTHANDLER_PRI_ANY);
```

不维护缓存的驱动程序不应注册`vm_lowmem`。这样做的成本不是零：内核在每个低内存事件上调用每个已注册的处理程序，无操作的处理程序为该调用链增加延迟。

对于`myfirst`，没有缓存，所以不使用`vm_lowmem`。为即将编写需要它的驱动程序的读者引入该模式。

### `power_suspend_early`和`power_resume`

挂起/恢复是一个敏感的生命周期。在`power_suspend_early`和`power_resume`之间，驱动程序的设备预期是静止的：没有I/O、没有中断、没有状态转换。具有必须在挂起之前保存和恢复之后恢复的硬件状态的驱动程序为两个事件注册处理程序。

对于不管理硬件的字符设备驱动程序，这些事件通常不适用。对于总线附加的驱动程序（PCI、USB、SPI），总线层处理大部分挂起/恢复簿记，驱动程序只需在其`device_method_t`表中提供`device_suspend`和`device_resume`方法。`EVENTHANDLER`方法适用于想要响应系统范围挂起而不被总线附加的驱动程序。

第26章将在`myfirst`成为USB驱动程序时重新审视挂起/恢复；那时总线层的机制是首选。

### 模块事件处理程序

与`SYSINIT`和`EVENTHANDLER`相关的是模块事件处理程序：内核为`MOD_LOAD`、`MOD_UNLOAD`、`MOD_QUIESCE`和`MOD_SHUTDOWN`调用的回调。大多数驱动程序不覆盖它；`DRIVER_MODULE`提供一个默认实现，适当调用`device_probe`和`device_attach`。

在模块加载时需要自定义行为的驱动程序（超出`SYSINIT`所能做的）可以提供自己的处理程序：

```c
static int
myfirst_modevent(module_t mod, int what, void *arg)
{
	switch (what) {
	case MOD_LOAD:
		/* 自定义加载行为。 */
		return (0);
	case MOD_UNLOAD:
		/* 自定义卸载行为。 */
		return (0);
	case MOD_QUIESCE:
		/* 我们可以被卸载吗？如果不能则返回errno。 */
		return (0);
	case MOD_SHUTDOWN:
		/* 关闭通知；通常无操作。 */
		return (0);
	default:
		return (EOPNOTSUPP);
	}
}
```

处理程序通过`moduledata_t`结构而不是`DRIVER_MODULE`连接。两种方法对于给定的模块名是互斥的；驱动程序选择其中之一。

对于大多数驱动程序，`DRIVER_MODULE`的默认是正确的，模块事件处理程序不被自定义。`myfirst`始终使用`DRIVER_MODULE`。

### 注销规范

使用`EVENTHANDLER`时最重要的单一规则是：注册一次，注销一次，分别在attach和detach中。违反规则时会出现两种失败模式。

第一种失败模式是**遗漏注销**。Detach运行，标签未注销，内核的事件列表仍然指向softc的处理程序，下一个事件触发到已释放的内存中。panic发生在远离原因的地方，因为下一个事件可能在detach后几分钟或几小时才触发。

修复是机械的：attach中的每个`EVENTHANDLER_REGISTER`在detach中都有匹配的`EVENTHANDLER_DEREGISTER`。第5节的标签清理模式使这变得容易：注册是带标签的获取，清理链以相反顺序注销。

第二种失败模式是**双重注册**。注册同一处理程序两次的驱动程序在内核事件列表中有两个条目；分离一次仅移除其中一个。内核然后有一个指向刚刚消失的softc的陈旧条目。

修复也是机械的：每个attach恰好注册一次。不要在从多个地方调用的助手中注册；不要懒散地响应第一个事件来注册。

### 完整生命周期示例

将`SYSINIT`、`EVENTHANDLER`和标签清理attach放在一起，具有全局池和关闭处理程序的`myfirst`驱动程序的完整生命周期运行如下：

在内核引导或模块加载时：
- `SI_SUB_TUNABLES`触发。attach中的`TUNABLE_*_FETCH`调用将看到其值。
- `SI_SUB_DRIVERS`触发。`myfirst_log_pool_init`运行（通过`SYSINIT`）。全局池已准备好。
- `SI_SUB_CONFIGURE`触发。`DRIVER_MODULE`注册驱动程序。Newbus探测；`myfirst_probe`和`myfirst_attach`为每个实例运行。
- 在`myfirst_attach`内部：锁、cdev、sysctl、日志状态、关闭处理程序已注册。

在运行时：
- `ioctl(fd, MYFIRSTIOC_SETMSG, buf)`更新消息。
- `devctl_notify`发出`MSG_CHANGED`；`devd`记录它。

在关闭时：
- 内核触发`shutdown_pre_sync`。`myfirst_shutdown`为每个注册的处理程序运行。
- 文件系统同步。
- `shutdown_final`触发。机器停止。

在模块卸载时（关闭之前）：
- `MOD_QUIESCE`触发。如果有设备正在使用，`myfirst_detach`返回`EBUSY`。
- `MOD_UNLOAD`触发。`myfirst_detach`为每个实例运行：注销处理程序、释放日志状态、销毁cdev、销毁锁。
- `SYSUNINIT`触发。`myfirst_log_pool_fini`运行。全局池被释放。
- 模块取消映射。

每一步都在一个明确定义的位置。每个获取都有匹配的释放。紧密遵循该模式的驱动程序是FreeBSD内核可以加载、运行和卸载任意次数而不累积状态的驱动程序。

### 决定注册什么

决定是否为事件注册的驱动程序作者应该问三个问题。

首先，**事件对这个驱动程序真的重要吗？** `vm_lowmem`对有缓存的驱动程序重要；对没有缓存的驱动程序是噪音。`shutdown_pre_sync`对需要安静化硬件的驱动程序重要；对伪驱动程序是噪音。不做有用事情的处理程序仍在每个事件上被调用，在每次触发时稍微减慢系统。

其次，**事件是正确的吗？** FreeBSD有几个关闭事件。`shutdown_pre_sync`在文件系统同步之前触发；`shutdown_post_sync`在之后触发；`shutdown_final`在停止之前触发。注册错误阶段的驱动程序可能过早刷新缓存（在应该刷新的数据之前）或过晚（在文件系统已经消亡之后）。

第三，**事件在内核版本间稳定吗？** `shutdown_pre_sync`长期以来一直稳定，可以安全使用。更新或更专门的事件可能在发布之间更改签名。针对特定FreeBSD发布（本书与14.3对齐）的驱动程序可以依赖该发布中的事件；针对一系列发布的驱动程序必须更加小心。

对于`myfirst`，交付的第25章作为演示注册`shutdown_pre_sync`。处理程序是无操作的：它只是记录关闭正在开始。注册、注销和标签清理是示例的重点，不是处理程序体。

### `SYSINIT`和`EVENTHANDLER`的常见错误

首次使用这些机制时会重复出现一些错误。

第一个错误是**在`SYSINIT`中运行重代码**。引导时代码运行在许多内核子系统仍在初始化的上下文中。调用复杂子系统的`SYSINIT`可能与该子系统自己的初始化竞争。规则是：`SYSINIT`代码应该最小且自包含。复杂的初始化属于驱动程序的attach例程，它在每个子系统启动后运行。

第二个错误是**使用`SYSINIT`而不是`device_attach`**。`SYSINIT`每次模块加载运行一次，但`device_attach`每个设备运行一次。在`SYSINIT`中初始化每设备状态的驱动程序犯了类别错误；每设备状态在`SYSINIT`时还不存在。

第三个错误是**忘记`EVENTHANDLER_REGISTER`上的优先级参数**。函数接受四个参数：事件名、回调、参数、优先级。一些驱动程序忘记优先级，传递了错误数量的参数；编译器用错误捕捉这个问题，但碰巧意外传递`0`的驱动程序以最低可能的优先级注册，这可能是错误的。

第四个错误是**不清零标签字段**。如果在失败路径上调用`EVENTHANDLER_DEREGISTER`时`sc->sc_shutdown_tag`未初始化，注销尝试移除从未注册的标签。内核检测到这一点（标签在其事件列表中不存在）注销是无操作，但模式很脆弱。更清晰的规范是在分配时将softc清零（Newbus通过`device_get_softc`自动执行此操作，但自己分配softc的驱动程序必须手动执行）并且永远不要到达未注册标签的注销。

### 第8节总结

`SYSINIT`和`EVENTHANDLER`是内核让驱动程序参与超出自身attach/detach窗口的生命周期的方式。`SYSINIT`在特定的引导阶段运行代码；`EVENTHANDLER`响应命名的内核事件运行代码。它们一起覆盖了每设备代码不够、驱动程序必须与整个系统接触的情况。

第25章末尾的`myfirst`使用`EVENTHANDLER_REGISTER`进行演示`shutdown_pre_sync`处理程序；注册、注销和标签清理形状都已到位。引入了`SYSINIT`但未使用，因为`myfirst`今天没有全局池。模式已种下；当未来章节的驱动程序确实需要它们时，读者将立即认出它们。

第8节完成后，本章要教授的每个机制都在驱动程序中。章节的剩余材料通过动手实验、挑战练习和出问题时的问题排除参考来应用这些机制。


## 动手实验

本节的实验在真实的FreeBSD 14.3系统上练习本章的机制。每个实验都有特定的可测量结果；运行实验后，你应该能够陈述你看到了什么以及它意味着什么。实验假设你有`examples/part-05/ch25-advanced/`伴随目录在手。

开始之前，构建`ch25-advanced/`顶部交付的驱动程序：

```console
# cd examples/part-05/ch25-advanced
# make clean
# make
# kldload ./myfirst.ko
# ls /dev/myfirst*
/dev/myfirst0
```

如果这些步骤中有任何失败，在继续之前修复工具链或源代码。实验假设有工作的基线。

### 实验1：重现日志洪水

目的：看到在热循环中触发时无限速`device_printf`和限速`DLOG_RL`之间的区别。

来源：`examples/part-05/ch25-advanced/lab01-log-flood/`包含一个小型用户空间程序，该程序以内核允许的最快速度对`/dev/myfirst0`调用`read()` 10,000次。

步骤1. 临时设置调试掩码以启用I/O类别和读路径上的printf：

```console
# sysctl dev.myfirst.0.debug.mask=0x4
```

掩码位`0x4`启用`MYF_DBG_IO`，读回调使用它。

步骤2. 首先用朴素的`DPRINTF`版本的驱动程序运行洪水。从`lab01-log-flood/unlimited/`构建并加载`myfirst-flood-unlimited.ko`：

```console
# make -C lab01-log-flood/unlimited
# kldunload myfirst
# kldload lab01-log-flood/unlimited/myfirst.ko
# dmesg -c > /dev/null
# ./lab01-log-flood/flood 10000
# dmesg | wc -l
```

预期结果：`dmesg`中大约10,000行。控制台也可能被填满；系统的日志缓冲区会循环，较早的消息会丢失。

步骤3. 卸载并从`lab01-log-flood/limited/`重新加载限速版本，它使用10 pps上限的`DLOG_RL`：

```console
# kldunload myfirst
# kldload lab01-log-flood/limited/myfirst.ko
# dmesg -c > /dev/null
# ./lab01-log-flood/flood 10000
# sleep 5
# dmesg | wc -l
```

预期结果：`dmesg`中大约50行。洪水现在每秒最多发出10条消息；10秒的测试窗口产生大约50条消息（第一秒的突发令牌加上后续秒的配额，所以确切计数可能不同，但应该在十以内）。

步骤4. 并排比较两个输出。限速版本是可读的；无限速版本不可读。两个驱动程序都有相同的读行为；只有日志记录规范不同。

记录：两种情况下洪水完成所用的挂钟时间。无限速版本明显较慢，因为控制台输出本身是瓶颈。限速有可见的性能好处以及清晰度好处。

### 实验2：使用`truss`进行errno审计

目的：看到当驱动程序返回不同的errno值时`truss(1)`报告什么，并校准你对从哪个代码路径返回哪个errno的直觉。

来源：`examples/part-05/ch25-advanced/lab02-errno-audit/`包含一个用户程序，该程序进行一系列故意无效的调用，以及一个在`truss`下运行它的脚本。

步骤1. 如果尚未加载，加载标准`myfirst.ko`：

```console
# kldload ./myfirst.ko
```

步骤2. 在`truss`下运行审计程序：

```console
# truss -f -o /tmp/audit.truss ./lab02-errno-audit/audit
# less /tmp/audit.truss
```

程序按顺序执行这些操作：
1. 打开`/dev/myfirst0`。
2. 发出未知ioctl命令（命令号99）。
3. 以NULL参数发出`MYFIRSTIOC_SETMSG`。
4. 写入零长度缓冲区。
5. 写入大于驱动程序接受的缓冲区。
6. 将`dev.myfirst.0.timeout_sec`设置为大于允许的值。
7. 关闭。

步骤3. 在`truss`输出中，找到每个操作并记下其errno。预期结果：

1. `open`：返回文件描述符。无errno。
2. `ioctl(_IOC=0x99)`：返回`ENOTTY`（内核将驱动程序的`ENOIOCTL`翻译为此值）。
3. `ioctl(MYFIRSTIOC_SETMSG, NULL)`：返回`EFAULT`（内核在处理程序运行前捕获NULL）。
4. `write(0 bytes)`：返回`0`（无错误，只是没有写入字节）。
5. `write(oversize)`：返回`EINVAL`（驱动程序拒绝超过其缓冲区大小的长度）。
6. `sysctl write out-of-range`：返回`EINVAL`（sysctl处理程序拒绝该值）。
7. `close`：返回0。无errno。

步骤4. 对于每个观察到的errno，定位返回它的驱动程序代码。从`truss`到内核源代码走调用链，确认你在`truss`中看到的errno是驱动程序返回的。此练习校准你"用户看到什么"和"驱动程序说什么"之间的心智映射。

### 实验3：可调参数重启行为

目的：验证加载器可调参数在模块首次加载时确实更改驱动程序的初始状态。

来源：`examples/part-05/ch25-advanced/lab03-tunable-reboot/`包含一个辅助脚本`apply_tunable.sh`。

步骤1. 加载标准模块且不设置可调参数，确认超时的初始值：

```console
# kldload ./myfirst.ko
# sysctl dev.myfirst.0.timeout_sec
dev.myfirst.0.timeout_sec: 5
```

默认值是5，在attach例程中设置。

步骤2. 卸载模块，设置加载器可调参数，重新加载，确认新的初始值：

```console
# kldunload myfirst
# kenv hw.myfirst.timeout_sec=12
# kldload ./myfirst.ko
# sysctl dev.myfirst.0.timeout_sec
dev.myfirst.0.timeout_sec: 12
```

通过`kenv(1)`设置的可调参数生效，因为attach中的`TUNABLE_INT_FETCH`在sysctl发布之前读取了它。

步骤3. 在运行时更改sysctl并确认更改被接受但不会传播回可调参数：

```console
# sysctl dev.myfirst.0.timeout_sec=25
dev.myfirst.0.timeout_sec: 12 -> 25
# kenv hw.myfirst.timeout_sec
hw.myfirst.timeout_sec="12"
```

可调参数仍然读取12；sysctl读取25。可调参数是初始值；sysctl是运行时值。它们在sysctl被写入的那一刻开始不同。

步骤4. 卸载并重新加载。可调参数值仍然是12（因为它在内核环境中），所以新sysctl从12开始，而不是25。这是生命周期：可调参数设置初始值，sysctl设置运行时值，卸载丢失运行时值，可调参数存活。

步骤5. 清除可调参数并重新加载：

```console
# kldunload myfirst
# kenv -u hw.myfirst.timeout_sec
# kldload ./myfirst.ko
# sysctl dev.myfirst.0.timeout_sec
dev.myfirst.0.timeout_sec: 5
```

回到attach时的默认值。生命周期端到端一致。

### 实验4：故意Attach失败注入

目的：验证清理链中的每个标签都到达，并且在attach中间注入失败时没有资源泄漏。

来源：`examples/part-05/ch25-advanced/lab04-failure-injection/`包含模块的四个构建变体，每个编译不同的失败注入点：

- `inject-mtx/`：在锁初始化后立即失败。
- `inject-cdev/`：在cdev创建后立即失败。
- `inject-sysctl/`：在sysctl树构建后立即失败。
- `inject-log/`：在日志状态初始化后立即失败。

每个变体仅定义第5节中`MYFIRST_DEBUG_INJECT_FAIL_*`宏中的一个。

步骤1. 构建并加载第一个变体。加载应该失败：

```console
# make -C lab04-failure-injection/inject-mtx
# kldload lab04-failure-injection/inject-mtx/myfirst.ko
kldload: an error occurred while loading module myfirst. Please check dmesg(8) for more details.
# dmesg | tail -3
myfirst0: attach: 阶段1完成
myfirst0: attach: 在mtx_init后注入失败
device_attach: myfirst0 attach returned 12
```

attach函数在注入的失败点返回`ENOMEM`（errno 12）。模块未加载：

```console
# kldstat -n myfirst
kldstat: can't find file: myfirst
```

步骤2. 对其他三个变体重复。每个应该在名称建议的特定阶段失败，每个应该将内核留在干净状态。要确认干净状态，检查剩余的sysctl OID、剩余的cdev和剩余的锁：

```console
# sysctl dev.myfirst 2>&1 | head
sysctl: unknown oid 'dev.myfirst'
# ls /dev/myfirst* 2>&1
ls: No match.
# dmesg | grep -i "witness\|leak"
```

预期：没有匹配。没有sysctl，没有cdev，没有witness投诉。清理链在工作。

步骤3. 运行自动构建每个变体并检查结果的组合回归脚本：

```console
# ./lab04-failure-injection/run.sh
```

脚本构建每个变体，加载它，确认加载失败，确认状态干净，并报告每个变体的单行摘要。所有四个变体通过意味着清理链中的每个标签都在真实内核上被测试，并释放了在该标签处持有的每个资源。

### 实验5：`shutdown_pre_sync`处理程序

目的：确认注册的关闭处理程序在真实关闭期间确实触发，并观察其相对于文件系统同步的排序。

来源：`examples/part-05/ch25-advanced/lab05-shutdown-handler/`包含一个版本的`myfirst.ko`，其`shutdown_pre_sync`处理程序向控制台打印一条独特的消息。

步骤1. 加载模块并通过读取attach上的日志验证处理程序已注册：

```console
# kldload lab05-shutdown-handler/myfirst.ko
# dmesg | tail -1
myfirst0: attach: shutdown_pre_sync处理程序已注册
```

步骤2. 发出重启。在测试机器上（不是生产机器），最简单的方式是：

```console
# shutdown -r +1 "测试myfirst关闭处理程序"
```

在机器关闭时观看控制台。预期序列：

```text
myfirst0: shutdown: howto=0x4
Syncing disks, buffers remaining... 0 0 0
Uptime: ...
```

`myfirst0: shutdown: howto=0x4`行出现在"Syncing disks"**之前**，因为`shutdown_pre_sync`在文件系统同步之前触发。如果处理程序消息出现在同步消息之后，注册在了错误的事件（`shutdown_post_sync`或`shutdown_final`）。如果消息从未出现，处理程序从未注册或从未注销（双重释放会导致panic，但静默缺失暗示注册错误）。

步骤3. 机器重启后，确认在关闭前卸载仍然干净地移除处理程序：

```console
# kldload lab05-shutdown-handler/myfirst.ko
# kldunload myfirst
# dmesg | tail -2
myfirst0: detach: 关闭处理程序已注销
myfirst0: detach: 完成
```

注销消息确认detach中的清理路径运行。attach/detach对是对称的；没有事件列表条目泄漏。

### 实验6：100次循环回归脚本

目的：运行持续的加载/卸载循环以捕获仅在重复attach/detach下出现的泄漏。这是第7节生产检查表中的测试。

来源：`examples/part-05/ch25-advanced/lab06-100-cycles/`包含`run.sh`，它执行100次kldload / sleep / kldunload循环并记录内核之前和之后的内存占用。

步骤1. 记录内核的初始内存占用：

```console
# vmstat -m | awk '$1=="Solaris" || $1=="kernel"' > /tmp/before.txt
# cat /tmp/before.txt
```

步骤2. 运行循环脚本：

```console
# ./lab06-100-cycles/run.sh
cycle 1/100: ok
cycle 2/100: ok
...
cycle 100/100: ok
done: 100个循环，0次失败，0个检测到的泄漏。
```

步骤3. 记录最终内存占用：

```console
# vmstat -m | awk '$1=="Solaris" || $1=="kernel"' > /tmp/after.txt
# diff /tmp/before.txt /tmp/after.txt
```

预期：没有显著差异。如果差异超过几KB（内核自己的簿记会波动），驱动程序有泄漏。

步骤4. 如果脚本报告任何失败，检查`run.sh`填充的`/tmp/myfirst-cycles.log`以找到第一个失败的循环。失败通常在注销步骤：缺少`EVENTHANDLER_DEREGISTER`或缺少`mtx_destroy`。

干净的100次循环运行是对驱动程序生命周期规范建立信心的最简单方式之一。在attach或detach链的每次实质性更改后重复它。

### 实验7：用户空间中的能力发现

目的：确认用户空间程序可以在运行时发现驱动程序的能力并相应地行为，如第4节设计的那样。

来源：`examples/part-05/ch25-advanced/lab07-getcaps/`包含`mfctl25.c`，`myfirstctl`的更新版本，在每次操作前发出`MYFIRSTIOC_GETCAPS`并跳过不支持的操作。

步骤1. 构建`mfctl25`：

```console
# make -C lab07-getcaps
```

步骤2. 针对标准第25章驱动程序运行并观察能力报告：

```console
# ./lab07-getcaps/mfctl25 caps
Driver reports capabilities:
  MYF_CAP_RESET
  MYF_CAP_GETMSG
  MYF_CAP_SETMSG
```

驱动程序报告三个能力。`MYF_CAP_TIMEOUT`位已定义但未设置，因为超时行为是sysctl，不是ioctl。

步骤3. 运行每个操作并确认程序仅尝试支持的操作：

```console
# ./lab07-getcaps/mfctl25 reset
# ./lab07-getcaps/mfctl25 getmsg
Current message: Hello from myfirst
# ./lab07-getcaps/mfctl25 setmsg "new message"
# ./lab07-getcaps/mfctl25 timeout
Timeout ioctl不支持；请改用sysctl dev.myfirst.0.timeout_sec。
```

最后一行是能力检查触发：程序请求`MYF_CAP_TIMEOUT`，驱动程序没有广告它，程序打印了有用的消息而不是发出会返回`ENOTTY`的ioctl。

步骤4. 加载旧构建（`lab07-getcaps/ch24/`中的第24章`myfirst.ko`）并重新运行：

```console
# kldunload myfirst
# kldload lab07-getcaps/ch24/myfirst.ko
# ./lab07-getcaps/mfctl25 caps
GETCAPS ioctl不支持。回退到默认功能集：
  MYF_CAP_RESET
  MYF_CAP_GETMSG
  MYF_CAP_SETMSG
```

当`GETCAPS`本身返回`ENOTTY`时，程序回退到匹配第24章已知行为的安全默认集。这是前向兼容性模式的实际运行。

步骤5. 重新加载第25章驱动程序以恢复测试状态：

```console
# kldunload myfirst
# kldload ./myfirst.ko
```

实验演示了能力发现让一个用户空间程序在两个驱动程序版本间正确工作，这正是该模式的全部要点。

### 实验8：Sysctl范围验证

目的：确认驱动程序公开的每个可写sysctl拒绝超出范围的值，并在拒绝时保持内部状态不变。

来源：`examples/part-05/ch25-advanced/lab08-sysctl-validation/`包含构建了范围检查的驱动程序和一个测试脚本`run.sh`，该脚本将每个sysctl驱动到其极限。

步骤1. 加载驱动程序并列出其可写sysctl：

```console
# kldload ./myfirst.ko
# sysctl -W dev.myfirst.0 | grep -v "^dev.myfirst.0.debug.classes"
dev.myfirst.0.timeout_sec: 5
dev.myfirst.0.max_retries: 3
dev.myfirst.0.log_ratelimit_pps: 10
dev.myfirst.0.debug.mask: 0
```

四个可写sysctl。每个都有特定的有效范围。

步骤2. 尝试将每个sysctl设置为零、其最大允许值和超过最大值一个的值：

```console
# sysctl dev.myfirst.0.timeout_sec=0
sysctl: dev.myfirst.0.timeout_sec: Invalid argument
# sysctl dev.myfirst.0.timeout_sec=60
dev.myfirst.0.timeout_sec: 5 -> 60
# sysctl dev.myfirst.0.timeout_sec=61
sysctl: dev.myfirst.0.timeout_sec: Invalid argument
# sysctl dev.myfirst.0.timeout_sec
dev.myfirst.0.timeout_sec: 60
```

超出范围的尝试被`EINVAL`拒绝；内部值不变。对60的有效赋值成功。

步骤3. 对其他sysctl重复：

- `max_retries`：有效范围1-100。尝试0、100、101。
- `log_ratelimit_pps`：有效范围1-10000。尝试0、10000、10001。
- `debug.mask`：有效范围0-0xff（定义的位）。尝试0、0xff、0x100。

对于每个，脚本报告通过或失败。通过每个案例的驱动程序有正确的处理程序级别验证。

步骤4. 检查`examples/part-05/ch25-advanced/myfirst_sysctl.c`中的sysctl处理程序并注意模式：

```c
static int
myfirst_sysctl_timeout_sec(SYSCTL_HANDLER_ARGS)
{
	struct myfirst_softc *sc = arg1;
	u_int new_val;
	int error;

	mtx_lock(&sc->sc_mtx);
	new_val = sc->sc_timeout_sec;
	mtx_unlock(&sc->sc_mtx);

	error = sysctl_handle_int(oidp, &new_val, 0, req);
	if (error != 0 || req->newptr == NULL)
		return (error);

	if (new_val < 1 || new_val > 60)
		return (EINVAL);

	mtx_lock(&sc->sc_mtx);
	sc->sc_timeout_sec = new_val;
	mtx_unlock(&sc->sc_mtx);
	return (0);
}
```

注意操作顺序：为读侧复制出当前值，调用`sysctl_handle_int`处理副本，在写入时验证，验证成功后在锁下提交。在验证之前提交的处理程序会将不一致的状态暴露给并发读取者。

步骤5. 确认sysctl描述有用（`sysctl -d`）：

```console
# sysctl -d dev.myfirst.0.timeout_sec
dev.myfirst.0.timeout_sec: 操作超时秒数（范围1-60）
```

描述说明了单位和范围。在未咨询任何文档的情况下读取sysctl的用户仍然可以正确设置它。

### 实验9：跨驱动程序的日志消息审计

目的：清点驱动程序中的每个日志消息并确认每个都遵循第1节和第2节的规范（`device_printf`、包含相关的errno、在热路径时限速）。

来源：`examples/part-05/ch25-advanced/lab09-log-audit/`包含审计脚本`audit.sh`和一个基于grep的检查器。

步骤1. 针对驱动程序源运行审计脚本：

```console
# cd examples/part-05/ch25-advanced
# ./lab09-log-audit/audit.sh
```

脚本grep源树中的每个`printf`、`device_printf`、`log`、`DPRINTF`和`DLOG_RL`调用，并将每个分类为：

- PASS：使用`device_printf`或`DPRINTF`，设备名称是隐式的。
- PASS：在热路径上使用`DLOG_RL`。
- WARN：使用没有设备上下文的`printf`（在`MOD_LOAD`时可能是合法的）。
- FAIL：在热路径上使用没有限速的`device_printf`。

预期输出（对于第25章标准驱动程序）：

```text
myfirst.c:    15个日志消息 - 15个PASS
myfirst_cdev.c:  6个日志消息 - 6个PASS
myfirst_ioctl.c: 4个日志消息 - 4个PASS
myfirst_sysctl.c: 0个日志消息
myfirst_log.c:   2个日志消息 - 2个PASS
总计：27个日志消息 - 0个WARN，0个FAIL
```

步骤2. 故意破坏一个消息（例如，将读回调中的`DPRINTF(sc, MYF_DBG_IO, ...)`改为裸`device_printf(sc->sc_dev, ...)`）并重新运行：

```text
myfirst_cdev.c: 6个日志消息 - 5个PASS，1个FAIL
  myfirst_cdev.c:83: 热路径上的device_printf未被限速
总计：27个日志消息 - 0个WARN，1个FAIL
```

审计捕获了回归。恢复更改并重新运行以确认计数回到零失败。

步骤3. 向非热路径添加新的日志消息（例如，在attach时的一次性初始化消息）。确认审计接受它为PASS：

```c
device_printf(dev, "以超时%u初始化\n",
    sc->sc_timeout_sec);
```

attach时的一次性消息不需要限速，因为它们每个实例每次加载恰好触发一次。

步骤4. 对于审计分类为PASS的每个消息，确认消息包含有意义的上下文。像"error"这样的消息对审计工具是PASS但对人类读者是FAIL。需要对grep输出的第二次手动检查来确认消息实际上有用。

实验演示了两点。首先，机械审计捕获分类规则（热路径上的限速、`device_printf`优于裸`printf`）但不能判断消息质量。其次，人工检查是确认消息包含足够上下文以便诊断的方式。两次检查一起给驱动程序一个真正有助于未来支持工程师的日志表面。

### 实验10：多版本兼容性矩阵

目的：确认第4节引入的能力发现模式实际上允许单个用户空间程序与三个不同版本的驱动程序工作。

来源：`examples/part-05/ch25-advanced/lab10-compat-matrix/`包含三个预构建的`.ko`文件，分别对应驱动程序版本1.6-debug、1.7-integration和1.8-maintenance，加上一个使用`MYFIRSTIOC_GETCAPS`（或回退）来决定尝试哪些操作的单一用户空间程序`mfctl-universal`。

步骤1. 依次加载每个驱动程序版本并对其运行`mfctl-universal --caps`：

```console
# kldload lab10-compat-matrix/v1.6/myfirst.ko
# ./lab10-compat-matrix/mfctl-universal --caps
Driver: 版本1.6-debug
GETCAPS ioctl: 不支持
使用回退能力集：
  MYF_CAP_GETMSG
  MYF_CAP_SETMSG

# kldunload myfirst
# kldload lab10-compat-matrix/v1.7/myfirst.ko
# ./lab10-compat-matrix/mfctl-universal --caps
Driver: 版本1.7-integration
GETCAPS ioctl: 不支持
使用回退能力集：
  MYF_CAP_RESET
  MYF_CAP_GETMSG
  MYF_CAP_SETMSG

# kldunload myfirst
# kldload lab10-compat-matrix/v1.8/myfirst.ko
# ./lab10-compat-matrix/mfctl-universal --caps
Driver: 版本1.8-maintenance
GETCAPS ioctl: 支持
Driver reports capabilities:
  MYF_CAP_RESET
  MYF_CAP_GETMSG
  MYF_CAP_SETMSG
```

三个驱动程序版本，一个用户空间程序，三个不同的能力决策。程序与每个版本一起工作。

步骤2. 依次测试每个能力并确认程序跳过不支持的操作：

```console
# kldunload myfirst
# kldload lab10-compat-matrix/v1.6/myfirst.ko
# ./lab10-compat-matrix/mfctl-universal reset
reset: 此驱动程序版本(1.6-debug)不支持
# ./lab10-compat-matrix/mfctl-universal getmsg
Current message: Hello from myfirst
```

在1.7中添加的重置操作在1.6上被干净地跳过。程序打印了有用的消息而不是发出会返回`ENOTTY`的ioctl。

步骤3. 阅读`mfctl-universal`源代码并注意三层回退：

```c
uint32_t
driver_caps(int fd, const char *version)
{
	uint32_t caps;

	if (ioctl(fd, MYFIRSTIOC_GETCAPS, &caps) == 0)
		return (caps);
	if (errno != ENOTTY)
		err(1, "GETCAPS ioctl");

	/* 按版本字符串回退。 */
	if (strstr(version, "1.8-") != NULL)
		return (MYF_CAP_RESET | MYF_CAP_GETMSG |
		    MYF_CAP_SETMSG);
	if (strstr(version, "1.7-") != NULL)
		return (MYF_CAP_RESET | MYF_CAP_GETMSG |
		    MYF_CAP_SETMSG);
	if (strstr(version, "1.6-") != NULL)
		return (MYF_CAP_GETMSG | MYF_CAP_SETMSG);

	/* 未知版本：使用最小安全集。 */
	return (MYF_CAP_GETMSG);
}
```

第一层直接询问驱动程序。第二层匹配已知版本字符串。第三层回退到每个版本的驱动程序都支持的最小集。

步骤4. 思考当1.9版本发布带有新能力位时会发生什么。程序不需要更新：1.9上的`MYFIRSTIOC_GETCAPS`将报告新位，程序将看到它，如果程序知道对应的操作它将使用它。如果程序不知道该操作，位被忽略。无论哪种方式，程序继续工作。

实验演示了能力发现不是一个抽象模式；它是让一个用户空间程序无需修改即可跨越三个驱动程序版本的具体机制。


## 挑战练习

本节的挑战超越实验。每个都要求你将驱动程序扩展到本章指出但未完成的方向。准备好时逐个完成；它们都不需要本章未涵盖的新内核知识，但每个都需要仔细地遍历现有代码。

### 挑战1：每类别速率限制

第1节在softc中规划了三个限速槽（`sc_rl_generic`、`sc_rl_io`、`sc_rl_intr`），但`DLOG_RL`宏使用单个pps值（`sc_log_pps`）。扩展驱动程序，使每个类别有自己的sysctl可配置pps上限：

- 在softc中沿`sc_log_pps`（它保持为通用上限）添加`sc_log_pps_io`和`sc_log_pps_intr`字段。
- 在`dev.myfirst.<unit>.log.pps_*`下添加匹配的sysctl，在`hw.myfirst.log_pps_*`下添加匹配的可调参数。
- 更新`DLOG_RL_IO`和`DLOG_RL_INTR`助手（或一个同时接受类别和pps值的通用助手）以遵守每类别上限。

编写一个在每类别中触发消息突发的短测试程序，并从`dmesg`确认每个类别独立限速。通用桶不应饿死I/O桶，反之亦然。

提示：最可重用的形状是一个辅助函数`myfirst_log_ratelimited(sc, class, fmt, ...)`，它根据类别位查找正确的限速状态和正确的pps上限。`DLOG_RL_*`宏成为该助手的薄包装。

### 挑战2：可写字符串sysctl

第3节警告了可写字符串sysctl的复杂性。正确实现一个。sysctl应该是`dev.myfirst.<unit>.message`，带有`CTLFLAG_RW`，应该允许操作员用单个`sysctl(8)`调用重写驱动程序内的消息。

要求：

1. 处理程序必须在更新时获取softc互斥锁。
2. 处理程序必须根据`sizeof(sc->sc_msg)`验证长度，并用`EINVAL`拒绝过大的字符串。
3. 处理程序必须使用`sysctl_handle_string`进行复制；不要重新实现用户空间访问。
4. 成功更新后，处理程序必须为`MSG_CHANGED`发出`devctl_notify`，就像ioctl一样。

测试：

```console
# sysctl dev.myfirst.0.message="hello from sysctl"
# sysctl dev.myfirst.0.message
dev.myfirst.0.message: hello from sysctl
# sysctl dev.myfirst.0.message="$(printf 'A%.0s' {1..1000})"
sysctl: dev.myfirst.0.message: Invalid argument
```

第二个`sysctl`应该失败（过大），驱动程序的消息应该不变。

考虑：ioctl和sysctl应该发出相同的`MSG_CHANGED`事件，还是不同的？两侧都在更新相同的底层状态；单个事件类型可能是正确的。在`MAINTENANCE.md`中文档你的决定。

### 挑战3：独立于Detach的`MOD_QUIESCE`处理程序

第7节指出`MOD_QUIESCE`和`MOD_UNLOAD`在概念上是不同的，但`myfirst`通过`myfirst_detach`处理两者。拆分它们，以便quiesce问题可以没有副作用地回答。

要求：

1. 向模块事件处理程序添加显式的`MOD_QUIESCE`检查。如果有设备打开，处理程序返回`EBUSY`，否则返回`0`。
2. 处理程序不调用`destroy_dev`、不销毁锁、不改变状态。它只读取`sc_open_count`。
3. 对于每个附加的实例，通过`devclass`迭代并检查每个softc。使用`DRIVER_MODULE`导出的`myfirst_devclass`符号。

提示：查看`/usr/src/sys/kern/subr_bus.c`中的`devclass_get_softc`和相关助手。它们是从没有`device_t`的模块级函数枚举softc的方式。

测试：打开`/dev/myfirst0`，尝试`kldunload myfirst`，确认它报告"module busy"并且驱动程序不变。关闭fd，重试卸载，确认它成功。

### 挑战4：基于排水的Detach替代`EBUSY`

本章的detach模式在驱动程序正在使用时拒绝卸载。更精心设计的模式是排空进行中的引用而不是拒绝。实现它。

要求：

1. 向softc添加`is_dying`布尔值，由`sc_mtx`保护。
2. 在`myfirst_open`中，在锁下检查`is_dying`，如果为真则返回`ENXIO`。
3. 在`myfirst_detach`中，在锁下设置`is_dying`。使用`mtx_sleep`配合条件变量或简单的带超时轮询循环等待`sc_open_count`达到零。
4. 在`sc_open_count`达到零后，继续执行destroy_dev和detach链的其余部分。

添加超时：如果`sc_open_count`在（比如说）30秒内未达到零，从detach返回`EBUSY`。操作员获得清晰的信号，驱动程序未在排空；他们可以杀死有问题的进程并重试。

测试：从一个shell循环打开`/dev/myfirst0`，从另一个shell调用`kldunload myfirst`，观察排空行为。

### 挑战5：Sysctl驱动的版本提升检查

编写一个小的用户空间程序，读取`dev.myfirst.<unit>.version`，解析版本字符串，并将其与程序要求的最低版本进行比较。如果驱动程序足够新，程序应打印"ok"，如果不够新，则打印"driver too old, please update"。

要求：

1. 将字符串`X.Y-tag`解析为整数。用清晰的错误拒绝格式错误的字符串。
2. 与最低版本`"1.8"`比较。报告`"1.7-integration"`的驱动程序应未通过检查；报告`"1.8-maintenance"`的驱动程序应通过；报告`"2.0-something"`的驱动程序应通过。
3. 成功时退出状态为`0`，失败时为非零，以便检查可以在shell脚本中使用。

思考：一个设计良好的程序能否依赖版本字符串进行兼容性检查，还是第4节的能力位掩码是更好的信号？没有单一的正确答案；练习是思考权衡。

### 挑战6：添加枚举打开文件描述符的Sysctl

添加一个新sysctl `dev.myfirst.<unit>.open_fds`，作为字符串返回当前打开了设备的进程的PID。这比听起来更难：驱动程序通常不跟踪哪个进程打开了每个fd。

提示：在`myfirst_open`中，将调用线程的PID存储在softc下的链表中。在`myfirst_close`中，移除相应的条目。在sysctl处理程序中，在softc互斥锁下遍历链表并构建PID的逗号分隔字符串。

边缘情况：

1. 多次打开的进程（多个fd、fork的子进程）应该出现一次还是多次？决定并文档化。
2. 列表必须在长度上有界（攻击者可能打开设备数百万次）。
3. sysctl值是只读的；处理程序不得修改列表。

思考：这些信息实际上有用吗，还是`fstat(1)`是做同样工作的更好工具？答案取决于驱动程序能否提供用户空间工具无法自行推导的信息。

### 挑战7：为`vm_lowmem`添加第二个`EVENTHANDLER`

`myfirst`今天没有缓存，但想象它有：一个用于读/写操作的预分配4 KB缓冲区池。在低内存下，驱动程序应该释放一些缓冲区回去。

实现一个合成缓存：在attach时分配64个`malloc(M_TEMP, 4096)`指针的数组。注册一个`vm_lowmem`处理程序，当触发时释放缓存的一半。重新附加重新分配它们。

要求：

1. 缓存分配在softc互斥锁下进行。
2. `vm_lowmem`处理程序获取互斥锁，扫描数组，`free()`前32个缓冲区。
3. 一个sysctl `dev.myfirst.<unit>.cache_free`报告当前空闲（NULL）槽的数量；操作员可以确认处理程序已触发。

测试：使用`stress -m 10 --vm-bytes 512M`循环将系统驱动到低内存压力，并观察`cache_free` sysctl。随着`vm_lowmem`反复触发，它应该随时间增长。

思考：这是事件旨在用于的吗？许多注册`vm_lowmem`的驱动程序的缓存远大于64个缓冲区；成本/效益不同。这是一个教学练习；真实驱动程序会更仔细地思考其缓存是否值得复杂性。

### 挑战8：返回结构化有效载荷的`MYFIRSTIOC_GETSTATS` Ioctl

到目前为止，驱动程序处理的每个ioctl都返回一个标量：一个整数、一个uint32或一个固定大小的字符串。添加一个`MYFIRSTIOC_GETSTATS` ioctl，返回包含驱动程序维护的每个计数器的结构化有效载荷。

要求：

1. 在`myfirst_ioctl.h`中定义`struct myfirst_stats`，包含`open_count`、`total_reads`、`total_writes`、`log_drops`（你添加的新计数器）和`last_error_errno`（另一个新计数器）的字段。
2. 添加`MYFIRSTIOC_GETSTATS`，命令号为6，声明为`_IOR('M', 6, struct myfirst_stats)`。
3. 处理程序在`sc_mtx`下将softc的计数器复制到有效载荷中并返回。
4. 在`GETCAPS`响应中广告新能力位`MYF_CAP_STATS`。
5. 更新`MAINTENANCE.md`以文档化新ioctl和新能力。

边缘情况：

1. 如果结构体大小以后改变会怎样？`_IOR`宏将大小烘焙到命令号中。添加字段提升命令号，这会破坏旧调用者。修复是从第一天起在结构体中包含`version`和`reserved`空间；任何未来添加重用保留空间。

2. 原子地返回所有计数器是否安全，还是它们需要单独的锁定？在复制期间持有`sc_mtx`是最简单的规范。

思考：这是ioctl设计开始感觉复杂的地方。对于简单的计数器快照，带有字符串格式输出的sysctl可能比带版本化结构体的ioctl更容易。你会选择哪个，为什么？

### 挑战9：基于Devctl的实时监控

添加第二个`devctl_notify`事件，每当限速桶丢弃消息时触发。事件应包含类别名和当前桶状态作为key=value数据。

要求：

1. 当`ppsratecheck`返回零（消息被丢弃）时，增加每类别丢弃计数器并发出一个`devctl_notify`，`system="myfirst"`，`type="LOG_DROPPED"`，`data="class=io drops=42"`。
2. devctl事件本身必须被限速；否则报告丢弃的行为成为另一次洪水。使用带有慢上限（例如1 pps）的第二个`ppsratecheck`来限制devctl发出。
3. 编写一个匹配事件并在每次触发时记录摘要的devd规则。

测试：运行实验1的洪水程序并确认`devctl`发出丢弃报告而不会自己洪水。

## 故障排除指南

当本章的机制表现异常时，症状通常是间接的：一条静默缺失的日志行、一个因错误原因拒绝加载的驱动程序、一个不出现的sysctl、一个不调用你处理程序的重新启动。本参考将常见症状映射到最可能负责的机制，以及驱动程序源中首先查看的位置。

### 症状：`kldload`返回"Exec format error"

模块是针对与运行内核不匹配的内核ABI构建的。典型原因是运行内核版本与编译时使用的`SYSDIR`之间的不匹配。

检查：`uname -r`和Makefile中`SYSDIR`的值。如果内核是14.3-RELEASE但构建从更新的15.0-CURRENT树获取了头文件，ABI是不同的。

修复：将`SYSDIR`指向与运行内核匹配的源树。第25章Makefile默认使用`/usr/src/sys`；在带有匹配`/usr/src`的14.3系统上，这是正确的。

### 症状：`kldload`对一个明显存在的文件返回"No such file or directory"

文件存在但内核的模块加载器无法解析它。常见原因：文件是来自不同机器的陈旧构建产物，或文件已损坏。

检查：`file myfirst.ko`应报告它是ELF 64-bit LSB shared object。如果它报告其他任何内容，从源重新构建。

### 症状：`kldload`成功但`kldstat`不显示模块

加载器决定自动卸载模块。当`MOD_LOAD`返回零但`DRIVER_MODULE`的`device_identify`没有找到任何设备时会发生这种情况。对于使用`nexus`作为父级的`myfirst`，这不应该发生；伪驱动程序总是能找到`nexus`。

检查：`dmesg | tail -20`查找像`module "myfirst" failed to register`的行。该消息指向出了什么问题。

### 症状：`kldload`报告"module busy"

驱动程序的先前实例仍然加载，并且在某处有一个打开的文件描述符。旧实例中的`MOD_QUIESCE`路径返回`EBUSY`。

检查：`fstat | grep myfirst`应显示持有fd的进程。杀死进程或关闭fd，然后重试`kldunload`。

### 症状：`sysctl dev.myfirst.0.debug.mask=0x4`返回"Operation not permitted"

调用者不是root。带有`CTLFLAG_RW`的sysctl通常需要root权限，除非显式标记为其他。

检查：你以root运行吗？先`sudo sysctl ...`或`su -`。

### 症状：新sysctl不出现在树中

`SYSCTL_ADD_*`要么没有被调用，要么被调用时使用了错误的上下文/树指针。最常见的错误是对每设备OID使用`SYSCTL_STATIC_CHILDREN`而不是`device_get_sysctl_tree`。

检查：在`myfirst_sysctl_attach`内部，确认使用了`ctx = device_get_sysctl_ctx(dev)`和`tree = device_get_sysctl_tree(dev)`，并且每个`SYSCTL_ADD_*`调用都将`ctx`作为第一个参数传递。

### 症状：可调参数似乎被忽略

`TUNABLE_*_FETCH`在attach时运行，但仅当可调参数在该时刻处于内核环境中时。常见错误是（a）在模块加载后设置可调参数，（b）名称输入错误，（c）忘记`kenv`不是持久的。

检查：
- 重新加载模块前`kenv hw.myfirst.timeout_sec`。值应该是你期望的。
- 传递给`TUNABLE_INT_FETCH`的字符串必须与`kenv`完全匹配。一侧的拼写错误是静默的。
- 通过`/boot/loader.conf`设置需要重新启动（或`kldunload`后跟`kldload`，它为特定模块的可调参数重新读取`loader.conf`）。

### 症状：日志消息应该触发但不出现在`dmesg`中

三个常见原因：

1. 调试类别位未设置。检查`sysctl dev.myfirst.0.debug.mask`；类别位必须启用。
2. 限速桶为空。如果消息通过`DLOG_RL`发出，前几个触发，其余被静默抑制。通过sysctl设置更高的pps上限或等待一秒让桶重新填充。
3. 消息已发出但被系统的`sysctl kern.msgbuf_show_timestamp`设置或`dmesg`缓冲区大小（`sysctl kern.msgbuf_size`）过滤。

检查：`dmesg -c > /dev/null`清除缓冲区，重现操作，重新读取缓冲区。清空的缓冲区应该只包含驱动程序的输出。

### 症状：日志消息出现一次后永远静默

限速桶永久拒绝消息。如果`rl_curpps`变得非常大而pps上限非常低，就会发生这种情况。检查`ppsratecheck`是否用稳定的`struct timeval`和稳定的`int *`（都是softc的成员）被调用；每次调用的栈变量会在每次调用时重置为零，算法会每次触发。

检查：限速状态必须在softc或其他持久位置，不能在局部变量中。

### 症状：Attach失败且清理链不释放资源

标签化goto缺少步骤或有短路链的孤立`return`。在每个标签的顶部添加`device_printf(dev, "到达标签fail_X\n")`并重新运行失败注入实验。不打印的标签是被跳过的标签。

常见原因：为了调试而插入的中间`return (error)`从未被移除。编译器不警告，因为链在语法上仍然有效；行为是错误的。

### 症状：Detach以"witness"警告panic

锁在被持有时被销毁，或锁在其所有者被销毁后被获取。witness子系统捕捉两者。回溯指向锁名，它映射到softc字段。

检查：detach链应该是attach的完全反向。常见错误是在`destroy_dev(sc->sc_cdev)`之前调用`mtx_destroy(&sc->sc_mtx)`：cdev的回调可能仍在运行，它们尝试获取锁，锁已经消失。修复：先销毁cdev，然后销毁锁。

### 症状：驱动程序在模块卸载时以悬空指针panic

`EVENTHANDLER_DEREGISTER`未被调用，内核触发了事件，回调指针指向已释放的内存。

检查：对于attach中的每个`EVENTHANDLER_REGISTER`，在detach中搜索`EVENTHANDLER_DEREGISTER`。计数必须匹配。如果计数匹配但panic仍然发生，softc中存储的标签已损坏；审计注册和注销之间的代码路径以查找内存篡改。

### 症状：`MYFIRSTIOC_GETVER`返回意外值

`myfirst_ioctl.h`中的ioctl版本整数与`myfirst_ioctl.c`写入缓冲区的不匹配。当头文件更新但处理程序仍然返回硬编码常量时会发生这种情况。

检查：处理程序应该写入`MYFIRST_IOCTL_VERSION`（头文件中的常量），而不是字面整数。

### 症状：`devctl_notify`事件从未出现在`devd.log`中

`devd(8)`未运行，或其配置与事件不匹配。

检查：
- `service devd status`确认守护进程正在运行。
- `grep myfirst /etc/devd/*.conf`应该找到规则。
- 前台`devd -Df`在事件到达时打印每个事件；重现操作并观看输出。

### 症状：100次循环回归脚本增长内核的内存占用

资源在每次加载/卸载循环中泄漏。常见罪魁祸首：没有匹配`free`的`malloc`、没有匹配注销的`EVENTHANDLER_REGISTER`、驱动程序手动调用但未在detach时调用`sysctl_ctx_free`的`sysctl_ctx_init`（第25章使用`device_get_sysctl_ctx`，由Newbus管理；分配自己上下文的驱动程序必须释放它）。

检查：`vmstat -m | grep myfirst`在之前和之后以查看驱动程序自己的内存消耗，以及`vmstat -m | grep solaris`以查看驱动程序可能间接分配的内核级结构。

### 症状：两个并发的`MYFIRSTIOC_SETMSG`调用交错写入

softc互斥锁未被持有在更新周围。两个线程同时写入`sc->sc_msg`，产生损坏的结果。

检查：ioctl处理程序中对`sc->sc_msg`和`sc->sc_msglen`的每个访问必须在`mtx_lock(&sc->sc_mtx) ... mtx_unlock(&sc->sc_mtx)`内。

### 症状：sysctl值在每次模块加载时重置

这是预期行为，不是错误。attach时默认值是可调参数评估的值，要么是`TUNABLE_INT_FETCH`默认值，要么是通过`kenv`设置的值。运行时sysctl写入在卸载时丢失。如果你想让值持久化，通过`kenv`或`/boot/loader.conf`设置它。

### 症状：`MYFIRSTIOC_GETCAPS`返回的值没有你刚添加的位

`myfirst_ioctl.c`文件已更新但未重新编译，或加载了错误的构建。还要检查switch语句中的处理程序使用`|=`操作符或包含每个位的单个赋值。

检查：从示例目录执行`make clean && make`。`kldstat -v | grep myfirst`确认加载模块的路径与你构建的匹配。

### 症状：SYSINIT在内核的分配器准备好之前触发

SYSINIT注册在过早的子系统ID。许多子系统（可调参数、锁、早期initcall）不允许使用`M_NOWAIT`调用`malloc`，更不用说`M_WAITOK`。如果你的回调调用`malloc`而内核在引导时panic，检查子系统ID。

检查：`SYSINIT(...)`中的子系统ID。对于分配内存的回调，使用`SI_SUB_DRIVERS`或更晚；不要使用`SI_SUB_TUNABLES`或更早。

### 症状：用`EVENTHANDLER_PRI_FIRST`注册的处理程序仍然运行得很晚

`EVENTHANDLER_PRI_FIRST`不是硬保证；它是排序队列中的优先级。如果另一个处理程序也用`EVENTHANDLER_PRI_FIRST`注册，它们之间的顺序是未定义的。文档化的优先级是粗粒度的；不支持细粒度排序。

检查：接受优先级是提示，不是契约。如果驱动程序绝对需要在特定其他处理程序之前或之后运行，设计是错误的；重构驱动程序使顺序不重要。

### 症状：`dmesg`不显示驱动程序的任何输出

驱动程序正在使用`printf`（libc风格的那个）而不是`device_printf`或`DPRINTF`。内核`printf`仍然工作但不承载设备名称，这使得消息难以过滤。

检查：驱动程序中的每条消息应该通过`device_printf(dev, ...)`或`DPRINTF(sc, class, fmt, ...)`。裸`printf`通常是错误。


## 快速参考

快速参考是第25章引入的宏、标志和函数的单页摘要。在材料熟悉后在键盘上使用它。

### 限速日志记录

```c
struct myfirst_ratelimit {
	struct timeval rl_lasttime;
	int            rl_curpps;
};

#define DLOG_RL(sc, rlp, pps, fmt, ...) do {                            \
	if (ppsratecheck(&(rlp)->rl_lasttime, &(rlp)->rl_curpps, pps)) \
		device_printf((sc)->sc_dev, fmt, ##__VA_ARGS__);        \
} while (0)
```

对可以在循环中触发的任何消息使用`DLOG_RL`。将`struct myfirst_ratelimit`放在softc中（不是栈上）。

### Errno词汇表

| Errno | 值 | 用途 |
|-------|-----|-----|
| `0` | 0 | 成功 |
| `EPERM` | 1 | 操作不允许（仅限root） |
| `ENOENT` | 2 | 没有此文件 |
| `EBADF` | 9 | 错误的文件描述符 |
| `ENOMEM` | 12 | 无法分配内存 |
| `EACCES` | 13 | 权限拒绝 |
| `EFAULT` | 14 | 错误的地址（用户指针） |
| `EBUSY` | 16 | 资源忙 |
| `ENODEV` | 19 | 没有此设备 |
| `EINVAL` | 22 | 无效参数 |
| `ENOTTY` | 25 | 设备不适合的ioctl |
| `ENOTSUP` / `EOPNOTSUPP` | 45 | 操作不支持 |
| `ENOIOCTL` | -3 | 此驱动程序未处理的ioctl（内部；内核映射为`ENOTTY`） |

### 可调参数系列

```c
TUNABLE_INT_FETCH("hw.myfirst.name",    &sc->sc_int_var);
TUNABLE_LONG_FETCH("hw.myfirst.name",   &sc->sc_long_var);
TUNABLE_BOOL_FETCH("hw.myfirst.name",   &sc->sc_bool_var);
TUNABLE_STR_FETCH("hw.myfirst.name",     sc->sc_str_var,
                                          sizeof(sc->sc_str_var));
```

在填充默认值后在attach中调用每个fetch一次。fetch仅在可调参数存在时更新变量。

### Sysctl标志摘要

| 标志 | 含义 |
|------|---------|
| `CTLFLAG_RD` | 只读 |
| `CTLFLAG_RW` | 读写 |
| `CTLFLAG_TUN` | 在attach时与加载器可调参数协作 |
| `CTLFLAG_RDTUN` | 只读 + 可调参数的简写 |
| `CTLFLAG_RWTUN` | 读写 + 可调参数的简写 |
| `CTLFLAG_MPSAFE` | 处理程序是MPSAFE的 |
| `CTLFLAG_SKIP` | 从默认`sysctl(8)`列表中隐藏OID |

### 版本标识符

- `MYFIRST_VERSION`：人类可读的发布字符串，例如`"1.8-maintenance"`。
- `MODULE_VERSION(myfirst, N)`：`MODULE_DEPEND`使用的整数。
- `MYFIRST_IOCTL_VERSION`：由`MYFIRSTIOC_GETVER`返回的整数；仅在线格式破坏时提升。

### 能力位

```c
#define MYF_CAP_RESET    (1U << 0)
#define MYF_CAP_GETMSG   (1U << 1)
#define MYF_CAP_SETMSG   (1U << 2)
#define MYF_CAP_TIMEOUT  (1U << 3)

#define MYFIRSTIOC_GETCAPS  _IOR('M', 5, uint32_t)
```

### 标签清理骨架

```c
static int
myfirst_attach(device_t dev)
{
	struct myfirst_softc *sc = device_get_softc(dev);
	int error;

	/* 按顺序获取资源 */
	mtx_init(&sc->sc_mtx, "myfirst", NULL, MTX_DEF);

	error = make_dev_s(...);
	if (error != 0)
		goto fail_mtx;

	myfirst_sysctl_attach(sc);

	error = myfirst_log_attach(sc);
	if (error != 0)
		goto fail_cdev;

	sc->sc_shutdown_tag = EVENTHANDLER_REGISTER(shutdown_pre_sync,
	    myfirst_shutdown, sc, SHUTDOWN_PRI_DEFAULT);
	if (sc->sc_shutdown_tag == NULL) {
		error = ENOMEM;
		goto fail_log;
	}

	return (0);

fail_log:
	myfirst_log_detach(sc);
fail_cdev:
	destroy_dev(sc->sc_cdev);
fail_mtx:
	mtx_destroy(&sc->sc_mtx);
	return (error);
}
```

### 模块化驱动程序的文件布局

```text
driver.h           公共类型
driver.c           模块粘合、cdevsw、attach/detach
driver_cdev.c      open/close/read/write
driver_ioctl.h     ioctl命令号
driver_ioctl.c     ioctl分发
driver_sysctl.c    sysctl树
driver_debug.h     DPRINTF宏
driver_log.h       限速结构
driver_log.c       限速助手
```

### 生产检查表

```text
[  ] MODULE_DEPEND为每个真实依赖声明。
[  ] MODULE_PNP_INFO如果驱动程序绑定到硬件则声明。
[  ] MOD_QUIESCE没有副作用地回答"你可以卸载吗？"。
[  ] devctl_notify为操作员相关事件发出。
[  ] MAINTENANCE.md当前。
[  ] devd.conf片段包含。
[  ] 每个日志消息是device_printf，包含errno，
     如果可以在循环中触发则被限速。
[  ] attach/detach存活100次加载/卸载循环。
[  ] sysctl拒绝超出范围的值。
[  ] ioctl有效载荷被边界检查。
[  ] 通过故意注入测试失败路径。
[  ] 版本控制规范：三个独立版本
     标识符，各自因自己的原因提升。
```

### SYSINIT子系统ID

| 常量 | 值 | 用途 |
|----------|-------|-----|
| `SI_SUB_TUNABLES` | 0x0700000 | 确立可调参数值 |
| `SI_SUB_KLD` | 0x2000000 | KLD和模块设置 |
| `SI_SUB_SMP` | 0x2900000 | 启动AP |
| `SI_SUB_DRIVERS` | 0x3100000 | 让驱动程序初始化 |
| `SI_SUB_CONFIGURE` | 0x3800000 | 配置设备 |

在子系统内：
- `SI_ORDER_FIRST` = 0x0
- `SI_ORDER_SECOND` = 0x1
- `SI_ORDER_MIDDLE` = 0x1000000
- `SI_ORDER_ANY` = 0xfffffff

### 关闭事件优先级

- `SHUTDOWN_PRI_FIRST`：早期运行。
- `SHUTDOWN_PRI_DEFAULT`：默认。
- `SHUTDOWN_PRI_LAST`：晚期运行。

### EVENTHANDLER骨架

```c
sc->sc_tag = EVENTHANDLER_REGISTER(shutdown_pre_sync,
    my_handler, sc, SHUTDOWN_PRI_DEFAULT);
/* ... 在detach中 ... */
EVENTHANDLER_DEREGISTER(shutdown_pre_sync, sc->sc_tag);
```

### 可调参数名称阶梯

可调参数和sysctl遵循层次命名约定。下表列出了本章引入的节点：

| 名称 | 种类 | 用途 |
|------|------|------|
| `hw.myfirst.debug_mask_default` | 可调参数 | 每个实例的初始调试掩码 |
| `hw.myfirst.timeout_sec` | 可调参数 | 初始操作超时秒数 |
| `hw.myfirst.max_retries` | 可调参数 | 初始重试计数 |
| `hw.myfirst.log_ratelimit_pps` | 可调参数 | 初始每秒消息上限 |
| `dev.myfirst.<unit>.version` | sysctl (RD) | 发布字符串 |
| `dev.myfirst.<unit>.open_count` | sysctl (RD) | 活跃fd计数 |
| `dev.myfirst.<unit>.total_reads` | sysctl (RD) | 生命周期读调用 |
| `dev.myfirst.<unit>.total_writes` | sysctl (RD) | 生命周期写调用 |
| `dev.myfirst.<unit>.message` | sysctl (RD) | 当前缓冲区内容 |
| `dev.myfirst.<unit>.message_len` | sysctl (RD) | 当前缓冲区长度 |
| `dev.myfirst.<unit>.timeout_sec` | sysctl (RWTUN) | 运行时超时 |
| `dev.myfirst.<unit>.max_retries` | sysctl (RWTUN) | 运行时重试计数 |
| `dev.myfirst.<unit>.log_ratelimit_pps` | sysctl (RWTUN) | 运行时pps上限 |
| `dev.myfirst.<unit>.debug.mask` | sysctl (RWTUN) | 运行时调试掩码 |
| `dev.myfirst.<unit>.debug.classes` | sysctl (RD) | 类别名和位值 |

将表格作为接口契约阅读。`hw.myfirst.*`系列在引导时设置；`dev.myfirst.*`系列在运行时调整。每个可写条目都有一个匹配的只读对应项，操作员可以用它确认当前值。

### Ioctl命令阶梯

第25章的ioctl头文件在魔术`'M'`下定义了这些命令：

| 命令 | 号码 | 方向 | 用途 |
|---------|--------|-----------|------|
| `MYFIRSTIOC_GETVER` | 0 | 读取 | 返回`MYFIRST_IOCTL_VERSION` |
| `MYFIRSTIOC_RESET` | 1 | 无数据 | 清零读/写计数器 |
| `MYFIRSTIOC_GETMSG` | 2 | 读取 | 复制当前消息出来 |
| `MYFIRSTIOC_SETMSG` | 3 | 写入 | 复制新消息进去 |
| （已退役） | 4 | n/a | 保留；不要重用 |
| `MYFIRSTIOC_GETCAPS` | 5 | 读取 | 能力位掩码 |

添加新命令意味着选择下一个未使用的号码。退役命令意味着在阶梯中保留其号码并在旁边标注`(已退役)`，不要重用号码。

## 真实世界驱动程序演练

到目前为止，本章在`myfirst`伪驱动程序上建立了其规范。本节将镜头转过来，看看相同的规范如何出现在FreeBSD 14.3中附带的驱动程序中。每个演练从`/usr/src`中的真实源文件开始，命名正在起作用的第25章模式，并指向模式可见的行。目的不是文档化特定驱动程序（它们自己的文档做这件事），而是展示第25章的习惯不是发明的：它们是工作中的树内驱动程序已经在实践的习惯。

用模式词汇阅读真实驱动程序是加速你自己判断的最快方式。一旦你在看到`ppsratecheck`时就能识别它，每个使用它的驱动程序就变得更快地阅读。

### `mmcsd(4)`：热路径上的限速错误日志

`/usr/src/sys/dev/mmc/mmcsd.c`的`mmcsd`驱动程序服务于MMC和SD卡存储。其上方的文件系统产生连续的块I/O流，每个失败的底层MMC请求产生一个潜在的错误日志行。没有限速，缓慢或不稳定的卡会在几秒钟内淹没`dmesg`。

驱动程序按softc声明其限速状态，如本章推荐：

```c
struct mmcsd_softc {
	...
	struct timeval log_time;
	int            log_count;
	...
};
```

`log_time`和`log_count`是`ppsratecheck`状态。每个在热路径上发出日志消息的地方都以相同方式包装`device_printf`：

```c
#define LOG_PPS  5 /* 每秒最多记录5个错误。 */

...

if (req.cmd->error != MMC_ERR_NONE) {
	if (ppsratecheck(&sc->log_time, &sc->log_count, LOG_PPS))
		device_printf(dev, "Error indicated: %d %s\n",
		    req.cmd->error,
		    mmcsd_errmsg(req.cmd->error));
	...
}
```

该模式恰好是本章引入的`DLOG_RL`形状，宏在原地展开。`LOG_PPS`被设置为每秒5条消息，状态存在于softc中，因此对热路径的重复调用共享同一个桶。

三个观察值得带走。首先，这个模式不是理论上的：一个正在发布的FreeBSD驱动程序在可以每秒触发数千次的热路径上使用它。其次，宏与内联的选择是品味问题；`mmcsd.c`直接编码调用，模式同样可读。第三，`LOG_PPS`常量是保守的（每秒5条）；作者偏好更少的消息而不是更多的。驱动程序作者可以调整pps上限以匹配预期的错误率和操作员的容忍度。

### `uftdi(4)`：模块依赖和PNP元数据

`/usr/src/sys/dev/usb/serial/uftdi.c`的`uftdi`驱动程序附加到基于FTDI芯片的USB串行适配器。它是依赖另一个内核模块的驱动程序的教科书示例：没有USB栈它无法工作。

在文件底部附近：

```c
MODULE_DEPEND(uftdi, ucom, 1, 1, 1);
MODULE_DEPEND(uftdi, usb, 1, 1, 1);
MODULE_VERSION(uftdi, 1);
```

声明了两个依赖。第一个是对`ucom`的，即`uftdi`构建在其上的通用USB串行框架。第二个是对`usb`，USB核心。两者都限定为恰好版本1。在没有`usb`或`ucom`的内核上加载`uftdi.ko`以清晰的错误失败；在子系统版本已提升超过1的内核上加载也失败，直到`uftdi`自己的声明被更新。

PNP元数据通过扩展为`MODULE_PNP_INFO`的宏发布：

```c
USB_PNP_HOST_INFO(uftdi_devs);
```

`USB_PNP_HOST_INFO`是在`/usr/src/sys/dev/usb/usbdi.h`中定义的USB特定助手。它以USB供应商/产品元组的正确格式字符串扩展为`MODULE_PNP_INFO`。`uftdi_devs`是`struct usb_device_id`条目的静态数组，驱动程序处理的每个（供应商、产品、接口）三元组一个。

这是第25章第7节生产准备就绪模式应用于真实硬件驱动程序：依赖已声明、元数据已发布、版本整数已存在。新USB串行适配器出现在系统上会导致`devd(8)`参考PNP元数据，识别`uftdi`作为驱动程序，并在尚未加载时加载它。一旦元数据正确，机制完全自动化。

第26章版本的`myfirst`将使用相同的模式和相同的助手。

### `iscsi(4)`：在`attach`中注册的关闭处理程序

`/usr/src/sys/dev/iscsi/iscsi.c`的iSCSI发起器保持与远程存储目标的开放连接。当系统关闭时，发起器必须在网络层被拆除之前优雅地关闭这些连接；否则远程端留下陈旧的会话。

关闭处理程序在attach时注册：

```c
sc->sc_shutdown_pre_eh = EVENTHANDLER_REGISTER(shutdown_pre_sync,
    iscsi_shutdown_pre, sc, SHUTDOWN_PRI_FIRST);
```

两个细节很重要。首先，注册的标签存储在softc（`sc->sc_shutdown_pre_eh`）中，因此后续注销可以引用它。其次，优先级是`SHUTDOWN_PRI_FIRST`，不是`SHUTDOWN_PRI_DEFAULT`：iSCSI驱动程序要在其他人开始关闭工作之前关闭连接，因为存储连接需要时间来干净地关闭。

注销发生在detach路径中：

```c
EVENTHANDLER_DEREGISTER(shutdown_pre_sync, sc->sc_shutdown_pre_eh);
```

一次注册，一次注销。softc中的标签保持它们绑定。

对于`myfirst`，本章的演示使用`SHUTDOWN_PRI_DEFAULT`，因为伪驱动程序没有好的理由早期运行。真实驱动程序根据什么依赖于什么来选择优先级：必须在其他驱动程序之前安静的驱动程序选择`SHUTDOWN_PRI_FIRST`；依赖于文件系统完整的驱动程序选择`SHUTDOWN_PRI_LAST`。优先级是一个设计决策，`iscsi`展示了一种做出它的方式。

### `ufs_dirhash`：`vm_lowmem`上的缓存驱逐

`/usr/src/sys/ufs/ufs/ufs_dirhash.c`的UFS目录哈希缓存是一个每文件系统的内存中加速器，用于目录查找。在正常操作下缓存是有益的；在内存压力下它成为负担，所以子系统注册一个`vm_lowmem`处理程序，丢弃缓存条目：

```c
EVENTHANDLER_REGISTER(vm_lowmem, ufsdirhash_lowmem, NULL,
    EVENTHANDLER_PRI_FIRST);
```

第四个参数是`EVENTHANDLER_PRI_FIRST`，要求在已注册的`vm_lowmem`处理程序列表中早期运行。dirhash作者选择早期执行，因为缓存是纯粹的可回收内存：及时释放它给其他处理程序（它们可能持有更脏或更不容易释放的状态）更好的成功机会。

回调本身做真正的工作：遍历哈希表，释放可以释放的条目，记录释放的内存。本质的设计点是，如果没有东西可以释放，回调不会panic；它只是返回，什么也不做。

这是第25章`vm_lowmem`模式在一个不是设备驱动程序但共享规范的子系统中。经验教训传递：如果`myfirst`有缓存，形状已经在这里。

### `tcp_subr`：没有Softc的`vm_lowmem`

`/usr/src/sys/netinet/tcp_subr.c`的TCP子系统也注册了`vm_lowmem`，但其注册以一种有启发性的方式不同：

```c
EVENTHANDLER_REGISTER(vm_lowmem, tcp_drain, NULL, LOWMEM_PRI_DEFAULT);
```

第三个参数（回调数据）是`NULL`，不是softc指针。TCP子系统没有单个softc；其状态分散在许多结构中。回调必须通过其他方式找到其状态（全局变量、每CPU变量、哈希表查找）。

这提出了本章暗示过的一个问题：何时可以接受传递`NULL`作为回调参数？答案是：当回调有另一种方式找到其状态时。对于有每设备softc的驱动程序，传递`sc`几乎总是正确的选择。对于有全局状态的子系统，传递`NULL`并让回调使用其已知的全局变量是好的。

`myfirst`将始终传递`sc`，因为`myfirst`是每设备驱动程序。发现自己编写子系统级别回调的读者应该认识到，当主题是全局的时，模式会以微妙的方式改变。

### `if_vtnet`：围绕特定故障的限速日志

`/usr/src/sys/dev/virtio/network/if_vtnet.c`的VirtIO网络驱动程序为`mmcsd`提供了一个更窄但有启发性的对照。`mmcsd`在每个热路径错误周围包装限速日志，`if_vtnet`仅在特定的不当行为周围使用`ppsratecheck`：一个设置了ECN位的TSO数据包，而VirtIO主机未协商ECN。调用点小而自包含：

```c
static struct timeval lastecn;
static int curecn;
...
if (ppsratecheck(&lastecn, &curecn, 1))
        if_printf(sc->vtnet_ifp,
            "TSO with ECN not negotiated with host\n");
```

两个细节值得指出。首先，限速状态在文件作用域声明为`static`，不是每softc。作者决定"VirtIO主机与guest关于ECN不一致"是系统级别的配置错误而不是每接口的故障，所以一个共享桶就够了；每softc状态会让单个虚拟接口的洪水饿死另一个的。其次，上限是1 pps，这是故意激进的：警告是信息性的，在整个系统中每秒出现不超过一次。预期警告经常触发的驱动程序设计者可以提高上限。

FreeBSD还提供`ratecheck(9)`，`ppsratecheck`的事件计数兄弟。`ratecheck`在上一次允许事件以来的时间超过阈值时触发；`ppsratecheck`在最近突发率低于上限时触发。它们互补：`ratecheck`在你想要消息之间的最小间隔时更好，`ppsratecheck`在你想要最大突发率时更好。

`if_vtnet`的启示是限速状态可以是全局的或每实例的，选择遵循预期的失败形状。第25章将状态放在softc中，因为`myfirst`的错误是每实例的；不同的驱动程序可能合理地做出不同的选择。

### `vt_core`：按序列的多个`EVENTHANDLER`注册

`/usr/src/sys/dev/vt/vt_core.c`的虚拟终端子系统在其生命周期的不同点注册多个事件处理程序。控制台窗口的关闭处理程序是其中之一：

```c
EVENTHANDLER_REGISTER(shutdown_pre_sync, vt_window_switch,
    vw, SHUTDOWN_PRI_DEFAULT);
```

`SHUTDOWN_PRI_DEFAULT`是中性优先级：VT切换在请求`SHUTDOWN_PRI_FIRST`的任何东西之后、在请求`SHUTDOWN_PRI_LAST`的任何东西之前运行。选择是故意的：终端切换对其他子系统没有排序要求，所以作者选择了默认，而不是声明驱动程序不需要的优先级。

对我们的目的来说重要的是`vt_core`在引导时注册此处理程序一次，并且从不注销它。生命周期是内核生命周期的驱动程序不需要注销；内核在其消失后从不调用处理程序。像`myfirst`这样的可以作为模块加载和卸载的驱动程序确实需要注销，因为卸载模块会销毁处理程序的代码。规则是：如果你的代码可以在内核继续运行时消失，则注销。

这种区别对模块作者很重要。内置代码路径通常看起来像是缺少注销，但它们不是；它们只是不需要它。逐字遵循内置模式的模块将在卸载时panic。

### FFS (`ffs_alloc.c`)：每条件限速桶

`/usr/src/sys/ufs/ffs/ffs_alloc.c`的FFS分配器是文件系统而不是设备驱动程序，但它面对的恰好是第25章所讲的日志洪水问题。反复用完块或inode的磁盘可以为每个失败的`write(2)`调用发出一个错误，这在实践中是无界的。分配器在四个不同的位置使用`ppsratecheck`，它限定限速状态范围的方式是桶设计的一个好教训。

每个挂载的文件系统在其挂载结构中承载限速状态。两个独立的桶处理两种不同类型的错误：

```c
/* "文件系统满"报告：块或inode耗尽。 */
um->um_last_fullmsg
um->um_secs_fullmsg

/* 柱组完整性报告。 */
um->um_last_integritymsg
um->um_secs_integritymsg
```

"文件系统满"桶在两个代码路径（`ffs_alloc`和`ffs_realloccg`，都写入像"写入失败，文件系统已满"的消息）加上inode耗尽之间共享。完整性桶在两个不同的完整性失败（柱校验和不匹配和魔术号不匹配）之间共享。在每个调用点，形状是相同的：

```c
if (ppsratecheck(&ump->um_last_fullmsg,
    &ump->um_secs_fullmsg, 1)) {
        UFS_UNLOCK(ump);
        ffs_fserr(fs, ip->i_number, "filesystem full");
        uprintf("\n%s: write failed, filesystem is full\n",
            fs->fs_fsmnt);
        ...
}
```

三个设计决策可见。首先，限速状态是每挂载点的，不是全局的。高写入的文件系统不应该抑制也在填充的另一个文件系统的消息。其次，上限是1 pps：每个挂载点每桶每秒最多一条消息。第三，相关消息共享一个桶（所有"满"消息；所有"完整性"消息），而不相关的消息有自己的。`ffs_alloc.c`的作者决定，被"文件系统满"淹没的操作员不应该也被同一挂载点的"inode用尽"淹没；两者是同一条件的症状，每秒一条消息就够了。

像`myfirst`这样的伪驱动程序可以直接借用该模式。如果`myfirst`有朝一日增长一类与容量相关的错误（比如写路径的"缓冲区满"和ioctl路径的"没有空闲槽"），它们应该放在同一个桶中。完全不同的失败（比如"命令版本不匹配"）应该有自己的桶。`ffs_alloc.c`应用于文件系统的规范对设备驱动程序来说保持不变。

### 阅读更多驱动程序

每个FreeBSD驱动程序都是特定团队如何解决第25章命名的问题的案例研究。`/usr/src/sys`的一些区域特别适合模式狩猎：

- `/usr/src/sys/dev/usb/` 用于USB驱动程序：到处都是`MODULE_DEPEND`和`MODULE_PNP_INFO`。
- `/usr/src/sys/dev/pci/` 用于PCI驱动程序：工业规模的标签清理attach例程。
- `/usr/src/sys/dev/cxgbe/` 用于复杂的现代驱动程序：限速日志记录、有数百个OID的sysctl树、通过模块ABI的版本控制。
- `/usr/src/sys/netinet/` 用于子系统级别的`EVENTHANDLER`使用。
- `/usr/src/sys/kern/subr_*.c` 用于许多不同子系统ID的`SYSINIT`示例。

阅读新驱动程序时，从找到其attach函数开始。计算获取和标签；它们应该匹配。找到detach函数。确认它以相反顺序释放资源。找到错误路径，看每个返回什么errno。在热路径上触发的任何日志消息附近查找`ppsratecheck`或`ratecheck`。查找`MODULE_DEPEND`声明。查找`EVENTHANDLER_REGISTER`并确认有匹配的注销。

一旦模式熟悉，这些检查中的每一个都只需要几秒钟。每一个都加强你自己的直觉，知道模式何时正在或未正确应用。

### 你不会看到什么

第25章推荐的一些模式并不出现在每个驱动程序中，它们的缺失并不总是错误。知道哪些模式是可选的可以防止你在任何地方期待它们。

`MAINTENANCE.md`是本书推荐的习惯，不是FreeBSD的要求。大多数树内驱动程序不发布每驱动程序的维护文件；相反，手册页是面向操作员的参考，发布说明承载变更日志。两种解决方案都有效；它们之间的选择是项目约定。

`devctl_notify`是可选的。许多驱动程序不发出任何事件，没有规则要求它们必须。当有操作员实际上想要对其做出反应的事件时，该模式是有价值的；对于行为安静且没有操作员可见状态更改的驱动程序，发出事件是不必要的。

`DRIVER_MODULE`之外的`SYSINIT`在现代驱动程序中不常见。大多数驱动程序级别的工作在`device_attach`中完成，它每实例运行。显式的`SYSINIT`注册在子系统和核心内核代码中最常见；单个驱动程序很少需要一个。第25章引入`SYSINIT`是因为读者最终会遇到它，而不是因为大多数驱动程序使用它。

显式的模块事件处理程序（驱动程序提供的`MOD_LOAD`/`MOD_UNLOAD`函数而不是`DRIVER_MODULE`）也不常见。它在驱动程序在加载时需要不适合Newbus模型的自定义行为时存在，但大多数驱动程序愉快地使用默认。

当你阅读一个省略这些模式之一的驱动程序时，缺失通常反映特定于该驱动程序的设计决策，不是规范的失败。模式是工具；不是每个工作都需要每个工具。

### 演练总结

本章引入的每个模式在FreeBSD源树的某个地方可见。`mmcsd`驱动程序限速其热路径日志。`uftdi`驱动程序声明其模块依赖和PNP元数据。`iscsi`驱动程序注册一个有优先级的`shutdown_pre_sync`处理程序。UFS dirhash缓存在`vm_lowmem`上释放内存。这些中的每一个都是第25章规范的真实、发布、测试过的应用。

用模式词汇阅读这些驱动程序加速你自己的直觉，比任何单作者教科书都快。模式重复；驱动程序不同。一旦你能命名模式，你阅读的每个新驱动程序都会强化它。

第25章末尾的`myfirst`驱动程序承载每个规范。你刚刚看到相同的规范被八个其他驱动程序承载。下一步是你自己在一个不在本书中的驱动程序上承载它们。那是一生的工作，基础现在在你的手中。

