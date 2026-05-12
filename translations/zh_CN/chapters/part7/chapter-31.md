---
title: "安全最佳实践"
description: "在 FreeBSD 设备驱动程序中实施安全措施"
partNumber: 7
partName: "精通主题：特殊场景与边缘情况"
chapter: 31
lastUpdated: "2026-04-19"
status: "complete"
author: "Edson Brandi"
reviewer: "TBD"
translator: "AI辅助翻译为简体中文"
estimatedReadTime: 240
language: "zh-CN"
---

# 安全最佳实践

## 引言

当你到达第31章时，你已经建立了一种对环境的理解，这是很少有作者要求你构建的。第29章教你编写能够在总线、架构和字长变化中存活下来的驱动程序。第30章教你编写在底层机器不是真正的机器而是虚拟机时、在使用它的进程不在主机上而是在 jail 中时能够正确运行的驱动程序。这两章都在讨论边界：硬件与内核之间的边界、主机与客户机之间的边界、主机与容器之间的边界。第31章要求你审视一个不同的边界，一个比上述任何一个都更贴近自身的边界，一个正因为贯穿你自己的代码中间而更容易被遗忘的边界。

本章讨论的边界是内核与所有与之通信的对象之间的边界。用户态程序、硬件本身、内核的其他部分、传递参数的引导加载程序、上周从供应商支持站点下载的固件二进制文件、升级后开始表现异常的设备。它们每一个都位于你所编写的驱动程序的信任边缘的另一侧。它们每一个都可以——有意或无意地——向驱动程序传递不符合其期望的内容。尊重这一边界的驱动程序是可以被信任来保护内核的驱动程序。不尊重这一边界的驱动程序，在敌对内容到来的那一天，会让敌意到达它本不应到达的代码。

本章的目标是培养那种尊重的习惯。它不是一本安全教科书，也不会试图将你变成漏洞研究人员。它要做的是教你像攻击者或粗心的程序那样看待你的驱动程序，识别将小漏洞转变为完整系统入侵的特定错误类别，并在你想要防止这些错误发生时找到正确的 FreeBSD 原语。

关于这在实践中意味着什么，说几句话。内核安全漏洞不仅仅是用户空间安全漏洞的更糟版本。用户空间程序中的缓冲区溢出可能损坏该程序的内存；驱动程序中的缓冲区溢出可能损坏内核，而内核为系统上的每个程序提供服务。用户空间解析器中的差一错误可能导致解析器崩溃；驱动程序中相同的差一错误可能为攻击者提供读取保存其他用户秘密的内核内存的方法，或者向内核的函数表写入任意字节的方法。后果不会随漏洞的大小线性扩展。它们随漏洞所在代码的权限级别而扩展，而内核位于该层次结构的顶端。

这就是为什么驱动程序安全不是一门独立的学科，不是附加在你一直在构建的编程技能之上的。它就是那些编程技能，只是在应用时考虑了特定的纪律。你在前面章节中看到的 `copyin()` 和 `copyout()` 调用变成了强制执行信任边界的工具。你学到的 `malloc()` 标志变成了控制内存生命周期的方式。你练习的加锁纪律变成了防止攻击者可能操纵的竞态的方法。你在第30章中简要看到的权限检查变成了防御非特权调用者进入他们不应到达的地方的第一道防线。从真正的意义上说，安全就是编写内核代码的纪律，以更高的标准来要求，并通过希望它失败的人的眼睛来审视。

本章分十步构建这一视图。第1节激发主题并解释驱动程序的安全模型为何不同于应用程序。第2节深入讲解内核代码中缓冲区溢出和内存损坏的机制，这是一个容易误解的话题，因为你在第4章学到的普通 C 语言在内核中有着更加危险的陷阱。第3节涵盖用户输入——驱动程序中可利用漏洞的最大来源——并讲解 `copyin(9)`、`copyout(9)` 和 `copyinstr(9)` 的安全使用方法。第4节转向内存分配和生命周期，包括对安全性至关重要的 FreeBSD 特定 `malloc(9)` 标志。第5节检查竞态条件和检查时间到使用时间（TOCTOU）漏洞，这是位于并发和安全交叉点的一类问题。第6节涵盖访问控制：驱动程序如何使用 `priv_check(9)`、`ucred(9)`、`jailed(9)` 和安全级别机制来检查调用者是否被允许执行其请求的操作。第7节讨论信息泄露，即驱动程序无意中泄露数据的微妙漏洞类别。第8节讨论日志记录和调试，如果对打印内容不够谨慎，这些本身也可能成为安全问题。第9节提炼了一组围绕安全默认值和故障安全行为的设计原则。第10节通过测试、加固以及对 FreeBSD 提供的用于在他人之前发现这些漏洞的工具的实际介绍来结束本章：`INVARIANTS`、`WITNESS`、内核净化器、`Capsicum(4)` 以及使用 syzkaller 进行模糊测试。

除了这十个节之外，本章还将涉及 `mac(4)` 框架、Capsicum 在约束 ioctl 调用者方面的作用、诸如 `copyinstr(9)` 之类的字符串安全惯用法，以及现代内核加固功能（如 ASLR、SSP 和可加载模块的 PIE）带来的构建陷阱。这些内容在最为相关的地方出现，绝不会以牺牲核心线索为代价。

开始之前最后一件事。编写安全的内核代码不是阅读清单然后勾选项目的事情。它是一种阅读自己代码的方式，以及一套随时间推移变得自动化的反射。第一次看到驱动程序在不检查长度的情况下调用 `copyin` 到固定大小的栈缓冲区时，你可能会想有什么问题；第一百次看到这种模式时，你会感到后背发凉。本章的目标不是教你每种漏洞的每种变体；那需要一架子书。目标是帮助你建立这些反射。一旦你拥有它们，你将默认编写更安全的驱动程序，你将在别人的代码造成问题之前发现危险的模式。

让我们开始吧。

## 读者指南：如何使用本章

第31章在概念性方面与前面几章有所不同。代码示例简短而聚焦；其价值在于它们所教授的思维方式。你可以通过仔细阅读来完成整章，即使你不输入一行代码，你也会成为更好的驱动程序作者。最后的实验将思考转化为肌肉记忆，挑战将思考推到真实漏洞存在的不舒服角落，但文本本身是本章的主要教学表面。

如果你选择**仅阅读路径**，计划大约三到四个集中小时。最后你将能够识别驱动程序安全漏洞的主要类别，解释为什么内核级漏洞会改变整个系统的信任模型，描述防御各类漏洞的 FreeBSD 原语，并勾勒出不安全模式的安全版本应该是什么样子。这是一个实质性的知识体系，对于许多读者来说，这是第一次阅读时本章应该结束的地方。

如果你选择**阅读加实验路径**，计划八到十二小时，分两到三次完成。实验基于一个名为 `secdev` 的微型教学驱动程序，你将在本章过程中编写它。每个实验都是一个简短、聚焦的练习：在一个实验中，你将修复一个故意不安全的 `ioctl` 处理程序；在另一个实验中，你将添加 `priv_check(9)` 并观察非特权和 jailed 进程尝试使用受限入口点时会发生什么；在另一个实验中，你将引入竞态条件，观察它在 `WITNESS` 下的表现，然后修复它；在最后一个实验中，你将运行一个简单的模糊测试器对驱动程序的 `ioctl` 接口进行测试，并阅读产生的崩溃报告。每个实验都留给你一个可工作的系统和实验日志中的一条记录；它们都不够长到耗尽一个晚上。

如果你选择**阅读加实验加挑战路径**，计划一两个长周末。挑战将 `secdev` 推入真实领域：你将添加 MAC 策略钩子，使站点本地策略可以覆盖驱动程序的默认值；你将为驱动程序的 ioctl 标记能力权限，使受 Capsicum 限制的进程仍然可以使用安全子集；你将为驱动程序的入口点编写一个简短的 syzkaller 描述文件；你将运行净化内核变体（`KASAN`、`KMSAN`）来查看它们能捕获正常 `INVARIANTS` 构建所遗漏的内容。每个挑战都是独立的；完成它们不需要阅读额外的章节。

关于实验环境的说明。你将继续使用前面章节中的 FreeBSD 14.3 临时机器。本章的实验不需要第二台机器，不需要 `bhyve(8)`，也不需要以重启后仍然存在的方式修改主机。你将加载和卸载内核模块，写入测试字符设备，仔细阅读 `dmesg`，编辑一小棵源代码文件树。如果出了问题，重启可以恢复主机。创建快照或引导环境仍然是个好主意，而且创建成本很低。

对本章的一条特别建议：**慢慢阅读**。安全的文字有时看似温和。这些想法在纸面上看起来很明显，但某个漏洞之所以是漏洞的原因可能需要一分钟的思考才能豁然开朗。抵制略读的诱惑。如果一段话描述了你未完全理解的竞态条件，停下来重读它。如果代码片段演示了信息泄露，在脑海中追踪泄露字节的路径，直到你能说出哪个字节来自哪里。仔细阅读的回报是一套将超越本书的反射。

### 先决条件

你应该对前面章节的所有内容感到熟悉。特别是，本章假设你已经理解了如何编写可加载的内核模块，驱动程序的 `open`、`read`、`write` 和 `ioctl` 入口点如何连接到 `/dev/` 节点，softc 如何分配并附加到 `device_t`，互斥锁和引用计数如何在第14章和第21章所教授的级别上工作，以及中断和定时回调如何与睡眠路径交互。如果其中任何部分不牢固，在开始之前简要复习将使示例更容易理解。

你还应该熟悉普通的 FreeBSD 系统管理：阅读 `dmesg`、加载和卸载模块、以非特权用户身份运行命令、创建简单的 jail，以及使用 `sysctl(8)` 观察和调整系统。本章将引用这些工具，但不会从头开始逐步讲解每一个。

不需要先前的安全研究背景。本章从零开始构建其词汇。

### 本章不涵盖的内容

一个负责任的章节会告诉你它遗漏了什么。本章不教授漏洞利用开发。它不教你如何编写 shellcode，如何构建 ROP 链，或如何将崩溃转变为代码执行。这些是合理的主题，但它们属于另一种书，它们所需的技能不是帮助你编写更安全驱动程序的技能。

本章不会把你变成安全审计员。为每个漏洞类别审计大型代码库是一门独立的学科，有自己的工具和节奏。本章确实给你的是胜任地审计你自己的驱动程序的能力，以及识别值得在别人的代码中标记的模式。

本章不能替代 FreeBSD 安全公告、CERT C 编码标准、SEI 的安全内核编程指南，或它讨论的 API 的手册页。它指向这些来源，并期望你在具体问题超出本章范围时查阅它们。本章的每个主要节末尾都有一个简短的手册页指针，以便你在本章之后的第一个查阅点是 FreeBSD 文档本身。

最后，本章不试图涵盖每一种可能的漏洞类别。它专注于对驱动程序最重要的类别，以及可以使用读者已经知道或可以在几页内学会的 FreeBSD 原语来解决的类别。一些罕见的漏洞类别，如 Spectre 风格的推测执行侧信道，只在文中略提；它们属于大多数驱动程序作者不需要也不应该从头编写的专门加固工作。

### 结构与节奏

第1节构建思维模型：当驱动程序出错时有什么风险，以及驱动程序的安全模型与应用程序有何不同。第2节讨论内核代码中的缓冲区溢出和内存损坏，包括它们与用户空间同类漏洞的微妙区别。第3节教授通过 `copyin(9)`、`copyout(9)`、`copyinstr(9)` 和相关原语安全处理用户提供的输入。第4节涵盖内存分配和生命周期：`malloc(9)` 上的标志、`free(9)` 和 `zfree(9)` 之间的区别，以及内核模块容易陷入的释放后使用模式。第5节转向竞态和 TOCTOU 漏洞，包括它们的安全相关表现方式。第6节涵盖访问控制和权限强制执行，从 `priv_check(9)` 到 `ucred(9)` 和 jail 再到安全级别机制。第7节讨论信息泄露和数据逃逸的惊人微妙方式。第8节讨论日志记录和调试，这些本身也可能成为安全问题。第9节收集安全默认值和故障安全设计的原则。第10节涵盖测试和加固：`INVARIANTS`、`WITNESS`、`KASAN`、`KMSAN`、`KCOV`、`Capsicum(4)`、`mac(4)` 框架，以及运行 syzkaller 对驱动程序 ioctl 接口进行模糊测试的演示。实验和挑战紧随其后，最后是通往第32章的桥梁。

按顺序阅读各节。每一节都假设你已经阅读了前一节，最后两节（第9节和第10节）将前面的内容综合为实用建议和工作流程。

### 逐节学习

本章中的每一节都涵盖一个核心思想。不要试图同时在脑中保持两节的内容。如果一节结束而你对其某个要点感到不确定，在开始下一节之前暂停，重读结尾段落，查阅引用的手册页。五分钟的巩固几乎总是比在两节之后发现基础不够牢固更快。

### 保持参考驱动程序在手边

本章在实验中构建一个名为 `secdev` 的教学驱动程序。你可以在 `examples/part-07/ch31-security/` 下找到它，以及起始代码、故意损坏的版本和修复后的变体。每个实验目录包含该步骤的驱动程序状态，以及其 `Makefile`、简短的 `README.md` 和任何支持脚本。克隆目录，边看边输入，每次更改后加载每个版本。在实验机器上运行不安全的代码并观察发生什么是课程的一部分；不要跳过实时测试。

### 打开 FreeBSD 源代码树

有几节指向真实的 FreeBSD 文件。本章中值得仔细阅读的有 `/usr/src/sys/sys/systm.h`（用于 `copyin`、`copyout`、`copyinstr`、`bzero` 和 `explicit_bzero` 的精确签名）、`/usr/src/sys/sys/priv.h`（用于 priv 常量和 `priv_check` 原型）、`/usr/src/sys/sys/ucred.h`（用于凭据结构）、`/usr/src/sys/sys/jail.h`（用于 `jailed()` 宏和 `prison` 结构）、`/usr/src/sys/sys/malloc.h`（用于分配标志）、`/usr/src/sys/sys/sbuf.h`（用于安全字符串构建器）、`/usr/src/sys/sys/capsicum.h`（用于能力权限）、`/usr/src/sys/sys/sysctl.h`（用于 `CTLFLAG_SECURE`、`CTLFLAG_PRISON`、`CTLFLAG_CAPRD` 和 `CTLFLAG_CAPWR` 标志），以及 `/usr/src/sys/kern/kern_priv.c`（用于 priv-check 实现）。当本章指向它们时，打开它们。源代码是权威；本书是通向它的指南。

### 保持实验日志

继续前面章节的实验日志。对于本章，为每个实验记录一个简短的笔记：你运行了哪些命令，哪些模块被加载了，`dmesg` 说了什么，什么让你惊讶。安全工作比大多数工作更受益于书面记录，因为它教你看到的漏洞往往是不可见的，直到你以正确的方式寻找它们，而上周的日志条目可能会在本周为你节省一小时的重新发现。

### 调整节奏

本章中的几个思想在你第二次遇到时比第一次更加深入人心。virtio 中的特性位在第30章中经过一天休息后更有意义了；同样的事情在这里也会发生，比如说 `copyin` 错误处理和 TOCTOU 安全的重新复制之间的区别。如果某个小节第一次阅读时模糊不清，标记它，继续前进，然后再回来。安全阅读需要耐心。

## 如何从本章获得最大收益

第31章奖励一种特定的参与方式。它引入的特定原语——`priv_check(9)`、`copyin(9)`、`sbuf(9)`、`zfree(9)`、`ppsratecheck(9)`——不是装饰性的；它们是安全驱动程序代码的基石。在阅读本章时你能建立的最有价值的习惯是在每个调用点问两个问题：这些数据从何而来，谁被允许导致它出现在这里？

### 以敌意的心态阅读

安全阅读要求你改变看待代码的方式。当本章向你展示一个从用户空间复制 `len` 字节到缓冲区的驱动程序时，不要像 `len` 的值是合理的那样阅读代码片段。把它当作 `len` 是 0xFFFFFFFF 来阅读。把它当作 `len` 是一个精心选择的、通过了一个明显检查但未通过更微妙检查的值来阅读。像一个无聊、聪明、不友好的人在睡前那样阅读代码。那种阅读才能发现漏洞。

### 运行你阅读的内容

当本章引入一个原语时，运行一个小例子。当它展示 `priv_check` 的模式时，编写一个两行的内核模块，用特定常量调用 `priv_check`，观察从非 root 进程调用其 ioctl 时会发生什么。当它描述 `CTLFLAG_SECURE` 对 sysctl 的影响时，在实验模块中设置一个虚拟 sysctl，升高和降低安全级别，观察行为变化。运行的系统能教授纯文字无法教授的东西。

### 输入实验代码

实验中的每一行代码都是为了教授某些东西。自己输入会让你慢下来，慢到足以注意到结构。复制粘贴代码通常感觉高效但实际上不是；输入内核代码的手指记忆是你学习它的一部分。即使实验要求你修复一个故意不安全的文件，也要自己输入修复，而不是粘贴建议的答案。

### 将 dmesg 视为文稿的一部分

本章中的几个漏洞只在内核日志输出中显现。`KASSERT` 触发、`WITNESS` 抱怨乱序的锁获取、你自己 `log(9)` 调用的速率限制警告——所有这些都出现在 `dmesg` 中，别无他处。在实验期间关注 `dmesg`。在第二个终端中使用 tail。当它们教授不明显的知识时，将相关行复制到日志中。

### 故意破坏事物

在本章的几个地方，以及一些实验中明确地，你将被要求运行不安全的代码来看看会发生什么。去做吧。实验机器上的内核崩溃是一次廉价的教育经历。每次实验后卸载模块，在日志中记录症状，然后继续。生产系统中的崩溃是昂贵的；实验环境的全部意义在于让你在代价低廉的地方自由地学习这些教训。

### 尽可能结对学习

如果你有学习伙伴，本章很适合结对学习。安全工作极大地受益于第二双眼睛。你们中的一个可以阅读代码寻找漏洞，另一个阅读文字；然后你们可以交换并比较笔记。两种阅读模式发现不同的东西，对话本身就是教育性的。

### 信任迭代

你不会在第一次阅读时记住每个标志、每个常量、每个 priv 标识符。这没关系。重要的是你记住主题的形状、原语的名称，以及当具体问题出现时去哪里查找。特定的标识符在你编写了两三个安全意识驱动程序后就会变成反射；它们不是记忆练习。

### 休息

安全阅读在认知上与性能工作或总线连接工作有着不同的强度。它要求你在脑中保持一个对手的模型，同时阅读旨在为朋友服务的代码。两小时的专注阅读加上真正的休息几乎总是比四小时的苦读更高效。

有了这些习惯，让我们从框定一切的问题开始：驱动程序安全为何重要？

## 第1节：驱动程序安全为何重要

人们很容易将驱动程序安全视为一般软件安全的子集，具有相同的技术和后果，只是应用于不同的代码库。这种框架并不完全错误，但它忽略了驱动程序的独特之处。驱动程序之所以值得拥有自己的一章来讨论安全，是因为驱动程序中安全漏洞的后果与用户空间程序中相同漏洞的后果不同，防御措施也不同。本节构建本章其余部分所依赖的思维模型。

### 内核信任什么

内核是系统中唯一被信任执行某些操作的部分。它是唯一能够读写任何物理内存地址的软件。它是唯一能够直接与硬件通信的软件。它是唯一能够授予或撤销用户空间进程权限的软件。它是保存每个用户的秘密和每个运行程序凭据的软件。当它决定某个请求是否应该成功时，它之上没有任何东西可以推翻这个决定。

这种特权正是拥有内核的全部意义。没有它，内核将无法强制执行使多用户系统成为可能的边界。有了它，内核承担着没有任何用户空间程序承担的责任：每一行内核代码都以整个系统的权限运行，每一行内核代码中的漏洞，原则上都可以被提升为整个系统的权限。

A driver is part of the kernel. Once loaded, a driver's code runs with the same privilege as the rest of the kernel. There is no finer-grained boundary inside the kernel that says "this code is only a driver, so it cannot touch the scheduler." A pointer dereference in your driver, if it lands on the wrong address, can corrupt any data structure the kernel uses. A buffer overflow in your driver, if it is large enough, can overwrite any function pointer the kernel uses. An uninitialised value in your driver, if it flows into the right place, can leak a neighbour's secrets. The kernel trusts the driver completely, because it has no mechanism to distrust it.

这种不对称性是首先要内化的东西。用户空间程序在内核之下运行，内核可以对它们强制执行规则。驱动程序在内核内部运行，除了驱动程序作者自己之外，没有人对它们强制执行规则。

### 内核漏洞改变信任模型

用户空间程序中的漏洞就是漏洞。内核中的漏洞，特别是非特权进程可以到达的驱动程序中的漏洞，通常是更糟糕的东西：它是对整个系统信任模型的改变。这是本章中最重要的思想，值得深入思考。

考虑用户空间文本编辑器中的一个小漏洞：一个写入一个额外字节到缓冲区的差一错误。在最坏的情况下，编辑器崩溃。也许用户丢失几分钟的工作。也许编辑器的沙箱捕获了崩溃，影响更小。后果受限于该用户已经能够做的事情；编辑器以用户权限运行，所以损害不会超出这些权限。

现在考虑驱动程序的 `ioctl` 处理程序中相同的差一错误。如果驱动程序可以从非特权进程到达，非特权用户可以触发差一错误。多出的一个字节落入内核内存。根据它落在哪里，它可能翻转内核用来决定谁被允许做什么的结构中的一个位。聪明的攻击者可以安排该位翻转来改变哪个进程拥有 root 权限。现在差一错误不再是崩溃；它是权限提升。非特权用户变成了 root。系统假设只有授权用户是 root 的信任模型不再成立。

这不是假设性的扩展。这是内核漏洞被转化为利用的标准方式。内核的数据结构在内存中彼此相邻。能够在内核某处写入单个字节的攻击者通常可以——通过足够的巧妙手段——将该字节引导到重要的结构中。在正确的数据结构中几个字节错位就变成了一个可工作的利用。在正确的位置几个字节可以造成"我的编辑器崩溃了"和"攻击者现在拥有这台机器"之间的区别。

This is why the mental framing of security must shift when you move from user-space to the kernel. You are not asking "what is the worst that can happen if this code goes wrong?" You are asking "what is the worst thing that someone could do to the system if they could steer this code to go wrong in exactly the way they wanted?" Those are different questions, and the second one is always the right one to ask inside the kernel.

### 风险清单

将抽象变为具体是有帮助的。如果驱动程序有一个可以从用户空间触发的漏洞，具体有什么风险？清单很长。以下是主要类别，作为在本章转向特定漏洞类别之前将风险印入你脑海的方式。

**权限提升。** 非特权用户获取 root 权限，或 jailed 用户获取主机级权限，或能力模式沙箱内的用户获取该沙箱之外的权限。

**任意内核内存读取。** 攻击者读取他们不应看到的内核内存。这包括加密密钥、密码哈希、碰巧在页缓存中的其他用户的文件内容，以及揭示其他有趣内存位置的内核自身数据结构。

**任意内核内存写入。** 攻击者写入他们不应写入的内核内存。这通常是权限提升的基础，因为它可以用来修改凭据结构、函数指针或其他安全关键状态。

**拒绝服务。** 攻击者导致内核崩溃、挂起或消耗过多资源使系统不再有用。可以被诱导无限循环、分配无限内存或从用户输入触发 `KASSERT` 的驱动程序都是拒绝服务的来源。

**信息泄露。** 攻击者了解他们不应知道的东西：内核指针（使 KASLR 失效）、未初始化缓冲区的内容（可能包含先前调用者的数据），或有关系统上其他进程或设备的元数据。

**持久化。** 攻击者安装通过重启后仍然存在的代码，通常通过写入内核将在引导时重新加载的文件，或通过损坏配置结构。

**沙箱逃逸。** 被限制在 jail、VM 客户机或 Capsicum 能力模式沙箱中的攻击者通过驱动程序漏洞逃逸其限制。

这些都是驱动程序中单个、合理错误的合理后果。错误通常是作者看来无害的东西：遗忘的长度检查、在复制出去之前未清零的结构、两个看起来互斥的路径之间的竞态。本章的目标是帮助你在这些错误变成清单上的任何项目之前看到它们。

### 真实世界的事件

每个主要内核都有基于驱动程序的安全事件历史。FreeBSD 也不例外。在不将其变成漏洞考古练习的情况下，值得列举几种特别有指导意义的事件类型。

经典的情况是**没有权限检查的 ioctl**，驱动程序暴露了一个做非 root 用户不应能做的事情的 ioctl，但在执行之前忘记调用 `priv_check(9)` 或等价物。修复只需添加一行；该漏洞可能启用以 root 身份的任意代码执行。这种模式在数十年间出现在多个内核中。

还有**通过未初始化内存的信息泄露**，驱动程序分配一个结构，填充一些字段，然后将结构复制到用户空间。驱动程序未填充的字段包含分配器碰巧返回的内容，可能包括上一个调用者的数据。随着时间的推移，攻击者已经能够从这类漏洞中提取内核指针、文件内容和加密密钥。

还有**看似无害路径中的缓冲区溢出**，驱动程序从固件二进制文件或 USB 描述符解析结构时不检查数据声称的长度字段。能够控制固件的攻击者（例如，通过连接恶意 USB 设备）可以触发溢出。这类漏洞特别有害，因为攻击者可以是物理存在的：插入一个 USB 闪存盘，然后走开。

还有**`open` 和 `read` 之间的竞态**，两个线程同时打开和读取设备，驱动程序的状态机在同步方面存在间隙。第二个线程观察到半初始化状态并触发崩溃，或更糟的是，被允许继续并看到本应被清除的数据。

还有**TOCTOU 漏洞**，驱动程序在用户空间结构中验证一个值，然后稍后信任该值仍然是相同的。在检查和使用之间，用户空间程序已经更改了该值，驱动程序现在操作从未验证过的数据。

这些都是可以预防的。每种都有一个众所周知的 FreeBSD 原语来防止它。本章的结构是围绕以正确的顺序教授这些原语。

### 安全思维方式

本章的一个反复出现的主题是，安全的代码来自一种特定的思维方式，而不是一组特定的技术。技术很重要；你需要了解它们。但没有思维方式的技术只能产生对作者想到的特定漏洞安全的代码，对作者没想到的每个漏洞则不安全。如果始终应用，思维方式即使在不完善的技术下也能持续产生安全的代码。

这种思维方式有三个习惯。第一，**对每个输入做最坏的假设**。你从用户空间、从设备、从固件、从总线、从 sysctl、从加载器可调参数读取的每个字节，都可能是攻击者能选择的最坏字节。不是因为大多数输入是敌对的，而是因为安全的代码必须即使在它们是敌对的时候也能正确工作。第二，**对环境做最少的假设**。不要仅仅因为测试设置使调用者是 root 就假设调用者是 root；去检查。不要仅仅因为上一个写入者说结构中的字段被清零了就假设它被清零了；自己去清零。不要假设系统处于安全级别 0；去测试。第三，**宁可关闭也不要开放**。当出现问题时，返回错误。当缺少某些东西时，拒绝继续。当检查失败时，停止。当规则不明确时选择不工作的驱动程序是难以被滥用的驱动程序；选择无论如何都继续工作的驱动程序是等待被利用的驱动程序。

这三个习惯不需要记忆。它们需要内化。本章是一个内化它们的工坊。

### 即使是 root 也不被信任

初学者有时会忽略的一个具体点：即使调用者是 root，驱动程序仍然必须验证调用者的输入。这似乎违反直觉。如果调用者是 root，他们已经可以做任何事情；验证他们的输入有什么意义？

重点在于"调用者是 root"是关于授权的声明，而不是关于正确性的声明。root 用户可以要求你的驱动程序做某事，内核会说是。但 root 用户也可能是一个意外传递错误长度的有缺陷的程序。root 用户可能是一个被攻击者接管的受损程序。root 用户可能正在运行一个将指针当作长度的笨拙脚本。在所有这些情况下，你的驱动程序仍然必须行为正常。

具体来说，如果 root 在 `ioctl` 参数中传递 `len` 为 0xFFFFFFFF，正确的行为是返回 `EINVAL`，而不是高兴地将 4GB 的用户内存 `copyin` 到内核缓冲区。Root 并不真的想要那样；root 正在运行一个有漏洞的程序。你的驱动程序的工作是防止该漏洞变成内核漏洞。

这就是为什么输入验证是普遍的。它不是关于不信任调用者；它是关于驱动程序保护自己和内核其余部分免受来自上方的错误——无论是有意还是无意的。

### 边界在哪里

驱动程序存在于几个边界之间。值得明确命名它们，因为不同类别的漏洞存在于不同的边界，防御措施也不同。

**用户-内核边界**将用户态与内核分开。从用户空间进入内核的数据必须被验证；从内核到用户空间的数据必须被清理。`copyin(9)` 和 `copyout(9)` 是安全跨越此边界的主要机制。本章的第3、4和7节处理此边界。

**驱动程序-总线边界**将驱动程序与硬件分开。从设备读取的数据并不总是可信的；恶意设备或固件漏洞可能呈现驱动程序未预期的值。例如，描述符中的长度字段必须受驱动程序自身期望的约束，而不是设备声称的值。第2节涉及这一点。

**权限边界**将不同级别的权限分开：root 与非 root、主机与 jail、内核与能力模式沙箱。权限检查强制执行此边界。第6节深入讨论此内容。

**模块-模块边界**将你的驱动程序与其他内核模块分开。这是防御最少的边界，因为内核默认完全信任自己的模块。这就是下一节讨论驱动程序漏洞爆炸半径的原因之一：它几乎总是比驱动程序本身更大。

### 本章的位置

第29章和第30章在两个意义上教授了环境：架构的和运营的。第29章教你使驱动程序在不同总线和架构之间可移植。第30章教你在虚拟化和容器化环境中使其行为正确。第31章教授第三种环境，即策略：管理员做出的安全相关选择和对手试图违反的选择。三者结合起来，这三章描述了运行时 FreeBSD 驱动程序周围的环境，以及驱动程序作者必须做什么才能成为该环境的负责任公民。

线索继续。第32章将转向设备树和嵌入式开发，这可能感觉像是主题的变化，但实际上是同一线索延续到新硬件。你在这里学到的安全习惯将伴随你到每个 ARM 开发板、每个 RISC-V 系统、每个驱动程序的权限和资源纪律与桌面系统同样重要的嵌入式目标。后面的章节将加深调试的内容，包括本章在高级别上介绍的一些技术。你现在建立的安全习惯将在本书的其余部分和你作为驱动程序作者的整个职业生涯中为你服务。

### 第1节总结

驱动程序是内核的一部分。驱动程序中的每个漏洞都是潜在的内核漏洞，每个内核漏洞都是对系统信任模型的潜在改变。因为爆炸半径如此之大，驱动程序中正确性的标准高于用户空间程序。本章的其余部分将逐步介绍对驱动程序最重要的特定漏洞类别，以及防御它们的 FreeBSD 原语。

本节要记住的一句话是：**在驱动程序中，漏洞不仅仅是错误；它们是对谁可以在系统上做什么的改变**。在阅读本章其余部分时记住这个框架，其他每一句话都会更容易理解。

了解了风险之后，我们转向第一个具体的漏洞类别：内核代码中的缓冲区溢出和内存损坏。

## 第2节：避免缓冲区溢出和内存损坏

缓冲区溢出及其同类——越界读写——是最古老且仍然是最常见的安全漏洞类别之一。它们出现在用户空间代码、内核代码以及不在语言层面强制执行边界的每种语言中。C 就是这样的语言，而内核 C 是具有更锐利边缘的这种语言，因此驱动程序是缓冲区漏洞的肥沃土壤。

本节解释这些漏洞如何出现在内核代码中，为什么它们通常比用户空间中的同类漏洞更严重，以及如何通过构造来编写避免它们的驱动程序代码。它假设读者记得第4章中的 C 内容以及第5章和第14章中的内核 C 内容。如果其中任何部分不牢固，在阅读本节之前简要复习将会有所回报。

### 缓冲区简短回顾

C 中的缓冲区是具有特定大小的内存区域。在驱动程序中，缓冲区来自几个地方。栈分配的缓冲区在函数内部声明为局部变量；它们在函数调用期间存在，分配和释放的成本很低。堆分配的缓冲区来自 `malloc(9)` 或 `uma_zalloc(9)`；它们只要驱动程序保持指向它们的指针就存在。静态分配的缓冲区在文件作用域声明；它们在整个模块的生命周期内存在。每种都有不同的属性和不同的陷阱。

所有缓冲区共有的一件事是大小。写入超过缓冲区末尾，或在其开头之前读取，或使用不适合的值进行索引，就是缓冲区溢出（或下溢）。溢出本身是机制；溢出写入的内容和写入的位置决定了严重程度。

A stack overflow in a driver is the most dangerous kind, because the stack holds return addresses, saved registers, and local variables for the whole call chain. A write past the end of a stack buffer can reach into the caller's return address, and from there into arbitrary code execution. A heap overflow is less directly exploitable, but heap buffers are often adjacent to other kernel data structures, and a heap overflow that lands on the right structure is a clear path to compromise. A static-buffer overflow is the least common but can still lead to compromise if the static buffer is next to other writeable module data.

The vocabulary of "stack" and "heap" overflow should feel familiar from user-space work. The mechanism is the same. The consequences are worse, because the kernel's code and data share an address space with everything else it does.

### 内核代码中溢出是如何发生的

溢出不是因为作者写入了他们不打算写入的内存。它们发生是因为作者写入了他们确实打算写入的内存，但长度或偏移量是错误的。最常见的错误形式是：

**Trusting a length from user space.** The driver's `ioctl` argument contains a length field, and the driver uses that length to decide how much to `copyin` or how large a buffer to allocate. If the length is not bounded, the user can pick a length that makes the copy misbehave.

**Off-by-one in a loop.** A loop that iterates over an array uses `<=` where `<` was intended, or `<` where `<=` was intended. The extra iteration touches memory just past the end of the array.

**Incorrect buffer size in a call.** A call to `copyin`, `strlcpy`, `snprintf`, or similar takes a size argument. The author passes `sizeof(buf)` for a pointer-typed buf, which yields the pointer's size (four or eight bytes) rather than the buffer's size. The call writes far too many bytes.

**Arithmetic overflow in a length calculation.** The author multiplies or adds lengths to compute a buffer size, and the multiplication overflows a 32-bit integer. The resulting "size" is small, the allocation succeeds, and the subsequent copy writes far more than was allocated.

**Truncating a string without terminating it.** The author uses `strncpy` or similar, but `strncpy` does not guarantee a null terminator; a later string operation reads past the end of the buffer.

**Skipping a length check because the code "obviously" cannot reach a bad state.** The author convinces themselves that a given path cannot produce a length greater than some bound, so no check is needed. The path can produce such a length, because the author missed a case.

每一种都是有对策的漏洞类别。本节的其余部分将逐步介绍这些对策。

### 限制一切

最简单和最有效的对策是限制每个长度。在使用来自不可信来源的长度之前，将其与已知最大值进行比较。在分配大小来自不可信来源的缓冲区之前，将大小与已知最大值进行比较。在复制到缓冲区之前，确认复制大小合适。

Concretely, if your `ioctl` handler takes a structure with a `u_int32_t len` field, add a check like this at the very top of the handler:

```c
#define SECDEV_MAX_LEN    4096

static int
secdev_ioctl_set_name(struct secdev_softc *sc, struct secdev_ioctl_args *args)
{
    char *kbuf;
    int error;

    if (args->len > SECDEV_MAX_LEN)
        return (EINVAL);

    kbuf = malloc(args->len + 1, M_SECDEV, M_WAITOK | M_ZERO);
    error = copyin(args->data, kbuf, args->len);
    if (error != 0) {
        free(kbuf, M_SECDEV);
        return (error);
    }
    kbuf[args->len] = '\0';

    /* use kbuf */

    free(kbuf, M_SECDEV);
    return (0);
}
```

函数的第一行就是边界。无论调用者传递什么，`args->len` 现在最多是 `SECDEV_MAX_LEN`。分配是有界的，复制是有界的，空终止在缓冲区内。这种模式是安全驱动程序代码的主力。

What should the bound be? It depends on the semantics of the argument. A name of a device might reasonably be bounded to a few hundred bytes. A configuration blob might be bounded to a few kilobytes. A firmware blob might be bounded to a few megabytes. Pick a number that is generous enough to accommodate legitimate use and small enough that its consequences, if reached, are bearable. If the bound is too small, users will complain about legitimate failures; if it is too large, an attacker can use the bound itself as an amplifier for denial of service. A generous bound is almost always the right choice.

Some drivers derive the bound from the structure of the hardware. A driver for a fixed-size register bank might bound reads and writes to the bank's size. A driver for a ring with 256 entries might bound the index to 255. Bounds derived from hardware structure are particularly robust, because they correspond to a physical constraint rather than an arbitrary choice.

### `sizeof(buf)` 陷阱

C 代码中最常见的缓冲区大小漏洞之一是 `sizeof(buf)` 和 `sizeof(*buf)` 之间的混淆，或者 `sizeof(buf)` 和 `buf` 指向的内存长度之间的混淆。当缓冲区传递给函数时，这个陷阱最常出现。

Consider this unsafe function:

```c
static void
bad_copy(char *dst, const char *src)
{
    strlcpy(dst, src, sizeof(dst));    /* WRONG */
}
```

Here, `dst` is a `char *`, so `sizeof(dst)` is the size of a pointer: 4 on 32-bit systems, 8 on 64-bit systems. The call to `strlcpy` tells it that the destination is 8 bytes long, regardless of how big the real buffer is. On a 64-bit system, the function writes up to 8 bytes and terminates, and the caller's 4096-byte buffer now contains a short string, which is probably not what anyone wanted. On any system, if the caller's buffer was less than 8 bytes, the call overflows it.

The fix is to pass the buffer size explicitly:

```c
static void
good_copy(char *dst, size_t dstlen, const char *src)
{
    strlcpy(dst, src, dstlen);
}
```

The callers then use `sizeof(their_buf)` at the call site, where `their_buf` is known to be the array:

```c
char name[64];
good_copy(name, sizeof(name), user_input);
```

这种模式在 FreeBSD 中如此常见，以至于许多内部函数都遵循它：它们接受 `(buf, bufsize)` 对而不是裸 `buf`。当你编写写入缓冲区的函数时，也这样做。六个月后阅读代码的未来自我会感谢你。

### 有界字符串函数

C's traditional string functions, `strcpy`, `strcat`, `sprintf`, were designed in an era when nobody took buffer overflows seriously. They do not take a size argument; they write until they see a null terminator. In kernel code, they are trouble, because the null terminator can be far away or absent entirely.

FreeBSD provides bounded alternatives:

- `strlcpy(dst, src, dstsize)`: copy at most `dstsize - 1` bytes plus a null terminator. Returns the length of the source string. Safe to use when you know `dstsize` correctly.
- `strlcat(dst, src, dstsize)`: append `src` to `dst`, ensuring the result is at most `dstsize - 1` bytes plus a null terminator. Like `strlcpy`, this is safe when `dstsize` is correct.
- `snprintf(dst, dstsize, fmt, ...)`: format into `dst`, writing at most `dstsize` bytes including the terminator. Returns the number of bytes that would have been written, which may be larger than `dstsize`. Check the return value if you need to know about truncation.

`strncpy` and `strncat` also exist, but they have surprising semantics. `strncpy` pads with nulls if the source is shorter than the destination size, and, more dangerously, it does not null-terminate if the source is longer. `strncat` is confusing in a different way. Prefer `strlcpy` and `strlcat` in new code.

For longer formatted output, the `sbuf(9)` API is safer still. It manages an auto-growing buffer with a clean interface for appending strings, printing formatted output, and bounding the final size. It is overkill for small fixed-size copies but excellent for anything that builds up a longer message. Section 8 returns to `sbuf` in the context of logging.

### 算术与溢出

A subtler class of buffer bug comes from arithmetic on sizes. The classic example is:

```c
uint32_t total = count * elem_size;          /* may overflow */
buf = malloc(total, M_SECDEV, M_WAITOK);
copyin(user_buf, buf, total);
```

If `count * elem_size` overflows a 32-bit `uint32_t`, `total` wraps around to a small number. The `malloc` succeeds with that small number. The `copyin` is asked for the same small number of bytes, which makes the allocation-and-copy pair itself safe. But a later piece of the driver may treat `count * elem_size` as if it produced the full amount, and write past the end of the buffer.

The fix is to check for overflow explicitly:

```c
#include <sys/limits.h>

if (count == 0 || elem_size == 0)
    return (EINVAL);
if (count > SIZE_MAX / elem_size)
    return (EINVAL);
size_t total = count * elem_size;
```

The division is exact (no rounding) for integer types, and the test `count > SIZE_MAX / elem_size` is equivalent to "would the multiplication overflow `size_t`?" This pattern is well worth memorising. It is one of those idioms that appears unnecessary in the common case and essential in the exceptional case.

On modern compilers, FreeBSD also has `__builtin_mul_overflow` and its siblings, which perform the arithmetic and report overflow in a single operation. They are a little more convenient when you have them, but the explicit division check works everywhere.

### 整数类型很重要

Closely related is the choice of integer types for lengths and offsets. If a length is stored as `int`, it can be negative, and a negative value sneaking into a call that expects an unsigned length can cause spectacular misbehaviour. If a length is stored as `short`, it can only represent values up to 32767, and a caller passing a value near that limit can cause truncation.

The safe types for lengths in FreeBSD are `size_t` (unsigned, at least 32 bits, often 64 on 64-bit platforms) and `ssize_t` (signed `size_t`, usually for return values that may be negative to indicate error). Use them consistently. When you take a length as input, convert it to `size_t` at the earliest opportunity. When you store a length, store it as `size_t`. When you pass a length to a FreeBSD primitive, pass a `size_t`.

If the length comes from user space and the user-facing structure uses a `uint32_t`, the conversion on a 64-bit kernel is safe (no truncation), and you should still validate the value before using it. If the user-facing structure uses `int64_t` and the kernel needs a `size_t`, check for negatives and for overflow before the conversion.

### 栈缓冲区廉价但有限

A stack buffer is a local array:

```c
static int
secdev_read_name(struct secdev_softc *sc, struct uio *uio)
{
    char name[64];
    int error;

    mtx_lock(&sc->sc_mtx);
    strlcpy(name, sc->sc_name, sizeof(name));
    mtx_unlock(&sc->sc_mtx);

    error = uiomove(name, strlen(name), uio);
    return (error);
}
```

Stack buffers are allocated automatically, freed automatically when the function returns, and are essentially free to use. They are ideal for small, short-lived data that does not need to outlive the function call.

The limit on stack buffers is the size of the kernel stack itself. FreeBSD's kernel stack is small, typically 16 KiB or 32 KiB depending on the architecture, and that stack must accommodate the whole call chain, including nested calls into the VFS, the scheduler, interrupt handlers, and so on. A driver function that declares a 4 KiB local buffer is already using a quarter of the stack. A driver function that declares a 32 KiB local buffer has almost certainly blown the stack, and the kernel will panic or corrupt memory when it happens.

A safe rule of thumb: keep local buffers under 512 bytes, and preferably under 256 bytes. For anything larger, allocate on the heap. The compiler will not warn you when you declare a stack buffer that is too large; it is the author's responsibility to keep stack usage bounded.

### 堆缓冲区及其生命周期

A heap buffer is allocated dynamically:

```c
char *buf;

buf = malloc(size, M_SECDEV, M_WAITOK | M_ZERO);
/* use buf */
free(buf, M_SECDEV);
```

Heap buffers can be arbitrarily large (up to the available memory), can outlive the function that allocates them, and give the author explicit control over when they are freed. They come at the cost of requiring deliberate attention: every allocation must be paired with a free, every free must happen after the last use, and every free must happen only once.

The rules for heap buffers are:

1. Always check the allocation if you used `M_NOWAIT`. With `M_WAITOK`, the allocation cannot fail; with `M_NOWAIT`, it can return `NULL` and your code must handle that.
2. Pair every `malloc` with exactly one `free`. Not zero, not two.
3. After calling `free`, do not access the buffer. If the pointer may be reused, set it to `NULL` immediately after the free, so that accidental use triggers a null-pointer panic rather than a subtle corruption.
4. If the buffer held sensitive data, zero it with `explicit_bzero` or use `zfree` before freeing.

Section 4 covers these rules in more depth, including the FreeBSD-specific flags on `malloc(9)`.

### 完整示例：安全和不安全的复制例程

To make the patterns concrete, here is an unsafe copy routine that you might find in a first-draft driver, followed by a safe rewrite. Read the unsafe version carefully and see if you can spot all the bugs before looking at the commentary.

```c
/* UNSAFE: do not use */
static int
secdev_bad_copy(struct secdev_softc *sc, struct secdev_ioctl_args *args)
{
    char buf[256];

    copyin(args->data, buf, args->len);
    buf[args->len] = '\0';
    strlcpy(sc->sc_name, buf, sizeof(sc->sc_name));
    return (0);
}
```

There are at least four bugs in those four lines.

First, `copyin`'s return value is ignored. If the user supplied a bad pointer, `copyin` returns `EFAULT`, but the function continues as if the copy succeeded. The subsequent operations on `buf` operate on whatever garbage the stack happened to hold.

Second, `args->len` is not bounded. If the user supplies a `len` of 1000, `copyin` writes 1000 bytes into a 256-byte stack buffer. The stack is corrupted. The driver has just become a vehicle for privilege escalation.

Third, `buf[args->len] = '\0'` writes past the end of the buffer even in the non-malicious case. If `args->len == sizeof(buf)`, the assignment is to `buf[256]`, which is one past the end of the 256-byte array.

Fourth, the function returns 0 regardless of whether anything went wrong. A caller receives a success code and has no way to know that the driver silently dropped their input.

Here is a safe rewrite:

```c
/* SAFE */
static int
secdev_copy_name(struct secdev_softc *sc, struct secdev_ioctl_args *args)
{
    char buf[256];
    int error;

    if (args->len == 0 || args->len >= sizeof(buf))
        return (EINVAL);

    error = copyin(args->data, buf, args->len);
    if (error != 0)
        return (error);

    buf[args->len] = '\0';

    mtx_lock(&sc->sc_mtx);
    strlcpy(sc->sc_name, buf, sizeof(sc->sc_name));
    mtx_unlock(&sc->sc_mtx);

    return (0);
}
```

The bound is now `args->len >= sizeof(buf)`, which ensures that the terminator at `buf[args->len]` fits. The `copyin` return value is checked and propagated. The write to `sc->sc_name` happens under the mutex that protects it, ensuring that another thread reading the field at the same time sees a consistent value. The function returns the error code the caller needs to understand what happened.

The unsafe version is eight lines; the safe version is thirteen. The five extra lines are the difference between a working driver and a security incident.

### 第二个完整示例：描述符长度

Here is a different class of bug that shows up in drivers for devices that present descriptor-like data (USB, virtio, PCIe configuration):

```c
/* UNSAFE */
static void
parse_descriptor(struct secdev_softc *sc, const uint8_t *buf, size_t buflen)
{
    size_t len = buf[0];
    const uint8_t *payload = &buf[1];

    /* copy the payload */
    memcpy(sc->sc_descriptor, payload, len);
}
```

The length is taken from the first byte of the buffer, which is a value the device (or an attacker impersonating it) can set arbitrarily. If `buf[0]` is 200, the `memcpy` copies 200 bytes, regardless of whether `buf` actually contains 200 bytes of valid data or whether `sc->sc_descriptor` is that large. If `buflen` is less than `buf[0] + 1`, the `memcpy` reads past the end of the caller's buffer. If `sizeof(sc->sc_descriptor)` is less than `buf[0]`, the `memcpy` writes past the end of the destination.

The safe version validates both sides of the copy:

```c
/* SAFE */
static int
parse_descriptor(struct secdev_softc *sc, const uint8_t *buf, size_t buflen)
{
    if (buflen < 1)
        return (EINVAL);

    size_t len = buf[0];

    if (len + 1 > buflen)
        return (EINVAL);
    if (len > sizeof(sc->sc_descriptor))
        return (EINVAL);

    memcpy(sc->sc_descriptor, &buf[1], len);
    return (0);
}
```

Three checks, each guarding a different invariant: the buffer has at least one byte, the stated length fits in the buffer, and the stated length fits in the destination. Each check protects against a different adversarial or accidental input.

A careful reader may notice that `len + 1 > buflen` can itself overflow if `len` is `SIZE_MAX`. For a `size_t` taken from a byte, `len` is at most 255, so the overflow cannot happen here; but if you write the same code for a 32-bit length field, the check should be rearranged to `len > buflen - 1` with an explicit `buflen >= 1` check. The habit of watching for arithmetic overflow is the same habit, applied at different scales.

### 缓冲区溢出作为一类漏洞

Stepping back from the specific examples: buffer overflows are not a single bug. They are a family of bugs whose members share a structure: the code writes to or reads from a buffer with an incorrect size or offset. The concrete examples above show several members of the family, but the underlying pattern is the same: a length came from somewhere less trustworthy than the code believed it was, and the code was not prepared.

The countermeasures also share a structure. They all amount to: do not trust the length; check it against a known bound before you use it; keep the bound tight; propagate errors when the check fails; use bounded primitives (`strlcpy`, `snprintf`, `sbuf(9)`) when you have a choice; watch for arithmetic overflow in length calculations; and keep stack buffers small. That short list, consistently applied, eliminates most buffer overflow bugs before they are written.

### 溢出之外的内存损坏

Not every memory-corruption bug is a buffer overflow. Drivers can corrupt memory in several other ways, and a complete treatment of safety must mention them.

**Use-after-free** is writing to, or reading from, a buffer after it has been freed. The allocator has almost certainly handed that memory to some other part of the kernel by now, so the write corrupts whatever that part of the kernel is doing. Section 4 covers use-after-free in depth.

**Double-free** is calling `free` twice on the same pointer. Depending on the allocator, this can corrupt the allocator's own data structures, leading to hard-to-diagnose panics minutes or hours later. Section 4 covers prevention.

**Out-of-bounds read** is the read-only cousin of buffer overflow. It does not corrupt memory directly, but it can leak information (see Section 7) and can cause the kernel to read from an unmapped page, which is a panic. It deserves the same countermeasures as overflow.

**Type confusion** is treating a block of memory as if it had a different type from what it actually has. For example, casting a pointer to the wrong structure type and accessing fields. In kernel C, type confusion is usually caught by the compiler, but it can still happen when a driver deals with void pointers or with structures shared across versions.

**Uninitialised memory use** is reading from a variable before assigning it a value. The value read is whatever happened to be in memory at that location, which may be previous callers' data. Section 7 covers this from the information-leak perspective.

Each of these has its own countermeasures, but the single most effective tool across all of them is the set of kernel sanitizers FreeBSD provides: `INVARIANTS`, `WITNESS`, `KASAN`, `KMSAN`, and `KCOV`. Section 10 covers these tools in depth. The short version: build your driver against a kernel with `INVARIANTS` and `WITNESS` always. Build it against a `KASAN`-enabled kernel during development. Run your tests under the sanitized kernel. The sanitizers will find bugs you would otherwise not find until a customer did.

### 编译器保护如何提供帮助，以及在哪里停止

FreeBSD kernels are usually compiled with several exploit-mitigation features enabled in the compiler. Understanding what they do is part of understanding why certain defensive habits matter more than others.

**Stack-smashing protection (SSP)** inserts a canary value on the stack between local variables and the saved return address. When the function returns, the canary is checked against a reference value; if it has been modified (because a stack-buffer overflow clobbered it), the kernel panics. SSP does not prevent the overflow from happening, but it prevents many overflows from gaining control of execution. Without SSP, overwriting the return address would redirect execution to attacker-controlled code on return. With SSP, the overwrite is detected and the kernel stops.

SSP is heuristic. Not every function gets a canary: functions without stack-allocated buffers, for example, do not need protection. The compiler applies SSP to functions that look risky. A driver author should not assume SSP will catch any particular bug; SSP catches some stack overflows, not all, and catches them only at function return, not at the moment of the overflow.

**kASLR** is orthogonal to SSP. It randomizes the base address of the kernel, loadable modules, and the stack. An attacker who wants to jump to a specific kernel function (say, to bypass a check) must first learn where that function is. kASLR makes this difficult. An information leak that exposes any kernel pointer can undo kASLR for the whole kernel: once you know one function's address, you know the offsets to all the others, and you can compute every other address.

**W^X enforcement** ensures that memory is either writable or executable, never both at once. Historically, attackers would overflow a buffer, write shellcode into the overflowed region, and jump to it. W^X breaks this by refusing to execute from writable memory. Modern attacks therefore use return-oriented programming (ROP), which chains together small snippets of existing code rather than introducing new code. ROP is still possible under W^X, but it is harder, and it is defeated by kASLR (ROP needs to know where the snippets are).

**Guard pages** surround kernel stacks with unmapped pages. A write past the end of the stack hits an unmapped page, causing a page fault that the kernel catches and turns into a panic. This prevents certain stack-smashing attacks from silently corrupting memory adjacent to the stack. The cost is one unusable page per kernel stack, which is cheap.

**Shadow stacks and CFI (control-flow integrity)** are under discussion and partial deployment in modern kernels. They aim to prevent attackers from redirecting execution by verifying that every indirect jump lands at a legitimate target. They are not yet standard in FreeBSD, but the direction of the industry is clear: more compiler-enforced restrictions on what exploit writers can do.

The lesson for driver authors: these protections are real, and they raise the cost of exploitation. But they do not prevent bugs. A buffer overflow is still a bug, even if SSP catches it. An information leak is still a bug, even if kASLR makes it less useful. The compiler protections are a last line of defense; the first line is still your careful code.

When the first line fails, the protections buy time: time for the bug to be found and fixed before an attacker chains it into a complete exploit. An information leak that, combined with a buffer overflow, would have been trivially exploitable in 1995 now requires both bugs to exist in the same driver and several more mitigations to fall. The effect is that bug reports that once meant "this is a root exploit" now often mean "this is a pre-condition for a root exploit". That is progress. But it is progress bought by the compiler, not by the code.

### 第2节总结

缓冲区溢出和内存损坏是 C 代码中最古老的安全漏洞，它们仍然是驱动程序代码出错的最常见方式。对策是众所周知的：限制每个长度，尽可能使用有界原语，注意算术溢出，保持栈缓冲区小，并在开发期间在内核净化器下运行。这些都不昂贵。对于存在于内核中的代码，所有这些都是不可协商的。

本节中的漏洞都来自错误大小的数据到达了错误的缓冲区。下一节转向一个密切相关的问题：错误形状的数据到达了错误的内核函数。这就是用户输入的问题，它是现实世界中驱动程序漏洞的最大来源。

## 第3节：安全处理用户输入

每个导出 `ioctl`、`read`、`write` 或 `mmap` 入口点的驱动程序都是接收用户输入的驱动程序。输入的形状各不相同，但原则不变：来自用户空间的数据必须跨越用户-内核边界，而跨越正是大多数驱动程序安全漏洞发生的地方。

FreeBSD 为驱动程序提供了一小组设计良好的原语来安全地跨越边界。这些原语是 `copyin(9)`、`copyout(9)`、`copyinstr(9)` 和 `uiomove(9)`。如果正确使用，它们使用户输入几乎不可能被错误处理。如果错误使用，它们将边界变成了一个巨大的漏洞。本节教授正确的使用方法。

### 用户-内核边界

在介绍原语之前，将边界本身变得生动是有帮助的。

用户空间程序有自己的地址空间。程序的指针指向仅在该地址空间中有意义的地址。指向程序内存中字节 `0x7fff_1234_5678` 的指针在内核中没有意义；内核对用户内存的视图是间接的，由虚拟内存子系统调解。

当程序进行包含指针的 `ioctl` 调用时（例如，指向驱动程序应该填充的结构的指针），内核不会接收对该内存的内核空间访问。内核接收用户空间地址。直接从内核代码解引用它是不安全的：地址可能无效（用户发送了垃圾指针），它可能指向当前不在内存中的内存（已换出），它可能根本没有映射到当前地址空间，或者它可能位于内核不应该读取的区域。

早期的 UNIX 内核有时在这里很粗心，直接解引用用户指针。结果是一类被称为 ptrace 风格攻击的漏洞，其中用户程序可以通过传递精心制作的指针来诱导内核读取或写入任意地址。现代内核，包括 FreeBSD，从不直接从内核代码解引用用户指针。它们总是通过验证和安全处理访问的专用原语。

原语本身很简单。在我们查看它们之前，关于词汇的一个说明：当手册页和内核说内核地址时，它们指的是在内核地址空间中有意义的地址。当它们说用户地址时，它们指的是由用户空间调用者提供的地址，该地址仅在该调用者的地址空间中有意义。原语在两者之间进行转换，并进行适当的安全检查。

### `copyin(9)` 和 `copyout(9)`

用户-内核边界的两个核心原语是 `copyin(9)` 和 `copyout(9)`：

```c
int copyin(const void *udaddr, void *kaddr, size_t len);
int copyout(const void *kaddr, void *udaddr, size_t len);
```

`copyin` 从用户地址 `udaddr` 复制 `len` 字节到内核地址 `kaddr`。`copyout` 从内核地址 `kaddr` 复制 `len` 字节到用户地址 `udaddr`。两者成功时返回0，如果复制的任何部分失败则返回 `EFAULT`，通常是因为用户地址无效、内存不在内存中，或者访问越过了调用者没有权限的内存。

签名在 `/usr/src/sys/sys/systm.h` 中声明。像大多数内核原语一样，它们名称简短，只做一件事。然而，它们做的那一件事是必不可少的。如果驱动程序通过任何其他方式读取或写入用户内存，驱动程序几乎肯定是错误的。

**Always check the return value.** This is the single most common source of copyin/copyout bugs: a driver calls `copyin` and proceeds as if it succeeded, when in fact it might have returned `EFAULT`. If the copy failed, the destination buffer contains whatever was there before (possibly uninitialised), and operating on it is a recipe for either a crash or an information disclosure. Every call to `copyin` or `copyout` must check the return value and either proceed with success or propagate the error:

```c
error = copyin(args->data, kbuf, args->len);
if (error != 0) {
    free(kbuf, M_SECDEV);
    return (error);
}
```

这种模式在 FreeBSD 内核中出现了数百次。学习它并在每个调用点使用它。

**Never reuse a pointer after a failed copy.** If `copyin` returned `EFAULT`, the buffer may have been partially written. Do not try to "rescue" a partial result; do not assume that the first few bytes are valid. Discard the buffer, zero it if the remains may be sensitive, and return the error.

**Always validate lengths before calling.** We have seen this in Section 2, but it bears repeating here. The `len` you pass to `copyin` comes from somewhere; if it comes from the caller's structure, it must be bounded before the call. An unbounded `len` in a `copyin` is one of the most dangerous patterns in a driver.

**`copyin` and `copyout` can sleep.** These primitives may cause the calling thread to sleep while waiting for a user page to be resident. This means they cannot be called from contexts where sleeping is forbidden: interrupt handlers, spin-mutex critical sections, and the like. If you need to transfer data to or from user space from such a context, defer the work to a different context (typically a taskqueue or a regular process context) and have that context do the copy.

### 用于字符串的 `copyinstr(9)`

来自用户空间的字符串是特殊情况。你不知道它有多长，只知道它以空终止。你想复制它，但不想复制超过你准备的缓冲区，并且你需要处理用户提供字符串在预期范围内没有终止符的情况。

用于此的原语是 `copyinstr(9)`：

```c
int copyinstr(const void *udaddr, void *kaddr, size_t len, size_t *lencopied);
```

`copyinstr` 从 `udaddr` 复制字节到 `kaddr`，直到遇到空字节或复制了 `len` 字节，以先到者为准。如果 `lencopied` 不为 NULL，`*lencopied` 设置为复制的字节数（包括终止符，如果找到的话）。返回值成功时为0，出错时为 `EFAULT`，如果在 `len` 字节内未找到终止符则为 `ENAMETOOLONG`。

The key safety rule is: **always pass a bounded `len`**. `copyinstr` without a bound (or with a huge bound) can cause large amounts of kernel memory to be written, and in older kernels could cause the kernel to scan huge amounts of user memory before giving up. In modern FreeBSD the scan itself is bounded by `len`, but you should still pass a tight bound appropriate to the string's expected size. A path name might reasonably be bounded to `MAXPATHLEN` (which is `PATH_MAX`, currently 1024 on FreeBSD). A device name might be bounded to 64. A command name might be bounded to 32. Pick a bound that fits the use and pass it.

A second safety rule is: **always check the return value**, and treat `ENAMETOOLONG` as a distinct condition from `EFAULT`. The former means the user tried to pass a longer string than you were willing to accept, which is plausibly a legitimate mistake. The latter means the user's pointer was invalid, which may or may not be a legitimate mistake. Your driver may want to return a different error to user space depending on which condition occurred.

A third safety rule is: **check the copied length if you care**. The `lencopied` parameter tells you how many bytes were actually written, including the terminator. If your code depends on knowing the exact length, check it. If your buffer is exactly `len` bytes long and `copyinstr` returned 0, the terminator is at `kbuf[lencopied - 1]`, and the string is `lencopied - 1` bytes long.

A safe use of `copyinstr`:

```c
static int
secdev_ioctl_set_name(struct secdev_softc *sc,
    struct secdev_ioctl_name *args)
{
    char name[SECDEV_NAME_MAX];
    size_t namelen;
    int error;

    error = copyinstr(args->name, name, sizeof(name), &namelen);
    if (error == ENAMETOOLONG)
        return (EINVAL);
    if (error != 0)
        return (error);

    /* namelen includes the terminator; the string is namelen - 1 bytes */
    KASSERT(namelen > 0, ("copyinstr returned zero-length success"));
    KASSERT(name[namelen - 1] == '\0', ("copyinstr missed terminator"));

    mtx_lock(&sc->sc_mtx);
    strlcpy(sc->sc_name, name, sizeof(sc->sc_name));
    mtx_unlock(&sc->sc_mtx);

    return (0);
}
```

The function takes a fixed-size stack buffer, calls `copyinstr` with a tight bound, handles the two error cases distinctly, asserts the invariants that `copyinstr` promises (`namelen > 0`, terminator at `name[namelen - 1]`), and copies into the softc under the lock. This is the canonical pattern.

### 用于流的 `uiomove(9)`

`read` and `write` entry points do not use `copyin`/`copyout` directly; they use `uiomove(9)`, which is a wrapper that handles the iteration over a `struct uio` descriptor. A `uio` describes an I/O operation with potentially multiple buffers (scatter-gather) and tracks how much has been transferred so far.

```c
int uiomove(void *cp, int n, struct uio *uio);
```

`uiomove` copies up to `n` bytes between the kernel buffer `cp` and whatever is described by `uio`. If `uio->uio_rw == UIO_READ`, the copy is kernel-to-user; if `UIO_WRITE`, user-to-kernel. The function updates `uio->uio_offset`, `uio->uio_resid`, and `uio->uio_iov` to reflect the bytes transferred.

Like `copyin`, `uiomove` returns 0 on success and an error code on failure. Like `copyin`, it can sleep. Like `copyin`, the caller must check the return value.

A typical `read` implementation:

```c
static int
secdev_read(struct cdev *dev, struct uio *uio, int flag)
{
    struct secdev_softc *sc = dev->si_drv1;
    char buf[128];
    size_t len;
    int error;

    mtx_lock(&sc->sc_mtx);
    len = strlcpy(buf, sc->sc_name, sizeof(buf));
    mtx_unlock(&sc->sc_mtx);

    if (len >= sizeof(buf))
        len = sizeof(buf) - 1;

    if (uio->uio_offset >= len)
        return (0);   /* EOF */

    error = uiomove(buf + uio->uio_offset, len - uio->uio_offset, uio);
    return (error);
}
```

This handles the case where the user reads past the end of the data (returning 0 to indicate EOF), bounds the copy to the size of the kernel buffer, and propagates any error from `uiomove`. It is a safe pattern for short, fixed data; longer data typically uses `sbuf(9)` internally and copies out with `sbuf_finish`/`sbuf_len`/`uiomove` at the end.

### 验证每个结构的每个字段

当 `ioctl` 接受结构时，驱动程序必须在信任任何字段之前验证每个字段。一个常见的错误是只验证驱动程序立即使用的字段，而忽略稍后使用的字段。结构在 `ioctl` 调用期间存在，驱动程序最终可能使用它未检查的字段。

Concretely, if your `ioctl` takes this structure:

```c
struct secdev_config {
    uint32_t version;       /* protocol version */
    uint32_t flags;         /* configuration flags */
    uint32_t len;           /* length of data */
    uint64_t data;          /* user pointer to data blob */
    char name[64];          /* human-readable name */
};
```

validate every field at the top of the handler:

```c
static int
secdev_ioctl_config(struct secdev_softc *sc, struct secdev_config *cfg)
{
    if (cfg->version != SECDEV_CONFIG_VERSION_1)
        return (ENOTSUP);

    if ((cfg->flags & ~SECDEV_CONFIG_FLAGS_MASK) != 0)
        return (EINVAL);

    if (cfg->len > SECDEV_CONFIG_MAX_LEN)
        return (EINVAL);

    /* Name must be null-terminated within the field. */
    if (memchr(cfg->name, '\0', sizeof(cfg->name)) == NULL)
        return (EINVAL);

    /* ... proceed to use the structure ... */
    return (0);
}
```

四个不变量，每个都被检查和强制执行。驱动程序现在知道 `version`、`flags`、`len` 和 `name` 都在预期范围内。它可以在没有进一步验证的情况下使用它们。没有这些检查，函数中后面的每次使用都变成另一个潜在的漏洞来源。

An important subtlety: when a structure includes reserved fields or padding, the driver must decide what to do when those fields are non-zero. The safe choice is usually to require them to be zero:

```c
if (cfg->reserved1 != 0 || cfg->reserved2 != 0)
    return (EINVAL);
```

This preserves the possibility of using those fields in a future version of the protocol without breaking compatibility: if every current caller passes zero, any future non-zero value is necessarily from a caller that knows about the new version. Without the check, the driver cannot later distinguish old callers (who happened to leave garbage in the reserved fields) from new callers (who are using the field for a new purpose).

### 验证分多部分传入的结构

Some `ioctl`s take a structure that contains a pointer to another block of data. The outer structure is copied in first; the pointer inside it then needs to be followed with a second `copyin`. Every field of both structures must be validated.

```c
struct secdev_ioctl_args {
    uint32_t version;
    uint32_t len;
    uint64_t data;    /* user pointer to a blob of `len` bytes */
};

static int
secdev_ioctl_something(struct secdev_softc *sc,
    struct secdev_ioctl_args *args)
{
    char *blob;
    int error;

    /* Validate the outer structure. */
    if (args->version != SECDEV_IOCTL_VERSION_1)
        return (ENOTSUP);
    if (args->len > SECDEV_MAX_BLOB)
        return (EINVAL);
    if (args->len == 0)
        return (EINVAL);

    blob = malloc(args->len, M_SECDEV, M_WAITOK | M_ZERO);

    /* Copy the inner blob. */
    error = copyin((const void *)(uintptr_t)args->data, blob, args->len);
    if (error != 0) {
        free(blob, M_SECDEV);
        return (error);
    }

    /* ... now validate the inner blob, whose shape depends on the version ... */

    free(blob, M_SECDEV);
    return (0);
}
```

The `uintptr_t` cast is worth commenting on. The user pointer arrives as a `uint64_t` in the structure, to avoid portability issues between 32-bit and 64-bit userlands. The cast to `uintptr_t` and then to `const void *` converts the integer representation back into a pointer. On a 64-bit kernel, this is a no-op; on a 32-bit kernel, the high bits of the `uint64_t` must be validated or dropped. FreeBSD runs on both, and 32-bit userland on a 64-bit kernel (via `COMPAT_FREEBSD32`) is a real case. Be explicit about the cast, and document the assumption.

### "冻结"问题

Some drivers have fields in user-space structures that are pointers, and the driver's convention is that the user-space memory stays valid until a particular operation completes. This pattern is common in drivers that do DMA directly from user memory.

The pattern is tricky because the user can, in principle, change the memory between the driver's validation and the driver's use. Pointer-based DMA is also the wrong idea in modern drivers; safer alternatives include:

- `mmap`, in which the driver maps kernel memory into user space for direct access, with the kernel retaining ownership of the memory and its validity.
- A copy-through-kernel approach, in which the driver always copies in, validates, and operates on the kernel copy.
- The `busdma(9)` framework, which handles user-space buffers correctly when they need to be DMA'd to hardware.

If you find yourself writing code that keeps a user-space pointer around and uses it at a later moment, stop and think. It is almost always the wrong design. Section 5 returns to this issue as a TOCTOU problem.

### 内核地址不会泄露到用户指针中

A recurring class of bug is when a driver, trying to communicate a pointer to user space, copies out a kernel address. The user receives a pointer to kernel memory, which is a spectacular information leak (it reveals the kernel's layout, defeating KASLR) and, if the user can somehow convince the kernel to treat the copied pointer as a user pointer, can become an arbitrary kernel-memory access.

The mistake is usually inadvertent. A common case is a structure that is shared between kernel and user space, and one of its fields is a pointer. If the driver fills in the field with a kernel pointer and then copies the structure to user space, the leak has happened.

The fix is structural: do not share structures between kernel and user that contain pointer fields intended to be used in either space. If a pointer field exists, make it `uint64_t` and treat it as an opaque integer. When the kernel fills in a user-visible pointer-like field, it must pick a value meaningful to user space, not reveal its own internal pointer.

A second class of leak is when a driver copies out a structure that contains uninitialised fields, and one of those fields happens to contain a kernel pointer (for example, because the allocator returned memory that was previously used for something that held a kernel pointer). Section 7 covers this in depth.

### `compat32` 和结构大小

FreeBSD supports running 32-bit user-space programs on a 64-bit kernel through the `COMPAT_FREEBSD32` machinery. For a driver, this means that the structure the caller passes may be a 32-bit structure, with different layout and size from the 64-bit version. If the driver expects the 64-bit structure and the caller passed the 32-bit one, the fields the driver reads will be at the wrong offsets, and the driver will read garbage.

Handling this is outside the scope of a typical driver; the framework helps by offering `ioctl32` entry points and automatic translation for many common cases. If your driver is used from 32-bit user-space and uses custom structures, consult the `freebsd32(9)` manual page and the `sys/compat/freebsd32` subsystem for guidance. Be aware of the issue, and test your driver from a 32-bit userland in the lab environment.

### 更大的示例：完整的 `ioctl` 处理程序

Combining the patterns in this section, here is what a complete, safe `ioctl` handler looks like for a hypothetical operation:

```c
static int
secdev_ioctl(struct cdev *dev, u_long cmd, caddr_t data, int flag,
    struct thread *td)
{
    struct secdev_softc *sc = dev->si_drv1;
    struct secdev_ioctl_args *args;
    char *blob;
    int error;

    switch (cmd) {
    case SECDEV_IOCTL_DO_THING:
        args = (struct secdev_ioctl_args *)data;

        /* 1. Validate every field of the outer structure. */
        if (args->version != SECDEV_IOCTL_VERSION_1)
            return (ENOTSUP);
        if ((args->flags & ~SECDEV_FLAGS_MASK) != 0)
            return (EINVAL);
        if (args->len == 0 || args->len > SECDEV_MAX_BLOB)
            return (EINVAL);

        /* 2. Check that the caller has permission, if required. */
        if ((args->flags & SECDEV_FLAG_PRIVILEGED) != 0) {
            error = priv_check(td, PRIV_DRIVER);
            if (error != 0)
                return (error);
        }

        /* 3. Allocate the kernel-side buffer. */
        blob = malloc(args->len, M_SECDEV, M_WAITOK | M_ZERO);

        /* 4. Copy in the user-space blob. */
        error = copyin((const void *)(uintptr_t)args->data, blob,
            args->len);
        if (error != 0) {
            free(blob, M_SECDEV);
            return (error);
        }

        /* 5. Do the work under the softc lock. */
        mtx_lock(&sc->sc_mtx);
        error = secdev_do_thing(sc, blob, args->len);
        mtx_unlock(&sc->sc_mtx);

        /* 6. Zero and free the kernel buffer (it held user data
         * that might be sensitive). */
        explicit_bzero(blob, args->len);
        free(blob, M_SECDEV);

        return (error);

    default:
        return (ENOTTY);
    }
}
```

Each numbered step is a distinct concern. Each step handles errors locally and propagates them. The allocation is bounded by the validated length; the copy is bounded by the same length; the permission check is explicit; the cleanup is symmetric with the allocation; the final return code communicates success or the specific failure. This is what a safe ioctl handler looks like. It is not short, but every line is there for a reason.

### 用户输入处理中的常见错误

A short checklist of the patterns to watch for, as a reference you can return to while reviewing your own code:

- `copyin` with a length from the user, without a prior bound check.
- `copyinstr` without an explicit bound.
- Return value of `copyin`, `copyout`, or `copyinstr` ignored.
- Structure fields used before they are validated.
- Pointer field cast from `uint64_t` to `void *` without thinking about 32-bit-userland compatibility.
- String field assumed null-terminated without a `memchr` check.
- Length used in arithmetic before being bounded.
- User-space pointer kept around and used later (TOCTOU territory).
- Kernel data structure (with pointer fields) directly copied out.
- Uninitialised fields copied out to user space.

If a code review turns up any of these, pause the review, fix the pattern, and then continue.

### 详细演示：从零开始设计安全的 Ioctl

The accumulated techniques of this section can feel like a long checklist. To show how they come together in practice, let us design a single ioctl carefully, from the user-space interface down to the kernel implementation.

**The problem.** Our driver needs an ioctl that lets a user set a configuration parameter consisting of a name string (bounded length), a mode (enum), and an opaque data blob (variable length). It should also return the driver's interpretation of the configuration (for example, the canonicalized form of the name).

**Defining the interface.** The user-visible structure, defined in a header that will ship with the driver, looks like:

```c
#define SECDEV_NAME_MAX   64
#define SECDEV_BLOB_MAX   (16 * 1024)

enum secdev_mode {
    SECDEV_MODE_OFF = 0,
    SECDEV_MODE_ON = 1,
    SECDEV_MODE_AUTO = 2,
};

struct secdev_config {
    char              sc_name[SECDEV_NAME_MAX];
    uint32_t          sc_mode;
    uint32_t          sc_bloblen;
    void             *sc_blob;
    /* output */
    char              sc_canonical[SECDEV_NAME_MAX];
};
```

Notes on the design:

The name is a fixed-size inline buffer, not a pointer. This is deliberate: it avoids a separate `copyin` for the name and makes the interface simpler. The trade-off is that the buffer is always copied even if the name is short, but for 64 bytes that is negligible.

The mode is `uint32_t` rather than `enum secdev_mode` directly, because struct members that cross the user/kernel boundary should have explicit widths. The kernel validates that the value is one of the known enum values.

The blob uses a separate pointer (`sc_blob`) and a length (`sc_bloblen`). The user sets both, and the kernel uses a second `copyin` to pull the data. The length is bounded by `SECDEV_BLOB_MAX`, a value we (the driver authors) choose based on what the driver is actually going to do with the data.

The canonical output is another fixed inline buffer. The user-space caller may or may not care about this output, but the kernel always fills it.

**The kernel handler.** Let us walk through the implementation step by step. The ioctl framework will copy the structure into the kernel for us, so by the time our handler runs, `cfg` points to kernel memory. The `sc_blob` field, however, is still a user-space pointer that we must handle ourselves.

```c
static int
secdev_ioctl_config(struct secdev_softc *sc, struct secdev_config *cfg,
    struct thread *td)
{
    char kname[SECDEV_NAME_MAX];
    char canonical[SECDEV_NAME_MAX];
    void *kblob = NULL;
    size_t bloblen;
    uint32_t mode;
    int error;

    /* Step 1: Privilege check. */
    error = priv_check(td, PRIV_DRIVER);
    if (error != 0)
        return (error);

    /* Step 2: Jail check. */
    if (jailed(td->td_ucred))
        return (EPERM);

    /* Step 3: Copy and validate the name. */
    bcopy(cfg->sc_name, kname, sizeof(kname));
    kname[sizeof(kname) - 1] = '\0';  /* ensure NUL termination */
    if (strnlen(kname, sizeof(kname)) == 0)
        return (EINVAL);
    if (!secdev_is_valid_name(kname))
        return (EINVAL);

    /* Step 4: Validate the mode. */
    mode = cfg->sc_mode;
    if (mode != SECDEV_MODE_OFF && mode != SECDEV_MODE_ON &&
        mode != SECDEV_MODE_AUTO)
        return (EINVAL);

    /* Step 5: Validate the blob length. */
    bloblen = cfg->sc_bloblen;
    if (bloblen > SECDEV_BLOB_MAX)
        return (EINVAL);

    /* Step 6: Copy in the blob. */
    if (bloblen > 0) {
        kblob = malloc(bloblen, M_SECDEV, M_WAITOK | M_ZERO);
        error = copyin(cfg->sc_blob, kblob, bloblen);
        if (error != 0)
            goto out;
    }

    /* Step 7: Apply the configuration under the lock. */
    mtx_lock(&sc->sc_mtx);
    if (sc->sc_blob != NULL) {
        explicit_bzero(sc->sc_blob, sc->sc_bloblen);
        free(sc->sc_blob, M_SECDEV);
    }
    sc->sc_blob = kblob;
    sc->sc_bloblen = bloblen;
    kblob = NULL;  /* ownership transferred */

    strlcpy(sc->sc_name, kname, sizeof(sc->sc_name));
    sc->sc_mode = mode;

    /* Produce the canonical form while still under the lock. */
    secdev_canonicalize(sc->sc_name, canonical, sizeof(canonical));
    mtx_unlock(&sc->sc_mtx);

    /* Step 8: Fill the output fields. */
    bzero(cfg->sc_canonical, sizeof(cfg->sc_canonical));
    strlcpy(cfg->sc_canonical, canonical, sizeof(cfg->sc_canonical));
    /* (The ioctl framework handles copyout of cfg itself.) */

out:
    if (kblob != NULL) {
        explicit_bzero(kblob, bloblen);
        free(kblob, M_SECDEV);
    }
    return (error);
}
```

Now review this against the patterns we have discussed.

Privilege check. `priv_check(PRIV_DRIVER)` is the first line of business. No unprivileged caller ever reaches the rest.

Jail check. `jailed()` before any host-affecting work.

Name validation. The name is read from the already-copied-in `cfg`, forced NUL-terminated (defensive, in case the user did not terminate it), and whitelisted through `secdev_is_valid_name` (which presumably refuses non-alphanumeric characters).

Mode validation. An explicit whitelist of known mode values. An unknown value returns `EINVAL` immediately.

Length validation. The blob length is checked against a defined maximum before being used for allocation. Without this check, a user could request a multi-gigabyte allocation.

Allocation with `M_ZERO`. The blob buffer is zeroed so that even if `copyin` fails partway, the contents are deterministic.

Error path cleanup. The `out:` label frees `kblob` if we did not transfer ownership. The `kblob = NULL` after transfer prevents a double-free. Every path through the function reaches `out:` with `kblob` in a consistent state.

Explicit zeroing before free. The old blob (if any) is zeroed before being replaced, on the assumption that it may have contained sensitive data. The new blob on error path is also zeroed for the same reason.

Locking. The softc is updated under `sc_mtx`. The canonical form is computed under the lock so the name and canonical match.

Output zeroing. `cfg->sc_canonical` is zeroed before being filled, so padding and any fields the canonicalizer did not set are guaranteed zero.

This function has about forty lines of actual code and roughly a dozen security-relevant decisions. Each decision individually is small; the compound effect is a function that is defensible against nearly every pattern discussed in this chapter. This is what secure driver code looks like in practice: not flashy, not tricky, just careful.

The key insight is that the careful code is the easiest to review, the easiest to maintain, and the one that tends to keep working as the driver evolves. Clever tricks, by contrast, are where bugs hide.

### 第3节总结

用户输入是实践中驱动程序安全漏洞的最大来源。FreeBSD 提供的原语（copyin、copyout、copyinstr、uiomove）设计良好且安全，但必须正确使用：有界长度、检查返回值、验证字段、清零缓冲区和正确大小的目标。在每个用户-内核边界穿越处一致应用这些规则的驱动程序是难以从用户空间攻击的驱动程序。

下一节转向一个密切相关的主题：内存分配。第2节和第3节中的模式假设 `malloc` 和 `free` 被安全使用。第4节使该假设明确化，并展示对 FreeBSD 分配器来说"安全"具体意味着什么。

## 第4节：安全使用内存分配

仔细验证输入但粗心分配内存的驱动程序只完成了一半的工作。内存分配和释放是驱动程序在恶劣条件下（拒绝服务、耗尽、敌对输入）行为最明显的地方，也是少数微妙漏洞——释放后使用、双重释放、泄漏——可能变成完整系统入侵的地方。本节涵盖 FreeBSD 分配器的安全模型以及使驱动程序不成为分配器漏洞农场的惯用法。

### 内核中的 `malloc(9)`

用于通用工作的主要内核分配器是 `malloc(9)`。它在 `/usr/src/sys/sys/malloc.h` 中的声明：

```c
void *malloc(size_t size, struct malloc_type *type, int flags);
void free(void *addr, struct malloc_type *type);
void zfree(void *addr, struct malloc_type *type);
```

与用户空间的 `malloc` 不同，内核版本接受两个额外参数。第一个 `type` 是一个 `struct malloc_type` 标签，用于标识内核的哪个部分正在使用内存。这允许 `vmstat -m` 按子系统报告内核每个部分使用了多少内存。每个驱动程序都应该用 `MALLOC_DECLARE` 和 `MALLOC_DEFINE` 声明自己的 `malloc_type`，以便其分配在记账中可见。

```c
#include <sys/malloc.h>

MALLOC_DECLARE(M_SECDEV);
MALLOC_DEFINE(M_SECDEV, "secdev", "Secure example driver");
```

The first argument, `M_SECDEV`, is the identifier; the second, `"secdev"`, is the short name that appears in `vmstat -m`; the third is a longer description. Use a naming scheme that makes it easy to find the driver's allocations in system output, especially when you are diagnosing a leak.

`flags` 参数控制分配的行为。三个标志是必不可少的：

- `M_WAITOK`: the allocator may sleep to satisfy the allocation. The call cannot fail; it either returns a valid pointer or the kernel panics (which it does only under very unusual circumstances).
- `M_NOWAIT`: the allocator must not sleep. If memory is not immediately available, the call returns `NULL`. The caller must check and handle the `NULL` case.
- `M_ZERO`: the returned memory is zeroed before being returned. Use this whenever the caller will fill in some but not all of the memory, to avoid leaking garbage.

There are others (`M_USE_RESERVE`, `M_NODUMP`, `M_NOWAIT`, `M_EXEC`), but these three are the ones a driver uses daily.

### 何时使用 `M_WAITOK`，何时使用 `M_NOWAIT`

The choice between `M_WAITOK` and `M_NOWAIT` is dictated by context, not preference.

Use `M_WAITOK` when the driver is in a context that can sleep. This is the case in most driver entry points: `open`, `close`, `read`, `write`, `ioctl`, `attach`, `detach`. In these contexts, sleeping is allowed, and the allocator's ability to sleep until memory is available is a significant simplification.

Use `M_NOWAIT` when the driver is in a context that cannot sleep. This is the case in interrupt handlers, inside spin-mutex critical sections, and inside certain callback paths that the kernel specifies as non-sleeping. In these contexts, `M_WAITOK` would trigger a `WITNESS` assertion and a panic. Even if `WITNESS` is not enabled, sleeping in a non-sleeping context can deadlock the system.

The rule of thumb: if you can use `M_WAITOK`, use it. It removes a whole class of error handling (the NULL check), and it makes the driver's behaviour more predictable under memory pressure. Only fall back to `M_NOWAIT` when the context forces it.

With `M_NOWAIT`, you must check the return value:

```c
buf = malloc(size, M_SECDEV, M_NOWAIT);
if (buf == NULL)
    return (ENOMEM);
```

Failure to check is a null-pointer panic waiting to happen. The compiler will not warn you about it.

### `M_ZERO` 是你的朋友

One of the subtlest classes of driver bug is the one where the driver allocates memory, fills in some fields, and then uses or exposes the rest. The "rest" is whatever the allocator happened to return, which in FreeBSD is whatever the allocator's free list last had there. If that memory held another subsystem's data before being freed, a driver that fails to clear it may accidentally expose that data (an information leak) or may behave incorrectly because a field it did not set has a non-zero value.

`M_ZERO` prevents both problems:

```c
struct secdev_state *st;

st = malloc(sizeof(*st), M_SECDEV, M_WAITOK | M_ZERO);
```

After this call, every byte of `*st` is zero. The driver can then fill in specific fields and trust that everything else is either zero or set explicitly. This is so important for safety that many FreeBSD driver authors treat `M_ZERO` as the default, adding it unless there is a specific reason not to.

The exception is large allocations where you are certain you will overwrite every byte before use (for example, a buffer that is immediately filled by `copyin`). In that case, `M_ZERO` is a small waste, and you can omit it. In all other cases, prefer `M_ZERO` and accept the small cost.

A particularly important case: **any structure that will be copied to user space must either have been `M_ZERO`'d at allocation time or have had every byte explicitly set before the copy**. Otherwise the structure may include kernel data that was there before. Section 7 returns to this.

### 用于高频分配的 `uma_zone`

For allocations that happen many times per second with a fixed size, FreeBSD offers the UMA zone allocator:

```c
uma_zone_t uma_zcreate(const char *name, size_t size, ...);
void *uma_zalloc(uma_zone_t zone, int flags);
void uma_zfree(uma_zone_t zone, void *item);
```

UMA zones are significantly faster than `malloc` for repeated small allocations, because they maintain per-CPU caches and avoid the global allocator lock for most operations. Drivers that handle network packets, I/O requests, or other high-frequency events typically use UMA zones instead of `malloc`.

The security properties of UMA zones are similar to those of `malloc`. You still pass `M_WAITOK` or `M_NOWAIT`. You still may pass `M_ZERO` (or you may use `uma_zcreate_arg`'s `uminit`/`ctor`/`dtor` arguments to manage initial state). You still must check NULL on `M_NOWAIT`.

UMA has one additional security consideration worth knowing: **items returned to a zone are not zeroed by default**. An item freed with `uma_zfree` may retain its previous contents and be handed out to a subsequent `uma_zalloc` with that same content. If the item held sensitive data, the driver must zero it before freeing, or must pass `M_ZERO` on every allocation, or must use the `uminit` constructor machinery to zero on allocation. The safest default is to use `explicit_bzero` on the item before calling `uma_zfree`.

### 释放后使用：它是什么以及为什么重要

A use-after-free bug occurs when a driver frees a pointer and then uses it. The allocator has, by now, almost certainly handed that memory to some other part of the kernel. Writes to the freed pointer corrupt that other part of the kernel; reads from it return whatever is now stored there.

The classic pattern:

```c
/* UNSAFE */
static void
secdev_cleanup(struct secdev_softc *sc)
{
    free(sc->sc_buf, M_SECDEV);
    /* sc->sc_buf is now dangling */

    /* later, elsewhere, something calls: */
    secdev_use_buf(sc);   /* crash or silent corruption */
}
```

The fix has two parts. First, set the pointer to NULL immediately after freeing it, so that any subsequent use is a null-pointer dereference (an immediate, diagnosable crash) rather than a dangling-pointer access (silent corruption):

```c
free(sc->sc_buf, M_SECDEV);
sc->sc_buf = NULL;
```

Second, audit the code paths that might still hold references to the freed memory. The NULL-assignment prevents crashes at `sc->sc_buf` accesses, but a local variable or a caller's parameter that still holds the old pointer is not protected. The discipline is to free memory only when you are sure no one else holds a pointer to it. Reference counts (`refcount(9)`) are the FreeBSD primitive for this.

A variant of the bug is the **use-after-detach** pattern, in which a driver frees its softc during `detach` but an interrupt handler or a callback still runs and accesses the freed softc. The fix is to drain all asynchronous activity before freeing in `detach`: cancel outstanding callouts with `callout_drain`, drain taskqueues with `taskqueue_drain`, teardown interrupt handlers with `bus_teardown_intr`, and so on. Once all async paths are quiesced, the free is safe.

### 双重释放：它是什么以及为什么重要

A double-free occurs when a driver calls `free` twice on the same pointer. The first `free` hands the memory back to the allocator. The second `free` corrupts the allocator's internal bookkeeping, because it tries to insert the same memory into the free list twice.

FreeBSD's allocator detects many double-frees and panics immediately (especially with `INVARIANTS` enabled). But some double-frees slip past the detection, and the consequences are subtle: a later allocation may return memory that is claimed to be available but is actually still in use somewhere.

The prevention is the same NULL-assignment pattern:

```c
free(sc->sc_buf, M_SECDEV);
sc->sc_buf = NULL;
```

`free(NULL, ...)` is defined to be a no-op in FreeBSD (as in most allocators), so a second call with `sc->sc_buf == NULL` does nothing. The NULL-assignment turns double-free into a safe no-op.

A related pattern is the **error-path double-free**, in which a function's cleanup logic frees a pointer, and then an outer function also frees the same pointer. The defence is to decide, explicitly, which function owns each allocation, and to have ownership transferred at clear moments. "Who frees this?" is a question that should have a clear answer at every line of the code.

### 内存泄漏是安全问题

A memory leak is a piece of memory that is allocated and never freed. In a long-running driver, leaks accumulate. Eventually the kernel runs out of memory, either for the driver's subsystem or for the system as a whole, and bad things happen.

Why is a leak a security problem? Two reasons. First, a leak is a denial-of-service vector: an attacker who can trigger an allocation without a corresponding free can exhaust memory. If the attacker is unprivileged but the driver's `ioctl` allocates memory on each call, the attacker can loop on `ioctl` until the kernel OOM-kills something important. Second, a leak often hides other bugs: the leak's accumulation pressure changes the behaviour of subsequent allocations (more frequent `M_NOWAIT` failures, more unpredictable page cache), which can make racy or allocation-dependent bugs surface in production.

The prevention is discipline in allocation ownership: for every `malloc`, there must be exactly one `free`, reachable on every code path. The FreeBSD `vmstat -m` tool makes leak tracking easier in practice: `vmstat -m | grep secdev` shows, per type, how many allocations are outstanding. A driver with a leak will show a steadily rising number under load; a driver without will show a stable number.

For new drivers, it is worth stress-testing the driver in the lab for leaks: open and close the device a million times in a loop, run the full `ioctl` matrix repeatedly, watch `vmstat -m` for the driver's type, and look for growth. Any sustained growth is a leak. Leaks found in the lab are a thousand times cheaper to fix than leaks found in production.

### 用于敏感数据的 `explicit_bzero` 和 `zfree`

Some data should not be allowed to linger in memory after the driver is done with it. Cryptographic keys, user passwords, device secrets, anything whose exposure in a memory snapshot would be harmful, must be erased before the memory is freed.

The naive approach is to use `bzero` or `memset(buf, 0, len)` before the free. This works, but it has a subtle flaw: the optimiser may remove the `bzero` if it can prove that the memory is not read after. The optimiser's logic is correct as far as language semantics go, but it defeats the security intent.

The correct primitive is `explicit_bzero(9)`:

```c
void explicit_bzero(void *buf, size_t len);
```

`explicit_bzero` is declared in `/usr/src/sys/sys/systm.h`. It performs the zeroing and is guaranteed by the compiler not to be optimised away, even if the memory is not read after. Use it for any buffer that holds sensitive data:

```c
explicit_bzero(key_buf, sizeof(key_buf));
free(key_buf, M_SECDEV);
```

FreeBSD also provides `zfree(9)`, which zeros the memory before freeing:

```c
void zfree(void *addr, struct malloc_type *type);
```

`zfree` is convenient: it combines the zero and the free into one call. It first zeros the memory using `explicit_bzero`, then frees it. Use `zfree` when you are about to free a buffer that held sensitive data. Use `explicit_bzero` followed by `free` if you need to zero the buffer without freeing it, or if you are working with memory from a source other than `malloc`.

A common question: what is "sensitive data"? The conservative answer is that any data that came from user space should be treated as sensitive, because you cannot know what it represents to the user. A more pragmatic answer is that data that is obviously a secret (a key, a password hash, a nonce, authentication material) must be zeroed, and data that might reveal information about the user's activities (file contents, network payloads, command text) should be zeroed when the driver is finished with it. When in doubt, zero. The cost is small.

### `malloc_type` 标签与可追溯性

The `malloc_type` tag on every allocation serves several purposes. It makes allocations visible in `vmstat -m`. It helps with panic debugging, because the tag is recorded in the allocator's metadata. It helps the allocator's own accounting, and in some configurations it enables per-type memory limits.

A driver that uses a single `malloc_type` for all its allocations is easier to audit than a driver that uses many. Create one tag per logical subsystem within the driver, not one per allocation site. For small drivers, a single tag is usually enough.

The declaration pattern:

```c
/* At the top of the driver source file: */
MALLOC_DECLARE(M_SECDEV);
MALLOC_DEFINE(M_SECDEV, "secdev", "Secure example driver");

/* Allocations throughout the driver use M_SECDEV: */
buf = malloc(size, M_SECDEV, M_WAITOK | M_ZERO);
```

The `MALLOC_DECLARE` declares the tag for external visibility; the `MALLOC_DEFINE` actually allocates it (and registers it with the accounting system). Both are needed. Do not put `MALLOC_DEFINE` in a header, because the kernel linker will complain about duplicate definitions if multiple object files include the header.

### Softc 的生命周期

The softc is the driver's per-instance state. It is typically allocated during `attach` and freed during `detach`. The softc's lifetime is one of the most important things to get right in a driver.

The allocation usually happens via `device_get_softc`, which returns a pointer to a structure whose size was declared at driver-registration time. This means the softc memory is owned by the bus, not by the driver; the driver does not call `malloc` for it, and the driver does not call `free`. The bus allocates the softc when the driver is bound to the device and frees it when the driver is detached.

But the softc often contains pointers to other memory that the driver did allocate. Those pointers must be freed in `detach`, in the reverse order of their allocation. A typical pattern:

```c
static int
secdev_detach(device_t dev)
{
    struct secdev_softc *sc = device_get_softc(dev);

    /* Reverse order of allocation. */

    /* 1. Stop taking new work. */
    destroy_dev(sc->sc_cdev);

    /* 2. Drain async activity. */
    callout_drain(&sc->sc_callout);
    taskqueue_drain(sc->sc_taskqueue, &sc->sc_task);

    /* 3. Free allocated resources. */
    if (sc->sc_blob != NULL) {
        explicit_bzero(sc->sc_blob, sc->sc_bloblen);
        free(sc->sc_blob, M_SECDEV);
        sc->sc_blob = NULL;
    }

    /* 4. Destroy synchronization primitives. */
    mtx_destroy(&sc->sc_mtx);

    /* 5. Release bus resources. */
    bus_release_resources(dev, secdev_spec, sc->sc_res);

    return (0);
}
```

Each step handles a specific concern. The order matters: destroy the device node before freeing resources the device callbacks depend on; drain async activity before freeing data the async paths might touch; destroy synchronization primitives last.

A slip in any of these orderings is a bug. The wrong order can produce use-after-free or double-free patterns. The lab later in the chapter walks through a detach function that has subtle ordering bugs and asks you to fix them.

### 完整的分配/释放模式

Pulling the patterns together, here is a safe allocation and use sequence:

```c
static int
secdev_load_blob(struct secdev_softc *sc, struct secdev_blob_args *args)
{
    char *blob = NULL;
    int error;

    if (args->len == 0 || args->len > SECDEV_MAX_BLOB)
        return (EINVAL);

    blob = malloc(args->len, M_SECDEV, M_WAITOK | M_ZERO);

    error = copyin((const void *)(uintptr_t)args->data, blob, args->len);
    if (error != 0)
        goto done;

    error = secdev_validate_blob(blob, args->len);
    if (error != 0)
        goto done;

    mtx_lock(&sc->sc_mtx);
    if (sc->sc_blob != NULL) {
        /* replace existing */
        explicit_bzero(sc->sc_blob, sc->sc_bloblen);
        free(sc->sc_blob, M_SECDEV);
    }
    sc->sc_blob = blob;
    sc->sc_bloblen = args->len;
    blob = NULL;  /* ownership transferred */
    mtx_unlock(&sc->sc_mtx);

done:
    if (blob != NULL) {
        explicit_bzero(blob, args->len);
        free(blob, M_SECDEV);
    }
    return (error);
}
```

The function has a single exit point via the `done` label. The `blob = NULL` after ownership transfer ensures that the cleanup at `done` sees the transfer and does not re-free. The `explicit_bzero` before every `free` zeroes the buffer in case it held sensitive data. The existing `sc->sc_blob`, if present, is zeroed and freed before being replaced, to avoid leaking the old blob's contents.

This pattern (single exit point, ownership transfer, explicit zeroing, checked allocation, checked copyin) appears in variations throughout the FreeBSD kernel. Learn it well.

### 深入了解 UMA 区域

`malloc(9)` is a general-purpose allocator suited to varying sizes. For fixed-size objects that are allocated and freed frequently, the UMA zone allocator is often the better choice. UMA stands for Universal Memory Allocator, and it is declared in `/usr/src/sys/vm/uma.h`.

A UMA zone is created once, at module load, and holds a pool of objects of a fixed size. `uma_zalloc(9)` returns an object from the pool (allocating a fresh one if necessary). `uma_zfree(9)` returns an object to the pool (or frees it back to the kernel if the pool is full). Because allocations come from a pre-configured pool, they are faster than general `malloc` and have better cache locality.

Creating a zone:

```c
static uma_zone_t secdev_packet_zone;

static int
secdev_modevent(module_t mod, int event, void *arg)
{
    switch (event) {
    case MOD_LOAD:
        secdev_packet_zone = uma_zcreate("secdev_packet",
            sizeof(struct secdev_packet),
            NULL,   /* ctor */
            NULL,   /* dtor */
            NULL,   /* init */
            NULL,   /* fini */
            UMA_ALIGN_PTR, 0);
        return (0);

    case MOD_UNLOAD:
        uma_zdestroy(secdev_packet_zone);
        return (0);
    }
    return (EOPNOTSUPP);
}
```

Using a zone:

```c
struct secdev_packet *pkt;

pkt = uma_zalloc(secdev_packet_zone, M_WAITOK | M_ZERO);
/* ... use pkt ... */
uma_zfree(secdev_packet_zone, pkt);
```

The security advantages of a UMA zone over `malloc`:

A zone can have a constructor and destructor that initialize or finalize objects. This can guarantee that every object returned to the caller is in a known state.

A zone is named, so `vmstat -z` attributes allocations to it. This helps detect leaks and unusual memory patterns in specific subsystems.

The pool of objects can be drained under memory pressure. A malloc allocation is held for its lifetime; a UMA zone object can be returned to the kernel when freed if the pool is above its high-water mark.

The security pitfalls:

An object returned to the zone is not automatically zeroed. If the zone holds objects that may contain sensitive data, either add a destructor that zeros, or zero explicitly before freeing:

```c
explicit_bzero(pkt, sizeof(*pkt));
uma_zfree(secdev_packet_zone, pkt);
```

Because UMA reuses objects quickly, an object you just freed may be handed to another caller almost immediately. If the other caller is a different thread in another subsystem, residual data could flow between them. The fix, again, is explicit zeroing.

A destructor function passed to `uma_zcreate` is called when an object is about to be freed back to the kernel (not when it returns to the pool). For zeroing on every free, use `M_ZERO` on `uma_zalloc` (which zeros on allocation, equivalent to `bzero` immediately after) or zero explicitly.

UMA zones are not appropriate for every driver allocation. For one-off or irregular allocations, `malloc(9)` is simpler. For high-frequency fixed-size objects, UMA wins on performance and makes memory accounting easier. Choose based on access pattern.

### 共享对象的引用计数

When an object in your driver can be held by multiple contexts (a softc that is referenced by both a callout and user-space file descriptors, for example), reference counting is the canonical tool for lifetime management. The `refcount(9)` family in `/usr/src/sys/sys/refcount.h` provides simple atomic helpers:

```c
refcount_init(&obj->refcnt, 1);  /* initial reference */
refcount_acquire(&obj->refcnt);  /* acquire an additional reference */
if (refcount_release(&obj->refcnt)) {
    /* last reference dropped; caller frees */
    free(obj, M_SECDEV);
}
```

The invariant is simple: each context that holds a pointer to the object also holds a reference. When it finishes, it releases. Whichever context is last to release is responsible for freeing.

Used correctly, refcounts prevent the classic "who frees it" ambiguity. Used incorrectly (unbalanced acquires and releases), they produce leaks or use-after-frees. The discipline is:

Every path that obtains a pointer to the object acquires a reference.

Every path that releases the pointer calls `refcount_release` and checks the return value.

A single "owning" reference is held by whoever created the object; the owner is the default releaser.

Even simple refcount usage catches a large class of lifetime bugs. For complex drivers with multiple concurrent contexts, refcounts are indispensable.

### 第4节总结

如果正确使用，FreeBSD 分配器是安全的。规则很简单：检查 `M_NOWAIT` 返回值，优先使用 `M_ZERO`，释放前清零敏感数据，在每个代码路径上将每次 `malloc` 与恰好一次 `free` 配对，释放后将指针设置为 NULL，在释放这些活动触及的结构之前排空异步活动，并使用每个驱动程序的 `malloc_type` 进行追溯。遵循这些规则的驱动程序不会有泄漏、释放后使用或双重释放。

下一节转向一类相关但不同的漏洞：竞态和 TOCTOU 漏洞。这些是两个线程或两个时刻交互不良的地方，安全后果往往隐藏其中。

## 第5节：防止竞态条件和 TOCTOU 漏洞

当驱动程序的正确性取决于它不控制的事件的相对时间时，就会发生竞态条件。TOCTOU 漏洞（检查时间到使用时间）是一种特殊的竞态，驱动程序在一个时刻检查条件，然后在较晚的时刻对相同数据采取行动，假设条件仍然为真。在此期间，某些东西发生了变化。检查是有效的。行动是有效的。组合是一个漏洞。从安全角度来看，竞态和 TOCTOU 漏洞是驱动程序可能拥有的最危险的缺陷之一，因为它们通常允许攻击者绕过单独阅读时看起来正确的检查。

第19章已经涵盖了并发、锁和同步原语。那里的目标是正确性。本节通过安全视角重新审视相同的工具。我们不是在问"我的驱动程序会崩溃吗"。我们是在问"攻击者能否安排时序使我写的检查变得无用"。

### 驱动程序中竞态是如何产生的

FreeBSD 驱动程序在多线程环境中运行。有几件事可以同时发生：

Two different user processes can call `read(2)`, `write(2)`, or `ioctl(2)` on the same device file. If the driver has a single `softc`, both calls run against the same state.

One thread can be running your ioctl handler while an interrupt handler for the same device fires on another CPU.

A user thread can be in the middle of your driver while a callout or taskqueue entry scheduled earlier also runs.

The device can be unplugged, causing `detach` to run while any of the above is still in progress.

在没有适当同步的情况下被多个上下文触及的任何共享数据都是潜在的竞态。当共享数据控制访问、验证输入、跟踪缓冲区大小或保存生命周期信息时，竞态就变成了安全问题。

### TOCTOU 模式

The simplest TOCTOU pattern in a driver looks like this:

```c
if (sc->sc_initialized) {
    use(sc->sc_buffer);
}
```

Read it carefully. Nothing about it is obviously wrong. The driver checks that the buffer is initialized, then uses it. But if another thread can set `sc->sc_initialized` to `false` and free `sc->sc_buffer` between the check and the use, the use touches freed memory. The attacker does not need to corrupt the flag or the pointer. They only need to arrange timing.

A more subtle TOCTOU happens with user memory:

```c
if (args->len > MAX_LEN)
    return (EINVAL);
error = copyin(args->data, kbuf, args->len);
```

Look at `args`. If it was already copied in, this is safe. But if `args` still points into user space, a second user thread can change `args->len` between the check and the `copyin`. The check validates the old length. The copy uses the new length. If the new length exceeds `MAX_LEN`, the `copyin` overruns `kbuf`.

The fix is copy-then-check, which we already covered in Section 3. The reason this technique exists is precisely because TOCTOU on user memory is a real attack vector. Always copy user data into kernel space first, then validate, then use.

### 真实世界示例：带路径的 Ioctl

Imagine an ioctl that takes a path and does something with the file:

```c
/* UNSAFE */
static int
secdev_open_path(struct secdev_softc *sc, struct secdev_path_arg *args)
{
    struct nameidata nd;
    int error;

    /* Check path length */
    if (strnlen(args->path, sizeof(args->path)) >= sizeof(args->path))
        return (ENAMETOOLONG);

    NDINIT(&nd, LOOKUP, 0, UIO_USERSPACE, args->path);
    error = namei(&nd);
    /* ... */
}
```

This has two races. First, `args->path` is still in user space if `args` was not copied in; a user thread can change it between the `strnlen` check and `namei`. Second, even if `args` was copied, using `UIO_USERSPACE` tells the VFS layer to read the path from user space, at which point the process can modify it again before VFS reads it. The fix is to copy the path into kernel space with `copyinstr(9)`, validate it as a kernel string, then pass it to VFS with `UIO_SYSSPACE`.

```c
/* SAFE */
static int
secdev_open_path(struct secdev_softc *sc, struct secdev_path_arg *args)
{
    struct nameidata nd;
    char kpath[MAXPATHLEN];
    size_t done;
    int error;

    error = copyinstr(args->path, kpath, sizeof(kpath), &done);
    if (error != 0)
        return (error);

    NDINIT(&nd, LOOKUP, 0, UIO_SYSSPACE, kpath);
    error = namei(&nd);
    /* ... */
}
```

The corrected version copies the path into the kernel exactly once, validates it (by virtue of `copyinstr` bounding the length and guaranteeing a NUL terminator), then hands a stable kernel string to the VFS layer. The user process can change `args->path` as often as it likes; we are no longer reading from there.

### 共享状态与加锁

For races between concurrent in-kernel contexts, the tool is a lock. FreeBSD offers several. The most common in drivers are:

`mtx_t`, a mutex, created with `mtx_init(9)`. Mutexes are fast, short, and must not be held across sleeps. Use them to protect a small critical section.

`sx_t`, a shared-exclusive lock, created with `sx_init(9)`. Shared-exclusive locks can be held across sleeps. Use them when the critical section includes something like `malloc(M_WAITOK)` or a VFS call.

`struct rwlock`, a read-write lock, for the read-mostly case. Multiple readers can hold the lock in shared mode; an exclusive writer excludes all readers.

`struct mtx` paired with condition variables (`cv_init(9)`, `cv_wait(9)`, `cv_signal(9)`) for producer-consumer patterns.

The rules for safe locking are simple and absolute:

Define exactly what data each lock protects. Write it in a comment next to the softc field.

Acquire the lock before reading or writing the protected data. Release it afterwards.

Do not hold locks longer than necessary. Long critical sections hurt performance and increase deadlock risk.

Acquire multiple locks in a consistent order across all code paths. Inconsistent ordering leads to deadlock.

Do not sleep while holding a mutex. Convert to an sx lock or drop the mutex first.

Do not call into user space (`copyin`, `copyout`) while holding a mutex. Copy first, then lock. Release, then copy back.

### 深入了解：修复有竞态的驱动程序

Consider the following minimal handler:

```c
/* UNSAFE: races on sc_open */
static int
secdev_open(struct cdev *dev, int oflags, int devtype, struct thread *td)
{
    struct secdev_softc *sc = dev->si_drv1;

    if (sc->sc_open)
        return (EBUSY);
    sc->sc_open = true;
    return (0);
}

static int
secdev_close(struct cdev *dev, int fflags, int devtype, struct thread *td)
{
    struct secdev_softc *sc = dev->si_drv1;

    sc->sc_open = false;
    return (0);
}
```

The intent is that only one process can have the device open at a time. The bug is that `sc_open` is checked and set without a lock. Two concurrent `open(2)` calls can both read `sc_open == false`, both decide they are the first, and both set it to `true`. Both succeed. Now two processes share a device that was meant to be exclusive. This is a real-world bug class that has affected real drivers. Fix:

```c
/* SAFE */
static int
secdev_open(struct cdev *dev, int oflags, int devtype, struct thread *td)
{
    struct secdev_softc *sc = dev->si_drv1;
    int error = 0;

    mtx_lock(&sc->sc_mtx);
    if (sc->sc_open)
        error = EBUSY;
    else
        sc->sc_open = true;
    mtx_unlock(&sc->sc_mtx);
    return (error);
}

static int
secdev_close(struct cdev *dev, int fflags, int devtype, struct thread *td)
{
    struct secdev_softc *sc = dev->si_drv1;

    mtx_lock(&sc->sc_mtx);
    sc->sc_open = false;
    mtx_unlock(&sc->sc_mtx);
    return (0);
}
```

Now the read and the write happen inside a single critical section. Only one caller at a time can be inside that section, so the check-then-set sequence is atomic from the perspective of any other caller.

### 分离时的生命周期竞态

The hardest races in drivers are lifetime races around `detach`. The device goes away, but a user thread is still inside your ioctl handler, or an interrupt is in flight, or a callout is pending. If `detach` frees the softc while one of these references it, you have a use-after-free.

FreeBSD gives you tools to handle this:

`callout_drain(9)` waits for any scheduled callout to finish before returning. Call it in `detach` before freeing anything the callout touches.

`taskqueue_drain(9)` and `taskqueue_drain_all(9)` wait for pending tasks to complete.

`destroy_dev(9)` marks a character device as gone and waits for all in-flight threads to leave the device's d_* methods before returning. After `destroy_dev`, no new threads can enter and no old threads remain.

`bus_teardown_intr(9)` removes an interrupt handler and waits for any in-flight instance of that handler to complete.

A correct `detach` function in a driver that has all of these resources looks roughly like:

```c
static int
secdev_detach(device_t dev)
{
    struct secdev_softc *sc = device_get_softc(dev);

    /* 1. Prevent new user-space entries. */
    if (sc->sc_cdev != NULL)
        destroy_dev(sc->sc_cdev);

    /* 2. Drain asynchronous activity. */
    callout_drain(&sc->sc_callout);
    taskqueue_drain_all(sc->sc_taskqueue);

    /* 3. Tear down interrupts (if any). */
    if (sc->sc_intr_cookie != NULL)
        bus_teardown_intr(dev, sc->sc_irq, sc->sc_intr_cookie);

    /* 4. Free resources. */
    /* ... */

    /* 5. Destroy lock last. */
    mtx_destroy(&sc->sc_mtx);
    return (0);
}
```

The order matters. We first stop accepting new work, then drain all in-flight work, then free the resources that the in-flight work was using. If we freed resources first and drained second, a callout still running could touch freed memory. That is a classic detach-time use-after-free, and it is a security bug, not just a crash.

### 原子操作与无锁代码

FreeBSD provides atomic operations (`atomic_add_int`, `atomic_cmpset_int`, and so on) in `/usr/src/sys/sys/atomic_common.h` and architecture-specific headers. Atomics are useful for counters, reference counts, and simple flags. They are not a substitute for locks when multiple related fields must change together.

A common beginner mistake is to say "I will use an atomic to avoid a lock". Sometimes this is correct. Often it leads to a subtly broken data structure because the atomic operation only makes one field safe, while the code really needed two fields updated together.

The safe rule is: if you can express the invariant with a single atomic read or write, an atomic may be appropriate. If the invariant spans multiple fields or a compound condition, use a lock.

### 引用计数作为生命周期工具

When an object can be referenced from multiple contexts, a refcount helps manage lifetime. `refcount_init`, `refcount_acquire`, and `refcount_release` (declared in `/usr/src/sys/sys/refcount.h`) give you a simple atomic refcount. The last release returns true, at which point the caller is responsible for freeing the object.

Refcounts solve the classic problem where context A and context B both hold a pointer to an object. Either can finish with it first. The one that finishes last frees it. Neither needs to know whether the other is done, because the refcount tracks that for them.

A driver that uses a refcount on its softc, or on per-open state, can release that state safely even under concurrent access. The cost is some care at every entry and exit point to balance acquires and releases.

### 排序与内存屏障

Modern CPUs reorder memory accesses. A write in your code may become visible to other CPUs in a different order than it was issued. This is usually invisible because locks on FreeBSD include the necessary barriers. When writing lock-free code, you may need explicit barriers (`atomic_thread_fence_acq`, `atomic_thread_fence_rel`, and variants). For almost all driver code, using a lock removes the need to think about barriers. That is another reason to prefer locks over hand-rolled lock-free constructs when you are still learning.

### 信号与睡眠安全

If your driver sleeps waiting for an event, using `msleep(9)`, `cv_wait(9)`, or `sx_sleep(9)`, use the interruptible variant (`msleep(..., PCATCH)`) when the wait is initiated by user space. Otherwise a stuck device can hold a process in an uninterruptible state forever, and a sufficiently patient attacker can use that to exhaust process slots. The interruptible wait lets the process be signalled.

Always check the return value of a sleep. If it returns a non-zero value, the sleep was interrupted (either by a signal or by another condition), and the driver should typically unwind and return to user space. Don't assume the condition is true just because the sleep returned.

### 速率限制与资源耗尽

A final race-related security concern is resource exhaustion. If an attacker can call your ioctl a million times per second, and each call allocates a kilobyte of kernel memory that is not freed until close, they can drive the system out of memory. This is a denial of service attack, and a careful driver defends against it.

The defenses are: cap per-open resource use, cap global resource use, rate-limit expensive operations. FreeBSD provides `eventratecheck(9)` and `ppsratecheck(9)` in `/usr/src/sys/sys/time.h` for rate limiting, and you can build your own counters where needed. The principle is that the cost to call your driver should not be wildly asymmetric: if a single call allocates megabytes of state, either the caller needs a privilege check or the driver needs a hard cap.

### 基于时代的回收：无锁读取器惯用法

For read-mostly data structures where readers must never block and writers are rare, FreeBSD provides an epoch-based reclamation framework in `/usr/src/sys/sys/epoch.h`. Readers enter an epoch, access the shared data without taking a lock, and exit the epoch. Writers update the data (usually by replacing a pointer) and then wait for all readers currently in an epoch to exit before freeing the old data.

The idiom is useful for driver code that has frequent reads on a hot path and wants to avoid locking overhead there. For example, a network driver that looks up a rule from a routing-table-like structure on every packet may want readers to run lock-free.

```c
epoch_enter(secdev_epoch);
rule = atomic_load_ptr(&sc->sc_rules);
/* use rule; must not outlive the epoch */
do_stuff(rule);
epoch_exit(secdev_epoch);
```

A writer replacing the rule set:

```c
new_rules = build_new_rules();
old_rules = atomic_load_ptr(&sc->sc_rules);
atomic_store_ptr(&sc->sc_rules, new_rules);
epoch_wait(secdev_epoch);
free(old_rules, M_SECDEV);
```

`epoch_wait` blocks until all readers that entered before the store have exited. After it returns, no reader can still be using `old_rules`, so it is safe to free.

The security considerations with epochs are subtle:

A reader inside an epoch may hold a pointer to something that is about to be replaced. The reader must finish using the pointer before exiting the epoch; any use after the exit is a use-after-free.

A reader inside an epoch cannot sleep. The epoch is an asymmetric lock: writers wait on readers, so a reader that sleeps can starve writers indefinitely.

The writer must ensure that the replacement operation is atomic from a reader's perspective. For a single pointer, an atomic store does the job. For more complex updates, two epochs or a read-copy-update sequence may be needed.

Used correctly, epochs give very high performance on read-heavy workloads. Used incorrectly (a reader that sleeps, or a writer that fails to wait), they produce races that are hard to reproduce and hard to diagnose. Beginners should prefer locks until the performance profile justifies the complexity of epoch-based code.

### 第5节总结

竞态和 TOCTOU 漏洞是基于时间的漏洞。它们发生在两个上下文在没有协调的情况下触及共享数据时，或者驱动程序在两个不同时间检查条件并对其采取行动时。防止它们的工具很简单：将用户数据复制到内核一次并从副本工作；在每次访问共享可变状态时使用锁；定义每个锁保护什么并在完整的检查-行动序列中持有它；在释放异步工作触及的结构之前排空异步工作；使用引用计数进行多上下文生命周期管理。

None of this is new to concurrency programming. What is new is the mindset: a race in a driver is not merely a correctness problem. It is a security problem, because an attacker can often arrange the timing they need to exploit it. The next section steps back from timing and looks at a different kind of defense: privilege checks, credentials, and access control.

## 第6节：访问控制与权限强制执行

并非驱动程序暴露的每个操作都应该对每个用户可用。读取温度传感器对每个人都可能没问题。重新编程设备的固件应该需要权限。向存储控制器写入原始字节可能需要更多权限。本节是关于 FreeBSD 驱动程序如何使用内核的凭据和权限机制来决定调用者是否被允许执行其请求的操作。

The tools are `struct ucred`, `priv_check(9)` and `priv_check_cred(9)`, jail-aware checks, securelevel checks, and the broader MAC and Capsicum frameworks.

### 调用者的凭据：struct ucred

Every thread running in the FreeBSD kernel carries a credential, a pointer to a `struct ucred`. The credential records who the thread is running as, which jail they are confined to, which groups they belong to, and other security attributes. From inside a driver, the credential is almost always reached via `td->td_ucred`, where `td` is the `struct thread *` passed to your entry point.

The structure is declared in `/usr/src/sys/sys/ucred.h`. The fields most relevant to drivers are:

`cr_uid`, the effective user ID. Usually what you check to answer "is this root".

`cr_ruid`, the real user ID.

`cr_gid`, the effective group ID.

`cr_prison`, a pointer to the jail the process is in. All processes have one. Unjailed processes belong to `prison0`.

`cr_flags`, a small set of flags including `CRED_FLAG_CAPMODE`, which indicates capability mode (Capsicum).

Do not check `cr_uid == 0` as your privilege gate. That is a common mistake and it is almost always wrong. The correct gate is `priv_check(9)`, which handles jails, securelevel, and MAC policies correctly. Checking `cr_uid` manually bypasses all of that and gives root inside a jail the same power as root on the host, which is not what jails are for.

### priv_check 和 priv_check_cred

The canonical primitive for "may the caller do this privileged thing" is `priv_check(9)`. Its prototype, from `/usr/src/sys/sys/priv.h`:

```c
int priv_check(struct thread *td, int priv);
int priv_check_cred(struct ucred *cred, int priv);
```

`priv_check` operates on the current thread. `priv_check_cred` operates on an arbitrary credential; you use it when the credential to check is not the running thread's, for example when validating an operation on behalf of a file that was opened earlier.

Both return 0 if the privilege is granted and an errno (typically `EPERM`) if not. The driver's pattern is almost always:

```c
error = priv_check(td, PRIV_DRIVER);
if (error != 0)
    return (error);
```

The `priv` argument selects one of several dozen named privileges. The full list lives in `/usr/src/sys/sys/priv.h` and covers areas like filesystem, networking, virtualization, and drivers. For most device drivers, the relevant names are:

`PRIV_DRIVER`, the generic driver privilege. Grants access to operations restricted to administrators.

`PRIV_IO`, raw I/O to hardware. More restrictive than `PRIV_DRIVER`, appropriate for operations that bypass the driver's usual abstractions and talk directly to hardware.

`PRIV_KLD_LOAD`, used by the module loader. You will not typically use this from a driver.

`PRIV_NET_*`, used by network-related operations.

Several dozen more. Read the list in `priv.h` and pick the most specific match for the operation being gated. `PRIV_DRIVER` is a reasonable default when nothing more specific fits.

A real-world example from the kernel: in `/usr/src/sys/dev/mmc/mmcsd.c`, the driver checks `priv_check(td, PRIV_DRIVER)` before allowing certain ioctls that would let a user reprogram the storage controller. In `/usr/src/sys/dev/syscons/syscons.c`, the console driver checks `priv_check(td, PRIV_IO)` before allowing operations that manipulate the hardware directly, since those bypass the normal tty abstraction.

### Jail 感知

FreeBSD jails (jail(8) and jail(9)) partition the system into compartments. Processes inside a jail share the host's kernel but have a restricted view of the system: their own hostname, their own network visibility, their own filesystem root, and reduced privileges. Inside a jail, `priv_check` refuses many privileges that would otherwise be granted to root. This is one of the main reasons to use `priv_check` instead of checking `cr_uid == 0`.

Some operations, however, make no sense inside a jail at all. Reprogramming device firmware, for example, is a host operation. A jailed root user should never be able to do it. For these, add an explicit jail check:

```c
if (jailed(td->td_ucred))
    return (EPERM);
error = priv_check(td, PRIV_DRIVER);
if (error != 0)
    return (error);
```

The `jailed()` macro, defined in `/usr/src/sys/sys/jail.h`, returns true if the credential's prison is anything other than `prison0`. For operations that should never be performed from within a jail, check this first.

For operations that should be allowed inside a jail but with restrictions, use the jail's own fields. `cred->cr_prison->pr_flags` carries per-jail flags; the jail framework also has helpers for checking whether certain capabilities are allowed in the jail. In most driver work you will not go beyond the simple `jailed()` check.

### 安全级别

FreeBSD supports a systemwide setting called securelevel. At securelevel 0, the system behaves normally. At higher securelevels, certain operations are restricted even for root: raw disk writes may be disabled, the system time cannot be set backwards, kernel modules cannot be unloaded, and so on. The rationale is that on a well-secured server, raising the securelevel at boot means an attacker who later gains root cannot disable logging, install a rootkit module, or rewrite core system files.

For drivers, the relevant helpers are declared in `/usr/src/sys/sys/priv.h`:

```c
int securelevel_gt(struct ucred *cr, int level);
int securelevel_ge(struct ucred *cr, int level);
```

Their return values are counterintuitive and worth studying carefully. They return 0 when the securelevel is **not** above or at the threshold (that is, the operation is allowed), and `EPERM` when the securelevel **is** above or at the threshold (the operation should be denied). In other words, the return value is ready to be used directly as an error code.

The usage pattern for a driver that should refuse to modify hardware at securelevel 1 or higher is:

```c
error = securelevel_gt(td->td_ucred, 0);
if (error != 0)
    return (error);
```

Read carefully: this says "return an error if the securelevel is greater than 0". When securelevel is 0 (normal), `securelevel_gt(cred, 0)` returns 0 and the check passes. When securelevel is 1 or higher, it returns `EPERM` and the operation is refused.

Most drivers do not need securelevel checks. They make sense for operations that are potentially system-destabilizing: reprogramming firmware, writing to raw disk sectors, lowering the system clock, and so on.

### 分层检查

A driver that wants to be defense-in-depth can layer these checks:

```c
static int
secdev_reset_hardware(struct secdev_softc *sc, struct thread *td)
{
    int error;

    /* Not inside a jail. */
    if (jailed(td->td_ucred))
        return (EPERM);

    /* Not at elevated securelevel. */
    error = securelevel_gt(td->td_ucred, 0);
    if (error != 0)
        return (error);

    /* Must have driver privilege. */
    error = priv_check(td, PRIV_DRIVER);
    if (error != 0)
        return (error);

    /* Okay, do the dangerous thing. */
    return (secdev_do_reset(sc));
}
```

Each check answers a different question. `jailed()` asks whether we are in the right security domain. `securelevel_gt` asks whether the system administrator has told the kernel to refuse this kind of operation. `priv_check` asks whether this particular thread has the appropriate privilege.

In many drivers, only the `priv_check` is strictly necessary, because it handles jails and securelevel through the MAC framework and the privilege definitions themselves. The explicit `jailed()` and `securelevel_gt` calls are appropriate for operations with known host-wide consequences. When in doubt, start with `priv_check(td, PRIV_DRIVER)` and add more layers only when you can explain what each additional check buys.

### Open、Ioctl 和其他路径上的凭据

When designing privilege checks, think about where in the driver's lifecycle they live. There are two main places:

At open time. If only privileged users should be able to open the device, check privileges in `d_open`. This is simplest and gives per-open enforcement: once a user has opened the device, they are free to do what that device allows. This is the model used, for example, by `/dev/mem`, which is openable only with appropriate privilege.

At operation time. If the device supports multiple operations with different privilege requirements, check each operation independently. A storage controller might allow reading device status to any user, reading SMART data to the owner of the device file, and triggering firmware update only to users with `PRIV_DRIVER`. Each operation has its own gate.

A driver can combine both: a privilege check on open to keep unprivileged users out entirely, and additional checks on specific ioctls for operations that need more.

An open-time check is easy to implement:

```c
static int
secdev_open(struct cdev *dev, int oflags, int devtype, struct thread *td)
{
    int error;

    error = priv_check(td, PRIV_DRIVER);
    if (error != 0)
        return (error);

    /* ... rest of open logic ... */
    return (0);
}
```

An ioctl-time check follows the same pattern; the `struct thread *td` argument is available in every entry point.

### 设备文件权限

Independent of in-driver privilege checks, FreeBSD also applies the usual UNIX permission model to device files themselves. When your driver calls `make_dev_s` or `make_dev_credf` to create a device node, you choose an owner, group, and mode. Those apply at the filesystem level: a user who fails the permission check on the device node never reaches your `d_open`.

The `make_dev_args` structure, declared in `/usr/src/sys/sys/conf.h`, includes `mda_uid`, `mda_gid`, and `mda_mode` fields. The pattern is:

```c
struct make_dev_args args;

make_dev_args_init(&args);
args.mda_devsw = &secdev_cdevsw;
args.mda_uid = UID_ROOT;
args.mda_gid = GID_OPERATOR;
args.mda_mode = 0640;
args.mda_si_drv1 = sc;
error = make_dev_s(&args, &sc->sc_cdev, "secdev");
```

`UID_ROOT` and `GID_OPERATOR` are conventional symbolic names. The mode `0640` means owner can read and write, group can read, others have no access. Choose these thoughtfully. A device that could expose sensitive data or cause hardware damage should not be world-readable or world-writable.

The usual pattern for a privileged device is mode `0600` (root-only) or `0660` (root and a specific group, often `operator` or `wheel`). Mode `0640` is common for devices readable by a trusted group for monitoring purposes. Modes like `0666` (world-writable) are almost never appropriate, even for simple pseudo-devices, unless the device really does nothing that should be restricted.

### Devfs 规则

Even if your driver creates the device node with a conservative mode, the system administrator can change that through devfs rules. A devfs rule can relax or restrict permissions based on device name, jail, and other criteria. Your driver should not assume the mode it set at creation is the mode the device will have at runtime; it should continue to apply its in-kernel checks regardless. The filesystem mode and the in-kernel `priv_check` defend different attackers; keep both.

### MAC 框架

The FreeBSD Mandatory Access Control framework, declared in `/usr/src/sys/security/mac/`, lets policy modules hook into the kernel and make access decisions based on richer labels than UNIX permissions. A MAC policy can, for example, restrict which users can access which devices even if UNIX permissions allow it, or log every use of a sensitive operation.

For driver authors, the point is this: `priv_check` already consults the MAC framework. When you use `priv_check`, you are opting into whatever MAC policies the administrator has configured. If you bypass `priv_check` and roll your own privilege check using `cr_uid`, you bypass MAC as well. That is one more reason to always use `priv_check`.

Writing your own MAC policy module is beyond the scope of this chapter; the MAC framework is a substantial subject and has its own documentation. The key takeaway is simply that MAC exists, `priv_check` honors it, and you should not fight it.

**A brief note on MAC policies shipped with FreeBSD.** The base system includes several MAC policies as loadable modules: `mac_bsdextended(4)` for file-system rule lists, `mac_portacl(4)` for network-port access control, `mac_biba(4)` for Biba integrity policy, `mac_mls(4)` for Multi-Level Security labels, and `mac_partition(4)` for partitioning processes into isolated groups. None of these need to be understood in detail by a driver author; the key point is that your driver, by using `priv_check`, gets their policy decisions for free. An administrator who enables `mac_bsdextended` gets additional filesystem-level restrictions; your driver does not need to know.

**MAC and the device node.** When you create a device with `make_dev_s`, the MAC framework may assign a label to the device node. Policies consult that label when access is attempted. A driver does not interact with labels directly; the framework handles it. But understanding that a label exists explains why, on a MAC-enabled system, access to your device may be refused even when UNIX permissions allow it. That is not a bug; it is MAC doing its job.

### Capsicum 与能力模式

Capsicum, declared in `/usr/src/sys/sys/capsicum.h`, is a capability system bolted onto FreeBSD. A process in capability mode has lost access to most global namespaces (no new file opens, no network with side effects, no arbitrary ioctl, and so on). It can only operate on file descriptors it already holds, and those file descriptors may themselves have limited rights (read only, write only, certain ioctls only, and so on).

Capsicum was introduced to FreeBSD through the work of Robert Watson and collaborators. It sits alongside the traditional UNIX permission model and adds a second, more granular layer. Where UNIX permissions ask "can this user access this resource by name", Capsicum asks "does this process have a capability for this specific object". The two layers work together: the user must have UNIX permission to open the file in the first place, but once the file descriptor exists, Capsicum can further restrict what the holder of the descriptor can do with it.

For a driver, the main Capsicum concern is: some of your ioctls may be inappropriate for a process in capability mode. The helper `IN_CAPABILITY_MODE(td)`, defined in `capsicum.h`, tells you whether the calling thread is in capability mode. A driver can check it and refuse operations that are unsafe:

```c
if (IN_CAPABILITY_MODE(td))
    return (ECAPMODE);
```

This is appropriate for operations with global side effects that a capability-mode process should not have access to. Examples might be an ioctl that reconfigures the global driver state, an ioctl that affects other processes or other file descriptors, or an ioctl that performs an operation that requires querying the global filesystem namespace. If your driver's ioctl needs to touch something that is not already named by the file descriptor it was called on, a capability-mode check is appropriate.

For most driver operations, however, the Capsicum story is simpler: the process that holds the file descriptor was granted the rights it needed when the descriptor was given to it. The driver does not need to re-check those rights; the file-descriptor layer already did. Just make sure your driver supports the normal cap-rights flow (it almost certainly does by default) and consider which individual ioctls should be marked with `CAP_IOCTL_*` rights at the VFS layer.

**Cap rights at ioctl granularity.** FreeBSD allows a file descriptor to be restricted to a specific subset of ioctls via `cap_ioctls_limit(2)`. For example, a process can hold a file descriptor that allows `FIOASYNC` and `FIONBIO` but no other ioctls. The restriction is enforced by the VFS layer, not by your driver, but the set of ioctls you expose is what defines what can be selected for restriction. A driver that implements only meaningful, well-documented ioctls makes it easier for consumers to apply sensible cap-ioctl restrictions.

**Examining Capsicum usage in the tree.** For real-world examples of Capsicum-aware code, look at `/usr/src/sys/net/if_tuntap.c` alongside the core capability files under `/usr/src/sys/kern/sys_capability.c`. Most individual drivers rely on the VFS layer to enforce `caprights`, and only add an explicit `IN_CAPABILITY_MODE(td)` check on the handful of operations with global side effects. The pattern is consistent: preserve the normal behavior, add an `IN_CAPABILITY_MODE` check where operations would be unsafe, and document which ioctls are sandbox-safe.

### 带安全标志的 Sysctl

Many drivers expose tunables and statistics through sysctls. A sysctl that exposes sensitive information, or that can be set to change driver behaviour, should use appropriate flags. From `/usr/src/sys/sys/sysctl.h`:

`CTLFLAG_SECURE` (value `0x08000000`) asks the sysctl framework to consult `priv_check(PRIV_SYSCTL_SECURE)` before allowing the operation. It is useful for sysctls that should not be changed at elevated securelevel.

`CTLFLAG_PRISON` allows the sysctl to be visible and writable from inside a jail (rarely wanted for drivers).

`CTLFLAG_CAPRD` and `CTLFLAG_CAPWR` allow the sysctl to be read or written from capability mode. By default, sysctls are inaccessible in capability mode.

`CTLFLAG_TUN` makes the sysctl settable as a loader tunable (from `/boot/loader.conf`).

`CTLFLAG_RD` vs `CTLFLAG_RW` determines read-only vs read-write access; prefer `CTLFLAG_RD` for anything that exposes state, and be deliberate about what you make writable.

A sysctl that exposes a driver-internal buffer for debugging should typically be `CTLFLAG_RD | CTLFLAG_SECURE` at minimum, and possibly not exist at all in production builds.

### 完整的权限门控 Ioctl

Putting the pieces together, here is what a privilege-gated ioctl looks like, end to end:

```c
static int
secdev_ioctl(struct cdev *dev, u_long cmd, caddr_t data, int fflag,
    struct thread *td)
{
    struct secdev_softc *sc = dev->si_drv1;
    int error;

    switch (cmd) {
    case SECDEV_GET_STATUS:
        /* Anyone with the device open can do this. */
        error = secdev_get_status(sc, (struct secdev_status *)data);
        break;

    case SECDEV_RESET:
        /* Resetting is privileged, jail-restricted, and securelevel-sensitive. */
        if (jailed(td->td_ucred)) {
            error = EPERM;
            break;
        }
        error = securelevel_gt(td->td_ucred, 0);
        if (error != 0)
            break;
        error = priv_check(td, PRIV_DRIVER);
        if (error != 0)
            break;
        error = secdev_do_reset(sc);
        break;

    default:
        error = ENOTTY;
        break;
    }

    return (error);
}
```

Different commands get different gates. The status command is unprivileged, since it just reads state. The reset command is the danger case, and it goes through the full layered check.

### 第6节总结

FreeBSD 驱动程序中的访问控制是几个层之间的协作。设备节点上的文件系统权限决定谁可以打开它。`priv_check(9)` 函数族决定线程是否可以执行给定的特权操作。Jail 检查决定操作在调用者的安全域中是否有意义。安全级别检查决定系统管理员是否完全允许此类操作。MAC 框架让策略模块在上面添加自己的意见。Capsicum 权限限制受能力限制的进程可以做什么。

The correct use of these tools comes down to a short list of rules: check the caller's credentials at the right points, prefer `priv_check` over ad-hoc UID checks, add `jailed()` and `securelevel_gt` when the operation has host-wide consequences, pick the most specific `PRIV_*` constant that fits the operation, and set conservative device-file modes in `make_dev_s`.

The next section looks at a different kind of leak: not a privilege escape, but an information escape. Even operations that are properly gated can inadvertently reveal kernel memory contents if they are not written carefully.

## 第7节：防止信息泄露

当不应该对用户空间可见的内核内存仍然被复制到用户空间时，就会发生信息泄露。经典形式是在未首先初始化结构的情况下将结构内容返回给用户。字段之间的任何填充字节或最后一个字段之后的尾随字节包含上次使用该内存时内核栈上或新分配页中的任何内容。那可能是密码、有助于绕过 ASLR 的指针、加密密钥或其他任何东西。

信息泄露有时被忽视为"没那么严重"。它们确实严重。在现代攻击链中，信息泄露通常是第一步：它使内核的地址空间布局随机化（kASLR）失效，并使其他利用变得可靠。一个以"只是泄露几个字节"开始的漏洞类别通常以"攻击者获得内核代码执行"结束。

### 信息泄露是如何发生的

There are three main ways a driver leaks information:

**Uninitialized structure fields copied to user space.** A structure has N defined fields plus padding and alignment slots. The code fills in the N fields and calls `copyout`. The padding goes along for the ride, carrying whatever uninitialized stack memory happened to be there.

**Partially initialized buffers.** The driver allocates a buffer, fills in some of it, and copies the whole buffer to user space. The uninitialized tail carries heap contents.

**Oversized replies.** The driver is asked for `N` bytes, but returns a buffer of size `M > N`. The extra `M - N` bytes contain whatever was in the tail of the source buffer.

**Reading beyond a NUL.** For string data, the driver copies a buffer up to its allocated size instead of up to the NUL terminator. The bytes after the NUL can carry any data that happened to be in that buffer earlier.

Each of these is easy to create by accident and easy to prevent once you know the pattern.

### 填充问题

Consider this structure:

```c
struct secdev_info {
    uint32_t version;
    uint64_t flags;
    uint16_t id;
    char name[32];
};
```

On a 64-bit system, the compiler inserts padding to align `flags` to 8 bytes. Between `version` (4 bytes) and `flags` (8 bytes), there are 4 bytes of padding. After `id` (2 bytes) and before `name` (1-byte alignment), there are 6 more bytes of padding at the end if the structure is sized up to a multiple of 8.

If your code does:

```c
struct secdev_info info;

info.version = 1;
info.flags = 0x12345678;
info.id = 42;
strncpy(info.name, "secdev0", sizeof(info.name));

error = copyout(&info, args->buf, sizeof(info));
```

then the padding bytes, which you never set, go out to user space. They contain whatever stack memory happened to be at those positions when the function was entered. That is an information leak.

The fix is universal and cheap: zero the structure first.

```c
struct secdev_info info;

bzero(&info, sizeof(info));      /* or memset(&info, 0, sizeof(info)) */
info.version = 1;
info.flags = 0x12345678;
info.id = 42;
strncpy(info.name, "secdev0", sizeof(info.name));

error = copyout(&info, args->buf, sizeof(info));
```

Now the padding is zero, as is any field you forgot to set. The cost is one call to `bzero`; the benefit is that your driver cannot leak kernel memory through this structure, no matter what fields are added later. Always zero structures before copyout.

An equivalent pattern using designated initializers works when you are declaring and initializing in one step:

```c
struct secdev_info info = { 0 };  /* or { } in some standards */
info.version = 1;
/* ... */
```

The `= { 0 }` zeros all bytes including padding. Combine this with setting the specific fields afterwards, and you have a clean pattern.

### 堆分配情况

When you allocate a buffer with `malloc(9)` and fill it before returning to user space, you have the same issue. Always use `M_ZERO` to zero-initialize, or explicitly zero the buffer before writing to it:

```c
buf = malloc(size, M_SECDEV, M_WAITOK | M_ZERO);
```

Even if you intend to fill every byte, using `M_ZERO` is cheap insurance: if a bug causes a partial fill, the unfilled bytes are zero rather than stale heap contents.

### 过大回复

A subtle form of leak happens when the driver returns more data than the user asked for. Imagine an ioctl that returns a list of items:

```c
/* User asks for up to user_len bytes of list data. */
if (user_len > sc->sc_list_bytes)
    user_len = sc->sc_list_bytes;

error = copyout(sc->sc_list, args->buf, sc->sc_list_bytes);  /* BUG: wrong length */
```

The driver copies `sc_list_bytes` bytes regardless of what the user asked for. If `sc_list_bytes > user_len`, the driver writes past `args->buf`, which is a different bug (buffer overflow in user space). If the driver is writing to a local buffer first and then copying out, a similar error would write past the local buffer.

The correct pattern is to clamp the length and use the clamped length for the copy:

```c
size_t to_copy = MIN(user_len, sc->sc_list_bytes);
error = copyout(sc->sc_list, args->buf, to_copy);
```

Information leaks through oversized replies are common when driver code evolves: the original author wrote a paired check-and-copy; a later change altered one side but not the other. Every copyout should use the already-validated kernel-side length, and that length should be bounded by the user's buffer size.

### 字符串与 NUL 终止符

Strings are a particularly rich source of information leaks because they have two different natural lengths: the length of the string (up to the NUL) and the size of the buffer it lives in. Suppose:

```c
char name[32];
strncpy(name, "secdev0", sizeof(name));  /* copies 8 bytes, NUL-padded */

/* ... later, maybe in a different function ... */
strncpy(name, "xdev", sizeof(name));     /* copies 5 bytes, NUL-padded */

copyout(name, args->buf, sizeof(name));  /* copies all 32 bytes */
```

The second `strncpy` overwrites the first five bytes with "xdev\0" and then pads the rest of the buffer with NULs. That happens to be safe because `strncpy` pads with NULs when the source is shorter than the destination. But if the buffer came from `malloc(9)` without `M_ZERO`, or from a stack buffer that was written to by earlier code, bytes after the NUL may contain stale data. Copying the full buffer then leaks it.

The safe pattern is to copy only up to the NUL, or to zero the buffer before writing:

```c
bzero(name, sizeof(name));
snprintf(name, sizeof(name), "%s", "secdev0");
copyout(name, args->buf, strlen(name) + 1);
```

`snprintf` guarantees NUL termination. Zeroing first ensures the bytes after the NUL are zero. The `+ 1` in the copy length includes the NUL itself.

Alternatively, copy only the string and let user space deal with its own padding:

```c
copyout(name, args->buf, strlen(name) + 1);
```

The cleanest pattern is to zero first and copy exactly the valid length.

### 敏感数据：释放前显式清零

When a driver allocates memory to hold sensitive data (cryptographic keys, user credentials, proprietary secrets), the memory should be zeroed explicitly before being freed. Otherwise the freed memory returns to the kernel allocator's free pool with the data still visible, and subsequent allocations from that pool may expose it.

FreeBSD provides `explicit_bzero(9)`, declared in `/usr/src/sys/sys/systm.h`, which zeroes memory in a way that the compiler cannot optimize away:

```c
explicit_bzero(sc->sc_secret, sc->sc_secret_len);
free(sc->sc_secret, M_SECDEV);
sc->sc_secret = NULL;
```

Ordinary `bzero` can be eliminated by the compiler if the data is not read after being zeroed, which is exactly the situation before a free. `explicit_bzero` is guaranteed to perform the zeroing. Use it whenever sensitive data is about to be freed or go out of scope.

There is also `zfree(9)`, declared in `/usr/src/sys/sys/malloc.h`, which zeroes and frees in one call:

```c
zfree(sc->sc_secret, M_SECDEV);
sc->sc_secret = NULL;
```

`zfree` knows the allocation size from the allocator metadata and zeroes that many bytes before freeing. This is usually the cleanest pattern for cryptographic material.

For UMA zones, the equivalent is that the zone itself can be asked to zero on free, or you can `explicit_bzero` the object before calling `uma_zfree`. For stack buffers with sensitive content, `explicit_bzero` at the end of the function is the right tool.

### 永远不要泄露内核指针

One specific form of information leak is returning a kernel pointer to user space. The kernel address of a softc, or of an internal buffer, is useful information to an attacker trying to exploit another bug. `printf("%p")` in log messages can also leak addresses. The general rule: do not put kernel addresses in user-visible output.

For sysctls and ioctls, the simplest rule is that no field in a user-facing structure should be a raw kernel pointer. If the driver wants to expose an identifier for a kernel object, use a small integer ID (an index into a table, for example), not the address of the object. Convert from one to the other inside the driver, never expose the raw pointer.

FreeBSD's `printf(9)` supports the `%p` format, which does print a pointer, but log messages in production drivers should avoid `%p` for anything where the pointer could aid exploitation. For debugging, `%p` is fine during development; before shipping the driver, audit `printf` and `log` calls to ensure no `%p` remains in paths accessible from user space.

### Sysctl 输出

Sysctls that expose structures have the same rules as ioctls. Zero the structure before filling it, clamp the output length to the caller's buffer, and avoid pointer leaks. The `sysctl_handle_opaque` helper is often used for raw structures; make sure the structure is fully initialized before the handle returns.

A safer pattern is to expose each field as its own sysctl, using `sysctl_handle_int`, `sysctl_handle_string`, and so on. This avoids the padding problem entirely because each value is copied out as a primitive. It is also more ergonomic for users: `sysctl secdev.stats.packets` is more useful than an opaque blob they have to decode.

### copyout 错误

`copyout` can fail. If the user buffer becomes unmapped between the validation and the copy, `copyout` returns `EFAULT`. Your driver must handle this cleanly: typically, return the error to the user, and make sure any partial success is rolled back.

A sequence like "allocate state, fill output buffer, copyout, commit state" is safer than "commit state, copyout". If the copyout fails in the second pattern, the state is committed but the user never learned what happened. If it fails in the first pattern, nothing was committed, and the user gets a clean error.

### 故意披露

Some sysctls and ioctls are explicitly designed to reveal information that would otherwise be private. These need an especially careful threat model. Ask: who is allowed to call this? What do they learn? Could a less-trusted attacker who obtains that information use it for something worse? A dmesg-style sysctl that exposes recent kernel messages is fine, but only because it has been scoped and filtered; exposing raw kernel log buffers without scoping is very different.

When in doubt, a sysctl that reveals sensitive data should be gated with `CTLFLAG_SECURE`, restricted to privileged users, and exposed only through paths that users must explicitly opt into. Default to less disclosure rather than more.

### 内核指针哈希

Sometimes a driver legitimately needs to expose something that identifies a kernel object, for debugging or for correlating events. The raw pointer address is the wrong answer for the reasons discussed. A better answer is a hashed or masked representation that identifies the object without revealing its address.

FreeBSD provides `%p` in `printf(9)`, which prints a pointer. It also provides a related mechanism where pointers can be "obfuscated" in user-visible output using a per-boot secret, so that two pointers in the same output are consistently distinguishable but their absolute values are not leaked. The support for this varies across subsystems; when designing your own output, consider whether a dense integer ID (an index into a table) is sufficient. Often it is.

For logs, `%p` is fine during development when logs are private. Before shipping, replace any `%p` in paths reachable from user space with either a debug-only guard (so the format is present only in debug builds) or with a non-pointer identifier.

### 第7节总结

Information leaks are the quieter cousin of buffer overflows: they do not crash, they do not corrupt, they merely send data to user space that should have stayed in the kernel. The tools to prevent them are simple and cheap. Zero structures before filling them. Use `M_ZERO` on heap allocations that will be copied to user space. Clamp copy lengths to the smaller of the caller's buffer and the kernel's source buffer. Use `explicit_bzero` or `zfree` for sensitive data before freeing. Keep kernel pointers out of user-visible output. Bound strings to their actual length, not their buffer size.

A driver that applies these habits consistently will not leak information through its interfaces. The next section moves to the debugging and diagnostics side: how to log without leaking, how to debug without leaving production-hostile code behind, and how to keep the operator informed without handing an attacker a map.

## 第8节：安全的日志记录与调试

每个驱动程序都会记录日志。`printf(9)` 和 `log(9)` 是驱动程序作者首先使用的工具之一，这是有充分理由的：一个放置得当的日志消息将神秘的失败转化为可读的叙述。但日志不是免费的。它们消耗磁盘空间，可能被淹没，并且可能泄露敏感数据。安全意识的驱动程序将日志记录视为一等设计关注，而不是调试的事后想法。

This section is about writing log messages that help operators without hurting security.

### 日志记录原语

FreeBSD drivers have two main ways to emit messages.

`printf(9)`, the same name as the C library function but with kernel-specific semantics, writes to the kernel message buffer and, if the console is active, to the console. It is unconditional: every `printf` call results in a message.

`log(9)`, declared in `/usr/src/sys/sys/syslog.h`, writes to the kernel log ring with a syslog-compatible priority. Messages go to the in-kernel log buffer (readable by `dmesg(8)`) and, via `syslogd(8)`, to the configured log destinations. The priority is the familiar syslog scale: `LOG_EMERG`, `LOG_ALERT`, `LOG_CRIT`, `LOG_ERR`, `LOG_WARNING`, `LOG_NOTICE`, `LOG_INFO`, `LOG_DEBUG`.

Use `log(9)` when you want the message to be filtered or routed by syslog. Use `printf(9)` when you want unconditional emission, typically for very important events or for output that should always appear on the console.

`device_printf(9)` is a small wrapper over `printf` that prefixes the message with the device name (`secdev0: ...`). Prefer it inside driver code so messages are easy to attribute.

### 该记录什么和不该记录什么

A security-aware driver logs:

**State transitions that matter.** Attach, detach, reset, firmware update, link up, link down. These let an operator correlate driver behaviour with system events.

**Errors from the hardware or from user requests.** A bad ioctl argument, a DMA error, a timeout, a CRC mismatch. These let the operator diagnose problems.

**Rate-limited summaries of anomalous events.** If a malformed ioctl is received a million times per second, log the first, summarize the rest.

A security-aware driver does not log:

**User data.** The contents of buffers the user passed in. You never know what is in them.

**Cryptographic material.** Keys, IVs, plaintext, ciphertext. Ever.

**Sensitive hardware state.** On a security device, some register contents are themselves secrets.

**Kernel addresses.** `%p` is fine in early development; it has no place in production logs.

**Details of authentication failures.** A log message that says "user jane failed check X because register was 0x5d" tells an attacker what check to defeat. A log that says "authentication failed" tells the operator there was a failure without tutoring the attacker.

Think about who reads the logs. On a multi-tenant server, other users may have log-reading privileges. On a shipped appliance, the log may be exported for remote support. Treat log messages as information that could end up on any surface the system touches.

### 速率限制

A noisy driver is a security problem. If an attacker can trigger a log message, they can trigger a million of them. Log flooding consumes disk space, slows the system, and buries legitimate messages. FreeBSD provides `eventratecheck(9)` and `ppsratecheck(9)` in `/usr/src/sys/sys/time.h`:

```c
int eventratecheck(struct timeval *lasttime, int *cur_pps, int max_pps);
int ppsratecheck(struct timeval *lasttime, int *cur_pps, int max_pps);
```

Both return 1 if the event is allowed through and 0 if it has been rate-limited. `lasttime` and `cur_pps` are per-call state you keep in your softc. `max_pps` is the limit in events per second.

Pattern:

```c
static struct timeval secdev_last_log;
static int secdev_cur_pps;

if (ppsratecheck(&secdev_last_log, &secdev_cur_pps, 5)) {
    device_printf(dev, "malformed ioctl from uid %u\n",
        td->td_ucred->cr_uid);
}
```

Now, no matter how many malformed ioctls the attacker sends, the driver emits at most 5 log messages per second. That is enough for the operator to notice something is happening without drowning the system.

Per-event rate limiting (one `lasttime`/`cur_pps` pair per event type) is better than a single global limit, because it prevents a flood of one event type from masking other events.

### 实践中的日志级别

A good rule of thumb is this:

`LOG_ERR` for unexpected driver failures that require operator attention. "DMA mapping failed", "device returned CRC error", "firmware update aborted".

`LOG_WARNING` for unusual but not necessarily critical situations. "Received oversized buffer, truncating", "falling back to polled mode".

`LOG_NOTICE` for events that are normal but worth recording. "Firmware version 2.1 loaded", "device attached".

`LOG_INFO` for high-volume status information that operators may filter.

`LOG_DEBUG` for debugging output. A production driver usually does not emit `LOG_DEBUG` unless the operator has enabled debug logging via a sysctl.

`LOG_EMERG` and `LOG_ALERT` are reserved for system-threatening conditions and are not typically emitted by device drivers.

Choosing the right level matters because operators configure syslog to filter by level. A driver that logs every received packet at `LOG_ERR` makes the logs useless.

### 调试日志与生产环境

During development, you will want verbose logging: every state transition, every entry and exit, every buffer allocation. That is fine. The question is how to turn it off in production without losing the ability to re-enable it when there is a bug to diagnose.

Two patterns are common:

**A sysctl-controlled debug level.** The driver reads a sysctl at the top of each log-worthy event and emits or suppresses the message based on the level. This allows runtime control without recompiling.

```c
static int secdev_debug = 0;
SYSCTL_INT(_hw_secdev, OID_AUTO, debug, CTLFLAG_RW,
    &secdev_debug, 0, "debug level");

#define SECDEV_DBG(level, fmt, ...) do {                    \
    if (secdev_debug >= (level))                            \
        device_printf(sc->sc_dev, fmt, ##__VA_ARGS__);      \
} while (0)
```

**Compile-time control.** A driver can use `#ifdef SECDEV_DEBUG` to include or exclude debug blocks. This is faster (no runtime check) but requires a rebuild to change. Often the two are combined: `#ifdef SECDEV_DEBUG` wraps the infrastructure, and the sysctl controls verbosity within that.

Either way, avoid `printf` calls in hot paths that are not guarded by some kind of conditional. An uncommented `printf` in an interrupt handler or a per-packet path is a performance disaster waiting to be enabled.

### 不留痕迹

Before committing driver changes, grep the driver for:

Raw `printf` calls without `device_printf` prefixes. These make log attribution harder.

`%p` format specifiers. If they appear in paths reachable from user space, replace with less sensitive formats (a sequence number, a hash, nothing).

`LOG_ERR` on user-triggerable events without rate limiting. Attackers can weaponize these.

`TODO`, `XXX`, `FIXME`, `HACK` near security-related code. Leaving these for reviewers is fine; shipping them is not.

Test-only fprintf-equivalents that were supposed to be removed.

### dmesg 与内核消息缓冲区

The kernel message buffer is a fixed-size ring buffer shared by every driver and the kernel itself. On a busy system, old messages scroll out as new ones arrive. A driver that floods the buffer pushes out useful messages from other drivers.

`dmesg(8)` shows the current contents of the buffer. Operators rely on it. Being a good citizen in the buffer means: log important things, do not log in hot paths, rate-limit everything triggerable by users, and do not flood.

The buffer size is tunable (`kern.msgbufsize` sysctl), but you cannot count on a particular size. Write as if every message is valuable and must compete with others for space.

### KTR 与追踪

For detailed tracing without the cost of `printf`, FreeBSD provides KTR (Kernel Tracing), declared in `/usr/src/sys/sys/ktr.h`. KTR macros, when enabled, record events in a compact in-kernel ring that is separate from the message buffer. A kernel compiled with `options KTR` can be queried with `sysctl debug.ktr.buf` and with `ktrdump(8)`.

KTR events are best for per-operation tracing where a `printf` would be too heavy. They are almost free at runtime when disabled. For a security-sensitive driver, KTR gives you a way to leave tracing infrastructure in the code without paying for it in production.

Other tracing frameworks (dtrace(1) via SDT probes) are worth learning for deep inspection. They are out of scope for this chapter, but know that they exist.

### 记录特权操作

A specific case worth calling out: when your driver successfully performs a privileged operation, log it. This creates an audit trail. If a firmware update happens, log who triggered it. If a hardware reset is issued, log it. If a device is reconfigured, log the change.

```c
log(LOG_NOTICE, "secdev: firmware update initiated by uid %u (euid %u)\n",
    td->td_ucred->cr_ruid, td->td_ucred->cr_uid);
```

The operator can later see who did what. If there is ever a security incident, this log is the first evidence. Make it accurate and make it hard to forge.

Do not over-log legitimate privileged use; a firmware update triggered by `freebsd-update` once a month is one message, not a thousand. But the single message should carry enough detail to reconstruct what happened: who, when, what, with what arguments.

### audit(4) 框架

For deeper audit trails than `log(9)` provides, FreeBSD includes an audit subsystem (`audit(4)`) based on the BSM (Basic Security Module) audit format originally from Solaris. When enabled via `auditd(8)`, the kernel emits structured audit records for many security-relevant events: logins, privilege changes, file access, and, increasingly, driver-specific events when drivers instrument themselves.

A driver that handles highly sensitive operations can emit custom audit records using `AUDIT_KERNEL_*` macros declared in `/usr/src/sys/security/audit/audit.h`. This is more involved than a `log(9)` call, but it produces records that fit into the operator's existing audit workflow, are structured (machine-readable), and can be forwarded to remote audit collectors for compliance.

For most drivers, `log(9)` with `LOG_NOTICE` and a clear message is enough. For drivers that must meet specific compliance requirements (government, financial, medical), consider investing in audit integration. The infrastructure is already in the kernel; you just need to call into it.

### 在驱动程序中使用 dtrace

Alongside logging, `dtrace(1)` lets an operator observe driver behavior without recompiling. A driver that declares Statically Defined Trace (SDT) probes through `sys/sdt.h` exposes well-defined hook points that dtrace scripts can latch onto.

```c
#include <sys/sdt.h>

SDT_PROVIDER_DECLARE(secdev);
SDT_PROBE_DEFINE2(secdev, , , ioctl_called, "u_long", "int");

static int
secdev_ioctl(struct cdev *dev, u_long cmd, caddr_t data, int fflag,
    struct thread *td)
{
    SDT_PROBE2(secdev, , , ioctl_called, cmd, td->td_ucred->cr_uid);
    /* ... */
}
```

An operator can then write a dtrace script that fires on `secdev:::ioctl_called` and counts or logs each event. The advantage over `log(9)` is that dtrace probes have essentially no cost when disabled, and they let the operator decide what to observe rather than forcing the driver author to anticipate every useful question.

For a security-focused driver, SDT probes on entry and exit of privileged operations let security monitoring tools observe usage patterns without the driver having to log every call. This is useful for anomaly detection: a sudden spike in ioctl calls from an unexpected UID, for example, can be flagged by a dtrace-based monitor.

### 第8节总结

Logging is how a driver talks to its operator. Like any communication, it can be clear or confused, honest or misleading, helpful or harmful. A security-aware driver logs important events with appropriate levels, avoids logging sensitive data, rate-limits anything an attacker can trigger, and uses debug infrastructure that can be turned on and off without recompilation. It prefers `device_printf(9)` for attribution, uses `log(9)` with thoughtful priorities, and never leaves `%p` or unguarded `printf` statements in production paths.

The next section takes a broader view. Beyond specific techniques (bounds-checking, privilege checks, safe logging), there is a design-level question: what should a driver do by default when something goes wrong? What fail-safe behavior should it exhibit? That is the subject of secure defaults.

## 第9节：安全默认值与故障安全设计

驱动程序的设计决策在任何单行代码编写之前就塑造了其安全性。两个驱动程序可以使用相同的 API、相同的分配器、相同的加锁原语，最终得到非常不同的安全态势，因为一个被设计为开放的，另一个被设计为封闭的。本节是关于使驱动程序默认安全的设计选择。

中心思想总结为一个原则：有疑问时，拒绝。故障时开放的驱动程序必须在每个分支中都是正确的才能安全。故障时关闭的驱动程序只需在它决定允许某事的狭窄路径中正确即可。

### 故障时关闭

The first and most important design decision is what happens when your code reaches a state it did not expect. Consider a switch statement:

```c
switch (op) {
case OP_FOO:
    return (do_foo());
case OP_BAR:
    return (do_bar());
}
return (0);   /* fall-through: everything else succeeds! */
```

This is a fail-open design. Any operation code that is not `OP_FOO` or `OP_BAR` succeeds silently, returning 0. That is almost never what you want. A new operation code added to the API but not handled in the driver becomes a silent no-op. An attacker who discovers this can use it to bypass checks.

The fail-closed version:

```c
switch (op) {
case OP_FOO:
    return (do_foo());
case OP_BAR:
    return (do_bar());
default:
    return (EINVAL);
}
```

Unknown operations explicitly return an error. If a new operation is added to the API, the compiler or the tests will tell you the moment you handle it and forget to update the switch, because the new case is needed to silence the `EINVAL`.

The same principle applies at every decision point. When a function checks a precondition:

```c
/* Fail open: if the check is inconclusive, allow the operation. */
if (bad_condition == true)
    return (EPERM);
return (0);

/* Fail closed: if the check is inconclusive, refuse. */
if (good_condition != true)
    return (EPERM);
return (0);
```

The second form fails closed: if the precondition cannot be proven good, the operation is refused. This is safer when `good_condition` has any chance of being false due to an error in setup, a race, or a bug.

### 白名单，不要黑名单

Closely related: when deciding what is allowed, whitelist the known-good rather than blacklisting the known-bad. Blacklists are always incomplete, because you cannot enumerate every bad input. Whitelists are finite by construction.

```c
/* Bad: blacklist */
if (c == '\n' || c == '\r' || c == '\0')
    return (EINVAL);

/* Good: whitelist */
if (!isalnum(c) && c != '-' && c != '_')
    return (EINVAL);
```

The blacklist missed `\t`, `\x7f`, every high-bit character, and so on. The whitelist made the allowed set explicit and refused everything else.

This applies to input validation generally. A driver that accepts a set of configuration names should explicitly list them. A driver that accepts a set of operation codes should enumerate them. If a user sends something that is not on the list, refuse.

### 最小有用接口

A driver exposes functionality to user space through device nodes, ioctls, sysctls, and sometimes network protocols. Every exposed entry is a potential attack surface. A secure driver exposes only what users actually need.

Before shipping an ioctl, ask: does anyone actually use this? If a debugging ioctl was useful during development but has no production role, remove it or compile it out behind a debug flag. If a sysctl exposes internal state that only matters for engineering, hide it behind `CTLFLAG_SECURE` and consider removing it.

The cost of removing an interface now is small: a few lines of code. The cost later, when the interface has shipped and has users, is much larger. Smaller interfaces are easier to review, easier to test, and have fewer opportunities for bugs.

### 打开时的最低权限

A device node can be created with restrictive or permissive modes. Start restrictive. A mode of `0600` or `0640` is almost always a better default than `0666`. If users complain that they cannot access the device, that is a conversation you want to have; you can always relax the mode, and the operator can use devfs rules to do so per-site. If users silently gain access they should not have, you will not have that conversation until something breaks.

Similarly, a driver that supports jails should default to not being accessible in jails unless there is a specific reason. The reasoning is the same: it is easier to open up later than to retrofit a closed policy onto an open one.

### 保守的默认值

Every configurable parameter has a default. Choose conservative ones.

A driver that has a configurable "allow user X to do Y" tunable should default to X = none. If an operator wants to grant access, they can change the tunable. If the default granted access, every deployment that missed the tunable would be open.

A driver that has a timeout should default to a short timeout. If the operation usually finishes quickly, a short default is fine. If it sometimes takes longer, the operator can bump the timeout. A long default is a denial-of-service opportunity.

A driver that has a buffer size limit should default to a small limit. Again, operators can raise it; attackers cannot.

### 纵深防御

No single security mechanism is perfect. A defense-in-depth driver assumes any one layer can fail and builds multiple layers.

Example: suppose a driver accepts an ioctl that requires privilege. The layers of defense are:

The device node mode blocks unprivileged users from opening the device at all.

A `priv_check` at open time blocks unprivileged users even if the mode is misconfigured.

A `priv_check` on the specific ioctl catches the case where an unprivileged user somehow reached the ioctl handler.

A `jailed()` check on the ioctl blocks jailed users.

Input validation on the ioctl arguments refuses malformed requests.

A rate-limit log records repeated malformed requests.

If all five are present, a failure in any one is contained by the others. If only one is present and it fails, the driver is compromised. Defense in depth costs a little more code and a little more CPU; it buys real resilience.

### 超时与看门狗

A driver that waits on external events should have timeouts. Hardware can fail to respond. User space can stop reading. Networks can stall. Without a timeout, a waiting driver can hold resources forever, and an attacker who controls the external event can deny service by simply not responding.

`msleep(9)` accepts a timeout argument in ticks. Use it. A sleep with no timeout is rarely the right answer in driver code.

For longer-lived operations, a watchdog timer can detect that an operation has stalled and take recovery action: abort, retry, or reset. The `callout(9)` framework is the usual mechanism.

### 有界资源使用

Every resource a driver can allocate on behalf of a caller should have a cap. Buffer sizes have maximum values. Per-open resource counts have maximum values. Global resource counts have maximum values. When a cap is hit, the driver returns an error, not an attempt at "best effort".

Without caps, a misbehaving or hostile process can exhaust resources. The exhaustion might be memory, file-descriptor-like state, interrupt-worthy events, or simply CPU time. Caps ensure that no single caller can dominate.

A reasonable default structure:

```c
#define SECDEV_MAX_BUFLEN     (1 << 20)   /* per buffer */
#define SECDEV_MAX_OPEN_BUFS  16          /* per open */
#define SECDEV_MAX_GLOBAL     256         /* driver-wide */
```

Check each cap explicitly before allocating. Return `EINVAL`, `ENOMEM`, or `ENOBUFS` as appropriate when the cap is hit.

### 安全的模块加载与卸载

A driver that supports being unloaded must handle cleanup correctly. An unsafe unload is a security bug. If unload leaves a callback registered, or a mapping in place, or a DMA in flight, then re-loading the module (or unloading and resuming) can touch memory that is no longer owned by the driver. That is a use-after-free waiting to happen.

The rule: if any part of `detach` or `unload` fails, either propagate the error (and keep the driver loaded) or drive the cleanup to completion. Partial teardown is worse than no teardown.

A reasonable strategy: make the unload path paranoid. It checks every resource and tears down every one that was allocated, in reverse order of allocation. It uses the `callout_drain` and `taskqueue_drain` helpers to wait for async work. Only after every such resource is quiet does it free the softc.

If any step fails, return `EBUSY` from `detach` and document that the driver cannot currently be unloaded. That is better than half-freeing and crashing later.

### 安全的并发入口

A driver's entry points (open, close, read, write, ioctl) can be called concurrently. The driver should be written as if every entry point could be called from any context at any time. Anything else is a race waiting to fire.

The practical implication: every entry point that touches shared state acquires the softc lock first. Every operation that uses resources from the softc does so under the lock. If the operation has to sleep or do user-space work, the code drops the lock, does the work, and re-acquires carefully, checking that the state it had not changed under its feet.

Concurrency is not an afterthought. It is part of the interface.

### 错误路径就是正常路径

A subtle aspect of secure design is that error paths get the same care as success paths. In a driver, error paths often free resources, release locks, and restore state. A bug on an error path is just as exploitable as a bug on a success path; often more so, because error paths are less tested.

Write every error path as if it were the happy path for a user who is trying to find bugs. Every `goto cleanup` or `out:` label is a candidate for a double-free, a missed unlock, or a left-behind mapping. Walk each error path mentally and confirm that:

Every resource allocated on the success path is freed on the error path.

No resource is freed twice.

Every lock held is released exactly once.

No error path leaves partially initialized state visible to other contexts.

A systematic pattern helps. The "single cleanup path" idiom (one label, cleanup proceeds in reverse order of allocation) catches most such bugs by construction:

```c
static int
secdev_do_something(struct secdev_softc *sc, struct secdev_arg *arg)
{
    void *kbuf = NULL;
    struct secdev_item *item = NULL;
    int error;

    kbuf = malloc(arg->len, M_SECDEV, M_WAITOK | M_ZERO);

    error = copyin(arg->data, kbuf, arg->len);
    if (error != 0)
        goto done;

    item = uma_zalloc(sc->sc_zone, M_WAITOK | M_ZERO);

    error = secdev_process(sc, kbuf, arg->len, item);
    if (error != 0)
        goto done;

    mtx_lock(&sc->sc_mtx);
    LIST_INSERT_HEAD(&sc->sc_items, item, link);
    mtx_unlock(&sc->sc_mtx);
    item = NULL;  /* ownership transferred */

done:
    if (item != NULL)
        uma_zfree(sc->sc_zone, item);
    free(kbuf, M_SECDEV);
    return (error);
}
```

Each allocation is paired with a cleanup at `done`. The cleanup uses `NULL` checks so that resources freed earlier (or never allocated) do not cause double-frees. Ownership transfers set the pointer to `NULL`, which suppresses the cleanup.

Consistent use of this pattern eliminates most cleanup-path bugs. The code is longer than an early-return style, but it is dramatically safer.

### 也不要信任你自己

A final aspect of fail-safe design is to assume that even your own code has bugs. Include `KASSERT(9)` checks for invariants. `KASSERT` does nothing when `INVARIANTS` is not configured (typical in release builds), but in developer kernels it checks every assertion and panics on failure. That turns a subtle corruption bug into a loud, debuggable panic.

```c
KASSERT(sc != NULL, ("secdev: NULL softc"));
KASSERT(len <= SECDEV_MAX_BUFLEN, ("secdev: len %zu too large", len));
```

Invariants documented as `KASSERT` help readers (future you, future colleagues) understand what the code expects. They also catch regressions that would otherwise silently corrupt state.

### 优雅降级与完全拒绝

A design choice that often arises in fail-safe work: when a non-critical part of an operation fails, should the driver continue with a degraded result, or should it refuse the operation entirely?

There is no universal answer. Each case depends on what the caller is likely to do with partial success. A driver that returns a packet with some fields uninitialized (because a subsystem call failed) is inviting the caller to trust the zero bytes as meaningful. A driver that fails the whole operation is more disruptive but less surprising.

For security-relevant operations, prefer full refusal. A privilege check that fails should not result in "most of the operation ran, but we did not do the privileged step"; it should result in the whole thing refused. A partial success that depended on the skipped step is a bug waiting to be found.

For non-security operations, graceful degradation is often the right call. If an optional statistics update fails, the main operation should still succeed. Document what the degradation looks like so callers can anticipate it.

### 案例研究：/dev/null 中真实世界的安全默认值

The FreeBSD `null` driver, at `/usr/src/sys/dev/null/null.c`, is worth studying as an example of secure-by-default design. It is one of the simplest drivers in the tree, yet its construction embodies most of the principles in this chapter.

It creates two device nodes, `/dev/null` and `/dev/zero`, both with world-accessible permissions (`0666`). This is intentional: they are meant to be used by every process, privileged or not, and neither can leak information or corrupt kernel state. The permission decision is deliberate and documented.

The read, write, and ioctl handlers are minimal. `null_read` returns 0 (end of file). `null_write` consumes input without touching kernel state. `zero_read` fills the user buffer with zeros using `uiomove_frombuf` against a static zero-filled buffer.

The ioctl handler returns `ENOIOCTL` for unknown commands, so the upper layers can translate to the proper error. A small set of specific `FIO*` commands for non-blocking and async behavior are handled, each doing only the minimal bookkeeping that makes sense for a null or zero stream.

The driver has no locking because it has no mutable state worth protecting: the zero buffer is constant, and the read/write operations do not modify any shared data. The absence of locking is not carelessness; it is a consequence of the design minimizing what is shared in the first place.

The driver's `detach` is straightforward, destroying the device nodes. Because there is no async state, no callouts, no interrupts, no taskqueues, the cleanup is correspondingly simple.

What makes this a good example of secure defaults is the discipline of not doing more than is needed. The driver does not speculatively add features, does not expose internal state, does not support ioctls that were not demanded by specific users. Its interface is minimal, which keeps its attack surface minimal. Its behaviour is predictable and has been exactly the same for decades.

Real drivers cannot always be this simple; most have state to manage, hardware to talk to, and operations to perform. But the design principle generalizes: the simpler the driver, the fewer the failure modes. When faced with a choice between adding functionality and leaving it out, the more secure choice is usually to leave it out.

### 第9节总结

Secure defaults come down to a disposition toward refusal. Default to `EINVAL` for unknown inputs. Default to restrictive modes on device nodes. Default to conservative limits on resources. Default to short timeouts. Default to strict privilege requirements. Whitelist, do not blacklist. Fail closed, not open.

None of these are exotic. They are design habits that add up. A driver built on them is not merely a driver that can be made secure; it is a driver that is secure by default, and has to be actively broken before it becomes insecure.

The next section brings the chapter to a close by looking at the other end of the development cycle: testing. How do you know your driver is as safe as you think it is? How do you hunt for the bugs that review missed?

## 第10节：测试与加固驱动程序

驱动程序之所以安全，不是因为你在编写时写了安全的代码。它之所以安全，是因为你彻底地测试了它，包括在你没有为其设计的条件下。本节是关于 FreeBSD 为你提供的在攻击者之前发现漏洞的工具，以及使驱动程序在演进过程中保持安全的习惯。

### 演示：使用 KASAN 发现漏洞

Before the general guidance, consider a specific scenario. You have a driver that passes all your functional tests but that you suspect has a memory-safety bug. You build a kernel with `options KASAN`, boot it, load your driver, and run a stress test. The test crashes the kernel with output that looks something like:

```text
==================================================================
ERROR: KASan: use-after-free on address 0xfffffe003c180008
Read of size 8 at 0xfffffe003c180008 by thread 100123

Call stack:
 kasan_report
 secdev_callout_fn
 softclock_call_cc
 ...

Buffer of size 4096 at 0xfffffe003c180000 was allocated by thread 100089:
 kasan_alloc_mark
 malloc
 secdev_attach
 ...

The buffer was freed by thread 100089:
 kasan_free_mark
 free
 secdev_detach
 ...
==================================================================
```

Read the output carefully. KASAN tells you the exact instruction that accessed freed memory (`secdev_callout_fn`), the exact allocation that was freed (in `secdev_attach`), and the exact free (in `secdev_detach`). Now the bug is obvious: the callout was scheduled at attach, but detach freed the buffer before draining the callout. When the callout fires after the free, it accesses the freed buffer.

The fix: add `callout_drain` to detach before the `free`. KASAN helped you find, in thirty seconds, a bug that might have taken hours or weeks to find by inspection, and that might never have been found in production until a customer reported a random crash.

KASAN is not free. The runtime overhead is substantial, both in CPU (perhaps 2 to 3 times slower) and in memory (each byte of allocated memory has an accompanying shadow byte). You would not run production with it. But for developer testing, and especially for driver authors, it is one of the most effective tools available.

KMSAN works analogously for uninitialized memory reads, and KCOV powers coverage-guided fuzzing. Together they address the main classes of memory-safety bugs: use-after-free (KASAN), uninitialized memory (KMSAN), and bugs not reached by your tests (KCOV plus a fuzzer).

### 使用内核净化器构建

A stock FreeBSD kernel is optimized for production. A development kernel for driver testing should be optimized for finding bugs. The options you add to the kernel config file turn on extra checking.

**`options INVARIANTS`** enables `KASSERT(9)`. Every assertion is checked at runtime. A failed assertion panics the kernel with a stack trace pointing to the assertion. This catches invariant violations that would otherwise corrupt data silently.

**`options INVARIANT_SUPPORT`** is implied by `INVARIANTS` but is sometimes needed as a separate option for modules built against an `INVARIANTS` kernel.

**`options WITNESS`** turns on the WITNESS lock-order checker. Every lock acquisition is recorded, and the kernel panics if a cycle is detected (A held, then B acquired; later, B held, then A acquired). This catches deadlock bugs before they deadlock.

**`options WITNESS_SKIPSPIN`** disables WITNESS for spin mutexes, which can reduce overhead at the cost of missing some checks.

**`options DIAGNOSTIC`** enables additional runtime checks in various subsystems. It is looser than `INVARIANTS` and catches some additional cases.

**`options KASAN`** enables the Kernel Address Sanitizer, which detects use-after-free, out-of-bounds access, and some uninitialized memory uses. It requires compiler support and a substantial memory overhead but is excellent for finding memory-safety bugs.

**`options KMSAN`** enables the Kernel Memory Sanitizer, which detects uses of uninitialized memory. This directly catches the information-leak bugs described in Section 7.

**`options KCOV`** enables kernel coverage tracking, which is what makes coverage-guided fuzzing work.

A driver-development kernel might add:

```text
options INVARIANTS
options INVARIANT_SUPPORT
options WITNESS
options DIAGNOSTIC
```

and, for deeper memory-safety testing, `KASAN` or `KMSAN` on supported architectures. Build that kernel, boot it, and run your driver against it. Many bugs surface immediately.

Production builds do not typically include these options (they slow the kernel significantly). Use them as a development safety net.

### 压力测试

A driver that passes functional tests can still fail under stress. Stress testing exercises the driver's concurrency, its allocation patterns, and its error paths at volumes that amplify race conditions.

A simple stress harness for a character device might:

Open the device from N processes concurrently.

Each process issues M ioctls with valid and invalid arguments in a random order.

A separate process periodically detaches and re-attaches the device (or kldunload/kldload).

This quickly exposes races between user-space operations and detach, which are among the hardest race categories to catch by inspection.

FreeBSD's `stress2` test framework at `https://github.com/pho/stress2` has a long history of finding kernel bugs. It includes scenarios for VFS, networking, and various subsystems. A driver author can learn a lot by reading those scenarios and adapting them to the driver's interface.

### 模糊测试

Fuzzing is the technique of generating large numbers of random or semi-random inputs and observing whether the program crashes, asserts, or misbehaves. Modern fuzzers are coverage-guided: they watch which code paths are exercised and evolve inputs that explore new paths. This is far more effective than purely random input.

For driver testing, the key fuzzer is **syzkaller**, an external project that understands syscall semantics and produces structured inputs. Syzkaller is not part of the FreeBSD base system; it is an external tool that runs on top of a FreeBSD kernel built with `KCOV` coverage instrumentation. Syzkaller has found many bugs in the FreeBSD kernel over the years, and a driver that wants to be exercised thoroughly benefits from being described in a syzkaller syscall description (`.txt` file under syzkaller's `sys/freebsd/`).

If your driver exposes a substantial ioctl or sysctl interface, consider writing a syzkaller description for it. The format is straightforward, and the investment pays off the first time syzkaller finds a bug no human reviewer would have spotted.

Simpler fuzzing approaches also work. A shell script that issues random ioctls with random arguments and watches `dmesg` for panics is better than no fuzzing at all. The goal is to generate inputs your design did not anticipate.

### ASLR、PIE 和栈保护

Modern FreeBSD kernels use several exploit-mitigation techniques. Understanding them is part of understanding why the bugs we have discussed matter.

**kASLR**, kernel Address Space Layout Randomization, places the kernel's code, data, and stacks at randomized addresses at boot. An attacker who wants to jump to kernel code, or to overwrite a specific function pointer, does not know where that code or pointer is. kASLR is foundational for making many memory-safety bugs unexploitable in practice.

Information leaks (Section 7) are particularly dangerous because they can defeat kASLR. A single leaked kernel pointer gives the attacker the base address and unlocks everything else.

**SSP**, the Stack-Smashing Protector, places a canary value on the stack between local variables and the return address. When a function returns, the canary is checked; if it has been overwritten (because a buffer overflow clobbered it on the way to the return address), the kernel panics. SSP does not prevent overflows but it prevents many of them from gaining control of execution.

Not every function is protected. The compiler applies SSP based on heuristics: functions with local buffers, functions that take addresses of locals, and so on. Understanding this means understanding why certain buffer-overflow patterns are more exploitable than others.

**PIE**, Position-Independent Executables, allows the kernel (and modules) to be relocated to random addresses. Combined with kASLR, this is what makes the randomization effective.

**Stack guards and guard pages** surround kernel stacks with unmapped pages. An attempt to write past the stack hits an unmapped page and panics rather than silently corrupting adjacent memory.

**W^X**, write-xor-execute, keeps kernel memory either writable or executable, never both. This prevents many classic exploits that relied on writing shellcode into memory and then jumping to it.

A driver author does not implement any of these; they are kernel-wide protections. But a driver's bugs can undermine them. An information leak defeats kASLR. A buffer overflow that reliably hits a function pointer or vtable defeats SSP. A use-after-free that races a fresh allocation gives an attacker controlled memory at a kernel address.

In short: the point of careful driver code is not just to avoid crashes. It is to keep the kernel's defenses intact. When your driver leaks a pointer, you did not merely expose information; you downgraded the entire system's exploit-mitigation posture.

### 审查你的差异

Every time you modify the driver, read the diff carefully. Look for:

New `copyin` or `copyout` calls: are the lengths clamped? Are the buffers zeroed first?

New privilege-sensitive operations: do they have `priv_check` or equivalent?

New locking: is the lock order consistent with other code?

New allocations: are they paired with frees on every path, including error paths?

New log messages: are they rate-limited? Do they leak sensitive data?

New user-visible fields in structures: are they initialized? Is the structure zeroed before the copyout?

A diff-review habit catches many regressions. If your project uses code review (it should), make these questions part of the checklist.

### 静态分析

FreeBSD kernel code can be analyzed by several static-analysis tools, including `cppcheck`, `clang-analyzer` (scan-build), and, increasingly, Coverity and GitHub CodeQL-style tools. These tools often report warnings that a human reviewer would miss: a conditional that can never be true, a pointer used after a path where it was freed, a missing null check.

Treat static-analysis warnings seriously. Most are false positives; some are real bugs. Silencing a warning should be a decision, not a reflex. When the tool is wrong, add a comment explaining why. When the tool is right, fix the code.

`syntax check` with `bmake` on the kernel source tree is a fast first pass. Running `clang --analyze` or `scan-build` against your driver is a deeper pass. Neither replaces review or testing, but both catch bugs at low cost.

### 代码审查

No tool replaces another pair of eyes. Review is especially important for security-relevant code. When proposing a change to a security-sensitive path, find someone else to look at it. Describe what the change is, what invariants it preserves, and what you checked. Be grateful when they find a problem you missed.

For open-source projects, the FreeBSD review system (`reviews.freebsd.org`) provides a convenient way to get external review. Use it. The community has a long tradition of thoughtful, security-aware review, and reviewers often catch things you would not.

### 修改后的测试

When a bug is found and fixed, add a test that would have caught it. This matters because:

The same bug class often recurs in other places. A test that catches the specific instance may catch future similar bugs.

Without a test, you have no way to know that the fix worked.

Without a test, a future refactoring may re-introduce the bug.

Tests can be unit tests (in user space, exercising individual functions), integration tests (loading the driver in a VM and exercising it), or fuzz cases (inputs that used to crash and should not now). All have their place.

### 持续集成

Automated testing on every change catches regressions early. A CI setup that builds the driver against a development kernel with `INVARIANTS`, `WITNESS`, and possibly `KASAN` runs the stress harness, and checks the result, is the backbone of a driver that stays safe.

For a driver in the FreeBSD tree, this is already provided by the project's CI. For out-of-tree drivers, setting up CI takes some effort but pays back quickly.

### 严肃对待漏洞报告

When someone reports a crash or a suspected vulnerability in your driver, treat it as real until you have evidence otherwise. Even a "harmless" bug may be exploitable in ways the reporter did not see. "I can crash the kernel with this ioctl" is not a minor issue; it is at minimum a denial-of-service bug, and very often a memory-safety bug that could become arbitrary code execution.

The FreeBSD security team (`secteam@freebsd.org`) is the right audience for vulnerability reports in the base system. For out-of-tree drivers, have a similar channel. Respond quickly, fix carefully, and credit the reporter when appropriate.

### 随时间推移进行加固

A driver's security posture is not static. New classes of bugs emerge. New mitigations become available. New attack techniques make old bugs more exploitable. Budget time every release cycle to:

Re-read the security-relevant paths of the driver.

Check for newly discovered compiler warnings or static-analysis findings.

Try the latest tools (KASAN, KMSAN, syzkaller) against the driver.

Update the privilege model if FreeBSD has added new `PRIV_*` codes or more specific checks.

Remove interfaces that no user actually needs.

The discipline of regular re-examination is what distinguishes a driver that is secure on the day it ships from one that stays secure through its lifetime.

### 事后处理：当漏洞变成 CVE 时该做什么

A realistic chapter on security must cover the possibility that, despite all the precautions, a bug in your driver gets reported externally as a vulnerability. The pathway is typically:

A researcher or user discovers unexpected behavior in your driver.

They investigate and determine that the behavior is a security bug: information leak, privilege escalation, crash-on-untrusted-input, or similar.

They report the bug, ideally via a responsible-disclosure channel (for FreeBSD base-system drivers, this is the `secteam@freebsd.org` address).

You receive the report.

The first response matters. Even if the bug turns out to be less serious than it looks, treat the reporter as a collaborator, not an adversary. Acknowledge receipt promptly. Ask clarifying questions if needed. Do not dismiss without investigation. Do not attempt to gag the reporter. Most vulnerability researchers want the bug fixed; if you cooperate, you get a fix faster and usually get public credit that reflects well on the project.

Triage the report technically. Can you reproduce the bug? Is it a crash, an information leak, a privilege escalation, or something else? What is the attacker model: who has to have access, and what do they gain? Is it exploitable in combination with other known bugs?

If confirmed, coordinate a fix. Keep in mind that for FreeBSD base-system drivers, the fix must flow through the project's normal review process and, where appropriate, through the security advisory process. For out-of-tree drivers, you have more flexibility but still should write the fix carefully and test it thoroughly.

Prepare the disclosure. Typical disclosure practice gives the project time to fix the bug before details become public. Industry norms are usually 90 days. Within that window, the advisory is prepared, a patched version is released, and public disclosure happens simultaneously with the release. Do not leak details early; do not delay past the agreed date.

Write the commit message carefully. Security fix commits should mention the vulnerability without giving attackers a roadmap. "Fix incorrect bounds check in secdev_write that could allow kernel memory disclosure" is better than either "tweak write" (too vague, reviewers miss it) or "Fix CVE-2026-12345, where an attacker can read arbitrary kernel memory by issuing a write of X bytes followed by a read, bypassing Y check" (too specific, attackers read your commit history before users can upgrade).

After the release, if details become public, be prepared to answer questions. Users want to know: am I vulnerable, how do I upgrade, and how can I tell if I was attacked? Have clear, calm answers ready.

Post-mortem the bug. Not to blame, but to learn. Why did the bug exist? Was there a pattern the review missed? Could a tool have caught it? Should the team's process change? Write the conclusions down; apply them in future work.

Security is a continuing practice, and post-incident learning is one of its most important parts. A project that fixes the bug and moves on has learned nothing; a project that reflects on why the bug occurred makes the next bug less likely.

### 第10节总结

Testing and hardening are how a careful design becomes a secure one. Build your development kernel with `INVARIANTS`, `WITNESS`, and, where possible, `KASAN` or `KMSAN`. Stress-test under concurrent load. Fuzz with syzkaller or, at minimum, with a random-input harness. Use static analysis. Review diffs. Respond seriously to bug reports. Re-test after every fix. Harden over time.

A driver does not become secure by accident. It becomes secure because the author assumed bugs existed, looked for them with every tool available, and fixed them one at a time.

## 动手实验

本章的实验构建一个名为 `secdev` 的小型字符设备，并引导你逐步使其更加安全。每个实验从提供的起始文件开始，要求你进行特定更改，并提供一个"已修复"的参考版本进行比较。按顺序完成它们。

这些实验设计为在 FreeBSD 14.3 虚拟机或测试主机上运行，内核崩溃是可以接受的。不要在运行重要服务的机器上运行它们；不安全驱动程序中的错误可能导致内核崩溃。

If you are running these labs inside a VM, make sure the VM is configured to write crash dumps to a location you can recover after reboot. Enable `dumpon(8)` and set `/etc/fstab` appropriately so that core dumps land in `/var/crash` after a panic. See `/usr/src/sbin/savecore/savecore.8` for details. This infrastructure is how you will diagnose any panics the labs provoke.

这些实验的配套文件位于 `examples/part-07/ch31-security/` 下。每个实验都有自己的子文件夹，包含 `secdev.c` 源文件、`Makefile`、描述实验的 `README.md`，以及适当情况下包含小型用户空间测试程序的 `test/` 子文件夹。

在完成实验时，在实验日志中保持运行日志：你修改了哪些文件，加载损坏版本时观察到了什么，修复后观察到了什么，以及任何意外行为。日志是一个学习工具；它迫使你表达你看到的东西，这是学习巩固的方式。

### 实验1：不安全的 secdev

**目标。** 构建、加载并测试故意不安全的 `secdev` 版本，确认它能工作，然后以安全思维阅读代码识别至少三个安全问题。

**先决条件。**

本实验假设你有一台可以加载和卸载内核模块的 FreeBSD 14.3 虚拟机或测试系统。你应该已经完成了模块构建章节（第2部分及之后），以便熟悉 `make`、`kldload`、`kldunload` 和设备节点访问。如果没有，暂停并复习这些章节；第31章的其余部分假设你对模块编译感到熟悉。

**步骤。**

1. Copy `examples/part-07/ch31-security/lab01-unsafe/` to a working directory on your FreeBSD test machine. You can either clone the book's companion repository or copy the files manually if you extracted them locally.

2. Read `secdev.c` carefully. Note what it does: it provides a `/dev/secdev` character device with `read`, `write`, and `ioctl` operations. `read` returns the contents of an internal buffer. `write` copies user data into the buffer. An ioctl (`SECDEV_GET_INFO`) returns a structure describing the device.

3. Read `Makefile`. It should be a standard FreeBSD kernel module makefile using `bsd.kmod.mk`.

4. Build the module with `make`. Address any build errors by consulting earlier chapters on module-building. A successful build produces `secdev.ko`.

5. Load the module with `kldload ./secdev.ko`. Verify with:
   ```
   kldstat | grep secdev
   ls -l /dev/secdev
   ```
   You should see the module listed and the device node present with whatever permissions the unsafe driver created.

6. Exercise the device as a normal functional test:
   ```
   echo "hello" > /dev/secdev
   cat /dev/secdev
   ```
   You should see `hello` printed back. If you do not, check `dmesg` for error messages.

7. Now, review the code with the security mindset from this chapter. For each of the following categories, find at least one issue in the unsafe code:
   - Buffer overflow opportunity.
   - Information leak opportunity.
   - Missing privilege check.
   - Unchecked user input.
   Write down each finding in your lab logbook, including the line number and the specific concern.

8. Unload the module with `kldunload secdev` when you are done. Verify with `kldstat` that it is gone.

**观察。**

The unsafe `secdev` has several issues by design. In `secdev_write`, the code calls `uiomove(sc->sc_buf, uio->uio_resid, uio)`, which copies `uio_resid` bytes regardless of `sizeof(sc->sc_buf)`. A write of 8192 bytes to a 4096-byte buffer overflows the internal buffer. Depending on what lies next to `sc_buf` in memory, this may or may not crash immediately, but it always corrupts adjacent kernel memory.

`SECDEV_GET_INFO` returns a `struct secdev_info` that is filled in field-by-field without being zeroed first. Any padding bytes between fields carry stack contents to user space. The structure likely has padding around the `uint64_t` members for alignment.

The device is created with `args.mda_mode = 0666` (or equivalent), allowing any user on the system to read and write. A user with no special privilege can corrupt the kernel buffer or leak information through the ioctl.

The ioctl handler does not check `priv_check` or similar. Any user who can open the device can issue any ioctl.

`secdev_read` copies `sc->sc_buflen` bytes regardless of the caller's buffer size, potentially reading beyond valid data if `sc_buflen` was ever larger than the currently valid content.

**额外探索。**

As a non-root user, try the operations that should be privileged and confirm that they succeed when they should not. Write a short C program that issues `SECDEV_GET_INFO` and prints the returned structure as a hex dump. Look for non-zero bytes in fields that were not explicitly set; those are leaked kernel data.

**总结。**

本实验的目标是模式识别。真正的驱动程序会有所有这些漏洞的更微妙版本，埋藏在数百行代码中。训练自己在简单的驱动程序中看到它们可以使它们在其他地方更容易被看到。将 `lab01-unsafe/secdev.c` 保留为不该做什么的参考。

### 实验2：缓冲区边界检查

**目标。** 修复 `write` 中的缓冲区溢出并在 `read` 中添加相应的长度检查。观察压力测试时驱动程序行为的差异。

**步骤。**

1. Start from `lab02-bounds/secdev.c`. This is `lab01`'s code plus some `TODO` comments marking where you will add checks.

2. In `secdev_write`, calculate how much data can safely be written to the internal buffer. Remember that `uiomove` writes at most the length you pass. Clamp `uio->uio_resid` to the remaining space before calling `uiomove`.

3. In `secdev_read`, make sure you only copy out as much data as is actually valid in the buffer, not its full allocated size.

4. Rebuild and re-test. With the fixes in place, a write of 10KB to a 4KB buffer should simply fill the buffer, not overflow it.

5. Stress-test the fixed driver:
   ```
   dd if=/dev/zero of=/dev/secdev bs=8192 count=100
   dd if=/dev/secdev of=/dev/null bs=8192 count=100
   ```
   Neither command should crash the kernel or produce warnings in `dmesg`. If they do, your bounds check is incomplete.

6. Compare your fix with `lab02-fixed/secdev.c`. If your fix is different but correct, that is fine; multiple solutions can be valid. If yours is incorrect, study the reference fix and understand where you went wrong.

**Building confidence.**

Write a small C program that issues writes of various sizes (0 bytes, 1 byte, buffer size, buffer size + 1, much larger than buffer size) and verifies that each returns the expected number of bytes written or a sensible error. This kind of boundary testing is what real driver tests look like.

**总结。**

Bounds checking is the simplest security fix and it catches a large fraction of real-world driver bugs. Internalize the pattern: every `uiomove`, `copyin`, `copyout`, and memcpy bounds the length against both source and destination sizes. The compiler cannot catch this for you; it is entirely the author's responsibility.

### 实验3：copyout 前清零

**目标。** 修复 `SECDEV_GET_INFO` ioctl 中的信息泄露。通过用户空间测试程序观察损坏版本和修复版本之间的差异。

**步骤。**

1. Start from `lab03-info-leak/secdev.c`. This contains the ioctl as in the original unsafe code.

2. Observe the structure definition. Note the padding between fields:
   ```c
   struct secdev_info {
       uint32_t version;
       /* 4 bytes of padding here on 64-bit systems */
       uint64_t flags;
       uint16_t id;
       /* 6 bytes of padding to align name to 8 bytes */
       char name[32];
   };
   ```
   Check the size with `pahole` or a small C program that prints `sizeof(struct secdev_info)`.

3. Before fixing, build and load the broken version. Run the test program provided in `lab03-info-leak/test/leak_check.c`. It issues the ioctl repeatedly and dumps the returned structure as a hex dump. Look at the padding bytes. You should see non-zero values that differ between runs; those are leaked kernel stack bytes.

4. In `secdev_ioctl`, before filling in the `struct secdev_info`, zero the structure with `bzero` (or use `= { 0 }` initialization).

5. Also fix the name field: use `snprintf` instead of `strncpy` to guarantee a NUL terminator, and copy only up to the NUL rather than the full buffer size.

6. Rebuild and re-test with the same `leak_check` program. The padding bytes should now be zero on every run. The visible behavior from user space is unchanged; the internal change is that padding bytes no longer carry stack contents.

7. Compare with `lab03-fixed/secdev.c`.

**A deeper exploration.**

If you have `KMSAN` built into your kernel, load the broken version of the driver and run `leak_check`. KMSAN should report an uninitialized read when the structure is copied out. This demonstrates why KMSAN is valuable: it catches information leaks that are invisible without it.

**总结。**

这个修复只需要一次 `bzero` 调用。好处是 ioctl 永远不会泄露信息，无论将来的更改添加或删除什么字段。让 `bzero`（或零初始化声明）成为任何将触及 `copyout`、`sysctl` 或类似边界的结构的反射。

### 实验4：添加权限检查

**目标。** 将设备限制为特权用户，并验证非特权访问被拒绝。

**步骤。**

1. Start from `lab04-privilege/secdev.c`.

2. Modify the device-node creation code in `secdev_modevent` (or `secdev_attach`, depending on structure) to use a restrictive mode (`0600`) and the root user and group:
   ```c
   args.mda_uid = UID_ROOT;
   args.mda_gid = GID_WHEEL;
   args.mda_mode = 0600;
   ```

3. In `secdev_open`, add a `priv_check(td, PRIV_DRIVER)` call at the top:
   ```c
   error = priv_check(td, PRIV_DRIVER);
   if (error != 0)
       return (error);
   ```
   Return the error if the check fails.

4. Rebuild and reload the module.

5. Test from a non-root shell:
   ```
   % cat /dev/secdev
   cat: /dev/secdev: Permission denied
   ```
   Open should fail with `EPERM` (reported as "Permission denied"). The filesystem mode blocks access before `d_open` is even reached.

6. Temporarily change the device-node mode (as root) with `chmod 0666 /dev/secdev`. Try again as non-root. This time the filesystem allows the open, but `priv_check` in `d_open` refuses:
   ```
   % cat /dev/secdev
   cat: /dev/secdev: Operation not permitted
   ```
   This demonstrates the in-kernel layer of the defense.

7. Reset the permissions with `chmod 0600 /dev/secdev` or reload the module to restore the default.

8. As root, the device should continue to work normally. Verify:
   ```
   # echo "hello" > /dev/secdev
   # cat /dev/secdev
   hello
   ```

9. Compare with `lab04-fixed/secdev.c`.

**Digging deeper.**

Try creating a jailed environment and running a shell as root inside the jail:
```console
# jail -c path=/ name=testjail persist
# jexec testjail sh
# cat /dev/secdev
```
Depending on whether your driver has added a `jailed()` check, the behavior differs. If the driver does not check `jailed`, jailed-root can still access the device. Add `if (jailed(td->td_ucred)) return (EPERM);` at the top of `secdev_open` and verify that the jailed access is now refused.

**总结。**

限制设备节点权限是两层防御：文件系统层和内核内 `priv_check`。两者结合使驱动程序对配置错误具有健壮性。在上面添加 `jailed()` 甚至阻止 jail 内的 root 访问敏感操作。每层防御不同的失败模式；不要依赖任何单一层。

### 实验5：速率限制的日志记录

**目标。** 为格式错误的 ioctl 添加速率限制的日志消息，并验证格式错误请求的洪泛不会淹没日志。

**步骤。**

1. Start from `lab05-ratelimit/secdev.c`.

2. Add a static `struct timeval` and a static `int` to hold rate-limit state. These are global per-driver, not per-softc, unless you specifically want per-device limits:
   ```c
   static struct timeval secdev_log_last;
   static int secdev_log_pps;
   ```

3. In `secdev_ioctl`, in the `default` branch (the case that handles unknown ioctls), use `ppsratecheck` to decide whether to log:
   ```c
   default:
       if (ppsratecheck(&secdev_log_last, &secdev_log_pps, 5)) {
           device_printf(sc->sc_dev,
               "unknown ioctl 0x%lx from uid %u\n",
               cmd, td->td_ucred->cr_uid);
       }
       return (ENOTTY);
   ```
   The third argument, `5`, is the maximum messages per second.

4. Rebuild and reload.

5. Write a tiny test program that issues a million bad ioctls in a tight loop:
   ```c
   #include <sys/ioctl.h>
   #include <fcntl.h>
   int main(void) {
       int fd = open("/dev/secdev", O_RDWR);
       for (int i = 0; i < 1000000; i++)
           ioctl(fd, 0xdeadbeef, NULL);
       return (0);
   }
   ```

6. While it runs (as root), monitor `dmesg -f`. You should see messages arriving, but at no more than about 5 per second. Without rate limiting, you would have a million messages.

7. Count the messages with something like `dmesg | grep "unknown ioctl" | wc -l`. Compare to one million (the number of attempts).

8. Compare with `lab05-fixed/secdev.c`.

**Variations to try.**

Replace `ppsratecheck` with `eventratecheck` and note the difference (event-based vs per-second). Experiment with different maximum rates. Add a suppressed-count summary that emits periodically ("suppressed N messages in last M seconds") for operator visibility.

**总结。**

速率限制的日志记录让你对可疑活动有可见性，而不会使驱动程序本身成为拒绝服务载体。将模式应用于任何可由用户操作触发的日志消息。成本是每个日志语句多几行；好处是你的驱动程序不再是攻击者可以用来淹没系统的工具。

### 实验6：安全分离

**目标。** 使 `secdev_detach` 在并发活动下安全。通过故意让卸载与活动使用竞态，观察修复如何防止释放后使用崩溃。

**步骤。**

1. Start from `lab06-detach/secdev.c`. This version introduces a small callout that periodically updates an internal counter, and an ioctl that sleeps briefly to simulate long-running work.

2. Review the current `detach` function. Note what it frees and in what order. The starter file intentionally has a flawed detach that frees the softc without draining.

3. Test the flawed version first (build with `INVARIANTS` and `WITNESS` in the kernel):
   - Start a test program that holds `/dev/secdev` open and issues the slow ioctl in a loop.
   - While it runs, issue `kldunload secdev`.
   - Observe the result. You may see a kernel panic, a stuck kldunload, or, if you are lucky, nothing visible (the race may not fire on every run). `WITNESS` may complain about lock state.
   - Rebuild and try again until you see the problem. Concurrent races can be flaky.

4. Now fix the detach:
   - Use `destroy_dev` on the cdev before any other cleanup, so that no new user-space thread can enter the driver, and any in-flight thread finishes before `destroy_dev` returns.
   - Add a `callout_drain` call before freeing the softc. This ensures that any in-flight callout has finished.
   - If the driver uses a taskqueue, add `taskqueue_drain_all`.
   - Only after all draining, free resources.

5. Rebuild and re-test the same race:
   - The user program continues running uninterrupted during `kldunload`.
   - After `kldunload`, the user program's next ioctl receives an error (typically `ENXIO`) because the cdev was destroyed.
   - The kernel remains stable. No panic, no WITNESS complaint.

6. Compare with `lab06-fixed/secdev.c`. Confirm that the fixed version handles in-flight activity safely.

**Understanding what happened.**

The flawed version races because:
- `destroy_dev` is called, or is not called early enough. In-flight d_* calls continue.
- The callout is scheduled in the future and has not fired yet.
- The softc is freed while something still references it.
- The freed softc is reused by the allocator for some other purpose.
- The callout fires, touches what it thinks is its softc, and corrupts whatever memory is now there.

The fix sequences the cleanup: stop accepting new entries first (`destroy_dev`), stop in-flight entries by waiting for them to leave (part of `destroy_dev`'s contract), stop scheduled work (`callout_drain`), and only then free state. Each step closes a door; nothing beyond a closed door can reach the memory being freed.

**总结。**

分离时竞态是最难通过检查捕获的驱动程序漏洞之一，因为漏洞仅在时间对齐时发生。在每个 `detach` 函数中防御性地使用 `destroy_dev`、`callout_drain` 和 `taskqueue_drain_all` 是你能采用的最高价值的习惯之一。机械地做，即使你认为你的驱动程序没有异步活动。下一个添加定时回调的作者可能会忘记；你的防御性 detach 会捕获他们。

### 实验7：处处使用安全默认值

**目标。** 将到目前为止的每一课应用到一个驱动程序："安全 secdev"。从骨架构建它，然后像执行安全审计一样审查最终结果。

**步骤。**

1. Start from `lab07-secure/secdev.c`. This is a skeleton with `TODO` markers in many places.

2. Fill in each `TODO`, applying the lessons from Labs 1 to 6 plus any additional defenses you think appropriate. Suggested additions:
   - A `MALLOC_DEFINE` for the driver's memory.
   - A softc mutex protecting all shared fields.
   - `priv_check(td, PRIV_DRIVER)` in `d_open` and in each privileged ioctl.
   - `jailed()` checks for operations that should not be available to jailed users.
   - `securelevel_gt` for operations that should be refused at elevated securelevel.
   - `bzero` on every structure before filling it for `copyout`.
   - `M_ZERO` on every allocation that will be copied to user space.
   - `explicit_bzero` on sensitive buffers before `free`.
   - Rate-limited `device_printf` on every log message triggerable from user space.
   - `destroy_dev`, `callout_drain`, and other drains in `detach` before any free.
   - A sysctl-controlled `secdev_debug` flag that gates verbose logging.
   - Input validation that whitelists allowed operation codes.
   - Bounded copies in both directions.
   - `KASSERT` statements documenting internal invariants.

3. Rebuild and load the module.

4. Run a comprehensive functional test to confirm everything still works:
   - As root, open the device, read, write, ioctl.
   - As non-root, confirm `/dev/secdev` is inaccessible.
   - Inside a jail, confirm sensitive operations are refused.

5. Run a security stress test:
   - Boundary cases (0-byte reads, exactly buffer-size writes, one-byte-over writes).
   - Malformed ioctls.
   - Concurrent open/read/write/close from multiple processes.
   - `kldunload` during active use.

6. Compare your work with `lab07-fixed/secdev.c`. Note differences. Where your version is more defensive, ask whether the extra defense is worth the complexity. Where the reference is more defensive, ask whether you missed a defense.

**A self-review.**

Once your lab 7 driver builds and passes tests, put on the reviewer hat. Go through the Security Checklist section of this chapter and confirm each item. Any items you cannot confirm are gaps in your driver. Fix them now, while the code is fresh; later, finding and fixing such gaps is slower and more error-prone.

**总结。**

这个实验是本章的综合。你完成的驱动程序仍然是一个简单的字符设备，但它是一个你在真正的 FreeBSD 树中看到不会感到尴尬的驱动程序。你在这里应用的实践是将业余驱动程序与专业驱动程序分开的相同实践。将你的实验7驱动程序保留为参考：当你编写第一个真正的驱动程序时，这是你将开始的骨架。

## 挑战练习

这些挑战超出了实验范围。它们更长、更开放，假设你已经完成了实验7。慢慢来。它们都不需要新的 FreeBSD API；它们需要更深入地应用你所学到的知识。

这些挑战旨在在几天或几周内尝试，而不是一次完成。它们锻炼判断力与编码能力："这安全吗"这个问题通常是"对什么威胁模型安全"。明确威胁模型是练习的一部分。

### 挑战1：添加多步骤 ioctl

Design and implement an ioctl that performs a multi-step operation on `secdev`: first, the user uploads a blob; second, the user requests processing; third, the user downloads the result. Each step is a separate ioctl call.

The challenge is to manage per-open state correctly: the blob uploaded in step 1 must be associated with the calling file descriptor, not globally. Two concurrent users must not see each other's blobs. State must be cleaned up when the file descriptor is closed, even if the user never reached step 3.

Security considerations: bound the blob size, validate each step of the state machine (cannot request processing without a blob; cannot download without completing processing), make sure partial state on error paths is cleaned up, and make sure a user-visible identifier (if you expose one) is not a kernel pointer.

### 挑战2：编写 syzkaller 描述

Write a syzkaller syscall description for `secdev`'s ioctl interface. The format is documented in the syzkaller repository. Install syzkaller and feed it your driver; run it for at least an hour (ideally longer) and see what it finds.

If it finds bugs, fix them. Write a note about what each bug was and how the fix works. If it does not find bugs in several hours, consider whether your syzkaller description really exercises the driver. Often a description that does not find bugs is not exploring the interface thoroughly.

### 挑战3：在自己的代码中检测双重释放

Intentionally introduce a double-free bug into a copy of your secure `secdev`. Build the module against a kernel with `INVARIANTS` and `WITNESS`. Load and exercise the module in a way that triggers the double-free. Observe what happens.

Now rebuild the kernel with `KASAN`. Load and exercise the same broken module. Observe the difference in how the bug is detected.

Write down what each sanitizer caught and how readable the output was. This exercise builds intuition for which sanitizer to reach for first in which situation.

### 挑战4：对现有驱动程序进行威胁建模

Pick a driver in the FreeBSD tree that you have not previously examined (something small, ideally under 2000 lines). Read it carefully. Write a threat model: who are the callers, what privileges do they need, what could go wrong, what mitigations are in place, what could be added?

The goal is not to find specific bugs. It is to practice the security mindset on real code. A good threat model is a few pages of prose that would let another engineer review the same driver efficiently.

### 挑战5：比较 `/dev/null` 和 `/dev/mem`

Open `/usr/src/sys/dev/null/null.c` and `/usr/src/sys/dev/mem/memdev.c` (or the per-architecture equivalents). Read both.

Write a short essay (a page or two) on the security differences. `/dev/null` is one of the simplest drivers in FreeBSD; what does it do, and why is it safe? `/dev/mem` is one of the most dangerous; what does it do, and how does FreeBSD keep it safe? What can you learn about the shape of secure driver code from the contrast?

## 故障排除与常见错误

A short catalogue of mistakes I have seen repeatedly in real driver code, with the symptom, the cause, and the fix.

### "有时能工作，有时不能"

**Symptom.** A test passes most of the time but occasionally fails. Running under load amplifies the failure rate.

**Cause.** Almost always a race condition. Something is being read and written concurrently without a lock.

**Fix.** Identify the shared state. Add a lock. Acquire the lock for the whole check-and-act sequence. Do not trust `atomic_*` operations to solve a multi-field invariant problem.

### "驱动程序在卸载时崩溃"

**Symptom.** `kldunload` triggers a panic or a stuck kernel.

**Cause.** A callout, taskqueue task, or kernel thread is still running when `detach` frees the structure it uses. Or an in-flight cdev operation is still in the driver's code when `destroy_dev` is skipped.

**Fix.** In `detach`, call `destroy_dev` before anything else. Then `callout_drain` every callout, `taskqueue_drain_all` every taskqueue, and wait for every kernel thread to exit. Only then free state. Structure the detach as a strict reverse of attach.

### "ioctl 以 root 身份可以工作，但以服务账户不行"

**Symptom.** User reports that root can use the device, but a non-root account cannot.

**Cause.** Device node permissions are too restrictive, or a `priv_check` call refuses the operation.

**Fix.** If the operation truly should be privileged, this is working as intended; document it. If not, reconsider: was the privilege check added in error? Is the device-node mode too tight? The correct answer depends on the operation; most real answers are "yes, it should be privileged, update the docs".

### "dmesg 被淹没"

**Symptom.** `dmesg` shows thousands of identical messages from the driver. Legitimate messages are being pushed out.

**Cause.** A log statement in a path triggerable from user space, without rate limiting.

**Fix.** Wrap the log in `ppsratecheck` or `eventratecheck`. Limit to a few per second. If the message is about an error, include a count of suppressed messages when the rate returns to normal (the rate helpers support this).

### "返回的结构中包含垃圾字节"

**Symptom.** A user-space tool reports seeing apparently random data in a field it did not expect to be set.

**Cause.** The driver did not zero the structure before filling and copying out. The "random" data is actually stack or heap content from before.

**Fix.** Add a `bzero` at the top of the function, or initialize the structure with `= { 0 }` at declaration. Never `copyout` an uninitialized structure.

### "内存泄漏但看不到在哪里"

**Symptom.** `vmstat -m` shows the driver's malloc type growing over time. Eventually the system runs out of memory.

**Cause.** An allocation path that does not pair with a free path, or an error path that returns without freeing.

**Fix.** Use a named malloc type (`MALLOC_DEFINE`). Audit every allocation. Walk every error path. Consider the single-cleanup-label pattern. Build with `INVARIANTS` and watch for allocator warnings on unload.

### "kldload 成功但设备没有出现在 /dev 中"

**Symptom.** `kldstat` shows the module loaded, but there is no `/dev/secdev` entry.

**Cause.** Usually an error in the `attach` sequence before `make_dev_s` is called, or `make_dev_s` itself failed silently.

**Fix.** Check the return value of `make_dev_s`. Add a `device_printf` reporting any error. Verify `attach` is being reached by adding a `device_printf` at the top.

### "简单的 C 测试通过，但循环执行的 shell 脚本失败"

**Symptom.** Single-shot testing works. Rapid repeated testing fails.

**Cause.** Likely a race between repeated operations, or a resource that is not being cleaned up between calls. Sometimes a TOCTOU bug that is timing-sensitive.

**Fix.** Stress-test harder. Use `dtrace` or `ktrace` to see what is happening. Look for state that persists across calls and should not.

### "KASAN 报告释放后使用，但我的 malloc/free 是平衡的"

**Symptom.** `KASAN` reports access to freed memory, but visual inspection of the driver shows each allocation freed exactly once.

**Cause.** A common subtle case: a callout or taskqueue task still holds a pointer to the freed object. The callout fires after free.

**Fix.** Trace the callout lifecycle. Ensure `callout_drain` (or equivalent) runs before any free. A related case is an asynchronous completion callback; ensure the operation is either completed or cancelled before the owning structure is freed.

### "WITNESS 抱怨锁顺序"

**Symptom.** `WITNESS` reports "lock order reversal" and identifies two locks that were acquired in inconsistent order.

**Cause.** At one point the code acquired lock A then lock B; at another point it acquired lock B then lock A. This can deadlock.

**Fix.** Decide on a canonical order for your locks. Document it. Acquire them in that order everywhere. If a code path legitimately needs the reverse order, use `mtx_trylock` with a backoff-and-retry pattern.

### "vmstat -m 显示负数的空闲计数"

**Symptom.** `vmstat -m` lists the driver's malloc type with a negative number of allocations, or with an inuse count that increases over time without bound.

**Cause.** A mismatched `malloc`/`free` type, or a leak where allocations happen without corresponding frees.

**Fix.** A negative free count almost always means a `free` call passed the wrong type tag. Audit every `free(ptr, M_TYPE)` and confirm the type matches the `malloc`. A continuously rising inuse count is a leak; audit every path that allocates and confirm it has a matching free on every exit.

### "驱动程序在 amd64 上工作但在 arm64 上崩溃"

**Symptom.** Functional testing on amd64 passes; the same driver panics on arm64.

**Cause.** Often a mismatch in structure padding or alignment. arm64 has different padding rules from amd64 for some structures. An access that is aligned on amd64 may be misaligned on arm64 and panic.

**Fix.** Use `__packed` carefully (it changes alignment), use `__aligned(N)` where alignment matters, and avoid assuming the size or layout of a structure matches between architectures. For fields crossing the user/kernel boundary, use explicit widths (`uint32_t` rather than `int`, `uint64_t` rather than `long`).

### "驱动程序编译无错误但 dmesg 显示内核构建警告"

**Symptom.** The module builds, but loading it produces warnings about unresolved symbols or ABI mismatches.

**Cause.** The module was built against a different kernel than the one it is being loaded into. The kernel ABI is not guaranteed stable across versions, so a module built against 14.2 may not load cleanly on 14.3.

**Fix.** Rebuild the module against the running kernel's source tree. `uname -r` shows the running kernel version; verify that `/usr/src` matches. If they do not, install the matching source (via `freebsd-update`, `svn`, or `git`, depending on your source distribution).

### "驱动程序间歇性地比预期慢"

**Symptom.** Benchmarks show occasional large latency spikes even under moderate load.

**Cause.** Often a lock-contention issue: multiple threads queue on a single mutex. Sometimes an allocator stall: `malloc(M_WAITOK)` in a hot path waits for memory to become available.

**Fix.** Use `dtrace` to profile lock contention (`lockstat` provider) and identify which lock is hot. Restructure to reduce the critical section, split the lock, or use a lock-free approach. For allocator stalls, preallocate or use a UMA zone with a high-water mark.

## 驱动程序代码审查安全检查清单

This section is a reference checklist you can keep next to your code as you review a driver, yours or someone else's. It is not exhaustive, but if every item on the list has been consciously considered, the driver is in much better shape than the average.

### 结构检查

The driver's module-load and module-unload paths are symmetric. Every resource allocated on load is freed on unload, and the order of freeing is the reverse of the order of allocation.

The driver uses `make_dev_s` or `make_dev_credf` (not the legacy `make_dev` alone) so that errors during device-node creation are reported and handled.

The device node is created with conservative permissions. Mode `0600` or `0640` is the default; anything more permissive has an explicit reason recorded in comments or commit messages.

The driver declares a named `malloc_type` via `MALLOC_DECLARE` and `MALLOC_DEFINE`. All allocations use this type.

Every lock in the driver has a comment next to its declaration saying what it protects. The comment is accurate.

### 输入与边界检查

Every `copyin` call is paired with a size argument that cannot exceed the destination buffer size.

Every `copyout` call uses a length that is the minimum of the caller's buffer size and the kernel source's size.

`copyinstr` is used for strings that should be NUL-terminated. The return value (including `done`) is checked.

Every ioctl argument structure is copied into kernel space before any of its fields are read.

`uiomove` calls pass a length that is clamped to the buffer being read from or written to, not `uio->uio_resid` alone.

Every user-provided length field is validated: non-zero when required, bounded below the appropriate maximum, checked against remaining buffer space.

### 内存管理

Every `malloc` call checks the return value if `M_NOWAIT` is used. `M_WAITOK` without `M_NULLOK` is never null-checked uselessly; the code relies on the allocator's guarantee.

Every `malloc` is paired with exactly one `free` on every code path. Success paths and error paths are both audited.

Sensitive data (keys, passwords, credentials, proprietary secrets) is zeroed with `explicit_bzero` or `zfree` before the memory is released.

Structures that will be copied to user space are zeroed before being filled.

Buffers allocated for user output use `M_ZERO` at allocation time to prevent stale-data leaks through the tail.

After a pointer is freed, it is either set to NULL or the scope immediately ends.

### 权限与访问控制

Operations that require administrative privilege call `priv_check(td, PRIV_DRIVER)` or a more specific `PRIV_*` constant.

Operations that should not be allowed inside a jail explicitly check `jailed(td->td_ucred)` and return `EPERM` if jailed.

Operations that depend on the system's securelevel call `securelevel_gt` or `securelevel_ge` and handle the return value correctly (note the inverted semantics: nonzero means refuse).

No operation uses `cr_uid == 0` as a privilege gate. `priv_check` is used instead.

Sysctls that expose sensitive data use `CTLFLAG_SECURE` or restrict themselves to privileged users via permission checks.

### 并发

Every field of the softc that is accessed by more than one context is protected by a lock.

The full check-and-act sequence (including lookups that decide whether an operation is legal) is held under the appropriate lock.

No `copyin`, `copyout`, or `uiomove` call is made while holding a mutex. If user-space I/O is needed, the code drops the lock, does the I/O, and re-acquires, checking invariants.

`detach` calls `destroy_dev` (or equivalent) first, then drains callouts, taskqueues, and interrupts, then frees state.

Callouts, taskqueues, and kernel threads are tracked so that every one of them can be drained during unload.

### 信息卫生

No kernel pointer (`%p` or equivalent) is returned to user space through an ioctl, sysctl, or log message in a user-triggerable path.

No user-triggerable log message is uncapped; `ppsratecheck` or similar wraps every such message.

Logs do not include user-supplied data that could contain control characters or sensitive information.

Debug logging is wrapped in a conditional (sysctl or compile-time) so that production builds do not emit it by default.

### 故障模式

Every switch statement has a `default:` branch that returns a sensible error.

Every parser or validator whitelists what is allowed, rather than blacklisting what is not.

Every operation with resource use has a cap. The cap is documented.

Every sleep has a finite timeout unless a genuine reason requires unbounded waiting (and even then, `PCATCH` is used to allow signals).

Every error path frees the resources its success path would have kept.

The driver's response to unexpected input is to refuse the operation, not to guess.

### 测试

The driver has been loaded and tested against a kernel built with `INVARIANTS` and `WITNESS`. No assertions fire and no lock-order violations are reported.

The driver has been tested under concurrent load (multiple processes, multiple open file descriptors, interleaved operations).

The driver has been tested under detach-time concurrency (a user is inside the driver while unload is attempted).

Some form of fuzzing (ideally syzkaller, at minimum a randomized shell test) has been run against the driver.

The driver has been reviewed by someone other than its author. The review was specifically for security considerations, not only functionality.

### 演进

The driver's security posture is re-examined at regular intervals. New compiler warnings and new sanitizer findings are triaged seriously. New FreeBSD privilege codes are considered. Unused interfaces are removed.

Bug reports against the driver are treated as possibly exploitable until proven otherwise.

Commit history shows that security-relevant changes receive careful commit messages that explain what was wrong and what the fix does.

## 深入了解真实世界的漏洞模式

The principles in this chapter are abstractions over real bugs that happened in real kernels. This section studies a few patterns that have appeared across the FreeBSD, Linux, and other open-source operating systems over the years. The goal is not to catalogue CVEs (there are whole databases for that) but to train pattern recognition.

### 不完整的复制

A classic pattern: a driver receives a variable-length user buffer. It copies a fixed header, extracts a length field from the header, then copies the variable portion according to that length.

```c
error = copyin(uaddr, &hdr, sizeof(hdr));
if (error != 0)
    return (error);

if (hdr.body_len > MAX_BODY)
    return (EINVAL);

error = copyin(uaddr + sizeof(hdr), body, hdr.body_len);
```

The bug is that the length check compares `body_len` against `MAX_BODY`, but `body` may be a fixed-size buffer sized differently. If `MAX_BODY` is defined carelessly, or if it was once the size of `body` but `body` has since shrunk, the copy overflows `body`.

Every time you see a pattern of "validate header, then copy body based on header", check that the length bound actually matches the destination buffer size. Use `sizeof(body)` directly if you can, rather than a macro that might drift.

### 符号混淆

A length is stored as `int` but should be non-negative. A caller passes `-1`. Your code:

```c
if (len > MAX_LEN)
    return (EINVAL);

buf = malloc(len, M_FOO, M_WAITOK);
copyin(uaddr, buf, len);
```

Does the first check pass? Yes, because `-1` is less than `MAX_LEN` when compared as a signed `int`. What happens in `malloc(len, ...)` with `len = -1`? On many platforms, `-1` silently becomes a very large positive `size_t`. The allocation fails (or worse, succeeds at a huge size), or `copyin` tries to copy a huge buffer.

The fix is to use unsigned types for sizes (preferably `size_t`), or to check for negative values explicitly:

```c
if (len < 0 || len > MAX_LEN)
    return (EINVAL);
```

Or, better, change the type so that negative values cannot exist:

```c
size_t len = arg->len;     /* copied from user, already size_t */
if (len > MAX_LEN)
    return (EINVAL);
```

Sign confusion is one of the most common root causes of buffer overflows in kernel code. Use `size_t` for sizes. Use `ssize_t` only when negative values are meaningful. Never mix signed and unsigned in a size check.

### 不完整的验证

A driver accepts a complex structure with many fields. The validation function checks some fields but forgets others:

```c
if (args->type > TYPE_MAX)
    return (EINVAL);
if (args->count > COUNT_MAX)
    return (EINVAL);
/* forgot to validate args->offset */

use(args->offset);  /* attacker-controlled */
```

The bug is that `args->offset` is used as an index into an array without being bounds-checked. An attacker supplies a huge offset and reads or writes kernel memory.

The fix is to treat validation as a checklist. For every field in the input structure, ask: what values are legal? Enforce them all. A helper function `is_valid_arg` that centralizes the validation and is called early is better than scattered checks.

### 错误路径上跳过的检查

A driver carefully validates input on the success path, but the error path cleans up based on a field that was never validated:

```c
if (args->count > COUNT_MAX)
    return (EINVAL);
buf = malloc(args->count * sizeof(*buf), M_FOO, M_WAITOK);
error = copyin(args->data, buf, args->count * sizeof(*buf));
if (error != 0) {
    /* error cleanup */
    if (args->free_flag)          /* untrusted field */
        some_free(args->ptr);     /* attacker-controlled */
    free(buf, M_FOO);
    return (error);
}
```

The error path uses `args->free_flag` and `args->ptr`, neither of which were validated. If the attacker arranges for `copyin` to fail (say, by unmapping the memory), the error path frees an attacker-controlled pointer, corrupting the kernel heap.

The lesson: validation must cover every field that any code path reads. It is tempting to think "the error path is unusual; it is fine". Attackers specifically aim for error paths because they are less tested.

### 双重查找

A driver looks up an object in a table by name or ID, then performs an operation. Between the lookup and the operation, the object is removed by another thread. The operation then acts on freed memory.

```c
obj = lookup(id);
if (obj == NULL)
    return (ENOENT);
do_operation(obj);   /* obj may have been freed in between */
```

The fix is to take a reference on the object (using a refcount) inside the lookup, hold the reference across the operation, and release it at the end. The lookup function takes the lock, increments the refcount, and releases the lock. The operation then works with a refcount-held pointer that cannot be freed out from under it. The release decrements the refcount; when it drops to zero, the last holder frees the object.

Reference counts are the FreeBSD-canonical answer to the double-lookup problem. See `/usr/src/sys/sys/refcount.h`.

### 增长的缓冲区

A buffer was once 256 bytes. A constant `BUF_SIZE = 256` was defined. The code checked `len <= BUF_SIZE` and copied `len` bytes into the buffer. Later, someone increased the buffer to 1024 bytes but forgot to update the constant. Or the constant was updated but an `sizeof(buf)` in one call was not, because it was not using the constant.

This class of bug is prevented by always using `sizeof` on the destination buffer directly, rather than a constant that may drift:

```c
char buf[BUF_SIZE];
if (len > sizeof(buf))     /* always matches the actual buf size */
    return (EINVAL);
```

Constants are useful when multiple places need the same bound. If you use a constant, keep the definition and the array adjacent in the source code, and consider adding a `_Static_assert(sizeof(buf) == BUF_SIZE, ...)` to catch drift.

### 结构中未检查的指针

A driver receives a structure from user space that contains pointers. The driver uses the pointers directly:

```c
error = copyin(uaddr, &cmd, sizeof(cmd));
/* cmd.data_ptr is user-space pointer */
use(cmd.data_ptr);   /* treating user pointer as kernel pointer */
```

This is a catastrophic bug: the pointer is a user-space address, but the code dereferences it as if it were kernel memory. On some architectures this may access whatever memory happens to be at that address in kernel space, which is usually garbage or invalid. On others, it faults. In some specific pathological cases, it accesses sensitive kernel data.

The fix: never dereference a pointer obtained from user space. Pointers in user-supplied structures must be passed to `copyin` or `copyout`, which correctly translate user addresses. Never treat them as kernel addresses.

### 被遗忘的 copyout

A driver reads a structure from user space, modifies it, but forgets to copy the modified version back:

```c
error = copyin(uaddr, &cmd, sizeof(cmd));
if (error != 0)
    return (error);

cmd.status = STATUS_OK;
/* forgot to copyout */
return (0);
```

This is a functional bug, not strictly a security bug, but its mirror image is: forgetting `copyin` and assuming a field was already set. "I set `cmd.status` in `copyin`, then I read it later" is wrong if the field was actually set by user space; the user's value is what the code reads.

Every structure that flows user-to-kernel and back needs a clear convention about when `copyin` and `copyout` happen, and what fields are authoritative in which direction. Document it and follow it.

### 意外的竞态

A driver takes a lock, reads a field, releases the lock, and then uses the value:

```c
mtx_lock(&sc->sc_mtx);
val = sc->sc_val;
mtx_unlock(&sc->sc_mtx);

/* ... some unrelated work ... */

mtx_lock(&sc->sc_mtx);
if (val == sc->sc_val) {
    /* act on val */
}
mtx_unlock(&sc->sc_mtx);
```

The driver assumes `val` is still current because it re-checks. But "act on val" uses the stale copy, not the current field. If `sc_val` is a pointer, the act may operate on a freed object. If `sc_val` is an index, the act may use a stale index.

The lesson: once you release a lock, any value you read under that lock is stale. If you need to re-act under the lock, re-read the state inside the re-acquisition. The `if (val == sc->sc_val)` check protects against changes; the act needs to use the current value, not the stored one.

### 静默截断

A driver receives a string of up to 256 bytes, stores it in a 128-byte buffer. The code uses `strncpy`:

```c
strncpy(sc->sc_name, user_name, sizeof(sc->sc_name));
```

`strncpy` stops at the destination size. But `strncpy` does not guarantee a NUL terminator if the source was longer. Later code does:

```c
printf("name: %s\n", sc->sc_name);
```

`printf("%s", ...)` reads until a NUL. If `sc_name` is not NUL-terminated, printf reads past the end of the array into adjacent memory, potentially leaking that memory in the log or crashing.

Safer options: `strlcpy` (guarantees NUL termination, truncates if needed), or `snprintf` (same guarantee with formatting). `strncpy` is a landmine; it is in the standard library only for historical reasons.

### 过度记录的事件

A driver logs every time an event fires. The event is user-triggerable. A user sends a million events in a loop. The kernel message buffer fills and overflows; legitimate messages are lost. The user has accomplished a denial-of-service on the logging subsystem itself.

The fix, as discussed in Section 8, is rate limiting. Every user-triggerable log message should be wrapped in a rate-limit check. A suppressed-count summary ("[secdev] 1234 suppressed messages in last 5 seconds") can be emitted periodically to inform the operator of ongoing flooding.

### 隐形漏洞

A driver works fine for years. Then a compiler update changes how it handles a specific idiom, or a kernel API changes semantics in a new FreeBSD release, and the driver's behaviour changes. A check that used to work silently stops working. Users do not notice until an exploit appears.

Invisible bugs are the strongest argument for `KASSERT`, sanitizers, and tests. A `KASSERT(p != NULL)` at the top of every function documents what that function expects. An `INVARIANTS` kernel catches the moment an invariant breaks. A good test suite notices when behavior changes.

The simpler the function and the clearer its contract, the fewer places invisible bugs can hide. This is one reason the FreeBSD kernel coding style described in `style(9)` values short functions with clear responsibilities: they are easier to reason about, which makes invisible bugs easier to avoid in the first place.

### 漏洞模式目录总结

Each of the patterns above has been seen in real kernel code. Many have been CVEs. The defenses are:

- Use `size_t` for sizes; avoid sign confusion.
- Whitelist validation; do not forget fields.
- Treat error paths with the same rigor as success paths.
- Use refcounts to manage object lifetime under concurrency.
- Use `sizeof` directly on the buffer rather than a drift-prone constant.
- Never dereference user pointers.
- Keep the `copyin` / `copyout` story explicit per field.
- Remember that a value read under a lock is stale after the lock is released.
- Use `strlcpy` or `snprintf`, never `strncpy`.
- Rate-limit every user-triggerable log.
- Write invariants as `KASSERT` so regressions are caught.

Memorize these patterns. Apply them as a mental checklist on every function you write or review.

## 附录：本章使用的头文件和 API

A short reference to the FreeBSD headers referenced throughout this chapter, grouped by topic. Each header is in `/usr/src/sys/` followed by the path listed.

### 内存与复制操作

- `sys/systm.h`: declarations for `copyin`, `copyout`, `copyinstr`, `bzero`, `explicit_bzero`, `printf`, `log`, and many kernel core primitives.
- `sys/malloc.h`: `malloc(9)`, `free(9)`, `zfree(9)`, `MALLOC_DECLARE`, `MALLOC_DEFINE`, M_* flags.
- `sys/uio.h`: `struct uio`, `uiomove(9)`, UIO_READ / UIO_WRITE constants.
- `vm/uma.h`: UMA zone allocator (`uma_zcreate`, `uma_zalloc`, `uma_zfree`, `uma_zdestroy`).
- `sys/refcount.h`: reference-count primitives (`refcount_init`, `refcount_acquire`, `refcount_release`).

### 权限与访问控制

- `sys/priv.h`: `priv_check(9)`, `priv_check_cred(9)`, `PRIV_*` constants, `securelevel_gt`, `securelevel_ge`.
- `sys/ucred.h`: `struct ucred` and its fields.
- `sys/jail.h`: `struct prison`, `jailed(9)` macro, prison-related helpers.
- `sys/capsicum.h`: Capsicum capabilities, `cap_rights_t`, `IN_CAPABILITY_MODE(td)`.
- `security/mac/mac_framework.h`: MAC framework hooks (mostly for policy writers, but reference).

### 加锁与并发

- `sys/mutex.h`: `struct mtx`, `mtx_init`, `mtx_lock`, `mtx_unlock`, `mtx_destroy`.
- `sys/sx.h`: shared/exclusive locks.
- `sys/rwlock.h`: read/write locks.
- `sys/condvar.h`: condition variables (`cv_init`, `cv_wait`, `cv_signal`).
- `sys/lock.h`: common lock infrastructure.
- `sys/atomic_common.h`: atomic operations (and architecture-specific headers).

### 设备文件与 Dev 基础设施

- `sys/conf.h`: `struct cdev`, `struct cdevsw`, `struct make_dev_args`, `make_dev_s`, `make_dev_credf`, `destroy_dev`.
- `sys/module.h`: `DRIVER_MODULE`, `MODULE_VERSION`, kernel module declarations.
- `sys/kernel.h`: SYSINIT, SYSUNINIT, and related kernel hook macros.
- `sys/bus.h`: `device_t`, device methods, `bus_alloc_resource`, `bus_teardown_intr`.

### 定时、速率限制、调用

- `sys/time.h`: `eventratecheck(9)`, `ppsratecheck(9)`, `struct timeval`.
- `sys/callout.h`: `struct callout`, `callout_init_mtx`, `callout_reset`, `callout_drain`.
- `sys/taskqueue.h`: task queue primitives (`taskqueue_create`, `taskqueue_enqueue`, `taskqueue_drain`).

### 日志与诊断

- `sys/syslog.h`: `LOG_*` priority constants for `log(9)`.
- `sys/kassert.h`: `KASSERT`, `MPASS`, assertion macros.
- `sys/ktr.h`: KTR tracing macros.
- `sys/sdt.h`: Statically Defined Tracing probes for dtrace(1).

### Sysctl

- `sys/sysctl.h`: `SYSCTL_*` macros, `CTLFLAG_*` flags including `CTLFLAG_SECURE`, `CTLFLAG_PRISON`, `CTLFLAG_CAPRD`, `CTLFLAG_CAPWR`.

### 网络（适用时）

- `sys/mbuf.h`: `struct mbuf`, mbuf allocation and manipulation.
- `net/if.h`: `struct ifnet`, network interface primitives.

### 时代与无锁

- `sys/epoch.h`: epoch-based reclamation primitives (`epoch_enter`, `epoch_exit`, `epoch_wait`).
- `sys/atomic_common.h` and architecture-specific atomic headers: memory barriers, atomic reads and writes.

### 追踪与可观测性

- `security/audit/audit.h`: kernel audit framework (when compiled in).
- `sys/sdt.h`: Statically Defined Tracing for dtrace integration.
- `sys/ktr.h`: KTR in-kernel tracing.

This appendix is not exhaustive; the full set of headers a driver may need is far larger. It covers the ones referenced in this chapter. When writing your own driver, `grep` through `/usr/src/sys/sys/` for the primitive you need, and read the header to understand what is available. Many of these headers are well commented and repay careful reading.

Reading the headers is itself a security practice. Every primitive has a contract: what arguments it accepts, what constraints it imposes, what it guarantees on success, what it returns on failure. A driver that uses a primitive without reading its contract is relying on assumptions that may not hold. A driver that reads the contract, and holds itself to it, is a driver that benefits from the kernel's own discipline.

Many of the headers listed above are themselves worth studying as examples of good kernel design. `sys/refcount.h` is small, carefully commented, and demonstrates how a simple primitive is built from atomic operations. `sys/kassert.h` shows how conditional compilation is used to build a feature that costs nothing in production but catches bugs in developer kernels. `sys/priv.h` shows how a long list of named constants can be organized by subsystem and used as the grammar of a policy. When you run out of ideas for how to structure your own driver's internals, these headers are a good place to find inspiration.

## 附录：延伸阅读

A short list of resources that go deeper into FreeBSD security than this chapter can:

**FreeBSD Architecture Handbook**, in particular the chapters on the jail subsystem, Capsicum, and MAC framework. Available online at `https://docs.freebsd.org/en/books/arch-handbook/`.

**FreeBSD Handbook security chapter**, which is oriented toward administrators but includes useful context on how system-level features (jails, securelevel, MAC) interact.

**Capsicum: Practical Capabilities for UNIX**, the original paper by Robert Watson, Jonathan Anderson, Ben Laurie, and Kris Kennaway. Explains the design rationale behind Capsicum, which helps when deciding how your driver should behave in capability mode.

**"The Design and Implementation of the FreeBSD Operating System"**, by Marshall Kirk McKusick, George V. Neville-Neil, and Robert N. M. Watson. The second edition covers FreeBSD 11; many security-relevant chapters remain applicable in later versions.

**style(9)**, the FreeBSD kernel coding style guide, available as a manual page: `man 9 style`. Readable kernel code is safer kernel code; the conventions in `style(9)` are part of how the tree stays reviewable at scale.

**KASAN, KMSAN, and KCOV documentation** in `/usr/src/share/man/` and related sections. Reading these helps you configure and interpret sanitizer output.

**syzkaller documentation**, at `https://github.com/google/syzkaller`. The `sys/freebsd/` directory contains syscall descriptions that illustrate how to describe your own driver's interface.

**CVE databases** such as `https://nvd.nist.gov/vuln/search` or `https://cve.mitre.org/`. Searching for "FreeBSD" or specific driver names shows real bugs that have been found and fixed. Reading a few CVE reports per month teaches a great deal about what kinds of bugs occur in practice.

**FreeBSD security advisories**, at `https://www.freebsd.org/security/advisories/`. These are official reports on fixed vulnerabilities. Many are kernel-side and relevant to driver authors.

**The FreeBSD source tree itself** is the largest and most authoritative reference. Spend time reading drivers similar to yours. Look at how they validate input, check privilege, manage locking, and handle detach. Imitating the patterns you see in well-reviewed code is one of the fastest ways to learn.

**Security mailing lists**, such as `freebsd-security@` and the broader `oss-security` list, carry daily traffic on kernel and driver issues across open-source projects. Subscribing passively and skimming a few posts a week builds awareness of threat trends without demanding much effort.

**Formal verification literature**, although specialist, has begun to touch kernel code. Projects like seL4 demonstrate what a fully verified microkernel looks like. FreeBSD is not that, but reading about formal verification shapes how you think about invariants and contracts in your own code.

**Books on secure coding practices in C** such as `Secure Coding in C and C++` by Robert Seacord translate well to kernel work, since kernel C is a dialect of the same language and has the same pitfalls, plus more. Chapter-by-chapter, they provide the mental catalogue of bugs that this chapter could only sketch.

**FreeBSD-specific books**, notably the McKusick, Neville-Neil, and Watson book mentioned above, but also older volumes that cover the evolution of specific subsystems. Reading about how jails evolved, how Capsicum was designed, or how MAC came to be helps you understand the rationale behind the primitives rather than just their mechanics.

**Conference talks** from BSDCan, EuroBSDCon, and AsiaBSDCon often touch security topics. Video archives let you watch years of past talks at your own pace. Many talks are given by active FreeBSD developers and reflect current thinking.

**Academic papers on operating system security** from venues such as USENIX Security, IEEE S&P, and CCS provide a longer-term view. Not every paper is relevant to drivers, but the ones that are deepen your understanding of threat models, attacker capabilities, and the theoretical basis for mitigations.

**The CVE feed**, particularly when filtered for kernel issues, is a continuous drip of real-world examples. Reading a few each week builds intuition for what bugs look like in practice and which classes recur most often.

**Your own code, six months later**. Rereading your earlier work with the benefit of distance is a valuable learning tool. The bugs you will notice are the bugs you have learned to see since you wrote it. Make a habit of this; schedule time for it.

The resources above, even a small subset of them, will keep you growing for years. Security is a field of continuous learning. This chapter is one step in that learning; the next step is yours.

Every security-minded driver author should have read at least a few of these. The field moves, and staying current is part of the craft.

## 总结

设备驱动程序中的安全不是单一技术。它是一种工作方式。每一行代码都为内核的安全承担一点责任。本章涵盖了主要支柱：

**The kernel trusts every driver fully.** Once code runs in the kernel, there is no sandbox, no isolation, no second chance. The driver author's discipline is the system's last line of defense.

**Buffer overflows and memory corruption** are the classical kernel vulnerability. They are prevented by bounding every copy, preferring bounded string functions, and treating pointer arithmetic with suspicion.

**User input crosses a trust boundary.** Every byte from user space must be copied into the kernel with `copyin(9)`, `copyinstr(9)`, or `uiomove(9)` before it is used. Every byte going back must be copied out with `copyout(9)` or `uiomove(9)`. The user-space memory is not trustworthy; kernel memory is. Keep them cleanly separated.

**Memory allocation** must be checked, balanced, and accounted for. Always check `M_NOWAIT` returns. Use `M_ZERO` by default. Pair every `malloc` with exactly one `free`. Use a per-driver `malloc_type` for accountability. Use `explicit_bzero` or `zfree` for sensitive data.

**Races and TOCTOU bugs** are caused by inconsistent locking or by treating user-space data as stable. Fix them with locks around shared state and by copying user data before validating.

**Privilege checks** use `priv_check(9)` as the canonical primitive. Layer with jail awareness and securelevel where appropriate. Set conservative device-node permissions. Let the MAC and Capsicum frameworks work alongside.

**Information leaks** are prevented by zeroing structures before filling them, bounding copy lengths on both ends, and keeping kernel pointers out of user-visible output.

**Logging** is part of the driver's interface. Use it to help the operator without helping the attacker. Rate-limit anything triggerable from user space. Do not log sensitive data.

**Secure defaults** mean failing closed, whitelisting rather than blacklisting, setting conservative default values, and treating error paths with the same care as success paths.

**Testing and hardening** turn careful code into trustworthy code. Build with `INVARIANTS`, `WITNESS`, and the kernel sanitizers. Stress-test. Fuzz. Review. Re-test.

这些都不是一次性的努力。驱动程序保持安全是因为其作者在代码的整个生命周期中，每次提交、每个版本都持续应用这些习惯。

The discipline is not glamorous. It is boring work: zero the structure, check the length, acquire the lock, use `priv_check`. But this boring work is exactly what keeps systems secure. An exploited kernel is a catastrophic event for users. An exploited driver is a foothold into the kernel. The person at the keyboard of that driver, deciding whether to add the bounds check or to skip it, is making a security decision that may be invisible for years and then suddenly matter very much.

Be the author who adds the bounds check.

### 再一次反思：安全作为职业身份

Something worth saying explicitly: the habits in this chapter are not merely techniques. They are what distinguishes a journeyman kernel author from an apprentice. Every mature kernel engineer carries this mental checklist not because they memorized it but because they have, over years, internalized a skepticism toward their own code. The skepticism is not anxiety. It is discipline.

Write code, and then read it back as if a stranger had written it. Ask what happens if the caller is hostile. Ask what happens if the value is zero, or negative, or impossibly large. Ask what happens if the other thread arrives between these two statements. Ask what happens on the error path you did not plan to test. Write the check. Write the assertion. Move on.

This is what professional kernel engineers do. It is not glamorous, it is rarely applauded, and it is what keeps the operating system we all rely on from falling apart. The kernel is not magic; it is millions of lines of carefully checked code, written and rewritten by people who treat every line as a small responsibility. Joining that profession means joining that discipline.

You have now been given the tools. The rest is practice.

## 展望：设备树与嵌入式开发

This chapter trained you to look at your driver from the outside, through the eyes of whoever might try to misuse it. The boundaries you learned to watch were invisible to the compiler but very real to the kernel: user space on one side, kernel memory on the other; one thread with privilege, another without; a length field the caller claimed, a length the driver had to verify. Chapter 31 was about *who is allowed to ask the driver to do something*, and *what the driver should check before it agrees*.

Chapter 32 shifts the perspective entirely. The question stops being *who wants this driver to run* and becomes *how does this driver find its hardware at all*. On the PC-like machines we have leaned on so far, that question had a comfortable answer. PCI devices announced themselves through standard configuration registers. ACPI-described peripherals appeared in a table the firmware handed to the kernel. The bus did the looking, the kernel probed each candidate, and your driver's `probe()` function only had to look at an identifier and say yes or no. Discovery was mostly someone else's problem.

On embedded platforms that assumption breaks. A small ARM board does not speak PCI, does not carry an ACPI BIOS, and does not hand the kernel a neat table of devices. The SoC has an I2C controller at a fixed physical address, three UARTs at three other fixed addresses, a GPIO bank at a fourth, a timer, a watchdog, a clock tree, and a dozen other peripherals soldered onto the board in a particular arrangement. Nothing in the silicon announces itself. If the kernel is going to attach drivers to these peripherals, something has to tell the kernel where they are, what they are, and how they relate.

That something is the **Device Tree**, and Chapter 32 is where you learn to work with it. You will see how `.dts` source files describe the hardware, how the Device Tree Compiler (`dtc`) turns them into the `.dtb` blobs the bootloader hands to the kernel, and how FreeBSD's FDT support walks those blobs to decide which drivers to attach. You will meet the `ofw_bus` interfaces, the `simplebus` enumerator, and the Open Firmware helpers (`ofw_bus_search_compatible`, `ofw_bus_get_node`, the property-reading calls) that turn a Device Tree node into a working driver attachment. You will compile a small overlay, load it, and watch a pedagogical driver attach in `dmesg`.

The security habits you have built in this chapter travel with you into that territory. A driver for an embedded board is still a driver: it still runs in kernel space, still copies data across user-space boundaries, still needs bounds checks, still takes locks, still cleans up in detach. An ARM board does not loosen any of those requirements. If anything, embedded systems raise the stakes, because the same board image may ship to thousands of devices in the field, each one harder to patch than a server in a data center. The disposition you have just learned, skeptical of inputs, careful with memory, conservative about privilege, is exactly the disposition an embedded driver author needs.

What changes in Chapter 32 is the set of helpers you call to discover your hardware and the files you read to know where to point them. The probe-attach-detach shape stays. The softc stays. The lifecycle stays. A handful of new calls and a new way of thinking about hardware description are what you add. The chapter builds them up gently, from the shape of a `.dts` file to a working driver that blinks an LED on a real or emulated board.

See you there.

## 关于习惯的最终说明

This chapter has been longer than some. The length is deliberate. Security is not a topic that can be summarized into a single punchy rule; it is a way of thinking that requires examples, practice, and repetition. A reader who finishes this chapter once will have been exposed to the patterns. A reader who returns to this chapter when starting a new driver will find new meaning in passages that seemed merely informative on the first read.

Here are the most important habits, condensed into a single list for you to carry forward. They are the reflexes that matter most in daily driver work:

Every user-space value is hostile until copied in, bounded, and validated.

Every length has a maximum. The maximum is enforced before anything uses the length.

Every structure copied to user space is zeroed first.

Every allocation is paired with a free on every code path.

Every critical section is held across the full check-and-act sequence it protects.

Every privilege-sensitive operation checks `priv_check` before acting.

Every detach path drains async work before freeing state.

Every log message triggerable from user space is rate-limited.

Every unknown input returns an error, never a silent success.

Every assumption worth making is worth writing as a `KASSERT`.

Nine lines. If these become automatic, you have the core of what this chapter teaches.

The craft grows from here. There are more patterns, more subtleties, more tools; you will encounter them as you read more FreeBSD source, as you review more code, as you write more drivers. What stays the same is the disposition: skeptical of hostile inputs, careful with memory, clear about lock boundaries, conservative about what to expose. That disposition is the one kernel engineers share across decades. You have it now. Use it well.

## 关于不断演变的威胁的说明

One further thought before the closing words. The threats we defend against today are not the threats we will defend against in ten years. Attackers evolve. Mitigations evolve. New classes of bugs are discovered, old classes are retired. A driver that was state-of-the-art in its defenses in 2020 may need updating to be considered safe in 2030.

This is not a reason for despair. It is a reason for continuous learning. Every year, a responsible driver author should read a few new security papers, try a few new sanitizers, and look at the recent CVEs affecting kernels similar to their own. Not to memorize specific vulnerabilities, but to keep a sense of where the bugs are being found today.

The patterns this chapter teaches are stable. Buffer overflows have been bugs since before UNIX. Use-after-free has been a bug since C had malloc. Race conditions have been bugs since kernels had multiple threads. The specific incarnations change, but the underlying defenses endure. A driver written with the disposition this chapter encourages will be mostly right in any decade; when the details shift, the author who built the disposition will adapt faster than one who merely memorized a checklist.

## 结语

A driver is small. A driver's influence is large. The code you write runs in the most privileged part of the system, touches memory that every other process depends on, and is trusted with the secrets of users who will never see your name. That trust is not automatic; it is earned, one careful line at a time, by authors who assumed the attacker was watching and built accordingly.

The authors of FreeBSD have been writing that kind of code for decades. The FreeBSD kernel is not perfect; no kernel of its scale can be. But it has a culture of care, a set of primitives that reward diligence, and a community that treats security bugs as learning opportunities rather than embarrassments. When you write a FreeBSD driver, you are writing into that culture. Your code will be read by people who know the difference between a buffer overflow and a buffer that happens to be large enough; who know the difference between a privilege check that catches root-outside-jail and one that catches root-inside-jail; who know that a race condition is not a rare timing fluke but a vulnerability waiting for the right attacker.

Write for those readers. Write for the user whose laptop runs your code without knowing it is there. Write for the maintainer who will inherit your work in ten years. Write for the reviewer who will spot the defensive check you added and feel quietly glad that someone thought of it.

That is what chapter 31 has been about. That is what the rest of your career as a kernel author will be about. Thank you for taking the time to work through it carefully. The chapter ends here; the practice begins tomorrow.
