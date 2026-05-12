---
title: "搭建你的实验环境"
description: "本章指导你搭建一个安全且就绪的 FreeBSD 实验环境，用于驱动程序开发。"
partNumber: 1
partName: "基础：FreeBSD、C 语言与内核"
chapter: 2
lastUpdated: "2026-04-20"
status: "complete"
author: "Edson Brandi"
reviewer: "TBD"
translator: "AI辅助翻译为简体中文"
estimatedReadTime: 60
language: "zh-CN"
---

# 搭建你的实验环境

在开始编写代码或探索 FreeBSD 内部机制之前，我们需要一个可以安全地进行实验、犯错和学习的地方。这个地方就是你的**实验环境**（lab environment）。在本章中，我们将创建你在本书其余部分将使用的基础：一个配置好用于驱动程序开发的 FreeBSD 系统。

把本章想象成准备你的**工作室**。就像木匠在制作家具之前需要合适的工作台、工具和安全装备一样，你需要一个可靠的 FreeBSD 安装、正确的开发工具，以及在出现问题时能够快速恢复的方法。内核编程是毫不留情的；驱动程序中的一个小错误就可能导致整个系统崩溃。拥有一个专门的实验环境意味着这些崩溃成为学习过程的一部分，而不是灾难。

完成本章后，你将能够：

- 理解将实验与主计算机隔离的重要性。
- 在使用虚拟机还是裸机安装之间做出选择。
- 逐步安装 FreeBSD 14.3。
- 配置系统，安装驱动程序开发所需的工具和源代码。
- 学习如何拍摄快照、管理备份和使用版本控制，这样你的进度永远不会丢失。

在此过程中，我们会安排**动手实验**，这样你不仅是阅读如何设置，而是真正动手操作。完成后，你将拥有一个安全、可重复、并为我们在后续章节中共同构建的一切做好准备的 FreeBSD 实验环境。

### 读者指南：如何使用本章

本章更偏重实践而非理论。把它看作是在开始真正实验之前设置 FreeBSD 实验环境的分步手册。你需要做出选择（虚拟机还是裸机），按照安装步骤操作，并配置你的 FreeBSD 系统。

使用本章的最佳方式是**边读边做**。不要只是浏览，而是真正安装 FreeBSD、拍摄快照、在实验日志中记录你的选择，并尝试练习。每一节都建立在前一节的基础上，所以到最后你将拥有一个与本书其余示例相匹配的完整环境。

如果你已经知道如何安装和配置 FreeBSD，可以浏览或跳过本章的部分内容，但不要跳过实验；它们确保你的设置与我们在整本书中使用的相匹配。

最重要的是要记住：这里的错误不是失败，它们是过程的一部分。这是你实验和学习的安全场所。

**预计完成本章的时间：** 1-2 小时，具体取决于你选择虚拟机还是裸机安装，以及你是否有安装操作系统的经验。

## 为什么实验环境很重要

在开始输入命令和编写我们的第一段代码之前，我们需要暂停片刻，思考*在哪里*进行所有这些工作。内核编程和设备驱动程序开发不同于编写简单的脚本或网页。当你实验内核时，你是在实验**操作系统的核心**。代码中的一个小错误可能导致机器冻结、意外重启，如果不小心的话甚至会损坏数据。

这并不意味着驱动程序开发是危险的，而是意味着我们需要采取预防措施，设置一个**安全的环境**，在这里错误是预期之中的、可恢复的，甚至作为学习过程的一部分受到鼓励。这个环境就是我们所说的**实验环境**。

你的实验环境应该是一个**专门的、隔离的环境**。就像化学家不会在没有防护设备的情况下在家庭餐桌上进行实验一样，你不应该在存放个人照片、工作文档或重要学校项目的同一台机器上运行未完成的内核代码。你需要一个专为探索和失败而设计的空间，因为失败是你学习的方式。

### 为什么不使用你的主计算机？

很容易这样想：*"我已经有一台运行 FreeBSD（或 Linux，或 Windows）的计算机，为什么不能直接使用它呢？"* 简短的回答：因为你的主计算机是用于生产力的，而不是用于实验的。如果你在测试驱动程序时意外导致内核恐慌（kernel panic），你不想丢失未保存的工作、在在线会议期间中断网络连接，甚至因为数据损坏而损坏文件系统。

你的实验设置给你自由：你可以破坏东西、重启，并在几分钟内恢复，毫无压力。这种自由对于学习至关重要。

### 虚拟机：初学者的最佳伙伴

大多数初学者（甚至许多经验丰富的开发者）从**虚拟机（VM）**开始。虚拟机就像是在你真实计算机内部运行的沙盒计算机。它的行为就像一台物理机器，但如果出现问题，你可以重置它、拍摄快照或在几分钟内重新安装 FreeBSD。你不需要备用笔记本电脑或服务器就可以开始开发驱动程序；你当前的计算机就可以托管你的实验环境。

我们将在下一节更详细地介绍虚拟化，但这里是重点：

- **安全的实验**：如果你的驱动程序导致内核崩溃，只有虚拟机会宕机，而不是你的宿主计算机。
- **轻松恢复**：快照让你可以保存虚拟机的状态，如果破坏了什么东西，可以立即回滚。
- **低成本**：不需要专门的硬件。
- **可移植**：你可以在计算机之间移动你的虚拟机。

### 般机：当你需要真实的东西时

有时只有真正的硬件才行，例如，如果你想为 PCIe 卡或需要直接访问机器总线的 USB 设备开发驱动程序。在这些情况下，在虚拟机中测试可能不够，因为并非所有虚拟化解决方案都能可靠地传递硬件。

如果你有一台旧的备用 PC，在那里安装 FreeBSD 可以为你提供最接近现实的测试环境。但请记住：般机设置没有虚拟机那样的安全网。如果你导致内核崩溃，机器会重启，你需要手动恢复。这就是为什么我建议从虚拟机开始，即使你最终会转向般机进行特定的硬件项目。

### 现实世界的例子

为了更具体地说明：想象你正在编写一个错误地解引用 NULL 指针的简单驱动程序（如果这听起来很技术性，不要担心，你以后会学到的）。在虚拟机上，你的系统可能会冻结，但通过重置和快照回滚，你可以在几分钟内恢复正常。在般机上，同样的错误可能导致文件系统损坏，需要漫长的恢复过程。这就是为什么安全的实验环境如此有价值。

### 动手实验：准备你的实验心态

在我们甚至安装 FreeBSD 之前，让我们做一个简单的练习来进入正确的心态。

1. 准备一个笔记本（实体的或数字的）。这将是你的**实验日志**。
2. 写下：
   - 今天的日期
   - 你将用于 FreeBSD 实验环境的机器（虚拟机还是物理机）
   - 你选择该选项的原因（安全性、便利性、访问真实硬件等）
3. 写下第一条记录：*"实验环境设置开始。目标：为 FreeBSD 驱动程序实验构建安全环境。"*

这可能感觉不必要，但保持**实验日志**将帮助你跟踪进度、以后重复成功的设置，以及在出现问题时进行调试。在专业驱动程序开发中，工程师会保留非常详细的笔记；现在开始这个习惯将使你像真正的系统开发者一样思考和工作。

### 小结

我们介绍了**实验环境**的概念，以及为什么它对安全的驱动程序开发如此重要。无论你选择虚拟机还是备用物理计算机，关键是拥有一个专门的、可以犯错的地方。

在下一节中，我们将更仔细地看看**虚拟机与般机设置的优缺点**。在该节结束时，你将确切知道哪种设置对你开始 FreeBSD 工作有意义。

## 选择你的设置：虚拟机还是般机

既然你理解了为什么专门的实验环境很重要，下一个问题是：**你应该在哪里构建它？** FreeBSD 可以以两种主要方式运行你的实验：

1. **在虚拟机（VM）内部**，运行在你现有操作系统之上。
2. **直接在物理硬件上**（通常称为*般机*）。

两种选择都可行，并且都在现实世界的 FreeBSD 开发中广泛使用。正确的选择取决于你的目标、你的硬件和你的舒适程度。让我们并排比较它们。

### 虚拟机：你的沙盒盒子

**虚拟机**是一个软件，让你可以像运行单独的计算机一样运行 FreeBSD，但在你现有计算机内部。流行的虚拟机解决方案包括：

- **VirtualBox**（免费且跨平台，非常适合初学者）。
- **VMware Workstation / Fusion**（商业软件，精致，广泛使用）。
- **bhyve**（FreeBSD 原生虚拟机监控程序，如果你想在 FreeBSD *上*运行 FreeBSD，这是理想选择）。

为什么开发者喜欢用虚拟机进行内核工作：

- **快照救急**：在尝试有风险的代码之前，你拍摄快照。如果系统恐慌或崩溃，你可以在几秒钟内恢复。
- **一台机器上有多个实验环境**：你可以创建多个 FreeBSD 实例，每个用于不同的项目。
- **易于共享**：你可以导出虚拟机镜像并与队友共享。

**何时选择虚拟机：**

- 你刚开始，想要尽可能最安全的环境。
- 你没有备用硬件可以专用。
- 你预期在学习过程中经常导致内核崩溃（你会的）。

### 般机：真家伙

**直接在硬件上**运行 FreeBSD 是你能获得的最接近"真实的东西"。这意味着 FreeBSD 作为机器上唯一的操作系统启动，直接与 CPU、内存、存储和外设通信。

优势：

- **真正的硬件测试**：在为 PCIe、USB 或其他物理设备开发驱动程序时必不可少。
- **性能**：没有虚拟机开销。你可以完全访问系统资源。
- **准确性**：一些错误只会在般机上出现，特别是与时序相关的问题。

劣势：

- **没有安全网**：如果内核崩溃，你的整台机器都会宕机。
- **恢复需要时间**：如果你损坏了操作系统，可能需要重新安装 FreeBSD。
- **需要专用硬件**：你需要一台可以完全用于实验的备用 PC 或笔记本电脑。

**何时选择般机：**

- 你计划为在虚拟机中不能很好工作的硬件开发驱动程序。
- 你已经有一台可以完全用于 FreeBSD 的备用机器。
- 你想要最大的真实性，即使这意味着更多风险。

### 混合策略

许多专业开发者**两者都用**。他们在虚拟机中进行大部分实验和原型设计，那里安全快速，只有当驱动程序足够稳定可以针对真实硬件测试时才转向般机。你不必永远只选择一种，你可以今天从虚拟机开始，以后如果需要再添加般机。

### 快速比较表

| 特性              | 虚拟机                         | 般机                         |
| ----------------- | ------------------------------ | ---------------------------- |
| **安全性**        | 很高（快照、回滚）             | 低（需要手动恢复）           |
| **性能**          | 稍低（开销）                   | 完整系统性能                 |
| **硬件访问**      | 有限/模拟设备                  | 完整、真实的硬件             |
| **设置难度**      | 简单快速                       | 中等（完整安装）             |
| **成本**          | 无（在你的 PC 上运行）         | 需要专用机器                 |
| **最适合**        | 初学者、安全学习               | 高级硬件测试                 |

### 动手实验：决定你的路径

1. 查看你当前的资源。你是否有一台可以用于 FreeBSD 实验的备用笔记本电脑或台式机？
   - 如果是 -> 般机对你来说是一个选择。
   - 如果否 -> 虚拟机是完美的起点。
2. 在你的**实验日志**中写下：
   - 你将使用哪个选项（虚拟机还是般机）。
   - 你选择它的原因。
   - 你预期的任何限制（例如，"使用虚拟机，可能还无法测试 USB 直通"）。
3. 如果你选择虚拟机，请记下你将使用哪个虚拟机监控程序（VirtualBox、VMware、bhyve 等）。

这个决定不会永远锁定你。你总是可以稍后添加第二个环境。现在的目标是开始一个安全、可靠的设置。

### 小结

我们比较了虚拟机和般机设置，看到了各自的优势和权衡。对于大多数初学者来说，从虚拟机开始是安全性、便利性和灵活性的最佳平衡。如果你以后需要与真实硬件交互，可以将般机系统添加到你的工具箱中。

在下一节中，我们将卷起袖子，进行实际的 **FreeBSD 14.3 安装**，首先在虚拟机中，然后我们将涵盖般机安装的关键点。这就是你的实验环境真正开始成形的地方。

## 安装 FreeBSD（虚拟机和般机）

此时，你已经选择是在**虚拟机**还是**般机**上设置 FreeBSD。现在是时候实际安装将作为我们所有实验基础的操作系统了。我们将专注于 **FreeBSD 14.3**，这是本文写作时最新的稳定版本，这样你所做的一切都与本书中的示例相匹配。

FreeBSD 的安装程序是基于文本的，但不要被它吓倒，它很简单，在不到 20 分钟内你将拥有一个准备好开发的系统。

### 下载 FreeBSD ISO

1. 访问 FreeBSD 官方下载页面：
    https://www.freebsd.org/where
2. 选择 **14.3-RELEASE** 镜像。
   - 如果你是在虚拟机中安装，下载 **amd64 Disk1 ISO**（`FreeBSD-14.3-RELEASE-amd64-disc1.iso`）。
   - 如果你是在真实硬件上安装，同样的 ISO 可以使用，不过如果你想要写入 USB 闪存盘，也可以考虑 **memstick 镜像**。

### 在 VirtualBox 中安装 FreeBSD（分步进行）

如果你还没有安装 **VirtualBox**，你需要在创建 FreeBSD 虚拟机之前设置它。VirtualBox 适用于 Windows、macOS、Linux 甚至 Solaris 宿主机。从官方网站下载最新版本：

https://www.virtualbox.org/wiki/Downloads

选择与你的宿主操作系统匹配的软件包（例如，Windows 宿主机或 macOS 宿主机），下载并按照安装程序操作。安装过程很简单，只需要几分钟。完成后，启动 VirtualBox，然后你就可以创建你的第一个 FreeBSD 虚拟机了。

现在你已经准备好了，让我们通过 VirtualBox 流程，因为这是大多数读者最容易的入门点。在 VMware 或 bhyve 中的步骤类似。

要开始，在你的计算机上执行 VirtualBox 应用程序，在主屏幕左侧栏选择 **Home**，然后点击 **New**，按照以下步骤操作：

1. 在 VirtualBox 中**创建新虚拟机**：

   - VM Name: `FreeBSD Lab`
   
   - VM Folder: 选择一个目录来存放你的 FreeBSD 虚拟机
   
   - ISO Image: 选择你上面下载的 FreeBSD ISO 文件
   
   - OS Edition: 留空
   
   - OS: 选择 `BSD`
   
   - OS Distribution: 选择 `FreeBSD`
   
   - OS Version: 选择 `FreeBSD (64-bit)`
   
     点击 **Next** 继续

![image-20250823183742036](https://freebsd.edsonbrandi.com/images/image-20250823183742036.png)

2. **分配资源**：

   - Base Memory: 至少 2 GB（推荐 4 GB）

   - Number of CPUs: 2 或更多（如果可用）

   - Disk Size: 30 GB 或更多
   
     点击 **Next** 继续

![image-20250823183937505](https://freebsd.edsonbrandi.com/images/image-20250823183937505.png)

3. **查看你的选项**：如果你对摘要满意，点击 **Finish** 创建虚拟机。

![image-20250823184925505](https://freebsd.edsonbrandi.com/images/image-20250823184925505.png)

4. **启动虚拟机**：点击绿色 **Start** 按钮。启动虚拟机时，它将使用你创建时指定的 FreeBSD 安装盘启动。

![image-20250823185259010](https://freebsd.edsonbrandi.com/images/image-20250823185259010.png)

5. **启动虚拟机**：虚拟机将显示 FreeBSD 启动加载器，按 **1** 继续启动进入 FreeBSD。

![image-20250823185756980](https://freebsd.edsonbrandi.com/images/image-20250823185756980.png)

6. **运行安装程序**：启动过程中，安装程序将自动运行，选择 **[ Install ]** 继续。

![image-20250823190016799](https://freebsd.edsonbrandi.com/images/image-20250823190016799.png)

7. **键盘布局**：选择你偏好的语言/键盘，默认是 US 布局。按 **Enter** 继续。

![image-20250823190046619](https://freebsd.edsonbrandi.com/images/image-20250823190046619.png)

8. **主机名**：输入你的实验环境的主机名，在示例中我选择了 `fbsd-lab`。按 **Enter** 继续。

![image-20250823190129010](https://freebsd.edsonbrandi.com/images/image-20250823190129010.png)

9. **发行版选择**：保留默认值（基本系统、内核）。按 **Enter** 继续。

![image-20250823190234155](https://freebsd.edsonbrandi.com/images/image-20250823190234155.png)

10. **分区**：选择 *Auto (UFS)*，除非你以后想学习 ZFS。按 **Enter** 继续。

![image-20250823190350815](https://freebsd.edsonbrandi.com/images/image-20250823190350815.png)

11. **分区**：选择 **[ Entire Disk ]**。按 **Enter** 继续。

![image-20250823190450571](https://freebsd.edsonbrandi.com/images/image-20250823190450571.png)

12. **分区方案**：选择 **GPT GUID Partition Table**。按 **Enter** 继续。

![image-20250823190622981](https://freebsd.edsonbrandi.com/images/image-20250823190622981.png)

13. **分区编辑器**：接受默认值，选择 **[Finish]**。按 **Enter** 继续。

![image-20250823190742861](https://freebsd.edsonbrandi.com/images/image-20250823190742861.png)

14. **确认**：在此屏幕中，你将确认要继续 FreeBSD 安装，此确认后安装程序将开始向硬盘写入数据。要继续安装，选择 **[Commit]** 并按 **Enter** 继续。

![image-20250823190903913](https://freebsd.edsonbrandi.com/images/image-20250823190903913.png)

15. **校验和验证**：在过程开始时，FreeBSD 安装程序将检查安装文件的完整性。

![image-20250823191020839](https://freebsd.edsonbrandi.com/images/image-20250823191020839.png)

16. **归档解压**：文件验证后，安装程序将文件解压到你的硬盘

![image-20250823191053163](https://freebsd.edsonbrandi.com/images/image-20250823191053163.png)

17. **Root 密码**：当安装程序完成解压文件后，你需要为 root 访问权限选择一个密码。选择一个你会记住的密码。按 **Enter** 继续。

![image-20250823191405000](https://freebsd.edsonbrandi.com/images/image-20250823191405000.png)

18. **网络配置**：选择你想要使用的网络接口（**em0**）并按 **Enter** 继续。

![image-20250823191520068](https://freebsd.edsonbrandi.com/images/image-20250823191520068.png)

19. **网络配置**：选择 **[ Yes ]** 在你的网络接口上启用 **IPv4** 并按 **Enter** 继续。

![image-20250823191559429](https://freebsd.edsonbrandi.com/images/image-20250823191559429.png)

20. **网络配置**：选择 **[ Yes ]** 在你的网络接口上启用 **DHCP**，如果你更喜欢使用静态 IP 地址，选择 **[ No ]**。按 **Enter** 继续。

![image-20250823191626027](https://freebsd.edsonbrandi.com/images/image-20250823191626027.png)

21. **网络配置**：选择 **[ No ]** 在你的网络接口上禁用 **IPv6** 并按 **Enter** 继续。

![image-20250823191705347](https://freebsd.edsonbrandi.com/images/image-20250823191705347.png)

22. **网络配置**：输入你首选的 DNS 服务器的 IP 地址，在示例中我使用的是 Google DNS。按 **Enter** 继续。

![image-20250823191748088](https://freebsd.edsonbrandi.com/images/image-20250823191748088.png)

23. **时区选择器**：为你的 FreeBSD 系统选择所需的时区，在此示例中我使用 **UTC**。按 **Enter** 继续。

![image-20250823191820859](https://freebsd.edsonbrandi.com/images/image-20250823191820859.png)

24. **确认时区**：确认你要使用的时区。选择 **[ YES ]** 并按 **Enter** 继续。

![image-20250823191849469](https://freebsd.edsonbrandi.com/images/image-20250823191849469.png)

25. **时间和日期**：安装程序会让你有机会手动调整日期和时间。通常选择 **[ Skip ]** 是安全的。按 **Enter** 继续。

![image-20250823191926758](https://freebsd.edsonbrandi.com/images/image-20250823191926758.png)

![image-20250823191957558](https://freebsd.edsonbrandi.com/images/image-20250823191957558.png)

26. **系统配置**：安装程序会让你选择一些在启动时启动的服务，选择 **ntpd** 并按 **Enter** 继续。

![image-20250823192055299](https://freebsd.edsonbrandi.com/images/image-20250823192055299.png)

27. **系统加固**：安装程序会让你启用一些在启动时应用的安全加固，暂时接受默认值并按 **Enter** 继续。

![image-20250823192128039](https://freebsd.edsonbrandi.com/images/image-20250823192128039.png)

28. **固件检查**：安装程序将验证你的硬件组件是否需要任何特定固件才能正常工作，并在需要时安装它。按 **Enter** 继续。

![image-20250823192211024](https://freebsd.edsonbrandi.com/images/image-20250823192211024.png)

29. **添加用户账户**：安装程序会让你有机会向系统添加普通用户。选择 **[ Yes ]** 并按 **Enter** 继续。

![image-20250823192233281](https://freebsd.edsonbrandi.com/images/image-20250823192233281.png)

30. **创建用户**：安装程序会要求你输入用户信息并回答一些基本问题，你应该选择你想要的**用户名**和**密码**，你可以**接受所有问题的默认答案**，除了问题***"Invite USER into other groups?"***，对于这个问题你需要回答"**wheel**"，这是 FreeBSD 中的组，允许你在普通会话期间使用命令 `su` 成为 root。

![image-20250823192452683](https://freebsd.edsonbrandi.com/images/image-20250823192452683.png)

31. **创建用户**：在你回答了所有问题并创建了用户后，FreeBSD 安装程序会问你是否想添加另一个用户。只需按 **Enter** 接受默认答案（否）即可进入最终配置菜单。

![image-20250823192600794](https://freebsd.edsonbrandi.com/images/image-20250823192600794.png)

32. **最终配置**：此时你已经完成了 FreeBSD 安装。此最终菜单允许你查看和更改你在之前步骤中所做的选择。选择 **Exit** 离开安装程序并按 **Enter**。

![image-20250823192642433](https://freebsd.edsonbrandi.com/images/image-20250823192642433.png)

33. **手动配置**：安装程序会问你是否想打开 shell 对新安装的系统进行手动配置。选择 **[ No ]** 并按 **Enter**。

![image-20250823192704460](https://freebsd.edsonbrandi.com/images/image-20250823192704460.png)

34. **弹出安装光盘**：在重启虚拟机之前，我们需要弹出用于安装的虚拟光盘。为此，在 VirtualBox 虚拟机窗口底部状态栏的 CD/DVD 图标上左键单击，然后在菜单"Remove Disk From Virtual Drive"上右键单击。

![image-20250823193213602](https://freebsd.edsonbrandi.com/images/image-20250823193213602.png)

如果由于某种原因你收到消息说虚拟光盘正在使用中无法弹出，点击按钮"Force Unmount"，之后你可以继续重启。

![image-20250823193252804](https://freebsd.edsonbrandi.com/images/image-20250823193252804.png)

35. **重启你的虚拟机**：在此菜单中按 **Enter** 重启你的 FreeBSD 虚拟机。

![image-20250823192732830](https://freebsd.edsonbrandi.com/images/image-20250823192732830.png)

### 在般机上安装 FreeBSD

如果你使用的是备用 PC 或笔记本电脑，你需要从可启动的 USB 闪存盘直接安装 FreeBSD。方法如下：

#### 步骤 1：准备 USB 闪存盘

- 你需要一个至少有 **2 GB 容量**的 USB 闪存盘。
- 确保备份其中的任何数据；该过程将擦除所有内容。

#### 步骤 2：下载正确的镜像

- 对于 USB 安装，下载 **memstick 镜像**（`FreeBSD-14.3-RELEASE-amd64-memstick.img`）。

#### 步骤 3：创建可启动 USB（Windows 说明）

在 Windows 上，最简单的工具是 **Rufus**：

1. 从 https://rufus.ie 下载 Rufus。
2. 插入你的 USB 闪存盘。
3. 打开 Rufus 并选择：
   - **Device**: 你的 USB 闪存盘。
   - **Boot selection**: 你下载的 FreeBSD memstick `.img` 文件。
   - **Partition scheme**: MBR
   - **Target System**: BIOS（或 UEFI-CSM）
   - **File system**: 保留默认。
4. 点击 *Start*。Rufus 会警告你所有数据将被销毁，接受它。
5. 等待过程完成。你的 USB 闪存盘现在可以启动了。

![image-20250823210622431](https://freebsd.edsonbrandi.com/images/image-20250823210622431.png)

如果你已经有类 UNIX 系统，你可以使用 `dd` 命令从终端创建 USB：

```console
% sudo dd if=FreeBSD-14.3-RELEASE-amd64-memstick.img of=/dev/da0 bs=1M
```

将 `/dev/da0` 替换为你的 USB 设备路径。

#### 步骤 4：从 USB 启动

1. 将 USB 闪存盘插入目标机器。
2. 进入 BIOS/UEFI 启动菜单（通常在启动时按 F12、Esc 或 Del）。
3. 选择 USB 驱动器作为启动设备。

#### 步骤 5：运行安装程序

FreeBSD 启动后，按照上述相同的安装程序步骤操作，我们在那里选择键盘布局、主机名、发行版等。

安装完成后，移除 USB 闪存盘并重启。FreeBSD 现在将从硬盘启动。

### 首次启动

安装后，你会看到 FreeBSD 启动菜单：

![image-20250823213050882](https://freebsd.edsonbrandi.com/images/image-20250823213050882.png)

然后是登录提示：

![image-20250823212856938](https://freebsd.edsonbrandi.com/images/image-20250823212856938.png)

恭喜！你的 FreeBSD 实验机器现在已启动并准备好配置。

### 小结

你刚刚完成了最重要的里程碑之一：在你的专用实验环境中安装 FreeBSD 14.3。无论是在虚拟机还是般机上，你现在都有了一个可以安全地破坏、修复和重建的干净系统，供你学习。

在下一节中，我们将介绍安装后应该立即进行的**初始配置**：设置网络、启用基本服务以及为开发工作准备系统。

## 首次启动和初始配置

当你的 FreeBSD 系统在安装后完成首次重启时，你会看到与 Windows 或 macOS 非常不同的东西。没有华丽的桌面，没有图标，没有"入门"向导。相反，你会看到**登录提示**。

不用担心，这是正常且有意为之的。FreeBSD 是一个类 UNIX 系统，设计用于稳定性和灵活性，而不是华丽的第一印象。默认环境故意保持最小，这样你（管理员）可以保持完全控制。把这想象成你第一次坐在新配置的实验机器前：shell 是空的，工具还没有安装，但系统已准备好被塑造成你的工作所需要的样子。

在本节中，我们将执行**必要的第一步**，使你的 FreeBSD 实验环境舒适、安全并为驱动程序开发做好准备。

### 登录

在登录提示处：

- 输入你在安装期间创建的用户名。
- 输入你的密码（记住 UNIX 系统在输入密码时不显示 `*`）。

你现在已作为普通用户进入 FreeBSD。

![image-20250823212710535](https://freebsd.edsonbrandi.com/images/image-20250823212710535.png)

### 切换到 Root 用户

一些任务，如安装软件或编辑系统文件，需要 **root 权限**。你应该避免一直以 root 登录（如果你输错命令风险太大），但在需要时临时切换到 root 是一个好习惯：

```console
% su -
Password:
```

输入你在安装期间设置的 root 密码。提示符将从 `%` 变为 `#`，这意味着你现在已是 root。

![image-20250823213238499](https://freebsd.edsonbrandi.com/images/image-20250823213238499.png)

### 设置主机名和时间

你的系统需要一个名称和正确的时间设置。

- 检查主机名：

  ```
  % hostname
  ```

  如果你想更改它，编辑 `/etc/rc.conf`：

  ```
  # ee /etc/rc.conf
  ```

  添加或调整这一行：

  ```
  hostname="fbsd-lab"
  ```

- 要同步时间，确保 NTP 已启用（如果你在安装期间选择了它，通常已启用）。你可以用以下命令测试：

  ```
  % date
  ```

  如果时间错误，暂时手动更正：

  ```
  # date 202508231530
  ```

  （这将日期/时间设置为 2025 年 8 月 23 日 15:30 - 格式为 `YYYYMMDDhhmm`）。

### 网络基础

大多数使用 DHCP 的安装"开箱即用"。要验证：

```console
% ifconfig
```

你应该看到一个接口（如 `em0`、`re0` 或虚拟机中的 `vtnet0`）及其 IP 地址。如果没有，你可能需要在 `/etc/rc.conf` 中启用 DHCP：

```ini
ifconfig_em0="DHCP"
```

将 `em0` 替换为你从 `ifconfig` 获得的实际接口名称。

![image-20250823213433266](https://freebsd.edsonbrandi.com/images/image-20250823213433266.png)

### 安装和配置 `sudo`

作为最佳实践，你应该使用 `sudo` 而不是为每个特权命令切换到 root。

1. 安装 sudo：

   ```
   # pkg install sudo
   ```

2. 将你的用户添加到 `wheel` 组（如果你创建用户时没有这样做）：

   ```
   # pw groupmod wheel -m yourusername
   ```

3. 现在，让我们启用 `wheel` 组使用 `sudo`。

   执行命令 `visudo` 并在打开的文件编辑器中搜索这些行：

	```sh
	##
	## User privilege specification
	##
	root ALL=(ALL:ALL) ALL

	## Uncomment to allow members of group wheel to execute any command
	# %wheel ALL=(ALL:ALL) ALL

	## Same thing without a password
	#%wheel ALL=(ALL:ALL) NOPASSWD: ALL
	```
删除 `#%wheel ALL=(ALL:ALL) NOPASSWD: ALL` 行中的 `#`，将光标放在要删除的字符上使用方向键，然后按 **x**，保存文件并退出编辑器，按 **ESC** 然后输入 **:wq** 并按 **Enter**。

4. 要验证它是否按预期工作，注销并重新登录，然后运行：

   ```
   % sudo whoami
   root
   ```

现在你的用户可以安全地执行管理任务，而无需一直以 root 登录。

### 更新系统

在安装开发工具之前，将你的系统更新到最新版本：

```console
# freebsd-update fetch install
# pkg update
# pkg upgrade
```

这确保你运行的是最新的安全补丁。

![image-20250823215034288](https://freebsd.edsonbrandi.com/images/image-20250823215034288.png)

### 创建舒适的环境

即使是小的调整也能让你的日常工作更顺畅：

- **启用命令历史和补全**（如果你使用 `tcsh`，用户的默认 shell，这已包含）。

- 在你的主目录中**编辑 `.cshrc`** 以添加有用的别名：

  ```
  alias ll 'ls -lh'
  alias cls 'clear'
  ```

- **安装更友好的编辑器**（可选）：

  ```
  # pkg install nano
  ```

### 实验环境的基本加固

即使这是一个**实验环境**，添加几层安全性也很重要。如果你启用 **SSH**，这一点尤其重要，无论你是在笔记本电脑上的虚拟机中运行 FreeBSD 还是在备用物理机器上运行。一旦 SSH 开启，你的系统就会接受远程登录，这意味着你应该采取一些预防措施。

你有两种简单的方法。选择你喜欢的一种；两种都适合实验环境。

#### 选项 A：最小的 `pf` 规则（阻止除 SSH 外的所有入站连接）

1. 启用 `pf` 并创建一个小规则集：

   ```
   # sysrc pf_enable="YES"
   # nano /etc/pf.conf
   ```

   在 `/etc/pf.conf` 中放入以下内容（将 `vtnet0`/`em0` 替换为你的接口）：

   ```sh
   set skip on lo
   
   ext_if = "em0"           # 虚拟机通常使用 vtnet0；在般机上你可能会看到 em0/re0/igb0 等。
   tcp_services = "{ ssh }"
   
   block in all
   pass out all keep state
   pass in on $ext_if proto tcp to (self) port $tcp_services keep state
   ```

2. 启动 `pf`（它将在重启后保持）：

   ```
   # service pf start
   ```

**虚拟机注意：** 如果你的虚拟机使用 NAT，你可能还需要在虚拟机监控程序中配置**端口转发**（例如 VirtualBox：主机端口 2222 -> 客户机端口 22），然后通过 `localhost -p 2222` 进行 SSH 连接。上面的 `pf` 规则仍在客户机**内部**适用。

#### 选项 B：使用内置的 `ipfw` 预设（非常适合初学者）

1. 使用 `workstation` 预设启用 `ipfw` 并打开 SSH：

   ```
   # sysrc firewall_enable="YES"
   # sysrc firewall_type="workstation"
   # sysrc firewall_myservices="22/tcp"
   # sysrc firewall_logdeny="YES"
   # service ipfw start
   ```

   - `workstation` 提供了一个"保护此机器"的有状态规则集，易于上手。
   - `firewall_myservices` 列出你想允许的入站服务；这里我们允许 TCP/22 上的 SSH。
   - 你可以稍后根据需要切换到其他预设（例如 `client`、`simple`）。

**提示：** 选择 `pf` **或** `ipfw` 中的一个，不要两者都用。对于第一个实验环境，`ipfw` 预设是最快的路径；小型 `pf` 规则集同样很好且非常明确。

#### 保持补丁更新

定期运行这些命令以保持最新：

```console
% sudo freebsd-update fetch install
% sudo pkg update && pkg upgrade
```

**为什么在虚拟机中要费心？** 因为虚拟机仍然是你网络上的真实机器。这里的良好习惯为你以后的生产环境做好准备。

### 小结

你的 FreeBSD 系统不再是一个裸骨架，它现在有了主机名、工作的网络、更新的基本系统和一个具有 `sudo` 访问权限的用户账户。你还应用了一小层但有意义的加固：一个简单但仍允许 SSH 的防火墙，以及定期更新。这些不仅仅是可选的调整，它们是让你成为负责任的系统开发者的习惯。

在下一节中，我们将安装驱动程序编程所需的**开发工具**，包括编译器、调试器、编辑器和 FreeBSD 源代码树本身。这就是你的实验环境从空白画布转变为真正的开发工作站的时刻。

## 为开发准备系统

现在你的 FreeBSD 实验环境已安装、更新并进行了轻度加固，是时候将其转变为适当的**驱动程序开发环境**了。此步骤添加构建、调试和版本控制内核代码所需的组件：编译器、调试器、版本控制系统和 FreeBSD 源代码树。没有这些，你将无法构建或测试我们将在后面章节中编写的代码。

好消息是 FreeBSD 已经包含了我们需要的大部分内容。在本节中，我们将安装缺失的部分，验证一切正常，并运行一个微小的"hello 模块"测试来证明你的实验环境已准备好进行驱动程序开发。

### 安装开发工具

FreeBSD 在基本系统中附带 **Clang/LLVM**。要确认：

```console
% cc --version
FreeBSD clang version 19.1.7 (...)
```

如果你看到类似上面的版本字符串，你就准备好编译 C 代码了。

不过，你还需要一些额外的工具：

```console
# pkg install git gmake gdb
```

- `git`：版本控制系统。
- `gmake`：GNU make（一些项目除了 FreeBSD 自己的 `make` 外还需要它）。
- `gdb`：GNU 调试器。

### 选择编辑器

每个开发者都有自己喜欢的编辑器。FreeBSD 默认包含 `vi`，功能强大但学习曲线陡峭。如果你是新手，可以安全地从 **`ee`（Easy Editor）** 开始，它会提供屏幕帮助指导你，或者安装 **`nano`**，它有更简单的快捷键，如 Ctrl+O 保存和 Ctrl+X 退出：

```console
% sudo pkg install nano
```

但迟早，你会想学习 **`vim`**，即 `vi` 的改进版本。它快速、高度可配置，在 FreeBSD 开发中被广泛使用。它的一大优势是**语法高亮**，使 C 代码更易于阅读。

#### 配置 Vim 的语法高亮

1. 安装 vim：

   ```
   # pkg install vim
   ```

2. 在你的主目录中创建配置文件：

   ```
   % ee ~/.vimrc
   ```

3. 添加这些行：

   ```
   syntax on
   set number
   set tabstop=8
   set shiftwidth=8
   set expandtab
   set autoindent
   set background=dark
   ```

   - `syntax on` -> 启用语法高亮。
   - `set number` -> 显示行号。
   - tab/缩进设置遵循 **FreeBSD 编码风格**（8 空格制表符，而不是 4）。
   - `set background=dark` -> 使颜色在深色终端上可读。

4. 保存文件并打开一个 C 程序：

   ```
   % vim hello.c
   ```

   你现在应该看到彩色的关键字、字符串和注释。

#### Nano 语法高亮

如果你更喜欢 `nano`，它也支持语法高亮。配置存储在 `/usr/local/share/nano/` 中。要为 C 启用它：

```console
% cp /usr/local/share/nano/c.nanorc ~/.nanorc
```

现在用 `nano` 打开 `.c` 文件，你会看到基本的高亮。

#### Easy Editor (ee)

`ee` 是最简单的选择，没有高亮，只是纯文本。它对初学者来说是安全的，非常适合快速编辑配置文件，但你可能会在驱动程序开发中逐渐不再使用它。

### 访问文档

**man 页面**是你内置的参考库。试试这个：

```console
% man 9 malloc
```

这将显示 `malloc(9)` 内核函数的手册页面。`(9)` 节号表示它是**内核接口**的一部分，这是我们以后大部分时间要待的地方。

其他有用的命令：

- `man 1 ls` -> 用户命令文档。
- `man 5 rc.conf` -> 配置文件格式。
- `man 9 intro` -> 内核编程接口概述。

### 安装 FreeBSD 源代码树

大多数驱动程序开发需要访问 FreeBSD 内核源代码。你将其存储在 `/usr/src` 中。

从这里开始，每当本书引用一个文件，如 `/usr/src/sys/kern/kern_module.c`，它指的是你即将克隆的源代码树中的一个真实文件。`/usr/src` 是 FreeBSD 系统上 FreeBSD 源代码树的传统位置，后面章节中每个 `/usr/src/...` 形式的路径都直接映射到下面 `src` 检出下的一个文件。后面的章节不会重新解释这个约定；它们只会引用该路径并期望你在该位置找到它。

用 Git 克隆：

```console
% sudo git clone --branch releng/14.3 --depth 1 https://git.FreeBSD.org/src.git /usr/src
```

这将需要几分钟并下载几 GB。完成后，你将拥有完整的内核源代码树。

用以下命令验证：

```console
% ls /usr/src/sys
```

你应该看到 `dev`、`kern`、`net` 和 `vm` 等目录。这些是 FreeBSD 内核所在的地方。

#### 警告：使你的头文件与运行中的内核匹配。

FreeBSD 对根据与你正在运行的内核匹配的确切头文件集构建可加载内核模块非常严格。如果你的内核是从 14.3-RELEASE 构建的，但 `/usr/src` 指向不同的分支或版本，你可能会遇到令人困惑的编译或加载错误。为了避免本书中练习的麻烦，请确保你在 `/usr/src` 中安装了 **FreeBSD 14.3** 源代码树，并且它与你的运行内核匹配。一个快速的检查是 `freebsd-version -k`，它应该打印 `14.3-RELEASE`，而你的 `/usr/src` 应该在 `releng/14.3` 分支上，如上所述。

**提示**：如果 `/usr/src` 已经存在并指向其他地方，你可以重新定位它：

```console
% sudo git -C /usr/src fetch --all --tags
% sudo git -C /usr/src checkout releng/14.3
% sudo git -C /usr/src pull --ff-only
```

有了内核和头文件对齐，你的示例模块将可靠地构建和加载。

### 测试你的工具包：一个"Hello 内核模块"

为了确认一切正常，让我们编译并加载一个微小的内核模块。这还不是一个驱动程序，但它证明你的实验环境可以构建并与内核交互。

1. 创建一个名为 `hello_world.c` 的文件：

```c
/*
 * hello_world.c - 简单的 FreeBSD 内核模块
 * 在加载和卸载时打印消息
 */

#include <sys/param.h>
#include <sys/kernel.h>
#include <sys/module.h>
#include <sys/systm.h>

/*
 * 加载处理程序 - 在模块加载时调用
 */
static int
hello_world_load(module_t mod, int cmd, void *arg)
{
    int error = 0;

    switch (cmd) {
    case MOD_LOAD:
        printf("Hello World! Kernel module loaded.\n");
        break;
    case MOD_UNLOAD:
        printf("Goodbye World! Kernel module unloaded.\n");
        break;
    default:
        error = EOPNOTSUPP;
        break;
    }

    return (error);
}

/*
 * 模块声明
 */
static moduledata_t hello_world_mod = {
    "hello_world",      /* 模块名称 */
    hello_world_load,   /* 事件处理程序 */
    NULL                /* 额外数据 */
};

/*
 * 向内核注册模块
 * DECLARE_MODULE(name, data, sub-system, order)
 */
DECLARE_MODULE(hello_world, hello_world_mod, SI_SUB_DRIVERS, SI_ORDER_MIDDLE);
MODULE_VERSION(hello_world, 1);
```

1. 创建一个 `Makefile`：

```console
# Makefile for hello_world kernel module

KMOD=   hello_world
SRCS=   hello_world.c

.include <bsd.kmod.mk>
```

1. 构建模块：

```console
# make
```

这应该创建一个文件 `hello.ko`。

1. 加载模块：

```console
# kldload ./hello_world.ko
```

检查系统日志中的消息：

```console
% dmesg | tail -n 5
```

你应该看到：

`Hello World! Kernel module loaded.`

1. 卸载模块：

```console
# kldunload hello_world.ko
```

再次检查：

```console
% dmesg | tail -n 5
```

你应该看到：

`Goodbye World! Kernel module unloaded.`

### 动手实验：验证你的开发设置

1. 安装 `git`、`gmake` 和 `gdb`。
2. 用 `% cc --version` 验证 Clang 正常工作。
3. 安装并配置带语法高亮的 `vim`，或者如果你喜欢，设置 `nano`。
4. 将 FreeBSD 14.3 源代码树克隆到 `/usr/src`。
5. 编写、编译并加载 `hello_world` 内核模块。
6. 在你的**实验日志**中记录结果（你看到"Hello, kernel world!"消息了吗？）。

### 小结

你现在已经为你的 FreeBSD 工作室配备了必要的工具：编译器、调试器、版本控制、文档和内核源代码本身。你甚至构建并加载了你的第一个内核模块，证明了你的设置端到端工作。

在下一节中，我们将看看**使用快照和备份**，这样你就可以自由地实验，而不必担心失去进度。这将给你信心去承担更大的风险，并在出问题时快速恢复。

## 使用快照和备份

设置**实验环境**的最大优势之一是你可以在没有恐惧的情况下进行实验。当你编写内核代码时，错误是不可避免的：错误的指针、无限循环或糟糕的卸载例程都可能导致整个操作系统崩溃。与其担心，你可以把崩溃当作学习过程的一部分，*如果*你有快速恢复的方法。

这就是**快照和备份**发挥作用的地方。快照让你可以"冻结"你的 FreeBSD 实验环境在一个安全点，然后在出问题时立即回滚。备份保护你的重要文件，比如你的代码或实验笔记，以防你需要重新安装系统。

在本节中，我们将探索两者。

### 虚拟机中的快照

如果你在虚拟机（VirtualBox、VMware、bhyve）中运行 FreeBSD，你有一个巨大的安全网：**快照**。

- 在 **VirtualBox** 或 **VMware** 中，快照从 GUI 管理，你可以通过几次点击保存、恢复和删除它们。
- 在 **bhyve** 中，快照通过**存储后端**管理，通常是 ZFS。你对保存虚拟机磁盘镜像的数据集进行快照，并在需要时回滚。

#### VirtualBox 的示例工作流程

1. 在完成初始设置后关闭你的 FreeBSD 虚拟机。

2. 在 VirtualBox 管理器中，选择你的虚拟机 -> **Snapshots** -> 点击 **Take**。

3. 命名为：`Clean FreeBSD 14.3 Install`。

   ![image-20250823231838089](https://freebsd.edsonbrandi.com/images/image-20250823231838089.png)

   ![image-20250823231940246](https://freebsd.edsonbrandi.com/images/image-20250823231940246.png)

4. 之后，在测试有风险的内核代码之前，拍摄另一个快照：`Before Hello Driver`。

   ![image-20250823232320392](https://freebsd.edsonbrandi.com/images/image-20250823232320392.png)

5. 如果系统崩溃或你破坏了网络，只需恢复快照。

![image-20250823232420760](https://freebsd.edsonbrandi.com/images/image-20250823232420760.png)

#### bhyve 中的示例工作流程（使用 ZFS）

如果你的虚拟机磁盘存储在 ZFS 数据集上，例如 `/zroot/vm/freebsd.img`：

1. 在实验前创建快照：

   ```
   # zfs snapshot zroot/vm@clean-install
   ```

2. 进行更改、测试代码，甚至导致内核崩溃。

3. 立即回滚：

   ```
   # zfs rollback zroot/vm@clean-install
   ```

### 般机上的快照

如果你直接在硬件上运行 FreeBSD，你没有 GUI 快照的便利。但如果你用 **ZFS** 安装 FreeBSD，你仍然可以访问相同的快照工具。

使用 ZFS：

```console
# zfs snapshot -r zroot@clean-install
```

- 这会创建你的根文件系统的快照。

- 如果出现问题，你可以回滚：

  ```
  # zfs rollback -r zroot@clean-install
  ```

ZFS 快照是即时的，不会复制数据，它们只跟踪更改。对于严肃的般机实验环境，强烈推荐使用 ZFS。

如果你用 **UFS** 而不是 ZFS 安装，你将没有快照功能。在这种情况下，依赖**备份**（见下文），如果你想要这个安全网，也许以后考虑用 ZFS 重新安装。

### 备份你的工作

快照保护**系统状态**，但你也需要保护你的**工作**——你的驱动程序代码、笔记和 Git 仓库。

简单的策略：

- **Git**：如果你使用 Git（你应该使用），将你的代码推送到远程服务，如 GitHub 或 GitLab。这是最好的备份。

- **Tar 包**：创建项目的归档：

  ```
  % tar czf mydriver-backup.tar.gz mydriver/
  ```

- **复制到宿主机**：如果使用虚拟机，将文件从客户机复制到宿主机（VirtualBox 共享文件夹，或通过 SSH 使用 `scp`）。

**注意**：把你的虚拟机视为可抛弃的，但你的**代码是珍贵的**。在测试危险的更改之前始终备份。

### 动手实验：破坏和修复

1. 如果你在 VirtualBox/VMware 中：

   - 创建一个名为 `Before Break` 的快照。
   - 以 root 身份运行一些无害但破坏性的操作（例如，删除 `/tmp/*`）。
   - 恢复快照并确认 `/tmp` 恢复正常。

2. 如果你在 bhyve 中使用 ZFS 支持的存储：

   - 对你的虚拟机数据集进行快照。
   - 在客户机内删除一个测试文件。
   - 回滚 ZFS 快照。

3. 如果你在使用 ZFS 的般机上：

   - 拍摄快照 `zroot@before-break`。
   - 删除一个测试文件。
   - 用 `zfs rollback` 回滚并确认文件已恢复。

4. 备份你的 `hello_world` 内核模块源代码：

   ```
   % tar czf hello-backup.tar.gz hello_world/
   ```

在你的**实验日志**中记录：你使用了哪种方法、花了多长时间，以及你现在对实验有多大信心。

### 小结

通过学习使用**快照和备份**，你为你的实验环境添加了最重要的安全网之一。现在你可以崩溃、破坏或错误配置 FreeBSD 并在几分钟内恢复。这种自由正是让实验环境如此有价值的原因——它让你专注于学习，而不是害怕犯错。

在下一节中，我们将设置**使用 Git 进行版本控制**，这样你就可以跟踪进度、管理实验并分享你的驱动程序。

## 设置版本控制

到目前为止，你已经准备好了 FreeBSD 实验环境、安装了工具，甚至构建了你的第一个内核模块。但想象一下：你对驱动程序进行了更改、测试它，突然一切都不工作了。你希望能回到上一个可工作的版本。或者你想保持两个不同的实验而不混淆它们。

这正是开发者使用**版本控制系统**的原因，这些工具记录你工作的历史，允许你回滚到以前的状态，并使与他人共享代码变得容易。在 FreeBSD 世界（以及大多数开源项目）中，标准是 **Git**。

在本节中，你将从第一天开始学习如何使用 Git 管理你的驱动程序。

### 为什么版本控制很重要

- **跟踪你的更改**：每个实验、每个修复、每个错误都被保存。
- **安全地撤销**：如果你的代码停止工作，你可以回滚到一个已知的良好版本。
- **组织实验**：你可以在"分支"中开发新想法而不破坏主代码。
- **分享你的工作**：如果你想获得他人的反馈或发布你的驱动程序，Git 使其变得容易。
- **专业习惯**：每个严肃的软件项目（包括 FreeBSD 本身）都使用版本控制。

把 Git 想象成你代码的**实验笔记本**，只是更聪明：它不仅记录你做了什么，还可以将你的代码恢复到过去任何时间点。

### 安装 Git

如果你还没有在第 2.5 节中安装 Git，现在安装：

```console
# pkg install git
```

检查版本：

```console
% git --version
git version 2.45.2
```

### 配置 Git（你的身份）

在使用 Git 之前，配置你的身份，这样你的提交会被正确标记：

```console
% git config --global user.name "Your Name"
% git config --global user.email "you@example.com"
```

如果你只是在本地实验，这不一定是你的真实姓名或电子邮件，但如果你曾经公开分享代码，使用一致的内容会有帮助。

你可以用以下命令检查你的设置：

```console
% git config --list
```

### 创建你的第一个仓库

让我们将你的 `hello_world` 内核模块置于版本控制之下。

1. 导航到你创建 `hello_world.c` 和 `Makefile` 的目录。

2. 初始化 Git 仓库：

   ```
   % git init
   ```

   这会创建一个隐藏的 `.git` 目录，Git 在其中存储其历史。

3. 添加你的文件：

   ```
   % git add hello_world.c Makefile
   ```

4. 进行第一次提交：

   ```
   % git commit -m "Initial commit: hello_world kernel module"
   ```

5. 检查历史：

   ```
   % git log
   ```

   你应该看到你的提交列出。

### 提交的最佳实践

- **编写清晰的提交消息**：描述更改了什么以及为什么。

  - 不好：`fix stuff`
  - 好：`Fix null pointer dereference in hello_loader()`

- **经常提交**：小的提交更容易理解和回滚。

- **保持实验分离**：如果你尝试一个新想法，创建一个分支：

  ```
  % git checkout -b experiment-null-fix
  ```

即使你从不分享你的代码，这些习惯也会帮助你更快地调试和学习。

### 使用远程仓库（可选）

目前，你可以将所有内容保持在本地。但如果你想在机器之间同步代码或公开分享，你可以将其推送到像 **GitHub** 或 **GitLab** 这样的远程服务。

基本工作流程：

```console
% git remote add origin git@github.com:yourname/mydriver.git
% git push -u origin main
```

这在实验环境中是可选的，但如果你想在云端备份工作，非常有用。

### 动手实验：为你的驱动程序进行版本控制

1. 在你的 `hello` 模块目录中初始化 Git 仓库。

2. 进行第一次提交。

3. 编辑 `hello_world.c`（例如，更改消息文本）。

4. 运行：

   ```
   % git diff
   ```

   以查看确切更改了什么。

5. 用清晰的消息提交更改。

6. 在你的**实验日志**中记录：

   - 你进行了多少次提交。
   - 每次提交做了什么。
   - 如果出现问题你会如何回滚。

### 小结

你现在已经迈出了使用 Git 的第一步，Git 是你开发者工具箱中最重要的工具之一。从现在开始，你在本书中编写的每个驱动程序都应该有自己的 Git 仓库。这样，你永远不会失去进度，你将始终有实验的记录。

在下一节中，我们将讨论**记录你的工作**——这是专业开发者的另一个关键习惯。一个写得好的 README 或提交消息可以是代码在一年后还能理解和必须从头重写之间的区别。

## 记录你的工作

软件开发不仅仅是编写代码，还要确保*你*（有时还有其他人）以后能理解那段代码。在开发 FreeBSD 驱动程序时，你经常会在几周或几个月后回到一个项目，问自己：*"我为什么写这个？我在测试什么？我改了什么？"*

没有文档，你会浪费数小时重新发现自己的思路。有了好的笔记，你可以准确地从你停下的地方继续。

把文档想象成你实验环境的**记忆**。就像科学家保留详细的实验笔记本一样，开发者应该保留清晰的笔记、README 和提交消息。

### 为什么文档很重要

- **未来的你会感谢你**：今天看起来显而易见的细节在一个月后会被遗忘。
- **调试变得更容易**：当出现问题时，笔记帮助你理解发生了什么变化。
- **分享更顺畅**：如果你发布你的驱动程序，其他人可以从你的 README 中学习。
- **专业习惯**：FreeBSD 本身以其高质量的文档而闻名，遵循这一传统使你的工作自然地融入生态系统。

### 编写简单的 README

每个项目都应该以 `README.md` 文件开始。至少包括：

1. **项目名称**：

   ```
   Hello Kernel Module
   ```

2. **描述**：

   ```
   A simple "Hello, kernel world!" module for FreeBSD 14.3.
   ```

3. **如何构建**：

   ```
   % make
   ```

4. **如何加载/卸载**：

   ```
   # kldload ./hello_world.ko
   # kldunload hello_world
   ```

5. **注释**：

   ```
   This was created as part of my driver development lab, Chapter 2.
   ```

### 使用提交消息作为文档

Git 提交消息是一种文档形式。它们共同讲述你项目的故事。遵循这些提示：

- 用现在时态编写提交消息（"Add feature"，而不是"Added feature"）。
- 让第一行简短（50 个字符或更少）。
- 如果需要，添加一个空行，然后是更长的解释。

示例：

```text
Fix panic when unloading hello module

The handler did not check for NULL before freeing resources,
causing a panic when unloading. Added a guard condition.
```

### 保持实验日志

在第 2.1 节中，我们建议开始一个实验日志。现在是养成这个习惯的好时机。在你的 Git 仓库根目录下保留一个文本文件（例如 `LABLOG.md`）。每次尝试新东西时，添加一个简短的条目：

```text
2025-08-23
- Built hello module successfully.
- Confirmed "Hello, kernel world!" appears in dmesg.
- Tried unloading/reloading multiple times, no errors.
- Next step: experiment with passing parameters to the module.
```

这个日志不需要精美——它只是为你准备的。以后在调试时，这些笔记可能是无价的。

### 辅助工具

- **Markdown**：README 和实验日志都可以用 Markdown（`.md`）编写，它在纯文本中易于阅读，在 GitHub/GitLab 上格式精美。
- **man 页面**：始终记下你使用的 man 页面（例如 `man 9 module`）。这将提醒你你的来源。
- **截图/日志**：如果你使用虚拟机，拍摄重要步骤的截图，或将命令输出保存到文件中（`dmesg > dmesg.log`）。

### 动手实验：记录你的第一个模块

1. 在你的 `hello_world` 模块目录中，创建一个 `README.md`，描述它的功能、如何构建以及如何加载/卸载。

2. 将你的 `README.md` 添加到 Git 并提交：

   ```
   % git add README.md
   % git commit -m "Add README for hello_world module"
   ```

3. 开始一个 `LABLOG.md` 文件并记录今天的活动。

4. 用以下命令查看你的 Git 历史：

   ```
   % git log --oneline
   ```

   以查看你的提交如何讲述你项目的故事。

### 小结

你现在学会了如何记录你的 FreeBSD 驱动程序实验，这样你永远不会忘记你做了什么或为什么。有了 `README`、有意义的提交消息和实验日志，你正在建立让你成为更专业、更高效开发者的习惯。

在下一节中，我们将总结本章，回顾你构建的一切：一个拥有正确工具、备份、版本控制和文档的安全 FreeBSD 实验环境，所有这些都为第 3 章中更深入地探索 FreeBSD 本身做好了准备。

## 总结

恭喜！你已经构建了你的 FreeBSD 实验环境！

在本章中，你：

- 理解了为什么**安全的实验环境**对驱动程序开发至关重要。
- 为你的情况选择了正确的设置——**虚拟机**或**般机**。
- 逐步安装了 **FreeBSD 14.3**。
- 执行了**初始配置**，包括网络、用户和基本加固。
- 安装了必要的**开发工具**：编译器、调试器、Git 和编辑器。
- 在你的编辑器中设置了**语法高亮**，使 C 代码更易于阅读。
- 将 **FreeBSD 14.3 源代码树**克隆到 `/usr/src`。
- 编译并测试了你的第一个**内核模块**。
- 学会了如何使用**快照和备份**从错误中快速恢复。
- 开始使用 **Git** 进行版本控制，并添加了 **README** 和**实验日志**来记录你的工作。

对于一章来说，这是令人印象深刻的进度。你现在拥有一个完整的工作室：一个可以随意编写、构建、测试、破坏和恢复的 FreeBSD 系统。

最重要的收获不仅仅是你安装的工具，而是**心态**：

- 预期错误。
- 记录你的过程。
- 使用快照、备份和 Git 来恢复和学习。

### 练习

1. **快照**
   - 拍摄你的虚拟机快照或般机上的 ZFS 快照。
   - 故意进行更改（例如，删除 `/tmp/testfile`）。
   - 回滚并验证系统已恢复。
2. **版本控制**
   - 对你的 `hello_world.c` 内核模块进行小的编辑。
   - 用 Git 提交更改。
   - 使用 `git log` 和 `git diff` 查看你的历史。
3. **文档**
   - 在你的 `LABLOG.md` 中添加新条目，描述今天的工作。
   - 用一个新注释更新你的 `README.md`（例如，提到 `uname -a` 的输出）。
4. **反思**
   - 在你的实验日志中回答：*我在第 2 章设置的三个最重要的安全网是什么？*

### 展望未来

在下一章中，我们将走进你的新 FreeBSD 实验环境，探索如何**使用系统本身**。你将学习 UNIX 命令、导航和文件管理的基础知识。这些技能将使你在 FreeBSD 中如鱼得水，为即将到来的更高级主题做好准备。

你的实验环境已经准备好了。现在该学习如何在其中工作了。
