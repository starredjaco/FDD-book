---
title: "设备树与嵌入式开发"
description: "使用设备树进行嵌入式系统的驱动开发"
partNumber: 7
partName: "精通主题：特殊场景与边缘情况"
chapter: 32
lastUpdated: "2026-04-20"
status: "complete"
author: "Edson Brandi"
reviewer: "TBD"
translator: "AI辅助翻译为简体中文"
estimatedReadTime: 195
language: "zh-CN"
---

# 设备树与嵌入式开发

## 引言

第31章训练你从外部审视你的驱动程序——站在可能试图滥用它的人的角度来看待它。你学会关注的边界对编译器来说是不可见的，但对内核来说却是非常真实的：一边是用户空间，另一边是内核内存；一个线程拥有特权，另一个没有；调用者声称的长度字段，驱动程序必须验证。那一章讨论的是谁被允许要求驱动程序做某事，以及驱动程序在同意之前应该检查什么。

第32章完全改变了视角。问题不再是*谁想让这个驱动程序运行*，而是*这个驱动程序如何找到它的硬件*。在我们一直依赖的机器上，答案很简单，你可以忽略它。PCI设备通过标准配置寄存器宣告自己的存在。一个ACPI描述的外设出现在固件传递给内核的表中。总线负责查找，内核探测每个候选者，你的驱动程序的 `probe()` 函数只需查看一个标识符并说是或否。发现工作主要是别人的问题。

在嵌入式平台上，这个假设就不成立了。一个小型ARM板不使用PCI，不携带ACPI BIOS，也没有一个固件层可以将整齐的设备表传递给内核。SoC在一个固定的物理地址上有I2C控制器，在其他三个固定地址上有三个UART，在第四个地址上有GPIObank、一个定时器、一个看门狗、一个时钟树、一个引脚复用器，以及硬件设计师以特定 arrangement焊接到板上的十几个其他外设。硅片中没有任何东西会宣告自己的存在。如果内核要将驱动程序附加到这些外设上，就必须告诉内核它们在哪里、它们是什么以及它们如何关联。

这个"东西"就是**设备树（Device Tree）**，学习如何使用它正是第32章的主题。

设备树不是驱动程序。它不像 `vfs` 或 `devfs` 那样是一个子系统意义上的东西。它是一个*数据结构*，一个文本形式的硬件描述，由引导加载程序在启动时传递给内核，然后内核遍历它来决定运行哪些驱动程序以及指向哪里。该结构有自己的文件格式、自己的编译器、自己的约定，以及嵌入式开发者在时间中逐渐学会的隐性约定。为设备树平台编写的驱动程序看起来几乎和你已经知道的驱动程序一样，只是在如何找到资源方面有几个重要的区别。这些区别就是本章的主题。

我们还将扩大视野。到目前为止，你的大多数驱动程序都在amd64上运行——这是驱动笔记本电脑、工作站和服务器的64位x86版本。这个架构不会消失，你对它的理解将继续为你服务。但FreeBSD运行的不仅仅是amd64。它运行在arm64上——这是驱动Raspberry Pi 4、Pine64、HoneyComb LX2、无数工业板和日益增长的云部分的64位ARM架构。它运行在32位ARM上，用于较旧的Pi、BeagleBone和嵌入式设备。它运行在RISC-V上——这是一个更新的开放架构，其第一个认真的FreeBSD支持在FreeBSD 13和14周期中成熟。在PC类世界之外的每一个架构上，设备树是驱动程序找到其硬件的方式。如果你想编写能在笔记本电脑之外的任何东西上运行的驱动程序，你需要知道它是如何工作的。

好消息是，当你跨越这个界限时，驱动程序的形式并不会改变太多。你的probe和attach例程仍然看起来像probe和attach例程。你的softc仍然生活在相同类型的结构中。你的生命周期仍然是相同的生命周期，包括加载和卸载、分离和清理。改变的是你调用来发现硬件地址、读取中断规范、打开时钟、拉起复位线、请求GPIO引脚的辅助函数集合。一旦你看了几个，这些辅助函数就有了家族相似性。FreeBSD源代码树在数百个地方使用它们，到本章结束时，你将一眼就能认出它们，并知道在你需要之前没用过的辅助函数时该在哪里查找。

本章分为八个部分。第1节介绍嵌入式FreeBSD世界及其运行的硬件平台。第2节解释设备树是什么，其文件如何组织，`.dts`源代码如何变成`.dtb`二进制blob，以及其中的节点和属性实际上意味着什么。第3节介绍FreeBSD对设备树的支持：`fdt(4)`框架、`ofw_bus`接口、`simplebus`枚举器，以及驱动程序用来读取属性的Open Firmware辅助函数。第4节是第一个具体的驱动程序编写部分。你将看到从probe到attach到detach的FDT感知驱动程序的形式，包括其 `ofw_compat_data` 表和对 `ofw_bus_search_compatible`、`ofw_bus_get_node` 以及属性读取辅助函数的调用。第5节转向DTB本身：如何用 `dtc` 编译 `.dts`，overlay如何工作，如何向现有板描述添加自定义节点，以及FreeBSD的构建工具如何组合在一起。第6节是关于调试的内容，包括启动时和运行时的调试，使用 `ofwdump(8)`、`dmesg` 和内核日志来找出节点为何没有匹配。第7节将各个部分组合成一个实用的GPIO驱动程序，该驱动程序从设备树获取引脚分配并切换LED。第8节是重构部分：我们将采用你在第7节构建的驱动程序，收紧其错误处理，暴露一个sysctl用于可观察性，并讨论打包嵌入式映像的样子。

沿途我们将触及每个认真的嵌入式驱动程序最终都会依赖的外设框架。`/usr/src/sys/dev/extres/` 下的时钟框架、稳压器框架和硬件复位框架是三个大的；`/usr/src/sys/dev/fdt/` 下的引脚控制框架是第四个。`/usr/src/sys/dev/gpio/` 下的GPIO框架是你读取和写入引脚的第一站。`interrupt-parent` 链将IRQ路由到中断树上，直到能够实际处理它们的控制器。设备树*overlay*，扩展名为 `.dtbo` 的文件，让你可以在启动时添加或修改节点，而不需要重建基础blob。在最高层面上，特别是在arm64上，单个内核二进制文件可以在FDT或ACPI上运行；选择两者之间的机制值得简要一看，因为它展示了为两者编写的驱动程序是如何分解的。

在我们开始之前还有最后一个框架说明。嵌入式工作从远处看可能会让人感到畏惧。词汇不熟悉，板子小而繁琐，文档比桌面平台更薄，第一次DTB不匹配导致无声启动挂起时很容易失去信心。但这些都不是远离它的理由。你将在本章中构建的核心技能——读取 `.dts` 文件并识别正在描述的内容——可以在一个漫长的下午学会。第二个技能——编写一个匹配兼容字符串并遍历所需属性的驱动程序——是你已经知道的相同类型的驱动程序编写，只有三四个新的辅助函数调用。第三个技能——构建和加载DTB并看到驱动程序附加——正是使内核工作令人愉快的那种反馈循环。到本章结束时，你将在真实的ARM板或模拟板上编写、编译、加载和观察一个FDT感知的驱动程序，你将拥有让你能够阅读任何FreeBSD嵌入式驱动程序并理解它如何找到其硬件的心理模型。

本书的其余部分将继续把驱动程序视为驱动程序，但你的工具包将已经增长。让我们开始吧。

## 读者指南：如何使用本章

第32章位于本书弧线中的一个特定位置。前面的章节假设了一个PC类机器，其中总线自行发现，驱动程序绑定到内核已经知道的设备。本章横向进入一个世界，在那里发现是驱动程序必须参与的问题。这一步比听起来更容易，但它确实需要你在思考硬件方面有一个小的转变，并且愿意阅读一些新类型的文件。

有两条阅读路径，以及一个可选的第三条路径供家里有嵌入式硬件的读者。

如果你选择**仅阅读路径**，计划花费三到四个专注的小时。你将完成本章，对设备树是什么、内核如何使用它、FDT感知驱动程序看起来像什么以及重要的FreeBSD源文件在哪里有一个清晰的心理模型。你不会输入驱动程序，但你将能够阅读一个并理解你看到的每个辅助函数调用。对于许多读者来说，这是第一次阅读的正确停止点。

如果你选择**阅读加实验路径**，计划在两到三次会话中花费七到十小时。实验围绕一个名为 `edled`（*嵌入式LED*的缩写）的小型教学驱动程序构建。在本章中，你将编写一个匹配自定义兼容字符串的最小FDT驱动程序，将其发展为一个从设备树读取其引脚号和定时器间隔的驱动程序，最后用一个整洁的detach路径和一个用于运行时可观察性的sysctl包装它。你将把一个小的 `.dts` 片段编译成 `.dtb` overlay，用FreeBSD的加载程序加载它，并在 `dmesg` 中观察驱动程序附加。这些步骤都不长；都不需要对 `make`、`sudo` 和shell有超过基本的熟悉度。

如果你有Raspberry Pi 3或4、BeagleBone、Pine64、Rock64或兼容ARM板的访问权限，你可以遵循**阅读加实验加硬件路径**。在这种情况下，`edled` 驱动程序会闪烁连接到真实引脚的真实LED，你将获得编写使物理事物发生的内核代码的满足体验。如果你没有硬件，不用担心。整个章节都可以用QEMU虚拟机模拟通用ARM平台来跟随，甚至可以在常规FreeBSD机器上进行打印模拟。你不会错过任何概念性材料。

### 先决条件

你应该对前面章节中的FreeBSD驱动程序骨架感到舒适：模块初始化和退出、`probe()` 和 `attach()`、softc分配和销毁、`DRIVER_MODULE()` 注册，以及 `bus_alloc_resource_any()` 和 `bus_setup_intr()` 的基础知识。如果这些有任何模糊之处，在本章之前简要回顾第6至14章会有所回报。本章中的辅助函数位于熟悉的 `bus_*` 辅助函数*旁边*；它们不替代它们。

你还应该对简单的FreeBSD系统管理感到舒适：用 `kldload(8)` 和 `kldunload(8)` 加载和卸载内核模块、阅读 `dmesg(8)`、编辑 `/boot/loader.conf` 以及以root身份运行命令。你将使用所有这些，但没有比前面章节已经要求的更深层次。

不需要嵌入式硬件的先前经验。如果你从未接触过ARM板，本章将教你所需的知识。如果你使用过Raspberry Pi但只是作为一个小型Linux机器，本章将给你一个关于其底层发生什么的新视角。

### 结构与节奏

第1节设置舞台：嵌入式FreeBSD在实践中是什么样子，FreeBSD支持哪些架构，你可能会遇到什么样的板子，以及为什么设备树是嵌入式平台找到的答案。第2节是最长的概念部分：它介绍设备树文件、其源代码和二进制形式、其节点-属性结构，以及你需要流利阅读它们的一小部分约定。第3节将对话带回FreeBSD具体内容，介绍 `fdt(4)` 框架、`ofw_bus` 接口和simplebus枚举器。第4节是第一个驱动程序编写部分；它从probe到attach到detach遍历FDT驱动程序的规范形式。第5节教你如何构建和修改 `.dtb` 文件以及FreeBSD的overlay系统如何工作。第6节是调试部分。第7节是实质性的工作示例：`edled` GPIO驱动程序。第8节是重构和完成部分。

在第8节之后，你将找到动手实验、挑战练习、故障排除参考、总结回顾，以及通往第33章的桥梁。

按顺序阅读各节。它们彼此构建，后面的节假设前面的节建立的词汇。如果你时间有限并想要最小的概念游览，阅读第1、2、3节，然后浏览第4节以获取其驱动程序骨架；这给了你领域的地图。

### 逐节学习

每个节覆盖主题的一个连贯部分。阅读一个，让它沉淀，然后开始下一个。如果一个节结束而一个点仍然模糊，暂停，重读结尾段落，并打开引用的FreeBSD源文件。快速查看真实代码往往能在三十秒内澄清散文只能围绕的内容。

### 保持实验驱动程序近在手边

本章使用的 `edled` 驱动程序位于本书仓库的 `examples/part-07/ch32-fdt-embedded/` 下。每个实验目录包含该步骤时驱动程序的状态，包括其 `Makefile`、简短的 `README.md`、相关的匹配DTS overlay以及任何支持脚本。克隆目录，就地工作，并在每次更改后加载每个版本。一个在 `dmesg` 中附加并切换你可以观察的引脚的内核模块是嵌入式工作中最具体的反馈循环；使用它。

### 打开FreeBSD源代码树

几个节指向真实的FreeBSD文件。本章最有用的保持打开的文件是 `/usr/src/sys/dev/fdt/simplebus.c` 和 `/usr/src/sys/dev/fdt/simplebus.h`，它们定义了每个子驱动程序绑定的简单FDT总线枚举器；`/usr/src/sys/dev/ofw/ofw_bus.h`、`/usr/src/sys/dev/ofw/ofw_bus_subr.h` 和 `/usr/src/sys/dev/ofw/ofw_bus_subr.c`，它们给你兼容性辅助函数和属性读取器；`/usr/src/sys/dev/ofw/openfirm.h`，它声明了较低级别的 `OF_*` 原语；`/usr/src/sys/dev/gpio/gpioled_fdt.c`，一个你将模仿的小型真实驱动程序；`/usr/src/sys/dev/gpio/gpiobusvar.h`，它定义了 `gpio_pin_get_by_ofw_idx()` 和其兄弟函数；以及 `/usr/src/sys/dev/fdt/fdt_pinctrl.h`，它定义了pinctrl API。当本章指向它们时打开它们。源代码是权威；本书是进入它的指南。

### 保持实验日志

继续前面章节的实验日志。对于本章，为每个实验记录简短笔记：你运行了哪些命令，哪个DTB加载了，`dmesg` 说了什么，你使用了哪个引脚，以及任何意外。嵌入式调试比大多数更需要留下书面痕迹，因为导致驱动程序不附加的变量（错过overlay、错误的phandle、交换的引脚号）容易忘记，重新发现代价高昂。

### 调整节奏

本章中的概念第二次遇到往往比第一次更容易。*phandle*、*ranges* 和 *interrupt-parent* 这些词可能会在一段时间内让你感到不适，然后突然豁然开朗。如果一个小节模糊，标记它，继续前进，然后再回来。大多数读者发现第2节（设备树格式本身）是本章最难的部分；之后，第3和第4节感觉容易，因为它们主要是*驱动程序代码*，而驱动程序代码的形式已经熟悉。

## 如何从本章获得最大收益

一些习惯将帮助你将本章的散文转化为持久的直觉。它们与帮助任何新子系统的习惯相同，针对嵌入式工作的特殊性进行了调整。

### 阅读真实源代码

学习阅读设备树文件和FDT感知驱动程序的最佳方式是阅读真实的文件。FreeBSD树附带数百个。选择你感兴趣的外设，在 `/usr/src/sys/dev/` 下找到其驱动程序，打开它并阅读。如果它顶部附近有一个 `ofw_compat_data` 表，并且 `probe()` 调用 `ofw_bus_search_compatible`，你正在看一个FDT感知驱动程序。注意它读取哪些属性。注意它如何获取资源。注意它在detach上做了什么和没做什么。

DTS方面值得同样的处理。FreeBSD树在 `/usr/src/sys/dts/` 下保留自定义板描述，在 `/usr/src/sys/contrib/device-tree/` 下保留上游Linux衍生的设备树，在 `/usr/src/sys/dts/arm/overlays/` 和 `/usr/src/sys/dts/arm64/overlays/` 下保留overlay。打开一个描述你听说过的板的 `.dts` 文件，像阅读代码一样阅读它：自上而下，注意层次结构、属性名称和评论。

### 运行你构建的内容

实验的意义在于它们以你可以观察的东西结束。当你加载模块而什么都没发生时，那也是信息；它通常意味着驱动程序没有匹配，本章将教你如何找出原因。反馈循环是整个意义所在。不要因为代码编译了就跳过加载步骤。

### 输入实验代码

`edled` 驱动程序故意很小。自己输入它会让你慢下来，足以注意到每行做什么。编写FDT样板代码的指尖记忆是值得拥有的。复制粘贴剥夺了你的那一点；即使你确定你可以从记忆中重现文件，也要抵制诱惑。

### 跟随节点

当你阅读设备树文件或 `dmesg` 启动日志而不认识一个属性时，跟随它。查看 `/usr/src/sys/contrib/device-tree/Bindings/` 中节点的绑定文档（如果存在）。在FreeBSD源代码中搜索属性名称，看看哪些驱动程序关心它。嵌入式工作充满了小型约定，每个约定一旦你在真实代码中看到它被使用就会变得明显。

### 将dmesg视为文稿的一部分

关于FDT发现几乎所有有趣的东西都出现在 `dmesg` 中，而不是shell中。当驱动程序附加时，当节点因其状态被禁用而被跳过时，当simplebus报告一个没有匹配驱动程序的子节点时，你在内核日志中找到这些消息，无处 else。在实验期间在第二个终端保持 `dmesg -a` 或 `tail -f /var/log/messages`。当它们教了一些不明显的东西时，将相关行复制到你的日志中。

### 故意破坏东西

本章中一些最有用的教训来自于观察驱动程序*失败*附加。兼容字符串中的拼写错误、错误的引脚号、禁用的状态、缺少的overlay，每一个都会在 `dmesg` 中产生不同风味的沉默。在实验室环境中通过这些失败学习将教你当同样的失败在真实工作中让你惊讶时所需的识别能力。不要只构建工作的驱动程序；故意破坏几个并看看内核说什么。

### 尽可能结对学习

嵌入式调试，像安全工作一样，受益于第二双眼睛。如果你有学习伙伴，一个人可以阅读 `.dts` 而另一个人阅读驱动程序；你可以交换视角并比较各自认为设置在做什么。对话往往会捕捉到单个读者容易滑过的小错误（交换的cell计数、误读的phandle、错误位置的中断cell）。

### 信任迭代

你不会在第一次通过时记住每个属性、每个标志、每个辅助函数。那没关系。第一次通过你需要的是主题的*形式*：原语的名称、使用它们的驱动程序的结构、当具体问题出现时要查看的地方。在你写了一个或两个FDT驱动程序后，标识符变成反射；它们不是一个记忆练习。

### 休息

嵌入式工作，像安全工作一样，认知密集。它要求你在阅读试图描述和控制它的软件时保持对物理硬件描述的头脑。两个专注的小时、一个适当的休息、另一个专注的小时几乎总是比一次坐着磨四小时更有成效。

有了这些习惯，让我们从宽泛的问题开始：什么是嵌入式FreeBSD，它对驱动程序作者的要求与桌面FreeBSD有何不同？

## 第1节：嵌入式FreeBSD系统简介

多年来，*嵌入式*这个词被使用得如此宽松，以至于值得暂停一下，说说我们在本书中用它指什么。对我们来说，嵌入式系统是一台设计来做特定工作的计算机，而不是一台通用机器。运行恒温控制循环的Raspberry Pi是嵌入式的。运行CNC控制器的BeagleBone是嵌入式的。运行防火墙设备的小型ARM盒子、运行专用传感器网关的RISC-V开发板、过去围绕MIPS SoC构建的路由器，所有这些都是嵌入式的。笔记本电脑即使很小也不是嵌入式的。服务器即使精简也不是嵌入式的。这个词关乎目的和约束，而不是大小。

从驱动程序作者的角度来看，嵌入式系统共享一些塑造工作的实际特征。硬件通常是一个带有许多片上外设的SoC，而不是带有插卡的主板。外设是固定的：它们不能被添加或移除，因为它们字面上就是硅的一部分。通常没有PCI意义上的可发现总线；外设位于已知物理地址，内核必须被告知它们在哪里。电源受限，有时严重受限。RAM有限。存储有限。启动流程简单，通常依赖像U-Boot这样的引导加载程序或小型EFI实现。用户界面（如果有）是最小的。内核启动，驱动程序附加，机器存在的运行的单个应用程序启动，这就是系统的生命。

FreeBSD运行在一个日益增长的嵌入式友好架构家族上。本章大多假设arm64，因为它是FreeBSD今天最广泛使用的嵌入式目标，但你学到的大部分直接适用于32位ARM、RISC-V，以及在较小程度上，较旧的MIPS和PowerPC平台。这些架构之间的差异对编译器和内核的最低级别代码很重要，但对于驱动程序编写，差异大多不存在。相同的FDT框架、相同的 `ofw_bus` 辅助函数、相同的simplebus枚举器，以及相同的GPIO和时钟API在所有架构上工作。你今天为Raspberry Pi编写的驱动程序，经过非常少的修改，明天就可以在RISC-V SiFive HiFive Unmatched上构建和运行。

### 嵌入式FreeBSD是什么样子

想象一个运行FreeBSD 14.3的Raspberry Pi 4。板子有一个四核ARM Cortex-A72 CPU、4GB RAM、一个PCIe根复合体（上面挂着千兆以太网控制器和USB主机控制器）、一个用于存储的SD卡槽、一个Broadcom VideoCore VI GPU、一个HDMI输出、四个USB端口、一个40引脚GPIO头，以及片上UART、SPI总线、I2C总线、脉宽调制器和定时器。大多数这些外设位于BCM2711 SoC内的固定内存映射地址。少数，如USB和以太网，挂在SoC的内部PCIe控制器上。SD卡由SoC内部的一个 speak SD协议的主机控制器驱动。

当你给板子上电时，GPU核心上的一个小型固件读取SD卡，找到FreeBSD EFI引导加载程序，并将控制权交给它。EFI加载程序读取 `/boot/loader.conf`，加载FreeBSD内核，加载描述硬件的设备树blob，加载任何修改描述的overlay blob，并跳转到内核。内核启动，将simplebus附加到树的根节点，遍历树的子节点，将驱动程序匹配到节点，将驱动程序附加到其硬件，系统启动。当你看到登录提示时，你的文件系统已经由SD主机驱动程序从SD卡挂载，你的网络接口已经由以太网驱动程序启动，你的USB设备已经探测并可用，普通用户空间工具（`ps`、`dmesg`、`sysctl`、`kldload`）的行为与在笔记本电脑上完全相同。

最后那段话对于PC来说不会是真的。在PC上，固件是BIOS或UEFI，外设在PCI上，内核不需要单独的描述硬件的blob，因为PCI总线本身描述其子节点。arm64世界没有这些奢侈品。它有设备树替代。

### 为什么嵌入式平台依赖设备树

嵌入式世界选择设备树是因为问题空间。一个小型SoC有数十个片上外设。每个SoC变体、每个板修订、每个硬件工程师做出的集成选择都塑造了哪些外设被启用、它们使用哪些引脚、它们的时钟如何接线以及它们的中断优先级是什么。野中有数千个不同的SoC，以及数万个不同的板。单个内核二进制文件不能承担将所有这些变体编译进去的成本。也不存在一个神奇的总线协议它可以用它来*询问*硬件板上有什么；硬件不知道，大多数它不能回答。

旧的解决方案，嵌入式Linux世界在设备树之前使用的，叫做*板文件*。每个板有一个编译进内核的C源文件，充满了描述外设、其地址、其中断和其时钟的静态结构。为五个板设计的内核有五个板文件。为五十个板设计的内核有五十个，每个都有自己的静态描述，每个需要每次板的硬件改变时重建内核。这种方法不能扩展。每个板修订都是一个内核发布。

设备树是替代板文件的方法。核心思想优雅：不是将硬件描述编码在内核内的C中，而是将其编码在一个生活在内核之外的单独文本文件中，将该文件编译成一个紧凑的二进制blob，让引导加载程序在启动时将blob传递给内核。内核然后读取blob，决定附加哪些驱动程序，并将描述其硬件的blob部分交给每个驱动程序。单个内核二进制文件现在可以在给定其DTB的任何板上运行。改变引脚分配或添加外设的板修订需要新的DTB，而不是新的内核。

这种方法起源于1990年代的IBM和Apple PowerPC世界，那里概念叫做*Open Firmware*，树是固件本身的一部分。现代扁平设备树（FDT）格式是其后代，经过简化和重新设计，以适应不携带整个Forth解释器的引导加载程序和内核，就像Open Firmware那样。在FreeBSD中你会看到两个名字。框架叫做 `fdt(4)`，辅助函数仍然位于 `sys/dev/ofw/` 下，因为那个OpenFirmware血统，读取属性的函数命名为 `OF_getprop`、`OF_child`、`OF_hasprop`，原因相同。当你在FreeBSD源代码中阅读 *Open Firmware*、*OFW* 和 *FDT* 时，它们都指同一传统的部分。

### 架构概述：FreeBSD on ARM、RISC-V和MIPS

FreeBSD 14.3支持几个嵌入式友好架构。最活跃使用的两个是arm64（ARM文档中称为AArch64的64位ARM架构）和armv7（带硬件浮点的32位ARM，有时写成 `armv7` 或 `armhf`）。RISC-V支持在FreeBSD 13和14周期中成熟，现在在真实板如SiFive HiFive Unmatched和VisionFive 2上运行。MIPS支持存在很长时间并驱动许多较旧的路由器和嵌入式设备；它在最近发布中已从基础系统中移除，所以本章不会详述它，但你为ARM和RISC-V学到的技能如果你回退到遗留MIPS平台会直接转移。

所有这些架构都使用设备树来描述不可发现的硬件。启动流程细节不同，但形式相同：某个阶段零固件启动CPU，某个阶段一引导加载程序（U-Boot常见；在arm64上EFI加载程序日益标准）加载内核和DTB，内核带着树接管。在arm64上，FreeBSD EFI加载程序干净地处理这个过程，树可以从磁盘上的文件（`/boot/dtb/*.dtb`）或，在服务器级硬件上，从固件本身来。在armv7板上，U-Boot通常提供DTB。在RISC-V上，情况是OpenSBI、U-Boot和EFI的混合，取决于板。

这些差异对驱动程序作者大多不重要。当你的驱动程序运行时内核看到的树是设备树，它给你的辅助函数是 `OF_*` 和 `ofw_bus_*`，你为它编写的驱动程序在使用相同框架的架构之间是可移植的。

### 典型限制：无ACPI、有限总线、电源约束

值得列举塑造驱动程序工作的嵌入式平台的限制，这样它们不会让你意外。

**通常无ACPI。** ACPI是大多数PC用来描述其不可发现硬件的固件到OS接口。它包含表、一个叫做AML的字节码语言和一个长规范。ARM服务器有时使用ACPI，FreeBSD在arm64上通过ACPI子系统支持该路径，但小型和中型嵌入式板几乎总是使用FDT。少数高端arm64系统可能同时携带ACPI和FDT描述，让固件在两者之间选择；FreeBSD可以处理任一，在arm64上有一个运行时开关决定附加哪个总线。对大多数嵌入式驱动程序的实际后果是你为FDT编写而不担心ACPI。我们将在第3节回到双支持案例。

**片上SoC外设无PCI式发现。** PCI或PCIe设备通过标准化配置空间中的供应商和设备ID宣告自己。内核扫描总线，找到设备，并派发给声称ID的驱动程序。ARM芯片上的片上外设没有那个宣告机制。内核知道UART在Raspberry Pi 4的地址 `0x7E201000` 的唯一方式是DTB说这样。这改变了驱动程序作者的心理模型：你不等待总线交给你一个探测的设备；你等待simplebus（或等效物）在树中找到你的节点并用那个节点的上下文派发你的probe。

**电源约束重要。** 嵌入式板可能在电池、小型USB适配器或以太网供电馈线上运行。一个在设备空闲时让时钟运行，或者在超过外设需要后保持稳压器启用的驱动程序，会损害整个系统。FreeBSD提供时钟和稳压器框架正是为了让驱动程序能在适当时候关闭东西。我们将在第3、4、7节触及它们。

**有限的RAM和存储。** 嵌入式板通常有256 MB到8 GB的RAM，以及几百兆字节到几十GB的存储。一个大量分配或在每个事件向控制台打印一屏调试输出的驱动程序会消耗系统可能没有的资源。为你预期面对的约束编写。

**更简单的启动流程，好或坏。** 嵌入式板上的启动过程通常比PC上更短更不宽容。如果内核找不到根文件系统，你可能没有一个方便的救援环境。如果DTB错误且正确的中断控制器没有附加，系统会无声挂起。这是嵌入式工作受益于工作调试电缆、串行控制台和进行小型、可测试更改而非英雄式更改的习惯的主要原因。

### 启动流程：从固件到驱动程序附加

为了使移动部件具体化，让我们在代表性arm64板上从上电到第一个驱动程序附加遍历发生的事情。细节因板而异，但形式一致。

1. 上电：第一个CPU核心从烧录到SoC中的启动ROM开始执行代码。这个ROM超出了你的控制；其工作是加载下一阶段。
2. 启动ROM从固定位置（通常是SD卡、eMMC或SPI闪存）读取阶段一加载程序。在Raspberry Pi板上这是VideoCore固件；在通用arm64板上通常是U-Boot。
3. 阶段一加载程序进行平台启动：它配置DRAM，设置早期时钟，初始化UART用于调试输出，并加载下一阶段。在安装FreeBSD的板上，下一阶段通常是FreeBSD EFI加载程序（`loader.efi`）。
4. FreeBSD EFI加载程序从启动媒体的ESP分区读取其配置，咨询 `/boot/loader.conf`，并加载三个东西：内核本身、`/boot/dtb/` 的DTB，以及 `/boot/dtb/overlays/` 的 `fdt_overlays` tunable列出的任何overlay。
5. 加载程序将控制权交给内核，连同指向加载的DTB的指针。
6. 内核启动。其机器依赖启动代码解析DTB以找到内存映射、CPU拓扑和树的根节点。基于根节点的顶层 `compatible` 字符串，arm64决定运行FDT路径还是ACPI路径。
7. 在FDT路径上，内核将 `ofwbus` 附加到树根，将 `simplebus` 附加到 `/soc` 节点（或该板上的等效物）。Simplebus遍历其子节点并为每个具有有效兼容字符串的节点创建 `device_t`。
8. 对于每个这些 `device_t`s，内核运行常规newbus探测循环。每个通过 `DRIVER_MODULE(mydrv, simplebus, ...)` 注册的驱动程序都有机会探测。其 `probe()` 返回最佳匹配的驱动程序获胜并调用其 `attach()`。驱动程序然后读取属性、分配资源并启动。
9. 附加过程对嵌套总线递归（`/soc` 的 `simple-bus` 子节点、带有自己子设备的I2C控制器、带有自己引脚消费者的GPIO bank），产生 `dmesg` 和 `devinfo -r` 中可见的完整设备树。
10. 当init运行时，系统启动所需的设备（UART、SD主机、以太网、USB、GPIO）都已附加，用户空间启动。

那个序列是本章每个驱动程序操作的背景。当你编写FDT感知驱动程序时，你正在编写为你的特定节点在第8步运行的代码。周围的机械无论有没有你都会运行；你控制的部分是读取你的节点和分配你的资源。

### 亲自查看的地方

看到真实FDT基础启动的最快方式是在Raspberry Pi 3或4上运行FreeBSD 14.3。映像可以从FreeBSD项目在标准arm64下载区域获得，设置有良好文档。如果你没有硬件，第二个最快方式是使用QEMU的 `virt` 平台，它模拟带有小型FDT描述外设集的通用arm64机器。来自普通FreeBSD arm64发布的内核和加载程序可以在里面运行。本章后面的实验笔记中有一个样本QEMU调用。

第三个选项，即使在amd64工作站上也很有用，是阅读 `.dts` 文件和消费它们的FreeBSD驱动程序。打开 `/usr/src/sys/contrib/device-tree/src/arm/bcm2835-rpi-b.dts` 或 `/usr/src/sys/contrib/device-tree/src/arm64/broadcom/bcm2711-rpi-4-b.dts`。自上而下跟随节点结构。注意顶层节点有一个命名板的 `compatible` 属性。注意子节点如何描述CPU、内存、时钟控制器、中断控制器、外设总线。然后在FreeBSD树中打开 `/usr/src/sys/arm/broadcom/bcm2835/` 并查看消费这些节点的驱动程序。你将开始看到描述和代码如何相遇。

### 总结本节

第1节设置了场景。嵌入式FreeBSD不是项目中的一个异国情调的角落；在不像PC的平台上，它就是项目本身。嵌入式工作推动你走向的架构——将硬件视为SoC上的一组固定外设而不是总线上的可发现设备——正是设备树支持的架构。设备树是板设计师知道的硬件描述和内核运行的驱动程序代码之间的桥梁。本章其余部分是关于学习那个桥梁，足以自信地走过去。

在下一节，我们将放慢速度，阅读设备树文件本身。在我们能够编写消费它们的驱动程序之前，我们需要知道它们如何结构化、它们的属性意味着什么，以及 `.dts` 源代码如何变成内核实际看到的 `.dtb` 二进制。
## 第2节：什么是设备树？

设备树是一个以树状节点组织的硬件文本描述，每个节点代表一个设备或总线，每个节点携带命名的属性来描述其地址、中断、时钟、与其他节点的关系，以及驱动程序可能需要知道的任何其他信息。这个描述就是嵌入式世界用来替代PCI枚举或ACPI表的东西。在嵌入式FreeBSD世界中，你将花费大量时间阅读、编写和推理这些文件。越早让它们变得亲切，后面每一节就越容易。

最好的开始方式是看一个例子。

### 最小设备树

这是一个最小的有趣设备树源文件，`.dts` 格式：

```dts
/dts-v1/;

/ {
    compatible = "acme,trivial-board";
    #address-cells = <1>;
    #size-cells = <1>;

    chosen {
        bootargs = "console=ttyS0,115200";
    };

    memory@80000000 {
        device_type = "memory";
        reg = <0x80000000 0x10000000>;
    };

    uart0: serial@10000000 {
        compatible = "ns16550a";
        reg = <0x10000000 0x100>;
        interrupts = <5>;
        clock-frequency = <24000000>;
        status = "okay";
    };
};
```

这是一个完整的、有效的、虽然很小的设备树。它描述了一个名为 `acme,trivial-board` 的虚构板子，该板子在物理地址 `0x80000000` 有256 MB RAM，在 `0x10000000` 有一个16550兼容的UART，该UART传递中断号5并以24 MHz运行。即使语法还不熟悉，其意图也是可读的：文件是一个硬件描述，以一种紧凑的领域专用格式编写。

让我们逐段解析。

### 树结构

第一行 `/dts-v1/;` 是一个强制性指令，声明DTS语法版本。你将编写或阅读的每个FreeBSD DTS文件都以它开头。它之前的任何内容都不是有效的DTS文件；把它当作shell脚本顶部的 `#!/bin/sh` 来对待。

文件的其余部分包含在一个以 `/ {` 开始、以 `};` 结束的单一块中。那个外层块是树的**根节点**。每个设备树只有一个根。它的子节点是外设和子总线；它们的子节点是更多的外设和子总线；依此类推。

节点通过**名称**标识。像 `serial@10000000` 这样的名称由基本名称（`serial`）和**单元地址**（`10000000`，即节点第一个寄存器区域的起始地址，以十六进制书写，不带 `0x` 前缀）组成。单元地址是一个约定，不是硬性要求；它的存在使得具有相同基本名称的节点可以被区分（你可以在不同地址有多个 `serial` 节点），同时也作为节点描述内容的可读提示。

**标签**，如 `uart0: serial@10000000` 中的 `uart0:`，是一个句柄，让树的其他部分可以引用这个节点。标签让你在文件的其他地方写 `&uart0` 来表示*标记为 `uart0` 的节点*。我们将在overlay部分使用标签。

节点包含**属性**。属性是名称后跟等号、值和分号：

```dts
compatible = "ns16550a";
```

一些属性没有值；它们仅作为标志存在：

```dts
interrupt-controller;
```

属性的值可以是字符串（`"ns16550a"`）、字符串列表（`"brcm,bcm2711", "brcm,bcm2838"`）、尖括号内的整数列表（`<0x10000000 0x100>`）、对其他节点的引用列表（`<&gpio0>`），或二进制字节串（`[01 02 03]`）。大多数日常属性是字符串、字符串列表或整数列表。树使用32位*cell*作为其基本整数单位；`<...>` 内的整数是cell，64位值表示为两个连续的cell（高位、低位）。

让我们回到最小示例，用新的眼光阅读每个属性。

### 阅读最小示例

根节点有三个属性：

```dts
compatible = "acme,trivial-board";
#address-cells = <1>;
#size-cells = <1>;
```

**`compatible`** 属性是根节点（以及任何节点）告诉世界它是什么的方式。它是设备树中最重要的属性。驱动程序通过它进行匹配。值是一个带有供应商前缀的字符串（`"acme,trivial-board"`），或者更常见的是，一个按特异性递减排列的字符串列表。例如，Raspberry Pi 4的根节点可能是 `compatible = "raspberrypi,4-model-b", "brcm,bcm2711";` 第一个字符串说"确切是这个板子"；第二个字符串说"属于使用BCM2711芯片的板子家族"。知道特定板子的驱动程序可以匹配第一个；只知道芯片的驱动程序可以匹配第二个。DTS规范称之为*兼容性列表*，FreeBSD和Linux都尊重它。

根节点的 **`#address-cells`** 和 **`#size-cells`** 属性描述子节点在 `reg` 属性中使用多少个32位cell来表示地址和大小。在32位板的根节点，两者通常都是1。在具有超过4 GB可寻址内存的64位板上，两者都是2。当你看到内存节点下的 `reg = <0x80000000 0x10000000>;` 时，你从父节点的cell计数知道这是一个地址cell和一个大小cell，这意味着区域在 `0x80000000`，大小为 `0x10000000`。如果 `#address-cells` 是2，区域将写成 `reg = <0x0 0x80000000 0x0 0x10000000>;`。

**`chosen`** 节点是根节点硬件子节点的一个特殊兄弟。它携带引导加载程序想传递给内核的参数：启动参数、控制台设备，有时是initrd位置。FreeBSD读取 `/chosen/bootargs` 并用它来填充内核环境。

**`memory@80000000`** 节点描述一块物理内存区域。内存节点携带 `device_type = "memory"` 和一个给出其范围的 `reg` 属性。FreeBSD的早期启动读取这些来构建其物理内存映射。

**`serial@10000000`** 节点是有趣的一个。它的 `compatible = "ns16550a"` 告诉内核*这是一个ns16550a UART*，这是一种非常常见的PC式串口芯片。它的 `reg = <0x10000000 0x100>` 说*我的寄存器位于物理地址 `0x10000000`，占用 `0x100` 字节*。它的 `interrupts = <5>` 说*我向我的中断父控制器传递中断号5*。它的 `clock-frequency = <24000000>` 说*我的参考时钟运行在24 MHz，所以除数应该从这里计算*。它的 `status = "okay"` 说*我已启用*；如果它说 `"disabled"`，驱动程序将跳过此节点。

这基本上就是设备树做的事情：它用一些属性描述每个外设，这些属性的含义由叫做**绑定（bindings）**的约定定义。UART的绑定告诉你UART节点应该携带什么属性。I2C控制器的绑定告诉你I2C控制器节点应该携带什么属性。依此类推。绑定单独文档化，FreeBSD树在 `/usr/src/sys/contrib/device-tree/Bindings/` 下附带一个大型绑定库。

### 源码与二进制：.dts、.dtsi 和 .dtb

设备树文件有三种容易混淆的文件类型。

**`.dts`** 是主要源码形式。`.dts` 文件描述整个板子或平台，是你输入给编译器的文件。

**`.dtsi`** 是包含片段，`i` 代表 *include*。一个典型的SoC系列有一个大型 `.dtsi` 描述SoC本身（其中断控制器、时钟树、片上外设），每个使用该SoC的板子有一个小型 `.dts`，通过 `#include` 包含 `.dtsi`，然后描述板子特定的附加内容（焊接到板上的外部设备、引脚配置、chosen节点）。你会在 `/usr/src/sys/contrib/device-tree/src/arm/` 和 `/usr/src/sys/contrib/device-tree/src/arm64/` 下看到许多 `.dtsi` 文件。

**`.dtb`** 是编译后的二进制形式。内核和引导加载程序处理 `.dtb` 文件，而不是 `.dts` 文件。`.dtb` 是 `dtc` 编译器处理 `.dts` 源码的输出。它紧凑，没有空白或注释，设计为由引导加载程序在几千字节的代码中解析。Raspberry Pi 4的 `.dtb` 文件通常约30 KB。

还有第四种不太常见的类型：

**`.dtbo`** 是编译后的*overlay*。Overlay是在加载时修改现有基础 `.dtb` 的片段：它们可以启用或禁用节点、添加新节点或更改属性。它们是FreeBSD和许多Linux发行版用来让用户自定义标准DTB而无需重建的机制。`.dtbo` 文件从 `.dtso`（device-tree-source-overlay）文件编译，通过引导加载程序的 `fdt_overlays` tunable加载。我们将在第5节遇到它们。

当你使用DTS工作时，你几乎总是在编写 `.dts` 或 `.dtso`。你用设备树编译器 `dtc` 编译它们，在FreeBSD中 `dtc` 位于 `devel/dtc` port中，安装为 `/usr/local/bin/dtc`。FreeBSD内核构建系统通过 `/usr/src/sys/tools/fdt/` 下的脚本调用 `dtc`，特别是 `make_dtb.sh` 和 `make_dtbo.sh`。

### 节点、属性、地址和Phandle

一些额外的概念反复出现，值得一次性命名。

**节点**是层次结构的单元。每个节点有一个名称，可选地有标签，可能有单元地址，以及零个或多个属性。节点嵌套；节点的子节点在其包围的花括号内描述。

**属性**是键值对。键是字符串。值按约定类型化：`compatible` 是字符串列表，`reg` 是整数列表，`status` 是字符串，`interrupts` 是整数列表（其长度取决于中断父控制器），等等。

节点名称中的**单元地址**（`serial@10000000`）反映 `reg` 属性的第一个cell。DTC编译器在它们不一致时会发出警告。你应该保持两者一致。

**`reg`** 属性描述设备的内存映射寄存器区域。其格式为 `<address size address size ...>`，每个地址-大小对是一个连续区域。大多数简单外设只有一个区域。有些有几个（例如，一个有主寄存器区域和单独中断寄存器块的外设）。

**地址cell和大小cell** 是存在于父节点上的 `#address-cells` 和 `#size-cells` 属性对，描述子节点中 `reg` 的格式。一个具有 `#address-cells = <1>; #size-cells = <1>;` 的SoC总线让子节点每个使用一个cell表示地址和大小。I2C总线通常有 `#address-cells = <1>; #size-cells = <0>;`，因为I2C子节点有地址但没有大小。

**中断**由一个或多个属性描述，取决于使用的风格。旧风格是 `interrupts = <...>;`，其cell由中断父控制器的约定解释。在基于ARM GIC的平台上，这是一个三cell形式：中断类型、中断号、中断标志。新风格，在内核中与旧风格混合使用，是 `interrupts-extended = <&gic 0 15 4>;`，它显式命名中断父控制器。无论哪种方式，cell告诉内核设备引发哪个硬件中断以及在什么条件下。

**phandle** 是编译器分配给每个节点的唯一整数。Phandle让其他节点引用此节点。当你写 `<&gpio0>` 时，编译器替换为标记为 `gpio0` 的节点的phandle。当你写 `<&gpio0 17 0>` 时，你传递三个cell：`gpio0` 的phandle、引脚号 `17` 和标志cell `0`。phandle之后的cell的含义由*提供者*的绑定定义。这是GPIO消费者、时钟消费者、复位消费者和中断消费者与它们的提供者通信的模式：第一个cell命名提供者，后续的cell说明哪个资源以及如何使用。

**`status`** 属性是一个小但关键的属性。`status = "okay";` 的节点已启用，驱动程序将探测它。`status = "disabled";` 的节点被跳过。Overlay经常切换此属性来打开或关闭外设，而不移除节点。FreeBSD的 `ofw_bus_status_okay()` 是当节点状态为okay时返回true的辅助函数。

**`label`** 和 **`alias`** 机制让你通过短名称而不是路径引用节点。像 `uart0:` 这样的标签是文件局部的句柄；在特殊的 `/aliases` 节点下定义的别名（`serial0 = &uart0;`）是内核可见的名称。FreeBSD对某些设备（如控制台）使用别名。

这就是你阅读典型设备树所需的大部分内容。一些更特殊的部分出现在特定绑定中（例如，时钟框架消费者的 `clock-names` 和 `clocks`，hwreset消费者的 `reset-names` 和 `resets`，DMA引擎消费者的 `dma-names` 和 `dmas`，引脚控制的 `pinctrl-0`、`pinctrl-names`），但它们都遵循相同的*命名索引列表*模式。

### 更实际的示例

为了给你一个真实SoC级别片段的感觉，这里有一个来自BCM2711（Raspberry Pi 4）描述的简化节点。完整文件位于 `/usr/src/sys/contrib/device-tree/src/arm/bcm2711.dtsi`。

```dts
soc {
    compatible = "simple-bus";
    #address-cells = <1>;
    #size-cells = <1>;
    ranges = <0x7e000000 0x0 0xfe000000 0x01800000>,
             <0x7c000000 0x0 0xfc000000 0x02000000>,
             <0x40000000 0x0 0xff800000 0x00800000>;
    dma-ranges = <0xc0000000 0x0 0x00000000 0x40000000>;

    gpio: gpio@7e200000 {
        compatible = "brcm,bcm2711-gpio", "brcm,bcm2835-gpio";
        reg = <0x7e200000 0xb4>;
        interrupts = <GIC_SPI 113 IRQ_TYPE_LEVEL_HIGH>,
                     <GIC_SPI 114 IRQ_TYPE_LEVEL_HIGH>;
        gpio-controller;
        #gpio-cells = <2>;
        interrupt-controller;
        #interrupt-cells = <2>;
    };

    spi0: spi@7e204000 {
        compatible = "brcm,bcm2835-spi";
        reg = <0x7e204000 0x200>;
        interrupts = <GIC_SPI 118 IRQ_TYPE_LEVEL_HIGH>;
        clocks = <&clocks BCM2835_CLOCK_VPU>;
        #address-cells = <1>;
        #size-cells = <0>;
        status = "disabled";
    };
};
```

自上而下阅读：

- `soc` 节点是片上外设总线。其 `compatible = "simple-bus"` 是告诉FreeBSD在此附加simplebus驱动程序的魔法令牌。
- 其 `ranges` 属性定义了从总线地址（CPU外设互连内部使用的"本地"地址，从 `0x7E000000` 开始）到CPU物理地址（从 `0xFE000000` 开始）的地址转换。FreeBSD读取此信息并在映射子节点 `reg` 属性时应用它。
- `gpio` 节点是GPIO控制器。它声明两个中断，将自己声明为gpio-controller（以便其他节点可以引用它），并每个GPIO引用使用两个cell（第一个cell是引脚号，第二个是标志字）。
- `spi0` 节点是位于总线地址 `0x7E204000` 的SPI控制器。在基础描述中它是 `status = "disabled"`，意味着它不会附加，直到overlay启用它。

每个嵌入式板描述大致都是这样：一个片上外设的树，每个都有一个兼容字符串标识要绑定什么驱动程序，一个 `reg` 表示其内存映射寄存器，一个 `interrupts` 表示其IRQ线，可能还有对时钟、复位、稳压器和引脚的引用。

### DTB如何加载

为完整性起见，了解DTB如何从磁盘到内核在启动时是有帮助的。

在运行FreeBSD 14.3的arm64板上，典型流程是：

1. EFI固件或引导加载程序从ESP读取FreeBSD EFI加载程序。
2. FreeBSD加载程序从 `/boot/kernel/kernel` 加载内核，从 `/boot/dtb/<board>.dtb` 加载DTB。DTB的文件名基于SoC系列选择。
3. 如果 `/boot/loader.conf` 设置了 `fdt_overlays="overlay1,overlay2"`，加载程序读取 `/boot/dtb/overlays/overlay1.dtbo` 和 `/boot/dtb/overlays/overlay2.dtbo`，将它们应用到内存中的基础DTB，并将合并结果交给内核。
4. 内核将合并的DTB作为其权威硬件描述。

在U-Boot驱动的板子上（armv7常见），流程类似，但加载程序是U-Boot本身。U-Boot的环境变量 `fdt_file` 和 `fdt_addr` 告诉它加载哪个DTB以及放在哪里。当U-Boot最终执行 `bootefi` 或 `booti` 时，它将DTB传递给FreeBSD加载程序或直接传递给内核。

在固件中携带FDT的EFI系统上（小型板子罕见，使用Server Base System Architecture的ARM服务器常见），固件将DTB存储为EFI配置表，内核从那里读取它。

对于驱动程序作者来说，启动的细节大多数时候不重要。重要的是，当你的驱动程序的probe被调用时，树已经被加载、解析并呈现给内核；无论它如何到达，你都用相同的 `OF_*` 辅助函数读取它。

### 总结本节

第2节介绍了设备树作为硬件描述语言。你已经看到了一个最小示例，遇到了节点和属性的核心概念，了解了 `.dts`、`.dtsi`、`.dtb` 和 `.dtbo` 文件的区别，并走过了SoC描述的实际片段。你还知道了DTB在启动时如何到达内核。

你还*不*知道的是FreeBSD的内核如何在加载树后消费它：哪个子系统附加到它，驱动程序调用哪些辅助函数来读取属性，以及驱动程序的 `probe()` 和 `attach()` 如何找到它们的节点。那是第3节的内容。

## 第3节：FreeBSD的设备树支持

FreeBSD通过一个设计早于FDT本身的框架处理设备树。该框架以Open Firmware命名，在整个源代码树中缩写为 **OFW**，因为它最初服务的API是为使用真正Open Firmware规范的PowerPC Mac和IBM系统设计的。当ARM世界在2000年代末标准化扁平设备树时，FreeBSD将FDT映射到相同的内部API。因此，FreeBSD 14.3中的驱动程序调用相同的 `OF_getprop()`，无论它运行在PowerPC Mac上、带有FDT blob的ARM板上，还是带有FDT blob的RISC-V板上。底层实现不同；上层接口统一。

本节介绍你需要知道的框架组件：实践中使用的 `fdt(4)` 接口、驱动程序在原始 `OF_*` 原语之上调用的 `ofw_bus` 辅助函数、枚举子设备的 `simplebus(4)` 总线驱动程序，以及你将经常使用的属性读取惯用法。到本节结束时，你将知道FreeBSD侧已经存在哪些代码以及它们在哪里；第4节将在其上构建一个驱动程序。

### fdt(4)框架概述

`fdt(4)` 是内核的扁平设备树支持。它提供解析二进制 `.dtb`、遍历它以查找节点、提取属性、应用overlay并通过 `OF_*` API呈现结果的代码。你可以把 `fdt(4)` 想象成下半部分，`ofw_bus` 是上半部分，`OF_*` 函数横跨两者。

实现OFW接口FDT侧的代码位于 `/usr/src/sys/dev/ofw/ofw_fdt.c`。它是 `ofw_if.m` kobj接口的特定实例。当内核调用 `OF_getprop()` 时，调用通过接口并最终到达FDT实现，后者遍历扁平blob。在PowerPC Mac上，它最终到达真正的Open Firmware实现；上面的驱动程序不需要知道或关心。

作为驱动程序作者，你几乎从不直接接触 `ofw_fdt.c`。你使用上面一层的辅助函数。

### OF_*：原始属性读取器

驱动程序调用的最低级API是 `/usr/src/sys/dev/ofw/openfirm.h` 中声明的 `OF_*` 函数族。你最常用的是一小部分。

`OF_getprop(phandle_t node, const char *prop, void *buf, size_t len)` 将属性的原始字节读入调用者提供的缓冲区。成功时返回读取的字节数，失败时返回 `-1`。缓冲区必须足够大以容纳预期长度。

`OF_getencprop(phandle_t node, const char *prop, pcell_t *buf, size_t len)` 读取其cell为大端序的属性，并在复制时将它们转换为主机字节序。几乎所有包含整数的属性都应该用此变体而不是 `OF_getprop()` 读取。

`OF_getprop_alloc(phandle_t node, const char *prop, void **buf)` 读取未知长度的属性。内核分配缓冲区并通过第三个参数返回指针。使用完毕后，调用 `OF_prop_free(buf)` 释放它。

`OF_hasprop(phandle_t node, const char *prop)` 如果命名属性存在则返回非零值，否则返回零。适用于可选属性，仅存在就有意义的情况。

`OF_child(phandle_t node)` 返回节点的第一个子节点。`OF_peer(phandle_t node)` 返回下一个兄弟节点。`OF_parent(phandle_t node)` 返回父节点。组合使用，它们让你遍历树。

`OF_finddevice(const char *path)` 返回给定路径（如 `"/chosen"` 或 `"/soc/gpio@7e200000"`）处节点的phandle。大多数驱动程序不需要这个，因为框架已经将它们的节点交给它们了。

`OF_decode_addr(phandle_t dev, int regno, bus_space_tag_t *tag, bus_space_handle_t *hp, bus_size_t *sz)` 是一个便利例程，用于非常早期的代码（主要是串行控制台驱动程序），为给定节点的寄存器 `regno` 设置总线空间映射，而不通过newbus。普通驱动程序使用 `bus_alloc_resource_any()` 代替，它通过probe期间设置的资源列表读取 `reg` 属性。

这些原语是基础。在实践中，你会通过稍微更方便的 `ofw_bus_*` 辅助函数间接调用它们，但上面这些是那些辅助函数内部使用的，在阅读真实驱动程序代码时值得认识它们。

### ofw_bus：兼容性辅助函数

FDT感知驱动程序做的最常见的事情是问：*这个设备与我知道如何驱动的东西兼容吗？* FreeBSD在 `OF_getprop` 之上提供了一小层辅助函数，使这些检查变得惯用。它们位于 `/usr/src/sys/dev/ofw/ofw_bus.h` 和 `/usr/src/sys/dev/ofw/ofw_bus_subr.h`，实现在 `/usr/src/sys/dev/ofw/ofw_bus_subr.c`。

值得了解的辅助函数，按你将遇到的顺序：

`ofw_bus_get_node(device_t dev)` 返回与 `device_t` 关联的phandle。它实现为一个内联函数，调用父总线的 `OFW_BUS_GET_NODE` 方法。对于simplebus的子设备，它返回产生此设备的DTS节点的phandle。

`ofw_bus_status_okay(device_t dev)` 如果节点的 `status` 属性不存在、为空或为 `"okay"` 则返回1；否则返回0。每个FDT感知的probe都应该在顶部调用它来跳过禁用的节点。

`ofw_bus_is_compatible(device_t dev, const char *string)` 如果节点 `compatible` 属性的任何条目完全等于 `string` 则返回1。简短、精确，是驱动程序只想要一个兼容字符串时的常用工具。

`ofw_bus_search_compatible(device_t dev, const struct ofw_compat_data *table)` 遍历驱动程序提供的表，如果其兼容字符串中的任何一个在节点的 `compatible` 列表中，则返回匹配的条目。这是支持多个芯片的驱动程序注册其兼容性的标准方式。表是一个条目数组，每个条目包含一个字符串和一个驱动程序可以用来记住匹配了哪个芯片的 `uintptr_t` cookie；表以字符串为 `NULL` 的哨兵条目结束。我们将在第4节看到完整的模式。

`ofw_bus_has_prop(device_t dev, const char *prop)` 是 `OF_hasprop(ofw_bus_get_node(dev), prop)` 的便利包装。

`ofw_bus_get_name(device_t dev)`、`ofw_bus_get_compat(device_t dev)`、`ofw_bus_get_type(device_t dev)` 和 `ofw_bus_get_model(device_t dev)` 从节点返回相应的字符串，不存在则返回 `NULL`。

这些是面包和黄油辅助函数。你会在几乎所有每个FDT感知驱动程序的probe和attach例程顶部看到它们。

### simplebus：默认枚举器

simplebus驱动程序是让所有这些在实践中工作的组件。它位于 `/usr/src/sys/dev/fdt/simplebus.c`，头文件在 `/usr/src/sys/dev/fdt/simplebus.h`。Simplebus有两个工作。

它的第一个工作是枚举子设备。当内核将simplebus附加到一个 `compatible` 包含 `"simple-bus"` 的节点（或因历史原因 `device_type` 为 `"soc"` 的节点）时，simplebus遍历该节点的子节点，为每个有 `compatible` 属性的子节点创建一个 `device_t`，并将它们送入newbus。这就是让你的驱动程序的probe被调用的原因；simplebus是你的驱动程序通过 `DRIVER_MODULE(mydrv, simplebus, ...)` 注册的总线。

它的第二个工作是 将子地址转换为CPU物理地址。父节点的 `ranges` 属性编码了子节点 `reg` 属性中出现的总线本地地址如何映射到CPU物理地址。Simplebus在 `simplebus_fill_ranges()` 中读取 `ranges`，并在设置每个子节点的资源列表时应用它，所以当你的驱动程序请求内存资源时，区域已经在CPU物理空间中。

决定simplebus是否应该附加到给定节点的核心probe代码位于 `/usr/src/sys/dev/fdt/simplebus.c` 的顶部附近。这里是它，去掉注释以简洁：

```c
if (!ofw_bus_status_okay(dev))
    return (ENXIO);

if (ofw_bus_is_compatible(dev, "syscon") ||
    ofw_bus_is_compatible(dev, "simple-mfd"))
    return (ENXIO);

if (!(ofw_bus_is_compatible(dev, "simple-bus") &&
      ofw_bus_has_prop(dev, "ranges")) &&
    (ofw_bus_get_type(dev) == NULL ||
     strcmp(ofw_bus_get_type(dev), "soc") != 0))
    return (ENXIO);

device_set_desc(dev, "Flattened device tree simple bus");
return (BUS_PROBE_GENERIC);
```

那个代码片段是整个probe风格的紧凑典范。测试 `status`，拒绝已知例外，确认节点看起来像简单总线，描述设备，返回probe置信度。树中每个FDT感知的probe都遵循这种形状的某种变体。

Simplebus通过两个父总线注册自己。在主要的ofw根节点上，它通过 `EARLY_DRIVER_MODULE(simplebus, ofwbus, ...)` 注册；并递归地在自身上通过 `EARLY_DRIVER_MODULE(simplebus, simplebus, ...)` 注册。递归是嵌套simple-bus节点如何被枚举的方式：一个simplebus父节点遇到 `compatible` 为 `"simple-bus"` 的子节点时，将另一个simplebus实例附加到它，然后那个实例枚举*它的*子节点。

对于大多数驱动程序工作，你不需要知道关于simplebus的更多事情，除了它存在以及你向它注册。你的驱动程序的模块注册将是 `DRIVER_MODULE(mydrv, simplebus, mydrv_driver, 0, 0);`，下游的一切将自动发生。

### 将组件映射到Probe调用

为了把移动部件组合在一起，让我们跟踪从DTB被加载到你的驱动程序的probe被调用之间发生的事情。

1. 加载程序将DTB交给内核。
2. 内核的早期arm64代码解析DTB以找到内存和CPU信息。
3. `ofwbus0` 伪设备附加到树根。
4. `ofwbus0` 为 `/soc` 节点（或任何 `compatible = "simple-bus"` 的节点）创建一个 `device_t`，并派发常规newbus探测循环。
5. simplebus驱动程序的probe运行，返回 `BUS_PROBE_GENERIC`，并被选中。
6. Simplebus的attach遍历 `/soc` 节点的子节点，为每个子节点创建一个 `device_t`。每个子节点的资源列表从其 `reg` 和 `interrupts` 属性填充，通过父节点的 `ranges` 转换。
7. 对于每个子节点，newbus探测循环运行。每个注册到simplebus的驱动程序都有机会探测。
8. 你的驱动程序的probe被调用。它调用 `ofw_bus_status_okay()`、`ofw_bus_search_compatible()`，如果匹配则返回 `BUS_PROBE_DEFAULT`。
9. 如果你的驱动程序在此节点的探测竞争中获胜，其attach被调用。此时 `device_t` 已经有一个用节点的内存和中断信息填充的资源列表。
10. 你的驱动程序为其内存区域和中断调用 `bus_alloc_resource_any()`，设置中断处理程序（如果有），映射内存，初始化硬件，并返回0表示成功。

从驱动程序作者的角度来看，前六步是机械的；第7到10步是你编写的代码。本章现在将放大那四个步骤。

### 向ofw_bus注册驱动程序

当你的驱动程序注册到simplebus时，它隐式选择加入OFW派发。模块注册行是：

```c
DRIVER_MODULE(mydrv, simplebus, mydrv_driver, 0, 0);
```

这告诉newbus：*将 `mydrv_driver` 描述的驱动程序作为simplebus的子设备附加*。你的 `device_method_t` 数组必须至少提供 `device_probe` 和 `device_attach` 方法。如果你有detach，添加 `device_detach`。如果你的驱动程序还实现OFW接口方法（在叶级别很少需要），也添加它们。

在某些平台上，对于想要同时附加到simplebus和 `ofwbus` 根节点的驱动程序（以防它们和根节点之间没有simplebus），通常添加第二个注册：

```c
DRIVER_MODULE(mydrv, ofwbus, mydrv_driver, 0, 0);
```

这就是 `gpioled_fdt.c` 做的事情，例如。它覆盖了 `gpio-leds` 节点直接位于根节点下而不是 `simple-bus` 下的平台。

### 编写兼容性表

支持多个芯片变体的驱动程序通常声明一个兼容字符串表：

```c
static const struct ofw_compat_data compat_data[] = {
    { "brcm,bcm2711-gpio",   1 },
    { "brcm,bcm2835-gpio",   2 },
    { NULL,                  0 }
};
```

然后在probe中：

```c
static int
mydrv_probe(device_t dev)
{
    if (!ofw_bus_status_okay(dev))
        return (ENXIO);

    if (ofw_bus_search_compatible(dev, compat_data)->ocd_str == NULL)
        return (ENXIO);

    device_set_desc(dev, "My FDT-Aware Driver");
    return (BUS_PROBE_DEFAULT);
}
```

在attach中，匹配的条目可以重新查找：

```c
static int
mydrv_attach(device_t dev)
{
    const struct ofw_compat_data *match;
    ...
    match = ofw_bus_search_compatible(dev, compat_data);
    if (match == NULL || match->ocd_str == NULL)
        return (ENXIO);

    sc->variant = match->ocd_data; /* BCM2711 为 1，BCM2835 为 2 */
    ...
}
```

`ocd_data` 字段是你在表中定义的cookie。它是一个普通的 `uintptr_t`，所以你可以用它作为整数判别符、指向每个变体结构的指针，或任何适合你驱动程序需求的东西。

### 读取属性

一旦你有了 `device_t`，读取其节点的属性就很简单了。典型模式：

```c
phandle_t node = ofw_bus_get_node(dev);
uint32_t val;

if (OF_getencprop(node, "clock-frequency", &val, sizeof(val)) <= 0) {
    device_printf(dev, "missing clock-frequency\n");
    return (ENXIO);
}
```

辅助函数是 `OF_getencprop` 用于整数属性（处理字节序）、`OF_getprop` 用于原始缓冲区、`OF_getprop_alloc` 用于未知长度的字符串。对于布尔属性（其存在就是信号，值为空），惯用法是 `OF_hasprop`：

```c
bool want_rts = OF_hasprop(node, "uart-has-rtscts");
```

对于列表属性，`OF_getprop_alloc` 或固定缓冲区变体让你拉取完整列表并遍历它。

### 获取资源

内存和中断资源通过你已经知道的标准 `bus_alloc_resource_any()` 调用获取：

```c
sc->mem_rid = 0;
sc->mem_res = bus_alloc_resource_any(dev, SYS_RES_MEMORY,
    &sc->mem_rid, RF_ACTIVE);
if (sc->mem_res == NULL) {
    device_printf(dev, "cannot allocate memory\n");
    return (ENXIO);
}

sc->irq_rid = 0;
sc->irq_res = bus_alloc_resource_any(dev, SYS_RES_IRQ,
    &sc->irq_rid, RF_ACTIVE | RF_SHAREABLE);
```

这是可能的，因为simplebus已经读取了你的节点的 `reg` 和 `interrupts` 属性，通过父节点的 `ranges` 转换了它们，并存储在资源列表中。索引 `0`、`1`、`2` 指的是相应列表的第一、第二和第三个条目。有多个 `reg` 区域的设备将在rid `0`、`1`、`2` 等处有多个内存资源。

对于超出普通内存和中断的任何东西，你进入外设框架。

### 外设框架：时钟、稳压器、复位、引脚控制、GPIO

嵌入式外设通常需要的不仅仅是内存区域。它们需要时钟打开、稳压器启用、复位线解除断言，可能还有引脚复用，有时还需要GPIO来驱动芯片选择或使能线。FreeBSD为每个提供了一个一致的框架集合。

**时钟框架。** 在 `/usr/src/sys/dev/extres/clk/clk.h` 中声明。消费者调用 `clk_get_by_ofw_index(dev, node, idx, &clk)` 获取节点 `clocks` 属性中列出的第N个时钟的句柄，或 `clk_get_by_ofw_name(dev, node, "fck", &clk)` 获取 `clock-names` 条目为 `"fck"` 的时钟。有了句柄后，消费者调用 `clk_enable(clk)` 打开它，`clk_get_freq(clk, &freq)` 查询其频率，`clk_disable(clk)` 在关闭时关闭它。

**稳压器框架。** 在 `/usr/src/sys/dev/extres/regulator/regulator.h` 中声明。`regulator_get_by_ofw_property(dev, node, "vdd-supply", &reg)` 通过命名属性获取稳压器；`regulator_enable(reg)` 启用它；`regulator_disable(reg)` 禁用它。

**硬件复位框架。** 在 `/usr/src/sys/dev/extres/hwreset/hwreset.h` 中声明。`hwreset_get_by_ofw_name(dev, node, "main", &rst)` 获取复位线；`hwreset_deassert(rst)` 将外设从复位中带出；`hwreset_assert(rst)` 将其放回。

**引脚控制框架。** 在 `/usr/src/sys/dev/fdt/fdt_pinctrl.h` 中声明。`fdt_pinctrl_configure_by_name(dev, "default")` 应用与节点的 `pinctrl-names = "default"` 槽位关联的引脚配置。大多数需要引脚控制的驱动程序只需从attach调用一次。

**GPIO框架。** 消费者侧在 `/usr/src/sys/dev/gpio/gpiobusvar.h` 中声明。`gpio_pin_get_by_ofw_idx(dev, node, idx, &pin)` 获取节点 `gpios` 属性中列出的第N个GPIO。`gpio_pin_setflags(pin, GPIO_PIN_OUTPUT)` 设置其方向。`gpio_pin_set_active(pin, value)` 驱动其电平。`gpio_pin_release(pin)` 将其归还。

这些框架是FreeBSD中嵌入式驱动程序通常比Linux对应物更短的原因。你不必编写时钟树、稳压器逻辑或GPIO控制器：你通过统一的消费者API消费它们，而提供者驱动程序是别人的问题。第7节的工作示例端到端使用了GPIO消费者API。

### 中断路由：快速了解interrupt-parent

FDT平台上的中断使用链式查找方案。节点的 `interrupts` 属性给出原始中断说明符，节点的 `interrupt-parent` 属性（或最近祖先的）命名应该解释它的控制器。该控制器反过来可能是另一个控制器的子节点（次级GIC重分发器、嵌套PLIC、某些SoC上的类I/O APIC桥），它进一步将中断向上路由，直到到达绑定到真实CPU向量的顶级控制器。

对于驱动程序作者，你通常不必考虑这个链。内核的中断资源已经作为中断控制器知道如何解释的cookie存在于你的资源列表中，`bus_setup_intr()` 在你请求IRQ时将cookie交回给控制器。重要的是你的节点有正确的 `interrupts = <...>;` 用于其直接中断父控制器，以及树的 `interrupt-parent` 或 `interrupts-extended` 链到达一个真正的控制器。当它没有时，你的中断将在启动时被静默丢弃。

内部机制位于 `/usr/src/sys/dev/ofw/ofw_bus_subr.c`，包括 `ofw_bus_lookup_imap()`、`ofw_bus_setup_iinfo()` 和相关辅助函数。除非你编写总线驱动程序，否则你可能永远不会直接调用它们。

### 简要了解Overlay

我们已经多次提到overlay。简短版本是overlay是一个小型DTB片段，通过标签（例如 `&i2c0` 或 `&gpio0`）引用基础树中的节点，并添加或修改属性或子节点。加载程序在内核看到之前将overlay合并到基础blob中。我们将在第5节回到overlay，在第7节的工作示例中使用它们；现在只需注意FreeBSD通过 `fdt_overlays` 加载程序tunable和 `/boot/dtb/overlays/` 下的文件支持它们。

### arm64上的ACPI与FDT

FreeBSD的arm64端口同时支持FDT和ACPI作为发现机制。内核采取哪条路径在早期启动时通过查看固件提供的内容来决定。如果提供了DTB且顶层 `compatible` 不暗示ACPI路径，内核附加FDT总线。如果提供了ACPI RSDP且固件指示SBSA兼容性，内核附加ACPI总线。相关代码在 `/usr/src/sys/arm64/arm64/nexus.c` 中，它处理两条路径；变量 `arm64_bus_method` 记录选择了哪一个。

对驱动程序作者的实际后果是为两种机制编写的驱动程序必须附加到两条总线。只关心FDT的驱动程序（大多数小型嵌入式驱动程序）只注册simplebus。服务于可能出现在ARM服务器（ACPI）或ARM嵌入式板（FDT）上的通用硬件的驱动程序两者都注册。`/usr/src/sys/dev/ahci/ahci_generic.c` 中的 `ahci_generic` 驱动程序就是这样一个双支持驱动程序；当你最终需要编写这样的驱动程序时，其源代码值得一读。本章的大部分内容我们将留在纯FDT侧。

### 总结本节

第3节给了你FreeBSD FDT支持的地图。你现在知道核心代码在哪里，哪些辅助函数用于哪个工作，以及各部分如何连接：`fdt(4)` 解析树，`OF_*` 原语读取属性，`ofw_bus_*` 辅助函数将原语包装成惯用检查，simplebus枚举子设备，外设框架交给你时钟、复位、稳压器、引脚和GPIO。

在下一节中，我们将使用这些组件编写一个真正的驱动程序。第4节从上到下遍历FDT感知驱动程序的完整骨架，详细到你可以将结构复制到自己的项目中并开始填充硬件特定的逻辑。

## 第4节：为基于FDT的系统编写驱动程序

本节遍历FDT感知FreeBSD驱动程序的完整形状。形状很简单；完整列出的原因是一旦你见过它，树中的每个FDT驱动程序都变得可读。你会开始在 `/usr/src/sys/dev/` 和 `/usr/src/sys/arm/` 下的数百个文件中注意到相同的模式，每个驱动程序都成为你可以改编的又一个模板。

我们将分六轮构建骨架。首先是头文件包含。然后是softc。然后是兼容性表。然后是 `probe()`。然后是 `attach()`。然后是 `detach()` 和模块注册。每轮都很短；到最后你将有一个完整的、可编译的最小驱动程序，当内核将它匹配到设备树节点时打印消息。

### 头文件包含

FDT驱动程序依赖于 `ofw` 和 `fdt` 目录中的少量头文件，以及通常的内核和总线头文件。典型集合：

```c
#include <sys/param.h>
#include <sys/systm.h>
#include <sys/bus.h>
#include <sys/kernel.h>
#include <sys/module.h>
#include <sys/lock.h>
#include <sys/mutex.h>
#include <sys/rman.h>
#include <sys/resource.h>
#include <sys/malloc.h>

#include <machine/bus.h>
#include <machine/resource.h>

#include <dev/ofw/openfirm.h>
#include <dev/ofw/ofw_bus.h>
#include <dev/ofw/ofw_bus_subr.h>
```

没有什么异国情调的。三个 `ofw` 头文件带来了属性读取器和兼容性辅助函数。如果你的驱动程序是GPIO、时钟、稳压器或hwreset的消费者，也添加它们的头文件：

```c
#include <dev/gpio/gpiobusvar.h>
#include <dev/extres/clk/clk.h>
#include <dev/extres/regulator/regulator.h>
#include <dev/extres/hwreset/hwreset.h>
```

引脚控制：

```c
#include <dev/fdt/fdt_pinctrl.h>
```

如果你的驱动程序实际上扩展simplebus而不是仅仅作为叶节点绑定到它：

```c
#include <dev/fdt/simplebus.h>
```

大多数叶驱动程序不需要simplebus头文件。只有当你实现一个枚举子设备的类总线驱动程序时才引入它。

### Softc

FDT感知驱动程序的softc是一个普通的softc，带有一些额外的字段来跟踪你通过OFW辅助函数获取的资源和引用：

```c
struct mydrv_softc {
    device_t        dev;
    struct resource *mem_res;   /* 内存区域 (bus_alloc_resource) */
    int             mem_rid;
    struct resource *irq_res;   /* 中断资源（如果有） */
    int             irq_rid;
    void            *irq_cookie;

    /* FDT特定状态。 */
    phandle_t       node;
    uintptr_t       variant;    /* 匹配的 ocd_data */

    /* 示例：获取的用于驱动芯片选择的GPIO引脚。 */
    gpio_pin_t      cs_pin;

    /* 示例：获取的时钟句柄。 */
    clk_t           clk;

    /* 通常的驱动程序状态：互斥锁、缓冲区等。 */
    struct mtx      sc_mtx;
};
```

与PCI或ISA驱动程序不同的字段是 `node`、`variant` 和消费者句柄（`cs_pin`、`clk` 等）。其他一切都是标准的。

### 兼容性表

兼容性表是驱动程序对一组设备树节点的声明。按照约定它声明为文件作用域且不可变：

```c
static const struct ofw_compat_data mydrv_compat_data[] = {
    { "acme,trivial-timer",    1 },
    { "acme,fancy-timer",      2 },
    { NULL,                    0 }
};
```

第二个字段 `ocd_data` 是一个 `uintptr_t` cookie。我喜欢用它作为整数判别符（基本变体为1，花哨变体为2）；你也可以用它作为指向每个变体配置结构的指针。表以第一个字段为 `NULL` 的哨兵条目结束。

### Probe例程

FDT感知驱动程序的规范probe：

```c
static int
mydrv_probe(device_t dev)
{

    if (!ofw_bus_status_okay(dev))
        return (ENXIO);

    if (ofw_bus_search_compatible(dev, mydrv_compat_data)->ocd_str == NULL)
        return (ENXIO);

    device_set_desc(dev, "ACME Trivial Timer");
    return (BUS_PROBE_DEFAULT);
}
```

三行逻辑。首先，如果节点被禁用则退出。其次，如果我们的兼容字符串都不匹配则退出。第三，设置描述性名称并返回 `BUS_PROBE_DEFAULT`。当多个驱动程序可能声明同一节点时，确切的返回值很重要；更专门的驱动程序可以返回 `BUS_PROBE_SPECIFIC` 以超越通用驱动程序，通用回退可以返回 `BUS_PROBE_GENERIC` 让任何更好的驱动程序获胜。对于大多数驱动程序，`BUS_PROBE_DEFAULT` 是正确的。

`ofw_bus_search_compatible(dev, compat_data)` 调用返回指向匹配条目的指针，如果没有匹配则返回指向哨兵条目的指针。哨兵的 `ocd_str` 是 `NULL`，所以测试 `NULL` 是表达*我们没有匹配任何东西*的惯用方式。一些驱动程序将返回的指针保存到局部变量并重用它；我们将在attach中这样做。

### Attach例程

Attach是真正工作发生的地方。规范的FDT attach：

```c
static int
mydrv_attach(device_t dev)
{
    struct mydrv_softc *sc;
    const struct ofw_compat_data *match;
    phandle_t node;
    uint32_t freq;
    int err;

    sc = device_get_softc(dev);
    sc->dev = dev;
    sc->node = ofw_bus_get_node(dev);
    node = sc->node;

    /* 记住我们匹配了哪个变体。 */
    match = ofw_bus_search_compatible(dev, mydrv_compat_data);
    if (match == NULL || match->ocd_str == NULL)
        return (ENXIO);
    sc->variant = match->ocd_data;

    /* 拉取任何必需的属性。 */
    if (OF_getencprop(node, "clock-frequency", &freq, sizeof(freq)) <= 0) {
        device_printf(dev, "missing clock-frequency property\n");
        return (ENXIO);
    }

    /* 分配内存和中断资源。 */
    sc->mem_rid = 0;
    sc->mem_res = bus_alloc_resource_any(dev, SYS_RES_MEMORY,
        &sc->mem_rid, RF_ACTIVE);
    if (sc->mem_res == NULL) {
        device_printf(dev, "cannot allocate memory resource\n");
        err = ENXIO;
        goto fail;
    }

    sc->irq_rid = 0;
    sc->irq_res = bus_alloc_resource_any(dev, SYS_RES_IRQ,
        &sc->irq_rid, RF_ACTIVE | RF_SHAREABLE);
    if (sc->irq_res == NULL) {
        device_printf(dev, "cannot allocate IRQ resource\n");
        err = ENXIO;
        goto fail;
    }

    /* 启用时钟（如果描述了一个）。 */
    if (clk_get_by_ofw_index(dev, node, 0, &sc->clk) == 0) {
        err = clk_enable(sc->clk);
        if (err != 0) {
            device_printf(dev, "could not enable clock: %d\n", err);
            goto fail;
        }
    }

    /* 应用pinctrl默认值（如果有）。 */
    (void)fdt_pinctrl_configure_by_name(dev, "default");

    /* 初始化锁和驱动程序状态。 */
    mtx_init(&sc->sc_mtx, device_get_nameunit(dev), NULL, MTX_DEF);

    /* 连接中断处理程序。 */
    err = bus_setup_intr(dev, sc->irq_res,
        INTR_TYPE_MISC | INTR_MPSAFE, NULL, mydrv_intr, sc,
        &sc->irq_cookie);
    if (err != 0) {
        device_printf(dev, "could not setup interrupt: %d\n", err);
        goto fail;
    }

    device_printf(dev, "variant %lu at %s, clock %u Hz\n",
        (unsigned long)sc->variant, device_get_nameunit(dev), freq);

    return (0);

fail:
    mydrv_detach(dev);
    return (err);
}
```

这有很多内容需要解析。让我们逐步分析。

1. `device_get_softc(dev)` 返回驱动程序的softc，FreeBSD在驱动程序附加时为你分配了它。
2. `ofw_bus_get_node(dev)` 返回我们DT节点的phandle。我们把它保存在softc中，因为detach也需要它。
3. 我们重新运行兼容性搜索并记录匹配了哪个变体。
4. 我们用 `OF_getencprop` 读取标量整数属性。调用返回读取的字节数，不存在时返回 `-1`，如果属性太短则返回某个较小的数字。我们将任何非正值视为失败。
5. 我们分配内存和IRQ资源。Simplebus已经从节点的 `reg` 和 `interrupts` 填充了资源列表，所以索引0和0是正确的。
6. 我们尝试获取时钟。这个驱动程序将时钟视为可选的，所以缺少 `clocks` 属性不是致命的。如果存在，我们启用它。
7. 我们应用默认引脚控制。
8. 我们初始化驱动程序互斥锁。
9. 我们设置中断处理程序，它将派发到 `mydrv_intr`。
10. 我们记录一条消息。
11. 在任何错误时，我们goto一个调用detach的单一清理路径。

单一清理路径值得单独讨论。这是一个嵌入式友好的模式，因为嵌入式驱动程序从许多不同的框架获取许多资源，试图在每个失败点写出清理代码很快变得不可读。相反，编写一个处理部分初始化softc状态的detach，并从失败路径调用它。这是FreeBSD树一致使用的模式；如果你遵循它，你的驱动程序将更容易阅读。

### Detach例程

合规的detach拆除attach可能设置的所有东西，并以相反的顺序进行：

```c
static int
mydrv_detach(device_t dev)
{
    struct mydrv_softc *sc;

    sc = device_get_softc(dev);

    if (sc->irq_cookie != NULL) {
        bus_teardown_intr(dev, sc->irq_res, sc->irq_cookie);
        sc->irq_cookie = NULL;
    }

    if (sc->irq_res != NULL) {
        bus_release_resource(dev, SYS_RES_IRQ, sc->irq_rid,
            sc->irq_res);
        sc->irq_res = NULL;
    }

    if (sc->mem_res != NULL) {
        bus_release_resource(dev, SYS_RES_MEMORY, sc->mem_rid,
            sc->mem_res);
        sc->mem_res = NULL;
    }

    if (sc->clk != NULL) {
        clk_disable(sc->clk);
        clk_release(sc->clk);
        sc->clk = NULL;
    }

    if (mtx_initialized(&sc->sc_mtx))
        mtx_destroy(&sc->sc_mtx);

    return (0);
}
```

有两点需要注意。首先，每个拆除步骤都有检查资源是否实际获取的保护。检查让detach可以正确地从正常卸载路径（所有资源都已获取）或失败attach路径（只有部分已获取）运行。其次，顺序是获取的相反顺序。中断处理程序先关闭再释放中断资源。时钟先禁用再释放。互斥锁最后销毁。

### 中断处理程序

中断处理程序是普通的FreeBSD中断例程。没有什么FDT特定的东西：

```c
static void
mydrv_intr(void *arg)
{
    struct mydrv_softc *sc = arg;

    mtx_lock(&sc->sc_mtx);
    /* 处理硬件事件。 */
    mtx_unlock(&sc->sc_mtx);
}
```

*FDT特定的*是中断资源在attach中设置的方式。资源通过simplebus从节点的 `interrupts` 属性来，simplebus通过interrupt-parent链转换了它，所以当你的驱动程序调用 `bus_alloc_resource_any(SYS_RES_IRQ, ...)` 时，资源已经代表了真正控制器处的真正硬件中断。

### 模块注册

驱动程序的模块注册将设备方法绑定到驱动程序并将驱动程序注册到父总线：

```c
static device_method_t mydrv_methods[] = {
    DEVMETHOD(device_probe,  mydrv_probe),
    DEVMETHOD(device_attach, mydrv_attach),
    DEVMETHOD(device_detach, mydrv_detach),

    DEVMETHOD_END
};

static driver_t mydrv_driver = {
    "mydrv",
    mydrv_methods,
    sizeof(struct mydrv_softc)
};

DRIVER_MODULE(mydrv, simplebus, mydrv_driver, 0, 0);
DRIVER_MODULE(mydrv, ofwbus,   mydrv_driver, 0, 0);
MODULE_VERSION(mydrv, 1);
SIMPLEBUS_PNP_INFO(mydrv_compat_data);
```

两个 `DRIVER_MODULE` 调用将驱动程序注册到simplebus和ofwbus根节点。后者覆盖节点直接位于根节点下而不是显式simple-bus下的平台或板子。`MODULE_VERSION` 为 `kldstat` 和依赖跟踪声明驱动程序的版本。`SIMPLEBUS_PNP_INFO` 发出 `kldstat -v` 可以打印的pnpinfo描述符；这是给操作员的小小便利，但没有它驱动程序也能工作。

### 组装完整的骨架

这里是将骨架组装成一个单一最小文件的代码，可以作为内核模块编译。它不做任何有用的事情；它只演示附加并在匹配时记录一条消息：

```c
#include <sys/param.h>
#include <sys/systm.h>
#include <sys/bus.h>
#include <sys/kernel.h>
#include <sys/module.h>
#include <sys/rman.h>

#include <machine/bus.h>
#include <machine/resource.h>

#include <dev/ofw/openfirm.h>
#include <dev/ofw/ofw_bus.h>
#include <dev/ofw/ofw_bus_subr.h>

struct fdthello_softc {
    device_t dev;
    phandle_t node;
};

static const struct ofw_compat_data compat_data[] = {
    { "freebsd,fdthello",  1 },
    { NULL,                0 }
};

static int
fdthello_probe(device_t dev)
{
    if (!ofw_bus_status_okay(dev))
        return (ENXIO);

    if (ofw_bus_search_compatible(dev, compat_data)->ocd_str == NULL)
        return (ENXIO);

    device_set_desc(dev, "FDT Hello Example");
    return (BUS_PROBE_DEFAULT);
}

static int
fdthello_attach(device_t dev)
{
    struct fdthello_softc *sc;

    sc = device_get_softc(dev);
    sc->dev = dev;
    sc->node = ofw_bus_get_node(dev);

    device_printf(dev, "attached, node phandle 0x%x\n", sc->node);
    return (0);
}

static int
fdthello_detach(device_t dev)
{
    device_printf(dev, "detached\n");
    return (0);
}

static device_method_t fdthello_methods[] = {
    DEVMETHOD(device_probe,  fdthello_probe),
    DEVMETHOD(device_attach, fdthello_attach),
    DEVMETHOD(device_detach, fdthello_detach),

    DEVMETHOD_END
};

static driver_t fdthello_driver = {
    "fdthello",
    fdthello_methods,
    sizeof(struct fdthello_softc)
};

DRIVER_MODULE(fdthello, simplebus, fdthello_driver, 0, 0);
DRIVER_MODULE(fdthello, ofwbus,   fdthello_driver, 0, 0);
MODULE_VERSION(fdthello, 1);
```

以及其匹配的 `Makefile`：

```make
KMOD=	fdthello
SRCS=	fdthello.c

SYSDIR?= /usr/src/sys

.include <bsd.kmod.mk>
```

你会在 `examples/part-07/ch32-fdt-embedded/lab01-fdthello/` 下找到它们。在任何安装了内核源码的FreeBSD系统上构建它们。模块只会在DTB包含 `compatible = "freebsd,fdthello"` 节点的平台上*附加*，但即使在amd64上它至少可以编译和干净地加载。

### 内核接下来做什么

当你在DTB包含匹配节点的arm64系统上加载该模块时，序列是：

1. 模块向simplebus和ofwbus注册 `fdthello_driver`。
2. Newbus遍历每个以 `simplebus` 或 `ofwbus` 为父设备的现有设备，并调用新注册驱动程序的probe。
3. 对于每个节点 `compatible = "freebsd,fdthello"` 的设备，probe返回 `BUS_PROBE_DEFAULT`。如果没有其他驱动程序已经附加（或者如果我们胜过它），我们的attach被调用。
4. Attach记录一条消息；设备现在已附加。

当你卸载模块时，detach为每个附加的实例运行，然后模块被卸载。在简单情况下，`kldunload fdthello` 完全清理。

### 检查你的工作

三种快速方法来判断你的驱动程序是否匹配：

1. **`dmesg`** 应该显示类似以下的行：
   ```
   fdthello0: <FDT Hello Example> on simplebus0
   fdthello0: attached, node phandle 0x8f
   ```
2. **`devinfo -r`** 应该显示你的设备附加在 `simplebus` 下的某处。
3. **`sysctl dev.fdthello.0.%parent`** 应该确认父总线。

如果你的模块加载但没有设备附加，probe没有匹配。最常见的原因是兼容字符串中的拼写错误、缺少或禁用的节点，或节点位于simplebus/ofwbus驱动程序未到达的地方。第6节详细讨论调试。

### 关于命名和供应商前缀的说明

真正的FreeBSD驱动程序匹配兼容字符串如 `"brcm,bcm2711-gpio"`、`"allwinner,sun50i-a64-mmc"` 或 `"st,stm32-uart"`。逗号前的前缀是供应商或社区名称；其余是特定芯片或系列。该约定在上游Linux和FreeBSD中都受到广泛尊重。当为实验发明新的兼容字符串时（如我们上面用 `"freebsd,fdthello"` 所做的那样），遵循相同的供应商斜杠标识符形式。不要发明单词兼容字符串；它们与现有的冲突，并让未来的读者困惑。

### 总结本节

第4节遍历了FDT驱动程序的形状。你已经看到了要包含的头文件、要定义的softc、要声明的兼容性表、要编写的probe和attach以及detach例程，以及将它们绑定在一起的模块注册。你有一个最小、完整的驱动程序，你可以现在编译和加载。它做的不多，但其结构与FreeBSD中每个FDT感知驱动程序使用的结构相同。

在下一节中，我们转向故事的另一半。没有设备树节点匹配的驱动程序是无用的。第5节教你如何构建和修改 `.dtb` 文件，FreeBSD的overlay系统如何工作，以及如何向现有板描述添加自己的节点，以便你的驱动程序有东西可以附加。

## 第5节：创建和修改设备树Blob

你现在有一个耐心等待 `compatible = "freebsd,fdthello"` 设备树节点的驱动程序。运行系统中没有这样的节点，所以什么也不探测。在本节中，我们将学习如何改变这种状况。我们将了解源码到二进制的流水线、overlay机制（让我们无需重建整个 `.dtb` 就能添加节点），以及决定内核在启动时实际看到哪个blob的加载程序tunable。

创建设备树blob不是经验丰富的内核黑客的入门仪式。它是一个普通的编辑任务。文件是文本，编译器是标准的，输出是位于 `/boot` 中的一个小型二进制文件。让它感觉陌生的只是很少有爱好者项目会遇到它。在嵌入式FreeBSD上，它是日常工作。

### 源码到二进制流水线

每个启动FreeBSD系统的 `.dtb` 都从一个或多个源文件开始。流水线很简单：

```text
.dtsi  .dtsi  .dtsi
   \    |    /
    \   |   /
     .dts (顶层源码)
       |
       | cpp (C预处理器)
       v
     .dts (预处理后)
       |
       | dtc (设备树编译器)
       v
     .dtb (二进制blob)
```

C预处理器首先运行。它展开 `#include` 指令、来自 `dt-bindings/gpio/gpio.h` 等头文件的宏定义，以及属性表达式中的算术。然后 `dtc` 编译器将预处理后的源码转换成内核可以解析的紧凑扁平格式。

Overlay文件经过相同的流水线，只是其源文件带有扩展名 `.dtso`，输出带有 `.dtbo`。唯一真正的语法差异是overlay源文件顶部的魔法咒语，我们稍后会看到。

FreeBSD的构建系统用两个小型shell脚本包装了这个流水线，你可以在 `/usr/src/sys/tools/fdt/make_dtb.sh` 和 `/usr/src/sys/tools/fdt/make_dtbo.sh` 中学习。它们将 `cpp` 和 `dtc` 链接在一起，为内核自己的 `dt-bindings` 头文件添加正确的包含路径，并将生成的blob写入构建树。当你为嵌入式平台 `make buildkernel` 时，这些脚本就是产生最终安装在系统 `/boot/dtb/` 中 `.dtb` 文件的东西。

### 安装工具

在FreeBSD上，`dtc` 可以通过port获得：

```console
# pkg install dtc
```

该包安装 `dtc` 二进制文件及其配套工具 `fdtdump`，后者打印现有blob的解码结构。如果你计划做任何overlay工作，两个都安装。FreeBSD基础树也在 `/usr/src/sys/contrib/device-tree/` 下附带 `dtc` 的副本，但port版本从用户空间更容易使用。

检查版本：

```console
$ dtc --version
Version: DTC 1.7.0
```

1.6及以上版本支持overlay。更早版本缺少 `/plugin/;` 指令，所以如果你继承了一个旧的构建环境，在继续之前先升级。

### 编写独立的.dts文件

我们将从一个完整的独立设备树源文件开始，让语法在添加overlay复杂性之前有时间沉淀。在内核树外创建一个名为 `tiny.dts` 的文件：

```dts
/dts-v1/;

/ {
    compatible = "example,tiny-board";
    model = "Tiny Example Board";
    #address-cells = <1>;
    #size-cells = <1>;

    chosen {
        bootargs = "-v";
    };

    cpus {
        #address-cells = <1>;
        #size-cells = <0>;

        cpu0: cpu@0 {
            device_type = "cpu";
            reg = <0>;
            compatible = "arm,cortex-a53";
        };
    };

    memory@0 {
        device_type = "memory";
        reg = <0x00000000 0x10000000>;
    };

    soc {
        compatible = "simple-bus";
        #address-cells = <1>;
        #size-cells = <1>;
        ranges;

        hello0: hello@10000 {
            compatible = "freebsd,fdthello";
            reg = <0x10000 0x100>;
            status = "okay";
        };
    };
};
```

第一行 `/dts-v1/;` 告诉 `dtc` 我们使用的是哪个版本的源码格式。版本1是目前唯一使用的版本，但该指令仍然是必需的。

之后，我们有根节点，包含一些预期的子节点。`cpus` 节点描述处理器拓扑，`memory@0` 节点声明物理地址零处256 MB的DRAM区域，`soc` 节点将片上外设分组在 `simple-bus` 下。在 `soc` 内部，我们的 `hello@10000` 节点为我们在第4节编写的 `fdthello` 驱动程序提供设备树匹配。

即使在这个小文件中，也有一些值得注意的东西。

首先，`#address-cells` 和 `#size-cells` 在 `soc` 节点内再次出现。父节点设置的值只适用于该父节点的直接子节点，所以树的每个关心地址的层级都必须声明它们。这里 `soc` 对地址和大小各使用一个cell，这就是为什么 `hello@10000` 内的 `reg = <0x10000 0x100>;` 正好列出两个 `u32` 值。

其次，`soc` 节点上的 `ranges;` 属性是空的。空的 `ranges` 意味着"此总线内的地址与外部地址一一对应"。如果 `soc` 被映射到与其子节点声明的不同的基地址，你会使用更长的 `ranges` 列表来表达转换。

第三，`status = "okay"` 在这里是显式的。没有它，每棵树隐式默认为okay，但许多板文件将可选外设设为 `status = "disabled"`，并期望overlay或板特定文件来翻转它们。养成每当驱动程序神秘地未能探测时检查此属性的习惯。

### 编译.dts文件

编译微型示例：

```console
$ dtc -I dts -O dtb -o tiny.dtb tiny.dts
```

`-I dts` 标志告诉 `dtc` 输入是文本源码，`-O dtb` 请求二进制blob输出。成功的编译不打印任何内容。语法错误会告诉你文件和行号。

你可以用 `fdtdump` 验证结果：

```console
$ fdtdump tiny.dtb | head -30
**** fdtdump is a low-level debugging tool, not meant for general use. ****
    Use the fdtput/fdtget/dtc tools to manipulate .dtb files.

/dts-v1/;
// magic:               0xd00dfeed
// totalsize:           0x214 (532)
...
```

那个往返确认blob有效且可解析。你现在可以用 `-dtb tiny.dtb` 将它放入QEMU运行，内核会尝试对它启动。在实践中你很少为真实板子手写完整的 `.dts`。你从供应商自己的源文件开始（例如 Raspberry Pi 4用 `/usr/src/sys/contrib/device-tree/src/arm64/broadcom/bcm2711-rpi-4-b.dts`），然后用overlay修改一部分节点。

### .dtsi包含文件的角色

`.dtsi` 扩展名用于设备树*包含*。这些文件包含旨在被拉入另一个 `.dts` 或 `.dtsi` 的树片段。编译器将它们视为与 `.dts` 文件相同，但文件名后缀告诉其他人（以及构建系统）该文件不是独立的。

现代SoC描述中的常见模式是：

```text
arm/bcm283x.dtsi          <- SoC定义
arm/bcm2710.dtsi          <- 系列细化 (Pi 3 血统)
arm/bcm2710-rpi-3-b.dts   <- 特定板子顶层文件，包含两者
```

每个 `.dtsi` 添加和细化节点。在较低文件中声明的标签可以从较高文件用 `&label` 语法引用来覆盖属性，而不需要重写节点。这是使少量共享SoC描述支持数十个相关板子成为可能的机制。

### 了解Overlay

真实SBC如Raspberry Pi 4的完整 `.dts` 有几十KB长。如果你只想启用SPI，或添加一个GPIO控制的外设，重建整个blob是浪费且容易出错的。overlay机制正是为这种情况存在的。

overlay是一个小型的特殊 `.dtb`，针对现有树。在加载时，FreeBSD加载程序将overlay合并到内存中的基础树，产生内核视为单个设备树的组合视图。磁盘上的基础 `.dtb` 从不被修改。这意味着同一个overlay可以在多个系统上各放一份来启用一个功能。

overlay源文件的语法在顶部使用两个魔法指令：

```dts
/dts-v1/;
/plugin/;
```

之后，源码通过标签引用基础树中的节点。编译器符号化地记录引用，加载程序在合并时根据基础树实际导出的标签来解析它们。这就是为什么overlay可以独立于它后来将要合并的确切基础树来编写和编译。

这里是一个将 `fdthello` 节点附加到现有 `soc` 总线的最小overlay：

```dts
/dts-v1/;
/plugin/;

/ {
    compatible = "brcm,bcm2711";

    fragment@0 {
        target = <&soc>;
        __overlay__ {
            hello0: hello@20000 {
                compatible = "freebsd,fdthello";
                reg = <0x20000 0x100>;
                status = "okay";
            };
        };
    };
};
```

外层 `compatible` 说这个overlay用于BCM2711树。加载程序使用该字符串拒绝与当前板子不匹配的overlay。在里面我们看到一个 `fragment@0` 节点。每个fragment通过 `target` 属性针对基础树中的一个现有节点。`__overlay__` 下的内容是合并到目标中的属性和子节点集合。

在这个例子中，我们在合并时 `&soc` 解析为的任何节点下添加一个 `hello@20000` 子节点。Raspberry Pi 4上的基础树在顶级SoC总线节点上声明了 `soc` 标签，所以合并后基础 `soc` 将获得一个新的 `hello@20000` 子节点，我们的驱动程序的probe将触发。

你也可以使用overlay来*修改*现有节点。如果你设置一个与现有属性同名的属性，overlay值替换原始值。如果你添加新属性，它直接出现。如果你添加新子节点，它被嫁接上去。机制是可加的，除了属性值替换。

### 编译和部署Overlay

构建overlay：

```console
$ dtc -I dts -O dtb -@ -o fdthello.dtbo fdthello-overlay.dts
```

`-@` 标志告诉编译器发出合并时所需的符号标签信息。没有它，引用标签的overlay会静默失败或产生无用的错误。

在运行FreeBSD的系统上，overlay位于 `/boot/dtb/overlays/`。文件名按约定需要以 `.dtbo` 结尾。加载程序默认在 `/boot/dtb/overlays` 查找；如果你想在其他地方暂存overlay，路径可以通过加载程序tunable覆盖。

要告诉加载程序应用哪些overlay，在 `/boot/loader.conf` 中添加一行：

```ini
fdt_overlays="fdthello,sunxi-i2c1,spigen-rpi4"
```

值是不带 `.dtbo` 扩展名的overlay基本名称的逗号分隔列表。顺序只在overlay相互交互时重要。启动时，加载程序读取列表，加载每个overlay，按顺序将它们合并到基础树中，并将组合的blob交给内核。

一个好的健全性检查是在串行控制台或HDMI屏幕上观察加载程序输出。当设置了 `fdt_overlays` 时，加载程序打印类似以下的行：

```text
Loading DTB overlay 'fdthello' (0x1200 bytes)
```

如果文件缺失或目标不匹配，加载程序打印警告并继续。你的驱动程序然后无法探测，因为overlay从未应用。检查加载程序的控制台输出是捕获这类失败的最快方式。

### 演练：向Raspberry Pi 4树添加节点

让我们在一个真实的场景中把机制组合起来。想象你正在为Raspberry Pi 4开发一个自定义子板。它包含一个连接在Pi的GPIO头GPIO18上的单个GPIO控制指示LED。你想让FreeBSD通过你自己的 `edled` 驱动程序（我们在第7节构建）驱动LED。你需要一个设备树节点。

首先，查看Pi 4的基础 `.dtb` 已经声明了什么。在安装了FreeBSD的运行Pi上，`ofwdump -ap | less` 或 `fdtdump /boot/dtb/broadcom/bcm2711-rpi-4-b.dtb | less` 给你完整的树。你主要关注的是 `soc` 和 `gpio` 节点，在那里你看到一个标签 `gpio = <&gpio>;` 从GPIO控制器导出。

接下来，编写overlay源码：

```dts
/dts-v1/;
/plugin/;

/ {
    compatible = "brcm,bcm2711";

    fragment@0 {
        target-path = "/soc";
        __overlay__ {
            edled0: edled@0 {
                compatible = "example,edled";
                status = "okay";
                leds-gpios = <&gpio 18 0>;
                label = "daughter-indicator";
            };
        };
    };
};
```

我们这里使用 `target-path` 而不是 `target`，因为目标是一个现有路径而不是标签。两种形式都有效；`target` 接受phandle引用，`target-path` 接受字符串。

`leds-gpios` 属性是设备树中描述GPIO引用的惯用方式。它是GPIO控制器的phandle，后跟该控制器上的GPIO编号，再后跟标志字。标志值为 `0` 表示高电平有效；`1` 表示低电平有效。Pi上的引脚复用通常不需要显式提及，因为Broadcom GPIO控制器通过同一寄存器集处理方向和功能。

编译并安装overlay：

```console
$ dtc -I dts -O dtb -@ -o edled.dtbo edled-overlay.dts
$ sudo cp edled.dtbo /boot/dtb/overlays/
```

将overlay添加到加载程序配置：

```console
# echo 'fdt_overlays="edled"' >> /boot/loader.conf
```

重启。在启动过程中，加载程序打印overlay加载行，基础DT在 `/soc` 下获得新的 `edled@0` 节点，`edled` 驱动程序的probe触发，子板LED进入软件控制。

### 检查结果

内核运行后，三个工具验证一切都落在了正确的位置：

```console
# ofwdump -p /soc/edled@0
```

打印新添加节点的属性。

```console
# sysctl dev.edled.0.%parent
```

确认驱动程序已附加并显示其父总线。

```console
# devinfo -r | less
```

显示FreeBSD看到的完整设备树，你的驱动程序在其中。

如果这些中有任何与overlay内容不一致，第6节帮助你诊断原因。

### 故障排除构建失败

大多数DT构建错误属于少数几个类别。

**未解析的引用。** 如果overlay引用了基础树未导出的标签如 `&gpio`，加载程序打印 `no symbol for <label>` 并拒绝应用overlay。通过使用绝对路径的 `target-path` 代替来修复，或者用 `-@` 重建基础 `.dtb` 以包含其符号。

**语法错误。** 这些显示为带有行号的 `dtc` 错误。常见的罪魁祸首是属性赋值末尾缺少分号、不平衡的花括号，以及混合单元类型的属性值（例如同一行混合尖括号整数和引号字符串）。

**Cell计数不匹配。** 如果父节点声明 `#address-cells = <2>` 而子节点的 `reg` 只给一个cell，编译器容忍它但内核会错误解析值。`ofwdump -p node` 和仔细阅读父节点的cell计数通常会揭示不匹配。

**重复节点名称。** 同一级别的两个节点不能共享相同的名称加单元地址。编译器会标记这个，但尝试添加与现有节点名称冲突的新节点的overlay在启动时会产生神秘的合并失败。选择唯一的名称或针对不同的路径。

### 内核的dtb加载过程

作为背景，了解最终合并的blob在加载程序交接后发生什么是有帮助的。

在arm64和其他几个平台上，加载程序将blob放在固定的物理地址，并在其启动参数块中将指针传递给内核。最早的内核代码在 `/usr/src/sys/arm64/arm64/machdep.c` 中，验证魔数和大小，将blob映射到内核虚拟内存，并用FDT子系统注册它。到Newbus开始附加设备时，blob已经完全解析，OFW API可以遍历它。

在amd64嵌入式系统上（罕见但存在），流程类似：UEFI通过配置表传递blob，加载程序发现它，内核通过相同的FDT API消费它。

blob从内核的角度是只读的。你永远不在运行时修改它。如果属性值需要更改，正确的位置是在源码中更改，而不是在活动树中。

### 总结本节

第5节教你如何在设备树源码和设备树二进制之间移动，overlay如何针对现有树，以及 `fdt_overlays` 如何将整个事情连接到FreeBSD启动过程中。你现在可以编写一个 `.dts`，编译它，将结果放在 `/boot/dtb/overlays/`，在 `loader.conf` 中列出它，并观察内核拾取你的节点。你在第4节编写的驱动程序现在有东西可以附加了。

在第6节中，我们将镜头转向检查实际在内核中降落了什么。当事情出错时（它们会出错的），良好的观察是回到工作系统的最短路径。

## 第6节：测试和调试FDT驱动程序

你编写的每个驱动程序在某个时刻都会未能探测。设备树在源码中看起来正确，你的兼容字符串读起来正确，`kldload` 无怨言地完成，但 `dmesg` 是沉默的。调试这类失败是一项独立的技能，你越早养成这个习惯，每个问题花费的时间就越少。

本节涵盖检查运行中的设备树、诊断探测失败、详细观察附加行为以及跟踪卸载问题的工具和技术。这里的很多材料也适用于总线驱动程序、外设驱动程序和伪设备。真正FDT特定的是读取树本身的工具集。

### ofwdump(8)工具

在运行的FreeBSD系统上，`ofwdump` 是你查看设备树的主要窗口。它从内核内树中打印节点和属性，所以它显示的正是驱动程序在探测时看到的东西。如果树在内核中是错的，它在 `ofwdump` 中也会是错的，这使你不必为了检查一个编辑而编译和重新启动。

最简单的调用打印整棵树：

```console
# ofwdump -a
```

在任何非平凡系统上通过管道传给 `less`；输出有数千行。

更集中的运行转储一个节点及其属性：

```console
# ofwdump -p /soc/hello@10000
Node 0x123456: /soc/hello@10000
    compatible:  freebsd,fdthello
    reg:         00 01 00 00 00 00 01 00
    status:      okay
```

`-p` 标志在节点名称旁边打印属性。整数值以字节串形式输出，因为 `ofwdump` 通常无法知道一个属性应该有多少个cell。你使用父节点的 `#address-cells` 和 `#size-cells` 来解释字节。

读取一个特定属性：

```console
# ofwdump -P compatible /soc/hello@10000
```

添加 `-R` 递归到给定节点的子节点。添加 `-S` 打印phandle，添加 `-r` 打印原始二进制（如果你想将数据通过管道传给另一个工具）。

熟悉 `ofwdump`。当有人说"检查树"时，这就是他们指的工具。

### 通过sysctl读取原始Blob

FreeBSD通过sysctl暴露未合并的基础blob：

```console
# sysctl -b hw.fdt.dtb | fdtdump
```

`-b` 标志告诉sysctl打印原始二进制；通过管道传给 `fdtdump` 解码它。这在你怀疑overlay已修改树并想比较合并前blob与合并后视图时很有用。`ofwdump` 显示合并后视图；`hw.fdt.dtb` 显示合并前基础。

### 在arm64上确认FDT模式

FreeBSD不暴露一个专门的sysctl来说"你在FDT上运行"或"你在ACPI上运行"。决定在启动非常早期由内核变量 `arm64_bus_method` 做出，从用户空间观察它的最简单方式是看 `dmesg` 在启动时打印的设备树。选择FDT路径的机器显示一个根行如：

```text
ofwbus0: <Open Firmware Device Tree>
simplebus0: <Flattened device tree simple bus> on ofwbus0
```

然后是其余的FDT子节点。选择ACPI路径的机器显示 `acpi0: <...>` 代替，你永远不会看到 `ofwbus0` 行。

在运行的系统上，你还可以运行 `devinfo -r` 并在层次结构中寻找 `ofwbus0`，或确认sysctl `hw.fdt.dtb` 存在。该sysctl只在启动时解析了DTB时注册，所以它的存在本身就是一个信号：

```console
# sysctl -N hw.fdt.dtb 2>/dev/null && echo "FDT is active" || echo "ACPI or neither"
```

`-N` 标志只要求sysctl提供名称，所以它在不打印blob字节的情况下成功。

在支持两种机制的板子上，选择它们的机制是加载程序tunable `kern.cfg.order`。在 `/boot/loader.conf` 中设置 `kern.cfg.order="fdt"` 强制内核先尝试FDT，只有在没有找到DTB时才回退到ACPI；`kern.cfg.order="acpi"` 做相反的事。在x86平台上，`hint.acpi.0.disabled="1"` 完全禁用ACPI附加，在固件行为不正常时有时有用。第3节更详细地涵盖了这种双重性；如果你曾经在一个FDT驱动程序在ARM服务器平台上拒绝附加时茫然失措，首先要验证的事情之一是内核实际选择了哪种总线方法。

### 调试不触发的Probe

最常见的症状是沉默：模块加载了，`kldstat` 显示它，但没有设备附加。probe要么从未运行，要么返回了 `ENXIO`。按以下检查清单走一遍。

**1. 内核树中是否存在该节点？**

```console
# ofwdump -p /soc/your-node
```

如果节点缺失，你的overlay没有应用。查看启动时加载程序的输出。重新检查 `/boot/loader.conf` 中的 `fdt_overlays=` 行。确认 `.dtbo` 文件在 `/boot/dtb/overlays/` 中。如果你怀疑是过期副本，重建overlay。

**2. status属性是否设置为okay？**

```console
# ofwdump -P status /soc/your-node
```

值为 `"disabled"` 会阻止节点被探测。基础板文件经常将可选外设声明为禁用，并留给overlay来启用它们。

**3. 兼容字符串是否与驱动程序期望的完全一致？**

overlay或驱动程序兼容表中的拼写错误是探测失败的最常见单一原因。逐字符比较它们：

```console
# ofwdump -P compatible /soc/your-node
```

对比驱动程序中的匹配行：

```c
{"freebsd,fdthello", 1},
```

即使供应商前缀不同（例如 `free-bsd,` 对比 `freebsd,`），也不会发生匹配。

**4. 父总线是否支持探测？**

FDT驱动程序附加到 `simplebus` 或 `ofwbus`。如果你的节点的父节点是其他东西（比如 `i2c` 总线节点），你的驱动程序必须改为向那个父节点注册。通过在 `ofwdump` 中向上一级查看来检查父节点。

**5. 驱动程序是否排在了已经匹配的另一个驱动程序之下？**

如果一个更通用的驱动程序先返回了 `BUS_PROBE_GENERIC`，你的新驱动程序需要返回更强的东西，如 `BUS_PROBE_DEFAULT` 或 `BUS_PROBE_SPECIFIC`。`devinfo -r` 显示实际附加的驱动程序。

### 添加临时调试输出

当以上都不揭示原因时，在 `probe` 和 `attach` 中添加 `device_printf` 调用来直接观察流程。在 `probe` 中：

```c
static int
fdthello_probe(device_t dev)
{
    device_printf(dev, "probe: node=%ld compat=%s\n",
        ofw_bus_get_node(dev),
        ofw_bus_get_compat(dev) ? ofw_bus_get_compat(dev) : "(none)");

    if (!ofw_bus_status_okay(dev)) {
        device_printf(dev, "probe: status not okay\n");
        return (ENXIO);
    }

    if (ofw_bus_search_compatible(dev, compat_data)->ocd_data == 0) {
        device_printf(dev, "probe: compat mismatch\n");
        return (ENXIO);
    }

    device_set_desc(dev, "FDT Hello Example");
    return (BUS_PROBE_DEFAULT);
}
```

这在每次probe调用时打印，所以预期会有噪音。在发布前删除打印。重点是暂时性地了解 `ofw_bus_*` 辅助函数返回了什么。

在 `attach` 中，打印你分配的资源rid和地址：

```c
device_printf(dev, "attach: mem=%#jx size=%#jx\n",
    (uintmax_t)rman_get_start(sc->mem),
    (uintmax_t)rman_get_size(sc->mem));
```

这确认 `bus_alloc_resource_any` 交回了有效范围。匹配成功但attach在这里崩溃通常意味着DT中的 `reg` 是错误的。

### 观察附加顺序和依赖关系

在嵌入式系统上，附加顺序不总是直观的。消费GPIO的驱动程序必须等待GPIO控制器先附加。如果你的驱动程序在控制器准备好之前尝试获取GPIO线，`gpio_pin_get_by_ofw_idx` 返回 `ENXIO`，你的attach失败。FreeBSD通过在驱动程序注册时表达的显式依赖关系，以及中断树的中断父节点遍历来处理排序。

使用 `devinfo -rv` 观察顺序：

```console
# devinfo -rv | grep -E '(gpio|edled|simplebus)'
```

如果 `edled` 出现在 `gpio` 之前，排序需要修复。通常的修复是消费者驱动程序中的 `MODULE_DEPEND` 行：

```c
MODULE_DEPEND(edled, gpiobus, 1, 1, 1);
```

这强制 `gpiobus` 先加载，确保GPIO控制器在 `edled` 附加时可用。

### 调试Detach和卸载

Detach调试比probe调试容易，因为detach在系统运行时执行，`printf` 输出立即到达 `dmesg`。你最可能遇到的两个问题是：

**卸载返回EBUSY。** 某个资源仍被驱动程序持有。常见原因是GPIO引脚或中断句柄未被释放。审计每个 `_get_` 调用并确保detach中有匹配的 `_release_`。

**卸载成功但模块在下次 `kldload` 时重新附加。** 这几乎总是因为detach留下了一个softc字段指向已释放的内存，第二次attach跟随了那个指针。将detach视为对attach构建的一切的仔细反向拆除。

一个有用的技巧是添加：

```c
device_printf(dev, "detach: entered\n");
...拆除...
device_printf(dev, "detach: complete\n");
```

如果第二行从未出现，detach中的某个东西挂起或panic了。

### 使用DTrace获得更深层可见性

对于更复杂的调查，DTrace可以在不触碰驱动程序源码的情况下跟踪整个内核的 `device_probe` 和 `device_attach`。一个显示每个attach调用的一行命令：

```console
# dtrace -n 'fbt::device_attach:entry { printf("%s", stringof(args[0]->softc)); }'
```

输出在启动期间很嘈杂，但在你 `kldload` 驱动程序时交互运行会自然过滤掉。DTrace的使用超出了本章的范围，但知道它存在值得花半页来设置。

### 在QEMU上测试

并非每个读者都有Raspberry Pi或BeagleBone来测试。QEMU可以模拟arm64 virt机器，在上面启动FreeBSD，让你加载驱动程序和overlay，无需任何真实硬件。virt机器使用自己的设备树，由QEMU自动生成；你的overlay可以用与真实板子完全相同的方式针对那棵树。唯一的注意事项是virt机器上的GPIO和类似的低级外设有限或不存在。对于纯DT和模块实验，它完全够用。

基本调用看起来像这样：

```console
$ qemu-system-aarch64 \
    -M virt \
    -cpu cortex-a72 \
    -m 2G \
    -kernel /path/to/kernel \
    -drive if=virtio,file=disk.img \
    -serial mon:stdio \
    -append "console=comconsole"
```

系统启动后，`kldload` 你的模块并在串行控制台上观察probe消息。

### 何时停止调试并重建

有时候，通过拆除驱动程序并从已知良好的骨架重建来修复bug更容易。第4节中的 `fdthello` 示例正是那个骨架。如果你发现自己追逐一个探测失败超过一个小时，复制 `fdthello`，重命名它，添加你的兼容字符串，验证平凡的情况附加了。然后一次一个地将真正的功能移植过来。你几乎总会在过程中找到bug。

### 总结本节

第6节为你配备了嵌入式驱动程序调试器的工具和习惯。你有 `ofwdump` 用于树，`hw.fdt.dtb` 用于原始blob，`devinfo -r` 用于附加设备视图，`MODULE_DEPEND` 用于排序，`device_printf` 用于临时可见性。你还有常见探测和detach失败的心理检查清单。

第7节现在将本章的所有理论组合成一个单一的工作示例：一个由GPIO支持的LED驱动程序，你构建、编译、加载并从 `.dts` overlay驱动它。如果你按顺序学完了本章，这个例子会感觉是我们已经看到的组件的直接综合。

## 第7节：实际示例：嵌入式板的GPIO驱动程序

本节完整演示一个小型但真实的名为 `edled`（嵌入式LED）的驱动程序的构建。该驱动程序：

1. 匹配 `compatible = "example,edled"` 的设备树节点。
2. 获取节点的 `leds-gpios` 属性中列出的GPIO引脚。
3. 暴露一个sysctl旋钮，用户可以切换以设置LED状态。
4. 在detach上干净地释放GPIO。

该驱动程序故意很小。一旦它工作了，你将能够改编它来驱动任何位于单个GPIO后面的东西，当你需要处理多个引脚、中断或更复杂的外设时，这些模式可以扩展。

### 你需要什么

跟随操作你需要：

- 运行内核14.3或更高版本的FreeBSD系统。
- 安装在 `/usr/src` 下的内核源码。
- `devel/dtc` port或类似来源的 `dtc`。
- 一块至少有一个空闲GPIO引脚和LED的板子（或者你可以在没有真实LED的情况下测试sysctl切换；驱动程序仍然会附加并记录状态变化）。

如果你在运行FreeBSD的Raspberry Pi 4上，GPIO 18是一个方便的选择，因为它不会与默认控制台或SD卡控制器冲突。Pi 3或Pi Zero 2以调整的GPIO编号相同方式工作。在BeagleBone Black上，从46路头上的众多空闲引脚中任选一个。

### 整体文件布局

我们将产生五个文件：

```text
edled.c            <- C源码
Makefile           <- 内核模块Makefile
edled.dts          <- DT overlay源码
edled.dtbo         <- 编译后的overlay (输出)
README             <- 读者笔记
```

`examples/` 树下的相应仓库布局是：

```text
examples/part-07/ch32-fdt-embedded/lab04-edled/
    edled.c
    edled.dts
    Makefile
    README.md
```

你可以在本章末尾的实验部分到达时从那棵树中复制文件。

### Softc

每个驱动程序实例需要一个小的状态块。`edled` 的softc持有：

- 设备句柄本身。
- GPIO引脚描述符。
- 当前开/关状态。
- 用于切换的sysctl oid。

```c
struct edled_softc {
    device_t        sc_dev;
    gpio_pin_t      sc_pin;
    int             sc_on;
    struct sysctl_oid *sc_oid;
};
```

`gpio_pin_t` 在 `/usr/src/sys/dev/gpio/gpiobusvar.h` 中定义。它是一个不透明句柄，携带GPIO控制器引用、引脚号和高电平有效/低电平有效标志。你永远不直接解引用它；你将它传递给 `gpio_pin_setflags`、`gpio_pin_set_active` 和 `gpio_pin_release`。

### 头文件

`edled.c` 顶部引入我们需要的定义：

```c
#include <sys/param.h>
#include <sys/systm.h>
#include <sys/kernel.h>
#include <sys/module.h>
#include <sys/malloc.h>
#include <sys/bus.h>
#include <sys/sysctl.h>

#include <machine/bus.h>
#include <machine/resource.h>

#include <dev/ofw/ofw_bus.h>
#include <dev/ofw/ofw_bus_subr.h>
#include <dev/ofw/openfirm.h>

#include <dev/gpio/gpiobusvar.h>
```

与第4节中的 `fdthello` 骨架相比，我们为旋钮添加了 `<sys/sysctl.h>`，为GPIO消费者API添加了 `<dev/gpio/gpiobusvar.h>`。

### 兼容性表

这个驱动程序只需要一个条目的小表：

```c
static const struct ofw_compat_data compat_data[] = {
    {"example,edled", 1},
    {NULL,            0}
};
```

真正的项目会选择它拥有的供应商前缀。使用 `"example,"` 标记兼容字符串为示例性的。当你发布产品时，用你的公司或项目前缀替换它。

### Probe

Probe使用与 `fdthello` 相同的模式：

```c
static int
edled_probe(device_t dev)
{
    if (!ofw_bus_status_okay(dev))
        return (ENXIO);

    if (ofw_bus_search_compatible(dev, compat_data)->ocd_data == 0)
        return (ENXIO);

    device_set_desc(dev, "Example embedded LED");
    return (BUS_PROBE_DEFAULT);
}
```

这里没有新东西。从第4节逐字复制probe的唯一原因是强调这个步骤在驱动程序之间有多重复；驱动程序之间有意义的差异几乎总是存在于attach、detach和操作层中。

### Attach

Attach是真正工作发生的地方。我们分配和初始化softc，获取GPIO引脚，将其配置为输出，设置为"关"，发布sysctl，并打印确认。

```c
static int
edled_attach(device_t dev)
{
    struct edled_softc *sc = device_get_softc(dev);
    phandle_t node = ofw_bus_get_node(dev);
    int error;

    sc->sc_dev = dev;
    sc->sc_on = 0;

    error = gpio_pin_get_by_ofw_property(dev, node,
        "leds-gpios", &sc->sc_pin);
    if (error != 0) {
        device_printf(dev, "cannot get GPIO pin: %d\n", error);
        return (error);
    }

    error = gpio_pin_setflags(sc->sc_pin, GPIO_PIN_OUTPUT);
    if (error != 0) {
        device_printf(dev, "cannot set pin flags: %d\n", error);
        gpio_pin_release(sc->sc_pin);
        return (error);
    }

    error = gpio_pin_set_active(sc->sc_pin, 0);
    if (error != 0) {
        device_printf(dev, "cannot set pin state: %d\n", error);
        gpio_pin_release(sc->sc_pin);
        return (error);
    }

    sc->sc_oid = SYSCTL_ADD_PROC(device_get_sysctl_ctx(dev),
        SYSCTL_CHILDREN(device_get_sysctl_tree(dev)),
        OID_AUTO, "state",
        CTLTYPE_INT | CTLFLAG_RW | CTLFLAG_NEEDGIANT,
        sc, 0, edled_sysctl_state, "I", "LED state (0=off, 1=on)");

    device_printf(dev, "attached, GPIO pin acquired, state=0\n");
    return (0);
}
```

这段代码中有几件值得检查的事情。

调用 `gpio_pin_get_by_ofw_property(dev, node, "leds-gpios", &sc->sc_pin)` 解析DT节点的 `leds-gpios` 属性，将phandle解析到GPIO控制器，消费引脚号，并产生一个即用型句柄。如果控制器尚未附加，此调用返回 `ENXIO`，这就是为什么我们在注册时表达对 `gpiobus` 的 `MODULE_DEPEND`。

`gpio_pin_setflags(sc->sc_pin, GPIO_PIN_OUTPUT)` 配置引脚方向。其他有效标志包括 `GPIO_PIN_INPUT`、`GPIO_PIN_PULLUP` 和 `GPIO_PIN_PULLDOWN`。你可以组合它们，例如 `GPIO_PIN_INPUT | GPIO_PIN_PULLUP`。

`gpio_pin_set_active(sc->sc_pin, 0)` 将引脚设置为其非活动电平。这里的"活动"考虑了极性，所以对于配置为低电平有效的引脚，值 `1` 驱动线路为低，`0` 驱动为高。我们之前讨论的DT标志cell就是决定这一点的。

`SYSCTL_ADD_PROC` 在 `dev.edled.<unit>.state` 创建一个节点，其处理程序是我们自己的 `edled_sysctl_state` 函数。`CTLFLAG_NEEDGIANT` 标志对于还没有适当锁定的小型驱动程序是合适的；生产驱动程序会使用专用的互斥锁并去掉Giant标志。

如果任何步骤失败，我们释放已经获取的东西并返回错误。在错误路径上泄露GPIO引脚会阻止其他驱动程序永远使用同一条线。

### Sysctl处理程序

sysctl处理程序读取或写入LED状态：

```c
static int
edled_sysctl_state(SYSCTL_HANDLER_ARGS)
{
    struct edled_softc *sc = arg1;
    int val = sc->sc_on;
    int error;

    error = sysctl_handle_int(oidp, &val, 0, req);
    if (error != 0 || req->newptr == NULL)
        return (error);

    if (val != 0 && val != 1)
        return (EINVAL);

    error = gpio_pin_set_active(sc->sc_pin, val);
    if (error == 0)
        sc->sc_on = val;
    return (error);
}
```

`SYSCTL_HANDLER_ARGS` 展开为标准sysctl处理程序签名。我们将当前值读入局部变量，调用 `sysctl_handle_int` 进行用户空间复制，如果用户提供了新值，我们进行健全性检查并通过GPIO API应用它。当前状态保存在softc中，所以没有写入的读取返回我们设置的上一个值。

### Detach

Detach必须以相反顺序释放attach获取的一切：

```c
static int
edled_detach(device_t dev)
{
    struct edled_softc *sc = device_get_softc(dev);

    if (sc->sc_pin != NULL) {
        (void)gpio_pin_set_active(sc->sc_pin, 0);
        gpio_pin_release(sc->sc_pin);
        sc->sc_pin = NULL;
    }
    device_printf(dev, "detached\n");
    return (0);
}
```

我们在释放引脚之前关闭LED。在模块卸载后让LED亮着对下一个驱动程序是不礼貌的；更糟糕的是引脚在断言时被释放，它驱动的任何东西都保持开启，直到其他东西重新声明那条线。sysctl上下文由newbus系统通过 `device_get_sysctl_ctx` 拥有，所以我们不显式释放oid；newbus为我们拆除它。

### 方法表和驱动程序注册

这里没有什么令人惊讶的：

```c
static device_method_t edled_methods[] = {
    DEVMETHOD(device_probe,  edled_probe),
    DEVMETHOD(device_attach, edled_attach),
    DEVMETHOD(device_detach, edled_detach),
    DEVMETHOD_END
};

static driver_t edled_driver = {
    "edled",
    edled_methods,
    sizeof(struct edled_softc)
};

DRIVER_MODULE(edled, simplebus, edled_driver, 0, 0);
DRIVER_MODULE(edled, ofwbus,    edled_driver, 0, 0);
MODULE_DEPEND(edled, gpiobus, 1, 1, 1);
MODULE_VERSION(edled, 1);
```

与 `fdthello` 相比的唯一添加是 `MODULE_DEPEND(edled, gpiobus, 1, 1, 1)`。三个整数参数是 `edled` 可以容忍的 `gpiobus` 的最低、首选和最高版本。`1, 1, 1` 值三元组意味着"任何1或以上版本"。在实践中这几乎总是你想要的。

### 完整源码

把所有东西放在一起：

```c
/*
 * edled.c - Example Embedded LED Driver
 *
 * Demonstrates a minimal FDT-driven GPIO consumer on FreeBSD 14.3.
 */

#include <sys/param.h>
#include <sys/systm.h>
#include <sys/kernel.h>
#include <sys/module.h>
#include <sys/malloc.h>
#include <sys/bus.h>
#include <sys/sysctl.h>

#include <machine/bus.h>
#include <machine/resource.h>

#include <dev/ofw/ofw_bus.h>
#include <dev/ofw/ofw_bus_subr.h>
#include <dev/ofw/openfirm.h>

#include <dev/gpio/gpiobusvar.h>

struct edled_softc {
    device_t        sc_dev;
    gpio_pin_t      sc_pin;
    int             sc_on;
    struct sysctl_oid *sc_oid;
};

static const struct ofw_compat_data compat_data[] = {
    {"example,edled", 1},
    {NULL,            0}
};

static int edled_sysctl_state(SYSCTL_HANDLER_ARGS);

static int
edled_probe(device_t dev)
{
    if (!ofw_bus_status_okay(dev))
        return (ENXIO);
    if (ofw_bus_search_compatible(dev, compat_data)->ocd_data == 0)
        return (ENXIO);
    device_set_desc(dev, "Example embedded LED");
    return (BUS_PROBE_DEFAULT);
}

static int
edled_attach(device_t dev)
{
    struct edled_softc *sc = device_get_softc(dev);
    phandle_t node = ofw_bus_get_node(dev);
    int error;

    sc->sc_dev = dev;
    sc->sc_on = 0;

    error = gpio_pin_get_by_ofw_property(dev, node,
        "leds-gpios", &sc->sc_pin);
    if (error != 0) {
        device_printf(dev, "cannot get GPIO pin: %d\n", error);
        return (error);
    }

    error = gpio_pin_setflags(sc->sc_pin, GPIO_PIN_OUTPUT);
    if (error != 0) {
        device_printf(dev, "cannot set pin flags: %d\n", error);
        gpio_pin_release(sc->sc_pin);
        return (error);
    }

    error = gpio_pin_set_active(sc->sc_pin, 0);
    if (error != 0) {
        device_printf(dev, "cannot set pin state: %d\n", error);
        gpio_pin_release(sc->sc_pin);
        return (error);
    }

    sc->sc_oid = SYSCTL_ADD_PROC(device_get_sysctl_ctx(dev),
        SYSCTL_CHILDREN(device_get_sysctl_tree(dev)),
        OID_AUTO, "state",
        CTLTYPE_INT | CTLFLAG_RW | CTLFLAG_NEEDGIANT,
        sc, 0, edled_sysctl_state, "I", "LED state (0=off, 1=on)");

    device_printf(dev, "attached, GPIO pin acquired, state=0\n");
    return (0);
}

static int
edled_detach(device_t dev)
{
    struct edled_softc *sc = device_get_softc(dev);

    if (sc->sc_pin != NULL) {
        (void)gpio_pin_set_active(sc->sc_pin, 0);
        gpio_pin_release(sc->sc_pin);
        sc->sc_pin = NULL;
    }
    device_printf(dev, "detached\n");
    return (0);
}

static int
edled_sysctl_state(SYSCTL_HANDLER_ARGS)
{
    struct edled_softc *sc = arg1;
    int val = sc->sc_on;
    int error;

    error = sysctl_handle_int(oidp, &val, 0, req);
    if (error != 0 || req->newptr == NULL)
        return (error);

    if (val != 0 && val != 1)
        return (EINVAL);

    error = gpio_pin_set_active(sc->sc_pin, val);
    if (error == 0)
        sc->sc_on = val;
    return (error);
}

static device_method_t edled_methods[] = {
    DEVMETHOD(device_probe,  edled_probe),
    DEVMETHOD(device_attach, edled_attach),
    DEVMETHOD(device_detach, edled_detach),
    DEVMETHOD_END
};

static driver_t edled_driver = {
    "edled",
    edled_methods,
    sizeof(struct edled_softc)
};

DRIVER_MODULE(edled, simplebus, edled_driver, 0, 0);
DRIVER_MODULE(edled, ofwbus,    edled_driver, 0, 0);
MODULE_DEPEND(edled, gpiobus, 1, 1, 1);
MODULE_VERSION(edled, 1);
```

大约140行C代码，包括头文件和空行。这是一个工作的、生产形态的FDT GPIO驱动程序。

### Makefile

与本书中每个内核模块一样，Makefile很简单：

```makefile
KMOD=   edled
SRCS=   edled.c

SYSDIR?= /usr/src/sys

.include <bsd.kmod.mk>
```

`bsd.kmod.mk` 处理其余的事情。在目录中输入 `make` 会产生 `edled.ko` 和 `edled.ko.debug`。

### Overlay源码

配套的 `.dts` overlay看起来像这样（针对Raspberry Pi 4调整；根据你的板子调整）：

```dts
/dts-v1/;
/plugin/;

/ {
    compatible = "brcm,bcm2711";

    fragment@0 {
        target-path = "/soc";
        __overlay__ {
            edled0: edled@0 {
                compatible = "example,edled";
                status = "okay";
                leds-gpios = <&gpio 18 0>;
                label = "lab-indicator";
            };
        };
    };
};
```

编译：

```console
$ dtc -I dts -O dtb -@ -o edled.dtbo edled.dts
```

并复制到 `/boot/dtb/overlays/`。

### 构建和加载

在目标系统上，将所有四个文件放在一个临时目录中，然后：

```console
$ make
$ sudo cp edled.dtbo /boot/dtb/overlays/
$ sudo sh -c 'echo fdt_overlays=\"edled\" >> /boot/loader.conf'
$ sudo reboot
```

重启后，你应该看到：

```console
# dmesg | grep edled
edled0: <Example embedded LED> on simplebus0
edled0: attached, GPIO pin acquired, state=0
```

如果你在GPIO 18上插入了LED，它目前是关闭的。验证：

```console
# sysctl dev.edled.0.state
dev.edled.0.state: 0
```

打开它：

```console
# sysctl dev.edled.0.state=1
dev.edled.0.state: 0 -> 1
```

LED亮了。关闭它：

```console
# sysctl dev.edled.0.state=0
dev.edled.0.state: 1 -> 0
```

完成。你有一个端到端工作的嵌入式驱动程序：从设备树源码，到内核模块，到用户空间控制。

### 检查结果设备

几个有用的查询确认驱动程序已良好集成：

```console
# devinfo -r | grep -A1 simplebus
# sysctl dev.edled.0
# ofwdump -p /soc/edled@0
```

第一个显示你的驱动程序在Newbus树中的位置。第二个列出驱动程序注册的所有sysctl。第三个确认DT节点有预期的属性。

### 值得一说的陷阱

以下是编写第一个GPIO消费者驱动程序时的经典错误。每个都很容易犯，但一旦你知道要找它们，也很容易避免。

**忘记在detach中释放引脚。** `kldunload` 成功，但引脚保持被占用。下次加载报告"引脚忙碌"。始终将每个 `gpio_pin_get_*` 与detach中的 `gpio_pin_release` 匹配。

**在父总线附加之前读取DT属性。** `leds-gpios` 解析返回 `ENXIO`。修复是确保GPIO控制器模块先加载，通过 `MODULE_DEPEND`。启动期间这会自动发生，因为静态内核两者都已驻留；在用 `kldload` 手动加载的实验中，你可能需要先显式加载 `gpiobus`。

**活动标志弄错。** 在LED的接线方式是GPIO灌电流（LED连接在 `3V3` 和引脚之间）的板子上，"开"对应低输出。在这种情况下，`leds-gpios = <&gpio 18 1>` 是正确的，`gpio_pin_set_active(sc->sc_pin, 1)` 将驱动引脚为低，点亮LED。如果LED行为相反，翻转标志。

**没有锁的sysctl改变状态。** 这个驱动程序使用 `CTLFLAG_NEEDGIANT` 作为快捷方式。在真正的驱动程序中，你分配一个 `struct mtx`，在sysctl处理程序中的GPIO调用周围获取它，并在不带Giant标志的情况下发布sysctl。对于单GPIO LED，实践中关系不大，但一旦你扩展驱动程序处理中断或共享状态，这个模式就很重要了。

### 总结本节

第7节兑现了本章的承诺。你构建了一个完整的FDT驱动GPIO消费者，通过overlay部署它，在运行的系统上加载它，并从用户空间操作它。你使用的组件——兼容表、OFW辅助函数、通过消费者框架的资源获取、sysctl注册、newbus注册——是FreeBSD中每个嵌入式驱动程序依赖的相同组件。

第8节着眼于如何将像 `edled` 这样的工作驱动程序变成一个健壮的驱动程序。工作是第一个里程碑。健壮是驱动程序赢得其在内核树中位置的那个里程碑。

## 第8节：重构和完成你的嵌入式驱动程序

第7节的驱动程序工作。你加载模块，翻转sysctl，LED表现正常。这是一个真正的成就，如果目标是在工作台上做一次实验，你可以到此为止。对于更严肃的事情，工作驱动程序需要变成*完成的*驱动程序：一个可以被陌生人阅读、被审查者审计、在系统中连续运行数月而被信任的驱动程序。

本节遍历将 `edled` 从工作变为完成的重构过程。这些改变不是为了让它做更多。它们是为了让它正确。同样的过程适用于你编写的任何驱动程序，包括你从其他项目改编或从Linux移植的驱动程序。

### 重构在这里意味着什么

"重构"是那些经常涵盖说话者想改变的任何东西的词之一。对于本节的目的，重构意味着：

1. 移除在快乐路径上恰好不会触发的潜在bug。
2. 添加生产驱动程序需要的锁定和错误路径。
3. 改进名称、布局和注释，让下一个读者不需要猜测。
4. 当attach主体变得太长时，将基础设施移出attach到辅助函数中。

这里没有任何东西改变驱动程序的外部行为。sysctl仍然读写相同的整数，LED仍然开关，DT绑定不会移动。改变的是当意外发生时驱动程序的可靠性。

### 第一轮：收紧Attach错误路径

原始的attach函数长出了一簇错误处理程序，每个都调用 `gpio_pin_release` 并返回。那可以工作，但它重复了清理工作。更干净的形状使用带有标签的单一退出块：

```c
static int
edled_attach(device_t dev)
{
    struct edled_softc *sc = device_get_softc(dev);
    phandle_t node = ofw_bus_get_node(dev);
    int error;

    sc->sc_dev = dev;
    sc->sc_on = 0;

    error = gpio_pin_get_by_ofw_property(dev, node,
        "leds-gpios", &sc->sc_pin);
    if (error != 0) {
        device_printf(dev, "cannot get GPIO pin: %d\n", error);
        goto fail;
    }

    error = gpio_pin_setflags(sc->sc_pin, GPIO_PIN_OUTPUT);
    if (error != 0) {
        device_printf(dev, "cannot set pin flags: %d\n", error);
        goto fail;
    }

    error = gpio_pin_set_active(sc->sc_pin, 0);
    if (error != 0) {
        device_printf(dev, "cannot set pin state: %d\n", error);
        goto fail;
    }

    sc->sc_oid = SYSCTL_ADD_PROC(device_get_sysctl_ctx(dev),
        SYSCTL_CHILDREN(device_get_sysctl_tree(dev)),
        OID_AUTO, "state",
        CTLTYPE_INT | CTLFLAG_RW,
        sc, 0, edled_sysctl_state, "I", "LED state (0=off, 1=on)");

    device_printf(dev, "attached, state=0\n");
    return (0);

fail:
    if (sc->sc_pin != NULL) {
        gpio_pin_release(sc->sc_pin);
        sc->sc_pin = NULL;
    }
    return (error);
}
```

`goto fail` 模式是惯用的FreeBSD内核风格。它将清理逻辑折叠到一个地方，使得未来的编辑不可能因为忘记几个相同的 `release` 调用中的一个而泄露资源。

### 第二轮：添加适当的锁定

`CTLFLAG_NEEDGIANT` 是一个快捷方式。正确的方法是围绕硬件访问持有per-softc互斥锁：

```c
struct edled_softc {
    device_t        sc_dev;
    gpio_pin_t      sc_pin;
    int             sc_on;
    struct sysctl_oid *sc_oid;
    struct mtx      sc_mtx;
};
```

在attach中初始化互斥锁：

```c
mtx_init(&sc->sc_mtx, device_get_nameunit(dev), "edled", MTX_DEF);
```

在detach中销毁它：

```c
mtx_destroy(&sc->sc_mtx);
```

在sysctl处理程序中的硬件调用周围获取它：

```c
static int
edled_sysctl_state(SYSCTL_HANDLER_ARGS)
{
    struct edled_softc *sc = arg1;
    int val, error;

    mtx_lock(&sc->sc_mtx);
    val = sc->sc_on;
    mtx_unlock(&sc->sc_mtx);

    error = sysctl_handle_int(oidp, &val, 0, req);
    if (error != 0 || req->newptr == NULL)
        return (error);

    if (val != 0 && val != 1)
        return (EINVAL);

    mtx_lock(&sc->sc_mtx);
    error = gpio_pin_set_active(sc->sc_pin, val);
    if (error == 0)
        sc->sc_on = val;
    mtx_unlock(&sc->sc_mtx);

    return (error);
}
```

注意我们在 `sysctl_handle_int` 周围放下了互斥锁。该调用可能向或从用户空间复制数据，这可能会睡眠，你不应在睡眠时持有互斥锁。我们交给 `sysctl_handle_int` 的值是本地副本，所以放下锁是安全的。

从 `SYSCTL_ADD_PROC` 调用中移除 `CTLFLAG_NEEDGIANT`。有了真正的锁，Giant不再需要。

### 第三轮：显式处理电源轨

在许多真实外设上，驱动程序负责在接触设备之前打开电源轨和参考时钟。FreeBSD在 `/usr/src/sys/dev/extres/regulator/` 和 `/usr/src/sys/dev/extres/clk/` 下提供了消费者API。即使分立LED不需要稳压器，更严肃的外设（比如SPI连接的加速度计）需要。为了保持 `edled` 作为有用的教学模板，我们展示机制如何嵌入。

在一个假设的DT节点下：

```dts
edled0: edled@0 {
    compatible = "example,edled";
    status = "okay";
    leds-gpios = <&gpio 18 0>;
    vled-supply = <&ldo_led>;
    clocks = <&clks 42>;
    label = "lab-indicator";
};
```

两个额外属性：`vled-supply` 引用稳压器phandle，`clocks` 引用时钟phandle。Attach像这样拾取它们：

```c
#include <dev/extres/clk/clk.h>
#include <dev/extres/regulator/regulator.h>

struct edled_softc {
    ...
    regulator_t     sc_reg;
    clk_t           sc_clk;
};

...

    error = regulator_get_by_ofw_property(dev, node, "vled-supply",
        &sc->sc_reg);
    if (error == 0) {
        error = regulator_enable(sc->sc_reg);
        if (error != 0) {
            device_printf(dev, "cannot enable regulator: %d\n",
                error);
            goto fail;
        }
    } else if (error != ENOENT) {
        device_printf(dev, "regulator lookup failed: %d\n", error);
        goto fail;
    }

    error = clk_get_by_ofw_index(dev, node, 0, &sc->sc_clk);
    if (error == 0) {
        error = clk_enable(sc->sc_clk);
        if (error != 0) {
            device_printf(dev, "cannot enable clock: %d\n", error);
            goto fail;
        }
    } else if (error != ENOENT) {
        device_printf(dev, "clock lookup failed: %d\n", error);
        goto fail;
    }
```

Detach以相反顺序释放它们：

```c
    if (sc->sc_clk != NULL) {
        clk_disable(sc->sc_clk);
        clk_release(sc->sc_clk);
    }
    if (sc->sc_reg != NULL) {
        regulator_disable(sc->sc_reg);
        regulator_release(sc->sc_reg);
    }
```

`ENOENT` 检查很重要。如果DT没有声明稳压器或时钟，`regulator_get_by_ofw_property` 和 `clk_get_by_ofw_index` 返回 `ENOENT`。支持多个板子的驱动程序——有些有专用电源轨，有些没有——将 `ENOENT` 视为"这里不需要"而不是致命错误。

### 第四轮：引脚复用设置

在GPIO引脚可以被重新用途化为UART、SPI、I2C或其他功能的SoC上，引脚复用控制器必须在驱动程序使用引脚之前被编程。FreeBSD通过 `/usr/src/sys/dev/fdt/fdt_pinctrl.h` 的 `pinctrl` 框架处理这个。请求特定配置的设备树节点使用 `pinctrl-names` 和 `pinctrl-N` 属性：

```dts
edled0: edled@0 {
    compatible = "example,edled";
    pinctrl-names = "default";
    pinctrl-0 = <&edled_pins>;
    ...
};

&pinctrl {
    edled_pins: edled_pins {
        brcm,pins = <18>;
        brcm,function = <1>;  /* GPIO output */
    };
};
```

在attach中，调用：

```c
fdt_pinctrl_configure_by_name(dev, "default");
```

在任何引脚访问之前。框架遍历 `pinctrl-0` 句柄，找到引用的节点，并通过SoC特定的pinctrl驱动程序应用其设置。

LED示例并不严格需要pinmux，因为Broadcom GPIO驱动程序作为 `gpio_pin_setflags` 的一部分配置引脚，但在OMAP、Allwinner和许多其他SoC上它是必不可少的。在你的教学模板中包含这个模式，以便读者看到它适合在哪里。

### 第五轮：风格和命名审计

慢慢阅读最终源码。要检查的东西：

- **一致的命名。** 所有函数以 `edled_` 开头，所有字段以 `sc_` 开头，所有常量大写。阅读源码的陌生人不应疑惑一个符号属于哪个驱动程序。
- **没有死代码。** 移除任何在引入期间有用但没有生产目的的 `device_printf` 或存根函数。
- **没有魔法数字。** 如果你在十个地方写 `sc->sc_on = 0`，定义一个enum或至少一个 `#define EDLED_OFF 0`。
- **简短注释只在代码意图不明显的地方。** 试图给每个函数添加docstring往往会使FreeBSD源码变得杂乱；简洁是本地的风格。
- **正确的包含顺序。** 按约定，`<sys/param.h>` 在前，然后是其他 `<sys/...>` 头文件，然后是 `<machine/...>`，然后是子系统特定的头文件如 `<dev/ofw/...>`。
- **行长度。** 坚持到80列。长函数调用使用FreeBSD缩进风格进行续行。
- **许可证头。** 每个FreeBSD源文件以项目的BSD风格许可证块开头。对于树外驱动程序，包含你自己的版权和许可证声明。

### 第六轮：静态分析

用提高警告级别运行编译器：

```console
$ make CFLAGS="-Wall -Wextra -Werror"
```

修复每个警告。警告要么表明真正的bug，要么表明不清楚的代码。两种情况下修复都改善驱动程序。

考虑运行scan-build：

```console
$ scan-build make
```

`scan-build` 是llvm clang分析器的一部分。它捕获编译器遗漏的空指针解引用和释放后使用bug。

### 第七轮：文档

驱动程序在没有阅读代码就能被理解之前不算完成。编写一页README覆盖：

- 驱动程序做什么。
- 它期望哪个DT绑定。
- 它有哪些模块依赖。
- 任何已知限制或板子特定说明。
- 如何构建、加载和操作它。

也为配套材料树包含一个简短的手册页。即使是存根 `edled(4)` 页也有价值；你可以稍后完善它。

### 打包和分发

树外驱动程序存在于几个规范位置：

- 作为非官方 `devel/` port，供用户在FreeBSD之上安装。
- 作为遵循FreeBSD项目常规布局的GitHub仓库。
- 作为与README和INSTALL文件一起发布的 `.tar.gz` 归档。

FreeBSD ports树欢迎已知稳定的驱动程序包。一旦驱动程序经过了一些实际使用的考验，提交 `devel/edled-kmod` port是一个合理的目标。

如果你的驱动程序足够通用以造福其他用户，考虑向上游贡献。审查过程仔细但积极，`freebsd-drivers@freebsd.org` 邮件列表是自然的起点。

### 与真实FreeBSD驱动程序对比审查

一旦 `edled` 变得紧凑，将它与 `/usr/src/sys/dev/gpio/gpioled_fdt.c` 比较，这是启发该示例的驱动程序。真实驱动程序稍大，因为它支持每个父节点多个LED，但其整体形状与你的匹配。注意它如何：

- 使用 `for (child = OF_child(leds); child != 0; child = OF_peer(child))` 遍历DT子节点。
- 调用 `OF_getprop_alloc` 读取变长标签字符串。
- 通过 `DRIVER_MODULE` 注册到 `simplebus` 和 `ofwbus`。
- 通过 `SIMPLEBUS_PNP_INFO` 声明其DT绑定，以便 `devmatch(8)` 的设备ID匹配工作。

完成自己的驱动程序后详细阅读真实驱动程序是你在该领域可以做的最有成效的事情之一。你会发现你从未见过的技术，你会认出现在从内部理解的 pattern。

### 总结本节

第8节遍历了每个驱动程序需要的完成过程。错误路径收紧，锁定纠正，电源和时钟处理明确化，引脚复用考虑在内，风格审计，分析运行，文档编写。你现在拥有的不再是一个实验；它是一个你可以正大光明地交给别人的驱动程序。

此时本章的技术材料已经完成。剩余部分是让你自己运行一切的动手实验、扩展所学内容的挑战练习、需要警惕的常见错误简短列表，以及闭合回到本书更广泛弧线的总结。

## 第9节：阅读真实的FreeBSD FDT驱动程序

我们已经构建了 `fdthello` 和 `edled`，两个为了教学目的而存在的驱动程序。它们是真实的，因为你可以将它们加载到FreeBSD系统上并看到它们附加，但它们很小，而且不携带在树中存在多年并被数十个贡献者触摸过的驱动程序所积累的智慧。要完成你作为FDT驱动程序作者的学徒期，你需要阅读不是作为教学材料开始的驱动程序。

本节从 `/usr/src/sys` 中挑选几个驱动程序，并带你浏览它们展示的内容。目标不是让你记住它们的源码，而是建立将阅读真实代码作为主要学习来源的习惯。本书的教学示例会在几个月内从记忆中褪去；真实驱动程序阅读是你可以在余下职业生涯中使用的技能。

### gpioled_fdt.c：edled的近亲

我们的 `edled` 驱动程序是刻意模仿 `/usr/src/sys/dev/gpio/gpioled_fdt.c` 的。心中想着 `edled` 来阅读真实的东西使对比很有启发性。真实驱动程序约150行，几乎和我们的一样大，但处理了我们选择简化的几个细节。

驱动程序的兼容性表列出一个条目：

```c
static struct ofw_compat_data compat_data[] = {
    {"gpio-leds", 1},
    {NULL,        0}
};
```

注意 `gpio-leds` 是一个无前缀字符串。这反映了一个早于当前供应商前缀约定的长期社区绑定。新绑定应始终使用前缀，但已建立的为了兼容性保持原样。

probe与我们的几乎相同：

```c
static int
gpioled_fdt_probe(device_t dev)
{
    if (!ofw_bus_status_okay(dev))
        return (ENXIO);
    if (ofw_bus_search_compatible(dev, compat_data)->ocd_data == 0)
        return (ENXIO);
    device_set_desc(dev, "OFW GPIO LEDs");
    return (BUS_PROBE_DEFAULT);
}
```

attach函数是驱动程序分叉的地方。`gpioled_fdt.c` 支持每个DT节点多个LED，遵循 `gpio-leds` 绑定，该绑定将每个LED列为单个父节点的子节点。模式是：

```c
static int
gpioled_fdt_attach(device_t dev)
{
    struct gpioled_softc *sc = device_get_softc(dev);
    phandle_t leds, child;
    ...

    leds = ofw_bus_get_node(dev);
    sc->sc_dev = dev;
    sc->sc_nleds = 0;

    for (child = OF_child(leds); child != 0; child = OF_peer(child)) {
        if (!OF_hasprop(child, "gpios"))
            continue;
        ...
    }
}
```

`OF_child` 和 `OF_peer` 是遍历设备树子节点的经典行走器。`OF_child(parent)` 返回第一个子节点或零。`OF_peer(node)` 返回下一个兄弟节点或在你到达末尾时返回零。这两行迭代惯用法是每个处理可变数量子条目的驱动程序的骨干。

在循环内部，驱动程序读取每个LED的属性：

```c
    name = NULL;
    len = OF_getprop_alloc(child, "label", (void **)&name);
    if (len <= 0) {
        OF_prop_free(name);
        len = OF_getprop_alloc(child, "name", (void **)&name);
    }
```

`OF_getprop_alloc` 为属性分配内存并返回长度。调用者负责用 `OF_prop_free` 释放缓冲区。注意回退：如果没有 `label` 属性，驱动程序使用节点的 `name` 代替。这种优雅的回退值得注意；它使驱动程序对绑定变体更宽容。

每个GPIO然后通过 `gpio_pin_get_by_ofw_idx` 调用获取，带有显式索引零，因为每个LED自己的 `gpios` 属性在该子节点范围内从零开始索引。驱动程序调用 `gpio_pin_setflags(pin, GPIO_PIN_OUTPUT)` 并用 `led(4)` 框架注册每个LED，使其在用户空间显示为 `/dev/led/<name>`。

### DRIVER_MODULE注册

模块注册行看起来像这样：

```c
static driver_t gpioled_driver = {
    "gpioled",
    gpioled_methods,
    sizeof(struct gpioled_softc)
};

DRIVER_MODULE(gpioled, ofwbus,    gpioled_driver, 0, 0);
DRIVER_MODULE(gpioled, simplebus, gpioled_driver, 0, 0);
MODULE_VERSION(gpioled, 1);
MODULE_DEPEND(gpioled, gpiobus, 1, 1, 1);
SIMPLEBUS_PNP_INFO(compat_data);
```

两个添加突出。`MODULE_DEPEND(gpioled, gpiobus, 1, 1, 1)` 我们已经见过了。新行是 `SIMPLEBUS_PNP_INFO(compat_data)`。这个宏扩展为一组模块元数据，`devmatch(8)` 等工具使用它来决定为给定DT节点自动加载哪个驱动程序。参数是probe使用的相同 `compat_data` 表，所以只有一个真相来源。

当你编写生产级驱动程序时，包含 `SIMPLEBUS_PNP_INFO` 以便自动加载工作。没有它，你的驱动程序不会被自动拾取，用户必须显式将它添加到 `loader.conf`。

### 从gpioled_fdt.c带走什么

将它与 `edled` 一起阅读，你会看到：

- 如何迭代DT节点中的多个子节点。
- 如何在属性名称之间回退。
- 如何使用 `OF_getprop_alloc` 和 `OF_prop_free` 处理变长字符串。
- 如何同时注册到 `led(4)` 框架和Newbus。
- 如何为自动匹配声明PNP信息。

这是在FreeBSD驱动程序中反复出现的五种模式。在真实源文件中见过一次后，你会在下一个打开的驱动程序中立即认出它们。

### bcm2835_gpio.c：总线提供者

`edled` 和 `gpioled_fdt.c` 是GPIO消费者。它们*使用*另一个驱动程序提供的GPIO引脚。在Raspberry Pi上*提供*这些引脚的驱动程序是 `/usr/src/sys/arm/broadcom/bcm2835/bcm2835_gpio.c`。阅读它展示了交易的另一面。

驱动程序的attach做得比我们的多得多：

- 分配GPIO控制器寄存器块的MMIO资源。
- 分配所有中断资源（BCM2835每个bank路由两条中断线）。
- 初始化互斥锁和跟踪每个引脚状态的驱动程序数据结构。
- 注册GPIO总线子设备以便消费者可以附加到它。
- 为所有可复用引脚注册pinmux功能。

从我们作为消费者的角度来看，最值得注意的事情是它暴露自己的方式。在attach深处：

```c
if ((sc->sc_busdev = gpiobus_attach_bus(dev)) == NULL) {
    device_printf(dev, "could not attach GPIO bus\n");
    return (ENXIO);
}
```

`gpiobus_attach_bus(dev)` 是创建消费者随后探测的gpiobus实例的调用。没有这个调用，任何消费者驱动程序都无法获取引脚，因为没有总线来解析phandle。

在文件底部，`DEVMETHOD` 条目将GPIO总线方法映射到驱动程序自己的函数：

```c
DEVMETHOD(gpio_pin_set,    bcm_gpio_pin_set),
DEVMETHOD(gpio_pin_get,    bcm_gpio_pin_get),
DEVMETHOD(gpio_pin_toggle, bcm_gpio_pin_toggle),
DEVMETHOD(gpio_pin_getcaps, bcm_gpio_pin_getcaps),
DEVMETHOD(gpio_pin_setflags, bcm_gpio_pin_setflags),
```

这些是我们的消费者每次做 `gpio_pin_set_active` 时间接调用的函数。`gpiobusvar.h` 中的消费者API是这个DEVMETHOD表上的薄层。

### ofw_iicbus.c：既是父又是子的总线

许多I2C控制器作为 `simplebus` 的子设备（它们的DT父节点）附加，然后自己作为单独I2C设备驱动程序的父节点。`/usr/src/sys/dev/iicbus/ofw_iicbus.c` 是一个值得浏览的好例子。它展示了驱动程序如何可以：

- 像任何FDT驱动程序一样探测和附加到自己的DT节点。
- 从其节点的DT子节点注册自己的子设备。

对子节点的遍历使用相同的 `OF_child`/`OF_peer` 惯用法，但对于每个子节点它用 `device_add_child` 创建新的Newbus设备，设置自己的OFW元数据，并依赖Newbus为能处理它的驱动程序运行探测（例如温度传感器或EEPROM）。

阅读这个驱动程序给你总线与消费者关系如何级联的感觉。FDT是一棵树；Newbus层次结构也是。树中间的驱动程序同时扮演父和子的角色。

### ofw_bus_subr.c：辅助函数本身

当你发现自己在不断查找 `ofw_bus_search_compatible` 到底做什么时，答案在 `/usr/src/sys/dev/ofw/ofw_bus_subr.c`。阅读你调用的辅助函数是一个被低估的理解驱动程序真正在做什么的方式。

简要浏览你最常遇到的辅助函数：

- `ofw_bus_is_compatible(dev, str)` 如果节点的 `compatible` 列表包含 `str` 则返回true。它遍历兼容列表中的所有条目，不只是第一个。
- `ofw_bus_search_compatible(dev, table)` 将相同的兼容列表与 `struct ofw_compat_data` 表中的条目进行匹配，并返回指向匹配条目的指针（或哨兵）。
- `ofw_bus_status_okay(dev)` 检查 `status` 属性。缺失状态默认为okay；`"okay"` 或 `"ok"` 是okay；其他任何东西（`"disabled"`、`"fail"`）都不是。
- `ofw_bus_has_prop(dev, prop)` 测试存在性而不读取。
- `ofw_bus_parse_xref_list_alloc` 和相关辅助函数读取phandle引用列表（`clocks`、`resets`、`gpios` 等使用的格式）并返回调用者必须释放的已分配数组。

阅读这些辅助函数确认系统中没有什么神奇的东西。它们是可读的C代码，遍历内核在启动时解析的同一个blob。

### simplebus.c：运行你驱动程序的驱动程序

如果你想理解为什么你的FDT驱动程序实际被探测，打开 `/usr/src/sys/dev/fdt/simplebus.c`。simplebus本身的probe和attach很短，一旦你知道要找什么，令人惊讶地具体。

`simplebus_probe` 检查节点有 `compatible = "simple-bus"`（或是SoC类节点）且没有父节点特定的特殊性。`simplebus_attach` 然后遍历节点的子节点，为每个创建新设备，解析每个子节点的 `reg` 和中断，并在新设备上调用 `device_probe_and_attach`。最后一个调用就是触发你驱动程序probe的东西。

关键行大致如下：

```c
for (node = OF_child(parent); node > 0; node = OF_peer(node)) {
    ...
    child = simplebus_add_device(bus, node, 0, NULL, -1, NULL);
    if (child == NULL)
        continue;
}
```

这个迭代就是将DT节点树转换为Newbus设备树的东西。存在的每个FDT驱动程序都通过这个循环进入系统。

阅读 `simplebus.c` 揭秘了"为什么我的驱动程序被调用"的问题。你看到，用普通的C，内核如何精确地从内存中的blob走到你的驱动程序probe的调用。如果你需要排查为什么你的probe没有运行，第一步是在正确的位置用 `device_printf` 器械化这个文件。

### 值得阅读的驱动程序调查

除了上面特定的驱动程序外，这里有一个 `/usr/src/sys` 中值得你花时间作为学习对象的FDT驱动程序简短列表。每个代表你可能遇到的模式。

- `/usr/src/sys/dev/gpio/gpioiic.c`：在GPIO引脚之上实现I2C总线的驱动程序。展示bit-banging模式。
- `/usr/src/sys/dev/gpio/gpiokeys.c`：将GPIO输入消费为键盘。展示来自GPIO的中断处理。
- `/usr/src/sys/dev/uart/uart_dev_ns8250.c`：带有FDT挂钩的平台无关UART驱动程序。展示通用驱动程序如何接受FDT附加路径以及其他总线类型。
- `/usr/src/sys/dev/sdhci/sdhci_fdt.c`：SD主机控制器的大型FDT驱动程序。展示生产驱动程序如何一起处理时钟、复位、稳压器和pinmux。
- `/usr/src/sys/arm/allwinner/aw_gpio.c`：Allwinner SoC系列的完整现代GPIO控制器。值得与 `bcm2835_gpio.c` 比较以查看同一问题的两种解决方案。
- `/usr/src/sys/arm/freescale/imx/imx_gpio.c`：i.MX6/7/8 GPIO驱动程序，另一个维护良好的参考。
- `/usr/src/sys/dev/extres/syscon/syscon.c`：将共享寄存器块暴露给多个驱动程序的"系统控制器"伪总线。有助于了解FreeBSD如何处理不能干净地适合"一个节点一个驱动程序"的DT模式。

你不需要从头到尾阅读这些。健康的习惯是每隔一两周挑选一个，浏览结构，然后专注于任何吸引你注意的小细节。随着时间的推移，这些阅读将在你脑中建立一个你见过工作的真实代码库。

### 使用grep作为学习工具

当你在阅读的驱动程序中发现一个新函数但不确定它做什么时，好的第一步是：

```console
$ grep -rn "function_name" /usr/src/sys | head
```

这显示函数被定义和调用的每个地方。通常头文件中的声明，结合两三个代表性调用点，就足以理解函数的用途。这比搜索网络更好，后者返回过时的文档和半记忆的论坛帖子。

对于特定的DT绑定，同样的技巧有效：

```console
$ grep -rn '"gpio-leds"' /usr/src/sys
```

输出告诉你引用该兼容字符串的每个文件，包括实现它的驱动程序、使用它的overlay以及测试它的测试。

### 总结本节

第9节给了你一个阅读列表和一个方法。真实的FreeBSD驱动程序是可用的最丰富资源，学会有效地阅读它们是一项与编写自己的驱动程序同样重要的技能。上面列表中的驱动程序展示了我们的教学示例简化了的模式。当你遇到困难、需要灵感，以及想知道生产级驱动程序如何处理你自己代码尚未遇到的边界情况时，它们是你应该去寻找的。

## 第10节：基于FDT系统中的中断管道

我们大部分时间将中断视为不透明的。在本节中我们打开盒子，看看设备树如何描述中断连接性、FreeBSD的中断框架（`intrng`）如何消费该描述，以及驱动程序如何请求一个在硬件需要关注时真正会触发的IRQ。

这个主题值得拥有自己的一节的原因是，现代SoC上的中断接线可能变得复杂。简单的平台有一个控制器、一组线路和一个扁平分配。复杂的平台有一个根控制器，几个将更宽的IRQ源多路复用到更窄输出的附属控制器，以及基于引脚的控制器（如GPIO作为中断），它们的线路通过多路复用树链接。理解链的驱动程序编写者可以在几分钟内调试奇怪的中断失败；不理解的人可能花数小时检查错误的东西。

### 中断树

设备树将中断表达为与主地址树并行的独立逻辑树。每个节点有一个中断父节点（其控制器），树通过控制器向上攀升，直到到达CPU实际从中获取异常的根控制器。

三个属性描述该树：

- **`interrupts`**：此节点的中断描述。其格式取决于它附加到的控制器。
- **`interrupt-parent`**：指向控制器的phandle，如果最近的祖先已经是一个中断控制器则不需要。
- **`interrupt-controller`**：标记节点为控制器的空值属性。消费者的 `interrupt-parent` 必须指向这样的节点。

一个示例片段：

```dts
&soc {
    gic: interrupt-controller@10000 {
        compatible = "arm,gic-v3";
        interrupt-controller;
        #interrupt-cells = <3>;
        reg = <0x10000 0x1000>, <0x11000 0x20000>;
    };

    uart0: serial@20000 {
        compatible = "arm,pl011";
        reg = <0x20000 0x100>;
        interrupts = <0 42 4>;
        interrupt-parent = <&gic>;
    };
};
```

GIC（通用中断控制器，标准arm64根控制器）声明 `#interrupt-cells = <3>`。每个附加到它的设备必须在其 `interrupts` 属性中提供三个cell。对于GICv3，三个cell是*类型、编号、标志*：`<0 42 4>` 表示"共享外设中断42，电平触发，高电平"。

如果省略 `interrupt-parent`，父节点是最近的具有 `interrupt-parent` 或设置了 `interrupt-controller` 属性的祖先。当驱动程序位于数层深度时，这个链可能不明显。

### 中断父节点链

考虑一个更实际的例子。在BCM2711（Raspberry Pi 4）上，GPIO控制器是自己的中断控制器：它将来自各个引脚的中断聚合成少数输出，馈送到GIC。连接到GPIO引脚的按钮在DT中看起来像这样：

```dts
&gpio {
    button_pins: button_pins {
        brcm,pins = <23>;
        brcm,function = <0>;       /* GPIO input */
        brcm,pull = <2>;           /* pull-up */
    };
};

button_node: button {
    compatible = "gpio-keys";
    pinctrl-names = "default";
    pinctrl-0 = <&button_pins>;
    key_enter {
        label = "enter";
        linux,code = <28>;
        gpios = <&gpio 23 0>;
        interrupt-parent = <&gpio>;
        interrupts = <23 3>;       /* edge trigger */
    };
};
```

两个属性命名了父控制器：`gpios = <&gpio ...>` 将GPIO控制器命名为引脚提供者，`interrupt-parent = <&gpio>` 将同一个控制器命名为中断提供者。这两个角色是不同的，必须独立声明。

GPIO控制器然后聚其中断线并向GIC报告。在GPIO驱动程序内部，当来自GIC的中断到达时，它识别哪个引脚触发了，并将事件派发给为该引脚的IRQ资源注册了处理程序的任何驱动程序。

当你的驱动程序为此节点请求中断时，FreeBSD遍历链：按钮驱动程序请求IRQ，GPIO控制器的intrng逻辑分配虚拟IRQ号，最终内核安排GIC的上游IRQ调用GPIO驱动程序的派发器，后者又调用按钮驱动程序的处理程序。你不需要自己编写任何这些管道；你只需请求IRQ并处理它。

### intrng框架

FreeBSD的 `intrng`（中断下一代）子系统是统一所有这些的组件。中断控制器实现 `pic_*` 方法：

```c
static device_method_t gpio_methods[] = {
    ...
    DEVMETHOD(pic_map_intr,      gpio_pic_map_intr),
    DEVMETHOD(pic_setup_intr,    gpio_pic_setup_intr),
    DEVMETHOD(pic_teardown_intr, gpio_pic_teardown_intr),
    DEVMETHOD(pic_enable_intr,   gpio_pic_enable_intr),
    DEVMETHOD(pic_disable_intr,  gpio_pic_disable_intr),
    ...
};
```

`pic_map_intr` 是读取DT属性并返回IRQ内部表示的那个。`pic_setup_intr` 附加处理程序。其余方法控制屏蔽和确认。

消费者驱动程序从不直接调用这些。它调用 `bus_alloc_resource_any(dev, SYS_RES_IRQ, ...)`，Newbus连同OFW资源代码遍历DT和intrng框架来解析IRQ。

### 实践中请求IRQ

FDT驱动程序中中断处理的完整形状如下：

```c
struct driver_softc {
    ...
    struct resource *irq_res;
    void *irq_cookie;
    int irq_rid;
};

static int
driver_attach(device_t dev)
{
    struct driver_softc *sc = device_get_softc(dev);
    int error;

    sc->irq_rid = 0;
    sc->irq_res = bus_alloc_resource_any(dev, SYS_RES_IRQ,
        &sc->irq_rid, RF_ACTIVE);
    if (sc->irq_res == NULL) {
        device_printf(dev, "cannot allocate IRQ\n");
        return (ENXIO);
    }

    error = bus_setup_intr(dev, sc->irq_res,
        INTR_TYPE_MISC | INTR_MPSAFE,
        NULL, driver_intr, sc, &sc->irq_cookie);
    if (error != 0) {
        device_printf(dev, "cannot setup interrupt: %d\n", error);
        bus_release_resource(dev, SYS_RES_IRQ, sc->irq_rid,
            sc->irq_res);
        return (error);
    }
    ...
}
```

中断的RID从零开始，为节点的 `interrupts` 列表中每个IRQ递增。有两个IRQ的节点将连续使用RID 0和1。

`bus_setup_intr` 注册处理程序。第四个参数是过滤函数（在中断上下文中运行）；第五个是线程处理程序（在专用内核线程中运行）。你不使用的那个传 `NULL`。`INTR_MPSAFE` 标志告诉框架处理程序不要求Giant锁。

detach中的拆除：

```c
static int
driver_detach(device_t dev)
{
    struct driver_softc *sc = device_get_softc(dev);

    if (sc->irq_cookie != NULL)
        bus_teardown_intr(dev, sc->irq_res, sc->irq_cookie);
    if (sc->irq_res != NULL)
        bus_release_resource(dev, SYS_RES_IRQ, sc->irq_rid,
            sc->irq_res);
    return (0);
}
```

未调用 `bus_teardown_intr` 是经典的卸载bug：IRQ保持连接到已释放的内存，下次触发时内核panic。

### 过滤器与线程处理程序

过滤器和线程处理程序之间的区别是新内核开发者经常感到困惑的主题之一。一个简短的入门有帮助。

*过滤器*在中断本身的上下文中运行，在高IPL，对它可以调用的东西有严格限制。它不能睡眠，不能分配内存，不能获取常规的可睡眠互斥锁。它只能获取自旋互斥锁。其目的是决定中断是否属于此设备，确认硬件条件，并要么简单地处理事件，要么调度线程处理程序做其余工作。

*线程处理程序*在专用内核线程中运行。它可以睡眠、分配和获取可睡眠锁。许多驱动程序在线程处理程序中完成所有工作，让过滤器为空。

对于像 `edled` 这样简单的驱动程序，我们从不在处理中断。如果我们扩展它来处理按钮，我们将从线程处理程序开始，只在性能分析表明必要时才引入过滤器。

### 边沿触发与电平触发

GIC `interrupts` 三元组的第三个cell是触发类型。常见值：

- `1`：上升沿
- `2`：下降沿
- `3`：任意沿
- `4`：高电平有效
- `8`：低电平有效

GPIO作为中断的节点使用不同的cell计数（通常为两个）和类似的编码。选择很重要。边沿触发的中断每次转换触发一次；电平触发的中断只要线路被断言就持续触发。在电平触发线上确认太晚的驱动程序最终会陷入中断风暴。

每个控制器的DT绑定文档指定了确切的cell计数和标志语义。有疑问时，在 `/usr/src/sys/contrib/device-tree/Bindings/` 中grep控制器系列。

### 调试不触发的中断

配置错误的中断症状通常很明确：硬件工作第一次，后续中断永远不来；或者系统启动但驱动程序的处理程序从不运行。

按顺序检查：

1. **`vmstat -i` 是否显示中断正在被计数？** 如果是，硬件正在断言但驱动程序没有确认它。检查你的过滤器或线程处理程序。
2. **DT的 `interrupts` 是否匹配控制器期望的格式？** cell计数和值是常见的罪魁祸首。
3. **`interrupt-parent` 是否指向正确的控制器？** 如果基于引脚的控制器是源但DT说的是GIC，请求将失败，因为GIC的cell格式不匹配。
4. **`bus_setup_intr` 是否返回了零？** 如果没有，读错误码。`EINVAL` 通常意味着IRQ资源没有完全映射；`ENOENT` 意味着IRQ号没有被任何控制器声明。

`intr_event_show` DTrace探针可以帮助高级调试，但上面四步检查不需要DTrace就能捕获大多数问题。

### 真实示例：gpiokeys.c

`/usr/src/sys/dev/gpio/gpiokeys.c` 值得阅读作为使用中断的GPIO消费者驱动程序的工作示例。对于每个子节点，它获取引脚，将其配置为输入，并通过 `gpio_alloc_intr_resource` 和 `bus_setup_intr` 挂接中断。过滤器非常短：它只是在一个工作项上调用 `taskqueue_enqueue`。实际的按键处理在内核的taskqueue上运行，而不是在中断上下文中。

这是小型中断驱动驱动程序的干净模式。一个只发信号的过滤器，一个做工作的worker。当你需要为自定义板外设实现类似的东西时，gpiokeys驱动程序是一个好模板。

### 总结本节

第10节拆开了我们早期示例保持隐藏的中断机械。你现在知道设备树如何描述中断连接性，FreeBSD的intrng如何将IRQ请求解析为具体的处理程序注册，过滤器和线程处理程序如何分工，以及如何调试配置错误的中断产生的失败类别。

本章的技术覆盖现在真正完成了。实验、练习和故障排除材料紧随其后。

## 动手实验

本章没有什么能在不实际运行的情况下记住。接下来的实验按难度排列。实验1是一个热身，你可以在任何带有内核源码的FreeBSD系统上完成，甚至是QEMU中的通用amd64笔记本电脑。实验2引入overlay，这意味着你需要一个arm64目标，要么是真实的（Raspberry Pi 4、BeagleBone、Pine64）要么是模拟的。实验3是一个调试练习，你会故意破坏一个DT并学习症状。实验4构建完整的 `edled` 驱动程序并通过它驱动LED。

所有实验文件发布在 `examples/part-07/ch32-fdt-embedded/`。每个实验有自己的子目录，包含 `README.md` 和所有你需要的源码。以下文本是自包含的，所以你可以直接从书中工作，但示例树在你想要将工作与已知良好参考对比时作为安全网存在。

### 实验1：构建并加载fdthello骨架

**目标：** 编译第4节的最小FDT感知驱动程序，在运行的FreeBSD系统上加载它，并确认即使不存在匹配DT节点时它也会向内核注册。

**你将学到：**

- 内核模块Makefile如何工作。
- `kldload` 和 `kldunload` 如何与模块注册交互。
- Newbus如何在驱动程序被引入时立即运行探测。

**步骤：**

1. 在安装了内核源码的FreeBSD 14.3系统上创建一个名为 `lab01-fdthello` 的临时目录。

2. 将第4节的完整 `fdthello.c` 源码清单保存到该目录。

3. 保存以下内容的Makefile：

   ```
   KMOD=   fdthello
   SRCS=   fdthello.c

   SYSDIR?= /usr/src/sys

   .include <bsd.kmod.mk>
   ```

4. 构建模块：

   ```
   $ make
   ```

   干净的构建在当前目录产生 `fdthello.ko` 和 `fdthello.ko.debug`。

5. 加载模块：

   ```
   # kldload ./fdthello.ko
   ```

   在没有匹配DT节点的系统上，没有探测成功。这是预期的。模块驻留，但没有 `fdthello0` 设备出现。

6. 验证模块已加载：

   ```
   # kldstat -m fdthello
   ```

7. 卸载：

   ```
   # kldunload fdthello
   ```

**预期结果：**

构建无警告完成。模块干净地加载和卸载。`kldstat` 在两个步骤之间显示 `fdthello.ko`。

**如果遇到障碍：**

- **`kldload` 报告 "module not found":** 确保你用了带前导 `./` 的 `./fdthello.ko`，这样 `kldload` 不会尝试系统模块路径。
- **构建失败 "no such file `bsd.kmod.mk`":** 通过 `pkgbase` 安装 `/usr/src` 或从git检出。
- **构建失败因为内核符号缺失:** 确认 `/usr/src/sys` 与运行内核的版本匹配。运行内核和源码树之间的不匹配是通常的原因。

起始文件位于 `examples/part-07/ch32-fdt-embedded/lab01-fdthello/`。

### 实验2：构建并部署Overlay

**目标：** 添加一个匹配 `fdthello` 驱动程序的DT节点，在arm64 FreeBSD板上作为overlay部署它，并观察驱动程序附加。

**你将学到：**

- 如何编写overlay源文件。
- `dtc -@` 如何产生overlay就绪的 `.dtbo` 输出。
- FreeBSD加载程序如何通过 `loader.conf` 的 `fdt_overlays` 应用overlay。
- 如何验证overlay正确落地。

**步骤：**

1. 在运行的arm64 FreeBSD系统（Raspberry Pi 4是参考目标）上，安装 `dtc`：

   ```
   # pkg install dtc
   ```

2. 在临时目录中，保存以下overlay源码为 `fdthello-overlay.dts`：

   ```
   /dts-v1/;
   /plugin/;

   / {
       compatible = "brcm,bcm2711";

       fragment@0 {
           target-path = "/soc";
           __overlay__ {
               hello@20000 {
                   compatible = "freebsd,fdthello";
                   reg = <0x20000 0x100>;
                   status = "okay";
               };
           };
       };
   };
   ```

3. 编译overlay：

   ```
   $ dtc -I dts -O dtb -@ -o fdthello.dtbo fdthello-overlay.dts
   ```

4. 将结果复制到加载程序的overlay目录：

   ```
   # cp fdthello.dtbo /boot/dtb/overlays/
   ```

5. 编辑 `/boot/loader.conf`（如果缺失则创建）以包含：

   ```
   fdt_overlays="fdthello"
   ```

6. 将你在实验1构建的 `fdthello.ko` 复制到 `/boot/modules/`：

   ```
   # cp /path/to/fdthello.ko /boot/modules/
   ```

7. 确保 `/boot/loader.conf` 中有 `fdthello_load="YES"`：

   ```
   fdthello_load="YES"
   ```

8. 重启：

   ```
   # reboot
   ```

9. 重启后，确认：

   ```
   # dmesg | grep fdthello
   fdthello0: <FDT Hello Example> on simplebus0
   fdthello0: attached, node phandle 0x...
   ```

**预期结果：**

驱动程序在启动时附加，其消息出现在 `dmesg`。`ofwdump -p /soc/hello@20000` 打印节点属性。

**如果遇到障碍：**

- **加载程序打印 "error loading overlay":** 通常 `.dtbo` 文件缺失或在错误目录。确认它在 `/boot/dtb/overlays/` 下且名称带 `.dtbo` 扩展名。
- **驱动程序不附加:** 使用第6节的检查清单：节点存在、状态okay、兼容完全匹配、父节点 `simplebus`。
- **你在非Pi板子上:** 更改overlay中顶层 `compatible` 以匹配你板子的基础兼容。`ofwdump -p /` 显示当前值。

起始文件位于 `examples/part-07/ch32-fdt-embedded/lab02-overlay/`。

### 实验3：调试损坏的设备树

**目标：** 给定一个故意损坏的overlay，识别三种不同的失败模式并修复每一个。

**你将学到：**

- 如何使用 `dtc`、`fdtdump` 和 `ofwdump` 读取blob。
- 如何将树内容与内核探测行为关联。
- 如何使用 `device_printf` 面包屑诊断探测不匹配。

**步骤：**

1. 将以下损坏的overlay复制到 `lab03-broken.dts`：

   ```
   /dts-v1/;
   /plugin/;

   / {
       compatible = "brcm,bcm2711";

       fragment@0 {
           target-path = "/soc";
           __overlay__ {
               hello@20000 {
                   compatible = "free-bsd,fdthello";
                   reg = <0x20000 0x100>;
                   status = "disabled";
               };
           };
       };

       fragment@1 {
           target-path = "/soc";
           __overlay__ {
               hello@30000 {
                   compatible = "freebsd,fdthello";
                   reg = <0x30000>;
                   status = "okay";
               };
           };
       };
   };
   ```

2. 编译并安装overlay：

   ```
   $ dtc -I dts -O dtb -@ -o lab03-broken.dtbo lab03-broken.dts
   # cp lab03-broken.dtbo /boot/dtb/overlays/
   ```

3. 编辑 `/boot/loader.conf` 加载这个overlay而不是 `fdthello`：

   ```
   fdt_overlays="lab03-broken"
   ```

4. 重启。观察：

   - 没有 `fdthello0` 设备附加。
   - `dmesg` 可能沉默或可能显示关于 `hello@30000` 的FDT解析警告。

5. 诊断。按顺序使用以下技术：

   **a) 比较兼容字符串：**

   ```
   # ofwdump -P compatible /soc/hello@20000
   # ofwdump -P compatible /soc/hello@30000
   ```

   第一个打印 `free-bsd,fdthello`，驱动程序不匹配。`free` 后的连字符是拼写错误。修复是将字符串纠正为 `freebsd,fdthello`。

   **b) 检查状态：**

   ```
   # ofwdump -P status /soc/hello@20000
   ```

   返回 `disabled`。即使兼容正确，驱动程序仍会跳过此节点。修复是设置 `status = "okay"`。

   **c) 检查reg属性：**

   ```
   # ofwdump -P reg /soc/hello@30000
   ```

   注意 `reg` 只有一个cell，而父节点期望地址加大小。在 `#address-cells = <1>` 和 `#size-cells = <1>` 的父节点下，`reg` 必须有两个cell。驱动程序会附加，但如果它分配资源，它会将大小误解为后面跟随的任何垃圾。修复是 `reg = <0x30000 0x100>;`。

6. 应用修复，重新编译，重新安装overlay，并重启。驱动程序应附加到一个或两个hello节点。

**预期结果：**

在所有三个修复后，`dmesg | grep fdthello` 显示两个附加设备，`hello@20000` 和 `hello@30000`，每个通过 `simplebus` 报告。

**如果遇到障碍：**

- **ofwdump 报告 "no such_node":** overlay没有应用。检查加载程序输出寻找overlay加载消息，检查 `.dtbo` 在加载程序期望的位置。
- **只有一个hello设备附加:** 三种bug中的一种仍存在。
- **内核panic:** 你几乎肯定因为cell计数仍错误而读到 `reg` 末尾之外。在诊断时回退到已知良好overlay。

起始文件位于 `examples/part-07/ch32-fdt-embedded/lab03-debug-broken/`。

### 实验4：端到端构建edled驱动程序

**目标：** 从第7节构建完整的 `edled` 驱动程序，编译它，通过DT overlay将其附加到Raspberry Pi 4的GPIO18，并从用户空间切换LED。

**你将学到：**

- 如何将GPIO资源获取集成到FDT驱动程序中。
- 如何暴露驱动硬件的sysctl。
- 如何使用 `dmesg`、`sysctl`、`ofwdump` 和 `devinfo -r` 在运行的系统上验证驱动程序。

**步骤：**

1. 在GPIO18（头上的引脚12）和地之间通过330欧姆电阻连接一个LED。如果你没有物理硬件，仍可以继续；驱动程序会附加并切换其逻辑状态，但没有东西会亮起。

2. 在临时目录中保存：

   - 来自第7节完整清单的 `edled.c`。
   - `KMOD=edled`、`SRCS=edled.c`、`SYSDIR?=/usr/src/sys` 和 `.include <bsd.kmod.mk>` 的 `Makefile`。
   - 来自第7节的 `edled.dts` overlay源码。

3. 构建模块：

   ```
   $ make
   ```

4. 编译overlay：

   ```
   $ dtc -I dts -O dtb -@ -o edled.dtbo edled.dts
   ```

5. 安装：

   ```
   # cp edled.ko /boot/modules/
   # cp edled.dtbo /boot/dtb/overlays/
   ```

6. 编辑 `/boot/loader.conf`：

   ```
   edled_load="YES"
   fdt_overlays="edled"
   ```

7. 重启。

8. 确认附加：

   ```
   # dmesg | grep edled
   edled0: <Example embedded LED> on simplebus0
   edled0: attached, GPIO pin acquired, state=0
   ```

9. 操作sysctl：

   ```
   # sysctl dev.edled.0.state
   dev.edled.0.state: 0

   # sysctl dev.edled.0.state=1
   dev.edled.0.state: 0 -> 1
   ```

   LED亮起。读回并确认：

   ```
   # sysctl dev.edled.0.state
   dev.edled.0.state: 1
   ```

10. 关闭LED并卸载驱动程序：

    ```
    # sysctl dev.edled.0.state=0
    # kldunload edled
    ```

**预期结果：**

驱动程序加载、附加、切换LED，并卸载而不留下资源在使用中。`gpioctl -l` 显示卸载后引脚返回到未配置状态。

**如果遇到障碍：**

- **dmesg 显示 "cannot get GPIO pin":** GPIO控制器模块尚未附加。验证 `gpiobus` 已加载：`kldstat -m gpiobus`。如果没有，在重试前 `kldload gpiobus`。
- **LED不亮:** 检查极性。如果DT中的标志是 `0`（高电平有效），引脚在活动时驱动3.3V。如果LED阴极连接到引脚，你要 `1`（低电平有效）。
- **kldunload 失败返回EBUSY:** 某进程仍有 `dev.edled.0` 打开，或驱动程序的detach路径留下了已获取的资源。审计detach。

起始文件位于 `examples/part-07/ch32-fdt-embedded/lab04-edled/`。

### 实验5：扩展edled以消费GPIO中断

**目标：** 修改实验4的 `edled` 驱动程序，使第二个GPIO（配置为带上拉电阻的输入）成为中断源。当引脚接地（一个按钮将其拉低）时，处理程序切换LED。

**你将学到：**

- 如何通过 `gpio_alloc_intr_resource` 获取GPIO中断资源。
- 如何用 `bus_setup_intr` 设置线程上下文处理程序。
- 如何通过共享状态协调中断路径和sysctl路径。
- 如何在detach中干净地拆除中断处理程序。

**步骤：**

1. 从实验4的 `edled.c` 开始。将其复制到新临时目录中的 `edledi.c`。

2. 向softc和DT绑定添加第二个GPIO。新的DT节点看起来像：

   ```
   edledi0: edledi@0 {
       compatible = "example,edledi";
       status = "okay";
       leds-gpios = <&gpio 18 0>;
       button-gpios = <&gpio 23 1>;
       interrupt-parent = <&gpio>;
       interrupts = <23 3>;
   };
   ```

   按钮使用GPIO 23，以通常的按钮布置接线：一条腿到引脚，另一条到地，带上拉到3.3V。

3. 更新兼容字符串表为 `"example,edledi"`，使驱动程序匹配新绑定。

4. 在softc中添加：

   ```c
   gpio_pin_t      sc_button;
   struct resource *sc_irq;
   void            *sc_irq_cookie;
   int             sc_irq_rid;
   ```

5. 在attach中，获取LED引脚后，获取按钮引脚和其中断：

   ```c
   error = gpio_pin_get_by_ofw_property(dev, node,
       "button-gpios", &sc->sc_button);
   if (error != 0) {
       device_printf(dev, "cannot get button pin: %d\n", error);
       goto fail;
   }

   error = gpio_pin_setflags(sc->sc_button,
       GPIO_PIN_INPUT | GPIO_PIN_PULLUP);
   if (error != 0) {
       device_printf(dev, "cannot configure button: %d\n", error);
       goto fail;
   }

   sc->sc_irq_rid = 0;
   sc->sc_irq = bus_alloc_resource_any(dev, SYS_RES_IRQ,
       &sc->sc_irq_rid, RF_ACTIVE);
   if (sc->sc_irq == NULL) {
       device_printf(dev, "cannot allocate IRQ\n");
       goto fail;
   }

   error = bus_setup_intr(dev, sc->sc_irq,
       INTR_TYPE_MISC | INTR_MPSAFE,
       NULL, edledi_intr, sc, &sc->sc_irq_cookie);
   if (error != 0) {
       device_printf(dev, "cannot setup interrupt: %d\n", error);
       goto fail;
   }
   ```

6. 编写中断处理程序：

   ```c
   static void
   edledi_intr(void *arg)
   {
       struct edled_softc *sc = arg;

       mtx_lock(&sc->sc_mtx);
       sc->sc_on = !sc->sc_on;
       (void)gpio_pin_set_active(sc->sc_pin, sc->sc_on);
       mtx_unlock(&sc->sc_mtx);
   }
   ```

   这是一个线程处理程序（作为第五个参数传给 `bus_setup_intr`，第四个为 `NULL` 作为过滤器）。它安全地获取互斥锁并调用GPIO框架。

7. 在detach中，以相反顺序添加拆除：

   ```c
   if (sc->sc_irq_cookie != NULL)
       bus_teardown_intr(dev, sc->sc_irq, sc->sc_irq_cookie);
   if (sc->sc_irq != NULL)
       bus_release_resource(dev, SYS_RES_IRQ, sc->sc_irq_rid,
           sc->sc_irq);
   if (sc->sc_button != NULL)
       gpio_pin_release(sc->sc_button);
   ```

8. 重建，重新部署overlay，重启，并按下按钮。

**预期结果：**

每次按下切换LED。sysctl仍可用于程序化控制。驱动程序干净卸载。

**如果遇到障碍：**

- **中断从不触发:** 确认上拉确实在空闲时将引脚拉高；检查DT触发cell（3 = 任意沿）；看 `vmstat -i` 查看是否有IRQ被计数用于你的设备。
- **单次按下重复中断（抖动）:** 机械按钮会抖动。简单的软件防抖可以通过在第一次后短窗口内忽略中断来完成。使用 `sbintime()` 和softc中的状态字段。
- **kldunload 失败返回EBUSY:** 你漏了一个 `bus_teardown_intr` 或 `gpio_pin_release`。

起始文件位于 `examples/part-07/ch32-fdt-embedded/lab05-edledi/`。

### 实验之后

在实验4结束时，你已经走过了嵌入式驱动程序工作的整个弧线：源码、overlay、内核模块、用户空间访问、拆除。本章剩余部分给你扩展已构建内容的方式，以及对常见陷阱的最后一瞥。

## 挑战练习

以下练习超越引导实验。它们不包含分步说明，因为目的就是让你将所学应用到非结构化问题。如果遇到困难，第5到8节的参考材料和 `/usr/src/sys/dev/gpio/` 与 `/usr/src/sys/dev/fdt/` 下的真实驱动程序是你最强的资源。

### 挑战1：每节点多个LED

修改 `edled` 以接受声明多个GPIO的DT节点，像真正的 `gpioled_fdt.c` 那样。绑定应像：

```dts
edled0: edled@0 {
    compatible = "example,edled-multi";
    led-red  = <&gpio 18 0>;
    led-amber = <&gpio 19 0>;
    led-green = <&gpio 20 0>;
};
```

每个LED暴露一个sysctl：`dev.edled.0.red`、`dev.edled.0.amber`、`dev.edled.0.green`。每个应独立行为。

*提示:* 在attach中遍历DT属性。为了更干净的结构，在softc中存储一个引脚句柄数组并在attach和detach中遍历它。`gpio_pin_get_by_ofw_property` 将属性名称作为其第三个参数，所以同一个驱动程序可以用小型查找表处理不同属性名称。

### 挑战2：支持闪烁定时器

扩展 `edled` 添加第二个sysctl `dev.edled.0.blink_ms`，当设为非零值时启动内核callout每隔 `blink_ms` 毫秒切换引脚。写 `0` 停止闪烁并让LED留在当前状态。

*提示:* 用 `callout_init_mtx` 将callout与per-softc互斥锁关联，用 `callout_reset` 调度它。记得在detach中 `callout_drain`，以免系统留下指向已释放内存的调度事件。

### 挑战3：泛化为任意GPIO输出

重命名并泛化 `edled` 为 `edoutput`，一个可以通过sysctl接口驱动任意GPIO输出线的驱动程序。从DT接受 `label` 属性并将其用作sysctl路径的一部分以便多个实例不冲突。添加 `dev.edoutput.0.pulse_ms` sysctl，驱动线路活动给定的毫秒数然后返回非活动。

*提示:* `device_get_unit(dev)` 给你单元号；当你需要一个组合的名称+单元字符串时用 `device_get_nameunit(dev)`。

### 挑战4：消费中断

如果你的板子暴露一个连接到GPIO输入的按钮（或者你可以用上拉电阻和跳线作为一次性模拟），修改驱动程序来监视输入引脚的边沿转换并通过 `device_printf` 记录它们。你需要用 `bus_alloc_resource_any(dev, SYS_RES_IRQ, ...)` 获取IRQ资源，用 `bus_setup_intr` 设置中断处理程序，并在detach中干净释放。

*提示:* 查看 `/usr/src/sys/dev/gpio/gpiokeys.c` 作为从FDT消费GPIO触发中断的驱动程序参考。

### 挑战5：为QEMU产生自定义设备树

为一个假设的嵌入式板编写完整的 `.dts`，包含：

- 单个ARM Cortex-A53核心。
- 256 MB RAM。
- 一个simplebus。
- 一个选定地址的UART。
- 一个simplebus下的 `edled` 节点，引用一个GPIO控制器。
- 一个你发明的小型GPIO控制器节点。

编译结果，用 `-dtb` 在QEMU下启动FreeBSD arm64内核，并观察驱动程序附加。GPIO控制器会失败因为没有东西驱动发明的硬件，但你将用自己的源码看到从DT到Newbus的完整路径。

*提示:* 使用 `/usr/src/sys/contrib/device-tree/src/arm64/arm/juno.dts` 作为结构参考。

### 挑战6：将真实驱动程序的DT绑定移植到新板子

挑选任何现有FreeBSD FDT驱动程序（例如 `sys/dev/iicbus/pmic/act8846.c`），通过研究驱动程序源码阅读其DT绑定，并为Raspberry Pi 4编写一个会附加它的完整DT片段。你不需要实际运行驱动程序；练习是从源码阅读绑定并产生正确的 `.dtsi` 片段。

*提示:* 读驱动程序的兼容表、其probe，以及任何 `of_` 调用来发现它期望什么属性。供应商内核源码树经常在文件顶部注释中文档化DT绑定。

### 挑战7：为QEMU下的嵌入式目标手写完整DT

挑战5邀请你为假设板子编写部分DT。挑战7更进一步：为你完全自己定义的QEMU arm64目标编写完整的 `.dts`。包含内存、定时器、一个PL011 UART、一个GIC、一个PL061风格的GPIO控制器，以及你自己设计的一个外设实例。在QEMU下用你的 `.dtb` 启动未修改的FreeBSD arm64内核。验证控制台在你选定的UART上启动，内核的设备探测遍历树。

*提示:* `qemu-system-aarch64` 命令支持 `-dtb` 加 `-machine virt,dumpdtb=out.dtb` 来发出一个你可以研究和改编的参考DT。

### 挑战8：实现简单MMIO外设驱动程序

为你也在QEMU下模拟的假设MMIO外设编写驱动程序。外设在固定地址暴露一个32位寄存器。读寄存器返回一个自由运行计数器；写零重置计数器。你的驱动程序应暴露一个sysctl `dev.counter.0.value` 来读写此寄存器。验证 `bus_read_4` 和 `bus_write_4` 按预期工作。通过编写一个小型QEMU设备模型模拟硬件，或通过重新用途化一个现有模拟区域来模拟，其值你可以观察。

*提示:* `bus_read_4(sc->mem, 0)` 返回你分配的内存资源内偏移0处的u32。bus_space(9)手册页是权威参考。

### 挑战之后

这些练习故意开放。如果你完成了任何一个，你已经内化了本章材料。如果你发现自己想要更多，`/usr/src/sys/dev/` 有各种大小的数十个FDT驱动程序。每周读一个驱动程序是嵌入式FreeBSD开发者可以建立的最好习惯之一。

## 常见错误和故障排除

本节是FDT驱动程序编写者最常绊倒的问题的集中参考。这里的一切都已经在第3到8节提到，但把要点收集在一个地方给你在特定症状出现时可以浏览的东西。

### 模块加载但没有设备附加

大多数探测失败归结为五个原因之一：

1. **`compatible` 中拼写错误。** 无论在DT源码还是驱动程序兼容表中。字符串必须逐字节匹配。
2. **节点有 `status = "disabled"`。** 要么修复基础树，要么写一个将status设为okay的overlay。
3. **错误的父总线。** 如果节点位于I2C或SPI控制器节点下，驱动程序必须向那个控制器的驱动程序类型注册，而不是 `simplebus`。
4. **Overlay没有应用。** 检查启动时加载程序输出寻找错误消息。确认 `.dtbo` 在 `/boot/dtb/overlays/` 且列在 `loader.conf` 中。
5. **驱动程序被另一个驱动程序超越。** 用 `devinfo -r` 看哪个驱动程序实际附加了。增加probe的返回值（`BUS_PROBE_DEFAULT` 是常见基线；`BUS_PROBE_SPECIFIC` 表示更精确匹配）。

### Overlay在启动时未能应用

加载程序在控制台打印其尝试。观察类似以下的行：

```text
Loading DTB overlay 'edled' (0x1200 bytes)
```

如果那行缺失，加载程序要么没找到文件要么跳过了。可能原因：

- 文件名以 `.dtbo` 结尾但 `fdt_overlays` 拼错了它。
- 文件在错误目录。默认是 `/boot/dtb/overlays/`。
- 基础blob和overlay在顶层 `compatible` 上不一致。加载程序拒绝应用顶层兼容与基础不匹配的overlay。
- `.dtbo` 没有用 `-@` 编译并引用了加载程序无法解析的标签。

### 无法分配资源

如果attach调用 `bus_alloc_resource_any(dev, SYS_RES_MEMORY, ...)` 并返回 `NULL`，最可能的原因是：

- DT节点中 `reg` 缺失或畸形。
- 节点的 `reg` 与父节点的 `#address-cells`/`#size-cells` 之间cell计数不匹配。
- 另一个驱动程序已经声明了同一区域。
- 父总线的 `ranges` 不覆盖请求的地址。

在调试时在attach中打印资源起始和大小：

```c
if (sc->mem == NULL) {
    device_printf(dev, "cannot allocate memory resource (rid=%d)\n",
        sc->mem_rid);
    goto fail;
}
device_printf(dev, "memory at %#jx len %#jx\n",
    (uintmax_t)rman_get_start(sc->mem),
    (uintmax_t)rman_get_size(sc->mem));
```

### GPIO获取失败

`gpio_pin_get_by_ofw_*` 返回 `ENXIO` 当：

- DT属性中引用的GPIO控制器尚未附加。
- 该控制器的引脚号超出范围。
- DT中的phandle错误。

第一个原因是最常见的。修复是 `MODULE_DEPEND(your_driver, gpiobus, 1, 1, 1)` 以便动态加载器先带起 `gpiobus`。

### 中断处理程序不触发

如果硬件应该引发中断但什么也没发生：

- 确认DT `interrupts` 属性正确。格式取决于父中断控制器的 `#interrupt-cells`。
- 确认 `bus_alloc_resource_any(SYS_RES_IRQ, ...)` 返回了有效资源。
- 确认 `bus_setup_intr` 返回零。
- 确认你的处理程序返回值是 `FILTER_HANDLED` 或 `FILTER_STRAY` 对于过滤器，或 `FILTER_SCHEDULE_THREAD` 如果你使用线程处理程序。

用 `vmstat -i` 看你的中断是否正在被计数。如果计数保持为零，中断甚至没有被路由到你的处理程序。

### 卸载返回EBUSY

Detach忘了释放某东西。仔细走你的attach并确认每个 `_get_` 在detach中有匹配的 `_release_` 调用。常见罪魁祸首：

- 用 `gpio_pin_get_by_*` 获取的GPIO引脚。
- 用 `bus_setup_intr` 设置的中断处理程序。
- 来自 `clk_get_by_*` 的时钟句柄。
- 来自 `regulator_get_by_*` 的稳压器句柄。
- 来自 `bus_alloc_resource_any` 的内存资源。

在每次释放上打印面包屑：

```c
device_printf(dev, "detach: releasing GPIO pin\n");
gpio_pin_release(sc->sc_pin);
```

如果面包屑在预期结束前停止，未清除的资源是最后打印行之后的那一个。

### 启动期间Panic

在FDT解析期间触发的panic通常意味着blob本身畸形，或驱动程序解引用了一个没有检查的 `OF_getprop` 结果。两个保护措施：

- 总是检查 `OF_getprop`、`OF_getencprop` 和 `OF_getprop_alloc` 的返回值。缺失属性返回 `-1` 或 `ENOENT`；将缺失视为存在会导致读取栈上接下来的任何东西。
- 当属性可选时，在调用 `OF_getprop` 前用 `OF_hasprop(node, "prop")`。

### DT编译错误

`dtc` 错误消息相当清晰。一些模式需要认识：

- **`syntax error`**: 缺失分号、不平衡花括号、或错误属性值语法。
- **`Warning (simple_bus_reg)`**: `simple-bus` 下的节点有 `reg` 但无 `ranges` 转换，或其 `reg` 不匹配父节点的cell计数。
- **`FATAL ERROR: Unable to parse input tree`**: 文件在粗略层面语法损坏。检查缺失 `/dts-v1/;` 或错误引号的字符串。
- **`ERROR (phandle_references): Reference to non-existent node or label`**: 一个 `&label` 引用编译器无法解析。这是 `-@` 重要之处；没有它，依赖基础树标签的overlay无法验证。

### 内核看到错误的硬件

如果你的驱动程序附加但从寄存器读垃圾：

- 仔细检查DT中 `reg` 值。
- 确认 `#address-cells` 和 `#size-cells` 在父级与你期望的匹配。
- 只在你确定地址安全时使用 `hexdump /dev/mem` 风格技术；读错误的MMIO范围可能挂起总线。

### 你改了驱动程序但什么没变

仔细检查：

- 编辑后你运行了 `make` 吗？
- 你复制了新 `.ko` 到 `/boot/modules/`（或显式加载本地的）吗？
- 你在加载新的之前卸载了旧模块吗？`kldstat -m driver` 显示当前驻留的模块。

避免最后一个陷阱的简单习惯是总是在 `kldload` 前显式 `kldunload`，或用 `kldload -f` 强制替换。

### 快速参考：最常用的OFW调用

为方便，这里是FDT驱动程序最常用的OFW和 `ofw_bus` 辅助函数的紧凑表。每个都在 `<dev/ofw/openfirm.h>` 或 `<dev/ofw/ofw_bus_subr.h>` 中声明。

| 调用                                         | 功能                                                   |
|----------------------------------------------|---------------------------------------------------------------|
| `ofw_bus_get_node(dev)`                      | 返回此设备DT节点的phandle。                 |
| `ofw_bus_get_compat(dev)`                    | 返回节点第一个兼容字符串，或NULL。     |
| `ofw_bus_get_name(dev)`                      | 返回节点名称部分（'@'之前）。                |
| `ofw_bus_status_okay(dev)`                   | 状态缺失、"okay"或"ok"时为true。                  |
| `ofw_bus_is_compatible(dev, s)`              | 任一兼容条目匹配s时为true。             |
| `ofw_bus_search_compatible(dev, tbl)`        | 返回compat_data表中匹配条目。            |
| `ofw_bus_has_prop(dev, s)`                   | 属性存在于节点时为true。                 |
| `OF_getprop(node, name, buf, len)`           | 复制原始属性字节。                                     |
| `OF_getencprop(node, name, buf, len)`        | 复制属性，u32 cell字节交换到主机字节序。   |
| `OF_getprop_alloc(node, name, bufp)`         | 分配并返回属性；调用者用OF_prop_free释放。|
| `OF_hasprop(node, name)`                     | 属性存在时返回非零。                      |
| `OF_child(node)`                             | 第一个子节点phandle，或0。                                   |
| `OF_peer(node)`                              | 下一个兄弟节点phandle，或0。                                  |
| `OF_parent(node)`                            | 父节点phandle，或0。                                        |
| `OF_finddevice(path)`                        | 按绝对路径查找节点。                             |

### 快速参考：最常用的外设调用

来自 `<dev/gpio/gpiobusvar.h>`：

| 调用                                            | 功能                                              |
|-------------------------------------------------|-----------------------------------------------------------|
| `gpio_pin_get_by_ofw_idx(dev, node, idx, &pin)` | 按 `gpios` 索引获取引脚。                             |
| `gpio_pin_get_by_ofw_name(dev, node, n, &pin)`  | 按命名引用获取引脚（如 `led-gpios`）。       |
| `gpio_pin_get_by_ofw_property(dev, n, p, &pin)` | 从命名DT属性获取引脚。                       |
| `gpio_pin_setflags(pin, flags)`                 | 配置方向、上拉等。                  |
| `gpio_pin_set_active(pin, val)`                 | 驱动输出到活动或非活动状态。                 |
| `gpio_pin_get_active(pin, &val)`                | 读当前输入或输出电平。                       |
| `gpio_pin_release(pin)`                         | 将引脚归还到未拥有池。                           |

来自 `<dev/extres/clk/clk.h>`：

| 调用                                      | 功能                                                   |
|-------------------------------------------|---------------------------------------------------------------|
| `clk_get_by_ofw_index(dev, node, i, &c)`  | 获取 `clocks` 属性中列出的第n个时钟。               |
| `clk_get_by_ofw_name(dev, node, n, &c)`   | 按名称获取时钟。                                        |
| `clk_enable(c)` / `clk_disable(c)`        | 打开或关闭时钟门控。                                     |
| `clk_get_freq(c, &f)`                     | 读当前频率Hz。                                 |
| `clk_release(c)`                          | 释放时钟句柄。                                         |

来自 `<dev/extres/regulator/regulator.h>`：

| 调用                                                  | 功能                                         |
|-------------------------------------------------------|-----------------------------------------------------|
| `regulator_get_by_ofw_property(dev, node, p, &r)`     | 从DT属性获取稳压器。                |
| `regulator_enable(r)` / `regulator_disable(r)`        | 打开或关闭轨。                               |
| `regulator_set_voltage(r, min, max)`                  | 在min..max范围内请求电压。             |
| `regulator_release(r)`                                | 释放稳压器句柄。                          |

来自 `<dev/extres/hwreset/hwreset.h>`：

| 调用                                         | 功能                                              |
|----------------------------------------------|-----------------------------------------------------------|
| `hwreset_get_by_ofw_name(dev, node, n, &h)`  | 按名称获取复位线。                               |
| `hwreset_get_by_ofw_idx(dev, node, i, &h)`   | 按索引获取复位线。                              |
| `hwreset_assert(h)` / `hwreset_deassert(h)`  | 将外设放入或带出复位。                      |
| `hwreset_release(h)`                         | 释放复位句柄。                                     |

来自 `<dev/fdt/fdt_pinctrl.h>`：

| 调用                                          | 功能                                             |
|-----------------------------------------------|----------------------------------------------------------|
| `fdt_pinctrl_configure_by_name(dev, name)`    | 应用命名的pinctrl状态。                           |
| `fdt_pinctrl_configure_tree(dev)`             | 递归对子节点应用 `pinctrl-0`。               |
| `fdt_pinctrl_register(dev, mapper)`           | 注册新pinctrl提供者。                         |

打印这个表，钉在工作站附近，每当开始新驱动程序时参考它。这些调用在几个项目内会成为第二本能。

### 总结检查清单

在宣布驱动程序完成之前，遍历这个清单：

- [ ] 模块在 `-Wall -Wextra` 下无警告干净构建。
- [ ] `kldload` 产生预期的附加消息。
- [ ] `kldunload` 成功无EBUSY。
- [ ] 重复加载卸载循环十几次不泄露资源。
- [ ] `devinfo -r` 显示驱动程序在树中预期位置。
- [ ] sysctl存在、可读、在意处可写。
- [ ] 驱动程序关心的所有DT属性在README中文档化。
- [ ] 配套overlay源码用 `dtc -@` 编译。
- [ ] 新读者可以阅读源码并理解它做什么。

## 设备树与嵌入式术语表

本术语表收集本章使用的术语，简要定义，以便你在首次遇到时可以检查术语而无需搜索正文。相关章节的交叉引用在有帮助的地方出现在括号中。

**ACPI**: 高级配置与电源接口。PC类和一些arm64服务器使用的FDT替代方案。FreeBSD内核在启动时选择其中一个。（第3节。）

**amd64**: FreeBSD的64位x86架构。通常使用ACPI而不是FDT，尽管FDT可以在专门的嵌入式x86案例中使用。

**arm64**: FreeBSD的64位ARM架构。在嵌入式板上默认使用FDT；在SBSA兼容服务器上使用ACPI。

**绑定（Bindings）**: 外设属性如何在设备树中拼写的文档化约定。例如，`gpio-leds` 绑定文档化LED控制器的DT节点应该携带什么属性。

**Blob**: `.dtb` 文件的非正式术语，因为从FDT解析器以外的任何东西看来它是一个不透明的二进制块。

**BSP**: 板支持包。在特定板上运行操作系统所需的文件集合（内核配置、设备树、加载程序提示，有时是驱动程序）。

**Cell**: 32位大端值，是DT属性的原子单位。属性值是cell序列。

**兼容字符串**: 驱动程序probe匹配的标识符，存储在DT节点的 `compatible` 属性中。通常有供应商前缀斜杠型号形式：`"brcm,bcm2711-gpio"`。

**兼容数据表**: 驱动程序通过 `ofw_bus_search_compatible` 在probe中遍历的 `struct ofw_compat_data` 条目数组。（第4节。）

**dtb**: 编译的设备树二进制。`dtc` 的输出；内核在启动时解析的格式。

**dtbo**: 编译的设备树overlay。一个小型二进制，加载程序在交给内核前合并到主 `.dtb` 中。

**dtc**: 设备树编译器。将源码翻译为二进制。

**dts**: 设备树源码。`dtc` 的文本输入。

**dtsi**: 设备树源码包含。一个旨在被 `#include` 的源码片段。

**边沿触发（Edge triggered）**: 在电平转换（上升、下降或两者）时触发的中断。与电平触发对比。

**FDT**: 扁平设备树。FreeBSD用于基于DT的系统的二进制格式和框架。也俗用于指整个概念。

**fdt_overlays**: 启动时列出要应用的overlay名称的loader.conf tunable。

**fdtdump**: 将 `.dtb` 文件解码为其源码可读近似的工具。

**片段（Fragment）**: overlay中命名目标并声明要合并内容的顶级条目。（第5节。）

**GPIO**: 通用输入/输出。可以驱动或读取线路的可编程数字引脚。

**intrng**: FreeBSD的中断下一代框架。统一中断控制器和消费者。（第10节。）

**kldload**: 将内核模块加载到运行系统的命令。

**kldunload**: 卸载已加载内核模块的命令。

**电平触发（Level triggered）**: 只要条件为真就断言的中断。必须在源端清除以停止触发。

**加载程序（Loader）**: FreeBSD的引导加载程序。读取配置、加载模块、合并DT overlay、将控制权交给内核。

**MMIO**: 内存映射IO。通过物理地址范围暴露的硬件寄存器集。

**Newbus**: FreeBSD的设备驱动框架。每个驱动程序注册到Newbus父子关系的树中。

**节点（Node）**: 设备树中的一个点。有名称（和可选单元地址）和一组属性。

**OFW**: Open Firmware。FreeBSD的FDT代码重用其API的历史标准。

**ofwbus**: FreeBSD用于Open Firmware派生设备枚举的顶级总线。

**ofwdump**: 从运行内核的DT打印节点和属性的用户空间工具。（第6节。）

**Overlay**: 修改现有树的部分 `.dtb`。通过标签或路径针对节点并在其下合并内容。（第5节。）

**phandle**: phantom handle。DT节点的32位整数标识符，用于在树内交叉引用节点。

**pinctrl**: 引脚控制框架。处理SoC引脚在其可能功能之间的复用。（第8节。）

**PNP info**: 驱动程序发布以标识其支持的DT兼容字符串的元数据。用于 `devmatch(8)` 自动加载。（第9节。）

**Probe**: 检查候选设备并报告是否可以驱动它的驱动程序方法。返回强度分数或错误。

**属性（Property）**: DT节点中的命名值。值可以是字符串、cell列表或不透明字节串。

**Reg**: 列出一个或多个描述外设占用的MMIO范围的（地址、大小）对的属性。

**根控制器（Root controller）**: 最顶层的中断控制器。在arm64系统上，通常是GIC。

**SBC**: 单板计算机。一块PCB上有CPU、内存和外设的嵌入式板。示例：Raspberry Pi、BeagleBone。

**SIMPLEBUS_PNP_INFO**: 将驱动程序的兼容表导出为模块元数据的宏。（第9节。）

**Simplebus**: 探测父节点 `compatible = "simple-bus"` 的DT子节点的FreeBSD驱动程序。它将DT节点转换为Newbus设备。

**softc**: "soft context"的简称。Newbus分配并传递给驱动程序方法的每设备状态结构。（第4节。）

**SoC**: 片上系统。包含CPU、内存控制器和许多外设块的集成电路。

**Status**: DT节点上指示设备是否启用（`"okay"`）的属性。缺失状态默认为okay。

**sysctl**: FreeBSD的系统控制接口。驱动程序可以发布用户空间读写的可调参数。

**目标（Target）**: overlay中片段修改的基础树中的节点。

**单元地址（Unit address）**: 节点名称中 `@` 后的数字部分。指示节点在父地址空间中存在的位置。

**供应商前缀（Vendor prefix）**: 兼容字符串中逗号前的部分。标识负责绑定的组织。

## 常见问题

这些是人们首次为FreeBSD编写FDT驱动程序时反复出现的问题。大多数在某处已经回答过；FAQ格式只是把简短答案放在一个地方。

**编写FDT驱动程序需要知道ARM汇编吗？**

不需要。驱动框架的整个意义是你在统一的API下用C工作。如果你在调试非常底层的崩溃，你可能会读反汇编，但那是例外，不是规则。

**我可以在amd64上编写FDT驱动程序，还是需要arm64硬件？**

你可以在amd64上开发并为arm64交叉编译。你也可以在amd64主机上用QEMU运行arm64 FreeBSD，这是任何不想等待Pi重启的人最常用的工作流程。对于最终验证，你最终需要真实硬件或忠实模拟器，但日常迭代适合笔记本电脑。

**simplebus和ofwbus有什么区别？**

`ofwbus` 是Open Firmware派生设备枚举的顶级根总线。`simplebus` 是覆盖 `compatible` 为 `"simple-bus"` 和类似简单枚举的DT节点的通用总线驱动程序。你的大多数驱动程序会注册到两者；`ofwbus` 处理根节点和特殊情况，`simplebus` 处理绝大多数外设总线。

**为什么我的驱动程序需要同时注册到ofwbus和simplebus？**

一些节点出现在 `simplebus` 下，一些直接出现在 `ofwbus` 下（特别是在树结构不寻常的系统上）。注册到两者意味着驱动程序在节点恰好落脚的地方都会附加。

**为什么我的overlay没有应用？**

遍历第6节的检查清单。最常见原因按顺序：错误的文件名或目录、`fdt_overlays` 中拼写错误、基础兼容不匹配、编译overlay时缺失 `-@`。

**驱动程序可以跨越多个DT节点吗？**

可以。单个驱动程序实例通常匹配一个节点，但attach函数可以遍历子节点或phandle引用来从多个节点收集状态。见第9节对 `gpioled_fdt.c` 的讨论关于子节点情况。

**我如何处理同时有ACPI和FDT描述的设备？**

编写两个兼容路径。大多数支持两个平台的大型FreeBSD驱动程序正是这样做：单独的probe函数向每个总线注册，共享代码在公共attach中。看 `sdhci_acpi.c` 和 `sdhci_fdt.c` 作为工作示例。

**FreeBSD树中找不到的DT绑定怎么办？**

DT绑定的真相来源是上游设备树规范加上Linux内核文档。FreeBSD在实用处使用相同绑定。如果你需要FreeBSD尚不支持的绑定，通常可以移植相关Linux驱动程序或编写消费相同绑定的原生FreeBSD驱动程序。

**我需要修改内核来添加新驱动程序吗？**

不需要。树外模块针对 `/usr/src/sys` 头文件编译并运行时加载。只有在你向上游贡献驱动程序或需要更改通用基础设施时才编辑内核树本身。

**我如何从amd64主机交叉编译arm64？**

使用FreeBSD构建系统中包含的交叉工具链：

```console
$ make TARGET=arm64 TARGET_ARCH=aarch64 buildworld buildkernel
```

这会在你的amd64工作站上构建完整的arm64系统映像。仅模块构建用更窄的目标遵循相同模式。

**有办法在匹配DT节点出现时自动加载我的驱动程序吗？**

有，通过 `devmatch(8)` 和 `SIMPLEBUS_PNP_INFO` 宏。声明你的兼容表，在驱动程序源码中包含 `SIMPLEBUS_PNP_INFO(compat_data)`，`devmatch` 会拾取它。

**我可以用C++编写FDT驱动程序吗？**

不可以。FreeBSD内核代码严格是C（和极少量汇编）。其他语言不支持，内核API假定C约定。

**我如何调试启动时DT解析期间的崩溃？**

早期崩溃很难。常用技术：启用详细启动消息（`loader.conf` 中 `-v`）、用 `KDB` 和 `DDB` 编译内核、使用串行控制台，如果有JTAG调试器就连接。也考虑直接在 `/usr/src/sys/dev/ofw/` 下的FDT解析代码中添加临时 `printf` 语句。

**共享FDT绑定的驱动程序模块会互相干扰吗？**

不会。每个模块注册其兼容表，匹配驱动程序的probe强度决定哪个获胜。如果两个驱动程序以相同强度声称相同兼容字符串，它们被加载的顺序决定获胜者。给每个驱动程序独特的强度以避免意外。

**我如何在kldload/kldunload循环间保留状态？**

你不能。卸载的模块丢失所有状态。如果你的驱动程序需要持久性，写入文件、sysctl的可调形式、内核tunable，或比模块寿命长的NVRAM或EEPROM中的位置。对于调试，卸载前将状态打印到 `dmesg` 并在下次加载后解析回来是一个可行的快捷方式。

**`compatible` 列表顺序重要吗？**

重要。节点可以列出多个兼容字符串，从最特定到最不特定。`compatible = "brcm,bcm2711-gpio", "brcm,bcm2835-gpio";` 声明节点主要是2711变体但作为回退与较旧的2835绑定兼容。声称2835兼容的驱动程序如果没有加载2711驱动程序将匹配此节点。顺序让固件以多个细节级别描述设备，以便新内核可以利用改进而不断开旧版。

**为什么FreeBSD有时使用与Linux不同的属性名称？**

大多数DT属性是共享的，但在FreeBSD行为与Linux期望不同的地方少数有意不同。当你移植驱动程序时，仔细阅读FreeBSD侧现有绑定；静默假定Linux语义是移植bug的常见来源。

**Newbus和intrng之间是什么关系？**

Newbus处理设备探测、附加和资源分配。intrng处理中断控制器注册和IRQ路由。它们互操作：Newbus为 `SYS_RES_IRQ` 的资源分配通过intrng找到正确的控制器，intrng将中断派发回Newbus注册的驱动程序处理程序。

## 总结

本章带你从"FreeBSD是一个通用操作系统"到"我可以编写一个适合嵌入式系统设备树的驱动程序"。这两个任务不一样。第一个是使用你已经知道的内核；第二个是理解一个全新的词汇来描述内核运行的硬件。

我们首先看了嵌入式FreeBSD实际上是什么：一个运行在SBC、工业板和定制硬件上的精简、有能力的系统。我们看到这些系统如何不通过PCI枚举或ACPI表，而是通过固件在启动时交给内核的静态设备树来描述自己。

然后我们学习了设备树语言本身：带名称和单元地址的节点、带cell结构值的属性、交叉引用的phandle，以及让我们无需重建基础blob就能添加或修改节点的 `/plugin/;` overlay语法。语言一个下午就能习惯；它建立的关于将硬件视为有父节点的、有地址的、有类型的树的习惯，会持续职业生涯。

有了语言之后，我们看了FreeBSD消费DT的机械。`fdt(4)` 子系统加载blob。OFW API遍历它。`ofw_bus_*` 辅助函数以兼容字符串和状态检查的形式暴露遍历。`simplebus(4)` 驱动程序枚举子节点。消费者框架（`clk`、`regulator`、`hwreset`、`pinctrl`、GPIO）都集成了DT phandle引用，使驱动程序可以通过统一模式获取资源。

武装了机械后，我们构建了一个驱动程序。首先是 `fdthello`，最小骨架，以最纯粹形式展示了FDT驱动程序的要求形状。然后是 `edled`，一个完整的GPIO驱动LED驱动程序，演示了attach、detach、sysctl和适当的资源管理。沿途我们看了如何编译overlay、通过 `loader.conf` 部署它们、用 `ofwdump` 运行时检查树、以及调试嵌入式驱动程序独特表现的那类失败。

最后，我们让 `edled` 经历了将工作驱动程序变成完成驱动程序的重构过程：收紧的错误路径、真正的互斥锁、可选的电源和时钟处理、pinctrl意识、风格审计。这是区分玩具和你愿意在生产中运行的驱动程序的工作。

本章的实验给你机会自己运行一切。挑战练习给你扩展的空间。故障排除部分给你东西坏掉时的去处。

### 你现在应该能够做到的事情

如果你完成了实验并仔细阅读了解释，你现在可以：

- 阅读陌生的 `.dts` 并解释它描述的硬件。
- 编写向现有板子添加外设的新 `.dts` 或 `.dtso`。
- 将该源码编译成二进制并部署到运行的系统。
- 从头编写FDT感知FreeBSD驱动程序。
- 通过标准消费者API获取内存、IRQ、GPIO、时钟和稳压器资源。
- 调试探测未触发、附加失败和detach卡住类问题。
- 识别板子在arm64上运行FDT模式还是ACPI模式，以及这对你的驱动程序意味着什么。

### 本章隐含的内容

有三个主题本章留给其他来源。它们不是书形的；它们是参考形的，如果试图完全覆盖会使本书陷于停顿。

第一，完整DT绑定规范。我们覆盖了你最可能使用的属性。由设备树社区维护的完整绑定目录可在Linux内核的 `Documentation/devicetree/bindings/` 树的在线文档浏览。FreeBSD遵循这些绑定的大多数，例外在 `/usr/src/sys/contrib/device-tree/Bindings/` 中针对FreeBSD依赖的子集注明。

第二，复杂SoC上的中断路由。我们触及了 `interrupt-parent` 链。具有多个GIC风格控制器、pinctrl管理中断和嵌套gpio作为中断节点的板子可能变得复杂。当简单情况不再够用时，FreeBSD `intrng` 子系统是要去的地方。

第三，不通过simplebus的外设的FDT支持：USB phy、具有分层PLL的SoC上的时钟树、DVFS的电压缩放。每个都是自己的子主题。本书附录指向权威来源。

### 关键要点

如果本章可以浓缩成单页，以下是值得记住的点：

1. **嵌入式系统通过设备树描述自己，而不是通过运行时枚举。** 固件交给内核的blob是存在什么硬件的权威描述。

2. **驱动程序通过兼容字符串匹配。** 你的probe查阅兼容表，与节点的 `compatible` 属性比较。准确获得这个字符串。

3. **`simplebus(4)` 是枚举器。** 每个FDT驱动程序的父节点要么是 `simplebus` 要么是 `ofwbus`。在模块加载时向两者注册。

4. **资源来自框架。** `bus_alloc_resource_any` 用于MMIO和IRQ，`gpio_pin_get_by_ofw_*` 用于GPIO，以及匹配的 `clk_*`、`regulator_*`、`hwreset_*` 调用用于电源和复位。在detach中释放每一个。

5. **Overlay无需重建就修改树。** 放在 `/boot/dtb/overlays/` 下并列在 `fdt_overlays` 中的小型 `.dtbo` 是在给定板上添加或启用外设的干净方式。

6. **中断接线遵循父链。** 节点的 `interrupt-parent` 和 `interrupts` 属性通过intrng连接到根控制器。理解链对于调试静默中断至关重要。

7. **真实驱动程序是最好的参考。** `/usr/src/sys/dev/gpio/`、`/usr/src/sys/dev/fdt/` 和每个SoC的平台树包含数十个FDT驱动程序，演示你可能需要的每种模式。

8. **错误路径和拆除很重要。** 加载并工作一次的驱动程序是简单情况。干净加载、卸载并重新加载一百次的驱动程序是健壮情况。

9. **工具简单但有效。** `dtc`、`ofwdump`、`fdtdump`、`devinfo -r`、`sysctl dev.<name>.<unit>` 和 `kldstat` 一起覆盖你需要的几乎每次检查。

10. **FDT只是几种硬件描述系统之一。** 在arm64服务器上，ACPI扮演相同角色。你在这里学到的驱动程序模式可转移，但匹配层改变。

### 继续之前

在将本章视为完成并转到第33章之前，花点时间验证以下内容。这些是将见过材料的学习者与拥有材料的学习者区分开的检查类型。

- 你可以不看地勾画出FDT感知驱动程序的骨架，包括兼容表、probe、attach、detach、方法表和 `DRIVER_MODULE` 注册。
- 你可以叙述一段DT源码，逐节点、逐属性地描述它。
- 你可以区分phandle和路径引用，并解释何时使用每个。
- 你可以解释为什么 `MODULE_DEPEND(your_driver, gpiobus, 1, 1, 1)` 重要以及没有它会出什么问题。
- 你知道哪个加载程序tunable控制overlay加载以及 `.dtbo` 文件在哪里。
- 给定一个未触发的probe，你在求助调试器之前有一个四五个东西需要验证的心理检查清单。
- 你知道 `SIMPLEBUS_PNP_INFO` 做什么以及为什么它对生产驱动程序重要。
- 你可以命名至少三个 `/usr/src/sys` 中的真实FDT驱动程序，并各用一句话描述它们演示了什么。

如果这些仍有任何感觉摇摇欲坠，回到相关章节并重读。本章的材料会累积；第33章将假定你已经内化了这里的大部分内容。

### 关于持续练习的说明

嵌入式FreeBSD回报定期练习。第一次为真实板子读 `.dts` 时，它看起来令人不知所措。第十次，你浏览你关心的节点。第一百次，你原地编辑。如果你桌上有空闲的SBC，让它工作起来。选一个传感器，写一个驱动程序，添加一行到loader.conf。技能会累积。你完成的每个小型驱动程序都是下一个的模板，模式跨板子和供应商携带。

### 连接回本书其余部分

第32章位于第7部分后期，这里的材料依赖于你早期遇到的层。简要浏览你现在以新视角看到的东西：

从第1和第2部分，内核模块骨架、`DRIVER_MODULE` 注册和 `kldload`/`kldunload` 生命周期。FDT驱动程序的形状是相同的形状，带有不同的探测策略。

从第3部分，Newbus框架。`simplebus` 是一个总线驱动程序，恰好从设备树而不是总线协议获取其子节点。你早期学到的每个Newbus模式在此上下文中适用，不变。

从第4部分，驱动程序到用户空间接口：`cdev`、`sysctl`、`ioctl`。我们的 `edled` 驱动程序使用sysctl作为其控制接口；在更大项目中它可能添加字符设备甚至netlink socket。

从第5和第6部分，关于并发、测试和内核调试的实用章节。这些工具全部适用于嵌入式驱动程序。唯一的区别是硬件通常更难到达，所以工具更重要。

从第7部分，关于32位平台的第29章和关于虚拟化的第30章都触及嵌入式角度。第31章的安全章节直接适用：嵌入式设备上的驱动程序通常暴露于产品用户空间运行的任何东西，相同的防御模式适用。

第32章之后的情况是，你拥有大多数类FreeBSD设备工作的完整入门工具包。你可以编写字符驱动程序、总线驱动程序、网络驱动程序，以及现在基于FDT的嵌入式驱动程序。第7部分的剩余章节完善和终结这些技能。

### 展望下一章

下一章，第33章，从"它工作吗"转向"它工作得怎么样"。性能调优和剖析是测量内核在真实负载下用真实数据实际做什么的艺术。在嵌入式系统上，这个问题有额外的力量。硬件通常很小，工作负载通常固定，"运行良好"和"运行不良"之间的差距以微秒测量。在第33章中，我们看FreeBSD给你的测量工具：`hwpmc`、`pmcstat`、DTrace、火焰图和内核自己的性能计数器探针。我们还看解释工具说什么所需的心理模型，以及调优错误东西的陷阱。

在第33章之后，第34到38章完成精通弧线：编码风格和可读性、为FreeBSD贡献、在平台间移植驱动程序、文档化你的工作，以及将几条早期线索汇聚在一起的毕业设计项目。每个都建立在你在这里做的事情上。你本章编写的 `edled` 驱动程序，通过挑战练习扩展，是一个合理的候选者，可以作为运行示例带入那些最终章节。

驱动程序工作完成了。测量工作开始。

### 临别赠言

在结束之前最后一个鼓励。嵌入式FreeBSD有一个安静、稳定的社区。板子便宜，源码开放，邮件列表耐心。如果你坚持下去，你会发现你编写的每个驱动程序都教给下一个需要的东西。本章的 `edled` 驱动程序是一件小事，但在编写它时你触摸了FDT解析器、OFW API、GPIO消费者框架、Newbus树和sysctl机制。那不是什么都没有。它是树中每个FDT驱动程序的骨干，以缩影方式演练。

保持练习。领域用稳步进步回报好奇心。下一章给你工具确保那个进步是朝正确方向的进步。

当你最终坐在陌生的板子前，视野将不再感觉陌生。你会知道请求它的 `.dts`，浏览 `soc` 下列出的外设，检查哪个GPIO控制器承载你关心的引脚，查找你需要匹配的兼容字符串。你刚刚建立的习惯与你同行。它们适用于Raspberry Pi，适用于BeagleBone，适用于定制工业板，适用于QEMU virt机器。它们适用于十年后尚不存在的板子，因为硬件描述的底层语言是稳定的。

那种可移植性，归根结底，是设备树设计的，也是本章设计要教的。欢迎来到嵌入式FreeBSD。

有了第32章，你的工具包基本完成。进入第33章，进入将告诉你刚刚编写的驱动程序是否如所需那样高效的测量工作。

阅读愉快，驱动程序编写愉快。
