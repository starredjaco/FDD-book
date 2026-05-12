---
title: "电源管理"
description: "第22章作为第四部分的收尾，教授myfirst驱动如何应对挂起、恢复和关机操作。从设备驱动角度解释电源管理的含义；ACPI睡眠状态（S0-S5）和PCI设备电源状态（D0-D3hot/D3cold）如何组合形成完整转换；DEVICE_SUSPEND、DEVICE_RESUME、DEVICE_SHUTDOWN和DEVICE_QUIESCE方法的作用及内核调用顺序；如何安全地静默中断、DMA、定时器和延迟工作；如何在恢复时恢复状态而不丢失数据；运行时电源管理与全系统挂起的区别；如何使用acpiconf、zzz和devctl从用户空间测试电源转换；如何调试冻结设备、丢失中断和恢复后DMA故障；以及如何将电源感知代码重构到独立文件中。驱动从1.4-dma版本发展到1.5-power版本，新增myfirst_power.c和myfirst_power.h文件，新增POWER.md文档，完成第四部分后驱动能够像处理attach-detach一样干净地处理挂起-恢复周期。"
partNumber: 4
partName: "硬件与平台级集成"
chapter: 22
lastUpdated: "2026-04-19"
status: "complete"
author: "Edson Brandi"
reviewer: "TBD"
translator: "AI辅助翻译为简体中文"
estimatedReadTime: 210
language: "zh-CN"
---

# 电源管理

## 读者指南与学习目标

第21章结束时驱动版本为`1.4-dma`。该驱动能够挂载到PCI设备、分配MSI-X向量、通过过滤器加任务流水线服务中断、通过`bus_dma(9)`缓冲区移动数据，并在要求分离时清理资源。对于一个能够启动、运行并最终卸载的驱动来说，这些机制已经完备。驱动尚未处理的是现代系统抛给它的第三种事件：电源即将发生变化的时刻。

电源变化与挂载和分离不同。挂载从零开始，以工作设备结束。分离从工作设备开始，以归零结束。两者都是驱动本身可以从容处理的一次性转换。挂起则不同。驱动进入挂起时已经在运行，有活跃的中断、活跃的DMA传输、活跃的定时器，以及内核仍期望响应请求的设备。驱动必须在狭窄的时间窗口内停止所有这些活动，将设备交给低功耗状态，在电源丢失后仍能记住需要知道的信息，然后在另一侧重新组装一切，仿佛什么都没发生过。理想情况下，用户什么也注意不到。笔记本盖子合上，一秒钟后打开，视频会议在同一个浏览器标签页中继续，仿佛中断从未发生。

第22章教授驱动如何实现这种幻象。本章的范围正是：从驱动层面理解电源管理；内核如何让驱动看到即将到来的电源转换；安全地静默设备以确保没有活动泄漏到转换期间意味着什么；如何保存恢复后驱动需要的状态；如何恢复状态使设备回到用户之前看到的行为；如何将同样的纪律扩展到通过运行时挂起的空闲设备节能；如何从用户空间测试转换；如何调试电源感知驱动面临的典型故障；以及如何组织新代码使驱动在成长过程中保持可读性。本章不涉及后续建立在纪律之上的主题。第23章深入讲解调试和追踪；第22章的电源感知回归脚本只是初步尝试，不是完整工具集。第六部分的网络驱动章节（第28章）增加iflib的电源钩子和多队列挂起协调；第22章保持使用单队列`myfirst`驱动。第七部分的高级实战章节探索热插拔和嵌入式平台的电源域管理；第22章聚焦于ACPI和PCIe主导的桌面和服务器场景。

第四部分的弧线在此以纪律而非新原语收尾。第16章通过`bus_space(9)`赋予驱动寄存器访问词汇。第17章通过模拟设备教会它像设备一样思考。第18章将其引入真实PCI设备。第19章赋予它在单个IRQ上的一对耳朵。第20章赋予它多对耳朵，每个设备关心的队列一个。第21章赋予它双手：能够向设备提供物理地址让设备自行运行传输的能力。第22章教驱动按要求停止所有这些操作，在系统睡眠时礼貌等待，并在系统唤醒时干净地重新开始。这种纪律是驱动在第四部分意义上能够自称生产就绪之前的最后缺失要素。后续章节增加可观测性、专业化和打磨；它们假设电源纪律已经就位。

### 为什么挂起和恢复值得单独一章

在这个阶段，一个自然的问题是，在第21章的深度之后，挂起和恢复是否真的需要整整一章。`myfirst`驱动已经有了干净的分离路径。分离已经释放中断、排空任务、拆除DMA，并将设备恢复到安静状态。驱动能否简单地让挂起调用分离、恢复调用挂载，就完成了？

答案是否定的，原因有三。

首先，**挂起不是分离**。分离是永久的。驱动在分离完成后不需要记住关于设备的任何信息；当设备回来时，它是从头开始的全新挂载。挂起是暂时的，驱动确实需要跨越它记住一些东西。它需要记住软件状态以便用户会话能够从中断处继续。它需要记住已分配的中断向量。它需要记住配置sysctl。它需要记住哪些客户端已打开设备。分离会忘记所有这些；挂起必须不能。两条路径中间共享清理步骤，但两端分道扬镳。将挂起视为分离加后续挂载在狭窄的机械意义上是正确的，在其他所有意义上都是错误的：它会丢弃用户会话、使`/dev/myfirst0`上的打开文件描述符无效、丢失sysctl状态，并要求内核在每次恢复时从原始PCI身份重新探测设备。这不是现代FreeBSD驱动的工作方式，第22章展示更好的模式。

其次，**时间预算不同**。分离可以做到彻底。一个花费五百毫秒分离的驱动对用户没有明显影响；分离发生在启动、模块卸载或设备移除时，这些时刻被认为是慢的。挂起必须在每台设备几十毫秒的预算内完成，在一台有百个设备的笔记本上，因为总和是用户注意到的盖子关闭延迟。一个执行完整分离式清理、等待队列自然排空、解除每个分配并在恢复时重建一切的驱动，在整个设备队列中会明显变慢。第22章的模式是快速停止活动、保存需要保存的内容、保留分配、从保存状态恢复。这种模式是使典型笔记本的挂起-恢复保持在一秒以内的原因。

第三，**内核为电源转换给驱动提供了特定契约**，该契约有自己的词汇、操作顺序和故障模式。`DEVICE_SUSPEND`和`DEVICE_RESUME` kobj方法不仅仅是"名字不同的分离和挂载"。它们在系统范围挂起序列的特定点被调用，设备树以特定顺序遍历，它们与PCI层的自动配置空间保存恢复、ACPI的睡眠状态机制、中断子系统的掩码和取消掩码调用，以及遍历设备树的`bus_generic_suspend`和`bus_generic_resume`助手交互。忽略契约的驱动可能在分离、DMA和中断处理期间仍然看起来正确，但仅在用户合上盖子时失败。这类故障以难以调试著称，因为难以重现，第22章投入时间明确契约以避免故障发生。

第22章通过具体地、用`myfirst`驱动作为运行示例教授这三个理念，赢得其位置。完成第22章的读者可以为任何FreeBSD驱动添加`device_suspend`、`device_resume`和`device_shutdown`方法，知道章节纪律适用于何处，并理解ACPI层、PCI层和驱动自身状态之间的交互。这项技能直接适用于读者将来工作的每个FreeBSD驱动。

### 第21章留给驱动的状态

继续之前简要检查。第22章扩展第21章第4阶段结束时产生的驱动，标记为版本`1.4-dma`。如果以下任何项目不确定，请在开始本章前返回第21章。

- 您的驱动干净编译并在`kldstat -v`中标识为`1.4-dma`。
- 驱动分配一个或三个MSI-X向量（取决于平台），注册每向量过滤器和任务，将每个向量绑定到CPU，并在挂载期间打印中断横幅。
- 驱动分配`bus_dma`标签，分配4 KB DMA缓冲区，将其加载到映射中，并通过`dev.myfirst.N.dma_bus_addr`暴露总线地址。
- 写入`dev.myfirst.N.dma_test_write=0xAA`触发主机到设备传输；写入`dev.myfirst.N.dma_test_read=1`触发设备到主机传输；两者都将成功记录到`dmesg`。
- 分离路径排空rx任务、排空模拟的callout、等待任何进行中的DMA完成、调用`myfirst_dma_teardown`、以相反顺序拆除MSI-X向量，并释放资源。
- 工作树中的`HARDWARE.md`、`LOCKING.md`、`SIMULATION.md`、`PCI.md`、`INTERRUPTS.md`、`MSIX.md`和`DMA.md`是最新的。
- 测试内核中启用了`INVARIANTS`、`WITNESS`、`WITNESS_SKIPSPIN`、`DDB`、`KDB`和`KDB_UNATTENDED`。

那就是第22章扩展的驱动。新增内容在代码行数上适中但在纪律上重要：新`myfirst_power.c`文件、匹配的`myfirst_power.h`头文件、少量新的softc字段跟踪挂起状态和保存的运行时状态、连接到`device_method_t`表的新`myfirst_suspend`和`myfirst_resume`入口点、新`myfirst_shutdown`方法、从新挂起路径调用第21章静默原语、恢复时不重复挂载的恢复路径、版本升级到`1.5-power`、新`POWER.md`文档，以及回归测试更新。心智模型也在成长：驱动开始将自己的生命周期视为挂载、运行、静默、睡眠、唤醒、再次运行、最终分离，而不仅仅是挂载、运行、分离。

### 您将学到什么

完成本章后，您应该能够：

- 描述电源管理对设备驱动意味着什么，区分系统级和设备级节能，命名全挂起-恢复周期与运行时电源转换的区别。
- 识别ACPI系统睡眠状态（S0、S1、S3、S4、S5）和PCI设备电源状态（D0、D1、D2、D3hot、D3cold），解释它们在单次转换中如何组合，识别各部分哪些是驱动的责任。
- 解释PCIe链路状态（L0、L0s、L1、L1.1、L1.2）和主动状态电源管理（ASPM）的作用，达到足以阅读数据手册并识别平台自动控制与驱动必须显式配置的程度。
- 向驱动的`device_method_t`表添加`DEVICE_SUSPEND`、`DEVICE_RESUME`、`DEVICE_SHUTDOWN`和（可选）`DEVICE_QUIESCE`条目，并实现每个使其与设备树中的`bus_generic_suspend(9)`和`bus_generic_resume(9)`组合。
- 解释在电源转换前安全静默设备意味着什么，应用模式：在设备上掩码中断、停止提交新DMA工作、排空进行中的传输、排空callout和taskqueue、按策略刷新或丢弃缓冲区，使设备处于定义的安静状态。
- 解释PCI层为何在`device_suspend`和`device_resume`周围自动保存恢复配置空间，驱动何时需要用自己的`pci_save_state`/`pci_restore_state`调用补充，何时不应这样做。
- 实现干净的恢复路径：重新启用总线主控、从驱动保存状态恢复设备寄存器、重新装备中断掩码、重新验证设备身份，使客户端重新接入设备而不丢失数据或引发虚假中断。
- 识别设备在挂起期间静默重置的情况，如何检测重置，以及如何仅重建实际丢失的状态。
- 实现运行时电源管理助手，将空闲设备放入D3并按需求唤醒回D0，讨论延迟与功耗权衡。
- 从用户空间用`acpiconf -s 3`或`zzz`触发全系统挂起，用`devctl suspend`和`devctl resume`进行每设备挂起，通过`devinfo -v`、`sysctl hw.acpi.*`和驱动自己的计数器观察转换。
- 调试电源感知代码的典型故障：冻结设备、丢失中断、恢复后DMA错误、丢失PME#唤醒事件、WITNESS关于挂起内持锁睡眠的警告。应用匹配的恢复模式。
- 将驱动的电源管理代码重构到专用`myfirst_power.c`/`myfirst_power.h`对，将驱动版本升级到`1.5-power`，扩展回归测试覆盖挂起和恢复，生成向下一读者解释子系统的`POWER.md`文档。
- 阅读真实驱动如`/usr/src/sys/dev/re/if_re.c`、`/usr/src/sys/dev/xl/if_xl.c`或`/usr/src/sys/dev/virtio/block/virtio_blk.c`中的电源管理代码，将每个调用映射回第22章引入的概念。

列表很长。项目很窄。本章目标是组合，而非孤立地学习单个项目。

### 本章不涵盖的内容

几个相邻主题明确推迟，以保持第22章聚焦于驱动端纪律。

- **高级ACPI内部机制**如AML解释器、SSDT/DSDT表、`_PSW`/`_PRW`/`_PSR`方法语义和ACPI按钮子系统。本章仅通过内核向驱动暴露的层使用ACPI；内部机制属于后续面向平台的章节。
- **休眠到磁盘机制（S4）**。FreeBSD的S4支持在x86上历史上一直不完整，驱动端契约本质上是S3的更严格版本。本章提及S4作为完整性内容，驱动目的将其视为S3。
- **Cpufreq、powerd和CPU频率调节**。这些影响CPU功耗，不影响设备功耗。设备处于D0的驱动不受CPU P状态影响；本章不追求CPU电源管理。
- **SR-IOV PF和VF之间的挂起协调**。虚拟功能挂起有自己的顺序约束，属于专门章节。
- **热插拔和意外移除**。通过物理拔出移除设备精神上类似于挂起，但使用不同代码路径（`BUS_CHILD_DELETED`、`device_delete_child`）。第七部分深入覆盖热插拔；第22章提及关系后继续。
- **Thunderbolt和USB-C坞站挂起**。这些组合ACPI、PCIe热插拔和USB电源管理，属于后续专门章节。
- **嵌入式平台电源域和时钟门控框架**如arm64和RISC-V上的设备树`power-domains`和`clocks`属性。本章全程使用x86 ACPI和PCI约定，在概念并行时提及嵌入式对应物。
- **自定义网络唤醒策略、模式唤醒策略和应用特定唤醒源**。本章解释唤醒源如何连接（PME#、USB远程唤醒、GPIO唤醒），不试图教授每个硬件特定变体。
- **`ksuspend`/`kresume`路径内部和内核在挂起周围的cpuset迁移**。驱动不直接看到这些；它们影响中断路由和CPU下线，不影响驱动的可见契约。

保持在这些界限内使第22章成为关于驱动端电源纪律的章节。词汇可迁移；专业化在后续章节添加细节而不需要新基础。

### 预计时间投入

- **仅阅读**：四到五小时。电源管理概念模型既不像DMA那样密集，也不像中断那样机械；大部分时间用于建立ACPI、PCI和驱动在转换期间如何组合的心理图景。
- **阅读加输入示例**：两到三次会话十到十二小时。驱动分三个阶段演进：骨架挂起和恢复带日志、完整静默和恢复、最终重构到`myfirst_power.c`。每个阶段都很短，但测试是刻意的：遗忘的`bus_dmamap_sync`或错过的中断掩码可能产生静默损坏，只在第五或第六次挂起-恢复周期显示。
- **阅读加所有实验和挑战**：四到五次会话十五到二十小时，包括压力驱动通过重复挂起-恢复周期的实验、强制故意恢复后故障并调试的实验，以及扩展驱动运行时空闲检测的挑战材料。

第3和第4节最密集。如果静默纪律或恢复顺序在首次通过时感觉晦涩，那是正常的。停下来，重读相应图表，在模拟设备上运行匹配练习，形状确定后继续。电源管理是工作心智模型反复回报的主题之一；值得慢慢建立。

### 先决条件

开始本章前，确认：

- 您的驱动源匹配第21章第4阶段（`1.4-dma`）。起点假设每个第21章原语：DMA标签和缓冲区、`dma_in_flight`跟踪器、`dma_cv`条件变量和干净拆除路径。
- 您的实验机运行FreeBSD 14.3，`/usr/src`在磁盘上并与运行内核匹配。
- 已构建、安装并干净启动带有`INVARIANTS`、`WITNESS`、`WITNESS_SKIPSPIN`、`DDB`、`KDB`和`KDB_UNATTENDED`的调试内核。`WITNESS`选项对挂起和恢复工作特别有价值，因为代码路径在非显而易见的锁下运行，内核的电源机制在转换期间收紧几个不变量。
- `bhyve(8)`或`qemu-system-x86_64`可用。第22章的实验在任一目标上工作。挂起-恢复测试不需要真实硬件；`devctl suspend`和`devctl resume`让您直接驱动驱动的电源方法而不涉及ACPI。
- `devinfo(8)`、`sysctl(8)`、`pciconf(8)`、`procstat(1)`、`devctl(8)`、`acpiconf(8)`（如果在带ACPI的真实硬件上）和`zzz(8)`命令在您的路径中。

如果以上任何项目不稳，现在修复。电源管理像DMA一样是潜在弱点在压力下显现的主题。一个在分离上几乎工作的驱动常在挂起时出问题；干净处理一次挂起的驱动常在第十次循环时失败，因为计数器回绕、映射泄漏或条件变量重新初始化不正确。`WITNESS`启用的调试内核是在开发时显现这些错误的工具。

### 如何充分利用本章

四个习惯会快速回报。

首先，将`/usr/src/sys/kern/device_if.m`和`/usr/src/sys/kern/subr_bus.c`加入书签。第一个文件定义`DEVICE_SUSPEND`、`DEVICE_RESUME`、`DEVICE_SHUTDOWN`和`DEVICE_QUIESCE`方法；第二个包含`bus_generic_suspend`、`bus_generic_resume`、`device_quiesce`和将用户空间请求转换为方法调用的devctl机制。在第2节开始时阅读一次，并在处理每节时返回，是对流利度最有用的一件事。

其次，保持三个真实驱动示例近在手边：`/usr/src/sys/dev/re/if_re.c`、`/usr/src/sys/dev/xl/if_xl.c`和`/usr/src/sys/dev/virtio/block/virtio_blk.c`。每个展示不同的电源管理风格。`if_re.c`是带网络唤醒支持、配置空间保存恢复和谨慎恢复路径的完整网络驱动。`if_xl.c`更简单：其`xl_shutdown`只是调用`xl_suspend`，`xl_suspend`停止芯片并设置网络唤醒。`virtio_blk.c`最小：`vtblk_suspend`设置标志并静默队列，`vtblk_resume`清除标志并重启I/O。第22章将在`myfirst`驱动的模式最能说明其行为的时刻引用每个。

第三，手工输入更改并用`devctl suspend`和`devctl resume`练习每个阶段。电源管理是小遗漏产生典型故障的地方：遗忘的中断掩码导致卡住的恢复；遗忘的`bus_dmamap_sync`导致陈旧数据；遗忘的状态变量导致驱动认为传输仍在进行。仔细输入并在每个阶段后运行回归脚本可以在发生时立即显现这些错误。

第四，完成第4节后，重读第21章的分离路径。第3节的静默纪律和第4节的恢复纪律都与第21章的分离共享基础设施：`callout_drain`、`taskqueue_drain`、`bus_dmamap_sync`、`pci_release_msi`。并排查看挂起-恢复和挂载-分离使差异可见。挂起不是分离；恢复不是挂载；但它们使用相同构建块，以不同方式组合，两遍查看这种组合值得额外半小时。

### 章节路线图

各节按顺序为：

1. **设备驱动中的电源管理是什么？** 大图景：驱动为何关心电源，系统级和设备级电源管理如何不同，ACPI S状态和PCI D状态意味着什么，PCIe链路状态和ASPM添加什么，以及唤醒源在读者最可能拥有的系统上是什么样子。概念先行，API随后。
2. **FreeBSD的电源管理接口。** Kobj方法：`DEVICE_SUSPEND`、`DEVICE_RESUME`、`DEVICE_SHUTDOWN`、`DEVICE_QUIESCE`。内核调用它们的顺序。`bus_generic_suspend`助手、`pci_suspend_child`路径，以及与ACPI的交互。第一个运行代码：第22章驱动的第1阶段（`1.5-power-stage1`），骨架处理器仅记录日志。
3. **安全静默设备。** 在电源转换前停止活动。掩码中断、停止DMA提交、排空进行中工作、排空callout和taskqueue、刷新策略敏感缓冲区。第2阶段（`1.5-power-stage2`）将骨架变为真实静默。
4. **在恢复时恢复状态。** 从保存状态重新初始化设备。PCI保存/恢复为您做什么和不做什么。重新启用总线主控、恢复设备寄存器、重新装备中断、验证身份、处理设备重置。第3阶段（`1.5-power-stage3`）添加匹配第2阶段静默的恢复路径。
5. **处理运行时电源管理。** 空闲设备节能。检测空闲。将设备放入D3并按需求带回D0。延迟与功耗。本章的可选部分，但有读者可以实验的实际草图。
6. **与电源框架交互。** 从用户空间测试转换。`acpiconf -s 3`和`zzz`用于全系统挂起。`devctl suspend`和`devctl resume`用于每设备挂起。`devinfo -v`用于观察电源状态。包装所有这些的回归脚本。
7. **调试电源管理问题。** 典型故障模式：冻结设备、丢失中断、恢复后DMA错误、丢失PME#唤醒、WITNESS警告。找到每个的调试模式。
8. **重构和版本控制您的电源感知驱动。** 最终拆分为`myfirst_power.c`和`myfirst_power.h`，更新的Makefile，`POWER.md`文档，版本升级。第4阶段（`1.5-power`）。

八个部分之后是`if_re.c`电源管理代码的扩展演示、ACPI睡眠状态、PCIe链路状态、唤醒源和devctl用户空间接口的深入探讨、一组动手实验、一组挑战练习、故障排除参考、收尾第22章故事并开启第23章的包装、桥梁，以及本章末尾的常用快速参考和词汇材料。参考材料意在您处理后续几章时重读；第22章的词汇（挂起、恢复、静默、关机、D0、D3、ASPM、PME#）是每个生产FreeBSD驱动共享的基础。

如果是第一次阅读，请线性阅读并按顺序做实验。如果是复习，第3、4、7节独立，适合单次阅读。

## 第1节：设备驱动中的电源管理是什么？

代码之前，先看图景。第1节教授电源管理在驱动所见层面的含义：为节能而合作的系统层、这些层定义的睡眠状态和设备电源状态、驱动可见性之下的PCIe链路级机制，以及将系统带回的唤醒源。完成第1节的读者可以用ACPI和PCI电源管理的具体词汇而非模糊首字母缩写阅读本章其余部分。

### 驱动为何必须关心电源

读者花费了前六章教驱动如何与设备对话。每章增加一种能力：内存映射寄存器、模拟后端、真实PCI、中断、多向量中断、DMA。在每章中，设备始终准备响应。BAR始终映射。中断向量始终装备。DMA引擎始终开启。这个假设便于教学，也是驱动在正常运行期间的假设。然而，这不是用户的假设。用户假设当笔记本睡眠时，电池缓慢消耗；当NVMe空闲时，它自我冷却；当Wi-Fi卡没有传输内容时，它不从电源拉取瓦特。这些假设是真实的平台工程，驱动是必须合作使工程工作的层之一。

合作意味着承认设备电源状态可能在驱动脚下改变。改变总是有通知：内核调用驱动中的方法告诉它即将发生什么。但通知只有在驱动正确处理时才有效。忽略通知的驱动使设备处于不一致状态，成本以电源转换特有的方式显现：无法唤醒的笔记本、设备级重置后拒绝响应的RAID控制器、盖子关闭后失去连接的USB坞站。这些故障每一个都映射回一个将电源事件视为可选而平台正确性假设其为强制的驱动。

风险不仅在于空闲功耗。不在系统挂起前静默DMA的驱动可能在CPU停止时损坏内存。不在总线进入低功耗状态前掩码中断的驱动可能导致虚假唤醒事件。不在恢复后恢复配置的驱动可能从曾经保存有效地址的寄存器读取零。这些每一个都是内核错误，用户报告的症状是"有时我的机器唤醒不了"。第22章的纪律是防止这类错误的工具。

### 系统级与设备级电源管理

两个听起来相似的词描述不同事物。现在值得厘清，因为本章两者都用，区分全程重要。

**系统级电源管理**是整个计算机的转换。用户按电源按钮、合上盖子或发出`shutdown -p now`。内核遍历设备树，要求每个驱动挂起其设备，然后要么将CPU置于低功耗状态（S1、S3），要么将内存内容写入磁盘（S4），要么关闭电源（S5）。系统中的每个驱动都参与转换。如果任何驱动拒绝，整个转换失败；内核打印如`DEVICE_SUSPEND(foo0) failed: 16`的消息，系统保持唤醒，用户看到笔记本屏幕变暗半秒然后恢复。

**设备级电源管理**是单个设备的转换。内核决定（或被`devctl suspend`告知）某个特定设备可以独立于其他设备进入低功耗状态。例如PCIe NIC在链路空闲几秒后进入D3，在数据包到达时回D0。整个系统保持在S0。其余设备继续工作。用户除空闲期后第一个数据包的延迟略有增加外什么也注意不到，因为NIC必须从D3唤醒。

系统级和设备级转换使用相同的驱动方法。`DEVICE_SUSPEND`既为完整S3转换（所有设备一起挂起）也为针对性`devctl suspend myfirst0`（仅`myfirst0`设备挂起）调用。驱动通常不区分两者；同样的静默纪律对两者都有效。区别在于调用周围的上下文：全系统挂起还禁用除引导CPU外的所有CPU、停止大多数内核线程，并期望每个驱动快速完成；每设备挂起保持系统其余部分运行并从普通内核上下文调用。驱动大多不需要关心。但它确实需要意识到两种上下文存在，因为只测试每设备挂起的驱动可能错过仅在完整系统挂起中出现的错误，反之亦然。

第22章练习两条路径。实验使用`devctl suspend`和`devctl resume`进行快速迭代，因为它们花费毫秒且不涉及ACPI。集成测试使用`acpiconf -s 3`（或`zzz`）练习穿过ACPI层和总线层次结构的完整路径。通过两项测试的驱动比只通过第一项的更有可能在生产中正确。

### ACPI系统睡眠状态（S0到S5）

在大多数读者工作的x86笔记本和服务器上，系统电源状态由ACPI规范描述为一小组称为**S状态**的字母和数字。每个S状态定义"系统有多少仍在运行"的独特级别。驱动不选择S状态（用户、BIOS或内核策略选择），但需要知道存在哪些以及每个对设备意味着什么。

**S0**是工作状态。CPU运行，RAM供电，所有设备处于需要的任何设备电源状态。读者迄今所做的一切都在S0中。这是系统启动进入的状态，也是仅在请求睡眠时才离开的状态。

**S1**称为"待机"或"轻度睡眠"。CPU停止执行，但CPU的寄存器和缓存被保留，RAM保持供电，大多数设备保持D0或D1。唤醒很快（通常一秒或更少）。在现代硬件上S1很少使用，因为S3更节能且唤醒几乎同样快。FreeBSD支持平台广播的S1；大多数平台不再这样做。

**S2**是S1和S3之间很少实现的状态。在大多数平台上不被广播，当被广播时，FreeBSD将其类似于S1处理。本章不再回到S2。

**S3**是"挂起到RAM"，用户语言中也称为"待机"或"睡眠"。CPU停止，CPU上下文丢失并必须在转换前保存，RAM通过自刷新机制保持供电，大多数设备进入D3或D3cold。唤醒在典型笔记本上花费一到三秒。这是用户在笔记本上合上盖子进入的状态。在服务器上，S3是`acpiconf -s 3`或`zzz`产生的状态。第22章的主要测试是S3转换。

**S4**是"挂起到磁盘"或"休眠"。RAM的全部内容写入磁盘映像，电源移除，系统在下次启动时通过读取映像恢复。在FreeBSD上，S4支持在x86上历史上一直不完整（可以产生内存映像但恢复路径不如Linux或Windows那样完善）。对于驱动目的，S4看起来像S3加结尾额外步骤：驱动像S3那样完全挂起。差异对驱动不可见。

**S5**是"软关机"。系统断电；只有唤醒电路（电源按钮、网络唤醒）正在供电。从驱动角度看，S5类似于系统关机；被调用的是`DEVICE_SHUTDOWN`方法，不是`DEVICE_SUSPEND`。

在真实硬件上，读者可以用以下命令查看平台支持的睡眠状态：

```sh
sysctl hw.acpi.supported_sleep_state
```

典型笔记本打印类似：

```text
hw.acpi.supported_sleep_state: S3 S4 S5
```

服务器可能只打印`S5`，因为ACPI挂起在数据中心机器上很少有意义的。VM可能根据虚拟机监控程序打印各种组合。`sysctl hw.acpi.s4bios`表明S4是否BIOS辅助（现代系统很少是）。`sysctl hw.acpi.sleep_state`让读者手动进入睡眠状态；`acpiconf -s 3`是首选的命令行包装器。

对于第22章目的，读者需要意识到S3（常见情况）和S5（关机情况）。S1和S2由驱动像S3一样处理；S4是S3的超集。本章全程将S3视为规范示例。

### PCI和PCIe设备电源状态（D0到D3cold）

设备电源状态由**D状态**描述，PCI规范独立于系统的S状态定义。驱动的方法最直接控制其设备的D状态，值得详细了解每个状态。

**D0**是全开状态。设备供电、时钟、可通过其配置空间和BAR访问，能够执行驱动要求的任何操作。第16到21章的所有工作都在设备处于D0时完成。`PCI_POWERSTATE_D0`是`/usr/src/sys/dev/pci/pcivar.h`中的符号常量。

**D1**和**D2**是PCI规范定义但未严格约束的中间低功耗状态。D1中的设备仍可访问其配置寄存器并可以响应某些I/O；D2中的设备可能丢失更多上下文。这些状态在现代PC上很少使用，因为D0到D3的跳跃通常更可取。大多数驱动不费心处理D1和D2。

**D3hot**是设备实际上关闭但总线仍供电、配置空间仍可访问（读取大多返回零或保存的配置）、如果配置可以发出PME#信号的低功耗状态。大多数设备在挂起期间进入D3hot。

**D3cold**是总线本身已断电的更低功耗状态。设备完全无法访问；从其配置空间读取返回全1。退出D3cold的唯一方法是平台恢复电源，这通常在平台（而非驱动）控制下发生。D3cold在全系统S3和S4期间很常见。

当驱动调用`pci_set_powerstate(dev, PCI_POWERSTATE_D3)`时，PCI层将设备从当前状态转换到D3（具体是D3hot；到D3cold的转换是平台的工作）。当驱动调用`pci_set_powerstate(dev, PCI_POWERSTATE_D0)`时，PCI层将设备带回D0。

FreeBSD中的PCI层还在系统挂起和恢复期间自动管理这些转换。PCI总线驱动为`bus_suspend_child`注册的`pci_suspend_child`函数先调用驱动的`DEVICE_SUSPEND`，然后（如果`hw.pci.do_power_suspend` sysctl为真，默认如此）将设备转换到D3。恢复时，`pci_resume_child`将设备转回D0，从缓存副本恢复配置空间，清除任何待处理的PME#信号，然后调用驱动的`DEVICE_RESUME`。读者可以用以下命令观察行为：

```sh
sysctl hw.pci.do_power_suspend
sysctl hw.pci.do_power_resume
```

两者默认为1。想要禁用自动D状态转换的读者（用于调试或用于在D3中行为异常的设备）可以将其设为0，此时驱动的`DEVICE_SUSPEND`和`DEVICE_RESUME`运行但设备在转换期间保持在D0。

对于第22章，重要事实是：

- 驱动的`DEVICE_SUSPEND`方法在D状态改变之前运行。驱动在设备仍处于D0时静默。
- 驱动的`DEVICE_RESUME`方法在设备已返回D0之后运行。驱动在设备可访问时恢复。
- 驱动通常不在挂起和恢复期间直接调用`pci_set_powerstate`。PCI层自动处理。
- 驱动通常不直接调用`pci_save_state`和`pci_restore_state`。PCI层也通过`pci_cfg_save`和`pci_cfg_restore`自动处理。
- 驱动确实保存和恢复自己的设备特定状态：硬件可能已丢失的BAR局部寄存器内容、跟踪运行时配置的softc字段、中断掩码值。PCI层不知道这些。

边界是PCI配置空间结束和BAR访问寄存器开始的地方。PCI层保护前者；驱动保护后者。

### PCIe链路状态和主动状态电源管理（ASPM）

在设备D状态之下的层是PCIe链路本身。根复杂体和端点之间的链路可以处于几种**L状态**之一，当链路上流量足够低时L状态之间的转换自动发生。

**L0**是全开链路状态。数据正常流动。延迟最小。这是设备活动时链路所处的状态。

**L0s**是链路空闲几微秒后进入的低功耗状态。发送方在一侧关闭其输出驱动器；链路是双向的，所以另一侧的L0s是独立的。从L0s恢复需要数百纳秒。这是平台在流量突发时可以自动进行的廉价节能。

**L1**是链路在更长空闲期（数十微秒）后进入的更深低功耗状态。双方关闭更多物理层电路。恢复需要微秒。这在延迟惩罚可接受的轻负载期间使用。

**L1.1**和**L1.2**是PCIe 3.0及更高版本对L1的细化，添加进一步的电源门控，以更慢的唤醒为代价允许更低的空闲电流。

**L2**是D3cold和S3期间使用的近乎关闭的链路状态；链路实际上关闭，唤醒需要完全重新协商。驱动通常不直接管理L2；它是设备进入D3cold的副作用。

控制L0和L0s/L1之间转换的机制称为**主动状态电源管理（ASPM）**。ASPM是通过链路两端的PCIe能力寄存器配置的每链路特性。它可以通过平台策略启用、禁用或限制为仅L0s。在FreeBSD上，ASPM通常由固件通过ACPI控制（`_OSC`方法告诉操作系统管理哪些能力）；除非明确告知，否则内核不会质疑固件策略。

对于第22章和大多数FreeBSD驱动，ASPM是平台关注点，不是驱动关注点。驱动不配置ASPM；平台配置。驱动不需要在挂起周围保存或恢复ASPM状态；PCI层将PCIe能力寄存器作为自动配置空间保存恢复的一部分处理。想要为特定设备禁用ASPM的驱动（例如，因为设备有已知的使L0s不安全的勘误）可以通过显式读写PCIe链路控制寄存器来做到，但这很少见且特定。

读者不需要向`myfirst`驱动添加ASPM代码。知道L状态存在、它们根据流量自动转换、驱动的D状态和链路的L状态相关但不同、平台处理ASPM配置就足够了。如果读者将来工作的驱动有指定ASPM勘误的数据表，读者会知道去哪里查找。

### 挂起-恢复周期的剖析

将各部分放在一起，从驱动角度看完整系统挂起-恢复周期看起来是这样，跟踪`myfirst`驱动穿过S3转换：

1. 用户合上笔记本盖子。ACPI按钮驱动（`acpi_lid`）注意到事件，根据系统策略触发到状态S3的睡眠请求。
2. 内核开始挂起序列。用户态守护进程暂停；内核冻结非必要线程。
3. 内核以相反子顺序遍历设备树并在每个设备上调用`DEVICE_SUSPEND`。PCI总线的`bus_suspend_child`调用`myfirst`驱动的`device_suspend`方法。
4. `myfirst`驱动的`device_suspend`运行。它在设备上掩码中断、停止接受新DMA请求、等待任何进行中的DMA完成、排空其任务队列、记录转换日志，并返回0表示成功。
5. PCI层注意到`myfirst`的挂起成功。它调用`pci_cfg_save`缓存PCI配置空间。如果`hw.pci.do_power_suspend`为1（默认），它通过`pci_set_powerstate(dev, PCI_POWERSTATE_D3)`将设备转换到D3hot。
6. 在树更高处，PCI总线本身、主机桥，最终平台经过它们自己的`DEVICE_SUSPEND`调用。ACPI装备其唤醒事件。CPU进入对应S3的低功耗状态。内存子系统进入自刷新。PCIe链路进入L2或类似状态。
7. 时间流逝。在驱动观察的尺度上，没有时间流逝；内核没有运行。
8. 用户打开盖子。平台的唤醒电路唤醒CPU。ACPI执行早期恢复步骤：CPU上下文恢复，内存刷新，平台固件重新初始化必须的内容。
9. 内核的恢复序列开始。它以正向顺序遍历设备树，在每个设备上调用`DEVICE_RESUME`。
10. 对于`myfirst`，PCI总线的`bus_resume_child`通过`pci_set_powerstate(dev, PCI_POWERSTATE_D0)`将设备转回D0。它调用`pci_cfg_restore`将缓存的配置空间写回设备。它用`pci_clear_pme`清除任何待处理的PME#信号。然后调用驱动的`device_resume`方法。
11. 驱动的`device_resume`运行。设备处于D0，其配置空间已恢复，其BAR寄存器为零或默认值。驱动根据需要重新启用总线主控，从保存状态写回设备特定寄存器，重新装备中断掩码，并将设备标记为已恢复。返回0。
12. 内核的恢复序列沿树继续。用户态线程解冻。用户看到工作系统，通常一到三秒内。

驱动有工作的每一步是第4步和第11步。其他一切都是平台或通用内核机制。驱动的工作是使这两步正确，并理解周围步骤足以解释观察到的行为。

### 唤醒源

已挂起的设备可以是系统唤醒的原因。发生的方式取决于总线：

- 在**PCIe**上，D3hot中的设备可以发出**PME#**信号（电源管理事件）。平台的根复杂体将PME#转换为唤醒事件，ACPI `_PRW`方法标识其使用的GPE（通用目的事件），ACPI子系统将GPE转换为从S3唤醒。在FreeBSD上，`pci_enable_pme(dev)`函数启用设备的PME#输出；`pci_clear_pme(dev)`清除任何待处理信号。`pci_has_pm(dev)`助手表明设备是否有电源管理能力。
- 在**USB**上，设备可以通过其标准USB描述符请求**远程唤醒**。主机控制器（`xhci`、`ohci`、`uhci`）将唤醒转换为上游的PME#或等效信号。驱动通常不直接处理这个；USB栈处理。
- 在**嵌入式平台**上，设备可以发出连接到平台唤醒逻辑的**GPIO**引脚。设备树`interrupt-extended`或`wakeup-source`属性标识哪些引脚是唤醒源。FreeBSD的GPIO intr框架处理这个。
- 在**网络唤醒**上，网络控制器在挂起期间监视魔法数据包或模式匹配，在看到一个时发出PME#。驱动和平台都必须配置；`if_re.c`中的`re_setwol`是驱动端的好例子。

对于`myfirst`驱动，模拟设备没有真正的唤醒源（它在模拟之外没有物理存在）。本章在适当位置解释机制，展示`pci_enable_pme`做什么以及在哪里调用，将实际唤醒触发留给模拟后端的手动触发。真实硬件驱动会在请求网络唤醒时在挂起路径中调用`pci_enable_pme`，在恢复路径中调用`pci_clear_pme`以确认任何待处理信号。

### 真实世界示例：Wi-Fi、NVMe、USB

用读者实际使用过的设备来锚定理念是有帮助的。考虑三个。

像Linux上`iwlwifi`或FreeBSD上`iwn`处理的**Wi-Fi适配器**是持续的电源管理公民。在S0中，它大部分时间花在芯片本身的低功耗空闲状态，与接入点关联但不活跃交换数据包；当看到数据包时，它唤醒到D0几毫秒，交换数据包，回到空闲。在系统挂起（S3）时，内核要求驱动保存状态，驱动告诉芯片干净断开关联（或者如果用户想要无线唤醒则设置WoWLAN模式），PCI层将芯片转换到D3。恢复时，反向发生：芯片回D0，驱动恢复状态，重新关联接入点。用户感知盖子打开后一到两秒延迟Wi-Fi恢复，这几乎完全是重新关联时间，不是驱动的恢复时间。

**NVMe SSD**通过自己的电源状态机制（NVMe规范中定义为PSx状态，PS0是全功率，更高数字是更低功耗）内部处理电源状态。NVMe驱动通过刷新队列、等待进行中命令完成、告诉控制器进入低功耗状态来参与系统挂起。恢复时，驱动恢复队列配置，告诉控制器回到PS0，系统恢复磁盘I/O。因为NVMe队列大且DMA密集，NVMe挂起路径是遗漏`bus_dmamap_sync`或队列排空的典型位置，表现为恢复后文件系统损坏。

**USB设备**由USB主机控制器驱动（通常是`xhci`）处理。主机控制器驱动是实现`DEVICE_SUSPEND`和`DEVICE_RESUME`的那个；单个USB驱动（键盘、存储、音频等）通过USB框架自己的挂起和恢复机制通知。USB设备驱动很少需要自己的`DEVICE_SUSPEND`方法；USB框架处理转换。

第22章中的`myfirst`驱动使用PCI端点模型，这是最常见的情况，也是其他情况特化的契约。先学习PCI模式为读者提供了后续查看Wi-Fi模式、NVMe模式和USB模式时所需的工具。

### 读者获得了什么

第1节是概念性的。读者不应感到有义务记住提到的每个状态的每个细节。读者应从中获得的是：

- 电源管理是分层系统。ACPI定义系统状态。PCI定义设备状态。PCIe定义链路状态。驱动最直接看到其设备状态。
- 每层的状态可以转换，转换可以组合。系统S3意味着每个设备的D3意味着每个链路的L2。每设备D3（来自运行时PM）不意味着系统S3；系统保持在S0。
- 驱动与PCI和ACPI层有特定契约。驱动负责在挂起时静默设备活动并在恢复时恢复设备状态。PCI层自动处理配置空间保存、D状态转换和PME#唤醒信号。ACPI处理系统范围的唤醒。
- 唤醒源存在并通过特定链连接（PME#、远程唤醒、GPIO）。驱动通常通过助手API启用和禁用它们；不直接与唤醒硬件对话。
- 测试也是分层的。`devctl suspend`/`devctl resume`仅练习驱动方法。`acpiconf -s 3`练习整个系统。好的回归脚本两者都用。

有了这个图景，第2节可以介绍驱动用来加入这个系统的FreeBSD特定API。

### 第1节收尾

第1节建立了驱动为何必须关心电源、系统级和设备级电源管理意味着什么、ACPI S状态和PCI D状态长什么样、PCIe L状态添加什么、从驱动角度看挂起-恢复周期如何流动、以及唤醒源是什么。它没有展示任何驱动代码；那是第2节的工作。读者现在拥有词汇和心智模型：挂起是平台宣布的转换，驱动静默；PCI层将设备移到D3；系统睡眠；唤醒时，PCI层将设备带回D0；驱动恢复。

牢记这个图景，下一节介绍FreeBSD的具体API：四个kobj方法（`DEVICE_SUSPEND`、`DEVICE_RESUME`、`DEVICE_SHUTDOWN`、`DEVICE_QUIESCE`），内核如何调用它们，它们如何与`bus_generic_suspend`和`pci_suspend_child`组合，以及`myfirst`驱动的方法表如何增长以包含它们。

## 第2节：FreeBSD的电源管理接口

第1节描述了ACPI、PCI、PCIe和唤醒源的分层世界。第2节将视野收窄到FreeBSD内核的接口：驱动实现的具体kobj方法、内核调度它们的方式，以及使整个方案可控的通用助手。到本节结束时，`myfirst`驱动有了骨架电源管理实现，可以编译、记录转换日志，并可以用`devctl suspend`和`devctl resume`练习。骨架尚不静默DMA或恢复状态；那是第3节和第4节的工作。第2节的工作是让内核调用驱动方法，以便本章其余部分有具体内容可构建。

### 四个Kobj方法

FreeBSD设备框架将驱动视为`/usr/src/sys/kern/device_if.m`中定义的kobj接口的实现。该文件是一个小型领域特定语言（`make -V`规则将其转换为函数指针和包装器的头文件），定义每个驱动可以实现的方法集。第16到21章的工作已填充常用方法：`DEVICE_PROBE`、`DEVICE_ATTACH`、`DEVICE_DETACH`。电源管理再添加四个，都在同一文件中有注释记录，读者可以直接阅读：

1. **`DEVICE_SUSPEND`**在内核决定将设备置于挂起状态时调用。方法运行时设备仍在D0，驱动仍对其负责。方法的工作是停止活动，如果需要则保存不会自动恢复的任何状态。返回0表示成功。返回非零否决挂起。

2. **`DEVICE_RESUME`**在设备从挂起返回D0的路上被调用。方法的工作是恢复硬件丢失的任何状态并恢复活动。返回0表示成功。返回非零导致内核记录警告；此时恢复无法有意义地否决，因为系统其余部分已经恢复。

3. **`DEVICE_SHUTDOWN`**在系统关机期间调用，让驱动为重启或断电将设备留在安全状态。许多驱动通过调用其挂起方法实现此功能，因为两项任务相似（干净停止设备）。返回0表示成功。

4. **`DEVICE_QUIESCE`**在框架希望驱动停止接受新工作但尚未决定分离时调用。这是分离的更软形式：设备仍挂载，资源仍分配，但驱动应拒绝新提交并让进行中的工作排空。此方法是可选的，比其他三个更少显式实现；`device_quiesce`在`DEVICE_DETACH`之前由devctl层自动调用，所以实现挂起和静默两者的驱动常在它们之间共享代码。

该文件还包含默认空操作实现：`null_suspend`、`null_resume`、`null_shutdown`、`null_quiesce`。不实现某个方法的驱动获得空操作，返回0不做任何事。这就是第16到21章没有显式提及这些方法的原因：空操作正被安静使用，对于设备永远供电且分离仅在模块卸载时发生的驱动，空操作为大多数工作负载提供正确行为。

第22章的第一步是用真实实现替换这些空操作。

### 向驱动方法表添加方法

FreeBSD驱动中的`device_method_t`数组列出驱动实现的kobj方法。`myfirst`驱动当前的方法数组（在`myfirst_pci.c`中）看起来像这样：

```c
static device_method_t myfirst_pci_methods[] = {
        DEVMETHOD(device_probe,   myfirst_pci_probe),
        DEVMETHOD(device_attach,  myfirst_pci_attach),
        DEVMETHOD(device_detach,  myfirst_pci_detach),

        DEVMETHOD_END
};
```

添加电源管理在机械上很简单：驱动添加三（或四）行`DEVMETHOD`。左边是来自`device_if.m`的kobj方法名；右边是驱动的实现。完整集合看起来像这样：

```c
static device_method_t myfirst_pci_methods[] = {
        DEVMETHOD(device_probe,    myfirst_pci_probe),
        DEVMETHOD(device_attach,   myfirst_pci_attach),
        DEVMETHOD(device_detach,   myfirst_pci_detach),
        DEVMETHOD(device_suspend,  myfirst_pci_suspend),
        DEVMETHOD(device_resume,   myfirst_pci_resume),
        DEVMETHOD(device_shutdown, myfirst_pci_shutdown),

        DEVMETHOD_END
};
```

`myfirst_pci_suspend`、`myfirst_pci_resume`和`myfirst_pci_shutdown`函数是新的；它们尚不存在。第2节其余部分展示每个在骨架层面做什么。

### 原型和返回值

四个方法中的每一个都有相同签名：`int`返回值和一个`device_t`参数。`device_t`是调用方法的设备，驱动可以用`device_get_softc(dev)`恢复softc指针。

```c
static int myfirst_pci_suspend(device_t dev);
static int myfirst_pci_resume(device_t dev);
static int myfirst_pci_shutdown(device_t dev);
static int myfirst_pci_quiesce(device_t dev);  /* 可选 */
```

返回值遵循通常的FreeBSD约定。零表示成功。非零是常规errno值，指示出了什么问题：`EBUSY`如果驱动无法挂起因为设备忙，`EIO`如果硬件报告错误，`EINVAL`如果驱动在不可能的状态下被调用。内核的反应因方法而异。

对于`DEVICE_SUSPEND`，非零返回**否决**挂起。内核中止挂起序列，在已成功挂起的驱动上调用`DEVICE_RESUME`，展开部分挂起。这是防止系统在关键设备处于驱动无法中断的状态时进入S3的机制。应谨慎使用；每当有事情发生就从每次挂起返回`EBUSY`是使挂起不可靠的可靠方式。好的驱动仅在设备处于真正无法挂起的状态时才否决。

对于`DEVICE_RESUME`，非零返回被记录但大多被忽略。到恢复运行时，系统正在回来，无论驱动是否喜欢。驱动应记录错误，将设备标记为损坏以便后续I/O干净失败，并返回。恢复时的否决太晚而无用。

对于`DEVICE_SHUTDOWN`，非零返回同样主要是信息性的。系统正在关机；驱动应尽力将设备留在安全状态，但关机失败不是紧急情况。

对于`DEVICE_QUIESCE`，非零返回阻止后续操作（通常是分离）继续。从`DEVICE_QUIESCE`返回`EBUSY`的驱动强制用户等待或使用`devctl detach -f`强制分离。

### 事件传递顺序

内核不会一次在所有驱动上调用`DEVICE_SUSPEND`。它以特定顺序遍历设备树，通常是挂起时**反向子顺序**和恢复时**正向子顺序**。这是因为当每个设备在依赖于它的设备**之后**挂起，在每个依赖于它的设备**之前**恢复时，挂起最安全。

考虑简化的树：

```text
nexus0
  acpi0
    pci0
      pcib0
        myfirst0
      pcib1
        em0
      xhci0
        umass0
```

在S3挂起时，`pci0`下子树的遍历在`pcib0`之前挂起`myfirst0`，在`pcib1`之前挂起`em0`，在`xhci0`之前挂起`umass0`。然后`pcib0`、`pcib1`和`xhci0`挂起。然后`pci0`。然后`acpi0`。然后`nexus0`。每个父设备仅在所有子设备挂起后挂起。

恢复时，顺序相反。`nexus0`先恢复，然后`acpi0`，然后`pci0`，然后`pcib0`、`pcib1`、`xhci0`。这些每个在其子设备上调用`pci_resume_child`，在调用子驱动的`DEVICE_RESUME`之前将子设备转回D0。所以`myfirst0`的`device_resume`运行时`pcib0`已活动，`pci0`已重新配置。

对驱动的实际后果是，在`DEVICE_SUSPEND`期间它仍可正常访问设备（父总线仍启动），在`DEVICE_RESUME`期间也可正常访问设备（父总线先恢复）。驱动不需要处理父设备已挂起的边缘情况。

有一个微妙之处：如果父总线报告其子设备必须以特定顺序挂起（ACPI可以通过这样做来表达隐式依赖），通用助手`bus_generic_suspend`尊重该顺序。`myfirst`驱动的父设备是PCI总线，不需要担心"子设备先于父设备"之外的顺序；PCI总线在其子设备之间没有强顺序。

### bus_generic_suspend、bus_generic_resume和PCI总线

**总线驱动**本身也是设备驱动，当内核在总线上调用`DEVICE_SUSPEND`时，总线通常必须在自身安静之前挂起所有子设备。手工实现会重复，所以内核在`/usr/src/sys/kern/subr_bus.c`中提供两个助手：

```c
int bus_generic_suspend(device_t dev);
int bus_generic_resume(device_t dev);
```

第一个以反向顺序遍历总线的子设备并在每个上调用`BUS_SUSPEND_CHILD`。第二个正向遍历并在每个上调用`BUS_RESUME_CHILD`。如果任何子设备的挂起失败，`bus_generic_suspend`通过恢复已挂起的子设备来展开。

典型的总线驱动直接使用这些助手：

```c
static device_method_t mybus_methods[] = {
        /* ... */
        DEVMETHOD(device_suspend, bus_generic_suspend),
        DEVMETHOD(device_resume,  bus_generic_resume),
        DEVMETHOD_END
};
```

`virtio_pci_modern`总线驱动在`/usr/src/sys/dev/virtio/pci/virtio_pci_modern.c`中就是这样做的，`vtpci_modern_suspend`和`vtpci_modern_resume`各自只调用`bus_generic_suspend(dev)`和`bus_generic_resume(dev)`。

**PCI总线本身**做更复杂的事情：其`bus_suspend_child`是`pci_suspend_child`，其`bus_resume_child`是`pci_resume_child`。这些助手（在`/usr/src/sys/dev/pci/pci.c`中）正是第1节描述的内容：挂起时它们调用`pci_cfg_save`缓存配置空间，然后调用驱动的`DEVICE_SUSPEND`，如果`hw.pci.do_power_suspend`为真则调用`pci_set_powerstate(child, PCI_POWERSTATE_D3)`。恢复时它们反转序列：转回D0，从缓存恢复配置，清除待处理PME#，调用驱动的`DEVICE_RESUME`。

直接挂载到PCI设备的`myfirst`驱动不自己实现总线方法；它是叶子驱动。其电源方法是针对自身设备状态的重要方法。但读者应意识到PCI总线在驱动方法两边都做了工作：挂起时，到驱动的`DEVICE_SUSPEND`运行时，PCI层已保存配置；恢复时，到驱动的`DEVICE_RESUME`运行时，PCI层已恢复配置并将设备带回D0。

### pci_save_state和pci_restore_state：驱动何时调用它们

`pci_cfg_save`/`pci_cfg_restore`处理的自动保存恢复覆盖标准PCI配置寄存器：BAR分配、命令寄存器、缓存行大小、中断线、MSI/MSI-X状态。对于大多数驱动，这已足够，驱动不需要显式调用`pci_save_state`或`pci_restore_state`。

然而，有些情况下驱动确实想手动保存配置。PCI API为此暴露两个助手函数：

```c
void pci_save_state(device_t dev);
void pci_restore_state(device_t dev);
```

`pci_save_state`是缓存当前配置的`pci_cfg_save`包装器。`pci_restore_state`将缓存配置写回；如果调用`pci_restore_state`时设备不在D0，助手在恢复前将其转换到D0。

驱动通常在两种场景下调用它们：

1. **在驱动自己发起的手动`pci_set_powerstate`前后**，例如在运行时电源管理助手中。如果驱动决定在系统处于S0时将空闲设备放入D3，它调用`pci_save_state`，然后`pci_set_powerstate(dev, PCI_POWERSTATE_D3)`。当它唤醒设备时，它调用`pci_set_powerstate(dev, PCI_POWERSTATE_D0)`后跟`pci_restore_state`。

2. **在`DEVICE_SUSPEND`和`DEVICE_RESUME`内部，当自动保存恢复被禁用时**。有些驱动为在D3中行为异常的设备将`hw.pci.do_power_suspend`设为0，自己管理电源状态。在这种情况下驱动也负责保存恢复配置。这是不常见模式。

第22章的`myfirst`驱动在第5节（运行时PM）使用场景1，驱动选择在系统保持S0时将设备停在D3。对于系统挂起，驱动不直接调用这些助手；PCI层处理。

### pci_has_pm助手

并非每个PCI设备都有PCI电源管理能力。旧设备和一些特殊用途设备不广播能力，意味着驱动无法依赖`pci_set_powerstate`或`pci_enable_pme`工作。内核提供助手检查：

```c
bool pci_has_pm(device_t dev);
```

如果设备暴露电源管理能力返回true，否则false。大多数现代PCIe设备返回true。希望对异常硬件健壮的驱动保护其电源相关调用：

```c
if (pci_has_pm(sc->dev))
        pci_enable_pme(sc->dev);
```

`/usr/src/sys/dev/re/if_re.c`中的Realtek驱动在其`re_setwol`和`re_clrwol`函数中使用此模式：如果设备没有PM能力，函数早返回而不尝试触碰电源管理。

### PME#：启用、禁用和清除

在有PM能力的设备上，驱动可以要求硬件在发生唤醒相关事件时发出PME#。API是三个短函数：

```c
void pci_enable_pme(device_t dev);
void pci_clear_pme(device_t dev);
/* 没有显式pci_disable_pme；pci_clear_pme同时清除待处理事件和禁用PME_En位。 */
```

`pci_enable_pme`在设备的电源管理状态/控制寄存器中设置PME_En位，使设备检测到的下一个电源管理事件导致其发出PME#。`pci_clear_pme`清除任何待处理PME状态位并清除PME_En。

例如，想要启用网络唤醒的驱动通常：

1. 配置设备自己的唤醒逻辑（设置模式过滤器、设置魔法数据包标志等）。
2. 在挂起路径中调用`pci_enable_pme(dev)`使设备实际可以发出PME#。
3. 在恢复路径中调用`pci_clear_pme(dev)`确认唤醒事件。

如果不调用`pci_enable_pme`，即使设备自己的唤醒逻辑触发也不会发出PME#。如果不在恢复时调用`pci_clear_pme`，陈旧的PME状态位可能导致未来虚假唤醒事件。

`myfirst`驱动不实现网络唤醒（模拟设备没有唤醒对象），所以这些调用不出现在主驱动代码中。第4节包含简短草图展示它们在真实驱动中的位置。

### 第一个骨架：第1阶段

有了所有背景，我们可以编写`myfirst`挂起、恢复和关机方法的第一个版本。第22章驱动的第1阶段不做实质性事情；仅记录日志并返回成功。目的是让内核调用方法，以便本章其余部分可以渐进测试。

首先，在`myfirst_pci.c`顶部附近添加原型：

```c
static int myfirst_pci_suspend(device_t dev);
static int myfirst_pci_resume(device_t dev);
static int myfirst_pci_shutdown(device_t dev);
```

接下来，扩展方法表：

```c
static device_method_t myfirst_pci_methods[] = {
        DEVMETHOD(device_probe,    myfirst_pci_probe),
        DEVMETHOD(device_attach,   myfirst_pci_attach),
        DEVMETHOD(device_detach,   myfirst_pci_detach),
        DEVMETHOD(device_suspend,  myfirst_pci_suspend),
        DEVMETHOD(device_resume,   myfirst_pci_resume),
        DEVMETHOD(device_shutdown, myfirst_pci_shutdown),

        DEVMETHOD_END
};
```

然后在文件末尾实现三个函数：

```c
static int
myfirst_pci_suspend(device_t dev)
{
        struct myfirst_softc *sc = device_get_softc(dev);

        device_printf(dev, "suspend (stage 1 skeleton)\n");
        atomic_add_64(&sc->power_suspend_count, 1);
        return (0);
}

static int
myfirst_pci_resume(device_t dev)
{
        struct myfirst_softc *sc = device_get_softc(dev);

        device_printf(dev, "resume (stage 1 skeleton)\n");
        atomic_add_64(&sc->power_resume_count, 1);
        return (0);
}

static int
myfirst_pci_shutdown(device_t dev)
{
        struct myfirst_softc *sc = device_get_softc(dev);

        device_printf(dev, "shutdown (stage 1 skeleton)\n");
        atomic_add_64(&sc->power_shutdown_count, 1);
        return (0);
}
```

在`myfirst.h`中的softc添加计数器字段：

```c
struct myfirst_softc {
        /* ... 现有字段 ... */

        uint64_t power_suspend_count;
        uint64_t power_resume_count;
        uint64_t power_shutdown_count;
};
```

在已添加`myfirst` sysctl树的任何函数中，在第21章计数器旁边暴露它们：

```c
SYSCTL_ADD_U64(ctx, kids, OID_AUTO, "power_suspend_count",
    CTLFLAG_RD, &sc->power_suspend_count, 0,
    "DEVICE_SUSPEND被调用的次数");
SYSCTL_ADD_U64(ctx, kids, OID_AUTO, "power_resume_count",
    CTLFLAG_RD, &sc->power_resume_count, 0,
    "DEVICE_RESUME被调用的次数");
SYSCTL_ADD_U64(ctx, kids, OID_AUTO, "power_shutdown_count",
    CTLFLAG_RD, &sc->power_shutdown_count, 0,
    "DEVICE_SHUTDOWN被调用的次数");
```

在Makefile中升级版本字符串：

```make
CFLAGS+= -DMYFIRST_VERSION_STRING=\"1.5-power-stage1\"
```

构建、加载并测试：

```sh
cd /path/to/driver
make clean && make
sudo kldload ./myfirst.ko
sudo devctl suspend myfirst0
sudo devctl resume myfirst0
sysctl dev.myfirst.0.power_suspend_count
sysctl dev.myfirst.0.power_resume_count
dmesg | tail -6
```

`dmesg`中的预期输出是：

```text
myfirst0: suspend (stage 1 skeleton)
myfirst0: resume (stage 1 skeleton)
```

计数器应各为1。如果不是，三件事之一出错：驱动未重新构建，方法表未包含新条目，或`devctl`报告错误因为内核无法以此名称找到设备。

### 骨架证明了什么

第1阶段看起来微不足道，但证明了对本章其余部分重要的三件事：

1. **内核正在传递方法。** 如果计数器递增，从`devctl`通过PCI总线到`myfirst`驱动的kobj调度正确连接。每个后续阶段都建立在此基础上，现在捕捉接线错误比添加真实静默代码后更容易。

2. **方法表正确源化。** 带方法名和驱动函数指针的`DEVMETHOD`行输入正确，头包含正确，`DEVMETHOD_END`终止符就位。此处错误在加载时产生内核恐慌，不是微妙的运行时失败。

3. **驱动计数转换。** 计数器将在整章作为廉价不变量检查有用。一旦系统空闲，`power_suspend_count`应始终等于`power_resume_count`；任何漂移表明两个方法之一有错误。

骨架就位后，第3节可以将挂起方法从仅记录日志的调用变为设备活动的真实静默。

### 分离、静默和挂起的说明

读者可能想知道`DEVICE_DETACH`、`DEVICE_QUIESCE`和`DEVICE_SUSPEND`如何关联。它们看起来相似；每个要求驱动停止做某事。以下是内核强制执行的实际区别：

- **`DEVICE_QUIESCE`**是最软的。它要求驱动停止接受新工作并排空进行中的工作，但设备仍挂载，资源仍分配，另一个请求可以重新激活它。内核在`DEVICE_DETACH`之前调用此方法，给驱动在设备忙时拒绝的机会。
- **`DEVICE_SUSPEND`**在中间。它要求驱动停止活动但保留资源分配，因为恢复时驱动会再次需要它们。设备的状态被保存（部分由内核通过PCI配置保存，部分由驱动通过自己的保存状态）。
- **`DEVICE_DETACH`**是最硬的。它要求驱动停止活动、释放所有资源并忘记设备。回来的唯一方法是通过全新挂载。

许多驱动通过重用挂起路径部分实现`DEVICE_QUIESCE`（停止中断、停止DMA、排空队列），通过直接调用挂起方法实现`DEVICE_SHUTDOWN`。`/usr/src/sys/dev/xl/if_xl.c`正是这样做的：`xl_shutdown(dev)`只是调用`xl_suspend(dev)`。关系是：

- `shutdown` ≈ `suspend`（对于大多数不区分关机特定行为如网络唤醒默认不同的驱动）
- `quiesce` ≈ `suspend`的一半（停止活动，不保存状态）
- `suspend` = quiesce + 保存状态
- `resume` = 恢复状态 + 取消静默
- `detach` = quiesce + 释放资源

第22章完整实现挂起、恢复和关机。它不为`myfirst`驱动实现`DEVICE_QUIESCE`，因为第21章分离路径已正确静默，`device_quiesce`会是冗余的。想要允许"停止I/O但保持设备挂载"状态（例如，优雅支持`devctl detach -f`）的驱动会将`DEVICE_QUIESCE`作为单独方法添加。本章为完整性提及此点后继续。

### 分离路径作为灵感

第21章分离路径是有用的参考，因为它已做了挂起需要做的大部分。分离路径掩码中断、排空rx任务和模拟callout、等待任何进行中的DMA完成，并调用`myfirst_dma_teardown`。挂起路径将做前三个（掩码、排空、等待）并跳过最后一个（拆除）。结构良好的驱动将这些公共步骤提取到共享助手，使两条路径使用相同代码。

第3节正是引入这些助手：`myfirst_stop_io`、`myfirst_drain_workers`、`myfirst_mask_interrupts`。每个是从第21章分离路径提取的小函数。挂起使用它们而不拆除资源；分离使用它们然后拆除。重用使两条路径构造性正确。

### 用WITNESS观察骨架

构建了带`WITNESS`的调试内核的读者现在可以运行骨架并观察任何锁顺序警告。应该没有，因为骨架不获取任何锁。第3节将在挂起路径添加锁获取，`WITNESS`会立即注意到顺序是否与分离中使用的顺序不一致。这是分阶段工作的好处之一：基线安静，所以任何后续警告清楚归因于引入它们的阶段。

### 第2节收尾

第2节建立了FreeBSD特定的电源管理API：四个kobj方法（`DEVICE_SUSPEND`、`DEVICE_RESUME`、`DEVICE_SHUTDOWN`、`DEVICE_QUIESCE`），它们的返回值语义，内核传递它们的顺序，遍历设备树的通用助手（`bus_generic_suspend`、`bus_generic_resume`），以及在驱动方法周围自动保存恢复配置空间的PCI特定助手（`pci_suspend_child`、`pci_resume_child`）。第1阶段骨架给了驱动三个主要方法的仅记录日志实现，向softc添加计数器，并验证`devctl suspend`和`devctl resume`正确调用它们。

骨架不做的是与设备交互。中断处理器继续触发；DMA传输在挂起返回时仍可能进行中；设备不会安静。第3节修复所有这些：它引入静默纪律，将公共停止I/O助手从第21章分离路径中提取出来，将第1阶段骨架变为真正在报告成功前静默设备的第2阶段驱动。

## 第3节：安全静默设备

第2节给了驱动骨架挂起、恢复和关机方法，它们仅记录日志。第3节将挂起骨架变为真实挂起：停止中断、停止DMA、排空延迟工作，并在返回前将设备留在定义的安静状态。正确做到这一点是电源管理中最难的单个部分，也是第21章原语回报的地方。如果您有干净的DMA拆除路径、干净的任务队列排空和干净的callout排空，您已拥有大部分所需内容；静默是将它们以正确顺序应用而不拆除恢复时需要的资源的艺术。

### 静默真正意味着什么

"静默"一词出现在FreeBSD的几个地方（`DEVICE_QUIESCE`、`device_quiesce`、`pcie_wait_for_pending_transactions`），它有特定含义：**将设备带到没有活动进行中、无法启动活动、且硬件不会引发更多中断或做更多DMA的状态**。设备仍完全挂载，仍有所有资源，仍注册了中断处理器，但它什么也不做，在被告知再次启动前不会做任何事。

静默不同于分离，因为分离会展开资源分配。静默也不同于简单的"设置一个标志表示设备忙，以便未来请求阻塞"，因为该标志仅防止新工作进入驱动；它不会停止硬件本身或内核侧基础设施（任务、callout）做任何事。

在第22章的意义上，已静默的设备具有这些属性：

1. 设备未发出也无法被激发发出中断。设备的任何中断掩码都设置为抑制所有源。中断处理器如果被调用，没有事要做。
2. 设备未执行DMA。任何进行中的DMA传输要么已完成要么已中止。引擎的控制寄存器要么处于空闲状态要么已显式重置。
3. 驱动的延迟工作已排空。任务队列中排队的任何任务都已执行或显式等待。任何callout都已排空且不会触发。
4. 驱动的softc字段反映静默状态。`dma_in_flight`标志为false。驱动保持的任何进行中计数器都为零。`suspended`标志为true，所以用户空间或其他驱动可能提交的任何新请求都会收到错误。

只有当四个属性都为真时，设备才真正安静。掩码中断但忘记排空任务队列的驱动仍有后台运行的任务。排空任务队列但忘记callout的驱动仍有可以触发的定时器。排空两者但忘记停止DMA的驱动可能有传输在CPU停止查看后将字节提交到内存。每个遗漏产生自己的典型故障，避免所有这些的最廉价方式是拥有一个以已知顺序执行整个纪律的函数。

### 顺序很重要

上述四个步骤不是独立的。它们有驱动必须尊重的依赖顺序，因为内核侧基础设施和设备交互。考虑如果顺序错误会发生什么。

**如果DMA在掩码中断之前停止**，中断可能在DMA停止和掩码之间到达。过滤器运行，看到陈旧状态位，调度任务。任务运行，期望DMA缓冲区已填充，发现陈旧数据，可能损坏驱动内部状态。最好先掩码中断，以便停止期间没有新中断到达。

**如果在设备停止产生中断之前排空任务队列**，新中断可能在排空返回后触发并在刚排空的队列上调度任务。任务稍后运行，与挂起序列不同步。最好先停止中断，以便不调度新任务。

**如果在停止DMA之前排空callout**，由callout驱动的模拟引擎可能在其callout被拆除时仍有传输在进行中。传输永不完成；`dma_in_flight`保持为true；驱动挂起等待无法到来的完成。最好先停止DMA，等待完成，然后排空callout。

第21章分离路径使用并针对第22章挂起调整的安全顺序是：

1. 将驱动标记为已挂起（设置softc标志使新请求反弹）。
2. 在设备上掩码所有中断（写入中断掩码寄存器）。
3. 停止DMA：如果传输在进行中，中止它并等待其到达终止状态。
4. 排空任务队列（任何正在运行的任务可以完成；不启动新任务）。
5. 排空任何callout（任何进行中的触发可以完成；不发生新触发）。
6. 验证不变量（`dma_in_flight == false`、`softc->busy == 0`等）。

每一步建立在前一步之上。到第6步，设备安静。

### 助手而非内联代码

天真的实现会将所有六个步骤内联在`myfirst_pci_suspend`中。这可行，但重复分离路径已有的代码，使两条路径更难维护。本章偏好的模式是将步骤提取为两条路径都调用的小助手函数。

三个助手足以覆盖整个纪律：

```c
static void myfirst_mask_interrupts(struct myfirst_softc *sc);
static int  myfirst_drain_dma(struct myfirst_softc *sc);
static void myfirst_drain_workers(struct myfirst_softc *sc);
```

每个有一项工作：

- `myfirst_mask_interrupts`写入设备的中断掩码寄存器以禁用驱动关心的每个向量。返回后，此设备不会到达任何中断。
- `myfirst_drain_dma`请求任何进行中的DMA传输停止（设置ABORT位）并等待直到`dma_in_flight`为false。成功返回0，如果设备在超时内未停止返回非零errno。
- `myfirst_drain_workers`在驱动的任务队列上调用`taskqueue_drain`，在模拟的callout上调用`callout_drain`。返回后，没有待处理的延迟工作。

挂起路径按顺序调用三者。分离路径也调用三者，加上`myfirst_dma_teardown`和资源释放调用。两条路径共享静默步骤，仅在末尾不同。

这是挂起路径使用的静默入口点：

```c
static int
myfirst_quiesce(struct myfirst_softc *sc)
{
        int err;

        MYFIRST_LOCK(sc);
        if (sc->suspended) {
                MYFIRST_UNLOCK(sc);
                return (0);  /* 已安静，无事可做 */
        }
        sc->suspended = true;
        MYFIRST_UNLOCK(sc);

        myfirst_mask_interrupts(sc);

        err = myfirst_drain_dma(sc);
        if (err != 0) {
                device_printf(sc->dev,
                    "quiesce: DMA did not stop cleanly (err %d)\n", err);
                /* 不让静默失败；仍需排空工作者。 */
        }

        myfirst_drain_workers(sc);

        return (err);
}
```

注意设计选择：`myfirst_quiesce`在DMA排空失败时不展开。不会停止的DMA是硬件问题，驱动无法作为响应取消掩码或取消挂起标志。驱动记录问题，向调用者报告错误，并继续排空工作者使其余状态仍一致。调用者（`myfirst_pci_suspend`）决定如何处理错误。

### 实现myfirst_mask_interrupts

对于`myfirst`驱动，掩码中断意味着写入设备的中断掩码寄存器。第19章模拟后端在已知偏移处已有`INTR_MASK`寄存器；驱动向其写入全1以禁用每个源。

```c
static void
myfirst_mask_interrupts(struct myfirst_softc *sc)
{
        MYFIRST_ASSERT_UNLOCKED(sc);

        /*
         * 在设备处禁用所有中断源。此写入后，
         * 硬件不会发出任何中断向量。任何已-
         * 待处理的状态位保留，但过滤器不会被调用
         * 去注意它们。
         */
        CSR_WRITE_4(sc, MYFIRST_REG_INTR_MASK, 0xFFFFFFFF);

        /*
         * 对于真实硬件：还要清除任何待处理状态位，以便我们
         * 在恢复时不看到陈旧中断。
         */
        CSR_WRITE_4(sc, MYFIRST_REG_INTR_STATUS, 0xFFFFFFFF);
}
```

该函数不持有softc锁。它仅通过`CSR_WRITE_4`与设备对话，这是不需要任何特定锁纪律的`bus_write_4`包装器。`MYFIRST_ASSERT_UNLOCKED`调用是启用WITNESS的不变量，捕捉错误持有锁调用此函数的调用者；它廉价且有用。

掩码值`0xFFFFFFFF`假设模拟的INTR_MASK寄存器使用1表示已掩码的语义。第19章模拟使用该约定；真实驱动应查阅其设备数据表。`myfirst`的寄存器映射记录在`INTERRUPTS.md`；读者可以在此复查约定。

微妙之处：在某些真实设备上，掩码中断仅阻止设备发出新中断；当前活动的中断继续发出，直到驱动通过状态寄存器确认。这就是为什么函数还清除INTR_STATUS：确保不留有任何可能恢复后再次触发的陈旧位。在模拟上，状态寄存器行为类似，所以同样的写入是正确的。

### 实现myfirst_drain_dma

排空DMA是三个助手中最微妙的，因为它必须等待设备。第21章驱动用`dma_in_flight`跟踪进行中的DMA并通过`dma_cv`通知完成。挂起路径复用完全相同的机制。

```c
static int
myfirst_drain_dma(struct myfirst_softc *sc)
{
        int err = 0;

        MYFIRST_LOCK(sc);
        if (sc->dma_in_flight) {
                /*
                 * 告诉引擎中止。中止位产生一个
                 * ERR状态，过滤器将其转换为通过
                 * cv_broadcast唤醒我们的任务。
                 */
                CSR_WRITE_4(sc, MYFIRST_REG_DMA_CTRL,
                    MYFIRST_DMA_CTRL_ABORT);

                /*
                 * 等待最多一秒让中止落地。
                 * cv_timedwait在我们睡眠时释放锁。
                 */
                err = cv_timedwait(&sc->dma_cv, &sc->mtx, hz);
                if (err == EWOULDBLOCK) {
                        device_printf(sc->dev,
                            "drain_dma: timeout waiting for abort\n");
                        /*
                         * 强制状态前进。此时设备
                         * 已超出我们可达范围；将
                         * 传输视为失败。
                         */
                        sc->dma_in_flight = false;
                }
        }
        MYFIRST_UNLOCK(sc);

        return (err == EWOULDBLOCK ? ETIMEDOUT : 0);
}
```

该函数请求DMA引擎中止，然后在第21章完成路径使用的条件变量上睡眠。如果完成到达，过滤器确认它并排队任务；任务调用`myfirst_dma_handle_complete`，它执行POSTREAD/POSTWRITE同步、清除`dma_in_flight`并广播CV。挂起路径的`cv_timedwait`返回，排空函数返回0，挂起继续。

如果完成在一秒内未到达（对于callout每几毫秒触发的模拟设备，一秒是宽裕超时），函数记录警告并强制`dma_in_flight`为false。这是防御性选择：在一秒内不响应中止的真实设备行为异常，驱动必须继续。将`dma_in_flight`保持为true会死锁挂起。防御性清除的成本是，非常慢的传输原则上可能在挂起返回后完成，写入驱动不再期望活跃的缓冲区。在模拟上，这不会发生因为callout在下一步排空。在真实硬件上，风险是硬件特定的，真实驱动会在此添加设备特定恢复。

返回值在干净排空时为0（包括已强制清除的超时情况），如果调用者需要知道超时发生了则为`ETIMEDOUT`。挂起路径记录错误但不否决挂起；到排空已超时时，设备已实际损坏。

### 实现myfirst_drain_workers

排空延迟工作更容易，因为不涉及设备。rx任务和模拟的callout都有已知的排空原语。

```c
static void
myfirst_drain_workers(struct myfirst_softc *sc)
{
        /*
         * 排空每向量rx任务。任何正在运行的任务被
         * 允许完成；不会启动新任务因为
         * 中断已掩码。
         */
        if (sc->rx_vector.has_task)
                taskqueue_drain(taskqueue_thread, &sc->rx_vector.task);

        /*
         * 排空模拟的DMA callout。任何进行中的触发被
         * 允许完成；不会发生新触发。
         *
         * 这是仅模拟的调用；真实硬件驱动会
         * 省略它。
         */
        if (sc->sim != NULL)
                myfirst_sim_drain_dma_callout(sc->sim);
}
```

该函数在释放锁的情况下安全调用。`taskqueue_drain`有文档说明自己做同步；`callout_drain`（`myfirst_sim_drain_dma_callout`内部包装）类似安全。

两个排空调用的重要属性：它们等待正在运行的工作完成，但不中途取消它。在`myfirst_dma_handle_complete`中途的任务将完成其工作，包括任何`bus_dmamap_sync`和计数器更新，在排空返回之前。这是我们要的行为：挂起不应中途打断任务，因为任务的不变量必须成立恢复路径才能正确。

### 更新挂起方法

有了三个助手，挂起方法很短：

```c
static int
myfirst_pci_suspend(device_t dev)
{
        struct myfirst_softc *sc = device_get_softc(dev);
        int err;

        device_printf(dev, "suspend: starting\n");

        err = myfirst_quiesce(sc);
        if (err != 0) {
                device_printf(dev,
                    "suspend: quiesce returned %d; continuing anyway\n",
                    err);
        }

        atomic_add_64(&sc->power_suspend_count, 1);
        device_printf(dev,
            "suspend: complete (dma in flight=%d, suspended=%d)\n",
            sc->dma_in_flight, sc->suspended);
        return (0);
}
```

挂起方法不将静默错误返回给内核。这是策略决策，值得解释。

从`DEVICE_SUSPEND`返回非零值会否决挂起，这有大的下游影响：内核展开部分挂起，向用户报告失败，使系统保持在S0。对于第22章的驱动，静默超时不值得那种程度的干扰。设备仍可访问；掩码中断和标记挂起标志足以防止任何活动。有未完成写入的存储控制器可能会在写入完成前否决挂起，因为丢失该写入会在转换期间损坏文件系统。每个驱动做出自己的决定。

此阶段的日志很详细。第7节将介绍通过sysctl关闭详细日志（或为调试调高）的能力。目前，额外细节有助于首次逐步执行挂起序列时使用。

### 保存硬件丢失的运行时状态

目前挂起路径仅停止活动。它未保存任何状态。对于大多数设备这是正确的：PCI层自动保存配置空间，设备的运行时寄存器（驱动通过BAR写入的那些）在恢复时要么从驱动的softc恢复要么从驱动的软件状态重新生成。`myfirst`模拟没有驱动关心保存的BAR局部状态；模拟在恢复时重新开始，驱动在那时写入需要的任何寄存器。

真实驱动可能更多。考虑`re(4)`驱动：其`re_setwol`函数在挂起前将与网络唤醒相关的寄存器写入NIC的EEPROM支持配置空间。这些值是设备私有的；PCI层不知道它们。如果驱动未在挂起时写入它们，NIC将不知道应该在魔法数据包上唤醒，网络唤醒将不工作。

对于第22章，`myfirst`驱动保存的唯一状态是中断掩码的挂起前值。第2阶段挂起向掩码寄存器写入`0xFFFFFFFF`，但第2阶段恢复需要知道之前的值（决定正常运行时启用哪些向量）。驱动将其存储在softc字段中：

```c
struct myfirst_softc {
        /* ... */
        uint32_t saved_intr_mask;
};
```

掩码助手保存它：

```c
static void
myfirst_mask_interrupts(struct myfirst_softc *sc)
{
        sc->saved_intr_mask = CSR_READ_4(sc, MYFIRST_REG_INTR_MASK);

        CSR_WRITE_4(sc, MYFIRST_REG_INTR_MASK, 0xFFFFFFFF);
        CSR_WRITE_4(sc, MYFIRST_REG_INTR_STATUS, 0xFFFFFFFF);
}
```

恢复路径将在设备重新初始化后将`sc->saved_intr_mask`写回掩码寄存器。这是状态保存恢复的最小示例；第4节详细展示完整恢复流程时将详细说明。

### suspended标志作为用户面向的不变量

在静默期间设置`sc->suspended = true`有第二个目的，除了抑制新请求：它使状态对用户空间可观察。驱动可以通过sysctl暴露该标志：

```c
SYSCTL_ADD_BOOL(ctx, kids, OID_AUTO, "suspended",
    CTLFLAG_RD, &sc->suspended, 0,
    "驱动是否处于挂起状态");
```

`devctl suspend myfirst0`之后，读者看到：

```sh
# sysctl dev.myfirst.0.suspended
dev.myfirst.0.suspended: 1
```

`devctl resume myfirst0`之后，值应回0（第4节将恢复路径连接以清除它）。这是无需从其他计数器推断就能检查驱动状态的快速方式。

### 处理DMA未进行中的情况

`myfirst_drain_dma`助手处理传输活跃运行的情况。它还应处理更常见的在挂起时刻没有进行中的情况，而不做任何不必要的事情。

上面的伪代码确实处理了这种情况：`if (sc->dma_in_flight)`守卫在标志为false时完全跳过中止和等待。函数立即返回0，挂起继续。

该路径很快：在空闲设备上，`myfirst_drain_dma`是锁获取、标志检查和锁释放。静默的成本由`taskqueue_drain`（通过任务队列线程做完整往返）和`callout_drain`（类似）主导。典型的空闲设备挂起花费几百微秒到几毫秒，由延迟工作排空主导，而非设备。

### 测试第2阶段

静默代码就位后，第2阶段测试更有趣。读者运行传输，然后立即挂起，观察：

```sh
# 从无活动开始。
sysctl dev.myfirst.0.dma_transfers_read
# 0

# 触发传输。传输应快速完成。
sudo sysctl dev.myfirst.0.dma_test_read=1
sysctl dev.myfirst.0.dma_transfers_read
# 1

# 现在压力测试路径：启动传输并立即挂起。
sudo sysctl dev.myfirst.0.dma_test_read=1 &
sudo devctl suspend myfirst0

# 检查状态。
sysctl dev.myfirst.0.suspended
# 1

sysctl dev.myfirst.0.power_suspend_count
# 1

sysctl dev.myfirst.0.dma_in_flight
# 0（传输已完成或已中止）

dmesg | tail -8
```

预期的`dmesg`输出显示挂起日志、进行中的DMA被中止以及完成。如果挂起返回后`dma_in_flight`仍为1，中止未生效，读者应检查模拟的中止处理。

然后恢复：

```sh
sudo devctl resume myfirst0

sysctl dev.myfirst.0.suspended
# 0（第4节实现后）

sysctl dev.myfirst.0.power_resume_count
# 1

# 尝试另一次传输检查设备已回来。
sudo sysctl dev.myfirst.0.dma_test_read=1
dmesg | tail -4
```

最后一次传输应成功；如果不成功，挂起将设备留在恢复未恢复的状态。第4节教授使这正确的恢复路径。

### 关于锁定的谨慎说明

静默代码从`DEVICE_SUSPEND`方法运行，该方法由内核的电源管理路径调用。该路径调用方法时不持有驱动锁；驱动负责自己的同步。本节的助手遵循特定纪律：

- `myfirst_mask_interrupts`不持有锁。它仅写入硬件寄存器，这在PCIe上是原子的。
- `myfirst_drain_dma`获取softc锁以读取`dma_in_flight`并使用`cv_timedwait`在持有锁时睡眠（这是睡眠互斥CV的正确使用）。
- `myfirst_drain_workers`不持有锁。`taskqueue_drain`和`callout_drain`做自己的同步，必须在不持有锁的情况下调用以避免死锁（被排空任务可能尝试获取同一个锁）。

完整静默序列因此多次获取释放锁：在`myfirst_quiesce`顶部短暂一次，在`myfirst_drain_dma`内部睡眠时一次，从不在`myfirst_drain_workers`内部。这是有意为之。在`taskqueue_drain`上持有锁会死锁，因为被排空任务在入口获取同一个锁。

在`WITNESS`下运行此代码的读者不会看到任何锁顺序警告，因为锁仅在CV睡眠窗口内持有，该窗口期间不获取其他锁。如果后续工作向驱动添加更多锁（例如，每向量锁），静默代码应继续注意在排空调用周围持有哪些锁。

### 与关机方法集成

关机方法与挂起共享几乎所有逻辑。合理实现是：

```c
static int
myfirst_pci_shutdown(device_t dev)
{
        struct myfirst_softc *sc = device_get_softc(dev);

        device_printf(dev, "shutdown: starting\n");
        (void)myfirst_quiesce(sc);
        atomic_add_64(&sc->power_shutdown_count, 1);
        device_printf(dev, "shutdown: complete\n");
        return (0);
}
```

与挂起的唯一区别是没有状态保存调用（关机是最终的；没有恢复需要保存状态）和没有静默返回值检查（关机无法有意义否决）。许多真实驱动遵循相同模式；`/usr/src/sys/dev/xl/if_xl.c`的`xl_shutdown`只是调用`xl_suspend`。`myfirst`驱动可用任一风格；本章偏好上面稍显显式的版本，因为它在代码中更清晰地表达意图。

### 第3节收尾

第3节将第1阶段骨架变为在挂起时真正静默设备的第2阶段驱动。它引入了静默纪律（标记挂起、掩码中断、停止DMA、排空工作者、验证），将步骤提取为三个助手函数（`myfirst_mask_interrupts`、`myfirst_drain_dma`、`myfirst_drain_workers`），解释它们必须运行的顺序及原因，展示如何将它们集成到挂起和关机方法中，并讨论锁定纪律。

第2阶段驱动尚不做的是正确回来。恢复方法仍是第1阶段骨架；它记录日志并返回0而不恢复任何东西。如果挂起的设备有任何硬件丢失的状态，该状态已消失，后续传输将失败。第4节修复恢复路径使完整挂起-恢复周期将设备留在之前相同的状态。

## Section 4: 恢复时恢复状态

Section 3 gave the driver a correct suspend path. Section 4 writes the matching resume. The resume is the complement of the suspend: every thing suspend stopped, resume restarts; every value suspend saved, resume writes back; every flag suspend set, resume clears. The sequence is not an exact mirror (resume runs in a different kernel context, with the PCI layer having already done work, and the device in a different state than where suspend left it), but the contents correspond one-to-one. Doing the resume correctly is a matter of respecting the contract the PCI layer has already partly fulfilled, and filling in the rest.

### What the PCI Layer Has Already Done

When the kernel's `DEVICE_RESUME` method is called on the driver, several things have already happened:

1. The CPU has come out of the S-state (resumed from S3 or S4 back to S0).
2. Memory has been refreshed and the kernel has re-established its own state.
3. The parent bus has been resumed. For `myfirst`, that means the PCI bus driver has already handled the host bridge and the PCIe root complex.
4. The PCI layer has called `pci_set_powerstate(dev, PCI_POWERSTATE_D0)` on the device, transitioning it from whatever low-power state it was in (typically D3hot) back to full power.
5. The PCI layer has called `pci_cfg_restore(dev, dinfo)`, which writes the cached configuration space values (BARs, command register, cache-line size, etc.) back into the device.
6. The PCI layer has called `pci_clear_pme(dev)` to clear any pending power-management event bits.
7. The MSI or MSI-X configuration, which is part of the cached state, has been restored. The driver's interrupt vectors are usable again.

At this point the PCI bus driver calls into `myfirst`'s `DEVICE_RESUME`. The device is in D0, with its BARs mapped, its MSI/MSI-X table restored, and its generic PCI state intact. What the driver has to restore is the device-specific state that the PCI layer did not know about: the BAR-local registers the driver wrote during or after attach.

For the `myfirst` simulation, the relevant BAR-local registers are the interrupt mask (which the suspend path deliberately set to all-masked) and the DMA registers (which may have been left in an aborted state). The driver needs to put them back to values that reflect normal operation.

### The Resume Discipline

A correct resume path does four things, in order:

1. **Re-enable bus-mastering**, in case the configuration-space restore did not do so or the PCI layer's automatic restore was disabled. This is `pci_enable_busmaster(dev)`. On modern FreeBSD it is usually redundant but harmless; older code paths or buggy BIOSes sometimes leave bus-mastering disabled. Calling it defensively is cheap.

2. **Restore any device-specific state** the driver saved during suspend. For `myfirst`, that means writing `saved_intr_mask` back to the INTR_MASK register. A real driver would also restore things like vendor-specific configuration bits, DMA engine programming, hardware timers, etc.

3. **Unmask interrupts and clear the suspended flag**, so the device can resume activity. This is the pivot point: before it, the device is still quiet; after it, the device can raise interrupts and accept work.

4. **Log the transition and update counters**, for observability and regression testing.

Here is what the pattern looks like in code:

```c
static int
myfirst_pci_resume(device_t dev)
{
        struct myfirst_softc *sc = device_get_softc(dev);
        int err;

        device_printf(dev, "resume: starting\n");

        err = myfirst_restore(sc);
        if (err != 0) {
                device_printf(dev,
                    "resume: restore failed (err %d)\n", err);
                atomic_add_64(&sc->power_resume_errors, 1);
                /*
                 * Continue anyway. By the time we're here, the system
                 * is coming back whether we like it or not.
                 */
        }

        atomic_add_64(&sc->power_resume_count, 1);
        device_printf(dev, "resume: complete\n");
        return (0);
}
```

The helper `myfirst_restore` does the three real steps:

```c
static int
myfirst_restore(struct myfirst_softc *sc)
{
        /* Step 1: re-enable bus-master (defensive). */
        pci_enable_busmaster(sc->dev);

        /* Step 2: restore device-specific state.
         *
         * For myfirst, this is just the interrupt mask. A real driver
         * would restore more: DMA engine programming, hardware timers,
         * vendor-specific configuration, etc.
         */
        if (sc->saved_intr_mask == 0xFFFFFFFF) {
                /*
                 * Suspend saved a fully-masked mask, which means the
                 * driver had no idea what the mask should be. Use the
                 * default: enable DMA completion, disable everything
                 * else.
                 */
                sc->saved_intr_mask = ~MYFIRST_INTR_COMPLETE;
        }
        CSR_WRITE_4(sc->dev, MYFIRST_REG_INTR_MASK, sc->saved_intr_mask);

        /* Step 3: clear the suspended flag and unmask the device. */
        MYFIRST_LOCK(sc);
        sc->suspended = false;
        MYFIRST_UNLOCK(sc);

        return (0);
}
```

The function returns 0 because no step above can fail in the `myfirst` simulation. A real driver would check the return values of its hardware initialisation calls and propagate any errors.

### Why pci_enable_busmaster Matters

Bus-mastering is a bit in the PCI command register that controls whether the device can issue DMA transactions. Without it, the device cannot read or write host memory; any DMA trigger would be silently ignored by the PCI host bridge.

Chapter 18 enabled bus-mastering during attach. The PCI layer's automatic config-space restore writes the command register back to its saved value, which includes the bus-master bit. So in principle the driver does not need to call `pci_enable_busmaster` again on resume. In practice, several things can go wrong:

- The platform firmware may reset the command register as part of waking the device.
- The `hw.pci.do_power_suspend` sysctl may be 0, in which case the PCI layer does not save and restore the config space.
- A device-specific quirk might clear bus-mastering as a side effect of the D3-to-D0 transition.

Calling `pci_enable_busmaster` unconditionally defensively in resume is a low-cost safety net. Several production FreeBSD drivers follow this pattern; `if_re.c`'s resume path is one example. The call is idempotent: if bus-mastering is already on, the call just re-asserts it.

### Restoring Device-Specific State

The `myfirst` simulation does not have much state the driver needs to restore manually. The BAR-local registers are:

- The interrupt mask (restored from `saved_intr_mask`).
- The interrupt status bits (were cleared in suspend; they should stay cleared until new activity arrives).
- The DMA engine registers (DMA_ADDR_LOW, DMA_ADDR_HIGH, DMA_LEN, DMA_DIR, DMA_CTRL, DMA_STATUS). These are transient: they hold the parameters of the current transfer. After resume, no transfer is in progress, so the values do not matter; the next transfer will overwrite them.

A real driver would have more. Consider a few examples:

- A storage driver might have a DMA descriptor ring whose base address the device learned during attach. After resume, the BAR-level register holding that base address may have been reset; the driver needs to reprogram it.
- A network driver might have filter tables (MAC addresses, multicast lists, VLAN tags) programmed into device registers. After resume, those tables may be empty; the driver rebuilds them from softc-side copies.
- A GPU driver might have register state for display timing, colour tables, hardware cursors. After resume, the driver restores the active mode.

For `myfirst`, the interrupt mask is the only BAR-local state that needs restoring. The pattern shown above is the template a real driver would adapt to its device.

### Validating Device Identity After Resume

Some devices are reset completely across a suspend-to-D3-cold cycle. The device that comes back is functionally the same, but its entire state has been reinitialised as if it had just powered on. A driver that assumed nothing changed would silently get wrong behaviour.

A defensive resume path can detect this by reading a known register value and comparing to what it read at attach time. For a PCI device, the vendor ID and device ID in configuration space are always the same (the PCI layer restored them), but some device-private register (a revision ID, a self-test register, a firmware version) can be checked:

```c
static int
myfirst_validate_device(struct myfirst_softc *sc)
{
        uint32_t magic;

        magic = CSR_READ_4(sc->dev, MYFIRST_REG_MAGIC);
        if (magic != MYFIRST_MAGIC_VALUE) {
                device_printf(sc->dev,
                    "resume: device identity mismatch (got %#x, "
                    "expected %#x)\n", magic, MYFIRST_MAGIC_VALUE);
                return (EIO);
        }
        return (0);
}
```

For the `myfirst` simulation, there is no magic register (the simulation was not built with post-resume validation in mind). A reader who wants to add one as a challenge can extend the simulation backend's register map with a read-only `MAGIC` register, and have the driver check it. The chapter's Lab 3 includes this as an option.

A real driver whose device truly does reset across D3cold needs this check, because without it a subtle failure can occur: the driver assumes the device's internal state machine is in state `IDLE`, but after the reset the state machine is actually in state `RESETTING`. Any command the driver sends is rejected, the driver interprets the rejection as a hardware fault, and the device is marked broken. Catching the reset explicitly and rebuilding state is cleaner.

### Detecting and Recovering from a Device Reset

If the validation finds a mismatch, the driver's recovery options depend on the hardware. For the `myfirst` simulation, the simplest response is to log, mark the device broken, and fail subsequent operations:

```c
if (myfirst_validate_device(sc) != 0) {
        MYFIRST_LOCK(sc);
        sc->broken = true;
        MYFIRST_UNLOCK(sc);
        return (EIO);
}
```

The softc grows a `broken` flag, and any user-facing request checks the flag and fails with an error. The detach path still works (detach always succeeds, even on a broken device), so the user can unload the driver and reload it.

A real driver that detects a reset has more options. A network driver might re-run its attach sequence from the point after `pci_alloc_msi` (which has been restored by the PCI layer). A storage driver might re-initialise its controller using the same code path attach used. The implementation depends heavily on the device; the pattern is "detect, then do whatever attach-time initialisation is still required".

The chapter's `myfirst` driver takes the simpler approach: it does not implement reset detection for the simulation, and the resume path does not include the validation call by default. The code above is provided as reference for a reader who wants to extend the driver as an exercise.

### Restoring DMA State

The Chapter 21 DMA setup allocates a tag, allocates memory, loads the map, and retains the bus address in the softc. None of that is visible in the BAR-local register map; the DMA engine learns the bus address only when the driver writes it to `DMA_ADDR_LOW` and `DMA_ADDR_HIGH` as part of starting a transfer.

This means the DMA state does not need restoration in the sense of "write registers". The tag, map, and memory are all kernel-side data structures; they survive suspend intact. The next transfer will program the DMA registers as part of its normal submission.

What might need restoration on a real device is:

- **The DMA descriptor ring base address**, if the device keeps a persistent pointer. A real NIC writes a base-address register once at attach and points the device at a ring of descriptors; after D3cold, that register may have been reset and the driver must reprogram it.
- **The DMA engine's enable bit**, if it is separate from individual transfers.
- **Any per-channel configuration** (burst size, priority, etc.) that is held in registers the PCI layer did not cache.

For `myfirst`, none of this applies. The DMA engine is programmed per transfer. Resume does not need any DMA-specific restoration beyond what the generic state restoration already covered.

### Re-Arming Interrupts

Masking interrupts was step 2 of suspend. Unmasking them is step 3 of resume. The Stage 3 resume writes `saved_intr_mask` back to the `INTR_MASK` register, which (by convention) writes 0 to the bits corresponding to enabled vectors and 1 to the bits for disabled vectors. After the write, the device is ready to assert interrupts on the enabled vectors as soon as there is reason to.

There is a subtlety around ordering. The resume path unmasks interrupts before it clears the `suspended` flag. That means a very unfortunate interrupt could arrive, call the filter, and find `suspended == true`. The filter would refuse to handle it and return `FILTER_STRAY`, which would leave the interrupt asserted.

To avoid that, the resume path takes the softc lock around the state change and does the unmask and the flag clear in the opposite order: clear `suspended` first, then unmask. That way any interrupt the device raises after the mask clears sees `suspended == false` and is handled normally.

The code in the previous snippet does this correctly: `myfirst_restore` writes the mask, then acquires the lock, clears the flag, and releases the lock. The order is important; reversing it creates a narrow window where interrupts could be lost.

### Wake Source Cleanup

If the driver enabled a wake source during suspend (`pci_enable_pme`), the resume path should clear any pending wake event (`pci_clear_pme`). The PCI layer's `pci_resume_child` helper already calls `pci_clear_pme(child)` before the driver's `DEVICE_RESUME`, so the driver does not usually need to call it again.

The one case where the driver might want to call `pci_clear_pme` explicitly is in a runtime-PM context where the driver is resuming the device while the system stays in S0. In that case `pci_resume_child` was not involved, and the driver is responsible for clearing the PME status itself.

A hypothetical sketch for a driver with wake-on-X:

```c
static int
myfirst_pci_resume(device_t dev)
{
        struct myfirst_softc *sc = device_get_softc(dev);

        if (pci_has_pm(dev))
                pci_clear_pme(dev);  /* defensive; PCI layer already did this */

        /* ... rest of the resume path ... */
}
```

For `myfirst`, there is no wake source, so the call does nothing useful; the chapter omits it from the main code and mentions the pattern here for completeness.

### Updating the Stage 3 Driver

Stage 3 brings together everything above into a single working resume. The diff against Stage 2 is:

- `myfirst.h` grows a `saved_intr_mask` field (added for Stage 2) and a `broken` flag.
- `myfirst_pci.c` gets a `myfirst_restore` helper and a rewritten `myfirst_pci_resume`.
- The Makefile version bumps to `1.5-power-stage3`.

Build and test:

```sh
cd /path/to/driver
make clean && make
sudo kldunload myfirst     # unload any previous version
sudo kldload ./myfirst.ko

# Quiet baseline.
sysctl dev.myfirst.0.dma_transfers_read
# 0
sysctl dev.myfirst.0.suspended
# 0

# Full cycle.
sudo devctl suspend myfirst0
sysctl dev.myfirst.0.suspended
# 1

sudo devctl resume myfirst0
sysctl dev.myfirst.0.suspended
# 0

# A transfer after resume should work.
sudo sysctl dev.myfirst.0.dma_test_read=1
sysctl dev.myfirst.0.dma_transfers_read
# 1

# Do it several times to make sure the path is stable.
for i in 1 2 3 4 5; do
  sudo devctl suspend myfirst0
  sudo devctl resume myfirst0
  sudo sysctl dev.myfirst.0.dma_test_read=1
done
sysctl dev.myfirst.0.dma_transfers_read
# 6 (1 + 5)

sysctl dev.myfirst.0.power_suspend_count dev.myfirst.0.power_resume_count
# should be equal, around 6 each
```

If the counters drift (suspend count not equal to resume count) or if `dma_test_read` starts failing after a suspend, something in the restore path is not putting the device back into a usable state. The first debugging step is to read the INTR_MASK and compare against `saved_intr_mask`; the second is to trace the DMA engine's status register and see if it is reporting an error.

### Interaction with the Chapter 20 MSI-X Setup

The `myfirst` driver from Chapter 20 uses MSI-X when available, with a three-vector layout (admin, rx, tx). The MSI-X configuration lives in the device's MSI-X capability registers and in a kernel-side table. The PCI layer's config-space save-and-restore covers the capability registers; the kernel-side state is not affected by the D-state transition.

This means the `myfirst` driver does not need to do anything special to restore its MSI-X vectors. The interrupt resources (`irq_res`) remain allocated, the cookies remain registered, the CPU bindings remain in place. When the device raises an MSI-X vector on resume, the kernel delivers it to the filter that was registered at attach time.

A reader who wants to verify this can write to one of the simulate sysctls after resume and observe that the corresponding per-vector counter increments:

```sh
sudo devctl suspend myfirst0
sudo devctl resume myfirst0
sudo sysctl dev.myfirst.0.intr_simulate_admin=1
sysctl dev.myfirst.0.vec0_fire_count
# should be incremented
```

If the counter does not increment, the MSI-X path has been disturbed. The most likely cause is a bug in the driver's own state management (the `suspended` flag was not cleared, or the filter is rejecting the interrupt for a different reason). The chapter's troubleshooting section has more detail.

### Handling a Failed Resume Gracefully

If some step of the resume fails, the driver has limited options. It cannot veto the resume (the kernel has no unwind path at this point). It cannot usually retry (the hardware state is uncertain). The best it can do is:

1. Log the failure prominently with `device_printf` so the user sees it in dmesg.
2. Increment a counter (`power_resume_errors`) that a regression script or an observability tool can check.
3. Mark the device broken so that subsequent requests fail cleanly rather than silently corrupting data.
4. Keep the driver attached, so the device-tree state stays consistent and the user can eventually unload and reload the driver.
5. Return 0 from `DEVICE_RESUME`, because the kernel expects it to succeed.

The "mark broken, keep attached" pattern is common in production drivers. It moves the failure from "mysterious later corruption" to "immediate user-visible error", which is a better debugging experience.

### A Short Detour: pci_save_state / pci_restore_state in Runtime PM

Section 2 mentioned that `pci_save_state` and `pci_restore_state` are sometimes called by the driver itself, typically in a runtime power-management helper. This is worth a concrete sketch before Section 5 builds it out.

A runtime PM helper that puts an idle device into D3 looks like:

```c
static int
myfirst_runtime_suspend(struct myfirst_softc *sc)
{
        int err;

        err = myfirst_quiesce(sc);
        if (err != 0)
                return (err);

        pci_save_state(sc->dev);
        err = pci_set_powerstate(sc->dev, PCI_POWERSTATE_D3);
        if (err != 0) {
                /* roll back */
                pci_restore_state(sc->dev);
                myfirst_restore(sc);
                return (err);
        }

        return (0);
}

static int
myfirst_runtime_resume(struct myfirst_softc *sc)
{
        int err;

        err = pci_set_powerstate(sc->dev, PCI_POWERSTATE_D0);
        if (err != 0)
                return (err);
        pci_restore_state(sc->dev);

        return (myfirst_restore(sc));
}
```

The pattern is similar to the system suspend/resume but uses the explicit PCI helpers because the PCI layer is not in the loop. Section 5 will turn this sketch into a real implementation and wire it to an idle-detection policy.

### A Reality Check Against a Real Driver

Before moving on, it is worth pausing and looking at a real driver's resume path. `/usr/src/sys/dev/re/if_re.c`'s `re_resume` function is about thirty lines. Its structure is:

1. Lock the softc.
2. If a MAC-sleep flag is set, take the chip out of sleep mode by writing a GPIO register.
3. Clear any wake-on-LAN patterns so normal receive filtering is not interfered with.
4. If the interface is administratively up, re-initialise it via `re_init_locked`.
5. Clear the `suspended` flag.
6. Unlock the softc.
7. Return 0.

The `re_init_locked` call is the substantive work: it reprograms the MAC address, resets the receive and transmit descriptor rings, re-enables interrupts on the NIC, and starts the DMA engines. For `myfirst`, the equivalent work is much shorter because the device is much simpler, but the shape is the same: acquire state, do hardware-specific reinitialisation, unlock, return.

A reader who reads `re_resume` after implementing `myfirst`'s resume will recognise the structure immediately. The vocabulary is the same; only the details differ.

### Wrapping Up Section 4

Section 4 completed the resume path. It showed what the PCI layer has already done by the time `DEVICE_RESUME` is called (D0 transition, config-space restore, PME# clear, MSI-X restore), what the driver still has to do (re-enable bus-master, restore device-specific registers, clear the suspended flag, unmask interrupts), and why each step is important. The Stage 3 driver can now do a full suspend-resume cycle and continue operating normally; the regression test can run several cycles in a row and verify the counters are consistent.

With Sections 3 and 4 together, the driver is power-aware in the system-suspend sense: it handles S3 and S4 transitions cleanly. What it still does not do is any device-level power saving while the system is running. That is runtime power management, and Section 5 teaches it.



## Section 5: 处理运行时电源管理

System suspend is a big, visible transition: the lid closes, the screen goes dark, the battery saves power for hours. Runtime power management is the opposite: dozens of small, invisible transitions a second, each saving a little, together saving much of the idle power a modern system draws. The user never notices them; the platform engineer lives or dies by their correctness.

This section is marked optional in the chapter outline because not every driver needs runtime PM. A driver for a device that is always active (a NIC on a busy server, a disk controller for the root filesystem) does not save power by attempting to suspend its device; the device is busy, and trying to suspend it wastes cycles setting up transitions that never complete. A driver for a device that is frequently idle (a webcam, a fingerprint reader, a WLAN card on a laptop) does benefit. Whether to add runtime PM is a policy decision driven by the device's usage profile.

For Chapter 22, we implement runtime PM on the `myfirst` driver as a learning exercise. The device is already simulated; we can pretend it is idle whenever no sysctl has been written in the last few seconds, and watch the driver go through the motions. The implementation is short, and it teaches the PCI-level primitives that a real runtime-PM driver uses.

### What Runtime PM Means in FreeBSD

FreeBSD does not currently have a centralised runtime-PM framework the way Linux does. There is no kernel-side "if the device has been idle for N milliseconds, call its idle hook" machinery. Instead, runtime PM is driver-local: the driver decides when to suspend and resume its device, using the same PCI-layer primitives (`pci_set_powerstate`, `pci_save_state`, `pci_restore_state`) it would use inside `DEVICE_SUSPEND` and `DEVICE_RESUME`.

This has two consequences. First, every driver that wants runtime PM implements its own policy: how long the device must be idle before suspending, what counts as idle, how quickly the device must wake on demand. Second, the driver must integrate its runtime PM with its system PM; the two paths share a lot of code and must not step on each other.

The pattern Chapter 22 uses is straightforward:

1. The driver adds a small state machine with states `RUNNING` and `RUNTIME_SUSPENDED`.
2. When the driver observes idleness (Section 5 uses a callout-based "no requests in the last 5 seconds" policy), it calls `myfirst_runtime_suspend`.
3. When the driver observes a new request while in `RUNTIME_SUSPENDED`, it calls `myfirst_runtime_resume` before processing the request.
4. On system suspend, if the device is in `RUNTIME_SUSPENDED`, the system-suspend path adjusts for it (the device is already quiesced; the system-suspend quiesce is a no-op, but the system resume has to bring the device back to D0).
5. On system resume, the driver returns to `RUNNING` unless it was explicitly runtime-suspended and wants to stay that way.

This is simpler than Linux's runtime PM framework, which has richer concepts (parent/child ref-counting, autosuspend timers, barriers). For a single driver on simple hardware, the FreeBSD approach is enough.

### 运行时PM状态机

The softc gains a state variable and a timestamp:

```c
enum myfirst_runtime_state {
        MYFIRST_RT_RUNNING = 0,
        MYFIRST_RT_SUSPENDED = 1,
};

struct myfirst_softc {
        /* ... */
        enum myfirst_runtime_state runtime_state;
        struct timeval             last_activity;
        struct callout             idle_watcher;
        int                        idle_threshold_seconds;
        uint64_t                   runtime_suspend_count;
        uint64_t                   runtime_resume_count;
};
```

The `idle_threshold_seconds` is a policy knob exposed through a sysctl; defaulting to five seconds gives quick observability without being so aggressive as to cause unnecessary wake-ups during normal use. A production driver would tune this per-device; five seconds is a learning-friendly value that makes the transitions visible without requiring hours of waiting.

The `idle_watcher` callout fires once a second to check the idle time. If the device has been idle longer than `idle_threshold_seconds` and is currently in `RUNNING`, the callout triggers `myfirst_runtime_suspend`.

### Implementation

The attach path starts the idle watcher:

```c
static void
myfirst_start_idle_watcher(struct myfirst_softc *sc)
{
        sc->idle_threshold_seconds = 5;
        microtime(&sc->last_activity);
        callout_init_mtx(&sc->idle_watcher, &sc->mtx, 0);
        callout_reset(&sc->idle_watcher, hz, myfirst_idle_watcher_cb, sc);
}
```

The callout is initialised with the softc mutex, so it acquires the mutex automatically when firing. That simplifies the callback: it runs under the lock.

The callback checks the time since the last activity and suspends if needed:

```c
static void
myfirst_idle_watcher_cb(void *arg)
{
        struct myfirst_softc *sc = arg;
        struct timeval now, diff;

        MYFIRST_ASSERT_LOCKED(sc);

        if (sc->runtime_state == MYFIRST_RT_RUNNING && !sc->suspended) {
                microtime(&now);
                timersub(&now, &sc->last_activity, &diff);

                if (diff.tv_sec >= sc->idle_threshold_seconds) {
                        /*
                         * Release the lock while suspending. The
                         * runtime_suspend helper acquires it again as
                         * needed.
                         */
                        MYFIRST_UNLOCK(sc);
                        (void)myfirst_runtime_suspend(sc);
                        MYFIRST_LOCK(sc);
                }
        }

        /* Reschedule. */
        callout_reset(&sc->idle_watcher, hz, myfirst_idle_watcher_cb, sc);
}
```

Note the lock-drop around `myfirst_runtime_suspend`. The suspend helper calls `myfirst_quiesce`, which acquires the lock itself. Holding the lock across it would deadlock.

Activity is recorded whenever the driver services a request. The Chapter 21 DMA path is a good hook: every time a user writes to `dma_test_read` or `dma_test_write`, the sysctl handler records activity:

```c
static int
myfirst_dma_sysctl_test_write(SYSCTL_HANDLER_ARGS)
{
        struct myfirst_softc *sc = arg1;
        /* ... existing code ... */

        /* Mark the device active before processing. */
        myfirst_mark_active(sc);

        /* If runtime-suspended, bring the device back before running. */
        if (sc->runtime_state == MYFIRST_RT_SUSPENDED) {
                int err = myfirst_runtime_resume(sc);
                if (err != 0)
                        return (err);
        }

        /* ... proceed with the transfer ... */
}
```

The `myfirst_mark_active` helper is a one-liner:

```c
static void
myfirst_mark_active(struct myfirst_softc *sc)
{
        MYFIRST_LOCK(sc);
        microtime(&sc->last_activity);
        MYFIRST_UNLOCK(sc);
}
```

### The Runtime-Suspend and Runtime-Resume Helpers

These were sketched in Section 4. Here are the fleshed-out versions:

```c
static int
myfirst_runtime_suspend(struct myfirst_softc *sc)
{
        int err;

        device_printf(sc->dev, "runtime suspend: starting\n");

        err = myfirst_quiesce(sc);
        if (err != 0) {
                device_printf(sc->dev,
                    "runtime suspend: quiesce failed (err %d)\n", err);
                /* Undo the suspended flag the quiesce set. */
                MYFIRST_LOCK(sc);
                sc->suspended = false;
                MYFIRST_UNLOCK(sc);
                return (err);
        }

        pci_save_state(sc->dev);
        err = pci_set_powerstate(sc->dev, PCI_POWERSTATE_D3);
        if (err != 0) {
                device_printf(sc->dev,
                    "runtime suspend: set_powerstate(D3) failed "
                    "(err %d)\n", err);
                pci_restore_state(sc->dev);
                myfirst_restore(sc);
                return (err);
        }

        MYFIRST_LOCK(sc);
        sc->runtime_state = MYFIRST_RT_SUSPENDED;
        MYFIRST_UNLOCK(sc);
        atomic_add_64(&sc->runtime_suspend_count, 1);

        device_printf(sc->dev, "runtime suspend: device in D3\n");
        return (0);
}

static int
myfirst_runtime_resume(struct myfirst_softc *sc)
{
        int err;

        MYFIRST_LOCK(sc);
        if (sc->runtime_state != MYFIRST_RT_SUSPENDED) {
                MYFIRST_UNLOCK(sc);
                return (0);  /* nothing to do */
        }
        MYFIRST_UNLOCK(sc);

        device_printf(sc->dev, "runtime resume: starting\n");

        err = pci_set_powerstate(sc->dev, PCI_POWERSTATE_D0);
        if (err != 0) {
                device_printf(sc->dev,
                    "runtime resume: set_powerstate(D0) failed "
                    "(err %d)\n", err);
                return (err);
        }
        pci_restore_state(sc->dev);

        err = myfirst_restore(sc);
        if (err != 0) {
                device_printf(sc->dev,
                    "runtime resume: restore failed (err %d)\n", err);
                return (err);
        }

        MYFIRST_LOCK(sc);
        sc->runtime_state = MYFIRST_RT_RUNNING;
        MYFIRST_UNLOCK(sc);
        atomic_add_64(&sc->runtime_resume_count, 1);

        device_printf(sc->dev, "runtime resume: device in D0\n");
        return (0);
}
```

The shape is identical to system suspend/resume except that the driver explicitly calls `pci_set_powerstate` and `pci_save_state`/`pci_restore_state`. The PCI layer's automatic transitions are not in the loop for runtime PM because the kernel is not coordinating a system-wide power change; the driver is on its own.

### Interaction Between Runtime PM and System PM

The two paths have to cooperate. Consider what happens if the device is runtime-suspended (in D3) when the user closes the laptop lid:

1. The kernel starts system suspend.
2. The PCI bus calls `myfirst_pci_suspend`.
3. Inside `myfirst_pci_suspend`, the driver notices that the device is already runtime-suspended. The quiesce is a no-op (nothing is happening). The PCI layer's automatic config-space save runs; it reads the config space (which is still accessible in D3) and caches it.
4. The PCI layer transitions the device from D3 to... wait, it is already in D3. The transition to D3 is a no-op.
5. The system sleeps.
6. On wake, the PCI layer transitions the device back to D0. The driver's `myfirst_pci_resume` runs. It restores state. But now the driver thinks the device is `RUNNING` (because system resume cleared the `suspended` flag), while conceptually it was runtime-suspended before. The next activity will use the device normally and set `last_activity`; the idle watcher will eventually re-suspend it if still idle.

The interaction is mostly benign; the worst that happens is that the device gets one extra trip through D0 before the idle watcher re-suspends it. A more polished implementation would remember the runtime-suspended state across the system suspend and restore it, but for a learning driver the simple approach is enough.

The reverse (system-suspending a device that is already runtime-suspended) is already correct in our implementation because `myfirst_quiesce` checks `suspended` and returns 0 if already set. The runtime-suspended path set `suspended = true` as part of its quiesce, so the system suspend's quiesce sees the flag and skips.

### Exposing Runtime-PM Controls Through Sysctl

The driver's runtime-PM policy can be controlled and observed through sysctls:

```c
SYSCTL_ADD_INT(ctx, kids, OID_AUTO, "idle_threshold_seconds",
    CTLFLAG_RW, &sc->idle_threshold_seconds, 0,
    "Runtime PM idle threshold (seconds)");
SYSCTL_ADD_U64(ctx, kids, OID_AUTO, "runtime_suspend_count",
    CTLFLAG_RD, &sc->runtime_suspend_count, 0,
    "Runtime suspends performed");
SYSCTL_ADD_U64(ctx, kids, OID_AUTO, "runtime_resume_count",
    CTLFLAG_RD, &sc->runtime_resume_count, 0,
    "Runtime resumes performed");
SYSCTL_ADD_INT(ctx, kids, OID_AUTO, "runtime_state",
    CTLFLAG_RD, (int *)&sc->runtime_state, 0,
    "Runtime state: 0=running, 1=suspended");
```

A reader can now do this:

```sh
# Watch the device idle out.
while :; do
        sysctl dev.myfirst.0.runtime_state dev.myfirst.0.runtime_suspend_count
        sleep 1
done &
```

After five seconds of inactivity, `runtime_state` flips from 0 to 1 and `runtime_suspend_count` increments. A write to any active sysctl triggers a resume and flips the state back:

```sh
sudo sysctl dev.myfirst.0.dma_test_read=1
# The log shows: runtime resume, then the test read
```

### Tradeoffs

Runtime PM trades wake-up latency for idle power. Every D3-to-D0 transition costs time (tens of microseconds on a PCIe link, including the ASPM exit), and on some devices costs energy (the transition itself draws current). For a device that is idle most of the time with rare bursts of activity, the trade is favorable. For a device that is active most of the time with rare idle periods, the cost of the transitions dominates.

The `idle_threshold_seconds` knob lets the platform tune this. A value of 0 or 1 is aggressive and suitable for a webcam that is used for seconds at a time and idle for minutes. A value of 60 is conservative and suitable for a NIC whose idle periods are short but frequent. A value of 0 (if allowed) would disable runtime PM entirely, which is appropriate for devices that should stay on at all times.

A second tradeoff is in code complexity. Runtime PM adds a state machine, a callout, an idle watcher, two more kobj-like helpers, and additional ordering concerns between the runtime and system PM paths. Each of those is small, but together they increase the surface area for bugs. Many FreeBSD drivers deliberately omit runtime PM for this reason; they let the device stay in D0 and rely on the device's internal low-power states (clock gating, PCIe ASPM) to save power. That is a defensible choice, and for drivers where correctness matters more than milliwatts, it is the right one.

Chapter 22's `myfirst` driver keeps runtime PM as an optional feature, gated by a build-time flag:

```make
CFLAGS+= -DMYFIRST_ENABLE_RUNTIME_PM
```

A reader can build with or without the flag; the Section 5 code is only compiled in when the flag is defined. The default for Stage 3 is to leave runtime PM off; Stage 4 enables it in the consolidated driver.

### A Word on Platform Runtime PM

Some platforms provide their own runtime-PM mechanism alongside the driver-local one. On arm64 and RISC-V embedded systems, the device tree may describe `power-domains` and `clocks` properties that the driver uses to turn off power domains and gate clocks. FreeBSD's `ext_resources/clk`, `ext_resources/regulator`, and `ext_resources/power` subsystems handle these.

Runtime PM on such a platform is more capable than PCI-only runtime PM because the platform can turn off entire SoC blocks (a USB controller, a display engine, a GPU) rather than just moving the PCI device to D3. The driver uses the same pattern (mark idle, turn off resources on idle, turn back on for activity) but through different APIs.

Chapter 22 stays with the PCI path because that is where the `myfirst` driver lives. A reader who later works on an embedded platform will find the same conceptual structure with platform-specific APIs. The chapter mentions the distinction here so the reader knows the territory exists.

### Wrapping Up Section 5

Section 5 added runtime power management to the driver. It defined a two-state machine (`RUNNING`, `RUNTIME_SUSPENDED`), a callout-based idle watcher, a pair of helpers (`myfirst_runtime_suspend`, `myfirst_runtime_resume`) that use the PCI layer's explicit power-state and state-save APIs, the activity-recording hooks in the DMA sysctl handlers, and the sysctl knobs that expose the policy to user space. It also discussed the interaction between runtime PM and system PM, the latency-vs-power tradeoff, and the alternative of platform-level runtime PM on embedded systems.

With Sections 2 through 5 in place, the driver now handles system suspend, system resume, system shutdown, runtime suspend, and runtime resume. What it does not yet do cleanly is explain how the reader tests all of these from user space. Section 6 turns to the user-space interface: `acpiconf`, `zzz`, `devctl suspend`, `devctl resume`, `devinfo -v`, and the regression test that wraps them together.



## Section 6: 与电源框架交互

A driver that handles suspend and resume correctly is only half of the story. The other half is being able to *test* that correctness, repeatedly and deliberately, from user space. Section 6 surveys the tools FreeBSD provides for that purpose, explains how each fits the driver's state model, and shows how to combine them into a regression script that exercises every path Sections 2 through 5 built.

### The Four User-Space Entry Points

Four commands cover almost everything a driver developer needs:

- **`acpiconf -s 3`** (and its variants) asks ACPI to put the whole system into sleep state S3. This is the most realistic test; it exercises the full path from user space through the kernel's suspend machinery through the PCI layer to the driver's methods.
- **`zzz`** is a thin wrapper around `acpiconf -s 3`. It reads `hw.acpi.suspend_state` (defaulting to S3) and enters the corresponding sleep state. For most users it is the most convenient way to suspend from a shell.
- **`devctl suspend myfirst0`** and **`devctl resume myfirst0`** trigger per-device suspend and resume through the `DEV_SUSPEND` and `DEV_RESUME` ioctls on `/dev/devctl2`. These only call the driver's methods; the rest of the system stays in S0. This is the fastest iteration target and what Chapter 22 uses for most development.
- **`devinfo -v`** lists all devices in the device tree with their current state. It shows whether a device is attached, suspended, or detached.

Each has strengths and weaknesses. `acpiconf` is realistic but slow (one to three seconds per cycle on typical hardware) and disruptive (the system actually sleeps). `devctl` is fast (milliseconds per cycle) but exercises only the driver, not the ACPI or platform code. `devinfo -v` is passive and cheap; it observes without changing state.

A good regression strategy uses all three: `devctl` for unit testing of the driver's methods, `acpiconf` for integration testing of the full suspend path, and `devinfo -v` as a quick sanity check.

### Using acpiconf to Suspend the System

On a machine with working ACPI, `acpiconf -s 3` is what Section 1 called a full system suspend. The command:

```sh
sudo acpiconf -s 3
```

does the following:

1. It opens `/dev/acpi` and checks that the platform supports S3 via the `ACPIIO_ACKSLPSTATE` ioctl.
2. It sends the `ACPIIO_REQSLPSTATE` ioctl to request S3.
3. The kernel begins the suspend sequence: paused userland, frozen threads, device tree traversal with `DEVICE_SUSPEND` on each device.
4. Assuming no driver vetoes, the kernel enters S3. The machine sleeps.
5. A wake event (the lid opens, the power button is pressed, a USB device sends a remote-wakeup signal) wakes the platform.
6. The kernel runs the resume sequence: `DEVICE_RESUME` on each device, unfreezing threads, resuming userland.
7. The shell prompt returns. The machine is back in S0.

For the `myfirst` driver to be exercised, the driver must be loaded before the suspend. The entire sequence from user perspective looks like:

```sh
sudo kldload ./myfirst.ko
sudo sysctl dev.myfirst.0.dma_test_read=1  # exercise it a bit
sudo acpiconf -s 3
# [laptop sleeps; user opens lid]
dmesg | grep myfirst
```

The `dmesg` output should show two lines from Chapter 22's logging:

```text
myfirst0: suspend: starting
myfirst0: suspend: complete (dma in flight=0, suspended=1)
myfirst0: resume: starting
myfirst0: resume: complete
```

If those lines are present and in that order, the driver's methods were called correctly by the full system path.

If the machine does not come back, the suspend path broke at some layer below `myfirst`. If the machine comes back but the driver is in a strange state (the sysctls return errors, the counters have strange values, DMA transfers fail), the problem is in `myfirst`'s suspend or resume implementation.

### Using zzz

On FreeBSD, `zzz` is a small shell script that reads `hw.acpi.suspend_state` and calls `acpiconf -s <state>`. It is not a binary; it is usually installed at `/usr/sbin/zzz` and is a few lines long. A typical invocation is:

```sh
sudo zzz
```

The default `hw.acpi.suspend_state` is `S3` on machines that support it. A reader who wants to test S4 (hibernate) can:

```sh
sudo sysctl hw.acpi.suspend_state=S4
sudo zzz
```

S4 support on FreeBSD has historically been partial; whether it works depends on the platform firmware and the filesystem layout. For Chapter 22's purposes, S3 is sufficient, and `zzz` is the convenient shorthand.

### Using devctl for Per-Device Suspend

The `devctl(8)` command was built to let a user manipulate the device tree from user space. It supports attach, detach, enable, disable, suspend, resume, and more. For Chapter 22, `suspend` and `resume` are the two that matter.

```sh
sudo devctl suspend myfirst0
sudo devctl resume myfirst0
```

The first command issues `DEV_SUSPEND` through `/dev/devctl2`; the kernel translates that into a call to `BUS_SUSPEND_CHILD` on the parent bus, which for a PCI device ends up calling `pci_suspend_child`, which saves config space, puts the device in D3, and calls the driver's `DEVICE_SUSPEND`. The reverse happens for resume.

The key differences from `acpiconf`:

- Only the target device and its children go through the transition. The rest of the system stays in S0.
- The CPU does not park. Userland does not freeze. The kernel does not sleep.
- The PCI device actually goes to D3hot (assuming `hw.pci.do_power_suspend` is 1). The reader can verify with `pciconf`:

```sh
# Before suspend: device should be in D0
pciconf -lvbc | grep -A 2 myfirst

# After devctl suspend myfirst0: device should be in D3
sudo devctl suspend myfirst0
pciconf -lvbc | grep -A 2 myfirst
```

The power state is usually shown in the `powerspec` line of `pciconf -lvbc`. Moving from `D0` to `D3` is the observable signal that the transition really happened.

### Using devinfo to Inspect Device State

The `devinfo(8)` utility lists the device tree with various levels of detail. The `-v` flag shows verbose information, including the device state (attached, suspended, or not present).

```sh
devinfo -v | grep -A 5 myfirst
```

Typical output:

```text
myfirst0 pnpinfo vendor=0x1af4 device=0x1005 subvendor=0x1af4 subdevice=0x0004 class=0x008880 at slot=5 function=0 dbsf=pci0:0:5:0
    Resource: <INTERRUPT>
        10
    Resource: <MEMORY>
        0xfeb80000-0xfeb80fff
```

The state is implicit in the output: if the device is suspended, the line shows the device and its resources without the "active" marker. An explicit state query can be done through the softc sysctl; the `dev.myfirst.0.%parent` and `dev.myfirst.0.%desc` keys tell the user where the device sits.

For Chapter 22, `devinfo -v` is most useful as a sanity check after a failed transition: if the device is missing from the output, the detach path ran; if the device is present but the resources are wrong, the attach or resume path left the device in an inconsistent state.

### Inspecting Power States Through sysctl

The PCI layer exposes power-state information through `sysctl` under `hw.pci`. Two variables are most relevant:

```sh
sysctl hw.pci.do_power_suspend
sysctl hw.pci.do_power_resume
```

Both default to 1, meaning the PCI layer transitions devices to D3 on suspend and back to D0 on resume. Setting either to 0 disables the automatic transition for debugging.

The ACPI layer exposes system-state information:

```sh
sysctl hw.acpi.supported_sleep_state
sysctl hw.acpi.suspend_state
sysctl hw.acpi.s4bios
```

The first lists which sleep states the platform supports (typically something like `S3 S4 S5`). The second is the state `zzz` enters (usually `S3`). The third says whether S4 is implemented through BIOS assistance.

For per-device observation, the driver exposes its own state through `dev.myfirst.N.*`. The Chapter 22 driver adds:

- `dev.myfirst.N.suspended`: 1 if the driver considers itself suspended, 0 otherwise.
- `dev.myfirst.N.power_suspend_count`: number of times `DEVICE_SUSPEND` has been called.
- `dev.myfirst.N.power_resume_count`: number of times `DEVICE_RESUME` has been called.
- `dev.myfirst.N.power_shutdown_count`: number of times `DEVICE_SHUTDOWN` has been called.
- `dev.myfirst.N.runtime_state`: 0 for `RUNNING`, 1 for `RUNTIME_SUSPENDED`.
- `dev.myfirst.N.runtime_suspend_count`, `dev.myfirst.N.runtime_resume_count`: runtime-PM counters.
- `dev.myfirst.N.idle_threshold_seconds`: runtime-PM idle threshold.

Between these sysctls and `dmesg`, a reader can see in full detail what the driver did during any transition.

### A Regression Script

The labs directory grows a new script: `ch22-suspend-resume-cycle.sh`. The script:

1. Records the baseline values of every counter.
2. Runs one DMA transfer to confirm the device is working.
3. Calls `devctl suspend myfirst0`.
4. Verifies `dev.myfirst.0.suspended` is 1.
5. Verifies `dev.myfirst.0.power_suspend_count` has incremented by 1.
6. Calls `devctl resume myfirst0`.
7. Verifies `dev.myfirst.0.suspended` is 0.
8. Verifies `dev.myfirst.0.power_resume_count` has incremented by 1.
9. Runs one more DMA transfer to confirm the device still works.
10. Prints a PASS/FAIL summary.

The full script is in the examples directory; a short outline of the logic:

```sh
#!/bin/sh
set -e

DEV="dev.myfirst.0"

if ! sysctl -a | grep -q "^${DEV}"; then
    echo "FAIL: ${DEV} not present"
    exit 1
fi

before_s=$(sysctl -n ${DEV}.power_suspend_count)
before_r=$(sysctl -n ${DEV}.power_resume_count)
before_xfer=$(sysctl -n ${DEV}.dma_transfers_read)

# Baseline: run one transfer.
sysctl -n ${DEV}.dma_test_read=1 > /dev/null

# Suspend.
devctl suspend myfirst0
[ "$(sysctl -n ${DEV}.suspended)" = "1" ] || {
    echo "FAIL: device did not mark suspended"
    exit 1
}

# Resume.
devctl resume myfirst0
[ "$(sysctl -n ${DEV}.suspended)" = "0" ] || {
    echo "FAIL: device did not clear suspended"
    exit 1
}

# Another transfer.
sysctl -n ${DEV}.dma_test_read=1 > /dev/null

after_s=$(sysctl -n ${DEV}.power_suspend_count)
after_r=$(sysctl -n ${DEV}.power_resume_count)
after_xfer=$(sysctl -n ${DEV}.dma_transfers_read)

if [ $((after_s - before_s)) -ne 1 ]; then
    echo "FAIL: suspend count did not increment by 1"
    exit 1
fi
if [ $((after_r - before_r)) -ne 1 ]; then
    echo "FAIL: resume count did not increment by 1"
    exit 1
fi
if [ $((after_xfer - before_xfer)) -ne 2 ]; then
    echo "FAIL: expected 2 transfers (pre+post), got $((after_xfer - before_xfer))"
    exit 1
fi

echo "PASS: one suspend-resume cycle completed cleanly"
```

Running the script repeatedly (say a hundred times in a tight loop) is a good stress test. A driver that passes one cycle but fails on the fiftieth usually has a resource leak or an edge case that only shows up under repetition. That class of bug is exactly what a regression script is meant to find.

### Running the Stress Test

The chapter's `labs/` directory also includes `ch22-suspend-stress.sh`, which runs the cycle script a hundred times:

```sh
#!/bin/sh
N=100
i=0
while [ $i -lt $N ]; do
    if ! sh ./ch22-suspend-resume-cycle.sh > /dev/null; then
        echo "FAIL on iteration $i"
        exit 1
    fi
    i=$((i + 1))
done
echo "PASS: $N cycles"
```

On a modern machine with the simulation-only myfirst driver, a hundred cycles takes about a second. If any iteration fails, the script stops and reports the iteration number. Running this after each change during development catches regressions immediately.

### Combining Runtime PM and User-Space Testing

The runtime-PM path needs a different test, because it is not triggered by user commands; it is triggered by idleness. The test looks like:

```sh
# Ensure runtime_state is running.
sysctl dev.myfirst.0.runtime_state
# 0

# Do nothing for 6 seconds.
sleep 6

# The callout should have fired and runtime-suspended the device.
sysctl dev.myfirst.0.runtime_state
# 1

# Counter should have incremented.
sysctl dev.myfirst.0.runtime_suspend_count
# 1

# Any activity should bring it back.
sysctl dev.myfirst.0.dma_test_read=1
sysctl dev.myfirst.0.runtime_state
# 0

sysctl dev.myfirst.0.runtime_resume_count
# 1
```

A reader watching `dmesg` during this will see the "runtime suspend: starting" and "runtime suspend: device in D3" lines after about five seconds of inactivity, then "runtime resume: starting" when the sysctl write arrives.

The chapter's lab directory includes `ch22-runtime-pm.sh` to automate this sequence.

### Interpreting Failure Modes

When a user-space test fails, the diagnostic path depends on which layer failed:

- **If `devctl suspend` returns a non-zero exit code**: the driver's `DEVICE_SUSPEND` returned a non-zero value, vetoing the suspend. Check `dmesg` for the driver's log output; the suspend method should be logging what went wrong.
- **If `devctl suspend` succeeds but `dev.myfirst.0.suspended` is 0 afterwards**: the driver's quiesce set the flag briefly but something cleared it. This usually means the quiesce is re-entering itself, or the detach path is racing the suspend.
- **If `devctl resume` succeeds but the next transfer fails**: the restore path did not fully reinitialise the device. Most commonly, an interrupt mask or a DMA register was not written; check the per-vector fire counters before and after resume to see whether interrupts are reaching the driver.
- **If `acpiconf -s 3` succeeds but the system does not come back**: a driver below `myfirst` in the tree is blocking resume. This is unusual in a test VM; it is the classic failure mode on real hardware with new drivers.
- **If `acpiconf -s 3` returns `EOPNOTSUPP`**: the platform does not support S3. Check `sysctl hw.acpi.supported_sleep_state`.

In all cases, the first source of information is `dmesg`. The Chapter 22 driver logs every transition; if the log lines do not appear, the method was not called, and the problem is at a layer below the driver.

### A Minimal Troubleshooting Flow

A compact flowchart for a failed suspend-resume cycle:

1. Is the driver loaded? `kldstat | grep myfirst`.
2. Is the device attached? `sysctl dev.myfirst.0.%driver`.
3. Do the suspend and resume methods log? `dmesg | tail`.
4. Did `dev.myfirst.0.suspended` toggle correctly? `sysctl dev.myfirst.0.suspended`.
5. Do the counters increment? `sysctl dev.myfirst.0.power_suspend_count dev.myfirst.0.power_resume_count`.
6. Does a post-resume transfer succeed? `sudo sysctl dev.myfirst.0.dma_test_read=1; dmesg | tail -2`.
7. Do the per-vector interrupt counters increment? `sysctl dev.myfirst.0.vec0_fire_count dev.myfirst.0.vec1_fire_count dev.myfirst.0.vec2_fire_count`.

Any "no" answer points to a specific layer of the implementation. Section 7 goes deeper into the common failure modes and how to debug them.

### Wrapping Up Section 6

Section 6 surveyed the user-space interface to the kernel's power-management machinery: `acpiconf`, `zzz`, `devctl suspend`, `devctl resume`, `devinfo -v`, and the relevant `sysctl` variables. It showed how to combine these tools into a regression script that exercises one suspend-resume cycle, and a stress script that runs a hundred cycles in a row. It discussed the runtime-PM test flow, the interpretation of the most common failure modes, and the minimal troubleshooting flowchart a reader can follow when a test fails.

With the user-space tools in hand, the next section dives into the characteristic failure modes the reader is likely to encounter while writing power-aware code, and how to debug each one.



## Section 7: 调试电源管理问题

Power management code has a special class of bugs. The machine sleeps; the machine wakes; the bug shows up an unknown time after the wake and looks like a generic malfunction rather than anything related to the power transition. The chain of cause and effect is longer than with most driver bugs, the reproduction is slower, and the user's bug report is usually "my laptop doesn't wake up sometimes", which contains almost no information the driver developer can use.

Section 7 is about recognising the characteristic symptoms, tracing them back to their likely causes, and applying the matching debugging patterns. It draws on the Chapter 22 `myfirst` driver for concrete examples, but the patterns apply to any FreeBSD driver.

### Symptom 1: 恢复后设备冻结

The most common power-management bug, both in learning drivers and in production ones, is a device that stops responding after resume. The driver attaches correctly at boot, works normally in S0, handles a suspend-resume cycle without visible error, and then on the next command it is silent. Interrupts do not fire. DMA transfers do not complete. Any read from a device register returns stale values or zeros.

The usual cause is that the device's registers were not written after resume. The device came back in a default state (interrupt mask all-masked, DMA engine disabled, whatever registers the hardware resets on D0 entry), the driver did not reprogram them, and so from the device's perspective nothing is configured to run.

**Debugging pattern.** Compare the device's register values before and after suspend. The `myfirst` driver exposes several of its registers through sysctls (if the reader adds them); otherwise, the reader can write a short kernel-space helper that reads each register and prints it. After a suspend-resume cycle:

1. Read the interrupt mask register. If it is `0xFFFFFFFF` (all masked), the resume path did not restore the mask.
2. Read the DMA control register. If it has the ABORT bit set, the abort from the quiesce never cleared.
3. Read the device's configuration space via `pciconf -lvbc`. The command register should have the bus-master bit set; if not, `pci_enable_busmaster` was missed in the resume path.

**Fix pattern.** The resume path should include an unconditional reprogram of every device-specific register the driver's normal operation depends on. Saving them at suspend time into the softc and restoring them at resume time is one approach; re-deriving them from softc state (the approach `re_resume` takes) is another. Either works; the choice depends on which is easier to prove correct for the specific device.

### Symptom 2: Lost Interrupts

A subtler variant of the frozen-device problem is lost interrupts: the device is responding to some calls, but its interrupts are not reaching the driver. The DMA engine accepts a START command, performs the transfer, raises the completion interrupt... and the interrupt count does not increment. The task queue does not get an entry. The CV does not broadcast. The transfer eventually times out, and the driver reports EIO.

Several things can cause this:

- The **interrupt mask** at the device is still all-masked. The device wants to raise the interrupt but the mask suppresses it. (Resume path bug.)
- The **MSI or MSI-X configuration** was not restored. The device is raising the interrupt, but the kernel does not route it to the driver's handler. (Unusual; the PCI layer should handle this automatically.)
- The **filter function pointer** was corrupted. Extremely unusual; usually indicates memory corruption somewhere else in the driver.
- The **suspended flag** is still true, and the filter is returning early. (Resume path bug: flag not cleared.)

**Debugging pattern.** Read the per-vector fire counters before and after the suspend-resume cycle. If the counter does not increment, the interrupt is not reaching the filter. Then check, in order:

1. Is the suspended flag cleared? `sysctl dev.myfirst.0.suspended`.
2. Is the interrupt mask at the device correct? Read the register.
3. Is the MSI-X table in the device correct? `pciconf -c` dumps the capability registers.
4. Is the kernel's MSI dispatch state consistent? `procstat -t` shows the interrupt threads.

**Fix pattern.** Make sure the resume path (a) clears the suspended flag under the lock, (b) unmasks the device's interrupt register after clearing the flag, (c) does not rely on MSI-X restoration the driver must do itself (unless specifically disabled via sysctl).

### Symptom 3: 恢复后DMA错误

A more dangerous class of bug is DMA that appears to work but produces wrong data. The driver programs the engine, the engine runs, the completion interrupt fires, the task runs, the sync is called, the driver reads the buffer... and the bytes are wrong. Not zeros, not garbage, just subtly incorrect: the pattern written previously, or the pattern from two cycles ago, or a pattern that indicates the DMA addressed the wrong page.

Causes:

- The **bus address cached in the softc** is stale. This is unusual for a static allocation (the address is set once at attach and should not change), but it can happen if the driver re-allocates the DMA buffer at resume time (a bad idea; see below).
- The **DMA engine's base-address register** was not reprogrammed after resume, and it has a stale value that points elsewhere.
- The **`bus_dmamap_sync` calls are missing or mis-ordered**. This is the classic DMA-correctness bug, and it is worth being alert for in resume paths because the driver-side code adjacent to the sync calls is often edited during a refactor.
- The **IOMMU translation table** was not restored. Very rare on FreeBSD because the IOMMU configuration is per-session and survives suspend on most platforms; but if the driver is running on a system where `DEV_IOMMU` is unusual, this can bite.

**Debugging pattern.** Add a known-pattern write before each DMA, a verify after each DMA, and log both. Reducing the cycle to "write 0xAA, sync, read, expect 0xAA" makes data-corruption bugs visible immediately.

```c
memset(sc->dma_vaddr, 0xAA, MYFIRST_DMA_BUFFER_SIZE);
bus_dmamap_sync(sc->dma_tag, sc->dma_map, BUS_DMASYNC_PREWRITE);
/* run transfer */
bus_dmamap_sync(sc->dma_tag, sc->dma_map, BUS_DMASYNC_POSTWRITE);
if (((uint8_t *)sc->dma_vaddr)[0] != 0xAA) {
        device_printf(sc->dev,
            "dma: corruption detected after transfer\n");
}
```

For the simulation, this should always succeed because the simulation does not modify the buffer on a write transfer. On real hardware, the pattern depends on the device. A reader debugging a real-hardware bug adapts the test.

**Fix pattern.** If the bus address is the problem, rebuild it on resume:

```c
/* In resume, after PCI restore is complete. */
err = bus_dmamap_load(sc->dma_tag, sc->dma_map,
    sc->dma_vaddr, MYFIRST_DMA_BUFFER_SIZE,
    myfirst_dma_single_map, &sc->dma_bus_addr,
    BUS_DMA_NOWAIT);
```

Only do this if the bus address actually changed, which is rare. More commonly, the fix is to write the base-address register at the start of every transfer (rather than relying on a persistent value) and to make sure the sync calls are in the right order.

### Symptom 4: Lost PME# Wake Events

On a device that supports wake-on-X, the symptom is "the device should have woken the system but did not". The driver reported a successful suspend; the system went to S3; the expected event (magic packet, button press, timer) happened; and the system stayed asleep.

Causes:

- **`pci_enable_pme` was not called** in the suspend path. The device's PME_En bit is 0, so even when the device would normally assert PME#, the bit is suppressed.
- **The device's own wake logic is not configured**. For a NIC, the wake-on-LAN registers must be programmed before suspend. For a USB host controller, the remote-wakeup capability must be enabled per-port.
- **The platform's wake GPE is not enabled**. This is usually a firmware matter; the ACPI `_PRW` method should have registered the GPE, but on some machines the BIOS disables it by default.
- **The PME status bit is set at the time of suspend**, and a stale PME# is what triggers the wake (instead of the expected event). The system appears to wake immediately after sleeping.

**Debugging pattern.** Read the PCI configuration space via `pciconf -lvbc`. The power-management capability's status/control register shows PME_En and the PME_Status bit. Before suspending, PME_Status should be 0 (no pending wake). After suspending with wake enabled, PME_En should be 1.

On a machine where the wake does not happen, check the BIOS settings for "wake on LAN", "wake on USB", etc. The driver can be perfect and the system still not wake if the platform is not configured.

**Fix pattern.** In the suspend path of a wake-capable driver:

```c
static int
myfirst_pci_suspend(device_t dev)
{
        struct myfirst_softc *sc = device_get_softc(dev);
        int err;

        /* ... quiesce as before ... */

        if (sc->wake_enabled && pci_has_pm(dev)) {
                /* Program device-specific wake logic here. */
                myfirst_program_wake(sc);
                pci_enable_pme(dev);
        }

        /* ... rest of suspend ... */
}
```

In the resume path:

```c
static int
myfirst_pci_resume(device_t dev)
{
        if (pci_has_pm(dev))
                pci_clear_pme(dev);
        /* ... rest of resume ... */
}
```

The `myfirst` driver in Chapter 22 does not implement wake (the simulation has no wake logic). The pattern above is shown for reference.

### Symptom 5: 挂起期间WITNESS警告

A debug kernel with `WITNESS` enabled often produces messages like:

```text
witness: acquiring sleepable lock foo_mtx @ /path/to/driver.c:123
witness: sleeping with non-sleepable lock bar_mtx @ /path/to/driver.c:456
```

These are lock-order violations or sleep-while-locked violations, and they often show up in suspend code because suspend does things the driver does not normally do: acquire locks, sleep, and coordinate multiple threads.

Causes:

- The suspend path acquires a lock and then calls a function that sleeps without explicit tolerance for sleeping-with-that-lock-held.
- The suspend path acquires locks in a different order than the rest of the driver, and `WITNESS` notices the reversal.
- The suspend path calls `taskqueue_drain` or `callout_drain` while holding the softc lock, which causes a deadlock if the task or callout tries to acquire the same lock.

**Debugging pattern.** Read the `WITNESS` message carefully. It includes the lock names and the source-line numbers where each was acquired. Trace the path from the acquisition to the sleep or lock reversal.

**Fix pattern.** The Chapter 22 `myfirst_quiesce` drops the softc lock before calling `myfirst_drain_workers` for exactly this reason. When extending the driver:

- Do not call `taskqueue_drain` with any driver lock held.
- Do not call `callout_drain` with the lock that the callout acquires.
- Sleep primitives (`pause`, `cv_wait`) must be called with only sleep-mutexes held (not spin-mutexes).
- If you need to drop a lock for a sleep, do so explicitly and reacquire after.

### Symptom 6: Counters That Do Not Match

The chapter's regression script expects `power_suspend_count == power_resume_count` after each cycle. When they drift, something is wrong.

Causes:

- The driver's `DEVICE_SUSPEND` was called but the driver returned early before incrementing the counter. (Often because of a sanity check that fired.)
- The driver's `DEVICE_RESUME` was not called, because `DEVICE_SUSPEND` returned non-zero and the kernel unwound.
- The counters are not atomic and a concurrent update lost an increment. (Unlikely if the code uses `atomic_add_64`.)
- The driver was unloaded and reloaded between counts, resetting them.

**Debugging pattern.** Run the regression script with `dmesg -c` cleared beforehand, and `dmesg` after each cycle. The log shows every method invocation; counting the log lines is an alternative to counting the counters, and any difference indicates a bug.

### Symptom 7: Hangs During Suspend

A hang during suspend is the worst diagnostic: the kernel is still running (the console still responds to break-to-DDB), but the suspend sequence is stuck in some driver's `DEVICE_SUSPEND`. Break into DDB and `ps` to see which thread is where:

```text
db> ps
...  0 myfirst_drain_dma+0x42 myfirst_pci_suspend+0x80 ...
```

**Debugging pattern.** Identify the hanging thread and the function it is stuck in. Usually it is a `cv_wait` or `cv_timedwait` that never completed, or a `taskqueue_drain` waiting on a task that will not finish.

**Fix pattern.** Add a timeout to any wait the suspend path does. The `myfirst_drain_dma` function uses `cv_timedwait` with a one-second timeout; a variant that uses `cv_wait` (no timeout) can hang indefinitely. The chapter's implementation always uses timed variants for this reason.

### Using DTrace to Trace Suspend and Resume

DTrace is an excellent tool for observing the power-management path at fine granularity without adding print statements. A simple D script that times each call:

```d
fbt::device_suspend:entry,
fbt::device_resume:entry
{
    self->ts = timestamp;
    printf("%s: %s %s\n", probefunc,
        args[0] != NULL ? stringof(args[0]->name) : "?",
        args[0] != NULL ? stringof(args[0]->desc) : "?");
}

fbt::device_suspend:return,
fbt::device_resume:return
/self->ts/
{
    printf("%s: returned %d after %d us\n",
        probefunc, arg1,
        (timestamp - self->ts) / 1000);
    self->ts = 0;
}
```

Save this as `trace-devpower.d` and run with `dtrace -s trace-devpower.d`. Any `devctl suspend` or `acpiconf -s 3` will produce output showing each device's suspend and resume times, and their return values.

For the `myfirst` driver specifically, `fbt::myfirst_pci_suspend:entry` and `fbt::myfirst_pci_resume:entry` are the probes. A D script focused on the driver:

```d
fbt::myfirst_pci_suspend:entry {
    self->ts = timestamp;
    printf("myfirst_pci_suspend: entered\n");
    stack();
}

fbt::myfirst_pci_suspend:return
/self->ts/ {
    printf("myfirst_pci_suspend: returned %d after %d us\n",
        arg1, (timestamp - self->ts) / 1000);
    self->ts = 0;
}
```

The `stack()` call prints the call stack at entry, which is useful to confirm that the method is being called from where you expect (the PCI bus's `bus_suspend_child`, for example).

### A Note on Logging Discipline

The Chapter 22 code logs generously during suspend and resume: every method logs entry and exit, and each helper logs its own events. That verbosity is helpful during development but annoying in production (every laptop suspend prints half a dozen lines to dmesg).

A good production driver exposes a sysctl that controls log verbosity:

```c
static int myfirst_power_verbose = 1;
SYSCTL_INT(_dev_myfirst, OID_AUTO, power_verbose,
    CTLFLAG_RWTUN, &myfirst_power_verbose, 0,
    "Verbose power-management logging (0=off, 1=on, 2=debug)");
```

And the logging becomes conditional:

```c
if (myfirst_power_verbose >= 1)
        device_printf(dev, "suspend: starting\n");
```

A reader who wants to enable debugging on a production system can set `dev.myfirst.power_verbose=2` temporarily, trigger the problem, and reset the variable. The Chapter 22 driver does not implement this tiering; the learning driver logs everything and accepts the noise.

### Using the INVARIANTS Kernel for Assertion Coverage

A debug kernel with `INVARIANTS` compiled in causes `KASSERT` macros to actually evaluate their conditions and panic on failure. The `myfirst_dma.c` and `myfirst_pci.c` code uses several KASSERTs; the power-management code adds more. For example, the quiesce invariant:

```c
static int
myfirst_quiesce(struct myfirst_softc *sc)
{
        /* ... */

        KASSERT(sc->dma_in_flight == false,
            ("myfirst: dma_in_flight still true after drain"));

        return (0);
}
```

On an `INVARIANTS` kernel, a bug that leaves `dma_in_flight` true causes an immediate panic with a useful message. On a production kernel, the assertion is compiled out and nothing happens. The learning driver deliberately runs on an `INVARIANTS` kernel to catch this class of bug.

Similarly, the resume path can assert:

```c
KASSERT(sc->suspended == true,
    ("myfirst: resume called but not suspended"));
```

This catches a bug where the driver somehow gets resume called without the matching suspend having happened (usually a bug in a parent bus driver, not in the `myfirst` driver itself).

### A Debugging Case Study

To bring the patterns together, consider a concrete scenario. The reader writes the Stage 2 suspend, runs a regression cycle, and sees:

```text
myfirst0: suspend: starting
myfirst0: drain_dma: timeout waiting for abort
myfirst0: suspend: complete (dma in flight=0, suspended=1)
myfirst0: resume: starting
myfirst0: resume: complete
```

Then:

```sh
sudo sysctl dev.myfirst.0.dma_test_read=1
# Returns EBUSY after a long delay
```

The user-visible symptom is that the post-resume transfer does not work. The log shows a drain timeout during suspend, which is the first anomaly.

**Hypothesis.** The DMA engine did not honor the ABORT bit. The driver force-cleared `dma_in_flight`, but the engine is still running; when the user triggers a new transfer, the engine is not ready.

**Test.** Check the engine's status register before and after the abort:

```c
/* In myfirst_drain_dma, after writing ABORT: */
uint32_t pre_status = CSR_READ_4(sc->dev, MYFIRST_REG_DMA_STATUS);
DELAY(100);  /* let the engine notice */
uint32_t post_status = CSR_READ_4(sc->dev, MYFIRST_REG_DMA_STATUS);
device_printf(sc->dev, "drain: status %#x -> %#x\n", pre_status, post_status);
```

Running the cycle again produces:

```text
myfirst0: drain: status 0x4 -> 0x4
```

Status 0x4 is RUNNING. The engine ignored the ABORT. That points to the simulation backend: the simulated engine might not implement abort, or might do so only when the simulation callout fires.

**Fix.** Look at the simulation's DMA engine code and verify the abort semantics. In this case, the simulation's engine handles the abort in its callout callback, which does not fire for a few milliseconds. Extend the drain timeout from 1 second (plenty) to... wait, 1 second is plenty for a callout that fires every few milliseconds. The real issue is elsewhere.

Further investigation reveals that the simulation's callout was drained *before* the DMA drain completed. The order in `myfirst_drain_workers` (task first, callout second) was wrong; it should be callout first, task second, because the callout is what drives the abort completion.

**Resolution.** Reorder the drain:

```c
static void
myfirst_drain_workers(struct myfirst_softc *sc)
{
        /*
         * Drain the callout first: it runs the simulated engine's
         * completion logic, and the drain-DMA path waits on that
         * completion. Draining the callout after drain_dma would let
         * drain_dma time out and force-clear the in-flight flag.
         *
         * Wait - actually, drain_dma has already completed by the time
         * we get here, because myfirst_quiesce calls it first. So the
         * order of the two drains inside this function does not matter
         * for that reason. But drain_workers is also called from detach,
         * where drain_dma may not have been called, and the order there
         * does matter.
         */
        if (sc->sim != NULL)
                myfirst_sim_drain_dma_callout(sc->sim);

        if (sc->rx_vector.has_task)
                taskqueue_drain(taskqueue_thread, &sc->rx_vector.task);
}
```

But wait: by the time `myfirst_drain_workers` is called from `myfirst_quiesce`, `myfirst_drain_dma` has already completed. The drain-dma wait is inside the drain-dma call; the drain-workers call only cleans up residual state. The order inside drain-workers is mostly aesthetic for suspend.

The real fix is earlier: `myfirst_drain_dma` itself should not have timed out. The 1-second timeout should have been plenty. The actual cause is different: perhaps the simulation's callout was not firing because the driver held a sysctl lock that blocked it. Or the writing of the ABORT bit did not reach the simulation because the simulation's MMIO handler was also blocked.

**Lesson.** Debugging power management issues is iterative. Each symptom suggests a hypothesis; each test narrows it down; the fix is often in a different layer than the one the symptom pointed to. The patience to follow the chain is what distinguishes good power-aware code from code that mostly works.

### Wrapping Up Section 7

Section 7 walked through the characteristic failure modes of power-aware drivers: frozen devices, lost interrupts, bad DMA, missed wake events, WITNESS complaints, counter drift, and outright hangs. For each, it showed the typical cause, a debugging pattern that narrows the problem down, and the fix pattern that removes it. It also introduced DTrace for measurement, discussed log discipline, and showed how `INVARIANTS` and `WITNESS` catch the class of bug that only shows up under specific conditions.

The debugging discipline in Section 7, like the quiesce discipline in Section 3 and the restore discipline in Section 4, is meant to stay with the reader beyond the `myfirst` driver. Every power-aware driver has some variation of these bugs lurking in its implementation; the patterns above are how to find them before they reach the user.

Section 8 brings Chapter 22 to a close by consolidating the Sections 2 through 7 code into a refactored `myfirst_power.c` file, bumping the version to `1.5-power`, adding a `POWER.md` document, and wiring up a final integration test.



## Section 8: 重构和版本化电源感知驱动程序

Stages 1 through 3 added the power-management code inline in `myfirst_pci.c`. That was convenient for teaching, because every change appeared next to the attach and detach code the reader already knew. It is less convenient for readability: `myfirst_pci.c` now has attach, detach, three power methods, and several helpers, and the file is long enough that a first-time reader has to scroll to find things.

Stage 4, the final version of the Chapter 22 driver, pulls all of the power-management code out of `myfirst_pci.c` and into a new file pair, `myfirst_power.c` and `myfirst_power.h`. This follows the same pattern as Chapter 20's `myfirst_msix.c` split and Chapter 21's `myfirst_dma.c` split: the new file has a narrow, well-documented API, and the caller in `myfirst_pci.c` uses only that API.

### The Target Layout

After Stage 4, the driver's source files are:

- `myfirst.c` - top-level glue, shared state, sysctl tree.
- `myfirst_hw.c`, `myfirst_hw_pci.c` - register-access helpers.
- `myfirst_sim.c` - simulation backend.
- `myfirst_pci.c` - PCI attach, detach, method table, and thin forwarding to subsystem modules.
- `myfirst_intr.c` - single-vector interrupt (Chapter 19 legacy path).
- `myfirst_msix.c` - multi-vector interrupt setup (Chapter 20).
- `myfirst_dma.c` - DMA setup, teardown, transfer (Chapter 21).
- `myfirst_power.c` - power management (Chapter 22, new).
- `cbuf.c` - circular-buffer support.

The new `myfirst_power.h` declares the public API of the power subsystem:

```c
#ifndef _MYFIRST_POWER_H_
#define _MYFIRST_POWER_H_

struct myfirst_softc;

int  myfirst_power_setup(struct myfirst_softc *sc);
void myfirst_power_teardown(struct myfirst_softc *sc);

int  myfirst_power_suspend(struct myfirst_softc *sc);
int  myfirst_power_resume(struct myfirst_softc *sc);
int  myfirst_power_shutdown(struct myfirst_softc *sc);

#ifdef MYFIRST_ENABLE_RUNTIME_PM
int  myfirst_power_runtime_suspend(struct myfirst_softc *sc);
int  myfirst_power_runtime_resume(struct myfirst_softc *sc);
void myfirst_power_mark_active(struct myfirst_softc *sc);
#endif

void myfirst_power_add_sysctls(struct myfirst_softc *sc);

#endif /* _MYFIRST_POWER_H_ */
```

The `_setup` and `_teardown` pair initialise and tear down the subsystem-level state (the callout, the sysctls). The per-transition functions wrap the same logic the Section 3 through Section 5 code built. The runtime-PM functions are compiled only when the build-time flag is defined.

### The myfirst_power.c File

The new file is about three hundred lines. Its structure mirrors `myfirst_dma.c`: header includes, static helpers, public functions, sysctl handlers, and `_add_sysctls`.

The helpers are the three from Section 3:

- `myfirst_mask_interrupts`
- `myfirst_drain_dma`
- `myfirst_drain_workers`

Plus one from Section 4:

- `myfirst_restore`

And, if runtime PM is enabled, two from Section 5:

- `myfirst_idle_watcher_cb`
- `myfirst_start_idle_watcher`

The public functions `myfirst_power_suspend`, `myfirst_power_resume`, and `myfirst_power_shutdown` become thin wrappers that call the helpers in the right order and update counters. The sysctl handlers expose the policy knobs and the observability counters.

### Updating myfirst_pci.c

The `myfirst_pci.c` file is now much shorter. Its three power methods each just forward to the power subsystem:

```c
static int
myfirst_pci_suspend(device_t dev)
{
        return (myfirst_power_suspend(device_get_softc(dev)));
}

static int
myfirst_pci_resume(device_t dev)
{
        return (myfirst_power_resume(device_get_softc(dev)));
}

static int
myfirst_pci_shutdown(device_t dev)
{
        return (myfirst_power_shutdown(device_get_softc(dev)));
}
```

The method table stays the same as Stage 1 set it up. The three prototypes above are now the only power-related code in `myfirst_pci.c`, apart from the call to `myfirst_power_setup` from attach and `myfirst_power_teardown` from detach.

The attach path grows one call:

```c
static int
myfirst_pci_attach(device_t dev)
{
        /* ... existing attach code ... */

        err = myfirst_power_setup(sc);
        if (err != 0) {
                device_printf(dev, "power setup failed\n");
                /* unwind */
                myfirst_dma_teardown(sc);
                /* ... rest of unwind ... */
                return (err);
        }

        myfirst_power_add_sysctls(sc);

        return (0);
}
```

The detach path grows a matching call:

```c
static int
myfirst_pci_detach(device_t dev)
{
        /* ... existing detach code ... */

        myfirst_power_teardown(sc);

        /* ... rest of detach ... */

        return (0);
}
```

`myfirst_power_setup` initialises the `saved_intr_mask`, the `suspended` flag, the counters, and (if runtime PM is enabled) the idle watcher callout. `myfirst_power_teardown` drains the callout and cleans up any subsystem-level state. The teardown must be done before the DMA teardown because the callout may still reference DMA state.

### Updating the Makefile

The new source file goes into the `SRCS` list, and the version bumps:

```make
KMOD=  myfirst
SRCS=  myfirst.c \
       myfirst_hw.c myfirst_hw_pci.c \
       myfirst_sim.c \
       myfirst_pci.c \
       myfirst_intr.c \
       myfirst_msix.c \
       myfirst_dma.c \
       myfirst_power.c \
       cbuf.c

CFLAGS+= -DMYFIRST_VERSION_STRING=\"1.5-power\"

# Optional: enable runtime PM.
# CFLAGS+= -DMYFIRST_ENABLE_RUNTIME_PM

.include <bsd.kmod.mk>
```

The `MYFIRST_ENABLE_RUNTIME_PM` flag is off by default in Stage 4; the runtime-PM code compiles but is wrapped in `#ifdef`. A reader who wants to experiment enables the flag at build time.

### Writing POWER.md

The Chapter 21 pattern set the precedent: every subsystem gets a markdown document that describes its purpose, its API, its state model, and its testing story. `POWER.md` is next.

A good `POWER.md` has these sections:

1. **Purpose**: a paragraph explaining what the subsystem does.
2. **Public API**: a table of function prototypes with one-line descriptions.
3. **State Model**: a diagram or text description of the states and transitions.
4. **Counters and Sysctls**: the read-only and read-write sysctls the subsystem exposes.
5. **Transition Flows**: what happens during each of suspend, resume, shutdown.
6. **Interaction with Other Subsystems**: how power management relates to DMA, interrupts, and the simulation.
7. **Runtime PM (optional)**: how runtime PM works and when it is enabled.
8. **Testing**: the regression and stress scripts.
9. **Known Limitations**: what the subsystem does not do yet.
10. **See Also**: cross-references to `bus(9)`, `pci(9)`, and the chapter text.

The full document is in the examples directory (`examples/part-04/ch22-power/stage4-final/POWER.md`); the chapter does not reproduce it inline, but a reader who wants to check the expected structure can open it.

### Regression Script

The Stage 4 regression script exercises every path:

```sh
#!/bin/sh
# ch22-full-regression.sh

set -e

# 1. Basic sanity.
sudo kldload ./myfirst.ko

# 2. One suspend-resume cycle.
sudo sh ./ch22-suspend-resume-cycle.sh

# 3. One hundred cycles in a row.
sudo sh ./ch22-suspend-stress.sh

# 4. A transfer before, during, and after a cycle.
sudo sh ./ch22-transfer-across-cycle.sh

# 5. If runtime PM is enabled, test it.
if sysctl -N dev.myfirst.0.runtime_state >/dev/null 2>&1; then
    sudo sh ./ch22-runtime-pm.sh
fi

# 6. Unload.
sudo kldunload myfirst

echo "FULL REGRESSION PASSED"
```

Each sub-script is a few dozen lines and tests one thing. Running the full regression after each change catches regressions immediately.

### Integration With the Existing Regression Tests

The Chapter 21 regression script checked:

- `dma_complete_intrs == dma_complete_tasks` (the task always sees every interrupt).
- `dma_complete_intrs == dma_transfers_write + dma_transfers_read + dma_errors + dma_timeouts`.

The Chapter 22 script adds:

- `power_suspend_count == power_resume_count` (every suspend has a matching resume).
- The `suspended` flag is 0 outside of a transition.
- After a suspend-resume cycle, the DMA counters still add up to the expected total (no phantom transfers).

The combined regression is the Chapter 22 full script. It exercises DMA, interrupts, MSI-X, and power management together. A driver that passes it is in good shape.

### Version History

The driver has now evolved through several versions:

- `1.0` - Chapter 16: MMIO-only driver, simulation backend.
- `1.1` - Chapter 18: PCI attach, real BAR.
- `1.2-intx` - Chapter 19: single-vector interrupt with filter+task.
- `1.3-msi` - Chapter 20: multi-vector MSI-X with fallback.
- `1.4-dma` - Chapter 21: `bus_dma` setup, simulated DMA engine, interrupt-driven completion.
- `1.5-power` - Chapter 22: suspend/resume/shutdown, refactored into `myfirst_power.c`, optional runtime PM.

Each version builds on the previous one. A reader who has the Chapter 21 driver working can apply the Chapter 22 changes incrementally and end up at `1.5-power` without having to rewrite any earlier code.

### A Final Integration Test on Real Hardware

If the reader has access to real hardware (a machine with a working S3 implementation), the Chapter 22 driver can be exercised through a full system suspend:

```sh
sudo kldload ./myfirst.ko
sudo sh ./ch22-suspend-resume-cycle.sh
sudo acpiconf -s 3
# [laptop sleeps; user opens lid]
# After resume, the DMA test should still work.
sudo sysctl dev.myfirst.0.dma_test_read=1
```

On most platforms where ACPI S3 works, the driver survives the full cycle. The `dmesg` output shows the suspend and resume lines just as `devctl` would trigger, confirming that the same method code runs in both contexts.

If the full-system test fails where the per-device test succeeded, the extra work that system suspend does (ACPI sleep-state transitions, CPU parking, RAM self-refresh) has exposed something the per-device test missed. The usual culprits are device-specific register values that the system's low-power state resets but the per-device D3 does not. A driver that tests only with `devctl` can miss these; a driver that tests with `acpiconf -s 3` at least once before claiming correctness is more reliable.

### The Chapter 22 Code in One Place

A compact summary of what the Stage 4 driver added:

- **One new file**: `myfirst_power.c`, about three hundred lines.
- **One new header**: `myfirst_power.h`, about thirty lines.
- **One new markdown document**: `POWER.md`, about two hundred lines.
- **Five new softc fields**: `suspended`, `saved_intr_mask`, `power_suspend_count`, `power_resume_count`, `power_shutdown_count`, plus the runtime-PM fields when that feature is enabled.
- **Three new `DEVMETHOD` lines**: `device_suspend`, `device_resume`, `device_shutdown`.
- **Three new helper functions**: `myfirst_mask_interrupts`, `myfirst_drain_dma`, `myfirst_drain_workers`.
- **Two new subsystem entry points**: `myfirst_power_setup`, `myfirst_power_teardown`.
- **Three new transition functions**: `myfirst_power_suspend`, `myfirst_power_resume`, `myfirst_power_shutdown`.
- **Six new sysctls**: the counter nodes and the suspended flag.
- **Several new lab scripts**: cycle, stress, transfer-across-cycle, runtime-PM.

The overall line increment is about seven hundred lines of code, plus a couple of hundred lines of documentation and script. For the capability the chapter added (a driver that correctly handles every power transition the kernel can throw at it), that is a proportionate investment.

### Wrapping Up Section 8

Section 8 closed the Chapter 22 driver's construction by splitting the power code into its own file, bumping the version to `1.5-power`, adding a `POWER.md` document, and wiring the final regression test. The pattern was familiar from Chapters 20 and 21: take the inline code, extract it into a subsystem with a small API, document the subsystem, and integrate it into the rest of the driver through function calls rather than direct field access.

The resulting driver is power-aware in every sense the chapter introduced: it handles `DEVICE_SUSPEND`, `DEVICE_RESUME`, and `DEVICE_SHUTDOWN`; it quiesces the device cleanly; it restores state correctly on resume; it optionally implements runtime power management; it exposes its state through sysctls; it has a regression test; and it survives full-system suspend on real hardware when the platform supports it.



## Deep Look: Power Management in /usr/src/sys/dev/re/if_re.c

The Realtek 8169 and compatible gigabit NICs are handled by the `re(4)` driver. It is an informative driver to read for Chapter 22 purposes because it implements the full suspend-resume-shutdown trio with wake-on-LAN support, and its code has been stable enough to represent a canonical FreeBSD pattern. A reader who has worked through Chapter 22 can open `/usr/src/sys/dev/re/if_re.c` and recognise the structure immediately.

> **Reading this walkthrough.** The paired `re_suspend()` and `re_resume()` listings in the subsections below are taken from `/usr/src/sys/dev/re/if_re.c`, and the method-table excerpt abbreviates the full `re_methods[]` array with a `/* ... other methods ... */` comment so the three power-related `DEVMETHOD` entries stand out. We kept the signatures, the lock-acquire and lock-release pattern, and the order of device-specific calls (`re_stop`, `re_setwol`, `re_clrwol`, `re_init_locked`) intact; the real method table has many more entries, and the surrounding file carries the helper implementations. Every symbol the listings name is a real FreeBSD identifier in `if_re.c` that you can find with a symbol search.

### The Method Table

The `re(4)` driver's method table includes the three power methods near the top:

```c
static device_method_t re_methods[] = {
        DEVMETHOD(device_probe,     re_probe),
        DEVMETHOD(device_attach,    re_attach),
        DEVMETHOD(device_detach,    re_detach),
        DEVMETHOD(device_suspend,   re_suspend),
        DEVMETHOD(device_resume,    re_resume),
        DEVMETHOD(device_shutdown,  re_shutdown),
        /* ... other methods ... */
};
```

This is exactly the pattern Chapter 22 teaches. The `myfirst` driver's method table looks the same.

### re_suspend

The suspend function is about a dozen lines:

```c
static int
re_suspend(device_t dev)
{
        struct rl_softc *sc;

        sc = device_get_softc(dev);

        RL_LOCK(sc);
        re_stop(sc);
        re_setwol(sc);
        sc->suspended = 1;
        RL_UNLOCK(sc);

        return (0);
}
```

Three calls do the work: `re_stop` quiesces the NIC (disables interrupts, halts DMA, stops the RX and TX engines), `re_setwol` programs the wake-on-LAN logic and calls `pci_enable_pme` if WoL is enabled, and `sc->suspended = 1` sets the softc flag.

Compare to `myfirst_power_suspend`:

```c
int
myfirst_power_suspend(struct myfirst_softc *sc)
{
        int err;

        device_printf(sc->dev, "suspend: starting\n");
        err = myfirst_quiesce(sc);
        /* ... error handling ... */
        atomic_add_64(&sc->power_suspend_count, 1);
        return (0);
}
```

The structure is identical. `re_stop` and `re_setwol` together are the equivalent of `myfirst_quiesce`; the chapter's driver does not have wake-on-X, so there is no analogue of `re_setwol`.

### re_resume

The resume function is about thirty lines:

```c
static int
re_resume(device_t dev)
{
        struct rl_softc *sc;
        if_t ifp;

        sc = device_get_softc(dev);

        RL_LOCK(sc);

        ifp = sc->rl_ifp;
        /* Take controller out of sleep mode. */
        if ((sc->rl_flags & RL_FLAG_MACSLEEP) != 0) {
                if ((CSR_READ_1(sc, RL_MACDBG) & 0x80) == 0x80)
                        CSR_WRITE_1(sc, RL_GPIO,
                            CSR_READ_1(sc, RL_GPIO) | 0x01);
        }

        /*
         * Clear WOL matching such that normal Rx filtering
         * wouldn't interfere with WOL patterns.
         */
        re_clrwol(sc);

        /* reinitialize interface if necessary */
        if (if_getflags(ifp) & IFF_UP)
                re_init_locked(sc);

        sc->suspended = 0;
        RL_UNLOCK(sc);

        return (0);
}
```

The steps map cleanly to Chapter 22's discipline:

1. **Take the controller out of sleep mode** (MAC sleep bit on some Realtek parts). This is a device-specific restore step.
2. **Clear any WOL patterns** via `re_clrwol`, which reverses what `re_setwol` did. This also calls `pci_clear_pme` implicitly through the clear.
3. **Re-initialise the interface** if it was up before suspend. `re_init_locked` is the same function attach calls to bring up the NIC; it reprograms the MAC, resets the descriptor rings, enables interrupts, and starts the DMA engines.
4. **Clear the suspended flag** under the lock.

The `myfirst_power_resume` equivalent:

```c
int
myfirst_power_resume(struct myfirst_softc *sc)
{
        int err;

        device_printf(sc->dev, "resume: starting\n");
        err = myfirst_restore(sc);
        /* ... */
        atomic_add_64(&sc->power_resume_count, 1);
        return (0);
}
```

Again the structure is identical. `myfirst_restore` corresponds to the combination of the MAC-sleep exit, `re_clrwol`, `re_init_locked`, and the flag clear.

### re_shutdown

The shutdown function is:

```c
static int
re_shutdown(device_t dev)
{
        struct rl_softc *sc;

        sc = device_get_softc(dev);

        RL_LOCK(sc);
        re_stop(sc);
        /*
         * Mark interface as down since otherwise we will panic if
         * interrupt comes in later on, which can happen in some
         * cases.
         */
        if_setflagbits(sc->rl_ifp, 0, IFF_UP);
        re_setwol(sc);
        RL_UNLOCK(sc);

        return (0);
}
```

Similar to `re_suspend`, plus the interface-flag clear (shutdown is final; marking the interface down prevents spurious activity). The pattern is nearly identical; `re_shutdown` is essentially a more defensive version of `re_suspend`.

### re_setwol

The wake-on-LAN setup is worth looking at because it shows how a real driver calls the PCI PM APIs:

```c
static void
re_setwol(struct rl_softc *sc)
{
        if_t ifp;
        uint8_t v;

        RL_LOCK_ASSERT(sc);

        if (!pci_has_pm(sc->rl_dev))
                return;

        /* ... programs device-specific wake registers ... */

        /* Request PME if WOL is requested. */
        if ((if_getcapenable(ifp) & IFCAP_WOL) != 0)
                pci_enable_pme(sc->rl_dev);
}
```

Three key patterns appear here that are worth copying into any power-aware driver that supports wake-on-X:

1. **`pci_has_pm(dev)` guard.** The function returns early if the device does not support power management. This prevents writes to registers that do not exist.
2. **Device-specific wake programming.** The bulk of the function writes Realtek-specific registers through `CSR_WRITE_1`. A driver for a different device would write different registers, but the placement (inside the suspend path, before `pci_enable_pme`) is the same.
3. **Conditional `pci_enable_pme`.** Only enable PME# if the user has actually asked for wake-on-X. If the user has not, the function still sets the relevant configuration bits (for consistency with the driver's interface capabilities) but does not call `pci_enable_pme`.

The inverse is `re_clrwol`:

```c
static void
re_clrwol(struct rl_softc *sc)
{
        uint8_t v;

        RL_LOCK_ASSERT(sc);

        if (!pci_has_pm(sc->rl_dev))
                return;

        /* ... clears the wake-related config bits ... */
}
```

Note that `re_clrwol` does not explicitly call `pci_clear_pme`; the PCI layer's `pci_resume_child` has already called it before the driver's `DEVICE_RESUME`. `re_clrwol` is responsible for undoing the driver-visible side of WoL configuration, not the kernel-visible PME status.

### What the Deep Look Shows

The Realtek driver is more complex than `myfirst` by every measure (more registers, more state, more device variants), and yet its power-management discipline is less complex. That is because complexity of the *device* does not map one-to-one to complexity of the *power-management code*. Chapter 22's discipline scales down as well as it scales up: a simple device has a simple power path; a complex device has a modestly more complex power path. The structure is the same.

A reader who has finished Chapter 22 can now open `if_re.c`, recognise every function and every pattern, and understand why each exists. That comprehension transfers: the same recognition applies to `if_xl.c`, `virtio_blk.c`, and hundreds of other FreeBSD drivers. Chapter 22 is not teaching a `myfirst`-specific API; it is teaching the FreeBSD power-management idiom, and the `myfirst` driver is the vehicle that made it concrete.



## Deep Look: Simpler Patterns in if_xl.c and virtio_blk.c

For contrast, two other FreeBSD drivers implement power management in even simpler ways.

### if_xl.c: Shutdown Calls Suspend

The 3Com EtherLink III driver in `/usr/src/sys/dev/xl/if_xl.c` has the minimal three-method setup:

```c
static int
xl_shutdown(device_t dev)
{
        return (xl_suspend(dev));
}

static int
xl_suspend(device_t dev)
{
        struct xl_softc *sc;

        sc = device_get_softc(dev);

        XL_LOCK(sc);
        xl_stop(sc);
        xl_setwol(sc);
        XL_UNLOCK(sc);

        return (0);
}

static int
xl_resume(device_t dev)
{
        struct xl_softc *sc;
        if_t ifp;

        sc = device_get_softc(dev);
        ifp = sc->xl_ifp;

        XL_LOCK(sc);

        if (if_getflags(ifp) & IFF_UP) {
                if_setdrvflagbits(ifp, 0, IFF_DRV_RUNNING);
                xl_init_locked(sc);
        }

        XL_UNLOCK(sc);

        return (0);
}
```

Two things stand out:

1. `xl_shutdown` is one line: it just calls `xl_suspend`. For this driver, shutdown and suspend do the same work, and the code does not need two copies.
2. There is no `suspended` flag in the softc. The driver assumes the normal lifecycle of attach → run → suspend → resume, and uses the `IFF_DRV_RUNNING` flag (which the TX path already checks) as the equivalent. This is a perfectly valid approach for a NIC whose main user-visible state is the interface's running state.

For the `myfirst` driver, the explicit `suspended` flag is preferred because the driver has no natural equivalent of `IFF_DRV_RUNNING`. A NIC driver can reuse what it already has; a learning driver declares what it needs.

### virtio_blk.c: Minimal Quiesce

The virtio block driver in `/usr/src/sys/dev/virtio/block/virtio_blk.c` has an even shorter suspend path:

```c
static int
vtblk_suspend(device_t dev)
{
        struct vtblk_softc *sc;
        int error;

        sc = device_get_softc(dev);

        VTBLK_LOCK(sc);
        sc->vtblk_flags |= VTBLK_FLAG_SUSPEND;
        /* XXX BMV: virtio_stop(), etc needed here? */
        error = vtblk_quiesce(sc);
        if (error)
                sc->vtblk_flags &= ~VTBLK_FLAG_SUSPEND;
        VTBLK_UNLOCK(sc);

        return (error);
}

static int
vtblk_resume(device_t dev)
{
        struct vtblk_softc *sc;

        sc = device_get_softc(dev);

        VTBLK_LOCK(sc);
        sc->vtblk_flags &= ~VTBLK_FLAG_SUSPEND;
        vtblk_startio(sc);
        VTBLK_UNLOCK(sc);

        return (0);
}
```

The comment `/* XXX BMV: virtio_stop(), etc needed here? */` is an honest acknowledgement that the author was not sure how thorough the quiesce should be. The existing code sets a flag, waits for the queue to drain (that is what `vtblk_quiesce` does), and returns. On resume, it clears the flag and restarts I/O.

For a virtio block device, this is enough because the virtio host (the hypervisor) implements its own quiesce when the guest says it is suspending. The driver only needs to stop submitting new requests; the host deals with the rest.

This shows an important pattern: **the driver's quiesce depth depends on how much of the hardware's state is the driver's responsibility**. A bare-metal driver (like `re(4)`) has to program hardware registers carefully because the hardware has no other ally. A virtio driver has the hypervisor as an ally; the host can handle most of the state for the guest. The `myfirst` driver, running on a simulated backend, is in a similar position: the simulation is an ally, and the driver's quiesce can be correspondingly simpler.

### 比较显示了什么

Reading multiple drivers' power-management code side by side is one of the best ways to build fluency. Each driver adapts the Chapter 22 pattern to its context: `re(4)` handles wake-on-LAN, `xl(4)` reuses `xl_shutdown = xl_suspend`, `virtio_blk(4)` trusts the hypervisor. The common thread is the structure: stop activity, save state, flag suspended, return 0 from suspend; on resume, clear flag, restore state, restart activity, return 0.

A reader who has Chapter 22 in memory can open any FreeBSD driver, find its `device_suspend` and `device_resume` in the method table, and read the two functions. Within a few minutes the driver's power policy is clear. That skill transfers to every driver the reader will ever work on; it is the single most useful takeaway from the chapter.



## 深入理解：ACPI睡眠状态详解

Section 1 introduced the ACPI S-states as a list. It is worth revisiting them with the driver's point of view in focus, because the driver sees slightly different things depending on which S-state the kernel is entering.

### S0：工作状态

S0 is the state the reader has worked in throughout Chapters 16 to 21. The CPU is executing, RAM is refreshed, the PCIe links are up. From the driver's point of view, S0 is continuous; everything is normal.

Within S0, however, there can still be fine-grained power transitions. The CPU may enter idle states (C1, C2, C3, etc.) between scheduler ticks. The PCIe link may enter L0s or L1 based on ASPM. Devices may enter D3 based on runtime PM. None of these require the driver to do anything beyond its own runtime-PM logic; they are transparent.

### S1：待机

S1 is historically the lightest sleep state. The CPU stops executing but its registers are preserved; RAM stays powered; device power stays at D0 or D1. Wake latency is fast (under a second).

On modern hardware, S1 is rarely supported. The platform's BIOS advertises only S3 and deeper. If the platform does advertise S1 and the user enters it, the driver's `DEVICE_SUSPEND` is still called; the driver does its usual quiesce. The difference is that the PCI layer typically does not transition to D3 for S1 (because the bus stays powered), so the device stays in D0 through the transition. The driver's save and restore are largely unused.

A driver that supports S1 cleanly also supports S3, because the driver-side work is a subset. No driver written for Chapter 22 needs to treat S1 specially.

### S2：保留

S2 is defined in the ACPI specification but almost never implemented. A driver can safely ignore it; FreeBSD's ACPI layer treats S2 as S1 or S3 depending on platform support.

### S3：挂起到内存

S3 is the canonical sleep state Chapter 22 targets. When the user enters S3:

1. The kernel's suspend sequence traverses the device tree, calling `DEVICE_SUSPEND` on each driver.
2. The PCI layer's `pci_suspend_child` caches configuration space for each PCI device.
3. The PCI layer transitions each PCI device to D3hot.
4. Higher-level subsystems (ACPI, the CPU's idle machinery) enter their own sleep states.
5. The CPU's context is saved to RAM; the CPU halts.
6. RAM enters self-refresh; the memory controller maintains the contents with minimal power.
7. The platform's wake circuitry is armed: the power button, lid switch, and any configured wake sources.
8. The system waits for a wake event.

When a wake event arrives:

1. The CPU resumes; its context is restored from RAM.
2. Higher-level subsystems resume.
3. The PCI layer walks the device tree and calls `pci_resume_child` for each device.
4. Each device is transitioned to D0; its configuration is restored; pending PME# is cleared.
5. Each driver's `DEVICE_RESUME` is called.
6. User space unfreezes.

The driver sees only steps 1 (suspend) and 5 (resume) of each sequence. The rest is kernel and platform machinery.

A subtle point: during S3, RAM is refreshed but the kernel is not running. This means any kernel-side state (the softc, the DMA buffer, the pending tasks) survives S3 unchanged. The only thing that may be lost is hardware state: configuration registers in the device may be reset; BAR-mapped registers may return to default values. The driver's job on resume is to re-program the hardware from the preserved kernel state.

### S4：挂起到磁盘（休眠）

S4 is the "hibernate" state. The kernel writes the full contents of RAM to a disk image, then enters S5. On wake, the platform boots, the kernel reads the image back, and the system continues from where it left off.

On FreeBSD, S4 has historically been partial. The kernel can produce the hibernation image on some platforms, but the restore path is not as mature as Linux's. For driver purposes, S4 is the same as S3: the `DEVICE_SUSPEND` and `DEVICE_RESUME` methods are called; the driver's quiesce and restore paths work without change. The extra platform-level work (writing the image) is transparent.

The one difference the driver might notice is that after S4 resume, the PCI configuration space is always restored from scratch (the platform has fully rebooted), so even if the driver were relying on `hw.pci.do_power_suspend` being 0 to keep the device in D0, after S4 the device will still have been through a full power cycle. This matters only for drivers that do platform-specific tricks during suspend; most drivers are oblivious.

### S5：软关机

S5 is system power-off. The power button, the battery (if any), and the wake circuitry still receive power; everything else is off.

From the driver's point of view, S5 looks like a shutdown: `DEVICE_SHUTDOWN` is called (not `DEVICE_SUSPEND`), the driver places the device in a safe state for power-off, and the system halts. There is no resume corresponding to S5; if the user presses the power button, the system boots from scratch.

Shutdown is not a power transition in the reversible sense; it is a termination. The driver's `DEVICE_SHUTDOWN` method is called once, and the driver does not expect to run again until the next boot. The chapter's `myfirst_power_shutdown` handles this correctly by quiescing the device (same as suspend) and not trying to save any state (because there is no resume to save for).

### 观察平台支持的状态

On any FreeBSD 14.3 system with ACPI, the supported states are exposed through a sysctl:

```sh
sysctl hw.acpi.supported_sleep_state
```

Typical outputs:

- A modern laptop: `S3 S4 S5`
- A server: `S5` (suspend not supported on many server platforms)
- A VM on bhyve: varies; usually `S5` only
- A VM on QEMU/KVM with `-machine q35`: often `S3 S4 S5`

If a driver is meant to work on a specific platform, the supported-state list tells you which transitions you need to test. A driver that only runs on servers does not need S3 testing; a driver meant for laptops does.

### 测试内容

For Chapter 22's purposes, the minimum test is:

- `devctl suspend` / `devctl resume`: always possible; tests the driver-side code path.
- `acpiconf -s 3` (if supported): tests the full system suspend.
- System shutdown (`shutdown -p now`): tests the `DEVICE_SHUTDOWN` method.

S4 and runtime PM are optional; they exercise less-used code paths. A driver that passes the minimum test on a platform that supports S3 is in good shape; extensions are icing.

### 睡眠状态到驱动程序方法的映射

A compact table of which kobj method is called for each transition:

| Transition          | Method             | Driver Action                                    |
|---------------------|--------------------|--------------------------------------------------|
| S0 → S1             | DEVICE_SUSPEND     | Quiesce; save state                              |
| S0 → S3             | DEVICE_SUSPEND     | Quiesce; save state (device likely goes to D3)   |
| S0 → S4             | DEVICE_SUSPEND     | Quiesce; save state (followed by hibernate)      |
| S0 → S5 (shutdown)  | DEVICE_SHUTDOWN    | Quiesce; leave in safe state for power-off       |
| S1/S3 → S0          | DEVICE_RESUME      | Restore state; unmask interrupts                 |
| S4 → S0 (resume)    | (attach from boot) | Normal attach, because the kernel booted fresh   |
| devctl suspend      | DEVICE_SUSPEND     | Quiesce; save state (device goes to D3)          |
| devctl resume       | DEVICE_RESUME      | Restore state; unmask interrupts                 |

The driver does not distinguish S1, S3, and S4 from its own code; it always does the same work. The differences are at the platform and kernel levels. That uniformity is what makes the pattern scalable: one suspend path, one resume path, multiple contexts.



## 深入理解：PCIe链路状态和ASPM实战

Section 1 sketched the PCIe link states (L0, L0s, L1, L1.1, L1.2, L2). It is worth seeing how they behave in practice, because understanding them helps the driver developer interpret latency measurements and power observations.

### 为什么链路有自己的状态

A PCIe link is a pair of high-speed differential lanes between two endpoints (root complex and device, or root complex and switch). Each lane has a transmitter and a receiver; each lane's transmitter consumes power to keep the channel in a known state. When traffic is low, the transmitters can be turned off in various degrees, and the link can be re-established quickly when traffic resumes. The L-states describe those degrees.

The link's state is separate from the device's D-state. A device in D0 can have its link in L1 (the link is idle; the device is not transmitting or receiving). A device in D3 has its link in L2 or similar (the link is off). A device in D0 with a busy link is in L0.

### L0：活动

L0 is the normal operating state. Both sides of the link are active; data can flow in either direction; latency is at its minimum (a few hundred nanoseconds round-trip on a modern PCIe).

When a DMA transfer is running or an MMIO read is pending, the link is in L0. The device's own logic and the PCIe host bridge both require L0 for the transaction.

### L0s：发送器待机

L0s is a low-power state where one side of the link's transmitter is turned off. The receiver stays on; the link can be brought back to L0 in under a microsecond.

L0s is entered automatically by the link logic when no traffic has been sent for a few microseconds. The platform's PCIe host bridge and the device's PCIe interface cooperate: when the transmit FIFO is empty and ASPM is enabled, the transmitter goes off. When new traffic arrives, the transmitter comes back on.

L0s is "asymmetric": each side independently enters and exits the state. A device's transmitter can be in L0s while the root complex's transmitter is in L0. This is useful because traffic is typically bursty: the CPU sends a DMA trigger, then does not send anything else for a while; the CPU's transmitter enters L0s quickly, while the device's transmitter stays in L0 because it is actively sending the DMA response.

### L1：双方待机

L1 is a deeper state where both transmitters are off. Neither side can send anything until the link is brought back to L0; the latency is measured in microseconds (5 to 65, depending on platform).

L1 is entered after a longer idle period than L0s. The exact threshold is configurable through ASPM settings; typical values are tens of microseconds of inactivity. L1 saves more power than L0s but costs more to exit.

### L1.1和L1.2：更深的L1子状态

PCIe 3.0 and later define sub-states of L1 that turn off additional parts of the physical layer. L1.1 (also called "L1 PM Substate 1") keeps the clock running but turns off more circuitry; L1.2 turns off the clock as well. The wake latencies increase (tens of microseconds for L1.1; hundreds for L1.2), but the idle power draws decrease.

Most modern laptops use L1.1 and L1.2 aggressively to extend battery life. A laptop that stays in L1.2 most of the idle time can have PCIe power draw in the single-digit milliwatts, compared to hundreds of milliwatts in L0.

### L2：近关闭

L2 is the state the link enters when the device is in D3cold. The link is effectively off; re-establishing it requires a full link-training sequence (tens of milliseconds). L2 is entered as part of the full-system suspend sequence; the driver does not manage it directly.

### 谁控制ASPM

ASPM is a per-link feature configured through the PCIe Link Capability and Link Control registers in both the root complex and the device. The configuration specifies:

- Whether L0s is enabled (one-bit field).
- Whether L1 is enabled (one-bit field).
- The exit latency thresholds the platform considers acceptable.

On FreeBSD, ASPM is usually controlled by the platform firmware through ACPI's `_OSC` method. The firmware tells the OS which capabilities to manage; if the firmware keeps ASPM control, the OS does not touch it. If the firmware hands over control, the OS may enable or disable ASPM per link based on policy.

For Chapter 22's `myfirst` driver, ASPM is the platform's job. The driver does not configure ASPM; it does not need to know whether the link is in L0 or L1 at any moment. The link's state is invisible to the driver from a functional standpoint (latency is the only observable effect).

### ASPM何时对驱动程序重要

There are specific situations where a driver does have to worry about ASPM:

1. **Known errata.** Some PCIe devices have bugs in their ASPM implementation that cause the link to wedge or produce corrupted transactions. The driver may need to explicitly disable ASPM for those devices. The kernel provides the PCIe Link Control register access through `pcie_read_config` and `pcie_write_config` for this purpose.

2. **Latency-sensitive devices.** A real-time audio or video device may not tolerate the microsecond-scale latency of L1. The driver may disable L1 while keeping L0s enabled.

3. **Power-sensitive devices.** A battery-powered device may want L1.2 always enabled. The driver may force L1.2 if the platform's default is less aggressive.

For the `myfirst` driver, none of these apply. The simulated device does not have a link at all; the real PCIe link (if any) is handled by the platform. The chapter mentions ASPM for completeness and moves on.

### 观察链路状态

On a system where the platform supports ASPM observation, the link state is exposed through `pciconf -lvbc`:

```sh
pciconf -lvbc | grep -A 20 myfirst
```

Look for lines like:

```text
cap 10[ac] = PCI-Express 2 endpoint max data 128(512) FLR NS
             link x1(x1) speed 5.0(5.0)
             ASPM disabled(L0s/L1)
             exit latency L0s 1us/<1us L1 8us/8us
             slot 0
```

The "ASPM disabled" on this line says ASPM is not currently active. "disabled(L0s/L1)" says the device supports both L0s and L1 but neither is enabled. On a system with aggressive ASPM, the line would read "ASPM L1" or similar.

The exit latencies tell the driver how long the transition back to L0 takes; a latency-sensitive driver can decide whether L1 is tolerable by looking at this number.

### 链路状态和功耗

A rough table of PCIe power draws (typical values; actual depend on implementation):

| State | Power (x1 link) | Exit Latency |
|-------|-----------------|--------------|
| L0    | 100-200 mW      | 0            |
| L0s   | 50-100 mW       | <1 µs        |
| L1    | 10-30 mW        | 5-65 µs      |
| L1.1  | 1-5 mW          | 10-100 µs    |
| L1.2  | <1 mW           | 50-500 µs    |
| L2    | near 0          | 1-100 ms     |

For a laptop with a dozen PCIe links all in L1.2 during idle, the aggregate savings relative to all-L0 can be in the watts. For a server with high-throughput links always in L0, ASPM is disabled and the power saving is zero.

Chapter 22 does not implement ASPM for `myfirst`. The chapter mentions it because understanding the link state machine is part of understanding the full power-management picture. A reader who later works on a driver with known ASPM errata will know where to look.



## 深入理解：唤醒源详解

Wake sources are the mechanisms that bring a suspended system or device back to active. Chapter 1 mentioned them briefly; this deeper look walks through the most common ones.

### PCIe上的PME#

The PCI spec defines the `PME#` signal (Power Management Event). When asserted, it tells the upstream root complex that the device has an event worth waking for. The root complex converts PME# into an ACPI GPE or interrupt, which the kernel handles.

A device that supports PME# has a PCI power-management capability (checked via `pci_has_pm`). The capability's control register includes:

- **PME_En** (bit 8): enable PME# generation.
- **PME_Status** (bit 15): set by the device when PME# is raised, cleared by software.
- **PME_Support** (read-only, bits 11-15 in PMC register): which D-states the device can raise PME# from (D0, D1, D2, D3hot, D3cold).

The driver's job is to set PME_En at the right time (usually before suspend) and to clear PME_Status at the right time (usually after resume). The `pci_enable_pme(dev)` and `pci_clear_pme(dev)` helpers do both jobs.

On a typical laptop, the root complex routes PME# to an ACPI GPE, which the kernel's ACPI driver picks up as a wake event. The chain looks like:

```text
device asserts PME#
  → root complex receives PME
  → root complex sets GPE status bit
  → ACPI hardware interrupts CPU
  → kernel wakes from S3
  → kernel's ACPI driver services the GPE
  → eventually: DEVICE_RESUME on the device that woke
```

The whole chain takes one to three seconds. The driver's role is minimal: it enabled PME# before suspend, and it will clear PME_Status after resume. Everything else is platform.

### USB远程唤醒

USB has its own wake mechanism called "remote wakeup". A USB device requests wake capability through its standard descriptor; the host controller enables the capability at enumeration time; when the device asserts a resume signal on its upstream port, the host controller propagates it.

From a FreeBSD driver perspective, USB remote wakeup is almost entirely handled by the USB host controller driver (`xhci`, `ohci`, `uhci`). Individual USB device drivers (for keyboards, storage, audio, etc.) participate through the USB framework's suspend and resume callbacks, but they do not deal with PME# directly. The USB host controller's own PME# is what actually wakes the system.

For Chapter 22 purposes, USB wake is a black box that works through the USB host controller driver. A reader who eventually writes a USB device driver will learn the framework's conventions then.

### 嵌入式平台上的GPIO唤醒

On embedded platforms (arm64, RISC-V), wake sources are typically GPIO pins connected to the SoC's wake logic. The device tree describes which pins are wake sources via `wakeup-source` properties and `interrupts-extended` pointing to the wake controller.

FreeBSD's GPIO intr framework handles these. A device driver whose hardware is wake-capable reads the device-tree `wakeup-source` property during attach, registers the GPIO as a wake source with the framework, and the framework does the rest. The mechanism is very different from PCIe PME#, but the driver-side API (mark wake enabled, clear wake status) is conceptually similar.

Chapter 22 does not exercise GPIO wake; the `myfirst` driver is a PCI device. Part 7 revisits embedded platforms and covers the GPIO path in detail.

### 局域网唤醒（WoL）

Wake on LAN is a specific implementation pattern for a network controller. The controller watches incoming packets for a "magic packet" (a specific pattern containing the controller's MAC address repeated many times) or for user-configured patterns. When a match is detected, the controller asserts PME# upstream.

From the driver's perspective, WoL requires:

1. Configuring the NIC's wake logic (magic-packet filter, pattern filters) before suspend.
2. Enabling PME# via `pci_enable_pme`.
3. On resume, disabling the wake logic (because normal packet processing would otherwise be influenced by the filters).

The `re(4)` driver's `re_setwol` is the canonical FreeBSD example. A reader building a NIC driver copies its structure and adapts the device-specific register programming.

### 盖子、电源按钮等唤醒

The laptop's lid switch, power button, keyboard (in some cases), and other platform inputs are wired to the platform's wake logic through ACPI. The ACPI driver handles the wake; individual device drivers are not involved.

The ACPI `_PRW` method on a device's object in the ACPI namespace declares which GPE that device's wake event uses. The OS reads `_PRW` during boot to configure the wake routing. The `myfirst` driver, as a simple PCI endpoint with no platform-specific wake source, does not have a `_PRW` method; its wake capability (if any) is purely through PME#.

### 驱动程序何时必须启用唤醒

A simple heuristic: the driver must enable wake if the user has asked for it (through an interface capability flag like `IFCAP_WOL` for NICs) and the hardware supports it (`pci_has_pm` returns true, the device's own wake logic is operational). Otherwise, the driver leaves wake disabled.

A driver that enables wake for every device by default wastes platform power; the wake circuitry and PME# routing cost a few milliwatts continuously. A driver that never enables wake frustrates users who want their laptop to wake on a network packet. The policy is "enable only when asked".

FreeBSD's interface capabilities (set via `ifconfig em0 wol wol_magic`) are the standard way users express the desire. The NIC driver reads the flags and configures WoL accordingly.

### 测试唤醒源

Testing wake is harder than testing suspend and resume, because testing wake requires the system to actually sleep and then an external event to wake it. Common approaches:

- **Magic packet from another machine.** Send a WoL magic packet to the suspended machine's MAC address. If WoL is working, the machine wakes in a few seconds.
- **Lid switch.** Close the lid, wait, open the lid. If the platform's wake routing is working, the machine wakes on open.
- **Power button.** Press the power button briefly while suspended. The machine should wake.

For a learning driver like `myfirst`, there is no meaningful wake source to test against. The chapter mentions wake mechanics for pedagogical completeness, not because the driver exercises them.



## 深入理解：hw.pci.do_power_suspend可调参数

One of the most important tunables for power-management debugging is `hw.pci.do_power_suspend`. It controls whether the PCI layer automatically transitions devices to D3 during system suspend. Understanding what it does and when to change it is worth a dedicated look.

### 默认值的作用

With `hw.pci.do_power_suspend=1` (the default), the PCI layer's `pci_suspend_child` helper, after calling the driver's `DEVICE_SUSPEND`, transitions the device to D3hot by calling `pci_set_power_child(dev, child, PCI_POWERSTATE_D3)`. On resume, `pci_resume_child` transitions back to D0.

This is the "power-save" mode. A device that supports D3 uses its lowest-power idle state during suspend. A laptop benefits because battery life during sleep is extended; a device that can sleep at a few milliwatts instead of a few hundred is worth the extra D-state transition.

### hw.pci.do_power_suspend=0的作用

With the tunable set to 0, the PCI layer does not transition the device to D3. The device stays in D0 throughout the suspend. The driver's `DEVICE_SUSPEND` runs; the driver quiesces activity; the device stays powered.

From a power-saving perspective, this is worse: the device continues to draw its D0 power budget during sleep. From a correctness perspective, it can be better for some devices:

- A device with broken D3 implementation may misbehave when transitioned. Staying in D0 avoids the transition bug.
- A device whose context is expensive to save and restore may prefer to stay in D0 during a short suspend. If the suspend is only a few seconds, the context-save cost exceeds the power-saving benefit.
- A device that is critical to the machine's core function (a console keyboard, for example) may need to stay alert even during suspend.

### 何时更改

For development and debugging, setting `hw.pci.do_power_suspend=0` can isolate bugs:

- If a resume bug appears only with the tunable at 1, the bug is in the D3-to-D0 transition (either in the PCI layer's config restore, or in the driver's handling of a device that has been reset).
- If a resume bug appears with the tunable at 0 as well, the bug is in the driver's `DEVICE_SUSPEND` or `DEVICE_RESUME` code, not in the D-state machinery.

For production, the default (1) is almost always right. Changing it globally affects every PCI device on the system; a better approach is a per-device override if one is needed, which typically lives in the driver itself.

### 验证可调参数生效

A quick way to verify is to check the device's power state with `pciconf` before and after a suspend:

```sh
# Before suspend (device should be in D0):
pciconf -lvbc | grep -A 5 myfirst

# With hw.pci.do_power_suspend=1 (default):
sudo devctl suspend myfirst0
pciconf -lvbc | grep -A 5 myfirst
# "powerspec" should show D3

# With hw.pci.do_power_suspend=0:
sudo sysctl hw.pci.do_power_suspend=0
sudo devctl resume myfirst0
sudo devctl suspend myfirst0
pciconf -lvbc | grep -A 5 myfirst
# "powerspec" should show D0

# Reset to default.
sudo sysctl hw.pci.do_power_suspend=1
sudo devctl resume myfirst0
```

The `powerspec` line in `pciconf -lvbc` output shows the current power state. Watching it change between D0 and D3 confirms the automatic transition is happening.

### 与pci_save_state的交互

When `hw.pci.do_power_suspend` is 1, the PCI layer automatically calls `pci_cfg_save` before transitioning to D3. When it is 0, the PCI layer does not call `pci_cfg_save`.

This has a subtle implication: if the driver wants to save configuration explicitly in the 0 case, it must call `pci_save_state` itself. The Chapter 22 pattern assumes the default (1) and does not call `pci_save_state` explicitly; a driver that wants to support both modes would need additional logic.

### 可调参数影响系统挂起还是devctl挂起？

Both. `pci_suspend_child` is called for both `acpiconf -s 3` and `devctl suspend`, and the tunable gates the D-state transition in both cases. A reader debugging with `devctl suspend` will see the same behavior as with a full system suspend, modulo the other platform work (CPU park, ACPI sleep state entry).

### 具体调试场景

Suppose the `myfirst` driver's resume fails intermittently: sometimes it works, sometimes `dma_test_read` after resume returns EIO. The counters are consistent (suspend count = resume count), the logs show both methods ran, but the post-resume DMA fails.

**Hypothesis 1.** The D3-to-D0 transition is producing an inconsistent device state. Verify by setting `hw.pci.do_power_suspend=0` and retrying.

If the bug disappears with the tunable at 0, the D-state machinery is involved. The fix might be in the driver's resume path (add a delay after the transition to let the device stabilise), in the PCI layer's config restore, or in the device itself.

**Hypothesis 2.** The bug is in the driver's own suspend/resume code, independent of D3. Verify by setting the tunable to 0 and retrying.

If the bug persists with the tunable at 0, the driver's code is the problem. The D3 transition is innocent.

This kind of bisection is common in power-management debugging. The tunable is the tool that lets you isolate the variable.



## 深入理解：DEVICE_QUIESCE和何时需要它

Section 2 briefly mentioned `DEVICE_QUIESCE` as the third power-management method alongside `DEVICE_SUSPEND` and `DEVICE_SHUTDOWN`. It is rarely implemented explicitly in FreeBSD drivers; a search of `/usr/src/sys/dev/` shows only a handful of drivers define their own `device_quiesce`. Understanding when you do need it and when you do not is worth a short section.

### DEVICE_QUIESCE的用途

The `device_quiesce` wrapper in `/usr/src/sys/kern/subr_bus.c` is called in several places:

- `devclass_driver_deleted`: when a driver is being unloaded, the framework calls `device_quiesce` on every instance before calling `device_detach`.
- `DEV_DETACH` via devctl: when the user runs `devctl detach myfirst0`, the kernel calls `device_quiesce` before `device_detach` unless the `-f` (force) flag is given.
- `DEV_DISABLE` via devctl: when the user runs `devctl disable myfirst0`, the kernel calls `device_quiesce` similarly.

In each case, the quiesce is a pre-check: "can the driver safely stop what it is doing?". A driver that returns EBUSY from `DEVICE_QUIESCE` prevents the subsequent detach or disable. The user gets an error, and the driver stays attached.

### 默认值的作用

If a driver does not implement `DEVICE_QUIESCE`, the default (`null_quiesce` in `device_if.m`) returns 0 unconditionally. The kernel proceeds with detach or disable.

For most drivers, this is fine. The driver's detach path handles any in-flight work, so there is nothing the quiesce would do that detach does not also do.

### 何时应该实现它

A driver implements `DEVICE_QUIESCE` explicitly when:

1. **Returning EBUSY is more informative than waiting.** If the driver has a concept of "busy" (a transfer in flight, an open file descriptor count, a filesystem mount), and the user can wait for it to become non-busy, the driver might refuse quiesce until busy is zero. `DEVICE_QUIESCE` returning EBUSY tells the user "the device is busy; wait and retry".

2. **The quiesce can be done faster than a full detach.** If detach is expensive (frees large resource tables, drains slow queues) but the device can be stopped cheaply, `DEVICE_QUIESCE` lets the kernel probe for readiness without paying detach's cost.

3. **The driver wants to distinguish quiesce from suspend.** If the driver wants to stop activity but not save state (because no resume is coming), implementing quiesce separately from suspend is a way to express that distinction in code.

For the `myfirst` driver, none of these apply. The Chapter 21 detach path already handles in-flight work; the Chapter 22 suspend path handles quiesce in the power-management sense. Adding a separate `DEVICE_QUIESCE` would be redundant.

### 来自bce(4)的示例

The Broadcom NetXtreme driver in `/usr/src/sys/dev/bce/if_bce.c` has a commented-out `DEVMETHOD(device_quiesce, bce_quiesce)` entry in its method table. The comment suggests the author considered implementing quiesce but did not. This is common: many drivers keep the line commented as a TODO that never gets implemented, because the default handles their use case.

The implementation, if the driver enabled it, would stop the NIC's TX and RX paths without freeing the hardware resources. A subsequent `device_detach` would then do the actual freeing. The split between "stop" and "free" is what `DEVICE_QUIESCE` would express.

### 与DEVICE_SUSPEND的关系

`DEVICE_QUIESCE` and `DEVICE_SUSPEND` do similar things: they stop the device's activity. The differences:

- **Lifecycle**: quiesce is between run and detach; suspend is between run and eventual resume.
- **Resources**: quiesce does not require the driver to save any state; suspend does.
- **Ability to veto**: both can return EBUSY; the consequences differ (quiesce prevents detach; suspend prevents the power transition).

A driver that implements both usually shares code: `foo_quiesce` might do "stop activity" and `foo_suspend` might do "call quiesce; save state; return". The `myfirst` driver's `myfirst_quiesce` helper is the shared code; the chapter does not wire it to a `DEVICE_QUIESCE` method, but doing so would be a small addition.

### myfirst的可选添加

As a challenge, the reader can add `DEVICE_QUIESCE` to `myfirst`:

```c
static int
myfirst_pci_quiesce(device_t dev)
{
        struct myfirst_softc *sc = device_get_softc(dev);

        device_printf(dev, "quiesce: starting\n");
        (void)myfirst_quiesce(sc);
        atomic_add_64(&sc->power_quiesce_count, 1);
        device_printf(dev, "quiesce: complete\n");
        return (0);
}
```

And the matching method-table entry:

```c
DEVMETHOD(device_quiesce, myfirst_pci_quiesce),
```

Testing it: `devctl detach myfirst0` calls quiesce before detach; the reader can verify by reading `dev.myfirst.0.power_quiesce_count` immediately before the detach takes effect.

The challenge is short and does not change the driver's overall structure; it just wires one more method. Chapter 22's consolidated Stage 4 does not include it by default, but the reader who wants the method can add it in a few lines.



## Hands-On Labs

Chapter 22 includes three hands-on labs that exercise the power-management path in progressively harder ways. Each lab has a script in `examples/part-04/ch22-power/labs/` that the reader can run as-is, plus extension ideas.

### 实验1：单周期挂起-恢复

The first lab is the simplest: one clean suspend-resume cycle with counter verification.

**Setup.** Load the Chapter 22 Stage 4 driver:

```sh
cd examples/part-04/ch22-power/stage4-final
make clean && make
sudo kldload ./myfirst.ko
```

Verify attach:

```sh
sysctl dev.myfirst.0.%driver
# Should return: myfirst
sysctl dev.myfirst.0.suspended
# Should return: 0
```

**Run.** Execute the cycle script:

```sh
sudo sh ../labs/ch22-suspend-resume-cycle.sh
```

Expected output:

```text
PASS: one suspend-resume cycle completed cleanly
```

**Verify.** Inspect the counters:

```sh
sysctl dev.myfirst.0.power_suspend_count
# Should return: 1
sysctl dev.myfirst.0.power_resume_count
# Should return: 1
```

Check `dmesg`:

```sh
dmesg | tail -6
```

Should show four lines (suspend start, suspend complete, resume start, resume complete) plus the pre-and-post transfer log lines.

**Extension.** Modify the cycle script to run two suspend-resume cycles instead of one, and verify that the counters increment by exactly 2 each.

### 实验2：百周期压力测试

The second lab runs the cycle script one hundred times in a row and checks that nothing drifts.

**Run.**

```sh
sudo sh ../labs/ch22-suspend-stress.sh
```

Expected output after a few seconds:

```text
PASS: 100 cycles
```

**Verify.** After the stress run, the counters should each be 100 (or 100 plus whatever was there before):

```sh
sysctl dev.myfirst.0.power_suspend_count
# 100 (or however many cycles were added)
```

**Observations to make.**

- How long does one cycle take? On the simulation, it should be a few milliseconds. On real hardware with D-state transitions, expect a few hundred microseconds to a few milliseconds.
- Does the system's load average change during the stress? The simulation is cheap; a hundred cycles on a modern machine should barely register.
- What happens if you run the DMA test during the stress? (`sudo sysctl dev.myfirst.0.dma_test_read=1` concurrently with the cycle loop.) A well-written driver should handle this gracefully; the DMA test succeeds if it happens during a `RUNNING` window and fails with EBUSY or similar if it happens during a transition.

**Extension.** Run the stress script with `dmesg -c` before to clear the log, then afterwards:

```sh
dmesg | wc -l
```

Should be close to 400 (four log lines per cycle, times 100 cycles). A log-line-per-cycle count lets you verify that every cycle actually executed through the driver.

### 实验3：跨周期传输

The third lab is the hardest: it starts a DMA transfer and immediately suspends in the middle of it, then resumes and verifies that the driver recovers.

**Setup.** The lab script is `ch22-transfer-across-cycle.sh`. It runs a DMA transfer in the background, sleeps a few milliseconds, calls `devctl suspend`, sleeps, calls `devctl resume`, and then starts another transfer.

**Run.**

```sh
sudo sh ../labs/ch22-transfer-across-cycle.sh
```

**Observations to make.**

- Does the first transfer complete, error out, or time out? The expected behavior is that the quiesce aborts it cleanly; the transfer reports EIO or ETIMEDOUT.
- Does the counter `dma_errors` or `dma_timeouts` increment? One of them should.
- Does `dma_in_flight` go back to false after the suspend?
- Does the post-resume transfer succeed normally? If yes, the driver's state is consistent and the cycle worked.

**Extension.** Reduce the sleep between the transfer start and the suspend to hit the corner case where the transfer is mid-execution at the moment of the suspend. That is where race conditions live; a driver that passes this test under aggressive timing has a solid quiesce implementation.

### 实验4：运行时PM（可选）

For readers building with `MYFIRST_ENABLE_RUNTIME_PM`, a fourth lab exercises the runtime-PM path.

**Setup.** Rebuild with runtime PM enabled:

```sh
cd examples/part-04/ch22-power/stage4-final
# Uncomment the CFLAGS line in the Makefile:
#   CFLAGS+= -DMYFIRST_ENABLE_RUNTIME_PM
make clean && make
sudo kldload ./myfirst.ko
```

**Run.**

```sh
sudo sh ../labs/ch22-runtime-pm.sh
```

The script:

1. Sets the idle threshold to 3 seconds (instead of the default 5).
2. Records baseline counters.
3. Waits 5 seconds without any activity.
4. Verifies `runtime_state` is `RUNTIME_SUSPENDED`.
5. Triggers a DMA transfer.
6. Verifies `runtime_state` is back to `RUNNING`.
7. Prints PASS.

**Observations to make.**

- During the idle wait, `dmesg` should show the "runtime suspend" log line approximately 3 seconds in.
- The `runtime_suspend_count` and `runtime_resume_count` should each be 1 at the end.
- The DMA transfer should succeed normally after the runtime resume.

**Extension.** Set the idle threshold to 1 second. Run the DMA test repeatedly in a tight loop. You should see no runtime-suspend transitions during the loop (because each test resets the idle timer), but as soon as the loop stops, the runtime suspend fires.

### 实验说明

All of the labs assume the driver is loaded and the system is idle enough that transitions happen on-demand. If another process is actively using the device (unlikely for `myfirst`, but common in real setups), the counters drift by unexpected amounts and the scripts' exact-increment checks fail. The scripts are designed for a quiet test environment, not a noisy one.

For realistic testing of the `re(4)` driver or other production drivers, the same script structure applies with the device name adjusted. The `devctl suspend`/`devctl resume` dance works for any PCI device the kernel manages.



## Challenge Exercises

The Chapter 22 challenge exercises push the reader beyond the baseline driver into territory that real-world drivers eventually have to handle. Each exercise is scoped to be achievable with the chapter's material and a few hours of work.

### 挑战1：实现sysctl唤醒机制

Extend the `myfirst` driver with a simulated wake source. The simulation already has a callout that can fire; add a new simulation feature that sets a "wake" bit on the device while it is in D3, and have the driver's `DEVICE_RESUME` path log the wake event.

**Hints.**

- Add a `MYFIRST_REG_WAKE_STATUS` register to the simulation backend.
- Add a `MYFIRST_REG_WAKE_ENABLE` register the driver writes during suspend.
- Have the simulation callout set the wake status bit after a random delay.
- On resume, the driver reads the register and logs whether a wake was observed.

**Verification.** After `devctl suspend; sleep 1; devctl resume`, the log should show the wake status. A follow-up `sysctl dev.myfirst.0.wake_events` should increment.

**Why this matters.** Wake source handling is one of the trickiest parts of real-hardware power management. Building it into the simulation lets the reader exercise the full contract without needing hardware.

### 挑战2：保存和恢复描述符环

The Chapter 21 simulation does not yet use a descriptor ring (transfers are one-at-a-time). Extend the simulation with a small descriptor ring, program its base address through a register at attach, and have the suspend path save the ring's base address into softc state. Have the resume path write the saved base address back.

**Hints.**

- The ring's base address is a `bus_addr_t` held in the softc.
- The register is `MYFIRST_REG_RING_BASE_LOW`/`_HIGH`.
- Saving and restoring is trivial; the point is to verify that *not* saving and restoring would break things.

**Verification.** After suspend-resume, the ring base register should hold the same value as before. Without the restore, it should hold zero.

**Why this matters.** Descriptor rings are what real high-throughput drivers use; a power-aware driver with a ring has to restore the base address on every resume. This exercise is a stepping stone to the kind of state management that production drivers like `re(4)` and `em(4)` perform.

### 挑战3：实现否决策略

Extend the suspend path with a policy knob that lets the user specify whether the driver should veto a suspend when the device is busy. Specifically:

- Add `dev.myfirst.0.suspend_veto_if_busy` as a read-write sysctl.
- If the sysctl is 1 and a DMA transfer is in flight, `myfirst_power_suspend` returns EBUSY without quiescing.
- If the sysctl is 0 (default), suspend always succeeds.

**Hints.** Set `suspend_veto_if_busy` to 1. Start a long DMA transfer (add a `DELAY` to the simulation's engine to make it last a second or two). Call `devctl suspend myfirst0` during the transfer. Verify that the suspend returns an error and `dev.myfirst.0.suspended` stays 0.

**Verification.** The kernel's unwind path runs; the driver is still in `RUNNING`; the transfer completes normally.

**Why this matters.** Vetoing is an effective tool and a dangerous one. Real-world policy decisions about whether to veto are nuanced (storage drivers often veto; NIC drivers usually do not). Implementing the mechanism makes the policy question tangible.

### 挑战4：添加恢复后自检

After resume, do a minimum-viable test of the device: write a known pattern to the DMA buffer, trigger a write transfer, read it back with a read transfer, and verify. If the test fails, mark the device broken and fail subsequent operations.

**Hints.**

- Add the self-test as a helper that runs from `myfirst_power_resume` after `myfirst_restore`.
- Use a well-known pattern like `0xDEADBEEF`.
- Use the existing DMA path; the self-test is just one write and one read.

**Verification.** Under normal operation, the self-test always passes. To verify it catches failures, add an artificial "fail once" mechanism to the simulation and trigger it; the driver should log the failure and mark itself broken.

**Why this matters.** Self-tests are a lightweight form of reliability engineering. A driver that catches its own failures at well-defined points is easier to debug than one that silently corrupts data until a user notices.

### 挑战5：实现手动pci_save_state/pci_restore_state

Most drivers let the PCI layer handle config-space save-and-restore automatically. Extend the Chapter 22 driver to optionally do it manually, gated by a sysctl `dev.myfirst.0.manual_pci_save`.

**Hints.**

- Read `hw.pci.do_power_suspend` and `hw.pci.do_power_resume` and set them to 0 when manual mode is enabled.
- Call `pci_save_state` explicitly in the suspend path, `pci_restore_state` in the resume path.
- Verify that the device still works after suspend-resume.

**Verification.** The device should function identically whether or not manual mode is enabled. Set the sysctl before a stress test and verify no drift.

**Why this matters.** Some real drivers need manual save/restore because the PCI layer's automatic handling interferes with device-specific quirks. Knowing when and how to take over the save/restore is a useful intermediate skill.



## Troubleshooting Reference

This section collects the common problems a reader may encounter while working through Chapter 22, with a short diagnostic and fix for each. The list is meant to be skimmable; if a problem matches, skip to the corresponding entry.

### "devctl: DEV_SUSPEND failed: Operation not supported"

The driver does not implement `DEVICE_SUSPEND`. Either the method table is missing the `DEVMETHOD(device_suspend, ...)` line, or the driver has not been rebuilt and reloaded.

**Fix.** Check the method table. Rebuild with `make clean && make`. Unload and reload.

### "devctl: DEV_SUSPEND failed: Device busy"

The driver returned `EBUSY` from `DEVICE_SUSPEND`, probably because of the veto logic from Challenge 3, or because the device is genuinely busy (DMA in flight, task running) and the driver chose to veto.

**Fix.** Check whether the `suspend_veto_if_busy` knob is set. Check `dma_in_flight`. Wait for activity to complete before suspending.

### "devctl: DEV_RESUME failed"

`DEVICE_RESUME` returned non-zero. The log should have more detail.

**Fix.** Check `dmesg | tail`. The resume log line should tell you what failed. Usually it is a hardware-specific init step that did not succeed.

### Device is suspended but `dev.myfirst.0.suspended` reads 0

The driver's flag is out of sync with the kernel's state. Probably a bug in the quiesce path: the flag was never set, or was cleared prematurely.

**Fix.** Add a `KASSERT(sc->suspended == true)` at the top of the resume path; run under `INVARIANTS` to catch the bug.

### `power_suspend_count != power_resume_count`

A cycle got one side but not the other. Check `dmesg` for errors; the log should show where the sequence broke.

**Fix.** Fix the code path that is missing. Usually an early return without the counter update.

### DMA transfers fail after resume

The restore path did not reinitialise the DMA engine. Check the INTR_MASK register, the DMA control registers, the `saved_intr_mask` value. Enable verbose logging to see the resume path's restoration sequence.

**Fix.** Add a missing register write to `myfirst_restore`.

### WITNESS complains about a lock held during suspend

The suspend path acquired a lock and then called a function that sleeps or tries to acquire another lock. Read the WITNESS message for the offending lock names.

**Fix.** Drop the lock before the sleeping call, or restructure the code so the lock is acquired only when needed.

### System does not wake from S3

A driver below `myfirst` is blocking resume. Unlikely to be `myfirst` itself unless the logs show an error from the driver specifically.

**Fix.** Boot into single-user mode, or load fewer drivers, and bisect. Check `dmesg` in the live system for the offending driver.

### Runtime PM never fires

The idle watcher callout is not running, or the `last_activity` timestamp is being updated too often.

**Fix.** Verify `callout_reset` is being called from the attach path. Verify `myfirst_mark_active` is not being called from unexpected code paths. Add logging to the callout callback to confirm it fires.

### Kernel panic during suspend

A KASSERT failed (on an `INVARIANTS` kernel) or a lock is held incorrectly. The panic message identifies the offending file and line.

**Fix.** Read the panic message. Match the file and line to the code. The fix is usually straightforward once the location is identified.



## Wrapping Up

Chapter 22 closes Part 4 by giving the `myfirst` driver the discipline of power management. At the start, `myfirst` at version `1.4-dma` was a capable driver: it attached to a PCI device, handled multi-vector interrupts, moved data through DMA, and cleaned up its resources on detach. What it lacked was the ability to participate in the system's power transitions. It would crash, leak, or silently fail if the user closed the laptop lid or asked the kernel to suspend the device. At the end, `myfirst` at version `1.5-power` handles every power transition the kernel can throw at it: system suspend to S3 or S4, per-device suspend through `devctl`, system shutdown, and optional runtime power management.

The eight sections walked the full progression. Section 1 established the big picture: why a driver cares about power, what ACPI S-states and PCI D-states are, what PCIe L-states and ASPM add, and what wake sources look like. Section 2 introduced FreeBSD's concrete APIs: the `DEVICE_SUSPEND`, `DEVICE_RESUME`, `DEVICE_SHUTDOWN`, and `DEVICE_QUIESCE` methods, the `bus_generic_suspend` and `bus_generic_resume` helpers, and the PCI layer's automatic config-space save and restore. The Stage 1 skeleton made the methods log and count transitions without doing any real work. Section 3 turned the suspend skeleton into a real quiesce: mask interrupts, drain DMA, drain workers, in that order, with helper functions shared between suspend and detach. Section 4 wrote the matching resume path: re-enable bus-master, restore device-specific state, clear the suspended flag, unmask interrupts. Section 5 added optional runtime power management with an idle-watcher callout and explicit `pci_set_powerstate` transitions. Section 6 surveyed the user-space interface: `acpiconf`, `zzz`, `devctl suspend`, `devctl resume`, `devinfo -v`, and the matching sysctls. Section 7 catalogued the characteristic failure modes and their debugging patterns. Section 8 refactored the code into `myfirst_power.c`, bumped the version to `1.5-power`, added `POWER.md`, and wired the final regression test.

What Chapter 22 did not do is scatter-gather power management for multi-queue drivers (that is a Part 6 topic, Chapter 28), hotplug and surprise-removal integration (a Part 7 topic), embedded-platform power domains (Part 7 again), or the internals of ACPI's AML interpreter (never covered in this book). Each of those is a natural extension built on Chapter 22's primitives, and each belongs in a later chapter where the scope matches. The foundation is in place; the specialisations add vocabulary without needing a new foundation.

The file layout has grown: 16 source files (including `cbuf`), 8 documentation files (`HARDWARE.md`, `LOCKING.md`, `SIMULATION.md`, `PCI.md`, `INTERRUPTS.md`, `MSIX.md`, `DMA.md`, `POWER.md`), and an extended regression suite that covers every subsystem. The driver is structurally parallel to production FreeBSD drivers; a reader who has worked through Chapters 16 through 22 can open `if_re.c`, `if_xl.c`, or `virtio_blk.c` and recognise every architectural part: register accessors, simulation backend, PCI attach, interrupt filter and task, per-vector machinery, DMA setup and teardown, sync discipline, power suspend, power resume, clean detach.

### A Reflection Before Chapter 23

Chapter 22 is the last chapter of Part 4, and Part 4 is the part that taught the reader how a driver talks to hardware. Chapters 16 through 21 introduced the primitives: MMIO, simulation, PCI, interrupts, multi-vector interrupts, DMA. Chapter 22 introduced the discipline: how those primitives survive power transitions. Together, the seven chapters take the reader from "no idea what a driver is" to "a working multi-subsystem driver that handles every hardware event the kernel can throw at it".

Chapter 22's teaching generalises. A reader who has internalised the suspend-quiesce-save-restore pattern, the interaction between driver and PCI layer, the runtime-PM state machine, and the debugging patterns will find similar shapes in every power-aware FreeBSD driver. The specific hardware differs; the structure does not. A driver for a NIC, a storage controller, a GPU, or a USB host controller applies the same vocabulary to its own hardware.

Part 5, which begins with Chapter 23, shifts focus. Part 4 was about the driver-to-hardware direction: how the driver talks to the device. Part 5 is about the driver-to-kernel direction: how the driver is debugged, traced, tooled, and stressed by the humans who maintain it. Chapter 23 starts that shift with debugging and tracing techniques that apply across every driver subsystem.

### What to Do If You Are Stuck

Three suggestions.

First, focus on the Stage 2 suspend and Stage 3 resume paths. If `devctl suspend myfirst0` followed by `devctl resume myfirst0` succeeds and a subsequent DMA transfer works, the core of the chapter is working. Every other piece of the chapter is optional in the sense that it decorates the pipeline, but if the pipeline fails, the chapter is not working and Section 3 or Section 4 is the right place to diagnose.

Second, open `/usr/src/sys/dev/re/if_re.c` and re-read `re_suspend`, `re_resume`, and `re_setwol`. Each function is about thirty lines. Every line maps to a Chapter 22 concept. Reading them once after completing the chapter should feel like familiar territory; the real driver's patterns will look like elaborations of the chapter's simpler ones.

Third, skip the challenges on the first pass. The labs are calibrated for Chapter 22's pace; the challenges assume the chapter's material is solid. Come back to them after Chapter 23 if they feel out of reach now.

Chapter 22's goal was to give the driver power-management discipline. If it has, Chapter 23's debugging and tracing machinery becomes a generalisation of what you already do instinctively rather than a new topic.

## 第4部分检查点

Part 4 has been the longest and densest stretch of the book so far. Seven chapters covered hardware resources, register I/O, PCI attach, interrupts, MSI and MSI-X, DMA, and power management. Before Part 5 changes the mode from "writing drivers" to "debugging and tracing them," confirm that the hardware-facing story is internalized.

By the end of Part 4 you should be able to do each of the following without searching:

- Claim a hardware resource with `bus_alloc_resource_any` or `bus_alloc_resource_anywhere`, access it through the `bus_space(9)` read/write and barrier primitives, and release it cleanly in detach.
- Read and write device registers through the `bus_space(9)` abstraction rather than raw pointer dereferences, with correct barrier discipline around sequences that must not be reordered.
- Match a PCI device through vendor, device, subvendor, and subdevice IDs; claim its BARs; and survive a forced detach without leaking resources.
- Register a top-half filter together with a bottom-half task or ithread via `bus_setup_intr`, in the order the kernel requires, and tear them down in reverse order under detach.
- Set up MSI or MSI-X vectors with a graceful fallback ladder from MSI-X to MSI to legacy INTx, and bind vectors to specific CPUs when the workload calls for it.
- Allocate, map, sync, and release DMA buffers using `bus_dma(9)` including the bounce-buffer case.
- Implement `device_suspend` and `device_resume` with register save and restore, I/O quiescing, and a post-resume self-test.

If any of those still requires a lookup, the labs to revisit are:

- Registers and barriers: Lab 1 (Observe the Register Dance) and Lab 8 (The Watchdog-Meets-Register Scenario) in Chapter 16.
- Simulated hardware under load: Lab 6 (Inject Stuck-Busy and Watch the Driver Wait) and Lab 10 (Inject a Memory-Corruption Attack) in Chapter 17.
- PCI attach and detach: Lab 4 (Claim the BAR and Read a Register) and Lab 5 (Exercise the cdev and Verify Detach Cleanup) in Chapter 18.
- Interrupt handling: Lab 3 (Stage 2, Real Filter and Deferred Task) in Chapter 19.
- MSI and MSI-X: Lab 4 (Stage 3, MSI-X With CPU Binding) in Chapter 20.
- DMA: Lab 4 (Stage 3, Interrupt-Driven Completion) and Lab 5 (Stage 4, Refactor and Regression) in Chapter 21.
- Power management: Lab 2 (One-Hundred-Cycle Stress) and Lab 3 (Transfer Across a Cycle) in Chapter 22.

Part 5 will expect the following as a baseline:

- A hardware-capable driver with observability already baked in: counters, sysctls, and `devctl_notify` calls at the important transitions. Chapter 23's debugging machinery works best when the driver already reports on itself.
- A regression script that can cycle the driver reliably, since Part 5 turns reproducibility into a first-class skill.
- A kernel built with `INVARIANTS` and `WITNESS`. Part 5 leans on both even more heavily than Part 4, especially in Chapter 23.
- The understanding that a bug in driver code is a bug in kernel code, which means user-space debuggers alone will not be enough and Part 5 will teach the kernel-space tools.

If those hold, Part 5 is ready for you. If one still looks shaky, a short lap through the relevant lab will pay back its time several times over.

## 通往第23章的桥梁

Chapter 23 is titled *Debugging and Tracing*. Its scope is the professional practice of finding bugs in drivers: tools like `ktrace`, `ddb`, `kgdb`, `dtrace`, and `procstat`; techniques for analysing panics, deadlocks, and data corruption; strategies for turning vague user reports into reproducible test cases; and the mindset of a driver developer who has to debug code running in kernel space with limited visibility.

Chapter 22 prepared the ground in four specific ways.

First, **you have observability counters everywhere**. The Chapter 22 driver exposes suspend, resume, shutdown, and runtime-PM counters through sysctls. Chapter 23's debugging techniques rely on observability; a driver that already tracks its own state is much easier to debug than one that does not.

Second, **you have a regression test**. The cycle and stress scripts from Section 6 are a first taste of what Chapter 23 expands: the ability to reproduce a bug on demand. A bug you cannot reproduce is a bug you cannot fix; Chapter 22's scripts are a foundation for the heavier testing Chapter 23 adds.

Third, **you have a working INVARIANTS / WITNESS debug kernel**. Chapter 22 leaned on both throughout; Chapter 23 builds on the same kernel for `ddb` sessions, post-mortem analysis, and kernel-crash reproduction.

Fourth, **you understand that bugs in driver code are bugs in kernel code**. Chapter 22 ran into hangs, frozen devices, lost interrupts, and WITNESS complaints. Each of those is a kernel bug in the user-visible sense; each requires a kernel-space debugging approach. Chapter 23 teaches that approach systematically.

Specific topics Chapter 23 will cover:

- Using `ktrace` and `kdump` to observe a process's system call trace in real time.
- Using `ddb` to break into the kernel debugger for post-mortem analysis or live inspection.
- Using `kgdb` with a core dump to recover the state of a crashed kernel.
- Using `dtrace` for in-kernel tracing without modifying the source.
- Using `procstat`, `top`, `pmcstat`, and related tools for performance observation.
- Strategies for minimising a bug: shrinking a reproducer, bisecting a regression, hypothesising and testing.
- Patterns for instrumenting a driver in production without disturbing behaviour.

You do not need to read ahead. Chapter 22 is sufficient preparation. Bring your `myfirst` driver at `1.5-power`, your `LOCKING.md`, your `INTERRUPTS.md`, your `MSIX.md`, your `DMA.md`, your `POWER.md`, your `WITNESS`-enabled kernel, and your regression script. Chapter 23 starts where Chapter 22 ended.

Part 4 is complete. Chapter 23 opens Part 5 by adding the observability and debugging discipline that separates a driver you wrote last week from a driver you can maintain for years.

The vocabulary is yours; the structure is yours; the discipline is yours. Chapter 23 adds the next missing piece: the ability to find and fix bugs that only show up in production.



## Reference: 第22章快速参考卡

A compact summary of the vocabulary, APIs, flags, and procedures Chapter 22 introduced.

### Vocabulary

- **Suspend:** a transition from D0 (full operation) to a lower-power state from which the device can be brought back.
- **Resume:** the transition back from the lower-power state to D0.
- **Shutdown:** the transition to a final state from which the device will not return.
- **Quiesce:** to bring a device to a state with no activity and no pending work.
- **System sleep state (S0, S1, S3, S4, S5):** ACPI-defined levels of system power.
- **Device power state (D0, D1, D2, D3hot, D3cold):** PCI-defined levels of device power.
- **Link state (L0, L0s, L1, L1.1, L1.2, L2):** PCIe-defined levels of link power.
- **ASPM (Active-State Power Management):** automatic transitions between L0 and L0s/L1.
- **PME# (Power Management Event):** a signal a device asserts when it wants to wake the system.
- **Wake source:** a mechanism by which a suspended device can request wakeup.
- **Runtime PM:** device-level power saving while the system stays in S0.

### Essential Kobj Methods

- `DEVMETHOD(device_suspend, foo_suspend)`: called to quiesce the device before a power transition.
- `DEVMETHOD(device_resume, foo_resume)`: called to restore the device after the power transition.
- `DEVMETHOD(device_shutdown, foo_shutdown)`: called to leave the device in a safe state for reboot.
- `DEVMETHOD(device_quiesce, foo_quiesce)`: called to stop activity without tearing down resources.

### Essential PCI APIs

- `pci_has_pm(dev)`: true if the device has a power-management capability.
- `pci_set_powerstate(dev, state)`: transition to `PCI_POWERSTATE_D0`, `D1`, `D2`, or `D3`.
- `pci_get_powerstate(dev)`: current power state.
- `pci_save_state(dev)`: cache the configuration space.
- `pci_restore_state(dev)`: write the cached configuration space back.
- `pci_enable_pme(dev)`: enable PME# generation.
- `pci_clear_pme(dev)`: clear pending PME status.
- `pci_enable_busmaster(dev)`: re-enable bus-master after a reset.

### Essential Bus Helpers

- `bus_generic_suspend(dev)`: suspend all children in reverse order.
- `bus_generic_resume(dev)`: resume all children in forward order.
- `device_quiesce(dev)`: call the driver's `DEVICE_QUIESCE`.

### Essential Sysctls

- `hw.acpi.supported_sleep_state`: list of S-states the platform supports.
- `hw.acpi.suspend_state`: default S-state for `zzz`.
- `hw.pci.do_power_suspend`: automatic D0->D3 transition on suspend.
- `hw.pci.do_power_resume`: automatic D3->D0 transition on resume.
- `dev.N.M.suspended`: driver's own suspended flag.
- `dev.N.M.power_suspend_count`, `power_resume_count`, `power_shutdown_count`.
- `dev.N.M.runtime_state`, `runtime_suspend_count`, `runtime_resume_count`.

### Useful Commands

- `acpiconf -s 3`: enter S3.
- `zzz`: wrapper around `acpiconf`.
- `devctl suspend <device>`: per-device suspend.
- `devctl resume <device>`: per-device resume.
- `devinfo -v`: device tree with state.
- `pciconf -lvbc`: PCI devices with power state.
- `sysctl -a | grep acpi`: all ACPI-related variables.

### Common Procedures

**Method table addition:**

```c
DEVMETHOD(device_suspend,  foo_suspend),
DEVMETHOD(device_resume,   foo_resume),
DEVMETHOD(device_shutdown, foo_shutdown),
```

**Suspend skeleton:**

```c
int foo_suspend(device_t dev) {
    struct foo_softc *sc = device_get_softc(dev);
    FOO_LOCK(sc);
    sc->suspended = true;
    FOO_UNLOCK(sc);
    foo_mask_interrupts(sc);
    foo_drain_dma(sc);
    foo_drain_workers(sc);
    return (0);
}
```

**Resume skeleton:**

```c
int foo_resume(device_t dev) {
    struct foo_softc *sc = device_get_softc(dev);
    pci_enable_busmaster(dev);
    foo_restore_registers(sc);
    FOO_LOCK(sc);
    sc->suspended = false;
    FOO_UNLOCK(sc);
    foo_unmask_interrupts(sc);
    return (0);
}
```

**Runtime-PM helper:**

```c
int foo_runtime_suspend(struct foo_softc *sc) {
    foo_quiesce(sc);
    pci_save_state(sc->dev);
    return (pci_set_powerstate(sc->dev, PCI_POWERSTATE_D3));
}

int foo_runtime_resume(struct foo_softc *sc) {
    pci_set_powerstate(sc->dev, PCI_POWERSTATE_D0);
    pci_restore_state(sc->dev);
    return (foo_restore(sc));
}
```

### 需要收藏的文件

- `/usr/src/sys/kern/device_if.m`: the kobj method definitions.
- `/usr/src/sys/kern/subr_bus.c`: `bus_generic_suspend`, `bus_generic_resume`, `device_quiesce`.
- `/usr/src/sys/dev/pci/pci.c`: `pci_suspend_child`, `pci_resume_child`, `pci_save_state`, `pci_restore_state`.
- `/usr/src/sys/dev/pci/pcivar.h`: `PCI_POWERSTATE_*` constants and inline API.
- `/usr/src/sys/dev/re/if_re.c`: production reference for suspend/resume with WoL.
- `/usr/src/sys/dev/xl/if_xl.c`: minimal suspend/resume pattern.
- `/usr/src/sys/dev/virtio/block/virtio_blk.c`: virtio-style quiesce.



## Reference: 第22章术语表

本章新术语的简要词汇表。

- **ACPI（高级配置与电源接口）：** 操作系统与平台固件之间用于电源管理的工业标准接口。
- **ASPM（活动状态电源管理）：** 自动PCIe链路状态转换。
- **D-state（D状态）：** 设备电源状态（D0到D3cold）。
- **DEVICE_QUIESCE：** 在不拆除资源的情况下停止活动的kobj方法。
- **DEVICE_RESUME：** 调用以将设备恢复到运行状态的kobj方法。
- **DEVICE_SHUTDOWN：** 系统关机时调用的kobj方法。
- **DEVICE_SUSPEND：** 在电源转换之前静默设备时调用的kobj方法。
- **GPE（通用事件）：** ACPI唤醒事件源。
- **L-state（L状态）：** PCIe链路电源状态。
- **链路状态机：** L0和L0s/L1之间的自动转换。
- **PME#（电源管理事件）：** 设备请求唤醒时断言的PCI信号。
- **电源管理能力：** 包含PM寄存器的PCI能力结构。
- **Quiesce（静默）：** 将设备带到没有活动和没有待处理工作的状态。
- **运行时PM：** 系统保持S0状态时的设备级节能。
- **S-state（S状态）：** ACPI系统睡眠状态（S0到S5）。
- **Shutdown（关机）：** 最终断电，通常导致重启或关机。
- **Sleep state（睡眠状态）：** 参见S-state。
- **Suspend（挂起）：** 临时断电，系统或设备可以从中恢复。
- **Suspended flag（挂起标志）：** 指示设备处于挂起状态的驱动本地标志。
- **Wake source（唤醒源）：** 挂起的系统或设备被唤醒的机制。
- **WoL（局域网唤醒）：** 由网络数据包触发的唤醒源。



## Reference: 关于电源管理理念的结语

本章的结尾段落。

电源管理是将驱动原型与生产驱动区分开来的纪律。在电源管理之前，驱动假设其设备始终开启且始终可用。在电源管理之后，驱动知道设备可以被置于睡眠状态，知道如何正确地将其置于睡眠状态，并且可以在真实用户运行的各种环境中被信任：每天开关数十次的笔记本电脑、挂起空闲设备以节能的服务器、在主机之间迁移的虚拟机、关闭整个电源域以延长电池寿命的嵌入式系统。

第22章的教训是，电源管理是有纪律的，而非魔法。FreeBSD内核给驱动一个特定的契约（四个kobj方法、调用顺序、与PCI层的交互），遵循契约是大部分工作。其余是硬件特定的：理解设备在D状态转换期间丢失哪些寄存器、硬件支持哪些唤醒源、驱动应该为运行时PM应用什么策略。模式在FreeBSD的每个电源感知驱动中都是相同的；将其内化一次将在后续几十个章节和数千行真实驱动代码中受益。

对于本章读者和本书的未来读者，第22章的电源管理模式是`myfirst`驱动架构的永久组成部分，也是读者工具箱中的永久工具。第23章假设它：调试驱动假设驱动具有第22章引入的可观测性计数器和结构化生命周期。第六部分的专业化章节假设它：每个生产级驱动都有电源路径。第七部分的性能章节（第33章）假设它：每个调优测量都必须考虑电源状态转换。词汇是每个生产FreeBSD驱动共享的词汇；模式是生产驱动赖以生存的模式；纪律是保持电源感知平台正确性的纪律。

第22章教授的技能不是"如何向单个PCI驱动添加挂起和恢复方法"。它是"如何将驱动的生命周期视为挂载、运行、静默、睡眠、唤醒、运行，最终分离，而不仅仅是挂载、运行、分离"。这项技能适用于读者将来工作的每个驱动。

第四部分完成。`myfirst`驱动处于`1.5-power`版本，在结构上与生产FreeBSD驱动平行，准备好迎接第五和第六部分中的调试、工具和专业章节。
