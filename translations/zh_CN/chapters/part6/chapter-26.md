---
title: "USB和串行驱动程序"
description: "第26章 opens 第6部分 by teaching transport-specific 驱动程序 development through USB and serial 设备. It explains what makes USB and 串行驱动程序 different from the generic character 驱动程序 built earlier in the book; introduces the USB mental model (host and 设备 roles, 设备 classes, 描述符, 接口, 端点, and the four transfer types); introduces the serial mental model (UART hardware, RS-232-style framing, 波特率, 奇偶校验, 流控制, and the FreeBSD tty discipline); walks through the organisation of FreeBSD's USB subsystem and the registration idioms that 驱动程序 use to 附加 to `uhub`; shows how a USB驱动程序 sets up bulk, interrupt, and 控制传输 through `usbd_transfer_setup` and handles them in 回调 that follow the `USB_GET_STATE` state machine; explains how a USB驱动程序 may expose a user-visible `/dev` 接口 through `usb_fifo` or a custom `cdevsw`; contrasts FreeBSD's two serial worlds, the `uart(4)` subsystem for real UART hardware and the `ucom(4)` 框架 for USB-to-serial bridges; teaches how 波特率, 奇偶校验, 停止位, and RTS/CTS 流控制 are carried through `struct termios` and programmed into hardware; and shows how to test USB and 串行驱动程序 behaviour without ideal physical hardware using `nmdm(4)`, `cu(1)`, `usb_template(4)`, QEMU USB redirection, and the existing 内核's own loopback facilities. The `myfirst` 驱动程序 gains a new transport-specific sibling, `myfirst_usb`, at version 1.9-usb, which 探测 a vendor/product identifier pair, 附加 on 设备 insertion, sets up one bulk-in and one bulk-out transfer, echoes received bytes back through a /dev node, and unwinds cleanly on 热拔出. The chapter prepares the reader for 第27章 (storage and the VFS layer) by establishing the two mental models a reader will reuse everywhere in 第6部分: a transport is a protocol plus a lifecycle, and a FreeBSD transport-specific 驱动程序 is a New总线 驱动程序 whose resources happen to be 总线 端点 rather than PCI BARs."
partNumber: 6
partName: "编写特定传输层驱动程序"
chapter: 26
lastUpdated: "2026-04-19"
status: "complete"
author: "Edson Brandi"
reviewer: "TBD"
translator: "AI辅助翻译为简体中文"
estimatedReadTime: 300
language: "zh-CN"
---

# USB和串行驱动程序

## 引言

第25章 closed 第5部分 with a 驱动程序 that the rest of the system could talk to. The `myfirst` 驱动程序 at version `1.8-maintenance` had a rate-limited logging macro, a careful errno vocabulary, loader tunables and writable sysctls, a three-way version split, a labelled-goto cleanup chain in 附加 and 分离, a clean modular source layout, `MODULE_DEPEND` and `MODULE_PNP_INFO` metadata, a `MAINTENANCE.md` document, a `shutdown_pre_sync` event handler, and a regression script that could load and unload the 驱动程序 a hundred times without leaking a single resource. What the 驱动程序 did not have was any contact with real hardware. The 字符设备 backed a 缓冲区 in 内核 memory. The sysctl counters tracked operations against that 缓冲区. The `MYFIRSTIOC_GETCAPS` ioctl announced capabilities that were implemented entirely in software. Everything the 驱动程序 did, it did without ever reading a byte off a wire.

第26章 begins the step outward. Instead of serving a 缓冲区 in RAM, the 驱动程序 will 附加 to a real 总线 and service a real 设备. The 总线 will be the Universal Serial Bus, because USB is the most approachable transport in FreeBSD: it is ubiquitous, its subsystem is extremely well organised, the 内核 接口 is designed around a small handful of structures and macros, and every FreeBSD developer already has a dozen USB 设备 on their desk. After USB, the chapter pivots to the subject that historically preceded USB and still lives alongside it everywhere from debug consoles to GPS modules: the 串行端口, in its classical form as a UART-driven RS-232 接口 and in its modern form as a USB-to-serial bridge. By the end of the chapter, the `myfirst` 驱动程序 family has grown a new transport-specific sibling, `myfirst_usb`, at version `1.9-usb`. That sibling knows how to 附加 to a real USB 设备, how to set up a bulk-in and a bulk-out transfer, how to echo received bytes through a `/dev` node, and how to survive the 设备 being yanked out of the port while the 驱动程序 is in use.

第26章 is the opening chapter of 第6部分. 第6部分 is organised around the observation that up to this point the book has been teaching the parts of FreeBSD 驱动程序 development that are *transport-neutral*: the New总线 model, the 字符设备 接口, synchronisation, interrupts, DMA, power management, debugging, integration, and maintenance. All of those disciplines apply to every 驱动程序 regardless of how the 设备 is 附加ed. 第6部分 shifts the focus. USB, storage, and networking each have their own 总线, their own lifecycle, their own data-flow pattern, and their own idiomatic way of integrating with the rest of the 内核. The disciplines you have built in Parts 1 through 5 carry over unchanged; what is new is the shape of the 接口 between your 驱动程序 and the specific subsystem it plugs into. 第26章 teaches that shape for USB and for serial 设备. 第27章 teaches it for storage 设备 and the VFS layer. 第28章 teaches it for 网络接口. Each of the three chapters is structurally parallel: the transport is introduced, the subsystem is mapped, a minimal 驱动程序 is built, and the reader is shown how to test without the specific hardware everyone happens not to have.

There is a deliberate pairing of USB and serial in this chapter. The two topics sit together because they are both first-class citizens of the same larger mental model: a transport is a *protocol plus a lifecycle*, and the 驱动程序 is the piece of code that carries data across the protocol boundary and keeps the lifecycle consistent with the 内核's view of the 设备. USB is a protocol with a rich four-transfer-type vocabulary and a 热插拔 lifecycle. A UART is a protocol with a much simpler byte-framing vocabulary and a statically-附加ed lifecycle. Studying them together makes the pattern visible. A student who has seen the USB 回调 state machine and the UART 中断处理程序 side by side understands that "FreeBSD's 驱动程序 model" is not a single shape but a family of shapes, each one adapted to the demands of its own transport.

The second reason to pair USB and serial is historical and practical. A very large number of what the operating system calls "USB 设备" are in fact 串行端口s in disguise. The FTDI FT232R chip, the Prolific PL2303, the Silicon Labs CP210x, and the WCH CH340 all expose a standard serial-port API to user space, but physically they sit on the USB 总线. FreeBSD handles that with the `ucom(4)` 框架: a USB驱动程序 寄存器 回调 with `ucom`, and `ucom` produces the user-visible `/dev/ttyU0` and `/dev/cuaU0` 设备 nodes and arranges for the termios-aware line discipline to operate correctly on top of a USB bulk-in and bulk-out pair. The reader who is about to write a USB驱动程序 is likely, sooner or later, to write a USB-to-串行驱动程序, and that 驱动程序 will be an intersection of the two worlds the chapter introduces. Putting the material into a single chapter makes the intersection visible.

A third reason is pedagogical. The `myfirst` 驱动程序 so far has been a pseudo-设备. The transition to real hardware is a conceptual step, not just a coding step. Many readers will find their first attempt at a hardware-backed 驱动程序 unsettling: interrupts arrive without asking, transfers can stall or time out, the 设备 can be unplugged mid-operation, and the 内核 has opinions about how fast you are allowed to respond. USB is the friendliest possible 引言 to that world because the USB subsystem does an unusually large 挂载 of work on the 驱动程序's behalf. Setting up a 批量传输 in USB is not the same kind of problem as setting up a DMA ring on a PCI Express NIC. The USB core manages the low-level DMA bookkeeping; your 驱动程序 works at the level of "tell me when this transfer completes". Learning the USB pattern first makes the later hardware chapters (storage, networking, embedded 总线es in Part 7) less intimidating because by then the basic shape of a transport-specific 驱动程序 is familiar.

The `myfirst` 驱动程序's path through this chapter is concrete. It picks up at version `1.8-maintenance` from the end of 第25章. It adds a new source file, `myfirst_usb.c`, compiled into a new 内核模块, `myfirst_usb.ko`. The new module declares itself dependent on `usb`, lists a single vendor and product identifier in its 匹配表, 探测 and 附加 on 热插拔, allocates one bulk-in and one bulk-out transfer, exposes a `/dev/myfirst_usb0` node, echoes incoming bytes to the 内核 log and copies them back out on a read, handles 分离 cleanly when the cable is pulled, and carries forward every 第25章 discipline without exception. The labs exercise each piece in turn. By the end of the chapter, there is a second 驱动程序 in the family, a new source layout to accommodate it, and a working example of a FreeBSD USB驱动程序 that the reader has typed themselves.

Because this is also a chapter about serial 设备, the chapter spends time on the serial half of its scope even though `myfirst_usb` itself is not a 串行驱动程序. The serial material teaches how `uart(4)` is laid out, how `ucom(4)` fits in, how termios carries 波特率 and 奇偶校验 and 流控制 from user space down to hardware, and how to test serial 接口 without physical hardware using `nmdm(4)`. The serial material does not build a new UART hardware 驱动程序 from scratch. Writing a UART hardware 驱动程序 is a specialised undertaking that is almost never the right choice in a modern environment: the existing `ns8250` 驱动程序 in the base system already handles every PC-compatible 串行端口, every common PCI serial card, and the ARM PL011 that most virtualised platforms present. The chapter teaches the serial subsystem at the level the reader actually needs: how it is organised, how to read existing 驱动程序, how termios reaches a 驱动程序's `param` method, how to use the subsystem from user space, and what to do when the goal is a USB-to-串行驱动程序 (the common case) rather than a new hardware 驱动程序 (the rare case).

The rhythm of 第26章 is the rhythm of pattern recognition. The reader will leave the chapter knowing what a USB驱动程序 looks like, what a 串行驱动程序 looks like, where the two overlap, where they differ from the pseudo-驱动程序 of Parts 2 through 5, and how to test both without a lab full of adapters. Those are the foundations of transport-specific 驱动程序 development. 第27章 will then apply the same discipline to storage, and 第28章 will apply it to networking, each time taking the same general pattern and bending it to a new transport's rules.

### Where 第25章 Left the Driver

A short checkpoint before the new work starts. 第26章 extends the 驱动程序 family produced at the end of 第25章, tagged as version `1.8-maintenance`. If any of the items below is uncertain, return to 第25章 and resolve it before starting this chapter, because the new material assumes every 第25章 primitive is working and every habit is in place.

- Your 驱动程序 source matches 第25章 Stage 4. `myfirst.ko` compiles cleanly, identifies itself as `1.8-maintenance` in `kldstat -v`, and carries the full `MYFIRST_VERSION`, `MODULE_VERSION`, and `MYFIRST_IOCTL_VERSION` triple.
- The source layout is split: `myfirst_总线.c`, `myfirst_cdev.c`, `myfirst_ioctl.c`, `myfirst_sysctl.c`, `myfirst_debug.c`, `myfirst_log.c`, with `myfirst.h` as the shared private header.
- The rate-limited log macro `DLOG_RL` is in place and tied to a `struct myfirst_ratelimit` inside the softc.
- The `goto fail;` cleanup chain in `myfirst_附加` is working and exercised by a deliberate failure lab.
- The regression script passes a hundred consecutive `kldload`/`kldunload` cycles with no residual OIDs, no orphaned cdev nodes, and no leaked memory.
- Your lab machine runs FreeBSD 14.3 with `/usr/src` on disk, a debug 内核 with `INVARIANTS`, `WITNESS`, `WITNESS_SKIPSPIN`, `DDB`, `KDB`, `KDB_UNATTENDED`, `KDTRACE_HOOKS`, and `DDB_CTF`, and a VM snapshot at the `1.8-maintenance` state you can revert to.

That 驱动程序, those files, and those habits are what 第26章 extends. The additions introduced in this chapter live almost entirely in a new file, `myfirst_usb.c`, which becomes a second 内核模块 sharing the same conceptual family as `myfirst.ko` but building a separate `myfirst_usb.ko`. The chapter's labs exercise each stage of the new module: 探测, 附加, transfer setup, 回调 handling, /dev exposure, and 分离. The chapter does not modify `myfirst.ko` itself; the existing 驱动程序 remains a reference implementation of Parts 2 through 5, and the new 驱动程序 is its USB-transport sibling.

### What You Will Learn

By the end of this chapter you will be able to:

- Explain what makes a transport-specific 驱动程序 different from the pseudo-驱动程序 built in Parts 2 through 5, and name the three broad categories of work a transport-specific 驱动程序 has to add to its New总线 foundation: matching rules, transfer mechanics, and lifecycle 热插拔 handling.
- Describe the USB mental model at the level needed to write a 驱动程序: host versus 设备 roles, hubs and ports, 设备 classes (CDC, HID, Mass Storage, Vendor), the 描述符 hierarchy (设备, configuration, 接口, 端点), the four transfer types (control, bulk, interrupt, isochronous), and the 热插拔 lifecycle.
- Read the output of `usbconfig` and `dmesg` for a USB 设备 and identify its vendor identifier, product identifier, 接口 class, 端点 addresses, 端点 types, and 数据包 sizes.
- Describe the serial mental model at the level needed to write a 驱动程序: the UART as a shift 注册 with a baud generator, RS-232 framing, start and 停止位, 奇偶校验, hardware 流控制 via RTS and CTS, software 流控制 via XON and XOFF, and the relationship between `struct termios`, `tty(9)`, and the 驱动程序's `param` 回调.
- Explain the difference between FreeBSD's `uart(4)` subsystem for real UART hardware and the `ucom(4)` 框架 for USB-to-serial bridges, and name the two worlds a serial-驱动程序 author must never confuse.
- Write a USB 设备驱动程序 that 附加 to `uhub`, declares a `STRUCT_USB_HOST_ID` 匹配表, implements `探测` and `附加` and `分离` methods, uses `usbd_transfer_setup` to configure a bulk-in and a bulk-out transfer, and unwinds cleanly through a labelled-goto chain.
- Write a USB 传输回调 that follows the `USB_GET_STATE` state machine, handles `USB_ST_SETUP` and `USB_ST_TRANSFERRED` correctly, distinguishes `USB_ERR_CANCELLED` from other errors, and responds to a stalled 端点 with `usbd_xfer_set_stall`.
- Expose a USB 设备 to user space through the `usb_fifo` 框架 or through a custom `make_dev_s`-注册ed `cdevsw`, and know when each is the right choice.
- Read an existing UART 驱动程序 in `/usr/src/sys/dev/uart/` with a pattern vocabulary that makes the code's intent clear on first pass, including the `uart_class`/`uart_ops` split, the method dispatch, the baud-divisor calculation, and the tty-side wakeup machinery.
- Translate a `struct termios` into the four arguments of a UART `param` method (baud, databits, stopbits, 奇偶校验), and know which termios flags belong to the hardware layer and which belong to the line discipline.
- Test a USB驱动程序 against a simulated 设备 using QEMU USB redirection or `usb_template(4)`, and test a 串行驱动程序 against a `nmdm(4)` null-modem pair without any hardware at all.
- Use `cu(1)`, `tip(1)`, `stty(1)`, `comcontrol(8)`, and `usbconfig(8)` to drive, configure, and inspect serial and USB 设备 from user space.
- Handle 热拔出 cleanly in a USB驱动程序's 分离 path: cancel outstanding transfers, drain callouts and taskqueues, release 总线 resources, and destroy cdev nodes, all while knowing that the 设备 may already be gone by the time the 分离 method runs.

The list is long because transport-specific 驱动程序 touch many surfaces at once. Each item is narrow and teachable. The chapter's work is making the set of items into a coherent, reusable mental picture.

### What This Chapter Does Not Cover

Several adjacent topics are explicitly deferred to later chapters so 第26章 stays focused on the foundations of USB and 串行驱动程序 development.

- **USB 等时传输 and high-bandwidth video/audio streaming** are mentioned at a conceptual level in Section 1 but not developed. Isochronous transfers are the most complex of the four transfer types and are almost always used through higher-level 框架 (audio, video capture) that deserve their own treatment. 第26章 focuses on control, bulk, and 中断传输, which together cover the vast majority of USB驱动程序 work.
- **USB 设备-mode and gadget programming** through `usb_template(4)` is introduced briefly for testing purposes but not built out. Writing a custom USB gadget is a specialised project outside the scope of a first transport-specific chapter.
- **The internals of the USB 主机控制器 驱动程序** (`xhci`, `ehci`, `ohci`, `uhci`) are outside scope. These 驱动程序 implement the low-level protocol machinery that `usbd_transfer_setup` eventually calls; a 驱动程序 author almost never has to modify them. The chapter treats them as a stable platform.
- **Writing a new UART hardware 驱动程序 from scratch** is outside scope. The existing `ns8250` 驱动程序 handles every common PC 串行端口, the `pl011` 驱动程序 handles most ARM platforms, and the embedded SoC 驱动程序 handle the rest. Writing a new UART 驱动程序 is the specialised work of porting FreeBSD to a new system-on-chip, which is its own topic (touched on in Part 7 alongside Device Tree and ACPI). 第26章 teaches the reader how to *read* and *understand* a UART 驱动程序 rather than how to write one.
- **Storage 驱动程序** (GEOM providers, 块设备, VFS integration) are the subject of 第27章. USB mass storage is touched on only as an example of a USB 设备 class, not as a 驱动程序 target.
- **Network 驱动程序** (`ifnet(9)`, mbufs, RX/TX ring bookkeeping) are the subject of 第28章. USB network adapters are mentioned as an example of CDC 以太网, not as a 驱动程序 target.
- **USB/IP for remote USB 设备 testing over a network** is mentioned as an option for readers who truly cannot obtain any USB pass-through, but is not developed. The standard testing pathway in this chapter is a local VM with 设备 redirection.
- **Quirks and vendor-specific workarounds** through `usb_quirk(4)` are mentioned but not developed. A 驱动程序 author who needs quirks is already past the level this chapter teaches.
- **Bluetooth, Wi-Fi, and other wireless transports that happen to use USB** as their physical 总线 are outside scope. Those stacks involve protocols well beyond USB itself and are their own bodies of work.
- **Transport-agnostic abstraction for multi-总线 驱动程序** (the same 驱动程序 logic plugging into PCI, USB, and serial via a common 接口) is deferred to Part 7's portability chapter.

Staying inside those lines keeps 第26章 a chapter about *the USB and serial transports*, not a chapter about every technique a senior transport-specific 内核 developer might use on a senior transport-specific 内核 problem.

### Where We Are in 第6部分

第6部分 has three chapters. 第26章 is the opening chapter and teaches transport-specific 驱动程序 development through USB and serial 设备. 第27章 teaches transport-specific 驱动程序 development through storage 设备 and the VFS layer. 第28章 teaches it through 网络接口. The three chapters are structurally parallel in the sense that each introduces a transport, maps the subsystem, builds a minimal 驱动程序, and teaches hardware-free testing.

第26章 is the right place to start 第6部分 for three reasons. The first is that USB is the most gentle 引言 to hardware-backed 驱动程序: its core abstractions are smaller than storage's (no GEOM graph, no VFS), smaller than networking's (no mbuf chains, no 环形缓冲区 with head/tail pointers and interrupts mitigation), and the subsystem handles a large share of the hard parts on the 驱动程序's behalf. The second is that USB appears everywhere. Even a reader who will never write a storage or 网络驱动程序 will probably write a USB驱动程序 at some point: a thermometer, a data logger, a custom serial adapter, a factory test fixture. The third is pedagogical. The pattern USB teaches, a subsystem with 探测-and-附加 lifecycles, transfer setup through a config array, 回调-based completion, and a clean 分离 on unplug, is the same pattern (with different specifics) that storage and networking teach. Seeing it first in USB makes the next two chapters recognisable.

第26章 bridges forward to 第27章 by closing on a note about lifecycle: the USB 分离 path is a dress rehearsal for the storage-设备 hot-removal path, and the patterns the reader has just practised will come back the moment an external USB disk is pulled in 第27章. It also bridges backward to 第25章 by carrying every 第25章 discipline forward: `MODULE_DEPEND`, `MODULE_PNP_INFO`, the labelled-goto pattern, the errno vocabulary, rate-limited logging, version discipline, and a regression script that exercises the new module as rigorously as 第25章 exercised the old one.

### A Small Note on Difficulty

If the transition from pseudo-驱动程序 to real hardware looks daunting on the first reading, that feeling is entirely normal. Every experienced FreeBSD developer had a first USB驱动程序 that did not 附加, a first serial session where `cu` refused to talk, and a first debug session where `dmesg` stayed silent. The chapter is structured to ease you into each of those moments with labs, troubleshooting notes, and exit points. If a section starts to feel overwhelming, the right move is not to push through but to stop, read the corresponding real 驱动程序 in `/usr/src`, and return when the real code makes the concept visible. The existing FreeBSD 驱动程序 are the single best teaching resource this chapter can point to, and the chapter will point to them often.

## 读者指南: How to Use This Chapter

第26章 has three layers of engagement, and you can pick the layer that fits your current situation. The layers are independent enough that you can read for understanding now and return for hands-on practice later without losing continuity.

**Reading only.** Three to four hours. Reading gives you the USB and serial mental models, the shape of the FreeBSD subsystems, and pattern recognition for reading existing 驱动程序. If you are not yet in a position to load 内核模块 (because your lab VM is unavailable, you are reading on a commute, or you have a planning meeting in thirty minutes), a reading-only pass is a worthwhile investment. The chapter is written so that the prose carries the teaching load; the code 块 are there to anchor the prose, not to replace it.

**Reading plus the hands-on labs.** Eight to twelve hours over two or three sessions. The labs guide you through building `myfirst_usb.ko`, exploring real USB 设备 with `usbconfig`, setting up a simulated serial link with `nmdm(4)`, talking to it with `cu`, and running a 热拔出 stress test. The labs are where the chapter turns from explanation into reflex. If you can spare eight to twelve hours across two or three sessions, do the labs. The cost of skipping them is that the patterns stay abstract instead of becoming habit.

**Reading plus the labs plus the challenge exercises.** Fifteen to twenty hours over three or four sessions. The challenge exercises push beyond the chapter's worked example into the territory where you have to adapt the pattern to a new requirement: add a control-transfer ioctl, port the 驱动程序 to the `usb_fifo` 框架, read an unfamiliar USB驱动程序 end-to-end, simulate a flaky cable with failure injection, or extend the regression script to cover the new module. The challenge material does not introduce new foundations; it stretches the ones the chapter has just taught. Spend time on the challenges in proportion to how much autonomy you expect to have on your next 驱动程序 project.

Do not rush. This is the first chapter in the book whose material depends on real hardware or convincing simulation. Set aside a 块 of time when you can watch `dmesg` after `kldload` and read it slowly. A USB驱动程序 that 附加 without errors is usually right; a USB驱动程序 whose 附加 messages you have not actually read is often wrong in a way that will cost you an hour of debugging two days later. The small discipline of reading the 附加 output as it happens, rather than assuming it, is a habit worth forming in 第26章 because every subsequent transport-specific chapter depends on it.

### Recommended Pacing

Three sitting structures work well for this chapter.

- **Two long sittings of four to six hours each.** First sitting: 引言, 读者指南, How to Get the Most Out of This Chapter, Section 1, Section 2, and Lab 1. Second sitting: Section 3, Section 4, Section 5, Labs 2 through 5, and the 总结. The advantage of long sittings is that you stay in the mental model long enough to connect Section 1's vocabulary to Section 3's 回调 code.

- **Four medium sittings of two to three hours each.** Sitting 1: 引言 through Section 1 and Lab 1. Sitting 2: Section 2 and Lab 2. Sitting 3: Section 3 and Labs 3 and 4. Sitting 4: Section 4, Section 5, Lab 5, and 总结. The advantage is that each sitting has a crisp milestone.

- **A linear reading pass followed by a hands-on pass.** Day one: read the entire chapter start to finish without running any code, to get the full mental model in place. Day two or day three: return to the chapter with a 内核 source tree and a lab VM open and work through the labs in sequence. The advantage of this approach is that the mental model is fully loaded before you touch code, which catches concept-level mistakes early.

Do not attempt the whole chapter in a single marathon session. The material is dense, and the USB 回调 state machine in particular does not reward tired reading.

### What a Good Study Session Looks Like

A good study session for this chapter has five elements visible at once. Put the book chapter on one side of your screen. Put the relevant FreeBSD source files in a second pane: `/usr/src/sys/dev/usb/usbdi.h`, `/usr/src/sys/dev/usb/misc/uled.c`, and `/usr/src/sys/dev/uart/uart_总线.h` are the three most useful to keep open. Put a terminal on your lab VM in a third pane. Put `man 4 usb`, `man 4 uart`, and `man 4 ucom` in a fourth pane for quick reference. Finally, keep a small note file open for questions you will want to answer later. If a term comes up you cannot define, write it in the note file and keep reading; if the same term comes up twice, look it up before continuing. This is the study posture that gets the most out of a long technical chapter.

### If You Do Not Have a USB Device to Test With

Many readers will not have a spare USB 设备 that matches the vendor/product identifiers in the worked example. That is fine. Section 5 teaches three ways to proceed: QEMU USB 设备 redirection from a host to a guest, `usb_template(4)` for FreeBSD-as-USB-设备, and the simulated-设备 approach that tests 驱动程序 logic without a real 总线 at all. The chapter's worked example is written so that the 驱动程序's 匹配表 can be swapped for one matching any USB 设备 you do have on your desk. A USB flash drive will do. A mouse will do. A keyboard will do. The chapter explains how to point the 驱动程序 at whatever 设备 you happen to have, at the cost of temporarily stealing that 设备 from the 内核's built-in 驱动程序, which the chapter also covers.

## How to Get the Most Out of This Chapter

Five habits pay off in this chapter more than in any of the earlier chapters.

First, **keep four short manual-page files open in a browser tab or a terminal pane**: `usb(4)`, `usb_quirk(4)`, `uart(4)`, and `ucom(4)`. These four pages together are the tightest overview the FreeBSD project has of the subsystems this chapter introduces. None of them is long. `usb(4)` describes the subsystem from the user's perspective and lists the `/dev` entries that appear. `usb_quirk(4)` lists the quirks table and explains what a quirk is, which will save you puzzlement later when you see quirk code in real 驱动程序. `uart(4)` describes the serial subsystem from the user's perspective. `ucom(4)` describes the USB-to-serial 框架. Skim each once at the start of the chapter. When the prose refers to "consult the manual page for details," return to the appropriate page. The manual pages are authoritative; this book is commentary.

Second, **keep three real 驱动程序 close to hand**. `/usr/src/sys/dev/usb/misc/uled.c` is a very small USB驱动程序 that talks to a USB-附加ed LED. It uses the `usb_fifo` 框架, which is one of the two user-visible patterns the chapter teaches, and its entire 附加 function is smaller than a page. `/usr/src/sys/dev/usb/misc/ugold.c` is a slightly larger USB驱动程序 that reads temperature data from a TEMPer thermometer through 中断传输. It demonstrates the other common transfer type and shows how a 驱动程序 uses a callout to pace its reads. `/usr/src/sys/dev/uart/uart_dev_ns8250.c` is the canonical 16550 UART 驱动程序; every PC 串行端口 in the world uses it. Read each of these three files once at the start of the chapter and once more at the end. The first read will feel largely opaque; the second will feel almost obvious. That change is the measure of progress this chapter offers.

Third, **type every code addition by hand**. The `myfirst_usb.c` file grows through the chapter in roughly a dozen small increments. Each increment corresponds to a paragraph or two of prose. Typing the code by hand is what turns the prose into muscle memory. Pasting the code skips the lesson. If that sounds pedantic, notice that every working USB驱动程序 author has written a USB驱动程序's 附加 function at least a dozen times; typing this one is the first of that dozen.

Fourth, **read `dmesg` after every `kldload`**. A USB驱动程序 produces a predictable pattern of 附加 messages: the 设备 is detected on a port, the 驱动程序 探测, the match succeeds, the 驱动程序 附加, the `/dev` node appears. If any of those steps is missing, something is wrong, and the sooner you notice the missing step, the sooner you fix it. The smallest discipline this chapter can give you is the habit of running `dmesg | tail -30` immediately after `kldload` and reading every line. If the output is boring, the 驱动程序 probably works. If the output surprises you, investigate before proceeding.

Fifth, **after every section, ask yourself what would happen if you pulled the cable**. The question sounds silly; it is central. A well-written transport-specific 驱动程序 is always one that handles being removed while in use. A USB驱动程序 in particular runs in a world where 热拔出 is the normal operating condition. If you find yourself writing a section of code and cannot answer "what if the cable were pulled right here," the section is not yet finished. 第26章 returns to this question often, not as rhetoric but as a discipline.

### What to Do When Something Does Not Work

It will not all work the first time. USB驱动程序 have a few common failure modes, and the chapter documents each of them in the troubleshooting section at the end. A short preview of the most common ones:

- The 驱动程序 compiles but does not 附加 when the 设备 is plugged in. Usually the 匹配表 has the wrong vendor or product identifier. The fix is to verify the identifier with `usbconfig dump_设备_desc`.
- The 驱动程序 附加 but the `/dev` node does not appear. Usually the `usb_fifo_附加` call failed because the name conflicts with an existing 设备. The fix is to change the `basename` or to 分离 the conflicting 驱动程序 first.
- The 驱动程序 附加 but the first transfer never completes. Usually `usbd_transfer_start` was not called, or the transfer was submitted with a zero-length 帧. The fix is to trace through `USB_ST_SETUP` and confirm that `usbd_xfer_set_帧_len` was called before `usbd_transfer_submit`.
- The 驱动程序 附加 but the 内核 panics on unplug. Usually the 分离 path is missing a `usbd_transfer_unsetup` call or a `usb_fifo_分离` call. The fix is to run the 分离 sequence under INVARIANTS and follow the WITNESS output back to the first dropped cleanup.

The troubleshooting section at the end of the chapter develops each of these cases in full, with diagnostic commands and expected output. The goal of this chapter is not to have everything work on the first try; the goal is to have a systematic debugging posture that turns every failure into a teachable moment.

### Roadmap Through the Chapter

The sections in order are:

1. **Understanding USB and Serial Device Fundamentals.** The USB mental model at the level needed to write a 驱动程序: host and 设备, hubs and ports, classes, 描述符, transfer types, 热插拔 lifecycle. The serial mental model: UART hardware, RS-232 framing, 波特率, 奇偶校验, 流控制, the tty discipline. The FreeBSD-specific split between `uart(4)` and `ucom(4)`. A first exercise with `usbconfig` and `dmesg` that grounds the vocabulary in a 设备 you can see.

2. **Writing a USB Device Driver.** The FreeBSD USB subsystem layout. The New总线 shape of a USB驱动程序. `STRUCT_USB_HOST_ID` and the 匹配表. `DRIVER_MODULE` with `uhub` as the parent. `MODULE_DEPEND` on `usb`. `USB_PNP_HOST_INFO` for auto-load. The 探测 method using `usbd_lookup_id_by_uaa`. The 附加 method, the softc layout, the labelled-goto cleanup chain in 附加 and 分离.

3. **Performing USB Data Transfers.** The `struct usb_config` array. `usbd_transfer_setup` and the lifetime of a `struct usb_xfer`. Control, bulk, and 中断传输 shapes. The `usb_回调_t` state machine and `USB_GET_STATE`. Stall handling with `usbd_xfer_set_stall`. Frame-level operations (`usbd_xfer_set_帧_len`, `usbd_copy_in`, `usbd_copy_out`). Creating a `/dev` entry for the USB 设备 through the `usb_fifo` 框架. A worked example that sends bytes down a bulk-out 端点 and reads bytes back from a bulk-in 端点.

4. **Writing a Serial (UART) Driver.** The `uart(4)` subsystem at the level needed to read a real 驱动程序. The `uart_class`/`uart_ops` split. The method table dispatched through kobj. The relationship between `uart_总线_附加` and `uart_tty_附加`. Baud rate, 数据位, 停止位, 奇偶校验, and the `param` method. RTS/CTS hardware 流控制. `struct termios` and how it reaches the 驱动程序. `/dev/ttyu*` versus `/dev/cuau*` in FreeBSD. The `ucom(4)` 框架 for USB-to-serial bridges. A guided reading of the `ns8250` 驱动程序 as a canonical example.

5. **Testing USB和串行驱动程序 Without Real Hardware.** `nmdm(4)` virtual null-modem pairs for serial testing. `cu(1)` and `tip(1)` for terminal access. `stty(1)` and `comcontrol(8)` for configuration. QEMU USB 设备 redirection for host-to-guest pass-through. `usb_template(4)` for FreeBSD-as-gadget testing. Software loopback patterns that exercise 驱动程序 logic without any 设备 at all. A reproducible test harness that runs a regression without human intervention.

After the five sections come a set of hands-on labs, a set of challenge exercises, a troubleshooting reference, a 总结 that closes 第26章's story, a bridge to 第27章, and a glossary. Read linearly on a first pass.

## 第1节： Understanding USB and Serial Device Fundamentals

The first section teaches the mental models the rest of the chapter relies on. USB and serial 设备 share a surprising 挂载 of machinery at the `tty`/`cdevsw` layer, and at the same time they differ dramatically at the transport layer. A reader who is clear on both the similarities and the differences will find Sections 2 through 5 straightforward. A reader who is not will find the subsequent code confusingly non-obvious. This section is the single best place to spend an extra thirty minutes if you want the rest of the chapter to feel easier.

The section is organised in three arcs. The first arc establishes what a *transport* is, and why transport-specific 驱动程序 look different from the pseudo-驱动程序 of Parts 2 through 5. The second arc teaches the USB model: host and 设备, hubs and ports, classes, 描述符, 端点, transfer types, and the 热插拔 lifecycle. The third arc teaches the serial model: the UART, RS-232 framing, 波特率, 奇偶校验, 流控制, and the FreeBSD-specific split between `uart(4)` and `ucom(4)`. A first exercise at the end grounds the vocabulary in a 设备 you can see with `usbconfig`.

### What a Transport Is, and Why It Matters Here

A *transport* is the protocol and the lifecycle by which a 设备 is connected to the rest of the system. Up to this point in the book, the `myfirst` 驱动程序 has had no transport. Its 设备 existed entirely in the New总线 tree, connected to the `nexus` through the `pseudo` parent, and its data flowed into a 缓冲区 in 内核 memory. That makes `myfirst` a *pseudo-设备*: a 设备 whose existence is entirely a software fiction. Pseudo-设备 are essential teaching tools. They let the reader learn New总线, softc management, 字符设备 接口, ioctl handling, locking, and the rest, without also learning the specifics of a 总线. By now, those topics are covered.

A transport-specific 驱动程序, by contrast, is one that 附加 to a *real* 总线. The 总线 has its own rules. It has its own way of saying "a new 设备 has appeared." It has its own way of delivering data. It has its own way of saying "a 设备 has been removed." A transport-specific 驱动程序 is still a New总线 驱动程序 (that never changes in FreeBSD), but its parent is no longer the abstract `pseudo` 总线. Its parent is `uhub` if it is a USB驱动程序, `pci` if it is a PCI 驱动程序, `acpi` or `fdt` if it is on an embedded platform, and so on. The 驱动程序's 附加 method receives arguments specific to that 总线. Its cleanup responsibilities include 总线-specific resources in addition to the ones it already had. Its lifecycle is the 总线's lifecycle, not the module's lifecycle.

Three broad categories of work distinguish a transport-specific 驱动程序 from the pseudo-驱动程序 of Parts 2 through 5. They are worth naming explicitly because they recur in every transport chapter in 第6部分.

The first is *matching*. A pseudo-驱动程序 附加 on module load; there is nothing to match because there is no real 设备. A transport-specific 驱动程序 has to declare which 设备 it handles. On USB, this means a 匹配表 of vendor and product identifiers. On PCI, it means a 匹配表 of vendor and 设备 identifiers. On ACPI or FDT, it means a 匹配表 of string identifiers. The 内核's 总线 code enumerates 设备 as they appear and offers each one to every 注册ed 驱动程序 in turn; the 驱动程序's 探测 method decides whether to claim the 设备. Getting the 匹配表 right is the first obstacle every transport-specific 驱动程序 faces.

The second is *transfer mechanics*. A pseudo-驱动程序's `read` and `write` methods touch a 缓冲区 in RAM. A transport-specific 驱动程序's `read` and `write` methods have to arrange for data to move across the 总线. On USB, this means setting up one or more transfers using `usbd_transfer_setup`, submitting them with `usbd_transfer_submit`, and handling completion in a 回调. On PCI, this means programming a DMA engine. On storage, this means translating 块 requests into 总线 transactions. The transfer mechanism is 总线-specific and is where most of a transport-specific 驱动程序's new code lives.

The third is *热插拔 lifecycle*. A pseudo-驱动程序 is loaded when the module is loaded and 分离ed when the module is unloaded. That is a simple lifecycle; `kldload` and `kldunload` are the only events it has to respond to. A transport-specific 驱动程序 has to deal with *热插拔*: the 设备 can appear and disappear independently of the module's lifecycle. A USB 设备 can be unplugged in the middle of a read. A SATA disk can be yanked out while the 文件系统 on it is 挂载ed. An 以太网 cable can be pulled while a TCP connection is open. The 驱动程序's 附加 method runs when a 设备 is physically inserted; its 分离 method runs when a 设备 is physically removed. The 分离 may happen while the 驱动程序 is still in use. Handling this correctly is the third big obstacle every transport-specific 驱动程序 faces.

The rest of 第6部分 is about those three categories of work in three different transports. 第26章 teaches USB and serial. 第27章 teaches storage. 第28章 teaches networking. The matching, the transfer mechanics, and the 热插拔 lifecycle look different in each transport, but the three-category structure repeats. That structure is what makes it possible to learn one transport well and then learn the next one quickly.

A useful shorthand: in Parts 2 through 5, you learned how to *be* a New总线 驱动程序. In 第6部分, you learn how to *附加* to a 总线 that has its own ideas about when and how you exist.

### The USB Mental Model

USB, the Universal Serial Bus, is a tree-structured, host-controlled, 热插拔gable serial 总线. Every one of those adjectives matters, and understanding each of them is the foundation of writing a USB驱动程序.

*Tree-structured* means that USB 设备 do not sit on a shared wire like 设备 on an I2C 总线 or an old ISA 总线. Every USB 设备 has exactly one upstream connection, to a parent hub. The root of the tree is the *root hub*, which is exposed by the USB 主机控制器. Downstream of the root hub are other hubs and 设备. A hub has a fixed number of downstream ports; each port can either be empty or connect to exactly one 设备. The tree is rebuilt on boot and updated every time a 设备 is connected or disconnected. On FreeBSD, `usbconfig` shows this tree; on a fresh boot of a typical desktop you will see something like:

```text
ugen0.1: <Intel EHCI root HUB> at usbus0, cfg=0 md=HOST spd=HIGH
ugen1.1: <AMD OHCI root HUB> at usbus1, cfg=0 md=HOST spd=FULL
ugen0.2: <Some Vendor Hub> at usbus0, cfg=0 md=HOST spd=HIGH
ugen0.3: <Some Vendor Mouse> at usbus0, cfg=0 md=HOST spd=LOW
```

The tree structure matters to a 驱动程序 author for two reasons. First, it tells you that when you write a USB驱动程序, your 驱动程序's *parent* in the New总线 tree is `uhub`. Every USB 设备 sits under a hub. When you write `DRIVER_MODULE(myfirst_usb, uhub, ...)`, you are telling the 内核 "my 驱动程序 附加 to children of `uhub`," which is the FreeBSD way of saying "my 驱动程序 附加 to USB 设备." Second, the tree structure means that enumeration is dynamic. The 内核 does not know what 设备 are on the tree until the tree is walked; a 驱动程序 is offered each 设备 as it appears, and has to decide whether to claim it.

*Host-controlled* means that one side of the 总线 is the master, the *host*, and all other sides are slaves, the *设备*. The host initiates every transfer; 设备 respond. A USB keyboard does not push keystrokes to the host whenever a key is pressed; the host polls the keyboard on an *interrupt 端点* many times per second, and the keyboard replies with "no new keys" or "key 'A' has been pressed" in response to each poll. This polling-and-response model has important consequences for a 驱动程序. Your 驱动程序, running on the host, has to initiate every transfer. A 设备 cannot spontaneously send data; it can only respond when the host asks. What looks from user space like "the 驱动程序 received data" is always, underneath, "the 驱动程序 had a pending receive transfer and the 主机控制器 notified us that the transfer completed."

For most of this chapter's purposes, you are writing *host-mode* 驱动程序: 驱动程序 that run on the host side. A FreeBSD system can also be configured as a USB *设备*, through the `usb_template(4)` subsystem, and present itself as a keyboard or mass storage 设备 or CDC 以太网 接口 to another host. Device-mode 驱动程序 are a specialised topic touched on only briefly in Section 5 for testing purposes.

*Hot-pluggable* means that 设备 can appear and disappear while the system is running, and the subsystem has to cope. The USB 主机控制器 notices when a 设备 is plugged in, a hub's port status 寄存器 tell it so, enumerates the new 设备 by asking it for its 描述符, assigns it an address on the 总线, and then offers it to any 驱动程序 whose 匹配表 applies. When a 设备 is unplugged, the 主机控制器 notices the port status change and tells the subsystem, which in turn calls the 驱动程序's 分离 method. The 分离 method may run at any time, including while the 驱动程序 is holding a transfer that will now never complete, while user space has the 驱动程序's `/dev` node open, or while the system is under load. Writing a correct 分离 method is the single hardest part of USB驱动程序 development. The chapter returns to this repeatedly.

*Serial* means that USB is a wire-level serial protocol: bytes flow one after another on a differential pair. The speed of the 总线 has evolved over the years: low-speed (1.5 Mbps), full-speed (12 Mbps), high-speed (480 Mbps), SuperSpeed (5 Gbps), and faster variants above that. From a 驱动程序 author's perspective, the speed is mostly transparent: the 主机控制器 and the USB core handle the electrical layer and the 数据包 framing, and your 驱动程序 works at the level of "here is a 缓冲区, please send it" or "here is a 缓冲区, please fill it." The speed determines how fast data can move, but the 驱动程序 code is the same.

With those four adjectives in place, the rest of the USB model falls into shape.

#### Device Classes and What They Mean to a Driver

Every USB 设备 belongs to one or more *classes*, and the class tells the host (and the 驱动程序) what kind of 设备 it is. Classes are numerical, defined by the USB Implementers Forum, and the values appear in 描述符. The ones a FreeBSD 驱动程序 author will see most often include:

- **HID (Human Interface Device)**, class 0x03. Keyboards, mice, joysticks, game controllers, and a long tail of programmable 设备 that pretend to be keyboards or mice. HID 设备 present reports through interrupt 端点; FreeBSD's HID subsystem handles them mostly generically, though a vendor-specific 驱动程序 can override.
- **Mass Storage**, class 0x08. USB flash drives, external disks, card readers. These 附加 through `umass(4)` to the CAM storage 框架.
- **Communications (CDC)**, class 0x02, with subclasses for ACM (modem-like serial), ECM (以太网), NCM (以太网 with multi-数据包 aggregation), and others. CDC ACM 设备 appear through `ucom(4)` as 串行端口s. CDC ECM and NCM 设备 appear through `cdce(4)` as 网络接口.
- **Audio**, class 0x01. Microphones, speakers, audio 接口. FreeBSD's audio stack handles these through `uaudio(4)`.
- **Printer**, class 0x07. USB printers. Handled through `ulpt(4)`.
- **Hub**, class 0x09. USB hubs themselves. Handled by the core `uhub(4)` 驱动程序.
- **Vendor-specific**, class 0xff. Any 设备 whose functionality does not fit a standard class. Almost every interesting hobby USB 设备 (USB-to-serial bridges, thermometers, relay controllers, programmers, loggers) is in this class.

When you write a USB驱动程序, you often write for a vendor-specific 设备 (class 0xff) and match on vendor/product identifiers. Occasionally you write for a standard-class 设备 that FreeBSD does not yet handle, or for a standard-class 设备 that has quirks requiring a dedicated 驱动程序. The class is not usually the match criterion; the vendor/product pair is. But the class tells you what 框架, if any, you should integrate with. If the 设备's class is CDC ACM, the right 框架 is `ucom`. If the class is HID, the right 框架 is `hid总线` (new in FreeBSD 14). If the class is 0xff, there is no 框架; you write a bespoke 驱动程序.

#### Descriptors: The Device's Self-Description

When the host enumerates a new USB 设备, it asks the 设备 to describe itself. The 设备 responds with a hierarchy of *描述符*. Descriptors are the single most important USB concept to get clear: they are the USB equivalent of the PCI configuration space, but richer and nested.

The hierarchy is:

```text
Device descriptor
  Configuration descriptor [1..N]
    Interface descriptor [1..M] (optionally with alternate settings)
      Endpoint descriptor [0..E]
```

A *设备 描述符* (`struct usb_设备_描述符` in `/usr/src/sys/dev/usb/usb.h`) describes the 设备 as a whole: its vendor identifier (`idVendor`), its product identifier (`idProduct`), its 设备 class, subclass, and protocol, its maximum 数据包 size on 端点 zero, its release number, and the number of configurations it supports. Most 设备 have one configuration, but the USB spec allows more (a camera that can run in high-bandwidth or low-bandwidth modes, for example).

A *configuration 描述符* (`struct usb_config_描述符`) describes one mode of operation: the number of 接口 it contains, whether the 设备 is self-powered or 总线-powered, its maximum power draw. When a 驱动程序 selects a configuration (by calling `usbd_req_set_config`, though in practice the USB core does this for you), the 设备's 端点 are activated.

An *接口 描述符* (`struct usb_接口_描述符`) describes one logical function of the 设备. A composite 设备, such as a USB printer with a built-in scanner, has one 接口 per function. Each 接口 has its own class, subclass, and protocol. A 驱动程序 can match on 接口 class rather than 设备 class; this is common when a 设备's overall class is "Miscellaneous" or "Composite" but one of its 接口 has a specific class. An 接口 can have multiple *alternate settings*, which select different 端点 layouts; audio streaming 接口 use alternate settings to offer different bandwidths.

An *端点 描述符* (`struct usb_端点_描述符`) describes one data channel. Endpoints have:

- An *address*, which is the 端点 number (0 through 15) combined with a direction bit (IN, meaning from 设备 to host, or OUT, meaning from host to 设备).
- A *type*, which is one of control, bulk, interrupt, or isochronous.
- A *maximum 数据包 size*, which is the largest single 数据包 the 端点 can handle.
- An *interval*, which for interrupt and isochronous 端点 tells the host how often to poll.

Endpoint zero is special: every 设备 has it, it is always a control 端点, and it is always bidirectional (one IN half and one OUT half). The USB core uses 端点 zero for enumeration (asking the 设备 for 描述符, setting its address, selecting its configuration). A 驱动程序 can also use 端点 zero for vendor-specific control requests, though 驱动程序 usually access it through helper functions rather than setting up a transfer directly.

The 描述符 hierarchy matters to a 驱动程序 because the 驱动程序's `探测` method has access to the 描述符 through the `struct usb_附加_arg` it receives, and its match logic often reads fields from them. The `struct usbd_lookup_info` inside `struct usb_附加_arg` carries the 设备's identifiers, its class, subclass, and protocol, the current 接口's class, subclass, and protocol, and a few other fields. The 匹配表 filters on some subset of those; the helper macros `USB_VP(v, p)`, `USB_VPI(v, p, info)`, `USB_IFACE_CLASS(c)`, and similar build entries that match different combinations of fields.

#### The Four Transfer Types

USB defines four transfer types, each suited to a different kind of data movement. A 驱动程序 picks one or more types for its 端点, and the choice affects everything about how the 驱动程序 is structured.

*Control transfers* are for setup, configuration, and command exchange. Every 设备 supports them on 端点 zero. They have a small, structured format: an eight-byte setup 数据包 (the `struct usb_设备_request`) followed by an optional data stage and a status stage. The setup 数据包 specifies what the request is doing: its direction (IN or OUT), its type (standard, class, or vendor), its recipient (设备, 接口, or 端点), and four fields (`bRequest`, `wValue`, `wIndex`, `wLength`) whose meaning depends on the request. Standard requests include `GET_DESCRIPTOR`, `SET_CONFIGURATION`, and so on; class and vendor requests are defined by the class specification or the vendor. Control transfers are reliable: the 总线 protocol guarantees delivery or returns a specific error. They are also relatively slow, because the 总线 allocates only a small share of its bandwidth to them.

*Bulk transfers* are for large, reliable, non-time-critical data. A USB flash drive uses 批量传输 for the actual data. A printer uses bulk OUT for the print stream. A USB-to-serial bridge uses bulk IN and bulk OUT for the two directions of the serial stream. Bulk transfers are reliable (errors are retried by the 总线 hardware), but they have no guaranteed timing: they use whatever bandwidth is left after control, interrupt, and isochronous traffic has been scheduled. In practice, on a lightly-loaded 总线, 批量传输 are very fast. On a heavily-loaded 总线, they can stall for milliseconds at a time. Bulk 端点 are the most common 端点 type for 设备-to-host or host-to-设备 streaming of data where latency is not critical.

*Interrupt transfers* are for small, time-sensitive data. The name is misleading: there are no hardware interrupts here. The "interrupt" refers to the fact that the 设备 needs to get the host's attention periodically, and the host polls the 端点 at a configurable interval to see whether there is new data. A USB keyboard uses 中断传输 to deliver keystrokes; a USB mouse uses them for movement reports; a thermometer uses them to deliver periodic readings. Interrupt 端点 have an `interval` field that tells the host how often to poll (in milliseconds for low- and full-speed 设备, in micro帧 for high-speed). A 驱动程序 that wants to know about input as it happens sets up an interrupt-IN transfer, submits it, and the USB core arranges the polling. When data arrives, the 驱动程序's 回调 fires.

*Isochronous transfers* are for streaming data with guaranteed bandwidth but no error recovery. USB audio and USB video use isochronous 端点. The 总线 reserves a fixed share of each 帧 for isochronous traffic, so the bandwidth is predictable, but transfers are not retried on error; if a 数据包 is corrupted, it is lost. This trade-off makes sense for audio and video, where a dropped sample is better than a stall. Isochronous transfers are the most complex to program because they typically operate on many small 帧 per transfer; the `struct usb_xfer` machinery supports up to thousands of 帧 per transfer. 第26章 introduces 等时传输 at the conceptual level and does not develop them further; real isochronous 驱动程序 (audio, video) are beyond the chapter's scope.

A typical vendor-specific USB 设备 that a hobbyist or a 驱动程序-development learner will write code for looks like this: a vendor/product identifier, one vendor-specific 接口, one bulk-IN 端点, one bulk-OUT 端点, and possibly an interrupt-IN 端点 for status events. That is the shape of the worked example in Sections 2 and 3.

#### The USB Hot-Plug Lifecycle

The 热插拔 lifecycle is the sequence of events that happens when a USB 设备 is inserted, in use, and removed. Writing a 驱动程序 that handles this lifecycle correctly is the most important single discipline in USB驱动程序 development.

When a 设备 is inserted, the 主机控制器 notices a port status change. It waits for the 设备 to stabilise, then resets the port and assigns the 设备 a temporary address of zero. It sends `GET_DESCRIPTOR` to 端点 zero on address zero, retrieves the 设备 描述符, and then assigns the 设备 a unique address with `SET_ADDRESS`. All subsequent communication uses the new address. The host sends `GET_DESCRIPTOR` for the full configuration 描述符 (including 接口 and 端点), chooses a configuration, and sends `SET_CONFIGURATION`. At that point the 设备's 端点 are active and the USB subsystem offers the 设备 to every 注册ed 驱动程序 in turn by calling each 驱动程序's `探测` method. The first 驱动程序 to claim the 设备 by returning a non-error code from `探测` wins; the subsystem then calls that 驱动程序's `附加` method.

During normal operation, the 驱动程序 submits transfers to its 端点, the 主机控制器 schedules them on the 总线, and the 回调 fire on completion. This is the steady state 第26章's code examples operate in.

When the 设备 is removed, the 主机控制器 notices another port status change. It does not wait; the electrical signal is gone immediately. The subsystem cancels any outstanding transfers on the 设备's 端点 (they complete in the 回调 with `USB_ERR_CANCELLED`), and then it calls the 驱动程序's `分离` method. The `分离` method has to release every resource the `附加` method acquired, including any `/dev` nodes it created, any locks, any 缓冲区, and any transfers. It has to do this in the face of the fact that other threads may be in the middle of calling into the 驱动程序 through those resources. A read in progress must be woken up and returned with an error. An ioctl in progress must be allowed to finish or interrupted. A 回调 that has just fired with `USB_ERR_CANCELLED` must not try to re-submit.

The 热插拔 lifecycle is why USB驱动程序 cannot be written the way pseudo-驱动程序 are written. In a pseudo-驱动程序, the module lifecycle (`kldload`/`kldunload`) is the only lifecycle; nothing unexpected happens. In a USB驱动程序, the 设备 lifecycle is separate from the module lifecycle and is driven by physical events. A user can unplug the 设备 while a user-space process is 块ed in `read()` on the 驱动程序's `/dev` node. The 驱动程序 must wake that process up and return an error. A well-written USB驱动程序 treats this as the normal case, not the edge case.

Section 2 will walk through the structure of a USB驱动程序 that handles this correctly. For now, keep the lifecycle in mind: 探测, 附加, steady-state transfers, 分离. Every USB驱动程序 has that sequence.

#### USB Speeds and What They Imply

USB has gone through several speed generations, and each matters to a 驱动程序 writer in different ways. Low-speed (1.5 Mbps) was the original USB 1.0 speed, mostly used by keyboards and mice. Full-speed (12 Mbps) was USB 1.1, used by printers, early cameras, and mass-storage 设备. High-speed (480 Mbps) was USB 2.0, which became the dominant speed for most 设备 in the 2000s. SuperSpeed (5 Gbps) was USB 3.0, which added a separate physical layer for high-throughput applications. SuperSpeed+ (10 Gbps and 20 Gbps) came with USB 3.1 and 3.2. USB 4.0 reuses the Thunderbolt physical layer and supports 40 Gbps.

For most 驱动程序 writing, only three differences between these speeds matter:

**Maximum 数据包 size.** Low-speed 端点 have a maximum 数据包 size of 8 bytes. Full-speed goes up to 64 bytes. High-speed bulk 端点 go up to 512 bytes. SuperSpeed bulk 端点 go up to 1024 bytes with burst support. Your 缓冲区 sizes in the transfer configuration should match the 端点's speed; using a 512-byte 缓冲区 on a full-speed bulk 端点 wastes memory because only 64 bytes fit in each 数据包.

**Isochronous bandwidth.** Isochronous transfers reserve bandwidth at a specific speed. A 设备 that asks for 1 MB/s of isochronous bandwidth can only be supported on a 主机控制器 that can provide it; on slower hosts, the 设备 must negotiate a lower rate or fail. This is why some USB audio 设备 work on one port but not another.

**Endpoint polling interval.** Interrupt 端点 are polled at a specific interval encoded in the 描述符's `bInterval` field. The units are milliseconds at low/full speed and "125 microsecond micro帧" at high/SuperSpeed. The 框架 handles the math; your 驱动程序 just declares the logical polling interval via the 端点 描述符 and the 框架 does the right thing.

For the 驱动程序 we write in this chapter (`myfirst_usb` and the UART bridges like FTDI), speed does not affect the code structure. A bulk-IN channel's 回调 is the same whether it runs at 12 Mbps or 5 Gbps. The differences are in the numbers, not the flow.

#### Endpoints, FIFOs, and Flow Control

A USB 端点 is logically an I/O queue at one end of a pipe. On the 设备 side, an 端点 corresponds to a hardware FIFO in the chip. On the host side, the 端点 is a 框架 abstraction. Between them, USB 数据包 flow under the control of the USB protocol itself, which handles retransmission, sequencing, and error detection.

The host cannot be told "the 设备 is full" the way you might expect on a traditional serial link. Instead, when a 设备 cannot accept more data (because its FIFO is full), it returns a NAK handshake. NAK means "try again later." The host will keep retrying, at the protocol level, until either the 设备 accepts the data (returns ACK) or some higher-level timeout fires. This is called NAK-limiting or 总线 throttling, and it happens invisibly to the 驱动程序: the 框架 sees the final ACK and delivers a successful completion.

Similarly, when the 设备 has no data to send (for a bulk-IN or interrupt-IN transfer), it returns NAK to the IN token, and the host polls again. From the 驱动程序's perspective, the transfer is simply "pending" until the 设备 has something to say.

This NAK mechanism is how USB handles 流控制 at the protocol level. Your 驱动程序 does not need to implement its own throttling logic for bulk and interrupt channels; the USB protocol does it. Where 流控制 does come into play is in higher-level protocols, where the 设备 might want to signal a logical end-of-message or a temporary unavailability. Those signals are protocol-specific and not part of USB itself.

#### Descriptors In Depth

USB 描述符 are the self-describing mechanism by which a 设备 tells the host what it is and how to talk to it. We introduced them briefly earlier; here is a more complete picture.

The 设备 描述符 is the root. It contains the 设备's 供应商ID, 产品ID, USB specification version, 设备 class/subclass/protocol (for 设备 that declare themselves at the 设备 level rather than the 接口 level), maximum 数据包 size for 端点 zero, and the number of configurations.

Configuration 描述符 describe complete configurations. A configuration is a set of 接口 that work together. Most 设备 have one configuration; some have multiple to support different modes of operation (e.g., a 设备 that can be either a printer or a scanner, selected by configuration).

Interface 描述符 describe functional subsets of the 设备. Each 接口 has a class, subclass, and protocol that tells the host what kind of 驱动程序 to use. A multi-function 设备 has multiple 接口 描述符 in the same configuration. Additionally, an 接口 can have alternate settings: different sets of 端点 selectable on the fly for things like "low bandwidth mode" vs "high bandwidth mode".

Endpoint 描述符 describe individual 端点 within an 接口. Each has an address (with direction bit), a transfer type, a maximum 数据包 size, and an interval (for interrupt and isochronous 端点).

String 描述符 hold human-readable strings: the manufacturer name, the product name, the serial number. These are optional; their presence is indicated by nonzero string indices in the other 描述符.

Class-specific 描述符 extend the standard 描述符 with class-specific metadata. HID 设备 have a report 描述符 that describes the format of the reports they send. Audio 设备 have 描述符 for audio controls. Mass-storage 设备 have 描述符 for 接口 subclasses.

The USB 框架 parses all of this at enumeration time and exposes the parsed data to 驱动程序 through the `struct usb_附加_arg`. Your 驱动程序 does not have to read 描述符 itself; it queries the 框架 for the information it needs. When the chapter says "the 接口's `bInterfaceClass`", what is meant is "the `bInterfaceClass` field of the 接口 描述符 the 框架 parsed and cached for us."

`usbconfig -d ugenN.M dump_all_config_desc` is how you see the parsed 描述符 from userland. Run that command on a few 设备 you own and note how the 描述符 look. You will see that even simple 设备 like a mouse have a nontrivial 描述符 tree: typically one 设备 描述符, one configuration 描述符, one 接口 描述符 (with class=HID), and one or two 端点 描述符 (for the HID report input and maybe an output).

#### Request-Response Over USB

The USB 控制传输 type supports a request-response pattern between host and 设备. A 控制传输 consists of three phases: a setup stage where the host sends an 8-byte setup 数据包 describing the request, an optional data stage where either the host sends data or the 设备 returns data, and a status stage where the recipient acknowledges the operation.

The setup 数据包 has five fields:

- `bmRequestType`: describes the direction (in or out), the type of request (standard, class, or vendor), and the recipient (设备, 接口, 端点, or other).
- `bRequest`: the request number. Standard requests have well-known numbers (GET_DESCRIPTOR = 6, SET_ADDRESS = 5, and so on). Class and vendor requests have class-specific or vendor-specific meanings.
- `wValue`: a 16-bit parameter, often used to specify a 描述符 index or a value to set.
- `wIndex`: another 16-bit parameter, often used to specify an 接口 or 端点.
- `wLength`: the number of bytes in the data stage (zero if there is no data stage).

Every USB 设备 must support a small set of standard requests: GET_DESCRIPTOR, SET_ADDRESS, SET_CONFIGURATION, and a few others. The 框架 handles all of these at enumeration time. Your 驱动程序 may also issue vendor-specific requests to configure the 设备 in ways the standard does not define.

For example, the FTDI 驱动程序 issues vendor-specific requests like `FTDI_SIO_SET_BAUD_RATE`, `FTDI_SIO_SET_LINE_CTRL`, and `FTDI_SIO_MODEM_CTRL` to program the chip. These requests are documented in FTDI's application notes; they are not part of USB itself, but they work over the USB control-transfer mechanism.

When your 驱动程序 needs to issue a vendor-specific control request, the pattern is the one we showed in 第3节： construct the setup 数据包, copy it into 帧 zero of a 控制传输, copy any data into 帧 one (for data-stage requests), and submit. The 框架 handles the three phases and calls your 回调 when the transfer completes.

### The Serial Mental Model

The serial side of 第26章 is about a much older and much simpler protocol than USB. Serial communication over a UART is one of the oldest ways two computers can talk to each other, and its simplicity is both its strength and its limitation. A reader coming to UART after USB will find the protocol almost trivially small. But the integration with the rest of the operating system, the tty discipline, 波特率 management, 奇偶校验, 流控制, and the two-worlds split between `uart(4)` and `ucom(4)`, is where most of the actual work lives.

#### The UART as a Piece of Hardware

A UART is a *Universal Asynchronous Receiver/Transmitter*: a chip that converts bytes into a serial bit stream on a wire and back again. The classical UART has two pins for data (TX and RX), two pins for 流控制 (RTS and CTS), four pins for modem status (DTR, DSR, DCD, RI), a ground pin, and occasionally a pin for a second "ring" signal that most modern equipment ignores. On a classic PC, the 串行端口 has a nine-pin or twenty-five-pin D-subminiature connector and operates at RS-232 voltage levels (typically +/- 12 V). Modern embedded UARTs usually operate at 3.3 V or 1.8 V logic levels; a level converter chip sits between the UART and the RS-232 connector if a compatible port is needed.

Inside the UART, the core is a shift 注册. When the 驱动程序 writes a byte to the UART's transmit 注册, the UART adds a start bit, shifts the byte out bit by bit at the configured 波特率, adds an optional 奇偶校验 bit, and then adds one or two 停止位. When a receiving UART detects a falling edge (the start bit), it samples the line at the middle of each bit time, assembles the bits into a byte, checks the 奇偶校验, verifies the stop bit, and then stores the byte in its receive 注册. If any of those steps fails (the 奇偶校验 does not match, the stop bit is wrong, the framing is off), the UART notes a framing error, a 奇偶校验 error, or a break condition in its status 注册.

On most modern UARTs, the single receive and transmit 寄存器 are backed by small first-in-first-out 缓冲区 (FIFOs). The 16550A UART, still the de facto standard, has a 16-byte FIFO on each side. A 驱动程序 that programs the FIFO with an appropriate "trigger level" can let the hardware 缓冲区 incoming bytes and raise an interrupt only when the FIFO passes the trigger level. This is the difference between "one interrupt per byte" (slow) and "one interrupt per trigger level" (fast). The 16550A's FIFO is a big part of why this chip became the universal PC standard.

The UART's speed is controlled by a *波特率 divisor*: the UART has an input clock (often 1.8432 MHz on classic PC hardware), and the 波特率 is the clock divided by 16 times the divisor. A divisor of 1 with a 1.8432 MHz clock gives 115200 baud. A divisor of 12 gives 9600 baud. The FreeBSD `ns8250` 驱动程序 computes the divisor from the requested 波特率 and programs it into the UART's divisor-latch 寄存器. Section 4 walks through this code.

RS-232 framing is the full protocol: start bit (one), 数据位 (five, six, seven, or eight), optional 奇偶校验 bit (none, odd, even, mark, or space), stop bit (one or two). A typical modern configuration is "8N1": eight 数据位, no 奇偶校验, one stop bit. An older configuration sometimes seen on industrial equipment is "7E1": seven 数据位, even 奇偶校验, one stop bit. The 驱动程序 programs the UART's line control 注册 to select the framing; `struct termios` carries the configuration from user space.

#### Flow Control

The UART can transmit faster than the receiver can read if the receiver's code is slow or is doing other work. *Flow control* is how the receiver tells the transmitter to pause. Two mechanisms exist.

*Hardware 流控制* uses two extra wires: *RTS* (Request To Send) from the receiver, and *CTS* (Clear To Send) from the transmitter's perspective (it is the wire the other side drives). When the receiver's 缓冲区 is filling up, it deasserts RTS. The transmitter, seeing CTS deasserted, stops transmitting. When the 缓冲区 empties, the receiver reasserts RTS, CTS asserts on the other side, and transmission resumes. Hardware 流控制 is reliable and requires no software overhead on either side; it is the default choice when the hardware supports it.

*Software 流控制*, also called XON/XOFF, uses two in-band bytes: XOFF (traditionally ASCII DC3, 0x13) to pause transmission, and XON (ASCII DC1, 0x11) to resume. The receiver sends XOFF when it is almost full and XON when it has room again. This mechanism works over a three-wire connection (TX, RX, ground) with no extra pins, at the cost of reserving two byte values for control use. If you are sending binary data that may contain 0x11 or 0x13, you cannot use software 流控制; hardware 流控制 is the only option.

FreeBSD's tty discipline handles software 流控制 entirely in software, at the line-discipline layer, with no involvement from the UART 驱动程序. Hardware 流控制 is partly in the 驱动程序 (the 驱动程序 programs the UART's automatic RTS/CTS feature if the chip supports it) and partly in the tty layer. A 驱动程序 author should know which 流控制 method the tty layer has selected; the CRTSCTS flag in `struct termios` signals hardware 流控制.

#### /dev/ttyuN and /dev/cuauN: A FreeBSD-Specific Quirk

The FreeBSD tty layer creates two 设备 nodes per 串行端口. The *callin* node is `/dev/ttyuN` (where N is the port number, 0 for the first port). The *callout* node is `/dev/cuauN`. The distinction is historical, from the days of dial-up modems, and remains useful.

A process opening `/dev/ttyuN` is saying "I want to answer an incoming call": the open 块 until the modem raises DCD (Data Carrier Detect). Once DCD is up, the open completes. When DCD drops, the open process receives SIGHUP. The node is for incoming connections.

A process opening `/dev/cuauN` is saying "I want to make an outgoing call": the open succeeds immediately, without 块ing on DCD. The process can then dial out or, on non-modem uses, simply talk to the 串行端口. The node is for outgoing connections, and more generally for any use that does not require modem semantics.

In modern use, when a 串行端口 is connected to something that is not a modem (a microcontroller, a console, a GPS receiver), the right node to open is almost always `/dev/cuau0`. Opening `/dev/ttyu0` on a non-modem port will usually hang, because DCD is never asserted. The distinction is FreeBSD-specific; Linux has no callout nodes and uses `/dev/ttyS0` or `/dev/ttyUSB0` for everything.

The chapter's labs will use `/dev/cuau0` and the simulated pair `/dev/nmdm0A`/`/dev/nmdm0B` for serial exercises. The callin nodes are not used.

#### Two Worlds: `uart(4)` and `ucom(4)`

FreeBSD separates real UART hardware from USB-to-serial bridges into two distinct subsystems. The separation is not visible from user space (a USB serial adapter and a built-in 串行端口 both appear as tty 设备), but it is very visible from inside the 内核, and a 驱动程序 author must not confuse the two.

`uart(4)` is the subsystem for real UARTs. Its scope includes the built-in 串行端口 on a PC motherboard, PCI serial cards, the PrimeCell `PL011` found on ARM embedded boards, the embedded SoC UARTs on i.MX, Marvell, Qualcomm, Broadcom, and Allwinner platforms, and so on. The `uart` subsystem lives in `/usr/src/sys/dev/uart/`. Its core code is in `uart_core.c` and `uart_tty.c`. Its canonical hardware 驱动程序 is `uart_dev_ns8250.c`. A 驱动程序 that 附加 to a real UART writes a `uart_class` and a small set of `uart_ops`, and the subsystem handles everything else. The `/dev` nodes that `uart(4)` creates are called `ttyu0`, `ttyu1`, and so on (callin) and `cuau0`, `cuau1`, and so on (callout).

`ucom(4)` is the 框架 for USB-to-serial bridges: FTDI, Prolific, Silicon Labs, WCH, and similar. Its scope is *not* a UART at all; it is a USB 设备 whose 端点 happen to behave like a 串行端口. The `ucom` 框架 lives in `/usr/src/sys/dev/usb/serial/`. Its header is `usb_serial.h`. Its body is `usb_serial.c`. A USB-to-串行驱动程序 writes USB 探测, 附加, and 分离 methods as in any other USB驱动程序, and then 寄存器 a `struct ucom_回调` with the 框架. The 回调 has entries for "open", "close", "set line parameters", "start reading", "stop reading", "start writing", and so on. The 框架 creates the `/dev` node (called `ttyU0`, `ttyU1` for callin, `cuaU0`, `cuaU1` for callout, note the capital U) and runs the tty discipline on top of the 驱动程序's USB transfers.

The two worlds never mix. `uart(4)` is for hardware that is physically a UART. `ucom(4)` is for USB 设备 that behave like a UART. A USB-to-serial adapter is a `ucom` 驱动程序, not a `uart` 驱动程序. A PCI serial card is a `uart` 驱动程序 (specifically, a shim in `uart_总线_pci.c`), not a `ucom` 驱动程序. The user-space 接口 is similar (both produce `cu*` 设备 nodes), but the 内核 code is entirely disjoint.

A historical note that sometimes confuses readers: FreeBSD once had a separate `sio(4)` 驱动程序 for 16550-family UARTs. `sio(4)` was retired years ago and is not present in FreeBSD 14.3. If you see references to `sio` in older documentation, translate them mentally to `uart(4)`. Do not try to find or extend `sio`; it is gone.

#### What termios Carries, and Where It Goes

`struct termios` is the user-space structure that configures a tty. It has five fields: `c_iflag` (input flags), `c_oflag` (output flags), `c_cflag` (control flags), `c_lflag` (local flags), `c_cc` (control characters), and two speed fields `c_ispeed` and `c_ospeed`. The fields are manipulated with `tcgetattr(3)`, `tcsetattr(3)`, and the shell command `stty(1)`.

A UART 驱动程序 cares almost exclusively about `c_cflag` and the speed fields. `c_cflag` carries:

- `CSIZE`: the character size (CS5, CS6, CS7, CS8).
- `CSTOPB`: if set, two 停止位; if clear, one.
- `PARENB`: if set, 奇偶校验 is enabled; the type depends on `PARODD`.
- `PARODD`: if set with `PARENB`, odd 奇偶校验; if clear with `PARENB`, even 奇偶校验.
- `CRTSCTS`: hardware 流控制.
- `CLOCAL`: ignore modem status lines; treat the link as local.
- `CREAD`: enable the receiver.

When user space calls `tcsetattr`, the tty layer checks the request, invokes the 驱动程序's `param` method (via the `tsw_param` 回调 in `ttydevsw`), and the 驱动程序 translates the termios fields into hardware 注册 settings. The `uart_tty.c` bridge code walks through this in full and is the best place to see the translation happen.

`c_iflag`, `c_oflag`, and `c_lflag` are mostly handled by the tty line discipline, not by the 驱动程序. They control things like whether the line discipline maps CR to LF, whether echo is enabled, whether canonical mode is active, and so on. A UART 驱动程序 does not need to know any of that; the tty layer handles it.

#### Flow Control at the Multiple Layers of a TTY

Flow control sounds like a single concept, but in practice there are several independent layers that can each throttle the data flow. Understanding the layers helps debug situations where data is mysteriously not flowing.

The lowest layer is electrical. On a real RS-232 line, 流控制 signals (RTS, CTS, DTR, DSR) are physical pins on the connector. The remote side's transmitter only sends data when its CTS pin is asserted. The local side asserts the RTS pin to tell the remote it is ready to receive. For this to work, the cable must pass RTS and CTS through correctly, and both ends must have 流控制 configured consistently.

The next layer is in the UART chip itself. Some 16650 and later UARTs have automatic 流控制: if configured, the chip itself monitors CTS and pauses the transmitter without 驱动程序 involvement. The `CRTSCTS` flag in `c_cflag` enables this.

The next layer is in the UART 框架's 环形缓冲区. When the RX ring fills past a high-water mark, the 框架 deasserts RTS (if 流控制 is enabled) to tell the remote side to pause. When it drains below a low-water mark, RTS is reasserted.

The next layer is the tty line discipline, which has its own input and output queues. The line discipline can also generate XON/XOFF bytes (0x11 and 0x13) if `IXON` and `IXOFF` are set in `c_iflag`. These are software 流控制 signals.

The highest layer is the userland program's read loop. If the program is slow at consuming data, bytes accumulate at every layer below it.

When debugging flow-control issues, check each layer. Use `stty -a -f /dev/cuau0` to see what `c_cflag` and `c_iflag` have active. Use `comcontrol /dev/cuau0` to see the current modem signals. Use a multimeter or oscilloscope on the physical signals if you can. Work down the layers until you find the one that is actually 块ing the flow.

#### Why Baud Rate Errors Are Insidious

A common class of serial bug is a baud-rate mismatch that almost works. Suppose one side is running at 115200 and the other at 114400 (which is what you get from a slightly-off crystal). Most bytes will come through, but a few will be corrupted. The exact error rate depends on the bit pattern. Long runs of one polarity drift further than alternating patterns.

Even worse, the error rate depends on the byte being sent. ASCII printable characters are in the range 0x20 to 0x7e, where the bits are well-distributed. Non-printable characters like 0xff or 0x00 are more likely to suffer bit errors because they present long runs of one polarity.

If you find your 串行驱动程序 "mostly works" but drops or corrupts a few bytes out of thousands, suspect a baud-rate mismatch before suspecting a logic bug in your 驱动程序. Compare the actual divisor the chip is using against the expected divisor. If they differ, the 波特率 is not what you asked for.

The 16550 uses a clock source (usually 1.8432 MHz) divided by a 16-bit divisor to produce 16 times the 波特率. For 115200, the divisor is `(1843200 / (115200 * 16)) = 1`. For 9600, it is 12. For arbitrary rates, the divisor may not be an integer, and the closest integer produces a rounded rate. A rate of 115200 requested from a 24 MHz clock would produce a divisor of `(24000000 / (115200 * 16)) = 13.02`, rounding to 13, giving an actual rate of `(24000000 / (13 * 16)) = 115384`, which is 0.16% off. Standard tolerance for serial communication is 2-3%, so 0.16% is fine.

When you configure a UART for a nonstandard 波特率, check whether the rate can be represented exactly. If not, test with actual data exchange, not just a loopback check.

#### Historical Note on Minor Numbers

Older FreeBSD versions encoded a lot of information into the minor numbers of serial 设备 files. Different minor numbers for the "callin" side vs the "callout" side, for hardware-flow vs software-flow, and for various lock states. This encoding is largely gone in modern FreeBSD; the distinctions are now handled by separate 设备 nodes with different names (`ttyu` vs `cuau`, with suffixes for lock and init states). If you see odd minor-number manipulation in old code, know that modern code does not need it.

#### 总结 Section 1

Section 1 has established the two mental models 第26章 depends on. The USB model is a tree-structured, host-controlled, 热插拔gable serial 总线 with four transfer types, a rich 描述符 hierarchy, and a lifecycle in which physical events drive 内核 events. The serial model is a simple shift-注册 hardware protocol with 波特率, 奇偶校验, 停止位, and optional 流控制, integrated into FreeBSD through a subsystem split between `uart(4)` for real UARTs and `ucom(4)` for USB-to-serial bridges, and exposed to user space through the tty discipline and 设备 nodes like `/dev/cuau0`.

Before moving on, spend a few minutes with `usbconfig` on a real system. The vocabulary you have just learned is easier to keep straight once you have seen a real USB 设备's 描述符 with your own eyes.

### Exercise: Use `usbconfig` and `dmesg` to Explore USB Devices on Your System

This exercise is a short hands-on checkpoint that grounds Section 1's vocabulary in a 设备 you can see. Perform it on your lab VM (or on any FreeBSD 14.3 system with at least one USB 设备 connected). It takes about fifteen minutes.

**Step 1. Inventory.** Run `usbconfig` with no arguments:

```console
$ usbconfig
ugen0.1: <Intel EHCI root HUB> at usbus0, cfg=0 md=HOST spd=HIGH (0mA)
ugen0.2: <Generic Storage> at usbus0, cfg=0 md=HOST spd=HIGH (500mA)
ugen0.3: <Logitech USB Mouse> at usbus0, cfg=0 md=HOST spd=LOW (98mA)
```

The first line is the root hub. Each other line is a 设备. Read the format: `ugenN.M` where N is the 总线 number and M is the 设备 number; the description in angle brackets is the 设备's string; `cfg` is the active configuration; `md` is the mode (HOST or DEVICE); `spd` is the 总线 speed (LOW, FULL, HIGH, SUPER); the parenthesised current is the maximum 总线-supplied power draw.

**Step 2. Dump a 设备's 描述符.** Pick one of the non-root-hub 设备 and dump its 设备 描述符:

```console
$ usbconfig -d ugen0.2 dump_device_desc

ugen0.2: <Generic Storage> at usbus0, cfg=0 md=HOST spd=HIGH (500mA)

  bLength = 0x0012
  bDescriptorType = 0x0001
  bcdUSB = 0x0200
  bDeviceClass = 0x0000  <Probed by interface class>
  bDeviceSubClass = 0x0000
  bDeviceProtocol = 0x0000
  bMaxPacketSize0 = 0x0040
  idVendor = 0x13fe
  idProduct = 0x6300
  bcdDevice = 0x0112
  iManufacturer = 0x0001  <Generic>
  iProduct = 0x0002  <Storage>
  iSerialNumber = 0x0003  <0123456789ABCDE>
  bNumConfigurations = 0x0001
```

Read each field. Notice that `bDeviceClass` is zero: that is the USB convention for "the class is defined per 接口, not at the 设备 level." For this 设备, the 接口 class will be Mass Storage (0x08).

**Step 3. Dump the active configuration.** Now dump the configuration 描述符, which includes the 接口 and 端点:

```console
$ usbconfig -d ugen0.2 dump_curr_config_desc

ugen0.2: <Generic Storage> at usbus0, cfg=0 md=HOST spd=HIGH (500mA)

  Configuration index 0

    bLength = 0x0009
    bDescriptorType = 0x0002
    wTotalLength = 0x0020
    bNumInterface = 0x0001
    bConfigurationValue = 0x0001
    iConfiguration = 0x0000  <no string>
    bmAttributes = 0x0080
    bMaxPower = 0x00fa

    Interface 0
      bLength = 0x0009
      bDescriptorType = 0x0004
      bInterfaceNumber = 0x0000
      bAlternateSetting = 0x0000
      bNumEndpoints = 0x0002
      bInterfaceClass = 0x0008  <Mass storage>
      bInterfaceSubClass = 0x0006  <SCSI>
      bInterfaceProtocol = 0x0050  <Bulk only>
      iInterface = 0x0000  <no string>

     Endpoint 0
        bLength = 0x0007
        bDescriptorType = 0x0005
        bEndpointAddress = 0x0081  <IN>
        bmAttributes = 0x0002  <BULK>
        wMaxPacketSize = 0x0200
        bInterval = 0x0000

     Endpoint 1
        bLength = 0x0007
        bDescriptorType = 0x0005
        bEndpointAddress = 0x0002  <OUT>
        bmAttributes = 0x0002  <BULK>
        wMaxPacketSize = 0x0200
        bInterval = 0x0000
```

Every field in Section 1's vocabulary is right there. The 接口 class is 0x08 (Mass Storage). The subclass is 0x06 (SCSI). The protocol is 0x50 (Bulk-only Transport). There are two 端点. Endpoint 0 has address 0x81 (the high bit indicates IN direction, the low five bits are the 端点 number, 1). Endpoint 1 has address 0x02 (the high bit is clear, meaning OUT; the 端点 number is 2). Both 端点 are bulk. Both have a maximum 数据包 size of 0x0200 = 512 bytes. The interval is zero because bulk 端点 do not use it.

**Step 4. Match this against `dmesg`.** Run `dmesg | grep -A 3 ugen0.2` (or look at the last boot's output for the matching 设备). You should see a line like:

```text
ugen0.2: <Generic Storage> at usbus0
umass0 on uhub0
umass0: <Generic Storage, class 0/0, rev 2.00/1.12, addr 2> on usbus0
```

This is the same information, formatted by the 内核's own logging. The 驱动程序 that 附加ed is `umass`, which is FreeBSD's USB mass 存储驱动程序, and it 附加ed to the Mass Storage 接口 class.

**Step 5. Try `usbconfig -d ugen0.3 dump_all_config_desc` on another 设备.** A mouse, a keyboard, or a flash drive will all work. Compare the 端点 types: a mouse has one interrupt-IN 端点; a flash drive has one bulk-IN and one bulk-OUT; a keyboard has one interrupt-IN. The pattern holds.

If you want a small additional exercise, write down the vendor and product identifiers of one of your 设备. In Section 2 you will be asked to put vendor and product identifiers into a 匹配表; using ones you can see now is concrete.

### 总结 Section 1

Section 1 has done four things. It established the mental model of a transport: the protocol plus the lifecycle, plus the three broad categories of work (matching, transfer mechanics, 热插拔 lifecycle) that a transport-specific 驱动程序 has to add to its New总线 foundation. It built the USB model: host and 设备, hubs and ports, classes, 描述符 with their nested structure, the four transfer types, and the 热插拔 lifecycle. It built the serial model: the UART as a shift 注册 with a baud generator, RS-232 framing, 波特率 and 奇偶校验 and 停止位, hardware and software 流控制, the FreeBSD-specific callin and callout node distinction, the two-worlds split between `uart(4)` and `ucom(4)`, and the role of `struct termios`. And it anchored the vocabulary in a concrete exercise that reads 描述符 off a real USB 设备 with `usbconfig`.

From here, the chapter turns to code. Section 2 builds a USB驱动程序 skeleton: 探测, 附加, 分离, 匹配表, registration macros. Section 3 makes that 驱动程序 do real work by adding transfers. Section 4 turns to the serial side, walks through the `uart(4)` subsystem with a real 驱动程序 as the guide, and explains where `ucom(4)` fits in. Section 5 brings the material back to the lab and teaches how to test USB and 串行驱动程序 without physical hardware. Each section builds on the mental models just established. If a later paragraph refers to a 描述符 or a transfer type and the term does not feel immediate, return to Section 1 for a quick refresher before continuing.

## 第2节： Writing a USB Device Driver

### Moving from Concepts to Code

Section 1 built a mental picture of USB: a host that talks to 设备 through a tree of hubs, 设备 that describe themselves with nested 描述符, four transfer types that cover every conceivable traffic pattern, and a 热插拔 lifecycle that 驱动程序 must respect because USB 设备 appear and disappear at any moment. Section 2 turns those concepts into a real 驱动程序 skeleton. By the end of this section, you will have a USB驱动程序 that compiles, loads, 附加 to a matching 设备, and 分离 cleanly when the 设备 is unplugged. It will not yet perform data transfers; that is the job of Section 3. But the scaffolding you build here is the same scaffolding every FreeBSD USB驱动程序 uses, from the tiniest notification LED to the most complex mass-storage controller.

The discipline you learned in 第25章 carries forward unchanged. Every resource must have an owner. Every successful allocation in `附加` must be paired with an explicit release in `分离`. Every failure path must leave the system in a clean state. The labelled-goto cleanup chain, the errno-returning helper functions, the softc-based resource tracking, the rate-limited logging: all of it still applies. What changes is the set of resources you manage. Instead of 总线 resources allocated through New总线 and a 字符设备 created through `make_dev`, you will manage USB transfer objects allocated through the USB stack and, optionally, a `/dev` entry created through the `usb_fifo` 框架. The shape of the code stays the same. Only the specific calls change.

This section moves from the outside in. It begins by explaining where a USB驱动程序 sits inside the FreeBSD USB subsystem, because placing the 驱动程序 in its correct environment is a prerequisite for understanding every call that follows. It then covers the 匹配表, which is how a USB驱动程序 declares which 设备 it wants. It walks through `探测` and `附加`, the two halves of the 驱动程序's entry point into the world. It covers the softc layout, which is where the 驱动程序 keeps its per-设备 state. It presents the cleanup chain, which is how the 驱动程序 unwinds its own work when `分离` is called. And it ends with the registration macros that bind the 驱动程序 to the 内核模块 system.

Along the way, the chapter uses `uled.c` as a recurring reference. That is a real FreeBSD 驱动程序, about three hundred lines long, located at `/usr/src/sys/dev/usb/misc/uled.c`. It is short enough to read end to end in a single sitting and rich enough to show every piece of machinery a USB驱动程序 needs. If you want to ground every idea in this section against real code, open that file now in another window and keep it open. Every time the chapter references a pattern, you will be able to see the pattern in a working 驱动程序.

### Where a USB Driver Sits in the FreeBSD Tree

FreeBSD's USB subsystem lives under `/usr/src/sys/dev/usb/`. That directory contains everything from the 主机控制器 驱动程序 at the bottom (`controller/ehci.c`, `controller/xhci.c`, and so on) to the class 驱动程序 higher up (`net/if_cdce.c`, `wlan/if_rum.c`, `input/ukbd.c`), to 串行驱动程序 (`serial/uftdi.c`, `serial/uplcom.c`), to generic 框架 code (`usb_设备.c`, `usb_transfer.c`, `usb_request.c`). When a new 驱动程序 is added to the tree, it goes into one of these subdirectories according to its role. A 驱动程序 for a blinking-LED gadget belongs under `misc/`. A 驱动程序 for a network adapter belongs under `net/`. A 驱动程序 for a serial adapter belongs under `serial/`. For your own work, you will not add files to `/usr/src/sys/dev/usb/` directly; you will build out-of-tree modules in your own workshop directory, the same way 第25章 did. The directory layout matters for reading the source, not for writing it.

Every FreeBSD USB驱动程序 sits somewhere in a small vertical stack. At the bottom is the 主机控制器 驱动程序, which actually talks to the silicon. Above that is the USB 框架, which handles 描述符 parsing, 设备 enumeration, transfer scheduling, hub routing, and the generic machinery every 设备 needs. Above the 框架 are the class 驱动程序, which you will write. A class 驱动程序 附加 to a USB 接口, not to the 总线 directly. This is the most important architectural point in the chapter.

In the New总线 tree, the 附加ment relationship looks like this:

```text
nexus0
  └─ pci0
       └─ ehci0   (or xhci0, depending on the host controller)
            └─ usbus0
                 └─ uhub0   (the root hub)
                      └─ uhub1 (a downstream hub, if present)
                           └─ [class driver]
```

The 驱动程序 you will write 附加 to `uhub`, not to `us总线`, not to `ehci`, and not to `pci`. The USB 框架 walks the 设备 描述符, creates a child for each 接口, and offers those children to class 驱动程序 through the new总线 探测 mechanism. When your 驱动程序's 探测 routine is called, it is being asked: "here is an 接口; is it yours?" The 匹配表 in your 驱动程序 is how you answer that question.

There is one subtle point to absorb. A USB 设备 can expose multiple 接口 simultaneously. A multi-function peripheral (say, a USB audio 设备 with a headset and a microphone on the same silicon) exposes one 接口 for playback and another for capture. FreeBSD gives each 接口 its own new总线 child, and each child can be claimed by a different 驱动程序. This is why USB驱动程序 附加 at the 接口 level: it lets the 框架 route 接口 independently. Your 驱动程序 should not assume the 设备 has only one 接口. When you write the 匹配表, you write it against a specific 接口, identified by its class, subclass, protocol, or by its vendor/product pair plus an optional 接口 number.

### The Match Table: Telling the Kernel Which Devices Are Yours

A USB驱动程序 advertises which 设备 it will accept through an array of `STRUCT_USB_HOST_ID` entries. This is analogous to the PCI 匹配表 from Chapter 23, but with USB-specific fields. The authoritative definition lives in `/usr/src/sys/dev/usb/usbdi.h`. Each entry specifies one or more of the following: a 供应商ID, a 产品ID, a 设备 class/subclass/protocol triple, an 接口 class/subclass/protocol triple, or a manufacturer-defined bcdDevice range. You can match broadly (any 设备 that advertises 接口 class 0x03, which is HID) or narrowly (the single 设备 with vendor 0x0403 and product 0x6001, which is an FTDI FT232). Most 驱动程序 match narrowly, because most real 设备 have 驱动程序-specific quirks that apply only to particular hardware revisions.

The 框架 provides convenience macros to build match entries without having to initialize each field by hand. The most common are `USB_VPI(vendor, product, info)` for vendor/product pairs with an optional 驱动程序-specific information field, and the more verbose form where you fill in `mfl_`, `pfl_`, `dcl_`, `dcsl_`, `dcpl_`, `icl_`, `icsl_`, `icpl_` flags to indicate which fields are significant. For clarity and maintainability, 驱动程序 written today tend to use the compact macros whenever they are applicable.

Here is how `uled.c` declares its 匹配表. The source is in `/usr/src/sys/dev/usb/misc/uled.c`:

```c
static const STRUCT_USB_HOST_ID uled_devs[] = {
    {USB_VPI(USB_VENDOR_DREAMCHEEKY, USB_PRODUCT_DREAMCHEEKY_WEBMAIL_NOTIFIER, 0)},
    {USB_VPI(USB_VENDOR_RISO_KAGAKU, USB_PRODUCT_RISO_KAGAKU_WEBMAIL_NOTIFIER, 0)},
};
```

Two entries, each naming a specific vendor/product pair. The third argument to `USB_VPI` is an unsigned integer that the 驱动程序 can use to distinguish variants at 探测 time; `uled` sets it to zero because both 设备 behave the same way. The vendor and product symbolic names resolve to numeric identifiers defined in `/usr/src/sys/dev/usb/usbdevs.h`, which is a large table generated from `/usr/src/sys/dev/usb/usbdevs`. Adding a new match entry for your own development hardware often means adding a line to `usbdevs` and regenerating the header, or bypassing the symbolic names entirely and writing the hexadecimal values directly in the 匹配表.

For your own out-of-tree 驱动程序, you do not need to touch `usbdevs` at all. You can write:

```c
static const STRUCT_USB_HOST_ID myfirst_usb_devs[] = {
    {USB_VPI(0x16c0, 0x05dc, 0)},  /* VOTI / generic test VID/PID */
};
```

The numeric form is perfectly acceptable. Use it when you are prototyping against a specific 设备 and do not yet want to propose additions to the upstream `usbdevs` file.

One important detail about 匹配表s: the `STRUCT_USB_HOST_ID` type includes a flag byte that records which fields are meaningful. When you use `USB_VPI`, the macro fills in those flags for you. If you hand-build an entry with literal braces, you must also fill in the flags yourself, because a zero flag byte means "match anything," and you rarely want that. Prefer the macros.

The 匹配表 is plain data. It does not allocate memory, it does not touch hardware, and it does not depend on any per-设备 state. It is loaded into the 内核 along with the module and used by the 框架 every time a new USB 设备 is enumerated.

### The `探测` Method

The USB 框架 calls a 驱动程序's `探测` method once per 接口 when a matching-like candidate is presented. The goal of `探测` is to answer a single question: "Should this 驱动程序 附加 to this 接口?" The method must not touch hardware. It must not allocate resources. It must not sleep. All it does is look at the USB 附加 argument, compare it against the 匹配表, and return either a 总线-探测 value (indicating a match, with an associated priority) or `ENXIO` (indicating that this 驱动程序 does not want this 接口).

The 附加 argument lives in a structure called `struct usb_附加_arg`, defined in `/usr/src/sys/dev/usb/usbdi.h`. It carries the 供应商ID, the 产品ID, the 设备 描述符, the 接口 描述符, and a handful of helper fields. New总线 lets a 驱动程序 retrieve it through `设备_get_ivars(dev)`. For USB驱动程序, the 框架 provides a wrapper called `usbd_lookup_id_by_uaa` that takes a 匹配表 and an 附加 argument and returns zero on a match or a nonzero errno on a miss. This wrapper encapsulates every case the 驱动程序 needs to handle: vendor/product matching, class/subclass/protocol matching, the flag-byte logic, and the 接口-level dispatch.

A complete 探测 method for our running example looks like this:

```c
static int
myfirst_usb_probe(device_t dev)
{
    struct usb_attach_arg *uaa = device_get_ivars(dev);

    if (uaa->usb_mode != USB_MODE_HOST)
        return (ENXIO);

    if (uaa->info.bConfigIndex != 0)
        return (ENXIO);

    if (uaa->info.bIfaceIndex != 0)
        return (ENXIO);

    return (usbd_lookup_id_by_uaa(myfirst_usb_devs,
        sizeof(myfirst_usb_devs), uaa));
}
```

The three guard clauses at the top of the function are worth explaining in detail, because they reflect standard USB-驱动程序 hygiene.

The first guard rejects the case where the USB stack is acting as a 设备 rather than a host. FreeBSD's USB stack can operate in USB-on-the-Go 设备 mode, where the machine itself appears as a USB peripheral to some other host. Most 驱动程序 are host-side 驱动程序 and have no meaningful behavior in 设备 mode, so they reject it immediately.

The second guard rejects configurations other than index zero. USB 设备 can expose multiple configurations, and a 驱动程序 usually targets one specific configuration. Restricting 探测 to configuration index zero keeps the logic simple for the common case.

The third guard rejects 接口 other than index zero. If the 设备 has multiple 接口 and you are writing a 驱动程序 for the first one, this clause is what ensures the 框架 does not offer you the other 接口 by mistake.

After the guards, the call to `usbd_lookup_id_by_uaa` does the real matching work. If the 设备's vendor, product, class, subclass, or protocol matches any entry in the table, the function returns zero, and the 探测 method returns zero, which the USB 框架 interprets as "this 驱动程序 wants this 设备." Returning `ENXIO` tells the 框架 to try another candidate 驱动程序. If no candidate wants the 设备, it ends up 附加ed to `ugen`, the generic USB驱动程序, which exposes raw 描述符 and transfers through `/dev/ugenN.M` nodes but provides no 设备-specific behavior.

A subtle point worth noting: `探测` returns zero for a match rather than a positive 总线-探测 value. Other FreeBSD 总线 框架 use positive values like `BUS_PROBE_DEFAULT` to indicate a priority, but for USB the convention is zero for match and a nonzero errno for non-match. The 框架 handles priority through the dispatch order rather than through 探测 return values.

### The `附加` Method

Once `探测` reports a match, the 框架 calls `附加`. This is where the 驱动程序 does real work: allocate its softc, record the parent 设备 pointer, lock the 接口, set up transfer channels (covered in Section 3), create a `/dev` entry if the 驱动程序 is user-facing, and log a short informational message. Every allocation and registration in `附加` has to be paired with a symmetric release in `分离`, and because any step can fail, the function must have a clear cleanup path from every failure point.

A minimal 附加 method looks like this:

```c
static int
myfirst_usb_attach(device_t dev)
{
    struct usb_attach_arg *uaa = device_get_ivars(dev);
    struct myfirst_usb_softc *sc = device_get_softc(dev);
    int error;

    device_set_usb_desc(dev);

    mtx_init(&sc->sc_mtx, "myfirst_usb", NULL, MTX_DEF);

    sc->sc_udev = uaa->device;
    sc->sc_iface_index = uaa->info.bIfaceIndex;

    error = usbd_transfer_setup(uaa->device, &sc->sc_iface_index,
        sc->sc_xfer, myfirst_usb_config, MYFIRST_USB_N_XFER,
        sc, &sc->sc_mtx);
    if (error != 0) {
        device_printf(dev, "usbd_transfer_setup failed: %d\n", error);
        goto fail_mtx;
    }

    sc->sc_dev = make_dev(&myfirst_usb_cdevsw, device_get_unit(dev),
        UID_ROOT, GID_WHEEL, 0644, "myfirst_usb%d", device_get_unit(dev));
    if (sc->sc_dev == NULL) {
        device_printf(dev, "make_dev failed\n");
        error = ENOMEM;
        goto fail_xfer;
    }
    sc->sc_dev->si_drv1 = sc;

    device_printf(dev, "attached\n");
    return (0);

fail_xfer:
    usbd_transfer_unsetup(sc->sc_xfer, MYFIRST_USB_N_XFER);
fail_mtx:
    mtx_destroy(&sc->sc_mtx);
    return (error);
}
```

Read through this function top to bottom. Each 块 does one thing.

The call to `设备_set_usb_desc` fills in the New总线 设备 description string from the USB 描述符. After this call, `设备_printf` messages will include the manufacturer and product strings read from the 设备 itself, which makes logs much more informative.

The call to `mtx_init` creates a 互斥锁 that will protect the per-设备 state. Every USB 传输回调 runs under this 互斥锁 (the 框架 takes it for you around the 回调), so everything the 回调 touches must be serialised by it. 第25章 introduced 互斥锁es; the usage here is the same.

The two `sc->sc_` assignments cache two pointers that the rest of the 驱动程序 will need. `sc->sc_udev` is the `struct usb_设备 *` that the 驱动程序 uses when issuing USB requests. `sc->sc_iface_index` identifies the 接口 index this 驱动程序 附加ed to, so later transfer-setup calls target the right 接口.

The call to `usbd_transfer_setup` is the biggest single operation in `附加`. It allocates and configures all the transfer objects the 驱动程序 will use, based on a configuration array (`myfirst_usb_config`) that Section 3 will examine in detail. If this call fails, the 驱动程序 has not yet allocated anything except the 互斥锁, so the cleanup path goes to `fail_mtx` and destroys the 互斥锁.

The call to `make_dev` creates the user-visible `/dev` node. The 第25章 pattern applies here: set `si_drv1` on the cdev so that the cdevsw handlers can retrieve the softc through `dev->si_drv1`. If this call fails, the cleanup path goes to `fail_xfer`, which also runs the unsetup for the transfers before destroying the 互斥锁.

The `return (0)` on the happy path is the contract with the 框架: a zero return means the 设备 is 附加ed and the 驱动程序 is ready.

The two labels at the bottom implement the labelled-goto cleanup chain from 第25章. Each label corresponds to the state the 驱动程序 has reached at the time the failure happened, and the cleanup fall-through runs exactly the teardown steps needed to undo the work done so far. When you read a FreeBSD 驱动程序 and see this pattern, you are looking at the same discipline you practised in 第25章 applied to a new set of resources.

One important detail about the USB 框架 that 第25章 did not need to cover: if you look at `uled.c` or any other real USB驱动程序, you will sometimes see `usbd_transfer_setup` accept a pointer to the 接口 index rather than an integer. The 框架 can modify that pointer in the case of virtual or multiplexed 接口; pass it by address, not by value. The skeleton above does this correctly.

### The Softc: Per-Device State

A USB驱动程序's softc is a plain C structure stored as the New总线 驱动程序 data for each 附加ed 设备. It is allocated automatically by the 框架 based on the size declared in the 驱动程序 描述符, and it is the place where all per-设备 mutable state lives. For our running example, the softc looks like this:

```c
struct myfirst_usb_softc {
    struct usb_device *sc_udev;
    struct mtx         sc_mtx;
    struct usb_xfer   *sc_xfer[MYFIRST_USB_N_XFER];
    struct cdev       *sc_dev;
    uint8_t            sc_iface_index;
    uint8_t            sc_flags;
#define MYFIRST_USB_FLAG_OPEN       0x01
#define MYFIRST_USB_FLAG_DETACHING  0x02
};
```

Let us walk through each member.

`sc_udev` is the opaque pointer the USB 框架 uses to identify the 设备. Every USB call that acts on the 设备 takes this pointer.

`sc_mtx` is the per-设备 互斥锁 that protects the softc itself and any shared state the 驱动程序 cares about. The 互斥锁 must be acquired before touching any field that a 传输回调 might also touch, and the 传输回调 always runs with this 互斥锁 held (the 框架 handles the locking for you when it invokes the 回调).

`sc_xfer[]` is an array of transfer objects, one per channel the 驱动程序 uses. Its size is a compile-time constant. Section 3 will discuss how each entry in this array is set up by the configuration array passed to `usbd_transfer_setup`.

`sc_dev` is the 字符设备 entry, if the 驱动程序 exposes a user-facing node. For 驱动程序 that do not expose a `/dev` node (some 驱动程序 only export data through `sysctl` or `devctl` events), this field can be omitted.

`sc_iface_index` records which 接口 on the USB 设备 this 驱动程序 附加ed to. It is used by transfer setup and, in multi-接口 驱动程序, as a discriminator in logging.

`sc_flags` is a bit vector for 驱动程序-private state. Two flags are declared here: `MYFIRST_USB_FLAG_OPEN` is set while a userland process holds the 设备 open, and `MYFIRST_USB_FLAG_DETACHING` is set at the start of `分离` so that any concurrent I/O path can see that it must abort quickly. This is an application of a standard pattern: setting a flag under the 互斥锁 at the start of 分离, so anyone else who wakes up sees it and bails out.

Real 驱动程序 often have many more fields: per-transfer 缓冲区, request queues, 回调-to-回调 state machines, timers, and so on. You add to the softc as the 驱动程序 grows. The guiding principle is that any state that persists between function calls, and is not global to the module, belongs in the softc.

### The `分离` Method

When a 设备 is unplugged, when the module is unloaded, or when userspace uses `devctl 分离`, the 框架 calls the 驱动程序's `分离` method. The 驱动程序's job is to release every resource it allocated in `附加`, cancel any in-flight work, make sure no 回调 is running, and return zero. If `分离` returns an error, the 框架 treats the 设备 as still 附加ed, which can create problems if the hardware has already physically vanished. Most 驱动程序 return zero unconditionally, or only return an error in very specific "设备 总线y" cases where the 驱动程序 implements its own reference counting for userspace handles.

The 分离 method for our running example is the symmetric cleanup of the 附加 method:

```c
static int
myfirst_usb_detach(device_t dev)
{
    struct myfirst_usb_softc *sc = device_get_softc(dev);

    mtx_lock(&sc->sc_mtx);
    sc->sc_flags |= MYFIRST_USB_FLAG_DETACHING;
    mtx_unlock(&sc->sc_mtx);

    if (sc->sc_dev != NULL) {
        destroy_dev(sc->sc_dev);
        sc->sc_dev = NULL;
    }

    usbd_transfer_unsetup(sc->sc_xfer, MYFIRST_USB_N_XFER);

    mtx_destroy(&sc->sc_mtx);

    return (0);
}
```

The first 块 sets the 分离ing flag under the 互斥锁. If another thread is about to take the 互斥锁 and start a new transfer, it will see the flag and refuse. The `destroy_dev` call removes the `/dev` entry; after it returns, no new open calls can arrive. The `usbd_transfer_unsetup` call cancels any in-flight transfers and waits for their 回调 to complete; after it returns, no 传输回调 can still be running. With no new openers and no running 回调, it is safe to destroy the 互斥锁.

There is a subtlety here that new 内核 programmers sometimes stumble over: the order matters. Destroying the `/dev` entry before unwinding the transfers ensures that no new user operation can start, but it does not stop the transfers that were already running when 分离 was called. That is `usbd_transfer_unsetup`'s job. Both steps are necessary, and the order (cdev first, then transfers, then 互斥锁) is the right one because each later step depends on no new work arriving during it.

One further point about 分离 and concurrency. The 框架 guarantees that no 探测, 附加, or 分离 runs concurrently with another 探测, 附加, or 分离 on the same 设备. But 传输回调 run on their own path, and they can be in progress at the exact moment 分离 is called. The combination of the 分离ing flag and `usbd_transfer_unsetup` is what makes this safe. If you add new resources to your 驱动程序, you must add symmetric cleanup that accounts for this concurrency.

### Registration Macros

Every FreeBSD 驱动程序 needs to 注册 itself with the 内核 so that the 内核 knows when to call its 探测, 附加, and 分离 routines. USB驱动程序 use a small set of macros that bind everything together into a 内核模块. The macros go at the bottom of the 驱动程序 file and look intimidating at first but are entirely mechanical once you know what each line does.

```c
static device_method_t myfirst_usb_methods[] = {
    DEVMETHOD(device_probe,  myfirst_usb_probe),
    DEVMETHOD(device_attach, myfirst_usb_attach),
    DEVMETHOD(device_detach, myfirst_usb_detach),
    DEVMETHOD_END
};

static driver_t myfirst_usb_driver = {
    .name    = "myfirst_usb",
    .methods = myfirst_usb_methods,
    .size    = sizeof(struct myfirst_usb_softc),
};

DRIVER_MODULE(myfirst_usb, uhub, myfirst_usb_driver, NULL, NULL);
MODULE_DEPEND(myfirst_usb, usb, 1, 1, 1);
MODULE_VERSION(myfirst_usb, 1);
USB_PNP_HOST_INFO(myfirst_usb_devs);
```

Let us read each 块.

The `设备_method_t` array lists the methods the 驱动程序 supplies. For a USB驱动程序 that does not implement extra new总线 children, the three entries shown are sufficient: 探测, 附加, 分离. More complex 驱动程序 might add `设备_suspend`, `设备_resume`, or `设备_shutdown`, but for the vast majority of USB驱动程序 the three basic entries are all that is needed. `DEVMETHOD_END` terminates the array; the 框架 requires it.

The `驱动程序_t` structure binds the methods array to a human-readable name and declares the softc size. The name is used in 内核 logs and by `devctl`. The softc size tells New总线 how much memory to allocate per 设备.

The `DRIVER_MODULE` macro 寄存器 the 驱动程序 with the 内核. The arguments are, in order: the module name, the parent 总线 name (always `uhub` for USB class 驱动程序), the 驱动程序 structure, and two optional hooks for events. The event hooks are rarely needed and are usually `NULL`.

The `MODULE_DEPEND` macro declares that this module needs `usb` to be loaded first. The three numbers are the minimum, preferred, and maximum compatible versions of the `usb` module. For most 驱动程序, `1, 1, 1` is correct: the USB 框架 has versioned its 接口 at 1 for a long time, and it would be unusual to require anything else.

The `MODULE_VERSION` macro declares this module's own version number. Other modules that want to depend on `myfirst_usb` would reference the number you declare here.

The `USB_PNP_HOST_INFO` macro is the last piece. It exports the 匹配表 into a format the `devd(8)` daemon can read, so that when a matching USB 设备 is plugged in, userspace can auto-load the module. This macro is a relatively recent addition to FreeBSD; older 驱动程序 may not have it. Including it is strongly recommended for any 驱动程序 that wants to participate in FreeBSD's USB plug-and-play system.

Together, these five declarations turn your 驱动程序 file into a loadable 内核模块. Once the file is compiled with a `Makefile` that uses `bsd.kmod.mk`, running `kldload myfirst_usb.ko` will bind the 驱动程序 to the 内核, and any matching 设备 plugged in afterwards will trigger your 探测 and 附加 routines.

### The Hot-Plug Lifecycle, Revisited in Code

Section 1 introduced the 热插拔 lifecycle at the level of mental model: a 设备 appears, the 框架 enumerates it, your 驱动程序 附加, userland interacts with it, the 设备 disappears, the 框架 calls 分离, your 驱动程序 cleans up. With the code in front of you, that narrative now has a concrete sequence:

1. The user plugs in a matching 设备.
2. The USB 框架 enumerates the 设备, reads all its 描述符, and decides which 接口 to offer to which 驱动程序.
3. For each 接口 that matches your 驱动程序's 匹配表, the 框架 creates a New总线 child and calls your `探测` method.
4. Your `探测` method returns zero.
5. The 框架 calls your `附加` method. You initialise the softc, set up transfers, create the `/dev` node, and return zero.
6. Userland opens the `/dev` node and begins issuing I/O. The 传输回调 from Section 3 start running.
7. The user unplugs the 设备.
8. The 框架 calls your `分离` method. You set the 分离ing flag, destroy the `/dev` node, call `usbd_transfer_unsetup` to cancel all in-flight transfers and wait for 回调 to finish, destroy the 互斥锁, and return zero.
9. The 框架 deallocates the softc and removes the New总线 child.

At every step, the 框架 handles the parts you do not have to write yourself. Your responsibility is narrow: react correctly to 探测, 附加, and 分离, and run 传输回调 that respect the state machine. The machinery around you handles enumeration, 总线 arbitration, transfer scheduling, hub routing, and the dozens of corner cases that USB layer imposes.

The lifecycle has one more subtle quirk that is worth naming. Between the user unplugging the 设备 and the 框架 calling `分离`, there is a brief window in which any in-flight transfer sees a special error: `USB_ERR_CANCELLED`. The transfer 框架 itself generates this error when it tears down the transfers in response to the disconnect. Section 3 will explain how to handle this error in the 回调 state machine. For now, know that it exists and that it is the 驱动程序's normal signal that the 设备 is going away.

### 总结 Section 2

Section 2 has given you a complete USB驱动程序 skeleton. The skeleton does not yet move data; that is Section 3's topic. But every other part of the 驱动程序 is in place: the 匹配表, the 探测 method, the 附加 method, the softc, the 分离 method, and the registration macros. You have seen how the USB 框架 routes a newly enumerated 设备 through your 探测 routine, how your 附加 routine takes ownership and sets up state, how the 驱动程序 integrates with New总线 through `设备_get_ivars` and `设备_get_softc`, and how the 分离 routine walks the allocation steps in reverse to leave the system clean.

Two themes from 第25章 have extended naturally into USB territory. First, the labelled-goto cleanup chain. Every resource you acquire has its own label, and every failure path falls through exactly the right sequence of teardown calls. When you compare `myfirst_usb_附加` above with the 附加 functions in `uled.c`, `ugold.c`, or `uftdi.c`, you will see the same pattern repeated. Second, the discipline of single-source-of-truth state in the softc. Every field has one owner, one lifecycle, and one clear place where it is initialised and destroyed. These habits are what make a 驱动程序 readable, portable, and maintainable.

Section 3 will now give this skeleton a voice. Transfer channels will be declared in a configuration array. The USB 框架 will allocate the underlying 缓冲区 and schedule the transactions. A 回调 will wake up each time a transfer completes or needs more data, and it will use a three-state state machine to decide what to do. The same discipline you just learned will apply, but the new concern is the data pipeline itself: how bytes move between the 驱动程序 and the 设备.

### Reading `uled.c` As a Complete Example

Before moving into transfers, it is worth pausing to read the canonical small-驱动程序 example end to end. The file `/usr/src/sys/dev/usb/misc/uled.c` is approximately three hundred lines of C that implements a 驱动程序 for the Dream Cheeky and Riso Kagaku USB webmail notifier LEDs: small USB gadgets with three coloured LEDs that a host program can light up. The 驱动程序 is short enough to hold in your head, self-contained, and it exercises every pattern we have discussed.

When you open the file, the first 块 you encounter is the standard set of header includes. A USB驱动程序 pulls in headers from several layers: `sys/param.h`, `sys/systm.h`, `sys/总线.h` for the fundamentals; `sys/module.h` for `MODULE_VERSION` and `MODULE_DEPEND`; the USB headers under `dev/usb/` for the 框架; and `usbdevs.h` for the symbolic vendor and product constants. Note that `usbdevs.h` is not a hand-maintained header: it is build-generated from the text file `/usr/src/sys/dev/usb/usbdevs` when the 内核 or module is compiled, so the constants it exposes reflect whatever entries the in-tree `usbdevs` file currently lists. `uled.c` also pulls in `sys/conf.h` and friends because it creates a 字符设备.

The second 块 is the softc declaration. `uled` keeps its state in a structure that has the 设备 pointer, a 互斥锁, an array of two transfer pointers (one for control, one for data), a 字符设备 pointer, a 回调 state pointer, and a small "color" byte that records the current LED colour. The softc is straightforward: every field is private, every allocation has one place where it is made and one place where it is freed.

The third 块 is the 匹配表. `uled` supports two vendors (Dream Cheeky and Riso Kagaku) with one 产品ID each. The `USB_VPI` macro fills in the flag byte for a vendor-plus-product match. The table is two entries, flat and simple.

The fourth 块 is the transfer configuration array. `uled` declares two channels: a control-out channel used to send SET_REPORT requests to the 设备 (which is how the LED colour is actually programmed), and an interrupt-in channel that reads status 数据包 from the LED. The control channel has `type = UE_CONTROL` and a 缓冲区 size big enough to hold the setup 数据包 plus the payload. The interrupt channel has `type = UE_INTERRUPT`, `direction = UE_DIR_IN`, and a 缓冲区 size that matches the LED's report size.

The fifth 块 is the 回调 functions. The control 回调 follows the three-state machine you saw in 第3节： in `USB_ST_SETUP`, it constructs a setup 数据包 and an eight-byte HID report payload, submits the transfer, and returns. In `USB_ST_TRANSFERRED`, it wakes any userland writer that was waiting for the colour change to complete. In the default case (errors), it handles cancellation gracefully and retries on other errors.

The interrupt 回调 is similar but without the setup-数据包 complication. It reads an eight-byte status report, checks whether it indicates a button press (the Riso Kagaku 设备 have an optional button), and rearms.

The sixth 块 is the character-设备 methods. `uled` exposes a `/dev/uled0` entry that accepts `write(2)` calls with a three-byte payload (red, green, blue). The `d_write` handler copies the three bytes into the softc, starts the 控制传输, and returns. When the transfer completes, the colour is actually programmed. The `d_read` handler is not implemented (LEDs do not have meaningful state to read), so reads return zero.

The seventh 块 is the New总线 methods: 探测, 附加, 分离. The 探测 uses `usbd_lookup_id_by_uaa` exactly as shown in Section 2. The 附加 calls `设备_set_usb_desc`, initialises the 互斥锁, calls `usbd_transfer_setup` with the configuration array, and creates the 字符设备. The 分离 runs these in reverse.

The eighth 块 is the registration macros. `DRIVER_MODULE(uled, uhub, ...)`, `MODULE_DEPEND(uled, usb, 1, 1, 1)`, `MODULE_VERSION(uled, 1)`, and `USB_PNP_HOST_INFO(uled_devs)`. Exactly the sequence you learned.

Reading through `uled.c` with the Section 2 vocabulary in hand, the whole file legibly maps onto the patterns you now understand. Every structural choice the 驱动程序 makes has a name. Every line of code is an instance of a general pattern. This is the kind of clarity that makes FreeBSD 驱动程序 readable.

Before continuing to Section 3, we recommend you actually open `uled.c` now and read it. Even if some lines are still obscure, the overall structure will match the mental model you have built. The details will make more sense as you progress through the rest of the chapter, and revisiting this file after finishing the chapter is an excellent way to consolidate the material.

## 第3节： Performing USB Data Transfers

### The Transfer Configuration Array

A USB驱动程序 declares its transfers up front, at compile time, through a small array of `struct usb_config` entries. Each entry describes one transfer channel: its type (control, bulk, interrupt, or isochronous), its direction (in or out), which 端点 it targets, how big its 缓冲区 is, which flags apply, and which 回调 function to invoke when the transfer completes. The 框架 reads this array once, during `附加`, when the 驱动程序 calls `usbd_transfer_setup`. From that point on, each channel behaves like a small state machine that the 驱动程序 drives through its 回调.

The configuration array is declarative. You are not programming the sequence of hardware operations; you are telling the 框架 what channels your 驱动程序 will use, and the 框架 builds the infrastructure to support them. This is an effective abstraction, and it is one of the reasons USB驱动程序 in FreeBSD are usually much shorter than equivalent 驱动程序 for 总线es like PCI that demand direct 注册 manipulation.

For our running example, we will declare three channels. A bulk-IN channel for reading data from the 设备, a bulk-OUT channel for writing data to the 设备, and an interrupt-IN channel for receiving asynchronous status events. A real 驱动程序 for a serial adapter or an LED notifier might use one or two of these; we use three to show the pattern applied to different transfer types.

```c
enum {
    MYFIRST_USB_BULK_DT_RD,
    MYFIRST_USB_BULK_DT_WR,
    MYFIRST_USB_INTR_DT_RD,
    MYFIRST_USB_N_XFER,
};

static const struct usb_config myfirst_usb_config[MYFIRST_USB_N_XFER] = {
    [MYFIRST_USB_BULK_DT_RD] = {
        .type      = UE_BULK,
        .endpoint  = UE_ADDR_ANY,
        .direction = UE_DIR_IN,
        .bufsize   = 512,
        .flags     = { .pipe_bof = 1, .short_xfer_ok = 1 },
        .callback  = &myfirst_usb_bulk_read_callback,
    },
    [MYFIRST_USB_BULK_DT_WR] = {
        .type      = UE_BULK,
        .endpoint  = UE_ADDR_ANY,
        .direction = UE_DIR_OUT,
        .bufsize   = 512,
        .flags     = { .pipe_bof = 1, .force_short_xfer = 0 },
        .callback  = &myfirst_usb_bulk_write_callback,
        .timeout   = 5000,
    },
    [MYFIRST_USB_INTR_DT_RD] = {
        .type      = UE_INTERRUPT,
        .endpoint  = UE_ADDR_ANY,
        .direction = UE_DIR_IN,
        .bufsize   = 16,
        .flags     = { .pipe_bof = 1, .short_xfer_ok = 1 },
        .callback  = &myfirst_usb_intr_callback,
    },
};
```

The enumeration at the top gives each channel a name and defines `MYFIRST_USB_N_XFER` as the total count. This is a common idiom; it keeps the channels symbolically accessible and makes it easy to add a new channel later. `MYFIRST_USB_N_XFER` is what you pass to `usbd_transfer_setup`, to `usbd_transfer_unsetup`, and to the softc's `sc_xfer[]` array declaration.

The array itself uses designated initialisers, which keeps the assignment of each channel to its enumeration index explicit. Let us walk through the fields.

`type` is one of `UE_CONTROL`, `UE_BULK`, `UE_INTERRUPT`, or `UE_ISOCHRONOUS`, from `/usr/src/sys/dev/usb/usb.h`. It has to match the 端点's type as declared in the USB 描述符. If you say `UE_BULK` but the 设备 has an interrupt 端点, `usbd_transfer_setup` will fail.

`端点` identifies the 端点 number, but in most 驱动程序 the special value `UE_ADDR_ANY` is used, which tells the 框架 to pick any 端点 whose type and direction match. This works because most USB 接口 have only one 端点 of each (type, direction) pair, so "any" is unambiguous. A 设备 with multiple bulk-in 端点 would require explicit 端点 addresses.

`direction` is `UE_DIR_IN` or `UE_DIR_OUT`. Again, this must match the 描述符.

`bufsize` is the size of the 缓冲区 the 框架 allocates for this channel. For 批量传输, 512 bytes is a common choice because that is the maximum 数据包 size for high-speed bulk 端点, so a single 512-byte 缓冲区 can hold exactly one 数据包. Larger 缓冲区 are supported, but for most purposes 512 or a small multiple is correct. For interrupt 端点, the 缓冲区 can be smaller because interrupt 数据包 are typically eight, sixteen, or sixty-four bytes.

`flags` is a bitfield struct (each flag is a one-bit integer). The flags affect how the 框架 handles short transfers, stalls, timeouts, and pipe behaviour.

- `pipe_bof` (pipe 块ed on failure): if the transfer fails, 块 further transfers on the same pipe until the 驱动程序 explicitly restarts it. This is usually set for both read and write 端点.
- `short_xfer_ok`: for incoming transfers, treat a transfer that completed with less data than requested as success rather than error. Setting this is what allows a bulk-IN channel to read responses of variable length from a 设备.
- `force_short_xfer`: for outgoing transfers, finish the transfer with a short 数据包 even when the data is aligned to a full 数据包 boundary. This is used by some protocols to signal the end of a message.
- Several other flags control more advanced behaviour; for most 驱动程序, `pipe_bof` plus `short_xfer_ok` (on reads) plus possibly `force_short_xfer` (on writes, protocol-dependent) is all that is needed.

`回调` is the function the 框架 calls whenever this channel needs attention. The 回调 is a `usb_回调_t`, which takes a pointer to the `struct usb_xfer` and returns void. All of the channel's state-machine logic lives inside the 回调.

`timeout` (in milliseconds) sets an upper bound on how long a transfer can wait before being forcibly completed with an error. Setting a timeout is useful for write channels, because it prevents a hung 设备 from stalling the 驱动程序 indefinitely. For read channels, leaving the timeout at zero (meaning "no timeout") is common, because reads are often expected to 块 waiting for the 设备 to have something to say.

This array, combined with `usbd_transfer_setup`, is all the 驱动程序 needs to declare its data pipeline. The 框架 allocates the underlying DMA 缓冲区, sets up the scheduling, and watches the pipes. The 驱动程序 never has to call into a 注册 or schedule a transaction by hand. It just writes 回调.

### Setting Up and Tearing Down Transfers

In the `附加` method shown in Section 2, the call to `usbd_transfer_setup` creates the channels from the configuration array:

```c
error = usbd_transfer_setup(uaa->device, &sc->sc_iface_index,
    sc->sc_xfer, myfirst_usb_config, MYFIRST_USB_N_XFER,
    sc, &sc->sc_mtx);
```

The arguments are, in order: the USB 设备 pointer, a pointer to the 接口 index (the 框架 can update it in certain multi-接口 scenarios), the destination array for the created transfer objects, the configuration array, the number of channels, the softc pointer (which is passed into 回调 via `usbd_xfer_softc`), and the 互斥锁 the 框架 will hold around each 回调.

If this call succeeds, `sc->sc_xfer[]` is populated with pointers to `struct usb_xfer` objects. Each object encapsulates a channel's state. From this point, the 驱动程序 can submit a transfer on a channel with `usbd_transfer_submit(sc->sc_xfer[i])`, and the 框架 will, in the fullness of time, call the corresponding 回调.

The symmetric teardown, shown in the `分离` method, is `usbd_transfer_unsetup`:

```c
usbd_transfer_unsetup(sc->sc_xfer, MYFIRST_USB_N_XFER);
```

This call does three things, in order. It cancels any in-flight transfer on each channel. It waits for the corresponding 回调 to run with `USB_ST_ERROR` or `USB_ST_CANCELLED`, so the 驱动程序 has a chance to clean up any per-transfer state. It frees the 框架's internal state for the channel. After `usbd_transfer_unsetup` returns, the `sc_xfer[]` entries are no longer valid, and the associated 回调 will not be invoked again.

This is the piece of machinery that makes 分离 safe in the presence of ongoing I/O. You do not need to implement your own "wait for outstanding transfers" logic. The 框架 provides it, atomically, through this single call.

### The Callback State Machine

Every 传输回调 follows the same three-state state machine. When the 框架 invokes the 回调, you ask `USB_GET_STATE(xfer)` for the current state, and then you handle it. The three possible states are declared in `/usr/src/sys/dev/usb/usbdi.h`:

- `USB_ST_SETUP`: the 框架 is ready to submit a new transfer on this channel. You should prepare the transfer (set its length, copy data into its 缓冲区, and so on) and call `usbd_transfer_submit`. If you have no work for this channel right now, simply return; the 框架 will leave the channel idle until something else triggers a submit.
- `USB_ST_TRANSFERRED`: the most recent transfer completed successfully. You should read out the results (copy received data out, decide what to do next) and either return (if the channel should go idle) or fall through to `USB_ST_SETUP` to start another transfer.
- `USB_ST_ERROR`: the most recent transfer failed. You should inspect `usbd_xfer_get_error(xfer)` to see why, handle the error (for most errors, you fall through to `USB_ST_SETUP` to retry after a short delay; for stalls, you issue a clear-stall), and decide whether to continue.

The typical shape of a bulk-read 回调 looks like this:

```c
static void
myfirst_usb_bulk_read_callback(struct usb_xfer *xfer, usb_error_t error)
{
    struct myfirst_usb_softc *sc = usbd_xfer_softc(xfer);
    struct usb_page_cache *pc;
    int actlen;

    usbd_xfer_status(xfer, &actlen, NULL, NULL, NULL);

    switch (USB_GET_STATE(xfer)) {
    case USB_ST_TRANSFERRED:
        pc = usbd_xfer_get_frame(xfer, 0);
        /*
         * Copy actlen bytes from pc into the driver's receive buffer.
         * This is where you hand the data to userland, to a queue,
         * or to another callback.
         */
        myfirst_usb_deliver_received(sc, pc, actlen);
        /* FALLTHROUGH */
    case USB_ST_SETUP:
tr_setup:
        /*
         * Arm a read for 512 bytes.  The actual amount received may
         * be less, because we enabled short_xfer_ok in the config.
         */
        usbd_xfer_set_frame_len(xfer, 0, usbd_xfer_max_len(xfer));
        usbd_transfer_submit(xfer);
        break;

    default:  /* USB_ST_ERROR */
        if (error == USB_ERR_CANCELLED) {
            /* The device is going away.  Do nothing. */
            break;
        }
        if (error == USB_ERR_STALLED) {
            /* Arm a clear-stall on the control pipe; the framework
             * will call us back in USB_ST_SETUP after the clear
             * completes. */
            usbd_xfer_set_stall(xfer);
        }
        goto tr_setup;
    }
}
```

Let us walk through every piece.

The first line retrieves the softc pointer from the transfer object. This is how the 回调 gets at the per-设备 state. It works because the softc was passed to `usbd_transfer_setup`, which stored it inside the transfer object.

The call to `usbd_xfer_status` fills in `actlen`, the number of bytes actually transferred on 帧 zero. For a read, this is how much data arrived. For a write, it is how much data was sent. The other three parameters (which this example does not use) give the total transfer length, the timeout, and a status flags pointer; most 回调 only need `actlen`.

The switch on `USB_GET_STATE(xfer)` is the state machine. In `USB_ST_TRANSFERRED`, the 回调 copies the received data out of the USB 帧 into the 驱动程序's own 缓冲区. The helper function `myfirst_usb_deliver_received` (which you would write) could push the data onto a queue, wake a sleeping read() on the `/dev` node, or feed a higher-level protocol parser.

The `FALLTHROUGH` after processing the transferred data takes the 回调 into the `USB_ST_SETUP` branch. This is the idiomatic pattern for channels that run continuously: every time a read finishes, immediately start another read. If the 驱动程序 wanted to stop reading after one transfer (say, a one-shot control request), it would `return;` at the end of `USB_ST_TRANSFERRED` instead of falling through.

In `USB_ST_SETUP`, `usbd_xfer_set_帧_len` sets the length of 帧 zero to the maximum the channel can handle, and `usbd_transfer_submit` hands the transfer to the 框架. The 框架 will start the actual hardware operation and, when complete, call the 回调 again with either `USB_ST_TRANSFERRED` or `USB_ST_ERROR`.

The `default` case is where error handling happens. Two errors get special treatment. `USB_ERR_CANCELLED` is the signal that the transfer is being torn down, typically because the 设备 was unplugged or `usbd_transfer_unsetup` was called. The 回调 must not resubmit the transfer in this case; if it did, it could race with the teardown and potentially touch memory that is about to be freed. Breaking out of the switch without calling `usbd_transfer_submit` is the correct behaviour.

`USB_ERR_STALLED` is the signal that the 端点 returned a STALL handshake, meaning the 设备 is refusing to accept more data until the host clears the stall. The call to `usbd_xfer_set_stall` schedules a clear-stall operation on the control 端点. After the clear-stall completes, the 框架 will call the 回调 again with `USB_ST_SETUP`, at which point the 驱动程序 can reissue the transfer. This logic is built into the 框架 so that every 驱动程序 gets the same correct behaviour with minimal code.

For any other error, the 回调 falls through to `tr_setup` and attempts to resubmit the transfer. This is a simple retry policy. A more sophisticated 驱动程序 might count consecutive errors and give up after a threshold, or it might escalate by calling `usbd_transfer_unsetup` on itself. For many 驱动程序, the default retry loop is sufficient.

### The Write Callback

The write 回调 has the same shape but its `USB_ST_SETUP` branch is more interesting, because it has to decide whether there is any data to write:

```c
static void
myfirst_usb_bulk_write_callback(struct usb_xfer *xfer, usb_error_t error)
{
    struct myfirst_usb_softc *sc = usbd_xfer_softc(xfer);
    struct usb_page_cache *pc;
    int actlen;
    unsigned int len;

    usbd_xfer_status(xfer, &actlen, NULL, NULL, NULL);

    switch (USB_GET_STATE(xfer)) {
    case USB_ST_TRANSFERRED:
        /* A previous write finished.  Wake any blocked writer. */
        wakeup(&sc->sc_xfer[MYFIRST_USB_BULK_DT_WR]);
        /* FALLTHROUGH */
    case USB_ST_SETUP:
tr_setup:
        len = myfirst_usb_dequeue_write(sc);
        if (len == 0) {
            /* Nothing to send right now.  Leave the channel idle. */
            break;
        }
        pc = usbd_xfer_get_frame(xfer, 0);
        myfirst_usb_copy_write_data(sc, pc, len);
        usbd_xfer_set_frame_len(xfer, 0, len);
        usbd_transfer_submit(xfer);
        break;

    default:  /* USB_ST_ERROR */
        if (error == USB_ERR_CANCELLED)
            break;
        if (error == USB_ERR_STALLED)
            usbd_xfer_set_stall(xfer);
        goto tr_setup;
    }
}
```

The main change is the logic at `tr_setup`. For a read, the 驱动程序 always wants another read armed, so the 回调 just sets the 帧 length and submits. For a write, the 驱动程序 only submits if there is something to send. The helper `myfirst_usb_dequeue_write` returns the number of bytes pulled from an internal transmit queue; if zero, the 回调 breaks out of the switch without submitting anything, which leaves the channel idle. When userspace later writes more data into the 设备, the 驱动程序 code that handles the `write()` system call queues the bytes and explicitly calls `usbd_transfer_start(sc->sc_xfer[MYFIRST_USB_BULK_DT_WR])`. That call fires an `USB_ST_SETUP` invocation of the 回调, which now finds data in the queue and submits it.

This interaction between the userspace I/O path and the transfer state machine is the heart of an interactive USB驱动程序. Reads are self-driving: once armed, they rearm themselves on every completion. Writes are demand-driven: they submit only when data is available and go idle otherwise. Both patterns run inside the same three-state machine; the difference is only in what happens at `USB_ST_SETUP`.

### Control Transfers

Control transfers do not typically run on continuously-armed channels; they are usually issued one-shot, either synchronously from a system-call handler or as a one-shot 回调 triggered by some 驱动程序 event. The `struct usb_config` for a control channel has `type = UE_CONTROL` and otherwise looks similar to the bulk and interrupt configurations. The 缓冲区 size must be at least eight bytes to hold the setup 数据包, and the 回调 deals with two 帧: 帧 zero is the setup 数据包, and 帧 one is the optional data phase.

The typical one-shot use is to issue a vendor-specific request at 驱动程序-load time. The FTDI 串行驱动程序, for example, uses 控制传输 to set the 波特率 and line parameters every time the user configures the 串行端口. Because the control 回调 is scheduled by the 框架 just like any other 传输回调, the code pattern is identical. What differs is the construction of the setup 数据包 in the `USB_ST_SETUP` branch.

For a control-read transfer, the code looks something like this:

```c
case USB_ST_SETUP: {
    struct usb_device_request req;
    req.bmRequestType = UT_READ_VENDOR_DEVICE;
    req.bRequest      = MY_VENDOR_GET_STATUS;
    USETW(req.wValue,  0);
    USETW(req.wIndex,  0);
    USETW(req.wLength, sizeof(sc->sc_status));

    pc = usbd_xfer_get_frame(xfer, 0);
    usbd_copy_in(pc, 0, &req, sizeof(req));

    usbd_xfer_set_frame_len(xfer, 0, sizeof(req));
    usbd_xfer_set_frame_len(xfer, 1, sizeof(sc->sc_status));
    usbd_xfer_set_frames(xfer, 2);
    usbd_transfer_submit(xfer);
    break;
}
```

The `USETW` macro stores a sixteen-bit value in the request structure in the little-endian byte order USB requires. The `usbd_copy_in` helper copies from a 内核 缓冲区 into a USB 帧. The `usbd_xfer_set_帧_len` and `usbd_xfer_set_帧` calls tell the 框架 how many 帧 the transfer spans and how long each is. For a control-read, 帧 zero is the setup 数据包 (eight bytes) and 帧 one is the data phase; the 框架 transparently handles the status phase at the end.

In the `USB_ST_TRANSFERRED` branch, the 驱动程序 reads the response out of 帧 one:

```c
case USB_ST_TRANSFERRED:
    pc = usbd_xfer_get_frame(xfer, 1);
    usbd_copy_out(pc, 0, &sc->sc_status, sizeof(sc->sc_status));
    /* sc->sc_status now holds the device's response. */
    break;
```

Control transfers are the right tool for configuration operations where latency and bandwidth do not matter but correctness and sequencing do. They are the wrong tool for streaming data; use bulk or 中断传输 for that.

### Interrupt Transfers

Interrupt transfers are conceptually the simplest of the four types. An interrupt-IN channel runs a continuous state machine that polls a single 端点 at regular intervals. Each time a 数据包 arrives from the 设备, the 回调 wakes up with `USB_ST_TRANSFERRED`. The 驱动程序 reads the 数据包, processes it (often by delivering it to userland), and falls through to rearm.

The 回调 for our interrupt channel is nearly identical to the bulk-read 回调:

```c
static void
myfirst_usb_intr_callback(struct usb_xfer *xfer, usb_error_t error)
{
    struct myfirst_usb_softc *sc = usbd_xfer_softc(xfer);
    struct usb_page_cache *pc;
    int actlen;

    usbd_xfer_status(xfer, &actlen, NULL, NULL, NULL);

    switch (USB_GET_STATE(xfer)) {
    case USB_ST_TRANSFERRED:
        pc = usbd_xfer_get_frame(xfer, 0);
        myfirst_usb_handle_interrupt(sc, pc, actlen);
        /* FALLTHROUGH */
    case USB_ST_SETUP:
tr_setup:
        usbd_xfer_set_frame_len(xfer, 0, usbd_xfer_max_len(xfer));
        usbd_transfer_submit(xfer);
        break;

    default:
        if (error == USB_ERR_CANCELLED)
            break;
        if (error == USB_ERR_STALLED)
            usbd_xfer_set_stall(xfer);
        goto tr_setup;
    }
}
```

The only meaningful difference from the bulk-read 回调 is that the 缓冲区 is smaller (interrupt 端点 数据包 are typically eight to sixty-four bytes) and the semantics of the data are usually "status update" rather than "stream payload." A USB HID 设备, for example, sends a sixty-four-byte report every few milliseconds describing key presses and mouse motions; an interrupt-IN channel polled continuously in this pattern is how the 内核 receives those reports.

Interrupt-OUT channels work the same way but in reverse: the 回调 has to decide whether to send something at each `USB_ST_SETUP`, analogous to the bulk-write pattern.

### Frame-Level Operations: What the Framework Gives You

USB transfers are composed of 帧. A 批量传输 with a large 缓冲区 might be broken into multiple 数据包 by the hardware; the 框架 hides that detail and presents the transfer as a single operation. A 控制传输, on the other hand, has an explicit 帧 structure (setup, data, status). An 等时传输 has one 帧 per scheduled 数据包. The 框架 exposes this structure through a small number of helper functions:

- `usbd_xfer_max_len(xfer)` returns the largest total length the channel can transfer in a single submit.
- `usbd_xfer_set_帧_len(xfer, 帧, len)` sets the length of a specific 帧.
- `usbd_xfer_set_帧(xfer, n)` sets the total number of 帧 in the transfer.
- `usbd_xfer_get_帧(xfer, 帧)` returns a page-cache pointer for a specific 帧, which is what you pass to `usbd_copy_in` and `usbd_copy_out`.
- `usbd_xfer_帧_len(xfer, 帧)` returns how many bytes were actually transferred in a given 帧 (for completions).
- `usbd_xfer_max_帧len(xfer)` returns the maximum per-帧 length for the channel.

For bulk and 中断传输, the vast majority of 驱动程序 only touch 帧 zero. For 控制传输, they touch 帧 zero and one. For 等时传输 (which we will not cover in this chapter), they loop over many 帧. The point is that the 框架 gives you complete control over the per-帧 data layout while hiding the hardware details that would otherwise make transfer scheduling a nightmare.

### The `usbd_copy_in` and `usbd_copy_out` Helpers

USB 缓冲区 are not plain C 缓冲区. They are allocated by the 框架 in a way that is addressable by the 主机控制器 hardware, which means they often live in DMA-accessible memory pages with platform-specific alignment requirements. The 框架 wraps these 缓冲区 in an opaque `struct usb_page_cache` object, and the 驱动程序 accesses them through two helpers:

- `usbd_copy_in(pc, offset, src, len)` copies `len` bytes from the plain C 缓冲区 `src` into the 框架-managed 缓冲区 at `offset`.
- `usbd_copy_out(pc, offset, dst, len)` copies `len` bytes out of the 框架-managed 缓冲区 at `offset` into the plain C 缓冲区 `dst`.

You never dereference a `struct usb_page_cache *` directly. You never assume it points to a contiguous memory region. You always go through the helpers. This keeps the 驱动程序 portable across platforms with different DMA constraints, and it is the standard convention throughout `/usr/src/sys/dev/usb/`.

If your 驱动程序 needs to fill a USB 缓冲区 with data from a mbuf chain or from a userland pointer, there are dedicated helpers for that too: `usbd_copy_in_mbuf`, `usbd_copy_from_mbuf`, and the `uiomove` interaction is handled through `usbd_m_copy_in` and related routines. Search the USB 框架 source for the right helper; there is almost certainly one that matches your need.

### Starting, Stopping, and Querying Transfers

Beyond the three 回调, the 驱动程序 interacts with transfer channels through a small number of control functions. The important ones are:

- `usbd_transfer_start(xfer)`: ask the 框架 to schedule a 回调 invocation in the `USB_ST_SETUP` state, even if the channel has been idle. Used when new data becomes available for a write channel.
- `usbd_transfer_stop(xfer)`: stop the channel. Any in-flight transfer is cancelled and the 回调 is invoked with `USB_ST_ERROR` (with `USB_ERR_CANCELLED`). No new 回调 happen until the 驱动程序 calls `usbd_transfer_start` again.
- `usbd_transfer_pending(xfer)`: returns true if a transfer is currently outstanding. Useful for deciding whether to submit a new one or defer.
- `usbd_transfer_drain(xfer)`: 块 until any outstanding transfer completes and the channel is idle. Used in teardown paths that need to wait for in-flight I/O before continuing.

These functions are safe to call while holding the 驱动程序's 互斥锁, and in fact most of them require it. The 框架 documentation and the existing 驱动程序 code show the expected usage patterns; when in doubt, grep for the function name in `/usr/src/sys/dev/usb/` and read how existing 驱动程序 use it.

### A Worked Example: Echo-Loop over USB

To make the transfer mechanics concrete, consider a small end-to-end scenario. The 驱动程序 exposes a `/dev/myfirst_usb0` entry that accepts writes and returns reads. A user process writes a string to the 设备; the 驱动程序 sends those bytes to the USB 设备 through the bulk-OUT channel. The 设备 bounces the bytes back through its bulk-IN 端点; the 驱动程序 receives them and hands them to any process currently 块ed in a `read()` on the `/dev` node. This is a useful exercise because it exercises both directions of the bulk pipeline and because it has a simple, observable success criterion: the string that goes in is the string that comes out.

The 驱动程序 needs a small transmit queue and a small receive queue, both protected by the softc 互斥锁. When userspace writes, the `d_write` handler acquires the 互斥锁, copies the bytes into the transmit queue, and calls `usbd_transfer_start(sc->sc_xfer[MYFIRST_USB_BULK_DT_WR])`. When userspace reads, the `d_read` handler acquires the 互斥锁, checks the receive queue; if empty, it sleeps on a channel related to the queue. The write 回调, running under the 互斥锁, dequeues bytes and submits the transfer. The read 回调, also under the 互斥锁, enqueues received bytes and wakes any 块ed reader.

The complete flow from userspace `write("hi")` to userspace `read()` seeing "hi" involves three threads of execution interleaved through the state machines:

1. User thread runs `write()`. Driver enqueues "hi" on the TX queue. Driver calls `usbd_transfer_start`. User thread returns.
2. Framework schedules the TX 回调 with `USB_ST_SETUP`. Callback dequeues "hi", copies it into 帧 zero, sets 帧 length to 2, submits. Callback returns.
3. Hardware performs the bulk-OUT transaction. Device echoes "hi" on bulk-IN.
4. Framework schedules the RX 回调 with `USB_ST_TRANSFERRED` (because an earlier `USB_ST_SETUP` had armed a read). Callback reads "hi" from 帧 zero into the RX queue, wakes any 块ed reader, falls through to re-arm the read, submits. Callback returns.
5. User thread, if it was 块ed in `read()`, wakes up. The `d_read` handler copies "hi" out of the RX queue into userspace. User thread returns.

At each step, the 互斥锁 is held exactly where it needs to be, the state machine moves cleanly between `USB_ST_SETUP` and `USB_ST_TRANSFERRED`, and the 驱动程序 does not have to think about 数据包 boundaries, DMA mappings, or hardware scheduling. The 框架 handles all of that.

### Putting the Whole Echo-Loop Driver Together

To make the echo-loop description concrete, let us walk through a complete skeleton for `myfirst_usb`. What follows is not a copy of the real 驱动程序 files in `examples/`; it is a narrative presentation of how the pieces fit. The full code is in the examples directory.

The 驱动程序 has one C source file, `myfirst_usb.c`, and a small header `myfirst_usb.h`. The header declares the softc structure, the constants for the transfer enumeration, and the prototypes for internal helper functions. The source file contains the 匹配表, the transfer configuration array, the 回调 functions, the character-设备 methods, the New总线 探测/附加/分离, and the registration macros.

The softc is as we described earlier: a USB 设备 pointer, a 互斥锁, the transfer array, a 字符设备 pointer, an 接口 index, a flags byte, and two internal 环形缓冲区 for RX and TX queued data. Each 环形缓冲区 is a fixed-size array (say, 4096 bytes) plus head and tail indices, protected by the 互斥锁.

The 匹配表 contains one entry:

```c
static const STRUCT_USB_HOST_ID myfirst_usb_devs[] = {
    {USB_VPI(0x16c0, 0x05dc, 0)},
};
```

The 0x16c0/0x05dc VID/PID pair is the Van Oosting Technologies Incorporated / OBDEV generic test VID/PID, which is free to use for prototyping.

The transfer configuration array is the three-channel array from Section 3. The 回调 are the bulk-read, bulk-write, and interrupt-read patterns we walked through.

The bulk-read 回调's `USB_ST_TRANSFERRED` branch calls a helper:

```c
static void
myfirst_usb_rx_enqueue(struct myfirst_usb_softc *sc,
    struct usb_page_cache *pc, int len)
{
    int space;
    unsigned int tail;

    space = MYFIRST_USB_RX_BUFSIZE - sc->sc_rx_count;
    if (space < len)
        len = space;  /* drop the excess; a real driver might flow-control */

    tail = (sc->sc_rx_head + sc->sc_rx_count) & (MYFIRST_USB_RX_BUFSIZE - 1);
    if (tail + len > MYFIRST_USB_RX_BUFSIZE) {
        /* wrap-around copy in two pieces */
        usbd_copy_out(pc, 0, &sc->sc_rx_buf[tail], MYFIRST_USB_RX_BUFSIZE - tail);
        usbd_copy_out(pc, MYFIRST_USB_RX_BUFSIZE - tail,
            &sc->sc_rx_buf[0], len - (MYFIRST_USB_RX_BUFSIZE - tail));
    } else {
        usbd_copy_out(pc, 0, &sc->sc_rx_buf[tail], len);
    }
    sc->sc_rx_count += len;

    /* Wake any sleeper. */
    wakeup(&sc->sc_rx_count);
}
```

This is a ring-缓冲区 enqueue with wrap-around handling. The `usbd_copy_out` helper is used to move bytes from the USB 帧 into the 环形缓冲区. If the 环形缓冲区 is full, bytes are dropped. A real 驱动程序 would likely either apply USB-level 流控制 (stop arming new reads) or grow the 缓冲区; for the lab, dropping is acceptable.

The bulk-write 回调's helper to dequeue data is the mirror image:

```c
static unsigned int
myfirst_usb_tx_dequeue(struct myfirst_usb_softc *sc,
    struct usb_page_cache *pc, unsigned int max_len)
{
    unsigned int len, head;

    len = sc->sc_tx_count;
    if (len > max_len)
        len = max_len;
    if (len == 0)
        return (0);

    head = sc->sc_tx_head;
    if (head + len > MYFIRST_USB_TX_BUFSIZE) {
        usbd_copy_in(pc, 0, &sc->sc_tx_buf[head], MYFIRST_USB_TX_BUFSIZE - head);
        usbd_copy_in(pc, MYFIRST_USB_TX_BUFSIZE - head,
            &sc->sc_tx_buf[0], len - (MYFIRST_USB_TX_BUFSIZE - head));
    } else {
        usbd_copy_in(pc, 0, &sc->sc_tx_buf[head], len);
    }
    sc->sc_tx_head = (head + len) & (MYFIRST_USB_TX_BUFSIZE - 1);
    sc->sc_tx_count -= len;
    return (len);
}
```

The character-设备 methods are straightforward. Open checks that the 设备 is not already open, sets the open flag, and arms the read channel:

```c
static int
myfirst_usb_open(struct cdev *dev, int flags, int devtype, struct thread *td)
{
    struct myfirst_usb_softc *sc = dev->si_drv1;

    mtx_lock(&sc->sc_mtx);
    if (sc->sc_flags & MYFIRST_USB_FLAG_OPEN) {
        mtx_unlock(&sc->sc_mtx);
        return (EBUSY);
    }
    sc->sc_flags |= MYFIRST_USB_FLAG_OPEN;
    sc->sc_rx_head = sc->sc_rx_count = 0;
    sc->sc_tx_head = sc->sc_tx_count = 0;
    usbd_transfer_start(sc->sc_xfer[MYFIRST_USB_BULK_DT_RD]);
    usbd_transfer_start(sc->sc_xfer[MYFIRST_USB_INTR_DT_RD]);
    mtx_unlock(&sc->sc_mtx);

    return (0);
}
```

Close clears the open flag and stops the read channel:

```c
static int
myfirst_usb_close(struct cdev *dev, int flags, int devtype, struct thread *td)
{
    struct myfirst_usb_softc *sc = dev->si_drv1;

    mtx_lock(&sc->sc_mtx);
    usbd_transfer_stop(sc->sc_xfer[MYFIRST_USB_BULK_DT_RD]);
    usbd_transfer_stop(sc->sc_xfer[MYFIRST_USB_INTR_DT_RD]);
    usbd_transfer_stop(sc->sc_xfer[MYFIRST_USB_BULK_DT_WR]);
    sc->sc_flags &= ~MYFIRST_USB_FLAG_OPEN;
    mtx_unlock(&sc->sc_mtx);

    return (0);
}
```

Read 块 until data is available, then copies bytes from the 环形缓冲区 to userspace:

```c
static int
myfirst_usb_read(struct cdev *dev, struct uio *uio, int flags)
{
    struct myfirst_usb_softc *sc = dev->si_drv1;
    unsigned int len;
    char tmp[128];
    int error = 0;

    mtx_lock(&sc->sc_mtx);
    while (sc->sc_rx_count == 0) {
        if (sc->sc_flags & MYFIRST_USB_FLAG_DETACHING) {
            mtx_unlock(&sc->sc_mtx);
            return (ENXIO);
        }
        if (flags & O_NONBLOCK) {
            mtx_unlock(&sc->sc_mtx);
            return (EAGAIN);
        }
        error = msleep(&sc->sc_rx_count, &sc->sc_mtx,
            PCATCH | PZERO, "myfirstusb", 0);
        if (error != 0) {
            mtx_unlock(&sc->sc_mtx);
            return (error);
        }
    }

    while (uio->uio_resid > 0 && sc->sc_rx_count > 0) {
        len = min(uio->uio_resid, sc->sc_rx_count);
        len = min(len, sizeof(tmp));
        /* Copy out of ring buffer into tmp (handles wrap-around) */
        myfirst_usb_rx_read_into(sc, tmp, len);
        mtx_unlock(&sc->sc_mtx);
        error = uiomove(tmp, len, uio);
        mtx_lock(&sc->sc_mtx);
        if (error != 0)
            break;
    }
    mtx_unlock(&sc->sc_mtx);
    return (error);
}
```

Notice the pattern: the 互斥锁 is held while manipulating the 环形缓冲区, but it is released around the `uiomove` call, because `uiomove` can sleep (to fault in user pages) and sleeping while holding a 互斥锁 is forbidden. The 互斥锁 is reacquired after `uiomove` returns.

Write is the mirror: copy bytes from user to TX 缓冲区, then kick the write channel:

```c
static int
myfirst_usb_write(struct cdev *dev, struct uio *uio, int flags)
{
    struct myfirst_usb_softc *sc = dev->si_drv1;
    unsigned int len, space, tail;
    char tmp[128];
    int error = 0;

    mtx_lock(&sc->sc_mtx);
    while (uio->uio_resid > 0) {
        if (sc->sc_flags & MYFIRST_USB_FLAG_DETACHING) {
            error = ENXIO;
            break;
        }
        space = MYFIRST_USB_TX_BUFSIZE - sc->sc_tx_count;
        if (space == 0) {
            /* buffer is full; wait for the write callback to drain it */
            error = msleep(&sc->sc_tx_count, &sc->sc_mtx,
                PCATCH | PZERO, "myfirstusbw", 0);
            if (error != 0)
                break;
            continue;
        }
        len = min(uio->uio_resid, space);
        len = min(len, sizeof(tmp));

        mtx_unlock(&sc->sc_mtx);
        error = uiomove(tmp, len, uio);
        mtx_lock(&sc->sc_mtx);
        if (error != 0)
            break;

        /* Copy tmp into TX ring buffer (handles wrap-around) */
        tail = (sc->sc_tx_head + sc->sc_tx_count) & (MYFIRST_USB_TX_BUFSIZE - 1);
        myfirst_usb_tx_buf_append(sc, tail, tmp, len);
        sc->sc_tx_count += len;
        usbd_transfer_start(sc->sc_xfer[MYFIRST_USB_BULK_DT_WR]);
    }
    mtx_unlock(&sc->sc_mtx);
    return (error);
}
```

Two things are worth noticing in write. First, when the TX 缓冲区 is full, the write handler sleeps on `sc_tx_count`; the write 回调's `USB_ST_TRANSFERRED` branch calls `wakeup(&sc_tx_count)` after draining some bytes, which wakes the sleeping writer. Second, the write handler calls `usbd_transfer_start` on every chunk it enqueues. This is safe (starting an already-running channel is a no-op) and it ensures the write 回调 is nudged even if the channel had gone idle.

With these four cdev methods and the three 传输回调, you have a complete minimum-viable USB echo 驱动程序. The full source is approximately three hundred lines: short enough to fit on a single screen, concrete enough to exercise the real API.

### Choosing Between `usb_fifo` and a Custom `cdevsw`

When a USB驱动程序 needs to expose a `/dev` entry to userland, FreeBSD offers two approaches. The first is the `usb_fifo` 框架, a generic byte-stream abstraction that gives you `/dev/ugenN.M.epM` style nodes with read, write, poll, and a small ioctl 接口. You declare a `struct usb_fifo_methods` with open, close, start-read, start-write, stop-read, and stop-write 回调, and the 框架 handles the cdev plumbing and the queueing. This is the path of least resistance; `uhid(4)` and `ucom(4)` both use it.

The second approach is a custom `cdevsw`, the same pattern you practised in Chapter 24. This gives you total control over the user 接口 at the cost of writing more code. It is appropriate when the 驱动程序 needs a very specific ioctl surface, when the read/write semantics do not fit a byte stream (for example, a message-oriented protocol), or when the 驱动程序 already fits poorly into the `usb_fifo` model.

For the running example we have built, a custom `cdevsw` is the right choice because we wrote the 附加 method that calls `make_dev` and the 分离 method that calls `destroy_dev`, which is exactly what a custom `cdevsw` requires. For a 驱动程序 that exposes a byte stream (a serial adapter, say), `usb_fifo` is simpler. When you write your next USB驱动程序, look at both options and pick the one whose 接口 matches your problem.

### Error Handling and Retry Policy

The retry loop that our bulk-read 回调 uses, "on any error, rearm and try again," is a reasonable default for ro总线t 驱动程序. But it is not the only policy, and sometimes it is the wrong one.

For a 设备 that might genuinely go away mid-transfer (a USB adapter whose physical connection has been removed before the 框架 has had a chance to notice), rearming indefinitely is a waste; the transfers will keep failing until 分离 is called. Adding a small retry counter and giving up after, say, five consecutive errors, keeps the log from filling with noise.

For a 设备 that implements a strict request-response protocol, an error might invalidate the entire session. In that case, the 回调 should not rearm; instead, it should mark the 驱动程序 as "in error" and let the user close and reopen the 设备 to reset.

For a 设备 that supports stall-and-clear as a normal flow-control mechanism, the `usbd_xfer_set_stall` path is in the happy path, not the error path. Some class protocols use stalls to signal "I am not ready right now"; the 框架's automatic clear-stall machinery handles this transparently.

Your choice of retry policy should match the real behaviour of the 设备 you are writing for. When in doubt, start with the simple "rearm on error" default, observe what happens when you plug and unplug the 设备 repeatedly, and refine from there.

### Timeouts and Their Consequences

A timeout on a USB transfer is not just a safety net against hardware stalls; it is an explicit statement about how long the 驱动程序 is willing to wait for an operation to complete before treating it as a failure. Choosing a timeout is a design decision that interacts with many other parts of the 驱动程序, and getting it right requires thinking through several scenarios.

The configuration field `timeout` in `struct usb_config` is measured in milliseconds. A value of zero means "no timeout"; the transfer will wait indefinitely. A positive value means "if the transfer has not completed after this many milliseconds, cancel it and deliver a timeout error to the 回调."

For a read channel on a bulk 端点, the usual choice is zero. Reads on bulk channels are waiting for the 设备 to have something to say, and if the 设备 is silent for minutes, that is not necessarily an error. A timeout would force the 驱动程序 to rearm the read every few seconds, which wastes time and produces noise in the log.

For a write channel, the usual choice is a modest positive value like 5000 (five seconds). If the 设备 fails to drain its FIFO in that time, something is wrong; rather than 块 an indefinite-length write, the 驱动程序 returns an error to userland, which can retry if it wishes.

For an interrupt-IN channel polling for status updates, the usual choice is either zero (like a bulk read) or a timeout that matches the expected polling interval from the 端点 描述符's `bInterval` field. Matching `bInterval` gives the 驱动程序 an explicit "I should have heard from the 设备 by now" signal.

For a 控制传输, timeouts matter most, because 控制传输 are how the 驱动程序 configures the 设备, and a 设备 that does not respond to configuration is wedged. A timeout of 500 to 2000 milliseconds is common. If the 设备 does not respond to a configuration request in a few seconds, the 驱动程序 should assume something is wrong.

What happens when a timeout fires? The 框架 calls the 回调 with `USB_ERR_TIMEOUT` as the error. The 回调 typically treats this as a transient failure and rearms (for repeating channels) or returns an error to the caller (for one-shot operations). A repeating read channel that keeps timing out is probably talking to a 设备 that is not responding; after a few consecutive timeouts, it may be worth escalating by calling `usbd_transfer_unsetup` or by logging a more visible warning.

One subtle interaction is worth mentioning: if the transfer has a timeout and the 驱动程序 also sets `pipe_bof` (pipe 块ed on failure), a timeout will 块 the pipe until the 驱动程序 explicitly clears the 块. This is usually what you want, because the pipe may be in an inconsistent state, and clearing the 块 (by submitting a fresh setup, or by calling `usbd_transfer_start`) is a good point to log what happened and decide what to do next.

### What Goes Wrong When Transfer Setup Fails

The `usbd_transfer_setup` call can fail for several reasons. Understanding each is useful both for debugging your own 驱动程序 and for reading the FreeBSD source when you encounter failures.

**Endpoint mismatch.** If the configuration array asks for an 端点 with a specific type/direction pair that does not exist on the 接口, the call fails with `USB_ERR_NO_PIPE`. This usually means the 匹配表 matched a 设备 that has a different 描述符 layout than the 驱动程序 expected; it is a bug in the 驱动程序.

**Unsupported transfer type.** If the configuration specifies `UE_ISOCHRONOUS` on a 主机控制器 that does not support 等时传输, or if the bandwidth reservation cannot be satisfied, the call fails. Isochronous is the most complex transfer type and the most likely to have platform-specific limitations.

**Out of memory.** The 框架 allocates DMA-capable 缓冲区 for the channels. If memory is low, the allocation fails. This is rare on modern systems but can happen on embedded platforms with tight memory budgets.

**Missing or invalid attributes.** If the configuration has a 缓冲区 size of zero, or a negative 帧 count, or an invalid flag combination, the call fails. Check the configuration against the declarations in `/usr/src/sys/dev/usb/usbdi.h`.

**Power management states.** If the 设备 has been suspended or is in a low-power state, some transfer setup requests will fail. This is mainly relevant for 驱动程序 that handle USB selective suspend.

When `usbd_transfer_setup` fails, the error code is an `usb_error_t` value, not a standard errno. The definitions are in `/usr/src/sys/dev/usb/usbdi.h`. The function `usbd_errstr` converts an error code to a printable string; use it in your `设备_printf` to make diagnostic messages informative.

### A Detail About `pipe_bof`

We mentioned `pipe_bof` (pipe 块ed on failure) as a flag in the transfer configuration, but the motivation for it deserves a closer look. USB 端点 are conceptually single-threaded from the 设备's perspective. When the host submits a bulk-OUT 数据包, the 设备 must process that 数据包 before accepting another. If the 数据包 fails, the 设备 may be in an indeterminate state, and the next 数据包 should not be sent until the 驱动程序 has had a chance to resynchronise.

`pipe_bof` tells the 框架 to pause the pipe when a transfer fails. The next `usbd_transfer_submit` will not actually start a hardware operation; instead, the 框架 waits until the 驱动程序 explicitly calls `usbd_transfer_start` on the channel, which acts as a "resume" signal. This lets the 驱动程序 do a clear-stall or otherwise resynchronise before the next transfer begins.

Without `pipe_bof`, the 框架 would immediately submit the next transfer after a failure, which might run into the same failure before the 驱动程序 has had a chance to react.

Setting `pipe_bof = 1` is the safe default for most 驱动程序. Clearing it is appropriate for 驱动程序 that want to keep a pipeline full even through occasional errors (for example, audio 驱动程序 where a brief glitch is preferable to a synchronous resynchronisation).

### `short_xfer_ok` and Data-Length Semantics

The `short_xfer_ok` flag is another configuration option whose meaning is worth spelling out. USB 批量传输 do not have an inherent end-of-message marker. If the host has a 缓冲区 of 512 bytes and the 设备 only has 100 bytes to send, what should happen? There are two possible answers.

With `short_xfer_ok` clear (the default), a transfer that completes with less data than requested is treated as an error. The 框架 delivers `USB_ERR_SHORT_XFER` to the 回调, and the 驱动程序 must decide whether to retry, ignore, or escalate.

With `short_xfer_ok` set, a short transfer is treated as success. The 回调 gets `USB_ST_TRANSFERRED` with `actlen` set to the actual number of bytes received. This is almost always what you want for bulk-IN on message-oriented protocols, where the 设备 decides how much data to send.

There is a corresponding flag for outgoing transfers: `force_short_xfer`. If set, a transfer whose data happens to be an exact multiple of the 端点's maximum 数据包 size will be padded with a zero-length 数据包 at the end to signal "end of message." USB treats a zero-length 数据包 as a valid transaction, and many protocols use it as an explicit boundary marker. The FTDI 驱动程序 sets this flag on its write channel, for example, because the FTDI protocol expects a trailing short 数据包.

Knowing which flag is appropriate requires knowing the protocol the 设备 implements. When you write a 驱动程序 for a 设备 documented with a public protocol specification, check the specification for how it handles boundaries. When you write a 驱动程序 for a poorly-documented 设备, set `short_xfer_ok` on reads (you can always count the bytes), and test both settings of `force_short_xfer` on writes to see which the 设备 accepts.

### Locking Rules Around Transfers

The USB 框架 imposes two locking rules that are essential to get right.

First, the 互斥锁 you pass to `usbd_transfer_setup` is held by the 框架 around every invocation of the 回调. You do not need to acquire it inside the 回调; it is already held. You also must not release it inside the 回调; doing so breaks the 框架's assumption and can cause random failures.

Second, every call from 驱动程序 code (not from the 回调) to one of `usbd_transfer_start`, `usbd_transfer_stop`, `usbd_transfer_submit`, `usbd_transfer_drain`, and `usbd_transfer_pending` must be made with the 互斥锁 held. This is because these functions read and write fields inside the transfer object that the 回调 also touches, and the 互斥锁 is what serialises access.

Practically, this means most 驱动程序 code that interacts with transfers looks like:

```c
mtx_lock(&sc->sc_mtx);
usbd_transfer_start(sc->sc_xfer[MYFIRST_USB_BULK_DT_WR]);
mtx_unlock(&sc->sc_mtx);
```

or in longer critical sections:

```c
mtx_lock(&sc->sc_mtx);
/* enqueue data */
enqueue(sc, data, len);
/* nudge the channel if it is idle */
if (!usbd_transfer_pending(sc->sc_xfer[MYFIRST_USB_BULK_DT_WR]))
    usbd_transfer_start(sc->sc_xfer[MYFIRST_USB_BULK_DT_WR]);
mtx_unlock(&sc->sc_mtx);
```

Drivers that violate these rules occasionally appear to work but fail intermittently on load, under heavy I/O, or during 分离. Getting the locking right from the start saves many hours of debugging later.

### 总结 Section 3

Section 3 has shown how data flows through a USB驱动程序. A configuration array declares the channels, `usbd_transfer_setup` allocates them, the 回调 drive them through the three-state machine, and `usbd_transfer_unsetup` tears them down. The 框架 abstracts away the hardware details: DMA 缓冲区, 帧 scheduling, 端点 arbitration, stall handling. The 驱动程序's job is to write 回调 that handle completion and to arrange the flow of data through the 回调.

Three themes are worth carrying forward. First, the three-state state machine (`USB_ST_SETUP`, `USB_ST_TRANSFERRED`, `USB_ST_ERROR`) is the same in every channel, regardless of transfer type. Learning to read a USB 回调 means learning to parse this state machine; once you know it, every 回调 in every USB驱动程序 in the tree is legible. Second, the `struct usb_page_cache` abstraction is the only safe way to move data into and out of USB 缓冲区. Never bypass `usbd_copy_in` and `usbd_copy_out`. Third, the locking discipline around `usbd_transfer_start`, `_stop`, and `_submit` is not optional; every call from 驱动程序 code must be made under the 互斥锁.

With Sections 1 through 3 in hand, you have a complete mental model of USB驱动程序 writing: the concepts, the skeleton, and the data pipeline. Section 4 now shifts to the serial side of 第6部分. The UART subsystem is older, simpler in some ways, more constrained in others, and its idioms are different from USB's. But many of the same habits carry over: match against what you support, 附加 in phases that can be cleanly reversed, drive the hardware through a state machine, and respect the locking.

> **Take a breath.** We have now worked through the USB half of the chapter: the host and 设备 roles, the 描述符 tree, the four transfer types, the 探测/附加/分离 skeleton, and the three-state `USB_ST_SETUP`/`USB_ST_TRANSFERRED`/`USB_ST_ERROR` 回调 machine that every USB驱动程序 runs. The rest of the chapter turns to the serial side: the `uart(4)` 框架 with its `ns8250` reference 驱动程序, integration with the TTY layer and `termios`, the `ucom(4)` bridge used by USB-to-serial adapters, and the tools and labs that let you test both kinds of 驱动程序 without real hardware. If you want to close the book and come back, this is a natural pause.

## 第4节： Writing a Serial UART Driver

### From USB to UART: A Shift of Landscape

Sections 2 and 3 gave you a complete USB驱动程序. The 框架 there was modern in every sense: 热插拔, DMA-aware, message-oriented, richly abstracted. Section 4 now turns to `uart(4)`, FreeBSD's 框架 for driving Universal Asynchronous Receiver/Transmitters. The landscape is different. Many UART chips are older than USB itself, and the 框架's design reflects that. There is no 热插拔 (a 串行端口 is usually soldered to the board). There is no DMA for most parts (the chip has a small FIFO you poll or an interrupt you handle). There is no 描述符 hierarchy (the chip does not advertise its capabilities; you know what you built against). And there is no notion of transfer channels; there is just the port, into which bytes go and out of which bytes come.

What the 框架 does provide is a disciplined split of responsibilities between three layers. At the bottom sits your 驱动程序, which knows how the chip's 寄存器 work, how its interrupts fire, how its FIFOs behave, and what platform-specific resources (IRQ line, I/O port range, clock source) it needs. In the middle sits the `uart(4)` 框架 itself, which handles registration, baud-rate configuration calculations, 缓冲区ing, TTY integration, and the scheduling of read and write work. At the top sits the TTY layer, which presents the port to userland as `/dev/ttyuN` and `/dev/cuauN` and handles terminal semantics: line editing, signal generation, control characters, and the vast vocabulary of `termios` knobs that `stty(1)` exposes.

You do not write the TTY layer. You do not write most of the `uart(4)` 框架. Your job, when you write a UART 驱动程序, is to implement a small set of hardware-specific methods that the 框架 calls when it needs to do something at the 注册 level. The 框架 then wires those methods into the rest of the 内核's serial machinery for free.

This section walks through that wiring. It covers the layout of the `uart(4)` 框架, the structures and methods you have to fill in, the canonical `ns8250` 驱动程序 as a concrete reference, and the integration with the TTY layer. It ends with the related `ucom(4)` 框架, which is how USB-to-serial bridges expose themselves to userland using the same TTY 接口 as a real UART.

### Where the `uart(4)` Framework Lives

The 框架 itself lives in `/usr/src/sys/dev/uart/`. If you list that directory, you see a handful of 框架 files and a family of hardware-specific 驱动程序.

The 框架 files are:

- `/usr/src/sys/dev/uart/uart.h`: the top-level header that defines the 框架's public API.
- `/usr/src/sys/dev/uart/uart_总线.h`: the structures for new总线 integration and the per-port softc.
- `/usr/src/sys/dev/uart/uart_core.c`: the 附加ment logic, the interrupt dispatcher, the polling loop, the link between `uart(4)` and `tty(4)`.
- `/usr/src/sys/dev/uart/uart_tty.c`: the `ttydevsw` implementation that maps `uart(4)` operations onto `tty(4)` operations.
- `/usr/src/sys/dev/uart/uart_cpu.h`, `uart_dev_*.c`: platform glue and console registration.

The hardware-specific 驱动程序 are files of the form `uart_dev_NAME.c` and occasionally `uart_dev_NAME.h`. The most important of these is `uart_dev_ns8250.c`, which implements the ns8250 family (including the 16450, 16550, 16550A, 16650, 16750, and many compatibles). Because the 16550A is effectively the standard UART for PC-style 串行端口s, this one 驱动程序 handles the majority of actual serial hardware in the world. When you want to learn how a real FreeBSD UART 驱动程序 looks, this is the file to open.

Other 驱动程序 in the directory handle chips that are not 16550-compatible: the Intel MID variant, the PL011 ARM UART used on Raspberry Pi and other ARM boards, the NXP i.MX UART, the Sun Microsystems Z8530, and so on. Each one follows the same pattern: fill in a `struct uart_class` and a `struct uart_ops`, 注册 with the 框架, and implement the hardware access methods.

### The `uart_class` Structure

Every UART 驱动程序 begins by declaring a `struct uart_class`, which is the hardware 描述符 that the 框架 uses to identify the chip family. The definition lives in `/usr/src/sys/dev/uart/uart_总线.h`. The structure looks like this (paraphrased; the real declaration has a few more fields):

```c
struct uart_class {
    KOBJ_CLASS_FIELDS;
    struct uart_ops *uc_ops;
    u_int            uc_range;
    u_int            uc_rclk;
    u_int            uc_rshift;
};
```

The `KOBJ_CLASS_FIELDS` macro pulls in the kobj machinery that Chapter 23 introduced (in the context of New总线). A `uart_class` is, at the 内核's abstract-object level, a kobj class whose instances are `uart_softc`. This is how the 框架 can call into 驱动程序-specific methods without needing an `if` ladder: the method dispatch is done by kobj lookup.

`uc_ops` is a pointer to the operations structure (coming next), which lists the chip-specific methods.

`uc_range` is how many bytes of 注册 address space the chip uses. For an ns16550-compatible UART, this is 8.

`uc_rclk` is the reference clock frequency in hertz. The 框架 uses this to compute baud-rate divisors. For a PC-style UART, the reference clock is usually 1,843,200 hertz (a specific multiple of the standard 波特率s).

`uc_rshift` is the 注册 address shift. On some 总线es, UART 寄存器 are spaced at intervals other than one byte (for example, every four bytes on some memory-mapped designs). A shift of zero means tight packing; a shift of two means each logical 注册 occupies four bytes of address space.

For our running example, the class declaration looks like this:

```c
static struct uart_class myfirst_uart_class = {
    "myfirst_uart class",
    myfirst_uart_methods,
    sizeof(struct myfirst_uart_softc),
    .uc_ops   = &myfirst_uart_ops,
    .uc_range = 8,
    .uc_rclk  = 1843200,
    .uc_rshift = 0,
};
```

The first three positional arguments are the `KOBJ_CLASS_FIELDS` entries: a name, a method table, and a per-instance size. The named fields are the UART-specific ones. For a 驱动程序 targeting 16550-compatible chips, these values are the conventional defaults.

### The `uart_ops` Structure

The `struct uart_ops` is where the real hardware-specific code lives. It is a table of function pointers that the 框架 calls at specific moments. The definition lives in `/usr/src/sys/dev/uart/uart_cpu.h`:

```c
struct uart_ops {
    int  (*probe)(struct uart_bas *);
    void (*init)(struct uart_bas *, int, int, int, int);
    void (*term)(struct uart_bas *);
    void (*putc)(struct uart_bas *, int);
    int  (*rxready)(struct uart_bas *);
    int  (*getc)(struct uart_bas *, struct mtx *);
};
```

Each operation takes a `struct uart_bas *` as its first argument. The "bas" stands for "总线 address space"; it is the 框架's abstraction for access to the chip's 寄存器. A 驱动程序 does not know or care whether the chip is in I/O space or in memory-mapped space; it just calls `uart_getreg(bas, offset)` and `uart_setreg(bas, offset, value)` (declared in `/usr/src/sys/dev/uart/uart.h`), and the 框架 routes the access correctly.

Let us go through the six operations in turn.

`探测` is called when the 框架 needs to know whether a chip of this class is present at a given address. The 驱动程序 typically pokes a 注册, reads it back, and returns zero if the readback matches (suggesting the chip is really there) or a nonzero errno otherwise. For an ns16550, the 探测 writes a test pattern to the scratch 注册 and reads it back.

`init` is called to initialise the chip to a known state. The arguments after the bas are `baudrate`, `databits`, `stopbits`, and `奇偶校验`. The 驱动程序 computes the divisor, writes the divisor-latch-access bit, writes the divisor, clears the divisor-latch, sets the line control 注册 for the requested data/stop/奇偶校验 configuration, enables the FIFOs, and enables the chip's interrupts. The exact 注册 sequence for a 16550 is several dozen lines of code and is documented in the chip's data sheet.

`term` is called to shut down the chip. It typically disables interrupts, flushes the FIFOs, and leaves the chip in a safe state.

`putc` sends a single character. This is used by the low-level console path and by polling-based diagnostic output. The 驱动程序 总线y-waits on the transmitter-holding-注册-empty flag, then writes the byte to the transmit 注册.

`rxready` returns nonzero if at least one byte is available to read. The 驱动程序 reads the line status 注册 and checks the data-ready bit.

`getc` reads a single character. Used by the low-level console for input. The 驱动程序 总线y-waits on the data-ready flag (or the caller ensures `rxready` just returned true), then reads the receive 注册.

These six methods are the entire hardware-specific surface for a UART 驱动程序 at the low level. Everything else (interrupt handling, 缓冲区ing, TTY integration, 热插拔 of PCIe UARTs, console selection) is provided by the 框架. A new UART 驱动程序 is, in effect, a six-function implementation plus a handful of declarations.

### A Closer Look at `ns8250`

The ns8250 驱动程序 at `/usr/src/sys/dev/uart/uart_dev_ns8250.c` is the best place to see these methods concretely. It is a mature, production-grade 驱动程序 that handles every variant of the 8250/16450/16550/16550A family. The 注册 definitions it uses (from `/usr/src/sys/dev/ic/ns16550.h`) are the same ones every UART-related header in the PC world uses. When you read this 驱动程序, you are reading, in effect, the reference implementation of a 16550 驱动程序 for FreeBSD.

The put-character implementation is instructive for its simplicity:

```c
static void
ns8250_putc(struct uart_bas *bas, int c)
{
    int limit;

    limit = 250000;
    while ((uart_getreg(bas, REG_LSR) & LSR_THRE) == 0 && --limit)
        DELAY(4);
    uart_setreg(bas, REG_DATA, c);
    uart_barrier(bas);
    limit = 250000;
    while ((uart_getreg(bas, REG_LSR) & LSR_TEMT) == 0 && --limit)
        DELAY(4);
}
```

The loop polls the line status 注册 (LSR) for the transmitter-holding-注册-empty (THRE) flag. When it is set, the transmit holding 注册 is ready to accept a byte. The 驱动程序 writes the byte to the data 注册 (REG_DATA) and then polls again for the transmitter-empty (TEMT) flag to ensure the byte has been shifted out before returning.

The `uart_barrier` call is a memory barrier that ensures the write to the data 注册 is visible to the hardware before subsequent reads. On platforms with weak memory ordering, missing this barrier would cause intermittent data loss.

The `DELAY(4)` yields four microseconds per iteration, and the `limit` counter is 250,000. Together, they give a one-second timeout before the loop gives up. For a real UART, 250,000 iterations is a cap that should never be reached in normal operation; it is a safety net for the pathological case where the chip is in an unexpected state.

The 探测 is equally direct:

```c
static int
ns8250_probe(struct uart_bas *bas)
{
    u_char val;

    /* Check known 0 bits that don't depend on DLAB. */
    val = uart_getreg(bas, REG_IIR);
    if (val & 0x30)
        return (ENXIO);
    return (0);
}
```

Bits 4 and 5 of the Interrupt Identification Register (IIR) are defined as always-zero for every variant of the 16550 family. If those bits read as one, this is not a real 16550 注册, and the 探测 rejects the address.

You could read the whole 驱动程序 in an afternoon. What you would come away with is a clear mental model: the methods are narrow, the 框架 is large, and the real engineering is in handling the quirks of specific chip revisions (a FIFO bug in the 16550 predecessor, an erratum in some PC chipsets, a signal-detect issue on certain Oxford 设备). A new UART 驱动程序 for a well-behaved chip is genuinely a small file.

### The `uart_softc` and How the Framework Uses It

Each instance of a UART 驱动程序 has a `struct uart_softc`, defined in `/usr/src/sys/dev/uart/uart_总线.h`. The 框架 allocates one per 附加ed port. Its important fields include a pointer to the `uart_bas` that describes the port's 注册 layout, the I/O resources (the IRQ, the memory range or I/O port range), the TTY 设备 附加ed to this port, the current line parameters, and two byte 缓冲区 (RX and TX) that the 框架 uses internally. The 驱动程序 does not usually allocate its own softc; it uses the 框架's `uart_softc`, with the hardware-specific extensions added through kobj class inheritance.

When the 框架 receives an interrupt from a UART, it calls a 框架-internal function that reads the interrupt-identification 注册, decides what kind of work the chip has requested (transmit-ready, receive-data-available, line-status, modem-status), and dispatches to the appropriate handler. The handlers pull data out of the chip's RX FIFO into the 框架's RX 环形缓冲区, or push data from the 框架's TX 环形缓冲区 into the chip's TX FIFO, or update state variables in response to modem-signal changes. The 中断处理程序 returns, and the TTY layer consumes the 环形缓冲区 at its own pace through the 框架's put-character and get-character paths.

This is why the 驱动程序's `uart_ops` table is so small. The high-volume work (moving bytes between the chip and the 环形缓冲区) is handled by shared 框架 code that reads the chip's 寄存器 through `uart_getreg` and `uart_setreg`. The 驱动程序 only needs to expose the low-level primitives; the composition is done for it.

### Integration with the TTY Layer

Above the `uart(4)` 框架 sits the TTY layer, defined in `/usr/src/sys/kern/tty.c` and friends. A UART port in FreeBSD appears to userland as two `/dev` nodes:

- `/dev/ttyuN`: the callin node. Opening it 块 until a carrier detect signal is asserted (which models an incoming call on a modem). It is used for 设备 that answer, not initiate, connections.
- `/dev/cuauN`: the callout node. Opening it does not wait for carrier detect. It is used for 设备 that initiate connections, or for developers who want to talk to a 串行端口 without pretending it is a modem.

The distinction is historical, dating from the era when 串行端口s were genuinely connected to analog modems with separate "someone is calling" and "I am initiating a call" semantics. FreeBSD preserves the distinction because some embedded and industrial workflows still rely on it, and because the implementation cost is minimal once the TTY layer's "two sides of the same port" pattern is in place.

The TTY layer calls into the `uart(4)` 框架 through a `ttydevsw` structure whose methods map neatly onto UART operations. The important entries include:

- `tsw_open`: called when userland opens the port. The 框架 enables interrupts, powers on the chip, and applies the default `termios`.
- `tsw_close`: called when the last userland reference is released. The 框架 drains the TX 缓冲区, disables interrupts (unless the port is also a console), and puts the chip in an idle state.
- `tsw_ioctl`: called for ioctls the TTY layer does not handle itself. Most UART-specific ioctls are handled by the 框架.
- `tsw_param`: called when `termios` changes. The 框架 reprograms the chip's 波特率, 数据位, 停止位, 奇偶校验, and 流控制.
- `tsw_outwakeup`: called when there is new data to transmit. The 框架 enables the transmit-ready interrupt if it was disabled; on the next IRQ, the 框架 pushes bytes from the 环形缓冲区 into the chip.

You do not usually have to write any of these. The 框架 in `uart_tty.c` implements them once for every UART 驱动程序. Your 驱动程序's only contribution is the six methods in `uart_ops`.

### The `termios` Interface in Practice

When a user runs `stty 115200` on a 串行端口, the following chain of calls happens:

1. `stty(1)` opens the port and issues a `TIOCSETA` ioctl carrying the new `struct termios`.
2. The 内核 TTY layer receives the ioctl and updates its internal copy of the port's termios.
3. The TTY layer calls `tsw_param` on the port's `ttydevsw`, passing the new termios.
4. The `uart(4)` 框架's `uart_param` implementation looks at the termios fields (`c_ispeed`, `c_ospeed`, `c_cflag` with its `CSIZE`, `CSTOPB`, `PARENB`, `PARODD`, `CRTSCTS` sub-bits) and calls the 驱动程序's `init` method with the corresponding raw values.
5. The 驱动程序's `init` method computes the divisor, writes the line-control 注册, reconfigures the FIFO, and returns.

None of this requires the 驱动程序 to know about termios. The translation from termios bits to raw integers is done by the 框架. The 驱动程序 sees only the raw values: baudrate in bits per second, databits (usually 5 through 8), stopbits (1 or 2), and a 奇偶校验 code.

This separation is what lets FreeBSD run the same `uart(4)` 框架 on top of radically different chips. A 16550 驱动程序 and a PL011 驱动程序 both implement the same six `uart_ops` methods. The termios-to-raw translation happens once, in 框架 code, for every chip family.

### Flow Control at the Register Level

Hardware 流控制 is typically driven by two signals on the UART: CTS (clear to send) and RTS (request to send). When CTS is asserted by the remote side, it is telling the local transmitter "I am ready for more data." When the local side asserts RTS, it is telling the remote transmitter the same thing. When either signal is not asserted, the corresponding transmitter pauses.

In a 16550, RTS is driven by a bit in the modem control 注册 (MCR), and CTS is read from a bit in the modem status 注册 (MSR). The 框架 exposes 流控制 through termios (`CRTSCTS` flag), through ioctls (`TIOCMGET`, `TIOCMSET`, `TIOCMBIS`, `TIOCMBIC`), and through automatic responses to FIFO fill levels.

When the receive FIFO fills past a threshold, the 驱动程序 deasserts RTS to ask the remote side to stop transmitting. When the FIFO drains below a different threshold, the 驱动程序 reasserts RTS. When the modem-status-interrupt fires because CTS changed, the 中断处理程序 enables or disables the transmit path accordingly. All of this is 框架 logic; the 驱动程序 only exposes the 注册-level primitives.

Software 流控制 (XON/XOFF) is handled entirely in the TTY layer, by inserting and interpreting the XON (0x11) and XOFF (0x13) bytes in the data stream. The UART 驱动程序 has no role in it.

### The Interrupt Handler Path in Detail

Beyond the six `uart_ops` methods, a real UART 驱动程序 usually implements an 中断处理程序. The 框架 provides a generic one in `uart_core.c` that works for the vast majority of chips, but the 驱动程序 can supply its own for chips with unusual behaviour. To understand what the 框架's generic handler does, and when you might want to override it, it helps to trace the handler's path.

When the hardware interrupt fires, the 框架's ISR reads the interrupt identification 注册 (IIR) through `uart_getreg`. The IIR encodes which of four conditions triggered the interrupt: line-status (a framing error or overrun occurred), received-data-available (at least one byte is in the receive FIFO), transmitter-holding-注册-empty (the TX FIFO wants more data), or modem-status (a modem signal changed state).

For line-status interrupts, the 框架 logs a warning (or increments a counter) and continues.

For received-data-available, the 框架 reads bytes out of the chip's RX FIFO one at a time, pushing each into the 驱动程序's internal RX 环形缓冲区. The loop continues until the receive-data-available flag clears. Once the 环形缓冲区 has bytes, the 框架 signals the TTY layer's input path, which will pull bytes out as the consumer is ready.

For transmitter-holding-注册-empty, the 框架 pulls bytes out of its internal TX 环形缓冲区 and pushes them into the chip's TX FIFO one at a time. The loop continues until the TX FIFO is full or the 环形缓冲区 is empty. Once the 环形缓冲区 is empty, the 框架 disables the transmit interrupt so the chip does not keep firing; the next `tsw_outwakeup` call (from the TTY layer, when there is new data) will reenable it.

For modem-status changes, the 框架 updates its internal modem-signal state and signals the TTY layer if the change is significant (for example, CTS deassertion when hardware 流控制 is enabled).

This is all done in interrupt context with the 驱动程序's 互斥锁 held. The 互斥锁 is a spin 互斥锁 (`MTX_SPIN`) for UART 驱动程序, because taking a sleepable 互斥锁 in an 中断处理程序 is forbidden. The 框架's helpers know this and use appropriate primitives.

When might a 驱动程序 want to override the generic handler? Three situations come to mind.

First, if the chip has unusual FIFO semantics. Some chips do not clear their interrupt identification 寄存器 in the obvious way; you have to drain the FIFO completely, or you have to read a specific 注册 to acknowledge. If your chip's data sheet describes such a quirk, you override the handler with chip-specific logic.

Second, if the chip has DMA support you want to use. The 框架's generic handler is PIO (programmed I/O): one byte per 注册 access. A chip with a DMA engine could move many bytes per interrupt, significantly reducing CPU overhead at high 波特率s. Implementing DMA requires chip-specific code.

Third, if the chip has hardware timestamping or other advanced features. Some embedded UARTs can timestamp individual received bytes with microsecond precision, which is invaluable for industrial protocols. The 框架 does not know about this, so the 驱动程序 must implement it.

For typical hardware, the generic handler is correct and performant. Do not override it without a specific reason.

### The TX and RX Ring Buffers

The `uart(4)` 框架 keeps two 环形缓冲区 inside each port's softc. These are separate from any 缓冲区ing on the chip itself: even if the chip has a 64-byte FIFO, the 框架 has its own 环形缓冲区 of some configurable size (typically 4 KB for each direction) that sit between the chip and the TTY layer.

The purpose of these 环形缓冲区 is to absorb bursts. Suppose the consumer of data is slow (a 总线y userland process), and the producer (the remote serial 设备) is pushing data at 115200 baud. Without a 环形缓冲区, the chip's 64-byte FIFO would fill up in about 6 milliseconds, and bytes would be lost. With a 4 KB 环形缓冲区, the 缓冲区 can absorb a 350-millisecond burst at 115200 baud, which is enough for userland to catch up in almost every realistic scenario.

The sizes of these 环形缓冲区 are not generally configurable per-驱动程序; they are baked into the 框架. The 环形缓冲区 implementation is in `uart_core.c` and uses the same kind of head/tail pointer arithmetic as the 环形缓冲区 in our USB echo 驱动程序.

When the TTY layer asks for bytes (through `ttydisc_rint`), the 框架 moves bytes out of the RX ring into the TTY layer's own input queue, which has its own 缓冲区ing and line-discipline processing (canonical mode, echo, signal generation, and so on). When userland writes bytes, they arrive at the 框架's `tsw_outwakeup` path and are moved into the TX ring; the 框架's transmit-empty 中断处理程序 pushes them from the ring into the chip.

This arrangement has a nice property: the 驱动程序, the 框架, and the TTY layer are all loosely coupled. The 驱动程序 only knows about the chip. The 框架 only knows about 寄存器 and 环形缓冲区. The TTY layer only knows about 缓冲区ing and line discipline. Each layer can be tested and reasoned about independently.

### Debugging Serial Drivers

When a 串行驱动程序 does not work, the symptoms can be confusing. Bytes go in, bytes come out, but the two do not match. The clock ticks, but the characters look like gibberish. The port opens, but writes return zero bytes. This section lists the diagnostic techniques that help.

**Log aggressively at 附加.** Use `设备_printf(dev, "附加ed at %x, IRQ %d\n", ...)` to verify the address and IRQ your 驱动程序 ended up with. If the address is wrong, no I/O will work; if the IRQ is wrong, no interrupts will fire. Attach messages are the first line of defence.

**Use `sysctl dev.uart.0.*` to inspect port state.** The `uart(4)` 框架 exports many per-port knobs and statistics through sysctl. Reading them shows the current 波特率, the number of bytes transmitted, the number of overruns, the modem signal state, and more. If `tx` is incrementing but `rx` is not, the transmitter works but the receiver does not; if both are zero, nothing is happening at all.

**Probe the hardware with `kgdb`.** If you have a 内核 crash dump or the ability to 附加 a 内核 debugger, you can inspect the `uart_softc` directly and read its 注册 values. This is invaluable when the chip is in a confused state that the software abstraction hides.

**Compare against a working 驱动程序.** If your modification broke something, bisect the change against the upstream `ns8250.c`. The difference will be small, and once you identify it, the fix is usually clear.

**Use `dd` for small, repeatable tests.** Instead of `cu` for debugging, use `dd if=/dev/zero of=/dev/cuau0 bs=1 count=100` to write exactly 100 bytes. Then `dd if=/dev/cuau0 of=output.bin bs=1 count=100` to read exactly 100 bytes (with a suitable timeout or a second open). This isolates timing and character-encoding issues that interactive `cu` might mask.

**Check the hardware 流控制 pins.** Many flow-control bugs are hardware, not software. Use a break-out board, a multimeter, or an oscilloscope to verify that DTR, RTS, CTS, and DSR are at the voltages you expect. If one is stuck floating, the chip's behaviour is undefined.

**Compare behaviour under `nmdm(4)`.** If your userland tool works with `nmdm(4)` but not with your 驱动程序, the bug is in the 驱动程序. If it fails with both, the bug is in the tool.

These techniques apply equally to `uart(4)` 驱动程序 and `ucom(4)` 驱动程序. The difference is that `uart(4)` problems often come down to 注册 manipulation (did you set the divisor correctly?), while `ucom(4)` problems often come down to USB transfers (did the 控制传输 to set the 波特率 actually succeed?). The debugging tools (USB: `usbconfig`, transfer statistics; UART: `sysctl`, chip 寄存器) are different, but the investigative mindset is the same.

### Writing a UART Driver Yourself

Putting the pieces together, a minimal UART 驱动程序 for an imaginary 注册-compatible chip would be organised like this:

1. Define 注册 offsets and bit positions in a local header.
2. Implement the six `uart_ops` methods: `探测`, `init`, `term`, `putc`, `rxready`, `getc`.
3. Declare a `struct uart_ops` initialised with those six methods.
4. Declare a `struct uart_class` initialised with the ops and the hardware parameters (range, reference clock, 注册 shift).
5. Implement the 中断处理程序 if the chip needs more than the 框架's default dispatch.
6. Register the 驱动程序 with New总线 using the standard macros.

Most new UART 驱动程序 in the tree are small. Oxford single-port PCIe UARTs, for example, are a few hundred lines because they are fundamentally 16550-compatible and only need a thin layer of PCI-specific 附加 code. Complex ones like the Z8530 are larger because the chip has a more complicated programming model; the 驱动程序 size tracks the chip's complexity, not the 框架's.

### Looking at `myfirst_uart.c` in Skeleton Form

For our running example, the skeleton of a minimal UART 驱动程序 looks like this:

```c
#include <sys/param.h>
#include <sys/systm.h>
#include <sys/bus.h>
#include <sys/module.h>
#include <sys/kernel.h>

#include <dev/uart/uart.h>
#include <dev/uart/uart_cpu.h>
#include <dev/uart/uart_bus.h>

#include "uart_if.h"

static int   myfirst_uart_probe(struct uart_bas *);
static void  myfirst_uart_init(struct uart_bas *, int, int, int, int);
static void  myfirst_uart_term(struct uart_bas *);
static void  myfirst_uart_putc(struct uart_bas *, int);
static int   myfirst_uart_rxready(struct uart_bas *);
static int   myfirst_uart_getc(struct uart_bas *, struct mtx *);

static struct uart_ops myfirst_uart_ops = {
    .probe   = myfirst_uart_probe,
    .init    = myfirst_uart_init,
    .term    = myfirst_uart_term,
    .putc    = myfirst_uart_putc,
    .rxready = myfirst_uart_rxready,
    .getc    = myfirst_uart_getc,
};

struct myfirst_uart_softc {
    struct uart_softc base;
    /* any chip-specific state would go here */
};

static kobj_method_t myfirst_uart_methods[] = {
    /* Most methods inherit from the framework. */
    { 0, 0 }
};

struct uart_class myfirst_uart_class = {
    "myfirst_uart class",
    myfirst_uart_methods,
    sizeof(struct myfirst_uart_softc),
    .uc_ops    = &myfirst_uart_ops,
    .uc_range  = 8,
    .uc_rclk   = 1843200,
    .uc_rshift = 0,
};
```

The inclusion of `uart_if.h` is notable: that header is generated at build time by the kobj machinery from the 接口 definition in `/usr/src/sys/dev/uart/uart_if.m`. It declares the method prototypes that the 框架 expects 驱动程序 to implement. When you write a new 驱动程序, you depend on this header.

The six methods themselves are straightforward once you have the chip's programming manual open. `init` computes the divisor from `uc_rclk` and the 波特率, writes the line control 注册 for the databits/stopbits/奇偶校验 combination, enables FIFOs, and sets the interrupt enable 注册 to the desired mask. `term` inverts `init`. `putc`, `getc`, and `rxready` each do a single-注册 access plus a spin on the status 注册.

A complete implementation of all six methods for a 16550-compatible chip is about three hundred lines. For a chip with quirks, it might grow to five hundred or more. The `ns8250` 驱动程序 is longer than most because it handles errata and variant detection for dozens of real chips, but the core logic of its six methods is still the standard pattern.

### The `ucom(4)` Framework: USB-to-Serial Bridges

Not every 串行端口 is a real UART on the system 总线. Many are USB adapters: a PL2303, a CP2102, an FTDI FT232, a CH340G. These chips expose a 串行端口 over USB, and FreeBSD's approach to supporting them is a small 框架 called `ucom(4)`. It lives in `/usr/src/sys/dev/usb/serial/`, alongside the 驱动程序 for each chip family.

`ucom(4)` is distinct from `uart(4)`. It does not use `uart_ops`, it does not use `uart_bas`, and it does not use the 环形缓冲区 inside `uart_core.c`. What it does is provide a TTY abstraction on top of USB transfers. A `ucom(4)` client declares itself through a `struct ucom_回调`:

```c
struct ucom_callback {
    void (*ucom_cfg_get_status)(struct ucom_softc *, uint8_t *, uint8_t *);
    void (*ucom_cfg_set_dtr)(struct ucom_softc *, uint8_t);
    void (*ucom_cfg_set_rts)(struct ucom_softc *, uint8_t);
    void (*ucom_cfg_set_break)(struct ucom_softc *, uint8_t);
    void (*ucom_cfg_set_ring)(struct ucom_softc *, uint8_t);
    void (*ucom_cfg_param)(struct ucom_softc *, struct termios *);
    void (*ucom_cfg_open)(struct ucom_softc *);
    void (*ucom_cfg_close)(struct ucom_softc *);
    int  (*ucom_pre_open)(struct ucom_softc *);
    int  (*ucom_pre_param)(struct ucom_softc *, struct termios *);
    int  (*ucom_ioctl)(struct ucom_softc *, uint32_t, caddr_t, int,
                      struct thread *);
    void (*ucom_start_read)(struct ucom_softc *);
    void (*ucom_stop_read)(struct ucom_softc *);
    void (*ucom_start_write)(struct ucom_softc *);
    void (*ucom_stop_write)(struct ucom_softc *);
    void (*ucom_tty_name)(struct ucom_softc *, char *pbuf, uint16_t buflen,
                         uint16_t unit, uint16_t subunit);
    void (*ucom_poll)(struct ucom_softc *);
    void (*ucom_free)(struct ucom_softc *);
};
```

The methods divide into three groups. Configuration methods (names prefixed with `ucom_cfg_`) are called to change the state of the underlying chip: set DTR, set RTS, change the 波特率, and so on. These methods run in the 框架's configuration thread, which is designed for making synchronous USB control requests. Flow methods (`ucom_start_read`, `ucom_start_write`, `ucom_stop_read`, `ucom_stop_write`) are called to enable or disable the data path on the underlying USB channels. The pre-methods (`ucom_pre_open`, `ucom_pre_param`) run on the caller's context before the 框架 schedules a configuration task, which is where a 驱动程序 validates userland-supplied arguments and returns an errno if they are unacceptable. The `ucom_ioctl` method translates chip-specific userland ioctls that the 框架 does not handle into USB requests.

A USB-to-串行驱动程序's job is to implement these 回调 in terms of USB transfers. When `ucom_cfg_param` is called with a new 波特率, the 驱动程序 issues a vendor-specific 控制传输 that programs the chip's baud-rate 注册. When `ucom_start_read` is called, the 驱动程序 starts a bulk-IN channel that delivers incoming bytes. When `ucom_start_write` is called, the 驱动程序 starts a bulk-OUT channel that flushes outgoing bytes.

The FTDI 驱动程序 at `/usr/src/sys/dev/usb/serial/uftdi.c` is the concrete reference. Its `ucom_cfg_param` implementation translates the termios fields into the FTDI's proprietary baud-rate divisor format (which is weird, because FTDI chips use a sub-integer divisor scheme that is almost but not quite standard 16550) and issues a 控制传输 to `bRequest = FTDI_SIO_SET_BAUD_RATE`. Its `ucom_start_read` starts the bulk-IN channel that reads from the FTDI's RX FIFO. Its `ucom_start_write` starts the bulk-OUT channel that writes to the FTDI's TX FIFO.

From userland's perspective, a `ucom(4)` 设备 looks identical to a `uart(4)` 设备. Both appear as `/dev/ttyuN` and `/dev/cuauN`. Both respond to `stty`, `cu`, `tip`, `minicom`, and every other serial tool. Both support the same termios flags. The distinction only matters to a 驱动程序 writer.

### Reading `uftdi.c` As a Complete Example

FTDI chips (FT232R, FT232H, FT2232H, and many others) are the most widely deployed USB-to-serial chips in the embedded world. If you ever work with microcontrollers, evaluation boards, 3D printers, or industrial sensors, you will encounter FTDI hardware. FreeBSD has supported FTDI since 4.x, and the current 驱动程序 lives in `/usr/src/sys/dev/usb/serial/uftdi.c`. At roughly three thousand lines, it is not short, but most of that length is devoted to the large 匹配表 (FTDI products are legion) and to chip-variant quirks (every few years FTDI adds a new FIFO size, a new baud-rate divisor scheme, or a new 注册). The pedagogically interesting core is a few hundred lines, and reading it is a direct reward for the conceptual work of Section 4.

When you open the file, the first thing to notice is the enormous 匹配表. FTDI assigns OEM-specific USB IDs to their customers, so the 匹配表 includes not just FTDI's own VID/PID pairs but also hundreds of VIDs and PIDs from companies that embed FTDI chips in their products. Sparkfun, Pololu, Olimex, Adafruit, various industrial vendors: every one has at least one entry in the uftdi 匹配表. The `STRUCT_USB_HOST_ID` array is a few hundred entries long, grouped with comments indicating which product family each cluster belongs to.

The softc comes next. An FTDI softc includes the USB 设备 pointer, a 互斥锁, the transfer array for the bulk-IN and bulk-OUT channels (FTDI 设备 use bulk for data, not interrupt), a `struct ucom_super_softc` for the `ucom(4)` layer, a `struct ucom_softc` for the per-port state, and FTDI-specific fields: the current baud-rate divisor, the current line control 注册 contents, the current modem control 注册 contents, and a few flags for the variant family (FT232, FT2232, FT232H, and so on). Each variant requires slightly different code for some operations, so the 驱动程序 keeps a variant identifier in the softc and branches on it in the operations that differ.

The transfer configuration array is where the FTDI 驱动程序's interaction with the USB 框架 is declared. It declares two channels: `UFTDI_BULK_DT_RD` for incoming data and `UFTDI_BULK_DT_WR` for outgoing. Each is a `UE_BULK` transfer with a moderate 缓冲区 size (the FTDI default is 64 bytes for low-speed and 512 bytes for full-speed, and the 驱动程序 picks the right size at 附加 based on the chip variant). The 回调 are `uftdi_read_回调` and `uftdi_write_回调`, and they follow the three-state pattern exactly as described in Section 3.

The `ucom_回调` structure is the next important 块. It wires the FTDI 驱动程序 into the `ucom(4)` 框架. The methods it provides include `uftdi_cfg_param` (called when the 波特率 or byte format changes), `uftdi_cfg_set_dtr` (called to assert or deassert DTR), `uftdi_cfg_set_rts` (same for RTS), `uftdi_cfg_open` and `uftdi_cfg_close` (called when a userland process opens or closes the 设备), and `uftdi_start_read`, `uftdi_start_write`, `uftdi_stop_read`, `uftdi_stop_write` (called to enable or disable the data channels). Each configuration method translates a high-level operation into a USB 控制传输 to the FTDI chip.

The baud-rate programming is one of the most instructive parts of the 驱动程序, because FTDI chips use a peculiar divisor scheme. Rather than the clean integer divisors a 16550 UART uses, FTDI supports a fractional divisor where the numerator is an integer and the denominator is computed from two bits that select one-eighth, one-quarter, three-eighths, one-half, or five-eighths. The function `uftdi_encode_baudrate` takes a requested 波特率 and the chip's reference clock and computes the closest valid divisor. It handles the edge cases (very low 波特率s, very high 波特率s on newer chips, standard rates like 115200 which are exactly representable, nonstandard rates like 31250 used by MIDI). The resulting sixteen-bit value is passed to `uftdi_set_baudrate`, which issues a 控制传输 to the FTDI's baud-rate 注册.

The line control 注册 (数据位, 停止位, 奇偶校验) is programmed through a similar sequence: the termios structure arrives at `uftdi_cfg_param`, the 驱动程序 extracts the relevant bits, encodes them into the FTDI's line-control format, and issues a 控制传输.

The modem control signals (DTR, RTS) are programmed through `uftdi_cfg_set_dtr` and `uftdi_cfg_set_rts`. These are the simplest transfers: a control-out with no payload, which the chip interprets as "set DTR to X" or "set RTS to Y."

The data path is in the two 回调. `uftdi_read_回调` handles the bulk-IN channel. On `USB_ST_TRANSFERRED`, it extracts the received bytes from the USB 帧 (ignoring the first two bytes, which are FTDI status bytes) and feeds them into the `ucom(4)` layer for delivery to userland. On `USB_ST_SETUP`, it rearms the read for another 缓冲区. `uftdi_write_回调` handles the bulk-OUT channel. On `USB_ST_SETUP`, it asks the `ucom(4)` layer for more data, copies it into a USB 帧, and submits the transfer. On `USB_ST_TRANSFERRED`, it rearms to check for more data.

Reading through `uftdi.c` with Section 4 vocabulary in hand, you can see how the entire `ucom(4)` 框架 pattern is instantiated for a specific chip. The FTDI-specific logic (baud-rate encoding, line-control encoding, modem-control setting) is isolated into helper functions. The 框架 integration is handled by the `ucom_回调` structure. The data flow is handled by the two 批量传输. If you were writing a 驱动程序 for a different USB-to-serial chip, you would copy this structure and change the chip-specific parts.

The existence of this 驱动程序 explains something that might otherwise be puzzling. Why did FreeBSD add `ucom(4)` as a separate 框架 rather than as part of `uart(4)`? Because the entire data-path machinery of a `uart(4)` 驱动程序 (中断处理程序s, 环形缓冲区, 注册 accesses) has no analogue in a USB-to-serial world. The FTDI chip's "FIFO" is an on-chip 缓冲区 that the 驱动程序 cannot directly access; it can only send bulk 数据包 to the chip and receive them back. The `uart(4)` machinery would be unused overhead. By having `ucom(4)` as a separate 框架 with its own data-path abstractions, FreeBSD can make a USB-to-串行驱动程序 like `uftdi` weigh just a few hundred lines of core logic rather than wrap an unnecessary layer of 16550 emulation.

When you finish reading `uftdi.c`, open `uplcom.c` (the Prolific PL2303 驱动程序) and `uslcom.c` (the Silicon Labs CP210x 驱动程序) in sequence. They follow the same structure with different chip-specific details. After reading all three, you will have a working understanding of how a USB-to-串行驱动程序 is organised in FreeBSD, and you will be ready to write one for any chip you encounter.

### Choosing Between `uart(4)` and `ucom(4)`

The choice is mechanical. If the chip sits on the system 总线 (PCI, ISA, a platform I/O port, a memory-mapped SoC peripheral), you write a `uart(4)` 驱动程序. If the chip sits on USB and exposes a serial 接口, you write a `ucom(4)` 驱动程序.

The two 框架 do not mix. You cannot take a `uart(4)` 驱动程序 and plug it into USB, and you cannot take a `ucom(4)` 驱动程序 and 附加 it to PCIe. They are independent implementations of the same user-visible abstraction (a TTY port), but with very different internals.

Beginners sometimes ask why the two 框架 exist at all, instead of a unified serial 框架 with a pluggable transport layer. The answer is historical: `uart(4)` was rewritten in its modern form in the early 2000s to replace the older `sio(4)` 驱动程序, and at that time USB serial support was a set of ad-hoc 驱动程序. When USB serial support was unified, the natural approach was to add a thin TTY-integration layer (`ucom(4)`) rather than retrofit `uart(4)`. The two are now independent because decoupling them has been stable and useful. A unification effort would be a significant project with modest payoff.

For your purposes as a beginning 驱动程序 writer, the rule is simple. If you are writing a 驱动程序 for a chip that lives on your motherboard's 串行端口s or on a PCIe card, use `uart(4)`. If you are writing a 驱动程序 for a USB dongle that pretends to be a 串行端口, use `ucom(4)`. The reference 驱动程序 for each case (`ns8250` for `uart(4)`, `uftdi` for `ucom(4)`) are the right places to learn the details.

### Differences Between Chip Variants

Working with real UART hardware quickly teaches you that "16550-compatible" is a spectrum, not a fixed specification. Here are the variants you are most likely to encounter and the differences that matter.

**8250.** The original, from the late 1970s. Has no FIFO; every received byte must be collected by the CPU before the next arrives. Software for 16550A will usually work, with reduced performance.

**16450.** Like 8250 but with some 注册 improvements and slightly more reliable behaviour. Still no FIFO.

**16550.** Introduced a 16-byte FIFO, but the original 16550 had buggy FIFO behaviour. Software should detect this and refuse to use the FIFO in the bad case.

**16550A.** Fixed the FIFO bugs. This is the canonical "16550" that every PC 串行驱动程序 targets. Reliable, widely compatible.

**16550AF.** Further revisions for clocking and margin. For software purposes, identical to 16550A.

**16650.** Extended the FIFO to 32 bytes and added automatic hardware 流控制. Mostly 16550A-compatible.

**16750.** Extended the FIFO to 64 bytes. Some chips with this label also have additional autobaud and high-speed modes. Software must decide whether to enable the extended FIFO.

**16950 (Oxford Semiconductor).** A 128-byte FIFO, additional flow-control features, and support for unusual 波特率s through a modified divisor scheme. Often seen on high-performance PCIe serial cards.

**UART-compatible SoC controllers.** Many embedded processors have built-in UARTs that are 注册-compatible with 16550 but with quirks: some have different clock rates, some have different 注册 offsets, some have DMA support, some have different interrupt semantics. The `ns8250` 驱动程序 in FreeBSD 探测 for these variants during 附加 and adjusts its behaviour accordingly.

The `ns8250` 驱动程序's 探测 logic reads several 寄存器 to determine which variant is present. It checks the IIR bits we saw earlier, reads the FIFO control 注册 to see what FIFO size is reported, checks for 16650/16750/16950 identification markers, and records the result in a variant field in the softc. The body of the 驱动程序 then branches on this field at a few places where the variants differ.

When you write a 驱动程序 for a new UART, decide upfront whether you want to target a single variant or a family. Targeting a single variant is simpler but limits the hardware you can support. Targeting a family requires variant detection logic like `ns8250`'s.

### The Console Path

FreeBSD can use a 串行端口 as the console. This is especially useful for embedded systems that do not have a display, for servers that do not have a keyboard and monitor, and for 内核 debugging (so that `printf` output goes somewhere visible even when the display 驱动程序 is broken).

The console path is tightly integrated with `uart(4)`. A UART that is designated as the console is 探测d early in boot, before most of the 内核 is initialised. The console's putc and getc methods are used to emit boot messages and to read boot-time keyboard input. Only after the full 内核 is up does the UART get 附加ed to the TTY layer in the normal way.

Two mechanisms select which port is the console. The boot loader can set a variable (typically `console=comconsole`) in the environment, which the 内核 reads at startup. Alternatively, the 内核 can be configured at build time with a specific port as the console (via `options UART_EARLY_CONSOLE` in a 内核 configuration file).

When a port is the console, it stays active across 驱动程序 unload and 分离. You cannot unload `uart` or disable the console port without losing console output. This constraint is enforced in the `uart(4)` 框架 and is usually invisible to 驱动程序 writers (you do not need to special-case the console port), but it is worth knowing about in case you see console-related oddities during testing.

### Comparing UART Drivers Across Architectures

One of FreeBSD's strengths is that the same `uart(4)` 框架 works across multiple architectures. An `x86_64` laptop with a 16550 on a PCIe card, an `aarch64` Raspberry Pi with a PL011 on-chip UART, and a `riscv64` development board with a SiFive-specific UART all expose the same TTY 接口 to userland. Only the 驱动程序 differs.

Here is a quick survey of the UART 驱动程序 in FreeBSD 14.3:

- `uart_dev_ns8250.c`: the 16550 family for x86 and many other platforms.
- `uart_dev_pl011.c`: the ARM PrimeCell PL011 UART, used on Raspberry Pi and many ARM SoCs.
- `uart_dev_imx.c`: the NXP i.MX UART, used on i.MX-based ARM boards.
- `uart_dev_z8530.c`: the Zilog Z8530, historically used on SPARC workstations.
- `uart_dev_ti8250.c`: a TI variant of the 16550 with additional features.
- `uart_dev_pl011.c` (sbsa variant): the SBSA-standardised ARM UART for server-class ARM hardware.
- `uart_dev_snps.c`: the Synopsys DesignWare UART, used on many RISC-V boards.

Open any two of these and compare their `uart_ops` implementations side by side. The structure is identical: six methods, each pointing at a function that reads or writes chip-specific 寄存器. The chip-specific details differ, but the 框架's API is the same.

This is the payoff of the layered design. A new UART 驱动程序 is a contained project: a few hundred lines of code, reusing all the 缓冲区ing and TTY integration from the 框架. If FreeBSD had to reimplement 缓冲区ing for every UART, the system would be much larger and much harder to verify.

### What About the USB CDC ACM Standard?

USB has a standard class for serial 设备, called CDC ACM (Communication Device Class, Abstract Control Model). Chips that implement CDC ACM advertise themselves with a specific class/subclass/protocol triple at the 接口 level, and they can be driven by a single generic 驱动程序 rather than a vendor-specific one. FreeBSD's generic CDC ACM 驱动程序 is `u3g.c` in `/usr/src/sys/dev/usb/serial/`, and it is also built on top of `ucom(4)`.

Many modern USB serial chips implement CDC ACM, so the generic 驱动程序 just works for them without a vendor-specific file. Others (like FTDI) use proprietary protocols that require a vendor-specific 驱动程序. The class/subclass/protocol triple in the 接口 描述符 is what tells you which case you are in; `usbconfig -d ugenN.M dump_all_config_desc` will show it.

When you are shopping for a USB serial adapter for development work, prefer chips that implement CDC ACM. They are cheaper, more portable, and do not require proprietary 驱动程序. FTDI chips are historically dominant in embedded development because of their reliability, and FreeBSD supports them well, but a modern CP2102 or CH340G running in CDC ACM mode is equally usable.

### 总结 Section 4

Section 4 has given you a complete picture of how 串行驱动程序 work in FreeBSD. You have seen the layering: `uart(4)` at the 框架 level, `ttydevsw` at the TTY integration level, `uart_ops` at the hardware level. You have seen the distinction between `uart(4)` for 总线-附加ed UARTs and `ucom(4)` for USB-to-serial bridges, and the practical rule for deciding which to use. You have seen, at a high level, the six hardware methods a UART 驱动程序 implements, the configuration 回调 a USB-to-串行驱动程序 implements, and how the TTY layer sits on top of both with one uniform 接口 to userland.

The level of depth in this section is necessarily lighter than the USB-side sections, because 串行驱动程序 in FreeBSD are more specialised than USB驱动程序 and you are more likely to read an existing one (or modify one) than to write a brand-new one from scratch. If you do find yourself writing a new UART 驱动程序 for a custom board, the path is clear: open `ns8250` in one window, open your chip's data sheet in another, and write the six methods one by one.

Two key takeaways 帧 Section 5. First, testing 串行驱动程序 does not require real hardware. FreeBSD ships a `nmdm(4)` null-modem 驱动程序 that creates pairs of virtual TTYs you can wire together, letting you exercise termios changes, 流控制, and data flow without plugging in anything. Second, testing USB驱动程序 without hardware is harder but not impossible: you can use QEMU with USB redirection to test against real 设备 through a VM, or you can use FreeBSD's USB gadget mode to make one machine present itself as a USB 设备 to another. Section 5 covers both. The goal is to enable a development loop that does not depend on cable-handling and on plugging things in and out.

## 第5节： Testing USB和串行驱动程序 Without Real Hardware

### Why This Section Exists

A beginning 驱动程序 writer often gets stuck at the same obstacle. They write a 驱动程序, compile it, want to try it, and discover they do not have the hardware, the hardware is behaving badly, the hardware is on the wrong machine, or the iteration loop of "change code, plug it in, see what happens, unplug it, change code again" is painfully slow and unreliable. Section 5 addresses this directly. FreeBSD provides several mechanisms that let you exercise 驱动程序 code paths without physical hardware, and knowing these mechanisms will save you hours of frustration.

The goal is not to pretend hardware is present when it is not. The goal is to give you tools that cover the parts of 驱动程序 development where hardware is incidental, so that when you do plug in real hardware, you already know your code path logic is correct and you are only validating the physical interaction. Debugging a 注册-level quirk is faster when you know that your locking, your state machines, and your user 接口 are already sound.

This section covers four such mechanisms: the `nmdm(4)` null-modem 驱动程序 for serial testing, basic userland tools for exercising TTYs (`cu`, `tip`, `stty`, `comcontrol`), QEMU with USB redirection for USB驱动程序 testing, and FreeBSD's USB gadget mode for presenting one machine as a USB 设备 to another. It ends with a short discussion of techniques that do not require any special tooling: unit tests at the functional layer, logging discipline, and assertion-driven development.

### The `nmdm(4)` Null-Modem Driver

`nmdm(4)` is a 内核模块 that creates pairs of linked virtual TTYs. When you write to one side, it comes out the other side, exactly as if you had connected two real 串行端口s with a null-modem cable. The 驱动程序 is in `/usr/src/sys/dev/nmdm/nmdm.c`, and it is loaded with:

```console
# kldload nmdm
```

Once loaded, you can instantiate pairs on demand simply by opening them. Run:

```console
# cu -l /dev/nmdm0A
```

This opens the `A` side of pair `0`. On another terminal, run:

```console
# cu -l /dev/nmdm0B
```

Whatever you type into one `cu` session will appear in the other. You have now created a pair of virtual TTYs, with no hardware involved. You can change 波特率s with `stty` and the change will be noticed on both sides. You can assert DTR and CTS through ioctls and see the effect on the other side.

The utility of `nmdm(4)` for 驱动程序 development is twofold. First, if you are writing a TTY-layer user (say, a 驱动程序 that spawns a shell on a virtual TTY, or a userland program that implements a protocol over a TTY), you can test it end-to-end against `nmdm(4)` without any hardware. Second, if you are writing a `ucom(4)` or `uart(4)` 驱动程序, you can compare its behaviour to `nmdm(4)`'s behaviour by running the same userland test against both. If your 驱动程序 misbehaves where `nmdm(4)` does not, the bug is in your 驱动程序; if both misbehave, the bug is probably in your userland test.

A small caveat: `nmdm(4)` does not simulate 波特率 delays. Whatever you write comes out the other side at memory speed. This is usually what you want (you do not want to wait through a real 9600-baud transmission for a hundred-kilobyte test payload), but it does mean that timing-sensitive protocols cannot be tested with `nmdm(4)` alone.

### The `cu(1)`, `tip(1)`, and `stty(1)` Toolbox

Whether you are using `nmdm(4)`, a real UART, or a USB-to-serial dongle, the userland tools you use to interact with a TTY are the same. The most important three are `cu(1)`, `tip(1)`, and `stty(1)`.

`cu` is the classic "call up" program. It opens a TTY, puts the terminal into raw mode, and lets you type bytes to the port and see bytes coming back. To open a port at a specific 波特率:

```console
# cu -l /dev/cuau0 -s 115200
```

The `-l` argument specifies the 设备, and `-s` specifies the 波特率. `cu` supports a handful of escape sequences (all starting with `~`) for exiting, sending files, and similar operations; `~.` is the standard "exit" escape and `~?` lists the others.

`tip` is a related tool with similar semantics but a different configuration mechanism. `tip` reads `/etc/remote` for named connection entries and can take a name argument rather than a 设备 path. For most purposes, `cu` and `tip` are interchangeable; `cu` is more convenient for one-off use.

`stty` prints or changes the termios parameters of a TTY. Run `stty -a -f /dev/ttyu0` to see every termios flag on the port. Run `stty 115200 -f /dev/ttyu0` to set the 波特率. Run `stty cs8 -parenb -cstopb -f /dev/ttyu0` to set eight 数据位, no 奇偶校验, one stop bit (the most common configuration in modern embedded work). The manual page is extensive, and the flags map almost directly onto the bits of `c_cflag`, `c_iflag`, `c_lflag`, and `c_oflag` in the `termios` struct.

Using these three tools together gives you a flexible way to poke at your 驱动程序 from userland. You can change settings with `stty`, open the port with `cu`, send and receive bytes, close the port, check the state with `stty` again, and repeat. If your 驱动程序's `tsw_param` implementation has a bug, `stty` will expose it: the settings you set will not read back correctly, or the port will behave differently than requested.

### The `comcontrol(8)` Utility

`comcontrol` is a specialised utility for 串行端口s. It sets port-specific parameters that are not exposed through termios. The two most important are the `drainwait` and the specific-RS-485 options. For beginner 驱动程序 testing, the more common use is inspecting port state: `comcontrol /dev/ttyu0` shows the current modem signals (DTR, RTS, CTS, DSR, CD, RI) and the current `drainwait`. You can also set the signals:

```console
# comcontrol /dev/ttyu0 dtr rts
```

sets DTR and RTS. This is useful for testing flow-control handling without writing a custom program.

### The `usbconfig(8)` Utility

On the USB side, `usbconfig(8)` is the Swiss Army knife. You used it at the end of Section 1 to inspect a 设备's 描述符. Several other subcommands are useful during 驱动程序 development:

- `usbconfig list`: list all 附加ed USB 设备.
- `usbconfig -d ugenN.M dump_all_config_desc`: print every 描述符 for a 设备.
- `usbconfig -d ugenN.M dump_设备_quirks`: print any quirks applied by the USB 框架.
- `usbconfig -d ugenN.M dump_stats`: print per-transfer statistics.
- `usbconfig -d ugenN.M suspend`: put the 设备 into the USB suspend state.
- `usbconfig -d ugenN.M resume`: wake it up.
- `usbconfig -d ugenN.M reset`: physically reset the 设备.

The `reset` command is particularly useful during development. A 驱动程序 under test can easily leave a 设备 in a confused state; `usbconfig reset` puts the 设备 back to the just-plugged-in condition without requiring a physical unplug.

### Testing USB Drivers with QEMU

QEMU, the generic CPU emulator, has strong USB support. You can run a FreeBSD guest inside QEMU and redirect real host USB 设备 into the guest. This is the single most useful technique for USB驱动程序 development, because it lets you test against real hardware while retaining all the iteration speed of working inside a VM.

On a FreeBSD host, install QEMU from ports:

```console
# pkg install qemu
```

Install a FreeBSD guest image into a disk file (the mechanics are covered in Chapter 4 and Appendix A). When you boot the guest, add USB redirection options:

```console
qemu-system-x86_64 \
  -drive file=freebsd.img,format=raw \
  -m 1024 \
  -device nec-usb-xhci,id=xhci \
  -device usb-host,bus=xhci.0,vendorid=0x0403,productid=0x6001
```

The `-设备 nec-usb-xhci` line adds a USB 3.0 controller to the guest. The `-设备 usb-host` line 附加 a specific USB 设备 from the host (identified by vendor and product) to that controller. When the guest boots, the 设备 appears on the guest's USB 总线 and can be enumerated by the guest's 内核.

This setup gives you the full iteration loop inside the VM. You can load your 驱动程序, unload it, reload a rebuilt version, all without physically handling any cables. You can use serial console or networking to interact with the VM. You can snapshot the VM state before a risky test and revert if the test panics.

The main limitation is USB isochronous support, which is less stable across emulators. For bulk, interrupt, and 控制传输 (the three types most 驱动程序 use), QEMU USB redirection is reliable enough to be your primary development environment.

### FreeBSD USB Gadget Mode

If QEMU is not available and you have two FreeBSD machines, there is another option: `usb_template(4)` and the dual-role USB support on some hardware let you make one machine present itself as a USB 设备 to another. The host machine sees a normal USB peripheral; the gadget machine is actually running the 设备 side of the USB protocol.

This is an advanced topic and the hardware support is variable. On x86 platforms with USB-on-the-Go-capable chipsets, on some ARM boards, and on specific embedded configurations, the setup works. On most desktop hardware, it does not. The gory details are in `/usr/src/sys/dev/usb/template/` and in the `usb_template(4)` manual page.

If you have the hardware to use this technique, it is the closest thing to a full end-to-end USB驱动程序 test without physical peripherals. If you do not, do not pursue it for a learning project; use QEMU instead.

### Techniques That Do Not Require Special Tooling

Beyond the 框架 above, there are several techniques that rely only on good 驱动程序 design.

First, design your 驱动程序 so that the hardware-independent parts can be unit-tested in userland. If your 驱动程序 has a protocol parser, a state machine, or a checksum calculator, factor those into functions that take plain C 缓冲区 and return plain C results. You can then compile those functions into a userland test program and run them against known inputs. This catches many bugs before they reach the 内核.

Second, log aggressively during development and quietly in production. The `DLOG_RL` macro from 第25章 is your friend: it lets you emit frequent diagnostic messages during development, with a sysctl to suppress them in production. Rate-limiting prevents log storms if something goes wrong.

Third, use assertions for invariants. `KASSERT(cond, ("message", args...))` will panic the 内核 if `cond` is false, but only in `INVARIANTS` 内核s. You can run your 驱动程序 in an `INVARIANTS` 内核 during development and in a production 内核 later, without changing the code. The Chapter 20 discussion of `INVARIANTS` is the reference.

Fourth, be rigorous about concurrency testing. Use `INVARIANTS` plus `WITNESS` (which tracks lock ordering) during development. If your 驱动程序 has a locking bug that almost always works but occasionally deadlocks, `WITNESS` will catch it on the first occurrence.

Fifth, write a simple userland client for your 驱动程序 and use it as part of your development loop. Even a ten-line program that opens the 设备, writes a known string, reads a known response, and checks the result is enormously useful. You can run it in a loop during stress testing, you can run it with `ktrace -f cmd` to get a trace of system calls, and you can run it under a debugger if something surprises you.

### A Walkthrough of QEMU USB Redirection

QEMU's USB support is the single most useful tool for USB驱动程序 development, so a more detailed walkthrough is in order. Suppose you want to develop a 驱动程序 for a specific FT232 adapter. Your host is a FreeBSD 14.3 machine, and you want to run your 驱动程序 on a guest FreeBSD 14.3 VM inside QEMU.

First, install QEMU and create a guest disk image:

```console
# pkg install qemu
# truncate -s 16G guest.img
```

Install FreeBSD into the image. The exact procedure is covered in Appendix A, but the short version is: boot a FreeBSD installer ISO as the CD-ROM, install onto the disk image, reboot.

Once the guest is installed, locate the host USB 设备 you want to redirect. Plug in the FT232 and note the vendor and 产品IDs from `usbconfig list`:

```text
ugen0.3: <FTDI FT232R USB UART> at usbus0
```

`usbconfig -d ugen0.3 dump_设备_desc` will show `idVendor = 0x0403` and `idProduct = 0x6001`.

Now start QEMU with USB redirection:

```console
qemu-system-x86_64 \
  -enable-kvm \
  -cpu host \
  -m 2048 \
  -drive file=guest.img,format=raw \
  -device nec-usb-xhci,id=xhci \
  -device usb-host,bus=xhci.0,vendorid=0x0403,productid=0x6001 \
  -net user -net nic
```

The `-设备 nec-usb-xhci` line adds a USB 3.0 controller to the VM. The `-设备 usb-host` line redirects the matching host 设备 into the VM. When the VM boots, the FT232 will appear as if it were plugged directly into the VM's USB port.

Inside the VM, run `dmesg` and look for the USB 附加:

```text
uhub0: 4 ports with 4 removable, self powered
uftdi0 on usbus0
uftdi0: <FTDI FT232R USB UART, class 255/0, rev 2.00/6.00, addr 2> on usbus0
```

Your 驱动程序 (whether `uftdi` or your own work-in-progress) will see a real FT232 with real 描述符, real transfer behaviour, and real quirks. You can unload and reload your 驱动程序 inside the VM without disconnecting anything; you can run 内核 with `INVARIANTS` and `WITNESS` without worrying about host-side impact; you can snapshot the VM and revert if a test goes badly.

A few subtleties to be aware of with USB redirection:

- Only one consumer can claim a USB 设备 at a time. If you redirect a 设备 into a VM, the host loses access to it until the VM releases it. This matters if you are redirecting something like a USB keyboard or mouse; choose a spare 设备 for development.

- USB 等时传输 have some quirks in QEMU. They work, but timing can be slightly off. For most 驱动程序 development, you will be working with bulk, interrupt, and 控制传输, so this is rarely a concern.

- Some 主机控制器s (particularly xHCI) can reset under heavy I/O. If your 驱动程序 behaves strangely under stress testing, try with a different `-设备` type (uhci, ehci, xhci) to see whether the issue is in your 驱动程序 or in the emulated controller.

- USB 3.0 SuperSpeed transfers are more reliable with `-设备 nec-usb-xhci`. Older `-usb` flag-based controllers are limited to USB 2.0.

When the VM is running, the iteration cycle becomes: edit code on the host, copy to the VM (or 挂载 a shared directory), build inside the VM, load, test, reload, repeat. A Makefile with a `test:` target that does all of this can cut iteration time to tens of seconds.

### Using `devd(8)` During Development

`devd(8)` is FreeBSD's 设备-event daemon. It reacts to 内核 notifications about 设备 附加 and 分离 and can run configured commands in response. During 驱动程序 development, `devd` is useful in two ways.

First, it can auto-load your module when a matching 设备 is plugged in. If your module is in `/boot/modules/` and your `USB_PNP_HOST_INFO` is set, `devd` will run `kldload` automatically when it sees a 设备 that would match.

Second, it can run diagnostic commands on 附加. A `/etc/devd.conf` entry like:

```text
attach 100 {
    device-name "myfirst_usb[0-9]+";
    action "logger -t myfirst-usb 'device attached: $device-name'";
};
```

will write a log line every time a `myfirst_usb` 设备 附加. For more elaborate diagnostics, you can invoke your own shell script that dumps state, starts userland consumers, or sends notifications.

During development, a useful pattern is to have `devd` open a `cu` session to a newly 附加ed `ucom` 设备, so you can exercise the 驱动程序 the moment it 附加:

```text
attach 100 {
    device-name "cuaU[0-9]+";
    action "setsid screen -dmS usb-serial cu -l /dev/$device-name -s 9600";
};
```

This runs the test in a 分离ed `screen` session, which you can later 附加 to with `screen -r usb-serial`.

### Writing a Simple Userland Test Harness

Most 驱动程序 bugs are exposed by actually running the 驱动程序 against userland. Even a short test program catches more bugs than reading the 驱动程序's code carefully. For our echo 驱动程序, a minimal test program looks like:

```c
#include <fcntl.h>
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <unistd.h>

int
main(int argc, char **argv)
{
    int fd;
    const char *msg = "hello";
    char buf[64];
    int n;

    fd = open("/dev/myfirst_usb0", O_RDWR);
    if (fd < 0) {
        perror("open");
        return (1);
    }

    if (write(fd, msg, strlen(msg)) != (ssize_t)strlen(msg)) {
        perror("write");
        close(fd);
        return (1);
    }

    n = read(fd, buf, sizeof(buf) - 1);
    if (n < 0) {
        perror("read");
        close(fd);
        return (1);
    }
    buf[n] = '\0';
    printf("got %d bytes: %s\n", n, buf);

    close(fd);
    return (0);
}
```

Compile with `cc -o lab03-test lab03-test.c`. Run with `./lab03-test`. The expected output is "got 5 bytes: hello".

Extensions to this test harness that catch more bugs:

- Loop the open/write/read/close cycle a thousand times. Memory leaks and resource leaks show up after a few hundred iterations.
- Fork multiple processes and have them all read/write concurrently. Race conditions manifest as random data corruption or deadlocks.
- Intentionally kill the test process mid-transfer. Driver-side state machines sometimes get confused when a userland consumer disappears unexpectedly.
- Send random-length writes (1 byte, 10 bytes, 100 bytes, 1 KB, 10 KB). Edge cases around short and long transfers are where many subtle bugs live.

Build these extensions incrementally. Each one will probably reveal a bug the previous version did not; each bug you fix will make your 驱动程序 more ro总线t.

### Logging Patterns for Development

During development, you want verbose logging. In production, you want silence. The pattern from 第25章 (`DLOG_RL` with a sysctl to control verbosity) carries over unchanged to USB and UART 驱动程序. Define a rate-limited logging macro that compiles to a no-op in production builds, and sprinkle it through every branch that might be interesting during debugging:

```c
#ifdef MYFIRST_USB_DEBUG
#define DLOG(sc, fmt, ...) \
    do { \
        if (myfirst_usb_debug) \
            device_printf((sc)->sc_dev, fmt "\n", ##__VA_ARGS__); \
    } while (0)
#else
#define DLOG(sc, fmt, ...) ((void)0)
#endif
```

Then in the 回调:

```c
case USB_ST_TRANSFERRED:
    DLOG(sc, "bulk read completed, actlen=%d", actlen);
    ...
```

Control `myfirst_usb_debug` through a sysctl:

```c
static int myfirst_usb_debug = 0;
SYSCTL_INT(_hw_myfirst_usb, OID_AUTO, debug, CTLFLAG_RWTUN,
    &myfirst_usb_debug, 0, "Enable debug logging");
```

Now you can turn logging on and off at runtime with `sysctl hw.myfirst_usb.debug=1`. During development, turn it on. During stress tests, turn it off (logging rate-limiting helps, but zero logging is even cheaper). During post-mortem analysis of a bug, turn it on and reproduce.

### A Test-Driven Workflow for 第26章

For the hands-on labs coming in the next section, a good workflow looks like this:

1. Write the 驱动程序 code.
2. Compile it. Fix build errors.
3. Load it in a test VM. Observe `dmesg` for 附加 failures.
4. Run a small userland client that exercises the 驱动程序's I/O paths.
5. Unload. Make a change. Go back to step 2.
6. Once the 驱动程序 behaves well in the VM, test it on real hardware as a sanity check.

Most of the time spent on this loop is in steps 1 through 4. Real hardware testing in step 6 is a validation step, not an iteration step. If you try to iterate on real hardware, you will waste time on plug-unplug cycles and on recovering from accidental misconfigurations; the VM saves you this.

A fresh install of FreeBSD in a small VM, configured to boot quickly and to have your 驱动程序's build directory 挂载ed as a shared 文件系统, is a highly productive development environment. Spending half a day to set one up pays back many times over in the days that follow.

### 总结 Section 5

Section 5 has given you the tools to develop USB and 串行驱动程序 without being tied to specific physical hardware. `nmdm(4)` covers the serial-port side for any test that does not need a real modem. QEMU USB redirection covers the USB side for nearly any 驱动程序 you might write. The `cu`, `tip`, `stty`, `comcontrol`, and `usbconfig` utilities give you the userland tools to exercise 驱动程序 code paths by hand. And the general techniques, from factoring hardware-independent code into userland-testable functions to using `INVARIANTS` and `WITNESS` for 内核-time correctness checking, work regardless of what transport you are writing for.

Having reached the end of Section 5, you have everything you need to start writing real USB and 串行驱动程序 for FreeBSD 14.3. The conceptual models, the code skeletons, the transfer mechanics, the TTY integration, and the testing environment are all in place. What remains is practice, which is the purpose of the next section.

## Common Patterns Across Transport Drivers

Now that we have walked through USB and serial in detail, it is worth stepping back and noting the patterns that recur. These patterns appear in 网络驱动程序 (第28章), 块 驱动程序 (第27章), and most other transport-specific 驱动程序 in FreeBSD. Recognising them saves time when you read a new 驱动程序.

### Pattern 1: Match Table, Probe, Attach, Detach

Every transport 驱动程序 begins with a 匹配表 describing which 设备 it supports. Every transport 驱动程序 has a 探测 method that tests a candidate against the 匹配表 and returns zero or `ENXIO`. Every transport 驱动程序 has an 附加 method that takes ownership of a matched 设备 and allocates all per-设备 state. Every transport 驱动程序 has a 分离 method that releases everything the 附加 method allocated, in reverse order.

The specifics vary. USB 匹配表s use `STRUCT_USB_HOST_ID`. PCI 匹配表s use `pcidev(9)` entries. ISA 匹配表s use resource descriptions. The content differs, but the structure is identical.

When you read a new 驱动程序, the first thing to find is the 匹配表. It tells you what hardware the 驱动程序 supports. The second thing to find is the 附加 method. It tells you what resources the 驱动程序 owns. The third thing to find is the 分离 method. It tells you the shape of the 驱动程序's resource hierarchy.

### Pattern 2: Softc As The Single Source of Per-Device State

Every transport 驱动程序 has a per-设备 softc. Every piece of mutable state lives in the softc. No global variables are used to hold per-设备 state (global configuration like module flags is fine). This pattern keeps multi-设备驱动程序 correct without surprise.

The softc's size is declared in the 驱动程序 structure. The 框架 allocates and frees the softc automatically. The 驱动程序 accesses it through `设备_get_softc(dev)` inside New总线 methods and through whatever 框架 helper (like `usbd_xfer_softc`) is appropriate in 回调.

Adding a new feature to a 驱动程序 often means adding a new field to the softc, a new initialisation step in 附加, a new cleanup step in 分离, and whatever code uses the field in between. When you structure changes this way, you rarely forget to clean things up, because the shape of the change makes the cleanup step obvious.

### Pattern 3: Labelled-Goto Cleanup Chain

When an 附加 method has to allocate several resources, each allocation has a failure path that unwinds all previous allocations. The labelled-goto chain from 第25章 implements this uniformly. Every resource has a label corresponding to "the state where this resource has been successfully allocated." A failure at any point jumps to the label for the state just before, which cleans up in reverse order.

This pattern is not aesthetically pleasing to some programmers (C's `goto` has a bad reputation), but it is pragmatically the cleanest way to handle an arbitrary number of cleanup steps in C. Alternatives like wrapping each resource in a separate function with its own cleanup are often more verbose. Alternatives like setting a flag per resource and testing it in a common cleanup routine add error-prone state management.

Whatever you think of `goto`, FreeBSD 驱动程序 use the labelled-goto pattern, and new 驱动程序 are expected to follow the convention.

### Pattern 4: Frameworks Hide Transport Details

Each transport has a 框架 that hides transport-specific details behind a uniform API. The USB 框架 hides DMA 缓冲区 management behind `usb_page_cache` and `usbd_copy_in/out`. The UART 框架 hides interrupt dispatching behind `uart_ops`. The network 框架 (第28章) will hide 数据包 缓冲区 management behind mbufs and `ifnet(9)`.

The value of these 框架 is that 驱动程序 become smaller and more portable. A 200-line UART 驱动程序 that supports dozens of chip variants would be impossible without the 框架. A 500-line USB驱动程序 that supports a complex protocol like USB audio would likewise be out of reach.

When you read a new 驱动程序, the parts you find most dense are usually the chip-specific logic. The parts that seem almost absent (the transfer scheduling, the 缓冲区 management, the TTY integration) are where the 框架 is doing its work.

### Pattern 5: Callbacks with State Machines

The USB 回调's three-state machine (`USB_ST_SETUP`, `USB_ST_TRANSFERRED`, `USB_ST_ERROR`) is the canonical example, but similar patterns appear in other transport 驱动程序. A 网络驱动程序's transmit completion 回调 has a similar structure. A 块 驱动程序's request completion 回调 is similar. The 框架 calls the 驱动程序 back at well-defined moments, and the 驱动程序 uses a state machine to decide what to do.

Learning to read these state machines is learning a universal 驱动程序-reading skill. The specific states differ from 框架 to 框架, but the pattern is recognisable.

### Pattern 6: Mutexes and Wakeups

Every 驱动程序 protects its softc with a 互斥锁. Userland-facing code (read, write, ioctl) takes the 互斥锁 while manipulating softc fields. Callback code runs with the 互斥锁 held (the 框架 acquires it before calling). Userland code releases the 互斥锁 before sleeping and reacquires it after waking. Wakeup calls from 回调 release any sleeper waiting on the relevant channel.

The specifics vary by transport, but the pattern is universal. Modern FreeBSD 驱动程序 are uniformly multithreaded and multi-CPU safe, which requires disciplined locking.

### Pattern 7: Errno-Returning Helpers

第25章 introduced the errno-returning helper function pattern: every internal function returns an integer errno (zero for success, nonzero for failure). Callers check the return value and propagate failure up through the stack. The 附加 method accumulates successful helpers in the labelled-goto chain; each helper's failure triggers the cleanup corresponding to its position.

This pattern requires discipline. Every helper must be consistent; no helper can return a "success value" that varies in meaning, and no helper can use global state to communicate failure. When followed rigorously, the pattern produces 驱动程序 where the control flow is legible and the error paths are easy to audit.

### Pattern 8: Version Declarations and Module Dependencies

Every 驱动程序 module declares its own version with `MODULE_VERSION`. Every 驱动程序 module declares its dependencies with `MODULE_DEPEND`. Dependencies are versioned ranges (minimum, preferred, maximum), which allows parallel development of 框架 and 驱动程序 to proceed without lockstep releases.

When a new major version of a 框架 is released with breaking API changes, the version range in `MODULE_DEPEND` is how a 驱动程序 expresses "I work with 框架 v1 or v2, but not v3." The 内核模块 loader refuses to load a 驱动程序 whose dependencies cannot be satisfied, which prevents many classes of silent breakage.

### Pattern 9: Cross-Framework Layering

Some 驱动程序 sit on top of multiple 框架. A USB-to-以太网 驱动程序 sits on top of `usbdi(9)` (for USB transfers) and `ifnet(9)` (for 网络接口 semantics). A USB-to-串行驱动程序 sits on top of `usbdi(9)` and `ucom(4)`. A USB mass-存储驱动程序 sits on top of `usbdi(9)` and CAM.

When you write a cross-框架 驱动程序, the structure is: you write 回调 for each 框架, and you orchestrate the interaction between them in your 驱动程序's helper code. The 框架 on top of which you sit defines how userland sees your 驱动程序. The 框架 below handles the transport.

Reading `uftdi.c` showed you this pattern: the 驱动程序 is a USB驱动程序 (it uses `usbdi(9)`) and a 串行驱动程序 (it uses `ucom(4)`), and the orchestration between the two is the heart of the file.

### Pattern 10: Early Attach Deferral

Some 驱动程序 cannot finish their 附加 work synchronously. For example, a 驱动程序 might need to read a configuration EEPROM that takes a few hundred milliseconds, or it might need to wait for a PHY to autonegotiate a link. These 驱动程序 use a deferred-附加 pattern: the New总线 附加 method queues a taskqueue task that does the slow work, then returns quickly.

This pattern keeps the system boot fast (no single 驱动程序 holds up boot by taking a long time in 附加) and lets 驱动程序 do their work asynchronously. The caller must be aware that 附加 "finishing" does not mean the 设备 is fully usable; a separate "ready" state has to be polled or signalled.

For USB and UART 驱动程序, 附加 is usually fast enough that deferral is not needed. For more complex 驱动程序 (network cards in particular), deferral is common. 第28章 will show an example.

### Pattern 11: Separate Data Path and Control Path

In every transport 驱动程序, two conceptual paths exist: the control path (configuration, state changes, error recovery) and the data path (the actual bytes moving through the 设备). Most 驱动程序 structure these as separate code paths, sometimes with separate locking.

The control path is low-bandwidth and infrequent. It can afford heavy locking and synchronous calls. The data path is high-bandwidth and continuous. It must be optimised for throughput: minimal locking, no synchronous calls, efficient 缓冲区 management.

The USB 框架 keeps them naturally separate: configuration through `usbd_transfer_setup` and 控制传输; data through bulk and 中断传输. The UART 框架 likewise: configuration through `tsw_param`; data through the 中断处理程序 and 环形缓冲区. Network 驱动程序 have the most pronounced separation: configuration through ioctls; data through the TX and RX queues.

Reading a new 驱动程序, knowing this separation exists helps you parse what each code 块 is doing. A function with extensive locking and error handling is probably control path. A function with short, tight code and careful 缓冲区 management is probably data path.

### Pattern 12: Reference Drivers

Every transport in FreeBSD has one or two "canonical" reference 驱动程序 that illustrate the patterns correctly and thoroughly. For USB, `uled.c` and `uftdi.c` are the references. For UART, `uart_dev_ns8250.c` is the reference. For networking, `em` (Intel 以太网) and `rl` (Realtek) are the references. For 块设备, `da` (direct-access storage) is the reference.

When you need to understand how to write a new 驱动程序 in an existing transport, the reference 驱动程序 is the right place to start. Do not try to understand the 框架 from its code alone; that is too abstract. Start from a working 驱动程序 and let it ground your understanding.

## 动手实验

These labs give you a chance to turn the reading into muscle memory. Each lab is designed to fit in a single sitting, ideally under an hour. They assume a FreeBSD 14.3 lab environment (either on physical hardware or inside a virtual machine), root access, and a working build environment as described in Chapter 3. The companion files for every lab in this chapter are available under `examples/part-06/ch26-usb-serial/` in the book's repository.

The labs build on each other but do not strictly depend on each other. You can skip a lab and come back to it later without losing continuity. The first three labs focus on USB; the last three focus on serial. Each lab has the same structure: a short summary, the steps, expected output, and a "what to watch for" note that highlights the learning goal.

### Lab 1: Exploring a USB Device with `usbconfig`

This lab exercises the 描述符 vocabulary from Section 1 by inspecting real USB 设备 on your machine. It does not involve writing any code.

**Goal.** Read the 描述符 of three different USB 设备 and identify their 接口 class, the number of 端点, and the 端点 types.

**Requirements.** A FreeBSD system with at least three USB 设备 plugged in. If you only have one machine and few USB ports, a USB hub with a few small peripherals (mouse, keyboard, flash drive) is ideal.

**Steps.**

1. Run `usbconfig list` as root. Record the `ugenN.M` identifiers of three 设备.

2. For each 设备, run:

   ```
   # usbconfig -d ugenN.M dump_all_config_desc
   ```

   Read through the output. Identify the `bInterfaceClass`, `bInterfaceSubClass`, and `bInterfaceProtocol` for each 接口. For each 端点 in each 接口, record the `bEndpointAddress` (including direction bit), the `bmAttributes` (including transfer type), and the `wMaxPacketSize`.

3. Build a small table. For each 设备, write down: 供应商ID, 产品ID, 接口 class (with name from the USB class list), number of 端点, and the transfer type of each 端点.

4. Match your table against `dmesg`. Confirm that the 驱动程序 that claimed each 设备 makes sense given the 接口 class you recorded.

5. Optional: repeat the exercise for a 设备 you have not seen before (someone else's keyboard, a USB audio 接口, a game controller). The more variety you see, the faster 描述符 reading becomes.

**Expected output.** A filled-in table with at least three rows. The exercise is successful if you can answer, for any 设备 in the table: "What class of 驱动程序 would handle this?"

**What to watch for.** Pay attention to 设备 that expose multiple 接口. A webcam, for example, often has an audio 接口 (for the microphone) in addition to its video 接口. A multi-function printer might expose a printer 接口, a scanner 接口, and a mass-storage 接口. Noticing these is what trains your eye for the multi-接口 logic in the `探测` method.

### Lab 2: Building and Loading the USB Driver Skeleton

This lab walks through building the skeleton 驱动程序 from Section 2, loading it, and observing its behaviour when a matching 设备 is plugged in.

**Goal.** Compile and load `myfirst_usb.ko`, and observe its 附加 and 分离 messages.

**Requirements.** The build environment from Chapter 3. The files under `examples/part-06/ch26-usb-serial/lab02-usb-skeleton/`. A USB 设备 whose vendor/product you can match. For development, a VOTI/OBDEV test VID/PID (0x16c0/0x05dc) is free to use; otherwise, pick a cheap prototyping 设备 (like an FT232 breakout board) and adjust the 匹配表 to match its IDs.

**Steps.**

1. Enter the lab directory:

   ```
   # cd examples/part-06/ch26-usb-serial/lab02-usb-skeleton
   ```

2. Read `myfirst_usb.c` and `myfirst_usb.h`. Identify the 匹配表, the 探测 method, the 附加 method, the softc, and the 分离 method. For each, trace how it relates to the Section 2 walkthrough.

3. Build the module:

   ```
   # make
   ```

   You should see `myfirst_usb.ko` created in the build directory.

4. Load the module:

   ```
   # kldload ./myfirst_usb.ko
   ```

   Run `kldstat | grep myfirst_usb` to confirm the module is loaded.

5. Plug in a matching 设备. Observe `dmesg`. You should see a line like:

   ```
   myfirst_usb0: <Vendor Product> on uhub0
   myfirst_usb0: attached
   ```

   If the 设备 does not match, nothing will happen. In that case, open `usbdevs` on the target machine, find the vendor/product of a 设备 you do have, and edit the 匹配表 accordingly. Rebuild, reload, and try again.

6. Unplug the 设备. Observe `dmesg`. You should see the 内核 remove the 设备. Your `分离` does not log anything explicitly in this minimal skeleton, but you can add a `设备_printf(dev, "分离ed\n")` if you want confirmation.

7. Unload the module:

   ```
   # kldunload myfirst_usb
   ```

**Expected output.** Attach messages in `dmesg` when the 设备 is plugged in. Clean unload with no panics when the module is removed.

**What to watch for.** If `kldload` fails with an error about symbol lookups, you probably forgot a `MODULE_DEPEND` line or misspelled a symbol name. If `附加` is never called but the 设备 is definitely present, the 匹配表 is wrong: check the vendor and 产品IDs in `usbconfig list` and verify they match what you wrote in `myfirst_usb_devs`. If `附加` is called but fails, check `设备_printf` output for the failure reason.

### Lab 3: A Bulk Loopback Test

This lab adds the transfer mechanics from Section 3 to the skeleton from Lab 2 and sends a few bytes through a USB 设备 that implements a loopback protocol. It is the first lab that actually moves data.

**Goal.** Add a bulk-OUT and bulk-IN channel to the 驱动程序, write a small userland client that sends a string and reads it back, and observe the roundtrip.

**Requirements.** A USB 设备 that implements bulk loopback. The simplest such 设备 for development is a USB gadget controller running a loopback program (possible on some ARM boards and on some development kits). If you do not have one, you can substitute a simpler exercise: 附加 the 驱动程序 to a USB flash drive, open one of its `ugen` 端点, and simply armed-submit-complete a single read transfer. The loop will fail (because flash drives do not echo data), but the mechanics of setup and submission will run correctly.

**Steps.**

1. Copy `lab02-usb-skeleton` to `lab03-bulk-loopback` as a working copy.

2. Add the bulk channels to the 驱动程序. Paste the config array from Section 3, the 回调 functions, and the userland interaction. Make sure the `/dev` entry your 驱动程序 creates supports `read(2)` and `write(2)`, which are what the lab test program uses.

3. Rebuild and reload the module.

4. Run the userland client:

   ```
   # ./lab03-test
   ```

   which you will find alongside the 驱动程序 in the lab directory. The program opens `/dev/myfirst_usb0`, writes "hello", reads up to 16 bytes, and prints them. If loopback works, the output is "hello".

5. Observe `dmesg` for any stall warnings or error messages.

**Expected output.** "hello" echoed back. If the remote 设备 does not implement loopback, the read will return after the channel's timeout with no data, which is also a valid test outcome for the purposes of exercising the state machine.

**What to watch for.** The most common mistake in this lab is mismatched 端点 directions. Remember: `UE_DIR_IN` means "the host reads from the 设备" and `UE_DIR_OUT` means "the host writes to the 设备". If you swap them, the transfers will fail with stalls. Watch also for missing locking around the userland read/write handlers; if you manipulate the transmit queue without the softc 互斥锁 held, you can race with the write 回调 and see bytes disappear.

### Lab 4: A Simulated Serial Driver with `nmdm(4)`

This lab is not about writing a 驱动程序; it is about learning the userland half of serial testing. The results will inform how you approach Lab 5 and how you debug any TTY-layer work in the future.

**Goal.** Create a pair of `nmdm(4)` virtual ports, observe how data flows, and exercise `stty` and `comcontrol` to see how termios and modem signals work.

**Requirements.** A FreeBSD system. No special hardware.

**Steps.**

1. Load the `nmdm` module:

   ```
   # kldload nmdm
   ```

2. In terminal A, open the `A` side:

   ```
   # cu -l /dev/nmdm0A -s 9600
   ```

3. In terminal B, open the `B` side:

   ```
   # cu -l /dev/nmdm0B -s 9600
   ```

4. Type in terminal A. Observe that the characters appear in terminal B. Type in terminal B; they appear in terminal A.

5. Exit `cu` in both terminals (type `~.`). In a third terminal, run:

   ```
   # stty -a -f /dev/nmdm0A
   ```

   Read through the output. Notice `9600` for the 波特率, `cs8 -parenb -cstopb` for the byte format, and various flags for line discipline.

6. Change the 波特率 on one side:

   ```
   # stty 115200 -f /dev/nmdm0A
   ```

   Then open the ports again with `cu -s 115200`. The 波特率 change is visible, even though `nmdm(4)` does not actually wait for serialised bits.

7. Run:

   ```
   # comcontrol /dev/ttyu0A
   ```

   ...or rather, the equivalent for the `nmdm` 字符设备. The `nmdm` pairs do not always have `comcontrol`-visible modem signals, depending on the FreeBSD version; if your version does not, skip this step.

**Expected output.** Text appears on the opposite side. `stty` shows termios flags. You now have a reproducible way to test TTY-layer behaviour on your machine.

**What to watch for.** The pair identifiers (`0`, `1`, `2`...) are implicit and allocated on first open. If you cannot open `/dev/nmdm5A` because nothing has opened `/dev/nmdm4A` yet, this is expected: pairs are created lazily in increasing order. Also note that `cu` uses a lock file in `/var/spool/lock/`; if you kill `cu` abruptly, the lock file may persist and prevent reopens. Delete it manually if you get a "port in use" error.

### Lab 5: Talking to a Real USB-to-Serial Adapter

This lab brings real hardware into the loop. You will use a USB-to-serial adapter (an FT232, a CP2102, a CH340G, or anything else FreeBSD supports) and a terminal program to exercise the full path from `ucom(4)` through the TTY layer to userland.

**Goal.** Plug in a USB-to-serial adapter, verify it 附加, and use `cu` to send data to it (perhaps by looping the TX and RX pins together with a jumper).

**Requirements.** A USB-to-serial adapter. A jumper wire (if you want to do a hardware loopback) or a second serial 设备 to talk to (a development board, an embedded computer, or an old serial modem).

**Steps.**

1. Plug in the adapter. Run `dmesg | tail` and confirm it 附加. You should see lines like:

   ```
   uftdi0 on uhub0
   uftdi0: <FT232R USB UART, class 0/0, ...> on usbus0
   ```

   and a `ucomN: <...>` line just after that.

2. Run `ls -l /dev/cuaU*`. The adapter's port is usually `/dev/cuaU0` for the first adapter, `/dev/cuaU1` for the second, and so on. (Note the capital-U suffix, which distinguishes USB-provided ports from the real UART ports at `/dev/cuau0`.)

3. Put a jumper wire between the TX and RX pins of the adapter. This creates a hardware loopback: whatever the adapter transmits comes back on its own RX line.

4. In one terminal, set the 波特率:

   ```
   # stty 9600 -f /dev/cuaU0
   ```

5. Open the port with `cu`:

   ```
   # cu -l /dev/cuaU0 -s 9600
   ```

   Type characters. Every character you type should appear twice: once as local echo (if your terminal is echoing), and once as the character coming back through the loopback. Disable the local echo in `cu` if it is confusing; the `stty -echo` will help.

6. Remove the jumper. Type characters. Now they will not come back, because there is nothing connected to RX.

7. Exit `cu` with `~.`. Unplug the adapter. Run `dmesg | tail` and verify clean 分离.

**Expected output.** Characters are echoed back when the jumper is in place and lost when it is not. The `dmesg` shows clean 附加 and 分离.

**What to watch for.** If the adapter 附加 but no `cuaU` 设备 appears, the underlying `ucom(4)` instance may have 附加ed but failed to create its TTY. Check `dmesg` for errors. If characters come out garbled, the 波特率 is probably wrong: make sure every stage of the path (your terminal, `cu`, `stty`, the adapter, and the far end) is set to the same rate. On older hardware, some USB-to-serial adapters do not reset their internal configuration when you open them; you may need to explicitly set the 波特率 with `stty` before `cu` will work correctly.

### Lab 6: Observing Hot-Plug Lifecycle

This lab does not require writing any new 驱动程序 code. It exercises the 热插拔 lifecycle we described conceptually in Section 1 and in code in Section 2, using the existing `uhid` or `ukbd` 驱动程序 as the test subject.

**Goal.** Plug in and unplug a USB 设备 repeatedly while monitoring 内核 logs, observing the full 附加/分离 sequence.

**Requirements.** A USB 设备 you can plug and unplug without disrupting your work session. A USB flash drive or a USB mouse are both safe; a USB keyboard is not (because 分离ing a keyboard in the middle of a session can strand your shell).

**Steps.**

1. Open a terminal window and run:

   ```
   # tail -f /var/log/messages
   ```

   or, if your system does not log 内核 messages to that file:

   ```
   # dmesg -w
   ```

   (The `-w` flag is a FreeBSD 14 addition that streams new 内核 messages as they arrive.)

2. Plug in your USB 设备. Observe the messages. You should see:
   - A message from the USB controller about the new 设备 appearing.
   - A message from `uhub` about the port powering up.
   - A message from the class 驱动程序 that matched the 设备 (e.g., `ums0` for a mouse, `umass0` for a flash drive).
   - Possibly a message from the higher-level subsystem (e.g., `da0` for a mass-storage 设备).

3. Unplug the 设备. Observe the messages. You should see:
   - A message from `uhub` about the port powering down.
   - A 分离 message from the class 驱动程序.

4. Repeat several times. Watch that every 附加 is matched by a 分离. Watch that no message is missed. Watch the timing; the 附加 sequence can take tens or hundreds of milliseconds because enumeration involves several 控制传输.

5. Write a tiny shell loop that records the 附加 and 分离 times:

   ```
   # dmesg -w | awk '/ums|umass/ { print systime(), $0 }'
   ```

   (Adjust the regex for the 设备 type you are using.) This gives you a machine-readable log of 附加 and 分离 timestamps.

**Expected output.** Clean 附加 and 分离 every time, with no dangling state.

**What to watch for.** Occasionally you will see a 设备 附加 and then immediately 分离 within a few hundred milliseconds. This usually indicates the 设备 is failing enumeration: either a bad cable, insufficient power, or a buggy 设备 firmware. If it happens consistently with one 设备, try a different USB port or a powered hub. Also watch for cases where the 内核 reports a stall during enumeration; these are rarely harmful but indicate that the enumeration needed multiple tries.

### Lab 7: Building a ucom(4) Skeleton from Scratch

This lab is an extended one that combines the USB and serial material from the chapter. You will build a minimal `ucom(4)` 驱动程序 skeleton that presents itself as a 串行端口 but is backed by a simple USB 设备.

**Goal.** Build a `ucom(4)` 驱动程序 skeleton that 附加 to a specific USB 设备, 寄存器 with the `ucom(4)` 框架, and provides empty implementations of the key 回调. The 驱动程序 will not actually talk to the hardware, but it will exercise the full `ucom(4)` registration path.

**Requirements.** The materials from Lab 2 (the USB驱动程序 skeleton). A USB 设备 you can match against (for testing, you can use the same VOTI/OBDEV VID/PID as in Lab 2, or any spare USB 设备 whose IDs you can read).

**Steps.**

1. Start from Lab 2 as a template. Copy the directory to `lab07-ucom-skeleton`.

2. Modify the softc to include a `struct ucom_super_softc` and a `struct ucom_softc`:

   ```c
   struct lab07_softc {
       struct ucom_super_softc sc_super_ucom;
       struct ucom_softc        sc_ucom;
       struct usb_device       *sc_udev;
       struct mtx               sc_mtx;
       struct usb_xfer         *sc_xfer[LAB07_N_XFER];
       uint8_t                  sc_iface_index;
       uint8_t                  sc_flags;
   };
   ```

3. Add a `struct ucom_回调` with stub implementations:

   ```c
   static void lab07_cfg_open(struct ucom_softc *);
   static void lab07_cfg_close(struct ucom_softc *);
   static int  lab07_pre_param(struct ucom_softc *, struct termios *);
   static void lab07_cfg_param(struct ucom_softc *, struct termios *);
   static void lab07_cfg_set_dtr(struct ucom_softc *, uint8_t);
   static void lab07_cfg_set_rts(struct ucom_softc *, uint8_t);
   static void lab07_cfg_set_break(struct ucom_softc *, uint8_t);
   static void lab07_start_read(struct ucom_softc *);
   static void lab07_stop_read(struct ucom_softc *);
   static void lab07_start_write(struct ucom_softc *);
   static void lab07_stop_write(struct ucom_softc *);
   static void lab07_free(struct ucom_softc *);

   static const struct ucom_callback lab07_callback = {
       .ucom_cfg_open       = &lab07_cfg_open,
       .ucom_cfg_close      = &lab07_cfg_close,
       .ucom_pre_param      = &lab07_pre_param,
       .ucom_cfg_param      = &lab07_cfg_param,
       .ucom_cfg_set_dtr    = &lab07_cfg_set_dtr,
       .ucom_cfg_set_rts    = &lab07_cfg_set_rts,
       .ucom_cfg_set_break  = &lab07_cfg_set_break,
       .ucom_start_read     = &lab07_start_read,
       .ucom_stop_read      = &lab07_stop_read,
       .ucom_start_write    = &lab07_start_write,
       .ucom_stop_write     = &lab07_stop_write,
       .ucom_free           = &lab07_free,
   };
   ```

   `ucom_pre_param` runs on the caller's context before the configuration task is scheduled; use it to reject unsupported termios values by returning a nonzero errno. `ucom_cfg_param` runs in the 框架's task context and is where you would issue the actual USB 控制传输 to reprogram the chip.

4. Implement each 回调 as a no-op for now. Add `设备_printf(sc->sc_super_ucom.sc_dev, "%s\n", __func__)` to each so that you can see which 回调 are being invoked.

5. In the 附加 method, after `usbd_transfer_setup`, call:

   ```c
   error = ucom_attach(&sc->sc_super_ucom, &sc->sc_ucom, 1, sc,
       &lab07_callback, &sc->sc_mtx);
   if (error != 0) {
       goto fail_xfer;
   }
   ```

6. In the 分离 method, call `ucom_分离(&sc->sc_super_ucom, &sc->sc_ucom)` before `usbd_transfer_unsetup`.

7. Add `MODULE_DEPEND(lab07, ucom, 1, 1, 1);` after the existing MODULE_DEPEND.

8. Build, load, plug in the 设备, and observe. In `dmesg`, you should see the 驱动程序 附加, and you should see a `cuaU0` 设备 appear in `/dev/`.

9. Run `cu -l /dev/cuaU0 -s 9600`. The `cu` command will open the 设备, which triggers several of the ucom 回调. Watch `dmesg` to see which ones fire. Close `cu` with `~.` and observe more 回调.

10. Run `stty -a -f /dev/cuaU0`. Observe that the port has default termios settings. Run `stty 115200 -f /dev/cuaU0` and observe that `lab07_cfg_param` is called.

11. Unplug the 设备. Observe clean 分离.

**Expected output.** The 驱动程序 附加 as a `ucom` 设备, creates `/dev/cuaU0`, and responds to configuration ioctls (even though the underlying USB 设备 does not actually do anything). Every 回调 invocation is visible in `dmesg`.

**What to watch for.** If the 驱动程序 附加 but `/dev/cuaU0` does not appear, check that `ucom_附加` succeeded. The return value is an errno; a nonzero value means failure. If it failed with `ENOMEM`, you are running out of memory for the TTY allocation. If it failed with `EINVAL`, one of the 回调 fields is probably null (look at `/usr/src/sys/dev/usb/serial/usb_serial.c` to see which fields are strictly required).

This lab is a building 块. A real `ucom` 驱动程序 (like `uftdi`) would fill in the 回调 with actual USB transfers to the chip. Starting from an empty skeleton and adding one 回调 at a time is a good way to build a new 驱动程序.

### Lab 8: Troubleshooting a Hung TTY Session

This lab is a diagnostic exercise. Given a malfunctioning serial setup, you will use the tools from Section 5 to find the problem.

**Goal.** Find why a `cu` session does not echo characters back after connecting to an `nmdm(4)` pair that has an unconfigured 波特率 on one side.

**Steps.**

1. Load `nmdm`:

   ```
   # kldload nmdm
   ```

2. Set different 波特率s on the two sides. This is contrived but mimics a real configuration bug:

   ```
   # stty 9600 -f /dev/nmdm0A
   # stty 115200 -f /dev/nmdm0B
   ```

3. Open both sides with `cu`, each with the mismatched rate:

   ```
   (terminal 1) # cu -l /dev/nmdm0A -s 9600
   (terminal 2) # cu -l /dev/nmdm0B -s 115200
   ```

4. Type in terminal 1. You will likely see characters appear in terminal 2, but possibly garbled. Or characters may not appear at all if the `nmdm(4)` 驱动程序 enforces rate matching strictly.

5. Exit both `cu` sessions.

6. Run `stty -a -f /dev/nmdm0A` and `stty -a -f /dev/nmdm0B`. Find the discrepancy.

7. Fix: set both sides to the same rate. Reopen `cu` and verify that the issue is resolved.

**What to watch for.** This lab teaches the diagnostic habit of checking both ends of a link. A mismatch at any one end produces problems; finding it requires looking at both. The diagnostic tools (`stty`, `comcontrol`) work from the command line and produce human-readable output. Making use of them is a simple first check before diving into deeper debugging.

### Lab 9: Monitoring USB Transfer Statistics

This lab explores the per-channel statistics the USB 框架 maintains, which can help identify performance issues or hidden errors.

**Goal.** Use `usbconfig dump_stats` to observe the transfer counts on a 总线y USB 设备 and identify whether the 设备 is performing as expected.

**Steps.**

1. Plug in a USB 设备 that you can exercise meaningfully. A USB flash drive is a good choice because you can trigger 批量传输 by copying files.

2. Identify the 设备:

   ```
   # usbconfig list
   ```

   Note the `ugenN.M` identifier.

3. Dump the baseline statistics:

   ```
   # usbconfig -d ugenN.M dump_stats
   ```

   Record the output.

4. Perform significant I/O to the 设备. For a flash drive, copy a large file:

   ```
   # cp /usr/src/sys/dev/usb/usb_transfer.c /mnt/usb_mount/
   ```

5. Dump the statistics again. Compare.

6. Note which counters changed. `xfer_completed` should have increased significantly. `xfer_err` should still be small.

7. Try to deliberately cause errors. Unplug the 设备 mid-transfer. Then plug it back in. Dump the stats on the new `ugenN.M` (a new one is allocated on replug).

**What to watch for.** The statistics reveal invisible behaviours. A 设备 that is mostly working but occasionally stalling will show `stall_count` nonzero. A 设备 that is dropping transfers will show `xfer_err` climbing. In normal operation, a healthy 设备 shows steady `xfer_completed` growth and zero errors.

If you are developing a 驱动程序 and the statistics show unexpected errors, that is a clue that something is wrong. The statistics are maintained by the USB 框架, not the 驱动程序, so they reflect reality regardless of whether the 驱动程序 notices.

## 挑战练习

Challenge exercises stretch your understanding. They are not strictly necessary for progressing to 第27章, but each one will deepen your grasp of USB and 串行驱动程序 work. Take your time. Read relevant FreeBSD source. Write small programs. Expect some challenges to take several hours.

### Challenge 1: Add a Third USB Endpoint Type

The skeleton in Section 2 supports 批量传输. Extend it to also handle an interrupt 端点. Add a new channel to the `struct usb_config` array with `.type = UE_INTERRUPT`, `.direction = UE_DIR_IN`, and a small 缓冲区 (say, sixteen bytes). Implement the 回调 as a continuous poll, reading a small status 数据包 from the 设备 on every interrupt-IN completion.

Test the change by comparing the behaviour of the three channels. Bulk channels should be quiet most of the time and only submit transfers when the 驱动程序 has work to do. The interrupt channel should run continuously, quietly consuming interrupt-IN 数据包 whenever the 设备 sends them.

A stretch goal: make the interrupt 回调 deliver received bytes to the same `/dev` node as the bulk channel. When userspace reads the node, it gets a merged view of bulk-in and interrupt-in data. This is a useful pattern for 设备 that have both streaming data and asynchronous status events.

### Challenge 2: Write a Minimal USB Gadget Driver

The running example is a host-side 驱动程序: the FreeBSD machine is the USB host, and the 设备 is the peripheral. Turn the example around by writing a USB gadget 驱动程序 that makes the FreeBSD machine present itself as a simple 设备 to another host.

This requires USB-on-the-Go hardware, so the challenge is only feasible on specific boards (some ARM development boards support it). The relevant source is in `/usr/src/sys/dev/usb/template/`. Start from `usb_template_cdce.c`, which implements the CDC 以太网 class, and modify it to implement a simpler vendor-specific class with one bulk-OUT 端点 that just swallows whatever the host sends.

This challenge teaches you how the USB 框架 looks from the other side. Many of the concepts are mirror-imaged: what was a transfer from the host's perspective is a transfer from the 设备's perspective, but the direction of the bulk arrow is reversed.

### Challenge 3: A Custom `termios` Flag Handler

The `termios` structure has many flags, and the `uart(4)` 框架 handles most of them automatically. Write a small modification to a UART 驱动程序 (or to a copy of `uart_dev_ns8250.c` in a private build) that makes the 驱动程序 log a `设备_printf` message every time a specific termios flag changes value.

Pick, say, `CRTSCTS` (hardware 流控制) as the flag to track. Add a log message in the 驱动程序's `param` path that prints "CRTSCTS=on" or "CRTSCTS=off" whenever the flag's new value differs from its old value.

Test the modification by running:

```console
# stty crtscts -f /dev/cuau0
# stty -crtscts -f /dev/cuau0
```

Verify that the log messages appear in `dmesg` and that they correspond correctly to the `stty` changes.

This challenge is about understanding exactly where in the call chain the termios change arrives at the 驱动程序. The answer (in `param`) is documented in the source, but seeing it with your own eyes is different from reading about it.

### Challenge 4: Parsing a Small USB Protocol

Pick a USB protocol you are curious about. HID is a good candidate because it is widely documented. CDC ACM is another good choice because it is simple. Pick one, read the specification on usb.org (the public parts), and write a small protocol parser in C that takes a 缓冲区 of bytes and prints what they mean.

For HID, the parser would consume reports: input reports, output reports, feature reports. It would look up the 设备's report 描述符 to learn the layout. It would print, for each report, the usage (mouse motion, button press, keyboard scan code) and the value.

For CDC ACM, the parser would consume the AT command set: a small set of commands that terminal programs use to configure modems. It would recognise the commands and report which ones the 驱动程序 would handle and which would be passed through to the 设备.

This is not a 驱动程序-writing challenge per se; it is a protocol-understanding challenge. Device 驱动程序 implement protocols, and being comfortable with protocol specifications is a core skill.

### Challenge 5: Ro总线tness Under Load

Take the echo-loop 驱动程序 from Lab 3 (or a similar 驱动程序 you have written) and stress-test it. Write a userland program that runs two threads: one constantly writes random bytes to the 设备, one constantly reads and verifies.

Run the program for an hour. Then run it for a day. Then unplug and replug the 设备 during the run and see whether the program recovers cleanly.

You will probably find bugs. Common ones include: write 回调 locking issues under concurrent access, races between close() and in-flight transfers, memory leaks from 缓冲区 that are allocated but never freed on specific error paths, and state machine bugs when a stall arrives at an unexpected moment.

Each bug you find will teach you something about where the 驱动程序's contract with its callers is subtle. Fix the bugs. Log what you learned. This is exactly the kind of work that separates a good 驱动程序 from a merely working one.

### Challenge 6: Implement Suspend/Resume Properly

Most USB驱动程序 do not implement suspend and resume handlers. The 框架 has defaults that work for the common case, but a 驱动程序 that holds long-term state (a queue of pending commands, a streaming context, a negotiated session) may need to save and restore that state around suspend cycles.

Extend the echo-loop 驱动程序 with `设备_suspend` and `设备_resume` methods. In suspend, flush any pending transfers and save a small 挂载 of state. In resume, restore the state and resubmit any pending work.

Test by running the system through a suspend cycle (on a laptop that supports it) while the 驱动程序 is running. Verify that after resume, the 驱动程序 continues working correctly and no state was lost.

This challenge teaches the subtleties of suspend/resume, including that hardware may be in a different state after resume than it was before suspend, and that all in-flight state must be reconstructed or abandoned.

### Challenge 7: Adding `poll(2)` Support

Most 驱动程序 shown in this chapter support `read(2)` and `write(2)` but not `poll(2)` or `select(2)`. These system calls let userland programs wait for I/O readiness on multiple 描述符 at once, which is essential for servers and interactive programs.

Add a `d_poll` method to the echo 驱动程序's `cdevsw`. The method should return a bitmask indicating which I/O events are currently possible: POLLIN if there is data to read, POLLOUT if there is space to write.

The hardest part of adding poll support is the wakeup logic. When a 传输回调 adds data to the RX queue, it must call `selwakeup` on the selinfo structure the poll mechanism uses. Similarly, when the write 回调 drains bytes from the TX queue and makes space, it must call `selwakeup` on the write selinfo.

This challenge will require reading `/usr/src/sys/kern/sys_generic.c` and `/usr/src/sys/sys/selinfo.h` to understand the selinfo mechanism.

### Challenge 8: Writing a Character-Counter ioctl

Add an ioctl to the echo 驱动程序 that returns the current TX and RX byte counters. The ioctl 接口 requires you to:

1. Define a magic number and struct for the ioctl in a header:
   ```c
   struct myfirst_usb_stats {
       uint64_t tx_bytes;
       uint64_t rx_bytes;
   };
   #define MYFIRST_USB_GET_STATS _IOR('U', 1, struct myfirst_usb_stats)
   ```

2. Implement a `d_ioctl` method that responds to `MYFIRST_USB_GET_STATS` by copying the counters out to userland.

3. Maintain the counters in the softc, incrementing them in the 传输回调.

4. Write a userland program that issues the ioctl and prints the results.

This challenge teaches the ioctl 接口, which is the standard way 驱动程序 expose non-streaming operations to userland. It also introduces you to the `_IOR`, `_IOW`, and `_IOWR` macros from `<sys/ioccom.h>`.

## 故障排除指南

Despite your best efforts, problems will occur. This section documents the most common classes of problem you will hit while working on USB and 串行驱动程序, with concrete steps to diagnose each.

### The Module Will Not Load

Symptom: `kldload myfirst_usb.ko` returns an error, typically with a message about unresolved symbols.

Causes and fixes:
- Missing `MODULE_DEPEND` entry. Add `MODULE_DEPEND(myfirst_usb, usb, 1, 1, 1);` to the 驱动程序.
- Missing `MODULE_DEPEND` on a second module, such as `ucom`. If your 驱动程序 uses `ucom_附加`, add a dependency on `ucom`.
- Compiled against a 内核 that does not match the running 内核. Rebuild the module against the currently-running sources.
- Kernel symbol table out of date. After 内核 upgrade, run `kldxref /boot/内核` to refresh.

If the error message mentions a specific symbol you did not write (like `ttycreate` or `cdevsw_open`), look up the missing symbol in the source tree to find out which subsystem it lives in, and add a `MODULE_DEPEND` on that module.

### The Driver Loads but Never Attaches

Symptom: `kldstat` shows the 驱动程序 loaded, but `dmesg` shows no 附加 message when the 设备 is plugged in.

Causes and fixes:
- Match table does not match the 设备. Compare the vendor and 产品IDs from `usbconfig list` against your `STRUCT_USB_HOST_ID` entries.
- Interface number mismatch. If the 设备 has multiple 接口 and your 探测 guards against `bIfaceIndex != 0`, try a different 接口.
- Probe returns `ENXIO` for some other reason. Add `设备_printf(dev, "探测 with class=%x subclass=%x\n", uaa->info.bInterfaceClass, uaa->info.bInterfaceSubClass);` at the top of `探测` temporarily to see what the 框架 is offering.
- Another 驱动程序 is claiming the 设备 first. Check `dmesg` for other 驱动程序 附加ments; you may need to explicitly unload the competing 驱动程序 with `kldunload` before yours can bind. Alternatively, give your 驱动程序 a higher priority through 总线-探测 return values (applies to PCI-like 总线es, not USB).

### The Driver Attaches but `/dev` Node Does Not Appear

Symptom: 附加 message in `dmesg`, but `ls /dev/` shows no corresponding entry.

Causes and fixes:
- `make_dev` call failed. Check the return value; if null, handle the error and log it.
- Wrong cdevsw. Make sure `myfirst_usb_cdevsw` is declared correctly with `d_version = D_VERSION` and valid `d_name`, `d_open`, `d_close`, `d_read`, `d_write`, `d_ioctl` where relevant.
- `si_drv1` not set. Although not strictly required for the node to appear, many bugs manifest as "the node appears but ioctls see a NULL softc" because `si_drv1` was not initialised.
- Permissions issue. The default 0644 permissions may restrict access; try 0666 temporarily during development.

### The Driver Panics on Detach

Symptom: unplugging the 设备 (or unloading the module) causes a 内核 panic.

Causes and fixes:
- Transfer 回调 running during 分离. You must call `usbd_transfer_unsetup` before destroying the 互斥锁. The 框架's cancellation and wait logic is what makes 分离 safe.
- `/dev` node open when 驱动程序 unloads. If userspace has the node open, the module cannot unload. Run `fstat | grep myfirst_usb` to see which process holds it, and kill the process or close the file.
- Memory freed before all uses complete. If you use deferred work (taskqueue, callout), you must cancel and wait for it before freeing the softc. The `taskqueue_drain` and `callout_drain` functions exist for this.
- Softc use-after-free. If you have code outside the 驱动程序 that holds a pointer to the softc, the softc can be freed while that pointer is still dangling. Redesign to avoid external softc pointers, or add reference counting.

### Transfers Stall

Symptom: 批量传输 appear to succeed at the submit call but never complete, or they complete with `USB_ERR_STALLED`.

Causes and fixes:
- Wrong 端点 direction. Verify the direction in your `struct usb_config` against the 端点's `bEndpointAddress` high bit.
- Wrong 端点 type. Verify that the `type` field matches the 端点's `bmAttributes` low bits.
- Transfer too large. If you set a 帧 length larger than the 端点's `wMaxPacketSize`, the 框架 will usually slice it into 数据包, but some 设备 reject a transfer that exceeds an internal 缓冲区.
- Device firmware stall. The remote 设备 is signalling "not ready." The 框架's automatic clear-stall should recover, but a persistent stall usually indicates a protocol error (wrong command, wrong sequence, missing authentication).

### Serial Characters Garbled

Symptom: bytes appear on the wire but are wrong or contain extra characters.

Causes and fixes:
- Baud rate mismatch. Every stage must agree. Use `stty` to check all stages.
- Byte format mismatch. Set databits, 奇偶校验, and stopbits to match. `stty cs8 -parenb -cstopb` is the most common configuration.
- Incorrect `termios` flag handling in the 驱动程序. If you modify `uart_dev_ns8250.c` and break `param`, the chip will be programmed wrong. Compare against the upstream file.
- Flow-control mismatch. If one side has `CRTSCTS` enabled and the other does not, bytes will be lost under load. Set both sides consistently.
- Cable issue. A bad cable or a cable with unusual pinouts (some RJ45-to-DB9 cables have nonstandard pinouts) can introduce bit errors. Swap cables to rule this out.

### A Process is Stuck in `read(2)` and Will Not Exit

Symptom: a program 块ed on the 驱动程序's `read()` path will not respond to Ctrl+C or `kill`.

Causes and fixes:
- Driver `d_read` sleeps without checking for signals. Use `msleep(..., PCATCH, ...)` (with the `PCATCH` flag) so the sleep returns `EINTR` when a signal arrives, and propagate the errno back to userspace.
- Driver `d_read` holds a non-interruptible lock. Verify that the sleep is on an interruptible condition variable and that the 互斥锁 is dropped during the sleep.
- Transfer 回调 is never arming the channel. If your `d_read` waits on a flag that only the read 回调 sets, and the read 回调 is never fired, the wait will never complete. Make sure the channel is started on `d_open` or at 附加 time.

### High CPU Usage When Idle

Symptom: the 驱动程序 consumes significant CPU even when no data is flowing.

Causes and fixes:
- Polling-based implementation. If your 驱动程序 polls a flag in a 总线y loop, rewrite it to sleep on an event.
- Callback firing excessively. The 框架 should not fire a 回调 without a state change, but some misconfigured channels can enter a "retry on error" loop that fires the 回调 as fast as the hardware can respond. Add a retry counter or a rate-limiter.
- Read 回调 with no work but always rearming. If the 设备 sends zero-byte transfers to signal "I have nothing to say," make sure your 回调 handles these gracefully without treating them as normal data.

### `usbconfig` Shows the Device but `dmesg` Is Silent

Symptom: `usbconfig list` shows the 设备, but no 驱动程序 附加 message appears.

Causes and fixes:
- Device 附加ed to `ugen` (the generic 驱动程序) because no specific 驱动程序 matched. This is the normal behaviour when there is no matching 驱动程序. Check the 匹配表s of the available 驱动程序. `pciconf -lv` will not help here because this is USB, not PCI; the USB equivalent is `usbconfig -d ugenN.M dump_设备_desc`.
- `devd` is disabled and auto-load is not happening. Enable `devd` by running `service devd onestart`, then plug the 设备 in again.
- Module file is not in a loadable path. `kldload` can take a full path (`kldload /path/to/module.ko`), but for automatic loading by `devd`, the module has to be in a directory `devd` is configured to search. `/boot/modules/` is the conventional location for out-of-tree modules on a production system.

### Debugging a Deadlock with `WITNESS`

Symptom: the 内核 hangs with the CPU stuck in a specific function, and `WITNESS` is enabled.

Causes and fixes:
- Lock order violation. `WITNESS` will log the violation on the serial console. Read the log: it will tell you which locks were taken in which order, and where the reverse order was observed. Fix by establishing a consistent lock acquisition order throughout your 驱动程序.
- Lock held across a sleep. If you hold a 互斥锁 and then call a function that sleeps, you can deadlock with any other thread that wants the 互斥锁. Identify the sleeping function (often hidden in an allocation or in a USB transfer wait), and restructure to release the 互斥锁 before the sleep.
- Lock taken in interrupt context that was first taken outside interrupt context without `MTX_SPIN`. FreeBSD 互斥锁es have two forms: default (`MTX_DEF`) can sleep, spin (`MTX_SPIN`) cannot. Taking a sleep 互斥锁 from an 中断处理程序 is a bug.

Enabling `WITNESS` during development (by building the 内核 with `options WITNESS` or by using `GENERIC-NODEBUG`'s `INVARIANTS`-enabled counterpart) catches many of these problems before they appear on a user's machine.

### A Driver That Appears Twice for the Same Device

Symptom: `dmesg` shows your 驱动程序 附加ing twice for a single 设备, creating `myfirst_usb0` and `myfirst_usb1` with the same USB IDs.

Causes and fixes:
- The 设备 has two 接口 and the 驱动程序 is matching both. Check `bIfaceIndex` in the 探测 method and match only the 接口 you actually support.
- The 设备 has multiple configurations and both are active. This is rare; if so, select the correct configuration explicitly in the 附加 method.
- Another 驱动程序 is 附加ed to one of the 接口. This is not a bug; it just means the 设备 is multi-接口 and different 驱动程序 claim different 接口. If you see `myfirst_usb0` and `ukbd0` for the same 设备, the 设备 has both a vendor-specific 接口 and a HID 接口, and the two 驱动程序 附加 independently.

### USB Serial Baud Rate Does Not Take Effect

Symptom: You `stty 115200 -f /dev/cuaU0`, but data exchange happens at a different rate.

Causes and fixes:
- The 控制传输 to program the 波特率 failed. Check `dmesg` for error messages from `ucom_cfg_param`. Instrument the 驱动程序 to log the result of the 控制传输.
- The chip's divisor encoding is wrong. Different FTDI variants use slightly different divisor formulas; check the variant detection in the 驱动程序.
- The peer is running at a different rate. As noted earlier in the chapter, both ends must agree.
- The cable or adapter is introducing its own rate limitation. Some USB-to-serial adapters silently renegotiate; this is rare but can happen with poor-quality cables.

### A Kernel Panic with "Spin lock held too long"

Symptom: The 内核 panics with this message, usually during high I/O on the 驱动程序.

Causes and fixes:
- A UART 驱动程序's `uart_ops` method is sleeping or 块ing. The six methods in `uart_ops` run with spin locks held (on some paths) and must not sleep, call non-spin-safe functions, or do long loops. Review the offending method for any expensive calls.
- The 中断处理程序 is not draining the interrupt source fast enough. If the handler takes longer than the interrupt rate, interrupts accumulate. Speed up the handler.
- Lock contention is causing priority inversion. Reduce the scope of the critical section, or break it up.

### A Device Never Completes Enumeration

Symptom: Plugging in a 设备 produces a `dmesg` line or two about enumeration starting, but never a completion message.

Causes and fixes:
- The 设备 is violating the USB specification. Some cheap or counterfeit 设备 have buggy firmware. If possible, try a different 设备.
- Insufficient power. Devices that claim more power than the port can supply will fail to enumerate. Try a powered hub.
- Electromagnetic interference. A bad cable or a bad port can cause bit errors during enumeration. Try different cables or ports.
- The USB 主机控制器 is in a confused state. Try unloading and reloading the 主机控制器 驱动程序, or (as a last resort) rebooting.

### Diagnostic Checklist When You Are Stuck

When a 驱动程序 under development is not behaving correctly and you do not know why, walk through this checklist in order. Each step eliminates a large class of possible problems.

1. Compile cleanly with `-Wall -Werror`. Many subtle bugs produce warnings.
2. Load in a 内核 built with `INVARIANTS` and `WITNESS`. Any locking or invariant violations will be caught immediately.
3. Enable your 驱动程序's debug logging. Run a minimal reproduction scenario and capture the logs.
4. Compare the 驱动程序's behaviour against a known-working 驱动程序 for similar hardware. Diffing behaviour reveals bugs that staring at your own code does not.
5. Simplify the scenario. Write a minimal userland test program. Use a minimal USB 设备 (or an `nmdm` pair for serial). Remove every variable you can.
6. Use `dtrace` on the USB 框架 functions. `usbd_transfer_submit:entry` and `usbd_transfer_submit:return` 探测 let you trace exactly which transfers were submitted and what happened to them.
7. Run the 驱动程序 with `WITNESS_CHECKORDER` enabled. Each time a 互斥锁 is taken, the order is verified against the accumulated history.
8. If the issue is intermittent, run under a stress-test harness that generates load for hours. Intermittent bugs become reproducible under sustained load.

This checklist is not exhaustive, but it covers the techniques that find the majority of 驱动程序 bugs.

## Reading the FreeBSD USB Source Tree: A Guided Tour

The `myfirst_usb` skeleton and the FTDI walkthrough have given you the shape of a USB驱动程序. But the real learning happens when you read existing 驱动程序 in the tree. Each one is a small lesson in how to apply the 框架 to a specific class of 设备. This section gives you a guided tour of five 驱动程序, ordered from simplest to most representative, and points out what each one teaches.

The pattern we recommend is this. Open each 驱动程序's source file next to this section. Read the opening comment 块 and the structure definitions first; those tell you what the 驱动程序 is for and what state it maintains. Then trace the lifecycle: 匹配表, 探测, 附加, 分离, registration. Only after the lifecycle is clear should you move on to the data path. This ordering mirrors how the 框架 itself treats the 驱动程序: first as a match candidate, then as an 附加ed 驱动程序, and only then as something that moves data.

### Tour 1: uled.c, the Simplest USB Driver

File: `/usr/src/sys/dev/usb/misc/uled.c`.

Start here. `uled.c` is the Dream Cheeky USB LED 驱动程序. It is under 400 lines. It implements a single output (setting the LED colour) through a single 控制传输. There is no input, no 批量传输, no 中断传输, no concurrent I/O. Everything about it is minimal, and for that reason everything about it is easy to read.

Key things to study in `uled.c`:

The 匹配表 has a single entry: `{USB_VPI(USB_VENDOR_DREAM_CHEEKY, USB_PRODUCT_DREAM_CHEEKY_WEBMAIL_NOTIFIER_2, 0)}`. This is the minimal match-by-VID/PID idiom. No subclass or protocol filtering; just vendor and product.

The softc is tiny. It contains a 互斥锁, the `usb_设备` pointer, the `usb_xfer` array, and the LED state. This is the minimum every USB驱动程序 needs.

The 探测 method is two lines: check that the 设备 is in host mode and return the result of `usbd_lookup_id_by_uaa` against the 匹配表. No 接口-index check, no complex matching. For a simple 设备 with a single function, this is enough.

The 附加 method allocates the transfer channel, creates a 设备-file entry with `make_dev`, and stores the pointers. No complex negotiation; the 设备 is ready after `附加` returns.

The I/O path is a single 控制传输 with a fixed setup. The 驱动程序 sets the 帧 length, fills in the color bytes with `usbd_copy_in`, and calls `usbd_transfer_submit`. That is it.

Read `uled.c` first. When you have read it once, the rest of the USB subsystem opens up. Every more complex 驱动程序 is a variation on this pattern.

### Tour 2: ugold.c, Adding Interrupt Transfers

File: `/usr/src/sys/dev/usb/misc/ugold.c`.

`ugold.c` drives a USB thermometer. It is still very short, under 500 lines, but it introduces 中断传输, which are the staple of HID-class 设备.

Key things to learn from `ugold.c`:

The 设备 publishes temperature readings periodically via an interrupt 端点. The 驱动程序's job is to listen on that 端点 and deliver the readings to userland via `sysctl`.

The `usb_config` array now has an entry for `UE_INTERRUPT`, with `UE_DIR_IN`. This tells the 框架 to set up a channel that polls the interrupt 端点.

The interrupt 回调 shows the canonical pattern: on `USB_ST_TRANSFERRED`, extract the received bytes with `usbd_copy_out`, parse them, update the softc. On `USB_ST_SETUP` (including the initial 回调 after `start`), set the 帧 length and submit. On `USB_ST_ERROR`, decide whether to recover or give up.

The 驱动程序 exposes readings through `sysctl` nodes created in `附加` and torn down in `分离`. This is a common pattern for 设备 that produce occasional readings: the interrupt 回调 writes to softc state, and userland reads from `sysctl` when it wants a value.

Compare `ugold.c` to `uled.c` after reading both. The control-transfer-only 驱动程序 and the interrupt-transfer 驱动程序 represent the two most common skeleton patterns. Most other USB驱动程序 are composed of variations of these two.

### Tour 3: udbp.c, Bidirectional Bulk Transfers

File: `/usr/src/sys/dev/usb/misc/udbp.c`.

`udbp.c` is the USB Double Bulk Pipe 驱动程序. It exists to test bidirectional bulk data flow between two computers connected by a special USB-to-USB cable. It is about 700 lines and gives you a complete working example of bulk read and bulk write.

Key things to learn from `udbp.c`:

The `usb_config` has two entries: one for `UE_BULK` `UE_DIR_OUT` (host-to-设备) and one for `UE_BULK` `UE_DIR_IN` (设备-to-host). This is the standard bulk-duplex pattern.

Each 回调 does the same three-state dance. On `USB_ST_SETUP`, set the 帧 length (or if it is a read, just submit). On `USB_ST_TRANSFERRED`, consume the completed data and re-arm. On `USB_ST_ERROR`, decide the recovery policy.

The 驱动程序 uses the netgraph 框架 to integrate with higher layers. This is a choice specific to the Double Bulk Pipe 设备. For a simple application, you would expose the bulk channels through a 字符设备, as `myfirst_usb` does.

Trace how the softc maintains the state of each direction independently. The receive 回调 rearms only when a 缓冲区 is available. The transmit 回调 rearms only when there is something to send. The two 回调 coordinate only through shared softc fields (counter of pending operations, queue pointers).

### Tour 4: uplcom.c, a USB-to-Serial Bridge

File: `/usr/src/sys/dev/usb/serial/uplcom.c`.

`uplcom.c` drives the Prolific PL2303, one of the most common USB-to-serial chips. At around 1400 lines, it is more substantial than the previous three, but every part of it maps directly onto the serial-驱动程序 pattern from Section 4 of this chapter.

Key things to learn from `uplcom.c`:

The `ucom_回调` structure fills in every configuration method you would expect a real 驱动程序 to implement: `ucom_cfg_open`, `ucom_cfg_param`, `ucom_cfg_set_dtr`, `ucom_cfg_set_rts`, `ucom_cfg_set_break`, `ucom_cfg_get_status`, `ucom_cfg_close`. Each of these calls the 框架-provided `ucom` primitives after issuing the chip-specific USB 控制传输.

Look at `uplcom_cfg_param`. It takes a `termios` structure, extracts the 波特率 and framing, and constructs a vendor-specific 控制传输 to program the chip. This is how a user's `stty 9600` call propagates through the layers: `stty` updates `termios`, the TTY layer calls `ucom_param`, the 框架 schedules the 控制传输, and `uplcom_cfg_param` programs the chip.

Compare `uplcom_cfg_param` with the corresponding function in `uftdi.c`. Both translate a `termios` to a vendor-specific control sequence, but the vendor protocols are entirely different. This illustrates why the 框架 insists on per-vendor 驱动程序: each chip has its own command set, and the 框架's job is only to give each 驱动程序 a uniform way to be called.

Note how the 驱动程序 handles reset, modem signals, and break. Each modem-line operation is a separate USB 控制传输. The cost of changing, say, DTR is one round-trip to the 设备, which on a 12 Mbps 总线 takes about 1 ms. This tells you why line signals change more slowly over USB-to-serial than over a native UART, and why protocols that toggle DTR frequently can behave differently through a USB-to-serial adapter.

### Tour 5: uhid.c, the Human Interface Device Driver

File: `/usr/src/sys/dev/usb/input/uhid.c`.

`uhid.c` is the generic HID 驱动程序. HID stands for Human Interface Device; it covers keyboards, mice, gamepads, touchscreens, and countless vendor-specific 设备 that conform to the HID class standard. `uhid.c` is roughly 1000 lines.

Key things to learn from `uhid.c`:

The 匹配表 uses class-based matching. Instead of listing every VID/PID, the 驱动程序 matches any 设备 that advertises the HID 接口 class. `UIFACE_CLASS(UICLASS_HID)` tells the 框架 to match any HID 接口, no matter which vendor made the 设备.

The 驱动程序 exposes the 设备 through a 字符设备, not through `ucom` or a networking 框架. The 字符设备 pattern lets userland programs open `/dev/uhidN` and issue `ioctl` calls to read HID 描述符, read reports, and set feature reports.

The interrupt 端点 delivers HID reports, and the 驱动程序 hands them up to userland through a 环形缓冲区 and `read`. This is the USB equivalent of a character-设备 interrupt-driven read loop.

Study how `uhid.c` uses the HID report 描述符 to understand what the 设备 is. The 描述符 is parsed at 附加 time, and the 驱动程序 populates its internal tables from the parse. Every HID 设备 describes itself this way; the 驱动程序 does not hard-code 设备 semantics.

### How to Study a Driver You Have Never Seen

Beyond the tour, you will encounter 驱动程序 in the tree that you have never seen. A general-purpose reading strategy helps:

Open the source file and scroll to the bottom. The registration macros are there. They tell you what the 驱动程序 附加 to (`uhub`, `usb`) and its name (`udbp`, `uhid`). Already you know where the 驱动程序 fits in the tree.

Scroll back up to the `usb_config` array (or the transfer declarations for non-USB驱动程序). Each entry is one channel. Count them; look at their types and directions. You now know the shape of the data path.

Look at the 探测 method. If it matches by VID/PID, the 设备 is vendor-specific. If it matches by class, the 驱动程序 supports a family of 设备. This tells you the scope of the 驱动程序.

Look at the 附加 method. Follow its labelled-goto chain. The labels give you the order of resource allocation: 互斥锁, channels, 字符设备, sysctls, and so on.

Finally, look at the data-path 回调. Each one is a three-state state machine. Read `USB_ST_TRANSFERRED` first; that is where the actual work happens. Then read `USB_ST_SETUP`; that is the kickoff. Then read `USB_ST_ERROR`; that is the recovery policy.

With this reading order, you can make sense of any USB驱动程序 in the tree in about 15 minutes. With practice, you will start to recognise patterns across 驱动程序 and know which ones are idiomatic (the ones to copy) and which ones are historical oddities (the ones to understand but not copy).

### Where to Go Beyond the Tour

The `/usr/src/sys/dev/usb/` tree has four subdirectories that are worth exploring:

`/usr/src/sys/dev/usb/misc/` contains simple, single-purpose 驱动程序: `uled`, `ugold`, `udbp`. If you are writing a new 设备-specific 驱动程序 that does not fit an existing class, read the 驱动程序 here to see how small 驱动程序 are structured.

`/usr/src/sys/dev/usb/serial/` contains the USB-to-serial bridge 驱动程序: `uftdi`, `uplcom`, `uslcom`, `u3g` (3G modems, which present as serial to userland), `uark`, `uipaq`, `uchcom`. If you are writing a new USB-to-串行驱动程序, start here.

`/usr/src/sys/dev/usb/input/` contains keyboard, mouse, and HID 驱动程序. `ukbd`, `ums`, `uhid`. If you are writing a new input 驱动程序, these are the patterns to follow.

`/usr/src/sys/dev/usb/net/` contains USB 网络驱动程序: `axge`, `axe`, `cdce`, `ure`, `smsc`. These are the 驱动程序 that bridge 第26章 to 第27章, because they combine the USB 框架 of this chapter with the `ifnet(9)` 框架 of the next. Reading one of them after finishing 第27章 is a productive exercise.

The `/usr/src/sys/dev/uart/` tree has fewer files but each is worth reading:

`/usr/src/sys/dev/uart/uart_core.c` is the 框架 core. Read this to understand what happens above your 驱动程序: how bytes flow in and out, how the TTY layer connects, how interrupts are dispatched.

`/usr/src/sys/dev/uart/uart_dev_ns8250.c` is the canonical reference 驱动程序. Read this after the 框架 core so you can see how a 驱动程序 plugs in.

`/usr/src/sys/dev/uart/uart_总线_pci.c` shows the PCI 总线-附加 glue for UARTs. If you ever need to write a UART 驱动程序 that 附加 to PCI, this is your starting point.

Each of these files is small enough to read in one sitting. Reading the source is not homework; it is how you learn a subsystem. 第26章 has given you the vocabulary and the mental model; the source is where you apply them.

## 性能考虑 for Transport Drivers

Most of 第26章 has focused on correctness: getting a 驱动程序 to 附加, do its work, and 分离 cleanly. Correctness always comes first. But once your 驱动程序 works, you will often want to know how fast it is, and whether its performance matches what the transport can sustain. This section gives you a practical 帧 for thinking about USB and UART performance without turning the chapter into a benchmarking manual.

### The USB Bus as a Shared Resource

Every 设备 on a USB 总线 shares the 总线 with every other 设备. The bandwidth is not divided fairly; it is allocated according to USB's scheduling rules. Control and interrupt 端点 get guaranteed periodic service. Bulk 端点 get what is left over, in a fair-share sense. Isochronous 端点 reserve bandwidth up front; if there is not enough, the allocation fails.

For a bulk-transferring 驱动程序, the practical upshot is this. Your effective bandwidth is the theoretical link speed (12, 480, 5000 Mbps) minus the overhead of other 设备' periodic traffic, minus the USB protocol overhead (roughly 10% on full-speed, less on higher speeds), minus the overhead of short transfers.

The last item is the one you can influence. A transfer of 16 KB is not 16 times more expensive than a transfer of 1 KB; the overhead of initiating and completing a transfer is fixed, and the data-transfer portion is close to linear in size. For high-throughput 批量传输, use large 缓冲区. The hardware is designed for this; the 框架 is designed for this; your 驱动程序 should be designed for this.

For an interrupt-transferring 驱动程序, the constraint is different. The interrupt 端点 polls at a fixed interval (configured by the 设备). The 框架 delivers a 回调 whenever the polled transfer completes. The maximum report rate is the 端点's polling rate. If the 设备 has a 1-ms interval, you get at most 1000 reports per second. Planning for interrupt-driven performance means planning around the polling rate.

### Latency: What Costs Microseconds, What Costs Milliseconds

USB is not a low-latency 总线. A single 控制传输 on full-speed USB takes roughly 1 ms round-trip. A single 批量传输 takes roughly 1 ms of framing overhead plus the time to move the data. Interrupt transfers are scheduled at the polling interval, so the minimum latency is the interval itself.

Compare this to native UART, where a character transmission takes roughly 1 ms at 9600 baud, 100 us at 115200 baud, and 10 us at 1 Mbps. A native UART 驱动程序 can push out a byte in hundreds of microseconds if it is well designed; a USB-to-serial bridge cannot match that, because each byte has to traverse USB first.

For your 驱动程序, this means: think about where latency matters for your use case. If you are building a monitoring 驱动程序 that reports once a second, USB is fine. If you are building an interactive controller where the user can feel each character round-trip, native UART is much better. If you are building a real-time control loop where characters must traverse in tens of microseconds, neither USB nor general-purpose UART is appropriate; you need a dedicated 总线 with known timing.

### When to Rearm: The Classic USB Tradeoff

A key decision in any streaming USB驱动程序 is where in the 回调 to re-arm the transfer. There are two viable patterns:

**Rearm after work.** In `USB_ST_TRANSFERRED`, do the work (parse the data, hand it up, update state), then rearm. Simple to implement. Has a latency cost: the time between the previous completion and the next submission is the time it took to do the work.

**Rearm before work, using multiple 缓冲区.** In `USB_ST_TRANSFERRED`, immediately rearm with a fresh 缓冲区, then do the work on the just-completed 缓冲区. This requires multiple `帧` in the `usb_config` (so the 框架 rotates through a pool of 缓冲区) or two parallel transfer channels. Has near-zero latency between transfers because the hardware always has a 缓冲区 ready.

Most 驱动程序 in the tree use the first pattern because it is simpler. The second pattern is used in high-throughput 驱动程序 where hiding the work latency matters. `ugold.c` is the first pattern; some of the USB 以太网 驱动程序 in `/usr/src/sys/dev/usb/net/` are the second.

### Buffer Sizing

For 批量传输, the 缓冲区 size is a knob. Larger 缓冲区 amortise the per-transfer overhead, but they also delay the delivery of partial data and increase memory usage. The typical values in the tree are between 1 KB and 64 KB.

For 中断传输, the 缓冲区 size is usually small (8 to 64 bytes) because the 端点 itself limits the report size. Do not make this larger than the 端点's `wMaxPacketSize`; the extra 缓冲区 is wasted.

For 控制传输, the 缓冲区 size is determined by the protocol of the specific operation. The `usb_设备_request` header is always 8 bytes; the data portion depends on the request.

### UART Performance

For a UART 驱动程序, performance is usually a question of interrupt efficiency. A 16550A with a FIFO depth of 16 bytes at 115200 baud needs to be serviced roughly every 1.4 ms in the worst case. If your 中断处理程序 takes longer than that, the FIFO overflows and data is lost. Modern UARTs (16750, 16950, ns16550 variants on embedded SoCs) often have deeper FIFOs (64, 128, or 256 bytes) specifically to relax this constraint.

The `uart(4)` 框架 handles the FIFO management for you through `uart_ops->rxready` and the 环形缓冲区. What you control as the 驱动程序 author is: how fast your implementation of `getc` is, how fast `putc` is, and whether your 中断处理程序 is sharing the CPU with other work.

For higher 波特率s (921600, 1.5M, 3M), a raw 16550A is not enough. These rates require either a chip with a larger FIFO or a 驱动程序 that uses DMA to move characters directly to memory. The `uart(4)` 框架 supports DMA-backed 驱动程序, but the vast majority of 驱动程序 (including `ns8250`) do not use it. DMA support is usually reserved for embedded platforms that specifically provide it.

### Concurrency and Lock Hold Times

A USB 回调 runs with the 驱动程序's 互斥锁 held. If the 回调 takes a long time (copying a large 缓冲区, doing complex processing), no other 回调 can run, and no 分离 can complete. Keep 回调 work short.

The idiomatic pattern for non-trivial work is: in the 回调, copy the data out of the 框架 缓冲区 into a private 缓冲区 in softc, then mark the data as ready and wake a consumer. The consumer (userland via `read`, or a worker taskqueue) does the heavy processing without the 驱动程序 互斥锁.

For a UART 驱动程序, the same principle applies. The `rxready` and `getc` methods must be fast because they run in interrupt context. Heavy processing is done later, outside the interrupt, by the TTY layer and user processes.

### Measuring, Not Guessing

The best way to answer a performance question is to measure. The `dtrace` hooks on `usbd_transfer_submit` and related functions let you time transfers to microsecond precision. `sysctl -a | grep usb` exposes per-设备 statistics. For UARTs, `sysctl -a dev.uart` and the TTY statistics in `vmstat` tell you where time is going.

Do not optimise a 驱动程序 blindly. Run the workload, measure, find the bottleneck, and fix what actually matters. For most 驱动程序, the bottleneck is not the transfer itself but something surrounding it: memory allocation, locking, or a poorly sized 缓冲区.

## 常见错误 When Writing Your First Transport Driver

The patterns in this chapter are the right way to write a transport 驱动程序. But patterns are easier to describe than to apply. Most first-time 驱动程序 are written with every pattern followed correctly in principle but misapplied in practice. This section lists the specific mistakes that appear most often when someone sits down to write a USB or UART 驱动程序 for the first time. Each mistake is paired with the correction and a short explanation of why the correction is necessary.

Read this section once before you write your first 驱动程序, and again when you are debugging one. The mistakes are surprisingly universal; almost every experienced FreeBSD 驱动程序 author has made several of them at some point.

### Mistake 1: Taking the Framework Mutex Explicitly in a Callback

The mistake looks like this:

```c
static void
my_bulk_read_callback(struct usb_xfer *xfer, usb_error_t error)
{
    struct my_softc *sc = usbd_xfer_softc(xfer);

    mtx_lock(&sc->sc_mtx);   /* <-- wrong */
    /* ... do work ... */
    mtx_unlock(&sc->sc_mtx);
}
```

The 框架 has already acquired the 互斥锁 before calling the 回调. Taking it a second time is a self-deadlock on most 互斥锁 implementations and an extra uncontested acquisition on others. On some 内核 configurations, it will panic immediately with a "recursive lock" assertion from WITNESS.

The correction is to simply not lock. The 框架 guarantees that 回调 are invoked with the softc 互斥锁 held. Your 回调 just does its work and returns; the 框架 releases the 互斥锁 on return.

### Mistake 2: Calling Framework Primitives Without the Mutex Held

The opposite mistake is also common:

```c
static int
my_userland_write(struct cdev *dev, struct uio *uio, int ioflag)
{
    struct my_softc *sc = dev->si_drv1;

    /* no lock taken */
    usbd_transfer_start(sc->sc_xfer[MY_BULK_TX]);   /* <-- wrong */
    return (0);
}
```

Most 框架 primitives (`usbd_transfer_start`, `usbd_transfer_stop`, `usbd_transfer_submit`) expect the caller to hold the associated 互斥锁. Calling them without the 互斥锁 is a race: the 框架's own state can be modified by a concurrent 回调 while you are issuing the primitive.

The correction is to take the 互斥锁 around the call:

```c
mtx_lock(&sc->sc_mtx);
usbd_transfer_start(sc->sc_xfer[MY_BULK_TX]);
mtx_unlock(&sc->sc_mtx);
```

This is the idiomatic pattern. The 框架 provides the locking; the 驱动程序 provides the 互斥锁.

### Mistake 3: Forgetting `USB_ERR_CANCELLED` Handling

The 框架 uses `USB_ERR_CANCELLED` to tell a 回调 that its transfer is being torn down (typically during 分离). If your 回调 handles this error the same way it handles other errors (for example by rearming the transfer), 分离 will hang forever because the transfer never actually stops.

The correct pattern is:

```c
case USB_ST_ERROR:
    if (error == USB_ERR_CANCELLED) {
        return;   /* do not rearm; the framework is tearing us down */
    }
    /* handle other errors, possibly rearm */
    break;
```

Omitting the cancellation check is one of the most common reasons a 驱动程序 分离 cleanly in development (because the ref count happens to be zero) but hangs in production (because a read was in-flight when 分离 ran).

### Mistake 4: Submitting to a Channel That Has Not Been Started

A transfer channel is inert until `usbd_transfer_start` has been called on it. Calling `usbd_transfer_submit` on an inactive channel is a no-op in some 框架 versions and a panic in others.

The correct pattern is to call `usbd_transfer_start` from userland-initiated work (in response to an open, for instance) and leave the channel active until 分离. Do not call `usbd_transfer_submit` directly; let `usbd_transfer_start` schedule the first 回调, and rearm from `USB_ST_SETUP` or `USB_ST_TRANSFERRED`.

### Mistake 5: Assuming `USB_GET_STATE` Returns the Real Hardware State

`USB_GET_STATE(xfer)` returns the state the 框架 wants the 回调 to handle at this moment. It does not report the underlying hardware state. The three states `USB_ST_SETUP`, `USB_ST_TRANSFERRED`, and `USB_ST_ERROR` are 框架 concepts, not hardware concepts.

In particular, `USB_ST_TRANSFERRED` means "the 框架 thinks this transfer completed." If the hardware is misbehaving (spurious transfer complete interrupts, split completions), the 回调 may be called with `USB_ST_TRANSFERRED` even when the actual transfer has not fully drained. This is rare, but when debugging, do not assume the 框架 state is ground truth about the hardware.

### Mistake 6: Using `M_WAITOK` in a Callback

A USB 回调 runs in an environment where sleeping is not allowed. Memory allocations in a 回调 must use `M_NOWAIT`. Using `M_WAITOK` will assert or panic.

A more subtle version of this mistake is calling a helper that internally uses `M_WAITOK`. For example, some 框架 helpers sleep; calling them from a 回调 is forbidden. If you need to do work that would require sleeping (DNS lookup, disk I/O, USB 控制传输 from a USB 回调), queue it to a taskqueue and let the taskqueue worker do the work outside the 回调.

### Mistake 7: Forgetting `MODULE_DEPEND` on `usb`

A USB驱动程序 module that does not declare `MODULE_DEPEND(my, usb, 1, 1, 1)` will fail to load with a cryptic unresolved-symbol error:

```text
link_elf_obj: symbol usbd_transfer_setup undefined
```

The symbol is undefined because the `usb` module has not been loaded, and the linker cannot resolve the 驱动程序's dependency on it. Adding the correct `MODULE_DEPEND` directive causes the 内核模块 loader to automatically load `usb` before your 驱动程序, which resolves the symbol and lets your 驱动程序 附加.

Every USB驱动程序 must have `MODULE_DEPEND(驱动程序name, usb, 1, 1, 1)`. Every UART-框架 驱动程序 must have `MODULE_DEPEND(驱动程序name, uart, 1, 1, 1)`. Every `ucom(4)` 驱动程序 must depend on both `usb` and `ucom`.

### Mistake 8: Mutable State in a Read-Only Path

Imagine a 驱动程序 that exposes a status field through a `sysctl`. The sysctl handler reads the field from the softc without taking the 互斥锁:

```c
static int
my_sysctl_status(SYSCTL_HANDLER_ARGS)
{
    struct my_softc *sc = arg1;
    int val = sc->sc_status;   /* <-- unlocked read */
    return (SYSCTL_OUT(req, &val, sizeof(val)));
}
```

If the field can be updated by a 回调 (which runs under the 互斥锁), and read by the sysctl handler (which does not take the 互斥锁), you have a data race. On modern platforms, word-sized reads are usually atomic, so the race is often invisible. But on platforms where they are not, or when the field is wider than a word, you can get torn reads.

The correction is to take the 互斥锁 for the read:

```c
mtx_lock(&sc->sc_mtx);
val = sc->sc_status;
mtx_unlock(&sc->sc_mtx);
```

Even if the race is invisible on x86, taking the lock documents your intent and protects against future changes (like widening the field to 64 bits).

### Mistake 9: Stale Pointers After `usbd_transfer_unsetup`

`usbd_transfer_unsetup` frees the transfer channels. The pointer in `sc->sc_xfer[i]` is no longer valid after the call returns. If any other code in your 驱动程序 uses that pointer after unsetup, the behaviour is undefined.

The correction is to zero the array after unsetup:

```c
usbd_transfer_unsetup(sc->sc_xfer, MY_N_TRANSFERS);
memset(sc->sc_xfer, 0, sizeof(sc->sc_xfer));   /* optional but defensive */
```

More importantly, structure your 分离 so that nothing in the 驱动程序 can observe the stale pointers. This usually means setting a "分离ing" flag in the softc before calling unsetup, and having every other code path check the flag before using the pointers.

### Mistake 10: Not Zeroing the Softc's `分离ing` Flag at Attach Time

If your softc uses a `分离ing` flag to coordinate 分离, the flag must start at zero when 附加 is called. This is normally automatic (the 框架 zero-fills the softc), but if you have any field that needs a non-zero initial value, be careful not to accidentally initialise `分离ing` to a nonzero value.

A 驱动程序 that starts with `分离ing = 1` will appear to "分离 before it ever 附加ed," which shows up as a 驱动程序 that 附加 normally but refuses to respond to any I/O.

### Mistake 11: Forgetting to Destroy the Device Node on Detach

If your 驱动程序 creates a 字符设备 with `make_dev` in 附加, you must destroy it with `destroy_dev` in 分离. Forgetting this leaves a stale `/dev` entry that points to freed memory. Userland programs that open the stale node will panic the 内核.

The correction is to call `destroy_dev(sc->sc_cdev)` in 分离, and always before the softc fields it references are freed.

A stronger pattern is to order the `destroy_dev` call first in 分离 (before any other cleanup). This 块 new opens and waits for existing opens to close, so that by the time the rest of 分离 runs, no userland code can reach the 驱动程序.

### Mistake 12: Racing on the Character Device Open

Even with `destroy_dev` in the right place, there is a window between 附加 succeeding and the first `open()` succeeding where the 驱动程序's state is being initialised. If your open handler assumes certain softc fields are valid, and 附加 has not finished initialising them when the first open arrives, the open will see garbage.

The correction is to call `make_dev` last in 附加, only after everything else is fully initialised. This way, the `/dev` entry does not appear until the 驱动程序 is ready to service opens. Correspondingly, call `destroy_dev` first in 分离, before tearing anything down.

### Mistake 13: Overlooking the TTY Layer's Own Locking

UART 驱动程序 integrate with the TTY layer, which has its own locking rules. In particular, the TTY layer holds `tty_lock` when it calls into the 驱动程序's `tsw_param`, `tsw_open`, and `tsw_close` methods. If the 驱动程序 then takes another lock inside these methods, the lock order is `tty_lock -> 驱动程序_互斥锁`. If any other code path takes the 驱动程序 互斥锁 and then the tty lock, you have a lock order inversion, and WITNESS will catch it.

The correction is to respect the lock order that the 框架 establishes. For UART 驱动程序, the order is documented in `/usr/src/sys/dev/uart/uart_core.c`. When in doubt, run under WITNESS with `WITNESS_CHECKORDER` enabled; it will detect any violation immediately.

### Mistake 14: Not Handling Zero-Length Data in Read or Write

A userland `read` or `write` with a zero-length 缓冲区 is legal. Your 驱动程序 must handle it, either by immediately returning zero or by propagating the zero-length request through the 框架. Forgetting this case often produces a 驱动程序 that "mostly works" but fails weird test scenarios.

The simplest correction is:

```c
if (uio->uio_resid == 0)
    return (0);
```

at the top of your read and write functions.

### Mistake 15: Copying Data Before Checking the Transfer Status

In a read path, a common mistake is to unconditionally copy data out of the USB 缓冲区:

```c
case USB_ST_TRANSFERRED:
    usbd_copy_out(pc, 0, sc->sc_rx_buf, actlen);
    /* hand data up to userland */
    break;
```

If the transfer was a short read (`actlen < wMaxPacketSize`), the copy is correct for exactly `actlen` bytes but the 驱动程序 code may assume more. If the transfer was empty (`actlen == 0`), the copy does nothing and any subsequent code that operates on "just-received data" works on stale data from the previous transfer.

The correction is to always check `actlen` before acting on the data:

```c
case USB_ST_TRANSFERRED:
    if (actlen == 0)
        goto rearm;   /* nothing received */
    usbd_copy_out(pc, 0, sc->sc_rx_buf, actlen);
    /* work with exactly actlen bytes */
rearm:
    /* re-submit */
    break;
```

### Mistake 16: Assuming `termios` Values Are in Standard Units

The `termios` structure's `c_ispeed` and `c_ospeed` fields contain 波特率 values, but the encoding has historical oddities. On FreeBSD, speeds are integer values (9600, 38400, 115200). On some other systems, they are indices into a table. Porting code that assumed index-based speeds to FreeBSD without checking is a common source of "the 驱动程序 thinks the 波特率 is 13 instead of 115200" bugs.

The correction is to look at the actual FreeBSD implementation: `/usr/src/sys/sys/termios.h` and `/usr/src/sys/kern/tty.c`. The 波特率 in FreeBSD `termios` is an integer bit rate. When your 驱动程序 receives a `termios` in `param`, read `c_ispeed` and `c_ospeed` as integers.

### Mistake 17: Missing `设备_set_desc` or `设备_set_desc_copy`

The `设备_set_desc` family of calls sets the human-readable description that `dmesg` shows when the 设备 附加. Without it, `dmesg` shows a generic label (like "my_drv0: <unknown>"), which is confusing for users and for your own debugging.

The correction is to call `设备_set_desc` in 探测 (not 附加), before returning `BUS_PROBE_GENERIC` or similar:

```c
static int
my_probe(device_t dev)
{
    /* ... match check ... */
    device_set_desc(dev, "My Device");
    return (BUS_PROBE_DEFAULT);
}
```

Use `设备_set_desc_copy` when the string is dynamic (constructed from 设备 data); the 框架 will free the copy when the 设备 is 分离ed.

### Mistake 18: `设备_printf` in the Data Path Without Rate Limiting

The `设备_printf` call is fine for occasional messages. In a data-path 回调, it is not, because every single transfer prints a line to `dmesg` and to the console. A 1 Mbps stream of characters becomes a flood of log messages.

The correction is the `DLOG_RL` pattern from 第25章: rate-limit data-path log messages to one per second, or one per thousand events, whichever is appropriate. Keep full logging in the configuration and error paths; rate-limit in the data path.

### Mistake 19: Not Waking Readers on Device Removal

If a userland program is 块ed in `read()` waiting for data, and the 设备 is unplugged, the 驱动程序 must wake the reader and return an error (typically `ENXIO` or `ENODEV`). Forgetting to do this leaves the read 块ed forever, which is a resource leak and a hang.

The correction is to wake all sleepers in 分离 before returning:

```c
mtx_lock(&sc->sc_mtx);
sc->sc_detaching = 1;
wakeup(&sc->sc_rx_queue);
wakeup(&sc->sc_tx_queue);
mtx_unlock(&sc->sc_mtx);
```

And in the read path, check the flag after waking:

```c
while (sc->sc_rx_head == sc->sc_rx_tail && !sc->sc_detaching) {
    error = msleep(&sc->sc_rx_queue, &sc->sc_mtx, PZERO | PCATCH, "myrd", 0);
    if (error != 0)
        break;
}
if (sc->sc_detaching)
    return (ENXIO);
```

This is the idiomatic pattern and avoids the classic "userland process hangs after you unplug the 设备" bug.

### Mistake 20: Thinking "It Works on My Machine" Is Enough

Driver bugs can be hardware-dependent. A 驱动程序 that works on one machine may fail on another because of timing differences, interrupt delivery differences, or hardware quirks in the USB controller. A 驱动程序 that works with one model of a 设备 may fail with another model of the same family because of firmware differences.

The correction is to test on multiple machines, multiple USB hosts (xHCI, EHCI, OHCI), and multiple 设备 if possible. When something works on one and fails on another, the difference is information. Trace both, compare, and the bug usually becomes clear.

### What To Do After You Make One of These Mistakes

You will make several of these mistakes. This is normal. The way to learn is: debug the failure, identify which mistake it was, understand why it caused the specific symptom, and add the correction to your mental toolkit. Keep a note of which mistakes you have made in practice. When you see a new 驱动程序 failure, check your note; the answer is usually a mistake you have already solved once.

The specific mistakes above are collected from the author's own experience writing and debugging USB and UART 驱动程序 on FreeBSD. They are not exhaustive, but they are representative of the kinds of issues that come up. Reading 驱动程序 in the tree, attending FreeBSD developer forums, and submitting your work for code review are all ways to accelerate this kind of learning.

## 总结

第26章 has taken you on a long tour. It began with the idea that a transport-specific 驱动程序 is a New总线 驱动程序 plus a set of rules about how the transport works. It then built out the two transport-specific layers we are focusing on in 第6部分: USB and serial.

On the USB side, you learned the host-and-设备 model, the 描述符 hierarchy, the four transfer types, and the 热插拔 lifecycle. You walked through a complete 驱动程序 skeleton: the 匹配表, the 探测 method, the 附加 method, the softc, the 分离 method, and the registration macros. You saw how `struct usb_config` declares transfer channels and how `usbd_transfer_setup` brings them to life. You followed the three-state 回调 state machine through bulk, interrupt, and 控制传输, and you saw how `usbd_copy_in` and `usbd_copy_out` move data between the 驱动程序 and the 框架's 缓冲区. You learned the locking rules around transfer operations and the retry policies 驱动程序 should choose. By the end of Section 3, you had a mental model that would let you write a bulk-loopback 驱动程序 from scratch.

On the serial side, you learned that the TTY layer sits on top of two distinct 框架: `uart(4)` for 总线-附加ed UARTs and `ucom(4)` for USB-to-serial bridges. You saw the six-method structure of a `uart(4)` 驱动程序, the role of `uart_ops` and `uart_class`, and how the `ns8250` canonical 驱动程序 implements each method. You learned how `termios` settings flow from `stty` through the TTY layer into the 驱动程序's `param` path, and how hardware 流控制 is implemented at the 注册 level. For USB-to-serial 设备, you saw the distinct `ucom_回调` structure and how configuration methods translate termios changes into vendor-specific USB 控制传输.

For testing, you learned about `nmdm(4)` for pure-TTY testing, QEMU USB redirection for USB development, and a handful of userland tools (`cu`, `tip`, `stty`, `comcontrol`, `usbconfig`) that make 驱动程序 development manageable even without constant hardware access. You saw that much of 驱动程序 work is not 注册-level wrestling but careful arrangement of data flow through well-defined abstractions.

The hands-on labs and challenge exercises gave you concrete problems to work on. Each lab is short enough to finish in a sitting, and each challenge extends one of the core ideas from the main text.

Three habits from earlier chapters extended naturally into 第26章. The labelled-goto cleanup chain from 第25章 is the same pattern used in USB and UART 附加 routines. The softc-as-single-source-of-truth discipline from 第25章 is applied identically to USB and UART 驱动程序 state. The errno-returning helper function pattern is unchanged. What 第26章 added was transport-specific vocabulary and transport-specific abstractions built on top of those habits.

There is also a habit that 第26章 has introduced which will stay with you: the three-state 回调 state machine (`USB_ST_SETUP`, `USB_ST_TRANSFERRED`, `USB_ST_ERROR`). Every USB驱动程序 uses it. Learning to read this state machine is learning to read every USB 回调 in the tree. When you open `uftdi.c`, `ucycom.c`, `uchcom.c`, or any other USB驱动程序, you will see the same pattern. Recognising it is recognising the USB 框架's core abstraction.

Transport-specific 驱动程序 are where the book's abstract 框架 concepts become concrete. From here on, every chapter in 第6部分 will deepen your practical skill with one more transport or one more kind of 内核 service. The New总线 foundation from Part 3, the character-设备 basics from Part 4, and the discipline themes from 第5部分 are all in play simultaneously. You are no longer learning concepts in isolation; you are using them together.

## 通往第 27

第27章 turns to 网络驱动程序. Much of the structure will feel familiar: there is a New总线 附加ment, there is per-设备 state (called `if_softc` in 网络驱动程序), there is a 匹配表, there is a 探测-and-附加 sequence, there are 热插拔 considerations, and there is an integration with a higher 框架. But the higher 框架 here is `ifnet(9)`, the 接口-框架 abstraction for network 设备, and its idioms are different from those of USB and serial.

A 网络驱动程序 does not expose a 字符设备. It exposes an 接口, which is visible to userland through `ifconfig(8)`, through `netstat -i`, and through the 套接字 layer. Instead of `read(2)` and `write(2)`, 网络驱动程序 handle 数据包 input and 数据包 output through the network stack's pipeline. Instead of `termios` for configuration, they handle `SIOCSIFFLAGS`, `SIOCADDMULTI`, `SIOCSIFMEDIA`, and a host of other network-specific ioctls.

Many network cards also happen to use USB or PCIe as their underlying transport. A USB 以太网 adapter, for example, sits on USB (via `if_cdce` or a vendor-specific 驱动程序) and exposes an `ifnet(9)` 接口. A PCIe 以太网 card sits on PCIe and also exposes an `ifnet(9)` 接口. 第27章 will show how the same `ifnet(9)` 框架 sits on top of these very different transports, and how the separation lets you write a 驱动程序 that focuses on the 数据包-level protocol without worrying about the details of its transport.

One specific thing to look forward to is the contrast between how USB delivers 数据包 (as transfer completions, one 缓冲区 at a time, with explicit 流控制 at the transfer level) and how PCIe-based network cards deliver 数据包 (as DMA-from-hardware events with 描述符 rings). The 数据包 pipeline in the network stack is designed to hide this difference from the upper layers, but a 驱动程序 author has to understand both models because they determine the 驱动程序's internal structure.

第27章 will then turn to 块 设备驱动程序 (storage). That chapter will cover the GEOM 框架, which is FreeBSD's layered 块-设备 infrastructure. Block 驱动程序 have their own idioms: a different way of matching 设备, a different way of exposing state (through GEOM providers and consumers), and a fundamentally different data flow model (read and write operations on 扇区, with a strong consistency model).

Parts 7, 8, and 9 then cover the more specialised topics: 内核 services and advanced 内核 idioms, debugging and testing in depth, and distribution and packaging. By the end of the book, you will have written and maintained 驱动程序 across several transport layers and several 内核 subsystems. The foundation you have built in Chapters 21 through 26 will be the common ground across all of that work.

For now, keep your `myfirst_usb` 驱动程序. You will not extend it in later chapters, but the patterns it demonstrates will appear again in network, storage, and 内核-service contexts. Having your own working example on hand, something you wrote and understand completely, is a resource that pays back many times over as the book progresses.

## 快速参考

This reference collects the most important APIs, constants, and file locations from 第26章 into one place. Keep it open while writing or reading a 驱动程序; it is faster than rediscovering each name from the source tree.

### USB Driver APIs

| Function | Purpose |
|----------|---------|
| `usbd_lookup_id_by_uaa(table, size, uaa)` | Match 附加 arg against 匹配表 |
| `usbd_transfer_setup(udev, &ifidx, xfer, config, n, priv, mtx)` | Allocate transfer channels |
| `usbd_transfer_unsetup(xfer, n)` | Free transfer channels |
| `usbd_transfer_submit(xfer)` | Queue a transfer for execution |
| `usbd_transfer_start(xfer)` | Activate a channel |
| `usbd_transfer_stop(xfer)` | Deactivate a channel |
| `usbd_transfer_pending(xfer)` | Query whether a transfer is outstanding |
| `usbd_transfer_drain(xfer)` | Wait for any pending transfer to complete |
| `usbd_xfer_softc(xfer)` | Retrieve the softc from a transfer |
| `usbd_xfer_status(xfer, &actlen, &sumlen, &a帧, &n帧)` | Query transfer results |
| `usbd_xfer_get_帧(xfer, i)` | Get page-cache pointer for 帧 i |
| `usbd_xfer_set_帧_len(xfer, i, len)` | Set length of 帧 i |
| `usbd_xfer_set_帧(xfer, n)` | Set total 帧 count |
| `usbd_xfer_max_len(xfer)` | Query max transfer length |
| `usbd_xfer_set_stall(xfer)` | Schedule clear-stall on this pipe |
| `usbd_copy_in(pc, offset, src, len)` | Copy into 框架 缓冲区 |
| `usbd_copy_out(pc, offset, dst, len)` | Copy out of 框架 缓冲区 |
| `usbd_errstr(err)` | Error code to string |
| `USB_GET_STATE(xfer)` | Current 回调 state |
| `USB_VPI(vendor, product, info)` | Compact 匹配表 entry |

### USB Transfer Types (`usb.h`)

- `UE_CONTROL`: 控制传输 (request-response)
- `UE_ISOCHRONOUS`: isochronous (periodic, no retry)
- `UE_BULK`: bulk (reliable, no timing guarantee)
- `UE_INTERRUPT`: interrupt (periodic, reliable)

### USB Transfer Direction

- `UE_DIR_IN`: 设备 to host
- `UE_DIR_OUT`: host to 设备
- `UE_ADDR_ANY`: 框架 picks any matching 端点

### USB Callback States (`usbdi.h`)

- `USB_ST_SETUP`: ready to submit a new transfer
- `USB_ST_TRANSFERRED`: previous transfer succeeded
- `USB_ST_ERROR`: previous transfer failed

### USB Error Codes (`usbdi.h`)

- `USB_ERR_NORMAL_COMPLETION`: success
- `USB_ERR_PENDING_REQUESTS`: outstanding work
- `USB_ERR_NOT_STARTED`: transfer not started
- `USB_ERR_CANCELLED`: transfer cancelled (e.g., 分离)
- `USB_ERR_STALLED`: 端点 stalled
- `USB_ERR_TIMEOUT`: timeout expired
- `USB_ERR_SHORT_XFER`: received less data than requested
- `USB_ERR_NOMEM`: out of memory
- `USB_ERR_NO_PIPE`: no matching 端点

### Registration Macros

- `DRIVER_MODULE(name, parent, 驱动程序, evh, arg)`: 注册 驱动程序 with 内核
- `MODULE_DEPEND(name, dep, min, pref, max)`: declare module dependency
- `MODULE_VERSION(name, version)`: declare module version
- `USB_PNP_HOST_INFO(table)`: export 匹配表 to `devd`
- `DEVMETHOD(name, func)`: declare method in method table
- `DEVMETHOD_END`: terminate method table

### UART Framework APIs

| Function | Header | Purpose |
|----------|--------|---------|
| `uart_getreg(bas, offset)` | `uart.h` | Read a UART 注册 |
| `uart_setreg(bas, offset, value)` | `uart.h` | Write a UART 注册 |
| `uart_barrier(bas)` | `uart.h` | Memory barrier for 注册 access |
| `uart_总线_探测(dev, regshft, regiowidth, rclk, rid, chan, quirks)` | `uart_总线.h` | Framework 探测 helper |
| `uart_总线_附加(dev)` | `uart_总线.h` | Framework 附加 helper |
| `uart_总线_分离(dev)` | `uart_总线.h` | Framework 分离 helper |

### `uart_ops` Methods

- `探测(bas)`: chip present?
- `init(bas, baud, databits, stopbits, 奇偶校验)`: initialise chip
- `term(bas)`: shut down chip
- `putc(bas, c)`: send one character (polling)
- `rxready(bas)`: is data available?
- `getc(bas, mtx)`: read one character (polling)

### `ucom_回调` Methods

- `ucom_cfg_open`, `ucom_cfg_close`: open/close hooks
- `ucom_cfg_param`: termios changed
- `ucom_cfg_set_dtr`, `ucom_cfg_set_rts`, `ucom_cfg_set_break`, `ucom_cfg_set_ring`: signal control
- `ucom_cfg_get_status`: read line and modem status bytes
- `ucom_pre_open`, `ucom_pre_param`: validation hooks (return errno)
- `ucom_ioctl`: chip-specific ioctl handler
- `ucom_start_read`, `ucom_stop_read`: enable/disable read
- `ucom_start_write`, `ucom_stop_write`: enable/disable write
- `ucom_tty_name`: customise the TTY 设备-node name
- `ucom_poll`: poll for events
- `ucom_free`: final cleanup

### Key Source Files

- `/usr/src/sys/dev/usb/usb.h`: USB protocol definitions
- `/usr/src/sys/dev/usb/usbdi.h`: USB驱动程序 接口, `USB_ERR_*` codes
- `/usr/src/sys/dev/usb/usbdi_util.h`: convenience helpers
- `/usr/src/sys/dev/usb/usbdevs.h`: Vendor/product constants (build-generated by the FreeBSD build system from `/usr/src/sys/dev/usb/usbdevs`; not present in a clean source tree until the 内核 or 驱动程序 is built)
- `/usr/src/sys/dev/usb/controller/`: Host controller 驱动程序
- `/usr/src/sys/dev/usb/misc/uled.c`: Simple LED 驱动程序 (reference)
- `/usr/src/sys/dev/usb/serial/uftdi.c`: FTDI 驱动程序 (reference)
- `/usr/src/sys/dev/usb/serial/usb_serial.h`: `ucom_回调` definition
- `/usr/src/sys/dev/usb/serial/usb_serial.c`: ucom 框架
- `/usr/src/sys/dev/uart/uart.h`: `uart_getreg`, `uart_setreg`, `uart_barrier`
- `/usr/src/sys/dev/uart/uart_总线.h`: `uart_class`, `uart_softc`, 总线 helpers
- `/usr/src/sys/dev/uart/uart_cpu.h`: `uart_ops`, CPU-side glue
- `/usr/src/sys/dev/uart/uart_core.c`: UART 框架 body
- `/usr/src/sys/dev/uart/uart_tty.c`: UART-TTY integration
- `/usr/src/sys/dev/uart/uart_dev_ns8250.c`: ns8250 reference 驱动程序
- `/usr/src/sys/dev/ic/ns16550.h`: 16550 注册 definitions
- `/usr/src/sys/dev/nmdm/nmdm.c`: null-modem 驱动程序

### Userland Diagnostic Commands

| Command | Purpose |
|---------|---------|
| `usbconfig list` | List USB 设备 |
| `usbconfig -d ugenN.M dump_all_config_desc` | Dump 描述符 |
| `usbconfig -d ugenN.M dump_stats` | Transfer statistics |
| `usbconfig -d ugenN.M reset` | Reset 设备 |
| `stty -a -f /dev/设备` | Show termios settings |
| `stty 115200 -f /dev/设备` | Set 波特率 |
| `comcontrol /dev/设备` | Show modem signals |
| `cu -l /dev/设备 -s speed` | Interactive session |
| `tip name` | Named connection (via `/etc/remote`) |
| `kldload mod.ko` | Load 内核模块 |
| `kldunload mod` | Unload 内核模块 |
| `kldstat` | List loaded modules |
| `dmesg -w` | Stream 内核 messages |
| `sysctl hw.usb.*` | Query USB 框架 |
| `sysctl dev.uart.*` | Query UART instances |

### Standard Development Flags

Debug-mode 内核 options to enable during development:
- `options INVARIANTS`: assertion checking
- `options INVARIANT_SUPPORT`: required alongside INVARIANTS
- `options WITNESS`: lock order checking
- `options WITNESS_SKIPSPIN`: skip spin locks in WITNESS (perf)
- `options WITNESS_CHECKORDER`: verify every lock acquisition
- `options DDB`: 内核 debugger
- `options KDB`: 内核 debugger support
- `options USB_DEBUG`: extensive USB logging

These options should be enabled on development machines, not production.

## 术语表

The following terms appeared in this chapter. Some are new; others were introduced earlier and are repeated here for convenience. Definitions are brief and intended as a quick reminder, not as a replacement for the main-text explanations.

**Address (USB).** A number from 1 to 127 that the host assigns to a 设备 during enumeration. Each physical 设备 on a 总线 has a unique address.

**Attach.** The 框架-called method where a 驱动程序 takes ownership of a newly discovered 设备, allocates resources, initialises state, and begins operation. Paired with `分离`.

**Bulk transfer.** A USB transfer type designed for reliable, high-throughput, non-time-critical data. Used for mass storage, printers, network adapters.

**Callout.** A FreeBSD mechanism for scheduling a function to run after a specific delay. Used by 驱动程序 for timeouts and periodic tasks.

**Callin node.** A TTY 设备 node (usually `/dev/ttyuN`) where opening 块 until carrier detect is asserted. Historically used for answering incoming modem calls.

**Callout node.** A TTY 设备 node (usually `/dev/cuauN`) where opening does not wait for carrier detect. Used for initiating connections or for non-modem 设备.

**CDC ACM.** Communication Device Class, Abstract Control Model. The USB standard for virtual 串行端口s. Handled in FreeBSD by the `u3g` 驱动程序.

**Character 设备.** A UNIX 设备 abstraction for byte-oriented 设备. Exposed to userland through `/dev` entries. Introduced in Chapter 24.

**Class 驱动程序.** A USB驱动程序 that handles an entire class of 设备 (all HID 设备, all mass-storage 设备) rather than a single vendor's product. Matches on 接口 class/subclass/protocol.

**Clear-stall.** A USB operation that clears a stall condition on an 端点. Handled by the FreeBSD USB 框架 when `usbd_xfer_set_stall` is called.

**Configuration (USB).** A named set of 接口 and 端点 a USB 设备 can expose. A 设备 usually has one configuration but may have several.

**Control transfer.** A USB transfer type designed for small, infrequent, request-response exchanges. Used for configuration and status.

**`cuau`.** Naming prefix for the callout-side TTY 设备 of a 总线-附加ed UART. Example: `/dev/cuau0`.

**`cuaU`.** Naming prefix for the callout-side TTY 设备 of a USB-provided 串行端口. Example: `/dev/cuaU0`.

**Descriptor (USB).** A small data structure a USB 设备 provides, describing itself or one of its components. Types include 设备, configuration, 接口, 端点, and string 描述符.

**Detach.** The 框架-called method where a 驱动程序 releases all resources and prepares for the 设备 to vanish. Paired with `附加`.

**`devd`.** The FreeBSD 设备-event daemon that reacts to 内核 notifications about 设备 附加 and 分离. Responsible for auto-loading modules for newly-discovered 设备.

**Device (USB).** A single physical USB peripheral connected to a port. Contains one or more configurations.

**DMA.** Direct Memory Access. A mechanism where hardware can read or write memory without CPU involvement. Used by high-performance USB 主机控制器s and PCIe network cards.

**Echo loopback.** A test configuration in which a 设备 echoes whatever it receives, used to validate bidirectional data flow.

**Endpoint.** A USB communication channel within an 接口. Each 端点 has a direction (IN or OUT) and a transfer type. Matches one hardware FIFO on the 设备.

**Enumeration.** The USB process by which a newly 附加ed 设备 is discovered, assigned an address, and has its 描述符 read by the host.

**FIFO (hardware).** A small 缓冲区 on a UART or USB chip that holds bytes during transfer. Typical 16550 FIFO is 16 bytes; many modern UARTs have 64 or 128.

**FTDI.** A company that makes popular USB-to-serial adapter chips. Drivers for FTDI chips are in `/usr/src/sys/dev/usb/serial/uftdi.c`.

**`ifnet(9)`.** The FreeBSD 框架 for network 设备驱动程序. Covered in 第27章.

**Interface (USB).** A logical grouping of 端点 within a USB 设备. A multi-function 设备 can expose multiple 接口.

**Interrupt handler.** A function the 内核 runs in response to a hardware interrupt. In the UART context, the 框架 provides a default 中断处理程序.

**Interrupt transfer.** A USB transfer type designed for low-bandwidth, periodic, latency-critical data. Used for keyboards, mice, HIDs.

**Isochronous transfer.** A USB transfer type designed for real-time streams with guaranteed bandwidth but no delivery guarantee. Used for audio and video.

**`kldload`, `kldunload`.** FreeBSD commands for loading and unloading 内核模块.

**`kobj`.** FreeBSD's object-oriented 内核 框架. Used for method dispatch in New总线 and other subsystems.

**Match table.** An array of `STRUCT_USB_HOST_ID` (for USB) or equivalent entries that a 驱动程序 uses to declare which 设备 it supports.

**Modem control 注册 (MCR).** A 16550 注册 that controls modem output signals (DTR, RTS).

**Modem status 注册 (MSR).** A 16550 注册 that reports modem input signals (CTS, DSR, CD, RI).

**`nmdm(4)`.** FreeBSD's null-modem 驱动程序. Creates pairs of linked virtual TTYs for testing. Loaded with `kldload nmdm`.

**ns8250.** A canonical 16550-compatible UART 驱动程序 for FreeBSD. At `/usr/src/sys/dev/uart/uart_dev_ns8250.c`.

**Pipe.** A term for a bidirectional USB transfer channel from the host's perspective. A host has one pipe per 端点.

**Port (USB).** A downstream 附加ment point on a hub. Each port can have one 设备 (which may itself be a hub).

**Probe.** The 框架-called method where a 驱动程序 examines a candidate 设备 and decides whether to 附加. Returns zero for a match, nonzero errno for a reject.

**Probe-and-附加.** The two-phase handshake by which New总线 binds 驱动程序 to 设备. Probe tests the match; 附加 does the work.

**Retry policy.** A 驱动程序's rule for what to do when a transfer fails. Common policies: rearm on every error, rearm up to N times then give up, rearm only for specific errors.

**Ring 缓冲区.** A fixed-size circular 缓冲区 used by the UART 框架 to 缓冲区 data between the chip and the TTY layer.

**RTS/CTS.** Request To Send / Clear To Send. Hardware flow-control signals on a 串行端口.

**Softc.** The per-设备 state a 驱动程序 maintains. Named after "software context" by analogy with hardware 注册 state.

**Stall (USB).** A signal from a USB 端点 that it is not ready to accept more data until explicitly cleared by the host.

**`stty(1)`.** Userland utility for inspecting and changing TTY settings. Maps directly onto `termios` fields.

**Taskqueue.** A FreeBSD mechanism for deferring work to a worker thread. Used by 驱动程序 that need to do something that cannot run in an interrupt context.

**`termios`.** A POSIX structure that describes a TTY's configuration: 波特率, 奇偶校验, 流控制, line discipline flags, and many others. Set and queried by `tcsetattr(3)` and `tcgetattr(3)` from userland, or by `stty(1)`.

**Transfer (USB).** A single logical operation on a USB channel. Can be a single 数据包 or many.

**TTY.** Teletype. The UNIX abstraction for a serial 设备. Character-at-a-time I/O, line discipline, signal generation, terminal control.

**`ttydevsw`.** The structure a TTY 驱动程序 uses to 注册 its operations with the TTY layer. Analogous to `cdevsw` for 字符设备.

**`ttyu`.** Naming prefix for the callin-side TTY 设备 of a 总线-附加ed UART. Example: `/dev/ttyu0`.

**`uart(4)`.** FreeBSD's 框架 for UART 驱动程序. Handles registration, 缓冲区ing, TTY integration. Drivers implement `uart_ops` hardware methods.

**`uart_bas`.** "UART Bus Access Structure." The 框架's abstraction for 注册 access to a UART, hiding whether the 寄存器 are in I/O space or memory-mapped.

**`uart_class`.** The 框架 描述符 that identifies a UART chip family. Pairs with `uart_ops` to give the 框架 everything it needs.

**`uart_ops`.** The table of six hardware-specific methods (`探测`, `init`, `term`, `putc`, `rxready`, `getc`) that a UART 驱动程序 implements.

**`ucom(4)`.** FreeBSD's 框架 for USB-to-serial 设备驱动程序. Sits on top of USB transfers, provides TTY integration.

**`ucom_回调`.** The structure a `ucom(4)` client uses to 注册 its 回调 with the 框架.

**`ugen(4)`.** FreeBSD's generic USB驱动程序. Exposes raw USB access through `/dev/ugenN.M` for userland programs. Used when no specific 驱动程序 matches.

**`uhub`.** The FreeBSD 驱动程序 for USB hubs (including the root hub). A class 驱动程序 附加 to `uhub`, not to the USB 总线 directly.

**`usbconfig(8)`.** Userland utility for inspecting and controlling USB 设备. Can dump 描述符, reset 设备, enumerate state.

**`usb_config`.** A C structure a USB驱动程序 uses to declare each of its transfer channels: type, 端点, direction, 缓冲区 size, flags, 回调.

**`usb_fifo`.** A USB 框架 abstraction for byte-stream `/dev` nodes. Generic alternative to writing a custom `cdevsw`.

**`usb_template(4)`.** FreeBSD's USB 设备-side (gadget) 框架. Used on hardware that can act as both USB host and USB 设备.

**`usb_xfer`.** An opaque structure representing a single USB transfer channel. Allocated by `usbd_transfer_setup`, freed by `usbd_transfer_unsetup`.

**`usbd_copy_in`, `usbd_copy_out`.** Helpers for copying data between plain C 缓冲区 and USB 框架 缓冲区. Must be used instead of direct pointer access.

**`usbd_lookup_id_by_uaa`.** Framework helper that compares a USB 附加 argument against a 匹配表 and returns zero on match.

**`usbd_transfer_setup`, `_unsetup`.** The calls that allocate and free transfer channels. Called from `附加` and `分离` respectively.

**`usbd_transfer_submit`.** The call that hands a transfer to the 框架 for execution on the hardware.

**`usbd_transfer_start`, `_stop`.** The calls that activate or deactivate a channel. Activate triggers a 回调 in `USB_ST_SETUP`; deactivate cancels in-flight transfers.

**`USB_ST_SETUP`, `_TRANSFERRED`, `_ERROR`.** The three states of a USB 传输回调, as returned by `USB_GET_STATE(xfer)`.

**`USB_ERR_CANCELLED`.** The error code the 框架 passes to a 回调 when a transfer is being torn down (typically during 分离).

**`USB_ERR_STALLED`.** The error code when a USB 端点 returns a STALL handshake. Usually handled by calling `usbd_xfer_set_stall`.

**VID/PID.** Vendor ID / Product ID. A pair of 16-bit numbers that uniquely identifies a USB 设备 model.

**`WITNESS`.** A FreeBSD 内核 debugging option that tracks lock acquisition order and warns about violations.

**Callin 设备.** A TTY 设备 (named `/dev/ttyuN` or `/dev/ttyUN`) that 块 on open until the modem's carrier detect (CD) signal is asserted. Used by programs that accept incoming calls.

**Callout 设备.** A TTY 设备 (named `/dev/cuauN` or `/dev/cuaUN`) that opens immediately without waiting for carrier detect. Used by programs that initiate connections.

**`comcontrol(8)`.** Userland utility for controlling TTY options (drain behaviour, DTR, 流控制) that are not exposed through `stty`.

**Descriptor (USB).** A data structure that a USB 设备 returns when the host asks for its identity, configuration, 接口, or 端点. Hierarchical: 设备 描述符 contains configuration 描述符; configurations contain 接口; 接口 contain 端点.

**Endpoint (USB).** A named, typed communication channel inside a USB 设备. Has an address (1 through 15), a direction (IN or OUT), a type (control, bulk, interrupt, isochronous), and a maximum 数据包 size.

**Line discipline.** The TTY layer's pluggable layer between the 驱动程序 and userland. Standard disciplines include `termios` (canonical and raw modes). Line disciplines translate between raw bytes and the behaviour a user program expects.

**`msleep(9)`.** The 内核 sleep primitive used to 块 a thread on a channel with a 互斥锁 held. Paired with `wakeup(9)`, it implements producer-consumer patterns inside 驱动程序.

**`mtx_sleep`.** A synonym for `msleep` used in some parts of the tree. Functionally identical.

**Open/close pair.** The 字符设备 methods `d_open` and `d_close`. Every 驱动程序 that exposes a `/dev` node must handle these. Opens are usually where channels are started; closes are usually where channels are stopped.

**Short transfer.** A USB transfer that completes with fewer bytes than requested. Normal for bulk IN (where the 设备 sends a short 数据包 to signal "end of message") and for interrupt IN (where the 设备 sends a short 数据包 when it has less data than the maximum). Always check `actlen`.

**`USETW`.** A FreeBSD macro for setting a little-endian 16-bit field inside a USB 描述符 缓冲区. The USB wire format is always little-endian, so `USETW` hides the byte-swap.

This glossary is not exhaustive; it covers the terms this chapter actually used. For a broader FreeBSD USB reference, the `usbdi(9)` manual page is the definitive source. For the UART 框架, the source in `/usr/src/sys/dev/uart/` is the reference. When you encounter an unfamiliar term in either place, check here first; if not defined, go to the source.

### A Closing Note on Terminology Precision

One last piece of advice on vocabulary. The USB, TTY, and FreeBSD communities each have their own careful distinctions between terms that sound like synonyms. Confusing these in conversation with more experienced developers is a quick way to sound unsure; using them precisely is a quick way to sound at home.

"Device" in the USB context means the whole USB peripheral (the keyboard, the mouse, the serial adapter). "Interface" means a logical grouping of 端点 inside the 设备. An 接口 implements one function; a 设备 can have multiple 接口. When you say "the USB 设备 is a composite 设备," you are saying it has multiple 接口.

"Endpoint" and "pipe" are related but distinct. An 端点 is on the 设备; a pipe is the host's view of a connection to that 端点. In FreeBSD 驱动程序 code, the term "transfer channel" is often used in place of "pipe," because "pipe" overloads a more common meaning in UNIX.

"Transfer" and "transaction" are also distinct. A transfer is a logical operation (a read request for N bytes); a transaction is the USB-level 数据包 exchange that realises it. A 批量传输 of 64 bytes to an 端点 with a maximum 数据包 size of 64 is one transfer and one transaction. A 批量传输 of 512 bytes to the same 端点 is one transfer and eight transactions.

"UART" and "串行端口" are closely related but not identical. A UART is the chip (or the chip's logic 块); a 串行端口 is the physical connector and its wiring. One UART can back multiple 串行端口s in some configurations; one 串行端口 is always backed by exactly one UART.

"TTY" and "terminal" are related. A TTY is the 内核 abstraction for character-at-a-time I/O; a terminal is the userland view. A TTY has a controlling terminal property; a terminal has a TTY that it uses. In 驱动程序 code, TTY is almost always the more precise term.

Getting these right in writing and in code comments signals that you understand the design. And when you read someone else's code or documentation, noticing which term they chose tells you which layer of abstraction they are thinking about.
