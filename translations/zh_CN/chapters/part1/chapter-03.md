---
title: "UNIX 入门指南"
description: "本章提供 UNIX 和 FreeBSD 基础知识的实践介绍。"
partNumber: 1
partName: "基础：FreeBSD、C 和内核"
chapter: 3
lastUpdated: "2026-04-20"
status: "complete"
author: "Edson Brandi"
reviewer: "TBD"
translator: "AI辅助翻译为简体中文"
language: "zh-CN"
estimatedReadTime: 120
---

# UNIX 入门指南

既然你的 FreeBSD 系统已经安装并运行起来了，是时候开始在系统中自如地工作了。FreeBSD 不仅是一个操作系统，它是五十多年前从 UNIX 开始的悠久传统的一部分。

在本章中，我们将首次真正深入探索这个系统。你将学习如何在文件系统中导航、在 shell 中运行命令、管理进程以及安装应用程序。在这个过程中，你会看到 FreeBSD 如何继承 UNIX 的简约和一致性哲学，以及为什么这对我们未来的驱动程序开发者来说很重要。

把本章当作你在 FreeBSD 中工作的**生存指南**。在我们开始深入 C 代码和内核内部机制之前，你需要能够自如地在系统中移动、操作文件以及使用每个开发者日常依赖的工具。

到本章结束时，你不仅会知道 *UNIX 是什么*；你还将能够像用户和有抱负的系统程序员一样自信地使用 FreeBSD。

## 读者指南：如何使用本章

本章不仅仅是可以略读的内容，它被设计为既是**参考手册**又是**实践训练营**。所需时间取决于你的学习方法：

- **仅阅读：** 以舒适的初学者速度阅读文本和示例，大约需要 **2 小时**。
- **阅读 + 实验：** 如果在阅读过程中暂停并在自己的 FreeBSD 系统中运行每个实践实验，大约需要 **4 小时**。
- **阅读 + 挑战：** 如果还要完成末尾的全部 46 道挑战练习，大约需要 **6 小时或更多**。

建议：不要试图一次性完成所有内容。将本章分成几个部分，每完成一部分后运行实验，然后再继续。当你感到自信并想检验自己的掌握程度时，再完成挑战练习。

## 引言：为什么 UNIX 很重要

在我们开始为 FreeBSD 编写设备驱动程序之前，我们需要停下来谈谈它们赖以生存的基础：**UNIX**。

你将为 FreeBSD 编写的每个驱动程序、你将探索的每个系统调用、你将阅读的每个内核消息，只有当你理解它们所在的操作系统时才有意义。对于初学者来说，UNIX 的世界可能感觉神秘，充满了奇怪的命令和与 Windows 或 macOS 截然不同的哲学。但一旦你理解了它的逻辑，你会发现它不仅易于接近，而且优雅。

本章旨在为你提供 FreeBSD 中 UNIX 的**入门介绍**。到最后，你将能够自如地导航系统、处理文件、运行命令、管理进程、安装应用程序，甚至编写小型脚本来自动化你的任务。这些是每个 FreeBSD 开发者的日常技能，在我们开始内核开发之前绝对是必不可少的。

### 为什么在编写驱动程序之前要学习 UNIX？

可以这样想：如果编写驱动程序就像制造发动机，那么 UNIX 就是围绕着它的整辆汽车。你需要知道燃料从哪里进入、仪表盘如何工作、控制装置做什么，然后才能安全地在引擎盖下更换零件。

以下是学习 UNIX 基础至关重要的几个原因：

- **UNIX 中的一切都是相互关联的。** 文件、设备、进程，它们都遵循一致的规则。一旦你知道这些规则，系统就变得可预测。
- **FreeBSD 是 UNIX 的直系后裔。** 命令、文件系统布局和整体哲学不是附加组件，而是其 DNA 的一部分。
- **驱动程序与用户空间集成。** 即使你的代码在内核中运行，它也会与用户程序、文件和进程交互。理解用户空间环境有助于你设计感觉自然和直观的驱动程序。
- **调试需要 UNIX 技能。** 当你的驱动程序出现异常行为时，你将依赖 `dmesg`、`sysctl` 和 shell 命令等工具来弄清楚发生了什么。

### 本章你将学到什么

到本章结束时，你将：

- 理解什么是 UNIX 以及 FreeBSD 如何融入其家族。
- 能够使用 shell 运行命令和管理文件。
- 在 FreeBSD 文件系统中导航并知道各种东西存放在哪里。
- 管理用户、组和文件权限。
- 监控进程和系统资源。
- 使用 FreeBSD 的包管理器安装和删除应用程序。
- 用 shell 脚本自动化任务。
- 使用 `dmesg` 和 `sysctl` 等工具窥视 FreeBSD 内部。

一路上，我会给你**实践实验**让你练习。仅阅读 UNIX 是不够的；你需要**触摸系统**。每个实验都涉及你将在 FreeBSD 安装上运行的真实命令，所以当你到达本章末尾时，你不仅会理解 UNIX，还会自信地使用它。

### 通向设备驱动程序的桥梁

如果这是一本关于编写驱动程序的书，为什么我们要花整整一章来讲 UNIX 基础？因为驱动程序不会孤立存在。当你最终加载自己的内核模块时，你会看到它出现在 `/dev` 下面。当你测试它时，你将使用 shell 命令来读写它。当你调试它时，你将依赖系统日志和监控工具。

所以把本章当作成为驱动程序开发者之前所需的**操作系统素养**。一旦你掌握了它，其他一切都会变得不那么令人生畏，更加合乎逻辑。

### 小结

在开篇部分，我们探讨了为什么 UNIX 对于任何想要编写 FreeBSD 驱动程序的人来说都很重要。驱动程序不会孤立存在；它们存在于遵循 UNIX 继承的规则、约定和哲学的更大操作系统中。理解这个基础是让其他一切——从使用 shell 到调试驱动程序——变得合乎逻辑而不是神秘的关键。

怀着这样的动机，是时候问下一个自然的问题了：**UNIX 到底是什么？** 为了前进，我们将仔细看看它的历史、指导原则以及至今仍影响 FreeBSD 的关键概念。

## 什么是 UNIX？

在你能够自如使用 FreeBSD 之前，理解什么是 UNIX 以及为什么它很重要会有所帮助。UNIX 不仅是一段软件，它是一个操作系统家族、一套设计选择，甚至是一种塑造了计算领域五十多年的哲学。FreeBSD 是其最重要的现代后裔之一，所以学习 UNIX 就像研究家谱，看看 FreeBSD 在其中的位置。

### UNIX 简史

UNIX 诞生于 **1969 年** 的贝尔实验室，当时 Ken Thompson 和 Dennis Ritchie 为 PDP-7 小型机创建了一个轻量级操作系统。在大型机庞大、昂贵且复杂的时代，UNIX 脱颖而出，因为它**小巧、优雅，专为实验而设计**。

**1973 年用 C 语言重写**是转折点。有史以来第一次，操作系统变得可移植：你可以通过重新编译将 UNIX 移植到不同的硬件上，而不必从头开始重写一切。这在 1970 年代是闻所未闻的，永远改变了系统设计的轨迹。

**伯克利的 BSD** 是直接导致 FreeBSD 的那部分故事。加州大学伯克利分校的研究生和研究人员获取了 AT&T 的 UNIX 源代码，并用现代功能扩展了它：

- **虚拟内存**（使程序不受物理 RAM 的限制）。
- **网络**（至今仍为互联网提供动力的 TCP/IP 协议栈）。
- **C shell**，带有脚本和作业控制功能。

在 **1990 年代**，在解决了关于 UNIX 源代码的法律纠纷后，FreeBSD 项目启动了。其使命：自由、开放地延续 BSD 传统，供任何人使用、修改和分享。

**今天**，FreeBSD 是那一线谱系的直接延续。它不是 UNIX 的模仿；它是活得好好的 UNIX 遗产。

你可能会想，*"我为什么要关心？"* 你应该关心，因为当你窥视 `/usr/src` 或输入 `ls` 和 `ps` 这样的命令时，你不仅在使用软件，你还在受益于数十年的问题解决和工艺，这些工作是数千名开发者在很久以前构建和完善这些工具时完成的。

### UNIX 哲学

UNIX 不仅是一个系统，它是一种**思维方式**。理解其哲学将使其他一切——从基本命令到设备驱动程序——感觉更加自然。

1. **做好一件事。**
   UNIX 不提供庞大的全能程序，而是提供专注的工具。

   例如：`grep` 只搜索文本。它不打开文件、编辑文件或格式化结果，它把那些留给其他工具。

2. **一切皆文件。**
   文件不仅是文档，它们是你与几乎一切交互的方式：设备、进程、套接字、日志。

   类比：把整个系统想象成一个图书馆。每本书、每张桌子，甚至管理员的笔记本都是同一个归档系统的一部分。

3. **构建小工具，然后组合它们。**
   这就是**管道操作符（`|`）**的天才之处。你取一个程序的输出并将其用作另一个程序的输入。

   例如：

   ```sh
   ps -aux | grep ssh
   ```

   这里，一个程序列出所有进程，另一个只筛选与 SSH 相关的进程。两个程序都不知道对方的存在，但 shell 将它们粘合在一起。

4. **尽可能使用纯文本。**
   文本文件易于阅读、编辑、共享和调试。FreeBSD 的 `/etc/rc.conf`（系统配置）就是一个纯文本文件。没有二进制注册表，没有专有格式。

当你开始编写设备驱动程序时，你会在各处看到这种哲学：你的驱动程序将在 `/dev` 下暴露一个**简单接口**，行为可预测，并与其他工具平滑集成。

### 今天的类 UNIX 系统

今天"UNIX"一词较少指单一操作系统，更多指一个**类 UNIX 系统家族**。

- **FreeBSD** - 本书的重点。用于服务器、网络设备、防火墙和嵌入式系统。以可靠性和文档著称。许多商业设备（路由器、存储系统）在底层默默运行着 FreeBSD。
- **Linux** - 创建于 1991 年，受 UNIX 原则启发。在数据中心、嵌入式设备和超级计算机中流行。与 FreeBSD 不同，Linux 不是直接的 UNIX 后裔，但共享相同的接口和理念。
- **macOS 和 iOS** - 构建在 Darwin 之上，一个基于 BSD 的基础。macOS 是经过 UNIX 认证的操作系统，意味着其命令行工具的行为像 FreeBSD 的。如果你使用 Mac，你已经有了一个 UNIX 系统。
- **其他** - 商业变体如 AIX、Solaris 或 HP-UX 仍然存在，但在企业环境之外很少见。

为什么这很重要：一旦你学会了 FreeBSD，你会在几乎任何其他类 UNIX 系统上感到自在。命令、文件系统布局和哲学都会延续。

### 关键概念和术语

以下是本书中你会看到的一些基本 UNIX 术语：

- **内核（Kernel）** - 操作系统的核心。它管理内存、CPU、设备和进程。你的驱动程序将驻留在这里。
- **Shell** - 解释你命令的程序。它是你与系统对话的主要工具。
- **用户空间（Userland）** - 内核之外的一切：命令、库、守护进程。这是你作为用户大部分时间呆的地方。
- **守护进程（Daemon）** - 后台服务（如用于远程登录的 `sshd` 或用于计划任务的 `cron`）。
- **进程（Process）** - 正在运行的程序。每个命令创建一个进程。
- **文件描述符（File descriptor）** - 内核给程序用于处理文件或设备的数字句柄。例如，0 = 标准输入，1 = 标准输出，2 = 标准错误。

提示：暂时不用担心记忆这些。把它们当作你稍后会再次遇到的角色。当你编写驱动程序时，你会像老朋友一样认识它们。

### UNIX 与 Windows 的区别

如果你主要使用 Windows，UNIX 的方法起初会感觉不同。以下是几点对比：

- **驱动器 vs. 统一树**
  Windows 使用驱动器字母（`C:\`、`D:\`）。UNIX 有一个以 `/` 为根的单一树。磁盘和分区被挂载到这棵树中。
- **注册表 vs. 文本文件**
  Windows 将设置集中存储在注册表中。UNIX 使用 `/etc` 和 `/usr/local/etc` 下的纯文本配置文件。你可以用任何文本编辑器打开它们。
- **GUI 优先 vs. CLI 优先**
  虽然 Windows 假设图形界面，UNIX 将命令行视为主要工具。图形环境存在，但 shell 总是可用。
- **权限模型**
  UNIX 从第一天起就是多用户的。每个文件都有针对所有者、组和其他人的权限（读、写、执行）。这使得安全性和共享更简单、更一致。

这些差异解释了为什么 UNIX 通常感觉"更严格"但也更透明。一旦你习惯了它，这种一致性就会变成巨大的优势。

### 日常生活中的 UNIX

即使你以前从未登录过 FreeBSD 系统，UNIX 已经在你周围：

- 你的 Wi-Fi 路由器或 NAS 可能运行 FreeBSD 或 Linux。
- Netflix 使用 FreeBSD 服务器提供流媒体视频。
- Sony 的 PlayStation 使用基于 FreeBSD 的操作系统。
- macOS 和 iOS 是 BSD UNIX 的直系后裔。
- Android 手机运行 Linux，另一个类 UNIX 系统。

学习 FreeBSD 不仅是为了编写驱动程序，它是学习**现代计算的语言**。

### 实践实验：你的第一个 UNIX 命令

让我们来点具体的。在上一章安装的 FreeBSD 中打开终端，尝试：

```sh
% uname -a
```

这会打印系统详情：操作系统、系统名称、发行版本、内核构建和机器类型。在 FreeBSD 14.x 上，你可能会看到：

```text
FreeBSD freebsd.edsonbrandi.com 14.3-RELEASE FreeBSD 14.3-RELEASE releng/14.3-n271432-8c9ce319fef7 GENERIC amd64
```

现在尝试这些命令：

```sh
% date
% whoami
% hostname
```

- `date` - 显示当前时间和日期。
- `whoami` - 告诉你登录的用户账户。
- `hostname` - 显示机器的网络名称。

最后，用 UNIX 的*"一切皆文件"*理念做一个小实验：

```sh
% echo "Hello FreeBSD" > /tmp/testfile
% cat /tmp/testfile
```

你刚刚创建了一个文件、写入内容、然后读回。这是你稍后与自己驱动程序对话所用的相同模型。

### 小结

在本节中，你学到 UNIX 不仅是一个操作系统，更是一个塑造了现代计算的理念和设计原则家族。你看到 FreeBSD 如何作为 BSD UNIX 的直系后裔融入这段历史，为什么其小工具和纯文本哲学使其有效，以及你将作为驱动程序开发者依赖的许多概念——如进程、守护进程和文件描述符——从 UNIX 诞生之初就是其一部分。

但知道 UNIX 是什么只完成了一半。要真正使用 FreeBSD，你需要一种**与它交互的方式**。这就是 shell 的作用——让你说出系统语言的命令解释器。在下一节，我们将开始使用 shell 来运行命令、探索文件系统，并获得每个 FreeBSD 开发者日常依赖的工具的实践经验。

## Shell：你通向 FreeBSD 的窗口

既然你知道了什么是 UNIX 以及为什么它很重要，是时候开始**与系统对话**了。在 FreeBSD（和其他类 UNIX 系统）中，你通过 **shell** 来做到这一点。

把 shell 想象成既是**解释器**又是**翻译器**：你以人类可读的形式输入命令，shell 将其传递给操作系统执行。它是你与 UNIX 世界之间的窗口。

### 什么是 Shell？

核心上，shell 只是一个程序，但非常特殊。它监听你输入的内容，弄清楚你的意图，然后请求内核执行。

一些常见的 shell 包括：

- **sh** - 原始的 Bourne shell。简单可靠。
- **csh / tcsh** - C shell 及其增强版本，具有受 C 语言启发的脚本功能。tcsh 是 FreeBSD 新用户的默认 shell。
- **bash** - Bourne Again Shell，在 Linux 中非常流行。
- **zsh** - 现代、用户友好的 shell，具有许多便利功能。

在 FreeBSD 14.x 上，如果你作为普通用户登录，你可能会使用 **tcsh**。如果你作为 root 管理员登录，你可能会看到 **sh**。如果你不确定使用的是哪个 shell，不必担心，我们稍后会介绍如何检查。

这对驱动程序开发者很重要：你将不断使用 shell 来编译、加载和测试你的驱动程序。知道如何导航它就像知道如何转动汽车的点火钥匙一样重要。

### 如何知道你正在使用哪个 Shell

FreeBSD 自带多个 shell，你可能会注意到它们之间的细微差异——例如，提示符可能看起来不同，或者某些快捷键的行为可能不同。不必担心：**核心 UNIX 命令在任何 shell 中都一样**。不过，了解你当前使用的是哪个 shell 仍然有帮助，特别是如果你后来决定编写脚本或自定义环境时。

输入：

```sh
% echo $SHELL
```

你会看到类似这样的内容：

```sh
/bin/tcsh
```

或

```sh
/bin/sh
```

这告诉你你的默认 shell。你现在不需要更改它；只需意识到 shell 可能看起来略有不同但共享相同的基本命令。

**实践提示**
还有一种快速检查当前进程运行哪个 shell 的方法：

```sh
% echo $0
```

这可能会显示 `-tcsh`、`sh` 或其他内容。它与 `$SHELL` 略有不同，因为 `$SHELL` 告诉你**默认 shell**（登录时获得的），而 `$0` 告诉你**当前实际运行的 shell**。如果你在会话中启动了不同的 shell（例如通过在提示符下输入 `sh`），`$0` 会反映这一点。

### 命令的结构

每个 shell 命令都遵循相同的简单模式：

```sh
command [options] [arguments]
```

- **command** - 你想运行的程序。
- **options** - 改变其行为的标志（通常以 `-` 开头）。
- **arguments** - 命令的目标，如文件名或目录。

例如：

```sh
% ls -l /etc
```

- `ls` = 列出目录内容。
- `-l` = "长格式"选项。
- `/etc` = 参数（要列出的目录）。

这种一致性是 UNIX 的优势之一：一旦你学会了模式，每个命令都感觉熟悉。

### 初学者必备命令

我们将介绍你将经常使用的核心命令。

#### 目录导航

- **pwd** - 打印工作目录
  显示你在文件系统中的位置。

  ```sh
  % pwd
  ```

  输出：

  ```
  /home/dev
  ```

- **cd** - 切换目录
  移动你到另一个目录。

  ```sh
  % cd /etc
  % pwd
  ```

  输出：

  ```
  /etc
  ```

- **ls** - 列表
  显示目录的内容。

  ```sh
  % ls
  ```

  输出可能包括：

  ```
  rc.conf   ssh/   resolv.conf
  ```

**提示**：尝试 `ls -lh` 获取人类可读的文件大小。

#### 文件和目录管理

- **mkdir** - 创建目录

  ```sh
  % mkdir projects
  ```

- **rmdir** - 删除目录（仅限空目录）

  ```sh
  % rmdir projects
  ```

- **cp** - 复制

  ```sh
  % cp file1.txt file2.txt
  ```

- **mv** - 移动（或重命名）

  ```sh
  % mv file2.txt notes.txt
  ```

- **rm** - 删除

  ```sh
  % rm notes.txt
  ```

**警告**：`rm` 不会要求确认。一旦删除，文件就消失了，除非你有备份。这是初学者常见的陷阱。

#### 查看文件内容

- **cat** - 连接并显示文件内容

  ```sh
  % cat /etc/rc.conf
  ```

- **less** - 滚动查看文件内容

  ```sh
  % less /etc/rc.conf
  ```

  使用箭头键或空格键，按 `q` 退出。

- **head / tail** - 显示文件开头或结尾，`-n` 参数指定要查看的行数

  ```sh
  % head -n 5 /etc/rc.conf
  % tail -n 5 /etc/rc.conf
  ```

#### 编辑文件

迟早，你需要编辑配置文件或源文件。FreeBSD 附带几个编辑器，各有不同的优势：

- **ee (Easy Editor)**

  - 默认安装。
  - 设计为对初学者友好，屏幕顶部有可见菜单。
  - 要保存，按 **Esc**，然后选择 *"Leave editor"* → *"Save changes."*
  - 如果你从未使用过 UNIX 编辑器，这是很好的选择。

- **vi / vim**

  - 传统 UNIX 编辑器，总是可用。
  - 非常强大，但学习曲线陡峭。
  - 初学者经常被困住，因为 `vi` 启动在*命令模式*而不是插入模式。
  - 要开始输入文本：按 **i**，写入文本，然后按 **Esc** 接着输入 `:wq` 保存并退出。
  - 你现在不需要掌握它，但每个系统管理员和开发者最终都会至少学习 `vi` 的基础知识。

- **nano**

  - 不是 FreeBSD 基础系统的一部分，但可以作为 root 运行以下命令轻松安装：

    ```sh
    # pkg install nano
    ```

  - 非常初学者友好，屏幕底部列出快捷键。
  - 如果你来自像 Ubuntu 这样的 Linux 发行版，你可能已经知道它。

**初学者提示**
从 `ee` 开始，在 FreeBSD 上习惯编辑文件。一旦准备好，学习 `vi` 的基础知识——它会永远在那里为你服务，即使在救援环境或最小系统中，其他编辑器都没有安装。

##### **实践实验：你的第一次编辑**

1. 用 `ee` 创建和编辑新文件：

   ```sh
   % ee hello.txt
   ```

   写一行短文本，保存并退出。

2. 用 `vi` 尝试同样的操作：

   ```sh
   % vi hello.txt
   ```

   按 `i` 插入，输入新内容，然后按 `Esc` 并输入 `:wq` 保存并退出。

3. 如果你安装了 `nano`：

   ```sh
   % nano hello.txt
   ```

   注意底部行显示如 `^O` 保存和 `^X` 退出的命令。

##### **常见初学者陷阱：被困在 `vi` 中**

几乎每个 UNIX 初学者都遇到过：你用 `vi` 打开文件，开始按键，一切都不按你预期发生。更糟的是，你无法弄清楚如何退出。

以下是发生了什么：

- `vi` 启动在**命令模式**，而不是输入模式。
- 要插入文本，按 **i**（插入）。
- 要返回命令模式，按 **Esc**。
- 要保存并退出：输入 `:wq` 并按 Enter。
- 要不保存退出：输入 `:q!` 并按 Enter。

**提示**：如果你意外打开 `vi` 只想逃离，按 **Esc**，输入 `:q!`，然后按 Enter。这会不保存退出。

### 提示和快捷键

一旦你习惯输入命令，你会很快发现 shell 有许多内置功能可以节省时间、减少错误。早期学习这些会让你更快感觉自在。

**关于 FreeBSD shell 的说明：**

- 新用户的**默认登录 shell** 通常是 **`/bin/tcsh`**，它支持 Tab 补全、箭头键导航历史和许多交互快捷键。
- 更精简的 **`/bin/sh`** shell 非常适合脚本和系统使用，但开箱即用不提供如 Tab 补全或箭头键历史等功能。
- 所以下面的一些快捷键如果不起作用，检查你使用的是哪个 shell（`echo $SHELL`）。

#### Tab 补全 (tcsh)

开始输入命令或文件名然后按 `Tab`。shell 会尝试为你补全。

```sh
% cd /et<Tab>
```

变为：

```sh
% cd /etc/
```

如果有多个匹配，按两次 `Tab` 查看可能性列表。
此功能在 `/bin/sh` 中不可用。

#### 命令历史 (tcsh)

按**上箭头**调出上一个命令，继续按向上走更远的历史。按**下箭头**向前移动。

```sh
% sysctl kern.hostname
```

你不必重新输入它，只需按上箭头然后按 Enter。
在 `/bin/sh` 中，你没有箭头键导航（但你仍然可以用 `!!` 再次运行命令）。

#### 通配符（globbing）

在*所有* shell 中工作，包括 `/bin/sh`。

```sh
% ls *.conf
```

列出所有以 `host` 开头并以 `.conf` 结尾的文件。

```sh
% ls host?.conf
```

匹配如 `host1.conf`、`hostA.conf` 的文件，但不匹配 `host.conf`。

#### 命令行编辑 (tcsh)

在 `tcsh` 中你可以用箭头键左右移动光标，或使用快捷键：

- **Ctrl+A** → 移动到行首。
- **Ctrl+E** → 移动到行尾。
- **Ctrl+U** → 删除光标到行首的所有内容。

- **快速重复命令（所有 shell）**

  ```sh
  % !!
  ```

  重新执行你的上一个命令。

  ```sh
  % !ls
  ```

  重复上一个以 `ls` 开头的命令。

**提示**：如果你想要更友好的交互 shell，坚持使用 **`/bin/tcsh`**（FreeBSD 用户默认）。如果你后来想要高级自定义，可以从包或 ports 安装如 `bash` 或 `zsh` 这样的 shell。但对于脚本编写，总是使用 **`/bin/sh`**，因为它保证存在且是系统的标准。

### 实践实验：导航和管理文件

让我们练习：

1. 进入你的主目录：

   ```sh
   % cd ~
   ```

2. 创建新目录：

   ```sh
   % mkdir unix_lab
   % cd unix_lab
   ```

3. 创建新文件：

   ```sh
   % echo "Hello FreeBSD" > hello.txt
   ```

4. 查看文件：

   ```sh
   % cat hello.txt
   ```

5. 制作副本：

   ```sh
   % cp hello.txt copy.txt
   % ls
   ```

6. 重命名：

   ```sh
   % mv copy.txt renamed.txt
   ```

7. 删除重命名的文件：

   ```sh
   % rm renamed.txt
   ```

通过完成这些步骤，你刚刚在文件系统中导航、创建文件、复制、重命名和删除——UNIX 日常工作的基本技能。

### 小结

shell 是你**通向 FreeBSD 的门户**。与系统的每一次交互——无论是运行命令、编译代码还是测试驱动程序——都通过它。在本节中，你学到了什么是 shell、命令如何构建，以及如何进行基本导航和文件管理。

接下来，我们将探索 **FreeBSD 如何组织其文件系统**。理解 `/etc`、`/usr` 和 `/dev` 等目录的布局将给你一个系统的心理地图，这在我们开始处理位于 `/dev` 下的设备驱动程序时特别重要。

## FreeBSD 文件系统布局

在 Windows 中，你可能习惯了像 `C:\` 和 `D:\` 这样的驱动器。在 UNIX 和 FreeBSD 中，没有驱动器字母。相反，一切都在一个以根 `/` 开始的**单一目录树**中。

这被称为**层次文件系统**。最顶层是 `/`，其他一切都像文件夹套文件夹一样在它下面分支出来。设备、配置文件和用户数据都组织在这棵树中。

这是一个简化地图：

```text
/
├── bin       → 基本用户命令 (ls, cp, mv)
├── sbin      → 系统管理命令 (ifconfig, shutdown)
├── etc       → 配置文件
├── usr
│   ├── bin   → 非基本用户命令
│   ├── sbin  → 非基本系统管理工具
│   ├── local → pkg 或 ports 安装的软件
│   └── src   → FreeBSD 源代码
├── var       → 日志、邮件、队列、临时运行时数据
├── home      → 用户主目录
├── dev       → 设备文件
└── boot      → 内核和引导加载程序
```

这里有一个表格，列出你将使用的一些最重要的目录：

| 目录         | 用途                                         |
| ------------ | -------------------------------------------- |
| `/`          | 整个系统的根。一切都从这里开始。              |
| `/bin`       | 基本命令行工具（早期引导时使用）。            |
| `/sbin`      | 系统二进制文件（如 `init`、`ifconfig`）。     |
| `/usr/bin`   | 用户命令行工具和程序。                        |
| `/usr/sbin`  | 管理员使用的系统级工具。                      |
| `/usr/src`   | FreeBSD 源代码（内核、库、驱动程序）。        |
| `/usr/local` | 包和已安装软件的位置。                        |
| `/boot`      | 内核和引导加载程序文件。                      |
| `/dev`       | 设备节点，代表设备的文件。                    |
| `/etc`       | 系统配置文件。                                |
| `/home`      | 用户主目录（如 `/home/dev`）。                |
| `/var`       | 日志文件、邮件队列、运行时文件。               |
| `/tmp`       | 临时文件，重启时清除。                        |

理解这个布局对驱动程序开发者至关重要，因为某些目录——特别是 `/dev`、`/boot` 和 `/usr/src`——直接与内核和驱动程序相关。但即使在这些之外，知道各种东西存放在哪里有助于你自信地导航。

**基础系统 vs 本地软件**：FreeBSD 的一个关键理念是基础系统和用户安装软件的分离。基础系统：内核、库和基本工具位于 `/bin`、`/sbin`、`/usr/bin` 和 `/usr/sbin`。你稍后用 pkg 或 ports 安装的一切都在 `/usr/local`。这种分离保持你的核心操作系统稳定，同时让你自由地添加和更新软件。

### 设备即文件：`/dev`

UNIX 的核心理念之一是**设备以文件形式出现**在 `/dev` 下。

例如：

- `/dev/null`：一个"黑洞"，丢弃你写入的任何内容。
- `/dev/zero`：输出无限零字节流。
- `/dev/random`：提供随机数据。
- `/dev/ada0`：你的第一个 SATA 磁盘。
- `/dev/da0`：USB 存储设备。
- `/dev/tty`：你的终端。

你可以使用与文件相同的工具与这些设备交互：

```sh
% echo "test" > /dev/null
% head -c 10 /dev/zero | hexdump
```

本书后面，当你创建驱动程序时，它会在这里暴露一个文件——例如 `/dev/hello`。写入该文件将实际运行你的内核代码。

### 绝对路径 vs. 相对路径

导航文件系统时，路径可以是：

- **绝对路径** - 从根 `/` 开始。例如：`/etc/rc.conf`
- **相对路径** - 从你当前位置开始。例如：`../notes.txt`

例如：

```sh
% cd /etc      # 绝对路径
% cd ..        # 相对路径：向上移动一级目录
```

**记住**：`/` 总是表示系统的根，而 `.` 表示"这里"，`..` 表示"上一级"。

#### 示例：使用绝对路径 vs 相对路径导航

假设你的主目录包含这个结构：

```text
/home/dev/unix_lab/
├── docs/
│   └── notes.txt
├── code/
│   └── test.c
└── tmp/
```

- 用**绝对路径**打开 `notes.txt`：

  ```sh
  % cat /home/dev/unix_lab/docs/notes.txt
  ```

- 从 `/home/dev/unix_lab` 内部用**相对路径**打开它：

  ```sh
  % cd /home/dev/unix_lab
  % cat docs/notes.txt
  ```

- 或者，如果你已经在 `docs` 目录中：

  ```sh
  % cd /home/dev/unix_lab/docs
  % cat ./notes.txt
  ```

绝对路径无论你在哪里都有效，而相对路径取决于你当前目录。作为开发者，你通常会在脚本中偏好绝对路径（更可预测），在交互工作时偏好相对路径（输入更快）。

### 实践实验：探索文件系统

让我们练习探索 FreeBSD 的布局：

1. 打印你当前的位置：

```sh
   % pwd
```

2. 进入根目录并列出其内容：

   ```sh
   % cd /
   % ls -lh
   ```

3. 查看 `/etc` 目录：

   ```sh
   % ls /etc
   % head -n 5 /etc/rc.conf
   ```

4. 探索 `/var/log` 并查看系统日志：

   ```sh
   % ls /var/log
   % tail -n 10 /var/log/messages
   ```

5. 检查 `/dev` 下的设备：

   ```sh
   % ls /dev | head
   ```

这个实验给你 FreeBSD 文件系统的"心理地图"，展示配置文件、日志和设备如何都组织在可预测的位置。

### 小结

在本节中，你学到 FreeBSD 使用从 `/` 开始的**单一层次文件系统**，有关键目录专门用于系统二进制文件、配置、日志、用户数据和设备。你还看到 `/dev` 如何将设备视为文件，这是你在编写驱动程序时将依赖的核心理念。

但文件和目录不只是关于结构；它们还关于**谁可以访问它们**。UNIX 是多用户系统，每个文件都有所有者、组和权限位，控制可以用它做什么。在下一节，我们将探索**用户、组和权限**，你将学习 FreeBSD 如何保持系统既安全又灵活。

## 用户、组和权限

UNIX 与早期 Windows 等系统最大的区别之一是 UNIX 从一开始就被设计为**多用户操作系统**。这意味着它假设多个人（或服务）可以同时使用同一台机器，并强制执行关于谁可以做什么的规则。

这种设计对安全性、稳定性和协作至关重要，作为驱动程序开发者，你需要很好地理解它，因为权限经常控制谁可以访问你的驱动程序设备文件。

### 用户和组

每个使用 FreeBSD 的人或服务都在一个**用户账户**下进行。

- 一个**用户**有一个用户名、一个数字 ID (UID) 和一个主目录。
- 一个**组**是用户的集合，由组名和组 ID (GID) 标识。

每个用户至少属于一个组，权限可以同时应用于个人和组。

你可以用以下命令查看你当前的身份：

   ```sh
% whoami
% id
   ```

示例输出：

```text
dev
uid=1001(dev) gid=1001(dev) groups=1001(dev), 0(wheel)
```

这里：

- 你的用户名是 `dev`。
- 你的 UID 是 `1001`。
- 你的主组是 `dev`。
- 你还属于 `wheel` 组，这允许访问管理权限（通过 `su` 或 `sudo`）。

### 文件所有权

在 FreeBSD 中，每个文件和目录都有一个**所有者**（一个用户）和一个**组**。

让我们用 `ls -l` 检查：

```sh
% ls -l hello.txt
```

输出：

```text
-rw-r--r--  1 dev  dev  12 Aug 23 10:15 hello.txt
```

分解：

- `-rw-r--r--` = 权限（我们稍后讲解）。
- `1` = 链接数（目前不重要）。
- `dev` = 所有者（创建文件的用户）。
- `dev` = 组（与文件关联的组）。
- `12` = 文件大小（字节）。
- `Aug 23 10:15` = 最后修改时间。
- `hello.txt` = 文件名。

所以这个文件属于用户 `dev` 和组 `dev`。

### 权限

权限控制用户可以对文件和目录做什么。有三类用户：

1. **所有者** - 拥有文件的用户。
2. **组** - 文件所属组的成员。
3. **其他人** - 其他所有人。

三种权限位：

- **r** = 读取（可以查看内容）。
- **w** = 写入（可以修改或删除）。
- **x** = 执行（对于程序，或在目录中，进入的能力）。

例如：

```text
-rw-r--r--
```

这意味着：

- **所有者** = 读 + 写。
- **组** = 只读。
- **其他人** = 只读。

所以所有者可以修改文件，但其他所有人只能查看它。

### 更改权限

要修改权限，你使用 **chmod** 命令。

两种方式：

**符号模式**

```sh
% chmod u+x script.sh
```

这为用户（`u`）添加执行权限（`+x`）。

**数字模式**

```sh
% chmod 750 script.sh
```

这里，数字代表权限：

- 7 = rwx
- 5 = r-x
- 0 = ---

所以 `750` 意思是：所有者 = rwx，组 = r-x，其他人 = ---。

### 更改所有权

有时你需要更改谁拥有文件。使用 `chown`：

   ```sh
% chown root:wheel hello.txt
   ```

现在文件由 root 所有，组为 wheel。

**注意**：更改所有权通常需要管理员权限。

### 实际场景：项目目录

假设你正在与队友一起做一个项目，你们都需要访问相同的文件。

以下是设置方法，以 root 身份运行这些命令：

1. 创建一个名为 `proj` 的组：

		```
   # pw groupadd proj
		```

2. 将两个用户添加到组：

   ```
   # pw groupmod proj -m dev,teammate
   ```

3. 创建目录并将其分配给该组：

   ```
   # mkdir /home/projdir
   # sudo chown dev:proj /home/projdir
   ```

4. 设置组权限，使成员可以写入：

   ```
   # chmod 770 /home/projdir
   ```

现在两个用户都可以在 `/home/projdir` 中工作，而其他人无法访问。

这正是 UNIX 系统安全执行协作的方式。

### 实践实验：权限实践

让我们练习：

1. 创建新文件：

   ```sh
   % echo "secret" > secret.txt
   ```

2. 检查其默认权限：

   ```sh
   % ls -l secret.txt
   ```

3. 移除其他人的读取权限：

   ```sh
   % chmod o-r secret.txt
   % ls -l secret.txt
   ```

4. 为用户添加执行权限：

   ```sh
   % chmod u+x secret.txt
   % ls -l secret.txt
   ```

5. 尝试更改所有权（需要 root）：

   ```
   % sudo chown root secret.txt
   % ls -l secret.txt
   ```

注意 `sudo` 会要求输入密码才能执行上面的步骤 5 中的 `chown` 命令。

使用这些命令，你已经在非常细粒度的层面上控制了文件访问——这个概念在我们创建驱动程序时直接适用，因为驱动程序也有 `/dev` 下的设备文件，具有所有权和权限规则。

### 小结

在本节中，你学到 FreeBSD 是一个**多用户系统**，每个文件都有所有者、组和控制访问的权限位。你看到如何检查和更改权限、如何管理所有权，以及如何安全地用组设置协作。

这些规则可能看起来简单，但它们是 FreeBSD 安全模型的骨干。稍后，当你编写驱动程序时，你在 `/dev` 下的设备文件也会有所有权和权限，控制谁可以打开和使用它们。

接下来，我们将看看**进程**，使系统活跃的运行程序。你将学习如何查看正在运行什么、如何管理进程，以及 FreeBSD 如何在幕后组织一切。

## 进程和系统监控

到目前为止，你学会了如何导航文件系统和管理文件。但操作系统不只是磁盘上的文件；它是**在内存中运行的程序**。这些运行的程序叫做**进程**，理解它们对于日常使用和驱动程序开发都至关重要。

### 什么是进程？

进程是运动中的程序。当你运行像 `ls` 这样的命令时，FreeBSD 会：

1. 将程序加载到内存。
2. 分配一个**进程 ID (PID)**。
3. 给它如 CPU 时间和内存等资源。
4. 跟踪它直到完成或停止。

进程是 FreeBSD 管理系统上发生的一切的方式。从你输入的 shell，到后台的守护进程，再到你的网络浏览器，它们都是进程。

**对驱动程序开发者**：当你编写驱动程序时，**用户空间的进程将与它对话**。了解进程如何创建和管理有助于你理解驱动程序如何被使用。

### 前台 vs. 后台进程

通常，当你运行命令时，它在**前台**运行，这意味着在那个终端中你不能做其他任何事情，直到它完成。

例如：

   ```sh
% sleep 10
   ```

这个命令暂停 10 秒。在这段时间里，你的终端"被阻塞"。

要在**后台**运行进程，在末尾添加 `&`：

```sh
% sleep 10 &
```

现在你立即拿回提示符，进程在后台运行。

你可以用以下命令查看后台作业：

```sh
% jobs
```

并把其中一个带回前台：

```sh
% fg %1
```

（其中 `%1` 是你用 `jobs` 看到的列表中的作业号）。

### 查看进程

要查看哪些进程正在运行，使用 `ps`：

```console
ps aux
```

示例输出：

```text
USER   PID  %CPU %MEM  VSZ   RSS  TT  STAT STARTED    TIME COMMAND
root     1   0.0  0.0  1328   640  -  Is   10:00AM  0:00.01 /sbin/init
dev   1024   0.0  0.1  4220  2012  -  S    10:05AM  0:00.02 -tcsh
dev   1055   0.0  0.0  1500   800  -  R    10:06AM  0:00.00 ps aux
```

这里：

- `PID` = 进程 ID。
- `USER` = 谁启动它。
- `%CPU` / `%MEM` = 正在使用的资源。
- `COMMAND` = 正在运行的程序。

#### 使用 `top` 观察进程和系统负载

虽然 `ps` 给你单一时刻的进程快照，有时你想要系统正在发生什么的**实时视图**。这就是 `top` 命令的作用。

```sh
% top
```

这打开一个持续更新的系统活动显示。默认情况下，每 2 秒刷新一次。要退出，按 **q**。

`top` 屏幕显示：

- **负载均值**（系统有多忙，取 1、5 和 15 分钟平均值）。
- **运行时间**（系统运行了多久）。
- **CPU 使用率**（用户、系统、空闲）。
- **内存和交换区使用率**。
- **进程列表**，按 CPU 使用率排序，所以你可以看到哪些程序工作最辛苦。

**`top` 输出示例（简化）：**  .

```text
last pid:  3124;  load averages:  0.06,  0.12,  0.14                                            up 0+20:43:11  11:45:09
17 processes:  1 running, 16 sleeping
CPU:  0.0% user,  0.0% nice,  0.0% system,  0.0% interrupt,  100% idle
Mem: 5480K Active, 1303M Inact, 290M Wired, 83M Buf, 387M Free
Swap: 1638M Total, 1638M Free

  PID USERNAME    THR PRI NICE   SIZE    RES STATE    C   TIME    WCPU COMMAND
 3124 dev           1  20    0    15M  3440K CPU3     3   0:00   0.03% top
 2780 dev           1  20    0    23M    11M select   0   0:00   0.01% sshd-session
  639 root          1  20    0    14M  2732K select   2   0:02   0.00% syslogd
  435 root          1  20    0    15M  4012K select   2   0:04   0.00% devd
  730 root          1  20    0    14M  2612K nanslp   0   0:00   0.00% cron
  697 root          2  20    0    18M  4388K select   3   0:00   0.00% qemu-ga
 2778 root          1  20    0    23M    11M select   1   0:00   0.00% sshd-session
  726 root          1  20    0    23M  9164K select   3   0:00   0.00% sshd
  760 root          1  68    0    14M  2272K ttyin    1   0:00   0.00% getty
```

这里我们可以看到：

- 系统已运行超过一天。
- 负载均值非常低（系统空闲）。
- CPU 大部分空闲。
- 内存大部分空闲。
- `yes` 命令（一个只是无限输出"y"的测试程序）几乎使用了所有 CPU。

##### 用 `uptime` 快速检查

如果你不需要 `top` 的完整细节，你可以使用：

```console
% uptime
```

显示类似：

```text
 3:45PM  up 2 days,  4:11,  2 users,  load averages:  0.32,  0.28,  0.25
```

这告诉你：

- 当前时间。
- 系统运行了多久。
- 多少用户登录。
- 负载均值（1、5、15 分钟）。

**提示**：负载均值是快速查看系统是否过载的方法。在单 CPU 系统上，负载均值 `1.00` 意味着 CPU 完全忙碌。在 4 核系统上，`4.00` 意味着所有核心都满载。

**实践实验：观察系统**

1. 运行 `uptime` 并记录系统的负载均值。

2. 在 FreeBSD 机器上打开两个终端。

3. 在第一个终端，启动一个忙碌的进程：

   ```sh
   % yes > /dev/null &
   ```

4. 在第二个终端运行 `top` 查看 `yes` 进程使用了多少 CPU。

5. 用 `kill %1` 或 `pkill yes` 停止 `yes` 命令，或者在第一个终端按 `ctrl+c`

6. 再次运行 `uptime`，注意负载均值比之前稍高，但会随时间降下来。

### 停止进程

有时进程行为异常或需要停止。你可以使用：

- **kill** - 向进程发送信号。

	```sh
		% kill 1055
		```

  （将 1055 替换为实际 PID）。

- **kill -9** - 强制进程立即终止。

  ```sh
  % kill -9 1055
  ```

仅在必要时使用 `kill -9`，因为它不给程序清理的机会。

当你使用 `kill` 时，你不是真的*"杀死"*进程；你在向它发送一个**信号**。信号是内核传递给进程的消息。

- 默认情况下，`kill` 发送 **SIGTERM（信号 15）**，礼貌地请求进程终止。行为良好的程序会清理并退出。
- 如果进程拒绝，你可以用 `kill -9 PID` 发送 **SIGKILL（信号 9）**。这强制进程立即停止，不进行清理。
- 另一个有用的是 **SIGHUP（信号 1）**，通常用于告诉守护进程（后台服务）重新加载配置。

尝试：

  ```sh
% sleep 100 &
% ps aux | grep sleep
% kill -15 <PID>   # 先尝试 SIGTERM
% kill -9 <PID>    # 如果仍在运行，使用 SIGKILL
  ```

作为未来的驱动程序开发者，这种区别很重要。你的代码可能需要优雅地处理终止，清理资源而不是让内核处于不稳定状态。

#### 进程层次：父进程和子进程

FreeBSD 中的每个进程（通常在 UNIX 系统中）都有一个**父进程**启动了它。例如，当你在 shell 中输入命令时，shell 进程是父进程，你运行的命令成为其子进程。

你可以使用带有自定义列的 `ps` 查看这种关系：

```sh
% ps -o pid,ppid,command | head -10
```

输出示例（简化）：

```yaml
  PID  PPID COMMAND
    1     0 /sbin/init
  534     1 /usr/sbin/cron
  720   534 /bin/sh
  721   720 sleep 100
```

这里你可以看到：

- 进程 **1** 是 `init`，所有进程的祖先。
- `cron` 由 `init` 启动。
- 一个 `sh` shell 进程由 `cron` 启动。
- `sleep 100` 进程由 shell 启动。

理解进程层次对调试很重要：如果父进程死亡，其子进程可能被**init 收养**。稍后，当你处理驱动程序时，你会看到系统守护进程和服务如何创建和管理与你的代码交互的子进程。

### 监控系统资源

FreeBSD 提供简单的命令检查系统健康：

- **df -h** - 显示磁盘使用情况。

	```sh
		% df -h
		```

  示例：

  ```yaml
  Filesystem  Size  Used  Avail Capacity  Mounted on
  /dev/ada0p2  50G   20G    28G    42%    /
  ```

- **du -sh** - 显示目录大小。

  ```
  % du -sh /var/log
  ```

- **freebsd-version** - 显示操作系统版本。

  ```
  % freebsd-version
  ```

- **sysctl** - 查询系统信息。

  ```sh
  % sysctl hw.model
  % sysctl hw.ncpu
  ```

输出可能显示你的 CPU 型号和核心数。

稍后，在编写驱动程序时，你经常使用 `dmesg` 和 `sysctl` 监控驱动程序如何与系统交互。

### 实践实验：处理进程

让我们练习：

1. 在后台运行一个 sleep 命令：

      ```sh
      % sleep 30 &
      ```

2. 检查运行中的作业：

   ```sh
   % jobs
   ```

3. 列出进程：

   ```sh
   % ps aux | grep sleep
   ```

4. 停止进程：

   ```sh
   % kill <PID>
   ```

5. 运行 `top` 并观察系统活动。按 `q` 退出。

6. 检查系统信息：

   ```sh
   % sysctl hw.model
   % sysctl hw.ncpu
   ```

### 小结

在本节中，你学到进程是 FreeBSD 中活动的、正在运行的程序。你看到如何启动它们、在前后台之间移动、用 `ps` 和 `top` 检查它们、用 `kill` 停止它们。你还探索了检查磁盘、CPU 和内存使用情况的基本系统监控命令。

进程是必不可少的，因为它们使系统活跃起来，作为驱动程序开发者，使用你驱动程序的程序总是作为进程运行。

但监控进程只是故事的一部分。要做真正的工作，你需要比基础系统包含的更多工具。FreeBSD 提供了一种干净灵活的方式来安装和管理额外软件，从像 `nano` 这样的简单工具到像网络服务器这样的大型应用程序。在下一节，我们将看看**FreeBSD 包系统和 Ports 集合**，这样你可以用需要的软件扩展系统。

## 安装和管理软件

FreeBSD 被设计为一个精简可靠的操作系统。开箱即用，你得到一个坚如磐石的**基础系统**——内核、系统库、基本工具和配置文件。这之外的一切——编辑器、编译器、服务器、监控工具，甚至桌面环境——都被视为**第三方软件**，FreeBSD 提供了两种优秀的方式来安装它：

1. **pkg** - 二进制包管理器：快速、简单、方便。
2. **Ports 集合** - 大规模基于源的构建系统，允许精细调整。

它们共同为 FreeBSD 提供了 UNIX 世界中最灵活的软件生态系统之一。

### 用 pkg 安装二进制包

`pkg` 工具是 FreeBSD 的现代包管理器。它让你可以访问由 FreeBSD ports 团队维护的**数万个预构建应用程序**。

当你用 `pkg` 安装包时，发生以下情况：

- 工具从 FreeBSD 镜像获取**二进制包**。
- 依赖项自动下载。
- 文件安装到 `/usr/local` 下。
- 包数据库跟踪安装了什么，以便你稍后可以更新或删除它。

#### 常用命令

- 更新包仓库：

   ```sh
  % sudo pkg update
  ```

- 搜索软件：

  ```sh
  % sudo pkg search htop
  ```

- 安装软件：

  ```sh
  % sudo pkg install htop
  ```

- 升级所有包：

  ```sh
  % sudo pkg upgrade
  ```

- 删除软件：

  ```sh
  % sudo pkg delete htop
  ```

对于初学者，`pkg` 是最快、最安全的软件安装方式。

### FreeBSD Ports 集合

**Ports 集合**是 FreeBSD 的皇冠上的明珠之一。它是位于 `/usr/ports` 下的一个**巨大的构建配方树**（称为"ports"）。每个 port 包含：

- 一个 **Makefile**，描述如何获取、打补丁、配置和构建软件。
- 用于验证完整性的校验和。
- 关于依赖项和许可的元数据。

当你从 ports 构建软件时，FreeBSD 从原始项目站点下载源代码，应用 FreeBSD 特定的补丁，并在你的系统上本地编译。

#### 为什么使用 Ports？

那么，既然有预构建的包，为什么还要从源代码构建？

- **定制** - 许多应用程序有可选功能。使用 ports，你可以在编译期间选择启用或禁用什么。
- **优化** - 高级用户可能想要针对他们的硬件调整编译标志。
- **前沿选项** - 有时新功能在进入二进制包之前先在 ports 中可用。
- **与 pkg 一致** - Ports 和包共享相同的基础设施。实际上，包是由 FreeBSD 构建集群从 ports 构建的。

#### 获取和探索 Ports 树

Ports 集合位于 `/usr/ports` 下，但在全新的 FreeBSD 系统上此目录可能还不存在。让我们检查：

```sh
% ls /usr/ports
```

如果你看到诸如 `archivers`、`editors`、`net`、`security`、`sysutils` 和 `www` 这样的类别，那么 Ports 已安装。如果目录缺失，你需要自己获取 Ports 树。

#### 用 Git 安装 Ports 集合

官方推荐的方式是使用 **Git**：

1. 确保 `git` 已安装：

   ```sh
   % sudo pkg install git
   ```

2. 克隆官方 Ports 仓库：

   ```sh
   % sudo git clone https://git.FreeBSD.org/ports.git /usr/ports
   ```

   这将创建 `/usr/ports` 并用整个 Ports 集合填充它。初始克隆可能需要一些时间，因为它包含数千个应用程序。

3. 稍后更新 ports 树，只需运行：

   ```sh
   % cd /usr/ports
   % sudo git pull
   ```

还有一个叫 `portsnap` 的旧工具，但 **Git 是现代推荐的方法**，因为它保持你的树直接与 FreeBSD 项目的仓库同步。

#### 浏览 Ports

安装 Ports 后，探索它：

```sh
% cd /usr/ports
% ls
```

你会看到文件和类别如：

```text
CHANGES         UIDs            comms           ftp             mail            portuguese      x11
CONTRIBUTING.md UPDATING        converters      games           math            print           x11-clocks
COPYRIGHT       accessibility   databases       german          misc            russian         x11-drivers
GIDs            arabic          deskutils       graphics        multimedia      science         x11-fm
Keywords        archivers       devel           hebrew          net             security        x11-fonts
MOVED           astro           dns             hungarian       net-im          shells          x11-servers
Makefile        audio           editors         irc             net-mgmt        sysutils        x11-themes
Mk              benchmarks      emulators       japanese        net-p2p         textproc        x11-toolkits
README          biology         filesystems     java            news            ukrainian       x11-wm
Templates       cad             finance         korean          polish          vietnamese
Tools           chinese         french          lang            ports-mgmt      www
```

每个类别都有特定应用程序的子目录。例如：

```sh
% cd /usr/ports/sysutils/memdump
% ls
```

这里你会找到像 `Makefile`、`distinfo`、`pkg-descr` 这样的文件，可能还有一个 `files/` 目录。这些是 FreeBSD 用来构建应用程序的"原料"：`Makefile` 定义过程，`distinfo` 确保完整性，`pkg-descr` 描述这个软件做什么，`files/` 包含任何 FreeBSD 特定的补丁。

#### 从 Ports 构建

例如：从 ports 安装 `memdump`。

```sh
% cd /usr/ports/sysutils/memdump
% sudo make install clean
```

在构建过程中，你可能会看到一个选项菜单，如启用传感器或颜色、安装文档等。这就是 ports 的优势所在——你控制编译哪些功能。

`make install clean` 过程做三件事：

- **install** - 构建并安装程序。
- **clean** - 删除临时构建文件。

#### 混合 Ports 和 Packages

一个常见问题：*我可以混合使用包和 ports 吗？*

可以，它们是兼容的，因为两者都是从同一个源代码树构建的。但是，如果你用自定义选项从 ports 重新构建某样东西，你应该小心，不要稍后意外地被二进制包更新覆盖。

许多用户用 `pkg` 安装大多数东西，但对于需要定制的特定应用程序使用 ports。

### 已安装软件的位置

`pkg` 和 ports 都将第三方软件安装到 `/usr/local` 下。这使它们与基础系统分离。

典型位置：

- **二进制文件** → `/usr/local/bin`
- **库** → `/usr/local/lib`
- **配置** → `/usr/local/etc`
- **手册页** → `/usr/local/man`

尝试：

```sh
% which nano
```

输出：

```text
/usr/local/bin/nano
```

这确认 nano 来自包/ports，而不是基础系统。

### 实际示例：安装 vim 和 htop

让我们尝试两种方法。

#### 使用 pkg

```sh
% sudo pkg install vim htop
```

运行它们：

```sh
% vim test.txt
% htop
```

#### 使用 Ports

```sh
% cd /usr/ports/sysutils/htop
% sudo make install clean
```

运行它：

```sh
% htop
```

注意 ports 版本可能会在构建期间询问可选功能，而 pkg 用默认设置安装。

### 实践实验：管理软件

1. 更新你的包仓库：

	```sh
		% sudo pkg update
		```

2. 用 pkg 安装 lynx：

   ```sh
   % sudo pkg install lynx
   % lynx https://www.freebsd.org
   ```

3. 搜索 bsdinfo：

   ```sh
   % pkg search bsdinfo
   ```

4. 从 ports 安装 bsdinfo：

   ```sh
   % cd /usr/ports/sysutils/bsdinfo
   % sudo make install clean
   ```

5. 运行 bsdinfo 确认它现已安装：

   ```sh
   % bsdinfo
   ```

6. 删除 nano：

   ```sh
   % sudo pkg delete nano
   ```

你现在已经用 pkg 和 ports 安装、运行和删除了软件——这两种互补的方法赋予了 FreeBSD 灵活性。

### 小结

在本节中，你学到 FreeBSD 如何处理第三方软件：

- **pkg 系统**给你快速、简单的二进制安装。
- **Ports 集合**提供基于源的灵活性和定制。
- 两种方法都安装到 `/usr/local` 下，保持基础系统分离和干净。

理解这个生态系统是 FreeBSD 文化的重要部分。许多管理员用 `pkg` 安装常用工具，在需要细粒度控制时转向 ports。作为开发者，你会欣赏这两种方法：pkg 为了便利，ports 当你想看看软件究竟如何构建和集成时。

但应用程序只是故事的一部分。FreeBSD **基础系统**——内核和核心工具也需要定期更新以保持安全和可靠。在下一节，我们将学习如何使用 `freebsd-update` 保持操作系统本身最新，这样你总是有坚实的基础可以构建。

## 保持 FreeBSD 更新

作为 FreeBSD 用户你能养成的最重要习惯之一是保持系统更新。更新修复安全问题、消灭错误，有时添加对新硬件的支持。不像更新应用程序的命令 `pkg update && pkg upgrade`，**`freebsd-update` 命令用于更新基础操作系统本身**，包括内核和核心工具。

保持系统当前确保你安全运行 FreeBSD，并给你与其他开发者相同的坚实基础。

### 为什么更新很重要

- **安全性：** 像任何软件一样，FreeBSD 偶尔有安全漏洞。更新快速修补这些问题。
- **稳定性：** 错误修复提高可靠性，这对开发驱动程序至关重要。
- **兼容性：** 更新带来对新 CPU、芯片组和其他硬件的支持。

不要把更新视为可有可无。它们是负责任系统管理的一部分。

### `freebsd-update` 工具

FreeBSD 用 `freebsd-update` 工具使更新变得简单。它的工作方式是：

1. **获取**关于可用更新的信息。
2. **应用**二进制补丁到你的系统。
3. 如果需要，**重启**到更新的内核。

这比从源代码重建系统要容易得多（稍后当我们需要那种级别的控制时，我们会学习）。

### 更新工作流程

这是标准流程：

1. **获取可用更新**

   ```sh
   % sudo freebsd-update fetch
   ```

   这会联系 FreeBSD 更新服务器并下载你版本的任何安全补丁或错误修复。

2. **查看变更**
    获取后，`freebsd-update` 可能会显示一个将被修改的配置文件列表。
    例如：

   ```yaml
   The following files will be updated as part of updating to 14.1-RELEASE-p3:
   /bin/ls
   /sbin/init
   /etc/rc.conf
   ```

   不要慌！这并不意味着你的系统坏了——只是有些文件会被更新。

   - 如果像 `/etc/rc.conf` 这样的系统配置文件在基础系统中发生了变化，你会被要求查看差异。
   - `freebsd-update` 使用合并工具显示并排变更。
   - 对初学者：如果你不确定，通常**接受默认值（保留本地版本）**是安全的。你总是可以稍后阅读 `/var/db/freebsd-update` 日志。

**提示：** 如果此时你对合并配置文件感到不安，可以跳过变更，稍后手动检查。

3. **安装更新**

   ```sh
   % sudo freebsd-update install
   ```

   这一步应用已下载的更新。

   - 如果更新只包含用户空间程序（如 `ls`、`cp`、库），你就完成了。
   - 如果更新包含**内核补丁**，你将被要求在安装后**重启**。

### 示例会话

这是正常更新的样子：

```sh
% sudo freebsd-update fetch
Looking up update.FreeBSD.org mirrors... 3 mirrors found.
Fetching metadata signature for 14.3-RELEASE from update1.FreeBSD.org... done.
Fetching metadata index... done.
Fetching 1 patches..... done.
Applying patches... done.
The following files will be updated as part of updating to 14.3-RELEASE-p1:
    /bin/ls
    /bin/ps
    /sbin/init
% sudo freebsd-update install
Installing updates... done.
```

如果内核被更新了：

```sh
% sudo reboot
```

重启后，你的系统完全补丁完毕。

### 使用 `freebsd-update` 更新内核

`freebsd-update` 有用的功能之一是它可以更新内核本身。你不必手动重建，除非你想运行自定义内核（本书稍后会涵盖）。

这意味着对大多数用户来说，保持安全和当前只是定期运行 `fetch` + `install` 的问题。

### 用 `freebsd-update` 升级到新版本

除了应用安全和错误修复补丁，`freebsd-update` 还可以将你的系统升级到**新的 FreeBSD 版本**。例如，如果你运行 **FreeBSD 14.2** 并想升级到 **14.3**，过程很简单。

工作流程有三步：

1. **获取升级文件**

   ```sh
   % sudo freebsd-update upgrade -r 14.3-RELEASE
   ```

   将 `14.3-RELEASE` 替换为你想升级到的版本。

2. **安装新组件**

   ```sh
   % sudo freebsd-update install
   ```

   这安装更新的第一阶段。如果内核被更新，你将需要重启：

   ```sh
   % sudo reboot
   ```

3. **重复安装**
    重启后，再次运行安装步骤以完成更新系统的其余部分：

   ```sh
   % sudo freebsd-update install
   ```

最后，你将运行新版本。你可以用以下命令确认：

```sh
% freebsd-version
```

**提示**：版本升级有时可能涉及配置文件合并（就像安全更新）。如果有疑问，保留你的本地版本——你总是可以稍后与存储在 `/var/db/freebsd-update/` 下的新默认值比较。

记住，在版本升级后更新你的**包**也是好主意，因为它们是针对新系统库构建的：

```sh
% sudo pkg update
% sudo pkg upgrade
```

### 实践实验：运行你的第一次更新

1. 检查你当前的 FreeBSD 版本：

   ```sh
   % freebsd-version -kru
   ```

   - `-k` → 内核
   - `-r` → 正在运行
   - `-u` → 用户空间

2. 运行 `freebsd-update fetch` 看是否有更新可用。

3. 仔细阅读关于配置文件合并的任何消息。如果不确定，选择**保留你的版本**。

4. 运行 `freebsd-update install` 应用更新。

5. 如果内核被更新，重启：

   ```sh
   % sudo reboot
   ```

**常见初学者陷阱：害怕配置文件合并**

当 `freebsd-update` 要求你合并变更时，可能看起来很吓人——很多文本、加减符号和提示。别担心。

- 如果有疑问，保留 `/etc/rc.conf` 或 `/etc/hosts` 等文件的本地版本。
- 系统仍然可以工作。
- 你总是可以稍后检查新的默认文件（它们存储在 `/var/db/freebsd-update/`）。

随着时间的推移，你会习惯解决这些合并，但在开始时，**选择保留配置是安全的路径**。

### 小结

只用两个命令，`freebsd-update fetch` 和 `freebsd-update install`，你现在知道如何保持 FreeBSD 基础系统补丁和安全。这个过程只需要几分钟，但确保你的环境对开发工作是安全可靠的。

稍后，当我们开始处理内核并编写驱动程序时，我们还将学习如何从源代码构建和安装自定义内核。但现在，你已经有了像专业人士一样维护系统的基本知识。

既然检查更新是你可能想定期做的事情，如果系统可以自动为你处理一些这类杂务不是很好吗？这正是我们接下来要看的内容：使用如 `cron`、`at` 和 `periodic` 等工具进行**调度和自动化**。

## 调度和自动化

UNIX 的最大优势之一是它被设计为让计算机为你处理重复任务。你不必等到午夜才运行备份，也不必每天早上登录启动监控脚本，你可以告诉 FreeBSD：

> *"在这个时间为我运行这个命令，每天，永远。"*

这不仅节省时间，还使你的系统更可靠。在 FreeBSD 中，主要的工具是：

1. **cron** - 用于重复任务，如备份或监控。
2. **at** - 用于你想稍后调度的一次性任务。
3. **periodic** - FreeBSD 的内置例程维护任务系统。

### 为什么自动化任务？

自动化重要是因为它增强我们的：

- **一致性** - 用 cron 调度的任务总是会运行，即使你忘记。
- **效率** - 你不必手动重复命令，只需写一次。
- **可靠性** - 自动化有助于避免错误。计算机不会忘记在周日晚上轮转日志。
- **系统维护** - FreeBSD 本身严重依赖 cron 和 periodic 来保持系统健康（轮转日志、更新数据库、运行安全检查）。

### cron：自动化的主力

`cron` 守护进程在后台持续运行。每分钟，它检查调度的任务列表（存储在 crontabs 中）并运行那些匹配当前时间的任务。

每个用户都有自己的**crontab**，系统有一个全局的。这意味着你可以调度个人任务（如清理主目录中的文件）而不触及系统任务。

### 理解 crontab 格式

crontab 格式有**五个字段**描述*何时*运行任务，后跟命令本身：

   ```yaml
minute   hour   day   month   weekday   command
   ```

- **minute**: 0-59
- **hour**: 0-23（24 小时制）
- **day**: 1-31
- **month**: 1-12
- **weekday**: 0-6（0 = 星期日，6 = 星期六）

帮助记忆的口诀：*"My Hungry Dog Must Wait."*（Minute, Hour, Day, Month, Weekday）

#### cron 任务示例

- 每天午夜运行：

	```
		0 0 * * * /usr/bin/date >> /home/dev/midnight.log
		```

- 每 15 分钟运行：

  ```
  */15 * * * * /home/dev/scripts/check_disk.sh
  ```

- 每周一早上 8 点运行：

  ```
  0 8 * * 1 echo "Weekly meeting" >> /home/dev/reminder.txt
  ```

- 每月 1 号凌晨 3:30 运行：

  ```
  30 3 1 * * /usr/local/bin/backup.sh
  ```

### 编辑和管理 Crontab

要编辑你的个人 crontab：

  ```
crontab -e
  ```

这会在默认编辑器（`vi` 或 `ee`）中打开你的 crontab。

要列出你的任务：

```console
crontab -l
```

要删除你的 crontab：

```console
crontab -r
```

### 日志去哪里？

当 cron 运行任务时，其输出（stdout 和 stderr）通过**邮件**发送给拥有该任务的用户。在 FreeBSD 上，这些邮件本地投递并存储在 `/var/mail/username` 中。

你也可以将输出重定向到日志文件使事情更简单：

```text
0 0 * * * /home/dev/backup.sh >> /home/dev/backup.log 2>&1
```

这里：

- `>>` 将输出追加到 `backup.log`。
- `2>&1` 将错误消息（stderr）重定向到同一文件。

这样，你总是知道你的 cron 任务做了什么，即使你不检查系统邮件。

### at：一次性调度

有时你不想要重复任务，你只是想让某个任务稍后运行一次。这就是 **at** 的用途。

在用户可以使用 **at** 之前，超级用户必须先将用户名追加到文件 `/var/at/at.allow`。

```sh 
# echo "dev" >> /var/at/at.allow
```

现在用户可以执行 `at` 命令。用法很简单，让我们看几个例子：

- 10 分钟后运行命令：

```sh
% echo "echo Hello FreeBSD > /home/dev/hello.txt" | at now + 10 minutes
```

- 明天上午 9 点运行命令：

```sh
  % echo "/usr/local/bin/htop" | at 9am tomorrow
```

用 `at` 调度的任务会排队并只运行一次。你可以用 `atq` 列出它们，用 `atrm` 删除它们。

### periodic：FreeBSD 的维护助手

FreeBSD 自带一个内置的整理系统叫 **periodic**。它是一个 shell 脚本框架，为你处理例程维护任务，这样你不必手动记住它们。

这些任务自动以**每日、每周和每月间隔**运行，这归功于已经在系统级 cron 文件 `/etc/crontab` 中配置的条目。这意味着全新安装的 FreeBSD 系统已经为你处理了许多杂务，你不用动一根手指。

#### 脚本存放位置

脚本组织在 `/etc/periodic` 下的目录中：

```text
/etc/periodic/daily
/etc/periodic/weekly
/etc/periodic/monthly
/etc/periodic/security
```

- **daily/** - 每天运行的任务（日志轮转、安全检查、数据库更新）。
- **weekly/** - 每周运行一次的任务（如更新 locate 数据库）。
- **monthly/** - 每月运行一次的任务（如月度会计报告）。
- **security/** - 额外的专注于系统安全的检查。

#### periodic 默认做什么

开箱即用包含的任务示例：

- **安全检查** - 查找 setuid 二进制文件、不安全的文件权限或已知漏洞。
- **日志轮转** - 压缩并归档 `/var/log` 下的日志，使它们不会永远增长。
- **数据库更新** - 重建辅助数据库，如 `locate` 命令使用的数据库。
- **临时文件清理** - 删除 `/tmp` 和其他缓存目录中的残留物。

它们运行后，periodic 脚本通常将结果摘要发送到 **root 用户的邮箱**（以 root 身份运行 `mail` 来阅读）。

**常见初学者陷阱："什么都没发生！"**

许多新的 FreeBSD 用户运行系统几天，知道 periodic 应该每天运行任务，但他们从未看到任何输出，以为它不起作用。实际上，periodic 的报告被发送到 **root 用户的邮件**，而不是显示在屏幕上。

要阅读它们，以 root 登录并运行：

```console
# mail
```

按 Enter 打开邮箱查看报告。你可以输入 `q` 退出邮件程序。

**提示：** 如果你更喜欢在普通用户收件箱接收这些报告，你可以在 `/etc/aliases` 中配置邮件转发，使 root 的邮件重定向到你的用户账户。

#### 手动运行 periodic

你不必等 cron 触发它们。你可以手动运行整套任务：

```sh
% sudo periodic daily
% sudo periodic weekly
% sudo periodic monthly
```

或直接运行一个脚本，例如：

```sh
% sudo /etc/periodic/security/100.chksetuid
```

#### 用 `periodic.conf` 定制 periodic

Periodic 不是黑盒。其行为通过 `/etc/periodic.conf` 和 `/etc/periodic.conf.local` 控制。

**最佳实践**：永远不要直接编辑脚本。相反，在 `periodic.conf` 中覆盖它们的行为——这使你的更改在 FreeBSD 更新基础系统时保持安全。

以下是一些你可能使用的常见选项：

- **启用或禁用任务**

  ```
  daily_status_security_enable="YES"
  daily_status_network_enable="NO"
  ```

- **控制日志处理**

  ```
  daily_clean_hoststat_enable="YES"
  weekly_clean_pkg_enable="YES"
  ```

- **启用 locate 数据库更新**

  ```
  weekly_locate_enable="YES"
  ```

- **控制 tmp 清理**

  ```
  daily_clean_tmps_enable="YES"
  daily_clean_tmps_days="3"
  ```

- **安全报告**

  ```
  daily_status_security_inline="YES"
  daily_status_security_output="mail"
  ```

要查看所有可用选项，使用命令 `man periodic.conf`

#### 发现所有可用检查

现在你知道 periodic 运行每日、每周和每月任务，但你可能想知道：*这些检查到底都是什么，它们做什么？*

有几种方法可以探索它们：

1. **直接列出脚本**

   ```sh
   % ls /etc/periodic/daily
   % ls /etc/periodic/weekly
   % ls /etc/periodic/monthly
   % ls /etc/periodic/security
   ```

   你会看到如 `100.clean-disks` 或 `480.leapfile-ntpd` 这样的文件名——脚本名称是描述性的，会给你关于脚本做什么的概念。数字帮助控制它们运行的顺序。

2. **阅读文档**

   手册页 `periodic(8)` 和 `periodic.conf(5)` 解释了许多可用的脚本及其选项。例如：

   ```
   man periodic.conf
   ```

   给你配置变量及其控制的摘要。

3. **检查脚本头部**
    用 `less` 打开 `/etc/periodic/*/` 中的任何脚本，阅读前几行注释。它们通常包含关于脚本目的的人类可读解释。

这意味着你永远不必猜测 periodic 在做什么；你总是可以检查脚本、预览其行为或阅读官方文档。

#### 这对开发者为什么重要

对于日常用户，periodic 保持系统整洁安全，无需额外努力。但作为开发者，你稍后可能想：

- 添加**自定义 periodic 脚本**来每天测试你的驱动程序或监控其健康状况。
- 轮转或清理你的驱动程序创建的自定义日志文件。
- 运行自动完整性检查（例如，验证你的驱动程序设备节点存在并响应）。

通过接入 periodic，你构建在 FreeBSD 本身用于自己整理的同一框架上。

**实践实验：探索和定制 periodic**

1. 列出可用的每日脚本：

   ```sh
   % ls /etc/periodic/daily
   ```

2. 手动运行它们：

   ```sh
   % sudo periodic daily
   ```

3. 打开 `/etc/periodic.conf`（如果不存在则创建）并添加：

   ```sh
   weekly_locate_enable="YES"
   ```

4. 预览每周任务会做什么：

   ```sh
   % sudo periodic weekly
   ```

5. 触发每周任务然后尝试：

   ```sh
   % locate passwd
   ```

### 实践实验：自动化任务

1. 调度一个每分钟运行的测试任务：

```sh
   % crontab -e
   */1 * * * * echo "Hello from cron: $(date)" >> /home/dev/cron_test.log
```

2. 等几分钟检查文件：

   ```sh
   % tail -n 5 /home/dev/cron_test.log
   ```

3. 用 `at` 调度一次性任务：

   ```sh
   % echo "date >> /home/dev/at_test.log" | at now + 2 minutes
   ```

   稍后检查：

   ```sh
   % cat /home/dev/at_test.log
   ```

4. 手动运行一个 periodic 任务：

   ```sh
   % sudo periodic daily
   ```

   你会看到关于日志文件、安全和系统状态的报告。

### 初学者常见陷阱

- 忘记设置**完整路径**。cron 任务不使用与你的 shell 相同的环境，所以始终使用完整路径（`/usr/bin/ls` 而不只是 `ls`）。
- 忘记重定向输出。如果你不重定向，结果可能会被静默邮寄给你。
- 重叠任务。小心不要调度冲突或运行太频繁的任务。

### 这对驱动程序开发者为什么重要

你可能在想为什么我们要花时间在 cron 任务和调度任务上。答案是自动化是**开发者最好的朋友**。当你开始编写设备驱动程序时，你会经常想要：

- 调度你的驱动程序的自动测试（例如，每晚检查它是否干净地加载和卸载）。
- 轮转和归档内核日志以跟踪驱动程序随时间的行为。
- 运行定期诊断，与你的驱动程序的 `/dev` 节点交互并记录结果以供分析。

通过现在掌握 cron 和 periodic，你将稍后已经知道如何设置这些后台例程，节省时间并尽早捕获错误。

### 小结

在本节中，你学到 FreeBSD 如何使用三个主要工具自动化任务：

- **cron** 用于重复任务，
- **at** 用于一次性调度，
- **periodic** 用于内置系统维护。

你练习了创建任务、检查输出，并学习了 FreeBSD 本身如何依赖自动化保持健康。

自动化很有用，但有时你需要超越固定调度。你可能想要链接命令、使用循环或添加逻辑来决定发生什么。这就是 **shell 脚本**的作用。在下一节，我们将编写你的第一个脚本，看看如何创建适合你需求的自定义自动化。

## Shell 脚本简介

你已经学会逐个运行命令。Shell 脚本让你**将这些命令保存到可重用的程序中**。在 FreeBSD 上，脚本编写原生且推荐的 shell 是 **`/bin/sh`**。这个 shell 遵循 POSIX 标准，在每个 FreeBSD 系统上都可用。

> **给 Linux 用户的提示**
>  许多 Linux 发行版上，示例使用 **bash**。在 FreeBSD 上，**bash 不是基础系统的一部分**。你可以用 `pkg install bash` 安装它，它会位于 `/usr/local/bin/bash`。要在 FreeBSD 上编写可移植、无依赖的脚本，使用 `#!/bin/sh`。

我们将逐步构建本节：shebang 和执行、变量和引号、条件、循环、函数、处理文件、返回码和基本调试。下面的每个示例脚本都**有完整注释**，以便完全的初学者也能跟上。

### 1) 你的第一个脚本：shebang、使其可执行、运行它

创建一个名为 `hello.sh` 的文件：

```sh
#!/bin/sh
# hello.sh   使用 FreeBSD 原生 /bin/sh 的第一个 shell 脚本
# 打印友好消息，包含当前日期和活动用户。

# 'date' 打印当前日期和时间
# 'whoami' 打印当前用户
echo "Hello from FreeBSD!"
echo "Date: $(date)"
echo "User: $(whoami)"
```

**提示：`#!`（Shebang）是什么意思？**

这个脚本的第一行是：

```sh
#!/bin/sh
```

这叫 **shebang 行**。两个字符 `#!` 告诉系统*哪个程序应该解释脚本*。

- `#!/bin/sh` 意思是："用 **sh** shell 运行这个脚本。"
- 在其他系统上，你可能还会看到 `#!/bin/tcsh`、`#!/usr/bin/env python3` 或 `#!/usr/bin/env bash`。

当你使脚本可执行并运行它时，系统查看这一行决定使用哪个解释器。没有它，脚本可能失败或表现不同，取决于你的登录 shell。

**经验法则**：始终在脚本顶部包含 shebang 行。在 FreeBSD 上，`#!/bin/sh` 是最安全、最可移植的选择。

现在让脚本可执行并运行它：

```sh
% chmod +x hello.sh       # 给用户执行权限
% ./hello.sh              # 从当前目录运行它
```

如果你得到"Permission denied"，你忘了 `chmod +x`。
如果你得到"Command not found"，你可能输入了 `hello.sh` 而没有 `./`，当前目录不包含在系统 `PATH` 中。

**提示**：不要感到必须立即掌握所有脚本功能。从小处开始，写一个打印你的用户名和日期的 2-3 行脚本。一旦你舒服了，添加条件（`if`），然后循环，然后函数。Shell 脚本就像乐高：一次构建一块。

### 2) 变量和引号

Shell 变量是无类型字符串。用 `name=value` 赋值，用 `$name` 引用。`=` 周围必须**没有空格**。

```sh
#!/bin/sh
# vars.sh   演示变量和正确的引号使用

name="dev"
greeting="Welcome"
# 双引号保留空格并展开变量。
echo "$greeting, $name"
# 单引号阻止展开。这打印字面字符。
echo '$greeting, $name'

# 命令替换捕获命令的输出。
today="$(date +%Y-%m-%d)"
echo "Today is $today"
```

初学者常见陷阱：

- 在 `=` 周围使用空格：`name = dev` 是错误。
- 当变量可能包含空格时忘记引号。养成使用 `"${var}"` 的习惯。

### 3) 退出状态和短路操作符

每个命令返回一个**退出状态**。零表示成功。非零表示错误。Shell 让你使用 `&&` 和 `||` 链接命令。

```sh
#!/bin/sh
# status.sh   显示退出码和条件链接

# 尝试列出一个存在的目录。'ls' 应该返回 0。
ls /etc && echo "Listing /etc succeeded"

# 尝试某个会失败的命令。'false' 总是返回非零。
false || echo "Previous command failed, so this message appears"

# 你可以用 $? 显式测试最后一个状态
echo "Last status was $?"
```

### 4) 测试和条件：`if`、`[ ]`、文件和数字

使用 `if` 配合 `test` 命令或其括号形式 `[ ... ]`。括号内必须有空格。

```sh
#!/bin/sh
# ifs.sh   演示文件和数字测试

file="/etc/rc.conf"

# -f 测试普通文件是否存在
if [ -f "$file" ]; then
  echo "$file exists"
else
  echo "$file does not exist"
fi

num=5
if [ "$num" -gt 3 ]; then
  echo "$num is greater than 3"
fi

# 字符串测试
user="$(whoami)"
if [ "$user" = "root" ]; then
  echo "You are root"
else
  echo "You are $user"
fi
```

有用的文件测试：

- `-e` 存在
- `-f` 普通文件
- `-d` 目录
- `-r` 可读
- `-w` 可写
- `-x` 可执行

数字比较：

- `-eq` 等于
- `-ne` 不等于
- `-gt` 大于
- `-ge` 大于等于
- `-lt` 小于
- `-le` 小于等于

### 5) 循环：`for` 和 `while`

循环让你在文件或输入行上重复工作。

```sh
#!/bin/sh
# loops.sh   /bin/sh 中的 for 和 while 循环

# 遍历路径名的 'for' 循环。始终用引号展开以安全处理空格。
for f in /etc/*.conf; do
  echo "Found conf file: $f"
done

# 安全地从文件读取行的 'while' 循环。
# 'IFS=' 和 'read -r' 避免裁剪空格和反斜杠转义。
count=0
while IFS= read -r line; do
  count=$((count + 1))
done < /etc/hosts
echo "The /etc/hosts file has $count lines"
```

POSIX sh 中的算术使用 `$(( ... ))` 进行简单整数运算。

### 6) case 语句用于整洁分支

`case` 非常适合你有多个模式要匹配时。

```sh
#!/bin/sh
# case.sh   使用 case 语句处理选项

action="$1"   # 第一个命令行参数

case "$action" in
  start)
    echo "Starting service"
    ;;
  stop)
    echo "Stopping service"
    ;;
  restart)
    echo "Restarting service"
    ;;
  *)
    echo "Usage: $0 {start|stop|restart}" >&2
    exit 2
    ;;
esac
```

### 7) 函数来组织你的脚本

函数保持代码可读和可重用。

```sh
#!/bin/sh
# functions.sh - 演示在 shell 脚本中使用函数和命令行参数。
#
# 用法：
#   ./functions.sh NUM1 NUM2
# 示例：
#   ./functions.sh 5 7
#   这将输出："[INFO] 5 + 7 = 12"

# 一个简单的打印信息消息的函数
say() {
  # "$1" 代表传递给函数的第一个参数
  echo "[INFO] $1"
}

# 一个求两整数之和的函数
sum() {
  # "$1" 和 "$2" 是第一个和第二个参数
  local a="$1"
  local b="$2"

  # 执行算术展开以相加它们
  echo $((a + b))
}

# --- 主脚本执行从这里开始 ---

# 确保用户提供了两个参数
if [ $# -ne 2 ]; then
  echo "Usage: $0 NUM1 NUM2"
  exit 1
fi

say "Beginning work"

# 用提供的参数调用 sum() 函数
result="$(sum "$1" "$2")"

# 以漂亮的格式打印结果
say "$1 + $2 = $result"
```

### 8) 实际示例：一个小型备份脚本

这个脚本为目录创建带时间戳的归档到 `~/backups`。它只使用 FreeBSD 基础系统可用的基本工具。

```sh
#!/bin/sh
# backup.sh   创建目录的带时间戳 tar 归档
# 用法：./backup.sh /path/to/source
# 注意：
#  - 使用 /bin/sh，所以在干净的 FreeBSD 14.x 安装上运行。
#  - 如果不存在则创建 ~/backups。
#  - 将归档命名为 sourcebasename-YYYYMMDD-HHMMSS.tar.gz

set -eu
# set -e：如果任何命令失败则立即退出
# set -u：将使用未设置的变量视为错误

# 验证输入
if [ $# -ne 1 ]; then
  echo "Usage: $0 /path/to/source" >&2
  exit 2
fi

src="$1"

# 验证源是目录
if [ ! -d "$src" ]; then
  echo "Error: $src is not a directory" >&2
  exit 3
fi

# 准备目标目录
dest="${HOME}/backups"
mkdir -p "$dest"

# 使用最后一个路径组件构建安全的归档名称
base="$(basename "$src")"
stamp="$(date +%Y%m%d-%H%M%S)"
archive="${dest}/${base}-${stamp}.tar.gz"

# 创建归档
# tar(1) 在基础系统中。标志意思是：
#  - c: 创建  - z: gzip  - f: 文件名  - C: 切换到目录
tar -czf "$archive" -C "$(dirname "$src")" "$base"

echo "Backup created: $archive"
```

运行它：

```sh
% chmod +x backup.sh
% ./backup.sh ~/directory_you_want_to_backup
```

你会在 `~/backups` 下找到归档。

### 9) 安全处理临时文件

永远不要硬编码像 `/tmp/tmpfile` 这样的名字。使用基础系统中的 `mktemp(1)`。

```sh
#!/bin/sh
# tmp_demo.sh   安全创建和清理临时文件

set -eu

tmpfile="$(mktemp -t myscript)"
# 安排成功或错误时退出清理
cleanup() {
  [ -f "$tmpfile" ] && rm -f "$tmpfile"
}
trap cleanup EXIT

echo "Temporary file is $tmpfile"
echo "Hello temp" > "$tmpfile"
echo "Contents: $(cat "$tmpfile")"
```

`trap` 调度一个函数在脚本退出时运行，这可以防止残留文件。

### 10) 调试你的脚本

- `set -x` 在执行前打印每个命令。在顶部附近添加，修复后移除。
- `echo` 进度消息，让用户知道正在发生什么。
- 显式检查返回码并处理失败。
- 通过重定向输出记录到文件：`mycmd >> ~/my.log 2>&1`。

示例：

```sh
#!/bin/sh
# debug_demo.sh   显示简单跟踪

# set -x 注释以禁用详细跟踪：
set -x

echo "Step 1"
ls /etc >/dev/null

echo "Step 2"
date
```

### 11) 综合应用：按类型整理下载

这个小工具将 `~/Downloads` 中的文件按扩展名分类到子文件夹。它演示了循环、case、测试和安全检查。

```sh
#!/bin/sh
# organize_downloads.sh - 按文件扩展名整理 ~/Downloads
#
# 用法：
#   ./organize_downloads.sh
#
# 创建 Documents、Images、Audio、Video、Archives、Other 等子目录
# 并安全地将匹配的文件移动到其中。

set -eu

downloads="${HOME}/Downloads"

# 创建临时文件存储文件列表
tmpfile=$(mktemp)

# 脚本退出时删除临时文件（正常或错误）
trap 'rm -f "$tmpfile"' EXIT

# 确保 Downloads 目录存在
if [ ! -d "$downloads" ]; then
  echo "Downloads directory not found at $downloads" >&2
  exit 1
fi

cd "$downloads"

# 如果缺失则创建目标文件夹
mkdir -p Documents Images Audio Video Archives Other

# 查找当前目录中的所有普通文件（非递归，排除隐藏文件）
# -maxdepth 1: 不搜索子目录
# -type f: 只查找普通文件（不是目录或符号链接）
# ! -name ".*": 排除隐藏文件（以点开头的文件）
count=0
find . -maxdepth 1 -type f ! -name ".*" > "$tmpfile"
while IFS= read -r f; do
  # 从路径中去掉开头的 "./"
  fname=${f#./}
  
  # 如果文件名为空则跳过（不应发生，但安全检查）
  [ -z "$fname" ] && continue

  # 将文件名扩展名转换为小写以进行匹配
  lower=$(printf '%s' "$fname" | tr '[:upper:]' '[:lower:]')

  case "$lower" in
    *.pdf|*.txt|*.md|*.doc|*.docx)  dest="Documents" ;;
    *.png|*.jpg|*.jpeg|*.gif|*.bmp) dest="Images" ;;
    *.mp3|*.wav|*.flac)             dest="Audio" ;;
    *.mp4|*.mkv|*.mov|*.avi)        dest="Video" ;;
    *.zip|*.tar|*.gz|*.tgz|*.bz2)   dest="Archives" ;;
    *)                              dest="Other" ;;
  esac

  echo "Moving '$fname' -> $dest/"
  mv -n -- "$fname" "$dest/"   # -n 防止覆盖现有文件
  count=$((count + 1))         # 增加计数器
done < "$tmpfile"              # 将临时文件输入 while 循环

if [ $count -eq 0 ]; then
  echo "No files to organize."
else
  echo "Done. Organized $count file(s)."
fi
```

### 实践实验：三个迷你任务

1. **编写一个日志记录器**
    创建 `logger.sh`，将带时间戳的行追加到 `~/activity.log`，包含当前目录和用户。运行它，然后用 `tail` 查看日志。
2. **检查磁盘空间**
    创建 `check_disk.sh`，当根文件系统使用超过 80% 时发出警告。使用 `df -h /` 并用 `${var%%%}` 风格裁剪或简单的 `awk` 解析百分比。如果超过阈值，以状态 1 退出，这样 cron 可以警告你。
3. **包装你的备份**
    创建 `backup_cron.sh`，调用之前的 `backup.sh` 并将输出记录到 `~/backup.log`。添加一个 crontab 条目，每天凌晨 3 点运行它。记住在脚本内使用完整路径。

所有脚本应以 `#!/bin/sh` 开头，包含解释每一步的注释，在变量展开周围使用引号，并在合理处处理错误。

### 初学者常见陷阱及如何避免

- **在 `#!/bin/sh` 脚本中使用 bash 功能。** 坚持使用 POSIX 结构。如果你需要 bash，在 shebang 中说明并记住它在 FreeBSD 上位于 `/usr/local/bin/bash`。
- **忘记引号变量。** 使用 `"${var}"` 防止词分割和 glob 意外。
- **假设 cron 下环境相同。** 始终使用完整路径并将输出重定向到日志文件。
- **硬编码临时文件名。** 使用 `mktemp` 和 `trap` 清理。
- **赋值时 `=` 周围有空格。** `name=value` 是正确的。`name = value` 不是。

### 小结

在本节中，你学习了在干净 FreeBSD 安装上运行的可移植脚本的**原生 FreeBSD 方式**。你现在可以编写带有 `/bin/sh` 的小程序、处理参数、测试条件、循环遍历文件、定义函数、安全使用临时文件以及用简单工具调试问题。在编写驱动程序时，脚本将帮助你重复测试、收集日志并可靠地打包构建。

在我们继续之前，提醒一下：你不需要记住每个结构或命令选项。在 UNIX 中高效工作的一部分是知道在**正确的时间在哪里找到正确的信息**。

下一节，我们将严格审视**可移植性**本身，看 shell 之间的细微差异、保持脚本在各系统间健壮的习惯，以及如何选择不会让你后来惊讶的特性。

## Shell 可移植性：处理边缘情况和 bash vs sh

到目前为止，我们使用 FreeBSD 的原生 `/bin/sh` shell 编写脚本，它遵循 POSIX 标准。这使我们的脚本可移植到不同的 UNIX 系统。但当你在网上探索 shell 脚本示例或收到其他开发者的贡献时，你会遇到为 **bash** 编写的脚本，使用了 POSIX sh 中不可用的功能。

理解 bash 和 sh 的区别，知道如何处理如异常文件名这样的边缘情况，将帮助你编写健壮的脚本并决定何时可移植性比便利性更重要。

### 问题：包含特殊字符的文件名

UNIX 允许文件名包含几乎任何字符，除了正斜杠 `/`（分隔目录）和空字符 `\0`。这意味着文件名可以合法包含空格、换行符、制表符或其他令人惊讶的字符。

让我们创建一个在文件名中包含换行符的文件，看看它如何影响我们的脚本：

```sh
% cd ~
% touch $'file_with\nnewline.txt'
% ls
file_with?newline.txt
```

`?` 出现是因为 `ls` 在显示文件名时替换不可打印字符。实际文件名包含：

```text
file_with
newline.txt
```

现在让我们看看当脚本尝试处理这个文件时会发生什么。

### 一个会出错的简单方法

这是一个列出主目录文件的简单脚本：

```sh
#!/bin/sh
# list_files.sh - 计算主目录中的文件

set -eu
cd "${HOME}"

count=0
while IFS= read -r f; do
  fname=${f#./}
  echo "File found: '$fname'"
  count=$((count + 1))
done << EOF
$(find . -maxdepth 1 -type f ! -name ".*" -print)
EOF

echo "Total files found: $count"
```

用我们的异常文件名运行这个脚本会产生错误结果：

```sh
% ./list_files.sh
File found: 'file_with'
File found: 'newline.txt'
Total files found: 2
```

脚本认为一个文件实际上是两个文件，因为 `find -print` 每行输出一个路径，而我们的文件名包含换行符字符。脚本在一个完全有效的 UNIX 文件名上出错了。

### bash 解决方案：使用空分隔符

修复这个问题的一种方法是使用空字符（`\0`）作为分隔符而不是换行符。Bash 通过 `read` 命令的 `-d` 选项支持这一点：

```sh
#!/usr/local/bin/bash
# list_files_bash.sh - 用 bash 正确处理异常文件名

set -eu
cd "${HOME}"

count=0
while IFS= read -r -d '' f; do
  fname=${f#./}
  echo "File found: '$fname'"
  count=$((count + 1))
done < <(find . -maxdepth 1 -type f ! -name ".*" -print0)

echo "Total files found: $count"
```

注意三个变化：

1. **Shebang**：改为 `#!/usr/local/bin/bash`（`pkg install bash` 后 bash 在 FreeBSD 上的位置）
2. **find 标志**：从 `-print` 改为 `-print0`（输出空分隔路径）
3. **read 选项**：添加 `-d ''` 告诉 `read` 使用空作为分隔符

这个版本工作正常：

```sh
% ./list_files_bash.sh
File found: 'file_with
newline.txt'
Total files found: 1
```

缺点？**这个脚本现在需要 bash**，它不是 FreeBSD 基础系统的一部分。它创建了一个依赖。

### POSIX 兼容的替代方案

如果可移植性比处理每个可能的边缘情况更重要，我们可以写一个避免 bash 特定功能的 POSIX 兼容版本：

```sh
#!/bin/sh
# list_files_posix.sh - POSIX 兼容的文件列表

set -eu
cd "${HOME}"

# 使用临时文件代替管道
tmpfile=$(mktemp)
trap 'rm -f "$tmpfile"' EXIT

# 将 find 结果存入临时文件
find . -maxdepth 1 -type f ! -name ".*" > "$tmpfile"

count=0
while IFS= read -r f; do
  fname=${f#./}
  [ -z "$fname" ] && continue
  
  echo "File found: '$fname'"
  count=$((count + 1))
done < "$tmpfile"

echo "Total files found: $count"
```

这个版本：

- 在任何 POSIX 兼容的 shell 上工作（不需要 bash）
- 使用临时文件代替管道以避免子 shell 变量问题
- 用 `trap` 自动清理
- 处理带空格和大多数特殊字符的文件名

限制？它仍然无法正确处理带换行符的文件名，因为 POSIX sh 的 `read` 命令没有办法使用不同的分隔符。对于这个版本：

```sh
% ./list_files_posix.sh
File found: 'file_with'
File found: 'newline.txt'
Total files found: 2
```

### 理解权衡

这揭示了 shell 脚本编写中的一个重要决策点：

**可移植性 vs 边缘情况覆盖**

| 方法             | 优点                               | 缺点                                   |
| ---------------- | ---------------------------------- | -------------------------------------- |
| **POSIX sh**     | 到处运行，无依赖                   | 无法处理带换行符的文件名               |
| **bash with -d** | 处理所有有效文件名                 | 需要安装 bash                          |
| **find -exec**   | POSIX 兼容，处理一切               | 更复杂的语法                           |

对于大多数实际脚本，POSIX 方法已经足够。带换行符的文件名在人工设计的示例或安全利用之外极其罕见。带空格、unicode 字符和其他可打印字符的文件名在 POSIX 版本中工作正常。

### 何时选择 bash

在以下情况下使用 bash：

- 你在编写个人工具，bash 保证可用
- 你确实需要处理带换行符的文件名（非常罕见）
- 你需要 bash 特定功能如数组、扩展正则或高级字符串操作
- 脚本是已经依赖 bash 的项目的一部分

在以下情况下使用 POSIX sh：

- 编写需要在任何 FreeBSD 系统上运行的系统管理脚本
- 为 FreeBSD 基础系统脚本做贡献
- 需要最大可移植性
- 脚本可能在救援模式或最小环境中运行

### 第三种选择：find -exec

为了完整性，这里有一个正确处理所有文件名且不需要 bash 的 POSIX 兼容方法：

```sh
#!/bin/sh
# list_files_exec.sh - 使用 find -exec 处理所有文件名

set -eu
cd "${HOME}"

find . -maxdepth 1 -type f ! -name ".*" -exec sh -c '
  for f; do
    fname=${f#./}
    printf "File found: '\''%s'\''\n" "$fname"
  done
' sh {} +
```

这之所以有效是因为 `find -exec` 将文件名作为参数传递，而不是通过管道或基于行的读取。它符合 POSIX 并处理每个边缘情况，但语法对初学者不太直观。

### 实用建议

编写 shell 脚本时：

1. **从 `/bin/sh` 开始** - 以 POSIX 兼容脚本开始
2. **引用你的变量** - 始终使用 `"$var"` 处理空格
3. **用异常文件名测试** - 创建带空格的测试文件
4. **记录依赖** - 如果使用 bash，在注释中说明
5. **接受合理的限制** - 不要为你永远不会遇到的边缘情况牺牲可移植性

我们之前编写的 organize_downloads.sh 脚本使用 POSIX 兼容的临时文件方法。它在正确处理绝大多数真实世界文件名的同时保持可移植到任何 FreeBSD 系统。

记住：**最好的脚本是能在目标环境中可靠工作的脚本**。不要为你永远不会遇到的边缘情况添加 bash 作为依赖，但如果在已经安装 bash 的系统上编写个人工具，不要用 POSIX 限制折磨自己。

### 小结

你现在看到可移植性是一个设计选择，而不是事后想法。你学到了在 FreeBSD 上必须到处运行的脚本偏向 POSIX `/bin/sh`，避免 bash 特定功能，使用 `printf` 而不是松散的 `echo`，默认引用变量，检查退出码，并选择清晰的 shebang 以便正确的解释器运行你的代码。一路上，我们重温了你已经知道的构建块：参数、条件、循环、函数，以及安全的临时文件并为可预测的行为在各系统间进行调整。

没有人能把所有细节都记在脑子里，你也不需要。下一节向你展示**FreeBSD 式查阅**的地方：手册页、`apropos`、内置帮助、FreeBSD 手册和社区资源。这些将成为你深入设备驱动程序开发时的日常伙伴。

## 在 FreeBSD 中寻求帮助和文档

没有人，即使是最有经验的开发者，能记住每个命令、选项或系统调用。像 FreeBSD 这样的 UNIX 系统的真正优势在于它自带**优秀的文档**并在你卡住时有支持性的社区可以帮助。

在本节中，我们将探索获取信息的主要方式：**手册页、FreeBSD 手册、在线资源和社区**。到最后，你将确切知道有问题时去哪里找，无论是关于使用 `ls` 还是编写设备驱动程序。

### man 页面的力量

**手册页**，或 **man 页面**，是 UNIX 的内置参考系统。每个命令、系统调用、库函数、配置文件和内核接口都有 man 页面。

你用 `man` 命令阅读它们，例如：

```console
% man ls
```

这打开 `ls`（列出目录内容的命令）的文档。用空格键滚动，`q` 退出。

#### man 页面章节

FreeBSD 将 man 页面组织成编号章节。同名可能存在于多个章节，所以你指定要哪个。

- **1** - 用户命令（如 `ls`、`cp`、`ps`）
- **2** - 系统调用（如 `open(2)`、`write(2)`）
- **3** - 库函数（C 标准库、数学函数）
- **4** - 设备驱动和特殊文件（如 `null(4)`、`random(4)`）
- **5** - 文件格式和约定（`passwd(5)`、`rc.conf(5)`）
- **7** - 杂项（协议、约定）
- **8** - 系统管理命令（如 `ifconfig(8)`、`shutdown(8)`）
- **9** - 内核开发者接口（对驱动程序编写者至关重要！）

例如：

```sh
% man 2 open      # 系统调用 open()
% man 9 bus_space # 访问设备寄存器的内核函数
```

#### man 章节 9：内核开发者手册

大多数 FreeBSD 用户生活在章节 **1**（用户命令），管理员在章节 **8**（系统管理）花费大量时间。但作为驱动程序开发者，你将在 **章节 9** 度过大量时间。

章节 9 包含只在内核内部可用的函数、宏和子系统的**内核开发者接口**文档。

一些例子：

- `man 9 device` - 设备驱动接口概述。
- `man 9 bus_space` - 访问硬件寄存器。
- `man 9 mutex` - 内核同步原语。
- `man 9 taskqueue` - 在内核中调度延迟工作。
- `man 9 malloc` - 内核内内存分配。

不像章节 2（系统调用）或章节 3（库），这些在**用户空间不可用**。它们是内核本身的一部分，你在编写驱动程序和内核模块时会使用它们。

把章节 9 当作 **FreeBSD 内核的开发者 API 手册**。

#### 实践预览

你还不需要理解所有细节，但可以偷看：

```sh
% man 9 device
% man 9 bus_dma
% man 9 sysctl
```

你会看到风格与用户命令 man 页面不同：它们专注于**内核函数、结构和使用示例**。

本书后面，当我们介绍新的内核功能时，我们会不断引用章节 9。把它当作你路上最重要的伙伴。

#### 搜索 man 页面

如果你不知道确切的命令名，用 `-k` 标志（等同于 `apropos`）：

```console
man -k network
```

这显示所有与网络相关的 man 页面。

另一个例子：

```console
man -k disk | less
```

这将显示与磁盘相关的工具、驱动程序和系统调用。

### FreeBSD 手册

**FreeBSD 手册**是操作系统的官方、全面指南。

你可以在网上阅读：

https://docs.freebsd.org/en/books/handbook/

手册涵盖：

- 安装 FreeBSD
- 系统管理
- 网络
- 存储和文件系统
- 安全和 jails
- 高级主题

手册是本书的**绝佳补充**。当我们专注于设备驱动程序开发时，手册给你可以随时回来查阅的广泛系统知识。

#### 其他文档

- **在线 man 页面**：https://man.freebsd.org
- **FreeBSD Wiki**：https://wiki.freebsd.org（社区维护的笔记、HOWTO 和进行中的文档）。
- **开发者手册**：https://docs.freebsd.org/en/books/developers-handbook 面向程序员。
- **Porter 手册**：https://docs.freebsd.org/en/books/porters-handbook 如果你为 FreeBSD 打包软件。

### 社区和支持

文档可以帮你走很远，但有时你需要与真人交谈。FreeBSD 有活跃且友好的社区。

- **邮件列表**：https://lists.freebsd.org
  - `freebsd-questions@` 用于一般用户帮助。
  - `freebsd-hackers@` 用于开发讨论。
  - `freebsd-drivers@` 专门用于设备驱动程序开发。
- **FreeBSD 论坛**：https://forums.freebsd.org 一个友好且对初学者友好的提问场所。
- **用户组**：
  - 世界各地有 **FreeBSD 和 BSD 用户组**组织聚会、讲座和研讨会。
  - 例子包括 *NYCBUG（纽约市 BSD 用户组）*、*BAFUG（湾区 FreeBSD 用户组）* 和许多大学社团。
  - 你通常可以通过 FreeBSD Wiki、本地技术邮件列表或 meetup.com 找到它们。
  - 如果你附近没有，考虑组建一个小型小组，即使少数爱好者在线或亲自会面也能成为宝贵的支持网络。
- **聊天**：
  - Libera.Chat 上的 **IRC**（`#freebsd`）。
  - **Discord** 社区存在且相当活跃，使用此链接加入：https://discord.com/invite/freebsd
- **Reddit**：https://reddit.com/r/freebsd

用户组和论坛特别有价值，因为你通常可以用母语提问，甚至遇到在你所在地区为 FreeBSD 做贡献的人。

#### 如何寻求帮助

有时每个人都会卡住。FreeBSD 的优势之一是它活跃和支持性的社区，但为了获得有用的答案，你需要提出清晰、完整、尊重的问题。

当你发帖到邮件列表、论坛、IRC 或 Discord 频道时，包括：

- **你的 FreeBSD 版本**
   运行：

  ```sh
  % uname -a
  ```

  这告诉帮助者你使用的确切版本、补丁级别和架构。

- **你试图做什么**
   描述你的目标，不只是失败的命令。帮助者有时可以建议比你尝试的更好的方法。

- **确切的错误消息**
   复制粘贴错误文本而不是意译。即使是小差异也很重要。

- **重现问题的步骤**
   如果其他人可以重复你的问题，他们通常可以更快解决。

- **你已经尝试过的**
   提及命令、配置更改或你查阅的文档。这显示你已做出努力，防止人们建议你已经做过的事情。

#### 糟糕帮助请求示例

> "Ports 不工作，我怎么修复它？"

这缺少版本、命令、错误和上下文。没人能回答而不去猜测。

#### 好的帮助请求示例

> "我在 amd64 上运行 FreeBSD 14.3-RELEASE。我尝试用 `cd /usr/ports/sysutils/htop && make install clean` 从 ports 构建 `htop`。构建失败，错误如下：
>
> ```
> error: ncurses.h: No such file or directory
> ```
>
> 我已经尝试了 `pkg install ncurses`，但错误仍然存在。我接下来应该检查什么？"

这简短但完整；版本、命令、错误和故障排除步骤都在。

**提示**：始终保持礼貌和耐心。记住，大多数 FreeBSD 贡献者是**志愿者**。清晰、尊重的问题不仅增加获得有用回复的机会，还建立社区中的善意。

### 实践实验：探索文档

1. 打开 `ls` 的 man 页面。找到并尝试至少两个你不知道的选项。

   ```sh
   % man ls
   ```

2. 用 `man -k` 搜索与磁盘相关的命令。

   ```sh
   % man -k disk | less
   ```

3. 打开 `open(2)` 的 man 页面并与 `open(3)` 比较。有什么区别？

4. 偷看内核开发者文档：

   ```sh
   % man 9 device
   ```

5. 访问 https://docs.freebsd.org/ 并找到关于系统启动（`rc.d`）的页面。与 `man rc.conf` 比较。

### 小结

FreeBSD 为你提供了自学的好工具。**man 页面**是你的第一站；它们总是存在于你的系统上、总是最新的，涵盖从基本命令到内核 API 的一切。**手册**是你的大局指南，**社区**邮件列表、论坛、用户组和在线聊天在你需要人类答案时提供帮助。

稍后，当你编写驱动程序时，你会严重依赖 man 页面（特别是章节 9）以及 FreeBSD 邮件列表和论坛中的讨论。知道如何找到信息与记忆命令一样重要。

接下来，我们将看看系统内部以**窥探内核消息和可调参数**。如 `dmesg` 和 `sysctl` 这样的工具让你看到内核正在做什么，并将在你开始加载和测试自己的设备驱动程序时变得至关重要。

## 窥探内核和系统状态

此时，你知道如何在 FreeBSD 中移动、管理文件、控制进程，甚至编写脚本。这让你成为有能力的用户。但编写驱动程序意味着走进**内核的思维**。你需要看到 FreeBSD 自己看到的东西：

- 检测到了什么硬件？
- 加载了哪些驱动程序？
- 内核中存在哪些可调旋钮？
- 设备如何呈现给操作系统？

FreeBSD 给你**三个神奇的窗口进入内核状态**：

1. **`dmesg`** - 内核的日记。
2. **`sysctl`** - 充满开关和仪表的控制面板。
3. **`/dev`** - 设备以文件形式出现的门口。

这三个工具将成为你的**伙伴**。每次添加或调试驱动程序时，你会使用它们。现在让我们一步步看看它们。

### dmesg：阅读内核的日记

把 FreeBSD 想象成飞行员启动飞机。随着系统引导，内核检查其硬件：CPU、内存、磁盘、USB 设备，每个驱动程序报告回来。这些消息不会丢失；它们存储在你随时可以用以下命令读取的缓冲区中：

```sh
% dmesg | less
```

你会看到类似这样的行：

```yaml
Copyright (c) 1992-2023 The FreeBSD Project.
Copyright (c) 1979, 1980, 1983, 1986, 1988, 1989, 1991, 1992, 1993, 1994
        The Regents of the University of California. All rights reserved.
FreeBSD is a registered trademark of The FreeBSD Foundation.
FreeBSD 14.3-RELEASE releng/14.3-n271432-8c9ce319fef7 GENERIC amd64
FreeBSD clang version 19.1.7 (https://github.com/llvm/llvm-project.git llvmorg-19.1.7-0-gcd708029e0b2)
VT(vga): text 80x25
CPU: AMD Ryzen 7 5800U with Radeon Graphics          (1896.45-MHz K8-class CPU)
  Origin="AuthenticAMD"  Id=0xa50f00  Family=0x19  Model=0x50  Stepping=0
  Features=0x1783fbff<FPU,VME,DE,PSE,TSC,MSR,PAE,MCE,CX8,APIC,SEP,MTRR,PGE,MCA,CMOV,PAT,PSE36,MMX,FXSR,SSE,SSE2,HTT>
  Features2=0xfff83203<SSE3,PCLMULQDQ,SSSE3,FMA,CX16,SSE4.1,SSE4.2,x2APIC,MOVBE,POPCNT,TSCDLT,AESNI,XSAVE,OSXSAVE,AVX,F16C,RDRAND,HV>
  AMD Features=0x2e500800<SYSCALL,NX,MMX+,FFXSR,Page1GB,RDTSCP,LM>
  AMD Features2=0x8003f7<LAHF,CMP,SVM,CR8,ABM,SSE4A,MAS,Prefetch,OSVW,PCXC>
  Structured Extended Features=0x219c07ab<FSGSBASE,TSCADJ,BMI1,AVX2,SMEP,BMI2,ERMS,INVPCID,RDSEED,ADX,SMAP,CLFLUSHOPT,CLWB,SHA>
  Structured Extended Features2=0x40061c<UMIP,PKU,OSPKE,VAES,VPCLMULQDQ,RDPID>
  Structured Extended Features3=0xac000010<FSRM,IBPB,STIBP,ARCH_CAP,SSBD>
  XSAVE Features=0xf<XSAVEOPT,XSAVEC,XINUSE,XSAVES>
  IA32_ARCH_CAPS=0xc000069<RDCL_NO,SKIP_L1DFL_VME,MDS_NO>
  AMD Extended Feature Extensions ID EBX=0x1302d205<CLZERO,XSaveErPtr,WBNOINVD,IBPB,IBRS,STIBP,STIBP_ALWAYSON,SSBD,VIRT_SSBD,PSFD>
  SVM: NP,NRIP,VClean,AFlush,NAsids=16
  ...
  ...
```

这是内核在告诉你：

- **它发现了什么硬件**，
- **哪个驱动程序认领了它**，
- 有时，**出了什么问题**。

本书后面，当你加载自己的驱动程序时，`dmesg` 是你寻找第一条"Hello, kernel!"消息的地方。

`dmesg` 的输出可能非常长，你可以用 `grep` 筛选只看你需要的内容，例如：

```sh
% dmesg | grep ada
```

这只会显示关于磁盘设备（`ada0`、`ada1`）的消息。

### sysctl：内核的控制面板

如果 `dmesg` 是日记，`sysctl` 就是**布满旋钮和仪表的仪表盘**。它在运行时暴露数千个内核变量：有些是只读的（系统信息），有些是可调的（系统行为）。

尝试这些命令：

```console
% sysctl kern.ostype
% sysctl kern.osrelease
% sysctl hw.model
% sysctl hw.ncpu
```

输出可能看起来像：

```text
kern.ostype: FreeBSD
kern.osrelease: 14.3-RELEASE
hw.model: AMD Ryzen 7 5800U with Radeon Graphics
hw.ncpu: 8
```

这里你刚刚问内核：

- 我运行的是什么操作系统？
- 什么版本？
- 什么 CPU？
- 多少核心？

#### 探索一切

要看到你用 `sysctl` 可以微调的所有参数，可以运行下面的命令：

```sh
% sysctl -a | less
```

这打印**整个控制面板**——数千个值。它们按类别组织：

- `kern.*` - 内核属性和设置。
- `hw.*` - 硬件信息。
- `net.*` - 网络栈详情。
- `vfs.*` - 文件系统设置。
- `debug.*` - 调试变量（通常对开发者有用）。

起初它让人眼花缭乱，但别担心，你会学会钓出重要的东西。

#### 更改值

有些 sysctl 是可写的。例如：

```sh
% sudo sysctl kern.hostname=myfreebsd
% hostname
```

你刚刚在运行时更改了主机名。

重要：这样做的更改重启后消失，除非保存在 `/etc/sysctl.conf` 中。

### /dev：设备诞生的地方

现在是最激动人心的部分。

FreeBSD 将设备表示为 `/dev` 内的**特殊文件**。这是 UNIX 最优雅的理念之一：

> 如果一切皆文件，那么一切都可以以一致的方式访问。

运行：

```sh
% ls -d /dev/* | less
```

你会看到大量名称：

- `/dev/null`- 数据消失的"黑洞"。
- `/dev/zero`- 无限零流。
- `/dev/random`- 加密安全的随机数。
- `/dev/tty`- 你的终端。
- `/dev/ada0`- 你的 SATA 磁盘。
- `/dev/da0`- USB 磁盘。

尝试交互：

```sh
echo "Testing" > /dev/null         # 静默丢弃输出
head -c 16 /dev/zero | hexdump     # 以十六进制显示零
head -c 16 /dev/random | hexdump   # 来自内核的随机字节
```

稍后，当你创建第一个驱动程序时，它会在这里显示为一个名为 `/dev/hello` 的文件。读取或写入该文件将触发**你的内核代码**。这是你将感受到用户空间和内核之间桥梁的时刻。

### 实践实验：你的第一次内部窥探

1. 查看所有内核消息：

	```sh
	   % dmesg | less
		```

2. 找到你的存储设备：

   ```sh
   % dmesg | grep ada
   ```

3. 询问内核关于你的 CPU：

   ```sh
   % sysctl hw.model
   % sysctl hw.ncpu
   ```

4. 暂时更改你的主机名：

   ```sh
   % sudo sysctl kern.hostname=mytesthost
   % hostname
   ```

5. 与特殊设备文件交互：

   ```
   % echo "Hello FreeBSD" > /dev/null
   % head -c 8 /dev/zero | hexdump
   % head -c 8 /dev/random | hexdump
   ```

通过这个简短的实验，你已经在读取内核消息、查询内核变量以及触摸设备节点——这正是专业开发者每天做的事。

### 从 Shell 到硬件：大局

要理解为什么如 `dmesg`、`sysctl` 和 `/dev` 这样的工具如此有用，有助于描绘 FreeBSD 如何分层：

```text
+----------------+
|   用户空间     |  ← 你运行的命令：ls, ps, pkg, scripts
+----------------+
        ↓
+----------------+
|   Shell (sh)   |  ← 将你的命令解释为系统调用
+----------------+
        ↓
+----------------+
|    内核        |  ← 处理进程、内存、设备、文件系统
+----------------+
        ↓
+----------------+
|   硬件         |  ← CPU、RAM、磁盘、USB、网卡
+----------------+
```

每当你 shell 中输入命令，它都会沿这个栈下传：

- **Shell** 解释它。
- **内核** 通过管理进程、内存和设备执行它。
- **硬件** 响应。

然后结果冒泡回来给你看。

理解这个流程对驱动程序开发者至关重要：当你与 `/dev` 交互时，你正在直接连接到内核，而内核与硬件通信。

### 初学者常见陷阱

探索内核可能很刺激，但这里有一些需要注意的常见错误：

1. **混淆 `dmesg` 与系统日志**

   - `dmesg` 只显示内核的环形缓冲区，不是所有日志。
   - 新消息可能在旧消息被推出后消失。
   - 对于完整日志，检查 `/var/log/messages`。

2. **忘记 `sysctl` 更改不持久**

   - 如果你用 `sysctl` 更改设置，它会在重启时重置。

   - 要使其永久，添加到 `/etc/sysctl.conf`。

   - 示例：

   ```sh
     % echo 'kern.hostname="myhost"' | sudo tee -a /etc/sysctl.conf
     ```

3. **覆写 `/dev` 中的文件**

   - `/dev` 条目不是普通文件；它们是与内核的实时连接。
   - 重定向输出到它们可能有实际效果。
   - 写入 `/dev/null` 是安全的，但写入随机数据到 `/dev/ada0`（你的磁盘）可能销毁它。
   - 经验法则：探索 `/dev/null`、`/dev/zero`、`/dev/random` 和 `/dev/tty`，但让存储设备（`ada0`、`da0`）保持原样，除非你确切知道你在做什么。

4. **期望 `/dev` 条目保持不变**

   - 设备随着硬件添加或移除而出现和消失。
   - 例如，插入 USB 棒可能创建 `/dev/da0`。
   - 不要在不验证的情况下将设备名称硬编码到脚本中。

5. **不在自动化中使用完整路径**

   - Cron 和其他自动化工具可能没有与你 shell 相同的 `PATH`。
   - 在脚本内核交互时始终使用完整路径（`/sbin/sysctl`、`/bin/echo`）。

### 小结

在本节中，你打开了三个神奇的窗口进入 FreeBSD 的内核：

- `dmesg`- 系统的日记，记录硬件检测和驱动程序消息。
- `sysctl` - 揭示（有时调整）内核设置的控制面板。
- `/dev`- 设备以文件形式诞生的地方。

要记住的**大局**是：每当你输入命令，它都通过 shell、下到内核、最终到硬件。然后结果冒泡回来给你看。如 `dmesg`、`sysctl` 和 `/dev` 这样的工具让你窥视那个流程，看到内核在幕后做什么。

这些不只是抽象工具；它们正是你将看到**自己的驱动程序**出现在系统中的方式。当你加载模块时，你会看到 `dmesg` 亮起来，你可能会用 `sysctl` 暴露一个旋钮，你将与 `/dev` 下的设备节点交互。

值得停下来思考这告诉你关于前方的路。`dmesg` 中描述硬件附加的每一行、以 `kern.` 或 `vm.` 开头的每个 `sysctl` 名称、`/dev` 下的每个文件都是用 C 编写的内核代码的可见面孔。当你运行 `dmesg` 时，你在读取驱动程序在 attach 期间传递给 `device_printf` 或 `printf` 的字符串。当你遍历 `sysctl -a` 时，你在遍历驱动程序和子系统用 `SYSCTL_INT`、`SYSCTL_ULONG` 和相关宏填充的树。当你打开 `/dev/null` 时，内核将你的 `read` 或 `write` 分发到你将在第 6 章遇到的驱动程序结构。你一直在看驱动程序代码的输出；接下来两章教你编写输入。

第 5 章带你进入 **内核实际使用的 C 语言**：固定宽度整数类型、用 `malloc(9)` 和 `free(9)` 的显式内存管理、中断上下文下的指针纪律，以及 FreeBSD 的内核规范形式（KNF）认为惯用的 C 子集。这不只是"又是 C"。内核不能从 libc 调用 `printf`，不能用普通 `malloc` 分配，不能假设通常的用户空间安全网。第 5 章向你展示什么改变以及为什么，这样你稍后写的代码能够编译、加载并行为可预测。

然后第 6 章将你刚刚重学的 C 组装成第一个驱动程序，逐步剖析 **FreeBSD 驱动程序的解剖**：softc 结构、Newbus 方法表、`DRIVER_MODULE` 宏，以及 `read` 或 `write` 从 `/dev/foo` 设备节点到服务它的驱动程序例程所走的路径。到第 6 章结束时，你将加载自己的模块，看到它在 `dmesg` 中宣布自己，并使用本章练习的相同命令确认它工作。

在我们继续开始学习 C 编程之前，让我们停下来巩固你在本章学到的一切。下一节将回顾关键概念并给你一组练习挑战，这些练习将帮助巩固这些新技能并为你即将进行的工作做准备。

## 小结

恭喜你！你刚刚完成了**第一次 UNIX 和 FreeBSD 导览**。始于抽象理念的东西现在正在变成实践技能。你可以在系统中移动、管理文件、编辑和安装软件、控制进程、自动化任务，甚至窥探内核的内部运作。

让我们花点时间回顾你在本章完成的内容：

- **什么是 UNIX 以及为什么它很重要** - 一种简约、模块化和"一切皆文件"的哲学，由 FreeBSD 继承。
- **Shell** - 你通向系统的窗口，命令遵循 `command [options] [arguments]` 的一致结构。
- **文件系统布局** - 从 `/` 开始的单一层次结构，`/etc`、`/usr/local`、`/var` 和 `/dev` 等目录有特殊角色。
- **用户、组和权限** - FreeBSD 安全模型的基础，控制谁可以读、写或执行。
- **进程** - 运动中的程序，使用如 `ps`、`top` 和 `kill` 的工具管理它们。
- **安装软件** - 使用 `pkg` 快速二进制安装，使用 **Ports 集合**实现基于源的灵活性。
- **自动化** - 用 `cron` 调度任务，用 `at` 一次性调度，用 `periodic` 维护。
- **Shell 脚本** - 使用 FreeBSD 原生 `/bin/sh` 将重复命令转化为可重用程序。
- **窥探内核** - 使用 `dmesg`、`sysctl` 和 `/dev` 在更深层次上观察系统。

这很多，但如果你还没感觉像专家也不必担心。本章的目标不是完美，而是**舒适**：在 shell 中的舒适、探索 FreeBSD 的舒适、看到 UNIX 底层运作的舒适。这种舒适将贯穿我们开始为系统编写真正的代码时。

### 练习场

如果你想要实践的方式巩固刚才阅读的内容，接下来的页面收集了**46 个可选练习**。它们都不是继续本书所必需的，所以把它们当作额外的：挑选那些涵盖你仍感不确定的领域的，跳过那些感觉多余的，稍后如果你发现它们有用再回来。

它们按主题分组，所以你可以逐节练习或按你喜欢混合。

### 文件系统和导航（8 道练习）

1. 使用 `pwd` 确认你当前目录，然后用 `cd` 进入 `/etc` 再回到你的主目录。
2. 在你的主目录创建目录 `unix_playground`。在其中创建三个子目录：`docs`、`code` 和 `tmp`。
3. 在 `unix_playground/docs` 中，创建一个名为 `readme.txt` 的文件，内容为"Welcome to FreeBSD"。使用 `echo` 和输出重定向。
4. 将 `readme.txt` 复制到 `tmp` 目录。用 `ls -l` 验证两个文件都存在。
5. 将 `tmp` 中的文件重命名为 `copy.txt`。然后用 `rm` 删除它。
6. 使用 `find` 定位 `/etc` 内的所有 `.conf` 文件。
7. 使用绝对路径将 `/etc/hosts` 复制到你的 `docs` 目录。然后用相对路径将它移动到 `tmp`。
8. 使用 `ls -lh` 以人类可读格式显示文件大小。`/etc` 中哪个文件最大？

### 用户、组和权限（6 道练习）

1. 在你的主目录创建一个名为 `secret.txt` 的文件。使它只有你能读。
2. 创建一个目录 `shared` 并给所有人读写访问权限（模式 777）。通过写入文件来测试它。
3. 使用 `id` 列出你的用户的 UID、GID 和组。
4. 对 `/etc/passwd` 和 `/etc/master.passwd` 使用 `ls -l`。比较它们的权限并解释为什么它们不同。
5. 创建一个文件并用 `sudo chown` 将其所有者更改为 `root`。尝试作为普通用户编辑它。发生了什么？
6. 用 `sudo adduser` 添加一个新用户。设置密码，以该用户登录，检查他们的默认主目录。

### 进程和系统监控（7 道练习）

1. 用 `sleep 60` 在前台启动一个进程。在它运行时，打开另一个终端并用 `ps` 找到它。
2. 用 `sleep 60 &` 在后台启动同样的进程。用 `jobs` 和 `fg` 把它带回前台。
3. 使用 `top` 找到此刻消耗最多 CPU 的进程。
4. 启动一个 `yes` 进程（`yes > /dev/null &`）来淹没 CPU。在 `top` 中观察它，然后用 `kill` 停止它。
5. 用 `uptime` 检查你的系统已运行多久。
6. 用 `df -h` 查看系统上有多少磁盘空间可用。哪个文件系统挂载在 `/` 上？
7. 运行 `sysctl vm.stats.vm.v_page_count` 查看系统上的内存页数。

### 安装和管理软件（pkg 和 Ports）（6 道练习）

1. 使用 `pkg search` 查找除 `nano` 外的文本编辑器。安装它，运行它，然后删除它。
2. 用 `pkg` 安装 `htop` 包。将其输出与内置的 `top` 比较。
3. 通过导航到 `/usr/ports/editors/nano` 探索 Ports 集合。查看 Makefile。
4. 用 `sudo make install clean` 从 ports 构建 `nano`。它询问你关于选项了吗？
5. 用 `git` 更新你的 ports 树。哪些类别被更新了？
6. 用 `which` 定位 `nano` 或 `htop` 二进制文件安装的位置。检查它在 `/usr/bin` 还是 `/usr/local/bin` 下。

### 自动化和调度（cron, at, periodic）（6 道练习）

1. 写一个 cron 任务，每 2 分钟记录当前日期和时间到 `~/time.log`。等待并用 `tail` 检查。
2. 写一个 cron 任务，每天午夜清理主目录中的所有 `.tmp` 文件。
3. 用 `at` 命令调度 5 分钟后给自己发一条消息。
4. 运行 `sudo periodic daily` 并阅读其输出。它执行什么类型的任务？
5. 添加一个 cron 任务，每天早上 8 点运行 `df -h` 并将结果记录到 `~/disk.log`。
6. 将 cron 任务输出重定向到自定义日志文件（`~/cron_output.log`）。确认正常输出和错误都被捕获。

### Shell 脚本（/bin/sh）（7 道练习）

1. 写一个脚本 `hello_user.sh`，打印你的用户名、当前日期和正在运行的进程数。使其可执行并运行。
2. 写一个脚本 `organize.sh`，将主目录中的所有 `.txt` 文件移动到名为 `texts` 的文件夹中。添加注释解释每一步。
3. 修改 `organize.sh`，也按文件类型创建子目录（`images`、`docs`、`archives`）。
4. 写一个脚本 `disk_alert.sh`，当根文件系统使用超过 80% 时警告你。
5. 写一个脚本 `logger.sh`，将带时间戳的条目追加到 `~/activity.log`，包含当前目录和用户。
6. 写一个脚本 `backup.sh`，将 `~/unix_playground` 创建 `.tar.gz` 归档到 `~/backups/`。
7. 扩展 `backup.sh`，使其只保留最后 5 个备份并自动删除较旧的。

### 窥探内核（dmesg, sysctl, /dev）（6 道练习）

1. 使用 `dmesg` 找到你主磁盘的型号。
2. 使用 `sysctl hw.model` 显示你的 CPU 型号，用 `sysctl hw.ncpu` 显示你有几核。
3. 用 `sysctl kern.hostname=mytesthost` 暂时更改你的主机名。用 `hostname` 检查。
4. 使用 `ls /dev` 列出设备节点。识别哪些代表磁盘、终端和虚拟设备。
5. 使用 `head -c 16 /dev/random | hexdump` 从内核读取 16 个随机字节。
6. 插入 USB 棒（如果可用）并运行 `dmesg | tail`。你能看到出现了哪个新的 `/dev/` 条目吗？

### 小结

通过这 **46 道练习**，你涵盖了本章的每个主要主题：

- 文件系统导航和布局
- 用户、组和权限
- 进程和监控
- 使用 pkg 和 ports 的软件安装
- 使用 cron、at 和 periodic 的自动化
- 使用 FreeBSD 原生 `/bin/sh` 的 shell 脚本
- 使用 dmesg、sysctl 和 /dev 的内核内省

通过完成它们，你将从*被动读者*变成**主动 UNIX 实践者**。你不仅会知道 FreeBSD 如何工作，你已经在*其中生活过*。

这些练习是你开始编程时需要的**肌肉记忆**。当我们到达 C 语言，然后是内核开发时，你将已经精通 UNIX 开发者的日常工具。

### 展望未来

下一章将介绍 **C 编程语言**，FreeBSD 内核的语言。这是你用来创建设备驱动程序的工具。如果你从未编过程序也不必担心，我们将像本章的 UNIX 一样逐步建立你的理解。

通过将你的新 UNIX 素养与 C 编程技能结合，你将准备好开始塑造 FreeBSD 内核本身。
