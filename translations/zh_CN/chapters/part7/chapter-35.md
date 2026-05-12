---
title: "异步 I/O 与事件处理"
description: "实现异步操作与事件驱动架构"
partNumber: 7
partName: "精通主题：特殊场景与边界情况"
chapter: 35
lastUpdated: "2026-04-20"
status: "complete"
author: "Edson Brandi"
reviewer: "TBD"
translator: "AI辅助翻译为简体中文"
language: "zh-CN"
estimatedReadTime: 135
---

# 异步 I/O 与事件处理

## 引言

到目前为止，我们编写的几乎每个驱动程序都运行在一个简单的调度之上。用户进程调用 `read(2)` 并等待。我们的驱动程序产生数据，内核将其复制出去，调用返回。用户进程调用 `write(2)` 并等待。我们的驱动程序消费数据，存储它，调用返回。用户线程在驱动程序工作期间休眠，工作完成后醒来。这就是同步模型，对于教授驱动程序来说，这是一个正确的起点，因为它匹配了普通函数调用的形状：你请求某物，你等待，你得到答案。

同步 I/O 适用于许多设备，但对其他设备则失效。键盘不会因为程序调用了 `read()` 就决定产生一次按键。串口的传入字节不会根据读取者的调度来计时。传感器可能以不规则的间隔产生数据，或者只有在物理世界中发生有趣的事情时才产生数据。如果我们坚持要求此类设备的每个用户必须阻塞在 `read()` 中直到下一个事件到达，我们就会强迫用户态程序陷入一个糟糕的选择。它要么为每个设备专门分配一个线程来阻塞，这使程序难以编写且对其他事件的响应变慢；要么在用户态循环中反复调用带有短超时的 `read()`，这浪费 CPU 周期，并且仍然会错过两次轮询之间发生的事件。

FreeBSD 通过给驱动程序提供一套异步通知机制来解决这个问题，每种机制都建立在同一个底层思想之上：进程不需要阻塞在 `read()` 中来获知数据已就绪。它可以改为注册对设备的兴趣，去做其他有用的工作，让内核在设备有事要说时唤醒它。这些机制在细节、性能特征和预期用例上各不相同，但它们共享一个共同的形状。等待者声明它在等待什么，驱动程序记录该兴趣，驱动程序稍后发现条件已满足，驱动程序发送通知，导致等待者被唤醒、调度或发出信号。

其中四种机制对驱动程序作者很重要。经典的 `poll(2)` 和 `select(2)` 系统调用让用户态程序询问内核一组文件描述符中哪些已就绪。较新的 `kqueue(2)` 框架提供了一个更高效、更富表现力的事件接口，是现代高性能应用程序的首选。通过 `FIOASYNC` 和 `fsetown()` 调用的 `SIGIO` 信号机制，每当设备状态改变时就向注册的进程发送信号。需要跟踪自身内部事件的驱动程序通常在 softc 内部构建一个小型事件队列，以便读者看到一致的可读记录序列，而不是原始硬件状态。

在本章中，我们将学习这些机制各自如何工作、如何在字符驱动程序中正确实现它们、如何组合它们以便单个驱动程序可以同时为 `poll(2)`、`kqueue(2)` 和 `SIGIO` 服务，以及如何审计产生的代码以发现异步编程的致命问题——微妙的竞争条件和唤醒丢失。我们将把每个部分建立在真实的 FreeBSD 14.3 源代码之上，看看 `if_tuntap.c`、`sys_pipe.c` 和 `evdev/cdev.c` 如何在生产中解决相同的问题。

到本章结束时，你将能够为阻塞驱动程序添加完整的异步支持而不破坏其同步语义。你将知道如何正确实现 `d_poll()`、`d_kqfilter()` 和 `FIOASYNC` 处理程序。你将理解为什么 `selrecord()` 和 `selwakeup()` 必须以特定顺序和特定锁定方式调用。你将知道什么是 `knlist`、`knote` 如何附加到它，以及为什么 `KNOTE_LOCKED()` 几乎是你在每个驱动程序中想要调用的函数。你将看到 `fsetown()` 和 `pgsigio()` 如何组合以向正确的进程发送信号。你将知道如何构建一个内部事件队列，将整个机制绑在一起，使每个异步通知引导读者找到驱动程序中单个、一致、定义良好的记录。

贯穿本章，我们将开发一个名为 `evdemo` 的配套驱动程序。它是一个模拟事件源的伪设备：时间戳、状态转换和用户态程序想要实时观察的偶尔"有趣"事件。本章的每一节都在 `evdemo` 上添加另一层，所以到最后你将拥有一个小而完整的异步驱动程序，你可以加载、检查和扩展。像上一章的 `bugdemo` 一样，`evdemo` 不接触真实硬件，所以每个实验都可以在 FreeBSD 开发虚拟机上安全运行。

## 读者指南：如何使用本章

本章位于本书第七部分，即精通主题部分，紧接在高级调试之后。它假设你至少已经编写了一个简单的字符驱动程序，知道如何安全地加载和卸载模块，并且已经使用过同步的 `read()`、`write()` 和 `ioctl()` 处理程序。如果这些感觉不确定，快速回顾第 8 章到第 12 章将在本章中多次回报。

你不需要完成之前每一章的精通章节才能跟上本章。已经掌握了基本字符驱动程序模式并接触过 `callout(9)` 或 `taskqueue(9)` 的读者将能够跟上。在前一章的材料必不可少的地方，我们会在相关章节给你简短的提醒。

本章材料是累积的。每一节向 `evdemo` 驱动程序添加新的异步机制，最后的重构将它们绑在一起。你可以浏览以了解某个特定机制，但实验最好按顺序阅读，因为后面的实验假设了前面实验的代码。

你不需要任何特殊硬件。一台适度的 FreeBSD 14.3 虚拟机足以完成本章的每个实验。串行控制台有用但不是必需的。你会想要打开第二个终端，这样你可以在驱动程序加载时观察 `dmesg`，运行用户空间测试程序，并在 `top(1)` 中监控等待通道。

合理的阅读进度如下。一次阅读前三节以构建 poll 和 select 的心智模型。休息一下。另一天阅读第 4 和第 5 节，因为 `kqueue` 和信号各自引入了一组新想法。按自己的节奏完成实验。本章故意很长：异步 I/O 是很多驱动程序复杂性所在的地方，匆忙通过材料是写出"大多数时间工作但在罕见情况下丢失唤醒"的驱动程序的最确定方式。

本章的一些代码故意做错事，以便我们可以看到常见错误的症状。这些示例被清楚标记。完成的实验做正确的事，最终重构的驱动程序可以安全加载。

## 如何从本章获得最大收益

本章重复你在每一节都会看到的模式。首先我们解释一个机制是什么以及它解决什么问题。然后我们展示用户态期望它如何表现，以便你理解驱动程序必须遵守的契约。然后我们查看真实的 FreeBSD 内核源代码，看看现有驱动程序如何实现该机制。最后我们将它应用到 `evdemo` 驱动程序的实验中。

几个习惯将帮助你吸收材料。

保持一个终端打开到 `/usr/src/`，这样你可以查阅本章引用的任何 FreeBSD 源文件。异步 I/O 是阅读真实驱动程序回报最大的领域之一，因为模式足够短，可以一次看完，驱动程序之间的变化教会你什么是本质的，什么是风格。当本章提到 `if_tuntap.c` 或 `sys_pipe.c` 时，打开文件查看。花一分钟阅读真实源代码比任何二手描述更能建立直觉。

保持第二个终端打开到你的 FreeBSD 虚拟机，这样你可以随着章节进展加载和卸载 `evdemo`。第一次看到代码时自己输入。`examples/part-07/ch35-async-io/` 下的配套文件包含完成的源代码，但输入代码建立了阅读无法建立的肌肉记忆。当一节引入新的回调时，添加到驱动程序，重新构建，重新加载，并在继续之前测试。

密切注意锁定。异步 I/O 是一个粗心的锁获取可以将干净的驱动程序变成死锁或静默数据损坏的地方。当本章显示在调用 `selrecord()` 或 `KNOTE_LOCKED()` 之前获取互斥锁时，注意顺序并问自己为什么必须这样。当实验说明说在修改事件队列之前获取 softc 互斥锁时，获取它。关于锁定的纪律是最可靠地区分工作的异步驱动程序和"大部分工作"的异步驱动程序的单个习惯。

最后，记住异步代码倾向于只在压力下揭示其错误。通过单线程测试的驱动程序仍然可能有在两个或三个线程竞争同一设备时显现的唤醒丢失或竞争。出于这个原因，本章的几个实验包括多读取者压力测试。不要跳过它们。在竞争下运行代码是证明它真正工作的最好方法。

带着这些习惯，让我们从同步和异步 I/O 之间的区别开始，以及何时每个是正确选择的问题。

## 1. 设备驱动程序中的同步与异步 I/O

同步 I/O 是到目前为止我们在几乎所有驱动程序中使用的模型。用户进程调用 `read(2)`。内核调度到我们的 `d_read` 回调。我们要么交出已经可用的数据，要么将调用线程置于条件变量上休眠直到数据到达。当数据准备好时，我们唤醒线程，它复制数据出去，`read(2)` 返回。用户程序在调用期间阻塞，然后恢复。

这个模式很容易推理。它匹配普通函数的工作方式：你调用，你等待，你得到结果。对于调用者需求驱动设备工作的设备，这也是非常合适的。磁盘读取器请求数据，磁盘控制器被指示去获取它。具有 `read_current_value` 操作的传感器自然适合同步调用。对于这些设备，用户进程总是知道何时询问，等待的成本就是实际 I/O 的成本。

但对于许多真实设备，驱动程序的工作不是由调用者的需求驱动的，而是由世界驱动的。

### 世界不等待 read()

考虑一个键盘。当按键被按下时，设备对谁在调用 `read(2)` 没有意见。用户按下键，中断触发，驱动程序从硬件中拉出扫描码，数据现在可用。如果用户态程序阻塞在 `read()` 中，它醒来并获得按键。如果没有程序在读取，按键停留在缓冲区中。如果多个程序共享对键盘的兴趣，在经典阻塞语义下只有一个接收按键，这几乎从来不是程序员想要的。

考虑一个串口。字节以线路的速度到达，独立于任何程序的接收准备情况。如果驱动程序将每个传入字节阻塞在读取者后面，它实际上强制读取者保持一个线程始终休眠在 `read()` 中，以防发生某事。那个线程不能做任何其他事情。一个设计良好的单个进程可能想要同时响应多个串口、网络套接字、定时器和键盘。同步模型无法表达这一点。

考虑一个 USB 传感器，只有当测量量越过阈值时才报告值。温度传感器可能只在温度变化超过半度时引发事件。运动传感器可能只在检测到运动时触发。设备自己的调度，而不是用户态的调度，决定数据何时准备好。阻塞在 `read()` 中的读取者可能等待毫秒、秒、分钟或永远。

这些情况每一个都共享一个属性：事件对程序的请求是外部的。驱动程序知道数据何时准备好。用户态不知道。如果用户态每次都要阻塞在 `read()` 中来了解驱动程序所知道的，程序就被驱动程序的节奏挟持了。

### 为什么忙等待是一个糟糕的答案

一个天真的解决方案是让用户态程序轮询驱动程序。不是调用一次 `read()` 并阻塞，而是以非阻塞模式反复调用 `read()`。用 `O_NONBLOCK` 打开 `/dev/...` 如果没有数据可用则立即返回。程序可以在循环中旋转，调用 `read()`，做其他工作，再次调用 `read()`，以此类推。

这个模式称为忙等待，它几乎总是错误的。即使什么也没发生，它也消耗 CPU，因为程序不断询问驱动程序是否有工作。它错过轮询之间发生的事件。它给每个事件增加延迟：在上次轮询后一百微秒按下的键要等到下次轮询才能被看到。而且它扩展性差：观察十个这样的设备的程序必须在每次迭代中轮询所有十个，使所有问题更糟。

忙等待在确切一种情况下是合适的：当轮询频率已知、设备延迟以微秒衡量、程序没有其他工作时。即使在那样的情况下，正确答案通常是在轮询之间使用 CPU 的高精度定时设施和 `usleep()` 而不是旋转。对于任何其他情况，忙等待是错误的工具。

同步阻塞模型和忙等待模型是谱系的两个端点。两者都浪费资源。我们想要的是第三个选项：用户态要求内核在设备准备好时告诉它，然后做其他工作直到内核举手。第三个选项就是异步 I/O 提供的。

### 异步 I/O 不仅仅是非阻塞读取

一个常见的初学者错误是认为异步 I/O 意味着用 `O_NONBLOCK` 调用 `read()`。并不是。非阻塞 `read()` 在数据不可用时立即返回；这是一个有用的属性，但它本身不是异步 I/O。没有通知机制的非阻塞 `read()` 只是稍加修饰的忙等待。

本章使用的术语中的异步 I/O 是驱动程序和用户态之间的通知协议。用户态不需要正在读取来了解驱动程序有数据。驱动程序不需要猜测谁感兴趣。当驱动程序状态以相关方式改变时，它通过定义良好的机制通知其等待者：`poll`/`select`、`kqueue`、`SIGIO` 或它们的某种组合。等待者醒来，读取数据，然后回去等待。

这个区别很重要，因为它分离了驱动程序中三个独立的关注点：

第一个关注点是等待注册。用户态程序通过调用 `poll()`、`kevent()` 或启用 `FIOASYNC` 来声明对设备的兴趣。驱动程序记住该注册，以便它稍后能找到等待者。

第二个关注点是唤醒传递。当驱动程序状态改变时，它调用 `selwakeup()`、`KNOTE_LOCKED()` 或 `pgsigio()` 来传递通知。这是一个与产生数据分离的操作。驱动程序可以在没有人注册时产生数据（例如，发生在任何人注册之前的初始填充期间）。驱动程序可以在不产生数据的情况下传递通知（例如，当设备挂起时）。如果注册了多个机制，驱动程序可以为一个数据单元传递多个通知。

第三个关注点是事件所有权。`SIGIO` 信号被传递给特定进程或进程组。`knote` 属于特定的 `kqueue`。`select()` 等待者属于特定线程。如果驱动程序无法将唤醒匹配到正确的所有者，通知就会丢失或传递给错误的一方。每个机制都有自己的将通知匹配到所有者的规则，我们必须为每一个分别正确理解这些规则。

保持这三个关注点清晰是本章的主要主题之一。异步驱动程序中的许多微妙错误来自将它们混淆。如果你发现自己在想为什么存在特定的唤醒调用或为什么持有特定的锁，十次中有九次答案在于保持注册、传递和所有权分离。

### 真实世界模式：想要异步 I/O 的事件源

命名异步 I/O 是正确选择的模式很有帮助，因为一旦你认出它们，你就会到处看到它们。

字符输入设备是经典情况。键盘、鼠标、触摸屏、操纵杆：每一个都在用户与之交互时产生事件，速率没人能提前预测。用户可能现在按键，或五分钟后。驱动程序知道事件何时到达。用户态需要一种方式来了解。

串行和网络接口是另一种情况。字节以线路的速度从线路到达。终端模拟器不想阻塞等待下一个字节，因为它还必须重绘屏幕、响应键盘输入并更新光标。网络程序不想阻塞等待下一个数据包，因为它通常必须同时观察多个套接字。

按条件报告的传感器是第三种情况。报告"按下"或"释放"的按钮。当测量值越过阈值时触发的温度传感器。运动检测器。门触点。所有这些都是严格意义上的事件驱动：直到世界发生有趣的事情之前什么也不会发生。

控制线和调制解调器信号是第四种情况。串口上的 `CARRIER`、`DSR` 和 `RTS` 线独立于数据流改变状态。关心它们的程序希望被告知它们何时改变，而不是连续轮询它们。

将多种事件组合到一个流中的任何设备是第五种情况。考虑一个将击键、鼠标移动和触摸屏事件聚合为统一事件流的 `evdev` 输入设备。驱动程序构建事件的内部队列，每个有趣的事物一条记录，读取者从队列中提取事件。我们稍后会在本章构建这个模式的一个小版本，因为它说明了事件队列、异步通知和同步 `read()` 语义如何组合成一个结构良好的驱动程序。

### 何时不使用异步 I/O

为了平衡，让我们命名一些异步 I/O 不是正确答案的情况。

唯一操作是按调用者请求进行批量传输的驱动程序没有理由暴露 `poll()` 或 `kqueue()`。如果每次交互都是用户发起的往返，同步阻塞模型既简单又正确。向这样的驱动程序添加异步通知只会增加复杂性。

数据速率高到任何通知开销都很重要的驱动程序可能需要完全不同的方法。`netmap(4)` 和类似的内核旁路框架正是为这种情况存在的，它们远远超出了本章的范围。普通的基于 `kqueue()` 的设计可以很好地处理每秒数百万个事件，但在某一点上，任何通知机制的成本都会成为瓶颈。

消费者是另一个内核子系统而不是用户态程序的驱动程序通常根本不需要面向用户态的异步通知。它需要内核内同步：互斥锁、条件变量、`callout(9)`、`taskqueue(9)`。这些是我们在前面章节研究的模式，当事件的两边都在内核内时，它们仍然是正确答案。

对于介于两者之间的一切，异步 I/O 是正确的工具，正确学习它是驱动程序作者可以获得的最持久的技能之一。接下来的三节构建心智模型和代码：首先是 `poll()` 和 `select()`，然后是 `selrecord()` 和 `selwakeup()`，然后是 `kqueue()`。后面的章节添加信号、事件队列和组合设计。

### 本章其余部分的心智模型

在我们继续之前，让我们固定一个将指导本章其余部分的心智模型。每个异步驱动程序有三类代码路径。

第一类是生产者路径。这是驱动程序了解到发生了某事的地方。对于硬件，它是中断处理程序。对于像 `evdemo` 这样的伪设备，它是任何模拟事件的代码。生产者的工作是更新驱动程序的内部状态，以便现在查看的读取者能看到新事件。

第二类是等待者路径。这是用户态调用者注册兴趣的地方。调用者的线程通过系统调用（`poll`、`select`、`kevent` 或 `ioctl(FIOASYNC)`）进入内核，内核调度到我们的 `d_poll` 或 `d_kqfilter` 回调，我们以生产者稍后能找到的方式记录调用者的兴趣。

第三类是传递路径。这是生产者通知等待者的地方。生产者刚刚更新了状态。它调用 `selwakeup()`、`KNOTE_LOCKED()`、`pgsigio()` 或它们的某种组合，这些调用唤醒等待的线程，然后通常调用 `read()` 来获取实际数据。

这个三路径模型是我们将接近每个机制的框架。当我们研究 `poll()` 时，我们会问：生产者在做什么，等待者注册什么，传递看起来像什么？当我们研究 `kqueue()` 时，我们会问同样的三个问题。当我们研究 `SIGIO` 时，同样的三个问题。机制在细节上各不相同，但它们都适合相同的形状，知道形状使每一个更容易学习。

建立心智模型后，让我们看看 `poll(2)` 和 `select(2)`，这三种机制中最古老和最可移植的。

## 2. 介绍 poll() 和 select()

`poll(2)` 和 `select(2)` 系统调用是 UNIX 对"我如何一次等待多个文件描述符？"这个问题的原始答案。它们已经在 UNIX 中存在了几十年，它们在每个重要平台上工作，并且它们仍然是用户态程序在一个循环中监视多个设备、套接字或管道的最可移植方式。

它们共享相同的底层抽象。程序传递一组文件描述符和它关心的事件掩码：可读、可写或异常。内核检查每个描述符，询问其驱动程序或子系统事件是否就绪，如果没有，则将调用线程置于休眠，直到有一个变为就绪或超时过期。当它醒来时，内核返回哪些描述符现在是活动的，程序可以服务它们。

从驱动程序的角度来看，`poll` 和 `select` 都汇入 `cdev` 上相同的 `d_poll` 回调。用户态程序使用的是 `poll(2)` 还是 `select(2)` 对驱动程序来说是不可见的。我们回答一个问题：给定调用者感兴趣的事件集，其中哪些现在就绪？如果没有就绪，我们也注册调用者，以便在某个东西变为就绪时我们可以唤醒它。

那个双重角色（现在回答，稍后注册）是 `d_poll` 契约的核心。驱动程序必须立即回答当前状态，如果答案是"没有"就不能忘记等待者。任何一半做错都会产生两个经典的 poll 错误。如果驱动程序在数据实际就绪时报告"未就绪"，调用者进入休眠且永不醒来，因为没有进一步事件会触发唤醒。如果驱动程序在什么都没就绪时未能注册等待者，调用者也永不醒来，因为驱动程序永远不知道在数据最终到达时唤醒谁。两个错误产生相同的症状（挂起的进程）并且都是未能实现正确模式的后果。

### 用户态期望 poll() 和 select() 做什么

在我们实现 `d_poll` 之前，了解用户态调用者在做什么很有帮助。用户代码通常看起来像这样：

```c
#include <poll.h>
#include <fcntl.h>
#include <unistd.h>

struct pollfd pfd[1];
int fd = open("/dev/evdemo", O_RDONLY);

pfd[0].fd = fd;
pfd[0].events = POLLIN;
pfd[0].revents = 0;

int r = poll(pfd, 1, 5000);   /* wait up to 5 seconds */
if (r > 0 && (pfd[0].revents & POLLIN)) {
    /* data is ready; do a read() now */
    char buf[64];
    ssize_t n = read(fd, buf, sizeof(buf));
    /* ... */
}
```

用户传递一个 `struct pollfd` 数组，每个都有一个指示它关心什么的 `events` 掩码。内核通过写入实际就绪事件的 `revents` 字段返回。第三个参数是以毫秒为单位的超时，`-1` 意味着"永远等待"，`0` 意味着"完全不阻塞，只是轮询状态"。

`select(2)` 用稍微不同的 API 做同样的事情：三个 `fd_set` 位图用于可读、可写和异常描述符，以及一个作为 `struct timeval` 的超时。在内核内部，两个调用都标准化为对每个涉及描述符的相同操作，最终到达我们的 `d_poll` 回调。

调用者期望这些语义：

如果任何请求的事件当前就绪，调用必须迅速返回并设置就绪事件。

如果没有请求的事件就绪且超时未过期，调用必须阻塞直到一个事件变为就绪或超时触发。

如果描述符在调用期间关闭或变为无效，内核返回 `POLLNVAL`、`POLLHUP` 或 `POLLERR`（视情况而定）。

驱动程序通常处理的事件掩码位是：

`POLLIN` 和 `POLLRDNORM`，两者都意味着"有数据可读"。FreeBSD 将 `POLLRDNORM` 定义为与 `POLLIN` 不同，但在大多数驱动程序代码中我们将它们一起处理，因为程序通常请求其中一个或另一个，并期望任何一个都能工作。

`POLLOUT` 和 `POLLWRNORM`，两者都意味着"设备有缓冲空间接受写入"。FreeBSD 将 `POLLWRNORM` 定义为与 `POLLOUT` 相同，所以在实践中它们是同一位。

`POLLPRI`，意味着"带外或优先数据可用"。大多数字符驱动程序没有优先级概念，将其留作不理。

`POLLERR`，意味着"设备上发生了错误"。当出现问题且设备无法恢复时，驱动程序通常设置此位。

`POLLHUP`，意味着"对端已挂断"。当从机关闭时，pty 主端会看到这个。当写入者关闭时，管道读取者会看到它。设备驱动程序通常在分离路径期间，或当分层服务断开连接时设置此位。

`POLLNVAL`，意味着"请求无效"。驱动程序通常将此位留给内核框架，当描述符无效或驱动程序没有 `d_poll` 时框架会设置它。

`POLLHUP` 和 `POLLIN` 的组合值得注意：当设备关闭且它有缓冲数据时，读取者应该看到 `POLLHUP` 和 `POLLIN`，因为缓冲数据仍然可以读取，即使不再有数据传来。编写良好的用户态程序显式处理这种情况。

### d_poll 回调

现在我们可以看看 `d_poll` 回调本身。它的签名定义在 `/usr/src/sys/sys/conf.h` 中：

```c
typedef int d_poll_t(struct cdev *dev, int events, struct thread *td);
```

`dev` 参数是我们的 `cdev`，我们从中通过 `dev->si_drv1` 获取 softc。`events` 参数是调用者感兴趣的事件掩码。`td` 参数是调用线程，我们需要将它传递给 `selrecord()` 以便内核可以将未来的唤醒匹配到正确的等待者。返回值是现在就绪的 `events` 子集。

一个骨架实现如下：

```c
static int
evdemo_poll(struct cdev *dev, int events, struct thread *td)
{
    struct evdemo_softc *sc = dev->si_drv1;
    int revents = 0;

    mtx_lock(&sc->sc_mtx);

    if (events & (POLLIN | POLLRDNORM)) {
        if (evdemo_event_ready(sc))
            revents |= events & (POLLIN | POLLRDNORM);
        else
            selrecord(td, &sc->sc_rsel);
    }

    if (events & (POLLOUT | POLLWRNORM))
        revents |= events & (POLLOUT | POLLWRNORM);

    mtx_unlock(&sc->sc_mtx);
    return (revents);
}
```

这是经典模式。让我们逐行遍历它。

我们获取 softc 互斥锁，因为我们即将查看驱动程序的内部状态，在我们决定事件是否就绪时，没有其他线程应该修改它。在调用 `selrecord()` 时持有锁也关闭了答案和注册之间的竞争，我们稍后会看到。

我们查看调用者关心的每个事件类型。对于可读事件，我们询问驱动程序是否有任何数据就绪。如果是，我们将匹配的位添加到 `revents`。如果不是，我们调用 `selrecord()` 将此线程注册为 `sc_rsel` selinfo 上的等待者。那个 selinfo 存在于 softc 中，在所有潜在等待者之间共享，是我们稍后在数据到达时将传递给 `selwakeup()` 的东西。

对于可写事件，在这个例子中我们没有一个可以填满的内部缓冲区，所以我们总是报告设备为可写。许多驱动程序属于这个类别：写入总是适合。具有有界缓冲区的驱动程序应该以检查可读状态的相同方式检查缓冲区状态，并且只在有空间时报告 `POLLOUT`。

我们释放锁并返回就绪事件掩码。

关于这个模式有三件事值得强调。

首先，我们在每种情况下都立即返回 `revents`。`d_poll` 回调不休眠。如果什么都没就绪，我们注册一个等待者并返回零。内核的通用 poll 框架负责实际阻塞：在 `d_poll` 返回后，如果没有文件描述符返回任何事件，内核原子地将线程休眠。驱动程序作者看不到这个休眠；它完全由内核中的 poll 调度逻辑处理。

其次，我们必须只为当前未就绪的事件类型调用 `selrecord()`。如果事件已就绪并且我们也调用 `selrecord()`，我们不会破坏任何东西（框架处理这种情况），但这是浪费：线程不会休眠，所以注册它是毫无意义的。"检查，如果未就绪则注册"模式保持工作量成比例。

第三，我们在检查期间持有的锁与在生产者路径中调用 `selwakeup()` 时将获取的锁相同。这就是阻止唤醒丢失竞争的原因：如果生产者在我们检查状态后但在注册等待者之前触发，生产者无法传递唤醒直到我们的 `selrecord()` 完成，所以唤醒会找到我们。我们将在第 3 节详细讨论这个问题。

### 在 cdevsw 上注册 d_poll 方法

为了让我们的驱动程序响应 `poll()` 调用，我们填充传递给 `make_dev()` 或 `make_dev_s()` 的 `struct cdevsw` 的 `d_poll` 字段：

```c
static struct cdevsw evdemo_cdevsw = {
    .d_version = D_VERSION,
    .d_name    = "evdemo",
    .d_open    = evdemo_open,
    .d_close   = evdemo_close,
    .d_read    = evdemo_read,
    .d_write   = evdemo_write,
    .d_ioctl   = evdemo_ioctl,
    .d_poll    = evdemo_poll,
};
```

如果我们不设置 `d_poll`，内核提供默认值。在 `/usr/src/sys/kern/kern_conf.c` 中，默认是 `no_poll`，它调用 `poll_no_poll()`。该默认值返回标准的可读和可写位，除非调用者请求了任何异乎寻常的东西，在这种情况下它返回 `POLLNVAL`。这种行为对于像 `/dev/null` 和 `/dev/zero` 这样总是就绪的设备是有意义的，但对于事件驱动的设备几乎从来不是你想要的。对于任何具有真正异步语义的驱动程序，你需要自己实现 `d_poll`。

### 真实驱动程序是什么样的

让我们看两个真实的实现，因为当你看到生产代码中的模式时，它会变得更清晰。

打开 `/usr/src/sys/net/if_tuntap.c` 并找到函数 `tunpoll`。它足够短可以引用：

```c
static int
tunpoll(struct cdev *dev, int events, struct thread *td)
{
    struct tuntap_softc *tp = dev->si_drv1;
    struct ifnet    *ifp = TUN2IFP(tp);
    int     revents = 0;

    if (events & (POLLIN | POLLRDNORM)) {
        IFQ_LOCK(&ifp->if_snd);
        if (!IFQ_IS_EMPTY(&ifp->if_snd)) {
            revents |= events & (POLLIN | POLLRDNORM);
        } else {
            selrecord(td, &tp->tun_rsel);
        }
        IFQ_UNLOCK(&ifp->if_snd);
    }
    revents |= events & (POLLOUT | POLLWRNORM);
    return (revents);
}
```

这几乎是我们骨架的逐字复制，以 `tun` 驱动程序的传出数据包队列作为数据源，以 `tun_rsel` selinfo 作为等待点。这里的锁是 `IFQ_LOCK`，队列锁，生产者在修改队列和调用 `selwakeuppri()` 之前也获取它。那种匹配的锁定是设计正确的关键。

现在打开 `/usr/src/sys/dev/evdev/cdev.c` 并找到 `evdev_poll`。这是一个稍长且更有指导意义的例子，因为它显式处理了撤销的设备：

```c
static int
evdev_poll(struct cdev *dev, int events, struct thread *td)
{
    struct evdev_client *client;
    int ret;
    int revents = 0;

    ret = devfs_get_cdevpriv((void **)&client);
    if (ret != 0)
        return (POLLNVAL);

    if (client->ec_revoked)
        return (POLLHUP);

    if (events & (POLLIN | POLLRDNORM)) {
        EVDEV_CLIENT_LOCKQ(client);
        if (!EVDEV_CLIENT_EMPTYQ(client))
            revents = events & (POLLIN | POLLRDNORM);
        else {
            client->ec_selected = true;
            selrecord(td, &client->ec_selp);
        }
        EVDEV_CLIENT_UNLOCKQ(client);
    }
    return (revents);
}
```

注意我们在骨架中没有的两个额外行为片段。

当客户端已被撤销（当设备正在分离而客户端仍然打开文件描述符时发生），函数返回 `POLLHUP` 以便用户态程序知道放弃。这是分离情况的正确处理。我们的骨架还没有做到这一点，但最终重构的 `evdemo` 会做。

驱动程序设置一个标志 `ec_selected` 来记住已注册等待者。这让生产者避免为从未轮询过的客户端调用 `selwakeup()`，这是一个小优化。大多数驱动程序跳过这个优化，每次都调用 `selwakeup()`，这更简单且仍然正确。

### 用户看到什么

在用户态方面，调用者不关心我们选择了哪种实现。它调用带有超时的 `poll()` 并查看结果。第一次调用如果什么都没就绪且超时过期则返回零，否则返回就绪描述符的正数。第二次调用查看 `revents` 位掩码并分发到正确的处理。

这就是异步 I/O 实现的干净分离。用户程序不知道或不关心 `selinfo` 或 `knlist`。它只知道它问了内核"这个准备好了吗？"并得到了答案。驱动程序的工作是使那个答案真实，并确保下一个相关事件会唤醒等待者。

### 第 2 节总结

我们现在有了 poll 和 select 的用户态视图、`d_poll` 的内核签名，以及一个注册等待者并报告可读事件的第一个骨架实现。但骨架仍然不完整。我们使用了 `selrecord()` 却没有解释它真正对 `struct selinfo` 做了什么，我们还没有看到匹配的产生通知的 `selwakeup()` 调用。这是下一节的主题，也是基于 poll 的异步 I/O 的微妙正确性问题所在。


## 3. 使用 selwakeup() 和 selrecord()

`selrecord()` 和 `selwakeup()` 是经典 poll-wait 协议的两半。它们从 4.2BSD 中 `select(2)` 的原始引入以来就一直在 BSD 内核中，并且仍然是在 FreeBSD 驱动程序中为 `poll(2)` 和 `select(2)` 实现等待/唤醒的规范方式。这一对在概述上很简单但在细节上很微妙，基于 poll 的驱动程序中最有趣的大多数错误都来自把微妙之处弄错。

本节带你逐步了解 selinfo 机制。首先我们看看 `struct selinfo` 实际包含什么。然后我们看看 `selrecord()` 实际做什么和不做什么。然后我们看 `selwakeup()` 及其同伴。最后我们检查经典的唤醒丢失竞争、防止它的锁定纪律，以及你可以用来确认驱动程序做对了的诊断技术。

### struct selinfo

打开 `/usr/src/sys/sys/selinfo.h` 并看定义：

```c
struct selinfo {
    struct selfdlist    si_tdlist;  /* List of sleeping threads. */
    struct knlist       si_note;    /* kernel note list */
    struct mtx          *si_mtx;    /* Lock for tdlist. */
};

#define SEL_WAITING(si)    (!TAILQ_EMPTY(&(si)->si_tdlist))
```

只有三个字段。`si_tdlist` 是当前在此 selinfo 上休眠的线程列表，因为它们调用了 `selrecord()` 并且它们的 `poll()` 或 `select()` 调用决定阻塞。`si_note` 是一个 `knlist`，我们将在第 4 节实现 `kqueue` 支持时遇到；它允许同一个 selinfo 同时为 `poll()` 和 `kqueue()` 等待者服务。`si_mtx` 是保护列表的锁。

`SEL_WAITING()` 宏告诉你当前是否有任何线程停在此 selinfo 上。驱动程序偶尔用它来决定是否值得调用 `selwakeup()`，尽管唤醒例程本身足够便宜，测试通常是不必要的。

关于 `struct selinfo` 有两个重要习惯：

首先，驱动程序必须在首次使用前将 selinfo 零初始化。通常的方式是将它嵌入一个通过 `malloc(..., M_ZERO)` 零化的 softc 中，但如果你单独分配一个 selinfo，你必须用 `bzero()` 或等价物将它零化。未初始化的 selinfo 会在首次调用 `selrecord()` 时崩溃内核。

其次，驱动程序必须在销毁 selinfo 之前排空其等待者。分离时的规范序列是 `seldrain(&sc->sc_rsel)` 后跟 `knlist_destroy(&sc->sc_rsel.si_note)`。`seldrain()` 调用唤醒任何当前停放的等待者，以便它们看到描述符已变为无效而不是永远阻塞。`knlist_destroy()` 调用清理 kqueue 等待者的 knote 列表，我们将在下一节实现。

### selrecord() 做什么

当驱动程序决定当前事件未就绪且线程需要等待时，从 `d_poll` 调用 `selrecord()`。其签名：

```c
void selrecord(struct thread *selector, struct selinfo *sip);
```

实现位于 `/usr/src/sys/kern/sys_generic.c`。其本质足够短可以总结：

1. 函数检查线程处于有效的 poll 上下文。
2. 它获取附加到线程的 `seltd` 结构的预分配 `selfd` 描述符之一。
3. 它将该描述符链接到线程的活动等待列表和 `selinfo` 的 `si_tdlist`。
4. 它在描述符上记住 selinfo 的互斥锁，以便唤醒路径知道要获取哪个锁。

要理解的关键是 `selrecord()` 不做什么。它不休眠线程。它不阻塞。它不将线程转换到任何阻塞状态。它只是记录这个线程对这个 selinfo 有兴趣的事实，以便稍后，当内核的 poll 调度代码决定阻塞线程（如果没有描述符返回任何事件）时，它知道线程停在哪里。

在所有线程的 `d_poll` 回调返回后，poll 调度代码查看结果。如果有任何文件描述符返回事件，调用立即返回而不阻塞。如果没有，线程进入休眠。休眠是在 `struct seltd` 内的每个线程条件变量上，唤醒通过该条件变量传递。selinfo 的作用是将线程的 `seltd` 链接到所有相关驱动程序，以便每个驱动程序稍后能找到线程。

这种"记录"和"休眠"的分离就是让单个 `poll()` 调用能监视许多文件描述符的原因。线程注册到它关心的每个驱动程序的每个 selinfo。当它们中的任何一个触发时，唤醒通过其 `seltd` 找到线程，并回到 poll 调度，然后查看所有注册的文件描述符以查看哪些就绪。

### selwakeup() 做什么

当驱动程序状态以可能满足等待者的方式改变时，从生产者路径调用 `selwakeup()`。其签名：

```c
void selwakeup(struct selinfo *sip);
```

还有一个称为 `selwakeuppri()` 的变体，它接受一个优先级参数，当驱动程序想要控制被唤醒线程恢复时的优先级时很有用。在实践中，`selwakeup()` 对几乎每个驱动程序都很好；`selwakeuppri()` 用于少数想要以牺牲公平性为代价强调延迟的子系统。

实现遍历 selinfo 的 `si_tdlist` 并向每个停放的线程的条件变量发送信号。它还遍历 selinfo 的 `si_note` 列表并向附加在那里的任何 knote 传递 kqueue 样式的通知，所以单个 `selwakeup()` 调用同时服务于 poll 等待者和 kqueue 等待者。

关键的是，`selwakeup()` 必须在驱动程序更新内部状态以反映新事件后调用。如果在数据可见之前调用 `selwakeup()`，被唤醒的线程再次通过 `d_poll` 运行，看到什么都没就绪（因为生产者还没有使其可见），重新注册，然后休眠。当生产者最终更新状态时，没有人被唤醒，因为重新注册发生在唤醒之后。驱动程序然后必须等待下一个事件来解锁等待者，这可能永远不会来。

正确的顺序总是：先更新状态，再唤醒。绝不反过来。

### 唤醒丢失竞争

基于 poll 的驱动程序中最著名的错误是唤醒丢失。它看起来像这样：

```c
/* 生产者线程 */
append_event(sc, ev);              /* 更新状态 */
selwakeup(&sc->sc_rsel);           /* 唤醒等待者 */

/* 消费者线程，在 d_poll 中 */
if (events & POLLIN) {
    if (event_ready(sc))
        revents |= POLLIN;
    else
        selrecord(td, &sc->sc_rsel);
}
return (revents);
```

如果生产者运行在消费者的 `event_ready()` 检查和消费者的 `selrecord()` 调用之间，唤醒就会丢失。消费者看到没有事件，生产者发布了一个事件并在空的等待者列表上调用 `selwakeup()`，然后消费者注册。没有人会再次调用 `selwakeup()` 直到下一个事件到达。消费者现在休眠直到下一个事件，即使事件已经就绪。

这就是检查和注册之间的经典 TOCTOU 竞争。标准的修复是使用单个互斥锁来序列化检查、注册和唤醒：

```c
/* 生产者线程 */
mtx_lock(&sc->sc_mtx);
append_event(sc, ev);
mtx_unlock(&sc->sc_mtx);
selwakeup(&sc->sc_rsel);

/* 消费者线程，在 d_poll 中 */
mtx_lock(&sc->sc_mtx);
if (events & POLLIN) {
    if (event_ready(sc))
        revents |= POLLIN;
    else
        selrecord(td, &sc->sc_rsel);
}
mtx_unlock(&sc->sc_mtx);
return (revents);
```

现在检查和注册相对于生产者是原子的。如果生产者在消费者检查之前更新状态，消费者看到事件并返回 `POLLIN` 而不注册。如果生产者即将在消费者处于临界区时更新状态，生产者必须等待消费者完成。在两种情况下，唤醒都到达消费者。

重要的微妙之处是 `selwakeup()` 在 softc 互斥锁之外调用。这是 FreeBSD 内核中的标准模式：在锁下更新状态，放弃锁，然后传递通知。`selwakeup()` 本身从许多上下文中安全调用是安全的，但它获取 selinfo 的内部互斥锁，我们不想在任意驱动程序锁内嵌套那个锁。实际上，规则是，在锁下跨越状态更新持有 softc 锁，放弃它，然后调用 `selwakeup()`。

你会在 FreeBSD 驱动程序中看到这种模式。在 `if_tuntap.c` 中，生产者路径从任何驱动程序锁之外调用 `selwakeuppri()`。在 `evdev/cdev.c` 中也是一样。生产者在其内部锁下更新状态，释放锁，然后发出唤醒。消费者，在 `d_poll` 中，跨越检查和 `selrecord()` 获取相同的锁。那个纪律消除了唤醒丢失竞争。

### 思考锁

为什么这有效？因为锁序列化两个特定操作：生产者的状态更新和消费者的检查加注册。`selwakeup()` 调用和线程随后的休眠在锁之外，但这没问题，因为底层机制的条件变量语义单独处理那个竞争。

这里是更详细的论据。假设消费者先获取锁。它检查状态，看到什么都没有，调用 `selrecord()` 注册，然后释放锁。一段时间后生产者获取锁，更新状态，释放锁，然后调用 `selwakeup()`。消费者已经注册，所以唤醒找到它。很好。

现在假设生产者先获取锁。它更新状态，释放锁，然后调用 `selwakeup()`。消费者还没有注册，所以唤醒没有找到等待者。这没问题，因为消费者还没有到达它会休眠的点；消费者仍然要获取锁。当消费者确实获取锁时，它检查状态，看到事件（因为生产者已经更新了它），然后返回 `POLLIN` 而不调用 `selrecord()`。消费者被正确通知。

第三种情况是棘手的。消费者刚刚检查了状态（在锁下）并即将调用 `selrecord()`，但实际上，因为锁一直被持有，这种情况不会发生。生产者在消费者释放锁之前无法更新状态，此时消费者已经注册。

所以锁定纪律是：始终在消费者的检查和注册期间持有锁，始终在生产者的状态更新期间持有锁。`selwakeup()` 调用本身发生在锁之外，因为它有自己的内部同步。

### 常见错误

几个错误值得显式指出。

在状态更新锁内调用 `selwakeup()` 在大多数情况下是错误的，因为 `selwakeup()` 本身可能需要获取其他锁（selinfo 互斥锁，线程的 selinfo 队列锁）。在 softc 互斥锁内做这件事创建了容易弄错的锁序机会。经验法则是，在锁下更新，放弃，然后 `selwakeup()`。

忘记唤醒所有感兴趣的 selinfo 是另一个常见错误。如果驱动程序有独立的读写 selinfo（例如，一个用于 `POLLIN` 等待者，一个用于 `POLLOUT` 等待者），它必须在状态改变时唤醒正确的那一个。唤醒错误的一个意味着实际等待者永远休眠。

在没有任何锁的情况下调用 `selrecord()` 创建了一个时间窗口，在这个窗口内事件可以在没有传递唤醒的情况下到达。这是我们刚刚分析的竞争，修复总是一样的：持有锁。

即使数据就绪也每次调用 `selrecord()` 不是正确性错误，但这是对每个线程 `selfd` 池的无意义负载。如果数据就绪，线程不会休眠，所以注册它是浪费的工作。"检查；如果就绪，返回；如果没有，注册"模式是正确的。

在已销毁的 selinfo 上调用 `selwakeup()` 是等待发生的崩溃。分离路径必须在释放 selinfo 或周围的 softc 之前调用 `seldrain()`。

### 诊断技术

当驱动程序的 poll 支持不工作时，有几个工具帮助你隔离问题。

第一个工具是 `top(1)`。加载驱动程序，在用户态程序中打开一个描述符，并让程序用长超时调用 `poll()`。在 `top -H` 中查看程序并检查 WCHAN 列。如果 poll 工作正常，线程的等待通道将是 `select` 或类似的东西。如果线程处于某种其他状态（运行、可运行、短休眠），poll 调用可能过早返回，或者程序可能在旋转。

第二个工具是驱动程序上的计数器。为每个 `selrecord()` 调用添加一个计数器，为每个 `selwakeup()` 调用添加一个，以及每次 `d_poll` 返回就绪掩码时添加一个。测试后，通过 `sysctl` 打印这些计数器。如果 `selrecord()` 触发但 `selwakeup()` 从不触发，生产者路径从未触发。如果 `selwakeup()` 触发但程序保持休眠，你可能因为在锁外发生状态更新和注册而丢失唤醒。

第三个工具是 `ktrace(1)` 和 `kdump(1)`。在 `ktrace` 下运行测试程序，转储将显示每个系统调用及其时间。调用 `poll()` 并阻塞的程序将在唤醒后显示 `RET poll` 条目，时间戳将告诉你唤醒实际何时到达。如果生产者事件发生在时间 T 而唤醒在几秒后到达，你有一个错误。

第四个工具是 DTrace，它可以插桩 `selwakeup` 本身。一个探测 `fbt:kernel:selwakeup:entry` 并打印调用驱动程序的 softc 指针的脚本显示系统中每个唤醒。如果你驱动程序的唤醒从未触发，DTrace 会毫秒级告诉你。

### 闭环：带 Poll 支持的 evdemo

将各个部分放在一起，这是我们的 `evdemo` 驱动程序正确支持 `poll()` 所需的最小额外代码：

```c
/* 在 softc 中 */
struct evdemo_softc {
    /* ... 现有字段 ... */
    struct selinfo sc_rsel;  /* 读选择器 */
};

/* 在 attach */
knlist_init_mtx(&sc->sc_rsel.si_note, &sc->sc_mtx);

/* 在 d_poll */
static int
evdemo_poll(struct cdev *dev, int events, struct thread *td)
{
    struct evdemo_softc *sc = dev->si_drv1;
    int revents = 0;

    mtx_lock(&sc->sc_mtx);
    if (events & (POLLIN | POLLRDNORM)) {
        if (sc->sc_nevents > 0)
            revents |= events & (POLLIN | POLLRDNORM);
        else
            selrecord(td, &sc->sc_rsel);
    }
    if (events & (POLLOUT | POLLWRNORM))
        revents |= events & (POLLOUT | POLLWRNORM);
    mtx_unlock(&sc->sc_mtx);

    return (revents);
}

/* 在生产者路径（对于 evdemo，这是从 callout 或 ioctl 触发的事件注入例程）*/
static void
evdemo_post_event(struct evdemo_softc *sc, struct evdemo_event *ev)
{
    mtx_lock(&sc->sc_mtx);
    evdemo_enqueue(sc, ev);
    mtx_unlock(&sc->sc_mtx);
    selwakeup(&sc->sc_rsel);
}

/* 在 detach */
seldrain(&sc->sc_rsel);
knlist_destroy(&sc->sc_rsel.si_note);
```

注意，即使我们还没有实现 kqueue，我们也在 selinfo 的嵌入 `si_note` knlist 上调用 `knlist_init_mtx()`。这几乎不花费我们什么，并使 selinfo 兼容我们将在第 4 节添加的 kqueue 支持。如果你不预先初始化 `si_note`，第一次尝试遍历 knlist 的 `selwakeup()` 调用会崩溃。许多驱动程序习惯性地在 attach 期间初始化 knlist。

还要注意，`evdemo_post_event` 辅助函数在更新事件计数时持有 softc 互斥锁，放弃互斥锁，然后调用 `selwakeup()`。那是标准的生产者模式，我们将贯穿本章其余部分重用它。

### 第 3 节总结

此时你已经拥有基于 poll 的异步 I/O 的所有概念和实践部分。你知道契约、内核结构、正确的锁定纪律和常见失败模式。你可以获取一个现有的阻塞驱动程序，添加 `d_poll` 支持，并让它在 `poll(2)` 和 `select(2)` 下正确行为。

问题是 `poll(2)` 和 `select(2)` 有众所周知的可扩展性限制。每次调用重新声明调用者感兴趣的一整套描述符，这是每次调用 O(N) 的。对于监视数千个描述符的程序，这种开销占主导地位。FreeBSD 自 20 世纪 90 年代末以来提供了更好的机制，即 `kqueue(2)`，这是下一节的主题。


## 4. 支持 kqueue 和 EVFILT_READ/EVFILT_WRITE

`kqueue(2)` 是 FreeBSD 的可扩展事件通知设施。与 `poll(2)` 和 `select(2)` 不同，它们要求用户态程序在每次调用时重新声明其兴趣，`kqueue(2)` 让程序注册兴趣一次，然后只询问实际触发的事件。对于监视一万个文件描述符且只有少数几个活动的程序，这是一个快速、交互式程序和一个慢速、负载程序之间的区别。

`kqueue` 也比 `poll` 更富表现力。除了基本的可读和可写过滤器，它还提供信号、定时器、文件系统事件、进程生命周期事件、用户定义事件和几个其他类别的过滤器。只想参与经典可读和可写通知的驱动程序仍然可以干净地融入框架；如果需要，更广泛的功能是可用的。

从驱动程序的角度来看，kqueue 支持向 `cdevsw` 添加一个回调，`d_kqfilter`，以及一组过滤器操作，一个 `struct filterops`，它为每种过滤器类型提供生命周期和事件传递函数。整个机制重用我们在第 3 节遇到的 `struct selinfo`，所以已经支持 `poll()` 的驱动程序可以通过编写大约一百行额外代码并调用少量新 API 来添加 `kqueue` 支持。

### 用户态看到的 kqueue 是什么样的

在我们实现驱动程序端之前，让我们看看用户程序是什么样的。调用者打开一个 `kqueue`，注册对文件描述符的兴趣，然后收获事件：

```c
#include <sys/event.h>

int kq = kqueue();
int fd = open("/dev/evdemo", O_RDONLY);

struct kevent change;
EV_SET(&change, fd, EVFILT_READ, EV_ADD | EV_CLEAR, 0, 0, NULL);
kevent(kq, &change, 1, NULL, 0, NULL);

for (;;) {
    struct kevent ev;
    int n = kevent(kq, NULL, 0, &ev, 1, NULL);
    if (n > 0 && ev.filter == EVFILT_READ) {
        char buf[256];
        ssize_t r = read(fd, buf, sizeof(buf));
        /* ... */
    }
}
```

`EV_SET` 宏构造一个描述兴趣的 `struct kevent`："监视文件描述符 `fd` 的 `EVFILT_READ` 事件，使用边触发（`EV_CLEAR`）语义，并保持它活动（`EV_ADD`）。"第一个 `kevent()` 调用注册该兴趣。循环然后以阻塞模式调用 `kevent()`，请求下一个事件，并在它到达时服务它。

驱动程序从不直接看到 `kqueue` 文件描述符或 `kevent` 结构。它只看到每个兴趣的 `struct knote` 及其附加的 `struct filterops`。注册通过框架流到我们的 `d_kqfilter` 回调，它选择正确的过滤器操作并将 knote 附加到我们的 softc。传递通过生产者路径中的 `KNOTE_LOCKED()` 调用流过，它遍历我们的 knote 列表并通知每个附加的 kqueue 就绪事件。

### 数据结构

驱动程序端有两个重要的结构：`struct filterops` 和 `struct knlist`。

`struct filterops`，定义在 `/usr/src/sys/sys/event.h` 中，保存每个过滤器的生命周期函数：

```c
struct filterops {
    int     f_isfd;
    int     (*f_attach)(struct knote *kn);
    void    (*f_detach)(struct knote *kn);
    int     (*f_event)(struct knote *kn, long hint);
    void    (*f_touch)(struct knote *kn, struct kevent *kev, u_long type);
    int     (*f_userdump)(struct proc *p, struct knote *kn,
                          struct kinfo_knote *kin);
};
```

驱动程序关心的字段是：

`f_isfd` 如果过滤器附加到文件描述符则为 1。几乎所有驱动程序过滤器都将其设置为 1。监视不与 fd 绑定的东西的过滤器（如 `EVFILT_TIMER`）会将其设置为 0。

`f_attach` 在 knote 正在附加到新注册的兴趣时调用。许多驱动程序将其保留为 `NULL`，因为所有附加工作都在 `d_kqfilter` 本身发生。

`f_detach` 在 knote 正在被移除时调用。驱动程序用它从其内部 knote 列表中注销 knote。

`f_event` 被调用以评估过滤器的条件当前是否满足。如果满足则返回非零，否则返回零。它是 kqueue 等价的 `d_poll` 中的状态检查。

`f_touch` 用于过滤器支持不应被视为完全重新注册的 `EV_ADD`/`EV_DELETE` 更新时使用。大多数驱动程序将其保留为 `NULL` 并接受默认行为。

`f_userdump` 用于内核内省，在驱动程序代码中可以保留 `NULL`。

`struct knlist`，定义在同一个头文件中，保存附加到特定对象的 knote 列表。它携带指向对象锁操作的指针，以便 kqueue 框架在传递事件时可以获取和释放正确的锁：

```c
struct knlist {
    struct  klist   kl_list;
    void    (*kl_lock)(void *);
    void    (*kl_unlock)(void *);
    void    (*kl_assert_lock)(void *, int);
    void    *kl_lockarg;
    int     kl_autodestroy;
};
```

驱动程序很少直接接触这个结构。框架提供辅助函数，从 `knlist_init_mtx()` 开始，用于常见情况，即 knlist 由单个互斥锁保护。

### 初始化 knlist

初始化 knlist 的最简单方式是：

```c
knlist_init_mtx(&sc->sc_rsel.si_note, &sc->sc_mtx);
```

第一个参数是要初始化的 knlist。第二个是驱动程序的互斥锁。框架存储该互斥锁，并在需要保护 knote 列表时获取它。knote 列表通常嵌入在 `struct selinfo` 中，正如我们在上一节看到的；重用同一个 selinfo 用于 poll 和 kqueue 等待者可以让单个 `selwakeup()` 调用同时覆盖两种机制。

对于已经通过 `M_ZERO` 零化 softc 的驱动程序，初始化只是 attach 期间的这一个调用。

### d_kqfilter 回调

`d_kqfilter` 回调是 kqueue 注册的入口点。其签名，在 `/usr/src/sys/sys/conf.h` 中，是：

```c
typedef int d_kqfilter_t(struct cdev *dev, struct knote *kn);
```

`dev` 参数是我们的 `cdev`。`kn` 参数是正在注册的 knote。回调决定哪些过滤器操作适用，将 knote 附加到我们的 knote 列表，并在成功时返回零。

支持 `EVFILT_READ` 的驱动程序的最小实现：

```c
static int
evdemo_kqfilter(struct cdev *dev, struct knote *kn)
{
    struct evdemo_softc *sc = dev->si_drv1;

    switch (kn->kn_filter) {
    case EVFILT_READ:
        kn->kn_fop = &evdemo_read_filterops;
        kn->kn_hook = sc;
        knlist_add(&sc->sc_rsel.si_note, kn, 0);
        return (0);
    default:
        return (EINVAL);
    }
}
```

让我们遍历这个。

对 `kn->kn_filter` 的 `switch` 决定我们处理的是哪种过滤器类型。只支持 `EVFILT_READ` 的驱动程序对其他任何东西返回 `EINVAL`。也支持 `EVFILT_WRITE` 的驱动程序有第二个 case，指向不同的过滤器操作结构。

我们设置 `kn->kn_fop` 为此过滤器类型的过滤器操作。kqueue 框架在 knote 生命周期进展时调用这些操作。

我们设置 `kn->kn_hook` 为 softc。knote 有这个通用指针供每驱动程序使用。我们的过滤器函数在它们被调用时将从 `kn->kn_hook` 中拉出 softc。

我们调用 `knlist_add()` 将 knote 链接到我们的 knote 列表。第三个参数 `islocked` 在这里是零，因为我们此时没有持有 knlist 锁。如果我们持有，我们会传递 1。

返回零表示成功。

### filterops 实现

过滤器操作是每个过滤器行为所在的地方。对于 `evdemo` 上的 `EVFILT_READ`，它们看起来像这样：

```c
static int
evdemo_kqread(struct knote *kn, long hint)
{
    struct evdemo_softc *sc = kn->kn_hook;
    int ready;

    mtx_assert(&sc->sc_mtx, MA_OWNED);

    kn->kn_data = sc->sc_nevents;
    ready = (sc->sc_nevents > 0);

    if (sc->sc_detaching) {
        kn->kn_flags |= EV_EOF;
        ready = 1;
    }

    return (ready);
}

static void
evdemo_kqdetach(struct knote *kn)
{
    struct evdemo_softc *sc = kn->kn_hook;

    knlist_remove(&sc->sc_rsel.si_note, kn, 0);
}

static const struct filterops evdemo_read_filterops = {
    .f_isfd   = 1,
    .f_attach = NULL,
    .f_detach = evdemo_kqdetach,
    .f_event  = evdemo_kqread,
};
```

`f_event` 函数，`evdemo_kqread`，每次框架想知道过滤器是否就绪时被调用。它查看 softc，在 `kn->kn_data` 中报告可用事件的数量（kqueue 用户依赖的约定，用于了解有多少数据可用），如果至少有一个事件等待则返回非零。它还在设备正在分离时翻转 `EV_EOF` 标志，这让用户态看到不再有事件到来。

注意 softc 互斥锁被持有的断言。框架获取我们的 knlist 的锁，我们通过 `knlist_init_mtx` 告诉它是 softc 互斥锁。因为 `f_event` 回调在该锁内被调用，我们可以安全地查看 `sc_nevents` 和 `sc_detaching`。

`f_detach` 函数在用户态不再关心此注册时从我们的 knlist 中移除 knote。

常量 `evdemo_read_filterops` 是 `d_kqfilter` 在上一小节中指向的。`f_isfd = 1` 告诉框架此过滤器与文件描述符绑定，这是任何驱动程序级过滤器的正确值。

### 通过 KNOTE_LOCKED 传递事件

在生产者端，我们需要在驱动程序状态改变时通知注册的 knote。宏是 `KNOTE_LOCKED()`，定义在 `/usr/src/sys/sys/event.h` 中：

```c
#define KNOTE_LOCKED(list, hint)    knote(list, hint, KNF_LISTLOCKED)
```

它接受一个 knlist 指针和一个提示。提示传递给每个 knote 的 `f_event` 回调，给生产者一种方式将上下文（例如，特定事件类型）传递给过滤器。大多数驱动程序传递零。

`KNOTE_LOCKED` 变体是当你已经持有 knlist 的锁时想要的。`KNOTE_UNLOCKED` 变体在你不持有时使用。由于 knlist 的锁通常是你的 softc 互斥锁，而且由于生产者路径的其余部分正在持有该锁，`KNOTE_LOCKED` 是通常的选择。

将它添加到我们的生产者路径：

```c
static void
evdemo_post_event(struct evdemo_softc *sc, struct evdemo_event *ev)
{
    mtx_lock(&sc->sc_mtx);
    evdemo_enqueue(sc, ev);
    KNOTE_LOCKED(&sc->sc_rsel.si_note, 0);
    mtx_unlock(&sc->sc_mtx);
    selwakeup(&sc->sc_rsel);
}
```

我们现在从同一个生产者通知 kqueue 和 poll 等待者。softc 互斥锁内的 `KNOTE_LOCKED` 遍历 knote 列表并评估每个 knote 的 `f_event`，向任何有活动等待者的 kqueue 排队通知。锁外的 `selwakeup` 唤醒 `poll()` 和 `select()` 等待者。两种机制是独立的，互不干扰。

### 分离：清理 knlist

在分离时，驱动程序必须在销毁之前排空 knlist。干净的序列是：

```c
knlist_clear(&sc->sc_rsel.si_note, 0);
seldrain(&sc->sc_rsel);
knlist_destroy(&sc->sc_rsel.si_note);
```

`knlist_clear()` 移除仍然附加的每个 knote。此调用后，仍然有 kqueue 注册的任何用户态程序将看到 knote 在下次收获时消失。`seldrain()` 唤醒任何停放的 `poll()` 等待者以便它们返回。`knlist_destroy()` 检查列表为空并释放内部资源。

顺序很重要。如果你在不先清除的情况下销毁 knlist，销毁会在列表非空的断言上 panic。如果你清除 knlist 但留下 poll 等待者停放，它们将休眠直到某物唤醒它们，这是浪费的。遵循上述序列，分离路径是干净的。

### 更完整的例子：管道

打开 `/usr/src/sys/kern/sys_pipe.c` 并查看管道 kqfilter 实现。它是内核中最广泛的例子之一，值得完整阅读，因为管道同时支持读和写过滤器，并有正确的 EOF 处理。关键部分是两个 filterops 结构：

```c
static const struct filterops pipe_rfiltops = {
    .f_isfd   = 1,
    .f_detach = filt_pipedetach,
    .f_event  = filt_piperead,
    .f_userdump = filt_pipedump,
};

static const struct filterops pipe_wfiltops = {
    .f_isfd   = 1,
    .f_detach = filt_pipedetach,
    .f_event  = filt_pipewrite,
    .f_userdump = filt_pipedump,
};
```

以及读过滤器的事件函数：

```c
static int
filt_piperead(struct knote *kn, long hint)
{
    struct file *fp = kn->kn_fp;
    struct pipe *rpipe = kn->kn_hook;

    PIPE_LOCK_ASSERT(rpipe, MA_OWNED);
    kn->kn_data = rpipe->pipe_buffer.cnt;
    if (kn->kn_data == 0)
        kn->kn_data = rpipe->pipe_pages.cnt;

    if ((rpipe->pipe_state & PIPE_EOF) != 0 &&
        ((rpipe->pipe_type & PIPE_TYPE_NAMED) == 0 ||
        fp->f_pipegen != rpipe->pipe_wgen)) {
        kn->kn_flags |= EV_EOF;
        return (1);
    }
    kn->kn_flags &= ~EV_EOF;
    return (kn->kn_data > 0);
}
```

注意 EOF 的处理，当管道不再处于 EOF 时显式清除 `EV_EOF`（这对命名管道有新写入者很重要），以及使用 `kn->kn_data` 报告可用数据量。这些是完成的驱动程序做对的细节。

### struct knote 的剖析

我们一直在传递 `struct knote` 指针而没有仔细看它，但驱动程序的生活一旦我们知道它包含什么就更容易。`struct knote`，定义在 `/usr/src/sys/sys/event.h` 中，是内核的每个注册记录。每次注册兴趣的 `kevent(2)` 调用创建恰好一个 knote，该 knote 持续直到注册被移除。对于驱动程序，knote 是货币单位：每个 knlist 操作接受一个 knote，每个过滤器回调接收一个 knote，每次传递遍历它们的一个列表。知道什么存在于结构内部将我们一直在遵循的回调契约变成我们可以推理而不仅仅是记忆的东西。

驱动程序关心的字段是整个结构的一个小子集，但每一个都值得注意。

`kn_filter` 标识用户态请求的过滤器类型。在 `d_kqfilter` 内部，这是我们 switch 的东西：`EVFILT_READ`、`EVFILT_WRITE`、`EVFILT_EXCEPT` 等等。值来自用户态提交的 `struct kevent` 的 `filter` 字段。只支持一种过滤器类型的驱动程序检查此字段并以 `EINVAL` 拒绝任何不匹配。

`kn_fop` 是将在该 knote 的生命周期内服务它的 `struct filterops` 表的指针。驱动程序在 `d_kqfilter` 内部设置此指针。该点之后，框架通过此指针调用以到达我们的 attach、detach、event 和 touch 回调。filterops 表在我们检查的驱动程序中总是 `static const`，因为框架不对它取引用，驱动程序被期望为 knote 的生命周期保持指针有效。

`kn_hook` 是一个通用的每驱动程序指针。驱动程序通常将其设置为 softc、每客户端状态记录或过滤器应该反应的任何对象。框架从不读取或写入它。当过滤器回调运行时，它们从 `kn_hook` 拉出驱动程序状态，而不是通过全局查找，这既避免了查找成本，也避免了全局查找可能引入的一类锁序问题。

`kn_hookid` 是 `kn_hook` 的整数伙伴，可用于每驱动程序标记。大多数驱动程序不管它。

`kn_data` 是过滤器的 `f_event` 回调将"有多少就绪"传回用户态的方式。对于可读过滤器，驱动程序约定存储可用字节或记录的数量。对于可写过滤器，它们存储可用空间量。用户态通过返回的 `struct kevent` 的 `data` 字段读取此值，像 `libevent` 这样的工具依赖该约定。`/dev/klog` 驱动程序在此存储原始字节计数，而 evdev 驱动程序存储按 `sizeof(struct input_event)` 缩放记录计数的队列深度（以字节为单位），因为 evdev 客户端读取 `struct input_event` 记录而不是原始字节。

`kn_sfflags` 和 `kn_sdata` 保存用户态通过 `struct kevent` 的 `fflags` 和 `data` 字段请求的每注册标志和数据。支持细粒度控制的过滤器，如 `EVFILT_TIMER` 及其周期或 `EVFILT_VNODE` 及其 note 掩码，查看这些以决定如何行为。简单驱动程序过滤器通常忽略它们。

`kn_flags` 保存框架在下次收获时传递给用户态的传递时标志。每个驱动程序使用的是 `EV_EOF`，它发出"此源不再会有数据到达"的信号。驱动程序在 `f_event` 中设置 `EV_EOF`，当设备正在分离、伪终端的对端已关闭、管道已失去其写入者，或就绪信号已变为永久时。

`kn_status` 是框架拥有的内部状态：`KN_ACTIVE`、`KN_QUEUED`、`KN_DISABLED`、`KN_DETACHED` 和少数其他。驱动程序绝不能修改它。驱动程序的工作只是通过 `f_event` 报告就绪；框架相应地更新 `kn_status`。

`kn_link`、`kn_selnext` 和 `kn_tqe` 是各种 kqueue 框架列表使用的链表链接字段。knlist 辅助程序代表我们操作它们。驱动程序绝不应直接接触它们。

放在一起，这些字段讲述一个简单的故事。驱动程序在 `d_kqfilter` 内部创建 knote 与其过滤器操作的关联，设置 `kn_hook` 和可选的 `kn_hookid` 以便过滤器回调可以恢复它们的上下文，然后让框架管理链接和状态。驱动程序通过 `f_event` 拥有就绪报告，除此之外没有别的。驱动程序和框架之间的交接是干净的，该领域的大多数驱动程序错误来自试图跨越该边界，要么通过修改框架拥有的状态标志，要么在 `f_detach` 触发后保留过期的 knote 指针。

一点值得强调：knote 比任何单个 `f_event` 调用都长，但它不会比 `f_detach` 长。一旦框架调用 `f_detach`，knote 正在被拆除；驱动程序必须从它附加的任何内部结构中取消挂钩，绝不能保留指针。驱动程序拥有的 `kn_hook` 指针必须被同样对待。如果驱动程序出于某种原因保持从 softc 字段到 knote 的反向指针（不常见，但有时对驱动程序发起的分离有用），它必须在框架释放 knote 之前的 `f_detach` 期间清除该反向指针。

### struct knlist 内部：驱动程序的等待室如何工作

`struct knlist`，声明在 `/usr/src/sys/sys/event.h` 中，是驱动程序累积当前对其中一个通知源感兴趣的 knote 的地方。每个可以唤醒 kqueue 等待者的驱动程序对象拥有至少一个 knlist。管道对象拥有两个，一个用于读取者，一个用于写入者。tty 也拥有两个，`t_inpoll` 和 `t_outpoll`，每个都有自己的 knlist。evdev 客户端对象每个客户端拥有一个。在我们的 `evdemo` 驱动程序中，我们借用我们已有的用于 poll 的 `struct selinfo.si_note`，所以同一个 knlist 是唤醒 poll 和 kqueue 消费者的那个。

结构本身很小：

```c
struct knlist {
    struct  klist   kl_list;
    void    (*kl_lock)(void *);
    void    (*kl_unlock)(void *);
    void    (*kl_assert_lock)(void *, int);
    void    *kl_lockarg;
    int     kl_autodestroy;
};
```

`kl_list` 是通过每个 knote 的 `kn_selnext` 字段链接的 `struct knote` 条目的单链表头。列表头由框架操作，绝不由驱动程序直接操作。

`kl_lock`、`kl_unlock` 和 `kl_assert_lock` 是框架在需要获取对象锁时使用的函数指针。knlist 不拥有自己的锁；它借用驱动程序的锁定机制。这就是为什么 `struct selinfo` 可以携带 knlist 而不创建单独的锁：锁是驱动程序已经声明的任何东西。

`kl_lockarg` 是传递给那些锁函数的参数。当我们用 `knlist_init_mtx(&knl, &sc->sc_mtx)` 初始化 knlist 时，框架在 `kl_lockarg` 中存储 `&sc->sc_mtx`，并安排锁回调包装 `mtx_lock` 和 `mtx_unlock`。驱动程序从不看到此接线，也从不需要。

`kl_autodestroy` 是少数特定子系统使用的标志，最著名的是 AIO，其中 knlist 存在于 `struct kaiocb` 内部，必须在请求完成时自动拆除。驱动程序代码几乎从不设置此标志。`/usr/src/sys/kern/vfs_aio.c` 中的 `aio_filtops` 路径是规范用法，记住该标志存在是有价值的，这样稍后阅读该文件不会让你惊讶。

锁契约值得强调，因为它是单个最常见的 kqueue 驱动程序错误来源。当框架调用我们的 `f_event` 时，它持有 knlist 锁，即我们的 softc 互斥锁。我们的 `f_event` 可能读取 softc 状态但绝不能再次获取 softc 互斥锁（它已经是我们的），绝不能休眠，绝不能阻塞在任何其他可能跨 `f_event` 调用持有的锁上。当我们调用 `KNOTE_LOCKED` 时，我们断言我们已经持有锁，所以框架在遍历列表的路上跳过锁定。当我们调用 `KNOTE_UNLOCKED` 时，框架代表我们获取和释放锁。在一个生产者路径内混合两种风格是负载下微妙双重锁定 panic 的经典来源。

与 `struct selinfo` 的统一值得注意。回到第 3 节，我们将 `struct selinfo` 视为仅用于 poll 的概念，但它实际上在其 `si_note` 成员中嵌入了一个 `struct knlist`。这就是已经支持 `poll()` 的驱动程序拥有其 softc 中 kqueue 基础设施的原因：添加 kqueue 很大程度上是用 `knlist_init_mtx` 初始化 knlist 并连接过滤器操作的问题。生产者路径已经调用 `selwakeup()`，它本身在适当的锁下遍历 `si_note` 并通知任何附加的 knote。用 `KNOTE_LOCKED(&sc->sc_rsel.si_note, 0)` 显式通知让我们更清楚，并让我们选择 kqueue 扇出相对于任何其他生产者工作发生的确切时间。在我们下面将阅读的驱动程序中，两种风格都出现；只要锁定一致，任何一种都是正确的。

### knlist 生命周期详述

knlist 的生命周期遵循拥有它的驱动程序对象的生命周期。knlist 在 attach 期间（对于真实硬件驱动程序是驱动程序的 attach 入口点，对于伪设备是 SYSINIT）进入存在，在用户态消费者的打开-读取-关闭周期中存活，并在 detach 时拆除。我们需要的函数，全部声明在 `/usr/src/sys/sys/event.h` 中并实现在 `/usr/src/sys/kern/kern_event.c` 中，是 `knlist_init`、`knlist_init_mtx`、`knlist_add`、`knlist_remove`、`knlist_clear` 和 `knlist_destroy`。

`knlist_init_mtx` 是几乎每个驱动程序调用的那个。它初始化列表头，将 knlist 配置为使用 `mtx_lock`/`mtx_unlock`，以驱动程序的互斥锁作为参数，并将 knlist 标记为活动。调用者传递指向 knlist 的指针（通常是 `&sc->sc_rsel.si_note`，或者对于具有每方向通知的驱动程序，还有 `&sc->sc_wsel.si_note`）和指向驱动程序中已存在的互斥锁的指针。

`knlist_init` 是通用形式，当驱动程序的锁机制不是简单互斥锁时使用。它接受三个函数指针（lock、unlock、assert）、传递给这些函数的参数指针和底层列表头。管道使用 `_mtx` 形式及其管道对互斥锁；套接字缓冲区使用定制的 `knlist_init`，因为它们有自己的锁定规则。大多数驱动程序不需要通用形式。

`knlist_add` 从 `d_kqfilter` 调用以将新注册的 knote 链接到列表。其原型是 `void knlist_add(struct knlist *knl, struct knote *kn, int islocked)`。`islocked` 参数告诉函数调用者是否已经持有 knlist 锁。如果为零，函数为我们获取锁。如果为一，我们断言我们已经持有它。在 `d_kqfilter` 内部不做额外锁定的驱动程序传递零；像 `/dev/klog` 这样在进入时获取 msgbuf 锁的驱动程序传递一。两种模式都是正确的；选择取决于驱动程序想在 `knlist_add` 调用周围保护什么。

`knlist_remove` 是反向操作，通常从 `f_detach` 回调调用。其原型是 `void knlist_remove(struct knlist *knl, struct knote *kn, int islocked)`。框架在已持有 knlist 锁时调用 `f_detach`，所以该上下文中 `islocked` 是一。如果驱动程序出于某种原因需要从 `f_detach` 外部移除特定 knote（这是不寻常的且很少正确），它必须安排自己的锁定。

`knlist_clear` 是驱动程序分离时使用的批量移除函数。它遍历列表，移除每个 knote，并将每个标记为 `EV_EOF | EV_ONESHOT`，以便用户态看到最终事件且注册被丢弃。签名 `void knlist_clear(struct knlist *knl, int islocked)` 实际上是 `/usr/src/sys/kern/kern_event.c` 中带有 NULL `struct thread *` 和设置的 kill 标志的 `knlist_cleardel` 的包装器，意味着"移除一切"。驱动程序在拆除 knlist 之前从 `detach` 调用此函数。


`knlist_destroy` 释放 knlist 的内部机制。在调用它之前，knlist 必须为空。如果你用有活动 knote 的 knlist 销毁，内核会断言并 panic。这就是我们之前看到的分离序列是严格的原因：

```c
knlist_clear(&sc->sc_rsel.si_note, 0);
seldrain(&sc->sc_rsel);
knlist_destroy(&sc->sc_rsel.si_note);
```

`knlist_clear` 清空列表。`seldrain` 唤醒仍然停放在同一个 selinfo 上的任何 `poll()` 等待者，以便它们的等待线程从内核返回。`knlist_destroy` 拆除内部结构并验证列表为空。如果跳过任何这些步骤，分离就变得不安全：尝试调用已卸载驱动程序的 `f_event` 的活动 knote 会崩溃内核；其 selinfo 已被释放的 poll 等待者会醒来面对一个悬空指针。

在 `/usr/src/sys/kern/kern_event.c` 中 `knlist_remove` 的实现中有两点进一步值得注意。它走入内部辅助函数 `knlist_remove_kq`，该函数也获取 kq 锁，以便移除与任何进行中的事件传递相干。它还在 `kn_status` 中设置 `KN_DETACHED` 以向框架其余部分发出此 knote 已消失的信号。驱动程序从不直接观察 `KN_DETACHED`，但理解它存在解释了为什么并发的分离和事件传递可以安全竞争：框架的内部状态机保持它们一致。

### kqfilter 回调契约

`d_kqfilter` 从 `/usr/src/sys/kern/kern_event.c` 中的 kqueue 注册路径调用，具体是通过文件描述符的 `fo_kqfilter` 方法从 `kqueue_register` 调用。当回调运行时，框架已经验证了文件描述符，分配了 `struct knote`，并填写了用户态的请求。我们的工作很窄：选择正确的 filterops，附加到正确的 knlist，并返回零。

`d_kqfilter` 必须做的。它必须检查 `kn->kn_filter` 以决定用户态请求的过滤器类型。它必须将 `kn->kn_fop` 设置为该类型的有效 `struct filterops`。它必须将 knote 附加到属于我们驱动程序的 knlist，通常通过调用 `knlist_add`。它必须在成功时返回零或在失败时返回合理的 errno。如果驱动程序无法服务请求的过滤器，`EINVAL` 是正确的答案。

`d_kqfilter` 绝不能做的。它绝不能休眠，因为 kqueue 注册路径持有不可安全休眠的锁。它绝不能用 `M_WAITOK` 分配内存，原因相同。它绝不能调用可能阻塞另一个进程的任何函数。如果驱动程序需要不止快速查找和 knlist 插入，它就做错了。回调本质上是一个快速路径接线操作。

进入时的锁状态值得理解。框架在调用 `d_kqfilter` 时不持有 knlist 锁。因此，如果我们自己没有获取 knlist 的锁，我们可以传递 `islocked = 0` 给 `knlist_add`。如果我们的驱动程序作为过滤器选择逻辑的一部分需要查看 softc 状态，例如像 evdev 驱动程序那样在撤销的 cdev 上报告 `ENODEV`，我们可以自己获取 softc 互斥锁，检查状态，用 `islocked = 1` 做 `knlist_add`，并在返回前释放互斥锁。下面的 evdev 示例展示了完全那个模式。

从 `d_kqfilter` 返回非零值意味着"用户态将从 `kevent(2)` 获得此 errno。"它不意味着"重试。"返回 `EAGAIN` 的驱动程序会困惑用户态，因为 `kevent` 不像 `read` 那样解释该值。对于不支持的过滤器坚持使用 `EINVAL`，对于撤销或拆除的设备使用 `ENODEV`，并避免聪明的错误返回。

关于何时调用 `d_kqfilter` 的一个微妙之处：用 `EV_ADD` 注册新兴趣的单个 `kevent(2)` 调用进入框架，发现此（文件，过滤器）对尚不存在 knote，分配一个，然后调用文件描述符的 fileops 上的 `fo_kqfilter`。那就是我们通过 cdev fileops 表到达 `d_kqfilter` 的地方。如果调用者改为更新现有注册（例如，用 `EV_ENABLE`/`EV_DISABLE` 在启用和禁用之间切换），我们的回调不参与；框架通过 `f_touch` 或直接状态操作在内部处理。

### 示例：/dev/klog 驱动程序

内核日志设备 `/dev/klog` 的树中最简单的真实驱动程序端 `kqfilter` 实现在 `/usr/src/sys/kern/subr_log.c` 中。其整个 kqueue 支持适合大约四十行，并使用我们一直在讨论的完全模式。让我们阅读它。

filterops 表是最小的，只有 detach 和 event 回调：

```c
static const struct filterops log_read_filterops = {
    .f_isfd   = 1,
    .f_attach = NULL,
    .f_detach = logkqdetach,
    .f_event  = logkqread,
};
```

attach 钩子是 NULL，因为所有驱动程序端工作在 `logkqfilter` 本身发生。不需要单独的 `f_attach` 回调；`d_kqfilter` 入口点做它需要做的一切。需要执行超出 `d_kqfilter` 所做的每 knote 设置的驱动程序可以使用 `f_attach`，但这不常见。

`logkqfilter` 是 `d_kqfilter` 回调：

```c
static int
logkqfilter(struct cdev *dev __unused, struct knote *kn)
{

    if (kn->kn_filter != EVFILT_READ)
        return (EINVAL);

    kn->kn_fop = &log_read_filterops;
    knlist_add(&logsoftc.sc_selp.si_note, kn, 1);

    return (0);
}
```

`/dev/klog` 驱动程序只支持可读事件；对任何其他过滤器类型的请求获得 `EINVAL`。回调将 `kn_fop` 设置为静态 filterops 表，然后将 knote 附加到 softc 的 selinfo knlist。这里对 `knlist_add` 的第三个参数是 `1`，意味着"调用者已经持有 knlist 锁。"驱动程序在进入回调之前获取消息缓冲锁是为了自己的原因，所以传递 `1` 是正确的。

事件函数同样短：

```c
static int
logkqread(struct knote *kn, long hint __unused)
{

    mtx_assert(&msgbuf_lock, MA_OWNED);

    kn->kn_data = msgbuf_getcount(msgbufp);
    return (kn->kn_data != 0);
}
```

它断言消息缓冲锁（这是 knlist 使用的），读取排队字节数，如果有任何可用则返回非零。用户态在下次收获时在 `kn->kn_data` 中看到字节计数。

detach 函数是一行：

```c
static void
logkqdetach(struct knote *kn)
{

    knlist_remove(&logsoftc.sc_selp.si_note, kn, 1);
}
```

它从 knlist 移除 knote，再次传递 `1`，因为框架在进入 `f_detach` 之前已经获取了锁。

最后一块是生产者。当日志超时触发且有新数据要通知等待者时，`/dev/klog` 在消息缓冲锁下调用 `KNOTE_LOCKED(&logsoftc.sc_selp.si_note, 0)`。那遍历 knlist，调用每个注册的 knote 的 `f_event`，并为任何有等待者的 kqueue 排队通知。零的提示被 `logkqread` 忽略，这是常见情况。

整个 kqueue 集成在子系统启动时通过 `knlist_init_mtx(&logsoftc.sc_selp.si_note, &msgbuf_lock)` 初始化一次。`/dev/klog` 在实践中从不卸载，所以这里没有可研究的拆除序列。那稍后出现在 evdev 示例中。

结论是这个代码有多小。FreeBSD 14.3 中真实驱动程序的完整、工作、生产级 `kqfilter` 集成不到四十行。kqueue 的复杂性在框架中，不在驱动程序的贡献中。

### 示例：TTY 读写过滤器

`/usr/src/sys/kern/tty.c` 中的终端子系统给我们下一步：一个同时支持可读和可写过滤器的驱动程序，并使用 `EV_EOF` 发出设备已消失的信号。该模式是任何想要暴露同一设备两个独立侧面的驱动程序使用的模式。

`/usr/src/sys/kern/tty.c` 中的两个 filterops 表是：

```c
static const struct filterops tty_kqops_read = {
    .f_isfd   = 1,
    .f_detach = tty_kqops_read_detach,
    .f_event  = tty_kqops_read_event,
};

static const struct filterops tty_kqops_write = {
    .f_isfd   = 1,
    .f_detach = tty_kqops_write_detach,
    .f_event  = tty_kqops_write_event,
};
```

`d_kqfilter` 入口点 `ttydev_kqfilter` 切换请求的过滤器并附加到两个 knlist 之一：

```c
static int
ttydev_kqfilter(struct cdev *dev, struct knote *kn)
{
    struct tty *tp = dev->si_drv1;
    int error;

    error = ttydev_enter(tp);
    if (error != 0)
        return (error);

    switch (kn->kn_filter) {
    case EVFILT_READ:
        kn->kn_hook = tp;
        kn->kn_fop = &tty_kqops_read;
        knlist_add(&tp->t_inpoll.si_note, kn, 1);
        break;
    case EVFILT_WRITE:
        kn->kn_hook = tp;
        kn->kn_fop = &tty_kqops_write;
        knlist_add(&tp->t_outpoll.si_note, kn, 1);
        break;
    default:
        error = EINVAL;
        break;
    }

    tty_unlock(tp);
    return (error);
}
```

这里有三件事值得注意。

首先，每个方向有自己的 selinfo（`t_inpoll`、`t_outpoll`）并因此有自己的 knlist。可读 knote 进入一个列表，可写 knote 进入另一个。这使生产者可以只通知改变的那一侧：当传入字符到达时，只有可读等待者醒来；当输出缓冲区排空时，只有可写等待者醒来。将两侧统一到一个 knlist 的驱动程序不得不为每次状态改变浪费周期唤醒所有人。

其次，对 `knlist_add` 的第三个参数是 `1`，因为 `ttydev_enter` 在 switch 运行之前已经获取了 tty 锁。tty 子系统在大多数入口点保持该锁，从进入到退出，所以里面的每个 knlist 操作都是已锁定的。

第三，读事件回调展示了我们之前描述的 `EV_EOF` 规则：

```c
static int
tty_kqops_read_event(struct knote *kn, long hint __unused)
{
    struct tty *tp = kn->kn_hook;

    tty_lock_assert(tp, MA_OWNED);

    if (tty_gone(tp) || (tp->t_flags & TF_ZOMBIE) != 0) {
        kn->kn_flags |= EV_EOF;
        return (1);
    } else {
        kn->kn_data = ttydisc_read_poll(tp);
        return (kn->kn_data > 0);
    }
}
```

如果 tty 已消失或是僵尸，设置 `EV_EOF` 且过滤器报告就绪，以便用户态醒来、读取、什么都得不到，并从 EOF 标志了解设备已结束。否则过滤器报告可读字节数以及该计数是否为正。写侧回调 `tty_kqops_write_event` 镜像此模式，报告输出缓冲区空闲空间的 `ttydisc_write_poll`。detach 回调简单地从它所在的任何列表中移除 knote，再次使用 `islocked = 1`。

tty 示例教给我们的是，有两个方向的驱动程序需要两个 knlist、两个 filterops 表、两个事件函数和一个将注册引导到正确位置的 `d_kqfilter`。生产者侧是对称的：传入字符在 `t_inpoll.si_note` 上触发 `KNOTE_LOCKED`；传出缓冲区空间在 `t_outpoll.si_note` 上触发相同。分离是干净且可预测的，它匹配用户态程序思考终端 I/O 的方式。

### 示例：evdev 分离规则

对于最后一个示例我们转向 `/usr/src/sys/dev/evdev/cdev.c` 中的输入事件子系统。其 kqfilter 在结构上类似 `/dev/klog`，但 evdev 驱动程序展示了前面两个示例略过的东西：完整的分离序列，即使在活动用户态进程可能仍有 kqueue 注册未完成时也能安全拆除 knlist。

filterops 和附加路径看起来很熟悉。evdev filterops 表是：

```c
static const struct filterops evdev_cdev_filterops = {
    .f_isfd   = 1,
    .f_detach = evdev_kqdetach,
    .f_event  = evdev_kqread,
};
```

`d_kqfilter` 实现添加了对撤销的重要额外检查，这使 evdev 比 `/dev/klog` 稍微丰富一点：

```c
static int
evdev_kqfilter(struct cdev *dev, struct knote *kn)
{
    struct evdev_client *client;
    int ret;

    ret = devfs_get_cdevpriv((void **)&client);
    if (ret != 0)
        return (ret);

    switch (kn->kn_filter) {
    case EVFILT_READ:
        kn->kn_fop = &evdev_cdev_filterops;
        kn->kn_hook = client;
        EVDEV_CLIENT_LOCKQ(client);
        if (client->ec_revoked)
            ret = ENODEV;
        else
            knlist_add(&client->ec_selp.si_note, kn, 1);
        EVDEV_CLIENT_UNLOCKQ(client);
        break;
    default:
        ret = EINVAL;
    }

    return (ret);
}
```

如果客户端已被撤销（因为设备正在离开或因为控制进程已显式撤销访问），驱动程序返回 `ENODEV` 而不是附加 knote。注意驱动程序围绕 `ec_revoked` 检查和 `knlist_add` 都获取自己的每客户端锁，所以两个操作相对于撤销是原子的。这是我们之前描述的契约，干净地应用：廉价查找、短暂持锁、不休眠、热路径中无内存分配。

事件函数从每客户端事件队列报告就绪：

```c
static int
evdev_kqread(struct knote *kn, long hint __unused)
{
    struct evdev_client *client = kn->kn_hook;

    EVDEV_CLIENT_LOCKQ_ASSERT(client);

    kn->kn_data = EVDEV_CLIENT_SIZEQ(client) *
                  sizeof(struct input_event);
    if (client->ec_revoked) {
        kn->kn_flags |= EV_EOF;
        return (1);
    }
    return (kn->kn_data != 0);
}
```

注意 `kn->kn_data` 约定：不只是"项目数"而是"以字节为单位的项目数"，因为用户态读取原始 `struct input_event` 值并期望 `read()` 返回方式的字节计数。这种细节对于使用 `kn->kn_data` 调整缓冲区大小的用户态库很重要。

`evdev_notify_event` 中的生产者路径组合子系统支持的每个异步通知机制：

```c
if (client->ec_blocked) {
    client->ec_blocked = false;
    wakeup(client);
}
if (client->ec_selected) {
    client->ec_selected = false;
    selwakeup(&client->ec_selp);
}
KNOTE_LOCKED(&client->ec_selp.si_note, 0);

if (client->ec_sigio != NULL)
    pgsigio(&client->ec_sigio, SIGIO, 0);
```

这是完整的异步生产者：阻塞 `read()` 等待者通过 `wakeup()` 发出信号，`poll()` 和 `select()` 等待者通过 `selwakeup()` 发出信号，kqueue 等待者通过 `KNOTE_LOCKED` 发出信号，注册的 SIGIO 消费者通过 `pgsigio` 发出信号。任何给定消费者恰好看到其中之一，但生产者不需要知道是哪一个；它无条件调用所有机制并让每个机制自己过滤。我们完成本章时 `evdemo` 驱动程序将采用相同的分层生产者。

分离序列是唯一有指导意义的部分。当 evdev 客户端离开时，驱动程序运行：

```c
knlist_clear(&client->ec_selp.si_note, 0);
seldrain(&client->ec_selp);
knlist_destroy(&client->ec_selp.si_note);
```

这正是我们描述的三步规则。结果是，仍然持有此客户端 kqueue 注册的任何用户态进程收获最终 `EV_EOF` 事件，然后看到注册消失；仍然停放在 selinfo 上的任何 `poll()` 等待者醒来并返回；任何即将回调到我们 filterops 的进行中 kqueue 传递在 knlist 内存释放之前安全完成。

弄错顺序将这从干净拆除变成 panic。在 `knlist_clear` 之前 `knlist_destroy` 在非空列表上断言。没有 `seldrain` 的 `knlist_clear` 留下 poll 等待者悬挂。没有前置 `knlist_clear` 的 `seldrain` 会工作但会留下指向即将消失的驱动程序的 kqueue 注册，第一次事件传递尝试会崩溃。遵循该序列。

evdev 示例将我们在本节覆盖的所有内容放在一起：撤销感知的附加、字节计数正确的事件报告、组合的生产者路径，以及尊重生命周期规则的拆除。模仿此模式的驱动程序在生产中表现良好。


### hint 参数：它是什么以及为什么存在

每个 `f_event` 回调接收一个 `long hint` 参数，我们一直安静地设置为零。值得理解该参数的作用，因为它在整个内核中并不总是零。

提示是从生产者通过到过滤器的 cookie。当生产者调用 `KNOTE_LOCKED(list, hint)` 时，框架将相同的 `hint` 值传递给每个过滤器的 `f_event`。完全由生产者和过滤器就值的含义达成一致。框架不解释它。

对于只有一种"就绪"含义的简单驱动程序，零是自然选择，过滤器忽略该参数。对于有多个生产者路径的驱动程序，提示可以区分它们。vnode 过滤器使用非零提示编码 `NOTE_DELETE`、`NOTE_RENAME` 和相关 vnode 级事件，`f_event` 函数测试提示位以决定在传递的事件中设置哪些 `kn->kn_fflags` 位。那超出了普通字符驱动程序需要的范围，但它解释了签名的通用性。

生产者侧是提示值的来源。驱动程序可以调用 `KNOTE_LOCKED(&sc->sc_rsel.si_note, MY_HINT_NEW_DATA)`，过滤器可以根据值切换到不同路径。在实践中，普通驱动程序传递零并保持过滤器简单。

### 传递事件：KNOTE_LOCKED 与 KNOTE_UNLOCKED，深度对比

`/usr/src/sys/sys/event.h` 中的两个传递宏是：

```c
#define KNOTE_LOCKED(list, hint)    knote(list, hint, KNF_LISTLOCKED)
#define KNOTE_UNLOCKED(list, hint)  knote(list, hint, 0)
```

两者都调用 `/usr/src/sys/kern/kern_event.c` 中相同的底层 `knote()` 函数，它遍历 knlist 并在每个 knote 上调用 `f_event`。区别在于第三个参数：`KNF_LISTLOCKED` 说"调用者已经持有 knlist 锁"，而零说"为我获取它。"

在它们之间选择是匹配生产者锁定路径的问题。如果生产者在已经获取驱动程序互斥锁的情况下被调用（因为它从锁定的 ISR 处理程序调用，或从需要锁做自己工作的生产者函数内部调用），`KNOTE_LOCKED` 是正确的。如果生产者在未锁定的情况下被调用（因为它在线程上下文中运行，且锁会专门为通知获取），`KNOTE_UNLOCKED` 是正确的。要避免的错误是在不实际持有锁的情况下调用 `KNOTE_LOCKED`，这在负载下会严重竞争，或者在持有锁的情况下调用 `KNOTE_UNLOCKED`，这会递归并 panic。

一个 ISR 上下文示例有帮助：如果设备中断处理程序调用一个获取 softc 互斥锁、做一些工作并需要通知 kqueue 等待者的下半部函数，最干净的模式是在持有的互斥锁内做工作和 `KNOTE_LOCKED` 调用，然后之后释放锁。互斥锁是 knlist 锁，所以 `KNOTE_LOCKED` 是要使用的。如果通知来自尚未获取锁的线程，线程获取锁、做工作、调用 `KNOTE_LOCKED`，然后释放锁；或者它使用 `KNOTE_UNLOCKED` 并让框架在遍历列表的路上短暂获取锁。

第二个微妙之处是列表为空时 `knote` 的行为。遍历空列表便宜但不是免费的；它仍然获取锁。传递非常高速率通知的驱动程序可以先测试 `KNLIST_EMPTY(list)` 并在没有等待者时跳过 `KNOTE_LOCKED` 调用。宏 `KNLIST_EMPTY`，在 `/usr/src/sys/sys/event.h` 中定义为 `SLIST_EMPTY(&(list)->kl_list)`，为了提示目的是安全的，无需锁就可以读取，因为过时读取的最坏情况是错过一微秒前添加的 knote 上的唤醒，而那个 knote 会在下次传递时注意到。在实践中，这种优化很少值得复杂性，但值得了解。

### 驱动程序 kqfilter 实现中的常见陷阱

在阅读树中 kqueue 感知驱动程序的过程中，少数反复出现的错误模式会出现。提前了解这些陷阱有助于避免它们。

忘记销毁 knlist。在 attach 中调用 `knlist_init_mtx` 但不在 detach 中调用 `knlist_destroy` 的驱动程序泄漏 knlist 的内部状态，更糟糕的是，可能留下活动 knote 悬空。修复是在每个分离路径中包含 clear-drain-destroy 序列。

在 `knlist_clear` 之前调用 `knlist_destroy`。`knlist_destroy` 断言列表为空。如果仍有任何 knote 附加，断言失败且内核 panic。总是先清除。

在未持有锁的情况下使用 `KNOTE_LOCKED`。这是微妙的，因为它大多数时间工作。在负载下，两个生产者可能在 knote 遍历中竞争，框架在遍历期间列表稳定的假设会崩溃。症状通常是 knote 指针损坏或 `f_event` 中的释放后使用。

在 `f_event` 中休眠。框架在调用我们时持有 knlist 锁，这是我们的 softc 互斥锁。在互斥锁下休眠是内核错误。如果 `f_event` 需要不能在 softc 互斥锁下访问的状态，设计就是错误的；将状态移入 softc 或在通知前预计算它。

返回过期的 `kn_data`。`kn->kn_data` 字段应反映过滤器被评估时的状态。在 `d_kqfilter` 中计算 `kn_data` 一次却忘记在 `f_event` 中更新的驱动程序将传递过期的字节计数给用户态。总是在 `f_event` 中重新计算它。

保持 `kn_hook` 指向已释放的内存。如果 `kn_hook` 设置为 softc，且 softc 在 knote 分离之前被释放，下一个 `f_event` 调用将解引用已释放的内存。这是 `knlist_clear` 和 `seldrain` 应该防止的，但只有在以正确顺序并在 softc 释放之前调用时才有效。驱动程序分离入口点中的分离顺序很重要。

只设置一次 `EV_EOF`。`EV_EOF` 是粘性的，一旦设置，用户态就会看到它，但 `f_event` 在 knote 生命周期中被调用多次。如果导致 `EV_EOF` 的条件可能再次变为假（例如，获得新写入者的命名管道），过滤器必须显式清除 `EV_EOF`。`/usr/src/sys/kern/sys_pipe.c` 中的管道过滤器演示了这一点：`filt_piperead` 根据管道的状态设置和清除 `EV_EOF`。

混淆 `f_isfd` 与 `f_attach`。`f_isfd = 1` 意味着过滤器绑定到文件描述符；几乎所有驱动程序过滤器都想要这个。`f_attach = NULL` 意味着"注册路径不需要超出 `d_kqfilter` 已做工作的每 knote 附加回调。"它们是独立的。驱动程序可以同时设置 `f_isfd = 1` 和 `f_attach = NULL`，这是常见情况。

从 `f_event` 返回错误。`f_event` 返回一个 int，但它是一个布尔值：零意味着"未就绪"，非零意味着"就绪。"它不是 errno。从 `f_event` 返回 `EINVAL` 意味着"就绪"，这几乎肯定不是驱动程序想要的。

### kqueue 框架的心智模型

值得暂停一下，组装一个适合我们所学内容的 kqueue 框架的心智模型。不同读者会发现不同模型有帮助；对驱动程序作者有效的一个是这样的。

想象每个驱动程序对象（一个 cdev、一个每客户端状态记录、一个管道、一个 tty）是一个小办公室。办公室有收件箱和发件箱，它们是 knlist。当访客（用户态程序）想要被告知收件箱有新邮件时，他们向办公室注册一张便利贴：他们的 kqueue 文件描述符加上他们关心的过滤器类型。办公室职员（我们的 `d_kqfilter` 回调）拿取便利贴，检查它属于哪个收件箱（`EVFILT_READ` 收件箱或 `EVFILT_WRITE` 发件箱），并将其钉在那里。便利贴记录通知谁（kqueue）和如何通知（`struct filterops` 回调）。

当邮件实际到达时（生产者路径插入一条记录并想要通知），办公室职员遍历收件箱便利贴，对每张检查条件当前是否满足（`f_event` 回调）。如果满足，职员拿起电话拨打访客的 kqueue，传递通知。访客在下次 `kevent(2)` 收获时读取通知。

当访客改变主意不再想要邮件通知时（移除注册），办公室职员拉下便利贴（`f_detach` 回调）。当办公室永久关闭时（驱动程序分离），职员一次性拉下所有便利贴（`knlist_clear`），唤醒任何物理坐在等候室的访客（`seldrain`），然后拆除便利贴公告板（`knlist_destroy`）。

公告板上的锁是驱动程序的 softc 互斥锁。职员在遍历通知、钉通知或拉通知时持有它。这就是为什么 `f_event` 绝不能休眠：职员在遍历列表时不能放下锁，因为其他职员可能带着更新到达。这也是为什么 `KNOTE_LOCKED` 是生产者已经持有锁时的正确调用：职员说"我已经拿着了"让框架跳过不必要的重新获取。

这个模型是简化的，它省略了像 `EV_CLEAR` 边缘语义和 `f_touch` 注册更新这样的复杂性，但它捕获了基本架构。驱动程序拥有公告板；框架拥有便利贴。驱动程序报告就绪；框架处理传递。驱动程序在分离时拆除公告板；框架的便利贴结构作为拆除的一部分被释放。

在阅读其他使用 kqueue 的子系统的代码时记住这幅图景，不熟悉的名称会映射回熟悉的角色。`kqueue_register` 是访客走进来提交便利贴。`knote` 是职员遍历公告板。`f_event` 是每张通知的单独就绪检查。`selwakeup` 是也到达公告板的一般火灾警报。名称不同；形状相同。

### 阅读 kern_event.c：好奇者指南

对于想超越回调的读者，kqueue 框架本身值得一看。`/usr/src/sys/kern/kern_event.c` 大约三千行，看起来令人生畏，但一旦我们知道要找什么，文件的结构是可预测的。

文件顶部附近声明了内置过滤器的静态 filterops 表。`file_filtops` 处理不提供自己 kqfilter 的文件描述符的通用读写过滤器；`timer_filtops` 处理 `EVFILT_TIMER`；`user_filtops` 处理 `EVFILT_USER`；还有更多。这些是框架在启动时安装的 filterops，阅读它们能很好地了解 filterops 表在生产代码中应该是什么样子。

静态声明之后是系统调用入口点：`kqueue`、`kevent` 和遗留变体。它们做参数验证并调度到核心机制。跟踪用户态调用通过内核的读者从这里开始。

核心机制是一组以 `kqueue_` 开头的函数。`kqueue_register` 处理 `EV_ADD`、`EV_DELETE`、`EV_ENABLE`、`EV_DISABLE` 和 `EV_RECEIPT`；它是框架通过 `fo_kqfilter` 调用我们 `d_kqfilter` 的地方。`kqueue_scan` 将就绪事件收获回用户态。`kqueue_acquire` 和 `kqueue_release` 引用计数 kqueue 以实现安全的并发访问。`kqueue_close` 在引用它的最后一个文件描述符关闭时拆除 kqueue。从 `kqueue_register` 顶部通过 `kqueue_expand`、`knote_attach` 和 `fo_kqfilter` 调用跟踪揭示了完整的注册路径。

`knote` 函数本身，大约在文件的三分之二处，是我们通过 `KNOTE_LOCKED` 和 `KNOTE_UNLOCKED` 到达的那个。它遍历 knlist，在每个 knote 上调用 `f_event`，并为任何报告就绪的排队通知。阅读它显示为什么我们的 `f_event` 上的锁断言是必要的，以及框架如何在 kqueue 通知之间交错列表遍历。遍历使用带临时指针的 `SLIST_FOREACH_SAFE`，所以遍历期间的 `f_detach` 不会损坏迭代。那个微妙的细节使并发的分离和传递安全。

再往下是 knlist 机制：`knlist_init`、`knlist_init_mtx`、`knlist_add`、`knlist_remove`、`knlist_cleardel`、`knlist_destroy` 和各种辅助函数。这些是我们一直在调用的函数。阅读它们确认我们一直依赖的锁语义，并显示 `islocked` 参数在辅助函数内部如何被消费。

文件接近末尾是内置过滤器的过滤器实现，名称如 `filt_timerattach`、`filt_user` 和 `filt_fileattach`。这些值得阅读，因为它们是过滤器应该结构化的最接近参考实现的东西。`/usr/src/sys/kern/sys_pipe.c` 中的管道过滤器是另一个好的参考；`/usr/src/sys/kern/uipc_socket.c` 中的套接字 kqueue 支持是第三个。

按顺序通过 `kqueue_register`、`knote` 和 `knlist_remove` 工作的读者将在一个下午结束时理解大部分框架。剩余的机制（自动销毁、定时器实现、进程和信号过滤器、vnode note 掩码）足够专业化，驱动程序作者可以跳过它们，除非有特定需要。本章的其余部分不需要它们中的任何一个。

### 我们尚未使用的驱动程序模式

树中有两种我们没在 `evdemo` 中使用的模式，因为它们不需要，但值得识别，以便在其他地方看到它们的读者知道它们是什么。

第一种是使用 `f_attach` 进行超出 `d_kqfilter` 所做的每 knote 设置。`EVFILT_TIMER` 过滤器使用 `f_attach` 在 knote 首次注册时启动一次性或重复定时器，使用 `f_detach` 停止它。`/usr/src/sys/kern/kern_event.c` 中的 `EVFILT_USER` 过滤器使用 `filt_userattach` 作为空操作，因为 knote 不附加到内核中的任何东西；用户触发的 `NOTE_TRIGGER` 机制完全通过 `f_touch` 处理传递。需要自己的每 knote 状态的驱动程序可以在 `f_attach` 中分配并在 `f_detach` 中释放，使用 `kn_hook` 或 `kn_hookid` 记住指针。几乎没有驱动程序实际需要这个，因为每注册状态通常自然适合 softc。

第二种是 `f_touch`，它拦截 `EV_ADD`、`EV_DELETE` 和 `EV_ENABLE`/`EV_DISABLE` 操作。`/usr/src/sys/kern/kern_event.c` 中的 `filt_usertouch` 函数是 `f_touch` 结构的好参考：它检查 `type` 参数（`EVENT_REGISTER`、`EVENT_PROCESS` 或 `EVENT_CLEAR` 之一）以决定用户态在请求什么并相应更新 knote 的字段。大多数驱动程序过滤器将 `f_touch` 保留为 NULL 并接受框架的默认行为，即在 `EV_ADD` 期间直接在 knote 中存储 `sfflags`、`sdata` 和事件标志。对于不需要注册更新时额外行为的过滤器，默认是正确的。

树使用但我们的驱动程序不用的第三种模式是 knlist 拆除的"kill"变体。`/usr/src/sys/kern/kern_event.c` 中的 `knlist_cleardel` 接受一个 `killkn` 标志，设置时强制每个 knote 离开列表，无论是否仍在使用。`knlist_clear` 是设置了此标志的常见包装器。想要跨事件保留 knote 的驱动程序（例如，将它们重新附加到新对象）可以用 `killkn` false 调用 `knlist_cleardel`，knote 将被取消挂钩但仍保持活动。这几乎从来不是驱动程序想要的。常见情况是 `knlist_clear`，它杀死并释放。

### 关于 EV_CLEAR、EV_ONESHOT 和边触发行为的说明

kqueue 框架通过 `struct kevent` 上的标志支持几种传递模式：

`EV_CLEAR` 使过滤器边触发：一旦 knote 触发，它不会再次触发，直到底层条件从假再次变为真。这是高通量描述符上可读和可写过滤器的常见选择，因为它避免用相同数据的重复通知轰炸用户态。

`EV_ONESHOT` 使过滤器恰好触发一次然后自动删除自己。它对一次性事件有用。

`EV_DISPATCH` 使过滤器每次 `kevent()` 收获最多触发一次，每次触发后自动禁用。用户态通过用 `EV_ENABLE` 重新注册来重新启用它。这是多线程用户态程序的首选模式，它们想确保只有一个线程对每个事件做出反应。

驱动程序的过滤器函数不需要知道这些标志；框架处理它们。驱动程序只报告底层条件是否满足，框架决定如何处理产生的 knote。

### 第 4 节总结

我们现在在驱动程序中有了 `kqueue` 支持。我们添加的总代码不多：一个 `d_kqfilter` 回调、一个 `struct filterops`、两个短过滤器函数和一个生产者中的 `KNOTE_LOCKED()` 调用。复杂性更多在于理解框架而不是编写大量代码。

但我们只覆盖了两个最常见的过滤器，`EVFILT_READ` 和 `EVFILT_WRITE`。本章范围故意排除了更深的 kqueue 主题，如用户定义的过滤器（`EVFILT_USER`）、自定义 `f_touch` 实现以及与 AIO 子系统的交互。这些足够专业化，很少出现在普通驱动程序中，它们会挤占大多数读者需要的材料。如果你确实需要它们，本节的材料让你准备好阅读 `/usr/src/sys/kern/kern_event.c` 的相应部分并理解你发现的内容。

回顾本节覆盖的内容，读者现在应该对在 kqueue 的非正式讨论中容易模糊的几个层次感到舒适。最外层是用户态 API：`kqueue(2)`、`kevent(2)` 以及程序提交和收获的 `struct kevent` 值。中间层是框架：`kqueue_register`、`knote`、`kqueue_scan` 以及将注册匹配到传递的机制。内层是驱动程序契约：`d_kqfilter`、`struct filterops`、`struct knote`、`struct knlist` 以及像 `knlist_init_mtx`、`knlist_add`、`knlist_remove`、`knlist_clear` 和 `knlist_destroy` 这样的少量辅助函数。三个层次通过定义良好的边界通信，理解哪个是哪个是在猜测 kqueue 和理解 kqueue 之间的区别。

我们还走过了三个真实驱动程序实现：`/dev/klog`、tty 子系统和 evdev 输入栈。每个都说明了 kqfilter 契约的不同方面。klog 驱动程序显示了 kqueue 感知驱动程序需要的最小值。tty 子系统显示了如何用两个独立的 knlist 处理两个方向。evdev 驱动程序显示了撤销感知的附加、字节计数正确的事件报告、组合的生产者路径（扇出到多个异步机制）以及严格的 clear-drain-destroy 分离序列。组合这三个模式的适当部分的驱动程序在生产中表现良好，遵循讨论的读者应该能够在树中的其他子系统中看到这些模式时识别它们。

在下一节中，我们转向第三个异步机制 `SIGIO`。与 `poll()` 和 `kqueue()` 不同，它们是拉式通知（用户态询问，内核回答），`SIGIO` 是推式的：每当设备状态改变时，内核向注册的进程发送信号。它更老、更简单，在多线程程序中有一些微妙的问题，但在特定情况下仍然有用，是标准驱动程序工具包的一部分。


## 5. 使用 SIGIO 和 FIOASYNC 的异步信号

第三种经典异步机制是信号驱动 I/O，也称为 `SIGIO` 通知，以它通常使用的信号命名。用户通过打开的文件描述符上的 `FIOASYNC` ioctl 启用它，用 `FIOSETOWN` 设置所有者，并为 `SIGIO` 安装一个处理程序。驱动程序在有相关状态改变时，向注册的所有者发送 `SIGIO`。该信号可以中断所有者中几乎任何系统调用，然后通常服务设备并返回其正常工作。

信号驱动 I/O 比 `kqueue` 更老，不如 `poll` 可扩展，在多线程程序中有一些微妙的问题。它在少数但真实的情况下仍然是正确的机制：想要最简单可能异步通知的单线程程序、使用 `trap` 的 shell 脚本以及已经使用 `SIGIO` 数十年且不会改变的遗留代码。FreeBSD 继续完全支持它，大多数普通字符驱动程序被期望遵守该机制。

### 信号驱动 I/O 在用户态的工作方式

使用 `SIGIO` 的用户程序做三件事。它为 `SIGIO` 安装信号处理程序。它告诉内核哪个进程应该拥有此描述符的信号。它启用异步通知。

代码大致如下：

```c
#include <signal.h>
#include <sys/filio.h>
#include <fcntl.h>
#include <unistd.h>

static volatile sig_atomic_t got_sigio;

static void
on_sigio(int sig)
{
    got_sigio = 1;
}

int
main(void)
{
    int fd = open("/dev/evdemo", O_RDONLY | O_NONBLOCK);

    struct sigaction sa;
    sa.sa_handler = on_sigio;
    sigemptyset(&sa.sa_mask);
    sa.sa_flags = 0;
    sigaction(SIGIO, &sa, NULL);

    int pid = getpid();
    ioctl(fd, FIOSETOWN, &pid);

    int one = 1;
    ioctl(fd, FIOASYNC, &one);

    for (;;) {
        pause();
        if (got_sigio) {
            got_sigio = 0;
            char buf[256];
            ssize_t n;
            while ((n = read(fd, buf, sizeof(buf))) > 0) {
                /* 处理数据 */
            }
        }
    }
}
```

ioctl 的顺序很重要。程序首先安装信号处理程序，以便 `SIGIO` 在到达时不会被忽略。然后用自己的 PID 调用 `FIOSETOWN`（正值表示进程，负值表示进程组），以便驱动程序知道向哪里传递信号。最后用非零值调用 `FIOASYNC` 以启用通知。

一旦启用异步通知，驱动程序中每个会满足 `POLLIN` 的状态改变都会导致向所有者发送 `SIGIO` 信号。程序的处理程序异步运行，设置一个标志，然后返回；主循环然后服务设备。用非阻塞读取将设备排空至空，因为在信号发送和处理程序运行之间，可能已经积累了多个事件。

### FIOASYNC、FIOSETOWN 和 FIOGETOWN Ioctl

打开 `/usr/src/sys/sys/filio.h` 查看 ioctl 定义：

```c
#define FIOASYNC    _IOW('f', 125, int)   /* set/clear async i/o */
#define FIOSETOWN   _IOW('f', 124, int)   /* set owner */
#define FIOGETOWN   _IOR('f', 123, int)   /* get owner */
```

这些是大多数文件描述符处理层已经理解的标准 ioctl。对于普通文件描述符（套接字、管道、pty），内核在不涉及驱动程序的情况下处理它们。然而对于 `cdev`，驱动程序负责实现它们，因为驱动程序拥有 ioctl 操作的状态。

FreeBSD 字符驱动程序中的常规方法是：

`FIOASYNC` 接受一个 `int *` 参数。非零启用异步通知。零禁用它。驱动程序在 softc 中存储该标志并使用它决定是否生成信号。

`FIOSETOWN` 接受一个 `int *` 参数。正值是 PID，负值是进程组 ID，零清除所有者。驱动程序使用 `fsetown()` 记录所有者。

`FIOGETOWN` 接受一个要填入的 `int *` 参数。驱动程序使用 `fgetown()` 检索当前所有者。

### fsetown、fgetown 和 funsetown

所有者跟踪机制在内核中使用一个 `struct sigio`。我们不必直接分配或管理该结构；`fsetown()` 和 `funsetown()` 辅助函数为我们做。公共 API，在 `/usr/src/sys/sys/sigio.h` 和 `/usr/src/sys/kern/kern_descrip.c` 中，由四个函数组成：

```c
int   fsetown(pid_t pgid, struct sigio **sigiop);
void  funsetown(struct sigio **sigiop);
pid_t fgetown(struct sigio **sigiop);
void  pgsigio(struct sigio **sigiop, int sig, int checkctty);
```

驱动程序在 softc 中存储单个 `struct sigio *`。所有四个辅助函数接受指向此指针的指针，因为它们可能作为工作的一部分替换整个结构。辅助函数负责引用计数、锁定和通过 `eventhandler(9)` 在进程退出期间的安全移除。

`fsetown()` 安装新所有者。它期望在被中断调用者的凭据可用时被调用（这在 ioctl 处理程序内部总是如此）。如果目标 PID 为零，它清除所有者。如果目标是正数，它查找进程。如果是负数，它查找进程组。成功返回零或失败返回 errno。

`funsetown()` 清除所有者并释放关联结构。驱动程序在关闭和分离期间调用它，以确保没有留下过时的引用。

`fgetown()` 返回当前所有者为 PID（正值）或进程组 ID（负值），如果没有设置所有者则为零。

`pgsigio()` 向所有者传递信号。第三个参数 `checkctty` 对于不是控制终端的驱动程序应该为零。这是驱动程序在启用异步通知时从生产者路径调用的函数。

### 在 evdemo 中实现 SIGIO

将各部分放在一起，以下是我们添加到驱动程序以支持 `SIGIO` 的内容：

在 softc 中：

```c
struct evdemo_softc {
    /* ... 现有字段 ... */
    struct sigio    *sc_sigio;
    bool             sc_async;
};
```

在 `d_ioctl` 中：

```c
static int
evdemo_ioctl(struct cdev *dev, u_long cmd, caddr_t data, int fflag,
    struct thread *td)
{
    struct evdemo_softc *sc = dev->si_drv1;
    int error = 0;

    switch (cmd) {
    case FIOASYNC:
        mtx_lock(&sc->sc_mtx);
        sc->sc_async = (*(int *)data != 0);
        mtx_unlock(&sc->sc_mtx);
        break;

    case FIOSETOWN:
        error = fsetown(*(int *)data, &sc->sc_sigio);
        break;

    case FIOGETOWN:
        *(int *)data = fgetown(&sc->sc_sigio);
        break;

    default:
        error = ENOTTY;
        break;
    }
    return (error);
}
```

在生产者路径中：

```c
static void
evdemo_post_event(struct evdemo_softc *sc, struct evdemo_event *ev)
{
    bool async;

    mtx_lock(&sc->sc_mtx);
    evdemo_enqueue(sc, ev);
    async = sc->sc_async;
    KNOTE_LOCKED(&sc->sc_rsel.si_note, 0);
    mtx_unlock(&sc->sc_mtx);

    selwakeup(&sc->sc_rsel);
    if (async)
        pgsigio(&sc->sc_sigio, SIGIO, 0);
}
```

在 `d_close` 或分离期间：

```c
static int
evdemo_close(struct cdev *dev, int flags, int fmt, struct thread *td)
{
    struct evdemo_softc *sc = dev->si_drv1;

    funsetown(&sc->sc_sigio);
    /* ... 其他关闭处理 ... */
    return (0);
}
```

让我们遍历各部分。

softc 增加了两个新字段：`sc_sigio`，我们传递给 `fsetown()` 及其同伴的指针，以及 `sc_async`，一个告诉生产者是否启用信号的标志。该标志在某种意义上与"sc_sigio 非空"是冗余的，但保持显式使生产者代码更清晰更快。

`d_ioctl` 处理程序实现三个 ioctl。我们为 `FIOASYNC` 获取 softc 互斥锁，因为我们更新 `sc_async`。我们不为 `FIOSETOWN` 和 `FIOGETOWN` 获取互斥锁，因为 `fsetown()` 和 `fgetown()` 辅助函数有自己的内部锁定，不应该在持有驱动程序锁的情况下调用。

在生产者中，我们在锁下将 `sc_async` 复制到局部变量，以便我们在锁外使用的值是一致的。如果我们在锁释放后简单地读取 `sc->sc_async`，另一个线程可能在此期间改变它，这是一个竞争。在锁下拍快照避免了竞争。

我们在 softc 锁外调用 `pgsigio()`，因为 `pgsigio()` 获取自己的锁，如果嵌套可能创建排序问题。模式与 `selwakeup()` 相同：在锁下更新，放弃，然后传递通知。

在 `d_close` 中，我们调用 `funsetown()` 清除所有者。这也处理了设置所有者的进程已经退出的情况，所以驱动程序不会泄漏 `struct sigio` 分配。如果进程已经退出，`funsetown()` 本质上是无操作；如果没有，调用清理注册。

### 注意事项：多线程程序中的信号语义

信号驱动 I/O 在多线程程序中有众所周知的弱点。主要问题是 POSIX 中的信号发送给进程，而不是特定线程。当内核传递 `SIGIO` 时，进程中任何掩码允许该信号的线程都可能是接收它的那个。对于想让特定线程服务通知的程序，这很不方便。

有变通方法。`pthread_sigmask()` 可用于在除应该服务通知的线程之外的所有线程中阻塞 `SIGIO`。如果想将信号转换为文件描述符上的可读事件，FreeBSD 通过 `kqueue(2)` 提供 `EVFILT_SIGNAL`，它让 kqueue 报告给定信号已传递给进程。FreeBSD 不提供 Linux 特有的 `signalfd(2)` 系统调用。最简单的变通方法，通常也是正确的，是直接将 `kqueue` 用于底层驱动程序事件：线程可以各自拥有单独的 kqueue，每个可以等待它们关心的确切事件，而不必与信号传递规则搏斗。

第二个弱点是信号中断系统调用。在默认 SA 标志下，被中断的系统调用返回 `EINTR`，程序必须检查这个并重试。这足够不寻常，以至于经常在编写时未考虑 `SIGIO` 的程序中产生错误。变通方法是在 `sa_flags` 中设置 `SA_RESTART`，这使内核自动重启被中断的系统调用。

第三个弱点是信号传递相对于程序执行是异步的。在程序正在进行数据结构更新时到达的信号，如果信号处理程序触及相同的结构，可能导致不一致的状态。修复是保持信号处理程序非常简单（设置一个标志，返回），并在主循环中做实际工作。

对于现代程序，`kqueue` 避免了所有这三个问题。对于遗留程序和简单单线程应用程序，`SIGIO` 很好，在驱动程序中实现它是少量代码。

### 真实驱动程序是什么样的

`if_tuntap.c` 驱动程序提供了 SIGIO 处理的代表性示例。在 softc 中：

```c
struct tuntap_softc {
    /* ... */
    struct sigio        *tun_sigio;
    /* ... */
};
```

在 ioctl 处理程序中，驱动程序分别为 `FIOSETOWN` 和 `FIOGETOWN` 调用 `fsetown()` 和 `fgetown()`，并存储 `FIOASYNC` 标志。在生产者路径中（当数据包准备好从接口读取时），驱动程序调用 `pgsigio()`：

```c
if (tp->tun_flags & TUN_ASYNC && tp->tun_sigio)
    pgsigio(&tp->tun_sigio, SIGIO, 0);
```

在关闭路径中，它调用 `funsetown()`。

`evdev/cdev.c` 驱动程序有类似的结构。这些是你在自己的驱动程序中将重用的模式。

### 从 Shell 测试 SIGIO

`SIGIO` 的一个好特性是你可以在 shell 中演示它而不编写任何代码。Bourne 风格的 shell（sh、bash）有一个内置的 `trap` 命令，在信号到达时运行一个动作。结合 `FIOASYNC` ioctl，我们可以在几行中设置一个测试：

```sh
trap 'echo signal received' SIGIO
exec 3< /dev/evdemo
# (在 fd 3 上启用 FIOASYNC 的机制在这里)
# 在另一个终端触发事件并观察 "signal received"
```

难点是没有直接的 shell 级方式来发出 `ioctl`。你需要要么一个小的 C 辅助程序，要么像某些 BSD 提供的 `ioctl(1)` 命令这样的工具，或者在被跟踪的子进程中使用 `truss`。对于本章的实验，我们提供了一个小的 `evdemo_sigio` 程序，它调用正确的 ioctl 然后简单地暂停，让 shell 的 `trap` 处理程序显示信号传递。

### 关于 POSIX AIO 的说明

FreeBSD 还支持 POSIX `aio_read(2)` 和 `aio_write(2)` API。这些超出普通字符驱动程序的范围，普通 `cdev` 驱动程序几乎从不需要实现任何特殊的东西来参与 AIO。本节剩余的小节解释了为什么是这样、AIO 实际上如何在内核内调度请求，以及驱动程序何时（如果有的话）应该考虑 AIO。目的是消除一个常见的困惑来源：当读者在 FreeBSD 文档中看到"文件上的异步 I/O"时，他们读到的是 POSIX AIO，很容易假设驱动程序需要自己的 AIO 机制才能成为一等公民。它不需要。

### AIO 如何调度：fo_aio_queue 和 aio_queue_file

当用户态程序调用 `aio_read(2)` 或 `aio_write(2)` 时，请求进入内核，被验证，并成为一个 `struct kaiocb`（内核 AIO 控制块）。从那里开始的代码路径值得跟踪，因为它解释了为什么字符驱动程序几乎从不需要对 POSIX AIO 做任何事情。

在 `/usr/src/sys/kern/vfs_aio.c` 中，调度在文件操作层完成。相关决策，在 `aio_aqueue` 内部，看起来像这样：

```c
if (fp->f_ops->fo_aio_queue == NULL)
    error = aio_queue_file(fp, job);
else
    error = fo_aio_queue(fp, job);
```

决策在文件操作层而不是 cdev 层做出。如果文件的 `struct fileops` 有自己的 `fo_aio_queue` 函数指针，AIO 委托给它。Vnode 文件操作设置 `fo_aio_queue = vn_aio_queue_vnops`，它将常规文件请求路由到知道如何与底层文件系统通信的路径。相比之下，cdev 文件的 fileops 将 `fo_aio_queue` 留为 NULL，所以 AIO 落入通用 `aio_queue_file` 路径。

`/usr/src/sys/kern/vfs_aio.c` 中的 `aio_queue_file` 尝试两件事。首先，如果底层对象看起来像块设备，它尝试 `aio_qbio`（基于 bio 的路径，在下一小节描述）。其次，如果 bio 路径不适用，它在一个 AIO 工作线程上调度 `aio_process_rw`。`aio_process_rw` 是一个基于守护进程的路径，它只是从 AIO 工作线程同步调用 `fo_read` 或 `fo_write`。换句话说，对于通用 cdev，"异步 I/O"是通过让内核线程代表应用程序执行同步 `read()` 或 `write()` 来实现的。

这就是为什么普通字符驱动程序不需要实现自己的 AIO 钩子。AIO 子系统不通过新入口点调用驱动程序；它调用 `fo_read` 和 `fo_write`，它们又调用驱动程序现有的 `d_read` 和 `d_write`。如果我们的驱动程序已经正确支持阻塞和非阻塞读取，它已经支持 AIO，只是通过工作线程。驱动程序端不需要额外代码。

### 块设备路径：aio_qbio 和 Bio 回调

对于块设备（磁盘、cd 等），工作线程路径效率低下，因为块 I/O 层已经有自己的异步完成机制。FreeBSD 通过 `/usr/src/sys/kern/vfs_aio.c` 中的 `aio_qbio` 利用这一点，它将请求作为 `struct bio` 直接提交到底层设备的 strategy 例程，并安排完成时调用 `aio_biowakeup`。bio 携带一个回指 `struct kaiocb` 的指针，以便完成能找到回到 AIO 框架的路。

`/usr/src/sys/kern/vfs_aio.c` 中的 `aio_biowakeup` 检索 bio 携带的 `struct kaiocb`，计算剩余字节数，并用结果调用 `aio_complete`。`aio_complete` 设置 kaiocb 上的状态和错误字段，将其标记为完成，然后调用 `aio_bio_done_notify`，它扇出到 kaiocb 上的任何 kqueue 注册、`aio_suspend` 中的任何阻塞等待者以及用户态通过 `aiocb.aio_sigevent` 字段请求的任何信号注册。

`aio_biocleanup` 是释放 bio 缓冲区映射并将 bio 本身返回其池的伴随辅助函数。AIO 路径上使用的每个 bio 都通过它，要么在唤醒路径上，要么在提交在多 bio 请求中途失败时的清理循环中。

此路径完全在块 I/O 层内部。不是块设备的字符驱动程序永远不会看到它。块设备驱动程序看到与从任何其他来源看到的完全相同的 bio：驱动程序无法分辨这个特定 bio 来自 `aio_read` 而不是缓冲区缓存页上的 `read`。这就是关键点。AIO 通过重用现有 strategy 契约融入块层，而不是添加并行路径。正确实现 strategy 例程的块驱动程序免费获得 AIO。

### 工作线程路径：aio_process_rw

当 `aio_qbio` 不适用时（几乎所有字符驱动程序都是如此），`aio_queue_file` 落入 `aio_schedule(job, aio_process_rw)`。那将作业放到 AIO 工作队列上。预生成的 AIO 守护线程之一（池大小可通过 `vfs.aio.max_aio_procs` sysctl 调整）选取它，运行 `aio_process_rw`，并执行实际 I/O。

`/usr/src/sys/kern/vfs_aio.c` 中的 `aio_process_rw` 是工作线程路径的核心。它从 kaiocb 的字段准备一个 `struct uio`，在文件上调用 `fo_read` 或 `fo_write`，并将返回值传递给 `aio_complete`。从驱动程序的角度来看，I/O 通过完全普通的读或写调用进入，有一个微妙区别：调用线程是 AIO 守护进程，不是提交请求的进程。用户凭据是正确的，因为 AIO 框架保留了它们，但进程上下文是 AIO 守护进程的。依赖 `curthread` 或 `curproc` 做自己的记账的驱动程序可能会看到令人惊讶的值；不依赖的驱动程序（几乎所有都是）无论调用者是用户自己的线程还是 AIO 守护进程，行为都相同。

工作线程路径在硬件意义上不是"异步"的。它在 API 意义上是"异步"的：用户态没有阻塞。替换发生在线程边界，而不是 I/O 边界，所以慢速设备在服务请求时仍然占用一个 AIO 工作线程。对于大多数 cdev 驱动程序，这正是正确的权衡。用户态得到它想要的编程模型；内核使用工作线程完成工作；驱动程序不做任何特殊的事情。如果驱动程序已经正确遵守 `O_NONBLOCK`，工作线程甚至可以向它提交非阻塞请求并通过正常路径将 `EAGAIN` 返回给用户态。

### 完成：aio_complete、aio_cancel 和 aio_return

一旦调用了 `aio_complete`，kaiocb 进入其完成状态。用户态程序最终将调用 `aio_return(2)` 检索字节计数，或 `aio_error(2)` 检查错误代码，或等待 kqueue 或信号被告知作业已完成。这些调用通过其用户态指针查找 kaiocb 并返回 `aio_complete` 设置的字段。

从驱动程序的角度来看，返回路径上没有什么要做。驱动程序不拥有 kaiocb，不释放它，也不直接发出完成信号。完成由 `aio_complete` 宣布；`aio_return` 是完全由内核的 AIO 层处理的用户态关注点。驱动程序的工作在它满足 `fo_read` 或 `fo_write` 调用时结束，或者在 strategy 例程对 bio 调用 `biodone` 时结束。

对于取消，`/usr/src/sys/kern/vfs_aio.c` 中的 `aio_cancel` 最终调用 `aio_complete(job, -1, ECANCELED)`。就是这样。作业被标记为完成并带有错误，通常的唤醒路径触发。驱动程序不需要了解取消，除非它实现了自己的长期运行请求持有队列，这是例外的。

一个区别值得明确指出。`aio_cancel` 是 AIO 内部使用的内核侧取消函数；它不是用户态系统调用。面向用户态的 `aio_cancel(2)` 接受一个文件描述符和一个指向 `aiocb` 的指针，要求内核取消一个或所有待处理请求。在内部，那最终在每个匹配的 kaiocb 上调用内核 `aio_cancel`。命名有点不幸；阅读源代码使哪个是哪个变得明显。

### EVFILT_AIO：AIO 如何使用 kqueue

值得了解（尽管不需要操作）的是 `EVFILT_AIO` 存在。声明在 `/usr/src/sys/sys/event.h` 中并实现在 `/usr/src/sys/kern/vfs_aio.c` 中作为 `aio_filtops` 表，它让用户态程序在 kqueue 上等待 AIO 完成。filterops 在 AIO 模块加载时通过 `kqueue_add_filteropts(EVFILT_AIO, &aio_filtops)` 注册一次。每个 kaiocb 的回调是：

```c
static const struct filterops aio_filtops = {
    .f_isfd   = 0,
    .f_attach = filt_aioattach,
    .f_detach = filt_aiodetach,
    .f_event  = filt_aio,
};
```

这里 `f_isfd` 为零，因为 AIO 注册以 kaiocb 为键，不是文件描述符。`filt_aioattach` 将 knote 链接到 kaiocb 自己的 knlist。`filt_aio` 通过检查 kaiocb 是否被标记为完成来报告完成状态。kaiocb 的 knlist 的 `kl_autodestroy` 字段被设置，所以 knlist 可以在 kaiocb 被释放时自动拆除。这是树中实际使用 `kl_autodestroy` 的少数地方之一，这使得 `vfs_aio.c` 值得阅读，如果你需要了解该标志是如何使用的。

这些都不是驱动程序的事。AIO 模块在启动时注册 `EVFILT_AIO` 一次，从那时起用户态可以通过 kqueue 等待完成，无需任何进一步的驱动程序参与。想要用户态能够通过 kqueue 等待驱动程序发起的事件的驱动程序通过 `EVFILT_READ` 或 `EVFILT_WRITE` 来做，而不是通过 `EVFILT_AIO`。

### 为什么 kqueue 是大多数驱动程序的正确答案

综合起来，对驱动程序作者的指导很清楚。

如果驱动程序是块设备，内核已经通过 `aio_qbio` 将 AIO 接入 bio 路径。不需要额外工作。正确服务其 strategy 例程的块驱动程序也正确服务 AIO。

如果驱动程序是发出事件并希望用户态在不阻塞线程的情况下等待它们的字符设备，正确的机制是 `kqueue`。用户态在驱动程序的文件描述符上注册 `EVFILT_READ` 或 `EVFILT_WRITE`，驱动程序通过 `KNOTE_LOCKED` 通知等待者。这就是我们贯穿本章一直在构建的，也是我们阅读过的所有驱动程序所做的。

如果驱动程序是用户态程序员出于可移植性原因想用 `aio_read(2)` 调用的字符设备，驱动程序端不需要额外工作。AIO 将通过工作线程服务请求，工作线程调用驱动程序现有的 `d_read`。用户态得到它想要的可移植性；驱动程序保持简单。

驱动程序可能考虑实现 `d_aio_read` 或 `d_aio_write` 的唯一时间是它有一个高性能、真正异步的硬件路径，可以在不阻塞工作线程的情况下完成工作，并且工作线程回退的成本会令人望而却步。这在普通驱动程序中极其罕见，确实有这种路径的驱动程序（主要是存储驱动程序）通常通过块层而不是作为 cdev 暴露它。

简而言之：对于 cdev 驱动程序，"实现 AIO"几乎总是意味着"实现 kqueue。"剩余的 AIO 机制属于内核，不属于我们。这就是我们想结束本章这一部分的方式，因为它闭合了循环：在四种异步机制（poll、kqueue、SIGIO、AIO）中，需要最多驱动程序代码的是 kqueue，需要最少的是 AIO。因此本章将时间花在了重要的机制上。

### 阅读 vfs_aio.c：指南

对于想跟踪 AIO 路径通过内核的读者，`/usr/src/sys/kern/vfs_aio.c` 组织如下。

文件顶部附近，讨论了 `struct kaiocb` 和 `struct kaioinfo`（通过周围代码中的注释，因为结构本身声明在 `/usr/src/sys/sys/aio.h` 中）。接下来出现 `filt_aioattach`/`filt_aiodetach`/`filt_aio` 一组静态函数和 `aio_filtops` 表。这些是 `EVFILT_AIO` 的 kqueue 集成。

之后是 SYSINIT 和模块注册，`aio_onceonly` 做一次性设置，包括 `kqueue_add_filteropts(EVFILT_AIO, &aio_filtops)`。这是系统范围 `EVFILT_AIO` 过滤器安装的地方。没有驱动程序参与；AIO 模块独立完成。

文件中间部分是 AIO 的核心：`aio_aqueue`（系统调用层入口点）、`aio_queue_file`（通用调度器）、`aio_qbio`（基于 bio 的路径）、`aio_process_rw`（工作线程路径）、`aio_complete`（完成宣布）和 `aio_bio_done_notify`（唤醒扇出）。从 `aio_aqueue` 依次通过它们中的每一个跟踪，映射了 AIO 请求从提交到完成的生命周期。

完成信号函数包括 `aio_bio_done_notify`，它遍历 kaiocb 的 knlist 并在任何注册的 `EVFILT_AIO` knote 上触发 `KNOTE_UNLOCKED`，唤醒 `aio_suspend` 中阻塞的任何线程，并通过 `pgsigio` 传递任何注册的信号。这是我们在 evdev 驱动程序中看到的组合生产者路径的 AIO 类似物。

取消在 `aio_cancel` 和系统调用层 `kern_aio_cancel` 中。kaiocb 上的 `aio_cancel` 简单地调用 `aio_complete(job, -1, ECANCELED)`，它将作业推过与成功作业相同的完成路径。用户态看到 `ECANCELED` 而不是字节计数。

文件以 `aio_read`、`aio_write`、`aio_suspend`、`aio_cancel`、`aio_return`、`aio_error` 和朋友的系统调用实现结束，加上 `lio_listio` 批量提交。这些最终都调用到文件中间的核心调度器中。

通过 `aio_aqueue` 到 `aio_queue_file`，通过 `aio_qbio` 或 `aio_process_rw`，然后通过 `aio_complete` 回到 `aio_bio_done_notify` 跟踪用户态 `aio_read` 的读者已经从头到尾看到了整个 AIO 路径。文件很长，但结构是规则的，与驱动程序相关的部分是整体的一小部分。

### 驱动程序端清单

既然我们已经讨论了 AIO 对驱动程序的要求和不要求，这里有一个简短的清单，驱动程序作者可以用作快速参考。

对于只需要基本可读事件通知的 cdev 驱动程序，AIO 不需要做任何事情。实现 `d_read`，实现 `d_poll` 或 `d_kqfilter` 用于非阻塞通知，用户态可以通过 AIO 工作线程使用 `aio_read(2)`，无需额外驱动程序代码。

对于想要对使用 AIO 获得可移植性的用户态程序友好的 cdev 驱动程序，同样的答案适用：不需要额外的东西。AIO 工作线程处理它。

对于块设备驱动程序，bio 层通过 `aio_qbio` 和 `aio_biowakeup` 处理 AIO。正确服务其 strategy 例程的块驱动程序也正确服务 AIO。同样，不需要额外的东西。

对于有真正异步硬件路径并想在不经过工作线程的情况下通过 AIO 暴露它的驱动程序，`cdevsw` 上的 `d_aio_read` 和 `d_aio_write` 钩子存在，但足够罕见，实现它们超出了本章范围。这样的驱动程序应该研究 `/usr/src/sys/kern/vfs_aio.c` 中的文件操作 `fo_aio_queue` 机制以及使用它的少数子系统。

对于其他所有驱动程序，答案更简单：实现 kqueue，让用户态以高效方式等待事件，并将 AIO 视为内核在没有驱动程序参与的情况下处理的用户态便利。

### 第 5 节总结

我们现在在驱动程序中有了三个独立的异步通知机制：`poll()`、`kqueue()` 和 `SIGIO`。每个单独来看相对较小，每个都可以在不干扰其他的情况下实现。每种情况下的模式相同：在等待者路径注册兴趣，在生产者路径传递通知，并小心锁定和清理。

但这三种机制假设驱动程序有一个明确定义的"事件就绪"概念。到目前为止，我们的讨论在事件实际上是什么方面一直是抽象的。在下一节中，我们看看驱动程序如何在内部组织其事件，以便单个 `read()` 调用可以产生干净、类型良好的记录而不是原始硬件状态。内部事件队列是将整个异步设计绑在一起的部分。


## 6. 内部事件队列和消息传递

到目前为止，我们将"事件就绪"视为一个模糊的条件。在真实驱动程序中，该条件通常是具体的：内部队列中有一条记录。生产者插入记录，消费者读取它们，异步通知机制告诉消费者队列何时获得或失去记录。把队列弄对是使驱动程序其余部分简单的原因。

事件队列有几个属性使其区别于原始字节缓冲区。每个条目是一个结构化记录，不是字节流：一个带载荷的类型化事件。条目被完整传递，不是部分传递：读取者要么获得完整记录要么不获得记录。队列有有界大小，所以生产者必须有队列填满时发生什么的策略：丢弃最旧的、丢弃最新的、报告错误或等待空间。队列按顺序消费：事件以它们被插入的顺序传递，除非设计显式允许否则不是这样。

仔细设计队列在整个驱动程序中都有回报。看到良好类型记录流的读取者可以编写简单、健壮的用户态代码。知道队列溢出策略的生产者可以在事件到达速度快于消费速度时做出明智决定。异步通知机制（`poll`、`kqueue`、`SIGIO`）都变得更干净，因为它们每一个都可以用队列空性而不是任意每设备状态来表达其条件。

### 设计事件记录

第一个决定是单个事件看起来像什么。我们的 `evdemo` 驱动程序的最小记录：

```c
struct evdemo_event {
    struct timespec ev_time;    /* 时间戳 */
    uint32_t        ev_type;    /* 事件类型 */
    uint32_t        ev_code;    /* 事件代码 */
    int64_t         ev_value;   /* 事件值 */
};
```

这镜像了像 `evdev` 这样的真实事件接口的布局，这不是偶然：一个时间戳加上一个（类型、代码、值）三元组足以描述大多数事件流，从键盘按键到传感器读数到游戏控制器上的按钮事件。时间戳让用户态重建事件何时发生，而不管它何时被消费，这对延迟敏感的应用程序很重要。

需要更多结构的驱动程序可以添加字段，但保持记录固定大小的纪律值得捍卫。固定大小的记录使队列的内存管理容易，使读取路径成为简单的复制，并避免了记录有可变长度时出现的 ABI 问题。

### 环形缓冲区

队列本身可以是固定容量的简单环形缓冲区：

```c
#define EVDEMO_QUEUE_SIZE 64

struct evdemo_softc {
    /* ... */
    struct evdemo_event sc_queue[EVDEMO_QUEUE_SIZE];
    u_int               sc_qhead;  /* 下次读取位置 */
    u_int               sc_qtail;  /* 下次写入位置 */
    u_int               sc_nevents;/* 排队事件计数 */
    u_int               sc_dropped;/* 溢出计数 */
    /* ... */
};

static inline bool
evdemo_queue_empty(const struct evdemo_softc *sc)
{
    return (sc->sc_nevents == 0);
}

static inline bool
evdemo_queue_full(const struct evdemo_softc *sc)
{
    return (sc->sc_nevents == EVDEMO_QUEUE_SIZE);
}

static void
evdemo_enqueue(struct evdemo_softc *sc, const struct evdemo_event *ev)
{
    mtx_assert(&sc->sc_mtx, MA_OWNED);

    if (evdemo_queue_full(sc)) {
        /* 溢出策略：丢弃最旧的。 */
        sc->sc_qhead = (sc->sc_qhead + 1) % EVDEMO_QUEUE_SIZE;
        sc->sc_nevents--;
        sc->sc_dropped++;
    }

    sc->sc_queue[sc->sc_qtail] = *ev;
    sc->sc_qtail = (sc->sc_qtail + 1) % EVDEMO_QUEUE_SIZE;
    sc->sc_nevents++;
}

static int
evdemo_dequeue(struct evdemo_softc *sc, struct evdemo_event *ev)
{
    mtx_assert(&sc->sc_mtx, MA_OWNED);

    if (evdemo_queue_empty(sc))
        return (-1);

    *ev = sc->sc_queue[sc->sc_qhead];
    sc->sc_qhead = (sc->sc_qhead + 1) % EVDEMO_QUEUE_SIZE;
    sc->sc_nevents--;
    return (0);
}
```

关于这段代码有几件事值得指出。

我们使用简单的模运算环而不是链表。这保持内存占用固定，避免事件时间的分配，并使队列从缓存行角度来看无锁（每次操作两次读取和一次写入）。大多数有此模式的驱动程序使用环。

我们单独跟踪 `sc_nevents` 而不是头和尾指针。单独使用头和尾，没有计数，会导致"空"和"满"之间的经典歧义：当头等于尾时，队列可能是任一状态。计数字段解决歧义并使快速路径廉价。

我们在 `evdemo_enqueue` 中内置了溢出策略。当队列满时，我们丢弃最旧的事件。对于最近事件比陈旧事件更有价值的事件流，这是正确的策略；安全日志或指标流可能更喜欢相反的方式。我们还递增 `sc_dropped` 以便用户态可以知道有多少事件丢失。

`evdemo_enqueue` 和 `evdemo_dequeue` 都断言持有 softc 互斥锁。这是一个结构性安全网：如果调用者忘记获取锁，断言在调试内核上触发并准确指向错误的调用点。没有断言，错误可能只在罕见时机下表现为静默队列损坏。

### 读取路径

有了队列，同步 `read()` 处理程序变得简短：

```c
static int
evdemo_read(struct cdev *dev, struct uio *uio, int flag)
{
    struct evdemo_softc *sc = dev->si_drv1;
    struct evdemo_event ev;
    int error = 0;

    while (uio->uio_resid >= sizeof(ev)) {
        mtx_lock(&sc->sc_mtx);
        while (evdemo_queue_empty(sc) && !sc->sc_detaching) {
            if (flag & O_NONBLOCK) {
                mtx_unlock(&sc->sc_mtx);
                return (error ? error : EAGAIN);
            }
            error = cv_wait_sig(&sc->sc_cv, &sc->sc_mtx);
            if (error != 0) {
                mtx_unlock(&sc->sc_mtx);
                return (error);
            }
        }
        if (sc->sc_detaching) {
            mtx_unlock(&sc->sc_mtx);
            return (0);
        }
        evdemo_dequeue(sc, &ev);
        mtx_unlock(&sc->sc_mtx);

        error = uiomove(&ev, sizeof(ev), uio);
        if (error != 0)
            return (error);
    }
    return (0);
}
```

模式是标准的：在调用者还有缓冲区空间时循环，队列为空时等待记录，在锁下出队一个，释放锁，通过 `uiomove(9)` 复制出去。我们通过在队列为空时返回 `EAGAIN` 来处理 `O_NONBLOCK`，并通过返回零（文件结束）来处理分离，以便读取者可以干净终止。

`cv_wait_sig()` 调用是一个也在信号传递时返回的条件变量等待，所以在 `read()` 中阻塞的读取者可以被 `SIGINT` 或其他信号中断。这是你可能从前面同步章节记得的可中断等待模式。条件变量从生产者路径发信号，我们接下来看。

### 集成生产者路径

生产者现在有三件事要做：将事件入队，通过条件变量发出任何阻塞读取者的信号，并通过我们研究过的三种机制传递异步通知：

```c
static void
evdemo_post_event(struct evdemo_softc *sc, struct evdemo_event *ev)
{
    bool async;

    mtx_lock(&sc->sc_mtx);
    evdemo_enqueue(sc, ev);
    async = sc->sc_async;
    cv_broadcast(&sc->sc_cv);
    KNOTE_LOCKED(&sc->sc_rsel.si_note, 0);
    mtx_unlock(&sc->sc_mtx);

    selwakeup(&sc->sc_rsel);
    if (async)
        pgsigio(&sc->sc_sigio, SIGIO, 0);
}
```

这是生产者的规范形状。所有状态更新和所有锁内通知发生在 softc 互斥锁内部；锁外通知发生在外面。顺序很重要：锁内的 `cv_broadcast` 和 `KNOTE_LOCKED` 在我们放弃锁之前发生，锁外的 `selwakeup` 和 `pgsigio` 在之后发生。

一个细节是使用 `cv_broadcast()` 而不是 `cv_signal()`。如果多个读取者阻塞在 `read()` 中，我们通常想唤醒所有它们，以便每个都可以尝试声明一条记录。用 `cv_signal()` 我们只唤醒一个，其他的保持睡眠直到另一个事件到达。在单读取者设计中 `cv_signal()` 没问题；在一般情况下 `cv_broadcast()` 更安全。

### Poll 和 Kqueue 集成

内部事件队列的美妙在于 `d_poll` 和 `d_kqfilter` 就队列状态而言变成单行代码：

```c
static int
evdemo_poll(struct cdev *dev, int events, struct thread *td)
{
    struct evdemo_softc *sc = dev->si_drv1;
    int revents = 0;

    mtx_lock(&sc->sc_mtx);
    if (events & (POLLIN | POLLRDNORM)) {
        if (!evdemo_queue_empty(sc))
            revents |= events & (POLLIN | POLLRDNORM);
        else
            selrecord(td, &sc->sc_rsel);
    }
    if (events & (POLLOUT | POLLWRNORM))
        revents |= events & (POLLOUT | POLLWRNORM);
    mtx_unlock(&sc->sc_mtx);

    return (revents);
}

static int
evdemo_kqread(struct knote *kn, long hint)
{
    struct evdemo_softc *sc = kn->kn_hook;

    mtx_assert(&sc->sc_mtx, MA_OWNED);

    kn->kn_data = sc->sc_nevents;
    if (sc->sc_detaching) {
        kn->kn_flags |= EV_EOF;
        return (1);
    }
    return (!evdemo_queue_empty(sc));
}
```

可读过滤器将 `kn->kn_data` 报告为排队事件的数量，每当队列非空时返回 true。用户态程序在 `ev.data` 中看到 `kn_data`，可以知道有多少事件可用而不必已经调用 `read()`。这是 kqueue API 的一个小而有用的特性，支持它不花费我们什么。

### 通过 sysctl 暴露队列指标

诊断友好的驱动程序通过 `sysctl(9)` 暴露其队列状态。对于 `evdemo` 我们添加几个计数器：

```c
SYSCTL_NODE(_dev, OID_AUTO, evdemo, CTLFLAG_RW, 0, "evdemo driver");

SYSCTL_UINT(_dev_evdemo, OID_AUTO, qsize, CTLFLAG_RD,
    &evdemo_qsize, 0, "queue capacity");
SYSCTL_UINT(_dev_evdemo, OID_AUTO, qlen, CTLFLAG_RD,
    &evdemo_qlen, 0, "current queue length");
SYSCTL_UINT(_dev_evdemo, OID_AUTO, dropped, CTLFLAG_RD,
    &evdemo_dropped, 0, "events dropped due to overflow");
SYSCTL_UINT(_dev_evdemo, OID_AUTO, posted, CTLFLAG_RD,
    &evdemo_posted, 0, "events posted since attach");
SYSCTL_UINT(_dev_evdemo, OID_AUTO, consumed, CTLFLAG_RD,
    &evdemo_consumed, 0, "events consumed by read(2)");
```

这些可以在多核系统上变成 `counter(9)` 计数器以获得缓存友好性，但简单的 `uint32_t` 对教学目的来说就可以了。有了这些计数器，`sysctl dev.evdemo` 调用一目了然地显示队列的运行时状态，这在调试似乎错过事件或丢弃事件的驱动程序时是无价的。

### 溢出策略：设计讨论

我们的代码在队列填满时丢弃最旧的事件。让我们思考什么时候这是正确的选择，什么时候不是。

丢弃最旧的对于最近事件比旧事件更有价值的情况是正确的。用户界面事件队列是一个好例子：醒来发现有一百个按键的程序通常关心最近那些，而不是五分钟前的那些。每个记录都有时间戳的遥测流类似：旧记录是陈旧的。

丢弃最新的对于队列代表不能有间隙的分类账的情况是正确的。安全日志绝不应该因溢出而丢失事件；它应该拒绝记录最新事件（并递增"丢弃"计数器）而不是静默重写历史。

阻塞生产者对于生产者实际上可以等待的情况是正确的。生产者是中断处理程序的驱动程序不能阻塞；生产者是用户空间写入调用的驱动程序可以。如果生产者可以等待，那么满队列成为减慢生产者以匹配消费者的背压，这通常正是你想要的。

返回错误对于调用者需要立即知道命令是否成功的请求-响应协议是正确的。这在 ioctl 路径中比在事件队列中更常见，但它是一个有效的策略。

常见错误是在不考虑哪一种适合设备的情况下选择策略。当底层数据是安全日志时丢弃旧事件的驱动程序将丢失证据。当 UI 需要响应性时丢弃新事件的驱动程序将感觉滞后。选择正确的策略是一个设计决定，值得在驱动程序的注释中记录下来，以便未来的维护者理解你为什么选择你所选择的。

### 避免部分读取

一个小而重要的细节：读取路径必须要么传递完整事件要么不传递事件。它绝不能复制半个事件并返回短读取计数，因为用户态调用者然后不得不跨多次调用重建事件，这是脆弱且容易出错的。

强制这一点的最简单方式是循环顶部的守卫：

```c
while (uio->uio_resid >= sizeof(ev)) {
    /* ... */
}
```

如果用户缓冲区剩余字节少于一个事件，我们简单地停止。调用者恰好获得尽可能多的完整事件。如果调用者传递零长度缓冲区，我们立即返回零字节，这是空读取的约定。

### 处理事件合并

有些驱动程序有合并事件的正当理由。如果键盘为同一个键产生"按下"紧接着"释放"，驱动程序可能想将它们折叠成单个"点击"事件以节省队列空间。我们的建议是在大多数情况下抵制这种诱惑。合并改变了事件语义，可能困惑期望原始事件编写的用户态程序。

在合并合理的地方（例如，以保留最终位置的方式合并鼠标移动），仔细实现它并记录下来。合并逻辑应该存在于入队路径中，而不是消费路径中，以便所有消费者看到一致的行为。

### 第 6 节总结

内部事件队列是将异步机制绑在一起的东西。每个通知、每个可读检查、每个 kqueue 过滤器、每个 SIGIO 传递：所有这些都归结为"队列是否非空？"一旦队列就位，驱动程序的其余部分变成接线问题，而不是设计问题。

在下一节中，我们看看在单个驱动程序中组合 `poll`、`kqueue` 和 `SIGIO` 的设计模式，以及确保组合正确的锁定审计。单独添加每个机制是容易的部分。让它们全部协同工作，用一个生产者和多个同时的不同类型等待者，是真实驱动程序工程发生的地方。


## 7. 组合异步技术

到目前为止我们逐个查看了 `poll`、`kqueue` 和 `SIGIO`，每个在自己的节中，有自己的锁定规则和唤醒模式。在真实驱动程序中，三种机制共存。单个生产者路径必须在特定顺序、特定锁下唤醒条件变量睡眠者、poll 等待者、kqueue knote 和信号所有者，而不丢弃任何唤醒且不死锁。

本节是关于把那个组合做对。它很大程度上是一个回顾和整合：我们已经单独看到了每个机制，现在我们看到了它们在一起。回顾值得做是因为机制之间的交互正是驱动程序错误喜欢隐藏的地方。单独使用一种机制不会导致可见问题的锁定顺序或通知时序的微小差异，在分层的几种机制时可能导致唤醒丢失或死锁。

### 何时使用每种机制

支持所有三种机制的驱动程序让其用户态客户端为工作选择正确的工具。三种机制有不同的优势：

`poll` 和 `select` 最具可移植性。需要在广泛的 UNIX 系统上不变运行的用户态程序会使用 `poll`。驱动程序应该支持 `poll`，因为它是最低共同点，而且实现它很便宜。

`kqueue` 最高效且最灵活。监视数千个描述符的用户态程序应该使用 `kqueue`。驱动程序应该支持 `kqueue`，因为它是新 FreeBSD 代码的首选机制，而且大多数关心性能的应用程序会选择它。

`SIGIO` 对于特定类别的程序最简单：使用 `trap` 的 shell 脚本、想要最简单可能通知的小型单线程程序以及遗留代码。驱动程序应该支持 `SIGIO`，因为工作量最小且支持的用例是真实的。

在实践中，几乎每个事件驱动设备的字符驱动程序都应该实现所有三种。代码很少，维护很低，用户态灵活性很高。

### 生产者路径模板

支持所有三种机制的驱动程序的规范生产者路径是：

```c
static void
driver_post_event(struct driver_softc *sc, struct event *ev)
{
    bool async;

    mtx_lock(&sc->sc_mtx);
    enqueue_event(sc, ev);
    async = sc->sc_async;
    cv_broadcast(&sc->sc_cv);
    KNOTE_LOCKED(&sc->sc_rsel.si_note, 0);
    mtx_unlock(&sc->sc_mtx);

    selwakeup(&sc->sc_rsel);
    if (async)
        pgsigio(&sc->sc_sigio, SIGIO, 0);
}
```

这个模板的每个部分都有其位置的原因。

`mtx_lock` 获取 softc 互斥锁。这是序列化驱动程序中所有状态转换的单个锁，所有读取者和写入者都遵守它。

`enqueue_event` 在锁内部。队列是共享状态，对它的任何更新必须相对于其他更新和状态读取是原子的。

`async = sc->sc_async` 在锁内部。这捕获异步标志的一致快照，以便我们可以在锁外使用它而不竞争。

`cv_broadcast` 在锁内部。条件变量要求在发信号时持有关联的互斥锁。信号立即传递，但阻塞线程的实际唤醒在互斥锁释放时发生。

`KNOTE_LOCKED` 在锁内部。它遍历 knote 列表并传递 kqueue 通知，它期望 knlist 的锁（即我们的 softc 互斥锁）被持有。

`mtx_unlock` 释放 softc 互斥锁。此点之后我们在临界区之外。

`selwakeup` 在锁外部。这是 `selwakeup` 的规范排序：它绝不能在任意驱动程序锁内部调用，因为它获取自己的内部锁。

`pgsigio` 由于相同原因在锁外部。

这个顺序是最不容易出错的安排。许多变体是可能的，但偏离此模式需要有特定原因的正当理由。

### 锁定顺序

有了四个不同的通知调用和一个状态更新，锁定顺序很重要。让我们梳理涉及哪些锁。

首先获取 softc 互斥锁，跨越状态更新和锁内通知持有它。

`cv_broadcast` 不获取我们已经持有的锁之外的任何额外锁。

`KNOTE_LOCKED` 评估每个 knote 的 `f_event` 回调。回调在持有 knlist 的锁（我们的 softc 互斥锁）的情况下执行。这些回调绝不能尝试获取任何额外锁，因为这样做会创建一个其他路径（比如 `d_poll` 中的消费者）可能以相反顺序获取的嵌套获取。在实践中，`f_event` 回调只读取状态，这正是我们设计的。

`selwakeup` 获取 selinfo 的内部互斥锁并遍历停放线程的列表，唤醒它们。这在 softc 互斥锁之外完成。在内部，`selwakeup` 也遍历 selinfo 的 knote 列表，但那已经被我们之前的 `KNOTE_LOCKED` 调用处理了；做两次无害但浪费，所以我们在有锁时做 `KNOTE_LOCKED`，让 `selwakeup` 只处理线程列表。

`pgsigio` 获取信号相关锁并向拥有进程或进程组传递信号。这在 softc 互斥锁之外。

锁定顺序规则是：首先 softc 互斥锁，绝不在 selinfo 或信号锁内嵌套。只要我们遵循此顺序，我们就不会死锁。

### 消费者路径

三个消费者路径中的每一个都以一致的方式使用 softc 互斥锁：

```c
/* 条件变量消费者：d_read */
mtx_lock(&sc->sc_mtx);
while (queue_empty(sc))
    cv_wait_sig(&sc->sc_cv, &sc->sc_mtx);
dequeue(sc, ev);
mtx_unlock(&sc->sc_mtx);

/* Poll 消费者：d_poll */
mtx_lock(&sc->sc_mtx);
if (queue_empty(sc))
    selrecord(td, &sc->sc_rsel);
else
    revents |= POLLIN;
mtx_unlock(&sc->sc_mtx);

/* Kqueue 消费者：f_event */
/* 由 kqueue 框架在已持有 softc 互斥锁的情况下调用 */
return (!queue_empty(sc));

/* SIGIO 消费者：完全在用户态处理；驱动程序
 * 只发送信号，从不消费它 */
```

所有三个消费者在 softc 互斥锁下检查队列。这就是关闭生产者状态更新和消费者检查之间的竞争的东西：如果生产者有锁，消费者等待并看到更新后的状态；如果消费者有锁，生产者等待并在消费者注册后发布。

### 常见陷阱

一些特定错误足够常见，值得显式命名。

**在生产者中忘记其中一个通知调用。** 规范排序看起来像样板序列，很容易漏掉四个调用中的一个。只测试一种机制的测试会通过，但其他机制会坏掉。代码审查和自动化测试在这里有帮助。

**在 `selwakeup` 或 `pgsigio` 期间持有锁。** 本章的建议是在这些调用之前释放锁。一些驱动程序意外持有锁（例如，因为生产者深处处于一个难以重构的锁定-解锁-锁定-解锁模式）。结果是潜在的死锁，只在特定锁被不同路径持有时才会显现。

**调用 `cv_signal` 而不是 `cv_broadcast`。** 单读取者驱动程序可以使用 `cv_signal`。允许多读取者的驱动程序必须使用 `cv_broadcast`，因为只有一个被信号的等待者会成功出队事件，其他必须看到更新后的状态重新休眠。如果你选择 `cv_signal` 然后稍后允许多读取者，你就引入了一个只在竞争下出现的潜在唤醒丢失。

**在 attach 时忘记 `knlist_init_mtx`。** 从不初始化其 knlist 的驱动程序会在第一次 `KNOTE_LOCKED` 调用时崩溃，因为 knlist 的锁函数指针为空。症状是在 `knote()` 内部的空指针解引用，如果你在重构中忘记了初始化调用，这可能会令人困惑。

**在关闭时忘记 `funsetown`。** 启用 `FIOASYNC` 然后退出而不关闭 fd 的进程留下过时的 `struct sigio`。内核通过为我们调用 `funsetown` 的 `eventhandler(9)` 处理进程退出，所以这通常是安全的，但在关闭期间泄漏结构仍然是一个错误。

**在分离时忘记 `seldrain` 和 `knlist_destroy`。** 当设备离开时，必须唤醒停放在 selinfo 上的等待者。忘记这一点留下等待者永远休眠，并可能在 selinfo 被释放时 panic 内核。

### 测试组合设计

测试支持所有三种机制的驱动程序的最好方式是并行运行三个用户态程序：

一个基于 `poll` 的读取者，监视事件并打印它们。

一个基于 `kqueue` 的读取者，用 `EVFILT_READ` 做同样的事。

一个基于 `SIGIO` 的读取者，启用 `FIOASYNC` 并在每个信号上打印。

以已知速率在驱动程序中触发事件并验证所有三个读取者都看到它们。如果任何读取者滞后或错过事件，该机制的接线有错误。驱动程序端的计数器在这里有帮助：如果驱动程序报告发布了 1000 个事件但读取者报告看到 900 个，十个通知中有一个被丢弃。

同时针对同一设备运行所有三个读取者以单机制测试不会的方式压力生产者。只在所有三个都活动时才会显现的任何锁定排序错误在此工作负载下会显示出来。

### 应用程序兼容性

行为良好的驱动程序可以期望与遗留和现代用户态代码、单线程和多线程程序、选择一种机制的代码和选择另一种的代码一起工作。实现这一点的方式是支持所有三种机制并遵守每种机制的文档化契约。

遗留的基于 `select` 的代码应该通过我们的 `poll` 实现工作，因为 `select` 在内核中被转换为 `poll`。

现代的基于 `kqueue` 的代码应该通过我们的 `d_kqfilter` 工作，因为 `kqueue` 是 FreeBSD 上事件驱动用户态的原生机制。

使用 `SIGIO` 的单线程程序应该通过我们的 `FIOASYNC`/`FIOSETOWN` 处理工作。

混合机制的程序（例如，用 `kqueue` 监视一些描述符并使用 `SIGIO` 处理紧急事件）也应该工作，因为驱动程序的生产者路径在每个事件上通知所有机制。

这就是"应用程序兼容性"对驱动程序意味着什么。遵守契约，通知所有等待者，正确处理清理，任何时代的用户态代码都会工作。

### 第 7 节总结

我们现在有了一个完整的图景。三种异步机制，一个生产者，一个队列，一组锁，一个分离序列。组合设计不比任何单个机制多很多代码；艺术在于正确处理锁和排序，以及测试组合以便潜在错误在发布前被发现。

下一节将此组合设计应用为我们不断演进的 `evdemo` 驱动程序的重构。我们将审计最终代码，看看改变了什么，并将驱动程序发布为版本 v2.5-async。重构是抽象建议变成具体的、可工作的源代码的地方。

## 8. 异步支持的最终重构

前面几节一次一个机制地构建 `evdemo`，所以我们现在拥有的代码是一个工作但有些随意的积累。在本节中，我们将驱动程序重构为一个连贯的整体，具有一致的锁定规则、完整的分离路径和一组暴露的计数器，让我们观察其行为。结果是 `examples/part-07/ch35-async-io/lab06-v25-async/` 处的配套驱动程序，它作为本章练习的参考实现。

称此为"最终"重构略显理想化：真实驱动程序从不真正完成。但在功能构建完成后进行重构是一个有用的习惯，因为这是代码结构作为整体而不是一系列添加变得可见的时候。在增量开发期间隐藏的错误在代码布局为单个流时通常变得明显。

### 线程安全审查

我们的审查从锁定开始。softc 中的每个状态元素现在都由 `sc_mtx` 保护，以下除外：

`sc_sigio` 由 `SIGIO_LOCK` 全局内部保护，而不是我们的 softc 互斥锁。这是正确的，因为 `fsetown`、`fgetown`、`funsetown` 和 `pgsigio` API 自己获取全局锁。我们绝不能在调用这些 API 之前获取 `sc_mtx`，否则我们会与内核其余部分的信号代码反转锁序。

`sc_rsel` 由其自己的 selinfo 互斥锁内部保护。我们不直接接触内部列表；我们只调用 `selrecord` 和 `selwakeup`。这些函数自己获取内部锁。

其他一切（队列、计数器、异步标志、分离标志、条件变量等待队列）由 `sc_mtx` 保护。

审计是：每个读取或写入这些字段之一的代码路径在访问之前获取 `sc_mtx`，之后释放它。让我们遍历每个路径。

附加：`sc_mtx` 在任何访问之前初始化。其他一切被零化。附加时不可能有并发访问，因为驱动程序的句柄尚不存在。

分离：获取 `sc_mtx` 以设置 `sc_detaching = true`，发出 `cv_broadcast` 和 `KNOTE_LOCKED`，释放锁，调用 `selwakeup`，然后调用 `destroy_dev_drain`。在 `destroy_dev_drain` 返回后，我们的回调不能再开始。我们可以然后 `seldrain`、`knlist_destroy`、`funsetown`、`mtx_destroy`、`cv_destroy`，并释放 softc。

打开：不严格需要 `sc_mtx`，因为打开由内核序列化，但为内部状态更新获取它很便宜且澄清了代码。

关闭：在 `sc_mtx` 外部调用 `funsetown`。

读取：`sc_mtx` 在队列检查、`cv_wait_sig` 调用和 `dequeue` 周围持有。`uiomove` 在锁外完成，因为 `uiomove` 可能页面故障，我们不想跨故障持有驱动程序锁。

写入：在 `evdemo` 中不适用，但在接受写入的驱动程序中，模式是对称的。

Ioctl：`FIOASYNC` 获取 `sc_mtx`；`FIOSETOWN` 和 `FIOGETOWN` 不获取，因为它们使用有自己的锁定的 `fsetown/fgetown`。

Poll：`sc_mtx` 在检查和 `selrecord` 调用期间持有。

Kqfilter：`sc_mtx` 在调用我们的 `f_event` 回调之前由 kqueue 框架获取。我们的 `d_kqfilter` 为 `knlist_add` 调用获取它。

生产者（从 callout 调用的 `evdemo_post_event`）：`sc_mtx` 跨越入队、`cv_broadcast` 和 `KNOTE_LOCKED` 调用持有；在 `selwakeup` 和 `pgsigio` 之前释放。

每个 softc 字段的每次读写都在 `sc_mtx` 或适当的外部锁下被考虑。这是你想在每个异步驱动程序上执行的审计，因为它是发布前发现潜在并发错误的审计。

### 完整附加序列

将附加路径放在一起，以调用必须发生的顺序：

```c
static int
evdemo_modevent(module_t mod, int event, void *arg)
{
    struct evdemo_softc *sc;
    int error = 0;

    switch (event) {
    case MOD_LOAD:
        sc = malloc(sizeof(*sc), M_EVDEMO, M_WAITOK | M_ZERO);
        mtx_init(&sc->sc_mtx, "evdemo", NULL, MTX_DEF);
        cv_init(&sc->sc_cv, "evdemo");
        knlist_init_mtx(&sc->sc_rsel.si_note, &sc->sc_mtx);
        callout_init_mtx(&sc->sc_callout, &sc->sc_mtx, 0);

        sc->sc_dev = make_dev(&evdemo_cdevsw, 0, UID_ROOT, GID_WHEEL,
            0600, "evdemo");
        sc->sc_dev->si_drv1 = sc;
        evdemo_sc_global = sc;
        break;
    /* ... */
    }
    return (error);
}
```

顺序是刻意的：首先初始化所有同步原语，然后注册回调（在 `make_dev` 调用后任何时候可以开始到达），然后通过 `si_drv1` 和全局指针发布 softc。

一个微妙之处是 `M_WAITOK`。我们想在附加时进行阻塞分配，因为我们在模块加载上下文中，总是被允许休眠。`M_ZERO` 是必需的，因为未初始化的 selinfo、knlist 或条件变量会崩溃内核。有了这些标志，分配要么以零化结构成功，要么模块加载干净地失败。

### 完整分离序列

分离路径更精细，因为我们必须与进行中的调用者和活动等待者协调：

```c
case MOD_UNLOAD:
    sc = evdemo_sc_global;
    if (sc == NULL)
        break;

    mtx_lock(&sc->sc_mtx);
    sc->sc_detaching = true;
    cv_broadcast(&sc->sc_cv);
    KNOTE_LOCKED(&sc->sc_rsel.si_note, 0);
    mtx_unlock(&sc->sc_mtx);
    selwakeup(&sc->sc_rsel);

    callout_drain(&sc->sc_callout);
    destroy_dev_drain(sc->sc_dev);

    seldrain(&sc->sc_rsel);
    knlist_destroy(&sc->sc_rsel.si_note);
    funsetown(&sc->sc_sigio);

    cv_destroy(&sc->sc_cv);
    mtx_destroy(&sc->sc_mtx);

    free(sc, M_EVDEMO);
    evdemo_sc_global = NULL;
    break;
```

这里的序列值得研究，因为它包含几个顺序敏感的步骤。

在锁下设置 `sc_detaching` 并广播是让阻塞读取者醒来并看到标志的东西。没有这个，卡在 `cv_wait_sig` 中的读取者会永远休眠，因为我们即将销毁条件变量。

`KNOTE_LOCKED` 调用（以及 `f_event` 中的 `EV_EOF` 路径）让任何 kqueue 等待者看到文件结束。

锁外的 `selwakeup` 唤醒 poll 等待者。它们返回用户态并看到它们的文件描述符变为无效。

`callout_drain` 停止模拟事件源。任何即将触发的 callout 首先完成；不会启动新的。

`destroy_dev_drain` 等待任何进行中的回调返回。此之后，`d_open`、`d_close`、`d_read`、`d_write`、`d_ioctl`、`d_poll` 和 `d_kqfilter` 都保证已经返回。

`seldrain` 清理任何残留的 selinfo 状态。

`knlist_destroy` 验证 knote 列表为空（它应该是，因为当文件描述符关闭时每个 knote 的 `f_detach` 被调用）并释放内部锁状态。

`funsetown` 清除信号所有者。

最后我们销毁条件变量和互斥锁，释放 softc，并清除全局指针。

这种仔细的排序是一个干净卸载的驱动程序和第二次加载时 panic 的驱动程序之间的区别。任何严肃驱动程序的测试方案包括"循环加载和卸载一百次"的练习，因为分离路径中的竞争窗口通常太窄，一次尝试无法命中。

### 暴露事件指标

完成的驱动程序通过 `sysctl` 暴露其事件指标：

```c
SYSCTL_NODE(_dev, OID_AUTO, evdemo, CTLFLAG_RW, 0, "evdemo driver");

static SYSCTL_NODE(_dev_evdemo, OID_AUTO, stats,
    CTLFLAG_RW, 0, "Runtime statistics");

SYSCTL_UINT(_dev_evdemo_stats, OID_AUTO, posted, CTLFLAG_RD,
    &evdemo_posted, 0, "Events posted since attach");
SYSCTL_UINT(_dev_evdemo_stats, OID_AUTO, consumed, CTLFLAG_RD,
    &evdemo_consumed, 0, "Events consumed by read(2)");
SYSCTL_UINT(_dev_evdemo_stats, OID_AUTO, dropped, CTLFLAG_RD,
    &evdemo_dropped, 0, "Events dropped due to overflow");
SYSCTL_UINT(_dev_evdemo_stats, OID_AUTO, qlen, CTLFLAG_RD,
    &evdemo_qlen, 0, "Current queue length");
SYSCTL_UINT(_dev_evdemo_stats, OID_AUTO, selwakeups, CTLFLAG_RD,
    &evdemo_selwakeups, 0, "selwakeup calls");
SYSCTL_UINT(_dev_evdemo_stats, OID_AUTO, knotes_delivered, CTLFLAG_RD,
    &evdemo_knotes_delivered, 0, "knote deliveries");
SYSCTL_UINT(_dev_evdemo_stats, OID_AUTO, sigio_sent, CTLFLAG_RD,
    &evdemo_sigio_sent, 0, "SIGIO signals sent");
```

每个计数器在生产者中 softc 锁下递增。计数器对于正确操作不是必需的，但它们对于驱动程序可观察是必需的。报告消费零事件而队列已满的驱动程序告诉我们读取者没有在排空。报告 selwakeup 多于 knote 传递的驱动程序告诉我们等待者混合的一些信息。报告许多 `sigio_sent` 但用户态没有可见效果的驱动程序告诉我们检查所有者的信号处理程序。

可观察性添加几乎不花什么成本，但在生产调试中多次回报。将其添加到最终重构是使驱动程序准备好实际使用的一部分。

### 驱动程序版本化

我们在代码和配套示例目录中标记此版本为 `v2.5-async`。约定是简单的 `MODULE_VERSION` 声明：

```c
MODULE_VERSION(evdemo, 25);
```

数字是版本的整数形式：25 代表 2.5。FreeBSD 的模块加载基础架构使用此数字在模块之间强制依赖约束。依赖特定版本 `evdemo` 的模块可以用 `MODULE_DEPEND(9)` 声明。对于我们的独立驱动程序，版本主要是信息性的，但每次功能发布时递增它是一个好习惯。

### 第 8 节总结

最终的 `evdemo` 驱动程序支持阻塞和非阻塞 `read()`、带 `selrecord`/`selwakeup` 的 `poll()`、带 `EVFILT_READ` 的 `kqueue()` 以及通过 `FIOASYNC`/`FIOSETOWN` 的 `SIGIO`。它有一个有丢弃最旧溢出策略的有界内部事件队列。它通过 `sysctl` 暴露计数器以实现可观察性。其附加和分离序列已经过线程安全审计。它是一个小驱动程序，大约四百行 C，但展示了本章教授的每个模式。

更重要的是，它是一个模板。你在这里看到的模式泛化到任何需要异步 I/O 的驱动程序。USB 输入设备用真实的 URB 回调替换模拟 callout。GPIO 驱动程序用真实中断处理程序替换 callout。网络伪设备用 mbuf 链替换事件队列。异步通知框架（poll、kqueue、SIGIO）在所有这些中保持相同。一旦你知道了模式，为新驱动程序添加异步支持就是接线问题，而不是设计问题。

我们现在已经覆盖了本章的核心材料。本章的下一部分是动手实践的：一系列实验，带你逐步构建 `evdemo`，一次添加一个机制，并用真实用户态程序验证行为。如果你一直在阅读而没有运行代码，现在是时候在 FreeBSD 虚拟机上打开终端并开始输入了。


## 动手实验

本节的实验增量构建 `evdemo`。每个实验对应本书配套源码中 `examples/part-07/ch35-async-io/` 下的一个文件夹。你可以从头输入每个实验（较慢但建立更强的直觉），或者从提供的源码开始并专注于实验教授的代码。两种方法都可以；选择最适合你学习风格的方式。

在开始之前有几个一般性说明。

每个实验使用相同的 `Makefile` 模式。一个 `KMOD` 行命名模块，一个 `SRCS` 行列出源码，`bsd.kmod.mk` 完成其余。在实验目录中运行 `make` 生成 `evdemo.ko`，`sudo kldload ./evdemo.ko` 加载它。`make test` 构建同一目录中的用户空间测试程序。

每个实验在 `/dev/evdemo` 暴露一个设备节点。如果你在构建新版本之前忘记卸载驱动程序的先前版本，加载将失败并显示"device already exists。"运行 `sudo kldunload evdemo` 清理，然后重新加载。

每个实验包含一个小测试程序，测试实验教授的机制。在驱动程序旁边运行测试程序端到端验证机制工作。如果测试程序挂起或报告错误，驱动程序中的某些东西坏了，实验的故障排除说明通常会帮助你找到它。

### 实验 1：同步基线

第一个实验建立后续实验构建的同步基线。我们这里的目标是一个支持在内部事件队列上阻塞 `read()` 的最小 `evdemo` 驱动程序。还没有异步机制。本实验教授队列数据结构和一切将叠加在其上的条件变量模式。

**文件：**

- `evdemo.c` - 驱动程序源码
- `evdemo.h` - 带事件记录定义的共享头文件
- `evdemo_test.c` - 用户空间读取器
- `Makefile` - 模块构建加测试目标

**步骤：**

1. 阅读实验目录的内容。熟悉 `evdemo_softc` 的结构，特别是队列字段和条件变量。

2. 构建驱动程序：`make`。

3. 构建测试程序：`make test`。

4. 加载驱动程序：`sudo kldload ./evdemo.ko`。

5. 在一个终端中，运行测试程序：`sudo ./evdemo_test`。程序打开 `/dev/evdemo` 并调用 `read()`，它会阻塞因为没有事件被发布。

6. 在第二个终端中，触发事件：`sudo sysctl dev.evdemo.trigger=1`。sysctl 在驱动程序中连接到用合成事件调用 `evdemo_post_event`。测试程序应该解除阻塞，打印事件，并再次调用 `read()`。

7. 触发更多事件。观察测试程序在每个到达时打印。

8. 卸载驱动程序：`sudo kldunload evdemo`。

**观察什么：** 测试程序中的 `read()` 调用在队列为空时阻塞，一次返回恰好一个事件。测试程序在等待时不旋转 CPU；你可以通过在第三个终端中观察 `top -H` 并注意到测试进程处于 `S`（睡眠）状态来确认，等待通道名为类似 `evdemo` 或通用的 `cv`。

**检查常见错误：** 如果测试程序立即返回零字节，队列可能报告自己为空，但 `read()` 路径没有在条件变量上等待。检查 `evdemo_read` 中的 while 循环实际调用了 `cv_wait_sig`。如果测试程序挂起且即使触发事件也不解除阻塞，检查生产者实际在互斥锁内调用了 `cv_broadcast`。

**要点：** 带条件变量的阻塞 `read()` 是同步基线。它工作，但对于需要监视多个描述符或在不始终有线程阻塞在 `read()` 中的情况下对事件做出反应的程序来说不够。接下来的实验添加异步支持。

### 实验 2：添加 poll() 支持

第二个实验向驱动程序添加 `d_poll`，以便用户态程序可以等待多个描述符或将 `evdemo` 集成到事件循环中。本实验教授 `selrecord`/`selwakeup` 模式。

**文件：**

- `evdemo.c` - 驱动程序源码（从实验 1 扩展）
- `evdemo.h` - 共享头文件
- `evdemo_test_poll.c` - 基于 poll 的测试程序
- `Makefile` - 模块构建加测试目标

**驱动程序从实验 1 的变化：**

向 softc 添加一个 `struct selinfo sc_rsel`。

在 attach 期间用 `knlist_init_mtx(&sc->sc_rsel.si_note, &sc->sc_mtx)` 初始化它。即使我们还没有使用 kqueue，预初始化 `si_note` knlist 很便宜，并使 selinfo 稍后兼容 kqueue 支持。

添加 `d_poll` 回调：

```c
static int
evdemo_poll(struct cdev *dev, int events, struct thread *td)
{
    struct evdemo_softc *sc = dev->si_drv1;
    int revents = 0;

    mtx_lock(&sc->sc_mtx);
    if (events & (POLLIN | POLLRDNORM)) {
        if (!evdemo_queue_empty(sc))
            revents |= events & (POLLIN | POLLRDNORM);
        else
            selrecord(td, &sc->sc_rsel);
    }
    if (events & (POLLOUT | POLLWRNORM))
        revents |= events & (POLLOUT | POLLWRNORM);
    mtx_unlock(&sc->sc_mtx);

    return (revents);
}
```

将其连接到 `cdevsw`：

```c
.d_poll = evdemo_poll,
```

在互斥锁释放后从 `evdemo_post_event` 调用 `selwakeup(&sc->sc_rsel)`。

在分离期间调用 `seldrain(&sc->sc_rsel)` 和 `knlist_destroy(&sc->sc_rsel.si_note)`。

**步骤：**

1. 复制实验 1 源码作为起点。
2. 应用上述更改。
3. 构建：`make`。
4. 构建测试程序：`make test`。
5. 加载：`sudo kldload ./evdemo.ko`。
6. 运行基于 poll 的测试：`sudo ./evdemo_test_poll`。它应该用 5 秒超时调用 `poll()` 并打印结果。没有事件发布时，`poll()` 在超时后返回零。
7. 在测试运行时触发事件：`sudo sysctl dev.evdemo.trigger=1`。`poll()` 调用应该立即返回并设置 `POLLIN`，程序应该读取事件。
8. 尝试带多个描述符的 `poll()`：测试程序的扩展模式打开 `/dev/evdemo` 两次并轮询两个描述符。触发事件并观察哪个触发。

**观察什么：** `poll()` 阻塞直到事件到达，而不是直到超时过去，当事件确实被触发时。程序不在 CPU 上旋转；它在内核中真正睡眠。你可以用 `top -H` 验证并查看 WCHAN，应该显示 `select` 或类似的等待通道。

**检查常见错误：** 如果 poll 立即返回 `POLLIN` 即使队列为空，检查你的队列空性检查是否正确。如果在触发事件后 poll 仍然超时返回，生产者没有调用 `selwakeup`，或者它在更新队列之前调用 `selwakeup`。如果在触发事件时内核 panic，selinfo 未正确初始化；检查在 softc 分配中使用了 `M_ZERO` 并且调用了 `knlist_init_mtx`。

**要点：** `poll()` 支持是一百行额外代码，给每个基于 poll 的用户态程序集成 `evdemo` 的能力。关键是锁定规则：softc 互斥锁将 `d_poll` 中的检查和注册与生产者中的队列更新序列化。没有锁，我们在第 3 节分析的竞争会导致偶尔的唤醒丢失。

### 实验 3：添加 kqueue 支持

第三个实验添加 `d_kqfilter`，以便使用 `kqueue(2)` 的程序可以集成 `evdemo`。本实验教授过滤器操作结构和 `KNOTE_LOCKED` 传递模式。

**文件：**

- `evdemo.c` - 驱动程序源码（从实验 2 扩展）
- `evdemo.h` - 共享头文件
- `evdemo_test_kqueue.c` - 基于 kqueue 的测试程序
- `Makefile`

**驱动程序从实验 2 的变化：**

添加过滤器操作：

```c
static int evdemo_kqread(struct knote *, long);
static void evdemo_kqdetach(struct knote *);

static const struct filterops evdemo_read_filterops = {
    .f_isfd = 1,
    .f_attach = NULL,
    .f_detach = evdemo_kqdetach,
    .f_event = evdemo_kqread,
};

static int
evdemo_kqread(struct knote *kn, long hint)
{
    struct evdemo_softc *sc = kn->kn_hook;

    mtx_assert(&sc->sc_mtx, MA_OWNED);
    kn->kn_data = sc->sc_nevents;
    if (sc->sc_detaching) {
        kn->kn_flags |= EV_EOF;
        return (1);
    }
    return (sc->sc_nevents > 0);
}

static void
evdemo_kqdetach(struct knote *kn)
{
    struct evdemo_softc *sc = kn->kn_hook;

    knlist_remove(&sc->sc_rsel.si_note, kn, 0);
}
```

添加 `d_kqfilter` 回调：

```c
static int
evdemo_kqfilter(struct cdev *dev, struct knote *kn)
{
    struct evdemo_softc *sc = dev->si_drv1;

    switch (kn->kn_filter) {
    case EVFILT_READ:
        kn->kn_fop = &evdemo_read_filterops;
        kn->kn_hook = sc;
        knlist_add(&sc->sc_rsel.si_note, kn, 0);
        return (0);
    default:
        return (EINVAL);
    }
}
```

将其连接到 `cdevsw`：

```c
.d_kqfilter = evdemo_kqfilter,
```

在生产者的临界区内添加 `KNOTE_LOCKED(&sc->sc_rsel.si_note, 0)` 调用。在 `cv_broadcast` 和 `mtx_unlock` 之间。

在分离的顶部添加 `knlist_clear(&sc->sc_rsel.si_note, 0)`，在 `seldrain` 之前，以移除任何未调用 `f_detach` 的仍附加的 knote（例如，因为 kqueue 在设备的 knote 仍附加时关闭了）。

**步骤：**

1. 复制实验 2 源码。
2. 应用上述更改。
3. 构建并加载。
4. 运行基于 kqueue 的测试：`sudo ./evdemo_test_kqueue`。程序打开 `/dev/evdemo`，创建一个 kqueue，为设备注册 `EVFILT_READ`，并以阻塞模式调用 `kevent()`。
5. 触发事件并观察 kqueue 读取器打印它们。

**观察什么：** kqueue 读取器通过 `kevent()` API 而不是通过 `poll()` 报告事件。它在 `ev.data` 中获得 `kn_data` 值，告诉它有多少事件排队。

**检查常见错误：** 如果 kqueue 读取器立即返回错误，`d_kqfilter` 可能因为错误的 case 返回 `EINVAL`。检查 switch 语句。如果 kqueue 读取器即使触发了事件也挂起，`KNOTE_LOCKED` 可能没有被调用，或者在锁外被调用。如果在模块卸载时内核 panic 抱怨非空 knote 列表，`knlist_clear` 缺失。

**要点：** `kqueue` 支持是又一百行代码。结构类似于 `poll`：事件回调中的检查，生产者中的传递，以及分离步骤。框架处理繁重的工作。

### 实验 4：添加 SIGIO 支持

第四个实验添加异步信号传递。本实验教授 `FIOASYNC`、`fsetown` 和 `pgsigio`。

**文件：**

- `evdemo.c` - 驱动程序源码（从实验 3 扩展）
- `evdemo.h`
- `evdemo_test_sigio.c` - 基于 SIGIO 的测试程序
- `Makefile`

**驱动程序从实验 3 的变化：**

向 softc 添加异步支持：

```c
bool              sc_async;
struct sigio     *sc_sigio;
```

向 ioctl 处理程序添加三个 ioctl：

```c
case FIOASYNC:
    mtx_lock(&sc->sc_mtx);
    sc->sc_async = (*(int *)data != 0);
    mtx_unlock(&sc->sc_mtx);
    break;

case FIOSETOWN:
    error = fsetown(*(int *)data, &sc->sc_sigio);
    break;

case FIOGETOWN:
    *(int *)data = fgetown(&sc->sc_sigio);
    break;
```

向生产者添加 `pgsigio` 传递，在锁外：

```c
if (async)
    pgsigio(&sc->sc_sigio, SIGIO, 0);
```

向关闭路径和分离路径添加 `funsetown(&sc->sc_sigio)`。

**步骤：**

1. 复制实验 3。
2. 应用上述更改。
3. 构建并加载。
4. 运行基于 SIGIO 的测试：`sudo ./evdemo_test_sigio`。程序安装 SIGIO 处理程序，用其 PID 调用 `FIOSETOWN`，调用 `FIOASYNC` 启用，然后在循环中暂停，每当处理程序设置标志时用非阻塞读取排空驱动程序。
5. 触发事件并观察程序打印每一个。

**观察什么：** 每个事件通过信号到达，而不是通过阻塞 `read()` 或 `poll()`。信号处理程序本身不从设备读取；它设置一个标志，主循环读取。这是 SIGIO 处理程序的标准模式。

**检查常见错误：** 如果测试程序没有看到任何信号，`FIOASYNC` 可能没有启用 `sc_async`，或者生产者没有检查 `sc_async`。还要检查在生产者触发之前调用了 `fsetown`。

如果测试程序因关于 SIGIO 的错误而中止，信号处理程序可能未安装，或者信号被屏蔽了。如果你想跨信号传递自动重启系统调用，使用 `sigprocmask` 或带 `SA_RESTART` 的 `sigaction`。

**要点：** SIGIO 从驱动程序的角度比 poll 或 kqueue 更简单：一个 ioctl 处理程序，一次 `fsetown` 调用，一次 `pgsigio` 调用。用户态方面更复杂，因为信号本身就有棘手的语义。

### 实验 5：事件队列

第五个实验专注于内部事件队列本身。我们重新组织驱动程序，使队列成为所有异步机制的唯一事实来源，并添加基于 sysctl 的内省，以便我们可以在运行时观察队列行为。

**文件：**

- `evdemo.c` - 带有精炼队列实现的驱动程序源码
- `evdemo.h` - 带事件记录的共享头文件
- `evdemo_watch.c` - 打印队列指标的诊断工具
- `Makefile`

**什么改变了：**

队列函数变成独立的、文档良好的。每个操作获取 softc 互斥锁，用 `mtx_assert` 断言它，并使用一致的命名约定。

`dev.evdemo.stats` 下的 `sysctl` 子树暴露队列长度、总发布事件数、总消费事件数和因溢出而丢弃的总事件数。

一个 `trigger` sysctl 允许用户态发布给定类型的合成事件，这简化了测试，无需编写和加载自定义测试程序。

一个 `burst` sysctl 一次发布一批事件，这测试队列的溢出行为。

**步骤：**

1. 复制实验 4。
2. 应用队列精炼：将 enqueue/dequeue 操作提取为清晰命名的辅助函数，添加计数器，添加 sysctl 条目。
3. 构建并加载。
4. 在循环中运行 `sysctl dev.evdemo.stats` 观察队列状态：`while :; do sysctl dev.evdemo.stats; sleep 1; done`。
5. 触发突发：`sudo sysctl dev.evdemo.burst=100`。观察队列填满，然后在队列满时丢弃溢出事件。
6. 在触发突发时运行任何读取器测试程序（poll、kqueue 或 SIGIO）。观察读取器排空队列。

**观察什么：** sysctl 中报告的队列长度跟踪已发布但尚未消费的事件数。当队列满时发布事件时丢弃计数器增长。当读取者慢于生产者时，已发布和已消费的计数器分歧，当读取者赶上时趋同。

**检查常见错误：** 如果丢弃计数器增长但溢出策略未触发，队列的满性检查有误。如果已发布计数器增长但已消费计数器不增长，生产者正在入队但读取者没有出队（这可能是正确的，如果没有读取者在运行，但通常意味着读取路径中的错误）。

**要点：** 事件队列是三种异步机制围绕的核心。有了 sysctl 可观察性，我们可以直接在各种负载下观察队列的行为并验证它在做我们期望的事情。

### 实验 6：组合的 v2.5-async 驱动程序

最终实验是整合的 `evdemo` 驱动程序，带有所有三种异步机制、经审计的锁定规则、暴露的指标和干净的分离路径。这是未来驱动程序可以建模的参考实现。

**文件：**

- `evdemo.c` - 完整参考驱动程序
- `evdemo.h` - 共享头文件
- `evdemo_test_poll.c` - 基于 poll 的测试
- `evdemo_test_kqueue.c` - 基于 kqueue 的测试
- `evdemo_test_sigio.c` - 基于 SIGIO 的测试
- `evdemo_test_combined.c` - 同时运行所有三个的测试
- `Makefile`

**本实验演示什么：**

组合测试程序 fork 三个子进程。一个使用 `poll`，一个使用 `kqueue`，一个使用 `SIGIO`。每个子进程打开自己的文件描述符到 `/dev/evdemo` 并监视事件。父进程以已知速率触发事件并在固定持续时间后报告。

**步骤：**

1. 构建并加载。
2. 运行组合测试：`sudo ./evdemo_test_combined`。它 fork 三个子进程，以每秒几百个的速率触发 1000 个事件，并在结束时打印摘要。
3. 观察所有三个读取器看到所有事件。

**观察什么：** sysctl 中的已发布计数器等于所有三个读取器看到的事件总和。没有机制丢弃事件。读取器在彼此几毫秒内完成，演示驱动程序对所有三个同时响应。

**检查常见错误：** 如果一个读取器始终落后，检查其机制的通知在每个事件上发出。如果三个读取器产生不同的事件计数，一种机制正在丢弃通知，这表明生产者中有唤醒丢失。

**要点：** 正确实现所有三种异步机制的驱动程序服务任何用户态调用者。这是你为事件驱动设备构建生产驱动程序时应该瞄准的目标。一旦你知道了模式，工作就是机械的。

### 实验 7：卸载压力测试

最终实验是分离路径的压力测试，因为分离是异步驱动程序中微妙错误倾向于隐藏的地方。

**文件：**

- 来自实验 6 的 `evdemo.c`
- `evdemo_stress.sh` - 在循环中加载、测试和卸载驱动程序的 shell 脚本

**步骤：**

1. 加载驱动程序。
2. 在一个终端中，持续在循环中运行组合测试。
3. 在另一个终端中，运行压力脚本：`sudo ./evdemo_stress.sh 100`。这连续一百次加载、测试、卸载和重新加载驱动程序，在并发读取者下测试附加和分离序列。
4. 观察没有 panic 发生，所有读取器在每次卸载-重新加载周期中干净终止，sysctl 计数器在每次附加时重置为零。

**观察什么：** 具有正确分离逻辑的驱动程序可以承受一百或一千次加载/卸载周期而不 panic、泄漏内存或挂起。具有不正确分离的驱动程序通常在十或二十个周期内 panic。

**检查常见错误：** 最常见的分离错误是在释放 softc 之前忘记排空进行中的调用者。`destroy_dev_drain` 是此操作的标准工具；没有它，进行中的 `read()` 或 `ioctl()` 可能触及已释放的 softc。

第二常见的错误是附加和分离初始化顺序之间的不匹配。`knlist_init_mtx` 必须在设备发布之前发生，因为 `kqfilter` 调用可以在之后立即到达。对称地，`knlist_destroy` 必须在设备排空之后发生。

**要点：** 压力测试卸载路径是异步驱动程序最有效的单一测试。如果你的驱动程序在并发负载下经受住 100 次加载/卸载周期，它可能是可靠的。


## 挑战练习

这些练习是可选的。它们基于实验来磨炼你在特定领域的技能。慢慢来；不用急。

### 挑战 1：双机制对决

修改 `evdemo_test_combined` 以测量每种机制的每事件延迟：生产者的 `evdemo_post_event` 调用和用户态读取器的 `read()` 返回之间的时间。使用 `CLOCK_MONOTONIC` 时钟并在事件记录本身上记录时间。

报告一个小表格，显示 `poll`、`kqueue` 和 `SIGIO` 各自的平均、中位数和第 99 百分位延迟。在无竞争（每个机制一个读取器）和有竞争（每个机制三个读取器）的情况下尝试。哪种机制在无竞争下延迟最低？在有竞争下呢？

预期答案是 `kqueue` 最低，`poll` 第二，`SIGIO` 可变（因为信号传递延迟取决于读取器的当前执行状态）。但细节取决于你的硬件，练习是测量而不是预测。

### 挑战 2：多读取器压力

打开二十个文件描述符到 `/dev/evdemo` 并使用 `kqueue` 从单个线程同时轮询所有这些。触发 10000 个事件并验证每个事件恰好传递到所有二十个描述符。

这测试驱动程序的 knote 列表正确处理多个 knote，以及 `KNOTE_LOCKED` 在每个事件上完全遍历列表。

### 挑战 3：观察唤醒丢失竞争

第三个挑战要求你故意破坏驱动程序以便你可以观察唤醒丢失。修改 `evdemo_post_event` 以在 softc 互斥锁外而不是内部更新队列并调用通知：

```c
/* 已破坏：与 d_poll 竞争 */
mtx_lock(&sc->sc_mtx);
evdemo_enqueue(sc, ev);
mtx_unlock(&sc->sc_mtx);
selwakeup(&sc->sc_rsel);
/* ... */
```

这解除了生产者的入队与消费者的检查加注册的绑定。在足够高的事件率和忙碌的消费者下，你应该偶尔看到 `poll()` 调用在事件已经发布后经过长延迟才返回。

尝试重现竞争。计时 `poll()` 调用。报告竞争作为事件率函数触发的频率。然后恢复正确的锁定并验证竞争消失。

本练习的重点不是编写有错误的代码。而是亲眼看到我们在第 3 节描述的锁定规则不是理论上的讲究而是真实的正确性属性。经历一次竞争胜过阅读一百次描述。

### 挑战 4：事件合并

向 `evdemo` 添加事件合并功能。当生产者发布的事件类型与队列中最新事件类型匹配时，将它们合并为带有递增计数器的单个事件，而不是追加新条目。这类似于一些驱动程序合并中断事件的方式。

用一百个相同类型的突发测试它。队列长度应该保持为一。现在用一百个交替类型的事件测试：队列应该填充交替条目。

挑战既关于设计用户态契约，也关于实现功能。读取者在合并发生时看到什么？它如何知道事件被合并了？当队列有一个条目但它代表许多事件时，kqueue `kn_data` 字段报告什么？

没有单一正确答案。在源码中记录你的设计选择并准备好为它们辩护。

### 挑战 5：POLLHUP 和 POLLERR

向驱动程序添加 `POLLHUP` 和 `POLLERR` 的优雅处理。当设备在用户态程序仍然打开它时被分离，该程序应该在其下一个 `poll()` 调用中看到 `POLLHUP`（如果仍有排队事件则连同 `POLLIN`）。当驱动程序有阻止未来操作的内部错误时，它应该设置错误标志并在后续 `poll()` 调用中报告 `POLLERR`。

通过安排驱动程序在读取者轮询时被分离来测试它。读取器应该带着 `POLLHUP` 醒来并干净退出。

这教授完整的 `poll()` 契约和 `revents` 位掩码的微妙之处。它也与分离逻辑重叠，那是设置 HUP 条件的正确地方。

### 挑战 6：evdev 风格兼容性

向 `evdemo` 添加兼容层，实现 evdev ioctl 集，以便你的驱动程序对现有 evdev 感知用户态程序可见。关键 ioctl 是 `EVIOCGVERSION`、`EVIOCGID`、`EVIOCGNAME` 和其他几个记录在 `/usr/src/sys/dev/evdev/input.h` 中的 ioctl。

这是一个更大的练习，对于理解真实输入设备如何将自己暴露给用户态真正有用。它需要仔细阅读 evdev 源码并选择合理的子集来实现。

### 挑战 7：端到端跟踪 kqueue 注册

使用 `dtrace(1)` 或 `ktrace(1)`，跟踪单个 `kevent(2)` 调用，该调用在 `evdemo` 文件描述符上注册 `EVFILT_READ`。你的跟踪应该覆盖：

- 进入 `kevent` 系统调用。
- 在 kqueue 框架中调用 `kqueue_register`。
- 在 cdev fileops 上调用 `fo_kqfilter`。
- 进入 `evdemo_kqfilter`（我们驱动程序的 `d_kqfilter`）。
- `knlist_add` 调用。
- 通过框架返回用户态。

在每个点捕获栈跟踪。然后在驱动程序中触发生产者事件并跟踪传递路径：

- 生产者中的 `KNOTE_LOCKED` 调用。
- 框架中进入 `knote`。
- 调用 `evdemo_kqread`（我们的 `f_event`）。
- 将通知排队到 kqueue 上。

最后，用户态用另一个 `kevent` 调用收获事件。也跟踪该路径：

- 第二次进入 `kevent` 系统调用。
- 调用 `kqueue_scan`。
- 遍历排队的 knote。
- 传递到用户态。

提交你的跟踪，并附上几句话说明每部分在做什么。本练习迫使人直接面对 kqueue 框架源码，是从"理解回调"到"理解框架"的最确定方式。完成此挑战的读者将有信心阅读树中任何使用 kqueue 的子系统。

提示：`dtrace -n 'fbt::kqueue_register:entry { stack(); }'` 是一个合理的起点。从那里向外扩展，随着你在源码中识别出它们，在 `knote`、`knlist_add`、`knlist_remove` 和你的驱动程序入口点上添加探针。

### 挑战 8：观察 knlist 锁定规则

编写一个小测试程序，从两个不同进程两次打开 `evdemo` 设备，在每个上注册 `EVFILT_READ`，然后触发一个生产者事件。使用 `dtrace` 测量单个传递期间 knlist 锁被获取和释放多少次。根据本章关于 `KNOTE_LOCKED` 和 knlist 遍历的教导提前预测次数；然后根据跟踪验证。

接下来，修改 `evdemo` 以便生产者使用 `KNOTE_UNLOCKED` 而不是 `KNOTE_LOCKED`（同时调整周围锁定使调用安全）。重复测量。获取次数应该改变，且改变应该匹配框架在两个代码路径中不同做的事。

提示：`dtrace -n 'mutex_enter:entry /arg0 == (uintptr_t)&sc->sc_mtx/ { @ = count(); }'` 如果你知道特定互斥锁的地址，将计算该互斥锁上的获取次数。你可以通过 `kldstat -v` 加一些符号检查找到地址。

## 常见错误故障排除

异步 I/O 错误倾向于归入可识别的类别。本节收集最常见的失败模式、其症状和通常原因，以便当你遇到一个时可以快速诊断。

### 症状：poll() 永不返回

一个 poll() 调用永远阻塞，即使事件被触发。

**原因 1：** 生产者没有调用 `selwakeup`。向 `evdemo_post_event` 添加一个计数器并验证它在事件触发时实际递增。

**原因 2：** 生产者在队列状态更新之前调用 `selwakeup`。验证 `selwakeup` 在 `mtx_unlock` 之后调用，而不是之前。

**原因 3：** 消费者的 `d_poll` 没有正确调用 `selrecord`。检查调用在 softc 互斥锁下进行，并且传递的 selinfo 与生产者唤醒的是同一个。

**原因 4：** 消费者检查了错误的状态。验证 `d_poll` 中的队列空性检查查看的是与生产者更新的相同字段。

### 症状：kqueue 事件触发但 read() 返回无数据

一个 kqueue 读取器收到 `EVFILT_READ` 事件但后续 `read()` 返回 `EAGAIN` 或零字节。

**原因 1：** 队列在 kqueue 事件传递和读取之间被另一个读取器排空。这是多读取器竞争的良性症状，不是错误。读取器应该在 `EAGAIN` 上循环并等待下一个事件。

**原因 2：** `f_event` 回调在队列实际为空时返回 true。检查 `evdemo_kqread` 逻辑。

**原因 3：** 事件在 `KNOTE_LOCKED` 被调用后被合并或重新归档。检查在 `KNOTE_LOCKED` 调用后可能移除事件的任何队列操作。

### 症状：SIGIO 被传递但处理程序未被调用

驱动程序调用 `pgsigio`，但用户态程序从不看到信号。

**原因 1：** 程序没有为 `SIGIO` 安装处理程序。默认情况下，`SIGIO` 被忽略，不传递。

**原因 2：** 程序用 `pthread_sigmask` 或 `sigprocmask` 阻塞了 `SIGIO`。检查信号掩码。

**原因 3：** 程序用错误的 PID 调用 `FIOSETOWN`，所以信号去往另一个进程。验证参数是当前进程的 PID。

**原因 4：** 驱动程序只在 `sc_async` 为 true 时调用 `pgsigio`，但用户态从未启用 `FIOASYNC`。检查 ioctl 处理程序正确更新 `sc_async`。

### 症状：模块卸载时内核 panic

内核在 `kldunload evdemo` 期间 panic。

**原因 1：** `knlist_destroy` 在仍有 knote 附加的 knlist 上被调用。在 `knlist_destroy` 之前添加 `knlist_clear` 以强制移除任何剩余 knote。

**原因 2：** `seldrain` 在进行中的调用者返回之前被调用。先调用 `destroy_dev_drain`，然后 `seldrain`。

**原因 3：** 条件变量在有线程仍在等待它时被销毁。在 `cv_destroy` 之前设置 `sc_detaching = true` 并 `cv_broadcast`。

**原因 4：** softc 在另一个线程仍持有指向它的指针时被释放。确保全局 softc 指针在 `destroy_dev_drain` 返回后清除，而不是之前。

### 症状：重复加载/卸载时内存泄漏

经过多次加载/卸载周期后，`vmstat -m` 显示驱动程序的 `MALLOC_DEFINE` 类型的分配在增长。

**原因 1：** softc 在分离时未被释放。检查调用了 `free(sc, M_EVDEMO)`。

**原因 2：** `funsetown` 未被调用。每次 `fsetown` 调用分配一个必须被释放的 `struct sigio`。

**原因 3：** 某些内部分配（例如，每读取器结构）在关闭时未被释放。审计每个分配路径并确认每个 `malloc` 有匹配的 `free`。

### 症状：负载下 poll() 唤醒缓慢

基于 poll 的读取器通常快速唤醒但偶尔需要大量时间才能看到事件。

**原因：** 繁忙系统上的调度器唤醒传递延迟在毫秒范围。这不是驱动程序错误；这是内核调度器的一般属性。

如果此延迟对你的用例不可接受，考虑使用带 `EV_CLEAR` 的 `kqueue`，它有稍低的唤醒开销，或者使用专用内核线程作为消费者而不是用户态进程。

### 症状：负载下事件被丢弃

驱动程序的 `dropped` sysctl 计数器在事件突发期间增长。

**原因：** 队列小于突发大小，溢出策略（丢弃最旧）正在生效。

这在默认策略下按设计工作。如果你的应用程序不能容忍丢弃，增加队列大小或将溢出策略更改为阻塞生产者。

### 症状：只有一读取器唤醒，即使多个在等待

几个读取者阻塞在 `read()` 或 `poll()` 中，但当事件发布时只有其中一个唤醒。

**原因：** 生产者调用 `cv_signal` 而不是 `cv_broadcast`。`cv_signal` 恰好唤醒一个睡眠者；`cv_broadcast` 唤醒所有。

对于有多个并发读取者的驱动程序，`cv_broadcast` 是正确的选择，因为每个读取者可能竞争事件，它们全部需要看到唤醒以决定是否重新休眠。

### 症状：设备在分离期间挂起

`kldunload` 不返回，内核显示线程阻塞在我们分离代码中的某处。

**原因 1：** 一个调用阻塞在 `d_read` 中，我们在等待 `destroy_dev_drain` 之前没有唤醒它。在调用 `destroy_dev_drain` 之前设置 `sc_detaching`、广播并唤醒 selinfo。

**原因 2：** 一个 callout 正在进行中，我们没有排空它。在 `destroy_dev_drain` 之前调用 `callout_drain`，否则 callout 可能在我们认为完成之后重新进入驱动程序。

**原因 3：** 一个线程停在 `cv_wait_sig` 上，条件将不再被广播。确保每个等待循环将 `sc_detaching` 检查为单独的退出条件。

### 症状：读取者唤醒但发现无事可做

一个读取者被 `poll`、`kqueue` 或阻塞的 `read` 唤醒，但在返回检查队列时发现队列空，不得不回去休眠。这在正确驱动程序中偶尔发生。

**原因：** 虚假唤醒是内核生活的正常部分。调度器可能传递一个本来是为另一个等待者的唤醒，共享同一 `selinfo` 的不同事件源可能触发，或者生产者和另一个消费者之间的竞争可能在此读取者有机会查看之前排空了队列。这些情况都不表示错误。

驱动程序和读取器中的正确响应是相同的：总是在唤醒后重新检查条件，将唤醒视为可能发生了某事的提示，而不是你期望的特定事件可用的保证。驱动程序中的每个等待循环应该看起来像我们在第 3 节中建立的模式，`cv_wait_sig` 在检查真实条件的 `while` 内。每个用户态读取者应该期望在唤醒后看到 `EAGAIN` 或零长度读取并循环回到再次 poll。

如果无工作的唤醒频繁到浪费大量 CPU，考虑生产者是否比必要的更频繁调用 `selwakeup`，例如在每个中间状态改变而不是只在读取者可见事件就绪时。在生产者端合并唤醒是修复；在消费者端禁用重新检查循环不是。

### 症状：模块卸载时 panic 显示"knlist not empty"

模块卸载路径在 `knlist_destroy` 中以断言失败 panic，显示类似"knlist not empty"或在 knlist 的列表头上打印非零计数。

**原因 1：** `knlist_destroy` 在没有前置 `knlist_clear` 的情况下被调用。`knlist_destroy` 断言列表为空；列表上的活动 knote 触发 panic。检查分离路径并确认 `knlist_clear(&sc->sc_rsel.si_note, 0)` 在 `knlist_destroy` 之前运行。

**原因 2：** 一个用户态进程仍有 kqueue 注册打开，驱动程序试图在不强制 knote 离开的情况下拆除。`knlist_clear` 调用正是为处理这种情况设计的：它将每个剩余 knote 标记为 `EV_EOF | EV_ONESHOT`，以便用户态看到最终事件，注册消解。如果驱动程序跳过 `knlist_clear` 以"让用户态自然分离"，断言触发。修复是在分离中无条件调用 `knlist_clear`。

**原因 3：** 分离路径在事件传递正在进行时被调用。kqueue 框架使用自己的内部锁定保持传递和分离一致，但在 `f_event` 仍在另一个线程上运行时拆除 softc 的驱动程序会损坏生命周期。确保在进入 clear-drain-destroy 序列之前所有生产者路径已停止（例如，通过设置 `sc_detaching` 标志并排空任何工作队列）。

### 症状：f_event 中因过期 kn_hook 而 panic

内核在驱动程序的 `f_event` 函数内部 panic，回溯显示通过 `kn->kn_hook` 解引用已释放或垃圾内存。

**原因 1：** softc 在 knlist 被拆除之前被释放。驱动程序的分离路径必须按该顺序在释放 softc 之前清除和销毁 knlist。反转顺序会留下指向已释放内存的活动 knote。

**原因 2：** 一个每客户端状态对象（例如，`evdev_client`）在 knote 仍引用它时被释放。每客户端状态的清理逻辑必须在释放客户端结构之前对客户端的 selinfo 运行 `knlist_clear`/`seldrain`/`knlist_destroy` 序列，而不是之后。

**原因 3：** 另一个代码路径意外对 softc 或客户端状态调用了 `free()`。内存调试器（支持平台上的 `KASAN`，或不支持平台上的手动投毒模式）将确认内存在 `f_event` 读取它时已被释放。这是一般的内存损坏调试练习；knote 是受害者，不是原因。

### 症状：KNOTE_LOCKED 因锁未持有断言而 panic

调用 `KNOTE_LOCKED` 的生产者路径在 knlist 锁检查内部以类似"mutex not owned"的断言 panic。

**原因：** 生产者在不实际持有 knlist 锁的情况下调用 `KNOTE_LOCKED`。`KNOTE_LOCKED` 是告诉框架"跳过锁定，调用者有它"的变体；如果调用者没有，框架的断言会捕获它。修复是获取锁（通常是 softc 互斥锁）围绕 `KNOTE_LOCKED` 调用，或者改为使用 `KNOTE_UNLOCKED` 让框架自己获取锁。

仔细阅读生产者路径。一个常见错误是在生产者函数中间为了某种其他原因（例如，调用不能在锁下调用的函数）部分放弃 softc 锁，然后忘记在 `KNOTE_LOCKED` 调用之前重新获取它。修复是重新获取锁或改为调用 `KNOTE_UNLOCKED`。

### 症状：kqueue 事件到达但 kn_data 始终为零

一个 kqueue 等待者醒来并读取一个 `struct kevent`，其 `data` 字段为零，即使驱动程序有事件待处理。

**原因 1：** `f_event` 函数只在特定条件下设置 `kn->kn_data`，其他情况下保持不动。框架保留上次写入的任何值，所以上次调用的过期零持续到下次传递。修复是在 `f_event` 顶部无条件计算并赋值 `kn->kn_data`。

**原因 2：** `f_event` 函数基于队列深度以外的条件返回非零，`kn_data` 字段未更新以反映实际计数。检查 `kn_data` 被赋予真实深度，而不是布尔值，且驱动返回值的比较与它一致。

### 症状：poll() 工作但 kqueue 从不触发

基于 poll 的等待者正确看到事件，但同一文件描述符上的 kqueue 等待者从不唤醒。

**原因 1：** 驱动程序的 `d_kqfilter` 入口点不在 cdevsw 中。检查 cdevsw 初始化器并确认 `.d_kqfilter = evdemo_kqfilter` 存在。没有它，kqueue 框架无法在描述符上注册 knote。

**原因 2：** 生产者调用 `selwakeup` 但不调用 `KNOTE_LOCKED`。`selwakeup` 确实遍历附加到 selinfo 的 knlist，但只在特定条件下；想要可靠唤醒 kqueue 等待者的驱动程序应该在生产者路径中显式调用 `KNOTE_LOCKED`（或 `KNOTE_UNLOCKED`）。

**原因 3：** `f_event` 函数始终返回零。检查就绪条件是否被正确评估。添加 `printf` 确认 `f_event` 正被调用；如果是但返回零，错误在就绪检查中，不在框架中。

### 一般建议

在调试异步驱动程序时，大量添加计数器。每个 `selrecord`、每个 `selwakeup`、每个 `KNOTE_LOCKED`、每个 `pgsigio` 都应该有一个计数器。当行为看起来不对时，打印计数器是告诉你哪个机制行为异常的最快方式。

在用户态侧使用 `ktrace` 查看系统调用何时返回的确切时间。如果驱动程序认为它在时间 T 传递了唤醒，而用户态认为它在时间 T+5 秒返回，唤醒被排队但没有传递，这通常意味着某处锁被持有太长时间。

在驱动程序中和 `selwakeup` 本身上使用 DTrace 探针。`fbt:kernel:selwakeup:entry` 探针显示全系统每个 selwakeup。`fbt:kernel:pgsigio:entry` 探针对信号传递做同样的事。缺失的调用在探针输出中显示为间隙。

不要怀疑框架。内核的异步 I/O 基础设施经过实战测试，在此级别几乎从没有错误。首先怀疑你自己的驱动程序，特别是锁定顺序和附加/分离序列。

## 总结

异步 I/O 是驱动程序正确性受到最严厉考验的地方之一。同步驱动程序可以在恰好串行运行的单线程流后面隐藏许多小的锁定错误。异步驱动程序暴露其锁定规则的每个角落、生产者和消费者之间的每个竞争、以及分离路径中每个微妙的排序约束。把异步驱动程序做对比编写同步版本更难，但回报是显著的：驱动程序同时服务许多用户，干净地与用户态事件循环集成，与现代框架良好配合，并避免阻塞和忙等待的性能病态。

我们在本章学习的机制是经典的。`poll()` 和 `select()` 在每个 UNIX 系统上可移植，在驱动程序中实现它们是一个回调和 `selinfo` 的事。`kqueue()` 是现代 FreeBSD 应用程序的首选机制，它添加了一个回调和一组过滤器操作。`SIGIO` 是最古老的机制，在多线程代码中有一些锋利的边缘，但对 shell 脚本和遗留程序仍然有用。

每种机制都有相同的底层形状：等待者注册兴趣，生产者检测条件，内核向等待者传递通知。细节不同，但形状不变。理解形状使每个特定机制更容易学习。我们在第 6 节构建的内部事件队列是将形状绑在一起的东西：每种机制用队列状态表达其条件，每个生产者在通知之前更新队列。

锁定规则是最一致地区分工作的异步驱动程序和损坏的异步驱动程序的单个习惯。在检查状态之前获取 softc 互斥锁。在更新状态之前获取它。在注册等待者之前获取它。在调用锁内通知（`cv_broadcast`、`KNOTE_LOCKED`）之前获取它。在调用锁外通知（`selwakeup`、`pgsigio`）之前释放它。此模式不是美学选择；它是防止唤醒丢失和死锁的模式。当你看到驱动程序中违反此模式时，问为什么，因为十次中有九次偏差是错误。

分离序列是第二个值得纪律的习惯。在锁下设置分离标志。广播以唤醒每个等待者。向 kqueue 等待者传递 `EV_EOF`。调用 `selwakeup` 释放 poll 等待者。调用 `callout_drain` 停止生产者。调用 `destroy_dev_drain` 等待进行中的调用者。只有在这所有之后你才能安全地 `seldrain`、`knlist_destroy`、`funsetown`、`cv_destroy`、`mtx_destroy`，并释放 softc。跳过任何步骤是卸载时 panic 的方法，这些 panic 特别痛苦诊断，因为它们发生在你测试的代码之后。

可观察性习惯是第三个。你在开发时间添加的每个计数器在驱动程序投入生产时节省数小时诊断。你暴露的每个 sysctl 条目给操作员和调试器一个查看驱动程序状态的窗口，无需重建内核。你声明的每个 DTrace 探针让远处的有生产事件的工程师看到你的代码而无需发布新软件。可观察性不是奢侈；它是功能，没有它编写驱动程序就是编写你无法调试的驱动程序。

你现在拥有 FreeBSD 驱动程序作者在普通工作中需要的每个异步 I/O 工具。你可以获取阻塞字符驱动程序，审计其状态转换，识别生产者和消费者路径，添加 `poll`、`kqueue` 和 `SIGIO` 支持，并在压力下验证整个东西。这些模式泛化到字符驱动程序之外：相同的机制适用于伪设备、有控制通道的网络设备、有文件事件的文件系统以及向用户态暴露事件流的任何其他子系统。

在我们继续之前有两个最终说明。

首先，异步 I/O 不是一次性教训。你会发现，在阅读更多 FreeBSD 源码时，这些模式的变体到处出现：在使用 `grouptaskqueue` 的网络驱动程序中，在使用 `kqueue` 处理文件事件的文件系统中，在与用户态共享环形缓冲区的审计子系统中。每个变体都是相同底层思想的实例。能够在看到时识别模式比记忆任何特定 API 更有价值。

其次，当你编写自己的驱动程序时，抵制发明自己的异步机制的诱惑。内核提供的机制覆盖几乎每个用例，用户态程序知道如何使用它们。自定义机制对你来说是工作，对你的用户来说是工作，对下一个维护驱动程序的人来说也是工作。重用标准模式。它们的存在是有原因的。

## 桥接到第 36 章：在没有文档的情况下创建驱动程序

下一章改变了我们面对的挑战类型。到目前为止，每一章都假设我们为之编写的设备有文档。我们知道它的寄存器、命令集、错误代码、时序要求。本书展示了如何将该文档转化为可工作的内核代码，以及如何测试、调试和优化结果。

但并非每个设备都有文档。驱动程序作者有时会遇到没有数据手册的硬件，要么因为供应商拒绝发布，要么因为硬件太老文档已丢失，要么因为设备是有文档但有未记录更改的衍生品。在这些情况下，驱动程序编写的技艺转向逆向工程：观察设备行为，推导其接口，从间接证据而不是规范产生可工作的驱动程序。

第 36 章是关于那种技艺的。我们将看看经验丰富的作者如何接近一个未文档化的设备。我们将研究观察设备行为的工具，从总线分析器和协议嗅探器到内核自己的内置跟踪设施。我们将学习如何通过实验构建寄存器映射，如何识别跨供应商的常见命令模式，以及如何在硬件信息不完整的情况下编写正确的驱动程序。

本章的异步机制将再次出现在那里，因为事件驱动硬件正是最值得仔细逆向工程的那种硬件。缺少文档的设备仍然以事件与世界通信，通过 `poll`、`kqueue` 和 `SIGIO` 使这些事件可见通常是弄清楚设备实际在做什么的第一步。

第 34 章的调试技能也很重要，因为未文档化的设备比有文档的设备产生更多令人惊讶的行为，`KASSERT`、`WITNESS` 和 `DTrace` 是及早捕获这些惊喜的工具。我们在第 2 到 7 部分建立的基础正是逆向工程章节所需要的。

如果你从头开始阅读本书，花一点时间欣赏你已经走了多远。你从一个空的源码树和对内核一无所知开始。你现在知道如何编写支持同步和异步 I/O 的驱动程序，正确处理并发，通过计数器和 DTrace 探针观察自己的行为，并可以在活系统上调试。到这一点你已经写了足够的驱动程序，内核不再是一个陌生的环境。它是一个你知道如何工作的地方。

下一章带着那个知识并问，如果设备的文档丢失了会怎样？当你从证据而不是规范工作时，同样的技艺是什么样的？答案，事实证明，是技艺的变化比你可能想象的要少。工具相同，规则相同，你建立的习惯带你走了大部分路。

让我们看看那是如何工作的。

