---
title: "使用设备文件"
description: "devfs、cdev 和设备节点如何为你的驱动提供一个安全、规整的用户界面。"
partNumber: 2
partName: "构建你的第一个驱动"
chapter: 8
lastUpdated: "2026-04-17"
status: "complete"
author: "Edson Brandi"
reviewer: "TBD"
translator: "AI辅助翻译为简体中文"
language: "zh-CN"
estimatedReadTime: 210
---

# 使用设备文件

## 读者指南与学习目标

在第七章中，你构建了 `myfirst`——一个真正的 FreeBSD 驱动程序，它能够干净地挂载、创建 `/dev/myfirst0`、打开和关闭该节点，并且在卸载时没有资源泄漏。这是第一个胜利，而且是真正的胜利。现在你的磁盘上已经有了一个可用的驱动骨架——一个内核会接受并根据你的命令释放的 `.ko` 文件，以及一个用户程序可以访问的 `/dev` 条目。

本章聚焦于这项工作中最容易被视为理所当然的部分：**设备文件本身**。第七章中创建 `/dev/myfirst0` 的那行代码很简洁，但它建立在一个名为 **devfs** 的子系统之上，而这个子系统是你的驱动在内核中所做的一切与用户指向它的每个工具或程序之间的桥梁。现在深入理解这座桥梁，将使第九章和第十章（开始真正传输数据的地方）变得不那么神秘。

### 为什么本章值得单独设立

第六章在概念层面介绍了设备文件模型，第七章则使用了足够多的相关内容来让驱动运行起来。这两章都没有停下来仔细审视这个界面本身。这不是疏忽。在一本从基本原理教授驱动编写的书中，设备文件值得专门用一章来讲解，因为在这个界面上犯的错误也是日后最难弥补的错误。

考虑一下这个界面需要承载什么。它承载身份（用户程序可以预测的路径）。它承载访问策略（谁被允许打开、读取或写入）。它承载多路复用（一个驱动、多个实例、多个同时打开者）。它承载生命周期（节点何时出现、何时消失，以及当用户程序正在调用时它消失了会发生什么）。它承载兼容性（传统名称与现代名称并存）。它承载可观测性（操作员可以从用户态看到和更改什么）。一个内部实现正确但界面设计错误的驱动，将会是一个操作员拒绝部署的驱动、安全审查人员标记的驱动、在 jail 中行为异常的驱动、在现实负载下卸载时死锁的驱动。

第七章给了你刚好足够的界面来证明路径可行。本章则给了你足够的内容来有目的地设计它。

### 第七章结束时驱动的状态

在扩展驱动之前，回顾一下 `myfirst` 的当前状态是值得的。你的第七章驱动最终具备了以下所有内容：

- 一条 `device_identify`、`device_probe` 和 `device_attach` 路径，创建了恰好一个 `nexus0` 的 Newbus 子设备，名为 `myfirst0`。
- 一个由 Newbus 分配的 softc，可通过 `device_get_softc(dev)` 访问。
- 一个互斥锁、一个位于 `dev.myfirst.0.stats` 下的 sysctl 树，以及三个只读计数器。
- 一个填充了 `d_open`、`d_close` 以及 `d_read` 和 `d_write` 桩函数的 `struct cdevsw`。
- 一个在 `attach` 中使用 `make_dev_s(9)` 创建并在 `detach` 中使用 `destroy_dev(9)` 移除的 `/dev/myfirst0` 节点。
- 一个单标签错误展开路径，在任何挂载步骤失败时使内核保持一致状态。
- 一个独占打开策略，用 `EBUSY` 拒绝第二次 `open(2)`。

第八章将此驱动作为起点，沿三个维度进行扩展：**形态**（节点叫什么以及如何分组）、**策略**（谁被允许使用它以及该策略如何在重启后保持）、以及**每描述符状态**（两个同时打开者如何拥有各自独立的簿记）。

### 你将学到什么

到本章结束时，你将能够：

- 解释 FreeBSD 中的设备文件是什么，以及为什么 `/dev` 不是一个普通目录。
- 描述 `struct cdev`、devfs vnode 和用户文件描述符之间的关系。
- 为新设备节点选择合理的所有权、组和权限值。
- 给设备节点一个结构化的名称（包括 `/dev` 下的子目录）。
- 创建别名，使单个 cdev 可以通过多个路径访问。
- 使用 `devfs_set_cdevpriv()` 附加每次打开的状态，并在文件描述符关闭时安全地清理它。
- 从用户态使用 `devfs.conf` 和 `devfs.rules` 持久地调整设备节点权限。
- 使用小型用户态 C 程序测试你的驱动，而不仅仅是使用 `cat` 和 `echo`。

### 你将构建什么

你将通过三个小步骤扩展第七章的 `myfirst` 驱动：

1. **阶段 0：更整洁的权限和结构化名称。** 节点从 `/dev/myfirst0` 移动到 `/dev/myfirst/0`，并带有用于实验室环境的组可访问变体。
2. **阶段 1：用户可见的别名。** 你添加 `/dev/myfirst` 作为 `/dev/myfirst/0` 的别名，使旧路径继续工作。
3. **阶段 2：每次打开的状态。** 每次 `open(2)` 通过 `devfs_set_cdevpriv()` 获得自己的小型计数器，你从用户态验证两个同时打开看到的是独立的值。

你还将编写一个简短的用户态程序 `probe_myfirst.c`，它打开设备、读取一些数据、报告看到的内容并干净地关闭。这个程序将在第九章实现真正的 `read(2)` 和 `write(2)` 路径时再次使用。

### 本章不涵盖的内容

有几个与 `/dev` 相关的主题被刻意推迟了：

- **完整的 `read` 和 `write` 语义。** 第七章将这些留作桩函数；第九章将使用 `uiomove(9)` 正确实现它们。本章仅为后续工作做准备。
- **克隆设备**（`clone_create`、`dev_clone` 事件处理器）。一旦基本模型扎实，这些值得后续仔细研究。
- **`ioctl(2)` 设计。** 通过 `ioctl` 检查和更改设备状态是一个独立的话题，属于书的后续部分。
- **GEOM 和存储设备。** GEOM 构建在 cdev 之上，但增加了自己的一整套栈。这属于第六部分。
- **网络接口节点和 `ifnet`。** 网络驱动不位于 `/dev` 之下。它们通过不同的界面出现，我们将在第六部分遇到。

保持范围紧凑正是重点。设备的界面很小；围绕它的纪律应该是你首先掌握的东西。

### 预计时间投入

- **仅阅读：** 约 30 分钟。
- **阅读加上对 `myfirst` 的代码更改：** 约 90 分钟。
- **阅读加上全部四个实验：** 两到三个小时，包括重建周期和用户态测试。

带有休息的稳定进度效果最好。本章比第七章短，但这里的理念几乎会出现在你将来阅读的每个驱动中。

### 前提条件

- 一个可用的第七章 `myfirst` 驱动，能够加载、挂载和干净地卸载。
- 实验环境中的 FreeBSD 14.3 及匹配的 `/usr/src`。
- 基本能阅读 `/usr/src` 路径下的文件，如 `/usr/src/sys/dev/null/null.c`。

### 如何从本章获得最大收益

在阅读本章的同时打开第七章的源代码并编辑同一个文件。你不是在开始一个新项目；你是在扩展已有的项目。当本章要求你检查某个 FreeBSD 文件时，真的用 `less` 打开它并滚动浏览。当你看过几个真正的驱动如何塑造它们的节点之后，设备文件模型的理解会快得多。

一个立刻见效的实用习惯：阅读时，在第二个终端中保持连接到刚启动的实验系统，用 `ls -l` 或 `stat(1)` 确认每个关于现有节点的说法。输入 `ls -l /dev/null` 并看到输出与文字描述匹配，这虽然微小，但它将抽象锚定在你可见的东西上。到本章进行到实验部分时，你将自然而然地去验证每个说法与运行中的内核是否一致，而不是仅仅信任文本。

第二个习惯：当本章提到 `/usr/src` 下的某个源文件时，与当前章节并排打开它。真正的 FreeBSD 驱动就是教科书；本书只是阅读指南。`/usr/src/sys/dev/null/null.c` 和 `/usr/src/sys/dev/led/led.c` 中的内容足够简短，几分钟就能浏览完，而每一个都由本章即将解释的决策所塑造。那里的一次简短浏览比这里任何数量的文字都更有价值。

### 本章路线图

如果你想把本章看作一条连续的线索，这里是它。各节按顺序排列：

1. 设备文件到底是什么，在理论和 `ls -l` 实践中。
2. devfs——`/dev` 背后的文件系统——是如何诞生以及它为你做了什么。
3. 排在设备文件后面的三个内核对象。
4. 所有权、组和模式如何塑造 `ls -l` 的输出以及谁能打开节点。
5. 名称如何被选择，包括单元号和子目录。
6. 一个 cdev 如何通过别名响应多个名称。
7. 每次打开的状态如何被注册、检索和清理。
8. 调用 `destroy_dev(9)` 后析构函数真正如何工作。
9. `devfs.conf` 和 `devfs.rules` 如何从用户态塑造策略。
10. 如何用你自己编写的小型用户态程序驱动设备。
11. 真正的 FreeBSD 驱动如何解决这些相同的问题。
12. 你的 `d_open` 应该返回哪些 errno 值，以及在什么时候。
13. 当这个界面上的某些东西看起来不对时应该使用哪些工具。
14. 四到八个实验，带你亲手实践每种模式。
15. 将模式延伸到现实场景的挑战练习。
16. 一份避免你措手不及的陷阱指南。

如果你是第一次阅读，请从头到尾跟着章节走。如果你在复习，可以单独阅读每一节；结构被设计为完整的概览，而不仅仅是线性教程。


## cdev、vnode 和文件描述符

打开一个设备文件，三个内核侧对象会悄悄地排列在你的文件描述符后面。理解这个三元组是写出恰好能工作的驱动和真正掌控其生命周期的驱动之间的区别。

第一个对象是 `struct cdev`——**设备的内核侧标识**。每个设备节点有一个 `struct cdev`，无论有多少程序打开了它。你的驱动用 `make_dev_s(9)` 创建它，用 `destroy_dev(9)` 销毁它。cdev 承载着节点的标识信息：它的名称、所有者、模式、调度系统调用的 `struct cdevsw`，以及两个驱动控制的槽位 `si_drv1` 和 `si_drv2`。第七章已经用 `si_drv1` 存储 softc 指针，这是它最常见用途。

第二个对象是一个 **devfs vnode**。vnode 是表示打开文件系统 inode 的通用 FreeBSD VFS 对象。每个设备节点在其下面有一个 vnode，就像普通文件一样，VFS 层使用 vnode 将操作路由到正确的文件系统。对于设备节点，那个文件系统是 devfs，而 devfs 将操作转发给 cdev。

第三个对象是**文件描述符**本身，在内核内部由 `struct file` 表示。与 cdev 不同，每个打开有一个 `struct file`，而不是每个设备一个。这是每次打开状态所在的地方。两个都打开 `/dev/myfirst0` 的进程共享同一个 cdev 但获得独立的文件结构，devfs 知道如何干净地保持这些结构独立。

将三者放在一起，单个 `read(2)` 的路径看起来像这样：

```text
user process
   read(fd, buf, n)
         |
         v
 file descriptor (struct file)  per-open state
         |
         v
 devfs vnode                     VFS routing
         |
         v
 struct cdev                     device identity
         |
         v
 cdevsw->d_read                  your driver's handler
```

上面的每个框独立存在，每个都有不同的生命周期。cdev 存活时间与你的驱动保持它存活的时间相同。vnode 存活时间与任何人在 VFS 层解析了该节点的时间相同。`struct file` 存活时间与用户进程保持其描述符打开的时间相同。当你编写驱动时，你只填充该图的最后一行，但了解上面的行非常有帮助。

### 追踪单个 read(2) 通过栈的过程

用具体的 `read(2)` 调用作为锚点，用散文形式走一遍故事。用户程序有这行代码：

```c
ssize_t n = read(fd, buf, 64);
```

以下是发生的事情。内核接收 `read(2)` 系统调用，在调用进程的文件描述符表中查找 `fd`。这产生一个 `struct file`。内核看到文件的类型是 vnode 支持的文件，其 vnode 位于 devfs 中，所以它通过通用文件操作向量分派到 devfs 的读取处理程序。

devfs 获取对底层 `struct cdev` 的引用，从中检索 `struct cdevsw` 的指针，并调用 `cdevsw->d_read`。那就是**你的**函数。在其中，你检查内核准备的 `struct uio`，通过 `struct cdev *dev` 参数查看设备，并可选择用 `devfs_get_cdevpriv` 恢复每次打开的结构。当你返回时，devfs 释放其对 cdev 的引用，读取调用展开回用户程序。

从这个追踪中得出几个值得记住的不变量：

- **如果 cdev 已经消失，你的处理程序永远不会运行。** 在 `destroy_dev(9)` 撤回节点和最后一个调用者放弃其引用之间，devfs 简单地拒绝新操作。
- **来自两个进程的两个调用可以同时到达 `d_read`。** devfs 和 VFS 层都不会代表你序列化调用者。并发控制是你的责任，本书第三部分专门讨论它。
- **你隐式服务的 `struct file` 对你的处理程序是隐藏的。** 你不需要知道哪个描述符触发了调用；你只需要 cdev、uio 和（可选的）cdevpriv 指针。

最后一点在实践中非常有用。通过向处理程序隐藏描述符，FreeBSD 给你一个干净的 API：所有每次描述符的簿记都通过 `devfs_set_cdevpriv` 和 `devfs_get_cdevpriv` 进行，你的处理程序代码保持简小。

### 为什么这对初学者很重要

这个模型产生两个实际后果，两者都会在下一章再次出现。

首先，**存储在 cdev 上的指针在所有打开之间共享**。如果你在 `si_drv1` 中存储一个计数器，打开节点的每个进程都会看到同一个计数器。这对驱动范围的状态（如 softc）非常完美，对每次会话的状态（如读取位置）则很糟糕。

其次，**内核不关心你的设备被打开了多少次**。除非你另行告知，每次 `open(2)` 都会直接通过。如果你需要独占访问（如第七章代码通过其 `is_open` 标志所做的那样），你必须自己强制执行。如果你需要每次打开的簿记，你将该簿记附加到文件描述符，而不是 cdev。我们将在本章结束前完成这两件事。

### 深入了解 struct cdev

你在第七章整章中一直通过指针使用 `struct cdev`。现在是时候查看其内部了。完整定义在 `/usr/src/sys/sys/conf.h` 中，重要字段如下：

```c
struct cdev {
        void            *si_spare0;
        u_int            si_flags;
        struct timespec  si_atime, si_ctime, si_mtime;
        uid_t            si_uid;
        gid_t            si_gid;
        mode_t           si_mode;
        struct ucred    *si_cred;
        int              si_drv0;
        int              si_refcount;
        LIST_ENTRY(cdev) si_list;
        LIST_ENTRY(cdev) si_clone;
        LIST_HEAD(, cdev) si_children;
        LIST_ENTRY(cdev) si_siblings;
        struct cdev     *si_parent;
        struct mount    *si_mountpt;
        void            *si_drv1, *si_drv2;
        struct cdevsw   *si_devsw;
        int              si_iosize_max;
        u_long           si_usecount;
        u_long           si_threadcount;
        union { ... }    __si_u;
        char             si_name[SPECNAMELEN + 1];
};
```

并非每个字段对初学者级别的驱动都很重要。有几个是重要的，了解它们代表什么可以在你第一次看陌生代码时节省数小时。

**`si_name`** 是 devfs 看到的节点的以 null 结尾的名称。当你向 `make_dev_s` 传递 `"myfirst/%d"` 和单元 `0` 时，这就是最终包含字符串 `myfirst/0` 的字段。辅助函数 `devtoname(struct cdev *dev)` 返回指向此字段的指针，是日志记录或调试输出的正确工具。

**`si_flags`** 是承载 cdev 状态标志的位字段。你的驱动最常接触的标志是 `SI_NAMED`（当 `make_dev*` 已将节点放入 devfs 时设置）和 `SI_ALIAS`（在通过 `make_dev_alias` 创建的别名上设置）。内核管理它们；你的代码很少（如果有的话）直接写入此字段。一个有用的阅读习惯：如果你在其他人的驱动中看到陌生的 `SI_*` 标志，在 `/usr/src/sys/sys/conf.h` 中查找它并阅读单行注释。

**`si_drv1`** 和 **`si_drv2`** 是两个通用的驱动控制槽位。第七章使用 `si_drv1` 存储 softc 指针，这是最常见的模式。当你需要时，`si_drv2` 可用于第二个指针。这些字段供你使用；内核从不触及它们。

**`si_devsw`** 是指向调度此 cdev 上操作的 `struct cdevsw` 的指针。它是节点与你的处理程序之间的链接。

**`si_uid`**、**`si_gid`**、**`si_mode`** 保存公布的所有权和模式。它们从你传递给 `make_dev_args_init` 的 `mda_uid`、`mda_gid`、`mda_mode` 参数设置。它们原则上可变，但更改它们的正确方式是通过 `devfs.conf` 或 `devfs.rules`，而不是直接赋值到结构中。

**`si_refcount`**、**`si_usecount`**、**`si_threadcount`** 是 devfs 用来在任何可能触及 cdev 时保持其存活的三个计数器。`si_refcount` 计算长期引用（cdev 列在 devfs 中，其他 cdev 可能对其设置别名）。`si_usecount` 计算活跃的用户空间文件描述符已打开此 cdev 的数量。`si_threadcount` 计算当前在此 cdev 的 `cdevsw` 处理程序内执行的内核线程数。你的驱动几乎从不直接读取这些；例程 `dev_ref`、`dev_rel`、`dev_refthread` 和 `dev_relthread` 代表你管理它们。概念上重要的是 `destroy_dev(9)` 将拒绝完成拆除 cdev 直到 `si_threadcount` 降至零；它会等待，短暂睡眠，直到每个进行中的处理程序都返回。

**`si_parent`** 和 **`si_children`** 将 cdev 链接到父子关系中。这就是 `make_dev_alias(9)` 如何将别名 cdev 连接到其主设备，以及某些克隆机制如何将每次打开的节点连接到其模板。大多数时候你不会与这些字段交互；知道它们存在并且是 devfs 能在主设备销毁时干净地展开别名的原因之一就足够了。

**`si_flags & SI_ETERNAL`** 值得简短说明。一些节点，特别是 `null` 为 `/dev/null`、`/dev/zero` 和 `/dev/full` 创建的那些，用 `MAKEDEV_ETERNAL_KLD` 标记为永恒。内核拒绝在正常操作期间销毁它们。当你开始编写在 KLD 加载时暴露设备并希望节点在卸载尝试期间保持存活的模块时，这就是那个开关。对于正在积极开发的驱动，不要动它。

### struct cdevsw：调度表

你的第七章 `cdevsw` 填充了少量字段。真正的结构更长，其余字段至少值得认识一下，因为你会在真正的驱动中遇到它们，迟早会想使用其中一些。

该结构在 `/usr/src/sys/sys/conf.h` 中定义为：

```c
struct cdevsw {
        int              d_version;
        u_int            d_flags;
        const char      *d_name;
        d_open_t        *d_open;
        d_fdopen_t      *d_fdopen;
        d_close_t       *d_close;
        d_read_t        *d_read;
        d_write_t       *d_write;
        d_ioctl_t       *d_ioctl;
        d_poll_t        *d_poll;
        d_mmap_t        *d_mmap;
        d_strategy_t    *d_strategy;
        void            *d_spare0;
        d_kqfilter_t    *d_kqfilter;
        d_purge_t       *d_purge;
        d_mmap_single_t *d_mmap_single;
        /* fields managed by the kernel, not touched by drivers */
};
```

逐一查看各字段。

**`d_version`** 是 ABI 标记。必须设置为 `D_VERSION`，一个在结构上方几行定义的值。内核在注册 cdevsw 时检查此字段，如果标记不匹配将拒绝继续。忘记设置它是一个经典的初学者错误：驱动编译、加载，然后在第一次打开时产生奇怪的错误或直接崩溃系统。始终在你编写的每个 `cdevsw` 中将 `d_version = D_VERSION` 设为第一个字段。

**`d_flags`** 承载一组 cdevsw 范围的标志。标志名称与其余结构一起定义。现在值得识别的有：

- `D_TAPE`、`D_DISK`、`D_TTY`、`D_MEM`：向内核提示设备的性质。对于大多数驱动，你将其保留为零。
- `D_TRACKCLOSE`：如果设置，devfs 对描述符上的每次 `close(2)` 都调用你的 `d_close`，而不仅仅是最后一次关闭。当你想在面对 `dup(2)` 时仍然可靠地运行每次描述符的清理时很有用。
- `D_MMAP_ANON`：匿名内存映射的特殊处理。`/dev/zero` 设置了这个，这就是 `mmap(..., /dev/zero, ...)` 产生零填充页面的方式。
- `D_NEEDGIANT`：强制在 Giant 锁下调度此 cdevsw 的处理程序。现代驱动不应该需要这个；如果你在代码中看到它，将其视为历史标记而不是要遵循的模型。
- `D_NEEDMINOR`：表示驱动使用 `clone_create` 为克隆的 cdev 分配次设备号。你在第八章不需要这个。

**`d_name`** 是内核在记录此 cdevsw 时使用的基名字符串。它也成为 `clone_create(9)` 机制在合成克隆设备时使用的模式的一部分。将其设置为简短、人类可读的字符串，如 `"myfirst"`。

**`d_open`**、**`d_close`**：会话边界。当用户程序在节点上调用 `open(2)` 或用 `close(2)` 释放其最后一个描述符时调用。第七章介绍了两者，本章完善你如何使用它们。

**`d_fdopen`**：对于想要直接传递 `struct file *` 的驱动的 `d_open` 替代方案。在初学者级别的驱动中很少见。除非未来的章节介绍它，否则忽略。

**`d_read`**、**`d_write`**：字节流操作。第七章将这些留作桩函数。第九章将用 `uiomove(9)` 实现它们。

**`d_ioctl`**：控制路径操作。第二十五章将深入讨论 `ioctl` 设计。现在，识别该字段并知道这是来自 `ioctl(2)` 的结构化命令落地的地方。

**`d_poll`**：由 `poll(2)` 调用以询问设备当前是否可读或可写。第十章作为 I/O 效率故事的一部分处理这个。

**`d_kqfilter`**：由 `kqueue(9)` 机制调用。同一章。

**`d_mmap`**、**`d_mmap_single`**：支持将设备映射到用户进程的地址空间。在初学者驱动中很少见，以后相关时再讨论。

**`d_strategy`**：由某些内核层（特别是旧的 `physio(9)` 路径）调用，将 I/O 块作为 `struct bio` 交给驱动。与你将在第二部分编写的伪设备无关。

**`d_purge`**：在销毁期间如果 cdev 的处理程序中仍有线程在运行，则由 devfs 调用。编写良好的 `d_purge` 会唤醒那些线程并说服它们快速返回以便销毁可以继续。大多数简单驱动不需要；第十章将在阻塞 I/O 的上下文中重新讨论这个。

当你设计自己的 cdevsw 时，你只填充对应你的设备实际支持的操作的字段。每个 `NULL` 字段是一个礼貌的拒绝：内核将其解释为"此设备不支持此操作"或"使用默认行为"，具体取决于哪个操作。不要触及备用字段。

### D_VERSION 标记及其存在原因

关于 `d_version` 的简短说明很有用，因为它会在你的驱动首次神秘注册失败时节省你的时间。

cdevsw 结构的内核接口在 FreeBSD 的生命周期中已经演变。字段在主要版本之间被添加、删除或更改类型。`d_version` 标记是内核确认你的模块是针对兼容的结构定义构建的方式。设置它的规范方式是：

```c
static struct cdevsw myfirst_cdevsw = {
        .d_version = D_VERSION,
        /* ...remaining fields... */
};
```

宏 `D_VERSION` 在 `/usr/src/sys/sys/conf.h` 中定义，每当结构以破坏 ABI 的方式更改时，内核团队都会更新它。针对新头文件构建的模块获得新标记。针对旧头文件构建的模块获得旧标记，内核会拒绝它们。

这是一个节省大量麻烦的小细节。每次都设置它。如果你曾经看到内核在加载时打印 cdevsw 版本不匹配，你的构建环境和运行内核已经漂移；针对你打算运行的内核的头文件重新构建模块。


### cdev 层面的引用计数

你在 `struct cdev` 上看到的计数器是使设备销毁安全的引擎。一个简单的图景方式：

- `si_refcount` 是"内核中仍有多少东西按住这个 cdev 的脖子"的计数。别名、克隆和某些簿记路径会递增它。当此值非零时，cdev 实际上无法被释放。
- `si_usecount` 是"有多少用户空间文件描述符打开了这个 cdev"的计数。它由 devfs 在成功的 `open(2)` 时递增，在 `close(2)` 时递减。你的驱动从不直接触及它。
- `si_threadcount` 是"现在有多少内核线程正在我的 `cdevsw` 处理程序之一内部执行"的计数。它由 `dev_refthread(9)` 在 devfs 代表你进入处理程序时递增，由 `dev_relthread(9)` 在处理程序返回时递减。你的驱动从不直接触及它。

使这可用的规则是：`destroy_dev(9)` 将阻塞直到 `si_threadcount` 降至零，并且在没有更多处理程序可以进入此 cdev 之前不会返回。这就是 `destroy_dev` 如何能够保证在它返回后，你的处理程序不会再被调用。本章后面标题为"安全销毁 cdev"的小节将重新讨论这个保证以及你需要其更强的兄弟 `destroy_dev_drain(9)` 的情况。

### 生命周期的再一轮

有了这个，上一小节的图比第一次有了更多意义。cdev 是一个生命周期受你的驱动控制的长期内核对象。vnode 是一个 VFS 层对象，只在文件系统层需要它时存活。`struct file` 是一个短期的每次打开对象，只在进程保持描述符打开时存活。在这三者之下，上面描述的计数器保持它们诚实。

你不需要记住任何这些。你需要识别的是形状。当你以后阅读驱动并看到 `dev_refthread` 或 `si_refcount` 时，你会记得它们是做什么的。当你观看 `destroy_dev` 在调试器中睡眠时，你会认出它正在等待 `si_threadcount` 下降。那种识别是将内核代码从谜题变成你可以推理的东西的关键。

## 权限、所有权和模式

当你的驱动调用 `make_dev_s(9)` 时，`struct make_dev_args` 上的三个字段决定了 `ls -l /dev/yournode` 将显示什么：

```c
args.mda_uid  = UID_ROOT;
args.mda_gid  = GID_WHEEL;
args.mda_mode = 0600;
```

`UID_ROOT`、`UID_BIN`、`UID_UUCP`、`UID_NOBODY`、`GID_WHEEL`、`GID_KMEM`、`GID_TTY`、`GID_OPERATOR`、`GID_DIALER` 以及少量相关名称在 `/usr/src/sys/sys/conf.h` 中定义。当存在知名身份时使用这些常量而不是原始数字。这使你的驱动更易读，并保护你免受数值漂移的静默影响。

模式是经典的 UNIX 权限三元组。每个位的含义与普通文件相同，但设备不关心执行位。有几种组合经常出现：

- `0600`：所有者读写。对于仍在开发中的驱动来说是最安全的默认值。
- `0660`：所有者和组读写。当你有定义良好的特权组（如 `operator` 或 `dialer`）时合适。
- `0644`：所有者读写，所有人可读。对于控制设备很少见，有时适用于只读状态或随机字节风格的节点。
- `0666`：所有人可读写。仅用于像 `/dev/null` 和 `/dev/zero` 这样故意无害的源。除非有真正的理由，否则不要使用这个。

经验法则很简单：问"谁实际需要触及这个节点？"并编码那个答案，不要更多。以后放宽权限很容易。在用户已经依赖更宽松的模式后收紧权限则不容易。

### 模式从何而来

值得明确说明谁决定节点上的最终模式。三个参与者有发言权：

1. **你的驱动**，通过 `make_dev_s()` 时的 `mda_uid`、`mda_gid` 和 `mda_mode` 字段。这是基线。
2. **`/etc/devfs.conf`**，可以在节点出现时应用一次性静态调整。这是操作员收紧或放宽特定路径权限的标准方式。
3. **`/etc/devfs.rules`**，可以应用基于规则的调整，通常用于过滤 jail 看到的内容。

如果驱动设置 `0600` 且没有配置其他内容，你会看到 `0600`。如果驱动设置 `0600` 且 `devfs.conf` 说 `perm myfirst/0 0660`，你会看到该节点的 `0660`。内核是机制；操作员的配置是策略。

### 你将遇到的命名组

FreeBSD 附带一小组出现在设备所有权中的知名组。每个都在 `/usr/src/sys/sys/conf.h` 中有匹配的常量。一份简要指南帮助你快速选择：

- **`GID_WHEEL`** (`wheel`)。受信任的管理员。当你不确定除了 root 谁应该有访问权时最安全的默认值。
- **`GID_OPERATOR`** (`operator`)。运行操作工具但不是完全管理员的用户。通常用于需要人工监督但不应每次都要求 `sudo` 的设备。
- **`GID_DIALER`** (`dialer`)。历史上用于串口拨出访问。仍用于用户空间拨号程序需要的 TTY 节点。
- **`GID_KMEM`** (`kmem`)。通过 `/dev/kmem` 风格的节点读取内核内存。非常敏感，很少是新驱动的正确选择。
- **`GID_TTY`** (`tty`)。终端设备的所有权。

当存在合适的命名组时，使用它。当没有合适的时，将组保留为 `wheel` 并通过 `devfs.conf` 为需要自己分组的站点添加条目。在驱动内部发明全新的组几乎从来都不值得。

### 一个实际示例

假设驱动基线是 `UID_ROOT`、`GID_WHEEL`、`0600`，你想让特定的实验室用户通过受控的组进行读写。顺序如下。

加载驱动后没有 `devfs.conf` 条目：

```sh
% ls -l /dev/myfirst/0
crw-------  1 root  wheel     0x5a Apr 17 10:02 /dev/myfirst/0
```

向 `/etc/devfs.conf` 添加一个部分：

```text
own     myfirst/0       root:operator
perm    myfirst/0       0660
```

应用并再次检查：

```sh
% sudo service devfs restart
% ls -l /dev/myfirst/0
crw-rw----  1 root  operator  0x5a Apr 17 10:02 /dev/myfirst/0
```

驱动没有重新加载。内核中的 cdev 是同一个对象。只有公布的所有权和模式改变了，它们改变是因为策略文件告诉 devfs 改变它们。这就是你想要的分层：驱动发布一个合理的基线，操作员塑造视图。

### 树中的案例研究

花一分钟了解真正的 FreeBSD 设备公布的权限是值得的，因为这些驱动做出的选择不是偶然的。每一个都是小型设计决策，每一个都与该类节点的威胁模型一致。

`/dev/null` 和 `/dev/zero` 发布时模式为 `0666`、`root:wheel`。系统上的每个人，无论是否有特权，都被允许打开它们并通过它们读取或写入。这是正确的选择，因为它们携带的数据可以无限量供应（零字节输出，字节被处理掉输入，没有硬件状态，没有秘密）。使它们更紧凑会破坏大量依赖其普遍可用的脚本、工具和编程习惯。创建它们的代码在 `/usr/src/sys/dev/null/null.c` 中，`make_dev_credf(9)` 的参数值得一看。

`/dev/random` 通常是模式 `0644`，任何人可读，仅 root 可写。读取访问故意广泛，因为许多用户态程序需要熵。写入访问狭窄，因为馈送熵池是特权操作。

`/dev/mem` 和 `/dev/kmem` 历史上是模式 `0640`，所有者 `root`，组 `kmem`。该组的存在正是为了让特权监控工具可以链接到它们而无需以 root 运行。模式紧凑，因为节点暴露原始内存；一个随意可读的 `/dev/mem` 将是灾难。如果你曾经看到驱动为一个承载硬件状态或内核内存的节点默认使用这么宽松的模式，将其视为缺陷。

`/dev/pf`——数据包过滤器的控制节点——是模式 `0600`、所有者 `root`、组 `wheel`。可以写入 `/dev/pf` 的用户可以更改防火墙规则。没有可接受的更宽松模式；接口的全部意义就是集中特权网络配置，任何更宽松的东西都会把防火墙变成自由争夺。

`/dev/bpf*`——Berkeley 数据包过滤器节点——是模式 `0600`、所有者 `root`、组 `wheel`。`/dev/bpf*` 的读取者可以看到附加接口上的每个数据包。那是不折不扣的特权，权限反映了这一点。

`/dev/ttyu*` 等硬件串口界面下的 TTY 节点通常是模式 `0660`、所有者 `uucp`、组 `dialer`。`dialer` 组存在是为了让一组受信任的用户可以运行拨出程序而无需 `sudo`。权限集是让预期工作流程发挥作用所需的最窄范围。

模式很容易命名：**FreeBSD 的基本系统从不选择宽松的设备权限，除非另一边的数据无害**。当你设计自己的节点时，使用该模式作为心智检查。如果你的节点携带可能伤害某人的数据，收紧模式。如果它携带可轻易重新生成和可轻易丢弃的数据，放宽是可辩护的；只有在有理由时才这样做。

### 应用于设备文件的最小权限

"最小权限"是一个过度使用的短语，但当它应用于设备文件时恰恰正确。作为驱动作者，你正在选择谁可以从用户态与你的代码对话，你可以设定下限。你做得比必要更宽的每个选择都是日后邀请错误的选择。

每个新节点的实用清单：

1. **用一句话命名主要消费者。** "监控守护进程每秒读取一次状态。""控制工具调用 ioctl 推送配置。""operator 组的用户可以读取原始数据包计数器。"如果你无法命名消费者，你就无法设置权限；你在猜测。
2. **从句子推导模式。** 以 `root:wheel` 身份运行并每秒读取一次的监控守护进程想要 `0600`。特权管理员子集运行的控制工具想要带有专用组的 `0660`。非特权仪表板消费的只读状态节点想要 `0644`。
3. **在 `mda_mode` 行旁边的注释中放入推理。** 未来的维护者会感谢你。未来的审计人员会更感谢你。
4. **默认使用 `UID_ROOT`。** 除非驱动显式建模非 root 守护进程身份，否则驱动创建的节点的所有者几乎没有理由是其他任何东西。

本书想要让你避免的相反习惯是"先打开以后再收紧"的冲动。已发布驱动上的权限很难收紧，因为到有人注意到时，某个用户的工作流依赖于宽松模式，收紧会破坏他们的一天。从紧凑开始。在你审查了真正请求后再放宽。

### 从宽松模式过渡到紧凑模式

偶尔你会继承一个完全开放的驱动并需要收紧它。正确的方法是三阶段：

**阶段 1：公告。** 在发行说明中、在驱动的首次挂载时的内核日志中、在你项目使用的任何面向操作员的渠道中公布计划更改。邀请至少一个发布周期的反馈。

**阶段 2：提供过渡路径。** 要么是一个为需要它的人重新打开旧模式的 `devfs.conf` 条目，要么是驱动在挂载时读取以选择其默认模式的 sysctl。重要的属性是，有合法需要保持旧模式的站点可以在不 fork 驱动的情况下这样做。

**阶段 3：翻转默认值。** 在过渡窗口结束后的下一个版本中，将驱动自己的 `mda_mode` 更改为更窄的值。`devfs.conf` 逃生舱保留给需要它的站点；其他人都获得更窄的默认值。

这些都不是 FreeBSD 特有的；这是任何管理良好的项目处理向后不兼容接口更改的方式。这里值得命名是因为设备文件权限恰好具有此属性：它们是你驱动公共接口的一部分。

### uid 和 gid 常量实际上是什么

`/usr/src/sys/sys/conf.h` 中定义的 `UID_*` 和 `GID_*` 常量**不**保证在每个系统上匹配用户和组数据库。头文件中选择的名称对应于 FreeBSD 基本系统在 `/etc/passwd` 和 `/etc/group` 中保留的身份，但本地修改的系统理论上可以重新编号它们，或者基于 FreeBSD 构建的产品可以添加自己的。实际上，在你将接触的每个 FreeBSD 系统上，常量都是匹配的。

要保持的纪律很简单：当存在符号名称时使用它，在发明新身份之前在头文件中查找。头文件当前至少定义了这些：

- 用户 ID：`UID_ROOT` (0)、`UID_BIN` (3)、`UID_UUCP` (66)、`UID_NOBODY` (65534)。
- 组 ID：`GID_WHEEL` (0)、`GID_KMEM` (2)、`GID_TTY` (4)、`GID_OPERATOR` (5)、`GID_BIN` (7)、`GID_GAMES` (13)、`GID_VIDEO` (44)、`GID_RT_PRIO` (47)、`GID_ID_PRIO` (48)、`GID_DIALER` (68)、`GID_NOGROUP` (65533)、`GID_NOBODY` (65534)。

如果你需要一个不在列表中的身份，基本系统可能没有保留一个。在这种情况下，将所有权保留为 `UID_ROOT`/`GID_WHEEL`，让操作员通过 `devfs.conf` 将你的节点映射到他们自己的本地组。在驱动内部发明新组几乎总是错误的做法。

### 三层策略：驱动、devfs.conf、devfs.rules

当你将驱动的基线与 `devfs.conf` 和 `devfs.rules` 结合时，你会得到一个值得从头到尾看一次的分层策略模型。考虑一个驱动用 `root:wheel 0600` 创建的设备。三层对其起作用：

- **第 1 层，驱动本身**：设定基线。每个 devfs 挂载上的每个 `/dev/myfirst/0` 都从 `root:wheel 0600` 开始。
- **第 2 层，`/etc/devfs.conf`**：每个主机 devfs 挂载应用一次，通常在启动时。可以更改所有权、模式或添加符号链接。在运行的主机上，在 `service devfs restart` 后，节点可能显示为 `root:operator 0660`。
- **第 3 层，`/etc/devfs.rules`**：根据附加到挂载的规则集在挂载时应用。使用规则集 `10` 的 jail 的 devfs 挂载看到过滤后、可能修改过的子集。同一节点可能在 jail 内被隐藏，或以进一步的模式和组调整取消隐藏。

这种分层的实际后果是**同一个 cdev 可以在同一时间在不同地方看起来不同**。在主机上它可能是 `0660`，由 `operator` 拥有。在 jail 中它可能是 `0640`，由 jail 内用户身份拥有。在另一个 jail 中它可能根本不存在。

这是一个特性，不是错误。它让你发布具有严格基线的驱动，让操作员无需编辑你的代码就能按环境放宽。第八章第 10 节通过一个实际示例演练这三层。


## 命名、单元号和子目录

`make_dev_s(9)` 的 printf 风格参数选择节点在 `/dev` 中出现的位置。在第七章中你使用：

```c
error = make_dev_s(&args, &sc->cdev, "myfirst%d", sc->unit);
```

这产生了 `/dev/myfirst0`。两个细节隐藏在其中。

第一个细节是 `sc->unit`。它是 FreeBSD 分配给你的设备实例的 Newbus 单元号。附加一个实例时，你得到 `0`。如果你的驱动支持多个实例，你可能会看到 `myfirst0`、`myfirst1` 等。

第二个细节是格式字符串本身。设备名称是相对于 `/dev` 的路径，它们可以包含斜杠。像 `"myfirst/%d"` 这样的名称不会产生一个带有斜杠的奇怪文件名；devfs 像文件系统一样解释斜杠，按需创建中间目录，并将节点放在里面。所以：

- 带单元 `0` 的 `"myfirst%d"` 给出 `/dev/myfirst0`。
- 带单元 `0` 的 `"myfirst/%d"` 给出 `/dev/myfirst/0`。
- `"myfirst/control"` 给出 `/dev/myfirst/control`，根本没有单元号。

将相关节点分组到子目录是暴露多个界面的驱动的常见模式。想想来自 `/usr/src/sys/dev/led/led.c` 的 `/dev/led/*`，或来自数据包过滤器子系统的 `/dev/pf`、`/dev/pflog*` 等。子目录使关系一目了然，保持 `/dev` 的顶层整洁，让操作员可以用单个 `devfs.conf` 行授予或拒绝整个集合的访问。

你将在本章为 `myfirst` 采用此模式。主数据路径从 `/dev/myfirst0` 移动到 `/dev/myfirst/0`。然后你将添加一个别名，以便任何记住以前布局的实验室脚本的旧路径继续工作。

### 真正 FreeBSD 树中的名称

浏览运行中的 FreeBSD 系统的 `/dev` 本身就很有教育意义，因为你在那里看到的命名约定是由你的驱动将面临的相同压力塑造的。按主题分组的简短浏览：

- **直接设备名称。** `/dev/null`、`/dev/zero`、`/dev/random`、`/dev/urandom`。每个节点一个 cdev，顶层，简短稳定的名称。适合没有层次结构的单例。
- **带单元号的名称。** `/dev/bpf0`、`/dev/bpf1`、`/dev/ttyu0`、`/dev/md0`。每个实例一个 cdev，从零开始编号。格式字符串看起来像 `"bpf%d"`，驱动管理单元号。
- **每驱动的子目录。** `/dev/led/*`、`/dev/pts/*`、某些配置下的 `/dev/ipmi*`。当单个驱动暴露许多相关节点时使用。使操作员策略简单：一个 `devfs.conf` 或 `devfs.rules` 条目可以覆盖整个集合。
- **拆分数据和控制节点。** `/dev/bpf`（克隆入口点）加上每次打开的克隆，`/dev/fido/*` 用于 FIDO 设备等。当驱动需要发现与数据的不同语义时使用。
- **方便的别名名称。** `/dev/stdin`、`/dev/stdout`、`/dev/stderr` 是 devfs 为当前进程的文件描述符提供的符号链接。`/dev/random` 和 `/dev/urandom` 曾经是别名；在现代 FreeBSD 中，它们是由同一个随机驱动服务的独立节点，但历史仍然可见。

你不需要记住这些模式。你需要识别它们，因为当你阅读现有驱动时，一旦命名约定被命名，它们都会更有意义。

### 每个设备多个节点

有些驱动暴露一个节点就够了。其他驱动暴露多个，每个都有不同的语义。常见的拆分是：

- 一个承载大量有效载荷（读取、写入、mmap）并用于高吞吐量使用的**数据节点**。
- 一个承载管理流量（配置、状态、重置）通常对监控工具可组读的**控制节点**。

当驱动这样做时，它在 `attach()` 中调用两次 `make_dev_s(9)` 并在 softc 中保留两个 cdev 指针。在第八章中，你将止步于一个数据节点加一个别名，但这个模式值得现在知道，这样你看到它时就能认出来。

实验 8.5 构建了一个最小化的 `myfirst` 双节点变体，数据节点在 `/dev/myfirst/0`，控制节点在 `/dev/myfirst/0.ctl`。每个节点有自己的 `cdevsw` 和自己的权限模式。实验的目的是展示代码中的模式；你后面章节的大部分驱动都会使用它。

### 深入了解 make_dev 家族

到目前为止，你为创建的每个节点都使用了 `make_dev_s(9)`。FreeBSD 实际上提供了一个小型 `make_dev*` 函数家族，每个都有略微不同的人体工程学。阅读现有驱动会让你接触到所有这些，知道何时使用哪个可以省去以后的麻烦。

完整声明在 `/usr/src/sys/sys/conf.h` 中。按现代程度递增的顺序：

```c
struct cdev *make_dev(struct cdevsw *_devsw, int _unit, uid_t _uid, gid_t _gid,
                      int _perms, const char *_fmt, ...);

struct cdev *make_dev_cred(struct cdevsw *_devsw, int _unit,
                           struct ucred *_cr, uid_t _uid, gid_t _gid, int _perms,
                           const char *_fmt, ...);

struct cdev *make_dev_credf(int _flags, struct cdevsw *_devsw, int _unit,
                            struct ucred *_cr, uid_t _uid, gid_t _gid, int _mode,
                            const char *_fmt, ...);

int make_dev_p(int _flags, struct cdev **_cdev, struct cdevsw *_devsw,
               struct ucred *_cr, uid_t _uid, gid_t _gid, int _mode,
               const char *_fmt, ...);

int make_dev_s(struct make_dev_args *_args, struct cdev **_cdev,
               const char *_fmt, ...);
```

逐一查看。

**`make_dev`** 是原始的位置参数形式。它直接返回新的 cdev 指针，或在任何错误时 panic。在错误时 panic 是一个强烈的暗示，它旨在用于无法恢复的代码路径，例如真正永恒设备的非常早期初始化。在新驱动中避免使用它。它仍在树中只是因为旧驱动使用它，而且因为其中一些驱动早期 panic 是真正可接受的地方。

**`make_dev_cred`** 添加凭证（`struct ucred *`）参数。凭证由 devfs 在应用规则时使用；它告诉系统"此 cdev 是由此凭证创建的"以进行规则匹配。大多数驱动为凭证传递 `NULL` 并获得默认行为。你会在响应用户请求按需克隆设备的驱动中看到这种形式；在其他地方不常见。

**`make_dev_credf`** 用标志字扩展 `make_dev_cred`。这是家族中第一个让你说"如果失败不要 panic；返回 `NULL` 让我可以处理它"的成员。

**`make_dev_p`** 是 `make_dev_credf` 的功能等价物，具有更清晰的返回值约定：它返回一个 `errno` 值（成功为零）并通过输出参数写入新的 cdev 指针。这是在 `make_dev_s` 存在之前编写的现代代码库中最广泛使用的形式。

**`make_dev_s`** 是现代推荐的形式。它接受预先填充的 `struct make_dev_args`（用 `make_dev_args_init_impl` 初始化，如下所述）并通过输出参数写入 cdev 指针。它返回一个 `errno` 值，成功为零。本书使用它的原因很简单：它是最易读的形式、最易扩展的形式（向参数结构添加新字段是 ABI 友好的）和最易错误检查的形式。

参数结构，同样来自 `/usr/src/sys/sys/conf.h`：

```c
struct make_dev_args {
        size_t         mda_size;
        int            mda_flags;
        struct cdevsw *mda_devsw;
        struct ucred  *mda_cr;
        uid_t          mda_uid;
        gid_t          mda_gid;
        int            mda_mode;
        int            mda_unit;
        void          *mda_si_drv1;
        void          *mda_si_drv2;
};
```

`mda_size` 由 `make_dev_args_init(a)` 自动设置；你从不触及它。`mda_flags` 承载下述 `MAKEDEV_*` 标志。`mda_devsw`、`mda_cr`、`mda_uid`、`mda_gid`、`mda_mode` 和 `mda_unit` 对应于旧形式的位置参数。`mda_si_drv1` 和 `mda_si_drv2` 让你预填充结果 cdev 上的驱动指针槽位，这就是你如何避免 `make_dev_s` 返回后但在你赋值之前 `si_drv1` 可能短暂为 `NULL` 的窗口。始终在调用前填充 `mda_si_drv1`。

### 你应该使用哪种形式？

对于新驱动，**使用 `make_dev_s`**。本书中的每个示例都使用它，你为自己编写的每个驱动都应该这样做，除非非常具体的原因强制否则。

对于阅读现有代码，识别所有这些。如果你发现一个调用 `make_dev(...)` 并忽略其返回值的驱动，你要么在看一个早于现代 API 的驱动，要么是作者决定失败时 panic 是可接受的驱动。两者在上下文中都是可辩护的；两者都不是新代码的正确默认值。

### MAKEDEV_* 标志

可以 OR 到 `mda_flags` 中（或作为第一个参数传递给 `make_dev_p` 和 `make_dev_credf`）的标志在 `/usr/src/sys/sys/conf.h` 中定义。每一个都有特定含义：

- **`MAKEDEV_REF`**：额外递增结果 cdev 的引用计数一次。当调用者计划跨通常会丢弃引用的事件长期持有 cdev 指针时使用。在初学者级别的驱动中很少见。
- **`MAKEDEV_NOWAIT`**：告诉分配器如果内存紧张不要等待。在内存不足条件下，函数返回 `ENOMEM`（对于 `make_dev_s`）或 `NULL`（对于旧形式）而不是阻塞。仅当你的调用者无法承受睡眠时使用。
- **`MAKEDEV_WAITOK`**：反向。告诉分配器为内存睡眠是安全的。这是 `make_dev` 和 `make_dev_s` 的默认值，所以你很少拼写出来。
- **`MAKEDEV_ETERNAL`**：将 cdev 标记为永不销毁。devfs 将拒绝在正常操作期间对其执行 `destroy_dev(9)`。由永恒的内核内设备如 `null`、`zero` 和 `full` 使用。不要在你计划卸载的驱动中设置这个。
- **`MAKEDEV_CHECKNAME`**：要求函数在创建之前根据 devfs 的规则验证节点名称。失败时它返回错误而不是创建命名错误的 cdev。在从用户输入合成名称的代码路径中有用。
- **`MAKEDEV_WHTOUT`**：创建"whiteout"条目，与堆叠文件系统结合使用以掩盖底层条目。你在驱动工作中不会遇到。
- **`MAKEDEV_ETERNAL_KLD`**：一个宏，当代码在可加载模块之外构建时扩展为 `MAKEDEV_ETERNAL`，当代码作为 KLD 构建时扩展为零。这让设备（如 `null`）的共享源在静态编译时设置标志，在作为模块加载时清除它，以便模块仍然可卸载。

对于典型的初学者级别驱动，标志字段为零，这就是配套代码树中的 `myfirst` 示例使用的值。当节点名称由用户输入构建或来自你未完全控制的字符串时，`MAKEDEV_CHECKNAME` 值得使用；对于传递常量格式字符串如 `"myfirst/%d"` 的驱动，标志没有增加任何有用的东西。


### cdevsw d_flags

与 `MAKEDEV_*` 标志分开，`cdevsw` 本身承载一个 `d_flags` 字段，塑造 devfs 和其他内核机制如何对待 cdev。这些标志在前面几节的 cdevsw 巡览中列出；本节是理解何时设置它们的地方。

**`D_TRACKCLOSE`** 是你在第八章中最可能想要的标志。默认情况下，devfs 仅在引用 cdev 的最后一个文件描述符被释放时调用你的 `d_close`。如果进程调用了 `dup(2)` 或 `fork(2)` 且两个描述符共享打开，`d_close` 在最后才触发一次。这通常是你想要的。如果你需要可靠的每次描述符关闭钩子，这就不是你想要的。设置 `D_TRACKCLOSE` 使 devfs 对每个描述符的每次 `close(2)` 都调用 `d_close`。对于使用 `devfs_set_cdevpriv(9)` 管理每次打开状态的驱动，析构函数通常是更好的钩子；`D_TRACKCLOSE` 在你的设备语义真正需要每次关闭都可观察时仍然有用。

**`D_MEM`** 将 cdev 标记为内存风格设备；`/dev/mem` 本身设置了它。它改变了某些内核路径处理对节点的 I/O 的方式。

**`D_DISK`**、**`D_TAPE`**、**`D_TTY`** 是设备类别的提示。现代驱动大多不设置它们，因为 GEOM 拥有磁盘，TTY 子系统拥有 TTY，磁带设备通过自己的层路由。你会在旧驱动上看到它们。

**`D_MMAP_ANON`** 改变映射设备如何产生页面。`zero` 设备设置了它；映射 `/dev/zero` 产生匿名的零填充页面。值得识别；在你编写想要相同语义的驱动之前不需要设置它。

**`D_NEEDGIANT`** 请求此 cdev 的所有 `cdevsw` 处理程序在 Giant 锁下调度。它作为未经 SMP 审计的驱动的安全毯而存在。新驱动不应设置此标志。如果你在 2010 年左右之后编写的代码中看到它，应持怀疑态度。

**`D_NEEDMINOR`** 告诉 devfs 驱动使用 `clone_create(9)` 按需分配次设备号。在你编写克隆驱动之前不会遇到这个，这超出了本章的范围。

你在 `myfirst` 中将设置的标志在大多数版本中是——没有。一旦第八章添加了每次打开的状态，驱动仍然不需要 `D_TRACKCLOSE`，因为 cdevpriv 析构函数涵盖了每次描述符的清理需求。

### 名称长度和名称字符

`make_dev_s` 接受 printf 风格的格式，产生 devfs 存储在 cdev 的 `si_name` 字段中的名称。该字段的大小是 `SPECNAMELEN + 1`，`SPECNAMELEN` 目前是 255。超过该长度的名称是错误的。

除了长度之外，名称必须可作为 devfs 下的文件系统路径接受。这意味着它不能包含空字节，不能使用 `.` 或 `..` 作为组件，不应该使用 shell 或脚本特殊解释的字符。最安全的集合是小写 ASCII 字母、数字和三个分隔符 `/`、`-` 和 `.`。其他字符有时有效有时无效；如果你曾经想在设备名称中使用空格、冒号或非 ASCII 字符，停下来选择一个更简单的名称。

### 单元号：它们从何而来

单元号是区分同一驱动实例的小整数。它们出现在设备名称中（`myfirst0`、`myfirst1`）、`sysctl` 分支中（`dev.myfirst.0`、`dev.myfirst.1`）和 cdev 的 `si_drv0` 字段中。

两种常见的分配方式：

**Newbus 分配。** 当你的驱动挂载到总线并且 Newbus 实例化设备时，总线分配一个单元号。你用 `device_get_unit(9)` 检索它并用作 `sc->unit`，正如第七章所做的。Newbus 保证该数字在驱动的命名空间中是唯一的。

**使用 `unrhdr` 显式分配。** 对于在 Newbus 流程之外创建节点的驱动，`unrhdr(9)` 分配器从池中分配单元号。`/usr/src/sys/dev/led/led.c` 使用这种方式：`sc->unit = alloc_unr(led_unit);`。LED 框架不为每个 LED 通过 Newbus 挂载，所以它不能向 Newbus 请求单元号；它维护自己的单元池。

对于构建在 Newbus 上的初学者驱动，第一种方式是要使用的。第二种在你编写可以按需多次实例化的伪设备时变得相关，这是后面章节的话题。

### 树中的命名约定

由于你可能在学习过程中阅读真正的 FreeBSD 驱动，识别其名称采取的形状是有帮助的。简短浏览：

- **`bpf%d`**：每个 BPF 实例一个节点。见于 `/usr/src/sys/net/bpf.c`。
- **`md%d`**：内存磁盘。`/usr/src/sys/dev/md/md.c`。
- **`led/%s`**：每个驱动一个子目录，每个 LED 一个节点。`/usr/src/sys/dev/led/led.c` 使用名称参数作为自由格式字符串，由调用者选择，例如 `led/ehci0`。
- **`ttyu%d`**、**`cuaU%d`**：硬件串口，配对的"入"和"出"节点。
- **`ptyp%d`**、**`ttyp%d`**：伪终端对。
- **`pts/%d`**：子目录中的现代 PTY 分配。
- **`fuse`**：FUSE 子系统的单例入口点。
- **`mem`**、**`kmem`**：内存检查的单例。
- **`pci`**、**`pciconf`**：PCI 总线检查接口。
- **`io`**：I/O 端口访问，单例。
- **`audit`**：审计子系统控制设备。

注意在大多数这些中，名称编码了驱动的身份。这是故意的。当操作员以后需要编写 `devfs.conf` 规则或防火墙规则或备份脚本时，他们匹配路径，可预测的路径使他们的工作更容易。

### 处理多个单元

你的第七章驱动在其 `device_identify` 回调中注册了恰好一个 Newbus 子设备，所以只有一个实例，唯一的单元号是 `0`。一些驱动需要多个实例，无论是在启动时还是按需。

对于在启动时以固定数量实例化的驱动，模式是在 `device_identify` 中添加更多子设备：

```c
static void
myfirst_identify(driver_t *driver, device_t parent)
{
        int i;

        for (i = 0; i < MYFIRST_INSTANCES; i++) {
                if (device_find_child(parent, driver->name, i) != NULL)
                        continue;
                if (BUS_ADD_CHILD(parent, 0, driver->name, i) == NULL)
                        device_printf(parent,
                            "myfirst%d: BUS_ADD_CHILD failed\n", i);
        }
}
```

Newbus 为每个子设备调用 `attach`，每次调用获得自己的 softc 和自己的单元号。你的 `make_dev_s` 格式字符串 `"myfirst/%d"` 与 `sc->unit` 然后产生 `/dev/myfirst/0`、`/dev/myfirst/1` 等。

对于按需实例化的驱动，架构非常不同。你通常暴露一个单一的"控制" cdev，当用户在其上执行操作时，驱动分配一个新实例和一个新 cdev。`/usr/src/sys/dev/md/md.c` 中的内存磁盘驱动就是一个明显的例子：`/dev/mdctl` 接受 `MDIOCATTACH` ioctl，每次成功的挂载产生一个新的 `/dev/mdN` cdev 通过 GEOM 层。伪终端子系统采用类似方法：打开 `/dev/ptmx` 的用户在另一端获得一个新分配的 `/dev/pts/N`。第八章不会带你走过那些机制；知道当你看到驱动从事件处理器内部而不是从 `attach` 创建 cdev 时，动态实例化就是你所看到的模式就够了。

### 小型弯路：devtoname 和朋友

三个小型辅助函数在驱动代码和本书后续中经常出现。值得收集：

- **`devtoname(cdev)`**：返回指向节点名称的指针。只读。用于日志记录：`device_printf(dev, "created /dev/%s\n", devtoname(sc->cdev))`。
- **`dev2unit(cdev)`**：返回 `si_drv0` 字段，按约定是单元号。在 `conf.h` 中定义为宏。
- **`device_get_nameunit(dev)`**：用于 `device_t`，返回 Newbus 范围的名称如 `"myfirst0"`。对互斥锁名称有用。

这三个在已知 cdev 或设备存活的上下文中使用是安全的，对于驱动处理程序来说总是如此。

## 别名：一个 cdev，多个名称

有时设备需要通过多个名称访问。也许你重命名了节点并希望旧名称在弃用期间继续工作。也许你想要一个总是指向单元 `0` 的稳定短名称，用户不需要知道当前是哪个单元。也许系统的其余部分已经有一个强烈的约定，你想与之良好配合。

FreeBSD 为此提供了 `make_dev_alias(9)`。别名本身是一个 `struct cdev`，但带有 `SI_ALIAS` 标志并共享与主节点相同的底层调度机制。打开别名的用户程序着陆在与打开主名称相同的 `cdevsw` 处理程序中。

签名，来自 `/usr/src/sys/sys/conf.h`：

```c
struct cdev *make_dev_alias(struct cdev *_pdev, const char *_fmt, ...);
int          make_dev_alias_p(int _flags, struct cdev **_cdev,
                              struct cdev *_pdev, const char *_fmt, ...);
```

你传入主 cdev、格式字符串和可选参数。你得到一个表示别名的新 cdev。完成后，用 `destroy_dev(9)` 销毁别名，与销毁任何其他 cdev 相同。

这是你将添加到 `myfirst_attach()` 的代码形状：

```c
sc->cdev_alias = make_dev_alias(sc->cdev, "myfirst");
if (sc->cdev_alias == NULL) {
        device_printf(dev, "failed to create /dev/myfirst alias\n");
        /* fall through; the primary node is still usable */
}
```

关于那个代码片段的两个观察。首先，创建别名失败不是致命的。主路径仍然工作，所以我们记录并继续。其次，你只需要在你计划在拆离时销毁别名时才需要保持别名 cdev 的指针。大多数驱动都需要，所以把它放在 softc 中 `cdev` 旁边。

### 别名与 devfs.conf 中的 `link`

熟悉 UNIX 符号链接的读者有时会问为什么 FreeBSD 提供两种不同的方式给设备第二个名称。区别是真实的，值得清楚说明。

`make_dev_alias(9)` 别名是一个**与主设备共享其调度机制的第二个 cdev**。当用户打开它时，devfs 直接走到你的 `cdevsw` 处理程序。文件系统中没有符号链接。对别名进行 `ls -l` 显示另一个字符特殊节点，有自己的模式和所有权。内核知道别名绑定到主 cdev（`SI_ALIAS` 标志和 `si_parent` 指针记录了那个关系），如果你的驱动记得对其调用 `destroy_dev(9)`，会在主设备消失时自动清理它。

`/etc/devfs.conf` 中的 `link` 指令在 devfs **内部**创建一个**符号链接**。`ls -l` 在类型字段中显示 `l` 和指向目标的箭头。打开它时，内核首先解析符号链接，然后打开目标。目标和链接有独立的权限和所有权；符号链接本身除了其存在之外不承载访问策略。

选择哪个？

- 当驱动本身有理由暴露额外名称时使用 `make_dev_alias`，例如一个短的知名形式或一个必须在权限级别与新的相同的旧路径。
- 当操作员想要一个方便的快捷方式且驱动没有意见时使用 `devfs.conf` 中的 `link`。那种链接不属于内核代码。

两种方法都有效。错误的选择不是危险的；通常只是笨拙。保持驱动代码精简，让操作员策略留在策略该在的地方。


### 三种给节点两个名称方式的比较表

简短比较将区别集中在一处：

| 属性                          | `make_dev_alias` | `devfs.conf link` | `ln -s` 符号链接 |
|-----------------------------------|:----------------:|:-----------------:|:-------------------------:|
| 存在于内核代码中              | 是              | 否                | 否                        |
| 存在于 devfs 中              | 是              | 是               | 否（存在于底层文件系统）|
| `ls -l` 显示为 `c`              | 是              | 否（显示为 `l`） | 否（显示为 `l`）         |
| 承载自己的模式和所有者    | 是              | 继承目标   | 继承目标           |
| 驱动卸载时自动清理     | 是              | 是（下次 `service devfs restart`） | 否 |
| 重启后保留                 | 仅当驱动加载时 | 是，如果在 `devfs.conf` 中 | 是，如果在 `/etc` 或类似位置下 |
| 适合驱动拥有的名称 | 是              | 否                | 否                        |
| 适合操作员快捷方式 | 否               | 是               | 有时                 |

模式是：驱动拥有其主要名称和任何承载策略的别名；操作员拥有不承载策略的方便链接。越过那条线就是未来维护痛苦的来源。

### `make_dev_alias_p` 变体

`make_dev_alias` 有一个兄弟，接受标志字并返回 `errno`，原因与主 `make_dev` 家族相同。其在 `/usr/src/sys/sys/conf.h` 中的声明：

```c
int make_dev_alias_p(int _flags, struct cdev **_cdev, struct cdev *_pdev,
                     const char *_fmt, ...);
```

有效标志是 `MAKEDEV_WAITOK`、`MAKEDEV_NOWAIT` 和 `MAKEDEV_CHECKNAME`。行为类似于 `make_dev_p`：成功为零，新 cdev 通过输出指针写入，失败时为非零 `errno` 值。

如果你的别名创建处于不能睡眠的路径中，使用 `make_dev_alias_p(MAKEDEV_NOWAIT, ...)` 并准备好 `ENOMEM`。在常规情况下，你的别名是在正常条件下 `attach` 期间创建的，`make_dev_alias(9)` 就可以了；它在内部使用 `MAKEDEV_WAITOK`。

### `make_dev_physpath_alias` 变体

有第三个别名函数 `make_dev_physpath_alias`，由想要除了逻辑名称之外还发布物理路径别名的驱动使用。它的存在是为了支持某些存储驱动暴露的 `/dev/something/by-path/...` 下的硬件拓扑路径。大多数初学者驱动从不需要它。

### 在树中阅读 `make_dev_alias` 的使用

一个有用的练习：在 `/usr/src/sys` 中 `grep` 搜索 `make_dev_alias` 并查看使用它的上下文。你会在想要在动态编号的旁边发布稳定名称的存储驱动中、在某些想要旧兼容名称的伪设备中，以及少量建模硬件拓扑的专业驱动中找到它。

大多数驱动不使用它，这没问题。当驱动使用时，原因几乎总是三者之一：

1. **旧路径兼容性。** 一个被重命名但必须保持旧名称工作的驱动。
2. **众所周知的快捷方式。** 一个总是解析到实例零或当前默认值的短名称，这样 shell 脚本可以写一个路径而不是协商单元号。
3. **拓扑暴露。** 一个反映硬件所在位置的名称，除了硬件是什么之外。

你的 `myfirst` 驱动正在使用情况 1：`/dev/myfirst` 作为 `/dev/myfirst/0` 的快捷方式，这样第七章的文字仍然可以解析。这是典型初学者使用的形状。

### 别名生命周期和销毁顺序

注册为别名的 cdev 设置了 `SI_ALIAS` 标志并通过 `si_parent` 反向指针链接到主 cdev 的 `si_children` 列表中。这意味着内核知道这个关系，即使你以略微错误的顺序拆除 cdev 也会做正确的事。这并不意味着你可以忽略顺序；它意味着销毁比一般内核对象的拆除更宽容。

在实践中，你应该在 `detach` 路径中遵循的规则是：**先销毁别名，然后是主设备**。配套代码树中的示例驱动这样做，原因是简单可读性。任何其他顺序使你的代码更难推理，审查者会标记它。

如果驱动完全遗漏了别名的 `destroy_dev` 调用，主设备的销毁会在主设备消失时自动展开别名；这就是 `destroy_devl` 在遍历 `si_children` 时所做的。但将该工作留给析构函数是浪费的，因为主设备持有的引用使其存活时间比需要的更长，而且操作员看到别名"稍后"消失而不是在卸载时干净地消失。只需销毁两者。

### 当别名开始发出异味时

一些使用别名的模式是值得命名的轻微代码异味：

- **别名链。** 别名的别名合法但几乎总是意味着驱动试图掩盖一个本应重新审视的命名决策。如果你发现想要别名的别名，停下来并重命名主设备。
- **太多别名。** 一两个是常规的。五个或更多表明驱动不确定它想被叫什么。重新审视命名。
- **模式差异巨大的别名。** 指向同一处理程序集但暴露截然不同权限模式的两个 cdev 与陷阱无法区分。使权限一致，或使用两个独立的主设备和两个独立的 `cdevsw` 值在代码中强制执行不同策略。

这些都不是错误。它们是设计正在漂移的信号。及早注意它们，驱动保持可读；忽略它们，驱动变成审查者恐惧的东西。

## 使用 devfs_set_cdevpriv 的每次打开状态

现在我们来到为本章为第九章做准备的部分。你的第七章驱动通过在 softc 中设置标志来强制执行**独占打开**。这行得通，但这是最粗糙的可能策略。许多真正的设备允许多个打开者并希望保留少量**每次文件描述符的**而非每次设备的簿记。想想日志流、状态源，或任何不同消费者想要自己读取位置的节点。

FreeBSD 为此提供了三个相关例程，在 `/usr/src/sys/sys/conf.h` 中声明，在 `/usr/src/sys/fs/devfs/devfs_vnops.c` 中实现：

```c
int  devfs_set_cdevpriv(void *priv, d_priv_dtor_t *dtr);
int  devfs_get_cdevpriv(void **datap);
void devfs_clear_cdevpriv(void);
```

模型简单且使用愉快：

1. 在你的 `d_open` 处理程序内部，分配一个小型的每次打开结构并调用 `devfs_set_cdevpriv(priv, dtor)`。内核将 `priv` 附加到当前文件描述符并记住 `dtor` 作为该描述符最终关闭时要调用的函数。
2. 在 `d_read`、`d_write` 或任何其他处理程序中，调用 `devfs_get_cdevpriv(&priv)` 检索指针。
3. 当进程调用 `close(2)`，或退出，或以其他方式放弃其对描述符的最后引用时，devfs 用 `priv` 调用你的析构函数。你释放你分配的任何东西。

你不需要担心相对于自己的 `d_close` 处理程序的清理顺序。Devfs 处理它。重要的不变量是每次成功的 `devfs_set_cdevpriv` 你的析构函数将被恰好调用一次。

来自 `/usr/src/sys/net/bpf.c` 的真实示例看起来像这样：

```c
d = malloc(sizeof(*d), M_BPF, M_WAITOK | M_ZERO);
error = devfs_set_cdevpriv(d, bpf_dtor);
if (error != 0) {
        free(d, M_BPF);
        return (error);
}
```

这本质上就是整个模式。BPF 分配一个每次打开的描述符，注册它，如果注册失败，释放分配并返回错误。析构函数 `bpf_dtor` 在描述符死亡时清理。你将为 `myfirst` 做同样的事情，用一个小得多的每次打开结构。

### myfirst 的最小化每次打开计数器

你将添加一个小型结构和一个析构函数。驱动中没有其他东西改变形状。

```c
struct myfirst_fh {
        struct myfirst_softc *sc;    /* back-pointer to the owning softc */
        uint64_t              reads; /* bytes this descriptor has read */
        uint64_t              writes;/* bytes this descriptor has written */
};

static void
myfirst_fh_dtor(void *data)
{
        struct myfirst_fh *fh = data;
        struct myfirst_softc *sc = fh->sc;

        mtx_lock(&sc->mtx);
        sc->active_fhs--;
        mtx_unlock(&sc->mtx);

        device_printf(sc->dev, "per-open dtor fh=%p reads=%lu writes=%lu\n",
            fh, (unsigned long)fh->reads, (unsigned long)fh->writes);

        free(fh, M_DEVBUF);
}
```

析构函数做了三件值得注意的事情。它在同一个保护其他 softc 计数器的互斥锁下递减 `active_fhs`，所以计数与 `d_open` 在打开描述符时看到的一致。它记录一行与 `open via ...` 消息形状匹配的内容，所以 `dmesg` 中的每次打开都有可见配对的析构函数。它在最后释放分配，在可能需要从 `fh` 读取的所有东西都已经运行之后。

在你的 `d_open` 中，分配其中之一并注册它：

```c
static int
myfirst_open(struct cdev *dev, int oflags, int devtype, struct thread *td)
{
        struct myfirst_softc *sc;
        struct myfirst_fh *fh;
        int error;

        sc = dev->si_drv1;
        if (sc == NULL || !sc->is_attached)
                return (ENXIO);

        fh = malloc(sizeof(*fh), M_DEVBUF, M_WAITOK | M_ZERO);
        fh->sc = sc;

        error = devfs_set_cdevpriv(fh, myfirst_fh_dtor);
        if (error != 0) {
                free(fh, M_DEVBUF);
                return (error);
        }

        mtx_lock(&sc->mtx);
        sc->open_count++;
        sc->active_fhs++;
        mtx_unlock(&sc->mtx);

        device_printf(sc->dev, "open via %s fh=%p (active=%d)\n",
            devtoname(dev), fh, sc->active_fhs);
        return (0);
}
```

注意两件事。首先，来自第七章的独占打开检查已经消失。有了每次打开状态，没有理由拒绝第二个打开者。如果你以后确实想要独占性，你仍然可以把它加回去；这是一个独立的决定。其次，析构函数将负责释放。你的 `d_close` 根本不需要触及 `fh`。

在稍后运行的处理程序中，如 `d_read`，你检索每次打开的结构：

```c
static int
myfirst_read(struct cdev *dev, struct uio *uio, int ioflag)
{
        struct myfirst_fh *fh;
        int error;

        error = devfs_get_cdevpriv((void **)&fh);
        if (error != 0)
                return (error);

        /* Real read logic arrives in Chapter 9. For now, report EOF
         * and leave the counter untouched so userland tests can observe
         * that the descriptor owns its own state.
         */
        (void)fh;
        return (0);
}
```

`(void)fh` 抑制"未使用变量"警告，直到第九章给它工作。现在这没问题。重要的是你的驱动有一个干净、工作、干净销毁的每次文件结构。从用户态你可以通过从两个进程打开设备并观察带有两个不同 `fh=` 指针的 device-printf 消息来确认布线。

### 析构函数保证什么

因为析构函数做了大部分工作，值得精确说明它何时运行以及那时世界处于什么状态。阅读 `/usr/src/sys/fs/devfs/devfs_vnops.c` 中的 `devfs_destroy_cdevpriv` 确认细节。

- 析构函数**每次成功的 `devfs_set_cdevpriv` 调用恰好运行一次**。如果函数返回 `EBUSY` 是因为描述符已经有私有数据，*你的*数据的析构函数永远不会被调用；你必须自己释放分配，就像示例代码所做的那样。
- 析构函数在**文件描述符被释放时**运行，而不是在你的 `d_close` 被调用时。对于普通的 `close(2)`，这两个时刻很近。对于在持有描述符时退出的进程，描述符作为退出清理的一部分被释放；析构函数仍然运行。对于通过 `fork(2)` 共享或通过 UNIX 域套接字传递的描述符，析构函数仅在最后一个引用丢弃时运行。
- 析构函数在没有代表你持有内核锁的情况下运行。如果你的析构函数触及 softc 状态，获取 softc 使用的任何锁，就像阶段 2 示例在递减 `active_fhs` 时所做的那样。
- 析构函数不能长时间阻塞。它不是永远睡眠的上下文，但也不是中断处理程序。把它当作普通的内核函数对待并保持简短。

### 当 `devfs_set_cdevpriv` 返回 EBUSY 时

`devfs_set_cdevpriv` 只能以一种有趣的方式失败：描述符已经有与其关联的私有数据。当某些东西——通常是你自己在之前调用中的代码——已经设置了一个 cdevpriv 而你试图设置另一个时会发生这种情况。干净的修复是做一次设置，早期进行，然后用 `devfs_get_cdevpriv` 在你需要的任何地方读回它。

由此产生两个注意事项。第一个是：不要从同一个打开调用两次 `devfs_set_cdevpriv`。第二个是：当调用失败时，在尝试设置之前释放你分配的任何东西。本章中的示例 `myfirst_open` 遵循这两个规则。当你将模式移植到自己的驱动时，牢记它们。

### 什么时候不使用 devfs_set_cdevpriv

每次打开状态不是一切的正确归属。将设备范围的状态保留在 softc 中，通过 `si_drv1` 可达。将每次打开的状态保留在 cdevpriv 结构中，通过 `devfs_get_cdevpriv` 可达。混合两者是写出在单打开者测试中工作并在两个进程同时出现时崩溃的驱动的最快方式。

`devfs_clear_cdevpriv(9)` 存在，你可能在第三方代码中看到它，但对于大多数驱动，通过析构函数的自动清理就足够了。只有在有具体原因时才使用 `devfs_clear_cdevpriv`，例如一个可以干净地响应 `ioctl(2)` 提前拆离每次打开状态的驱动。如果你不确定是否需要它，你就不需要。

### devfs_set_cdevpriv 内部：机制如何工作

你调用的两个函数从外面看几乎微不足道。它们驱动的机制值得看一次，因为了解它的形状使每个边缘情况更容易推理。

来自 `/usr/src/sys/fs/devfs/devfs_vnops.c`：

```c
int
devfs_set_cdevpriv(void *priv, d_priv_dtor_t *priv_dtr)
{
        struct file *fp;
        struct cdev_priv *cdp;
        struct cdev_privdata *p;
        int error;

        fp = curthread->td_fpop;
        if (fp == NULL)
                return (ENOENT);
        cdp = cdev2priv((struct cdev *)fp->f_data);
        p = malloc(sizeof(struct cdev_privdata), M_CDEVPDATA, M_WAITOK);
        p->cdpd_data = priv;
        p->cdpd_dtr = priv_dtr;
        p->cdpd_fp = fp;
        mtx_lock(&cdevpriv_mtx);
        if (fp->f_cdevpriv == NULL) {
                LIST_INSERT_HEAD(&cdp->cdp_fdpriv, p, cdpd_list);
                fp->f_cdevpriv = p;
                mtx_unlock(&cdevpriv_mtx);
                error = 0;
        } else {
                mtx_unlock(&cdevpriv_mtx);
                free(p, M_CDEVPDATA);
                error = EBUSY;
        }
        return (error);
}
```

浏览重要部分的简短旅程：

- `curthread->td_fpop` 是当前分发的文件指针。devfs 在调用你的 `d_open` 之前设置它，在之后取消设置。如果你从一个没有分发活动的上下文调用 `devfs_set_cdevpriv`，`fp` 会是 `NULL`，函数会返回 `ENOENT`。实际上这只在你从错误的上下文调用它时发生，例如从不绑定到文件的定时器回调。
- 一个小型记录 `struct cdev_privdata` 从专用 malloc 桶 `M_CDEVPDATA` 分配。它承载三个字段：你的指针、你的析构函数和到 `struct file` 的反向指针。
- 两个线程同时为同一个描述符进入此函数将是灾难，所以单个互斥锁 `cdevpriv_mtx` 保护关键部分。检查 `fp->f_cdevpriv == NULL` 就是防止双重注册：如果记录已经附加，新记录被释放，`EBUSY` 返回。
- 成功时，记录被插入两个列表：描述符自己的指针 `fp->f_cdevpriv`，和 cdev 的所有描述符私有记录列表 `cdp->cdp_fdpriv`。第一个使 `devfs_get_cdevpriv` 成为单指针查找。第二个使 devfs 在 cdev 被销毁时可以遍历每个活动记录。

析构函数路径同样小：

```c
void
devfs_destroy_cdevpriv(struct cdev_privdata *p)
{

        mtx_assert(&cdevpriv_mtx, MA_OWNED);
        KASSERT(p->cdpd_fp->f_cdevpriv == p,
            ("devfs_destoy_cdevpriv %p != %p",
             p->cdpd_fp->f_cdevpriv, p));
        p->cdpd_fp->f_cdevpriv = NULL;
        LIST_REMOVE(p, cdpd_list);
        mtx_unlock(&cdevpriv_mtx);
        (p->cdpd_dtr)(p->cdpd_data);
        free(p, M_CDEVPDATA);
}
```

有两件事要注意。首先，析构函数是在**互斥锁释放后**调用的，所以你的析构函数可以获取自己的锁，而不会有与 `cdevpriv_mtx` 死锁的风险。其次，记录本身在你的析构函数返回后立即被释放，所以指向它的陈旧指针将是释放后使用。如果你的析构函数把指针藏在其他地方，藏数据的副本，而不是记录。


### 与 fork、dup 和 SCM_RIGHTS 的交互

UNIX 中的文件描述符有三种常见的倍增方式：`dup(2)`、`fork(2)` 和通过 UNIX 域套接字使用 `SCM_RIGHTS` 传递。每种都产生对同一 `struct file` 的额外引用。devfs 的 cdevpriv 机制在所有三种中行为一致。

在 `dup(2)` 或 `fork(2)` 之后，新的文件描述符引用与原始**相同**的 `struct file`。cdevpriv 记录以 `struct file` 为键，而不是描述符号，所以两个描述符共享记录。你的析构函数恰好触发一次，当指向该文件的最后一个描述符被释放时。那个最后释放可以是显式的 `close(2)`、关闭一切的隐式 `exit(3)`，甚至终止进程的崩溃。

通过 `SCM_RIGHTS` 传递描述符从 cdevpriv 的角度来看是同样的故事。接收进程获得一个指向同一 `struct file` 的新描述符。记录保持附加；析构函数仍然仅在最后一个引用丢弃时触发，这现在可能在套接字另一端的进程中。

这通常正是你想要的，因为它匹配用户的心智模型。每个概念上的打开一个每次打开状态。如果你曾经需要一个不同的模型，例如每个 `dup(2)` 的描述符应该有自己的状态的模型，解决方案是在你的 `cdevsw` 上设置 `D_TRACKCLOSE` 并在 `d_open` 本身内分配每次描述符的状态而不使用 `devfs_set_cdevpriv`。那是不寻常的；普通驱动不需要它。

### 树中真实使用的巡览

为了巩固模式，以下是三个以可识别方式使用 cdevpriv 的驱动的简短巡览。你不需要理解每个驱动整体上做什么；只关注设备文件形状。

**`/usr/src/sys/net/bpf.c`** 是规范示例。它的 `bpfopen` 分配一个每次打开的描述符，调用 `devfs_set_cdevpriv(d, bpf_dtor)`，并设置一小堆计数器和状态。析构函数 `bpf_dtor` 拆除所有这些：它从其 BPF 接口拆离描述符、释放计数器、排空选择列表并丢弃引用。模式正是本章所描述的，加上第六部分将重新讨论的大量 BPF 特有机制。

**`/usr/src/sys/fs/fuse/fuse_device.c`** 采用相同模式并在其上层叠 FUSE 特有状态。打开分配一个 `struct fuse_data`，用 `devfs_set_cdevpriv` 注册它，每个后续处理程序用 `devfs_get_cdevpriv` 检索它。析构函数拆除 FUSE 会话。

**`/usr/src/sys/opencrypto/cryptodev.c`** 使用 cdevpriv 管理每次打开的加密会话状态。每次打开获得自己的簿记，析构函数清理它。

这三个驱动在子系统级别几乎没有任何共同之处：一个是关于数据包捕获，一个是关于用户态文件系统，一个是关于硬件加密卸载。它们共享的是设备文件形状。同样的三个步骤，同样的顺序，同样的原因。

### 每次打开结构中放置什么的模式

既然你知道了机制，设计问题就是你的每次打开结构应该持有哪些字段。一些模式在真正的驱动中反复出现。

**计数器。** 读取的字节数、写入的字节数、进行的调用数、报告的错误数。每个描述符拥有自己的计数器。`myfirst` 在阶段 2 已经用 `reads` 和 `writes` 做到了这一点。

**读取位置。** 如果你的驱动暴露一个可寻道的字节流，当前偏移量属于每次打开结构，而不是 softc。两个处于不同偏移量的读取者是原因。

**订阅句柄。** 如果描述符正在读取事件，并且 `poll` 或 `kqueue` 需要知道是否有更多事件挂起给这个特定描述符，订阅记录属于这里。第十章使用此模式。

**过滤状态。** 像 BPF 这样的驱动让每个描述符安装过滤程序。该程序的编译形式是每次描述符的。同样，属于每次打开结构。

**预留或票据。** 如果驱动分出稀缺资源（硬件槽位、DMA 通道、共享缓冲区范围）并将它们绑定到打开，记录进入每次打开状态。当描述符关闭时，析构函数自动释放预留。

**凭证快照。** 一些驱动想在打开时记住谁打开了描述符，与当前正在读取或写入的人分开。在打开时捕获 `td->td_ucred` 的快照是常见模式。凭证是引用计数的（`crhold`/`crfree`），析构函数是丢弃引用的正确地方。

不是每个驱动都需要所有这些。列表是菜单，不是清单。当你设计驱动时，遍历它并问"哪些信息属于这个节点的这次特定打开？"答案进入每次打开结构。

### 关于从 softc 交叉引用每次打开记录的警告

每次打开状态出现的一个诱惑是 softc 携带指向每次打开记录的返回指针，这样广播事件给每个描述符就变成简单的列表遍历。诱惑是可以理解的；实现充满了边缘情况。两个线程争相关闭最后一个描述符而第三个试图广播是打破直接代码的场景，修复它往往需要比你想要添加的更多锁。

FreeBSD 的答案是 `devfs_foreach_cdevpriv(9)`，一个基于回调的迭代器，在正确的锁下遍历附加到给定 cdev 的每次打开记录。如果你曾经需要这个模式，使用那个函数并给它一个回调。不要维护自己的列表。

我们不会在第八章使用 `devfs_foreach_cdevpriv`。这里提到它是因为如果你在 FreeBSD 树中扫描 `cdevpriv`，你会找到它，你应该将它识别为自己重新发明迭代的安全替代方案。

## 安全销毁 cdev

将 cdev 放入 devfs 的行为是常规的。再次取出它需要思考。第七章教了你 `destroy_dev(9)`，对于表现良好的驱动的简单路径，这就是你需要的全部。真正的驱动有时需要更多。本节演练销毁辅助函数家族，解释它们保证什么，并展示每一个在什么是正确工具。

### 排空模型

让我们从销毁必须回答的问题开始："什么时候释放 softc 和卸载模块是安全的？"天真的答案是"在 `destroy_dev` 返回之后"，这几乎正确。仔细的答案是"在 `destroy_dev` 返回**并且**没有更多内核线程可以在我此 cdev 的任何处理程序中之后"。

你前面遇到的 `struct cdev` 计数器是内核跟踪这个的方式。`si_threadcount` 每次 devfs 代表用户系统调用进入你的处理程序之一时递增，每次处理程序返回时递减。`destroy_devl`——`destroy_dev` 调用的内部函数——监视那个计数器。以下是来自 `/usr/src/sys/kern/kern_conf.c` 的相关摘录：

```c
while (csw != NULL && csw->d_purge != NULL && dev->si_threadcount) {
        csw->d_purge(dev);
        mtx_unlock(&cdp->cdp_threadlock);
        msleep(csw, &devmtx, PRIBIO, "devprg", hz/10);
        mtx_lock(&cdp->cdp_threadlock);
        if (dev->si_threadcount)
                printf("Still %lu threads in %s\n",
                    dev->si_threadcount, devtoname(dev));
}
while (dev->si_threadcount != 0) {
        /* Use unique dummy wait ident */
        mtx_unlock(&cdp->cdp_threadlock);
        msleep(&csw, &devmtx, PRIBIO, "devdrn", hz / 10);
        mtx_lock(&cdp->cdp_threadlock);
}
```

两个循环。第一个循环在驱动提供 `d_purge` 时调用它；第二个只是等待。在两种情况下结果相同：`destroy_dev` 在 `si_threadcount` 为零之前不会返回。这就是使销毁安全的**排空**行为。当调用返回时，没有线程在任何处理程序内，也没有新线程可以进入，因为 `si_devsw` 已被清除。

这对你的代码意味着：**在 `destroy_dev(sc->cdev)` 返回之后，用户空间中没有任何东西可以触发对此 cdev 调用你的处理程序**。你可以自由销毁那些处理程序依赖的 softc 成员。

### 四个销毁函数

FreeBSD 暴露了四个相关的 cdev 销毁函数。每个处理略有不同的情况。

**`destroy_dev(struct cdev *dev)`**

普通情况。同步：等待进行中的处理程序完成，然后从 devfs 取消链接 cdev 并释放内核的主引用。在第七章和本书中每个单线程销毁路径中使用。要求调用者可睡眠并且不持有进行中的处理程序可能需要的任何锁。

**`destroy_dev_sched(struct cdev *dev)`**

延迟形式。在 taskqueue 上调度销毁并立即返回。当调用上下文不能睡眠时有用，例如从锁下运行的回调中。实际销毁异步发生，调用者在函数返回时不能假设它已完成。

**`destroy_dev_sched_cb(struct cdev *dev, void (*cb)(void *), void *arg)`**

相同的延迟形式，但带有一个在销毁完成后运行的回调。当你需要在知道 cdev 真正消失后进行后续工作（例如释放 softc）时使用。

**`destroy_dev_drain(struct cdevsw *csw)`**

清扫。等待注册到给定 `cdevsw` 的**每个** cdev 完全销毁，包括通过延迟形式调度的。当你即将取消注册或释放 `cdevsw` 本身时使用，例如在发布多个驱动的模块的 `MOD_UNLOAD` 处理程序内部。

### destroy_dev_drain 存在是为了防止的竞争

排空是一个微妙的点，解释它的最好方式是用它修复的场景。

假设你的模块导出一个 `cdevsw`。在 `MOD_UNLOAD` 中，你的代码调用 `destroy_dev(sc->cdev)` 然后返回成功。内核继续拆除模块。一切看起来很好，直到稍后通过 `destroy_dev_sched` 早期调度的延迟任务终于运行。该任务在清理过程中解引用 `struct cdevsw`。`cdevsw` 已随模块一起被取消映射。内核 panic。

竞争窗口很窄但真实。`destroy_dev_drain` 是修复：在你确信不会再创建新 cdev 后对 `cdevsw` 调用它，它不会返回直到注册到该 `cdevsw` 的每个 cdev 都完成了销毁。只有那时让模块走才是安全的。

如果你的驱动从 `attach` 创建一个 cdev，从 `detach` 销毁它，从不使用延迟形式，你不需要 `destroy_dev_drain`。`myfirst` 不需要它。管理克隆 cdev 或从事件处理器销毁 cdev 的真正驱动通常需要。

### detach 中的操作顺序

鉴于以上所有，带有主 cdev、别名和每次打开状态的驱动的 `detach` 处理程序中正确的操作顺序是：

1. 如果有任何描述符仍然打开，拒绝拆离。返回 `EBUSY`。你的 `active_fhs` 计数器是检查的正确东西。
2. 用 `destroy_dev(sc->cdev_alias)` 销毁别名 cdev。这从 devfs 取消链接别名并排空对其的任何进行中调用。
3. 用 `destroy_dev(sc->cdev)` 销毁主 cdev。主设备同上。
4. 用 `sysctl_ctx_free(9)` 拆除 sysctl 树。
5. 用 `mtx_destroy(9)` 销毁互斥锁。
6. 清除 `is_attached` 标志，以防仍有东西读取它。
7. 返回零。

注意步骤 2 和 3 各有两个目的。它们从 devfs 移除节点使没有新打开可以到达，它们排空进行中的调用使没有处理程序仍在运行当步骤 4 尝试释放处理程序会读取的状态时。

模式很简单。唯一出错的方法是在排空的 `destroy_dev` 完成之前释放东西。坚持这个顺序，你就会安全。

### 负载下的卸载

一个健康的直觉建设练习是推理当 `kldunload` 到达而用户态程序正在你的设备上的 `read(2)` 内时会发生什么。

按时间线走一遍：

- 内核开始卸载模块。它调用你的 `MOD_UNLOAD` 处理程序，最终在你的 Newbus 设备上调用 `device_delete_child`，这调用了你的 `detach`。
- 你的 `detach` 到达 `destroy_dev(sc->cdev)`。此调用是同步的，将等待进行中的处理程序完成。
- 用户态的 `read(2)` 当前正在执行你的 `d_read`。`si_threadcount` 是 1。
- `destroy_dev` 睡眠，监视 `si_threadcount`。
- 你的 `d_read` 返回。`si_threadcount` 降至 0。
- `destroy_dev` 返回。你的 `detach` 继续进行 sysctl 和互斥锁拆除。
- 用户态的 `read(2)` 已经将其字节返回给用户空间。描述符仍然打开。
- 同一进程中同一描述符上后续的 `read(2)` 现在干净地失败，因为 cdev 已经消失。

这就是"先销毁节点，然后拆除其依赖项"为你带来的。用户态可以观察到不一致状态的窗口被内核的排空行为做得极其微小。


## 持久策略：devfs.conf 和 devfs.rules

你的驱动修复每个节点的**基线**模式、所有者和组。持久的操作员侧调整属于 `/etc/devfs.conf` 和 `/etc/devfs.rules`。两个文件都是 FreeBSD 基本系统的标准部分，两者都应用于主机上的每个 devfs 挂载。

### devfs.conf：一次性、每路径调整

`devfs.conf` 是最简单的工具。每行在匹配设备节点出现时应用一次性调整。格式在 `devfs.conf(5)` 中有文档。常见指令是 `own`、`perm` 和 `link`：

```console
# /etc/devfs.conf
#
# Adjustments applied once when each node appears.

own     myfirst/0       root:operator
perm    myfirst/0       0660
link    myfirst/0       myfirst-primary
```

这三行说：每次 `/dev/myfirst/0` 出现时，chown 为 `root:operator`，设置其模式为 `0660`，并创建一个名为 `/dev/myfirst-primary` 的指向它的符号链接。在运行中的系统上重启 devfs 服务以应用更改：

```sh
% sudo service devfs restart
```

`devfs.conf` 对于小型稳定的实验设置没问题。它不是一个策略引擎。如果你需要条件规则或 jail 特有过滤，使用 `devfs.rules`。

### devfs.rules：基于规则的，用于 jail

`devfs.rules` 描述命名的规则集；每个规则集是模式和操作列表。jail 在其 `jail.conf(5)` 中按名称引用规则集，当 jail 自己的 devfs 挂载出现时，内核遍历匹配的规则集并过滤节点集。格式在 `devfs(8)` 和 `devfs.rules(5)` 中有文档。

一个小示例：

```text
# /etc/devfs.rules

[myfirst_lab=10]
add path 'myfirst/*' unhide
add path 'myfirst/*' mode 0660 group operator
```

这定义了一个编号为 `10`、名为 `myfirst_lab` 的规则集。它取消隐藏 `myfirst/` 下的任何节点（jail 默认隐藏节点），然后设置它们为 `operator` 组可读可写。要使用规则集，在 `jail.conf` 中命名它：

```ini
devfs_ruleset = 10;
```

我们不会在本章设置 jail。这里的重点是识别：当你在 jail 配置中看到 `devfs_ruleset` 或在操作员文档中看到 `service devfs restart` 时，你正在查看叠加在你驱动暴露之上的策略，而不是在驱动内部的策略。保持你的驱动在基线上诚实，让这些文件塑造操作员允许的内容。

### 完整的 devfs.conf 语法

`devfs.conf` 有一个小型稳定的语法。每行是一个指令。空行和以 `#` 开头的行被忽略。行中任何位置的 `#` 开始到行尾的注释。只存在三个指令关键字：

- **`own   path   user[:group]`**：将 `path` 的所有权更改为 `user`，如果给出 `:group` 也更改为该组。用户和组可以是密码数据库中存在的名称或数字 ID。
- **`perm  path   mode`**：将 `path` 的模式更改为给定的八进制模式。前导零可选但传统上使用。
- **`link  path   linkname`**：在 `/dev/linkname` 创建指向 `/dev/path` 的符号链接。

每个指令对相对于 `/dev` 给出的路径的节点操作。路径可以直接命名设备，也可以命名匹配设备族的 glob。glob 字符是 `*`、`?` 和括号中的字符类。

操作在节点首次出现在 `/dev` 下时应用。对于启动时存在的节点，那意味着在早期 `service devfs start` 阶段。对于稍后出现的节点（如驱动模块加载时），操作在匹配的 cdev 被添加到 devfs 时应用。

在运行中的系统上 `service devfs restart` 的效果是对 `/dev` 中当前存在的任何东西重新运行 `/etc/devfs.conf` 中的每个指令。这就是你如何将新添加的指令应用于已经存在的设备。

### devfs.rules 深入

`devfs.rules` 是一个不同的物种。它不是对路径应用一次性指令，而是定义 devfs 挂载可以引用的**命名规则集**。每个规则集是规则列表；每个规则按模式匹配路径并应用操作。

文件位于 `/etc/devfs.rules`，基本系统在 `/etc/defaults/devfs.rules` 发布默认值。格式在 `devfs.rules(5)` 和 `devfs(8)` 中有文档。

规则集由括号标题引入：

```text
[rulesetname=number]
```

`number` 是一个小整数，devfs 内部识别规则集的方式。`rulesetname` 是供 jail 配置使用的人类可读标签。标题后的规则属于该规则集直到下一个标题。

规则以 `add` 关键字开头并命名路径模式和操作。常见操作是：

- **`unhide`**：使匹配节点可见。从 `devfsrules_hide_all` 派生的规则集使用此来白名单特定节点集。
- **`hide`**：使匹配节点不可见。用于从默认集中移除某些东西。
- **`group name`**：更改匹配节点的组。
- **`user name`**：更改所有者。
- **`mode N`**：将模式更改为八进制 `N`。
- **`include $name`**：包含另一个名为 `$name` 的规则集的规则。

包含指令是 FreeBSD 发布规则集组合的方式。`devfsrules_jail` 规则集以 `add include $devfsrules_hide_all` 开始以建立干净的画布，然后包含 `devfsrules_unhide_basic` 为每个合理程序期望的少数节点，然后 `devfsrules_unhide_login` 为 PTY 和标准描述符，然后在其上添加几个 jail 特有路径。

### 一个完整的 jail 示例从头到尾

为了巩固理论，这里是一个读者可以在实验系统上应用的完整示例。它假设你已经构建并加载了第八章阶段 2 驱动，主机上存在 `/dev/myfirst/0`。

**步骤 1：在 `/etc/devfs.rules` 中定义规则集。** 添加到文件末尾：

```text
[myfirst_jail=100]
add include $devfsrules_jail
add path 'myfirst'   unhide
add path 'myfirst/*' unhide
add path 'myfirst/*' mode 0660 group operator
```

规则集编号为 `100`（任何未使用的小整数都可以；`100` 安全地高于发布编号）。它包含默认 jail 规则集，所以 jail 仍然有 `/dev/null`、`/dev/zero`、PTY 和普通 jail 需要的所有其他东西。然后它取消隐藏 `myfirst/` 目录和其中的节点，并设置它们的模式和组。

**步骤 2：创建 jail。** 一个最小的 `/etc/jail.conf` 条目：

```text
myfirstjail {
        path = "/jails/myfirstjail";
        host.hostname = "myfirstjail.example.com";
        mount.devfs;
        devfs_ruleset = 100;
        exec.start = "/bin/sh";
        exec.stop  = "/bin/sh -c 'exit'";
        persist;
}
```

创建 jail 根：

```sh
% sudo mkdir -p /jails/myfirstjail
% sudo bsdinstall jail /jails/myfirstjail
```

如果你已经有适合实验室的 jail 创建方法，用它替代 `bsdinstall`。

**步骤 3：启动 jail 并检查。**

```sh
% sudo service devfs restart
% sudo service jail start myfirstjail
% sudo jexec myfirstjail ls -l /dev/myfirst
total 0
crw-rw----  1 root  operator  0x5a Apr 17 10:00 0
```

节点以规则集指定的所有权和模式出现在 jail 内。如果规则集没有取消隐藏它，jail 会根本看不到 `myfirst` 目录。

**步骤 4：证明它。** 在 `/etc/devfs.rules` 中注释掉 `add path 'myfirst/*' unhide` 行，运行 `sudo service devfs restart`，并重新进入 jail：

```sh
% sudo jexec myfirstjail ls -l /dev/myfirst
ls: /dev/myfirst: No such file or directory
```

节点对 jail 不可见。主机仍然看到它。驱动没有重新加载。文件中的策略完全决定 jail 看到什么。

这个从头到尾的练习是实验 8.7 演练的内容。在散文中展示一次是为了建立模式：**规则集塑造 jail 看到什么，而驱动没有任何不同**。你的驱动的工作是暴露一个合理的基线；规则集的工作是按环境过滤和调整。


## 从用户态操作你的设备

Shell 工具会让你走得出乎意料地远。你在第七章中已经知道这些：

```sh
% ls -l /dev/myfirst/0
% sudo cat </dev/myfirst/0
% echo "hello" | sudo tee /dev/myfirst/0 >/dev/null
```

它们仍然有用，特别是 `ls -l` 用于确认权限更改生效。但在某个时候你会想从自己编写的程序中打开设备，这样你可以控制时序、测量行为并模拟现实的用户代码。`examples/part-02/ch08-working-with-device-files/userland/` 下的配套文件包含一个完全做这件事的小型探测程序。相关部分看起来像这样：

```c
#include <err.h>
#include <fcntl.h>
#include <stdio.h>
#include <unistd.h>

int
main(int argc, char **argv)
{
        const char *path = (argc > 1) ? argv[1] : "/dev/myfirst/0";
        char buf[64];
        ssize_t n;
        int fd;

        fd = open(path, O_RDWR);
        if (fd < 0)
                err(1, "open %s", path);

        n = read(fd, buf, sizeof(buf));
        if (n < 0)
                err(1, "read %s", path);

        printf("read %zd bytes from %s\n", n, path);

        if (close(fd) != 0)
                err(1, "close %s", path);

        return (0);
}
```

两件事要注意。首先，代码中没有设备特定的东西。这是与针对普通文件编写的相同的 `open`、`read`、`close`。这就是 UNIX 传统在发挥作用。其次，编译和运行此程序给你一个可重复的方式来驱动你的驱动，而不用担心 shell 引用。在第九章中，你将扩展它以写入数据、测量字节计数并跨描述符比较每次打开的状态。

对阶段 2 驱动运行一次应该产生类似：

```sh
% cc -Wall -Werror -o probe_myfirst probe_myfirst.c
% sudo ./probe_myfirst
read 0 bytes from /dev/myfirst/0
```

零字节，因为 `d_read` 仍然返回 EOF。数字很无聊；整个路径工作了的事实不无聊。

### 第二个探测：用 stat(2) 检查

读取设备节点的元数据与打开它一样有启发性。FreeBSD 的 `stat(1)` 命令和 `stat(2)` 系统调用都报告 devfs 公布的内容。一个围绕 `stat(2)` 构建的小型程序使得比较主节点和别名并确认它们解析到同一个 cdev 变得容易。

配套源码 `examples/part-02/ch08-working-with-device-files/userland/stat_myfirst.c` 看起来像这样：

```c
#include <err.h>
#include <stdio.h>
#include <sys/stat.h>
#include <sys/types.h>

int
main(int argc, char **argv)
{
        struct stat sb;
        int i;

        if (argc < 2) {
                fprintf(stderr, "usage: %s path [path ...]\n", argv[0]);
                return (1);
        }

        for (i = 1; i < argc; i++) {
                if (stat(argv[i], &sb) != 0)
                        err(1, "stat %s", argv[i]);
                printf("%s: mode=%06o uid=%u gid=%u rdev=%#jx\n",
                    argv[i],
                    (unsigned)(sb.st_mode & 07777),
                    (unsigned)sb.st_uid,
                    (unsigned)sb.st_gid,
                    (uintmax_t)sb.st_rdev);
        }
        return (0);
}
```

对主节点和别名运行它应该显示两个路径上相同的 `rdev`：

```sh
% sudo ./stat_myfirst /dev/myfirst/0 /dev/myfirst
/dev/myfirst/0: mode=020660 uid=0 gid=5 rdev=0x5a
/dev/myfirst:   mode=020660 uid=0 gid=5 rdev=0x5a
```

`rdev` 是 devfs 用来标记节点的标识符，它是两个名称真正指向同一底层 cdev 的最简单证明。模式中的 `020000` 高位说"字符特殊文件"；低位是熟悉的 `0660`。

### 第三个探测：并行打开

阶段 2 驱动允许多个进程同时持有设备打开，每个获得自己的每次打开结构。确认布线的一个好方法是运行一个从同一进程内多次打开节点的程序，持有每个描述符片刻，并报告发生了什么。

配套源码 `examples/part-02/ch08-working-with-device-files/userland/parallel_probe.c` 正是做到这一点：

```c
#include <err.h>
#include <fcntl.h>
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>

#define MAX_FDS 8

int
main(int argc, char **argv)
{
        const char *path = (argc > 1) ? argv[1] : "/dev/myfirst/0";
        int fds[MAX_FDS];
        int i, n;

        n = (argc > 2) ? atoi(argv[2]) : 4;
        if (n < 1 || n > MAX_FDS)
                errx(1, "count must be 1..%d", MAX_FDS);

        for (i = 0; i < n; i++) {
                fds[i] = open(path, O_RDWR);
                if (fds[i] < 0)
                        err(1, "open %s (fd %d of %d)", path, i + 1, n);
                printf("opened %s as fd %d\n", path, fds[i]);
        }

        printf("holding %d descriptors; press enter to close\n", n);
        (void)getchar();

        for (i = 0; i < n; i++) {
                if (close(fds[i]) != 0)
                        warn("close fd %d", fds[i]);
        }
        return (0);
}
```

运行它并同时观察 `dmesg`：

```sh
% sudo ./parallel_probe /dev/myfirst/0 4
opened /dev/myfirst/0 as fd 3
opened /dev/myfirst/0 as fd 4
opened /dev/myfirst/0 as fd 5
opened /dev/myfirst/0 as fd 6
holding 4 descriptors; press enter to close
```

你应该在 `dmesg` 中看到四个 `open via myfirst/0 fh=<ptr> (active=N)` 行，每个带有不同的指针。当你按 Enter 时，随着每个描述符关闭，四个 `per-open dtor fh=<ptr>` 行跟随。这是每次打开状态确实是每次描述符的最有力证据。

### 第四个探测：压力测试

短小的压力测试反复锻炼析构函数路径，捕获单打开测试会遗漏的泄漏。`examples/part-02/ch08-working-with-device-files/userland/stress_probe.c` 循环打开和关闭：

```c
#include <err.h>
#include <fcntl.h>
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>

int
main(int argc, char **argv)
{
        const char *path = (argc > 1) ? argv[1] : "/dev/myfirst/0";
        int iters = (argc > 2) ? atoi(argv[2]) : 1000;
        int i, fd;

        for (i = 0; i < iters; i++) {
                fd = open(path, O_RDWR);
                if (fd < 0)
                        err(1, "open (iter %d)", i);
                if (close(fd) != 0)
                        err(1, "close (iter %d)", i);
        }
        printf("%d iterations completed\n", iters);
        return (0);
}
```

对加载的驱动运行它，然后验证活动打开计数器回到零：

```sh
% sudo ./stress_probe /dev/myfirst/0 10000
10000 iterations completed
% sysctl dev.myfirst.0.stats.active_fhs
dev.myfirst.0.stats.active_fhs: 0
% sysctl dev.myfirst.0.stats.open_count
dev.myfirst.0.stats.open_count: 10000
```

如果 `active_fhs` 在程序退出后仍然在零以上，你的析构函数在某些路径上未能运行，你有一个真正的泄漏需要调查。如果 `open_count` 匹配迭代计数，每次打开都被看到了。压力探测是一个粗糙的工具，但它快速并捕获最常见的错误。


## 从设备文件视角阅读真正的 FreeBSD 驱动

没有什么比阅读必须解决你正在解决的相同问题的驱动更能巩固设备文件模型了。本节是 `/usr/src/sys` 中三个驱动的引导式巡览。目标不是理解每个驱动的全部。目标是看它们每个如何塑造其设备文件，这样你在脑中建立一个模式库。

每次巡览遵循相同的形状：打开文件，找到 `cdevsw`，找到 `make_dev` 调用，找到 `destroy_dev` 调用，注意什么是惯用的，什么是不寻常的。

### 巡览 1：/usr/src/sys/dev/null/null.c

`null` 模块是树中最小的良好示例。在编辑器中打开它。它足够短可以一次读完。

首先注意：一个文件中有**三个** `cdevsw` 结构。

```c
static struct cdevsw full_cdevsw = {
        .d_version =    D_VERSION,
        .d_read =       zero_read,
        .d_write =      full_write,
        .d_ioctl =      zero_ioctl,
        .d_name =       "full",
};

static struct cdevsw null_cdevsw = {
        .d_version =    D_VERSION,
        .d_read =       (d_read_t *)nullop,
        .d_write =      null_write,
        .d_ioctl =      null_ioctl,
        .d_name =       "null",
};

static struct cdevsw zero_cdevsw = {
        .d_version =    D_VERSION,
        .d_read =       zero_read,
        .d_write =      null_write,
        .d_ioctl =      zero_ioctl,
        .d_name =       "zero",
        .d_flags =      D_MMAP_ANON,
};
```

三个不同的节点，三个不同的 `cdevsw` 值，没有 softc。模块在其 `MOD_LOAD` 处理程序中注册三个 cdev：

```c
full_dev = make_dev_credf(MAKEDEV_ETERNAL_KLD, &full_cdevsw, 0,
    NULL, UID_ROOT, GID_WHEEL, 0666, "full");
null_dev = make_dev_credf(MAKEDEV_ETERNAL_KLD, &null_cdevsw, 0,
    NULL, UID_ROOT, GID_WHEEL, 0666, "null");
zero_dev = make_dev_credf(MAKEDEV_ETERNAL_KLD, &zero_cdevsw, 0,
    NULL, UID_ROOT, GID_WHEEL, 0666, "zero");
```

注意 `MAKEDEV_ETERNAL_KLD`。当此代码静态编译到内核中时，宏扩展为 `MAKEDEV_ETERNAL` 并将 cdev 标记为永不销毁。当相同代码作为可加载模块构建时，宏扩展为零，cdev 可以在卸载期间被销毁。

还要注意模式 `0666` 和 `root:wheel`。null 模块服务的所有东西都被故意设置为每个人都可以访问。

卸载与加载一样简单：

```c
destroy_dev(full_dev);
destroy_dev(null_dev);
destroy_dev(zero_dev);
```

每个 cdev 一个 `destroy_dev`。没有要拆除的 softc。没有每次打开的状态。没有超出内核提供的锁定。这就是最小化的样子。

**从 null 中复制什么：** 设置 `d_version` 的习惯，给每个 `cdevsw` 自己的 `d_name` 的习惯，加载和卸载之间的对称性，使用简单命名处理程序而不是发明抽象的意愿。

**从 null 中不碰什么：** `MAKEDEV_ETERNAL_KLD`。你的驱动应该是可卸载的，所以你不想要永恒标志。`null` 模块是特殊的，因为它创建的节点早于几乎所有其他内核子系统，并且预期在内核的生命周期内保持存活。

### 巡览 2：/usr/src/sys/dev/led/led.c

LED 框架在结构复杂性上上升了一级。它仍然足够小可以在一次坐下来读完。`null` 没有 softc，`led` 有完整的每 LED softc。`null` 创建三个单例，`led` 按需为每个 LED 创建一个 cdev。

首先看单个 `cdevsw`：

```c
static struct cdevsw led_cdevsw = {
        .d_version =    D_VERSION,
        .d_write =      led_write,
        .d_name =       "LED",
};
```

所有 LED 一个 `cdevsw`。框架将它用于它创建的每个 cdev，依靠 `si_drv1` 来区分它们。这个定义的极简主义本身就是一课：`led` 不实现 `d_open`、`d_close` 或 `d_read`，因为操作员与 LED 的每次交互都是用 `echo` 写入的模式字符串。从节点读取没有意义，打开时也不需要跟踪会话状态，所以驱动简单地让这些字段未设置。devfs 将每个 `NULL` 槽位解释为"使用默认行为"，这对 `d_read` 是返回零字节，对 `d_open` 和 `d_close` 是什么都不做。当你设计自己的 `cdevsw` 值时，记住这一点：只填充你的设备真正需要的，其余的不要动。

每 LED softc 存在于文件顶部附近定义的 `struct ledsc` 中：

```c
struct ledsc {
        LIST_ENTRY(ledsc)       list;
        char                    *name;
        void                    *private;
        int                     unit;
        led_t                   *func;
        struct cdev             *dev;
        /* ... more state ... */
};
```

它在 `dev` 字段中承载到其 cdev 的反向指针，以及从 `unrhdr(9)` 池而不是 Newbus 分配的单元号：

```c
sc->unit = alloc_unr(led_unit);
```

实际的 `make_dev` 调用就在下面：

```c
sc->dev = make_dev(&led_cdevsw, sc->unit,
    UID_ROOT, GID_WHEEL, 0600, "led/%s", name);
```

注意路径：`"led/%s"`。框架创建的每个 LED 都落在 `/dev/led/` 子目录中，使用调用驱动选择的自由格式名称（例如 `led/ehci0`）。这就是框架如何保持其节点分组的。

在 `make_dev` 之后，框架立即存储 softc 指针：

```c
sc->dev->si_drv1 = sc;
```

这是 `mda_si_drv1` 之前的方式，早于 `make_dev_s`。新驱动应该通过 args 结构传递 `mda_si_drv1`，这样指针在 cdev 变得可达之前就被设置了。

销毁是一次调用：

```c
destroy_dev(dev);
```

简单。同步。调用者层面没有延迟销毁，没有排空循环。框架依赖内核在 `destroy_dev` 中的排空行为来完成任何进行中的处理程序。

**从 led 中复制什么：** 命名约定（每个框架一个子目录），softc 布局（反向指针加身份字段加回调指针），用于不是来自 Newbus 的单元号的干净的 `alloc_unr`/`free_unr` 模式。

**从 led 中不碰什么：** `make_dev` 后的 `sc->dev->si_drv1 = sc` 赋值。使用 `make_dev_s` 中的 `mda_si_drv1` 替代。

### 巡览 3：/usr/src/sys/dev/md/md.c

内存磁盘驱动比前两个大，其大部分内容与设备文件无关。它是关于 GEOM 的、关于后备存储的、关于 swap 支持和 vnode 支持的实例的。出于我们的目的，我们看一个特定的东西：控制节点 `/dev/mdctl`。

在 `md.c` 顶部附近找到 `cdevsw` 声明：

```c
static struct cdevsw mdctl_cdevsw = {
        .d_version =    D_VERSION,
        .d_ioctl =      mdctlioctl,
        .d_name =       MD_NAME,
};
```

只设置了两个字段。`d_version` 和 `d_ioctl` 和一个名称。没有 `d_open`、`d_close`、`d_read` 或 `d_write`。控制节点专门通过 `ioctl(2)` 使用：创建 md、附加后备存储、销毁 md。这是树中许多控制接口的形状。

cdev 在文件底部附近创建：

```c
status_dev = make_dev(&mdctl_cdevsw, INT_MAX, UID_ROOT, GID_WHEEL,
    0600, MDCTL_NAME);
```

`INT_MAX` 是当单元号不重要时单例的常见模式：它将 cdev 放在驱动实例单元号的任何合理范围之外。`0600` 和 `root:wheel` 是你期望的具有真正特权的控制节点的狭窄基线。

销毁发生在模块卸载路径中：

```c
destroy_dev(status_dev);
```

又一次，一次调用。

**从 md 中复制什么：** 为数据路径在别处（在 md 的情况下，在 GEOM 中）的子系统暴露单一控制 cdev 的模式，以及具有真正特权的节点的非常窄的权限。

**从 md 中不碰什么：** `md` 是一个大型子系统；不要试图将其结构复制为模板。复制控制节点想法；将 GEOM 分层留给第二十七章。

### 四个驱动的综合

经过四次巡览后，将共性和差异排列在一处是有帮助的。每一行是驱动属性；每一列是一个驱动。

| 属性                  | `null`       | `led`          | `md`             | `bpf`             |
|---------------------------|--------------|----------------|------------------|-------------------|
| 多少个 `cdevsw` 值  | 3            | 1              | 1（加上 GEOM）    | 1                 |
| 每次挂载的 cdev          | 总共 3      | 每个 LED 1     | 1 个控制 + 多个 | 1 个加上 1 个别名    |
| Softc？                    | 否           | 是            | 是              | 是（每次打开）    |
| /dev 中的子目录？     | 否           | 是 (`led/*`)  | 否               | 否               |
| 权限模式           | `0666`       | `0600`         | `0600`           | `0600`            |
| 使用 `devfs_set_cdevpriv`？| 否           | 否             | 否               | 是               |
| 使用克隆？             | 否           | 否             | 否               | 否               |
| 使用 `make_dev_alias`？    | 否           | 否             | 否               | 是               |
| `d_close` 已填充？      | 否           | 否             | 否               | 否               |
| 使用 `destroy_dev_drain`？ | 否           | 否             | 否               | 否               |
| 主要用例          | 伪数据  | 硬件控制   | 子系统控制    | 数据包捕获    |

每一列都是可辩护的。每个驱动都选择了适合其工作的最简单的特性集。到第八章结束时你的 `myfirst` 驱动的概况更接近 `led` 而不是其他的：一个 `cdevsw`、每次实例的 softc、子目录命名、窄权限，加上用于每次打开状态的 `devfs_set_cdevpriv`（`led` 不需要）和别名（`led` 不使用）。

那个概况是一个好的位置。它足够大以表明你参与了真正的机制，足够小以至于驱动的每一行都有存在的理由。


## 动手实验

这些实验就地扩展第七章驱动。你不应该需要从头重新输入任何东西。配套目录镜像了各个阶段。

### 实验 8.1：结构化名称和更严格的权限

**目标。** 将设备从 `/dev/myfirst0` 移动到 `/dev/myfirst/0`，并将组更改为 `operator`，模式 `0660`。

**步骤.**

1. 在 `myfirst_attach()` 中，将 `make_dev_s()` 格式字符串更改为 `"myfirst/%d"`。
2. 将 `args.mda_gid` 从 `GID_WHEEL` 更改为 `GID_OPERATOR`，将 `args.mda_mode` 从 `0600` 更改为 `0660`。
3. 重建并重新加载：

   ```sh
   % make clean && make
   % sudo kldload ./myfirst.ko
   % ls -l /dev/myfirst
   total 0
   crw-rw----  1 root  operator  0x5a Apr 17 09:41 0
   ```

4. 确认 `operator` 组中的普通用户现在可以在没有 `sudo` 的情况下从节点读取。在 FreeBSD 上，你用 `pw groupmod operator -m yourname` 将用户添加到该组，然后启动一个新的 shell。
5. 卸载驱动并确认 `/dev/myfirst/` 目录随节点一起消失。

**成功标准。**

- `/dev/myfirst/0` 在加载时出现，在卸载时消失。
- `ls -l /dev/myfirst/0` 显示 `crw-rw----  root  operator`。
- `operator` 组成员可以 `cat </dev/myfirst/0` 而无错误。

### 实验 8.2：添加别名

**目标.** 将 `/dev/myfirst` 暴露为 `/dev/myfirst/0` 的别名。

**步骤.**

1. 向 softc 添加 `struct cdev *cdev_alias` 字段。
2. 在 `myfirst_attach()` 中成功调用 `make_dev_s()` 后，调用：

   ```c
   sc->cdev_alias = make_dev_alias(sc->cdev, "myfirst");
   if (sc->cdev_alias == NULL)
           device_printf(dev, "failed to create alias\n");
   ```

3. 在 `myfirst_detach()` 中，在销毁主 cdev 之前销毁别名：

   ```c
   if (sc->cdev_alias != NULL) {
           destroy_dev(sc->cdev_alias);
           sc->cdev_alias = NULL;
   }
   if (sc->cdev != NULL) {
           destroy_dev(sc->cdev);
           sc->cdev = NULL;
   }
   ```

4. 重建、重新加载并验证：

   ```sh
   % ls -l /dev/myfirst /dev/myfirst/0
   ```

   两个路径都应该响应。`sudo cat </dev/myfirst` 和 `sudo cat </dev/myfirst/0` 应该行为相同。

**成功标准.**

- 驱动加载时两个路径都存在。
- 卸载时两个路径都消失。
- 如果别名创建失败，驱动不会 panic 或泄漏；暂时注释掉 `make_dev_alias` 行以确认这一点。

### 实验 8.3：每次打开状态

**目标.** 给每次 `open(2)` 自己的小型结构，并从用户态验证两个描述符看到独立的数据。

**步骤.**

1. 添加如本章前面所示的 `struct myfirst_fh` 类型和 `myfirst_fh_dtor()` 析构函数。
2. 重写 `myfirst_open()` 以分配 `myfirst_fh`，调用 `devfs_set_cdevpriv()`，并在注册失败时释放。移除独占打开检查。
3. 重写 `myfirst_read()` 和 `myfirst_write()`，使每个都以调用 `devfs_get_cdevpriv(&fh)` 开始。暂时保持主体不变；第九章会填充它。
4. 重建、重新加载，然后并排运行两个 `probe_myfirst` 进程：

   ```sh
   % (sudo ./probe_myfirst &) ; sudo ./probe_myfirst
   ```

5. 在 `dmesg` 中，确认两个 `open (per-open fh=...)` 消息显示不同的指针。

**成功标准.**

- 两个同时打开成功。没有 `EBUSY`。
- 内核日志中出现两个不同的 `fh=` 指针。
- `kldunload myfirst` 只在两个探测都退出后才可能。

### 实验 8.4：devfs.conf 持久化

**目标.** 使实验 8.1 中的所有权更改在重启后保留，而不再次编辑驱动。

**步骤.**

1. 在实验 8.1 中，将 `args.mda_gid` 和 `args.mda_mode` 恢复为第七章默认值（`GID_WHEEL`、`0600`）。
2. 创建或编辑 `/etc/devfs.conf` 并添加：

   ```
   own     myfirst/0       root:operator
   perm    myfirst/0       0660
   ```

3. 在不重启的情况下应用更改：

   ```sh
   % sudo service devfs restart
   ```

4. 重新加载驱动并确认 `ls -l /dev/myfirst/0` 再次显示 `root  operator  0660`，即使驱动本身请求的是 `root  wheel  0600`。

**成功标准.**

- 加载驱动且 `devfs.conf` 就位时，节点显示 `devfs.conf` 值。
- 加载驱动且注释掉 `devfs.conf` 行并重启 devfs 时，节点返回驱动的基线。

**注意.** 实验 8.4 是操作员侧实验。驱动在步骤之间不改变。重点是看到双层策略模型在工作：驱动设定基线，`devfs.conf` 塑造视图。

### 实验 8.5：双节点驱动（数据和控制）

**目标.** 扩展 `myfirst` 以暴露两个不同的节点：位于 `/dev/myfirst/0` 的数据节点和位于 `/dev/myfirst/0.ctl` 的控制节点，每个有自己的 `cdevsw` 和自己的权限模式。

**前提条件.** 完成实验 8.3（带有每次打开状态的阶段 2）。

**步骤.**

1. 在驱动中定义第二个 `struct cdevsw`，命名为 `myfirst_ctl_cdevsw`，设置 `d_name = "myfirst_ctl"` 并只留下 `d_ioctl` 桩函数（你不会实现 ioctl 命令；只需让函数存在并返回 `ENOTTY`）。
2. 向 softc 添加 `struct cdev *cdev_ctl` 字段。
3. 在 `myfirst_attach` 中，创建数据节点后，用第二个 `make_dev_s` 调用创建控制节点。使用 `"myfirst/%d.ctl"` 作为格式。将模式设置为 `0640`，组设置为 `GID_WHEEL`，使控制节点比数据节点更窄。
4. 也通过 `mda_si_drv1` 为控制 cdev 传递 `sc`，这样 `d_ioctl` 可以找到它。
5. 在 `myfirst_detach` 中，**在**数据 cdev **之前**销毁控制 cdev。记录每次销毁。
6. 重建、重新加载并验证：

   ```sh
   % ls -l /dev/myfirst
   total 0
   crw-rw----  1 root  operator  0x5a Apr 17 10:02 0
   crw-r-----  1 root  wheel     0x5b Apr 17 10:02 0.ctl
   ```

**成功标准.**

- 两个节点在加载时出现。
- 两个节点在卸载时消失。
- 数据节点可被 `operator` 组读取；控制节点不能。
- 非 root 非 wheel 用户尝试 `cat </dev/myfirst/0.ctl` 因 `Permission denied` 失败。

**注意.** 在真正的驱动中，控制节点是 `ioctl` 配置命令所在的地方。本章不实现任何 `ioctl` 命令；那是第二十五章的工作。实验 8.5 的重点是展示你可以在一个驱动中有两个具有不同策略的节点。

### 实验 8.6：并行探测验证

**目标.** 使用配套树中的 `parallel_probe` 工具证明每次打开状态确实是每次描述符的。

**前提条件.** 完成实验 8.3。阶段 2 驱动已加载。

**步骤.**

1. 构建用户态工具：

   ```sh
   % cd examples/part-02/ch08-working-with-device-files/userland
   % make
   ```

2. 用四个描述符运行 `parallel_probe`：

   ```sh
   % sudo ./parallel_probe /dev/myfirst/0 4
   opened /dev/myfirst/0 as fd 3
   opened /dev/myfirst/0 as fd 4
   opened /dev/myfirst/0 as fd 5
   opened /dev/myfirst/0 as fd 6
   holding 4 descriptors; press enter to close
   ```

3. 打开第二个终端并检查 `dmesg`：

   ```sh
   % dmesg | tail -20
   ```

   你应该看到四个 `open via myfirst/0 fh=<pointer> (active=N)` 行，每个带有不同的指针值。

4. 在第二个终端中，检查活动打开 sysctl：

   ```sh
   % sysctl dev.myfirst.0.stats.active_fhs
   dev.myfirst.0.stats.active_fhs: 4
   ```

5. 回到第一个终端并按 Enter。探测关闭所有四个描述符。再次检查 `dmesg`：

   ```sh
   % dmesg | tail -10
   ```

   你应该看到四个 `per-open dtor fh=<pointer>` 行，每个描述符一个，具有与打开日志中出现的相同指针值。

6. 验证 `active_fhs` 回到零：

   ```sh
   % sysctl dev.myfirst.0.stats.active_fhs
   dev.myfirst.0.stats.active_fhs: 0
   ```

**成功标准.**

- 打开日志中有四个不同的 `fh=` 指针。
- 析构函数日志中有四个匹配的指针。
- `active_fhs` 递增到四并递减回零。
- 没有关于内存泄漏或意外状态的内核日志消息。

**注意.** 实验 8.6 是你能轻易产生的每次打开状态被隔离的最有力证据。如果任何步骤失败，最常见的罪魁祸首是错过了对 `devfs_set_cdevpriv` 的调用或没有递减 `active_fhs` 的析构函数。


### 实验 8.7：jail 的 devfs.rules

**目标.** 通过 devfs 规则集使 `/dev/myfirst/0` 在 jail 内可见。

**前提条件.** 你的实验系统上有一个可用的 FreeBSD jail。如果还没有，跳过此实验，在第七部分之后返回。

**步骤.**

1. 向 `/etc/devfs.rules` 添加规则集：

   ```
   [myfirst_jail=100]
   add include $devfsrules_jail
   add path 'myfirst'   unhide
   add path 'myfirst/*' unhide
   add path 'myfirst/*' mode 0660 group operator
   ```

2. 向 jail 的 `jail.conf` 添加 devfs 条目：

   ```
   myfirstjail {
           path = "/jails/myfirstjail";
           host.hostname = "myfirstjail.example.com";
           mount.devfs;
           devfs_ruleset = 100;
           exec.start = "/bin/sh";
           persist;
   }
   ```

3. 重新加载 devfs 并启动 jail：

   ```sh
   % sudo service devfs restart
   % sudo service jail start myfirstjail
   ```

4. 在 jail 内，确认节点：

   ```sh
   % sudo jexec myfirstjail ls -l /dev/myfirst
   ```

5. 通过注释掉 `add path 'myfirst/*' unhide` 行、重启 devfs 和 jail 来验证规则集工作，并观察节点消失。

**成功标准.**

- `/dev/myfirst/0` 以模式 `0660` 和组 `operator` 出现在 jail 内。
- 移除 unhide 规则会从 jail 内移除节点。
- 主机无论 jail 的规则集如何都继续看到节点。

**注意.** jail 配置通常在后面的章节中讨论；此实验是预览，以演示驱动侧结果。如果在你的系统上实验困难，在你为其他目的配置了 jail 之后再回来。

## 挑战练习

这些在实验基础上构建。慢慢来；它们中没有引入新机制，只是延伸了你刚练习的机制。

### 挑战 1：使用别名

更改 `probe_myfirst.c` 以默认打开 `/dev/myfirst` 而不是 `/dev/myfirst/0`。从内核日志确认你的 `d_open` 运行了，并且 `devfs_set_cdevpriv` 每次 `open(2)` 恰好成功一次。然后将路径改回来。你不应该需要编辑驱动。

### 挑战 2：观察每次打开的清理

在 `myfirst_fh_dtor()` 内部添加一个 `device_printf` 来记录正在释放的 `fh` 指针。运行 `probe_myfirst` 一次并确认每次运行在 `dmesg` 中恰好出现一行析构函数。然后编写一个小程序，打开设备，睡眠 30 秒，然后在不调用 `close(2)` 的情况下退出。确认进程退出时析构函数仍然触发。清理不是客气；它是保证。

### 挑战 3：实验 devfs.rules

如果你已经配置了 FreeBSD jail，向 `/etc/devfs.rules` 添加一个 `myfirst_lab` 规则集，使 `/dev/myfirst/*` 在 jail 内可见。启动 jail，从内部打开设备，并确认驱动看到了新的打开。如果你还没有 jail，现在跳过此挑战，在第七部分之后返回。

### 挑战 4：阅读两个更多驱动

从 `/usr/src/sys/dev/` 中选择两个你还没有阅读的驱动。好的候选有 `/usr/src/sys/dev/random/randomdev.c`、`/usr/src/sys/dev/hwpmc/hwpmc_mod.c`、`/usr/src/sys/dev/kbd/kbd.c`，或其他任何足够短可以浏览的。对每个驱动，找出：

- `cdevsw` 定义及其 `d_name`。
- `make_dev*` 调用及其设置的权限模式。
- `destroy_dev` 调用，或它们的缺失。
- 驱动是否使用 `devfs_set_cdevpriv`。
- 驱动是否在 `/dev` 下创建子目录。

为每个驱动写一小段分类其设备文件界面。重点是磨练你的眼光；没有单一正确的分类法。

### 挑战 5：devd 配置

编写一个最小的 `/etc/devd.conf` 规则，在 `/dev/myfirst/0` 每次出现或消失时记录一条消息。devd 配置格式在 `devd.conf(5)` 中有文档。起始模板：

```text
notify 100 {
        match "system"      "DEVFS";
        match "subsystem"   "CDEV";
        match "cdev"        "myfirst/0";
        action              "/usr/bin/logger -t myfirst event=$type";
};
```

安装规则，重启 devd（`service devd restart`），加载和卸载驱动，然后验证 `grep myfirst /var/log/messages` 显示两个事件。

### 挑战 6：添加状态节点

修改 `myfirst` 以在数据节点旁边暴露一个只读状态节点。状态节点位于 `/dev/myfirst/0.status`，模式 `0444`，所有者 `root:wheel`。它的 `d_read` 返回一个简短的纯文本字符串，总结驱动当前状态：

```ini
attached_at=12345
active_fhs=2
open_count=17
```

提示：在 softc 中分配一个小型固定大小缓冲区，在互斥锁下格式化字符串，如果你已经读过第九章就用 `uiomove(9)` 返回给用户，或者现在用手动实现。

如果你对 `uiomove` 还不熟悉，将此挑战推迟到第九章之后。这是第九章所教内容的自然首次使用。

## 设备文件操作的错误码

你的 `d_open` 和 `d_close` 返回的每个非零值都告诉 devfs 一些具体内容。你选择的 errno 值是你的驱动与曾经触及你的节点的每个用户程序之间的契约。正确选择不花任何代价；选择错误会产生你初次阅读时无法理解的错误报告。

本节调查在设备文件界面上实际出现的 errno 值。第九章将单独处理 `d_read` 和 `d_write` 的 errno 选择，因为数据路径的选择在性质上不同。这里我们专注于打开、关闭和 ioctl 相邻返回。

### 简短列表

按你使用频率的大致顺序：

- **`ENXIO`（没有此类设备或地址）**："设备不处于可以被打开的状态。"当驱动已挂载但未准备好、已知硬件缺失、softc 处于瞬态时使用。用户看到 `Device not configured`。
- **`EBUSY`（设备忙）**："设备已经打开，此驱动不允许并发访问。"用于独占打开策略。用户看到 `Device busy`。
- **`EACCES`（权限被拒绝）**："呈现此打开的凭证不被允许。"内核通常在你的处理程序运行之前捕获权限失败，但驱动可以检查二级策略（例如，一个拒绝为读取而打开的仅 `ioctl` 节点）并自己返回 `EACCES`。
- **`EPERM`（操作不允许）**："操作需要调用者没有的特权。"精神上类似于 `EACCES`，但针对特权区别（`priv_check(9)` 失败）而不是 UNIX 文件权限。
- **`EINVAL`（无效参数）**："调用结构有效但驱动不接受这些参数。"当 `oflags` 指定驱动拒绝的组合时使用。
- **`EAGAIN`（资源暂时不可用）**："设备原则上可以打开，但现在不行。"当你有临时短缺（槽位已满、资源正在重新配置）且用户应稍后重试时使用。用户看到 `Resource temporarily unavailable`。
- **`EINTR`（被中断的系统调用）**：当处理程序内部的睡眠被信号中断时返回。你通常不会从 `d_open` 返回这个，因为打开通常不会可中断地睡眠。它在数据路径处理程序中更常见。
- **`ENOENT`（没有此类文件或目录）**：几乎总是由 devfs 本身在路径无法解析时合成。驱动很少从自己的处理程序返回这个。
- **`ENODEV`（设备不支持此操作）**："操作本身有效但此设备不支持它。"当驱动的二级接口拒绝另一个接口支持的操作时使用。
- **`EOPNOTSUPP`（操作不支持）**：`ENODEV` 的表亲。在某些子系统中用于类似情况。

### 哪个值用于哪种情况？

真正的驱动落入模式。以下是你最常编写的模式。

**模式 A：驱动已挂载但 softc 尚未准备好。** 你可能在两阶段挂载中遇到这个，cdev 在某些初始化完成之前创建，或者在拆离期间 cdev 仍然存在时。

```c
if (sc == NULL || !sc->is_attached)
        return (ENXIO);
```

**模式 B：独占打开策略。**

```c
mtx_lock(&sc->mtx);
if (sc->is_open) {
        mtx_unlock(&sc->mtx);
        return (EBUSY);
}
sc->is_open = 1;
mtx_unlock(&sc->mtx);
```

这是第七章所做的。第八章的阶段 2 移除了独占检查，因为每次打开状态可用；`EBUSY` 不再需要。

**模式 C：只读节点拒绝写入。**

```c
if ((oflags & FWRITE) != 0)
        return (EACCES);
```

当节点概念上是只读的，为写入而打开是调用者错误时使用。

**模式 D：仅特权接口。**

```c
if (priv_check(td, PRIV_DRIVER) != 0)
        return (EPERM);
```

当非特权调用者尝试打开在文件系统模式之外强制执行额外特权检查的节点时返回 `EPERM`。

**模式 E：暂时不可用。**

```c
if (sc->resource_in_flight) {
        return (EAGAIN);
}
```

当驱动稍后可以接受打开但现在不行，用户应重试时使用。

### 从 d_close 返回错误

`d_close` 有自己的考虑。内核通常不关心 close 的错误，因为当 `close(2)` 返回给用户态时，描述符已经消失了。但 close 仍然是你最后一次注意到失败并记录它的机会，一些调用者可能会检查。最安全的模式是：

- 从普通关闭路径返回零。
- 只有当真正不寻常的事情发生且用户态应该知道时才返回非零 errno。
- 有疑问时，用 `device_printf(9)` 记录并返回零。

一个从 `d_close` 返回随机错误的驱动是一个测试会神秘失败的驱动，因为大多数用户态代码忽略关闭错误。把 errno 留给 open 和 ioctl，在那里它很重要。


## 检查 /dev 的工具

几个小型实用工具值得了解，因为一旦你到达第九章，你将依赖它们快速确认行为。本节以足够使用的深度介绍每一个，并以两个简短的故障排除演练结束。

### ls -l 用于权限和存在性

第一站。`ls -l /dev/yourpath` 确认存在性、类型、所有权和模式。如果加载后节点缺失，你的 `make_dev_s` 可能失败了；检查 `dmesg` 获取错误码。

devfs 目录上的 `ls -l` 工作方式与你期望的一样：`ls -l /dev/myfirst` 列出子目录中的条目。结合 `-d`，它报告目录本身：

```sh
% ls -ld /dev/myfirst
dr-xr-xr-x  2 root  wheel  512 Apr 17 10:02 /dev/myfirst
```

devfs 子目录的模式默认是 `0555`，它不能通过 `devfs.conf` 直接配置。子目录存在只是因为里面至少有一个节点；当里面最后一个节点消失时，目录也消失。

### stat 和 stat(1)

`stat(1)` 打印任何节点的结构化视图。默认输出冗长，包括时间戳。更有用的形式是自定义格式：

```sh
% stat -f '%Sp %Su %Sg %T %N' /dev/myfirst/0
crw-rw---- root operator Character Device /dev/myfirst/0
```

占位符在 `stat(1)` 中有文档。上面五个是权限、用户名、组名、文件类型描述和路径。这种形式在需要稳定文本表示的脚本中很有用。

为了比较两个路径以检查它们解析到同一个 cdev，`stat -f '%d %i %Hr,%Lr'` 打印文件系统的设备、inode 和 `rdev` 的主次组件。在引用同一 cdev 的两个 devfs 节点上，`rdev` 组件将匹配。

### fstat(1)：谁打开了它？

`fstat(1)` 列出系统上每个打开的文件。过滤到设备路径，它告诉你哪些进程打开了该节点：

```sh
% fstat /dev/myfirst/0
USER     CMD          PID   FD MOUNT      INUM MODE         SZ|DV R/W NAME
root     probe_myfir  1234    3 /dev          4 crw-rw----   0,90 rw  /dev/myfirst/0
```

这是解决"`kldunload` 返回 `EBUSY` 但我不知道为什么"谜题的工具。针对你的节点运行它，识别有问题的进程，要么等它完成要么终止它。

`fstat -u username` 按用户过滤，当你怀疑特定用户的守护进程持有节点时有用。`fstat -p pid` 检查一个进程。

### procstat -f：进程优先视图

`fstat(1)` 列出文件并告诉你谁持有它们。`procstat -f pid` 做逆操作：它列出给定进程持有的文件。当你有一个运行程序的 PID 并想确认它当前打开了哪些设备节点时，这就是工具：

```sh
% procstat -f 1234
  PID COMM                FD T V FLAGS    REF  OFFSET PRO NAME
 1234 probe_myfirst        3 v c rw------   1       0     /dev/myfirst/0
```

列 `T` 显示文件类型（`v` 表示 vnode，包括设备文件），列 `V` 显示 vnode 类型（`c` 表示字符设备 vnode）。这是确认调试器显示给你的内容的最快方式。

### devinfo(8)：Newbus 侧

`devinfo(8)` 根本不看 devfs。它遍历 Newbus 设备树并打印设备层次结构。你的 `nexus0` 的 `myfirst0` 子设备在那里显示，无论 cdev 是否存在：

```sh
% devinfo -v
nexus0
  myfirst0
  pcib0
    pci0
      <...lots of PCI children...>
```

这是当 `/dev` 中缺少某些东西时你使用的工具，你需要检查设备本身是否挂载了。如果 `devinfo` 显示 `myfirst0` 但 `ls /dev` 不显示，你的 `make_dev_s` 失败了。如果两者都不显示设备，你的 `device_identify` 或 `device_probe` 没有创建子设备。两个不同的错误，两个不同的修复。

`-r` 标志过滤到以特定设备为根的 Newbus 层次结构，在有大量 PCI 设备的复杂系统中变得有用。

### devfs(8)：规则集和规则

`devfs(8)` 是 devfs 规则集的低级管理接口。你在第 10 节遇到了它。三个子命令经常出现：

- `devfs rule showsets` 列出当前加载的规则集编号。
- `devfs rule -s N show` 打印规则集 `N` 内的规则。
- `devfs rule -s N add path 'pattern' action args` 在运行时添加规则。

运行时添加的规则不持久；要使它们永久，添加到 `/etc/devfs.rules` 并运行 `service devfs restart`。

### dmesg：发生了什么的日志

驱动的每个 `device_printf` 和 `printf` 调用最终进入内核消息缓冲区，`dmesg`（或 `dmesg -a`）打印它。当这个界面上出了问题时，`dmesg` 是首先查看的地方：

```sh
% dmesg | tail -20
```

你的挂载和拆离消息、任何 `make_dev_s` 失败、以及销毁路径的任何 panic 消息都落在这里。养成在开发期间用打开 `tail -f /var/log/messages` 的第二个终端监视 `dmesg` 的习惯。

### 故障排除演练 1：节点缺失

"我期望 `/dev/myfirst/0` 存在但它不存在"的清单。

1. 模块加载了吗？`kldstat | grep myfirst`。
2. 挂载运行了吗？`devinfo -v | grep myfirst`。
3. `make_dev_s` 成功了吗？`dmesg | tail` 应该显示你的挂载成功消息。
4. devfs 挂载在 `/dev` 上了吗？`mount | grep devfs`。
5. 你在正确的路径查看吗？如果你的格式字符串是 `"myfirst%d"`，节点是 `/dev/myfirst0`，不是 `/dev/myfirst/0`。拼写错误时有发生。
6. `devfs.rules` 条目在隐藏节点吗？`devfs rule showsets` 并检查。

十次中有九次，前三个问题之一就能得出答案。

### 故障排除演练 2：kldunload 返回 EBUSY

"我可以加载模块但不能卸载它"的清单。

1. 节点仍然打开吗？`fstat /dev/myfirst/0` 显示持有者。
2. 你的拆离自己返回 `EBUSY` 吗？检查 `dmesg` 中来自驱动的消息。阶段 2 的拆离在 `active_fhs > 0` 时返回 `EBUSY`。
3. `devfs.conf` 的 `link` 指向你的节点吗？如果目标被持有打开，链接可以保持引用。
4. 内核线程卡在你的处理程序之一内部了吗？在 `dmesg` 中寻找 `Still N threads in foo` 消息。如果存在，你需要一个 `d_purge`。

大多数 `EBUSY` 是打开的描述符。其他情况很少见。

## 陷阱和值得注意的事项

一份捕捉初学者最常见的错误的实地指南。每个都命名了症状、原因和治疗方法。

- **在 softc 准备好之前创建设备节点。** *症状：* 驱动加载后立即打开导致 NULL 解引用。*原因：* `si_drv1` 仍然未设置，或 `open()` 查询的 softc 字段尚未初始化。*治疗方法：* 在 `make_dev_args` 中设置 `mda_si_drv1` 并在 `make_dev_s` 调用之前完成 softc 字段。把 `make_dev_s` 想成是发布，而不是准备。
- **在设备节点之前销毁 softc。** *症状：* 在 `kldunload` 期间或之后偶尔 panic。*原因：* 在 `detach()` 中颠倒了拆除顺序。*治疗方法：* 始终先销毁 cdev，然后是别名，然后是锁，然后是 softc。cdev 是门；在拆除门后面的房间之前先关上它。
- **在 cdev 上存储每次打开的状态。** *症状：* 一个用户时工作正常，两个用户时状态混乱。*原因：* 读取位置或类似的每次描述符数据存储在 `si_drv1` 或 softc 中。*治疗方法：* 将它们移动到 `struct myfirst_fh` 中并用 `devfs_set_cdevpriv` 注册。
- **忘记 `/dev` 的更改不持久。** *症状：* 你手工运行的 `chmod` 在重启或模块重新加载后消失。*原因：* devfs 是实时的，不是磁盘上的。*治疗方法：* 将更改放入 `/etc/devfs.conf` 并 `service devfs restart`。
- **拆离时泄漏别名。** *症状：* `kldunload` 返回 `EBUSY`，驱动卡住了。*原因：* 别名 cdev 仍然活跃。*治疗方法：* 在 `detach()` 中主设备之前对别名调用 `destroy_dev(9)`。
- **两次调用 `devfs_set_cdevpriv`。** *症状：* 第二次调用返回 `EBUSY`，你的处理程序将错误返回给用户。*原因：* `open` 中两个独立路径都试图注册私有数据，或处理程序为同一打开运行了两次。*治疗方法：* 审计代码路径，使每次 `d_open` 调用恰好一次成功的 `devfs_set_cdevpriv`。
- **分配 `fh` 但在错误路径上未释放。** *症状：* 与失败打开相关的稳定内存泄漏。*原因：* `devfs_set_cdevpriv` 返回了错误，分配被遗弃了。*治疗方法：* 在 `malloc` 之后和成功 `devfs_set_cdevpriv` 之前的任何错误上，显式 `free` 分配。
- **混淆别名和符号链接。** *症状：* 通过 `devfs.conf` 在 `link` 上设置的权限与驱动在主设备上公布的不匹配。*原因：* 在同一名称上混合两种机制。*治疗方法：* 每个名称选一个工具；当驱动拥有名称时使用别名，当操作员方便是目标时使用符号链接。
- **为"只是测试"使用宽开放模式。** *症状：* 以 `0666` 发送到预发布环境的驱动突然需要收紧而不破坏消费者。*原因：* 临时实验模式变成了默认值。*治疗方法：* 默认为 `0600`，仅在具体消费者请求时放宽，并在 `mda_mode` 行旁边的注释中注明原因。
- **在新代码中使用 `make_dev`。** *症状：* 驱动编译并工作，但审查者标记了调用。*原因：* `make_dev` 是家族中最旧的形式，失败时 panic。*治疗方法：* 使用带有填充的 `struct make_dev_args` 的 `make_dev_s`。较新的形式更易读、更易错误检查、对未来的 API 添加更友好。
- **忘记 `D_VERSION`。** *症状：* 驱动加载但第一次 `open` 返回神秘失败，或内核打印 cdevsw 版本不匹配。*原因：* `cdevsw` 的 `d_version` 字段被留为零。*治疗方法：* 在每个 `cdevsw` 字面量中将 `.d_version = D_VERSION` 设为第一个字段。
- **因为"能编译"而发布带 `D_NEEDGIANT` 的代码。** *症状：* 驱动工作但每个操作都在 Giant 锁后序列化，使 SMP 密集型工作负载变慢。*原因：* 标志是从旧驱动复制的，或为消除警告而添加的，从未移除。*治疗方法：* 删除标志。如果你的驱动确实需要 Giant 才能维持，它有一个真正的锁定错误需要真正的修复，而不是一个标志。

## 总结

你现在足够好地理解了驱动和用户空间之间的层面，可以有目的地塑造它了。具体来说：

- `/dev` 不是磁盘上的目录。它是内核活动对象的 devfs 视图。
- `struct cdev` 是你的节点的内核侧标识。vnode 是 VFS 到达它的方式。`struct file` 是单个 `open(2)` 在内核中的方式。
- `mda_uid`、`mda_gid` 和 `mda_mode` 设定 `ls -l` 显示的基线。`devfs.conf` 和 `devfs.rules` 在其上层叠操作员策略。
- 节点的路径是你的格式字符串说的任何内容，包括斜杠。`/dev` 下的子目录是分组相关节点的正常且受欢迎的方式。
- `make_dev_alias(9)` 让一个 cdev 响应多个名称。记住在拆除主设备时销毁别名。
- `devfs_set_cdevpriv(9)` 给每次 `open(2)` 自己的状态，带有自动清理。这是你在下一章中最依赖的工具。

你带入第九章的驱动是你开始时的同一个 `myfirst`，但有更清晰的名称、更合理的权限集和准备好承载读取位置、字节计数和真正 I/O 需要的小型簿记的每次打开状态。保持文件打开。你很快就会再次编辑它。

### 展望第九章

在第九章中，我们将正确填充 `d_read` 和 `d_write`。你将学习内核如何用 `uiomove(9)` 在用户内存和内核内存之间移动字节，为什么 `struct uio` 看起来是那样，以及如何设计一个对短读取、短写入、未对齐缓冲区和行为不端的用户程序安全的驱动。你刚布线的每次打开状态将承载读取偏移和写入状态。别名将保持旧用户界面在驱动增长时继续工作。你在这里设置的权限模型将在你开始发送真正数据时保持你的实验脚本诚实。

具体来说，第九章将需要你添加到 `struct myfirst_fh` 的字段用于两件事。`reads` 计数器将增加一个匹配的 `read_offset` 字段，使每个描述符记住它在合成数据流中的位置。`writes` 计数器将由一个小型环形缓冲区补充，`d_write` 向其中追加而 `d_read` 从中排空。你在每个处理程序中用 `devfs_get_cdevpriv` 检索的 `fh` 指针将是所有这些状态的入口。

你在实验 8.2 中创建的别名将在不做任何更改的情况下继续工作：`/dev/myfirst` 和 `/dev/myfirst/0` 都将产生数据，描述符之间的每次状态将是独立的。

你在实验 8.1 和 8.4 中设置的权限将仍然是开发的正确默认值：足够紧凑以在原始用户触及设备时强制有意识的 `sudo`，足够开放以 `operator` 组中的测试工具可以运行数据路径测试而无需升级。

你建造了一扇形状良好的门。在下一章中，门后面的房间将活跃起来。


## 参考：make_dev_s 和 cdevsw 一览

本参考将最有用的声明和标志值收集在一处，交叉引用到解释每一个的章节部分。编写自己的驱动时保持打开它；花费一天的大多数错误都是关于这些值之一的错误。

### 规范的 make_dev_s 骨架

单节点驱动的规范模板：

```c
struct make_dev_args args;
int error;

make_dev_args_init(&args);
args.mda_devsw   = &myfirst_cdevsw;
args.mda_uid     = UID_ROOT;
args.mda_gid     = GID_OPERATOR;
args.mda_mode    = 0660;
args.mda_si_drv1 = sc;

error = make_dev_s(&args, &sc->cdev, "myfirst/%d", sc->unit);
if (error != 0) {
        device_printf(dev, "make_dev_s: %d\n", error);
        /* unwind and return */
        goto fail;
}
```

### 规范的 cdevsw 骨架

```c
static struct cdevsw myfirst_cdevsw = {
        .d_version = D_VERSION,
        .d_name    = "myfirst",
        .d_open    = myfirst_open,
        .d_close   = myfirst_close,
        .d_read    = myfirst_read,
        .d_write   = myfirst_write,
        .d_ioctl   = myfirst_ioctl,     /* 在第二十五章添加 */
        .d_poll    = myfirst_poll,      /* 在第十章添加 */
        .d_kqfilter = myfirst_kqfilter, /* 在第十章添加 */
};
```

省略的字段等同于 `NULL`，内核将其解释为"不支持"或"使用默认行为"，具体取决于哪个字段。

### make_dev_args 结构

来自 `/usr/src/sys/sys/conf.h`：

```c
struct make_dev_args {
        size_t         mda_size;         /* 由 make_dev_args_init 设置 */
        int            mda_flags;        /* MAKEDEV_* 标志 */
        struct cdevsw *mda_devsw;        /* 必需 */
        struct ucred  *mda_cr;           /* 通常为 NULL */
        uid_t          mda_uid;          /* 参见 conf.h 中的 UID_* */
        gid_t          mda_gid;          /* 参见 conf.h 中的 GID_* */
        int            mda_mode;         /* 八进制模式 */
        int            mda_unit;         /* 单元号 (0..INT_MAX) */
        void          *mda_si_drv1;      /* 通常是 softc */
        void          *mda_si_drv2;      /* 第二个驱动指针 */
};
```

### MAKEDEV 标志字

| 标志                   | 含义                                                 |
|------------------------|---------------------------------------------------------|
| `MAKEDEV_REF`          | 创建时添加额外引用。                     |
| `MAKEDEV_NOWAIT`       | 不为内存睡眠；如果紧张返回 `ENOMEM`。      |
| `MAKEDEV_WAITOK`       | 为内存睡眠（`make_dev_s` 的默认值）。            |
| `MAKEDEV_ETERNAL`      | 将 cdev 标记为永不销毁。                 |
| `MAKEDEV_CHECKNAME`    | 验证名称；对错误名称返回错误。           |
| `MAKEDEV_WHTOUT`       | 创建 whiteout 条目（堆叠文件系统）。          |
| `MAKEDEV_ETERNAL_KLD`  | 静态时为 `MAKEDEV_ETERNAL`，作为 KLD 构建时为零。  |

### cdevsw d_flags 字段

| 标志             | 含义                                                          |
|------------------|------------------------------------------------------------------|
| `D_TAPE`         | 类别提示：磁带设备。                                      |
| `D_DISK`         | 类别提示：磁盘设备（旧式；现代磁盘使用 GEOM）。      |
| `D_TTY`          | 类别提示：TTY 设备。                                       |
| `D_MEM`          | 类别提示：内存设备，如 `/dev/mem`。                 |
| `D_TRACKCLOSE`   | 对每个描述符的每次 `close(2)` 调用 `d_close`。         |
| `D_MMAP_ANON`    | 此 cdev 的匿名 mmap 语义。                          |
| `D_NEEDGIANT`    | 强制 Giant 锁调度。在新代码中避免。                    |
| `D_NEEDMINOR`    | 驱动使用 `clone_create(9)` 分配次设备号。       |

### 常见的 UID 和 GID 常量

| 常量       | 数值 | 用途                                    |
|----------------|---------|--------------------------------------------|
| `UID_ROOT`     | 0       | 超级用户。大多数节点的默认所有者。   |
| `UID_BIN`      | 3       | 守护进程可执行文件。                        |
| `UID_UUCP`     | 66      | UUCP 子系统。                            |
| `UID_NOBODY`   | 65534   | 非特权占位符。                  |
| `GID_WHEEL`    | 0       | 受信任的管理员。                    |
| `GID_KMEM`     | 2       | 读取内核内存的权限。              |
| `GID_TTY`      | 4       | 终端设备。                     |
| `GID_OPERATOR` | 5       | 操作工具。                         |
| `GID_BIN`      | 7       | 守护进程拥有的文件。                        |
| `GID_VIDEO`    | 44      | 视频帧缓冲区访问。                  |
| `GID_DIALER`   | 68      | 串口拨出程序。             |
| `GID_NOGROUP`  | 65533   | 无组。                                  |
| `GID_NOBODY`   | 65534   | 非特权占位符。                  |

### 销毁函数

| 函数                           | 何时使用                                                    |
|------------------------------------|----------------------------------------------------------------|
| `destroy_dev(cdev)`                | 普通、带排空的同步销毁。                  |
| `destroy_dev_sched(cdev)`          | 当你不能睡眠时的延迟销毁。                    |
| `destroy_dev_sched_cb(cdev,cb,arg)`| 带后续回调的延迟销毁。                |
| `destroy_dev_drain(cdevsw)`        | 在释放 cdevsw 之前等待其所有 cdev 完成。 |
| `delist_dev(cdev)`                 | 从 devfs 移除 cdev 但尚未完全销毁。         |

### 每次打开状态函数

| 函数                                   | 用途                                           |
|--------------------------------------------|---------------------------------------------------|
| `devfs_set_cdevpriv(priv, dtor)`           | 将私有数据附加到当前描述符。    |
| `devfs_get_cdevpriv(&priv)`                | 检索当前描述符的私有数据。 |
| `devfs_clear_cdevpriv()`                   | 提前拆离并运行析构函数。              |
| `devfs_foreach_cdevpriv(dev, cb, arg)`     | 遍历 cdev 上的所有每次打开记录。           |

### 别名函数

| 函数                                             | 用途                                    |
|------------------------------------------------------|--------------------------------------------|
| `make_dev_alias(pdev, fmt, ...)`                     | 为主 cdev 创建别名。        |
| `make_dev_alias_p(flags, &cdev, pdev, fmt, ...)`     | 创建带标志和错误返回的别名。|
| `make_dev_physpath_alias(...)`                       | 创建拓扑路径别名。              |

### 术语表

本章使用的一些术语的简短词汇表：

- **cdev**：设备文件的内核侧标识，每个节点一个。
- **cdevsw**：将操作映射到驱动处理程序的调度表。
- **cdevpriv**：通过 `devfs_set_cdevpriv(9)` 附加到文件描述符的每次打开状态。
- **devfs**：将 cdev 作为 `/dev` 下节点呈现的虚拟文件系统。
- **mda_***：传递给 `make_dev_s(9)` 的 `make_dev_args` 结构的成员。
- **softc**：由 Newbus 分配并可通过 `device_get_softc(9)` 访问的每设备私有数据。
- **SI_***：存储在 `struct cdev` 的 `si_flags` 字段中的标志。
- **D_***：存储在 `struct cdevsw` 的 `d_flags` 字段中的标志。
- **MAKEDEV_***：通过 `mda_flags` 传递给 `make_dev_s(9)` 及其相关函数的标志。
- **UID_*** 和 **GID_***：标准用户和组身份的符号常量。
- **destroy_dev_drain**：卸载创建了许多 cdev 的模块时使用的 cdevsw 级排空函数。
- **devfs.conf**：用于持久节点所有者和模式的主机级策略文件。
- **devfs.rules**：塑造 devfs 每挂载视图的规则集文件，主要用于 jail。

术语表将随书的进展而增长。第八章介绍了它需要的大部分术语；后续章节将添加自己的并引用回此列表。

### 去哪里阅读更多

- `make_dev(9)`、`destroy_dev(9)`、`cdev(9)` 手册页了解 API 界面。
- `devfs(5)`、`devfs.conf(5)`、`devfs.rules(5)`、`devfs(8)` 了解文件系统层文档。
- `/usr/src/sys/sys/conf.h` 了解规范的结构和标志定义。
- `/usr/src/sys/kern/kern_conf.c` 了解 `make_dev*` 家族的实现。
- `/usr/src/sys/fs/devfs/devfs_vnops.c` 了解 `devfs_set_cdevpriv` 和相关函数的实现。
- `/usr/src/sys/fs/devfs/devfs_rule.c` 了解规则子系统。

此参考故意保持简短。章节是推理所在的地方；此节只是查找表。


## 设备文件界面的实用工作流

了解 API 只是工作的一半。知道何时使用哪个 API，以及如何快速发现问题，是另一半。本节收集将使接下来的几章进展顺利的工作流：编辑驱动的内部循环、早期捕获 bug 的习惯，以及在重大更改前值得运行的检查清单。

### 内部循环

"内部循环"是编辑、构建、加载、测试、卸载、再次编辑的循环。你的第七章脚本已经有了这个的一个版本。对于第八章，内部循环变得更丰富一些，因为有更多用户可见的界面需要验证。

当你正在开发 `myfirst` 的某个阶段时，一个有用的顺序：

```sh
% cd ~/drivers/myfirst
% sudo kldunload myfirst 2>/dev/null || true
% make clean && make
% sudo kldload ./myfirst.ko
% dmesg | tail -5
% ls -l /dev/myfirst /dev/myfirst/0 2>/dev/null
% sysctl dev.myfirst.0.stats
% sudo ./probe_myfirst /dev/myfirst/0
% sudo kldunload myfirst
% dmesg | tail -3
```

每一行都有目的。第一次卸载是防御性的：之前的测试留下了已加载的模块，这清除了状态。`make clean && make` 从头重建以避免陈旧的目标文件。第一个 `dmesg | tail -5` 显示挂载消息。`ls -l` 和 `sysctl` 确认用户可见的界面存在且内部计数器已初始化。探测程序锻炼数据路径。最后的卸载和 `dmesg` 确认拆离消息。

如果任何步骤产生了意外结果，你知道是哪个步骤。这就是将循环脚本化的价值：不是为了节省输入，而是使失败信号明确。

随第七章示例发布的 `rebuild.sh` 辅助脚本为你封装了其中的大部分。实验 8 不变地重用它。

### 善于阅读 dmesg

`dmesg` 是你的驱动做了什么的叙述。善于阅读它是值得早期建立的习惯。

内核的默认环形缓冲区可以显示来自早期启动和运行时活动的数万行。当你正在开发特定的驱动时，三种技术使相关的切片可见：

**测试前清除。** `sudo dmesg -c > /dev/null` 清除缓冲区。下一次加载/卸载循环然后产生一个小而集中的日志。在实验之间使用这个。

**按标签过滤。** `dmesg | grep myfirst` 将视图缩小到你的驱动产生的行，假设你的 `device_printf` 调用发出驱动名称。它们确实如此，因为 `device_printf(9)` 在每行前面加上设备的 Newbus 名称。

**实时监视。** 在第二个终端中运行 `tail -f /var/log/messages`。每个到达 `dmesg` 的驱动消息也出现在那里，带有时间戳。这在长时间运行的测试期间特别有用，如实验 8.6 的并行探测练习。

### 监视 fstat

对于拆离问题，`fstat(1)` 是你的朋友。两个习惯用法经常出现：

```sh
% fstat /dev/myfirst/0
```

简单查找；显示所有持有节点打开的进程。输出列是用户、命令、pid、fd、挂载、inum、模式、rdev、r/w、名称。

```sh
% fstat -p $$ | grep myfirst
```

将搜索限制到当前 shell。当你不确定当前 shell 是否有来自早期测试的残留描述符打开时有用。

```sh
% fstat -u $USER | grep myfirst
```

限制到你自己的用户进程。类似的用例，更广的范围。

### 每个驱动的 sysctl 搭档

从第七章起，你的驱动已经在 `dev.myfirst.0.stats` 下暴露了一个 sysctl 树。第八章阶段 2 向该树添加了 `active_fhs`。当你运行实验时，sysctl 是最便宜的可能的观察工具：

```sh
% sysctl dev.myfirst.0.stats
dev.myfirst.0.stats.attach_ticks: 123456
dev.myfirst.0.stats.open_count: 42
dev.myfirst.0.stats.active_fhs: 0
dev.myfirst.0.stats.bytes_read: 0
```

每个计数器都是对驱动认为什么是真的的检查。你期望的与 sysctl 显示的差异始终是一个信号。如果 `active_fhs` 在没有描述符应该打开时非零，你有一个泄漏。如果 `open_count` 小于你打开设备的次数，你的挂载路径运行了两次或你的计数器有竞争。

Sysctl 比任何其他观察机制都便宜。只要数字或短字符串信息足够，优先使用它们而不是读取设备本身。

### 每次代码更改的快速检查清单

在提交对驱动的更改之前，走一遍以下内容。在这里花十分钟可以节省以后的数小时调试。

1. 驱动仍能从干净的树构建吗？
2. 它仍能在没有打开描述符的系统上干净地加载和卸载吗？
3. 用户可见的界面（`ls -l /dev/myfirst/...`）与你的代码意图匹配吗？
4. 权限仍然是它们应该是的狭窄默认值吗？
5. 如果你更改了 `attach`，每个错误路径是否仍完全展开？
6. 如果你更改了 `detach`，当描述符被持有时驱动是否仍能干净地卸载（要么干净地返回 `EBUSY`，要么如果你选择了不同的策略，不会泄漏）？
7. 如果你更改了每次打开的状态，`stress_probe`、`parallel_probe` 和 `hold_myfirst` 是否仍按预期运行？
8. 你是否引入了任何应该用 `if (bootverbose)` 限制的 `device_printf` 调用，以免它们淹没日志？
9. 你是否留下了任何 `#if 0` 或调试打印？现在就把它们移除。
10. 如果你更改了所有权或模式，`devfs.conf` 是否仍产生预期的覆盖？

这个检查清单故意无聊。这就是重点。一个无聊、可靠的过程每次都胜过英雄式的调试会话。

### 发布前的快速检查清单

当你为"完成"准备驱动的某个阶段时，检查清单会变得更长一些。以上所有，加上：

1. 从真正干净的树 `make clean && make` 构建无警告。
2. `kldload ./myfirst.ko; sleep 0.1; kldunload myfirst` 完成十次无问题。
3. `stress_probe 10000` 完成无问题且 `active_fhs` 返回零。
4. `parallel_probe 8` 打开八个描述符、保持并干净地关闭。内核日志显示八个不同的 `fh=` 指针和八个析构函数。
5. 当描述符打开时 `kldunload` 干净地返回 `EBUSY`，而不是 panic。
6. 带有放宽条目的 `devfs.conf` 在 `service devfs restart` 时应用。
7. 对 `/dev/myfirst*` 的 `ls -l` 审计显示没有意外的模式或所有权。
8. `dmesg` 包含你期望的恰好挂载和拆离消息，没有警告或错误。
9. 源代码没有注释掉的实验、TODO 行和调试辅助。
10. 你的驱动暴露的 sysctl 是描述性的并在代码注释中记录。

任何提交都不应跳过第 1 到第 3 项。它们是你能买到的最便宜的高价值保险。

### 添加新节点的工作流

端到端演练一次将锚定前面的章节。假设你决定 `myfirst` 应该在 `/dev/myfirst/status` 暴露一个额外的只读状态节点，与编号的数据节点不同。以下是你的做法。

**步骤 1：设计。** 决定节点的形状。它属于与数据节点相同的 `cdevsw`，还是不同的？一个只回答 `read(2)` 返回简短文本摘要的状态节点通常想要自己的只设置了 `d_read` 的 `cdevsw`，因为策略与数据节点不同。决定权限模式。任何人只读建议 `0444`；操作员只读建议 `0440` 加上适当的组。

**步骤 2：声明。** 将新的 `cdevsw`、其 `d_read` 处理程序和 `struct cdev *cdev_status` 字段添加到 softc。

**步骤 3：实现。** 编写 `d_read` 处理程序。它基于 softc 状态格式化短字符串并通过 `uiomove(9)` 返回。对于第八章，你可能先存根它，在第九章后填充。

**步骤 4：接线。** 在 `attach` 中，为状态节点添加 `make_dev_s` 调用。在 `detach` 中，在数据节点销毁之前添加 `destroy_dev` 调用。

**步骤 5：测试。** 重建、重新加载、检查、锻炼、卸载。检查 `ls -l` 显示具有预期模式的状态节点。检查 `cat /dev/myfirst/status` 工作并产生合理的输出。检查整个驱动仍能干净地卸载。

**步骤 6：文档。** 在驱动源代码中添加描述节点的注释。如果状态是数字的并且也适合那里，在 `dev.myfirst.0.stats` sysctl 中添加条目。在你保留的任何更改日志中记录更改。

六个步骤，每一步都很小，每一步都很具体。那就是 bug 保持可见的粒度级别。

### 诊断缺失节点的工作流

第八章工具部分的故障排除演练给了你一个简短的检查清单。这是一个适合索引卡的更完整的工作流。

**阶段 1：是模块吗？**

- `kldstat | grep myfirst` 显示模块。
- `dmesg | grep myfirst` 显示挂载消息。

如果模块未加载或挂载未运行，先修复那个。

**阶段 2：是 Newbus 吗？**

- `devinfo -v | grep myfirst` 显示 Newbus 设备。

如果 Newbus 什么也没显示，你的 `device_identify` 或 `device_probe` 没有创建子设备。看那里。

**阶段 3：是 devfs 吗？**

- `ls -l /dev/myfirst` 列出目录（或报告它缺失）。
- `dmesg | grep 'make_dev'` 显示 `make_dev_s` 的任何失败。

如果 Newbus 正常但 devfs 什么也没显示，`make_dev_s` 返回了错误。检查你的路径格式字符串、你的 `mda_devsw`、你的参数结构。

**阶段 4：是策略吗？**

- `devfs rule showsets` 列出活动的规则集。
- `devfs rule -s N show` 列出规则集 N 中的规则。

如果 devfs 有 cdev 但你的 jail 或本地会话看不到它，规则集在隐藏它。

每个失败都映射到这四个阶段之一。按顺序检查它们，你几乎总能在不到一分钟内确定原因。

### 审查他人驱动的工作流

当你审查触及驱动的设备文件界面的拉取请求时，有用的问题是：

- 每个 `make_dev_s` 都有匹配的 `destroy_dev` 吗？
- `make_dev_s` 之后的每个错误路径在返回前都调用 `destroy_dev` 吗？
- `detach` 是否销毁它创建的每个 cdev？
- `si_drv1` 是通过 `mda_si_drv1` 填充而不是事后赋值吗？
- 权限模式对于节点的目的可辩护吗？
- cdevsw 的 `d_version` 设置为 `D_VERSION` 了吗？
- 节点应该支持的所有 `d_*` 处理程序都存在吗，每个关于其 errno 返回是否一致？
- 如果驱动使用 `devfs_set_cdevpriv`，每次打开是否恰好有一次成功的设置和恰好一个析构函数？
- 如果驱动使用别名，它们是否在 `detach` 中主设备之前销毁？
- 如果驱动有多个 cdev，它是否在其卸载路径中调用 `destroy_dev_drain`？

这是审查检查清单，不是教程。审查更快，因为每个问题都有是或否的答案，每个是都可以机械地检查。

### 保持实验室日志

实验室日志是一个小笔记本或文本文件，你记录你做了什么、看到了什么、学到了什么。本书从第二章开始就推荐这个。在第八章，它以特定方式回报：你将多次运行相同类型的实验，简短的笔记让你避免两次重复相同的错误。

日志条目的有用模板：

```text
Date: 2026-04-17
Driver: myfirst stage 2
Goal: verify per-open state is isolated across two processes
Steps:
 - loaded stage 2 kmod
 - ran parallel_probe with count=4
 - observed 4 distinct fh= pointers in dmesg
 - observed active_fhs=4 in sysctl
 - closed, observed 4 destructor lines, active_fhs=0
Result: as expected
Notes: first run missed destructor lines because dmesg ring buffer
       was full; dmesg -c before the test solved it
```

每个实验两分钟，不再多。价值在数月后出现，当你正在追踪一个新问题，日志搜索揭示相同症状曾在不同情况下出现过一次。

### 常见设计问题及如何思考它们

驱动作者到达设备文件阶段时一些问题会反复出现。每个都在真正的审查讨论中出现过不止一次。答案很短；推理值得内化。

**问：我应该在 `device_identify` 还是 `device_attach` 中创建 cdev？**

在 `device_attach`。`identify` 回调运行非常早，在驱动实例有 softc 之前。cdev 想要通过 `mda_si_drv1` 引用 softc，这意味着 softc 必须已经存在。第七章设置了这个模式；保持它。

**问：我应该在 `attach` 和 `detach` 之外创建额外的 cdev 吗？**

如果它们真正是每次驱动实例的，把它们放在 `attach` 中并在 `detach` 中销毁。如果它们是动态的，响应用户操作创建，在接收用户请求的任何处理程序中创建它们，并在稍后的处理程序展开请求时或驱动拆离时销毁它们。仔细跟踪它们；丢失的 cdev 是泄漏的常见来源。

**问：我应该设置 `D_TRACKCLOSE` 吗？**

通常不。通过 `devfs_set_cdevpriv` 的每次打开状态机制几乎涵盖了 `D_TRACKCLOSE` 会有诱惑力的所有情况，而且它自动清理自己。只有当你需要你的 `d_close` 在每个描述符关闭时运行，而不仅仅是最后一个时才设置 `D_TRACKCLOSE`。真正的用例很少；TTY 驱动和少数其他符合。

**问：我应该允许多个打开者吗？**

默认是，通过每次打开状态。独占访问有时对于一次只能支持一个会话的硬件是必要的，但它是一个选择，不是默认。第七章强制排他性作为教学举措；第八章阶段 2 正好解除了那个限制，因为它不是常见情况。

**问：失败的打开我应该返回 `ENXIO` 还是 `EBUSY`？**

驱动没准备好时 `ENXIO`。设备原则上可以打开但现在不行时 `EBUSY`。用户可见的消息是不同的，阅读你内核日志的操作员会感谢你选择了正确的那个。

**问：我应该 `strdup` 从用户态获取的字符串吗？**

不在打开路径上。如果处理程序有正当理由在调用之后记住用户提供的字符串，使用带显式大小的 `malloc(9)` 并复制字符串进去。永远不要在处理程序返回后依赖指向用户态内存的指针；它可能不再有效，即使有效，内核也不应长期信任用户态拥有的内存。

**问：softc 应该记住哪些描述符打开了它吗？**

通常不。通过 `devfs_set_cdevpriv` 的每次打开状态是答案。如果你需要迭代机制，`devfs_foreach_cdevpriv` 存在且正确。不要在 softc 中维护你自己的描述符指针列表；锁定非平凡，内核已经提供了正确答案。

**问：我的拆离应该何时拒绝 `EBUSY`？**

当驱动无法用当前状态安全地拆除自己时。打开的描述符是最常见的原因。一些驱动也在硬件正在主动传输或控制操作正在进行时拒绝。早期干净地报错；不要试图从 `detach` 内部强制系统进入干净状态。

**问：我可以在描述符打开时卸载驱动吗？**

如果 `detach` 拒绝就不行。如果你的 `detach` 接受这种情况，内核仍会排空进行中的处理程序，但打开的描述符保留在现有文件表上直到进程关闭它们，那些描述符然后将从进一步操作返回 `ENXIO`（或类似）。对于教学驱动，拒绝 `EBUSY` 是更干净的选择。

这些问题是你第一次真正的驱动审查中会出现的问题。在这里看过一次意味着当审查者提问时你不是第一次看到它们。

### 常见设计选择的决策树

当你坐下来设计新节点或更改现有节点时，问题往往落入一小组分支。下面的树是野外指南，不是算法；真正的设计总是涉及判断，但知道树的形状有帮助。

**开始：我想通过 `/dev` 暴露一些东西。**

**分支 1：节点承载什么类型的状态？**

- **无会话，简单数据源或汇**（如 `/dev/null`、`/dev/zero`）：无 softc，无每次打开状态，每个行为一个 `cdevsw`。在 `MOD_LOAD` 处理程序中使用 `make_dev` 或 `make_dev_s`。模式通常 `0666`。
- **每次设备硬件**（如串口、传感器、LED）：每个实例一个 softc，每个实例一个 cdev。使用 `attach`/`detach` 模式。模式通常 `0600` 或 `0660`。
- **子系统控制**（如 `/dev/pf` 或 `/dev/mdctl`）：一个 cdev 暴露仅 `d_ioctl` 的操作。模式 `0600`。
- **每次会话状态**（如 BPF、如 FUSE）：每个会话一个 cdev 或一个克隆入口点。通过 `devfs_set_cdevpriv` 的每次打开状态。模式 `0600`。

**分支 2：用户如何发现节点？**

- **稳定的固定名称**（如 `/dev/null`）：将名称放入 `make_dev` 格式字符串并保留。
- **每次实例编号名称**（如 `/dev/myfirst0`）：在格式字符串中使用 `%d`，用 `device_get_unit(9)` 获取数字。
- **子目录分组**（如 `/dev/led/foo`）：在格式字符串内使用 `/`；devfs 按需创建目录。
- **按需每次打开实例**：使用克隆。稍后覆盖。

**分支 3：谁可以触及它？**

- **任何人**：`UID_ROOT`、`GID_WHEEL`、模式 `0666`。罕见；仅用于无害节点。
- **仅 root**：`UID_ROOT`、`GID_WHEEL`、模式 `0600`。任何有特权东西的默认值。
- **Root 加操作员组**：`UID_ROOT`、`GID_OPERATOR`、模式 `0660`。常见于动手特权工具。
- **Root 写，任何人读**：`UID_ROOT`、`GID_WHEEL`、模式 `0644`。用于状态节点。
- **自定义命名组**：在 `/etc/group` 中定义组，使用 `devfs.conf` 在节点创建时调整所有权。不要在驱动内部发明组。

**分支 4：多少并发打开者？**

- **一次恰好一个**：独占打开模式，softc 中的标志，在 `d_open` 中互斥锁下检查，冲突时返回 `EBUSY`。无 `devfs_set_cdevpriv`。
- **多个，每个有独立状态**：移除独占检查，在 `d_open` 中分配每次打开结构，调用 `devfs_set_cdevpriv`，用 `devfs_get_cdevpriv` 读回。
- **多个，全部共享驱动范围状态**：每次打开什么都不分配；只需在其互斥锁下读取和写入 softc。

**分支 5：当驱动在用户活动时卸载会发生什么？**

- **拒绝卸载**，只要任何描述符打开就从 `detach` 返回 `EBUSY`。这是干净的默认值。
- **接受卸载**但使打开的描述符无效。在这种情况下你需要一个 `d_purge` 处理程序来唤醒任何阻塞的线程并说服它们迅速返回。更复杂；只有在拒绝会使系统处于更糟糕状态时才这样做。

**分支 6：用户和操作员需要什么样的名称调整？**

- **驱动本身维护的第二个名称**（旧路径，众所周知的快捷方式）：`attach` 中的 `make_dev_alias(9)`，`detach` 中对其 `destroy_dev(9)`。
- **操作员维护的第二个名称**：`/etc/devfs.conf` 中的 `link`。驱动什么都不做。
- **每个主机的权限放宽或收窄**：`/etc/devfs.conf` 中的 `own` 和 `perm`。驱动保持其基线。
- **Jail 过滤视图**：`/etc/devfs.rules` 中的规则集，在 `jail.conf` 中引用。驱动没有发言权。

**分支 7：用户态程序如何从驱动接收事件？**

- **通过读取轮询**：只需要交出字节的驱动。`d_read` 和 `d_write`。
- **带信号的阻塞读取**：应该在 SIGINT 上解除阻塞的驱动。第十章覆盖。
- **Poll/select**：`d_poll`。第十章覆盖。
- **Kqueue**：`d_kqfilter`。第十章覆盖。
- **devd 通知**：来自驱动的 `devctl_notify(9)`；`/etc/devd.conf` 中的操作员侧规则。
- **sysctl 拉取**：用于无文件描述符成本的可观测性。始终是 `/dev` 界面的补充。

这棵树不涵盖每种情况。它涵盖足够使驱动作者可以在不惊慌的情况下导航前几个设计决策。当出现树中没有的新问题时，写下问题和你的解决方案；那就是树如何为你个人生长的。

### 关于过度工程的警告

一些设计诱惑值得特别指出，因为它们倾向于将简单驱动变成复杂驱动而没有收益。

- **在 `read`/`write` 上发明你自己的 IPC 协议**。如果消息是结构化的，使用 `ioctl(2)`（第二十五章）。
- **在 `ioctl` 命令中嵌入微小语言**以便用户可以"脚本化"驱动。这几乎总是功能属于用户态的信号。
- **通过一个 `cdevsw` 多路复用许多不相关的子系统**。如果两个界面有不同的语义，给它们两个 `cdevsw` 值；不花费任何东西且读起来更好。
- **添加 `D_NEEDGIANT` 以消除 SMP 警告**。警告是正确的；修复锁定。
- **处理来自每个可能用户态程序的每个可能 `errno` 值**。为你的情况选择正确的一个并坚持它。标准 `err(3)` 家族做其余的。

"尽可能简单，但不要更简单"的纪律在这个级别特别重要。驱动代码的每一行都是负载下可能有 bug 的一行。精简的驱动更容易审查、更容易调试、更容易移植、更容易移交给下一个维护者。



## 动手实验

这些实验就地扩展第七章驱动。你不应该需要从头重新输入任何东西。配套目录镜像了各个阶段。

### 实验 8.1：结构化名称和更严格的权限

**目标。** 将设备从 `/dev/myfirst0` 移动到 `/dev/myfirst/0`，并将组更改为 `operator`，模式 `0660`。

**步骤。**

1. 在 `myfirst_attach()` 中，将 `make_dev_s()` 格式字符串更改为 `"myfirst/%d"`。
2. 将 `args.mda_gid` 从 `GID_WHEEL` 更改为 `GID_OPERATOR`，将 `args.mda_mode` 从 `0600` 更改为 `0660`。
3. 重建并重新加载：

   ```sh
   % make clean && make
   % sudo kldload ./myfirst.ko
   % ls -l /dev/myfirst
   total 0
   crw-rw----  1 root  operator  0x5a Apr 17 09:41 0
   ```

4. 确认 `operator` 组中的普通用户现在可以在没有 `sudo` 的情况下从节点读取。在 FreeBSD 上，你用 `pw groupmod operator -m yourname` 将用户添加到该组，然后启动一个新的 shell。
5. 卸载驱动并确认 `/dev/myfirst/` 目录随节点一起消失。

**成功标准。**

- `/dev/myfirst/0` 在加载时出现，在卸载时消失。
- `ls -l /dev/myfirst/0` 显示 `crw-rw----  root  operator`。
- `operator` 组成员可以 `cat </dev/myfirst/0` 而无错误。

### 实验 8.2：添加别名

**目标.** 将 `/dev/myfirst` 暴露为 `/dev/myfirst/0` 的别名。

**步骤.**

1. 向 softc 添加 `struct cdev *cdev_alias` 字段。
2. 在 `myfirst_attach()` 中成功调用 `make_dev_s()` 后，调用：

   ```c
   sc->cdev_alias = make_dev_alias(sc->cdev, "myfirst");
   if (sc->cdev_alias == NULL)
           device_printf(dev, "failed to create alias\n");
   ```

3. 在 `myfirst_detach()` 中，在销毁主 cdev 之前销毁别名：

   ```c
   if (sc->cdev_alias != NULL) {
           destroy_dev(sc->cdev_alias);
           sc->cdev_alias = NULL;
   }
   if (sc->cdev != NULL) {
           destroy_dev(sc->cdev);
           sc->cdev = NULL;
   }
   ```

4. 重建、重新加载并验证：

   ```sh
   % ls -l /dev/myfirst /dev/myfirst/0
   ```

   两个路径都应该响应。`sudo cat </dev/myfirst` 和 `sudo cat </dev/myfirst/0` 应该行为相同。

**成功标准.**

- 驱动加载时两个路径都存在。
- 卸载时两个路径都消失。
- 如果别名创建失败，驱动不会 panic 或泄漏；暂时注释掉 `make_dev_alias` 行以确认这一点。

### 实验 8.3：每次打开状态

**目标.** 给每次 `open(2)` 自己的小型结构，并从用户态验证两个描述符看到独立的数据。

**步骤.**

1. 添加如本章前面所示的 `struct myfirst_fh` 类型和 `myfirst_fh_dtor()` 析构函数。
2. 重写 `myfirst_open()` 以分配 `myfirst_fh`，调用 `devfs_set_cdevpriv()`，并在注册失败时释放。移除独占打开检查。
3. 重写 `myfirst_read()` 和 `myfirst_write()`，使每个都以调用 `devfs_get_cdevpriv(&fh)` 开始。暂时保持主体不变；第九章会填充它。
4. 重建、重新加载，然后并排运行两个 `probe_myfirst` 进程：

   ```sh
   % (sudo ./probe_myfirst &) ; sudo ./probe_myfirst
   ```

5. 在 `dmesg` 中，确认两个 `open (per-open fh=...)` 消息显示不同的指针。

**成功标准.**

- 两个同时打开成功。没有 `EBUSY`。
- 内核日志中出现两个不同的 `fh=` 指针。
- `kldunload myfirst` 只在两个探测都退出后才可能。

### 实验 8.4：devfs.conf 持久化

**目标.** 使实验 8.1 中的所有权更改在重启后保留，而不再次编辑驱动。

**步骤.**

1. 在实验 8.1 中，将 `args.mda_gid` 和 `args.mda_mode` 恢复为第七章默认值（`GID_WHEEL`、`0600`）。
2. 创建或编辑 `/etc/devfs.conf` 并添加：

   ```
   own     myfirst/0       root:operator
   perm    myfirst/0       0660
   ```

3. 在不重启的情况下应用更改：

   ```sh
   % sudo service devfs restart
   ```

4. 重新加载驱动并确认 `ls -l /dev/myfirst/0` 再次显示 `root  operator  0660`，即使驱动本身请求的是 `root  wheel  0600`。

**成功标准.**

- 加载驱动且 `devfs.conf` 就位时，节点显示 `devfs.conf` 值。
- 加载驱动且注释掉 `devfs.conf` 行并重启 devfs 时，节点返回驱动的基线。

**注意.** 实验 8.4 是操作员侧实验。驱动在步骤之间不改变。重点是看到双层策略模型在工作：驱动设定基线，`devfs.conf` 塑造视图。

### 实验 8.5：双节点驱动（数据和控制）

**目标.** 扩展 `myfirst` 以暴露两个不同的节点：位于 `/dev/myfirst/0` 的数据节点和位于 `/dev/myfirst/0.ctl` 的控制节点，每个有自己的 `cdevsw` 和自己的权限模式。

**前提条件.** 完成实验 8.3（带有每次打开状态的阶段 2）。

**步骤.**

1. 在驱动中定义第二个 `struct cdevsw`，命名为 `myfirst_ctl_cdevsw`，设置 `d_name = "myfirst_ctl"` 并只留下 `d_ioctl` 桩函数（你不会实现 ioctl 命令；只需让函数存在并返回 `ENOTTY`）。
2. 向 softc 添加 `struct cdev *cdev_ctl` 字段。
3. 在 `myfirst_attach` 中，创建数据节点后，用第二个 `make_dev_s` 调用创建控制节点。使用 `"myfirst/%d.ctl"` 作为格式。将模式设置为 `0640`，组设置为 `GID_WHEEL`，使控制节点比数据节点更窄。
4. 也通过 `mda_si_drv1` 为控制 cdev 传递 `sc`，这样 `d_ioctl` 可以找到它。
5. 在 `myfirst_detach` 中，**在**数据 cdev **之前**销毁控制 cdev。记录每次销毁。
6. 重建、重新加载并验证：

   ```sh
   % ls -l /dev/myfirst
   total 0
   crw-rw----  1 root  operator  0x5a Apr 17 10:02 0
   crw-r-----  1 root  wheel     0x5b Apr 17 10:02 0.ctl
   ```

**成功标准.**

- 两个节点在加载时出现。
- 两个节点在卸载时消失。
- 数据节点可被 `operator` 组读取；控制节点不能。
- 非 root 非 wheel 用户尝试 `cat </dev/myfirst/0.ctl` 因 `Permission denied` 失败。

**注意.** 在真正的驱动中，控制节点是 `ioctl` 配置命令所在的地方。本章不实现任何 `ioctl` 命令；那是第二十五章的工作。实验 8.5 的重点是展示你可以在一个驱动中有两个具有不同策略的节点。

### 实验 8.6：并行探测验证

**目标.** 使用配套树中的 `parallel_probe` 工具证明每次打开状态确实是每次描述符的。

**前提条件.** 完成实验 8.3。阶段 2 驱动已加载。

**步骤.**

1. 构建用户态工具：

   ```sh
   % cd examples/part-02/ch08-working-with-device-files/userland
   % make
   ```

2. 用四个描述符运行 `parallel_probe`：

   ```sh
   % sudo ./parallel_probe /dev/myfirst/0 4
   opened /dev/myfirst/0 as fd 3
   opened /dev/myfirst/0 as fd 4
   opened /dev/myfirst/0 as fd 5
   opened /dev/myfirst/0 as fd 6
   holding 4 descriptors; press enter to close
   ```

3. 打开第二个终端并检查 `dmesg`：

   ```sh
   % dmesg | tail -20
   ```

   你应该看到四个 `open via myfirst/0 fh=<pointer> (active=N)` 行，每个带有不同的指针值。

4. 在第二个终端中，检查活动打开 sysctl：

   ```sh
   % sysctl dev.myfirst.0.stats.active_fhs
   dev.myfirst.0.stats.active_fhs: 4
   ```

5. 回到第一个终端并按 Enter。探测关闭所有四个描述符。再次检查 `dmesg`：

   ```sh
   % dmesg | tail -10
   ```

   你应该看到四个 `per-open dtor fh=<pointer>` 行，每个描述符一个，具有与打开日志中出现的相同指针值。

6. 验证 `active_fhs` 回到零：

   ```sh
   % sysctl dev.myfirst.0.stats.active_fhs
   dev.myfirst.0.stats.active_fhs: 0
   ```

**成功标准.**

- 打开日志中有四个不同的 `fh=` 指针。
- 析构函数日志中有四个匹配的指针。
- `active_fhs` 递增到四并递减回零。
- 没有关于内存泄漏或意外状态的内核日志消息。

**注意.** 实验 8.6 是你能轻易产生的每次打开状态被隔离的最有力证据。如果任何步骤失败，最常见的罪魁祸首是错过了对 `devfs_set_cdevpriv` 的调用或没有递减 `active_fhs` 的析构函数。


### 实验 8.7：jail 的 devfs.rules

**目标.** 通过 devfs 规则集使 `/dev/myfirst/0` 在 jail 内可见。

**前提条件.** 你的实验系统上有一个可用的 FreeBSD jail。如果还没有，跳过此实验，在第七部分之后返回。

**步骤.**

1. 向 `/etc/devfs.rules` 添加规则集：

   ```
   [myfirst_jail=100]
   add include $devfsrules_jail
   add path 'myfirst'   unhide
   add path 'myfirst/*' unhide
   add path 'myfirst/*' mode 0660 group operator
   ```

2. 向 jail 的 `jail.conf` 添加 devfs 条目：

   ```
   myfirstjail {
           path = "/jails/myfirstjail";
           host.hostname = "myfirstjail.example.com";
           mount.devfs;
           devfs_ruleset = 100;
           exec.start = "/bin/sh";
           persist;
   }
   ```

3. 重新加载 devfs 并启动 jail：

   ```sh
   % sudo service devfs restart
   % sudo service jail start myfirstjail
   ```

4. 在 jail 内，确认节点：

   ```sh
   % sudo jexec myfirstjail ls -l /dev/myfirst
   ```

5. 通过注释掉 `add path 'myfirst/*' unhide` 行、重启 devfs 和 jail 来验证规则集工作，并观察节点消失。

**成功标准.**

- `/dev/myfirst/0` 以模式 `0660` 和组 `operator` 出现在 jail 内。
- 移除 unhide 规则会从 jail 内移除节点。
- 主机无论 jail 的规则集如何都继续看到节点。

**注意.** jail 配置通常在后面的章节中讨论；此实验是预览，以演示驱动侧结果。如果在你的系统上实验困难，在你为其他目的配置了 jail 之后再回来。

### 实验 8.8：Destroy-Dev Drain

**目标.** 演示当一个 `cdevsw` 随许多 cdev 一起被释放时 `destroy_dev` 和 `destroy_dev_drain` 之间的区别。

**前提条件.** 完成实验 8.3。你的驱动已加载且安静。

**步骤.**

1. 审查阶段 2 拆离代码。单 cdev 驱动不需要 `destroy_dev_drain`。实验模拟一个不这样做的多 cdev 驱动会出什么问题。
2. 构建驱动的 `stage4-destroy-drain` 变体（在配套树中）。此变体在挂载中创建五个 cdev，在拆离中使用 `destroy_dev_sched` 调度它们的销毁，而不排空。
3. 加载变体，然后在用户态进程持有其中一个 cdev 打开时立即卸载它：

   ```sh
   % sudo kldload ./stage4.ko
   % sudo ./hold_myfirst 60 /dev/myfirstN/3 &
   % sudo kldunload stage4
   ```

4. 观察内核日志。你应该看到抱怨，或根据时间，panic。变体故意不安全。
5. 切换到阶段 4 源代码的"修复"版本，它在每个 cdev 的 destroy-sched 调用之后调用 `destroy_dev_drain(&mycdevsw)`。重复加载/保持/卸载序列。
6. 确认修复版本干净地卸载，在模块消失之前等待持有的描述符关闭。

**成功标准.**

- 损坏的变体在持有描述符卸载时产生可观察的问题（消息、挂起或 panic）。
- 修复版本干净地完成卸载。
- 阅读源代码使哪个调用产生了差异变得清楚。

**注意.** 此实验故意触发坏状态。在一次性 VM 中运行它，而不是你关心的系统。重点是建立关于为什么 `destroy_dev_drain` 存在的直觉；一旦你看过损坏路径失败，你会记得在多 cdev 驱动中调用它。



## 挑战练习

这些在实验基础上构建。慢慢来；它们中没有引入新机制，只是延伸了你刚练习的机制。

### 挑战 1：使用别名

更改 `probe_myfirst.c` 以默认打开 `/dev/myfirst` 而不是 `/dev/myfirst/0`。从内核日志确认你的 `d_open` 运行了，并且 `devfs_set_cdevpriv` 每次 `open(2)` 恰好成功一次。然后将路径改回来。你不应该需要编辑驱动。

### 挑战 2：观察每次打开的清理

在 `myfirst_fh_dtor()` 内部添加一个 `device_printf` 来记录正在释放的 `fh` 指针。运行 `probe_myfirst` 一次并确认每次运行在 `dmesg` 中恰好出现一行析构函数。然后编写一个小程序，打开设备，睡眠 30 秒，然后在不调用 `close(2)` 的情况下退出。确认进程退出时析构函数仍然触发。清理不是客气；它是保证。

### 挑战 3：实验 devfs.rules

如果你已经配置了 FreeBSD jail，向 `/etc/devfs.rules` 添加一个 `myfirst_lab` 规则集，使 `/dev/myfirst/*` 在 jail 内可见。启动 jail，从内部打开设备，并确认驱动看到了新的打开。如果你还没有 jail，现在跳过此挑战，在第七部分之后返回。

### 挑战 4：阅读两个更多驱动

从 `/usr/src/sys/dev/` 中选择两个你还没有阅读的驱动。好的候选有 `/usr/src/sys/dev/random/randomdev.c`、`/usr/src/sys/dev/hwpmc/hwpmc_mod.c`、`/usr/src/sys/dev/kbd/kbd.c`，或其他任何足够短可以浏览的。对每个驱动，找出：

- `cdevsw` 定义及其 `d_name`。
- `make_dev*` 调用及其设置的权限模式。
- `destroy_dev` 调用，或它们的缺失。
- 驱动是否使用 `devfs_set_cdevpriv`。
- 驱动是否在 `/dev` 下创建子目录。

为每个驱动写一小段分类其设备文件界面。重点是磨练你的眼光；没有单一正确的分类法。

### 挑战 5：devd 配置

编写一个最小的 `/etc/devd.conf` 规则，在 `/dev/myfirst/0` 每次出现或消失时记录一条消息。devd 配置格式在 `devd.conf(5)` 中有文档。起始模板：

```text
notify 100 {
        match "system"      "DEVFS";
        match "subsystem"   "CDEV";
        match "cdev"        "myfirst/0";
        action              "/usr/bin/logger -t myfirst event=$type";
};
```

安装规则，重启 devd（`service devd restart`），加载和卸载驱动，然后验证 `grep myfirst /var/log/messages` 显示两个事件。

### 挑战 6：添加状态节点

修改 `myfirst` 以在数据节点旁边暴露一个只读状态节点。状态节点位于 `/dev/myfirst/0.status`，模式 `0444`，所有者 `root:wheel`。它的 `d_read` 返回一个简短的纯文本字符串，总结驱动当前状态：

```ini
attached_at=12345
active_fhs=2
open_count=17
```

提示：在 softc 中分配一个小型固定大小缓冲区，在互斥锁下格式化字符串，如果你已经读过第九章就用 `uiomove(9)` 返回给用户，或者现在用手动实现。

如果你对 `uiomove` 还不熟悉，将此挑战推迟到第九章之后。这是第九章所教内容的自然首次使用。



## 设备文件操作的错误码

你的 `d_open` 和 `d_close` 返回的每个非零值都告诉 devfs 一些具体内容。你选择的 errno 值是你的驱动与曾经触及你的节点的每个用户程序之间的契约。正确选择不花任何代价；选择错误会产生你初次阅读时无法理解的错误报告。

本节调查在设备文件界面上实际出现的 errno 值。第九章将单独处理 `d_read` 和 `d_write` 的 errno 选择，因为数据路径的选择在性质上不同。这里我们专注于打开、关闭和 ioctl 相邻返回。

### 简短列表

按你使用频率的大致顺序：

- **`ENXIO`（没有此类设备或地址）**："设备不处于可以被打开的状态。"当驱动已挂载但未准备好、已知硬件缺失、softc 处于瞬态时使用。用户看到 `Device not configured`。
- **`EBUSY`（设备忙）**："设备已经打开，此驱动不允许并发访问。"用于独占打开策略。用户看到 `Device busy`。
- **`EACCES`（权限被拒绝）**："呈现此打开的凭证不被允许。"内核通常在你的处理程序运行之前捕获权限失败，但驱动可以检查二级策略（例如，一个拒绝为读取而打开的仅 `ioctl` 节点）并自己返回 `EACCES`。
- **`EPERM`（操作不允许）**："操作需要调用者没有的特权。"精神上类似于 `EACCES`，但针对特权区别（`priv_check(9)` 失败）而不是 UNIX 文件权限。
- **`EINVAL`（无效参数）**："调用结构有效但驱动不接受这些参数。"当 `oflags` 指定驱动拒绝的组合时使用。
- **`EAGAIN`（资源暂时不可用）**："设备原则上可以打开，但现在不行。"当你有临时短缺（槽位已满、资源正在重新配置）且用户应稍后重试时使用。用户看到 `Resource temporarily unavailable`。
- **`EINTR`（被中断的系统调用）**：当处理程序内部的睡眠被信号中断时返回。你通常不会从 `d_open` 返回这个，因为打开通常不会可中断地睡眠。它在数据路径处理程序中更常见。
- **`ENOENT`（没有此类文件或目录）**：几乎总是由 devfs 本身在路径无法解析时合成。驱动很少从自己的处理程序返回这个。
- **`ENODEV`（设备不支持此操作）**："操作本身有效但此设备不支持它。"当驱动的二级接口拒绝另一个接口支持的操作时使用。
- **`EOPNOTSUPP`（操作不支持）**：`ENODEV` 的表亲。在某些子系统中用于类似情况。

### 哪个值用于哪种情况？

真正的驱动落入模式。以下是你最常编写的模式。

**模式 A：驱动已挂载但 softc 尚未准备好。** 你可能在两阶段挂载中遇到这个，cdev 在某些初始化完成之前创建，或者在拆离期间 cdev 仍然存在时。

```c
if (sc == NULL || !sc->is_attached)
        return (ENXIO);
```

**模式 B：独占打开策略。**

```c
mtx_lock(&sc->mtx);
if (sc->is_open) {
        mtx_unlock(&sc->mtx);
        return (EBUSY);
}
sc->is_open = 1;
mtx_unlock(&sc->mtx);
```

这是第七章所做的。第八章的阶段 2 移除了独占检查，因为每次打开状态可用；`EBUSY` 不再需要。

**模式 C：只读节点拒绝写入。**

```c
if ((oflags & FWRITE) != 0)
        return (EACCES);
```

当节点概念上是只读的，为写入而打开是调用者错误时使用。

**模式 D：仅特权接口。**

```c
if (priv_check(td, PRIV_DRIVER) != 0)
        return (EPERM);
```

当非特权调用者尝试打开在文件系统模式之外强制执行额外特权检查的节点时返回 `EPERM`。

**模式 E：暂时不可用。**

```c
if (sc->resource_in_flight) {
        return (EAGAIN);
}
```

当驱动稍后可以接受打开但现在不行，用户应重试时使用。

**模式 F：驱动特定的无效组合。**

```c
if ((oflags & O_NONBLOCK) != 0 && !sc->supports_nonblock) {
        return (EINVAL);
}
```

当调用者的 `oflags` 指定你的驱动不实现的模式时使用。

### 从 d_close 返回错误

`d_close` 有自己的考虑。内核通常不关心 close 的错误，因为当 `close(2)` 返回给用户态时，描述符已经消失了。但 close 仍然是你最后一次注意到失败并记录它的机会，一些调用者可能会检查。最安全的模式是：

- 从普通关闭路径返回零。
- 只有当真正不寻常的事情发生且用户态应该知道时才返回非零 errno。
- 有疑问时，用 `device_printf(9)` 记录并返回零。

一个从 `d_close` 返回随机错误的驱动是一个测试会神秘失败的驱动，因为大多数用户态代码忽略关闭错误。把 errno 留给 open 和 ioctl，在那里它很重要。

### 将你的 errno 映射到用户消息

`/usr/include/errno.h` 中定义的值通过 `strerror(3)` 和 `perror(3)` 有稳定的文本表示。用户态程序中的每个 `err(3)` 和 `warn(3)` 消息都将使用这些。映射的简短表格：

| errno             | `strerror` 文本                   | 典型用户程序行为 |
|-------------------|-----------------------------------|-------------------------------|
| `ENXIO`           | Device not configured             | 等待或放弃；报告清楚 |
| `EBUSY`           | Device busy                       | 稍后重试或中止            |
| `EACCES`          | Permission denied                 | 提示 `sudo` 或退出       |
| `EPERM`           | Operation not permitted           | 类似于 `EACCES`             |
| `EINVAL`          | Invalid argument                  | 报告调用代码中的 bug      |
| `EAGAIN`          | Resource temporarily unavailable  | 短暂延迟后重试       |
| `EINTR`           | Interrupted system call           | 重试，通常在循环中        |
| `ENOENT`          | No such file or directory         | 验证驱动是否已加载         |
| `ENODEV`          | Operation not supported by device | 报告设计不匹配          |
| `EOPNOTSUPP`      | Operation not supported           | 报告设计不匹配          |

本书附录 E 收集了内核 errno 值的完整列表及其含义。对于第八章，上面的列表涵盖了你在设备文件界面上会需要的所有东西。

### 选择 errno 前的快速检查清单

当你不确定哪个 errno 合适时，问三个问题：

1. **问题是关于身份吗？** "此设备现在无法打开"是 `ENXIO`。"此设备不存在"是 `ENOENT`。很少是驱动的调用；devfs 通常处理它。
2. **问题是关于权限吗？** "你没有权限"是 `EACCES`。"你缺少特定特权"是 `EPERM`。
3. **问题是关于参数吗？** "调用结构良好但驱动不会接受这些参数"是 `EINVAL`。

当两个 errno 值都可能合适时，选择其文本表示与你希望沮丧用户阅读的内容匹配的那个。记住 errno 值变成你无法控制的工具中的错误消息，内核意图与面向用户文本之间的映射越清晰，你的驱动就会被审查得越友善。

### 简短叙述：三次选择 errno

为了让抽象具体化，这里是从真正的驱动审查对话中提取的三个小场景。每个都是关于单个 errno 值的选择。

**场景 1。过早的打开。**

一个驱动挂载板载传感器。传感器在上电后需要一百毫秒才能产生有效数据。在那百毫秒期间，尝试读取的用户程序会得到垃圾。

驱动初稿在预热窗口期间从 `d_open` 返回 `EAGAIN`。审查者标记了它。`EAGAIN` 意味着"稍后重试"，这没问题，但面向用户的文本是"Resource temporarily unavailable"，这与用户看到的不匹配：设备存在且原则上可以打开，但还没有产生数据。

修订稿在预热期间返回 `ENXIO`。用户看到"Device not configured"，这更接近真相。编写良好的用户态程序如果想要等待设备可以特殊处理该 errno。典型的工具会打印清晰的消息并退出。

教训：考虑用户看到什么，而不仅仅是你内部意味着什么。

**场景 2。错误的权限错误。**

一个驱动有一个可配置模式：sysctl 可以将其设置为"只读"。当 sysctl 设置后，`d_write` 返回错误。初稿返回 `EPERM`。审查者标记了它。`EPERM` 是关于特权的；内核在特定 `priv_check(9)` 调用失败时使用它。但在这个驱动中，没有执行特权检查；设备只是处于只读状态。

修订稿返回 `EROFS`、"Read-only file system"。文本映射对于这个场景几乎完美。

教训：越近的 errno 值通常是越好的 errno 值。不要默认为每个拒绝都用 `EPERM`。

**场景 3。繁忙的文件。**

一个强制执行独占访问的驱动在第二个打开者到达时从 `d_open` 返回 `EBUSY`。那是正确的。在代码审查中，一位审查者指出驱动还在进行中重新配置期间拒绝的控制节点 ioctl 上返回 `EBUSY`。审查论点是这些是不同的情况，两个 `EBUSY` 的使用会混淆正在阅读日志的操作员。

讨论达成了一个折衷：打开路径独占检查用 `EBUSY`，重新配置进行中的情况用 `EAGAIN`。区别在于打开路径拒绝是"将繁忙直到另一个用户关闭"，而重新配置拒绝是"稍等片刻，它会自己清除"。

教训：如果关于用户下一步行动的推理不同，两个感觉相似的情况可能映射到不同的 errno 值。

这些场景很小，但原则不小。每个 errno 值都是对用户下一步该做什么的提示。选择它时要看到用户的视角，而不仅仅是你自己的。

### 使用 `err(3)` 和 `warn(3)` 练习 errno 值

FreeBSD libc 中的 `err(3)` 家族在操作失败时打印干净的"程序：消息：errno 字符串"。你的用户态探测使用 `err(3)` 因为它是获得可读错误的最短路径。你可以通过运行故意触发每个错误的探测来验证你的驱动的 errno 选择：

```c
fd = open("/dev/myfirst/0", O_RDWR);
if (fd < 0)
        err(1, "open /dev/myfirst/0");
```

当驱动返回 `EBUSY` 时，程序打印：

```text
probe_myfirst: open /dev/myfirst/0: Device busy
```

当驱动返回 `ENXIO` 时，程序打印：

```text
probe_myfirst: open /dev/myfirst/0: Device not configured
```

对你能构造的每个错误情况运行探测。大声读出消息。如果其中任何一个会混淆没有读过你驱动源代码的用户，重新考虑 errno。

### 你的驱动几乎不应返回的 errno 值

为了平衡，列出很少适合设备文件打开或关闭的值：

- **`ENOMEM`**：让 `malloc` 调用通过通过你的函数返回它来报告这个，但不要发明它。
- **`EIO`**：保留给硬件 I/O 错误。如果你的设备没有硬件，这个值不合适。
- **`EFAULT`**：当用户态给内核一个坏指针时使用。在打开路径上你很少接触用户指针，所以 `EFAULT` 不合适。
- **`ESRCH`**："没有此类进程"。对于设备文件操作不太可能是正确的。
- **`ECHILD`**：进程关系 errno。不适用。
- **`EDOM`** 和 **`ERANGE`**：数学错误。不适用。

有疑问时，如果值没有出现在第八章前面的"简短列表"中，它几乎肯定是打开或关闭的错误。把不寻常的值保留给真正产生它们的特殊操作。



## 检查 /dev 的工具

几个小型实用工具值得了解，因为一旦你到达第九章，你将依赖它们快速确认行为。本节以足够使用的深度介绍每一个，并以两个简短的故障排除演练结束。

### ls -l 用于权限和存在性

第一站。`ls -l /dev/yourpath` 确认存在性、类型、所有权和模式。如果加载后节点缺失，你的 `make_dev_s` 可能失败了；检查 `dmesg` 获取错误码。

devfs 目录上的 `ls -l` 工作方式与你期望的一样：`ls -l /dev/myfirst` 列出子目录中的条目。结合 `-d`，它报告目录本身：

```sh
% ls -ld /dev/myfirst
dr-xr-xr-x  2 root  wheel  512 Apr 17 10:02 /dev/myfirst
```

devfs 子目录的模式默认是 `0555`，它不能通过 `devfs.conf` 直接配置。子目录存在只是因为里面至少有一个节点；当里面最后一个节点消失时，目录也消失。

### stat 和 stat(1)

`stat(1)` 打印任何节点的结构化视图。默认输出冗长，包括时间戳。更有用的形式是自定义格式：

```sh
% stat -f '%Sp %Su %Sg %T %N' /dev/myfirst/0
crw-rw---- root operator Character Device /dev/myfirst/0
```

占位符在 `stat(1)` 中有文档。上面五个是权限、用户名、组名、文件类型描述和路径。这种形式在需要稳定文本表示的脚本中很有用。

为了比较两个路径以检查它们解析到同一个 cdev，`stat -f '%d %i %Hr,%Lr'` 打印文件系统的设备、inode 和 `rdev` 的主次组件。在引用同一 cdev 的两个 devfs 节点上，`rdev` 组件将匹配。

### fstat(1)：谁打开了它？

`fstat(1)` 列出系统上每个打开的文件。过滤到设备路径，它告诉你哪些进程打开了该节点：

```sh
% fstat /dev/myfirst/0
USER     CMD          PID   FD MOUNT      INUM MODE         SZ|DV R/W NAME
root     probe_myfir  1234    3 /dev          4 crw-rw----   0,90 rw  /dev/myfirst/0
```

这是解决"`kldunload` 返回 `EBUSY` 但我不知道为什么"谜题的工具。针对你的节点运行它，识别有问题的进程，要么等它完成要么终止它。

`fstat -u username` 按用户过滤，当你怀疑特定用户的守护进程持有节点时有用。`fstat -p pid` 检查一个进程。

### procstat -f：进程优先视图

`fstat(1)` 列出文件并告诉你谁持有它们。`procstat -f pid` 做逆操作：它列出给定进程持有的文件。当你有一个运行程序的 PID 并想确认它当前打开了哪些设备节点时，这就是工具：

```sh
% procstat -f 1234
  PID COMM                FD T V FLAGS    REF  OFFSET PRO NAME
 1234 probe_myfirst        3 v c rw------   1       0     /dev/myfirst/0
```

列 `T` 显示文件类型（`v` 表示 vnode，包括设备文件），列 `V` 显示 vnode 类型（`c` 表示字符设备 vnode）。这是确认调试器显示给你的内容的最快方式。

### devinfo(8)：Newbus 侧

`devinfo(8)` 根本不看 devfs。它遍历 Newbus 设备树并打印设备层次结构。你的 `nexus0` 的 `myfirst0` 子设备在那里显示，无论 cdev 是否存在：

```sh
% devinfo -v
nexus0
  myfirst0
  pcib0
    pci0
      <...lots of PCI children...>
```

这是当 `/dev` 中缺少某些东西时你使用的工具，你需要检查设备本身是否挂载了。如果 `devinfo` 显示 `myfirst0` 但 `ls /dev` 不显示，你的 `make_dev_s` 失败了。如果两者都不显示设备，你的 `device_identify` 或 `device_probe` 没有创建子设备。两个不同的错误，两个不同的修复。

`-r` 标志过滤到以特定设备为根的 Newbus 层次结构，在有大量 PCI 设备的复杂系统中变得有用。

### devfs(8)：规则集和规则

`devfs(8)` 是 devfs 规则集的低级管理接口。你在第 10 节遇到了它。三个子命令经常出现：

- `devfs rule showsets` 列出当前加载的规则集编号。
- `devfs rule -s N show` 打印规则集 `N` 内的规则。
- `devfs rule -s N add path 'pattern' action args` 在运行时添加规则。

运行时添加的规则不持久；要使它们永久，添加到 `/etc/devfs.rules` 并运行 `service devfs restart`。

### sysctl dev.* 和其他层次结构

`sysctl dev.myfirst` 打印你的驱动命名空间下的每个 sysctl 变量。从第七章起，你已经有了一个 `dev.myfirst.0.stats` 树。读取它确认 softc 存在、挂载已运行、计数器正在推进。

Sysctl 是 `/dev` 的补充界面。它们主要用于可观测性；它们比打开设备更便宜；它们没有文件描述符成本。当一条信息简单到可以是数字或短字符串时，考虑将其暴露为 sysctl 而不是设备节点上的读取。

### kldstat：模块加载了吗？

当节点缺失时，"我的驱动甚至加载了吗？"这个问题值得先问。

```sh
% kldstat | grep myfirst
 8    1 0xffffffff82a00000     3a50 myfirst.ko
```

如果你在 `kldstat` 中看到模块，模块就在内核中。如果 `devinfo` 显示设备但 `ls /dev` 不显示节点，问题在你的驱动内部。如果 `kldstat` 不显示模块，问题在外面：你忘了 `kldload`，或加载失败。检查 `dmesg`。

### dmesg：发生了什么的日志

驱动的每个 `device_printf` 和 `printf` 调用最终进入内核消息缓冲区，`dmesg`（或 `dmesg -a`）打印它。当这个界面上出了问题时，`dmesg` 是首先查看的地方：

```sh
% dmesg | tail -20
```

你的挂载和拆离消息、任何 `make_dev_s` 失败、以及销毁路径的任何 panic 消息都落在这里。养成在开发期间用打开 `tail -f /var/log/messages` 的第二个终端监视 `dmesg` 的习惯。

### 故障排除演练 1：节点缺失

"我期望 `/dev/myfirst/0` 存在但它不存在"的清单。

1. 模块加载了吗？`kldstat | grep myfirst`。
2. 挂载运行了吗？`devinfo -v | grep myfirst`。
3. `make_dev_s` 成功了吗？`dmesg | tail` 应该显示你的挂载成功消息。
4. devfs 挂载在 `/dev` 上了吗？`mount | grep devfs`。
5. 你在正确的路径查看吗？如果你的格式字符串是 `"myfirst%d"`，节点是 `/dev/myfirst0`，不是 `/dev/myfirst/0`。拼写错误时有发生。
6. `devfs.rules` 条目在隐藏节点吗？`devfs rule showsets` 并检查。

十次中有九次，前三个问题之一就能得出答案。

### 故障排除演练 2：kldunload 返回 EBUSY

"我可以加载模块但不能卸载它"的清单。

1. 节点仍然打开吗？`fstat /dev/myfirst/0` 显示持有者。
2. 你的拆离自己返回 `EBUSY` 吗？检查 `dmesg` 中来自驱动的消息。阶段 2 的拆离在 `active_fhs > 0` 时返回 `EBUSY`。
3. `devfs.conf` 的 `link` 指向你的节点吗？如果目标被持有打开，链接可以保持引用。
4. 内核线程卡在你的处理程序之一内部了吗？在 `dmesg` 中寻找 `Still N threads in foo` 消息。如果存在，你需要一个 `d_purge`。

大多数 `EBUSY` 是打开的描述符。其他情况很少见。

### 关于习惯的说明

这些工具都不罕见。它们是 FreeBSD 管理的日常工具。重要的是当某些东西看起来不对时以已知顺序伸手去拿它们的习惯。前三次你调试缺失的节点，你会摸索正确的工具；第四次，顺序会感觉自动。在问题还小的时候现在就建立那种反射。



## 陷阱和值得注意的事项

一份捕捉初学者最常见的错误的实地指南。每个都命名了症状、原因和治疗方法。

- **在 softc 准备好之前创建设备节点。** *症状：* 驱动加载后立即打开导致 NULL 解引用。*原因：* `si_drv1` 仍然未设置，或 `open()` 查询的 softc 字段尚未初始化。*治疗方法：* 在 `make_dev_args` 中设置 `mda_si_drv1` 并在 `make_dev_s` 调用之前完成 softc 字段。把 `make_dev_s` 想成是发布，而不是准备。
- **在设备节点之前销毁 softc。** *症状：* 在 `kldunload` 期间或之后偶尔 panic。*原因：* 在 `detach()` 中颠倒了拆除顺序。*治疗方法：* 始终先销毁 cdev，然后是别名，然后是锁，然后是 softc。cdev 是门；在拆除门后面的房间之前先关上它。
- **在 cdev 上存储每次打开的状态。** *症状：* 一个用户时工作正常，两个用户时状态混乱。*原因：* 读取位置或类似的每次描述符数据存储在 `si_drv1` 或 softc 中。*治疗方法：* 将它们移动到 `struct myfirst_fh` 中并用 `devfs_set_cdevpriv` 注册。
- **忘记 `/dev` 的更改不持久。** *症状：* 你手工运行的 `chmod` 在重启或模块重新加载后消失。*原因：* devfs 是实时的，不是磁盘上的。*治疗方法：* 将更改放入 `/etc/devfs.conf` 并 `service devfs restart`。
- **拆离时泄漏别名。** *症状：* `kldunload` 返回 `EBUSY`，驱动卡住了。*原因：* 别名 cdev 仍然活跃。*治疗方法：* 在 `detach()` 中主设备之前对别名调用 `destroy_dev(9)`。
- **两次调用 `devfs_set_cdevpriv`。** *症状：* 第二次调用返回 `EBUSY`，你的处理程序将错误返回给用户。*原因：* `open` 中两个独立路径都试图注册私有数据，或处理程序为同一打开运行了两次。*治疗方法：* 审计代码路径，使每次 `d_open` 调用恰好一次成功的 `devfs_set_cdevpriv`。
- **分配 `fh` 但在错误路径上未释放。** *症状：* 与失败打开相关的稳定内存泄漏。*原因：* `devfs_set_cdevpriv` 返回了错误，分配被遗弃了。*治疗方法：* 在 `malloc` 之后和成功 `devfs_set_cdevpriv` 之前的任何错误上，显式 `free` 分配。
- **混淆别名和符号链接。** *症状：* 通过 `devfs.conf` 在 `link` 上设置的权限与驱动在主设备上公布的不匹配。*原因：* 在同一名称上混合两种机制。*治疗方法：* 每个名称选一个工具；当驱动拥有名称时使用别名，当操作员方便是目标时使用符号链接。
- **为"只是测试"使用宽开放模式。** *症状：* 以 `0666` 发送到预发布环境的驱动突然需要收紧而不破坏消费者。*原因：* 临时实验模式变成了默认值。*治疗方法：* 默认为 `0600`，仅在具体消费者请求时放宽，并在 `mda_mode` 行旁边的注释中注明原因。
- **在新代码中使用 `make_dev`。** *症状：* 驱动编译并工作，但审查者标记了调用。*原因：* `make_dev` 是家族中最旧的形式，失败时 panic。*治疗方法：* 使用带有填充的 `struct make_dev_args` 的 `make_dev_s`。较新的形式更易读、更易错误检查、对未来的 API 添加更友好。*如何更早捕获：* 在你的驱动上运行 `mandoc -Tlint` 并阅读 `make_dev(9)` 中的 `SEE ALSO`。
- **忘记 `D_VERSION`。** *症状：* 驱动加载但第一次 `open` 返回神秘失败，或内核打印 cdevsw 版本不匹配。*原因：* `cdevsw` 的 `d_version` 字段被留为零。*治疗方法：* 在每个 `cdevsw` 字面量中将 `.d_version = D_VERSION` 设为第一个字段。*如何更早捕获：* 包含该字段的代码模板使你永远不会键入没有它的 `cdevsw`。
- **因为"能编译"而发布带 `D_NEEDGIANT` 的代码。** *症状：* 驱动工作但每个操作都在 Giant 锁后序列化，使 SMP 密集型工作负载变慢。*原因：* 标志是从旧驱动复制的，或为消除警告而添加的，从未移除。*治疗方法：* 删除标志。如果你的驱动确实需要 Giant 才能维持，它有一个真正的锁定错误需要真正的修复，而不是一个标志。
- **在测试脚本中硬编码十六进制标识符。** *症状：* 测试在稍有不同的机器上失败，因为 `ls -l` 输出中的 `0x5a` 在那里不同。*原因：* devfs 的 `rdev` 标识符在重启、内核或系统之间不稳定。*治疗方法：* 比较两个路径的 `stat -f '%d %i'` 来检查别名等价性，而不是从 `ls -l` 抓取十六进制标识符。
- **假设 `devfs.conf` 在你的驱动加载之前运行。** *症状：* 驱动节点的 `devfs.conf` 行在 `kldload` 后不生效。*原因：* `service devfs start` 在启动早期运行，在运行时加载的模块之前。*治疗方法：* 加载驱动后 `service devfs restart`，或静态编译驱动使其节点在 devfs 启动前存在。
- **依赖包含非 POSIX 字符的节点名称。** *症状：* shell 脚本因引用错误而中断；`devfs.rules` 模式无法匹配。*原因：* 节点名称使用空格、冒号或非 ASCII 字符。*治疗方法：* 坚持使用小写 ASCII 字母、数字和三个分隔符 `/`、`-`、`.`。其他字符有时有效有时无效，而"有时无效"总是在最糟糕的时刻出现。
- **在 `d_open` 的错误路径上泄漏每次打开状态。** *症状：* 微妙的内存泄漏，通过运行压力测试数小时后检测到。*原因：* `malloc` 成功，`devfs_set_cdevpriv` 失败，分配被遗弃而未释放。*治疗方法：* `d_open` 中 `malloc` 和成功 `devfs_set_cdevpriv` 之间的每个错误路径必须 `free` 分配。先写错误路径，再写成功路径，是一个有用的习惯。
- **在同一次打开中两次注册 `devfs_set_cdevpriv`。** *症状：* 第二次调用返回 `EBUSY`，用户在打开时看到 `Device busy`，原因不明。*原因：* `d_open` 中两个独立的代码路径都试图附加私有数据，或打开处理程序为同一文件运行两次。*治疗方法：* 审计代码路径，使每次 `d_open` 调用恰好一次成功的 `devfs_set_cdevpriv`。如果驱动真的想替换数据，先使用 `devfs_clear_cdevpriv(9)`，但这几乎总是设计需要重新思考的信号。

### 真正关于生命周期的陷阱

一个单独的陷阱集群来自对生命周期的混淆。它们值得明确指出。

- **在 cdev 被销毁之前释放 softc。** *症状：* `kldunload` 后不久 panic，通常是处理程序中的 NULL 解引用或释放后使用。*原因：* 驱动在 `detach` 中在 `destroy_dev` 完成排空 cdev 之前拆除了 softc 状态，然后进行中的处理程序解引用了已释放的状态。*治疗方法：* 先销毁 cdev 并依赖其排空行为；只在那之后拆除 softc。*如何更早捕获：* 在观看 `dmesg` 的内核 panic 时运行任何压力测试；竞争在中等负载的 SMP 系统上很容易触发。
- **假设 `destroy_dev` 立即返回。** *症状：* 死锁，通常在持有锁然后调用最终需要相同锁的函数的处理程序中。*原因：* `destroy_dev` 阻塞直到进行中的处理程序返回；如果调用者持有其中一个处理程序需要的锁，系统死锁。*治疗方法：* 永远不要在持有进行中处理程序可能需要的锁时调用 `destroy_dev`。对于 `detach` 中的常见情况，什么也不要持有。
- **忘记在错误展开上设置 `is_attached = 0`。** *症状：* 失败的加载-卸载-重新加载循环后的微妙错误行为；处理程序认为设备仍然挂载并尝试使用已释放的状态。*原因：* 一个 `goto fail_*` 路径没有清除标志。*治疗方法：* 第七章的单标签展开模式；最后的 fail 标签总是在返回前清除 `is_attached`。

### 权限和策略中的陷阱

两类权限相关的错误倾向于在驱动发布很久后才出现。

- **因为用 `0600` 创建就假设节点"仅对 root 可见"。** *症状：* 安全审查将节点标记为可从不应看到它的 jail 访问。*原因：* 单独的模式不过滤 jail 可见性；`devfs.rules` 是过滤器，默认值可能足够包容以将节点传递给 jail。*治疗方法：* 如果节点绝不能在 jail 内可见，确保默认 jail 规则集隐藏它。`devfs_rules_hide_all` 是保守的起点。
- **依赖 `devfs.conf` 在共享实验室机器上保持节点秘密。** *症状：* 协作者更改 `devfs.conf`，节点变得对所有人可读。*原因：* `devfs.conf` 是操作员策略；任何对 `/etc` 有写访问权限的操作员都可以更改它。*治疗方法：* 驱动自己的基线应该在没有任何 `devfs.conf` 条目的情况下是安全的。将 `devfs.conf` 视为权限放宽器，永远不是相对于根本安全基线的权限收紧器。

### 可观测性中的陷阱

少数陷阱与代码无关，但与驱动有多容易调试密切相关。

- **以全音量记录每次打开和关闭。** *症状：* 内核消息缓冲区充满例程驱动噪音；真正的错误更难找到。*原因：* 驱动对每个 `d_open` 和 `d_close` 使用 `device_printf`。*治疗方法：* 用 `if (bootverbose)` 限制例程消息，或一旦驱动稳定就完全移除它们。将 `device_printf` 留给生命周期事件和真正的错误。
- **不暴露足够的 sysctl 来诊断异常状态。** *症状：* 用户报告 bug，你无法分辨驱动认为发生了什么，向驱动添加诊断需要重建和重新加载。*原因：* sysctl 树稀疏。*治疗方法：* 慷慨地暴露计数器。`active_fhs`、`open_count`、`read_count`、`write_count`、`error_count` 很便宜。添加 `attach_ticks` 和 `last_event_ticks` 让操作员分辨驱动运行了多久以及最近何时活动。



## 最终学习计划

如果你想在实验和挑战之外加深对材料的掌握，这里是在你完成章节后一周的建议计划。

**第 1 天：重读一节。** 选择第一次阅读时感觉最弱的任何单一章节，并在文本旁边打开配套树重读它。只是读。不要尝试编码。

**第 2 天：从头重建阶段 2。** 从第七章的阶段 2 源代码开始，逐一提交第八章阶段描述的每个更改。在每个阶段将你的工作与配套树进行比较。

**第 3 天：故意破坏驱动。** 一次引入三个不同的 bug：跳过析构函数、忘记销毁别名、返回错误的 errno。预测每个 bug 会做什么。运行探测。看看失败是否匹配你的预测。

**第 4 天：端到端阅读 `null.c` 和 `led.c`。** 两个小驱动，专注于设备文件界面。为每个写一段总结你注意到的东西。

**第 5 天：添加挑战 6 的状态节点。** 用目前手写的 `uiomove` 等价物实现只读状态节点；第九章将展示真正的习惯用法。

**第 6 天：尝试 jail 实验。** 如果你还没有做实验 8.7，现在做。Jail 值得设置的努力，因为后面的章节将假设熟悉。

**第 7 天：继续前进。** 不要等到感觉"掌握"了第八章。你会在后面的章节自然地回到它的材料。变得流利的方法是继续构建；变得卡住的方法是等待完美。



## 配套树快速参考

因为配套源代码树是本章教学方式的一部分，一个快速索引可以帮助你在实验和挑战期间找到东西。

### 驱动阶段

- `examples/part-02/ch08-working-with-device-files/stage0-structured-name/` 是实验 8.1 的输出：第七章的阶段 2 驱动，节点移动到 `/dev/myfirst/0`，所有权收紧为 `root:operator 0660`。
- `examples/part-02/ch08-working-with-device-files/stage1-alias/` 是实验 8.2 的输出：阶段 0 加上 `make_dev_alias("myfirst")`。
- `examples/part-02/ch08-working-with-device-files/stage2-perhandle/` 是实验 8.3 的输出：阶段 1 加上 `devfs_set_cdevpriv` 每次打开状态和移除独占打开检查。这是章节大多数其他练习使用的驱动。
- `examples/part-02/ch08-working-with-device-files/stage3-two-nodes/` 是实验 8.5 的输出：在 `/dev/myfirst/%d.ctl` 添加控制节点，有自己的 `cdevsw` 和更窄的权限模式。
- `examples/part-02/ch08-working-with-device-files/stage4-destroy-drain/` 是实验 8.8 的练习：一个多 cdev 驱动演示 `destroy_dev` 单独和 `destroy_dev_drain` 之间的区别。用 `make CFLAGS+=-DUSE_DRAIN=1` 构建正确变体。

### 用户态探测

- `userland/probe_myfirst.c`：一次性打开、读取、关闭。
- `userland/hold_myfirst.c`：打开并睡眠而不关闭，以在进程退出时锻炼 cdevpriv 析构函数。
- `userland/stat_myfirst.c`：报告一个或多个路径的 `stat(2)` 元数据；用于比较别名和主设备。
- `userland/parallel_probe.c`：从一个进程打开 N 个描述符、保持、关闭所有。
- `userland/stress_probe.c`：循环打开/关闭以抖出泄漏。
- `userland/devd_watch.sh`：订阅 `devd(8)` 事件并过滤 `myfirst`。

### 配置示例

- `devfs/devfs.conf.example`：实验 8.4 持久化条目。
- `devfs/devfs.rules.example`：实验 8.7 jail 规则集。
- `devfs/devd.conf.example`：挑战 5 devd 规则。
- `jail/jail.conf.example`：实验 8.7 引用规则集 100 的 jail 定义。

### 阶段如何不同

每个阶段都是针对第七章阶段 2 的差异。阅读章节后一个有用的第一个练习是在每对阶段之间运行 `diff` 并阅读结果。更改小到可以逐行理解，差异比重新阅读每个源文件更紧凑地讲述章节代码更改的递进故事。

```sh
% diff -u examples/part-02/ch07-writing-your-first-driver/stage2-final/myfirst.c \
         examples/part-02/ch08-working-with-device-files/stage0-structured-name/myfirst.c

% diff -u examples/part-02/ch08-working-with-device-files/stage0-structured-name/myfirst.c \
         examples/part-02/ch08-working-with-device-files/stage1-alias/myfirst.c

% diff -u examples/part-02/ch08-working-with-device-files/stage1-alias/myfirst.c \
         examples/part-02/ch08-working-with-device-files/stage2-perhandle/myfirst.c
```

每个差异应该是一小撮添加，没有意外的减法。如果你看到令人惊讶的更改，章节文本是推理所在。

### 关于稍后重用此树

这里的阶段不是要成为"最终"驱动。它们是对应章节检查点的快照。当你继续进入第九章时，你将就地编辑阶段 2，它将继续增长。当你到达第二部分末尾时，驱动已经演变成比任何单个阶段捕获的更丰富的东西。那就是重点：每一章添加一层，配套树在那里单独展示每一层，这样你可以看到进展。



## 关于界面的结束反思

本书的每一章都教授不同的东西，但少数几章教授贯穿驱动编写整个实践的东西。第八章是其中之一。具体的主题是设备文件，但更广泛的主题是**界面设计**：你如何塑造你控制的代码和你不控制的世界之间的边界？

UNIX 哲学有一个存活了半个世纪的答案。让边界看起来尽可能像普通文件。让 `open`、`read`、`write` 和 `close` 的现有词汇承担重任。选择名称和权限，使操作员可以在不阅读你的源代码的情况下推理它们。只暴露用户需要的，不要更多。如此积极地清理自己，以至于内核可以告诉你何时你丢失了某些东西的跟踪。用注释、`device_printf` 或 sysctl 记录你做的每个选择。

这些原则都不是设备文件独有的。它们再次出现在网络界面设计、存储分层、内核内部 API、与内核对话的用户态工具中。我们在 `/dev` 下的小界面上花了整整一章的原因是，同样的习惯，在这里在具体和有界的东西上实践，将服务于你继续接触的内核的每一层。

当你阅读 `/usr/src/sys` 中感觉优雅的驱动时，原因之一几乎总是其设备文件界面狭窄且诚实。当你阅读感觉混乱的驱动时，原因之一几乎总是其设备文件界面设计匆忙，或为短期压力放宽，从未再次收紧。本章的目标是帮助你注意到那个差异，并给你词汇和纪律来编写第一种驱动而不是第二种。



## 总结

你现在足够好地理解了驱动和用户空间之间的层面，可以有目的地塑造它了。具体来说：

- `/dev` 不是磁盘上的目录。它是内核活动对象的 devfs 视图。
- `struct cdev` 是你的节点的内核侧标识。vnode 是 VFS 到达它的方式。`struct file` 是单个 `open(2)` 在内核中的方式。
- `mda_uid`、`mda_gid` 和 `mda_mode` 设定 `ls -l` 显示的基线。`devfs.conf` 和 `devfs.rules` 在其上层叠操作员策略。
- 节点的路径是你的格式字符串说的任何内容，包括斜杠。`/dev` 下的子目录是分组相关节点的正常且受欢迎的方式。
- `make_dev_alias(9)` 让一个 cdev 响应多个名称。记住在拆除主设备时销毁别名。
- `devfs_set_cdevpriv(9)` 给每次 `open(2)` 自己的状态，带有自动清理。这是你在下一章中最依赖的工具。

你带入第九章的驱动是你开始时的同一个 `myfirst`，但有更清晰的名称、更合理的权限集和准备好承载读取位置、字节计数和真正 I/O 需要的小型簿记的每次打开状态。保持文件打开。你很快就会再次编辑它。

### 简短自我检查

在继续之前，确保你可以不回头看章节就回答以下每个问题。如果任何答案模糊，在开始第九章之前重新访问相关章节。

1. `struct cdev`、devfs vnode 和 `struct file` 之间有什么区别？
2. `make_dev_s(9)` 从哪里获取它创建节点的所有权和模式？
3. 为什么 `/dev/yournode` 上的 `chmod` 不能在重启后存活？
4. `make_dev_alias(9)` 做什么，它与 `devfs.conf` 中的 `link` 有何不同？
5. 用 `devfs_set_cdevpriv(9)` 注册的析构函数何时运行，何时不运行？
6. 你如何从用户态确认两个路径解析到同一个 cdev？
7. 为什么每个 `cdevsw` 都需要 `D_VERSION`，缺少时会发生什么？
8. 你何时会选择 `make_dev_s` 而不是 `make_dev_p`，为什么？
9. `destroy_dev(9)` 给你关于当前在处理程序内的线程什么保证？
10. 如果 jail 看不到 `/dev/myfirst/0` 但主机看到，隐藏它的策略在哪里，你会如何检查它？

如果你能用你自己的话回答所有十个，下一章会感觉像自然的延续而不是跳跃。

### 按主题组织的回顾

章节覆盖了很多。这里是按主题而不是按章节重新组织材料的简短内容，这样你可以锚定你学到的东西。

**关于内核和文件系统之间的关系：**

- devfs 是一个虚拟文件系统，将内核的 `struct cdev` 对象的实时集合呈现为 `/dev` 下的类似文件节点。
- 它没有磁盘存储。每个节点反映内核当前持有的东西。
- 它只支持对其节点的一小部分定义良好的操作集。
- 交互式进行的更改（例如用 `chmod`）不持久。持久策略位于 `/etc/devfs.conf` 和 `/etc/devfs.rules`。

**关于你的驱动与之交互的对象：**

- `struct cdev` 是设备节点的内核侧标识。每个节点一个，无论有多少文件描述符指向它。
- `struct cdevsw` 是你的驱动提供的调度表。它将每种操作映射到你代码中的处理程序。
- `struct file` 和 devfs vnode 位于用户的文件描述符和你的 cdev 之间。它们承载每次打开状态并路由操作。

**关于创建和销毁节点：**

- `make_dev_s(9)` 是现代推荐的创建 cdev 的方式。填写 `struct make_dev_args`，传入，取回一个 cdev。
- `make_dev_alias(9)` 为现有 cdev 创建第二个名称。别名是一流的 cdev；内核使它们与主设备保持同步。
- `destroy_dev(9)` 同步销毁 cdev，排空进行中的处理程序。它的兄弟 `destroy_dev_sched` 和 `destroy_dev_drain` 分别覆盖延迟和清扫情况。

**关于每次打开状态：**

- `devfs_set_cdevpriv(9)` 将驱动提供的指针连同析构函数附加到当前文件描述符。
- `devfs_get_cdevpriv(9)` 在稍后的处理程序内检索该指针。
- 析构函数每次成功的 `set` 调用恰好触发一次，当文件描述符的最后引用丢弃时。
- 这是现代 FreeBSD 驱动中每次打开簿记的主要机制。

**关于策略：**

- 驱动在调用 `make_dev_s` 时设置基线模式、uid 和 gid。
- `/etc/devfs.conf` 可以在主机 devfs 挂载上按节点调整这些。
- `/etc/devfs.rules` 可以定义按挂载过滤和调整的命名规则集，通常用于 jail。
- 三层可以作用于同一个 cdev，顺序很重要。

**关于用户态：**

- `ls -l`、`stat(1)`、`fstat(1)`、`procstat(1)`、`devinfo(8)`、`devfs(8)`、`sysctl(8)` 和 `kldstat(8)` 是检查和操作你的驱动暴露的界面的日常工具。
- 打开、读取、关闭和 `stat` 设备的小型用户态 C 程序值得编写。它们给你对时序的控制，让你干净地测试边缘情况。

**关于纪律：**

- 默认为狭窄权限，仅在具体消费者请求时放宽。
- 使用命名常量（`UID_ROOT`、`GID_WHEEL`）而不是原始数字。
- 按创建的相反顺序销毁。
- 在返回前的每个错误路径上释放分配。
- 用 `device_printf(9)` 记录生命周期事件，这样 `dmesg` 讲述你的驱动正在做什么的故事。

那是很多。你不需要一次保持所有。实验和挑战是材料变成肌肉记忆的地方；文本只是阅读指南。

### 展望第九章

在第九章中，我们将正确填充 `d_read` 和 `d_write`。你将学习内核如何用 `uiomove(9)` 在用户内存和内核内存之间移动字节，为什么 `struct uio` 看起来是那样，以及如何设计一个对短读取、短写入、未对齐缓冲区和行为不端的用户程序安全的驱动。你刚布线的每次打开状态将承载读取偏移和写入状态。别名将保持旧用户界面在驱动增长时继续工作。你在这里设置的权限模型将在你开始发送真正数据时保持你的实验脚本诚实。

具体来说，第九章将需要你添加到 `struct myfirst_fh` 的字段用于两件事。`reads` 计数器将增加一个匹配的 `read_offset` 字段，使每个描述符记住它在合成数据流中的位置。`writes` 计数器将由一个小型环形缓冲区补充，`d_write` 向其中追加而 `d_read` 从中排空。你在每个处理程序中用 `devfs_get_cdevpriv` 检索的 `fh` 指针将是所有这些状态的入口。

你在实验 8.2 中创建的别名将在不做任何更改的情况下继续工作：`/dev/myfirst` 和 `/dev/myfirst/0` 都将产生数据，描述符之间的每次状态将是独立的。

你在实验 8.1 和 8.4 中设置的权限将仍然是开发的正确默认值：足够紧凑以在原始用户触及设备时强制有意识的 `sudo`，足够开放以 `operator` 组中的测试工具可以运行数据路径测试而无需升级。

你建造了一扇形状良好的门。在下一章中，门后面的房间将活跃起来。



## 参考：make_dev_s 和 cdevsw 一览

本参考将最有用的声明和标志值收集在一处，交叉引用到解释每一个的章节部分。编写自己的驱动时保持打开它；花费一天的大多数错误都是关于这些值之一的错误。

### 规范的 make_dev_s 骨架

单节点驱动的规范模板：

```c
struct make_dev_args args;
int error;

make_dev_args_init(&args);
args.mda_devsw   = &myfirst_cdevsw;
args.mda_uid     = UID_ROOT;
args.mda_gid     = GID_OPERATOR;
args.mda_mode    = 0660;
args.mda_si_drv1 = sc;

error = make_dev_s(&args, &sc->cdev, "myfirst/%d", sc->unit);
if (error != 0) {
        device_printf(dev, "make_dev_s: %d\n", error);
        /* unwind and return */
        goto fail;
}
```

### 规范的 cdevsw 骨架

```c
static struct cdevsw myfirst_cdevsw = {
        .d_version = D_VERSION,
        .d_name    = "myfirst",
        .d_open    = myfirst_open,
        .d_close   = myfirst_close,
        .d_read    = myfirst_read,
        .d_write   = myfirst_write,
        .d_ioctl   = myfirst_ioctl,     /* 在第二十五章添加 */
        .d_poll    = myfirst_poll,      /* 在第十章添加 */
        .d_kqfilter = myfirst_kqfilter, /* 在第十章添加 */
};
```

省略的字段等同于 `NULL`，内核将其解释为"不支持"或"使用默认行为"，具体取决于哪个字段。

### make_dev_args 结构

来自 `/usr/src/sys/sys/conf.h`：

```c
struct make_dev_args {
        size_t         mda_size;         /* 由 make_dev_args_init 设置 */
        int            mda_flags;        /* MAKEDEV_* 标志 */
        struct cdevsw *mda_devsw;        /* 必需 */
        struct ucred  *mda_cr;           /* 通常为 NULL */
        uid_t          mda_uid;          /* 参见 conf.h 中的 UID_* */
        gid_t          mda_gid;          /* 参见 conf.h 中的 GID_* */
        int            mda_mode;         /* 八进制模式 */
        int            mda_unit;         /* 单元号 (0..INT_MAX) */
        void          *mda_si_drv1;      /* 通常是 softc */
        void          *mda_si_drv2;      /* 第二个驱动指针 */
};
```

### MAKEDEV 标志字

| 标志                   | 含义                                                 |
|------------------------|---------------------------------------------------------|
| `MAKEDEV_REF`          | 创建时添加额外引用。                     |
| `MAKEDEV_NOWAIT`       | 不为内存睡眠；如果紧张返回 `ENOMEM`。      |
| `MAKEDEV_WAITOK`       | 为内存睡眠（`make_dev_s` 的默认值）。            |
| `MAKEDEV_ETERNAL`      | 将 cdev 标记为永不销毁。                 |
| `MAKEDEV_CHECKNAME`    | 验证名称；对错误名称返回错误。           |
| `MAKEDEV_WHTOUT`       | 创建 whiteout 条目（堆叠文件系统）。          |
| `MAKEDEV_ETERNAL_KLD`  | 静态时为 `MAKEDEV_ETERNAL`，作为 KLD 构建时为零。  |

### cdevsw d_flags 字段

| 标志             | 含义                                                          |
|------------------|------------------------------------------------------------------|
| `D_TAPE`         | 类别提示：磁带设备。                                      |
| `D_DISK`         | 类别提示：磁盘设备（旧式；现代磁盘使用 GEOM）。      |
| `D_TTY`          | 类别提示：TTY 设备。                                       |
| `D_MEM`          | 类别提示：内存设备，如 `/dev/mem`。                 |
| `D_TRACKCLOSE`   | 对每个描述符的每次 `close(2)` 调用 `d_close`。         |
| `D_MMAP_ANON`    | 此 cdev 的匿名 mmap 语义。                          |
| `D_NEEDGIANT`    | 强制 Giant 锁调度。在新代码中避免。                    |
| `D_NEEDMINOR`    | 驱动使用 `clone_create(9)` 分配次设备号。       |

### 常见的 UID 和 GID 常量

| 常量       | 数值 | 用途                                    |
|----------------|---------|--------------------------------------------|
| `UID_ROOT`     | 0       | 超级用户。大多数节点的默认所有者。   |
| `UID_BIN`      | 3       | 守护进程可执行文件。                        |
| `UID_UUCP`     | 66      | UUCP 子系统。                            |
| `UID_NOBODY`   | 65534   | 非特权占位符。                  |
| `GID_WHEEL`    | 0       | 受信任的管理员。                    |
| `GID_KMEM`     | 2       | 读取内核内存的权限。              |
| `GID_TTY`      | 4       | 终端设备。                     |
| `GID_OPERATOR` | 5       | 操作工具。                         |
| `GID_BIN`      | 7       | 守护进程拥有的文件。                        |
| `GID_VIDEO`    | 44      | 视频帧缓冲区访问。                  |
| `GID_DIALER`   | 68      | 串口拨出程序。             |
| `GID_NOGROUP`  | 65533   | 无组。                                  |
| `GID_NOBODY`   | 65534   | 非特权占位符。                  |

### 销毁函数

| 函数                           | 何时使用                                                    |
|------------------------------------|----------------------------------------------------------------|
| `destroy_dev(cdev)`                | 普通、带排空的同步销毁。                  |
| `destroy_dev_sched(cdev)`          | 当你不能睡眠时的延迟销毁。                    |
| `destroy_dev_sched_cb(cdev,cb,arg)`| 带后续回调的延迟销毁。                |
| `destroy_dev_drain(cdevsw)`        | 在释放 cdevsw 之前等待其所有 cdev 完成。 |
| `delist_dev(cdev)`                 | 从 devfs 移除 cdev 但尚未完全销毁。         |

### 每次打开状态函数

| 函数                                   | 用途                                           |
|--------------------------------------------|---------------------------------------------------|
| `devfs_set_cdevpriv(priv, dtor)`           | 将私有数据附加到当前描述符。    |
| `devfs_get_cdevpriv(&priv)`                | 检索当前描述符的私有数据。 |
| `devfs_clear_cdevpriv()`                   | 提前拆离并运行析构函数。              |
| `devfs_foreach_cdevpriv(dev, cb, arg)`     | 遍历 cdev 上的所有每次打开记录。           |

### 别名函数

| 函数                                             | 用途                                    |
|------------------------------------------------------|--------------------------------------------|
| `make_dev_alias(pdev, fmt, ...)`                     | 为主 cdev 创建别名。        |
| `make_dev_alias_p(flags, &cdev, pdev, fmt, ...)`     | 创建带标志和错误返回的别名。|
| `make_dev_physpath_alias(...)`                       | 创建拓扑路径别名。              |

### 引用计数辅助函数

通常不被驱动直接调用。此处列出以供识别。

| 函数                         | 用途                                                |
|----------------------------------|--------------------------------------------------------|
| `dev_ref(cdev)`                  | 获取长期引用。                        |
| `dev_rel(cdev)`                  | 释放长期引用。                        |
| `dev_refthread(cdev, &ref)`      | 为处理程序调用获取引用。                |
| `dev_relthread(cdev, ref)`       | 释放处理程序调用的引用。                  |

### 去哪里阅读更多

- `make_dev(9)`、`destroy_dev(9)`、`cdev(9)` 手册页了解 API 界面。
- `devfs(5)`、`devfs.conf(5)`、`devfs.rules(5)`、`devfs(8)` 了解文件系统层文档。
- `/usr/src/sys/sys/conf.h` 了解规范的结构和标志定义。
- `/usr/src/sys/kern/kern_conf.c` 了解 `make_dev*` 家族的实现。
- `/usr/src/sys/fs/devfs/devfs_vnops.c` 了解 `devfs_set_cdevpriv` 和相关函数的实现。
- `/usr/src/sys/fs/devfs/devfs_rule.c` 了解规则子系统。

本参考故意保持简短。章节是推理所在的地方；此节只是查找表。

### 简明模式目录

下表总结了章节展示的主要模式，每个都配对解释它的章节部分。当你正在构建驱动中途需要快速定位时，先扫描这个列表。

| 模式                                           | 章节中的部分                                |
|---------------------------------------------------|-------------------------------------------------------|
| 在 `attach` 中创建一个数据节点，在 `detach` 中销毁 | 第七章，在第八章实验 8.1 中引用           |
| 将节点移动到 `/dev` 下的子目录   | 命名、单元号和子目录             |
| 同时暴露数据节点和控制节点       | 每设备多个节点；实验 8.5                   |
| 添加别名使驱动在两条路径上应答  | 别名：一个 cdev，多个名称；实验 8.2       |
| 在操作员级别放宽或收窄权限| 持久策略；实验 8.4                           |
| 在 jail 内隐藏或暴露节点              | 持久策略；实验 8.7                           |
| 给每个打开自己的状态和计数器        | 用 `devfs_set_cdevpriv` 的每次打开状态；实验 8.3    |
| 运行对崩溃安全的预打开分配 | 每次打开状态；挑战 2                     |
| 用 `EBUSY` 强制独占打开              | 错误码；配方 1                                |
| 在一次拆离中拆除多个 cdev               | 安全销毁 cdev；实验 8.8                     |
| 通过 devd 在用户态响应节点创建      | 从用户态操作你的设备；挑战 5    |
| 比较两条路径以验证它们共享 cdev    | 从用户态操作你的设备                 |
| 通过 sysctl 暴露驱动状态               | 实用工作流；第七章引用             |

每行命名一个模式。每个模式在章节某处有一个简短配方。当你面对设计问题时，找到合适的行并跟随链接回去。

### 按操作的常见 errno 值

哪个操作常规使用哪些 errno 值的紧凑交叉引用。与第 13 节配对。

| 操作                | 常见 errno 返回                                       |
|--------------------------|------------------------------------------------------------|
| `d_open`                 | `0`、`ENXIO`、`EBUSY`、`EACCES`、`EPERM`、`EINVAL`、`EAGAIN`|
| `d_close`                | 几乎总是 `0`；记录异常情况，不要返回它们 |
| `d_read`                 | 成功时 `0`，设备消失时 `ENXIO`，坏缓冲区 `EFAULT`，信号 `EINTR`，非阻塞重试 `EAGAIN` |
| `d_write`                | 与 `d_read` 相同系列，加上空间不足 `ENOSPC`    |
| `d_ioctl`（第二十五章）   | 成功时 `0`，未知命令 `ENOTTY`，坏参数 `EINVAL` |
| `d_poll`（第十章）    | 返回 revents 掩码，不是 errno                       |

你的第八章驱动主要与前两行有关。第九章将扩展到第三和第四行。

### 章节中使用术语的简短词汇表

供没有见过每个术语或想要快速提醒的读者使用。

- **cdev**：设备文件的内核侧标识，每个节点一个。
- **cdevsw**：将操作映射到驱动处理程序的调度表。
- **cdevpriv**：通过 `devfs_set_cdevpriv(9)` 附加到文件描述符的每次打开状态。
- **devfs**：将 cdev 作为 `/dev` 下节点呈现的虚拟文件系统。
- **mda_***：传递给 `make_dev_s(9)` 的 `make_dev_args` 结构的成员。
- **softc**：由 Newbus 分配并可通过 `device_get_softc(9)` 访问的每设备私有数据。
- **SI_***：存储在 `struct cdev` 的 `si_flags` 字段中的标志。
- **D_***：存储在 `struct cdevsw` 的 `d_flags` 字段中的标志。
- **MAKEDEV_***：通过 `mda_flags` 传递给 `make_dev_s(9)` 及其相关函数的标志。
- **UID_*** 和 **GID_***：标准用户和组身份的符号常量。
- **destroy_dev_drain**：卸载创建了许多 cdev 的模块时使用的 cdevsw 级排空函数。
- **devfs.conf**：用于持久节点所有者和模式的主机级策略文件。
- **devfs.rules**：塑造 devfs 每挂载视图的规则集文件，主要用于 jail。

词汇表将随书的进展而增长。第八章介绍了它需要的大部分术语；后续章节将添加自己的并引用回此列表。



## 巩固和回顾

在你放下章节之前，再过一遍材料是值得的。本节以逐节结构无法完全做到的方式将各部分联系在一起。

### 最重要的三个想法

如果你只能从第八章记住三件事，让它们是这些：

**第一，`/dev` 是内核维护的实时文件系统。** 每个节点都由你的驱动拥有的 `struct cdev` 支持。你在 `/dev` 中看到的任何东西都不是持久的；它是内核当前状态的窗口。当你编写驱动时，你正在添加和移除该窗口，内核诚实地反映你的更改。

**第二，设备文件界面是你驱动公共接口的一部分。** 名称、权限、所有权、别名的存在、你实现的操作集、你返回的 errno 值、销毁顺序，所有这些都是用户依赖的决策。从一开始就将它们视为契约。事后放宽或收紧总是比第一次选择正确的基线更具破坏性。

**第三，每次打开状态是每次描述符信息的正确归宿。** `devfs_set_cdevpriv(9)` 存在是因为 UNIX 的描述符模型比单个 softc 可以表示的更具表达力。当两个进程打开同一个节点时，它们每个都值得有自己的视图。给它们每次打开状态花费一个小分配和一个析构函数；替代方案是一个你不愿调试的共享状态竞争迷宫。

第八章的其他所有内容都阐述了这三个想法之一。

### 你结束章节时的驱动形状

到实验 8.8 结束时，你的 `myfirst` 驱动已经成长为比第七章结束时更像真正的 FreeBSD 驱动的东西。具体来说：

- 它有一个 softc、一个互斥锁和一个 sysctl 树。
- 它在 `/dev` 下的子目录中创建节点，有意选择所有权和模式。
- 它为旧名称提供别名，以便现有用户继续工作。
- 它在每次 `open(2)` 时分配每次打开状态，并在所有情况下通过可靠触发的析构函数清理它。
- 它计数活动打开并在任何仍然存活时拒绝拆离。
- 它在 `detach` 期间以合理的顺序销毁其 cdev。

那就是 `/usr/src/sys/dev` 中几乎每个小型驱动的形状。你不需要从头构建你编写的每个驱动；大多数时候，你将从一个看起来完全像这样的模板开始，并在其上添加子系统特定的逻辑。

### 开始第九章前要练习什么

巩固章节材料的简短练习列表，按递增的延伸程度大致排序：

1. **不看配套树，逐阶段重建 `myfirst`。** 打开第七章的阶段 2 源代码。从头做实验 8.1 的更改。然后实验 8.2 的更改。然后实验 8.3。将你的结果与配套树的阶段 2 源代码比较。差异是值得理解的东西。
2. **故意破坏一个阶段。** 在实验 8.3 中引入故意的 bug（例如，跳过 `devfs_set_cdevpriv` 调用）。预测加载和运行并行探测时会发生什么。运行它。看看失败是否匹配你的预测。
3. **添加第三个 cdev。** 用服务不同命名空间的第二个控制节点扩展实验 8.5 的阶段 3 驱动。观察节点与驱动同步出现和消失。
4. **编写用户态服务。** 编写一个小型守护进程，启动时打开 `/dev/myfirst/0`，保持描述符，并通过读取和记录来响应 SIGUSR1。安装它。在驱动加载和卸载时测试它。注意当驱动在守护进程仍有描述符打开时卸载时会发生什么。
5. **阅读新驱动。** 从 `/usr/src/sys/dev` 中选择一个你还没有触及的驱动，通过设备文件透镜阅读它，并使用第 15 节的决策树分类它。写一段描述你发现的内容。

每个练习需要三十分钟到一小时。做其中两三个足以将章节材料从"我读过一次"移动到"我对它感到舒适"。做全部五个给你将服务于本书其余部分的直觉。

第九章紧随其后。门后的房间活跃起来。
