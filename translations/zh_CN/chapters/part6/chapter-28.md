---
title: "编写网络驱动程序"
description: "为 FreeBSD 开发网络接口驱动程序"
partNumber: 6
partName: "编写特定传输层驱动程序"
chapter: 28
lastUpdated: "2026-04-19"
status: "complete"
author: "Edson Brandi"
reviewer: "TBD"
translator: "AI辅助翻译为简体中文"
estimatedReadTime: 240
language: "zh-CN"
---
# 编写网络驱动程序

## 引言

在上一章中，你构建了一个存储驱动程序。文件系统位于其上，缓冲区缓存向它提供 BIO 请求，你的代码将数据块传送到一块 RAM 并返回。这已经与前面章节中的字符设备世界有所不同，因为存储驱动程序不是由持有文件描述符的单个进程轮询的。它由上方的许多层驱动，所有层协作将 `write(2)` 调用转化为持久化的块，你的驱动程序必须安静地坐在那条链的底部，依次处理每个请求。

网络驱动程序是第三种动物。它不是像字符设备那样面向一个进程的字节流，也不是像存储设备那样供文件系统挂载的块可寻址表面。它是一个**接口**。它位于机器的网络协议栈的一侧和传输介质（真实的或模拟的）的另一侧之间。数据包到达该介质时，驱动程序将它们转换为 mbuf 并向上传递给协议栈。数据包以 mbuf 的形式离开协议栈时，驱动程序将它们转换为线路上的比特，或者转换为你选择使用的任何线路替代物。链路状态变化时驱动程序会报告，媒体速度变化时驱动程序也会报告。用户输入 `ifconfig mynet0 up`，内核通过 `if_ioctl` 将该请求路由到你的代码中。内核期望的是一种特定形式的合作，而不是特定的读写序列。

本章将教你那种合作的形式。你将学习 FreeBSD 期望网络驱动程序是什么样的，学习内核中表示接口的核心对象——名为 `ifnet` 的结构体，以及包装它的现代 `if_t` 不透明句柄。你将学习如何分配 `ifnet`，如何向协议栈注册它，如何将其暴露为 `ifconfig` 可以看到的命名接口。你将学习数据包如何通过发送回调进入你的驱动程序，以及你如何通过 `if_input` 将数据包反向推入协议栈。你将学习 mbuf 如何携带这些数据包，链路状态和媒体状态如何报告，`IFF_UP` 和 `IFF_DRV_RUNNING` 等标志如何使用，以及驱动程序在卸载时如何干净地分离。在本章结束时，你将拥有一个名为 `mynet` 的可用伪以太网驱动程序，你可以加载、配置、用 `ping`、`tcpdump` 和 `netstat` 进行测试，然后卸载而不留下任何痕迹。

你将要构建的驱动程序故意保持小型。现代 FreeBSD 中真正的以太网驱动程序通常基于 `iflib(9)` 编写，这是一个共享框架，负责处理大多数生产网卡上的环形缓冲区、中断调节和数据包调度。当你为 100Gb 网卡编写驱动程序时，这套机制非常出色，我们将在后续章节中回到这个话题。但它的脚手架太多了，会掩盖核心思想。为了教你网络驱动程序到底是什么，我们将编写经典的、iflib 之前的形式：一个拥有自己的发送函数和接收路径的普通 `ifnet` 驱动程序。一旦你清楚地理解了这一点，iflib 就像是在你已经了解的东西上面增加了一层便利。

与第27章一样，本章很长，因为主题是分层的。与 `/dev` 驱动程序不同，网络驱动程序被包裹在自己的词汇表中：以太网帧、接口克隆器、链路状态、媒体描述符、`if_transmit`、`if_input`、`bpfattach`、`ether_ifattach`。我们将一次一个概念地仔细介绍这些词汇，并将每个概念建立在真实 FreeBSD 源代码树中的代码基础上。你将看到 `epair(4)`、`disc(4)` 以及 UFS 栈中我们可以为自己驱动程序借鉴的模式。到最后，你将能够在任何 FreeBSD 源文件中识别出网络驱动程序的结构。

目标不是一个生产级网卡驱动程序。目标是给你一个完整的、诚实的、正确的理解，关于硬件和 FreeBSD 网络协议栈之间的那层，通过文字、代码和实践逐步建立。一旦这个心智模型稳固了，阅读 `if_em.c`、`if_bge.c` 或 `if_ixl.c` 就变成了识别模式和查找不熟悉部分的事情。没有这个心智模型，它们看起来就像宏和位操作的风暴。有了它，它们看起来就像另一个驱动程序，做着与你的 `mynet` 驱动程序相同的事情，只是底层有硬件而已。

慢慢来。阅读时打开一个 FreeBSD shell。保持一本实验日志。不要把网络协议栈想象成你代码上方的一个黑盒，而要把它想象成一个期望与驱动程序进行清晰的、契约式握手的对等方。你的工作是干净地履行那个契约。

## 读者指南：如何使用本章

本章延续了第27章中建立的模式：篇幅长、内容累积递进、节奏经过精心设计。这个主题是新的，词汇也是新的，所以在让你输入代码之前，我们会比平时更谨慎地处理开头的几个小节。

如果你选择**仅阅读路线**，请计划大约两到三个集中注意力的小时。你将获得一个清晰的心智模型，了解网络驱动程序是什么、它如何融入 FreeBSD 的网络协议栈、以及真实驱动程序中的代码在做什么。这是第一次使用本章时完全合理的方式，在你没有时间重新构建内核模块的日子里，这通常是正确的选择。

如果你选择**阅读加实验路线**，请计划大约五到八个小时，分布在一个或两个晚上。你将编写、构建和加载一个可用的伪以太网驱动程序，用 `ifconfig` 启动它，观察它的计数器移动，用 `ping` 向它发送数据包，用 `tcpdump` 查看它们，然后关闭一切并干净地卸载模块。这些实验设计为在任何最新的 FreeBSD 14.3 系统上都是安全的，包括虚拟机。

如果你选择**阅读加实验加挑战路线**，请计划一个周末或几个晚上。这些挑战将驱动程序扩展到实际中有意义的方向：添加一个真正的模拟链路伙伴并在两个接口之间共享队列，支持不同的链路状态，暴露一个 sysctl 来注入错误，以及在 `iperf3` 下测量行为。每个挑战都是独立的，只使用本章已经涵盖的内容。

无论你选择哪条路线，都不要跳过接近末尾的故障排除部分。网络驱动程序以少数几种特征方式失败，学会识别这些模式从长远来看比记住 `ifnet` 中每个函数的名称更有价值。故障排除材料放在后面是为了可读性，但你可能会在运行实验时发现自己需要回头查阅它。

关于先决条件的一点说明。你应该对第26章和第27章的所有内容感到自如：编写内核模块、分配和释放 softc、推理加载和卸载路径、以及在 `kldload` 和 `kldunload` 下测试你的工作。你还应该对 FreeBSD 的用户态足够熟悉，能够在不停下来检查标志的情况下运行 `ifconfig`、`netstat -in`、`tcpdump` 和 `ping`。如果其中任何内容感觉不确定，快速浏览前面相应的章节将在以后节省你的时间。

你应该在一台一次性的 FreeBSD 14.3 机器上工作。专用的虚拟机是最好的选择，因为网络驱动程序本质上会与主机系统的路由表和接口列表交互。一个小型实验虚拟机让你可以实验而不用担心会搞乱你的主系统。开始之前的快照是廉价的保险。

### 按小节顺序学习

本章按照渐进的方式组织。第1节解释网络驱动程序做什么以及它与你已经编写的字符驱动程序和存储驱动程序有何不同。第2节介绍 `ifnet` 对象，这是整个网络子系统的核心数据结构。第3节逐步介绍接口的分配、命名和注册，包括接口克隆器。第4节处理发送路径，从 `if_transmit` 到 mbuf 处理。第5节处理接收路径，包括 `if_input` 和模拟数据包生成。第6节涵盖媒体描述符、接口标志和链路状态通知。第7节展示如何使用 FreeBSD 的标准网络工具测试驱动程序。第8节以干净的分离、模块卸载和重构建议结束。

你应该按顺序阅读这些小节。每一节都假设前面的小节在你的脑海中还记忆犹新，实验也是相互构建的。如果你跳到中间，内容会看起来很奇怪。

### 动手输入代码

动手输入仍然是内化内核习语最有效的方式。`examples/part-06/ch28-network-driver/` 下的配套文件是为了让你检查你的工作，而不是让你跳过输入。阅读代码和编写代码是不同的，而阅读网络驱动程序特别容易被动地进行，因为代码通常看起来像一个很长的 switch 语句。编写它迫使你思考每个分支。

### 打开 FreeBSD 源代码树

你将被多次要求打开真实的 FreeBSD 源代码文件，而不仅仅是配套示例。本章感兴趣的文件包括 `/usr/src/sys/net/if.h`、`/usr/src/sys/net/if_var.h`、`/usr/src/sys/net/if_disc.c`、`/usr/src/sys/net/if_epair.c`、`/usr/src/sys/net/if_ethersubr.c`、`/usr/src/sys/net/if_clone.c`、`/usr/src/sys/net/if_media.h` 和 `/usr/src/sys/sys/mbuf.h`。这些文件都是主要参考资料，本章的文字会反复引用它们。如果你还没有克隆或安装 14.3 源代码树，现在是一个好时机。

### 使用实验日志

工作时请保持你在第26章中开始的日志打开。你需要记录加载模块前后的 `ifconfig` 输出、你用来发送流量的确切命令、`netstat -in` 报告的计数器、`tcpdump -i mynet0` 的输出以及任何警告或崩溃。网络工作特别适合使用日志，因为同一个命令 `ifconfig mynet0` 在加载-配置-使用-卸载周期的不同点会产生不同的输出，在你自己的笔记中看到这些差异会让概念更加牢固。

### 把握节奏

如果在某个特定小节你的理解变得模糊，停下来。重新阅读前一个小节。尝试一个小实验，例如 `ifconfig lo0` 或 `netstat -in` 来查看一个真实的接口，然后思考它与本章所教内容的对应关系。内核中的网络编程回报缓慢而刻意的接触。为了以后识别术语而浏览本章远不如好好阅读一个小节、做一个实验、然后继续来得有用。

## 如何从本章获得最大收益

本章的结构是每一节在前一节的基础上恰好增加一个新概念。为了充分利用这种结构，请把本章当作一个工作坊而不是参考手册。你不是来这里找快速答案的。你是来建立一个正确的心智模型的，了解接口是什么、驱动程序如何与内核对话、以及网络协议栈如何回应。

### 按小节学习

不要不停地从头到尾阅读整个章节。读一节，然后暂停。尝试与之配套的实验或练习。查看相关的 FreeBSD 源代码。在你的日志中写几行。然后才继续。内核中的网络编程是强烈累积的，跳过通常意味着你会因为两节之前解释过的原因而对下一个内容感到困惑。

### 保持驱动程序运行

一旦你在第3节中加载了驱动程序，在阅读时尽量保持它处于加载状态。修改它，重新加载，用 `ifconfig` 触碰它，用 `ping` 向它发送数据包，用 `tcpdump` 观察它们。拥有一个活的、可观察的例子远比任何数量的阅读更有价值，特别是对于网络代码，因为反馈循环很快：内核要么接受你的配置，要么拒绝它，计数器要么移动，要么不移动。

### 参阅手册页

FreeBSD 的手册页是教学材料的一部分，不是一种单独的形式。手册的第9节是内核接口所在的地方。本章将引用 `ifnet(9)`、`mbuf(9)`、`ifmedia(9)`、`ether(9)` 和 `ng_ether(4)` 等页面，以及 `ifconfig(8)`、`netstat(1)`、`tcpdump(1)`、`ping(8)` 和 `ngctl(8)` 等用户态页面。请与本章一起阅读它们。它们比你想象的要短，而且它们是由编写你正在学习的内核的同一社区编写的。

### 输入代码，然后修改它

当你从配套示例构建驱动程序时，首先输入它。一旦它工作了，开始修改。重命名一个方法，观察构建失败。移除发送函数中的一个 `if` 分支，观察在 `ping` 下会发生什么。硬编码一个更小的 MTU，观察 `ifconfig` 的反应。内核代码通过刻意的变异远比通过纯粹的阅读变得可理解，而网络代码特别适合变异，因为每个更改都会在 `ifconfig` 或 `netstat` 中产生立即可见的效果。

### 善用工具

FreeBSD 为你提供了丰富的工具来检查网络协议栈：`ifconfig`、`netstat`、`tcpdump`、`ngctl`、`sysctl net.`、`arp`、`ndp`。使用它们。当出现问题时，第一步几乎绝不是阅读更多源代码。而是询问系统处于什么状态。用 `ifconfig mynet0` 和 `netstat -in` 检查一分钟通常比五分钟的 `grep` 更有信息量。

### 适当休息

网络代码充满了小的、精确的步骤。一个遗漏的标志或一个未设置的回调会产生看起来神秘的行为，直到你停下来、深呼吸、再次追踪数据流。两到三个集中注意力的小时通常比七小时的冲刺更有效率。如果你发现自己犯了三次同样的打字错误，或者在没有阅读的情况下复制粘贴，那就是你该站起来休息十分钟的信号。

有了这些习惯，让我们开始吧。

## 第1节：网络驱动程序做什么

网络驱动程序有一个听起来简单但实际上是分层的任务：它在传输层和 FreeBSD 网络协议栈之间移动数据包。其他一切都由此而来。要理解这句话的真正含义，我们需要放慢速度，检查它的每个部分。什么是数据包？什么是传输层？协议栈到底是什么？驱动程序如何坐在它们之间而不成为瓶颈或微妙错误的来源？

### 内核中的数据包

在用户态中，你很少处理原始数据包。你打开一个套接字，调用 `send` 或 `recv`，内核负责将你的有效载荷封装在 TCP 中，将其包装在 IP 中，添加以太网头部，最后将整个构造交给驱动程序。在内核中，相同的数据包由一个名为 **mbuf** 的结构链表表示。mbuf 是一个小的内存单元，通常为256字节，保存数据包数据和一个小的头部。如果数据包大于单个 mbuf 可以容纳的大小，内核通过 `m_next` 指针将多个 mbuf 链接在一起，有效载荷的总长度记录在 `m->m_pkthdr.len` 中。如果数据包不适合单个 mbuf 集群，内核使用由 mbuf 引用的外部缓冲区，这是我们在后面章节中将重新讨论的机制。

从驱动程序的角度来看，数据包几乎总是以 mbuf 链的形式呈现，第一个 mbuf 携带数据包头部。第一个 mbuf 的标志中设置了 `M_PKTHDR`，这告诉你 `m->m_pkthdr` 包含有效字段，如总数据包长度、VLAN 标签、校验和标志和接收接口。每个处理发送数据包的驱动程序都从检查交给它的 mbuf 开始，每个交付接收数据包的驱动程序都从构建一个正确形状的 mbuf 开始。

我们将在第4节和第5节中更详细地介绍 mbuf 的构建和拆卸。现在，词汇才是最重要的。一个 mbuf 就是一个数据包。一个 mbuf 链是一个有效载荷跨越多个 mbuf 的数据包。链中的第一个 mbuf 携带数据包头部。链的其余部分继续有效载荷，每个 mbuf 通过 `m_next` 指向下一个。

### 传输层

传输层是驱动程序在硬件端与之通信的对象。对于物理以太网网卡，它是实际的线路，通过 DMA 缓冲区、硬件环和芯片中断的组合来访问。对于 USB 以太网适配器，它是我们在第26章中介绍的 USB 端点管道。对于无线网卡，它是无线电。对于伪设备——也就是我们将在本章中构建的——传输层是模拟的：我们将假装我们发送的数据包出现在某条其他虚拟线路上，我们将假装传入的数据包以由定时器驱动的规则间隔从中到达。

`ifnet` 抽象的美妙之处在于网络协议栈不关心你有哪种传输层。协议栈看到一个接口。它给接口 mbuf 来发送。它期望接口交给它已接收的 mbuf。无论数据包实际是通过六类电缆、无线电波、USB 总线还是我们控制的一块内存传输的，表面都是相同的。这种统一性是让 FreeBSD 支持数十种网络设备而无需为每种设备重写网络代码的原因。

### 网络协议栈

「协议栈」是位于驱动程序之上并实现协议的代码集合的简称。从最低层到最高层依次为：以太网成帧、ARP 和邻居发现、IPv4 和 IPv6、TCP 和 UDP、套接字缓冲区，以及将 `send` 和 `recv` 转换为协议栈操作的系统调用层。在 FreeBSD 中，代码位于 `/usr/src/sys/net/`、`/usr/src/sys/netinet/`、`/usr/src/sys/netinet6/` 和相关目录中，它通过每个 `ifnet` 上携带的一小组明确定义的函数指针与驱动程序通信。

对于本章，你不需要了解协议栈的内部。你需要了解驱动程序所看到的外部接口。该接口是：

* 协议栈调用你的发送函数 `if_transmit`，并交给你一个 mbuf。你的工作是将该 mbuf 转换为传输层可以接受的内容。
* 协议栈调用你的 ioctl 处理程序 `if_ioctl`，以响应用户态命令，例如 `ifconfig mynet0 up` 或 `ifconfig mynet0 mtu 1400`。你的工作是履行该请求或返回一个合理的错误。
* 当接口转换为 up 状态时，协议栈调用你的初始化函数 `if_init`。你的工作是准备传输层以供使用。
* 你调用 `ifp->if_input(ifp, m)` 或使用现代写法 `if_input(ifp, m)` 将接收到的数据包交给协议栈。你的工作是确保 mbuf 格式正确且数据包完整。

这就是契约。其余的都是细节。

### 网络驱动程序与字符驱动程序的区别

你已经在第14章和第18章中构建了字符驱动程序。字符驱动程序位于 `/dev/` 内部，由用户态通过 `open(2)` 打开，并通过 `read(2)` 和 `write(2)` 与一个或多个进程交换字节。它有一个 `cdevsw` 表。由打开它的任何人轮询和推送。

网络驱动程序不是这些东西中的任何一个。它不在 `/dev/` 中。不是由进程 `open(2)` 的。没有 `cdevsw`。网络接口最接近用户可见文件句柄的东西是绑定到它的套接字，即使那也是由协议栈而不是由驱动程序中介的。

网络驱动程序没有 `cdevsw`，而是有一个 `struct ifnet`。它没有 `d_read`，而是有 `if_input`，但在另一端：驱动程序调用它，而不是由用户态调用。它没有 `d_write`，而是有 `if_transmit`，由协议栈调用。它没有 `d_ioctl`，而是有 `if_ioctl`，由协议栈响应 `ifconfig` 和相关工具时调用。顶层结构看起来相似，但参与者之间的关系不同。在字符驱动程序中，你等待来自用户态的读写。在网络驱动程序中，你嵌入在一个管道中，协议栈是你的主要合作者，而用户态是旁观者而不是直接的对等方。

这种视角的转变值得在你编写任何代码之前内化。当字符驱动程序出现问题时，问题通常是「用户态做了什么？」当网络驱动程序出现问题时，问题通常是「协议栈期望我的驱动程序做什么，我为什么没有做到？」

### 网络驱动程序与存储驱动程序的区别

正如你在第27章中看到的，存储驱动程序通常也不是 `/dev/` 端点。它确实暴露了一个块设备节点，但对其的访问几乎总是由位于其上的文件系统来中介。请求以 BIO 的形式下传，驱动程序处理它们，完成通过 `biodone(bp)` 发出信号。

网络驱动程序共享存储驱动程序的「我坐在子系统下面，而不是用户态旁边」的形式，但它上方的子系统非常不同。存储子系统在 BIO 层面是深度同步的，因为每个请求都有一个明确定义的完成事件。网络流量不是这样的。驱动程序发送一个数据包，但没有从驱动程序冒泡到任何特定请求者的逐包完成回调。协议栈信任驱动程序能够干净地成功或失败，递增计数器，然后继续。同样，接收到的数据包不是对特定早期发送的回复：它们只是到达，驱动程序必须随时将它们导入 `if_input`。

另一个区别是并发性。存储驱动程序通常有单个 BIO 路径并依次处理每个 BIO。网络驱动程序经常同时从多个 CPU 上下文被调用，因为协议栈并行服务许多套接字，而现代硬件在多个队列上交付接收事件。我们不会在本章中涵盖这种复杂性，但你已经应该意识到网络驱动程序的锁定约定是严格的。我们将构建的 `mynet` 驱动程序足够小，单个互斥锁就足够了，但即使如此，何时获取它、何时在向上调用之前释放它的纪律也很重要。

### `ifconfig`、`netstat` 和 `tcpdump` 的角色

每个 FreeBSD 用户都知道 `ifconfig`。从网络驱动程序作者的角度来看，`ifconfig` 是内核期望用户命令到达你的驱动程序的主要方式。当用户运行 `ifconfig mynet0 up` 时，内核将其转换为名为 `mynet0` 的接口上的 `SIOCSIFFLAGS` ioctl。调用到达你的 `if_ioctl` 回调，你决定如何处理它。用户态命令和内核端回调之间的对称性几乎是一对一的。

`netstat -in` 向内核请求每个 `ifnet` 上携带的接口统计信息。你的驱动程序通过在发送和接收路径的适当时刻调用 `if_inc_counter(ifp, IFCOUNTER_*, n)` 来更新这些计数器。计数器集在 `/usr/src/sys/net/if.h` 中定义，包括 `IFCOUNTER_IPACKETS`、`IFCOUNTER_OPACKETS`、`IFCOUNTER_IBYTES`、`IFCOUNTER_OBYTES`、`IFCOUNTER_IERRORS`、`IFCOUNTER_OERRORS`、`IFCOUNTER_IMCASTS`、`IFCOUNTER_OMCASTS` 和 `IFCOUNTER_OQDROPS` 等。这些计数器是用户在 `netstat` 和 `systat` 中看到的。

`tcpdump` 依赖于一个名为 Berkeley Packet Filter（BPF）的独立子系统。每个想要对 `tcpdump` 可见的接口都必须通过 `bpfattach()` 向 BPF 注册，驱动程序发送或接收的每个数据包都必须通过 `BPF_MTAP()` 或 `bpf_mtap2()` 在发送出去或向上传递之前呈现给 BPF。我们将在我们的驱动程序中这样做。这是你向系统其余部分支付的小礼貌，以便工具能够工作。

### 一张有用的图

值得用一张图来结束这一节。下面的图片显示了我们描述的各个部分如何组合在一起。暂时不要记住它。只要习惯这个形状。我们将在后面的章节中回到每个框。

```text
          +-------------------+
          |     userland      |
          |   ifconfig(8),    |
          |   tcpdump(1),     |
          |   ping(8), ...    |
          +---------+---------+
                    |
     socket calls,  |  ifconfig ioctls
     tcpdump via bpf|
                    v
          +---------+---------+
          |     network       |
          |      stack        |
          |  TCP/UDP, IP,     |
          |  Ethernet, ARP,   |
          |  routing, BPF     |
          +---------+---------+
                    |
        if_transmit |    if_input
                    v
          +---------+---------+
          |    network        |
          |     driver        |    <-- that is where we live
          |   (ifnet, softc)  |
          +---------+---------+
                    |
                    v
          +---------+---------+
          |    transport      |
          |   real NIC, USB,  |
          |   radio, loopback,|
          |   or simulation   |
          +-------------------+
```

驱动程序上方的框是协议栈和用户态。下方的框是传输层。你的驱动程序，在那条中间线上，是系统中 `struct ifnet` 遇到 `struct mbuf` 遇到线路的唯一地方。那是你的领地。

### 追踪数据包穿越协议栈的过程

追踪一个特定数据包从诞生到消亡是很有用的，因为这将上图中的关系与真实代码固定下来。让我们追踪一个由 `ping 192.0.2.99` 在名为 `mynet0` 的接口上生成的出站 ICMP 回显请求，该接口已被分配地址 `192.0.2.1/24`。

`ping(8)` 程序打开一个原始 ICMP 套接字，通过 `sendto(2)` 写入一个回显请求有效载荷。在内核内部，`/usr/src/sys/kern/uipc_socket.c` 中的套接字层将有效载荷复制到一个新的 mbuf 链中。套接字是未连接的，因此每次写入都携带一个目标地址，套接字层将其转发给协议层。协议层位于 `/usr/src/sys/netinet/raw_ip.c`，它附加一个 IP 头部并调用 `/usr/src/sys/netinet/ip_output.c` 中的 `ip_output`。`ip_output` 执行路由查找，找到一个指向 `mynet0` 的路由条目。它还注意到目标不是广播地址，也不是它已经知道 MAC 地址的链路本地邻居，因此必须触发 ARP。

此时 IP 层调用 `/usr/src/sys/net/if_ethersubr.c` 中定义的 `ether_output`。`ether_output` 注意到下一跳地址未解析，首先发出一个 ARP 请求。ARP 机制位于 `/usr/src/sys/netinet/if_ether.c`，它构造一个广播 ARP 帧，将其包装在一个新的 mbuf 中，然后调用 `ether_output_frame`，后者又调用 `ifp->if_transmit`。这就是我们的 `mynet_transmit` 函数。我们在发送回调中收到的 mbuf 已经包含一个完整的以太网帧：目标 MAC `ff:ff:ff:ff:ff:ff`，源 MAC 为我们编造的地址，EtherType `0x0806`（ARP），以及 ARP 有效载荷。

我们在那个点做每个驱动程序所做的事情：验证、计数、BPF 捕获和释放。因为我们是伪驱动程序，我们释放帧而不是将其交给硬件。在真实的网卡驱动程序中，我们会将 mbuf 交给 DMA 并在完成中断触发时稍后释放它。无论哪种方式，从驱动程序的角度来看，mbuf 已经到达了其生命的终点。

当 ARP 请求悬而未决时，协议栈将原始 ICMP 有效载荷排队在 ARP 等待队列中。当 ARP 回复在可配置的超时时间内没有到达时，协议栈放弃该数据包并递增 `IFCOUNTER_OQDROPS`。当然，在我们的伪驱动程序中，永远不会收到回复，因为模拟线路的另一端没有任何东西。这就是为什么 `ping` 最终打印「100.0% 数据包丢失」并无成功退出的原因。没有回复不是我们驱动程序中的错误；这是我们所选择模拟的传输层的属性。

现在追踪反向路径。我们在 `mynet_rx_timer` 中每秒生成的合成 ARP 请求，最初是在驱动程序内部用 `MGETHDR` 分配的内存。我们填入以太网头部、ARP 头部和 ARP 有效载荷。我们捕获 BPF。我们调用 `if_input`，它解引用 `ifp->if_input` 并进入 `ether_input`。`ether_input` 查看 EtherType 并将有效载荷分派到 `arpintr`（或其现代等价物，即 `ether_demux` 内部的直接调用）。ARP 代码检查发送方和目标 IP，注意到目标不是我们，静默丢弃该帧。完成。

在两个方向上，驱动程序都是一个简短的直通：一个 mbuf 到达，一个 mbuf 离开，计数器移动，BPF 看到其间的一切。这种简单性具有欺骗性，因为每一步都有一个不可违反的契约，但模式确实是这么短。

### 你上方的队列调度

你从驱动程序中看不到它们，但协议栈有队列调度规则来控制数据包如何传递给 `if_transmit`。历史上，驱动程序有一个 `if_start` 回调，协议栈会将数据包放在内部队列（`if_snd`）上以便稍后分发。现代驱动程序使用 `if_transmit` 并直接接收 mbuf，让驱动程序或 `drbr(9)` 辅助库在内部管理任何每 CPU 队列。

实际上，几乎所有现代驱动程序都使用 `if_transmit`，让协议栈一次交给它们一个数据包。因为 `if_transmit` 在产生数据包的线程上被调用（通常是 TCP 重传定时器或写入套接字的线程），发送路径通常在启用了抢占的常规内核线程上。这很重要，因为这意味着你通常不能假设发送以提升的优先级运行，你不得在长时间操作中持有互斥锁。

少数驱动程序仍然使用经典的 `if_start` 模型，其中协议栈填充一个队列并调用 `if_start` 来排空它。该模型对于具有简单硬件排队的驱动程序来说更简单，但在负载下灵活性较差。`epair(4)` 直接使用 `if_transmit`。`disc(4)` 实现了自己微小的 `discoutput`，从 `ether_output` 的预发送路径中调用。大多数真正的网卡驱动程序使用 `if_transmit`，配合由 `drbr` 驱动的内部每 CPU 队列。

对于 `mynet`，我们使用 `if_transmit` 且没有内部队列。这是最简单的设计，它匹配最小真实驱动程序在低带宽链路上的做法。

### 关于数据包捕获可见性的说明

数据包捕获是网络驱动程序与字符驱动程序感觉不同的关键原因之一。字符驱动程序的流量对外部观察者是不可见的，因为对于任意 `/dev/` 流量没有类似 `tcpdump` 的工具。相比之下，网络驱动程序的流量可以在多个级别同时观察：BPF 在驱动程序级别捕获，pflog 在包过滤级别，接口计数器在内核级别，套接字缓冲区在用户态级别。所有这些可观测性对驱动程序作者来说是免费的，只要驱动程序在正确的点捕获 BPF 并更新计数器。

这种不寻常的外部可见性水平是调试的福音。当你无法判断一个数据包为什么流动或没有流动时，你几乎总是可以通过 `tcpdump`、`netstat`、`arp` 和 `route monitor` 的组合来回答问题。那是一套有能力的工具集，我们将在整个实验中使用它。

### 第1节小结

我们已经布好了场景。网络驱动程序在协议栈和传输层之间移动 mbuf。它呈现一个名为 `ifnet` 的标准化接口。它由协议栈通过固定回调驱动。它通过 `if_input` 向上推送接收到的流量。它通过少量内核约定对 `ifconfig`、`netstat` 和 `tcpdump` 可见。

有了那个粗略的形状，我们可以看看 `ifnet` 对象本身。那是第2节的主题。

## 第2节：认识 `ifnet`

每个运行中的 FreeBSD 系统上的网络接口在内核中由一个 `struct ifnet` 表示。该结构是网络子系统的核心对象。当 `ifconfig` 列出接口时，它本质上是在遍历一个 `ifnet` 对象列表。当协议栈选择路由时，它最终落在一个 `ifnet` 上并调用其发送函数。当驱动程序报告链路状态时，它更新 `ifnet` 内部的字段。学习 `ifnet` 不是可选的。本章中的其他一切都建立在它之上。

### `ifnet` 的位置

`struct ifnet` 的声明在 `/usr/src/sys/net/if_var.h` 中。多年来，FreeBSD 一直趋向于将其视为不透明的，在新驱动程序代码中引用它的推荐方式是通过 typedef `if_t`，它是指向底层结构的指针：

```c
typedef struct ifnet *if_t;
```

旧的驱动程序代码直接访问 `ifp->if_softc`、`ifp->if_flags`、`ifp->if_mtu` 和类似字段。新的驱动程序代码倾向于使用访问器函数，如 `if_setsoftc(ifp, sc)`、`if_getflags(ifp)`、`if_setflags(ifp, flags)` 和 `if_setmtu(ifp, mtu)`。两种风格仍然存在于源代码树中，现有驱动程序如 `/usr/src/sys/net/if_disc.c` 仍然使用直接字段访问。不透明风格是内核发展的方向，但你在未来几年内仍会看到两种风格。

在本章中，我们将使用在给定上下文中最清晰的方式。当直接字段风格使代码更小、更易读时，我们将使用它。当访问器使意图更清晰时，我们将使用它。你应该能够阅读两种形式。

### 你需要关心的最少字段

`struct ifnet` 有几十个字段。好消息是驱动程序只直接接触其中的一小部分。你将在我们构建的驱动程序中设置或检查的字段大致如下：

* **身份。** `if_softc` 指回你驱动程序的私有结构体，`if_xname` 是接口名称（例如 `mynet0`），`if_dname` 是家族名称（`"mynet"`），`if_dunit` 是单元编号。
* **能力和计数。** `if_mtu` 是最大传输单元，`if_baudrate` 是以比特每秒为单位的报告线路速率，`if_capabilities` 和 `if_capenable` 描述卸载能力，如 VLAN 标记和校验和卸载。
* **标志。** `if_flags` 保存由用户态设置的接口级标志：`IFF_UP`、`IFF_BROADCAST`、`IFF_SIMPLEX`、`IFF_MULTICAST`、`IFF_POINTOPOINT`、`IFF_LOOPBACK`。`if_drv_flags` 保存驱动程序私有标志；最重要的是 `IFF_DRV_RUNNING`，表示驱动程序已分配其每接口资源并准备好移动流量。
* **回调。** `if_init`、`if_ioctl`、`if_transmit`、`if_qflush` 和 `if_input` 是协议栈调用的函数指针。其中一些有长期存在的直接字段；访问器等价物是 `if_setinitfn`、`if_setioctlfn`、`if_settransmitfn`、`if_setqflushfn` 和 `if_setinputfn`。
* **统计。** 每计数器访问器 `if_inc_counter(ifp, IFCOUNTER_*, n)` 递增 `netstat -in` 显示的计数器。
* **BPF 钩子。** `if_bpf` 是 BPF 使用的不透明指针。你的驱动程序通常不会直接读取它，但当你调用 `bpfattach(ifp, ...)` 和 `BPF_MTAP(ifp, m)` 时，系统会管理它。
* **媒体和链路状态。** `ifmedia` 位于你的 softc 中，而不是 `ifnet` 中，但接口通过调用 `if_link_state_change(ifp, LINK_STATE_*)` 报告链路状态。

如果列表看起来很长，请记住大多数驱动程序设置每个字段一次然后就不管了。驱动程序的工作在回调中，不在 ifnet 字段本身中。

### `ifnet` 的生命周期

`struct ifnet` 经历与 `device_t` 或 softc 相同的高层阶段：分配、配置、注册、活跃生命周期和拆卸。调用图为：

```text
  if_alloc(type)         -> returns a fresh ifnet, not yet attached
     |
     | configure fields
     |  if_initname()       set the name
     |  if_setsoftc()       point at your softc
     |  if_setinitfn()      set if_init callback
     |  if_setioctlfn()     set if_ioctl
     |  if_settransmitfn()  set if_transmit
     |  if_setqflushfn()    set if_qflush
     |  if_setflagbits()    set IFF_BROADCAST, etc.
     |  if_setmtu()         set MTU
     v
  if_attach(ifp)         OR ether_ifattach(ifp, mac)
     |
     | live interface
     |  if_transmit called by stack
     |  if_ioctl called by stack
     |  driver calls if_input to deliver received packets
     |  driver calls if_link_state_change on link events
     v
  ether_ifdetach(ifp)    OR if_detach(ifp)
     |
     | finish teardown
     v
  if_free(ifp)
```

附加和分离调用有两种常见的变体。不需要以太网布线的普通伪接口使用 `if_attach` 和 `if_detach`。伪或真实以太网接口使用 `ether_ifattach` 和 `ether_ifdetach`。以太网变体包装了普通变体，并添加了二层以太网接口所需的额外设置，包括 `bpfattach`、地址注册，以及将 `ifp->if_input` 和 `ifp->if_output` 连接到 `ether_input` 和 `ether_output`。我们将在驱动程序中使用以太网变体，因为它为我们提供了一个熟悉的 MAC 地址接口，`ifconfig`、`ping` 和 `tcpdump` 都可以直接理解而无需特殊处理。

如果你打开 `/usr/src/sys/net/if_ethersubr.c` 查看 `ether_ifattach`，你会看到正是这个逻辑：将 `if_addrlen` 设置为 `ETHER_ADDR_LEN`，将 `if_hdrlen` 设置为 `ETHER_HDR_LEN`，将 `if_mtu` 设置为 `ETHERMTU`，调用 `if_attach`，然后安装通用以太网输入和输出例程，最后调用 `bpfattach`。值得完整阅读那个函数。它很短，它向你展示了驱动程序通过使用 `ether_ifattach` 而不是裸 `if_attach` 免费获得了什么。

### 为什么 `ifnet` 不是 `cdevsw`

很容易将 `ifnet` 看作只是「网络的 cdevsw」。不是这样的。`cdevsw` 是 `devfs` 用来将用户态的 `read`、`write`、`ioctl`、`open` 和 `close` 分派到驱动程序的入口表。`ifnet` 是网络协议栈自身为每个接口维护的一等对象。即使用户态进程从未接触过该接口，协议栈仍然关心它的 `ifnet`，因为路由表、ARP 和数据包转发都依赖于它。

如果你思考 `ifconfig` 如何与内核对话，你可以看到这一点。它不打开 `/dev/mynet0`。它打开一个套接字并在该套接字上发出 ioctl，将接口名称作为参数传递。内核然后按名称查找 `ifnet` 并对其调用 `if_ioctl`。用户态端没有指向你接口的文件描述符。接口是协议栈级别的实体，不是 `/dev/` 实体。

这就是为什么我们需要一个全新的对象：因为网络需要一个持久的、内核内部的句柄，无论哪个进程在做什么都存在。`ifnet` 就是那个句柄。

### 伪接口与真实网卡接口

内核中的每个接口，无论是伪的还是真实的，都有一个 `ifnet`。环回接口 `lo0` 有一个。我们将研究的 `disc` 接口有一个。每个 `emX` 以太网适配器都有一个。每个 `wlanX` 无线接口都有一个。`ifnet` 是通用货币。

伪接口与真实网卡在实例化方式上不同。真实网卡接口在总线探测期间由驱动程序的 `attach` 方法创建，就像第26章中的 USB 和 PCI 驱动程序附加它们的设备一样。伪接口在模块加载时创建，或者通过 `ifconfig mynet0 create` 按需创建，通过一种称为**接口克隆器**的机制。我们将为 `mynet` 使用接口克隆器，这意味着用户将能够动态创建接口，就像他们今天可以创建 epair 接口一样：

```console
# ifconfig mynet create
mynet0
# ifconfig mynet0 up
# ifconfig mynet0
mynet0: flags=8843<UP,BROADCAST,RUNNING,SIMPLEX,MULTICAST> metric 0 mtu 1500
```

我们将在第3节中描述克隆器。现在，只需要知道克隆是模块根据用户请求向运行中的系统贡献一个或多个 `ifnet` 对象的方式就够了。

### 深入了解关键 `ifnet` 字段

因为 `ifnet` 是你的驱动程序最常写入的结构，在打开代码之前，稍微深入地了解它的一些字段会有帮助。你不需要记住完整的声明。你需要的是对布局足够熟悉，以便在不不断翻回 `if_var.h` 的情况下阅读驱动程序代码。

`if_xname` 是一个字符数组，保存接口的用户可见名称，例如 `mynet0`。它由 `if_initname` 设置，从那一刻起被协议栈视为只读。当你阅读 `ifconfig -a` 输出时，每一行以接口名称开头的行都在打印 `if_xname` 的副本。

`if_dname` 和 `if_dunit` 分别记录驱动程序家族名称和单元编号。对于我们驱动程序的每个实例，`if_dname` 都是 `"mynet"`，而 `if_dunit` 对于 `mynet0` 是 `0`，对于 `mynet1` 是 `1`，依此类推。网络协议栈使用这些字段将接口索引到各种哈希中，`ifconfig` 在将接口名称匹配到驱动程序家族时也使用它们。

`if_softc` 是指回你驱动程序私有每接口结构体的反向指针。协议栈调用的每个回调都会传递一个 `ifp` 参数，大多数回调做的第一件事就是从 `ifp->if_softc`（或 `if_getsoftc(ifp)`）中取出 softc。如果你在创建期间忘记设置 `if_softc`，你的回调将解引用一个 NULL 指针，内核将会崩溃。

`if_type` 是来自 `/usr/src/sys/net/if_types.h` 的类型常量。`IFT_ETHER` 用于类似以太网的接口，`IFT_LOOP` 用于环回，`IFT_IEEE80211` 用于无线，`IFT_TUNNEL` 用于通用隧道，还有几十个其他类型。协议栈偶尔会根据 `if_type` 专门化行为，例如在决定如何格式化链路层地址以供显示时。

`if_addrlen` 和 `if_hdrlen` 描述链路层地址长度（以太网为 6 字节，InfiniBand 为 8 字节，纯三层隧道为 0）和链路层头部长度（普通以太网为 14 字节，带标签的以太网为 22 字节）。`ether_ifattach` 会用以太网默认值为你设置这两个字段。其他链路层辅助函数会用它们自己的值来设置。

`if_flags` 是用户可见标志的位掩码，如 `IFF_UP` 和 `IFF_BROADCAST`。`if_drv_flags` 是驱动程序私有标志的位掩码，如 `IFF_DRV_RUNNING`。它们是分开的，因为它们有不同的访问规则。用户可以写入 `if_flags`；只有驱动程序写入 `if_drv_flags`。将它们混淆是一个典型的错误。

`if_capabilities` 和 `if_capenable` 描述卸载特性。`if_capabilities` 是硬件声称它能做的事情。`if_capenable` 是当前已启用的功能。这种分离允许用户态在运行时通过 `ifconfig mynet0 -rxcsum` 或 `ifconfig mynet0 +tso` 切换卸载，驱动程序则遵守该选择。我们将在第 6 节中看到它与 `SIOCSIFCAP` 的交互。

`if_mtu` 是以字节为单位的最大传输单元。它是接口可以承载的最大三层有效载荷，不包括链路层头部。以太网默认值为 1500。巨型帧以太网通常支持 9000 或 9216。`if_baudrate` 是以比特每秒为单位的信息性线路速率字段；仅供参考。

`if_init` 是一个函数指针，在接口转换为 up 状态时被调用。其签名为 `void (*)(void *softc)`。`if_ioctl` 在目标为此接口的套接字 ioctl 时被调用；签名为 `int (*)(struct ifnet *, u_long, caddr_t)`。`if_transmit` 在发送数据包时被调用；签名为 `int (*)(struct ifnet *, struct mbuf *)`。`if_qflush` 在刷新驱动程序私有队列时被调用；签名为 `void (*)(struct ifnet *)`。`if_input` 是另一个方向的函数指针：驱动程序调用它（通常通过 `if_input(ifp, m)` 辅助函数）将接收到的 mbuf 交给协议栈。

`if_snd` 是遗留的发送队列，由仍然使用 `if_start` 回调而不是 `if_transmit` 的驱动程序使用。对于使用 `if_transmit` 的现代驱动程序，`if_snd` 不再使用。你在源代码树中读到的大多数教科书示例（包括我们的 `if_disc.c` 参考）不再接触 `if_snd`。

`if_bpf` 是 BPF 附加指针。BPF 本身管理该值；驱动程序将其视为不透明的。`BPF_MTAP` 和相关宏在内部使用它。

`if_data` 是一个大型结构体，承载每接口统计信息、媒体描述符和各种杂项字段。现代驱动程序避免直接接触 `if_data`，而是通过 `if_inc_counter` 及相关函数来操作。`if_data` 结构体仍然存在是为了向后兼容和供用户态可见的统计信息。

这远非一个详尽的列表；`struct ifnet` 总共有五十多个字段。但上面列出的那些是你的驱动程序最可能接触到的字段，熟悉它们的名称将使后续每个代码清单更容易阅读。

### 访问器 API 详解

`if_t` 不透明句柄自 FreeBSD 12 以来一直在发展一系列访问器函数。模式是一致的：以前你会写 `ifp->if_flags |= IFF_UP`，现在你写 `if_setflagbits(ifp, IFF_UP, 0)`。以前你会写 `ifp->if_softc = sc`，现在你写 `if_setsoftc(ifp, sc)`。其动机是让内核能够演化 `struct ifnet` 的内部布局而不破坏驱动程序。

访问器函数包括：

* `if_setsoftc(ifp, sc)` 和 `if_getsoftc(ifp)` 用于 softc 指针。
* `if_setflagbits(ifp, set, clear)` 和 `if_getflags(ifp)` 用于 `if_flags`。
* `if_setdrvflagbits(ifp, set, clear)` 和 `if_getdrvflags(ifp)` 用于 `if_drv_flags`。
* `if_setmtu(ifp, mtu)` 和 `if_getmtu(ifp)` 用于 MTU。
* `if_setbaudrate(ifp, rate)` 和 `if_getbaudrate(ifp)` 用于报告的线路速率。
* `if_sethwassist(ifp, assist)` 和 `if_gethwassist(ifp)` 用于校验和卸载提示。
* `if_settransmitfn(ifp, fn)` 用于 `if_transmit`。
* `if_setioctlfn(ifp, fn)` 用于 `if_ioctl`。
* `if_setinitfn(ifp, fn)` 用于 `if_init`。
* `if_setqflushfn(ifp, fn)` 用于 `if_qflush`。
* `if_setinputfn(ifp, fn)` 用于 `if_input`。
* `if_inc_counter(ifp, ctr, n)` 用于统计计数器。

其中一些是内联函数，在幕后仍然直接访问字段；其他的是包装器，将来可能引用微妙不同的字段布局。现在使用访问器不花任何代价，并能保护你的驱动程序免受未来的变动影响。

对于 `mynet`，我们主要使用直接字段风格，因为现有的参考驱动程序如 `if_disc.c` 和 `if_epair.c` 仍在使用它，与源代码树其余部分的一致性对读者来说是有价值的。当你毕业编写自己的新驱动程序时，可以随意选择访问器。两种风格都是正确的。

### 代码初览

在我们继续之前，让我们看一个微小的代码片段，它总结了驱动程序与 `ifnet` 关系的形状。这是你在第 3 节中会更完整地输入的模式，但看到骨架已经很有用了：

```c
struct mynet_softc {
    struct ifnet    *ifp;
    struct mtx       mtx;
    uint8_t          hwaddr[ETHER_ADDR_LEN];
    /* ... fields for simulation state ... */
};

static int
mynet_transmit(struct ifnet *ifp, struct mbuf *m)
{
    /* pass packet to the transport, or drop it */
}

static int
mynet_ioctl(struct ifnet *ifp, u_long cmd, caddr_t data)
{
    /* handle SIOCSIFFLAGS, SIOCSIFMTU, ... */
}

static void
mynet_init(void *arg)
{
    /* make the interface ready to move traffic */
}

static void
mynet_create(void)
{
    struct mynet_softc *sc = malloc(sizeof(*sc), M_MYNET, M_WAITOK | M_ZERO);
    struct ifnet *ifp = if_alloc(IFT_ETHER);

    sc->ifp = ifp;
    mtx_init(&sc->mtx, "mynet", NULL, MTX_DEF);
    ifp->if_softc = sc;
    if_initname(ifp, "mynet", 0);
    ifp->if_flags = IFF_BROADCAST | IFF_SIMPLEX | IFF_MULTICAST;
    ifp->if_init = mynet_init;
    ifp->if_ioctl = mynet_ioctl;
    ifp->if_transmit = mynet_transmit;
    ifp->if_qflush = mynet_qflush;

    /* fabricate a MAC address ... */
    ether_ifattach(ifp, sc->hwaddr);
}
```

先不要输入这段代码。它只是一个草图，还有几个部分缺失。我们将在第 3 节中填充它们。现在重要的是形状：分配、配置、附加。源代码树中的每个驱动程序都这样做，只是根据它所在的总线和它对话的传输层有所不同。

### 第2节小结

`ifnet` 对象是内核对网络接口的表示。它有身份字段、能力字段、标志、回调、计数器和媒体状态。它由 `if_alloc` 创建，由驱动程序配置，并通过 `if_attach` 或 `ether_ifattach` 安装到系统中。伪接口驱动程序通过接口克隆器按需创建 `ifnet`。真正的网卡驱动程序在探测和附加期间创建其 `ifnet`。

你现在掌握了词汇。在第 3 节中，我们将通过创建和注册一个真正可用的网络接口来运用它。但在那之前，值得花一点时间阅读一个使用我们即将编写的相同模式的真实驱动程序。下一个小节将引导你阅读 `if_disc.c`，它是 FreeBSD 源代码树中经典的「最简伪以太网」驱动程序。

### `if_disc.c` 导读

在你的编辑器中打开 `/usr/src/sys/net/if_disc.c`。它大约有两百行代码，每一行都很有教益。`disc(4)` 驱动程序创建一个接口，其唯一工作是静默丢弃它收到的每个待发送数据包。它相当于数据包世界的 `/dev/null`。因为它如此之小，它展示了伪驱动程序的形状而没有任何干扰。

该文件以标准许可证头部开始，然后是一组现在应该看起来很熟悉的 `#include` 指令。`net/if.h` 和 `net/if_var.h` 用于接口结构体，`net/ethernet.h` 用于以太网特定的辅助函数，`net/if_clone.h` 用于克隆器 API，`net/bpf.h` 用于数据包捕获，`net/vnet.h` 用于 VNET 感知。这几乎正是我们将在 `mynet.c` 中使用的包含集。

接下来是一些模块级声明。字符串 `discname = "disc"` 是克隆器将暴露的家族名称。`M_DISC` 是 `vmstat -m` 记账的内存类型标签。`VNET_DEFINE_STATIC(struct if_clone *, disc_cloner)` 声明一个每 VNET 的克隆器变量，`V_disc_cloner` 宏提供访问垫片。这些片段在你几页后在我们自己的驱动程序中写入相同的三行时都会被识别出来。

softc 声明特别短。`struct disc_softc` 只保存一个 `ifnet` 指针。这就是一个丢弃驱动程序所需的全部状态：每个 softc 一个接口，没有计数器，没有队列，没有定时器。我们的 `mynet` softc 会更长，因为我们有模拟的接收路径、媒体描述符和互斥锁，但「每个接口一个 softc」的模式是相同的。

向下浏览文件到 `disc_clone_create`。它首先用 `M_WAITOK | M_ZERO` 分配 softc，因为克隆器是从用户上下文调用的，可以承受睡眠。然后它用 `if_alloc(IFT_LOOP)` 分配 `ifnet`。注意 `disc` 使用 `IFT_LOOP` 而不是 `IFT_ETHER`，因为它的链路层语义更像环回而不是以太网。`IFT_*` 常量的选择很重要，因为协议栈查询 `if_type` 来决定调用哪个链路层辅助函数。我们的驱动程序将使用 `IFT_ETHER`，因为我们想使用 `ether_ifattach`。

然后 `disc_clone_create` 调用 `if_initname(ifp, discname, unit)`，设置 softc 指针，将 `if_mtu` 设置为 `DSMTU`（一个本地定义的值），并将 `if_flags` 设置为 `IFF_LOOPBACK | IFF_MULTICAST`。回调 `if_ioctl`、`if_output` 和 `if_init` 被设置。注意 `disc` 设置的是 `if_output` 而不是 `if_transmit`，因为环回风格的驱动程序仍然连接到经典的输出路径。我们的以太网驱动程序将通过 `ether_ifattach` 使用 `if_transmit`。

然后是 `if_attach(ifp)`，它在没有以太网特定设置的情况下将接口注册到协议栈。接着是 `bpfattach(ifp, DLT_NULL, sizeof(u_int32_t))`，使用空链路类型向 BPF 注册（告诉 `tcpdump` 期望一个携带有效载荷地址族的四字节头部）。我们的驱动程序将通过 `ether_ifattach` 自动使用 `DLT_EN10MB`。

销毁路径 `disc_clone_destroy` 是对称的：它调用 `bpfdetach`、`if_detach`、`if_free`，最后调用 `free(sc, M_DISC)`。我们的驱动程序会稍微复杂一些，因为我们有 callout 和媒体描述符需要拆卸，但骨架是相同的。

发送路径 `discoutput` 只有三行代码。它检查数据包族，填入四字节的 BPF 头部，捕获 BPF，更新计数器，并释放 mbuf。这就是一个「丢弃一切」驱动程序需要做的全部。我们的 `mynet_transmit` 会更长，但在结构上它做的事情完全相同，只是稍微更有纪律：验证、捕获、计数、释放。

ioctl 处理程序 `discioctl` 处理 `SIOCSIFADDR`、`SIOCSIFFLAGS` 和 `SIOCSIFMTU`，对所有其他请求返回 `EINVAL`。对于一个最小的伪驱动程序来说这已经足够了。我们的驱动程序会更精细，因为我们添加了媒体描述符并将未知 ioctl 委托给 `ether_ioctl`，但 switch 语句的形状是相同的。

最后，克隆器注册在 `vnet_disc_init` 中通过 `if_clone_simple(discname, disc_clone_create, disc_clone_destroy, 0)` 完成，包装在 `VNET_SYSINIT` 中，并由调用 `if_clone_detach` 的 `VNET_SYSUNINIT` 匹配。同样，这正是我们将使用的模式。

阅读 `disc` 的收获是，FreeBSD 源代码树中一个可工作的伪驱动程序大约是两百行代码。其中大部分行是样板代码，你设置一次然后就不必管了。有趣的部分是 softc、克隆器和少数回调。其余的都是节奏。

不要觉得必须记住 `disc`。现在只需慢慢读一次。当我们开始编写 `mynet` 时，回到这一节，你会发现我们输入的大部分内容都是相同的模式，只是为类似以太网的行为、数据包接收和媒体描述符添加了一些内容。在我们详细阐述之前，以最纯粹的形式看到一次这个模式是值得的。

## 第3节：创建和注册网络接口

是时候写代码了。在本节中，我们将构建 `mynet` 的骨架，一个伪以太网驱动程序。它对系统其余部分将表现为一个普通的以太网接口。用户态将能够通过 `ifconfig mynet create` 创建一个实例，分配 IPv4 地址，启动它、关闭它、销毁它，就像 `epair` 和 `disc` 一样。我们暂时不处理真正的数据包移动。第 4 节和第 5 节将处理发送和接收路径。这里我们专注于创建、注册和基本元数据。

### 项目布局

本章的所有配套文件位于 `examples/part-06/ch28-network-driver/` 下。本节的骨架在 `examples/part-06/ch28-network-driver/lab01-skeleton/` 中。如果你是手动输入的，请创建目录；如果你喜欢先阅读再实验，可以查看文件。我们将用于本章的顶层布局是：

```text
examples/part-06/ch28-network-driver/
  Makefile
  mynet.c
  README.md
  shared/
  lab01-skeleton/
  lab02-transmit/
  lab03-receive/
  lab04-media/
  lab05-bpf/
  lab06-detach/
  lab07-reading-tree/
  challenge01-shared-queue/
  challenge02-link-flap/
  challenge03-error-injection/
  challenge04-iperf3/
  challenge05-sysctl/
  challenge06-netgraph/
```

顶层 `mynet.c` 是整个章节的参考驱动程序，从第 3 节的骨架到第 8 节的最终清理代码不断演进。`lab0x` 目录包含引导你完成相应实验步骤的 README 文件。每个挑战在完成的驱动程序之上添加一个小功能，`shared/` 包含多个实验引用的辅助脚本和笔记。

### Makefile

让我们从构建文件开始。伪以太网驱动程序的内核模块是整个源代码树中最简单的 Makefile 之一。我们的将如下所示：

```console
# Makefile for mynet - Chapter 28 (Writing a Network Driver).
#
# Builds the chapter's reference pseudo-Ethernet driver,
# mynet.ko, which demonstrates ifnet registration through an
# interface cloner, minimal transmit and receive paths, and
# safe load and unload lifecycle.

KMOD=   mynet
SRCS=   mynet.c opt_inet.h opt_inet6.h

SYSDIR?=    /usr/src/sys

.include <bsd.kmod.mk>
```

这与 `/usr/src/sys/modules/if_disc/Makefile` 使用的 Makefile 非常接近，这正是你想要的基于克隆的伪接口驱动程序。两个小区别：我们不设置 `.PATH`，因为我们的源文件在当前目录而不是 `/usr/src/sys/net/`；我们显式设置 `SYSDIR`，以便在没有为其提供系统配置的机器上构建也能正常工作。除此之外，这是你自第 10 章以来看到的标准 `bsd.kmod.mk` 模式。

### 头文件包含与模块胶水代码

打开你的编辑器，用以下前言开始 `mynet.c`。每个 include 都有特定的作用，所以我们将在进行过程中逐一注释：

```c
#include <sys/param.h>
#include <sys/systm.h>
#include <sys/kernel.h>
#include <sys/module.h>
#include <sys/malloc.h>
#include <sys/lock.h>
#include <sys/mutex.h>
#include <sys/mbuf.h>
#include <sys/socket.h>
#include <sys/sockio.h>
#include <sys/callout.h>

#include <net/if.h>
#include <net/if_var.h>
#include <net/if_arp.h>
#include <net/ethernet.h>
#include <net/if_types.h>
#include <net/if_clone.h>
#include <net/if_media.h>
#include <net/bpf.h>
#include <net/vnet.h>
```

第一个块引入了你已经从前几章中了解的核心内核头文件：参数、系统调用、模块机制、内存分配器、锁定、mbuf、套接字 IO 常量和 callout 子系统。第二个块引入了网络特定的头文件：`if.h` 用于 `ifnet` 结构体和标志，`if_var.h` 用于内联辅助函数，`if_arp.h` 用于地址解析常量，`ethernet.h` 用于以太网成帧，`if_types.h` 用于接口类型常量如 `IFT_ETHER`，`if_clone.h` 用于克隆器 API，`if_media.h` 用于媒体描述符，`bpf.h` 用于 `tcpdump` 支持，`vnet.h` 用于 VNET 感知，我们的使用方式与 `/usr/src/sys/net/if_disc.c` 相同。

接下来，是模块范围的内存类型和接口家族名称：

```c
static const char mynet_name[] = "mynet";
static MALLOC_DEFINE(M_MYNET, "mynet", "mynet pseudo Ethernet driver");

VNET_DEFINE_STATIC(struct if_clone *, mynet_cloner);
#define V_mynet_cloner  VNET(mynet_cloner)
```

`mynet_name` 是我们将传递给 `if_initname` 的字符串，以便接口命名为 `mynet0`、`mynet1` 等。`M_MYNET` 是内存类型标签，以便 `vmstat -m` 显示驱动程序正在使用多少内存。`VNET_DEFINE_STATIC` 是 VNET 感知的：它给每个虚拟网络栈自己的克隆器变量。这镜像了 `/usr/src/sys/net/if_disc.c` 中的 `VNET_DEFINE_STATIC(disc_cloner)` 声明。我们将在第 8 节中简要回到 VNET。

函数、宏和结构体名称是进入 FreeBSD 源代码树的持久参考。行号会随版本变化。仅作为 FreeBSD 14.3 定位参考：在 `/usr/src/sys/net/if_disc.c` 中，`VNET_DEFINE_STATIC(disc_cloner)` 声明位于第 79 行附近，`vnet_disc_init` 中的 `if_clone_simple` 调用位于第 134 行附近；在 `/usr/src/sys/net/if_epair.c` 中，`epair_transmit` 从第 324 行附近开始，`epair_ioctl` 从第 429 行附近开始；在 `/usr/src/sys/sys/mbuf.h` 中，`MGETHDR` 兼容宏位于第 1125 行附近。打开文件并跳转到符号。

### softc 结构

正如你从前几章中所知，softc 是你的驱动程序分配的私有每实例结构体，用于跟踪一个设备的状态。对于网络驱动程序，softc 是每接口的。以下是我们在这一阶段的 softc 结构：

```c
struct mynet_softc {
    struct ifnet    *ifp;
    struct mtx       mtx;
    uint8_t          hwaddr[ETHER_ADDR_LEN];
    struct ifmedia   media;
    struct callout   rx_callout;
    int              rx_interval_hz;
    bool             running;
};

#define MYNET_LOCK(sc)      mtx_lock(&(sc)->mtx)
#define MYNET_UNLOCK(sc)    mtx_unlock(&(sc)->mtx)
#define MYNET_ASSERT(sc)    mtx_assert(&(sc)->mtx, MA_OWNED)
```

字段很直观。`ifp` 是我们创建的接口对象。`mtx` 是一个互斥锁，用于在并发发送、ioctl 和拆卸期间保护 softc。`hwaddr` 是我们编造的六字节以太网地址。`media` 是我们通过 `SIOCGIFMEDIA` 暴露的媒体描述符。`rx_callout` 和 `rx_interval_hz` 用于我们在第 5 节中构建的模拟接收路径。`running` 反映驱动程序对接口当前是否活动的感知。

底部的宏为我们提供了简短、可读的锁定原语。它们是许多 FreeBSD 驱动程序中使用的风格约定，包括 `/usr/src/sys/dev/e1000/if_em.c` 和 `/usr/src/sys/net/if_epair.c`。

### `mynet_create` 的骨架

现在是本节的主要部分。我们将编写一个从克隆器调用的函数，用于创建和注册新接口。这个函数是初始化代码的核心。让我们逐步构建它，然后组装各个部分。

首先，分配 softc 和 `ifnet`：

```c
struct mynet_softc *sc;
struct ifnet *ifp;

sc = malloc(sizeof(*sc), M_MYNET, M_WAITOK | M_ZERO);
ifp = if_alloc(IFT_ETHER);
if (ifp == NULL) {
    free(sc, M_MYNET);
    return (ENOSPC);
}
sc->ifp = ifp;
mtx_init(&sc->mtx, "mynet", NULL, MTX_DEF);
```

我们使用 `M_WAITOK | M_ZERO`，因为这是从用户上下文路径（克隆器）调用的，我们需要零初始化的内存。`IFT_ETHER` 来自 `/usr/src/sys/net/if_types.h`：它将我们的接口声明为以太网接口用于内核的簿记目的，这很重要，因为协议栈使用 `if_type` 来决定应用什么链路层语义。

接下来，编造一个 MAC 地址。在真正的网卡驱动程序中，硬件有一个带有唯一出厂分配 MAC 的 EEPROM。我们没有这种奢侈，所以我们要发明一个。本地管理的单播地址以一个字节开始，该字节的第二低位被设置，最低位被清除。经典方式是 `02:xx:xx:xx:xx:xx`。我们将做类似 `epair(4)` 在其 `epair_generate_mac` 函数中做的事情：

```c
arc4rand(sc->hwaddr, ETHER_ADDR_LEN, 0);
sc->hwaddr[0] = 0x02;  /* locally administered, unicast */
```

`arc4rand` 是一个内核内部的熵支持随机函数，定义在 `/usr/src/sys/libkern/arc4random.c` 中。它适合用于 MAC 地址编造。然后我们将第一个字节强制为 `0x02`，使地址既是本地管理的又是单播的，这正是 IEEE 为非出厂分配的地址保留的。

接下来，配置接口字段：

```c
if_initname(ifp, mynet_name, unit);
ifp->if_softc = sc;
ifp->if_flags = IFF_BROADCAST | IFF_SIMPLEX | IFF_MULTICAST;
ifp->if_capabilities = IFCAP_VLAN_MTU;
ifp->if_capenable = IFCAP_VLAN_MTU;
ifp->if_transmit = mynet_transmit;
ifp->if_qflush = mynet_qflush;
ifp->if_ioctl = mynet_ioctl;
ifp->if_init = mynet_init;
ifp->if_baudrate = IF_Gbps(1);
```

`if_initname` 设置 `if_xname`（接口的唯一名称）以及驱动程序的家族名称和单元编号。`if_softc` 将接口与我们的私有结构体关联起来，以便回调可以找到它。这些标志将接口标记为支持广播、单工（意味着它不能听到自己的传输，这对以太网网卡来说是正确的）和支持多播。`IFCAP_VLAN_MTU` 表示我们可以转发 VLAN 标记的帧，其总有效载荷超过基线以太网 MTU 四个字节。回调是我们稍后将实现的函数。`if_baudrate` 是信息性的；`IF_Gbps(1)` 报告每秒一千兆比特，大致匹配平均模拟链路可能声称的速率。

接下来，设置媒体描述符。这是 `SIOCGIFMEDIA` 将返回的内容，也是 `ifconfig mynet0` 用来打印媒体行的内容：

```c
ifmedia_init(&sc->media, 0, mynet_media_change, mynet_media_status);
ifmedia_add(&sc->media, IFM_ETHER | IFM_1000_T | IFM_FDX, 0, NULL);
ifmedia_add(&sc->media, IFM_ETHER | IFM_AUTO, 0, NULL);
ifmedia_set(&sc->media, IFM_ETHER | IFM_AUTO);
```

`ifmedia_init` 注册两个回调：一个是协议栈在用户更改媒体时调用的，另一个是它调用以了解当前媒体状态的。`ifmedia_add` 声明接口支持的特定媒体类型。`IFM_ETHER | IFM_1000_T | IFM_FDX` 表示「以太网，1000BaseT，全双工」；`IFM_ETHER | IFM_AUTO` 表示「以太网，自动协商」。`ifmedia_set` 选择默认值。`ifconfig mynet0` 将反映此选择。

接下来，初始化模拟接收 callout。我们将在第 5 节中实现它，但现在我们准备该字段，以便 `mynet_create` 使 softc 完全可用：

```c
callout_init_mtx(&sc->rx_callout, &sc->mtx, 0);
sc->rx_interval_hz = hz;  /* one simulated packet per second */
```

`callout_init_mtx` 用 softc 的互斥锁注册我们的 callout，以便 callout 系统在调用处理程序时为我们获取和释放锁。这是内核中广泛使用的模式，它避免了一整类锁排序错误。

最后，将接口附加到以太网层：

```c
ether_ifattach(ifp, sc->hwaddr);
```

这个单一调用做了大量的工作。它将 `if_addrlen`、`if_hdrlen` 和 `if_mtu` 设置为以太网默认值，它调用 `if_attach` 注册接口，它安装 `ether_input` 和 `ether_output` 作为链路层输入和输出处理程序，它调用 `bpfattach(ifp, DLT_EN10MB, ETHER_HDR_LEN)` 使 `tcpdump -i mynet0` 立即工作。此调用之后，接口是活动的：用户态可以看到它、为其分配地址，并开始对其发出 ioctl。

### `mynet_destroy` 的骨架

销毁是创建的逆序操作。以下是骨架：

```c
static void
mynet_destroy(struct mynet_softc *sc)
{
    struct ifnet *ifp = sc->ifp;

    MYNET_LOCK(sc);
    sc->running = false;
    MYNET_UNLOCK(sc);

    callout_drain(&sc->rx_callout);

    ether_ifdetach(ifp);
    if_free(ifp);

    ifmedia_removeall(&sc->media);
    mtx_destroy(&sc->mtx);
    free(sc, M_MYNET);
}
```

我们将 softc 标记为不再运行，排空 callout 使没有计划的接收事件可以触发，调用 `ether_ifdetach` 注销接口，释放 ifnet，移除所有已分配的媒体条目，销毁互斥锁，并释放 softc。顺序很重要：你不能在 callout 可能仍在其上运行时释放 `ifnet`，也不能在 callout 可能仍会获取互斥锁时销毁它。`callout_drain` 为我们提供了同步保证，即它返回后不再有回调会触发。

### 注册克隆器

两个部分将 `mynet_create` 和 `mynet_destroy` 连接到内核：克隆器注册和模块处理函数。以下是克隆器代码：

```c
static int
mynet_clone_create(struct if_clone *ifc, int unit, caddr_t params)
{
    return (mynet_create_unit(unit));
}

static void
mynet_clone_destroy(struct ifnet *ifp)
{
    mynet_destroy((struct mynet_softc *)ifp->if_softc);
}

static void
vnet_mynet_init(const void *unused __unused)
{
    V_mynet_cloner = if_clone_simple(mynet_name, mynet_clone_create,
        mynet_clone_destroy, 0);
}
VNET_SYSINIT(vnet_mynet_init, SI_SUB_PSEUDO, SI_ORDER_ANY,
    vnet_mynet_init, NULL);

static void
vnet_mynet_uninit(const void *unused __unused)
{
    if_clone_detach(V_mynet_cloner);
}
VNET_SYSUNINIT(vnet_mynet_uninit, SI_SUB_INIT_IF, SI_ORDER_ANY,
    vnet_mynet_uninit, NULL);
```

`if_clone_simple` 注册一个简单克隆器，即名称匹配通过精确前缀进行的克隆器（`mynet` 后跟可选的单元编号）。`/usr/src/sys/net/if_disc.c` 在 `vnet_disc_init`（`disc` 驱动程序的 VNET 初始化例程）内部使用相同的调用。创建函数接收一个单元编号并负责生成一个新接口。销毁函数接收一个 `ifnet` 并负责将其移除。`SYSINIT` 和 `SYSUNINIT` 宏确保克隆器在模块加载时注册，在模块卸载时注销。

`mynet_create_unit` 辅助函数将两半粘合在一起。它接收一个单元编号，执行我们上面描述的分配，调用 `ether_ifattach`，成功时返回零，失败时返回错误。完整清单在 `lab01-skeleton/` 下的配套文件中。

### 模块处理函数

最后，是标准的模块样板代码：

```c
static int
mynet_modevent(module_t mod, int type, void *data __unused)
{
    switch (type) {
    case MOD_LOAD:
    case MOD_UNLOAD:
        return (0);
    default:
        return (EOPNOTSUPP);
    }
}

static moduledata_t mynet_mod = {
    "mynet",
    mynet_modevent,
    NULL
};

DECLARE_MODULE(mynet, mynet_mod, SI_SUB_PSEUDO, SI_ORDER_ANY);
MODULE_DEPEND(mynet, ether, 1, 1, 1);
MODULE_VERSION(mynet, 1);
```

模块处理函数本身没有做任何有趣的事情。真正的初始化发生在 `vnet_mynet_init` 中，`VNET_SYSINIT` 安排在 `SI_SUB_PSEUDO` 时调用它。这种分割对于非 VNET 驱动程序来说并非绝对必要，但遵循 `disc(4)` 和 `epair(4)` 的模式使我们的驱动程序为 VNET 使用做好准备，并与源代码树其余部分使用的约定匹配。

`MODULE_DEPEND(mynet, ether, 1, 1, 1)` 声明对 `ether` 模块的依赖，以便在我们尝试使用 `ether_ifattach` 之前加载以太网支持。`MODULE_VERSION(mynet, 1)` 声明我们自己的版本号，以便其他模块在需要时可以依赖我们。

### 深入了解接口克隆器

接口克隆器值得稍微绕道一下，因为它们驱动了伪驱动程序的大部分生命周期，而且 API 比我们目前使用的 `if_clone_simple` 调用略微丰富。

克隆器是注册到网络协议栈的命名工厂。它携带一个名称前缀、一个创建回调、一个销毁回调，以及可选的匹配回调。当用户态运行 `ifconfig mynet create` 时，协议栈遍历其克隆器列表，寻找前缀与字符串 `mynet` 匹配的克隆器。如果找到，它选取一个单元编号，调用创建回调，并返回结果接口的名称。

API 有两种风格。`if_clone_simple` 使用默认匹配规则注册克隆器：名称必须以克隆器的前缀开头，后面可以跟一个单元编号。`if_clone_advanced` 使用调用者提供的匹配函数注册克隆器，允许更灵活的命名。`epair(4)` 使用 `if_clone_advanced`，因为它的接口以成对形式出现，命名为 `epairXa` 和 `epairXb`。我们使用 `if_clone_simple`，因为 `mynet0`、`mynet1` 等已经足够好了。

在创建回调中，你有两条信息：克隆器本身（通过它可以查找兄弟接口）和请求的单元编号（如果用户没有指定，可能是 `IF_MAXUNIT`，在这种情况下你选择一个空闲单元）。在我们的驱动程序中，我们接受克隆器告诉我们的任何单元编号，并将其直接传递给 `if_initname`。

销毁回调更简单：它接收要销毁的接口的 `ifnet` 指针，必须拆卸一切。克隆器框架为我们处理接口列表；我们不需要自己维护一个。

当模块卸载时，`if_clone_detach` 遍历克隆器创建的接口列表，并为每个接口调用销毁回调。之后，克隆器本身被注销。这种两步拆卸就是使 `kldunload` 干净的原因：即使用户在卸载前忘记 `ifconfig mynet0 destroy`，克隆器也会处理它。

如果你的驱动程序需要向创建路径暴露额外的参数（例如，`epair` 风格驱动程序中的伙伴接口名称），克隆器框架支持创建回调的 `caddr_t params` 参数，它携带用户通过 `ifconfig mynet create foo bar` 提供的字节。我们在这里不使用该机制，但它存在且值得了解。

### `ether_ifattach` 内部发生了什么

我们在 `mynet_create_unit` 的末尾调用了 `ether_ifattach(ifp, sc->hwaddr)`，只说了它「做了大量的工作」。让我们打开 `/usr/src/sys/net/if_ethersubr.c` 看看那些工作到底是什么，因为理解它可以使我们驱动程序其余部分的行为变得可预测而不是神秘。

`ether_ifattach` 首先设置 `ifp->if_addrlen = ETHER_ADDR_LEN` 和 `ifp->if_hdrlen = ETHER_HDR_LEN`。这些字段告诉协议栈一个帧前面有多少字节的链路层寻址和头部。对于以太网，这两个值都是常量：6 字节的 MAC 和 14 字节的头部。

接下来，如果驱动程序尚未设置更大的值，它设置 `ifp->if_mtu = ETHERMTU`（1500 字节，IEEE 以太网默认值）。我们的驱动程序在 `if_alloc` 之后将 `if_mtu` 保留为零，所以 `ether_ifattach` 给我们默认值。我们之后可以覆盖它；支持巨型帧的驱动程序可能会在 `ether_ifattach` 之前将 `if_mtu` 设置为 9000。

然后它将链路层输出函数 `if_output` 设置为 `ether_output`。`ether_output` 是通用的三层到二层处理程序：它接收一个带有 IP 头部和目标地址的数据包，在需要时解析 ARP 或邻居发现，构造以太网头部，并调用 `if_transmit`。这个间接链就是允许来自套接字的 IP 数据包透明地通过协议栈并到达我们驱动程序的原因。

它将 `if_input` 设置为 `ether_input`。`ether_input` 是反向的：它接收一个完整的以太网帧，剥离以太网头部，根据 EtherType 进行分派，并将有效载荷上交给相应的协议（IPv4、IPv6、ARP、LLC 等）。当我们的驱动程序调用 `if_input(ifp, m)` 时，它实际上是在调用 `ether_input(ifp, m)`。

然后它将 MAC 地址存储在接口的地址列表中，使其通过 `getifaddrs(3)` 和 `ifconfig` 对用户态可见。这就是 `ifconfig mynet0` 打印 `ether` 行的方式。

然后它调用 `if_attach(ifp)`，将接口注册到全局列表，分配所需的协议栈端状态，并使接口对用户态可见。

最后它调用 `bpfattach(ifp, DLT_EN10MB, ETHER_HDR_LEN)`，使用以太网链路类型将接口注册到 BPF。从那一刻起，`tcpdump -i mynet0` 将找到该接口，并期望带有 14 字节以太网头部的帧。

一次函数调用做了大量的工作。手动完成所有这些是合法的（许多旧驱动程序这样做），但容易出错。`ether_ifattach` 是那些真正使编写驱动程序变得更容易的辅助函数之一，阅读其函数体是有回报的，因为它揭开了从「我分配了一个 ifnet」到「协议栈完全了解我的接口」之间发生的事情的神秘面纱。

互补函数 `ether_ifdetach` 以正确的逆序执行反向操作。它是在拆卸期间调用的正确函数，也是我们在 `mynet_destroy` 中调用的函数。

### 编译、加载和验证

此时，即使没有发送和接收逻辑，骨架也应该能够构建和加载。以下是验证流程：

```console
# cd examples/part-06/ch28-network-driver
# make
# kldload ./mynet.ko

# ifconfig mynet create
mynet0
# ifconfig mynet0
mynet0: flags=8802<BROADCAST,SIMPLEX,MULTICAST> metric 0 mtu 1500
        ether 02:a3:f1:22:bc:0d
        media: Ethernet autoselect
        status: no carrier
        groups: mynet
```

确切的 MAC 地址会有所不同，因为 `arc4rand` 每次给你一个不同的随机地址。输出的其余部分应该非常接近。如果是这样，你就成功了：你拥有一个活跃的、已注册的、命名的、有 MAC 地址的网络接口，对所有标准工具可见，而无需处理任何真正的数据包。这已经是一个重要的成就。

销毁接口并卸载模块以完成生命周期：

```console
# ifconfig mynet0 destroy
# kldunload mynet
```

`kldstat` 应该显示模块已消失。`ifconfig -a` 不应再列出 `mynet0`。如果留下了任何东西，我们将在第 8 节中介绍如何诊断它。

### 协议栈现在了解我们的哪些信息

`ether_ifattach` 返回后，协议栈了解我们接口的几个重要事实：

* 它是 `IFT_ETHER` 类型。
* 它支持广播、单工和多播。
* 它有一个特定的 MAC 地址。
* 它有一个 1500 字节的默认 MTU。
* 它有一个发送回调、一个 ioctl 回调、一个初始化回调和一个媒体处理程序。
* 它已通过 `DLT_EN10MB` 封装附加到 BPF。
* 其链路状态当前未定义（我们尚未调用 `if_link_state_change`）。

其他一切——数据包移动、计数器更新、链路状态——将在后续小节中活跃起来。骨架故意保持小型。这是你第一次可以指向你系统上的某个东西，诚实地说出「那是我的网络接口」。在这个句子上停下来想一想。它标志着本书中一个真正的里程碑。

### 常见错误

在本节中容易犯两个错误，它们都会产生令人困惑的症状。

第一个是忘记调用 `ether_ifattach` 而直接调用 `if_attach`。这完全合法，但会产生一个非以太网伪接口，你的驱动程序必须安装自己的 `if_input` 和 `if_output` 处理程序，而且在你自己调用 `bpfattach` 之前 `tcpdump` 无法工作。如果你看到一个看起来应该能工作的接口但 `tcpdump -i mynet0` 抱怨链路类型，检查你是否使用了 `ether_ifattach`。

第二个错误是用 `M_NOWAIT` 而不是 `M_WAITOK` 分配 softc。`M_NOWAIT` 在中断上下文中是正确的，但 `mynet_clone_create` 通过 `ifconfig create` 路径在常规用户上下文中运行，`M_WAITOK` 是正确的选择。在这里使用 `M_NOWAIT` 会引入一个罕见但无益的分配失败路径。

### 第3节小结

你现在有了一个可工作的骨架。接口已存在、已注册、拥有以太网地址，并且可以按需创建和销毁。协议栈准备好通过 `if_transmit`、`if_ioctl` 和 `if_init` 调用我们的驱动程序，但我们尚未实现这些回调的主体。第 4 节处理发送路径。这是你感受最深切的部分，因为一旦它工作，`ping` 就开始通过你的代码推送真正的字节。

## 第4节：处理数据包发送

发送是数据包流的出站半部分。当内核的网络协议栈决定一个数据包需要通过 `mynet0` 离开时，它将数据包打包在一个 mbuf 链中并调用我们的 `if_transmit` 回调。我们的工作是接受 mbuf，对它做适当的事情，然后释放它。在本节中，我们将构建一个完整的发送路径，验证 mbuf，更新计数器，捕获 BPF 以便 `tcpdump` 看到数据包，并处理该帧。因为 `mynet` 是一个没有真正线路的伪设备，我们最初在计数后丢弃数据包。这与 `disc(4)` 在 `/usr/src/sys/net/if_disc.c` 中所做的类似，足以端到端地演示完整的发送流程。

### 协议栈如何到达我们的驱动

在我们打开编辑器之前，让我们追踪一个数据包如何从进程到达我们的驱动程序。当一个进程在一个绑定到分配给 `mynet0` 的 IP 地址的 TCP 套接字上调用 `send(2)` 时，以下序列大致如下发生。不必担心记住每一步；重点是看到我们的代码在更大的图景中处于什么位置。

1. 套接字层将用户有效载荷复制到 mbuf 中并传递给 TCP。
2. TCP 对有效载荷进行分段，添加 TCP 头部，并将段传递给 IP。
3. IP 添加 IP 头部，查找路由，并通过 `ether_output` 将结果传递给以太网层。
4. `ether_output` 解析下一跳 MAC 地址（如果需要则通过 ARP），前置以太网头部，并在输出接口上调用 `if_transmit`。
5. 我们的 `if_transmit` 函数被调用，`ifp` 指向 `mynet0`，mbuf 指向准备好传输的完整以太网帧。

从那一刻起，帧就属于我们了。我们必须将其发送出去、干净地丢弃，或者排队等待稍后交付。无论选择哪种方式，我们必须恰好释放 mbuf 一次。双重释放会导致内核损坏，释放后使用会导致神秘崩溃，忘记释放会泄漏 mbuf 直到机器耗尽。

### 发送回调函数签名

`if_transmit` 回调的原型是：

```c
int mynet_transmit(struct ifnet *ifp, struct mbuf *m);
```

它在 `/usr/src/sys/net/if_var.h` 中声明为 typedef `if_transmit_fn_t`。返回值是一个 errno：成功时为零，或错误（例如 `ENOBUFS`）如果数据包无法排队。真正的网卡驱动程序很少返回非零值，因为它们更倾向于静默丢弃并递增 `IFCOUNTER_OERRORS`。模仿真实行为的伪驱动程序通常也这样做。

以下是我们将实现的完整回调：

```c
static int
mynet_transmit(struct ifnet *ifp, struct mbuf *m)
{
    struct mynet_softc *sc = ifp->if_softc;
    int len;

    if (m == NULL)
        return (0);
    M_ASSERTPKTHDR(m);

    /* Reject oversize frames. Leave a little slack for VLAN. */
    if (m->m_pkthdr.len > (ifp->if_mtu + sizeof(struct ether_vlan_header))) {
        m_freem(m);
        if_inc_counter(ifp, IFCOUNTER_OERRORS, 1);
        return (E2BIG);
    }

    /* If the interface is administratively down, drop. */
    if ((ifp->if_flags & IFF_UP) == 0 ||
        (ifp->if_drv_flags & IFF_DRV_RUNNING) == 0) {
        m_freem(m);
        if_inc_counter(ifp, IFCOUNTER_OERRORS, 1);
        return (ENETDOWN);
    }

    /* Let tcpdump see the outgoing packet. */
    BPF_MTAP(ifp, m);

    /* Count it. */
    len = m->m_pkthdr.len;
    if_inc_counter(ifp, IFCOUNTER_OPACKETS, 1);
    if_inc_counter(ifp, IFCOUNTER_OBYTES, len);
    if (m->m_flags & (M_BCAST | M_MCAST))
        if_inc_counter(ifp, IFCOUNTER_OMCASTS, 1);

    /* In a real NIC we would DMA this to hardware. Here we just drop. */
    m_freem(m);
    return (0);
}
```

让我们逐一讲解。这是发送例程的形状变得清晰的地方，所以值得慢慢阅读代码。

### NULL 检查

前两行处理协议栈用 NULL 指针调用我们的防御性情况。在正常操作中这不应该发生，但内核是防御性编程值得的地方。在 NULL 输入上返回 `0` 是标准习惯；`if_epair.c` 在 `epair_transmit` 的顶部也做了同样的事情。

### `M_ASSERTPKTHDR`

下一行是来自 `/usr/src/sys/sys/mbuf.h` 的宏，它断言 mbuf 已设置 `M_PKTHDR`。到达驱动程序发送回调的每个 mbuf 必须是数据包的头部，因此必须携带有效的数据包头部。断言这一点可以捕获系统中其他地方 mbuf 操作引起的错误。在生产内核中，断言被编译掉，但在源代码树中存在它记录了契约，在 `INVARIANTS` 内核中它在开发期间捕获不良使用。

### MTU 验证

注释 `/* Reject oversize frames. */` 下面的代码块拒绝大于接口 MTU 加上 VLAN 头部少量余量的数据包。`/usr/src/sys/net/if_epair.c` 中的 `epair_transmit` 做了完全相同的检查；寻找 `if (m->m_pkthdr.len > (ifp->if_mtu + sizeof(struct ether_vlan_header)))` 守卫，它 `m_freem` 帧并递增 `IFCOUNTER_OERRORS`。我们为 `ether_vlan_header` 留出余量，因为 VLAN 标记帧在基本以太网头部之外还携带四个额外字节，而且我们在第 3 节中声明了 `IFCAP_VLAN_MTU`，所以我们应该兑现该能力。

拒绝时，我们用 `m_freem(m)` 释放 mbuf 并递增 `IFCOUNTER_OERRORS`。我们还返回 `E2BIG` 作为给调用者的提示，尽管在实践中协议栈很少查看返回值，除非是决定是否在本地丢弃。

### 状态验证

注释 `/* If the interface is administratively down, drop. */` 下面的 `if` 代码块检查两个条件。`IFF_UP` 由 `ifconfig mynet0 up` 设置，由 `ifconfig mynet0 down` 清除，它是用户态表示接口应该或不应该承载流量的方式。`IFF_DRV_RUNNING` 是驱动程序内部的「我已分配好资源并准备好传输流量」标志。如果其中任何一个被清除，我们就没有理由发送数据包，所以我们丢弃它并递增错误计数器。

这个检查在所有情况下对正确性来说并非严格必要，因为协议栈通常避免通过 down 的接口路由流量。但防御性驱动程序仍然会检查，因为协议栈的状态视图和驱动程序的状态视图之间的竞争确实会发生，特别是在接口拆卸期间。

### BPF 捕获

`BPF_MTAP(ifp, m)` 是一个宏，如果接口上有活动的数据包捕获会话，它有条件地调用 BPF。在当前源代码树中它展开为 `bpf_mtap_if((_ifp), (_m))`。该宏定义在 `/usr/src/sys/net/bpf.h` 中。当 `tcpdump -i mynet0` 正在运行时，BPF 已附加到接口的 `if_bpf` 指针上，宏将传出数据包的副本交给它。当没有人监听时，宏快速返回，开销可忽略不计。

放置位置很重要。我们在丢弃之前捕获，因为我们希望 `tcpdump` 能看到数据包，即使我们正在模拟一个 down 的接口。真正的网卡驱动程序捕获得稍早一些，就在将帧交给硬件 DMA 之前，但思想是相同的。

### 计数器更新

每次发送相关有四个计数器：

* `IFCOUNTER_OPACKETS`：已发送的数据包数。
* `IFCOUNTER_OBYTES`：已发送的总字节数。
* `IFCOUNTER_OMCASTS`：已发送的多播或广播帧数。
* `IFCOUNTER_OERRORS`：发送期间观察到的错误数。

`if_inc_counter(ifp, IFCOUNTER_*, n)` 是更新这些计数器的正确方式。它定义在 `/usr/src/sys/net/if.c` 中，内部使用每 CPU 计数器，使来自多个 CPU 的并发调用不会竞争。不要直接访问 `if_data` 字段：内部结构多年来已经改变，访问器是稳定的接口。

因为协议栈已经计算了数据包长度并填充了 `m->m_pkthdr.len`，我们在释放 mbuf 之前将其缓存到本地 `len` 变量中。在 `m_freem(m)` 之后读取 `m->m_pkthdr.len` 将是释放后使用，所以本地变量不是风格选择，而是正确性选择。

### 最终丢弃

`m_freem(m)` 释放整个 mbuf 链。它通过 `m_next` 指针遍历链并释放其中的每个 mbuf。你不需要手动释放每一个。如果你只用 `m_free(m)`，你会释放第一个 mbuf 并泄漏其余的。混淆 `m_freem` 和 `m_free` 是最常见的初学者错误之一。惯用名称是：

* `m_free(m)`：释放单个 mbuf。在驱动程序中很少调用。
* `m_freem(m)`：释放整个链。这几乎总是你想要的。

在真正的网卡驱动程序中，我们不会使用 `m_freem(m)`，而是将帧交给硬件 DMA，稍后在发送完成中断中释放 mbuf。对于我们的伪驱动程序，我们丢弃它。这是源代码树中 `if_disc.c` 的行为：模拟发送，释放 mbuf，然后返回。

### 队列刷新回调

除了 `if_transmit` 之外，协议栈期望一个名为 `if_qflush` 的简单回调。它在协议栈想要刷新驱动程序内部排队的任何数据包时被调用。因为我们的驱动程序不排队，回调没有工作要做：

```c
static void
mynet_qflush(struct ifnet *ifp __unused)
{
}
```

这与 `/usr/src/sys/net/if_epair.c` 中的 `epair_qflush` 相同。维护自己数据包队列的驱动程序（现在比以前少见了）在这里有更多工作要做。我们没有。

### `mynet_init` 回调

在第 3 节中分配的第三个回调是 `mynet_init`，即协议栈在接口转换为 up 状态时调用的函数。对我们来说很简单：

```c
static void
mynet_init(void *arg)
{
    struct mynet_softc *sc = arg;

    MYNET_LOCK(sc);
    sc->running = true;
    sc->ifp->if_drv_flags |= IFF_DRV_RUNNING;
    sc->ifp->if_drv_flags &= ~IFF_DRV_OACTIVE;
    callout_reset(&sc->rx_callout, sc->rx_interval_hz,
        mynet_rx_timer, sc);
    MYNET_UNLOCK(sc);

    if_link_state_change(sc->ifp, LINK_STATE_UP);
}
```

在初始化时，我们标记自己正在运行，清除 `IFF_DRV_OACTIVE`（一个表示「发送队列已满，在我清除之前不要再调用我」的标志），启动我们将在第 5 节中描述的接收模拟 callout，并宣布链路已连接。末尾的 `if_link_state_change` 调用使 `ifconfig` 在此接口上报告 `status: active`。记住放置顺序：我们先设置 `IFF_DRV_RUNNING`，然后宣布链路，按此顺序。反转顺序会告诉协议栈在一个驱动程序仍在初始化的接口上链路已连接，协议栈可能在我们准备好之前就开始向我们推送流量。

### 仔细审视顺序和锁

上面的代码足够简单，让人感觉锁定是多余的。为什么我们需要一个互斥锁？有两个原因。

第一个原因是 `if_transmit` 和 `if_ioctl` 并发运行。协议栈可以在一个 CPU 上调用 `if_transmit`，而用户态在另一个 CPU 上发出 `ifconfig mynet0 down`，这转化为 `if_ioctl(SIOCSIFFLAGS)` 在那个 CPU 上运行。没有互斥锁，这两个回调可以同时读写 softc 状态。互斥锁让我们能够推理状态转换。

第二个原因是第 5 节中基于 callout 的接收模拟在触发时接触 softc。没有互斥锁，callout 和 `if_ioctl` 可能冲突，你会得到经典的「我正在遍历的列表刚刚在我脚下改变了」风格的错误。同样，每个 softc 一个互斥锁足以使这些交互安全。

我们选择了一个简单的锁定规则：softc 互斥锁是大锁。发送快速路径之外的每个 softc 访问都获取它。`mynet_transmit` 中的发送快速路径不获取互斥锁，因为 `if_transmit` 为并发调用者设计，我们只接触 ifnet 计数器和 BPF，它们本身都是线程安全的。如果我们要添加由发送更新的驱动程序特定共享状态，我们会为该状态添加更细粒度的锁。

这是一个简化。真正的高性能网卡驱动程序使用更复杂的锁定，通常有每队列锁、每 CPU 状态和每数据包健全性检查。单互斥锁设计对于伪驱动程序和任何低速率接口来说绝对没问题；对于生产环境的 100Gb 驱动程序它会成为瓶颈，这也是现代 iflib 框架存在的原因之一。我们将在后面的章节中涉及 iflib。

### 使用 `m_pullup` 进行数据包处理

真正的网络驱动程序经常需要在决定如何处理数据包之前读取数据包深处的字段。VLAN 驱动程序需要读取 802.1Q 标签。桥接驱动程序需要读取源 MAC 以更新转发表。硬件卸载驱动程序需要读取 IP 和 TCP 头部以决定是否可以在硬件中计算校验和。

问题在于接收到的 mbuf 链不保证任何特定字节位于任何特定 mbuf 中。第一个 mbuf 可能只保存前十四个字节（以太网头部），而下一个 mbuf 保存其余部分。一个将 `mtod(m, struct ip *)` 转换并越过以太网头部的驱动程序将读取到无意义的数据，除非它首先确保所需的字节是连续的。

内核正是为此目的提供了 `m_pullup(m, len)`。`m_pullup` 保证 mbuf 链的前 `len` 个字节位于第一个 mbuf 中。如果已经是这样，它就是无操作。如果不是，它通过将字节移入第一个 mbuf 来重塑链，如果第一个 mbuf 太小则可能分配一个新的 mbuf。它返回一个（可能不同的）mbuf 指针，或分配失败时返回 NULL，在这种情况下 mbuf 链已被为你释放。

需要检查头部的驱动程序的习惯写法是：

```c
m = m_pullup(m, sizeof(struct ether_header) + sizeof(struct ip));
if (m == NULL) {
    if_inc_counter(ifp, IFCOUNTER_IQDROPS, 1);
    return;
}
eh = mtod(m, struct ether_header *);
ip = (struct ip *)(eh + 1);
```

`mynet` 不需要这样做，因为我们在发送路径中不检查数据包内容。但你会在真正的驱动程序中到处看到 `m_pullup`，特别是在接收端和二层辅助函数中。

一个相关的函数 `m_copydata(m, offset, len, buf)` 将字节从 mbuf 链复制到调用者提供的缓冲区中。当你想读取一些字节而不修改链时，这是正确的工具。`m_copyback` 是另一个方向：在给定偏移量处向链写入字节，如果需要则扩展链。

另一个常用的辅助函数是 `m_defrag(m, how)`，它将链扁平化为单个（大型）mbuf。这被硬件具有最大散点-收集计数的驱动程序使用。如果发送帧跨越的 mbuf 数量超过硬件能处理的数量，驱动程序会退回到 `m_defrag`，将有效载荷复制到一个连续的单个集群中。

在阅读真正驱动程序的过程中你会遇到所有这些函数。现在，知道它们存在，并且知道 mbuf 布局是真正的驱动程序必须认真对待的事情，就足够了。

### 深入了解 mbuf 结构

因为 mbuf 是网络协议栈的通用货币，在它们的结构上多花几页时间是值得的。驱动程序对 mbuf 所做的决策决定了驱动程序是否快速、正确和可维护。

mbuf 结构体本身位于 `/usr/src/sys/sys/mbuf.h` 中。在 FreeBSD 14.3 中，其磁盘布局大致如下（为教学简化）：

```c
struct mbuf {
    struct m_hdr    m_hdr;      /* fields common to every mbuf */
    union {
        struct {
            struct pkthdr m_pkthdr;  /* packet header, if M_PKTHDR */
            union {
                struct m_ext m_ext;  /* external storage, if M_EXT */
                char         m_pktdat[MLEN - sizeof(struct pkthdr)];
            } MH_dat;
        } MH;
        char    M_databuf[MLEN]; /* when no packet header */
    } M_dat;
};
```

两个联合体内部有两个联合体变体。该布局捕获了 mbuf 可以处于几种模式之一的事实：

* 数据内联存储的普通 mbuf（约 200 字节可用）。
* 数据内联存储的数据包头部 mbuf（由于头部占空间，可用空间略少）。
* 数据存储在外部集群（`m_ext`）中的数据包头部 mbuf。
* 数据存储在外部集群中的非头部 mbuf。

`m_flags` 字段通过 `M_PKTHDR` 和 `M_EXT` 位指示哪个变体生效。

集群是一个更大的预分配缓冲区，在现代 FreeBSD 上通常为 2048 字节。mbuf 在 `m_ext.ext_buf` 中保存指向集群的指针，集群通过 `m_ext.ext_count` 进行引用计数。集群存在是因为许多数据包大于普通 mbuf 可以容纳的大小，而为每个大数据包分配新缓冲区是昂贵的。

当你调用 `MGETHDR(m, M_NOWAIT, MT_DATA)` 时，你会得到一个带有内联数据的数据包头部 mbuf。当你调用 `m_getcl(M_NOWAIT, MT_DATA, M_PKTHDR)` 时，你会得到一个附加了外部集群的数据包头部 mbuf。第二种形式可以在不链接的情况下保存约 2000 字节，这对以太网大小的数据包很方便。

### mbuf 链与分散-聚集

因为单个 mbuf 只能容纳有限的字节，许多数据包跨越通过 `m_next` 链接的多个 mbuf。头部 mbuf 上的 `m_pkthdr.len` 字段保存总数据包长度；链中每个 mbuf 上的 `m_len` 保存该 mbuf 的贡献。它们的关系是 `m_pkthdr.len == sum(链中所有 m_len)`，任何不匹配都是错误。

这种链接有几个优点。它让协议栈可以廉价地前置头部：要添加以太网头部，协议栈可以分配一个新的 mbuf，填入头部，并将其链接为新的头部。它让协议栈可以廉价地分割数据包：TCP 可以通过遍历链而不是复制数据来对大有效载荷进行分段。它让硬件使用分散-收集 DMA：网卡可以通过发出多个 DMA 描述符（每个 mbuf 一个）来传输链。

代价是驱动程序必须小心地遍历链。如果你进行 `mtod(m, struct ip *)` 转换，而 IP 头部分布在第一个和第二个 mbuf 之间，你会读到垃圾数据。`m_pullup` 是针对该错误的防御，每个严肃的驱动程序在需要检查头部时都使用它。

### mbuf 类型及其含义

每个 mbuf 上的 `m_type` 字段分类其用途：

* `MT_DATA`：普通数据包数据。这是你用于网络数据包的类型。
* `MT_HEADER`：专门用于保存协议头部的 mbuf。
* `MT_SONAME`：套接字地址结构。由套接字层代码使用。
* `MT_CONTROL`：辅助套接字控制数据。
* `MT_NOINIT`：未初始化的 mbuf。驱动程序永远不会看到。

对于驱动程序代码，`MT_DATA` 几乎总是正确的。协议栈在内部处理其他类型。

### 数据包头字段

头部 mbuf 上的 `m_pkthdr` 结构体携带随数据包通过协议栈的字段。对驱动程序作者最相关的一些：

* `len`：mbuf 链的总长度。
* `rcvif`：接收数据包的接口。驱动程序在构建接收到的 mbuf 时设置此项。
* `flowid` 和 `rsstype`：数据包流的哈希，用于多队列分发。
* `csum_flags` 和 `csum_data`：硬件校验和状态。具有 TX 校验和卸载的驱动程序读取这些；具有 RX 校验和卸载的驱动程序写入这些。
* `ether_vtag` 和 `m_flags` 中的 `M_VLANTAG` 标志：硬件提取的 VLAN 标签，如果正在使用 VLAN 硬件标记。
* `vt_nrecs` 和其他 VLAN 字段：用于更复杂的 VLAN 配置。
* `tso_segsz`：TSO 帧的段大小。

这些字段中的大多数在数据包到达驱动程序之前由更高层设置。就我们的目的而言，在接收期间设置 `rcvif`，在发送期间读取 `len` 就够了。其他字段是 iflib 及其前身用于卸载协调的钩子；伪驱动程序可以安全地忽略它们。

### 引用计数的外部缓冲区

当集群附加到 mbuf 时，集群是引用计数的。这允许数据包复制（通过 `m_copypacket`）而不复制有效载荷：两个 mbuf 可以共享同一个集群，集群只有在两个 mbuf 都释放其引用时才被释放。BPF 使用这种机制在不强制复制的情况下捕获数据包。

对于驱动程序代码来说，这主要是透明的。你对 mbuf 调用 `m_freem`，如果 mbuf 有外部集群，集群的引用计数递减；如果达到零，集群被释放。你不必显式地考虑引用计数。但你应该知道它们存在，因为它们解释了为什么 `BPF_MTAP` 可以很便宜：它不复制数据包，只是获取一个额外的引用。

### 接收分配模式

真正的网卡驱动程序通常在初始化时分配 mbuf 并附加集群，用这些 mbuf 填充接收环，并让硬件 DMA 写入它们。模式是：

```c
for (i = 0; i < RX_RING_SIZE; i++) {
    struct mbuf *m = m_getcl(M_WAITOK, MT_DATA, M_PKTHDR);
    rx_ring[i].mbuf = m;
    rx_ring[i].dma_addr = pmap_kextract((vm_offset_t)mtod(m, char *));
    rx_ring[i].desc->addr = rx_ring[i].dma_addr;
    rx_ring[i].desc->status = 0;
}
```

当硬件接收到数据包时，它将数据包数据写入由某个描述符指向的集群中，设置状态以指示完成，并引发中断。驱动程序的接收例程查看状态，获取 mbuf，从描述符的长度字段设置 `m->m_pkthdr.len` 和 `m->m_len`，捕获 BPF，调用 `if_input`，然后为描述符分配替换的 mbuf。

我们的伪驱动程序使用一个更简单的模式：每次接收定时器触发时分配一个新的 mbuf。这对于教学驱动程序来说完全没问题，因为分配率很低。在更高速率下你会想要预分配模式，因为在初始化时批量分配 mbuf 并回收它们比每个数据包分配一个要便宜得多。

### 与 mbuf 相关的常见错误

即使了解了上述内容，驱动程序代码中仍然有一些反复出现的错误：

* 在链头部使用 `m_free` 而不是 `m_freem`。你释放了第一个 mbuf 并泄漏了其余的。
* 在构建数据包时忘记正确设置 `m_pkthdr.len`。协议栈读取 `m_pkthdr.len` 而不是遍历链，所以如果两者不一致，解码会静默失败。
* 在 `m_freem` 之后读取 `m_pkthdr.len`。总是在释放之前将长度缓存到本地变量中。
* 混淆 `m->m_len`（此 mbuf 的长度）与 `m->m_pkthdr.len`（链的总长度）。对于单个 mbuf 数据包它们相等；对于链它们不同。
* 不遍历链就读取超过 `m_len` 的内容。如果你需要第一个 mbuf 之外的字节，使用 `m_pullup` 或 `m_copydata`。
* 修改不属于你的 mbuf。一旦你将 mbuf 交给 `if_input`，它就不再是你的了。
* 分配时不检查 NULL。`m_gethdr(M_NOWAIT, ...)` 在内存压力下可能返回 NULL，驱动程序必须优雅地处理这种情况。

如果你了解规则，这些错误很容易避免，阅读其他驱动程序是内化它们的最佳方式。

### 真实驱动程序中的多队列发送

现代硬件网卡可以在多个队列上并行发送。一个 10Gb 网卡通常有八个或十六个发送队列，每个都有自己的硬件环形缓冲区、自己的 DMA 描述符和自己的完成中断。驱动程序根据数据包源地址和目标地址的哈希在队列之间分配出站数据包，使来自不同流的流量去往不同队列，并可以在不同的 CPU 核心上并发处理。

这远远超出了我们伪驱动程序的需求。但该模式值得认识，因为它在生产驱动程序中显著出现。关键部分是：

* 一个队列选择函数，接受 mbuf 并返回驱动程序队列数组的索引。`mynet` 只有一个队列（或零个，取决于你怎么计数），所以这一步是微不足道的。真正的驱动程序通常使用 `m->m_pkthdr.flowid` 作为预计算的哈希。
* 每队列锁和每队列软件队列（通常由 `drbr(9)` 管理），允许并发生产者无竞争地入队数据包。
* 发送触发，在生产者入队且硬件空闲时将软件队列排入硬件。
* 完成回调，通常来自硬件中断，释放已完成传输的 mbuf。

`if_transmit` 原型被设计为自然地适应这种模式。调用者产生一个 mbuf 并将其交给 `if_transmit`。驱动程序要么立即排队（在我们这样的简单情况下），要么将其分派到适当的硬件队列（在多队列情况下）。无论哪种方式，调用者看到的都是一个单一的函数调用，不需要知道下面有多少个队列。

我们将在后面的章节讨论 iflib 时回到多队列设计。现在，知道我们在这里构建的单队列模型是真正的驱动程序所扩展的简化就足够了。

### 关于 `drbr(9)` 辅助库的补充说明

`drbr` 代表「驱动程序环形缓冲区」，它是一个辅助库，用于想要维护自己的每队列软件队列的驱动程序。API 定义并实现为 `/usr/src/sys/net/ifq.h` 中的 `static __inline` 函数；没有单独的 `drbr.c` 或 `drbr.h` 文件。辅助函数包装了底层的 `buf_ring(9)` 环形缓冲区，提供显式的入队和出队操作，以及捕获 BPF、计数数据包和与发送线程同步的辅助函数。`drbr` 构建的多生产者、单消费者形状，这是发送队列的典型形状，其中许多线程入队但单个出队线程将环形缓冲区排入硬件。

使用 `drbr` 的驱动程序通常有一个类似以下草图的发送函数：

```c
int
my_transmit(struct ifnet *ifp, struct mbuf *m)
{
    struct mydrv_softc *sc = ifp->if_softc;
    struct mydrv_txqueue *txq = select_queue(sc, m);
    int error;

    error = drbr_enqueue(ifp, txq->br, m);
    if (error)
        return (error);
    taskqueue_enqueue(txq->tq, &txq->tx_task);
    return (0);
}
```

生产者入队到环形缓冲区并触发 taskqueue。taskqueue 消费者然后从环形缓冲区出队并将帧交给硬件。这解耦了生产者（可以是任何 CPU）和消费者（在每个队列的专用工作线程上运行），这正是多核系统上效果良好的结构。

`mynet` 不使用 `drbr`，因为我们既没有多个队列也没有硬件可以触发。但该模式值得看一次，因为它出现在源代码树中每个注重性能的驱动程序中。

### 测试发送路径

像第 3 节那样构建、加载并创建接口，然后向其发送流量：

```console
# kldload ./mynet.ko
# ifconfig mynet create
mynet0
# ifconfig mynet0 inet 192.0.2.1/24 up
# ping -c 1 192.0.2.99
PING 192.0.2.99 (192.0.2.99): 56 data bytes
--- 192.0.2.99 ping statistics ---
1 packets transmitted, 0 packets received, 100.0% packet loss
# netstat -in -I mynet0
Name    Mtu Network     Address              Ipkts Ierrs ...  Opkts Oerrs
mynet0 1500 <Link#12>   02:a3:f1:22:bc:0d        0     0        1     0
mynet0    - 192.0.2.0/24 192.0.2.1                0     -        0     -
```

关键行是 `Opkts 1`。即使 ping 没有收到回复，我们可以看到一个数据包已经通过我们的驱动程序发送。没有回复的原因是 `mynet0` 是一个伪接口，另一端什么也没有。我们将在第 5 节中为它提供模拟到达路径。

在另一个终端保持 `tcpdump -i mynet0 -n` 运行，重复 `ping`，你会看到传出的 ARP 请求和 IPv4 数据包被捕获。这确认 `BPF_MTAP` 已正确连接。

### 陷阱

一些错误在学生代码甚至经验丰富的驱动程序中反复出现。让我们逐一讲解，以便你学会识别它们。

**两次释放 mbuf。** 如果你的发送函数有多个退出路径，其中之一忘记跳过 `m_freem`，同一个 mbuf 会被释放两次。内核通常会因损坏的空闲列表消息而崩溃。修复方法是构造函数使用拥有释放操作的单个退出，或在释放后将 `m` 置为 null 并在再次释放之前检查。

**完全未释放 mbuf。** 同一错误的另一面。如果你从 `if_transmit` 返回而未释放或排队 mbuf，你就会泄漏它。在低速驱动程序中，这可能需要数小时才能注意到；在高速驱动程序中，机器很快就会耗尽 mbuf 内存。`vmstat -z | grep mbuf` 是发现此问题的最佳工具。

**假设 mbuf 适合单个内存块。** 即使是一个简单的以太网帧也可以分布在链中的多个 mbuf 中，特别是在 IP 分片或 TCP 分段之后。如果你需要检查头部，要么使用 `m_pullup` 将头部拉入第一个 mbuf，要么小心地遍历链。

**忘记捕获 BPF。** `tcpdump -i mynet0` 对接收到的数据包仍然有效，但会遗漏发送的数据包，你的调试会更加困难，因为对话的两半会出现不对称。

**在 `m_freem` 之后更新计数器。** 我们已经提到过这一点。总是在释放之前将 `m->m_pkthdr.len` 读入本地变量，或在释放之前完成所有计数器更新。

**用错误参数调用 `if_link_state_change`。** `LINK_STATE_UP`、`LINK_STATE_DOWN` 和 `LINK_STATE_UNKNOWN` 是 `/usr/src/sys/net/if.h` 中定义的三个值。传递随机整数如 `1` 可能碰巧匹配 `LINK_STATE_DOWN`，但会使代码不可读且脆弱。

### 第4节小结

发送路径是网络协议栈和驱动程序如何合作的最清晰演示。我们接受 mbuf，验证它，计数它，让 BPF 看到它，然后释放它。真正的硬件驱动程序在底部添加 DMA 和硬件环；骨架保持不变。

我们还缺少一个大的部分：接收路径。没有它，我们的接口只说但从不听。第 5 节构建那一半。

## 第5节：处理数据包接收

接收是数据包流入站的半部分。数据包从传输层到达，驱动程序负责将它们转换为 mbuf，交给 BPF，并通过 `if_input` 向上传递给协议栈。在真正的网卡驱动程序中，到达是一个中断或描述符环完成。在我们的伪驱动程序中，我们将用每秒触发一次的 callout 来模拟到达并构造合成数据包。机制是人工的，但代码路径与真正的驱动程序在初始描述符环出队之后所做的相同。

### 回调方向

发送向下流动：协议栈调用驱动程序。接收向上流动：驱动程序调用协议栈。你不注册一个供协议栈调用的接收回调。相反，每当数据包到达时，你调用 `if_input(ifp, m)`（或等价的 `(*ifp->if_input)(ifp, m)`），协议栈接管。`ether_ifattach` 安排 `ifp->if_input` 指向 `ether_input`，所以当我们调用 `if_input` 时，以太网层接收帧，剥离以太网头部，根据 EtherType 进行分派，并将有效载荷上交给 IPv4、IPv6、ARP 或其所属的位置。

这是从 `if_transmit` 的一个重要思维转变。协议栈不轮询你的驱动程序。它等待被调用。你的驱动程序是接收的主动方。每当你有准备好的帧，你就发起调用。协议栈做其余的事情。

### 模拟到达

让我们构建一个模拟到达路径。思路是：每秒唤醒一次，构建一个包含有效以太网帧的小型 mbuf，并将其送入协议栈。该帧将是一个针对不存在 IP 地址的广播 ARP 请求。这很容易构建，对测试很有用，因为 `tcpdump` 会清晰地显示它，而且对系统的其余部分无害。

首先，callout 处理程序：

```c
static void
mynet_rx_timer(void *arg)
{
    struct mynet_softc *sc = arg;
    struct ifnet *ifp = sc->ifp;

    MYNET_ASSERT(sc);
    if (!sc->running) {
        return;
    }
    callout_reset(&sc->rx_callout, sc->rx_interval_hz,
        mynet_rx_timer, sc);
    MYNET_UNLOCK(sc);

    mynet_rx_fake_arp(sc);

    MYNET_LOCK(sc);
}
```

callout 使用 `callout_init_mtx` 和 softc 互斥锁初始化，因此系统在调用我们之前会获取我们的互斥锁。这使我们可以免费使用 `MYNET_ASSERT`：锁已经被持有。我们检查是否仍在运行，重新调度定时器以等待下一个节拍，释放锁，执行实际工作，然后在返回时重新获取锁。释放锁很重要，因为 `if_input` 可能需要一些时间并可能获取其他锁。在持有驱动程序互斥锁的情况下向上调用协议栈是锁序反转的经典配方。

接下来，数据包构造本身：

```c
static void
mynet_rx_fake_arp(struct mynet_softc *sc)
{
    struct ifnet *ifp = sc->ifp;
    struct mbuf *m;
    struct ether_header *eh;
    struct arphdr *ah;
    uint8_t *payload;
    size_t frame_len;

    frame_len = sizeof(*eh) + sizeof(*ah) + 2 * (ETHER_ADDR_LEN + 4);
    MGETHDR(m, M_NOWAIT, MT_DATA);
    if (m == NULL) {
        if_inc_counter(ifp, IFCOUNTER_IQDROPS, 1);
        return;
    }

    m->m_pkthdr.len = m->m_len = frame_len;
    m->m_pkthdr.rcvif = ifp;

    eh = mtod(m, struct ether_header *);
    memset(eh->ether_dhost, 0xff, ETHER_ADDR_LEN);   /* broadcast */
    memcpy(eh->ether_shost, sc->hwaddr, ETHER_ADDR_LEN);
    eh->ether_type = htons(ETHERTYPE_ARP);

    ah = (struct arphdr *)(eh + 1);
    ah->ar_hrd = htons(ARPHRD_ETHER);
    ah->ar_pro = htons(ETHERTYPE_IP);
    ah->ar_hln = ETHER_ADDR_LEN;
    ah->ar_pln = 4;
    ah->ar_op  = htons(ARPOP_REQUEST);

    payload = (uint8_t *)(ah + 1);
    memcpy(payload, sc->hwaddr, ETHER_ADDR_LEN);     /* sender MAC */
    payload += ETHER_ADDR_LEN;
    memset(payload, 0, 4);                            /* sender IP 0.0.0.0 */
    payload += 4;
    memset(payload, 0, ETHER_ADDR_LEN);               /* target MAC */
    payload += ETHER_ADDR_LEN;
    memcpy(payload, "\xc0\x00\x02\x63", 4);          /* target IP 192.0.2.99 */

    BPF_MTAP(ifp, m);

    if_inc_counter(ifp, IFCOUNTER_IPACKETS, 1);
    if_inc_counter(ifp, IFCOUNTER_IBYTES, frame_len);

    if_input(ifp, m);
}
```

这里有很多内容需要消化，但大部分都很简单。让我们逐步讲解。

### `MGETHDR`：分配链的头部

`MGETHDR(m, M_NOWAIT, MT_DATA)` 分配一个新的 mbuf 并将其准备为数据包链的头部。它通过 `/usr/src/sys/sys/mbuf.h` 中的兼容宏块展开为 `m_gethdr(M_NOWAIT, MT_DATA)`（`#define MGETHDR(m, how, type) ((m) = m_gethdr((how), (type)))` 条目，紧挨着 `MGET` 和 `MCLGET`）。`M_NOWAIT` 告诉分配器失败而不是睡眠，这是合适的，因为我们可能在不允许睡眠的上下文中运行（这个特定的回调是一个 callout，不能睡眠）。`MT_DATA` 是通用数据的 mbuf 类型。

分配失败时，我们递增 `IFCOUNTER_IQDROPS`（输入队列丢弃）并返回。大多数驱动程序以这种方式统计由 mbuf 耗尽导致的丢弃。

### 设置数据包头字段

一旦我们有了 mbuf，我们在数据包头部中设置三个字段：

* `m->m_pkthdr.len`：数据包的总长度。这是链中所有 `m_len` 的总和。对于像我们这样的单 mbuf 数据包，`m_pkthdr.len` 等于 `m_len`。
* `m->m_len`：此 mbuf 中数据的长度。我们将整个帧存储在第一个（也是唯一的）mbuf 中。
* `m->m_pkthdr.rcvif`：数据包到达的接口。协议栈使用此字段进行路由决策和报告。

一个小的 mbuf（约 256 字节）可以舒适地容纳我们 42 字节的以太网 ARP 帧。如果我们构建一个更大的帧，我们会使用 `MGET` 和外部缓冲区，或使用 `m_getcl` 获取集群支持的 mbuf，或将多个 mbuf 链接在一起。我们将在后面的章节中重新讨论这些模式。

### 写入以太网头部

`mtod(m, struct ether_header *)` 是 `/usr/src/sys/sys/mbuf.h` 中的一个宏，它将 `m_data` 转换为所请求类型的指针。它代表"mbuf to data"。我们使用它在数据包起始处获取一个可写的 `struct ether_header` 指针，并填入目标 MAC（广播 `ff:ff:ff:ff:ff:ff`）、源 MAC（我们接口的 MAC）和 EtherType（`ETHERTYPE_ARP`，网络字节序）。

以太网头部是协议栈期望我们接口提供的最小二层封装，因为我们使用 `ether_ifattach` 附加。`ether_input` 将剥离此头部并根据 EtherType 进行分派。

### 写入 ARP 报文体

在以太网头部之后是 ARP 头部本身，然后是 ARP 有效载荷（发送方 MAC、发送方 IP、目标 MAC、目标 IP）。字段名称和常量来自 `/usr/src/sys/net/if_arp.h`。我们放入一个真实的发送方 MAC（我们的），发送方 IP 为 `0.0.0.0`，目标 MAC 为零，目标 IP 为 `192.0.2.99`。最后一个地址位于 RFC 5737 为文档和示例保留的 TEST-NET-1 范围内，对于永远不会离开我们系统的合成数据包来说，这是一个礼貌的选择。

这些都不是生产级的 ARP 代码。我们不是试图解析任何东西。我们正在生成一个格式良好的帧，以太网输入层会识别它、解析它、在计数器中记录它，然后丢弃它（因为目标 IP 不是我们的）。这正是教学驱动程序所需的真实度级别。

### 交给 BPF

`BPF_MTAP(ifp, m)` 给 `tcpdump` 一个查看传入帧的机会。我们在调用 `if_input` 之前捕获，因为 `if_input` 可能以使捕获显示令人困惑数据的方式修改 mbuf。真正的驱动程序总是在消费之前捕获。

### 递增输入计数器

`IFCOUNTER_IPACKETS` 和 `IFCOUNTER_IBYTES` 分别统计接收到的数据包数和字节数。如果帧是广播或多播，我们还需要递增 `IFCOUNTER_IMCASTS`。这里为简洁起见省略了，但完整的配套文件包含它。

### 调用 `if_input`

`if_input(ifp, m)` 是最后一步。它是 `/usr/src/sys/net/if_var.h` 中的一个内联辅助函数，解引用 `ifp->if_input`（`ether_ifattach` 将其设置为 `ether_input`）并调用它。从那一刻起，mbuf 就是协议栈的责任了。如果协议栈接受数据包，它会使用它并最终释放它。如果协议栈拒绝数据包，它会释放它并递增 `IFCOUNTER_IERRORS`。无论哪种方式，我们都不得再次触碰 `m`。

这是与发送互补的规则：在发送中，驱动程序拥有 mbuf 直到它被释放或交给硬件；在接收中，协议栈在你调用 `if_input` 的那一刻就取得了所有权。正确处理这些所有权规则是编写网络驱动程序中最重要的纪律。

### 验证接收路径

构建并加载更新后的驱动程序，启动接口，然后观察 `tcpdump`：

```console
# kldload ./mynet.ko
# ifconfig mynet create
mynet0
# ifconfig mynet0 inet 192.0.2.1/24 up
# tcpdump -i mynet0 -n
tcpdump: verbose output suppressed, use -v or -vv for full protocol decode
listening on mynet0, link-type EN10MB (Ethernet), capture size 262144 bytes
14:22:01.000 02:a3:f1:22:bc:0d > ff:ff:ff:ff:ff:ff, ethertype ARP, Request who-has 192.0.2.99 tell 0.0.0.0, length 28
14:22:02.000 02:a3:f1:22:bc:0d > ff:ff:ff:ff:ff:ff, ethertype ARP, Request who-has 192.0.2.99 tell 0.0.0.0, length 28
...
```

每秒你应该看到一个合成的 ARP 请求飞过。如果你随后检查 `netstat -in -I mynet0`，`Ipkts` 计数器应该在不断攀升。协议栈接受数据包，检查 ARP，判断它不是一个发给自己的查询（因为 `192.0.2.99` 未分配给该接口），然后静默丢弃它。这正是我们想要的结果，它证明完整的接收路径工作正常。

### 所有权：一张图

因为所有权规则如此重要，用一张图来总结它们会很有帮助。下图总结了在每个阶段谁拥有 mbuf：

```text
Transmit:
  stack allocates mbuf
  stack calls if_transmit(ifp, m)    <-- ownership handed to driver
  driver inspects, counts, taps, drops or sends
  driver must m_freem(m) exactly once
  return 0 to stack

Receive:
  driver allocates mbuf (MGETHDR/MGET)
  driver fills in data
  driver taps BPF
  driver calls if_input(ifp, m)      <-- ownership handed to stack
  driver MUST NOT touch m again
```

如果你把这两张图记在脑海中，你就不会在自己的驱动程序中搞错 mbuf 所有权。

### 在竞争条件下保持接收安全

生产驱动程序的接收路径通常从中断处理程序或硬件队列完成例程中调用，运行在一个 CPU 上，而另一个 CPU 可能正在发送或处理 ioctl。我们这里展示的模式是安全的，因为：

* 我们在「我是否在运行？」检查周围持有互斥锁。
* 我们在 mbuf 分配和数据包构造的繁重工作之前释放互斥锁。
* 我们在调用 `if_input` 之前释放互斥锁，`if_input` 可能反过来调用协议栈并获取其他锁。
* 我们在 `if_input` 返回后重新获取互斥锁，以便 callout 框架能看到一致的状态。

真正的驱动程序通常添加每 CPU 接收队列、通过 taskqueue 的延迟处理和无锁计数器。所有这些都只是同一模式的改进。核心不变量保持不变：不要在持有驱动锁的情况下向上调用，不要在 mbuf 已向上提交后触碰它。

### 替代方案：使用 `if_epoch`

FreeBSD 12 引入了网络 epoch 机制 `net_epoch`，用于在不使用长生命周期锁的情况下访问某些数据结构。现代驱动程序通常在接收代码周围进入 net epoch，使其对路由表、ARP 表和 `ifnet` 列表某些部分的访问既安全又快速。你会看到许多驱动程序中使用 `NET_EPOCH_ENTER(et)` 和 `NET_EPOCH_EXIT(et)`。对于我们简单的伪驱动程序，进入 net epoch 会增加我们不需要的复杂性。我们在这里提及它，是为了让你在阅读 `if_em.c` 或 `if_bge.c` 时能认出它，我们将在后面的章节中回到这个话题。

### 真实网卡驱动程序中的接收路径

我们的模拟接收路径是人工的，但其周围的结构与真正的驱动程序所使用的完全相同。区别在于 mbuf 的来源和谁调用接收例程，而不在于接收例程之后做什么。本小节将带你了解典型的真实驱动程序接收路径，以便你下次在源代码树中打开以太网驱动程序时能认出它。

在真正的网卡上，数据包以 DMA 写入环形缓冲区中接收描述符的方式到达。硬件用驱动程序在初始化期间提供的预分配 mbuf 指针、长度和指示描述符是否准备好供驱动程序处理的状态字段来填充每个描述符。当描述符就绪时，硬件要么引发中断，要么设置一个驱动程序通过轮询注意到的位，或者两者兼有。

驱动程序的接收例程从最后处理的索引开始遍历环形缓冲区。对于每个就绪描述符，它读取长度和状态，修复相应的 mbuf 使其具有正确的 `m_len` 和 `m_pkthdr.len`，设置 `m->m_pkthdr.rcvif = ifp`，捕获 BPF，更新计数器，并调用 `if_input`。然后它分配一个替换 mbuf放回描述符，以便未来的数据包有地方着陆，并推进头指针。

此循环持续进行，直到环形缓冲区为空或驱动程序已处理完其每次调用的预算。在一次中断中处理太多数据包会饿死其他中断并损害其他设备的延迟；处理太少则浪费上下文切换。32 或 64 个数据包的预算是典型的。

接收循环之后，驱动程序更新硬件的尾指针以反映新补充的描述符。如果任何描述符仍然就绪，驱动程序要么重新启用中断，要么通过 taskqueue 调度自己再次运行。

发送的完成例程是镜像：它遍历发送环，寻找状态指示硬件已完成的描述符，释放相应的 mbuf，并更新驱动程序对可用发送槽位的感知。

你会在 `/usr/src/sys/dev/e1000/em_txrx.c` 及其他以太网硬件的等价文件中看到所有这些。环形缓冲区机制乍一看令人生畏，但其目的始终相同：从硬件 DMA 产生 mbuf 并通过 `if_input` 向上传递。我们的伪驱动程序从 `malloc` 产生 mbuf 并通过 `if_input` 向上传递。向上传递是相同的；只有 mbuf 的来源不同。

### 使用任务队列的延迟接收处理

高速率驱动程序中一个常见的改进是将实际接收处理从中断上下文延迟到 taskqueue。中断处理程序做最少量的工作（通常是向硬件确认中断并调度任务），taskqueue 工作线程做环形遍历和 `if_input` 调用。

为什么要延迟？因为 `if_input` 在协议栈内部可能做大量工作，包括 TCP 处理、套接字缓冲区填充和睡眠操作。在那么长时间内持有一个 CPU 在中断处理程序中对其他设备的中断延迟是有害的。将接收处理移到 taskqueue 让调度器将其与其他工作交错进行。

FreeBSD 的 taskqueue 子系统 `/usr/src/sys/kern/subr_taskqueue.c` 提供了驱动程序可以定位的每 CPU 工作线程。接收中断处理程序看起来像这样：

```c
static void
my_rx_intr(void *arg)
{
    struct mydrv_softc *sc = arg;

    /* Acknowledge the interrupt. */
    write_register(sc, RX_INT_STATUS, RX_READY);

    /* Defer the actual work. */
    taskqueue_enqueue(sc->rx_tq, &sc->rx_task);
}

static void
my_rx_task(void *arg, int pending __unused)
{
    struct mydrv_softc *sc = arg;

    mydrv_rx_drain(sc);       /* walk the ring and if_input each packet */
}
```

同样，`mynet` 是一个伪驱动程序，不需要这种复杂性。但看到这个模式意味着当你阅读 `if_em.c` 或 `if_ixl.c` 并看到 `taskqueue_enqueue` 时，你就知道什么正在被延迟以及为什么。

### 理解 `net_epoch`

FreeBSD 中的 `net_epoch` 框架是为网络子系统适配的基于 epoch 的回收机制的实现。其目的是让网络数据结构（路由表、ARP 表、接口列表等）的读者在不获取锁的情况下读取这些结构，同时确保写入者不会在读者可能仍在查看某个结构时释放它。

API 很简单。读者使用 `NET_EPOCH_ENTER(et)` 进入 epoch，使用 `NET_EPOCH_EXIT(et)` 退出，其中 `et` 是一个每次调用的跟踪器变量。在进入和退出之间，读者可以安全地解引用指向受保护数据结构的指针。想要释放受保护对象的写入者调用 `epoch_call` 来延迟释放，直到所有当前读者都退出。

对于驱动程序代码，其相关性在于：你从接收路径调用的协议栈例程，包括 `ether_input` 及其下游调用者，期望在调用者位于 net epoch 内部时被调用。因此，一些驱动程序将 `if_input` 调用包裹在 `NET_EPOCH_ENTER`/`NET_EPOCH_EXIT` 中。其他驱动程序（包括大多数基于 callout 的伪驱动程序）则依赖 `if_input` 本身在进入时如果不在 epoch 内则自动进入 epoch 这一事实。

对于 `mynet`，我们不显式进入 epoch。`if_input` 会为我们处理。如果你想格外小心，或者操作在一个已知未进入 epoch 的上下文中，你可以像这样包裹你的调用：

```c
struct epoch_tracker et;

NET_EPOCH_ENTER(et);
if_input(ifp, m);
NET_EPOCH_EXIT(et);
```

这是你在更新的驱动程序中会看到的惯用法。我们在正文中省略了它，因为它为我们的伪驱动程序增加了噪音而不改变行为。在可能从不寻常的上下文（例如工作队列或调度在非网络 CPU 上的定时器节拍）触发 `if_input` 的驱动程序中，你会想要显式包裹。

### 接收背压

一个接收数据包速度超过协议栈处理能力的驱动程序最终会耗尽其环形缓冲区。真正的驱动程序以两种方式之一处理：丢弃最旧的待处理数据包并更新 `IFCOUNTER_IQDROPS`，或者停止接收新描述符让硬件自己丢弃。

在软件伪驱动程序中没有会耗尽描述符的硬件，但你仍然应该考虑背压。如果你的模拟接收路径生成数据包的速度超过协议栈消费的速度，你最终会看到 mbuf 分配失败，或者系统开始在不排空的情况下将数据包排队到套接字缓冲区中。实际的防御是通过 callout 间隔限制自己的速率，并在长时间测试中观察 `vmstat -z | grep mbuf`。

对于 `mynet`，我们每秒生成一个合成 ARP。这比任何合理的背压阈值低几个数量级。但如果你将 `sc->rx_interval_hz` 增加到像 `hz / 1000`（每毫秒一个数据包）这样激进的值，你就是要求内核从单个驱动程序吸收每秒一千个 ARP，你会看到代价。

### 常见错误

最常见的接收路径错误如下。

**忘记 `M_PKTHDR` 纪律。** 如果你不用 `MGETHDR` 构造 mbuf，你就不会得到数据包头部，协议栈会断言失败或行为异常。始终对头部 mbuf 使用 `MGETHDR`（或 `m_gethdr`），对后续的 mbuf 使用 `MGET`（或 `m_get`）。

**忘记设置 `m_len` 和 `m_pkthdr.len`。** 协议栈使用 `m_pkthdr.len` 来决定数据包有多大，使用 `m_len` 来遍历链。如果这些值错误，解码会静默失败。

**在 `if_input` 期间持有驱动互斥锁。** 协议栈在 `if_input` 内部可能花费很长时间，并可能尝试获取其他锁。在向上调用之前释放驱动锁是避免死锁的纪律。

**在 `if_input` 之后触碰 `m`。** 协议栈可能已经释放或重新排队了 mbuf。将 `if_input` 视为一扇单向门。

**在没有链路层头部的情况下送入原始数据。** 因为我们使用了 `ether_ifattach`，`ether_input` 期望一个完整的以太网帧。如果你送入一个裸 IPv4 数据包，它会拒绝该帧并递增 `IFCOUNTER_IERRORS`。

### 第5节小结

我们现在有了通过驱动程序的双向流量。发送消费来自协议栈的 mbuf；接收为协议栈产生 mbuf。在此之间我们有 BPF 钩子、计数器更新和互斥锁纪律。我们还没有的是关于链路状态、媒体描述符和接口标志的完整叙述。那是第 6 节的内容。

## 第6节：媒体状态、标志和链路事件

到目前为止我们专注于数据包。但网络接口不仅仅是一个数据包搬运工。它是网络协议栈中有状态的参与者。它会上线也会下线。它有一个媒体类型，媒体可以改变。它的链路可以出现和消失。协议栈关心所有这些转换，用户态工具将它们呈现给管理员。在本节中，我们向 `mynet` 添加状态管理层。

### 接口标志：`IFF_` 和 `IFF_DRV_`

你已经见过 `IFF_UP` 和 `IFF_DRV_RUNNING`。还有更多，它们分为两个以不同方式工作的家族。

`IFF_` 标志定义在 `/usr/src/sys/net/if.h` 中，是用户可见的标志。它们是 `ifconfig` 读取和写入的。常见的包括：

* `IFF_UP` (`0x1`)：接口在管理上处于 up 状态。
* `IFF_BROADCAST` (`0x2`)：接口支持广播。
* `IFF_POINTOPOINT` (`0x10`)：接口是点对点的。
* `IFF_LOOPBACK` (`0x8`)：接口是环回接口。
* `IFF_SIMPLEX` (`0x800`)：接口无法听到自己的发送。
* `IFF_MULTICAST` (`0x8000`)：接口支持多播。
* `IFF_PROMISC` (`0x100`)：接口处于混杂模式。
* `IFF_ALLMULTI` (`0x200`)：接口正在接收所有多播。
* `IFF_DEBUG` (`0x4`)：用户请求了调试追踪。

这些标志主要通过用户态通过 `SIOCSIFFLAGS` 设置和清除。你的驱动程序应该对它们的变化做出反应：当 `IFF_UP` 从清除变为设置时，初始化；当从设置变为清除时，停止。

`IFF_DRV_` 标志也位于 `if.h` 中，是驱动程序私有的。它们存储在 `ifp->if_drv_flags` 中（不是 `if_flags`）。用户态无法看到或修改它们。最重要的两个是：

* `IFF_DRV_RUNNING` (`0x40`)：驱动程序已分配其每接口资源并可以移动流量。与较老的 `IFF_RUNNING` 别名相同。
* `IFF_DRV_OACTIVE` (`0x400`)：驱动程序的输出队列已满。协议栈不应再调用 `if_start` 或 `if_transmit`，直到此标志清除。

把 `IFF_UP` 想象成用户的意图，把 `IFF_DRV_RUNNING` 想象成驱动程序的准备状态。两者都需要为真才能传输流量。

### `SIOCSIFFLAGS` ioctl

当用户态运行 `ifconfig mynet0 up` 时，它在接口的标志字段中设置 `IFF_UP` 并发出 `SIOCSIFFLAGS`。协议栈通过我们的 `if_ioctl` 回调分派此 ioctl。我们的工作是注意到标志变化并做出反应。

以下是在网络驱动程序中处理 `SIOCSIFFLAGS` 的经典模式：

```c
case SIOCSIFFLAGS:
    MYNET_LOCK(sc);
    if (ifp->if_flags & IFF_UP) {
        if ((ifp->if_drv_flags & IFF_DRV_RUNNING) == 0) {
            MYNET_UNLOCK(sc);
            mynet_init(sc);
            MYNET_LOCK(sc);
        }
    } else {
        if (ifp->if_drv_flags & IFF_DRV_RUNNING) {
            MYNET_UNLOCK(sc);
            mynet_stop(sc);
            MYNET_LOCK(sc);
        }
    }
    MYNET_UNLOCK(sc);
    break;
```

让我们解析这段代码。

如果 `IFF_UP` 被设置，我们检查驱动程序是否已经在运行。如果不是，我们调用 `mynet_init` 进行初始化。如果驱动程序已经在运行，我们什么都不做：用户再次设置该标志是无操作。

如果 `IFF_UP` 未设置，我们检查是否曾在运行。如果是，我们调用 `mynet_stop` 来停止。如果不是，同样是空操作。

我们在调用 `mynet_init` 或 `mynet_stop` 之前释放锁，因为这些函数可能需要时间并可能在内部重新获取锁。「解锁、调用、重新加锁」的模式是 ioctl 处理程序的标准惯用法。

### 编写 `mynet_stop`

`mynet_init` 我们在第 4 节中已经写过。它的对应物 `mynet_stop` 类似但是反向的：

```c
static void
mynet_stop(struct mynet_softc *sc)
{
    struct ifnet *ifp = sc->ifp;

    MYNET_LOCK(sc);
    sc->running = false;
    ifp->if_drv_flags &= ~IFF_DRV_RUNNING;
    callout_stop(&sc->rx_callout);
    MYNET_UNLOCK(sc);

    if_link_state_change(ifp, LINK_STATE_DOWN);
}
```

我们清除运行标志，丢弃 `IFF_DRV_RUNNING` 位以便协议栈知道我们不再承载流量，停止接收 callout，并向协议栈宣布链路断开。这是初始化函数的对称伙伴。

### 链路状态：`if_link_state_change`

`if_link_state_change(ifp, state)` 是驱动程序报告链路转换的经典方式。值来自 `/usr/src/sys/net/if.h`：

* `LINK_STATE_UNKNOWN` (0)：驱动程序不知道链路状态。这是初始值。
* `LINK_STATE_DOWN` (1)：无载波，无法到达链路伙伴。
* `LINK_STATE_UP` (2)：链路已连接，可以到达链路伙伴，存在载波。

协议栈记录新状态，发送路由套接字通知，唤醒在该接口状态上睡眠的线程，并通过 `ifconfig` 的 `status:` 行通知用户态。真正的网卡驱动程序从链路状态变化中断处理程序中调用 `if_link_state_change`，通常是在 PHY 自动协商完成或丢失时。对于伪驱动程序，我们根据驱动程序自己的逻辑选择何时调用它。

值得认真考虑何时调用此函数。在 `mynet_init` 中，我们在设置 `IFF_DRV_RUNNING` 之后用 `LINK_STATE_UP` 调用它。在 `mynet_stop` 中，我们在清除 `IFF_DRV_RUNNING` 之后用 `LINK_STATE_DOWN` 调用它。如果你反转顺序，你会短暂地报告一个未运行的接口链路已连接，或者一个仍然声称正在运行的接口链路已断开。协议栈可以应对，但反转的症状会令人困惑。

### 媒体描述符

在链路状态之上是媒体。媒体是对正在使用的连接类型的描述：10BaseT、100BaseT、1000BaseT、10GBaseSR 等等。它与链路状态不同：连接即使在链路断开时也可以有已知的媒体类型。

FreeBSD 的媒体子系统位于 `/usr/src/sys/net/if_media.c` 及其头文件 `/usr/src/sys/net/if_media.h`。驱动程序通过一个小型 API 使用它：

* `ifmedia_init(ifm, dontcare_mask, change_fn, status_fn)`：初始化描述符。
* `ifmedia_add(ifm, word, data, aux)`: add a media entry.
* `ifmedia_set(ifm, word)`: choose the default entry.
* `ifmedia_ioctl(ifp, ifr, ifm, cmd)`：处理 `SIOCGIFMEDIA` 和 `SIOCSIFMEDIA`。

「字」是一个组合了媒体子类型和标志的位字段。对于以太网驱动程序，你将 `IFM_ETHER` 与 `IFM_1000_T`（1000BaseT）、`IFM_10G_T`（10GBaseT）或 `IFM_AUTO`（自动协商）等子类型组合。完整的子类型集合在 `if_media.h` 中列举。

我们在第 3 节中设置了描述符：

```c
ifmedia_init(&sc->media, 0, mynet_media_change, mynet_media_status);
ifmedia_add(&sc->media, IFM_ETHER | IFM_1000_T | IFM_FDX, 0, NULL);
ifmedia_add(&sc->media, IFM_ETHER | IFM_AUTO, 0, NULL);
ifmedia_set(&sc->media, IFM_ETHER | IFM_AUTO);
```

回调是协议栈在用户态查询或设置媒体时调用的函数：

```c
static int
mynet_media_change(struct ifnet *ifp __unused)
{
    /* In a real driver, program the PHY here. */
    return (0);
}

static void
mynet_media_status(struct ifnet *ifp, struct ifmediareq *imr)
{
    struct mynet_softc *sc = ifp->if_softc;

    imr->ifm_status = IFM_AVALID;
    if (sc->running)
        imr->ifm_status |= IFM_ACTIVE;
    imr->ifm_active = IFM_ETHER | IFM_1000_T | IFM_FDX;
}
```

`mynet_media_change` 是一个桩：伪驱动程序没有 PHY 可以重新编程。`mynet_media_status` 是 `ifconfig` 通过 `SIOCGIFMEDIA` 报告的内容：`ifm_status` 在我们运行时获得 `IFM_AVALID`（状态字段有效）和 `IFM_ACTIVE`（链路当前活动），`ifm_active` 告诉调用者我们实际使用的媒体。

ioctl 处理程序将媒体请求路由到 `ifmedia_ioctl`：

```c
case SIOCGIFMEDIA:
case SIOCSIFMEDIA:
    error = ifmedia_ioctl(ifp, ifr, &sc->media, cmd);
    break;
```

这正是 `/usr/src/sys/net/if_epair.c` 中 `epair_ioctl` 内部 `SIOCSIFMEDIA` / `SIOCGIFMEDIA` case 所使用的模式。

有了这些，`ifconfig mynet0` 将报告类似以下内容：

```text
mynet0: flags=8843<UP,BROADCAST,RUNNING,SIMPLEX,MULTICAST> metric 0 mtu 1500
        ether 02:a3:f1:22:bc:0d
        inet 192.0.2.1 netmask 0xffffff00 broadcast 192.0.2.255
        media: Ethernet autoselect (1000baseT <full-duplex>)
        status: active
```

### 处理 MTU 变更

`SIOCSIFMTU` 是用户运行 `ifconfig mynet0 mtu 1400` 时发出的 ioctl。一个行为良好的驱动程序检查请求的值是否在其支持的范围内，然后更新 `if_mtu`。我们的代码：

```c
case SIOCSIFMTU:
    if (ifr->ifr_mtu < 68 || ifr->ifr_mtu > 9216) {
        error = EINVAL;
        break;
    }
    ifp->if_mtu = ifr->ifr_mtu;
    break;
```

68 字节的下限匹配最小的 IPv4 有效载荷加头部。9216 的上限是一个慷慨的巨型帧边界。真正的驱动程序有更窄的范围，匹配其硬件能处理的大小。我们保持范围宽松，因为这是一个伪驱动程序。

### 处理多播组变更

`SIOCADDMULTI` 和 `SIOCDELMULTI` 表示用户已在接口上添加或删除了多播组。对于实现硬件多播过滤的真正网卡，驱动程序每次都会重新编程过滤器。我们的伪驱动程序没有过滤器，所以我们只是确认请求：

```c
case SIOCADDMULTI:
case SIOCDELMULTI:
    /* Nothing to program. */
    break;
```

这足以保证正确操作。协议栈将根据其内部组列表向接口传递多播流量，我们不需要做任何特殊的事情。

### 组装 ioctl 处理函数

有了以上所有内容，完整的 `mynet_ioctl` 如下所示：

```c
static int
mynet_ioctl(struct ifnet *ifp, u_long cmd, caddr_t data)
{
    struct mynet_softc *sc = ifp->if_softc;
    struct ifreq *ifr = (struct ifreq *)data;
    int error = 0;

    switch (cmd) {
    case SIOCSIFFLAGS:
        MYNET_LOCK(sc);
        if (ifp->if_flags & IFF_UP) {
            if ((ifp->if_drv_flags & IFF_DRV_RUNNING) == 0) {
                MYNET_UNLOCK(sc);
                mynet_init(sc);
                MYNET_LOCK(sc);
            }
        } else {
            if (ifp->if_drv_flags & IFF_DRV_RUNNING) {
                MYNET_UNLOCK(sc);
                mynet_stop(sc);
                MYNET_LOCK(sc);
            }
        }
        MYNET_UNLOCK(sc);
        break;

    case SIOCSIFMTU:
        if (ifr->ifr_mtu < 68 || ifr->ifr_mtu > 9216) {
            error = EINVAL;
            break;
        }
        ifp->if_mtu = ifr->ifr_mtu;
        break;

    case SIOCADDMULTI:
    case SIOCDELMULTI:
        break;

    case SIOCGIFMEDIA:
    case SIOCSIFMEDIA:
        error = ifmedia_ioctl(ifp, ifr, &sc->media, cmd);
        break;

    default:
        /* Let the common ethernet handler process this. */
        error = ether_ioctl(ifp, cmd, data);
        break;
    }

    return (error);
}
```

`default` case 委托给 `ether_ioctl`，它处理每个以太网驱动程序以相同方式处理的 ioctl（例如常见情况下的 `SIOCSIFADDR`、`SIOCSIFCAP`）。这为我们节省了编写十五行样板代码。`/usr/src/sys/net/if_epair.c` 在 `epair_ioctl` 中 switch 的 `default` 分支中做了同样的事情。

### 标志一致性规则

在编写驱动程序状态转换时，你应该记住一些一致性规则：

1. `IFF_DRV_RUNNING` 跟随 `IFF_UP`，而不是反过来。用户设置 `IFF_UP`，驱动程序响应设置或清除 `IFF_DRV_RUNNING`。
2. 链路状态变化应该在 `IFF_DRV_RUNNING` 转换之后发生，而不是之前。
3. 在设置 `IFF_DRV_RUNNING` 时启动的 callout 和 taskqueue 应该在清除它时停止或排空。
4. `if_input` 调用应该只在 `IFF_DRV_RUNNING` 被设置时发生。否则，你正在协议栈尚未完成启动的接口上传递数据包。
5. `if_transmit` 可能在 `IFF_UP` 被清除时仍然被调用，因为用户态和协议栈之间存在竞争。你的发送路径应该检查标志并在任一标志被清除时优雅地丢弃。

这些规则在每个编写良好的驱动程序代码中都是隐含的。将它们明确化在你初学时是有用的。

### 接口能力详解

我们在第 3 节中设置 `IFCAP_VLAN_MTU` 时简要提到了能力。能力值得在此更全面地讨论，因为它们是驱动程序告诉协议栈自己能执行哪些卸载的方式，而且它们对于快速驱动程序保持快速越来越核心。

`if_capabilities` 字段定义在 `/usr/src/sys/net/if.h` 中，是硬件能执行的能力的位掩码。`if_capenable` 字段是当前已启用能力的位掩码。它们是分开的，因为用户态可以在运行时通过 `ifconfig mynet0 -rxcsum` 或 `ifconfig mynet0 +tso` 切换单个卸载，驱动程序必须遵守该选择。

常见的能力包括：

* `IFCAP_RXCSUM` 和 `IFCAP_RXCSUM_IPV6`：驱动程序将在硬件中验证 IPv4 和 IPv6 校验和，并在 mbuf 的 `m_pkthdr.csum_flags` 中用 `CSUM_DATA_VALID` 标记正确校验的数据包。
* `IFCAP_TXCSUM` 和 `IFCAP_TXCSUM_IPV6`：驱动程序将在硬件中为 `m_pkthdr.csum_flags` 请求的出站数据包计算 TCP、UDP 和 IP 校验和。
* `IFCAP_TSO4` 和 `IFCAP_TSO6`：驱动程序接受大型 TCP 段，硬件将它们在线路上拆分为 MTU 大小的帧。这显著减少了 TCP 密集型工作负载的 CPU 负载。
* `IFCAP_LRO`：驱动程序将多个接收到的 TCP 段聚合为单个大型 mbuf 后再向上传递。接收端 TSO 的对称操作。
* `IFCAP_VLAN_HWTAGGING`：驱动程序将在硬件而不是软件中添加和剥离 802.1Q VLAN 标签。这为每个 VLAN 帧节省了一次 mbuf 复制。
* `IFCAP_VLAN_MTU`：驱动程序可以承载总长度因额外 4 字节标签而略微超过标准以太网 MTU 的 VLAN 标记帧。
* `IFCAP_JUMBO_MTU`：驱动程序支持有效载荷大于 1500 字节的帧。
* `IFCAP_WOL_MAGIC`：使用魔术包的局域网唤醒。
* `IFCAP_POLLING`：经典设备轮询，现在很少使用。
* `IFCAP_NETMAP`：驱动程序支持 `netmap(4)` 内核旁路数据包 I/O。
* `IFCAP_TOE`：TCP 卸载引擎。罕见，但存在于一些高端网卡上。

声明一个能力就是向协议栈做出你会兑现它的承诺。如果你声称 `IFCAP_TXCSUM` 但实际上并没有为出站帧计算 TCP 校验和，内核会愉快地交给你带有未计算校验和的数据包并期望你完成工作。接收方会得到损坏的帧并丢弃它们。症状是静默的数据丢失，这调试起来很痛苦。

对于 `mynet`，我们诚实地只声明我们能兑现的内容。`IFCAP_VLAN_MTU` 是我们声称的唯一能力，我们通过在发送路径中接受最大到 `ifp->if_mtu + sizeof(struct ether_vlan_header)` 的帧来兑现它。

一个行为良好的驱动程序还在其 ioctl 处理程序中处理 `SIOCSIFCAP`，以便用户可以切换特定的卸载：

```c
case SIOCSIFCAP:
    mask = ifr->ifr_reqcap ^ ifp->if_capenable;
    if (mask & IFCAP_VLAN_MTU)
        ifp->if_capenable ^= IFCAP_VLAN_MTU;
    /* Reprogram hardware if needed. */
    break;
```

对于伪驱动程序没有硬件可以重新编程，但用户可见的切换仍然有效，因为 ioctl 更新了 `if_capenable`，每个后续的发送决策都会读取该字段。

### `ether_ioctl` 通用处理函数

我们之前看到 `mynet_ioctl` 将未知 ioctl 委托给 `ether_ioctl`。值得看看那个函数做了什么，因为它解释了为什么大多数驱动程序只需显式处理少量 ioctl。

`ether_ioctl` 定义在 `/usr/src/sys/net/if_ethersubr.c` 中，是每个以太网接口以相同方式处理的 ioctl 的通用处理程序。其职责包括：

* `SIOCSIFADDR`：用户正在为接口分配 IP 地址。`ether_ioctl` 处理 ARP 探测和地址注册。如果接口已关闭且应该启动，它会调用驱动程序的 `if_init` 回调。
* `SIOCGIFADDR`：返回接口的链路层地址。
* `SIOCSIFMTU`：如果驱动程序不提供自己的处理程序，`ether_ioctl` 通过更新 `if_mtu` 执行通用 MTU 更改。
* `SIOCADDMULTI` 和 `SIOCDELMULTI`：更新驱动程序的多播过滤器（如果存在）。
* 各种与能力相关的 ioctl。

因为默认处理程序处理了这么多，驱动程序通常只需要处理需要驱动程序特定逻辑的 ioctl：用于上/下转换的 `SIOCSIFFLAGS`、用于重新编程媒体的 `SIOCSIFMEDIA`，以及用于切换能力的 `SIOCSIFCAP`。其余的都落到 `ether_ioctl`。

这种委托模型是让编写小型以太网驱动程序变得愉快的原因之一：你编写特定于你驱动程序的代码，通用代码处理其余的事情。

### 硬件多播过滤

对于真正的网卡，多播过滤通常在硬件中完成。驱动程序将一组 MAC 地址编程到硬件过滤表中，网卡只传递目标地址与表中地址匹配的帧。当用户运行 `ifconfig mynet0 addm 01:00:5e:00:00:01` 加入多播组时，协议栈发出 `SIOCADDMULTI`，驱动程序必须更新过滤表。

真正的驱动程序中的典型模式是：

```c
case SIOCADDMULTI:
case SIOCDELMULTI:
    if (ifp->if_drv_flags & IFF_DRV_RUNNING) {
        MYDRV_LOCK(sc);
        mydrv_setup_multicast(sc);
        MYDRV_UNLOCK(sc);
    }
    break;
```

`mydrv_setup_multicast` 遍历接口的多播列表（通过 `if_maddr_rlock` 及相关函数访问）并将每个地址编程到硬件过滤器中。这段代码枯燥但重要；搞错它意味着像 mDNS（Bonjour、Avahi）、基于 IGMP 的路由和 IPv6 邻居发现等多播应用会静默出错。

对于 `mynet` 我们没有硬件过滤器，所以我们只是接受 `SIOCADDMULTI` 和 `SIOCDELMULTI` 而不做任何事情。协议栈仍然为我们跟踪多播组列表，我们的接收路径不过滤，所以一切正常。

如果你将来编写带有硬件多播过滤的驱动程序，阅读 `/usr/src/sys/dev/e1000/if_em.c` 中的 `em_multi_set` 函数可以找到该模式的清晰示例。

### 第6节小结

我们已经涵盖了网络驱动的状态半部分。标志、链路状态、媒体描述符以及将它们联系在一起的 ioctl。结合第 4 节和第 5 节的发送和接收路径，我们现在有了一个在 `ifnet` 边界上与简单的真实以太网驱动程序无法区分的驱动程序。

在我们可以称驱动程序完成之前，我们需要确保能够使用 FreeBSD 生态系统提供的工具对其进行彻底测试。那是第 7 节的内容。

## 第7节：使用标准网络工具测试驱动程序

一个驱动程序的好坏取决于你对它能正常工作的信心。信心不是来自盯着代码看。它来自运行驱动程序、从外部与之交互并观察结果。本节将带你了解标准的 FreeBSD 网络工具，并展示如何使用每个工具来测试 `mynet` 的特定方面。

### 加载、创建、配置

从干净状态开始。如果模块已加载，卸载它，然后加载新构建并创建第一个接口：

```console
# kldstat | grep mynet
# kldload ./mynet.ko
# ifconfig mynet create
mynet0
```

`ifconfig mynet0` 应该显示带有 MAC 地址、无 IP、除默认集之外无标志、以及显示"autoselect"的媒体描述符的接口。分配地址并启动它：

```console
# ifconfig mynet0 inet 192.0.2.1/24 up
# ifconfig mynet0
mynet0: flags=8843<UP,BROADCAST,RUNNING,SIMPLEX,MULTICAST> metric 0 mtu 1500
        ether 02:a3:f1:22:bc:0d
        inet 192.0.2.1 netmask 0xffffff00 broadcast 192.0.2.255
        media: Ethernet autoselect (1000baseT <full-duplex>)
        status: active
        groups: mynet
```

`UP` 和 `RUNNING` 标志确认用户的意图和驱动程序的准备状态都已就位。`status: active` 行来自我们的媒体回调。媒体描述包括 `1000baseT`，因为那是 `mynet_media_status` 返回的。

### 使用 `netstat` 检查

`netstat -in -I mynet0` 显示每接口计数器。最初，一切都是零；等待几秒让接收模拟启动，计数器应该攀升：

```console
# netstat -in -I mynet0
Name    Mtu Network      Address                  Ipkts Ierrs ...  Opkts Oerrs
mynet0 1500 <Link#12>   02:a3:f1:22:bc:0d           3     0        0     0
mynet0    - 192.0.2.0/24 192.0.2.1                   0     -        0     -
```

第一行的 `Ipkts` 统计我们的接收定时器产生的合成 ARP 请求数量。它应该大约每秒增加一。如果不是，说明 `rx_interval_hz` 设置错误，或者 callout 在 `mynet_init` 中没有被启动，或者 `running` 为 false。

### 使用 `tcpdump` 捕获

`tcpdump -i mynet0 -n` 捕获我们接口上的所有流量。你应该看到每秒生成的合成 ARP 请求，以及你自己的 `ping` 尝试导致的任何流量：

```console
# tcpdump -i mynet0 -n
tcpdump: verbose output suppressed, use -v or -vv for full protocol decode
listening on mynet0, link-type EN10MB (Ethernet), capture size 262144 bytes
14:30:12.000 02:a3:f1:22:bc:0d > ff:ff:ff:ff:ff:ff, ethertype ARP, Request who-has 192.0.2.99 tell 0.0.0.0, length 28
14:30:13.000 02:a3:f1:22:bc:0d > ff:ff:ff:ff:ff:ff, ethertype ARP, Request who-has 192.0.2.99 tell 0.0.0.0, length 28
...
```

"link-type EN10MB (Ethernet)" 确认 BPF 将我们视为以太网接口，这是 `ether_ifattach` 为我们调用 `bpfattach(ifp, DLT_EN10MB, ETHER_HDR_LEN)` 的结果。切换到 `-v` 或 `-vv` 可以看到更完整的协议解码。

### 使用 `ping` 生成流量

通过 ping 我们分配的子网中的某个 IP 来触发出站流量：

```console
# ping -c 3 192.0.2.99
PING 192.0.2.99 (192.0.2.99): 56 data bytes
--- 192.0.2.99 ping statistics ---
3 packets transmitted, 0 packets received, 100.0% packet loss
```

所有三个 ping 都丢失了，因为我们的伪驱动程序模拟的线缆另一端什么也没有。但发送计数器在移动：

```console
# netstat -in -I mynet0
Name    Mtu Network     Address                Ipkts Ierrs ... Opkts Oerrs
mynet0 1500 <Link#12>   02:a3:f1:22:bc:0d         30     0       6     0
```

6 个已发送的数据包是三个 ping 加上协议栈为解析 `192.0.2.99` 而发出的三个 ARP 广播请求。你可以用 `tcpdump` 验证这一点。

### `arp -an`

`arp -an` 显示系统的 ARP 缓存。`192.0.2.99` 的条目在协议栈等待永远不会到来的 ARP 回复时应显示为不完整。大约一分钟后它们会过期。

### `sysctl net.link` and `sysctl net.inet`

网络子系统暴露了丰富的每接口和每协议 sysctl。`sysctl net.link.ether` 控制以太网层行为。`sysctl net.inet.ip` 控制 IP 层行为。虽然这些都不是 `mynet` 特有的，但了解它们是好的。诊断伪驱动程序行为时常用的一个是 `sysctl net.link.ether.inet.log_arp_wrong_iface=0`，它可以静默关于 ARP 流量出现在意外接口上的日志消息。

### 使用 `ifstated` 或 `devd` 监控链路事件

FreeBSD 通过路由套接字传播链路状态变化。你可以用 `route monitor` 实时观察：

```console
# route monitor
```

当你运行 `ifconfig mynet0 down` 然后运行 `ifconfig mynet0 up` 时，`route monitor` 会打印与我们通过 `if_link_state_change` 宣布的链路状态变化相对应的 `RTM_IFINFO` 消息。那是 `devd` 用于其 `notify` 事件的相同机制，也是脚本可以响应链路翻转的方式。

### 测试 MTU 变更

```console
# ifconfig mynet0 mtu 9000
# ifconfig mynet0
mynet0: ... mtu 9000
```

将 MTU 更改为合理的值并观察 `ifconfig` 反映变化。尝试一个超出范围的值并验证内核拒绝它：

```console
# ifconfig mynet0 mtu 10
ifconfig: ioctl SIOCSIFMTU (set mtu): Invalid argument
```

该错误来自我们的 `SIOCSIFMTU` 处理程序返回 `EINVAL`。

### 测试媒体命令

```console
# ifconfig mynet0 media 10baseT/UTP
ifconfig: requested media type not found
```

这会失败，因为我们没有注册 `IFM_ETHER | IFM_10_T` 作为可接受的媒体类型。在 `mynet_create_unit` 中注册它并重新构建即可使命令成功。

```console
# ifconfig mynet0 media 1000baseT
# ifconfig mynet0 | grep media
        media: Ethernet 1000baseT <full-duplex>
```

### 与 `if_disc` 对比

同时加载 `if_disc` 并比较：

```console
# kldload if_disc
# ifconfig disc create
disc0
# ifconfig disc0 inet 192.0.2.50/24 up
```

`disc0` 是一个更简单的伪驱动程序。它在 `discoutput` 函数中通过丢弃来忽略每个出站数据包。它没有接收路径。在 ping `192.0.2.50` 时运行 `tcpdump -i disc0` 显示出站 ICMP 帧但没有入站 ARP 活动。与我们仍然每秒显示合成 ARP 帧的 `mynet0` 对比。

这种对比是有用的，因为它展示了从"丢弃一切"到"模拟完整以太网接口"的步骤有多小。我们添加了 MAC 地址、媒体描述符、callout 和数据包构建器。其余的一切，包括接口注册、BPF 钩子、标志，都已经在模式中了。

### 使用 `iperf3` 进行压力测试

`iperf3` 可以使真正的以太网链路饱和。在我们的伪驱动程序上它不会产生有意义的吞吐量数字（数据包无处可去），但它会非常用力地测试 `if_transmit`：

```console
# iperf3 -c 192.0.2.99 -t 10
Connecting to host 192.0.2.99, port 5201
iperf3: error - unable to connect to server: Connection refused
```

连接失败因为没有服务器，但 `netstat -in -I mynet0` 会显示 `Opkts` 随着 `iperf3` 导致的 TCP 重传和 ARP 请求快速攀升。在另一个终端观察 `vmstat 1` 并确保系统负载保持合理。如果你看到大量时间花在驱动程序中，你可能有一个值得调查的锁热点。

### 脚本化测试运行

你可以将上述命令包装成一个以已知顺序测试驱动程序的小型 shell 脚本。以下是一个最小示例：

```sh
#!/bin/sh

set -e

echo "== load =="
kldload ./mynet.ko

echo "== create =="
ifconfig mynet create

echo "== configure =="
ifconfig mynet0 inet 192.0.2.1/24 up

echo "== traffic =="
(tcpdump -i mynet0 -nn -c 5 > /tmp/mynet-tcpdump.txt 2>&1) &
sleep 3
ping -c 2 192.0.2.99 || true
wait
cat /tmp/mynet-tcpdump.txt

echo "== counters =="
netstat -in -I mynet0

echo "== teardown =="
ifconfig mynet0 destroy
kldunload mynet
```

将它保存在 `examples/part-06/ch28-network-driver/lab05-bpf/run.sh` 下，标记为可执行，并以 root 身份运行。它在十秒内让驱动程序走过整个生命周期。当以后出现问题时，像这样脚本化的基线对于发现回归是无价的。

### 注意事项

测试时请注意：

* 加载和卸载时的 `dmesg` 输出，查看意外警告。
* 操作前后的 `netstat -in -I mynet0`，确认计数器向预期方向移动。
* 卸载后的 `kldstat`，确认模块已消失。
* `destroy` 后的 `ifconfig -a`，确认没有孤立接口留下。
* `vmstat -m | grep mynet`，确认卸载时内存被释放。
* 负载测试运行前后的 `vmstat -z | grep mbuf`，确认 mbuf 计数稳定。

一个在冷加载时正确的驱动程序仍然可能在卸载时泄漏，或在负载下泄漏，或在罕见竞争条件下导致内核崩溃。上面列出的工具是抵御所有这些类别 bug 的第一道防线。

### 使用 DTrace 进行深度观测

FreeBSD 的 DTrace 实现是驱动程序可观测性的强大工具，一旦你知道一些模式，你就会经常使用它。基本思想是内核中的每个函数入口和出口都是探测点，每个探测点都可以从用户态检测而无需修改代码。

要统计我们的发送函数被调用的频率：

```console
# dtrace -n 'fbt::mynet_transmit:entry { @c = count(); }'
```

在一个终端中运行它，在另一个终端中生成流量，你会看到计数攀升。要观察每次调用的数据包长度：

```console
# dtrace -n 'fbt::mynet_transmit:entry { printf("len=%d", args[1]->m_pkthdr.len); }'
```

DTrace 脚本可以更加精细。以下是一个按源 IP 分组统计已发送数据包的脚本，如果接口承载 IPv4 流量：

```console
# dtrace -n 'fbt::mynet_transmit:entry /args[1]->m_pkthdr.len > 34/ {
    this->ip = (struct ip *)(mtod(args[1], struct ether_header *) + 1);
    @src[this->ip->ip_src.s_addr] = count();
}'
```

这种可观测性很难手动添加到驱动程序中，但 DTrace 免费给了你。使用它。当你无法判断一个数据包为什么流动或不流动时，你自己函数上的 DTrace 探测几乎总能揭示答案。

网络驱动程序工作中一些额外有用的单行命令：

```console
# dtrace -n 'fbt::if_input:entry { @ifs[stringof(args[0]->if_xname)] = count(); }'
```

这统计整个系统中每次 `if_input` 的调用，按接口名称分组。这是验证你的接收路径是否到达协议栈的快速方法。

```console
# dtrace -n 'fbt::if_inc_counter:entry /args[1] == 1/ {
    @[stringof(args[0]->if_xname)] = count();
}'
```

这统计 `IFCOUNTER_IPACKETS`（在枚举中值为 1）的 `if_inc_counter` 调用，按接口名称分组。与 `netstat -in` 相比，它让你实时看到增量。

不要害怕 DTrace。它一开始看起来令人生畏，因为类似脚本的语法，但使用 DTrace 调试驱动程序通常只需几分钟，而等效的 printf 调试需要数小时。你投资学习 DTrace 惯用法每一分钟都会得到多次回报。

### 驱动程序的内核调试器技巧

当网络驱动程序崩溃或挂起时，内核调试器（`ddb` 或 `kgdb`）是最后的手段。一些驱动程序特定的技巧：

* 崩溃后，`show mbuf`（或 `show pcpu`、`show alltrace`、`show lockchain`，取决于你在调查什么）遍历 mbuf 分配或每 CPU 数据或阻塞线程链。知道调用哪个是实践的问题。
* `show ifnet <pointer>` 打印给定地址的 `ifnet` 结构内容。当崩溃消息说"ifp = 0xffff..."时很有用。softc 的等价物取决于驱动程序。
* `bt` 打印栈跟踪。大多数时候你想要 `bt <tid>`，其中 `<tid>` 是感兴趣的线程 ID。
* `continue` 恢复执行，但在真正崩溃后通常不安全。收集信息然后 `reboot`。

对于非崩溃调试，`kgdb /boot/kernel/kernel /var/crash/vmcore.0` 让你对崩溃转储进行事后分析。在有崩溃转储分区的实验虚拟机上开发驱动程序是一个舒适的工作流程：崩溃、重启、从容地查看转储。

### 使用 `systat -if` 查看实时计数器

`systat -if 1` 打开一个每秒刷新的 ncurses 视图，显示每接口的计数器速率。它是 `netstat -in` 的有用补充，因为你可以实时观察流量升降，而不需要阅读终端日志。

```text
                    /0   /1   /2   /3   /4   /5   /6   /7   /8   /9   /10
     Load Average   ||
          Interface          Traffic               Peak                Total
             mynet0     in      0.000 KB/s      0.041 KB/s         0.123 KB
                       out      0.000 KB/s      0.047 KB/s         0.167 KB
```

此视图中的速率由 `systat` 从我们在 `if_transmit` 和接收路径中递增的计数器计算得出。如果速率与你期望的不匹配，第一个怀疑应该是某个计数器被更新了两次，或者它在 `m_freem` 之后被更新，或者它使用了 `IFCOUNTER_OPACKETS` 而应该使用 `IFCOUNTER_IPACKETS`。`systat -if` 使这些错误非常明显。

### 第7节小结

你现在有了一个经过测试的驱动程序。它加载、配置、双向传输流量、向用户态报告状态、与 BPF 合作，并响应链路事件。剩下的是生命周期的最后阶段：干净分离、模块卸载和一些重构建议。那是第 8 节的内容。

## 第8节：清理、分离和网络驱动程序的重构

每个驱动程序都有开始和结束。开始是我们在本章中构建的模式：分配、配置、注册、运行。结束是对称的拆卸：停止、注销、释放。一个在卸载时泄漏单个字节的驱动程序不是一个正确的驱动程序，无论它在活跃生命期间表现多好。在本节中，我们最终确定清理路径，回顾卸载纪律，并提供重构建议，使代码随着增长保持可维护。

### 完整的拆卸序列

将我们说过的所有内容放在一起，一个 `mynet` 接口的完整拆卸如下所示：

```c
static void
mynet_destroy(struct mynet_softc *sc)
{
    struct ifnet *ifp = sc->ifp;

    MYNET_LOCK(sc);
    sc->running = false;
    ifp->if_drv_flags &= ~IFF_DRV_RUNNING;
    MYNET_UNLOCK(sc);

    callout_drain(&sc->rx_callout);

    ether_ifdetach(ifp);
    if_free(ifp);

    ifmedia_removeall(&sc->media);
    mtx_destroy(&sc->mtx);
    free(sc, M_MYNET);
}
```

顺序很重要。让我们逐步讲解。

**步骤 1：标记不再运行。** 在互斥锁下设置 `sc->running = false` 并清除 `IFF_DRV_RUNNING` 意味着任何并发的 callout 调用都能看到更新并干净退出。仅此一步不足以停止正在运行的 callout，但它确实阻止了新工作被调度。

**步骤 2：排空 callout。** `callout_drain(&sc->rx_callout)` 阻塞调用线程，直到任何正在进行的 callout 调用完成，且不会再有进一步的调用发生。`callout_drain` 返回后，可以安全地访问 softc 而不必担心 callout 会再次触发。这是与 callout 同步的最干净方式，也是我们推荐在每个使用 callout 的驱动程序中使用的模式。

**步骤 3：分离接口。** `ether_ifdetach(ifp)` 撤销 `ether_ifattach` 所做的工作。它调用 `if_detach`，将接口从全局列表中移除，撤销其地址，并使任何缓存的指针失效。它还调用 `bpfdetach` 以便 BPF 释放其句柄。此调用后，接口不再对用户态或协议栈可见。

**步骤 4：释放 ifnet。** `if_free(ifp)` 释放内存。此调用后，`ifp` 指针无效，不得使用。

**步骤 5：清理驱动程序私有状态。** `ifmedia_removeall` 释放我们添加的媒体条目。`mtx_destroy` 拆除互斥锁。`free` 释放 softc。

以任何方式弄错这个序列都会导致微妙的 bug。在排空 callout 之前释放 softc 会导致 callout 触发时的释放后使用。在分离之前释放 ifnet 会导致协议栈各处的级联失败。在排空 callout（它在入口处重新获取互斥锁）之前销毁互斥锁会导致经典的「销毁已加锁互斥锁」崩溃。「停止、分离、释放」的纪律是保持拆卸干净的关键。

### 克隆器销毁路径

回想一下我们使用 `if_clone_simple` 注册了克隆器，传递了 `mynet_clone_create` 和 `mynet_clone_destroy`。销毁函数在用户态运行 `ifconfig mynet0 destroy` 或模块卸载并克隆器分离时由克隆器框架调用。我们的实现是一个简单的包装器：

```c
static void
mynet_clone_destroy(struct ifnet *ifp)
{
    mynet_destroy((struct mynet_softc *)ifp->if_softc);
}
```

克隆器框架遍历它创建的接口列表并为每个接口调用销毁函数。它自己不做排空或解锁。那是驱动程序的责任，`mynet_destroy` 正确地完成了它。

### 模块卸载

当调用 `kldunload mynet` 时，内核使用 `MOD_UNLOAD` 调用模块事件处理函数。我们的模块处理函数没有做任何有趣的事情；繁重的工作由我们注册的 VNET sysuninit 完成：

```c
static void
vnet_mynet_uninit(const void *unused __unused)
{
    if_clone_detach(V_mynet_cloner);
}
```

`if_clone_detach` 做两件事。首先，它通过为每个接口调用我们的 `mynet_clone_destroy` 来销毁通过克隆器创建的每个接口。其次，它注销克隆器本身，以便不能再创建新接口。此调用后，我们驱动程序的每个痕迹都从内核状态中消失了。

试试看：

```console
# ifconfig mynet create
mynet0
# ifconfig mynet create
mynet1
# kldunload mynet
# ifconfig -a
```

`mynet0` 和 `mynet1` 应该消失了。控制台没有消息，没有残留计数器，没有剩余的克隆器。这就是一次成功的卸载。

### 内存统计

`vmstat -m | grep mynet` 显示我们 `M_MYNET` 标签的当前分配：

```console
# vmstat -m | grep mynet
         Type InUse MemUse Requests  Size(s)
        mynet     0     0K        7  2048
```

卸载后的 `InUse 0` 和 `MemUse 0K` 确认我们没有泄漏。`Requests` 统计生命周期分配。如果你卸载并重新加载多次，`Requests` 会攀升但 `InUse` 每次都回到零。如果 `InUse` 在卸载后曾经保持非零，你就有泄漏。

### 处理卡住的定时回调

开发过程中偶尔你会修改驱动程序，导致 callout 无法干净地排空。症状是 `kldunload` 挂起，或系统因关于已加锁互斥锁的消息而崩溃。根本原因几乎总是以下之一：

* callout 处理程序重新获取了互斥锁但没有重新调度自身，而 `callout_drain` 在最后一次计划触发完成之前被调用。
* callout 处理程序卡在等待另一个线程持有的锁上。
* callout 本身在排空之前从未被正确停止。

第一道防线是 `callout_init_mtx` 配合 softc 互斥锁：这设置了一个自动获取模式，使排空在构造上就是正确的。第二道防线是一致地使用 `callout_stop` 或 `callout_drain`，避免在同一个 callout 上混合使用两者。

如果卸载挂起，使用 `ps -auxw` 找到有问题的线程，然后对运行中的内核使用 `kgdb`（通过 `/dev/mem` 和 `bin/kgdb /boot/kernel/kernel`）查看它卡在什么上。卡住的帧几乎总是在 callout 代码中，修复几乎总是先排空再销毁互斥锁。

### VNET 注意事项

FreeBSD 的网络协议栈支持 VNET，即与 jail 或 VNET 实例关联的虚拟网络协议栈。如果驱动程序想要允许每个 VNET 创建接口，它可以是 VNET 感知的；如果每系统一组接口就够了，它也可以是非 VNET 感知的。

我们在克隆器注册中使用了 `VNET_DEFINE_STATIC` 和 `VNET_SYSINIT`/`VNET_SYSUNINIT`。这个选择使我们的驱动程序隐式地成为 VNET 感知的：每个 VNET 获得自己的克隆器，`mynet` 接口可以在任何 VNET 中创建。对于一个小型伪驱动程序，这不需要任何成本并为我们带来了灵活性。

VNET 的更深层方面，包括使用 `if_vmove` 在 VNET 之间移动接口以及处理 VNET 拆卸，超出了本章的范围，将在本书后面的第 30 章中涵盖。现在只需知道我们的驱动程序遵循使其与 VNET 兼容的约定即可。

### 重构建议

我们构建的驱动程序是一个约 500 行 C 代码的单文件。对于教学示例来说很舒适。在具有更多功能的生产驱动程序中，文件会增长，你会想要拆分它。以下是几乎每个驱动程序最终都会做的拆分。

**将 ifnet 胶水与数据路径分离。** ifnet 注册、克隆器逻辑和 ioctl 处理随时间趋于稳定。数据路径，即发送和接收，随着硬件功能变化而演进。将它们拆分为 `mynet_if.c` 和 `mynet_data.c` 可以使大多数文件小而专注。

**隔离后端。** 在真正的网卡驱动程序中，后端是硬件特定的代码：寄存器访问、DMA、MSI-X、环形缓冲区。在伪驱动程序中，后端是模拟。无论哪种方式，将后端放在 `mynet_backend.c` 中并使用干净的接口，可以在不触碰 ifnet 代码的情况下替换后端。

**分离 sysctl 和调试。** 随着驱动程序增长，你会添加用于诊断控制的 sysctl、用于调试的计数器，可能还有 DTrace SDT 探测。这些倾向于以混乱的方式积累。将它们保留在 `mynet_sysctl.c` 中可以保持主文件可读。

**保持头文件公开。** 一个声明 softc 和跨文件原型的 `mynet_var.h` 或 `mynet.h` 头文件是保持拆分编译的胶水。将该头文件视为一个迷你公共 API。

**版本化驱动程序。** `MODULE_VERSION(mynet, 1)` 是最低限度。当你添加重要功能时，递增版本。依赖你模块的下游消费者可以要求最低版本，内核用户可以通过 `kldstat -v` 知道他们加载的是哪个版本的驱动程序。

### 功能标志和能力

以太网驱动程序通过 `if_capabilities` 和 `if_capenable` 声明能力。我们设置了 `IFCAP_VLAN_MTU`。真正的驱动程序可能声明的其他能力包括：

* `IFCAP_HWCSUM`：硬件校验和卸载。
* `IFCAP_TSO4`、`IFCAP_TSO6`：IPv4 和 IPv6 的 TCP 分段卸载。
* `IFCAP_LRO`：大接收卸载。
* `IFCAP_VLAN_HWTAGGING`：硬件 VLAN 标记。
* `IFCAP_RXCSUM`、`IFCAP_TXCSUM`：接收和发送校验和卸载。
* `IFCAP_JUMBO_MTU`：巨型帧支持。
* `IFCAP_LINKSTATE`：硬件链路状态事件。
* `IFCAP_NETMAP`：`netmap(4)` 支持，用于高速包 I/O。

对于伪驱动程序，大多数这些都不相关。虚假声明它们会导致问题，因为协议栈随后会尝试使用它们并期望它们工作。保持能力集诚实：只声明你的驱动程序实际支持的功能。

### 编写运行脚本

与驱动程序一起产出的最有用的工件之一是一个小型 shell 脚本，用于测试其整个生命周期。我们在第 7 节中展示的骨架已经是该脚本的 80%。扩展它：

* 每次操作后的一致性检查（`ifconfig -a | grep mynet0` 或 `netstat -in -I mynet0 | ...`）。
* 可选地将每步记录到文件以供事后检查。
* 末尾的清理块，确保即使前面的步骤失败，系统也处于已知状态。

一个好的运行脚本是实现无回归开发的最有价值的工具。我们鼓励你在挑战中扩展驱动程序时维护一个。

### 整理文件

最后，关于代码风格。真正的 FreeBSD 驱动程序遵循 KNF（内核标准格式），即 `style(9)` 中记录的编码风格。总结：用制表符缩进，函数定义的大括号与函数在同一行，结构体和枚举的大括号在下一行，尽可能使用 80 列行宽，函数调用的左括号前不加空格等等。如果你一致地遵循 KNF，你的驱动程序将更容易合并到上游（也更容易在一年后阅读）。

### 处理部分初始化失败

我们专注于了顺利的路径。如果 `mynet_create_unit` 中途失败会怎样？假设 `if_alloc` 成功，`mtx_init` 运行了，`ifmedia_init` 设置了媒体，然后某个辅助缓冲区的 `malloc` 返回 NULL。我们需要干净地回滚，因为用户刚刚让 `ifconfig mynet create` 失败了，我们不能留下任何痕迹。

回滚的惯用法是函数末尾附近的一块标签，每个标签撤销初始化的一个步骤：

```c
static int
mynet_create_unit(int unit)
{
    struct mynet_softc *sc;
    struct ifnet *ifp;
    int error = 0;

    sc = malloc(sizeof(*sc), M_MYNET, M_WAITOK | M_ZERO);
    ifp = if_alloc(IFT_ETHER);
    if (ifp == NULL) {
        error = ENOSPC;
        goto fail_alloc;
    }

    sc->ifp = ifp;
    mtx_init(&sc->mtx, "mynet", NULL, MTX_DEF);
    /* ... other setup ... */

    ether_ifattach(ifp, sc->hwaddr);
    return (0);

fail_alloc:
    free(sc, M_MYNET);
    return (error);
}
```

这种模式在内核代码中很常见，使回滚变得枯燥。每个标签负责紧接在其上面的步骤。总体形状是「如果步骤 N 中的某个东西失败，跳转到标签 N-1 并从那里展开」。

对于我们的驱动程序，创建早期唯一现实的失败点是 `if_alloc`。如果它成功了，其余的设置（互斥锁初始化、媒体初始化、ether_ifattach）要么是绝对可靠的，要么是足够幂等的，不需要回滚。但回滚的形状很重要，因为更复杂的驱动程序会有更多失败点，同样的模式可以干净地扩展。

### 与执行中的回调同步

除了 callout 之外，当我们拆卸接口时其他异步代码可能还在运行。Taskqueue 任务、中断处理程序和基于定时器的重新武装函数都需要在释放内存之前停止。

内核为 taskqueue 任务提供了 `taskqueue_drain(tq, task)`，类似于 callout 的 `callout_drain`。对于中断，`bus_teardown_intr` 和 `bus_release_resource` 确保中断处理程序不会再被调用。对于处理程序自行重新调度的可重新武装 callout，`callout_drain` 仍然做正确的事情：它等待当前调用完成并阻止进一步的重新武装。

拆卸路径的一般规则：

1. 清除异步代码检查的任何「运行中」或「已武装」标志。
2. 依次排空每个异步源（taskqueue、callout、中断）。
3. 从上层分离（`ether_ifdetach`）。
4. 释放内存。

跳过步骤 1 通常是「销毁已加锁互斥锁」崩溃的原因，因为互斥锁被销毁时异步代码仍在运行。跳过步骤 2 是释放后使用的原因。步骤 3 和步骤 4 必须按此顺序执行，否则协议栈可能会在回调被释放后尝试调用它们。

### 一个实际的错误场景

为了使上述内容更具体，想象一个微妙的 bug。假设在开发过程中我们在 `callout_drain` 之前调用了 `mtx_destroy`。callout 已被调度，用户运行 `ifconfig mynet0 destroy`，我们的销毁函数销毁了互斥锁，然后已调度的 callout 触发。callout 尝试获取互斥锁（因为我们用 `callout_init_mtx` 注册了它），看到一个已销毁的互斥锁，并触发断言：「获取已销毁的互斥锁」。系统崩溃，栈跟踪指向 callout 代码。

修复方法是反转顺序：先 `callout_drain`，后 `mtx_destroy`。一般原则是同步原语在所有消费者都已知停止后才最后销毁。

这种 bug 很容易引入，但如果你以前没见过就很难诊断。拥有一个明确的「停止、分离、释放」心智模型可以防止它。

### 第8节小结

完整的生命周期现在掌握在你手中。加载、克隆器注册、每接口创建、带有发送、接收、ioctl 和链路事件的活跃生命周期、每接口销毁、克隆器分离、模块卸载。你可以构建、测试、拆卸和重建，确信内核返回到干净状态。

接下来的章节是本章的动手部分：引导你完成我们描述的里程碑的实验、扩展驱动程序的挑战、故障排除提示和总结。

## 动手实验

下面的实验按章节流程排序。每个建立在前一个之上，所以按顺序进行。配套文件位于 `examples/part-06/ch28-network-driver/` 下，每个实验都有自己的 README，包含具体的命令。

开始之前，确保你在一个拥有 root 权限的 FreeBSD 14.3 实验虚拟机上，有一个可以构建内核模块的干净工作区目录，以及一个如果出现问题可以返回的新快照状态。在开始实验之前做快照是一个小的投资，在你第一次需要它时就会收回成本。

每个实验都以一个简短的「检查点」块结束，列出你应在实验日志中记录的具体观察。如果你的实验日志已经有这些观察，你可以继续。如果没有，返回上一步重做。实验的累积结构意味着实验 2 中遗漏的观察会使实验 4 变得令人困惑。

### 实验1：构建和加载骨架

**目标。** 构建第 3 节的骨架驱动程序，加载它，创建一个实例，并观察默认状态。

**步骤。**

1. `cd examples/part-06/ch28-network-driver/`
2. `make` 并注意警告。构建应该产生 `mynet.ko` 且没有警告。
3. `kldload ./mynet.ko`。控制台不应出现任何消息；`kldstat` 应列出 `mynet` 为已存在。
4. `ifconfig mynet create` 应打印 `mynet0`。
5. `ifconfig mynet0` 并在日志中记录输出。特别注意标志、MAC 地址、媒体行和状态。
6. `kldstat -v | grep mynet` 并验证模块存在且在预期地址加载。
7. `sysctl net.generic.ifclone` 并确认 `mynet` 出现在克隆器列表中。
8. `ifconfig mynet0 destroy`。接口应消失。
9. `kldunload mynet`。模块应干净地卸载。
10. `kldstat` 和 `ifconfig -a` 确认没有残留。

**注意观察。** `ifconfig mynet0` 输出应显示标志 `BROADCAST,SIMPLEX,MULTICAST`、一个 MAC 地址、一行"Ethernet autoselect"的媒体行和"no carrier"的状态。如果缺少其中任何一个，重新检查 `mynet_create_unit` 函数和 `ifmedia_init` 调用。

**实验日志检查点。**

* 记录分配给 `mynet0` 的确切 MAC 地址。
* 记录 `if_mtu` 的初始值。
* 注意 `ifconfig mynet0 up` 前后报告的标志。
* 注意 `status:` 是否在 "no carrier" 和 "active" 之间变化。

**如果出现问题。** 最常见的实验 1 失败是由缺少头文件导致的构建错误。确保 `/usr/src/sys/` 下的内核源代码树与运行中的内核版本匹配。如果 `kldload` 失败并显示"module already present"，用 `kldunload mynet` 卸载任何先前的实例然后重试。如果 `ifconfig mynet create` 返回"Operation not supported"，克隆器未注册，你需要重新检查 `VNET_SYSINIT` 调用。

### 实验2：练习发送路径

**目标。** 验证当流量离开接口时 `if_transmit` 被调用。

**步骤。**

1. 如实验 1 那样创建接口并启动它。
2. `ifconfig mynet0 inet 192.0.2.1/24 up`。`UP` 和 `RUNNING` 标志现在都应出现。
3. 在一个终端中，运行 `tcpdump -i mynet0 -nn`。
4. 在另一个终端中，运行 `ping -c 3 192.0.2.99`。
5. 观察 `tcpdump` 打印的 ARP 和 ICMP 流量。
6. `netstat -in -I mynet0` 并记录计数器。`Opkts` 列应显示至少四个（三次 ICMP 请求加上 ARP 广播尝试）。
7. 修改发送函数为每次调用返回 `ENOBUFS` 并重新构建。
8. 卸载并重新加载，重复 `ping`，观察 `Opkts` 停止增长且 `Oerrors` 增加。
9. 还原修改并重新构建。
10. 可选：在生成流量时运行 DTrace 单行命令 `dtrace -n 'fbt::mynet_transmit:entry { @c = count(); }'`，确认每次调用都到达你的发送函数。

**注意观察。** 在步骤 5 中，每次 `ping` 产生一个 ARP 广播（因为协议栈不知道 `192.0.2.99` 的 MAC）和每次 ping 尝试一个 ICMP 回显请求，但 ARP 回复永远不会来，所以后续的 ping 只添加 ICMP 请求。理解为什么会这样，以及它在 `tcpdump` 中看起来是什么样，是这个实验的重要部分。

**实验日志检查点。**

* 记录三次 ping 后的确切 `Opkts` 计数。
* 记录 `Obytes` 计数并验证它匹配 ARP 帧（42 字节）加上三个 ICMP 帧的预期总和。
* 注意当你故意返回 `ENOBUFS` 时 `Oerrors` 有什么变化。

**如果出现问题。** 如果 ping 后 `Opkts` 为零，你的 `if_transmit` 回调没有被调用。检查创建期间是否设置了 `ifp->if_transmit = mynet_transmit`。如果 `Obytes` 在增长但 `Opkts` 没有，说明某个计数器调用缺失或到达了错误的计数器。如果 `tcpdump` 没有显示出站流量，发送中缺少 BPF 捕获；在释放前添加 `BPF_MTAP(ifp, m)`。

### 实验3：练习接收路径

**目标。** 验证 `if_input` 将数据包送入协议栈。

**步骤。**

1. 创建接口并启动它。
2. `tcpdump -i mynet0 -nn`。
3. 等待五秒并确认每秒出现一个合成 ARP 请求。
4. `netstat -in -I mynet0` 并确认 `Ipkts` 与数据包计数匹配。
5. 将 `sc->rx_interval_hz = hz / 10;` 修改并重新构建。
6. 卸载、重新加载、重新创建。观察速率变为每秒十个数据包。
7. 还原为每秒一个数据包。
8. 可选：注释掉接收路径中的 `BPF_MTAP` 调用，重新构建，观察 `tcpdump` 不再显示合成 ARP 但 `Ipkts` 仍然递增。这确认 BPF 可见性和计数器更新是独立的。
9. 可选：注释掉 `if_input` 调用（保留 `BPF_MTAP`），重新构建，观察相反的行为：`tcpdump` 看到帧，但 `Ipkts` 不动，因为协议栈从未真正收到帧。

**注意观察。** `Ipkts` 计数器应该每个合成帧恰好递增一次。如果不是，BPF 捕获可能看到了帧但 `if_input` 没有被调用，或者调用与拆卸存在竞争。

**实验日志检查点。**

* 记录 `tcpdump` 时间戳显示的连续合成 ARP 之间的间隔。
* 记录 ARP 帧中的 MAC 地址并确认源 MAC 与接口地址匹配。
* 观察之前和之后 `arp -an` 显示什么；`192.0.2.99` 的条目应保持不完整。

**如果出现问题。** 如果 `tcpdump` 中没有合成 ARP 出现，callout 没有触发。检查 `mynet_init` 中是否调用了 `callout_reset` 以及当时 `sc->running` 是否为 true。如果 `tcpdump` 显示 ARP 但 `Ipkts` 为零，计数器没有更新（或者在 `if_input` 之后更新，而 `if_input` 已经释放了 mbuf）。

### 实验4：媒体和链路状态

**目标。** 观察链路状态、媒体和接口标志之间的区别。

**步骤。**

1. 创建并配置接口。
2. `ifconfig mynet0` 并注意 `status` 和 `media` 行。
3. `ifconfig mynet0 down`。
4. `ifconfig mynet0` 并注意 `status` 的变化。
5. `ifconfig mynet0 up`。
6. 在另一个终端中，`route monitor` 并在观察输出的同时重复步骤 3 和 5。
7. `ifconfig mynet0 media 1000baseT mediaopt full-duplex` 并确认 `ifconfig mynet0` 反映了更改。
8. 在 `mynet_create_unit` 中添加第三个媒体条目 `IFM_ETHER | IFM_100_TX | IFM_FDX`，重新构建，验证 `ifconfig mynet0 media 100baseTX mediaopt full-duplex` 现在成功。
9. 删除该条目并重新构建。验证相同的命令现在以"requested media type not found"失败。

**注意观察。** `route monitor` 在每次链路状态转换时打印 `RTM_IFINFO` 消息。`ifconfig mynet0` 的 `status:` 行在驱动程序运行且链路已连接时显示 `active`，在驱动程序调用 `LINK_STATE_DOWN` 时显示 `no carrier`。

**实验日志检查点。**

* 记录 `route monitor` 中确切的 `RTM_IFINFO` 消息文本。
* 通过在四种可能的组合（up 或 down 与链路 up 或 down 交叉）下捕获 `ifconfig mynet0` 的输出来记录 `IFF_UP` 和 `LINK_STATE_UP` 之间的区别。
* 观察在所有四种状态下 `status:` 和接口标志是否保持一致。

**如果出现问题。** 如果 `status:` 在接口启动后仍然停留在"no carrier"，说明你没有从 `mynet_init` 调用 `if_link_state_change(ifp, LINK_STATE_UP)`。如果 `ifconfig mynet0 media 1000baseT` 失败并显示"requested media type not found"，说明你没有通过 `ifmedia_add` 注册 `IFM_ETHER | IFM_1000_T`，或者你用错误的标志注册了它。

### 实验5：`tcpdump` 和 BPF

**目标。** 确认 BPF 能看到出站和入站数据包。

**步骤。**

1. 创建并配置接口，IP 为 `192.0.2.1/24`。
2. `tcpdump -i mynet0 -nn > /tmp/dump.txt &`
3. 等待十秒。
4. `ping -c 3 192.0.2.99`。
5. 再等待十秒。
6. `kill %1`。
7. `cat /tmp/dump.txt` 并识别合成 ARP 请求、你的 `ping` 生成的 ARP 广播以及 ICMP 回显请求。
8. 从 `mynet_transmit` 中移除 `BPF_MTAP` 调用并重新构建。重复。注意出站 ICMP 不再出现在 `tcpdump` 输出中。
9. 恢复 `BPF_MTAP` 调用。
10. 实验过滤器：`tcpdump -i mynet0 -nn 'arp'` 应仅显示合成 ARP 和你 ping 的 ARP，而 `tcpdump -i mynet0 -nn 'icmp'` 应仅显示 ICMP 回显请求。
11. 观察 `tcpdump` 启动输出中的 link-type 行。它应显示 `EN10MB (Ethernet)`，因为 `ether_ifattach` 为我们设置了这个。如果显示 `NULL`，说明接口没有使用以太网语义附加。

**注意观察。** 本练习证明 BPF 可见性对每个数据包不是自动的。驱动程序有责任在发送和接收路径上都进行捕获。

**实验日志检查点。**

* 记录你观察到的每种帧类型的一行完整 `tcpdump` 输出：合成 ARP、出站 ARP、出站 ICMP 回显请求。
* 记录 `tcpdump` 打印的 link-type 行。
* 注意当你从发送中移除 `BPF_MTAP` 时输出发生了什么。

**如果出现问题。** 如果 `tcpdump` 从未显示任何数据包，说明 `bpfattach` 没有被调用（通常是因为你忘了 `ether_ifattach`）。如果它显示接收到的数据包但不显示发送的，你的发送捕获缺失。如果它显示发送的数据包但不显示接收的，你的接收捕获缺失。如果链路类型错误，接口类型或 `bpfattach` 调用是错误的。

### 实验6：干净分离

**目标。** 验证卸载将系统返回到干净状态。

**步骤。**

1. 创建三个接口：`mynet create` 三次。
2. 为每个接口配置 `192.0.2.0/24` 中的不同 IP（例如 `192.0.2.1/24`、`192.0.2.2/24`、`192.0.2.3/24`）。
3. `vmstat -m | grep mynet` 并记录分配计数。
4. `kldunload mynet`（不要先销毁）。
5. `ifconfig -a` 并确认 `mynet0`、`mynet1`、`mynet2` 都不存在了。
6. `vmstat -m | grep mynet` 并确认 `InUse` 返回零。
7. 按顺序重复步骤 1 到 6 五次。每轮应使 `InUse` 为零且不应留下任何孤立状态。
8. 可选：通过从 `mynet_destroy` 中移除 `callout_drain` 调用来引入人工 bug。重新构建、加载、创建接口并卸载。观察会发生什么（通常是崩溃，这是了解 `callout_drain` 为什么存在的戏剧性方式）。
9. 恢复 `callout_drain` 调用。

**注意观察。** 克隆器分离路径应遍历所有三个接口，在每个上调用 `mynet_clone_destroy`，并释放所有内存。如果任何接口残留，或 `InUse` 非零，说明拆卸中的某些部分有错误。

**实验日志检查点。**

* 记录每轮加载-创建-卸载前后的 `InUse` 值。
* 注意 `vmstat -m | grep mynet` 中的 `Requests` 列；它应单调递增，因为它记录了生命周期分配。
* 记录 `dmesg` 中任何意外消息。

**如果出现问题。** 如果 `kldunload` 挂起，一个 callout 或 taskqueue 任务仍在运行。使用 `ps -auxw` 找到内核线程并使用 `procstat -k <pid>` 查看其栈跟踪。如果卸载后 `InUse` 保持非零，你有内存泄漏；通常的嫌疑是某个接口没有被调用 `mynet_destroy`，这意味着 `if_clone_detach` 没有找到它。

### 实验7：阅读真实源码树

**目标。** 将你构建的内容与 `/usr/src/sys/net/` 中的内容联系起来。

**步骤。**

1. 将 `/usr/src/sys/net/if_disc.c` 和你的 `mynet.c` 并排打开。对于以下每一项，在两个文件中找到对应的代码：
   * 克隆器注册。
   * softc 分配。
   * 接口类型（`IFT_LOOP` vs `IFT_ETHER`）。
   * BPF 附加。
   * 发送路径。
   * ioctl 处理。
   * 克隆器销毁。
2. 打开 `/usr/src/sys/net/if_epair.c` 并做同样的练习。注意 `if_clone_advanced` 的使用、配对逻辑和 `ifmedia_init` 的使用。
3. 打开 `/usr/src/sys/net/if_ethersubr.c` 并找到 `ether_ifattach`。逐行追踪，并将每一行与我们在第 3 节中说的它做什么进行交叉参考。
4. 打开 `/usr/src/sys/net/bpf.c` 并找到 `bpf_mtap_if`，这是 `BPF_MTAP` 展开后的函数。注意活跃对等体的快速路径检查。

**注意观察。** 这个实验的目标是识别，不是理解。你不需要理解 `epair(4)` 或 `ether_ifattach` 的每一行。你只需要看到我们在驱动程序中使用的相同模式出现在真正的源代码树中，你可能遇到的新代码是你已经知道的主题的变体。

**实验日志检查点。**

* 从 `if_disc.c`、`if_epair.c` 和 `if_ethersubr.c` 中各记录一个你现在能理解到足以口头解释的函数名。
* 注意这些文件中任何让你惊讶或与你从本章建立假设相矛盾的模式。

## 挑战练习

下面的挑战以小型、自包含的方向扩展驱动程序。每个都设计为一到两个集中注意力的会话内可完成，并且只依赖本章已教授的内容。

### 挑战1：成对接口之间的共享队列

**简要说明。** 修改 `mynet`，使创建两个成对接口（`mynet0a` 和 `mynet0b`）的行为像 `epair(4)`：在一个接口上发送会导致帧出现在另一个接口上。

**提示。** 使用带有匹配函数的 `if_clone_advanced`，像 `epair.c` 那样。在两个 softc 结构之间共享一个队列。使用 callout 或 taskqueue 在另一端出队并调用 `if_input`。

**预期结果。** 当你从分配给 `mynet0b` 的 IP ping 分配给 `mynet0a` 的 IP 时，回复应该真的能回来。你已经构建了两个线缆互相插入的软件模拟。

**关键设计问题。** 你在哪里存储共享队列？你如何确保在一侧发送的数据包不能被原始发送者看到（`IFF_SIMPLEX` 契约）？你如何处理只有一对中的一侧启动的情况？

**建议结构。** 添加一个拥有两个 softc 的 `struct mynet_pair`，让每个 softc 携带一个指向 pair 的指针。A 侧的发送函数将 mbuf 入队到 B 侧的输入队列并调度一个 taskqueue。taskqueue 出队并在 B 侧调用 `if_input`。在 pair 结构中使用互斥锁保护队列。

### 挑战2：链路翻转模拟

**简要说明。** 添加一个 sysctl `net.mynet.flap_interval`，当非零时，使驱动程序每隔 `flap_interval` 秒翻转链路的上下状态。

**提示。** 使用一个 callout，交替调用 `if_link_state_change` 和 `LINK_STATE_UP` 和 `LINK_STATE_DOWN`。观察对 `route monitor` 的影响。

**预期结果。** 启用翻转时，`ifconfig mynet0` 应在选定间隔交替显示 `status: active` 和 `status: no carrier`。`route monitor` 应在每个转换时打印 `RTM_IFINFO` 消息。

**扩展。** 使翻转间隔为每接口而非全局。你可以通过在 `net.mynet.<ifname>` 下为每个接口创建 sysctl 节点来实现，这需要使用 `sysctl_add_oid` 和类似的动态 sysctl API。

### 挑战3：错误注入

**简要说明。** 添加一个 sysctl `net.mynet.drop_rate`，设置出站帧被以错误丢弃的百分比。

**提示。** 在 `mynet_transmit` 中，通过 `arc4random` 生成一个随机数。如果它低于配置的百分比，递增 `IFCOUNTER_OERRORS`，释放 mbuf 并返回。否则像以前一样继续。

**预期结果。** `drop_rate` 设置为 50 时，`ping` 应显示大约 50% 的丢包率而不是 100%。（记住，没有 drop_rate 时的"100% 丢包"是因为没有回复返回，不是因为发送丢弃。所以设置 drop_rate=50 你仍然得到 100% ping 丢包；但如果你将此挑战与挑战 1 的成对接口组合，组合行为应该是 50% ping 丢包。）

**扩展。** 添加一个单独的 `rx_drop_rate` 来丢弃合成接收帧。观察接收计数器输出在发送丢弃和接收丢弃之间的区别。

### 挑战4：iperf3 压力测试

**简要说明。** 使用 `iperf3` 压力测试发送路径并测量驱动程序处理帧的速度。

**提示。** 运行 `iperf3 -c 192.0.2.99 -t 10 -u -b 1G` 来生成 UDP 洪水。观察前后的 `netstat -in -I mynet0`。观察 `vmstat 1` 的系统负载。考虑你需要改变驱动程序中的什么来支持更高速率：每 CPU 计数器、无锁发送路径、基于 taskqueue 的延迟处理。

**预期结果。** iperf3 运行不会产生有意义的带宽数字（因为没有服务器确认任何东西），但它会快速推高 `Opkts`。注意发送路径上的任何 CPU 热点。如果你已经与挑战 1 组合，成对接口设置应显示数据包穿过模拟链路。

**测量技巧。** 使用 `pmcstat` 或 `dtrace` 来分析时间花在哪里。发送路径是查找锁竞争的合理位置。如果你在 `mynet_transmit` 中看到 softc 互斥锁上的 `mtx_lock` 高速率，那是你在争用一个真正驱动程序会按队列拆分的锁的迹象。

### 挑战5：每接口 sysctl 树

**简要说明。** 在 `net.mynet.mynet0.*` 下暴露每接口的运行时控制和统计信息。

**提示。** 使用 `sysctl_add_oid` 在接口创建时动态添加每接口 sysctl，在接口销毁时移除它们。一个常见的模式是在静态根节点下创建每实例上下文，并为特定控制和统计信息附加子叶子节点。

**预期结果。** `sysctl net.mynet.mynet0.rx_interval_hz` 应该能读写接收间隔，覆盖编译时默认值。`sysctl net.mynet.mynet0.rx_packets_generated` 应该能读取一个每次合成接收定时器触发时递增的计数器。

**扩展。** 添加一个 `rx_enabled` sysctl 来暂停和恢复合成接收定时器。通过在切换 sysctl 的同时观察 `tcpdump` 来验证行为。

### 挑战6：Netgraph 节点

**简要说明。** 将 `mynet` 暴露为 netgraph 节点，以便可以将其接入 netgraph 框架。

**提示。** 这是一个较长的挑战，因为它需要熟悉 `netgraph(4)`。阅读 `/usr/src/sys/netgraph/ng_ether.c` 作为接口暴露为 netgraph 节点的参考示例。添加一个单一钩子，在我们的 `if_transmit` 和 `if_input` 之前或之后提供数据包拦截。

**预期结果。** netgraph 节点存在后，你应该能够使用 `ngctl` 附加一个过滤器或重定向节点，并观察数据包通过 netgraph 链流动。

此挑战是所有挑战中最开放的。如果你达到一个可用的骨架，你本质上已经完成了从"hello world"驱动程序到完全参与 FreeBSD 高级网络基础设施的驱动程序的路径。

## 故障排除和常见错误

网络驱动程序以少数几种特征方式失败。学会识别它们可以节省数小时的调试时间。

### 症状：`ifconfig mynet create` 返回 "Operation not supported"

**可能原因。** 克隆器未注册，或克隆器名称不匹配。检查 `V_mynet_cloner` 是否在 `vnet_mynet_init` 中初始化，以及 `mynet_name` 字符串是否是用户正在输入的。

**诊断。** `sysctl net.generic.ifclone` 列出所有已注册的克隆器。如果 `mynet` 缺失，说明注册没有发生。

### 症状：`ifconfig mynet0 up` 挂起或崩溃

**可能原因。** `mynet_init` 函数在持有 softc 互斥锁时做了会睡眠的操作，或者在持有互斥锁的情况下向上调用了协议栈。

**诊断。** 如果系统挂起，进入调试器（控制台中按 `Ctrl-Alt-Esc`）并输入 `ps` 查看哪个线程卡住了，然后 `trace TID` 获取栈跟踪。寻找有问题的锁获取。

### 症状：`tcpdump -i mynet0` 看不到任何数据包

**可能原因。** `BPF_MTAP` 没有被调用，或者接口设置期间没有调用 `bpfattach`。

**诊断。** `bpf_peers_present(ifp->if_bpf)` 在 `tcpdump` 运行时应返回 true。如果不是，检查是否调用了 `ether_ifattach`。如果 `ether_ifattach` 已调用但数据路径中没有 `BPF_MTAP`，在发送和接收中都添加该调用。

### 症状：`ping` 显示 100% 丢包（预期）但 `Opkts` 保持为零

**可能原因。** `if_transmit` 没有被调用，或者它提前返回而没有递增计数器。

**诊断。** `dtrace -n 'fbt::mynet_transmit:entry { @[probefunc] = count(); }'` 统计函数被调用的频率。如果为零，协议栈没有向我们分派，设置期间对 `ifp->if_transmit` 的赋值（或如果你切换到辅助函数，`if_settransmitfn` 调用）有问题。

### 症状：`kldunload` 因 "destroying locked mutex" 崩溃

**可能原因。** 互斥锁在另一个线程（通常是 callout）仍持有它时被销毁。

**诊断。** 审查拆卸顺序。`callout_drain` 必须在 `mtx_destroy` 之前调用。`ether_ifdetach` 必须在 `if_free` 之前调用。如果 callout 锁定了 softc 互斥锁，`callout_drain` 必须在该互斥锁消失之前发生。

### 症状：`netstat -in -I mynet0` 的 `Opkts` 高于 `systat -if` 中的 `Opkts`

**可能原因。** 发送路径中某个计数器被递增了两次。

**诊断。** 检查代码路径。一个常见错误是在驱动程序和辅助函数中都递增了 `IFCOUNTER_OPACKETS`。

### 症状：模块加载但 `ifconfig mynet create` 产生内核警告

**可能原因。** `ifnet` 的某个字段未正确初始化，或在没有有效 MAC 地址的情况下调用了 `ether_ifattach`。

**诊断。** 警告后运行 `dmesg`。内核通常会打印足够的上下文来识别有问题的字段。

### 症状：`kldunload` 返回但 `ifconfig -a` 仍显示 `mynet0`

**可能原因。** 克隆器分离没有遍历所有接口。这通常是接口在克隆器路径之外创建的迹象，或 `if_clone` 数据结构不同步。

**诊断。** 卸载后 `sysctl net.generic.ifclone` 不应列出 `mynet`。如果列出了，说明 `if_clone_detach` 没有完成。

### 症状：`iperf3` 负载下间歇性崩溃

**可能原因。** 发送路径和 ioctl 路径之间的竞争，通常是一个路径加锁而另一个没有。

**诊断。** 使用启用 `INVARIANTS` 和 `WITNESS` 的内核运行。这些选项添加锁序和断言检查，能立即捕获大多数竞争。它们是网络驱动程序开发的最佳工具。

### 症状：`ifconfig mynet0 mtu 9000` 成功但巨型帧失败

**可能原因。** 驱动程序声明了它实际无法传输的 MTU 范围。我们的参考驱动程序为了简单使用宽范围，但真正的驱动程序有硬件决定的硬上限。

**诊断。** 发送一个大于配置 MTU 的帧并观察 `IFCOUNTER_OERRORS` 递增。将声明的上限与实际能力对齐。

### 症状：`dmesg` 显示 "acquiring a destroyed mutex"

**可能原因。** callout、taskqueue 任务或中断处理程序在 `mtx_destroy` 被调用后获取互斥锁。几乎总是由不正确的拆卸顺序导致。

**诊断。** 追踪你的 `mynet_destroy`。`callout_drain` 及等价的排空操作必须在 `mtx_destroy` 之前发生。正确的顺序是「停止、分离、销毁」，而不是「销毁、停止」。

### 症状：`WITNESS` 报告锁序反转

**可能原因。** 两个线程以相反的顺序获取同一对锁。在网络驱动程序中，这最常发生在 softc 互斥锁和协议栈内部锁（如 ARP 表锁或路由表锁）之间。

**诊断。** 仔细阅读 `WITNESS` 输出；它显示两个栈跟踪。修复通常是在调用协议栈之前释放驱动互斥锁（例如，在 `if_input` 或 `if_link_state_change` 之前），我们在本章中一直建议这样做。

### 症状：中等负载下丢包

**可能原因。** mbuf 耗尽（检查 `vmstat -z | grep mbuf`）或发送队列没有背压并静默丢弃。

**诊断。** 负载前后的 `vmstat -z | grep mbuf`。如果 `mbuf` 或 `mbuf_cluster` 分配在攀升但不被归还，你有 mbuf 泄漏。如果它们被归还但驱动程序内部队列在丢弃，你需要扩大队列或实现背压。

### 症状：`ifconfig mynet0 inet6 2001:db8::1/64` 没有效果

**可能原因。** IPv6 未编译进你的内核，或接口没有声明 `IFF_MULTICAST`（IPv6 需要它）。

**诊断。** `sysctl net.inet6.ip6.v6only` 及类似命令告诉你 IPv6 是否存在。`ifconfig mynet0` 显示标志；确保 `MULTICAST` 是其中之一。

### 症状：模块加载但 `ifconfig mynet create` 不产生接口也不报错

**可能原因。** 克隆器的创建函数返回成功但从未实际分配接口。很容易通过在调用 `if_alloc` 之前返回 0 来造成。

**诊断。** 在创建回调开头添加 `printf("mynet_clone_create called\n")`。如果消息出现但没有创建接口，bug 在 printf 和 `if_attach` 调用之间。

### 症状：`sysctl net.link.generic` 返回意外结果

**可能原因。** 驱动程序损坏了通用 sysctl 处理程序读取的 `ifnet` 字段。这很罕见但表明有更深的 bug。

**诊断。** 使用启用 `INVARIANTS` 的内核运行并查找断言失败。有问题的写入通常在 `ifnet` 字段被初始化的地方附近。

## 快速参考表

下表总结了本章介绍的最常用 API 和常量。在实验过程中保持此页打开。

### 生命周期函数

| Function | Purpose |
| --- | --- |
| `if_alloc(type)` | Allocate a new `ifnet` of the given IFT_ type. |
| `if_free(ifp)` | Free an `ifnet` after detach. |
| `if_attach(ifp)` | Register the interface with the stack. |
| `if_detach(ifp)` | Unregister the interface. |
| `ether_ifattach(ifp, mac)` | Register an Ethernet-like interface. Wraps `if_attach` plus `bpfattach` and sets Ethernet defaults. |
| `ether_ifdetach(ifp)` | Undo `ether_ifattach`. |
| `if_initname(ifp, family, unit)` | Set the interface name. |
| `bpfattach(ifp, dlt, hdrlen)` | Register with BPF manually. Done automatically by `ether_ifattach`. |
| `bpfdetach(ifp)` | Unregister from BPF. Done automatically by `ether_ifdetach`. |
| `if_clone_simple(name, create, destroy, minifs)` | Register a simple cloner. |
| `if_clone_advanced(name, minifs, match, create, destroy)` | Register a cloner with a custom match function. |
| `if_clone_detach(cloner)` | Tear down a cloner and all its interfaces. |
| `callout_init_mtx(co, mtx, flags)` | Initialise a callout associated with a mutex. |
| `callout_reset(co, ticks, fn, arg)` | Schedule or rearm a callout. |
| `callout_stop(co)` | Cancel a callout. |
| `callout_drain(co)` | Synchronously wait for a callout to finish. |
| `ifmedia_init(ifm, mask, change, status)` | Initialise a media descriptor. |
| `ifmedia_add(ifm, word, data, aux)` | Add a supported media entry. |
| `ifmedia_set(ifm, word)` | Choose the default media. |
| `ifmedia_ioctl(ifp, ifr, ifm, cmd)` | Handle `SIOCGIFMEDIA` and `SIOCSIFMEDIA`. |
| `ifmedia_removeall(ifm)` | Free all media entries on teardown. |

### 数据路径函数

| Function | Purpose |
| --- | --- |
| `if_transmit(ifp, m)` | The driver's outbound callback. |
| `if_input(ifp, m)` | Deliver a mbuf to the stack. |
| `if_qflush(ifp)` | Flush any driver-internal queues. |
| `BPF_MTAP(ifp, m)` | Tap a frame to BPF if any observers. |
| `bpf_mtap2(bpf, data, dlen, m)` | Tap with a prepended header. |
| `m_freem(m)` | Free an entire mbuf chain. |
| `m_free(m)` | Free a single mbuf. |
| `MGETHDR(m, how, type)` | Allocate a mbuf as the head of a packet. |
| `MGET(m, how, type)` | Allocate a mbuf as a chain continuation. |
| `m_gethdr(how, type)` | Alternative form of MGETHDR. |
| `m_pullup(m, len)` | Ensure the first len bytes are contiguous. |
| `m_copydata(m, off, len, buf)` | Read bytes from a chain without consuming it. |
| `m_defrag(m, how)` | Flatten a chain into a single mbuf. |
| `mtod(m, type)` | Cast `m_data` to the requested type. |
| `if_inc_counter(ifp, ctr, n)` | Increment a per-interface counter. |
| `if_link_state_change(ifp, state)` | Report a link transition. |

### 常用 `IFF_` 标志

| Flag | Meaning |
| --- | --- |
| `IFF_UP` | Administratively up. User-controlled. |
| `IFF_BROADCAST` | Supports broadcast. |
| `IFF_DEBUG` | Debug tracing requested. |
| `IFF_LOOPBACK` | Loopback interface. |
| `IFF_POINTOPOINT` | Point-to-point link. |
| `IFF_RUNNING` | Alias for `IFF_DRV_RUNNING`. |
| `IFF_NOARP` | ARP disabled. |
| `IFF_PROMISC` | Promiscuous mode. |
| `IFF_ALLMULTI` | Receive all multicast. |
| `IFF_SIMPLEX` | Cannot hear own transmissions. |
| `IFF_MULTICAST` | Supports multicast. |
| `IFF_DRV_RUNNING` | Driver-private: resources allocated. |
| `IFF_DRV_OACTIVE` | Driver-private: transmit queue full. |

### 常用 `IFCAP_` 能力

| Capability | Meaning |
| --- | --- |
| `IFCAP_RXCSUM` | IPv4 receive checksum offload. |
| `IFCAP_TXCSUM` | IPv4 transmit checksum offload. |
| `IFCAP_RXCSUM_IPV6` | IPv6 receive checksum offload. |
| `IFCAP_TXCSUM_IPV6` | IPv6 transmit checksum offload. |
| `IFCAP_TSO4` | IPv4 TCP segmentation offload. |
| `IFCAP_TSO6` | IPv6 TCP segmentation offload. |
| `IFCAP_LRO` | Large receive offload. |
| `IFCAP_VLAN_HWTAGGING` | Hardware VLAN tagging. |
| `IFCAP_VLAN_MTU` | VLAN over standard MTU. |
| `IFCAP_JUMBO_MTU` | Jumbo frames supported. |
| `IFCAP_POLLING` | Polled rather than interrupt-driven. |
| `IFCAP_WOL_MAGIC` | Wake-on-LAN magic packet. |
| `IFCAP_NETMAP` | `netmap(4)` support. |
| `IFCAP_TOE` | TCP offload engine. |
| `IFCAP_LINKSTATE` | Hardware link-state events. |

### 常用 `IFCOUNTER_` 计数器

| Counter | Meaning |
| --- | --- |
| `IFCOUNTER_IPACKETS` | Packets received. |
| `IFCOUNTER_IERRORS` | Receive errors. |
| `IFCOUNTER_OPACKETS` | Packets transmitted. |
| `IFCOUNTER_OERRORS` | Transmit errors. |
| `IFCOUNTER_COLLISIONS` | Collisions (Ethernet). |
| `IFCOUNTER_IBYTES` | Bytes received. |
| `IFCOUNTER_OBYTES` | Bytes transmitted. |
| `IFCOUNTER_IMCASTS` | Multicast packets received. |
| `IFCOUNTER_OMCASTS` | Multicast packets transmitted. |
| `IFCOUNTER_IQDROPS` | Receive queue drops. |
| `IFCOUNTER_OQDROPS` | Transmit queue drops. |
| `IFCOUNTER_NOPROTO` | Packets for unknown protocol. |

### 常用接口 ioctl

| Ioctl | When issued | Driver responsibility |
| --- | --- | --- |
| `SIOCSIFFLAGS` | `ifconfig up` / `down` | Bring the driver up or down. |
| `SIOCSIFADDR` | `ifconfig inet 1.2.3.4` | Address assignment. Usually handled by `ether_ioctl`. |
| `SIOCSIFMTU` | `ifconfig mtu N` | Validate and update `if_mtu`. |
| `SIOCADDMULTI` | Multicast group joined | Reprogram hardware filter. |
| `SIOCDELMULTI` | Multicast group left | Reprogram hardware filter. |
| `SIOCGIFMEDIA` | `ifconfig` display | Return current media. |
| `SIOCSIFMEDIA` | `ifconfig media X` | Reprogram PHY or equivalent. |
| `SIOCSIFCAP` | `ifconfig ±offloads` | Toggle offloads. |
| `SIOCSIFNAME` | `ifconfig name X` | Rename the interface. |

## 阅读真实的网络驱动程序

巩固你理解的最佳方式之一是阅读 FreeBSD 源代码树中的真实驱动程序。本节引导你浏览几个说明重要模式的驱动程序，并建议一个建立在你所学内容之上的阅读顺序。你不需要理解这些文件中的每一行。目标是识别：在不同大小和用途的驱动程序内部看到 `ether_ifattach`、`if_transmit`、`if_input`、`ifmedia_init` 等熟悉的骨架。

### 阅读 `/usr/src/sys/net/if_tuntap.c`

`tun(4)` 和 `tap(4)` 驱动程序在此文件中一起实现。它们给用户态一个文件描述符，通过它数据包可以在内核内外流动。阅读 `if_tuntap.c` 向你展示驱动程序如何桥接第 14 章的用户态字符设备世界和本章的网络协议栈世界。

打开文件并寻找以下标志：

* 顶部的 `cdevsw` 声明，这是用户态打开 `/dev/tun0` 或 `/dev/tap0` 的方式。
* `tunstart` 函数，将数据包从内核接口队列移动到用户态读取。
* `tunwrite` 函数，将数据包从用户态写入通过 `if_input` 移入内核。
* `tuncreate` 函数，分配 ifnet 并注册它。

你会看到 `tap` 使用 `ether_ifattach`，`tun` 使用普通的 `if_attach`，因为两种风格在链路层语义上不同：`tap` 是一个看起来像以太网的隧道，而 `tun` 是没有链路层的纯 IP 隧道。此文件是展示选择 `ether_ifattach` 还是 `if_attach` 如何波及驱动程序其余部分的绝佳案例研究。

注意 `tuntap` 没有像 `disc` 那样以相同方式使用接口克隆器。它在用户态打开 `/dev/tapN` 时按需创建接口，这展示了接口可以产生的又一种方式。这是克隆器模式的变体，而不是对它的偏离。

### 阅读 `/usr/src/sys/net/if_bridge.c`

网桥驱动程序实现了多个接口之间的软件以太网桥接。它是一个更大的文件（超过三千行），但其核心是相同的：它为每个网桥创建一个 ifnet，通过 `if_input` 钩子从成员接口接收帧，在 MAC 地址到端口的表中查找目的地，并通过出站端口上的 `if_transmit` 转发帧。

`if_bridge.c` 特别有教益的地方在于它本身既是 `ifnet` 接口的客户端又是提供者。它是客户端，因为它向成员接口发送帧。它是提供者，因为它暴露了其他代码可以使用的网桥接口。阅读它向你展示了如何编写一个透明地分层在其他驱动程序之上的驱动程序。

### 阅读 `/usr/src/sys/dev/e1000/if_em.c`

`em(4)` 驱动程序是 Intel e1000 级硬件的 PCI 以太网驱动程序的典型示例。它比我们的伪驱动程序大得多，因为它做了真正硬件所需的一切：PCI 附加、寄存器编程、EEPROM 读取、MSI-X 分配、环形缓冲区管理、DMA、中断处理等等。

然而，如果你眯起眼睛跳过硬件特定的部分，你会到处看到我们熟悉的模式：

* `em_if_attach_pre` 分配 softc。
* `em_if_attach_post` 填充 ifnet。
* `em_if_init` 是 `if_init` 回调。
* `em_if_ioctl` 是 `if_ioctl` 回调。
* `em_if_tx` 是发送回调（通过 iflib 包装）。
* `em_if_rx` 是接收回调（通过 iflib 包装）。
* `em_if_detach` 是分离函数。

该驱动程序使用 `iflib(9)` 而不是原始 `ifnet` 调用，但 iflib 本身就是我们一直使用的相同 API 之上的薄层。阅读 `em` 是看到真正驱动程序如何从我们的小型教学示例扩展的好方法。

先关注发送函数。你会看到描述符环管理、DMA 映射、TSO 处理和校验和卸载决策。状态量更大，但每个决策都有明确的目的，映射到我们讨论过的某个概念。

### 阅读 `/usr/src/sys/dev/virtio/network/if_vtnet.c`

`vtnet(4)` 驱动程序用于虚拟机使用的 VirtIO 网络适配器。它比 `em` 小但仍然比我们的伪驱动程序大。它使用 `virtio(9)` 作为其传输层而不是 `bus_space(9)` 加 DMA 环，如果你不太熟悉 PCI 硬件，这使代码更容易跟随。

`vtnet` 在 `mynet` 之后是特别好的第二个真实驱动程序阅读选择，因为：

* 它在几乎所有 FreeBSD 虚拟机中使用。
* 其源代码干净且注释良好。
* 它演示了多队列发送和接收。
* 它展示了卸载如何与发送路径交互。

花一个晚上阅读发送路径和接收路径。你可能会发现自己立即识别出 70% 到 80% 的模式，不熟悉的 20% 将是像 VirtIO 队列管理这样属于传输层而非网络驱动程序契约的东西。

### 阅读 `/usr/src/sys/net/if_lagg.c`

链路聚合驱动程序实现了 802.3ad LACP、轮询、故障转移和其他绑定协议。它本身是一个 ifnet 并在成员 ifnet 之上聚合。阅读它是了解聚合驱动程序如何分层在叶子驱动程序之上的练习，它向你展示了 `ifnet` 抽象的全部威力：绑定接口对协议栈来说看起来与单个网卡一样。

### 建议阅读顺序

如果你有时间进行更深入的学习，按以下顺序阅读：

1. `if_disc.c`：最小的伪驱动程序。你会认出其中的所有内容。
2. `if_tuntap.c`：伪驱动程序加上用户空间字符设备接口。
3. `if_epair.c`：带有模拟线路的成对伪驱动程序。
4. `if_bridge.c`：基于 ifnet 层的驱动程序。
5. `if_vtnet.c`：小型 VirtIO 真实驱动程序。
6. `if_em.c`：使用 iflib 的全功能真实驱动程序。
7. `if_lagg.c`：聚合驱动程序。
8. `if_wg.c`：WireGuard 隧道驱动程序。现代、加密、有趣。

按此顺序后你将看到足够多的驱动程序，以至于源代码树中几乎任何驱动程序都变得可读了。不熟悉的部分将归类为「这是硬件特定的」或「这是我还没学过的子系统」，两者都是有限且可征服的。

### 养成阅读习惯

培养每月阅读一个驱动程序的习惯。随机挑选一个，阅读附加函数，浏览发送和接收路径。你会惊讶于你的词汇量和阅读速度增长得有多快。到年底，你将能在从未见过的驱动程序中识别模式，「我应该在哪里寻找这个功能」的直觉会变得更敏锐。

阅读也是写作的最佳准备。当你需要给从未接触过的驱动程序添加新功能的那一刻，读过三十个驱动程序的经验意味着你大致知道去哪里找、模仿什么。

## 生产环境考量

本章大部分内容是关于理解的。在结束之前，简短一节关于从教学驱动程序转向将在生产环境中生存的驱动程序时什么会改变。

### 性能

生产驱动程序通常以每秒数据包数、每秒字节数或以微秒为单位的延迟来衡量。我们在本章构建的伪驱动程序在这些维度上都没有压力。如果你尝试将 `mynet` 用于真正的工作负载，你会很快碰到单互斥锁设计、同步 `m_freem` 和单队列分派的限制。

典型的改进包括：

* 每队列锁而非 softc 级锁。
* `drbr(9)` 用于每 CPU 发送环。
* 基于 taskqueue 的延迟接收处理。
* 使用 `m_getcl` 预分配 mbuf 池。
* 在某些路径中通过直接分派辅助函数绕过 `if_input`。
* 流哈希以将套接字固定到特定 CPU。
* 用于内核旁路工作负载的 Netmap 支持。

每个优化都增加代码。一个 10 Gbps 网卡的生产质量驱动程序可能有 3000 到 10000 行 C 代码，相比之下我们的教学驱动程序只有 500 行。

### 可靠性

生产驱动程序预期可以在持续运行数月而不泄漏内存、崩溃内核或计数器值漂移。使之成为可能的实践包括：

* 在 QA 中使用启用 `INVARIANTS` 和 `WITNESS` 的内核运行，以便断言能及早捕获 bug。
* 编写覆盖每个生命周期路径的回归测试。
* 让驱动程序通过压力测试（如 `iperf3`、pktgen 或 netmap pkt-gen）长时间运行。
* 为每个错误路径配备计数器来仪表化驱动程序，以便运维人员能在现场诊断问题。
* 通过 `dmesg`、sysctl 和 SDT 探测提供清晰的诊断信息。

这些实践对于将在规模上部署的驱动程序来说不是可选的。它们是准入成本。

### 可观测性

一个编写良好的生产驱动程序通过 sysctl、计数器和 DTrace 探测暴露足够的状态，使得运维人员可以在不添加 printf 或重新构建内核的情况下诊断大多数问题。经验法则是每个重要代码路径都应该有计数器或探测点，每个依赖运行时状态的决策都应该可以在不重建内核的情况下查询。

对于 `mynet` 我们只有内置的 ifnet 计数器。生产版本会添加用于发送路径入口、接收路径丢弃和中断处理程序调用等内容的每驱动程序计数器。这些计数器递增很廉价，当问题到来时无价。

### 向后兼容性

发布在发行版中的驱动程序也必须在未来的发行版上工作，最好不需要修改。FreeBSD 内核随时间演进其内部 API，深入接触结构的驱动程序可能会在这些结构变化时崩溃。

我们在第 2 节中介绍的访问器 API 是防御之一。使用 `if_setflagbits` 而不是 `ifp->if_flags |= flag` 使你免受布局变化的影响。同样，`if_inc_counter` 而不是直接计数器更新使你免受计数器表示变化的影响。

对于生产驱动程序，只要访问器现成就优先使用访问器风格。

### 许可证和上游提交

你打算合并到上游的驱动程序必须与 FreeBSD 源代码树兼容地许可，通常是两条款 BSD 许可证。它还应该遵循 KNF（`style(9)`），包含 `share/man/man4` 下的手册页，包含 `sys/modules` 下的模块 Makefile，并通过 FreeBSD 贡献流程提交（截至本文撰写时为 Phabricator 审查）。

像 `mynet` 这样的教学驱动程序不需要担心上游合并，但如果你编写驱动程序的意图是发布给其他人，这些是将你的 C 代码变成社区工件的额外考量。

## 总结

停下来欣赏一下你刚刚做了什么。你：

* 从头构建了你的第一个网络驱动程序。
* 通过 `ifnet` 和 `ether_ifattach` 将其注册到网络协议栈。
* 实现了接受 mbuf、捕获 BPF、更新计数器并清理的发送路径。
* 实现了构建 mbuf、交给 BPF 并传递给协议栈的接收路径。
* 处理了接口标志、链路状态转换和媒体描述符。
* 使用 `ifconfig`、`netstat`、`tcpdump`、`ping` 和 `route monitor` 测试了驱动程序。
* 在接口销毁和模块卸载时干净清理，没有泄漏。

比这些单独成就更重要的是你内化了一个心智模型。网络驱动程序是与内核网络协议栈契约的参与者。契约有固定的形状：几个向下的回调，一个向上的调用，少数标志，几个计数器，一个媒体描述符，一个链路状态。一旦你能清楚地看到这个形状，FreeBSD 源代码树中的每个网络驱动程序都变得可理解。生产驱动程序更大，但它们在根本上没有不同。

### 本章未涵盖的内容

有几个主题在可达范围内，但被有意推迟以保持本章的可控性。

**iflib(9)。** 大多数生产驱动程序在 FreeBSD 14 上使用的现代网卡驱动程序框架。iflib 在许多驱动程序之间共享发送和接收环形缓冲区，并为硬件网卡提供更简单的面向回调模型。我们在本章手工编写的模式正是 iflib 自动化的内容，所以你在这里学到的一切仍然有效。我们将在后面研究特定硬件驱动程序的章节中讨论 iflib。

**DMA 接收和发送。** 真正的网卡通过 DMA 映射的环形缓冲区移动数据包数据。我们在前面章节中介绍的 `bus_dma(9)` API 就是实现方式。向驱动程序添加 DMA 将 mbuf 构建故事变成「映射 mbuf，将映射地址交给硬件，等待完成中断，取消映射」。那是大量的额外代码，值得在后面的章节中单独处理。

**MSI-X 和中断调节。** 现代网卡有多个中断向量并支持中断合并。我们使用 callout 因为是伪驱动程序。真正的驱动程序使用中断处理程序。中断调节（让硬件将多个完成事件聚合为更少的中断）对性能至关重要。

**netmap(4)。** 一些高性能应用使用的内核旁路快速路径。驱动程序通过调用 `netmap_attach()` 并暴露每队列环形缓冲区来选择加入。它是吞吐量敏感用例的专门化。

**polling(4)。** 一种较旧的技术，驱动程序由内核线程轮询而非由中断驱动。仍然可用但不如以前常用。

**VNET 详解。** 我们设置驱动程序为 VNET 兼容的，但没有探索使用 `if_vmove` 在 VNET 之间移动接口意味着什么，或从驱动程序角度 VNET 拆卸看起来是什么样。第 30 章将访问该领域。

**硬件卸载。** 校验和卸载、TSO、LRO、VLAN 标记、加密卸载。所有这些都是真正的网卡可能暴露的能力。声明它们的驱动程序必须兑现它们，这导向了一个我们未触及的丰富设计空间。

**无线。** `wlan(4)` 驱动程序与以太网驱动程序根本不同，因为它们处理 802.11 帧格式、扫描、认证和管理帧。`ifnet` 仍然存在，但它位于一个非常不同的链路层之上。我们将在后面的章节中讨论无线驱动程序。

**网络图（`netgraph(4)`）。** FreeBSD 的包过滤和分类框架。它很大程度上正交于驱动程序编写，但值得了解用于高级网络架构。

**桥接和 VLAN 接口。** 聚合或修改流量的虚拟接口。它们构建在 `ifnet` 之上，与我们的驱动程序完全一样，但角色非常不同。

这些主题每个都值得自己的章节。你在这里构建的是那些探险出发的稳定大本营。

### 最后的思考

网络驱动程序有作为内核工程苛刻子领域的名声。它们当之无愧：约束很紧、与协议栈的交互很多、性能期望很高、面向用户的命令很多。但一旦你能看到它，网络驱动程序的结构是干净的。这就是本章给你的：看到结构的能力。

现在去读 `if_em.c`、`if_bge.c` 或 `if_tuntap.c`。你会认出骨架。softc。`ether_ifattach` 调用。`if_transmit`。`if_ioctl` 的 switch。接收处理程序中的 `if_input`。`bpfattach` 和 `BPF_MTAP`。无论代码在哪里增加复杂性，它都是在你已经以微缩形式构建的骨架上增加复杂性。

与第 27 章一样，本章很长，因为主题是分层的。我们试图让每一层在下一层到来之前平稳着陆。如果某个特定小节没有沉淀下来，回去重做对应的实验。内核学习是强烈累积的。重新过一遍某个小节通常比第一次过下一节效果更好。

### 延伸阅读

**手册页。** `ifnet(9)`、`ifmedia(9)`、`mbuf(9)`、`ether(9)`、`bpf(9)`、`polling(9)`、`ifconfig(8)`、`netstat(1)`、`tcpdump(1)`、`route(8)`、`ngctl(8)`。按此顺序阅读。

**FreeBSD 架构手册。** 网络章节是很好的补充材料。

**Kirk McKusick 等人的「FreeBSD 操作系统的设计与实现」。** 网络栈章节特别相关。

**Wright 和 Stevens 的「TCP/IP 详解 卷2」。** BSD 派生网络栈的经典逐步解析。虽然有些过时，但在深度上仍然独一无二。

**FreeBSD 源码树。** `/usr/src/sys/net/`、`/usr/src/sys/netinet/`、`/usr/src/sys/dev/e1000/`、`/usr/src/sys/dev/bge/`、`/usr/src/sys/dev/mlx5/`。本章讨论的每个模式都扎根于该代码中。

**邮件列表档案。** `freebsd-net@` 是最相关的列表。阅读历史讨论是学习从未进入正式文档的习语的好方法。

**GitHub 镜像上的提交历史。** FreeBSD 仓库有出色的历史记录。`git log --follow sys/net/if_var.h` 是查看 ifnet 抽象如何演进的良好起点。

**FreeBSD 开发者峰会幻灯片。** 在可用时，这些通常包括以网络为主题的会议。

**其他 BSD。** NetBSD 和 OpenBSD 有略微不同的网络驱动程序框架，但核心思想是相同的。在阅读 FreeBSD 对应物后阅读另一个 BSD 的驱动程序是理解什么是通用的、什么是 FreeBSD 特有的好方法。

## ifnet 相关子系统指南

你已经构建了一个驱动程序。你已经阅读了一些真正的驱动程序。在结束本章之前，让我们调查一下周围的子系统，以便你知道在需要时去哪里找。

### `arp(4)` 和邻居发现

IPv4 的 ARP 位于 `/usr/src/sys/netinet/if_ether.c`。它是将 IP 地址映射到 MAC 地址的子系统。驱动程序通常不直接与 ARP 交互；它们通过发送和接收路径携带数据包（包括 ARP 请求和回复），`ether_input` 和 `arpresolve` 内部的 ARP 代码处理其余的事情。

IPv6 的等价物是邻居发现，位于 `/usr/src/sys/netinet6/nd6.c`。它使用 ICMPv6 而不是单独的协议，但角色相同：为链路本地交付将 IPv6 地址映射到 MAC 地址。

### `bpf(4)`

Berkeley 数据包过滤器子系统位于 `/usr/src/sys/net/bpf.c`。BPF 是用户态可见的数据包捕获机制。`tcpdump(1)`、`libpcap(3)` 和许多其他工具使用 BPF。驱动程序通过 `bpfattach` 注册到 BPF（由 `ether_ifattach` 自动完成）并通过 `BPF_MTAP` 将数据包捕获到 BPF（你需要手动做）。

BPF 过滤器是用 BPF 伪机器语言编写的程序，在用户态编译为字节码并在内核中执行。它们是让 `tcpdump 'port 80'` 高效工作的原因：过滤器在数据包被复制到用户态之前运行，所以只传输匹配的数据包。

### `route(4)`

路由子系统位于 `/usr/src/sys/net/route.c` 并且随时间增长（最近的 `nhop(9)` 下一跳抽象是一个显著变化）。驱动程序间接地与路由交互：当它们报告链路状态变化时，路由子系统更新度量；当它们发送时，协议栈已经完成了路由查找。我们在实验中使用的 `route monitor` 订阅路由事件并显示它们。

### `if_clone(4)`

`/usr/src/sys/net/if_clone.c` 中的克隆器子系统是我们在本章中一直使用的。它管理每驱动程序克隆器列表并将 `ifconfig create` 和 `ifconfig destroy` 请求分派到正确的驱动程序。

### `pf(4)`

包过滤器位于 `/usr/src/sys/netpfil/pf/`。它独立于任何特定驱动程序，通过 `pfil(9)` 作为数据包路径上的钩子运行。驱动程序通常不直接与 `pf` 交互；通过协议栈的流量被透明过滤。

### `netmap(4)`

`netmap(4)` 是 `/usr/src/sys/dev/netmap/` 中的内核旁路数据包 I/O 框架。支持 netmap 的驱动程序将它们的环形缓冲区直接暴露给用户态，绕过正常的 `if_input` 和 `if_transmit` 路径。这允许应用程序以线速接收和发送数据包而无需内核参与。只有少数驱动程序原生支持 netmap；其余的使用一个 shim 以牺牲一些性能为代价模拟 netmap 语义。

### `netgraph(4)`

`netgraph(4)` 是 FreeBSD 的模块化数据包处理框架，位于 `/usr/src/sys/netgraph/`。它让你在内核中构建任意的数据包处理节点图，从用户态通过 `ngctl` 配置。驱动程序可以将自己暴露为 netgraph 节点（参见 `ng_ether.c`），netgraph 可用于实现隧道、以太网上的 PPP、加密链路和许多其他功能，而无需修改协议栈本身。

### `iflib(9)`

`iflib(9)` 是 `/usr/src/sys/net/iflib.c` 中用于高性能以太网驱动程序的现代框架。它接管网卡驱动程序中的例行部分（环形缓冲区管理、中断处理、TSO 分段、LRO 聚合），让驱动程序作者提供硬件特定的回调。在已转换为 iflib 的驱动程序中，驱动程序代码通常比等效的纯 ifnet 实现减少 30% 到 50%。参见附录 F 获取跨 iflib 和非 iflib 驱动程序语料库的可重现行数比较。附录 F 的 iflib 部分还在该范围的下限处对 ixgbe 转换提交固定了一个具体的每驱动程序测量。

目前，`iflib` 超出了本章的范围。附录 F 的 iflib 部分提供了一个可重现的行数比较，显示了框架在已转换驱动程序上节省了多少代码。

### 环境概览

网络驱动程序生活在丰富的环境中。它之上是 ARP、IP、TCP、UDP 和套接字层。它旁边是 BPF、`pf`、netmap 和 netgraph。它之下是硬件，或传输模拟，或到用户态的管道。这些组件每一个都有自己的约定，深入学习其中任何一个都是有意义的投资。本章给你的，是对中央对象 `ifnet` 足够的熟悉度，使你能接近这些子系统中的任何一个而不被吓倒。

## 调试场景：一个完整示例

结束关于驱动程序编写章节的最佳方式之一是走一遍具体的调试会话。下面的场景是复合的：它将几种不同驱动程序 bug 的症状和修复组合成一个叙述，以便「出问题了，让我们找到它」的完整弧线可见。

### 问题描述

你加载 `mynet`，创建接口，分配 IP，运行 `ping`。ping 报告 100% 丢包，正如预期（我们的伪驱动程序另一端什么也没有）。但 `netstat -in -I mynet0` 在多次 ping 后仍然显示 `Opkts 0`。发送路径中的某个东西坏了。

### 第一个假设：发送函数没有被调用。

你运行 `dtrace -n 'fbt::mynet_transmit:entry { printf("called"); }'`。即使在 ping 期间也没有输出。这确认了 `if_transmit` 没有被调用。

### 调查原因

你打开源代码，发现 `ifp->if_transmit = mynet_transmit;` 存在。你通过 `ifconfig` 的报告在运行时检查 `ifp->if_transmit`（没有直接的方法从用户空间读取函数指针，但 DTrace 探针可以做到）：

```console
# dtrace -n 'fbt::ether_output_frame:entry {
    printf("if_transmit = %p", args[0]->if_transmit);
}'
```

输出显示的地址与你预期的不同。仔细检查发现 `ether_ifattach` 用自己的包装器覆盖了 `if_transmit`。你在 `if_ethersubr.c` 中 grep `if_transmit` 并确认 `ether_ifattach` 设置了 `ifp->if_output = ether_output` 但没有触碰 `if_transmit`。所以 `if_transmit` 应该仍然是你的函数。

你回到源代码，注意到你在 `ether_ifattach` 之前设置了 `ifp->if_transmit = mynet_transmit;`，但你不小心在第二次赋值中通过遗留的 `if_start` 字段进行了设置，而你忘了删除。遗留的 `if_start` 机制在某些条件下优先，内核最终调用 `if_start` 而不是 `if_transmit`。

你删除多余的 `if_start` 赋值并重新构建。发送函数现在被调用了。

### 第二个问题：计数器不一致

发送函数现在被调用了，`Opkts` 在增长。但 `Obytes` 低得可疑：它每次 ping 只增加一，而不是按 ping 的字节长度增加。你重新检查计数器更新代码：

```c
if_inc_counter(ifp, IFCOUNTER_OBYTES, 1);
```

常量 `1` 应该是 `len`。你输入了错误的参数。你将其改为 `if_inc_counter(ifp, IFCOUNTER_OBYTES, len)` 并重新构建。`Obytes` 现在按预期数量增长。

### 第三个问题：接收路径似乎间歇性失效

合成的 ARP 大部分时间出现，但偶尔会停止几秒钟。你向 `mynet_rx_timer` 添加一个 DTrace 探针，看到函数在固定间隔被调用，但有些调用提前返回而没有生成帧。

你检查 `mynet_rx_fake_arp`，发现它使用 `M_NOWAIT` 进行 mbuf 分配。在内存压力下，`M_NOWAIT` 返回 NULL，接收路径静默丢弃。你在分配失败路径添加检测：

```c
if (m == NULL) {
    if_inc_counter(ifp, IFCOUNTER_IQDROPS, 1);
    return;
}
```

你检查计数器：它与丢失的帧数匹配。你找到了原因：你的测试虚拟机上的瞬时 mbuf 压力。修复方案要么接受偶尔的丢包（它们是合法的并且被正确计数），要么如果 callout 可以容忍睡眠则切换到 `M_WAITOK`（它不能，因为 callout 在不可睡眠的上下文中运行）。

在这种情况下，接受丢包是正确的。因此修复方案是使行为在仪表板上可见：你添加一个 sysctl 暴露此特定接口上的 `IFCOUNTER_IQDROPS`，并在驱动程序文档中注明。

### 这个场景教给我们什么

三个独立的 bug。没有一个是灾难性的。每一个都需要不同的工具组合来诊断：DTrace 用于函数追踪，代码阅读用于理解 API，计数器用于观察运行时效果。

教训是驱动程序 bug 倾向于隐藏在显眼的地方。驱动程序调试的第一条规则是"不要信任，要验证"。第二条规则是"计数器和工具会告诉你"。第三条规则是"如果计数器不能告诉你需要什么，添加更多计数器或更多探针"。

通过练习，这样的调试会话变得更快。你培养了首先选择哪个工具的直觉，一个在第一次加载就工作的驱动程序和一个需要六次迭代的驱动程序之间的区别变成了更短的调试周期。

## 关于测试纪律的说明

在我们真正关闭本章之前，关于测试纪律的几段话。教学驱动程序可以随意测试。一个你打算长期维护的驱动程序值得更严谨的方法。

### 单元级思维

你的驱动程序中的每个回调都有一个小而定义明确的契约。`mynet_transmit` 接收一个 ifnet 和一个 mbuf，验证、计数、tap 并释放。`mynet_ioctl` 接收一个 ifnet 和一个 ioctl 代码，分发并返回一个 errno。这些都可以独立地被测试。

在实践中，单元测试内核代码很困难，因为内核不容易嵌入到用户空间测试框架中。但你可以通过设计代码使每个回调的大部分是纯粹的来近似这种规范：给定输入，产生输出，不触碰全局状态。`mynet_transmit` 中的验证块就是一个好例子：它不触碰任何东西，除了 `ifp->if_mtu` 和局部变量。

"这个回调有一个契约；这里是行使该契约的用例；这里是每个用例的预期行为"这种心智模型是良好测试的基础。

### 生命周期测试

每个驱动程序都应该在其完整生命周期内进行测试：加载、创建、配置、承载流量、停止、销毁、卸载。实验 6 的脚本是这种测试的最低版本。更严格的版本将包括：

* 同时创建多个接口。
* 在流量流动时卸载（以低速率，以确保安全）。
* 重复的加载/卸载循环以捕获泄漏。
* 启用 INVARIANTS 和 WITNESS 的测试。

### 错误路径测试

驱动程序中的每个错误路径都需要可以被测试到。如果 `if_alloc` 失败，创建函数是否干净地回滚？如果 ioctl 返回错误，调用者是否能处理？如果 callout 无法分配 mbuf，驱动程序是否保持一致？

一个有用的技术是故障注入：添加一个 sysctl 来概率性地使特定操作失败（`if_alloc`、`m_gethdr` 等），并在启用故障注入的情况下运行生命周期测试。这暴露了在生产中几乎从不触发但仍然可能在负载下发生的错误路径。

### 回归测试

每当你修复一个 bug，添加一个能捕获它的测试。即使是一个简单的 shell 脚本——加载驱动程序、测试特定功能并检查计数器——也是一个回归测试。

随着时间的推移，回归测试套件成为防止重新引入 bug 的护栏。它还记录了你保证的行为。一个新贡献者阅读测试套件比阅读任何数量的代码更能清楚地了解驱动程序承诺了什么。

### 警惕潜在问题

有些问题只在数小时或数天的操作后才显现：缓慢的内存泄漏、计数器漂移、罕见的竞态条件。长时间运行的测试是发现这些问题的唯一方法。一个没有在代表负载下浸泡至少 24 小时就部署到生产的驱动程序还没有准备好。

对于 `mynet`，浸泡可能简单到"让驱动程序加载一天，最后检查 `vmstat -m` 和 `vmstat -z`"。对于真正的驱动程序，浸泡可能涉及在真实工作负载下的太字节小时级流量。规模不同；原则相同。

## `mynet.c` 完整走读

在我们关闭本章之前，值得展示参考驱动程序的简明端到端走读。目标是在一个地方看到整个驱动程序，每一步都有简短注释，这样你可以在不需要在第 3 到第 6 节之间跳转的情况下可视化完整形状。

### 文件级前言

驱动程序以许可证头、版权声明和我们在第 3 节中描述的 include 块开始。在 include 之后，文件声明了内存类型、克隆器变量和 softc 结构：

```c
static const char mynet_name[] = "mynet";
static MALLOC_DEFINE(M_MYNET, "mynet", "mynet pseudo Ethernet driver");

VNET_DEFINE_STATIC(struct if_clone *, mynet_cloner);
#define V_mynet_cloner  VNET(mynet_cloner)

struct mynet_softc {
    struct ifnet    *ifp;
    struct mtx       mtx;
    uint8_t          hwaddr[ETHER_ADDR_LEN];
    struct ifmedia   media;
    struct callout   rx_callout;
    int              rx_interval_hz;
    bool             running;
};

#define MYNET_LOCK(sc)      mtx_lock(&(sc)->mtx)
#define MYNET_UNLOCK(sc)    mtx_unlock(&(sc)->mtx)
#define MYNET_ASSERT(sc)    mtx_assert(&(sc)->mtx, MA_OWNED)
```

这里的每个字段和宏都有我们之前讨论过的目的。softc 承载每实例状态；锁定宏记录何时应持有互斥锁；VNET 感知的克隆器是 `ifconfig mynet create` 产生新接口的机制。

### 前向声明

一小块驱动程序作为回调暴露的静态函数的前向声明：

```c
static int      mynet_clone_create(struct if_clone *, int, caddr_t);
static void     mynet_clone_destroy(struct ifnet *);
static int      mynet_create_unit(int unit);
static void     mynet_destroy(struct mynet_softc *);
static void     mynet_init(void *);
static void     mynet_stop(struct mynet_softc *);
static int      mynet_transmit(struct ifnet *, struct mbuf *);
static void     mynet_qflush(struct ifnet *);
static int      mynet_ioctl(struct ifnet *, u_long, caddr_t);
static int      mynet_media_change(struct ifnet *);
static void     mynet_media_status(struct ifnet *, struct ifmediareq *);
static void     mynet_rx_timer(void *);
static void     mynet_rx_fake_arp(struct mynet_softc *);
static int      mynet_modevent(module_t, int, void *);
static void     vnet_mynet_init(const void *);
static void     vnet_mynet_uninit(const void *);
```

前向声明是对读者的礼貌。它们让你扫描文件顶部就能看到驱动程序导出的每个命名函数，无需搜索定义。

### 克隆器分发

克隆器的创建和销毁函数是薄包装器，将真正的工作委托给每单元辅助函数：

```c
static int
mynet_clone_create(struct if_clone *ifc __unused, int unit, caddr_t params __unused)
{
    return (mynet_create_unit(unit));
}

static void
mynet_clone_destroy(struct ifnet *ifp)
{
    mynet_destroy((struct mynet_softc *)ifp->if_softc);
}
```

保持克隆器回调小巧是一个值得遵循的惯例。它使真正的工作函数（`mynet_create_unit`、`mynet_destroy`）容易独立测试，并使克隆器粘合代码变得枯燥无趣。

### 每单元创建

每单元创建函数是真正设置发生的地方：

```c
static int
mynet_create_unit(int unit)
{
    struct mynet_softc *sc;
    struct ifnet *ifp;

    sc = malloc(sizeof(*sc), M_MYNET, M_WAITOK | M_ZERO);
    ifp = if_alloc(IFT_ETHER);
    if (ifp == NULL) {
        free(sc, M_MYNET);
        return (ENOSPC);
    }
    sc->ifp = ifp;
    mtx_init(&sc->mtx, "mynet", NULL, MTX_DEF);

    arc4rand(sc->hwaddr, ETHER_ADDR_LEN, 0);
    sc->hwaddr[0] = 0x02;  /* locally administered, unicast */

    if_initname(ifp, mynet_name, unit);
    ifp->if_softc = sc;
    ifp->if_flags = IFF_BROADCAST | IFF_SIMPLEX | IFF_MULTICAST;
    ifp->if_capabilities = IFCAP_VLAN_MTU;
    ifp->if_capenable = IFCAP_VLAN_MTU;
    ifp->if_transmit = mynet_transmit;
    ifp->if_qflush = mynet_qflush;
    ifp->if_ioctl = mynet_ioctl;
    ifp->if_init = mynet_init;
    ifp->if_baudrate = IF_Gbps(1);

    ifmedia_init(&sc->media, 0, mynet_media_change, mynet_media_status);
    ifmedia_add(&sc->media, IFM_ETHER | IFM_1000_T | IFM_FDX, 0, NULL);
    ifmedia_add(&sc->media, IFM_ETHER | IFM_AUTO, 0, NULL);
    ifmedia_set(&sc->media, IFM_ETHER | IFM_AUTO);

    callout_init_mtx(&sc->rx_callout, &sc->mtx, 0);
    sc->rx_interval_hz = hz;

    ether_ifattach(ifp, sc->hwaddr);
    return (0);
}
```

你可以在这里一处看到第 3 节中的每个概念：softc 和 ifnet 分配、MAC 制造、字段配置、媒体设置、callout 初始化，以及最终向协议栈注册接口的 `ether_ifattach`。

### 销毁

销毁以相反的顺序镜像创建：

```c
static void
mynet_destroy(struct mynet_softc *sc)
{
    struct ifnet *ifp = sc->ifp;

    MYNET_LOCK(sc);
    sc->running = false;
    ifp->if_drv_flags &= ~IFF_DRV_RUNNING;
    MYNET_UNLOCK(sc);

    callout_drain(&sc->rx_callout);

    ether_ifdetach(ifp);
    if_free(ifp);

    ifmedia_removeall(&sc->media);
    mtx_destroy(&sc->mtx);
    free(sc, M_MYNET);
}
```

同样，每一步都是我们讨论过的。顺序是静默、分离、释放。

### 初始化和停止

"未运行"和"运行"之间的转换由两个小函数处理：

```c
static void
mynet_init(void *arg)
{
    struct mynet_softc *sc = arg;

    MYNET_LOCK(sc);
    sc->running = true;
    sc->ifp->if_drv_flags |= IFF_DRV_RUNNING;
    sc->ifp->if_drv_flags &= ~IFF_DRV_OACTIVE;
    callout_reset(&sc->rx_callout, sc->rx_interval_hz,
        mynet_rx_timer, sc);
    MYNET_UNLOCK(sc);

    if_link_state_change(sc->ifp, LINK_STATE_UP);
}

static void
mynet_stop(struct mynet_softc *sc)
{
    MYNET_LOCK(sc);
    sc->running = false;
    sc->ifp->if_drv_flags &= ~IFF_DRV_RUNNING;
    callout_stop(&sc->rx_callout);
    MYNET_UNLOCK(sc);

    if_link_state_change(sc->ifp, LINK_STATE_DOWN);
}
```

两者是对称的，都遵循在 `if_link_state_change` 之前释放锁的规则，并且都维护我们在第 6 节中描述的一致性规则。

### 数据路径

发送和模拟接收是驱动程序的核心：

```c
static int
mynet_transmit(struct ifnet *ifp, struct mbuf *m)
{
    struct mynet_softc *sc = ifp->if_softc;
    int len;

    if (m == NULL)
        return (0);
    M_ASSERTPKTHDR(m);

    if (m->m_pkthdr.len > (ifp->if_mtu + sizeof(struct ether_vlan_header))) {
        m_freem(m);
        if_inc_counter(ifp, IFCOUNTER_OERRORS, 1);
        return (E2BIG);
    }

    if ((ifp->if_flags & IFF_UP) == 0 ||
        (ifp->if_drv_flags & IFF_DRV_RUNNING) == 0) {
        m_freem(m);
        if_inc_counter(ifp, IFCOUNTER_OERRORS, 1);
        return (ENETDOWN);
    }

    BPF_MTAP(ifp, m);

    len = m->m_pkthdr.len;
    if_inc_counter(ifp, IFCOUNTER_OPACKETS, 1);
    if_inc_counter(ifp, IFCOUNTER_OBYTES, len);
    if (m->m_flags & (M_BCAST | M_MCAST))
        if_inc_counter(ifp, IFCOUNTER_OMCASTS, 1);

    m_freem(m);
    return (0);
}

static void
mynet_qflush(struct ifnet *ifp __unused)
{
}

static void
mynet_rx_timer(void *arg)
{
    struct mynet_softc *sc = arg;

    MYNET_ASSERT(sc);
    if (!sc->running)
        return;
    callout_reset(&sc->rx_callout, sc->rx_interval_hz,
        mynet_rx_timer, sc);
    MYNET_UNLOCK(sc);

    mynet_rx_fake_arp(sc);

    MYNET_LOCK(sc);
}
```

伪造 ARP 辅助函数构建合成帧并将其传递给协议栈：

```c
static void
mynet_rx_fake_arp(struct mynet_softc *sc)
{
    struct ifnet *ifp = sc->ifp;
    struct mbuf *m;
    struct ether_header *eh;
    struct arphdr *ah;
    uint8_t *payload;
    size_t frame_len;

    frame_len = sizeof(*eh) + sizeof(*ah) + 2 * (ETHER_ADDR_LEN + 4);
    MGETHDR(m, M_NOWAIT, MT_DATA);
    if (m == NULL) {
        if_inc_counter(ifp, IFCOUNTER_IQDROPS, 1);
        return;
    }
    m->m_pkthdr.len = m->m_len = frame_len;
    m->m_pkthdr.rcvif = ifp;

    eh = mtod(m, struct ether_header *);
    memset(eh->ether_dhost, 0xff, ETHER_ADDR_LEN);
    memcpy(eh->ether_shost, sc->hwaddr, ETHER_ADDR_LEN);
    eh->ether_type = htons(ETHERTYPE_ARP);

    ah = (struct arphdr *)(eh + 1);
    ah->ar_hrd = htons(ARPHRD_ETHER);
    ah->ar_pro = htons(ETHERTYPE_IP);
    ah->ar_hln = ETHER_ADDR_LEN;
    ah->ar_pln = 4;
    ah->ar_op  = htons(ARPOP_REQUEST);

    payload = (uint8_t *)(ah + 1);
    memcpy(payload, sc->hwaddr, ETHER_ADDR_LEN);
    payload += ETHER_ADDR_LEN;
    memset(payload, 0, 4);
    payload += 4;
    memset(payload, 0, ETHER_ADDR_LEN);
    payload += ETHER_ADDR_LEN;
    memcpy(payload, "\xc0\x00\x02\x63", 4);

    BPF_MTAP(ifp, m);
    if_inc_counter(ifp, IFCOUNTER_IPACKETS, 1);
    if_inc_counter(ifp, IFCOUNTER_IBYTES, frame_len);
    if_inc_counter(ifp, IFCOUNTER_IMCASTS, 1);  /* broadcast counts as multicast */

    if_input(ifp, m);
}
```

### ioctl 和媒体回调

ioctl 处理程序和两个媒体回调：

```c
static int
mynet_ioctl(struct ifnet *ifp, u_long cmd, caddr_t data)
{
    struct mynet_softc *sc = ifp->if_softc;
    struct ifreq *ifr = (struct ifreq *)data;
    int error = 0;

    switch (cmd) {
    case SIOCSIFFLAGS:
        MYNET_LOCK(sc);
        if (ifp->if_flags & IFF_UP) {
            if ((ifp->if_drv_flags & IFF_DRV_RUNNING) == 0) {
                MYNET_UNLOCK(sc);
                mynet_init(sc);
                MYNET_LOCK(sc);
            }
        } else {
            if (ifp->if_drv_flags & IFF_DRV_RUNNING) {
                MYNET_UNLOCK(sc);
                mynet_stop(sc);
                MYNET_LOCK(sc);
            }
        }
        MYNET_UNLOCK(sc);
        break;

    case SIOCSIFMTU:
        if (ifr->ifr_mtu < 68 || ifr->ifr_mtu > 9216) {
            error = EINVAL;
            break;
        }
        ifp->if_mtu = ifr->ifr_mtu;
        break;

    case SIOCADDMULTI:
    case SIOCDELMULTI:
        break;

    case SIOCGIFMEDIA:
    case SIOCSIFMEDIA:
        error = ifmedia_ioctl(ifp, ifr, &sc->media, cmd);
        break;

    default:
        error = ether_ioctl(ifp, cmd, data);
        break;
    }

    return (error);
}

static int
mynet_media_change(struct ifnet *ifp __unused)
{
    return (0);
}

static void
mynet_media_status(struct ifnet *ifp, struct ifmediareq *imr)
{
    struct mynet_softc *sc = ifp->if_softc;

    imr->ifm_status = IFM_AVALID;
    if (sc->running)
        imr->ifm_status |= IFM_ACTIVE;
    imr->ifm_active = IFM_ETHER | IFM_1000_T | IFM_FDX;
}
```

### 模块胶水和克隆器注册

文件底部是模块处理程序、VNET sysinit/sysuninit 函数和模块声明：

```c
static void
vnet_mynet_init(const void *unused __unused)
{
    V_mynet_cloner = if_clone_simple(mynet_name, mynet_clone_create,
        mynet_clone_destroy, 0);
}
VNET_SYSINIT(vnet_mynet_init, SI_SUB_PSEUDO, SI_ORDER_ANY,
    vnet_mynet_init, NULL);

static void
vnet_mynet_uninit(const void *unused __unused)
{
    if_clone_detach(V_mynet_cloner);
}
VNET_SYSUNINIT(vnet_mynet_uninit, SI_SUB_INIT_IF, SI_ORDER_ANY,
    vnet_mynet_uninit, NULL);

static int
mynet_modevent(module_t mod __unused, int type, void *data __unused)
{
    switch (type) {
    case MOD_LOAD:
    case MOD_UNLOAD:
        return (0);
    default:
        return (EOPNOTSUPP);
    }
}

static moduledata_t mynet_mod = {
    "mynet",
    mynet_modevent,
    NULL
};

DECLARE_MODULE(mynet, mynet_mod, SI_SUB_PSEUDO, SI_ORDER_ANY);
MODULE_DEPEND(mynet, ether, 1, 1, 1);
MODULE_VERSION(mynet, 1);
```

### 代码行数与密度

完整的 `mynet.c` 大约有 500 行 C 代码。整个教学驱动程序，从顶部的许可证头到底部的 `MODULE_VERSION`，比生产驱动程序中许多单个函数还要短。这种紧凑性并非巧合：伪驱动程序没有硬件需要通信，因此它们可以专注于 `ifnet` 契约，仅此而已。

请阅读配套材料中的完整文件。如果你还没有这样做，请自己输入一份副本。构建它。加载它。修改它。在你内化其形态之前，不要继续下一章。

## 完整生命周期追踪

端到端地查看读者从 shell 运行普通命令时发生的事件序列会很有帮助。下面的追踪遵循我们已经建立的心智模型，但将其编织成一个连续的故事。把它当作动画翻页书来读，而不是参考表格。

### 追踪1：从 kldload 到 ifconfig up

想象你坐在一台全新的 FreeBSD 14.3 机器的键盘前。你以前从未加载过 `mynet`。你输入第一条命令：

```console
# kldload ./mynet.ko
```

接下来会发生什么？加载器读取 `mynet.ko` 的 ELF 头，将模块重定位到内核内存中，并遍历模块的 `modmetadata_set` 链接器集。它找到 `mynet` 的 `DECLARE_MODULE` 记录并调用 `mynet_modevent(mod, MOD_LOAD, data)`。我们的处理程序不做任何工作，直接返回零。加载器还处理 `MODULE_DEPEND` 记录，由于 `ether` 已经是基本内核的一部分，依赖关系立即得到满足。

然后遍历 `VNET_SYSINIT` 的链接器集。我们的 `vnet_mynet_init()` 被触发。它调用 `if_clone_simple()`，传入名称 `mynet` 和两个回调 `mynet_clone_create` 与 `mynet_clone_destroy`。内核在 VNET 克隆器列表中注册一个新的克隆器。此时，还没有任何接口存在：克隆器只是一个工厂。

Shell 提示符返回。你输入：

```console
# ifconfig mynet create
```

`ifconfig(8)` 打开一个数据报套接字并在其上发出 `SIOCIFCREATE2` ioctl，传入名称 `mynet`。内核的克隆分发器找到 `mynet` 克隆器并调用 `mynet_clone_create(cloner, unit, params, params_len)`，使用第一个可用的单元号，即零。我们的回调分配一个 `mynet_softc`，锁定其互斥锁，调用 `if_alloc(IFT_ETHER)`，填充回调，初始化媒体表，生成 MAC 地址，调用 `ether_ifattach()`，并返回零。在 `ether_ifattach()` 内部，内核调用 `if_attach()`，它将接口链接到全局接口列表，调用 `bpfattach()` 以便 `tcpdump(8)` 可以观察它，通过 `devd(8)` 将设备发布到用户空间，并运行所有已注册的 `ifnet_arrival_event` 处理程序。

Shell 提示符再次返回。你输入：

```console
# ifconfig mynet0 up
```

相同的套接字，相同类型的 ioctl，不同的命令：`SIOCSIFFLAGS`。内核按名称查找接口，找到 `mynet0`，并调用 `mynet_ioctl(ifp, SIOCSIFFLAGS, data)`。我们的处理程序观察到 `IFF_UP` 已设置但 `IFF_DRV_RUNNING` 未设置，因此它调用 `mynet_init()`。该函数将 `running` 翻转为 true，在接口上设置 `IFF_DRV_RUNNING`，调度第一个 callout 节拍，然后返回。ioctl 返回零。Shell 提示符返回。

你输入：

```console
# ping -c 1 -t 1 192.0.2.99
```

此时，网络栈尝试 ARP 解析。它构建一个 ARP 请求包，格式为 Ethernet + ARP，并为该接口调用 `ether_output()`。`ether_output()` 前置以太网头，调用 `if_transmit()`，这是一个调用我们 `mynet_transmit()` 函数的宏。我们的发送函数递增计数器、tap BPF、释放 mbuf 并返回零。`tcpdump -i mynet0` 会看到正在传输的 ARP 请求。

与此同时，因为我们的驱动程序也在 callout 定时器上生成伪造的入站 ARP 响应，下一个 callout 节拍合成一个 ARP 回复，调用 `if_input()`，协议栈相信它收到了来自 `192.0.2.99` 的响应。`ping` 发送 ICMP 回显请求，我们的驱动程序 tap 它，释放 mbuf，并记录成功。`ping` 永远不会收到回复，因为我们的驱动程序只伪造 ARP；但生命周期完全按预期工作，而且没有崩溃。

这个序列看起来微不足道，但它几乎执行了驱动程序中的每个代码路径。内化它。

### 追踪2：从 ifconfig down 到 kldunload

现在你来清理。你输入：

```console
# ifconfig mynet0 down
```

又是 `SIOCSIFFLAGS`，这次 `IFF_UP` 被清除。我们的 ioctl 处理程序看到 `IFF_DRV_RUNNING` 已设置但 `IFF_UP` 未设置，因此它调用 `mynet_stop()`。该函数将 `running` 翻转为 false，清除 `IFF_DRV_RUNNING`，排空 callout，然后返回。后续的发送尝试将被 `mynet_transmit()` 拒绝，因为有 `running` 检查。

```console
# ifconfig mynet0 destroy
```

`SIOCIFDESTROY` ioctl。内核找到拥有此接口的克隆器，并调用 `mynet_clone_destroy(cloner, ifp)`。我们的回调调用 `mynet_stop()`（双重保险：接口已经 down 了），然后调用 `ether_ifdetach()`，后者内部调用 `if_detach()`。`if_detach()` 将接口从全局列表中取消链接，排空所有引用，调用 `bpfdetach()`，通知 `devd(8)`，并运行 `ifnet_departure_event` 处理程序。我们的回调然后调用 `ifmedia_removeall()` 释放媒体列表，销毁互斥锁，用 `if_free()` 释放 `ifnet`，并用 `free()` 释放 softc。

```console
# kldunload mynet
```

加载器遍历 `VNET_SYSUNINIT` 并调用 `vnet_mynet_uninit()`，后者通过 `if_clone_detach()` 分离克隆器。然后 `mynet_modevent(mod, MOD_UNLOAD, data)` 运行并返回零。加载器将模块从内核内存中取消映射。系统干净了。

序列中的每个命令对应驱动程序中的一个特定回调。如果某个命令挂起，损坏的回调通常是显而易见的。如果某个命令崩溃，堆栈追踪直接指向它。练习这个追踪直到它变得机械化；你的余生作为驱动程序作者都在走它的变体。

## 关于网络驱动程序的常见误解

初学者带着一些反复出现的误解来到本章。明确指出它们有助于你避免日后出现微妙的 bug。

**"驱动程序解析以太网头。"** 并非如此。对于接收，驱动程序根本不解析以太网头：它将原始帧交给 `ether_input()`（在以太网框架下从 `if_input()` 调用），`ether_input()` 负责解析。对于发送，通用层的 `ether_output()` 前置以太网头；你的发送回调通常看到的是完整的帧，只需将其字节移出即可。驱动程序的工作是移动帧，而不是理解协议。

**"驱动程序必须了解 IP 地址。"** 不需要。以太网驱动程序完全在 IP 之下操作。它处理 MAC 地址、帧大小、多播过滤器和链路状态，但从不查看 IP 头。当你使用 `ifconfig mynet0 192.0.2.1/24` 将网络驱动程序附加到地址时，该分配存储在一个特定于协议族的结构中（一个 `struct in_ifaddr`），驱动程序从不接触它。驱动程序只看到出站帧和只产生入站帧：这些帧携带的是 IPv4、IPv6、ARP 还是其他异类协议，都不在它的职责范围内。

**"`IFF_UP` 意味着接口可以发送数据包。"** 部分正确。`IFF_UP` 意味着管理员说"我希望这个接口处于活动状态"。驱动程序通过初始化硬件（或者在我们的情况下，将 `running` 翻转为 true）并设置 `IFF_DRV_RUNNING` 来响应。这个区别很重要。`IFF_UP` 是用户意图；`IFF_DRV_RUNNING` 是驱动程序状态。只有后者可靠地指示驱动程序已准备好发送帧。如果你在发送前只检查 `IFF_UP`，你偶尔会将帧发送到半初始化的硬件状态并看着机器崩溃。

**"BPF 是调试时才启用的东西。"** BPF 对每个网络驱动程序始终处于开启状态。`ether_ifattach()` 内部的 `bpfattach()` 调用无条件地将接口注册到 Berkeley 包过滤器框架。当没有 BPF 监听器时，`BPF_MTAP()` 很便宜；它检查一个原子计数器然后返回。当存在监听器时，mbuf 被克隆并传递给每个监听器。你不需要做任何特殊的事情就能让 `tcpdump` 在你的驱动程序上工作；你只需要在两个路径上都调用 `BPF_MTAP()`。忘记那一个调用是新驱动程序在计数器中显示数据包但在 `tcpdump` 中什么也不显示的最常见原因。

**"如果我的驱动程序崩溃，内核会清理。"** 错误。驱动程序内部的崩溃就是内核内部的崩溃。没有进程边界来控制损害。如果你的发送函数解引用空指针，机器就会崩溃。如果你的回调泄漏了一个互斥锁，每个后续触及该接口的调用都会挂起。防御性地编写代码。在负载下测试。使用 INVARIANTS 和 WITNESS 构建。

**"网络驱动程序比存储驱动程序慢。"** 本质上并非如此。现代网卡每秒处理数千万个数据包，使用 `iflib(9)` 的良好驱动程序可以跟上。困惑来自于每个单独的数据包与存储请求相比非常小，因此笨拙设计的逐包开销会立即可见。一个粗糙的存储驱动程序可能仍然能达到线速的 80%，因为单次 I/O 移动 64 KiB；一个粗糙的网络驱动程序在线速的 10% 就会崩溃，因为每帧只有 1.5 KiB，逐帧开销占主导地位。

**"一旦我的驱动程序通过了 `ifconfig`，我就完成了。"** 远非如此。一个通过 `ifconfig up` 但在 `jail`、`vnet` 或模块卸载下失败的驱动程序仍然可能破坏生产系统。你在测试部分构建的严格测试矩阵才是真正的标准。许多生产 bug 只在功能交叉点被发现：VLAN 加 TSO、巨型帧加校验和卸载、混杂模式加多播过滤、快速 up/down 循环加 BPF 监听器。

每个误解都可以追溯到之前阅读中技术上准确但不完整的片段。既然你已经编写了一个驱动程序，这些边缘就变得清晰了。

## 数据包如何实际到达和离开你的驱动

值得放慢脚步，追踪数据包经过的确切路径。FreeBSD 网络栈的地理结构比你想象的更古老，其中大部分从驱动程序的角度是不可见的。了解这个地理结构会让你遇到的 bug 更容易诊断。

### 出站路径

当用户空间进程在 UDP 套接字上调用 `send()` 时，路径如下：

1. 系统调用进入内核并到达套接字层。数据使用 `sosend()` 从用户空间复制到内核 mbuf 中。
2. 套接字层将 mbuf 交给协议层，在本例中是 UDP。UDP 前置 UDP 头并将数据包交给 IP。
3. IP 前置 IP 头，使用路由表选择输出路由，并通过路由的 `rt_ifp` 指针将数据包交给接口特定的输出函数。对于以太网接口，该函数是 `ether_output()`。
4. `ether_output()` 调用 `arpresolve()` 查找目标 MAC 地址。如果 ARP 缓存有条目，执行继续。如果没有，数据包在 ARP 内部排队，并发送 ARP 请求；排队的数据包将在回复到达时被释放。
5. `ether_output()` 前置以太网头并调用 `if_transmit(ifp, m)`，这是驱动程序 `if_transmit` 回调上的一个薄宏。
6. 你的 `mynet_transmit()` 运行。它可以将 mbuf 排队到硬件、tap BPF、更新计数器，并根据是否拥有 mbuf 来释放或保留它。

六层，其中只有一层是你的驱动程序。其余的都是你永远不必触碰的场景。但是当 bug 发生时，理解哪一层可能负责是两小时修复和两天修复之间的区别。

### 入站路径

对于接收侧，路径以相反方向运行：

1. 一个帧到达线路（或者在我们的情况下，由驱动程序合成）。
2. 驱动程序用 `m_gethdr()` 构建 mbuf，填充 `m_pkthdr.rcvif`，用 `BPF_MTAP()` tap BPF，并调用 `if_input(ifp, m)`。
3. `if_input()` 是一个薄包装器，调用接口的 `if_input` 回调。对于以太网接口，`ether_ifattach()` 将此回调设置为 `ether_input()`。
4. `ether_input()` 检查以太网头，查找以太网类型（IPv4、IPv6、ARP 等），并调用适当的多路分解例程：IPv4 调用 `netisr_dispatch(NETISR_IP, m)`，ARP 调用 `netisr_dispatch(NETISR_ARP, m)`，依此类推。
5. netisr 框架可选地将数据包推迟到工作线程，然后将其传递给特定协议的输入例程。对于 IPv4，这是 `ip_input()`。
6. IP 解析头，执行源/目标检查，查询路由表以决定数据包是本地的还是转发的，然后将其上送到传输层或将其送回进行转发。
7. 如果数据包是给本地主机的且协议是 UDP，`udp_input()` 验证 UDP 校验和并将有效载荷传递到匹配套接字的接收缓冲区。
8. 调用 `recv()` 的用户空间进程被唤醒并读取数据。

接收侧有八层，同样，只有一层是你的驱动程序。但是看看 `m_pullup()` 可能在多少个地方被调用来使头在内存中连续，mbuf 可能在多少个地方被释放，计数器可能在多少个地方被递增。如果你看到 `ifconfig mynet0` 报告收到了数据包但 `tcpdump -i mynet0` 什么也没显示，差距最可能出现在步骤 2 和步骤 3 之间（你的 `BPF_MTAP()` 缺失或错误）。如果 `tcpdump` 显示了数据包但 `netstat -s` 显示它们被丢弃，差距最可能出现在步骤 6 和步骤 7 之间（路由表不认为接口拥有目标地址）。

### 为什么这些路径知识对驱动作者很重要

理解这个地理结构赋予你诊断能力。当出现问题时，你可以提出有针对性的问题。计数器在递增吗？出站路径的步骤 6 触发了。BPF 看到数据包了吗？你的 `BPF_MTAP()` 调用存在且接口标记为运行中。数据包到达对端了吗？硬件实际发送了它。每个问题对应地理结构中的一个特定检查点，每个检查点缩小了可能 bug 的范围。

生产驱动程序通过发送和接收环、批处理、硬件卸载和中断调节扩展了这个地理。每个优化改变了路径；但没有一个改变了整体形状。这个形状现在就值得记住，在优化使其变得混乱之前。

## 本章未涵盖的其他内容

一份诚实的省略清单帮助你了解接下来该学什么，它比人为的总结更精确地为第 29 章做了铺垫。

**硬件初始化。** 我们没有涉及 PCI 枚举、总线资源分配、中断设置或 DMA 环构建。为此，请仔细阅读 `/usr/src/sys/dev/e1000/if_em.c` 等驱动程序，特别是 `em_attach()` 和 `em_allocate_transmit_structures()`。你会看到 `bus_alloc_resource_any()`、`bus_setup_intr()`、`bus_dma_tag_create()` 和 `bus_dmamap_create()` 的实际运用。这些是让物理网卡真正移动数据的函数。

**iflib。** `iflib(9)` 框架抽象了现代以太网驱动程序的大部分繁琐部分。作为粗略的数量级估计，FreeBSD 14.3 中的新网卡驱动程序通常由大约 1,500 行硬件特定代码加上对 `iflib` 的调用组成，而不是完全手写环管理所需的约 10,000 行代码。我们提到了 `iflib` 但没有教授它，因为教学驱动程序不用它更简单。2026 年生产中的真正驱动程序可能使用 `iflib`。

**校验和卸载。** 现代网卡在硬件中计算 TCP、UDP 和 IP 校验和。设置 `IFCAP_RXCSUM`、`IFCAP_TXCSUM` 及其 IPv6 对应项需要驱动程序支持和 mbuf 标志操作（`CSUM_DATA_VALID`、`CSUM_PSEUDO_HDR` 等）。弄错了会默默地只为某些用户损坏流量。最好的入门是 `if_em.c` 的 `em_transmit_checksum_setup()` 函数，配合 `ether_input()` 查看标志如何向上传播。

**分段卸载。** TSO（发送）、LRO（接收）和 GSO（通用分段卸载）让主机将多段帧交给网卡，由硬件（或驱动程序级助手）将其拆分为 MTU 大小的片段。作为入门，请阅读 `tcp_output()` 并追踪它如何与 `if_hwtsomax` 和 `IFCAP_TSO4` 协作。

**多播过滤。** 真正的驱动程序根据通过 `SIOCADDMULTI` 通告的成员资格来编程硬件多播哈希表。我们 stub 了这些 ioctl；真正的实现遍历 `ifp->if_multiaddrs` 并操作网卡上的哈希寄存器。

**VLAN 处理。** 真正的驱动程序设置 `IFCAP_VLAN_HWTAGGING` 并允许 `vlan(4)` 将标记和去标记交给硬件处理。没有它，每个 VLAN 标记的帧都通过软件 `vlan_input()` 和 `vlan_output()` 处理，速度较慢但更简单。我们的驱动程序是 VLAN 透明的：它原样携带标记帧。

**通过 SIOCSIFCAP 进行卸载协商。** `ifconfig mynet0 -rxcsum` 在运行时切换能力。真正的驱动程序必须优雅地处理能力在运行中途改变的情况：刷新环、重新配置硬件，然后再次接受流量。

**SR-IOV。** 单根 I/O 虚拟化让物理网卡向虚拟机监控器呈现多个虚拟功能。FreeBSD 的支持（`iov(9)`）是不简单的。我们没有涉及它。

**无线。** 无线驱动程序使用 `net80211(4)`，一个构建在 `ifnet` 之上的独立框架。它们有丰富的状态机、复杂的速率控制、加密卸载和完全不同的监管合规要求。阅读 `/usr/src/sys/dev/ath/if_ath.c` 是值得花一个下午的，但它教授的大部分内容与我们这里构建的正交。

**InfiniBand 和 RDMA。** 完全超出范围。它们使用 `/usr/src/sys/ofed/` 和一个独立的 OS 无关 verb 框架。

**虚拟化专用加速。** `netmap(4)`、`vhost(4)` 和 DPDK 风格的用户空间快速路径存在且在 2026 年的生产环境中很重要。它们是职业生涯后期的主题。

我们没有完整涵盖这些中的任何一个。我们给你每个的指引，所以当你的工作需要其中之一时，你知道从哪里开始阅读。

## 历史背景：为什么 ifnet 看起来是这样

跨过桥之前的最后一站：一堂简短的历史课。了解 `ifnet` 从何而来，使其一些粗糙边缘不那么令人惊讶。

最早的 UNIX 网络协议栈在 1970 年代末根本没有 `ifnet` 结构。每个驱动程序通过零散的约定提供一组临时的回调。当 4.2BSD 在 1983 年引入套接字 API 和现代 TCP/IP 协议栈时，BSD 团队也引入了 `struct ifnet` 作为协议代码和驱动程序代码之间的统一接口。早期版本大约有十几个字段：名称、单元号、一组标志、一个输出回调和一些计数器。与现代 `struct ifnet` 相比，它看起来几乎是空的。

在接下来的四十年中，`struct ifnet` 不断增长。BPF 在 1980 年代末被添加。多播支持在 1990 年代初到达。IPv6 支持在 1990 年代末被添加。接口克隆、媒体层和链路状态事件在 2000 年代出现。卸载能力、VNET、校验和卸载标志、TSO、LRO 和 VLAN 卸载在 2010 年代出现。到 FreeBSD 11 在 2016 年发布时，该结构已经变得足够笨重，以至于项目引入了 `if_t` 不透明类型和 `if_get*`/`if_set*` 访问器函数，以便结构的布局可以在不破坏模块二进制兼容性的情况下改变。

这段历史解释了几件事。它解释了为什么 `ifnet` 同时有 `if_ioctl` 和 `if_ioctl2`；为什么有些字段通过宏访问而其他的直接访问；为什么 `IFF_*` 和 `IFCAP_*` 作为并行的标志空间存在；为什么克隆器 API 同时有 `if_clone_simple()` 和 `if_clone_advanced()`；为什么 `ether_ifattach()` 作为 `if_attach()` 的包装器存在。每个添加都解决了一个真实的问题。累积的重量是生活在一个从未有过干净重启机会的运行系统内部的代价。

对你来说，实际的收获是 ifnet 的表面积很大且有点不一致。把它当作地质学而不是建筑学来读。地层记录了 UNIX 网络历史中的真实事件。一旦你知道它们是地层，不一致就变得可导航了。

## 自我评估：你真的掌握了这些内容吗？

在继续之前，用具体的评分标准衡量自己的理解。网络驱动程序作者应该能够在不看本章的情况下回答以下每个问题。诚实地完成。如果你无法回答某个问题，重新阅读相关小节；不要只是浏览直到答案看起来熟悉。

**概念问题。**

1. `IFF_UP` 和 `IFF_DRV_RUNNING` 之间有什么区别，哪一个决定帧是否实际被发送？
2. 说出你的驱动程序必须提供的三个回调，对于每一个，描述未能正确实现它会导致什么后果。
3. 为什么内核为伪接口生成随机本地管理的 MAC 地址，必须设置什么位来标记地址为本地管理？
4. 当 `ether_input()` 接收到以太网帧时，`m_pkthdr` 的哪个字段告诉栈帧来自哪个接口，为什么每个入站 mbuf 都需要正确设置它？
5. `net_epoch` 保护什么，为什么它被认为比传统读锁更轻量？

**机械问题。**

6. 凭记忆写出从 `if_alloc()` 到 `ether_ifattach()` 创建最小可行接口的函数调用序列。你不需要记住参数列表；只需名称和顺序。
7. 写出将出站 mbuf 馈送给 BPF 的确切宏调用。写出入站的对应宏调用。
8. 给定一个可能被分片的 mbuf 链，哪个辅助函数给你一个适合 DMA 的单一平坦缓冲区？哪个辅助函数确保至少前 `n` 个字节是连续的？
9. `ifconfig mynet0 192.0.2.1/24` 产生哪个 ioctl？内核中哪一层实际处理它：通用 `ifioctl()` 分发器、`ether_ioctl()` 还是你的驱动程序的 `if_ioctl` 回调？为什么？
10. 你的驱动程序使用 `callout_init_mtx(&sc->tick, &sc->mtx, 0)`。互斥锁参数的目的是什么，如果传 `NULL` 会出现什么 bug？

**调试问题。**

11. `ifconfig mynet0 up` 瞬间返回，但 `ping` 十分钟后 `netstat -in` 显示接口有零个数据包。描述最可能的三个原因以及你会运行哪些命令来区分它们。
12. 模块干净地加载。`ifconfig mynet create` 成功。`ifconfig mynet0 destroy` 因"锁定断言"消息而崩溃。最可能的 bug 是什么？你会如何修复它？
13. `tcpdump -i mynet0` 显示出站数据包但从不说入站的，即使 `netstat -in` 显示 RX 计数器在递增。几乎可以确定缺少哪个函数调用，在哪个代码路径上？
14. 你在接口仍然存在的情况下运行 `kldunload mynet`。会发生什么？用户应该遵循什么安全序列？生产驱动程序如何在这些条件下拒绝卸载？
15. 循环运行 `ifconfig mynet0 up` 后立即 `ifconfig mynet0 down` 一百次，导致机器在第五十次迭代时因 mbuf 队列损坏而崩溃。分析可能的 bug 类别和修复方法。

**高级问题。**

16. 用你自己的话解释 `net_epoch` 提供了什么而互斥锁不能提供的，以及在网络驱动程序中何时使用其中一个而不是另一个。
17. 如果你的驱动程序声明 `IFCAP_VLAN_HWTAGGING`，与默认情况相比，这如何改变你的发送回调看到的 mbuf？
18. 内核有两条不同的入站帧传递路径：一条通过 `netisr_dispatch()`，一条通过直接分发。它们是什么，驱动程序何时需要关心使用哪一条？
19. `if_transmit` 和较老的 `if_start`/`if_output` 对之间有什么区别，新驱动程序应该使用哪个？
20. 描述 FreeBSD 14.3 系统上 VNET 的生命周期，并解释为什么通过 `VNET_SYSINIT` 注册的克隆器在每个 VNET 中产生一个克隆器而不是单个全局克隆器。

如果你毫不犹豫地回答了每个问题，你就为第 29 章做好了准备。如果五个或更多问题让你感到困惑，在继续之前再花一个会话时间学习本章。下一章建立在你已经牢固掌握这些材料的基础上。

## 延伸阅读和源码学习

下面的参考书目很小、有针对性，并按对你当前阶段驱动程序作者的有用程度排序。把它当作完成第 28 章后几周的阅读清单，而不是一个压倒性的书架。

**必读，按顺序。**

- `/usr/src/sys/net/if.c`：通用接口机制。从 `if_alloc()`、`if_attach()`、`if_detach()` 和 ioctl 分发器 `ifioctl()` 开始。这是实际运行你在驱动程序中调用的生命周期函数的文件。
- `/usr/src/sys/net/if_ethersubr.c`：以太网成帧。阅读 `ether_ifattach()`、`ether_ifdetach()`、`ether_output()`、`ether_input()` 和 `ether_ioctl()`。这四个函数构成了你的驱动程序与以太网层之间的契约。
- `/usr/src/sys/net/if_disc.c`：最小伪驱动程序。不到 200 行。绝对最小可行 `ifnet` 的参考。
- `/usr/src/sys/net/if_epair.c`：成对伪驱动程序。编写在两个实例之间共享结构的克隆器的最清晰参考。
- `/usr/src/sys/dev/virtio/network/if_vtnet.c`：现代半虚拟化驱动程序。足够小可以完整阅读，足够真实可以教你关于环、校验和卸载、多队列和类似硬件的资源管理。

**接下来阅读，当时机成熟时。**

- `/usr/src/sys/dev/e1000/if_em.c` 及附带的 `em_txrx.c`、`if_em.h`：生产级 Intel 网卡驱动程序。更大、更精细，但代表了真实世界的驱动程序复杂性。
- `/usr/src/sys/net/iflib.c` 和 `/usr/src/sys/net/iflib.h`：iflib 框架。在研究了 `if_em.c` 之后阅读，这样你可以识别 iflib 接管的那些结构。
- `/usr/src/sys/net/if_lagg.c`：链路聚合驱动程序。多接口编排、故障转移和模式选择的详细研究。
- `/usr/src/sys/net/if_bridge.c`：软件桥接。学习多播转发、学习型网桥和 STP 状态机的极佳材料。

**值得阅读的手册页。**

- `ifnet(9)`：接口框架。
- `mbuf(9)`：数据包缓冲区系统。
- `bpf(9)` 和 `bpf(4)`：Berkeley 包过滤器。
- `ifmedia(9)`：媒体框架。
- `ether(9)`：以太网辅助函数。
- `vnet(9)`：虚拟化网络栈。
- `net_epoch(9)`：网络 epoch 同步原语。
- `iflib(9)`：iflib 框架。
- `netmap(4)`：高速用户空间包 I/O。
- `netgraph(4)` 和 `netgraph(3)`：netgraph 框架。
- `if_clone(9)`：接口克隆。

**Books and papers.**

4.4BSD 设计书籍（特别是 McKusick、Bostic、Karels 和 Quarterman 的「4.4BSD 操作系统的设计与实现」）仍然是关于套接字和接口层如何产生的最佳长篇解释。FreeBSD 开发者手册中关于内核编程和可加载模块的章节是下一步的一般背景。对于高速数据包处理，Luigi Rizzo 的 `netmap` 论文是基础性的；它们解释了现代高性能数据包管道背后的技术和原理。

保持阅读日志。当你读完一个文件时，写一段话总结什么让你惊讶、你想重新看什么、以及你认为可能为自己的驱动程序偷用什么。经过六个月的这种练习，你对生产驱动程序结构的直觉增长速度将超出你的预期。

## 常见问题

新驱动程序作者在编写第一个 `ifnet` 驱动程序时倾向于问相同的问题。以下是最常见的问题，附有简短、有针对性的回答。每个回答都是一个路标，不是详尽的论述；如果你想要更多细节，沿着面包屑回到本章的相关小节。

**Q: 我可以不使用 `ether_ifattach` 编写以太网驱动程序吗？**

技术上可以；实际上不行。`ether_ifattach()` 将 `if_input` 设置为 `ether_input()`，用 `bpfattach()` 钩住 BPF，并配置十几个小的默认行为。跳过它意味着手动重新实现每一个默认值。绕过 `ether_ifattach()` 的唯一原因是如果你的驱动程序实际上不是以太网，那种情况下你会直接使用 `if_attach()` 并提供自己的成帧回调。

**Q: `if_transmit` 和 `if_output` 有什么区别？**

`if_output` 是较老的、协议无关的输出回调。对于以太网驱动程序，它被 `ether_ifattach()` 设置为 `ether_output()`，它在调用 `if_transmit` 之前处理 ARP 解析和以太网成帧。`if_transmit` 是你编写的驱动程序特定回调。简而言之：`if_output` 是协议栈调用的；`if_transmit` 是 `if_output` 调用的；你的驱动程序提供后者。

**Q: 我需要在 ioctl 回调中处理 `SIOCSIFADDR` 吗？**

不需要直接处理。`ether_ioctl()` 处理以太网接口的地址配置。你的回调应该通过 switch 语句的 `default:` 分支将未识别的 ioctl 委托给 `ether_ioctl()`，地址相关的 ioctl 将通过该路径正确流动。

**Q: 我如何知道帧何时真正被硬件发送了？**

对于我们的伪驱动程序，「发送」是同步的：`mynet_transmit()` 立即释放 mbuf。对于真正的网卡驱动程序，硬件通过中断或环描述符标志发出完成信号；驱动程序的发送完成处理程序（有时称为「tx reaper」）遍历环、释放 mbuf 并更新计数器。阅读 `if_em.c` 的 `em_txeof()` 获取具体示例。

**Q: 为什么 `ifconfig mynet0 delete` 不调用我的驱动程序？**

因为地址配置存在于协议层，而不是接口层。从以太网接口删除地址由 `in_control()`（对于 IPv4）或 `in6_control()`（对于 IPv6）处理。你的驱动程序不知道这些操作；它只通过路由变更和 ARP 表更新间接地感知它们。

**Q: 为什么我的驱动程序在从 callout 调用 `if_inc_counter()` 时会崩溃？**

几乎可以确定是因为你持有一个在其他地方获取的非递归互斥锁。`if_inc_counter()` 在现代 FreeBSD 上从任何上下文调用都是安全的，但如果你的 callout 获取了一个 callout 基础设施已经持有的锁，就会死锁。最安全的模式是在不持有任何驱动程序特定锁的情况下调用 `if_inc_counter()`，并在锁内单独更新你自己的计数器。

**Q: 如何让我的驱动程序出现在 `sysctl net.link.generic.ifdata.mynet0.link` 中？**

你不需要。该 sysctl 树由通用 `ifnet` 层自动填充。每个通过 `if_attach()`（直接或通过 `ether_ifattach()`）注册的接口都会获得一个 sysctl 节点。如果你的缺失了，说明你的接口没有正确附加。

**Q: 我的驱动程序在 FreeBSD 14.3 上工作但在 FreeBSD 13.x 上构建失败。为什么？**

`if_t` 不透明类型及关联的访问器函数在 FreeBSD 13 和 14 之间已稳定，但一些辅助 API 只在 14 中出现。例如，`if_clone_simple()` 已经存在多年，但一些计数器访问器辅助函数是新的。要么使用 `__FreeBSD_version` 守卫在两个版本上干净编译，要么在驱动程序中明确声明需要 FreeBSD 14.0 或更高版本。

**Q: 我想编写一个在一个接口上接收数据包并在另一个接口上重新发送的驱动程序。那是网络驱动程序吗？**

不完全是。那是一个桥接器或转发器。FreeBSD 内核有 `if_bridge(4)` 用于桥接，`netgraph(4)` 用于任意包管道，`pf(4)` 用于过滤和策略。在 2026 年从零开始编写自己的转发代码几乎从来不是正确的答案；现有的框架维护得更好、更快、更灵活。在编写新驱动程序之前，先阅读和配置它们。

**Q: 我需要担心网络驱动程序中的字节序吗？**

只在特定边界上。以太网帧按照约定是网络字节序（大端序）；如果你自己解析以太网头，`ether_type` 字段需要 `ntohs()`。在 mbuf 内部，数据以网络字节序存储，而不是主机本机字节序。`ether_input()` 和 `ether_output()` 函数为你处理转换，因此大多数驱动程序代码不直接接触字节序。

**Q: 我何时使用 `m_pullup()` 还是 `m_copydata()`？**

`m_pullup(m, n)` 改变 mbuf 链，使前 `n` 个字节在内存中连续存储，使得用指针类型转换安全访问它们。`m_copydata(m, off, len, buf)` 从 mbuf 链中复制字节到你提供的单独缓冲区。当你想就地读取并可能修改头字段时使用 `m_pullup()`。当你想获取一个快照用于检查而不扰动 mbuf 时使用 `m_copydata()`。

**Q: 为什么 `netstat -I mynet0 1` 有时即使数据包正在交换也显示零字节？**

你可能递增了 `IFCOUNTER_IPACKETS` 或 `IFCOUNTER_OPACKETS` 但没有同时递增 `IFCOUNTER_IBYTES` 或 `IFCOUNTER_OBYTES`。每秒显示单独显示字节；如果字节计数器从未移动，`netstat -I` 报告零吞吐量。始终同时更新包计数和字节计数。

**Q: 如何在模块卸载时销毁所有克隆接口？**

最简单的方法是让 `if_clone_detach()` 替你做；克隆分离辅助函数遍历克隆器的接口列表并逐个销毁。如果你想防止泄漏，也可以在调用 `if_clone_detach()` 之前显式枚举属于克隆器的接口并销毁它们。较短的路径通常更好，因为辅助函数经过了测试，而你的可能没有。

**Q: 我的驱动程序在 `ping` 下工作但在大型 `iperf3` 运行时崩溃。这通常是什么原因？**

在高包速率下，驱动程序中所有微妙的并发 bug 都会暴露出来。常见原因包括：在锁外更新在多个 CPU 上运行的计数器、释放前未正确排空的 mbuf 队列、在关机期间触发的 callout、接口分离后调用的 `BPF_MTAP()`。启用 WITNESS 和 INVARIANTS 运行；锁定断言几乎总能捕获它。

## 关于工艺的简短结语

我们在机制上花了很多页：回调、锁、mbuf、ioctl、计数器。机制是必要的，但还不够。一个好的网络驱动程序是一个有纪律的作者的产物，而不仅仅是一组正确的回调。

那种纪律体现在小地方。它体现在即使在测试套件从未捕获泄漏的情况下也决定在分离时排空 callout。它体现在决定以正确的顺序更新计数器，使 `netstat -s` 在长时间运行中数据一致。它体现在决定当资源无法分配时清晰记录一次日志，而不是保持沉默或淹没日志。它体现在决定在分配 softc 时使用 `M_ZERO`，这样即使忘记显式初始化，结构中未来添加的任何字段也从已知的零开始。

每个决定都是小的。累积的效果是一个在第一天有效的驱动程序和一个在第一千天有效的驱动程序之间的区别。你正在训练一种习惯，而不是记忆一种语法。在习惯形成期间对自己有耐心；这需要数年时间。

伟大的 FreeBSD 驱动程序作者，你在 `$FreeBSD$` 标签和提交日志中看到名字的那些人，不是因为他们比你更了解 API 而变得伟大的。他们之所以伟大是因为像审视别人的作品一样审视自己的作品，并修复他们发现的每一个小缺陷。这种实践是可以扩展的。尽早养成这个习惯。

## 网络驱动程序术语小词典

下面是一个简短的术语表，面向希望在一个地方回顾本章核心词汇的读者。把它当作复习参考，而不是正文解释的替代品。

- **ifnet.** 表示网络接口的内核数据结构。每个已附加的接口恰好有一个 `ifnet`。大多数现代代码使用不透明句柄 `if_t`。
- **ether_ifattach.** `if_attach()` 之上的包装器，设置以太网特定的默认值，包括 BPF 钩子和标准 `if_input` 函数。
- **克隆器 (cloner).** 伪接口的工厂。通过 `if_clone_simple()` 或 `if_clone_advanced()` 注册。负责响应 `ifconfig name create` 和 `ifconfig name0 destroy` 创建和销毁接口。
- **mbuf.** 内核的数据包缓冲区。一个带有元数据的小结构，可选的嵌入有效载荷，以及指向链式数据额外缓冲区的指针。用 `m_gethdr()` 分配，用 `m_freem()` 释放。
- **softc.** 每实例的驱动程序状态。在克隆器创建回调中用 `malloc(M_ZERO)` 分配，在克隆器销毁回调中释放。传统上指向互斥锁、媒体描述符、callout 和接口。
- **BPF.** Berkeley 包过滤器，一个让 `tcpdump` 等用户空间工具观察接口流量的框架。驱动程序在发送和接收路径上都通过 `BPF_MTAP()` 钩入它。
- **IFF_UP.** 由 `ifconfig name0 up` 设置的管理标志。指示用户激活接口的意图。
- **IFF_DRV_RUNNING.** 驱动程序控制的标志，指示驱动程序已准备好发送和接收数据包。在硬件（或伪硬件）初始化完成后在驱动程序内部设置。
- **媒体 (Media).** 链路速度、双工、自动协商和相关物理层属性的抽象。通过 `ifmedia(9)` 框架管理。
- **链路状态 (Link state).** 一个三值指示器（`LINK_STATE_UP`、`LINK_STATE_DOWN`、`LINK_STATE_UNKNOWN`），通过 `if_link_state_change()` 报告。由路由守护进程和用户空间工具使用。
- **VNET.** FreeBSD 的虚拟化网络栈。每个 VNET 有自己的接口列表、路由表和套接字。伪驱动程序通常使用 `VNET_SYSINIT` 在每个 VNET 中注册克隆器。
- **net_epoch.** 用于划分网络栈中读侧临界区的轻量级同步原语。比传统读锁更快。
- **IFCAP.** 驱动程序和协议栈之间协商的能力位字段（`IFCAP_RXCSUM`、`IFCAP_TSO4` 等）。控制给定接口上哪些卸载处于活动状态。
- **IFCOUNTER.** 由 `netstat` 显示的命名计数器（`IFCOUNTER_IPACKETS`、`IFCOUNTER_OBYTES` 等）。驱动程序通过 `if_inc_counter()` 更新。
- **以太网类型 (Ethernet type).** 以太网帧头中标识封装协议的 16 位字段。值定义在 `net/ethernet.h` 中，`ETHERTYPE_IP` 和 `ETHERTYPE_ARP` 是最常见的。
- **巨型帧 (Jumbo frame).** 大于标准 1500 字节 MTU 的以太网帧，通常为 9000 字节。驱动程序通过 `ifp->if_capabilities |= IFCAP_JUMBO_MTU` 声明支持。
- **混杂模式 (Promiscuous mode).** 接口将每个观察到的帧传递给协议栈的模式，不仅仅是那些寻址到自身 MAC 的帧。通过 `IFF_PROMISC` 控制。用于网络分析工具。
- **多播 (Multicast).** 寻址到一组接收者而不是单个目的地的帧。驱动程序通过 `SIOCADDMULTI` 和 `SIOCDELMULTI` 跟踪组成员资格，通常编程硬件哈希过滤器。
- **校验和卸载 (Checksum offload).** 网卡在硬件中计算 TCP、UDP 和 IP 头校验和的能力。通过 `IFCAP_RXCSUM` 和 `IFCAP_TXCSUM` 协商；通过 `m_pkthdr.csum_flags` 按 mbuf 标记。
- **TSO（TCP 分段卸载）.** 主机将大型 TCP 段交给网卡，网卡将其拆分为 MTU 大小的片段的能力。通过 `IFCAP_TSO4` 和 `IFCAP_TSO6` 协商。
- **LRO（大接收卸载）.** TSO 的接收侧对应。网卡或软件层在将顺序入站段交给协议栈之前将它们聚合成单个大型 mbuf 链。
- **VLAN 标记.** 以太网帧中标识 VLAN 成员资格的四字节插入。驱动程序可以声明 `IFCAP_VLAN_HWTAGGING` 以将插入和移除卸载到硬件。
- **MSI-X.** 消息信号中断，有线 IRQ 的现代替代。允许网卡按队列引发独立中断。
- **中断调节.** 网卡将多个完成事件合并为更少中断的技术，在高包速率下减少开销。
- **环形缓冲区 (Ring buffer).** 驱动程序和网卡之间共享的描述符循环队列。发送环将数据包馈送给硬件；接收环从硬件传递数据包。
- **iflib.** FreeBSD 的现代网卡驱动程序框架。抽象环管理、中断处理和 mbuf 流，使驱动程序作者可以专注于硬件特定代码。
- **netmap.** 高性能包 I/O 框架，给用户空间直接访问驱动环的权限，绕过大部分网络栈。
- **netgraph.** 用于从可重用节点组合包处理管道的灵活框架。大部分与驱动程序编写正交，但通常与网络架构相关。
- **pf.** FreeBSD 的包过滤器。一个防火墙和 NAT 引擎，通过 `pfil(9)` 钩子内联在 `ether_input()` 和 `ether_output()` 之间。驱动程序不直接与它交互；钩子由通用层插入。
- **pfil.** 包过滤器接口，防火墙通过它附加到转发路径。为 `pf` 和 `ipfw` 等框架提供观察和修改数据包的稳定位置。
- **if_transmit.** 每驱动程序的出站回调，在接口分配期间设置。接收 mbuf 链，负责将其排队到硬件或丢弃它。
- **if_input.** 每接口的入站回调。对于以太网驱动程序，由 `ether_ifattach()` 设置为 `ether_input()`。驱动程序通过 `if_input(ifp, m)` 辅助函数调用它，将接收到的帧向上传递给协议栈。
- **if_ioctl.** 每驱动程序的 ioctl 回调。处理接口级 ioctl，如 `SIOCSIFFLAGS`、`SIOCSIFMTU` 和 `SIOCSIFMEDIA`。将以太网驱动程序未知的 ioctl 委托给 `ether_ioctl()`。

在阅读第 29 章及后续章节时，请将本术语表放在手边。每个术语出现的频率足够高，快速参考的价值不言而喻。

## 第6部分检查点

第 6 部分将第 1 到第 5 部分的规范置于三种截然不同的传输方式之下：USB、基于 GEOM 的存储和基于 `ifnet` 的网络。在第 7 部分回到累积的 `myfirst` 主线并开始推进可移植性、安全性、性能和工艺之前，确认这三种传输方式的词汇已经沉淀为相同的底层模型。

到第 6 部分结束时，你应该能够做到以下每一项：

- 通过 `usb_request_methods` 框架附加到 USB 设备：为控制、批量、中断和等时端点配置传输；通过传输回调分发读写；并将热插拔和热拔作为正常操作条件来处理。
- 编写一个插入 GEOM 的存储驱动程序：通过 `g_new_providerf` 配置提供者，在类的 `start` 例程中服务 BIO 请求，在脑中走通 `g_down`/`g_up` 线程，并在挂载负载下干净地拆解。
- 编写一个通过 `ether_ifattach` 呈现 `ifnet` 的网络驱动程序：为出站路径实现 `if_transmit`，为入站路径调用 `if_input`，集成 `bpf` 和媒体状态，并通过 `ether_ifdetach` 清理。
- 解释为什么这三种传输方式在表面上看起来如此不同，但共享第 1 到第 5 部分中相同的底层规范：Newbus 附加、softc 管理、资源分配、锁定、拆解顺序、可观测性、生产规范。

如果其中任何一项仍然感觉不稳定，需要重新复习的实验是：

- USB 路径：第 26 章的实验 2（构建和加载 USB 驱动程序骨架）、实验 3（批量环回测试）、实验 6（观察热插拔生命周期）和实验 7（从零构建 ucom(4) 骨架）。
- GEOM 存储路径：第 27 章的实验 2（构建骨架驱动程序）、实验 3（实现 BIO 处理程序）、实验 4（增加大小并挂载 UFS）和实验 10（故意破坏）。
- 网络路径：第 28 章的实验 1（构建和加载骨架）、实验 2（练习发送路径）、实验 3（练习接收路径）、实验 5（`tcpdump` 和 BPF）和实验 6（干净分离）。

第 7 部分期望以下作为基线：

- 舒适地在 `cdevsw`、GEOM 和 `ifnet` 之间切换，将它们视为同一 Newbus-and-softc 核心之上的三种惯用语，而不是三个不相关的主题。
- 理解第 7 部分回到单线程 `myfirst` 主线进行可移植性、安全性、性能、追踪、内核调试器工作和与社区互动工艺的最后打磨。第 6 部分的传输特定演示不再继续；它们的教训会延续。
- 一个你亲手触摸过的三种真实传输方式的心理库，这样当第 29 章谈论跨后端抽象时，你是从经验中汲取，而不是从你只读过的例子中。

如果这些都成立，第 7 部分就为你准备好了。最后九章是本书将一个有能力的驱动程序作者转变为工匠的部分；第 1 到第 6 部分奠定的基础是使这种转变成为可能的东西。

## 展望：通往第29章的桥梁

你刚刚编写了一个网络驱动程序。下一章 **可移植性与驱动程序抽象** 从你已经掌握的具体细节中拉远，提出一个问题：我们如何编写在 FreeBSD 众多支持的架构上良好工作的驱动程序，以及我们如何构建驱动程序代码使其部分可以在不同硬件后端之间重用？

这个问题在第 28 章之后比之前更加尖锐。你现在已经为三个截然不同的子系统编写了驱动程序：基于 `cdevsw` 的字符设备、基于 GEOM 的存储设备和基于 `ifnet` 的网络设备。这三者在表面上看起来不同，但它们共享惊人的管道量：探测和附加、softc 分配、资源管理、生命周期控制、卸载干净度。第 29 章将把这一观察转化为实际的重构：隔离硬件相关代码，在公共 API 后面分离后端，准备驱动程序在 x86、ARM 和 RISC-V 上同样编译。

你不会在第 29 章中编写新类型的驱动程序。你将学习如何使你已经编写的驱动程序更健壮、更可移植、更可维护。这是一种不同类型的进步，当你开始开发一个将要存活多年的驱动程序时，这种进步就很重要了。

在继续之前，卸载你在本章中创建的每个模块，销毁每个接口，确保 `netstat -in` 回到无聊的基线。在实验日志本上简要记下什么有效、什么让你困惑来结束。让眼睛休息一分钟。然后，当你准备好时，翻到下一页。

你已经赢得了这一步。
