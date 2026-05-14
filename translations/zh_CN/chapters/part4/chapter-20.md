---
title: "高级中断处理"
description: "第20章将第19章中断驱动扩展为支持MSI和MSI-X。它讲解传统INTx、MSI和MSI-X的区别；如何用pci_msi_count(9)和pci_msix_count(9)查询能力计数；如何用pci_alloc_msi(9)和pci_alloc_msix(9)分配向量；如何构建从MSI-X向下到MSI再向下到传统INTx的回退阶梯；如何用分离的driver_filter_t函数注册每向量过滤器处理程序；如何设计中断安全的每向量数据结构；如何给每个向量特定角色和特定CPU亲和；以及如何安全拆除多向量驱动。驱动程序从1.2-intr发展到1.3-msi，新增一个msix专用文件，并为第21章的DMA做好准备。"
partNumber: 4
partName: "硬件与平台级集成"
chapter: 20
lastUpdated: "2026-04-19"
status: "complete"
author: "Edson Brandi"
reviewer: "TBD"
translator: "AI辅助翻译为简体中文"
language: "zh-CN"
estimatedReadTime: 165
---

# 高级中断处理

## 读者指南与学习目标

第19章结束时，驱动程序可以倾听其设备。`myfirst`模块版本`1.2-intr`有在传统PCI INTx线上注册的一个过滤器处理程序、在taskqueue上的一个延迟工作任务、用于bhyve实验室目标测试的模拟中断sysctl、严格拆除顺序、以及保持中断代码整洁的新`myfirst_intr.c`文件。第11章锁定纪律延续：过滤器中用原子、任务中用睡眠锁、处理程序上`INTR_MPSAFE`、通过`FILTER_STRAY`的共享IRQ安全。驱动程序行为像碰巧有单个中断源的小型真实驱动程序。

驱动程序尚未做的是利用PCIe提供的所有东西。现代PCIe设备不需要与其邻居共享单条线。它可以通过PCI 2.2引入的消息信号机制（MSI）或PCIe添加的更丰富的每功能表MSI-X请求专用中断。有多个队列的设备（有接收和发送队列的NIC、有admin和I/O提交队列的NVMe控制器、有事件队列的现代USB3主控制器）通常希望每个队列一个中断而不是整个设备一个共享中断。第20章教驱动程序如何请求。

本章范围正是此转换：MSI和MSI-X在硬件层面是什么、FreeBSD如何以额外IRQ资源表示它们、驱动程序如何查询能力计数并分配向量、从MSI-X向下到MSI向下到传统INTx的回退阶梯实践上如何工作、如何在同一设备注册几个不同过滤器函数、如何设计每向量数据结构使每个向量的处理程序触及自己的状态、如何给每个向量匹配设备NUMA放置的CPU亲和、如何用`bus_describe_intr(9)`标记每个向量使`vmstat -i`告诉操作者哪个向量做什么、以及如何以正确顺序拆除所有这些内容。本章暂不涉及DMA，那是第21章；一旦描述符环进入画面，每队列接收和发送向量变得特别有价值，但同时教授两者会稀释两者。

第20章保持几个相邻话题在一定距离。完整DMA（`bus_dma(9)`标签、描述符环、bounce缓冲区、DMA描述符周围的缓存一致性）在第21章。Iflib的多队列框架，用一层每队列iflib机制包装MSI-X，是第6部分话题（第28章）供想要iflib风格网络路径的读者。更丰富的每功能MSI-X掩码表操作（通过MSI-X表直接将特定消息地址引导到特定CPU）讨论但不端到端实现。通过IOMMU的平台特定中断重映射、SR-IOV向量共享和PCIe AER驱动中断恢复留给后续章节。第20章保持在它能良好覆盖的范围内，在话题值得单独成章的地方明确移交。

多向量工作依赖每个较早的第4部分层。第16章给了驱动程序寄存器访问词汇。第17章教它像设备一样思考。第18章将其介绍给真实PCI设备。第19章在单个IRQ上给了它耳朵。第20章给它一套耳朵，设备想进行的每次对话各一个。第21章教那些耳朵与设备自己访问RAM的能力协作。每章添加一层。每层依赖之前的层。第20章是驱动程序停止假装设备只有一件事要说并开始将其作为真正的多队列机器对待的地方。

### 为什么MSI-X值得单独一章

此时您可能在问为什么MSI和MSI-X需要自己的章节。第19章驱动程序有在遗留IRQ线上工作的中断处理程序。如果过滤器加任务管道已经正确，为什么不继续使用它？MSI-X真的需要整整一章新材料吗？

三个原因。

首先是规模。共享系统上的单IRQ线迫使该线上每个驱动程序通过一个`intr_event`串行化。在有数十PCIe设备的主机上，如果遗留INTx机制是唯一选项，它会瓶颈整个系统。MSI-X让每个设备（以及设备内每个队列）有自己的专用`intr_event`，由自己的ithread或过滤器处理程序服务，绑定到自己的CPU。现代服务器用MSI-X处理每秒一千万数据包与相同工作负载在遗留INTx上的差异是"可能"和"不可能"之间的差异；MSI-X是前者成为现实的原因。

其次是位置性。单中断线，内核对路由中断到的CPU只有一个选择，该选择对设备全局。用MSI-X，每个向量可以绑定到不同CPU，好的驱动程序将每个向量绑定到NUMA本地于其服务队列的CPU。这样做的缓存行优势真实：其中断在最终消费数据包的同一CPU上触发的接收队列避免在遗留设置上主导的跨插槽缓存流量。

第三是清洁。即使对于不需要高吞吐的驱动程序，MSI或MSI-X可以简化处理程序。用专用线，过滤器不需要处理共享IRQ情况。用每事件类专用向量（admin、接收、发送、错误），每个处理程序更小更专门化，整个驱动更易阅读。好的驱动即使性能不需要也使用MSI-X，因为代码变好。

第20章通过具体教授所有三个好处赢得位置。读者完成本章能分配向量、路由它们、描述它们、拆除它们，有演示模式端到端的工作驱动程序。

### 第19章为驱动程序留下的起点

您应该站在哪里简短回顾。第20章扩展第19章阶段4结束时产生的驱动程序，标记为版本`1.2-intr`。如果以下任何项目感觉不确定，请在开始本章之前返回第19章。

- 您的驱动程序编译干净，在`kldstat -v`中标识为`1.2-intr`。
- 在暴露virtio-rnd设备的bhyve或QEMU guest上，驱动程序attach、将BAR 0分配为`SYS_RES_MEMORY`、将遗留IRQ分配为带`rid = 0`的`SYS_RES_IRQ`、通过`bus_setup_intr(9)`用`INTR_TYPE_MISC | INTR_MPSAFE`注册过滤器处理程序、创建`/dev/myfirst0`、并支持`dev.myfirst.N.intr_simulate`sysctl。
- 过滤器读取`INTR_STATUS`、递增每位计数器、确认、为`DATA_AV`入队延迟任务、并返回正确`FILTER_*`值。
- 任务（`myfirst_intr_data_task_fn`）在名为`myfirst_intr`优先级`PI_NET`的taskqueue上的线程上下文运行、读取`DATA_OUT`、更新softc、并广播`sc->data_cv`。
- detach路径清除`INTR_MASK`、调用`bus_teardown_intr`、排空并释放taskqueue、释放IRQ资源、分离硬件层、并释放BAR。
- `HARDWARE.md`、`LOCKING.md`、`SIMULATION.md`、`PCI.md`和`INTERRUPTS.md`是当前的。
- `INVARIANTS`、`WITNESS`、`WITNESS_SKIPSPIN`、`DDB`、`KDB`和`KDB_UNATTENDED`在您的测试内核中启用。

那个驱动程序是第20章扩展的基础。新增在范围上可观：一个新文件（`myfirst_msix.c`）、一个新头文件（`myfirst_msix.h`）、几个新softc字段跟踪每向量状态、新每向量过滤器函数家族、setup helper中新回退阶梯、每向量`bus_describe_intr`调用、可选CPU绑定、版本提升到`1.3-msi`、新`MSIX.md`文档、以及回归测试更新。心智模型也增长：驱动程序开始将中断视为向量源而不是单个事件流。

### 您将学到什么

当您继续下一章时，您将能够：

- 描述MSI和MSI-X中断在硬件层面是什么、每个在PCIe上如何信号（作为内存写入而不是电平变化）、以及为何两种机制与传统INTx共存。
- 解释MSI和MSI-X关键区别：MSI向量计数（从连续块的1到32）、MSI-X向量计数（最多2048独立可寻址向量）、以及MSI-X提供MSI不提供的每向量地址和掩码能力。
- 通过`pci_msi_count(9)`和`pci_msix_count(9)`查询设备MSI和MSI-X能力，知道返回计数意指什么。
- 通过`pci_alloc_msi(9)`和`pci_alloc_msix(9)`分配MSI或MSI-X向量，处理内核分配少于请求向量的情况，并从分配失败恢复。
- 构建三层回退阶梯：MSI-X优先（如可用）、然后MSI（如MSI-X不可用或分配失败）、然后传统INTx。每层核心使用相同`bus_setup_intr`模式，但rid和每向量处理程序结构不同。
- 用正确rid分配每向量IRQ资源（遗留INTx用rid=0；MSI和MSI-X向量用rid=1, 2, 3, ...）。
- 每向量注册不同过滤器处理程序，使每个向量有自己的目的（admin、接收队列N、发送队列N、错误）。
- 设计每向量状态（每队列计数器、每队列任务、每队列锁）使在不同CPU上并发运行的处理器不争夺共享数据。
- 用`bus_describe_intr(9)`描述每个向量使`vmstat -i`和`devinfo -v`以有意义名称显示每个向量。
- 用`bus_bind_intr(9)`将每个向量绑定到特定CPU，并用`LOCAL_CPUS`或`INTR_CPUS`的`bus_get_cpus(9)`查询设备NUMA本地CPU集。
- 处理部分分配失败：设备有八个向量，内核给了我们三个；调整驱动使用三个并通过轮询或调度任务做剩余工作。
- 正确拆除多向量驱动：每向量`bus_teardown_intr`、每向量`bus_release_resource`、然后末尾单个`pci_release_msi(9)`调用。
- 在attach时记录单个清晰dmesg摘要行说明中断模式（MSI-X / N向量、MSI / K向量或传统INTx），使操作者瞬间看到驱动最终使用的层次。
- 将多向量代码拆分到`myfirst_msix.c`、更新模块`SRCS`行、将驱动标记为`1.3-msi`、并产生`MSIX.md`记录每向量目的和观察计数器模式。

列表很长；每个项目范围狭窄。本章意义在于组合。

### 本章不涵盖的内容

几个相邻话题明确推迟，以保持第20章专注。

- **DMA。**`bus_dma(9)`标签、`bus_dmamap_load(9)`、scatter-gather列表、bounce缓冲区、DMA描述符周围的缓存一致性、以及设备如何向RAM写入完成在第21章。第20章给驱动多个向量；第21章给设备移动数据的能力。每半独立有价值；一起是每个现代性能驱动程序的骨干。
- **iflib(9)和多队列网络框架。**iflib是厚的、有观点的框架，用每队列ithread、每队列DMA池和很多通用驱动不需要的机制包装MSI-X。第20章教授原始模式；第6部分网络章节（第28章）以iflib词汇重访。
- **通过MSI-X向量的PCIe AER恢复。**高级错误报告在某些设备上可以通过自己的MSI-X向量信号。第20章提及可能性；完整恢复路径是后续章节话题。
- **SR-IOV和每VF中断。**单根IO虚拟化虚拟功能有自己的MSI-X能力和自己的每VF向量。第20章驱动是物理功能；VF故事是后续章节特化。
- **每向量线程优先级调整。**驱动可以向每个向量的`bus_setup_intr`flags传递不同优先级，或以不同优先级每向量使用`taskqueue_start_threads`。第20章对每个向量使用`INTR_TYPE_MISC | INTR_MPSAFE`不调整优先级；第7部分性能章节（第33章）覆盖调整故事。
- **使用PCIe能力的现代virtio-PCI传输。**`virtio_pci_modern(4)`驱动将virtqueue通知放入能力结构并为virtqueue完成使用MSI-X向量。第20章驱动仍目标遗留virtio-rnd BAR；将适配到真实生产设备的读者会跟随第20章模式但从现代virtio PCI布局读取。

保持在这些界限内使第20章成为关于多向量中断处理的章节。词汇可以迁移；后续特定章节将词汇应用于DMA、iflib、AER和SR-IOV。

### 预估时间投入

- **仅阅读**：四到五小时。MSI/MSI-X概念模型不复杂，但每向量纪律、回退阶梯和CPU亲和故事需要仔细阅读。
- **阅读加输入工作示例**：两到三次会话共十到十二小时。驱动分四个阶段演进：回退阶梯、多向量、每向量处理程序、重构。每个阶段小但需要对每向量状态仔细注意。
- **阅读加所有实验和挑战**：四到五次会话共十六到二十小时，包括阅读真实驱动（`virtio_pci.c`、`if_em.c`MSI-X代码、`nvme.c`admin+IO向量分离）、设置暴露MSI-X的bhyve或QEMU guest、以及运行本章回归测试。

第3、5和6节最密集。如果每向量处理程序模式在第一次阅读时感觉陌生，这是正常的。停下来、重读第3节图示、在形状稳定后继续。

### 前提条件

在开始本章之前，确认：

- 您的驱动源码匹配第19章阶段4（`1.2-intr`）。起点假设每个第19章原语：过滤器加任务管道、模拟中断sysctl、`ICSR_*`访问器宏、干净拆除。
- 您的实验室机器运行FreeBSD 14.3，磁盘上有`/usr/src`并匹配运行内核。
- 已构建、安装并干净启动具有`INVARIANTS`、`WITNESS`、`WITNESS_SKIPSPIN`、`DDB`、`KDB`和`KDB_UNATTENDED`的调试内核。
- `bhyve(8)`或`qemu-system-x86_64`可用。对于MSI-X实验，guest必须暴露MSI-X能力启用的设备。QEMU的`virtio-rng-pci`有MSI-X；bhyve的`virtio-rnd`使用遗留virtio且不将MSI-X暴露给主机驱动作为默认配置事项。本章指出哪些实验需要哪种环境。
- `devinfo(8)`、`vmstat(8)`、`pciconf(8)`和`cpuset(1)`工具在您的路径中。

如果上述任何项目不稳固，现在修复。MSI-X倾向于暴露驱动中断上下文纪律的任何潜在弱点，因为多个处理程序可同时在多CPU上运行；调试内核的`WITNESS`在第20章开发期间特别有价值。

### 如何充分利用本章

四个习惯会很快带来回报。

首先，保持`/usr/src/sys/dev/pci/pcireg.h`和`/usr/src/sys/dev/pci/pcivar.h`与新文件`/usr/src/sys/dev/pci/pci.c`和`/usr/src/sys/dev/virtio/pci/virtio_pci.c`一起收藏。前两个来自第18章，定义能力常量（`PCIY_MSI`、`PCIY_MSIX`、`PCIM_MSIXCTRL_*`）和访问器包装器。第三个是内核的`pci_msi_count_method`、`pci_alloc_msi_method`、`pci_alloc_msix_method`和`pci_release_msi_method`实现。第四个是完整MSI-X分配阶梯及回退的干净真实驱动示例。每个文件值得半小时阅读。

其次，在实验主机和guest上运行`pciconf -lvc`。`-c`标志告诉`pciconf`打印每个设备的能力列表，你会看到哪些设备暴露MSI、MSI-X或两者。查看自己的机器是理解为何MSI-X在现代PCIe中是默认的最快方式。

再次，手动输入更改并运行每个阶段。MSI-X代码中微妙的每向量错误产生仅在并发负载下出现的bug。仔细输入、观察`dmesg`的attach横幅、并在每个阶段后运行回归测试，可在错误便宜修复时捕获它们。

第四，在第5节后阅读`/usr/src/sys/dev/nvme/nvme_ctrlr.c`的MSI-X设置（查找`nvme_ctrlr_allocate_bar`和`nvme_ctrlr_construct_admin_qpair`）。`nvme(4)`是第20章教授的管理加N队列模式的干净真实驱动示例。文件很长但MSI-X代码只是一小部分；其余阅读可选但有教育意义。

### 本章路线图

各节顺序如下：

1. **什么是MSI和MSI-X？** 硬件画面：消息信号中断在PCIe上如何工作、MSI与MSI-X区别、为何现代设备偏好它们。
2. **在驱动中启用MSI。** 两种模式中较简单的。查询计数、分配、注册处理程序。第20章驱动阶段1（`1.3-msi-stage1`）。
3. **管理多个中断向量。** 本章核心。每向量rid、每向量过滤器函数、每向量softc状态、每向量`bus_describe_intr`。阶段2（`1.3-msi-stage2`）。
4. **设计中断安全数据结构。** 为什么多向量意味多CPU、每个向量处理程序可和不可触摸什么锁、如何结构每队列状态。一种纪律，不是阶段提升。
5. **使用MSI-X获得高灵活性。** 更完整的机制。表布局、每向量绑定、用`bus_get_cpus`NUMA感知放置。阶段3（`1.3-msi-stage3`）。
6. **处理向量特定事件。** 每向量处理程序函数、每向量延迟工作、`nvme(4)`驱动大规模使用的模式。
7. **用MSI/MSI-X拆除和清理。** 每向量拆除、然后单个`pci_release_msi`调用。保持一切安全的顺序规则。
8. **重构和版本化多向量驱动。** 最终拆分到`myfirst_msix.c`、新`MSIX.md`、版本提升到`1.3-msi`、以及回归通过。阶段4。

八个节之后是动手实验、挑战练习、故障排除参考、收尾总结结束第20章故事并开启第21章、以及通往第21章的桥梁。本章末尾参考和速查材料旨在在阅读第21章时重读；第20章词汇（向量、rid、每向量softc、亲和、拆除顺序）是第21章DMA工作依赖的基础。

如果这是您的第一次阅读，请按顺序线性阅读并按顺序做实验。如果您是重温，第3和5节独立，适合单次阅读。

## 第1节：什么是MSI和MSI-X？

在驱动程序代码之前，先看硬件画面。第1节在PCIe总线和中断控制器层面教授消息信号中断是什么，不涉及任何FreeBSD特定词汇。理解第1节的读者可以将内核MSI/MSI-X路径作为具体对象而不是模糊抽象阅读本章其余部分。

### 遗留INTx的问题

第19章教授了传统PCI INTx中断模型：每个PCI功能有一条中断线（通常是INTA、INTB、INTC、INTD之一），线是电平触发，同一条物理线上的多个设备共享它。第19章驱动通过先读取INTR_STATUS并在无设置时返回`FILTER_STRAY`正确处理了共享情况。

INTx工作。但它有三个随系统规模增长重要的问题。

首先是**共享开销**。共享线上有十个设备要求每个驱动在每次中断被调用，仅读取自己的状态寄存器并发现中断不是它的。在大多数中断合法（线忙）的系统上，这是每个事件几个额外`bus_read_4`调用；在一个设备风暴的系统上，每个其他驱动过滤器不必要运行。每个事件CPU开销小但在每秒数百万事件间累积。

其次是**无每队列分离**。现代NIC有四、八、十六或六十四个接收队列和匹配数量的发送队列。每个队列想要自己的中断：当接收队列3有数据包时，只有接收队列3的处理程序应该运行，在靠近该队列使用内存的CPU上。用INTx设备只有一条线，所以要么驱动从一个处理程序轮询每个队列（昂贵慢）要么设备只支持一个队列（对十千兆NIC不可接受）。

第三是**无每事件类型CPU亲和**。共享线在一个CPU触发（中断控制器路由到的那个）。在设备附着到插槽0的NUMA系统上，在插槽1CPU触发中断比在插槽0CPU触发更糟：处理程序代码在插槽1运行但设备内存驻留在插槽0，每次寄存器读取跨越插槽间架构。用INTx驱动不能说"请在CPU3触发此中断"；内核选择，驱动对每事件类型无影响。

MSI和MSI-X修复全部三个。机制与INTx根本不同：代替在专用线上电信号，设备对特定地址执行内存写入，CPU的中断控制器将写入视为中断。这将中断数量与物理线数量解耦，让每个消息信号中断有自己的目的地址（因此自己的CPU），并完全消除共享线问题。

### MSI触发中断实际如何发生

物理上，MSI中断是PCIe架构上的写事务。设备对特定地址执行特定值的写入。内存控制器识别地址属于中断控制器MSI区域，将写入路由到APIC（或GIC，或平台中断控制器）。中断控制器解码地址确定哪个CPU应该接收中断，解码写入值确定CPU应该运行哪个向量（IDT或等效的哪个条目）。CPU然后像任何中断一样分发：保存状态、跳转向量处理程序、运行内核中断分发。

从驱动角度，流程几乎与遗留INTx相同：

1. 设备有事件。
2. 中断触发。
3. 内核调用驱动过滤器处理程序。
4. 过滤器读取状态、确认、处理或延迟、返回。
5. 如延迟，任务稍后运行。
6. 拆除在detach路径进行。

区别在步骤2机制和设置时分配模型。设备不在断言线；在写入内存。内核不需要预安排目的线；它有消息地址和消息值池。每个向量对应一对（地址、值）。设备在MSI能力结构中存储这些对，需要中断时使用它们执行写入。

### MSI：两者中较简单的

MSI（消息信号中断）是较老较简单的机制。在1999年PCI 2.2引入，MSI让设备请求1到32中断向量，作为连续2的幂块分配（1、2、4、8、16或32）。设备在配置空间有单个MSI能力结构，包含：

- 消息地址寄存器（写入目的地址，典型是APICMSI区域）。
- 消息数据寄存器（写入值，编码向量号）。
- 消息控制寄存器（启用位、功能掩码位、请求向量数等）。

设备想信号向量N（N是0到计数-1）时，它将消息数据寄存器基值OR上N写入消息地址。中断控制器解复用写入值分发正确向量。

MSI关键属性：

- **单能力块。** 设备有一个MSI能力，不是每向量一个。
- **连续向量。** 块是2的幂，作为单元分配。
- **有限计数。** 每功能最多32向量。
- **无每向量掩码。** 整块一起掩码或解除掩码（通过功能掩码位如支持）。
- **无每向量地址。** 所有向量共享单个消息地址寄存器；向量号在写入数据低位。

MSI是对遗留INTx的重大改进，但有局限：无每向量掩码和32向量上限。大多数想要多向量的驱动最终偏好MSI-X。

### MSI-X：更完整的机制

MSI-X在PCI 3.0（2004）引入并在PCIe扩展，移除MSI局限。设备有MSI-X能力结构加MSI-X**表**（每向量条目数组）和**待决位数组**（PBA）。能力结构指向设备BAR中的一个或多个，表和PBA驻留其中。

每个MSI-X表条目包含：

- 消息地址寄存器（每向量）。
- 消息数据寄存器（每向量）。
- 向量控制寄存器（每向量掩码位）。

设备想信号向量N时，在表中查找条目N、读取该条目的地址和数据、执行写入。中断控制器根据写入内容分发。

MSI-X关键属性：

- **每向量地址和数据。** 每个向量可通过编程不同地址路由到不同CPU。
- **每向量掩码。** 单个向量可禁用而不禁用整块。
- **每功能最多2048向量。** 有多队列的NVMe控制器在此快乐；有64接收队列加64发送队列加一些admin向量的NIC适合。
- **表在BAR中。** 表位置通过MSI-X能力寄存器可发现；`pci_msix_table_bar(9)`和`pci_msix_pba_bar(9)`返回哪个BAR持有每个。
- **更复杂设置。** 驱动必须分配表、编程每个条目、然后启用。

实践上，现代PCIe设备对任何多向量用例偏好MSI-X，为向后兼容或单向量简单设备保留MSI。内核内部处理大部分表编程；驱动工作是查询计数、分配、注册每向量处理程序。

### FreeBSD如何抽象区别

内核在小组访问函数后隐藏大部分MSI与MSI-X区别。从`/usr/src/sys/dev/pci/pcivar.h`：

- `pci_msi_count(dev)`返回设备广播的MSI向量计数（无MSI能力则为0）。
- `pci_msix_count(dev)`返回MSI-X向量计数（无MSI-X能力则为0）。
- `pci_alloc_msi(dev, &count)`和`pci_alloc_msix(dev, &count)`分配向量。`count`是输入输出：输入是期望计数，输出是实际分配计数。
- `pci_release_msi(dev)`释放MSI和MSI-X向量（内部处理任一情况）。

驱动不直接与MSI-X表交互；内核代驱动完成。驱动看到的是，成功分配后，设备通过`bus_alloc_resource_any(9)`配合`SYS_RES_IRQ`显得有额外IRQ资源可用，对分配向量使用`rid = 1, 2, 3, ...`。驱动然后对每个资源注册过滤器处理程序，方式与第19章对遗留线注册相同。

对称是刻意的。第19章在`rid = 0`处理遗留IRQ的相同`bus_setup_intr(9)`调用，在`rid = 1, 2, 3, ...`处理每个MSI或MSI-X向量。每个`INTR_MPSAFE`规则、每个`FILTER_*`返回值约定、每个共享IRQ纪律（对MSI，向量在角落情况技术上可共享`intr_event`）、以及第19章每个拆除顺序延续。

### 回退阶梯

健壮驱动按偏好顺序尝试机制，分配失败时回退到下一个。规范阶梯：

1. **MSI-X优先。** 如果`pci_msix_count(dev)`非零，尝试`pci_alloc_msix(dev, &count)`。如果成功，使用MSI-X。在近代PCIe设备上这是偏好路径。
2. **MSI其次。** 如果MSI-X不可用或分配失败，检查`pci_msi_count(dev)`。如果非零，尝试`pci_alloc_msi(dev, &count)`。如果成功，使用MSI。
3. **遗留INTx最后。** 如果MSI-X和MSI都不可用，回退到第19章遗留路径用`rid = 0`。

真实驱动实现此阶梯以便在可能着陆的每个系统上工作，从只支持MSI-X的新NVMe驱动到只支持INTx的遗留芯片组。第20章驱动同样做；第2节写MSI路径，第5节写MSI-X路径，第8节将它们组合为单个回退阶梯。

### 真实世界示例

使用MSI和MSI-X设备的简短介绍。

**现代NIC。** 典型的10或25 Gbps NIC暴露16到64个MSI-X向量：每个接收队列一个、每个发送队列一个，以及少量用于管理、错误和链路状态事件。Intel的`igc(4)`、`em(4)`、`ix(4)`和`ixl(4)`都遵循此模式；Broadcom的`bnxt(4)`、Mellanox的`mlx4(4)`和`mlx5(4)`以及Chelsio的`cxgbe(4)`也是如此。`iflib(9)`框架为许多驱动包装了MSI-X分配。

**NVMe存储控制器。** NVMe控制器有一个管理队列和最多65535个I/O队列。实践中，驱动为管理队列分配一个MSI-X向量，为每个I/O队列分配一个，最多`NCPU`个。FreeBSD的`nvme(4)`驱动正是这样做的；代码可读且值得研究。

**现代USB主控制器。** xHCI（USB 3）主控制器通常公开一个MSI-X向量用于命令完成事件环，高性能变体上还有多个每插槽事件环。`xhci(4)`驱动的设置路径展示了管理加事件模式。

**GPU。** 现代独立GPU有许多MSI-X向量：一个用于命令缓冲区、一个或多个用于显示、每个引擎一个、一个用于电源管理等。树外drm-kmod驱动大量使用MSI-X。

**VM中的Virtio设备。** 当FreeBSD客户机在bhyve、KVM或VMware下运行时，现代virtio-PCI传输使用MSI-X：一个向量用于配置更改事件，每个virtqueue一个。`virtio_pci_modern(4)`驱动实现了这一点。

这些驱动都遵循第20章教授的相同模式：查询、分配、注册每向量处理程序、绑定到CPU、描述。具体细节不同（多少向量、如何分配给事件、如何绑定到CPU），但结构是恒定的。

### 为什么是MSI-X而不是MSI

读者可能问：既然MSI-X严格比MSI更强大，为什么MSI仍存在？两个原因。

首先是向后兼容。早于PCI 3.0的设备和主板可能支持MSI但不支持MSI-X。想在旧硬件上工作的驱动需要MSI回退。生态系统大部分已前进，但旧设备长尾仍存在。

其次是简单性。用一两个向量的MSI比MSI-X设置更简单（无表编程、无BAR查询）。对于中断需求在MSI32向量上限内且不需要每向量掩码的设备，MSI是更轻选择。许多简单PCIe设备为此原因只暴露MSI。

第20章驱动的实际回答：总是先试MSI-X，如不可用回退到MSI，如两者都不可用回退到遗留INTx。过去十年写的每个真实FreeBSD驱动使用此阶梯。

### MSI-X 流程图

```text
  Device    Config space    MSI-X table (in BAR)     Interrupt controller     CPU
 --------   ------------   ---------------------    --------------------    -----
   |             |                 |                         |                |
   | event N    |                 |                         |                |
   | occurs     |                 |                         |                |
   |            |                 |                         |                |
   | read       |                 |                         |                |
   | entry N   -+---------------->|                         |                |
   | from table |   address_N,    |                         |                |
   |            |   data_N        |                         |                |
   |<-----------+-----------------|                         |                |
   |                              |                         |                |
   | memory-write to address_N                             |                |
   |-----------------------------+------------------------->|                |
   |                              |                         |                |
   |                              |                         | steer to CPU  |
   |                              |                         |-------------->|
   |                              |                         |               | filter_N
   |                              |                         |               | runs
   |                              |                         |               |
   |                              |                         | EOI           |
   |                              |                         |<--------------|
```

图中省略了MSI-X表读取（设备在发出写入之前在内部执行）和中断控制器的解复用逻辑，但它捕捉了机制的本质：设备的事件触发内存写入，内存写入变成中断，中断分发到过滤器。过滤器做第19章过滤器所做的同样工作。唯一区别是在MSI-X上，每个向量有不同的过滤器。

### 练习：在系统上查找支持 MSI 的设备

在进入第2节之前，一个简短练习让能力图景具体化。

在实验主机上运行：

```sh
sudo pciconf -lvc
```

`-c` 标志告诉 `pciconf(8)` 打印每个设备的能力列表。你会看到类似条目：

```text
vgapci0@pci0:0:2:0: ...
    ...
    cap 05[d0] = MSI supports 1 message, 64 bit
    cap 10[a0] = PCI-Express 2 endpoint max data 128(128)
em0@pci0:0:25:0: ...
    ...
    cap 01[c8] = powerspec 2  supports D0 D3  current D0
    cap 05[d0] = MSI supports 1 message, 64 bit
    cap 11[e0] = MSI-X supports 4 messages
```

每个 `cap 05` 是MSI能力。每个 `cap 11` 是MSI-X能力。等号后的描述告诉你该模式下设备支持多少消息（向量）。

从输出中选三个设备。对每个，记录：

- MSI计数（如有）。
- MSI-X计数（如有）。
- 驱动当前使用哪一个。（可从设备的 `vmstat -i` 条目推断：如果看到多个 `name:queueN` 行，驱动正在使用MSI-X。）

没有很多PCIe设备的主机可能只显示MSI能力；笔记本电脑通常MSI-X使用有限。有多个NIC和NVMe驱动的现代服务器显示许多具有高向量计数的MSI-X能力（某些NIC为64或更多）。

阅读第2节时保持此输出打开。"cap 11[XX] = MSI-X supports N messages"词汇是内核的 `pci_msix_count(9)` 返回给驱动的内容，也是分配阶梯在attach时查询的内容。

### 第1节收尾

MSI和MSI-X是遗留INTx的近代消息信号后继者。MSI提供最多32向量，作为连续块分配，单个目的地址；MSI-X提供最多2048向量，每向量地址、每向量数据和每向量掩码。两者在PCIe上作为中断控制器解码为向量分发的内存写入信号。

内核在`pci_msi_count(9)`、`pci_msix_count(9)`、`pci_alloc_msi(9)`、`pci_alloc_msix(9)`和`pci_release_msi(9)`后抽象区别。每个分配向量在`rid = 1, 2, 3, ...`成为IRQ资源，驱动通过`bus_setup_intr(9)`为每个注册过滤器处理程序，完全与第19章对`rid = 0`遗留IRQ所做的相同。

健壮驱动实现三层回退阶梯：MSI-X偏好、MSI回退、遗留INTx最后手段。第2节写此阶梯的MSI部分。第5节写MSI-X部分。第8节组装完整阶梯。
## 第2节：在驱动中启用MSI

第1节建立了硬件模型。第2节让驱动程序开始工作。任务范围窄：扩展第19章attach路径，使其在回退到`rid = 0`遗留IRQ前，驱动尝试分配MSI向量。如果分配成功，驱动使用MSI向量代替遗留线。如果分配失败（要么设备不支持MSI要么内核无法分配），驱动完全按第19章代码回退到遗留路径。

第2节的意义是在MSI-X多向量复杂性使画面更忙之前孤立介绍MSI API。单向量MSI路径本质与单向量遗留INTx路径相同；只有分配调用和rid改变。最小变化是好第一阶段。

### 阶段1产生什么

阶段1扩展第19章阶段4驱动为两层回退：MSI优先，遗留INTx回退。过滤器处理程序是相同第19章过滤器。taskqueue相同。sysctl相同。改变的是分配路径：`myfirst_intr_setup`先检查`pci_msi_count(9)`，如非零调用`pci_alloc_msi(9)`请求一个向量。如果成功，IRQ资源在`rid = 1`；如果失败，驱动穿透到`rid = 0`做遗留INTx。

驱动还在单个`dmesg`行中记录中断模式，使操作者一眼知道驱动最终使用的层次。这是小但重要的可观察性特性，每个真实FreeBSD驱动实现；第20章遵循约定。

### MSI计数查询

第一步是问设备广播多少MSI向量：

```c
int msi_count = pci_msi_count(sc->dev);
```

返回值如果设备无MSI能力则为0；否则是设备MSI能力控制寄存器广播的向量数。典型值是1、2、4、8、16或32（MSI要求2的幂计数最多32）。

返回0不意味设备无中断；意味设备不暴露MSI。驱动应穿透到下一层。

### MSI分配调用

第二步是请内核分配向量：

```c
int count = 1;
int error = pci_alloc_msi(sc->dev, &count);
```

`count`是输入输出参数。输入是驱动想要的向量数。输出是内核实际分配的数量。内核允许分配少于请求的；需要至少特定计数的驱动必须检查返回值。

对于第20章阶段1，驱动请求一个向量。如果内核返回1，驱动继续。如果内核返回0（罕见但在竞争系统上可能）或返回错误，驱动释放任何分配并回退到遗留INTx。

微妙点：即使`pci_alloc_msi`返回非零，驱动**必须**在拆除时调用`pci_release_msi(dev)`撤销分配。与`bus_alloc_resource_any`/`bus_release_resource`不同，MSI家族使用单个`pci_release_msi`调用撤销通过`pci_alloc_msi`或`pci_alloc_msix`分配的所有向量。

### 每向量资源分配

在设备级分配MSI向量后，驱动现在必须为每个向量分配`SYS_RES_IRQ`资源。对于单个MSI向量，rid是1：

```c
int rid = 1;  /* MSI向量从rid 1开始 */
struct resource *irq_res;

irq_res = bus_alloc_resource_any(sc->dev, SYS_RES_IRQ, &rid, RF_ACTIVE);
if (irq_res == NULL) {
    /* 释放MSI分配并回退。 */
    pci_release_msi(sc->dev);
    goto fallback;
}
```

注意与第19章遗留分配两个区别：

首先，**rid是1不是0**。MSI向量从1开始编号，留下rid 0给遗留INTx。如果驱动同时使用两者（不应该），rid不重叠。

其次，**RF_SHAREABLE未设置**。MSI向量是每功能的；不与其他驱动共享。`RF_SHAREABLE`标志仅与遗留INTx相关。在MSI资源分配上设置它无害但无意义。

### MSI向量上的过滤器处理程序

过滤器处理程序函数与第19章相同：

```c
int myfirst_intr_filter(void *arg);
```

向量触发时内核调用过滤器，完全如遗留线断言时调用第19章过滤器。过滤器读取`INTR_STATUS`、确认、为`DATA_AV`入队任务、返回`FILTER_HANDLED`（或零位时`FILTER_STRAY`）。过滤器体无需改变。

`bus_setup_intr(9)`调用相同：

```c
error = bus_setup_intr(sc->dev, irq_res,
    INTR_TYPE_MISC | INTR_MPSAFE,
    myfirst_intr_filter, NULL, sc,
    &sc->intr_cookie);
```

函数签名、flags、参数（`sc`）和输出cookie都是第19章模式。

一个小改进：`bus_describe_intr(9)`现在可用模式特定名称标记向量：

```c
bus_describe_intr(sc->dev, irq_res, sc->intr_cookie, "msi");
```

此后，`vmstat -i`显示处理程序为`irq<N>: myfirst0:msi`（对某个内核选择的N）。操作者瞬间看到驱动正在使用MSI。

### 构建回退

放在一起，阶段1`myfirst_intr_setup`变为两层回退阶梯：先试MSI，回退到遗留INTx。代码：

```c
int
myfirst_intr_setup(struct myfirst_softc *sc)
{
    int error, msi_count, count;

    TASK_INIT(&sc->intr_data_task, 0, myfirst_intr_data_task_fn, sc);
    sc->intr_tq = taskqueue_create("myfirst_intr", M_WAITOK,
        taskqueue_thread_enqueue, &sc->intr_tq);
    taskqueue_start_threads(&sc->intr_tq, 1, PI_NET,
        "myfirst intr taskq");

    /*
     * 第一层：尝试MSI。
     */
    msi_count = pci_msi_count(sc->dev);
    if (msi_count > 0) {
        count = 1;
        if (pci_alloc_msi(sc->dev, &count) == 0 && count == 1) {
            sc->irq_rid = 1;
            sc->irq_res = bus_alloc_resource_any(sc->dev,
                SYS_RES_IRQ, &sc->irq_rid, RF_ACTIVE);
            if (sc->irq_res != NULL) {
                error = bus_setup_intr(sc->dev, sc->irq_res,
                    INTR_TYPE_MISC | INTR_MPSAFE,
                    myfirst_intr_filter, NULL, sc,
                    &sc->intr_cookie);
                if (error == 0) {
                    bus_describe_intr(sc->dev,
                        sc->irq_res, sc->intr_cookie,
                        "msi");
                    sc->intr_mode = MYFIRST_INTR_MSI;
                    device_printf(sc->dev,
                        "interrupt mode: MSI, 1 vector\n");
                    goto enabled;
                }
                bus_release_resource(sc->dev,
                    SYS_RES_IRQ, sc->irq_rid, sc->irq_res);
                sc->irq_res = NULL;
            }
            pci_release_msi(sc->dev);
        }
    }

    /*
     * 第二层：回退到遗留INTx。
     */
    sc->irq_rid = 0;
    sc->irq_res = bus_alloc_resource_any(sc->dev, SYS_RES_IRQ,
        &sc->irq_rid, RF_SHAREABLE | RF_ACTIVE);
    if (sc->irq_res == NULL) {
        device_printf(sc->dev, "cannot allocate legacy IRQ\n");
        taskqueue_free(sc->intr_tq);
        sc->intr_tq = NULL;
        return (ENXIO);
    }
    error = bus_setup_intr(sc->dev, sc->irq_res,
        INTR_TYPE_MISC | INTR_MPSAFE,
        myfirst_intr_filter, NULL, sc,
        &sc->intr_cookie);
    if (error != 0) {
        bus_release_resource(sc->dev, SYS_RES_IRQ,
            sc->irq_rid, sc->irq_res);
        sc->irq_res = NULL;
        taskqueue_free(sc->intr_tq);
        sc->intr_tq = NULL;
        return (error);
    }
    bus_describe_intr(sc->dev, sc->irq_res, sc->intr_cookie, "legacy");
    sc->intr_mode = MYFIRST_INTR_LEGACY;
    device_printf(sc->dev,
        "interrupt mode: legacy INTx (rid=0)\n");

enabled:
    /* 在设备端启用中断。 */
    MYFIRST_LOCK(sc);
    if (sc->hw != NULL)
        CSR_WRITE_4(sc, MYFIRST_REG_INTR_MASK,
            MYFIRST_INTR_DATA_AV | MYFIRST_INTR_ERROR |
            MYFIRST_INTR_COMPLETE);
    MYFIRST_UNLOCK(sc);

    return (0);
}
```

代码有三个明显块：

1. MSI尝试块（`if (msi_count > 0)`防护内的行）。
2. 遗留回退块。
3. 不管哪层都运行的`enabled:`启用块。

MSI尝试做完整序列：计数查询、分配、在rid 1分配IRQ资源、注册处理程序。如果任何步骤失败，代码释放成功的内容（如已分配的资源、如已成功分配的MSI）并穿透。

遗留回退本质是第19章设置，不变。

`enabled:`块在设备写入`INTR_MASK`。无论得到MSI还是遗留，设备端掩码相同。

回退结构是真实驱动所做的。阅读`virtio_pci.c`设置代码的读者会看到更大规模的相同模式：几次带连续回退的尝试。

### intr_mode字段和dmesg摘要

softc获得新字段：

```c
enum myfirst_intr_mode {
    MYFIRST_INTR_LEGACY = 0,
    MYFIRST_INTR_MSI = 1,
    MYFIRST_INTR_MSIX = 2,
};

struct myfirst_softc {
    /* ... 现有字段 ... */
    enum myfirst_intr_mode intr_mode;
};
```

字段记录驱动最终使用的层次。attach时`device_printf`打印它：

```text
myfirst0: interrupt mode: MSI, 1 vector
```

或：

```text
myfirst0: interrupt mode: legacy INTx (rid=0)
```

阅读`dmesg`的操作者看到此行并知道哪条路径活跃。调试驱动的读者也看到它；如果驱动在读者期望MSI时回退到遗留，此行立即标记问题。

`intr_mode`字段也通过只读sysctl暴露以便用户空间工具读取：

```c
SYSCTL_ADD_INT(&sc->sysctl_ctx,
    SYSCTL_CHILDREN(sc->sysctl_tree), OID_AUTO, "intr_mode",
    CTLFLAG_RD, &sc->intr_mode, 0,
    "Interrupt mode: 0=legacy, 1=MSI, 2=MSI-X");
```

想知道是否有`myfirst`实例使用MSI-X的脚本可以汇总所有单元的`intr_mode`值。

### 拆除需要改变什么

第19章的拆除路径调用`bus_teardown_intr`、排空并释放taskqueue、释放IRQ资源。对于阶段1，需要一个额外调用：如果驱动使用了MSI，必须在释放IRQ资源后调用`pci_release_msi`：

```c
void
myfirst_intr_teardown(struct myfirst_softc *sc)
{
    /* 在设备端禁用。 */
    MYFIRST_LOCK(sc);
    if (sc->hw != NULL && sc->bar_res != NULL)
        CSR_WRITE_4(sc, MYFIRST_REG_INTR_MASK, 0);
    MYFIRST_UNLOCK(sc);

    /* 拆除处理程序。 */
    if (sc->intr_cookie != NULL) {
        bus_teardown_intr(sc->dev, sc->irq_res, sc->intr_cookie);
        sc->intr_cookie = NULL;
    }

    /* 排空并销毁taskqueue。 */
    if (sc->intr_tq != NULL) {
        taskqueue_drain(sc->intr_tq, &sc->intr_data_task);
        taskqueue_free(sc->intr_tq);
        sc->intr_tq = NULL;
    }

    /* 释放IRQ资源。 */
    if (sc->irq_res != NULL) {
        bus_release_resource(sc->dev, SYS_RES_IRQ, sc->irq_rid,
            sc->irq_res);
        sc->irq_res = NULL;
    }

    /* 如使用了MSI则释放。 */
    if (sc->intr_mode == MYFIRST_INTR_MSI ||
        sc->intr_mode == MYFIRST_INTR_MSIX)
        pci_release_msi(sc->dev);

    sc->intr_mode = MYFIRST_INTR_LEGACY;
}
```

`pci_release_msi`调用是有条件的：仅在驱动实际分配了MSI或MSI-X时调用。在驱动只使用遗留INTx时调用在现代FreeBSD中是空操作，但条件更清晰。

注意顺序：先IRQ资源释放，然后`pci_release_msi`。这与分配顺序相反（分配时`pci_alloc_msi`在`bus_alloc_resource_any`之前）。规则是第18和19章的通用拆除规则：以设置逆序撤销。

### 验证阶段1

在设备支持MSI的guest上（QEMU的`virtio-rng-pci`支持；bhyve的`virtio-rnd`不支持），阶段1驱动应以MSI attach：

```text
myfirst0: <Red Hat Virtio entropy source (myfirst demo target)> ... on pci0
myfirst0: BAR0 allocated: 0x20 bytes at 0xfebf1000
myfirst0: hardware layer attached to BAR: 32 bytes
myfirst0: interrupt mode: MSI, 1 vector
```

在设备只支持遗留的guest上（bhyve的`virtio-rnd`典型地）：

```text
myfirst0: BAR0 allocated: 0x20 bytes at 0xc1000000
myfirst0: hardware layer attached to BAR: 32 bytes
myfirst0: interrupt mode: legacy INTx (rid=0)
```

两种情况都正确。驱动在任一模式运行；行为（过滤器、任务、计数器、模拟中断sysctl）完全相同。

`sysctl dev.myfirst.0.intr_mode`返回0（遗留）、1（MSI）或2（MSI-X，第5节添加后）。回归脚本用此验证期望模式。

### 阶段 1 不做什么

阶段1添加了一个向量的MSI，但尚未利用MSI的多向量潜力。单个MSI向量在功能上几乎与单个遗留IRQ相同（仅在有许多设备的系统上获得可扩展性优势，单设备实验室很少显示）。阶段1的价值在于引入回退阶梯习语并建立`intr_mode`可观察性；阶段2及以后使用这些基础添加多向量处理。

### 此阶段常见错误

简短列表。

**对MSI使用rid = 0。** MSI向量rid是1不是0。在已分配MSI向量的设备上请求`rid = 0`返回遗留INTx资源，不是MSI向量。驱动最终在错误线上有处理程序。修复：第一个MSI或MSI-X向量用`rid = 1`。

**拆除时忘记`pci_release_msi`。** 内核MSI分配状态在`bus_release_resource`后仍存活于IRQ资源。无`pci_release_msi`，下次attach尝试会失败因为内核仍认为驱动拥有MSI向量。修复：使用MSI或MSI-X时总在拆除中调用`pci_release_msi`。

**忘记INTx回退。** 只尝试MSI并在失败时返回错误的驱动在支持MSI的系统上工作但在旧系统上失败。修复：总是提供遗留INTx回退。

**忘记在拆除时恢复sc->intr_mode。** `intr_mode`字段记录层次。不重置，未来重新attach可能读到过期值。不是严重bug（attach总是设置它），但清洁重要。修复：在拆除中重置为`LEGACY`（或中性值）。

**计数不匹配。** `pci_alloc_msi`可分配少于请求的向量；如果驱动在count为0时假设`count == 1`，代码解引用未分配资源。修复：总是检查返回计数。

**不释放就两次调用`pci_alloc_msi`。** 每设备同时只能有一个MSI（或MSI-X）分配活跃。不释放第一个就尝试第二次分配返回错误。修复：如果驱动想改变分配（比如从MSI到MSI-X），先调用`pci_release_msi`。

### 检查点：阶段 1 正常工作

在第3节之前，确认阶段1已就位：

- `kldstat -v | grep myfirst` 显示版本 `1.3-msi-stage1`。
- `dmesg | grep myfirst` 显示attach横幅及 `interrupt mode:` 行，指示MSI或遗留。
- `sysctl dev.myfirst.0.intr_mode` 返回0或1。
- `vmstat -i | grep myfirst` 显示处理程序，描述符为 `myfirst0:msi` 或 `myfirst0:legacy`。
- `sudo sysctl dev.myfirst.0.intr_simulate=1` 仍驱动第19章流水线。
- `kldunload myfirst` 干净运行；无泄漏。

如果MSI路径在你的guest上失败，尝试QEMU而非bhyve。如果MSI路径在其中一个工作而另一个不工作，通过 `pciconf -lvc` 验证设备的MSI能力是否暴露。

### 第2节收尾

在驱动中启用MSI是三个新调用（`pci_msi_count`、`pci_alloc_msi`、`pci_release_msi`）、一个IRQ资源分配改变（rid = 1代替0）、和一个新softc字段（`intr_mode`）。回退阶梯添加第二层：尝试MSI，回退到遗留。每个`bus_setup_intr`、每个过滤器、每个taskqueue任务和第19章每个拆除步骤延续不变。

阶段1处理单个MSI向量。第3节移至多向量：几个不同过滤器函数、几个每向量softc状态、以及现代驱动广泛使用的每队列处理程序模式开始。

## 第3节：管理多个中断向量

阶段1添加了一个向量的MSI。第3节扩展驱动处理多个向量，每个有自己的角色。动机示例是有多件事要说的设备：有接收队列和发送队列的NIC、有admin和I/O队列的NVMe控制器、有接收就绪和发送空事件的UART。

第20章驱动没有真实多队列设备；virtio-rnd目标每次中断最多单个事件类。为教学目的，我们以与第19章模拟中断相同的方式模拟多向量行为：sysctl接口让读者对特定向量触发模拟中断，驱动的过滤器和任务机制演示真实驱动如何处理多向量情况。

到第3节结束时驱动处于版本`1.3-msi-stage2`，有三个MSI-X向量：admin向量、"rx"向量和"tx"向量。每个向量有自己的过滤器函数、自己的延迟任务和自己的计数器。过滤器读取`INTR_STATUS`仅确认与其向量相关的位；任务做向量特定工作。

关于三向量计数重要说明。MSI受限于2的幂向量计数（1、2、4、8、16或32），所以请求恰好3向量被`pci_alloc_msi(9)`以`EINVAL`拒绝（见`/usr/src/sys/dev/pci/pci.c`的`pci_alloc_msi_method`）。MSI-X无此限制且容易分配3向量。回退阶梯的MSI层因此请求单个MSI向量并回退到第19章单处理程序模式；仅MSI-X层给驱动三个每向量过滤器。第5节明确说明，第8节重构保持MSI层简单。

### 每向量设计

设计包含三个向量：

- **管理向量（vector 0, rid 1）。** 处理 `ERROR` 和配置变更事件。低频率；很少运行。
- **RX 向量（vector 1, rid 2）。** 处理 `DATA_AV` 事件（接收就绪）。以数据路径速率运行。
- **TX 向量（vector 2, rid 3）。** 处理 `COMPLETE` 事件（发送完成）。以数据路径速率运行。

每个向量拥有：

- 一个独立的 `struct resource *`（该向量的 IRQ 资源）。
- 一个独立的 `void *intr_cookie`（内核对处理程序的不透明句柄）。
- 一个独立的过滤函数（`myfirst_admin_filter`、`myfirst_rx_filter`、`myfirst_tx_filter`）。
- 一组独立的计数器（这样每个 CPU 上并发运行的过滤器不会争用单个共享计数器）。
- 一个独立的 `bus_describe_intr` 名称（`admin`、`rx`、`tx`）。
- 一个独立的延迟任务（仅 RX 需要；管理向量和 TX 内联处理其工作）。

每向量状态存储在 softc 内部的每向量结构数组中：

```c
#define MYFIRST_MAX_VECTORS 3

enum myfirst_vector_id {
	MYFIRST_VECTOR_ADMIN = 0,
	MYFIRST_VECTOR_RX,
	MYFIRST_VECTOR_TX,
};

struct myfirst_vector {
	struct resource		*irq_res;
	int			 irq_rid;
	void			*intr_cookie;
	enum myfirst_vector_id	 id;
	struct myfirst_softc	*sc;
	uint64_t		 fire_count;
	uint64_t		 stray_count;
	const char		*name;
	driver_filter_t		*filter;
	struct task		 task;
	bool			 has_task;
};

struct myfirst_softc {
	/* ... existing fields ... */
	struct myfirst_vector	vectors[MYFIRST_MAX_VECTORS];
	int			num_vectors;   /* actually allocated */
};
```

几个值得解释的设计要点。

**每向量 `struct myfirst_softc *sc` 反向指针。** 每个过滤器通过 `bus_setup_intr` 接收的参数是每向量结构（`struct myfirst_vector *`），而不是全局 softc。每向量结构包含一个指向 softc 的反向指针，以便过滤器在需要时可以访问共享状态。这是 `nvme(4)` 用于每队列向量的模式，也是每个多队列驱动程序遵循的模式。

**每向量计数器。** 每个向量有自己的 `fire_count` 和 `stray_count`。在两个 CPU 上运行的两个过滤器可以各自递增自己的计数器而不会产生原子争用；原子操作仍然使用，但每个原子操作访问不同的缓存行。

**每向量过滤指针。** `filter` 字段存储指向向量过滤函数的指针。这不是严格必需的（我们可以在单个通用过滤器中使用 switch），但它使每向量特化变得明确：每个向量的过滤器是静态已知的。

**每向量任务。** 并非每个向量都需要任务。管理向量和 TX 内联处理其工作（递增计数器、更新标志、可能唤醒等待者）。RX 延迟到任务，因为它需要广播条件变量，这需要线程上下文。`has_task` 标志使每向量差异变得明确。

### 过滤器函数

三个不同的过滤函数，每个向量一个：

```c
int
myfirst_admin_filter(void *arg)
{
	struct myfirst_vector *vec = arg;
	struct myfirst_softc *sc = vec->sc;
	uint32_t status;

	status = ICSR_READ_4(sc, MYFIRST_REG_INTR_STATUS);
	if ((status & (MYFIRST_INTR_ERROR)) == 0) {
		atomic_add_64(&vec->stray_count, 1);
		return (FILTER_STRAY);
	}

	atomic_add_64(&vec->fire_count, 1);
	ICSR_WRITE_4(sc, MYFIRST_REG_INTR_STATUS, MYFIRST_INTR_ERROR);
	atomic_add_64(&sc->intr_error_count, 1);
	return (FILTER_HANDLED);
}

int
myfirst_rx_filter(void *arg)
{
	struct myfirst_vector *vec = arg;
	struct myfirst_softc *sc = vec->sc;
	uint32_t status;

	status = ICSR_READ_4(sc, MYFIRST_REG_INTR_STATUS);
	if ((status & MYFIRST_INTR_DATA_AV) == 0) {
		atomic_add_64(&vec->stray_count, 1);
		return (FILTER_STRAY);
	}

	atomic_add_64(&vec->fire_count, 1);
	ICSR_WRITE_4(sc, MYFIRST_REG_INTR_STATUS, MYFIRST_INTR_DATA_AV);
	atomic_add_64(&sc->intr_data_av_count, 1);
	if (sc->intr_tq != NULL)
		taskqueue_enqueue(sc->intr_tq, &vec->task);
	return (FILTER_HANDLED);
}

int
myfirst_tx_filter(void *arg)
{
	struct myfirst_vector *vec = arg;
	struct myfirst_softc *sc = vec->sc;
	uint32_t status;

	status = ICSR_READ_4(sc, MYFIRST_REG_INTR_STATUS);
	if ((status & MYFIRST_INTR_COMPLETE) == 0) {
		atomic_add_64(&vec->stray_count, 1);
		return (FILTER_STRAY);
	}

	atomic_add_64(&vec->fire_count, 1);
	ICSR_WRITE_4(sc, MYFIRST_REG_INTR_STATUS, MYFIRST_INTR_COMPLETE);
	atomic_add_64(&sc->intr_complete_count, 1);
	return (FILTER_HANDLED);
}
```

每个过滤器都有相同的形式：读取状态、检查该向量关注的位、确认、更新计数器、可选地入队任务、返回。区别在于每个过滤器关注哪个位以及递增哪些计数器。

几个值得注意的细节。

**杂散检查是每向量的。** 每个过滤器检查自己的位，而不是任何位。如果过滤器被调用处理一个它不处理的事件（因为设置的位属于不同的向量），过滤器返回 `FILTER_STRAY`。这对 MSI-X 意义较小（每个向量有自己的专用消息，所以设备永远不会触发"错误"的向量），但对共享单个能力的多个向量的 MSI 意义更大。

**计数器共享。** 每向量计数器（`vec->fire_count`、`vec->stray_count`）是该向量特有的。全局计数器（`sc->intr_data_av_count` 等）是共享的，仍然用于本章的每位可观察性。同时拥有两者让读者可以交叉检查：RX 过滤器的触发计数应该近似等于全局的 `data_av_count`。

**过滤器不会睡眠。** 第 19 章所有过滤器上下文规则都延续下来：没有睡眠锁、没有 `malloc(M_WAITOK)`、没有阻塞。过滤器只使用原子操作和直接 BAR 访问。

### 每向量任务

只有 RX 有任务；管理向量和 TX 在过滤器中处理其工作。RX 任务本质上就是第 19 章的任务：

```c
static void
myfirst_rx_task_fn(void *arg, int npending)
{
	struct myfirst_vector *vec = arg;
	struct myfirst_softc *sc = vec->sc;

	MYFIRST_LOCK(sc);
	if (sc->hw != NULL && sc->pci_attached) {
		sc->intr_last_data = CSR_READ_4(sc, MYFIRST_REG_DATA_OUT);
		sc->intr_task_invocations++;
		cv_broadcast(&sc->data_cv);
	}
	MYFIRST_UNLOCK(sc);
}
```

任务在共享的 `intr_tq` 任务队列上的线程上下文中运行（第 19 章以 `PI_NET` 优先级创建它）。同一个任务队列服务所有每向量任务；对于具有真正独立的每队列工作的驱动程序，每个向量可能有自己的任务队列，但第 20 章使用一个。

### 分配多个向量

阶段 2 的设置代码比阶段 1 更长，因为它处理多个向量：

```c
int
myfirst_intr_setup(struct myfirst_softc *sc)
{
	int error, wanted, allocated, i;

	TASK_INIT(&sc->vectors[MYFIRST_VECTOR_RX].task, 0,
	    myfirst_rx_task_fn, &sc->vectors[MYFIRST_VECTOR_RX]);
	sc->vectors[MYFIRST_VECTOR_RX].has_task = true;
	sc->vectors[MYFIRST_VECTOR_ADMIN].filter = myfirst_admin_filter;
	sc->vectors[MYFIRST_VECTOR_RX].filter = myfirst_rx_filter;
	sc->vectors[MYFIRST_VECTOR_TX].filter = myfirst_tx_filter;
	sc->vectors[MYFIRST_VECTOR_ADMIN].name = "admin";
	sc->vectors[MYFIRST_VECTOR_RX].name = "rx";
	sc->vectors[MYFIRST_VECTOR_TX].name = "tx";
	for (i = 0; i < MYFIRST_MAX_VECTORS; i++) {
		sc->vectors[i].id = i;
		sc->vectors[i].sc = sc;
	}

	sc->intr_tq = taskqueue_create("myfirst_intr", M_WAITOK,
	    taskqueue_thread_enqueue, &sc->intr_tq);
	taskqueue_start_threads(&sc->intr_tq, 1, PI_NET,
	    "myfirst intr taskq");

	/*
	 * Try to allocate a single MSI vector. MSI requires a power-of-two
	 * count (PCI specification and /usr/src/sys/dev/pci/pci.c's
	 * pci_alloc_msi_method enforces this), so we cannot request the
	 * MYFIRST_MAX_VECTORS = 3 we want; we ask for 1 and fall back to
	 * the Chapter 19 single-handler pattern at rid=1, the same way
	 * sys/dev/virtio/pci/virtio_pci.c's vtpci_alloc_msi() does.
	 *
	 * MSI-X, covered in Section 5, is the tier where we actually
	 * obtain three distinct vectors; MSI-X is not constrained to
	 * power-of-two counts.
	 */
	allocated = 1;
	if (pci_msi_count(sc->dev) >= 1 &&
	    pci_alloc_msi(sc->dev, &allocated) == 0 && allocated >= 1) {
		sc->vectors[MYFIRST_VECTOR_ADMIN].filter = myfirst_intr_filter;
		sc->vectors[MYFIRST_VECTOR_ADMIN].name = "msi";
		error = myfirst_intr_setup_vector(sc, MYFIRST_VECTOR_ADMIN, 1);
		if (error == 0) {
			sc->intr_mode = MYFIRST_INTR_MSI;
			sc->num_vectors = 1;
			device_printf(sc->dev,
			    "interrupt mode: MSI, 1 vector "
			    "(single-handler fallback)\n");
			goto enabled;
		}
		pci_release_msi(sc->dev);
	}

	/*
	 * MSI allocation failed or was unavailable. Fall back to legacy
	 * INTx with a single vector-0 handler that handles every event
	 * class in one place.
	 */

fallback_legacy:
	sc->vectors[MYFIRST_VECTOR_ADMIN].irq_rid = 0;
	sc->vectors[MYFIRST_VECTOR_ADMIN].irq_res = bus_alloc_resource_any(
	    sc->dev, SYS_RES_IRQ,
	    &sc->vectors[MYFIRST_VECTOR_ADMIN].irq_rid,
	    RF_SHAREABLE | RF_ACTIVE);
	if (sc->vectors[MYFIRST_VECTOR_ADMIN].irq_res == NULL) {
		device_printf(sc->dev, "cannot allocate legacy IRQ\n");
		taskqueue_free(sc->intr_tq);
		sc->intr_tq = NULL;
		return (ENXIO);
	}
	error = bus_setup_intr(sc->dev,
	    sc->vectors[MYFIRST_VECTOR_ADMIN].irq_res,
	    INTR_TYPE_MISC | INTR_MPSAFE,
	    myfirst_intr_filter, NULL, sc,
	    &sc->vectors[MYFIRST_VECTOR_ADMIN].intr_cookie);
	if (error != 0) {
		bus_release_resource(sc->dev, SYS_RES_IRQ,
		    sc->vectors[MYFIRST_VECTOR_ADMIN].irq_rid,
		    sc->vectors[MYFIRST_VECTOR_ADMIN].irq_res);
		sc->vectors[MYFIRST_VECTOR_ADMIN].irq_res = NULL;
		taskqueue_free(sc->intr_tq);
		sc->intr_tq = NULL;
		return (error);
	}
	bus_describe_intr(sc->dev,
	    sc->vectors[MYFIRST_VECTOR_ADMIN].irq_res,
	    sc->vectors[MYFIRST_VECTOR_ADMIN].intr_cookie, "legacy");
	sc->intr_mode = MYFIRST_INTR_LEGACY;
	sc->num_vectors = 1;
	device_printf(sc->dev,
	    "interrupt mode: legacy INTx (1 handler for all events)\n");

enabled:
	/* Enable interrupts at the device. */
	MYFIRST_LOCK(sc);
	if (sc->hw != NULL)
		CSR_WRITE_4(sc, MYFIRST_REG_INTR_MASK,
		    MYFIRST_INTR_DATA_AV | MYFIRST_INTR_ERROR |
		    MYFIRST_INTR_COMPLETE);
	MYFIRST_UNLOCK(sc);

	return (0);
}
```

代码有三个阶段：MSI 尝试、MSI 回退清理和传统回退。MSI 尝试遍历向量，调用辅助函数（`myfirst_intr_setup_vector`）来分配和注册每个向量。在任何向量失败时，代码按相反顺序展开并穿透到传统模式。

辅助函数：

```c
static int
myfirst_intr_setup_vector(struct myfirst_softc *sc, int idx, int rid)
{
	struct myfirst_vector *vec = &sc->vectors[idx];
	int error;

	vec->irq_rid = rid;
	vec->irq_res = bus_alloc_resource_any(sc->dev, SYS_RES_IRQ,
	    &vec->irq_rid, RF_ACTIVE);
	if (vec->irq_res == NULL)
		return (ENXIO);

	error = bus_setup_intr(sc->dev, vec->irq_res,
	    INTR_TYPE_MISC | INTR_MPSAFE,
	    vec->filter, NULL, vec, &vec->intr_cookie);
	if (error != 0) {
		bus_release_resource(sc->dev, SYS_RES_IRQ, vec->irq_rid,
		    vec->irq_res);
		vec->irq_res = NULL;
		return (error);
	}

	bus_describe_intr(sc->dev, vec->irq_res, vec->intr_cookie,
	    "%s", vec->name);
	return (0);
}
```

辅助函数小而对称：分配资源、设置处理程序、描述它。传递给 `bus_setup_intr` 的参数是每向量结构（`vec`），而不是 softc。过滤器接收 `vec` 作为其 `void *arg`，并在需要 softc 时使用 `vec->sc`。

每向量拆除辅助函数：

```c
static void
myfirst_intr_teardown_vector(struct myfirst_softc *sc, int idx)
{
	struct myfirst_vector *vec = &sc->vectors[idx];

	if (vec->intr_cookie != NULL) {
		bus_teardown_intr(sc->dev, vec->irq_res, vec->intr_cookie);
		vec->intr_cookie = NULL;
	}
	if (vec->irq_res != NULL) {
		bus_release_resource(sc->dev, SYS_RES_IRQ, vec->irq_rid,
		    vec->irq_res);
		vec->irq_res = NULL;
	}
}
```

拆除是设置的逆过程：拆除处理程序、释放资源。

### 完整拆除路径

多向量拆除为每个活动向量调用每向量辅助函数，然后一次性释放 MSI 分配：

```c
void
myfirst_intr_teardown(struct myfirst_softc *sc)
{
	int i;

	MYFIRST_LOCK(sc);
	if (sc->hw != NULL && sc->bar_res != NULL)
		CSR_WRITE_4(sc, MYFIRST_REG_INTR_MASK, 0);
	MYFIRST_UNLOCK(sc);

	/* Per-vector teardown. */
	for (i = 0; i < sc->num_vectors; i++)
		myfirst_intr_teardown_vector(sc, i);

	/* Drain tasks. */
	if (sc->intr_tq != NULL) {
		for (i = 0; i < sc->num_vectors; i++) {
			if (sc->vectors[i].has_task)
				taskqueue_drain(sc->intr_tq,
				    &sc->vectors[i].task);
		}
		taskqueue_free(sc->intr_tq);
		sc->intr_tq = NULL;
	}

	/* Release MSI if used. */
	if (sc->intr_mode == MYFIRST_INTR_MSI ||
	    sc->intr_mode == MYFIRST_INTR_MSIX)
		pci_release_msi(sc->dev);

	sc->num_vectors = 0;
	sc->intr_mode = MYFIRST_INTR_LEGACY;
}
```

顺序是现在熟悉的：在设备处掩码、拆除处理程序、排空任务、释放 MSI。每向量循环做每向量工作。

### 按向量模拟中断

第 19 章的模拟中断 sysctl 每次触发一个处理程序。阶段 2 扩展了这个概念：每个向量一个 sysctl，或一个带有向量索引字段的单个 sysctl。本章的代码采用更简单的每向量单个 sysctl 形式：

```c
SYSCTL_ADD_PROC(ctx, kids, OID_AUTO, "intr_simulate_admin",
    CTLTYPE_UINT | CTLFLAG_WR | CTLFLAG_MPSAFE,
    &sc->vectors[MYFIRST_VECTOR_ADMIN], 0,
    myfirst_intr_simulate_vector_sysctl, "IU",
    "Simulate admin vector interrupt");
SYSCTL_ADD_PROC(ctx, kids, OID_AUTO, "intr_simulate_rx", ...);
SYSCTL_ADD_PROC(ctx, kids, OID_AUTO, "intr_simulate_tx", ...);
```

处理程序：

```c
static int
myfirst_intr_simulate_vector_sysctl(SYSCTL_HANDLER_ARGS)
{
	struct myfirst_vector *vec = arg1;
	struct myfirst_softc *sc = vec->sc;
	uint32_t mask = 0;
	int error;

	error = sysctl_handle_int(oidp, &mask, 0, req);
	if (error != 0 || req->newptr == NULL)
		return (error);

	MYFIRST_LOCK(sc);
	if (sc->hw == NULL || sc->bar_res == NULL) {
		MYFIRST_UNLOCK(sc);
		return (ENODEV);
	}
	bus_write_4(sc->bar_res, MYFIRST_REG_INTR_STATUS, mask);
	MYFIRST_UNLOCK(sc);

	/*
	 * Invoke this vector's filter if it has one (MSI-X). On single-
	 * handler tiers (MSI with 1 vector, or legacy INTx) only slot 0
	 * has a registered filter, so we fall through to it. The Chapter 19
	 * myfirst_intr_filter handles all three status bits in one pass.
	 */
	if (vec->filter != NULL)
		(void)vec->filter(vec);
	else if (sc->vectors[MYFIRST_VECTOR_ADMIN].filter != NULL)
		(void)sc->vectors[MYFIRST_VECTOR_ADMIN].filter(
		    &sc->vectors[MYFIRST_VECTOR_ADMIN]);
	return (0);
}
```

从用户空间：

```sh
sudo sysctl dev.myfirst.0.intr_simulate_admin=2  # ERROR bit, admin vector
sudo sysctl dev.myfirst.0.intr_simulate_rx=1     # DATA_AV bit, rx vector
sudo sysctl dev.myfirst.0.intr_simulate_tx=4     # COMPLETE bit, tx vector
```

每向量的 `intr_count` 计数器独立计数。读者可以通过触发每个 sysctl 并观察相应的 `vec->fire_count` 计数器上升来验证每向量行为。

### 传统回退时发生什么

当驱动程序回退到传统 INTx（因为 MSI 不可用或失败）时，只有一个处理程序覆盖所有三个事件类。代码将第 19 章的 `myfirst_intr_filter` 分配给管理向量的槽位，并在 `rid = 0` 上使用该单个过滤器。管理向量的过滤器变成一个多事件处理程序，查看所有三个状态位并相应地分发。

这是一个小而重要的细节：第 19 章的过滤器仍然存在并在传统路径上重用，而每向量过滤器仅在 MSI 或 MSI-X 可用时使用。检查驱动程序的读者会看到两者，差异在第 3 节的注释中解释。

### 阶段2的dmesg横幅

在驱动程序落在 MSI-X 层的虚拟机上（这是唯一提供三个向量的层；MSI 层由于前面解释的原因回退到单处理程序设置）：

```text
myfirst0: BAR0 allocated: 0x20 bytes at 0xfebf1000
myfirst0: hardware layer attached to BAR: 32 bytes
myfirst0: interrupt mode: MSI-X, 3 vectors
```

`vmstat -i | grep myfirst` 显示三行独立的内容：

```text
irq256: myfirst0:admin                 12         1
irq257: myfirst0:rx                    98         8
irq258: myfirst0:tx                    45         4
```

（确切的 IRQ 编号因平台而异；x86 上 MSI 分配的 IRQ 一旦 I/O APIC 范围用尽就从 256 范围开始。）

在只有 MSI 可用的虚拟机上，驱动程序报告单处理程序回退：

```text
myfirst0: interrupt mode: MSI, 1 vector (single-handler fallback)
```

`vmstat -i` 显示一行，因为驱动程序在该单个 MSI 向量上使用第 19 章的模式。

每向量细分（三行）是使多向量驱动程序可观察的原因。观察计数器的操作员可以知道哪个向量活跃以及频率。

### 阶段2常见错误

一个简短的列表。

**将 softc（而非向量）作为过滤器参数传递。** 如果你传递 `sc` 而不是 `vec`，过滤器无法知道它服务于哪个向量。修复：将 `vec` 传递给 `bus_setup_intr`；过滤器通过 `vec->sc` 访问 `sc`。

**忘记初始化 `vec->sc`。** 每向量结构由 `myfirst_init_softc` 零初始化；除非显式设置，`vec->sc` 保持为 NULL。没有它，过滤器的 `vec->sc->mtx` 访问是空指针解引用。修复：在设置期间、任何处理程序注册之前设置 `vec->sc = sc`。

**对多个向量使用相同的 rid。** MSI rid 是 1、2、3...；对管理向量和 RX 向量都重用 rid 1 意味着实际上只注册了一个处理程序。修复：按向量顺序分配 rid。

**每向量处理程序在没有锁的情况下访问共享状态。** 在不同 CPU 上运行的两个过滤器都尝试写入单个 `sc->counter`。没有原子操作或自旋锁，增量会丢失更新。修复：尽可能使用每向量计数器，对任何共享计数器使用原子操作。

**每向量任务存储在错误位置。** 如果任务在 softc 中而不是在向量结构中，两个向量入队"同一个"任务会冲突。修复：将任务存储在向量结构中，并将向量作为任务参数传递。

**部分设置失败时缺少每向量拆除。** goto 级联必须精确撤销成功的向量。缺少清理会留下已分配的 IRQ 资源。修复：使用每向量拆除辅助函数并在部分失败情况下向后迭代。

### 总结 Section 3

管理多个中断向量是一组三个新模式：每向量状态（在 `struct myfirst_vector` 数组中）、每向量过滤函数和每向量 `bus_describe_intr` 名称。每个向量在自己的 rid 上有自己的 IRQ 资源、自己的过滤函数（只读取与其向量相关的状态位）、自己的计数器和（可选）自己的延迟任务。单个 MSI 或 MSI-X 分配调用处理设备端状态；每向量 `bus_alloc_resource_any` 和 `bus_setup_intr` 调用单独处理每个处理程序。

第 2 节的回退阶梯自然扩展：首先尝试 N 个向量的 MSI；部分失败时，释放并尝试单个处理程序的传统 INTx。每向量拆除辅助函数使部分失败展开和干净拆除对称。

第 4 节是锁和数据结构部分。它检查当多个过滤器在多个 CPU 上同时运行时会发生什么，以及什么同步规范保持共享状态正常。



## 第4节：设计中断安全的数据结构

第 3 节添加了多个向量，每个都有自己的处理程序。第 4 节检查后果：多个处理程序可以在多个 CPU 上并发运行，处理程序共享的任何数据都必须相应地保护。规范并不新鲜；它是第 11 章锁模型的多 CPU 特化。新的是第 20 章的驱动程序有三条（或更多）并发过滤器上下文路径而不是一条。

第 4 节是多向量改变驱动程序状态形状的章节，而不仅仅是处理程序的数量。

### 新的并发图景

第 19 章的驱动程序有一个过滤器和一个任务。过滤器在内核路由中断到的任何 CPU 上运行；任务在任务队列的工作线程上运行。原则上它们两个可以同时运行：例如过滤器在 CPU 0 上，任务在 CPU 3 上。原子计数器和 `sc->mtx`（由任务持有，而不是过滤器）提供了所需的同步。

第 20 章的多向量驱动程序有三个过滤器和一个任务。在 MSI-X 系统上，每个过滤器有自己的 `intr_event`，所以每个都可以独立地在不同的 CPU 上触发。一微秒内到达的三个中断突发可以看到三个过滤器同时在三个 CPU 上运行。单个任务仍然通过任务队列串行化，但过滤器不是。

过滤器接触的数据分为三类：

1. **每向量状态。** 每个向量自己的计数器、自己的 cookie、自己的资源。向量之间没有共享。不需要同步。
2. **共享计数器。** 由任何过滤器更新的计数器（全局 `intr_data_av_count`、`intr_error_count` 等）。必须是原子的。
3. **共享设备状态。** BAR 本身、softc 的 `sc->hw` 指针、`sc->pci_attached`、互斥锁保护的字段。访问规则取决于上下文。

规范是保持每向量状态真正是每向量的，对共享计数器使用原子操作，并遵循第 11 章的锁规则处理任何需要睡眠互斥锁的内容。

### 每向量状态：默认情况

最简单的同步是没有同步。如果一段状态只被一个向量的过滤器接触（并且不被其他任何东西接触），则不需要锁。以下情况属于此类：

- `vec->fire_count`：仅由此向量的过滤器递增，通过 sysctl 读取器路径由 sysctl 处理程序读取。原子加就够了；过滤器和 sysctl 之间不需要锁，因为 sysctl 原子读取。
- `vec->stray_count`：相同模式。
- `vec->intr_cookie`：在设置时写入一次，在拆除时读取。单写入者，有序访问。
- `vec->irq_res`：相同模式。

大多数每向量状态属于此类。softc 中的 `struct myfirst_vector` 数组是关键模式：每个向量的状态存在于自己的槽中，只被自己的过滤器接触。

### 共享计数器：原子操作

全局每位置计数器（第 19 章引入了 `sc->intr_data_av_count` 等）由相应向量的过滤器更新。只有一个过滤器更新每个计数器，所以从技术上讲它们是"除了名字外是每向量的"。但读者可以想象一个场景：`INTR_STATUS` 中出现的位模式需要 RX 和管理向量都递增共享计数器。更安全的方法：使每次更新原子化。

第 20 章在整个过滤器路径中使用 `atomic_add_64`：

```c
atomic_add_64(&sc->intr_data_av_count, 1);
```

这很便宜（x86 上是一条锁定指令，arm64 上是屏障加加法），它让过滤器可以在任何 CPU 上运行而不必担心丢失更新。

`atomic_add_64` 在重度共享计数器上的代价是缓存行乒乓：来自不同 CPU 的每次增量都会使其他 CPU 上的缓存行失效。对于每秒从多个 CPU 递增百万次的计数器，这是可测量的性能损失。缓解措施是使计数器真正是每 CPU 的（使用 `counter(9)` 或 `DPCPU_DEFINE`）并仅在读取时求和；第 20 章的驱动程序没有达到那个规模，所以普通原子操作就可以。

### 共享设备状态：互斥锁规范

`sc->hw`、`sc->pci_attached`、`sc->bar_res`：这些在附加期间设置，在分离期间拆除。在稳定状态下，它们是只读的。过滤器在没有锁的情况下访问它们，因为生命周期规范（启用前附加，分离前禁用）确保指针在过滤器可以运行时有效。

规则：在没有锁的情况下访问 `sc->hw` 或 `sc->bar_res` 的过滤器必须确信附加-分离顺序保证指针有效。第 20 章第 7 节详细讲解顺序。就第 4 节的目的而言，信任规范：当过滤器运行时，设备已附加且指针有效。

### 每向量锁：何时需要

有时每向量状态比计数器更丰富。从接收队列读取并更新每队列数据结构（比如 mbuf 环）的向量需要自旋锁来保护环免受同一向量两次同时触发的影响。等等，在 MSI-X 系统上同一向量可以同时触发两次吗？

在 MSI-X 上，内核保证每个 `intr_event` 一次只交付给一个 CPU；单个向量不会重入自身。两个不同的向量可以同时在两个 CPU 上运行，但向量 N 不能同时在 CPU 3 和 CPU 5 上运行。

这意味着：**每向量状态不需要每向量锁**来防止来自同一向量的并发访问。它可能需要在过滤器和任务之间通信时使用一个（任务在不同 CPU 上运行，可能与过滤器并发），但自旋锁在那里就足够了，而且通信通常通过原子操作进行。

自旋锁在以下情况下变得有用：

- 驱动程序对多个向量使用单个过滤函数，内核可以并发调度两个向量的该过滤器。（第 20 章阶段 2 每个向量有单独的过滤器，所以这不适用。）
- 驱动程序在过滤器（填充环）和任务（排空环）之间共享接收环。自旋锁保护环索引；过滤器获取自旋锁，添加到环，释放。任务获取，排空，释放。

第 20 章的驱动程序在过滤器中不使用自旋锁；每向量计数器是原子的，共享状态通过任务中现有的 `sc->mtx` 处理。真实驱动程序在更丰富的场景中可能需要自旋锁。

### 每CPU数据：高级选项

对于非常高速率的驱动程序，即使共享数据上的原子计数器也会成为瓶颈。解决方案是每 CPU 数据：每个 CPU 有自己的计数器副本，过滤器递增自己 CPU 的副本（没有跨 CPU 流量），sysctl 读取器对每 CPU 值求和。

FreeBSD 的 `counter(9)` API 提供了这个：`counter_u64_t` 是每 CPU 数组的句柄，`counter_u64_add(c, 1)` 递增当前 CPU 的槽位，`counter_u64_fetch(c)` 在读取时对所有 CPU 的槽位求和。实现使用每 CPU 数据区域（底层是 `DPCPU_DEFINE`），在热路径上与普通非原子增量一样便宜。

第 20 章的驱动程序不使用 `counter(9)`；对于演示的规模，普通原子操作就足够了。真正的高吞吐量驱动程序（万兆网卡、百万 IOPS 的 NVMe 控制器）广泛使用 `counter(9)`。编写此类驱动程序的读者应该在第 20 章之后学习 `counter(9)`。

### 锁定顺序和多向量复杂情况

第 15 章确立了驱动程序的锁顺序：`sc->mtx -> sc->cfg_sx -> sc->stats_cache_sx`。第 19 章的过滤器不获取锁（仅原子操作）；任务获取 `sc->mtx`。第 20 章的每向量过滤器仍然不获取锁（仅原子操作），所以过滤器路径不贡献新的锁顺序边。每向量任务仍然获取 `sc->mtx`，与第 19 章的单个任务相同。

多个任务并发运行时的锁顺序需要一个小扩展。当管理任务和 RX 任务都获取 `sc->mtx` 时，它们在互斥锁上串行化。只要每个任务及时释放互斥锁，这就可以了；如果管理任务在等待慢速操作时持有 `sc->mtx`，RX 任务会停滞。第 15 章的"没有长时间持有的互斥锁"规则在这里也适用。

WITNESS 捕获大多数锁顺序问题。对于第 20 章，锁顺序故事本质上与第 19 章相同，因为过滤器路径是无锁的（仅原子操作），任务路径都获取相同的单个 `sc->mtx`。

### 内存模型：为什么原子操作重要

一个值得明确说明的微妙点。在多 CPU 系统中，一个 CPU 的写入不会立即对其他 CPU 可见。CPU 0 对 `sc->intr_count++` 的写入（没有原子操作）可能停留在 CPU 0 的存储缓冲区中，需要纳秒或微秒才能传播到 CPU 3 对相同内存的视图。在那个窗口中，CPU 3 可能读取到写入前的值。

`atomic_add_64` 包含一个内存屏障，强制写入在指令返回前全局可见。这就是使计数器值在 CPU 间"一致"的原因：增量后的任何读取者都能看到新值。

对于计数器状态，这种级别的一致性足够了。计数器在任何时刻的绝对值不重要；重要的是值单调增长并达到正确的总和。`atomic_add_64` 保证两者。

对于更丰富的共享状态（比如多个过滤器更新的共享数据结构索引），内存模型变得更微妙。驱动程序需要自旋锁，它同时提供互斥和内存屏障。第 20 章的驱动程序不需要这种级别的机制；第 19 章的原子规范延续下来。

### 可观察性：sysctl中的每向量计数器

每个向量有自己的 sysctl 子树，以便操作员查询：

```c
char name[32];
for (int i = 0; i < MYFIRST_MAX_VECTORS; i++) {
	snprintf(name, sizeof(name), "vec%d_fire_count", i);
	SYSCTL_ADD_U64(&sc->sysctl_ctx,
	    SYSCTL_CHILDREN(sc->sysctl_tree), OID_AUTO, name,
	    CTLFLAG_RD, &sc->vectors[i].fire_count, 0,
	    "Fire count for this vector");
}
```

从用户空间：

```sh
sysctl dev.myfirst.0 | grep vec
```

```text
dev.myfirst.0.vec0_fire_count: 42    # admin
dev.myfirst.0.vec0_stray_count: 0
dev.myfirst.0.vec1_fire_count: 9876  # rx
dev.myfirst.0.vec1_stray_count: 0
dev.myfirst.0.vec2_fire_count: 4523  # tx
dev.myfirst.0.vec2_stray_count: 0
```

操作员可以一目了然地看到哪些向量在触发以及以什么速率。在 MSI-X 上杂散计数应保持为零（每个向量有自己的专用消息），但在 MSI 或传统模式下，当共享过滤器看到不同向量的事件时可能会计数。

### 总结 Section 4

多向量驱动程序改变了并发图景：几个过滤器可以在几个 CPU 上同时运行。规范是在可能的情况下按向量设计状态，对共享计数器使用原子操作，并遵循现有的第 11 章锁顺序处理任何需要睡眠互斥锁的内容。每 CPU 计数器（`counter(9)`）可用于非常高速率的驱动程序，但对于第 20 章是过度的。

驱动程序的锁顺序没有获得新边，因为过滤器路径保持无锁（仅原子操作），任务都获取 `sc->mtx`。WITNESS 仍然捕获锁顺序问题；原子规范仍然捕获其余部分。

第 5 节转向更强大的机制：MSI-X。API 非常相似（`pci_msix_count` + `pci_alloc_msix` 代替 MSI 对），但扩展性和 CPU 亲和性选项更丰富。



## 第5节：使用MSI-X获得高灵活性

第 2 节介绍了单向量的 MSI，第 3 节扩展到多 MSI 向量，第 4 节讲解了并发影响。第 5 节转向 MSI-X：现代 PCIe 设备在管理超过少数几个中断时使用的更完整机制。API 与 MSI 的平行，所以代码变更很小；概念上的变化是 MSI-X 让驱动程序通过 `bus_bind_intr(9)` 和 `bus_get_cpus(9)` 将每个向量绑定到特定 CPU，这对真实性能很重要。

### MSI-X计数和分配API

API 与 MSI 的镜像相似：

```c
int msix_count = pci_msix_count(sc->dev);
```

`pci_msix_count(9)` 返回设备声明的 MSI-X 向量数量（如果没有 MSI-X 能力则返回 0）。计数来自 MSI-X 能力的 `Table Size` 字段加一；`Table Size = 7` 的设备声明 8 个向量。

分配类似：

```c
int count = desired;
int error = pci_alloc_msix(sc->dev, &count);
```

相同的输入输出 `count` 参数，相同的语义：内核可能分配少于请求的数量。与 MSI 不同，MSI-X 允许非二的幂计数，所以如果驱动程序请求 3 个向量，内核可以给出 3 个。

相同的 `pci_release_msi(9)` 调用释放 MSI-X 向量；没有单独的 `pci_release_msix`。函数名是历史遗留；它同时处理 MSI 和 MSI-X。

### 扩展的回退阶梯

第 20 章驱动程序的完整回退阶梯是：

1. **MSI-X** 使用所需的向量计数。
2. **MSI** 使用所需的向量计数，如果 MSI-X 不可用或分配失败。
3. **传统 INTx** 使用单个处理程序处理所有内容，如果 MSI-X 和 MSI 都失败。

代码结构平行于第 3 节的两层阶梯，在顶部扩展了第三层：

```c
/* Tier 0: MSI-X. */
wanted = MYFIRST_MAX_VECTORS;
if (pci_msix_count(sc->dev) >= wanted) {
	allocated = wanted;
	if (pci_alloc_msix(sc->dev, &allocated) == 0 &&
	    allocated == wanted) {
		for (i = 0; i < wanted; i++) {
			error = myfirst_intr_setup_vector(sc, i, i + 1);
			if (error != 0)
				goto fail_msix;
		}
		sc->intr_mode = MYFIRST_INTR_MSIX;
		sc->num_vectors = wanted;
		device_printf(sc->dev,
		    "interrupt mode: MSI-X, %d vectors\n", wanted);
		myfirst_intr_bind_vectors(sc);
		goto enabled;
	}
	if (allocated > 0)
		pci_release_msi(sc->dev);
}

/* Tier 1: MSI. */
/* ... Section 3 MSI code ... */

fail_msix:
for (i -= 1; i >= 0; i--)
	myfirst_intr_teardown_vector(sc, i);
pci_release_msi(sc->dev);
/* fallthrough to MSI attempt, then legacy. */
```

结构很直接：每一层都有相同的模式（查询计数、分配、设置向量、标记模式、描述）。代码遵循熟悉的瀑布模式。

### 使用bus_bind_intr进行向量绑定

一旦分配了 MSI-X，驱动程序可以选择将每个向量绑定到特定 CPU。API 是：

```c
int bus_bind_intr(device_t dev, struct resource *r, int cpu);
```

`cpu` 是从 0 到 `mp_ncpus - 1` 的整数 CPU ID。成功时，中断被路由到该 CPU。失败时，函数返回 errno；驱动程序将其视为非致命提示并继续不绑定。

对于第 20 章的三向量驱动程序，合理的绑定是：

- **管理向量**：CPU 0（控制工作，任何 CPU 都可以）。
- **RX 向量**：CPU 1（真实 RX 队列的缓存局部性好处）。
- **TX 向量**：CPU 2（类似的局部性好处）。

在双 CPU 系统上，绑定会折叠；在多 CPU 系统上，驱动程序应该使用 `bus_get_cpus(9)` 查询哪些 CPU 是设备 NUMA 节点的本地 CPU 并相应地分发向量。

绑定辅助函数：

```c
static void
myfirst_intr_bind_vectors(struct myfirst_softc *sc)
{
	int i, cpu, ncpus;
	int err;

	if (mp_ncpus < 2)
		return;  /* nothing to bind */

	ncpus = mp_ncpus;
	for (i = 0; i < sc->num_vectors; i++) {
		cpu = i % ncpus;
		err = bus_bind_intr(sc->dev, sc->vectors[i].irq_res, cpu);
		if (err != 0) {
			device_printf(sc->dev,
			    "bus_bind_intr vector %d to CPU %d: %d\n",
			    i, cpu, err);
		}
	}
}
```

代码是轮询绑定：向量 0 到 CPU 0，向量 1 到 CPU 1，依此类推，对 CPU 计数取模。在有三个向量的双 CPU 系统上，向量 0 和向量 2 都落在 CPU 0 上；在四 CPU 系统上，每个向量有自己的 CPU。

更复杂的驱动程序使用 `bus_get_cpus(9)`：

```c
cpuset_t local_cpus;
int ncpus_local;

if (bus_get_cpus(sc->dev, LOCAL_CPUS, sizeof(local_cpus),
    &local_cpus) == 0) {
	/* Use only CPUs in local_cpus for binding. */
	ncpus_local = CPU_COUNT(&local_cpus);
	/* ... pick from local_cpus ... */
}
```

`LOCAL_CPUS` 参数返回与设备在同一 NUMA 域中的 CPU。`INTR_CPUS` 参数返回适合处理设备中断的 CPU（通常排除绑定到关键工作的 CPU）。关心 NUMA 性能的驱动程序使用这些将向量放置在 NUMA 本地 CPU 上。

第 20 章的驱动程序默认不使用 `bus_get_cpus(9)`；更简单的轮询绑定对实验来说足够了。一个挑战练习添加 NUMA 感知绑定。

### MSI-X的dmesg摘要

第 20 章驱动程序打印如下一行：

```text
myfirst0: interrupt mode: MSI-X, 3 vectors
```

每向量 CPU 绑定在 `vmstat -i` 中可见（vmstat -i 中的每 CPU 总计不是每向量的；它们是聚合的）以及在 `cpuset -g -x <irq>` 输出中（每个向量一个查询）：

```sh
for irq in 256 257 258; do
    echo "IRQ $irq:"
    cpuset -g -x $irq
done
```

典型输出：

```text
IRQ 256:
irq 256 mask: 0
IRQ 257:
irq 257 mask: 1
IRQ 258:
irq 258 mask: 2
```

（IRQ 编号取决于平台的分配。）

检查驱动程序中断设置的操作员可以看到哪些向量在哪里触发。

### 每向量bus_describe_intr

每个 MSI-X 向量应该有描述。第 3 节的代码已经通过 `bus_describe_intr(9)` 设置了它们：

```c
bus_describe_intr(sc->dev, vec->irq_res, vec->intr_cookie,
    "%s", vec->name);
```

此后，`vmstat -i` 显示每个向量及其角色：

```text
irq256: myfirst0:admin                 42         4
irq257: myfirst0:rx                 12345      1234
irq258: myfirst0:tx                  5432       543
```

操作员可以看到哪个向量是管理、哪个是 RX、哪个是 TX，以及每个的繁忙程度。这对于多向量驱动程序是必不可少的可观察性。

### MSI-X表和BAR考虑

一个值得一提的细节，尽管驱动程序不直接与它交互。MSI-X 能力结构指向一个**表**和一个**待决位数组**（PBA），每个都位于设备的某个 BAR 中。容纳每个的 BAR 可通过 `pci_msix_table_bar(9)` 和 `pci_msix_pba_bar(9)` 发现：

```c
int table_bar = pci_msix_table_bar(sc->dev);
int pba_bar = pci_msix_pba_bar(sc->dev);
```

每个返回 BAR 索引（0 到 5）或 -1（如果设备没有 MSI-X 能力）。对于大多数设备，表和 PBA 在 BAR 0 或 BAR 1 中；对于某些设备，它们与内存映射寄存器共享一个 BAR（驱动程序的 BAR 0）。

内核在内部处理表编程。驱动程序的唯一交互是：

- 确保包含表的 BAR 已分配（以便内核可以访问它）。在某些设备上，这需要驱动程序分配额外的 BAR。
- 调用 `pci_alloc_msix` 并让内核完成其余工作。

对于第 20 章的驱动程序，virtio-rnd 目标（或其在 QEMU 中支持 MSI-X 的等效设备）的表在 BAR 1 或专用区域中。第 18 章代码分配了 BAR 0；内核通过分配基础设施隐式处理 MSI-X 表 BAR。

想要检查表 BAR 的驱动程序：

```c
device_printf(sc->dev, "MSI-X table in BAR %d, PBA in BAR %d\n",
    pci_msix_table_bar(sc->dev), pci_msix_pba_bar(sc->dev));
```

这对诊断目的很有用。

### 分配少于请求的向量

一个微妙的情况：设备声明 3 个 MSI-X 向量，驱动程序请求 3 个，但内核只分配 2 个。驱动程序怎么做？

答案取决于驱动程序的设计。选项：

1. **附加失败。** 如果驱动程序无法以更少的向量运行，返回错误。这对灵活的驱动程序来说很少见，但对于有严格硬件要求的驱动程序是可能的。
2. **使用得到的。** 如果驱动程序可以用 2 个向量运行（例如将 RX 和 TX 合并为一个），使用这 2 个并调整配置。这对针对多种硬件的驱动程序很常见。
3. **释放并回退。** 如果 2 个向量由于某种原因比 1 个 MSI 向量更差，释放 MSI-X 并尝试 MSI。这不常见。

第 20 章的驱动程序选择选项 1：如果它没有获得正好 `MYFIRST_MAX_VECTORS`（3）个向量，它释放 MSI-X 并回退到 MSI。更复杂的驱动程序会使用选项 2；第 20 章的教学专注于更简单的模式。

真正的 FreeBSD 驱动程序通常使用选项 2，带有一个辅助函数来计算如何在所需角色之间分发分配的向量。`nvme(4)` 驱动程序是一个例子：如果它请求 N 个 I/O 队列的向量并获得较少，它会相应减少 I/O 队列的数量。

### 在bhyve与QEMU上测试MSI-X

关于实验的一个实际细节。bhyve 的遗留 virtio-rnd 设备（第 18 和 19 章使用的）不暴露 MSI-X；它是一个仅遗留的 virtio 传输。要在虚拟机中练习 MSI-X，读者需要以下之一：

- **QEMU 带 `-device virtio-rng-pci`**（不是 `-device virtio-rng`，它是遗留的）。现代 virtio-rng-pci 暴露 MSI-X。
- **现代 bhyve 模拟** 具有 MSI-X 的非 virtio-rnd 设备。第 20 章不使用此路径。
- **支持 MSI-X 的真实硬件**（大多数现代 PCIe 设备）。

QEMU 是第 20 章实验的实际选择。驱动程序的回退阶梯确保它仍在 bhyve 上工作（回退到遗留）；专门测试 MSI-X 需要 QEMU 或真实硬件。

### MSI-X设置中的常见错误

一个简短的列表。

**使用 `pci_release_msix`。** 这个函数在 FreeBSD 中不存在；释放由 `pci_release_msi(9)` 处理，它同时适用于 MSI 和 MSI-X。修复：使用 `pci_release_msi`。

**绑定到设备无法到达的 CPU。** 某些平台（很少）有不在中断控制器可路由集中的 CPU。`bus_bind_intr` 调用返回错误；忽略它并继续。修复：记录错误但不使附加失败。

**期望 vmstat -i 显示每 CPU 细分。** `vmstat -i` 聚合每事件计数。每 CPU 细分可通过 `cpuset -g -x <irq>`（或原始形式的 `sysctl hw.intrcnt`）获得。操作员必须在正确的地方查找。修复：为您的驱动程序记录可观察性路径。

**未能检查 `allocated` 与 `wanted`。** 当驱动程序无法处理部分分配时接受它会导致微妙的错误（应该触发的向量从不触发）。修复：预先决定策略（失败、适应或释放）并相应编码。

### 总结 Section 5

MSI-X 是更完整的机制：一个每向量可寻址表，由内核代表驱动程序编程，具有每向量 CPU 亲和性和每向量掩码供需要它们的驱动程序使用。API 与 MSI 的紧密镜像（`pci_msix_count` + `pci_alloc_msix` + `pci_release_msi`），每向量资源分配与第 3 节的 MSI 代码相同。新部分是用于 CPU 亲和性的 `bus_bind_intr(9)` 和用于 NUMA 本地 CPU 查询的 `bus_get_cpus(9)`。

对于第 20 章的驱动程序，MSI-X 是首选层；回退阶梯首先尝试 MSI-X，回退到 MSI，最后到传统 INTx。第 3 节的每向量处理程序、计数器和任务在 MSI-X 上工作不变；只有分配调用改变。

第 6 节是向量特定事件变得明确的地方。每个向量有自己的目的、自己的过滤器逻辑和自己的可观察行为。第 20 章阶段 3 是驱动程序看起来像真正的多队列设备的阶段，尽管底层硅（virtio-rnd 目标）更简单。



## 第6节：处理向量特定事件

第 2 到 5 节构建了多向量处理的基础设施。第 6 节是每向量角色变得明确的章节。每个向量有它处理的特定事件类；每个过滤器有它做的特定检查；每个任务有它执行的特定唤醒。阶段 3 的驱动程序将向量视为命名的、有目的的实体，而不是可互换的槽位。

第 20 章驱动程序的三个向量有不同的职责：

- **管理向量** 处理 `ERROR` 事件。过滤器读取状态、确认，并（在真正错误时）记录消息。管理工作不频繁但不能丢弃。
- **RX 向量** 处理 `DATA_AV`（接收可用）事件。过滤器确认并将数据处理工作延迟到广播条件变量的每向量任务。
- **TX 向量** 处理 `COMPLETE`（发送完成）事件。过滤器确认并可选地唤醒等待发送完成的线程。过滤器内联处理簿记。

每个向量可通过模拟中断 sysctl 独立测试，通过其计数器独立观察，并独立绑定到 CPU。驱动程序开始看起来像一个小型的真正多队列设备。

### 管理向量

管理向量处理罕见但重要的事件：配置更改、错误、链路状态更改（对于网卡）、温度警报（对于传感器）。它的工作通常很小：记录事件、更新状态标志、唤醒轮询状态的用户空间等待者。

对于第 20 章驱动程序，管理向量处理第 17 章的 `ERROR` 位。过滤器：

```c
int
myfirst_admin_filter(void *arg)
{
	struct myfirst_vector *vec = arg;
	struct myfirst_softc *sc = vec->sc;
	uint32_t status;

	status = ICSR_READ_4(sc, MYFIRST_REG_INTR_STATUS);
	if ((status & MYFIRST_INTR_ERROR) == 0) {
		atomic_add_64(&vec->stray_count, 1);
		return (FILTER_STRAY);
	}

	atomic_add_64(&vec->fire_count, 1);
	ICSR_WRITE_4(sc, MYFIRST_REG_INTR_STATUS, MYFIRST_INTR_ERROR);
	atomic_add_64(&sc->intr_error_count, 1);
	return (FILTER_HANDLED);
}
```

在真实设备上，管理过滤器可能还会检查辅助寄存器（比如错误代码寄存器）并根据严重程度决定是否安排恢复任务。第 20 章的驱动程序保持简单：计数并确认。

### RX向量

RX 向量是数据路径向量。对于网卡，它会处理接收到的数据包。对于 NVMe 驱动器，读请求的完成。对于第 20 章驱动程序，它处理第 17 章的 `DATA_AV` 位。

过滤器很小（确认并入队任务）；任务做真正的工作。第 3 节展示了两者。任务：

```c
static void
myfirst_rx_task_fn(void *arg, int npending)
{
	struct myfirst_vector *vec = arg;
	struct myfirst_softc *sc = vec->sc;

	MYFIRST_LOCK(sc);
	if (sc->hw != NULL && sc->pci_attached) {
		sc->intr_last_data = CSR_READ_4(sc, MYFIRST_REG_DATA_OUT);
		sc->intr_task_invocations++;
		cv_broadcast(&sc->data_cv);
	}
	MYFIRST_UNLOCK(sc);
}
```

在真正的驱动程序中，任务会遍历接收描述符环、构建 mbuf 并将它们传递给网络栈。对于第 20 章的演示，它读取 `DATA_OUT`、存储值、广播条件变量，并让任何等待的 cdev 读取者唤醒。

`npending` 参数是任务自上次运行以来被入队的次数。对于高速 RX 路径，运行一次并看到 `npending = 5` 的任务知道它落后了（5 个中断合并为 1 次任务运行），可以相应地调整批次大小。第 20 章的任务忽略 `npending`；真正的驱动程序使用它进行批处理。

### TX向量

TX 向量是发送完成向量。对于网卡，它表示驱动程序交给硬件的数据包已发送，缓冲区可以回收。对于 NVMe 驱动器，它表示写请求已完成。

对于第 20 章驱动程序，它处理第 17 章的 `COMPLETE` 位。过滤器在线完成工作（不需要任务）：

```c
int
myfirst_tx_filter(void *arg)
{
	struct myfirst_vector *vec = arg;
	struct myfirst_softc *sc = vec->sc;
	uint32_t status;

	status = ICSR_READ_4(sc, MYFIRST_REG_INTR_STATUS);
	if ((status & MYFIRST_INTR_COMPLETE) == 0) {
		atomic_add_64(&vec->stray_count, 1);
		return (FILTER_STRAY);
	}

	atomic_add_64(&vec->fire_count, 1);
	ICSR_WRITE_4(sc, MYFIRST_REG_INTR_STATUS, MYFIRST_INTR_COMPLETE);
	atomic_add_64(&sc->intr_complete_count, 1);
	return (FILTER_HANDLED);
}
```

TX 过滤器的仅在线设计是一个刻意的选择。在真正的 TX 完成路径上，过滤器可能记录完成计数，任务可能遍历 TX 描述符环以回收缓冲区。对于第 20 章的演示，完成只是被计数。

另一种设计是让 TX 也使用任务。是否这样做取决于任务要做的工作：如果工作量大（遍历环、回收数十个缓冲区），任务是值得的；如果工作量小（单次减少正在进行的计数器），在过滤器中在线处理就可以了。第 20 章选择 TX 在线处理以说明并非每个向量都需要任务。

### 向量到事件映射

在 MSI-X 上，每个向量是独立的；触发向量 1 传递给 RX 过滤器，触发向量 2 传递给 TX 过滤器。从向量到事件的映射是驱动程序设计的一部分，不是内核的选择。

在具有多个向量的 MSI 上，如果多个事件同时触发，内核原则上可以快速连续地分发多个向量。驱动程序的过滤器必须各自读取状态寄存器，并仅声明属于其向量的位。

在 legacy INTx 上，只有一个向量和一个过滤器。过滤器一次性处理所有三类事件。

第 20 章的代码处理所有三种情况：MSI-X 上的每向量过滤器仅读取自己的位，MSI 上的每向量过滤器做同样的操作（使用相同的位检查逻辑），legacy INTx 上的单个过滤器处理所有三个位。

### 模拟每向量事件

第 3 节的模拟 sysctl 让读者可以独立地测试每个向量。从用户空间：

```sh
# Simulate an admin interrupt (ERROR).
sudo sysctl dev.myfirst.0.intr_simulate_admin=2

# Simulate an RX interrupt (DATA_AV).
sudo sysctl dev.myfirst.0.intr_simulate_rx=1

# Simulate a TX interrupt (COMPLETE).
sudo sysctl dev.myfirst.0.intr_simulate_tx=4
```

每个 sysctl 将其特定的位写入 `INTR_STATUS` 并调用相应向量的过滤器。在 MSI-X 层级上，三个过滤器都存在，因此每个 sysctl 都会触发其自己的每向量过滤器，其每向量计数器会递增。在 MSI 层级（槽位 0 的单个向量）和 legacy INTx 层级（槽位 0 的单个向量）上，槽位 1 和 2 没有注册过滤器，因此模拟辅助函数将调用路由到槽位 0 的过滤器。第 19 章的 `myfirst_intr_filter` 在线处理所有三个位，因此全局 `intr_count`、`intr_data_av_count`、`intr_error_count` 和 `intr_complete_count` 计数器仍能正确移动。在单处理程序层级上，槽位 1 和 2 的每向量计数器保持为零，这是正确的可观察性信号，表明驱动程序不是以三个向量运行的。

读者可以从用户空间观察管道：

```sh
while true; do
    sudo sysctl dev.myfirst.0.intr_simulate_rx=1
    sleep 0.1
done &
watch sysctl dev.myfirst.0 | grep -E "vec|intr_"
```

计数器大约每秒递增 10 次。RX 向量的计数器与 `intr_data_av_count` 匹配；任务的调用计数匹配。

### 动态向量分配

一个微妙但重要的点。驱动程序的设计在固定数组中有三个向量，具有固定的角色。更灵活的驱动程序可能在运行时发现可用向量的数量并动态分配角色。模式如下：

```c
/* Discover how many vectors we got. */
int nvec = actually_allocated_msix_vectors(sc);

/* Assign roles based on nvec. */
if (nvec >= 3) {
	/* Full design: admin, rx, tx. */
	sc->vectors[0].filter = myfirst_admin_filter;
	sc->vectors[1].filter = myfirst_rx_filter;
	sc->vectors[2].filter = myfirst_tx_filter;
	sc->num_vectors = 3;
} else if (nvec == 2) {
	/* Compact: admin+tx share one vector, rx has its own. */
	sc->vectors[0].filter = myfirst_admin_tx_filter;
	sc->vectors[1].filter = myfirst_rx_filter;
	sc->num_vectors = 2;
} else if (nvec == 1) {
	/* Minimal: one filter handles everything. */
	sc->vectors[0].filter = myfirst_intr_filter;
	sc->num_vectors = 1;
}
```

这种动态适应是生产驱动程序所做的。第 20 章的驱动程序使用更简单的固定方法；一个挑战练习添加动态变体。

### 来自nvme(4)的模式

对于真实示例，`nvme(4)` 驱动程序将管理队列与 I/O 队列分开处理。其过滤器函数因队列类型不同；其中断计数按队列跟踪。模式如下：

```c
/* In nvme_ctrlr_construct_admin_qpair: */
qpair->intr_idx = 0;  /* vector 0 for admin */
qpair->intr_rid = 1;
qpair->res = bus_alloc_resource_any(ctrlr->dev, SYS_RES_IRQ,
    &qpair->intr_rid, RF_ACTIVE);
bus_setup_intr(ctrlr->dev, qpair->res, INTR_TYPE_MISC | INTR_MPSAFE,
    NULL, nvme_qpair_msix_handler, qpair, &qpair->tag);

/* For each I/O queue: */
for (i = 0; i < ctrlr->num_io_queues; i++) {
	ctrlr->ioq[i].intr_rid = i + 2;  /* I/O vectors at rid 2, 3, ... */
	/* ... similar bus_alloc_resource_any + bus_setup_intr ... */
}
```

每个队列有自己的 `intr_rid`、自己的资源、自己的标签（cookie）、自己的处理程序参数。管理队列使用一个向量；每个 I/O 队列使用自己的向量。模式随队列数量线性扩展。

第 20 章的驱动程序是这个的小版本：三个固定向量而不是一个管理加 N 个 I/O。扩展故事直接转移。

### 可观察性：每向量速率

一个有用的诊断：计算滑动窗口中每个向量的速率：

```sh
#!/bin/sh
prev_admin=$(sysctl -n dev.myfirst.0.vec0_fire_count)
prev_rx=$(sysctl -n dev.myfirst.0.vec1_fire_count)
prev_tx=$(sysctl -n dev.myfirst.0.vec2_fire_count)
sleep 1
curr_admin=$(sysctl -n dev.myfirst.0.vec0_fire_count)
curr_rx=$(sysctl -n dev.myfirst.0.vec1_fire_count)
curr_tx=$(sysctl -n dev.myfirst.0.vec2_fire_count)

echo "admin: $((curr_admin - prev_admin)) /s"
echo "rx:    $((curr_rx    - prev_rx   )) /s"
echo "tx:    $((curr_tx    - prev_tx   )) /s"
```

输出是最后一秒的每向量速率。在循环中运行模拟中断 sysctl 的读者可以看到速率上升；观察真实工作负载的读者可以看到哪个向量繁忙。

### 总结 Section 6

处理向量特定事件意味着每个向量有自己的过滤函数、自己的计数器、自己的（可选）任务和自己的可观察行为。模式扩展：第 20 章演示的三个向量，生产网卡的数十个，NVMe 控制器的数百个。每向量分离使每个部分小而特定、可维护。

第 7 节是拆除章节。多向量驱动程序需要单独拆除每个向量，按正确顺序，然后在最后调用一次 `pci_release_msi`。顺序严格但不复杂；第 7 节详细讲解。



## 第7节：MSI/MSI-X的拆除和清理

第 19 章的拆除是一对操作：在一个向量上 `bus_teardown_intr`，然后在一个 IRQ 资源上 `bus_release_resource`。第 20 章的拆除是每向量重复相同的一对操作，最后调用一次 `pci_release_msi` 来撤销 MSI 或 MSI-X 设备级分配。

本节明确顺序，遍历部分失败情况，并强调确认干净拆除的可观察性检查。

### 所需顺序

对于多向量驱动程序，分离序列是：

1. **如果忙则拒绝。** 与第 19 章相同：如果驱动程序有打开的描述符或正在进行的工作，返回 `EBUSY`。
2. **标记为不再附加。**
3. **销毁 cdev。**
4. **在设备处禁用中断。** 清除 `INTR_MASK` 使设备停止请求。
5. **对每个向量按逆序：**
   a. 对向量的 cookie 调用 `bus_teardown_intr`。
   b. 对向量的 IRQ 资源调用 `bus_release_resource`。
6. **排空所有每向量任务。** 每个已初始化的任务。
7. **销毁任务队列。**
8. **调用 `pci_release_msi`** 一次，如果 `intr_mode` 是 MSI 或 MSI-X 则无条件调用。
9. **分离硬件层并释放 BAR**，与之前相同。
10. **取消初始化 softc。**

新步骤是 5（每向量循环而不是单对）和 8（`pci_release_msi`）。步骤 1-4 和 9-10 与第 19 章相同。

### 为什么每向量逆序

每向量逆序循环是针对向量间依赖关系的防御措施。在第 20 章的简单驱动程序上，向量是独立的：在向量 1 之前拆除向量 2 是可以的。在向量 2 的过滤器读取向量 1 的过滤器写入的状态的驱动程序上，顺序很重要：先拆除写入者（向量 1），然后读取者（向量 2）。

对于第 20 章驱动程序的正确性，正向和逆向顺序都是安全的。对于未来更改的鲁棒性，逆向顺序更可取。

### 每向量拆除代码

来自第 3 节，每向量拆除辅助函数：

```c
static void
myfirst_intr_teardown_vector(struct myfirst_softc *sc, int idx)
{
	struct myfirst_vector *vec = &sc->vectors[idx];

	if (vec->intr_cookie != NULL) {
		bus_teardown_intr(sc->dev, vec->irq_res, vec->intr_cookie);
		vec->intr_cookie = NULL;
	}
	if (vec->irq_res != NULL) {
		bus_release_resource(sc->dev, SYS_RES_IRQ, vec->irq_rid,
		    vec->irq_res);
		vec->irq_res = NULL;
	}
}
```

辅助函数对部分设置具有鲁棒性：如果向量从未有 cookie（设置在 `bus_setup_intr` 之前失败），`if` 检查跳过拆除调用。如果资源从未分配，第二个 `if` 检查跳过释放。相同的辅助函数用于设置期间的部分失败展开和分离期间的完全拆除。

### 完整拆除

```c
void
myfirst_intr_teardown(struct myfirst_softc *sc)
{
	int i;

	MYFIRST_LOCK(sc);
	if (sc->hw != NULL && sc->bar_res != NULL)
		CSR_WRITE_4(sc, MYFIRST_REG_INTR_MASK, 0);
	MYFIRST_UNLOCK(sc);

	/* Tear down each vector's handler, in reverse. */
	for (i = sc->num_vectors - 1; i >= 0; i--)
		myfirst_intr_teardown_vector(sc, i);

	/* Drain and destroy the taskqueue, including per-vector tasks. */
	if (sc->intr_tq != NULL) {
		for (i = 0; i < sc->num_vectors; i++) {
			if (sc->vectors[i].has_task)
				taskqueue_drain(sc->intr_tq,
				    &sc->vectors[i].task);
		}
		taskqueue_free(sc->intr_tq);
		sc->intr_tq = NULL;
	}

	/* Release the MSI/MSI-X allocation if used. */
	if (sc->intr_mode == MYFIRST_INTR_MSI ||
	    sc->intr_mode == MYFIRST_INTR_MSIX)
		pci_release_msi(sc->dev);

	sc->num_vectors = 0;
	sc->intr_mode = MYFIRST_INTR_LEGACY;
}
```

代码结构直接：在设备处禁用、每向量逆序拆除、每向量任务排空、释放任务队列、如果使用则释放 MSI。模式直接转移到任何多向量驱动程序。

### 部分附加失败回退

在设置期间，如果向量 N 注册失败，代码必须展开成功的向量 0 到 N-1。模式如下：

```c
for (i = 0; i < MYFIRST_MAX_VECTORS; i++) {
	error = myfirst_intr_setup_vector(sc, i, i + 1);
	if (error != 0)
		goto fail_vectors;
}

/* Success, continue. */

fail_vectors:
	/* Undo vectors 0 through i-1. */
	for (i -= 1; i >= 0; i--)
		myfirst_intr_teardown_vector(sc, i);
	pci_release_msi(sc->dev);
	/* Fall through to next tier or final failure. */
```

"`i -= 1` 很重要：在 `goto` 之后，`i` 是失败的向量（它在成功的设置之后）。我们撤销向量 0 到 i-1，这是成功注册的集合。每向量拆除辅助函数也可以安全地调用失败向量的槽位，因为其字段为 NULL（设置没有足够远来填充它们）。

### 可观察性：验证干净拆除

在 `kldunload myfirst` 之后，应该满足以下条件：

- `kldstat -v | grep myfirst` 不返回任何内容。
- `devinfo -v | grep myfirst` 不返回任何内容。
- `vmstat -i | grep myfirst` 不返回任何内容。
- `vmstat -m | grep myfirst` 显示零活跃分配。

任何失败都指向清理错误：

- `vmstat -i` 条目仍然存在意味着该向量没有调用 `bus_teardown_intr`。
- `vmstat -m` 泄漏意味着每向量任务没有被排空或任务队列没有被释放。
- `devinfo -v` 条目仍然存在（罕见）意味着设备的detach未完成。

### 跨加载-卸载周期的MSI资源泄漏

MSI/MSI-X 驱动程序的一个特定关注点：忘记 `pci_release_msi` 会留下设备已分配的 MSI 状态。同一驱动程序（或同一设备的不同驱动程序）的下次 `kldload` 将无法分配 MSI 向量，因为内核认为它们已在使用中。

`dmesg` 中的症状：

```text
myfirst0: pci_alloc_msix returned EBUSY
```

或类似。修复方法是确保 `pci_release_msi` 在每个拆除路径上运行，包括部分失败回退。

一个有用的测试：加载、卸载、加载。如果第二次加载以相同的 MSI 模式成功，拆除是正确的。如果第二次加载回退到更低层，拆除泄漏了。

### 拆除中的常见错误

一个简短的列表。

**忘记 `pci_release_msi`。** 最常见的错误。症状：下次 MSI 分配尝试失败。修复：使用 MSI 或 MSI-X 时始终调用它。

**仅在使用 legacy INTx 的驱动程序上调用 `pci_release_msi`。** 技术上是无操作，但显式检查使意图更清晰。修复：调用前检查 `intr_mode`。

**错误的每向量拆除顺序。** 对于具有向量间依赖关系的驱动程序，逆序循环很重要。对于第 20 章的驱动程序，顺序不是关键依赖，但逆序规范成本低且值得保持。

**排空从未初始化的任务。** 如果向量没有 `has_task`，排空其未初始化的 `task` 字段会产生垃圾。修复：排空前检查 `has_task`。

**泄漏任务队列。** `taskqueue_drain` 不会释放任务队列；`taskqueue_free` 会。两者都需要。修复：调用两者。

**部分设置回退过多。** 如果向量 2 失败且回退代码也拆除向量 2（从未设置），NULL 解引用随之而来。每向量辅助函数的 NULL 检查可以防止这种情况，但级联逻辑也应该小心。修复：使用 `i -= 1` 从正确的向量开始回退。

### 总结 Section 7

多向量驱动程序的拆除是循环中的每向量，最后是单个 `pci_release_msi`。每向量辅助函数在完全拆除和部分失败展开之间共享。卸载后的可观察性检查与第 19 章使用的相同；任何泄漏都指向特定的错误。

第 8 节是重构部分：将多向量代码拆分到 `myfirst_msix.c`，更新 `INTERRUPTS.md` 以反映新功能，将版本升级到 `1.3-msi`，并运行回归测试。驱动程序在第 7 节后功能完整；第 8 节使其可维护。



## 第8节：重构和版本化多向量驱动程序

多向量中断处理程序正在工作。第 8 节是整理章节。它将 MSI/MSI-X 代码拆分到自己的文件，更新模块元数据，用新的多向量细节扩展 `INTERRUPTS.md` 文档，将版本升级到 `1.3-msi`，并运行回归测试。

这是第四个以重构章节结束的章节。重构累积：第 16 章拆分出硬件层，第 17 章模拟，第 18 章 PCI 附加，第 19 章遗留中断。第 20 章添加 MSI/MSI-X 层。每个责任有自己的文件；主 `myfirst.c` 大致保持相同大小；驱动程序扩展。

### 最终文件布局

第 20 章结束时：

```text
myfirst.c           - Main driver
myfirst.h           - Shared declarations
myfirst_hw.c        - Ch16 hardware access layer
myfirst_hw_pci.c    - Ch18 hardware-layer extension
myfirst_hw.h        - Register map
myfirst_sim.c       - Ch17 simulation backend
myfirst_sim.h       - Simulation interface
myfirst_pci.c       - Ch18 PCI attach
myfirst_pci.h       - PCI declarations
myfirst_intr.c      - Ch19 interrupt handler (legacy + filter+task)
myfirst_intr.h      - Ch19 interrupt interface + ICSR macros
myfirst_msix.c      - Ch20 MSI/MSI-X multi-vector layer (NEW)
myfirst_msix.h      - Ch20 multi-vector interface (NEW)
myfirst_sync.h      - Part 3 synchronisation
cbuf.c / cbuf.h     - Ch10 circular buffer
Makefile            - kmod build
HARDWARE.md, LOCKING.md, SIMULATION.md, PCI.md, INTERRUPTS.md, MSIX.md (NEW)
```

`myfirst_msix.c` 和 `myfirst_msix.h` 是新文件。`MSIX.md` 是新文档。第19章的 `myfirst_intr.c` 保留；它现在处理 legacy-INTx 回退，而 `myfirst_msix.c` 处理 MSI 和 MSI-X 路径。

### myfirst_msix.h头文件

```c
#ifndef _MYFIRST_MSIX_H_
#define _MYFIRST_MSIX_H_

#include <sys/taskqueue.h>

struct myfirst_softc;

enum myfirst_intr_mode {
	MYFIRST_INTR_LEGACY = 0,
	MYFIRST_INTR_MSI = 1,
	MYFIRST_INTR_MSIX = 2,
};

enum myfirst_vector_id {
	MYFIRST_VECTOR_ADMIN = 0,
	MYFIRST_VECTOR_RX,
	MYFIRST_VECTOR_TX,
	MYFIRST_MAX_VECTORS
};

struct myfirst_vector {
	struct resource		*irq_res;
	int			 irq_rid;
	void			*intr_cookie;
	enum myfirst_vector_id	 id;
	struct myfirst_softc	*sc;
	uint64_t		 fire_count;
	uint64_t		 stray_count;
	const char		*name;
	driver_filter_t		*filter;
	struct task		 task;
	bool			 has_task;
};

int  myfirst_msix_setup(struct myfirst_softc *sc);
void myfirst_msix_teardown(struct myfirst_softc *sc);
void myfirst_msix_add_sysctls(struct myfirst_softc *sc);

#endif /* _MYFIRST_MSIX_H_ */
```

公共 API 是三个函数：设置、拆除、add_sysctls。枚举类型和每向量结构被导出，以便 `myfirst.h` 可以包含它们，softc 可以有每向量数组。

### 完整Makefile

```makefile
# Makefile for the Chapter 20 myfirst driver.

KMOD=  myfirst
SRCS=  myfirst.c \
       myfirst_hw.c myfirst_hw_pci.c \
       myfirst_sim.c \
       myfirst_pci.c \
       myfirst_intr.c \
       myfirst_msix.c \
       cbuf.c

CFLAGS+= -DMYFIRST_VERSION_STRING=\"1.3-msi\"

.include <bsd.kmod.mk>
```

SRCS 列表中增加一个源文件；版本字符串升级。

### 版本字符串

`1.2-intr` 到 `1.3-msi`。版本升级反映了一项重要的能力添加：多向量中断处理。次版本号升级是合适的；用户可见的接口（cdev）未改变。

### MSIX.md文档

新文档位于源代码旁边：

```markdown
# myfirst 驱动程序中的 MSI 和 MSI-X 支持

## 总结

驱动程序按顺序探测设备的 MSI-X、MSI 和传统 INTx 能力，并使用第一个成功分配的方式。驱动程序的中断计数器、数据路径和 cdev 行为与最终使用的层级无关。

## 设置顺序

`myfirst_msix_setup()` 尝试三个层级：

1. MSI-X，使用 MYFIRST_MAX_VECTORS (3) 个向量。成功时：
   - 在 rid=1, 2, 3 分配每向量 IRQ 资源。
   - 为每个向量注册不同的过滤器函数。
   - 用每向量名称调用 bus_describe_intr。
   - 将每个向量绑定到 CPU（轮询或 NUMA 感知）。
2. MSI，使用 MYFIRST_MAX_VECTORS 个向量。相同的每向量模式。
3. 遗留 INTx，使用单个处理程序覆盖 rid=0 的所有三类事件。

## 每向量分配

| 向量 | 目的 | 处理 | 内联/延迟 |
|------|------|------|-----------|
| 0    | admin | INTR_STATUS.ERROR | 内联 |
| 1    | rx    | INTR_STATUS.DATA_AV | 延迟 (任务) |
| 2    | tx    | INTR_STATUS.COMPLETE | 内联 |

在 MSI-X 上，每个向量有自己的 intr_event、自己的 CPU 亲和性（通过 bus_bind_intr）和自己的 bus_describe_intr 标签（"admin"、"rx"、"tx"）。在 MSI 上，驱动程序获得单个向量并回退到第 19 章的单处理程序模式，因为 MSI 要求向量计数为 2 的幂次方（pci_alloc_msi 以 EINVAL 拒绝 count=3）。在遗留 INTx 上，单个过滤器覆盖所有三个位。

## sysctl

- `dev.myfirst.N.intr_mode`: 0 (遗留), 1 (MSI), 2 (MSI-X)。
- `dev.myfirst.N.vec{0,1,2}_fire_count`: 每向量触发计数。
- `dev.myfirst.N.vec{0,1,2}_stray_count`: 每向量杂散计数。
- `dev.myfirst.N.intr_simulate_admin`, `.intr_simulate_rx`,
  `.intr_simulate_tx`: 模拟每向量中断。

## 拆除顺序

1. 在设备端禁用中断（清除 INTR_MASK）。
2. 每向量逆序：bus_teardown_intr, bus_release_resource。
3. 排空并释放每向量任务和任务队列。
4. 如果 intr_mode 是 MSI 或 MSI-X，调用一次 pci_release_msi。

## dmesg 摘要行

附加时单行显示：

- "interrupt mode: MSI-X, 3 vectors"
- "interrupt mode: MSI, 1 vector (single-handler fallback)"
- "interrupt mode: legacy INTx (1 handler for all events)"

## 已知限制

- MYFIRST_MAX_VECTORS 硬编码为 3。适配分配计数的动态设计是第 20 章的挑战练习。
- CPU 绑定是轮询的。通过 bus_get_cpus 的 NUMA 感知绑定是挑战练习。
- DMA 是第 21 章内容。
- iflib 集成超出了范围。

## 另见

- `INTERRUPTS.md` 关于第 19 章遗留路径详情。
- `HARDWARE.md` 关于寄存器映射。
- `LOCKING.md` 关于完整锁规则。
- `PCI.md` 关于 PCI attach 行为。
```

该文档为未来的读者在一页中呈现了多向量设计的完整图景。

### 回归测试通过


第 20 章的回归测试是第 19 章的超集：

1. 干净编译。`make` 生成 `myfirst.ko` 无警告。
2. 加载。`kldload` 显示 attach 横幅包括 `interrupt mode:` 行。
3. 验证模式。`sysctl dev.myfirst.0.intr_mode` 返回 0、1 或 2（取决于 guest）。
4. 每向量 attach。`vmstat -i | grep myfirst` 显示 N 行（legacy 为 1，MSI 或 MSI-X 为 3）。
5. 每向量描述。每个条目有正确的名称（`admin`、`rx`、`tx` 或 `legacy`）。
6. 模拟中断。每个向量的计数器独立计数。
7. 任务运行。RX 向量的模拟中断驱动 `intr_task_invocations`。
8. 干净分离。`devctl detach myfirst0` 拆除所有向量。
9. 卸载后加载。第二次 `kldload` 使用相同的层级（测试 `pci_release_msi` 是否工作）。
10. vmstat -m 不显示泄漏。卸载后，没有 myfirst 分配残留。

回归脚本运行所有十个检查。在带有 virtio-rng-pci 的 QEMU 上，测试练习 MSI-X 路径；在带有 virtio-rnd 的 bhyve 上，测试练习 legacy-INTx 回退。驱动程序的回退阶梯确保它在两者上都能工作。

### 重构完成的内容

在第 20 章开始时，`1.2-intr` 的 `myfirst` 在 legacy 线上有一个中断处理程序。在第 20 章结束时，`1.3-msi` 的 `myfirst` 有三层回退阶梯（MSI-X → MSI → legacy）、MSI 或 MSI-X 上三个每向量过滤器、每向量计数器、每向量 CPU 亲和性和单一干净的拆除路径。驱动程序的文件数增加了两个；文档增加了一个；功能能力大幅增长。

代码具有 FreeBSD 的典型特征。第一次打开驱动程序的贡献者会发现熟悉的结构：每向量数组、每向量过滤器函数、三层设置阶梯、每个向量的 bus_describe_intr、拆除时的单个 `pci_release_msi`。这些模式出现在每个多队列 FreeBSD 驱动程序中。

### 总结 Section 8

重构遵循既定的形式：新层的新文件、导出公共接口的新头文件、解释行为的新文档、版本提升、回归通过。第 20 章的层是多向量中断处理；第 19 章保持单向量 legacy 回退。它们一起形成了驱动程序所需的完整中断故事。

第 20 章的教学主体已完成。接下来是实验、挑战、故障排除、总结以及通往第 21 章的桥梁。



## 一起阅读真实驱动：virtio_pci.c

在实验之前，简短地浏览一个广泛使用 MSI-X 的真实 FreeBSD 驱动程序。`/usr/src/sys/dev/virtio/pci/virtio_pci.c` 是 legacy 和现代 virtio-PCI 传输的共享核心；它保存了每个 virtio 设备使用的中断分配阶梯。在第 20 章之后阅读此文件是模式识别的简短练习；中断部分几乎所有的内容都映射到第 20 章刚刚教授的内容。

### 分配阶梯

`virtio_pci.c` 有一个名为 `vtpci_alloc_intr_resources` 的辅助函数（确切名称因 FreeBSD 版本略有不同）。其结构为：

```c
static int
vtpci_alloc_intr_resources(struct vtpci_common *cn)
{
	int error;

	/* Tier 0: MSI-X. */
	error = vtpci_alloc_msix(cn, nvectors);
	if (error == 0) {
		cn->vtpci_flags |= VTPCI_FLAG_MSIX;
		return (0);
	}

	/* Tier 1: MSI. */
	error = vtpci_alloc_msi(cn);
	if (error == 0) {
		cn->vtpci_flags |= VTPCI_FLAG_MSI;
		return (0);
	}

	/* Tier 2: legacy INTx. */
	return (vtpci_alloc_intx(cn));
}
```

这三个层级正是第 20 章的阶梯。每个层级成功时在公共状态上设置标志并返回 0。失败时尝试下一个层级。

### MSI-X分配辅助函数

`vtpci_alloc_msix` 查询计数，决定请求多少向量（基于设备使用的虚拟队列数量），并调用 `pci_alloc_msix`：

```c
static int
vtpci_alloc_msix(struct vtpci_common *cn, int nvectors)
{
	int error, count;

	if (pci_msix_count(cn->vtpci_dev) < nvectors)
		return (ENOSPC);

	count = nvectors;
	error = pci_alloc_msix(cn->vtpci_dev, &count);
	if (error != 0)
		return (error);
	if (count != nvectors) {
		pci_release_msi(cn->vtpci_dev);
		return (ENXIO);
	}
	return (0);
}
```

模式：检查计数、分配、验证分配是否与请求匹配、不匹配则释放。如果设备公布的向量少于所需，立即返回 `ENOSPC`。如果 `pci_alloc_msix` 分配的计数少于请求，代码释放并返回 `ENXIO`。

第 20 章的代码遵循此确切逻辑（第 5 节展示了完整版本）。

### 每向量资源分配

一旦分配了 MSI-X，virtio 遍历向量并为每个向量注册处理程序：

```c
static int
vtpci_register_msix_vectors(struct vtpci_common *cn)
{
	int i, rid, error;

	rid = 1;  /* MSI-X vectors start at rid 1 */
	for (i = 0; i < cn->vtpci_num_vectors; i++) {
		cn->vtpci_vectors[i].res = bus_alloc_resource_any(
		    cn->vtpci_dev, SYS_RES_IRQ, &rid, RF_ACTIVE);
		if (cn->vtpci_vectors[i].res == NULL)
			/* ... fail ... */;
		rid++;
		error = bus_setup_intr(cn->vtpci_dev,
		    cn->vtpci_vectors[i].res,
		    INTR_TYPE_MISC | INTR_MPSAFE,
		    NULL, vtpci_vq_handler,
		    &cn->vtpci_vectors[i], &cn->vtpci_vectors[i].cookie);
		if (error != 0)
			/* ... fail ... */;
	}
	return (0);
}
```

有两件事与第20章匹配：

- 第一个向量的 `rid = 1`，每个向量递增。
- 过滤器（此处为 `NULL`）和处理程序（`vtpci_vq_handler`）模式。注意 virtio 使用仅 ithread 的处理程序（filter=NULL），而不是过滤器加任务的管道。这是更简单的选项，适用于 virtio 的每向量工作。

`vtpci_vq_handler` 函数是每向量的工作线程。每个向量获得自己的参数（`&cn->vtpci_vectors[i]`），处理程序使用该参数来识别要服务的虚拟队列。

### 拆除

Virtio 的拆除遵循第20章的模式：

```c
static void
vtpci_release_intr_resources(struct vtpci_common *cn)
{
	int i;

	for (i = 0; i < cn->vtpci_num_vectors; i++) {
		if (cn->vtpci_vectors[i].cookie != NULL) {
			bus_teardown_intr(cn->vtpci_dev,
			    cn->vtpci_vectors[i].res,
			    cn->vtpci_vectors[i].cookie);
		}
		if (cn->vtpci_vectors[i].res != NULL) {
			bus_release_resource(cn->vtpci_dev, SYS_RES_IRQ,
			    rman_get_rid(cn->vtpci_vectors[i].res),
			    cn->vtpci_vectors[i].res);
		}
	}

	if (cn->vtpci_flags & (VTPCI_FLAG_MSI | VTPCI_FLAG_MSIX))
		pci_release_msi(cn->vtpci_dev);
}
```

每向量拆除（bus_teardown_intr + bus_release_resource），最后单个 `pci_release_msi`。该顺序与第20章的 `myfirst_msix_teardown` 匹配。

一个值得注意的细节：virtio 使用 `rman_get_rid` 从资源中恢复 rid，而不是单独存储它。第20章的驱动程序将 rid 存储在每向量结构中；两种方法都可以，但存储方法更清晰且更易于调试。

### Virtio演示教学的内容

三个教训直接迁移到第20章的设计：

1. **三层回退阶梯是标准模式**。每个想在多种硬件上工作的驱动程序都以相同方式实现它。
2. **每向量资源管理使用从1开始递增的rid**。这在FreeBSD的PCI基础设施中是通用的。
3. **`pci_release_msi` 只调用一次，无论向量数量多少**。每向量拆除释放IRQ资源；设备级释放处理MSI状态。

能够端到端理解 `vtpci_alloc_intr_resources` 的读者已经内化了第20章的词汇。对于更丰富的示例，`/usr/src/sys/dev/nvme/nvme_ctrlr.c` 展示了大规模下的相同模式，带有一个管理向量加上最多 `NCPU` 个I/O向量。



## 深入了解向量到 CPU 放置

第5节简要介绍了 `bus_bind_intr(9)`。本节深入探讨为什么CPU放置很重要，真实驱动如何选择CPU，以及权衡是什么。

### NUMA图景

在单插槽系统上，所有CPU共享单个内存控制器和单个缓存层次结构。CPU之间的放置仅对缓存亲和性重要（处理程序的代码和数据在上次运行它的CPU上会是热的）。"CPU 0"和"CPU 3"之间的性能差异很小。

在多插槽NUMA系统上，情况发生了变化。每个插槽有自己的内存控制器、自己的L3缓存和自己的PCIe根复合体。连接到插槽0的PCIe设备位于该插槽的根复合体上；其寄存器通过内存映射到由插槽0控制器处理的地址范围。来自该设备的中断触发；处理程序读取`INTR_STATUS`；读取访问设备的BAR，该BAR位于插槽0上；运行处理程序的CPU必须在插槽0上，否则读取会跨插槽间互连。

插槽间互连（在Intel系统上：UPI或更早的QPI；在AMD上：Infinity Fabric）比插槽内缓存访问慢得多。在错误插槽上运行的处理程序看到寄存器读取耗时数十纳秒而不是个位数；数据位于错误插槽上的接收队列在数据包到达用户空间的路上每个包都要跨越互连。

放置良好的向量将处理程序的工作保持在设备所在的插槽上。

### 查询NUMA局部性

FreeBSD 通过 `bus_get_cpus(9)` 向驱动程序暴露 NUMA 拓扑。API：

```c
int bus_get_cpus(device_t dev, enum cpu_sets op, size_t setsize,
    struct _cpuset *cpuset);
```

`op` 参数选择要查询的集合：

- `LOCAL_CPUS`：与设备在同一 NUMA 域中的 CPU。
- `INTR_CPUS`：适合处理设备中断的 CPU（通常是 `LOCAL_CPUS`，除非操作员排除了某些 CPU）。

`cpuset` 是输出参数；成功时，它包含所查询集合中 CPU 的位图。

Example use:

```c
cpuset_t local_cpus;
int num_local;

if (bus_get_cpus(sc->dev, INTR_CPUS, sizeof(local_cpus),
    &local_cpus) == 0) {
	num_local = CPU_COUNT(&local_cpus);
	device_printf(sc->dev, "device has %d interrupt-suitable CPUs\n",
	    num_local);
}
```

驱动程序使用 `CPU_FFS(&local_cpus)` 查找集合中的第一个 CPU，使用 `CPU_CLR(cpu, &local_cpus)` 标记为已使用，并进行迭代。

A round-robin bind that respects NUMA locality:

```c
static void
myfirst_msix_bind_vectors_numa(struct myfirst_softc *sc)
{
	cpuset_t local_cpus;
	int cpu, i;

	if (bus_get_cpus(sc->dev, INTR_CPUS, sizeof(local_cpus),
	    &local_cpus) != 0) {
		/* No NUMA info; round-robin across all CPUs. */
		myfirst_msix_bind_vectors_roundrobin(sc);
		return;
	}

	if (CPU_EMPTY(&local_cpus))
		return;

	for (i = 0; i < sc->num_vectors; i++) {
		if (CPU_EMPTY(&local_cpus))
			bus_get_cpus(sc->dev, INTR_CPUS,
			    sizeof(local_cpus), &local_cpus);
		cpu = CPU_FFS(&local_cpus) - 1;  /* FFS returns 1-based */
		CPU_CLR(cpu, &local_cpus);
		(void)bus_bind_intr(sc->dev,
		    sc->vectors[i].irq_res, cpu);
	}
}
```

代码获取本地 CPU 集合，选择编号最小的 CPU，将向量 0 绑定到它，从集合中清除该 CPU，选择下一个最小的，将向量 1 绑定到它，以此类推。如果集合用尽（向量数多于本地 CPU 数），则刷新并继续。

第20章的驱动程序未包含此 NUMA 感知绑定；挑战练习要求读者添加它。

### 操作者视角

操作员可以使用 `cpuset` 覆盖内核的放置：

```sh
# Get current placement for IRQ 257.
sudo cpuset -g -x 257

# Bind IRQ 257 to CPU 3.
sudo cpuset -l 3 -x 257

# Bind to a set of CPUs (kernel picks one when the interrupt fires).
sudo cpuset -l 2,3 -x 257
```

这些命令覆盖驱动程序通过 `bus_bind_intr` 设置的任何放置。操作员可能会这样做，将关键中断从用户工作负载 CPU 上移开（用于实时应用），或将流量集中在特定 CPU 上（用于诊断目的）。

驱动程序的 `bus_bind_intr` 调用设置初始放置；操作员可以覆盖。一个行为良好的驱动程序设置合理的默认值并尊重操作员的更改（它会自动这样做，因为 `bus_bind_intr` 只是写入操作系统管理的 CPU 亲和性状态，操作员随后可以修改）。

### 测量效果

查看 NUMA 局部性价值的具体方法：在处理程序绑定到本地 CPU 时运行高中断率工作负载，然后绑定到远程 CPU，比较延迟。在双插槽系统上，远程 CPU 的处理程序每次中断通常需要 1.5 到 3 倍的时间（以 CPU 周期计）。

FreeBSD 的 DTrace 提供者可以测量这一点：

```sh
sudo dtrace -n '
fbt::myfirst_intr_filter:entry { self->ts = vtimestamp; }
fbt::myfirst_intr_filter:return /self->ts/ {
    @[cpu] = quantize(vtimestamp - self->ts);
    self->ts = 0;
}'
```

输出是过滤器延迟的每 CPU 直方图。读者可以在观察向量放置的同时运行此命令，确认延迟差异。

### 向量放置何时重要

- 高中断率（每向量每秒超过几千次）。
- 处理程序中较大的缓存行占用（处理程序的代码和数据占据多个缓存行）。
- 在同一插槽上具有下游处理流程的共享接收路径。
- 具有多个插槽且 PCIe 设备连接到特定插槽的 NUMA 系统。

### 向量放置何时不重要

- 低速率中断（每秒几十次或更少）。
- 单插槽系统。
- 执行最少工作的处理程序（第20章的管理向量）。
- 无论如何都在单个 CPU 上运行的驱动程序（单 CPU 嵌入式系统）。

第20章的驱动程序在正常测试中属于"关系不大"的类别，但本章教授的模式直接迁移到确实需要它的驱动程序。



## 深入了解向量分配策略

第6节展示了固定分配模式（向量0 = admin，1 = rx，2 = tx）。本节探讨真实驱动程序使用的其他分配策略。

### 每队列一向量

最简单且最常见的策略。每个队列（接收队列、发送队列、管理队列等）有自己专用的向量。驱动程序为 `N` 个接收队列、`M` 个发送队列和 1 个管理队列分配 `N+M+1` 个向量。

优点：
- 每向量处理程序逻辑简单。
- 每个队列的中断率独立。
- CPU 亲和性是每队列的（易于固定到 NUMA 本地 CPU）。

缺点：
- 对于具有许多队列的驱动程序，消耗很多向量。
- 每个队列的 ithread 在低速率队列上增加了开销。

这是 `nvme(4)` 使用的模式。

### 合并RX+TX向量

一些驱动程序将单个队列对的 RX 和 TX 合并到一个向量中。具有 8 个队列对的 NIC 将使用 8 个合并向量加上几个用于管理。当向量触发时，过滤器检查 RX 和 TX 状态位并相应地分发。

优点：
- 每队列对向量数量减半。
- 同一队列对的 RX 和 TX 往往彼此 NUMA 本地（它们共享相同的描述符环内存）。

缺点：
- 过滤器稍微复杂。
- RX 和 TX 在负载下可能相互干扰（RX 突发填满处理程序时间，延迟 TX 完成）。

这是一种中间设计，被一些消费级 NIC 使用。

### 所有队列共用一个向量

一些资源受限的设备（低成本 NIC、小型嵌入式设备）总共只有一两个 MSI-X 向量。驱动程序对所有队列使用单个向量，并根据状态寄存器分发到每个队列。

优点：
- 在向量少的硬件上工作。
- 分配简单。

缺点：
- 没有每队列亲和性。
- 过滤器要做更多工作来决定分发什么。

这是非常低端硬件上驱动程序使用的模式。

### 动态每 CPU 分配

一个巧妙的设计：为每个 CPU 分配一个向量，并动态地将队列分配给向量。一个 RX 队列一次由一个 CPU "拥有"；它在该 CPU 的向量上处理。如果工作负载变化，驱动程序可以将队列重新映射到不同的 CPU。

优点：
- 最佳的每 CPU 缓存亲和性。
- 适应工作负载变化。

缺点：
- 复杂的分配和重新映射逻辑。
- 不容易推理。

一些高端 NIC 驱动程序（Mellanox ConnectX 系列、Intel 800 系列）使用此变体。

### 第 20 章的策略

第20章的驱动程序使用三个向量的固定分配策略。这是说明多向量设计的最简单策略，不涉及 NUMA 细节或动态重新映射。真实驱动程序通常从这种设计开始，并根据需求演进到更复杂的模式。

一个挑战练习要求读者实现动态每 CPU 分配策略作为扩展。



## 深入了解中断调节和合并

一个与 MSI-X 相邻的概念值得简单提及。现代高吞吐量设备通常支持**中断调节**或**合并**：设备缓冲事件（传入的数据包、完成通知），并在时间阈值或计数阈值处为多个事件触发单个中断。

### 为什么调节重要

每秒接收一千万数据包的 NIC 如果每个数据包触发一个中断，将触发一千万个中断。这太多了；CPU 将把所有时间花在进入和退出中断处理程序上。解决方案是批处理：NIC 每 50 微秒触发一次中断，在这 50 微秒内 NIC 累积到达的任何数据包。处理程序一次性处理所有累积的数据包。

合并以延迟换取吞吐量：每个数据包交付到用户空间最多延迟 50 微秒，但 CPU 以可管理的中断率每秒处理数百万数据包。

### 驱动程序如何控制调节

机制是设备特定的。常见形式：

- **基于时间：**设备在配置的时间间隔后触发（例如 50 微秒）。
- **基于计数：**设备在 N 个事件后触发（例如 16 个数据包）。
- **组合：**先达到哪个阈值就触发哪个。
- **自适应：**设备（或驱动程序）根据观察到的速率调整阈值。

驱动程序通常通过设备寄存器编程阈值。MSI-X 机制本身不提供调节；它是一个与 MSI-X 配合使用的设备特性，因为 MSI-X 允许每向量分配。

### 第 20 章的驱动程序不做调节

第 20 章的驱动程序没有调节。每个模拟中断产生一次过滤器调用。在真实硬件上，这在高速率下会是个问题；在实验室中没问题。

像 `em(4)`、`ix(4)`、`ixl(4)` 和 `mgb(4)` 这样的真实驱动程序都有调节参数。`sysctl` 接口将它们暴露为可调参数：

```sh
sysctl dev.em.0 | grep itr
```

将本章驱动程序适配到真实设备的读者应研究该设备的调节控制。该机制与 MSI-X 正交；两者结合提供高性能中断处理。



## 来自真实 FreeBSD 驱动程序的模式

对 `/usr/src/sys/dev/` 中出现的多向量模式的一次巡览。每个模式都是来自真实驱动程序的简短代码片段，附有说明其对第20章的指导意义的注释。

### 模式：nvme(4) 管理 + I/O 向量分离

`/usr/src/sys/dev/nvme/nvme_ctrlr.c` has the canonical admin-plus-N pattern:

```c
/* Allocate one vector for admin + N for I/O. */
num_trackers = MAX(1, MIN(mp_ncpus, ctrlr->max_io_queues));
num_vectors_requested = num_trackers + 1;  /* +1 for admin */
num_vectors_allocated = num_vectors_requested;
pci_alloc_msix(ctrlr->dev, &num_vectors_allocated);

/* Admin queue uses vector 0 (rid 1). */
ctrlr->adminq.intr_rid = 1;
ctrlr->adminq.res = bus_alloc_resource_any(ctrlr->dev, SYS_RES_IRQ,
    &ctrlr->adminq.intr_rid, RF_ACTIVE);
bus_setup_intr(ctrlr->dev, ctrlr->adminq.res,
    INTR_TYPE_MISC | INTR_MPSAFE,
    NULL, nvme_qpair_msix_handler, &ctrlr->adminq, &ctrlr->adminq.tag);

/* I/O queues use vectors 1..N (rid 2..N+1). */
for (i = 0; i < ctrlr->num_io_queues; i++) {
	ctrlr->ioq[i].intr_rid = i + 2;
	/* same pattern ... */
}
```

为什么重要：管理加 N 模式是一个向量处理不频繁的高优先级工作（错误、异步事件），N 个向量处理速率受限的每队列工作时的正确选择。第20章的管理/rx/tx 拆分是此模式的微型版本。

### 模式：ixgbe 的队列对向量

`/usr/src/sys/dev/ixgbe/ix_txrx.c` uses a queue-pair design where each vector handles both the RX and TX of one queue pair:

```c
/* One vector per queue pair + 1 for link. */
for (i = 0; i < num_qpairs; i++) {
	que[i].rid = i + 1;
	/* Filter checks both RX and TX status bits and dispatches. */
	bus_setup_intr(..., ixgbe_msix_que, &que[i], ...);
}
/* Link-state vector is the last one. */
link.rid = num_qpairs + 1;
bus_setup_intr(..., ixgbe_msix_link, sc, ...);
```

为什么重要：合并的每队列对 RX+TX 设计在不牺牲每队列亲和性的情况下将向量计数减半。适用于设备具有许多队列但向量很少的情况。

### 模式：virtio_pci 的每虚拟队列向量

`/usr/src/sys/dev/virtio/pci/virtio_pci.c` has one vector per virtqueue:

```c
int nvectors = ... /* count of virtqueues + 1 for config */;
pci_alloc_msix(dev, &nvectors);
for (i = 0; i < nvectors; i++) {
	vec[i].rid = i + 1;
	/* Each vector gets the per-virtqueue data as its arg. */
	bus_setup_intr(dev, vec[i].res, ..., virtio_vq_intr, &vec[i], ...);
}
```

为什么重要：virtio 的每虚拟队列分配是任何半虚拟化设备的模型。向量数等于虚拟队列数加管理/配置。

### 模式：ahci 的每端口向量

`/usr/src/sys/dev/ahci/ahci_pci.c` uses one vector per SATA port:

```c
for (i = 0; i < ahci->nports; i++) {
	ahci->ports[i].rid = i + 1;
	/* ... */
}
```

为什么重要：存储控制器通常使用每端口向量分配，以便不同端口上的 I/O 完成可以在不同 CPU 上并发处理。

### 模式：iflib 的隐藏向量管理

使用 `iflib(9)` 的驱动程序（例如 `em(4)`、`igc(4)`、`ix(4)`、`ixl(4)`、`mgb(4)`）不直接管理向量。相反，它们向 iflib 的注册表注册每队列处理函数，由 iflib 进行分配和绑定：

```c
static struct if_shared_ctx em_sctx_init = {
	/* ... */
	.isc_driver = &em_if_driver,
	.isc_tx_maxsize = EM_TSO_SIZE,
	/* ... */
};

static int
em_if_msix_intr_assign(if_ctx_t ctx, int msix)
{
	struct e1000_softc *sc = iflib_get_softc(ctx);
	int error, rid, i, vector = 0;

	/* iflib has already called pci_alloc_msix; sc knows the count. */
	for (i = 0; i < sc->rx_num_queues; i++, vector++) {
		rid = vector + 1;
		error = iflib_irq_alloc_generic(ctx, ..., rid, IFLIB_INTR_RXTX,
		    em_msix_que, ...);
	}
	return (0);
}
```

为什么重要：iflib 在干净的 API 后面抽象了 MSI-X 分配和每队列绑定。使用 iflib 的驱动程序比裸 MSI-X 驱动程序更简单，但放弃了一些灵活性。iflib 模式是新的 FreeBSD 网络驱动程序的正确选择；裸 MSI-X 模式是非网络设备或 iflib 不合适时的正确选择。

### 这些模式教会了什么

所有这些驱动程序都遵循第20章教授的相同结构模式：

1. 查询向量计数。
2. 分配向量。
3. 对每个向量：在 rid=i+1 分配 IRQ 资源，注册处理程序，描述。
4. 将向量绑定到 CPU。
5. 拆除时：逆序每向量拆除，然后 `pci_release_msi`。

驱动程序之间的差异在于：

- 向量数量（1 个、少量、几十个或数百个）。
- 向量如何分配（管理+N、队列对、每端口、每虚拟队列）。
- iflib 是否处理分配。
- 每个过滤器函数做什么（管理 vs 数据路径）。

拥有第20章词汇的读者可以立即识别这些差异。



## 性能观察：测量 MSI-X 的收益

本节将本章的性能声明建立在具体的测量基础上。

### 测试设置

假设您在 QEMU 上使用 `virtio-rng-pci`（因此 MSI-X 处于活动状态）和多 CPU guest 运行第20章的驱动程序。`intr_simulate_rx` sysctl 让您从用户空间循环触发中断：

```sh
# In one shell, drive simulated RX interrupts as fast as possible.
while true; do
    sudo sysctl dev.myfirst.0.intr_simulate_rx=1 >/dev/null 2>&1
done
```

### 使用 DTrace 测量

在另一个 shell 中，测量过滤器的每次调用 CPU 时间及其运行的 CPU：

```sh
sudo dtrace -n '
fbt::myfirst_rx_filter:entry { self->ts = vtimestamp; self->c = cpu; }
fbt::myfirst_rx_filter:return /self->ts/ {
    @lat[self->c] = quantize(vtimestamp - self->ts);
    self->ts = 0;
    self->c = 0;
}'
```

输出是过滤器延迟的每 CPU 直方图。如果 `bus_bind_intr` 将 RX 向量放置在 CPU 1 上，直方图应显示所有调用都在 CPU 1 上，延迟在数百纳秒到个位数微秒之间。

### 结果显示什么

在放置良好的 MSI-X 向量上：

- 每次调用都在同一个 CPU（绑定的 CPU）上。
- 延迟一致地短（热缓存行保持在一个 CPU 上）。
- 没有跨 CPU 缓存抖动。

在遗留 INTx 共享线上：

- 调用分散到各个 CPU（内核随机路由）。
- 延迟更加多变（每个新 CPU 上的冷缓存行）。
- 跨 CPU 缓存流量出现在性能计数器中。

差异可以以每次调用纳秒为单位测量。对于每秒处理几百次中断的驱动程序，差异不可见。对于每秒处理一百万次中断的驱动程序，差异就是"工作"和"不工作"之间的差异。

### 一般性教训

第20章的机制对于低速率驱动程序来说是过度的。对于高速率驱动程序来说它是必需的。本章教授的模式从"每秒执行一百次中断的演示驱动程序"扩展到"执行一千万次的生成 NIC"。知道特定驱动程序在哪个尺度上，决定了第20章的建议在实践中有多少重要性。



## 深入了解多向量驱动程序的 sysctl 树设计

第20章的驱动程序将其每向量计数器暴露为扁平 sysctl（`vec0_fire_count`、`vec1_fire_count`、`vec2_fire_count`）。对于具有许多向量的驱动程序，扁平命名空间变得笨拙。本节展示如何使用 `SYSCTL_ADD_NODE` 构建每向量 sysctl 树。

### 扁平与树形的权衡

扁平命名空间（第20章使用的）：

```text
dev.myfirst.0.vec0_fire_count: 42
dev.myfirst.0.vec1_fire_count: 9876
dev.myfirst.0.vec2_fire_count: 4523
dev.myfirst.0.vec0_stray_count: 0
dev.myfirst.0.vec1_stray_count: 0
dev.myfirst.0.vec2_stray_count: 0
```

优点：简单，无需 `SYSCTL_ADD_NODE` 调用。
缺点：顶层有许多同级节点；没有分组。

树形命名空间：

```text
dev.myfirst.0.vec.admin.fire_count: 42
dev.myfirst.0.vec.admin.stray_count: 0
dev.myfirst.0.vec.rx.fire_count: 9876
dev.myfirst.0.vec.rx.stray_count: 0
dev.myfirst.0.vec.tx.fire_count: 4523
dev.myfirst.0.vec.tx.stray_count: 0
```

优点：对每向量状态进行分组；可扩展到许多向量；使用名称而非编号。
缺点：需要更多代码来设置。

### 构建树的代码

```c
void
myfirst_msix_add_sysctls(struct myfirst_softc *sc)
{
	struct sysctl_ctx_list *ctx = &sc->sysctl_ctx;
	struct sysctl_oid *parent = sc->sysctl_tree;
	struct sysctl_oid *vec_node;
	struct sysctl_oid *per_vec_node;
	int i;

	/* Create the "vec" parent node. */
	vec_node = SYSCTL_ADD_NODE(ctx, SYSCTL_CHILDREN(parent),
	    OID_AUTO, "vec", CTLFLAG_RD, NULL,
	    "Per-vector interrupt statistics");

	for (i = 0; i < MYFIRST_MAX_VECTORS; i++) {
		/* Create "vec.<name>" node. */
		per_vec_node = SYSCTL_ADD_NODE(ctx,
		    SYSCTL_CHILDREN(vec_node),
		    OID_AUTO, sc->vectors[i].name,
		    CTLFLAG_RD, NULL,
		    "Per-vector statistics");

		/* Add fire_count under it. */
		SYSCTL_ADD_U64(ctx, SYSCTL_CHILDREN(per_vec_node),
		    OID_AUTO, "fire_count", CTLFLAG_RD,
		    &sc->vectors[i].fire_count, 0,
		    "Times this vector's filter was called");

		/* Add stray_count. */
		SYSCTL_ADD_U64(ctx, SYSCTL_CHILDREN(per_vec_node),
		    OID_AUTO, "stray_count", CTLFLAG_RD,
		    &sc->vectors[i].stray_count, 0,
		    "Stray returns from this vector");

		/* Other per-vector fields... */
	}
}
```

`SYSCTL_ADD_NODE` 调用创建中间节点；随后的 `SYSCTL_ADD_U64` 调用在其下附加叶计数器。树形结构在 `sysctl` 输出中自动可见。

### 查询树

```sh
# Show all per-vector stats.
sysctl dev.myfirst.0.vec

# Show just the rx vector.
sysctl dev.myfirst.0.vec.rx

# Show only fire counts.
sysctl -n dev.myfirst.0.vec.admin.fire_count dev.myfirst.0.vec.rx.fire_count dev.myfirst.0.vec.tx.fire_count
```

树形结构使 sysctl 命名空间更易读，特别是对于具有许多向量的驱动程序（具有 32 个 I/O 队列的 NVMe，或具有 16 个队列对的 NIC）。

### 何时使用树

对于第20章的三向量驱动程序，扁平命名空间即可。对于具有八个或更多向量的驱动程序，树形结构变得有价值。编写生产驱动程序的读者应使用树形结构。

### 常见错误

- **泄露父节点。** `SYSCTL_ADD_NODE` 在 `sc->sysctl_ctx` 中注册节点；它与上下文的其他部分一起被释放。不需要显式释放。
- **忘记为处理程序参数传入 `NULL`。** `SYSCTL_ADD_NODE` 不是 CTLPROC；它是一个纯分组节点。处理程序参数为 `NULL`。
- **将错误父节点传递给子 `SYSCTL_ADD_*` 调用。** `vec_node` 的子节点应使用 `SYSCTL_CHILDREN(vec_node)`，而不是 `SYSCTL_CHILDREN(parent)`。

这种树形设计模式是暴露多向量状态的最干净方式。第20章的挑战练习建议将其作为扩展实现。



## 深入了解 MSI-X 设置中的错误路径

第3节和第5节展示了快乐路径的设置代码。本节介绍可能出错的情况以及如何诊断。

### 故障模式 1：pci_msix_count 返回 0

症状：由于计数为 0，跳过了 MSI-X 尝试。

原因：设备没有 MSI-X 能力，或 PCI 总线驱动程序未发现它。

修复：用 `pciconf -lvc` 确认。如果设备公布有 MSI-X 但 `pci_msix_count` 返回 0，设备的 PCI 配置已损坏或内核的探测未找到它；这很少见且难以从驱动程序修复。

### 故障模式 2：pci_alloc_msix 返回 EINVAL

症状：分配失败，返回 `EINVAL`。

原因：驱动程序请求的计数大于设备公布的最大值，或者请求为 0。

修复：将请求计数限制在 `pci_msix_count` 返回的值以内。始终至少请求 1。

### 故障模式 3：pci_alloc_msix 返回少于请求的向量数

症状：调用后的 `count` 小于请求值。

原因：内核的向量池部分耗尽；设备分配得到了剩余的任何向量。

修复：预先决定是接受、适应还是释放。第20章的驱动程序释放并回退到 MSI。

### 故障模式 4：bus_alloc_resource_any 为 MSI-X 向量返回 NULL

症状：`pci_alloc_msix` 成功后，每向量 IRQ 分配失败。

原因：
- rid 错误（使用 0 而不是 i+1）。
- 先前已释放（双重释放）。
- 总线层的 IRQ 资源耗尽。

修复：检查 rid 是否为 i+1。审计释放代码。记录错误。

### 故障模式 5：bus_setup_intr 为每向量处理程序返回 EINVAL

症状：`bus_setup_intr` 失败。

原因：
- 过滤器和 ithread 均为 NULL。
- 缺少 `INTR_TYPE_*` 标志。
- 先前已设置（双重设置）。

修复：确保过滤器参数非 NULL。包含 `INTR_TYPE_*` 标志。审计设置代码以防止双重注册。

### 故障模式 6：bus_bind_intr 返回错误

症状：`bus_bind_intr` 返回非零值。

原因：
- 平台不支持重新绑定。
- CPU 超出范围。
- 内核配置（NO_SMP、NUMA 禁用）。

修复：视为非致命（`device_printf` 警告并继续）。驱动程序在不绑定的情况下也能工作。

### 故障模式 7：vmstat -i 显示向量但计数器不增加

症状：内核看到向量但过滤器从未触发。

原因：
- 设备的 `INTR_MASK` 为零（第19章的问题）。
- 设备重置了其中断状态。
- 硬件 bug 或 bhyve/QEMU 配置问题。

修复：验证设备的 INTR_MASK。使用模拟中断 sysctl 确认过滤器是否至少能工作。

### 故障模式 8：第二次 kldload 回退到较低层级

症状：第一次加载使用 MSI-X；卸载；第二次加载使用 legacy 或 MSI。

原因：拆除时未调用 `pci_release_msi`。

修复：审计拆除路径。确保 `pci_release_msi` 在每条成功的分配路径上运行。

### 故障模式 9：WITNESS 在多向量设置时崩溃

症状：`WITNESS` 在每向量设置中报告锁顺序违规或"持锁睡眠"。

原因：在 `bus_setup_intr` 调用期间持有 `sc->mtx`。总线钩子可能睡眠，而在睡眠期间持有互斥锁是非法的。

修复：在调用 `bus_setup_intr` 之前释放 `sc->mtx`。如有需要之后重新获取。

### 故障模式 10：部分设置没有正确清理

症状：attach 失败；第二次 attach 失败，显示"资源正在使用"。

原因：部分失败的 goto 级联没有完全撤销。某些每向量状态残留。

修复：确保级联回退到失败的向量处停止，而不是越过它。一致地使用每向量辅助函数。



## 额外故障排除

第20章的读者可能遇到的一些额外故障模式。

### "QEMU guest 不暴露 MSI-X"

原因：QEMU 版本太旧，或 guest 使用 legacy virtio 启动。

修复：更新 QEMU 到较新版本。在 guest 中检查：

```sh
pciconf -lvc | grep -B 1 -A 2 'cap 11'
```

如果没有出现 `cap 11` 行，则 MSI-X 不可用。切换到 QEMU 的现代 virtio-rng-pci，使用 `-device virtio-rng-pci,disable-legacy=on`。

### "intr_simulate_rx 增加 fire_count 但任务从不运行"

原因：任务的 `TASK_INIT` 未被调用，或任务队列未启动。

修复：验证设置中调用了 `TASK_INIT(&vec->task, 0, myfirst_rx_task_fn, vec)`。验证调用了 `taskqueue_start_threads(&sc->intr_tq, ...)`。

### "每向量计数器增加但杂散计数成比例上升"

原因：过滤器的状态检查错误，或多个向量在同一个位上触发。

修复：每个过滤器应检查其特定的位。如果两个过滤器都试图处理 `DATA_AV`，一个会赢，另一个会看到杂散。

### "cpuset -g -x $irq 报告所有向量的掩码为 0"

原因：`bus_bind_intr` 未被调用，或它被调用时指定了 CPU 0（掩码 1）。

修复：如果故意不绑定，"mask 0"可能是平台特定的。如果尝试了绑定，检查 `bus_bind_intr` 的返回值。

### "驱动加载成功但 dmesg 没有显示 attach 横幅"

原因：`device_printf` 在横幅刷新之前调用，或横幅位于非常早期的启动缓冲区中。

修复：`dmesg -a` 显示完整消息缓冲区。验证 `dmesg -a | grep myfirst`。

### "Detach 在多向量设置后挂起"

原因：拆除尝试继续时，向量的处理程序仍在运行。`bus_teardown_intr` 阻塞等待它完成。

修复：确保在 `bus_teardown_intr` *之前*清除设备的 `INTR_MASK`，这样不会有新的处理程序被分发。确保过滤器不会永远循环；保持短运行时间纪律。

### "pci_alloc_msix 成功但只有部分向量触发"

原因：设备实际上没有在其应有的向量上发出信号。可能是驱动程序 bug（忘记启用）或设备问题。

修复：使用模拟中断 sysctl 确认每个向量的过滤器是否工作。如果模拟路径工作但真实事件不触发向量，问题出在设备端。



## 实例演练：追踪事件通过三个层级

为了使回退阶梯具体化，以下是在三个层级上追踪同一事件（模拟的 DATA_AV 中断）的完整过程。

### 第 3 层：遗留 INTx

在带有 virtio-rnd（不暴露 MSI-X）的 bhyve 上，驱动程序回退到在 rid 0 处有一个处理程序的 legacy INTx。

1. 用户运行 `sudo sysctl dev.myfirst.0.intr_simulate_admin=1`（或 `intr_simulate_rx=1` 等）。
2. sysctl 处理程序获取 `sc->mtx`，将位写入 INTR_STATUS，释放，调用过滤器。
3. 单个 `myfirst_intr_filter`（来自第19章）运行。它读取 INTR_STATUS，看到该位，确认，并将任务入队（对于 DATA_AV）或内联处理（对于 ERROR/COMPLETE）。
4. `intr_count` 递增。
5. 在 legacy 上，只有一个向量，因此所有三个模拟中断 sysctl 通过同一个过滤器。

观察结果：
- `sysctl dev.myfirst.0.intr_mode` 返回 0。
- `vmstat -i | grep myfirst` 显示一行。
- 每向量计数器不存在（legacy 模式使用第19章的计数器）。

### 第 2 层：MSI

在支持 MSI 但不支持 MSI-X 的系统上，驱动程序分配单个 MSI 向量。MSI 要求向量计数为 2 的幂，因此驱动程序不能在此请求 3；它请求 1 并使用第19章的单处理程序模式。

1. 用户运行 `sudo sysctl dev.myfirst.0.intr_simulate_admin=1`（或 `intr_simulate_rx=1`，或 `intr_simulate_tx=4`）。
2. 因为在 MSI 层级上只设置了一个向量，所有三个每向量模拟 sysctl 都通过第19章的同一个 `myfirst_intr_filter` 路由。
3. 过滤器读取 INTR_STATUS，看到该位，确认，然后内联处理或将任务入队。

观察结果：
- `sysctl dev.myfirst.0.intr_mode` 返回 1。
- `vmstat -i | grep myfirst` 显示一行（rid=1 处的单个 MSI 处理程序，标记为"msi"）。
- 槽位 1 和 2 的每向量计数器保持为 0，因为只有槽位 0 在使用；第19章的全局计数器（`intr_count`、`intr_data_av_count` 等）发生变化。

### 第 1 层：MSI-X

在带有 virtio-rng-pci 的 QEMU 上，驱动程序分配具有 3 个向量的 MSI-X，每个向量绑定到一个 CPU。

1. 用户运行 `sudo sysctl dev.myfirst.0.intr_simulate_rx=1`。
2. sysctl 直接调用 rx 过滤器（模拟路径不经过硬件）。
3. `myfirst_rx_filter` 运行（在调用 sysctl 的任何 CPU 上，因为模拟不经过内核的中断分发）。
4. 计数器递增；任务运行。

观察结果：
- `sysctl dev.myfirst.0.intr_mode` 返回 2。
- `vmstat -i | grep myfirst` 显示三行；每行有不同的 IRQ 号。
- 每个 IRQ 的 `cpuset -g -x <irq>` 显示不同的 CPU 掩码。

真实的（非模拟）MSI-X 中断会在绑定的 CPU 上分发；模拟旁路使其在调用线程的 CPU 上运行。这是模拟技术的局限性，但不影响正确性。

### 教训

所有三个层级都驱动相同的过滤器逻辑和相同的任务。唯一的区别是：

- IRQ 资源使用哪个 rid（legacy 为 0，MSI/MSI-X 为 1+）。
- `pci_alloc_msi` 还是 `pci_alloc_msix` 成功。
- 注册了多少个过滤器函数（legacy 为 1，MSI/MSI-X 为 3）。
- 真实中断在哪个 CPU 上分发。

编写良好的驱动程序在所有三个层级上工作方式相同。第20章的回退阶梯确保了这一点。



## 实践实验：跨三个层级的回归测试

一个练习回退阶梯以确认所有三个层级都能工作的实验。

### 设置

您需要两个测试环境：

- **环境 A**：带有 virtio-rnd 的 bhyve。驱动程序回退到 legacy INTx。
- **环境 B**：带有 virtio-rng-pci 的 QEMU。驱动程序使用 MSI-X。

（第三个只有 MSI 而没有 MSI-X 的环境在现代平台上很难可靠构建。只有当读者拥有 MSI-X 失败但 MSI 工作的系统时，MSI 路径才会被练习。）

### 过程

1. 在环境 A 上，加载 `myfirst.ko`。验证：

```sh
sysctl dev.myfirst.0.intr_mode   # returns 0
vmstat -i | grep myfirst          # one line
```

2. 通过模拟中断 sysctl 练习管道。所有三个都应该工作，虽然在 legacy 上它们都经过同一个过滤器。

```sh
sudo sysctl dev.myfirst.0.intr_simulate_admin=2
sudo sysctl dev.myfirst.0.intr_simulate_rx=1
sudo sysctl dev.myfirst.0.intr_simulate_tx=4
sysctl dev.myfirst.0.intr_count   # should be 3
```

3. 卸载。验证无泄漏。

4. 在环境 B 上，重复：

```sh
sysctl dev.myfirst.0.intr_mode   # returns 2
vmstat -i | grep myfirst          # three lines
for irq in <IRQs>; do cpuset -g -x $irq; done
```

5. 练习每向量管道。每个 sysctl 应递增其自己向量的计数器。

```sh
sudo sysctl dev.myfirst.0.intr_simulate_admin=2
sysctl dev.myfirst.0.vec.admin.fire_count  # 1
sysctl dev.myfirst.0.vec.rx.fire_count     # 0
sysctl dev.myfirst.0.vec.tx.fire_count     # 0
```

6. 卸载。验证无泄漏。

### 预期观察

- 两个环境都干净地附加。
- dmesg 摘要行为每个环境显示正确的模式。
- 在 MSI-X 上，每向量计数器独立跳动。
- 在 legacy 上，单个计数器覆盖所有事件。
- 两个环境中卸载后均无泄漏。

### 如果某层失败该怎么办

如果 MSI-X 层级在环境 B 上失败：

1. 验证 QEMU 版本足够新。旧版本（5.0 之前）有 quirks。
2. 在 guest 中检查 `pciconf -lvc`；MSI-X 能力应该可见。
3. 检查 `dmesg` 中来自 `pci_alloc_msix` 的错误。

如果 legacy 层级在环境 A 上失败：

1. 检查 `pciconf -lvc` 以确认设备的中断线配置。
2. 确保 `virtio_rnd` 尚未附加（第18章注意事项）。
3. 在 `dmesg` 中查找 `pci_alloc_resource` 失败。



## 扩展挑战：构建生产级驱动程序

一个可选的练习，适合于希望在现实规模上练习多向量设计的读者。

### 目标

采用第20章的驱动程序，扩展它以动态处理 N 个队列，其中 N 在附加时根据分配的 MSI-X 向量计数确定。每个队列具有：

- 自己的向量（MSI-X 向量 1+queue_id）。
- 自己的过滤器函数（或从向量参数识别队列的共享函数）。
- 自己的计数器。
- 自己任务队列上的自己的任务。
- 自己的 NUMA 本地 CPU 绑定。

### 实现大纲

1. 用运行时选择的计数替换 `MYFIRST_MAX_VECTORS`。
2. 动态分配 `vectors[]` 数组（使用 `malloc`）。
3. Allocate a separate taskqueue per vector.
4. Use `bus_get_cpus(INTR_CPUS, ...)` to distribute vectors across NUMA-local CPUs.
5. 添加随向量计数扩展的 sysctl。

### 测试

在具有不同 MSI-X 向量计数的 guest 上运行驱动程序。对每个计数，验证：
- 模拟中断的触发计数器跳动。
- CPU 亲和性遵守 NUMA 局部性。
- 拆除干净。

### 本练习的内容

- Dynamic memory management in a driver.
- `bus_get_cpus` API。
- 每队列任务队列（之前的挑战3）。
- 运行时 sysctl 树构建（之前的挑战7）。

这是一个重要的练习，可能需要数小时。结果是一个在结构上类似于生产级 NIC 和 NVMe 驱动程序的驱动程序。



## 参考：中断和任务工作的优先级值

快速参考，第20章驱动程序可能使用的优先级常量（来自 `/usr/src/sys/sys/priority.h`）：

```text
PI_REALTIME  = PRI_MIN_ITHD + 0   (highest; rarely used)
PI_INTR      = PRI_MIN_ITHD + 4   (common hardware interrupt level)
PI_AV        = PI_INTR            (audio/video)
PI_NET       = PI_INTR            (network)
PI_DISK      = PI_INTR            (block storage)
PI_TTY       = PI_INTR            (terminal/serial)
PI_DULL      = PI_INTR            (low-priority hardware)
PI_SOFT      = PRI_MIN_ITHD + 8   (soft interrupts)
```

常见的硬件优先级都映射到 `PI_INTR`；这些名称是意图的区别，而不是调度优先级的区别。第20章的驱动程序对其任务队列使用 `PI_NET`；任何硬件级优先级都可以等效工作。



## 参考：MSI-X 驱动程序有用的 DTrace 单行命令

对于希望动态观察第20章驱动程序行为的读者。

### 统计每个 CPU 的过滤器调用次数

```sh
sudo dtrace -n '
fbt::myfirst_admin_filter:entry, fbt::myfirst_rx_filter:entry,
fbt::myfirst_tx_filter:entry { @[probefunc, cpu] = count(); }'
```

显示哪个过滤器在哪个 CPU 上运行。

### 每个过滤器花费的时间

```sh
sudo dtrace -n '
fbt::myfirst_rx_filter:entry { self->ts = vtimestamp; }
fbt::myfirst_rx_filter:return /self->ts/ {
    @[probefunc] = quantize(vtimestamp - self->ts);
    self->ts = 0;
}'
```

RX 过滤器 CPU 时间的直方图。

### 模拟中断与真实中断的比率

```sh
sudo dtrace -n '
fbt::myfirst_intr_simulate_vector_sysctl:entry { @sims = count(); }
fbt::myfirst_rx_filter:entry { @filters = count(); }'
```

如果 `filters > sims`，一些真实中断正在触发。

### 任务延迟

```sh
sudo dtrace -n '
fbt::myfirst_rx_filter:entry { self->ts = vtimestamp; }
fbt::myfirst_rx_task_fn:entry /self->ts/ {
    @lat = quantize(vtimestamp - self->ts);
    self->ts = 0;
}'
```

从过滤器到任务调用的时间直方图。显示任务队列的调度延迟。



## 参考：第四部分结束前的结束语

第16章到第20章为 `myfirst` 驱动程序构建了完整的中断和硬件故事。每章添加一层：

- 第16章：寄存器访问。
- 第17章：设备行为模拟。
- 第18章：PCI 附加。
- 第19章：单向量中断处理。
- 第20章：多向量 MSI/MSI-X。

第21章将添加 DMA，完成第4部分的硬件层。那时，`myfirst` 驱动程序在结构上将是一个真正的驱动程序：具有 MSI-X 中断和基于 DMA 的数据传输的 PCI 设备。它与生产驱动程序的区别在于它所说的特定协议（实际上没有；它是一个演示）和它针对的设备（一个 virtio-rnd 抽象）。

已经内化这五章的读者可以打开 `/usr/src/sys/dev/` 中的任何 FreeBSD 驱动程序并识别这些模式。这种识别是第4部分最深的回报。



## 动手实验

这些实验是分级的检查点。每个实验建立在前一个实验的基础上，对应于本章的一个阶段。完成所有五个实验的读者将拥有一个完整的多年驱动程序、一个用于 MSI-X 的工作 QEMU 测试环境，以及一个验证回退阶梯所有三个层级的回归脚本。

时间预算假设读者已经阅读了相关章节。

### 实验 1：发现 MSI 和 MSI-X 能力

Time: thirty minutes.

目标：建立对系统上哪些设备支持 MSI 和 MSI-X 的直觉。

Steps:

1. 运行 `sudo pciconf -lvc > /tmp/pci_caps.txt`。`-c` 标志包含能力列表。
2. 搜索 MSI 能力：`grep -B 1 "cap 05" /tmp/pci_caps.txt`。
3. 搜索 MSI-X 能力：`grep -B 1 "cap 11" /tmp/pci_caps.txt`。
4. 对于三个支持 MSI-X 的设备，记录：
   - 设备名称（`pci0:B:D:F`）。
   - 支持的 MSI-X 消息数。
   - 驱动程序当前是否正在使用 MSI-X（检查 `vmstat -i` 中同一设备名称的多行）。
5. 比较支持 MSI 的设备总数与支持 MSI-X 的设备总数。现代系统通常有比仅 MSI 设备更多的 MSI-X 设备。

Expected observations:

- NIC 通常公布具有许多向量的 MSI-X（4 到 64）。
- SATA 和 NVMe 控制器公布 MSI-X（NVMe 通常有几十个向量）。
- 一些遗留设备（音频芯片、USB 控制器）仅公布 MSI。
- 少数非常老的设备既不公布，依赖 legacy INTx。

本实验关乎词汇。没有代码。回报是第2节和第5节的分配调用变得具体。

### 实验 2：阶段 1，MSI 回退阶梯

时间：两到三小时。

目标：用 MSI 优先的回退阶梯扩展第19章的驱动程序。版本目标：`1.3-msi-stage1`。

Steps:

1. 从第19章阶段4开始，将驱动程序源代码复制到新的工作目录。
2. 向 `myfirst.h` 添加 `intr_mode` 字段和枚举。
3. 修改 `myfirst_intr_setup`（在 `myfirst_intr.c` 中）以先尝试 MSI 分配，回退到 legacy INTx。
4. 修改 `myfirst_intr_teardown` 以在使用 MSI 时调用 `pci_release_msi`。
5. 添加 `dev.myfirst.N.intr_mode` sysctl。
6. 将 `Makefile` 版本字符串更新为 `1.3-msi-stage1`。
7. 编译（`make clean && make`）。
8. 在 guest 上加载。注意驱动程序报告哪种模式：

```sh
sudo kldload ./myfirst.ko
sudo dmesg | tail -5
sysctl dev.myfirst.0.intr_mode
```

在带有 virtio-rng-pci 的 QEMU 上，驱动程序应报告 `MSI, 1 vector`（或类似信息）。在带有 virtio-rnd 的 bhyve 上，应报告 `legacy INTx`。

9. 卸载并验证无泄漏。

Common failures:

- 缺少 `pci_release_msi`：下次加载失败或回退到 legacy。
- rid 错误（对 MSI 使用 0）：`bus_alloc_resource_any` 返回 NULL。
- 未检查返回计数：驱动程序使用少于预期的向量继续运行。

### 实验 3：阶段 2，多向量分配 (MSI)

时间：三到四小时。

目标：扩展到具有每向量处理程序的三个 MSI 向量。版本目标：`1.3-msi-stage2`。

Steps:

1. 从实验2开始，向 `myfirst.h` 添加 `myfirst_vector` 结构和每向量数组。
2. 编写三个过滤器函数：`myfirst_admin_filter`、`myfirst_rx_filter`、`myfirst_tx_filter`。
3. 编写 `myfirst_intr_setup_vector` 和 `myfirst_intr_teardown_vector` 辅助函数。
4. 修改 `myfirst_intr_setup` 以尝试为 `MYFIRST_MAX_VECTORS` 向量调用 `pci_alloc_msi`，独立设置每个向量。
5. 修改 `myfirst_intr_teardown` 以循环每向量。
6. 添加每向量计数器 sysctl（`vec0_fire_count`、`vec1_fire_count`、`vec2_fire_count`）。
7. 添加每向量模拟中断 sysctl（`intr_simulate_admin`、`intr_simulate_rx`、`intr_simulate_tx`）。
8. 将版本提升到 `1.3-msi-stage2`。
9. 编译，加载，验证：

```sh
sysctl dev.myfirst.0.intr_mode   # should be 1 on QEMU
vmstat -i | grep myfirst          # should show 3 lines
```

10. 练习每个向量：

```sh
sudo sysctl dev.myfirst.0.intr_simulate_admin=2  # ERROR
sudo sysctl dev.myfirst.0.intr_simulate_rx=1     # DATA_AV
sudo sysctl dev.myfirst.0.intr_simulate_tx=4     # COMPLETE
sysctl dev.myfirst.0 | grep vec
```

每个向量的计数器应独立递增。

11. 卸载，验证无泄漏。

### 实验 4：阶段 3，带 CPU 绑定的 MSI-X

时间：三到四小时。

目标：优先使用 MSI-X 而非 MSI，将每个向量绑定到 CPU。版本目标：`1.3-msi-stage3`。

Steps:

1. 从实验3开始，更改回退阶梯以先尝试 MSI-X（通过 `pci_msix_count` 和 `pci_alloc_msix`），MSI 作为第二层，legacy 作为最后。
2. 添加 `myfirst_msix_bind_vectors` 辅助函数，为每个向量调用 `bus_bind_intr`。
3. 在所有向量注册后调用绑定辅助函数。
4. 更新 dmesg 摘要行以区分 MSI-X 和 MSI。
5. 将版本提升到 `1.3-msi-stage3`。
6. 编译，在带有 `virtio-rng-pci` 的 QEMU 上加载。验证：

```sh
sysctl dev.myfirst.0.intr_mode   # 在 QEMU 上应为 2
sudo dmesg | grep myfirst | grep MSI-X
```

附加行应为 `interrupt mode: MSI-X, 3 vectors`。

7. 检查每向量 CPU 绑定：

```sh
# For each myfirst IRQ, show its CPU binding.
vmstat -i | grep myfirst
# (Note the IRQ numbers, then:)
for irq in <IRQ1> <IRQ2> <IRQ3>; do
    echo "IRQ $irq:"
    cpuset -g -x $irq
done
```

在多 CPU guest 上，每个向量应绑定到不同的 CPU。

8. 练习每个向量（与实验3相同）。

9. 分离并重新附加：

```sh
sudo devctl detach myfirst0
sudo devctl attach pci0:0:4:0
sysctl dev.myfirst.0.intr_mode  # 应仍为 2
```

10. 卸载，验证无泄漏。

### 实验 5：阶段 4，重构、回归、版本

时间：三到四小时。

目标：将多向量代码移入 `myfirst_msix.c`，编写 `MSIX.md`，运行回归测试。版本目标：`1.3-msi`。

Steps:

1. 从实验4开始，创建 `myfirst_msix.c` 和 `myfirst_msix.h`。
2. 将每向量过滤器函数、辅助函数、设置、拆除和 sysctl 注册移入 `myfirst_msix.c`。
3. 将 legacy-INTx 回退保留在 `myfirst_intr.c`（第19章的文件）中。
4. 在 `myfirst_pci.c` 中，用对 `myfirst_msix.c` 的调用替换旧的中断设置/拆除调用。
5. 更新 `Makefile` 将 `myfirst_msix.c` 添加到 SRCS。将版本提升到 `1.3-msi`。
6. 编写 `MSIX.md` 记录多向量设计。
7. 编译、加载、运行完整的回归脚本（来自配套示例）。
8. 确认所有三个层级都工作（通过在 bhyve 上使用 virtio-rnd 测试 legacy，在 QEMU 上使用 virtio-rng-pci 测试 MSI-X）。

Expected outcomes:

- `1.3-msi` 版本的驱动程序在 bhyve（legacy 回退）和 QEMU（MSI-X）上都能工作。
- `myfirst_intr.c` 现在只包含第19章的单处理程序回退路径。
- `myfirst_msix.c` 包含第20章的多向量逻辑。
- `MSIX.md` 清晰地记录了该设计。



## 挑战练习

这些挑战练习建立在实验的基础上，将驱动程序扩展至本章未涉及的方向。

### 挑战 1：动态向量计数适配

修改设置以适应内核实际分配的任何向量计数。如果请求 3 个但只分配了 2 个，驱动程序仍应能使用 2 个工作（将 admin 和 tx 合并到一个向量中）。如果只分配了 1 个，将所有内容合并到一个中。

此练习教授回退阶梯中的"适应"策略。

### 挑战 2：NUMA 感知的 CPU 绑定

使用 `bus_get_cpus(dev, INTR_CPUS, ...)` 将轮询 CPU 绑定替换为 NUMA 感知绑定。用 `cpuset -g -x <irq>` 验证向量落在与设备同一 NUMA 域的 CPU 上。

在单插槽系统上，此练习是理论性的；在多插槽测试主机上，效果是可测量的。

### 挑战 3：每向量任务队列

目前每个向量共享一个任务队列。修改驱动程序，使每个向量有自己的任务队列（有自己的工作线程）。用 DTrace 测量延迟影响。

此练习引入每向量工作者，并展示它们何时有帮助、何时有害。

### 挑战 4：每向量 MSI-X 掩码控制

MSI-X 表的向量控制寄存器每个向量有一个掩码位。添加一个 sysctl，让操作员在运行时掩码单个向量。验证被掩码的向量停止接收中断。

提示：掩码位通过直接 MSI-X 表访问编程，这是比第20章涵盖更深的话题。FreeBSD 的 MSI-X 实现可能直接暴露也可能不直接暴露这一点；读者可能需要使用 `bus_teardown_intr` 然后 `bus_setup_intr` 作为更高级别的"软掩码"。

### 挑战5：实现中断调节

对于模拟驱动程序，调节很容易原型化：一个将 N 个模拟中断合并为一次任务运行的 sysctl。实现合并，测量延迟与吞吐量的权衡。

### 挑战6：运行时向量重分配

添加一个 sysctl，让操作员重新分配哪个向量处理哪个事件类（例如，交换 RX 和 TX）。演示重新分配后，模拟中断 RX 触发 TX 过滤器，反之亦然。

### 挑战7：每队列 Sysctl 树

将每向量 sysctl 重构为适当的树形结构：`dev.myfirst.N.vec.admin.fire_count`、`dev.myfirst.N.vec.rx.fire_count` 等。使用 `SYSCTL_ADD_NODE` 创建树节点。

### 挑战8：DTrace 监测

编写一个 DTrace 脚本，显示每个向量过滤器调用的每 CPU 分布。将每 CPU 分解绘制为直方图。这是确认 CPU 绑定正在工作的诊断方法。



## 故障排除和常见错误

### "pci_alloc_msix 返回 EBUSY 或 ENXIO"

可能的原因：

1. 设备未以支持 MSI-X 的方式连接（例如，bhyve 上的 legacy virtio-rnd）。检查 `pciconf -lvc`。
2. 先前加载的驱动程序在拆除时未调用 `pci_release_msi`。重新启动或尝试再次 `kldunload` + `kldload`。
3. 内核用尽了中断向量。在现代 x86 上很少见，在低向量平台上可能发生。

### "vmstat -i 在 MSI-X guest 上只显示一行"

可能的原因：`pci_alloc_msix` 成功但只分配了 1 个向量。检查返回计数与请求计数。接受（将工作合并到一个）或释放并回退。

### "过滤器触发但 vec->fire_count 保持为零"

可能的原因：`sc` 参数与 `vec` 混淆。处理程序接收 `vec`，而不是 `sc`。检查 `bus_setup_intr` 的参数。

### "多次加载/卸载循环后 kldunload 时驱动崩溃"

可能的原因：拆除时未调用 `pci_release_msi`。设备级 MSI 状态在加载之间泄漏；最终内核的内部记账混乱。

### "不同的向量都在同一个 CPU 上触发"

可能的原因：`bus_bind_intr` 静默失败。检查返回值并记录非零结果。

### "MSI-X 分配成功但 vmstat -i 不显示事件"

可能的原因：设备的 `INTR_MASK` 写入针对了错误的寄存器或被跳过。验证掩码已设置（第17章/第19章诊断）。

### "杂散中断在 MSI-X 管理向量上累积"

可能的原因：管理过滤器的状态检查错误；过滤器应该处理时返回了 `FILTER_STRAY`。检查 `status & MYFIRST_INTR_ERROR` 检查。

### "legacy 回退上的共享 IRQ 行为与 MSI-X 不同"

这是预期的。在 legacy INTx 上，单个处理程序看到每个事件位；在 MSI-X 上，每个向量只看到自己的事件。练习每向量杂散计数的测试在这两种模式下不同。

### "阶段 2 编译通过但阶段 3 在 `bus_get_cpus` 链接错误处失败"

原因：`bus_get_cpus` 可能在较旧的 FreeBSD 版本中不可用，或可能需要特定的 `#include <sys/bus.h>` 放置。检查 include 顺序。

### "QEMU guest 不暴露 MSI-X 即使使用 virtio-rng-pci"

可能的原因：较旧的 QEMU 版本默认使用 legacy virtio。在 guest 中检查 `pciconf -lvc`；如果未列出 MSI-X，则 guest 正在使用 legacy。更新 QEMU 或使用 `-device virtio-rng-pci,disable-modern=off,disable-legacy=on`。



## 总结

第 20 章赋予驱动程序处理多个中断向量的能力。起点是 `1.2-intr`，在遗留 INTx 线上有一个处理程序。终点是 `1.3-msi`，具有三层回退阶梯（MSI-X、MSI、遗留）、三个每向量过滤器处理程序、每向量计数器和任务、每向量 CPU 绑定、干净的多向量拆除，以及新的 `myfirst_msix.c` 文件和 `MSIX.md` 文档。

八个部分涵盖了完整的进展。第 1 节在硬件层面介绍了 MSI 和 MSI-X。第 2 节添加了 MSI 作为遗留 INTx 的单向量替代方案。第 3 节扩展到多向量 MSI。第 4 节检查了多个 CPU 上多个过滤器的并发影响。第 5 节转向具有每向量 CPU 绑定的 MSI-X。第 6 节编纂了每向量事件角色。第 7 节整合了拆除。第 8 节重构为最终布局。

第 20 章没有做的是 DMA。每个向量的处理程序仍然只触碰寄存器；设备还没有能力直接访问 RAM。第 21 章将改变这一点。DMA 引入了新的复杂性（一致性、分散-聚集、映射），与中断交互（完成中断表示 DMA 传输完成）。第 20 章的中断机制已准备好处理完成中断；第 21 章编写 DMA 部分。

文件布局已经增长：14 个源文件（包括 `cbuf`），6 个文档文件（`HARDWARE.md`、`LOCKING.md`、`SIMULATION.md`、`PCI.md`、`INTERRUPTS.md`、`MSIX.md`），以及不断增长的回归测试套件。驱动程序在结构上已经与生产 FreeBSD 驱动程序平行。

### 第21章前的反思

第 20 章是第四部分中纯粹关于中断的最后一章。第 21 章转向 DMA，即关于移动数据。两者是互补的：中断发出事件信号；DMA 移动这些事件所涉及的数据。高性能驱动程序同时使用两者：接收描述符由设备通过 DMA 填充到 RAM，然后完成中断通知驱动程序处理描述符。

第 20 章的每向量处理程序已经是正确的形式。每个接收队列的完成中断触发自己的向量；每个向量的过滤器确认并延迟；任务遍历接收环（由 DMA 填充，第 21 章）并向上传递数据包。第 21 章编写 DMA 部分；第 20 章的中断部分已经就位。

本章的教学也具有普遍性。掌握了第 20 章三层回退阶梯、每向量状态设计、CPU 绑定和干净拆除的读者，会在每个多队列 FreeBSD 驱动程序中发现类似的模式。具体设备不同；结构不变。

### 如果遇到困难该怎么办

三个建议。

首先，仔细阅读 `/usr/src/sys/dev/virtio/pci/virtio_pci.c`，重点关注 `vtpci_alloc_intr_resources` 系列函数。该模式与第 20 章完全匹配，代码足够紧凑，可以一次读完。

其次，在 bhyve 客户机（遗留回退）和 QEMU 客户机（MSI-X）上都运行本章的回归测试。看到同一驱动程序在两个目标上正确运行，确认回退阶梯是正确的。

第三，第一次阅读时跳过挑战。实验是为第 20 章的节奏校准的；挑战假设材料已掌握。如果现在觉得难以完成，可以在第 21 章之后再回来。

第 20 章的目标是为驱动程序提供多向量中断路径。如果已经实现，第 21 章的 DMA 工作将成为补充而不是一个全新的主题。



## 通往第21章的桥梁

第 21 章标题为 *DMA 和高速数据传输*。其范围是第 20 章刻意未涉及的主题：设备直接读写 RAM 的能力，无需驱动程序参与每个字。 具有 64 条目接收描述符环的 NIC 通过 DMA 从线路上填充这些条目；单个中断信号表示"N 条目已就绪"。驱动程序的处理程序遍历环并处理条目。没有 DMA，驱动程序必须从设备寄存器读取每个字节，这无法扩展。

第 20 章以三种具体方式做好了准备。

第一，**你有每向量完成中断**。每个队列的接收完成和发送完成可以触发专用向量。第 21 章的 DMA 环工作插入第 20 章的每向量过滤器和任务；过滤器看到"完成 N 到 M 就绪"，任务处理它们。

第二，**你有每 CPU 处理程序放置**。DMA 环的内存位于特定的 NUMA 节点上；处理它的处理程序应该在该节点上的 CPU 上运行。第 20 章的 `bus_bind_intr` 工作是机制。第 21 章扩展了这一点：DMA 内存也以 NUMA 感知方式分配，因此环、处理程序和处理都在同一节点上。

第三，**你有拆除规范**。DMA 添加了更多资源（DMA 标签、DMA 映射、DMA 内存区域），每个都需要自己的拆除步骤。第 19/20 章的每向量拆除模式自然扩展到每队列 DMA 清理。

第 21 章将涵盖的具体主题：

- DMA 是什么，内存映射 I/O 与 DMA 的区别。
- `bus_dma(9)`：标签、映射和 DMA 状态机。
- `bus_dma_tag_create` 描述 DMA 需求（对齐、边界、地址范围）。
- `bus_dmamap_create` 和 `bus_dmamap_load` 设置 DMA 传输。
- 同步：DMA 前后的 `bus_dmamap_sync`。
- 反弹缓冲区：它们是什么以及何时使用。
- 缓存一致性：为什么 CPU 和设备在不同时间看到不同的内容。
- 分散-聚集列表：不连续的物理地址。
- 环形缓冲区：生产者-消费者描述符环模式。

你不需要提前阅读。第 20 章已足够准备。带上你的 `myfirst` 驱动程序（`1.3-msi` 版本）、`LOCKING.md`、`INTERRUPTS.md`、`MSIX.md`、启用 `WITNESS` 的内核和回归脚本。第 21 章从第 20 章结束的地方开始。

硬件对话正在深入。词汇是你的；结构是你的；规范是你的。第 21 章添加下一个缺失的部分：设备无需请求即可移动数据的能力。



## 参考：第20章快速参考卡片

第 20 章引入的词汇、API、宏和过程的简洁总结。

### 词汇

- **MSI (Message Signalled Interrupts)**: PCI 2.2 mechanism. 1 to 32 vectors, contiguous, single address.
- **MSI-X**: PCIe mechanism. Up to 2048 vectors, per-vector address, per-vector mask, table in a BAR.
- **vector**: a single interrupt source identified by an index.
- **rid**: the resource ID used with `bus_alloc_resource_any`. 0 for legacy INTx, 1+ for MSI and MSI-X.
- **intr_mode**: the driver's record of which tier it is using (legacy, MSI, or MSI-X).
- **fallback ladder**: try MSI-X first, then MSI, then legacy INTx.
- **per-vector state**: counters, filter, task, cookie, resource per vector.
- **CPU binding**: routing a vector to a specific CPU via `bus_bind_intr`.
- **LOCAL_CPUS / INTR_CPUS**: CPU-set queries for NUMA-aware placement.

### 核心 API

- `pci_msi_count(dev)`: query MSI vector count.
- `pci_msix_count(dev)`: query MSI-X vector count.
- `pci_alloc_msi(dev, &count)`: allocate MSI vectors.
- `pci_alloc_msix(dev, &count)`: allocate MSI-X vectors.
- `pci_release_msi(dev)`: release MSI or MSI-X vectors.
- `pci_msix_table_bar(dev)`, `pci_msix_pba_bar(dev)`: identify table/PBA BARs.
- `bus_alloc_resource_any(dev, SYS_RES_IRQ, &rid, RF_ACTIVE)`: allocate per-vector IRQ resource.
- `bus_setup_intr(dev, res, flags, filter, ihand, arg, &cookie)`: register per-vector handler.
- `bus_teardown_intr(dev, res, cookie)`: unregister per-vector handler.
- `bus_describe_intr(dev, res, cookie, "name")`: label per-vector handler.
- `bus_bind_intr(dev, res, cpu)`: bind vector to a specific CPU.
- `bus_get_cpus(dev, op, size, &set)`: query NUMA-local CPUs (op = LOCAL_CPUS or INTR_CPUS).

### 核心宏

- `PCIY_MSI = 0x05`: MSI capability ID.
- `PCIY_MSIX = 0x11`: MSI-X capability ID.
- `PCIM_MSIXCTRL_TABLE_SIZE = 0x07FF`: mask for vector count.
- `PCI_MSIX_MSGNUM(ctrl)`: macro to extract vector count from control register.
- `MYFIRST_MAX_VECTORS`: driver-defined constant (3 in Chapter 20).

### 常用过程

**Implement the three-tier fallback ladder:**

1. `pci_msix_count(dev)`; if > 0, try `pci_alloc_msix`.
2. On failure, `pci_msi_count(dev)`; if > 0, try `pci_alloc_msi`.
3. On failure, fall back to legacy INTx with `rid = 0` and `RF_SHAREABLE`.

**Register per-vector handlers (MSI-X):**

1. Loop from `i = 0` to `num_vectors - 1`.
2. For each: `bus_alloc_resource_any(dev, SYS_RES_IRQ, &rid, RF_ACTIVE)` with `rid = i + 1`.
3. `bus_setup_intr(dev, vec->irq_res, INTR_TYPE_MISC | INTR_MPSAFE, vec->filter, NULL, vec, &vec->intr_cookie)`.
4. `bus_describe_intr(dev, vec->irq_res, vec->intr_cookie, vec->name)`.
5. `bus_bind_intr(dev, vec->irq_res, target_cpu)`.

**Tear down a multi-vector driver:**

1. Clear `INTR_MASK` at the device.
2. For each vector (reverse order): `bus_teardown_intr`, `bus_release_resource`.
3. Drain each per-vector task.
4. Free the taskqueue.
5. `pci_release_msi(dev)` if MSI or MSI-X was used.

### 实用命令

- `pciconf -lvc`: list devices with capability lists.
- `vmstat -i`: show per-handler interrupt counts.
- `cpuset -g -x <irq>`: query CPU affinity for an IRQ.
- `cpuset -l <cpu> -x <irq>`: set CPU affinity for an IRQ.
- `sysctl dev.myfirst.0.intr_mode`: query driver's interrupt mode.

### 建议收藏的文件

- `/usr/src/sys/dev/pci/pcivar.h`: MSI/MSI-X inline wrappers.
- `/usr/src/sys/dev/pci/pcireg.h`: capability IDs and bit fields.
- `/usr/src/sys/dev/pci/pci.c`: kernel-side implementation of `pci_alloc_msi`/`msix`.
- `/usr/src/sys/dev/virtio/pci/virtio_pci.c`: clean MSI-X fallback-ladder example.
- `/usr/src/sys/dev/nvme/nvme_ctrlr.c`: per-queue MSI-X pattern at scale.



## 参考：第20章术语词汇表

**affinity**: the mapping from an interrupt vector to a specific CPU (or set of CPUs).

**bus_bind_intr(9)**: function to route an interrupt vector to a specific CPU.

**bus_get_cpus(9)**: function to query CPU sets associated with a device (local, interrupt-suitable).

**capability list**: the linked list of PCI device capabilities in configuration space.

**coalescing**: buffering multiple events into one interrupt to reduce rate.

**cookie**: the opaque handle returned by `bus_setup_intr(9)`, used by `bus_teardown_intr(9)`.

**fallback ladder**: the sequence MSI-X → MSI → legacy INTx that drivers implement.

**intr_mode**: driver-side enum recording which interrupt tier is active.

**INTR_CPUS**: cpu_sets enum value; CPUs suitable for handling device interrupts.

**LOCAL_CPUS**: cpu_sets enum value; CPUs in the same NUMA domain as the device.

**MSI**: Message Signalled Interrupts, PCI 2.2.

**MSI-X**: the fuller mechanism, PCIe.

**moderation**: buffering interrupts at the device level to trade latency for throughput.

**NUMA**: Non-Uniform Memory Access; multi-socket system architecture.

**per-vector state**: the softc fields specific to one vector (counters, filter, task, cookie, resource).

**pci_msi_count(9) / pci_msix_count(9)**: capability-count queries.

**pci_alloc_msi(9) / pci_alloc_msix(9)**: vector allocation.

**pci_release_msi(9)**: release of MSI/MSI-X (handles both).

**rid**: resource ID. 0 for legacy INTx, 1+ for MSI/MSI-X vectors.

**stray interrupt**: an interrupt that no filter claims.

**taskqueue**: FreeBSD's deferred-work primitive.

**vector**: a single interrupt source in the MSI or MSI-X mechanism.

**vmstat -i**: diagnostic showing per-handler interrupt counts.



## 参考：完整的第4阶段 myfirst_msix.c 演练

对于希望在一个地方看到最终多向量层注释的读者，本附录遍历了配套示例中的 `myfirst_msix.c`，展示每个函数并解释设计选择。

### 文件开头

```c
#include <sys/param.h>
#include <sys/systm.h>
#include <sys/kernel.h>
#include <sys/bus.h>
#include <sys/lock.h>
#include <sys/mutex.h>
#include <sys/condvar.h>
#include <sys/rman.h>
#include <sys/sysctl.h>
#include <sys/taskqueue.h>
#include <sys/types.h>
#include <sys/smp.h>

#include <machine/atomic.h>
#include <machine/bus.h>
#include <machine/resource.h>

#include <dev/pci/pcireg.h>
#include <dev/pci/pcivar.h>

#include "myfirst.h"
#include "myfirst_hw.h"
#include "myfirst_intr.h"
#include "myfirst_msix.h"
```

include 列表比 `myfirst_intr.c` 的更长：`<dev/pci/pcireg.h>` 和 `<dev/pci/pcivar.h>` 用于 MSI/MSI-X API，`<sys/smp.h>` 用于 `mp_ncpus`，以及 `<machine/atomic.h>` 用于每向量计数器增量。请注意，即使文件不直接使用 `PCIY_MSI` 或类似常量，也会引入 `<dev/pci/pcireg.h>`；`pcivar.h` 中的访问器内联函数依赖于它。

### 每向量辅助函数

```c
static int
myfirst_msix_setup_vector(struct myfirst_softc *sc, int idx, int rid)
{
	struct myfirst_vector *vec = &sc->vectors[idx];
	int error;

	vec->irq_rid = rid;
	vec->irq_res = bus_alloc_resource_any(sc->dev, SYS_RES_IRQ,
	    &vec->irq_rid, RF_ACTIVE);
	if (vec->irq_res == NULL)
		return (ENXIO);

	error = bus_setup_intr(sc->dev, vec->irq_res,
	    INTR_TYPE_MISC | INTR_MPSAFE,
	    vec->filter, NULL, vec, &vec->intr_cookie);
	if (error != 0) {
		bus_release_resource(sc->dev, SYS_RES_IRQ,
		    vec->irq_rid, vec->irq_res);
		vec->irq_res = NULL;
		return (error);
	}

	bus_describe_intr(sc->dev, vec->irq_res, vec->intr_cookie,
	    "%s", vec->name);
	return (0);
}

static void
myfirst_msix_teardown_vector(struct myfirst_softc *sc, int idx)
{
	struct myfirst_vector *vec = &sc->vectors[idx];

	if (vec->intr_cookie != NULL) {
		bus_teardown_intr(sc->dev, vec->irq_res, vec->intr_cookie);
		vec->intr_cookie = NULL;
	}
	if (vec->irq_res != NULL) {
		bus_release_resource(sc->dev, SYS_RES_IRQ,
		    vec->irq_rid, vec->irq_res);
		vec->irq_res = NULL;
	}
}
```

这些辅助函数是第 3 节中的对称对。每个函数都接受一个向量索引，并对该槽位的 `vec` 进行操作。设置辅助函数在失败时使向量保持干净状态，在这个意义上是幂等的；拆除辅助函数即使设置未完成也可以安全调用。

### 每向量过滤函数

三个过滤器的区别仅在于它们检查的位。它们的共同形式：

```c
int
myfirst_msix_rx_filter(void *arg)
{
	struct myfirst_vector *vec = arg;
	struct myfirst_softc *sc = vec->sc;
	uint32_t status;

	status = ICSR_READ_4(sc, MYFIRST_REG_INTR_STATUS);
	if ((status & MYFIRST_INTR_DATA_AV) == 0) {
		atomic_add_64(&vec->stray_count, 1);
		return (FILTER_STRAY);
	}

	atomic_add_64(&vec->fire_count, 1);
	ICSR_WRITE_4(sc, MYFIRST_REG_INTR_STATUS, MYFIRST_INTR_DATA_AV);
	atomic_add_64(&sc->intr_data_av_count, 1);
	if (sc->intr_tq != NULL)
		taskqueue_enqueue(sc->intr_tq, &vec->task);
	return (FILTER_HANDLED);
}
```

admin 过滤器检查 `MYFIRST_INTR_ERROR`，tx 过滤器检查 `MYFIRST_INTR_COMPLETE`。每个过滤器都会增加相应的全局计数器和每向量计数器。只有 rx 过滤器会入队任务。

### RX 任务

```c
static void
myfirst_msix_rx_task_fn(void *arg, int npending)
{
	struct myfirst_vector *vec = arg;
	struct myfirst_softc *sc = vec->sc;

	MYFIRST_LOCK(sc);
	if (sc->hw != NULL && sc->pci_attached) {
		sc->intr_last_data = CSR_READ_4(sc, MYFIRST_REG_DATA_OUT);
		sc->intr_task_invocations++;
		cv_broadcast(&sc->data_cv);
	}
	MYFIRST_UNLOCK(sc);
}
```

任务在线程上下文中运行并安全地获取 `sc->mtx`。它在触碰共享状态之前检查 `sc->pci_attached`，以防止任务在分离期间运行时的竞争条件。

### 主设置函数

设置函数协调回退阶梯：

```c
int
myfirst_msix_setup(struct myfirst_softc *sc)
{
	int error, wanted, allocated, i;

	/* Initialise per-vector state common to all tiers. */
	for (i = 0; i < MYFIRST_MAX_VECTORS; i++) {
		sc->vectors[i].id = i;
		sc->vectors[i].sc = sc;
	}
	TASK_INIT(&sc->vectors[MYFIRST_VECTOR_RX].task, 0,
	    myfirst_msix_rx_task_fn,
	    &sc->vectors[MYFIRST_VECTOR_RX]);
	sc->vectors[MYFIRST_VECTOR_RX].has_task = true;
	sc->vectors[MYFIRST_VECTOR_ADMIN].filter = myfirst_msix_admin_filter;
	sc->vectors[MYFIRST_VECTOR_RX].filter = myfirst_msix_rx_filter;
	sc->vectors[MYFIRST_VECTOR_TX].filter = myfirst_msix_tx_filter;
	sc->vectors[MYFIRST_VECTOR_ADMIN].name = "admin";
	sc->vectors[MYFIRST_VECTOR_RX].name = "rx";
	sc->vectors[MYFIRST_VECTOR_TX].name = "tx";

	sc->intr_tq = taskqueue_create("myfirst_intr", M_WAITOK,
	    taskqueue_thread_enqueue, &sc->intr_tq);
	taskqueue_start_threads(&sc->intr_tq, 1, PI_NET,
	    "myfirst intr taskq");

	wanted = MYFIRST_MAX_VECTORS;

	/* Tier 0: MSI-X. */
	if (pci_msix_count(sc->dev) >= wanted) {
		allocated = wanted;
		if (pci_alloc_msix(sc->dev, &allocated) == 0 &&
		    allocated == wanted) {
			for (i = 0; i < wanted; i++) {
				error = myfirst_msix_setup_vector(sc, i,
				    i + 1);
				if (error != 0) {
					for (i -= 1; i >= 0; i--)
						myfirst_msix_teardown_vector(
						    sc, i);
					pci_release_msi(sc->dev);
					goto try_msi;
				}
			}
			sc->intr_mode = MYFIRST_INTR_MSIX;
			sc->num_vectors = wanted;
			myfirst_msix_bind_vectors(sc);
			device_printf(sc->dev,
			    "interrupt mode: MSI-X, %d vectors\n", wanted);
			goto enabled;
		}
		if (allocated > 0)
			pci_release_msi(sc->dev);
	}

try_msi:
	/*
	 * Tier 1: MSI with a single vector. MSI requires a power-of-two
	 * count, so we cannot request MYFIRST_MAX_VECTORS (3) here. We
	 * request 1 vector and fall back to the Chapter 19 single-handler
	 * pattern, matching the approach sys/dev/virtio/pci/virtio_pci.c
	 * takes in vtpci_alloc_msi().
	 */
	allocated = 1;
	if (pci_msi_count(sc->dev) >= 1 &&
	    pci_alloc_msi(sc->dev, &allocated) == 0 && allocated >= 1) {
		sc->vectors[MYFIRST_VECTOR_ADMIN].filter = myfirst_intr_filter;
		sc->vectors[MYFIRST_VECTOR_ADMIN].name = "msi";
		error = myfirst_msix_setup_vector(sc, MYFIRST_VECTOR_ADMIN, 1);
		if (error == 0) {
			sc->intr_mode = MYFIRST_INTR_MSI;
			sc->num_vectors = 1;
			device_printf(sc->dev,
			    "interrupt mode: MSI, 1 vector "
			    "(single-handler fallback)\n");
			goto enabled;
		}
		pci_release_msi(sc->dev);
	}

try_legacy:
	/* Tier 2: legacy INTx. */
	sc->vectors[MYFIRST_VECTOR_ADMIN].filter = myfirst_intr_filter;
	sc->vectors[MYFIRST_VECTOR_ADMIN].irq_rid = 0;
	sc->vectors[MYFIRST_VECTOR_ADMIN].irq_res = bus_alloc_resource_any(
	    sc->dev, SYS_RES_IRQ,
	    &sc->vectors[MYFIRST_VECTOR_ADMIN].irq_rid,
	    RF_SHAREABLE | RF_ACTIVE);
	if (sc->vectors[MYFIRST_VECTOR_ADMIN].irq_res == NULL) {
		taskqueue_free(sc->intr_tq);
		sc->intr_tq = NULL;
		return (ENXIO);
	}
	error = bus_setup_intr(sc->dev,
	    sc->vectors[MYFIRST_VECTOR_ADMIN].irq_res,
	    INTR_TYPE_MISC | INTR_MPSAFE,
	    myfirst_intr_filter, NULL, sc,
	    &sc->vectors[MYFIRST_VECTOR_ADMIN].intr_cookie);
	if (error != 0) {
		bus_release_resource(sc->dev, SYS_RES_IRQ,
		    sc->vectors[MYFIRST_VECTOR_ADMIN].irq_rid,
		    sc->vectors[MYFIRST_VECTOR_ADMIN].irq_res);
		sc->vectors[MYFIRST_VECTOR_ADMIN].irq_res = NULL;
		taskqueue_free(sc->intr_tq);
		sc->intr_tq = NULL;
		return (error);
	}
	bus_describe_intr(sc->dev,
	    sc->vectors[MYFIRST_VECTOR_ADMIN].irq_res,
	    sc->vectors[MYFIRST_VECTOR_ADMIN].intr_cookie, "legacy");
	sc->intr_mode = MYFIRST_INTR_LEGACY;
	sc->num_vectors = 1;
	device_printf(sc->dev,
	    "interrupt mode: legacy INTx (1 handler for all events)\n");

enabled:
	MYFIRST_LOCK(sc);
	if (sc->hw != NULL)
		CSR_WRITE_4(sc, MYFIRST_REG_INTR_MASK,
		    MYFIRST_INTR_DATA_AV | MYFIRST_INTR_ERROR |
		    MYFIRST_INTR_COMPLETE);
	MYFIRST_UNLOCK(sc);

	return (0);
}
```

该函数很长，因为它处理三个层级，每个层级都有自己的分配、每向量设置循环和部分失败回退。追踪流程的读者会看到首先尝试 MSI-X，任何失败都会降级到 MSI，再失败则降级到 legacy。`enabled:` 标签可以从任何成功的层级到达。

legacy 层级是第 19 章的路径：一个过滤器（来自 `myfirst_intr.c` 的 `myfirst_intr_filter`），`rid = 0`，`RF_SHAREABLE`。每向量计数器在这个层级上并不真正使用；第 19 章的代码有自己的计数。

### 拆除函数

```c
void
myfirst_msix_teardown(struct myfirst_softc *sc)
{
	int i;

	MYFIRST_LOCK(sc);
	if (sc->hw != NULL && sc->bar_res != NULL)
		CSR_WRITE_4(sc, MYFIRST_REG_INTR_MASK, 0);
	MYFIRST_UNLOCK(sc);

	for (i = sc->num_vectors - 1; i >= 0; i--)
		myfirst_msix_teardown_vector(sc, i);

	if (sc->intr_tq != NULL) {
		for (i = 0; i < sc->num_vectors; i++) {
			if (sc->vectors[i].has_task)
				taskqueue_drain(sc->intr_tq,
				    &sc->vectors[i].task);
		}
		taskqueue_free(sc->intr_tq);
		sc->intr_tq = NULL;
	}

	if (sc->intr_mode == MYFIRST_INTR_MSI ||
	    sc->intr_mode == MYFIRST_INTR_MSIX)
		pci_release_msi(sc->dev);

	sc->num_vectors = 0;
	sc->intr_mode = MYFIRST_INTR_LEGACY;
}
```

该函数遵循严格的顺序：在设备处禁用、逆序每向量拆除、每向量任务排空、释放任务队列、释放 MSI。没有意外；对称性是回报。

### 绑定函数

```c
static void
myfirst_msix_bind_vectors(struct myfirst_softc *sc)
{
	int i, cpu;
	int err;

	if (mp_ncpus < 2)
		return;

	for (i = 0; i < sc->num_vectors; i++) {
		cpu = i % mp_ncpus;
		err = bus_bind_intr(sc->dev, sc->vectors[i].irq_res, cpu);
		if (err != 0)
			device_printf(sc->dev,
			    "bus_bind_intr vec %d: %d\n", i, err);
	}
}
```

轮询绑定。仅在 MSI-X 上调用（该函数在 MSI 或 legacy 上无用；设置阶梯在这些层级上跳过它）。在单 CPU 系统上，函数提前返回而不进行绑定。

### sysctl 函数

```c
void
myfirst_msix_add_sysctls(struct myfirst_softc *sc)
{
	struct sysctl_ctx_list *ctx = &sc->sysctl_ctx;
	struct sysctl_oid_list *kids = SYSCTL_CHILDREN(sc->sysctl_tree);
	char name[32];
	int i;

	SYSCTL_ADD_INT(ctx, kids, OID_AUTO, "intr_mode",
	    CTLFLAG_RD, &sc->intr_mode, 0,
	    "0=legacy, 1=MSI, 2=MSI-X");

	for (i = 0; i < MYFIRST_MAX_VECTORS; i++) {
		snprintf(name, sizeof(name), "vec%d_fire_count", i);
		SYSCTL_ADD_U64(ctx, kids, OID_AUTO, name,
		    CTLFLAG_RD, &sc->vectors[i].fire_count, 0,
		    "Times this vector's filter was called");
		snprintf(name, sizeof(name), "vec%d_stray_count", i);
		SYSCTL_ADD_U64(ctx, kids, OID_AUTO, name,
		    CTLFLAG_RD, &sc->vectors[i].stray_count, 0,
		    "Stray returns from this vector");
	}

	SYSCTL_ADD_PROC(ctx, kids, OID_AUTO, "intr_simulate_admin",
	    CTLTYPE_UINT | CTLFLAG_WR | CTLFLAG_MPSAFE,
	    &sc->vectors[MYFIRST_VECTOR_ADMIN], 0,
	    myfirst_intr_simulate_vector_sysctl, "IU",
	    "Simulate admin vector interrupt");
	SYSCTL_ADD_PROC(ctx, kids, OID_AUTO, "intr_simulate_rx",
	    CTLTYPE_UINT | CTLFLAG_WR | CTLFLAG_MPSAFE,
	    &sc->vectors[MYFIRST_VECTOR_RX], 0,
	    myfirst_intr_simulate_vector_sysctl, "IU",
	    "Simulate rx vector interrupt");
	SYSCTL_ADD_PROC(ctx, kids, OID_AUTO, "intr_simulate_tx",
	    CTLTYPE_UINT | CTLFLAG_WR | CTLFLAG_MPSAFE,
	    &sc->vectors[MYFIRST_VECTOR_TX], 0,
	    myfirst_intr_simulate_vector_sysctl, "IU",
	    "Simulate tx vector interrupt");
}
```

该函数构建三个只读的每向量计数器 sysctl 和三个只写的模拟中断 sysctl。树形样式（挑战 7）留作练习。

### 代码行数

完整的 `myfirst_msix.c` 文件大约有 330 行。这对驱动程序来说是一个相当大的补充，但它带来了第 20 章的所有功能：三层回退、每向量处理程序、每向量计数器、CPU 绑定、干净的拆除。

相比之下，第 19 章的 `myfirst_intr.c` 大约有 250 行。第 20 章的文件在绝对长度上并没有长多少；每向量逻辑增加了复杂性，但每个部分都很小。



## 参考：关于多向量理念的结语

本章的结束语。

多向量驱动程序与单向量驱动程序在本质上没有区别。它具有相同的过滤器形式、相同的任务模式、相同的拆除顺序、相同的锁规则。变化的是数量：N 个过滤器而不是一个，N 次拆除而不是一次，N 个任务而不是一个。设计的质量取决于这 N 个部分如何干净地共存。

第 20 章的教训是，多向量处理是一种对称性练习。每个向量在结构层面看起来都与其他向量相同；每个都有自己的计数器、自己的过滤器、自己的描述。分配的代码、处理的代码、拆除的代码：它们都遍历向量并执行相同的操作 N 次。循环的简洁性使得 N 向量驱动程序易于管理；如果每个向量都是特殊的，这样的驱动程序将无法扩展。

对于本书读者和未来的读者来说，第 20 章的多向量模式是 `myfirst` 驱动程序架构的永久组成部分，也是读者工具箱中的永久工具。第 21 章将假定这一点：每队列 DMA 环、每队列完成中断、每队列 CPU 放置。这些词汇是每个高性能 FreeBSD 驱动程序共享的词汇；这些模式是内核自己的测试驱动程序使用的模式；这种规范是生产驱动程序遵循的规范。

第 20 章教授的技能不是"如何为 virtio-rng-pci 分配 MSI-X"。而是"如何设计多向量驱动程序、分配其向量、将它们放置在 CPU 上、按向量路由事件，并干净地拆除所有内容"。这项技能适用于读者将来会遇到的每一个多队列设备。
