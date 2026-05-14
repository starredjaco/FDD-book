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

## 第4节：恢复时恢复状态

第3节给了驱动正确的挂起路径。第4节编写匹配的恢复。恢复是挂起的补集：挂起停止的一切，恢复重启；挂起保存的每个值，恢复写回；挂起设置的每个标志，恢复清除。顺序不是精确的镜像（恢复在不同的内核上下文中运行，PCI层已经做了一些工作，设备处于与挂起留下的不同状态），但内容一一对应。正确执行恢复意味着尊重PCI层已经部分完成的契约，并填补其余部分。

### PCI层已经做了什么

当内核在驱动上调用`DEVICE_RESUME`方法时，几件事已经发生：

1. CPU已退出S状态（从S3或S4恢复到S0）。
2. 内存已刷新，内核已重新建立自己的状态。
3. 父总线已恢复。对于`myfirst`，这意味着PCI总线驱动已经处理了主机桥和PCIe根复合体。
4. PCI层已在设备上调用`pci_set_powerstate(dev, PCI_POWERSTATE_D0)`，将其从任何低功耗状态（通常是D3hot）转回全功率。
5. PCI层已调用`pci_cfg_restore(dev, dinfo)`，将缓存的配置空间值（BAR、命令寄存器、缓存行大小等）写回设备。
6. PCI层已调用`pci_clear_pme(dev)`以清除任何待处理的电源管理事件位。
7. MSI或MSI-X配置（作为缓存状态的一部分）已恢复。驱动的中断向量可再次使用。

此时PCI总线驱动调用`myfirst`的`DEVICE_RESUME`。设备处于D0，其BAR已映射，其MSI/MSI-X表已恢复，其通用PCI状态完好。驱动需要恢复的是PCI层不知道的设备特定状态：驱动在挂载期间或之后写入的BAR局部寄存器。

对于`myfirst`模拟，相关的BAR局部寄存器是中断掩码（挂起路径特意设置为全掩码）和DMA寄存器（可能已处于中止状态）。驱动需要将它们恢复为反映正常操作的值。

### 恢复纪律

正确的恢复路径按顺序做四件事：

1. **重新启用总线主控**，以防配置空间恢复未这样做或PCI层的自动恢复被禁用。这是`pci_enable_busmaster(dev)`。在现代FreeBSD上通常是冗余的但无害；旧代码路径或有缺陷的BIOS有时会让总线主控保持禁用。防御性调用成本低。

2. **恢复驱动在挂起期间保存的任何设备特定状态**。对于`myfirst`，这意味着将`saved_intr_mask`写回INTR_MASK寄存器。真实驱动还会恢复供应商特定配置位、DMA引擎编程、硬件定时器等。

3. **取消掩码中断并清除挂起标志**，使设备可以恢复活动。这是转折点：在此之前，设备仍然安静；在此之后，设备可以引发中断并接受工作。

4. **记录转换并更新计数器**，用于可观测性和回归测试。

以下是代码中的模式：

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

助手`myfirst_restore`执行三个实际步骤：

```c
static int
myfirst_restore(struct myfirst_softc *sc)
{
        /* 步骤1：重新启用总线主控（防御性）。 */
        pci_enable_busmaster(sc->dev);

        /* 步骤2：恢复设备特定状态。
         *
         * 对于myfirst，这只是中断掩码。真实驱动
         * 会恢复更多：DMA引擎编程、硬件定时器、
         * 供应商特定配置等。
         */
        if (sc->saved_intr_mask == 0xFFFFFFFF) {
                /*
                 * 挂起保存了一个全掩码的掩码，这意味着
                 * 驱动不知道掩码应该是什么。使用
                 * 默认值：启用DMA完成，禁用其他
                 * 所有内容。
                 */
                sc->saved_intr_mask = ~MYFIRST_INTR_COMPLETE;
        }
        CSR_WRITE_4(sc->dev, MYFIRST_REG_INTR_MASK, sc->saved_intr_mask);

        /* 步骤3：清除挂起标志并取消掩码设备。 */
        MYFIRST_LOCK(sc);
        sc->suspended = false;
        MYFIRST_UNLOCK(sc);

        return (0);
}
```

该函数返回0，因为上述步骤在`myfirst`模拟中不会失败。真实驱动会检查其硬件初始化调用的返回值并传播任何错误。

### 为什么pci_enable_busmaster很重要

总线主控是PCI命令寄存器中的一个位，控制设备是否可以发起DMA事务。没有它，设备无法读写主机内存；任何DMA触发都会被PCI主机桥静默忽略。

第18章在挂载期间启用了总线主控。PCI层的自动配置空间恢复将命令寄存器写回其保存的值，其中包括总线主控位。因此原则上驱动不需要在恢复时再次调用`pci_enable_busmaster`。实际上，可能出几件事：

- 平台固件可能在唤醒设备时重置命令寄存器。
- `hw.pci.do_power_suspend` sysctl可能为0，此时PCI层不保存和恢复配置空间。
- 设备特定的怪癖可能在D3到D0转换时清除总线主控作为副作用。

在恢复中无条件防御性地调用`pci_enable_busmaster`是低成本的安全网。几个生产FreeBSD驱动遵循此模式；`if_re.c`的恢复路径是一个例子。该调用是幂等的：如果总线主控已开启，调用只是重新断言它。

### 恢复设备特定状态

`myfirst`模拟没有太多驱动需要手动恢复的状态。BAR局部寄存器有：

- 中断掩码（从`saved_intr_mask`恢复）。
- 中断状态位（在挂起期间已清除；它们应保持清除直到新活动到达）。
- DMA引擎寄存器（DMA_ADDR_LOW、DMA_ADDR_HIGH、DMA_LEN、DMA_DIR、DMA_CTRL、DMA_STATUS）。这些是瞬态的：它们持有当前传输的参数。恢复后没有传输在进行中，所以值不重要；下一次传输会覆盖它们。

真实驱动会更多。考虑几个例子：

- 存储驱动可能有一个DMA描述符环，其基址是设备在挂载期间学到的。恢复后，持有该基址的BAR级寄存器可能已被重置；驱动需要重新编程它。
- 网络驱动可能有编程到设备寄存器中的过滤表（MAC地址、多播列表、VLAN标签）。恢复后，这些表可能为空；驱动从softc侧副本重建它们。
- GPU驱动可能有显示定时、色彩表、硬件光标的寄存器状态。恢复后，驱动恢复活动模式。

对于`myfirst`，中断掩码是需要恢复的唯一BAR局部状态。上面展示的模式是真实驱动会针对其设备调整的模板。

### 恢复后验证设备身份

有些设备在挂起到D3cold的周期中完全重置。回来的设备功能上相同，但其整个状态已像刚上电一样重新初始化。假设什么都没改变的驱动会静默产生错误行为。

防御性的恢复路径可以通过读取已知寄存器值并与挂载时读取的值比较来检测这一点。对于PCI设备，配置空间中的供应商ID和设备ID始终相同（PCI层已恢复它们），但可以检查某些设备私有寄存器（修订ID、自检寄存器、固件版本）：

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

对于`myfirst`模拟，没有魔术寄存器（模拟在构建时未考虑恢复后验证）。想要作为挑战添加一个的读者可以扩展模拟后端的寄存器映射，添加一个只读`MAGIC`寄存器，并让驱动检查它。本章的实验3将其作为选项包含。

真正在D3cold中重置的设备的真实驱动需要此检查，因为没有它可能发生微妙的故障：驱动假设设备的内部状态机处于`IDLE`状态，但重置后状态机实际处于`RESETTING`状态。驱动发送的任何命令都被拒绝，驱动将拒绝解释为硬件故障，设备被标记为损坏。显式捕获重置并重建状态更干净。

### 检测和恢复设备重置

如果验证发现不匹配，驱动的恢复选项取决于硬件。对于`myfirst`模拟，最简单的响应是记录日志、标记设备为损坏、并使后续操作失败：

```c
if (myfirst_validate_device(sc) != 0) {
        MYFIRST_LOCK(sc);
        sc->broken = true;
        MYFIRST_UNLOCK(sc);
        return (EIO);
}
```

softc增加一个`broken`标志，任何面向用户的请求检查该标志并以错误失败。分离路径仍然有效（分离总是成功，即使设备损坏），所以用户可以卸载驱动并重新加载。

检测到重置的真实驱动有更多选项。网络驱动可能从`pci_alloc_msi`之后的点重新运行其挂载序列（该序列已被PCI层恢复）。存储驱动可能使用挂载使用的相同代码路径重新初始化其控制器。实现严重依赖于设备；模式是"检测，然后执行仍然需要的任何挂载时初始化"。

本章的`myfirst`驱动采取更简单的方法：它不为模拟实现重置检测，恢复路径默认不包含验证调用。上面的代码作为参考提供给想要作为练习扩展驱动的读者。

### 恢复DMA状态

第21章的DMA设置分配标签、分配内存、加载映射，并在softc中保留总线地址。这些都不在BAR局部寄存器映射中可见；DMA引擎仅在驱动作为启动传输的一部分将总线地址写入`DMA_ADDR_LOW`和`DMA_ADDR_HIGH`时才学习到它。

这意味着DMA状态不需要以"写入寄存器"的意义进行恢复。标签、映射和内存都是内核侧数据结构；它们在挂起中完好无损。下一次传输将作为正常提交的一部分编程DMA寄存器。

在真实设备上可能需要恢复的是：

- **DMA描述符环基址**，如果设备保持持久指针。真实NIC在挂载时写入一次基址寄存器并将设备指向描述符环；D3cold后，该寄存器可能已被重置，驱动必须重新编程它。
- **DMA引擎的启用位**，如果它与单个传输分开。
- **任何PCI层未缓存的每通道配置**（突发大小、优先级等）。

对于`myfirst`，这些都不适用。DMA引擎是按传输编程的。恢复不需要任何DMA特定的恢复，除了通用状态恢复已覆盖的内容。

### 重新装备中断

掩码中断是挂起的第2步。取消掩码是恢复的第3步。第3阶段恢复将`saved_intr_mask`写回`INTR_MASK`寄存器，它（按约定）将0写入对应启用向量的位，将1写入对应禁用向量的位。写入后，设备准备在有理由时在启用的向量上断言中断。

关于顺序有一个微妙之处。恢复路径在清除`suspended`标志之前取消掩码中断。这意味着一个非常不幸的中断可能在掩码清除后到达、调用过滤器、并发现`suspended == true`。过滤器会拒绝处理并返回`FILTER_STRAY`，这会使中断保持断言状态。

为避免这种情况，恢复路径在状态更改周围获取softc锁，并以相反顺序执行取消掩码和标志清除：先清除`suspended`，然后取消掩码。这样设备在掩码清除后引发的任何中断都会看到`suspended == false`并被正常处理。

前面代码片段中的代码正确执行了这一点：`myfirst_restore`写入掩码，然后获取锁、清除标志、释放锁。顺序很重要；反转它会创建一个可能丢失中断的狭窄窗口。

### 唤醒源清理

如果驱动在挂起期间启用了唤醒源（`pci_enable_pme`），恢复路径应清除任何待处理的唤醒事件（`pci_clear_pme`）。PCI层的`pci_resume_child`助手已在驱动的`DEVICE_RESUME`之前调用`pci_clear_pme(child)`，所以驱动通常不需要再次调用它。

驱动可能想显式调用`pci_clear_pme`的一种情况是在运行时PM上下文中，驱动在系统保持S0时恢复设备。在这种情况下`pci_resume_child`未参与，驱动负责自己清除PME状态。

支持wake-on-X的驱动的假设草图：

```c
static int
myfirst_pci_resume(device_t dev)
{
        struct myfirst_softc *sc = device_get_softc(dev);

        if (pci_has_pm(dev))
                pci_clear_pme(dev);  /* 防御性；PCI层已经做了 */

        /* ... 恢复路径的其余部分 ... */
}
```

对于`myfirst`，没有唤醒源，所以调用没有实际用途；本章从主代码中省略它并在此处提及模式以供完整性。

### 更新第3阶段驱动

第3阶段将上述所有内容整合到单个可工作的恢复中。与第2阶段的差异是：

- `myfirst.h`增加`saved_intr_mask`字段（为第2阶段添加）和`broken`标志。
- `myfirst_pci.c`获得`myfirst_restore`助手和重写的`myfirst_pci_resume`。
- Makefile版本升级到`1.5-power-stage3`。

构建并测试：

```sh
cd /path/to/driver
make clean && make
sudo kldunload myfirst     # 卸载任何先前版本
sudo kldload ./myfirst.ko

# 安静基线。
sysctl dev.myfirst.0.dma_transfers_read
# 0
sysctl dev.myfirst.0.suspended
# 0

# 完整周期。
sudo devctl suspend myfirst0
sysctl dev.myfirst.0.suspended
# 1

sudo devctl resume myfirst0
sysctl dev.myfirst.0.suspended
# 0

# 恢复后的传输应正常工作。
sudo sysctl dev.myfirst.0.dma_test_read=1
sysctl dev.myfirst.0.dma_transfers_read
# 1

# 多次执行以确保路径稳定。
for i in 1 2 3 4 5; do
  sudo devctl suspend myfirst0
  sudo devctl resume myfirst0
  sudo sysctl dev.myfirst.0.dma_test_read=1
done
sysctl dev.myfirst.0.dma_transfers_read
# 6 (1 + 5)

sysctl dev.myfirst.0.power_suspend_count dev.myfirst.0.power_resume_count
# 应相等，各约6
```

如果计数器漂移（挂起计数不等于恢复计数）或如果`dma_test_read`在挂起后开始失败，恢复路径中的某些东西没有将设备放回可用状态。第一个调试步骤是读取INTR_MASK并与`saved_intr_mask`比较；第二个是跟踪DMA引擎的状态寄存器看它是否报告错误。

### 与第20章MSI-X设置的交互

第20章的`myfirst`驱动在可用时使用MSI-X，采用三向量布局（admin、rx、tx）。MSI-X配置存在于设备的MSI-X能力寄存器和内核侧表中。PCI层的配置空间保存恢复覆盖能力寄存器；内核侧状态不受D状态转换影响。

这意味着`myfirst`驱动不需要做任何特殊操作来恢复其MSI-X向量。中断资源（`irq_res`）保持分配，cookie保持注册，CPU绑定保持不变。当设备在恢复时引发MSI-X向量时，内核将其传递给在挂载时注册的过滤器。

想要验证这一点的读者可以在恢复后写入一个模拟sysctl并观察相应的每向量计数器递增：

```sh
sudo devctl suspend myfirst0
sudo devctl resume myfirst0
sudo sysctl dev.myfirst.0.intr_simulate_admin=1
sysctl dev.myfirst.0.vec0_fire_count
# 应已递增
```

如果计数器未递增，MSI-X路径已被干扰。最可能的原因是驱动自身状态管理中的错误（`suspended`标志未清除，或过滤器因不同原因拒绝中断）。本章的故障排除部分有更多细节。

### 优雅处理恢复失败

如果恢复的某个步骤失败，驱动选项有限。它不能否决恢复（内核此时没有展开路径）。通常不能重试（硬件状态不确定）。它能做的最好是：

1. 用`device_printf`显眼地记录失败，以便用户在dmesg中看到。
2. 递增计数器（`power_resume_errors`），回归脚本或可观测性工具可以检查。
3. 标记设备为损坏，以便后续请求干净地失败而非静默损坏数据。
4. 保持驱动挂载，使设备树状态保持一致，用户最终可以卸载并重新加载驱动。
5. 从`DEVICE_RESUME`返回0，因为内核期望它成功。

"标记损坏、保持挂载"模式在生产驱动中常见。它将失败从"神秘的后续损坏"移动到"立即可见的用户错误"，这是更好的调试体验。

### 短暂弯路：运行时PM中的pci_save_state / pci_restore_state

第2节提到`pci_save_state`和`pci_restore_state`有时由驱动自身调用，通常在运行时电源管理助手中。在第5节构建它之前，值得做一个具体的草图。

将空闲设备放入D3的运行时PM助手如下：

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
                /* 回滚 */
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

模式与系统挂起/恢复类似，但使用显式PCI助手，因为PCI层不在循环中。第5节将把这个草图变为真实实现并连接到空闲检测策略。

### 与真实驱动的现实检验

继续之前，值得停下来看看真实驱动的恢复路径。`/usr/src/sys/dev/re/if_re.c`的`re_resume`函数大约三十行。其结构为：

1. 锁定softc。
2. 如果MAC睡眠标志已设置，通过写GPIO寄存器将芯片从睡眠模式取出。
3. 清除任何网络唤醒模式，以免正常接收过滤不被干扰。
4. 如果接口在管理上已启动，通过`re_init_locked`重新初始化。
5. 清除`suspended`标志。
6. 解锁softc。
7. 返回0。

`re_init_locked`调用是实质工作：它重新编程MAC地址，重置接收和发送描述符环，重新启用NIC上的中断，并启动DMA引擎。对于`myfirst`，等效工作短得多因为设备简单得多，但形状相同：获取状态、执行硬件特定重新初始化、解锁、返回。

在实现`myfirst`的恢复后阅读`re_resume`的读者会立即识别出结构。词汇相同；只有细节不同。

### 第4节收尾

第4节完成了恢复路径。它展示了到`DEVICE_RESUME`被调用时PCI层已经做了什么（D0转换、配置空间恢复、PME#清除、MSI-X恢复），驱动仍然需要做什么（重新启用总线主控、恢复设备特定寄存器、清除挂起标志、取消掩码中断），以及每个步骤为何重要。第3阶段驱动现在可以完成完整的挂起-恢复周期并继续正常操作；回归测试可以连续运行多个周期并验证计数器一致。

第3节和第4节一起，驱动在系统挂起意义上是电源感知的：它干净地处理S3和S4转换。它仍然不做的是系统运行时的任何设备级节能。那就是运行时电源管理，第5节教授它。



## 第5节：处理运行时电源管理

系统挂起是一个大的、可见的转换：盖子关闭，屏幕变暗，电池节能数小时。运行时电源管理正好相反：每秒数十次小的、不可见的转换，每次节省一点，一起节省现代系统空闲功耗的大部分。用户从不注意它们；平台工程师因其正确性而生或死。

本节在章节大纲中标记为可选，因为并非每个驱动都需要运行时PM。始终活跃的设备的驱动（繁忙服务器上的NIC、根文件系统的磁盘控制器）不通过尝试挂起其设备来节能；设备正忙，尝试挂起它浪费设置从不完成的转换的周期。经常空闲的设备的驱动（网络摄像头、指纹读取器、笔记本上的WLAN卡）确实受益。是否添加运行时PM是由设备使用配置文件驱动的策略决策。

对于第22章，我们在`myfirst`驱动上实现运行时PM作为学习练习。设备已经是模拟的；我们可以假装它在最近几秒内没有sysctl写入时是空闲的，并观察驱动完成这些操作。实现很短，它教授真实运行时PM驱动使用的PCI级原语。

### FreeBSD中运行时PM意味着什么

FreeBSD目前没有像Linux那样的集中式运行时PM框架。没有内核侧的"如果设备已空闲N毫秒，调用其空闲钩子"机制。相反，运行时PM是驱动本地的：驱动决定何时挂起和恢复其设备，使用它在`DEVICE_SUSPEND`和`DEVICE_RESUME`内部会使用的相同PCI层原语（`pci_set_powerstate`、`pci_save_state`、`pci_restore_state`）。

这有两个后果。首先，每个想要运行时PM的驱动实现自己的策略：设备必须空闲多长时间才挂起、什么算空闲、设备按需求唤醒必须多快。其次，驱动必须将运行时PM与其系统PM集成；两条路径共享大量代码且不能互相冲突。

第22章使用的模式很简单：

1. 驱动添加一个具有`RUNNING`和`RUNTIME_SUSPENDED`状态的小状态机。
2. 当驱动观察到空闲（第5节使用基于callout的"最近5秒没有请求"策略）时，它调用`myfirst_runtime_suspend`。
3. 当驱动在`RUNTIME_SUSPENDED`中观察到新请求时，它在处理请求之前调用`myfirst_runtime_resume`。
4. 在系统挂起时，如果设备处于`RUNTIME_SUSPENDED`，系统挂起路径对其进行调整（设备已静默；系统挂起的静默是无操作，但系统恢复必须将设备带回D0）。
5. 在系统恢复时，驱动返回`RUNNING`，除非它被显式运行时挂起并想保持那样。

这比Linux的运行时PM框架更简单，后者有更丰富的概念（父/子引用计数、自动挂起定时器、屏障）。对于简单硬件上的单个驱动，FreeBSD方法足够了。

### 运行时PM状态机

softc增加一个状态变量和一个时间戳：

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

`idle_threshold_seconds`是通过sysctl暴露的策略旋钮；默认为五秒提供快速可观测性而不至于在正常使用中因过于激进导致不必要的唤醒。生产驱动会按设备调整此值；五秒是一个学习友好的值，使转换可见而不需要等待数小时。

`idle_watcher` callout每秒触发一次检查空闲时间。如果设备空闲时间超过`idle_threshold_seconds`且当前处于`RUNNING`，callout触发`myfirst_runtime_suspend`。

### 实现

挂载路径启动空闲监视器：

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

callout用softc互斥锁初始化，所以触发时自动获取互斥锁。这简化了回调：它在锁下运行。

回调检查自上次活动以来的时间并在需要时挂起：

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
                         * 在挂起期间释放锁。运行时挂起助手
                         * 根需要再次获取它。
                         */
                        MYFIRST_UNLOCK(sc);
                        (void)myfirst_runtime_suspend(sc);
                        MYFIRST_LOCK(sc);
                }
        }

        /* 重新调度。 */
        callout_reset(&sc->idle_watcher, hz, myfirst_idle_watcher_cb, sc);
}
```

注意`myfirst_runtime_suspend`周围的锁释放。挂起助手调用`myfirst_quiesce`，它自己获取锁。跨它持有锁会死锁。

活动在驱动服务请求时记录。第21章的DMA路径是一个好的钩子：每次用户写入`dma_test_read`或`dma_test_write`时，sysctl处理器记录活动：

```c
static int
myfirst_dma_sysctl_test_write(SYSCTL_HANDLER_ARGS)
{
        struct myfirst_softc *sc = arg1;
        /* ... 现有代码 ... */

        /* 处理前标记设备为活跃。 */
        myfirst_mark_active(sc);

        /* 如果运行时已挂起，在运行前将设备带回。 */
        if (sc->runtime_state == MYFIRST_RT_SUSPENDED) {
                int err = myfirst_runtime_resume(sc);
                if (err != 0)
                        return (err);
        }

        /* ... 继续传输 ... */
}
```

`myfirst_mark_active`助手是一行代码：

```c
static void
myfirst_mark_active(struct myfirst_softc *sc)
{
        MYFIRST_LOCK(sc);
        microtime(&sc->last_activity);
        MYFIRST_UNLOCK(sc);
}
```

### 运行时挂起和运行时恢复助手

这些在第4节中已做草图。以下是完整版本：

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

形状与系统挂起/恢复相同，只是驱动显式调用`pci_set_powerstate`和`pci_save_state`/`pci_restore_state`。PCI层的自动转换不参与运行时PM，因为内核没有协调系统范围的电源更改；驱动独自处理。

### 运行时PM和系统PM之间的交互

两条路径必须协作。考虑当用户合上笔记本盖子时设备已运行时挂起（在D3中）会发生什么：

1. 内核开始系统挂起。
2. PCI总线调用`myfirst_pci_suspend`。
3. 在`myfirst_pci_suspend`内部，驱动注意到设备已运行时挂起。静默是无操作（没有事情在发生）。PCI层的自动配置空间保存运行；它读取配置空间（在D3中仍可访问）并缓存它。
4. PCI层将设备从D3转换到……等等，它已经在D3中。到D3的转换是无操作。
5. 系统睡眠。
6. 唤醒时，PCI层将设备转回D0。驱动的`myfirst_pci_resume`运行。它恢复状态。但现在驱动认为设备是`RUNNING`（因为系统恢复清除了`suspended`标志），而概念上它之前是运行时挂起的。下一个活动会正常使用设备并设置`last_activity`；空闲监视器最终会重新挂起它如果仍然空闲。

交互大多是良性的；最坏情况是设备在空闲监视器重新挂起它之前多经历一次D0。更完善的实现会在系统挂起间记住运行时挂起状态并恢复它，但对于学习驱动简单方法就足够了。

反向（系统挂起已运行时挂起的设备）在我们的实现中已经正确，因为`myfirst_quiesce`检查`suspended`并在已设置时返回0。运行时挂起路径作为其静默的一部分设置了`suspended = true`，所以系统挂起的静默看到标志并跳过。

### 通过Sysctl暴露运行时PM控制

驱动的运行时PM策略可以通过sysctl控制和观察：

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

读者现在可以这样做：

```sh
# 观察设备空闲。
while :; do
        sysctl dev.myfirst.0.runtime_state dev.myfirst.0.runtime_suspend_count
        sleep 1
done &
```

五秒不活动后，`runtime_state`从0翻转到1，`runtime_suspend_count`递增。写入任何活动sysctl触发恢复并将状态翻转回来：

```sh
sudo sysctl dev.myfirst.0.dma_test_read=1
# 日志显示：运行时恢复，然后测试读取
```

### 权衡

运行时PM用唤醒延迟换取空闲功耗。每次D3到D0转换花费时间（PCIe链路上数十微秒，包括ASPM退出），在某些设备上还花费能量（转换本身消耗电流）。对于大部分时间空闲、偶尔有活动突发的设备，交换是有利的。对于大部分时间活跃、偶尔有空闲期的设备，转换成本占主导。

`idle_threshold_seconds`旋钮让平台调整这一点。值为0或1是激进的，适合每次使用几秒、空闲几分钟的网络摄像头。值为60是保守的，适合空闲期短但频繁的NIC。值为0（如果允许）会完全禁用运行时PM，适合应始终保持开启的设备。

第二个权衡在代码复杂性方面。运行时PM添加状态机、callout、空闲监视器、两个额外类kobj助手，以及运行时和系统PM路径之间的额外排序关注。每个都很小，但它们一起增加了错误的攻击面。许多FreeBSD驱动因此故意省略运行时PM；它们让设备保持在D0并依赖设备自身的内部低功耗状态（时钟门控、PCIe ASPM）来节能。这是一个可辩护的选择，对于正确性比毫瓦更重要的驱动，这是正确的选择。

第22章的`myfirst`驱动将运行时PM保持为可选功能，由构建时标志控制：

```make
CFLAGS+= -DMYFIRST_ENABLE_RUNTIME_PM
```

读者可以在有或没有该标志的情况下构建；第5节代码仅在定义标志时编译。第3阶段的默认是关闭运行时PM；第4阶段在合并驱动中启用它。

### 关于平台运行时PM的说明

一些平台在驱动本地的运行时PM之外提供自己的运行时PM机制。在arm64和RISC-V嵌入式系统上，设备树可能描述`power-domains`和`clocks`属性，驱动使用它们关闭电源域和门控时钟。FreeBSD的`ext_resources/clk`、`ext_resources/regulator`和`ext_resources/power`子系统处理这些。

此类平台上的运行时PM比仅PCI的运行时PM更强大，因为平台可以关闭整个SoC块（USB控制器、显示引擎、GPU）而不仅仅是将PCI设备移到D3。驱动使用相同的模式（标记空闲、空闲时关闭资源、活动时重新开启）但通过不同的API。

第22章停留在PCI路径上，因为那是`myfirst`驱动所在。之后在嵌入式平台上工作的读者会发现相同的概念结构和平台特定API。本章在此提及区别，以便读者知道该领域存在。

### 第5节收尾

第5节为驱动添加了运行时电源管理。它定义了两状态机（`RUNNING`、`RUNTIME_SUSPENDED`）、基于callout的空闲监视器、使用PCI层显式电源状态和状态保存API的一对助手（`myfirst_runtime_suspend`、`myfirst_runtime_resume`）、DMA sysctl处理器中的活动记录钩子，以及向用户空间暴露策略的sysctl旋钮。它还讨论了运行时PM和系统PM之间的交互、延迟与功耗的权衡，以及嵌入式系统上平台级运行时PM的替代方案。

第2到5节就位后，驱动现在处理系统挂起、系统恢复、系统关机、运行时挂起和运行时恢复。它尚未干净地解释的是读者如何从用户空间测试所有这些。第6节转向用户空间接口：`acpiconf`、`zzz`、`devctl suspend`、`devctl resume`、`devinfo -v`，以及将它们组合在一起的回归测试。



## 第6节：与电源框架交互

正确处理挂起和恢复的驱动只是故事的一半。另一半是能够从用户空间*测试*这种正确性，重复地、有目的地。第6节调查FreeBSD为此目的提供的工具，解释每个如何适合驱动的状态模型，并展示如何将它们组合成回归脚本，练习第2到5节构建的每条路径。

### 四个用户空间入口点

四个命令几乎覆盖驱动开发者需要的一切：

- **`acpiconf -s 3`**（及其变体）请求ACPI将整个系统置入睡眠状态S3。这是最真实的测试；它练习从用户空间通过内核挂起机制通过PCI层到驱动方法的完整路径。
- **`zzz`**是`acpiconf -s 3`的薄包装器。它读取`hw.acpi.suspend_state`（默认为S3）并进入相应的睡眠状态。对于大多数用户，它是从shell挂起最方便的方式。
- **`devctl suspend myfirst0`**和**`devctl resume myfirst0`**通过`/dev/devctl2`上的`DEV_SUSPEND`和`DEV_RESUME` ioctl触发每设备挂起和恢复。这些仅调用驱动的方法；系统其余部分保持在S0。这是最快的迭代目标，也是第22章大部分开发使用的。
- **`devinfo -v`**列出设备树中所有设备及其当前状态。它显示设备是已挂载、已挂起还是已分离。

每个都有优缺点。`acpiconf`真实但慢（典型硬件上每个周期一到三秒）且具有破坏性（系统实际睡眠）。`devctl`快（每个周期毫秒）但仅练习驱动，不练习ACPI或平台代码。`devinfo -v`是被动的且廉价的；它观察而不改变状态。

好的回归策略使用所有三个：`devctl`用于驱动方法的单元测试，`acpiconf`用于完整挂起路径的集成测试，`devinfo -v`用于快速健全性检查。

### 使用acpiconf挂起系统

在有正常ACPI的机器上，`acpiconf -s 3`是第1节所称的完整系统挂起。命令：

```sh
sudo acpiconf -s 3
```

执行以下步骤：

1. 打开`/dev/acpi`并通过`ACPIIO_ACKSLPSTATE` ioctl检查平台是否支持S3。
2. 发送`ACPIIO_REQSLPSTATE` ioctl请求S3。
3. 内核开始挂起序列：暂停用户态、冻结线程、以`DEVICE_SUSPEND`遍历设备树的每个设备。
4. 假设没有驱动否决，内核进入S3。机器睡眠。
5. 唤醒事件（盖子打开、电源按钮按下、USB设备发送远程唤醒信号）唤醒平台。
6. 内核运行恢复序列：在每个设备上`DEVICE_RESUME`，解冻线程，恢复用户态。
7. shell提示符返回。机器回到S0。

要使`myfirst`驱动被测试，驱动必须在挂起之前加载。从用户角度看完整序列如下：

```sh
sudo kldload ./myfirst.ko
sudo sysctl dev.myfirst.0.dma_test_read=1  # 稍微锻炼一下
sudo acpiconf -s 3
# [笔记本睡眠；用户打开盖子]
dmesg | grep myfirst
```

`dmesg`输出应显示第22章日志记录的两行：

```text
myfirst0: suspend: starting
myfirst0: suspend: complete (dma in flight=0, suspended=1)
myfirst0: resume: starting
myfirst0: resume: complete
```

如果这些行存在且按此顺序，驱动的方法已被完整系统路径正确调用。

如果机器未回来，挂起路径在`myfirst`之下的某层中断。如果机器回来了但驱动处于奇怪状态（sysctl返回错误、计数器有奇怪值、DMA传输失败），问题在`myfirst`的挂起或恢复实现中。

### 使用zzz

在FreeBSD上，`zzz`是一个小型shell脚本，读取`hw.acpi.suspend_state`并调用`acpiconf -s <state>`。它不是二进制文件；通常安装在`/usr/sbin/zzz`，只有几行长。典型调用是：

```sh
sudo zzz
```

在支持S3的机器上，默认`hw.acpi.suspend_state`是`S3`。想要测试S4（休眠）的读者可以：

```sh
sudo sysctl hw.acpi.suspend_state=S4
sudo zzz
```

FreeBSD上的S4支持历史上一直不完整；是否工作取决于平台固件和文件系统布局。对于第22章的目的，S3足够了，`zzz`是方便的简写。

### 使用devctl进行每设备挂起

`devctl(8)`命令是为了让用户从用户空间操作设备树而构建的。它支持挂载、分离、启用、禁用、挂起、恢复等。对于第22章，`suspend`和`resume`是两个重要的。

```sh
sudo devctl suspend myfirst0
sudo devctl resume myfirst0
```

第一个命令通过`/dev/devctl2`发出`DEV_SUSPEND`；内核将其转换为在父总线上调用`BUS_SUSPEND_CHILD`，对于PCI设备最终调用`pci_suspend_child`，它保存配置空间、将设备放入D3、并调用驱动的`DEVICE_SUSPEND`。恢复时反向发生。

与`acpiconf`的主要区别：

- 只有目标设备及其子设备经历转换。系统其余部分保持在S0。
- CPU不停车。用户态不冻结。内核不睡眠。
- PCI设备实际进入D3hot（假设`hw.pci.do_power_suspend`为1）。读者可以用`pciconf`验证：

```sh
# 挂起前：设备应在D0
pciconf -lvbc | grep -A 2 myfirst

# devctl suspend myfirst0后：设备应在D3
sudo devctl suspend myfirst0
pciconf -lvbc | grep -A 2 myfirst
```

电源状态通常在`pciconf -lvbc`的`powerspec`行中显示。从`D0`到`D3`的变化是转换真正发生的可观察信号。

### 使用devinfo检查设备状态

`devinfo(8)`工具以各种详细程度列出设备树。`-v`标志显示详细信息，包括设备状态（已挂载、已挂起或不存在）。

```sh
devinfo -v | grep -A 5 myfirst
```

典型输出：

```text
myfirst0 pnpinfo vendor=0x1af4 device=0x1005 subvendor=0x1af4 subdevice=0x0004 class=0x008880 at slot=5 function=0 dbsf=pci0:0:5:0
    Resource: <INTERRUPT>
        10
    Resource: <MEMORY>
        0xfeb80000-0xfeb80fff
```

状态在输出中是隐含的：如果设备已挂起，该行显示设备及其资源但没有"active"标记。显式状态查询可以通过softc sysctl完成；`dev.myfirst.0.%parent`和`dev.myfirst.0.%desc`键告诉用户设备在哪里。

对于第22章，`devinfo -v`作为失败转换后的健全性检查最有用：如果设备从输出中缺失，分离路径已运行；如果设备存在但资源错误，挂载或恢复路径将设备留在了不一致状态。

### 通过sysctl检查电源状态

PCI层通过`hw.pci`下的`sysctl`暴露电源状态信息。两个变量最相关：

```sh
sysctl hw.pci.do_power_suspend
sysctl hw.pci.do_power_resume
```

两者默认为1，意味着PCI层在挂起时将设备转换到D3，在恢复时转回D0。将任一设为0可禁用自动转换用于调试。

ACPI层暴露系统状态信息：

```sh
sysctl hw.acpi.supported_sleep_state
sysctl hw.acpi.suspend_state
sysctl hw.acpi.s4bios
```

第一个列出平台支持哪些睡眠状态（通常类似`S3 S4 S5`）。第二个是`zzz`进入的状态（通常是`S3`）。第三个表明S4是否通过BIOS辅助实现。

对于每设备观察，驱动通过`dev.myfirst.N.*`暴露自己的状态。第22章驱动添加：

- `dev.myfirst.N.suspended`：如果驱动认为自身已挂起则为1，否则为0。
- `dev.myfirst.N.power_suspend_count`：`DEVICE_SUSPEND`被调用的次数。
- `dev.myfirst.N.power_resume_count`：`DEVICE_RESUME`被调用的次数。
- `dev.myfirst.N.power_shutdown_count`：`DEVICE_SHUTDOWN`被调用的次数。
- `dev.myfirst.N.runtime_state`：0表示`RUNNING`，1表示`RUNTIME_SUSPENDED`。
- `dev.myfirst.N.runtime_suspend_count`、`dev.myfirst.N.runtime_resume_count`：运行时PM计数器。
- `dev.myfirst.N.idle_threshold_seconds`：运行时PM空闲阈值。

在这些sysctl和`dmesg`之间，读者可以完全详细地看到驱动在任何转换期间做了什么。

### 回归脚本

labs目录新增一个脚本：`ch22-suspend-resume-cycle.sh`。该脚本：

1. 记录每个计数器的基线值。
2. 运行一次DMA传输确认设备正常工作。
3. 调用`devctl suspend myfirst0`。
4. 验证`dev.myfirst.0.suspended`为1。
5. 验证`dev.myfirst.0.power_suspend_count`增加了1。
6. 调用`devctl resume myfirst0`。
7. 验证`dev.myfirst.0.suspended`为0。
8. 验证`dev.myfirst.0.power_resume_count`增加了1。
9. 再运行一次DMA传输确认设备仍然正常工作。
10. 打印PASS/FAIL摘要。

完整脚本在examples目录中；逻辑的简要大纲：

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

重复运行脚本（比如在紧凑循环中运行一百次）是很好的压力测试。通过一次循环但在第五十次失败的驱动通常有资源泄漏或仅在重复下才出现的边缘情况。这类错误正是回归脚本要发现的。

### 运行压力测试

本章的`labs/`目录还包括`ch22-suspend-stress.sh`，它将循环脚本运行一百次：

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

在带仅模拟myfirst驱动的现代机器上，一百次循环大约需要一秒。如果任何迭代失败，脚本停止并报告迭代号。在开发期间每次更改后运行它可以立即捕获回归。

### 结合运行时PM和用户空间测试

运行时PM路径需要不同的测试，因为它不是由用户命令触发的；它是由空闲触发的。测试如下：

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

在此期间观察`dmesg`的读者会在大约五秒不活动后看到"runtime suspend: starting"和"runtime suspend: device in D3"行，然后在sysctl写入到达时看到"runtime resume: starting"。

本章的lab目录包含`ch22-runtime-pm.sh`来自动化此序列。

### 解释失败模式

当用户空间测试失败时，诊断路径取决于哪一层失败：

- **如果`devctl suspend`返回非零退出码**：驱动的`DEVICE_SUSPEND`返回了非零值，否决了挂起。检查`dmesg`中驱动的日志输出；挂起方法应该记录了出了什么问题。
- **如果`devctl suspend`成功但之后`dev.myfirst.0.suspended`为0**：驱动的静默短暂设置了标志但某物清除了它。这通常意味着静默正在重入自身，或分离路径在与挂起竞争。
- **如果`devctl resume`成功但下一次传输失败**：恢复路径没有完全重新初始化设备。最常见的是中断掩码或DMA寄存器未被写入；检查恢复前后的每向量触发计数器以查看中断是否到达驱动。
- **如果`acpiconf -s 3`成功但系统没有回来**：设备树中`myfirst`下面的某个驱动阻止了恢复。这在测试VM中不常见；这是真实硬件上新驱动的典型故障模式。
- **如果`acpiconf -s 3`返回`EOPNOTSUPP`**：平台不支持S3。检查`sysctl hw.acpi.supported_sleep_state`。

在所有情况下，第一个信息来源是`dmesg`。第22章驱动记录每个转换；如果日志行未出现，方法未被调用，问题在驱动以下的层。

### 最小故障排除流程

失败挂起-恢复周期的紧凑流程图：

1. 驱动已加载吗？`kldstat | grep myfirst`。
2. 设备已挂载吗？`sysctl dev.myfirst.0.%driver`。
3. 挂起和恢复方法有日志吗？`dmesg | tail`。
4. `dev.myfirst.0.suspended`正确切换了吗？`sysctl dev.myfirst.0.suspended`。
5. 计数器增加了吗？`sysctl dev.myfirst.0.power_suspend_count dev.myfirst.0.power_resume_count`。
6. 恢复后传输成功吗？`sudo sysctl dev.myfirst.0.dma_test_read=1; dmesg | tail -2`。
7. 每向量中断计数器增加了吗？`sysctl dev.myfirst.0.vec0_fire_count dev.myfirst.0.vec1_fire_count dev.myfirst.0.vec2_fire_count`。

任何"否"答案指向实现的特定层。第7节更深入地讨论常见故障模式及如何调试它们。

### 第6节收尾

第6节调查了内核电源管理机制的用户空间接口：`acpiconf`、`zzz`、`devctl suspend`、`devctl resume`、`devinfo -v`和相关`sysctl`变量。它展示了如何将这些工具组合成练习一次挂起-恢复周期的回归脚本，以及连续运行一百个周期的压力脚本。它讨论了运行时PM测试流程、最常见故障模式的解释，以及测试失败时读者可以遵循的最小故障排除流程图。

有了用户空间工具，下一节深入探讨读者在编写电源感知代码时可能遇到的典型故障模式，以及如何调试每种故障。



## 第7节：调试电源管理问题

电源管理代码有一类特殊的错误。机器睡眠；机器唤醒；错误在唤醒后未知时间出现，看起来像通用故障而非与电源转换相关的任何东西。因果链比大多数驱动错误更长，重现更慢，用户的错误报告通常是"我的笔记本有时无法唤醒"，其中几乎不包含驱动开发者可以使用的信息。

第7节关于识别典型症状、将其追溯到可能原因，并应用匹配的调试模式。它以第22章`myfirst`驱动为例，但模式适用于任何FreeBSD驱动。

### Symptom 1: 恢复后设备冻结

最常见的电源管理错误，无论在学习驱动还是生产驱动中，都是恢复后设备停止响应。驱动在启动时正确挂载，在S0中正常工作，处理挂起-恢复周期没有可见错误，然后在下一个命令时沉默。中断不触发。DMA传输不完成。从设备寄存器读取返回陈旧值或零。

通常原因是恢复后设备寄存器未被写入。设备以默认状态回来（中断掩码全掩码、DMA引擎禁用、硬件在D0入口时重置的任何寄存器），驱动未重新编程它们，所以从设备角度看没有配置运行的东西。

**调试模式。** 比较挂起前后的设备寄存器值。`myfirst`驱动通过sysctl暴露几个寄存器（如果读者添加了它们）；否则，读者可以写一个短内核空间助手读取每个寄存器并打印。在挂起-恢复周期后：

1. 读取中断掩码寄存器。如果是`0xFFFFFFFF`（全掩码），恢复路径未恢复掩码。
2. 读取DMA控制寄存器。如果ABORT位已设置，静默的中止从未清除。
3. 通过`pciconf -lvbc`读取设备的配置空间。命令寄存器应有总线主控位设置；如果没有，恢复路径遗漏了`pci_enable_busmaster`。

**修复模式。** 恢复路径应包括驱动正常操作依赖的每个设备特定寄存器的无条件重编程。在挂起时将它们保存到softc并在恢复时恢复是一种方法；从softc状态重新派生（`re_resume`采取的方法）是另一种。两者都有效；选择取决于哪个更容易为特定设备证明正确。

### 症状2：丢失中断

冻结设备问题的更微妙变体是丢失中断：设备响应某些调用，但其中断未到达驱动。DMA引擎接受START命令，执行传输，引发完成中断……而中断计数不递增。任务队列未获得条目。CV不广播。传输最终超时，驱动报告EIO。

几种事情可能导致这个：

- 设备处的**中断掩码**仍然全掩码。设备想要引发中断但掩码抑制了它。（恢复路径错误。）
- **MSI或MSI-X配置**未被恢复。设备正在引发中断，但内核未将其路由到驱动的处理器。（不常见；PCI层应自动处理。）
- **过滤器函数指针**被损坏。极不常见；通常指示驱动中其他地方的内存损坏。
- **suspended标志**仍为true，过滤器提前返回。（恢复路径错误：标志未清除。）

**调试模式。** 读取挂起-恢复周期前后的每向量触发计数器。如果计数器未递增，中断未到达过滤器。然后按顺序检查：

1. 挂起标志是否已清除？`sysctl dev.myfirst.0.suspended`。
2. 设备处的中断掩码是否正确？读取寄存器。
3. 设备中的MSI-X表是否正确？`pciconf -c`转储能力寄存器。
4. 内核的MSI调度状态是否一致？`procstat -t`显示中断线程。

**修复模式。** 确保恢复路径（a）在锁下清除挂起标志，（b）在清除标志后取消掩码设备的中断寄存器，（c）不依赖驱动必须自己做的MSI-X恢复（除非通过sysctl显式禁用）。

### Symptom 3: 恢复后DMA错误

更危险的一类错误是DMA看起来正常工作但产生错误数据。驱动编程引擎，引擎运行，完成中断触发，任务运行，同步被调用，驱动读取缓冲区……但字节是错误的。不是零，不是垃圾，只是微妙地不正确：之前写入的模式，或两个周期前的模式，或指示DMA寻址错误页面的模式。

原因：

- **softc中缓存的总线地址**过时了。这对于静态分配不常见（地址在挂载时设置一次不应改变），但如果驱动在恢复时重新分配DMA缓冲区（一个坏主意；见下文），可能发生。
- 恢复后**DMA引擎的基址寄存器**未被重编程，它有过时值指向其他地方。
- **`bus_dmamap_sync`调用缺失或顺序错误**。这是经典的DMA正确性错误，在恢复路径中值得警惕，因为同步调用附近的驱动侧代码在重构期间经常被编辑。
- **IOMMU转换表**未被恢复。在FreeBSD上非常罕见，因为IOMMU配置是每会话的，在大多数平台上在挂起中存活；但如果驱动运行在`DEV_IOMMU`不常见的系统上，这可能发生。

**调试模式。** 在每次DMA前添加已知模式写入，在每次DMA后验证，并记录两者。将周期简化为"写0xAA、同步、读、期望0xAA"使数据损坏错误立即可见。

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

对于模拟，这应始终成功因为模拟在写传输时不修改缓冲区。在真实硬件上，模式取决于设备。调试真实硬件错误的读者调整测试。

**修复模式。** 如果总线地址是问题，在恢复时重建它：

```c
/* In resume, after PCI restore is complete. */
err = bus_dmamap_load(sc->dma_tag, sc->dma_map,
    sc->dma_vaddr, MYFIRST_DMA_BUFFER_SIZE,
    myfirst_dma_single_map, &sc->dma_bus_addr,
    BUS_DMA_NOWAIT);
```

只在总线地址实际改变时才这样做，这很罕见。更常见的是，修复是在每次传输开始时写入基址寄存器（而非依赖持久值）并确保同步调用顺序正确。

### Symptom 4: Lost PME# Wake Events

在支持wake-on-X的设备上，症状是"设备应该唤醒系统但没有"。驱动报告了成功挂起；系统进入S3；预期事件（魔法数据包、按钮按下、定时器）发生了；系统保持睡眠。

原因：

- 挂起路径中**未调用`pci_enable_pme`**。设备的PME_En位为0，所以即使设备通常会断言PME#，该位也被抑制。
- **设备自身的唤醒逻辑未配置**。对于NIC，网络唤醒寄存器必须在挂起前编程。对于USB主机控制器，远程唤醒能力必须按端口启用。
- **平台的唤醒GPE未启用**。这通常是固件事务；ACPI `_PRW`方法应已注册GPE，但在某些机器上BIOS默认禁用它。
- **挂起时PME状态位已设置**，陈旧的PME#是触发唤醒的东西（而非预期事件）。系统看起来在睡眠后立即唤醒。

**调试模式。** 通过`pciconf -lvbc`读取PCI配置空间。电源管理能力的状态/控制寄存器显示PME_En和PME_Status位。挂起前，PME_Status应为0（无待处理唤醒）。启用唤醒挂起后，PME_En应为1。

在唤醒未发生的机器上，检查BIOS设置中的"网络唤醒"、"USB唤醒"等。驱动可以完美但如果平台未配置系统仍不会唤醒。

**修复模式。** 在支持唤醒的驱动的挂起路径中：

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

第22章的`myfirst`驱动不实现唤醒（模拟没有唤醒逻辑）。上面的模式作为参考展示。

### Symptom 5: 挂起期间WITNESS警告

启用`WITNESS`的调试内核经常产生如下消息：

```text
witness: acquiring sleepable lock foo_mtx @ /path/to/driver.c:123
witness: sleeping with non-sleepable lock bar_mtx @ /path/to/driver.c:456
```

这些是锁顺序违规或持锁睡眠违规，它们经常在挂起代码中出现因为挂起做驱动通常不做的事情：获取锁、睡眠、协调多个线程。

原因：

- 挂起路径获取锁然后调用在未显式容忍持该锁睡眠的情况下睡眠的函数。
- 挂起路径以与驱动其余部分不同的顺序获取锁，`WITNESS`注意到逆转。
- 挂起路径在持有softc锁时调用`taskqueue_drain`或`callout_drain`，如果任务或callout尝试获取同一个锁则导致死锁。

**调试模式。** 仔细阅读`WITNESS`消息。它包括锁名称和每个被获取的源代码行号。跟踪从获取到睡眠或锁逆转的路径。

**修复模式。** 第22章的`myfirst_quiesce`正是因此原因在调用`myfirst_drain_workers`之前释放softc锁。扩展驱动时：

- 不要在持有任何驱动锁时调用`taskqueue_drain`。
- 不要在持有callout获取的锁时调用`callout_drain`。
- 睡眠原语（`pause`、`cv_wait`）必须在仅持有睡眠互斥锁时调用（不是自旋互斥锁）。
- 如果需要为睡眠释放锁，显式这样做并在之后重新获取。

### 症状6：计数器不匹配

本章的回归脚本期望每个周期后`power_suspend_count == power_resume_count`。当它们漂移时，出了问题。

原因：

- 驱动的`DEVICE_SUSPEND`被调用但驱动在递增计数器之前提前返回。（通常是因为健全性检查触发了。）
- 驱动的`DEVICE_RESUME`未被调用，因为`DEVICE_SUSPEND`返回非零且内核展开了。
- 计数器不是原子的，并发更新丢失了一次递增。（如果代码使用`atomic_add_64`则不太可能。）
- 驱动在计数之间被卸载并重新加载，重置了它们。

**调试模式。** 在运行回归脚本前清除`dmesg -c`，在每个周期后运行`dmesg`。日志显示每个方法调用；计数日志行是计数计数器的替代方案，任何差异指示错误。

### 症状7：挂起期间挂起

挂起期间的挂起是最差的诊断：内核仍在运行（控制台仍响应break-to-DDB），但挂起序列卡在某个驱动的`DEVICE_SUSPEND`中。中断进入DDB并`ps`查看哪个线程在哪里：

```text
db> ps
...  0 myfirst_drain_dma+0x42 myfirst_pci_suspend+0x80 ...
```

**调试模式。** 识别挂起的线程及其卡在的函数。通常是`cv_wait`或`cv_timedwait`从未完成，或`taskqueue_drain`等待不会完成的任务。

**修复模式。** 为挂起路径的任何等待添加超时。`myfirst_drain_dma`函数使用带一秒超时的`cv_timedwait`；使用`cv_wait`（无超时）的变体可能无限期挂起。本章的实现始终使用定时变体因此原因。

### 使用DTrace跟踪挂起和恢复

DTrace是以细粒度观察电源管理路径而不添加打印语句的优秀工具。一个计时每次调用的简单D脚本：

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

将此保存为`trace-devpower.d`并用`dtrace -s trace-devpower.d`运行。任何`devctl suspend`或`acpiconf -s 3`都会产生显示每个设备挂起和恢复时间及其返回值的输出。

对于`myfirst`驱动，`fbt::myfirst_pci_suspend:entry`和`fbt::myfirst_pci_resume:entry`是探针。专注于驱动的D脚本：

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

`stack()`调用在入口打印调用栈，这对于确认方法是否从您期望的位置被调用很有用（例如PCI总线的`bus_suspend_child`）。

### 关于日志纪律的说明

第22章代码在挂起和恢复期间慷慨地记录日志：每个方法记录入口和出口，每个助手记录自己的事件。这种详细程度在开发期间有帮助但在生产中烦人（每次笔记本挂起向dmesg打印半打行）。

好的生产驱动暴露控制日志详细程度的sysctl：

```c
static int myfirst_power_verbose = 1;
SYSCTL_INT(_dev_myfirst, OID_AUTO, power_verbose,
    CTLFLAG_RWTUN, &myfirst_power_verbose, 0,
    "Verbose power-management logging (0=off, 1=on, 2=debug)");
```

日志记录变得有条件：

```c
if (myfirst_power_verbose >= 1)
        device_printf(dev, "suspend: starting\n");
```

想要在生产系统上启用调试的读者可以临时设置`dev.myfirst.power_verbose=2`，触发问题，然后重置变量。第22章驱动不实现这种分层；学习驱动记录一切并接受噪音。

### 使用INVARIANTS内核进行断言覆盖

编译了`INVARIANTS`的调试内核使`KASSERT`宏实际评估其条件并在失败时恐慌。`myfirst_dma.c`和`myfirst_pci.c`代码使用多个KASSERT；电源管理代码添加更多。例如，静默不变量：

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

在`INVARIANTS`内核上，让`dma_in_flight`为true的错误会导致立即恐慌并带有有用消息。在生产内核上，断言被编译掉，什么也不发生。学习驱动故意在`INVARIANTS`内核上运行以捕获这类错误。

类似地，恢复路径可以断言：

```c
KASSERT(sc->suspended == true,
    ("myfirst: resume called but not suspended"));
```

这捕获驱动某种方式在匹配挂起未发生时被调用恢复的错误（通常是父总线驱动中的错误，不是`myfirst`驱动本身的）。

### 调试案例研究

将模式结合在一起，考虑一个具体场景。读者编写第2阶段挂起，运行回归周期，并看到：

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

用户可见的症状是恢复后的传输不工作。日志显示挂起期间的排空超时，这是第一个异常。

**假设。** DMA引擎未遵守ABORT位。驱动强制清除`dma_in_flight`，但引擎仍在运行；当用户触发新传输时，引擎未准备好。

**测试。** 检查中止前后的引擎状态寄存器：

```c
/* In myfirst_drain_dma, after writing ABORT: */
uint32_t pre_status = CSR_READ_4(sc->dev, MYFIRST_REG_DMA_STATUS);
DELAY(100);  /* let the engine notice */
uint32_t post_status = CSR_READ_4(sc->dev, MYFIRST_REG_DMA_STATUS);
device_printf(sc->dev, "drain: status %#x -> %#x\n", pre_status, post_status);
```

再次运行周期产生：

```text
myfirst0: drain: status 0x4 -> 0x4
```

状态0x4是RUNNING。引擎忽略了ABORT。这指向模拟后端：模拟引擎可能未实现中止，或仅在模拟callout触发时才这样做。

**修复。** 查看模拟的DMA引擎代码并验证中止语义。在这种情况下，模拟引擎在其callout回调中处理中止，回调几毫秒才触发一次。将排空超时从1秒（宽裕）扩展到……等等，1秒对于每几毫秒触发的callout已经宽裕。真正的问题在别处。

进一步调查揭示模拟的callout在DMA排空完成*之前*就被排空了。`myfirst_drain_workers`中的顺序（先任务后callout）是错误的；应该是先callout后任务，因为callout驱动中止完成。

**解决方案。** 重新排序排空：

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

但是等等：到`myfirst_drain_workers`从`myfirst_quiesce`被调用时，`myfirst_drain_dma`已完成。排空DMA等待在排空DMA调用内部；排空工作者调用仅清理残余状态。排空工作者内部的顺序对挂起主要是美观的。

真正的修复更早：`myfirst_drain_dma`本身不应超时。一秒超时应已宽裕。实际原因不同：也许模拟的callout未触发因为驱动持有阻塞它的sysctl锁。或ABORT位的写入未到达模拟因为模拟的MMIO处理器也被阻塞。

**教训。** 调试电源管理问题是迭代的。每个症状提出假设；每个测试缩小范围；修复通常在症状指向的不同层中。遵循链条的耐心是区分好的电源感知代码和勉强工作的代码的关键。

### Wrapping Up Section 7

第7节遍历了电源感知驱动的典型故障模式：冻结设备、丢失中断、错误DMA、错过唤醒事件、WITNESS抱怨、计数器漂移和完全挂起。对于每种，它展示了典型原因、缩小问题范围的调试模式，以及消除问题的修复模式。它还介绍了DTrace用于测量，讨论了日志纪律，并展示了`INVARIANTS`和`WITNESS`如何捕获仅在特定条件下出现的错误类。

第7节的调试纪律，像第3节的静默纪律和第4节的恢复纪律一样，旨在留在读者身上超越`myfirst`驱动。每个电源感知驱动的实现中都潜伏着这些错误的某种变体；上面的模式是如何在它们到达用户之前找到它们。

第8节通过将第2到7节的代码整合到重构的`myfirst_power.c`文件中，将版本升级到`1.5-power`，添加`POWER.md`文档，并连接最终集成测试来结束第22章。



## 第8节：重构和版本化电源感知驱动

第1到3阶段在`myfirst_pci.c`中内联添加了电源管理代码。这对教学方便，因为每个更改出现在读者已知的挂载和分离代码旁边。但对可读性不太方便：`myfirst_pci.c`现在有挂载、分离、三个电源方法和几个助手，文件足够长，首次读者需要滚动来找到东西。

第4阶段，第22章驱动的最终版本，将所有电源管理代码从`myfirst_pci.c`中拉出到新的文件对`myfirst_power.c`和`myfirst_power.h`中。这遵循与第20章`myfirst_msix.c`拆分和第21章`myfirst_dma.c`拆分相同的模式：新文件有狭窄的、良好文档化的API，`myfirst_pci.c`中的调用者仅使用该API。

### 目标布局

第4阶段后，驱动的源文件为：

- `myfirst.c` - 顶层粘合、共享状态、sysctl树。
- `myfirst_hw.c`、`myfirst_hw_pci.c` - 寄存器访问助手。
- `myfirst_sim.c` - 模拟后端。
- `myfirst_pci.c` - PCI挂载、分离、方法表，以及到子系统的薄转发。
- `myfirst_intr.c` - 单向量中断（第19章遗留路径）。
- `myfirst_msix.c` - 多向量中断设置（第20章）。
- `myfirst_dma.c` - DMA设置、拆除、传输（第21章）。
- `myfirst_power.c` - 电源管理（第22章，新增）。
- `cbuf.c` - 循环缓冲区支持。

新的`myfirst_power.h`声明电源子系统的公共API：

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

`_setup`和`_teardown`对初始化和拆除子系统级状态（callout、sysctl）。每转换函数包装第3到5节代码构建的相同逻辑。运行时PM函数仅在定义构建时标志时编译。

### myfirst_power.c文件

新文件大约三百行。其结构镜像`myfirst_dma.c`：头文件包含、静态助手、公共函数、sysctl处理器和`_add_sysctls`。

助手是第3节的三个：

- `myfirst_mask_interrupts`
- `myfirst_drain_dma`
- `myfirst_drain_workers`

加上第4节的一个：

- `myfirst_restore`

以及，如果启用了运行时PM，第5节的两个：

- `myfirst_idle_watcher_cb`
- `myfirst_start_idle_watcher`

公共函数`myfirst_power_suspend`、`myfirst_power_resume`和`myfirst_power_shutdown`成为以正确顺序调用助手并更新计数器的薄包装器。sysctl处理器暴露策略旋钮和可观测性计数器。

### 更新myfirst_pci.c

`myfirst_pci.c`文件现在短得多。它的三个电源方法每个仅转发到电源子系统：

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

方法表与第1阶段设置的相同。上面的三个原型现在是`myfirst_pci.c`中唯一的电源相关代码，除了从挂载调用`myfirst_power_setup`和从分离调用`myfirst_power_teardown`。

挂载路径增加一个调用：

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

分离路径增加一个匹配调用：

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

`myfirst_power_setup`初始化`saved_intr_mask`、`suspended`标志、计数器，以及（如果运行时PM启用）空闲监视器callout。`myfirst_power_teardown`排空callout并清理任何子系统级状态。拆除必须在DMA拆除之前完成，因为callout可能仍引用DMA状态。

### 更新Makefile

新源文件放入`SRCS`列表，版本升级：

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

`MYFIRST_ENABLE_RUNTIME_PM`标志在第4阶段默认关闭；运行时PM代码编译但被`#ifdef`包装。想要实验的读者在构建时启用标志。

### 编写POWER.md

第21章的模式确立了先例：每个子系统获得一个markdown文档描述其目的、API、状态模型和测试故事。`POWER.md`是下一个。

好的`POWER.md`有这些部分：

1. **目的**：一段解释子系统做什么的文字。
2. **公共API**：函数原型及其一行描述的表格。
3. **状态模型**：状态和转换的图表或文字描述。
4. **计数器和Sysctl**：子系统暴露的只读和读写sysctl。
5. **转换流程**：挂起、恢复、关机各期间发生什么。
6. **与其他子系统的交互**：电源管理如何与DMA、中断和模拟相关。
7. **运行时PM（可选）**：运行时PM如何工作以及何时启用。
8. **测试**：回归和压力脚本。
9. **已知限制**：子系统尚未做什么。
10. **另见**：`bus(9)`、`pci(9)`和章节文本的交叉引用。

完整文档在examples目录（`examples/part-04/ch22-power/stage4-final/POWER.md`）；本章不在正文中复制它，但想检查预期结构的读者可以打开它。

### 回归脚本

第4阶段回归脚本练习每条路径：

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

每个子脚本几十行测试一件事。每次更改后运行完整回归可以立即捕获回归。

### 与现有回归测试的集成

第21章回归脚本检查了：

- `dma_complete_intrs == dma_complete_tasks`（任务总是看到每个中断）。
- `dma_complete_intrs == dma_transfers_write + dma_transfers_read + dma_errors + dma_timeouts`。

第22章脚本添加：

- `power_suspend_count == power_resume_count`（每个挂起有匹配的恢复）。
- `suspended`标志在转换之外为0。
- 挂起-恢复周期后，DMA计数器仍加到预期总数（无幻影传输）。

组合回归是第22章的完整脚本。它一起练习DMA、中断、MSI-X和电源管理。通过它的驱动状态良好。

### 版本历史

驱动现在已通过多个版本演进：

- `1.0` - 第16章：仅MMIO驱动、模拟后端。
- `1.1` - 第18章：PCI挂载、真实BAR。
- `1.2-intx` - 第19章：带过滤器+任务的单向量中断。
- `1.3-msi` - 第20章：带回退的多向量MSI-X。
- `1.4-dma` - 第21章：`bus_dma`设置、模拟DMA引擎、中断驱动完成。
- `1.5-power` - 第22章：挂起/恢复/关机、重构到`myfirst_power.c`、可选运行时PM。

每个版本建立在前一个之上。有第21章驱动工作的读者可以增量应用第22章更改并最终达到`1.5-power`而无需重写任何先前代码。

### 真实硬件上的最终集成测试

如果读者有真实硬件（有正常S3实现的机器），第22章驱动可以通过完整系统挂起来测试：

```sh
sudo kldload ./myfirst.ko
sudo sh ./ch22-suspend-resume-cycle.sh
sudo acpiconf -s 3
# [laptop sleeps; user opens lid]
# After resume, the DMA test should still work.
sudo sysctl dev.myfirst.0.dma_test_read=1
```

在ACPI S3工作的大多数平台上，驱动在完整周期中存活。`dmesg`输出显示挂起和恢复行，正如`devctl`会触发的那样，确认相同方法代码在两个上下文中运行。

如果全系统测试在每设备测试成功的地方失败，系统挂起做的额外工作（ACPI睡眠状态转换、CPU停车、RAM自刷新）暴露了每设备测试遗漏的东西。通常罪魁祸首是系统低功耗状态重置但每设备D3不重置的设备特定寄存器值。仅用`devctl`测试的驱动可能遗漏这些；在声称正确性之前至少用`acpiconf -s 3`测试一次的驱动更可靠。

### 第22章代码一览

第4阶段驱动添加内容的紧凑摘要：

- **一个新文件**：`myfirst_power.c`，约三百行。
- **一个新头文件**：`myfirst_power.h`，约三十行。
- **一个新markdown文档**：`POWER.md`，约二百行。
- **五个新softc字段**：`suspended`、`saved_intr_mask`、`power_suspend_count`、`power_resume_count`、`power_shutdown_count`，加上启用该功能时的运行时PM字段。
- **三行新`DEVMETHOD`**：`device_suspend`、`device_resume`、`device_shutdown`。
- **三个新助手函数**：`myfirst_mask_interrupts`、`myfirst_drain_dma`、`myfirst_drain_workers`。
- **两个新子系统入口点**：`myfirst_power_setup`、`myfirst_power_teardown`。
- **三个新转换函数**：`myfirst_power_suspend`、`myfirst_power_resume`、`myfirst_power_shutdown`。
- **六个新sysctl**：计数器节点和suspended标志。
- **几个新实验脚本**：cycle、stress、transfer-across-cycle、runtime-PM。

总行增量约为七百行代码，加上几百行文档和脚本。对于本章添加的能力（一个正确处理内核可能抛出的每个电源转换的驱动），这是成比例的投入。

### Wrapping Up Section 8

第8节通过将电源代码拆分为自己的文件、将版本升级到`1.5-power`、添加`POWER.md`文档并连接最终回归测试来结束第22章驱动的构建。模式与第20和21章熟悉：取内联代码、用小API将其提取为子系统、文档化子系统，并通过函数调用而非直接字段访问将其集成到驱动其余部分。

产生的驱动在本章引入的每个意义上都是电源感知的：它处理`DEVICE_SUSPEND`、`DEVICE_RESUME`和`DEVICE_SHUTDOWN`；它干净地静默设备；它在恢复时正确恢复状态；它可选地实现运行时电源管理；它通过sysctl暴露状态；它有回归测试；当平台支持时它在真实硬件上存活完整系统挂起。



## 深入理解：/usr/src/sys/dev/re/if_re.c中的电源管理

Realtek 8169和兼容的千兆NIC由`re(4)`驱动处理。对于第22章目的来说，它是一个值得阅读的信息性驱动，因为它实现了带网络唤醒支持的完整挂起-恢复-关机三重奏，其代码已足够稳定代表规范的FreeBSD模式。完成第22章的读者可以打开`/usr/src/sys/dev/re/if_re.c`并立即识别结构。

> **阅读本演练。** 下面小节中成对的`re_suspend()`和`re_resume()`清单取自`/usr/src/sys/dev/re/if_re.c`，方法表摘录用`/* ... other methods ... */`注释缩写了完整的`re_methods[]`数组，使三个电源相关的`DEVMETHOD`条目突出。我们保持了签名、锁获取和锁释放模式，以及设备特定调用（`re_stop`、`re_setwol`、`re_clrwol`、`re_init_locked`）的顺序不变；真实方法表有更多条目，周围文件包含助手实现。清单中命名的每个符号都是`if_re.c`中的真实FreeBSD标识符，可以通过符号搜索找到。

### 方法表

`re(4)`驱动的方法表在顶部附近包含三个电源方法：

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

这正是第22章教授的模式。`myfirst`驱动的方法表看起来一样。

### re_suspend

挂起函数大约十几行：

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

三个调用完成工作：`re_stop`静默NIC（禁用中断、停止DMA、停止RX和TX引擎），`re_setwol`编程网络唤醒逻辑并在WoL启用时调用`pci_enable_pme`，`sc->suspended = 1`设置softc标志。

与`myfirst_power_suspend`比较：

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

结构是相同的。`re_stop`和`re_setwol`一起等效于`myfirst_quiesce`；本章驱动没有wake-on-X，所以没有`re_setwol`的类似物。

### re_resume

恢复函数大约三十行：

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

这些步骤干净地映射到第22章的纪律：

1. **将控制器从睡眠模式取出**（某些Realtek部件上的MAC睡眠位）。这是设备特定的恢复步骤。
2. **通过`re_clrwol`清除任何WOL模式**，它逆转`re_setwol`所做的。这还通过清除隐式调用`pci_clear_pme`。
3. **如果挂起前接口已启动，重新初始化接口**。`re_init_locked`与挂载调用以启动NIC的函数相同；它重新编程MAC、重置描述符环、启用中断、启动DMA引擎。
4. **在锁下清除suspended标志**。

`myfirst_power_resume`的等效：

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

结构再次相同。`myfirst_restore`对应MAC睡眠退出、`re_clrwol`、`re_init_locked`和标志清除的组合。

### re_shutdown

关机函数为：

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

与`re_suspend`类似，加上接口标志清除（关机是最终的；标记接口为down防止虚假活动）。模式几乎相同；`re_shutdown`本质上是`re_suspend`的更具防御性的版本。

### re_setwol

网络唤醒设置值得一看，因为它展示了真实驱动如何调用PCI PM API：

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

这里出现三个关键模式，值得复制到任何支持wake-on-X的电源感知驱动中：

1. **`pci_has_pm(dev)`守卫。** 如果设备不支持电源管理，函数提前返回。这防止写入不存在的寄存器。
2. **设备特定唤醒编程。** 函数主体通过`CSR_WRITE_1`写入Realtek特定寄存器。不同设备的驱动会写入不同寄存器，但位置（在挂起路径内，`pci_enable_pme`之前）相同。
3. **条件`pci_enable_pme`。** 仅在用户实际请求了wake-on-X时启用PME#。如果用户没有，函数仍设置相关配置位（为与驱动接口能力一致）但不调用`pci_enable_pme`。

逆向是`re_clrwol`：

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

注意`re_clrwol`不显式调用`pci_clear_pme`；PCI层的`pci_resume_child`已在驱动的`DEVICE_RESUME`之前调用了它。`re_clrwol`负责撤销WoL配置的驱动可见侧，而非内核可见的PME状态。

### 深入理解展示了什么

Realtek驱动在各方面都比`myfirst`更复杂（更多寄存器、更多状态、更多设备变体），但其电源管理纪律却不太复杂。这是因为*设备*的复杂性不一对一映射到*电源管理代码*的复杂性。第22章的纪律向下扩展和向上扩展一样好：简单设备有简单电源路径；复杂设备有适度更复杂的电源路径。结构是相同的。

完成第22章的读者现在可以打开`if_re.c`，识别每个函数和每个模式，并理解每个为什么存在。这种理解力可以迁移：相同的识别适用于`if_xl.c`、`virtio_blk.c`和数百个其他FreeBSD驱动。第22章不是在教授`myfirst`特定的API；它在教授FreeBSD电源管理惯用法，`myfirst`驱动是使之具体的载体。



## 深入理解：if_xl.c和virtio_blk.c中的更简单模式

作为对比，另外两个FreeBSD驱动以更简单的方式实现电源管理。

### if_xl.c：关机调用挂起

/usr/src/sys/dev/xl/if_xl.c`中的3Com EtherLink III驱动有最小的三方法设置：

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

两件事引人注目：

1. `xl_shutdown`是一行：它只调用`xl_suspend`。对于此驱动，关机和挂起做相同工作，代码不需要两份副本。
2. softc中没有`suspended`标志。驱动假设正常的挂载→运行→挂起→恢复生命周期，并使用TX路径已检查的`IFF_DRV_RUNNING`标志作为等效。这对于主要用户可见状态是接口运行状态的NIC是完全有效的方法。

对于`myfirst`驱动，显式`suspended`标志更受偏好因为驱动没有`IFF_DRV_RUNNING`的自然等效。NIC驱动可以重用它已有的；学习驱动声明它需要的。

### virtio_blk.c：最小静默

/usr/src/sys/dev/virtio/block/virtio_blk.c`中的virtio块驱动有更短的挂起路径：

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

注释`/* XXX BMV: virtio_stop(), etc needed here? */`是作者不确定静默应该多彻底的诚实承认。现有代码设置标志、等待队列排空（这就是`vtblk_quiesce`做的），然后返回。恢复时，它清除标志并重启I/O。

对于virtio块设备，这足够了因为virtio主机（虚拟机监控程序）在客户说它正在挂起时实现自己的静默。驱动只需停止提交新请求；主机处理其余。

这展示了一个重要模式：**驱动的静默深度取决于硬件状态有多少是驱动的责任**。裸机驱动（如`re(4)`）必须小心编程硬件寄存器因为硬件没有其他盟友。virtio驱动有虚拟机监控程序作为盟友；主机可以为客户处理大部分状态。`myfirst`驱动运行在模拟后端上，处于类似位置：模拟是盟友，驱动的静默可以相应更简单。

### 比较显示了什么

并排阅读多个驱动的电源管理代码是建立流利度的最佳方式之一。每个驱动将第22章模式适应其上下文：`re(4)`处理网络唤醒，`xl(4)`重用`xl_shutdown = xl_suspend`，`virtio_blk(4)`信任虚拟机监控程序。共同线索是结构：停止活动、保存状态、标记已挂起、从挂起返回0；恢复时，清除标志、恢复状态、重启活动、返回0。

心中有第22章的读者可以打开任何FreeBSD驱动，在方法表中找到其`device_suspend`和`device_resume`，阅读这两个函数。几分钟内驱动的电源策略就清楚了。这项技能迁移到读者将来工作的每个驱动；它是本章最有用的单一收获。



## 深入理解：ACPI睡眠状态详解

第1节以列表形式介绍了ACPI S状态。值得以驱动的视角重新审视它们，因为驱动看到的东西略有不同，取决于内核正在进入哪个S状态。

### S0：工作状态

S0是读者在第16到21章中一直工作的状态。CPU执行中，RAM刷新中，PCIe链路已启动。从驱动角度看，S0是连续的；一切正常。

然而在S0内，仍可以有细粒度电源转换。CPU可以在调度器节拍之间进入空闲状态（C1、C2、C3等）。PCIe链路可以基于ASPM进入L0s或L1。设备可以基于运行时PM进入D3。这些都不需要驱动做任何超出自身运行时PM逻辑的事情；它们是透明的。

### S1：待机

S1历史上是最轻的睡眠状态。CPU停止执行但其寄存器被保留；RAM保持供电；设备电源保持D0或D1。唤醒延迟快（一秒以内）。

在现代硬件上，S1很少被支持。平台的BIOS只广播S3及更深的状态。如果平台确实广播S1且用户进入它，驱动的`DEVICE_SUSPEND`仍被调用；驱动做其通常的静默。区别是PCI层通常不为S1转换到D3（因为总线保持供电），所以设备在转换期间保持D0。驱动的保存和恢复大部分未使用。

干净支持S1的驱动也支持S3，因为驱动侧工作是子集。为第22章编写的驱动不需要特殊处理S1。

### S2：保留

S2在ACPI规范中定义但几乎从未实现。驱动可以安全忽略它；FreeBSD的ACPI层根据平台支持将S2视为S1或S3。

### S3：挂起到内存

S3是第22章瞄准的规范睡眠状态。当用户进入S3时：

1. 内核的挂起序列遍历设备树，在每个驱动上调用`DEVICE_SUSPEND`。
2. PCI层的`pci_suspend_child`为每个PCI设备缓存配置空间。
3. PCI层将每个PCI设备转换到D3hot。
4. 更高级子系统（ACPI、CPU的空闲机制）进入自己的睡眠状态。
5. CPU的上下文保存到RAM；CPU停止。
6. RAM进入自刷新；内存控制器以最小功率维持内容。
7. 平台的唤醒电路装备就绪：电源按钮、盖子开关和任何配置的唤醒源。
8. 系统等待唤醒事件。

当唤醒事件到达时：

1. CPU恢复；其上下文从RAM恢复。
2. 更高级子系统恢复。
3. PCI层遍历设备树并为每个设备调用`pci_resume_child`。
4. 每个设备转换到D0；其配置恢复；待处理的PME#被清除。
5. 每个驱动的`DEVICE_RESUME`被调用。
6. 用户空间解冻。

驱动只看到每个序列的步骤1（挂起）和步骤5（恢复）。其余是内核和平台机制。

一个微妙的点：在S3期间，RAM被刷新但内核未运行。这意味着任何内核侧状态（softc、DMA缓冲区、待处理任务）在S3中不变地存活。唯一可能丢失的是硬件状态：设备中的配置寄存器可能被重置；BAR映射寄存器可能返回默认值。驱动在恢复时的工作是从保留的内核状态重新编程硬件。

### S4：挂起到磁盘（休眠）

S4是"休眠"状态。内核将RAM的全部内容写入磁盘映像，然后进入S5。唤醒时，平台启动，内核读取映像回，系统从它离开的地方继续。

在FreeBSD上，S4历史上一直不完整。内核可以在某些平台上产生休眠映像，但恢复路径不如Linux成熟。对于驱动目的，S4与S3相同：`DEVICE_SUSPEND`和`DEVICE_RESUME`方法被调用；驱动的静默和恢复路径无需更改即可工作。额外的平台级工作（写入映像）是透明的。

驱动可能注意到的一个区别是S4恢复后，PCI配置空间始终从头恢复（平台已完全重启），所以即使驱动依赖`hw.pci.do_power_suspend`为0保持设备在D0，S4后设备仍会经历完整电源周期。这仅对在挂起期间做平台特定技巧的驱动有影响；大多数驱动对此无感知。

### S5：软关机

S5是系统断电。电源按钮、电池（如果有）和唤醒电路仍接收电源；其他一切关闭。

从驱动角度看，S5看起来像关机：`DEVICE_SHUTDOWN`被调用（不是`DEVICE_SUSPEND`），驱动将设备置于安全断电状态，系统停止。S5没有对应的恢复；如果用户按电源按钮，系统从头启动。

关机不是可逆意义上的电源转换；它是终止。驱动的`DEVICE_SHUTDOWN`方法被调用一次，驱动不期望在下次启动前再次运行。本章的`myfirst_power_shutdown`通过静默设备（与挂起相同）且不尝试保存任何状态（因为没有恢复需要保存）来正确处理这一点。

### 观察平台支持的状态

在任何带ACPI的FreeBSD 14.3系统上，支持的状态通过sysctl暴露：

```sh
sysctl hw.acpi.supported_sleep_state
```

典型输出：

- 现代笔记本：`S3 S4 S5`
- 服务器：`S5`（许多服务器平台不支持挂起）
- bhyve上的VM：各不相同；通常只有`S5`
- 带有`-machine q35`的QEMU/KVM上的VM：通常`S3 S4 S5`

如果驱动要在特定平台上工作，支持状态列表告诉您需要测试哪些转换。仅在服务器上运行的驱动不需要S3测试；为笔记本设计的驱动需要。

### 测试内容

对于第22章的目的，最低测试是：

- `devctl suspend` / `devctl resume`：始终可能；测试驱动侧代码路径。
- `acpiconf -s 3`（如果支持）：测试完整系统挂起。
- 系统关机（`shutdown -p now`）：测试`DEVICE_SHUTDOWN`方法。

S4和运行时PM是可选的；它们练习较少使用的代码路径。在支持S3的平台上通过最低测试的驱动状态良好；扩展是锦上添花。

### 睡眠状态到驱动程序方法的映射

每种转换调用哪个kobj方法的紧凑表格：

| 转换                | 方法               | 驱动操作                                         |
|---------------------|--------------------|--------------------------------------------------|
| S0 → S1             | DEVICE_SUSPEND     | 静默；保存状态                                    |
| S0 → S3             | DEVICE_SUSPEND     | 静默；保存状态（设备可能进入D3）                    |
| S0 → S4             | DEVICE_SUSPEND     | 静默；保存状态（随后休眠）                          |
| S0 → S5（关机）      | DEVICE_SHUTDOWN    | 静默；使设备处于断电安全状态                       |
| S1/S3 → S0          | DEVICE_RESUME      | 恢复状态；取消屏蔽中断                             |
| S4 → S0（恢复）      | （从启动挂载）       | 正常挂载，因为内核是全新引导的                      |
| devctl suspend      | DEVICE_SUSPEND     | 静默；保存状态（设备进入D3）                       |
| devctl resume       | DEVICE_RESUME      | 恢复状态；取消屏蔽中断                             |

驱动不从自己的代码中区分S1、S3和S4；它总是做相同的工作。差异在平台和内核层。这种统一性使模式可扩展：一个挂起路径，一个恢复路径，多个上下文。



## 深入理解：PCIe链路状态和ASPM实战

第1节概述了PCIe链路状态（L0、L0s、L1、L1.1、L1.2、L2）。值得在实践中看看它们的行为，因为理解它们有助于驱动开发者解释延迟测量和功耗观察。

### 为什么链路有自己的状态

PCIe链路是两个端点（根复合体和设备，或根复合体和交换器）之间的一对高速差分通道。每个通道有发送器和接收器；每个通道的发送器消耗功率保持通道在已知状态。当流量低时，发送器可以不同程度关闭，链路可以在流量恢复时快速重新建立。L状态描述这些程度。

链路的状态独立于设备的D状态。D0中的设备可以使其链路处于L1（链路空闲；设备不在传输或接收）。D3中的设备使其链路处于L2或类似状态（链路关闭）。D0中带繁忙链路的设备处于L0。

### L0：活动

L0是正常操作状态。链路两侧都活跃；数据可以双向流动；延迟最小（现代PCIe上往返几百纳秒）。

当DMA传输正在运行或MMIO读取待处理时，链路处于L0。设备自身的逻辑和PCIe主机桥都需要L0进行事务。

### L0s：发送器待机

L0s是链路一侧发送器关闭的低功耗状态。接收器保持开启；链路可以在一微秒内回到L0。

L0s在几微秒没有发送流量时由链路逻辑自动进入。平台的PCIe主机桥和设备的PCIe接口协作：当发送FIFO为空且ASPM启用时，发送器关闭。当新流量到达时，发送器重新开启。

L0s是"非对称的"：每侧独立进入和退出状态。设备的发送器可以在L0s中而根复合体的发送器在L0中。这很有用因为流量通常是突发的：CPU发送DMA触发，然后一段时间不发送任何东西；CPU的发送器快速进入L0s，而设备的发送器保持在L0因为它正在积极发送DMA响应。

### L1：双方待机

L1是更深的状态，两侧发送器都关闭。在链路回到L0之前，任何一方都不能发送任何东西；延迟以微秒计（5到65，取决于平台）。

L1在比L0s更长的空闲期后进入。确切阈值通过ASPM设置配置；典型值是数十微秒不活动。L1比L0s节省更多功率但退出成本更高。

### L1.1和L1.2：更深的L1子状态

PCIe 3.0及更高版本定义了关闭物理层额外部分的L1子状态。L1.1（也称为"L1 PM子状态1"）保持时钟运行但关闭更多电路；L1.2也关闭时钟。唤醒延迟增加（L1.1数十微秒；L1.2数百微秒），但空闲功耗降低。

大多数现代笔记本积极使用L1.1和L1.2以延长电池寿命。大部分空闲时间保持在L1.2的笔记本PCIe功耗可以是个位毫瓦，而L0中是数百毫瓦。

### L2：近关闭

L2是设备处于D3cold时链路进入的状态。链路实际上关闭；重新建立它需要完整链路训练序列（数十毫秒）。L2作为完整系统挂起序列的一部分进入；驱动不直接管理它。

### 谁控制ASPM

ASPM是通过根复合体和设备两端的PCIe链路能力和链路控制寄存器配置的每链路特性。配置指定：

- 是否启用L0s（一位字段）。
- 是否启用L1（一位字段）。
- 平台认为可接受的退出延迟阈值。

在FreeBSD上，ASPM通常由平台固件通过ACPI的`_OSC`方法控制。固件告诉操作系统管理哪些能力；如果固件保持ASPM控制，操作系统不触碰它。如果固件移交控制，操作系统可以基于策略按链路启用或禁用ASPM。

对于第22章的`myfirst`驱动，ASPM是平台的工作。驱动不配置ASPM；它不需要知道链路在任何时刻处于L0还是L1。从功能角度看，链路的状态对驱动是不可见的（延迟是唯一可观察的效果）。

### ASPM何时对驱动程序重要

在特定情况下驱动确实需要关注ASPM：

1. **已知勘误。** 某些PCIe设备的ASPM实现有缺陷，会导致链路卡死或产生损坏的事务。驱动可能需要为这些设备显式禁用ASPM。内核通过`pcie_read_config`和`pcie_write_config`提供PCIe链路控制寄存器访问用于此目的。

2. **延迟敏感设备。** 实时音频或视频设备可能无法容忍L1的微秒级延迟。驱动可以在保持L0s启用的同时禁用L1。

3. **功耗敏感设备。** 电池供电的设备可能希望L1.2始终启用。如果平台默认不够激进，驱动可以强制L1.2。

对于`myfirst`驱动，这些都不适用。模拟设备根本没有链路；真实PCIe链路（如果有）由平台处理。本章为完整性提及ASPM后继续。

### 观察链路状态

在平台支持ASPM观察的系统上，链路状态通过`pciconf -lvbc`暴露：

```sh
pciconf -lvbc | grep -A 20 myfirst
```

查找如下行：

```text
cap 10[ac] = PCI-Express 2 endpoint max data 128(512) FLR NS
             link x1(x1) speed 5.0(5.0)
             ASPM disabled(L0s/L1)
             exit latency L0s 1us/<1us L1 8us/8us
             slot 0
```

此行上的"ASPM disabled"表示ASPM当前不活跃。"disabled(L0s/L1)"表示设备支持L0s和L1但两者都未启用。在有激进ASPM的系统上，该行会显示"ASPM L1"或类似内容。

退出延迟告诉驱动回到L0的转换需要多长时间；延迟敏感的驱动可以通过查看此数字决定L1是否可容忍。

### 链路状态和功耗

PCIe功耗粗略表格（典型值；实际取决于实现）：

| 状态  | 功耗（x1链路）   | 退出延迟     |
|-------|-----------------|--------------|
| L0    | 100-200 mW      | 0            |
| L0s   | 50-100 mW       | <1 µs        |
| L1    | 10-30 mW        | 5-65 µs      |
| L1.1  | 1-5 mW          | 10-100 µs    |
| L1.2  | <1 mW           | 50-500 µs    |
| L2    | near 0          | 1-100 ms     |

对于空闲时所有12条PCIe链路都在L1.2的笔记本，相对于全L0的总节省可以达到瓦特级别。对于高吞吐量链路始终在L0的服务器，ASPM被禁用，节能为零。

第22章不为`myfirst`实现ASPM。本章提及它因为理解链路状态机是理解完整电源管理图景的一部分。之后在有已知ASPM勘误的驱动上工作的读者会知道去哪里查找。



## 深入理解：唤醒源详解

唤醒源是将挂起的系统或设备带回活动状态的机制。第1节简要提及了它们；这个深入理解遍历最常见的几种。

### PCIe上的PME#

PCI规范定义`PME#`信号（电源管理事件）。断言时，它告诉上游根复合体设备有值得唤醒的事件。根复合体将PME#转换为ACPI GPE或中断，由内核处理。

支持PME#的设备有PCI电源管理能力（通过`pci_has_pm`检查）。能力的控制寄存器包括：

- **PME_En**（位8）：启用PME#生成。
- **PME_Status**（位15）：设备在PME#被引发时设置，由软件清除。
- **PME_Support**（只读，PMC寄存器中的位11-15）：设备可以从哪些D状态引发PME#（D0、D1、D2、D3hot、D3cold）。

驱动的工作是在正确的时间设置PME_En（通常在挂起前）和在正确的时间清除PME_Status（通常在恢复后）。`pci_enable_pme(dev)`和`pci_clear_pme(dev)`助手完成这两个工作。

在典型笔记本上，根复合体将PME#路由到ACPI GPE，内核的ACPI驱动将其作为唤醒事件拾取。链如下：

```text
device asserts PME#
  → root complex receives PME
  → root complex sets GPE status bit
  → ACPI hardware interrupts CPU
  → kernel wakes from S3
  → kernel's ACPI driver services the GPE
  → eventually: DEVICE_RESUME on the device that woke
```

整个链需要一到三秒。驱动的角色很小：它在挂起前启用PME#，它将在恢复后清除PME_Status。其余是平台的工作。

### USB远程唤醒

USB有自己的唤醒机制称为"远程唤醒"。USB设备通过其标准描述符请求唤醒能力；主机控制器在枚举时启用该能力；当设备在其上游端口上断言恢复信号时，主机控制器传播它。

从FreeBSD驱动角度看，USB远程唤醒几乎完全由USB主机控制器驱动（`xhci`、`ohci`、`uhci`）处理。单个USB设备驱动（用于键盘、存储、音频等）通过USB框架自己的挂起和恢复回调参与，但它们不直接处理PME#。USB主机控制器自身的PME#是实际唤醒系统的东西。

对于第22章的目的，USB唤醒是通过USB主机控制器驱动工作的黑盒。最终编写USB设备驱动的读者将在那时学习框架的约定。

### 嵌入式平台上的GPIO唤醒

在嵌入式平台（arm64、RISC-V）上，唤醒源通常是连接到SoC唤醒逻辑的GPIO引脚。设备树通过`wakeup-source`属性和指向唤醒控制器的`interrupts-extended`描述哪些引脚是唤醒源。

FreeBSD的GPIO intr框架处理这些。硬件具有唤醒能力的设备驱动在挂载期间读取设备树`wakeup-source`属性，用框架将GPIO注册为唤醒源，框架做其余的工作。该机制与PCIe PME#非常不同，但驱动侧API（标记唤醒启用、清除唤醒状态）在概念上相似。

第22章不练习GPIO唤醒；`myfirst`驱动是PCI设备。第七部分重新审视嵌入式平台并详细覆盖GPIO路径。

### 局域网唤醒（WoL）

局域网唤醒是网络控制器的特定实现模式。控制器监视传入数据包中的"魔法数据包"（包含控制器MAC地址重复多次的特定模式）或用户配置的模式。当检测到匹配时，控制器向上游断言PME#。

从驱动角度看，WoL需要：

1. 在挂起前配置NIC的唤醒逻辑（魔法数据包过滤器、模式过滤器）。
2. 通过`pci_enable_pme`启用PME#。
3. 在恢复时，禁用唤醒逻辑（因为否则正常数据包处理会被过滤器影响）。

`re(4)`驱动的`re_setwol`是规范的FreeBSD例子。构建NIC驱动的读者复制其结构并适应设备特定的寄存器编程。

### 盖子、电源按钮等唤醒

笔记本的盖子开关、电源按钮、键盘（在某些情况下）和其他平台输入通过ACPI连接到平台的唤醒逻辑。ACPI驱动处理唤醒；单个设备驱动不参与。

ACPI命名空间中设备对象上的ACPI `_PRW`方法声明该设备的唤醒事件使用哪个GPE。操作系统在启动时读取`_PRW`以配置唤醒路由。`myfirst`驱动作为没有平台特定唤醒源的简单PCI端点，没有`_PRW`方法；其唤醒能力（如果有）纯粹通过PME#。

### 驱动程序何时必须启用唤醒

简单的启发式：驱动必须在用户请求时（通过接口能力标志如NIC的`IFCAP_WOL`）且硬件支持时（`pci_has_pm`返回true，设备自身唤醒逻辑可操作）启用唤醒。否则，驱动保持唤醒禁用。

默认为每个设备启用唤醒的驱动浪费平台功率；唤醒电路和PME#路由持续消耗几毫瓦。从不启用唤醒的驱动让希望笔记本在收到网络数据包时唤醒的用户沮丧。策略是"仅在请求时启用"。

FreeBSD的接口能力（通过`ifconfig em0 wol wol_magic`设置）是用户表达愿望的标准方式。NIC驱动读取标志并相应配置WoL。

### 测试唤醒源

测试唤醒比测试挂起和恢复更难，因为测试唤醒需要系统实际睡眠然后外部事件唤醒它。常见方法：

- **从另一台机器发送魔法数据包。** 向挂起机器的MAC地址发送WoL魔法数据包。如果WoL工作，机器在几秒内唤醒。
- **盖子开关。** 关闭盖子、等待、打开盖子。如果平台的唤醒路由工作，机器在打开时唤醒。
- **电源按钮。** 在挂起时短暂按电源按钮。机器应唤醒。

对于像`myfirst`这样的学习驱动，没有有意义的唤醒源可以测试。本章出于教学完整性提及唤醒机制，不是因为驱动练习它们。



## 深入理解：hw.pci.do_power_suspend可调参数

电源管理调试中最重要的可调参数之一是`hw.pci.do_power_suspend`。它控制PCI层是否在系统挂起期间自动将设备转换到D3。理解它做什么以及何时更改它值得专门看一下。

### 默认值的作用

`hw.pci.do_power_suspend=1`（默认）时，PCI层的`pci_suspend_child`助手在调用驱动的`DEVICE_SUSPEND`后，通过调用`pci_set_power_child(dev, child, PCI_POWERSTATE_D3)`将设备转换到D3hot。恢复时，`pci_resume_child`转回D0。

这是"节能"模式。支持D3的设备在挂起期间使用其最低功耗空闲状态。笔记本受益因为睡眠期间电池寿命延长；能以几毫瓦而非几百毫瓦睡眠的设备值得额外的D状态转换。

### hw.pci.do_power_suspend=0的作用

可调参数设为0时，PCI层不将设备转换到D3。设备在整个挂起期间保持在D0。驱动的`DEVICE_SUSPEND`运行；驱动静默活动；设备保持供电。

从节能角度看，这更差：设备在睡眠期间继续消耗D0功率预算。从正确性角度看，对某些设备可以更好：

- 有损坏D3实现的设备在转换时可能行为异常。保持D0避免转换错误。
- 上下文保存和恢复代价昂贵的设备在短暂挂起期间可能偏好保持D0。如果挂起仅几秒，上下文保存成本超过节能收益。
- 对机器核心功能关键的设备（例如控制台键盘）可能需要在挂起期间保持警觉。

### 何时更改

对于开发和调试，设置`hw.pci.do_power_suspend=0`可以隔离错误：

- 如果恢复错误仅在可调参数为1时出现，错误在D3到D0转换中（在PCI层的配置恢复中，或在驱动处理已重置设备中）。
- 如果恢复错误在可调参数为0时也出现，错误在驱动的`DEVICE_SUSPEND`或`DEVICE_RESUME`代码中，不在D状态机制中。

对于生产，默认（1）几乎总是正确的。全局更改它影响系统上的每个PCI设备；更好的方法是如果需要则按设备覆盖，通常位于驱动自身中。

### 验证可调参数生效

快速验证方法是在挂起前后用`pciconf`检查设备的电源状态：

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

`pciconf -lvbc`输出中的`powerspec`行显示当前电源状态。观察它在D0和D3之间变化确认自动转换正在发生。

### 与pci_save_state的交互

当`hw.pci.do_power_suspend`为1时，PCI层在转换到D3之前自动调用`pci_cfg_save`。当为0时，PCI层不调用`pci_cfg_save`。

这有一个微妙的含义：如果驱动想在0情况下显式保存配置，它必须自己调用`pci_save_state`。第22章的模式假设默认（1）并且不显式调用`pci_save_state`；想要支持两种模式的驱动需要额外逻辑。

### 可调参数影响系统挂起还是devctl挂起？

两者都影响。`pci_suspend_child`既为`acpiconf -s 3`也为`devctl suspend`调用，可调参数在两种情况下都控制D状态转换。用`devctl suspend`调试的读者将看到与完整系统挂起相同的行为，除了其他平台工作（CPU停车、ACPI睡眠状态进入）。

### 具体调试场景

假设`myfirst`驱动的恢复间歇性失败：有时工作，有时恢复后`dma_test_read`返回EIO。计数器一致（挂起计数=恢复计数），日志显示两个方法都运行了，但恢复后DMA失败。

**假设1。** D3到D0转换产生了不一致的设备状态。通过设置`hw.pci.do_power_suspend=0`并重试验证。

如果错误在可调参数为0时消失，D状态机制涉及其中。修复可能在驱动的恢复路径中（在转换后添加延迟让设备稳定）、在PCI层的配置恢复中、或在设备本身中。

**假设2。** 错误在驱动自己的挂起/恢复代码中，独立于D3。通过设置可调参数为0并重试验证。

如果错误在可调参数为0时持续，驱动的代码是问题。D3转换是清白的。

这种二分法在电源管理调试中很常见。可调参数是让您隔离变量的工具。



## 深入理解：DEVICE_QUIESCE和何时需要它

第2节简要提及`DEVICE_QUIESCE`作为`DEVICE_SUSPEND`和`DEVICE_SHUTDOWN`旁边的第三个电源管理方法。它在FreeBSD驱动中很少被显式实现；搜索`/usr/src/sys/dev/`显示只有少数驱动定义了自己的`device_quiesce`。理解何时需要它何时不需要值得一小节。

### DEVICE_QUIESCE的用途

`/usr/src/sys/kern/subr_bus.c`中的`device_quiesce`包装器在几个地方被调用：

- `devclass_driver_deleted`：当驱动被卸载时，框架在调用`device_detach`之前在每个实例上调用`device_quiesce`。
- 通过devctl的`DEV_DETACH`：当用户运行`devctl detach myfirst0`时，内核在`device_detach`之前调用`device_quiesce`，除非给出`-f`（强制）标志。
- 通过devctl的`DEV_DISABLE`：当用户运行`devctl disable myfirst0`时，内核类似地调用`device_quiesce`。

在每种情况下，静默是预检查："驱动能安全停止正在做的事情吗？"。从`DEVICE_QUIESCE`返回EBUSY的驱动阻止后续分离或禁用。用户得到错误，驱动保持挂载。

### 默认值的作用

如果驱动不实现`DEVICE_QUIESCE`，默认（`device_if.m`中的`null_quiesce`）无条件返回0。内核继续分离或禁用。

对于大多数驱动，这没问题。驱动的分离路径处理任何进行中的工作，所以静默能做的分离也都做。

### 何时应该实现它

驱动在以下情况下显式实现`DEVICE_QUIESCE`：

1. **返回EBUSY比等待更有信息量。** 如果驱动有"忙碌"的概念（传输进行中、打开的文件描述符计数、文件系统挂载），且用户可以等待它变为非忙碌，驱动可以拒绝静默直到忙碌为零。`DEVICE_QUIESCE`返回EBUSY告诉用户"设备正忙；请等待并重试"。

2. **静默可以比完整分离更快完成。** 如果分离代价高昂（释放大型资源表、排空慢速队列）但设备可以廉价停止，`DEVICE_QUIESCE`让内核在不付出分离成本的情况下探测就绪状态。

3. **驱动想区分静默和挂起。** 如果驱动想停止活动但不保存状态（因为不会有恢复），将静默与挂起分开实现是在代码中表达这种区别的方式。

对于`myfirst`驱动，这些都不适用。第21章的分离路径已处理进行中的工作；第22章的挂起路径在电源管理意义上处理静默。添加单独的`DEVICE_QUIESCE`是冗余的。

### 来自bce(4)的示例

/usr/src/sys/dev/bce/if_bce.c`中的Broadcom NetXtreme驱动在其方法表中有注释掉的`DEVMETHOD(device_quiesce, bce_quiesce)`条目。注释表明作者考虑了实现静默但没有。这很常见：许多驱动将该行作为TODO注释保留，因为默认处理了它们的用例。

如果驱动启用它，实现会停止NIC的TX和RX路径而不释放硬件资源。后续的`device_detach`然后做实际释放。"停止"和"释放"之间的分割是`DEVICE_QUIESCE`要表达的。

### 与DEVICE_SUSPEND的关系

`DEVICE_QUIESCE`和`DEVICE_SUSPEND`做类似的事：它们停止设备的活动。区别：

- **生命周期**：静默在运行和分离之间；挂起在运行和最终恢复之间。
- **资源**：静默不要求驱动保存任何状态；挂起需要。
- **否决能力**：两者都可以返回EBUSY；后果不同（静默阻止分离；挂起阻止电源转换）。

实现两者的驱动通常共享代码：`foo_quiesce`可能做"停止活动"，`foo_suspend`可能做"调用静默；保存状态；返回"。`myfirst`驱动的`myfirst_quiesce`助手是共享代码；本章不将其连接到`DEVICE_QUIESCE`方法，但这样做是一个小的添加。

### myfirst的可选添加

作为挑战，读者可以向`myfirst`添加`DEVICE_QUIESCE`：

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

以及匹配的方法表条目：

```c
DEVMETHOD(device_quiesce, myfirst_pci_quiesce),
```

测试它：`devctl detach myfirst0`在分离前调用静默；读者可以在分离生效前立即读取`dev.myfirst.0.power_quiesce_count`来验证。

挑战很短且不改变驱动的整体结构；它只是连接了一个更多方法。第22章的合并第4阶段默认不包含它，但想要该方法的读者可以在几行中添加。



## 动手实验

第22章包括三个动手实验，以逐步加难的方式练习电源管理路径。每个实验在`examples/part-04/ch22-power/labs/`中有脚本，读者可以按原样运行，还有扩展想法。

### 实验1：单周期挂起-恢复

第一个实验最简单：带计数器验证的一次干净挂起-恢复周期。

**设置。** 加载第22章第4阶段驱动：

```sh
cd examples/part-04/ch22-power/stage4-final
make clean && make
sudo kldload ./myfirst.ko
```

验证挂载：

```sh
sysctl dev.myfirst.0.%driver
# Should return: myfirst
sysctl dev.myfirst.0.suspended
# Should return: 0
```

**运行。** 执行周期脚本：

```sh
sudo sh ../labs/ch22-suspend-resume-cycle.sh
```

预期输出：

```text
PASS: one suspend-resume cycle completed cleanly
```

**验证。** 检查计数器：

```sh
sysctl dev.myfirst.0.power_suspend_count
# Should return: 1
sysctl dev.myfirst.0.power_resume_count
# Should return: 1
```

检查`dmesg`：

```sh
dmesg | tail -6
```

应显示四行（挂起开始、挂起完成、恢复开始、恢复完成）加上前后传输日志行。

**扩展。** 修改周期脚本运行两次挂起-恢复周期而非一次，并验证计数器各精确递增2。

### 实验2：百周期压力测试

第二个实验连续运行周期脚本一百次并检查没有任何漂移。

**运行。**

```sh
sudo sh ../labs/ch22-suspend-stress.sh
```

几秒后的预期输出：

```text
PASS: 100 cycles
```

**验证。** 压力运行后，计数器应各为100（或100加上之前已有的）：

```sh
sysctl dev.myfirst.0.power_suspend_count
# 100 (or however many cycles were added)
```

**观察内容。**

- 一个周期需要多长时间？在模拟上，应为几毫秒。在带D状态转换的真实硬件上，预期几百微秒到几毫秒。
- 压力期间系统的负载平均值是否变化？模拟代价低；现代机器上一百个周期几乎不应有影响。
- 如果在压力期间运行DMA测试会怎样？（`sudo sysctl dev.myfirst.0.dma_test_read=1`与周期循环并发。）编写良好的驱动应优雅处理；DMA测试如果在`RUNNING`窗口期间发生则成功，如果在转换期间发生则以EBUSY或类似方式失败。

**扩展。** 运行压力脚本前用`dmesg -c`清除日志，之后：

```sh
dmesg | wc -l
```

应接近400（每个周期四行日志，乘以100个周期）。每周期日志行计数让您验证每个周期实际通过驱动执行了。

### 实验3：跨周期传输

第三个实验最难：它启动DMA传输并立即在中间挂起，然后恢复并验证驱动恢复了。

**设置。** 实验脚本是`ch22-transfer-across-cycle.sh`。它在后台运行DMA传输，睡眠几毫秒，调用`devctl suspend`，睡眠，调用`devctl resume`，然后启动另一次传输。

**运行。**

```sh
sudo sh ../labs/ch22-transfer-across-cycle.sh
```

**观察内容。**

- 第一次传输是完成、出错还是超时？预期行为是静默干净地中止它；传输报告EIO或ETIMEDOUT。
- 计数器`dma_errors`或`dma_timeouts`是否递增？其中之一应该。
- 挂起后`dma_in_flight`是否回到false？
- 恢复后传输是否正常成功？如果是，驱动状态一致，周期工作正常。

**扩展。** 减少传输开始和挂起之间的睡眠以命中传输在挂起时刻正在执行中的角落情况。那是竞态条件存在的地方；在激进时序下通过此测试的驱动有坚实的静默实现。

### 实验4：运行时PM（可选）

对于用`MYFIRST_ENABLE_RUNTIME_PM`构建的读者，第四个实验练习运行时PM路径。

**设置。** 启用运行时PM重新构建：

```sh
cd examples/part-04/ch22-power/stage4-final
# Uncomment the CFLAGS line in the Makefile:
#   CFLAGS+= -DMYFIRST_ENABLE_RUNTIME_PM
make clean && make
sudo kldload ./myfirst.ko
```

**运行。**

```sh
sudo sh ../labs/ch22-runtime-pm.sh
```

脚本：

1. 将空闲阈值设为3秒（而非默认的5）。
2. 记录基线计数器。
3. 等待5秒无任何活动。
4. 验证`runtime_state`为`RUNTIME_SUSPENDED`。
5. 触发DMA传输。
6. 验证`runtime_state`回到`RUNNING`。
7. 打印PASS。

**观察内容。**

- 在空闲等待期间，`dmesg`应在大约3秒后显示"runtime suspend"日志行。
- `runtime_suspend_count`和`runtime_resume_count`在结束时各自应为1。
- DMA传输在运行时恢复后应正常成功。

**扩展。** 将空闲阈值设为1秒。在紧凑循环中重复运行DMA测试。循环期间不应看到运行时挂起转换（因为每次测试重置空闲定时器），但循环一停止，运行时挂起就触发。

### 实验说明

所有实验假设驱动已加载且系统足够空闲使转换按需发生。如果另一个进程正在积极使用设备（对`myfirst`不太可能，但在真实设置中常见），计数器会以意外数量漂移，脚本的精确增量检查失败。脚本是为安静测试环境设计的，不是嘈杂的。

对于`re(4)`驱动或其他生产驱动的真实测试，相同的脚本结构适用，只需调整设备名称。`devctl suspend`/`devctl resume`舞蹈适用于内核管理的任何PCI设备。



## Challenge Exercises

第22章的挑战练习将读者推到基线驱动之外，进入现实世界驱动最终必须处理的领域。每个练习的范围都可以通过本章材料和几小时的工作完成。

### 挑战1：实现sysctl唤醒机制

用模拟的唤醒源扩展`myfirst`驱动。模拟已有可以触发的callout；添加一个新的模拟功能，在设备处于D3时在设备上设置"wake"位，并让驱动的`DEVICE_RESUME`路径记录唤醒事件。

**Hints.**

- 向模拟后端添加`MYFIRST_REG_WAKE_STATUS`寄存器。
- 添加驱动在挂起期间写入的`MYFIRST_REG_WAKE_ENABLE`寄存器。
- 让模拟callout在随机延迟后设置唤醒状态位。
- 恢复时，驱动读取寄存器并记录是否观察到唤醒。

**验证。** 执行`devctl suspend; sleep 1; devctl resume`后，日志应显示唤醒状态。后续的`sysctl dev.myfirst.0.wake_events`应增加。

**为什么重要。** 唤醒源处理是真实硬件电源管理中最棘手的部分之一。将其构建到模拟中让读者在不需要硬件的情况下练习完整的契约。

### 挑战2：保存和恢复描述符环

第21章的模拟尚不使用描述符环（传输是逐个进行的）。用小型描述符环扩展模拟，在挂载时通过寄存器编程其基地址，让挂起路径将环的基地址保存到softc状态中。让恢复路径将保存的基地址写回。

**Hints.**

- 环的基地址是保存在softc中的`bus_addr_t`。
- 寄存器为`MYFIRST_REG_RING_BASE_LOW`/`_HIGH`。
- 保存和恢复是微不足道的；关键是要验证*不*保存和恢复会导致问题。

**验证。** 挂起-恢复后，环基寄存器应保持与之前相同的值。如果不恢复，它应保持为零。

**为什么重要。** 描述符环是真实高吞吐量驱动使用的；带环的电源感知驱动必须在每次恢复时恢复基地址。此练习是通往`re(4)`和`em(4)`等生产驱动执行的状态管理类型的跳板。

### 挑战3：实现否决策略

用策略旋钮扩展挂起路径，让用户指定驱动是否应在设备忙碌时否决挂起。具体来说：

- 添加`dev.myfirst.0.suspend_veto_if_busy`作为读写sysctl。
- 如果sysctl为1且有DMA传输进行中，`myfirst_power_suspend`返回EBUSY而不静默。
- 如果sysctl为0（默认），挂起始终成功。

**提示。** 将`suspend_veto_if_busy`设为1。启动一次长DMA传输（向模拟引擎添加`DELAY`使其持续一两秒）。在传输期间调用`devctl suspend myfirst0`。验证挂起返回错误且`dev.myfirst.0.suspended`保持为0。

**验证。** 内核的展开路径运行；驱动仍处于`RUNNING`；传输正常完成。

**为什么重要。** 否决是有效的工具，也是危险的工具。关于是否否决的现实世界策略决策是微妙的（存储驱动通常否决；NIC驱动通常不否决）。实现机制使策略问题变得具体。

### 挑战4：添加恢复后自检

恢复后，对设备进行最小可行测试：向DMA缓冲区写入已知模式，触发写传输，用读传输读回并验证。如果测试失败，标记设备为损坏并使后续操作失败。

**Hints.**

- 将自检添加为从`myfirst_power_resume`中`myfirst_restore`之后运行的助手。
- 使用熟知的模式如`0xDEADBEEF`。
- 使用现有DMA路径；自检只是一次写入和一次读取。

**验证。** 正常操作下，自检始终通过。要验证它能捕获故障，向模拟添加人为的"fail once"机制并触发它；驱动应记录故障并将自身标记为损坏。

**为什么重要。** 自检是可靠性工程的轻量级形式。在定义明确的点捕获自身故障的驱动比静默损坏数据直到用户注意到的驱动更容易调试。

### 挑战5：实现手动pci_save_state/pci_restore_state

大多数驱动让PCI层自动处理配置空间的保存和恢复。扩展第22章驱动以可选地手动执行，通过sysctl `dev.myfirst.0.manual_pci_save`控制。

**Hints.**

- 读取`hw.pci.do_power_suspend`和`hw.pci.do_power_resume`并在手动模式启用时将它们设为0。
- 在挂起路径中显式调用`pci_save_state`，在恢复路径中调用`pci_restore_state`。
- 验证设备在挂起-恢复后仍然正常工作。

**验证。** 无论是否启用手动模式，设备功能应相同。在压力测试前设置sysctl并验证无漂移。

**为什么重要。** 某些真实驱动需要手动保存/恢复，因为PCI层的自动处理会干扰设备特定的怪异行为。了解何时以及如何接管保存/恢复是有用的中级技能。



## Troubleshooting Reference

本节收集读者在学习第22章时可能遇到的常见问题，每个都有简短的诊断和修复。列表设计为可快速浏览；如果问题匹配，跳到相应条目。

### "devctl: DEV_SUSPEND failed: Operation not supported"

驱动未实现`DEVICE_SUSPEND`。要么方法表缺少`DEVMETHOD(device_suspend, ...)`行，要么驱动尚未重新构建和重新加载。

**修复。** 检查方法表。用`make clean && make`重新构建。卸载并重新加载。

### "devctl: DEV_SUSPEND failed: Device busy"

驱动从`DEVICE_SUSPEND`返回`EBUSY`，可能因为挑战3的否决逻辑，或因为设备确实忙碌（DMA进行中、任务运行中）且驱动选择否决。

**修复。** 检查`suspend_veto_if_busy`旋钮是否已设置。检查`dma_in_flight`。等待活动完成后再挂起。

### "devctl: DEV_RESUME failed"

`DEVICE_RESUME`返回非零。日志应有更多细节。

**修复。** 检查`dmesg | tail`。恢复日志行应告诉您什么失败了。通常是硬件特定的初始化步骤未成功。

### Device is suspended but `dev.myfirst.0.suspended` reads 0

驱动的标志与内核状态不同步。可能是静默路径中的错误：标志从未被设置，或被过早清除。

**修复。** 在恢复路径顶部添加`KASSERT(sc->suspended == true)`；在`INVARIANTS`下运行以捕获错误。

### `power_suspend_count != power_resume_count`

一个周期获得了一侧但没有另一侧。检查`dmesg`中的错误；日志应显示序列在哪里中断。

**修复。** 修复缺失的代码路径。通常是缺少计数器更新的提前返回。

### DMA transfers fail after resume

恢复路径没有重新初始化DMA引擎。检查INTR_MASK寄存器、DMA控制寄存器、`saved_intr_mask`值。启用详细日志以查看恢复路径的恢复序列。

**修复。** 向`myfirst_restore`添加缺失的寄存器写入。

### WITNESS complains about a lock held during suspend

挂起路径获取了锁然后调用了睡眠或尝试获取另一个锁的函数。阅读WITNESS消息中的违规锁名称。

**修复。** 在睡眠调用之前释放锁，或重构代码使锁仅在需要时获取。

### System does not wake from S3

`myfirst`下面的某个驱动阻止了恢复。不太可能是`myfirst`本身，除非日志特别显示来自驱动的错误。

**修复。** 引导进入单用户模式，或加载更少驱动，并二分查找。在活动系统中检查`dmesg`中的违规驱动。

### Runtime PM never fires

空闲监视器callout未运行，或`last_activity`时间戳更新太频繁。

**修复。** 验证`callout_reset`正在从挂载路径调用。验证`myfirst_mark_active`未从意外代码路径调用。向callout回调添加日志以确认它触发。

### Kernel panic during suspend

KASSERT失败（在`INVARIANTS`内核上）或锁被不正确持有。恐慌消息标识了违规的文件和行号。

**修复。** 阅读恐慌消息。将文件和行号匹配到代码。一旦识别了位置，修复通常很简单。



## Wrapping Up

第22章通过赋予`myfirst`驱动电源管理纪律来结束第4部分。开始时，`myfirst`在`1.4-dma`版本已是一个有能力的驱动：它挂载到PCI设备、处理多向量中断、通过DMA移动数据、并在分离时清理资源。它缺少的是参与系统电源转换的能力。如果用户合上笔记本盖子或要求内核挂起设备，它会崩溃、泄漏或静默失败。最终，`myfirst`在`1.5-power`版本处理内核可能抛出的每个电源转换：系统挂起到S3或S4、通过`devctl`的每设备挂起、系统关机和可选的运行时电源管理。

八个部分走完了完整的进程。第1节建立了全局视图：为什么驱动关心电源、ACPI S状态和PCI D状态是什么、PCIe L状态和ASPM添加了什么、唤醒源长什么样。第2节介绍了FreeBSD的具体API：`DEVICE_SUSPEND`、`DEVICE_RESUME`、`DEVICE_SHUTDOWN`和`DEVICE_QUIESCE`方法，`bus_generic_suspend`和`bus_generic_resume`助手，以及PCI层的自动配置空间保存和恢复。第1阶段骨架让方法记录日志和计数转换而不做任何实际工作。第3节将挂起骨架变成真正的静默：屏蔽中断、排空DMA、排空工作线程，按此顺序，共享挂起和分离之间的助手函数。第4节编写了匹配的恢复路径：重新启用总线主控、恢复设备特定状态、清除挂起标志、取消屏蔽中断。第5节添加了带空闲监视器callout和显式`pci_set_powerstate`转换的可选运行时电源管理。第6节调查了用户空间接口：`acpiconf`、`zzz`、`devctl suspend`、`devctl resume`、`devinfo -v`和匹配的sysctl。第7节编目了特征性故障模式及其调试模式。第8节将代码重构到`myfirst_power.c`，将版本升级到`1.5-power`，添加了`POWER.md`，并连接了最终回归测试。

第22章没有做的是多队列驱动的分散-收集电源管理（那是第6部分第28章的话题）、热插拔和意外移除集成（第7部分话题）、嵌入式平台电源域（也是第7部分）、或ACPI的AML解释器内部机制（本书不涉及）。每一个都是基于第22章原语的自然扩展，每一个都属于范围匹配的后续章节。基础已就位；专业化添加词汇而无需新基础。

文件布局已经增长：16个源文件（包括`cbuf`）、8个文档文件（`HARDWARE.md`、`LOCKING.md`、`SIMULATION.md`、`PCI.md`、`INTERRUPTS.md`、`MSIX.md`、`DMA.md`、`POWER.md`），以及覆盖每个子系统的扩展回归套件。驱动在结构上与生产FreeBSD驱动平行；学完第16至22章的读者可以打开`if_re.c`、`if_xl.c`或`virtio_blk.c`并识别每个架构部分：寄存器访问器、模拟后端、PCI挂载、中断过滤器和任务、每向量机制、DMA设置和拆卸、同步纪律、电源挂起、电源恢复、干净分离。

### A Reflection Before Chapter 23

第22章是第4部分的最后一章，第4部分教授读者驱动如何与硬件对话。第16至21章介绍了原语：MMIO、模拟、PCI、中断、多向量中断、DMA。第22章介绍了纪律：这些原语如何在电源转换中存活。这七章一起将读者从"不知道驱动是什么"带到"一个处理内核可能抛出的每个硬件事件的工作中的多子系统驱动"。

第22章的教学是可推广的。内化了挂起-静默-保存-恢复模式、驱动与PCI层的交互、运行时PM状态机和调试模式的读者会在每个电源感知的FreeBSD驱动中找到相似的形状。具体硬件不同；结构不变。NIC、存储控制器、GPU或USB主机控制器的驱动将相同的词汇应用于自己的硬件。

从第23章开始的第5部分转移了重点。第4部分是关于驱动到硬件的方向：驱动如何与设备对话。第5部分是关于驱动到内核的方向：驱动如何被维护它的人类调试、追踪、工具化和压力测试。第23章用适用于每个驱动子系统的调试和追踪技术开始这一转变。

### What to Do If You Are Stuck

Three suggestions.

首先，专注于第2阶段挂起和第3阶段恢复路径。如果`devctl suspend myfirst0`后跟`devctl resume myfirst0`成功且后续DMA传输工作，本章的核心就在工作。本章的其他部分在装饰管道的意义上是可选的，但如果管道失败，本章就不在工作，第3节或第4节是诊断的正确位置。

其次，打开`/usr/src/sys/dev/re/if_re.c`并重读`re_suspend`、`re_resume`和`re_setwol`。每个函数大约三十行。每一行映射到第22章的一个概念。完成本章后阅读它们应该感觉像熟悉的领域；真实驱动的模式看起来像是本章更简单模式的详细阐述。

第三，第一遍跳过挑战。实验为第22章的节奏校准；挑战假设本章的材料已扎实。如果现在觉得难以触及，第23章后再回来。

第22章的目标是给驱动电源管理纪律。如果做到了，第23章的调试和追踪机制就成为您已本能所做之事的推广，而非新话题。

## 第4部分检查点

第4部分是本书迄今为止最长最密集的段落。七章涵盖了硬件资源、寄存器I/O、PCI挂载、中断、MSI和MSI-X、DMA和电源管理。在第5部分将模式从"编写驱动"变为"调试和追踪它们"之前，确认硬件面向的故事已内化。

到第4部分结束时，您应该能够在不查找的情况下完成以下每一项：

- 用`bus_alloc_resource_any`或`bus_alloc_resource_anywhere`声明硬件资源，通过`bus_space(9)`读/写和屏障原语访问它，并在分离时干净地释放。
- 通过`bus_space(9)`抽象而非原始指针解引用读写设备寄存器，在不得重排序的序列周围有正确的屏障纪律。
- 通过供应商、设备、子供应商和子设备ID匹配PCI设备；声明其BAR；并在强制分离后存活而不泄漏资源。
- 通过`bus_setup_intr`注册上半部过滤器连同下半部任务或ithread，按内核要求的顺序，并在分离时以相反顺序拆卸。
- 设置MSI或MSI-X向量，具有从MSI-X到MSI到传统INTx的优雅回退阶梯，并在工作负载需要时将向量绑定到特定CPU。
- 使用`bus_dma(9)`分配、映射、同步和释放DMA缓冲区，包括弹跳缓冲区情况。
- 实现带寄存器保存和恢复、I/O静默和恢复后自检的`device_suspend`和`device_resume`。

如果其中任何一项仍需查找，值得重做的实验是：

- 寄存器和屏障：第16章实验1（观察寄存器之舞）和实验8（看门狗遇上寄存器场景）。
- 负载下的模拟硬件：第17章实验6（注入卡住忙碌并观察驱动等待）和实验10（注入内存损坏攻击）。
- PCI挂载和分离：第18章实验4（声明BAR并读取寄存器）和实验5（练习cdev并验证分离清理）。
- 中断处理：第19章实验3（第2阶段，真实过滤器和延迟任务）。
- MSI和MSI-X：第20章实验4（第3阶段，带CPU绑定的MSI-X）。
- DMA：第21章实验4（第3阶段，中断驱动完成）和实验5（第4阶段，重构和回归）。
- 电源管理：第22章实验2（百周期压力）和实验3（跨周期传输）。

第5部分期望以下作为基线：

- 一个已内置可观测性的硬件能力驱动：计数器、sysctl和在重要转换处的`devctl_notify`调用。第23章的调试机制在驱动已自我报告时效果最佳。
- 能够可靠循环驱动的回归脚本，因为第5部分将可重现性变成一等技能。
- 用`INVARIANTS`和`WITNESS`构建的内核。第5部分比第4部分更依赖两者，特别是在第23章。
- 理解驱动代码中的错误就是内核代码中的错误，这意味着仅用户空间调试器不够用，第5部分将教授内核空间工具。

如果这些成立，第5部分为您准备好了。如果某一项仍不稳定，短时间重做相关实验将获得数倍回报。

## 通往第23章的桥梁

第23章题为*调试与追踪*。其范围是查找驱动中错误的专业实践：`ktrace`、`ddb`、`kgdb`、`dtrace`和`procstat`等工具；分析恐慌、死锁和数据损坏的技术；将模糊用户报告转化为可重现测试用例的策略；以及在有限可见性下必须调试内核空间运行代码的驱动开发者心态。

第22章以四种具体方式准备了基础。

首先，**您到处都有可观测性计数器**。第22章驱动通过sysctl暴露挂起、恢复、关机和运行时PM计数器。第23章的调试技术依赖可观测性；已跟踪自身状态的驱动比不跟踪的更容易调试。

其次，**您有回归测试**。第6节的循环和压力脚本是第23章扩展内容的初体验：按需重现错误的能力。无法重现的错误是无法修复的错误；第22章的脚本是第23章添加的更重测试的基础。

第三，**您有可工作的INVARIANTS/WITNESS调试内核**。第22章全程依赖两者；第23章在同一内核上构建`ddb`会话、事后分析和内核崩溃重现。

第四，**您理解驱动代码中的错误就是内核代码中的错误**。第22章遇到了挂起、冻结设备、丢失中断和WITNESS警告。每一个在用户可见意义上都是内核错误；每一个都需要内核空间调试方法。第23章系统地教授该方法。

第23章将涵盖的具体主题：

- 使用`ktrace`和`kdump`实时观察进程的系统调用追踪。
- 使用`ddb`进入内核调试器进行事后分析或实时检查。
- 使用`kgdb`配合核心转储恢复崩溃内核的状态。
- 使用`dtrace`在不修改源代码的情况下进行内核内追踪。
- 使用`procstat`、`top`、`pmcstat`及相关工具进行性能观察。
- 最小化错误的策略：缩小重现器、二分回归、假设和测试。
- 在生产环境中检测驱动而不干扰行为的模式。

您不需要提前阅读。第22章已是充分准备。带上您的`myfirst`驱动`1.5-power`版本、`LOCKING.md`、`INTERRUPTS.md`、`MSIX.md`、`DMA.md`、`POWER.md`、启用`WITNESS`的内核和回归脚本。第23章从第22章结束的地方开始。

第4部分已完成。第23章通过添加可观测性和调试纪律来开启第5部分，这将上周写的驱动与可以维护多年的驱动区分开来。

词汇是您的；结构是您的；纪律是您的。第23章添加下一个缺失的部分：查找和修复仅在生产环境中出现的错误的能力。



## Reference: 第22章快速参考卡

第22章引入的词汇、API、标志和过程的紧凑摘要。

### Vocabulary

- **挂起（Suspend）：** 从D0（完全操作）到设备可以恢复的较低功耗状态的转换。
- **恢复（Resume）：** 从较低功耗状态回到D0的转换。
- **关机（Shutdown）：** 到设备不会返回的最终状态的转换。
- **静默（Quiesce）：** 将设备带到无活动且无待处理工作的状态。
- **系统睡眠状态（S0、S1、S3、S4、S5）：** ACPI定义的系统功耗级别。
- **设备电源状态（D0、D1、D2、D3hot、D3cold）：** PCI定义的设备功耗级别。
- **链路状态（L0、L0s、L1、L1.1、L1.2、L2）：** PCIe定义的链路功耗级别。
- **ASPM（活动状态电源管理）：** L0和L0s/L1之间的自动转换。
- **PME#（电源管理事件）：** 设备在想要唤醒系统时断言的信号。
- **唤醒源：** 挂起设备可以通过其请求唤醒的机制。
- **运行时PM：** 系统保持在S0时的设备级节能。

### Essential Kobj Methods

- `DEVMETHOD(device_suspend, foo_suspend)`：在电源转换前调用来静默设备。
- `DEVMETHOD(device_resume, foo_resume)`：在电源转换后调用来恢复设备。
- `DEVMETHOD(device_shutdown, foo_shutdown)`：调用来使设备处于重启的安全状态。
- `DEVMETHOD(device_quiesce, foo_quiesce)`：调用来停止活动而不拆除资源。

### Essential PCI APIs

- `pci_has_pm(dev)`：如果设备有电源管理能力则返回true。
- `pci_set_powerstate(dev, state)`：转换到`PCI_POWERSTATE_D0`、`D1`、`D2`或`D3`。
- `pci_get_powerstate(dev)`：当前电源状态。
- `pci_save_state(dev)`：缓存配置空间。
- `pci_restore_state(dev)`：将缓存的配置空间写回。
- `pci_enable_pme(dev)`：启用PME#生成。
- `pci_clear_pme(dev)`：清除待处理的PME状态。
- `pci_enable_busmaster(dev)`：重置后重新启用总线主控。

### Essential Bus Helpers

- `bus_generic_suspend(dev)`：以相反顺序挂起所有子设备。
- `bus_generic_resume(dev)`：以正向顺序恢复所有子设备。
- `device_quiesce(dev)`：调用驱动的`DEVICE_QUIESCE`。

### Essential Sysctls

- `hw.acpi.supported_sleep_state`：平台支持的S状态列表。
- `hw.acpi.suspend_state`：`zzz`的默认S状态。
- `hw.pci.do_power_suspend`：挂起时自动D0->D3转换。
- `hw.pci.do_power_resume`：恢复时自动D3->D0转换。
- `dev.N.M.suspended`：驱动自身的挂起标志。
- `dev.N.M.power_suspend_count`、`power_resume_count`、`power_shutdown_count`。
- `dev.N.M.runtime_state`、`runtime_suspend_count`、`runtime_resume_count`。

### Useful Commands

- `acpiconf -s 3`：进入S3。
- `zzz`：`acpiconf`的包装器。
- `devctl suspend <device>`：每设备挂起。
- `devctl resume <device>`：每设备恢复。
- `devinfo -v`：带状态的设备树。
- `pciconf -lvbc`：带电源状态的PCI设备。
- `sysctl -a | grep acpi`：所有ACPI相关变量。

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

- `/usr/src/sys/kern/device_if.m`：kobj方法定义。
- `/usr/src/sys/kern/subr_bus.c`：`bus_generic_suspend`、`bus_generic_resume`、`device_quiesce`。
- `/usr/src/sys/dev/pci/pci.c`：`pci_suspend_child`、`pci_resume_child`、`pci_save_state`、`pci_restore_state`。
- `/usr/src/sys/dev/pci/pcivar.h`：`PCI_POWERSTATE_*`常量和内联API。
- `/usr/src/sys/dev/re/if_re.c`：带WoL的挂起/恢复生产参考。
- `/usr/src/sys/dev/xl/if_xl.c`：最小挂起/恢复模式。
- `/usr/src/sys/dev/virtio/block/virtio_blk.c`：virtio风格的静默。



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
